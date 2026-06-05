# Cascade Layers

## The Idea

**In plain English:** Cascade layers are a way to organize your CSS styles into named groups, where you decide upfront which group wins when two groups try to style the same thing. Think of it as giving each group of styles a rank — the higher-ranked group always beats the lower one, no matter how the styles are written.

**Real-world analogy:** Imagine a school dress code ruled by three groups: the school board, teachers, and individual students. The school board sets the overarching rules, teachers can add guidelines on top, and students can choose within what's left. If a teacher says "wear a name badge" but the school board already said "no badges on Fridays," the school board wins — because their rank is higher.

- The school board = the `reset` or `base` layer (lowest priority, broadest rules)
- The teachers = the `components` layer (middle priority, more specific rules)
- The students = the `utilities` or `overrides` layer (highest priority, final say)

---

## Overview

CSS Cascade Layers (`@layer`) provide explicit control over the cascade by allowing you to organize CSS into layers with defined priority. Layers let you control specificity and source order at a higher level than traditional cascade rules.

This solves common problems with:

- Third-party CSS conflicting with custom styles
- Reset/normalize styles interfering with components
- Utility classes losing to component styles
- Managing large codebases with multiple teams

## The Problem Without Layers

```css
/* reset.css - loaded first */
button {
  border: none;
  padding: 0;
}

/* components.css - loaded second */
.button {
  padding: 0.5rem 1rem; /* Wins due to source order */
}

/* utilities.css - loaded third */
.p-0 {
  padding: 0; /* Should win but specificity is equal */
}

/* Developer expectation: utility wins */
/* Reality: components.css wins due to source order */
```

## Basic Syntax

```css
/* Define layer order (most important step!) */
@layer reset, base, components, utilities;

/* Add styles to layers */
@layer reset {
  * {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }
}

@layer base {
  body {
    font-family: system-ui;
    line-height: 1.5;
  }
}

@layer components {
  .button {
    padding: 0.5rem 1rem;
    background: blue;
  }
}

@layer utilities {
  .p-0 {
    padding: 0 !important; /* Usually needs !important without layers */
  }
}

/* With layers: utilities win regardless of specificity or source order */
```

## Layer Priority Rules

### 1. Layer Order Matters Most

```css
/* Declare layer order first */
@layer framework, components, overrides;

/* Later layers WIN over earlier layers */
/* overrides > components > framework */

@layer framework {
  .card {
    border: 1px solid gray; /* Lowest priority */
  }
}

@layer components {
  .card {
    border: 2px solid blue; /* Medium priority */
  }
}

@layer overrides {
  .card {
    border: 3px solid red; /* Highest priority - WINS */
  }
}

/* Result: 3px red border, regardless of specificity */
```

### 2. Unlayered Styles Have Higher Priority

```css
@layer base, components;

@layer base {
  p {
    color: gray;
  }
}

@layer components {
  .text {
    color: blue;
  }
}

/* Unlayered styles */
p {
  color: red; /* WINS over all layered styles */
}

/* Priority: unlayered > components > base */
```

### 3. Specificity Within Layers

```css
@layer components {
  /* Within same layer, normal specificity rules apply */
  p {
    color: blue; /* Specificity: 0,0,1 */
  }
  
  .text {
    color: red; /* Specificity: 0,1,0 - WINS within layer */
  }
}
```

## Layer Definition Patterns

### 1. Upfront Declaration (Recommended)

```css
/* Declare all layers first - establishes clear priority */
@layer reset, base, theme, layout, components, utilities, overrides;

/* Then populate them */
@layer reset {
  /* Reset styles */
}

@layer utilities {
  /* Utility classes */
}
/* Order of population doesn't matter, declaration order does */
```

### 2. Incremental Definition

```css
/* First mention creates the layer and sets priority */
@layer base {
  /* Base styles */
}

@layer components {
  /* Components */
}

/* Later additions append to existing layers */
@layer base {
  /* More base styles - added to base layer */
}
```

### 3. Anonymous Layers

```css
/* Anonymous layers - useful for isolation without naming */
@layer {
  .temp-override {
    color: red;
  }
}

/* Can't be referenced again - one-off isolation */
```

## Nested Layers

```css
/* Define nested layer structure */
@layer framework {
  @layer reset {
    * { margin: 0; }
  }
  
  @layer base {
    body { font-size: 16px; }
  }
}

/* Access nested layers with dot notation */
@layer framework.reset {
  button { border: none; }
}

@layer framework.base {
  h1 { font-size: 2rem; }
}

/* Priority: framework.base > framework.reset */
```

### Complex Nesting

```css
/* Declare nested structure upfront */
@layer framework {
  @layer reset, base, components;
}

@layer custom {
  @layer utilities, overrides;
}

/* Priority (innermost matters within parent):
   - custom.overrides (highest)
   - custom.utilities
   - framework.components
   - framework.base
   - framework.reset (lowest)
*/
```

## Importing with Layers

