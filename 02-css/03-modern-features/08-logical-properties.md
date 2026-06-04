# Logical Properties and Values

## Overview

CSS Logical Properties provide flow-relative (logical) equivalents to physical properties. Instead of `left`/`right`/`top`/`bottom`, you use `inline-start`/`inline-end`/`block-start`/`block-end`, which adapt to writing mode and direction.

This enables truly internationalized layouts that work correctly in:
- Left-to-right languages (English, French)
- Right-to-left languages (Arabic, Hebrew)
- Vertical writing modes (Japanese, Mongolian)

## Why Logical Properties Matter

```css
/* PHYSICAL (problematic for internationalization) */
.card {
  margin-left: 1rem;    /* Always left side, even in RTL */
  padding-right: 2rem;  /* Always right side, even in RTL */
  border-bottom: 1px solid;
  text-align: left;
}

/* LOGICAL (adapts to writing mode and direction) */
.card {
  margin-inline-start: 1rem;    /* Start of inline direction */
  padding-inline-end: 2rem;     /* End of inline direction */
  border-block-end: 1px solid;  /* End of block direction */
  text-align: start;
}

/* In RTL mode (dir="rtl"): */
/* margin-inline-start becomes right margin */
/* padding-inline-end becomes left padding */
```

## Block vs Inline Axes

### Horizontal Writing Mode (Default)

```
        block-start (top)
              ↑
inline-start  ←   →  inline-end
  (left)              (right)
              ↓
         block-end (bottom)
```

### Vertical Writing Mode

```
Writing-mode: vertical-rl (Japanese)

block-start ← → block-end
   (right)       (left)
     ↓
 inline-end
  (bottom)
     ↑
inline-start
   (top)
```

## Core Concepts

### Block Axis
- Direction in which content blocks stack
- Horizontal writing: top to bottom (vertical axis)
- Vertical writing: right to left or left to right

### Inline Axis
- Direction in which text flows
- Horizontal writing: left to right or right to left
- Vertical writing: top to bottom

## Property Mappings

### Margin

| Physical | Logical | LTR Equivalent | RTL Equivalent |
|----------|---------|----------------|----------------|
| `margin-top` | `margin-block-start` | top | top |
| `margin-bottom` | `margin-block-end` | bottom | bottom |
| `margin-left` | `margin-inline-start` | left | **right** |
| `margin-right` | `margin-inline-end` | right | **left** |
| `margin: T R B L` | `margin-block: T B` + `margin-inline: L R` | - | - |

```css
/* Physical */
.box {
  margin-top: 1rem;
  margin-right: 2rem;
  margin-bottom: 1rem;
  margin-left: 2rem;
}

/* Logical (shorthand) */
.box {
  margin-block: 1rem;      /* block-start and block-end */
  margin-inline: 2rem;     /* inline-start and inline-end */
}

/* Logical (longhand) */
.box {
  margin-block-start: 1rem;
  margin-block-end: 1rem;
  margin-inline-start: 2rem;
  margin-inline-end: 2rem;
}
```

### Padding

| Physical | Logical |
|----------|---------|
| `padding-top` | `padding-block-start` |
| `padding-bottom` | `padding-block-end` |
| `padding-left` | `padding-inline-start` |
| `padding-right` | `padding-inline-end` |
| - | `padding-block: start end` |
| - | `padding-inline: start end` |

```css
/* Physical */
.card {
  padding: 2rem 3rem; /* top/bottom left/right */
}

/* Logical equivalent */
.card {
  padding-block: 2rem;   /* block-start/end */
  padding-inline: 3rem;  /* inline-start/end */
}
```

### Border

```css
/* Physical */
.element {
  border-left: 2px solid blue;
  border-right-width: 3px;
  border-top-color: red;
  border-bottom-style: dashed;
}

/* Logical */
.element {
  border-inline-start: 2px solid blue;
  border-inline-end-width: 3px;
  border-block-start-color: red;
  border-block-end-style: dashed;
}
```

### Positioning (Inset)

| Physical | Logical |
|----------|---------|
| `top` | `inset-block-start` |
| `bottom` | `inset-block-end` |
| `left` | `inset-inline-start` |
| `right` | `inset-inline-end` |
| - | `inset-block: start end` |
| - | `inset-inline: start end` |
| `top right bottom left` | `inset: block-start inline-end block-end inline-start` |

