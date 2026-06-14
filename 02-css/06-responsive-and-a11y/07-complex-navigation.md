# Complex Multi-Level Responsive Navigation

## The Idea

**In plain English:** You have a navigation menu with multiple levels — a top bar with categories, dropdowns under each category, and sub-menus under those. On desktop, hovering a category shows a dropdown. On mobile, none of that works — you need a completely different interaction model: a hamburger button, a full-screen or drawer panel, and a way to drill into nested levels one at a time.

**Real-world analogy:** Think of a large department store.

- On the **ground floor (desktop)**, you see all the department signs at once — "Clothing", "Electronics", "Home" — and you can glance sideways to see sub-sections like "Men's", "Women's", "Kids'" all visible at the same time.
- On a **single escalator (mobile)**, you can only see one floor at a time. You ride up to "Clothing", then decide whether to go to "Men's" or "Women's", then go deeper into "Jackets". You go deeper one step at a time — and there's always a back button to ride down.
- The **store map board at the entrance** = the hamburger menu button.
- **Each floor** = a navigation level (Level 1 → Level 2 → Level 3).
- **The escalator** = the slide-in transition between levels.
- **"You are here" signs** = breadcrumb trail showing your current depth.

The key insight: desktop and mobile aren't the same navigation — they're the same *data* rendered with two completely different *interaction patterns*.

---

## Learning Objectives

- Understand why CSS alone cannot handle 3+ level dynamic navigation on mobile
- Build the CSS foundation: off-canvas drawer, overlay, level transitions
- Know exactly where React state and Angular signals take over from CSS
- Implement ARIA correctly for nested navigation (`aria-expanded`, `aria-haspopup`)
- Handle dynamic/data-driven menu trees in both frameworks
- Avoid the common production pitfalls: scroll lock, focus trap, animation jank

---

## Why CSS Alone Isn't Enough

A simple 2-level hamburger can be done with a CSS `:checked` checkbox hack. But 3+ levels with dynamic data breaks CSS-only approaches because:

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Toggle open/closed | ✅ with `:checked` hack | — |
| Show/hide a single dropdown | ✅ with `:hover` / `:focus-within` | — |
| Track *which* level is currently active | ❌ | CSS has no integer state |
| Navigate into level 3 from level 2 | ❌ | Requires knowing which item was clicked |
| Render menu items from a JSON API | ❌ | CSS doesn't fetch or loop over data |
| Trap focus inside the open drawer | ❌ | Requires JS `focusin` listeners |
| Lock body scroll when drawer is open | ❌ | `overflow: hidden` on `body` needs JS to add/remove |
| Announce state to screen readers dynamically | ❌ | `aria-expanded` toggle requires JS |
| Animate level transitions (slide left/right) | ⚠️ | CSS can animate, but JS must trigger the class |

**Conclusion:** CSS owns the visual layer (drawer shape, overlay, transitions). JS/framework owns the state layer (which level, which item is open, focus, scroll lock).

---

## HTML & CSS Foundation

### The Structure

