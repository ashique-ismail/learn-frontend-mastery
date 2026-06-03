# Flexbox vs Grid — When to Use Which

## The Core Distinction

**Flexbox** — one-dimensional layout. Control alignment along a single axis (row OR column).

**Grid** — two-dimensional layout. Control both rows AND columns simultaneously.

---

## Flexbox Is Best When

Content drives the layout — items determine their sizes:

```css
/* Navigation bar — items size to their content, distributed evenly */
nav {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 16px;
}

/* Card actions — buttons at end, auto space between */
.card-footer {
  display: flex;
  justify-content: flex-end;
  gap: 8px;
}

/* Centering a single item */
.container {
  display: flex;
  justify-content: center;
  align-items: center;
}
```

**Also great for:** sidebars, toolbars, chip lists, centering.

---

## Grid Is Best When

The layout drives the content — a defined structure that items fill:

```css
/* Page layout — explicit rows and columns */
.page {
  display: grid;
  grid-template-areas:
    "header header"
    "sidebar main"
    "footer footer";
  grid-template-columns: 250px 1fr;
  grid-template-rows: auto 1fr auto;
  min-height: 100vh;
}

/* Card grid — responsive, items fill the grid */
.card-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 24px;
}

/* Holy grail layout — items precisely placed in 2D */
```

**Also great for:** image galleries, dashboards, article layouts with pull quotes.

---

## Key Technical Differences

| Feature | Flexbox | Grid |
|---|---|---|
| Dimensions | 1D (row or column) | 2D (rows and columns) |
| Item alignment | Along main + cross axis | In 2D grid areas |
| Item sizing | Flex grow/shrink/basis | Track sizing (fr, minmax, auto) |
| Gaps | `gap` | `gap`, `row-gap`, `column-gap` |
| Item spanning | No (same track) | Yes (`grid-column: span 2`) |
| Order | `order` property | `order` or `grid-area` placement |
| Implicit tracks | No | Yes (implicit grid) |

---

## The Overlap Zone

Many layouts work fine with either. Prefer Grid when you have explicit structure; prefer Flexbox when content drives size.

```css
/* Works with both — a row of buttons */
.button-row {
  display: flex;  /* content drives width of each button */
  gap: 8px;
}

/* More naturally expressed with Grid if you want equal columns */
.button-row {
  display: grid;
  grid-auto-flow: column;
  grid-auto-columns: 1fr; /* equal width buttons */
  gap: 8px;
}
```

---

## Combine Both

Grid for page structure, Flexbox inside cells:

```css
/* Grid handles the overall layout */
.dashboard {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  gap: 24px;
}

/* Flexbox handles content inside each card */
.card {
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}
```

---

## Common Interview Questions

**Q: When would you use `flex-wrap` and what's the Grid equivalent?**
`flex-wrap: wrap` lets flex items wrap to new rows when they don't fit. Grid `auto-fill` / `auto-fit` does similar but with proper 2D alignment — items align across rows because Grid knows about the row structure.

**Q: What's `fr` in Grid?**
Fractional unit. After fixed and content sizes are satisfied, `fr` distributes remaining space. `1fr 2fr` = first column gets 1/3, second gets 2/3 of remaining space.

**Q: Can you make a responsive grid without media queries?**
Yes: `grid-template-columns: repeat(auto-fill, minmax(200px, 1fr))` creates as many columns as fit, each at least 200px wide.
