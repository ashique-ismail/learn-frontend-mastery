# CSS Subgrid

## Overview

Subgrid allows a grid item to inherit the grid tracks (rows and/or columns) from its parent grid container. This solves the problem of aligning nested grid items with the parent grid structure.

**Browser Support**: Modern browsers (2023+). Check [caniuse.com](https://caniuse.com/css-subgrid) for current status.

## The Problem Subgrid Solves

### Without Subgrid
```css
.parent {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
}

.child {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr; /* Separate grid, no alignment */
}
```

The child's grid tracks are **independent** from the parent's. Items in the child grid won't align with items in the parent grid.

### With Subgrid
```css
.parent {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
}

.child {
  display: grid;
  grid-template-columns: subgrid; /* Inherits parent's column tracks */
}
```

Now the child uses the **same column tracks** as the parent, ensuring perfect alignment.

## Basic Syntax

```css
.grid-item {
  display: grid;
  
  /* Inherit parent's column tracks */
  grid-template-columns: subgrid;
  
  /* Inherit parent's row tracks */
  grid-template-rows: subgrid;
  
  /* Inherit both */
  grid-template-columns: subgrid;
  grid-template-rows: subgrid;
}
```

## When to Use Subgrid

### 1. Card Grid with Aligned Internal Elements

**Problem**: Cards have different content heights, causing internal elements to misalign.

```css
.card-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 20px;
}

.card {
  display: grid;
  grid-template-rows: subgrid; /* Align card internals across all cards */
  grid-row: span 3; /* Card spans 3 rows */
}

/* Now these align perfectly across all cards */
.card__image { grid-row: 1; }
.card__title { grid-row: 2; }
.card__actions { grid-row: 3; }
```

### 2. Form Layouts

Align labels and inputs across nested fieldsets:

```css
.form {
  display: grid;
  grid-template-columns: 150px 1fr;
  gap: 10px;
}

.fieldset {
  display: grid;
  grid-template-columns: subgrid; /* Labels and inputs align with form */
  grid-column: 1 / -1; /* Span all columns */
}
```

### 3. Data Tables with Grouped Rows

```css
.table {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
}

.row-group {
  display: grid;
  grid-template-columns: subgrid;
  grid-column: 1 / -1; /* Span all columns */
}
```

## Spanning Tracks with Subgrid

The subgrid inherits only the tracks it spans:

```css
.parent {
  display: grid;
  grid-template-columns: repeat(6, 1fr);
}

.child {
  display: grid;
  grid-column: 2 / 5; /* Spans columns 2, 3, 4 (3 tracks) */
  grid-template-columns: subgrid; /* Inherits those 3 tracks */
}
```

**Key point**: If the item spans 3 columns, the subgrid has 3 column tracks.

## Gaps in Subgrid

### Gap Inheritance

By default, subgrids **do not inherit gaps** from the parent.

```css
.parent {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 20px;
}

.child {
  grid-template-columns: subgrid;
  /* No gap inherited; you must specify gap explicitly */
  gap: 10px;
}
```

### Gap Override

You can set different gaps in the subgrid:

```css
.parent {
  gap: 20px;
}

.child {
  grid-template-columns: subgrid;
  gap: 0; /* Different gap than parent */
}
```

## Named Grid Lines and Areas

### Named Lines are Inherited

```css
.parent {
  display: grid;
  grid-template-columns: 
    [sidebar-start] 250px 
    [sidebar-end content-start] 1fr 
    [content-end];
}

.child {
  grid-template-columns: subgrid;
  /* Can reference inherited line names */
}

.grandchild {
  grid-column: sidebar-start / content-end;
}
```

### Subgrid with Named Areas

```css
.parent {
  display: grid;
  grid-template-areas:
    "header header"
    "sidebar main"
    "footer footer";
}

.child {
  grid-area: main;
  display: grid;
  grid-template-rows: subgrid; /* Inherits main area's rows */
}
```

## Subgrid on One Axis Only

You can subgrid columns OR rows (or both):

```css
.parent {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-template-rows: repeat(3, 100px);
}

.child {
  display: grid;
  grid-template-columns: subgrid; /* Inherit columns */
  grid-template-rows: auto auto;  /* Define own rows */
}
```

**Common pattern**: Subgrid columns for alignment, define own rows for content flow.

## Nested Subgrids

Subgrids can be nested multiple levels deep:

```css
.level1 {
  display: grid;
  grid-template-columns: repeat(6, 1fr);
}

.level2 {
  display: grid;
  grid-template-columns: subgrid;
  grid-column: 1 / -1;
}

.level3 {
  display: grid;
  grid-template-columns: subgrid; /* Inherits from level2, which inherits from level1 */
  grid-column: 2 / 5;
}
```

## Practical Example: Magazine Layout

```css
.magazine {
  display: grid;
  grid-template-columns: repeat(12, 1fr);
  grid-template-rows: auto;
  gap: 20px;
}

.article {
  display: grid;
  grid-column: span 12;
  grid-template-columns: subgrid; /* Use parent's 12 columns */
  gap: 20px;
}

/* Different article layouts, all aligned to the same grid */
.article--full {
  grid-template-areas: "image image image image image image image image image image image image";
}

.article--hero {
  grid-template-areas: 
    "image image image image image image image image content content content content";
}

.article--sidebar {
  grid-template-areas:
    "content content content content content content content content content sidebar sidebar sidebar";
}
```

## Subgrid with Auto-Placement

Subgrid items can use auto-placement:

```css
.parent {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
}

.child {
  display: grid;
  grid-template-columns: subgrid;
  grid-column: 1 / -1;
}

/* These auto-place into the subgrid */
.grandchild {
  /* Automatically placed into next available cell */
}
```

## Intrinsic Sizing and Subgrid

Subgrids affect parent track sizing when using `auto` or `min-content`/`max-content`:

```css
.parent {
  display: grid;
  grid-template-rows: auto auto auto;
}

.child {
  display: grid;
  grid-template-rows: subgrid;
  grid-row: span 3;
}

/* Content in grandchild affects parent's row heights */
.grandchild {
  height: 500px; /* This influences the parent's 'auto' row sizing */
}
```

## Debugging Subgrids

Use browser DevTools:
- **Firefox**: Best subgrid debugging; shows inherited tracks clearly
- **Chrome/Edge**: Grid inspector highlights subgrids differently
- **Safari**: Grid overlay works but less detailed

## Fallbacks for Older Browsers

### Feature Detection
```css
.child {
  display: grid;
  grid-template-columns: repeat(3, 1fr); /* Fallback */
}

@supports (grid-template-columns: subgrid) {
  .child {
    grid-template-columns: subgrid;
  }
}
```

### Graceful Degradation
Without subgrid, the layout still works but without perfect alignment:

```css
.card {
  display: grid;
  grid-template-rows: auto 1fr auto; /* Fallback: approximate structure */
}

@supports (grid-template-rows: subgrid) {
  .card {
    grid-template-rows: subgrid;
    grid-row: span 3;
  }
}
```

## Common Patterns

### Aligned Card Grid
```css
.cards {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  grid-auto-rows: auto auto 1fr auto; /* 4 rows per card */
  gap: 20px;
}

.card {
  display: grid;
  grid-template-rows: subgrid;
  grid-row: span 4;
}
```

### Form with Nested Groups
```css
.form {
  display: grid;
  grid-template-columns: [labels] 150px [inputs] 1fr;
  gap: 10px;
}

.form-group {
  display: grid;
  grid-template-columns: subgrid;
  grid-column: 1 / -1;
}

.label { grid-column: labels; }
.input { grid-column: inputs; }
```

### Responsive Subgrid
```css
.parent {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
}

.child {
  display: grid;
  grid-template-columns: subgrid;
  grid-column: 1 / -1;
}
```

## Gotchas and Limitations

1. **No gap inheritance**: Must explicitly set gaps on subgrid
2. **Must be a grid item**: Only grid items can use subgrid (not arbitrary descendants)
3. **Span required for alignment**: Subgrid must span the tracks you want to inherit
4. **`minmax()` complexity**: Subgrid items with `minmax()` can cause unexpected sizing
5. **Browser support**: Still not universal (IE11 never, older browsers no)

## Subgrid vs Nested Grid

| Aspect | Nested Grid | Subgrid |
|--------|-------------|---------|
| Track definition | Independent | Inherits from parent |
| Alignment | Not aligned with parent | Aligned with parent |
| Sizing influence | Doesn't affect parent | Can affect parent track sizes |
| Use case | Independent layouts | Aligned, coordinated layouts |

## Interview Questions to Master

1. What problem does subgrid solve that nested grids can't?
2. If a subgrid spans 4 columns, how many column tracks does it have?
3. Are gaps inherited by subgrids?
4. Can you use subgrid on only one axis?
5. How do named grid lines work with subgrid?
6. What's a fallback strategy for browsers without subgrid support?
7. How does subgrid affect parent track sizing?

## Resources

- [MDN Subgrid](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Grid_Layout/Subgrid)
- [CSS Tricks: Subgrid](https://css-tricks.com/css-subgrid/)
- [Rachel Andrew's Subgrid Examples](https://gridbyexample.com/examples/#css-grid-level-2-examples)
