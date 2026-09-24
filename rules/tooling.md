# Tooling: Biome, shadcn, Docker, CI/CD, root layout

## Package manager & scripts

Bun, not npm/pnpm/yarn — `bun.lock` committed, `bunx` for one-off runners.
Standard scripts:

```json
"scripts": {
  "dev": "next dev",
  "build": "next build",
  "start": "next start",
  "lint": "biome check",
  "format": "biome format --write"
}
```

(`dev`/`build` get prefixed with Prisma generation when the Prisma module is
present — see prisma.md.)

## Biome — not ESLint, not Prettier

```json
{
  "$schema": "https://biomejs.dev/schemas/2.4.6/schema.json",
  "vcs": { "enabled": true, "clientKind": "git", "useIgnoreFile": true },
  "files": { "ignoreUnknown": true, "includes": ["**", "!src/components/ui", "!node_modules", "!.next"] },
  "formatter": { "enabled": true, "indentStyle": "space", "indentWidth": 2, "lineWidth": 120 },
  "linter": { "enabled": true, "rules": { "recommended": true, "suspicious": { "noUnknownAtRules": "off" } } },
  "javascript": { "formatter": { "quoteStyle": "double", "trailingCommas": "all", "semicolons": "always" } },
  "css": { "parser": { "cssModules": true, "tailwindDirectives": true }, "formatter": { "enabled": true } },
  "assist": { "enabled": true, "actions": { "source": { "organizeImports": "on" } } }
}
```

`src/components/ui` (shadcn-generated primitives) is always excluded from
linting — it's generated code, don't fight its style or "fix" it to match
Biome's rules. `noUnknownAtRules: "off"` accommodates Tailwind's `@apply`/
`@theme` at-rules. Double quotes, trailing commas, semicolons always, 2-space
indent, 120-col lines — don't introduce Prettier or an ESLint config
alongside this.

CI runs `bunx biome ci --reporter=github ./src` (fails the build) and
separately `bunx biome lint --reporter=json ./src` uploaded as an artifact
for review — see the GitHub Actions section below.

## shadcn/ui and theming

`components.json`, adding/customizing components, `cn()`, and the
light/dark theme system (CSS variables, `next-themes`, the root layout's
`ThemeProvider`) have their own file — see ui.md.

## `next.config.ts`

```ts
import type { NextConfig } from "next";
import createNextIntlPlugin from "next-intl/plugin";

const withNextIntl = createNextIntlPlugin("./src/i18n/request.ts");

const nextConfig: NextConfig = {
  output: "standalone", // needed for the Docker runner stage, see below
  outputFileTracingRoot: __dirname,
};

export default withNextIntl(nextConfig);
```

`serverActions.bodySizeLimit` gets bumped in the `experimental` block for
apps that accept large file uploads through a server action. `images.remotePatterns`
gets an explicit allowlist per external image host actually used — don't use
a wildcard hostname.

## Root layout wiring order

better-auth needs no provider in the root layout at all (its client is
store-backed — see auth.md), so the layout stays about as thin as this
stack gets. `NextIntlClientProvider` wraps everything (it's what makes
translations available to every client component below it); `TooltipProvider`
(shadcn) wraps `children` for hover-card/tooltip primitives used anywhere in
the tree; `Toaster` is a sibling, not a wrapper, since it renders its own
portal rather than providing context. Any app-specific `providers/*` wrapper
(see file-organization.md) nests between those two, outer to inner in
whatever order one depends on another:

```tsx
export const dynamic = "force-dynamic";

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const locale = await getLocale();
  const messages = await getMessages();

  return (
    <html lang={locale} className={cn(inter.variable, localFont.variable)}>
      <body className="antialiased font-sans">
        <NextIntlClientProvider messages={messages}>
          <TooltipProvider>{children}</TooltipProvider>
          <Toaster position="bottom-center" />
        </NextIntlClientProvider>
      </body>
    </html>
  );
}
```

`export const dynamic = "force-dynamic";` — `getLocale()` reads a cookie/the
`Accept-Language` header on every request (see translations.md), so static
optimization of the root layout isn't applicable; declare it explicitly
rather than relying on Next's automatic detection.

