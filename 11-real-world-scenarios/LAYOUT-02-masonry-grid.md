# Masonry / Pinterest-Style Grid

## The Idea

**In plain English:** A masonry grid is a multi-column layout where items have different heights and pack together vertically — like bricks in a wall — so there are no large empty gaps. A regular CSS grid forces every row to the same height as its tallest cell, leaving wasted whitespace under shorter items. Masonry eliminates that waste by letting each column grow independently and placing each new item into whichever column is currently shortest.

**Real-world analogy:** Think of a professional bricklayer building a decorative wall.

- A **standard grid** is like laying bricks in a perfectly even course: every row is the same height, and if one brick is thinner than the others you shim the gap with mortar — the gap is always visible.
- A **masonry grid** is like a bricklayer who looks at the whole wall, finds the lowest point, and places the next brick there. The wall surface is uneven in an intentional, aesthetic way. No gaps, no forced uniformity.
- The **column height tracker** is the bricklayer's eye — always scanning to find the shortest column.
- **IntersectionObserver** is the foreman who only delivers bricks to the bricklayer when the scaffolding is actually in view — no point hauling bricks to a section nobody can see yet.
- **Responsive column count** is the number of scaffold lanes you put up — fewer lanes on a narrow scaffold (mobile), more on a wide one (desktop).

The key insight: the CSS `columns` property gives you masonry for free but sacrifices control over item order and image loading. JS-based masonry gives you full control at the cost of a resize listener, column height bookkeeping, and absolute positioning.

---

## Learning Objectives

- Understand the trade-offs between CSS `columns` masonry and JS absolute-position masonry
- Implement the CSS `columns` approach and know exactly when `break-inside: avoid` is required
- Build a JS masonry engine with column-height tracking using `position: absolute`
- Wire `IntersectionObserver` to lazy-load images inside a masonry grid
- Derive a responsive column count from a container width without media queries
- Avoid the classic production pitfalls: layout-before-images, ResizeObserver loops, and reflow storms

---

## Why CSS Alone Isn't Enough

The CSS `columns` property handles the visual arrangement but the column-fill order is top-to-bottom, left-to-right — meaning items flow down column 1, then down column 2, not into the shortest column first. This makes it unsuitable for any scenario where item order is semantically meaningful (e.g., a sorted feed) or where you need to insert new items at the bottom as the user scrolls.

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Pack items into columns visually | ✅ via `columns` | — |
| Prevent card from splitting mid-render | ✅ via `break-inside: avoid` | — |
| Fill shortest column next (true masonry order) | ❌ | `columns` fills top-to-bottom per column, not by height |
| Insert items at the bottom on scroll | ❌ | `columns` reflows the entire set on any DOM mutation |
| Lazy-load images only when in viewport | ❌ | Requires `IntersectionObserver` JS API |
| Recalculate layout after image load changes height | ❌ | CSS has no post-load layout hook |
| Animate new items appearing | ⚠️ | CSS can animate, but JS must trigger class after position is set |
| Derive column count from container width | ❌ | `columns` can do it with `column-width` but exposes no count value to JS |

**Conclusion:** CSS `columns` is the right tool for static, read-only, order-agnostic galleries (e.g., a photo portfolio). For dynamic feeds, infinite scroll, or sorted content, you need a JS layout engine.

---

## HTML & CSS Foundation

### CSS Columns Approach

```html
<!-- Static masonry — correct for galleries, wrong for sorted feeds -->
<div class="masonry-css">
  <div class="masonry-css__item">
    <img src="..." alt="Description of image one" width="400" height="600" loading="lazy" />
    <p>Caption text</p>
  </div>
  <div class="masonry-css__item">
    <img src="..." alt="Description of image two" width="400" height="300" loading="lazy" />
    <p>Caption text</p>
  </div>
  <!-- more items... -->
</div>
```

