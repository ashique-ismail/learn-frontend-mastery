# Breadcrumb with Overflow Truncation

## The Idea

**In plain English:** A breadcrumb trail shows where you are in a site hierarchy — "Home > Products > Electronics > Laptops > 15-inch Models". The problem is that on narrow screens or with deeply nested paths, this trail grows too long to display in one line. You need to intelligently truncate the middle of the path while always keeping the first item ("Home") and the last item (the current page) visible.

**Real-world analogy:** Think of a GPS device showing your route.

- The **full route** on a big map screen shows every turn: "Start → Highway 1 → Exit 42 → Oak Street → Elm Avenue → 123 Main St".
- On the **small dashboard display**, the GPS only shows "Start … 123 Main St" with an ellipsis in the middle — you can always see where you started and where you're going, but the intermediate steps are hidden.
- If you tap the display, it **expands** to show all the stops.
- The breadcrumb has the same two modes: **collapsed** (start + end visible, middle truncated with ellipsis) and **expanded** (all items visible, possibly wrapping).

The engineering complexity: CSS `text-overflow: ellipsis` only works on a single-line block element — it can't truncate a list of items at the midpoint. You need either a container query or JavaScript to measure and decide which items to hide.

---

## Learning Objectives

- Understand why `text-overflow: ellipsis` cannot truncate a list of breadcrumb items
- Use CSS `container queries` to respond to available breadcrumb width
- Implement JS-based overflow detection to hide middle items
- Build the "expand on click" interaction with correct ARIA
- Handle the edge case where only 1 or 2 items exist (no truncation needed)
- Use `aria-label="breadcrumb"` and `aria-current="page"` correctly

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Truncate a long text string with ellipsis | ✅ `text-overflow: ellipsis` on a single element | — |
| Detect that the breadcrumb row is overflowing | ⚠️ `overflow: hidden` hides it but not detects it | CSS cannot read `scrollWidth` vs `clientWidth` |
| Always keep first and last item visible | ❌ | CSS cannot target "all except first and last" dynamically |
| Insert "…" button in the middle | ❌ | CSS cannot add interactive DOM nodes |
| Expand on click to show all items | ❌ | Requires toggling hidden attribute on DOM nodes |
| Respond to container width (not viewport) | ✅ Container queries | Only works for styling, not DOM manipulation |

---

## HTML & CSS Foundation

```html
<nav aria-label="Breadcrumb" class="breadcrumb-nav">
  <ol class="breadcrumb" role="list">
    <li class="breadcrumb__item">
      <a href="/" class="breadcrumb__link">Home</a>
    </li>

    <!-- Hidden middle items when collapsed -->
    <li class="breadcrumb__item breadcrumb__item--hidden" aria-hidden="true">
      <a href="/products" class="breadcrumb__link">Products</a>
    </li>
    <li class="breadcrumb__item breadcrumb__item--hidden" aria-hidden="true">
      <a href="/products/electronics" class="breadcrumb__link">Electronics</a>
    </li>

    <!-- Ellipsis toggle — shown when items are hidden -->
    <li class="breadcrumb__item breadcrumb__item--ellipsis" aria-hidden="true">
      <button
        class="breadcrumb__expand-btn"
        aria-label="Show full path"
        aria-expanded="false"
      >…</button>
    </li>

    <li class="breadcrumb__item">
      <a href="/products/electronics/laptops" class="breadcrumb__link">Laptops</a>
    </li>

    <!-- Current page: no link, aria-current -->
    <li class="breadcrumb__item breadcrumb__item--current">
      <span aria-current="page">15-inch Models</span>
    </li>
  </ol>
</nav>
```

```css
/* ─── Container query setup ─── */
.breadcrumb-nav {
  container-type: inline-size;
  container-name: breadcrumb;
}

/* ─── Breadcrumb row ─── */
.breadcrumb {
  display: flex;
  flex-wrap: nowrap;         /* single line — overflow is hidden */
  align-items: center;
  gap: 0;
  margin: 0;
  padding: 0;
  list-style: none;
  overflow: hidden;
  max-width: 100%;
}

/* ─── Items ─── */
.breadcrumb__item {
  display: flex;
  align-items: center;
  white-space: nowrap;
  min-width: 0;             /* allow flex children to shrink */
}

/* Separator via ::after pseudo-element */
.breadcrumb__item:not(:last-child)::after {
  content: '/';
  padding: 0 0.5rem;
  color: #9ca3af;
  font-size: 0.875rem;
  flex-shrink: 0;
}

/* ─── Links and current ─── */
.breadcrumb__link {
  color: #4b5563;
  text-decoration: none;
  font-size: 0.875rem;
  overflow: hidden;
  text-overflow: ellipsis;   /* truncate an individual overly long label */
  max-width: 160px;
  display: block;
}

.breadcrumb__link:hover,
.breadcrumb__link:focus-visible {
  color: #2563eb;
  text-decoration: underline;
}

.breadcrumb__item--current span {
  font-size: 0.875rem;
  color: #111827;
  font-weight: 500;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 200px;
  display: block;
}

/* ─── Ellipsis toggle button ─── */
.breadcrumb__expand-btn {
  background: #f3f4f6;
  border: 1px solid #d1d5db;
  border-radius: 4px;
  padding: 0.125rem 0.5rem;
  font-size: 1rem;
  line-height: 1;
  cursor: pointer;
  color: #6b7280;
  transition: background 0.15s;
}

.breadcrumb__expand-btn:hover {
  background: #e5e7eb;
}

/* ─── Hidden state ─── */
.breadcrumb__item--hidden {
  display: none;
}

.breadcrumb__item--ellipsis {
  display: flex;
}

/* When breadcrumb is expanded, show hidden items, hide ellipsis */
.breadcrumb.is-expanded .breadcrumb__item--hidden {
  display: flex;
}

.breadcrumb.is-expanded .breadcrumb__item--ellipsis {
  display: none;
}

/* ─── Container query: wide enough to show all items ─── */
@container breadcrumb (min-width: 600px) {
  .breadcrumb__item--hidden {
    display: flex;
  }
  .breadcrumb__item--ellipsis {
    display: none;
  }
  .breadcrumb__link {
    max-width: none;
  }
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .breadcrumb__expand-btn { transition: none; }
}
```

