# Resumability

## Overview

Resumability is a rendering paradigm where the server serializes the entire application state — including event listeners, component boundaries, and execution context — directly into HTML. The client never re-runs framework bootstrapping or component rendering code. Instead, it "resumes" from exactly where the server left off, executing only the code paths triggered by user interaction.

This is a fundamental departure from hydration, which re-executes server work on the client unconditionally.

---

## The Hydration Problem: Paying Twice

Traditional SSR with hydration has a structural performance tax that scales with application size.

### What Hydration Actually Does

```
Server                          Client
──────                          ──────
1. Execute all components       1. Download full framework (~40KB+)
2. Render to HTML string        2. Download all component code
3. Send HTML + JS bundles       3. Parse and evaluate all JS
4. (wait for client)            4. Re-execute all components
                                5. Attach event listeners
                                6. Page is interactive ← TTI
```

Steps 1–5 on the client re-do work the server already completed. The HTML is there, looking correct, but the browser cannot interact with it until the framework has walked the entire component tree again to "claim" it.

### Why This Matters at Scale

```typescript
// A typical React page with SSR:
// The server renders this once.
// The client ALSO renders this — just to attach onClick to <button>.

function ProductPage({ product }: Props) {
  const [qty, setQty] = useState(1);

  return (
    <div>
      {/* Static: header, images, descriptions, specs — none of these
          need JS, yet the client must traverse them all during hydration
          just to find the interactive parts. */}
      <Header />
      <ProductImages images={product.images} />
      <ProductDescription text={product.description} />
      <SpecTable specs={product.specs} />
      <Reviews productId={product.id} />

      {/* This single button is why we're doing all this work: */}
      <QuantityPicker value={qty} onChange={setQty} />
      <AddToCartButton productId={product.id} qty={qty} />
    </div>
  );
}
```

The browser must evaluate and re-render `<Header>`, `<ProductImages>`, `<Reviews>`, and every static node just to reach `<AddToCartButton>`. On a mid-range Android device this can take 3–8 seconds for large pages.

### The Hydration Equation

```
TTI ≈ (JS download time) + (JS parse time) + (full tree traversal time)

All three terms scale with component count, regardless of how many
components are actually interactive.
```

This is the problem resumability solves.

---

## What Resumability Actually Is

Resumability flips the model: instead of the client reconstructing application state from code, the server encodes the complete state into the HTML payload. The client deserializes and is instantly ready — no framework startup, no re-rendering.

```
Server                              HTML Output
──────                              ───────────
1. Execute components               <button
2. Capture component state             on:click="./chunk-abc.js#handleClick[s1]"
3. Serialize event listeners    →      q:id="1"
4. Serialize component state           q:key="product-42"
5. Embed everything in HTML         >
                                      Add to Cart
                                    </button>

Client
──────
1. Parse HTML (browser native)
2. Register global event listener (one, for the whole page)
3. TTI ← instant. Zero framework code downloaded yet.

User clicks "Add to Cart":
4. Download only chunk-abc.js
5. Deserialize state s1
6. Execute handleClick
```

The key insight: the client does not need to know about `<Header>`, `<ProductImages>`, or any other component that has no interaction. That code is never downloaded.

---

## How Qwik Implements Resumability

Qwik is the primary framework built around resumability. Its approach involves three interconnected mechanisms: state serialization, lazy execution, and optimizer boundaries.

### State Serialization into HTML

Qwik embeds a JSON blob called `qwik/json` into the page that captures the complete application state snapshot.

```html
<!-- Server-rendered Qwik output -->
<html q:version="1.0" q:base="/build/">
  <head>
    <!-- No blocking scripts. No framework JS downloaded. -->
  </head>
  <body q:ctx="">

    <div q:id="0">
      <button
        on:click="./q-chunk-BkqXrCXb.js#AddToCart_onClick_1_click[0 1]"
        q:id="1"
      >
        Add to Cart
      </button>

      <span q:id="2">Qty: 1</span>
    </div>

    <!-- Full application state is embedded here -->
    <script type="qwik/json">
    {
      "__refs": ["#1", "#2"],
      "objs": [
        {"productId": "42", "qty": 1},
        {"cartCount": 0}
      ],
      "subs": [["2 #text 0 qty"]]
    }
    </script>

    <!-- Single global listener — the ONLY script tag at startup -->
    <script>
      window.qwikevents = window.qwikevents || [];
      window.qwikevents.push("click");
    </script>
  </body>
</html>
```

