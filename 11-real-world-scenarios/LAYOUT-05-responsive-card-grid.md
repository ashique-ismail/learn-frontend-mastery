# Responsive Card Grid (auto-fill / auto-fit)

## The Idea

**In plain English:** A card grid is a collection of equal-height cards arranged in rows and columns — product cards, blog posts, team members, etc. The grid should automatically reflow: on a wide screen you get 4 columns, on a tablet 2, on mobile 1 — without a single media query, just CSS Grid's intrinsic sizing. The trick is choosing between `auto-fill` and `auto-fit`, and understanding exactly what `minmax()` means in each case.

**Real-world analogy:** Think of a hardware store peg-board wall.

- The **peg-board** is your grid container. It has a fixed total width.
- The **hooks** are the grid columns — you can fit as many as the board allows given the hook size.
- **`auto-fill`**: The board pre-installs every possible hook position, even if some are empty. You can see the ghost outlines of empty slots.
- **`auto-fit`**: The board only installs hooks where items actually hang. Empty positions collapse to zero width — items stretch to fill the available space.
- The **`minmax(240px, 1fr)`** rule: each hook is at least 240px wide (min), and they all stretch equally to fill the board (1fr max).

The practical difference: `auto-fill` leaves empty space at the end of an incomplete last row; `auto-fit` stretches the last row's cards to fill the width — which can make a single card stretch to full width, which often looks broken.

---

## Learning Objectives

- Understand `auto-fill` vs `auto-fit` with concrete visual outcomes
- Use `minmax(min, max)` to set a floor on column width
- Control implicit grid row heights so cards stay equal height
- Build card skeleton screens that match the loaded card layout exactly
- Use `@container` queries for card-level responsive styling
- Avoid the common `auto-fit` trap (single card stretching to 100%)

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Responsive columns without breakpoints | ✅ `auto-fill` / `auto-fit` with `minmax` | — |
| Equal card heights in a row | ✅ CSS Grid implicit rows | — |
| Skeleton screen animation | ✅ CSS `@keyframes` shimmer | — |
| Detect how many columns are rendered | ❌ | CSS cannot read computed column count |
| Lazy-load images as cards enter viewport | ❌ | Requires IntersectionObserver |
| Fetch and inject card data | ❌ | JS / framework required |
| Detect card container width for card-level responsive | ✅ Container queries | Needs explicit `container-type` |

---

## HTML & CSS Foundation

```html
<section class="card-grid-section" aria-label="Product listings">

  <!-- Grid container -->
  <ul class="card-grid" role="list" aria-busy="false" aria-live="polite">

    <!-- Loaded card -->
    <li class="card" style="--card-color: #3b82f6">
      <article>
        <div class="card__image-wrap">
          <img
            src="/products/widget.webp"
            alt="Blue Widget"
            class="card__img"
            width="400"
            height="250"
            loading="lazy"
          />
        </div>
        <div class="card__body">
          <h3 class="card__title">Blue Widget</h3>
          <p class="card__desc">A very fine widget in blue.</p>
          <div class="card__footer">
            <span class="card__price">$29.99</span>
            <button class="card__btn">Add to cart</button>
          </div>
        </div>
      </article>
    </li>

    <!-- Skeleton card (while loading) -->
    <li class="card card--skeleton" aria-hidden="true">
      <article>
        <div class="card__image-wrap skeleton-box"></div>
        <div class="card__body">
          <div class="skeleton-line skeleton-line--title"></div>
          <div class="skeleton-line skeleton-line--body"></div>
          <div class="skeleton-line skeleton-line--body skeleton-line--short"></div>
        </div>
      </article>
    </li>

  </ul>

</section>
```

