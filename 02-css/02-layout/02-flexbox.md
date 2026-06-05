# Flexbox: Main/Cross Axis, flex-grow/shrink/basis, Alignment

## The Idea

**In plain English:** Flexbox is a CSS tool that lets you arrange items inside a box — lining them up in a row or a column, and controlling how they spread out, shrink, or stretch to fill the available space. "CSS" means the styling language that controls how web pages look.

**Real-world analogy:** Imagine a coat rack with a fixed rail (the container) where you hang several coat hooks (the items). The rail runs left to right, and you decide how the hooks are spaced — all bunched at the left, spread evenly, or centered. If you run out of rail space, the hooks either squish closer together or spill onto a second rail below.
- The coat rack rail = the flex container (the parent element with `display: flex`)
- The coat hooks = the flex items (the child elements inside the container)
- The left-to-right direction of the rail = the main axis (the primary direction items flow)
- The top-to-bottom direction (across rails) = the cross axis (perpendicular to the main axis)

---

## Learning Objectives
- Master the flexbox layout model and its two-axis system
- Understand how flex-grow, flex-shrink, and flex-basis work together
- Control alignment on both main and cross axes
- Build common layout patterns with flexbox
- Debug flexbox sizing issues effectively

---

## The Flexbox Model

**Flexbox** (Flexible Box Layout) is a one-dimensional layout system for distributing space along a **single axis** (row or column).

**Key concepts:**
- **Flex container** - the parent with `display: flex`
- **Flex items** - direct children of the flex container
- **Main axis** - primary direction (horizontal for row, vertical for column)
- **Cross axis** - perpendicular to main axis

```css
.container {
  display: flex;  /* or display: inline-flex; */
}
```

---

## Axes and Directions

### Main Axis (Primary Direction)

Controlled by `flex-direction`:

```css
.container {
  flex-direction: row;          /* → (default) */
  flex-direction: row-reverse;  /* ← */
  flex-direction: column;       /* ↓ */
  flex-direction: column-reverse; /* ↑ */
}
```

**Visualization:**

```
flex-direction: row (default)
┌─────────────────────────────┐
│ ┌───┐ ┌───┐ ┌───┐          │
│ │ 1 │ │ 2 │ │ 3 │   →      │  Main axis
│ └───┘ └───┘ └───┘          │
└─────────────────────────────┘
       ↕ Cross axis

flex-direction: column
┌─────────────────┐
│ ┌─────────────┐ │
│ │      1      │ │  ↓ Main axis
│ └─────────────┘ │
│ ┌─────────────┐ │
│ │      2      │ │
│ └─────────────┘ │
│ ┌─────────────┐ │
│ │      3      │ │
│ └─────────────┘ │
└─────────────────┘
  ↔ Cross axis
```

---

## Container Properties

### `flex-wrap` - Control Line Wrapping

```css
.container {
  flex-wrap: nowrap;       /* Default - single line, may overflow */
  flex-wrap: wrap;         /* Multi-line, wrap to next line */
  flex-wrap: wrap-reverse; /* Multi-line, wrap in reverse */
}
```

**Example:**

```css
.container {
  display: flex;
  flex-wrap: wrap;
  width: 400px;
}

.item {
  width: 150px;
}
```

**Result:** Items wrap when they don't fit (3 items = 450px > 400px container).

```
┌─────────────────────────┐
│ ┌─────┐ ┌─────┐        │  Line 1
│ │  1  │ │  2  │        │
│ └─────┘ └─────┘        │
│ ┌─────┐                │  Line 2 (wrapped)
│ │  3  │                │
│ └─────┘                │
└─────────────────────────┘
```

---

### `flex-flow` - Shorthand for Direction + Wrap

```css
.container {
  flex-flow: row wrap;
  /* Same as */
  flex-direction: row;
  flex-wrap: wrap;
}
```

---

### `justify-content` - Main Axis Alignment

