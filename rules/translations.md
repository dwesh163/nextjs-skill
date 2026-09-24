# Translations

`next-intl`, `src/messages/en.json` and `src/messages/fr.json` — check the
app's actual `defaultLocale`, it varies per project. The two files must
always stay structurally identical (same key paths in both) — never add a
key to one without the other. Extra locales (`de.json`, etc.) are added the
same way.

## `src/i18n/` is a folder, not a flat `i18n.ts`

Two files, cleanly split by what they need: `config.ts` is pure
constants — no I/O, safe to import from a client component, a server
component, or `proxy.ts` alike — and `request.ts` is the actual next-intl
wiring, which does need to read cookies/headers and so is server-only.

```ts
// src/i18n/config.ts
export type Locale = (typeof locales)[number];
export const locales = ["en", "fr"] as const;
export const defaultLocale: Locale = "fr"; // whatever the app's actual default is
```

```ts
// src/i18n/request.ts
import { getRequestConfig } from "next-intl/server";
import { getUserLocale } from "@/services/locale";

export default getRequestConfig(async () => {
  const locale = await getUserLocale();
  return { locale, messages: (await import(`../messages/${locale}.json`)).default };
});
```

`next.config.ts`'s `createNextIntlPlugin(...)` points at `request.ts`
specifically, not the folder — see tooling.md.

## Locale detection (`src/services/locale.ts` or `src/lib/locale.ts`)

```ts
"use server";
import { cookies, headers } from "next/headers";
import { defaultLocale, type Locale, locales } from "@/i18n/config";

const COOKIE_NAME = "locale";

export async function getUserLocale() {
  const cookieLocale = (await cookies()).get(COOKIE_NAME)?.value;
  if (cookieLocale && locales.includes(cookieLocale as Locale)) return cookieLocale as Locale;

  const acceptLanguage = (await headers()).get("accept-language");
  if (acceptLanguage) {
    const preferred = getPreferredLocale(acceptLanguage); // parses q-values, falls back to short locale
    if (preferred) return preferred;
  }
  return defaultLocale;
}

export async function setUserLocale(locale: Locale) {
  (await cookies()).set(COOKIE_NAME, locale);
}
```

## Root schema (larger, domain-heavy apps)

```
app        — global chrome: title, navigation, dropdown (header account menu), footer
actions    — generic, reusable vocabulary: plain words/verbs + shared action tables below
status     — every status enum, grouped by entity (status.product.*, status.orders.*)
entities   — static data per entity: names, field labels, page-agnostic strings
pages      — content that only exists on one route
errors     — generic error boundary + not-found messages
common     — cross-cutting shared strings (validation.*, actions like cancel/save)
```

Smaller apps flatten this to a handful of top-level namespaces (`home`,
`notFound`, `error`, `common`) keyed roughly by page/feature instead of by
the app/actions/status/entities split above — match the granularity already
in the repo rather than importing a full multi-domain schema into a small
app.

**`entities.*` holds data, never actions.** An entity block has
`singular`/`plural` (lowercase nouns) and `title: { singular, plural }`
(capitalized display forms). Never add an `actions` sub-key under an entity —
action messages belong in the shared `actions` table, parameterized by
`{entity}`/`{entities}`, not duplicated per entity.

## The `actions` tables

Shared, generic tables keyed by verb (`create`, `update`, `delete`,
`bulk.delete`, `publish`, `archive`, ...). Never use compound camelCase keys
like `createError` — nest instead (`error.create`, `bulk.delete`):

```json
"actions": {
  "success": { "create": "...", "update": "...", "bulk": { "delete": "..." } },
  "error":   { "create": "...", "update": "...", "bulk": { "delete": "..." } },
  "dialog":  { "create": "...", "delete": { "title": "...", "description": "..." } }
}
```

Call with `translations.actions("success.create", { Entity, entity })`.
`success`/`dialog.create` use `{Entity}` (capitalized, sentence start);
`error`/`dialog.delete` use `{entity}`/`{entities}` (lowercase, mid-sentence).
Bulk/count messages take `{count}` and work for both a single item
(`count: 1`) and a real bulk selection — don't write separate single/bulk
copies of the same message.

Compute `entity`/`entities`/`Entity` once per component from the entity's own
translation block, not re-derived ad hoc at each call site (and `gender`
alongside them — see below).

## Gender: store it on the entity, agree via ICU `select`

French grammatical gender (créé/créée, ce/cette, un/une) is a property of the
**entity**, not something to design copy around. Every `entities.*` block
carries a `gender` field alongside `singular`/`plural`/`title`:

```json
"entities": {
  "products": { "singular": "produit", "plural": "produits", "gender": "male", "title": { "singular": "Produit", "plural": "Produits" } },
  "categories": { "singular": "catégorie", "plural": "catégories", "gender": "female", "title": { "singular": "Catégorie", "plural": "Catégories" } }
}
```

Shared `actions` messages that need agreement take `{gender}` as an ICU
`select` alongside `{Entity}`/`{entity}`:

```json
"success": {
  "create": "{Entity} {gender, select, male {créé} other {créée}} avec succès"
},
"dialog": {
  "delete": {
    "title": "Supprimer {article} {entity} ?",
    "description": "{gender, select, male {Ce} other {Cette}} {entity} sera définitivement {gender, select, male {supprimé} other {supprimée}}."
  }
}
```

Compute `gender` once per component alongside `entity`/`Entity`, from the
same entity translation block:

```ts
const entity = translations.product("singular");
const Entity = translations.product("title.singular");
const gender = translations.product("gender");
// translations.actions("success.create", { Entity, gender })
```

`{gender, select, male {...} other {...}}` is the pattern — `other` covers
`female` (ICU `select` only special-cases the keys you name; anything else
falls through to `other`, so `female` doesn't need its own branch unless a
third grammatical gender is ever in play). This composes with ICU use for
grammatical **number** in French prepositions
(`{number, select, plural {aux} other {au}}`) as a second `select` in the
same string when a message needs both.

## Don't duplicate translated text

Before adding a key, check whether the same string (or the same idea with a
different noun) already exists. When two strings differ only by an entity
name, that's the signal to parametrize with `{entity}`/`{Entity}` instead of
writing a new key — not a reason to keep two near-identical keys around.

## Component-side conventions

**Always** bind translations through an object, never a bare hook:

```ts
// ✅
const translations = {
  actions: useTranslations("actions"),
  product: useTranslations("entities.products"),
};

// ❌ never do this, even for a single namespace
const translations = useTranslations("pages.admin.banners");
const actions = useTranslations("actions"); // ❌ second top-level binding
```

Every `useTranslations`/`getTranslations` call must be a property of the
same `translations` object — no sibling `const foo = useTranslations(...)`
declarations, even if there's only one namespace to bind.

Never render `toast.error(error)` with a raw service error string — map the
boolean presence of an error to a translated `actions.error.*` message
instead (see services-and-errors.md for why the raw message is unsafe to
display).

Before committing hand-written nested ICU (`select`/`plural`), sanity-check
the syntax with `@formatjs/icu-messageformat-parser` — it's valid but easy to
get subtly wrong by hand.
