---
name: nextjs-skill
description: House conventions for Next.js App Router apps built with TypeScript, Bun, Biome, Prisma, better-auth and next-intl. Use when scaffolding a new Next.js project, or writing/reviewing code in one that follows this stack — auth and the proxy.ts route guard, the service layer and typed errors, file organization, i18n, zod validation, permissions, shadcn/ui and theming, or Docker/GitHub Actions CI.
metadata:
  tags: nextjs, react, typescript, prisma, better-auth, next-intl, app-router, bun, biome, zod
---

## When to use

Load this skill whenever you're touching a Next.js (App Router) project that
matches this stack: TypeScript, Bun, Biome (not ESLint/Prettier), Tailwind +
shadcn/ui, `next-intl`, `better-auth`, Prisma, `zod`. Also use it when asked
to scaffold a new Next.js project from scratch — these are the conventions to
reproduce.

These conventions are distilled from a range of production Next.js apps, from
small internal tools to full multi-domain platforms, plus a scaffolding CLI
that bootstraps new projects into this same shape. They're opinionated and
consistent by default — match the existing shape in whatever repo you're in
rather than inventing a new one.

Not every app needs every piece described here. Two dimensions vary the most
from project to project, and each reference file below says so explicitly
where it matters:

- **Does the app own its database, or proxy another service?** A Prisma-backed
  app and a thin BFF over an external REST API need a different service-layer
  shape — see services-and-errors.md.
- **Where do permissions come from — an IdP's groups, or something local?**
  Group/role claims from an OIDC provider and app-local or delegated
  authorization need different wiring — see permissions.md.

Pick the case that matches the project you're in; don't import machinery
(better-auth's access-control plugin, `ServiceResponse`, a database at all)
that a smaller app doesn't need.

In an existing repo, the code itself usually answers which case applies —
read it, don't ask what's already on disk. But if the repo is ambiguous or
only partially set up (e.g. no Prisma schema yet but unclear if that's
intentional, or an i18n folder with only one locale ever added to it), don't
guess which way to extend it — ask directly, same as the fresh-scaffold case
below.

Scaffolding a fresh project is the clearer version of the same problem:
there's no code to read at all, so don't silently default to the full stack
table below. Ask the user first, as a short batch of questions rather than
one at a time:

- **Database** — does it own data via Prisma, or is it a thin BFF/proxy with
  no local DB (services-and-errors.md's Shape B, see prisma.md)?
- **i18n** — multiple locales via `next-intl`, or single-language with no
  translation layer at all (see translations.md)?
- **Auth** — needed at all, and if so DB-backed vs stateless sessions, which
  OIDC provider (see auth.md)?
- **Permissions** — IdP groups, app-local/delegated roles, org-scoped
  multi-tenant, or nothing beyond "logged in" (see permissions.md)?
- **Billing** — `@better-auth/stripe`, per-user or per-organization, or not
  needed (see stripe.md)?

Only wire in what the user actually confirms — this is the same
"don't import machinery a smaller app doesn't need" rule above, applied
before any code exists instead of after.

If a repo has its own `rules/`, `CLAUDE.md` or `AGENTS.md`, that repo's
version wins over this skill where the two disagree; those are living
documents for that specific codebase.

## Stack at a glance

| Concern | Choice |
|---|---|
| Framework | Next.js 16, App Router, TypeScript, `src/` dir, `@/*` alias |
| Package manager | Bun (`bun.lock`, `bunx`) |
| Lint/format | Biome (`biome.json`) — no ESLint, no Prettier |
| Styling | Tailwind v4 + shadcn/ui (`components.json`, `style: new-york`), `cva`, `next-themes` — see ui.md |
| Auth | `better-auth`, DB-backed sessions, OIDC provider, `src/proxy.ts` guard |
| i18n | `next-intl`, `en`/`fr`, cookie + `Accept-Language` detection |
| Database | Prisma (7.x), `pg` adapter, schema at `src/prisma/schema.prisma` |
| Validation | `zod` v4, schema-factory pattern for localized messages |
| Permissions | IdP groups → better-auth's `admin` plugin + access control, or app-local/delegated — see permissions.md |
| Forms | `react-hook-form` + `@hookform/resolvers/zod` |
| Containers | Multi-stage Bun `Dockerfile`, `output: "standalone"` |
| CI/CD | GitHub Actions, reusable `_lint.yaml`/`_build.yaml`, release-on-version-bump |
| Commits | `[tag] Capitalized description` — not Conventional Commits — see git-workflow.md |
| Local dev DB | `make up` — `Makefile` + `docker-compose.dev.yml`, not inline `docker run` |
| Billing (optional) | `@better-auth/stripe`, per-user or per-organization — see stripe.md |

## Reference files

Read the matching file before touching that area of a codebase:

- [rules/file-organization.md](rules/file-organization.md) — `src/` layout, domain mirroring across `types`/`validations`/`services`/`components`, the `providers/` folder, generic cross-domain component folders (`dialog/`, ...), naming conventions
- [rules/auth.md](rules/auth.md) — `better-auth` setup, OIDC providers, DB-backed vs stateless sessions, `proxy.ts` route guard, token refresh, `getUser()`/`session()`, optional plugins (two-factor/TOTP)
- [rules/services-and-errors.md](rules/services-and-errors.md) — the service layer (`ServiceResponse<T>`/`toResponse` vs. thin REST-BFF), the `ExpectedError` hierarchy, the not-found/error-boundary trio
- [rules/client-components.md](rules/client-components.md) — server vs. `"use client"` boundary, data-fetching split, server actions, and when a real `api/.../route.ts` is still justified over a server function
- [rules/validation.md](rules/validation.md) — zod schema-factory pattern, shared `common.ts` schemas, wiring into `react-hook-form`
- [rules/translations.md](rules/translations.md) — `next-intl` message schema, ICU rules, gender handling, component-side conventions
- [rules/prisma.md](rules/prisma.md) — schema location, `prisma.config.ts`, client singleton, generated-client placement
- [rules/permissions.md](rules/permissions.md) — better-auth's access control for IdP-groups permissions vs. app-local/delegated authorization vs. org-scoped multi-tenant roles, route-level checks, when to reach for which
- [rules/ui.md](rules/ui.md) — `components.json`, adding/customizing shadcn components, `cva` variants, `cn()`, light/dark theming (CSS variables, `next-themes`, the theme toggle)
- [rules/tooling.md](rules/tooling.md) — Biome config, Dockerfile, GitHub Actions workflows, root layout wiring, env files
- [rules/git-workflow.md](rules/git-workflow.md) — commit message format and tag vocabulary, version-bump commits and how they trigger a release, why to prefer several small commits, the Makefile/docker-compose local dev setup
- [rules/stripe.md](rules/stripe.md) — `@better-auth/stripe` billing, per-user vs per-organization subscriptions, the `authorizeReference` permission check, webhook routing

These are living documents: when you learn a convention doesn't hold in a new
repo, or a repo evolves one, update the matching file instead of silently
diverging.
