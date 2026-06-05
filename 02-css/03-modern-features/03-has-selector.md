# CSS `:has()` Selector (Parent Selector)

## The Idea

**In plain English:** The `:has()` selector is a way to style a container in CSS based on what is inside it — for example, making a box look different if it happens to contain a photo. CSS normally can only style things based on their parents or what comes after them, but `:has()` flips that around so a parent can "notice" its children and change its own appearance.

**Real-world analogy:** Imagine a backpack that automatically changes color depending on what you put inside it — if you pack a lunchbox, the backpack turns blue; if there is no lunchbox, it stays grey.

- The backpack = the parent element (e.g., `.card`)
- What is inside the backpack (the lunchbox) = the child element being checked (e.g., `img`)
- The color change = the CSS styles applied when the condition is true

---

## Overview

The `:has()` pseudo-class (called the "parent selector" or "relational selector") allows you to style an element based on its **descendants** or **siblings**. This was previously impossible in CSS without JavaScript.

**Browser Support**: Modern browsers (2023+).

## Basic Syntax

```css
parent:has(child) {
  /* Style parent if it contains child */
}
```

## Selecting Based on Descendants

### Has Direct Child

```css
/* Style article if it contains an image */
article:has(img) {
  display: grid;
  grid-template-columns: 1fr 300px;
}

/* Without image, defaults to single column */
article {
  display: block;
}
```

### Has Specific Descendant

```css
/* Card with a .featured class inside */
.card:has(.featured) {
  border: 2px solid gold;
  box-shadow: 0 4px 12px rgba(255, 215, 0, 0.3);
}
```

### Multiple Conditions

```css
/* Form with invalid input */
form:has(input:invalid) {
  border-color: red;
}

/* Form with all valid inputs */
form:has(input):not(:has(input:invalid)) {
  border-color: green;
}
```

## Selecting Based on Siblings

### Adjacent Sibling

```css
/* Label before a focused input */
input:focus + label { /* Old way (sibling selector) */
  color: blue;
}

/* Label after a focused input (NEW!) */
label:has(+ input:focus) {
  color: blue;
}
```

### General Sibling

```css
/* Section that comes before a .highlighted section */
section:has(~ .highlighted) {
  opacity: 0.5;
}
```

## Practical Use Cases

### 1. Card with Optional Image

```css
.card {
  display: flex;
  flex-direction: column;
}

.card:has(img) {
  flex-direction: row;
  gap: 1rem;
}

.card:has(img) img {
  width: 200px;
  object-fit: cover;
}
```

### 2. Form Validation Feedback

```css
/* Show error icon next to invalid input */
.form-group:has(input:invalid) .error-icon {
  display: block;
}

/* Green border for valid groups */
.form-group:has(input:valid) {
  border-color: green;
}

/* Highlight form with errors */
form:has(.error) {
  background: #fff3cd;
  padding: 1rem;
}
```

### 3. Interactive Parent on Child Hover

```css
/* Card changes when image inside is hovered */
.card:has(img:hover) {
  box-shadow: 0 8px 16px rgba(0,0,0,0.2);
  transform: translateY(-4px);
  transition: all 0.3s;
}
```

### 4. Accordion State

```css
.accordion-item:has(input:checked) .content {
  max-height: 500px;
}

.accordion-item:has(input:checked) .icon {
  transform: rotate(180deg);
}
```

### 5. Table Row with Checkbox

```css
tr:has(input[type="checkbox"]:checked) {
  background: #e3f2fd;
}

/* "Select All" behavior styling */
table:has(thead input:checked) tbody tr {
  background: #f5f5f5;
}
```

### 6. Navigation with Active Section

```css
/* Highlight nav item when its section is in view */
nav:has(#section1:target) a[href="#section1"],
nav:has(#section2:target) a[href="#section2"] {
  color: blue;
  font-weight: bold;
}
```

### 7. Empty State Handling

```css
/* Show message if list is empty */
.list:not(:has(li))::after {
  content: "No items found";
  display: block;
  padding: 2rem;
  text-align: center;
  color: #666;
}
```

### 8. Quantity-Based Styling

```css
/* If list has 10+ items, use smaller font */
ul:has(> li:nth-child(10)) li {
  font-size: 0.9rem;
}

/* If only one item, center it */
ul:has(> li:only-child) {
  justify-content: center;
}
```

## Advanced Patterns

### Negation with `:has()`

```css
/* Article WITHOUT an image */
article:not(:has(img)) {
  max-width: 65ch;
}

/* Section without heading */
section:not(:has(h2)) {
  padding-top: 0;
}
```

### Combining with Other Selectors

