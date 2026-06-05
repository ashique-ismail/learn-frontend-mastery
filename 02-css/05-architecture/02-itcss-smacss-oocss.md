# ITCSS, SMACSS, and OOCSS

## The Idea

**In plain English:** ITCSS, SMACSS, and OOCSS are three different systems for organising the styling rules of a website so they don't clash or become a tangled mess as the project grows. Think of them as filing systems — each one has a different way of deciding where each rule goes and why.

**Real-world analogy:** Imagine a large restaurant kitchen with many cooks. The head chef creates rules for how the kitchen is organised: ingredients are stored by type (spices on one shelf, proteins in the fridge), prep tasks are done in a set order (chop before you cook), and each dish has a standard recipe so any cook can make it consistently.

- The shelves organised by ingredient type = ITCSS layers (styles grouped from broadest/least specific to most specific)
- The step-by-step prep order = SMACSS categories (styles sorted by purpose: base, layout, module, state, theme)
- The standard reusable recipes = OOCSS objects (reusable style patterns combined to build any dish/component)

---

## Overview

Three foundational CSS architectural methodologies that organize styles by purpose, specificity, and reusability. Each provides a different lens for structuring large-scale CSS codebases.

## ITCSS (Inverted Triangle CSS)

### Philosophy

Created by Harry Roberts, ITCSS organizes CSS from generic to specific, low to high specificity, forming an inverted triangle. Styles cascade naturally without fighting specificity.

### The Seven Layers

```
     Settings      (variables, config)
      Tools        (mixins, functions)
     Generic       (resets, normalize)
    Elements       (bare HTML elements)
   Objects         (layout patterns)
  Components       (UI components)
 Utilities         (helpers, overrides)
```

#### 1. Settings

Global variables, configuration:

```css
/* settings/_colors.css */
:root {
  --color-primary: #007bff;
  --color-secondary: #6c757d;
  --color-success: #28a745;
  --color-danger: #dc3545;
  
  --spacing-unit: 8px;
  --border-radius: 4px;
}
```

```scss
// settings/_config.scss
$breakpoints: (
  'small': 480px,
  'medium': 768px,
  'large': 1024px,
  'xlarge': 1200px
);

$font-sizes: (
  'small': 0.875rem,
  'base': 1rem,
  'large': 1.25rem,
  'xlarge': 1.5rem
);
```

#### 2. Tools

Mixins, functions (preprocessors only):

```scss
// tools/_mixins.scss
@mixin respond-to($breakpoint) {
  @media (min-width: map-get($breakpoints, $breakpoint)) {
    @content;
  }
}

@mixin truncate {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

// tools/_functions.scss
@function spacing($multiplier: 1) {
  @return calc(var(--spacing-unit) * #{$multiplier});
}
```

#### 3. Generic

Browser resets, normalize, box-sizing:

```css
/* generic/_reset.css */
*,
*::before,
*::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

/* generic/_normalize.css */
body {
  line-height: 1.5;
  -webkit-font-smoothing: antialiased;
}

img,
picture,
video,
canvas,
svg {
  display: block;
  max-width: 100%;
}

input,
button,
textarea,
select {
  font: inherit;
}
```

#### 4. Elements

Bare HTML elements, no classes:

```css
/* elements/_headings.css */
h1, h2, h3, h4, h5, h6 {
  font-weight: 700;
  line-height: 1.2;
}

h1 { font-size: 2.5rem; }
h2 { font-size: 2rem; }
h3 { font-size: 1.75rem; }

/* elements/_links.css */
a {
  color: var(--color-primary);
  text-decoration: none;
}

a:hover {
  text-decoration: underline;
}

/* elements/_forms.css */
input,
textarea,
select {
  border: 1px solid #ddd;
  border-radius: var(--border-radius);
  padding: 8px 12px;
}
```

#### 5. Objects

Layout patterns, structural classes (OOCSS principles):

