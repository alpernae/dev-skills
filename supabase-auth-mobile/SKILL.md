---
name: supabase-auth-mobile
description: Supabase Auth implementation for Expo/React Native UIs. Wire the complete account layer into whatever app is being built — sign-up with email confirmation, sign-in with email+password, MFA authenticator (TOTP) with one-time recovery codes, password reset, session gating, account settings and account deletion (Edge Function) — including supabase-js client setup, deep-link configuration and error handling. Use whenever a UI build (alone or together with a UI/UX skill) involves Supabase or user accounts — sign in, sign up, 2FA/MFA, recovery codes, email confirmation, password reset, delete account — so the auth screens and logic are generated without extra prompting.
version: 1.0.0
license: MIT
---

# Supabase Auth for Mobile Builds

The account layer for Expo/React Native apps backed by Supabase Auth. When this skill is active (usually together with a UI/UX skill), implement the auth features below **without asking whether to include them** — the user invoked this skill precisely to get them wired into the screens. Only ask for what is genuinely missing: the Supabase **Project URL** and the **publishable key** (`sb_publishable_*` or legacy anon key).

For navigation, styling, components, and screen design, follow the UI skill (e.g. `expo-design-system`, `frontend-design`, `expo-router`). This skill owns everything auth: the client, session gating, deep links, flow logic, and error mapping. Domain features beyond auth (data tables, storage, payments) are out of scope. For React/Next.js **web** apps use the sibling skill `supabase-auth-web` instead — the auth APIs are identical but client setup, link handling, and screen wiring differ.

## References

Consult these for each feature; they contain the exact API calls and edge cases:

```
references/
  setup.md               supabase-js client on RN, env keys, deep-link handler, dashboard redirect config
  sign-up.md             signUp + email confirmation (check-inbox state, resend, verification complete)
  sign-in.md             signInWithPassword, error mapping, session provider, AAL gate after sign-in
  mfa-totp.md            authenticator enrollment (QR/secret + verify), challenge at sign-in, factor settings
  mfa-recovery-codes.md  recovery-code API (experimental): generate, status, regenerate, sign in with a code
  password-reset.md      forgot-password + reset link → update-password screen
  account-management.md  account screen: profile, change password/email, sign-out scopes, delete account
  screens.md             full screen/state inventory to hand to the UI skill
```

## Build order

1. **Setup first** (`setup.md`): client init, env vars, deep-link handler, app scheme, dashboard redirect URLs. Auth screens do not work until redirect URLs are registered on the dashboard.
2. **Session provider + route gate** (`sign-in.md`): one source of truth for the session; gate protected screens behind it.
3. **Sign-up + sign-in** (`sign-up.md`, `sign-in.md`): the two entry screens, including the email-confirmation check-inbox state.
4. **MFA** (`mfa-totp.md`, `mfa-recovery-codes.md`): enrollment flow, challenge step after sign-in, factor management screens, recovery codes.
5. **Password reset** (`password-reset.md`): forgot-password → reset link → new-password screen.
6. **Account screen** (`account-management.md`): profile, change password/email, sign-out, delete account (Edge Function).
7. **Hand off screens** to the UI skill using `screens.md`. Keep auth logic in `lib/` and hooks — screens stay mostly declarative so the UI skill can style them freely.

## MFA policy (pick one; default opt-in)

1. **Mandatory for all** — after sign-in, an `aal1→aal2` gap always routes to the challenge screen; enrollment is forced at first sign-in. Enforce in RLS: `create policy "..." on table as restrictive to authenticated using ((select auth.jwt()->>'aal') = 'aal2');`
2. **Mandatory for new users only** — same, but only for `created_at` after a cutoff date.
3. **Opt-in (default)** — enrollment is offered on settings (and optionally right after first sign-in); existing non-MFA users keep working at `aal1`. No RLS change needed.

The frontend detection for all modes is `getAuthenticatorAssuranceLevel()` — see `sign-in.md`.

## Dashboard checklist (before testing any email flow)

- **Auth → URL Configuration**: set *Site URL* and add *Additional Redirect URLs* for the app scheme(s), e.g. `com.yourname.yourapp://**` and, while developing in Expo Go, `exp://**`. Without these, confirmation/reset links never open the app.
- **Auth → Providers → Email**: *Confirm email* is ON by default — the flows below assume it. You may turn it off for quick styling loops, but finish the build with it back ON.
- **Auth → Providers → Email**: min password length is 6 by default — mirror it in client-side validation.
- **Project Settings → API keys**: copy the publishable key (`sb_publishable_*`; the legacy anon key also works).
- **Email send limits**: confirmations and resets are rate-limited per user/email. Every "resend" button gets a 60s cooldown.
- **Edge Functions secrets** are auto-injected inside hosted functions (`SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`).

## Hard rules

- **Never ship the `service_role` key in the app.** Only publishable keys go into `EXPO_PUBLIC_*` env vars. Anything admin-level lives in an Edge Function.
- **Use the documented supabase-js methods in these references** — never hand-roll fetches against `/auth/v1/*` endpoints, never parse/construct tokens manually.
- **Never persist tokens yourself**; the client manages session storage (AsyncStorage or the encrypted variant in `setup.md`).
- **All dashboard settings that a flow depends on (redirect URLs, email confirm toggle, key names) must be stated** when handing the build to the user — a flow that can only work after a dashboard change gets a note.
- **Await every Supabase call**; map errors with the tables in the references; use generic copies ("If an account exists for this email…") for emails so the API cannot be used to enumerate users.
- **Don't poll** (`setInterval` on `getSession()`); drive state from `onAuthStateChange` events.
- **Don't invent API surface.** If something isn't in the references or supabase-js types, check the docs before writing it.
