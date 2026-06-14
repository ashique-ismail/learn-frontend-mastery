# Sticky Sidebar with Scroll Sync

## The Idea

**In plain English:** You have a long-form page — a documentation article, a product page, or a legal document — with a sidebar listing section headings. The sidebar sticks to the viewport as you scroll. As each section enters the viewport, the corresponding sidebar link highlights. Clicking a link smoothly scrolls the page to that section, but accounts for a fixed header so the section heading isn't hidden behind it.

**Real-world analogy:** Think of a tour guide walking alongside you through a museum gallery.

- The **museum map in the guide's hand** = the sidebar. It stays with you the whole time regardless of how far you walk.
- **Each room you enter** = a section becoming active as you scroll into it. The guide points to the current room on the map without you doing anything.
- **Clicking a room on the map** = the guide walks you directly to that room, stopping you far enough from the doorway so you can actually see the room's name sign — not hidden behind the door frame (the fixed header).
- **The door frame** = the fixed header. Without accounting for it, you would stop exactly at the threshold and the room name would be obscured.
- **The guide's total height from floor to eye level** = the calculated `top` offset on the sticky sidebar. The guide must know how tall they are relative to the wall to stand at the right spot.

The key insight: "sticky" positioning solves the visual anchoring; `IntersectionObserver` solves the active-state detection; a calculated scroll offset solves the header-obscuring problem. All three are required, and each solves a distinct problem CSS alone cannot unify.

---

## Learning Objectives

- Understand why `position: sticky` requires a calculated `top` value to sit below a fixed header
- Know when `IntersectionObserver` is the right tool versus `scroll` event listeners
- Build active-link highlighting driven purely by viewport observation rather than scroll math
- Implement smooth-scroll-to-section with a dynamic header offset in both frameworks
- Handle edge cases: last section, page resize changing header height, sections taller than the viewport
- Write accessible sidebar navigation with correct ARIA roles and keyboard support

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Stick sidebar to viewport while scrolling | ✅ `position: sticky; top: Xpx` | — |
| Offset the sticky top value to clear the fixed header | ⚠️ Only if header height is a fixed known value baked into CSS | Breaks when header height is dynamic (responsive, announcement banners, CLS) |
| Detect which section is currently in the viewport | ❌ | CSS has no viewport intersection state |
| Highlight the corresponding sidebar link | ❌ | Depends on detected active section — requires JS state |
| Smooth-scroll to a section on sidebar click | ⚠️ `scroll-behavior: smooth` on `html` | Scrolls to the element's top edge, ignoring the fixed header offset |
| Apply a per-click header offset to `scrollTo` | ❌ | CSS cannot compute `element.getBoundingClientRect() - headerHeight` |
| Update sidebar highlight on programmatic scroll | ❌ | CSS has no event model |

**Conclusion:** CSS owns `position: sticky`, scroll behavior defaults, and all visual styling. JS owns the header-height measurement, the `IntersectionObserver` setup, the active-state signal, and the offset-corrected scroll-to logic.

---

## HTML & CSS Foundation

### The Structure

