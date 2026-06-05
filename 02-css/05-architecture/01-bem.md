# BEM (Block Element Modifier)

## The Idea

**In plain English:** BEM is a system for naming the CSS classes you put on HTML elements, so that every class name tells you exactly which component it belongs to, which part of that component it is, and whether it looks different from the default. A "class" is just a label you attach to an HTML element so your stylesheet knows how to style it.

**Real-world analogy:** Think of a school uniform system. Every student wears a uniform (the block), which has specific pieces like a shirt, trousers, and a blazer (the elements), and some students wear a prefect badge or a sports-house colour stripe to show a variation (the modifier).

- The uniform = the Block (the self-contained component, e.g. `.uniform`)
- The shirt, trousers, blazer = the Elements (parts that only make sense as part of the uniform, e.g. `.uniform__shirt`)
- The prefect badge or colour stripe = the Modifier (a variation that changes how a part looks or behaves, e.g. `.uniform__blazer--prefect`)

---

## Overview

BEM is a naming methodology that provides a structured approach to naming CSS classes, making code more maintainable, predictable, and self-documenting. Created by Yandex, BEM stands for Block, Element, Modifier.

## Naming Convention

### Structure

```
block__element--modifier
```

### Components

1. **Block**: Standalone entity that is meaningful on its own
   - `.button`, `.card`, `.menu`, `.header`

2. **Element**: Part of a block with no standalone meaning, semantically tied to its block
   - `.block__element`
   - `.button__icon`, `.card__title`, `.menu__item`

3. **Modifier**: Flag on block or element, used to change appearance or behavior
   - `.block--modifier`, `.block__element--modifier`
   - `.button--primary`, `.card__title--large`

### Naming Rules

```css
/* Block */
.card { }

/* Element */
.card__header { }
.card__body { }
.card__footer { }

/* Modifier on block */
.card--featured { }
.card--compact { }

/* Modifier on element */
.card__header--dark { }
.card__title--large { }

/* Boolean modifiers */
.button--disabled { }
.menu__item--active { }

/* Key-value modifiers */
.button--size-small { }
.button--size-large { }
.button--theme-primary { }
.button--theme-secondary { }
```

## Practical Examples

### Basic Component

```html
<article class="card card--featured">
  <header class="card__header card__header--dark">
    <h2 class="card__title">Article Title</h2>
    <p class="card__subtitle">Subtitle here</p>
  </header>
  <div class="card__body">
    <p class="card__text">Content goes here...</p>
  </div>
  <footer class="card__footer">
    <button class="card__button card__button--primary">Read More</button>
  </footer>
</article>
```

```css
/* Block */
.card {
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

/* Block modifier */
.card--featured {
  border: 2px solid gold;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

/* Elements */
.card__header {
  padding: 20px;
  border-bottom: 1px solid #eee;
}

.card__header--dark {
  background: #333;
  color: white;
}

.card__title {
  margin: 0;
  font-size: 1.5rem;
}

.card__body {
  padding: 20px;
}

.card__footer {
  padding: 20px;
  border-top: 1px solid #eee;
}

.card__button {
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.card__button--primary {
  background: #007bff;
  color: white;
}
```

### Navigation Menu

```html
<nav class="menu menu--horizontal">
  <ul class="menu__list">
    <li class="menu__item menu__item--active">
      <a href="#" class="menu__link">Home</a>
    </li>
    <li class="menu__item">
      <a href="#" class="menu__link">About</a>
    </li>
    <li class="menu__item menu__item--has-dropdown">
      <a href="#" class="menu__link">Products</a>
      <ul class="menu__dropdown">
        <li class="menu__dropdown-item">
          <a href="#" class="menu__dropdown-link">Product 1</a>
        </li>
      </ul>
    </li>
  </ul>
</nav>
```

```css
.menu {
  background: white;
  padding: 0;
}

.menu--horizontal .menu__list {
  display: flex;
}

.menu--vertical .menu__list {
  display: block;
}

.menu__list {
  list-style: none;
  margin: 0;
  padding: 0;
}

.menu__item {
  position: relative;
}

.menu__item--active .menu__link {
  font-weight: bold;
  color: #007bff;
}

.menu__item--has-dropdown:hover .menu__dropdown {
  display: block;
}

.menu__link {
  display: block;
  padding: 15px 20px;
  text-decoration: none;
  color: #333;
}

.menu__dropdown {
  display: none;
  position: absolute;
  top: 100%;
  left: 0;
  background: white;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}
```

## Benefits

### 1. No Nesting Conflicts

