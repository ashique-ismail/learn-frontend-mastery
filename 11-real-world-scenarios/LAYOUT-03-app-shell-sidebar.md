# App Shell with Collapsible Sidebar

## The Idea

**In plain English:** You have a persistent application shell — a top header, a side navigation panel, and a main content area. On desktop the sidebar sits beside the content at all times, but can collapse to a narrow icon-only rail to reclaim horizontal space. On mobile the sidebar becomes a full overlay drawer triggered by a hamburger button. The user's preference (open or collapsed) persists between page loads via `localStorage`, and the current sidebar state can optionally be encoded in the URL's search params so a deep link preserves the layout the user expects.

**Real-world analogy:** Think of a filing cabinet next to a large desk.

- The **filing cabinet (sidebar)** holds your navigation and quick links. When you are actively browsing files you pull it fully open so labels are readable. When you are focused on writing you push it most of the way closed — the cabinet is still there, but you only see the colored tab edges (icon-only rail), not the full labels.
- The **desk surface (main area)** expands to fill the space the cabinet gave up. Nothing moves off-screen; the layout reflows.
- The **sticky note on the cabinet** = `localStorage` — it remembers whether you left the cabinet open or closed last time you sat down.
- On a **laptop tray table (mobile)**, the filing cabinet doesn't fit beside the desk. Instead, it slides in from the left on demand (overlay drawer) and covers part of the desk, then slides away when you are done.
- The **URL bar** acts like a whiteboard you can share with a colleague — if the sidebar state is in the URL, your colleague opens the same layout when they click the link.

The key insight: the sidebar has three distinct modes — *expanded* (desktop, labels visible), *collapsed/rail* (desktop, icons only), and *overlay* (mobile, full-screen drawer) — and each mode requires different CSS, different ARIA, and slightly different state management.

---

## Learning Objectives

- Build a CSS Grid app shell using named template areas that reflows correctly when the sidebar expands and collapses
- Manage sidebar open/collapsed state with URL `searchParams` as the source of truth and `localStorage` as a silent fallback
- Understand when to use a modal overlay sidebar (mobile) versus an inline sidebar (desktop) and the CSS breakpoint boundary between them
- Implement a focus trap that activates only when the sidebar is in overlay mode
- Apply correct ARIA patterns: `aria-expanded`, `aria-controls`, `aria-hidden`, and `role="dialog"` on the overlay variant
- Avoid the common pitfalls: CSS Grid gap not collapsing, scrollbar flicker during sidebar transition, and mismatched state on back-navigation

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Sidebar shape, width, transition | ✅ | — |
| CSS Grid named areas reflow | ✅ | — |
| Read/write `localStorage` for persistence | ❌ | CSS has no storage API |
| Sync state with URL `searchParams` | ❌ | CSS cannot read or write the URL |
| Toggle `aria-expanded` on the trigger button | ❌ | ARIA attributes require JS to mutate |
| Apply `aria-hidden` to sidebar when overlay is closed | ❌ | Requires JS DOM mutation |
| Focus trap in overlay mode | ❌ | Requires `keydown` listeners and DOM queries |
| Restore focus to trigger on overlay close | ❌ | Requires JS `focus()` call |
| Lock body scroll while mobile overlay is open | ❌ | `overflow: hidden` on `body` must be added/removed by JS |
| Detect viewport width and switch between inline/overlay modes | ⚠️ | CSS media queries switch the visual presentation, but JS must know the mode to apply correct ARIA and focus behavior |

**Conclusion:** CSS Grid owns the layout geometry and the visual transition. JS/framework owns every stateful concern: persistence, URL sync, ARIA, focus trap, and mode detection.

---

## HTML & CSS Foundation

### The Structure

