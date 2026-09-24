# Authentication

`better-auth`, its Prisma adapter, a single OIDC provider, and a route guard
in `src/proxy.ts`. Session state lives in the database (a real `session`
table via the adapter), not a signed JWT cookie — better-auth manages the
`user`/`session`/`account`/`verification` tables itself (see prisma.md on
generating that part of the schema).

> better-auth's plugin API moves fast. The shape below (`genericOAuth`,
> `admin` plugin, `createAccessControl`) is accurate as of writing — check
> the installed version's docs for exact option names before treating any
> single field name as gospel; the overall architecture (adapter, plugins,
> `auth.api.*` server calls, `authClient.*` client calls) is stable.

## `src/lib/auth/server.ts` — the server instance

`auth/` is a folder, not `lib/auth.ts` + `lib/auth-client.ts` — "auth" is a
real category with two files in it (a server instance and a client
instance), so it nests instead of getting hyphenated apart, same rule as
file-organization.md's "nest instead of hyphenating" (`session-provider.tsx`
→ `providers/session.tsx` is the same move).

```ts
import { betterAuth } from "better-auth";
import { prismaAdapter } from "better-auth/adapters/prisma";
import { genericOAuth } from "better-auth/plugins";
import { prisma } from "@/lib/prisma";

export const auth = betterAuth({
  database: prismaAdapter(prisma, { provider: "postgresql" }),
  plugins: [
    genericOAuth({
      config: [
        {
          providerId: "identity-provider",
          clientId: process.env.CLIENT_ID!,
          clientSecret: process.env.CLIENT_SECRET!,
          discoveryUrl: `${process.env.IDENTITY_ISSUER}/.well-known/openid-configuration`,
          scopes: ["openid", "email", "profile"],
          // Extra IdP claims (group membership, an internal username) that don't map to
          // better-auth's default user fields land here, at account-link time.
          mapProfileToUser: (profile) => ({
            username: profile.preferred_username ?? profile.sub,
            groups: profile.groups ?? [],
          }),
        },
      ],
    }),
  ],
  user: {
    additionalFields: {
      username: { type: "string", required: false },
      groups: { type: "string[]", required: false },
    },
  },
});

export async function session() {
  const { headers } = await import("next/headers");
  return auth.api.getSession({ headers: await headers() });
}

export async function getUser() {
  const result = await session();
  if (!result?.user) throw new Error("User not authenticated"); // or UnauthenticatedError, see services-and-errors.md
  return result.user;
}
```

`user.additionalFields` is what makes `username`/`groups` (and whatever else
a given app needs) show up, fully typed, on `session.user` everywhere —
there's no manual module-augmentation step like the old `next-auth.d.ts`
pattern; types are inferred straight from this config
(`typeof auth.$Infer.Session`).

