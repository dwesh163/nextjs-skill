# Server vs. client components

Default to a server component. Only add `"use client";` when the component
actually needs client-side behavior.

## Signals that a component needs `"use client"`

- `useState`/`useEffect`/`useTransition`/any React hook that isn't a plain
  server-side `await`.
- `useTranslations` (the **sync** `next-intl` hook). If a component only
  reads translations and fetches data with no interactivity, prefer
  `getTranslations` from `next-intl/server` (async) and keep it a server
  component instead of reaching for `"use client"` just to call
  `useTranslations`.
- Event handlers (`onClick`, `onChange`, form submission via
  `react-hook-form`), toast triggers, browser-only APIs (`window`,
  drag-and-drop).

If none of these apply — the component just awaits services and renders —
leave it a server component with no directive. `"use client";` is always the
first line of the file, followed by a blank line, e.g.:

```tsx
"use client";

import { useTranslations } from "next-intl";
```

## Data fetching split

- **Server component**: `const translations = { x: await getTranslations("...") }`,
  call services directly with `await`, no loading state needed.
- **Client component**: fetch inside `useEffect` (or receive a server action
  passed down), track `loading`/`data` state, `useTranslations` (sync) for
  text.

### Detail/edit pages: fetch server-side, extract interactivity

Route-level `[id]`/`[slug]` pages that only *read* a record, or read-then-edit
it, are `async` server components — not client components with a
`useEffect` fetch:

```tsx
export default async function OrderDetail({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const { data: order } = await ordersService.get(id);
  if (!order) return <EntityNotFound entity={IDENTITIES.ORDERS} />;
  // render — no loading state needed, the page blocks server-side
}
```

No `loading`/`useState<T | undefined>` dance — the page simply doesn't
render until the data is there. Anything interactive on the page (a status
dropdown, a full edit form) is extracted into its own `"use client"`
component that receives the already-fetched record as a prop and calls the
same server functions directly on submit/click. Prefer `router.refresh()`
after a mutation over lifting/duplicating server state into client
`useState` — the server component re-fetches and the page re-renders with
fresh data.

Reach for the old client-fetch pattern only when there's truly no
server-renderable parent to hang the fetch off of (e.g. a widget that only
ever mounts client-side, like a cart item count in the header that has to
react to client-only state).

## Server actions vs. services

Files under `src/actions/` start with `"use server";` and export the action
functions directly — **not** wrapped in `toResponse`; server actions can
throw and the caller (usually a client component's submit handler in a
try/catch) is responsible for catching. Service files (`src/services/*.ts`)
also start with `"use server";` but in Shape-A apps (see
services-and-errors.md) their exports always return `ServiceResponse<T>` via
`toResponse`, never throw to the caller. Don't blur the two: a server action
is a thin entrypoint (often just calling a service and re-throwing or
translating its error), a service is where the actual business logic and
permission checks live.

```ts
// src/actions/auth.ts
"use server";
import { headers } from "next/headers";
import { auth } from "@/lib/auth/server";

export async function signOutAction() {
  await auth.api.signOut({ headers: await headers() });
}
```

(A client component can also just call `authClient.signOut()` directly from
`@/lib/auth/client` — see auth.md — without a server action at all; reach
for the action wrapper above only when the sign-out needs to also do
something server-only in the same request, e.g. invalidate app-specific
state beyond the session itself.)

## Server functions over API routes

Default to a server function (a `"use server"` service/action called
directly) for anything a component in this app needs from the server —
reads, mutations, all of it. Don't reach for `src/app/api/.../route.ts` as
the default way for the frontend to talk to the backend; that's the
pre-Server-Actions pattern (`fetch("/api/...")` from a client component) and
it reintroduces exactly what server functions exist to remove: a hand-rolled
HTTP boundary, manual JSON (de)serialization, and a second place
error-handling has to be reimplemented.

Reach for an actual route handler only when the request genuinely isn't
"a component calling a server function" — a few real cases:

- **The response isn't JSON/a React-serializable value.** A generated PDF, a
  file download with `Content-Disposition: attachment`, an image served with
  a specific `Content-Type` — a server action's return value goes through
  React's serialization and can't set response headers or stream bytes. This
  needs `new NextResponse(buffer, { headers: {...} })` from a route:

  ```ts
  // src/app/api/orders/[orderNumber]/invoice/route.ts
  export async function GET(_req: Request, { params }: { params: Promise<{ orderNumber: string }> }) {
    const { orderNumber } = await params;
    const { data: order } = await ordersService.get(orderNumber);
    if (!order) return new NextResponse("Not found", { status: 404 });
    const buffer = await generateInvoicePdf(order);
    return new NextResponse(new Uint8Array(buffer), {
      headers: { "Content-Type": "application/pdf", "Content-Disposition": `attachment; filename="${orderNumber}.pdf"` },
    });
  }
  ```

- **A long-lived streaming connection** (Server-Sent Events, or anything that
  holds the response open) — a server action is a single request/response
  RPC call, it can't push updates over time. This needs a route with
  `export const dynamic = "force-dynamic"` and a streaming `Response`.

- **An external caller that isn't this app's own frontend** — a webhook from
  a third-party service, a callback URL handed to an external integration, a
  public API another system consumes. Server functions are only invokable
  from this app's own client bundle; anything else needs a real HTTP
  endpoint to POST to.

- **The framework requires it** — `better-auth`'s own
  `api/auth/[...all]/route.ts` isn't a choice, the OAuth callback needs a
  real HTTP endpoint to land on.

If a route serves a byte response gated behind auth (file downloads,
generated documents), check the session **inside the route handler itself**
— don't assume `proxy.ts` already covers it. A `proxy.ts` matcher typically
excludes paths ending in a file extension so static assets pass through
untouched (see auth.md); a route like `/api/files/[filename]` whose own URL
ends in `.webp`/`.pdf` matches that same exclusion and never goes through
`proxy.ts` at all:

```ts
// The proxy.ts matcher excludes any path ending in a file extension (for static
// assets), which also matches this route's own URLs — so proxy.ts never runs
// here and this route must check auth itself.
export async function GET(_request: Request, { params }: { params: Promise<{ filename: string }> }) {
  const session = await auth();
  if (!session?.user) return new NextResponse("Unauthorized", { status: 401 });
  // ...read and return the file
}
```
