# File organization

This base layout holds regardless of app size — a small internal tool and a
large multi-domain platform both use it, the large one just has more domain
folders under each top-level directory.

## Top-level `src/` layout

```
src/
  app/                  — routes (App Router). Route groups like (root), (public) split
                          public vs. authenticated chrome without affecting the URL.
  actions/              — "use server" files exporting Server Actions directly (can throw)
  services/             — "use server" files, one per domain, the data-access layer
  components/
    ui/                 — shadcn/ui primitives, generated — don't hand-edit, don't lint
    dialog/              — generic, cross-domain wrappers (ConfirmDialog, DeleteButton, ...)
    <domain>/             — feature UI, one folder per domain
  constants/              — status configs, enums, static lookup tables, per domain
  types/                   — Prisma-derived / API-derived types only, no zod here
  validations/             — zod schemas, one file per domain (see validation.md)
  lib/
    auth/
      server.ts                — the betterAuth instance, getUser()/session()
      client.ts                 — createAuthClient(), no provider needed
    prisma.ts                    — the Prisma client singleton
    permissions.ts                — access-control statements/roles, if using them (see permissions.md)
    utils.ts, format.ts, log.ts, api.ts — cross-cutting helpers
  providers/                — thin "use client" wrappers around context providers
  i18n/
    config.ts                — locales, defaultLocale, the Locale type — no I/O, importable anywhere
    request.ts                — next-intl's getRequestConfig, wired to the locale service below
  messages/                    — en.json, fr.json
  prisma/                       — schema.prisma, migrations/, seed.ts
  proxy.ts                      — route guard (Next.js 16 name for middleware.ts)
  fonts/                        — local font files
```

`generated/` (Prisma client, generated zod schemas) lives **outside** `src/`,
at the repo root, and is gitignored — it's build output, not source.

## Domain mirroring

A domain (`product`, `order`, `organisation`, `stage`, ...) gets one file per
concern, named after the domain, spread across parallel folders:

```
src/types/<domain>.ts        — type only (Prisma-derived or API-derived)
src/validations/<domain>.ts  — zod schema(s), factory pattern if localized
src/constants/<domain>.ts    — status configs, enums, static lookups (only if needed)
src/services/<domain-plural>.ts — "use server" CRUD functions, namespace-exported
src/components/<domain>/     — UI for that domain
```

Look at how an existing domain is laid out before creating a new one — match
the same split rather than inventing a new shape. `services/<domain>.ts` is
always the domain's **plural** form (`orders.ts`, `organisations.ts`,
`stages.ts`), matching the plural namespace object it exports — see
services-and-errors.md for why (it's what keeps the singular free for a
locally fetched record: `const order = await orders.get(id)`).
`types`/`validations`/`constants` files stay singular (`order.ts`,
`stage.ts`) since they're not imported as a namespace you call methods on.

## Component file names inside a domain folder

Recurring names, pick whichever apply to the feature:

- `table.tsx` — the admin list/CRUD view
- `form.tsx` — create/edit form (often a dialog, see validation.md)
- `add.tsx` — a button that adds/toggles the item into a collection
- `item.tsx` — a single row rendered inside a list/sheet
- `sheet.tsx` — a slide-over panel
- `card.tsx` / `details.tsx` — read-only display

Generic, near-identical admin CRUD tables across entities (categories,
qualities, tags, ...) should share one component driven by a `namespace`/
`entity` prop rather than being copy-pasted per entity. Reach for extending an
existing generic table before writing a new near-identical one; only entities
with materially different CRUD needs get their own `table.tsx`.

## Generic component folders — not `ui/`, not a domain

`components/` isn't just `ui/` (shadcn primitives) plus one folder per
domain — a third category holds compositional wrappers that are genuinely
reusable across every domain, built *on top of* `ui/` primitives but with
zero domain knowledge baked in. `components/dialog/` is the recurring
example: a generic confirm dialog and a generic delete-button both take
their copy and their action as props, so every domain reuses the same
component instead of each domain writing its own delete-confirmation dialog:

```tsx
// components/dialog/confirm.tsx — fully generic, no domain knowledge
export function ConfirmDialog({
  trigger, title, description, onConfirm, destructive,
}: { trigger: ReactNode; title: string; description?: string; onConfirm: () => void | Promise<void>; destructive?: boolean }) {
  // renders an AlertDialog from components/ui/, calls onConfirm on confirm
}

// components/dialog/delete.tsx — the same idea, specialized to the
// type-the-name-to-confirm pattern, still with zero domain knowledge
export function DeleteButton({
  name, redirectTo, onDelete,
}: { name: string; redirectTo: string; onDelete: () => Promise<void> }) {
  // renders a Trash2 button that opens an AlertDialog requiring `name` to be retyped
}
```

A domain then just supplies its own copy and its own action — no per-domain
dialog component:

