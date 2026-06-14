# Reduced Motion Animations

## The Idea

**In plain English:** Some users experience nausea, dizziness, or seizures when they see large animations, parallax scrolling, or elements that fly across the screen. The operating system exposes a preference — "reduce motion" — that the user sets in their accessibility settings. Your job as a frontend engineer is to detect that preference and replace intense motion with subtle or instant alternatives, without stripping all visual feedback entirely.

**Real-world analogy:** Think of a cinema with two projection modes.

- In **standard mode**, the film uses sweeping camera pans, fast cuts, and kinetic action sequences. Most people enjoy it.
- In **accessibility mode**, the same story is told with slower transitions, static establishing shots, and gentle fades. The *narrative* is identical — only the *delivery* changes.
- The **usher at the door** = the OS accessibility setting. They hand you the right ticket before you sit down.
- **Your CSS, hooks, and services** = the projection booth. They read the ticket and switch reels automatically.
- A **bad projection booth** ignores the ticket entirely and shows the full kinetic version to someone who gets motion sick — that is the cost of skipping `prefers-reduced-motion`.

The key insight: "reduced motion" does not mean "no animation." It means replacing vestibular-triggering motion (translate, scale, rotate, parallax) with motion that does not affect the user's sense of spatial orientation (opacity, color, instant state changes).

---

## Learning Objectives

- Understand the vestibular system impact and why `prefers-reduced-motion` is a medical-grade requirement
- Write the CSS media query correctly and know the two values: `reduce` vs `no-preference`
- Understand the instant vs. animated fallback strategy and when each applies
- Build a `useReducedMotion` React hook that reads and reacts to live OS preference changes
- Build an Angular `MotionService` using signals that drives component animation decisions
- Integrate GSAP's `gsap.matchMedia()` with `prefers-reduced-motion` for script-driven animations
- Avoid the common pitfalls: disabling opacity transitions, blocking JS animations, inconsistent fallback gaps

---

## Why CSS Alone Isn't Enough

CSS can handle the simple case — disabling transitions on static elements. But a production app has script-driven animations, JavaScript timers, third-party libraries, and stateful motion that CSS cannot reach.

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Disable CSS transitions for reduced-motion users | ✅ with `@media (prefers-reduced-motion: reduce)` | — |
| Disable CSS keyframe animations | ✅ same media query | — |
| Stop a GSAP timeline mid-play | ❌ | GSAP runs in JS; CSS has no access to JS timers |
| Pass "should animate?" flag to a child component | ❌ | CSS cannot pass props or signals to framework components |
| React live to OS setting change without page reload | ❌ | `window.matchMedia` listener required in JS |
| Skip a `requestAnimationFrame` scroll parallax loop | ❌ | JS rAF loop is invisible to CSS |
| Disable a Lottie or Three.js animation | ❌ | Third-party renderers ignore CSS entirely |
| Unit-test that the preference is respected | ❌ | CSS media queries cannot be mocked in jest/vitest |

**Conclusion:** CSS owns the static layer (transitions, keyframes, CSS animations). JS owns everything else — GSAP timelines, scroll listeners, third-party animation libraries, and the reactive plumbing that gives components a boolean to branch on.

---

## HTML & CSS Foundation

### The Fallback Strategy

There are two valid strategies. Choose based on the animation's purpose:

- **Instant swap** — for spatial/positional motion (slide-in, translate, scale, parallax). Remove the motion entirely. The element just appears at its final state.
- **Opacity-only fallback** — for feedback animations (loading spinner, badge pulse, success flash). Replace movement with a gentle fade. The user still gets visual feedback, just without spatial disorientation.

### CSS Structure