```html
<!-- The toggle button (visible on mobile only) -->
<button class="nav-toggle" aria-label="Open menu" aria-expanded="false" aria-controls="mobile-nav">
  <span class="hamburger-icon"></span>
</button>

<!-- The overlay backdrop -->
<div class="nav-overlay" aria-hidden="true"></div>

<!-- The navigation drawer -->
<nav id="mobile-nav" class="nav-drawer" aria-label="Main navigation">

  <!-- Level 1: top-level items -->
  <ul class="nav-level" data-level="1">
    <li>
      <button class="nav-item-btn" aria-haspopup="true" aria-expanded="false">
        Clothing
        <span class="nav-chevron" aria-hidden="true">›</span>
      </button>
    </li>
    <li>
      <button class="nav-item-btn" aria-haspopup="true" aria-expanded="false">
        Electronics
        <span class="nav-chevron" aria-hidden="true">›</span>
      </button>
    </li>
  </ul>

  <!-- Level 2: slides in from the right -->
  <ul class="nav-level nav-level--child" data-level="2" aria-label="Clothing" hidden>
    <li>
      <button class="nav-back-btn">
        <span aria-hidden="true">‹</span> Back
      </button>
    </li>
    <li>
      <button class="nav-item-btn" aria-haspopup="true" aria-expanded="false">
        Men's
        <span class="nav-chevron" aria-hidden="true">›</span>
      </button>
    </li>
    <li>
      <a class="nav-item-link" href="/clothing/women">Women's</a>
    </li>
  </ul>

</nav>

<!-- Desktop nav (separate, always visible on wide screens) -->
<nav class="desktop-nav" aria-label="Main navigation">
  <!-- dropdown structure lives here -->
</nav>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --nav-drawer-width: 320px;
  --nav-transition: 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  --nav-overlay-bg: rgba(0, 0, 0, 0.5);
  --nav-level-bg: #ffffff;
  --nav-level-2-bg: #f8f8f8;
}

/* ─── Mobile toggle button ─── */
.nav-toggle {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 44px;   /* minimum tap target */
  height: 44px;
  background: none;
  border: none;
  cursor: pointer;
}

/* Hamburger icon via CSS */
.hamburger-icon,
.hamburger-icon::before,
.hamburger-icon::after {
  display: block;
  width: 24px;
  height: 2px;
  background: currentColor;
  transition: transform var(--nav-transition);
}
.hamburger-icon { position: relative; }
.hamburger-icon::before,
.hamburger-icon::after {
  content: '';
  position: absolute;
  left: 0;
}
.hamburger-icon::before { top: -7px; }
.hamburger-icon::after  { top:  7px; }

/* When drawer is open, morph hamburger to X */
.nav-toggle[aria-expanded="true"] .hamburger-icon {
  background: transparent;
}
.nav-toggle[aria-expanded="true"] .hamburger-icon::before {
  transform: translateY(7px) rotate(45deg);
}
.nav-toggle[aria-expanded="true"] .hamburger-icon::after {
  transform: translateY(-7px) rotate(-45deg);
}

/* ─── Overlay ─── */
.nav-overlay {
  position: fixed;
  inset: 0;
  background: var(--nav-overlay-bg);
  opacity: 0;
  visibility: hidden;
  transition: opacity var(--nav-transition), visibility var(--nav-transition);
  z-index: 100;
}

.nav-overlay.is-visible {
  opacity: 1;
  visibility: visible;
}

/* ─── Drawer ─── */
.nav-drawer {
  position: fixed;
  top: 0;
  left: 0;
  width: var(--nav-drawer-width);
  max-width: 100vw;
  height: 100dvh;     /* dvh = dynamic viewport height, avoids mobile browser chrome issue */
  background: var(--nav-level-bg);
  transform: translateX(-100%);
  transition: transform var(--nav-transition);
  overflow: hidden;   /* children slide within the drawer, not the page */
  z-index: 101;
  will-change: transform;
}

.nav-drawer.is-open {
  transform: translateX(0);
}

/* ─── Level panels ─── */
/*
  Each level is a full-width panel stacked horizontally.
  They slide left/right inside the overflow:hidden drawer.
  All levels are absolutely positioned; JS adds .is-active.
*/
.nav-level {
  position: absolute;
  inset: 0;
  overflow-y: auto;
  overflow-x: hidden;
  transform: translateX(100%);   /* off-screen to the right by default */
  transition: transform var(--nav-transition);
  background: var(--nav-level-bg);
  list-style: none;
  padding: 0;
  margin: 0;
}

.nav-level[data-level="1"] {
  transform: translateX(0);       /* level 1 starts visible */
}

.nav-level.is-active {
  transform: translateX(0);       /* current level slides in */
}

.nav-level.is-exiting {
  transform: translateX(-100%);   /* previous level slides out to the left */
}

/* Level 2+ has slightly different background to give depth cue */
.nav-level--child {
  background: var(--nav-level-2-bg);
}

/* ─── Nav items ─── */
.nav-item-btn,
.nav-item-link,
.nav-back-btn {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
  min-height: 52px;    /* comfortable tap target */
  padding: 0 1.25rem;
  background: none;
  border: none;
  border-bottom: 1px solid #eee;
  font-size: 1rem;
  text-align: left;
  cursor: pointer;
  text-decoration: none;
  color: inherit;
}

.nav-back-btn {
  font-weight: 600;
  color: #0066cc;
}

.nav-chevron {
  font-size: 1.25rem;
  color: #999;
  transition: transform 0.2s;
}

/* ─── Desktop nav (hidden on mobile) ─── */
.desktop-nav {
  display: none;
}

/* ─── Body scroll lock (added by JS) ─── */
body.nav-open {
  overflow: hidden;
}

/* ─── Responsive: show desktop nav, hide mobile controls ─── */
@media (min-width: 1024px) {
  .nav-toggle,
  .nav-drawer,
  .nav-overlay {
    display: none;
  }

  .desktop-nav {
    display: flex;
    align-items: center;
    gap: 0;
  }

  /* Desktop dropdown */
  .desktop-nav .nav-item {
    position: relative;
  }

  .desktop-nav .dropdown {
    position: absolute;
    top: 100%;
    left: 0;
    min-width: 200px;
    background: #fff;
    box-shadow: 0 4px 16px rgba(0,0,0,0.12);
    border-radius: 4px;
    opacity: 0;
    visibility: hidden;
    transform: translateY(-8px);
    transition: opacity 0.2s, transform 0.2s, visibility 0.2s;
    z-index: 200;
  }

  /* Show dropdown on hover AND on focus-within (keyboard users) */
  .desktop-nav .nav-item:hover .dropdown,
  .desktop-nav .nav-item:focus-within .dropdown {
    opacity: 1;
    visibility: visible;
    transform: translateY(0);
  }
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .nav-drawer,
  .nav-level,
  .nav-overlay,
  .desktop-nav .dropdown {
    transition: none;
  }
}
```

