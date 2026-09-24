# Permissions

Where authorization logic lives varies more than almost anything else in
this stack, and it should — it depends entirely on who owns the permission
model. Two cases cover most of it, the second with three sub-shapes.

## Case 1: group/role claims from an IdP (better-auth's access control)

The IdP itself is the source of truth for who's in what group (an OIDC
`groups` claim, an Entra ID app role, etc.), and the app translates that
into fine-grained, resource-level permissions. Reach for better-auth's
`admin` plugin + `createAccessControl` once checks go beyond "logged in or
not" — an admin area, per-user resource ownership, several distinct roles
each seeing a different slice of the same data.

A small app with just "logged in or not" gating needs nothing beyond
`proxy.ts` checking for a session (see auth.md) — don't introduce an access
control layer for a binary check.

### `src/lib/permissions.ts` — statements and roles

A **statement** declares every resource and the actions that exist on it;
a **role** is a named subset of those actions:

```ts
import { createAccessControl } from "better-auth/plugins/access";

export const statement = {
  product: ["create", "read", "update", "delete"],
  order: ["read", "update"],
} as const;

export const ac = createAccessControl(statement);

export const roles = {
  admin: ac.newRole({
    product: ["create", "read", "update", "delete"],
    order: ["read", "update"],
  }),
  member: ac.newRole({
    product: ["read"],
    order: ["read"],
  }),
};
```

This is the direct equivalent of a hand-rolled ability-rule table: the
`statement` is the fixed universe of actions-per-resource (extend it here,
not locally, when a new resource needs permission checks), and each role in
`roles` is one named point in that space. Add a role by adding an entry to
`roles`, not by inventing a new ad hoc check somewhere else.

### `src/lib/auth/server.ts` — wiring the `admin` plugin

```ts
import { admin as adminPlugin } from "better-auth/plugins";
import { ac, roles } from "./permissions";

export const auth = betterAuth({
  // ...database, provider config from auth.md...
  plugins: [
    adminPlugin({ ac, roles }),
    // ...genericOAuth/socialProviders...
  ],
});
```

### Mapping IdP groups onto a role at sign-in

The role has to land on the user *somehow* — map it from the IdP's group
claim in the OAuth provider's profile-mapping hook (see auth.md), computed
once at account-link time rather than re-derived from raw groups on every
request:

```ts
genericOAuth({
  config: [{
    providerId: "identity-provider",
    // ...
    mapProfileToUser: (profile) => ({
      groups: profile.groups ?? [],
      role: profile.groups?.includes("app-admins") ? "admin" : "member",
    }),
  }],
}),
```

This is the one real architectural difference from a request-time ability
builder: the role is **persisted** on the user row (better-auth's `admin`
plugin expects a `role` column), not rebuilt from scratch on every request
from the session's group list. A user whose IdP group membership changes
picks up the new role the next time they sign in (when
`mapProfileToUser`/the equivalent update hook runs again), not instantly on
their next request — acceptable for almost everything permission-related;
if an app genuinely needs an instant revocation path, that's a reason to
force a re-authentication, not to move back to a request-time ability
rebuild.

### Checking a permission

Server-side (authoritative — this is the one that actually gates a
mutation):

```ts
const { success } = await auth.api.userHasPermission({
  body: { userId: user.id, permissions: { product: ["update"] } },
});
if (!success) throw new ForbiddenError(IDENTITIES.PRODUCTS);
```

Client-side (for conditionally rendering UI — a hidden button is a UX
nicety, never the actual gate; the server check above is what's load-bearing):
the `admin` plugin's client-side helper checks a role string you already
have (from `useSession()`) against the same `ac`/`roles` definitions,
without a network round-trip.

### Wiring into `proxy.ts`

`proxy.ts` still only checks session **presence** (see auth.md) — it
doesn't call `userHasPermission` (that's a DB round-trip, exactly what the
optimistic-check guidance in auth.md says to avoid at that layer). Route-level
permission gating happens where the route's data actually gets fetched (the
layout or page for that route segment), using the same server-side check
as above, not in `proxy.ts` itself.

### In services

Always check permissions **first**, before any Prisma call, and throw
`ForbiddenError(entity)` (see services-and-errors.md) rather than silently
filtering results:

```ts
const user = await getUser();
const { success } = await auth.api.userHasPermission({
  body: { userId: user.id, permissions: { product: ["update"] } },
});
if (!success) throw new ForbiddenError(IDENTITIES.PRODUCTS);
```

**One role per user, site-wide** is what the `admin` plugin above gives —
right for a single-tenant app. A multi-tenant app where a user belongs to
several organizations, with a *different* role in each, isn't this case —
skip to 2c below.

## Case 2: permissions managed internally

The IdP only proves *who* someone is — it doesn't hand back a group claim
the app can build a role from, either because the IdP genuinely has nothing
to offer beyond identity, or because the permission model belongs to this
app (or another service the app talks to), not to the IdP. Three
sub-shapes:

**2a. Inline checks, no access-control layer (Shape A/C — app owns the data).**
Skip the statement/role indirection entirely when there's no group-derived
role to build from — just check ownership/role directly in the service,
against whatever the app's own `User`/`Role` model says:

```ts
// src/services/stages.ts (see services-and-errors.md on the stages/stage naming split)
export const stages = {
  async update(id: string, data: StageInput) {
    const { workspaceId } = await getUser(); // "internal" here: workspace ownership the app itself tracks
    const existing = await prisma.stage.findFirst({ where: { id, workspaceId } });
    if (!existing) throw new NotFoundError(IDENTITIES.STAGES); // scoping the query to workspaceId, not a separate
    // permission check, is itself the authorization: a stage outside the caller's workspace looks like a missing one.
    return prisma.stage.update({ where: { id }, data });
  },
};
```

This is usually enough for a single-tenant-per-user or workspace-scoped app:
authorization *is* the query's `where` clause, not a separate permission
check layered on top. Reach for Case 1's access control only once there's
more than one axis of "who can do what" to reason about (multiple roles ×
multiple resource types), not for straightforward ownership scoping.

**2b. Delegated to an upstream API (Shape B — thin BFF).** When the Next.js
app is a BFF in front of another service, that service usually owns the
actual RBAC model too — roles, role-assignments, and the enforcement of
"can this user do this" all live server-side behind the API, not in the
Next.js app. The app's own "permissions"/"roles" services are then just thin
passthroughs (`GET`/`POST` calls to the upstream API's own permissions
endpoints), and a `403` from the upstream API is the actual enforcement
point — the Next.js app mostly just reflects role/permission data in the UI
(who has what role, a picker to assign one) rather than deciding access
itself:

```ts
// src/services/permissions.ts — thin passthrough, the upstream API is the actual authority
export async function listRoles(): Promise<Role[]> {
  const { data } = await api.get<Role[]>("/roles");
  return data;
}
export async function assignRole(userId: string, roleId: string) {
  "use server";
  await api.post(`/members/${userId}/roles`, { roleId }); // a 403 here IS the permission check
  revalidatePath("/members");
}
```

Don't duplicate the upstream service's authorization logic client-side in
this case (e.g. building a local access-control role table from a
locally-cached copy of the role list) — it drifts from the real source of
truth and adds a second place bugs can hide. Let the API's response
(success or `403`) be the answer, and only mirror role data in the UI for
*display*.

**2c. Org-scoped roles (Shape A/C — a multi-tenant app with teams).** A user
belongs to several organizations and has a *different* role in each —
membership and role assignment are entirely the app's own doing (an org
owner invites someone and picks their role), which is exactly why this is
Case 2, internally managed, not Case 1: no IdP is involved in deciding who's
an "owner" of a given organization. What it shares with Case 1 is the
formal machinery — better-auth's `organization` plugin uses the same
`createAccessControl` statement/role shape as the `admin` plugin, just
scoped to a `referenceId` (the organization id) instead of applied globally
to the user.

```ts
// src/lib/permissions.ts
import { createAccessControl } from "better-auth/plugins/access";
import { defaultStatements, ownerAc, adminAc, memberAc } from "better-auth/plugins/organization/access";

const statement = {
  ...defaultStatements, // the plugin's own resources: organization, member, invitation
  project: ["create", "read", "update", "delete"],
} as const;

export const ac = createAccessControl(statement);

export const roles = {
  owner: ac.newRole({ ...ownerAc.statements, project: ["create", "read", "update", "delete"] }),
  admin: ac.newRole({ ...adminAc.statements, project: ["create", "read", "update"] }),
  member: ac.newRole({ ...memberAc.statements, project: ["read"] }),
};
```

```ts
import { organization } from "better-auth/plugins";
import { ac, roles } from "@/lib/permissions";

export const auth = betterAuth({
  // ...database, provider config from auth.md...
  plugins: [organization({ ac, roles }) /* , genericOAuth/socialProviders, etc. */],
});
```

A user can be a member of more than one organization, so requests need to
say *which* one they mean — either an explicit id passed alongside the
request, or the session's current "active organization"
(`authClient.organization.setActive({ organizationId })`, client-side, when
the UI has an org switcher). Checking a permission takes that id
explicitly rather than assuming a request implicitly means "the active
one" — explicit is safer, and matches this stack's usual stance
(services-and-errors.md's `IDENTITIES`, `ForbiddenError(entity)`: name
things, don't infer them from ambient state):

```ts
// src/services/projects.ts
export const projects = {
  async remove(id: string, organizationId: string) {
    const user = await getUser();
    const { success } = await auth.api.hasPermission({
      headers: await headers(),
      body: { organizationId, permissions: { project: ["delete"] } },
    });
    if (!success) throw new ForbiddenError(IDENTITIES.PROJECTS);
    return prisma.project.delete({ where: { id, organizationId } });
  },
};
```

Outside a request/response cycle — inside a plugin hook that only hands you
a `referenceId` and a `user`, not `headers` to call `auth.api.hasPermission`
with — resolve the membership row directly and check the role object's own
`.authorize(...)` locally instead. stripe.md's `authorizeReference` section
is a fully worked example of exactly this shape (there, gating who can
manage an organization's Stripe subscription); the same local-check pattern
applies to any other plugin hook shaped the same way, not just billing.

Pick Case 1 when the IdP hands back group/role claims and the app needs a
single, site-wide role built from them; pick Case 2a when the app owns its
data and authorization reduces to simple ownership; pick Case 2b when
another service already owns the RBAC model; pick Case 2c when the app has
its own multi-tenant teams/organizations and a user's role varies per team.
Don't introduce the access-control machinery (1 or 2c) into an app that
fits 2a or 2b just because this pattern exists elsewhere — it adds real
weight (the statement/role tables, the plugin wiring) that only pays for
itself once there's a real permission translation to do, not for
straightforward ownership scoping.