---

## React Implementation

```tsx
// Breadcrumb.tsx
import { useState, useRef, useEffect, useLayoutEffect } from 'react';

export interface BreadcrumbItem {
  label: string;
  href?: string;       // omit for current page
}

interface BreadcrumbProps {
  items: BreadcrumbItem[];
  maxVisible?: number; // default: auto-detect via overflow
}

export function Breadcrumb({ items, maxVisible }: BreadcrumbProps) {
  const [expanded, setExpanded] = useState(false);
  const [collapsedIndices, setCollapsedIndices] = useState<Set<number>>(new Set());
  const olRef = useRef<HTMLOListElement>(null);

  // Always keep index 0 (first) and items.length-1 (last) visible.
  // Collapse middle items when overflow is detected.
  useLayoutEffect(() => {
    const ol = olRef.current;
    if (!ol || expanded) return;

    const collapseMiddle = () => {
      // Reset to show all
      setCollapsedIndices(new Set());

      requestAnimationFrame(() => {
        if (!ol) return;
        const isOverflowing = ol.scrollWidth > ol.clientWidth;
        if (!isOverflowing) return;

        // Collapse from middle outward
        const middleIndices = items
          .map((_, i) => i)
          .filter(i => i !== 0 && i !== items.length - 1);

        // Hide all middle items
        setCollapsedIndices(new Set(middleIndices));
      });
    };

    collapseMiddle();

    const ro = new ResizeObserver(collapseMiddle);
    ro.observe(ol);
    return () => ro.disconnect();
  }, [items, expanded]);

  const current = items[items.length - 1];
  const hasHidden = collapsedIndices.size > 0 && !expanded;

  return (
    <nav aria-label="Breadcrumb" className="breadcrumb-nav">
      <ol
        ref={olRef}
        className={`breadcrumb ${expanded ? 'is-expanded' : ''}`}
        role="list"
      >
        {items.map((item, i) => {
          const isHidden  = collapsedIndices.has(i) && !expanded;
          const isCurrent = i === items.length - 1;

          return (
            <li
              key={item.href ?? item.label}
              className={[
                'breadcrumb__item',
                isHidden ? 'breadcrumb__item--hidden' : '',
                isCurrent ? 'breadcrumb__item--current' : '',
              ].join(' ').trim()}
              aria-hidden={isHidden ? true : undefined}
            >
              {isCurrent ? (
                <span aria-current="page">{item.label}</span>
              ) : (
                <a className="breadcrumb__link" href={item.href}>
                  {item.label}
                </a>
              )}
            </li>
          );
        })}

        {/* Ellipsis toggle — placed between first and last */}
        {hasHidden && (
          <li
            className="breadcrumb__item breadcrumb__item--ellipsis"
            style={{ order: 1 }} // CSS order between first (order:0) and rest
            aria-hidden="true"
          >
            <button
              className="breadcrumb__expand-btn"
              aria-label={`Show ${collapsedIndices.size} hidden steps`}
              aria-expanded={false}
              onClick={() => setExpanded(true)}
            >
              …
            </button>
          </li>
        )}
      </ol>
    </nav>
  );
}
```

---

## Angular Implementation

```typescript
// breadcrumb.model.ts
export interface BreadcrumbItem {
  label: string;
  href?: string;
}
```