```css
/* ─── Grid container ─── */
.card-grid {
  display: grid;
  /*
    auto-fill: creates as many columns as possible at minimum 260px wide.
    Empty slots at end of row are visible (don't collapse).
    Use this when you want consistent column count across rows.
  */
  grid-template-columns: repeat(auto-fill, minmax(260px, 1fr));
  gap: 1.5rem;
  list-style: none;
  margin: 0;
  padding: 0;
  /* Implicit rows all have the same height via subgrid or auto */
  grid-auto-rows: 1fr;
}

/*
  auto-fit alternative — use this if you want cards to stretch on partial rows:
  grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));

  Danger: a single card on the last row will stretch to full width.
  Guard with: max-width on the card itself.
*/

/* ─── Card container query setup ─── */
.card {
  container-type: inline-size;
  container-name: card;
  /* Ensures card stretches to fill implicit row height */
  display: flex;
  flex-direction: column;
}

.card article {
  display: flex;
  flex-direction: column;
  height: 100%;
  background: #fff;
  border: 1px solid #e5e7eb;
  border-radius: 12px;
  overflow: hidden;
  transition: box-shadow 0.2s, transform 0.2s;
}

.card article:hover {
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.1);
  transform: translateY(-2px);
}

/* ─── Card image ─── */
.card__image-wrap {
  /* Aspect ratio box: always 16:9 regardless of image size */
  aspect-ratio: 16 / 9;
  overflow: hidden;
  flex-shrink: 0;
}

.card__img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
  transition: transform 0.3s;
}

.card article:hover .card__img {
  transform: scale(1.04);
}

/* ─── Card body ─── */
.card__body {
  display: flex;
  flex-direction: column;
  flex: 1;          /* push footer to bottom */
  padding: 1rem;
  gap: 0.5rem;
}

.card__title {
  font-size: 1rem;
  font-weight: 600;
  margin: 0;
  line-height: 1.3;
}

.card__desc {
  font-size: 0.875rem;
  color: #6b7280;
  margin: 0;
  flex: 1;          /* stretch so footer is always at bottom */
}

.card__footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-top: auto;
  padding-top: 0.75rem;
  border-top: 1px solid #f3f4f6;
}

.card__price {
  font-weight: 700;
  font-size: 1.0625rem;
  color: #111827;
}

.card__btn {
  padding: 0.375rem 0.875rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 0.8125rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
}

.card__btn:hover  { background: #1d4ed8; }
.card__btn:focus-visible {
  outline: 2px solid #2563eb;
  outline-offset: 2px;
}

/* ─── Container query: narrow card gets stacked layout ─── */
@container card (max-width: 300px) {
  .card__footer {
    flex-direction: column;
    align-items: flex-start;
    gap: 0.5rem;
  }
  .card__btn { width: 100%; text-align: center; }
}

/* ─── Skeleton shimmer ─── */
@keyframes shimmer {
  0%   { background-position: -400px 0; }
  100% { background-position:  400px 0; }
}

.skeleton-box,
.skeleton-line {
  background: linear-gradient(
    90deg,
    #f3f4f6 25%,
    #e5e7eb 37%,
    #f3f4f6 63%
  );
  background-size: 800px 100%;
  animation: shimmer 1.4s infinite linear;
  border-radius: 4px;
}

.skeleton-box {
  aspect-ratio: 16 / 9;
  border-radius: 0;
}

.skeleton-line {
  height: 14px;
  margin-bottom: 8px;
}

.skeleton-line--title  { height: 18px; width: 70%; }
.skeleton-line--body   { width: 100%; }
.skeleton-line--short  { width: 50%; }

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .skeleton-box,
  .skeleton-line { animation: none; background: #e5e7eb; }
  .card article  { transition: none; }
  .card__img     { transition: none; }
}
```

---

## React Implementation

```tsx
// CardGrid.tsx
import { useState, useEffect } from 'react';

interface Product {
  id: string;
  title: string;
  description: string;
  price: number;
  image: string;
  imageAlt: string;
}

interface CardGridProps {
  products: Product[];
  isLoading?: boolean;
  skeletonCount?: number;
  onAddToCart: (id: string) => void;
}

export function CardGrid({
  products,
  isLoading = false,
  skeletonCount = 8,
  onAddToCart,
}: CardGridProps) {
  return (
    <section className="card-grid-section" aria-label="Product listings">
      <ul
        className="card-grid"
        role="list"
        aria-busy={isLoading}
        aria-live="polite"
        aria-label="Products"
      >
        {isLoading
          ? Array.from({ length: skeletonCount }, (_, i) => (
              <SkeletonCard key={`skeleton-${i}`} />
            ))
          : products.map(product => (
              <ProductCard
                key={product.id}
                product={product}
                onAddToCart={onAddToCart}
              />
            ))}
      </ul>
    </section>
  );
}

/* ── Loaded card ── */
function ProductCard({
  product,
  onAddToCart,
}: {
  product: Product;
  onAddToCart: (id: string) => void;
}) {
  return (
    <li className="card">
      <article>
        <div className="card__image-wrap">
          <img
            src={product.image}
            alt={product.imageAlt}
            className="card__img"
            width={400}
            height={250}
            loading="lazy"
          />
        </div>
        <div className="card__body">
          <h3 className="card__title">{product.title}</h3>
          <p className="card__desc">{product.description}</p>
          <div className="card__footer">
            <span className="card__price">
              ${product.price.toFixed(2)}
            </span>
            <button
              className="card__btn"
              onClick={() => onAddToCart(product.id)}
              aria-label={`Add ${product.title} to cart`}
            >
              Add to cart
            </button>
          </div>
        </div>
      </article>
    </li>
  );
}

/* ── Skeleton card ── */
function SkeletonCard() {
  return (
    <li className="card card--skeleton" aria-hidden="true">
      <article>
        <div className="card__image-wrap skeleton-box" />
        <div className="card__body">
          <div className="skeleton-line skeleton-line--title" />
          <div className="skeleton-line skeleton-line--body" />
          <div className="skeleton-line skeleton-line--body skeleton-line--short" />
        </div>
      </article>
    </li>
  );
}
```

---

## Angular Implementation

