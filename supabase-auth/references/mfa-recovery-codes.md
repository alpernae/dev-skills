# MFA — Recovery codes (experimental)

Backup single-use codes that stand in for the authenticator: if the user loses their TOTP device, they verify with one of these at the challenge screen. Docs: <https://supabase.com/docs/reference/javascript/auth-mfa-recovery-codes>.

**Requirements:** recent `@supabase/supabase-js` (if `supabase.auth.mfa.recoveryCodes` is `undefined`, upgrade the package) **and** the experimental opt-in on the client:

```ts
createClient(url, publishableKey, {
  auth: { experimental: { recoveryCodes: true } },
});
```

## Generate — right after successful enrollment

```ts
const { data, error } = await supabase.auth.mfa.recoveryCodes.generate({
  friendlyName: "Backup codes", // optional
});

// data.codes: string[] — show them grouped, e.g. "K4M9-X7QP-2AB8-HT3Z"
const formatted = data.codes.map((c) => c.match(/.{1,4}/g)!.join("-"));
```

**The plaintext codes are returned exactly once and can never be fetched again.** The success UI must therefore be a dedicated screen:

1. Show the full list (grid, monospace), one row per code.
2. "Copy all" button (`expo-clipboard`'s `setStringAsync`).
3. Explicit warning: "Save these somewhere safe — they're shown only once. Each code works one time."
4. A confirmation ("I've saved my codes") that is required before continuing back into the app.
5. Never persist the codes in app storage, logs, or screenshots; never re-fetch them later.

Generate is for the *initial* set and for replacing a missing one. If a set already exists, `generate()` is refused — use `regenerate()` (below). Check with `getStatus()` first.

## Status + regenerate (MFA settings screen)

```ts
const { data } = await supabase.auth.mfa.recoveryCodes.getStatus();
// counts for the current set: total codes and how many are still unused — never returns the codes
```

```ts
const { data } = await supabase.auth.mfa.recoveryCodes.regenerate();
// fresh set shown ONCE (same once-only UI as generate); the previous set is invalidated
```

```ts
const { error } = await supabase.auth.mfa.recoveryCodes.unenroll();
// removes the recovery-code set (e.g. user disables MFA entirely)
```

Settings row copy: "Recovery codes: X unused of Y" + actions *Show remaining / Regenerate (previous codes stop working) / Remove all*. Regeneration reuses the once-only listing screen.

## Sign-in fallback (MFA challenge screen)

The challenge screen (`mfa-totp.md`) offers "Can't use your authenticator? **Use a recovery code**":

```ts
const norm = (raw: string) => (raw.replace(/[^a-zA-Z0-9]/gi, "").match(/.{1,4}/g) ?? [raw]).join("-");

const { error } = await supabase.auth.mfa.recoveryCodes.verify({
  code: norm(input), // single-use; upgrades the session to aal2 on success
});
```

- Normalize input exactly as above (users type codes with or without the dashes/spaces from their saved copy).
- Wrong/used/expired code → inline "That code isn't valid — check it and try again" (also teaches them the remaining count).
- On success the session is `aal2` — identical continuation to a TOTP verify; do not treat this path specially downstream.
- After the signal that all codes are used (status/`unused = 0`), still route through the normal flow — the remaining remediation (factor deletion by an admin) is in `mfa-totp.md`.