```css
/* Traditional nested approach - prone to specificity issues */
.header .nav .item a { }
.footer .nav .item a { }

/* BEM approach - flat specificity */
.header-nav__link { }
.footer-nav__link { }
```

### 2. Self-Documenting

Class names clearly communicate:
- Component ownership (block name)
- Component part (element name)
- State or variation (modifier name)

```html
<!-- Immediately clear what this is and its state -->
<button class="button button--primary button--disabled">
  Click me
</button>
```

### 3. Predictable

```css
/* Always know where to find styles */
.search { } /* Block level */
.search__input { } /* Element level */
.search__button { } /* Element level */
.search--compact { } /* Variation */
```

### 4. Reusability

Blocks are independent and can be reused anywhere:

```html
<!-- Same button block used in different contexts -->
<header class="header">
  <button class="button button--small">Menu</button>
</header>

<main class="content">
  <button class="button button--primary">Submit</button>
</main>
```

### 5. No Specificity Wars

All selectors have same specificity (single class):

```css
.button { } /* specificity: 0,0,1,0 */
.button--primary { } /* specificity: 0,0,1,0 */
.button__icon { } /* specificity: 0,0,1,0 */
```

## When to Use BEM

### Good Use Cases

1. **Component Libraries**
   - Building reusable UI components
   - Maintaining consistency across projects

2. **Large Teams**
   - Multiple developers working on same codebase
   - Need for clear naming conventions

3. **Long-Term Projects**
   - Code maintainability is priority
   - Self-documenting code is valuable

4. **Plain CSS/SCSS**
   - No CSS-in-JS or CSS Modules
   - Need namespace isolation manually

### When to Consider Alternatives

1. **Small Projects**: Overhead might not be worth it
2. **Rapid Prototyping**: Speed over structure
3. **Using CSS-in-JS**: Automatic scoping makes BEM redundant
4. **Utility-First Framework**: Tailwind already provides naming strategy
5. **CSS Modules**: Automatic local scoping reduces naming burden

## Common Mistakes

### 1. Over-Nesting Elements

```css
/* WRONG - elements chained */
.block__element1__element2__element3 { }

/* RIGHT - flat structure */
.block__element1 { }
.block__element2 { }
.block__element3 { }
```

**Rule**: Elements should be direct children in naming, regardless of DOM nesting.

### 2. Mixing Block and Element

```html
<!-- WRONG -->
<div class="card">
  <div class="header card__header">
    <!-- 'header' is both block and element -->
  </div>
</div>

<!-- RIGHT -->
<div class="card">
  <div class="card__header">
  </div>
</div>
```

### 3. Modifier Without Base Class

```html
<!-- WRONG -->
<button class="button--primary">Click</button>

<!-- RIGHT -->
<button class="button button--primary">Click</button>
```

**Rule**: Modifiers extend blocks/elements, they don't replace them.

### 4. Using Modifiers for States

```html
<!-- QUESTIONABLE -->
<button class="button button--clicked">Click</button>

<!-- BETTER - use data attributes or ARIA for states -->
<button class="button" aria-pressed="true">Click</button>
```

```css
.button[aria-pressed="true"] {
  /* styles for pressed state */
}
```

### 5. Grandfathers (Element of Element)

```html
<!-- WRONG -->
<div class="card">
  <div class="card__header">
    <h2 class="card__header__title">Title</h2>
  </div>
</div>

<!-- RIGHT -->
<div class="card">
  <div class="card__header">
    <h2 class="card__title">Title</h2>
  </div>
</div>
```

## BEM with Modern CSS

### CSS Nesting

Modern CSS nesting can make BEM more ergonomic:

```css
/* With CSS Nesting (native or preprocessor) */
.card {
  background: white;
  border-radius: 8px;
  
  &__header {
    padding: 20px;
    border-bottom: 1px solid #eee;
  }
  
  &__body {
    padding: 20px;
  }
  
  &__footer {
    padding: 20px;
    border-top: 1px solid #eee;
  }
  
  &--featured {
    border: 2px solid gold;
  }
}
```

### :has() Selector

Parent state based on children:

```css
/* Card with active item changes appearance */
.card:has(.card__item--active) {
  border-color: #007bff;
}

/* Button group with disabled button */
.button-group:has(.button--disabled) {
  opacity: 0.7;
}
```

### Container Queries

BEM with container-aware components:

```css
.card {
  container-type: inline-size;
  container-name: card;
}

@container card (min-width: 400px) {
  .card__title {
    font-size: 2rem;
  }
  
  .card--featured .card__header {
    padding: 30px;
  }
}
```

### CSS Custom Properties

