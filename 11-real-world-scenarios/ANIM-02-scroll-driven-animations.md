# Scroll-Driven Animations

## The Idea

**In plain English:** Scroll-driven animations let you tie the playback of a CSS animation directly to how far a user has scrolled — either down the page or within a specific container. Instead of an animation playing once on a timer, its progress is controlled by scroll position. As you scroll down, elements fade in, progress bars fill up, or images scale into view — all driven purely by the scroll offset, with no JavaScript `scroll` event listeners required when using the native CSS API.

**Real-world analogy:** Think of a movie on a DVD player.

- The **CSS `animation-timeline: scroll()`** approach is like a movie where the progress bar on the DVD player is wired directly to the scroll wheel on your remote. Turn the scroll wheel down one notch, the movie advances one second. Scroll back up, the movie rewinds. You are the projector — your scroll position is the playhead.
- The **`view()` timeline** is like a sensor that activates only when a specific scene enters the TV screen. The animation plays only while that scene is visible in the viewport — not relative to the whole film, but relative to when *that element* is on screen.
- **GSAP ScrollTrigger** is like hiring a projectionist who watches the scroll wheel and manually cranks the film for you. More labor-intensive, requires a third party, but can show any film (not just CSS animations) and works in every theater (every browser).
- The **`prefers-reduced-motion` guard** is the theater's accessibility mode: the projectionist is told to skip all panning shots and cut directly to scenes, because the audience member gets motion sickness.

The key insight: `scroll()` is progress relative to the scroll container; `view()` is progress relative to when the element enters and exits the viewport. These are fundamentally different mental models.

---

## Learning Objectives

- Understand the difference between `scroll()` and `view()` timelines and when to choose each
- Use `animation-range` to control which portion of scroll progress triggers the animation
- Apply `@keyframes` with a scroll timeline to build a reading-progress bar, reveal-on-scroll, and parallax effect
- Implement GSAP ScrollTrigger as a complete JavaScript fallback for browsers without native support
- Guard all motion with `prefers-reduced-motion` at both the CSS and JS layers
- Integrate scroll animations in React (with `useEffect` + `IntersectionObserver` fallback) and Angular (with signals + a directive)

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Animate an element as it enters the viewport | ✅ with `animation-timeline: view()` | — |
| Fill a progress bar tied to page scroll | ✅ with `animation-timeline: scroll()` | — |
| Play a scroll animation only once (no rewind) | ❌ | CSS scroll timelines always reverse when you scroll back up |
| Trigger a callback when scroll reaches a threshold | ❌ | CSS cannot call JavaScript functions |
| Coordinate multiple elements with staggered delays based on scroll velocity | ❌ | CSS has no concept of scroll velocity |
| Support Safari < 18 and Firefox < 110 | ❌ | `animation-timeline` is not supported; needs JS polyfill or fallback |
| Scrub a complex SVG path-drawing animation | ⚠️ | Possible with `stroke-dashoffset` + scroll timeline, but fragile |
| Animate layout properties (`height`, `grid-template`) | ❌ | Only transform/opacity/filter can be composited; layout animations require JS |

**Conclusion:** CSS scroll timelines own the simple, stateless, reversible effects. JS (GSAP or IntersectionObserver) owns one-shot reveals, callbacks, complex sequencing, and cross-browser compatibility.

---

## HTML & CSS Foundation

### Reading Progress Bar — `scroll()` Timeline

```html
<!-- Fixed bar at the top of the page -->
<div class="reading-progress" role="progressbar" aria-label="Reading progress" aria-valuemin="0" aria-valuemax="100"></div>

<!-- Page content -->
<main class="content">
  <article><!-- long article content --></article>
</main>
```

```css
/* ─── Tokens ─── */
:root {
  --progress-height: 4px;
  --progress-color: #0066cc;
  --reveal-duration: 1ms; /* non-zero required for animation-timeline to attach */
}

/* ─── Reading progress bar ─── */
/*
  animation-timeline: scroll() attaches to the root scroller by default.
  scaleX() goes from 0 to 1 as the page scrolls from top to bottom.
  transform-origin: left ensures it grows from the left edge, not the center.
*/
@keyframes grow-progress {
  from { transform: scaleX(0); }
  to   { transform: scaleX(1); }
}

.reading-progress {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: var(--progress-height);
  background: var(--progress-color);
  transform-origin: left center;
  transform: scaleX(0);                    /* initial state */
  animation: grow-progress linear;
  animation-timeline: scroll();            /* tied to root scroll container */
  z-index: 1000;
}
```

