# Production Failure: Memory Leak Patterns

## Overview

Memory leaks in frontend apps cause progressive slowdown, tab crashes, and degraded performance that only manifests after extended use — which means they often slip through QA and only surface from user complaints. These are the real-world patterns, not the textbook examples.

---

## Leak 1: Event Listeners in React Portals

### The Bug

```tsx
// Modal rendered in a Portal — unmounts from React tree
function Modal({ onClose }: { onClose: () => void }) {
  useEffect(() => {
    // Listens to keydown on document to close on Escape
    document.addEventListener('keydown', (e) => {
      if (e.key === 'Escape') onClose();
    });
    // ❌ No cleanup — listener stays after modal unmounts
  }, []);

  return createPortal(<div className="modal">...</div>, document.body);
}
```

**Why it leaks:** React cleans up Portal DOM nodes, but does NOT clean up listeners registered on `document` outside the component's DOM tree. Every time the modal mounts, a new listener is added. After 10 modal opens: 10 listeners. After 100: 100. Each holds a closure over `onClose`, preventing GC.

### The Fix

```tsx
useEffect(() => {
  const handler = (e: KeyboardEvent) => {
    if (e.key === 'Escape') onClose();
  };
  document.addEventListener('keydown', handler);
  return () => document.removeEventListener('keydown', handler); // ← cleanup
}, [onClose]);
```

---

## Leak 2: RxJS Subscriptions in Angular

### The Bug

```typescript
@Component({ selector: 'app-ticker' })
export class TickerComponent implements OnInit {
  prices: number[] = [];

  ngOnInit() {
    // Route navigates away → component destroyed → subscription keeps running
    this.priceService.stream$.subscribe(price => {
      this.prices.push(price);     // mutates detached component
      this.cdr.detectChanges();    // triggers change detection on dead component
    });
  }
}
```

**Why it leaks:** The subscription continues emitting after the component is destroyed. The component instance is referenced by the subscription closure — it can't be garbage collected. `detectChanges()` on a destroyed view throws errors. In a router-heavy app opening and closing this route 50 times: 50 active subscriptions, 50 component instances in memory.

### Detection

Angular DevTools → Profiler shows memory growing with navigation. Chrome Memory tab → take heap snapshot → search for component class name → multiple instances where one is expected.

### Fix Patterns

```typescript
// Option A: takeUntilDestroyed (Angular 16+) — preferred
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

constructor() {
  this.priceService.stream$
    .pipe(takeUntilDestroyed())
    .subscribe(price => this.prices.push(price));
}

// Option B: DestroyRef manually
private destroyRef = inject(DestroyRef);
ngOnInit() {
  this.priceService.stream$
    .pipe(takeUntilDestroyed(this.destroyRef))
    .subscribe(...);
}

// Option C: async pipe (auto-unsubscribes)
// template: {{ prices$ | async }}
prices$ = this.priceService.stream$.pipe(
  scan((acc, price) => [...acc, price], [] as number[])
);
```

---

## Leak 3: Closures Capturing Large Objects

### The Bug

```javascript
function setupTable(data) {
  const largeDataset = processData(data); // 50MB dataset

  const button = document.querySelector('#export');
  button.addEventListener('click', () => {
    // Only uses one field — but closes over ALL of largeDataset
    exportToCSV(largeDataset.headers);
  });
}

setupTable(bigData); // largeDataset now lives until button is removed
```

**Why it leaks:** The event listener closure holds a reference to `largeDataset` even though it only needs `largeDataset.headers`. As long as the button exists, `largeDataset` (50MB) can't be GC'd.

### Fix

```javascript
function setupTable(data) {
  const largeDataset = processData(data);
  const headers = largeDataset.headers; // extract only what's needed

  document.querySelector('#export').addEventListener('click', () => {
    exportToCSV(headers); // closes over only `headers`, not the whole dataset
  });
  // largeDataset can now be GC'd after setupTable returns
}
```

---

## Leak 4: Detached DOM Nodes Held in Maps/Arrays

### The Bug

```javascript
// Caching DOM nodes for "performance"
const nodeCache = new Map();

function renderItem(id, container) {
  const node = document.createElement('div');
  node.textContent = id;
  container.appendChild(node);
  nodeCache.set(id, node); // ← holds reference forever
}

// Later: container is removed from DOM
container.remove();
// But nodeCache still holds all the nodes → they can't be GC'd
// The entire detached subtree stays in memory
```

**Detection in DevTools:**
Memory tab → Heap snapshot → filter "Detached" → shows all detached DOM nodes still in memory.

### Fix

```javascript
// Option A: WeakMap (allows GC when the node has no other references)
const nodeCache = new WeakMap(); // keys are objects — GC'd when key is collected

// Option B: Explicit cleanup
function cleanupContainer(container) {
  container.querySelectorAll('[data-id]').forEach(node => {
    nodeCache.delete(node.dataset.id);
  });
  container.remove();
}

// Option C: Never cache DOM nodes — reconstruct from data model
```

---