```css
/* Physical */
.absolute {
  position: absolute;
  top: 10px;
  right: 20px;
  bottom: 10px;
  left: 20px;
}

/* Logical */
.absolute {
  position: absolute;
  inset-block: 10px;    /* start and end */
  inset-inline: 20px;   /* start and end */
}

/* Even shorter with inset shorthand */
.absolute {
  position: absolute;
  inset: 10px 20px; /* block inline */
}
```

### Dimensions

| Physical | Logical |
|----------|---------|
| `width` | `inline-size` |
| `min-width` | `min-inline-size` |
| `max-width` | `max-inline-size` |
| `height` | `block-size` |
| `min-height` | `min-block-size` |
| `max-height` | `max-block-size` |

```css
/* Physical */
.container {
  width: 80%;
  max-width: 1200px;
  min-height: 100vh;
}

/* Logical */
.container {
  inline-size: 80%;
  max-inline-size: 1200px;
  min-block-size: 100vh;
}
```

### Border Radius

```css
/* Physical corners */
.card {
  border-top-left-radius: 8px;
  border-top-right-radius: 8px;
  border-bottom-right-radius: 4px;
  border-bottom-left-radius: 4px;
}

/* Logical corners */
.card {
  border-start-start-radius: 8px;  /* block-start + inline-start */
  border-start-end-radius: 8px;    /* block-start + inline-end */
  border-end-end-radius: 4px;      /* block-end + inline-end */
  border-end-start-radius: 4px;    /* block-end + inline-start */
}
```

### Text Alignment

```css
/* Physical */
.text {
  text-align: left;  /* Problematic in RTL */
}

/* Logical */
.text {
  text-align: start; /* Adapts to direction */
}

/* Values: start, end, center, justify */
```

### Float and Clear

```css
/* Physical */
.sidebar {
  float: left;
}

/* Logical */
.sidebar {
  float: inline-start;
}

/* Values: inline-start, inline-end */
```

## Writing Modes

```css
/* Horizontal (default) */
.horizontal {
  writing-mode: horizontal-tb; /* Top to bottom */
}

/* Vertical right-to-left (Japanese) */
.vertical-rl {
  writing-mode: vertical-rl;
  /* Block axis: right to left */
  /* Inline axis: top to bottom */
}

/* Vertical left-to-right (Mongolian) */
.vertical-lr {
  writing-mode: vertical-lr;
  /* Block axis: left to right */
  /* Inline axis: top to bottom */
}
```

## Direction

```css
/* Left-to-right (default) */
.ltr {
  direction: ltr;
}

/* Right-to-left (Arabic, Hebrew) */
.rtl {
  direction: rtl;
}

/* Usually set on root */
html[dir="rtl"] {
  direction: rtl;
}
```

## Practical Examples

### RTL-Ready Card Component

```css
.card {
  /* Spacing adapts to direction */
  padding-block: 1.5rem;
  padding-inline: 2rem;
  
  /* Border on inline-start (left in LTR, right in RTL) */
  border-inline-start: 4px solid var(--accent);
  
  /* Rounded corners */
  border-start-start-radius: 8px;
  border-end-start-radius: 8px;
}

.card-icon {
  /* Icon on start side */
  margin-inline-end: 1rem;
  float: inline-start;
}

.card-title {
  text-align: start;
}

.card-actions {
  /* Buttons align to end */
  text-align: end;
  margin-block-start: 1rem;
}
```

### Responsive Sidebar

```css
.layout {
  display: grid;
  grid-template-columns: 250px 1fr;
  gap: 2rem;
}

.sidebar {
  /* Sticky to block-start */
  position: sticky;
  inset-block-start: 2rem;
  
  /* Border on inline-end (right in LTR, left in RTL) */
  border-inline-end: 1px solid #ddd;
  padding-inline-end: 1rem;
}

.sidebar-nav a {
  display: block;
  padding-block: 0.5rem;
  padding-inline-start: 1rem;
  text-align: start;
}

.sidebar-nav a.active {
  border-inline-start: 3px solid var(--primary);
}
```

