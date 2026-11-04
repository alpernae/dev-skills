# Screens — inventory & states

The handoff contract between this skill and the UI skill. Route paths are suggestions; adapt to the project's router conventions (expo-router files, drawers, etc.). Auth logic stays in `lib/` and hooks — each screen consumes it and stays mostly declarative.

## Screen inventory

| Route | Screen | Implements (reference) |
| --- | --- | --- |
| `/sign-in` | Sign in | `signInWithPassword`, error table, links to sign-up + forgot password (sign-in.md) |
| `/sign-up` | Sign up | `signUp` with `emailRedirectTo`, validation (sign-up.md) |
| `/confirm-email` | Check your inbox | resend with 60s cooldown, pending state across restarts (sign-up.md) |
| `/mfa/challenge` | MFA code | TOTP 6-digit entry, "use a recovery code" fallback (mfa-totp.md, mfa-recovery-codes.md) |
| `/mfa/enroll` | Set up authenticator | enroll → QR/manual → verify → success → recovery codes (mfa-totp.md, mfa-recovery-codes.md) |
| `/mfa/settings` | Manage MFA | factor list + unenroll, recovery-status/regenerate (mfa-totp.md, mfa-recovery-codes.md) |
| `/forgot-password` | Forgot password | `resetPasswordForEmail`, generic success copy (password-reset.md) |
| `/reset-password` | Set new password | deep-link target, `updateUser({ password })`, expired-link state (password-reset.md) |
| `/account` | Account | profile, change password/email, sign-out scopes, MFA links, delete (account-management.md) |
| (modal) | Delete account | destructive confirm, type-to-"DELETE" (account-management.md) |

Non-screen infrastructure (not user-facing styling targets): session provider + route gate, deep-link handler, `lib/supabase.ts`. If the project uses expo-router, structural specifics come from the `expo-router` skill.

## State matrix

Every screen implements all of these unless noted; show them in the handoff so the UI doesn't have to invent them:

- **idle** — default form state
- **submitting** — button-level spinner; disable submits while pending; await carefully (double-tap = double account-creation)
- **error** — mapped inline copy from the reference tables; never leave errors only as console logs
- **success** — exactly what each reference's flow says happens next (navigate, or a persistence-dependent copy)

Screen-specific states worth designing explicitly:

- `/sign-in`: field-level validation, "email not confirmed" branch (promote to `/confirm-email`), generic-safety copy
- `/confirm-email`: cooldown countdown on resend, "if it didn't arrive check spam", pending-email banner on app reopen
- `/mfa/challenge`: two tabs (TOTP code | recovery code), remaining-count hint if status is available, error-shake on bad code
- `/mfa/enroll`: scan / manual-key / verify / success steps; success includes the once-only recovery-code list + copy-all + "I saved them" gate
- `/mfa/settings`: factor rows with created date, removal confirm modal, "codes unused/total" row, regenerate confirm ("old codes stop working")
- `/reset-password`: idle/submitting/success/expired-link (expired → resend path)
- `/account`: destructive modal with typed confirmation

## Input affordances (tell the UI skill)

- email fields: lowercase, trimmed, keyboard email type
- password fields: show/hide toggle; min length hint from the dashboard value (6 by default)
- TOTP code input: 6 digits, numeric keypad, auto-advance/focus segments if styled that way (value stays a 6-digit string)
- recovery-code input: text field tolerant of dashes/spaces — normalization happens in `lib/`, not the input
- all submits: keyboard-safe (scroll until inputs are visible), a11y labels on icon buttons

## Copy baseline (generic, anti-enumeration)

- sign-up/inbox: "We sent a confirmation link to {email}."
- forgot: "If an account exists for this email, you'll receive a reset link."
- wrong credentials: "Wrong email or password."
- locked mfa: "Can't access your authenticator? Use a recovery code."
- delete account: explicit data-loss sentence + type-to-confirm word

Fit copy to the brand voice; never change its *informational content* (e.g. never claim "we checked your email doesn't exist").
