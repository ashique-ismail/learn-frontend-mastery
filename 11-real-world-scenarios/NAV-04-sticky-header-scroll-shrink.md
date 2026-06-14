# Sticky Header with Scroll-Aware Shrink

## The Idea

**In plain English:** When you first load a page, the header is tall — it has a large logo, padding, maybe a tagline. As you scroll down, the header stays at the top of the screen (sticky) but shrinks to a compact version to give you more reading space. Scroll back to the top, and it expands again. This is the "scroll shrink" or "compact header" pattern used on news sites, app homepages, and e-commerce stores.

**Real-world analogy:** Think of the retractable sun visor in a car.

- When the car is parked in the shade (top of page), the visor is fully down — large, blocking the whole windshield area.
- When you're driving in bright sun (scrolled down), you pull it up to a compact position that shields just your eyes but still lets you see the road — the header shrinks to show only the essentials: logo and nav.
- The visor doesn't disappear; it just becomes minimal. You can still reach it at any time.
- The transition between positions is smooth, not a jump — like the visor sliding up gradually.

The engineering challenge: reading scroll position efficiently without layout thrash, animating size and spacing via CSS custom properties (not JS DOM writes per frame), and using IntersectionObserver as a sentinel to detect when the user has scrolled past the hero section.

---

## Learning Objectives

- Distinguish `position: sticky` from `position: fixed`
- Use a CSS custom property (`--header-shrink`) as an animation driver
- Use IntersectionObserver (not scroll events) to toggle the compact class
- Avoid layout thrash: never read `scrollY` inside a `requestAnimationFrame` loop
- Transition header height, padding, and logo size via CSS transitions only
- Handle the edge case where the user loads the page already scrolled down

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Keep header at top of viewport | ✅ `position: sticky` | — |
| Detect that user has scrolled past hero | ❌ | CSS cannot observe scroll thresholds |
| Toggle compact class on scroll threshold | ❌ | Class toggling requires JS |
| Animate height, padding, font size | ✅ CSS `transition` on custom properties | — |
| Avoid scroll event performance issues | ❌ | JS scroll listener throttle/passive required |
| Restore full size when back at top | ❌ | Requires reading scroll state |

---

## HTML & CSS Foundation

```html
<!-- Sentinel element: when it leaves the viewport, header becomes compact -->
<div class="header-sentinel" aria-hidden="true"></div>

<header class="site-header" id="site-header">
  <div class="site-header__inner">

    <a href="/" class="site-header__logo" aria-label="Home">
      <img
        src="/logo.svg"
        alt="Acme Corp"
        class="site-header__logo-img"
        width="120"
        height="40"
      />
    </a>

    <nav class="site-header__nav" aria-label="Primary navigation">
      <ul role="list">
        <li><a href="/products">Products</a></li>
        <li><a href="/pricing">Pricing</a></li>
        <li><a href="/about">About</a></li>
      </ul>
    </nav>

    <a href="/get-started" class="site-header__cta">Get Started</a>

  </div>
</header>

<!-- Page content below -->
<main id="main">…</main>
```

```css
/* ─── Design tokens ─── */
:root {
  --header-height-full:    80px;
  --header-height-compact: 52px;
  --header-pad-full:       1.5rem;
  --header-pad-compact:    0.75rem;
  --header-logo-scale-full:    1;
  --header-logo-scale-compact: 0.75;
  --header-transition: 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  --header-bg: #ffffff;
  --header-shadow-full:    0 1px 0 rgba(0,0,0,0.08);
  --header-shadow-compact: 0 2px 12px rgba(0,0,0,0.12);
}

/* ─── Sentinel ─── */
/* A 1px tall invisible element at the very top of the page.
   When it exits the viewport, IntersectionObserver fires. */
.header-sentinel {
  height: 1px;
  width: 100%;
  position: absolute;
  top: 0;
  left: 0;
  pointer-events: none;
}

/* ─── Header base styles ─── */
.site-header {
  position: sticky;
  top: 0;
  z-index: 100;
  background: var(--header-bg);
  box-shadow: var(--header-shadow-full);
  transition:
    box-shadow var(--header-transition),
    background var(--header-transition);
  will-change: transform;  /* hint compositing layer */
}

/* ─── Inner layout ─── */
.site-header__inner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 2rem;
  height: var(--header-height-full);
  padding: 0 var(--header-pad-full);
  max-width: 1280px;
  margin: 0 auto;
  /* Transition height and padding via CSS, not JS */
  transition:
    height var(--header-transition),
    padding var(--header-transition);
}

/* ─── Logo ─── */
.site-header__logo-img {
  display: block;
  transform-origin: left center;
  transform: scale(var(--header-logo-scale-full));
  transition: transform var(--header-transition);
}

/* ─── Compact state — added by IntersectionObserver ─── */
.site-header.is-compact {
  box-shadow: var(--header-shadow-compact);
}

.site-header.is-compact .site-header__inner {
  height: var(--header-height-compact);
  padding: 0 var(--header-pad-compact);
}

.site-header.is-compact .site-header__logo-img {
  transform: scale(var(--header-logo-scale-compact));
}

/* ─── Nav links ─── */
.site-header__nav ul {
  display: flex;
  gap: 0.25rem;
  list-style: none;
  margin: 0;
  padding: 0;
}

.site-header__nav a {
  display: block;
  padding: 0.5rem 0.75rem;
  color: #374151;
  text-decoration: none;
  font-size: 0.9375rem;
  font-weight: 500;
  border-radius: 4px;
  transition: background var(--header-transition), color var(--header-transition);
}

.site-header__nav a:hover,
.site-header__nav a:focus-visible {
  background: #f3f4f6;
  color: #111827;
}

/* ─── CTA button ─── */
.site-header__cta {
  padding: 0.5rem 1.25rem;
  background: #2563eb;
  color: white;
  text-decoration: none;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  white-space: nowrap;
  transition: background var(--header-transition);
}

.site-header__cta:hover { background: #1d4ed8; }

/* ─── Reduced motion: no transitions ─── */
@media (prefers-reduced-motion: reduce) {
  .site-header,
  .site-header__inner,
  .site-header__logo-img {
    transition: none;
  }
}
```