```css
/* ─── Tokens ─── */
:root {
  --transition-move: 0.4s cubic-bezier(0.4, 0, 0.2, 1);
  --transition-fade: 0.25s ease;
  --transition-bounce: 0.6s cubic-bezier(0.34, 1.56, 0.64, 1);
}

/* ─── Base animations ─── */

/* Slide-in panel — spatial motion, instant fallback */
.panel {
  transform: translateX(-100%);
  transition: transform var(--transition-move);
}

.panel.is-open {
  transform: translateX(0);
}

/* Loading pulse — replace movement with opacity pulse */
@keyframes pulse-scale {
  0%, 100% { transform: scale(1);    opacity: 1;   }
  50%       { transform: scale(1.08); opacity: 0.7; }
}

@keyframes pulse-fade {
  0%, 100% { opacity: 1;   }
  50%       { opacity: 0.4; }
}

.loading-badge {
  animation: pulse-scale 1.4s ease-in-out infinite;
}

/* Entrance animation — elements fly up into position */
@keyframes slide-up {
  from { transform: translateY(24px); opacity: 0; }
  to   { transform: translateY(0);    opacity: 1; }
}

@keyframes fade-in {
  from { opacity: 0; }
  to   { opacity: 1; }
}

.card-enter {
  animation: slide-up 0.5s ease forwards;
}

/* ─── Reduced motion overrides ─── */
@media (prefers-reduced-motion: reduce) {
  /*
    Rule 1: Kill all CSS transitions that involve spatial properties.
    Visibility/opacity transitions are safe to keep.
  */
  *,
  *::before,
  *::after {
    animation-duration:        0.01ms !important;
    animation-iteration-count: 1      !important;
    transition-duration:       0.01ms !important;
    scroll-behavior:           auto   !important;
  }

  /*
    Rule 2: Re-enable opacity transitions that give non-spatial feedback.
    Override the nuclear rule above for specific safe transitions.
  */
  .loading-badge {
    animation: pulse-fade 1.8s ease-in-out infinite;
    /* Duration kept; only scale removed */
  }

  /*
    Rule 3: Slide-in panels must skip to final state immediately.
    The 0.01ms duration makes this effectively instant.
  */
  .panel {
    /* transition-duration already killed by Rule 1 */
    /* No extra rule needed — the panel snaps open */
  }

  /*
    Rule 4: Card entrances fade in instead of sliding up.
    Apply the safe keyframe variant.
  */
  .card-enter {
    animation-name: fade-in;
    /* duration is 0.01ms from Rule 1 — effectively instant appearance */
    /* If a very short fade is acceptable, override duration here: */
    animation-duration: 0.15s !important;
  }
}
```

**Important nuance about the nuclear `*` rule:** Setting `animation-duration: 0.01ms` on everything is widely recommended but has a side effect — it kills every animation, including safe opacity-only ones. The pattern above intentionally re-enables specific safe animations after the nuclear rule. Order matters: the more-specific selectors after the `*` override it.

### The `prefers-reduced-motion: no-preference` Pattern

Some engineers write the media query inverted — define the animation *only* inside `no-preference`. This is equally valid and avoids the override problem:

```css
/* Inverted approach: animate only when safe to do so */
.panel {
  transform: translateX(-100%);
  /* no transition by default */
}

.panel.is-open {
  transform: translateX(0);
}

@media (prefers-reduced-motion: no-preference) {
  .panel {
    transition: transform var(--transition-move);
  }
}
```

This approach is cleaner for new code. The nuclear override approach is better when retrofitting an existing large codebase.

---

## React Implementation

### The Hook — `useReducedMotion`

```tsx
// useReducedMotion.ts
import { useState, useEffect } from 'react';

/**
 * Returns true when the user has requested reduced motion at the OS level.
 * Reactively updates if the user changes the preference while the app is open.
 */
export function useReducedMotion(): boolean {
  const query = '(prefers-reduced-motion: reduce)';

  const [prefersReduced, setPrefersReduced] = useState<boolean>(() => {
    // SSR guard: window is not available during server rendering
    if (typeof window === 'undefined') return false;
    return window.matchMedia(query).matches;
  });

  useEffect(() => {
    if (typeof window === 'undefined') return;

    const mediaQuery = window.matchMedia(query);
    const handler = (event: MediaQueryListEvent) => {
      setPrefersReduced(event.matches);
    };

    // addEventListener is preferred over deprecated addListener
    mediaQuery.addEventListener('change', handler);
    return () => mediaQuery.removeEventListener('change', handler);
  }, []);

  return prefersReduced;
}
```

### Animated Panel — Instant vs. Animated

