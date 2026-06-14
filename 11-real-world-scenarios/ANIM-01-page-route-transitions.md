# Page / Route Transition Animations

## The Idea

**In plain English:** When a user clicks a link and navigates from one page to another, the old page and new page swap instantly by default — a jarring hard-cut. Route transition animations replace that hard-cut with a visual gesture: the old page fades or slides out while the new one fades or slides in. This communicates direction, maintains spatial context, and makes the app feel like a cohesive product rather than a collection of disconnected HTML documents.

**Real-world analogy:** Think of a physical photo album.

- A **hard-cut navigation** is like ripping one photo out and slamming down the next — disorienting and abrupt.
- A **cross-fade transition** is like a professional slide projector dissolving from one slide to the next — smooth, continuous.
- A **slide transition** is like turning a page in the album — you know whether you went forward or backward based on the direction of motion.
- **`prefers-reduced-motion`** is like asking a friend with motion sickness if they want to skip the animated slide show and just flip directly to the photo — same content, no animation.

The router manages *which* page is visible. The animation layer manages *how* the swap happens. These are separate concerns and must stay separate.

---

## Learning Objectives

- Understand the browser-native View Transitions API and when it applies
- Use `startViewTransition` to animate between routes without a JS animation library
- Implement Framer Motion `AnimatePresence` with React Router for component-level transitions
- Build Angular route animations using `@angular/animations` with route state data
- Guard all motion behind `prefers-reduced-motion` to respect accessibility settings
- Avoid the common pitfalls: flash of unstyled content, scroll position bugs, concurrent transitions

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Animate a single element entering the DOM | ✅ with `@keyframes` + `animation` | — |
| Animate an element leaving the DOM | ❌ | CSS cannot delay DOM removal |
| Know the previous route to determine slide direction | ❌ | CSS has no routing state |
| Coordinate enter/exit timing (old out, new in) | ❌ | Requires JS to manage simultaneous animations |
| Snapshot the old page for cross-fade | ❌ | Requires View Transitions API (browser-native) or JS |
| Skip animation for `prefers-reduced-motion` | ✅ with `@media` | — |

**Conclusion:** CSS owns the visual keyframes. JS/framework owns the trigger — deciding *when* to animate, *which* animation to play (based on route direction), and *when* to remove the old DOM node.

---

## HTML & CSS Foundation

### View Transitions API — Minimal CSS

```css
/* The browser creates ::view-transition-old and ::view-transition-new
   pseudo-elements automatically when startViewTransition() is called. */

/* Default cross-fade override — customize timing */
::view-transition-old(root) {
  animation: 200ms ease-out both fade-out;
}

::view-transition-new(root) {
  animation: 300ms ease-in both fade-in;
}

@keyframes fade-out {
  from { opacity: 1; }
  to   { opacity: 0; }
}

@keyframes fade-in {
  from { opacity: 0; }
  to   { opacity: 1; }
}

/* Slide variant — forward navigation */
::view-transition-old(root) {
  animation: 300ms ease-in-out both slide-out-left;
}
::view-transition-new(root) {
  animation: 300ms ease-in-out both slide-in-right;
}

@keyframes slide-out-left {
  to { transform: translateX(-30px); opacity: 0; }
}
@keyframes slide-in-right {
  from { transform: translateX(30px); opacity: 0; }
}

/* Honor reduced motion */
@media (prefers-reduced-motion: reduce) {
  ::view-transition-old(root),
  ::view-transition-new(root) {
    animation: none;
  }
}
```

### Base Page Wrapper

```html
<!-- Each page route renders into this wrapper -->
<main id="page-root" class="page">
  <!-- route content -->
</main>
```

```css
.page {
  min-height: 100dvh;
  /* Prevent layout shifts during transition */
  contain: layout style;
}
```

---

## React Implementation

### Using the View Transitions API with React Router v6

```tsx
// router.tsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';
import { startTransition } from 'react';

// Wrap React Router's navigate with startViewTransition
function withViewTransition(navigate: () => void) {
  if (!document.startViewTransition) {
    navigate();
    return;
  }
  document.startViewTransition(() => {
    startTransition(navigate);
  });
}

// Custom Link that triggers view transition
import { useNavigate, type To } from 'react-router-dom';

interface TransitionLinkProps extends React.AnchorHTMLAttributes<HTMLAnchorElement> {
  to: To;
  children: React.ReactNode;
}

export function TransitionLink({ to, children, ...rest }: TransitionLinkProps) {
  const navigate = useNavigate();

  const handleClick = (e: React.MouseEvent<HTMLAnchorElement>) => {
    e.preventDefault();
    withViewTransition(() => navigate(to));
  };

  return (
    <a href={String(to)} onClick={handleClick} {...rest}>
      {children}
    </a>
  );
}
```

### Framer Motion AnimatePresence with React Router v6

