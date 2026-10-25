# Password reset

Docs: <https://supabase.com/docs/reference/javascript/auth-resetpasswordforemail>, <https://supabase.com/docs/reference/javascript/auth-updateuser>.

Flow: sign-in screen "Forgot password?" → forgot screen → `resetPasswordForEmail` → check-inbox copy → user clicks the →email link → deep link opens the app → `reset-password` route → new-password form → `updateUser({ password })` → done.

## Forgot-password screen

```ts
import * as Linking from "expo-linking";

const { error } = await supabase.auth.resetPasswordForEmail(email, {
  redirectTo: Linking.createURL("/reset-password"), // must be on the dashboard allowlist
});
```

- On success (or without an explicit error): show **generic** copy — "If an account exists for this email, you'll receive a reset link." Never reveal whether the email is registered.
- Rate-limited (429): show inline "Too many requests — try again in a minute." Keep a 60s cooldown on the resend button like `sign-up.md`.
- Recovery links expire (default 1 hour; dashboard configurable) — the expired-link state belongs to the next screen.

## Update-password screen (the `/reset-password` deep-link target)

The deep-link handler from `setup.md` exchanges the code in the URL and produces a session. You have two equivalent triggers — **pick one pattern per app** (and state which in the handoff):

1. The deep link targets `/reset-password`, so presenting the screen is the route-based trigger; or
2. The session provider sees `PASSWORD_RECOVERY` on `onAuthStateChange` and navigates programmatically:

```tsx
supabase.auth.onAuthStateChange((event) => {
  if (event === "PASSWORD_RECOVERY") {
    // navigate to /reset-password
  }
});
```

The form: new password + confirm, client-side validation ≥ 6 chars (mirror the dashboard minimum), then:

```ts
const { error } = await supabase.auth.updateUser({ password });
```

- Success: confirmation copy ("Your password has been updated."), then continue into the app. The AAL gate may route into the MFA challenge first (see `sign-in.md`) — handle that gracefully; don't sign the user out.
- Optional post-reset hardening: `await supabase.auth.signOut({ scope: "others" })` to drop sessions on other devices (keep the current one).
- **Expired/invalid link**: the route shows but the exchange in `handleAuthUrl` fails, so no session exists. Detect it (session is null after handling the link) and render the recovery-state UI: "This link expired. Send yourself a new one" + a button that fires `resetPasswordForEmail` again. Never silently leave the user stuck on a non-functional form.

## Notes

- The same `updateUser({ password })` call powers "Change password" from the account screen (no email required while signed in) — see `account-management.md`; don't route change-password through the email-verification flow.
- Recovery sessions are `aal1`. If MFA is enrolled, the sign-in-style challenge is required before sensitive screens — the AAL gate already covers this if the provider checks it consistently.
- If the dashboard disables *Confirm email*, reset links still work only via the user's address; keep the flow but adjust copy tests accordingly.
