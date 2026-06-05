# CSS Nesting

## The Idea

**In plain English:** CSS nesting lets you write styles for elements inside other elements by placing them inside the same block of code, rather than writing separate rules for each one. Think of it as grouping related instructions together so you can read them all in one place.

**Real-world analogy:** Imagine a filing cabinet for a school. The cabinet has a drawer labeled "Science Class", and inside that drawer are folders for "Notes", "Homework", and "Tests" — all belonging to Science Class. You do not scatter those folders randomly across the whole room.

- The filing cabinet drawer labeled "Science Class" = the parent CSS selector (e.g., `.card`)
- The folders inside the drawer = the nested child selectors (e.g., `.card-title`, `.card-footer`)
- Putting folders inside the correct drawer = nesting child rules inside the parent rule in your CSS

---

## Overview

Native CSS nesting allows you to nest selectors inside other selectors, similar to preprocessors like Sass, but with native browser support. This improves code organization, reduces repetition, and maintains context in your stylesheets.

Unlike preprocessor nesting, native CSS nesting follows stricter rules to maintain performance and avoid parsing ambiguities.

## Basic Syntax

```css
/* Before: repetitive selectors */
.card { }
.card .card-title { }
.card .card-content { }
.card .card-footer { }

/* After: nested structure */
.card {
  background: white;
  
  .card-title {
    font-size: 1.5rem;
  }
  
  .card-content {
    padding: 1rem;
  }
  
  .card-footer {
    border-top: 1px solid #ddd;
  }
}
```

## The Nesting Selector (`&`)

The `&` represents the parent selector:

```css
.button {
  background: blue;
  color: white;
  
  /* & represents .button */
  &:hover {
    background: darkblue;
  }
  
  &:active {
    background: navy;
  }
  
  &.button-large {
    font-size: 1.2rem;
  }
  
  &.button-small {
    font-size: 0.875rem;
  }
}

/* Compiles to: */
/* .button { background: blue; color: white; } */
/* .button:hover { background: darkblue; } */
/* .button:active { background: navy; } */
/* .button.button-large { font-size: 1.2rem; } */
/* .button.button-small { font-size: 0.875rem; } */
```

## Direct Nesting Rules

### Rule 1: Type selectors must use `&`

```css
/* INVALID: type selector without & */
.card {
  p {
    color: gray;
  }
}

/* VALID: type selector with & */
.card {
  & p {
    color: gray;
  }
}

/* VALID: class selectors can omit & */
.card {
  .card-title {
    font-size: 1.5rem;
  }
}
```

### Rule 2: Compound selectors need `&`

```css
/* INVALID: looks like a descendant selector */
.button {
  :hover {
    /* Parser confusion: is this .button :hover or .button:hover? */
  }
}

/* VALID: explicit with & */
.button {
  &:hover {
    background: darkblue;
  }
}
```

### Rule 3: Classes/IDs can directly nest

```css
.nav {
  /* These are valid without & */
  .nav-item { }
  #logo { }
  [aria-expanded] { }
  
  /* But & is more explicit and clear */
  & .nav-item { }
  & #logo { }
  & [aria-expanded] { }
}
```

## Nesting Depth

```css
/* Avoid deep nesting (3+ levels) */
.page {
  .section {
    .card {
      .card-header {
        .card-title {
          /* Too deep! Specificity issues */
        }
      }
    }
  }
}

/* BETTER: flatter structure */
.page {
  /* Page-level styles */
}

.section {
  /* Section-level styles */
}

.card {
  /* Card styles */
  
  .card-header {
    /* Immediate children only */
  }
  
  .card-title {
    /* Keep it shallow */
  }
}
```

## Pseudo-classes and Pseudo-elements

```css
.link {
  color: blue;
  text-decoration: none;
  
  /* Pseudo-classes */
  &:hover {
    color: darkblue;
    text-decoration: underline;
  }
  
  &:focus {
    outline: 2px solid blue;
    outline-offset: 2px;
  }
  
  &:active {
    color: navy;
  }
  
  &:visited {
    color: purple;
  }
  
  /* Pseudo-elements */
  &::before {
    content: '→ ';
  }
  
  &::after {
    content: ' ←';
  }
}
```

## Modifier Patterns