**What CSS owns:** drawer position, slide animation, overlay fade, hamburger → X morph, desktop dropdown show/hide, reduced-motion fallback.

**What CSS cannot own:** which level is active, which items are expanded, body scroll lock, focus trap, ARIA state toggling.

---

## React Implementation

### Data Shape

```tsx
interface NavItem {
  id: string;
  label: string;
  href?: string;           // leaf node — no children
  children?: NavItem[];    // branch node — has submenu
}

const navData: NavItem[] = [
  {
    id: 'clothing',
    label: 'Clothing',
    children: [
      {
        id: 'mens',
        label: "Men's",
        children: [
          { id: 'mens-jackets', label: 'Jackets', href: '/clothing/mens/jackets' },
          { id: 'mens-shirts',  label: 'Shirts',  href: '/clothing/mens/shirts'  },
        ],
      },
      { id: 'womens', label: "Women's", href: '/clothing/womens' },
    ],
  },
  {
    id: 'electronics',
    label: 'Electronics',
    children: [
      { id: 'phones',  label: 'Phones',  href: '/electronics/phones'  },
      { id: 'laptops', label: 'Laptops', href: '/electronics/laptops' },
    ],
  },
];
```

### The Hook — Navigation State

```tsx
// useNavigation.ts
import { useState, useCallback, useEffect, useRef } from 'react';
import type { NavItem } from './types';

interface NavLevel {
  items: NavItem[];
  label: string;        // used for aria-label and back button text
}

export function useNavigation(root: NavItem[]) {
  const [isOpen, setIsOpen]     = useState(false);
  const [stack, setStack]       = useState<NavLevel[]>([
    { items: root, label: 'Main menu' },
  ]);

  // The current level is always the top of the stack
  const currentLevel = stack[stack.length - 1];
  const depth        = stack.length - 1;     // 0 = root level

  const open = useCallback(() => {
    setIsOpen(true);
    document.body.classList.add('nav-open');
  }, []);

  const close = useCallback(() => {
    setIsOpen(false);
    setStack([{ items: root, label: 'Main menu' }]); // reset to root on close
    document.body.classList.remove('nav-open');
  }, [root]);

  const drillInto = useCallback((item: NavItem) => {
    if (!item.children) return;
    setStack(prev => [...prev, { items: item.children!, label: item.label }]);
  }, []);

  const goBack = useCallback(() => {
    setStack(prev => prev.length > 1 ? prev.slice(0, -1) : prev);
  }, []);

  // Close on Escape
  useEffect(() => {
    if (!isOpen) return;
    const onKey = (e: KeyboardEvent) => {
      if (e.key === 'Escape') close();
    };
    document.addEventListener('keydown', onKey);
    return () => document.removeEventListener('keydown', onKey);
  }, [isOpen, close]);

  // Cleanup body class on unmount
  useEffect(() => {
    return () => document.body.classList.remove('nav-open');
  }, []);

  return { isOpen, open, close, drillInto, goBack, currentLevel, depth, stack };
}
```

