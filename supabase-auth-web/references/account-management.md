# Account page + account deletion

The authenticated settings page. Structure it as sections; every action is optional individually — the delete-account function is the piece that requires extra scaffolding (an Edge Function).

## Sections

1. **Profile** — email read-only (`session.user.email`), display name / metadata editable:

```ts
await supabase.auth.updateUser({ data: { full_name: newName } }); // → USER_UPDATED event
```

2. **Change password** — no email required while signed in. On `supabase-js ≥ 2.102` verify the old one server-side first:

```ts
await supabase.auth.updateUser({ password: newPwd, current_password: currentPwd });
```

3. **Change email**:

```ts
await supabase.auth.updateUser({ email: newEmail });
```

Supabase emails the *current* address to confirm; the deep link routes through `/auth/confirm` (Next.js) or straight into the app (client-only default templates); `USER_UPDATED` carries the change. Copy: "We sent a confirmation email to your current address." If the project uses code-based change templates, the call is `verifyOtp({ type: "email_change", email, token })` — check templates first.

4. **Sessions**:

```ts
await supabase.auth.signOut({ scope: "local" });   // this browser — copy "Sign out on this device"
await supabase.auth.signOut({ scope: "global" });  // all devices — copy "Sign out everywhere"
```

The JS default is `global`; label buttons accordingly (`sign-in.md`). Next.js: `router.refresh()` after either.

5. **MFA** — links to the MFA pages: enrollment, factor list/unenroll, recovery-code status/`regenerate`/`unenroll` (`mfa-totp.md`, `mfa-recovery-codes.md`).

6. **Danger zone — Delete account.**

## Delete account — Edge Function

Deleting an auth user is `supabase.auth.admin.deleteUser()` — **service-role only**. It runs inside an Edge Function; the `service_role` key is never in the app (even Next.js env — a `NEXT_PUBLIC_` var would expose it).

```ts
// supabase/functions/delete-account/index.ts
import { createClient } from "npm:@supabase/supabase-js@2";

const json = (status: number, body: unknown) =>
  new Response(JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } });

Deno.serve(async (req) => {
  try {
    const supabase = createClient(
      Deno.env.get("SUPABASE_URL")!,
      Deno.env.get("SUPABASE_ANON_KEY")!,
    );

    // supabase.functions.invoke attaches the user's JWT automatically — verify it server-side
    const token = req.headers.get("Authorization")?.replace(/^Bearer /i, "");
    if (!token) return json(401, { error: "No authorization header" });

    const { data: claims, error: verifyErr } = await supabase.auth.getUser(token);
    if (verifyErr || !claims?.user) return json(401, { error: "Invalid token" });

    const admin = createClient(
      Deno.env.get("SUPABASE_URL")!,
      Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!, // auto-injected in hosted Edge Functions
      { auth: { autoRefreshToken: false, persistSession: false } },
    );

    // delete storage objects owned by the user first if applicable

    const { error } = await admin.auth.admin.deleteUser(claims.user.id); // 2nd arg true = soft-delete
    if (error) return json(400, { error: error.message });
    return json(200, { ok: true });
  } catch (e) {
    return json(400, { error: e instanceof Error ? e.message : String(e) });
  }
});
```

Deploy: `supabase functions deploy delete-account` (local runs: `supabase functions serve delete-account --env-file ./supabase/.env` with `SUPABASE_SERVICE_ROLE_KEY` set there).

App call (either stack, from the client):

```ts
const { error } = await supabase.functions.invoke("delete-account");
```

Next.js alternative: the same logic can live in a Server Action / Route Handler using `process.env.SUPABASE_SERVICE_ROLE_KEY` (never `NEXT_PUBLIC_`) — an Edge Function stays the canonical cross-platform choice; pick one and say so in the handoff.

## Deletion UX rules

- Destructive confirm modal: state exactly what is lost; require typed confirmation (e.g. "DELETE").
- Data cleanup is the project's own responsibility: reference domain tables from `auth.users` with `ON DELETE CASCADE` so their rows go with the user; storage files are **not** auto-deleted — handle inside the function if used.
- After success: `await supabase.auth.signOut({ scope: "local" })` (the old session may already be dead — treat the sign-out error as success), clear caches/state, land on the sign-in/welcome page (client-only React) or `router.replace("/sign-in")` + refresh (Next.js).
- No extra features (data export etc.) unless the user asks — keep this skill scoped.
