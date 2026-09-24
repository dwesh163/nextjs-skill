# Validation

`zod` schemas live in `src/validations/<domain>.ts`, never in
`src/types/<domain>.ts` (which stays type-only). Check which major version
of `zod` a project is actually on before writing new schemas — v4's
`z.string({ error: msg })` error-message syntax differs from v3's
`z.string({ message: msg })` / `z.string().email(msg)`; don't assume v4 just
because it's newer.

## Factory pattern for localized messages

When a schema's error messages need to be localized (any form the user
directly interacts with), export a **factory** instead of a bare schema —
`buildXSchema(msgs)` — so both a client form and a server-side
defense-in-depth check can build it with their own message source:

```ts
// src/validations/common.ts — shared primitives every domain schema can build on
import { z } from "zod";

export type CommonSchemaMessages = {
  required: string;
  invalidEmail: string;
};

// Server-side default messages (English) — defense-in-depth; the client always
// validates first with translated messages from the factory below.
export const defaultSchemaMessages: CommonSchemaMessages = {
  required: "Required",
  invalidEmail: "Invalid email",
};

export function requiredString(msgs: Pick<CommonSchemaMessages, "required">) {
  return z.string().min(1, msgs.required);
}
```

```ts
// src/validations/organisation.ts
import { requiredString, type CommonSchemaMessages, defaultSchemaMessages } from "./common";

export function buildCreateOrganisationSchema(msgs: CommonSchemaMessages) {
  return z.object({
    id: requiredString(msgs),
    displayName: requiredString(msgs),
    description: z.string().optional(),
  });
}

// A non-localized instance for server-side/defense-in-depth use.
export const createOrganisationSchema = buildCreateOrganisationSchema(defaultSchemaMessages);
export type CreateOrganisationValues = z.infer<typeof createOrganisationSchema>;
```

If a schema has no user-facing message at all (server-only payload shape,
e.g. an internal update DTO), a plain exported schema is fine — don't wrap it
in a factory for its own sake. Same schema, different input normalization
(e.g. a raw HTML `<input type="number">` string vs. the server's numeric
payload) gets two factories sharing the same message parameter type, not one
schema doing double duty with `.transform()` gymnastics.

## `useValidationMessages` hook

The client-side half of the factory pattern — binds `next-intl` translations
into the shape a schema factory expects:

```ts
// src/hooks/use-validation-messages.ts
"use client";
import { useTranslations } from "next-intl";
import { useMemo } from "react";
import type { CommonSchemaMessages } from "@/validations/common";

export function useValidationMessages(): CommonSchemaMessages {
  const t = useTranslations("common.validation");
  return useMemo(() => ({ required: t("required"), invalidEmail: t("invalidEmail") }), [t]);
}
```

## Wiring into a form (`react-hook-form` + `@hookform/resolvers/zod`)

```tsx
"use client";
const validationMessages = useValidationMessages();
const schema = useMemo(() => buildCreateOrganisationSchema(validationMessages), [validationMessages]);
const { register, handleSubmit, formState: { errors } } = useForm<CreateOrganisationValues>({
  resolver: zodResolver(schema),
  defaultValues: { id: "", displayName: "", description: "" },
});
```

Rebuild the schema with `useMemo` keyed on the translated messages, not once
at module scope — otherwise a locale switch won't update validation error
text. Submit handlers wrap the mutation in `useTransition`, `toast.success`/
`toast.error` on the result, and `router.refresh()` (or close a dialog +
`reset()`) on success — never render a raw thrown/service error string in the
toast (see services-and-errors.md).

## Auto-generated zod from the Prisma schema (optional, larger apps)

Some apps additionally run `prisma-zod-generator` to emit "pure model" zod
schemas straight from `schema.prisma` into `generated/zod/` — useful for
runtime-validating a full Prisma model shape, not a substitute for the
hand-written, localized form schemas above. Config lives in
`src/prisma/zod-generator.config.json`:

```json
{
  "mode": "custom",
  "useMultipleFiles": true,
  "pureModels": true,
  "optionalFieldBehavior": "optional",
  "variants": { "pure": { "enabled": true }, "input": { "enabled": false }, "result": { "enabled": false } }
}
```

Only reach for this when a domain genuinely needs schema-shaped runtime
validation of a full Prisma model (e.g. validating an external payload against
a table's columns) — most forms are better served by the hand-written factory
schema above, which can express messages, cross-field rules, and
client/server input differences the generator can't.