### The Components

```tsx
// MobileNav.tsx
import { useRef, useEffect } from 'react';
import { useNavigation } from './useNavigation';
import type { NavItem } from './types';

interface Props {
  items: NavItem[];
}

export function MobileNav({ items }: Props) {
  const { isOpen, open, close, drillInto, goBack, currentLevel, depth } =
    useNavigation(items);

  const drawerRef    = useRef<HTMLElement>(null);
  const toggleBtnRef = useRef<HTMLButtonElement>(null);

  // Focus trap: keep focus inside the drawer when open
  useEffect(() => {
    if (!isOpen || !drawerRef.current) return;

    const focusable = drawerRef.current.querySelectorAll<HTMLElement>(
      'a[href], button:not([disabled])'
    );
    const first = focusable[0];
    const last  = focusable[focusable.length - 1];

    first?.focus();

    const trap = (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return;
      if (e.shiftKey) {
        if (document.activeElement === first) {
          e.preventDefault();
          last?.focus();
        }
      } else {
        if (document.activeElement === last) {
          e.preventDefault();
          first?.focus();
        }
      }
    };

    document.addEventListener('keydown', trap);
    return () => document.removeEventListener('keydown', trap);
  }, [isOpen, currentLevel]); // re-run when level changes (new focusable items)

  // Return focus to toggle button on close
  useEffect(() => {
    if (!isOpen) toggleBtnRef.current?.focus();
  }, [isOpen]);

  return (
    <>
      {/* Hamburger toggle */}
      <button
        ref={toggleBtnRef}
        className="nav-toggle"
        aria-label={isOpen ? 'Close menu' : 'Open menu'}
        aria-expanded={isOpen}
        aria-controls="mobile-nav"
        onClick={isOpen ? close : open}
      >
        <span className="hamburger-icon" aria-hidden="true" />
      </button>

      {/* Overlay */}
      <div
        className={`nav-overlay ${isOpen ? 'is-visible' : ''}`}
        aria-hidden="true"
        onClick={close}
      />

      {/* Drawer */}
      <nav
        id="mobile-nav"
        ref={drawerRef}
        className={`nav-drawer ${isOpen ? 'is-open' : ''}`}
        aria-label="Main navigation"
        aria-hidden={!isOpen}
      >
        {/* Breadcrumb hint for screen readers */}
        {depth > 0 && (
          <p className="sr-only" aria-live="polite">
            Navigated into {currentLevel.label}
          </p>
        )}

        <NavLevel
          level={currentLevel}
          depth={depth}
          onDrillInto={drillInto}
          onBack={goBack}
        />
      </nav>
    </>
  );
}

/* ── Single level panel ── */
interface NavLevelProps {
  level: { items: NavItem[]; label: string };
  depth: number;
  onDrillInto: (item: NavItem) => void;
  onBack: () => void;
}

function NavLevel({ level, depth, onDrillInto, onBack }: NavLevelProps) {
  return (
    <ul
      className="nav-level is-active"
      role="list"
      aria-label={level.label}
    >
      {depth > 0 && (
        <li>
          <button className="nav-back-btn" onClick={onBack}>
            <span aria-hidden="true">‹</span> Back
          </button>
        </li>
      )}
      {level.items.map(item => (
        <NavItem
          key={item.id}
          item={item}
          onDrillInto={onDrillInto}
        />
      ))}
    </ul>
  );
}

/* ── Single item: either a link (leaf) or drill-in button (branch) ── */
interface NavItemProps {
  item: NavItem;
  onDrillInto: (item: NavItem) => void;
}

function NavItem({ item, onDrillInto }: NavItemProps) {
  if (!item.children) {
    // Leaf node — navigate to href
    return (
      <li>
        <a className="nav-item-link" href={item.href}>
          {item.label}
        </a>
      </li>
    );
  }
  // Branch node — drill into sub-level
  return (
    <li>
      <button
        className="nav-item-btn"
        aria-haspopup="true"
        onClick={() => onDrillInto(item)}
      >
        {item.label}
        <span className="nav-chevron" aria-hidden="true">›</span>
      </button>
    </li>
  );
}
```

