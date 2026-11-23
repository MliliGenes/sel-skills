# Next.js patterns

Contents: root layout · providers · query client · prefetch + hydrate · Server Action with zod · segment conventions

Verify against the installed Next.js major version; these target App Router with React 19.

## Root layout (fonts + theme)

```tsx
// src/app/layout.tsx
import type { Metadata } from "next";
import { Geist, Geist_Mono } from "next/font/google";
import "./globals.css";
import { Providers } from "./providers";

const geistSans = Geist({ variable: "--font-geist-sans", subsets: ["latin"] });
const geistMono = Geist_Mono({ variable: "--font-geist-mono", subsets: ["latin"] });

export const metadata: Metadata = {
  title: { default: "App", template: "%s | App" },
  description: "...",
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body className={`${geistSans.variable} ${geistMono.variable} font-sans antialiased`}>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

`suppressHydrationWarning` is required on `<html>` with `next-themes`, which sets the class before hydration. Map the font variables to Tailwind in `globals.css` (see `shadcn-tailwind.md`).

## Providers

```tsx
// src/app/providers.tsx
"use client";

import { QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";
import { ThemeProvider } from "next-themes";
import { Toaster } from "@/components/ui/sonner";
import { getQueryClient } from "@/lib/query-client";

export function Providers({ children }: { children: React.ReactNode }) {
  const queryClient = getQueryClient();
  return (
    <ThemeProvider attribute="class" defaultTheme="system" enableSystem disableTransitionOnChange>
      <QueryClientProvider client={queryClient}>
        {children}
        <Toaster richColors />
        <ReactQueryDevtools initialIsOpen={false} />
      </QueryClientProvider>
    </ThemeProvider>
  );
}
```

Don't create the `QueryClient` at module scope on the server: it would be shared across requests and leak data between users.

## Query client

```ts
// src/lib/query-client.ts
import { QueryClient, defaultShouldDehydrateQuery, isServer } from "@tanstack/react-query";

function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: { staleTime: 60 * 1000 },
      dehydrate: {
        // also send in-flight queries so streaming prefetch works
        shouldDehydrateQuery: (query) =>
          defaultShouldDehydrateQuery(query) || query.state.status === "pending",
      },
    },
  });
}

let browserQueryClient: QueryClient | undefined;

export function getQueryClient() {
  if (isServer) return makeQueryClient(); // new per call on the server
  return (browserQueryClient ??= makeQueryClient()); // singleton in the browser
}
```

## Prefetch on the server, hydrate on the client

```tsx
// src/app/posts/page.tsx  (Server Component)
import { dehydrate, HydrationBoundary } from "@tanstack/react-query";
import { getQueryClient } from "@/lib/query-client";
import { postsQueryOptions } from "@/features/posts/queries";
import { PostList } from "@/features/posts/components/post-list";

export default async function PostsPage() {
  const queryClient = getQueryClient();
  await queryClient.prefetchQuery(postsQueryOptions());
  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <PostList />
    </HydrationBoundary>
  );
}
```

```tsx
// src/features/posts/components/post-list.tsx
"use client";
import { useQuery } from "@tanstack/react-query";
import { postsQueryOptions } from "../queries";

export function PostList() {
  const { data, isPending, error } = useQuery(postsQueryOptions());
  // render skeleton / error / list
}
```

Because the server and the client call the same `postsQueryOptions()`, the keys match and the client renders instantly with data, then keeps it fresh.

## Server Action with zod validation

```ts
// src/features/posts/actions.ts
"use server";

import { z } from "zod";
import { revalidatePath } from "next/cache";

const createPostSchema = z.object({
  title: z.string().min(1, "Title is required").max(120),
  body: z.string().max(5000).optional(),
});

export type CreatePostState =
  | { ok: true }
  | { ok: false; errors: Record<string, string[] | undefined> };

export async function createPost(_prev: CreatePostState | null, formData: FormData): Promise<CreatePostState> {
  const parsed = createPostSchema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) {
    // zod v4: z.flattenError(parsed.error); zod v3: parsed.error.flatten()
    return { ok: false, errors: z.flattenError(parsed.error).fieldErrors };
  }
  // check auth/permissions here, then write to the DB
  revalidatePath("/posts");
  return { ok: true };
}
```

Use it from a client form with `useActionState(createPost, null)`; the `pending` flag it returns drives the disabled submit button. Authorize inside the action: anyone can call a Server Action endpoint directly.

## Route segment conventions

- `loading.tsx` - streamed skeleton for the segment.
- `error.tsx` - must be a Client Component; receives `error` and `reset`.
- `not-found.tsx` + `notFound()` - for missing records.
- `layout.tsx` - persistent shell; don't fetch per-page data here if it blocks children.
- Route groups `(marketing)`, `(app)` - separate layouts without changing URLs.
- `generateMetadata` - per-page titles from fetched data.
- Next 16 renamed `middleware.ts` to `proxy.ts`; check the installed version before creating either.
