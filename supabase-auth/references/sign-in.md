# Sign-in

Docs: <https://supabase.com/docs/reference/javascript/auth-signinwithpassword>, <https://supabase.com/docs/reference/javascript/auth-onauthstatechange>, <https://supabase.com/docs/guides/auth/auth-mfa>.

## Call

```ts
const { data, error } = await supabase.auth.signInWithPassword({ email, password });
```

Error mapping (keep copies generic — never confirm which part was wrong):

| Error message | UI |
| --- | --- |
| `Invalid login credentials` | "Wrong email or password." inline error |
| `Email not confirmed` | Send the user to the check-inbox screen (with resend, see `sign-up.md`) |
| 429 rate-limit style | "Too many attempts — try again in a minute." |
| anything else / network | generic retry copy |

## After success — the MFA gate

Immediately after a credentials sign-in, decide whether the challenge screen is needed:

```ts
const { data } = await supabase.auth.mfa.getAuthenticatorAssuranceLevel();
// data.currentLevel, data.nextLevel — of: "aal1" | "aal2"
```

| currentLevel | nextLevel | Meaning |
| --- | --- | --- |
| aal1 | aal1 | no MFA enrolled → straight into the app |
| aal1 | aal2 | **factors enrolled, not verified → route to `/mfa/challenge`** |
| aal2 | aal2 | MFA verified at this level → straight into the app |
| aal2 | aal1 | factor was just disabled (stale JWT; fixes itself on refresh) |

Helper to keep in `lib/`:

```ts
export async function needsMfaChallenge(): Promise<boolean> {
  const { data } = await supabase.auth.mfa.getAuthenticatorAssuranceLevel();
  return data.nextLevel === "aal2" && data.currentLevel !== "aal2";
}
```

The method is near-instant (it rarely touches the network) — safe to await before rendering.

## Session provider

One source of truth for session state (`lib/session.tsx` or similar). The route gate and every auth screen consume it — do not scatter `getSession()` calls through the UI:

```tsx
const [session, setSession] = useState<Session | null>(null);
const [loading, setLoading] = useState(true);

useEffect(() => {
  supabase.auth.getSession().then(({ data }) => {
    setSession(data.session);
    setLoading(false);
  });

  const { data: { subscription } } = supabase.auth.onAuthStateChange((event, next) => {
    setSession(next);
  });
  return () => subscription.unsubscribe();
}, []);
```

Event cheat-sheet:

| Event | Use it for |
| --- | --- |
| `INITIAL_SESSION` | session restored from storage on cold start |
| `SIGNED_IN` | credentials sign-in, confirmed sign-up via deep link, MFA verify, `setSession`/`exchangeCodeForSession` — land in the app |
| `SIGNED_OUT` | land on `/sign-in` |
| `TOKEN_REFRESHED` | ignored (silent) |
| `USER_UPDATED` | user object changed (email, password, metadata) — update pending UI from `session.user` |
| `PASSWORD_RECOVERY` | a recovery deep link completed — route to `/reset-password` (`password-reset.md`). This or the deep-link *route* is the reset trigger; pick one pattern for the app |

Rules: keep the listener lean (set state, maybe one navigate — no cascading awaits); gate rendering on `loading` (splash) so the first frame never flashes the sign-in screen over a restored session.

## Route gating

- expo-router projects: gate protected routes behind `session` (e.g. `Stack.Protect guard={session} redirect="/sign-in"`). Structural specifics live in the `expo-router` skill.
- Plain RN: render `session ? <App /> : gate` — with the MFA challenge potentially between them, per the AAL check above.

## Sign out (also used by account settings, `account-management.md`)

```ts
await supabase.auth.signOut({ scope: "global" }); // default in JS: revoke on ALL devices
await supabase.auth.signOut({ scope: "local" });  // current device only
```

The JS default is `global` — make the button's copy say which one it does ("Sign out" vs "Sign out everywhere"), because a plain `signOut()` does **not** match the "sign out of this device" expectation many UIs imply. Other devices controlled by this one get `SIGNED_OUT` semantics through a refresh failure, not an event — don't rely on cross-device realtime events.