Distributes space **along the main axis**.

```css
.container {
  justify-content: flex-start;    /* ← (default) */
  justify-content: flex-end;      /* → */
  justify-content: center;        /* Center items */
  justify-content: space-between; /* Even space between items */
  justify-content: space-around;  /* Even space around items */
  justify-content: space-evenly;  /* Equal space everywhere */
}
```

**Visual comparison:**

```
flex-start (default):
┌─────────────────────────────┐
│ ┌───┐ ┌───┐ ┌───┐          │
│ │ 1 │ │ 2 │ │ 3 │          │
│ └───┘ └───┘ └───┘          │
└─────────────────────────────┘

flex-end:
┌─────────────────────────────┐
│          ┌───┐ ┌───┐ ┌───┐ │
│          │ 1 │ │ 2 │ │ 3 │ │
│          └───┘ └───┘ └───┘ │
└─────────────────────────────┘

center:
┌─────────────────────────────┐
│      ┌───┐ ┌───┐ ┌───┐     │
│      │ 1 │ │ 2 │ │ 3 │     │
│      └───┘ └───┘ └───┘     │
└─────────────────────────────┘

space-between:
┌─────────────────────────────┐
│ ┌───┐     ┌───┐     ┌───┐  │
│ │ 1 │     │ 2 │     │ 3 │  │
│ └───┘     └───┘     └───┘  │
└─────────────────────────────┘

space-around:
┌─────────────────────────────┐
│  ┌───┐   ┌───┐   ┌───┐     │
│  │ 1 │   │ 2 │   │ 3 │     │
│  └───┘   └───┘   └───┘     │
└─────────────────────────────┘

space-evenly:
┌─────────────────────────────┐
│   ┌───┐   ┌───┐   ┌───┐    │
│   │ 1 │   │ 2 │   │ 3 │    │
│   └───┘   └───┘   └───┘    │
└─────────────────────────────┘
```

---

### `align-items` - Cross Axis Alignment (All Items)

Aligns items **on the cross axis** within a single line.

```css
.container {
  align-items: stretch;     /* Default - stretch to fill container */
  align-items: flex-start;  /* Align to start of cross axis */
  align-items: flex-end;    /* Align to end of cross axis */
  align-items: center;      /* Center on cross axis */
  align-items: baseline;    /* Align text baselines */
}
```

**Visual (flex-direction: row):**

```
stretch (default):
┌─────────────────────────────┐
│ ┌───────┐ ┌───────┐ ┌─────┐│  All items stretch to container height
│ │   1   │ │   2   │ │  3  ││
│ │       │ │       │ │     ││
│ └───────┘ └───────┘ └─────┘│
└─────────────────────────────┘

flex-start:
┌─────────────────────────────┐
│ ┌───┐ ┌───┐ ┌───┐          │  Items at top
│ │ 1 │ │ 2 │ │ 3 │          │
│ └───┘ └───┘ └───┘          │
│                             │
└─────────────────────────────┘

center:
┌─────────────────────────────┐
│                             │
│ ┌───┐ ┌───┐ ┌───┐          │  Items centered vertically
│ │ 1 │ │ 2 │ │ 3 │          │
│ └───┘ └───┘ └───┘          │
└─────────────────────────────┘

baseline:
┌─────────────────────────────┐
│ ┌─────┐ ┌───┐ ┌───────┐    │  Text baselines aligned
│ │ Big │ │ 2 │ │ Small │    │
│ │     │ └───┘ └───────┘    │
│ └─────┘                     │
└─────────────────────────────┘
```

---

### `align-content` - Cross Axis Alignment (Multiple Lines)

Only applies when `flex-wrap: wrap` creates multiple lines.

```css
.container {
  flex-wrap: wrap;
  align-content: stretch;       /* Default */
  align-content: flex-start;    /* Pack lines at start */
  align-content: flex-end;      /* Pack lines at end */
  align-content: center;        /* Center lines */
  align-content: space-between; /* Space between lines */
  align-content: space-around;  /* Space around lines */
  align-content: space-evenly;  /* Even space */
}
```