```css
/* Header with navigation AND search */
header:has(nav):has(.search) {
  display: grid;
  grid-template-columns: auto 1fr auto;
}

/* Paragraph immediately after heading */
p:has(+ h2) {
  margin-bottom: 2rem;
}
```

### State Management

```css
/* Body with modal open */
body:has(.modal[open]) {
  overflow: hidden;
}

/* Page with sidebar expanded */
.page:has(.sidebar.expanded) .main-content {
  margin-left: 300px;
}
```

### Conditional Layouts

```css
.container {
  display: grid;
  grid-template-columns: 1fr;
}

/* Two columns if aside exists */
.container:has(aside) {
  grid-template-columns: 1fr 300px;
}

/* Three columns if both sidebar and aside */
.container:has(.sidebar):has(aside) {
  grid-template-columns: 250px 1fr 300px;
}
```

## `:has()` vs JavaScript

### Before `:has()` (JavaScript)

```javascript
const cards = document.querySelectorAll('.card');
cards.forEach(card => {
  if (card.querySelector('img')) {
    card.classList.add('has-image');
  }
});
```

```css
.card.has-image {
  flex-direction: row;
}
```

### With `:has()` (Pure CSS)

```css
.card:has(img) {
  flex-direction: row;
}
```

**Benefits**:
- No JavaScript required
- Automatically reactive (no mutation observers needed)
- Better performance
- Cleaner code

## Performance Considerations

`:has()` can be expensive because it requires checking descendants/siblings:

### Optimize Scope

```css
/* ❌ Expensive: checks entire document */
:has(.active) {
  /* ... */
}

/* ✅ Better: scoped to specific container */
.nav:has(.active) {
  /* ... */
}
```

### Limit Depth

```css
/* ❌ Deep nesting */
body:has(main:has(article:has(section:has(.widget)))) { }

/* ✅ Flatter */
body:has(.widget) { }
```

### Use Specific Selectors

```css
/* ❌ Generic */
div:has(div) { }

/* ✅ Specific */
.card:has(.card__image) { }
```

## Combining with Container Queries

```css
.card {
  container-type: inline-size;
}

.card:has(img) {
  display: grid;
  grid-template-columns: 1fr 1fr;
}

@container (max-width: 400px) {
  .card:has(img) {
    grid-template-columns: 1fr;
  }
}
```

## Browser Support and Fallbacks

### Feature Detection

```css
/* Fallback */
.card {
  flex-direction: column;
}

/* Progressive enhancement */
@supports selector(:has(*)) {
  .card:has(img) {
    flex-direction: row;
  }
}
```

### JavaScript Polyfill

For older browsers, you'll need a JavaScript fallback:

```javascript
if (!CSS.supports('selector(:has(*))')) {
  // Add classes manually
  document.querySelectorAll('.card').forEach(card => {
    if (card.querySelector('img')) {
      card.classList.add('has-image');
    }
  });
}
```

## Common Gotchas

### 1. `:has()` Cannot be Nested

```css
/* ❌ Invalid */
:has(:has(.foo)) { }
```

### 2. Works with Any Selector

```css
/* ✅ Valid complex selectors */
article:has(> section > .featured) { }
form:has([type="checkbox"]:checked) { }
```

### 3. Sibling Combinators Reverse Direction

```css
/* Traditional: target sibling AFTER */
h2 + p { } /* p after h2 */

/* With :has(): target element BEFORE */
h2:has(+ p) { } /* h2 before p */
```

## Limitations

1. **Cannot nest `:has()`**: No `:has(:has())`
2. **Cannot use in `:visited`**: Security restriction
3. **Performance impact**: Use judiciously on large DOMs
4. **No browser support for `:has(:has())`**: Can't double-nest

## Comparison: `:has()` vs Alternatives

| Feature | Before `:has()` | With `:has()` |
|---------|-----------------|---------------|
| Parent selection | Impossible (JS required) | Native CSS |
| Sibling before | Impossible | `elem:has(+ sibling)` |
| Conditional layout | JS + classes | Pure CSS |
| Reactivity | Manual (observers) | Automatic |

## Interview Questions to Master

1. What problem does `:has()` solve?
2. Can you use `:has()` to select siblings? How?
3. What are the performance implications of `:has()`?
4. How is `:has(+ sibling)` different from `+ sibling`?
5. Can you nest `:has()` selectors?
6. How would you provide a fallback for browsers without `:has()` support?
7. Give an example of using `:has()` for form validation feedback.

## Resources

- [MDN :has()](https://developer.mozilla.org/en-US/docs/Web/CSS/:has)
- [CSS Tricks: :has() Guide](https://css-tricks.com/the-css-has-selector/)
- [WebKit: :has() Examples](https://webkit.org/blog/13096/css-has-pseudo-class/)
- [Can I Use: :has()](https://caniuse.com/css-has)
