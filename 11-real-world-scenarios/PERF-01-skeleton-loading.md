# Skeleton Loading Screens

## The Idea

**In plain English:** Instead of showing a blank page or a spinning indicator while content loads, you show the exact same layout the real content will occupy — filled with animated grey placeholders. The page feels instantly populated, the user understands what is coming, and there is no jarring layout shift when the real data arrives.

**Real-world analogy:** Think of a printed newspaper layout dummy.

- Before articles are written, editors lay out a page with grey lorem-ipsum blocks that have the exact same column widths, image proportions, and headline sizes as the real content.
- A typesetter looking at that dummy knows exactly where everything will go — the page structure is already understood.
- When real copy arrives, it drops into the prepared slots with no rearrangement.
- The **shimmer wave** moving across the blocks = the printing press running across the page, signalling that work is in progress.
- The **content-matched shapes** (wide rectangle for a headline, circular thumbnail for an avatar, narrow lines for body text) = the dummy columns matching the real article proportions.
- **Layout shift on reveal** = accidentally leaving a lorem block in the final print — a mistake that breaks the reader's trust.

The key insight: a skeleton is not decoration. It is a structural promise to the browser and to the user that the final content will fit exactly here, with no reflow.

---

## Learning Objectives

- Build a shimmer animation using `@keyframes`, `background: linear-gradient`, and `background-size`
- Understand why content-matched shapes are essential to prevent Cumulative Layout Shift (CLS)
- Manage skeleton timing and duration through CSS custom properties
- Wire skeletons into React Suspense boundaries correctly
- Implement the same pattern in Angular using signals and `@defer`
- Avoid the production pitfalls: over-animation, mis-matched shapes, accessibility noise

---

## Why CSS Alone Isn't Enough

The shimmer animation and shape styling are pure CSS. Deciding *when* to show the skeleton vs. the real content requires framework state.

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Shimmer animation (the moving highlight) | ✅ `@keyframes` + gradient | — |
| Skeleton shape layout (rectangles, circles) | ✅ `width`, `height`, `border-radius` | — |
| Switch from skeleton to real content | ❌ | Requires knowledge of async loading state |
| Fetch data and track loading/error/success | ❌ | CSS has no async model |
| Prevent flash of skeleton on fast connections | ❌ | Needs a minimum display timer in JS |
| Match skeleton to real content dimensions | ⚠️ | CSS sets the shape, JS must ensure the real content matches |
| Announce loading state to screen readers | ❌ | `aria-busy` must be toggled by JS |
| Integrate with React Suspense or Angular `@defer` | ❌ | Framework-level boundary, not CSS |

**Conclusion:** CSS owns the shimmer, shape, and timing tokens. The framework owns the async state machine that decides which tree to render.

---

## HTML & CSS Foundation

### The Shimmer Animation

```css
/* ─── Skeleton tokens ─── */
:root {
  --sk-base-color:      #e2e5e7;
  --sk-highlight-color: #f0f2f4;
  --sk-duration:        1.4s;
  --sk-delay:           0s;
  --sk-border-radius:   4px;
  --sk-animation-timing: ease-in-out;
}

/* ─── Core shimmer ─── */
/*
  Technique: an oversized gradient (200% wide) is animated
  from left to right. background-size: 200% 100% means the
  gradient is twice as wide as the element. Shifting
  background-position from 100% to -100% moves the highlight
  band across.
*/
@keyframes skeleton-shimmer {
  0%   { background-position: 100% 50%; }
  100% { background-position: -100% 50%; }
}

.skeleton {
  background-color: var(--sk-base-color);
  background-image: linear-gradient(
    90deg,
    var(--sk-base-color)     0%,
    var(--sk-highlight-color) 50%,
    var(--sk-base-color)     100%
  );
  background-size: 200% 100%;
  animation: skeleton-shimmer var(--sk-duration) var(--sk-animation-timing)
             var(--sk-delay) infinite;
  border-radius: var(--sk-border-radius);
  /* Prevent the skeleton from being read by screen readers */
  /* aria-hidden is added in HTML/JSX */
}

/* ─── Reduced motion: replace moving shimmer with a static pulse ─── */
@media (prefers-reduced-motion: reduce) {
  .skeleton {
    animation: skeleton-pulse 2s ease-in-out infinite;
    background-image: none;
  }
}

@keyframes skeleton-pulse {
  0%, 100% { opacity: 1; }
  50%       { opacity: 0.5; }
}
```