### Desktop Dropdown (React)

```tsx
// DesktopNav.tsx — hover + focus-within handled by CSS, JS only for ARIA
import { useState } from 'react';
import type { NavItem } from './types';

export function DesktopNav({ items }: { items: NavItem[] }) {
  return (
    <nav className="desktop-nav" aria-label="Main navigation">
      <ul role="list" style={{ display: 'flex', gap: 0, listStyle: 'none', margin: 0, padding: 0 }}>
        {items.map(item => (
          <TopLevelItem key={item.id} item={item} />
        ))}
      </ul>
    </nav>
  );
}

function TopLevelItem({ item }: { item: NavItem }) {
  const [isOpen, setIsOpen] = useState(false);

  if (!item.children) {
    return (
      <li className="nav-item">
        <a href={item.href}>{item.label}</a>
      </li>
    );
  }

  return (
    <li
      className="nav-item"
      onMouseEnter={() => setIsOpen(true)}
      onMouseLeave={() => setIsOpen(false)}
    >
      <button
        aria-haspopup="true"
        aria-expanded={isOpen}
        onFocus={() => setIsOpen(true)}
        onBlur={() => setIsOpen(false)}
      >
        {item.label}
      </button>
      {isOpen && (
        <ul className="dropdown" role="menu">
          {item.children.map(child => (
            <li key={child.id} role="menuitem">
              <a href={child.href ?? '#'}>{child.label}</a>
            </li>
          ))}
        </ul>
      )}
    </li>
  );
}
```

---

## Angular Implementation

### Data Shape & Service

```typescript
// nav.model.ts
export interface NavItem {
  id: string;
  label: string;
  href?: string;
  children?: NavItem[];
}

export interface NavLevel {
  items: NavItem[];
  label: string;
}
```

```typescript
// nav.service.ts
import { Injectable, signal, computed } from '@angular/core';
import { NavItem, NavLevel } from './nav.model';

@Injectable({ providedIn: 'root' })
export class NavService {
  private _isOpen  = signal(false);
  private _stack   = signal<NavLevel[]>([]);

  readonly isOpen       = this._isOpen.asReadonly();
  readonly currentLevel = computed(() => {
    const s = this._stack();
    return s[s.length - 1] ?? null;
  });
  readonly depth = computed(() => this._stack().length - 1);

  init(root: NavItem[]) {
    this._stack.set([{ items: root, label: 'Main menu' }]);
  }

  open() {
    this._isOpen.set(true);
    document.body.classList.add('nav-open');
  }

  close() {
    this._isOpen.set(false);
    this._stack.update(s => [s[0]]); // reset to root
    document.body.classList.remove('nav-open');
  }

  drillInto(item: NavItem) {
    if (!item.children) return;
    this._stack.update(s => [...s, { items: item.children!, label: item.label }]);
  }

  goBack() {
    this._stack.update(s => s.length > 1 ? s.slice(0, -1) : s);
  }
}
```

### Mobile Nav Component