```tsx
// AnimatedRoutes.tsx
import { AnimatePresence, motion } from 'framer-motion';
import { useLocation, Routes, Route } from 'react-router-dom';
import { useReducedMotion } from 'framer-motion';

// Route-level variants
const pageVariants = {
  initial: { opacity: 0, y: 12 },
  animate: { opacity: 1, y: 0 },
  exit:    { opacity: 0, y: -12 },
};

const pageTransition = {
  type: 'tween',
  ease: 'easeInOut',
  duration: 0.25,
};

// Wrapper applied to every page component
export function PageWrapper({ children }: { children: React.ReactNode }) {
  const shouldReduceMotion = useReducedMotion();

  return (
    <motion.div
      variants={shouldReduceMotion ? {} : pageVariants}
      initial="initial"
      animate="animate"
      exit="exit"
      transition={pageTransition}
      style={{ width: '100%' }}
    >
      {children}
    </motion.div>
  );
}

// Top-level animated routes
export function AnimatedRoutes() {
  const location = useLocation();

  return (
    // mode="wait" ensures exit animation completes before enter starts
    <AnimatePresence mode="wait" initial={false}>
      <Routes location={location} key={location.pathname}>
        <Route
          path="/"
          element={
            <PageWrapper>
              <HomePage />
            </PageWrapper>
          }
        />
        <Route
          path="/about"
          element={
            <PageWrapper>
              <AboutPage />
            </PageWrapper>
          }
        />
        <Route
          path="/products/:id"
          element={
            <PageWrapper>
              <ProductPage />
            </PageWrapper>
          }
        />
      </Routes>
    </AnimatePresence>
  );
}
```

### Direction-Aware Slide Transitions

```tsx
// useNavigationDirection.ts
import { useRef, useEffect } from 'react';
import { useLocation } from 'react-router-dom';

// Simple history stack to determine forward vs back
const ROUTE_ORDER = ['/', '/about', '/products'];

export function useNavigationDirection() {
  const location = useLocation();
  const prevPathRef = useRef<string>(location.pathname);

  const prevIdx = ROUTE_ORDER.indexOf(prevPathRef.current);
  const currIdx = ROUTE_ORDER.indexOf(location.pathname);
  const direction = currIdx >= prevIdx ? 1 : -1;

  useEffect(() => {
    prevPathRef.current = location.pathname;
  }, [location.pathname]);

  return direction;
}

// SlidePageWrapper.tsx
export function SlidePageWrapper({ children }: { children: React.ReactNode }) {
  const direction = useNavigationDirection();
  const shouldReduceMotion = useReducedMotion();

  const variants = {
    initial: { x: direction * 60, opacity: 0 },
    animate: { x: 0, opacity: 1 },
    exit:    { x: direction * -60, opacity: 0 },
  };

  return (
    <motion.div
      variants={shouldReduceMotion ? {} : variants}
      initial="initial"
      animate="animate"
      exit="exit"
      transition={{ type: 'spring', stiffness: 300, damping: 30 }}
    >
      {children}
    </motion.div>
  );
}
```

---

## Angular Implementation

### Route Animation Setup

```typescript
// app.animations.ts
import {
  trigger, transition, style, animate, query, group
} from '@angular/animations';

export const routeAnimations = trigger('routeAnimations', [
  // Default cross-fade for any route change
  transition('* <=> *', [
    // Both old and new page are in the DOM simultaneously during the transition
    query(':enter, :leave', [
      style({ position: 'absolute', width: '100%', top: 0, left: 0 }),
    ], { optional: true }),

    group([
      query(':leave', [
        animate('200ms ease-out', style({ opacity: 0 })),
      ], { optional: true }),
      query(':enter', [
        style({ opacity: 0 }),
        animate('300ms 100ms ease-in', style({ opacity: 1 })),
      ], { optional: true }),
    ]),
  ]),
]);
```

### Route State for Direction-Aware Animations

```typescript
// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: '',
    component: HomeComponent,
    data: { animationState: 'home' },
  },
  {
    path: 'about',
    component: AboutComponent,
    data: { animationState: 'about' },
  },
  {
    path: 'products/:id',
    component: ProductComponent,
    data: { animationState: 'product' },
  },
];
```

### App Shell Component

```typescript
// app.component.ts
import {
  Component, inject, signal, HostBinding
} from '@angular/core';
import { RouterOutlet } from '@angular/router';
import { routeAnimations } from './app.animations';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet],
  animations: [routeAnimations],
  template: `
    <main
      [@routeAnimations]="getRouteAnimationState(outlet)"
      style="position: relative; overflow: hidden;"
    >
      <router-outlet #outlet="outlet" />
    </main>
  `,
})
export class AppComponent {
  // Reduce motion: skip animations entirely
  private prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

  getRouteAnimationState(outlet: RouterOutlet): string {
    if (this.prefersReducedMotion) return 'none';
    return outlet?.activatedRouteData?.['animationState'] ?? 'none';
  }
}
```

