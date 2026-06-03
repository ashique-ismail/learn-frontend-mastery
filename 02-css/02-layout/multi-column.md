# CSS Multi-Column Layout

## Overview

CSS Multi-Column Layout (often called "multicol") allows content to flow into multiple columns, similar to newspaper or magazine layouts. It's specifically designed for flowing text content across columns, not for general page layout.

## Basic Syntax

### Define Number of Columns

```css
.container {
  column-count: 3; /* Exactly 3 columns */
}
```

### Define Column Width

```css
.container {
  column-width: 250px; /* Each column at least 250px wide */
}
```

**Behavior**: Browser creates as many columns as will fit given the constraint.

### Shorthand: `columns`

```css
.container {
  columns: 3; /* Same as column-count: 3 */
  columns: 250px; /* Same as column-width: 250px */
  columns: 3 250px; /* Both: 3 columns, each 250px wide */
}
```

## Column Count vs Column Width

### `column-count`
- **Fixed number** of columns
- Column width adjusts to container width
- Columns always fill available width

### `column-width`
- **Minimum** width for columns
- Number of columns adjusts to fit
- More columns created if space available

### Both Together
```css
.container {
  column-count: 4;
  column-width: 200px;
}
```

**Behavior**: Browser uses whichever creates narrower columns:
- Container 1000px wide → 4 columns × 250px = follows `column-count`
- Container 600px wide → 3 columns × 200px = follows `column-width` (can't fit 4 × 200px)

## Responsive Columns (Recommended Pattern)

```css
.text {
  column-width: 300px; /* Optimal reading width */
  column-count: 3; /* Max 3 columns */
}
```

This creates a responsive layout:
- Wide screens: 3 columns
- Medium screens: 2 columns
- Narrow screens: 1 column

No media queries needed!

## Column Gaps

```css
.container {
  columns: 3;
  column-gap: 40px; /* Space between columns */
  /* or */
  gap: 40px; /* Works in multicol too */
}
```

**Default**: `1em` gap (not `0`).

## Column Rules (Divider Lines)

```css
.container {
  columns: 3;
  column-rule: 1px solid #ccc; /* Like a vertical border between columns */
  
  /* Or individual properties */
  column-rule-width: 2px;
  column-rule-style: dotted;
  column-rule-color: blue;
}
```

**Important**: Column rules don't take up space (unlike borders). They're drawn *in* the gap.

## Spanning Columns

### `column-span: all`

Forces an element to span across all columns:

```css
.container {
  columns: 3;
}

h2 {
  column-span: all; /* Spans all 3 columns */
  margin-top: 2em;
  margin-bottom: 1em;
}
```

**Only values**: `none` (default) or `all` (no `span 2` or partial spans).

### Use Case: Subheadings

```css
article {
  columns: 2;
}

article h2 {
  column-span: all; /* Break out of columns for section headings */
  background: #f5f5f5;
  padding: 10px;
  margin: 20px 0;
}
```

## Column Breaks

Control where content breaks across columns:

### `break-before`, `break-after`, `break-inside`

```css
h2 {
  break-before: column; /* Force column break before heading */
}

h2 {
  break-after: avoid-column; /* Avoid break right after heading */
}

figure {
  break-inside: avoid-column; /* Keep figure together, don't split */
}

p {
  break-inside: avoid; /* Avoid breaking paragraphs across columns */
}
```

**Values**:
- `auto` (default)
- `avoid` / `avoid-column` – Avoid breaks
- `column` – Force a break
- `page` – For print media
- `avoid-page` – For print media

**Note**: Support varies; not all values work in all browsers.

## Column Filling

Controls how content distributes across columns:

```css
.container {
  columns: 3;
  column-fill: balance; /* Default: balance content across columns */
  column-fill: auto;    /* Fill columns sequentially (left to right) */
}
```

**`balance`**: Tries to make column heights equal.  
**`auto`**: Fills first column completely before moving to next (only works with constrained height).

### When to Use `auto`

```css
.container {
  columns: 3;
  column-fill: auto;
  height: 500px; /* Height constraint required for 'auto' to work */
}
```

## Styling Individual Columns

**Unfortunately, you cannot directly style individual columns** (no `:nth-column()` selector).

Workarounds:
1. Use `column-rule` for visual separation
2. Style content elements, not columns themselves
3. Use Grid/Flexbox if you need column-specific styling

## Practical Examples

### Newspaper Layout

```css
article {
  column-width: 300px;
  column-gap: 40px;
  column-rule: 1px solid #ddd;
  text-align: justify;
  hyphens: auto;
}

article h2 {
  column-span: all;
  font-size: 2em;
  margin: 1em 0 0.5em;
}

article img {
  max-width: 100%;
  height: auto;
  break-inside: avoid;
}
```

### Masonry-Style Layout (Lists)

```css
.list {
  columns: 4 200px; /* 4 columns, min 200px each */
  column-gap: 20px;
}

.list li {
  break-inside: avoid; /* Keep list items together */
  margin-bottom: 10px;
}
```

### FAQ Section

```css
.faq {
  columns: 2;
  column-gap: 60px;
}

.faq-item {
  break-inside: avoid; /* Don't split Q&A pairs */
  margin-bottom: 2em;
}

.faq-item h3 {
  break-after: avoid-column; /* Keep question with answer */
}
```

### Cards in Columns

```css
.cards {
  columns: 3 250px;
  column-gap: 20px;
}

.card {
  break-inside: avoid;
  display: inline-block; /* Prevents margin collapsing issues */
  width: 100%;
  margin-bottom: 20px;
}
```

## Multicol with Grid/Flexbox Children

You can combine multicol with other layout methods:

```css
.container {
  columns: 2;
}

.grid-item {
  display: grid;
  grid-template-columns: auto 1fr;
  gap: 10px;
  break-inside: avoid;
}
```

## Accessibility Concerns

### Reading Order

Multi-column layout can create a confusing reading order for screen readers:

```
Visual order:     Screen reader order:
┌────┬────┐       1 → 2 → 3 → 4 → 5 → 6
│ 1  │ 4  │       (top to bottom, left to right)
│ 2  │ 5  │
│ 3  │ 6  │
└────┴────┘
```

**Best practices**:
- Only use for continuous text content
- Avoid for UI elements or navigation
- Consider `column-span: all` for important headings

### Cognitive Load

Multiple columns can increase cognitive load:
- Harder to scan
- Requires more eye movement
- Can break reading rhythm

**Recommendation**: Use judiciously; consider UX over aesthetics.

## When to Use Multi-Column Layout

### ✅ Good Use Cases
- Long-form text articles (magazines, blogs)
- Terms and conditions / legal text
- Lists with many short items
- Testimonials
- FAQs

### ❌ Bad Use Cases
- Navigation menus (use Flexbox/Grid)
- Card layouts (use Grid)
- Forms (use Grid)
- Anything requiring column-specific styling
- Interactive content that needs a clear flow

## Multicol vs Grid: Key Differences

| Aspect | Multi-Column | Grid |
|--------|--------------|------|
| Purpose | Flowing text content | Structured layouts |
| Control | Automatic column distribution | Explicit item placement |
| Responsive | Automatically adjusts columns | Requires explicit rules |
| Column styling | Cannot style individual columns | Full control per grid area |
| Best for | Continuous text | Distinct content blocks |

## Print Styles

Multi-column layout works excellently in print:

```css
@media print {
  article {
    columns: 2;
    column-gap: 0.5in;
    orphans: 3;
    widows: 3;
  }
  
  h2 {
    break-after: avoid;
    page-break-after: avoid; /* Legacy print support */
  }
}
```

## Browser Support

Excellent support in all modern browsers, but with vendor prefixes for older versions:

```css
.container {
  -webkit-columns: 3;
  -moz-columns: 3;
  columns: 3;
  
  -webkit-column-gap: 40px;
  -moz-column-gap: 40px;
  column-gap: 40px;
}
```

**In 2023+**: Prefixes generally unnecessary for modern browsers.

## Common Gotchas

1. **Margin collapsing**: Still applies within columns but not across columns
2. **Floats**: Float elements can disrupt column layout
3. **Absolute positioning**: Positioned relative to the multicol container, not individual columns
4. **Backgrounds**: Applied to the entire container, not individual columns
5. **`column-span`**: Only `all` or `none` (no partial spans)
6. **`break-inside`**: Not universally supported; test thoroughly

## Performance Considerations

Multi-column layout is generally performant, but:
- Extremely tall containers with many columns can cause reflow issues
- Complex break rules increase layout calculation time
- Use `contain: layout` for large multicol containers to isolate reflows

## Interview Questions to Master

1. What's the difference between `column-count` and `column-width`?
2. What are the only two values for `column-span`?
3. How does `column-rule` differ from borders?
4. When would you use `column-fill: auto` instead of `balance`?
5. Why shouldn't you use multi-column layout for navigation?
6. How do column breaks work and what are their limitations?
7. What's the default `column-gap` value?

## Resources

- [MDN Multi-column Layout](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Columns)
- [CSS Tricks: Guide to Columns](https://css-tricks.com/guide-responsive-friendly-css-columns/)
- [Smashing Magazine: When to Use Columns](https://www.smashingmagazine.com/2019/01/css-multiple-column-layout-multicol/)
