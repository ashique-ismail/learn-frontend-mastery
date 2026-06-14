# Virtualized Infinite-Scroll Table

## The Idea

**In plain English:** You have a table with potentially tens of thousands of rows — a trade blotter, an audit log, a product catalog. Rendering all rows at once makes the browser grind to a halt: 50,000 `<tr>` elements create a DOM with 50,000+ nodes, and every scroll event forces the browser to paint them all. Virtualization solves this by only rendering the rows that are actually visible on screen, swapping them in and out as the user scrolls. Infinite scroll extends this by fetching the next page of data automatically when the user nears the bottom, making the table feel endless without loading everything upfront.

**Real-world analogy:** Imagine a long paper receipt rolling out of a machine.

- The full receipt is 10 meters long, but the **window cut into the machine** is only 20 cm tall — you only see 20 cm at a time.
- As you pull the paper down, the machine **prints new lines ahead** of what you can see and **destroys the lines** that have scrolled far above the window — they are gone from the paper, but if you scroll back up, the machine reprints them from memory.
- The **printer mechanism** = the virtualization engine (TanStack Virtual / CDK Virtual Scroll).
- The **roll of paper waiting to be printed** = the next page of data fetched via Intersection Observer.
- **Lines with different heights** (itemized vs. summary rows) = dynamic row heights that the engine must measure individually rather than assuming a fixed size.
- **"Loading..." lines printed at the bottom** = skeleton rows shown while the next API page is in flight.

The key insight: virtualization and infinite scroll are independent concerns that compose together — virtualization is a rendering optimization, infinite scroll is a data-fetching pattern.

---

## Learning Objectives

- Understand why DOM size, not data size, is the real performance constraint
- Configure TanStack Virtual with dynamic row heights using `measureElement`
- Configure Angular CDK Virtual Scroll with a size strategy for variable heights
- Wire an Intersection Observer to trigger next-page fetches without scroll event listeners
- Implement programmatic scroll-to-row by index and by item identity
- Show skeleton rows during loading without layout shift
- Handle edge cases: row height estimation, overscan, infinite fetch loops

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Limit visible rows to the viewport | ✅ `overflow: hidden` on container | — |
| Remove off-screen DOM nodes | ❌ | CSS cannot delete or create DOM elements |
| Measure individual row heights after render | ❌ | CSS cannot read computed dimensions back into JS |
| Detect when the user scrolls near the bottom | ❌ | `scroll` events and `IntersectionObserver` require JS |
| Fetch next page from an API | ❌ | No network access from CSS |
| Show skeleton rows while data loads | ⚠️ | CSS can style skeletons, but JS decides when to render them |
| Maintain total scroll height as data grows | ❌ | The spacer element size must be set via JS |
| Scroll to an arbitrary row index programmatically | ❌ | `scroll-snap` only works for fixed uniform layouts |

**Conclusion:** CSS owns shimmer animations, row styling, and the scroll container's `overflow`. JS owns everything else — the virtual window, the spacer, the fetch trigger, and the skeleton visibility logic.

---

## HTML & CSS Foundation

### The Scroll Container Shape

