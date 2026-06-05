# CSS Houdini

## The Idea

**In plain English:** CSS Houdini is a set of tools built into the browser that lets developers write code which plugs directly into how the browser draws and animates web pages — instead of fighting against the browser's built-in styling rules, you get to extend them from the inside. Think of "CSS" as the language that controls how a webpage looks, and "Houdini" as the escape hatch that lets your code reach inside the browser's paint-and-layout machinery.

**Real-world analogy:** Imagine a restaurant kitchen where all the standard dishes are cooked by the head chef (the browser) following fixed recipes (built-in CSS). Normally, if you want something the menu doesn't offer, a waiter has to run out to a food truck parked outside, get the dish made there, and carry it back in — slow and clunky. CSS Houdini is like being given your own dedicated station inside the kitchen, with the same stoves and tools as the head chef, so your custom dish is made right alongside everything else at full speed.

- The head chef's fixed recipes = the browser's built-in CSS rendering rules
- The food truck outside = slow JavaScript workarounds that run on the main thread
- Your dedicated station inside the kitchen = a Houdini worklet running inside the browser's paint or layout pipeline
- The dish you cook = a custom visual effect, layout, or animation that performs just like a native CSS feature

---

## Overview

CSS Houdini is a collection of browser APIs that expose the CSS engine's internals to JavaScript, allowing developers to extend CSS with custom properties, custom paint effects, custom layout algorithms, and custom animation worklets. Before Houdini, any CSS feature that browsers didn't natively support required JavaScript workarounds that were slow, janky, or inaccessible.

```text
Without Houdini — custom visual effects:
  JS reads element size → calculates pixel positions → draws to canvas
  or
  JS updates inline styles on every scroll/animation frame → layout thrash
  Problems:
    - Runs on main thread (blocks interaction)
    - Requires DOM/CSSOM reads (forces layout)
    - Not compositable (no GPU acceleration by default)
    - Can't be used in CSS declarations (not a CSS property)

With Houdini — custom paint worklet:
  Register once → use as background: paint(my-effect) in CSS
  Runs in a worker (off main thread)
  Composited by the browser's rendering pipeline
  Parameterized via CSS custom properties
  Performance: equivalent to native CSS backgrounds
```

Houdini consists of several distinct APIs that target different stages of the browser's rendering pipeline:

```text
Rendering pipeline stage → Houdini API
──────────────────────────────────────
JavaScript / Style      → Properties & Values API (CSS.registerProperty)
Layout                  → Layout API (CSS.layoutWorklet)
Paint                   → Paint API (CSS.paintWorklet)
Composite / Animation   → Animation Worklet API
```

## Properties & Values API

The most widely supported Houdini API. Defines custom properties (CSS variables) with a type, initial value, and inheritance behavior — making them truly typed and animatable.

```typescript
// Register a typed custom property
CSS.registerProperty({
  name: '--highlight-color',
  syntax: '<color>',
  inherits: false,
  initialValue: 'transparent',
});

CSS.registerProperty({
  name: '--progress',
  syntax: '<number>',
  inherits: false,
  initialValue: '0',
});

CSS.registerProperty({
  name: '--angle',
  syntax: '<angle>',
  inherits: false,
  initialValue: '0deg',
});
```

```css
/* Without registerProperty: CSS variables can't be transitioned */
/* They have no type, so the browser doesn't know how to interpolate */
.card {
  --offset: 0;
  transition: --offset 0.3s; /* Does nothing without Houdini */
}

/* With registerProperty: type <length> means browser can interpolate */
/* JS: CSS.registerProperty({ name: '--offset', syntax: '<length>', inherits: false, initialValue: '0px' }) */
.card {
  --offset: 0px;
  transform: translateY(var(--offset));
  transition: --offset 0.3s ease-out; /* Now works! */
}

.card:hover {
  --offset: -8px;
}
```

### Alternatively: `@property` CSS Rule

```css
/* @property does not require JavaScript — pure CSS Houdini */
@property --gradient-angle {
  syntax: '<angle>';
  inherits: false;
  initial-value: 0deg;
}

@property --shimmer-position {
  syntax: '<percentage>';
  inherits: false;
  initial-value: -100%;
}

/* Animated gradient border using typed custom property */
.animated-border {
  background: conic-gradient(
    from var(--gradient-angle),
    #6366f1,
    #8b5cf6,
    #ec4899,
    #6366f1
  );
  animation: rotate-gradient 3s linear infinite;
}

@keyframes rotate-gradient {
  to { --gradient-angle: 360deg; }
  /* This works ONLY because --gradient-angle is a typed <angle> property */
  /* Without @property, this keyframe animation does nothing */
}
```