```css
.button {
  padding: 0.5rem 1rem;
  background: blue;
  color: white;
  
  /* Variants with & */
  &.button-primary {
    background: var(--primary);
  }
  
  &.button-secondary {
    background: var(--secondary);
  }
  
  /* States */
  &[disabled] {
    opacity: 0.5;
    cursor: not-allowed;
  }
  
  &[aria-pressed="true"] {
    background: var(--pressed);
  }
}
```

## Nested Media Queries

```css
.sidebar {
  width: 300px;
  
  /* Media query inside selector */
  @media (max-width: 768px) {
    width: 100%;
    order: 2;
  }
  
  @media (min-width: 1200px) {
    width: 350px;
  }
  
  /* Nested elements with their own media queries */
  .sidebar-nav {
    display: flex;
    
    @media (max-width: 768px) {
      flex-direction: column;
    }
  }
}
```

## Nested Container Queries

```css
.card-wrapper {
  container-type: inline-size;
  
  .card {
    display: flex;
    flex-direction: column;
    
    /* Container query inside nested selector */
    @container (min-width: 400px) {
      flex-direction: row;
      
      .card-image {
        width: 40%;
      }
    }
    
    @container (min-width: 600px) {
      .card-title {
        font-size: 2rem;
      }
    }
  }
}
```

## Nested @supports

```css
.grid {
  display: block;
  
  /* Feature query inside selector */
  @supports (display: grid) {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
    gap: 1rem;
  }
  
  @supports (container-type: inline-size) {
    container-type: inline-size;
    
    @container (min-width: 800px) {
      gap: 2rem;
    }
  }
}
```

## Nested Cascade Layers

```css
@layer components {
  .button {
    padding: 0.5rem 1rem;
    
    /* Nested layer inside selector */
    @layer variants {
      &.button-primary {
        background: blue;
      }
    }
    
    @layer states {
      &:hover {
        opacity: 0.9;
      }
    }
  }
}
```

## Contextual Styling

```css
.theme {
  /* Light theme by default */
  --bg: white;
  --text: black;
  
  /* Dark theme variant */
  &.theme-dark {
    --bg: black;
    --text: white;
  }
  
  /* Components adapt to theme context */
  .card {
    background: var(--bg);
    color: var(--text);
    
    &:hover {
      opacity: 0.9;
    }
  }
}
```

## Differences from Sass Nesting

| Feature | Sass | Native CSS |
| ------- | ---- | ---------- |
| Type selector nesting | `p { }` works | Must use `& p { }` |
| Parent reference | `&` is optional for classes | `&` required for pseudo-classes |
| Interpolation | `#{$var}` supported | Not supported |
| Multiple `&` | Can repeat in selector | Can repeat in selector |
| `&` placement | Can be anywhere | Can be anywhere |
| Property nesting | Supported | Not supported |

### Sass Property Nesting (Not in Native CSS)

```scss
/* Sass only - NOT VALID in native CSS */
.box {
  margin: {
    top: 1rem;
    bottom: 1rem;
  }
  
  border: {
    style: solid;
    width: 1px;
    color: black;
  }
}

/* Native CSS requires full property names */
.box {
  margin-top: 1rem;
  margin-bottom: 1rem;
  border-style: solid;
  border-width: 1px;
  border-color: black;
}
```

## BEM with Nesting

```css
/* BEM naming still works well with nesting */
.card {
  /* Block */
  background: white;
  
  /* Elements */
  &__header {
    border-bottom: 1px solid #ddd;
  }
  
  &__title {
    font-size: 1.5rem;
  }
  
  &__content {
    padding: 1rem;
  }
  
  /* Modifiers */
  &--featured {
    border: 2px solid gold;
  }
  
  &--compact {
    .card__content {
      padding: 0.5rem;
    }
  }
}

/* Compiles to: */
/* .card { } */
/* .card__header { } */
/* .card__title { } */
/* .card__content { } */
/* .card--featured { } */
/* .card--compact .card__content { } */
```

## Advanced `&` Usage

### Multiple Parent References

```css
.button {
  /* & can appear multiple times */
  .theme-dark & {
    /* Compiles to: .theme-dark .button */
    border-color: white;
  }
  
  .sidebar &,
  .modal & {
    /* Compiles to: .sidebar .button, .modal .button */
    width: 100%;
  }
}
```

### Concatenation

