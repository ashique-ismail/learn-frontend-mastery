# View Transitions API

## Overview

The View Transitions API provides a native browser mechanism for animating between different states of a page — or between entirely different pages — without the complexity of maintaining two DOM trees simultaneously, synchronizing animation libraries, or writing layout thrashing animation code. The browser captures the old state as a screenshot, applies the new DOM state, then animates between them using CSS.

```
Before View Transitions:
  Navigate/update → Instant visual jump
  Custom animation → Show old DOM + new DOM simultaneously
                   → Synchronize unmount/mount with animation timing
                   → Prevent layout thrash
                   → Clean up after animation completes
  Cost: hundreds of lines of state management + animation code

With View Transitions:
  document.startViewTransition(() => updateDOM())
  → Browser captures old state
  → Your callback runs (updates DOM)
  → Browser animates old → new
  Browser handles the cross-fade, the timing, the cleanup
```

The API has two modes: **same-document** transitions (for SPAs, AJAX navigation) and **cross-document** transitions (for multi-page apps navigating between real URLs, available in Chrome 126+).

## Same-Document View Transitions

### Basic Usage

```typescript
// Simplest case: wrap any DOM mutation in startViewTransition
function navigateTo(newContent: string): void {
  // Without View Transitions: instant swap
  document.getElementById('content')!.innerHTML = newContent;

  // With View Transitions: animated swap
  document.startViewTransition(() => {
    document.getElementById('content')!.innerHTML = newContent;
  });
}
```

`startViewTransition()` accepts a callback (sync or async) that performs DOM mutations. The browser:
1. Captures the current page state as a screenshot (the "old" state)
2. Calls your callback — DOM updates happen here
3. Captures the new page state
4. Animates between old and new using CSS

### The ViewTransition Object

```typescript
const transition = document.startViewTransition(async () => {
  // Async mutations are supported
  const data = await fetchPageData(url);
  renderPage(data);
});

// Promise that resolves when the animation starts
await transition.ready;

// Promise that resolves when the animation finishes
await transition.finished;

// Promise that resolves when the new state is ready (before animation)
await transition.updateCallbackDone;

// Skip the animation and jump to final state
transition.skipTransition();
```

### TypeScript Feature Detection

```typescript
function withViewTransition(updateFn: () => void | Promise<void>): void {
  if (!document.startViewTransition) {
    // Fallback: run the update without animation
    void updateFn();
    return;
  }
  document.startViewTransition(updateFn);
}

// Usage in a router
async function navigate(url: string): Promise<void> {
  withViewTransition(async () => {
    const html = await fetch(url).then(r => r.text());
    const doc = new DOMParser().parseFromString(html, 'text/html');
    document.title = doc.title;
    document.getElementById('main')!.replaceWith(doc.getElementById('main')!);
  });
}
```

## CSS `::view-transition-*` Pseudo-Elements

When a transition starts, the browser creates a tree of pseudo-elements:

```
::view-transition                     ← root overlay (covers viewport)
  ::view-transition-group(root)       ← group for each named element
    ::view-transition-image-pair(root)
      ::view-transition-old(root)     ← screenshot of old state
      ::view-transition-new(root)     ← live capture of new state
  ::view-transition-group(hero-image)
    ::view-transition-image-pair(hero-image)
      ::view-transition-old(hero-image)
      ::view-transition-new(hero-image)
```

### Default Animation

By default, the browser cross-fades between old and new states:

```css
/* Default behavior (built-in browser stylesheet) */
::view-transition-old(root),
::view-transition-new(root) {
  animation-duration: 0.25s;
}

::view-transition-old(root) {
  animation-name: -ua-view-transition-fade-out;
}

::view-transition-new(root) {
  animation-name: -ua-view-transition-fade-in;
}
```

### Custom Transition Animations