```css
/* Import directly into a layer */
@import url('reset.css') layer(reset);
@import url('framework.css') layer(framework);
@import url('components.css') layer(components);

/* Or create anonymous layer */
@import url('vendor.css') layer;

/* With media queries */
@import url('print.css') layer(print) print;

/* Imported layers respect declaration order */
```

## Practical Use Cases

### 1. Managing Third-Party CSS

```css
/* Establish layer order */
@layer vendor, custom;

/* Third-party framework goes in vendor layer */
@import url('bootstrap.css') layer(vendor);

/* Your custom styles in custom layer */
@layer custom {
  .button {
    /* Overrides Bootstrap without !important */
    background: var(--brand-color);
  }
}
```

### 2. Design System Architecture

```css
/* Clear layer hierarchy */
@layer 
  reset,
  tokens,
  base,
  layout,
  components,
  utilities,
  exceptions;

@layer reset {
  /* Modern CSS reset */
  *, *::before, *::after {
    box-sizing: border-box;
  }
  * { margin: 0; }
}

@layer tokens {
  /* Design tokens */
  :root {
    --color-primary: #0066cc;
    --spacing-md: 1rem;
  }
}

@layer base {
  /* Element defaults */
  body {
    font-family: system-ui;
    color: var(--color-text);
  }
}

@layer layout {
  /* Layout primitives */
  .container { max-width: 1200px; }
  .grid { display: grid; }
}

@layer components {
  /* UI components */
  .button { /* ... */ }
  .card { /* ... */ }
}

@layer utilities {
  /* Utility classes */
  .hidden { display: none; }
  .sr-only { /* ... */ }
}

@layer exceptions {
  /* One-off overrides with documentation */
  .legacy-page .button {
    /* Temporary override for legacy page */
  }
}
```

### 3. Team Collaboration

```css
/* Clear boundaries between team contributions */
@layer 
  core,
  team-design,
  team-marketing,
  team-product,
  hotfixes;

@layer core {
  @import url('design-system.css');
}

@layer team-design {
  /* Design team's additions */
}

@layer team-marketing {
  /* Marketing team's pages */
}

@layer team-product {
  /* Product team's features */
}

@layer hotfixes {
  /* Highest priority for urgent fixes */
  /* Should be empty in production */
}
```

### 4. Progressive Enhancement

```css
@layer base, enhanced, modern;

@layer base {
  /* Works everywhere */
  .card {
    border: 1px solid gray;
    padding: 1rem;
  }
}

@layer enhanced {
  /* For browsers with better support */
  @supports (display: grid) {
    .card-grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
    }
  }
}

@layer modern {
  /* Latest features */
  @supports (container-type: inline-size) {
    .card-wrapper {
      container-type: inline-size;
    }
    
    @container (min-width: 400px) {
      .card {
        display: flex;
      }
    }
  }
}
```

## !important Reversal in Layers

```css
@layer base, overrides;

@layer base {
  .text {
    color: blue !important; /* Lower priority */
  }
}

@layer overrides {
  .text {
    color: red; /* WINS even without !important */
  }
}

/* Important declarations reverse layer order! */
/* With !important: base > overrides (reversed) */
/* Without !important: overrides > base (normal) */
```

### Important Priority Order

```css
@layer A, B;

/* Priority from highest to lowest: */
/* 1. Unlayered !important */
/* 2. Layer A !important (earlier layers win) */
/* 3. Layer B !important */
/* 4. Unlayered normal */
/* 5. Layer B normal (later layers win) */
/* 6. Layer A normal */
```

## Reordering Layers

```css
/* Cannot reorder after first declaration */
@layer A, B, C;

/* This creates NEW layers D, E, not reorder */
@layer D, E;

/* Final order: A, B, C, D, E */

/* To reorder, must be in first declaration: */
/* @layer C, B, A; */
```

## Debugging Layers

```css
/* Use DevTools to inspect layer cascade */
/* Chrome/Edge: Shows layer in Styles panel */
/* Firefox: Shows layer in Rules panel */

/* Temporary debug helper */
@layer debug {
  * {
    outline: 1px solid red !important;
  }
}

/* Place debug layer at different positions to test priority */
```

## Browser Support and Fallbacks

```css
/* Feature detection */
@supports (at-rule(@layer)) {
  @layer base, components;
  
  @layer base {
    /* Layer-aware styles */
  }
}

@supports not (at-rule(@layer)) {
  /* Fallback for older browsers */
  /* Use specificity/source order */
  
  /* Higher specificity for "higher priority" styles */
  .components .button {
    /* More specific selectors */
  }
}
```

## Performance Considerations

1. **No runtime cost**: Layers are resolved at stylesheet parse time
2. **Simpler selectors**: Can use lower specificity, better for rendering
3. **Better caching**: Clear layer boundaries enable modular caching strategies

```css
/* Before layers: high specificity needed */
.page .section .card .button.primary {
  /* Specificity: 0,4,1 */
}

/* With layers: low specificity sufficient */
@layer components {
  .button-primary {
    /* Specificity: 0,1,0 but layer controls priority */
  }
}
```

## Anti-Patterns

### 1. Too Many Layers