What's happening here:
- `on:click` stores a URL to the specific JS chunk and the exact exported symbol to call
- `q:id` attributes create a stable reference map for DOM nodes
- `qwik/json` contains reactive state objects and their subscriptions
- There is exactly one tiny inline script that registers global event delegation

### The $ Suffix: Optimizer Boundary Markers

The `$` suffix in Qwik is not syntax sugar — it is an explicit signal to the Qwik optimizer to extract the function into a separately downloadable chunk.

```typescript
import { component$, useStore, $ } from '@builder.io/qwik';

// component$ tells the optimizer:
// "The render function body is a lazy boundary.
//  Do NOT include this in the initial bundle."
export const AddToCart = component$((props: { productId: string }) => {
  const state = useStore({ qty: 1, loading: false });

  // $ here tells the optimizer:
  // "Extract this handler into its own chunk.
  //  Only download it when this specific interaction fires."
  const handleAdd = $(async () => {
    state.loading = true;
    await addToCart(props.productId, state.qty);
    state.loading = false;
  });

  // This render function itself is also lazy —
  // it runs on the server and its output is serialized.
  // It does NOT re-run on the client unless state changes.
  return (
    <button onClick$={handleAdd} disabled={state.loading}>
      {state.loading ? 'Adding...' : 'Add to Cart'}
    </button>
  );
});
```

Without `$`, the optimizer would bundle everything together and lose laziness. The `$` suffix is a contract: "this function can be serialized as a reference and loaded on demand."

The optimizer transforms this at build time:

```typescript
// What the Qwik optimizer generates (simplified):

// Chunk: q-AddToCart-render.js
// Only loaded if the component needs to re-render
export const AddToCart_render = () => { ... };

// Chunk: q-AddToCart-handler.js
// Only loaded when the click actually fires
export const AddToCart_onClick_1_click = async (state, props) => {
  state.loading = true;
  await addToCart(props.productId, state.qty);
  state.loading = false;
};
```

### How Event Handling Works Without Framework Startup

```typescript
// Qwik's global event listener (the ONLY thing that runs at startup)
// Simplified version of what Qwik injects:

document.addEventListener('click', async (event) => {
  const target = event.target as Element;
  const onClickAttr = target.closest('[on:click]')?.getAttribute('on:click');

  if (!onClickAttr) return;

  // Parse: "./q-chunk-BkqXrCXb.js#AddToCart_onClick_1_click[0 1]"
  const [chunkUrl, rest] = onClickAttr.split('#');
  const [symbol, stateRefs] = rest.split('[');

  // Restore state from qwik/json using the refs [0 1]
  const state = deserializeStateRefs(stateRefs);

  // Only NOW download the handler chunk
  const module = await import(chunkUrl);
  const handler = module[symbol];

  // Execute with restored state
  await handler(state, event);
});
```

This is a significant architectural shift: the framework is event-driven rather than eagerly initialized.

### useStore and Reactive Serialization

```typescript
import { component$, useStore, useSignal } from '@builder.io/qwik';

export const ProductPage = component$(() => {
  // useStore creates a reactive proxy.
  // On the server: its value is captured in qwik/json.
  // On the client: it's restored from that JSON, not re-computed.
  const state = useStore({
    qty: 1,
    selectedVariant: 'blue',
    reviewsVisible: false,
  });

  // useSignal is a primitive reactive value — also serialized.
  const cartCount = useSignal(0);

  return (
    <div>
      {/* This component's markup is static after SSR.
          It will not re-render unless state.qty changes. */}
      <QuantityInput
        value={state.qty}
        onIncrement$={() => state.qty++}
        onDecrement$={() => state.qty > 1 && state.qty--}
      />

      {/* Conditional rendering is also serialized.
          If reviewsVisible is false at SSR time, this branch
          is not rendered and its JS is never downloaded. */}
      {state.reviewsVisible && <ReviewPanel />}

      <button onClick$={() => (state.reviewsVisible = true)}>
        Show Reviews
      </button>
    </div>
  );
});
```