BEM with themeable components:

```css
.button {
  --button-bg: #007bff;
  --button-color: white;
  --button-padding: 10px 20px;
  
  background: var(--button-bg);
  color: var(--button-color);
  padding: var(--button-padding);
}

.button--small {
  --button-padding: 5px 10px;
}

.button--theme-danger {
  --button-bg: #dc3545;
}
```

## Variants and Alternatives

### Two Dashes vs One Dash

```css
/* Standard BEM */
.block__element--modifier { }

/* Alternative (Harry Roberts) */
.block__element-modifier { }
```

### Camel Case vs Kebab Case

```css
/* Kebab case (more common) */
.user-card__profile-image--large { }

/* Camel case */
.userCard__profileImage--large { }
```

### Namespace Prefixes

```css
/* Component prefix */
.c-card { }
.c-card__header { }

/* Object prefix */
.o-layout { }
.o-layout__item { }

/* Utility prefix */
.u-margin-top { }

/* JavaScript hook prefix */
.js-toggle { }
```

## Team Workflow Considerations

### 1. Documentation

Create a style guide:

```markdown
# BEM Naming Guide

## Blocks
- Nouns describing standalone components
- Examples: `.button`, `.card`, `.modal`

## Elements
- Parts of blocks, use descriptive names
- Examples: `.card__header`, `.modal__close-button`

## Modifiers
- States or variations
- Examples: `.button--primary`, `.card--featured`
```

### 2. Code Reviews

Check for:
- Consistent naming patterns
- No element chaining (`.block__el1__el2`)
- Modifiers used with base class
- Logical block boundaries

### 3. Linting

Use stylelint with BEM plugins:

```json
{
  "plugins": ["stylelint-selector-bem-pattern"],
  "rules": {
    "plugin/selector-bem-pattern": {
      "preset": "bem",
      "componentSelectors": "^\\.{componentName}(?:__[a-z]+)?(?:--[a-z]+)?$"
    }
  }
}
```

### 4. Component Directory Structure

```
components/
├── card/
│   ├── card.html
│   ├── card.css
│   ├── card.js
│   └── card.test.js
├── button/
│   ├── button.html
│   ├── button.css
│   └── button.js
```

### 5. Avoiding BEM Fatigue

Strategies for reducing verbosity:

```scss
// Use mixins for common patterns
@mixin element($name) {
  &__#{$name} {
    @content;
  }
}

@mixin modifier($name) {
  &--#{$name} {
    @content;
  }
}

.card {
  @include element(header) {
    padding: 20px;
  }
  
  @include modifier(featured) {
    border: 2px solid gold;
  }
}
```

## BEM vs Other Methodologies

| Aspect | BEM | Other Approaches |
|--------|-----|------------------|
| Learning Curve | Moderate | Varies |
| Specificity | Flat | Can be nested |
| Naming Length | Long | Shorter |
| Scalability | Excellent | Varies |
| Tool Support | Good | Varies |
| Team Adoption | Requires discipline | Easier to start |

## Migration Strategy

### From Traditional CSS

1. Identify logical blocks
2. Rename nested selectors to BEM elements
3. Convert state classes to modifiers
4. Remove tag selectors from CSS

```css
/* Before */
.header nav ul li a { }
.header nav ul li.active a { }

/* After */
.header-nav__link { }
.header-nav__item--active .header-nav__link { }
```

### Gradual Adoption

```css
/* Mix old and new during transition */
.legacy-component { }

/* New components use BEM */
.new-card { }
.new-card__header { }
.new-card--featured { }
```

## Performance Considerations

### Class Name Length

While BEM creates longer class names, the impact is minimal:

- Gzips well (repetitive patterns compress)
- No runtime performance impact
- Slightly larger HTML/CSS file size

### Selector Performance

BEM's single-class selectors are fastest:

```css
/* Fast - BEM */
.card__title { }

/* Slower - descendant selector */
.card .title { }

/* Slowest - complex selector */
.container > .card:not(.featured) .title { }
```

## Conclusion

BEM provides a robust, scalable methodology for CSS architecture. While it requires discipline and produces longer class names, the benefits of maintainability, predictability, and team collaboration make it valuable for medium to large projects. Modern CSS features like nesting and custom properties can reduce BEM's verbosity while maintaining its structural benefits.

### Key Takeaways

1. Use flat element naming (no chaining)
2. Always include base class with modifiers
3. Keep blocks independent and reusable
4. Use modifiers for variations, attributes for states
5. Combine with modern CSS for best ergonomics
6. Establish team conventions early
7. Use linting to enforce consistency
