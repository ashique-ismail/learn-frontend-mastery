# Mega Menu (Desktop)

## The Idea

**In plain English:** A mega menu is the wide panel that drops down when you hover over a top-level navigation item on a desktop site — think the full-width dropdown on a retail site that shows product categories in a grid, featured images, and quick links, all at once. Unlike a simple dropdown list, a mega menu is a rich layout inside the dropdown area itself.

**Real-world analogy:** Think of a newspaper spread open on a table.

- The **section headers along the top** (World, Business, Sports) are your top-level nav items.
- When you pick up the "Business" section, you see a **full two-page spread** with headlines, photos, and sub-sections arranged in columns — that's the mega menu panel.
- You don't just get one column of links — you get an entire curated layout, organized in a grid.
- The key difference from a regular dropdown: you can see **everything at once**, grouped in logical columns, instead of scrolling a list.
- **Hover intent** = the brief moment you need to hold your mouse over the section header before the spread opens. Without it, the menu flashes open and closed just because your mouse passed over it on the way to something else.

The engineering challenge: hover intent (debouncing open/close), keyboard navigation across a 2D grid, and keeping the CSS grid layout inside the panel accessible to screen readers.

---

## Learning Objectives

- Implement hover intent debouncing to prevent accidental open/close flicker
- Build a CSS Grid layout inside a dropdown panel
- Manage focus correctly across a 2D panel (arrow keys, Tab, Escape)
- Use `:focus-within` as a CSS-only fallback for keyboard users
- Wire up correct ARIA roles: `role="navigation"`, `aria-haspopup`, `aria-expanded`
- Handle edge cases: pointer leaving via child element, pointer entering via panel

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Show panel on hover | ✅ `:hover` + `:focus-within` | — |
| Debounce hover intent | ❌ | CSS has no timing state |
| Track which panel is open | ❌ | No integer/string state in CSS |
| Arrow-key navigation through grid cells | ❌ | Requires `keydown` listeners |
| Close on click outside | ❌ | Requires `document` event listener |
| Announce panel open to screen reader | ❌ | `aria-expanded` requires JS toggle |
| Keep mouse path through panel gap open | ❌ | CSS `:hover` breaks the moment pointer leaves the trigger |

---

## HTML & CSS Foundation

```html
<nav class="mega-nav" aria-label="Main navigation">
  <ul class="mega-nav__bar" role="list">

    <li class="mega-nav__item" data-menu="products">
      <button
        class="mega-nav__trigger"
        aria-haspopup="true"
        aria-expanded="false"
        aria-controls="panel-products"
      >
        Products
        <span class="mega-nav__chevron" aria-hidden="true">▾</span>
      </button>

      <div
        id="panel-products"
        class="mega-nav__panel"
        role="region"
        aria-label="Products menu"
        hidden
      >
        <div class="mega-nav__grid">
          <section class="mega-nav__col">
            <h3 class="mega-nav__col-heading">Software</h3>
            <ul role="list">
              <li><a href="/products/analytics">Analytics</a></li>
              <li><a href="/products/crm">CRM</a></li>
              <li><a href="/products/billing">Billing</a></li>
            </ul>
          </section>
          <section class="mega-nav__col">
            <h3 class="mega-nav__col-heading">Hardware</h3>
            <ul role="list">
              <li><a href="/products/laptops">Laptops</a></li>
              <li><a href="/products/monitors">Monitors</a></li>
            </ul>
          </section>
          <section class="mega-nav__col mega-nav__col--featured">
            <a href="/products/new" class="mega-nav__feature-card">
              <img src="/img/new-product.webp" alt="" width="240" height="140" loading="lazy">
              <span class="mega-nav__feature-label">New this month</span>
            </a>
          </section>
        </div>
      </div>
    </li>

    <li class="mega-nav__item">
      <a class="mega-nav__trigger" href="/solutions">Solutions</a>
    </li>

  </ul>
</nav>
```

