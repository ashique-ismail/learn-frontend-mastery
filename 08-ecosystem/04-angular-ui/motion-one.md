# Motion One

## What It Is

Motion One is a lightweight (3.8 KB) animation library built on the native Web Animations API (WAAPI). It works with any framework (Angular, React, Vue, vanilla JS) and is significantly smaller than GSAP while covering most animation use cases.

Created by the same author as Framer Motion.

---

## Core Animations

```ts
import { animate, timeline, scroll, inView, stagger } from 'motion';

// Basic animation: element, keyframes, options
animate('#box', { x: 200, opacity: 1 }, { duration: 0.5, easing: 'ease-out' });

// Multiple properties with keyframes
animate('.card', {
  transform: ['translateY(20px)', 'translateY(0)'],
  opacity: [0, 1],
}, { duration: 0.4 });
```

---

## Timeline — Sequenced Animations

```ts
import { timeline } from 'motion';

timeline([
  ['.header',   { opacity: [0, 1], y: [-20, 0] }, { duration: 0.4 }],
  ['.subtitle', { opacity: [0, 1], x: [-20, 0] }, { duration: 0.3, at: '-0.2' }], // overlap
  ['.cta',      { opacity: [0, 1], scale: [0.9, 1] }, { duration: 0.3 }],
]);
// at: '-0.2' starts 0.2s before the previous animation ends
```

---

## Scroll-Driven Animations

```ts
import { scroll, animate } from 'motion';

// Animate as the user scrolls
scroll(
  animate('.progress-bar', { scaleX: [0, 1] }),
  { target: document.body }
);

// Animate element based on its position in the viewport
scroll(
  animate('.parallax-image', { y: ['-10%', '10%'] }),
  { target: document.querySelector('.parallax-section') }
);
```

---

## `inView` — Animate When Entering Viewport

```ts
import { inView, animate } from 'motion';

inView('.card', (info) => {
  animate(info.target, { opacity: [0, 1], y: [20, 0] }, { duration: 0.5 });
  // Return a cleanup function if you want reverse animation on exit
  return () => animate(info.target, { opacity: 0 });
});
```

---

## `stagger` — Staggered List Animation

```ts
import { animate, stagger } from 'motion';

animate(
  '.list-item',
  { opacity: [0, 1], y: [20, 0] },
  { delay: stagger(0.05), duration: 0.3 }
);
```

---

## Angular Integration

```ts
import { Component, ElementRef, OnInit, viewChild } from '@angular/core';
import { animate, inView } from 'motion';

@Component({
  selector: 'app-animated-section',
  template: `
    <section #section>
      <h2 #title>Section Title</h2>
      <p #content>Content here</p>
    </section>
  `
})
export class AnimatedSectionComponent implements OnInit {
  title = viewChild<ElementRef>('title');
  content = viewChild<ElementRef>('content');

  ngOnInit() {
    const titleEl = this.title()?.nativeElement;
    const contentEl = this.content()?.nativeElement;

    inView(titleEl, () => {
      animate(titleEl, { opacity: [0, 1], y: [20, 0] }, { duration: 0.5 });
      animate(contentEl, { opacity: [0, 1], y: [20, 0] }, { duration: 0.5, delay: 0.15 });
    });
  }
}
```

---

## Performance: WAAPI vs JS Animation

Motion One uses the browser's native **Web Animations API**:
- Animations run on the compositor thread (GPU) for `transform` and `opacity`
- No main thread blocking for smooth animations
- Browser handles interpolation natively

Compared to `requestAnimationFrame`-based libraries (GSAP for complex animations):
- WAAPI: better default performance, less code
- GSAP: more control (pause/reverse/scrub), better for timeline orchestration

---

## Comparison with GSAP

| | Motion One | GSAP |
|---|---|---|
| Bundle size | 3.8 KB | ~30 KB (core) + plugins |
| Scroll animations | Built-in | ScrollTrigger plugin |
| Timeline | Basic | Full-featured |
| Pause/reverse/scrub | Limited | Full control |
| Stagger | Built-in | Built-in |
| Physics | No | Via GSAP plugins |
| Best for | Simpler animations, scroll effects | Complex choreography, games, interactive |

---

## Comparison with @angular/animations

| | Motion One | @angular/animations |
|---|---|---|
| Integration | Imperative (JS) | Declarative (template) |
| Trigger | Manual | Angular state changes |
| Scroll animations | Yes | No |
| Timeline | Yes | Limited (with `group`/`sequence`) |
| Bundle size | 3.8 KB | Part of @angular/core |
| Best for | Custom interactions, scroll effects | State transitions, route animations |

---

## Common Interview Questions

**Q: What is the Web Animations API?**
A browser-native JavaScript API for creating animations: `element.animate(keyframes, options)`. It runs on the compositor thread for GPU-accelerated animations and is the foundation Motion One and Framer Motion (for non-layout animations) are built on.

**Q: When would you choose Motion One over @angular/animations?**
For scroll-driven animations, enter-viewport animations, or timeline-sequenced animations that aren't tied to Angular state changes. `@angular/animations` is better when you want animations to automatically respond to component state and template changes.

**Q: Does Motion One work with Angular SSR?**
Motion One manipulates the DOM, so it must run in the browser. Use `isPlatformBrowser()` or `AfterViewInit` lifecycle hook (which only runs client-side) to initialize animations.
