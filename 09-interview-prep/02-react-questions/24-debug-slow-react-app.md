# Debugging a Slow React App — Systematic Approach

## The Idea

**In plain English:** Debugging a slow React app means finding the specific parts of your website that are doing too much unnecessary work — like re-drawing things on screen that didn't actually change — and fixing them so the app responds faster.

**Real-world analogy:** Imagine a restaurant kitchen that's slow. Every time one customer orders a burger, the chef also re-cooks every other table's food "just in case" it changed. A smart manager watches the kitchen, figures out which stations are wasting effort, and tells each chef to only re-cook when their specific order actually changes.

- The restaurant kitchen = the React app rendering on screen
- A chef re-cooking unchanged food = a React component re-rendering when nothing changed
- The smart manager watching = the React DevTools Profiler tracking what re-renders and why
- Telling chefs to only cook when needed = using `React.memo`, `useMemo`, or splitting Context to stop unnecessary work

---

## Step 1: Identify What "Slow" Means

Before profiling, clarify the symptom:
- **Slow initial load** → bundle size, SSR, network
- **Slow interactions** → unnecessary re-renders, long tasks, layout thrash
- **Slow after time** → memory leaks, growing state
- **Slow for some users** → device/network, not your dev machine

---

## Step 2: Measure with Production Metrics

**Core Web Vitals (real users):**
- LCP (Largest Contentful Paint) — load speed
- INP (Interaction to Next Paint) — responsiveness
- CLS (Cumulative Layout Shift) — visual stability

Tools: Lighthouse (in Chrome DevTools), PageSpeed Insights, web-vitals library

```js
import { onINP, onLCP, onCLS } from 'web-vitals';

onINP(({ value }) => {
  if (value > 200) console.warn('Slow interaction:', value, 'ms');
});
```

---

## Step 3: React DevTools Profiler

The most direct tool for React-specific performance.

```
Chrome → React DevTools → Profiler tab → Record → Interact → Stop
```

What to look for:
- **Grey components** — didn't re-render (good)
- **Yellow/orange components** — re-rendered this commit
- **Ranked chart** — which components took the most time
- **"Why did this render?"** — shows which prop/state/context changed

```jsx
// Enable why-did-you-render in development
import whyDidYouRender from '@welldone-software/why-did-you-render';
whyDidYouRender(React, { trackAllPureComponents: true });
```

---

## Step 4: Identify Unnecessary Re-Renders

### Cause 1: Context Causing Widespread Re-Renders

```jsx
// ❌ Every consumer re-renders when ANY value changes
const AppContext = createContext({ user, theme, cart });

// ✅ Split contexts by update frequency
const UserContext = createContext(user);
const ThemeContext = createContext(theme);
const CartContext = createContext(cart);
```

### Cause 2: Unstable Props/References

```jsx
// ❌ New array on every render → child always re-renders
function Parent() {
  return <Child items={[1, 2, 3]} />;  // new reference every render
}

// ✅ Stable reference
const ITEMS = [1, 2, 3];
function Parent() {
  return <Child items={ITEMS} />;
}
```

### Cause 3: Missing Memoization

```jsx
// ❌ filterItems runs on every render
function ProductList({ products, filter }) {
  const filtered = filterItems(products, filter); // expensive!
  return <List items={filtered} />;
}

// ✅ Only recomputes when inputs change
function ProductList({ products, filter }) {
  const filtered = useMemo(
    () => filterItems(products, filter),
    [products, filter]
  );
  return <List items={filtered} />;
}
```

---

## Step 5: Fix Long Tasks (INP Issues)

Use Chrome DevTools → Performance tab. Look for **red triangles** on the main thread (tasks > 50ms).

Common causes:
- Rendering large lists without virtualization
- Synchronous computations in event handlers
- Third-party scripts blocking the main thread

**Virtualization for long lists:**
```jsx
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualList({ items }) {
  const parentRef = useRef(null);
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
  });

  return (
    <div ref={parentRef} style={{ height: 400, overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize() }}>
        {virtualizer.getVirtualItems().map(item => (
          <div key={item.key} style={{ transform: `translateY(${item.start}px)` }}>
            {items[item.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

**Deferred updates with `useTransition`:**
```jsx
const [isPending, startTransition] = useTransition();

function handleInput(value) {
  setInputValue(value);             // urgent — update immediately
  startTransition(() => {
    setSearchResults(search(value)); // non-urgent — can be deferred
  });
}
```

---

## Step 6: Bundle Size Analysis

```bash
npx next build --analyze   # Next.js
npx vite-bundle-visualizer  # Vite

# Or with source-map-explorer
npm run build
npx source-map-explorer 'build/static/js/*.js'
```

Common culprits:
- Moment.js (600 KB) → use date-fns or Day.js
- Lodash (entire bundle) → import specific functions `import debounce from 'lodash/debounce'`
- Polyfills for modern browsers
- Unoptimized third-party libraries

---

## Step 7: Code Splitting

```jsx
// Route-level splitting
const AdminPanel = lazy(() => import('./AdminPanel'));

// Feature-level splitting
const RichEditor = lazy(() => import('./RichEditor'));

function Editor({ isRich }) {
  if (!isRich) return <SimpleEditor />;
  return (
    <Suspense fallback={<Skeleton />}>
      <RichEditor />
    </Suspense>
  );
}
```

---

## Common Interview Questions

**Q: How do you find which components are causing unnecessary re-renders?**
React DevTools Profiler + `why-did-you-render` library. The Profiler shows which components re-rendered in each commit and why (prop change, state change, context change, parent re-render).

**Q: When should you NOT use `useMemo` and `useCallback`?**
When the computation is cheap (cheaper than the memoization overhead), when values change frequently (memoization never hits), or when the component rarely re-renders anyway. Premature memoization adds complexity without benefit.

**Q: What's the most impactful thing to fix first?**
Usually unnecessary re-renders from unstable Context values or missing `React.memo` on expensive child components. Bundle size matters for initial load; re-render frequency matters for interaction speed.