```css
/* Custom: slide in from right */
@keyframes slide-in-right {
  from { transform: translateX(100%); }
  to   { transform: translateX(0); }
}

@keyframes slide-out-left {
  from { transform: translateX(0); }
  to   { transform: translateX(-100%); }
}

::view-transition-old(root) {
  animation: 300ms ease-in slide-out-left;
}

::view-transition-new(root) {
  animation: 300ms ease-out slide-in-right;
}

/* Fade only the new content in (old disappears immediately) */
::view-transition-old(root) {
  display: none; /* or animation: none */
}

::view-transition-new(root) {
  animation: 400ms ease-out fade-in;
}

/* Respect user preference for reduced motion */
@media (prefers-reduced-motion: reduce) {
  ::view-transition-old(root),
  ::view-transition-new(root) {
    animation-duration: 0.01ms;
  }
}
```

## Named View Transitions — Shared Element Transitions

The most powerful feature: assign a name to an element and the browser will animate that specific element's position and size from old state to new state — like a "hero" animation.

```css
/* Mark an element for independent transition tracking */
.product-card.featured {
  /* The view-transition-name creates a named capture group */
  view-transition-name: featured-product;
  /* Each named element must be unique on the page */
}

/* Custom animation for just this named element */
::view-transition-group(featured-product) {
  animation-duration: 400ms;
  animation-timing-function: cubic-bezier(0.25, 0.46, 0.45, 0.94);
}
```

```typescript
// E-commerce: animate product card expanding into detail view
function navigateToProduct(productId: string): void {
  const card = document.getElementById(`product-${productId}`);

  // Set the name BEFORE the transition
  card!.style.viewTransitionName = 'active-product';

  document.startViewTransition(() => {
    // Remove the name from the card in the new state
    // (it will be on the detail page hero instead)
    router.navigate(`/products/${productId}`);
  });
}
```

```css
/* Animate the hero image on product detail page */
.product-detail .hero-image {
  view-transition-name: active-product;
  contain: layout; /* Required when view-transition-name is set */
}

/* The browser automatically interpolates position/size between old and new */
/* No keyframe needed for the position animation — browser does this automatically */
/* Only customize timing/easing */
::view-transition-group(active-product) {
  animation-duration: 500ms;
  animation-timing-function: ease-in-out;
}
```

## Direction-Aware Transitions

Different animations depending on navigation direction:

```typescript
// Track navigation direction
let navigationDirection: 'forward' | 'backward' = 'forward';

history.pushState = new Proxy(history.pushState, {
  apply(target, thisArg, args) {
    navigationDirection = 'forward';
    return Reflect.apply(target, thisArg, args);
  }
});

window.addEventListener('popstate', () => {
  navigationDirection = 'backward';
});

// Apply direction class before transition
function navigate(url: string, direction: 'forward' | 'backward'): void {
  document.documentElement.dataset.transitioning = direction;

  document.startViewTransition(async () => {
    await router.navigate(url);
  });
}
```

```css
/* Slide direction based on navigation direction */
html[data-transitioning="forward"] ::view-transition-old(root) {
  animation-name: slide-out-left;
}

html[data-transitioning="forward"] ::view-transition-new(root) {
  animation-name: slide-in-right;
}

html[data-transitioning="backward"] ::view-transition-old(root) {
  animation-name: slide-out-right;
}

html[data-transitioning="backward"] ::view-transition-new(root) {
  animation-name: slide-in-left;
}
```

## Cross-Document View Transitions (MPA)

For multi-page apps (traditional navigation between real HTML pages), cross-document transitions require opt-in from BOTH the origin page and the destination page:

```css
/* Both pages must include this — enabling the feature */
@view-transition {
  navigation: auto; /* Enable cross-document transitions for same-origin navigations */
}

/* page-a.html */
.product-card {
  view-transition-name: product-hero;
}

/* page-b.html (product detail) */
.detail-hero {
  view-transition-name: product-hero;
  /* Browser automatically animates position/size of this name between pages */
}
```

```css
/* Customize cross-document transition animation */
/* Triggered on the destination page */
@keyframes morph-in {
  from {
    opacity: 0;
    scale: 0.8;
  }
}

::view-transition-new(product-hero) {
  animation: 400ms ease-out morph-in;
}
```

## React Integration