```css
.icon {
  &-small {
    /* Compiles to: .icon-small */
    width: 16px;
  }
  
  &-medium {
    /* Compiles to: .icon-medium */
    width: 24px;
  }
  
  &-large {
    /* Compiles to: .icon-large */
    width: 32px;
  }
}
```

### Complex Selectors

```css
.article {
  & > .article-header {
    /* Direct child */
  }
  
  & + .article {
    /* Adjacent sibling */
    margin-top: 2rem;
  }
  
  & ~ .article {
    /* General sibling */
  }
  
  .sidebar & {
    /* Descendant of .sidebar */
    font-size: 0.875rem;
  }
}
```

## Practical Examples

### Navigation Menu

```css
.nav {
  display: flex;
  gap: 1rem;
  
  .nav-item {
    padding: 0.5rem 1rem;
    
    &:hover {
      background: #f5f5f5;
    }
    
    &.active {
      background: blue;
      color: white;
    }
    
    @media (max-width: 768px) {
      width: 100%;
      
      &:hover {
        background: #e5e5e5;
      }
    }
  }
  
  @media (max-width: 768px) {
    flex-direction: column;
    
    &.expanded {
      display: flex;
    }
    
    &:not(.expanded) {
      display: none;
    }
  }
}
```

### Form Components

```css
.form-field {
  margin-bottom: 1rem;
  
  .form-label {
    display: block;
    margin-bottom: 0.5rem;
    font-weight: 600;
    
    &[required]::after {
      content: ' *';
      color: red;
    }
  }
  
  .form-input {
    width: 100%;
    padding: 0.5rem;
    border: 1px solid #ccc;
    
    &:focus {
      outline: 2px solid blue;
      outline-offset: 2px;
      border-color: blue;
    }
    
    &:invalid {
      border-color: red;
    }
    
    &[disabled] {
      background: #f5f5f5;
      cursor: not-allowed;
    }
  }
  
  .form-error {
    color: red;
    font-size: 0.875rem;
    margin-top: 0.25rem;
    display: none;
    
    .form-field.has-error & {
      display: block;
    }
  }
}
```

### Card Component

```css
.card {
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  
  .card-image {
    border-radius: 8px 8px 0 0;
    aspect-ratio: 16 / 9;
    object-fit: cover;
  }
  
  .card-content {
    padding: 1rem;
  }
  
  .card-title {
    font-size: 1.5rem;
    margin-bottom: 0.5rem;
    
    a {
      color: inherit;
      text-decoration: none;
      
      &:hover {
        color: blue;
      }
    }
  }
  
  .card-footer {
    padding: 1rem;
    border-top: 1px solid #eee;
    display: flex;
    justify-content: space-between;
  }
  
  /* Variants */
  &.card-horizontal {
    display: flex;
    
    .card-image {
      border-radius: 8px 0 0 8px;
      width: 40%;
    }
    
    .card-content {
      flex: 1;
    }
  }
  
  &.card-featured {
    border: 2px solid gold;
    
    .card-title {
      color: gold;
    }
  }
  
  /* States */
  &:hover {
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.15);
  }
  
  /* Responsive */
  @media (max-width: 768px) {
    &.card-horizontal {
      flex-direction: column;
      
      .card-image {
        border-radius: 8px 8px 0 0;
        width: 100%;
      }
    }
  }
}
```

## Performance Considerations

```css
/* Nesting creates descendant selectors */
.parent {
  .child {
    /* Compiles to: .parent .child */
    /* Browser matches from right to left */
    /* Finds all .child, checks if ancestor is .parent */
  }
}

/* Sometimes direct child is more performant */
.parent {
  & > .child {
    /* Compiles to: .parent > .child */
    /* Only checks immediate parent */
  }
}

/* Shallow nesting is better for performance */
/* Keep specificity low */
```

## Current Limitations

