# Setup — client, keys, deep links

Do this before any screen. Docs: <https://supabase.com/docs/reference/javascript/initializing>, <https://supabase.com/docs/guides/auth/native-mobile-deep-linking>.

## Install

```bash
npx expo install @supabase/supabase-js @react-native-async-storage/async-storage expo-linking expo-auth-session
```

- `expo-auth-session` provides the `QueryParams` helper used to parse auth deep-link URLs.
- For the MFA QR code you need `react-native-svg` (or `react-native-qrcode-svg`) later — see `mfa-totp.md`.

## Keys (`.env`, git-ignored)

```
EXPO_PUBLIC_SUPABASE_URL=https://<project-ref>.supabase.co
EXPO_PUBLIC_SUPABASE_PUBLISHABLE_KEY=sb_publishable_...
```

Source: Dashboard → Project Settings → **API keys** — the *publishable* key (or the legacy *anon* key). Both are safe client-side; RLS protects the database. **Never** the `service_role` key.

## Client — `lib/supabase.ts`

```ts
import "react-native-url-polyfill/auto";
import AsyncStorage from "@react-native-async-storage/async-storage";
import { createClient } from "@supabase/supabase-js";

export const supabase = createClient(
  process.env.EXPO_PUBLIC_SUPABASE_URL!,
  process.env.EXPO_PUBLIC_SUPABASE_PUBLISHABLE_KEY!,
  {
    auth: {
      storage: AsyncStorage,
      autoRefreshToken: true,
      persistSession: true,
      detectSessionInUrl: false, // native app: links arrive via deep links, not a browser URL
      // experimental: { recoveryCodes: true },  // uncomment when wiring recovery codes (mfa-recovery-codes.md)
    },
  },
);
```

Encrypted variant (hardening): sessions larger than SecureStore's 2048-byte limit are encrypted with an AES key kept in `expo-secure-store`, stored in AsyncStorage (`LargeSecureStore` — official example: <https://supabase.com/docs/reference/javascript/initializing?example=react-native-options-async-storage>). Offer it when the user asks for hardened storage; AsyncStorage is the documented baseline.

## Deep links

Auth emails (confirmation, password reset, email change) open the app via deep links. Three parts:

**1. App scheme — `app.json`:**

```json
{ "expo": { "scheme": "com.yourname.yourapp" } }
```

**2. Dashboard → Auth → URL Configuration → Additional Redirect URLs:**

Add every URL the app will use, e.g. `com.yourname.yourapp://**` plus `exp://**` while testing in Expo Go. Redirects that are not in the allowlist are ignored by Supabase Auth — a missing entry is the #1 reason "the email link does nothing".

**3. Shared deep-link handler — `lib/auth-link.ts`** (covers every email flow — confirmation, reset, email change — in one place):

```ts
import * as Linking from "expo-linking";
import * as QueryParams from "expo-auth-session/build/QueryParams";
import { supabase } from "@/lib/supabase";

/** Called with any URL the app is opened with from an auth email. */
export async function handleAuthUrl(url: string): Promise<void> {
  const { params, errorCode } = QueryParams.getQueryParams(url);
  if (errorCode || !params) return;

  if (typeof params.code === "string" && params.code) {
    // PKCE flow: the email link carries ?code=... and the verifier is in app storage
    await supabase.auth.exchangeCodeForSession(params.code);
    return;
  }
  const access = typeof params.access_token === "string" ? params.access_token : undefined;
  const refresh = typeof params.refresh_token === "string" ? params.refresh_token : undefined;
  if (access && refresh) {
    // implicit flow fallback (older templates): tokens are in the fragment/query
    await supabase.auth.setSession({ access_token: access, refresh_token: refresh });
  }
}
```

Wire it at the root (cold start + warm links):

```tsx
import { useEffect } from "react";
import { Linking } from "react-native";
import { handleAuthUrl } from "@/lib/auth-link";

useEffect(() => {
  Linking.getInitialURL().then((url) => url && handleAuthUrl(url));
  const sub = Linking.addEventListener("url", ({ url }) => handleAuthUrl(url));
  return () => sub.remove();
}, []);
```

Notes:

- Build redirect targets at runtime with `Linking.createURL("/reset-password")` — it resolves to the right scheme for the environment (custom scheme in prod/dev builds, `exp://…` in Expo Go).
- Give each flow a distinct path (`/confirm-email`, `/reset-password`) so the router can land users on the right screen; register both with `//**` on the dashboard.
- Universal links (`applinks:`) are a production improvement over custom schemes — dedicated setup, see the Expo deep-linking guide.
- Test a deep link on device/simulator without a real email: `npx uri-scheme open "com.yourname.yourapp://reset-password" --ios` (Android: `--android`).