```typescript
// React Router v6 integration
import { useNavigate } from 'react-router-dom';
import { flushSync } from 'react-dom';

export function useViewTransitionNavigate() {
  const navigate = useNavigate();

  return (to: string, options?: { replace?: boolean }) => {
    if (!document.startViewTransition) {
      navigate(to, options);
      return;
    }

    document.startViewTransition(() => {
      // flushSync ensures React updates happen synchronously inside the transition
      flushSync(() => {
        navigate(to, options);
      });
    });
  };
}

// Usage
function ProductCard({ product }: { product: Product }) {
  const navigate = useViewTransitionNavigate();

  return (
    <div
      className="product-card"
      style={{ viewTransitionName: `product-${product.id}` } as React.CSSProperties}
      onClick={() => navigate(`/products/${product.id}`)}
    >
      <img src={product.image} alt={product.name} />
      <h3>{product.name}</h3>
    </div>
  );
}
```

```typescript
// React 19 introduces first-class View Transition support
// import { unstable_ViewTransition as ViewTransition } from 'react';

// In React Router v7 (future): built-in view transition support
// <Link to="/products/1" viewTransition>See product</Link>
```

## Angular Integration

```typescript
// Angular 17+ with Router view transitions
import { provideRouter, withViewTransitions } from '@angular/router';

// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withViewTransitions({
      onViewTransitionCreated: ({ transition }) => {
        // Access the ViewTransition object for custom control
        transition.ready.then(() => {
          console.log('Transition ready');
        });
      }
    })),
  ]
};
```

```css
/* Angular component — assign view transition name */
/* app-product-card */
:host {
  view-transition-name: var(--product-id); /* Use CSS custom property for uniqueness */
  display: block;
  contain: layout; /* Required for elements with view-transition-name */
}
```

```typescript
// Angular component
@Component({ template: `
  <div [style.view-transition-name]="'product-' + product.id">
    {{ product.name }}
  </div>
`})
export class ProductCardComponent {
  @Input() product!: Product;
}
```

## Performance Considerations

```
View Transitions internally:
1. Freeze rendering (compositor takes snapshot)
2. Run your callback (DOM mutations)
3. Capture new state
4. Create pseudo-element tree (gpu-accelerated layers)
5. Run CSS animations on GPU
6. Tear down pseudo-elements

Performance implications:
- Snapshot and composite operations are fast (GPU)
- Large pages with many named transitions = more GPU memory
- Transitions run off main thread — smooth even under JS load
- Should NOT use view-transition-name on more than ~10-15 elements simultaneously
- Each named element becomes a GPU layer — excessive use causes memory pressure

containment requirement:
- Elements with view-transition-name must NOT overflow their stacking context
- Use `contain: layout` or `contain: strict` on named elements
- `overflow: hidden` + `position: relative` is often sufficient
```

## Browser Support

```
Same-document View Transitions:
  Chrome 111+     — full support
  Edge 111+       — full support
  Safari 18+      — full support
  Firefox         — in development (behind flag as of mid-2025)

Cross-document View Transitions:
  Chrome 126+     — supported
  Edge 126+       — supported
  Safari 18.2+    — supported
  Firefox         — not yet

Detection and fallback:
  'startViewTransition' in document   → same-document support
  CSS.supports('@view-transition ...')→ cross-document support
```

## Common Mistakes

```typescript
// WRONG — mutations outside the callback aren't captured
document.getElementById('content')!.classList.add('new-state');
document.startViewTransition(() => {
  // Too late — old state already captured
});

// CORRECT — all mutations inside the callback
document.startViewTransition(() => {
  document.getElementById('content')!.classList.add('new-state');
});

// WRONG — duplicate view-transition-name (crashes the transition)
// Two elements with view-transition-name: hero at the same time
document.querySelectorAll('.product-card').forEach(el => {
  el.style.viewTransitionName = 'hero'; // Only ONE element can have each name
});

// CORRECT — unique names per element
document.querySelectorAll('.product-card').forEach((el, i) => {
  el.style.viewTransitionName = `product-${i}`;
});

// WRONG — not using flushSync with React
document.startViewTransition(() => {
  setCurrentPage('home'); // React batches this — DOM might not update synchronously
});

// CORRECT
document.startViewTransition(() => {
  flushSync(() => setCurrentPage('home'));
});
```

## Interview Questions

