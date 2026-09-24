# UI: shadcn/ui and theming

## `components.json`

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": true,
  "tsx": true,
  "tailwind": { "css": "src/app/globals.css", "baseColor": "neutral", "cssVariables": true },
  "iconLibrary": "lucide",
  "aliases": { "components": "@/components", "utils": "@/lib/utils", "ui": "@/components/ui", "lib": "@/lib", "hooks": "@/hooks" }
}
```

`style: "new-york"`, `neutral` base color, Lucide icons, CSS-variable-driven
theming (`cssVariables: true` — see the theming section below, this is what
makes the light/dark split possible without a second set of utility
classes).

## Adding and customizing components

`bunx shadcn@latest add <component>` — never hand-write a component shadcn
already provides. Once added, though, the file is yours to customize:
shadcn isn't a package you import from `node_modules`, it's copy-and-own —
unlike Prisma's generated client (prisma.md) there's no regeneration step
that would wipe an edit. `components/ui/` being excluded from Biome and
called "generated" elsewhere in this skill (tooling.md,
file-organization.md) means "don't write a new one from scratch when the
CLI already provides it" and "don't let the linter fight its style," not
"never touch the file again" — adjusting a component's variants, default
props, or markup after adding it is normal and expected.

### Extending variants with `cva`

shadcn's own components (`Button`, `Badge`, ...) are built with
`class-variance-authority`; a domain-specific styled component follows the
same shape instead of inventing conditional `className` logic ad hoc:

```tsx
import { cva, type VariantProps } from "class-variance-authority";
import { cn } from "@/lib/utils";

const statusBadgeVariants = cva("inline-flex items-center rounded-full px-2 py-0.5 text-xs font-medium", {
  variants: {
    status: {
      active: "bg-green-100 text-green-800",
      pending: "bg-yellow-100 text-yellow-800",
      archived: "bg-neutral-100 text-neutral-600",
    },
  },
  defaultVariants: { status: "pending" },
});

export function StatusBadge({ status, className }: { status: "active" | "pending" | "archived"; className?: string }) {
  return <span className={cn(statusBadgeVariants({ status }), className)} />;
}
```

A component like this is generic and cross-domain (any entity with a status
can use it), so it belongs in `components/ui/` alongside shadcn's own
primitives, or in a generic folder like `components/dialog/` if it's more
of a composition than a primitive — see file-organization.md's "Generic
component folders" section for that distinction.

### `cn()` — the class-merging utility

```ts
// src/lib/utils.ts
import { type ClassValue, clsx } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

Every component that accepts a `className` prop and needs to merge it with
its own default classes uses `cn(...)` — never manual template-literal
string concatenation. `clsx` handles conditional classes (`cn("base", isActive && "active")`);
`twMerge` then resolves conflicting Tailwind utilities (two different
`px-*` values, say) by keeping the last one, instead of shipping both
classes to the DOM and letting CSS source order decide by accident.

## Theming: light/dark mode via CSS variables

`next-themes` (`bun add next-themes`) drives the light/dark/system toggle;
shadcn's `cssVariables: true` setup is what makes a single set of utility
classes (`bg-background`, `text-foreground`, ...) resolve to different
actual colors per theme, instead of needing `dark:bg-x` variants sprinkled
through every component.

### `globals.css` — token definitions

Tailwind v4 shape: light tokens under `:root`, dark tokens under `.dark`,
mapped into Tailwind's own color namespace via `@theme inline`. Exact token
names/values come from `bunx shadcn@latest init` (or `add` for a fresh
component that introduces new tokens) — this shows the *shape*, don't
hand-transcribe specific color values from here:

```css
@import "tailwindcss";
@import "tw-animate-css";

@custom-variant dark (&:is(.dark *));

:root {
  --radius: 0.625rem;
  --background: oklch(1 0 0);
  --foreground: oklch(0.145 0 0);
  --primary: oklch(0.205 0 0);
  --primary-foreground: oklch(0.985 0 0);
  /* ...every other token shadcn's init generates... */
}

.dark {
  --background: oklch(0.145 0 0);
  --foreground: oklch(0.985 0 0);
  --primary: oklch(0.922 0 0);
  --primary-foreground: oklch(0.205 0 0);
  /* ... */
}

@theme inline {
  --color-background: var(--background);
  --color-foreground: var(--foreground);
  --color-primary: var(--primary);
  --color-primary-foreground: var(--primary-foreground);
  --radius-sm: calc(var(--radius) - 4px);
  /* ... */
}

@layer base {
  * { @apply border-border outline-ring/50; }
  body { @apply bg-background text-foreground; }
}
```

Add a new *semantic* color (not a one-off component tweak) by adding the
token in both `:root` and `.dark`, then mapping it in `@theme inline` —
never hardcode a raw color value in a component when an existing token
already means the same thing (`text-foreground`, not
`text-[oklch(0.145_0_0)]`).

### `src/providers/theme.tsx` — the concrete instance of the generic pattern

This is the real, worked example of the schematic shown in
file-organization.md's `providers/` section — a thin `"use client"` wrapper
around a third-party client component, kept behind a stable, app-owned
import path:

```tsx
"use client";
import { ThemeProvider as NextThemesProvider, type ThemeProviderProps } from "next-themes";

export function ThemeProvider({ children, ...props }: ThemeProviderProps) {
  return <NextThemesProvider {...props}>{children}</NextThemesProvider>;
}
```

### Root layout wiring

Builds on tooling.md's baseline root layout — same
`NextIntlClientProvider`/`TooltipProvider`/`Toaster` nesting, with
`ThemeProvider` added as one more outermost wrapper. It goes outermost
because it sets a class on `<html>` before anything else needs to render,
and it doesn't depend on locale or any other provider (no ordering
constraint the way, say, a provider needing session data would have).
`suppressHydrationWarning` on `<html>` specifically (not anywhere else) is
required: `next-themes` sets the theme class via an inline script before
React hydrates, which otherwise trips a server/client markup mismatch
warning on that one element:

```tsx
export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const locale = await getLocale();
  const messages = await getMessages();

  return (
    <html lang={locale} suppressHydrationWarning className={cn(inter.variable, localFont.variable)}>
      <body className="antialiased font-sans">
        <ThemeProvider attribute="class" defaultTheme="system" enableSystem disableTransitionOnChange>
          <NextIntlClientProvider messages={messages}>
            <TooltipProvider>{children}</TooltipProvider>
            <Toaster position="bottom-center" />
          </NextIntlClientProvider>
        </ThemeProvider>
      </body>
    </html>
  );
}
```

`disableTransitionOnChange` stops every color-transitioning element from
visibly animating at once the instant the theme flips — a small but
noticeable polish detail, not required for correctness.

### The toggle itself

```tsx
// src/components/theme-toggle.tsx
"use client";
import { Moon, Sun } from "lucide-react";
import { useTheme } from "next-themes";
import { Button } from "@/components/ui/button";

export function ThemeToggle() {
  const { theme, setTheme } = useTheme();
  return (
    <Button variant="ghost" size="icon" onClick={() => setTheme(theme === "dark" ? "light" : "dark")}>
      <Sun className="dark:hidden" />
      <Moon className="hidden dark:block" />
    </Button>
  );
}
```

Flat at `components/theme-toggle.tsx`, not nested under a `theme/` folder —
consistent with file-organization.md's nest-instead-of-hyphenate rule:
"theme" would only earn its own folder if there were more than one file
under it, the way `providers/theme.tsx` (a different concern — the
provider, not the toggle UI) already has its own slot under `providers/`.
Same flat-at-the-root treatment as `header.tsx`/`footer.tsx`/`language.tsx`
— single-file, app-wide chrome pieces, not a domain.