This is the baseline. An app with a light/dark theme toggle adds one more
outermost wrapper (`ThemeProvider` from `next-themes`) plus
`suppressHydrationWarning` on `<html>` — see ui.md, which shows the full
layout with that addition rather than repeating it here.

## Dockerfile — multi-stage, Bun build + Node runtime, standalone output

```dockerfile
FROM oven/bun:1-slim AS deps
WORKDIR /app
COPY package.json bun.lock* ./
RUN bun install --frozen-lockfile

FROM oven/bun:1-slim AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN node node_modules/.bin/next build

FROM node:22-slim AS runner
WORKDIR /app
RUN groupadd --system --gid 1001 nodejs && useradd --system --uid 1001 --gid nodejs nextjs
ENV NODE_ENV=production NEXT_TELEMETRY_DISABLED=1 PORT=3000 HOSTNAME="0.0.0.0"
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
COPY --from=builder --chown=nextjs:nodejs /app/public ./public
USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

Runs the built server as a non-root user, copies only the `standalone`
output (requires `output: "standalone"` in `next.config.ts`) plus static
assets and `public/` — not the full `node_modules`. `next build` is invoked
via `node node_modules/.bin/next build` rather than `bunx next build` in the
builder stage (avoids Bun re-resolving/downloading in a stage that already
has `node_modules` installed). The runner stage is `node:22-slim`, not
`oven/bun` — the Bun image ships no `node` binary, so `CMD ["node",
"server.js"]` fails on it; `deps`/`builder` stay Bun-based, only the actual
runtime needs Node.

## GitHub Actions — reusable workflows

Two callable workflows (`_lint.yaml`, `_build.yaml`) invoked from `pr.yaml`
and `release.yaml`, so lint/build logic is defined once:

**`_lint.yaml`** — Biome CI check (fails on lint errors) + a separate
non-failing lint report uploaded as an artifact:

```yaml
on:
  workflow_call:
jobs:
  lint:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v5
      - uses: oven-sh/setup-bun@v2
        with: { bun-version: latest }
      - run: bun install --frozen-lockfile
      - run: bunx biome ci --reporter=github ./src
      - if: always()
        run: bunx biome lint --reporter=json ./src > biome-report.json || true
      - if: always()
        uses: actions/upload-artifact@v7
        with: { name: biome-lint-report, path: biome-report.json, retention-days: 7 }
```

**`_build.yaml`** — takes `tags`/`version` as inputs, builds and pushes a
Docker image, outputs whether it succeeded (consumed by the release job that
creates the GitHub release only if the push actually worked):

```yaml
on:
  workflow_call:
    inputs:
      tags: { type: string, required: true }
      version: { type: string, required: true }
    outputs:
      build_success: { value: ${{ jobs.build.outputs.build_success }} }
jobs:
  build:
    runs-on: ubuntu-24.04
    permissions: { contents: read, packages: write }
    outputs: { build_success: ${{ steps.result.outputs.success }} }
    steps:
      - uses: actions/checkout@v5
      - run: jq --arg v "${{ inputs.version }}" '.version = $v' package.json > package.json.tmp && mv package.json.tmp package.json
      - uses: docker/setup-buildx-action@v4
      - uses: docker/login-action@v4
        with: { registry: ghcr.io, username: ${{ github.actor }}, password: ${{ secrets.GITHUB_TOKEN }} }
      - uses: docker/build-push-action@v7
        with: { context: ., push: true, tags: ${{ inputs.tags }}, platforms: linux/amd64, cache-from: type=gha, cache-to: "type=gha,mode=max" }
      - id: result
        run: echo "success=true" >> $GITHUB_OUTPUT
