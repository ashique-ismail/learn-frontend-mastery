# CSS Custom Properties (CSS Variables)

## Overview

CSS Custom Properties (commonly called "CSS Variables") allow you to define reusable values in CSS that can be referenced throughout your stylesheet. Unlike preprocessor variables (Sass, Less), CSS variables are **live** — they can be modified at runtime via JavaScript and respond to the cascade.

## Basic Syntax

### Declaration

```css
:root {
  --primary-color: #3498db;
  --spacing-unit: 8px;
  --font-stack: 'Helvetica Neue', Arial, sans-serif;
}
```

**Convention**: Prefix with `--` (required by spec).

### Usage

```css
.button {
  background-color: var(--primary-color);
  padding: var(--spacing-unit);
  font-family: var(--font-stack);
}
```

## Scope and Inheritance

### Global Scope (`:root`)

```css
:root {
  --color: blue;
}
```

Available to all elements.

### Local Scope

```css
.card {
  --card-padding: 20px;
}

.card__body {
  padding: var(--card-padding); /* Inherited from .card */
}
```

Variables are **inherited** by descendant elements.

### Scoping Example

```css
:root {
  --color: red;
}

.section-blue {
  --color: blue; /* Overrides for this section and descendants */
}

p {
  color: var(--color); /* red by default, blue inside .section-blue */
}
```

## Fallback Values

```css
.element {
  /* If --primary-color is not defined, use #333 */
  color: var(--primary-color, #333);
  
  /* Nested fallbacks */
  color: var(--primary-color, var(--fallback-color, black));
}
```

## Invalid at Computed-Value Time (IACVT)

If a custom property's value is invalid for a specific property, it becomes **invalid at computed-value time**:

```css
:root {
  --color: 20px; /* Valid custom property (any value allowed) */
}

p {
  color: var(--color); /* IACVT! 20px is not valid for color */
  /* Result: color becomes 'unset', not the fallback */
}
```

**Key point**: The fallback is only used if the variable is **undefined**, not if it's invalid.

## Use Cases

### Design Tokens

```css
:root {
  /* Colors */
  --color-primary: #3498db;
  --color-secondary: #2ecc71;
  --color-danger: #e74c3c;
  --color-text: #333;
  --color-text-muted: #666;
  
  /* Spacing */
  --space-xs: 4px;
  --space-sm: 8px;
  --space-md: 16px;
  --space-lg: 24px;
  --space-xl: 48px;
  
  /* Typography */
  --font-size-sm: 0.875rem;
  --font-size-base: 1rem;
  --font-size-lg: 1.25rem;
  --font-size-xl: 1.5rem;
  
  /* Shadows */
  --shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.1);
  --shadow-md: 0 4px 6px rgba(0, 0, 0, 0.1);
  --shadow-lg: 0 10px 15px rgba(0, 0, 0, 0.1);
  
  /* Border radius */
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 16px;
  --radius-full: 9999px;
  
  /* Transitions */
  --transition-fast: 150ms ease;
  --transition-base: 250ms ease;
  --transition-slow: 500ms ease;
}
```

### Theming

```css
:root {
  --bg-primary: white;
  --bg-secondary: #f5f5f5;
  --text-primary: #333;
  --text-secondary: #666;
}

[data-theme="dark"] {
  --bg-primary: #1a1a1a;
  --bg-secondary: #2a2a2a;
  --text-primary: #f5f5f5;
  --text-secondary: #ccc;
}

body {
  background: var(--bg-primary);
  color: var(--text-primary);
}
```

### Component Variants

```css
.button {
  --button-bg: var(--color-primary);
  --button-text: white;
  --button-padding: var(--space-md);
  
  background: var(--button-bg);
  color: var(--button-text);
  padding: var(--button-padding);
}

.button--large {
  --button-padding: var(--space-lg);
}

.button--danger {
  --button-bg: var(--color-danger);
}
```

### Responsive Values

```css
:root {
  --container-width: 1200px;
}

@media (max-width: 1024px) {
  :root {
    --container-width: 960px;
  }
}

@media (max-width: 768px) {
  :root {
    --container-width: 100%;
  }
}

.container {
  max-width: var(--container-width);
}
```

### Calculations