```html
<!-- Fixed header (creates the offset problem) -->
<header class="site-header" id="site-header">
  <a href="/" class="logo">Brand</a>
  <nav aria-label="Global"><!-- ... --></nav>
</header>

<!-- Page layout: content + sidebar -->
<div class="page-layout">

  <!-- Main content: each <section> is a scroll target -->
  <main class="page-content" id="main-content">
    <section id="introduction" class="content-section">
      <h2>Introduction</h2>
      <p>...</p>
    </section>

    <section id="installation" class="content-section">
      <h2>Installation</h2>
      <p>...</p>
    </section>

    <section id="configuration" class="content-section">
      <h2>Configuration</h2>
      <p>...</p>
    </section>

    <section id="api-reference" class="content-section">
      <h2>API Reference</h2>
      <p>...</p>
    </section>
  </main>

  <!-- Sidebar: sticky navigation -->
  <aside class="page-sidebar" aria-label="On this page">
    <nav aria-label="Table of contents">
      <p class="sidebar-heading" aria-hidden="true">On this page</p>
      <ul class="toc-list" role="list">
        <li>
          <a class="toc-link" href="#introduction" aria-current="false">
            Introduction
          </a>
        </li>
        <li>
          <a class="toc-link" href="#installation" aria-current="false">
            Installation
          </a>
        </li>
        <li>
          <a class="toc-link" href="#configuration" aria-current="false">
            Configuration
          </a>
        </li>
        <li>
          <a class="toc-link" href="#api-reference" aria-current="false">
            API Reference
          </a>
        </li>
      </ul>
    </nav>
  </aside>

</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --header-height: 64px;       /* must match the actual rendered header height */
  --sidebar-width: 240px;
  --sidebar-gap: 2rem;
  --toc-active-color: #0066cc;
  --toc-active-bg: #f0f7ff;
  --toc-transition: 0.15s ease;
  --content-max-width: 800px;
}

/* ─── Fixed header ─── */
.site-header {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  height: var(--header-height);
  background: #fff;
  border-bottom: 1px solid #e5e7eb;
  display: flex;
  align-items: center;
  padding: 0 1.5rem;
  z-index: 100;
}

/* Push page content below the fixed header */
body {
  padding-top: var(--header-height);
}

/* ─── Page layout ─── */
.page-layout {
  display: grid;
  grid-template-columns: 1fr var(--sidebar-width);
  grid-template-areas: "content sidebar";
  gap: var(--sidebar-gap);
  max-width: calc(var(--content-max-width) + var(--sidebar-width) + var(--sidebar-gap));
  margin: 0 auto;
  padding: 2rem 1.5rem;
  align-items: start;  /* critical: prevents sidebar from stretching to content height */
}

.page-content  { grid-area: content; }
.page-sidebar  { grid-area: sidebar; }

/* ─── Sticky sidebar ─── */
/*
  top must equal the fixed header height plus any desired breathing room.
  The value in CSS assumes a static header height. For dynamic heights,
  JS reads the actual rendered height and sets --header-height on :root.
*/
.page-sidebar {
  position: sticky;
  top: calc(var(--header-height) + 1.5rem);  /* 1.5rem breathing room below header */
  max-height: calc(100vh - var(--header-height) - 3rem);
  overflow-y: auto;
}

/* ─── TOC list ─── */
.sidebar-heading {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  color: #6b7280;
  margin: 0 0 0.75rem;
  padding: 0 0.75rem;
}

.toc-list {
  list-style: none;
  margin: 0;
  padding: 0;
}

.toc-link {
  display: block;
  padding: 0.375rem 0.75rem;
  border-radius: 6px;
  font-size: 0.875rem;
  color: #374151;
  text-decoration: none;
  transition: color var(--toc-transition), background-color var(--toc-transition);
  border-left: 2px solid transparent;
}

.toc-link:hover {
  color: var(--toc-active-color);
  background-color: var(--toc-active-bg);
}

/* Active state: CSS class added by JS */
.toc-link.is-active,
.toc-link[aria-current="true"] {
  color: var(--toc-active-color);
  background-color: var(--toc-active-bg);
  border-left-color: var(--toc-active-color);
  font-weight: 500;
}

/* ─── Content sections ─── */
.content-section {
  margin-bottom: 4rem;
  scroll-margin-top: calc(var(--header-height) + 1.5rem);
  /*
    scroll-margin-top is the correct CSS mechanism for offsetting
    scroll-into-view targets below fixed headers.
    It works with anchor clicks AND programmatic scrollIntoView().
  */
}

/* ─── Responsive: collapse sidebar on narrow screens ─── */
@media (max-width: 900px) {
  .page-layout {
    grid-template-columns: 1fr;
    grid-template-areas:
      "sidebar"
      "content";
  }

  .page-sidebar {
    position: static;        /* sticky doesn't make sense in a stacked layout */
    max-height: none;
    overflow-y: visible;
  }

  .toc-list {
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem;
  }

  .toc-link {
    border-left: none;
    border-bottom: 2px solid transparent;
    border-radius: 4px;
  }

  .toc-link.is-active,
  .toc-link[aria-current="true"] {
    border-left-color: transparent;
    border-bottom-color: var(--toc-active-color);
  }
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  html {
    scroll-behavior: auto;
  }
}
```

**What CSS owns:** sticky positioning, active-link styling, `scroll-margin-top` offset for the fixed header (native CSS fix), responsive layout switch, reduced motion opt-out.