```

Use `ghcr.io/${{ github.repository }}` (lowercased) as the default registry
for a new project; swap in an org's private registry (e.g. Quay) by changing
only the `login-action`'s `registry`/credentials and the tag prefix — the
rest of both workflows stays identical.

Always all four files — `_lint.yaml`, `_build.yaml`, `pr.yaml`,
`release.yaml` — even on a small, single-workflow project. Collapsing them
into one combined file saves two files but loses the reuse (`pr.yaml` and
`release.yaml` both call the same `_lint`/`_build`, so lint/build logic is
defined exactly once) for no real benefit; keep the split from the start
rather than starting collapsed and migrating later.

**`pr.yaml`** — computes a `<branch>-<short-sha>` image tag, runs lint, then
build, both as reusable-workflow calls, build gated on lint passing:

```yaml
name: Pull Request

on:
  pull_request:

jobs:
  prepare:
    name: Prepare
    runs-on: ubuntu-24.04
    outputs:
      tag: ${{ steps.compute.outputs.tag }}
      version: ${{ steps.compute.outputs.version }}
    steps:
      - name: Compute image tag
        id: compute
        run: |
          IMAGE=$(echo "ghcr.io/${{ github.repository }}" | tr '[:upper:]' '[:lower:]')
          BRANCH=$(echo "${{ github.head_ref }}" | sed 's/[^a-zA-Z0-9._-]/-/g')
          SHA="${{ github.sha }}"
          VERSION="${BRANCH}-${SHA:0:7}"
          echo "tag=${IMAGE}:${VERSION}" >> $GITHUB_OUTPUT
          echo "version=${VERSION}" >> $GITHUB_OUTPUT

  lint:
    uses: ./.github/workflows/_lint.yaml

  build:
    name: Build and Push
    needs: [prepare, lint]
    uses: ./.github/workflows/_build.yaml
    with:
      tags: ${{ needs.prepare.outputs.tag }}
      version: ${{ needs.prepare.outputs.version }}
    secrets: inherit
```

**`release.yaml`** — triggered on push to `main`. Detects the version,
skips everything else if that version is already released (this is what
makes it safe to just merge PRs without a separate manual release step),
builds/pushes `latest` + the version tag, then creates the GitHub release.
Version detection and release-note generation are delegated to
[`dwesh163/actions`](https://github.com/dwesh163/actions) (`detect-version`,
`release`) instead of hand-rolled `jq`/`gh` steps — reuse the same composite
actions across every repo on this workflow rather than re-deriving the
version-detection/changelog logic per project:

```yaml
name: Release

on:
  push:
    branches: [main]

jobs:
  lint:
    uses: ./.github/workflows/_lint.yaml

  detect-version:
    name: Detect Version
    runs-on: ubuntu-24.04
    permissions:
      contents: read
    outputs:
      version: ${{ steps.detect.outputs.version }}
      tags: ${{ steps.tags.outputs.tags }}
      should_release: ${{ steps.detect.outputs.release_exist == 'false' }}
    steps:
      - name: Checkout
        uses: actions/checkout@v5

      - name: Detect version
        id: detect
        uses: dwesh163/actions/detect-version@main
        with:
          type: js
          github-token: ${{ secrets.GITHUB_TOKEN }}

      - name: Compute image tags
        id: tags
        run: |
          IMAGE=$(echo "ghcr.io/${{ github.repository }}" | tr '[:upper:]' '[:lower:]')
          {
            echo "tags<<EOF"
            echo "${IMAGE}:latest"
            echo "${IMAGE}:${{ steps.detect.outputs.version }}"
            echo "EOF"
          } >> $GITHUB_OUTPUT

  build:
    name: Build and Push
    needs: [lint, detect-version]
    if: needs.detect-version.outputs.should_release == 'true'
    permissions:
      contents: read
      packages: write
    uses: ./.github/workflows/_build.yaml
    with:
      tags: ${{ needs.detect-version.outputs.tags }}
      version: ${{ needs.detect-version.outputs.version }}
    secrets: inherit

  create-release:
    name: Create Release
    needs: [detect-version, build]
    if: needs.build.outputs.build_success == 'true'
    runs-on: ubuntu-24.04
    permissions:
      contents: write
    steps:
      - name: Checkout
        uses: actions/checkout@v5
        with:
          fetch-depth: 0

      # A GitHub App token, not secrets.GITHUB_TOKEN: the default token's commits/releases
      # don't trigger other workflows and can't reach outside this repo — needed the moment
      # release automation has to touch a second repo (see the chart-bump extension below).
      - name: Generate release app token
        id: release-app-token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.GH_APP_ID }}
          private-key: ${{ secrets.GH_APP_SECRET }}

      - name: Create release
        uses: dwesh163/actions/release@main
        with:
          version: ${{ needs.detect-version.outputs.version }}
          github-token: ${{ steps.release-app-token.outputs.token }}
