# Centering an Element — 5 Ways

## 1. Flexbox (Most Versatile)

```css
.container {
  display: flex;
  justify-content: center;  /* horizontal */
  align-items: center;       /* vertical */
}
```

**Use when:** Container wraps the element you want to center. Works for both horizontal and vertical in one rule. Supports multiple items.

**Gotcha:** Container needs a height for vertical centering to have effect.

---

## 2. Grid with `place-items`

```css
.container {
  display: grid;
  place-items: center; /* shorthand for align-items + justify-items */
}

/* or more explicit: */
.container {
  display: grid;
  align-items: center;
  justify-items: center;
}
```

**Use when:** Same as flexbox, but even more concise. `place-items: center` is the shortest possible centering.

```css
/* Full-page centering */
body {
  display: grid;
  place-items: center;
  min-height: 100vh;
  margin: 0;
}
```

---

## 3. Absolute Positioning + `transform` (Classic)

```css
.container {
  position: relative;
}

.centered {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}
```

**How it works:**
- `top: 50%` / `left: 50%` moves the element's top-left corner to center
- `translate(-50%, -50%)` shifts it back by half its own width/height

**Use when:** The element must be absolutely positioned (modals, tooltips, overlays). Does NOT require knowing element dimensions.

**Gotcha:** Element is removed from flow. Parent must have `position: relative` (or non-static).

---

## 4. Absolute Positioning + `margin: auto` (Requires Dimensions)

```css
.container {
  position: relative;
}

.centered {
  position: absolute;
  inset: 0; /* top: 0; right: 0; bottom: 0; left: 0 */
  margin: auto;
  width: 300px;   /* MUST have explicit width */
  height: 200px;  /* MUST have explicit height */
}
```

**How it works:** `inset: 0` stretches the element to all edges. `margin: auto` distributes remaining space equally on all sides.

**Use when:** Element has known fixed dimensions. Slightly more readable than the transform approach for fixed-size elements.

---

## 5. `margin: auto` in Block Context (Horizontal Only)

```css
.centered {
  width: 600px;      /* MUST have explicit width */
  margin: 0 auto;    /* auto on left and right */
}
```

**Use when:** Centering a block element horizontally within its parent (content container, article body). The classic layout centering technique.

**Gotcha:** Only horizontal. Width must be set or be shorter than the parent. Doesn't work for `inline` or `inline-block` elements (use `text-align: center` on parent instead).

---

## Bonus: Logical Properties Version

```css
/* Flexbox centering with logical properties */
.container {
  display: flex;
  justify-content: center;
  align-items: center;
}

/* Margin auto centering with logical properties */
.centered {
  max-inline-size: 600px;
  margin-inline: auto;
}
```

---

## Quick Decision Guide

| Scenario | Method |
|---|---|
| Center anything, container has height | Flexbox or Grid |
| Center one item, full viewport | `display: grid; place-items: center` |
| Overlay/modal with absolute positioning | Absolute + transform |
| Fixed-size element, absolute positioned | Absolute + margin: auto |
| Content container, horizontal only | `margin: 0 auto` with width |

---

## Common Interview Questions

**Q: Why does `margin: auto` not center vertically?**
In block flow, `auto` top/bottom margins collapse to 0. The spec defines vertical margins as not participating in the distribution of remaining space (only horizontal). Exception: absolutely positioned elements with `inset: 0` — then auto margins DO work vertically.

**Q: What's the difference between `align-items` and `align-content` in flexbox?**
`align-items` aligns items within a single line (cross-axis alignment of each row). `align-content` distributes multiple lines when flex wrap creates multiple rows. `align-items` is what you need for single-line centering.
