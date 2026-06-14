# Skip Links & Focus Management on Route Change

## The Idea

**In plain English:** When a screen reader or keyboard user lands on your page, they are dumped at the top of the DOM every time — including your 40-item global navigation. Without skip links they must press Tab dozens of times before reaching the actual content. In a Single Page Application the problem doubles: the page URL changes but the browser never reloads, so focus stays wherever it was — usually inside the navigation link that was just clicked, now orphaned inside a component that has been unmounted.

**Real-world analogy:** Imagine a building with a long corridor of staff offices before the main hall.

- The **"Skip to main content" link** = a side door at the start of the corridor that lets you bypass every office and step straight into the hall.
- The **page title announcement** = the receptionist who reads the room name over the intercom each time you enter a new hall — so you always know where you are.
- **SPA route change without focus management** = the side door leading you into a random supply cupboard instead of the new hall, with no intercom announcement at all.
- **Programmatic focus on `<main>`** = the guide who grabs your elbow after the door and says "you're now in the Conference Room — take it from here."

The key insight: a visual user sees the new page instantly; a screen reader user needs an explicit, programmed hand-off — a focus target, an announcement, or both.

---

## Learning Objectives

- Understand why skip links must be the first focusable element in the DOM
- Implement a visible-on-focus skip link that does not break layout
- Know the two SPA focus-management strategies: focus the `<main>` landmark vs. focus an `<h1>` heading
- Announce the new page title using `aria-live` without double-announcing
- Integrate focus management into React Router v6 and Angular Router
- Avoid the common pitfalls: `tabindex="-1"` omission, double announcements, scroll-but-no-focus

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Show skip link only on keyboard focus | ✅ with `:focus` / `:focus-visible` | — |
| Move focus to `<main>` after route change | ❌ | `.focus()` is a JS imperative API |
| Announce new page title to screen reader | ❌ | Requires an `aria-live` region updated by JS |
| Detect when a route change has completed | ❌ | Requires router event subscription |
| Manage `tabindex` on non-interactive elements | ❌ | Attribute mutation needs JS |
| Scroll `<main>` into view after focus | ❌ | `scrollIntoView()` is JS |

**Conclusion:** CSS owns the skip link's visual reveal on focus. Every action that happens after a route change — focus placement, title announcement, scroll — belongs to JavaScript.

---

## HTML & CSS Foundation

```html
<!-- Must be the very first element inside <body> -->
<a class="skip-link" href="#main-content">Skip to main content</a>

<header><!-- global nav --></header>

<!-- tabindex="-1" allows programmatic focus; the id is the skip link target -->
<main id="main-content" tabindex="-1">
  <h1>Page Title</h1>
  <!-- page content -->
</main>

<!-- aria-live region for SPA route announcements -->
<!-- kept empty; JS writes the new page title into it after each navigation -->
<div
  class="sr-only"
  aria-live="polite"
  aria-atomic="true"
  id="route-announcer"
></div>
```

```css
/* ── Skip link: visually hidden until focused ── */
.skip-link {
  position: absolute;
  top: -100%;
  left: 0;
  z-index: 9999;
  padding: 0.75rem 1.25rem;
  background: #1a1a1a;
  color: #fff;
  font-weight: 600;
  font-size: 1rem;
  text-decoration: none;
  border-radius: 0 0 4px 0;
  /* Use top offset rather than transform so it doesn't
     interfere with other stacking contexts */
  transition: top 0.15s;
}

.skip-link:focus {
  top: 0;
  outline: 3px solid #ffbf47;
  outline-offset: 2px;
}

/* ── Screen-reader-only utility ── */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

/* ── Remove the focus ring on <main> — it was focused programmatically,
   not by the user pressing Tab ── */
main:focus {
  outline: none;
}
```

**The `tabindex="-1"` rule:** Adding it to `<main>` lets JavaScript call `.focus()` on it. Without it, most browsers silently ignore `.focus()` calls on non-interactive elements. It does NOT put `<main>` in the Tab order — that remains unchanged.

---

## React Implementation

### The Route Announcer Hook

```tsx
// hooks/useRouteAnnouncer.ts
import { useEffect, useRef } from 'react';
import { useLocation } from 'react-router-dom';

/**
 * After every route change:
 * 1. Write the new document.title into an aria-live region.
 * 2. Move focus to <main> so keyboard users start from the top of the content.
 *
 * Call this hook once in your root layout component.
 */
export function useRouteAnnouncer(): void {
  const location = useLocation();
  const announcerRef = useRef<HTMLDivElement | null>(null);

  useEffect(() => {
    // Lazily find or create the announcer element.
    // We look it up by id rather than keeping a ref so this hook
    // stays decoupled from any particular JSX tree.
    let announcer = document.getElementById('route-announcer') as HTMLDivElement | null;

    if (!announcer) {
      announcer = document.createElement('div');
      announcer.id = 'route-announcer';
      announcer.setAttribute('aria-live', 'polite');
      announcer.setAttribute('aria-atomic', 'true');
      announcer.className = 'sr-only';
      document.body.appendChild(announcer);
    }

    announcerRef.current = announcer;
  }, []);

  useEffect(() => {
    const announcer = announcerRef.current;
    if (!announcer) return;

    // Clear first so screen readers detect a change even when
    // navigating to the same route (e.g. search filters updating the URL).
    announcer.textContent = '';

    // Defer so the new page's <title> has had a chance to update.
    const timer = setTimeout(() => {
      announcer.textContent = document.title || 'Page loaded';

      // Move focus to <main> so keyboard users land at the top of
      // the new content rather than wherever they were before.
      const main = document.getElementById('main-content');
      if (main) {
        main.focus({ preventScroll: false });
      }
    }, 100);

    return () => clearTimeout(timer);
  }, [location.pathname, location.search]);
}
```

