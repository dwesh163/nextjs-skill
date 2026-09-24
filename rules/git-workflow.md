# Commits, version bumps, and local dev environment

## Commit message format

`[tag] Capitalized imperative description` — **not** Conventional Commits
(`feat: ...`, `fix: ...`). The tag is lowercase inside square brackets, one
space, then a capitalized, imperative-mood description (no trailing period):

```
[feature] Add cart item quantity stepper
[fix] Correct header height CSS variable syntax for consistency
[refactor] Remove unused order status resolution logic
[chore] Bump dependencies in bun.lock
[version] Bump to 1.10.8 in package.json
```

Tag vocabulary, in order of how often each is actually used — stick to the
first five, they cover nearly everything:

| Tag | For |
|---|---|
| `feature` | New user-facing or API-facing capability |
| `fix` | Bug fix |
| `refactor` | Behavior-preserving code change |
| `chore` | Dependency bumps, config, CI/tooling changes, non-code housekeeping |
| `version` | The dedicated version-bump commit (see below) — nothing else in the diff |
| `docs` | Documentation-only change (rare, but keep it separate from `chore`) |

Never `feat`, `fix:`, `feature:`, or any Conventional-Commits-style colon
syntax — that format doesn't match this convention and breaks the
release-notes generator (see tooling.md's GitHub Actions/GitLab CI section,
which buckets commits into changelog sections by matching `^\S+ \[(tag)\]`
against the tag list above). A commit that slips in as `feat: ...` still
gets committed, it just silently falls into the changelog's generic
"Other" bucket instead of "✨ Features" — annoying, not fatal, but avoid it.

## Version-bump commits

A version bump is **its own commit, touching only `package.json`'s
`"version"` field** — nothing else in the diff, not bundled with the feature
or fix that motivated it:

```
[version] Bump to 1.10.8 in package.json
```
```diff
-  "version": "1.10.7",
+  "version": "1.10.8",
```

This matters mechanically, not just stylistically: the release workflow
(tooling.md) reads the version straight out of `package.json` on every push
to the main branch, checks whether a release for that version already
exists, and builds+releases only if it doesn't. **Pushing a `[version]`
commit to main is what triggers a release** — there's no separate manual
release step. Bump only once a batch of merged work is actually ready to
ship (typically after several `[feature]`/`[fix]`/`[refactor]` commits have
landed), not on every single commit — a version bump per commit would
trigger a release per commit, which defeats the point of batching. Follow
semver: patch for fixes/small refactors, minor for new backward-compatible
features, major for breaking changes.

## Prefer several small commits over one large one

Build a feature as a sequence of small, independently-tagged commits rather
than one commit bundling everything:

```
[add] Add validation schema for the new resource
[refactor] Wire the create/update services to use the new schema
[add] Add the create/edit form component
[update] Replace the old inline form with the new form component
```

rather than a single `[feature] Add resource management`. Two reasons this
matters here specifically, beyond the usual "smaller diffs review better":

- The release-notes generator turns each commit into its own changelog
  bullet — one giant commit becomes one vague bullet ("Add resource
  management"), where several scoped commits produce a changelog a reviewer
  or user can actually skim.
- Each commit gets tagged on its own merits (`refactor` vs `add` vs `fix`),
  which keeps the tag counts meaningful instead of everything defaulting to
  `feature` because that's the only tag that was true for *part* of a large
  commit.

This isn't a call to fragment unrelated changes into artificial pieces —
each commit should still be a coherent, working step. It's a call to *not*
squash a naturally multi-step change (schema → service → UI → wiring) into
one commit just to have fewer commits.

## Local dev environment

A `Makefile` + `docker-compose.dev.yml`, not an inline `docker run` one-liner
stuffed into `package.json`'s `dev` script. A raw `docker run --rm` line
works, but it's an unreadable wall of flags buried in a JSON string with no
way to discover it short of reading `package.json`, and it can't grow (a
second service, `.env` generation) without becoming worse. A `Makefile`
gives every one of those steps a name, documents itself via `make help`, and
scales to more than "start Postgres" without changing shape:

```makefile
SHELL := /bin/bash

ENV            ?= dev
SECRETS_FILE   ?= /path/to/secrets.yml
ENV_FILE       ?= .env
COMPOSE_FILE   ?= docker-compose.dev.yml
BUN            ?= bun
DOCKER_COMPOSE ?= docker compose

POSTGRES_USER     ?= postgres
POSTGRES_PASSWORD ?= postgres
POSTGRES_DB       ?= app_dev
POSTGRES_PORT     ?= 58432

export POSTGRES_USER POSTGRES_PASSWORD POSTGRES_DB POSTGRES_PORT

.PHONY: help
## Print this help
help:
	@echo "Available targets:"; grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort

.PHONY: env
## Generate .env from SECRETS_FILE plus the local Postgres vars above
env:
	@test -f "$(SECRETS_FILE)" || { echo "Missing $(SECRETS_FILE)"; exit 1; }
	@{ \
		echo "DATABASE_URL=postgresql://$(POSTGRES_USER):$(POSTGRES_PASSWORD)@localhost:$(POSTGRES_PORT)/$(POSTGRES_DB)?schema=public"; \
		# ...append secrets read out of SECRETS_FILE (e.g. via yq) here...
	} > $(ENV_FILE)

.PHONY: up
## Generate .env, start Postgres in the background, run the dev server
up: env
	$(DOCKER_COMPOSE) -f $(COMPOSE_FILE) up -d
	$(BUN) run dev

.PHONY: down
## Stop Postgres
down:
	$(DOCKER_COMPOSE) -f $(COMPOSE_FILE) down

.PHONY: clean
## Stop containers, remove volumes and the .next cache
clean:
	$(DOCKER_COMPOSE) -f $(COMPOSE_FILE) down -v
	rm -rf .next
```

```yaml
# docker-compose.dev.yml
services:
  postgres:
    image: postgres:18-alpine
    restart: unless-stopped
    ports:
      - "${POSTGRES_PORT:-58432}:5432"
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-postgres}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-postgres}
      POSTGRES_DB: ${POSTGRES_DB:-app_dev}
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

`env` is a separate target from `up` (not folded into it) so a developer can
regenerate `.env` alone after a secret rotates, without also restarting the
containers. `ENV ?= dev` as an overridable variable lets the same Makefile
read a different block/environment out of the secrets source
(`make ENV=staging env`) without duplicating targets. Keep `POSTGRES_PORT`
non-default (e.g. `58432`, not `5432`) to avoid colliding with a Postgres
instance already running natively on the developer's machine.

With the `Makefile` in place, `package.json`'s `dev` script goes back to
being plain: `bunx prisma generate && next dev` — `make up` is what a
developer actually runs, and it's what starts Postgres before calling into
that script.
