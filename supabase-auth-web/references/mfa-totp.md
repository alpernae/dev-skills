# MFA — Authenticator (TOTP)

Docs: <https://supabase.com/docs/guides/auth/auth-mfa/totp>, <https://supabase.com/docs/reference/javascript/auth-mfa>.

Identical API surface on both web stacks — calls come from Client Components (browser client). Two flows: **enrollment** and the **challenge step** at every sign-in. Factor type is always `totp` (no SMS/phone here).

## Enrollment flow

Steps 1–4: create factor → show QR/manual key → verify code → success. Hold state (`factor`, `code`, error) in the component.

**Step 1 — create the factor:**

```ts
const { data: factor, error } = await supabase.auth.mfa.enroll({ factorType: "totp" });
// factor.id, factor.totp.qr_code, factor.totp.secret, factor.totp.uri (otpauth://... for manual entry)
```

**Step 2 — show the QR.** `factor.totp.qr_code` is an SVG already encoded as a data URL — it goes straight into an `<img>` tag (the web path from the docs' official enrollment example):

```tsx
<img src={factor.totp.qr_code} alt="Scan this QR code with your authenticator app" width={220} height={220} />
```

Some authenticator flows need the manual key instead — show it grouped: `factor.totp.secret.match(/.{4}/g)?.join(" ")`.

**Step 3 — verify** (user types the 6-digit code from their authenticator):

```ts
const { data: chal } = await supabase.auth.mfa.challenge({ factorId: factor.id });
const { error } = await supabase.auth.mfa.verify({
  factorId: factor.id,
  challengeId: chal.id,
  code,
});
// retry on error = always create a NEW challenge first
// (challengeAndVerify({ factorId, code }) replaces the two calls in one-shot flows)
```

**Step 4 — success.** The factor is `verified`, the session upgrades to `aal2`, and **all of the user's other sessions are logged out** — the success copy should say so. Immediately follow with **recovery-code generation** (`mfa-recovery-codes.md`) — the backup for a lost authenticator; do not treat as optional.

Next.js: after enrollment success, `router.refresh()` so RSC-protected chrome re-renders with the upgraded token.

Errors worth specific copies: factor limit (max 2 active factors per user), challenge exhaustion, network.

## Challenge step (sign-in)

Reached when `needsMfaChallenge()` is true after `signInWithPassword` (`sign-in.md`). Page: code input (6 digits, `inputMode="numeric"`) + "Use a recovery code instead" fallback (`mfa-recovery-codes.md`).

```ts
const { data: factors } = await supabase.auth.mfa.listFactors();
// factors.data.totp: [{ id, friendly_name?, factor_type, status }], factors.data.phone: [...]
const factor = factors.data.totp[0];
if (!factor) throw new Error("No TOTP factors found!");

const { data: chal } = await supabase.auth.mfa.challenge({ factorId: factor.id });
const { error } = await supabase.auth.mfa.verify({
  factorId: factor.id,
  challengeId: chal.id,
  code,
});
```

- More than one factor → offer a picker.
- New challenge per attempt — never reuse a `challengeId` across verifications.
- On success the session refreshes in the background → `SIGNED_IN` → route into the app. Next.js: `router.refresh()` after routing so server-rendered sections see `aal2`.
- TOTP codes are valid for one 30s window (+1 interval of clock skew); nothing to enforce client-side beyond a clean error state.

## Factor settings (account page)

`listFactors()` lists; unenroll removes:

```ts
const { error } = await supabase.auth.mfa.unenroll({ factorId });
```

- Unenrolling a `verified` factor requires an `aal2` session; at `aal1` first `await supabase.auth.reauthenticate()` and route through the challenge page (AAL check tells you which).
- The token stays `aal2` until refresh — call `supabase.auth.refreshSession()` right after unenroll if the UI reflects AAL immediately.
- Always gate removal behind a confirm modal — losing the factor without recovery codes locks the user out of `aal2`.
- If everything is lost (authenticator gone, recovery codes exhausted), remediation is a **support-side admin action**: `supabaseAdmin.auth.admin.mfa.deleteFactor({ id, userId })` from a service-role context (Edge Function or server route). Mention it in the settings page's error copy if you build it.
