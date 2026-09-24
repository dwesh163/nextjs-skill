# Stripe billing (`@better-auth/stripe`)

A separate package, not part of core `better-auth`:
`bun add @better-auth/stripe stripe`. Builds on the base auth setup in
auth.md — read that first, this file only covers what the Stripe plugin
adds on top.

> Same epistemic note as auth.md: the shape below is accurate as of
> writing, better-auth's plugin surface moves fast — verify exact option/
> method names against the installed version before treating any single
> one as fixed, especially `authorizeReference`'s `action` values later in
> this file, which are the least certain detail here.

## Base setup — billing per user

```ts
import { betterAuth } from "better-auth";
import { stripe } from "@better-auth/stripe";
import Stripe from "stripe";
import { prismaAdapter } from "better-auth/adapters/prisma";
import { prisma } from "@/lib/prisma";

const stripeClient = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: "2025-11-17.clover",
});

export const auth = betterAuth({
  database: prismaAdapter(prisma, { provider: "postgresql" }),
  plugins: [
    stripe({
      stripeClient,
      stripeWebhookSecret: process.env.STRIPE_WEBHOOK_SECRET!,
      createCustomerOnSignUp: true, // creates the Stripe Customer at sign-up, not lazily on first checkout
      subscription: {
        enabled: true,
        plans: [
          {
            name: "pro",
            priceId: process.env.STRIPE_PRICE_PRO_MONTHLY!,
            annualDiscountPriceId: process.env.STRIPE_PRICE_PRO_YEARLY!,
            limits: { projects: 20, seats: 5 },
          },
          { name: "free", priceId: "", limits: { projects: 3, seats: 1 } },
        ],
      },
    }),
    // ...genericOAuth/socialProviders, admin, etc. — see auth.md
  ],
});
```

`limits` on each plan is app-defined data, read and enforced by the app
itself — services-and-errors.md's permission-check-first convention applies
here too: check the caller's plan limits before a mutation that would
exceed them, in the same position in the function as a `ForbiddenError`
check, right after (or alongside) the permission check from permissions.md.
Stripe and better-auth store and hand back `limits` verbatim; neither
interprets or enforces it.

**No new route file, no separate webhook endpoint.** Like every better-auth
plugin, Stripe's routes — including the webhook receiver — mount under the
same `api/auth/[...all]/route.ts` catch-all already wired for auth (see
auth.md). Point the Stripe dashboard's webhook URL at
`https://<app>/api/auth/stripe/webhook`; don't add a second
`api/stripe/webhook/route.ts`.

**Client plugin counterpart**, same pairing rule as every other better-auth
plugin (a server-side plugin needs a matching client-side plugin
registered, same as `genericOAuthClient()` in auth.md):

```ts
// src/lib/auth/client.ts
import { stripeClient } from "@better-auth/stripe/client";

export const authClient = createAuthClient({
  plugins: [stripeClient({ subscription: true }), genericOAuthClient()],
});
```

Starting a checkout / listing an existing subscription, both client-side:

```ts
await authClient.subscription.upgrade({
  plan: "pro",
  successUrl: "/billing?success=true",
  cancelUrl: "/billing",
}); // redirects to Stripe Checkout

const { data: subscriptions } = await authClient.subscription.list();
```

**Schema and env vars, same discipline as everywhere else in this stack:**
the plugin adds its own tables (a `subscription` table, a
`stripeCustomerId` field on `user`) — regenerate via
`bunx @better-auth/cli generate` (see prisma.md), don't hand-write them.
`STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, and one price-id var per plan
go in `.env`/`.env.example` like any other secret (see tooling.md) — never
hardcoded. The webhook secret specifically is what lets
`stripeWebhookSecret` verify a webhook payload actually came from Stripe
rather than an unauthenticated POST to a guessable URL.

## Billing per organization instead of per user

A multi-tenant app usually wants one subscription per *organization*, not
per user — everyone on a team shares the team's plan. Pass a `referenceId`
(the organization id) through `subscription.upgrade` instead of relying on
the implicit per-user scoping from the base setup above:

```ts
await authClient.subscription.upgrade({
  plan: "pro",
  referenceId: organizationId,
  successUrl: "/billing?success=true",
  cancelUrl: "/billing",
});
```

This needs better-auth's `organization` plugin alongside `stripe` — it's
what gives the app members-with-roles-per-organization to check
`referenceId` against in the first place:

```ts
import { organization } from "better-auth/plugins";