### Content-Matched Skeleton Shapes

```html
<!--
  This HTML mirrors a real "user profile card" layout.
  Every skeleton element has the same dimensions as its
  real counterpart — this is what prevents layout shift.
-->
<div class="profile-card" aria-busy="true" aria-label="Loading profile">

  <!-- Avatar: 48px circle — matches <img class="avatar"> which is also 48px circle -->
  <div class="skeleton skeleton-avatar" aria-hidden="true"></div>

  <div class="profile-info">
    <!-- Name line: 60% width — matches typical name length -->
    <div class="skeleton skeleton-text skeleton-text--title" aria-hidden="true"></div>

    <!-- Sub-line: 40% width — matches the username/handle -->
    <div class="skeleton skeleton-text skeleton-text--subtitle" aria-hidden="true"></div>
  </div>

  <!-- Action button placeholder -->
  <div class="skeleton skeleton-button" aria-hidden="true"></div>

</div>
```

```css
/* ─── Shape helpers ─── */
.skeleton-avatar {
  width: 48px;
  height: 48px;
  border-radius: 50%;   /* circle */
  flex-shrink: 0;
}

.skeleton-text {
  height: 14px;
  border-radius: 7px;   /* pill — looks like a text line */
  margin-bottom: 8px;
}

.skeleton-text--title {
  width: 60%;
  height: 18px;
}

.skeleton-text--subtitle {
  width: 40%;
}

.skeleton-button {
  width: 88px;
  height: 36px;
  border-radius: 18px;  /* pill button shape */
  margin-left: auto;
}

/* ─── Card layout (same for skeleton and real content) ─── */
.profile-card {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 16px;
  border: 1px solid #eee;
  border-radius: 8px;
}

.profile-info {
  flex: 1;
  min-width: 0;   /* allow flex child to shrink below content size */
}
```

**Why 200% background-size?** A gradient that spans exactly 100% of the element has nowhere to travel — it is static. At 200%, the highlight band starts off-screen to the right (`background-position: 100%`), sweeps through the element center, and exits off-screen to the left (`background-position: -100%`), producing the moving highlight effect.

**Why content-matched shapes matter for CLS:** If your skeleton text line is 14px tall but your real `<p>` with line-height renders at 22px, the layout shifts 8px downward on every card when content loads. Multiply by 20 cards and you get a catastrophic CLS score. Match height *and* width as closely as possible.

---

## React Implementation

### Base Skeleton Component

```tsx
// Skeleton.tsx
import type { CSSProperties, ReactNode } from 'react';

interface SkeletonProps {
  width?: string | number;
  height?: string | number;
  borderRadius?: string | number;
  /** Stack multiple lines with even spacing */
  lines?: number;
  /** Override shimmer CSS custom properties per-instance */
  duration?: string;
  delay?: string;
  className?: string;
  style?: CSSProperties;
}

export function Skeleton({
  width = '100%',
  height = 16,
  borderRadius,
  lines = 1,
  duration,
  delay,
  className = '',
  style,
}: SkeletonProps) {
  const inlineVars: CSSProperties = {
    ...(duration && { '--sk-duration': duration } as CSSProperties),
    ...(delay    && { '--sk-delay':    delay    } as CSSProperties),
  };

  if (lines > 1) {
    return (
      <div style={{ display: 'flex', flexDirection: 'column', gap: 8 }}>
        {Array.from({ length: lines }, (_, i) => (
          <div
            key={i}
            className={`skeleton ${className}`}
            aria-hidden="true"
            style={{
              width: i === lines - 1 ? '70%' : width, // last line is shorter
              height,
              borderRadius: borderRadius ?? undefined,
              ...inlineVars,
              ...style,
            }}
          />
        ))}
      </div>
    );
  }

  return (
    <div
      className={`skeleton ${className}`}
      aria-hidden="true"
      style={{ width, height, borderRadius: borderRadius ?? undefined, ...inlineVars, ...style }}
    />
  );
}
```