---

## React Implementation

```tsx
// useStickyHeader.ts
import { useEffect, useRef, useState } from 'react';

/**
 * Returns whether the header should be in compact mode.
 * Uses IntersectionObserver on a sentinel element — zero scroll event cost.
 */
export function useStickyHeader() {
  const [isCompact, setIsCompact] = useState(false);
  const sentinelRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const sentinel = sentinelRef.current;
    if (!sentinel) return;

    // Check initial scroll position (page may have loaded mid-scroll)
    if (window.scrollY > 0) setIsCompact(true);

    const observer = new IntersectionObserver(
      ([entry]) => {
        // sentinel is NOT intersecting = user has scrolled away from top
        setIsCompact(!entry.isIntersecting);
      },
      {
        root: null,       // viewport
        threshold: 0,     // fire as soon as any part exits
      }
    );

    observer.observe(sentinel);
    return () => observer.disconnect();
  }, []);

  return { isCompact, sentinelRef };
}
```

```tsx
// SiteHeader.tsx
import { useRef } from 'react';
import { useStickyHeader } from './useStickyHeader';

interface NavItem { label: string; href: string; }

interface SiteHeaderProps {
  navItems: NavItem[];
  logoSrc: string;
  logoAlt: string;
  ctaLabel: string;
  ctaHref: string;
}

export function SiteHeader({
  navItems,
  logoSrc,
  logoAlt,
  ctaLabel,
  ctaHref,
}: SiteHeaderProps) {
  const { isCompact, sentinelRef } = useStickyHeader();

  return (
    <>
      {/* Sentinel: must be in normal document flow at the very top */}
      <div ref={sentinelRef} className="header-sentinel" aria-hidden="true" />

      <header
        className={`site-header ${isCompact ? 'is-compact' : ''}`}
        id="site-header"
      >
        <div className="site-header__inner">

          <a href="/" className="site-header__logo" aria-label="Home">
            <img
              src={logoSrc}
              alt={logoAlt}
              className="site-header__logo-img"
              width={120}
              height={40}
            />
          </a>

          <nav className="site-header__nav" aria-label="Primary navigation">
            <ul role="list">
              {navItems.map(item => (
                <li key={item.href}>
                  <a href={item.href}>{item.label}</a>
                </li>
              ))}
            </ul>
          </nav>

          <a href={ctaHref} className="site-header__cta">
            {ctaLabel}
          </a>

        </div>
      </header>
    </>
  );
}
```

---

## Angular Implementation