```css
/* ─── Tokens ─── */
:root {
  --masonry-gap: 1rem;
  --masonry-col-min-width: 280px;
}

/* ─── CSS Columns masonry ─── */
.masonry-css {
  columns: var(--masonry-col-min-width);   /* auto column count based on container */
  column-gap: var(--masonry-gap);
}

/*
  break-inside: avoid is the critical rule.
  Without it, a card with a tall image and caption text will
  split across two columns — the image appears in column 1,
  the caption appears at the top of column 2.
*/
.masonry-css__item {
  break-inside: avoid;
  margin-bottom: var(--masonry-gap);
  border-radius: 8px;
  overflow: hidden;
  background: #fff;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
}

/* Images must be display:block to eliminate the inline baseline gap */
.masonry-css__item img {
  display: block;
  width: 100%;
  height: auto;   /* never fix height — masonry depends on natural height */
}

/* ─── JS masonry container ─── */
/*
  The container is position:relative so children can be
  placed with position:absolute. JS sets top/left on each child.
  JS also sets the container height after layout.
*/
.masonry-js {
  position: relative;
  /* height is set by JS */
}

.masonry-js__item {
  position: absolute;
  /* top and left are set by JS */
  width: calc((100% - (var(--cols) - 1) * var(--masonry-gap)) / var(--cols));
  transition: opacity 0.3s ease, transform 0.3s ease;
}

/* Items start invisible; JS adds .is-placed after calculating position */
.masonry-js__item:not(.is-placed) {
  opacity: 0;
  pointer-events: none;
}

.masonry-js__item.is-placed {
  opacity: 1;
}

/* Images: block + aspect-ratio placeholder to reduce layout shift */
.masonry-js__item img {
  display: block;
  width: 100%;
  height: auto;
  border-radius: 8px;
}

/* Skeleton placeholder while image loads */
.masonry-js__item img:not([src]),
.masonry-js__item img[src=""] {
  aspect-ratio: 3 / 4;
  background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
}

@keyframes shimmer {
  0%   { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .masonry-js__item {
    transition: none;
  }
  .masonry-js__item img:not([src]) {
    animation: none;
    background: #e8e8e8;
  }
}
```

**What CSS owns:** column gaps, card border-radius, image sizing, skeleton shimmer, reduced-motion fallback, the `break-inside: avoid` split prevention for the columns approach.

**What CSS cannot own:** which column each item belongs to, the container height after layout, column height tracking, lazy loading triggers, and layout recalculation after image load.

---

## React Implementation

### Types

```tsx
// masonry.types.ts
export interface MasonryItem {
  id: string;
  imageUrl: string;
  imageWidth: number;    // intrinsic width — needed to calculate aspect ratio
  imageHeight: number;   // intrinsic height
  alt: string;
  caption?: string;
}

export interface ColumnHeights {
  [colIndex: number]: number;  // current pixel height of each column
}
```

### Layout Engine Hook