**Difference from `align-items`:**
- `align-items` - aligns items within each line
- `align-content` - aligns the lines themselves

---

### `gap` - Spacing Between Items (Modern)

```css
.container {
  display: flex;
  gap: 1rem;              /* All gaps */
  gap: 1rem 2rem;         /* row-gap | column-gap */
  row-gap: 1rem;
  column-gap: 2rem;
}
```

**Benefits over margins:**
- Only adds space **between** items, not at edges
- Works in both directions
- Simpler than `:not(:last-child)` tricks

**Example:**

```css
/* Old way - margins */
.item {
  margin-right: 1rem;
}
.item:last-child {
  margin-right: 0;
}

/* New way - gap */
.container {
  display: flex;
  gap: 1rem;
}
```

---

## Item Properties

### `order` - Visual Reordering

Changes visual order without affecting DOM order (for accessibility, DOM order matters).

```css
.item1 { order: 2; }
.item2 { order: 1; }  /* Appears first */
.item3 { order: 3; }
```

**Default:** All items have `order: 0`.

**Sorted by:** Lower order values appear first.

**Use case:** Visual reordering for responsive layouts.

```css
/* Mobile: logo, menu, search */
/* Desktop: logo, search, menu */

.logo { order: 1; }
.menu { order: 3; }
.search { order: 2; }

@media (min-width: 768px) {
  .menu { order: 2; }
  .search { order: 3; }
}
```

**Warning:** Avoid for interactive elements (keyboard navigation follows DOM order).

---

### `flex-grow` - Growing Factor

How much an item should **grow** relative to siblings when **extra space** is available.

```css
.item {
  flex-grow: 0;  /* Default - don't grow */
  flex-grow: 1;  /* Grow to fill space */
  flex-grow: 2;  /* Grow twice as much as flex-grow: 1 items */
}
```

**Example:**

```css
.container {
  display: flex;
  width: 600px;
}

.item1 { width: 100px; flex-grow: 1; }
.item2 { width: 100px; flex-grow: 2; }
.item3 { width: 100px; flex-grow: 1; }
```

**Calculation:**
1. Total width: 600px
2. Used space: 100px + 100px + 100px = 300px
3. Extra space: 600px - 300px = 300px
4. Total grow factor: 1 + 2 + 1 = 4
5. Each grow unit: 300px / 4 = 75px

**Final widths:**
- item1: 100px + (1 × 75px) = 175px
- item2: 100px + (2 × 75px) = 250px
- item3: 100px + (1 × 75px) = 175px

---

### `flex-shrink` - Shrinking Factor

How much an item should **shrink** relative to siblings when **space is insufficient**.

```css
.item {
  flex-shrink: 1;  /* Default - shrink proportionally */
  flex-shrink: 0;  /* Don't shrink (may cause overflow) */
  flex-shrink: 2;  /* Shrink twice as much */
}
```

**Example:**

```css
.container {
  display: flex;
  width: 400px;
}

.item1 { width: 200px; flex-shrink: 1; }
.item2 { width: 200px; flex-shrink: 2; }
.item3 { width: 200px; flex-shrink: 1; }
```

**Calculation:**
1. Container: 400px
2. Total width: 200px + 200px + 200px = 600px
3. Overflow: 600px - 400px = 200px (need to shrink)
4. Total shrink factor: 1 + 2 + 1 = 4
5. Each shrink unit: 200px / 4 = 50px

**Final widths:**
- item1: 200px - (1 × 50px) = 150px
- item2: 200px - (2 × 50px) = 100px
- item3: 200px - (1 × 50px) = 150px

---

### `flex-basis` - Base Size Before Growing/Shrinking

The **initial size** of an item before `flex-grow` or `flex-shrink` is applied.