**What CSS cannot own:** reading the actual rendered header height, detecting which section is in the viewport, toggling the `is-active` class, and performing offset-corrected programmatic scrolling for browsers that do not support `scroll-margin-top` uniformly.

---

## React Implementation

### The Hook — Scroll Sync

```tsx
// useScrollSpy.ts
import { useState, useEffect, useRef, useCallback } from 'react';

interface ScrollSpyOptions {
  /** Selector for the fixed header — its height is read at runtime */
  headerSelector?: string;
  /** Extra pixels of breathing room below the header */
  rootMarginExtra?: number;
}

export function useScrollSpy(
  sectionIds: string[],
  options: ScrollSpyOptions = {}
) {
  const { headerSelector = '#site-header', rootMarginExtra = 24 } = options;

  const [activeId, setActiveId] = useState<string>(sectionIds[0] ?? '');
  const [headerHeight, setHeaderHeight] = useState(0);
  const observerRef = useRef<IntersectionObserver | null>(null);

  // Measure header height and keep it updated on resize
  useEffect(() => {
    const header = document.querySelector<HTMLElement>(headerSelector);
    if (!header) return;

    const measure = () => {
      setHeaderHeight(header.getBoundingClientRect().height);
      // Keep the CSS custom property in sync so sticky top is always correct
      document.documentElement.style.setProperty(
        '--header-height',
        `${header.getBoundingClientRect().height}px`
      );
    };

    measure();

    const ro = new ResizeObserver(measure);
    ro.observe(header);
    return () => ro.disconnect();
  }, [headerSelector]);

  // Set up IntersectionObserver once header height is known
  useEffect(() => {
    if (headerHeight === 0) return;

    observerRef.current?.disconnect();

    /*
      rootMargin top is negative so the intersection threshold starts
      below the sticky header. This means a section is only considered
      "active" when its top edge is below the header — exactly what the
      user can actually read.
    */
    const topOffset = -(headerHeight + rootMarginExtra);

    observerRef.current = new IntersectionObserver(
      (entries) => {
        /*
          Find the topmost intersecting entry (the one closest to the
          top of the viewport). Using a single "last intersected" pattern
          is unreliable for fast scrolling; iterating all entries is safer.
        */
        const visible = entries
          .filter(e => e.isIntersecting)
          .sort((a, b) => a.boundingClientRect.top - b.boundingClientRect.top);

        if (visible.length > 0) {
          setActiveId(visible[0].target.id);
        }
      },
      {
        rootMargin: `${topOffset}px 0px 0px 0px`,
        threshold: 0,
      }
    );

    sectionIds.forEach(id => {
      const el = document.getElementById(id);
      if (el) observerRef.current!.observe(el);
    });

    return () => observerRef.current?.disconnect();
  }, [sectionIds, headerHeight, rootMarginExtra]);

  // Smooth scroll to a section with header offset correction
  const scrollToSection = useCallback(
    (id: string, event?: React.MouseEvent) => {
      event?.preventDefault();

      const target = document.getElementById(id);
      if (!target) return;

      /*
        scroll-margin-top handles most cases natively.
        We also do a manual scrollTo as a fallback and for fine control.
      */
      const top =
        target.getBoundingClientRect().top +
        window.scrollY -
        headerHeight -
        rootMarginExtra;

      window.scrollTo({ top, behavior: 'smooth' });
      setActiveId(id); // optimistic update — observer will confirm
    },
    [headerHeight, rootMarginExtra]
  );

  return { activeId, scrollToSection, headerHeight };
}
```

### The Components

```tsx
// TableOfContents.tsx
import { useScrollSpy } from './useScrollSpy';

export interface TocSection {
  id: string;
  label: string;
}

interface Props {
  sections: TocSection[];
}

export function TableOfContents({ sections }: Props) {
  const sectionIds = sections.map(s => s.id);
  const { activeId, scrollToSection } = useScrollSpy(sectionIds);

  return (
    <aside className="page-sidebar" aria-label="On this page">
      <nav aria-label="Table of contents">
        <p className="sidebar-heading" aria-hidden="true">On this page</p>
        <ul className="toc-list" role="list">
          {sections.map(section => {
            const isActive = section.id === activeId;
            return (
              <li key={section.id}>
                <a
                  href={`#${section.id}`}
                  className={`toc-link ${isActive ? 'is-active' : ''}`}
                  aria-current={isActive ? 'true' : undefined}
                  onClick={e => scrollToSection(section.id, e)}
                >
                  {section.label}
                </a>
              </li>
            );
          })}
        </ul>
      </nav>
    </aside>
  );
}
```

```tsx
// DocumentPage.tsx — the full page composition
import { TableOfContents, type TocSection } from './TableOfContents';

