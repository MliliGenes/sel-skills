---
name: next-frontend
description: Build, scaffold, style, and review modern React frontends with Next.js (App Router), shadcn/ui, Tailwind CSS v4, and the TanStack libraries (Query, Table, Form, Virtual), plus the supporting ecosystem - icon libraries, next/font, theming, motion, forms/validation, toasts, charts. Use this skill whenever the user is building or debugging a Next.js or React web UI, mentions shadcn, Tailwind, TanStack Query/Table/Form, data fetching, data tables, forms, dark mode, icons, fonts, or asks which frontend libraries to pick, even if they never name the whole stack.
---

# Next.js + shadcn/ui + TanStack frontend

This skill encodes sensible defaults for a modern React frontend so the output is consistent, idiomatic, and doesn't need rework. The stack moves fast, so the defaults below are a starting point, not gospel: always let the existing project win over this document.

## Before writing any code

1. **Detect the project.** Read `package.json`, `components.json`, `next.config.*`, and the folder layout (`app/` vs `pages/`, `src/` or not). Note the Next.js major version, Tailwind version (v3 has `tailwind.config.js`, v4 is CSS-first), installed shadcn components, and which icon library `components.json` names. Match what is there instead of imposing the defaults.
2. **Verify versions when it matters.** If the code depends on a specific API (Next caching, TanStack Query options, shadcn CLI flags), run `npm view <pkg> version` or check the docs rather than relying on memory. Wrong-version code is the most common failure with this stack.
3. **Pick one tool per job.** Don't add a second icon library, a second form library, or a second state manager. If the project already has one, use it.

## Default stack

| Job | Default | Reach for instead when... |
|---|---|---|
| Framework | Next.js App Router, TypeScript | Existing Pages Router project: stay on it |
| UI components | shadcn/ui (copied into the repo) | - |
| Styling | Tailwind v4, `tw-animate-css`, `cva`, `clsx` + `tailwind-merge` (`cn`) | Project on Tailwind v3: keep v3 conventions |
| Icons | `lucide-react` (shadcn default) | Need more variety: Phosphor, Tabler, Hugeicons. Brand logos: `react-icons/si` |
| Fonts | `next/font` with Geist + Geist Mono | Inter, JetBrains Mono, Space Grotesk, local files via `next/font/local` |
| Server state | TanStack Query | Pure server-rendered page with no client interactivity: just `await` in a Server Component |
| Tables | TanStack Table (shadcn Data Table) | Simple static table: plain shadcn `Table` |
| Forms | `react-hook-form` + `zod` (shadcn form components) | TanStack Form if the project already uses TanStack heavily |
| Long lists | TanStack Virtual | Under ~200 rows: don't virtualize |
| URL state | `nuqs` | - |
| Client state | `zustand`, only when Query + URL + component state aren't enough | - |
| Theme | `next-themes` | - |
| Toasts | `sonner` (shadcn `sonner`) | - |
| Motion | `motion` | GSAP for timelines and scroll-driven scenes |
| Charts | `recharts` via shadcn `chart` | - |

TanStack Router and TanStack Start own routing themselves. Don't combine them with Next.js. If the user wants TanStack Router, the right base is Vite or TanStack Start, and the Next-specific advice below doesn't apply.

## Scaffolding a new project

```bash
npx create-next-app@latest my-app --ts --tailwind --eslint --app --src-dir --import-alias "@/*"
cd my-app
npx shadcn@latest init
npx shadcn@latest add button card input label dialog dropdown-menu sonner skeleton
npm i @tanstack/react-query @tanstack/react-query-devtools next-themes zod nuqs
```

Run `npx create-next-app@latest --help` if a flag errors; the CLI changes between majors. For forms add `react-hook-form @hookform/resolvers`; for tables `@tanstack/react-table`; for long lists `@tanstack/react-virtual`.

Suggested layout:

```
src/
  app/                    routes, layout.tsx, providers.tsx, globals.css
  components/ui/          shadcn output (edit freely)
  components/             app-level shared components
  features/<name>/        components/, queries.ts, schemas.ts, actions.ts
  lib/                    utils.ts (cn), query-client.ts, api.ts
  hooks/
```

Group by feature, not by file type, so a change to "posts" touches one folder.

## Rules that prevent the usual bugs