```html
<!--
  Three layers:
  1. .vt-scroll-container — the fixed-height, overflow:auto element
  2. .vt-total-height-spacer — a div whose height equals total estimated row heights,
     creating the correct scrollbar thumb size
  3. .vt-row-window — absolutely positioned, holds only the visible rows
-->
<div class="vt-scroll-container" role="grid" aria-label="Data table" aria-rowcount="50000">
  <div class="vt-total-height-spacer">
    <div class="vt-row-window">
      <!-- Virtual rows rendered here -->
      <div class="vt-row" role="row" aria-rowindex="1" data-index="0">
        <!-- cells -->
      </div>
      <!-- Skeleton row while loading -->
      <div class="vt-row vt-row--skeleton" role="row" aria-busy="true" aria-label="Loading">
        <div class="vt-skeleton-cell"></div>
        <div class="vt-skeleton-cell vt-skeleton-cell--wide"></div>
        <div class="vt-skeleton-cell"></div>
      </div>
    </div>
  </div>

  <!-- Sentinel element: entering viewport triggers next-page fetch -->
  <div class="vt-fetch-sentinel" aria-hidden="true"></div>
</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --vt-row-estimated-height: 48px;
  --vt-container-height: 600px;
  --vt-border-color: #e2e8f0;
  --vt-row-hover-bg: #f7fafc;
  --vt-skeleton-base: #e2e8f0;
  --vt-skeleton-shine: #f0f4f8;
  --vt-transition: 150ms ease;
}

/* ─── Scroll container ─── */
.vt-scroll-container {
  height: var(--vt-container-height);
  overflow-y: auto;
  overflow-x: hidden;
  position: relative;
  border: 1px solid var(--vt-border-color);
  border-radius: 6px;
  /* Prevent iOS rubber-band from confusing the Intersection Observer */
  overscroll-behavior: contain;
  /* GPU-accelerated scrolling on iOS */
  -webkit-overflow-scrolling: touch;
  will-change: scroll-position;
}

/* ─── Spacer: sets the total scrollable height ─── */
/*
  JS sets this element's height to (estimatedRowHeight * totalRows).
  It is the positioning context for the row window.
*/
.vt-total-height-spacer {
  position: relative;
  width: 100%;
  /* height set inline by virtualization engine */
}

/* ─── Row window: only visible rows ─── */
.vt-row-window {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  /* transform: translateY(...) set by JS to position the visible slice */
}

/* ─── Individual rows ─── */
.vt-row {
  display: grid;
  grid-template-columns: 80px 1fr 120px 100px;
  align-items: center;
  border-bottom: 1px solid var(--vt-border-color);
  padding: 0 12px;
  min-height: var(--vt-row-estimated-height);
  /* Rows may have variable height — min-height, not height */
  box-sizing: border-box;
  transition: background-color var(--vt-transition);
}

.vt-row:hover { background-color: var(--vt-row-hover-bg); }

/* ─── Header row ─── */
.vt-row--header {
  position: sticky;
  top: 0;
  background: #fff;
  z-index: 1;
  font-weight: 600;
  border-bottom: 2px solid var(--vt-border-color);
}

/* ─── Skeleton rows ─── */
.vt-row--skeleton { pointer-events: none; }

.vt-skeleton-cell {
  height: 14px;
  border-radius: 3px;
  background: linear-gradient(
    90deg,
    var(--vt-skeleton-base) 25%,
    var(--vt-skeleton-shine) 50%,
    var(--vt-skeleton-base) 75%
  );
  background-size: 200% 100%;
  animation: vt-shimmer 1.4s infinite;
  width: 60%;
}

.vt-skeleton-cell--wide { width: 85%; }

@keyframes vt-shimmer {
  0%   { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}

/* ─── Fetch sentinel ─── */
.vt-fetch-sentinel {
  position: absolute;
  bottom: 0;
  height: 1px;          /* invisible, but must have size for IO to detect */
  width: 100%;
  pointer-events: none;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .vt-skeleton-cell {
    animation: none;
    background: var(--vt-skeleton-base);
  }
}
```

**What CSS owns:** shimmer animation, row grid layout, sticky header, hover state, sentinel sizing, reduced-motion fallback.

**What CSS cannot own:** which rows are rendered, their vertical offset, the spacer height, the fetch trigger, or skeleton visibility.

---

## React Implementation

### Types

```tsx
// types.ts
export interface RowData {
  id: string;
  index: number;
  ticker: string;
  description: string;   // can be long — causes variable row height
  quantity: number;
  price: number;
}

export interface Page {
  rows: RowData[];
  nextCursor: string | null;
}
```

### Data Fetching Hook