1. **Explain how the View Transitions API works internally — what does the browser do when `startViewTransition()` is called?**
   The browser freezes the current rendered state and captures it as a static screenshot (the old state), promoted to a GPU layer. It then calls the provided callback, allowing DOM mutations to run synchronously or asynchronously. Once the callback promise resolves, the browser captures the new DOM state. It constructs a pseudo-element tree (`::view-transition-old` for the screenshot, `::view-transition-new` for the live new content) and runs CSS animations on these layers — typically off the main thread. When animations complete, the pseudo-element tree is torn down and normal rendering resumes.

2. **What is a named view transition and when should you use one?**
   A named view transition, set via `view-transition-name: <identifier>` CSS property, tells the browser to track a specific element independently from the page-level transition. The browser captures the named element's position, size, and appearance in both old and new states, then smoothly interpolates between them — creating a "shared element" or "hero" animation. Use them for: navigating from a list item to its detail page (card expands), image galleries (thumbnail zooms to full size), tab content transitions. Limit to ~10–15 named elements simultaneously to avoid GPU memory pressure, and ensure each name is unique on the page.

3. **How do same-document and cross-document view transitions differ?**
   Same-document transitions are triggered imperatively via `document.startViewTransition()` and work for any DOM mutation within a single page — SPA navigation, AJAX content swaps, component state changes. Cross-document transitions happen automatically during browser navigation between real URLs (MPA navigation). They require opt-in from both origin and destination pages via `@view-transition { navigation: auto }` in CSS. Cross-document transitions cannot be triggered programmatically — they fire on any same-origin navigation. Same-document transitions have wider browser support (Chrome 111+) versus cross-document (Chrome 126+).

4. **Why must React's `flushSync` be used inside `startViewTransition` callbacks?**
   `startViewTransition` expects DOM mutations to complete synchronously (or via the returned Promise). React 18's concurrent mode batches state updates and applies them asynchronously. If you call `setState` inside a transition callback without `flushSync`, React schedules the update for a future render, but the browser's callback has already returned — the browser captures the new state BEFORE React has applied the DOM changes. `flushSync` forces React to immediately process all pending state updates and commit them to the DOM synchronously within the callback.

5. **How would you implement forward/backward slide animations for SPA navigation using View Transitions?**
   Track navigation direction by intercepting `history.pushState` (forward) and listening to `popstate` events (backward). Before calling `startViewTransition`, set a data attribute on `<html>` indicating direction (e.g., `data-transitioning="forward"`). In CSS, define `::view-transition-old(root)` and `::view-transition-new(root)` animation overrides scoped to each direction attribute. The old element slides out in the appropriate direction, the new element slides in from the opposite side. Clear the data attribute after `transition.finished` resolves to avoid interfering with future transitions.

6. **What does `contain: layout` do and why is it required for named view transition elements?**
   `contain: layout` establishes a layout containment boundary — it tells the browser that layout changes inside this element don't affect elements outside it, and vice versa. The View Transitions API requires this (or an equivalent containing block) for named transition elements to properly calculate the element's position and size for the captured snapshot. Without it, the browser might capture incorrect geometry, especially for elements whose visual position depends on ancestors. In practice, `position: relative` or `overflow: hidden` on a parent often provides sufficient containment.

7. **What are the browser support gaps for View Transitions and how would you handle them?**
   Same-document: Firefox lacks support as of mid-2025 (in development). Cross-document: requires Chrome/Edge 126+ or Safari 18.2+. Detection: `'startViewTransition' in document` for same-document; `CSS.supports("@view-transition { navigation: auto }")` for cross-document. Fallback: `startViewTransition` is not called — run the DOM mutation directly without animation. The feature is inherently a progressive enhancement — the UI works without it, transitions are optional polish. Never put critical state changes only in the transition callback.

8. **Describe how you would implement a list-to-detail shared element transition in a React SPA.**
   In the list view, give each item a unique `view-transition-name` based on its ID (e.g., `product-42`). In the detail view, assign the same `view-transition-name` to the hero element. Use a custom `useViewTransitionNavigate` hook that wraps `useNavigate` in `document.startViewTransition` with `flushSync`. When navigating, the browser interpolates position and size of the element between list thumbnail and detail hero. Handle the cleanup: if the user navigates back, ensure the list view re-mounts with the same name on the correct element. Use CSS `::view-transition-group(product-42)` to customize the animation timing for that specific element.