```html
<!-- App Shell root -->
<div class="app-shell" data-sidebar="expanded">

  <!-- ① Top header spans full width -->
  <header class="app-header">
    <button
      class="sidebar-toggle"
      aria-label="Collapse sidebar"
      aria-expanded="true"
      aria-controls="app-sidebar"
    >
      <!-- icon -->
    </button>
    <span class="app-logo">MyApp</span>
  </header>

  <!-- ② Sidebar — inline on desktop, overlay on mobile -->
  <aside
    id="app-sidebar"
    class="app-sidebar"
    aria-label="Application navigation"
  >
    <nav aria-label="Main">
      <ul role="list">
        <li>
          <a href="/dashboard" class="nav-link" aria-current="page">
            <span class="nav-icon" aria-hidden="true">⊞</span>
            <span class="nav-label">Dashboard</span>
          </a>
        </li>
        <li>
          <a href="/reports" class="nav-link">
            <span class="nav-icon" aria-hidden="true">◈</span>
            <span class="nav-label">Reports</span>
          </a>
        </li>
      </ul>
    </nav>
  </aside>

  <!-- ③ Overlay backdrop — only visible on mobile -->
  <div class="sidebar-overlay" aria-hidden="true"></div>

  <!-- ④ Main content area -->
  <main id="main-content" class="app-main" tabindex="-1">
    <slot />
  </main>

</div>
```

### The CSS

```css
/* ─── Design tokens ─── */
:root {
  --sidebar-width-expanded: 260px;
  --sidebar-width-collapsed: 60px;
  --header-height: 56px;
  --sidebar-transition: 0.25s cubic-bezier(0.4, 0, 0.2, 1);
  --sidebar-bg: #1e1e2e;
  --sidebar-text: #cdd6f4;
  --header-bg: #181825;
  --main-bg: #f4f4f5;
  --overlay-bg: rgba(0, 0, 0, 0.5);
}

/* ─── App shell grid ─── */
/*
  Named areas:
    "header  header"
    "sidebar main"

  The sidebar column width is driven by a CSS custom property
  that JS updates when the sidebar collapses.
*/
.app-shell {
  display: grid;
  grid-template-columns: var(--sidebar-col-width, var(--sidebar-width-expanded)) 1fr;
  grid-template-rows: var(--header-height) 1fr;
  grid-template-areas:
    "header  header"
    "sidebar main";
  min-height: 100dvh;
  transition: grid-template-columns var(--sidebar-transition);
}

/* Collapsed state — JS sets data-sidebar="collapsed" */
.app-shell[data-sidebar="collapsed"] {
  --sidebar-col-width: var(--sidebar-width-collapsed);
}

/* ─── Header ─── */
.app-header {
  grid-area: header;
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 0 1rem;
  background: var(--header-bg);
  color: #fff;
  position: sticky;
  top: 0;
  z-index: 10;
}

/* ─── Sidebar — desktop inline ─── */
.app-sidebar {
  grid-area: sidebar;
  background: var(--sidebar-bg);
  color: var(--sidebar-text);
  overflow: hidden;             /* clips the nav-label text as sidebar narrows */
  overflow-y: auto;
  transition: width var(--sidebar-transition);
  width: var(--sidebar-col-width, var(--sidebar-width-expanded));
  position: sticky;
  top: var(--header-height);
  height: calc(100dvh - var(--header-height));
}

/* Nav links */
.nav-link {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.75rem 1rem;
  color: var(--sidebar-text);
  text-decoration: none;
  white-space: nowrap;
  border-radius: 6px;
  transition: background 0.15s;
}
.nav-link:hover,
.nav-link[aria-current="page"] {
  background: rgba(255, 255, 255, 0.08);
}
.nav-icon {
  flex-shrink: 0;
  font-size: 1.25rem;
  width: 28px;
  text-align: center;
}
.nav-label {
  opacity: 1;
  transition: opacity var(--sidebar-transition);
}

/* Hide labels when collapsed — they are still in the DOM for screen readers */
.app-shell[data-sidebar="collapsed"] .nav-label {
  opacity: 0;
  pointer-events: none;
}

/* ─── Main area ─── */
.app-main {
  grid-area: main;
  background: var(--main-bg);
  padding: 2rem;
  overflow-y: auto;
  /* outline: none keeps focus ring away on programmatic focus */
  outline: none;
}

/* ─── Overlay backdrop — hidden on desktop ─── */
.sidebar-overlay {
  display: none;
}

/* ─── Toggle button ─── */
.sidebar-toggle {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 40px;
  background: none;
  border: none;
  color: inherit;
  cursor: pointer;
  border-radius: 6px;
  flex-shrink: 0;
}
.sidebar-toggle:hover {
  background: rgba(255, 255, 255, 0.1);
}

/* ─── Mobile: sidebar becomes overlay ─── */
@media (max-width: 768px) {
  /* Collapse the grid sidebar column to zero — the sidebar floats above */
  .app-shell {
    grid-template-columns: 0 1fr;
    grid-template-areas:
      "header header"
      "main   main";
  }

  /* Sidebar is fixed overlay, off-screen by default */
  .app-sidebar {
    position: fixed;
    top: 0;
    left: 0;
    width: var(--sidebar-width-expanded);
    height: 100dvh;
    z-index: 200;
    transform: translateX(-100%);
    transition: transform var(--sidebar-transition);
    will-change: transform;
  }

  /* Open state — JS adds data-sidebar="open" on mobile */
  .app-shell[data-sidebar="open"] .app-sidebar {
    transform: translateX(0);
  }

  /* Overlay backdrop */
  .sidebar-overlay {
    display: block;
    position: fixed;
    inset: 0;
    background: var(--overlay-bg);
    z-index: 199;
    opacity: 0;
    visibility: hidden;
    transition: opacity var(--sidebar-transition), visibility var(--sidebar-transition);
  }

  .app-shell[data-sidebar="open"] .sidebar-overlay {
    opacity: 1;
    visibility: visible;
  }
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .app-shell,
  .app-sidebar,
  .sidebar-overlay,
  .nav-label {
    transition: none;
  }
}
```