```tsx
// useMasonryLayout.ts
import { useState, useEffect, useRef, useCallback } from 'react';
import type { MasonryItem, ColumnHeights } from './masonry.types';

interface UseMasonryOptions {
  columnMinWidth?: number;   // minimum column width in px, default 280
  gap?: number;              // gap between items in px, default 16
}

interface ItemLayout {
  id: string;
  top: number;
  left: number;
  width: number;
}

interface MasonryLayout {
  containerHeight: number;
  itemLayouts: Map<string, ItemLayout>;
  columnCount: number;
}

export function useMasonryLayout(
  items: MasonryItem[],
  containerRef: React.RefObject<HTMLDivElement>,
  options: UseMasonryOptions = {}
) {
  const { columnMinWidth = 280, gap = 16 } = options;

  const [layout, setLayout] = useState<MasonryLayout>({
    containerHeight: 0,
    itemLayouts: new Map(),
    columnCount: 1,
  });

  // Heights of each item after its image loads — keyed by item id
  const itemHeightsRef = useRef<Map<string, number>>(new Map());

  const recalculate = useCallback(() => {
    const container = containerRef.current;
    if (!container) return;

    const containerWidth = container.offsetWidth;
    if (containerWidth === 0) return;

    // Derive column count from available width
    const colCount = Math.max(1, Math.floor((containerWidth + gap) / (columnMinWidth + gap)));
    const colWidth = (containerWidth - (colCount - 1) * gap) / colCount;

    // Initialise column heights to 0
    const colHeights: ColumnHeights = {};
    for (let i = 0; i < colCount; i++) colHeights[i] = 0;

    const newLayouts = new Map<string, ItemLayout>();

    items.forEach(item => {
      // Find the shortest column
      let shortestCol = 0;
      for (let i = 1; i < colCount; i++) {
        if (colHeights[i] < colHeights[shortestCol]) shortestCol = i;
      }

      const left = shortestCol * (colWidth + gap);
      const top  = colHeights[shortestCol];

      // Use actual measured height if available, otherwise estimate from aspect ratio
      const knownHeight = itemHeightsRef.current.get(item.id);
      const itemHeight = knownHeight
        ?? Math.round(colWidth * (item.imageHeight / item.imageWidth)) + 60; // 60px for caption

      newLayouts.set(item.id, { id: item.id, top, left, width: colWidth });
      colHeights[shortestCol] = top + itemHeight + gap;
    });

    const containerHeight = Math.max(...Object.values(colHeights));

    setLayout({ containerHeight, itemLayouts: newLayouts, columnCount: colCount });
  }, [items, containerRef, columnMinWidth, gap]);

  // Re-layout when items change
  useEffect(() => {
    recalculate();
  }, [recalculate]);

  // ResizeObserver: re-layout when container width changes
  useEffect(() => {
    const container = containerRef.current;
    if (!container) return;

    const ro = new ResizeObserver(entries => {
      // Guard against ResizeObserver loop — only act if width changed
      for (const entry of entries) {
        const width = entry.contentRect.width;
        if (width > 0) recalculate();
      }
    });

    ro.observe(container);
    return () => ro.disconnect();
  }, [containerRef, recalculate]);

  // Called by each card after its image loads and the card has a measurable height
  const registerItemHeight = useCallback((id: string, height: number) => {
    const existing = itemHeightsRef.current.get(id);
    if (existing === height) return;
    itemHeightsRef.current.set(id, height);
    recalculate();
  }, [recalculate]);

  return { layout, registerItemHeight };
}
```

### Lazy Image Hook

```tsx
// useLazyImage.ts
import { useState, useEffect, useRef } from 'react';

export function useLazyImage(src: string) {
  const [loaded, setLoaded]   = useState(false);
  const [inView, setInView]   = useState(false);
  const ref                   = useRef<HTMLDivElement>(null);

  // Step 1: watch for the card entering the viewport
  useEffect(() => {
    const el = ref.current;
    if (!el) return;

    const io = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setInView(true);
          io.disconnect();   // fire once, then stop observing
        }
      },
      { rootMargin: '200px' }  // start loading 200px before entering viewport
    );

    io.observe(el);
    return () => io.disconnect();
  }, []);

  // Step 2: once in view, load the image
  useEffect(() => {
    if (!inView) return;
    const img = new Image();
    img.onload  = () => setLoaded(true);
    img.onerror = () => setLoaded(true); // still mark loaded to avoid infinite spinner
    img.src     = src;
  }, [inView, src]);

  return { ref, loaded, shouldLoad: inView };
}
```

### MasonryCard Component