```typescript
// breadcrumb.component.ts
import {
  Component, Input, signal, computed,
  ElementRef, AfterViewInit, OnDestroy,
  ViewChild, ChangeDetectionStrategy, ChangeDetectorRef, inject
} from '@angular/core';
import { NgFor, NgIf } from '@angular/common';
import { BreadcrumbItem } from './breadcrumb.model';

@Component({
  selector: 'app-breadcrumb',
  standalone: true,
  imports: [NgFor, NgIf],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <nav aria-label="Breadcrumb" class="breadcrumb-nav">
      <ol
        #olEl
        class="breadcrumb"
        [class.is-expanded]="expanded()"
        role="list"
      >
        <li
          *ngFor="let item of items; let i = index; let last = last; trackBy: trackByLabel"
          class="breadcrumb__item"
          [class.breadcrumb__item--hidden]="isHidden(i)"
          [class.breadcrumb__item--current]="last"
          [attr.aria-hidden]="isHidden(i) || null"
        >
          <span *ngIf="last" aria-current="page">{{ item.label }}</span>
          <a *ngIf="!last" class="breadcrumb__link" [href]="item.href">{{ item.label }}</a>
        </li>

        <li
          *ngIf="hasHidden()"
          class="breadcrumb__item breadcrumb__item--ellipsis"
          aria-hidden="true"
          style="order: 1"
        >
          <button
            class="breadcrumb__expand-btn"
            [attr.aria-label]="'Show ' + hiddenCount() + ' hidden steps'"
            aria-expanded="false"
            (click)="expand()"
          >…</button>
        </li>
      </ol>
    </nav>
  `,
})
export class BreadcrumbComponent implements AfterViewInit, OnDestroy {
  @Input() items: BreadcrumbItem[] = [];
  @ViewChild('olEl') olEl!: ElementRef<HTMLOListElement>;

  expanded   = signal(false);
  collapsed  = signal<Set<number>>(new Set());
  private ro?: ResizeObserver;
  private cdr = inject(ChangeDetectorRef);

  hasHidden   = computed(() => this.collapsed().size > 0 && !this.expanded());
  hiddenCount = computed(() => this.collapsed().size);

  isHidden(i: number): boolean {
    return this.collapsed().has(i) && !this.expanded();
  }

  expand() { this.expanded.set(true); }

  trackByLabel(_: number, item: BreadcrumbItem) { return item.label; }

  ngAfterViewInit() {
    this.ro = new ResizeObserver(() => this.detectOverflow());
    this.ro.observe(this.olEl.nativeElement);
    this.detectOverflow();
  }

  private detectOverflow() {
    const ol = this.olEl.nativeElement;
    if (this.expanded()) return;

    // Reset first
    this.collapsed.set(new Set());
    this.cdr.detectChanges();

    requestAnimationFrame(() => {
      if (ol.scrollWidth > ol.clientWidth) {
        const middle = this.items
          .map((_, i) => i)
          .filter(i => i !== 0 && i !== this.items.length - 1);
        this.collapsed.set(new Set(middle));
        this.cdr.markForCheck();
      }
    });
  }

  ngOnDestroy() { this.ro?.disconnect(); }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| `<nav aria-label="Breadcrumb">` wraps the list | Identifies the landmark region |
| `<ol>` used (ordered list) | Reflects hierarchical, ordered nature of breadcrumb path |
| Last item has `aria-current="page"` | Screen readers announce "current page" |
| Hidden items have `aria-hidden="true"` | Collapsed items are not read by screen reader |
| Ellipsis button has descriptive `aria-label` | Announces how many steps are hidden |
| Ellipsis button has `aria-expanded` | Reflects current collapsed/expanded state |
| Separator is CSS `::after`, not DOM text | Decorative `/` not read aloud |
| Individual long labels have `text-overflow: ellipsis` | Prevents overflow of very long segment names |
| Focus is not disrupted on expand | Expanding adds items to DOM without removing focus |

---

## Production Pitfalls

**1. `text-overflow: ellipsis` does nothing on flex children**
`text-overflow` requires `overflow: hidden` and `white-space: nowrap` on the same element that holds the text — not a parent. A flex child with `min-width: 0` and no `overflow: hidden` will push other items out of view instead of truncating. Fix: apply `overflow: hidden; text-overflow: ellipsis; white-space: nowrap` directly to the `<a>` or `<span>`, not the `<li>`.

**2. ResizeObserver loop warning in console**
`ResizeObserver loop limit exceeded` is a benign browser warning that fires when your ResizeObserver callback itself triggers a resize. Fix: wrap the collapse logic in `requestAnimationFrame` so it runs after the current paint cycle.

**3. Container queries require `container-type` on the right ancestor**
Setting `container-type: inline-size` on the `<nav>` means the container is the nav element. If the nav is 100% of a narrow sidebar, the container query breakpoint fires correctly. But if it's set on `body` or a layout div, the container width doesn't match what you see. Fix: always set the container query on the nearest scrollable or layout-constraining ancestor.

**4. Screen readers read the expanded state incorrectly after expand**
When collapsed items have `aria-hidden="true"` and are then revealed by removing that attribute, some screen readers re-announce the entire list. Fix: use the `aria-live="polite"` region trick — place a visually hidden `<span aria-live="polite">` and update its text to "Full path now visible" on expand.

**5. Hard-coded `maxVisible` prop breaks on dynamic content**
If you pass `maxVisible={3}` but the labels are unusually long, those 3 items still overflow. Fix: rely on the `ResizeObserver`-based overflow detection rather than a static count.
