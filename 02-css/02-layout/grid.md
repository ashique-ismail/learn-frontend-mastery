# CSS Grid Layout

## Overview

CSS Grid is a two-dimensional layout system that allows you to create complex layouts by defining rows and columns simultaneously. Unlike Flexbox (which is one-dimensional), Grid excels at creating structured layouts with precise control over both axes.

## Grid Container vs Grid Items

```css
.container {
  display: grid; /* or inline-grid */
}
```

When an element becomes a grid container:
- All direct children become **grid items**
- Items are placed into grid cells defined by rows and columns
- Items can span multiple cells

## Explicit vs Implicit Grid

### Explicit Grid
Defined using `grid-template-rows` and `grid-template-columns`:

```css
.grid {
  display: grid;
  grid-template-columns: 100px 200px 100px; /* 3 columns */
  grid-template-rows: 50px 100px; /* 2 rows */
}
```

### Implicit Grid
Created automatically when items are placed outside the explicit grid:

```css
.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  /* Only explicit columns defined, rows are implicit */
  grid-auto-rows: 100px; /* Controls implicit row size */
  grid-auto-flow: row; /* or column, dense */
}
```

## Track Sizing

### Fixed Sizes
```css
grid-template-columns: 200px 150px 100px;
```

### Flexible Sizing with `fr`
```css
/* fr = fraction of available space */
grid-template-columns: 1fr 2fr 1fr; /* 25%, 50%, 25% of available space */
```

### `minmax()`
```css
/* Column is at least 100px, but can grow to take 1fr */
grid-template-columns: minmax(100px, 1fr) 2fr;
```

### `auto`
```css
/* Size based on content */
grid-template-columns: auto 1fr auto;
```

### `fit-content()`
```css
/* Size based on content, but not exceeding 300px */
grid-template-columns: fit-content(300px) 1fr;
```

## `repeat()` Function

```css
/* Instead of: 1fr 1fr 1fr 1fr */
grid-template-columns: repeat(4, 1fr);

/* Mix with other values */
grid-template-columns: 200px repeat(3, 1fr) 100px;

/* Repeat pattern */
grid-template-columns: repeat(3, 100px 200px); /* 100px 200px 100px 200px 100px 200px */
```

### Auto-fill vs Auto-fit

**`auto-fill`**: Creates as many tracks as will fit, leaving empty tracks if items don't fill them.

```css
/* Creates as many 200px columns as fit in container, even if empty */
grid-template-columns: repeat(auto-fill, 200px);
```

**`auto-fit`**: Creates as many tracks as will fit, then collapses empty tracks to 0px.

```css
/* Creates columns as needed, collapses extras */
grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
```

**Key difference**: `auto-fit` collapses empty tracks, allowing remaining items to grow.

## Placing Items

### Line-based Placement
Grid lines are numbered starting from 1:

```css
.item {
  /* Place from column line 1 to 3 (spans 2 columns) */
  grid-column-start: 1;
  grid-column-end: 3;
  /* Shorthand */
  grid-column: 1 / 3;
  
  /* Span syntax */
  grid-column: 1 / span 2; /* Start at 1, span 2 columns */
  
  /* Negative line numbers count from the end */
  grid-column: 1 / -1; /* First to last line (full width) */
}
```

### Shorthand Properties
```css
.item {
  grid-area: row-start / col-start / row-end / col-end;
  grid-area: 1 / 1 / 3 / 4; /* row 1-3, column 1-4 */
}
```

## Named Grid Lines

```css
.grid {
  grid-template-columns: 
    [sidebar-start] 250px 
    [sidebar-end main-start] 1fr 
    [main-end];
  grid-template-rows: 
    [header-start] 100px 
    [header-end content-start] 1fr 
    [content-end footer-start] 50px 
    [footer-end];
}

.sidebar {
  grid-column: sidebar-start / sidebar-end;
  grid-row: content-start / content-end;
}
```

## Grid Template Areas

Define named grid areas for semantic layouts:

```css
.container {
  display: grid;
  grid-template-columns: 200px 1fr 200px;
  grid-template-rows: auto 1fr auto;
  grid-template-areas:
    "header header  header"
    "sidebar main   aside"
    "footer footer  footer";
}

.header  { grid-area: header; }
.sidebar { grid-area: sidebar; }
.main    { grid-area: main; }
.aside   { grid-area: aside; }
.footer  { grid-area: footer; }
```

**Empty cells** use `.`:
```css
grid-template-areas:
  "header header  header"
  "sidebar main   ."      /* Empty cell on right */
  "footer footer  footer";
```

## Gaps

```css
.grid {
  /* Modern syntax */
  row-gap: 20px;
  column-gap: 30px;
  gap: 20px 30px; /* row column */
  gap: 20px; /* same for both */
  
  /* Legacy syntax (deprecated but still works) */
  grid-row-gap: 20px;
  grid-column-gap: 30px;
}
```

**Important**: Gaps only appear *between* items, not on outer edges.

## Alignment

### Justify (Inline/Horizontal Axis)

**`justify-items`**: Aligns items *within their cells*
```css
.grid {
  justify-items: start | end | center | stretch; /* default: stretch */
}
```