```typescript
// mobile-nav.component.ts
import {
  Component, OnInit, OnDestroy, inject,
  HostListener, ElementRef, effect
} from '@angular/core';
import { NgFor, NgIf } from '@angular/common';
import { NavService } from './nav.service';
import { NavItem } from './nav.model';

@Component({
  selector: 'app-mobile-nav',
  standalone: true,
  imports: [NgFor, NgIf],
  template: `
    <!-- Toggle button -->
    <button
      class="nav-toggle"
      [attr.aria-expanded]="navService.isOpen()"
      [attr.aria-label]="navService.isOpen() ? 'Close menu' : 'Open menu'"
      aria-controls="mobile-nav"
      (click)="toggle()"
    >
      <span class="hamburger-icon" aria-hidden="true"></span>
    </button>

    <!-- Overlay -->
    <div
      class="nav-overlay"
      [class.is-visible]="navService.isOpen()"
      aria-hidden="true"
      (click)="navService.close()"
    ></div>

    <!-- Drawer -->
    <nav
      id="mobile-nav"
      #drawerEl
      class="nav-drawer"
      [class.is-open]="navService.isOpen()"
      [attr.aria-hidden]="!navService.isOpen()"
      aria-label="Main navigation"
    >
      <!-- Screen reader live announcement on level change -->
      <p
        *ngIf="navService.depth() > 0"
        class="sr-only"
        aria-live="polite"
      >
        Navigated into {{ navService.currentLevel()?.label }}
      </p>

      <!-- Level panel -->
      <ul
        *ngIf="navService.currentLevel() as level"
        class="nav-level is-active"
        [attr.aria-label]="level.label"
      >
        <!-- Back button for sub-levels -->
        <li *ngIf="navService.depth() > 0">
          <button class="nav-back-btn" (click)="navService.goBack()">
            <span aria-hidden="true">‹</span> Back
          </button>
        </li>

        <!-- Items -->
        <li *ngFor="let item of level.items; trackBy: trackById">
          <!-- Leaf node -->
          <a
            *ngIf="!item.children"
            class="nav-item-link"
            [href]="item.href"
          >{{ item.label }}</a>

          <!-- Branch node -->
          <button
            *ngIf="item.children"
            class="nav-item-btn"
            aria-haspopup="true"
            (click)="navService.drillInto(item)"
          >
            {{ item.label }}
            <span class="nav-chevron" aria-hidden="true">›</span>
          </button>
        </li>
      </ul>
    </nav>
  `,
})
export class MobileNavComponent implements OnInit, OnDestroy {
  navService = inject(NavService);
  private el = inject(ElementRef);

  toggle() {
    this.navService.isOpen() ? this.navService.close() : this.navService.open();
  }

  trackById(_: number, item: NavItem) {
    return item.id;
  }

  // Close on Escape
  @HostListener('document:keydown.escape')
  onEscape() {
    if (this.navService.isOpen()) this.navService.close();
  }

  // Focus trap — re-evaluate whenever the drawer opens or level changes
  private focusTrapCleanup?: () => void;

  constructor() {
    effect(() => {
      const open = this.navService.isOpen();
      this.focusTrapCleanup?.();
      if (!open) return;

      // Wait for DOM update then trap focus
      setTimeout(() => {
        const drawer = this.el.nativeElement.querySelector('#mobile-nav');
        if (!drawer) return;

        const focusable: HTMLElement[] = Array.from(
          drawer.querySelectorAll('a[href], button:not([disabled])')
        );
        focusable[0]?.focus();

        const handler = (e: KeyboardEvent) => {
          if (e.key !== 'Tab') return;
          const first = focusable[0];
          const last  = focusable[focusable.length - 1];
          if (e.shiftKey && document.activeElement === first) {
            e.preventDefault(); last?.focus();
          } else if (!e.shiftKey && document.activeElement === last) {
            e.preventDefault(); first?.focus();
          }
        };

        document.addEventListener('keydown', handler);
        this.focusTrapCleanup = () => document.removeEventListener('keydown', handler);
      }, 50);
    });
  }

  ngOnInit() {
    // NavService.init() called from parent or app shell
  }

  ngOnDestroy() {
    this.focusTrapCleanup?.();
    document.body.classList.remove('nav-open');
  }
}
```

### Providing Data from Parent