### Content-Matched Skeleton Components

```tsx
// ProfileCardSkeleton.tsx
import { Skeleton } from './Skeleton';

export function ProfileCardSkeleton() {
  return (
    <div
      className="profile-card"
      role="status"
      aria-label="Loading profile"
      aria-busy="true"
    >
      {/* Avatar */}
      <Skeleton width={48} height={48} borderRadius="50%" />

      {/* Text lines */}
      <div className="profile-info">
        <Skeleton width="60%" height={18} style={{ marginBottom: 8 }} />
        <Skeleton width="40%" height={14} />
      </div>

      {/* Action button */}
      <Skeleton width={88} height={36} borderRadius={18} style={{ marginLeft: 'auto' }} />
    </div>
  );
}

// ArticleCardSkeleton.tsx — mirror the real ArticleCard layout exactly
export function ArticleCardSkeleton() {
  return (
    <article
      className="article-card"
      role="status"
      aria-label="Loading article"
      aria-busy="true"
    >
      {/* Hero image area */}
      <Skeleton width="100%" height={200} borderRadius="8px 8px 0 0" />

      <div style={{ padding: 16 }}>
        {/* Category tag */}
        <Skeleton width={80} height={22} borderRadius={11} style={{ marginBottom: 12 }} />

        {/* Headline — 2 lines */}
        <Skeleton lines={2} height={20} style={{ marginBottom: 12 }} />

        {/* Body preview — 3 lines */}
        <Skeleton lines={3} height={14} style={{ marginBottom: 16 }} />

        {/* Author row */}
        <div style={{ display: 'flex', alignItems: 'center', gap: 8 }}>
          <Skeleton width={32} height={32} borderRadius="50%" />
          <Skeleton width={120} height={14} />
          <Skeleton width={60} height={14} style={{ marginLeft: 'auto' }} />
        </div>
      </div>
    </article>
  );
}
```

### React Suspense Integration

```tsx
// ArticleFeed.tsx
import { Suspense } from 'react';
import { ArticleCardSkeleton } from './ArticleCardSkeleton';
import { ArticleList } from './ArticleList';  // uses use() or a data library

// Suspense fallback: render as many skeletons as the real list will show.
// If you know the page size is 6, show 6 skeletons — not a spinner.
function ArticleFeedSkeleton() {
  return (
    <div className="article-grid" aria-label="Loading articles" aria-busy="true">
      {Array.from({ length: 6 }, (_, i) => (
        <ArticleCardSkeleton key={i} />
      ))}
    </div>
  );
}

export function ArticleFeed() {
  return (
    <Suspense fallback={<ArticleFeedSkeleton />}>
      <ArticleList />
    </Suspense>
  );
}
```

### Custom Hook: Minimum Skeleton Display Time

On fast connections the skeleton may flash for 50ms — visually worse than showing nothing at all. A minimum display duration smooths this out.