### Reveal on Scroll — `view()` Timeline

```css
/*
  animation-timeline: view() ties animation progress to how much of
  the element is within the viewport.
  animation-range controls which portion of that intersection triggers the animation.
  
  "entry 0% entry 30%" means:
    - 0%:  the element's leading edge just entered the viewport (animation at 0%)
    - 30%: the element has scrolled 30% of its own height into the viewport (animation at 100%)
  After 30%, the animation holds at its final keyframe value.
*/
@keyframes fade-up {
  from {
    opacity: 0;
    transform: translateY(40px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.reveal {
  animation: fade-up linear both;
  animation-timeline: view();
  animation-range: entry 0% entry 30%;
}

/*
  "both" fill mode keeps the final state after the animation completes
  and the initial state before it begins.
  Without "both", elements would flash back to opacity:0 after passing.
*/
```

### Stagger Multiple Elements with `animation-range`

```css
/*
  Each card gets the same timeline but a different range start,
  creating a natural stagger without JavaScript.
*/
.card-grid .card:nth-child(1) { animation-range: entry 0%  entry 25%; }
.card-grid .card:nth-child(2) { animation-range: entry 5%  entry 30%; }
.card-grid .card:nth-child(3) { animation-range: entry 10% entry 35%; }

/* All three share the same keyframe */
.card-grid .card {
  animation: fade-up linear both;
  animation-timeline: view();
}
```

### Parallax Effect — `scroll()` with Named Timeline

```css
/*
  Named scroll timelines let you attach to a specific scrolling container,
  not just the root. This enables parallax within a modal or overflow container.
*/
.parallax-section {
  scroll-timeline-name: --section-scroll;
  scroll-timeline-axis: block;
  overflow-y: scroll;
  height: 400px;
}

@keyframes parallax-move {
  from { transform: translateY(0); }
  to   { transform: translateY(-60px); }
}

.parallax-bg {
  animation: parallax-move linear;
  animation-timeline: --section-scroll;
}
```

### Global Reduced-Motion Guard

```css
/*
  This single media query disables ALL scroll-driven animations.
  Place it at the end of your stylesheet so specificity doesn't
  fight the individual animation declarations above.
*/
@media (prefers-reduced-motion: reduce) {
  .reading-progress,
  .reveal,
  .card-grid .card,
  .parallax-bg {
    animation: none;
    /* Ensure revealed elements are still visible */
    opacity: 1;
    transform: none;
  }
}
```

**What CSS owns:** progress bar scrubbing, entry-based reveals, staggered ranges, named container timelines, and reduced-motion fallback.

**What CSS cannot own:** one-shot (play-once) animations, scroll velocity detection, triggering callbacks, complex SVG sequence coordination, and Safari/Firefox support before 2024.

---

## React Implementation

### Feature Detection Hook

```tsx
// useScrollTimeline.ts
import { useEffect, useState } from 'react';

/**
 * Returns true only when the browser natively supports
 * CSS animation-timeline. Used to decide whether to load GSAP.
 */
export function useScrollTimelineSupport(): boolean {
  const [supported, setSupported] = useState(false);

  useEffect(() => {
    // CSS.supports() checks the property+value pair
    const ok = CSS.supports('animation-timeline', 'scroll()');
    setSupported(ok);
  }, []);

  return supported;
}
```

### Reveal-on-Scroll with IntersectionObserver Fallback