```typescript
// app.component.ts
import { Component, OnInit, inject } from '@angular/core';
import { NavService } from './nav.service';
import { NAV_DATA } from './nav-data';   // your static or API-sourced data

@Component({
  selector: 'app-root',
  template: `<app-mobile-nav />`
})
export class AppComponent implements OnInit {
  private navService = inject(NavService);

  ngOnInit() {
    this.navService.init(NAV_DATA);
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Toggle button has `aria-expanded` | Updated by JS on every open/close |
| Toggle button has `aria-controls` pointing to drawer id | Static attribute |
| Drawer has `aria-hidden="true"` when closed | Prevents virtual cursor from entering |
| Each drill-in button has `aria-haspopup="true"` | Signals to screen reader that a submenu exists |
| Level change announced to screen reader | `aria-live="polite"` region updates on level change |
| Focus trapped inside open drawer | Tab/Shift+Tab cycle stays within drawer |
| Escape closes drawer | `keydown` listener on document |
| Focus returns to toggle button on close | `useEffect` / `effect()` watches `isOpen` |
| Back button is the first focusable element on sub-levels | Rendered first in the list |
| Minimum tap target 44×44px | Enforced via CSS `min-height` |
| Animation respects `prefers-reduced-motion` | `@media` query disables all transitions |

---

## Production Pitfalls

**1. `height: 100vh` breaks on mobile browsers**
Mobile browsers include their chrome (address bar) in `100vh`. The drawer is taller than the visible screen. Fix: use `height: 100dvh` (dynamic viewport height). Add a fallback: `height: 100vh; height: 100dvh`.

**2. Body scroll bleeds through the overlay**
`overflow: hidden` on `body` alone doesn't stop iOS Safari from scrolling the background. Fix: also apply `position: fixed; width: 100%` to `body` when the nav is open. Store the scroll position first so you can restore it on close:
```js
const scrollY = window.scrollY;
document.body.style.top = `-${scrollY}px`;
// on close:
document.body.style.top = '';
window.scrollTo(0, scrollY);
```

**3. Animation jank on low-end devices**
`translateX` on a large element can cause compositing issues. Fix: add `will-change: transform` to `.nav-drawer` (already in the CSS above) and make sure nothing inside the drawer triggers layout during the animation.

**4. Level transition feels abrupt**
If the previous level disappears instantly before the new one slides in, it looks like a flash. Fix: keep both the exiting and entering level visible simultaneously using the `.is-exiting` class, which slides the old level to the left while the new level slides in from the right.

**5. Focus trap breaks after level change**
After drilling into a sub-level, the focusable elements list changes. If the trap is set once on open and never refreshed, Tab will cycle through stale elements. Fix: re-run the focus trap setup every time the level changes (the `effect()` dependency in Angular, the `[isOpen, currentLevel]` dep array in React).

**6. Dynamic data causes flicker on first render**
If the nav data loads asynchronously, the drawer may render empty for a frame. Fix: use a skeleton loader or `@defer` block (Angular) / `Suspense` (React) at the nav level, not inside individual items.

---

## Interview Angle

**Q: "How would you handle a 3-level navigation that needs to work on both desktop and mobile?"**

Strong answer covers four points:
1. **Same data, two rendering strategies** — desktop uses hover-based dropdowns with `focus-within` fallback; mobile uses a drill-down drawer with a stack data structure.
2. **CSS owns the visual layer** — transitions, positioning, and the hamburger icon morph. Explain why CSS alone breaks at 3+ levels (no integer state, no DOM mutation).
3. **JS owns the state layer** — a stack of visited levels, body scroll lock, focus trap, and ARIA toggling. In React: a custom hook with `useState` + `useEffect`. In Angular: a signal-based service with `effect()`.
4. **Accessibility is non-negotiable** — `aria-expanded`, `aria-haspopup`, `aria-hidden` on the drawer, focus trap, Escape key, and `aria-live` announcement on level change.

**Follow-up: "What breaks in production that wouldn't show up in a demo?"**
iOS `100vh`, body scroll bleed-through on Safari, focus trap invalidation after level change, animation jank — the pitfalls section above covers these exactly.