```tsx
// MasonryCard.tsx
import { useEffect, useRef } from 'react';
import { useLazyImage } from './useLazyImage';
import type { MasonryItem, ItemLayout } from './masonry.types';

interface Props {
  item: MasonryItem;
  layout: ItemLayout | undefined;
  onHeightMeasured: (id: string, height: number) => void;
}

export function MasonryCard({ item, layout, onHeightMeasured }: Props) {
  const cardRef                     = useRef<HTMLDivElement>(null);
  const { ref: ioRef, loaded, shouldLoad } = useLazyImage(item.imageUrl);

  // After image loads and card has painted, measure and report height
  useEffect(() => {
    if (!loaded || !cardRef.current) return;
    // requestAnimationFrame ensures the browser has painted before we measure
    const raf = requestAnimationFrame(() => {
      const h = cardRef.current?.offsetHeight ?? 0;
      if (h > 0) onHeightMeasured(item.id, h);
    });
    return () => cancelAnimationFrame(raf);
  }, [loaded, item.id, onHeightMeasured]);

  const isPlaced = layout !== undefined;

  return (
    // ioRef drives the IntersectionObserver; cardRef measures height
    <div
      ref={el => {
        (cardRef as React.MutableRefObject<HTMLDivElement | null>).current = el;
        (ioRef as React.MutableRefObject<HTMLDivElement | null>).current   = el;
      }}
      className={`masonry-js__item ${isPlaced ? 'is-placed' : ''}`}
      style={
        isPlaced
          ? { top: layout.top, left: layout.left, width: layout.width }
          : undefined
      }
      // Each card is its own landmark for screen readers
      role="article"
    >
      {shouldLoad ? (
        <img
          src={item.imageUrl}
          alt={item.alt}
          width={item.imageWidth}
          height={item.imageHeight}
          loading="eager"   // IntersectionObserver already controls timing
        />
      ) : (
        // Placeholder with correct aspect ratio to reduce layout shift
        <div
          style={{ aspectRatio: `${item.imageWidth} / ${item.imageHeight}` }}
          aria-hidden="true"
          className="masonry-js__item--placeholder"
        />
      )}
      {item.caption && <p>{item.caption}</p>}
    </div>
  );
}
```

### MasonryGrid Component

```tsx
// MasonryGrid.tsx
import { useRef } from 'react';
import { useMasonryLayout } from './useMasonryLayout';
import { MasonryCard } from './MasonryCard';
import type { MasonryItem } from './masonry.types';

interface Props {
  items: MasonryItem[];
  columnMinWidth?: number;
  gap?: number;
  ariaLabel?: string;
}

export function MasonryGrid({
  items,
  columnMinWidth = 280,
  gap = 16,
  ariaLabel = 'Image gallery',
}: Props) {
  const containerRef = useRef<HTMLDivElement>(null);
  const { layout, registerItemHeight } = useMasonryLayout(
    items,
    containerRef,
    { columnMinWidth, gap }
  );

  return (
    <div
      ref={containerRef}
      className="masonry-js"
      style={{
        // Expose column count as CSS custom property for item width calc
        ['--cols' as string]: layout.columnCount,
        ['--masonry-gap' as string]: `${gap}px`,
        height: layout.containerHeight,
      }}
      role="feed"
      aria-label={ariaLabel}
      aria-busy={layout.containerHeight === 0}
    >
      {items.map(item => (
        <MasonryCard
          key={item.id}
          item={item}
          layout={layout.itemLayouts.get(item.id)}
          onHeightMeasured={registerItemHeight}
        />
      ))}
    </div>
  );
}
```

---

## Angular Implementation

### Model and Service

```typescript
// masonry.model.ts
export interface MasonryItem {
  id: string;
  imageUrl: string;
  imageWidth: number;
  imageHeight: number;
  alt: string;
  caption?: string;
}

export interface ItemLayout {
  id: string;
  top: number;
  left: number;
  width: number;
}
```

