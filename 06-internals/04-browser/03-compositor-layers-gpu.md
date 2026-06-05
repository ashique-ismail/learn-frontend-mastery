# Compositor Threads and GPU Layer Promotion

## The Idea

**In plain English:** When your browser draws a webpage, it splits the visual content into separate layers (like stacked transparent sheets) and hands them to your computer's graphics chip (GPU) to combine into the final image. This lets certain animations run silently on the GPU without interrupting the main program doing all the other work.

**Real-world analogy:** Imagine a stage production where the director (main thread) pre-paints scenery onto separate transparent slides, then hands them to a dedicated projectionist (compositor thread) who layers them on the screen. The projectionist can slide, fade, or stack the panels on the fly — without ever stopping the director from writing the next scene.

- The director pre-painting scenes = the main thread computing layout and paint
- Each transparent slide = a compositor layer (a GPU texture)
- The projectionist layering slides = the compositor thread combining layers on the GPU
- Physically sliding a panel = animating with `transform` or `opacity` (no repainting needed)

---

## Overview

The browser rendering pipeline has multiple threads. Understanding which work runs on which thread — and how to move expensive work off the main thread onto the compositor — is the foundation of achieving consistent 60fps animations and avoiding jank.

---

## The Rendering Threads

```
┌──────────────────────────────────────────────────────────┐
│  Main Thread                                             │
│  JavaScript → Style → Layout → Paint → Composite record │
│  (blocked by JS execution, layout, paint work)           │
├──────────────────────────────────────────────────────────┤
│  Compositor Thread                                       │
│  Composites already-painted layers                       │
│  Handles scroll and some animations independently        │
│  (never blocked by JS)                                   │
├──────────────────────────────────────────────────────────┤
│  Raster Threads (pool)                                   │
│  Convert paint commands to pixels (rasterize tiles)      │
│  Run on CPU or GPU via Skia                              │
├──────────────────────────────────────────────────────────┤
│  GPU Process                                             │
│  Uploads rasterized tiles to GPU memory                  │
│  Composites final frame via OpenGL/Metal/Vulkan          │
└──────────────────────────────────────────────────────────┘
```

### What the Compositor Thread Does

The compositor thread takes already-painted GPU textures (layers) and combines them into the final frame. Because it runs independently of the main thread:

- **Scroll is smooth** even when JavaScript is running
- **CSS animations on `transform` and `opacity` are smooth** — they animate on the compositor, never touching the main thread
- **Pinch-to-zoom is smooth** — composite-only operation

This is why `transform: translateY(...)` animates smoothly even during a long JS task, but `top: ...` does not — `top` triggers layout, which requires the main thread.

---

## Composite-Only Properties

Only two CSS properties can be animated entirely on the compositor thread, requiring no layout or paint work on the main thread:

| Property | Why compositor-only |
|---|---|
| `transform` (translate, rotate, scale, matrix) | Moves/resizes a pre-painted texture — no repaint |
| `opacity` | Adjusts alpha during compositing — no repaint |

```css
/* ✓ 60fps on compositor thread — never blocks main thread */
@keyframes slide-in {
  from { transform: translateX(-100%); }
  to   { transform: translateX(0); }
}

/* ❌ Triggers layout + paint on main thread every frame */
@keyframes slide-in-bad {
  from { left: -100%; }
  to   { left: 0; }
}

/* ✓ Fade — compositor-only */
@keyframes fade {
  from { opacity: 0; }
  to   { opacity: 1; }
}

/* ❌ Color change triggers repaint every frame */
@keyframes color-bad {
  from { background-color: red; }
  to   { background-color: blue; }
}
```

The `filter` property (blur, brightness, etc.) is also GPU-accelerated in modern browsers — it runs on the compositor and does not trigger layout.

---

## How Layer Promotion Works

The browser paints the page into a single layer by default. For elements expected to animate or scroll independently, it creates a **separate compositor layer** (a separate GPU texture). Compositing these textures together is cheap — it's just GPU blending.

### What Triggers Layer Promotion

```css
/* Explicit promotion hints */
will-change: transform;        /* modern — tells browser to promote */
will-change: opacity;
will-change: transform, opacity;

/* Legacy hack — still works but wastes memory */
transform: translateZ(0);
transform: translate3d(0, 0, 0);

/* Other promotion triggers (browser-determined) */
position: fixed;              /* scrolls independently from document */
video, canvas, iframe;        /* always separate layers */
/* CSS animations on transform/opacity */
/* overflow: scroll with hardware scrolling */
/* mix-blend-mode (some values) */
```

### DevTools: Layers Panel

```
DevTools → More Tools → Layers

Shows:
- All compositor layers on the page
- Why each was promoted ("will-change: transform", "position: fixed", etc.)
- Memory consumed per layer (texture size × RGBA bytes)
- Number of repaints per layer
```

---

## The Layer Memory Trade-off

Each compositor layer is a GPU texture — it consumes GPU memory proportional to its pixel area × 4 bytes (RGBA):

```
A 1920×1080 layer = 1920 × 1080 × 4 = ~8 MB GPU memory

100 such layers = ~800 MB GPU memory
→ memory pressure → GPU swapping → frame drops
```

**Layer explosion anti-pattern:**

```css
/* This promotes EVERY list item to its own layer */
.list-item {
  will-change: transform;  /* ← applied to 500 items = 500 layers */
}

/* Fix: only promote the item being animated */
.list-item--animating {
  will-change: transform;  /* applied dynamically in JS */
}
```

