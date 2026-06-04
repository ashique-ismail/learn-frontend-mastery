# Qwik and Resumability

## Overview

Qwik is a JavaScript framework that rethinks how applications start in the browser. Traditional frameworks (React, Angular, Vue) require **hydration** — downloading and executing all component JavaScript before the app becomes interactive. On slow connections or low-end devices, this means seconds of frozen UI. Qwik introduces **resumability**: the server serializes application state and component logic references (QRLs) into the HTML, and the browser "resumes" where the server left off with zero JavaScript execution on load.

```
Hydration (React/Angular):
  HTML received ──► Parse HTML ──► Download JS (100–500KB)
                                      │
                                   Execute JS (re-render entire tree)
                                      │
                                   Register all event listeners
                                      │
                                   App interactive (1–5s on 3G)

Resumability (Qwik):
  HTML received ──► Parse HTML ──► App interactive (< 100ms)
                       │
                  Event listener found? ──► Download ONLY that handler's JS
                                              │
                                           Execute handler
```

## Why Hydration is Expensive

```
React SSR hydration steps:
1. Server renders HTML string (fast)
2. Browser downloads entire React bundle (slow — often 150KB+ gzipped)
3. React walks the entire component tree in JS (CPU-intensive)
4. React reconciles virtual DOM with actual DOM
5. React attaches event listeners to ALL interactive elements
6. App is now interactive

Cost: proportional to NUMBER OF COMPONENTS, not user interaction
Even if user clicks nothing, ALL code executes
```

## Qwik's Core Insight: Lazy Execution

In Qwik, code only executes when it's needed:

```typescript
// This component's JavaScript is NOT downloaded on page load
// Only the HTML is rendered. The onClick handler is downloaded
// ONLY when the user actually clicks the button.

import { component$, useSignal } from '@builder.io/qwik';

export const Counter = component$(() => {
  const count = useSignal(0);

  return (
    <div>
      <p>Count: {count.value}</p>
      {/* onClick$ — the $ signals lazy loading boundary */}
      <button onClick$={() => count.value++}>
        Increment
      </button>
    </div>
  );
});
```

The HTML output includes a reference to the handler, not the handler itself:

```html
<!-- Qwik serialized output -->
<div q:id="0">
  <p>Count: <!--t=count-->0</p>
  <button on:click="./chunk-abc123.js#Counter_onClick_xyz">
    Increment
  </button>
</div>
```

## QRL — Qwik URL

A QRL (Qwik URL) is a lazy reference to a function. It encodes both the chunk URL and the symbol to import:

```typescript
// Internal QRL structure:
// "./chunk-abc123.js#Counter_onClick_xyz"
//  ─────────────────  ───────────────────
//  chunk to download   exported symbol name

// Qwik creates QRLs automatically for $ suffix functions
import { $, QRL } from '@builder.io/qwik';

// Manually create a QRL
const handler: QRL<() => void> = $(() => {
  console.log('Lazily loaded!');
});

// QRLs can be passed as props
interface MyProps {
  onClick: QRL<() => void>;
}
```

## The `$` Suffix Convention

The `$` suffix marks a **lazy boundary** — Qwik's optimizer splits code at these boundaries:

```typescript
import { component$, useStore, useTask$, $ } from '@builder.io/qwik';

export const App = component$(() => {
  // component$ — the component body is in its own chunk
  const state = useStore({ count: 0, name: 'World' });

  // useTask$ — side effects, can be async, serializable
  useTask$(({ track }) => {
    track(() => state.count);
    // Runs when count changes — this code is in its own chunk too
    console.log('Count changed:', state.count);
  });

  // Event handler in its own chunk
  const increment = $(() => {
    state.count++;
  });

  return (
    <div>
      <h1>Hello, {state.name}!</h1>
      <p>Count: {state.count}</p>
      <button onClick$={increment}>+1</button>
      {/* Inline handler — also its own chunk */}
      <button onClick$={() => { state.count = 0; }}>Reset</button>
    </div>
  );
});
```

## Signals in Qwik

Qwik uses signals for fine-grained reactivity — only the minimal DOM is updated:

```typescript
import { component$, useSignal, useComputed$, useStore } from '@builder.io/qwik';

export const ProductCard = component$(() => {
  // Primitive signal
  const quantity = useSignal(1);
  const price = useSignal(29.99);

  // Computed signal — recalculates only when dependencies change
  const total = useComputed$(() => quantity.value * price.value);

  // Store — reactive plain object
  const user = useStore({
    name: 'Alice',
    preferences: {
      currency: 'USD',
    },
  });

  return (
    <div>
      {/* Only this text node re-renders when quantity changes */}
      <span>Qty: {quantity.value}</span>

      {/* Only this text node re-renders when total changes */}
      <strong>${total.value.toFixed(2)}</strong>

      <button onClick$={() => quantity.value++}>+</button>
      <button onClick$={() => quantity.value = Math.max(1, quantity.value - 1)}>-</button>
    </div>
  );
});
```