---

## Comparison: Resumability vs Adjacent Strategies

| Dimension | Resumability (Qwik) | Partial Hydration | Streaming SSR | Islands Architecture |
|---|---|---|---|---|
| **Core idea** | Serialize execution context; resume on interaction | Hydrate only interactive subtrees | Stream HTML progressively; hydrate as chunks arrive | Isolate interactive components; hydrate each independently |
| **Framework startup cost** | Near zero — one global listener | Reduced — only interactive subtrees | Full (React/Angular) but deferred | Per-island cost; independent |
| **JS downloaded at startup** | ~1KB event registrar | Framework + interactive component bundles | Full framework bundle | Per-island bundles, loaded independently |
| **When component code executes** | Only when interaction targets it | On hydration of that subtree | When its HTML chunk is hydrated | When the island is scheduled for hydration |
| **State management** | Fully serialized into HTML | Developer manages; often prop-drilling | Standard; managed in memory | Isolated per island; cross-island is hard |
| **Re-execution of server work** | None | Partial (only interactive parts) | Full (deferred, but still full) | Full per island |
| **TTI characteristics** | Near-instant for any page size | Scales with interactive surface area | Faster FCP; TTI still depends on JS | Good; scales well for content-heavy pages |
| **Serialization constraints** | Functions must be `$`-wrapped; closures have limits | None | None | None |
| **Primary framework** | Qwik | Custom / React with RSC | Next.js App Router, React 18 | Astro |
| **Maturity** | Emerging (production-ready but small ecosystem) | Mature technique | Mature (React 18+) | Mature (Astro 2+) |
| **Cross-component communication** | Natural (reactive store serialized) | Standard within framework | Standard | Complex; islands are isolated |

### Where Each Strategy Wins

```
Page interactivity level:
Low ──────────────────────────────────── High
│                                           │
Islands Architecture          Resumability
Partial Hydration             Streaming SSR (full SPA)
(content-heavy pages)         (complex interactive apps)
```

- **Static content with occasional interaction**: Islands or partial hydration
- **Complex apps on slow devices where startup cost is the bottleneck**: Resumability
- **Large apps where FCP matters but full interactivity is needed**: Streaming SSR
- **Micro-frontends or multi-framework pages**: Islands

---

## Trade-offs of Resumability

### The Genuine Advantages

1. **TTI is O(1) relative to page size.** The startup cost does not grow as you add more components, because no component code runs until it's needed.

2. **Prefetching becomes low-risk.** Since chunks are tiny and interaction-specific, Qwik can prefetch likely-needed chunks during idle time without wasting bandwidth.

3. **True progressive enhancement.** A page is interactive at the granularity of individual event handlers, not entire component trees.

### The Real Costs

**Serialization constraints are non-trivial.** Not everything can be serialized into JSON. Functions captured in closures must be extracted via `$` — this fails if the closure captures non-serializable values.

```typescript
// This will cause a Qwik serialization error:
const handler = $(() => {
  // WebSocket instances cannot be serialized
  const ws = new WebSocket('ws://...');
  ws.send('hello'); // Error: cannot serialize ws
});

// Fix: move non-serializable resources outside the $ boundary
// or use Qwik's resource management APIs
```

**Mental model shift is steep.** Developers coming from React or Angular must internalize that:
- Every `$` function is a separate async chunk
- Closures behave differently across serialization boundaries
- Re-renders are not always top-down; they are fine-grained reactive

