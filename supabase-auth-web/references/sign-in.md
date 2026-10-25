# Sign-in

Docs: <https://supabase.com/docs/reference/javascript/auth-signinwithpassword>, <https://supabase.com/docs/guides/auth/server-side/nextjs> (auth-method policy), <https://supabase.com/docs/guides/auth/auth-mfa>.

## Call (both stacks — client-side form, `'use client'` on Next.js)

```ts
const { data, error } = await supabase.auth.signInWithPassword({ email, password });
```

Error mapping (generic — never reveal which field was wrong):

| Error message | UI |
| --- | --- |
| `Invalid login credentials` | "Wrong email or password." inline error |
| `Email not confirmed` | send to the check-inbox page (resend UI, see `sign-up.md`) |
| 429 rate-limit style | "Too many attempts — try again in a minute." |
| otherwise / network | generic copy |

## After success — the MFA gate

Immediately decide whether the challenge page is needed:

```ts
const { data } = await supabase.auth.mfa.getAuthenticatorAssuranceLevel();
// data.currentLevel, data.nextLevel — "aal1" | "aal2"
```

| currentLevel | nextLevel | Meaning |
| --- | --- | --- |
| aal1 | aal1 | no MFA enrolled → straight into the app |
| aal1 | aal2 | **enrolled, not verified → route to `/mfa/challenge`** |
| aal2 | aal2 | verified → straight into the app |
| aal2 | aal1 | factor just disabled (stale token; resolves on refresh) |

Helper for `lib/`:

```ts
export async function needsMfaChallenge(): Promise<boolean> {
  const { data } = await supabase.auth.mfa.getAuthenticatorAssuranceLevel();
  return data.nextLevel === "aal2" && data.currentLevel !== "aal2";
}
```

It's near-instant (rarely touches the network) — safe to await before routing.

## Auth-check policy (Next.js server code)

From the SSR guide — use the right checker in the right place:

| Method | Use it for |
| --- | --- |
| `supabase.auth.getClaims()` | **Protect pages/data** (proxy, server components). Verifies the token's signature locally (asymmetric keys, default) or via the auth server; refreshes near expiry. Never forged away by editing the cookie. |
| `supabase.auth.getUser()` | Server-confirmed, most current user record (network call) — e.g. before profile edits. |
| `supabase.auth.getSession()` | Raw tokens only. **Never inside server code** — it reads the cookie without revalidation; a forged cookie would render another user's page. Fine for initial UI state in client code. |

Gating: the proxy example in `setup.md` already gates with `claims.sub`. In MFA-mandatory mode, additionally check `claims.aal === "aal2"` (the `aal` claim is on the JWT) and redirect to `/mfa/challenge` instead of 403 — the MFA docs explicitly recommend a redirect to an authentication surface, not an error, since users may simply have abandoned the challenge step.

## Session state — client only (React SPA)

One provider near the root; consume everywhere; never scatter `getSession()` through pages:

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

Events: `INITIAL_SESSION` (restored), `SIGNED_IN`, `SIGNED_OUT` (→ `/sign-in`), `TOKEN_REFRESHED` (ignore), `USER_UPDATED`, `PASSWORD_RECOVERY` (→ `/reset-password`, see `password-reset.md`). Keep the listener lean; route on events, not on polling.

Next.js apps under cookie-SSR don't strictly need this provider — server components read `getClaims()` / the session per request; keep a small client provider anyway for reactive chrome (header avatar, nav) and to transport the session to client action pages. Call `router.refresh()` after any client-side auth mutation (sign in, sign out, verify, update) so Server Components pick up the new cookie state.

## Route gating

- **Next.js**: the proxy redirect (`setup.md`) is the authoritative gate; client-side hops (like sending the user with `?next=` to sign-in, or routing to `/mfa/challenge` after `needsMfaChallenge()`) polish the UI. The database itself stays protected by RLS regardless of what the UI shows.
- **Client-only React**: render `session ? <AppRoutes /> : <AuthGate />` off the provider; wait for `loading` (splash) so a restored session never flashes the sign-in page. Proxy-based gating doesn't exist for SPAs — the client guard is the only checkpoint, which is fine for UI routing while the database itself stays protected by RLS.

## Sign out (also used by account settings, `account-management.md`)

```ts
await supabase.auth.signOut({ scope: "global" }); // JS default — revoke on ALL devices
await supabase.auth.signOut({ scope: "local" });  // current browser only
```

Exactly like native: make the button copy say which it does ("Sign out" vs "Sign out everywhere") — a plain `signOut()` is `global`. Under Next.js cookie sessions the browser client clears the cookie via `setAll`; follow with `router.refresh()`.
