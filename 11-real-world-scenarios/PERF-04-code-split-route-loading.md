# Code-Split Route with Loading UI

## The Idea

**In plain English:** When a user navigates to `/dashboard`, the browser should not have to download the code for every other page in your app. Code splitting means each route's JavaScript is bundled separately and downloaded only when needed. The moment the user clicks a link, there is a gap — the chunk is downloading. You need to show something useful during that gap, and handle the case where the chunk fails to load entirely.

**Real-world analogy:** Think of a large reference book.

- A monolithic bundle is printing the **entire encyclopedia** and handing it to the user at the door. They wait a long time upfront, but once inside every page is instant.
- Code splitting is a **library with books on shelves**. You get the index (main bundle) instantly. When you want Chapter 7, the librarian fetches it. There is a brief wait, but you only ever carry the chapter you need.
- The **"Fetching your book…" sign at the desk** is the Suspense fallback / skeleton loader — it tells the user something is happening.
- The **"Sorry, that book is damaged"** sign is the error boundary — it catches chunk load failures (CDN errors, stale deploys) and shows a recoverable error instead of a blank white screen.
- `React.lazy` is the library catalogue. `@defer` in Angular is the same concept.

---

## Learning Objectives

- Understand what a code-split chunk is and when the browser fetches it
- Use `React.lazy` + `Suspense` to lazy-load route components
- Build a skeleton loader that matches the page's layout (avoid layout shift)
- Implement an `ErrorBoundary` that catches `ChunkLoadError` specifically
- Use Angular's `@defer` block with `@loading`, `@error`, and `@placeholder` subtemplates
- Understand the retry pattern for transient chunk load failures

---

## Why CSS Alone Isn't Enough

Not applicable — code splitting is a build-tool and framework concern. CSS owns only the skeleton shimmer animation.

---

## HTML & CSS Foundation

### Skeleton Loader

```html
<!-- Page-level skeleton that mimics the dashboard layout -->
<div class="skeleton-page" aria-hidden="true" aria-label="Loading dashboard">
  <!-- Header skeleton -->
  <div class="skeleton-bar skeleton-bar--title"></div>

  <!-- Card grid skeleton -->
  <div class="skeleton-grid">
    <div class="skeleton-card">
      <div class="skeleton-bar skeleton-bar--label"></div>
      <div class="skeleton-bar skeleton-bar--value"></div>
    </div>
    <div class="skeleton-card">
      <div class="skeleton-bar skeleton-bar--label"></div>
      <div class="skeleton-bar skeleton-bar--value"></div>
    </div>
    <div class="skeleton-card">
      <div class="skeleton-bar skeleton-bar--label"></div>
      <div class="skeleton-bar skeleton-bar--value"></div>
    </div>
  </div>

  <!-- Content block skeleton -->
  <div class="skeleton-bar skeleton-bar--body"></div>
  <div class="skeleton-bar skeleton-bar--body skeleton-bar--short"></div>
</div>
```

```css
:root {
  --skeleton-base: #e5e7eb;
  --skeleton-shine: #f3f4f6;
  --skeleton-radius: 6px;
  --skeleton-anim-duration: 1.5s;
}

/* Shimmer keyframe */
@keyframes skeleton-shimmer {
  0%   { background-position: -200% 0; }
  100% { background-position:  200% 0; }
}

.skeleton-bar {
  border-radius: var(--skeleton-radius);
  background: linear-gradient(
    90deg,
    var(--skeleton-base) 25%,
    var(--skeleton-shine) 50%,
    var(--skeleton-base) 75%
  );
  background-size: 200% 100%;
  animation: skeleton-shimmer var(--skeleton-anim-duration) ease-in-out infinite;
}

.skeleton-bar--title  { height: 32px; width: 40%;  margin-bottom: 1.5rem; }
.skeleton-bar--label  { height: 14px; width: 60%;  margin-bottom: 0.5rem; }
.skeleton-bar--value  { height: 28px; width: 80%; }
.skeleton-bar--body   { height: 16px; width: 100%; margin-bottom: 0.75rem; }
.skeleton-bar--short  { width: 65%; }

.skeleton-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 1rem;
  margin-bottom: 2rem;
}

.skeleton-card {
  padding: 1.25rem;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
}

/* Error state */
.chunk-error {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 3rem 1rem;
  text-align: center;
  gap: 1rem;
}

.chunk-error__heading { font-size: 1.25rem; font-weight: 600; }
.chunk-error__message { color: #6b7280; }

.btn-retry {
  padding: 0.5rem 1.25rem;
  background: #3b82f6;
  color: #fff;
  border: none;
  border-radius: 6px;
  font-size: 0.9rem;
  cursor: pointer;
}

@media (prefers-reduced-motion: reduce) {
  .skeleton-bar { animation: none; }
}
```

---

## React Implementation

### Lazy Routes with Suspense