```css
/* objects/_container.css */
.o-container {
  max-width: 1200px;
  margin-left: auto;
  margin-right: auto;
  padding-left: 16px;
  padding-right: 16px;
}

/* objects/_layout.css */
.o-layout {
  display: flex;
  flex-wrap: wrap;
  gap: var(--spacing-unit);
}

.o-layout__item {
  flex: 1 1 0%;
}

/* objects/_media.css */
.o-media {
  display: flex;
  gap: 16px;
}

.o-media__figure {
  flex: 0 0 auto;
}

.o-media__body {
  flex: 1 1 0%;
}

/* objects/_stack.css */
.o-stack > * + * {
  margin-top: var(--spacing-unit);
}
```

#### 6. Components

Specific UI components (BEM naming):

```css
/* components/_button.css */
.c-button {
  display: inline-block;
  padding: 12px 24px;
  background: var(--color-primary);
  color: white;
  border: none;
  border-radius: var(--border-radius);
  cursor: pointer;
  font-weight: 600;
}

.c-button--secondary {
  background: var(--color-secondary);
}

.c-button--small {
  padding: 6px 12px;
  font-size: 0.875rem;
}

/* components/_card.css */
.c-card {
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.c-card__header {
  padding: 20px;
  border-bottom: 1px solid #eee;
}

.c-card__body {
  padding: 20px;
}
```

#### 7. Utilities

Single-purpose helpers, highest specificity:

```css
/* utilities/_spacing.css */
.u-margin-top-none { margin-top: 0 !important; }
.u-margin-top-small { margin-top: 8px !important; }
.u-margin-top { margin-top: 16px !important; }
.u-margin-top-large { margin-top: 24px !important; }

.u-padding { padding: 16px !important; }

/* utilities/_text.css */
.u-text-center { text-align: center !important; }
.u-text-right { text-align: right !important; }
.u-text-truncate {
  overflow: hidden !important;
  text-overflow: ellipsis !important;
  white-space: nowrap !important;
}

/* utilities/_display.css */
.u-hidden { display: none !important; }
.u-visually-hidden {
  position: absolute !important;
  width: 1px !important;
  height: 1px !important;
  padding: 0 !important;
  margin: -1px !important;
  overflow: hidden !important;
  clip: rect(0, 0, 0, 0) !important;
  white-space: nowrap !important;
  border: 0 !important;
}
```

### File Structure

```
styles/
├── settings/
│   ├── _colors.scss
│   ├── _typography.scss
│   └── _config.scss
├── tools/
│   ├── _mixins.scss
│   └── _functions.scss
├── generic/
│   ├── _reset.css
│   └── _normalize.css
├── elements/
│   ├── _headings.css
│   ├── _links.css
│   └── _forms.css
├── objects/
│   ├── _container.css
│   ├── _layout.css
│   └── _media.css
├── components/
│   ├── _button.css
│   ├── _card.css
│   └── _nav.css
├── utilities/
│   ├── _spacing.css
│   ├── _text.css
│   └── _display.css
└── main.scss (imports all in order)
```

### Benefits

1. **Predictable Specificity**: Each layer naturally has higher specificity
2. **No Specificity Wars**: Clear hierarchy prevents conflicts
3. **Reusability**: Objects layer promotes DRY patterns
4. **Scalability**: Easy to add new components/utilities
5. **Performance**: Can split-load (critical CSS from top layers)

---

## SMACSS (Scalable and Modular Architecture for CSS)

### Philosophy

Created by Jonathan Snook, SMACSS categorizes CSS rules by purpose and provides naming conventions for each category.

### The Five Categories

#### 1. Base

Element selectors only, no classes:

```css
/* base.css */
html {
  font-size: 16px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}

body {
  line-height: 1.6;
  color: #333;
}

a {
  color: #0066cc;
}

a:hover {
  color: #004080;
}
```

#### 2. Layout

Major page sections (prefix: `l-` or `layout-`):

```css
/* layout.css */
.l-header {
  position: sticky;
  top: 0;
  background: white;
  border-bottom: 1px solid #ddd;
  z-index: 100;
}

.l-sidebar {
  width: 250px;
  flex-shrink: 0;
}

.l-main {
  flex: 1;
  padding: 20px;
}

.l-footer {
  background: #f5f5f5;
  padding: 40px 20px;
}

/* Grid system */
.l-grid {
  display: grid;
  gap: 20px;
}

.l-grid--2col {
  grid-template-columns: repeat(2, 1fr);
}

.l-grid--3col {
  grid-template-columns: repeat(3, 1fr);
}
```

