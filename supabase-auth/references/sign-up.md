# Sign-up + email confirmation

Docs: <https://supabase.com/docs/reference/javascript/auth-signup>, <https://supabase.com/docs/reference/javascript/auth-resend>.

Flow (Confirm email ON — the default): form → `signUp` → **check-your-inbox** screen → user clicks the email link → deep link (`setup.md` handler exchanges it) → `SIGNED_IN` event → route into the app (or straight into optional MFA enrollment, `mfa-totp.md`).

## Call

```ts
import * as Linking from "expo-linking";

const link = Linking.createURL("/confirm-email");

const { data, error } = await supabase.auth.signUp({
  email,
  password,
  options: {
    emailRedirectTo: link,        // must match the dashboard redirect allowlist
    data: { full_name: fullName } // lands in user_metadata; feed profile tables from it
  },
});
```

Validate before calling: trim/downcase email, password ≥ 6 chars (mirror the dashboard min length), confirm-password match if the form has one.

## Result states

- `error` set → map the copy:
  - `"User already registered"` (only happens when the project has user-enumeration protection disabled) → offer the Sign-in screen.
  - otherwise → generic copy.
- No error and **`data.session` is present** → confirmation is disabled (or instant): the user is signed in — continue into the app.
- No error and **`data.session` is null** → confirmation required → show the check-inbox state. This is also the correct behavior under enumeration protection (results deliberately look identical for existing emails), so never branch on "did the email exist".

## Check-your-inbox screen

- Copy: "We sent a confirmation link to {email}." Show the a11y-safe email in plain text.
- **Resend** with a 60s cooldown after every successful send (rate limits apply):

```ts
const { error } = await supabase.auth.resend({
  type: "signup",
  email,
  options: { emailRedirectTo: link },
});
```

- The completion is automatic through the deep-link handler — do **not** build a "paste the code/OTP from the email" step unless the project deliberately switched its templates to code-based (then the template vars are `{{ .TokenHash }}` / `{{ .Token }}` and you verify with `verifyOtp({ type: "signup", email, token })`; confirm the direction with the user first).
- Persist the pending email so the state survives an app restart while the user checks their mail.
- When `SIGNED_IN` fires (session provider, `sign-in.md`), navigate into the app; if your MFA policy enrolls new users right away, land there first.

## After first verified sign-in

Natural places to offer MFA enrollment per the policy in `SKILL.md`: immediately after confirmation (post-signup onboarding) or from the account screen. Both are covered in `mfa-totp.md`.
