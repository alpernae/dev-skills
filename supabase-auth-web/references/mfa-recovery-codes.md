# MFA — Recovery codes (experimental)

Backup single-use codes that stand in for the authenticator at the challenge step (lost-device path). Docs: <https://supabase.com/docs/reference/javascript/auth-mfa-recovery-codes>.

**Requirements:** recent `@supabase/supabase-js` (if `supabase.auth.mfa.recoveryCodes` is `undefined`, upgrade the package) **and** the experimental opt-in on the client — both stacks have it:

```ts
// Next.js browser client
createBrowserClient(url, key, { auth: { experimental: { recoveryCodes: true } } });

// Client-only
createClient(url, key, { auth: { experimental: { recoveryCodes: true } } });
```

## Generate — right after successful enrollment

```ts
const { data, error } = await supabase.auth.mfa.recoveryCodes.generate({
  friendlyName: "Backup codes", // optional
});

// data.codes: string[] — show grouped, e.g. "K4M9-X7QP-2AB8-HT3Z"
const formatted = data.codes.map((c) => c.match(/.{1,4}/g)!.join("-"));
```

**The plaintext codes are returned exactly once and can never be fetched again.** The success page must therefore be a dedicated screen:

1. Full list (grid, monospace), one row per code.
2. "Copy all" button (`navigator.clipboard.writeText`).
3. Explicit warning: "Save these somewhere safe — they're shown only once. Each code works one time."
4. A confirmation ("I've saved my codes") that must be acknowledged before continuing.
5. Never persist the codes in app storage, logs, or screenshots; never re-fetch them later.

Generate is for the *initial* set and replacing a missing one. If a set already exists, `generate()` is refused — check `getStatus()` first, then `regenerate()` (below).

## Status + regenerate (MFA settings page)

```ts
const { data } = await supabase.auth.mfa.recoveryCodes.getStatus();
// counts for the current set: total codes and how many remain unused — never returns the codes
```

```ts
const { data } = await supabase.auth.mfa.recoveryCodes.regenerate();
// fresh set shown ONCE (same once-only listing UI); the previous set is invalidated
```

```ts
const { error } = await supabase.auth.mfa.recoveryCodes.unenroll();
// removes the recovery-code set (e.g. user disables MFA entirely)
```

Settings row copy: "Recovery codes: X unused of Y" + actions *Show remaining / Regenerate (previous codes stop working) / Remove all*. Regeneration reuses the once-only listing page.

## Sign-in fallback (MFA challenge page)

The challenge page (`mfa-totp.md`) offers "Can't use your authenticator? **Use a recovery code**":

```ts
const norm = (raw: string) => (raw.replace(/[^a-zA-Z0-9]/gi, "").match(/.{1,4}/g) ?? [raw]).join("-");

const { error } = await supabase.auth.mfa.recoveryCodes.verify({
  code: norm(input), // single-use; upgrades the session to aal2 on success
});
```

- Normalize input exactly as above (users paste/type codes with or without dashes/spaces).
- Wrong/used/expired code → inline "That code isn't valid — check it and try again" (also surfaces the remaining count).
- On success the session is `aal2` — identical continuation to a TOTP verify (route into the app; Next.js: `router.refresh()`).
- When all codes are used (`getStatus` unused = 0), keep the normal flow — the remaining remediation (factor deletion by an admin) is in `mfa-totp.md`.