```tsx
// useMinLoadingTime.ts
import { useState, useEffect, useRef } from 'react';

/**
 * Returns true while loading OR within the minimum display window.
 * Prevents a sub-50ms flash of the skeleton on fast connections.
 */
export function useMinLoadingTime(
  isLoading: boolean,
  minMs: number = 400
): boolean {
  const [show, setShow]     = useState(isLoading);
  const timerRef            = useRef<ReturnType<typeof setTimeout> | null>(null);
  const startedAt           = useRef<number | null>(null);

  useEffect(() => {
    if (isLoading) {
      startedAt.current = Date.now();
      setShow(true);
    } else {
      const elapsed = startedAt.current ? Date.now() - startedAt.current : minMs;
      const remaining = Math.max(0, minMs - elapsed);

      if (remaining === 0) {
        setShow(false);
      } else {
        timerRef.current = setTimeout(() => setShow(false), remaining);
      }
    }

    return () => {
      if (timerRef.current) clearTimeout(timerRef.current);
    };
  }, [isLoading, minMs]);

  return show;
}

// Usage in a component:
function UserProfile({ userId }: { userId: string }) {
  const { data, isLoading } = useUser(userId);
  const showSkeleton = useMinLoadingTime(isLoading, 400);

  if (showSkeleton) return <ProfileCardSkeleton />;

  return <ProfileCard user={data!} />;
}
```

---

## Angular Implementation

### Skeleton Component

```typescript
// skeleton.component.ts
import { Component, Input, computed, signal } from '@angular/core';
import { NgStyle, NgFor, NgIf } from '@angular/common';

@Component({
  selector: 'app-skeleton',
  standalone: true,
  imports: [NgStyle, NgFor, NgIf],
  template: `
    <ng-container *ngIf="lines() > 1; else single">
      <div style="display:flex; flex-direction:column; gap:8px">
        <div
          *ngFor="let i of lineIndexes()"
          class="skeleton"
          aria-hidden="true"
          [ngStyle]="lineStyle(i)"
        ></div>
      </div>
    </ng-container>
    <ng-template #single>
      <div
        class="skeleton"
        aria-hidden="true"
        [ngStyle]="singleStyle()"
      ></div>
    </ng-template>
  `,
})
export class SkeletonComponent {
  @Input() width:        string | number = '100%';
  @Input() height:       string | number = 16;
  @Input() borderRadius: string | number = 4;
  @Input() lines:        number          = 1;
  @Input() duration:     string          = '';
  @Input() delay:        string          = '';

  private _lines = computed(() => this.lines);

  lineIndexes() {
    return Array.from({ length: this.lines }, (_, i) => i);
  }

  lineStyle(i: number): Record<string, string> {
    return {
      width:        i === this.lines - 1 ? '70%' : this.px(this.width),
      height:       this.px(this.height),
      borderRadius: this.px(this.borderRadius),
      ...(this.duration && { '--sk-duration': this.duration }),
      ...(this.delay    && { '--sk-delay':    this.delay    }),
    };
  }

  singleStyle(): Record<string, string> {
    return {
      width:        this.px(this.width),
      height:       this.px(this.height),
      borderRadius: this.px(this.borderRadius),
      ...(this.duration && { '--sk-duration': this.duration }),
      ...(this.delay    && { '--sk-delay':    this.delay    }),
    };
  }

  private px(v: string | number): string {
    return typeof v === 'number' ? `${v}px` : v;
  }
}
```

### Angular `@defer` Integration

Angular 17+ ships `@defer` blocks — the framework-native equivalent of React Suspense. Combine it with skeleton components for zero-boilerplate loading states.

```typescript
// article-feed.component.ts
import { Component } from '@angular/core';
import { ArticleListComponent } from './article-list.component';
import { ArticleCardSkeletonComponent } from './article-card-skeleton.component';

@Component({
  selector: 'app-article-feed',
  standalone: true,
  imports: [ArticleListComponent, ArticleCardSkeletonComponent],
  template: `
    @defer (on viewport; prefetch on idle) {
      <app-article-list />
    } @loading (minimum 400ms) {
      <!--
        minimum 400ms: Angular will not dismiss the loading
        block before 400ms have elapsed, preventing flash.
      -->
      <div class="article-grid" aria-label="Loading articles" aria-busy="true">
        @for (i of skeletonCount; track i) {
          <app-article-card-skeleton />
        }
      </div>
    } @error {
      <p role="alert">Failed to load articles. Please try again.</p>
    }
  `,
})
export class ArticleFeedComponent {
  readonly skeletonCount = Array.from({ length: 6 }, (_, i) => i);
}
```