```tsx
// router.tsx  (React Router v6 example)
import { lazy, Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { DashboardSkeleton } from './DashboardSkeleton';
import { ChunkErrorBoundary } from './ChunkErrorBoundary';

// Webpack/Vite will split each of these into a separate chunk
const Dashboard  = lazy(() => import('./pages/Dashboard'));
const Analytics  = lazy(() => import('./pages/Analytics'));
const Settings   = lazy(() => import('./pages/Settings'));

// Preload on hover (optional perf win)
const preload = (factory: () => Promise<unknown>) => () => { factory(); };

export function AppRouter() {
  return (
    <BrowserRouter>
      <Routes>
        <Route
          path="/dashboard"
          element={
            <ChunkErrorBoundary>
              <Suspense fallback={<DashboardSkeleton />}>
                <Dashboard />
              </Suspense>
            </ChunkErrorBoundary>
          }
        />
        <Route
          path="/analytics"
          element={
            <ChunkErrorBoundary>
              <Suspense fallback={<DashboardSkeleton />}>
                <Analytics />
              </Suspense>
            </ChunkErrorBoundary>
          }
        />
        <Route
          path="/settings"
          element={
            <ChunkErrorBoundary>
              <Suspense fallback={<DashboardSkeleton />}>
                <Settings />
              </Suspense>
            </ChunkErrorBoundary>
          }
        />
      </Routes>
    </BrowserRouter>
  );
}
```

### Skeleton Component

```tsx
// DashboardSkeleton.tsx
export function DashboardSkeleton() {
  return (
    <div className="skeleton-page" aria-busy="true" aria-label="Loading page content">
      <div className="skeleton-bar skeleton-bar--title" />
      <div className="skeleton-grid">
        {[0, 1, 2].map(i => (
          <div key={i} className="skeleton-card">
            <div className="skeleton-bar skeleton-bar--label" />
            <div className="skeleton-bar skeleton-bar--value" />
          </div>
        ))}
      </div>
      <div className="skeleton-bar skeleton-bar--body" />
      <div className="skeleton-bar skeleton-bar--body skeleton-bar--short" />
    </div>
  );
}
```

### Error Boundary for Chunk Load Failures

```tsx
// ChunkErrorBoundary.tsx
import { Component, type ReactNode } from 'react';

interface Props { children: ReactNode; }
interface State { hasError: boolean; isChunkError: boolean; retryCount: number; }

// Webpack chunk load errors have a specific name
function isChunkLoadError(error: unknown): boolean {
  if (!(error instanceof Error)) return false;
  return (
    error.name === 'ChunkLoadError' ||
    /Loading chunk \d+ failed/.test(error.message) ||
    /Loading CSS chunk \d+ failed/.test(error.message)
  );
}

export class ChunkErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false, isChunkError: false, retryCount: 0 };

  static getDerivedStateFromError(error: unknown): Partial<State> {
    return {
      hasError: true,
      isChunkError: isChunkLoadError(error),
    };
  }

  handleRetry = () => {
    // Force a page reload on chunk errors — the chunk might exist in the new deploy
    if (this.state.isChunkError) {
      window.location.reload();
      return;
    }
    this.setState(prev => ({
      hasError: false,
      retryCount: prev.retryCount + 1,
    }));
  };

  render() {
    if (!this.state.hasError) return this.props.children;

    return (
      <div className="chunk-error" role="alert">
        <h2 className="chunk-error__heading">
          {this.state.isChunkError ? 'Page failed to load' : 'Something went wrong'}
        </h2>
        <p className="chunk-error__message">
          {this.state.isChunkError
            ? 'A new version of the app may have been deployed. Refreshing will fix this.'
            : 'An unexpected error occurred. Please try again.'}
        </p>
        <button className="btn-retry" onClick={this.handleRetry}>
          {this.state.isChunkError ? 'Refresh page' : 'Try again'}
        </button>
      </div>
    );
  }
}
```

### Preloading Chunks on Link Hover (Performance)

```tsx
// NavLink with preload on hover
import { lazy } from 'react';
import { Link } from 'react-router-dom';

const dashboardChunk = () => import('./pages/Dashboard');

export function NavLink() {
  return (
    <Link
      to="/dashboard"
      onMouseEnter={() => dashboardChunk()} // kick off the download 100-200ms early
      onFocus={() => dashboardChunk()}
    >
      Dashboard
    </Link>
  );
}
```

---

## Angular Implementation

### `@defer` with Loading, Error, and Placeholder

```typescript
// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: 'dashboard',
    // Route-level code splitting: Angular CLI splits this into its own chunk
    loadComponent: () =>
      import('./pages/dashboard/dashboard.component')
        .then(m => m.DashboardComponent),
  },
  {
    path: 'analytics',
    loadComponent: () =>
      import('./pages/analytics/analytics.component')
        .then(m => m.AnalyticsComponent),
  },
];
```

### `@defer` for Feature-Level Splitting Inside a Component