```

`detect-version`'s `type: js` reads the version straight out of
`package.json`; `release` generates the same tagged-commit changelog
described in git-workflow.md (`[feature]`/`[fix]`/`[refactor]`/`[chore]`/
`[docs]`, bucketed into the same sections) and creates the GitHub release —
match that commit-tagging convention for the generated notes to be useful.

**Extension point — bumping a downstream Helm chart repo.** When the app is
deployed via a Helm chart that lives in a separate repo, append one more job
after `create-release` rather than changing anything about the four files
above:

```yaml
  bump-chart:
    name: Bump Chart
    needs: [detect-version, create-release]
    runs-on: ubuntu-24.04
    steps:
      - name: Bump chart version
        uses: dwesh163/actions/bump-chart@main
        with:
          chart-repo: <org>/charts
          chart-name: <app-name>
          app-version: ${{ needs.detect-version.outputs.version }}
          github-app-id: ${{ secrets.GH_APP_ID }}
          github-app-private-key: ${{ secrets.GH_APP_SECRET }}
```

This is the shape every extension to `release.yaml` should take: a new job,
`needs: [..., create-release]`, appended at the end — never a change to
`_lint.yaml`/`_build.yaml`'s reusable contract (their `workflow_call`
inputs/outputs are what `pr.yaml` and `release.yaml` both depend on) and
never a restructuring of `detect-version`'s outputs just to feed one more
downstream job.

## GitLab CI (when the project is on GitLab instead of GitHub)

Same shape as the GitHub Actions workflows above, ported to
`.gitlab-ci.yml` — four stages (`lint` → `detect` → `build` → `release`):

```yaml
stages: [lint, detect, build, release]
variables:
  REGISTRY_PATH: 'registry.example.com/my-org/my-app'

lint:
  stage: lint
  image: node:22-alpine   # swap to a Bun image if the runners support it — see note below
  before_script:
    - npm install
    - npx prisma generate
  script:
    - npm run lint
    - npx tsc --noEmit

build:
  stage: build
  image: docker:latest
  services: [docker:dind]
  variables:
    DOCKER_HOST: tcp://docker:2375   # dind runs as a sibling container, not a mounted socket
    DOCKER_TLS_CERTDIR: ''
  before_script:
    - echo "$REGISTRY_PASSWORD" | docker login -u "$REGISTRY_USERNAME" --password-stdin registry.example.com
  script:
    - docker build -t $REGISTRY_PATH:latest -t $REGISTRY_PATH:$VERSION . && docker push $REGISTRY_PATH:latest && docker push $REGISTRY_PATH:$VERSION
```

`detect` (a release-existence check via the platform's Releases API, with a
`dotenv` artifact passing `VERSION`/`SHOULD_BUILD` between jobs) and
`release` (POSTs to that same Releases API) fill the same role as
`detect-version`/`create-release` in the GitHub Actions version — same
policy (skip the build if the version's release already exists), different
platform API. Port the *shape* (lint → detect-version → build → release,
skip-if-already-released) rather than trying to force GitHub Actions syntax
into `.gitlab-ci.yml`.

If CI runners can't run Bun at all (some older/restricted runner images
lack the CPU features Bun's baseline build needs), fall back to `node`/`npm`
for the lint job specifically rather than blocking on it — local dev and the
Docker build can still be Bun-based; only the CI runner needs the Node
fallback.

## `.env` / `.env.example`

Every secret-bearing var goes in `.env.example` with a placeholder value
(never a real secret, even a dev one) and is duplicated into a real `.env`
only for local dev (gitignored). `AUTH_SECRET` is generated fresh per project
(`randomBytes(32).toString("base64")`), never copied from another project's
`.env`.