```css
/* BAD: over-fragmentation */
@layer 
  reset,
  normalize,
  base,
  tokens,
  primitives,
  atoms,
  molecules,
  organisms,
  templates,
  pages,
  utilities,
  helpers,
  overrides,
  exceptions,
  hotfixes;

/* GOOD: essential categories */
@layer 
  reset,
  base,
  components,
  utilities,
  overrides;
```

### 2. Mixing Paradigms

```css
/* CONFUSING: layers + high specificity */
@layer components {
  .page .section .card .button.primary.large {
    /* Defeats the purpose of layers */
  }
}

/* CLEAR: layers + low specificity */
@layer components {
  .button-primary-large {
    /* Let layers handle priority */
  }
}
```

### 3. Unclear Layer Purpose

```css
/* BAD: vague naming */
@layer stuff, things, misc;

/* GOOD: clear intent */
@layer reset, base, components;
```

## Advanced Patterns

### Conditional Layer Loading

```css
/* Load layer based on context */
@layer theme-light {
  @media (prefers-color-scheme: light) {
    :root { --bg: white; }
  }
}

@layer theme-dark {
  @media (prefers-color-scheme: dark) {
    :root { --bg: black; }
  }
}
```

### Component-Scoped Layers

```css
@layer components {
  @layer buttons {
    .button { /* Base button */ }
    .button-primary { /* Primary variant */ }
  }
  
  @layer cards {
    .card { /* Base card */ }
    .card-elevated { /* Elevated variant */ }
  }
}

/* Organized hierarchy without naming conflicts */
```

## Migration Strategy

```css
/* Step 1: Wrap existing CSS in layers */
@layer legacy {
  /* All existing styles */
}

/* Step 2: Add new layer for improvements */
@layer modern {
  /* New styles with lower specificity */
}

/* Step 3: Gradually move styles from legacy to modern */
/* Step 4: Eventually remove legacy layer */
```

## Interview Questions

**Q1: How do cascade layers change CSS priority?**

**A:** Cascade layers introduce a new priority level above specificity and source order. Styles in later-declared layers win over earlier layers, regardless of specificity. Within a layer, normal cascade rules (specificity, source order) apply. Unlayered styles have higher priority than any layered styles.

**Q2: Why does `!important` reverse layer order?**

**A:** `!important` reverses layer order so that base/reset layers can be protected from accidental overrides. Important declarations in earlier layers win over later layers, ensuring foundational styles marked `!important` can't be easily overridden. This maintains the integrity of critical base styles.

**Q3: What's the difference between layered and unlayered styles?**

**A:** Unlayered styles always have higher priority than layered styles (except for `!important`, which reverses this). This allows progressive migration - existing unlayered CSS continues to work while new layered CSS is added. It also provides an "escape hatch" for exceptional overrides.

**Q4: How do nested layers work?**

**A:** Nested layers create a hierarchy. Priority is determined by the outermost layer first, then inner layers. Example: `framework.base` vs `custom.reset` - `custom` layer wins regardless of the inner layer names. Access nested layers with dot notation: `@layer framework.base { }`.

**Q5: When would you use anonymous layers?**

**A:** Anonymous layers (declared with `@layer { }` without a name) are useful for:

- One-off isolation that doesn't need to be referenced again
- Importing third-party CSS without naming it: `@import 'vendor.css' layer;`
- Temporary overrides during development
- Creating throwaway containment contexts

**Q6: How do layers interact with media queries?**

**A:** Media queries work inside layers normally. The layer priority is evaluated first, then the media query condition. Example:

```css
@layer base, components;

@layer base {
  @media (min-width: 768px) {
    .card { font-size: 1.2rem; }
  }
}

@layer components {
  .card { font-size: 1rem; } /* Wins even on wide screens */
}
```

**Q7: Can you reorder layers after declaring them?**

**A:** No. Layer order is established on first declaration and cannot be changed. Subsequent `@layer` statements with new names create additional layers at the end, they don't reorder existing layers. This is intentional - the order should be established upfront for clarity.

## Resources

- [MDN: @layer](https://developer.mozilla.org/en-US/docs/Web/CSS/@layer)
- [CSS Cascading and Inheritance Spec](https://www.w3.org/TR/css-cascade-5/)
- [Miriam Suzanne: Cascade Layers Explainer](https://css.oddbird.net/layers/)
- [Bramus: CSS Cascade Layers](https://www.bram.us/2021/09/15/the-future-of-css-cascade-layers-css-at-layer/)
- [Web.dev: Cascade Layers](https://web.dev/css-cascade-layers/)
- [Can I Use: Cascade Layers](https://caniuse.com/css-cascade-layers)

## Summary

Cascade layers provide explicit control over CSS priority, replacing complex specificity management with clear layer ordering. Declare layer order upfront, use later layers for higher priority styles, and keep unlayered styles for exceptional overrides. Layers enable better third-party CSS integration, clearer design system architecture, and simpler selectors. The result is more maintainable, predictable stylesheets with fewer specificity battles and reduced need for `!important`.