```tsx
// AnimatedPanel.tsx
import { useRef, useEffect } from 'react';
import { useReducedMotion } from './useReducedMotion';

interface AnimatedPanelProps {
  isOpen: boolean;
  children: React.ReactNode;
}

export function AnimatedPanel({ isOpen, children }: AnimatedPanelProps) {
  const reduced = useReducedMotion();
  const panelRef = useRef<HTMLDivElement>(null);

  // When reduced motion is on, skip the transition class entirely.
  // The panel appears/disappears via display or visibility change only.
  useEffect(() => {
    const panel = panelRef.current;
    if (!panel) return;

    if (reduced) {
      // No transition: set final state immediately
      panel.style.transform = isOpen ? 'translateX(0)' : 'translateX(-100%)';
      panel.style.transition = 'none';
    } else {
      // Full animation: let CSS handle it via class toggle
      panel.style.transform = '';
      panel.style.transition = '';
    }
  }, [isOpen, reduced]);

  return (
    <div
      ref={panelRef}
      className={`panel ${isOpen ? 'is-open' : ''}`}
      aria-hidden={!isOpen}
      // Inline style overrides handled by useEffect above
    >
      {children}
    </div>
  );
}
```

### Card List with Staggered Entrance

```tsx
// CardList.tsx
import { useReducedMotion } from './useReducedMotion';

interface Card {
  id: string;
  title: string;
  body: string;
}

interface CardListProps {
  cards: Card[];
}

export function CardList({ cards }: CardListProps) {
  const reduced = useReducedMotion();

  return (
    <ul style={{ listStyle: 'none', padding: 0 }}>
      {cards.map((card, index) => (
        <li
          key={card.id}
          className={reduced ? 'card-enter-reduced' : 'card-enter'}
          style={
            // Stagger only when motion is allowed; instant appearance when reduced
            reduced ? undefined : { animationDelay: `${index * 80}ms` }
          }
        >
          <article>
            <h3>{card.title}</h3>
            <p>{card.body}</p>
          </article>
        </li>
      ))}
    </ul>
  );
}
```

```css
/* card-enter-reduced: visible immediately, no spatial movement */
.card-enter-reduced {
  opacity: 1;
}
```

### GSAP Integration in React

```tsx
// HeroAnimation.tsx
import { useRef, useLayoutEffect } from 'react';
import { gsap } from 'gsap';
import { useReducedMotion } from './useReducedMotion';

export function HeroAnimation() {
  const containerRef  = useRef<HTMLDivElement>(null);
  const headlineRef   = useRef<HTMLHeadingElement>(null);
  const sublineRef    = useRef<HTMLParagraphElement>(null);
  const ctaRef        = useRef<HTMLButtonElement>(null);
  const reduced       = useReducedMotion();

  useLayoutEffect(() => {
    const ctx = gsap.context(() => {
      if (reduced) {
        // Reduced motion: snap elements to final state, no tween
        gsap.set([headlineRef.current, sublineRef.current, ctaRef.current], {
          opacity: 1,
          y: 0,
        });
        return;
      }

      // Full animation: staggered slide-up entrance
      const tl = gsap.timeline({ defaults: { ease: 'power3.out' } });
      tl.from(headlineRef.current, { y: 40, opacity: 0, duration: 0.7 })
        .from(sublineRef.current,  { y: 24, opacity: 0, duration: 0.5 }, '-=0.3')
        .from(ctaRef.current,      { y: 16, opacity: 0, duration: 0.4 }, '-=0.2');
    }, containerRef);

    return () => ctx.revert(); // cleanup on unmount
  }, [reduced]); // re-run if user toggles OS setting while on page

  return (
    <div ref={containerRef} className="hero">
      <h1 ref={headlineRef}>Build faster, ship confident.</h1>
      <p ref={sublineRef}>The platform for senior engineers.</p>
      <button ref={ctaRef} type="button">Get started</button>
    </div>
  );
}
```

### GSAP `matchMedia` — The Library-Native Approach