## CSS Paint API (Paintlet)

A Paint Worklet runs in a worker thread and implements a `paint()` function that receives a rendering context (similar to Canvas 2D) and draws into it. The result is used as a CSS image.

```javascript
// my-highlight-paint.js — runs in a Worklet context (not the main thread)
registerPaint('highlighted-text', class {

  // Declare which CSS custom properties this paintlet uses
  // These values are passed into the paint() function
  static get inputProperties() {
    return ['--highlight-color', '--highlight-height', '--highlight-offset'];
  }

  // Does this paint depend on cursor/pointer position?
  static get contextOptions() {
    return { alpha: true };
  }

  paint(ctx, geom, properties) {
    const color = properties.get('--highlight-color').toString().trim();
    const height = parseInt(properties.get('--highlight-height').toString()) || 8;
    const offset = parseInt(properties.get('--highlight-offset').toString()) || 2;

    ctx.fillStyle = color || 'rgba(255, 220, 0, 0.6)';

    // Draw a highlight bar at the bottom of the element
    ctx.fillRect(
      0,                           // x
      geom.height - height - offset,  // y (from bottom)
      geom.width,                  // width
      height                       // height
    );
  }
});
```

```javascript
// main.js — register the worklet on the main thread
if ('paintWorklet' in CSS) {
  CSS.paintWorklet.addModule('/my-highlight-paint.js');
}
```

```css
@property --highlight-color {
  syntax: '<color>';
  inherits: true;
  initial-value: rgba(255, 220, 0, 0.6);
}

.heading-highlight {
  --highlight-color: rgba(99, 102, 241, 0.4);
  --highlight-height: 12;
  --highlight-offset: 4;
  background: paint(highlighted-text);
  /* Falls back gracefully if Houdini not supported */
}

/* Animate the highlight color */
.heading-highlight:hover {
  --highlight-color: rgba(236, 72, 153, 0.5);
  transition: --highlight-color 0.3s;
}
```

### Advanced Paint Worklet — Ripple Effect

```javascript
// ripple-paint.js
registerPaint('ripple', class {
  static get inputProperties() {
    return ['--ripple-x', '--ripple-y', '--ripple-progress', '--ripple-color'];
  }

  paint(ctx, geom, props) {
    const x = parseFloat(props.get('--ripple-x').toString()) || 0;
    const y = parseFloat(props.get('--ripple-y').toString()) || 0;
    const progress = parseFloat(props.get('--ripple-progress').toString()) || 0;
    const color = props.get('--ripple-color').toString().trim() || 'rgba(255,255,255,0.3)';

    if (progress <= 0) return;

    const maxRadius = Math.sqrt(
      Math.pow(Math.max(x, geom.width - x), 2) +
      Math.pow(Math.max(y, geom.height - y), 2)
    );

    ctx.fillStyle = color;
    ctx.globalAlpha = 1 - progress;
    ctx.beginPath();
    ctx.arc(x, y, maxRadius * progress, 0, 2 * Math.PI);
    ctx.fill();
  }
});
```

```css
@property --ripple-x { syntax: '<number>'; inherits: false; initial-value: 0; }
@property --ripple-y { syntax: '<number>'; inherits: false; initial-value: 0; }
@property --ripple-progress { syntax: '<number>'; inherits: false; initial-value: 0; }

.ripple-button {
  background: paint(ripple), var(--button-bg, #6366f1);
  position: relative;
  overflow: hidden;
}
```

```typescript
// Trigger the ripple animation with Web Animations API
function addRipple(element: HTMLElement, event: MouseEvent): void {
  const rect = element.getBoundingClientRect();
  const x = event.clientX - rect.left;
  const y = event.clientY - rect.top;

  element.style.setProperty('--ripple-x', String(x));
  element.style.setProperty('--ripple-y', String(y));

  element.animate(
    { '--ripple-progress': ['0', '1'] } as PropertyIndexedKeyframes,
    { duration: 600, easing: 'ease-out' }
  );
}

document.querySelectorAll('.ripple-button').forEach((btn) => {
  btn.addEventListener('click', (e) => addRipple(btn as HTMLElement, e as MouseEvent));
});
```