```tsx
<DeleteButton
  name={organisation.displayName}
  redirectTo="/organisations"
  onDelete={() => organisations.remove(organisation.id)}
/>
```

The same principle extends to other cross-cutting concerns as an app grows
— a `forms/` folder for generic field-rendering wrappers, a `layout/` folder
for page-chrome pieces reused across every route — the test is the same
one: does this component take everything domain-specific as props, with
nothing about one particular entity hardcoded inside it? If yes, it's a
generic folder, not a domain folder, no matter how "domain-ish" the feature
it supports feels.

## The `providers/` folder

One thin `"use client"` wrapper file per context provider used in the root
layout — `providers/<name>.tsx`, imported as `@/providers/<name>`. Reach for
one whenever a genuine client-side context provider is needed: fully custom
app code with its own hooks/state (a realtime/websocket connection, a
feature-flag context), or a third-party client component you want behind a
stable, app-owned import path instead of reaching into the library directly
at every call site — `providers/theme.tsx`, wrapping `next-themes` (see
ui.md), is the concrete, real example of that second case in this stack.

```tsx
// src/providers/feature-flags.tsx — schematic: fully custom state follows this shape
"use client";
import { createContext, useContext, useState, type ReactNode } from "react";

const FeatureFlagsContext = createContext<FeatureFlagsContextValue | null>(null);

export function FeatureFlagsProvider({ children }: { children: ReactNode }) {
  const [flags, setFlags] = useState<FeatureFlags>({});
  return <FeatureFlagsContext.Provider value={{ flags, setFlags }}>{children}</FeatureFlagsContext.Provider>;
}
export const useFeatureFlags = () => useContext(FeatureFlagsContext)!;
```

Auth and i18n don't need one of these in this stack's defaults: better-auth's
client (`useSession()`, see auth.md) is store-backed, not context-based, so
it works with no provider at all; `next-intl`'s `NextIntlClientProvider` is
already a ready-to-compose client component with nothing app-specific to
add, used directly inline in the root layout (see tooling.md). Theming
(`providers/theme.tsx`, wrapping `next-themes` — see ui.md) is the one
piece in this stack's own defaults that does need the wrapper, since
`next-themes`' provider is exactly the "third-party client component behind
a stable import path" case. Don't leave the folder empty out of habit
either way — add a wrapper once something genuinely needs shared
client-side state, skip it when using a library's component directly inline
works just as well.

## Naming: nest instead of hyphenating

File names are kebab-case, never PascalCase/camelCase (`ProductCard.tsx`,
`sessionProvider.tsx`) — but a hyphen joining two meaningful words is usually
a sign the name wants to be a folder instead. When the first word names a
real domain/category the codebase already organizes by (or should), nest
rather than hyphenate:

- ❌ `components/product-card.tsx` → ✅ `components/product/card.tsx`
- ❌ `components/session-provider.tsx` → ✅ `providers/session.tsx` — the
  folder already says "provider", don't repeat it in the filename
- ❌ `components/order-table.tsx` → ✅ `components/order/table.tsx`

Keep a hyphenated, flat filename when the compound name does **not**
correspond to a real domain/category folder — nesting would just invent a
meaningless folder with one file in it. `components/ui/data-table.tsx` is a
generic, reusable table primitive: "data" isn't a domain, so it stays flat
and hyphenated rather than becoming `components/ui/data/table.tsx`. If that
same table turns out to be specific to one domain (an admin product listing,
say), it belongs under that domain instead — `components/product/table.tsx`,
not `components/product-table.tsx`.

Rule of thumb before naming a file `x-y.tsx`: is `x` a domain this codebase
already organizes by (or would `x/` earn its own folder as more files show
up)? If yes, nest: `x/y.tsx`. If `x` is a generic, non-domain descriptor
(`data`, `base`, `shared`) with no folder of its own planned, keep the
hyphen — that's what `data-table.tsx` and similar shared primitives already
do. This doesn't apply to the `use-<thing>.tsx` hook-naming convention below
— `use-` is a functional prefix, not a domain word, and always stays
hyphenated.

## Feature sub-folders

When a component grows enough internal pieces to need its own hook/sub-parts
(filters, uploads, multi-step forms), give it a folder instead of one huge
file: `components/product/filters/{use-product-filters.tsx,fields.tsx,sidebar.tsx}`.
The hook file is named `use-<thing>.tsx` even when it also exports non-hook
helpers.

## Imports

- `@/...` for anything under `src/`.
- Relative paths for `generated/` (Prisma client/zod schemas, outside `src/`)
  — these become long `../../../../../../generated/...` chains from deeply
  nested routes. That's expected; don't try to alias around it, and don't
  move `generated/` under `src/` to "fix" it (it must stay gitignored and
  regenerable).

## `.gitignore` additions this stack always needs

```
# next.js
/.next/
/out/

# env files
.env*
!.env.example

# typescript
*.tsbuildinfo
next-env.d.ts

# prisma
/generated
```