**What CSS owns:** named grid area reflow, sidebar slide transition, overlay fade, nav-label visibility, reduced-motion fallback.

**What CSS cannot own:** reading/writing `localStorage`, syncing `searchParams`, toggling ARIA attributes, activating the focus trap in overlay mode, locking body scroll.

---

## React Implementation

### Types

```tsx
// sidebar.types.ts
export type SidebarMode = 'expanded' | 'collapsed';   // desktop
export type MobileState = 'open' | 'closed';           // mobile overlay
```

### The Hook — State, Persistence, and URL Sync

```tsx
// useSidebar.ts
import { useState, useCallback, useEffect, useLayoutEffect } from 'react';
import type { SidebarMode } from './sidebar.types';

const STORAGE_KEY = 'sidebar-mode';
const PARAM_KEY   = 'sidebar';

function readInitialMode(): SidebarMode {
  // 1. URL search param wins (enables shareable layout)
  const params = new URLSearchParams(window.location.search);
  const fromUrl = params.get(PARAM_KEY);
  if (fromUrl === 'expanded' || fromUrl === 'collapsed') return fromUrl;

  // 2. localStorage fallback (restores last session preference)
  try {
    const stored = localStorage.getItem(STORAGE_KEY);
    if (stored === 'expanded' || stored === 'collapsed') return stored;
  } catch {
    // localStorage unavailable (private mode, etc.)
  }

  // 3. Default
  return 'expanded';
}

export function useSidebar() {
  const [mode, setMode]             = useState<SidebarMode>(readInitialMode);
  const [mobileOpen, setMobileOpen] = useState(false);
  const [isMobile, setIsMobile]     = useState(false);

  // Detect mobile breakpoint
  useLayoutEffect(() => {
    const mq = window.matchMedia('(max-width: 768px)');
    const update = (e: MediaQueryListEvent | MediaQueryList) => {
      setIsMobile(e.matches);
      if (e.matches) setMobileOpen(false); // close on resize to mobile
    };
    update(mq);
    mq.addEventListener('change', update);
    return () => mq.removeEventListener('change', update);
  }, []);

  // Write mode to URL + localStorage whenever it changes (desktop only)
  useEffect(() => {
    if (isMobile) return;

    // Persist to localStorage
    try {
      localStorage.setItem(STORAGE_KEY, mode);
    } catch { /* ignore */ }

    // Sync to URL without triggering a navigation
    const url = new URL(window.location.href);
    url.searchParams.set(PARAM_KEY, mode);
    window.history.replaceState(null, '', url.toString());
  }, [mode, isMobile]);

  const toggleDesktop = useCallback(() => {
    setMode(prev => (prev === 'expanded' ? 'collapsed' : 'expanded'));
  }, []);

  const openMobile  = useCallback(() => setMobileOpen(true),  []);
  const closeMobile = useCallback(() => setMobileOpen(false), []);

  // Body scroll lock when overlay is open
  useEffect(() => {
    if (!mobileOpen) return;
    const scrollY = window.scrollY;
    document.body.style.overflow = 'hidden';
    document.body.style.position = 'fixed';
    document.body.style.top      = `-${scrollY}px`;
    document.body.style.width    = '100%';
    return () => {
      document.body.style.overflow = '';
      document.body.style.position = '';
      document.body.style.top      = '';
      document.body.style.width    = '';
      window.scrollTo(0, scrollY);
    };
  }, [mobileOpen]);

  // Derive the data-sidebar attribute value for the root element
  const shellAttr: string = isMobile
    ? (mobileOpen ? 'open' : 'closed')
    : mode;

  return {
    mode,
    mobileOpen,
    isMobile,
    shellAttr,
    toggleDesktop,
    openMobile,
    closeMobile,
  };
}
```