const SECTIONS: TocSection[] = [
  { id: 'introduction',  label: 'Introduction'  },
  { id: 'installation',  label: 'Installation'  },
  { id: 'configuration', label: 'Configuration' },
  { id: 'api-reference', label: 'API Reference' },
];

export function DocumentPage() {
  return (
    <>
      <header className="site-header" id="site-header">
        <a href="/" className="logo">Brand</a>
      </header>

      <div className="page-layout">
        <main className="page-content" id="main-content">
          <section id="introduction" className="content-section">
            <h2>Introduction</h2>
            <p>Long content...</p>
          </section>

          <section id="installation" className="content-section">
            <h2>Installation</h2>
            <p>Long content...</p>
          </section>

          <section id="configuration" className="content-section">
            <h2>Configuration</h2>
            <p>Long content...</p>
          </section>

          <section id="api-reference" className="content-section">
            <h2>API Reference</h2>
            <p>Long content...</p>
          </section>
        </main>

        <TableOfContents sections={SECTIONS} />
      </div>
    </>
  );
}
```

---

## Angular Implementation

### Service — Scroll Spy State

```typescript
// scroll-spy.service.ts
import { Injectable, signal, computed, effect, OnDestroy } from '@angular/core';

export interface TocSection {
  id: string;
  label: string;
}

@Injectable({ providedIn: 'root' })
export class ScrollSpyService implements OnDestroy {
  private _activeId   = signal<string>('');
  private _headerHeight = signal<number>(0);

  readonly activeId     = this._activeId.asReadonly();
  readonly headerHeight = this._headerHeight.asReadonly();

  private observer: IntersectionObserver | null = null;
  private headerObserver: ResizeObserver | null = null;

  init(sectionIds: string[], headerSelector = '#site-header') {
    this.measureHeader(headerSelector);

    // Wait until header height is measured before observing sections
    const waitForHeight = () => {
      const h = this._headerHeight();
      if (h === 0) {
        requestAnimationFrame(waitForHeight);
        return;
      }
      this.observeSections(sectionIds, h);
    };
    waitForHeight();
  }

  private measureHeader(selector: string) {
    const header = document.querySelector<HTMLElement>(selector);
    if (!header) return;

    const measure = () => {
      const h = header.getBoundingClientRect().height;
      this._headerHeight.set(h);
      document.documentElement.style.setProperty('--header-height', `${h}px`);
    };

    measure();
    this.headerObserver = new ResizeObserver(measure);
    this.headerObserver.observe(header);
  }

  private observeSections(ids: string[], headerHeight: number) {
    this.observer?.disconnect();

    const topOffset = -(headerHeight + 24);

    this.observer = new IntersectionObserver(
      (entries) => {
        const visible = entries
          .filter(e => e.isIntersecting)
          .sort((a, b) => a.boundingClientRect.top - b.boundingClientRect.top);

        if (visible.length > 0) {
          this._activeId.set(visible[0].target.id);
        }
      },
      {
        rootMargin: `${topOffset}px 0px 0px 0px`,
        threshold: 0,
      }
    );

    ids.forEach(id => {
      const el = document.getElementById(id);
      if (el) this.observer!.observe(el);
    });
  }

  scrollToSection(id: string): void {
    const target = document.getElementById(id);
    if (!target) return;

    const top =
      target.getBoundingClientRect().top +
      window.scrollY -
      this._headerHeight() -
      24;

    window.scrollTo({ top, behavior: 'smooth' });
    this._activeId.set(id); // optimistic
  }