```tsx
// useRevealOnScroll.ts
import { useEffect, useRef, RefObject } from 'react';

interface RevealOptions {
  threshold?: number;   // 0–1, fraction of element visible before triggering
  once?: boolean;       // if true, never re-hides after first reveal
  rootMargin?: string;  // e.g. "0px 0px -10% 0px"
}

/**
 * Attaches an IntersectionObserver to the returned ref.
 * When the element enters the viewport, adds "is-revealed" class.
 * Use this as the JS fallback when animation-timeline is unsupported.
 */
export function useRevealOnScroll<T extends HTMLElement>(
  options: RevealOptions = {}
): RefObject<T> {
  const { threshold = 0.15, once = true, rootMargin = '0px' } = options;
  const ref = useRef<T>(null);

  useEffect(() => {
    const el = ref.current;
    if (!el) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          el.classList.add('is-revealed');
          if (once) observer.unobserve(el);
        } else if (!once) {
          el.classList.remove('is-revealed');
        }
      },
      { threshold, rootMargin }
    );

    observer.observe(el);
    return () => observer.disconnect();
  }, [threshold, once, rootMargin]);

  return ref;
}
```

### Reading Progress Bar Component

```tsx
// ReadingProgress.tsx
import { useEffect, useRef } from 'react';
import { useScrollTimelineSupport } from './useScrollTimeline';

export function ReadingProgress() {
  const nativeSupported = useScrollTimelineSupport();
  const barRef = useRef<HTMLDivElement>(null);

  // JS fallback for browsers without animation-timeline support
  useEffect(() => {
    if (nativeSupported) return;   // CSS handles it natively
    const bar = barRef.current;
    if (!bar) return;

    const update = () => {
      const { scrollTop, scrollHeight, clientHeight } = document.documentElement;
      const progress = scrollTop / (scrollHeight - clientHeight);
      bar.style.transform = `scaleX(${progress})`;
    };

    window.addEventListener('scroll', update, { passive: true });
    return () => window.removeEventListener('scroll', update);
  }, [nativeSupported]);

  return (
    <div
      ref={barRef}
      className={`reading-progress ${nativeSupported ? 'reading-progress--native' : ''}`}
      role="progressbar"
      aria-label="Reading progress"
      aria-valuemin={0}
      aria-valuemax={100}
    />
  );
}
```

```css
/* Two variants: native CSS vs JS-driven */
.reading-progress {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 4px;
  background: #0066cc;
  transform-origin: left center;
  transform: scaleX(0);
  z-index: 1000;
}

/* Native: CSS does all the work */
.reading-progress--native {
  animation: grow-progress linear;
  animation-timeline: scroll();
}

/* JS fallback: style.transform is set imperatively */

@media (prefers-reduced-motion: reduce) {
  .reading-progress {
    display: none;   /* progress bar is purely decorative; hide it */
  }
}
```

### GSAP ScrollTrigger Fallback Component

```tsx
// ScrollReveal.tsx
import { useEffect, useRef, ReactNode } from 'react';
import { useScrollTimelineSupport } from './useScrollTimeline';

interface Props {
  children: ReactNode;
  className?: string;
  delay?: number;       // stagger delay in seconds
}

/**
 * If CSS animation-timeline is supported, renders children with the
 * "reveal" CSS class and lets the browser handle the animation.
 *
 * If not, dynamically imports GSAP + ScrollTrigger and animates imperatively.
 * GSAP is loaded only once and only when needed — no bundle cost for
 * users on modern browsers.
 */
export function ScrollReveal({ children, className = '', delay = 0 }: Props) {
  const nativeSupported = useScrollTimelineSupport();
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (nativeSupported || !ref.current) return;

    let cleanup: (() => void) | undefined;

    (async () => {
      // Dynamic import keeps GSAP out of the initial bundle
      const { gsap } = await import('gsap');
      const { ScrollTrigger } = await import('gsap/ScrollTrigger');
      gsap.registerPlugin(ScrollTrigger);

      const el = ref.current!;

      // Set initial state
      gsap.set(el, { opacity: 0, y: 40 });

      const trigger = ScrollTrigger.create({
        trigger: el,
        start: 'top 85%',       // when element top is 85% down the viewport
        onEnter: () => {
          gsap.to(el, {
            opacity: 1,
            y: 0,
            duration: 0.6,
            delay,
            ease: 'power2.out',
          });
        },
        once: true,             // mirrors CSS "animation-fill-mode: both" + one-shot intent
      });

      cleanup = () => trigger.kill();
    })();

    return () => cleanup?.();
  }, [nativeSupported, delay]);

  // Check prefers-reduced-motion in JS as well
  const motionOk = !window.matchMedia('(prefers-reduced-motion: reduce)').matches;

  if (!motionOk) {
    // Skip all animation scaffolding; render plain
    return <div className={className}>{children}</div>;
  }

  return (
    <div
      ref={ref}
      className={`${nativeSupported ? 'reveal' : ''} ${className}`}
    >
      {children}
    </div>
  );
}
```