```tsx
// useInfiniteRows.ts
import { useState, useCallback, useRef } from 'react';
import type { RowData, Page } from './types';

const PAGE_SIZE = 50;

async function fetchPage(cursor: string | null): Promise<Page> {
  const params = new URLSearchParams({ limit: String(PAGE_SIZE) });
  if (cursor) params.set('cursor', cursor);
  const res = await fetch(`/api/rows?${params}`);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();
}

export function useInfiniteRows() {
  const [rows, setRows]       = useState<RowData[]>([]);
  const [loading, setLoading] = useState(false);
  const [hasMore, setHasMore] = useState(true);
  const cursorRef             = useRef<string | null>(null);
  // Ref prevents stale closure in Intersection Observer callback
  const loadingRef            = useRef(false);

  const loadMore = useCallback(async () => {
    if (loadingRef.current || !hasMore) return;
    loadingRef.current = true;
    setLoading(true);

    try {
      const page = await fetchPage(cursorRef.current);
      setRows(prev => [...prev, ...page.rows]);
      cursorRef.current = page.nextCursor;
      if (!page.nextCursor) setHasMore(false);
    } catch (err) {
      console.error('Failed to load rows:', err);
    } finally {
      loadingRef.current = false;
      setLoading(false);
    }
  }, [hasMore]);

  return { rows, loading, hasMore, loadMore };
}
```

### Intersection Observer Hook

```tsx
// useFetchSentinel.ts
import { useEffect, useRef } from 'react';

export function useFetchSentinel(
  onIntersect: () => void,
  enabled: boolean,
  scrollRoot: React.RefObject<HTMLElement | null>,
) {
  const sentinelRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!enabled || !sentinelRef.current) return;

    const sentinel = sentinelRef.current;

    // rootMargin: fire 200px before the sentinel reaches the viewport bottom
    // so data loads before the user actually hits the end.
    // root must be the scroll container, not the browser viewport —
    // otherwise it fires incorrectly inside modals or nested scroll layouts.
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) onIntersect();
      },
      {
        root: scrollRoot.current,
        rootMargin: '0px 0px 200px 0px',
        threshold: 0,
      },
    );

    observer.observe(sentinel);
    return () => observer.disconnect();
  }, [onIntersect, enabled, scrollRoot]);

  return sentinelRef;
}
```

### Virtualized Table Component

