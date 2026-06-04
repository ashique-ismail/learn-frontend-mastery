# Whiteboard: Debugging Walkthrough

## Overview

Some interviews present broken code and ask you to diagnose and fix it live. Others describe a symptom and expect you to reason through the cause. This guide covers the systematic approach, the canonical bug categories interviewers reach for, and worked walkthroughs of real scenarios.

---

## The Debugging Protocol

**Say this out loud:**

1. **Reproduce it** — "Before I do anything, I want to confirm I can reproduce it with the simplest possible case."
2. **Form a hypothesis** — "Based on the symptom, my first guess is X."
3. **Gather evidence** — "I'd add a log / breakpoint here to check the value of Y."
4. **Narrow the blast radius** — "The bug is in this function, not the caller — I can rule out Z because..."
5. **Fix and verify** — "After the fix, I'd re-run these specific scenarios to confirm."

Never guess-and-patch. Walk the interviewer through reasoning, not random edits.

---

## Category 1: Async and Closure Bugs

### Classic: stale loop index

```javascript
// Bug: all buttons log "3"
for (var i = 0; i < 3; i++) {
  const btn = document.createElement('button');
  btn.textContent = `Button ${i}`;
  btn.addEventListener('click', () => console.log(i)); // ← i is shared
  document.body.appendChild(btn);
}

// Fix A: let (block scope)
for (let i = 0; i < 3; i++) { ... }

// Fix B: IIFE closure (legacy)
for (var i = 0; i < 3; i++) {
  (function(index) {
    btn.addEventListener('click', () => console.log(index));
  })(i);
}
```

**Explain:** `var` is function-scoped, so all three callbacks close over the same `i`. By the time any click fires, the loop has finished and `i === 3`. `let` creates a new binding per iteration.

### Race condition in async effects (React)

```jsx
// Bug: switching tabs quickly shows stale data
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then((r) => r.json())
      .then((data) => setUser(data)); // ← no cancellation guard
  }, [userId]);
}

// Fix: ignore stale responses
useEffect(() => {
  let stale = false;

  fetch(`/api/users/${userId}`)
    .then((r) => r.json())
    .then((data) => {
      if (!stale) setUser(data);
    });

  return () => { stale = true; }; // cleanup sets flag before next effect
}, [userId]);

// Fix 2: AbortController
useEffect(() => {
  const ctrl = new AbortController();
  fetch(`/api/users/${userId}`, { signal: ctrl.signal })
    .then((r) => r.json())
    .then(setUser)
    .catch((e) => { if (e.name !== 'AbortError') throw e; });
  return () => ctrl.abort();
}, [userId]);
```

### Infinite re-render loop

```jsx
// Bug: component re-renders forever
function App() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    setCount(count + 1); // ← each render triggers this, which triggers next render
  }, [count]); // ← count in deps array re-fires the effect
}

// Fix: functional update removes the dependency
useEffect(() => {
  setCount((prev) => prev + 1);
}, []); // ← only once

// Or: reconsider why you need this at all
```

---

## Category 2: `this` Binding Bugs

```javascript
class Timer {
  constructor() {
    this.count = 0;
  }

  start() {
    // Bug: this is undefined (or window in non-strict)
    setInterval(function() {
      console.log(this.count++); // TypeError
    }, 1000);
  }
}

// Fix A: arrow function
start() {
  setInterval(() => {
    console.log(this.count++); // ✓ — arrow inherits class `this`
  }, 1000);
}

// Fix B: bind
start() {
  setInterval(function() {
    console.log(this.count++);
  }.bind(this), 1000);
}

// Fix C: class field (arrow method)
start = () => {
  setInterval(() => console.log(this.count++), 1000);
}
```

---

## Category 3: Angular Change Detection Bugs

### OnPush component doesn't update

```typescript
// Bug: list renders stale after push mutation
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<li *ngFor="let item of items">{{ item }}</li>`,
})
export class ListComponent {
  @Input() items: string[] = [];
}

// In parent:
this.items.push('new item'); // ← mutates same array reference — OnPush skips diff

// Fix: return a new array reference
this.items = [...this.items, 'new item']; // ← new reference triggers OnPush
```

### Effect runs but view doesn't update (Signals + manual ChangeDetectorRef)

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `{{ label }}`,
})
export class LabelComponent implements OnInit {
  label = '';

  constructor(private cd: ChangeDetectorRef) {}

  ngOnInit(): void {
    fromEvent(window, 'resize').subscribe(() => {
      this.label = `${window.innerWidth}px`;
      // Bug: view doesn't update because OnPush and no CD trigger
      this.cd.markForCheck(); // ← Fix: schedule a check
    });
  }
}
```

