# CSS Units: px, em, rem, %, vw/vh, dvh/svh/lvh, ch, ex, fr

## The Idea

**In plain English:** CSS units are the measurement system you use to tell a browser how big or small things should be — just like how you might measure a room in feet, a recipe in cups, or a person's height in centimetres, each unit serves a different purpose.

**Real-world analogy:** Imagine you are decorating a room and you tell the furniture store "I want a sofa that takes up half the wall." Whether the wall is 3 metres or 6 metres wide, the sofa always fills half of it. But if you said "I want a sofa exactly 2 metres wide," it is fixed no matter what. CSS units work the same way — some are fixed measurements and some are relative to something else.
- The fixed "2 metres wide" sofa = `px` (always the same size, no matter the screen)
- The "half the wall" sofa = `%` or `fr` (scales relative to the surrounding space)
- The "same height as the door frame" shelf = `em` or `rem` (scales relative to a font size, like a reference point in the room)

---

## Learning Objectives
- Master absolute and relative unit types and when to use each
- Understand viewport units and their mobile viewport variants
- Use font-relative units (em, rem, ch, ex) effectively
- Apply the `fr` unit in Grid layouts
- Build responsive, accessible interfaces with appropriate unit choices

---

## Unit Categories

CSS units fall into two main categories:

### Absolute Units
Fixed size, regardless of context.

### Relative Units
Size computed relative to another value (parent font size, viewport, etc.).

---

## Absolute Units

### `px` (Pixels)

**Definition:** One device pixel on standard displays; scaled on high-DPI screens.

```css
.box {
  width: 300px;
  height: 200px;
  border: 1px solid black;
}
```

**Characteristics:**
- Precise, predictable
- Not scalable with user font preferences
- Fine for borders (1px, 2px), but problematic for text and spacing

**When to use:**
- Borders (1px, 2px)
- Shadows (1px, 2px offsets)
- Icon sizes (when not scalable)

