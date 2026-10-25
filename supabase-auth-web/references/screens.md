# Pages — inventory & states

The handoff contract between this skill and the UI skill. Paths work for both stacks (Next.js App Router pages; React Router/Vite routes). Auth logic lives in `lib/` and hooks/pages stay mostly declarative for the UI skill to style.

## Page inventory

| Path | Page | Implements (reference) |
| --- | --- | --- |
| `/sign-in` | Sign in | `signInWithPassword`, error table, links to sign-up + forgot (sign-in.md) |
| `/sign-up` | Sign up | `signUp` + `emailRedirectTo`, validation (sign-up.md) |
| `/confirm-email` | Check your inbox | resend with 60s cooldown, pending state (sign-up.md) |
| `/mfa/challenge` | MFA code | TOTP 6-digit, "use a recovery code" fallback (mfa-totp.md, mfa-recovery-codes.md) |
| `/mfa/enroll` | Set up authenticator | enroll → QR/manual → verify → success → recovery codes (mfa-totp.md, mfa-recovery-codes.md) |
| `/mfa/settings` | Manage MFA | factor list + unenroll, recovery status/regenerate (mfa-totp.md, mfa-recovery-codes.md) |
| `/forgot-password` | Forgot password | `resetPasswordForEmail`, generic success copy (password-reset.md) |
| `/account/update-password` (Next) / `/reset-password` (client-only) | Set new password | `updateUser({ password })`, expired-link state (password-reset.md) |
| `/account` | Account | profile, change password/email, sign-out scopes, MFA links, delete (account-management.md) |
| `/auth/confirm` (Next) | — infra | token-exchange Route Handler (`setup.md`) — never a styled page, but it needs the `/auth/auth-code-error` error page visible to users |

Infrastructure, not user-facing styling targets: `lib/supabase/*`, proxy/`middleware`, the session provider. Web-specific styling frameworks are the UI skill's domain entirely.

## State matrix

Every page implements all of these unless noted; include them in the handoff so the UI doesn't have to invent them:

- **idle** — default form state
- **submitting** — button-level spinner; disable submits while pending; await carefully (double-click = double sign-up attempt)
- **error** — mapped inline copy from the reference tables; never console-only errors
- **success** — exactly what each reference's flow says happens next

Page-specific states worth designing explicitly:

- `/sign-in`: field-level validation, "email not confirmed" branch (→ `/confirm-email`), `?next=` awareness (post-sign-in route target), MFA-needed branch (→ `/mfa/challenge`)
- `/confirm-email`: cooldown countdown on resend, "check spam" hint, pending banner on app reopen
- `/mfa/challenge`: two tabs (TOTP code | recovery code), remaining-count hint if status is available, error state on bad code
- `/mfa/enroll`: scan / manual-key / verify / success steps; success includes the once-only recovery-code list + copy-all + "I saved them" gate
- `/mfa/settings`: factor rows with friendly name + created date, removal confirm modal, "codes unused/total" row, regenerate confirm ("old codes stop working")
- `/account/update-password` & `/reset-password`: idle/submitting/success/expired-link (expired → resend path)
- `/account`: destructive modal with typed confirmation

## Input affordances (tell the UI skill)

- email: `type="email"` + `autoComplete="email"`, lowercase/trimmed
- passwords: `type="password"` with show/hide toggle; `autoComplete="current-password"` (sign-in, update-password) or `"new-password"` (sign-up, update-password); min-length hint from the dashboard value (6 default)
- TOTP code: 6 digits, `inputMode="numeric"` + `autoComplete="one-time-code"`, auto-advance/focus segments if styled that way (value stays one 6-digit string)
- recovery-code input: text field tolerant of dashes/spaces — normalization happens in `lib/`, not the input
- all submits: disable on pending, `aria-busy` on spinners, labels on every icon button

## Copy baseline (generic, anti-enumeration)

- sign-up/inbox: "We sent a confirmation link to {email}."
- forgot: "If an account exists for this email, you'll receive a reset link."
- wrong credentials: "Wrong email or password."
- locked mfa: "Can't access your authenticator? Use a recovery code."
- delete account: explicit data-loss sentence + type-to-confirm word

Fit copy to the brand voice; never change its *informational content* (e.g. never claim "we checked your email doesn't exist").