export const auth = betterAuth({
  database: prismaAdapter(prisma, { provider: "postgresql" }),
  plugins: [
    organization(), // members, roles (owner/admin/member by default), invitations
    stripe({
      stripeClient,
      stripeWebhookSecret: process.env.STRIPE_WEBHOOK_SECRET!,
      subscription: {
        enabled: true,
        plans: [{ name: "pro", priceId: process.env.STRIPE_PRICE_PRO!, limits: { seats: 10 } }],
        authorizeReference: async ({ user, referenceId, action }) => {
          /* see below — this is the one part that must not be skipped */
        },
      },
    }),
  ],
});
```

`createCustomerOnSignUp` (from the base setup) creates a Customer for the
*user*, which isn't what gates an organization's billing — the org-scoped
Stripe Customer for a given `referenceId` gets created the first time
`subscription.upgrade({ referenceId })` runs against it. Check the
installed version's docs if the app needs that customer to exist eagerly
(at organization-creation time) rather than lazily on first checkout.

### `authorizeReference` is not optional

**Without it, any member of any organization can upgrade, downgrade, or
cancel any other organization's subscription** — `referenceId` is just a
string the client sends; nothing stops a member of org A from passing org
B's id unless the plugin is told how to check it. This hook is the only
thing standing between "any authenticated user" and "this org's billing."
Always implement it when using per-organization billing; never ship the
default (no check) to production.

The wrong version — checking a raw role string against a hardcoded literal:

```ts
// ❌ works, but "owner" is a hardcoded string with no compile-time check, and
// doesn't compose if the app ever needs a role other than owner to manage billing
authorizeReference: async ({ user, referenceId }) => {
  const member = await prisma.member.findFirst({ where: { organizationId: referenceId, userId: user.id } });
  return member?.role === "owner";
},
```

The right version — a real permission check through better-auth's access
control (see permissions.md), not a role-name string comparison:

```ts
// src/lib/permissions.ts
import { createAccessControl } from "better-auth/plugins/access";
import { defaultStatements, ownerAc, adminAc, memberAc } from "better-auth/plugins/organization/access";

const statement = {
  ...defaultStatements, // the organization plugin's own resources (organization, member, invitation)
  billing: ["view", "manage"],
} as const;

export const ac = createAccessControl(statement);

export const roles = {
  owner: ac.newRole({ ...ownerAc.statements, billing: ["view", "manage"] }),
  admin: ac.newRole({ ...adminAc.statements, billing: ["view"] }),
  member: ac.newRole({ ...memberAc.statements, billing: [] }),
};
```

```ts
import { organization } from "better-auth/plugins";
import { ac, roles } from "@/lib/permissions";

organization({ ac, roles }), // same statement/role shape as the admin plugin in permissions.md, scoped per-org instead of globally
stripe({
  // ...
  subscription: {
    // ...
    authorizeReference: async ({ user, referenceId, action }) => {
      const member = await prisma.member.findFirst({ where: { organizationId: referenceId, userId: user.id } });
      if (!member) return false; // not even a member of this org — never authorized, full stop

      const role = roles[member.role as keyof typeof roles];
      if (!role) return false;

      // Map the action better-auth passes in to the permission it actually needs.
      // Fail closed for anything not explicitly recognized here — deny by default, not allow.
      const required = action === "list-subscription" ? "view" : "manage";
      return role.authorize({ billing: [required] }).success;
    },
  },
}),
```

**Named `billing`, not `money`.** `money` doesn't say what actions exist on
it or read as a resource name next to `organization`/`member`/`invitation`
in the same statement; `billing` does, and matches how this concept is
named almost everywhere else (an app's own UI, Stripe's own vocabulary).
`subscription` was the other candidate and was rejected specifically
because the Stripe plugin's own vocabulary already uses "subscription" for
the Stripe entity itself (`subscription.upgrade`, `subscription.list`) —
reusing it as the permission-resource name too invites confusion between
"a permission about billing" and "CRUD on a subscription record." Pick
whatever single word is unambiguous in the app's own domain language if
neither fits; the principle (a real, statement-defined resource — not a
role-name string comparison, not "money") is what matters, not this exact
spelling.

This is permissions.md's Case 2c (org-scoped roles), not Case 1 — org
membership and roles here are entirely the app's own doing (an org owner
invites someone and picks their role), not derived from an IdP claim.
`authorizeReference` is where a plugin that takes a bare `referenceId`
(Stripe, here) gets told how to resolve that id back into "does this
specific user have this specific permission on this specific organization"
instead of trusting the id at face value — see permissions.md's Case 2c for
the same local-`.authorize()`-check shape applied to a non-billing example.