```css
:root {
  --base-spacing: 8px;
}

.element {
  margin: calc(var(--base-spacing) * 2); /* 16px */
  padding: calc(var(--base-spacing) / 2); /* 4px */
}
```

## JavaScript Integration

### Reading Variables

```javascript
// Get computed value
const root = document.documentElement;
const primaryColor = getComputedStyle(root).getPropertyValue('--primary-color');
console.log(primaryColor); // "#3498db"

// From specific element
const card = document.querySelector('.card');
const padding = getComputedStyle(card).getPropertyValue('--card-padding');
```

### Setting Variables

```javascript
// Set on root
document.documentElement.style.setProperty('--primary-color', '#e74c3c');

// Set on specific element
element.style.setProperty('--local-var', '20px');
```

### Dynamic Theming

```javascript
function setTheme(theme) {
  if (theme === 'dark') {
    document.documentElement.setAttribute('data-theme', 'dark');
  } else {
    document.documentElement.removeAttribute('data-theme');
  }
}

// Toggle theme
button.addEventListener('click', () => {
  const current = document.documentElement.getAttribute('data-theme');
  setTheme(current === 'dark' ? 'light' : 'dark');
});
```

### Animating with Variables

```javascript
// Animate a custom property
element.style.setProperty('--progress', '0%');

function animate() {
  let progress = 0;
  const interval = setInterval(() => {
    progress += 1;
    element.style.setProperty('--progress', `${progress}%`);
    if (progress >= 100) clearInterval(interval);
  }, 10);
}
```

```css
.progress-bar {
  width: var(--progress, 0%);
  transition: width 0.3s ease;
}
```

## Advanced Patterns

### Computed Properties

```css
:root {
  --color-h: 200;
  --color-s: 50%;
  --color-l: 50%;
}

.element {
  background: hsl(var(--color-h), var(--color-s), var(--color-l));
}

.element:hover {
  --color-l: 60%; /* Lighter on hover */
}
```

### Conditional Values with `@supports`

```css
.element {
  --size: 100px;
}

@supports (aspect-ratio: 1) {
  .element {
    --size: auto;
  }
}

.element {
  width: var(--size);
}
```

### Spacing Scale

```css
:root {
  --space-unit: 0.5rem;
  --space-1: calc(var(--space-unit) * 1);
  --space-2: calc(var(--space-unit) * 2);
  --space-3: calc(var(--space-unit) * 3);
  --space-4: calc(var(--space-unit) * 4);
  --space-6: calc(var(--space-unit) * 6);
  --space-8: calc(var(--space-unit) * 8);
  --space-12: calc(var(--space-unit) * 12);
  --space-16: calc(var(--space-unit) * 16);
}
```

### Type Scale

```css
:root {
  --font-size-base: 16px;
  --scale-ratio: 1.25;
  
  --font-size-sm: calc(var(--font-size-base) / var(--scale-ratio));
  --font-size-md: var(--font-size-base);
  --font-size-lg: calc(var(--font-size-base) * var(--scale-ratio));
  --font-size-xl: calc(var(--font-size-base) * var(--scale-ratio) * var(--scale-ratio));
}
```

## Custom Properties in `@property` (Houdini)

Allows defining the **type** and **initial value** of custom properties:

```css
@property --gradient-angle {
  syntax: '<angle>';
  initial-value: 0deg;
  inherits: false;
}

.element {
  --gradient-angle: 45deg;
  background: linear-gradient(var(--gradient-angle), blue, red);
  transition: --gradient-angle 0.5s; /* Now animatable! */
}

.element:hover {
  --gradient-angle: 180deg;
}
```

**Benefits**:
- Type checking
- Enables animation/transition of custom properties
- Better performance

**Browser support**: Modern browsers (2023+).

## Limitations and Gotchas

### 1. Cannot be Used in Media Queries

```css
/* ❌ Does not work */
:root {
  --breakpoint: 768px;
}

@media (max-width: var(--breakpoint)) { /* Invalid */ }
```

### 2. Cannot be Used in Selectors

```css
/* ❌ Does not work */
:root {
  --state: 'hover';
}

.button:var(--state) { /* Invalid */ }
```

### 3. Case Sensitive

```css
:root {
  --Color: red;
  --color: blue;
}

p {
  color: var(--Color); /* red */
  background: var(--color); /* blue */
}
```

### 4. No Units on Numbers