### Composed Page Example

```tsx
// ArticlePage.tsx
import { ReadingProgress } from './ReadingProgress';
import { ScrollReveal } from './ScrollReveal';

const cards = [
  { id: 1, title: 'Performance', body: 'Use transform and opacity for GPU-composited animations.' },
  { id: 2, title: 'Accessibility', body: 'Always guard motion with prefers-reduced-motion.' },
  { id: 3, title: 'Progressive enhancement', body: 'CSS-first, JS-fallback. Never the other way around.' },
];

export function ArticlePage() {
  return (
    <>
      <ReadingProgress />

      <main>
        <section className="card-grid">
          {cards.map((card, i) => (
            <ScrollReveal key={card.id} delay={i * 0.1}>
              <div className="card">
                <h2>{card.title}</h2>
                <p>{card.body}</p>
              </div>
            </ScrollReveal>
          ))}
        </section>
      </main>
    </>
  );
}
```

---

## Angular Implementation

### Scroll Animation Directive

```typescript
// scroll-reveal.directive.ts
import {
  Directive, ElementRef, Input, OnInit, OnDestroy, inject
} from '@angular/core';

/**
 * Attribute directive that applies scroll-reveal animation.
 * On browsers with native animation-timeline support, it adds the
 * "reveal" CSS class and steps aside. On older browsers it sets up
 * an IntersectionObserver and toggles "is-revealed" instead.
 */
@Directive({
  selector: '[appScrollReveal]',
  standalone: true,
})
export class ScrollRevealDirective implements OnInit, OnDestroy {
  @Input('appScrollRevealDelay') delay: number = 0;   // CSS animation-delay in ms
  @Input('appScrollRevealOnce') once: boolean = true;

  private el = inject(ElementRef<HTMLElement>);
  private observer?: IntersectionObserver;

  private get nativeSupported(): boolean {
    return CSS.supports('animation-timeline', 'scroll()');
  }

  private get reducedMotion(): boolean {
    return window.matchMedia('(prefers-reduced-motion: reduce)').matches;
  }

  ngOnInit() {
    const el: HTMLElement = this.el.nativeElement;

    if (this.reducedMotion) {
      // Make sure the element is visible with no animation overhead
      el.style.opacity = '1';
      el.style.transform = 'none';
      return;
    }

    if (this.nativeSupported) {
      el.classList.add('reveal');
      if (this.delay) {
        // Offset the animation-range start via a CSS custom property
        el.style.setProperty('--reveal-delay', `${this.delay}ms`);
      }
      return;
    }

    // IntersectionObserver fallback
    el.style.opacity = '0';
    el.style.transform = 'translateY(40px)';
    el.style.transition = `opacity 0.6s ease ${this.delay}ms, transform 0.6s ease ${this.delay}ms`;

    this.observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          el.style.opacity = '1';
          el.style.transform = 'none';
          if (this.once) this.observer?.unobserve(el);
        } else if (!this.once) {
          el.style.opacity = '0';
          el.style.transform = 'translateY(40px)';
        }
      },
      { threshold: 0.15 }
    );

    this.observer.observe(el);
  }

  ngOnDestroy() {
    this.observer?.disconnect();
  }
}
```

### Reading Progress Service