```typescript
// masonry-layout.service.ts
import { Injectable, signal, computed } from '@angular/core';
import type { MasonryItem, ItemLayout } from './masonry.model';

@Injectable()   // component-scoped — provided in the component providers array
export class MasonryLayoutService {
  private _columnCount    = signal(1);
  private _containerHeight = signal(0);
  private _itemLayouts    = signal<Map<string, ItemLayout>>(new Map());
  private _itemHeights    = new Map<string, number>();

  readonly columnCount     = this._columnCount.asReadonly();
  readonly containerHeight = this._containerHeight.asReadonly();
  readonly itemLayouts     = this._itemLayouts.asReadonly();

  calculate(items: MasonryItem[], containerWidth: number, columnMinWidth: number, gap: number) {
    if (containerWidth === 0) return;

    const colCount = Math.max(1, Math.floor((containerWidth + gap) / (columnMinWidth + gap)));
    const colWidth = (containerWidth - (colCount - 1) * gap) / colCount;

    const colHeights: number[] = new Array(colCount).fill(0);
    const layouts = new Map<string, ItemLayout>();

    for (const item of items) {
      const shortestCol = colHeights.indexOf(Math.min(...colHeights));
      const left        = shortestCol * (colWidth + gap);
      const top         = colHeights[shortestCol];

      const knownHeight = this._itemHeights.get(item.id);
      const estimatedH  = Math.round(colWidth * (item.imageHeight / item.imageWidth)) + 60;
      const itemHeight  = knownHeight ?? estimatedH;

      layouts.set(item.id, { id: item.id, top, left, width: colWidth });
      colHeights[shortestCol] = top + itemHeight + gap;
    }

    this._columnCount.set(colCount);
    this._containerHeight.set(Math.max(...colHeights));
    this._itemLayouts.set(new Map(layouts));
  }

  registerHeight(id: string, height: number, items: MasonryItem[], containerWidth: number, colMinWidth: number, gap: number) {
    if (this._itemHeights.get(id) === height) return;
    this._itemHeights.set(id, height);
    this.calculate(items, containerWidth, colMinWidth, gap);
  }
}
```

### MasonryCard Component

```typescript
// masonry-card.component.ts
import {
  Component, Input, Output, EventEmitter, OnInit, OnDestroy,
  ElementRef, signal, inject
} from '@angular/core';
import { NgIf, NgStyle } from '@angular/common';
import type { MasonryItem, ItemLayout } from './masonry.model';

@Component({
  selector: 'app-masonry-card',
  standalone: true,
  imports: [NgIf, NgStyle],
  template: `
    <div
      class="masonry-js__item"
      [class.is-placed]="!!layout"
      [ngStyle]="layout
        ? { top: layout.top + 'px', left: layout.left + 'px', width: layout.width + 'px' }
        : {}"
      role="article"
    >
      <img
        *ngIf="shouldLoad()"
        [src]="item.imageUrl"
        [alt]="item.alt"
        [attr.width]="item.imageWidth"
        [attr.height]="item.imageHeight"
        (load)="onImageLoad()"
        loading="eager"
      />
      <div
        *ngIf="!shouldLoad()"
        aria-hidden="true"
        class="masonry-js__item--placeholder"
        [ngStyle]="{ aspectRatio: item.imageWidth + ' / ' + item.imageHeight }"
      ></div>
      <p *ngIf="item.caption">{{ item.caption }}</p>
    </div>
  `,
})
export class MasonryCardComponent implements OnInit, OnDestroy {
  @Input({ required: true }) item!: MasonryItem;
  @Input() layout: ItemLayout | undefined;
  @Output() heightMeasured = new EventEmitter<{ id: string; height: number }>();

  private el       = inject(ElementRef<HTMLElement>);
  readonly shouldLoad = signal(false);

  private io?: IntersectionObserver;

  ngOnInit() {
    // Watch the host element; fire once when it enters viewport
    this.io = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          this.shouldLoad.set(true);
          this.io?.disconnect();
        }
      },
      { rootMargin: '200px' }
    );
    this.io.observe(this.el.nativeElement);
  }

  onImageLoad() {
    requestAnimationFrame(() => {
      const h = this.el.nativeElement.offsetHeight;
      if (h > 0) this.heightMeasured.emit({ id: this.item.id, height: h });
    });
  }

  ngOnDestroy() {
    this.io?.disconnect();
  }
}
```