  ngOnDestroy() {
    this.observer?.disconnect();
    this.headerObserver?.disconnect();
  }
}
```

### Table of Contents Component

```typescript
// toc.component.ts
import {
  Component, OnInit, OnDestroy, Input, inject
} from '@angular/core';
import { NgFor } from '@angular/common';
import { ScrollSpyService, TocSection } from './scroll-spy.service';

@Component({
  selector: 'app-toc',
  standalone: true,
  imports: [NgFor],
  template: `
    <aside class="page-sidebar" aria-label="On this page">
      <nav aria-label="Table of contents">
        <p class="sidebar-heading" aria-hidden="true">On this page</p>
        <ul class="toc-list" role="list">
          <li *ngFor="let section of sections; trackBy: trackById">
            <a
              [href]="'#' + section.id"
              class="toc-link"
              [class.is-active]="scrollSpy.activeId() === section.id"
              [attr.aria-current]="scrollSpy.activeId() === section.id ? 'true' : null"
              (click)="onLinkClick($event, section.id)"
            >
              {{ section.label }}
            </a>
          </li>
        </ul>
      </nav>
    </aside>
  `,
})
export class TocComponent implements OnInit {
  @Input({ required: true }) sections: TocSection[] = [];

  scrollSpy = inject(ScrollSpyService);

  ngOnInit() {
    this.scrollSpy.init(this.sections.map(s => s.id));
  }

  onLinkClick(event: MouseEvent, id: string) {
    event.preventDefault();
    this.scrollSpy.scrollToSection(id);
  }

  trackById(_: number, section: TocSection) {
    return section.id;
  }
}
```

### Document Page Component

```typescript
// document-page.component.ts
import { Component } from '@angular/core';
import { TocComponent, type TocSection } from './toc.component';

const SECTIONS: TocSection[] = [
  { id: 'introduction',  label: 'Introduction'  },
  { id: 'installation',  label: 'Installation'  },
  { id: 'configuration', label: 'Configuration' },
  { id: 'api-reference', label: 'API Reference' },
];