```typescript
// scroll-progress.service.ts
import { Injectable, signal, OnDestroy } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class ScrollProgressService implements OnDestroy {
  /** 0–1 fraction of page scrolled. Updated only when CSS fallback is active. */
  readonly progress = signal(0);

  private readonly nativeSupported =
    CSS.supports('animation-timeline', 'scroll()');

  private handler = () => {
    const { scrollTop, scrollHeight, clientHeight } = document.documentElement;
    this.progress.set(scrollTop / (scrollHeight - clientHeight));
  };

  constructor() {
    if (!this.nativeSupported) {
      window.addEventListener('scroll', this.handler, { passive: true });
    }
  }

  ngOnDestroy() {
    window.removeEventListener('scroll', this.handler);
  }
}
```

### Reading Progress Bar Component

```typescript
// reading-progress.component.ts
import { Component, inject, computed } from '@angular/core';
import { ScrollProgressService } from './scroll-progress.service';

@Component({
  selector: 'app-reading-progress',
  standalone: true,
  template: `
    <div
      class="reading-progress"
      [class.reading-progress--native]="isNative"
      [style.transform]="progressTransform()"
      role="progressbar"
      aria-label="Reading progress"
      [attr.aria-valuenow]="progressPct()"
      aria-valuemin="0"
      aria-valuemax="100"
    ></div>
  `,
})
export class ReadingProgressComponent {
  private svc = inject(ScrollProgressService);

  readonly isNative = CSS.supports('animation-timeline', 'scroll()');

  /** Only used in the JS-driven (non-native) path */
  readonly progressTransform = computed(() =>
    this.isNative ? '' : `scaleX(${this.svc.progress()})`
  );

  readonly progressPct = computed(() =>
    Math.round(this.svc.progress() * 100)
  );
}
```

### Standalone Page Component