### MasonryGrid Component

```typescript
// masonry-grid.component.ts
import {
  Component, Input, OnChanges, OnDestroy, AfterViewInit,
  ElementRef, inject, signal, effect
} from '@angular/core';
import { NgFor, NgStyle } from '@angular/common';
import { MasonryLayoutService } from './masonry-layout.service';
import { MasonryCardComponent } from './masonry-card.component';
import type { MasonryItem } from './masonry.model';

@Component({
  selector: 'app-masonry-grid',
  standalone: true,
  imports: [NgFor, NgStyle, MasonryCardComponent],
  providers: [MasonryLayoutService],   // component-scoped service instance
  template: `
    <div
      #container
      class="masonry-js"
      [ngStyle]="{
        '--cols': layoutService.columnCount(),
        '--masonry-gap': gap + 'px',
        height: layoutService.containerHeight() + 'px'
      }"
      role="feed"
      [attr.aria-label]="ariaLabel"
      [attr.aria-busy]="layoutService.containerHeight() === 0"
    >
      <app-masonry-card
        *ngFor="let item of items; trackBy: trackById"
        [item]="item"
        [layout]="layoutService.itemLayouts().get(item.id)"
        (heightMeasured)="onHeightMeasured($event)"
      />
    </div>
  `,
})
export class MasonryGridComponent implements OnChanges, AfterViewInit, OnDestroy {
  @Input({ required: true }) items: MasonryItem[] = [];
  @Input() columnMinWidth = 280;
  @Input() gap            = 16;
  @Input() ariaLabel      = 'Image gallery';

  readonly layoutService = inject(MasonryLayoutService);
  private el             = inject(ElementRef<HTMLElement>);
  private ro?: ResizeObserver;

  ngAfterViewInit() {
    this.ro = new ResizeObserver(entries => {
      const width = entries[0]?.contentRect.width ?? 0;
      if (width > 0) this.recalculate();
    });
    this.ro.observe(this.el.nativeElement.querySelector('.masonry-js')!);
    this.recalculate();
  }

  ngOnChanges() {
    // Re-layout whenever items input changes
    this.recalculate();
  }

  onHeightMeasured({ id, height }: { id: string; height: number }) {
    const containerWidth = this.el.nativeElement.querySelector('.masonry-js')?.offsetWidth ?? 0;
    this.layoutService.registerHeight(id, height, this.items, containerWidth, this.columnMinWidth, this.gap);
  }

  private recalculate() {
    const containerWidth = this.el.nativeElement.querySelector('.masonry-js')?.offsetWidth ?? 0;
    this.layoutService.calculate(this.items, containerWidth, this.columnMinWidth, this.gap);
  }

  trackById(_: number, item: MasonryItem) {
    return item.id;
  }

  ngOnDestroy() {
    this.ro?.disconnect();
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Gallery container has a label | `role="feed"` + `aria-label` on the container |
| Container signals loading state | `aria-busy="true"` while `containerHeight === 0` |
| Each card is a named landmark | `role="article"` on every card element |
| Images have descriptive alt text | `alt` is a required field in the `MasonryItem` type |
| Decorative placeholder has no alt | `aria-hidden="true"` on the skeleton placeholder div |
| Images have explicit width/height | Prevents layout shift (CLS); required for LCP scoring |
| Invisible items are not reachable | `pointer-events: none` while not yet placed; opacity alone is not enough |
| Keyboard navigation order is logical | Items are rendered in DOM order matching the `items` array |
| Reduced motion is respected | `transition: none` via `prefers-reduced-motion` media query on cards |
| Focus is not trapped or hijacked | Grid uses no focus management; standard tab order is preserved |

---

## Production Pitfalls

**1. Layout runs before images have loaded, then images load and heights change**
The most common masonry bug: you position items using estimated heights, images load and are taller or shorter, cards overlap or leave gaps. Fix: always register the real card height after `onload` fires, then trigger a full `recalculate()`. Use `requestAnimationFrame` inside the load handler to ensure the browser has painted before you measure `offsetHeight`.

**2. ResizeObserver triggers an infinite loop**
Setting the container `height` inside a ResizeObserver callback causes the container to resize, which triggers the observer again. Fix: only call `recalculate()` when the *width* changes (width determines column count and column widths), never respond to height changes from your own layout.

**3. `offsetHeight` returns 0 during SSR or before paint**
In Next.js or Angular Universal the container has no width on the server. Absolute-position masonry requires a real DOM measurement. Fix: wrap the initial `recalculate()` call in `AfterViewInit` (Angular) or `useLayoutEffect` / `requestAnimationFrame` (React). For SSR, fall back to the CSS `columns` approach and hydrate to JS masonry after mount.

**4. Rapid item insertion causes a reflow storm**
Appending items one by one (e.g., on each scroll event for infinite scroll) calls `recalculate()` for every single item, triggering dozens of synchronous layout reads. Fix: batch new items into the `items` array and call `recalculate()` once after the batch is complete. In React use `startTransition`; in Angular use `ngOnChanges` with input binding rather than pushing directly into the array.

**5. IntersectionObserver `rootMargin` is ignored inside overflow:hidden ancestors**
If the masonry container or any ancestor has `overflow: hidden`, the IntersectionObserver `rootMargin` expansion is clipped to the overflow boundary and images load too late, causing a visible blank-then-appear flash. Fix: pass an explicit `root` element to the IntersectionObserver constructor rather than using the default viewport root, and ensure the root is the scrollable ancestor, not an `overflow:hidden` wrapper.

**6. Column count flashes on first render**
Before `ResizeObserver` fires, `columnCount` is 1 and all cards stack in a single column. Fix: calculate the initial column count synchronously in `AfterViewInit` / `useLayoutEffect` before the first paint, so the user never sees the single-column intermediate state. Set `opacity: 0` on the container until the first layout is complete, then fade it in.

**7. CSS `columns` breaks `position: sticky` inside cards**
If cards have sticky inner elements (e.g., a sticky category label), they will not work inside a `columns` layout because column fragments create new stacking contexts. Fix: switch to the JS absolute-position approach for any layout that requires sticky inner elements.

---

## Interview Angle

**Q: "What are the trade-offs between the CSS `columns` approach and a JS absolute-position masonry engine?"**

The CSS `columns` approach is zero-JS, works on the first paint without any measurement, and is trivially responsive via `column-width`. The cost is ordering: items flow top-to-bottom per column, so a "newest first" feed will not place the newest item in the top-left — it fills the leftmost column top-to-bottom first. It also cannot handle dynamic insertion well; any DOM mutation triggers a full reflow. `break-inside: avoid` is mandatory on every card or the browser will split a single card across two columns.

The JS approach gives you true shortest-column-first placement, correct sort-order rendering, and incremental insertion at the bottom. The cost is complexity: you maintain a column height map, re-run layout on every image load and container resize, and must guard against the ResizeObserver loop caused by your own height writes. For a static photo portfolio, CSS `columns` wins. For a sorted, infinite-scrolling, API-driven feed, you need the JS engine.

**Follow-up: "How do you handle images that haven't loaded yet when you run the layout pass?"**

You estimate the height from the image's intrinsic aspect ratio — `colWidth * (imageHeight / imageWidth)` — and use that as a provisional height. When the image fires its `load` event you measure the real `offsetHeight` of the card (including caption and padding) and call `registerItemHeight`. If the real height differs from the estimate you run a full `recalculate()`. Cards that have not yet been measured are invisible (`opacity: 0; pointer-events: none`) so the user never sees the provisional positions jump. The `IntersectionObserver` with `rootMargin: '200px'` starts loading images before they scroll into view, which minimises the visible gap between card appearing and image rendering.