## Leak 5: CSS-in-JS Style Sheet Accumulation

### The Bug

```tsx
// styled-components or Emotion generates a new class on every unique prop combination
function ListItem({ index }: { index: number }) {
  return (
    <StyledItem delay={index * 0.1}> {/* new class for EVERY index value */}
      ...
    </StyledItem>
  );
}

const StyledItem = styled.div<{ delay: number }>`
  animation-delay: ${p => p.delay}s; // unique per index
`;

// List with 10,000 items: 10,000 CSS classes injected into <style> tags
// Stylesheet grows unboundedly. Browser style recalculation slows to a crawl.
```

**Why it's bad:** CSS-in-JS libraries inject styles into `<style>` tags in the `<head>`. They don't remove old rules when components unmount. A virtualized list re-rendering items with different `index` props generates an ever-growing stylesheet.

### Fix

```tsx
// Option A: CSS custom properties for dynamic values — one class, variable changes
const StyledItem = styled.div`
  animation-delay: var(--delay);
`;
<StyledItem style={{ '--delay': `${index * 0.1}s` }} />

// Option B: CSS modules + inline style for dynamic parts
<div className={styles.item} style={{ animationDelay: `${index * 0.1}s` }} />

// Option C: Atomic CSS (Tailwind) — finite class set, no runtime generation
```

---

## Leak 6: Timer and Interval Accumulation

### The Bug

```tsx
function LiveClock() {
  const [time, setTime] = useState(new Date());

  // Bug: running in StrictMode? Double-invoked, one interval leaks.
  // Bug: component re-renders? No — but what if this is in a conditional?
  useEffect(() => {
    setInterval(() => setTime(new Date()), 1000);
    // ❌ interval ID not stored, can't be cleared
  }, []);
}

// Worse pattern: interval created inside a callback, not useEffect
function Dashboard() {
  const startPolling = () => {
    setInterval(fetchData, 5000); // new interval every time startPolling() is called
  };
  return <button onClick={startPolling}>Start</button>;
  // User clicks 5 times: 5 intervals running simultaneously
}
```

### Fix

```tsx
useEffect(() => {
  const id = setInterval(() => setTime(new Date()), 1000);
  return () => clearInterval(id); // ← always store and clear
}, []);

// For polling: use a ref to ensure only one interval exists
const intervalRef = useRef<number | null>(null);
const startPolling = () => {
  if (intervalRef.current) return; // already running
  intervalRef.current = window.setInterval(fetchData, 5000);
};
const stopPolling = () => {
  clearInterval(intervalRef.current!);
  intervalRef.current = null;
};
```

---

## How to Find Memory Leaks in Production

### Chrome DevTools Workflow

```
1. Performance Monitor (DevTools → More Tools → Performance Monitor)
   → Watch JS Heap Size while navigating the app
   → If it grows and never shrinks after GC: there's a leak

2. Heap Snapshots (Memory tab)
   → Take snapshot 1 (baseline)
   → Do the suspect operation (navigate, open modal, etc.)
   → Take snapshot 2
   → Comparison view: objects added between snapshots
   → Sort by "# Delta" — large positive counts = leak

3. Allocation Timeline
   → Record while performing operation
   → Blue bars = allocations that were later freed (good)
   → Gray bars = allocations still in memory after 1+ GC cycles (investigate)

4. Detached DOM filter
   → Heap snapshot → filter by "Detached HTMLElement"
   → Each result is a DOM node no longer in the document but still in memory
   → Expand to see what's holding a reference
```

### In Production (PerformanceObserver)

```javascript
// Track heap growth over time — alert if crossing threshold
if ('memory' in performance) {
  setInterval(() => {
    const { usedJSHeapSize, jsHeapSizeLimit } = performance.memory;
    const usage = usedJSHeapSize / jsHeapSizeLimit;
    if (usage > 0.85) {
      analytics.track('high_memory_usage', { usage, page: location.pathname });
    }
  }, 30_000);
}
```

---

## Interview Questions

**Q: How would you find a memory leak that only manifests after 30 minutes of usage?**
A: Use Chrome's Allocation Timeline — record for 30 minutes, then look for allocations that don't get freed across GC cycles. Alternatively, automate a Puppeteer script that navigates repeatedly and takes heap snapshots at intervals, comparing growth. Look for growing `Detached HTMLElement` or `EventListener` counts between snapshots.

**Q: Why does the async pipe in Angular prevent memory leaks?**
A: `async` pipe calls `subscribe()` and stores the subscription internally. When the component is destroyed, Angular's change detection infrastructure calls `ngOnDestroy` on the pipe, which calls `unsubscribe()`. You never manage the subscription lifecycle manually — it's handled by the pipe's implementation.

**Q: When would you choose WeakMap over Map for caching?**
A: When the cache keys are objects (DOM nodes, component instances) and you want the cache entries to be automatically garbage collected when the key object is no longer referenced elsewhere. Map holds a strong reference to its keys — they can never be GC'd. WeakMap holds weak references — if the key has no other references, the entry disappears and the value is eligible for GC.