```css
.item {
  flex-basis: auto;   /* Default - use width/height */
  flex-basis: 200px;  /* Base size is 200px */
  flex-basis: 50%;    /* 50% of container */
  flex-basis: 0;      /* Zero base (useful with flex-grow) */
}
```

**Priority:**
```
flex-basis (if not 'auto') > width/height > content size
```

**Example:**

```css
.item {
  width: 100px;
  flex-basis: 200px;  /* flex-basis wins, item is 200px base */
}
```

**`flex-basis: 0` trick:**

```css
/* Equal width columns, ignoring content */
.item {
  flex-grow: 1;
  flex-basis: 0;  /* Start from 0, divide space equally */
}
```

**Without `flex-basis: 0`:**
- Items grow from their content size
- Unequal columns if content differs

**With `flex-basis: 0`:**
- All items start from 0
- Space divided equally by `flex-grow`

---

### `flex` Shorthand - Combining grow, shrink, basis

```css
.item {
  flex: <grow> <shrink> <basis>;
}
```

**Common patterns:**

```css
/* Don't grow or shrink, fixed size */
.item { flex: 0 0 200px; }

/* Grow to fill space, shrink if needed, zero base */
.item { flex: 1 1 0%; }

/* Shorthand for above */
.item { flex: 1; }  /* Equivalent to 1 1 0% */

/* Grow but don't shrink, auto base */
.item { flex: 1 0 auto; }

/* Don't grow or shrink, content-based size */
.item { flex: none; }  /* Equivalent to 0 0 auto */

/* Grow and shrink from content size */
.item { flex: auto; }  /* Equivalent to 1 1 auto */
```

**Best practice:** Always use the `flex` shorthand, not individual properties.

**Why?** The shorthand sets sensible defaults for omitted values:
- `flex: 1` → `1 1 0%` (not `1 1 auto`)
- `flex: 200px` → `1 1 200px` (not `0 0 200px`)

---

### `align-self` - Override Cross Axis Alignment for One Item

```css
.container {
  align-items: center;  /* All items centered */
}

.item-special {
  align-self: flex-end;  /* This one at end */
}
```

**Values:** Same as `align-items` (stretch, flex-start, flex-end, center, baseline).

---

## Common Layout Patterns

### Pattern 1: Horizontal Navbar

```css
.navbar {
  display: flex;
  gap: 1rem;
  padding: 1rem;
  background: #333;
}

.navbar a {
  color: white;
  text-decoration: none;
}

.navbar .spacer {
  flex: 1;  /* Push subsequent items to the right */
}
```

```html
<nav class="navbar">
  <a href="/">Logo</a>
  <a href="/about">About</a>
  <a href="/contact">Contact</a>
  <div class="spacer"></div>
  <a href="/login">Login</a>
</nav>
```

---

### Pattern 2: Card with Header, Body, Footer

```css
.card {
  display: flex;
  flex-direction: column;
  height: 400px;
}

.card-header,
.card-footer {
  flex: 0 0 auto;  /* Fixed size, don't grow or shrink */
}

.card-body {
  flex: 1;  /* Grow to fill remaining space */
  overflow: auto;  /* Scroll if content too long */
}
```

---

### Pattern 3: Holy Grail Layout (Header, 3 Columns, Footer)

```css
body {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

header, footer {
  flex: 0 0 auto;
}

.main-content {
  flex: 1;  /* Grow to fill space */
  display: flex;
}

.sidebar-left,
.sidebar-right {
  flex: 0 0 200px;  /* Fixed width sidebars */
}

.content {
  flex: 1;  /* Content fills remaining width */
}
```

---

### Pattern 4: Equal-Width Columns

```css
.container {
  display: flex;
  gap: 1rem;
}

.column {
  flex: 1;  /* All columns equal width */
}
```

---

### Pattern 5: Responsive Columns (Wrap)

```css
.container {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
}

.item {
  flex: 1 1 250px;  /* Grow, shrink, min 250px */
}
```