**Tooling and ecosystem maturity.** As of 2025, Qwik's community, component libraries, and debugging tools are substantially smaller than React or Angular. The Vite-based build pipeline with the optimizer is fast but adds a build step with novel failure modes.

**Testing complexity.** Standard React Testing Library patterns do not apply. Testing Qwik components requires Qwik-specific utilities and an understanding of when chunks load.

**Serialization overhead in HTML.** For data-heavy pages, embedding full state in `qwik/json` can inflate HTML size. A page with 200 products and their reactive state will have a larger HTML payload than an equivalent React SSR page.

---

## Influence on the Broader Ecosystem

Resumability has not stayed confined to Qwik. Its ideas are reshaping how React, Angular, and the ecosystem think about the server/client split.

### React Server Components: Partial Resumability

React Server Components (RSC) are a direct response to the "re-execution of server work" problem, though they solve it differently from Qwik's full resumability.

```tsx
// app/ProductPage.tsx — Server Component (no "use client")
// This component renders on the server and NEVER ships to the client.
// Its output is serialized as the "React Server Component Payload" (RSC payload),
// a JSON-like wire format distinct from HTML.

import { getProduct } from '@/lib/db';
import { AddToCart } from './AddToCart'; // Client component

export default async function ProductPage({ id }: { id: string }) {
  // Direct async data access — impossible in a traditional React component
  const product = await getProduct(id);

  return (
    <div>
      {/* These components ship zero JS — they only exist as HTML */}
      <h1>{product.name}</h1>
      <ProductImages images={product.images} />
      <ProductDescription text={product.description} />

      {/* This is the only client component — only its code ships */}
      <AddToCart productId={product.id} />
    </div>
  );
}
```

```tsx
// AddToCart.tsx
'use client'; // This directive marks the server/client boundary

import { useState } from 'react';

// Only this file is included in the JS bundle.
// ProductPage, ProductImages, and ProductDescription ship no JS.
export function AddToCart({ productId }: { productId: string }) {
  const [loading, setLoading] = useState(false);

  return (
    <button
      onClick={async () => {
        setLoading(true);
        await addToCartAction(productId);
        setLoading(false);
      }}
      disabled={loading}
    >
      Add to Cart
    </button>
  );
}
```

The key difference from Qwik: RSC eliminates re-execution of server components, but client components (`'use client'`) still undergo full hydration. It is "partial resumability at the component-type boundary" rather than at the event-handler boundary.

### Angular Non-Destructive Hydration

Angular v16+ introduced non-destructive hydration, and v17 made it the default. Before this, Angular's hydration destroyed server-rendered DOM and rebuilt it from scratch — the most expensive possible approach.

```typescript
// main.ts — Angular 17+ with non-destructive hydration
import { bootstrapApplication } from '@angular/platform-browser';
import { provideClientHydration } from '@angular/platform-browser';
import { AppComponent } from './app.component';

bootstrapApplication(AppComponent, {
  providers: [
    // This tells Angular: reuse the server-rendered DOM nodes
    // rather than destroying and recreating them
    provideClientHydration(),
  ],
});
```

```typescript
// app.component.ts
@Component({
  selector: 'app-root',
  template: `
    <header>
      <!-- Angular will "claim" these existing DOM nodes
           rather than creating new ones -->
      <nav>
        <a routerLink="/">Home</a>
        <a routerLink="/products">Products</a>
      </nav>
    </header>
    <router-outlet />
  `,
})
export class AppComponent {}
```

Angular is also developing incremental hydration (experimental in v19), which defers hydration of components based on triggers like viewport visibility or user interaction — directionally similar to Qwik's lazy execution but still re-executing component code when triggered.

```typescript
// Angular incremental hydration (v19 experimental)
@Component({
  selector: 'app-reviews',
  template: `...`,
})
export class ReviewsComponent {}

// In template:
// @defer (on viewport) {
//   <app-reviews />
// } @placeholder {
//   <div>Loading reviews...</div>
// }
```

### Where the Ecosystem Is Heading

