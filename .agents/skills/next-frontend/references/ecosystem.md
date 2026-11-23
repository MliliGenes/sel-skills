# Ecosystem catalog

Contents: icons · fonts · styling helpers · motion · forms & validation · state & URL · data & networking · UI extras · content · testing & tooling

Add libraries only when the feature needs them. Check bundle impact for anything imported on every page.

## Icons

| Library | Install | Notes |
|---|---|---|
| lucide-react (default) | `npm i lucide-react` | Named imports tree-shake. Type props with `LucideIcon`. Brand icons were removed: use another source for logos. |
| Phosphor | `npm i @phosphor-icons/react` | Six weights (thin to duotone): good when you want expressive variants |
| Tabler | `npm i @tabler/icons-react` | Very large set, consistent stroke |
| Hugeicons | `npm i @hugeicons/react @hugeicons/core-free-icons` | Supported by the shadcn CLI as an icon library option |
| react-icons | `npm i react-icons` | Aggregator. Use `react-icons/si` (Simple Icons) for brand logos. Import from the specific sub-path |
| Iconify | `npm i @iconify/react` | Huge catalog but fetches at runtime: avoid for above-the-fold UI |

Keep one primary icon set. Size with `size-4` / `size-5`, color with `text-*`, and set `aria-hidden` on decorative icons.

## Fonts (`next/font`)

```tsx
import { Inter, JetBrains_Mono } from "next/font/google";
const sans = Inter({ variable: "--font-inter", subsets: ["latin"], display: "swap" });
const mono = JetBrains_Mono({ variable: "--font-jetbrains", subsets: ["latin"] });
```

Then map in `globals.css`: `@theme inline { --font-sans: var(--font-inter); --font-mono: var(--font-jetbrains); }`.

- Self-hosted files: `next/font/local` with `src: "./fonts/Name.woff2"` (and `variable`). Use for brand or pixel fonts.
- Non-Google catalogs: Fontsource (`npm i @fontsource-variable/<name>`) imported in the layout.
- Limit to two families (sans + mono, or sans + one display face) and prefer variable fonts so weights don't multiply requests.
- Request only the subsets you need (`latin`, plus others if the content requires).

Pairings that work well:
- Geist + Geist Mono: neutral product UI (the scaffold default)
- Inter + JetBrains Mono: dense dashboards and dev tools
- Space Grotesk + Inter: techy, slightly quirky headings
- Plus Jakarta Sans or DM Sans: friendly SaaS/marketing
- Instrument Serif or Fraunces (headings) + Inter (body): editorial feel
- Press Start 2P, Silkscreen, or VT323 (display only) + a readable mono: retro/pixel themes. Keep body text in a legible font.
- Arabic or other scripts: add the matching subset or family (e.g. Noto Sans Arabic, IBM Plex Sans Arabic, Cairo) and set `dir="rtl"` where needed.

## Styling helpers

| Package | Purpose |
|---|---|
| `tailwindcss`, `@tailwindcss/postcss` | Tailwind v4 core and PostCSS plugin |
| `tw-animate-css` | Animation utilities used by shadcn (replaces `tailwindcss-animate` on v4) |
| `class-variance-authority` | Component variants (`cva`) |
| `clsx`, `tailwind-merge` | `cn()` helper |
| `prettier-plugin-tailwindcss` | Auto-sorts class names |
| `@tailwindcss/typography` | `prose` styles for rendered markdown (load with `@plugin "@tailwindcss/typography";` in v4) |

## Motion

- `motion` (`import { motion } from "motion/react"`): component animations, layout transitions, gestures, `AnimatePresence`. Successor to framer-motion.
- `gsap` + `@gsap/react` (`useGSAP`): timelines, ScrollTrigger scenes, fine control. Clean up with the hook's scope.
- `lenis`: smooth scrolling (use sparingly; it changes native scroll behavior).
- 3D: `three` + `@react-three/fiber` + `@react-three/drei`. Load canvases with `next/dynamic` and `ssr: false`.
- Always respect `prefers-reduced-motion`.

## Forms & validation

- `zod`: schemas shared between client form, Server Action, and API types (`z.infer`). `valibot` is a smaller alternative.
- `react-hook-form` + `@hookform/resolvers`: default form state manager with shadcn's form components.
- `@tanstack/react-form`: alternative with strong type inference; see `tanstack-recipes.md`.
- `input-otp`, `react-day-picker`: used by shadcn's OTP and calendar components.

## State & URL

- `nuqs`: typed URL query-string state (filters, tabs, pagination). Needs its adapter/provider per its docs for your Next version.
- `zustand`: small client stores (UI state, wizard steps). Don't mirror server data into it, that's TanStack Query's job.
- `jotai`: atomic alternative if the project already uses it.
- `next-themes`: dark/light/system.

## Data & networking

- `@tanstack/react-query` (+ `-devtools`, `@tanstack/eslint-plugin-query` to catch key and option mistakes).
- A thin `fetch` wrapper in `lib/api.ts` (base URL, JSON, error normalization), or `ky` / `ofetch`. Skip axios in new code unless already present.
- `@t3-oss/env-nextjs` + zod: validated environment variables at build/start.
- `msw`: mock API in tests and local dev.
- Backend pieces (only when asked): Better Auth or Auth.js, Drizzle or Prisma, tRPC, Supabase.

## UI extras

- `sonner` toasts, `cmdk` command menu, `vaul` drawers, `embla-carousel-react`, `react-resizable-panels`: all wired through shadcn components.
- `recharts`: shadcn `chart` wrapper handles theming.
- `@dnd-kit/core` + `@dnd-kit/sortable`: drag and drop, sortable lists, kanban.
- `date-fns`: dates. Use `Intl.DateTimeFormat` / `Intl.RelativeTimeFormat` first for plain formatting.
- `react-dropzone` or native input for uploads; `react-easy-crop` for avatars.

## Content

- `react-markdown` + `remark-gfm` (+ `rehype-sanitize` for untrusted input): render markdown safely.
- `shiki`: accurate code highlighting (run on the server to avoid shipping the grammar to the client).
- `next-mdx-remote` or `@next/mdx`: MDX content.

## Testing & tooling

- Unit/component: `vitest` + `@testing-library/react` + `@testing-library/user-event`.
- End to end: `playwright`.
- Lint/format: Next's ESLint config, or Biome as a single fast tool; `prettier` with the Tailwind plugin if using Prettier.
- Type checking: `tsc --noEmit` in CI; keep `strict: true`.
- Accessibility: `eslint-plugin-jsx-a11y` (bundled in Next's config), and axe via Playwright for page-level checks.
- Bundle: `@next/bundle-analyzer` when a page feels heavy.
