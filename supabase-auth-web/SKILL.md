---
name: supabase-auth-web
description: Supabase Auth implementation for React web apps — plain React (Vite/CRA) and Next.js with @supabase/ssr. Wire the complete account layer into whatever app is being built — sign-up with email confirmation, sign-in with email+password, MFA authenticator (TOTP) with one-time recovery codes, password reset, session gating, and account deletion (Edge Function) — including client setup (cookie-based SSR vs client-only), Next.js proxy/middleware session refresh, the /auth/confirm token-exchange endpoint, email template configuration, redirect URL allowlist, and error handling. Use whenever a UI build (alone or together with a UI/UX skill) involves Supabase or user accounts — sign in, sign up, 2FA/MFA, recovery codes, email confirmation, password reset, delete account — so the auth screens and logic are generated without extra prompting.
version: 1.0.0
license: MIT
---

# Supabase Auth for Web Builds

The account layer for React web apps backed by Supabase Auth. When this skill is active (usually together with a UI/UX skill), implement the auth features below **without asking whether to include them** — the user invoked this skill precisely to get them wired into the pages. Only ask for what is genuinely missing: the Supabase **Project URL** and the **publishable key**, and — only if the stack can't be detected from the project (e.g. empty repo) — whether it's Next.js (SSR) or client-only React.

Two supported stacks, detected from the project:

1. **Next.js (App Router)** — cookie-based sessions via `@supabase/ssr`, `proxy.ts`/`middleware.ts` session refresh, `/auth/confirm` route handler for email links. The auth APIs below get called from Client Components browsing (`'use client'`), pages protected in the proxy.
2. **Client-only React (Vite, CRA, SPA)** — plain `@supabase/supabase-js` with browser storage; email links are handled automatically on page load (`detectSessionInUrl`); routing/redirect decisions client-side.

Auth APIs (`signUp`, `signInWithPassword`, `mfa.*`) are identical in both — stack differences are confined to `setup.md` and the link-handling notes. The same APIs on mobile with Expo live in the sibling skill `supabase-auth`.

## References

```
references/
  setup.md               clients per stack, env keys, proxy/middleware, /auth/confirm, email templates, dashboard config
  sign-up.md             signUp + email confirmation (check-inbox page, resend, verification complete)
  sign-in.md             signInWithPassword, error mapping, getClaims/getUser/getSession policy, AAL gate
  mfa-totp.md            authenticator enrollment (QR/secret + verify), challenge step, factor settings
  mfa-recovery-codes.md  recovery-code API (experimental): generate, status, regenerate, sign in with a code
  password-reset.md      forgot-password + reset link → update-password page (both stacks)
  account-management.md  account page: profile, change password/email, sign-out scopes, delete account
  screens.md             page/state inventory to hand to the UI skill
```

## Build order

1. **Setup first** (`setup.md`): clients, env, proxy (Next.js) or SPA client (client-only), email templates + redirect allowlist. No email flow works until the redirect URLs and (Next.js) the `/auth/confirm` endpoint exist.
2. **Session gate** (`sign-in.md`): proxy-based gating (Next.js) or session provider (client-only).
3. **Sign-up + sign-in** pages (`sign-up.md`, `sign-in.md`), including the check-your-inbox state.
4. **MFA** (`mfa-totp.md`, `mfa-recovery-codes.md`): enrollment, challenge step, settings pages, recovery codes.
5. **Password reset** (`password-reset.md`).
6. **Account page** (`account-management.md`), including delete-account Edge Function.
7. **Hand off pages** to the UI skill using `screens.md`. Auth logic lives in `lib/`/hooks; pages stay mostly declarative for the UI skill to style.

## MFA policy (pick one; default opt-in)

1. **Mandatory for all** — after sign-in, an `aal1→aal2` gap always routes to the challenge page; additionally enforce in RLS: `create policy "..." on table as restrictive to authenticated using ((select auth.jwt()->>'aal') = 'aal2');` and — for Next.js — check `claims.aal` in the proxy and redirect to the challenge page (per the docs: redirect, don't 401).
2. **Mandatory for new users only** — same, with a `created_at` cutoff (DB-side policy in the MFA guide).
3. **Opt-in (default)** — enrollment offered on the account page (and optionally after first sign-in); non-MFA users keep working at `aal1`.

Frontend detection for all modes: `getAuthenticatorAssuranceLevel()` — see `sign-in.md`.

## Dashboard checklist (before testing any email flow)

- **Auth → URL Configuration**: *Site URL* should be the app origin (e.g. `http://localhost:3000`); *Additional Redirect URLs* must include every URL the flows redirect through — for Next.js that means `<origin>/auth/confirm/**` and the `next` destination paths; simplest safe dev setup is `<origin>/**` plus `http://localhost:3000/**`. Without entries here, confirmation/reset links dead-end.
- **Email templates** (`Auth → Templates`): the Next.js SSR pattern requires switching *Confirm signup* and *Reset password* templates to the `token_hash` form — exact HTML is in `setup.md`. Client-only apps may keep the default templates.
- **Auth → Providers → Email**: *Confirm email* is ON by default (hosted projects). Min password length is 6 by default — mirror it in client-side validation.
- **Built-in SMTP rate limit is harsh**: ~2 emails/hour on the shared sender. Testing resend flows against it will 429 quickly — either set up custom SMTP now, or test with the CLI's local Mailpit (`supabase status` shows the URL). Every "resend" button still gets a 60s cooldown.
- **Project Settings → API keys**: publishable key (`sb_publishable_*`; legacy anon also works).
- **Edge Functions secrets** (`SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`) are auto-injected in hosted functions.

## Hard rules

- **`service_role` key stays server-side only**: Edge Functions or Next.js server env (`process.env.SUPABASE_SERVICE_ROLE_KEY`, **never `NEXT_PUBLIC_`**). Anything admin-level (delete user, admin factor deletion) runs behind an authenticated server route.
- **Inside Next.js server code (proxy, server components) never trust `getSession()`** — the session cookie can be forged. Verify with `supabase.auth.getClaims()` (documented default: verifies signature, refreshes near expiry). Client code may rely on `getSession()` for initial UI state only.
- **Use the documented supabase-js methods in these references** — never hand-roll fetches against `/auth/v1/*`, never parse/construct tokens manually.
- **State the flow you picked in the handoff**: SSR-cookie (token_hash + `/auth/confirm`) vs client-only default — a login that works only after a dashboard/template change gets an explicit note.
- **Await every Supabase call**; map errors with the tables in the references; keep email copies generic ("If an account exists for this email…") so the API can't be used to enumerate users.
- **Don't poll** with `setInterval`; the browser client refreshes on visibility focus, the proxy covers SSR. Use `onAuthStateChange` for UI state.
- **Don't invent API surface.** If something isn't in the references or supabase-js types, check the docs first.