Named `session()`, not `auth()` — `auth` is already taken two lines up by
the betterAuth instance itself (`export const auth = betterAuth(...)`), so
a wrapper function can't reuse that name in the same module; this is the
same class of naming collision as `organisations`/`organisations.list()` in
services-and-errors.md, and the same fix (pick a name that isn't already
bound, don't fight the shadow). `session()` mirrors better-auth's own
`getSession` naming and returns the raw `{ session, user } | null` — the
honest, unopinionated primitive. `getUser()` is the convenience layer most
call sites actually want (the current user or a thrown error, no null
check at every call site) — built on top of `session()`, not a duplicate
implementation. Reach for `session()` directly only when `null` is a
meaningful, handled case (an optional sign-in state on a public page), not
a caught exception.

## Variant: a built-in social provider instead of `genericOAuth`

better-auth ships first-class providers for the common IdPs (`socialProviders.microsoft`
for Entra ID/Azure AD among others) that need less config than the generic
OIDC plugin, at the cost of less control over exactly which OIDC endpoint/
scope set gets used:

```ts
export const auth = betterAuth({
  database: prismaAdapter(prisma, { provider: "postgresql" }),
  socialProviders: {
    microsoft: {
      clientId: process.env.ENTRA_ID!,
      clientSecret: process.env.ENTRA_SECRET!,
      tenantId: process.env.ENTRA_TENANT_ID!,
    },
  },
});
```

Reach for `genericOAuth` (the main example above) for any IdP without a
built-in provider — a self-hosted Ory Hydra/Keycloak instance, or any
standards-compliant OIDC issuer identified only by its discovery URL. Reach
for a built-in `socialProviders` entry when the IdP has one and its default
scope/claim set is already enough; it's less config for the same result.
Both populate `session.user` the same way once `mapProfileToUser` (generic)
or the provider's own defaults (built-in) are configured — nothing else
downstream (`getUser()`, `proxy.ts`, services) needs to know which one is in
use.

## Two ways to persist a session: DB-backed (default) vs stateless

**DB-backed is the default and the right starting point.** Every session is
a row in the `session` table (see prisma.md); the cookie only holds an
opaque token referencing that row. `auth.api.getSession()` looks it up (an
optional `session.cookieCache` config can serve reads from a short-lived
signed cookie for a few seconds to cut down on DB round-trips, without
changing the DB as the source of truth). This fits Shape A/C naturally (the
app owns its own database, see services-and-errors.md) — there's already a
connection open for everything else.

**Stateless — the `jwt` plugin — is the exception, for one specific case:**
Shape B, a thin BFF in front of an external API that needs to verify the
caller *itself*, without calling back into the Next.js app's session store.
Add the plugin and the app additionally mints a self-contained, signed JWT
(with a JWKS endpoint the upstream API can verify against) alongside the
normal DB session:

```ts
import { jwt } from "better-auth/plugins";

export const auth = betterAuth({
  // ...database, provider config from above...
  plugins: [jwt() /* , genericOAuth(...) / socialProviders */],
});
```

```ts
// src/lib/api.ts — Shape B, forwarding a self-verifiable token instead of the IdP's own access token
const { token } = await auth.api.getToken({ headers: await headers() });
config.headers.Authorization = `Bearer ${token}`;
```

This is a *different* token than the one the Token refresh section below
forwards — that one is the original IdP's OAuth access token, useful when
the upstream API already trusts that IdP directly. The `jwt` plugin's token
is minted *by this app*, useful when the upstream API is meant to trust the
Next.js app itself as an identity boundary instead (or can't maintain a live
lookup against it). Default to DB-backed sessions and the IdP's own access
token; reach for `jwt()` only once a specific upstream consumer actually
needs a self-verifiable token instead of a session/token lookup.

## `src/proxy.ts` — the route guard

Next.js 16 renamed `middleware.ts` to `proxy.ts` (same mechanism, see
SKILL.md). better-auth's own guidance is explicit here: don't call
`auth.api.getSession()` (a real DB round-trip) from the proxy — check for
the session **cookie's presence** only, and do the real, authoritative
session check in the layout/page/service that actually needs it. This is an
optimistic check, not authorization — see permissions.md for where the
real permission check happens.

```ts
import { type NextRequest, NextResponse } from "next/server";
import { getSessionCookie } from "better-auth/cookies";
import { PROTECTED_ROUTES } from "./constants/routes";

export default async function proxy(req: NextRequest) {
  const { pathname } = req.nextUrl;

  if (Object.values(PROTECTED_ROUTES).some(({ path }) => path.test(pathname))) {
    const sessionCookie = getSessionCookie(req);
    if (!sessionCookie) {
      const signInUrl = new URL("/sign-in", req.url);
      signInUrl.searchParams.set("callbackUrl", req.url);
      return NextResponse.redirect(signInUrl);
    }
  }

  return NextResponse.next();
}

export const config = {
  matcher: ["/((?!api/auth|sign-in|error|_next/static|_next/image|favicon.ico|.*\\.[a-zA-Z]+$).*)"],
};
```

`getSessionCookie` only parses/validates the cookie shape — no database
call, no need to lazily import the full `auth` instance the way a
JWT-callback-based library needed to keep its server bundle out of the edge
runtime; there's nothing heavy to keep out here in the first place.
`PROTECTED_ROUTES` is the same `Record<string, { path: RegExp }>` shape as
before (`src/constants/routes.ts`, or `permissions.ts` in apps using the
`admin`-plugin access control — see permissions.md). Always exclude
`api/auth`, the sign-in page itself, `error`, static assets, and
files-with-extensions from the matcher.

Unlike a library with its own hosted sign-in page, better-auth expects the
app to own a real `/sign-in` route — a normal page that calls
`authClient.signIn.social(...)` (built-in provider) or
`authClient.signIn.oauth2(...)` (`genericOAuth`) client-side, typically from
a single "Sign in" button, reading the `callbackUrl` search param proxy.ts
set and passing it through as `callbackURL`.

## `src/app/api/auth/[...all]/route.ts` — the one route handler

```ts
import { toNextJsHandler } from "better-auth/next-js";
import { auth } from "@/lib/auth/server";

export const { GET, POST } = toNextJsHandler(auth);
```

One catch-all route, not two — better-auth doesn't need a separate
sign-in-redirect entrypoint the way a library whose `signIn()` call has to
happen server-side does; `authClient.signIn.*` triggers the OAuth redirect
directly from the client, so there's nothing for a second route to do.

## `src/lib/auth/client.ts` — the client instance, no provider needed

```ts
"use client";
import { createAuthClient } from "better-auth/react";
import { genericOAuthClient } from "better-auth/client/plugins"; // only if using the genericOAuth plugin above

export const authClient = createAuthClient({
  plugins: [genericOAuthClient()],
});

export const { useSession, signIn, signOut } = authClient;
```

`useSession()` works directly, anywhere in the client tree — better-auth's
React client is backed by an external store, not React context, so there's
**no `<SessionProvider>` wrapper needed** in the root layout (unlike the
`providers/session.tsx` pattern an older, context-based auth library
needed — see file-organization.md's `providers/` section, which still
applies to other providers, just not this one). The root layout stays a
plain server component with no auth-specific wiring at all beyond whatever
`getUser()`-based data it fetches itself.

## Token refresh

better-auth stores the OAuth `accessToken`/`refreshToken`/
`accessTokenExpiresAt` on the `account` row itself and refreshes
automatically when asked for a current token — no hand-rolled refresh-cache
/ deduplication logic to write (a real problem with a JWT-in-cookie
approach, effectively solved here by the fact that the token lives
server-side in a table, not duplicated into a client-readable cookie).
Where a service needs a live access token to call an upstream API (the
Shape B BFF pattern in services-and-errors.md), fetch it through the API
instead of reading a stale value off the session:

```ts
// src/lib/api.ts
export const api = axios.create({ baseURL: `${process.env.API_URL}/v1` });
api.interceptors.request.use(async (config) => {
  const { headers } = await import("next/headers");
  const { accessToken } = await auth.api.getAccessToken({
    body: { providerId: "identity-provider" },
    headers: await headers(),
  });
  if (accessToken) config.headers.Authorization = `Bearer ${accessToken}`;
  return config;
});
```

`getAccessToken` refreshes under the hood if the stored token is expired,
using the stored refresh token — this replaces the old
`REFRESH_BUFFER_SECONDS` + in-flight-promise-cache pattern entirely; that
complexity existed specifically to avoid racing a rotating refresh token
across concurrent requests in a JWT-cookie design, and doesn't apply once
the token lives in one place (the database) instead of being re-derived
per-request from a signed cookie.

## Optional plugin: two-factor / TOTP

```ts
import { twoFactor } from "better-auth/plugins";

export const auth = betterAuth({
  // ...database, provider config from above...
  plugins: [
    twoFactor({ issuer: "MyApp" }), // shown inside the user's authenticator app
    // ...genericOAuth/socialProviders, admin, etc.
  ],
});
```

```ts
// src/lib/auth/client.ts
import { twoFactorClient } from "better-auth/client/plugins";

export const authClient = createAuthClient({
  plugins: [twoFactorClient(), genericOAuthClient()],
});
```

Enrollment is a client-side flow: `authClient.twoFactor.enable({ password })`
returns a `totpURI` (render it as a QR code for the user's authenticator
app) and a set of backup codes; `authClient.twoFactor.verifyTotp({ code })`
confirms the six-digit code at sign-in once it's enabled.

**The friction point for this stack's default setup:** `enable`/`disable`
require the account's *password* to re-confirm a security-sensitive
change — but every example in this file so far is OAuth-only
(`genericOAuth`/`socialProviders`), with no password ever set. An
OAuth-only account can't call `enable` as written above. Two ways around
it, pick based on what the app actually needs:

- Also turn on better-auth's `emailAndPassword` provider, even if OAuth
  stays the *primary* sign-in method — the password exists only to gate
  2FA enrollment/changes, not for everyday sign-in.
- Skip TOTP entirely and rely on the IdP's own MFA instead. If every user
  signs in through an enterprise IdP (Entra ID, Ory Hydra, etc.), that IdP
  is very likely already enforcing its own second factor upstream, and an
  app-level TOTP layer is redundant rather than additive. TOTP earns its
  place specifically when the app supports (or primarily uses) email/password
  sign-in and needs a second factor independent of any IdP.

Check the installed version's docs for the current re-authentication story
before assuming the password requirement above is exactly as described —
this is precisely the kind of plugin-surface detail the epistemic note at
the top of this file is about.

## Optional plugin: Stripe billing (`@better-auth/stripe`)

Billing (subscriptions, per-user or per-organization, plan-limit
enforcement, webhooks) has its own file — see stripe.md. It follows the
same plugin-pairing shape as everything above (a server plugin plus its
client counterpart, routes mounted under the same `api/auth/[...all]`
catch-all, its own CLI-generated tables) — stripe.md picks up from here.

## Never bypass `getUser()`/`session()`

Every service function that needs the current user calls `getUser()` (or
`session()` directly if `null` is a meaningful, handled outcome rather than
an error) — don't read cookies/headers manually or call
`auth.api.getSession()` ad hoc to reconstruct identity in a one-off place.