## CSS Layout API

Define custom layout algorithms — how children are positioned within a container. This is how flexbox and grid work internally; Houdini exposes that mechanism.

```javascript
// masonry-layout.js — custom masonry layout
registerLayout('masonry', class {
  static get inputProperties() {
    return ['--columns', '--gap'];
  }

  async intrinsicSizes() {}

  async layout(children, edges, constraints, styleMap) {
    const columns = parseInt(styleMap.get('--columns') || '3');
    const gap = parseInt(styleMap.get('--gap') || '16');

    const colWidth = (constraints.fixedInlineSize - gap * (columns - 1)) / columns;
    const colHeights = new Array(columns).fill(0);
    const childFragments = [];

    for (const child of children) {
      // Find shortest column
      const shortestCol = colHeights.indexOf(Math.min(...colHeights));

      const childFragment = await child.layoutNextFragment({
        fixedInlineSize: colWidth,
      });

      childFragments.push({
        fragment: childFragment,
        inlineOffset: shortestCol * (colWidth + gap),
        blockOffset: colHeights[shortestCol],
      });

      colHeights[shortestCol] += childFragment.blockSize + gap;
    }

    return {
      autoBlockSize: Math.max(...colHeights),
      childFragments,
    };
  }
});
```

```javascript
// Register the layout worklet
CSS.layoutWorklet.addModule('/masonry-layout.js');
```

```css
.masonry-grid {
  display: layout(masonry);
  --columns: 3;
  --gap: 16;
}

.masonry-grid > * {
  /* Children are laid out by the custom algorithm */
}
```

Note: The Layout API has limited browser support (Chrome only behind a flag as of 2025). The native CSS `masonry` value for `grid-template-rows` is now going through standardization and is a better choice for masonry specifically.

## Animation Worklet API

Run animations in a dedicated compositor thread, synchronized with scroll position or time, without ever touching the main thread:

```javascript
// scroll-animation.js (Animation Worklet)
registerAnimator('scroll-driven', class {
  animate(currentTime, effects) {
    // currentTime comes from scroll position (0–1 range)
    effects.forEach(effect => {
      effect.localTime = currentTime * 1000; // Convert to milliseconds
    });
  }
});
```

```javascript
// main.js — scroll-driven animation
async function setupScrollAnimation() {
  await CSS.animationWorklet.addModule('/scroll-animation.js');

  const element = document.querySelector('.parallax-element');

  const keyframeEffect = new KeyframeEffect(
    element,
    [{ transform: 'translateY(0px)' }, { transform: 'translateY(-200px)' }],
    { duration: 1000, fill: 'both' }
  );

  const scrollSource = document.scrollingElement;
  const scrollTimeline = new ScrollTimeline({
    scrollSource,
    orientation: 'vertical',
  });

  const workletAnimation = new WorkletAnimation(
    'scroll-driven',
    keyframeEffect,
    scrollTimeline
  );

  workletAnimation.play();
}
```

Note: The Animation Worklet API has been largely superseded by the native **CSS Scroll-Driven Animations** spec (Chrome 115+), which achieves the same results declaratively in CSS without JavaScript or worklets:

```css
/* Modern approach: CSS Scroll-Driven Animations (no Houdini needed) */
.parallax {
  animation: parallax-move linear both;
  animation-timeline: scroll(root block);
  animation-range: 0% 50%;
}

@keyframes parallax-move {
  from { transform: translateY(0); }
  to   { transform: translateY(-200px); }
}
```

## Why Houdini Matters for Performance

```text
JavaScript animation vs CSS Houdini comparison:

JS canvas overlay (common pattern):
  requestAnimationFrame callback
    → Read scroll position (CSSOM read → forces layout)
    → Calculate particle positions
    → Clear canvas
    → Draw 200 particles
  Runs on main thread
  Scroll/interaction can cause drops if rAF is busy
  GPU: canvas is composited but drawing is CPU

CSS Paint Worklet:
  Runs in PaintWorklet thread (off main thread)
  Triggered by browser's paint pipeline, not rAF
  Cannot read DOM (no layout thrash possible)
  Composited by the browser's rendering engine
  Parameterized via CSS custom properties (typed, animatable)
  CSS Transition / Web Animations API can animate the properties
    → Animation on compositor thread
    → Paint on worklet thread
    → Zero main thread involvement for animation loop

Result: 60fps animations even when main thread is busy processing data
```