### Form Layouts

```css
.form-field {
  margin-block-end: 1.5rem;
}

.form-label {
  display: block;
  margin-block-end: 0.5rem;
  text-align: start;
}

.form-input {
  inline-size: 100%;
  padding-block: 0.75rem;
  padding-inline: 1rem;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.form-error {
  /* Error message aligns to start */
  margin-block-start: 0.25rem;
  margin-inline-start: 0.5rem;
  text-align: start;
  color: var(--error);
}

.form-actions {
  /* Buttons align to end */
  display: flex;
  justify-content: flex-end;
  gap: 1rem;
  margin-block-start: 2rem;
}
```

### Navigation Bar

```css
.navbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-block: 1rem;
  padding-inline: 2rem;
  border-block-end: 1px solid #eee;
}

.navbar-logo {
  /* Logo on start side */
  margin-inline-end: 2rem;
}

.navbar-menu {
  display: flex;
  gap: 1.5rem;
  list-style: none;
  margin: 0;
  padding: 0;
}

.navbar-item {
  padding-inline: 0.5rem;
}

.navbar-actions {
  /* Actions on end side */
  margin-inline-start: auto;
}
```

## Complete Comparison Table

### Spacing

| Physical | Logical | Notes |
|----------|---------|-------|
| `margin-left` | `margin-inline-start` | Start of text flow |
| `margin-right` | `margin-inline-end` | End of text flow |
| `margin-top` | `margin-block-start` | Start of content stack |
| `margin-bottom` | `margin-block-end` | End of content stack |
| `padding-*` | Same pattern | Same as margin |

### Positioning

| Physical | Logical | Notes |
|----------|---------|-------|
| `left` | `inset-inline-start` | Inline axis start |
| `right` | `inset-inline-end` | Inline axis end |
| `top` | `inset-block-start` | Block axis start |
| `bottom` | `inset-block-end` | Block axis end |

### Dimensions

| Physical | Logical | Notes |
|----------|---------|-------|
| `width` | `inline-size` | Text flow dimension |
| `height` | `block-size` | Stack dimension |
| `min-width` | `min-inline-size` | - |
| `max-width` | `max-inline-size` | - |
| `min-height` | `min-block-size` | - |
| `max-height` | `max-block-size` | - |

### Borders

| Physical | Logical | Notes |
|----------|---------|-------|
| `border-left` | `border-inline-start` | - |
| `border-right` | `border-inline-end` | - |
| `border-top` | `border-block-start` | - |
| `border-bottom` | `border-block-end` | - |
| `border-top-left-radius` | `border-start-start-radius` | block-start + inline-start |
| `border-top-right-radius` | `border-start-end-radius` | block-start + inline-end |
| `border-bottom-right-radius` | `border-end-end-radius` | block-end + inline-end |
| `border-bottom-left-radius` | `border-end-start-radius` | block-end + inline-start |

## Migration Strategy

### Gradual Adoption

```css
/* Start with layout-critical properties */
.component {
  /* High-impact: spacing, positioning */
  margin-inline-start: 2rem;  /* Instead of margin-left */
  padding-inline: 1rem;       /* Instead of padding-left/right */
  inset-inline-start: 0;      /* Instead of left: 0 */
  
  /* Lower priority: decorative */
  border-left: 1px solid; /* Can migrate later */
}
```

### When NOT to Use Logical Properties

```css
/* Visual/decorative effects that shouldn't flip */
.shadow-right {
  box-shadow: 10px 5px 5px rgba(0,0,0,0.3);
  /* Intentionally asymmetric visual effect */
}

/* Coordinate systems (Canvas, SVG) */
.svg-element {
  x: 100; /* SVG coordinate, not layout property */
}

/* Physical media queries */
@media (min-width: 768px) { /* Viewport width is physical */ }
```

## Browser Support

```css
/* Feature detection */
@supports (margin-inline-start: 0) {
  .element {
    margin-inline-start: 1rem;
  }
}

@supports not (margin-inline-start: 0) {
  /* Fallback for older browsers */
  .element {
    margin-left: 1rem;
  }
  
  [dir="rtl"] .element {
    margin-left: 0;
    margin-right: 1rem;
  }
}
```

## Common Gotchas