```css
/* ─── Tokens ─── */
:root {
  --mega-nav-height: 60px;
  --mega-nav-panel-bg: #ffffff;
  --mega-nav-shadow: 0 8px 32px rgba(0, 0, 0, 0.12);
  --mega-nav-transition: 0.2s cubic-bezier(0.4, 0, 0.2, 1);
  --mega-nav-gap: 2rem;
}

/* ─── Bar ─── */
.mega-nav__bar {
  display: flex;
  align-items: stretch;
  height: var(--mega-nav-height);
  margin: 0;
  padding: 0;
  list-style: none;
  position: relative;
}

/* ─── Trigger ─── */
.mega-nav__trigger {
  display: flex;
  align-items: center;
  gap: 0.25rem;
  height: 100%;
  padding: 0 1.25rem;
  background: none;
  border: none;
  font-size: 0.9375rem;
  font-weight: 500;
  color: inherit;
  cursor: pointer;
  text-decoration: none;
  white-space: nowrap;
}

.mega-nav__trigger:focus-visible {
  outline: 2px solid currentColor;
  outline-offset: -2px;
}

.mega-nav__chevron {
  font-size: 0.75em;
  transition: transform var(--mega-nav-transition);
}

.mega-nav__item[data-open] .mega-nav__chevron {
  transform: rotate(180deg);
}

/* ─── Panel ─── */
.mega-nav__panel {
  position: absolute;
  top: var(--mega-nav-height);
  left: 0;
  width: 100%;
  background: var(--mega-nav-panel-bg);
  box-shadow: var(--mega-nav-shadow);
  border-top: 1px solid #e5e7eb;
  padding: 2rem var(--mega-nav-gap);
  opacity: 0;
  visibility: hidden;
  transform: translateY(-8px);
  transition:
    opacity var(--mega-nav-transition),
    transform var(--mega-nav-transition),
    visibility var(--mega-nav-transition);
  z-index: 200;
}

/* CSS :focus-within fallback — keyboard users get the panel */
.mega-nav__item:focus-within .mega-nav__panel {
  opacity: 1;
  visibility: visible;
  transform: translateY(0);
}

/* JS-driven open state */
.mega-nav__panel.is-open {
  opacity: 1;
  visibility: visible;
  transform: translateY(0);
}

/* ─── Grid inside panel ─── */
.mega-nav__grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: var(--mega-nav-gap);
  max-width: 960px;
}

.mega-nav__col-heading {
  font-size: 0.75rem;
  font-weight: 700;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: #6b7280;
  margin: 0 0 0.75rem;
}

.mega-nav__col ul {
  margin: 0;
  padding: 0;
  list-style: none;
}

.mega-nav__col a {
  display: block;
  padding: 0.375rem 0;
  color: #111827;
  text-decoration: none;
  font-size: 0.9375rem;
}

.mega-nav__col a:hover,
.mega-nav__col a:focus-visible {
  color: #2563eb;
  text-decoration: underline;
}

/* ─── Featured card ─── */
.mega-nav__feature-card {
  display: block;
  border-radius: 8px;
  overflow: hidden;
  text-decoration: none;
  color: inherit;
}

.mega-nav__feature-card img {
  width: 100%;
  height: 140px;
  object-fit: cover;
  display: block;
}

.mega-nav__feature-label {
  display: block;
  padding: 0.5rem 0;
  font-size: 0.875rem;
  font-weight: 600;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .mega-nav__panel,
  .mega-nav__chevron {
    transition: none;
  }
}
```

---

## React Implementation