```tsx
// VirtualizedTable.tsx
import { useRef, useCallback, useEffect } from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';
import { useInfiniteRows } from './useInfiniteRows';
import { useFetchSentinel } from './useFetchSentinel';
import type { RowData } from './types';

const SKELETON_COUNT = 5;

export function VirtualizedTable() {
  const { rows, loading, hasMore, loadMore } = useInfiniteRows();
  const scrollContainerRef = useRef<HTMLDivElement>(null);

  // Load the first page on mount
  useEffect(() => { loadMore(); }, [loadMore]);

  const rowVirtualizer = useVirtualizer({
    count: rows.length + (loading ? SKELETON_COUNT : 0),
    getScrollElement: () => scrollContainerRef.current,

    // Estimated height used before measurement.
    // Accuracy matters: a bad estimate causes scroll position jumps
    // when the virtualizer corrects the spacer after measuring real heights.
    estimateSize: () => 48,

    // Dynamic height: after each row mounts, TanStack measures it via
    // ResizeObserver and updates the total spacer height.
    // The data-index attribute on the row element is required for this to work.
    measureElement: (el) => el.getBoundingClientRect().height,

    overscan: 5,    // render 5 extra rows above and below the visible area
  });

  // Sentinel triggers next page fetch
  const sentinelRef = useFetchSentinel(loadMore, hasMore && !loading, scrollContainerRef);

  // Programmatic scroll-to-row by index
  const scrollToRow = useCallback((index: number) => {
    rowVirtualizer.scrollToIndex(index, { align: 'start', behavior: 'smooth' });
  }, [rowVirtualizer]);

  // Scroll-to-row by item identity — searches loaded rows first
  const scrollToId = useCallback((id: string) => {
    const index = rows.findIndex(r => r.id === id);
    if (index === -1) return;
    rowVirtualizer.scrollToIndex(index, { align: 'center', behavior: 'smooth' });
  }, [rows, rowVirtualizer]);

  const virtualItems = rowVirtualizer.getVirtualItems();

  return (
    <div style={{ display: 'flex', flexDirection: 'column', gap: 8 }}>
      <button onClick={() => scrollToRow(500)} style={{ alignSelf: 'flex-start' }}>
        Jump to row 500
      </button>

      {/* Sticky header — outside the scroll container */}
      <div className="vt-row vt-row--header" role="row">
        <span role="columnheader">#</span>
        <span role="columnheader">Description</span>
        <span role="columnheader">Qty</span>
        <span role="columnheader">Price</span>
      </div>

      {/* Scroll container */}
      <div
        ref={scrollContainerRef}
        className="vt-scroll-container"
        role="grid"
        aria-label="Trade blotter"
        aria-rowcount={rows.length}
        aria-busy={loading}
      >
        {/* Spacer: height = total estimated height of all rows */}
        <div
          className="vt-total-height-spacer"
          style={{ height: rowVirtualizer.getTotalSize() }}
        >
          {/* Row window: absolutely positioned, translated to visible slice */}
          <div
            className="vt-row-window"
            style={{
              transform: `translateY(${virtualItems[0]?.start ?? 0}px)`,
            }}
          >
            {virtualItems.map((virtualRow) => {
              const isSkeleton = virtualRow.index >= rows.length;
              const row: RowData | undefined = rows[virtualRow.index];

              return isSkeleton ? (
                <div
                  key={`skeleton-${virtualRow.index}`}
                  className="vt-row vt-row--skeleton"
                  role="row"
                  aria-busy="true"
                  aria-label="Loading"
                  data-index={virtualRow.index}
                  ref={rowVirtualizer.measureElement}
                >
                  <div className="vt-skeleton-cell" />
                  <div className="vt-skeleton-cell vt-skeleton-cell--wide" />
                  <div className="vt-skeleton-cell" />
                  <div className="vt-skeleton-cell" />
                </div>
              ) : (
                <div
                  key={row!.id}
                  className="vt-row"
                  role="row"
                  aria-rowindex={virtualRow.index + 1}
                  data-index={virtualRow.index}
                  // The ref callback hands the DOM node to TanStack Virtual
                  // so it can call measureElement and get the real height.
                  ref={rowVirtualizer.measureElement}
                >
                  <span role="gridcell">{row!.index}</span>
                  <span role="gridcell">{row!.description}</span>
                  <span role="gridcell">{row!.quantity.toLocaleString()}</span>
                  <span role="gridcell">${row!.price.toFixed(2)}</span>
                </div>
              );
            })}
          </div>
        </div>

        {/* Fetch sentinel — inside the scroll container, at the very bottom */}
        <div ref={sentinelRef} className="vt-fetch-sentinel" aria-hidden="true" />
      </div>

      {!hasMore && (
        <p role="status" aria-live="polite" style={{ textAlign: 'center', color: '#718096' }}>
          All {rows.length} rows loaded.
        </p>
      )}
    </div>
  );
}
```

---

## Angular Implementation

### Model and Service

```typescript
// virtual-table.model.ts
export interface RowData {
  id: string;
  index: number;
  ticker: string;
  description: string;
  quantity: number;
  price: number;
}

export interface Page {
  rows: RowData[];
  nextCursor: string | null;
}
```

```typescript
// virtual-table.service.ts
import { Injectable, signal, computed } from '@angular/core';
import { RowData, Page } from './virtual-table.model';

@Injectable({ providedIn: 'root' })
export class VirtualTableService {
  private _rows    = signal<RowData[]>([]);
  private _loading = signal(false);
  private _hasMore = signal(true);
  private _cursor  = signal<string | null>(null);

  readonly rows    = this._rows.asReadonly();
  readonly loading = this._loading.asReadonly();
  readonly hasMore = this._hasMore.asReadonly();
  readonly total   = computed(() => this._rows().length);

  async loadMore(): Promise<void> {
    if (this._loading() || !this._hasMore()) return;
    this._loading.set(true);

    try {
      const params = new URLSearchParams({ limit: '50' });
      const cursor = this._cursor();
      if (cursor) params.set('cursor', cursor);

      const res  = await fetch(`/api/rows?${params}`);
      const page: Page = await res.json();

      this._rows.update(prev => [...prev, ...page.rows]);
      this._cursor.set(page.nextCursor);
      if (!page.nextCursor) this._hasMore.set(false);
    } finally {
      this._loading.set(false);
    }
  }
}
```