```tsx
// HeroAnimationMatchMedia.tsx
// Preferred for complex GSAP setups — no React hook dependency needed.
import { useRef, useLayoutEffect } from 'react';
import { gsap } from 'gsap';

export function HeroAnimationMatchMedia() {
  const containerRef = useRef<HTMLDivElement>(null);

  useLayoutEffect(() => {
    const mm = gsap.matchMedia();

    mm.add(
      {
        // Two contexts: one for each motion preference
        isReduced:    '(prefers-reduced-motion: reduce)',
        isAnimatable: '(prefers-reduced-motion: no-preference)',
      },
      (context) => {
        const { isReduced } = context.conditions as { isReduced: boolean; isAnimatable: boolean };

        if (isReduced) {
          gsap.set(containerRef.current, { opacity: 1 });
          return;
        }

        const tl = gsap.timeline();
        tl.from(containerRef.current, { opacity: 0, y: 32, duration: 0.6 });
      }
    );

    return () => mm.revert();
  }, []);

  return <div ref={containerRef} className="hero">Content</div>;
}
```

---

## Angular Implementation

### `MotionService` with Signals

```typescript
// motion.service.ts
import { Injectable, signal, computed, effect, OnDestroy } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class MotionService implements OnDestroy {
  private readonly query = '(prefers-reduced-motion: reduce)';
  private readonly mediaQuery: MediaQueryList | null =
    typeof window !== 'undefined' ? window.matchMedia(this.query) : null;

  // Core signal — true when OS has reduce-motion set
  private readonly _prefersReduced = signal<boolean>(
    this.mediaQuery?.matches ?? false
  );

  // Public readonly signal for consumers
  readonly prefersReduced = this._prefersReduced.asReadonly();

  // Derived: animation duration in ms (0 when reduced, full value when allowed)
  readonly durationMs = computed(() => (this._prefersReduced() ? 0 : 400));

  // Derived: CSS transition string for inline styles
  readonly transitionStyle = computed(() =>
    this._prefersReduced()
      ? 'none'
      : 'transform 0.4s cubic-bezier(0.4, 0, 0.2, 1)'
  );

  private readonly changeHandler = (event: MediaQueryListEvent) => {
    this._prefersReduced.set(event.matches);
  };

  constructor() {
    this.mediaQuery?.addEventListener('change', this.changeHandler);
  }

  ngOnDestroy() {
    this.mediaQuery?.removeEventListener('change', this.changeHandler);
  }
}
```

### Animated Panel Component

```typescript
// animated-panel.component.ts
import {
  Component, Input, inject, computed, HostBinding
} from '@angular/core';
import { MotionService } from './motion.service';

@Component({
  selector: 'app-animated-panel',
  standalone: true,
  template: `<ng-content />`,
  styles: [`
    :host {
      display: block;
      will-change: transform, opacity;
    }
  `],
})
export class AnimatedPanelComponent {
  @Input() isOpen = false;

  private motionService = inject(MotionService);

  // Bind transform directly as a host style — driven by signal
  @HostBinding('style.transform')
  get transform(): string {
    return this.isOpen ? 'translateX(0)' : 'translateX(-100%)';
  }

  @HostBinding('style.transition')
  get transition(): string {
    return this.motionService.transitionStyle();
  }

  @HostBinding('attr.aria-hidden')
  get ariaHidden(): boolean {
    return !this.isOpen;
  }
}
```

### Card List with Reduced Motion Branching

```typescript
// card-list.component.ts
import { Component, Input, inject, computed } from '@angular/core';
import { NgFor } from '@angular/common';
import { MotionService } from './motion.service';

interface Card {
  id: string;
  title: string;
  body: string;
}

@Component({
  selector: 'app-card-list',
  standalone: true,
  imports: [NgFor],
  template: `
    <ul>
      <li
        *ngFor="let card of cards; let i = index; trackBy: trackById"
        [class]="animationClass()"
        [style.animation-delay]="animationDelay(i)"
      >
        <article>
          <h3>{{ card.title }}</h3>
          <p>{{ card.body }}</p>
        </article>
      </li>
    </ul>
  `,
  styles: [`ul { list-style: none; padding: 0; }`],
})
export class CardListComponent {
  @Input() cards: Card[] = [];

  private motionService = inject(MotionService);

  // Switch animation class based on preference
  readonly animationClass = computed(() =>
    this.motionService.prefersReduced() ? 'card-enter-reduced' : 'card-enter'
  );

  // Stagger only when motion is allowed
  animationDelay(index: number): string {
    return this.motionService.prefersReduced() ? '0ms' : `${index * 80}ms`;
  }

  trackById(_: number, card: Card): string {
    return card.id;
  }
}
```

