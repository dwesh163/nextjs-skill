# nextjs-skill

A [Claude Code](https://claude.com/claude-code) skill capturing house
conventions for building Next.js (App Router) apps with TypeScript, Bun,
Biome, Prisma, better-auth, and next-intl — auth, file organization, the
service layer, permissions, billing, i18n, validation, CI/CD, and git
workflow.

## What this is

Not a starter template — a set of documented conventions Claude Code loads
when working in a matching project, so generated or edited code follows the
same shape consistently instead of reinventing structure every time. See
[`SKILL.md`](./SKILL.md) for the full index and the two dimensions
(database ownership, permission source) that decide which variant of a
convention applies to a given app.

## Install

Symlink (or clone) this repo into your Claude Code skills directory:

```
ln -s /path/to/nextjs-skill ~/.claude/skills/nextjs-skill
```

Or drop it into a project's own `.claude/skills/` directory to scope it to
that one repo instead of every session.

## What's covered

- [`rules/file-organization.md`](rules/file-organization.md) — `src/` layout, domain mirroring, the `providers/` folder, generic component folders, naming conventions
- [`rules/auth.md`](rules/auth.md) — better-auth setup, OIDC providers, DB-backed vs. stateless sessions, `proxy.ts`, token refresh, optional plugins (two-factor/TOTP)
- [`rules/stripe.md`](rules/stripe.md) — `@better-auth/stripe` billing, per-user and per-organization subscriptions, the `authorizeReference` permission check
- [`rules/permissions.md`](rules/permissions.md) — IdP-groups, app-local, delegated, and org-scoped authorization — which case applies when
- [`rules/services-and-errors.md`](rules/services-and-errors.md) — the service layer, typed errors, the not-found/error-boundary trio
- [`rules/client-components.md`](rules/client-components.md) — server vs. client component boundary, server functions vs. API routes
- [`rules/validation.md`](rules/validation.md) — zod schema-factory pattern for localized forms
- [`rules/translations.md`](rules/translations.md) — next-intl message schema and conventions
- [`rules/prisma.md`](rules/prisma.md) — schema layout, client singleton, CLI-generated auth tables
- [`rules/ui.md`](rules/ui.md) — shadcn/ui components, `cva` variants, `cn()`, light/dark theming with `next-themes`
- [`rules/tooling.md`](rules/tooling.md) — Biome, Docker, GitHub Actions
- [`rules/git-workflow.md`](rules/git-workflow.md) — commit message format, version bumps, local dev environment

## Updating

These are living documents. When a convention changes, update the matching
file in the same change — don't let this drift from what the code actually
does.