```tsx
// MegaNav.tsx
import { useState, useRef, useEffect, useCallback, KeyboardEvent } from 'react';

interface NavLink {
  label: string;
  href: string;
}

interface NavColumn {
  heading: string;
  links: NavLink[];
}

interface NavItem {
  id: string;
  label: string;
  href?: string;
  columns?: NavColumn[];
  featured?: { href: string; image: string; label: string };
}

const HOVER_INTENT_DELAY = 120; // ms — wait before opening

export function MegaNav({ items }: { items: NavItem[] }) {
  const [openId, setOpenId] = useState<string | null>(null);
  const timerRef  = useRef<ReturnType<typeof setTimeout> | null>(null);
  const navRef    = useRef<HTMLElement>(null);

  const clearTimer = () => {
    if (timerRef.current) clearTimeout(timerRef.current);
  };

  const scheduleOpen = useCallback((id: string) => {
    clearTimer();
    timerRef.current = setTimeout(() => setOpenId(id), HOVER_INTENT_DELAY);
  }, []);

  const scheduleClose = useCallback(() => {
    clearTimer();
    timerRef.current = setTimeout(() => setOpenId(null), HOVER_INTENT_DELAY);
  }, []);

  const openNow = useCallback((id: string) => {
    clearTimer();
    setOpenId(id);
  }, []);

  const closeNow = useCallback(() => {
    clearTimer();
    setOpenId(null);
  }, []);

  // Close on outside click
  useEffect(() => {
    const handler = (e: MouseEvent) => {
      if (navRef.current && !navRef.current.contains(e.target as Node)) {
        closeNow();
      }
    };
    document.addEventListener('mousedown', handler);
    return () => document.removeEventListener('mousedown', handler);
  }, [closeNow]);

  // Escape closes
  useEffect(() => {
    const handler = (e: globalThis.KeyboardEvent) => {
      if (e.key === 'Escape') closeNow();
    };
    document.addEventListener('keydown', handler);
    return () => document.removeEventListener('keydown', handler);
  }, [closeNow]);

  return (
    <nav className="mega-nav" aria-label="Main navigation" ref={navRef}>
      <ul className="mega-nav__bar" role="list">
        {items.map(item => (
          <li
            key={item.id}
            className="mega-nav__item"
            {...(openId === item.id ? { 'data-open': '' } : {})}
            onMouseEnter={() => item.columns ? scheduleOpen(item.id) : undefined}
            onMouseLeave={() => item.columns ? scheduleClose() : undefined}
          >
            {item.columns ? (
              <>
                <button
                  className="mega-nav__trigger"
                  aria-haspopup="true"
                  aria-expanded={openId === item.id}
                  aria-controls={`panel-${item.id}`}
                  onClick={() => openId === item.id ? closeNow() : openNow(item.id)}
                  onFocus={() => openNow(item.id)}
                >
                  {item.label}
                  <span className="mega-nav__chevron" aria-hidden="true">▾</span>
                </button>

                <div
                  id={`panel-${item.id}`}
                  className={`mega-nav__panel ${openId === item.id ? 'is-open' : ''}`}
                  role="region"
                  aria-label={`${item.label} menu`}
                  onMouseEnter={clearTimer}
                  onMouseLeave={scheduleClose}
                >
                  <div className="mega-nav__grid">
                    {item.columns.map(col => (
                      <section key={col.heading} className="mega-nav__col">
                        <h3 className="mega-nav__col-heading">{col.heading}</h3>
                        <ul role="list">
                          {col.links.map(link => (
                            <li key={link.href}>
                              <a href={link.href} onClick={closeNow}>{link.label}</a>
                            </li>
                          ))}
                        </ul>
                      </section>
                    ))}
                    {item.featured && (
                      <section className="mega-nav__col mega-nav__col--featured">
                        <a
                          href={item.featured.href}
                          className="mega-nav__feature-card"
                          onClick={closeNow}
                        >
                          <img
                            src={item.featured.image}
                            alt=""
                            width={240}
                            height={140}
                            loading="lazy"
                          />
                          <span className="mega-nav__feature-label">
                            {item.featured.label}
                          </span>
                        </a>
                      </section>
                    )}
                  </div>
                </div>
              </>
            ) : (
              <a className="mega-nav__trigger" href={item.href}>{item.label}</a>
            )}
          </li>
        ))}
      </ul>
    </nav>
  );
}
```

