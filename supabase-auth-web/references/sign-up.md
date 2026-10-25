# Sign-up + email confirmation

Docs: <https://supabase.com/docs/guides/auth/passwords#signing-up-with-an-email-and-password>, <https://supabase.com/docs/reference/javascript/auth-resend>.

Flow (Confirm email ON — the default): form → `signUp` → **check-your-inbox** page → user clicks the email link → token exchange → session lands (cookies or browser URL) → into the app.

## Call (both stacks; the redirect target differs)

```ts
// Next.js SSR: value is the LANDING page (the /auth/confirm endpoint is baked into the template)
const emailRedirectTo = `${window.location.origin}/account`;

// Client-only with default templates: the value IS where tokens arrive — plain landing URL
const emailRedirectTo = window.location.origin;

const { data, error } = await supabase.auth.signUp({
  email,
  password,
  options: {
    emailRedirectTo,
    data: { full_name: fullName }, // lands in user_metadata; feed profile tables from it
  },
});
```

Validate before calling: trim/downcase email, password ≥ 6 chars (mirror the dashboard min length), confirm-password match. The values passed to `emailRedirectTo` must be on the dashboard's redirect allowlist (`setup.md`).

## Result states (same in both stacks)

- `error` set → map the copy:
  - `"User already registered"` (only when user-enumeration protection is disabled) → offer the Sign-in page.
  - otherwise → generic copy.
- No error and **`data.session` is present** → confirmation is disabled: user is signed in — continue into the app.
- No error and **`data.session` is null** → confirmation required → check-your-inbox state. This is also correct under enumeration protection (results deliberately look identical for existing emails); never branch on "did the email exist".

## Check-your-inbox page

- Copy: "We sent a confirmation link to {email}".
- **Resend** with a 60s cooldown after every successful send (hosted built-in SMTP is limited to ~2 emails/hour; a custom SMTP makes testing pleasant):

```ts
const { error } = await supabase.auth.resend({
  type: "signup",
  email,
  options: { emailRedirectTo }, // same target as signUp
});
```

- **Next.js**: the link goes to `/auth/confirm` → `verifyOtp({ type: "email", token_hash })` → cookie session → redirect to `next`. Full page reload handles the session; Server Components re-render authenticated. Remember `/auth/auth-code-error` page for expired/failed links (show "The link expired — resend from the sign-up page").
- **Client-only (default templates)**: the link lands on `emailRedirectTo` with tokens in the fragment; the library parses them on load and fires `SIGNED_IN`. A tiny landing page ("Verifying…") that waits for the event or calls `getSession()` once before routing is fine; response errors (expired link, already-confirmed) can render the resend UI.
- Persist nothing user-visible except the pending email.

## After first verified sign-in

Natural places to offer MFA enrollment per the policy in `SKILL.md`: right after confirmation (post-signup onboarding) or from the account page — covered in `mfa-totp.md`.