### Signal-Based Loading State (Without `@defer`)

```typescript
// profile.component.ts
import { Component, OnInit, inject, signal, computed } from '@angular/core';
import { UserService } from './user.service';
import { ProfileCardComponent } from './profile-card.component';
import { ProfileCardSkeletonComponent } from './profile-card-skeleton.component';
import { NgIf } from '@angular/common';

@Component({
  selector: 'app-profile',
  standalone: true,
  imports: [NgIf, ProfileCardComponent, ProfileCardSkeletonComponent],
  template: `
    <app-profile-card-skeleton
      *ngIf="showSkeleton()"
      role="status"
      aria-label="Loading profile"
      aria-busy="true"
    />
    <app-profile-card
      *ngIf="!showSkeleton() && user()"
      [user]="user()!"
    />
    <p *ngIf="error()" role="alert" class="error-message">
      {{ error() }}
    </p>
  `,
})
export class ProfileComponent implements OnInit {
  private userService = inject(UserService);

  readonly user         = signal<User | null>(null);
  readonly error        = signal<string | null>(null);
  private isLoading     = signal(true);
  private minTimeElapsed = signal(false);

  // Only hide skeleton once BOTH loading is done AND min time has passed
  readonly showSkeleton = computed(
    () => this.isLoading() || !this.minTimeElapsed()
  );

  ngOnInit() {
    // Start minimum display timer
    setTimeout(() => this.minTimeElapsed.set(true), 400);

    this.userService.getCurrentUser().subscribe({
      next:  (u) => { this.user.set(u);              this.isLoading.set(false); },
      error: (e) => { this.error.set(e.message);     this.isLoading.set(false); },
    });
  }
}
```

### Content-Matched Angular Skeleton