#### 3. Module

Reusable components (no prefix or `m-`):

```css
/* modules/button.css */
.button {
  display: inline-block;
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.button-primary {
  background: #007bff;
  color: white;
}

.button-secondary {
  background: #6c757d;
  color: white;
}

/* modules/card.css */
.card {
  background: white;
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 20px;
}

.card-header {
  border-bottom: 1px solid #eee;
  padding-bottom: 10px;
  margin-bottom: 10px;
}

.card-title {
  font-size: 1.5rem;
  font-weight: 600;
}
```

#### 4. State

Dynamic states (prefix: `is-` or `has-`):

```css
/* state.css */
.is-hidden {
  display: none;
}

.is-visible {
  display: block;
}

.is-active {
  font-weight: bold;
  color: #007bff;
}

.is-disabled {
  opacity: 0.5;
  pointer-events: none;
  cursor: not-allowed;
}

.is-loading {
  position: relative;
  pointer-events: none;
}

.is-loading::after {
  content: '';
  position: absolute;
  inset: 0;
  background: rgba(255, 255, 255, 0.7);
}

.has-error {
  border-color: #dc3545;
}

.has-success {
  border-color: #28a745;
}
```

State rules can modify modules:

```css
/* Module with state */
.button.is-disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.nav-item.is-active {
  border-bottom: 2px solid #007bff;
}
```

#### 5. Theme

Visual variations (prefix: `theme-`):

```css
/* themes/dark.css */
.theme-dark {
  --bg-primary: #1a1a1a;
  --bg-secondary: #2d2d2d;
  --text-primary: #ffffff;
  --text-secondary: #b3b3b3;
  --border-color: #404040;
}

.theme-dark body {
  background: var(--bg-primary);
  color: var(--text-primary);
}

.theme-dark .card {
  background: var(--bg-secondary);
  border-color: var(--border-color);
}

/* themes/high-contrast.css */
.theme-high-contrast {
  --bg-primary: #000000;
  --text-primary: #ffffff;
  --link-color: #ffff00;
}
```

### Naming Conventions

```
Base:      element selectors (a, h1, p)
Layout:    l-header, l-sidebar, l-grid
Module:    button, card, nav
State:     is-active, is-hidden, has-error
Theme:     theme-dark, theme-light
```

### Usage Example

```html
<div class="l-container">
  <header class="l-header">
    <nav class="nav">
      <a href="#" class="nav-item is-active">Home</a>
      <a href="#" class="nav-item">About</a>
    </nav>
  </header>
  
  <main class="l-main">
    <article class="card theme-dark">
      <h2 class="card-title">Article Title</h2>
      <button class="button button-primary is-disabled">
        Click Me
      </button>
    </article>
  </main>
</div>
```

---

## OOCSS (Object-Oriented CSS)

### Philosophy

Created by Nicole Sullivan, OOCSS applies object-oriented programming principles to CSS for maximum reusability and minimal duplication.

### Core Principles

#### 1. Separation of Structure and Skin

**Structure**: Layout properties (width, height, margin, padding, position)
**Skin**: Visual properties (color, background, border, font)

**Bad (Coupled):**

```css
.button-primary {
  width: 200px;
  padding: 10px 20px;
  background: #007bff;
  color: white;
  border-radius: 4px;
}

.button-secondary {
  width: 200px;
  padding: 10px 20px;
  background: #6c757d;
  color: white;
  border-radius: 4px;
}
```

**Good (Separated):**

```css
/* Structure */
.button {
  display: inline-block;
  padding: 10px 20px;
  border-radius: 4px;
  text-align: center;
  cursor: pointer;
}

/* Skin */
.skin-primary {
  background: #007bff;
  color: white;
}

.skin-secondary {
  background: #6c757d;
  color: white;
}
```

```html
<button class="button skin-primary">Primary</button>
<button class="button skin-secondary">Secondary</button>
```

#### 2. Separation of Container and Content

Content should look the same regardless of its container.

**Bad (Location-Dependent):**

```css
.header h2 {
  font-size: 2rem;
  color: #333;
}

.sidebar h2 {
  font-size: 2rem;
  color: #333;
}

.footer h2 {
  font-size: 1.5rem;
  color: #666;
}
```