```typescript
// features-page.component.ts
import { Component } from '@angular/core';
import { ScrollRevealDirective } from './scroll-reveal.directive';
import { ReadingProgressComponent } from './reading-progress.component';

interface FeatureCard {
  id: number;
  title: string;
  body: string;
}

@Component({
  selector: 'app-features-page',
  standalone: true,
  imports: [ScrollRevealDirective, ReadingProgressComponent],
  template: `
    <app-reading-progress />

    <main>
      <section class="card-grid">
        <div
          *ngFor="let card of cards; let i = index; trackBy: trackById"
          class="card"
          appScrollReveal
          [appScrollRevealDelay]="i * 100"
        >
          <h2>{{ card.title }}</h2>
          <p>{{ card.body }}</p>
        </div>
      </section>
    </main>
  `,
})
export class FeaturesPageComponent {
  cards: FeatureCard[] = [
    { id: 1, title: 'Performance',    body: 'GPU-composited properties only.' },
    { id: 2, title: 'Accessibility',  body: 'Respect prefers-reduced-motion.' },
    { id: 3, title: 'Progressive',    body: 'CSS first, JS fallback.' },
  ];

  trackById(_: number, card: FeatureCard) {
    return card.id;
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| `prefers-reduced-motion: reduce` disables CSS animations | `@media` block sets `animation: none` and resets `opacity`/`transform` |
| `prefers-reduced-motion` checked in JS fallback path | `window.matchMedia(...)` check before attaching IntersectionObserver or GSAP |
| GSAP animations skip when motion is reduced | Early return in `useEffect` / `ngOnInit` before GSAP import |
| Reading progress bar is labelled for screen readers | `role="progressbar"` with `aria-label`, `aria-valuemin`, `aria-valuemax`, `aria-valuenow` |
| Reveal animations do not hide content permanently | `animation-fill-mode: both` ensures final state is visible; IO fallback always ends at `opacity:1` |
| Parallax/scroll effects do not cause nausea | Effects are gentle (max 60px translate); no rotation or zoom on page scroll |
| GSAP loaded only when needed | Dynamic `import()` prevents bundle cost for users on modern browsers |
| Elements are visible without JavaScript | CSS `reveal` class only applied after feature detection; SSR renders content normally |

---

## Production Pitfalls

**1. `animation-timeline` requires a non-zero `animation-duration`**
Setting `animation-duration: 0` causes the browser to ignore the scroll timeline entirely. The element stays at the `from` keyframe. Fix: always set `animation-duration` to at least `1ms` when using a scroll or view timeline. The actual number is irrelevant since scroll position controls playback speed, not time.

**2. `view()` reverses the animation when scrolling back up**
Unlike a one-shot `IntersectionObserver`, `view()` timeline plays forward on scroll down and backward on scroll up. Cards that "reveal" on the way down will "un-reveal" on the way up. If you need one-shot behavior (most product pages do), use the IntersectionObserver JS path with `once: true`, not `view()`. Or accept the reversible behavior and design your UI around it.

**3. Stacking contexts break `view()` inside `overflow: hidden` parents**
`animation-timeline: view()` by default uses the nearest scroll ancestor. If a parent has `overflow: hidden` or `overflow: clip`, the timeline treats that parent as the scroll container, making the element immediately "fully in view" and jumping to its final state. Fix: use `animation-timeline: view(block root)` to explicitly anchor to the root scroller, or restructure the DOM to remove the overflow ancestor.

**4. GSAP ScrollTrigger and native CSS conflict if both activate on the same element**
If your feature detection fails silently (e.g., `CSS.supports` returns false on a browser that partially supports it), both the CSS `reveal` class and the GSAP animation will apply, causing a double-animation or opacity conflict. Fix: make the feature detection the single source of truth — if `nativeSupported` is true, never attach the GSAP trigger, and vice versa. Test in a real browsing context with DevTools device emulation, not just user-agent sniffing.

**5. `will-change: transform` on many elements causes GPU memory pressure**
Adding `will-change: transform` to every `.reveal` card promotes each to its own compositor layer. On a page with 100+ cards, this exhausts GPU memory on mid-range devices and causes worse performance than no optimization at all. Fix: only apply `will-change` to elements actively being animated. Remove it in an `animationend` listener, or never add it for scroll-timeline animations that continuously update.

**6. GSAP ScrollTrigger dynamic import causes a visible delay on first scroll**
If the GSAP bundle (around 30 KB gzipped) loads lazily on first scroll interaction, there is a 100–300ms window where elements are invisible (set to `opacity: 0`) but not yet animated. Users see a flash of blank content. Fix: preload GSAP with `<link rel="modulepreload">` on pages where it will be needed, or trigger the dynamic import on `DOMContentLoaded` rather than on first scroll.

**7. `animation-range` percentages are relative to the element size, not the viewport**
`animation-range: entry 0% entry 30%` means 30% of the *element's* height has entered the viewport, not 30% of the viewport height. A very tall element (e.g., a full-screen hero section) will animate extremely slowly because 30% of it is a huge scroll distance. Fix: use explicit lengths — `animation-range: entry 0px entry 200px` — or combine `cover` with percentage values: `animation-range: cover 0% cover 20%`.

---

## Interview Angle

**Q: "What is the difference between `animation-timeline: scroll()` and `animation-timeline: view()`?"**

`scroll()` ties the animation's playback to the scroll position of a scroll container — typically the page itself. When the user has scrolled 0% of the page, the animation is at 0%; at 100% of the page, the animation is at 100%. It is agnostic about what element is being animated and purely tracks the scroll container's progress. A reading progress bar is the canonical use case.

`view()` ties playback to how much of *that specific element* is visible within the scroll container's viewport. When the element's leading edge enters the viewport, the animation begins. When the element is fully visible, the animation is at its midpoint. When it exits, the animation reaches 100%. `animation-range` lets you narrow that window — `entry 0% entry 30%` means the full animation plays during just the entry phase. Cards fading in as they enter the viewport are the canonical use case.

**Follow-up: "How do you handle browsers that don't support `animation-timeline`?"**

The strategy is feature detection, not user-agent sniffing: `CSS.supports('animation-timeline', 'scroll()')` returns a boolean. On unsupported browsers, load GSAP ScrollTrigger via dynamic `import()` — this keeps the bundle size zero for users on modern browsers. For the simpler reveal-on-scroll case, `IntersectionObserver` is sufficient and ships in every browser since 2018. The CSS classes for the native path and the JS fallback must be mutually exclusive — both applying simultaneously causes conflicting transforms and opacity values. Always check `prefers-reduced-motion` in both layers: the CSS `@media` block disables the CSS path, and a `window.matchMedia` check bails out of the JS path before any animation is registered.
