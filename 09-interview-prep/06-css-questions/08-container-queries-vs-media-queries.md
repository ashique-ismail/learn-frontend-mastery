# Container Queries vs Media Queries

## The Core Difference

**Media queries** — respond to the **viewport** (browser window) size.

**Container queries** — respond to the **parent container's** size.

---

## Why Container Queries Were Needed

The component-reuse problem with media queries:

```css
/* ❌ Media query: This card looks right at desktop viewport */
@media (min-width: 768px) {
  .card {
    display: flex;
    flex-direction: row;
  }
}

/* BUT if the card is in a sidebar (narrow container), it's still horizontal
   even though it doesn't have room for a horizontal layout */
```

A card component can't know where it'll be placed. It might be in a wide main area, a narrow sidebar, or a modal. Media queries use viewport width as a proxy for "available space," but that breaks when layout changes.

---

## Container Query Syntax

```css
/* 1. Define a containment context on the parent */
.card-container {
  container-type: inline-size; /* respond to width changes */
  /* or: container-type: size (respond to width AND height) */
}

/* 2. Optional: name the container */
.sidebar {
  container-type: inline-size;
  container-name: sidebar; /* named container */
}

/* Shorthand: */
.card-container {
  container: sidebar / inline-size;
}

/* 3. Write container queries targeting the container */
@container (min-width: 500px) {
  .card {
    display: flex;
    flex-direction: row;
  }
}

/* Target a named container */
@container sidebar (min-width: 300px) {
  .widget { font-size: 0.875rem; }
}
```

---

## Real-World Example: Card Component

```html
<!-- Same card, different contexts -->
<aside class="sidebar" style="container-type: inline-size">
  <div class="card">...</div>  <!-- gets vertical layout -->
</aside>

<main class="main-content" style="container-type: inline-size">
  <div class="card">...</div>  <!-- gets horizontal layout at 500px+ -->
</main>
```

```css
/* Card defaults to vertical layout */
.card {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.card img { width: 100%; aspect-ratio: 16/9; }

/* Horizontal layout when container is wide enough */
@container (min-width: 500px) {
  .card {
    flex-direction: row;
  }

  .card img {
    width: 200px;
    flex-shrink: 0;
    aspect-ratio: 1;
  }
}
```

---

## Container Query Units (`cq*`)

Like viewport units but relative to the nearest container:

```css
.card-title {
  font-size: clamp(1rem, 5cqw, 2rem);
  /* 5% of the container's width — fluid within the container */
}
```

| Unit | Definition |
|---|---|
| `cqw` | 1% of container's width |
| `cqh` | 1% of container's height |
| `cqi` | 1% of container's inline size |
| `cqb` | 1% of container's block size |
| `cqmin` | smaller of `cqi` or `cqb` |
| `cqmax` | larger of `cqi` or `cqb` |

---

## When to Use Each

**Use media queries for:**
- Page-level layout changes (sidebar appears at desktop)
- Navigation changes (hamburger on mobile)
- Hiding/showing major sections
- Font-size adjustments on the root element
- Feature detection (`@supports`)

**Use container queries for:**
- Component-level responsive behavior
- Cards, widgets, tiles that appear in different contexts
- Any component you reuse across different layout regions
- Design system components

---

## Browser Support

Container queries are supported in all modern browsers since 2022:
- Chrome 105+, Safari 16+, Firefox 110+
- Safe for production use

---

## Common Interview Questions

**Q: Can you nest container queries?**
Yes. Each `container-type` creates a new containment context. Inner `@container` queries respond to the nearest ancestor container.

**Q: What's the difference between `container-type: inline-size` and `container-type: size`?**
`inline-size` enables containment for the inline dimension only (typically width). `size` enables both inline and block dimensions. Using `size` requires explicit height on the container or it collapses to 0.

**Q: Does `container-type` affect layout?**
`container-type: inline-size` establishes a block formatting context (like `overflow: hidden`). This means floats inside are contained, and margin collapsing works differently. Usually not a problem but worth knowing.

**Q: Can a container query target an ancestor several levels up?**
Named containers solve this. Use `container-name: my-layout` on any ancestor, then `@container my-layout (min-width: 800px)` targets that specific ancestor regardless of nesting depth.