1. **No property nesting** (unlike Sass)
2. **Type selectors must use `&`**
3. **Media queries always wrap the selector** (can't merge multiple)
4. **No selector interpolation**
5. **Browser support** (check Can I Use)

## Browser Support

```css
/* Feature detection */
@supports (selector(&)) {
  /* Native nesting supported */
  .card {
    .card-title {
      font-size: 1.5rem;
    }
  }
}

@supports not (selector(&)) {
  /* Fallback: write selectors flat */
  .card-title {
    font-size: 1.5rem;
  }
}
```

## Migration from Sass

```scss
/* Sass */
.component {
  margin: {
    top: 1rem;
    bottom: 1rem;
  }
  
  p {
    color: gray;
  }
  
  &:hover {
    opacity: 0.9;
  }
}
```

```css
/* Native CSS equivalent */
.component {
  margin-top: 1rem;      /* No property nesting */
  margin-bottom: 1rem;
  
  & p {                  /* Must use & with type selector */
    color: gray;
  }
  
  &:hover {             /* Same as Sass */
    opacity: 0.9;
  }
}
```

## Best Practices

### 1. Keep Nesting Shallow

```css
/* BAD: too deep */
.page .section .card .header .title {
  /* High specificity, hard to override */
}

/* GOOD: 2-3 levels max */
.card {
  .card-header {
    .card-title {
      /* Reasonable depth */
    }
  }
}

/* BETTER: BEM or flat structure */
.card__title {
  /* Low specificity, easy to override */
}
```

### 2. Use `&` for Clarity

```css
/* Less clear */
.button {
  .button-icon {
    /* Is this descendant or modifier? */
  }
}

/* More clear */
.button {
  & .button-icon {
    /* Obviously a descendant */
  }
  
  &-icon {
    /* Obviously a modifier/variant */
  }
}
```

### 3. Group Related Styles

```css
.button {
  /* Base styles */
  padding: 0.5rem 1rem;
  
  /* Pseudo-classes */
  &:hover { }
  &:focus { }
  &:active { }
  
  /* Modifiers */
  &.primary { }
  &.secondary { }
  
  /* States */
  &[disabled] { }
  &[aria-pressed] { }
  
  /* Responsive */
  @media (max-width: 768px) { }
}
```

## Interview Questions

**Q1: What's the main difference between native CSS nesting and Sass nesting?**

**A:** Native CSS requires `&` before type selectors and pseudo-classes for clarity (e.g., `& p { }`, `&:hover { }`), while Sass allowed direct nesting. Native CSS doesn't support property nesting (`margin: { top: 1rem; }`). These restrictions prevent parser ambiguities and ensure performance.

**Q2: Why must type selectors use `&` in native CSS nesting?**

**A:** Without `&`, type selectors create parser ambiguities. `p { color: red; }` inside `.card { }` could mean either `.card p` (descendant) or a separate `p` rule. Requiring `& p { }` makes the intent explicit and prevents parsing confusion.

**Q3: How does nesting affect specificity?**

**A:** Nesting generates descendant selectors, increasing specificity. `.card { .title { } }` compiles to `.card .title` (specificity: 0,2,0). Deep nesting creates high specificity that's hard to override. Keep nesting shallow to maintain low specificity.

**Q4: Can media queries be nested inside selectors?**

**A:** Yes, and this is a major benefit. You can nest `@media`, `@container`, `@supports`, and `@layer` inside selectors, keeping responsive logic colocated with the component. The compiled output wraps the selector in the media query.

**Q5: How does `&` work in complex scenarios?**

**A:** `&` represents the complete parent selector. It can:

- Appear multiple times: `.theme &` → `.theme .button`
- Concatenate: `&-large` → `.button-large`
- Combine with selectors: `& > .child` → `.button > .child`
- Reference ancestors: `.parent &` → `.parent .button`

**Q6: When should you avoid nesting?**

**A:** Avoid nesting when:

- It creates more than 3 levels of depth
- Specificity becomes problematic
- Selectors aren't logically related
- You're trying to replicate flat selector lists
- Performance is critical (high selector overhead)

Use flat selectors with naming conventions (BEM) as an alternative.

## Resources

- [MDN: CSS Nesting](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_nesting)
- [CSS Nesting Module Spec](https://www.w3.org/TR/css-nesting-1/)
- [Chrome Developers: CSS Nesting](https://developer.chrome.com/articles/css-nesting/)
- [Ahmad Shadeed: CSS Nesting](https://ishadeed.com/article/css-nesting/)
- [Can I Use: CSS Nesting](https://caniuse.com/css-nesting)

## Summary

Native CSS nesting improves code organization by allowing selectors to nest inside others, reducing repetition and maintaining context. Use `&` explicitly for type selectors and pseudo-classes, nest media queries for colocated responsive logic, and keep nesting shallow (2-3 levels max) to avoid specificity issues. While similar to Sass nesting, native CSS has stricter rules to prevent parser ambiguities. The result is more maintainable stylesheets without the need for preprocessors.