**Good (Location-Independent):**

```css
.title-large {
  font-size: 2rem;
  color: #333;
}

.title-medium {
  font-size: 1.5rem;
  color: #666;
}
```

```html
<header class="header">
  <h2 class="title-large">Header Title</h2>
</header>

<aside class="sidebar">
  <h2 class="title-large">Sidebar Title</h2>
</aside>

<footer class="footer">
  <h2 class="title-medium">Footer Title</h2>
</footer>
```

### OOCSS Patterns

#### Media Object

Classic OOCSS pattern for image + text:

```css
.media {
  display: flex;
  gap: 16px;
}

.media-figure {
  flex: 0 0 auto;
}

.media-figure--reverse {
  order: 2;
}

.media-body {
  flex: 1;
}
```

```html
<div class="media">
  <div class="media-figure">
    <img src="avatar.jpg" alt="User">
  </div>
  <div class="media-body">
    <h3>User Name</h3>
    <p>User description...</p>
  </div>
</div>
```

#### Flag Object

Image and text vertically centered:

```css
.flag {
  display: flex;
  align-items: center;
  gap: 12px;
}

.flag-figure {
  flex: 0 0 auto;
}

.flag-body {
  flex: 1;
}
```

#### Box Object

Generic container with optional header/body/footer:

```css
.box {
  background: white;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.box-header {
  padding: 15px;
  border-bottom: 1px solid #ddd;
}

.box-body {
  padding: 15px;
}

.box-footer {
  padding: 15px;
  border-top: 1px solid #ddd;
}
```

### Composition Example

```html
<div class="box skin-primary">
  <div class="box-header">
    <div class="media">
      <div class="media-figure">
        <img src="icon.png" alt="" width="40" height="40">
      </div>
      <div class="media-body">
        <h3 class="title-medium">Card Title</h3>
      </div>
    </div>
  </div>
  <div class="box-body">
    <p>Content goes here...</p>
  </div>
  <div class="box-footer">
    <button class="button skin-secondary">Action</button>
  </div>
</div>
```

---

## Comparison Table

| Aspect | ITCSS | SMACSS | OOCSS |
|--------|-------|--------|-------|
| **Primary Goal** | Manage specificity | Organize by purpose | Maximize reusability |
| **Structure** | 7 layers (triangle) | 5 categories | Objects + composition |
| **Specificity** | Low to high by layer | Varies by category | Flat, single class |
| **Learning Curve** | Moderate | Easy | Moderate |
| **File Organization** | By layer | By category | By object |
| **Naming Convention** | Flexible (often BEM) | Prefixes (l-, is-) | Descriptive classes |
| **Preprocessor Required** | No (but helps) | No | No |
| **Best For** | Large teams, specificity control | Medium projects, clear categories | Component libraries, DRY code |
| **Utilities** | Bottom layer | Not emphasized | Not emphasized |
| **State Management** | In components/utilities | Dedicated layer | Mixed with objects |
| **Theming** | Settings + CSS vars | Dedicated layer | Via skins |

## When to Use Each

### Choose ITCSS When:

- Specificity conflicts are a major concern
- You need predictable CSS cascade
- Working with large teams
- Building design systems
- Want clear separation of concerns

### Choose SMACSS When:

- Need simple, intuitive organization
- Team prefers clear categories
- Want quick onboarding
- Medium-sized projects
- Value explicit state management

### Choose OOCSS When:

- Maximum reusability is priority
- Building component libraries
- Want minimal CSS bloat
- Need highly composable patterns
- Working with modular systems

## Combining Methodologies

### ITCSS + BEM

Most common combination:

```css
/* Objects layer - structural */
.o-media { }
.o-media__figure { }
.o-media__body { }

/* Components layer - BEM naming */
.c-card { }
.c-card__header { }
.c-card--featured { }

/* Utilities layer */
.u-margin-top { }
```

### SMACSS + OOCSS

Categorization + composition:

```css
/* Layout */
.l-container { }

/* Module (OOCSS structure) */
.media { }
.media-figure { }

/* Module (OOCSS skin) */
.skin-primary { }

/* State */
.is-active { }
```

