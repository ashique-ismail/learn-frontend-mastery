# GSAP with React

## What GSAP Is

GSAP (GreenSock Animation Platform) is the industry-standard JavaScript animation library. It provides performant, cross-browser animations for any DOM element, SVG, CSS property, or JavaScript object.

---

## `useGSAP` Hook (React 18 Compatible)

The official GSAP React hook handles cleanup automatically:

```bash
npm install gsap @gsap/react
```

```tsx
import { useGSAP } from '@gsap/react';
import gsap from 'gsap';
import { useRef } from 'react';

gsap.registerPlugin(useGSAP); // register the plugin

function AnimatedCard() {
  const cardRef = useRef<HTMLDivElement>(null);
  const titleRef = useRef<HTMLHeadingElement>(null);

  useGSAP(() => {
    // Animations defined here are automatically cleaned up on unmount
    gsap.from(cardRef.current, {
      opacity: 0,
      y: 50,
      duration: 0.6,
      ease: 'power2.out',
    });

    gsap.from(titleRef.current, {
      opacity: 0,
      x: -30,
      duration: 0.4,
      delay: 0.2,
    });
  }, { scope: cardRef }); // scope: queries inside cardRef only

  return (
    <div ref={cardRef} className="card">
      <h2 ref={titleRef}>Animated Card</h2>
    </div>
  );
}
```

---

## `gsap.context()` for Cleanup (Without `useGSAP`)

```tsx
useEffect(() => {
  const ctx = gsap.context(() => {
    // All GSAP animations created here are tracked
    gsap.from('.card', { opacity: 0, y: 30, stagger: 0.1 });
    gsap.to('.title', { color: 'blue', duration: 1 });
  }, containerRef); // scope to containerRef

  return () => ctx.revert(); // kills all tracked animations on unmount
}, []);
```

---

## Core Animation Methods

```ts
// gsap.to() — animate FROM current state TO these values
gsap.to('.box', { x: 200, duration: 1 });

// gsap.from() — animate FROM these values TO current state
gsap.from('.box', { opacity: 0, scale: 0.5, duration: 0.5 });

// gsap.fromTo() — define both start and end explicitly
gsap.fromTo('.box',
  { opacity: 0, x: -100 },
  { opacity: 1, x: 0, duration: 0.8 }
);

// gsap.set() — set immediately (no animation)
gsap.set('.box', { opacity: 0, visibility: 'visible' });
```

---

## Timeline — Sequenced Animations

```tsx
function HeroSection() {
  const containerRef = useRef(null);

  useGSAP(() => {
    const tl = gsap.timeline({ defaults: { ease: 'power2.out' } });

    tl.from('.hero-title', { opacity: 0, y: 40, duration: 0.8 })
      .from('.hero-subtitle', { opacity: 0, y: 20, duration: 0.6 }, '-=0.4') // overlap
      .from('.hero-cta', { opacity: 0, scale: 0.8, duration: 0.4 }, '-=0.2')
      .from('.hero-image', { opacity: 0, x: 60, duration: 1 }, '<0.2'); // relative to prev start

  }, { scope: containerRef });

  return (
    <section ref={containerRef}>
      <h1 className="hero-title">Welcome</h1>
      <p className="hero-subtitle">Build amazing things</p>
      <button className="hero-cta">Get Started</button>
      <img className="hero-image" src="hero.png" />
    </section>
  );
}
```

Position parameter values:
- `'-=0.4'` — start 0.4s before previous ends (overlap)
- `'+=0.2'` — start 0.2s after previous ends (gap)
- `'<'` — start at same time as previous
- `'<0.2'` — start 0.2s after previous start

---

## ScrollTrigger

```bash
npm install gsap  # ScrollTrigger is included in GSAP 3
```

```tsx
import gsap from 'gsap';
import { ScrollTrigger } from 'gsap/ScrollTrigger';
import { useGSAP } from '@gsap/react';

gsap.registerPlugin(ScrollTrigger);

function ScrollAnimatedSection() {
  const sectionRef = useRef(null);

  useGSAP(() => {
    gsap.from('.animated-item', {
      opacity: 0,
      y: 60,
      stagger: 0.15,
      duration: 0.8,
      ease: 'power2.out',
      scrollTrigger: {
        trigger: sectionRef.current,
        start: 'top 80%',     // when top of trigger hits 80% of viewport
        end: 'bottom 20%',
        toggleActions: 'play none none reverse', // onEnter onLeave onEnterBack onLeaveBack
        // scrub: true,        // tie animation to scroll position
        // pin: true,          // pin element while animating
        // markers: true,      // debug: show start/end lines
      },
    });
  }, { scope: sectionRef });

  return (
    <section ref={sectionRef}>
      <div className="animated-item">Item 1</div>
      <div className="animated-item">Item 2</div>
      <div className="animated-item">Item 3</div>
    </section>
  );
}
```

---

## Stagger

```tsx
// Stagger: animate multiple elements with a delay between each
gsap.from('.card', {
  opacity: 0,
  y: 30,
  stagger: 0.1,           // 0.1s between each
  duration: 0.6,
  ease: 'power2.out',
});

// Advanced stagger
gsap.from('.grid-item', {
  opacity: 0,
  scale: 0,
  stagger: {
    amount: 0.8,          // total stagger duration
    from: 'center',       // start from center of selection
    grid: [3, 4],         // 3 rows x 4 cols grid layout
    ease: 'power2.inOut',
  },
});
```

---

## React-Specific Patterns

```tsx
// Animate on state change
function ExpandablePanel({ isOpen }: { isOpen: boolean }) {
  const panelRef = useRef<HTMLDivElement>(null);

  useGSAP(() => {
    gsap.to(panelRef.current, {
      height: isOpen ? 'auto' : 0,
      duration: 0.3,
      ease: 'power2.inOut',
    });
  }, { dependencies: [isOpen] }); // re-run when isOpen changes

  return <div ref={panelRef} style={{ overflow: 'hidden' }}>...</div>;
}
```

---

## Common Interview Questions

**Q: Why GSAP over CSS animations?**
GSAP handles: hardware-accelerated properties via transform (CSS can too), timeline sequencing (CSS can't), scroll-driven animations (ScrollTrigger), complex choreography with precise timing, pausing/reversing/scrubbing. CSS animations are great for simple cases; GSAP for complex choreography.

**Q: Why does GSAP recommend `useGSAP` over plain `useEffect`?**
`useGSAP` scopes animations to the component and automatically reverses them on unmount (killing timelines, ScrollTriggers, etc.) without manual cleanup. It handles the quirks of React 18 Strict Mode's double-mounting correctly.

**Q: How do you avoid FOUC (Flash of Unstyled Content) with GSAP `from()` animations?**
Use `gsap.set()` to set the initial state before any paint, then use `gsap.to()` instead of `gsap.from()`. Or set initial styles via CSS (opacity: 0) and animate to `opacity: 1`.
