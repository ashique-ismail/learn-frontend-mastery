# Container Queries

## Overview

Container queries allow you to apply styles to an element based on the size of its **container** rather than the viewport. This enables true component-based responsive design, where components adapt based on their context, not the global page size.

Before container queries, responsive design was page-centric (media queries). With container queries, design becomes component-centric.

## Basic Syntax

```css
/* Define a containment context */
.card-container {
  container-type: inline-size; /* or size, normal */
  /* Optional: name the container */
  container-name: card;
}

/* Query the container */
@container (min-width: 400px) {
  .card {
    display: grid;
    grid-template-columns: 1fr 2fr;
  }
}

/* Query a named container */
@container card (min-width: 400px) {
  .card-title {
    font-size: 2rem;
  }
}
```

## Container Types

### `container-type` Values

```css
/* inline-size: queries inline dimension only (width in horizontal writing mode) */
.sidebar {
  container-type: inline-size; /* Most common, best performance */
}

/* size: queries both dimensions (width and height) */
.chart-wrapper {
  container-type: size; /* Can create layout loops if not careful */
}

/* normal: establishes container for style queries but not size queries */
.theme-wrapper {
  container-type: normal; /* Used with style queries */
}
```

**Key Difference:**
- `inline-size`: Only tracks width (horizontal writing mode), allows height to be determined by content
- `size`: Tracks both dimensions, but requires explicit height on container (can't use auto)

### Performance Considerations

```css
/* PREFERRED - inline-size (no layout loops) */
.responsive-component {
  container-type: inline-size; /* Height can flow naturally */
}

/* CAREFUL - size (requires explicit height) */
.modal-content {
  container-type: size;
  height: 500px; /* Must have explicit height, can't use auto */
}
```

## Container Names

```css
/* Name containers for specificity */
.sidebar {
  container-name: sidebar;
  container-type: inline-size;
}

.main {
  container-name: main;
  container-type: inline-size;
}

/* Query specific named containers */
@container sidebar (min-width: 300px) {
  .widget { /* Styles for widgets in sidebar */ }
}

@container main (min-width: 600px) {
  .widget { /* Different styles for widgets in main */ }
}

/* Shorthand syntax */
.card {
  container: card / inline-size; /* name / type */
}
```

## Container Query Units

New length units relative to container dimensions:

| Unit | Meaning | Relative To |
|------|---------|-------------|
| `cqw` | Container Query Width | 1% of container width |
| `cqh` | Container Query Height | 1% of container height |
| `cqi` | Container Query Inline | 1% of container inline size |
| `cqb` | Container Query Block | 1% of container block size |
| `cqmin` | Container Query Min | Smaller of `cqi` or `cqb` |
| `cqmax` | Container Query Max | Larger of `cqi` or `cqb` |

```css
.card-container {
  container-type: inline-size;
}

.card-title {
  /* Font size scales with container, not viewport */
  font-size: clamp(1rem, 5cqi, 3rem);
}

.card-image {
  /* Image adapts to container width */
  width: 90cqw;
  max-width: 500px;
}

.card-padding {
  /* Spacing scales with container */
  padding: 2cqi 3cqi;
}
```

## Practical Examples

### Responsive Card Component

```css
/* Container setup */
.card-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 1rem;
}

.card-wrapper {
  container-type: inline-size;
}

/* Component adapts to available space */
.card {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

/* Small container: stacked layout */
@container (max-width: 349px) {
  .card-image {
    aspect-ratio: 16 / 9;
  }
  
  .card-title {
    font-size: 1.2rem;
  }
}

/* Medium container: side-by-side layout */
@container (min-width: 350px) and (max-width: 549px) {
  .card {
    flex-direction: row;
  }
  
  .card-image {
    width: 40%;
    aspect-ratio: 1;
  }
  
  .card-content {
    flex: 1;
  }
}

/* Large container: enhanced layout */
@container (min-width: 550px) {
  .card {
    flex-direction: row;
  }
  
  .card-image {
    width: 45%;
    aspect-ratio: 4 / 3;
  }
  
  .card-title {
    font-size: 2rem;
  }
  
  .card-metadata {
    display: flex;
    gap: 1rem;
  }
}
```

### Responsive Navigation

```css
.nav-container {
  container-type: inline-size;
}

.nav {
  display: flex;
}

/* Collapsed menu for narrow containers */
@container (max-width: 599px) {
  .nav {
    flex-direction: column;
  }
  
  .nav-toggle {
    display: block;
  }
  
  .nav-items {
    display: none;
  }
  
  .nav-items.expanded {
    display: flex;
    flex-direction: column;
  }
}

/* Full menu for wider containers */
@container (min-width: 600px) {
  .nav-items {
    display: flex;
    flex-direction: row;
    gap: 2rem;
  }
  
  .nav-toggle {
    display: none;
  }
}
```

### Data Table Transformation

```css
.table-wrapper {
  container-type: inline-size;
}

.data-table {
  width: 100%;
}

/* Standard table layout */
@container (min-width: 640px) {
  .data-table {
    display: table;
  }
}

/* Card layout for narrow containers */
@container (max-width: 639px) {
  .data-table,
  .data-table tbody,
  .data-table tr,
  .data-table th,
  .data-table td {
    display: block;
  }
  
  .data-table thead {
    display: none;
  }
  
  .data-table tr {
    margin-bottom: 1rem;
    border: 1px solid #ddd;
    border-radius: 8px;
    padding: 1rem;
  }
  
  .data-table td::before {
    content: attr(data-label) ": ";
    font-weight: bold;
  }
}
```

## Style Queries (Experimental)

Query container custom properties:

```css
.theme-wrapper {
  container-name: theme;
  --theme: dark;
}

/* Query custom property values */
@container style(--theme: dark) {
  .content {
    background: #1a1a1a;
    color: #ffffff;
  }
}

@container style(--theme: light) {
  .content {
    background: #ffffff;
    color: #1a1a1a;
  }
}
```

## Nested Container Queries

```css
.page-container {
  container-name: page;
  container-type: inline-size;
}

.sidebar {
  container-name: sidebar;
  container-type: inline-size;
}

/* Nested containers create independent contexts */
@container page (min-width: 1200px) {
  .sidebar {
    width: 300px;
  }
  
  /* Sidebar's children query sidebar, not page */
  @container sidebar (min-width: 250px) {
    .widget {
      padding: 2rem;
    }
  }
}
```

## Container Queries vs Media Queries

| Aspect | Media Queries | Container Queries |
|--------|---------------|-------------------|
| Scope | Global viewport | Specific container |
| Use Case | Page-level layout | Component-level design |
| Reusability | Component tied to viewport | Component truly modular |
| Nesting | Limited usefulness | Natural component hierarchy |
| Units | `vw`, `vh`, `vmin`, `vmax` | `cqw`, `cqh`, `cqi`, `cqb` |

```css
/* MEDIA QUERY: component breaks at specific viewport width */
@media (max-width: 768px) {
  .card {
    /* What if card is in a narrow sidebar at 1920px viewport? */
    flex-direction: column;
  }
}

/* CONTAINER QUERY: component adapts to actual available space */
@container (max-width: 400px) {
  .card {
    /* Works in sidebar, modal, grid, anywhere */
    flex-direction: column;
  }
}
```

## Use Cases

### 1. Truly Reusable Components

```css
/* Same component adapts differently based on context */
.product-card-wrapper {
  container-type: inline-size;
}

/* In wide main content area: horizontal layout */
/* In narrow sidebar: vertical layout */
/* No need to know the viewport size */
```

### 2. Dynamic Layout Grids

```css
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1rem;
}

/* Each grid item is a container */
.grid-item {
  container-type: inline-size;
}

/* Content adapts to grid item width, not viewport */
@container (min-width: 300px) {
  .grid-item-content {
    padding: 2rem;
    font-size: 1.2rem;
  }
}
```

### 3. Sidebar/Main Layouts

```css
/* Sidebar might be hidden on mobile, visible on desktop */
/* Components in sidebar need different breakpoints than main */
.sidebar {
  container-type: inline-size;
}

.main {
  container-type: inline-size;
}

/* Components query their immediate container */
/* Not the viewport */
```

## Browser Support Considerations

```css
/* Feature detection */
@supports (container-type: inline-size) {
  .card-wrapper {
    container-type: inline-size;
  }
  
  @container (min-width: 400px) {
    .card {
      display: grid;
    }
  }
}

/* Fallback for older browsers */
@supports not (container-type: inline-size) {
  @media (min-width: 768px) {
    .card {
      display: grid;
    }
  }
}
```

## Common Gotchas

### 1. Container Must Have Layout

```css
/* BAD: inline element can't be container */
span {
  container-type: inline-size; /* No effect */
}

/* GOOD: block or inline-block */
.wrapper {
  display: block; /* or flex, grid, inline-block */
  container-type: inline-size;
}
```

### 2. Size Containment Requires Explicit Dimensions

```css
/* BAD: size with auto height causes layout loop */
.box {
  container-type: size;
  height: auto; /* Children can't query height that depends on them */
}

/* GOOD: explicit height */
.box {
  container-type: size;
  height: 500px; /* Or use inline-size if only width matters */
}

/* BETTER: inline-size for most cases */
.box {
  container-type: inline-size; /* Height flows naturally */
}
```

### 3. Container Queries Don't Replace Media Queries

```css
/* Still use media queries for page-level concerns */
@media (max-width: 768px) {
  .sidebar {
    display: none; /* Page layout decision */
  }
}

/* Use container queries for component adaptation */
@container (max-width: 400px) {
  .widget {
    padding: 1rem; /* Component styling decision */
  }
}
```

### 4. Containment Side Effects

```css
.container {
  container-type: inline-size;
  /* Creates containment: */
  /* - New stacking context */
  /* - New containing block for positioned descendants */
  /* - Cuts off overflow */
}

/* Be aware of implications */
.absolute-child {
  position: absolute;
  /* Now positioned relative to .container, not nearest positioned ancestor */
}
```

## Performance Benefits

1. **Localized Recalculation**: Container queries only recalculate when container size changes, not viewport
2. **Better Component Isolation**: Reduces style invalidation scope
3. **Fewer Media Query Checks**: Components query their container once, not global viewport repeatedly

```css
/* Performance-conscious pattern */
.page {
  display: grid;
  grid-template-columns: 250px 1fr;
}

/* Only sidebar needs to recalculate when resized */
.sidebar {
  container-type: inline-size;
}

/* Main content independent of sidebar */
.main {
  container-type: inline-size;
}
```

## Advanced Patterns

### Container-Driven Typography

```css
.content-wrapper {
  container-type: inline-size;
}

.content {
  /* Fluid typography based on container */
  font-size: clamp(1rem, 2.5cqi, 1.5rem);
  line-height: calc(1.4 + 0.2cqi);
  
  /* Spacing scales with container */
  --space-sm: 1cqi;
  --space-md: 2cqi;
  --space-lg: 4cqi;
  
  padding: var(--space-md);
  gap: var(--space-sm);
}
```

### Conditional Feature Display

```css
.card-wrapper {
  container-type: inline-size;
}

/* Progressive enhancement based on space */
.card-tags {
  display: none; /* Hidden by default */
}

@container (min-width: 300px) {
  .card-tags {
    display: flex; /* Show when space available */
  }
}

.card-description-long {
  display: none;
}

@container (min-width: 400px) {
  .card-description-short {
    display: none;
  }
  
  .card-description-long {
    display: block;
  }
}
```

## Interview Questions

**Q1: What's the main difference between media queries and container queries?**

**A:** Media queries respond to viewport dimensions (global), while container queries respond to a specific container's dimensions (local). This makes components truly modular - the same component can adapt differently when placed in a narrow sidebar vs. wide main content area, without knowing anything about the page layout.

**Q2: Why is `container-type: inline-size` preferred over `size`?**

**A:** `inline-size` only establishes containment on the inline axis (usually width), allowing the block axis (height) to size naturally based on content. `size` requires explicit dimensions on both axes, which can create circular dependencies (layout loops) and is less flexible. Use `inline-size` unless you specifically need to query both dimensions.

**Q3: How do container query units differ from viewport units?**

**A:** Container query units (`cqi`, `cqw`, etc.) are relative to the container's dimensions, not the viewport. This allows truly responsive component internals - `5cqi` means 5% of the container's inline size, so the same value produces different results in different contexts.

**Q4: What happens when you nest containers?**

**A:** Each container creates an independent containment context. A nested container queries its own dimensions, not its parent container's. This enables component composition where each level can adapt independently to its available space.

**Q5: What are the containment side effects to watch for?**

**A:** Container types establish containment, which creates:
1. A new stacking context (affects z-index)
2. A containing block for absolutely positioned descendants
3. A new formatting context
4. Paint and layout containment (can clip overflow)

These can affect how positioned elements, z-index, and overflow behave.

**Q6: When should you still use media queries?**

**A:** Use media queries for:
- Page-level layout decisions (show/hide sidebar)
- Navigation structure changes
- Global design token adjustments
- Features dependent on device capabilities (hover, pointer precision)
- Print styles

Use container queries for component-level responsive behavior.

## Resources

- [MDN: CSS Container Queries](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Container_Queries)
- [CSS Containment Module Spec](https://www.w3.org/TR/css-contain-3/)
- [Ahmad Shadeed: Container Queries Guide](https://ishadeed.com/article/css-container-query-guide/)
- [Web.dev: Container Queries](https://web.dev/cq-stable/)
- [Una Kravets: Container Queries](https://una.im/container-queries/)
- [Can I Use: Container Queries](https://caniuse.com/css-container-queries)

## Summary

Container queries revolutionize responsive design by making components context-aware rather than viewport-aware. They enable truly modular, reusable components that adapt to their container's size. Use `container-type: inline-size` for most cases, leverage container query units for scalable internals, and combine with media queries for page-level concerns. The result is more maintainable, composable, and flexible responsive design.