### Virtualized Table Component with CDK Virtual Scroll

```typescript
// virtual-table.component.ts
import {
  Component, OnInit, OnDestroy, inject,
  ViewChild, ElementRef, AfterViewInit, computed
} from '@angular/core';
import { CdkVirtualScrollViewport, ScrollingModule } from '@angular/cdk/scrolling';
import { NgFor, NgIf, DecimalPipe, CurrencyPipe } from '@angular/common';
import { VirtualTableService } from './virtual-table.service';
import { RowData } from './virtual-table.model';

// CDK Virtual Scroll uses a fixed item size strategy by default.
// For variable heights pair with AutoSizeVirtualScrollStrategy (experimental)
// or implement a custom VirtualScrollStrategy — see pitfalls section.
const ESTIMATED_ROW_HEIGHT = 48;
const LOAD_THRESHOLD = 8;    // fetch next page when within 8 rows of the end
const SKELETON_COUNT = 5;

type DisplayRow = RowData | { __skeleton: true; id: string; index: number };

@Component({
  selector: 'app-virtual-table',
  standalone: true,
  imports: [ScrollingModule, NgFor, NgIf, DecimalPipe, CurrencyPipe],
  template: `
    <!-- Sticky header -->
    <div class="vt-row vt-row--header" role="row">
      <span role="columnheader">#</span>
      <span role="columnheader">Description</span>
      <span role="columnheader">Qty</span>
      <span role="columnheader">Price</span>
    </div>

    <!-- CDK virtual scroll viewport -->
    <cdk-virtual-scroll-viewport
      #viewport
      class="vt-scroll-container"
      [itemSize]="estimatedRowHeight"
      role="grid"
      aria-label="Trade blotter"
      [attr.aria-rowcount]="svc.total()"
      [attr.aria-busy]="svc.loading()"
      (scrolledIndexChange)="onScrollIndexChange($event)"
    >
      <div
        *cdkVirtualFor="
          let row of displayRows();
          trackBy: trackById;
          templateCacheSize: 20
        "
        [class]="row.__skeleton ? 'vt-row vt-row--skeleton' : 'vt-row'"
        role="row"
        [attr.aria-rowindex]="isData(row) ? row.index + 1 : null"
        [attr.aria-busy]="row.__skeleton ? true : null"
      >
        <!-- Data row -->
        <ng-container *ngIf="isData(row)">
          <span role="gridcell">{{ row.index }}</span>
          <span role="gridcell">{{ row.description }}</span>
          <span role="gridcell">{{ row.quantity | number }}</span>
          <span role="gridcell">{{ row.price | currency }}</span>
        </ng-container>

        <!-- Skeleton row -->
        <ng-container *ngIf="row.__skeleton">
          <div class="vt-skeleton-cell"></div>
          <div class="vt-skeleton-cell vt-skeleton-cell--wide"></div>
          <div class="vt-skeleton-cell"></div>
          <div class="vt-skeleton-cell"></div>
        </ng-container>
      </div>
    </cdk-virtual-scroll-viewport>

    <!-- Sentinel: Intersection Observer target -->
    <div #sentinel class="vt-fetch-sentinel" aria-hidden="true"></div>

    <p
      *ngIf="!svc.hasMore()"
      role="status"
      aria-live="polite"
      style="text-align: center; color: #718096"
    >
      All {{ svc.total() }} rows loaded.
    </p>
  `,
})
export class VirtualTableComponent implements OnInit, AfterViewInit, OnDestroy {
  svc = inject(VirtualTableService);

  @ViewChild('viewport', { read: CdkVirtualScrollViewport })
  viewport!: CdkVirtualScrollViewport;

  @ViewChild('sentinel', { read: ElementRef })
  sentinel!: ElementRef<HTMLDivElement>;

  @ViewChild('viewport', { read: ElementRef })
  viewportEl!: ElementRef<HTMLElement>;

  readonly estimatedRowHeight = ESTIMATED_ROW_HEIGHT;

  // Merge real rows + skeleton placeholders into one array for *cdkVirtualFor
  readonly displayRows = computed<DisplayRow[]>(() => {
    const rows = this.svc.rows();
    const loading = this.svc.loading();
    const skeletons: DisplayRow[] = loading
      ? Array.from({ length: SKELETON_COUNT }, (_, i) => ({
          __skeleton: true as const,
          id: `skeleton-${i}`,
          index: rows.length + i,
        }))
      : [];
    return [...rows, ...skeletons];
  });

  private observer?: IntersectionObserver;

  ngOnInit(): void {
    this.svc.loadMore();
  }

  ngAfterViewInit(): void {
    this.setupIntersectionObserver();
  }

  private setupIntersectionObserver(): void {
    if (!this.sentinel?.nativeElement || !this.viewportEl?.nativeElement) return;

    this.observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) this.svc.loadMore();
      },
      // root must be the CDK scroll viewport element, not the browser window.
      // Without this, the observer misfires inside modals or nested scrollers.
      {
        root: this.viewportEl.nativeElement,
        rootMargin: '0px 0px 200px 0px',
        threshold: 0,
      },
    );

    this.observer.observe(this.sentinel.nativeElement);
  }

  // Belt-and-suspenders: also fetch when CDK reports near-end scroll index
  onScrollIndexChange(firstVisibleIndex: number): void {
    const total = this.displayRows().length;
    if (firstVisibleIndex + LOAD_THRESHOLD >= total) {
      this.svc.loadMore();
    }
  }

  // Programmatic scroll to row index
  scrollToRow(index: number): void {
    this.viewport?.scrollToIndex(index, 'smooth');
  }

  // Scroll to row by identity
  scrollToId(id: string): void {
    const index = this.svc.rows().findIndex(r => r.id === id);
    if (index !== -1) this.scrollToRow(index);
  }

  isData(row: DisplayRow): row is RowData {
    return !('__skeleton' in row && row.__skeleton);
  }

  trackById(_: number, row: DisplayRow): string {
    return row.id;
  }

  ngOnDestroy(): void {
    this.observer?.disconnect();
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Container has `role="grid"` and `aria-label` | Set on the scroll container element |
| `aria-rowcount` reflects total known rows | Updated reactively via signal / state |
| `aria-busy="true"` on container while loading | Toggled by loading state |
| Each data row has `role="row"` and `aria-rowindex` | Set on every rendered row div |
| Skeleton rows have `role="row"` and `aria-busy="true"` | Announced as "loading" to assistive technology |
| Column headers have `role="columnheader"` | Applied to sticky header cells |
| `aria-live="polite"` region announces "all rows loaded" | Rendered when `hasMore` becomes false |
| Intersection Observer uses `rootMargin`, not `scroll` events | Reduces main-thread load; more reliable on touch devices |
| Skeleton shimmer animation disabled under `prefers-reduced-motion` | CSS `@media` query removes the animation |
| Sticky header remains visible and keyboard-reachable | Positioned outside the scroll container |
| Focus is not lost when rows re-render during scroll | TanStack/CDK keyed renders preserve focus on visible cells |

---

## Production Pitfalls

**1. Scroll position jumps when row heights are measured**
When `estimateSize` is wrong and rows are taller than estimated, TanStack Virtual corrects the spacer height after measurement, causing the scroll position to jump. Fix: calibrate `estimateSize` against real data — sample a few rows server-side and use the p90 height as your estimate. The jump will still happen on the first render of each row, but only once, not continuously.

**2. `measureElement` fires before layout is complete**
If you call `measureElement` synchronously during render, the browser may not have applied CSS yet, so `getBoundingClientRect().height` returns 0. TanStack Virtual handles this correctly via an internal `ResizeObserver` — but only if you pass the DOM node via the `ref` prop and never call `measureElement` manually in an effect. Make sure `data-index` is set on the row element; TanStack uses it to map the DOM node back to the right virtual item.

**3. Intersection Observer fires with the wrong root**
If the sentinel is inside the scroll container but you omit `root: scrollContainer`, the IO uses the browser viewport as its root. On a simple full-page layout this works by coincidence. In a modal, iframe, or nested scroll layout it fires too early or never. Always explicitly set `root` to the scroll container element.

**4. Infinite fetch loop when the sentinel is always visible**
If the container height is larger than the loaded data (e.g., only 10 rows loaded into a 600px container), the sentinel at the bottom is always intersecting. Every load triggers another load. Fix: set `hasMore = false` when the API returns fewer rows than `PAGE_SIZE`, and guard `loadMore` with a `loading` flag (both a React state and a ref, to avoid stale closures in the IO callback).

**5. `trackBy` identity missing from `*cdkVirtualFor`**
Without `trackBy`, Angular tears down and recreates every rendered row on every change — defeating the purpose of virtualization. Always provide a stable identity function (`trackBy: trackById`) that returns the row's unique `id`, not its array index.

**6. Programmatic `scrollToIndex` on unloaded rows**
If you call `scrollToRow(9999)` but only 100 rows are loaded, the virtualizer scrolls to the end of the current data, not row 9999. Fix: expose `scrollToId(id)` backed by a `findIndex` call against loaded rows. For scenarios that require jumping to truly unloaded data, first issue a cursor-skip API call to load the target page, then scroll.

**7. CDK `FixedSizeVirtualScrollStrategy` breaks with variable heights**
CDK's built-in strategy assumes every item has the same height. For truly variable heights, use `AutoSizeVirtualScrollStrategy` from `@angular/cdk/scrolling` (experimental) or implement `VirtualScrollStrategy` yourself. The trade-off: auto-size must render each item once off-screen to measure it, costing one extra paint cycle per item the first time it enters the viewport.

---

## Interview Angle

**Q: "We have a table that needs to display 100,000 rows. How do you approach this?"** A strong answer covers four layers in order. First, data fetching: never load all 100,000 rows upfront — use cursor-based pagination. An Intersection Observer on a sentinel element near the bottom of the scroll container triggers the next-page fetch without scroll event listeners, which fire on every pixel of movement and saturate the main thread. Second, rendering: virtualize. Only the rows currently visible, plus a small overscan buffer, exist in the DOM. TanStack Virtual manages this via a spacer div (whose height equals total estimated row height) and a row window div (absolutely positioned, translated to the visible slice). Angular CDK Virtual Scroll does the same with `cdk-virtual-scroll-viewport`. Third, dynamic heights: if rows have variable heights (multiline descriptions, expandable cells), wire `measureElement` so TanStack Virtual can observe each row's real height via a `ResizeObserver` and correct the spacer. Provide an accurate `estimateSize` to minimize layout jumps on first render. Fourth, UX during loading: show skeleton rows at the indices beyond the last loaded row. Skeletons have `aria-busy="true"` and the same grid layout as real rows, so neither the layout nor the ARIA tree shifts when real data arrives.

**Follow-up: "What breaks in a modal or inside another scrollable container?"** Intersection Observer needs its `root` set to the scroll container element, not the default `null` which uses the browser viewport. In a modal, the scroll container is the modal body. If you omit `root`, the sentinel is often already intersecting when the modal opens, and `loadMore` fires in a loop before the first page is even displayed. Also: CDK Virtual Scroll's `scrolledIndexChange` is reliable only when the viewport element is the actual scroll ancestor — verify with `overflow: auto` on exactly one ancestor. Multiple nested `overflow: auto` elements cause CDK to miscalculate scroll offset and render the wrong slice.