**When NOT to use:**
- Font sizes (users can't scale)
- Spacing (not responsive to user preferences)
- Layout dimensions (not responsive)

**Accessibility issue:** Fixed pixel sizes ignore user font size preferences (e.g., users who need larger text).

---

### Other Absolute Units (Rarely Used)

| Unit | Name | Size | Use Case |
|------|------|------|----------|
| `cm` | Centimeters | 1cm = 96px/2.54 | Print stylesheets |
| `mm` | Millimeters | 1mm = 1/10 cm | Print stylesheets |
| `in` | Inches | 1in = 96px | Print stylesheets |
| `pt` | Points | 1pt = 1/72 in | Print stylesheets |
| `pc` | Picas | 1pc = 12pt | Print stylesheets |

**Note:** These are **physical units** for print media, not screens.

---

## Relative Units: Font-Relative

### `em`

**Definition:** Relative to the **font-size** of the **element itself** (or parent if used on `font-size`).

#### For font-size:
```css
.parent {
  font-size: 16px;
}

.child {
  font-size: 2em;  /* 2 × 16px = 32px */
}
```

#### For other properties:
```css
.element {
  font-size: 20px;
  padding: 1em;     /* 1 × 20px = 20px */
  margin: 0.5em;    /* 0.5 × 20px = 10px */
}
```

**Key insight:** `em` compounds when nested.

```css
.parent {
  font-size: 16px;
}

.child {
  font-size: 1.5em;  /* 24px */
}

.grandchild {
  font-size: 1.5em;  /* 1.5 × 24px = 36px (compounding!) */
}
```

**Use cases:**
- Spacing relative to element's font size (padding, margin)
- Icon sizing (1em matches text height)
- Button padding (scales with button text)

**Pitfalls:**
- Compounding makes nested calculations hard to predict
- Prefer `rem` for font sizes to avoid compounding

---

### `rem` (Root EM)

**Definition:** Relative to the **root element's** (`<html>`) font-size.

```css
html {
  font-size: 16px;  /* Default browser base */
}

.element {
  font-size: 1.5rem;  /* 1.5 × 16px = 24px */
  margin: 2rem;       /* 2 × 16px = 32px */
}

.nested {
  font-size: 1.5rem;  /* Still 24px (no compounding) */
}
```

**Benefits:**
- Consistent sizing across the entire document
- No compounding issues
- Respects user's browser font size preferences

**Best practice for accessibility:**
```css
html {
  font-size: 100%;  /* Respects user preference (usually 16px) */
}

body {
  font-size: 1rem;  /* 16px if user hasn't changed settings */
}

h1 {
  font-size: 2.5rem;  /* 40px */
}

h2 {
  font-size: 2rem;    /* 32px */
}
```

**Comparison:**

```css
/* Bad - px doesn't scale with user preferences */
h1 { font-size: 32px; }

/* Good - rem scales with user preferences */
h1 { font-size: 2rem; }
```

**When to use `rem` vs `em`:**
- **Font sizes:** Use `rem` (avoids compounding)
- **Spacing inside components:** Use `em` (scales with component text)
- **Global spacing:** Use `rem` (consistent across site)

---

### `ch` (Character Width)

**Definition:** Width of the `0` (zero) character in the element's font.

```css
.input {
  width: 20ch;  /* Wide enough for ~20 characters */
}

.code {
  max-width: 80ch;  /* Classic 80-column width */
}
```

**Use cases:**
- Input field widths (e.g., postal code: `width: 6ch`)
- Optimal line length for readability (45-75ch)
- Monospace text layouts

**Real-world example:**

```css
.article-content {
  max-width: 65ch;  /* Optimal reading width */
  margin: 0 auto;
}
```

**Note:** In proportional fonts, `ch` is an **approximation**. Works best with monospace fonts.

---

### `ex` (x-Height)

**Definition:** Height of the lowercase `x` in the element's font.

```css
.superscript {
  vertical-align: 1ex;  /* Raise by x-height */
}
```

**Use cases:**
- Vertical alignment relative to text baseline
- Icon alignment with text
- Rarely used in modern CSS

---

### `cap` (Cap Height) - Modern

**Definition:** Height of capital letters in the element's font.

```css
.icon {
  height: 1cap;  /* Match capital letter height */
}
```

**Browser support:** Limited (2023+).

---

### `lh` (Line Height) - Modern

**Definition:** Computed line-height of the element.

```css
.element {
  line-height: 1.5;
  margin-bottom: 1lh;  /* One line of spacing */
}
```

**Browser support:** Limited (2023+).

---

## Relative Units: Viewport-Based

### `vw` (Viewport Width)

**Definition:** 1% of the viewport width.

```css
.full-width {
  width: 100vw;  /* Full viewport width */
}

.half-screen {
  width: 50vw;   /* Half viewport width */
}
```

**Use case:** Full-width sections that break out of containers.

**Gotcha:** Includes scrollbar width, may cause horizontal overflow.

```css
/* May cause horizontal scrollbar */
.full-width {
  width: 100vw;  /* Viewport width + scrollbar */
}

/* Better */
.full-width {
  width: 100%;   /* Parent's width */
}
```

---

### `vh` (Viewport Height)

**Definition:** 1% of the viewport height.

```css
.hero {
  height: 100vh;  /* Full viewport height */
}

.section {
  min-height: 50vh;  /* At least half viewport */
}
```

**Mobile Safari gotcha:** Address bar affects `vh` on scroll.

```
Initial load:    100vh includes address bar
After scroll:    100vh excludes address bar (address bar hides)
Result:          Content jumps
```

**Solution:** Use new dynamic viewport units (see below).

---

### `vmin` and `vmax`

**`vmin`:** 1% of the **smaller** viewport dimension.

**`vmax`:** 1% of the **larger** viewport dimension.

```css
/* On a 1920×1080 viewport */
1vmin = 1% of 1080px = 10.8px  (height is smaller)
1vmax = 1% of 1920px = 19.2px  (width is larger)

/* On a 768×1024 viewport (portrait phone) */
1vmin = 1% of 768px = 7.68px   (width is smaller)
1vmax = 1% of 1024px = 10.24px (height is larger)
```

**Use case:** Responsive sizing that adapts to orientation.

```css
.logo {
  width: 20vmin;  /* Scales based on smallest dimension */
}
```

---

### New Viewport Units: `dvh`, `svh`, `lvh` (Dynamic, Small, Large)

**Problem with `vh` on mobile:**
- Browser UI (address bar, toolbar) shows/hides on scroll
- `100vh` value changes, causing layout shift

**Solution:** Three viewport variants.

#### `svh` (Small Viewport Height)
**Assumes UI is visible** (smallest possible viewport).

```css
.hero {
  height: 100svh;  /* Height when browser UI is shown */
}
```

**Use case:** Ensure content is always visible, even with browser UI.

---

#### `lvh` (Large Viewport Height)
**Assumes UI is hidden** (largest possible viewport).

```css
.hero {
  height: 100lvh;  /* Height when browser UI is hidden */
}
```

**Use case:** Maximize space when browser UI hides.

---

#### `dvh` (Dynamic Viewport Height)
**Changes as UI shows/hides** (tracks current viewport).

```css
.hero {
  height: 100dvh;  /* Adjusts as browser UI appears/disappears */
}
```

**Use case:** Most accurate to actual viewport, but may cause reflows.

---

**Recommendation:**

```css
/* Best for most cases - ensures content is always visible */
.section {
  min-height: 100svh;
}

/* Alternative - if you want dynamic behavior */
.section {
  min-height: 100dvh;
}

/* Fallback for older browsers */
.section {
  min-height: 100vh;     /* Fallback */
  min-height: 100dvh;    /* Override if supported */
}
```

**Equivalent units for width:**
- `dvw`, `svw`, `lvw` (dynamic, small, large viewport width)
- `dvmin`, `svmin`, `lvmin`
- `dvmax`, `svmax`, `lvmax`

**Browser support:** Chrome 108+, Safari 15.4+, Firefox 101+.

---

## Percentage `%`

**Definition:** Relative to the **parent element's** corresponding property.

### Width and Height
```css
.parent {
  width: 800px;
  height: 600px;
}

.child {
  width: 50%;   /* 50% of 800px = 400px */
  height: 25%;  /* 25% of 600px = 150px (only if parent has explicit height) */
}
```

**Height gotcha:** Percentage height only works if parent has explicit height.

```css
/* Doesn't work */
.parent {
  /* height: auto (default) */
}

.child {
  height: 100%;  /* No effect */
}

/* Works */
.parent {
  height: 500px;  /* or 100vh, or min-height */
}

.child {
  height: 100%;  /* Now works */
}
```

---

### Margin and Padding
**Percentage relative to parent's WIDTH** (even for top/bottom).

```css
.parent {
  width: 400px;
}

.child {
  margin-top: 10%;     /* 10% of 400px = 40px */
  padding-bottom: 5%;  /* 5% of 400px = 20px */
}
```

**Use case:** Maintaining aspect ratios (pre-`aspect-ratio`).

```css
.video-container {
  width: 100%;
  padding-bottom: 56.25%;  /* 16:9 ratio */
  position: relative;
}

.video-container iframe {
  position: absolute;
  inset: 0;
}
```

---

### Font-size
**Percentage relative to parent's font-size.**

```css
.parent {
  font-size: 20px;
}

.child {
  font-size: 150%;  /* 1.5 × 20px = 30px */
}
```

**Equivalent to `em` for font-size:**
```css
font-size: 150%;  /* Same as 1.5em */
```

---

## Grid-Specific Unit: `fr` (Fraction)

**Definition:** Fraction of **available space** in a grid container.

```css
.grid {
  display: grid;
  grid-template-columns: 1fr 2fr 1fr;
  /* Columns split as 25%, 50%, 25% of available space */
}
```

**How it works:**
1. Grid calculates fixed sizes (px, auto)
2. Remaining space is divided by `fr` units

**Example:**

```css
.grid {
  display: grid;
  width: 1000px;
  grid-template-columns: 200px 1fr 2fr;
}
```

**Calculation:**
1. Fixed: 200px
2. Remaining: 1000px - 200px = 800px
3. Total `fr` units: 1 + 2 = 3
4. 1fr = 800px / 3 = 266.67px
5. 2fr = 533.33px

**Result:** Columns are 200px, 266.67px, 533.33px.

---

**`fr` vs `%`:**

```css
/* Percentage - includes gaps */
.grid-percent {
  display: grid;
  grid-template-columns: 50% 50%;
  gap: 20px;
  /* Causes overflow (100% + 20px gap) */
}

/* fr - excludes gaps */
.grid-fr {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 20px;
  /* Works perfectly (gap is subtracted before calculating fr) */
}
```

**Best practice:** Use `fr` for flexible grid columns, not `%`.

---

## Unit Conversion Cheat Sheet

| From | To | Formula |
|------|-----|---------|
| px → rem | rem = px / root font-size | 32px ÷ 16px = 2rem |
| rem → px | px = rem × root font-size | 2rem × 16px = 32px |
| em → px | px = em × element font-size | 1.5em × 20px = 30px |
| px → % | % = (px / parent px) × 100 | 200px / 800px = 25% |

---

## Choosing the Right Unit: Decision Tree

```
What are you sizing?

├─ Font size?
│   ├─ Use `rem` (respects user preferences, no compounding)
│
├─ Spacing (margin, padding)?
│   ├─ Component-internal? → `em` (scales with component)
│   └─ Global/consistent? → `rem`
│
├─ Layout width?
│   ├─ Full width? → `%` or `fr` (in grid)
│   ├─ Fixed? → `px` (rarely needed)
│   └─ Responsive? → `vw`, `%`, or `fr`
│
├─ Layout height?
│   ├─ Full viewport? → `100dvh` (or `svh`)
│   ├─ Percentage? → `%` (only if parent has explicit height)
│   └─ Dynamic? → `auto` or `min-height: 100dvh`
│
├─ Borders, shadows?
│   └─ Use `px` (fine details)
│
├─ Line length (readability)?
│   └─ Use `ch` (e.g., `max-width: 65ch`)
│
└─ Grid columns?
    └─ Use `fr` (flexible, gap-aware)
```

---

## Real-World Examples

### Example 1: Responsive Typography

```css
html {
  font-size: 100%;  /* Respects user preference */
}

body {
  font-size: 1rem;    /* 16px default */
  line-height: 1.6;
}

h1 { font-size: 2.5rem; }   /* 40px */
h2 { font-size: 2rem; }     /* 32px */
h3 { font-size: 1.75rem; }  /* 28px */
h4 { font-size: 1.5rem; }   /* 24px */
h5 { font-size: 1.25rem; }  /* 20px */
h6 { font-size: 1rem; }     /* 16px */

p { font-size: 1rem; }
small { font-size: 0.875rem; } /* 14px */

/* Fluid typography (modern) */
h1 {
  font-size: clamp(2rem, 5vw + 1rem, 3.5rem);
  /* Min 2rem, scales with viewport, max 3.5rem */
}
```

---

### Example 2: Card Component

```css
.card {
  /* Layout */
  width: 100%;
  max-width: 400px;
  
  /* Spacing relative to card's font size */
  padding: 1.5em;
  
  /* Typography */
  font-size: 1rem;  /* Base size */
}

.card__title {
  font-size: 1.5em;      /* 1.5 × card font-size */
  margin-bottom: 0.5em;  /* 0.5 × title font-size */
}

.card__body {
  font-size: 1em;
  margin-bottom: 1em;
}

.card__footer {
  font-size: 0.875em;  /* Smaller footer text */
  color: #666;
}
```

**Why `em` for spacing?** If card's font-size changes (e.g., in a larger variant), all spacing scales proportionally.

```css
.card--large {
  font-size: 1.25rem;  /* All em-based spacing scales up */
}
```

---

### Example 3: Full-Screen Hero

```css
.hero {
  /* Modern approach - dynamic viewport */
  min-height: 100dvh;
  
  /* Fallback for older browsers */
  min-height: 100vh;
  min-height: 100dvh;
  
  /* Or use small viewport (ensures content visible) */
  min-height: 100svh;
  
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 2rem;
}
```

---

### Example 4: Responsive Grid

```css
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 2rem;
  padding: 2rem;
}
```

**Breakdown:**
- `minmax(250px, 1fr)` - Columns are at least 250px, grow to fill space
- `auto-fit` - Fits as many columns as possible
- `gap: 2rem` - Consistent spacing
- `fr` - Distributes remaining space equally

---

### Example 5: Optimal Reading Width

```css
.article {
  max-width: 65ch;  /* ~65 characters per line (optimal readability) */
  margin: 0 auto;
  padding: 2rem;
  font-size: 1.125rem;  /* Slightly larger for readability */
  line-height: 1.7;
}
```

---

## Common Mistakes

### ❌ Mistake 1: Using px for font sizes

```css
/* Bad - ignores user preferences */
body { font-size: 16px; }

/* Good - respects user preferences */
body { font-size: 1rem; }
```

---

### ❌ Mistake 2: Using vh for mobile layouts

```css
/* Bad - causes jumps on mobile when address bar shows/hides */
.section { height: 100vh; }

/* Good - accounts for mobile UI */
.section { min-height: 100dvh; }

/* Better - ensures content visible */
.section { min-height: 100svh; }
```

---

### ❌ Mistake 3: Percentage height without parent height

```css
/* Bad - no effect */
.parent { /* height: auto */ }
.child { height: 50%; }  /* Doesn't work */

/* Good */
.parent { height: 500px; }
.child { height: 50%; }  /* 250px */

/* Better - use viewport units */
.child { height: 50vh; }
```

---

### ❌ Mistake 4: Using % for grid columns

```css
/* Bad - doesn't account for gap */
.grid {
  display: grid;
  grid-template-columns: 50% 50%;
  gap: 20px;  /* Causes horizontal overflow */
}

/* Good - fr accounts for gap */
.grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 20px;  /* Works perfectly */
}
```

---

### ❌ Mistake 5: Compounding em units

```css
/* Bad - compounding */
.parent { font-size: 1.5em; }    /* 24px if base is 16px */
.child { font-size: 1.5em; }     /* 36px (compounds) */
.grandchild { font-size: 1.5em; } /* 54px (yikes!) */

/* Good - use rem */
.parent { font-size: 1.5rem; }     /* 24px */
.child { font-size: 1.5rem; }      /* 24px */
.grandchild { font-size: 1.5rem; } /* 24px */
```

---

## Accessibility Best Practices

✅ **Use `rem` for font sizes** - respects user preferences

✅ **Set root font-size to 100%** - don't override user's base size

```css
/* Good */
html { font-size: 100%; }

/* Bad */
html { font-size: 10px; }  /* Breaks user preferences */
```

✅ **Use relative units for spacing** - scales with text

✅ **Avoid fixed pixel heights** - can clip zoomed content

✅ **Test with 200% zoom** - ensure layout doesn't break

---

## Modern Unit Combinations

### Fluid Typography with `clamp()`

```css
h1 {
  font-size: clamp(2rem, 5vw + 1rem, 4rem);
  /* Min: 2rem, Preferred: 5vw + 1rem, Max: 4rem */
}
```

**Benefit:** Responsive without media queries.

---

### Container Query Units (Modern)

```css
.card {
  container-type: inline-size;
}

.card__title {
  font-size: clamp(1rem, 5cqi, 2rem);
  /* 5cqi = 5% of container's inline size */
}
```

**Container units:**
- `cqw` - 1% of container width
- `cqh` - 1% of container height
- `cqi` - 1% of container inline size
- `cqb` - 1% of container block size
- `cqmin` / `cqmax` - smaller/larger dimension

---

## Key Takeaways

✅ **Use `rem` for font sizes** - accessible, predictable

✅ **Use `em` for component-internal spacing** - scales with component

✅ **Use `dvh` or `svh` instead of `vh` on mobile** - accounts for browser UI

✅ **Use `fr` in Grid, not `%`** - gap-aware

✅ **Use `ch` for optimal line length** - ~65ch for readability

✅ **Use `px` only for fine details** - borders, shadows

✅ **Avoid fixed pixel sizes for text** - breaks accessibility

✅ **Set `html { font-size: 100%; }`** - respect user preferences

✅ **Test with browser zoom** - ensure layout scales properly

---

## Resources

- [MDN: CSS Values and Units](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Values_and_Units)
- [MDN: Large, Small, and Dynamic Viewport Units](https://developer.mozilla.org/en-US/docs/Web/CSS/length#relative_length_units_based_on_viewport)
- [A Complete Guide to CSS Units](https://css-tricks.com/the-lengths-of-css/)
- [Modern Fluid Typography](https://www.smashingmagazine.com/2022/01/modern-fluid-typography-css-clamp/)
- [PX, EM or REM Media Queries?](https://zellwk.com/blog/media-query-units/)
