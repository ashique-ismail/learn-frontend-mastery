# Shared Element Transition (Hero Animation)

## The Idea

**In plain English:** You have a grid of product cards. The user clicks one card and navigates to the product detail page. Instead of the card disappearing and the detail page appearing from nowhere, the card's image smoothly grows and moves to become the hero image on the detail page. The element appears to travel from its position in the grid to its new, larger position on the detail page — as if it is the same physical object, just repositioned.

**Real-world analogy:** Think of a street performer doing a magic trick.

- The performer holds up a small coin in their left hand (the card thumbnail).
- They close their fist and move their hand slowly across their body.
- They open their right fist to reveal a large coin (the detail hero image) — appearing to be the *same* coin, just transformed.
- The audience's eye follows the motion. They never lose track of the object.
- If the performer just dropped one coin and picked up another — that is a **hard navigation** without a shared element transition.
- The smooth cross-body move — that is the hero animation. The user's eye is guided. They always know where they are.

This technique is also called a "hero transition" (named after Android's shared element transitions) or a "magic move."

---

## Learning Objectives

- Understand the View Transitions API `view-transition-name` property
- Implement shared element transitions using the View Transitions API with React Router
- Use Framer Motion `layoutId` for React-based shared element transitions
- Build the FLIP technique as a manual fallback for browsers without View Transition support
- Understand Angular's approach using a combination of route animations and element cloning
- Handle the accessibility implications of animated navigation

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Animate an element in the same page | ✅ with `transition` or `@keyframes` | — |
| Animate an element across two different route/page states | ❌ | Old DOM and new DOM don't coexist; CSS can't measure positions across route changes |
| Capture the old element's bounding box before navigation | ❌ | Requires JS `getBoundingClientRect()` before the DOM changes |
| Move an element from grid position to hero position | ❌ | Two different DOM nodes in two different page states |
| Animate the *size* change (small thumbnail → large hero) | ❌ | Size change requires knowing both start and end sizes |
| Match element across route boundaries | ❌ | CSS has no concept of element identity across page changes |

**Conclusion:** The View Transitions API solves this at the browser level using `view-transition-name`. Framer Motion solves it at the React level using `layoutId`. The FLIP technique solves it manually with JS geometry calculations.

---

## HTML & CSS Foundation

### View Transitions API Approach

```css
/* On the LIST page: assign a unique transition name per card */
.product-card[data-id="42"] img {
  view-transition-name: product-image-42;
}

/* On the DETAIL page: assign the same name to the hero image */
.product-hero[data-id="42"] {
  view-transition-name: product-image-42;
}

/* The browser automatically creates ::view-transition-group(product-image-42)
   and animates it from the card position to the hero position */

/* Customize the animation timing for the shared element */
::view-transition-group(product-image-42) {
  animation-duration: 400ms;
  animation-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
}

/* The rest of the page cross-fades normally */
::view-transition-old(root) {
  animation: 200ms ease-out fade-and-scale-out;
}
::view-transition-new(root) {
  animation: 300ms ease-in fade-and-scale-in;
}

@keyframes fade-and-scale-out {
  to { opacity: 0; }
}
@keyframes fade-and-scale-in {
  from { opacity: 0; }
}

@media (prefers-reduced-motion: reduce) {
  ::view-transition-group(*),
  ::view-transition-old(root),
  ::view-transition-new(root) {
    animation: none;
  }
}
```

### Dynamic `view-transition-name` via Inline Style

```html
<!-- List page: each card gets a unique name via inline style -->
<div class="product-grid">
  <a
    class="product-card"
    href="/products/42"
    style="--card-id: 42"
  >
    <img
      src="/images/product-42-thumb.jpg"
      alt="Product name"
      style="view-transition-name: product-image-42;"
    />
    <p>Product Name</p>
  </a>
</div>

<!-- Detail page: hero gets matching name -->
<article class="product-detail">
  <img
    class="product-hero"
    src="/images/product-42-full.jpg"
    alt="Product name"
    style="view-transition-name: product-image-42;"
  />
</article>
```

---

## React Implementation

### View Transitions API with React Router

```tsx
// ProductCard.tsx
import { useNavigate } from 'react-router-dom';
import { startTransition } from 'react';

interface Product {
  id: number;
  name: string;
  thumbnail: string;
  price: number;
}

interface ProductCardProps {
  product: Product;
}

export function ProductCard({ product }: ProductCardProps) {
  const navigate = useNavigate();

  const handleClick = (e: React.MouseEvent) => {
    e.preventDefault();

    const navigateFn = () => {
      startTransition(() => navigate(`/products/${product.id}`));
    };

    if (document.startViewTransition) {
      document.startViewTransition(navigateFn);
    } else {
      navigateFn();
    }
  };

  return (
    <a
      href={`/products/${product.id}`}
      className="product-card"
      onClick={handleClick}
    >
      <img
        src={product.thumbnail}
        alt={product.name}
        // Dynamic view-transition-name: must be unique per card on screen
        style={{ viewTransitionName: `product-image-${product.id}` }}
        className="product-card__image"
      />
      <div className="product-card__info">
        <h2>{product.name}</h2>
        <p>${product.price}</p>
      </div>
    </a>
  );
}
```

```tsx
// ProductDetail.tsx
import { useParams } from 'react-router-dom';
import { useProduct } from '../hooks/useProduct';

export function ProductDetail() {
  const { id } = useParams<{ id: string }>();
  const product = useProduct(Number(id));

  if (!product) return <div>Loading...</div>;

  return (
    <article className="product-detail">
      <img
        src={product.fullImage}
        alt={product.name}
        // Same naming convention as the card — browser matches them
        style={{ viewTransitionName: `product-image-${product.id}` }}
        className="product-detail__hero"
      />
      <section className="product-detail__content">
        <h1>{product.name}</h1>
        <p>${product.price}</p>
        <p>{product.description}</p>
      </section>
    </article>
  );
}
```

### Framer Motion `layoutId` Approach

```tsx
// MotionProductCard.tsx
import { motion } from 'framer-motion';
import { useNavigate } from 'react-router-dom';

export function MotionProductCard({ product }: ProductCardProps) {
  const navigate = useNavigate();

  return (
    <motion.a
      href={`/products/${product.id}`}
      className="product-card"
      onClick={(e) => {
        e.preventDefault();
        navigate(`/products/${product.id}`);
      }}
      layout
    >
      {/* layoutId links this image to the hero image on the detail page */}
      <motion.img
        layoutId={`product-image-${product.id}`}
        src={product.thumbnail}
        alt={product.name}
        className="product-card__image"
        // Animate the image's border-radius as it expands to hero
        style={{ borderRadius: 8 }}
        transition={{ type: 'spring', stiffness: 300, damping: 30 }}
      />
      <motion.div layout="position" className="product-card__info">
        <h2>{product.name}</h2>
        <p>${product.price}</p>
      </motion.div>
    </motion.a>
  );
}
```

```tsx
// MotionProductDetail.tsx
import { motion } from 'framer-motion';
import { useParams } from 'react-router-dom';
import { useProduct } from '../hooks/useProduct';

export function MotionProductDetail() {
  const { id } = useParams<{ id: string }>();
  const product = useProduct(Number(id));

  if (!product) return null;

  return (
    <motion.article
      className="product-detail"
      initial={{ opacity: 0 }}
      animate={{ opacity: 1 }}
      exit={{ opacity: 0 }}
    >
      {/* layoutId must match exactly what was used in the card */}
      <motion.img
        layoutId={`product-image-${product.id}`}
        src={product.fullImage}
        alt={product.name}
        className="product-detail__hero"
        style={{ borderRadius: 0 }}
        transition={{ type: 'spring', stiffness: 300, damping: 30 }}
      />
      <motion.section
        className="product-detail__content"
        initial={{ y: 20, opacity: 0 }}
        animate={{ y: 0, opacity: 1 }}
        transition={{ delay: 0.15 }}
      >
        <h1>{product.name}</h1>
        <p>${product.price}</p>
        <p>{product.description}</p>
      </motion.section>
    </motion.article>
  );
}
```

### FLIP Fallback (Manual, No Library)

```tsx
// useFLIP.ts — First, Last, Invert, Play
import { useRef, useCallback } from 'react';

export function useFLIP<T extends HTMLElement>() {
  const elementRef = useRef<T>(null);
  const firstRect = useRef<DOMRect | null>(null);

  // Call BEFORE the DOM change — captures "First" position
  const capture = useCallback(() => {
    if (elementRef.current) {
      firstRect.current = elementRef.current.getBoundingClientRect();
    }
  }, []);

  // Call AFTER the DOM change — calculates delta and plays animation
  const play = useCallback((duration = 400) => {
    const el = elementRef.current;
    const first = firstRect.current;
    if (!el || !first) return;

    // "Last" position
    const last = el.getBoundingClientRect();

    // "Invert" — apply a transform to make element appear at "First" position
    const deltaX = first.left - last.left;
    const deltaY = first.top  - last.top;
    const scaleX = first.width  / last.width;
    const scaleY = first.height / last.height;

    // Check prefers-reduced-motion
    if (window.matchMedia('(prefers-reduced-motion: reduce)').matches) return;

    el.style.transform = `translate(${deltaX}px, ${deltaY}px) scale(${scaleX}, ${scaleY})`;
    el.style.transformOrigin = 'top left';
    el.style.transition = 'none';

    // Force reflow so browser registers the inverted state
    el.getBoundingClientRect();

    // "Play" — animate to natural (Last) position
    el.style.transition = `transform ${duration}ms cubic-bezier(0.4, 0, 0.2, 1)`;
    el.style.transform = '';

    el.addEventListener('transitionend', () => {
      el.style.transition = '';
      el.style.transformOrigin = '';
    }, { once: true });
  }, []);

  return { elementRef, capture, play };
}
```

---

## Angular Implementation

### View Transitions API with Angular Router

```typescript
// product-list.component.ts
import { Component, inject } from '@angular/core';
import { Router } from '@angular/router';
import { NgFor } from '@angular/common';
import { ProductService } from './product.service';

@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [NgFor],
  template: `
    <div class="product-grid">
      <a
        *ngFor="let product of products(); trackBy: trackById"
        class="product-card"
        [href]="'/products/' + product.id"
        (click)="navigateWithTransition($event, product.id)"
      >
        <img
          [src]="product.thumbnail"
          [alt]="product.name"
          [style.view-transition-name]="'product-image-' + product.id"
          class="product-card__image"
        />
        <h2>{{ product.name }}</h2>
      </a>
    </div>
  `,
})
export class ProductListComponent {
  private router = inject(Router);
  products = inject(ProductService).products;

  navigateWithTransition(event: MouseEvent, id: number): void {
    event.preventDefault();

    const doNav = () => this.router.navigate(['/products', id]);

    if ('startViewTransition' in document) {
      (document as Document & { startViewTransition: (cb: () => void) => void })
        .startViewTransition(doNav);
    } else {
      doNav();
    }
  }

  trackById(_: number, product: { id: number }) {
    return product.id;
  }
}
```

```typescript
// product-detail.component.ts
import { Component, inject, computed } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { toSignal } from '@angular/core/rxjs-interop';
import { map } from 'rxjs';
import { ProductService } from './product.service';

@Component({
  selector: 'app-product-detail',
  standalone: true,
  template: `
    <article *ngIf="product() as p" class="product-detail">
      <img
        [src]="p.fullImage"
        [alt]="p.name"
        [style.view-transition-name]="'product-image-' + p.id"
        class="product-detail__hero"
      />
      <section class="product-detail__content">
        <h1>{{ p.name }}</h1>
        <p>{{ p.description }}</p>
      </section>
    </article>
  `,
})
export class ProductDetailComponent {
  private route = inject(ActivatedRoute);
  private productService = inject(ProductService);

  private id = toSignal(
    this.route.params.pipe(map(p => Number(p['id'])))
  );

  product = computed(() => {
    const id = this.id();
    return id ? this.productService.getById(id) : null;
  });
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| `prefers-reduced-motion` disables hero animation | CSS `::view-transition-group(*)` sets `animation: none`; Framer `useReducedMotion()` returns early |
| Focus moves to new page heading on transition end | `document.startViewTransition().finished.then(() => h1.focus())` |
| `view-transition-name` values must be unique per page | Enforce with product ID suffix; two elements with the same name on one page will cause the transition to skip |
| Alt text unchanged between thumbnail and hero | Same product name in both `alt` attributes — screen reader user doesn't need to re-read it |
| Navigation announced to screen reader | `<title>` update, `aria-live` region, or router's built-in scroll/focus behavior |
| Back navigation reverses animation | Angular/React Router both support back detection via `popstate`; reverse the slide direction |
| Transition does not block interaction | View Transitions API is async; avoid `pointer-events: none` on the full page during transition |

---

## Production Pitfalls

**1. Two elements with the same `view-transition-name` on one page crashes the transition**
If a product appears in both a grid and a "recently viewed" sidebar, both would get the same name. The browser silently skips the transition. Fix: ensure `view-transition-name` is unique per visible page — scope by section if needed (`product-grid-image-42`, `sidebar-image-42`).

**2. Framer `layoutId` must exist in the DOM at both ends of the transition**
If the list page unmounts before the detail page mounts (common with `AnimatePresence mode="wait"`), there is no source element to animate from. Fix: use `mode="sync"` or `mode="popLayout"` for shared element transitions, not `mode="wait"`.

**3. `object-fit: cover` on images causes visual distortion during FLIP**
During scale animation, an image with `object-fit: cover` will show different crop regions as it scales. Fix: use `object-position` to anchor the crop, or animate `clip-path` alongside the transform.

**4. `view-transition-name` on an element inside a `transform` context**
If the card is inside a CSS Grid or a `transform: translateZ(0)` stacking context, the captured position may be off. Fix: apply `contain: layout` carefully; avoid `transform` on ancestors of elements with `view-transition-name`.

**5. Angular's `NgIf` destroys the element before the animation frame**
`*ngIf="false"` immediately removes the DOM node, so FLIP capture sees a null element. Fix: use `@if` with Angular animations deferring the removal, or clone the element manually before removing it.

---

## Interview Angle

**Q: "How would you build a shared element transition between a product grid and a detail page?"**

Strong answer covers four points:
1. **Browser-native first** — `view-transition-name` on both elements with the same value. The browser captures screenshots, animates between them. Zero JS animation code for the element itself.
2. **Framer Motion `layoutId` as the React cross-browser answer** — same `layoutId` string on both the card image and the hero image. Framer handles measurement and animation automatically.
3. **FLIP for manual control** — capture `getBoundingClientRect()` before the DOM change, compute delta, invert with `transform`, then play back to identity. Works everywhere, needs no library.
4. **Critical constraints** — unique names per page (View Transitions API), `AnimatePresence mode` selection (Framer), reduced-motion guard, and focus management after transition.

**Follow-up: "What is the FLIP technique and why does forcing a reflow matter?"**
FLIP stands for First, Last, Invert, Play. After applying the inverted transform, you must read a layout property (like `getBoundingClientRect()`) to force the browser to flush the style before starting the animation. Without the reflow flush, the browser batches the style change and the animation start together — meaning there is no visual start state to animate from, and the element snaps directly to its final position.