### Focus Trap Hook

```tsx
// useFocusTrap.ts
import { useEffect, RefObject } from 'react';

const FOCUSABLE =
  'a[href], button:not([disabled]), input:not([disabled]), [tabindex]:not([tabindex="-1"])';

export function useFocusTrap(
  containerRef: RefObject<HTMLElement | null>,
  active: boolean
) {
  useEffect(() => {
    if (!active || !containerRef.current) return;

    const el = containerRef.current;
    const nodes = Array.from(el.querySelectorAll<HTMLElement>(FOCUSABLE));
    const first = nodes[0];
    const last  = nodes[nodes.length - 1];

    // Move focus into the container
    first?.focus();

    const handler = (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return;
      if (nodes.length === 0) { e.preventDefault(); return; }
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

    document.addEventListener('keydown', handler);
    return () => document.removeEventListener('keydown', handler);
  }, [active, containerRef]);
}
```

### App Shell Component

```tsx
// AppShell.tsx
import { useRef, useEffect, useCallback } from 'react';
import { useSidebar } from './useSidebar';
import { useFocusTrap } from './useFocusTrap';

interface NavItem {
  id: string;
  label: string;
  href: string;
  icon: string;
}

const NAV_ITEMS: NavItem[] = [
  { id: 'dashboard', label: 'Dashboard', href: '/dashboard', icon: '⊞' },
  { id: 'reports',   label: 'Reports',   href: '/reports',   icon: '◈' },
  { id: 'settings',  label: 'Settings',  href: '/settings',  icon: '⚙' },
];

interface AppShellProps {
  children: React.ReactNode;
  currentPath: string;
}

export function AppShell({ children, currentPath }: AppShellProps) {
  const {
    mode,
    mobileOpen,
    isMobile,
    shellAttr,
    toggleDesktop,
    openMobile,
    closeMobile,
  } = useSidebar();

  const sidebarRef    = useRef<HTMLElement>(null);
  const toggleBtnRef  = useRef<HTMLButtonElement>(null);
  const mainRef       = useRef<HTMLElement>(null);

  // Focus trap — only active in mobile overlay mode
  useFocusTrap(sidebarRef, isMobile && mobileOpen);

  // Return focus to toggle button when overlay closes
  useEffect(() => {
    if (isMobile && !mobileOpen) {
      toggleBtnRef.current?.focus();
    }
  }, [isMobile, mobileOpen]);

  // Close mobile overlay on Escape
  useEffect(() => {
    if (!mobileOpen) return;
    const onKey = (e: KeyboardEvent) => {
      if (e.key === 'Escape') closeMobile();
    };
    document.addEventListener('keydown', onKey);
    return () => document.removeEventListener('keydown', onKey);
  }, [mobileOpen, closeMobile]);

  const handleToggle = isMobile ? openMobile : toggleDesktop;
  const isExpanded   = isMobile ? mobileOpen : mode === 'expanded';

  return (
    <div
      className="app-shell"
      data-sidebar={shellAttr}
    >
      {/* Header */}
      <header className="app-header">
        <button
          ref={toggleBtnRef}
          className="sidebar-toggle"
          aria-label={isExpanded ? 'Collapse sidebar' : 'Expand sidebar'}
          aria-expanded={isExpanded}
          aria-controls="app-sidebar"
          onClick={handleToggle}
        >
          {/* Replace with your icon component */}
          <span aria-hidden="true">{isExpanded ? '◀' : '▶'}</span>
        </button>
        <span className="app-logo">MyApp</span>
      </header>

      {/* Sidebar */}
      <aside
        id="app-sidebar"
        ref={sidebarRef}
        className="app-sidebar"
        aria-label="Application navigation"
        // In overlay mode, hide from AT when closed
        aria-hidden={isMobile && !mobileOpen ? true : undefined}
        // dialog role only applies when it is a modal overlay
        role={isMobile ? 'dialog' : undefined}
        aria-modal={isMobile ? true : undefined}
      >
        {isMobile && (
          <button
            className="sidebar-close-btn"
            aria-label="Close navigation"
            onClick={closeMobile}
          >
            ✕
          </button>
        )}
        <nav aria-label="Main">
          <ul role="list" style={{ listStyle: 'none', margin: 0, padding: '0.5rem' }}>
            {NAV_ITEMS.map(item => (
              <li key={item.id}>
                <a
                  className="nav-link"
                  href={item.href}
                  aria-current={currentPath === item.href ? 'page' : undefined}
                  // Provide tooltip on collapsed icon-only rail
                  title={mode === 'collapsed' && !isMobile ? item.label : undefined}
                >
                  <span className="nav-icon" aria-hidden="true">{item.icon}</span>
                  <span className="nav-label">{item.label}</span>
                </a>
              </li>
            ))}
          </ul>
        </nav>
      </aside>

      {/* Overlay backdrop (mobile only) */}
      <div
        className="sidebar-overlay"
        aria-hidden="true"
        onClick={closeMobile}
      />

      {/* Main content */}
      <main
        id="main-content"
        ref={mainRef}
        className="app-main"
        tabIndex={-1}
      >
        {children}
      </main>
    </div>
  );
}
```