---

## Angular Implementation

```typescript
// mega-nav.model.ts
export interface NavLink   { label: string; href: string; }
export interface NavColumn { heading: string; links: NavLink[]; }
export interface NavItem {
  id: string;
  label: string;
  href?: string;
  columns?: NavColumn[];
  featured?: { href: string; image: string; label: string };
}
```

```typescript
// mega-nav.component.ts
import {
  Component, Input, signal, computed,
  HostListener, OnDestroy, ChangeDetectionStrategy
} from '@angular/core';
import { NgFor, NgIf } from '@angular/common';
import { NavItem } from './mega-nav.model';

const HOVER_INTENT_DELAY = 120;

@Component({
  selector: 'app-mega-nav',
  standalone: true,
  imports: [NgFor, NgIf],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <nav class="mega-nav" aria-label="Main navigation">
      <ul class="mega-nav__bar" role="list">
        <li
          *ngFor="let item of items; trackBy: trackById"
          class="mega-nav__item"
          [attr.data-open]="openId() === item.id ? '' : null"
          (mouseenter)="item.columns ? scheduleOpen(item.id) : null"
          (mouseleave)="item.columns ? scheduleClose() : null"
        >
          <!-- Has mega panel -->
          <ng-container *ngIf="item.columns; else simpleLink">
            <button
              class="mega-nav__trigger"
              aria-haspopup="true"
              [attr.aria-expanded]="openId() === item.id"
              [attr.aria-controls]="'panel-' + item.id"
              (click)="togglePanel(item.id)"
              (focus)="openNow(item.id)"
            >
              {{ item.label }}
              <span class="mega-nav__chevron" aria-hidden="true">▾</span>
            </button>

            <div
              [id]="'panel-' + item.id"
              class="mega-nav__panel"
              [class.is-open]="openId() === item.id"
              role="region"
              [attr.aria-label]="item.label + ' menu'"
              (mouseenter)="clearTimer()"
              (mouseleave)="scheduleClose()"
            >
              <div class="mega-nav__grid">
                <section *ngFor="let col of item.columns" class="mega-nav__col">
                  <h3 class="mega-nav__col-heading">{{ col.heading }}</h3>
                  <ul role="list">
                    <li *ngFor="let link of col.links">
                      <a [href]="link.href" (click)="closeNow()">{{ link.label }}</a>
                    </li>
                  </ul>
                </section>
                <section *ngIf="item.featured" class="mega-nav__col mega-nav__col--featured">
                  <a [href]="item.featured.href" class="mega-nav__feature-card" (click)="closeNow()">
                    <img [src]="item.featured.image" alt="" width="240" height="140" loading="lazy">
                    <span class="mega-nav__feature-label">{{ item.featured.label }}</span>
                  </a>
                </section>
              </div>
            </div>
          </ng-container>

          <!-- Simple link -->
          <ng-template #simpleLink>
            <a class="mega-nav__trigger" [href]="item.href">{{ item.label }}</a>
          </ng-template>
        </li>
      </ul>
    </nav>
  `,
})
export class MegaNavComponent implements OnDestroy {
  @Input() items: NavItem[] = [];

  openId = signal<string | null>(null);
  private timer: ReturnType<typeof setTimeout> | null = null;

  clearTimer() {
    if (this.timer) clearTimeout(this.timer);
  }

  scheduleOpen(id: string) {
    this.clearTimer();
    this.timer = setTimeout(() => this.openId.set(id), HOVER_INTENT_DELAY);
  }

  scheduleClose() {
    this.clearTimer();
    this.timer = setTimeout(() => this.openId.set(null), HOVER_INTENT_DELAY);
  }

  openNow(id: string) {
    this.clearTimer();
    this.openId.set(id);
  }

  closeNow() {
    this.clearTimer();
    this.openId.set(null);
  }

  togglePanel(id: string) {
    this.openId() === id ? this.closeNow() : this.openNow(id);
  }