---

## Category 4: Memory Leaks

### Leak 1: Unsubscribed observable (Angular)

```typescript
// Bug: subscription lives forever across route changes
@Component({ selector: 'app-ticker' })
export class TickerComponent implements OnInit {
  ngOnInit(): void {
    interval(1000).subscribe((n) => console.log(n)); // never cleaned up
  }
}

// Fix A: takeUntilDestroyed (Angular 16+)
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

ngOnInit(): void {
  interval(1000)
    .pipe(takeUntilDestroyed(this.destroyRef))
    .subscribe((n) => console.log(n));
}

// Fix B: explicit unsubscribe
private sub = new Subscription();
ngOnInit(): void { this.sub.add(interval(1000).subscribe(...)); }
ngOnDestroy(): void { this.sub.unsubscribe(); }
```

### Leak 2: DOM event listener not removed (vanilla JS)

```javascript
// Bug: old listener persists across re-renders / view transitions
function attachSearch() {
  document.addEventListener('keydown', handler);
  // ...if this runs multiple times, handlers stack up
}

// Fix: either remove explicitly or use { once: true }
function attachSearch() {
  document.addEventListener('keydown', handler);
  return () => document.removeEventListener('keydown', handler);
}

// In React
useEffect(() => {
  const cleanup = attachSearch();
  return cleanup;
}, []);
```

### Leak 3: Detached DOM nodes in cache

```javascript
// Bug: keeps reference to removed DOM tree
const cache = new Map();

function loadView(key) {
  if (cache.has(key)) return cache.get(key); // ← holds removed subtrees in memory
  const el = buildView(key);
  cache.set(key, el);
  return el;
}

// Fix: WeakRef + FinalizationRegistry OR simply limit cache size
// For most use cases: limit with LRU (see live-coding-patterns.md)
```

---

## Category 5: CSS / Layout Bugs

### z-index doesn't work

```
// Bug: modal (z-index: 9999) appears behind a header (z-index: 10)

Diagnosis:
  - The modal's parent has `transform: translateY(0)` or `filter: blur(0)` applied.
  - These CSS properties create a NEW STACKING CONTEXT.
  - Inside a stacking context, z-index is relative to siblings within that context,
    not to the global document.

Fix options:
  1. Move the modal to a portal (document.body) so it escapes the stacking context.
  2. Remove the property creating the stacking context on the parent.
  3. Use the <dialog> element — it renders in the top-layer, above all stacking contexts.
```

### Flex item overflows container

```css
/* Bug: long text in flex child causes parent overflow */
.container { display: flex; }
.child { /* no min-width set */ }

/* Diagnosis: flex items default to min-width: auto,
   which is "shrink to content width" — content can exceed container */

/* Fix A */
.child { min-width: 0; overflow: hidden; text-overflow: ellipsis; }

/* Fix B */
.child { overflow-wrap: break-word; }
```

---

## Walkthrough: Diagnosing a Slow React App

**Symptom:** Typing in a search input lags with ~200ms delay.

**Step 1 — React DevTools Profiler**
- Record while typing.
- Look for components that re-render on every keystroke that shouldn't.

**Step 2 — Identify the expensive render**
- Suppose `<ProductGrid>` renders 500 items and re-renders on every keystroke.
- `<ProductGrid>` receives `filters` as a prop; `filters` is re-created inline in JSX: `<ProductGrid filters={{ query, sort }} />`.

**Step 3 — Root cause**
- Object literal `{ query, sort }` is a new reference every render.
- `ProductGrid` is wrapped in `React.memo` but memo uses shallow comparison — `{}` !== `{}`.

**Step 4 — Fix**
```jsx
// Stabilize the reference
const filters = useMemo(() => ({ query, sort }), [query, sort]);
return <ProductGrid filters={filters} />;
```

**Step 5 — Verify**
- Re-profile: `<ProductGrid>` no longer appears in the flame chart on every keystroke.

---

## Interview Tips for Debugging Questions

- **Announce your mental model** before diving into the code: "My first suspicion is a closure issue."
- **Use the elimination technique**: "This can't be a timing bug because it fails synchronously."
- **Reference browser/framework tools**: DevTools Performance, React Profiler, Angular DevTools, memory snapshots.
- **Distinguish symptoms from causes**: "The symptom is extra renders; the cause is the unstable reference."
- **Propose a test** even if you can't run it: "I'd add a console.log here to confirm the value at this point."
