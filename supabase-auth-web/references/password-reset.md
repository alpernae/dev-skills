# Password reset

Docs: <https://supabase.com/docs/guides/auth/passwords#resetting-a-password>, <https://supabase.com/docs/reference/javascript/auth-resetpasswordforemail>.

The handler pattern differs by stack; the security posture is identical (see the docs note: `resetPasswordForEmail()` deliberately doesn't reveal whether an account exists — no email, no error for unknown addresses).

## Next.js (SSR — token_hash pattern, template already in `setup.md`)

**Forgot-password page** (public):

```ts
const { error } = await supabase.auth.resetPasswordForEmail(email, {
  redirectTo: `${window.location.origin}/account/update-password`, // the ?next= destination
});
```

- On success (or without an explicit error): generic copy — "If an account exists for this email, you'll receive a reset link." Rate-limited (429): "Too many requests — try again in a minute." 60s cooldown on resend. Remember built-in SMTP ≈ 2 emails/hour.
- The email link goes to `/auth/confirm?...&type=recovery&next=...` → the route handler exchanges → cookie session → redirect to `/account/update-password`.
- **Update-password page** (client component; proxy-protected so only an exchanged session reaches it): new password + confirm, client-side check ≥ 6 chars, then:

```ts
await supabase.auth.updateUser({ password: newPwd });
```

- Success: confirmation copy, then into the app (`router.replace` + `router.refresh()`). Optional hardening: `await supabase.auth.signOut({ scope: "others" })`.
- Expired/invalid link → `/auth/confirm` redirects to `/auth/auth-code-error` — that page renders "The link expired. Send yourself a new one" + a resend button pointing back to `/forgot-password`.

## Client-only React (default templates, implicit flow)

**Forgot-password page** (public):

```ts
const { error } = await supabase.auth.resetPasswordForEmail(email, {
  redirectTo: `${window.location.origin}/reset-password`,
});
```

Same copy rules as above. The link lands on `/reset-password` with tokens in the fragment; the library parses them on page load and fires `PASSWORD_RECOVERY` — the reset page's provider routes straight into the new-password form:

```tsx
supabase.auth.onAuthStateChange((event) => {
  if (event === "PASSWORD_RECOVERY") setShowNewPasswordForm(true);
});

// form submit
await supabase.auth.updateUser({ password: newPwd });
```

- Success → confirmation → app. Optional `signOut({ scope: "others" })`.
- **Expired link state**: the URL visit fails to produce a session — after the initial event settle (`getSession === null` and no `PASSWORD_RECOVERY` within the load), render "This link expired. Send yourself a new one" + resend path. Never leave a non-functional form on screen.
- If the project adopted the token_hash pattern (per `setup.md`), the SPA gets a client `/auth/confirm` page (starts with `verifyOtp({ type, token_hash })`) and both branches behave identically after that.

## Notes

- Recovery sessions are `aal1`; the AAL gate may route into the MFA challenge before the update-password page — handled by the normal sign-in gate, not special-cased.
- Recovery links expire (default 1 hour, dashboard-configurable).
- The same `updateUser({ password })` powers "Change password" from the account page while signed in — no email required (`account-management.md`); on `supabase-js ≥ 2.102` you can also require the old one server-verified: `updateUser({ password, current_password })`.