**`justify-content`**: Aligns the *entire grid* within the container
```css
.grid {
  justify-content: start | end | center | stretch | space-between | space-around | space-evenly;
}
```

**`justify-self`**: Override alignment for individual items
```css
.item {
  justify-self: start | end | center | stretch;
}
```

### Align (Block/Vertical Axis)

**`align-items`**: Aligns items *within their cells*
```css
.grid {
  align-items: start | end | center | stretch | baseline;
}
```

**`align-content`**: Aligns the *entire grid* within the container
```css
.grid {
  align-content: start | end | center | stretch | space-between | space-around | space-evenly;
}
```

**`align-self`**: Override alignment for individual items
```css
.item {
  align-self: start | end | center | stretch | baseline;
}
```

### Shorthand: `place-*`
```css
.grid {
  place-items: <align-items> <justify-items>;
  place-items: center; /* centers both axes */
  
  place-content: <align-content> <justify-content>;
}

.item {
  place-self: <align-self> <justify-self>;
}
```

## Grid Auto Flow

Controls how auto-placed items are inserted:

```css
.grid {
  grid-auto-flow: row; /* default: fill rows first */
  grid-auto-flow: column; /* fill columns first */
  grid-auto-flow: row dense; /* pack items tightly, may reorder */
  grid-auto-flow: column dense;
}
```

**`dense`**: Attempts to fill holes in the grid, which may cause items to appear out of order (bad for accessibility).

## Common Patterns

### Holy Grail Layout
```css
.container {
  display: grid;
  grid-template-columns: 200px 1fr 200px;
  grid-template-rows: auto 1fr auto;
  min-height: 100vh;
}
```

### Responsive Grid (No Media Queries)
```css
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 20px;
}
```

### Centered Layout with Max Width
```css
.grid {
  display: grid;
  grid-template-columns: 1fr min(1200px, 100%) 1fr;
}

.content {
  grid-column: 2;
}
```

### Full-Bleed Layout
```css
.grid {
  display: grid;
  grid-template-columns: 
    1fr 
    min(65ch, 100%) 
    1fr;
}

.content { grid-column: 2; }
.full-bleed { grid-column: 1 / -1; }
```

### Asymmetric Grid
```css
.grid {
  display: grid;
  grid-template-columns: repeat(12, 1fr);
  gap: 20px;
}

.span-8 { grid-column: span 8; }
.span-4 { grid-column: span 4; }
```

## Grid Item Layering

Items can overlap by occupying the same cells:

```css
.item1 {
  grid-area: 1 / 1 / 3 / 3;
  z-index: 1;
}

.item2 {
  grid-area: 2 / 2 / 4 / 4;
  z-index: 2; /* Appears on top */
}
```

## `grid-template` Shorthand

```css
/* grid-template: rows / columns */
.grid {
  grid-template: 100px 1fr / 200px 1fr;
  /* Equivalent to:
     grid-template-rows: 100px 1fr;
     grid-template-columns: 200px 1fr; */
}

/* With areas */
.grid {
  grid-template:
    "header header" 100px
    "sidebar main" 1fr
    / 200px 1fr;
}
```

## Grid vs Flexbox: When to Use Which

**Use Grid when:**
- You need two-dimensional control (rows AND columns)
- You're designing the overall page layout
- You know the structure ahead of time
- Items should align in both dimensions

**Use Flexbox when:**
- Content is one-dimensional (a row OR a column)
- Content size should determine layout
- You need to distribute space among items dynamically
- You're building a component (navbar, card layout)

**Use both**: Grid for page structure, Flexbox for components within grid cells.

## Performance Considerations

1. **Avoid unnecessary nesting**: Grid items don't need wrapper elements
2. **Use `fr` over percentages**: More performant and flexible
3. **Explicit grid when possible**: Reduces layout recalculations
4. **`grid-auto-flow: dense`**: Use cautiously; can impact accessibility (screen reader order)

## Browser DevTools

All modern browsers have excellent Grid inspection tools:
- Chrome/Edge: Grid badge in Elements panel
- Firefox: Grid overlay with line numbers and area names
- Safari: Grid overlay in Web Inspector

## Gotchas

1. **Margins don't collapse** in Grid (unlike block layout)
2. **`z-index` works** without `position: relative`
3. **Percentage gaps** are relative to the container's corresponding dimension
4. **`auto` tracks** size based on content; can cause overflow if content is too wide
5. **Named lines** ending in `-start` and `-end` create implicit area names

## Interview Questions to Master

1. What's the difference between `auto-fill` and `auto-fit`?
2. When does an implicit grid get created?
3. How do `fr` units calculate available space when mixed with fixed sizes?
4. What's the difference between `justify-content` and `justify-items`?
5. How would you create a responsive grid without media queries?
6. What happens when grid items overlap?
7. Explain the difference between Grid and Flexbox and when to use each.

## Resources

- [CSS Grid Garden](https://cssgridgarden.com/) - Interactive learning game
- [Grid by Example](https://gridbyexample.com/) - Comprehensive patterns by Rachel Andrew
- [MDN CSS Grid Layout](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Grid_Layout)
