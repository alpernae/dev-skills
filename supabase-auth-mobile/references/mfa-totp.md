# MFA — Authenticator (TOTP)

Docs: <https://supabase.com/docs/guides/auth/auth-mfa/totp>, <https://supabase.com/docs/reference/javascript/auth-mfa>.

Two flows: **enrollment** (settings screen, or optionally right after first sign-in) and the **challenge step** at every sign-in. Factor type is always `totp` here (no SMS/phone).

## Enrollment flow

Three UI steps + success. State to hold: `factorId`, `factor` (secret/QR), `code`, error.

**Step 1 — create the factor:**

```ts
const { data: factor, error } = await supabase.auth.mfa.enroll({ factorType: "totp" });
// factor.id, factor.totp.qr_code (raw SVG string), factor.totp.secret, factor.totp.uri (otpauth://...)
```

**Scan step — render the QR** (native needs an SVG renderer):

```tsx
import { SvgXml } from "react-native-svg";

<SvgXml xml={factor.totp.qr_code} width={220} height={220} />
```

Either render the SVG directly (preferred — Supabase returns the QR ready to display, so no QR library is needed) or draw a QR from `factor.totp.uri` with a QR-rendering library if the design calls for custom styling.

**Manual key step:** some users add the key by hand. Show the secret, grouped:

```ts
const grouped = factor.totp.secret.match(/.{4}/g)?.join(" ") ?? factor.totp.secret;
```

**Verify step** (user types the 6-digit code from their authenticator):

```ts
const { data: chal } = await supabase.auth.mfa.challenge({ factorId: factor.id });
const { error } = await supabase.auth.mfa.verify({
  factorId: factor.id,
  challengeId: chal.id,
  code,
});
// retry on error = always create a NEW challenge first
// (or replace both calls with challengeAndVerify({ factorId, code }) when enrollment is one-shot)
```

On success: the factor flips to `verified`, the session upgrades to `aal2`, and **all of the user's other sessions are logged out** — the success screen should say so (this is standard MFA enrollment behavior, not a bug).

Immediately follow the success screen with **recovery-code generation** (`mfa-recovery-codes.md`) — the backup for a lost authenticator. Do not treat as optional.

Errors worth specific copies: factor limit (max 2 active factors per user — show "remove one from settings first"), challenge exhaustion, network.

## Challenge step (sign-in)

Reached from `needsMfaChallenge()` after `signInWithPassword` (`sign-in.md`). Screen: code input (6-digit, numeric keyboard) + a "Use a recovery code instead" fallback link (`mfa-recovery-codes.md`).

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

- If more than one factor exists, offer a picker; default to the first.
- A new challenge per attempt (never reuse a `challengeId` across verifications).
- On success the client refreshes the session in the background → `SIGNED_IN` → the session provider routes into the app. Don't route before the event fires.
- TOTP codes are valid for one 30s window (+1 interval clock skew); deny nothing extra at the client.

## Factor settings (account screen)

Same `listFactors()` call lists existing factors; deletion:

```ts
const { error } = await supabase.auth.mfa.unenroll({ factorId });
```

- Unenrolling a `verified` factor requires the session to be `aal2`. If the user is at `aal1`, first run `supabase.auth.reauthenticate()` and route them through the challenge screen (`supabase.auth.mfa.getAuthenticatorAssuranceLevel()` tells you).
- The session stays `aal2` until the token refreshes — call `supabase.auth.refreshSession()` right after unenroll if the UI reflects AAL immediately.
- Always gate removal behind a confirm modal — losing the factor without recovery codes locks the user out of `aal2`.
- If everything is lost (authenticator gone, recovery codes exhausted), recovery is a **support-side admin action**: `supabaseAdmin.auth.admin.mfa.deleteFactor({ id, userId })` from a service-role context (Edge Function). Mention this path in the settings screen's error copy if you build it.