The trajectory is clear: every major framework is moving toward **reducing the amount of JavaScript that must execute before the user can interact**.

```
Past (2015–2020):         Present (2021–2025):       Direction (2025+):
──────────────────        ────────────────────        ──────────────────
Full client hydration  →  Partial hydration        →  Fine-grained lazy
                          RSC wire format             execution
All JS upfront         →  Code splitting by route  →  Code splitting by
                                                       interaction target
Framework owns DOM     →  Framework claims DOM     →  Framework defers to
(destructive hydrate)     (non-destructive)           browser natively
```

Signals (now in Angular, Solid, and via React's experimental work) are also part of this: fine-grained reactivity means the framework can compute exactly which DOM nodes need updating without traversing the full component tree, reducing re-render cost alongside startup cost.

---

## When Resumability Matters

### High-Impact Scenarios

**Low-end devices and emerging markets.** A mid-range Android phone in 2024 parses and executes JavaScript 5–10x slower than a MacBook Pro. If your target audience includes users on budget devices or constrained networks, TTI from full hydration may be 8–12 seconds. Resumability collapses this to under a second.

**TTI-critical landing pages.** Conversion rate is directly correlated with TTI. E-commerce pages, sign-up flows, and marketing pages where every second of latency costs revenue are strong candidates.

**Large content pages with sparse interactivity.** A news article, product detail page, or documentation page may have 200 server-rendered components but only 3–5 interactive elements. Full hydration penalizes you for the 195 static components. Resumability charges only for what the user actually touches.

**Pages where FID/INP are failing Core Web Vitals.** If your First Input Delay or Interaction to Next Paint metrics are poor, it is often because the main thread is blocked executing hydration code. Resumability eliminates this entirely.

### Where Resumability Is Not the Right Tool

```typescript
// Complex real-time dashboard: WebSocket state, frequent updates,
// every component is interactive — resumability's lazy loading
// becomes an overhead, not a benefit. Use SSR + full hydration.

// Simple static site: No interactivity at all.
// Plain SSG or static files are simpler and equally fast.

// Highly stateful app (Figma-like, Google Docs-like):
// Serializing and deserializing complex object graphs is costly.
// The app will be downloading JS constantly anyway.
```

---

## Interview Q&A

### Q1: What is the fundamental difference between resumability and hydration, and why does it matter?

**Model answer:**

Hydration re-executes framework initialization code on the client after the server has already rendered the HTML. The browser must download the entire framework and component tree, re-traverse every node, and attach event listeners before the page becomes interactive. This work scales linearly with component count.

Resumability serializes the execution context — component state, reactive subscriptions, and event listener locations — directly into the HTML. The client never re-runs component code. It reads the serialized state and registers a single global event listener. When an interaction fires, only the specific handler chunk is downloaded and executed.

The practical impact is that TTI becomes nearly constant regardless of page complexity. A 500-component Qwik page has the same startup cost as a 5-component Qwik page, because neither downloads or executes component code at startup. In contrast, a 500-component React SSR page has 100x the startup cost of a 5-component page.

This matters most on slow networks and low-end devices, and for TTI-critical surfaces like e-commerce checkout flows and landing pages.

---

### Q2: Explain the role of the `$` suffix in Qwik. What does it do at build time vs runtime?

**Model answer:**

The `$` suffix is an optimizer boundary marker. At build time, Qwik's Vite plugin (the optimizer) treats any function wrapped with `$` as a lazy chunk boundary. It extracts the function into its own separate JS file and replaces it in-place with a `QRL` (Qwik Resource Locator) — a serializable reference that encodes the chunk URL and the exported symbol name.

At runtime, a QRL is just a string like `"./q-chunk-abc.js#MyComponent_onClick_1_click"`. This string can be embedded in HTML attributes (`on:click="./q-chunk-abc.js#..."`) and serialized into `qwik/json`. When the browser needs to call that function, it fetches the URL, imports the module, and calls the symbol.

Without `$`, the optimizer would bundle everything together, eliminating lazy loading. The `$` forces the developer to be explicit about extraction boundaries, which is why Qwik has a somewhat steeper learning curve than React — you're making architectural decisions that were previously implicit.

A key constraint: functions marked with `$` must only capture serializable values in their closures. Capturing a DOM reference, a class instance with methods, or a WebSocket connection will fail, because those cannot be embedded in JSON. This is the main serialization constraint developers encounter.

---

### Q3: React Server Components and Qwik resumability both aim to reduce client-side JS. How do they differ structurally?

**Model answer:**

They solve the problem at different granularities.

React Server Components draw a **static boundary at component type**: components marked without `'use client'` are server-only and ship zero JS. Components with `'use client'` ship their full code and undergo standard hydration. This is a coarse-grained split — you choose at author time which components are server-only. Once a client component is hydrated, it behaves identically to traditional React hydration: the full component code runs, and re-renders are top-down from the nearest state owner.

Qwik resumability draws a **dynamic boundary at event handler**: all components are lazy by default. Their render code does not run on the client unless reactive state change forces a re-render of that specific component. Even then, only that component's code is fetched, not its entire subtree. Event handlers are individually lazy-loaded at interaction time.

The structural consequence: in an RSC app, if you have 10 client components on a page, all 10 hydrate at startup. In a Qwik app with 10 interactive components, zero component code runs at startup, and each handler loads only when triggered.

RSC is a substantial improvement over traditional SSR, but it is not resumability. Qwik's model is more aggressive but requires more developer awareness of serialization constraints and the optimizer's behavior.

---

### Q4: A senior engineer argues that resumability is overengineered for most use cases and that streaming SSR with code splitting achieves similar TTI. How do you respond?

**Model answer:**

This is a fair challenge and the answer depends on what "similar TTI" means in practice.

Streaming SSR with code splitting improves FCP (the user sees content sooner) and reduces initial bundle size (only the current route's code loads). But it does not change the fundamental cost: once a chunk is downloaded, the framework must still traverse all hydrated components, attach event listeners, and initialize reactive state. That work is deferred but not eliminated.

For a page with 20 interactive components, streaming SSR still eventually runs hydration code for all 20. On a fast desktop, this may be imperceptible. On a mid-range Android phone processing 400KB of JavaScript at 2–3MB/s, you can measure it clearly in TTI traces.

The more precise framing is: streaming SSR optimizes **when** hydration work starts (after critical content is visible), while resumability eliminates the hydration work itself. For content-heavy pages with sparse interactivity — which describes the majority of e-commerce and media pages — resumability is not overengineered; it is correctly scoped. The 195 static components pay zero cost at runtime in Qwik, whereas they pay hydration cost in any React or Angular app regardless of streaming.

Where the senior engineer is right: for genuinely complex interactive apps (dashboards, editors, real-time collaboration tools), almost every component will be interactive, so the lazy-loading benefit diminishes and the serialization constraints become a real cost. For those apps, streaming SSR with code splitting is the pragmatic choice. The decision is use-case dependent, not categorical.

---

## Key Takeaways

1. Hydration re-executes server work on the client; resumability serializes server execution context so the client can continue without re-executing it.
2. Qwik serializes event listeners as URL references in HTML attributes and embeds full component state in `qwik/json`. The client starts with a single global listener.
3. The `$` suffix is an explicit optimizer boundary — it extracts a function into a separately downloadable chunk and replaces it with a serializable QRL.
4. TTI in a resumable app is nearly independent of component count; in a hydration-based app it scales linearly.
5. Serialization constraints are the primary developer cost: closures must only capture serializable values.
6. React Server Components achieve partial resumability at the component-type boundary. Angular non-destructive hydration and incremental hydration move the ecosystem in the same direction.
7. Resumability is highest-value for TTI-critical, content-heavy pages on low-end devices. For fully interactive apps, its benefits diminish and its costs grow.
8. The ecosystem direction is unambiguous: fine-grained lazy execution and reduced startup cost are now first-class concerns across all major frameworks.