```typescript
// article-card-skeleton.component.ts
import { Component } from '@angular/core';
import { SkeletonComponent } from './skeleton.component';

@Component({
  selector: 'app-article-card-skeleton',
  standalone: true,
  imports: [SkeletonComponent],
  template: `
    <article class="article-card" aria-hidden="true">
      <!-- Hero image -->
      <app-skeleton width="100%" [height]="200" borderRadius="8px 8px 0 0" />

      <div style="padding:16px">
        <!-- Tag -->
        <app-skeleton [width]="80" [height]="22" borderRadius="11px"
          style="margin-bottom:12px" />

        <!-- Headline -->
        <app-skeleton [lines]="2" [height]="20" style="margin-bottom:12px" />

        <!-- Body preview -->
        <app-skeleton [lines]="3" [height]="14" style="margin-bottom:16px" />

        <!-- Author row -->
        <div style="display:flex; align-items:center; gap:8px">
          <app-skeleton [width]="32" [height]="32" borderRadius="50%" />
          <app-skeleton [width]="120" [height]="14" />
          <app-skeleton [width]="60" [height]="14" style="margin-left:auto" />
        </div>
      </div>
    </article>
  `,
})
export class ArticleCardSkeletonComponent {}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Screen readers skip the shimmer elements | `aria-hidden="true"` on every `.skeleton` div |
| Loading state announced to screen readers | Container has `role="status"` and `aria-busy="true"` |
| Success state announced when content loads | Replace `aria-busy="true"` with `aria-busy="false"` or remove it; `role="status"` announces the change |
| Error state announced | Error container uses `role="alert"` (assertive live region) |
| Skeleton does not interfere with keyboard focus | No focusable elements inside a skeleton; all are `div`/`span` |
| Reduced motion users are not nauseated by shimmer | `@media (prefers-reduced-motion: reduce)` replaces shimmer with a static opacity pulse |
| CLS is prevented so viewport does not jump | Skeleton shapes match real content dimensions exactly (height, width, border-radius) |
| Meaningful page title during loading | `<title>` is not "Loading…" — use "Users — MyApp" and let `aria-busy` signal loading state within the page |

---

## Production Pitfalls

**1. Skeleton shapes that do not match real content cause CLS**
If a skeleton paragraph block is 80px tall but the real `<p>` renders at 96px due to line-height differences, every card shifts 16px when data lands. Audit shapes by toggling loading state in DevTools and measuring the shift with the CLS metric in Lighthouse. Fix: define shared sizing tokens (e.g., `--card-hero-height: 200px`) used by both the skeleton and the real component.

**2. Animating too many elements simultaneously tanks CPU**
A page with 30 skeletons, each running its own `background-position` animation, creates 30 independent compositing layers. On mid-range Android devices this causes visible frame drops. Fix: use a single `@keyframes` animation on all elements via the same class (the browser can share the timeline) and ensure `.skeleton` triggers GPU compositing with `will-change: background-position` only if needed (measure first — `will-change` has memory cost).

**3. Sub-200ms skeleton flash is worse than no skeleton**
On a fast connection, the skeleton appears for one frame, then real content slams in. The flash is more disorienting than a blank-then-content transition. Fix: use a minimum display timer (400ms is a good baseline). React: the `useMinLoadingTime` hook above. Angular: `@defer` has a built-in `minimum` duration parameter.

**4. Using a generic spinner inside a skeleton slot destroys CLS guarantees**
If one section of your skeleton is a spinner instead of a shape-matched placeholder, that section will reflow when the spinner is replaced by real content. The whole point of a skeleton is stable geometry. Fix: every loading placeholder must occupy the same bounding box as its real content counterpart.

**5. React Suspense boundaries placed too high cause excessive skeleton area**
Wrapping an entire page in one Suspense boundary means the whole page skeletonises when any data dependency suspends. Fix: co-locate Suspense boundaries with the specific component that depends on that data. A user profile sidebar should have its own boundary, independent of the main content area.

**6. Stale skeleton left visible after error**
If an error occurs, `isLoading` becomes false but some codepaths forget to also hide the skeleton. The user sees an error message overlaid on or beneath a still-shimmering skeleton. Fix: a three-state model (`'loading' | 'success' | 'error'`) is safer than a boolean — each state renders exactly one UI branch.

**7. CSS custom property inheritance breaks scoped durations**
If you set `--sk-duration` on a parent container to stagger skeletons, and a child component also sets it, the child value wins and the stagger is lost. Fix: apply delay stagger through a class or `style` attribute on the individual skeleton element, not on an ancestor container that could be overridden downstream.

---

## Interview Angle

**Q: "How do you implement a skeleton loading screen and what makes it better than a spinner?"**

A skeleton preserves layout geometry during loading. A spinner communicates "wait" but gives no information about what is arriving — the layout shifts when real content loads. A skeleton is a structural contract: every skeleton element occupies the same bounding box as the real content that will replace it. This eliminates Cumulative Layout Shift (CLS), which directly affects Core Web Vitals and user perception of performance. The shimmer is achieved with a single `@keyframes` animation that shifts a `background-position` across an oversized gradient — no JavaScript needed for the visual effect. The key CSS levers are `background-size: 200% 100%` (the gradient must be wider than the element to create travel distance) and `background-position` animated from `100%` to `-100%`.

**Follow-up: "How do you integrate skeletons with React Suspense without causing a flash of loading state on fast connections?"**

React Suspense shows the fallback immediately when a component suspends, which on fast connections produces a sub-100ms flash that is more jarring than seeing nothing. The fix is a minimum display timer: track when loading started, and when `isLoading` becomes false, only dismiss the skeleton once both the data is ready and at least 400ms have elapsed. In code this is a custom hook that keeps a separate `show` boolean, sets it true when loading starts, and sets it false only after `Math.max(0, minMs - elapsed)` milliseconds. Angular's `@defer` block has a first-class `minimum` duration param that does this declaratively.