# TanStack recipes

Contents: key factories + queryOptions · mutation with optimistic update · infinite query · server-side data table · TanStack Form · virtualized list

Targets Query v5, Table v8, Form v1, Virtual v3. Check installed versions first.

## Key factory + queryOptions

```ts
// src/features/posts/queries.ts
import { queryOptions } from "@tanstack/react-query";
import { api } from "@/lib/api";

export type Post = { id: string; title: string; liked: boolean };

export const postKeys = {
  all: ["posts"] as const,
  list: (filters: Record<string, unknown> = {}) => [...postKeys.all, "list", filters] as const,
  detail: (id: string) => [...postKeys.all, "detail", id] as const,
};

export const postsQueryOptions = (filters: Record<string, unknown> = {}) =>
  queryOptions({
    queryKey: postKeys.list(filters),
    queryFn: () => api.get<Post[]>("/posts", filters),
  });

export const postQueryOptions = (id: string) =>
  queryOptions({
    queryKey: postKeys.detail(id),
    queryFn: () => api.get<Post>(`/posts/${id}`),
  });
```

Use the same options object in `prefetchQuery`, `useQuery`, `useSuspenseQuery`, and `invalidateQueries`.

## Mutation with optimistic update

```ts
const qc = useQueryClient();

const toggleLike = useMutation({
  mutationFn: (id: string) => api.post(`/posts/${id}/like`),
  onMutate: async (id) => {
    await qc.cancelQueries({ queryKey: postKeys.detail(id) });
    const previous = qc.getQueryData<Post>(postKeys.detail(id));
    qc.setQueryData<Post>(postKeys.detail(id), (old) => old && { ...old, liked: !old.liked });
    return { previous };
  },
  onError: (_err, id, ctx) => {
    qc.setQueryData(postKeys.detail(id), ctx?.previous); // roll back
    toast.error("Couldn't update. Try again.");
  },
  onSettled: (_data, _err, id) => qc.invalidateQueries({ queryKey: postKeys.detail(id) }),
});
```

For simple cases skip the optimism: `onSuccess: () => qc.invalidateQueries({ queryKey: postKeys.all })`.

## Infinite query

```ts
const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfiniteQuery({
  queryKey: postKeys.list({ infinite: true }),
  queryFn: ({ pageParam }) => api.get<{ items: Post[]; nextCursor: string | null }>("/posts", { cursor: pageParam }),
  initialPageParam: null as string | null,
  getNextPageParam: (last) => last.nextCursor ?? undefined,
});
const items = data?.pages.flatMap((p) => p.items) ?? [];
```

Trigger `fetchNextPage` from an IntersectionObserver sentinel (or `react-intersection-observer`) rather than a scroll listener.

## Server-side data table (shadcn Data Table pattern)

```tsx
"use client";
import { keepPreviousData, useQuery } from "@tanstack/react-query";
import {
  ColumnDef, PaginationState, SortingState,
  flexRender, getCoreRowModel, useReactTable,
} from "@tanstack/react-table";
import { useState } from "react";

const columns: ColumnDef<Post>[] = [ /* defined OUTSIDE the component */ ];

export function PostsTable() {
  const [pagination, setPagination] = useState<PaginationState>({ pageIndex: 0, pageSize: 20 });
  const [sorting, setSorting] = useState<SortingState>([]);

  const { data } = useQuery({
    queryKey: ["posts", "table", pagination, sorting],
    queryFn: () => api.get<{ rows: Post[]; total: number }>("/posts", { ...pagination, sorting }),
    placeholderData: keepPreviousData, // no flicker between pages
  });

  const table = useReactTable({
    data: data?.rows ?? [],
    columns,
    rowCount: data?.total,
    state: { pagination, sorting },
    onPaginationChange: setPagination,
    onSortingChange: setSorting,
    manualPagination: true,
    manualSorting: true,
    getCoreRowModel: getCoreRowModel(),
  });

  // render with shadcn <Table>, flexRender(header/cell), and prev/next buttons
}
```

Swap the two `useState`s for `nuqs` (`useQueryState`) when the table state should survive reloads and be shareable by link. Without a backend doing the work, drop the `manual*` flags and add `getPaginationRowModel`, `getSortedRowModel`, `getFilteredRowModel`.

## TanStack Form with zod

```tsx
import { useForm } from "@tanstack/react-form";
import { z } from "zod";

const schema = z.object({ email: z.email("Enter a valid email") });

export function SubscribeForm() {
  const form = useForm({
    defaultValues: { email: "" },
    validators: { onChange: schema },
    onSubmit: async ({ value }) => { await subscribe(value); },
  });

  return (
    <form onSubmit={(e) => { e.preventDefault(); form.handleSubmit(); }}>
      <form.Field name="email">
        {(field) => (
          <div>
            <Input
              value={field.state.value}
              onBlur={field.handleBlur}
              onChange={(e) => field.handleChange(e.target.value)}
              aria-invalid={field.state.meta.errors.length > 0}
            />
            {field.state.meta.errors.map((err, i) => (
              <p key={i} className="text-sm text-destructive">{err?.message}</p>
            ))}
          </div>
        )}
      </form.Field>
      <form.Subscribe selector={(s) => s.isSubmitting}>
        {(isSubmitting) => <Button disabled={isSubmitting}>Subscribe</Button>}
      </form.Subscribe>
    </form>
  );
}
```

(`z.email()` is zod v4; use `z.string().email()` on v3.) If the project already uses `react-hook-form` with shadcn's form components, keep that instead of mixing the two.

## Virtualized list

```tsx
"use client";
import { useVirtualizer } from "@tanstack/react-virtual";
import { useRef } from "react";

export function VirtualList({ rows }: { rows: Post[] }) {
  const parentRef = useRef<HTMLDivElement>(null);
  const virtualizer = useVirtualizer({
    count: rows.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 48,
    overscan: 8,
  });

  return (
    <div ref={parentRef} className="h-[600px] overflow-auto">
      <div style={{ height: virtualizer.getTotalSize(), position: "relative" }}>
        {virtualizer.getVirtualItems().map((v) => (
          <div
            key={v.key}
            style={{ position: "absolute", top: 0, left: 0, width: "100%", height: v.size, transform: `translateY(${v.start}px)` }}
          >
            {rows[v.index].title}
          </div>
        ))}
      </div>
    </div>
  );
}
```

The scroll container needs a fixed height, otherwise nothing virtualizes. For variable heights, pass `ref={virtualizer.measureElement}` and `data-index` on items.