```javascript
// Add will-change before animation, remove after
item.classList.add('list-item--animating');
item.addEventListener('animationend', () => {
  item.classList.remove('list-item--animating');
}, { once: true });
```

---

## contain: layout | paint | size | style

The `contain` property lets you tell the browser that an element's subtree is isolated — changes inside it cannot affect anything outside it. This enables the browser to skip work for unaffected parts of the page.

```css
/* Contain layout: element's layout doesn't affect outside */
.widget {
  contain: layout;
}

/* Contain paint: element's children are clipped to its bounds */
/* Browser can skip painting this element if it's off-screen */
.card {
  contain: paint;     /* also creates a new stacking context */
}

/* content = layout + paint + style (common combination) */
.isolated-widget {
  contain: content;
}

/* strict = layout + paint + style + size */
.fully-isolated {
  contain: strict;
  width: 300px;  /* must set explicit size when contain: size */
  height: 200px;
}
```

**`contain: paint`** creates a new stacking context and prevents children from visually overflowing — the browser can skip painting the entire subtree if the container is off-screen. Critical for virtualized lists.

**`content-visibility: auto`** extends this further — the browser skips rendering off-screen content entirely:

```css
.lazy-section {
  content-visibility: auto;
  contain-intrinsic-size: 0 500px; /* estimated height to avoid scroll jitter */
}
```

---

## Diagnosing Jank in DevTools

### Identifying Paint Storms

1. DevTools → Rendering → Paint flashing (checkbox)
2. Interact with the page — green overlays show painted areas
3. If large areas flash on every scroll or animation → investigate

### Identifying Layer Count

1. DevTools → More Tools → Layers
2. Count layers and check memory
3. Look for `will-change: transform` applied to many elements
4. Check "Reasons for promotion" — `(no reason given)` means implicit promotion (browser-decided)

### Reading the Performance Flame Chart for Compositing

```
In the flame chart:
- "Update Layer Tree"   = browser determining which elements need layers
- "Composite Layers"    = compositor thread doing its work (usually <1ms)
- "Paint"               = main thread rasterizing (reduce this)
- "Rasterize Paint"     = raster thread work (GPU or CPU)

If "Composite Layers" takes >2ms → too many layers or textures too large
If "Paint" takes >5ms every frame → something is triggering repaint on the hot path
```

---

## CSS Properties: Rendering Cost Tiers

```
Tier 1 (cheapest — compositor only):
  transform, opacity, filter

Tier 2 (paint — no layout):
  color, background-color, border-color, box-shadow, outline
  visibility, border-radius (in some contexts)

Tier 3 (most expensive — triggers layout):
  width, height, padding, margin, border-width
  top, left, right, bottom (on positioned elements)
  font-size, line-height
  display, float, position
```

Animating Tier 3 properties triggers the full pipeline: Layout → Paint → Composite, every frame.

---

## FLIP Animation Technique

FLIP (First, Last, Invert, Play) lets you animate layout-triggering property changes using only `transform`:

```javascript
function flipAnimate(element, changeFn) {
  // First: record current position
  const first = element.getBoundingClientRect();

  // Last: apply the layout change
  changeFn();

  // Last: record new position
  const last = element.getBoundingClientRect();

  // Invert: transform element back to "first" position
  const dx = first.left - last.left;
  const dy = first.top  - last.top;
  element.style.transform = `translate(${dx}px, ${dy}px)`;

  // Play: animate to identity transform (compositor-only)
  requestAnimationFrame(() => {
    element.style.transition = 'transform 300ms ease-out';
    element.style.transform  = '';
    element.addEventListener('transitionend', () => {
      element.style.transition = '';
    }, { once: true });
  });
}

// Usage: animate a card from its old grid position to new position
flipAnimate(card, () => {
  grid.insertBefore(card, grid.firstChild); // triggers layout
});
```

The View Transitions API automates FLIP for same-document and cross-document navigations.

---

## Interview Questions

**Q: Why do `transform` and `opacity` animate at 60fps when `top` and `background-color` don't?**
A: `transform` and `opacity` are compositor-only properties — the browser animates them by updating a pre-painted GPU texture's matrix or alpha value on the compositor thread, independently of the main thread. `top` triggers layout (position recalculation across the subtree), and `background-color` triggers paint (rasterizing new pixels) — both must happen on the main thread, which can be blocked by JavaScript.

**Q: What is a compositor layer and what creates one?**
A: A compositor layer is a separate GPU texture that the compositor thread combines with others to produce the final frame. It's created by `will-change: transform/opacity`, CSS animations on compositable properties, `position: fixed/sticky`, `<video>/<canvas>/<iframe>`, and certain `mix-blend-mode` values. Each layer consumes GPU memory — too many layers cause memory pressure and frame drops.

**Q: What is the trade-off with `will-change: transform`?**
A: It hints to the browser to promote the element to a compositor layer, enabling smooth animation. The cost is GPU memory — the element is pre-rasterized as a texture. Applying it to many elements (like every list item) causes layer explosion: hundreds of textures consume hundreds of MB of GPU memory, eventually causing the GPU to thrash. Apply it only to actively animating elements, and remove it when the animation ends.

**Q: What does `contain: paint` do for performance?**
A: It tells the browser the element's children cannot visually overflow its bounds, creating an isolated paint region. The browser can skip painting the entire subtree if the container is fully off-screen — critical for virtualized list items. It also creates a new stacking context, so `z-index` inside it doesn't affect the rest of the page.