---

## Angular Implementation

### Model

```typescript
// sidebar.model.ts
export type SidebarMode = 'expanded' | 'collapsed';
```

### Sidebar Service

```typescript
// sidebar.service.ts
import { Injectable, signal, computed, effect } from '@angular/core';
import type { SidebarMode } from './sidebar.model';

const STORAGE_KEY = 'sidebar-mode';
const PARAM_KEY   = 'sidebar';

@Injectable({ providedIn: 'root' })
export class SidebarService {
  private readonly _mode       = signal<SidebarMode>(this.readInitialMode());
  private readonly _mobileOpen = signal(false);
  private readonly _isMobile   = signal(false);

  readonly mode        = this._mode.asReadonly();
  readonly mobileOpen  = this._mobileOpen.asReadonly();
  readonly isMobile    = this._isMobile.asReadonly();

  readonly shellAttr = computed<string>(() =>
    this._isMobile()
      ? (this._mobileOpen() ? 'open' : 'closed')
      : this._mode()
  );

  readonly isExpanded = computed(() =>
    this._isMobile() ? this._mobileOpen() : this._mode() === 'expanded'
  );

  constructor() {
    // Sync mode changes to URL + localStorage (desktop only)
    effect(() => {
      const mode    = this._mode();
      const mobile  = this._isMobile();
      if (mobile) return;

      try { localStorage.setItem(STORAGE_KEY, mode); } catch { /* ignore */ }

      const url = new URL(window.location.href);
      url.searchParams.set(PARAM_KEY, mode);
      window.history.replaceState(null, '', url.toString());
    });
  }

  initMediaQuery(): void {
    const mq = window.matchMedia('(max-width: 768px)');
    const update = (e: MediaQueryList | MediaQueryListEvent) => {
      this._isMobile.set(e.matches);
      if (e.matches) this._mobileOpen.set(false);
    };
    update(mq);
    mq.addEventListener('change', update);
  }

  toggleDesktop(): void {
    this._mode.update(m => m === 'expanded' ? 'collapsed' : 'expanded');
  }

  openMobile(): void  { this._mobileOpen.set(true);  }
  closeMobile(): void { this._mobileOpen.set(false); }

  private readInitialMode(): SidebarMode {
    const params = new URLSearchParams(window.location.search);
    const fromUrl = params.get(PARAM_KEY);
    if (fromUrl === 'expanded' || fromUrl === 'collapsed') return fromUrl;
    try {
      const stored = localStorage.getItem(STORAGE_KEY);
      if (stored === 'expanded' || stored === 'collapsed') return stored;
    } catch { /* ignore */ }
    return 'expanded';
  }
}
```

