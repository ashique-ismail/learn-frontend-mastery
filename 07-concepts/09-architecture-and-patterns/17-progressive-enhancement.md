# Progressive Enhancement

## Overview

Progressive Enhancement (PE) is a layered approach to web development: start with a solid semantic HTML foundation that works everywhere, then layer CSS for presentation, then JavaScript for behavior. Each layer enhances the experience for capable environments while remaining functional without it.

It is the philosophical opposite of **graceful degradation** (build for the best browser, then add fallbacks). PE builds for the worst case first and enhances upward.

---

## The Three Layers

```
┌─────────────────────────────────────────┐
│          JavaScript (behavior)          │  ← fails gracefully if absent
├─────────────────────────────────────────┤
│           CSS (presentation)            │  ← degrades to unstyled HTML
├─────────────────────────────────────────┤
│     Semantic HTML (structure/content)   │  ← works in any environment
└─────────────────────────────────────────┘
```

**Layer 1 — Semantic HTML must work alone:**
- A form must submit via native `<form action="/search" method="GET">`
- Navigation must work as `<a href="/products">` links
- Content must be readable without CSS
- Screen readers must get full meaning from the markup

**Layer 2 — CSS enhances visual presentation:**
- Layout, typography, color, animation
- Component is usable even if CSS fails to load (CDN outage, corporate proxy)

**Layer 3 — JavaScript enhances behavior:**
- Client-side validation on top of server-side validation
- SPA navigation on top of regular `<a href>` links
- Optimistic UI on top of functional form submission

---

## Why It Still Matters

Despite modern browser ubiquity, PE addresses real failure modes:

- **Flaky networks** — JS bundles fail to load on 2G or intermittent connections
- **Corporate proxies** — Some strip JavaScript or block CDN domains
- **Screen readers and bots** — Crawlers and assistive tech work better with real HTML
- **Performance budget exceeded** — When JS hasn't loaded yet, the page should still function
- **JavaScript errors** — A runtime exception shouldn't brick the entire page
- **SEO** — Server-rendered HTML is more reliably indexed than JS-rendered content
- **Smart TVs, older devices** — Limited JS engine support

---

## Practical Implementation

### Forms

```html
<!-- Layer 1: works without JS — posts to server -->
<form action="/search" method="GET">
  <input type="search" name="q" placeholder="Search..." />
  <button type="submit">Search</button>
</form>
```

```javascript
// Layer 3: enhance with client-side behavior
document.querySelector('form[action="/search"]')
  ?.addEventListener('submit', async (e) => {
    e.preventDefault(); // only intercept if JS is running
    const q = new FormData(e.target).get('q');
    const results = await fetchSearch(q);
    renderResults(results); // SPA behavior
  });
```

If JS fails: form submits to `/search?q=...`, server returns a full HTML page. The user loses the instant SPA behavior but gets working search.

### Navigation

```html
<!-- Layer 1: real links — work without JS, bookmarkable, accessible -->
<nav>
  <a href="/products">Products</a>
  <a href="/about">About</a>
</nav>
```

```javascript
// Layer 3: intercept for client-side navigation
document.querySelectorAll('nav a').forEach(link => {
  link.addEventListener('click', (e) => {
    e.preventDefault();
    router.navigate(link.href); // SPA navigation
  });
});
```

### Feature Detection (Not Browser Detection)

```javascript
// ❌ Browser detection — brittle, breaks on new browsers
if (navigator.userAgent.includes('Chrome')) {
  useSomeFeature();
}

// ✓ Feature detection — test for the capability
if ('IntersectionObserver' in window) {
  // Enhanced: lazy loading via IntersectionObserver
  setupLazyLoading();
} else {
  // Baseline: load all images immediately
}

if (CSS.supports('container-type', 'inline-size')) {
  // Use container queries
} else {
  // Fall back to media queries
}

if ('share' in navigator) {
  showNativeShareButton();
} else {
  showFallbackShareLinks();
}
```

### CSS `@supports`

```css
/* Baseline layout: works everywhere */
.grid {
  display: flex;
  flex-wrap: wrap;
}

/* Enhancement: container queries for capable browsers */
@supports (container-type: inline-size) {
  .grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  }
}

/* Color enhancement */
.button {
  background-color: #3b82f6; /* fallback */
}

@supports (color: oklch(0 0 0)) {
  .button {
    background-color: oklch(60% 0.15 250); /* wider gamut */
  }
}
```