### 1. Mixing Physical and Logical

```css
/* INCONSISTENT: mixing paradigms */
.bad {
  margin-left: 1rem;           /* Physical */
  padding-inline-start: 2rem;  /* Logical */
  /* Confusing and error-prone */
}

/* GOOD: consistent approach */
.good {
  margin-inline-start: 1rem;
  padding-inline-start: 2rem;
}
```

### 2. Shorthand Reset

```css
/* Physical shorthand overrides logical */
.element {
  margin-inline-start: 2rem;
  margin: 1rem; /* Resets margin-inline-start! */
}

/* Be careful with order */
.element {
  margin: 1rem;               /* Set all sides */
  margin-inline-start: 2rem;  /* Then override specific side */
}
```

### 3. Transform Origin

```css
/* transform-origin is physical by convention */
.rotated {
  transform-origin: left center; /* Stays physical */
  /* There's no transform-origin: inline-start center */
}
```

### 4. Grid and Flexbox

```css
/* Grid template areas are logical by default */
.grid {
  grid-template-areas:
    "header header"
    "sidebar content"; /* Already direction-aware */
}

/* But grid-column-start is physical */
.item {
  grid-column-start: 1; /* Column 1 is always column 1 */
}
```

## Performance

Logical properties have **no performance penalty**. They're resolved at style computation time, not runtime. Modern browsers handle them as efficiently as physical properties.

## Interview Questions

**Q1: What problem do logical properties solve?**

**A:** Logical properties make layouts automatically adapt to different writing modes and text directions without JavaScript or duplicate CSS. A single set of styles works correctly for LTR languages (English), RTL languages (Arabic, Hebrew), and vertical writing modes (Japanese), eliminating the need for direction-specific overrides.

**Q2: Explain the difference between block and inline axes.**

**A:** The inline axis is the direction text flows (left-to-right in English, top-to-bottom in vertical Japanese). The block axis is the direction content blocks stack (top-to-bottom in English, right-to-left in vertical Japanese). Logical properties reference these flow-relative axes instead of physical screen coordinates.

**Q3: When would you NOT use logical properties?**

**A:** Don't use logical properties for:
- Intentionally asymmetric visual effects (shadows, gradients)
- Coordinate systems (SVG, Canvas)
- Physical media queries (viewport dimensions)
- Transform origins
- Properties that should remain fixed regardless of direction

**Q4: How does `inset` shorthand work?**

**A:** `inset` is a shorthand for positioning:
- `inset: 10px` → all sides 10px
- `inset: 10px 20px` → block: 10px, inline: 20px
- `inset: 10px 20px 30px 40px` → block-start, inline-end, block-end, inline-start

It follows logical flow, not physical TRBL (top-right-bottom-left).

**Q5: What happens when you mix physical and logical properties?**

**A:** They coexist but can conflict. Physical shorthands (like `margin: 1rem`) will override previously set logical properties (like `margin-inline-start: 2rem`). For maintainability, pick one paradigm and stick with it. Logical is preferred for new code.

**Q6: How do logical properties interact with Flexbox?**

**A:** Flexbox is already flow-relative. `justify-content` and `align-items` work along flex container's axes, which respect `flex-direction` and writing mode. Logical properties complement this for margins, padding, and borders on flex items.

## Resources

- [MDN: CSS Logical Properties](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Logical_Properties)
- [CSS Logical Properties Spec](https://www.w3.org/TR/css-logical-1/)
- [Ahmad Shadeed: Logical Properties Guide](https://ishadeed.com/article/css-logical-properties/)
- [Rachel Andrew: Logical Properties](https://rachelandrew.co.uk/archives/2021/04/13/getting-started-with-css-logical-properties/)
- [Can I Use: CSS Logical Properties](https://caniuse.com/css-logical-props)

## Summary

CSS Logical Properties enable truly international layouts by using flow-relative values instead of physical directions. Use `inline-start/end` for left/right, `block-start/end` for top/bottom, and `inset` for positioning. They adapt automatically to writing mode and text direction, making your CSS work correctly across languages without duplication. Adopt them progressively, starting with spacing and positioning, while avoiding them for intentional visual effects. The result is more maintainable, accessible, and globally-ready stylesheets.