### App Shell Component

```typescript
// app-shell.component.ts
import {
  Component, OnInit, OnDestroy, inject,
  HostListener, ElementRef, ViewChild, effect, AfterViewInit
} from '@angular/core';
import { NgIf, NgFor } from '@angular/common';
import { RouterOutlet, RouterLink, RouterLinkActive } from '@angular/router';
import { SidebarService } from './sidebar.service';

interface NavItem {
  id: string;
  label: string;
  href: string;
  icon: string;
}

const NAV_ITEMS: NavItem[] = [
  { id: 'dashboard', label: 'Dashboard', href: '/dashboard', icon: '⊞' },
  { id: 'reports',   label: 'Reports',   href: '/reports',   icon: '◈' },
  { id: 'settings',  label: 'Settings',  href: '/settings',  icon: '⚙' },
];

@Component({
  selector: 'app-shell',
  standalone: true,
  imports: [NgIf, NgFor, RouterOutlet, RouterLink, RouterLinkActive],
  template: `
    <div class="app-shell" [attr.data-sidebar]="sidebar.shellAttr()">

      <!-- Header -->
      <header class="app-header">
        <button
          #toggleBtn
          class="sidebar-toggle"
          [attr.aria-label]="sidebar.isExpanded() ? 'Collapse sidebar' : 'Expand sidebar'"
          [attr.aria-expanded]="sidebar.isExpanded()"
          aria-controls="app-sidebar"
          (click)="handleToggle()"
        >
          <span aria-hidden="true">{{ sidebar.isExpanded() ? '◀' : '▶' }}</span>
        </button>
        <span class="app-logo">MyApp</span>
      </header>

      <!-- Sidebar -->
      <aside
        id="app-sidebar"
        #sidebarEl
        class="app-sidebar"
        aria-label="Application navigation"
        [attr.aria-hidden]="sidebar.isMobile() && !sidebar.mobileOpen() ? true : null"
        [attr.role]="sidebar.isMobile() ? 'dialog' : null"
        [attr.aria-modal]="sidebar.isMobile() ? true : null"
      >
        <button
          *ngIf="sidebar.isMobile()"
          class="sidebar-close-btn"
          aria-label="Close navigation"
          (click)="sidebar.closeMobile()"
        >✕</button>

        <nav aria-label="Main">
          <ul role="list" style="list-style:none;margin:0;padding:.5rem">
            <li *ngFor="let item of navItems; trackBy: trackById">
              <a
                class="nav-link"
                [routerLink]="item.href"
                routerLinkActive="active"
                [attr.title]="isIconOnly() ? item.label : null"
              >
                <span class="nav-icon" aria-hidden="true">{{ item.icon }}</span>
                <span class="nav-label">{{ item.label }}</span>
              </a>
            </li>
          </ul>
        </nav>
      </aside>

      <!-- Overlay backdrop -->
      <div
        class="sidebar-overlay"
        aria-hidden="true"
        (click)="sidebar.closeMobile()"
      ></div>

      <!-- Main content -->
      <main id="main-content" class="app-main" tabindex="-1">
        <router-outlet />
      </main>
    </div>
  `,
})
export class AppShellComponent implements OnInit, OnDestroy, AfterViewInit {
  sidebar   = inject(SidebarService);
  navItems  = NAV_ITEMS;

  @ViewChild('toggleBtn') toggleBtn!: ElementRef<HTMLButtonElement>;
  @ViewChild('sidebarEl') sidebarEl!: ElementRef<HTMLElement>;

  private focusTrapHandler?: (e: KeyboardEvent) => void;
  private savedScrollY = 0;

  constructor() {
    // React to mobile overlay open/close
    effect(() => {
      const open   = this.sidebar.mobileOpen();
      const mobile = this.sidebar.isMobile();

      if (mobile && open) {
        this.lockBodyScroll();
        // Defer focus trap setup until DOM is updated
        setTimeout(() => this.activateFocusTrap(), 0);
      } else {
        this.deactivateFocusTrap();
        this.unlockBodyScroll();
        if (mobile && !open) {
          this.toggleBtn?.nativeElement.focus();
        }
      }
    });
  }

  ngOnInit(): void {
    this.sidebar.initMediaQuery();
  }

  ngAfterViewInit(): void { /* ViewChild refs now available */ }

  ngOnDestroy(): void {
    this.deactivateFocusTrap();
    this.unlockBodyScroll();
  }

  handleToggle(): void {
    this.sidebar.isMobile()
      ? this.sidebar.openMobile()
      : this.sidebar.toggleDesktop();
  }

  isIconOnly(): boolean {
    return !this.sidebar.isMobile() && this.sidebar.mode() === 'collapsed';
  }

  trackById(_: number, item: NavItem): string {
    return item.id;
  }

  @HostListener('document:keydown.escape')
  onEscape(): void {
    if (this.sidebar.mobileOpen()) this.sidebar.closeMobile();
  }

  private activateFocusTrap(): void {
    const el = this.sidebarEl?.nativeElement;
    if (!el) return;

    const FOCUSABLE =
      'a[href], button:not([disabled]), input:not([disabled]), [tabindex]:not([tabindex="-1"])';
    const nodes = Array.from(el.querySelectorAll<HTMLElement>(FOCUSABLE));
    const first = nodes[0];
    const last  = nodes[nodes.length - 1];
    first?.focus();

    this.focusTrapHandler = (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return;
      if (e.shiftKey) {
        if (document.activeElement === first) { e.preventDefault(); last?.focus(); }
      } else {
        if (document.activeElement === last)  { e.preventDefault(); first?.focus(); }
      }
    };
    document.addEventListener('keydown', this.focusTrapHandler);
  }

  private deactivateFocusTrap(): void {
    if (this.focusTrapHandler) {
      document.removeEventListener('keydown', this.focusTrapHandler);
      this.focusTrapHandler = undefined;
    }
  }

  private lockBodyScroll(): void {
    this.savedScrollY = window.scrollY;
    document.body.style.overflow = 'hidden';
    document.body.style.position = 'fixed';
    document.body.style.top      = `-${this.savedScrollY}px`;
    document.body.style.width    = '100%';
  }

  private unlockBodyScroll(): void {
    document.body.style.overflow = '';
    document.body.style.position = '';
    document.body.style.top      = '';
    document.body.style.width    = '';
    window.scrollTo(0, this.savedScrollY);
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Toggle button has `aria-expanded` reflecting sidebar state | JS updates on every toggle |
| Toggle button has `aria-controls` pointing to sidebar `id` | Static attribute `aria-controls="app-sidebar"` |
| Sidebar gets `role="dialog"` and `aria-modal="true"` in overlay mode | Applied conditionally when `isMobile` is true |
| Sidebar `aria-hidden="true"` when mobile overlay is closed | Prevents virtual cursor from entering hidden panel |
| Focus trapped inside sidebar when overlay is open | Tab/Shift+Tab cycle constrained within sidebar DOM |
| Escape key closes mobile overlay | `keydown` listener on document, cleaned up on close |
| Focus returns to toggle button when overlay closes | `useEffect` / `effect()` triggers `.focus()` after close |
| Collapsed icon-only rail still has text labels for screen readers | `nav-label` spans remain in DOM; only `opacity: 0` + `pointer-events: none` hides them visually |
| Icon-only tooltips for mouse users on collapsed rail | `title` attribute added when sidebar is collapsed |
| Active page link announced to screen readers | `aria-current="page"` on current nav link |
| Sidebar close button is first focusable element in mobile overlay | Rendered before the nav list |
| All interactive elements meet 44×44px minimum | Enforced via `min-height` / `width`/`height` in CSS |
| Body scroll locked while mobile overlay is open | `position: fixed` + `overflow: hidden` applied to body |
| Animations respect `prefers-reduced-motion` | `@media (prefers-reduced-motion: reduce)` disables all transitions |

---

## Production Pitfalls

**1. CSS Grid column transition flickers a scrollbar**
When the sidebar column animates from 260px to 60px, the main content area briefly becomes narrower than its content, causing a horizontal scrollbar to flash. Fix: add `overflow-x: hidden` to `.app-main` and ensure content inside uses `max-width: 100%`.

**2. `localStorage` throws in third-party iframe contexts**
Calling `localStorage.setItem` in a cross-origin iframe throws a `SecurityError`. Always wrap every `localStorage` call in a try/catch. If storage is unavailable, fall back gracefully — the UI still works, it just does not persist the preference.

**3. URL `searchParams` causes a mismatch after browser back navigation**
If you use `history.replaceState` to sync the sidebar mode to the URL, clicking the browser back button changes the URL but does not trigger your React/Angular state update. Fix: listen to the `popstate` event and re-read the URL param to reconcile state:
```ts
window.addEventListener('popstate', () => {
  const params = new URLSearchParams(window.location.search);
  const mode = params.get('sidebar');
  if (mode === 'expanded' || mode === 'collapsed') setMode(mode);
});
```

**4. Icon-only rail is inaccessible without tooltip text**
When the sidebar collapses, sighted users see icons. Screen reader users still hear the `nav-label` text because it is in the DOM. However, keyboard-only mouse-visual hybrid users (sighted keyboard users) have no way to discover icon meaning without hover. Fix: add `title` attribute to icon-only links as a minimum; prefer a proper tooltip component with `aria-describedby` for richer experiences.

**5. `sticky` sidebar does not account for a page that has a sticky subheader**
If you add a secondary toolbar below the main header, the sticky sidebar `top: var(--header-height)` calculation breaks. Fix: use a CSS custom property `--sticky-offset` that you update via JS when a subheader mounts, or use CSS `calc()` with multiple named properties so all sticky elements coordinate.

**6. Mobile overlay re-opens on hot-module reload in development**
If your `mobileOpen` state is preserved across HMR cycles, the overlay stays open but the focus trap handler is gone (it was registered in the previous module instance). Fix: always close the sidebar programmatically in a React `useEffect` cleanup or Angular `ngOnDestroy` so HMR produces a clean slate.

**7. `aria-hidden` on the sidebar breaks when a portal renders inside it**
If a tooltip or dropdown inside the sidebar uses a React/Angular portal to escape the sidebar DOM (common with floating-ui or CDK overlay), those portal elements will not inherit `aria-hidden="true"` from the sidebar. They will remain reachable by screen readers even when the sidebar is logically hidden. Fix: close all popovers before the sidebar hides, or apply `aria-hidden` to the portal element explicitly.

---

## Interview Angle

**Q: "How would you build a collapsible sidebar that persists its state and can be shared via a URL link?"**

The answer requires covering three separate concerns. First, the layout layer: a CSS Grid shell with named template areas and a custom property (`--sidebar-col-width`) that drives the column width. JS updates the custom property (or a `data-sidebar` attribute) so the grid reflows via a CSS `transition`. CSS owns the animation; JS owns the trigger. Second, the persistence layer: URL `searchParams` are the primary source of truth because they are shareable and survive hard refreshes. `localStorage` is the fallback for sessions where the URL was never manually set. Read priority is URL → localStorage → default. Writes go to both simultaneously via `history.replaceState` (no page reload) and `localStorage.setItem`. Third, handle the `popstate` event so that the browser Back button reconciles state instead of leaving a stale URL with the wrong sidebar position.

**Follow-up: "How does the sidebar change on mobile and what accessibility work does that require?"**

On mobile the sidebar becomes a modal overlay — it floats above the content on a fixed layer instead of being inline with the grid. That shift from inline to modal changes the ARIA contract entirely: the sidebar now needs `role="dialog"` and `aria-modal="true"`, a focus trap must activate (Tab cycles stay inside the sidebar), the trigger gets an updated `aria-expanded`, the sidebar gets `aria-hidden="true"` when closed, and Escape must close it. The body needs scroll-lock to prevent the content behind the overlay from scrolling. None of that ARIA or focus work is needed in the desktop inline variant — the two modes share HTML and CSS but require genuinely different JS behavior.
