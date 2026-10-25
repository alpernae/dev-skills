# Account screen + account deletion

The authenticated settings screen. Structure it as sections; every action defined here is optional individually — but the delete-account function is the one piece that requires extra scaffolding (an Edge Function).

## Sections

1. **Profile** — email read-only (`session.user.email`), display name / metadata editable:

```ts
await supabase.auth.updateUser({ data: { full_name: newName } }); // → USER_UPDATED event
```

2. **Change password** — current session suffices; no email:

```ts
await supabase.auth.updateUser({ password: newPwd });
```

3. **Change email**:

```ts
await supabase.auth.updateUser({ email: newEmail });
```

Supabase emails the *current* address to confirm; after the user confirms (deep link → same `handleAuthUrl`), `USER_UPDATED` carries the changed email. Copy: "We sent a confirmation email to your current address." Secure email change (confirm from old address) is the dashboard default; if the project uses code-based change templates, the method is `verifyOtp({ type: "email_change", email, token })` — check templates before building a paste-a-code step.

4. **Sessions**:

```ts
await supabase.auth.signOut({ scope: "local" });   // this device only — needs an explicit copy like "Sign out on this device"
await supabase.auth.signOut({ scope: "global" });  // all devices — copy "Sign out everywhere"
```

Confirm the semantics against the copy (`sign-in.md`); wrong labeling here is a common bug.

5. **MFA** — links to the MFA screens: enrollment, factor list/unenroll, recovery-code status/`unenroll`/`regenerate` (`mfa-totp.md`, `mfa-recovery-codes.md`).

6. **Danger zone — Delete account.**

## Delete account — Edge Function

Deleting an auth user is `supabase.auth.admin.deleteUser()` — **service-role only**. It must run inside an Edge Function; the `service_role` key is never in the app.

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

Deploy: `supabase functions deploy delete-account` (for local runs, `supabase functions serve delete-account --env-file ./supabase/.env` with `SUPABASE_SERVICE_ROLE_KEY` set there).

App call:

```ts
const { error } = await supabase.functions.invoke("delete-account");
// error → show copy; success → confirm + signOut
```

## Deletion UX rules

- Confirm modal (destructive): state exactly what is lost, require typed confirmation (e.g. "DELETE").
- Data cleanup is the project's own responsibility: reference domain tables from `auth.users` with `ON DELETE CASCADE` so their rows go with the user; storage files are **not** auto-deleted — handle inside the function if used.
- After success: `await supabase.auth.signOut({ scope: "local" })` (the earlier session may already be dead — treat the sign-out error as success), clear back-stack state, land on the sign-in/welcome screen.
- No extra features here (data export etc.) unless the user asks — keep this skill scoped to the list above.