---

## PE in Modern SPAs

"My app requires JavaScript" is a common objection. PE still applies:

### Server-Side Rendering as Layer 1

```tsx
// Next.js App Router — page renders to HTML on server
// Works for crawlers, initial load before JS hydrates
export default async function ProductPage({ params }) {
  const product = await fetchProduct(params.id);
  return (
    <article>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <form action="/api/cart" method="POST">
        <input type="hidden" name="productId" value={product.id} />
        <button type="submit">Add to Cart</button>
        {/* Works without JS via Server Action */}
      </form>
    </article>
  );
}
```

### React Server Actions as PE-Friendly Enhancement

```tsx
// Server Action — the form works without client JS
async function addToCart(formData: FormData) {
  'use server';
  const productId = formData.get('productId');
  await db.cart.add(productId);
  revalidatePath('/cart');
}

// Client enhancement: add optimistic UI on top
export function AddToCartButton({ productId }: { productId: string }) {
  const [optimistic, setOptimistic] = useOptimistic(false);

  return (
    <form action={addToCart}>
      <input type="hidden" name="productId" value={productId} />
      <button
        type="submit"
        onClick={() => setOptimistic(true)}
        disabled={optimistic}
      >
        {optimistic ? 'Adding...' : 'Add to Cart'}
      </button>
    </form>
  );
}
```

### Angular Universal + Deferred Enhancement

```typescript
// Server renders full HTML — works before hydration
@Component({
  selector: 'app-search',
  template: `
    <!-- Works as a real form server-side -->
    <form method="GET" action="/search" (ngSubmit)="onSubmit()">
      <input type="search" name="q" [ngModel]="query" />
      <button type="submit">Search</button>
    </form>

    <!-- Only loaded after hydration (client JS) -->
    @defer (on idle) {
      <app-suggestions [query]="query" />
    }
  `
})
export class SearchComponent {
  query = '';
  onSubmit() { /* SPA navigation enhancement */ }
}
```

---

## PE vs Graceful Degradation

| | Progressive Enhancement | Graceful Degradation |
|---|---|---|
| **Start from** | Minimum viable (HTML) | Maximum desired (full app) |
| **Add layers** | Upward — CSS, then JS | Downward — add fallbacks |
| **Philosophy** | Core content first | Best experience first, degrade |
| **Result** | Reliable core across all environments | May break badly in poor conditions |
| **Best for** | Public-facing content, e-commerce, forms | Internal tools, known-browser environments |

---

## When NOT to Use PE

PE is a spectrum, not a binary rule. Full PE is sometimes impractical:

- **Real-time collaborative apps** (Google Docs) — meaningful use requires JS; a bare HTML fallback adds no value
- **Data visualization dashboards** — charts without JS are empty containers
- **Maps and interactive experiences** — no sensible HTML fallback exists

In these cases, focus on: SSR for initial HTML/SEO, skeleton loading states, error boundaries, and offline handling — rather than a no-JS fallback.

---

## Interview Questions

**Q: What's the difference between progressive enhancement and graceful degradation?**  
A: PE starts with the simplest baseline (semantic HTML) and adds layers. Graceful degradation starts with the full experience and adds fallbacks for lesser environments. PE tends to produce more resilient results because every capability is explicitly added — you don't accidentally forget a fallback.

**Q: "My app requires JavaScript — why does progressive enhancement matter?"**  
A: Even JS-dependent apps benefit from PE principles: SSR ensures the initial HTML is meaningful before hydration, real `<a href>` links are crawlable and bookmarkable, forms that fall back to server submission survive JS errors, and feature detection prevents silent failures in partially-supported environments.

**Q: How do you detect feature support in CSS?**  
A: Use `@supports` for property/value detection. `CSS.supports('property', 'value')` is the JS equivalent. Target the capability, not the browser — `@supports (display: grid)` rather than checking for Chrome.

**Q: How does progressive enhancement relate to accessibility?**  
A: They reinforce each other. Semantic HTML (PE's foundation) is what assistive technologies read. When you enhance a native `<button>` with JS rather than making a `<div>` interactive, screen readers get correct role/state for free. PE's "start with HTML" rule and accessibility's "use the right element" rule converge on the same markup.