```typescript
// product.model.ts
export interface Product {
  id: string;
  title: string;
  description: string;
  price: number;
  image: string;
  imageAlt: string;
}
```

```typescript
// card-grid.component.ts
import {
  Component, Input, Output, EventEmitter, ChangeDetectionStrategy
} from '@angular/core';
import { NgFor, NgIf, DecimalPipe } from '@angular/common';
import { Product } from './product.model';

@Component({
  selector: 'app-card-grid',
  standalone: true,
  imports: [NgFor, NgIf, DecimalPipe],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <section class="card-grid-section" aria-label="Product listings">
      <ul
        class="card-grid"
        role="list"
        [attr.aria-busy]="isLoading"
        aria-live="polite"
        aria-label="Products"
      >
        <!-- Skeleton state -->
        <li
          *ngIf="isLoading"
          *ngFor="let s of skeletons; trackBy: trackByIndex"
          class="card card--skeleton"
          aria-hidden="true"
        >
          <article>
            <div class="card__image-wrap skeleton-box"></div>
            <div class="card__body">
              <div class="skeleton-line skeleton-line--title"></div>
              <div class="skeleton-line skeleton-line--body"></div>
              <div class="skeleton-line skeleton-line--body skeleton-line--short"></div>
            </div>
          </article>
        </li>

        <!-- Loaded cards -->
        <li
          *ngIf="!isLoading"
          *ngFor="let product of products; trackBy: trackById"
          class="card"
        >
          <article>
            <div class="card__image-wrap">
              <img
                [src]="product.image"
                [alt]="product.imageAlt"
                class="card__img"
                width="400"
                height="250"
                loading="lazy"
              />
            </div>
            <div class="card__body">
              <h3 class="card__title">{{ product.title }}</h3>
              <p class="card__desc">{{ product.description }}</p>
              <div class="card__footer">
                <span class="card__price">${{ product.price | number:'1.2-2' }}</span>
                <button
                  class="card__btn"
                  [attr.aria-label]="'Add ' + product.title + ' to cart'"
                  (click)="addToCart.emit(product.id)"
                >Add to cart</button>
              </div>
            </div>
          </article>
        </li>
      </ul>
    </section>
  `,
})
export class CardGridComponent {
  @Input() products: Product[] = [];
  @Input() isLoading = false;
  @Input() skeletonCount = 8;
  @Output() addToCart = new EventEmitter<string>();

  get skeletons() {
    return Array.from({ length: this.skeletonCount });
  }

  trackById(_: number, product: Product) { return product.id; }
  trackByIndex(i: number) { return i; }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| `<ul role="list">` for card collection | Semantic list, list role explicit for CSS reset contexts |
| `aria-busy="true"` during loading | Screen reader announces "busy" while skeletons show |
| `aria-live="polite"` on grid | Announces when cards appear after loading |
| Skeleton cards have `aria-hidden="true"` | Animated placeholders are not read aloud |
| Card image has meaningful `alt` text | Describes the product, not "product image" |
| `Add to cart` button has `aria-label` with product name | Distinguishes buttons when multiple cards are present |
| Card interactive element order follows visual order | Title → description → price → button |
| Images have explicit `width` and `height` | Prevents layout shift (CLS) before load |
| Card hover effect does not trigger on keyboard focus alone | `article:hover` not `article:focus` (separate focus style) |

---

## Production Pitfalls

**1. `auto-fit` makes the last lonely card stretch to full width**
If you have 7 items in a 3-column grid, the last item fills the entire row on its own. This looks broken for product cards. Fix: use `auto-fill` instead of `auto-fit`, or add `max-width` to the card itself.

**2. `minmax(260px, 1fr)` with padding causes unintended overflow**
If the grid container has padding, the available width is reduced. A card grid of `minmax(260px, 1fr)` in a 300px container creates a single column — fine. But if the container itself has `padding: 1rem`, the math is: `300px - 32px = 268px`, which barely fits one 260px column, but on very narrow containers it still triggers a scrollbar. Fix: use `min(260px, 100%)` as the `minmax` minimum to prevent any single column from ever exceeding 100% of the container.

**3. Equal-height cards fail when `grid-auto-rows` is not set**
By default, implicit grid rows size to the tallest item in the row — cards in the same row are equal height, but rows with different content heights have different row heights. This is usually fine, but if you add a card without content in a mixed row, it can be unexpectedly short. Fix: use `grid-auto-rows: 1fr` to equalize all implicit rows.

**4. Skeleton count doesn't match loaded card count**
If you show 8 skeletons but only 5 products load, the layout jumps. Fix: if you know the total count from the API (via `X-Total-Count` header or a `total` field), use that count for skeletons, not an arbitrary number.

**5. Layout shift when images load**
Without explicit `width` and `height` on `<img>`, the browser doesn't know the image's aspect ratio before it loads, causing the card to resize and push everything below it down. Fix: always set `width` and `height` attributes, and use `aspect-ratio: 16/9` on the image wrapper as a fallback.