```css
:root {
  --spacing: 16; /* Just the number */
}

.element {
  /* ❌ Doesn't work */
  padding: var(--spacing)px;
  
  /* ✅ Use calc() */
  padding: calc(var(--spacing) * 1px);
}
```

### 5. Whitespace Matters

```css
:root {
  --space: 10 px; /* "10 px" with space — likely invalid */
}
```

## Performance Considerations

1. **Inheritance is fast**: Changing a variable on an ancestor efficiently updates descendants
2. **Avoid excessive nesting**: Deep inheritance chains can slow style recalculation
3. **Use sparingly in animations**: Animating custom properties can be less performant than animating native properties
4. **`@property` helps**: Defined types/initial values improve performance

## Naming Conventions

### Semantic Names

```css
:root {
  --color-primary: #3498db;
  --color-danger: #e74c3c;
  --color-success: #2ecc71;
}
```

### Generic Names (Avoid)

```css
/* ❌ Not semantic */
:root {
  --color-1: #3498db;
  --blue: #3498db;
}
```

### Component-Scoped

```css
.card {
  --card-bg: white;
  --card-border: #ddd;
  --card-padding: 1rem;
}
```

### BEM-style Naming

```css
.button {
  --button__color: var(--color-primary);
  --button__padding: var(--space-md);
}
```

## Browser Support

Excellent support in all modern browsers (IE11 no support).

### Fallbacks

```css
.element {
  color: #3498db; /* Fallback for old browsers */
  color: var(--primary-color, #3498db);
}
```

---

## Supplementary: CSS Anchor Positioning

CSS Anchor Positioning (`anchor-name` / `position-anchor`) lets you position any element relative to another element (the "anchor") without JavaScript — solving the classic tooltip/popover/dropdown positioning problem natively.

```css
/* Step 1: name the anchor element */
.trigger-button {
  anchor-name: --my-button;
}

/* Step 2: position an element relative to that anchor */
.tooltip {
  position: absolute;         /* must be absolutely positioned */
  position-anchor: --my-button;

  /* inset-area: shorthand for positioning around the anchor */
  inset-area: top;            /* center above the anchor */

  /* or use anchor() for precise control */
  left:   anchor(left);       /* align left edges */
  bottom: anchor(top);        /* place directly above */
  margin-bottom: 8px;
}
```

### `anchor()` Function

```css
.dropdown {
  position: absolute;
  position-anchor: --trigger;

  /* Align right edge of dropdown to right edge of trigger */
  right:  anchor(right);
  /* Place top of dropdown at bottom of trigger */
  top:    anchor(bottom);

  /* Fallback if it would overflow viewport */
  position-try-fallbacks: --flip-above;
}

@position-try --flip-above {
  top:    auto;
  bottom: anchor(top); /* flip to above trigger when no space below */
}
```

### `inset-area` Shorthand

```css
/* Grid of positions around the anchor: */
.tooltip { inset-area: top center; }    /* above, centered */
.tooltip { inset-area: bottom left; }   /* below, left-aligned */
.tooltip { inset-area: right; }         /* to the right */
.tooltip { inset-area: top span-left; } /* above, spanning left side */
```

### JavaScript No Longer Needed For

- Tooltip positioning (above/below based on viewport space)
- Dropdown menus aligned to triggers
- Popovers that follow their anchor element during scroll
- Context menus at pointer position

### Browser Support
Chrome 125+, Edge 125+. Not yet in Firefox or Safari (2024). Use `@supports (anchor-name: --x)` to feature-detect. Libraries like Floating UI remain the cross-browser solution for now.

---

## Interview Questions to Master

1. What's the difference between CSS variables and preprocessor (Sass/Less) variables?
2. How do CSS custom properties inherit?
3. When is a fallback value used vs. when does a property become invalid?
4. Can you use CSS variables in media queries?
5. How can you modify CSS variables with JavaScript?
6. What is `@property` and what problem does it solve?
7. How would you implement a dark mode using CSS variables?

## Resources

- [MDN CSS Custom Properties](https://developer.mozilla.org/en-US/docs/Web/CSS/--*)
- [MDN Using CSS Custom Properties](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_custom_properties)
- [CSS Tricks: A Complete Guide to Custom Properties](https://css-tricks.com/a-complete-guide-to-custom-properties/)
