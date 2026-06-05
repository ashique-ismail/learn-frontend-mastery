# Concurrent Rendering — What Changed in React 18

## The Idea

**In plain English:** Concurrent rendering is React's ability to work on updating your screen in small, pausable chunks instead of doing everything all at once — so if something more important comes up (like you typing), React can stop what it was doing, handle that first, then come back to the bigger task.

**Real-world analogy:** Imagine a chef preparing a big fancy meal (a slow, complex dish) when a customer suddenly asks for a glass of water. Instead of ignoring the request until the full meal is done, the chef pauses, pours the water immediately, then returns to cooking.

- The chef = React
- The complex meal being cooked = a slow, expensive render (like filtering a huge list)
- The customer's urgent water request = a high-priority update (like a keypress or button click)

---

## What Concurrent Rendering Is

Before React 18, rendering was **synchronous and uninterruptible** — once React started rendering, it couldn't stop until done. Long renders blocked user interactions.

React 18's Concurrent Mode makes rendering **interruptible** — React can pause a render in progress, handle a higher-priority update (like a keypress), then resume or discard the paused work.

```
Legacy rendering (blocking):
  Start render → Cannot pause → Cannot interrupt → DOM update

Concurrent rendering:
  Start render → Pause for urgent update → Resume or restart → DOM update
```

---

## Key New APIs in React 18

### `startTransition` — Mark Non-Urgent Updates

```tsx
import { startTransition, useTransition } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  function handleSearch(e: React.ChangeEvent<HTMLInputElement>) {
    setQuery(e.target.value);  // urgent: update input immediately

    startTransition(() => {
      // non-urgent: can be interrupted/deferred
      setResults(computeExpensiveSearch(e.target.value));
    });
  }

  return (
    <>
      <input value={query} onChange={handleSearch} />
      {isPending && <Spinner />}  {/* show while transition is pending */}
      <ResultsList results={results} />
    </>
  );
}
```

Without `startTransition`: typing feels laggy because React re-renders the full results list on every keystroke.

With `startTransition`: keystrokes are instantly reflected; results update when React gets to them (interruptible).

### `useDeferredValue` — Defer a Value

```tsx
const deferredQuery = useDeferredValue(query);
// deferredQuery lags behind query; always shows "old" value until React is free
// Useful when you can't wrap the state setter in startTransition

const filteredItems = useMemo(
  () => items.filter(item => item.name.includes(deferredQuery)),
  [items, deferredQuery]
);
```

---

## Automatic Batching

React 17: state updates inside setTimeout, Promises, and native event handlers each caused a separate re-render.

React 18: **all** state updates are batched automatically regardless of context:

```tsx
// React 17: 2 separate renders
setTimeout(() => {
  setCount(c => c + 1); // render 1
  setFlag(f => !f);      // render 2
}, 1000);

// React 18: 1 render (automatic batching)
setTimeout(() => {
  setCount(c => c + 1); // \
  setFlag(f => !f);      //  → batched into 1 render
}, 1000);

// Opt out of batching when needed
import { flushSync } from 'react-dom';
flushSync(() => setCount(c => c + 1)); // immediate re-render
flushSync(() => setFlag(f => !f));      // separate re-render
```

---

## Concurrent Suspense

In React 18, Suspense works properly with concurrent rendering — showing stale content while new data loads instead of showing a fallback spinner:

```tsx
function UserPage({ userId }) {
  const [isPending, startTransition] = useTransition();

  function navigateToUser(id) {
    startTransition(() => setUserId(id)); // transition: keep old UI until new is ready
  }

  return (
    <>
      {isPending && <TopLoadingBar />}  {/* subtle loading indicator */}
      <Suspense fallback={<Skeleton />}>
        <UserProfile userId={userId} />   {/* shows while loading */}
      </Suspense>
    </>
  );
}
```

Without startTransition: `<Skeleton />` flashes every time userId changes.
With startTransition: old user profile stays visible until new one is ready.

---

## Streaming SSR

React 18 enables HTML streaming — the server sends HTML in chunks as components resolve:

```tsx
// Next.js App Router / React 18 streaming
export default async function Page() {
  return (
    <div>
      <Header />  {/* sent immediately */}
      <Suspense fallback={<Skeleton />}>
        <SlowDataComponent />  {/* streamed when ready */}
      </Suspense>
    </div>
  );
}
```

The browser can render the `<Header />` immediately and display the skeleton while waiting for `<SlowDataComponent />` data.

---

## Summary of React 18 Changes

| Feature | Before React 18 | React 18 |
|---|---|---|
| Rendering | Blocking (uninterruptible) | Concurrent (interruptible) |
| Batching | Event handlers only | Automatic everywhere |
| Transitions | N/A | `startTransition`/`useTransition` |
| Deferred values | N/A | `useDeferredValue` |
| Suspense + navigation | Spinner flash | Stale content during transition |
| SSR | Waterfall HTML | Streaming with Suspense |

---

## Common Interview Questions

**Q: Does Concurrent Mode break existing React code?**
Potentially — if components have side effects in render (which they shouldn't), they might run multiple times. React 18 Strict Mode double-invokes effects in development to surface these issues.

**Q: What's the difference between `startTransition` and `setTimeout`?**
`setTimeout` delays work but still runs it synchronously when executed. `startTransition` marks work as interruptible — React can pause and resume it. If a higher-priority update arrives, React can discard the transition work and restart.

**Q: When should you use `useDeferredValue` vs `useTransition`?**
`useTransition`: when you control the state setter (wrap the setter call). `useDeferredValue`: when you receive a value as a prop or can't control when the state updates (e.g., from an external library).