### Root Layout Component

```tsx
// components/RootLayout.tsx
import { Outlet, NavLink } from 'react-router-dom';
import { useRouteAnnouncer } from '../hooks/useRouteAnnouncer';

export function RootLayout() {
  useRouteAnnouncer();

  return (
    <>
      {/* 1. Skip link — must come first in the DOM */}
      <a className="skip-link" href="#main-content">
        Skip to main content
      </a>

      <header>
        <nav aria-label="Main navigation">
          <ul>
            <li><NavLink to="/">Home</NavLink></li>
            <li><NavLink to="/products">Products</NavLink></li>
            <li><NavLink to="/about">About</NavLink></li>
          </ul>
        </nav>
      </header>

      {/* 2. Main landmark with tabindex="-1" for programmatic focus */}
      <main id="main-content" tabIndex={-1}>
        <Outlet />
      </main>

      {/* 3. aria-live announcer (sr-only, populated by the hook) */}
      <div
        id="route-announcer"
        className="sr-only"
        aria-live="polite"
        aria-atomic="true"
      />
    </>
  );
}
```

### App Entry with Router

```tsx
// App.tsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';
import { RootLayout } from './components/RootLayout';
import { HomePage } from './pages/HomePage';
import { ProductsPage } from './pages/ProductsPage';

// Each page component should set document.title in a useEffect or
// via a library like react-helmet-async.
const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    children: [
      { index: true, element: <HomePage /> },
      { path: 'products', element: <ProductsPage /> },
    ],
  },
]);

export function App() {
  return <RouterProvider router={router} />;
}
```

### Per-Page Title Management

```tsx
// hooks/usePageTitle.ts
import { useEffect } from 'react';

/**
 * Sets document.title for the current page.
 * Format: "Page Name | Site Name"
 */
export function usePageTitle(pageTitle: string, siteName = 'My App'): void {
  useEffect(() => {
    document.title = `${pageTitle} | ${siteName}`;
  }, [pageTitle, siteName]);
}

// Usage inside a page component:
// function ProductsPage() {
//   usePageTitle('Products');
//   return <h1>Products</h1>;
// }
```

---

## Angular Implementation

### Route Announcer Service

```typescript
// services/route-announcer.service.ts
import { Injectable, OnDestroy } from '@angular/core';
import { Router, NavigationEnd } from '@angular/router';
import { filter, Subscription } from 'rxjs';
import { Title } from '@angular/platform-browser';

@Injectable({ providedIn: 'root' })
export class RouteAnnouncerService implements OnDestroy {
  private subscription = new Subscription();
  private announcer: HTMLElement | null = null;

  constructor(private router: Router, private titleService: Title) {}

  /** Call once from AppComponent.ngOnInit() */
  init(): void {
    this.announcer = this.getOrCreateAnnouncer();

    this.subscription.add(
      this.router.events
        .pipe(filter(e => e instanceof NavigationEnd))
        .subscribe(() => {
          // Clear first so the same text re-triggers if navigating
          // to the same route with different params.
          if (this.announcer) this.announcer.textContent = '';

          setTimeout(() => {
            const title = this.titleService.getTitle();
            if (this.announcer) {
              this.announcer.textContent = title || 'Page loaded';
            }

            // Move focus to the main landmark
            const main = document.getElementById('main-content');
            if (main) main.focus({ preventScroll: false });
          }, 100);
        })
    );
  }

  private getOrCreateAnnouncer(): HTMLElement {
    let el = document.getElementById('route-announcer');
    if (!el) {
      el = document.createElement('div');
      el.id = 'route-announcer';
      el.setAttribute('aria-live', 'polite');
      el.setAttribute('aria-atomic', 'true');
      el.className = 'sr-only';
      document.body.appendChild(el);
    }
    return el;
  }

  ngOnDestroy(): void {
    this.subscription.unsubscribe();
  }
}
```

### App Component

```typescript
// app.component.ts
import { Component, OnInit } from '@angular/core';
import { RouterOutlet } from '@angular/router';
import { RouteAnnouncerService } from './services/route-announcer.service';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet],
  template: `
    <!-- Skip link — first element in the DOM -->
    <a class="skip-link" href="#main-content">Skip to main content</a>

    <header>
      <nav aria-label="Main navigation">
        <ul>
          <li><a routerLink="/">Home</a></li>
          <li><a routerLink="/products">Products</a></li>
        </ul>
      </nav>
    </header>

    <!-- tabindex="-1" so JS can call .focus() on it -->
    <main id="main-content" tabindex="-1">
      <router-outlet />
    </main>

    <div
      id="route-announcer"
      class="sr-only"
      aria-live="polite"
      aria-atomic="true"
    ></div>
  `,
})
export class AppComponent implements OnInit {
  constructor(private announcer: RouteAnnouncerService) {}

  ngOnInit(): void {
    this.announcer.init();
  }
}
```

