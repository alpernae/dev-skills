# Setup — clients, proxy, email-link endpoint, dashboard config

Docs: <https://supabase.com/docs/guides/auth/server-side/nextjs>, <https://supabase.com/docs/guides/auth/passwords>, <https://supabase.com/docs/guides/auth/redirect-urls>.

## Next.js (App Router, SSR)

**Install:**

```bash
npm install @supabase/supabase-js @supabase/ssr
```

**Env — `.env.local`** (publishable keys only; never `service_role` here):

```
NEXT_PUBLIC_SUPABASE_URL=https://<project-ref>.supabase.co
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=sb_publishable_...
```

**Browser client — `lib/supabase/client.ts`** (Client Components; singleton):

```ts
import { createBrowserClient } from "@supabase/ssr";

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY!,
  );
}
```

**Server client — `lib/supabase/server.ts`** (Server Components, Route Handlers, Server Actions; new instance per request):

```ts
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function createClient() {
  const cookieStore = await cookies();

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) => cookieStore.set(name, value, options));
          } catch {
            // called from a Server Component → can't write cookies; the proxy refreshes instead
          }
        },
      },
    },
  );
}
```

**Proxy — `proxy.ts`** (project root or `src/`). Refreshes the session on every request and gates protected paths. On Next.js ≤ 15 the same code goes in `middleware.ts` with `export async function middleware`:

```ts
import { type NextRequest, NextResponse } from "next/server";
import { createServerClient } from "@supabase/ssr";

export async function proxy(request: NextRequest) {
  let supabaseResponse = NextResponse.next({ request });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll();
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) => request.cookies.set(name, value));
          supabaseResponse = NextResponse.next({ request });
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options));
        },
      },
    },
  );

  // verifies the signature and refreshes near expiry — NEVER use getSession() in server code
  const { data } = await supabase.auth.getClaims();
  const claims = data?.claims;

  const isAuthenticated = Boolean(claims?.sub); // sub = user id
  const isProtected = ["/account", "/mfa"].some((p) => request.nextUrl.pathname.startsWith(p));

  if (!isAuthenticated && isProtected) {
    const url = request.nextUrl.clone();
    url.pathname = "/sign-in";
    url.searchParams.set("next", request.nextUrl.pathname);
    return NextResponse.redirect(url);
  }

  return supabaseResponse;
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)"],
};
```

Keep an early `return supabaseResponse` — any custom response must first copy `supabaseResponse.cookies.getAll()` and the `cache-control`/`expires`/`pragma` headers onto it, or refreshed sessions leak away.

**Email-link exchange — `app/auth/confirm/route.ts`** (required for the token_hash email templates below — works for both signup confirmation and recovery):

```ts
import { type EmailOtpType } from "@supabase/supabase-js";
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url);
  const token_hash = searchParams.get("token_hash");
  const type = searchParams.get("type") as EmailOtpType | null; // "email" | "recovery" | ...
  const next = searchParams.get("next") ?? "/";
  const redirectTo = request.nextUrl.clone();
  redirectTo.pathname = next;

  if (token_hash && type) {
    const supabase = await createClient();
    const { error } = await supabase.auth.verifyOtp({ type, token_hash });
    if (!error) {
      return NextResponse.redirect(redirectTo);
    }
  }

  redirectTo.pathname = "/auth/auth-code-error"; // error page with instructions + "resend" link
  return NextResponse.redirect(redirectTo);
}
```

**Email templates** (`Auth → Templates`) — replace the template HTML for *Confirm signup* and *Reset password* with the token-hash form (docs: <https://supabase.com/docs/guides/auth/passwords>):

```html
<!-- Confirm signup -->
<h2>Confirm your email address</h2>
<p>Follow the link below to confirm this email address and finish signing up.</p>
<a href="{{ .SiteURL }}/auth/confirm?token_hash={{ .TokenHash }}&type=email&next={{ .RedirectTo }}">Confirm email address</a>
```

```html
<!-- Reset password -->
<p>We received a request to reset your password. Follow the link below to choose a new one.</p>
<a href="{{ .SiteURL }}/auth/confirm?token_hash={{ .TokenHash }}&type=recovery&next={{ .RedirectTo }}">Reset password</a>
```

**Dashboard → Auth → URL Configuration**: Site URL = app origin; redirect URLs include the `next` targets (e.g. `http://localhost:3000/account/update-password`) and `http://localhost:3000/auth/confirm/**` — or simply `http://localhost:3000/**` while developing.

## Client-only React (Vite, CRA, SPA)

**Install:**

```bash
npm install @supabase/supabase-js
```

**Env:** Vite → `.env` with `VITE_SUPABASE_URL`, `VITE_SUPABASE_PUBLISHABLE_KEY` (`import.meta.env`); CRA → `REACT_APP_SUPABASE_URL`, … (`process.env`).

**Client — `lib/supabase.ts`:**

```ts
import { createClient } from "@supabase/supabase-js";

export const supabase = createClient(
  import.meta.env.VITE_SUPABASE_URL!,
  import.meta.env.VITE_SUPABASE_PUBLISHABLE_KEY!,
  // auth defaults are fine for SPAs: localStorage-backed session, PKCE-implicit flows, automatic URL detection
);
```

No proxy, no cookie adapter, no exchange endpoint. With the **default email templates**, confirmation/recovery links carry tokens in the URL fragment and the library detects them on page load (`detectSessionInUrl` defaults to true) — pages pick the result up via `onAuthStateChange` (`SIGNED_IN`, `PASSWORD_RECOVERY`). Redirects like `emailRedirectTo: \`${window.location.origin}/reset-password\`` are what you pass in calls.

If the project deliberately adopts the token_hash template pattern above, add a client-side `/auth/confirm` page that parses `token_hash`/`type`/`next` from `location.search` and calls `supabase.auth.verifyOtp({ type, token_hash })` before navigating to `next` — same contract, client-side. Keep one pattern; don't mix templates.