### Triple Hybrid

ITCSS structure + SMACSS naming + OOCSS principles:

```css
/* ITCSS: Objects layer */
/* OOCSS: Structural pattern */
.o-media { }

/* ITCSS: Components layer */
/* SMACSS: State prefix */
.c-card.is-active { }

/* ITCSS: Utilities layer */
.u-text-center { }
```

## Team Workflow Considerations

### Documentation

Document your chosen approach:

```markdown
# CSS Architecture

We use ITCSS with BEM naming conventions.

## Layer Order
1. Settings (variables)
2. Tools (mixins)
3. Generic (resets)
4. Elements (bare HTML)
5. Objects (layout patterns, o- prefix)
6. Components (UI components, c- prefix, BEM naming)
7. Utilities (helpers, u- prefix, !important allowed)

## Naming Conventions
- Objects: `.o-object-name`
- Components: `.c-block__element--modifier`
- Utilities: `.u-utility-name`
- States: `is-`, `has-` prefixes
- JavaScript hooks: `.js-hook-name` (no styles)
```

### Code Review Checklist

- [ ] Styles in correct layer/category?
- [ ] Appropriate specificity for layer?
- [ ] Following naming conventions?
- [ ] No location-dependent styles? (OOCSS)
- [ ] Reusable patterns extracted to objects?
- [ ] States using correct prefixes?

### Tooling

#### Stylelint Configuration

```json
{
  "rules": {
    "selector-class-pattern": [
      "^([a-z][a-z0-9]*)((__|--|-)[a-z0-9]+)*$",
      {
        "message": "Class should follow BEM naming convention"
      }
    ],
    "selector-max-specificity": "0,3,0",
    "selector-max-id": 0,
    "selector-no-qualifying-type": [
      true,
      {
        "ignore": ["attribute", "class"]
      }
    ]
  }
}
```

#### Build Process

```javascript
// Import order enforced
import './settings/_variables.css';
import './tools/_mixins.scss';
import './generic/_reset.css';
import './elements/_typography.css';
import './objects/_container.css';
import './components/_button.css';
import './utilities/_spacing.css';
```

### Migration Strategy

#### Gradual Adoption

1. **Phase 1**: Introduce layer/category structure
   - Create directories
   - Move existing files to appropriate layers

2. **Phase 2**: Refactor high-impact components
   - Extract common patterns to objects
   - Apply naming conventions

3. **Phase 3**: New code follows architecture
   - Old code stays as-is
   - Refactor on touch

4. **Phase 4**: Complete migration
   - Dedicate sprint to cleanup
   - Remove old styles

## Performance Considerations

### Critical CSS

ITCSS makes critical CSS easy:

```html
<head>
  <style>
    /* Inline: Settings, Generic, Elements, Objects */
  </style>
  <link rel="preload" href="components.css" as="style">
  <link rel="stylesheet" href="components.css">
</head>
```

### Code Splitting

Split by layer for better caching:

```javascript
// vendor.css (rarely changes)
import 'generic/reset.css';
import 'elements/base.css';

// layout.css (occasionally changes)
import 'objects/container.css';
import 'objects/grid.css';

// components.css (frequently changes)
import 'components/*.css';

// utilities.css (rarely changes)
import 'utilities/*.css';
```

### Specificity Performance

Lower specificity = faster selector matching:

```css
/* Fast - OOCSS flat classes */
.button { }

/* Medium - SMACSS module */
.nav-item { }

/* Slower - nested */
.nav .item { }

/* Slowest - complex chain */
.header > .nav ul li a { }
```

## Conclusion

All three methodologies remain relevant and can be combined effectively:

- **ITCSS**: Best for managing specificity and scale
- **SMACSS**: Best for intuitive organization
- **OOCSS**: Best for reusability and composition

Modern projects often use ITCSS as the structural foundation, OOCSS principles for objects/components, and BEM for naming. The key is choosing what fits your team's needs and maintaining consistency.

### Recommended Approach

For most teams:

```
ITCSS (structure) + BEM (naming) + OOCSS (reusability principles)
```

This combination provides:
- Clear organization (ITCSS)
- Predictable naming (BEM)
- Minimal duplication (OOCSS)
- Scalability (all three)
