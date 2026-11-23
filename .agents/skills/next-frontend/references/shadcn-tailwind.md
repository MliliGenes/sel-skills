# shadcn/ui + Tailwind v4

Contents: components.json · cn helper · globals.css token skeleton · theming · component picker · CLI tips

## components.json

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "",
    "css": "src/app/globals.css",
    "baseColor": "neutral",
    "cssVariables": true
  },
  "iconLibrary": "lucide",
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  }
}
```

The empty `tailwind.config` is correct for Tailwind v4. `iconLibrary` decides which icons the CLI writes into added components; changing it later means re-adding or editing those components. `style` and available options depend on the CLI version: let `init` choose, don't hand-write this file unless repairing it.

## cn helper

```ts
// src/lib/utils.ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

## globals.css skeleton (Tailwind v4)

```css
@import "tailwindcss";
@import "tw-animate-css";

@custom-variant dark (&:is(.dark *));

:root {
  --radius: 0.625rem;
  --background: oklch(1 0 0);
  --foreground: oklch(0.145 0 0);
  --card: oklch(1 0 0);
  --card-foreground: oklch(0.145 0 0);
  --primary: oklch(0.205 0 0);
  --primary-foreground: oklch(0.985 0 0);
  --secondary: oklch(0.97 0 0);
  --secondary-foreground: oklch(0.205 0 0);
  --muted: oklch(0.97 0 0);
  --muted-foreground: oklch(0.556 0 0);
  --accent: oklch(0.97 0 0);
  --accent-foreground: oklch(0.205 0 0);
  --destructive: oklch(0.577 0.245 27.325);
  --border: oklch(0.922 0 0);
  --input: oklch(0.922 0 0);
  --ring: oklch(0.708 0 0);
}

.dark {
  --background: oklch(0.145 0 0);
  --foreground: oklch(0.985 0 0);
  --card: oklch(0.205 0 0);
  --card-foreground: oklch(0.985 0 0);
  --primary: oklch(0.922 0 0);
  --primary-foreground: oklch(0.205 0 0);
  --secondary: oklch(0.269 0 0);
  --secondary-foreground: oklch(0.985 0 0);
  --muted: oklch(0.269 0 0);
  --muted-foreground: oklch(0.708 0 0);
  --accent: oklch(0.269 0 0);
  --accent-foreground: oklch(0.985 0 0);
  --destructive: oklch(0.704 0.191 22.216);
  --border: oklch(1 0 0 / 10%);
  --input: oklch(1 0 0 / 15%);
  --ring: oklch(0.556 0 0);
}

@theme inline {
  --font-sans: var(--font-geist-sans);
  --font-mono: var(--font-geist-mono);

  --color-background: var(--background);
  --color-foreground: var(--foreground);
  --color-card: var(--card);
  --color-card-foreground: var(--card-foreground);
  --color-primary: var(--primary);
  --color-primary-foreground: var(--primary-foreground);
  --color-secondary: var(--secondary);
  --color-secondary-foreground: var(--secondary-foreground);
  --color-muted: var(--muted);
  --color-muted-foreground: var(--muted-foreground);
  --color-accent: var(--accent);
  --color-accent-foreground: var(--accent-foreground);
  --color-destructive: var(--destructive);
  --color-border: var(--border);
  --color-input: var(--input);
  --color-ring: var(--ring);

  --radius-sm: calc(var(--radius) - 4px);
  --radius-md: calc(var(--radius) - 2px);
  --radius-lg: var(--radius);
  --radius-xl: calc(var(--radius) + 4px);
}

@layer base {
  * { @apply border-border outline-ring/50; }
  body { @apply bg-background text-foreground; }
}
```

This is a shape reference. The CLI generates the real file (more tokens: popover, sidebar, chart colors); add to it rather than replacing it.

## Theming recipe

To re-theme the whole app, change only the CSS variables in `:root` and `.dark`:
- Brand color: edit `--primary` (and `--ring` to match). Keep `--primary-foreground` at readable contrast.
- Personality: `--radius` (0 = sharp/brutalist, 0.25rem = crisp, 1rem = soft) and `--font-sans`.
- Add a new semantic token (e.g. `--success`) in `:root` and `.dark`, then expose it in `@theme inline` as `--color-success`, so `bg-success` works.
- Define colors in OKLCH so lightness steps stay perceptually even between light and dark.
- Use a theme generator (the shadcn themes page or tweakcn) for a starting palette, then paste the variables.

Express any visual direction here instead of in scattered utility classes.

## Which shadcn component for what

| Need | Component |
|---|---|
| Buttons, links styled as buttons | `button` (`asChild` with `next/link`) |
| Forms | `form` or `field`, `input`, `textarea`, `select`, `checkbox`, `radio-group`, `switch`, `label` |
| Overlay / modal | `dialog`; mobile bottom sheet: `drawer` (vaul); side panel: `sheet`; confirm: `alert-dialog` |
| Menus | `dropdown-menu`, `context-menu`, `menubar`, `navigation-menu` |
| Command palette / combobox | `command` (cmdk), `popover` + `command` |
| Feedback | `sonner` toasts, `alert`, `skeleton`, `progress`, `badge` |
| Data | `table` + TanStack Table, `chart` (recharts), `pagination` |
| Layout | `sidebar`, `resizable`, `tabs`, `accordion`, `collapsible`, `separator`, `scroll-area`, `aspect-ratio` |
| Date | `calendar` (react-day-picker) + `popover` |
| Misc | `avatar`, `tooltip`, `hover-card`, `carousel` (embla), `input-otp`, `breadcrumb`, `toggle-group` |

## CLI tips

```bash
npx shadcn@latest add button            # add one
npx shadcn@latest add sidebar-07        # add a block (layout starting point)
npx shadcn@latest diff                  # check for upstream changes (if supported by installed CLI)
```

Use `-o` / `--overwrite` only when you intend to discard local edits to that component. shadcn also ships an MCP server (`npx shadcn@latest mcp init`) that lets an agent browse and install components from registries; suggest it if the user works in an MCP-capable editor.

## Gotchas

- Components importing from `@/components/ui/...` fail if the `@/*` alias isn't in `tsconfig.json` `paths`.
- Radix-based components that rely on `asChild` need a single child element that forwards refs.
- Tooltips need a `TooltipProvider` at or above the usage (the generated sidebar already includes one).
- Dark-mode utilities (`dark:`) rely on the `@custom-variant dark` line matching how `next-themes` is configured (`attribute="class"`).
