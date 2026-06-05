# GSAP with Angular

## The Idea

**In plain English:** GSAP is a JavaScript library that lets you animate things on a webpage — like making a heading slide in, a button bounce, or elements fade as you scroll. Angular is a framework for building web apps, and this topic is about using GSAP's animation superpowers inside Angular's component system safely and efficiently.

**Real-world analogy:** Imagine a theatre stage director (Angular) who controls when actors enter the scene. The director only gives the cue after the stage is fully set and all props are in place. GSAP is the choreographer who tells each actor exactly how to move once they're on stage.

- The stage director (Angular) = the component lifecycle that controls when code runs
- Waiting until the stage is set = `AfterViewInit`, the lifecycle hook that runs only after the DOM is ready
- The choreographer's instructions = GSAP timeline that defines how and when each element animates

---

## Overview

GSAP (GreenSock Animation Platform) works in Angular the same way it works anywhere — imperative, DOM-manipulating animations. The Angular integration focuses on:
- Using `AfterViewInit` lifecycle for DOM-safe initialization
- Cleaning up animations in `ngOnDestroy`
- Working with Angular's `ElementRef` and `ViewChild`

---

## Basic Setup

```bash
npm install gsap
```

```ts
import gsap from 'gsap';
import { ScrollTrigger } from 'gsap/ScrollTrigger';

// Register plugins once (e.g., in main.ts or app.component.ts constructor)
gsap.registerPlugin(ScrollTrigger);
```

---

## Component Integration with `AfterViewInit`

```ts
import {
  Component,
  ElementRef,
  ViewChild,
  AfterViewInit,
  OnDestroy,
  viewChildren,
  inject,
  PLATFORM_ID,
  isPlatformBrowser,
} from '@angular/core';
import gsap from 'gsap';

@Component({
  selector: 'app-hero',
  template: `
    <section class="hero" #heroSection>
      <h1 #heroTitle>Build Amazing Apps</h1>
      <p #heroSubtitle>With Angular and GSAP</p>
      <button #heroCta>Get Started</button>
    </section>
  `
})
export class HeroComponent implements AfterViewInit, OnDestroy {
  @ViewChild('heroSection') heroSection!: ElementRef<HTMLElement>;
  @ViewChild('heroTitle') heroTitle!: ElementRef<HTMLElement>;
  @ViewChild('heroSubtitle') heroSubtitle!: ElementRef<HTMLElement>;
  @ViewChild('heroCta') heroCta!: ElementRef<HTMLElement>;

  private timeline?: gsap.core.Timeline;
  private platformId = inject(PLATFORM_ID);

  ngAfterViewInit() {
    // Guard for SSR — GSAP cannot run on the server
    if (!isPlatformBrowser(this.platformId)) return;

    this.timeline = gsap.timeline();

    this.timeline
      .from(this.heroTitle.nativeElement, {
        opacity: 0, y: 40, duration: 0.7, ease: 'power2.out'
      })
      .from(this.heroSubtitle.nativeElement, {
        opacity: 0, y: 20, duration: 0.5, ease: 'power2.out'
      }, '-=0.3')
      .from(this.heroCta.nativeElement, {
        opacity: 0, scale: 0.9, duration: 0.4, ease: 'back.out(1.7)'
      }, '-=0.2');
  }

  ngOnDestroy() {
    // Kill timeline and all animations on destroy
    this.timeline?.kill();
    ScrollTrigger.getAll().forEach(st => st.kill()); // if using ScrollTrigger
  }
}
```

---

## List Stagger with `@for` / `*ngFor`

```ts
@Component({
  template: `
    <ul #featureList>
      @for (feature of features; track feature.id) {
        <li class="feature-item">{{ feature.name }}</li>
      }
    </ul>
  `
})
export class FeaturesComponent implements AfterViewInit, OnDestroy {
  @ViewChild('featureList') featureList!: ElementRef;

  features = [
    { id: 1, name: 'Fast' },
    { id: 2, name: 'Accessible' },
    { id: 3, name: 'TypeSafe' },
  ];

  private ctx?: gsap.Context;

  ngAfterViewInit() {
    this.ctx = gsap.context(() => {
      // Query inside the component's DOM — .feature-item selector scoped to featureList
      gsap.from('.feature-item', {
        opacity: 0,
        x: -30,
        stagger: 0.1,
        duration: 0.4,
        ease: 'power2.out',
      });
    }, this.featureList.nativeElement); // scope context to this element
  }

  ngOnDestroy() {
    this.ctx?.revert(); // cleans up all animations scoped to this context
  }
}
```

---

## ScrollTrigger Integration

```ts
import { ScrollTrigger } from 'gsap/ScrollTrigger';

@Component({
  selector: 'app-scroll-section',
  template: `
    <section #section class="scroll-section">
      <h2 #heading>{{ title }}</h2>
      <p #body>{{ content }}</p>
    </section>
  `
})
export class ScrollSectionComponent implements AfterViewInit, OnDestroy {
  @ViewChild('section') section!: ElementRef;
  @ViewChild('heading') heading!: ElementRef;
  @ViewChild('body') body!: ElementRef;

  private scrollTrigger?: ScrollTrigger;

  ngAfterViewInit() {
    gsap.from([this.heading.nativeElement, this.body.nativeElement], {
      opacity: 0,
      y: 40,
      stagger: 0.2,
      duration: 0.6,
      scrollTrigger: {
        trigger: this.section.nativeElement,
        start: 'top 80%',
        end: 'bottom 20%',
        toggleActions: 'play none none reverse',
      },
    });
  }

  ngOnDestroy() {
    // ScrollTrigger instances attached to this component's elements
    ScrollTrigger.getAll()
      .filter(st => this.section.nativeElement.contains(st.trigger as Node))
      .forEach(st => st.kill());
  }
}
```

---

## Signals-Based Animation Trigger

```ts
@Component({
  template: `
    <div #box class="box"></div>
    <button (click)="animate()">Animate</button>
  `
})
export class SignalAnimationComponent implements AfterViewInit {
  @ViewChild('box') box!: ElementRef;
  isAnimating = signal(false);

  ngAfterViewInit() {
    effect(() => {
      if (this.isAnimating()) {
        gsap.to(this.box.nativeElement, {
          x: 200,
          rotation: 360,
          duration: 1,
          ease: 'power2.inOut',
          onComplete: () => this.isAnimating.set(false),
        });
      } else {
        gsap.to(this.box.nativeElement, { x: 0, rotation: 0, duration: 0.5 });
      }
    });
  }

  animate() { this.isAnimating.set(true); }
}
```

---

## Common Interview Questions

**Q: Why must GSAP be initialized in `AfterViewInit` instead of `OnInit`?**
`OnInit` runs before Angular has created the component's DOM. `AfterViewInit` runs after the view is fully rendered — only then are `nativeElement` references valid and safe to animate.

**Q: How do you handle GSAP in SSR (Angular Universal)?**
GSAP manipulates DOM elements — it cannot run on Node.js. Use `isPlatformBrowser(this.platformId)` to guard all GSAP initialization. Server-rendered HTML will be static; GSAP animations start after hydration on the client.

**Q: How does `gsap.context()` help with Angular?**
It scopes all GSAP animations created inside the context to a specific DOM element. Calling `ctx.revert()` in `ngOnDestroy` undoes all animations and cleans up event listeners created within that context — preventing memory leaks.