@Component({
  selector: 'app-document-page',
  standalone: true,
  imports: [TocComponent],
  template: `
    <header class="site-header" id="site-header">
      <a href="/" class="logo">Brand</a>
    </header>

    <div class="page-layout">
      <main class="page-content" id="main-content">
        <section id="introduction" class="content-section">
          <h2>Introduction</h2>
          <p>Long content...</p>
        </section>

        <section id="installation" class="content-section">
          <h2>Installation</h2>
          <p>Long content...</p>
        </section>

        <section id="configuration" class="content-section">
          <h2>Configuration</h2>
          <p>Long content...</p>
        </section>

        <section id="api-reference" class="content-section">
          <h2>API Reference</h2>
          <p>Long content...</p>
        </section>
      </main>

      <app-toc [sections]="sections" />
    </div>
  `,
})
export class DocumentPageComponent {
  sections = SECTIONS;
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Sidebar landmark has an accessible name | `<aside aria-label="On this page">` |
| TOC navigation has an accessible name | `<nav aria-label="Table of contents">` |
| Active link is marked for screen readers | `aria-current="true"` on the active `<a>` (not just a visual class) |
| Links are real anchor elements | `href="#section-id"` — degrades gracefully if JS fails; native fragment nav works |
| Sticky sidebar does not obscure focused content | `scroll-margin-top` on every section guarantees the section heading clears the header when focused via keyboard or scroll-into-view |
| Keyboard navigation works without JS | Native anchor links scroll the page; `scroll-margin-top` handles header offset |
| Header height changes are handled | `ResizeObserver` on the header keeps `--header-height` and `top` offset in sync |
| Reduced motion is respected | `@media (prefers-reduced-motion: reduce)` removes `scroll-behavior: smooth`; observer still updates active link |
| Sidebar heading is not in the tab order | `aria-hidden="true"` on the decorative "On this page" label |
| Overflow sidebar is scrollable | `overflow-y: auto` on `.page-sidebar` ensures very long TOCs are reachable |

---

## Production Pitfalls

**1. `position: sticky` silently stops working when `align-items: stretch` is the grid default**
The stickiest gotcha: if the sidebar's grid parent uses the default `align-items: stretch`, the sidebar's grid cell grows to match the content column's full height. A sticky element cannot scroll within a container that is exactly as tall as the element itself — there is nothing to scroll relative to. Fix: set `align-items: start` on the grid container. This is easily forgotten and causes hours of confusion because sticky "looks fine" on short pages.

**2. `IntersectionObserver` picks the wrong active section when scrolling fast**
If you only react to entries where `isIntersecting` transitions from `false → true`, fast scrolling can skip entries entirely, leaving the sidebar stale. Fix: on each callback, collect all currently intersecting entries from the list, sort by `boundingClientRect.top`, and take the topmost one. Also call `observe()` on existing entries after each re-init to pick up the current state.

**3. The last section can never become active on short pages**
If the last section is short and the page does not scroll far enough to bring it fully into the observer's intersection zone, it never activates. Fix: observe the sections with `threshold: 0` so any pixel of the section entering the adjusted root margin triggers activation. Additionally, add a scroll-end check: if `window.scrollY + window.innerHeight >= document.body.scrollHeight - 10`, force-activate the last section ID.

**4. Header height is wrong on first render (layout shift, banners)**
If a cookie consent banner or announcement bar renders above the fixed header, the measured height at component mount is wrong. Fix: use `ResizeObserver` on the header element and re-synchronize `--header-height` and the observer's `rootMargin` on every size change. Disconnect the old `IntersectionObserver` and create a fresh one with the updated margin.

**5. Programmatic scroll and the observer race each other**
When the user clicks a TOC link, you set `activeId` optimistically and call `scrollTo`. During the scroll animation, the observer fires repeatedly and may flip `activeId` to an intermediate section, causing the sidebar to flicker. Fix: use a `isScrollingProgrammatically` ref/flag, set it to `true` on click, and ignore observer updates while it is set. Clear the flag in a `scrollend` event listener (or a debounced `scroll` fallback):
```ts
let programmatic = false;
function scrollToSection(id) {
  programmatic = true;
  window.scrollTo({ top, behavior: 'smooth' });
  window.addEventListener('scrollend', () => { programmatic = false; }, { once: true });
}
// In the observer callback:
if (!programmatic) setActiveId(topVisibleEntry.target.id);
```

**6. `scroll-margin-top` is not enough for programmatic scrolls in all browsers**
`scroll-margin-top` is well-supported for `scrollIntoView()` and native anchor clicks. It is NOT respected by `window.scrollTo({ top })`. Always compute the manual offset (`element.getBoundingClientRect().top + window.scrollY - headerHeight`) when calling `scrollTo` directly, and use `scroll-margin-top` as the CSS layer for native anchor behaviour and keyboard focus.

---

## Interview Angle

**Q: "How would you implement a sticky sidebar that highlights the current section as the user scrolls?"**

Strong answers cover four distinct layers:
1. **CSS sticky positioning** — `position: sticky; top: calc(var(--header-height) + gap)` on the sidebar, with `align-items: start` on the grid parent (the most commonly forgotten requirement). Use `scroll-margin-top` on each section so native anchor clicks and keyboard focus already respect the header height without JS.
2. **IntersectionObserver for active detection** — explain why this is preferable to `scroll` event listeners: no main-thread thrashing, no manual `getBoundingClientRect` in a hot event loop. Describe the `rootMargin` trick: a negative top margin equal to the header height shifts the intersection "viewport" down so sections trigger only when they are actually readable, not hidden behind the header.
3. **Offset-corrected programmatic scroll** — `scroll-behavior: smooth` on `html` handles the animation; `window.scrollTo({ top: element.top + scrollY - headerHeight })` handles the offset. Mention the `scrollend` + programmatic-flag pattern to prevent observer flicker during the animation.
4. **Dynamic header height** — `ResizeObserver` on the header keeps everything synchronized when responsive breakpoints, cookie banners, or content changes alter the header's rendered height. Hardcoding `64px` in CSS works only in demos.

**Follow-up: "What breaks when the last section is shorter than the viewport?"**

The last section may never fully enter the observer zone because the page runs out of scroll distance before the section crosses the `rootMargin` threshold. Fix with a scroll-end guard that force-activates the last ID when `scrollY + innerHeight >= document.body.scrollHeight - tolerance`. This edge case is nearly always absent from initial implementations and is a strong differentiator in senior-level interviews.
