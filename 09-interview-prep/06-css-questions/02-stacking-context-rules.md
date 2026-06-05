# Stacking Context Creation Rules

## The Idea

**In plain English:** A stacking context is like a self-contained stack of papers on a desk — elements inside it are layered among themselves, and the entire stack is then placed as one unit on the bigger desk alongside other stacks. The `z-index` property (a number that controls which element appears in front) only competes with other elements inside the same stack, not across different stacks.

**Real-world analogy:** Imagine a theatre stage with multiple shadow boxes (framed display cases) mounted on the back wall, each box containing small figurines arranged at different depths inside it. Even if a figurine in one box is pushed to the very front of its box, it can never appear in front of a figurine sitting in a box mounted higher up on the wall.

- The theatre wall = the browser page
- Each shadow box = a stacking context
- The depth of a figurine inside its box = the `z-index` of an element within its stacking context
- The height of the box on the wall = the `z-index` of the stacking context itself among other contexts

---

## What Is a Stacking Context?

A stacking context is an isolated layer in the page's painting order. Elements inside a stacking context are painted and stacked relative to each other, and the whole context is treated as a single unit when compositing with the rest of the page.

**Key insight:** `z-index` only works within the same stacking context. A `z-index: 9999` inside a stacking context with `z-index: 1` can never appear above an element outside that context with `z-index: 2`.

---

## What Creates a Stacking Context?

```css
/* The root element always creates one */

/* Position + z-index (the classic one) */
.element {
  position: relative | absolute | fixed | sticky;
  z-index: /* any value except auto */;
}

/* Opacity less than 1 */
.element { opacity: 0.99; } /* not just 0! */

/* Any transform */
.element { transform: translateX(0); } /* even identity transform */

/* Any filter */
.element { filter: blur(0); }

/* will-change with certain values */
.element { will-change: transform | opacity | filter; }

/* Flex/Grid item with z-index (not auto) */
.flex-child { z-index: 1; } /* if parent is flex/grid */

/* isolation */
.element { isolation: isolate; } /* explicit, clean way to create context */

/* mix-blend-mode (not normal) */
.element { mix-blend-mode: multiply; }

/* -webkit-overflow-scrolling: touch (legacy iOS) */
```

---

## Painting Order Within a Stacking Context

From bottom to top:
1. Background and borders of the stacking context element
2. Child stacking contexts with negative `z-index`
3. Block-level non-positioned elements
4. Floating elements
5. Inline-level elements
6. Child stacking contexts with `z-index: 0` or `auto`
7. Child stacking contexts with positive `z-index`

---

## The Classic Bug

```html
<div class="modal-backdrop" style="z-index: 100;">
  <div class="modal" style="z-index: 200;">
    Modal content
  </div>
</div>

<div class="tooltip-container" style="transform: translateX(0);">
  <!-- transform creates a stacking context! -->
  <div class="tooltip" style="z-index: 999;">
    This tooltip appears BEHIND the backdrop despite z-index: 999
    because it's inside a different stacking context
  </div>
</div>
```

**Fix:** Move the tooltip outside the transformed container, or use `portal` (React/Angular) to render it at the document body level.

---

## `isolation: isolate`

The clean, explicit way to create a stacking context without side effects:

```css
/* Create a stacking context just for isolation purposes */
.component {
  isolation: isolate;
}
/* This doesn't change positioning, opacity, or any visual effect
   It just creates a new stacking context for z-index containment */
```

Useful for component encapsulation — ensure a component's internal z-index wars don't affect the rest of the page.

---

## Debugging Stacking Issues

Chrome DevTools → Layers panel shows which elements have their own composited layers. For stacking context issues:

1. Add `outline: 1px solid red` to suspect elements to confirm boundaries
2. Check for `transform`, `opacity < 1`, `filter` on ancestors (creates context unexpectedly)
3. Check flex/grid parent's children — `z-index` on a flex item creates a context
4. Use `isolation: isolate` to explicitly scope z-index

---

## Common Interview Questions

**Q: Why does `opacity: 0.99` create a stacking context?**
Compositing semi-transparent elements requires knowing their complete layer, so the browser creates an isolated layer (stacking context) for the element. `opacity: 1` is fully opaque and needs no special compositing.

**Q: Does `position: fixed` always create a stacking context?**
Yes. All non-static positioned elements with a z-index other than `auto` create a stacking context. `position: fixed` always creates one because it positions relative to the viewport (which requires isolation).

**Q: How does `will-change` create a stacking context?**
`will-change: transform` tells the browser to pre-promote the element to its own compositor layer in preparation for an animation. Pre-promoting requires creating a stacking context.