### Signal Comparison with React

```typescript
// React — coarse-grained (re-renders entire component subtree)
function Counter() {
  const [count, setCount] = useState(0);
  // When count changes, ALL of Counter() re-runs
  // Including creating new objects, calling other hooks, diffing vDOM
  return (
    <div>
      <ExpensiveChild /> {/* Re-renders even though count doesn't affect it */}
      <p>{count}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}

// Qwik — fine-grained (only the DOM node that uses the signal re-renders)
const Counter = component$(() => {
  const count = useSignal(0);
  return (
    <div>
      <ExpensiveChild /> {/* Does NOT re-render when count changes */}
      <p>{count.value}</p> {/* Only this text node updates */}
      <button onClick$={() => count.value++}>+</button>
    </div>
  );
});
```

## Server-Side Rendering and State Serialization

Qwik serializes component state into the HTML so the browser can resume without re-running server code:

```typescript
// routes/index.tsx — Qwik City (the meta-framework)
import { component$, useSignal } from '@builder.io/qwik';
import { routeLoader$ } from '@builder.io/qwik-city';

// Runs on server — data is serialized into HTML
export const useProductData = routeLoader$(async ({ params }) => {
  const product = await db.products.findById(params.id);
  return product;
});

export default component$(() => {
  const product = useProductData();
  const quantity = useSignal(1);

  // product.value is available immediately — no loading state needed
  // It was serialized into the HTML by the server
  return (
    <div>
      <h1>{product.value.name}</h1>
      <p>{product.value.description}</p>
      <input
        type="number"
        value={quantity.value}
        onInput$={(e) => {
          quantity.value = parseInt((e.target as HTMLInputElement).value);
        }}
      />
    </div>
  );
});
```

The serialized HTML looks like:

```html
<script type="qwik/json">
{
  "refs": {"0": "product", "1": "quantity"},
  "ctx": {"product": {"name": "Widget", "price": 29.99}, "quantity": 1},
  "objs": [...]
}
</script>
```

## Qwik City — The Meta-Framework

```typescript
// routes/products/[id]/index.tsx
import { component$ } from '@builder.io/qwik';
import { routeLoader$, routeAction$, Form } from '@builder.io/qwik-city';
import { z } from 'zod';

export const useProduct = routeLoader$(async ({ params, error }) => {
  const product = await fetchProduct(params.id);
  if (!product) throw error(404, 'Product not found');
  return product;
});

// Server action — runs on server when form is submitted
export const useAddToCart = routeAction$(
  async (data, { cookie }) => {
    const cart = cookie.get('cart')?.json() ?? [];
    cart.push({ productId: data.productId, quantity: data.quantity });
    cookie.set('cart', JSON.stringify(cart), { path: '/' });
    return { success: true };
  },
  // Zod validation
  z.object({
    productId: z.string(),
    quantity: z.coerce.number().min(1).max(99),
  })
);

export default component$(() => {
  const product = useProduct();
  const addToCart = useAddToCart();

  return (
    <div>
      <h1>{product.value.name}</h1>
      <Form action={addToCart}>
        <input type="hidden" name="productId" value={product.value.id} />
        <input type="number" name="quantity" defaultValue={1} />
        <button type="submit">Add to Cart</button>
        {addToCart.value?.success && <p>Added!</p>}
      </Form>
    </div>
  );
});
```

## Resumability vs Hydration Comparison

```
Metric                    | React (hydration) | Qwik (resumability)
──────────────────────────|───────────────────|────────────────────
JS on initial load        | Full bundle       | ~1KB bootstrap
Time to interactive (3G)  | 3–8s              | < 200ms
First interaction latency | 0ms (hydrated)    | 50–200ms (lazy load handler)
Re-render cost            | Component subtree | Fine-grained (signal-level)
Mental model              | VDOM diffing      | Resumable state machine
SSR requirement           | Optional (nice)   | Required (fundamental)
Progressive hydration     | Manual            | Automatic/always
Code splitting            | Manual            | Automatic ($ boundaries)
```

## Qwik Optimizer

The Qwik optimizer is a build-time Vite plugin that transforms `$` calls into lazy chunk references:

```typescript
// Before optimization (your code):
const App = component$(() => {
  return <button onClick$={() => alert('clicked')}>Click</button>;
});

// After optimization (what ships to browser):
// chunk-main.js
const App = component(qrl('./chunk-handlers-abc.js', 's_onClick'));

// chunk-handlers-abc.js (only downloaded when button is clicked)
export const s_onClick = () => alert('clicked');
```

## Prefetching — Eliminating Interaction Latency

Qwik can speculatively prefetch interaction chunks during idle time:

```typescript
// qwik-city.config.ts
import { defineConfig } from 'vite';
import { qwikCity } from '@builder.io/qwik-city/vite';

export default defineConfig({
  plugins: [
    qwikCity({
      // Strategy: prefetch visible component handlers during idle
      prefetchStrategy: {
        implementation: {
          linkInsert: 'html-append',
          workerFetchInsert: 'always',
          prefetchEvent: 'always',
        },
      },
    }),
  ],
});
```

## When to Choose Qwik

```
Good fit for Qwik:
  ✓ Content-heavy sites (news, e-commerce, marketing)
  ✓ Apps with users on slow connections / low-end devices
  ✓ High Core Web Vitals requirements (LCP, INP, TBT)
  ✓ Large applications where tree-shaking isn't enough

Less ideal for Qwik:
  ✗ Highly interactive SPAs (dashboards, editors) — hydration cost amortizes
  ✗ Small apps where bundle size doesn't matter
  ✗ Teams deeply invested in React ecosystem
  ✗ Apps requiring rich third-party JS libraries (many assume hydrated DOM)
```

## Interview Questions

1. **What is the fundamental difference between resumability and hydration?**
   Hydration re-executes all component code in the browser to reconstruct framework state, then attaches event listeners. It downloads and runs code proportional to the number of components regardless of user interaction. Resumability serializes the framework state (signal values, component references as QRLs) into HTML. The browser restores state from the serialized data without re-executing any code. Event handlers are downloaded and executed only when triggered.

2. **What is a QRL and how does Qwik's optimizer create them?**
   A QRL (Qwik URL) is a lazy reference to a function — a string containing the chunk URL and the exported symbol name (e.g., `"./chunk-abc.js#handler_xyz"`). The Qwik optimizer, a Vite/Rollup plugin, scans code at build time and extracts functions at `$` boundaries into separate chunks, replacing them with QRL references. At runtime, Qwik resolves QRLs by dynamic `import()` when needed.

3. **How does Qwik's fine-grained reactivity differ from React's component re-rendering model?**
   React re-renders at the component level — a state change triggers re-execution of the component function and all its children (unless memoized). Qwik uses signals: each signal maintains a set of subscribers (specific DOM nodes or computed values). A signal change updates only the subscribed DOM nodes directly, bypassing the component function entirely. This eliminates the need for `memo`, `useMemo`, and `useCallback` optimizations.

4. **What are the trade-offs of Qwik's lazy execution for first interactions?**
   First interaction latency: when a user clicks a button for the first time, Qwik must download and execute the handler chunk before responding. This can be 50–200ms on slow connections — longer than React's 0ms (since it's already hydrated). Qwik mitigates this with: (1) prefetching handlers during idle time, (2) chunk size — handlers are tiny (often < 1KB), (3) HTTP/2 push and service worker prefetch. For most users on reasonable connections, this is imperceptible.

5. **Explain how Qwik serializes state into HTML and what constraints this places on the state.**
   Qwik uses `JSON.stringify`-like serialization with a custom serializer that handles circular references and special objects. Constraints: (1) State must be JSON-serializable — no functions, Dates become strings, Maps/Sets need special handling. (2) Closures cannot be serialized — event handlers must reference state through useStore/useSignal, not local variables. (3) Sensitive data in state will be in the HTML — don't serialize secrets.

6. **How does Qwik City's `routeLoader$` differ from Next.js `getServerSideProps`?**
   Both run on the server before rendering. Key differences: (1) routeLoader$ data is serialized into the HTML and immediately available without a client-side fetch — it's part of the resumable state. (2) Multiple loaders can run in parallel without coordination. (3) Loaders are co-located with components. (4) In Next.js App Router, server components achieve similar colocation, but the data isn't "resumable" in Qwik's sense — React still needs to hydrate the component tree.

7. **What happens when a Qwik component has a prop that is a function? How are callbacks serialized?**
   Functions cannot be serialized as JSON, so Qwik uses QRLs. When you write `onClick$={myHandler}`, the `$` suffix tells the optimizer to extract `myHandler` into a chunk. The prop is stored as a QRL string. When the event fires, Qwik resolves the QRL, downloads the chunk, imports the handler, and calls it with the event. This is why all event handlers in Qwik must use the `$` suffix — non-$ functions are assumed to be called immediately during SSR render only.

8. **Compare Qwik's approach to code splitting with React.lazy(). What does Qwik automate that React requires manual configuration for?**
   `React.lazy()` splits at the component level — you must manually identify which components to lazy-load and wrap them in `Suspense`. Qwik's optimizer automatically splits at every `$` boundary — not just components, but also individual event handlers, effects, and computed values. A single Qwik component might produce 5–10 chunks (one per handler). React lazy loading loads an entire component with all its handlers; Qwik loads only the specific handler that was triggered.
