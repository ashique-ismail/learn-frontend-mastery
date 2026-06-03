# useEffect vs useLayoutEffect

## The Key Difference

Both run after render, but at different points in the browser's rendering pipeline:

```
React renders → DOM updated → useLayoutEffect runs → Browser paints → useEffect runs
                              ↑ synchronous           ↑ async (after paint)
```

- **`useLayoutEffect`** — fires synchronously after DOM mutations, before the browser paints. Blocks painting until it completes.
- **`useEffect`** — fires asynchronously after the browser has painted. Does not block painting.

---

## When to Use `useLayoutEffect`

Use it when you need to **read or mutate the DOM before the user sees it** — to prevent a visible flash or layout jump.

```tsx
// Measure DOM element and set tooltip position BEFORE paint
function Tooltip({ anchor, content }) {
  const tooltipRef = useRef<HTMLDivElement>(null);

  useLayoutEffect(() => {
    const anchorRect = anchor.current.getBoundingClientRect();
    const tooltip = tooltipRef.current;
    if (!tooltip) return;

    // Position tooltip based on anchor measurement
    // This runs before paint → no visible jump
    tooltip.style.top = `${anchorRect.bottom + 8}px`;
    tooltip.style.left = `${anchorRect.left}px`;
  }, [anchor]);

  return <div ref={tooltipRef} className="tooltip">{content}</div>;
}
```

Other use cases for `useLayoutEffect`:
- Synchronizing scroll position
- Measuring element dimensions before first paint
- Animations that must start from a measured position
- Setting initial focus without flash

---

## When to Use `useEffect` (Default)

Almost everything else — async operations, subscriptions, analytics, localStorage:

```tsx
function DataPage({ userId }) {
  const [data, setData] = useState(null);

  // ✅ useEffect for async data fetching (doesn't need DOM measurement)
  useEffect(() => {
    let cancelled = false;
    fetchData(userId).then(d => { if (!cancelled) setData(d); });
    return () => { cancelled = true; };
  }, [userId]);

  // ✅ useEffect for subscriptions
  useEffect(() => {
    const sub = eventBus.subscribe('update', handler);
    return () => sub.unsubscribe();
  }, []);
}
```

---

## The Visual Difference

```tsx
function FlashyComponent() {
  const ref = useRef(null);

  // useEffect: user SEES the element at top, then it jumps to 200px
  useEffect(() => {
    ref.current.style.transform = 'translateX(200px)'; // visible jump!
  }, []);

  // useLayoutEffect: element appears at 200px directly, no jump
  useLayoutEffect(() => {
    ref.current.style.transform = 'translateX(200px)'; // no flash
  }, []);

  return <div ref={ref}>Box</div>;
}
```

---

## SSR Consideration

`useLayoutEffect` **cannot run on the server** (no DOM). Using it in a server-rendered component generates a warning:

```
Warning: useLayoutEffect does nothing on the server
```

Solutions:
```tsx
// Option 1: Use useEffect — only applies when the DOM difference doesn't matter SSR-side
useEffect(() => { /* measurement */ }, []);

// Option 2: Gate it
const useIsomorphicLayoutEffect =
  typeof window !== 'undefined' ? useLayoutEffect : useEffect;

useIsomorphicLayoutEffect(() => {
  // runs as useLayoutEffect client-side, useEffect server-side
}, []);
```

---

## Execution Order

```tsx
function Parent() {
  useLayoutEffect(() => { console.log('Parent useLayoutEffect'); });
  useEffect(() => { console.log('Parent useEffect'); });
  return <Child />;
}

function Child() {
  useLayoutEffect(() => { console.log('Child useLayoutEffect'); });
  useEffect(() => { console.log('Child useEffect'); });
  return <div />;
}

// Output:
// Child useLayoutEffect   ← children run first (bottom-up)
// Parent useLayoutEffect
// Child useEffect         ← then async effects, also bottom-up
// Parent useEffect
```

---

## Common Interview Questions

**Q: If `useLayoutEffect` is more powerful, why not always use it?**
It blocks the browser from painting until it completes — slowing down the perceived render. For most side effects (data fetching, subscriptions), the DOM doesn't need to be read/mutated before paint, so `useEffect` is correct and more performant.

**Q: What happens if `useLayoutEffect` takes a long time?**
The page appears frozen/blank until it completes. The browser cannot paint while `useLayoutEffect` runs synchronously. This is why it should only be used for quick DOM measurements/mutations.

**Q: Does React 18 change anything about these hooks?**
In Concurrent Mode, `useEffect` may run at different times than in Strict Mode. Strict Mode deliberately fires effects twice (mount → unmount → remount) in development to help find cleanup bugs. `useLayoutEffect` timing is unchanged.