**Behavior:**
- Items grow to fill space
- Wrap to next line when < 250px

---

### Pattern 6: Centering (Horizontal + Vertical)

```css
.container {
  display: flex;
  justify-content: center;  /* Horizontal */
  align-items: center;      /* Vertical */
  min-height: 100vh;
}
```

---

### Pattern 7: Sticky Footer

```css
body {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

main {
  flex: 1;  /* Pushes footer to bottom */
}

footer {
  flex: 0 0 auto;
}
```

---

## Debugging Flexbox

### DevTools

**Chrome/Edge/Firefox:**
1. Inspect flex container
2. Elements panel shows a **Flexbox** badge
3. Click badge to highlight main/cross axes
4. Hover over items to see flex values

---

### Common Issues

#### Issue 1: Items Not Growing

**Problem:**
```css
.item {
  width: 200px;
  flex-grow: 1;  /* Doesn't grow? */
}
```

**Cause:** `width` acts as `flex-basis: auto`, which reads `width`.

**Fix:**
```css
.item {
  flex: 1 1 0%;  /* Start from 0, grow equally */
}
```

---

#### Issue 2: Items Overflowing

**Problem:** Text overflows container.

**Cause:** Default `min-width: auto` prevents shrinking below content.

**Fix:**
```css
.item {
  min-width: 0;  /* Allow shrinking below content width */
  overflow: hidden;
  text-overflow: ellipsis;
}
```

---

#### Issue 3: Unequal Heights with `align-items: stretch`

**Problem:** Items have different heights despite `stretch`.

**Cause:** Items have explicit `height` or `align-self`.

**Fix:** Remove `height` or use `align-self: stretch`.

---

#### Issue 4: Gap Not Working

**Problem:** `gap` has no effect.

**Cause:** Old browser (pre-2021).

**Fallback:**
```css
.container {
  display: flex;
  margin: -0.5rem;  /* Negative margin trick */
}

.item {
  margin: 0.5rem;
}
```

---

## Flexbox vs Grid

| Use Case | Flexbox | Grid |
|----------|---------|------|
| One-dimensional layout (row or column) | ✅ | ❌ |
| Two-dimensional layout (rows and columns) | ❌ | ✅ |
| Content-driven sizing | ✅ | ❌ |
| Equal-height columns | ✅ | ✅ |
| Responsive wrapping | ✅ | ✅ |
| Explicit placement | ❌ | ✅ |

**When to use Flexbox:**
- Navigation bars
- Button groups
- Card layouts (single row/column)
- Centering
- Dynamic content sizes

**When to use Grid:**
- Page layouts
- Complex 2D layouts
- Precise item placement
- Overlapping elements

---

## Key Takeaways

✅ **Flexbox is one-dimensional** (main axis + cross axis)

✅ **`flex: 1` is shorthand for `1 1 0%`**, not `1 1 auto`

✅ **Use `gap` for spacing**, not margins (cleaner, simpler)

✅ **`justify-content` controls main axis**, `align-items` controls cross axis

✅ **`flex-basis` sets base size before growing/shrinking**

✅ **Set `min-width: 0` on items** if text overflows

✅ **Use `flex-wrap: wrap` for responsive layouts**

✅ **`align-content` only applies with multiple lines** (wrap)

✅ **Order doesn't change DOM order** (affects accessibility)

---

## Resources

- [MDN: Flexbox](https://developer.mozilla.org/en-US/docs/Learn/CSS/CSS_layout/Flexbox)
- [CSS Tricks: A Complete Guide to Flexbox](https://css-tricks.com/snippets/css/a-guide-to-flexbox/)
- [Flexbox Froggy](https://flexboxfroggy.com/) - Interactive game
- [Flexbox Defense](http://www.flexboxdefense.com/) - Tower defense game
- [Flexbox Patterns](https://www.flexboxpatterns.com/)
