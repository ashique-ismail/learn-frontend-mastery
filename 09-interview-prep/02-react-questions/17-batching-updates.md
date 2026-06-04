# How Does React Batch Updates?

## What Batching Means

Batching groups multiple state updates into a **single re-render**, avoiding unnecessary intermediate renders.

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  function handleClick() {
    setCount(c => c + 1);
    setFlag(f => !f);
    // Without batching: 2 renders
    // With batching: 1 render (both updates applied together)
  }
}
```

---

## React 17 and Earlier — Partial Batching

Only batched updates inside **React event handlers** (synthetic events):

```tsx
// ✅ Batched in React 17 (inside React event handler)
<button onClick={() => {
  setCount(c => c + 1); // \
  setFlag(f => !f);      //  → 1 render
}}>

// ❌ NOT batched in React 17 (inside setTimeout)
setTimeout(() => {
  setCount(c => c + 1); // render 1
  setFlag(f => !f);      // render 2 (two separate renders!)
}, 1000);

// ❌ NOT batched in React 17 (inside a Promise)
fetchData().then(() => {
  setCount(c => c + 1); // render 1
  setFlag(f => !f);      // render 2
});
```

---

## React 18 — Automatic Batching Everywhere

React 18 batches **all** state updates regardless of context:

```tsx
// ✅ All batched in React 18

// setTimeout
setTimeout(() => {
  setCount(c => c + 1); // \
  setFlag(f => !f);      //  → 1 render (batched!)
}, 1000);

// Promises
fetchData().then(() => {
  setCount(c => c + 1); // \
  setFlag(f => !f);      //  → 1 render (batched!)
});

// Native event listeners
document.addEventListener('click', () => {
  setCount(c => c + 1); // \
  setFlag(f => !f);      //  → 1 render (batched!)
});
```

---

## How Batching Works Internally

React uses a **scheduler** (the Scheduler package) to queue state updates. When a state setter is called, React doesn't immediately re-render — it schedules the update. At the end of the current execution context (event handler, microtask, etc.), React processes all queued updates together.

```
setCount called → queued
setFlag called  → queued
[end of event handler]
React processes queue → compute new state → single render
```

---

## `flushSync` — Opt Out of Batching

When you need an immediate, synchronous render before the next update:

```tsx
import { flushSync } from 'react-dom';

function handleClick() {
  flushSync(() => {
    setCount(c => c + 1); // renders immediately
  });
  // DOM has been updated here — can read layout

  flushSync(() => {
    setFlag(f => !f);      // another immediate render
  });
}
```

Use cases for `flushSync`:
- Reading DOM measurements between state updates
- Integrating with third-party libraries that expect synchronous updates
- Animations that must read layout between renders

---

## Batching with Concurrent Features

React 18 also batches updates triggered by `startTransition`:

```tsx
const [isPending, startTransition] = useTransition();

startTransition(() => {
  setQuery(newQuery);   // \
  setPage(1);            //  → batched into 1 low-priority render
});
```

Transition updates are batched together and treated as lower priority than urgent updates (from user input).

---

## Practical Implications

```tsx
// ✅ These are equivalent in React 18
function handleAdd() {
  setItems(prev => [...prev, newItem]);
  setCount(c => c + 1);
  setLoading(false);
  // All 3 batched → 1 render
}

// ❌ Avoid this pattern — defeats batching
function handleAdd() {
  setItems(prev => [...prev, newItem]);
  setTimeout(() => {
    setCount(c => c + 1); // In React 17: separate render; In React 18: still batched
  }, 0);
}
```

---

## Common Interview Questions

**Q: What's the problem with multiple renders from unbatched updates?**
Each intermediate state triggers a render with potentially inconsistent state. If `setCount` renders first, the component sees `count+1` but old `flag` — a briefly inconsistent state that could cause bugs or flicker.

**Q: Does batching work inside `useEffect`?**
Yes in React 18 — state updates inside `useEffect` are batched together if they happen synchronously. Async updates within `useEffect` (after an `await`) are also batched in React 18.

**Q: How does React know when to flush the batch?**
It flushes at the end of the "work loop" — after the synchronous event handler or microtask completes. It uses the browser's microtask queue (via `queueMicrotask` or `MessageChannel`) to schedule the flush at the right time.