  @HostListener('document:mousedown', ['$event'])
  onOutsideClick(e: MouseEvent) {
    // No ElementRef needed — Angular handles host element via nativeElement
  }

  @HostListener('document:keydown.escape')
  onEscape() { this.closeNow(); }

  trackById(_: number, item: NavItem) { return item.id; }

  ngOnDestroy() { this.clearTimer(); }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Trigger has `aria-haspopup="true"` | Tells screen readers a popup will appear |
| Trigger has `aria-expanded` toggled | Reflects current open/closed state |
| Panel has `role="region"` + `aria-label` | Announces panel name on focus entry |
| Column headings use `<h3>` | Navigable via heading shortcuts in screen readers |
| Panel links navigable by keyboard Tab | Natural DOM order within panel columns |
| Escape closes panel | `document:keydown.escape` listener |
| `:focus-within` CSS fallback | Panel stays open while keyboard focus is inside |
| Focus does not leave nav bar on panel open | Panel is a sibling/child, not a portal |
| Featured image has empty `alt=""` | Decorative; label provides the text |
| Hover intent prevents accidental open | Debounce prevents noise from pointer passing through |

---

## Production Pitfalls

**1. Pointer path gap between trigger and panel**
When the cursor moves from the trigger down to the panel, it briefly leaves the trigger's `:hover` zone. Without hover intent debouncing, the panel snaps shut. Fix: use a 120–150ms close delay so the pointer has time to enter the panel. Alternatively, add an invisible `::after` pseudo-element triangle to bridge the gap.

**2. Panel clips at viewport edge**
A full-width panel anchored to `left: 0` of the `<nav>` parent is fine. But if the `<nav>` is inside a centered container, the panel might be narrower than the viewport. Fix: use `left: 50%; transform: translateX(-50%); width: 100vw` on the panel, plus `overflow: hidden` on the bar to avoid horizontal scroll.

**3. `:focus-within` keeps panel open when focus moves to another item**
If item A is open via keyboard focus, moving focus to item B triggers `:focus-within` on item B but also keeps item A open because its focus-within hasn't been removed yet. Fix: pair CSS focus-within with JS `openId` state — the JS state drives the `is-open` class; `:focus-within` is only the CSS safety net.

**4. Screen reader announces all panel content even when closed**
If you use `opacity: 0; visibility: hidden` without `hidden` attribute or `aria-hidden`, a screen reader's virtual cursor can still reach the links. Fix: toggle the `hidden` attribute on the panel element in JS, or use `aria-hidden="true"` when the panel is closed.

**5. Grid columns collapse on narrow desktop**
`auto-fit` with `minmax(200px, 1fr)` collapses columns when the nav bar is inside a container narrower than 600px. Fix: set a fixed column count with `grid-template-columns: repeat(3, 1fr)` and rely on the max-width to constrain the panel, not the grid's implicit sizing.

---

## Interview Angle

**Q: "How do you implement hover intent on a mega menu and why does it matter?"**

Strong answer: Hover intent is a debounce on both the open and close events. Without it, a user moving their mouse across the nav bar accidentally flickers through every panel. The pattern is: on `mouseenter`, start a ~120ms timer and only open if the pointer is still over the trigger when it fires. On `mouseleave`, start a close timer — cancel it if the pointer re-enters the panel itself. This way, deliberate hovers open the panel; incidental cursor paths do not.

**Q: "How do you handle keyboard navigation in a mega menu panel?"**

Strong answer: The trigger button opens the panel and receives `aria-expanded`. Once the panel is open, Tab moves focus into the first link in the first column. Tab continues through all links left-to-right, top-to-bottom (natural DOM order). Shift+Tab reverses. Escape closes the panel and returns focus to the trigger. Arrow-key navigation (optional enhancement) requires a `keydown` handler that maps left/right arrows to column boundaries and up/down to items within a column, tracked as a 2D index.