### Slide Animation with Route Data Direction

```typescript
// slide.animations.ts
import {
  trigger, transition, style, animate, query, group
} from '@angular/animations';

export const slideAnimations = trigger('routeAnimations', [
  // forward: home → about → product
  transition('home => about, about => product, home => product', [
    query(':enter, :leave', style({ position: 'absolute', width: '100%' }), { optional: true }),
    group([
      query(':leave', animate('300ms ease-in-out', style({ transform: 'translateX(-100%)' })), { optional: true }),
      query(':enter', [
        style({ transform: 'translateX(100%)' }),
        animate('300ms ease-in-out', style({ transform: 'translateX(0)' })),
      ], { optional: true }),
    ]),
  ]),
  // backward
  transition('about => home, product => about, product => home', [
    query(':enter, :leave', style({ position: 'absolute', width: '100%' }), { optional: true }),
    group([
      query(':leave', animate('300ms ease-in-out', style({ transform: 'translateX(100%)' })), { optional: true }),
      query(':enter', [
        style({ transform: 'translateX(-100%)' }),
        animate('300ms ease-in-out', style({ transform: 'translateX(0)' })),
      ], { optional: true }),
    ]),
  ]),
]);
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| `prefers-reduced-motion` respected | CSS `@media` disables View Transition keyframes; `useReducedMotion()` hook skips Framer variants; Angular guard checks `matchMedia` |
| No animation longer than 400ms | Keep transitions under 300ms; exits under 200ms |
| Focus moves to new page content on navigation | Call `document.getElementById('main-content')?.focus()` after transition ends |
| Scroll position resets to top on navigation | React Router `ScrollRestoration`; Angular `scrollPositionRestoration: 'top'` in router config |
| Screen reader announces page change | `<title>` update on route change; `aria-live` region or focus move triggers re-read |
| No content flicker during transition | Set `contain: layout style` on page wrapper; avoid layout-triggering properties in animations |
| Concurrent navigation handled | Framer `mode="wait"` prevents overlap; View Transitions API cancels ongoing transitions automatically |

---

## Production Pitfalls

**1. `AnimatePresence` key must be the pathname, not the full location object**
Using `location` as the key causes re-mounts on search param changes (e.g., `?page=2`). Fix: use `location.pathname` as the `key` prop on `<Routes>`.

**2. View Transitions API and React 18 concurrent mode conflict**
Calling `startViewTransition` without wrapping the state update in `startTransition` causes React to flush synchronously, defeating concurrent rendering benefits. Fix: always use `startTransition(() => navigate(to))` inside `startViewTransition`.

**3. Scroll position jumps mid-animation**
Both the old and new page are in the DOM simultaneously during Framer `mode="wait"`. If the new page is taller, the page height changes mid-animation. Fix: set `position: absolute` on exiting pages and `overflow: hidden` on the containing wrapper.

**4. Angular route animation `query(':enter')` returns nothing**
This happens when `ChangeDetectionStrategy.OnPush` delays the view update. Fix: use `RouterOutlet` directly in the template without an extra wrapper `*ngIf`, and ensure the animation runs after the outlet activates.

**5. Memory leak from abandoned transitions**
If the user clicks a new link before the previous transition finishes, orphaned animation callbacks can fire on unmounted components. Fix: cancel Framer's exit animation by using `AnimatePresence mode="wait"` or `mode="popLayout"`, and abort View Transitions with the returned promise's rejection.

**6. iOS Safari does not support View Transitions API (as of 2024)**
Always feature-detect: `if (document.startViewTransition)`. Provide a graceful no-op fallback that simply executes the navigation without animation.

---

## Interview Angle

**Q: "How would you implement animated page transitions in a React SPA without degrading performance or accessibility?"**

Strong answer covers four points:
1. **Two strategies, pick by project need** — View Transitions API is browser-native, zero-KB, but limited browser support; Framer Motion `AnimatePresence` is cross-browser but adds bundle weight. Know the trade-offs.
2. **`mode="wait"` is critical in Framer** — without it, entering and exiting components overlap, causing layout thrash and visual glitches. Explain what `mode="popLayout"` adds (FLIP-based exit so layout updates instantly).
3. **`prefers-reduced-motion` is not optional** — vestibular disorders make full-page motion animations genuinely harmful. Disable or replace with a simple opacity transition when the media query matches.
4. **Scroll and focus management** — transitions break the browser's default scroll restoration and focus move to new content. Both must be handled explicitly.

**Follow-up: "The View Transitions API — what does `startViewTransition` actually do under the hood?"**
It pauses rendering, takes a screenshot of the current state as `::view-transition-old(root)`, makes the DOM update, renders the new state into `::view-transition-new(root)`, then runs both pseudo-elements through CSS animations simultaneously before removing them.