```typescript
// dashboard.component.ts
import { Component } from '@angular/core';
import { DashboardSkeletonComponent } from './dashboard-skeleton.component';

@Component({
  selector: 'app-dashboard',
  standalone: true,
  imports: [DashboardSkeletonComponent],
  template: `
    <h1>Dashboard</h1>

    <!--
      @defer defers loading the HeavyChartComponent chunk until the
      trigger fires. Angular generates a separate JS chunk for it.
    -->
    @defer (on viewport) {
      <!-- Component loaded from its own chunk -->
      <app-heavy-chart />
    } @loading (minimum 300ms) {
      <!-- Shown while the chunk + component initialize -->
      <app-dashboard-skeleton aria-busy="true" />
    } @error {
      <!-- Shown if the chunk download fails -->
      <div class="chunk-error" role="alert">
        <p class="chunk-error__heading">Chart failed to load</p>
        <button class="btn-retry" (click)="reloadChart()">Retry</button>
      </div>
    } @placeholder {
      <!-- Shown before the trigger fires (before viewport entry) -->
      <div class="placeholder-chart" aria-hidden="true">
        Chart will load when scrolled into view
      </div>
    }
  `,
})
export class DashboardComponent {
  reloadChart() {
    // Trigger a re-evaluation of the defer block
    window.location.reload();
  }
}
```

### Skeleton Component

```typescript
// dashboard-skeleton.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-dashboard-skeleton',
  standalone: true,
  template: `
    <div class="skeleton-page">
      <div class="skeleton-bar skeleton-bar--title"></div>
      <div class="skeleton-grid">
        @for (n of [0,1,2]; track n) {
          <div class="skeleton-card">
            <div class="skeleton-bar skeleton-bar--label"></div>
            <div class="skeleton-bar skeleton-bar--value"></div>
          </div>
        }
      </div>
      <div class="skeleton-bar skeleton-bar--body"></div>
      <div class="skeleton-bar skeleton-bar--body skeleton-bar--short"></div>
    </div>
  `,
})
export class DashboardSkeletonComponent {}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Skeleton is hidden from screen readers | `aria-hidden="true"` on skeleton root — it conveys no real content |
| Loading state is communicated | `aria-busy="true"` on the container that will hold the loaded content |
| Error state uses `role="alert"` | Announced immediately by screen readers |
| Retry button is focusable and labeled | Standard `<button>` with descriptive text |
| Page title updates after navigation | React Router / Angular Router manage `<title>` updates — verify with `@angular/router` `TitleStrategy` |
| No layout shift during skeleton → content swap | Skeleton dimensions match real content dimensions |

---

## Production Pitfalls

**1. `ChunkLoadError` after a new deploy**
When you deploy a new build, old chunk filenames are invalidated. Users with the old app open in their browser will get 404s when trying to load new chunks. The `ChunkErrorBoundary` detects this and offers a "Refresh page" button instead of a generic error.

**2. Suspense fallback flashes for fast connections**
On fast connections, the chunk downloads in under 50ms. The skeleton flashes briefly — more jarring than just waiting. Use a `minimum` display time in Angular's `@loading (minimum 300ms)` or a `startTransition` / `useTransition` wrapper in React to defer showing the spinner at all.

**3. Error boundary must be a class component in React**
`getDerivedStateFromError` and `componentDidCatch` have no hooks equivalent. The boundary must be a class component. Keep one well-tested boundary and reuse it.

**4. Not preloading on navigation intent**
Users who hover over a nav link before clicking give you 100-200ms of preload time for free. Call `import('./pages/Foo')` in `onMouseEnter` / `onFocus`. The browser caches the in-flight request, so clicking immediately after hover is instant.

**5. Nesting `Suspense` at the wrong level**
Placing `<Suspense>` at the very root means a single lazy child suspends the entire app. Place boundaries at the route level (or lower) so that loading one section does not blank the entire UI.

---

## Interview Angle

**Q: "How does code splitting work and what problem does it solve?"**

Strong answer: A bundler like Vite or Webpack splits the application into multiple chunks — one per route or feature. The main bundle stays small (faster initial load); chunks download on demand. In React, `React.lazy(() => import('./Page'))` tells the bundler to create a split point. `Suspense` provides the loading UI during the download. In Angular, `loadComponent` in the route config achieves the same, and `@defer` offers even finer-grained splitting inside components.

**Follow-up: "What happens when a chunk fails to load in production?"**

A `ChunkLoadError` is thrown and bubbles to the nearest `ErrorBoundary`. The most common cause is a new deploy — the old hashed chunk URL no longer exists. The correct recovery is a full page reload (not a React re-render), because the new page load will fetch the new chunk manifest. Detect `ChunkLoadError` specifically and call `window.location.reload()`.

**Follow-up: "How do you prevent the skeleton from flashing on fast connections?"**

React 18's `useTransition` / `startTransition` marks the navigation as non-urgent, deferring the Suspense fallback until a configurable threshold (default 0ms — use a 200ms wrapper). Angular's `@loading (minimum 300ms)` guarantees the loading template stays visible for at least 300ms, preventing a flash.
