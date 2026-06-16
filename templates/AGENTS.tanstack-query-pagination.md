---
name: tanstack_query_pagination_agent
description: Project agent rules for TanStack Query pagination and infinite scrolling
---

Use this template when implementing paginated lists, load-more lists, or infinite scrolling with TanStack Query v5 in a React TypeScript project.

## Core Rules

- Confirm the project uses `@tanstack/react-query` v5 before using this pattern.
- Keep network fetchers separate from query hooks.
- Keep response types near the API contract that owns them.
- Include every variable that changes fetched data in the `queryKey`.
- Use `enabled` when a required id, search param, auth state, or filter is missing.
- Do not store server-list data in local state when TanStack Query can own it.
- Use `placeholderData: keepPreviousData` for page-number pagination.
- Use `useInfiniteQuery` for cursor, "load more", chat history, or scroll-triggered fetching.
- Guard `fetchNextPage` with `hasNextPage && !isFetching && !isFetchingNextPage`.
- Remove debug logging before committing.

Official docs to verify unfamiliar behavior:

- https://tanstack.com/query/latest/docs/framework/react/guides/paginated-queries
- https://tanstack.com/query/latest/docs/framework/react/guides/infinite-queries

## File Organization

Prefer this shape unless the project already has a stronger local convention:

```text
src/
|-- services/
|   |-- api.ts       # axios/fetch functions
|   `-- queries.ts   # TanStack Query hooks
`-- types/
    `-- item.ts      # response and item types
```

For larger codebases, colocate inside the feature:

```text
src/features/events/
|-- api/
|   |-- events.api.ts
|   `-- events.queries.ts
`-- types.ts
```

## Query Key Pattern

Use stable keys. Include page, cursor dependencies, filters, ids, sort, and search terms.

```ts
export const eventKeys = {
    all: ['events'] as const,
    lists: () => [...eventKeys.all, 'list'] as const,
    list: (params: EventListParams) => [...eventKeys.lists(), params] as const,
    detail: (eventId: string) => [...eventKeys.all, 'detail', eventId] as const,
}
```

If the existing project uses simple array keys such as `['events', page]`, keep that style consistent. Do not mix several key styles casually.

## Page-Number Pagination

Use this when the API accepts `page` and returns pagination metadata such as `totalPages`.

### Types

```ts
export interface PaginationMeta {
    page: number
    limit: number
    totalPages: number
    totalItems: number
}

export interface PaginatedResponse<TItem> {
    items: TItem[]
    pagination: PaginationMeta
}
```

Rename `items` to the real resource name when that is the local API contract, such as `events` or `orders`.

### Fetcher

```ts
interface GetItemsParams {
    page: number
    limit?: number
    search?: string
}

export const getItems = async ({
    page,
    limit = 12,
    search,
}: GetItemsParams): Promise<PaginatedResponse<Item>> => {
    const response = await axiosInstance.get('/api/items', {
        params: {
            page,
            limit,
            search: search || undefined,
        },
    })

    return response.data
}
```

Prefer `params` over manually building query strings. It avoids missing URL encoding when filters are added later.

### Query Hook

```ts
import { keepPreviousData, useQuery } from '@tanstack/react-query'

export const useItemsPagination = (params: GetItemsParams) => {
    return useQuery({
        queryKey: ['items', params],
        queryFn: () => getItems(params),
        placeholderData: keepPreviousData,
    })
}
```

If using simple page-only keys:

```ts
export const useItemsPagination = (page = 1) => {
    return useQuery({
        queryKey: ['items', page],
        queryFn: () => getItems({ page }),
        placeholderData: keepPreviousData,
    })
}
```

### Component Usage

```tsx
import { useState } from 'react'
import { useItemsPagination } from '@/services/queries'

const ItemList = () => {
    const [page, setPage] = useState(1)
    const {
        data,
        isPending,
        isError,
        isFetching,
        isPlaceholderData,
    } = useItemsPagination({ page, limit: 12 })

    if (isPending) return <p>Loading...</p>
    if (isError) return <p>Something went wrong.</p>

    const items = data?.items ?? []
    const totalPages = data?.pagination.totalPages ?? 1

    return (
        <div>
            {isFetching && !isPlaceholderData && <p>Refreshing...</p>}

            <ul>
                {items.map(item => (
                    <li key={item.id}>{item.name}</li>
                ))}
            </ul>

            <button
                type="button"
                disabled={page === 1}
                onClick={() => setPage(current => Math.max(current - 1, 1))}
            >
                Previous
            </button>

            <button
                type="button"
                disabled={isPlaceholderData || page >= totalPages}
                onClick={() => setPage(current => current + 1)}
            >
                Next
            </button>
        </div>
    )
}
```

When filters change, reset the page to `1` before fetching the new filtered list.

## Cursor Infinite Query

Use this when the API returns a cursor such as `nextCursor`.

### Types

```ts
export interface InfinitePage<TItem> {
    items: TItem[]
    nextCursor?: number | string | null
}
```

Use the real response field names from the API. For example, a chat API might return `{ messages, nextCursor }`.

### Fetcher

```ts
interface GetFeedPageParams {
    cursor?: number | string
    roomId: string
}