```text
Performance characteristics:
  Paint Worklet     → Runs off main thread, can't read DOM
  Layout Worklet    → Runs off main thread, receives child sizes not DOM
  Animation Worklet → Runs on compositor thread, synced with frame clock
  @property/CSS.registerProperty → Runs in style resolution (main thread, fast)
```

## Browser Support

```text
Feature                          | Chrome | Edge | Firefox | Safari
─────────────────────────────────|────────|──────|─────────|───────
CSS.registerProperty             | 78+    | 79+  | 128+    | 16.4+
@property                        | 85+    | 85+  | 128+    | 16.4+
CSS Paint API (paintWorklet)     | 65+    | 79+  | No      | No
CSS Layout API (layoutWorklet)   | Flag   | Flag | No      | No
Animation Worklet API            | 71+    | 79+  | No      | No

Practical production use:
  @property / CSS.registerProperty → Use freely (good cross-browser support)
  Paint API → Chrome/Edge only, use feature detection with fallbacks
  Layout API → Experimental only, don't use in production
```

## Feature Detection and Fallbacks

```typescript
// Comprehensive feature detection
const houdiniSupport = {
  paintWorklet: 'paintWorklet' in CSS,
  layoutWorklet: 'layoutWorklet' in CSS,
  animationWorklet: 'animationWorklet' in CSS,
  registerProperty: 'registerProperty' in CSS,
  atProperty: CSS.supports('@property --test { syntax: "<color>"; inherits: false; initial-value: red; }'),
};

// Load paint worklet only if supported
async function initPaintWorklets(): Promise<void> {
  if (!houdiniSupport.paintWorklet) {
    console.log('CSS Paint API not supported, using fallback styles');
    document.documentElement.classList.add('no-houdini');
    return;
  }

  try {
    await CSS.paintWorklet.addModule('/paint/ripple.js');
    await CSS.paintWorklet.addModule('/paint/highlight.js');
    document.documentElement.classList.add('has-houdini');
  } catch (err) {
    console.error('Failed to load paint worklet:', err);
    document.documentElement.classList.add('no-houdini');
  }
}
```

```css
/* Fallback for no Paint API */
.highlight-text {
  /* Default: solid underline */
  text-decoration: underline 2px rgba(99, 102, 241, 0.6);
}

.has-houdini .highlight-text {
  text-decoration: none;
  background: paint(highlighted-text);
}
```

## Real-World Use Cases

```text
1. Animated gradient borders
   @property --gradient-angle → conic-gradient → @keyframes
   No JS, no Canvas, pure CSS

2. Custom progress indicators
   Paint worklet draws arc/path → parameterized by --progress custom property
   CSS animation drives --progress → off main thread

3. Particle/noise backgrounds
   Paint worklet generates noise pattern → deterministic for given seed
   Much more expressive than box-shadow or filter tricks

4. Ripple/focus animations on interactive elements
   Paint worklet draws expanding circle → driven by Web Animations API
   Input coordinates via CSS custom properties

5. Staggered skeleton loaders
   @property animates shimmer position → CSS handles multiple elements
   No JS timer synchronization needed

6. Advanced clip paths driven by scroll
   @property --clip-progress → clip-path: inset(calc(100% - var(--clip-progress) * 100%))
   CSS Scroll-Driven Animations or GSAP drives --clip-progress
```

## Interview Questions

1. **What is CSS Houdini and why was it created?**
   CSS Houdini is a set of browser APIs that expose the CSS engine's rendering pipeline to JavaScript. It was created because the gap between what browsers natively support in CSS and what designers want is large, and JavaScript workarounds for unsupported CSS features are slow (run on the main thread, cause layout thrash, block interaction) and inaccessible (not real CSS, can't cascade or inherit). Houdini lets developers extend the CSS engine itself — writing code that runs inside the browser's paint, layout, or animation stages — with the same performance characteristics as native CSS features.

