# Implement Infinite Scroll Component

## Core Concept

Use `IntersectionObserver` to watch a "sentinel" element at the bottom of the list. When it enters the viewport, load the next page.

---

## Custom Hook

```ts
function useInfiniteScroll(
  onLoadMore: () => void,
  { threshold = 0.1, rootMargin = '100px' } = {}
) {
  const sentinelRef = useRef<HTMLDivElement>(null);
  const observerRef = useRef<IntersectionObserver | null>(null);

  useEffect(() => {
    const sentinel = sentinelRef.current;
    if (!sentinel) return;

    observerRef.current = new IntersectionObserver(
      entries => {
        if (entries[0].isIntersecting) {
          onLoadMore();
        }
      },
      { threshold, rootMargin }
    );

    observerRef.current.observe(sentinel);
    return () => observerRef.current?.disconnect();
  }, [onLoadMore, threshold, rootMargin]);

  return sentinelRef;
}
```

---

## Component with TanStack Query

```tsx
function ProductList() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage, status } =
    useInfiniteQuery({
      queryKey: ['products'],
      queryFn: ({ pageParam = 0 }) => api.getProducts({ page: pageParam, limit: 20 }),
      getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
    });

  const loadMore = useCallback(() => {
    if (hasNextPage && !isFetchingNextPage) {
      fetchNextPage();
    }
  }, [hasNextPage, isFetchingNextPage, fetchNextPage]);

  const sentinelRef = useInfiniteScroll(loadMore);

  if (status === 'pending') return <Spinner />;
  if (status === 'error') return <ErrorMessage />;

  const products = data.pages.flatMap(page => page.items);

  return (
    <div role="feed" aria-busy={isFetchingNextPage}>
      {products.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}

      {/* Sentinel element — watched by IntersectionObserver */}
      <div ref={sentinelRef} aria-hidden="true" />

      {isFetchingNextPage && <Spinner />}
      {!hasNextPage && <p>All products loaded</p>}
    </div>
  );
}
```

---

## Manual Pagination (Without TanStack Query)

```tsx
function useInfiniteData<T>(fetchPage: (page: number) => Promise<{ items: T[]; hasMore: boolean }>) {
  const [items, setItems] = useState<T[]>([]);
  const [page, setPage] = useState(0);
  const [hasMore, setHasMore] = useState(true);
  const [loading, setLoading] = useState(false);

  const loadMore = useCallback(async () => {
    if (loading || !hasMore) return;
    setLoading(true);

    try {
      const result = await fetchPage(page);
      setItems(prev => [...prev, ...result.items]);
      setHasMore(result.hasMore);
      setPage(p => p + 1);
    } catch (error) {
      console.error(error);
    } finally {
      setLoading(false);
    }
  }, [loading, hasMore, page, fetchPage]);

  // Load first page on mount
  useEffect(() => { loadMore(); }, []); // eslint-disable-line

  return { items, hasMore, loading, loadMore };
}
```

---

## Accessibility

```tsx
// Announce new items to screen readers
function ProductList() {
  const { items, loading } = useInfiniteData(fetchProducts);

  return (
    <>
      {/* Live region announces when new items are loaded */}
      <div aria-live="polite" className="sr-only">
        {loading ? 'Loading more products...' : `${items.length} products loaded`}
      </div>

      <ul role="feed">
        {items.map(item => (
          <li key={item.id}><ProductCard product={item} /></li>
        ))}
      </ul>
    </>
  );
}
```

---

## Common Interview Questions

**Q: Why IntersectionObserver over scroll events?**
Scroll events fire hundreds of times per second and require `getBoundingClientRect()` (forces layout/reflow). IntersectionObserver is async, throttled by the browser, and much more performant.

**Q: How do you prevent duplicate loads?**
Guard with `if (loading || !hasMore) return` before triggering the fetch. The `isFetchingNextPage` flag from TanStack Query serves the same purpose.

**Q: What's `rootMargin` used for?**
Pre-fetches the next page before the user reaches the bottom. `rootMargin: '200px'` means the observer fires when the sentinel is 200px from entering the viewport, giving time for the request to complete.