**Next.js**
- Server Components by default. Add `"use client"` only at the leaf that needs state, effects, or browser APIs, because every client boundary ships JS to the user.
- In recent Next versions `params` and `searchParams` are Promises: `const { id } = await params`.
- Load initial data in Server Components. Use TanStack Query for what happens after load: mutations, polling, infinite scroll, optimistic updates, shared client cache. When both are needed, prefetch on the server and hydrate (see `references/next-patterns.md`).
- Validate every Server Action input with zod on the server, even if the client validated too. The client is not a trust boundary.
- Use `next/image`, `next/link`, `next/font`, and the Metadata API. Add `loading.tsx` and `error.tsx` for route segments that fetch.
- Keep secrets out of anything with `NEXT_PUBLIC_`.

**Tailwind v4 + shadcn**
- Style with semantic tokens (`bg-background`, `text-muted-foreground`, `border-border`) so dark mode and re-theming work from CSS variables alone. Hardcoded hex or `bg-gray-900` in components breaks theming.
- Theme in `globals.css` via `:root`, `.dark`, and `@theme inline`. There is normally no `tailwind.config.js` in v4. Don't rewrite the generated `globals.css` wholesale; edit tokens in place.
- Add components with `npx shadcn@latest add <name>`, not by hand-copying, so dependencies and imports come out right. Once added, the code is yours: change `components/ui/*` to change the design system, and compose in `components/` for app-specific pieces.
- Merge classes with `cn()`; define variants with `cva`. Prefer `size-4` over `w-4 h-4` and `gap-*` over `space-*` for new code.
- Reach for shadcn blocks (`sidebar-*`, `login-*`, `dashboard-*`) before designing a layout from scratch.

**TanStack**
- Query v5 uses a single object argument everywhere. Share `queryOptions()` objects between server prefetch and client `useQuery` so keys can't drift. Use key factories.
- Set a non-zero default `staleTime` (e.g. 60s) or hydrated data refetches immediately on mount.
- `isPending` means "no data yet"; `isFetching` means "a request is in flight". `gcTime` replaced `cacheTime`.
- After a mutation, invalidate the affected keys. Use optimistic updates only where instant feedback matters, and always roll back in `onError`.
- Table: define `columns` outside the component (or memoize), and use `manualPagination` / `manualSorting` / `manualFiltering` when the server does the work. Keep table state in the URL with `nuqs` when users should be able to share or reload a view. If the React Compiler is enabled, check that `useReactTable` still re-renders correctly.

## Quality checklist (run mentally before finishing)

- Keyboard: every interactive element reachable and visibly focused; dialogs trap focus (shadcn/Radix does this, don't break it by replacing the primitives).
- Every icon-only button has an `aria-label`.
- Loading, empty, and error states exist for every data view (use `Skeleton`, not spinners, for layout-shaped content).
- Dark mode works: no hardcoded colors, `suppressHydrationWarning` on `<html>` when using `next-themes`.
- No layout shift from fonts or images (`next/font`, explicit image dimensions).
- Forms show field-level errors and disable the submit button while pending.
- Run `npx tsc --noEmit`, the linter, and `npm run build` when a shell and project are available. Report anything that couldn't be run instead of implying it passed.

## Design quality

This skill covers engineering structure. For distinctive visual direction (typography choices, layout, avoiding templated look), also consult the `frontend-design` skill when it is available, and use the tokens in `references/shadcn-tailwind.md` as the place to express that direction.

## Reference files

Read only what the task needs:

- `references/next-patterns.md` - root layout with fonts and theme, providers, query client, server prefetch + hydrate, Server Action with zod, route-segment conventions.
- `references/tanstack-recipes.md` - query key factories, `queryOptions`, mutations with optimistic updates, infinite query, server-side data table, TanStack Form, virtualized list.
- `references/shadcn-tailwind.md` - `components.json`, Tailwind v4 `globals.css` token skeleton, `cn` helper, theming and dark mode, which shadcn components to reach for.
- `references/ecosystem.md` - catalog of useful libraries (icons, fonts and pairings, motion, forms, state, dates, charts, markdown/code, testing, tooling) with install lines and when to use each.

## Delivering output

When working in a project with a shell, edit the real files. In a chat sandbox without the user's repo, write the files under the same paths the project would use and present them, with the exact install and `shadcn add` commands the user needs to run. Next.js apps can't be previewed in a chat artifact; say so rather than substituting a different framework silently.