```typescript
// sticky-header.directive.ts
import {
  Directive, ElementRef, OnInit, OnDestroy,
  Renderer2, inject
} from '@angular/core';

/**
 * [appStickyHeader] directive — attaches to the <header> element.
 * Creates a sentinel div before the header and observes it.
 */
@Directive({
  selector: '[appStickyHeader]',
  standalone: true,
})
export class StickyHeaderDirective implements OnInit, OnDestroy {
  private el      = inject(ElementRef<HTMLElement>);
  private renderer = inject(Renderer2);
  private observer?: IntersectionObserver;

  ngOnInit() {
    // Create sentinel element before the header
    const sentinel = this.renderer.createElement('div') as HTMLElement;
    this.renderer.addClass(sentinel, 'header-sentinel');
    this.renderer.setAttribute(sentinel, 'aria-hidden', 'true');
    this.renderer.insertBefore(
      this.el.nativeElement.parentNode,
      sentinel,
      this.el.nativeElement
    );

    // Check initial scroll
    if (window.scrollY > 0) {
      this.renderer.addClass(this.el.nativeElement, 'is-compact');
    }

    this.observer = new IntersectionObserver(
      ([entry]) => {
        if (!entry.isIntersecting) {
          this.renderer.addClass(this.el.nativeElement, 'is-compact');
        } else {
          this.renderer.removeClass(this.el.nativeElement, 'is-compact');
        }
      },
      { root: null, threshold: 0 }
    );

    this.observer.observe(sentinel);
  }

  ngOnDestroy() {
    this.observer?.disconnect();
  }
}
```

```typescript
// site-header.component.ts
import { Component, Input } from '@angular/core';
import { NgFor } from '@angular/common';
import { StickyHeaderDirective } from './sticky-header.directive';

interface NavItem { label: string; href: string; }

@Component({
  selector: 'app-site-header',
  standalone: true,
  imports: [NgFor, StickyHeaderDirective],
  template: `
    <header appStickyHeader class="site-header" id="site-header">
      <div class="site-header__inner">

        <a href="/" class="site-header__logo" aria-label="Home">
          <img
            [src]="logoSrc"
            [alt]="logoAlt"
            class="site-header__logo-img"
            width="120"
            height="40"
          />
        </a>

        <nav class="site-header__nav" aria-label="Primary navigation">
          <ul role="list">
            <li *ngFor="let item of navItems">
              <a [href]="item.href">{{ item.label }}</a>
            </li>
          </ul>
        </nav>

        <a [href]="ctaHref" class="site-header__cta">{{ ctaLabel }}</a>

      </div>
    </header>
  `,
})
export class SiteHeaderComponent {
  @Input() navItems: NavItem[] = [];
  @Input() logoSrc  = '';
  @Input() logoAlt  = '';
  @Input() ctaLabel = '';
  @Input() ctaHref  = '';
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Header landmark present | `<header>` is implicitly `role="banner"` |
| Logo link has `aria-label="Home"` | Meaningful name for screen readers |
| Skip-to-content link above header | `<a href="#main" class="skip-link">Skip to content</a>` |
| Compact state does not remove nav items | Only visual size changes, all items remain in DOM and focusable |
| Sentinel has `aria-hidden="true"` | Decorative element invisible to screen readers |
| Nav has `aria-label="Primary navigation"` | Distinguishes from other nav landmarks |
| CTA link is meaningful text, not an icon | Descriptive text label; no `aria-label` needed |
| Transitions respect `prefers-reduced-motion` | `@media` query disables all transitions |
| Header `id="site-header"` | Allows skip link target and `aria-controls` references |

---

## Production Pitfalls

**1. `position: sticky` requires an explicit `top` value**
Without `top: 0`, the header scrolls away with the page even with `position: sticky`. This is a silent failure — the element is sticky but sticks at its original offset. Always set `top: 0`.

**2. Parent element with `overflow: hidden` breaks sticky**
`position: sticky` requires the scroll container to be a scrollable ancestor, not any ancestor with `overflow: hidden`. If a layout wrapper has `overflow: hidden`, sticky stops working. Fix: remove `overflow: hidden` from all ancestors, or use `overflow: clip` (which does not create a scroll container).

**3. Sentinel element inside the sticky header itself doesn't work**
The sentinel must be in the document's normal flow, above the sticky element — not inside it. If the sentinel is a child of the header, it scrolls with the header and the IntersectionObserver fires at the wrong time.

**4. Animating `height` triggers layout, causing jank**
Changing `height` causes reflow on every frame. Fix: instead of animating height directly, animate `padding-block` (top and bottom padding) and `font-size` on the logo. The header's height is determined by its content, and CSS transitions on padding avoid explicit height calculations.

**5. Page load with scroll position already non-zero**
If the user navigates via browser back button, the page restores scroll position, but the IntersectionObserver fires asynchronously. There is a brief flash where the full-size header shows before snapping to compact. Fix: check `window.scrollY > 0` synchronously during initialization and set `is-compact` before the first paint.

**6. iOS Safari `position: sticky` inside flex column**
On older iOS, `sticky` inside a `display: flex; flex-direction: column` container doesn't work. Fix: ensure the parent has an explicit height (e.g., `height: 100vh`) so the flex container is also the scroll container.