### Per-Route Title via Router Data

```typescript
// app.routes.ts
import { Routes } from '@angular/router';
import { inject } from '@angular/core';
import { Title } from '@angular/platform-browser';

// A reusable resolver that sets the page title from route data
export function titleResolver(route: import('@angular/router').ActivatedRouteSnapshot) {
  const title = inject(Title);
  const pageTitle = route.data['title'] as string | undefined;
  if (pageTitle) title.setTitle(`${pageTitle} | My App`);
  return true;
}

export const routes: Routes = [
  { path: '', loadComponent: () => import('./pages/home.component').then(m => m.HomeComponent),
    data: { title: 'Home' }, resolve: { title: titleResolver } },
  { path: 'products', loadComponent: () => import('./pages/products.component').then(m => m.ProductsComponent),
    data: { title: 'Products' }, resolve: { title: titleResolver } },
];
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Skip link is the first focusable element | First child of `<body>` in DOM order |
| Skip link target has `tabindex="-1"` | `<main tabindex="-1">` or `id="main-content"` |
| Skip link is visible when focused | CSS `:focus` rule moves it from `top: -100%` to `top: 0` |
| Skip link uses `href="#id"` pointing to main | ID matches `<main id="main-content">` |
| Page title updated after route change | `document.title` set in page component / resolver |
| New page title announced to screen reader | `aria-live="polite"` region, cleared then re-set after 100ms |
| Focus moved to `<main>` after route change | `.focus()` called on the `<main>` element |
| `<main>` focus ring suppressed | `main:focus { outline: none }` — was focused by JS, not Tab |
| Announcer does not double-announce | Text is cleared before re-set so both NVDA and VoiceOver pick up the change |
| Works on initial page load | First navigation is also a `NavigationEnd` event in Angular; React hook fires on mount |

---

## Production Pitfalls

**1. The 100ms timeout is load-order-dependent**
If a page component sets `document.title` inside a `useEffect` (which runs after paint), the announcer fires before the title has updated. Fix: either use a route-level resolver that sets the title synchronously before the component mounts, or increase the timeout to 200ms as a last resort. Prefer the resolver approach.

**2. `<main>` without `tabindex="-1"` silently swallows `.focus()` calls**
In most browsers, calling `.focus()` on a non-interactive element that lacks `tabindex` simply does nothing — no error, no warning. The cursor stays wherever it was. Always add `tabindex="-1"` to your focus target.

**3. Hash-change navigation is not a route change**
If you use `href="#section"` anchor links within a page, the `NavigationEnd` event also fires in Angular (or `location` changes in React Router). You will falsely announce "Products | My App" when the user merely jumped to a section heading. Fix: compare `location.pathname` before and after — only announce when the path changes.

**4. Double announcement in NVDA + Chrome**
Some screen reader + browser combinations announce the `<title>` element change AND the `aria-live` region. Fix: prepend the word "Navigated to" to your `aria-live` text so even if both fire, the user gets useful, non-duplicate information. e.g. "Navigated to Products | My App".

**5. focus() scrolls past sticky headers**
If you have a sticky `<header>`, focusing `<main>` scrolls it to the very top, hiding the first heading behind the header. Fix: use `scroll-margin-top` on `<main>` matching the header height, or focus the first `<h1>` inside `<main>` rather than `<main>` itself.

**6. Server-side rendering: announcer div must not be pre-rendered**
If the `aria-live` div is server-rendered with content already in it, screen readers may not pick up the first client-side update. Render it empty (`textContent = ''`) or create it in a client-only effect.

---

## Interview Angle

**Q: "How do you handle focus management in a React SPA after a route change?"**

Strong answer covers three layers:
1. **The problem** — browser reloads reset focus to the top; SPA route changes do not. Focus stays on the old navigation link. Screen reader users are left stranded with no indication that the page changed.
2. **The mechanism** — listen to `NavigationEnd` (Angular) or `location` changes (React Router `useLocation`). After the event fires, call `.focus()` on `<main id="main-content" tabindex="-1">`. Separately, update an `aria-live="polite"` region with the new page title.
3. **The nuances** — the 100ms deferral to let the title update first; the clear-then-set pattern to re-trigger the live region on same-route navigations; `scroll-margin-top` if there is a sticky header; not re-announcing on hash-only changes.

**Q: "What's wrong with a skip link that uses `display: none` when not focused?"**

`display: none` removes the element from the accessibility tree entirely, including when it receives focus. Screen readers will not announce it. The correct pattern is to position it off-screen with `position: absolute; top: -100%` or the `.sr-only` clip technique, then bring it on-screen on `:focus`.