2. **What is `@property` and what does it enable that regular CSS custom properties cannot do?**
   `@property` (and its JavaScript equivalent `CSS.registerProperty`) adds a type declaration, inheritance flag, and initial value to a CSS custom property. Regular CSS custom properties are untyped strings — the browser cannot interpolate between `--color: red` and `--color: blue` because it doesn't know they're colors. With `@property --my-color { syntax: '<color>'; }`, the browser understands the type and can animate/transition the property using color interpolation. This enables animating gradients, conic-gradients, and any CSS value that uses custom properties — which would be impossible with plain CSS variables.

3. **Explain the execution context of a CSS Paint Worklet. What can and cannot it access?**
   A Paint Worklet runs in a `PaintWorkletGlobalScope` — a special worker-like context separate from both the main thread and regular Web Workers. It CAN access: the Canvas 2D rendering context (limited subset), the element's geometry (width/height via the `geom` parameter), CSS custom property values (declared via `inputProperties`), and `Math` / basic JS primitives. It CANNOT access: the DOM, CSSOM, network (no fetch), storage, or other browser APIs. This isolation is intentional — it prevents layout thrash (no DOM reads) and allows the worklet to run off the main thread in the browser's paint pipeline.

4. **How does CSS Houdini's Paint API compare to using a Canvas element for custom visuals?**
   A Canvas element sits in the DOM stacking context — the JavaScript that draws to it runs on the main thread via `requestAnimationFrame`, competes with user interaction, and can cause jank under load. The draw calls themselves are CPU-bound. A Paint Worklet runs off the main thread in the browser's paint pipeline. It's composited by the browser like any other CSS background or image. Animations driven by CSS transitions/Web Animations API targeting the worklet's `inputProperties` run on the compositor thread. The result: paint worklet animations are smooth even when the main thread is busy, at the cost of a more constrained drawing API and no DOM access.

5. **What practical problem does `CSS.registerProperty` solve for design systems?**
   Design systems frequently use CSS custom properties for theming. Without `@property`, those variables cannot be transitioned or animated — hover effects on theme colors must use opacity tricks or shadow manipulation. With `@property`, a design system can declare `--brand-color` as a typed `<color>` property, and all components can use `transition: --brand-color 200ms` to get smooth color transitions for free. It also enables typed validation — the browser can warn when `--spacing-md` receives a non-`<length>` value. This makes CSS custom properties as robust as TypeScript types for CSS values.

6. **Describe a real scenario where a CSS Paint Worklet provides meaningful performance improvement over a JavaScript alternative.**
   An infinite scroll feed with highlighted keywords: a JavaScript approach would listen to `scroll` events, calculate which text nodes are visible, and add/modify DOM elements for highlights — constant DOM reads and writes blocking the main thread. With a Paint Worklet: the highlight logic is a `paint()` function that receives the highlight color and position as CSS custom properties. The Web Animations API animates those properties targeting the worklet. The browser computes the animation values on the compositor thread and feeds them to the Paint Worklet off the main thread. The main thread is never involved in the highlight animation loop, keeping scroll and interaction responsive.

7. **What is the current production readiness of each Houdini API?**
   `@property` / `CSS.registerProperty`: production-ready, broad support (Chrome 85+, Firefox 128+, Safari 16.4+). CSS Paint API: Chrome/Edge only, no Firefox/Safari support — use with feature detection and fallbacks, appropriate for progressive enhancement on Chromium-targeted apps. CSS Layout API: experimental/flags only — do not use in production. Animation Worklet: Chrome/Edge only, largely superseded by CSS Scroll-Driven Animations which are now the standard approach for scroll-linked animations. The practical production recommendation is: use `@property` freely, use Paint API as progressive enhancement for Chrome/Edge, avoid Layout and Animation Worklets.

8. **How would you architect a design system to leverage `@property` for animatable tokens?**
   Register all animatable design tokens at application init using `CSS.registerProperty` (or via `@property` in a base stylesheet): colors as `<color>`, spacing as `<length>`, transitions as `<time>`. Define all component styles using these custom properties. Add `transition` declarations referencing the token properties — components automatically get smooth transitions when tokens change. For theme switching (light/dark), animate the token values rather than toggling classes — this enables a smooth animated theme transition by animating `--bg-color`, `--text-color`, etc. The browser interpolates all typed tokens simultaneously, giving a cinematic theme switch with zero JavaScript animation code.