export const getFeedPage = async ({
    cursor = 0,
    roomId,
}: GetFeedPageParams): Promise<InfinitePage<Message>> => {
    const response = await axiosInstance.get(`/api/messages/${roomId}`, {
        params: {
            cursor,
        },
    })

    return response.data
}
```

### Query Hook

```ts
import { useInfiniteQuery } from '@tanstack/react-query'

export const useInfiniteMessages = (roomId: string | undefined) => {
    return useInfiniteQuery({
        queryKey: ['messages', roomId],
        queryFn: ({ pageParam }) =>
            getFeedPage({
                roomId: roomId as string,
                cursor: pageParam,
            }),
        initialPageParam: 0,
        getNextPageParam: lastPage => lastPage.nextCursor ?? undefined,
        enabled: !!roomId,
    })
}
```

Do not call the fetcher with an invalid id. Use `enabled` for missing ids.

### Load More Component

```tsx
const Messages = ({ roomId }: { roomId?: string }) => {
    const {
        data,
        fetchNextPage,
        hasNextPage,
        isFetching,
        isFetchingNextPage,
        status,
    } = useInfiniteMessages(roomId)

    if (status === 'pending') return <p>Loading...</p>
    if (status === 'error') return <p>Something went wrong.</p>

    const messages = data.pages.flatMap(page => page.items)

    return (
        <div>
            <button
                type="button"
                disabled={!hasNextPage || isFetching}
                onClick={() => {
                    if (hasNextPage && !isFetching) {
                        fetchNextPage()
                    }
                }}
            >
                {isFetchingNextPage ? 'Loading more...' : 'Load more'}
            </button>

            {messages.map(message => (
                <article key={message.id}>{message.body}</article>
            ))}
        </div>
    )
}
```

TanStack Query allows only one ongoing fetch for an infinite query cache entry by default. Always guard scroll-triggered calls.

## Infinite Scroll With Intersection Observer

Use a sentinel element at the end or top of the list. Keep the hook small and pass in the query state.

```tsx
import { useEffect } from 'react'
import { useInView } from 'react-intersection-observer'

interface InfiniteScrollSentinelProps {
    hasNextPage: boolean
    isFetching: boolean
    fetchNextPage: () => unknown
}

export const InfiniteScrollSentinel = ({
    hasNextPage,
    isFetching,
    fetchNextPage,
}: InfiniteScrollSentinelProps) => {
    const { ref, inView } = useInView({
        threshold: 0.2,
    })

    useEffect(() => {
        if (inView && hasNextPage && !isFetching) {
            fetchNextPage()
        }
    }, [fetchNextPage, hasNextPage, inView, isFetching])

    return <div ref={ref} aria-hidden="true" />
}
```

For chat history loading from the top, place the sentinel near the top and preserve scroll position after older messages load. For normal feeds, place it near the bottom.

## Mutations And Invalidation

Invalidate the list keys that can be affected by the mutation.

```ts
const queryClient = useQueryClient()

const createItem = useMutation({
    mutationFn: createItemApi,
    onSuccess: () => {
        queryClient.invalidateQueries({ queryKey: ['items'] })
    },
})
```

For detail mutations, invalidate both detail and list keys when the list shows changed fields.

```ts
queryClient.invalidateQueries({ queryKey: ['items'] })
queryClient.invalidateQueries({ queryKey: ['items', 'detail', itemId] })
```

## Checklist

- Fetchers return typed response data and do not contain React hooks.
- Query hooks wrap fetchers and own `queryKey`, `queryFn`, `enabled`, and pagination options.
- Page-number queries include page and filters in the `queryKey`.
- Page-number queries use `placeholderData: keepPreviousData`.
- UI uses `isPlaceholderData` to disable unsafe next-page navigation.
- Infinite queries include `initialPageParam`.
- Infinite queries return `undefined` or `null` from `getNextPageParam` when no page remains.
- Infinite scroll guards `fetchNextPage` with `hasNextPage && !isFetching`.
- Lists flatten `data.pages` safely with `flatMap`.
- Missing ids use `enabled`; fetchers should not build invalid URLs.
- Mutations invalidate all impacted list and detail keys.
- Loading, empty, error, fetching-more, and end-of-list states are handled.