### GSAP Hero in Angular

```typescript
// hero.component.ts
import {
  Component, ElementRef, ViewChild, AfterViewInit,
  OnDestroy, inject, effect
} from '@angular/core';
import { gsap } from 'gsap';
import { MotionService } from './motion.service';

@Component({
  selector: 'app-hero',
  standalone: true,
  template: `
    <section #container class="hero">
      <h1 #headline>Build faster, ship confident.</h1>
      <p  #subline>The platform for senior engineers.</p>
      <button #cta type="button">Get started</button>
    </section>
  `,
})
export class HeroComponent implements AfterViewInit, OnDestroy {
  @ViewChild('container') containerEl!: ElementRef<HTMLElement>;
  @ViewChild('headline')  headlineEl!:  ElementRef<HTMLHeadingElement>;
  @ViewChild('subline')   sublineEl!:   ElementRef<HTMLParagraphElement>;
  @ViewChild('cta')       ctaEl!:       ElementRef<HTMLButtonElement>;

  private motionService = inject(MotionService);
  private gsapCtx?: gsap.Context;

  ngAfterViewInit() {
    this.buildAnimation();

    // Re-build animation if the OS preference changes while the component is live
    effect(() => {
      // Reading the signal inside effect() registers the dependency
      const _ = this.motionService.prefersReduced();
      this.buildAnimation();
    });
  }

  private buildAnimation() {
    this.gsapCtx?.revert();
    this.gsapCtx = gsap.context(() => {
      const elements = [
        this.headlineEl.nativeElement,
        this.sublineEl.nativeElement,
        this.ctaEl.nativeElement,
      ];

      if (this.motionService.prefersReduced()) {
        gsap.set(elements, { opacity: 1, y: 0 });
        return;
      }

      const tl = gsap.timeline({ defaults: { ease: 'power3.out' } });
      tl.from(this.headlineEl.nativeElement, { y: 40, opacity: 0, duration: 0.7 })
        .from(this.sublineEl.nativeElement,  { y: 24, opacity: 0, duration: 0.5 }, '-=0.3')
        .from(this.ctaEl.nativeElement,      { y: 16, opacity: 0, duration: 0.4 }, '-=0.2');
    }, this.containerEl.nativeElement);
  }

  ngOnDestroy() {
    this.gsapCtx?.revert();
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Spatial animations disabled for reduced-motion users | `@media (prefers-reduced-motion: reduce)` sets duration to `0.01ms` on all transitions/animations |
| Safe opacity/fade animations preserved | Specific selectors after the nuclear rule re-enable opacity-only animations |
| OS preference change handled without page reload | `window.matchMedia('...').addEventListener('change', handler)` in hook/service |
| GSAP timelines respect the preference | `gsap.matchMedia()` or manual check of `prefersReduced` signal before building timeline |
| Third-party libraries (Lottie, Framer Motion, etc.) wrapped | Each library has its own config; wrap with hook/service-driven boolean to `pause()` or set `autoplay: false` |
| SSR/server rendering does not crash | Guard all `window.matchMedia` calls with `typeof window !== 'undefined'` |
| Reduced motion preference is unit-testable | Mock `window.matchMedia` in vitest/jest; test that `durationMs` returns `0` |
| Animated decorative elements (particles, parallax) stopped entirely | `useReducedMotion` / `prefersReduced` signal used to skip `requestAnimationFrame` loops |
| Scroll-behavior respects the preference | `scroll-behavior: auto !important` inside the media query |
| User does not lose essential visual feedback | Loading states, success indicators use opacity fallback, not complete removal |

---

## Production Pitfalls

**1. Applying `transition: none` removes ALL transitions including opacity**
The nuclear `*` rule is effective but indiscriminate. A button hover that only fades between two colors is perfectly safe — yet it gets killed. Fix: after the nuclear rule, re-enable specific opacity-only transitions using more-specific selectors. Alternatively, use the inverted `prefers-reduced-motion: no-preference` approach for new code so you never have to undo anything.

**2. `matchMedia` is called on every render instead of once**
Calling `window.matchMedia(query)` synchronously inside a render function (React) or a getter (Angular) is fine for reading the current value, but attaching a `change` listener inline causes a new listener to be registered on every render/change-detection cycle. Fix: in React, initialize state from `matchMedia` in the `useState` initializer (run once) and attach the listener only inside `useEffect`. In Angular, attach the listener once in the constructor or `ngOnInit`, not in a computed property.

**3. GSAP timelines are built once and never updated when the OS setting changes**
If the user starts in an environment where motion is allowed, the timeline is built with full animation. If they then enable "Reduce Motion" in their OS settings (possible mid-session on some platforms), the timeline does not revert. Fix: use `gsap.matchMedia()` which automatically reacts to media query changes, or in Angular use an `effect()` that watches the signal and calls `gsapCtx.revert()` before rebuilding.

**4. `animation-duration: 0.01ms` breaks GSAP and Web Animations API timelines**
GSAP honors the CSS `animation-duration` custom property in some configurations, but more critically, if your code uses the Web Animations API (`element.animate(...)`) those are not CSS animations and are not affected by the media query override at all. Fix: always check the reduced-motion preference in JS for script-driven animations. Do not rely on CSS overrides alone.

**5. Parallax scroll listeners are never stopped**
A rAF-driven parallax loop registered in `useEffect` or `ngAfterViewInit` keeps running regardless of the CSS media query. The elements stay at their base position thanks to CSS, but the JS loop burns CPU for no benefit. Fix: read the preference at the time the loop is registered and skip `requestAnimationFrame` entirely when reduced. If live toggling is needed, cancel the rAF handle inside the `matchMedia` change listener.

**6. Reduced motion fallback introduces a FOUC (flash of unstyled content)**
When using the inverted `no-preference` approach, elements start without transitions and animate in only when the media query fires. If the media query is evaluated asynchronously (e.g., in a React `useEffect` with no SSR initial value), there is a frame where `prefersReduced` is `false` by default and a full animation fires before the hook reads the real OS value. Fix: initialize `useState` synchronously using the `matchMedia().matches` getter inside the state initializer function, which runs synchronously on mount before first paint.

---

## Interview Angle

**Q: "A designer wants a rich entrance animation where cards slide in with a stagger. How do you make that accessible?"**

Strong answer covers three layers. First, CSS: define the `slide-up` keyframe animation and add a `@media (prefers-reduced-motion: reduce)` block that either zeroes the duration or swaps in a `fade-in` keyframe. Second, JS/framework: CSS cannot pass a "should animate?" flag to the component, so build a `useReducedMotion` hook (React) or `MotionService` signal (Angular) that reads `window.matchMedia` once synchronously and registers a `change` listener for live updates. The component reads this flag and conditionally applies the stagger delay — no delay when reduced. Third, the semantic distinction: "reduced motion" does not mean "no animation." Cards should still appear visibly, just without spatial translation that could cause disorientation.

**Follow-up: "What if the cards are animated by GSAP rather than CSS keyframes?"**

CSS media queries have no reach into GSAP timelines. The fix is either `gsap.matchMedia()` — which accepts named conditions including `prefers-reduced-motion: reduce` and automatically reverts the matched context when the preference changes — or, within a framework, reading the signal/hook value before building the timeline and re-running the build logic inside a reactive effect if the preference changes mid-session. The critical mistake to avoid is building the GSAP timeline once on mount and never revisiting it: the timeline will play the full animation for users who enable reduced motion after the component has already mounted.

**Follow-up: "How would you test that reduced motion is actually respected?"**

Mock `window.matchMedia` in the test environment to return `matches: true`. In React, render the component and assert that the element's class or inline style indicates the non-translating variant. For GSAP, spy on `gsap.set` to assert it was called instead of `gsap.timeline`. For Angular, set the `MotionService._prefersReduced` signal to `true` (expose a test helper or use `TestBed` to override the service) and assert that `durationMs()` returns `0` and components apply the correct classes. Never rely on end-to-end tests alone for this — the OS preference is hard to simulate in a headless browser reliably.
