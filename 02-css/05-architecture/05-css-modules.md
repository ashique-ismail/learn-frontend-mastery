# CSS Modules

## The Idea

**In plain English:** CSS Modules is a system that automatically gives every CSS style a unique, one-of-a-kind name so that styles written for one part of your website never accidentally affect another part. Think of it as a tool that keeps your styling rules strictly "local" to the component (a self-contained chunk of a webpage) they belong to.

**Real-world analogy:** Imagine a school where every student wears a name badge that includes their homeroom number, like "Room 3 - Alex" and "Room 7 - Alex". Even though two students share the name Alex, the full badge label makes each one unique so teachers never mix them up.
- The student's first name = the CSS class name you write (e.g., `.button`)
- The homeroom number added to the badge = the unique hash the build tool appends automatically (e.g., `Button_button__2Rfj3`)
- Calling on "Room 3 - Alex" specifically = referencing `styles.button` in your component so only that component's styles are applied

---

## Overview

CSS Modules is a build-time solution that provides automatic local scoping for CSS class names. It transforms class names to be unique, preventing naming collisions and enabling true component-level encapsulation without runtime overhead.

## Core Concept

### The Problem

Traditional CSS has a global namespace:

```css
/* button.css */
.button {
  background: blue;
}

/* card.css */
.button {
  background: red; /* Collision! */
}
```

```html
<!-- Which style wins? -->
<button class="button">Click me</button>
```

### The CSS Modules Solution

```css
/* Button.module.css */
.button {
  background: blue;
}
```

Compiled to:

```css
.Button_button__2Rfj3 {
  background: blue;
}
```

```jsx
// Button.jsx
import styles from './Button.module.css';

function Button() {
  return <button className={styles.button}>Click me</button>;
}
// Renders: <button class="Button_button__2Rfj3">Click me</button>
```

## Local vs Global Scope

### Default: Local Scope

All classes are local by default:

```css
/* Component.module.css */
.container {
  padding: 20px;
}

.title {
  font-size: 2rem;
  font-weight: bold;
}

.button {
  background: blue;
  color: white;
}
```

Compiles to:

```css
.Component_container__1a2b3 {
  padding: 20px;
}

.Component_title__4c5d6 {
  font-size: 2rem;
  font-weight: bold;
}

.Component_button__7e8f9 {
  background: blue;
  color: white;
}
```

### Explicit Global Scope

Use `:global()` for global classes:

```css
/* Component.module.css */

/* Local (default) */
.button {
  padding: 10px;
}

/* Global */
:global(.btn-primary) {
  background: blue;
}

/* Mix local and global */
.container :global(.legacy-class) {
  margin: 10px;
}

/* Multiple globals */
:global {
  .global-class-1 {
    color: red;
  }
  .global-class-2 {
    color: blue;
  }
}
```

Usage:

```jsx
import styles from './Component.module.css';

function Component() {
  return (
    <div className={styles.container}>
      {/* Local class */}
      <button className={styles.button}>Local</button>
      
      {/* Global class (use string directly) */}
      <button className="btn-primary">Global</button>
    </div>
  );
}
```

### Explicit Local Scope

Use `:local()` when you change default behavior:

```css
/* If config sets global as default */
:local(.button) {
  background: blue;
}
```

## Composition

### composes Keyword

Share styles between classes:

```css
/* Button.module.css */

.baseButton {
  display: inline-block;
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: 600;
  transition: all 0.2s;
}

.primaryButton {
  composes: baseButton;
  background: #007bff;
  color: white;
}

.secondaryButton {
  composes: baseButton;
  background: #6c757d;
  color: white;
}

.largeButton {
  composes: primaryButton;
  padding: 15px 30px;
  font-size: 1.2rem;
}
```

Compiled to:

```jsx
styles.primaryButton
// Returns: "Button_baseButton__a1b2c Button_primaryButton__d3e4f"

styles.largeButton
// Returns: "Button_baseButton__a1b2c Button_primaryButton__d3e4f Button_largeButton__g5h6i"
```

Usage:

```jsx
import styles from './Button.module.css';

function Buttons() {
  return (
    <>
      <button className={styles.primaryButton}>Primary</button>
      <button className={styles.secondaryButton}>Secondary</button>
      <button className={styles.largeButton}>Large Primary</button>
    </>
  );
}
```

### Composing from Other Files

```css
/* base.module.css */
.reset {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

.typography {
  font-family: system-ui, sans-serif;
  line-height: 1.5;
}
```

```css
/* Button.module.css */
.button {
  composes: reset from './base.module.css';
  composes: typography from './base.module.css';
  padding: 10px 20px;
  background: blue;
  color: white;
}
```

### Composing from Global

```css
/* Component.module.css */
.container {
  composes: container from global;
  /* Add local styles */
  padding-top: 20px;
}
```

## @value (CSS Variables Alternative)

### Basic Values

```css
/* colors.module.css */
@value primaryColor: #007bff;
@value secondaryColor: #6c757d;
@value successColor: #28a745;
@value dangerColor: #dc3545;

@value basePadding: 16px;
@value borderRadius: 4px;
```

Usage:

```css
/* Button.module.css */
@value primaryColor, borderRadius from './colors.module.css';

.button {
  background: primaryColor;
  border-radius: borderRadius;
  padding: 10px 20px;
}
```

### Computed Values

```css
/* spacing.module.css */
@value baseUnit: 8px;
@value small: calc(baseUnit / 2);    /* 4px */
@value medium: baseUnit;              /* 8px */
@value large: calc(baseUnit * 2);    /* 16px */
@value xlarge: calc(baseUnit * 3);   /* 24px */
```

### Media Query Values

```css
/* breakpoints.module.css */
@value small: (max-width: 599px);
@value medium: (min-width: 600px) and (max-width: 1023px);
@value large: (min-width: 1024px);
```

```css
/* Component.module.css */
@value small, large from './breakpoints.module.css';

.container {
  padding: 10px;
}

@media small {
  .container {
    padding: 5px;
  }
}

@media large {
  .container {
    padding: 20px;
  }
}
```

### @value vs CSS Custom Properties

| Feature | @value | CSS Custom Properties |
|---------|---------|----------------------|
| Scope | File/module | Global or scoped |
| Dynamic | No (build-time) | Yes (runtime) |
| Browser Support | All (compiled away) | Modern browsers |
| JavaScript Access | No | Yes (getComputedStyle) |
| Media Queries | Yes | No |
| Inheritance | No | Yes |
| Performance | Better (static) | Good (native) |

**Use @value when:**
- Values don't change at runtime
- Need build-time optimization
- Working with older browsers

**Use CSS Custom Properties when:**
- Values change dynamically (themes, states)
- Need JavaScript interaction
- Want inheritance/cascade

## Naming Collision Prevention

### Automatic Hashing

```css
/* Button.module.css */
.button { }
```

Different hash generation strategies:

```javascript
// webpack.config.js
{
  loader: 'css-loader',
  options: {
    modules: {
      // Development: readable names
      localIdentName: '[path][name]__[local]--[hash:base64:5]',
      // Example: components-Button__button--2Rfj3
      
      // Production: minimal names
      localIdentName: '[hash:base64:8]',
      // Example: a1b2c3d4
    }
  }
}
```

### Naming Conventions

While collision is prevented automatically, follow conventions for clarity:

```css
/* Card.module.css */

/* Components */
.card { }
.cardHeader { }
.cardBody { }
.cardFooter { }

/* Modifiers (BEM-style) */
.cardFeatured { }
.cardCompact { }

/* States */
.cardActive { }
.cardDisabled { }

/* Elements */
.title { }
.subtitle { }
.image { }
.button { }
```

**CamelCase vs kebab-case:**

```css
/* Camelcase (recommended for JS) */
.cardHeader { }    /* Access: styles.cardHeader */

/* Kebab-case (needs bracket notation) */
.card-header { }   /* Access: styles['card-header'] */
```

### File Naming

```
Component.module.css    (Recommended)
Component.module.scss
Component.module.less

component.module.css    (Alternative)
component.mod.css       (Some setups)
```

## Build Tool Integration

### Webpack

```javascript
// webpack.config.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.module\.css$/,
        use: [
          'style-loader',
          {
            loader: 'css-loader',
            options: {
              modules: {
                localIdentName: '[name]__[local]___[hash:base64:5]',
              },
            },
          },
        ],
      },
      // Regular CSS (non-modules)
      {
        test: /\.css$/,
        exclude: /\.module\.css$/,
        use: ['style-loader', 'css-loader'],
      },
    ],
  },
};
```

With SCSS:

```javascript
{
  test: /\.module\.scss$/,
  use: [
    'style-loader',
    {
      loader: 'css-loader',
      options: {
        modules: true,
      },
    },
    'sass-loader',
  ],
}
```

### Vite

```javascript
// vite.config.js
export default {
  css: {
    modules: {
      // Hash pattern
      generateScopedName: '[name]__[local]___[hash:base64:5]',
      
      // Or custom function
      generateScopedName: (name, filename, css) => {
        // Custom logic
        return `myprefix_${name}_${hash(css)}`;
      },
      
      // Localize @keyframes names
      localsConvention: 'camelCase', // or 'dashes', 'camelCaseOnly'
    },
  },
};
```

### Next.js

Built-in support, zero config:

```jsx
// pages/index.js
import styles from './index.module.css';

export default function Home() {
  return <div className={styles.container}>Hello</div>;
}
```

Configuration (optional):

```javascript
// next.config.js
module.exports = {
  webpack(config) {
    config.module.rules.forEach((rule) => {
      if (rule.oneOf) {
        rule.oneOf.forEach((oneOf) => {
          if (oneOf.test && oneOf.test.toString().includes('module')) {
            oneOf.use.forEach((use) => {
              if (use.loader && use.loader.includes('css-loader')) {
                use.options.modules.mode = 'local';
              }
            });
          }
        });
      }
    });
    return config;
  },
};
```

### Create React App

Built-in support:

```jsx
// File: Button.module.css
.button { }

// File: Button.jsx
import styles from './Button.module.css';
<button className={styles.button}>Click</button>
```

### Parcel

Automatic support for `.module.css` files:

```jsx
import styles from './Component.module.css';
```

## Usage with React

### Basic Component

```css
/* Button.module.css */
.button {
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: 600;
}

.primary {
  composes: button;
  background: #007bff;
  color: white;
}

.secondary {
  composes: button;
  background: #6c757d;
  color: white;
}

.disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
```

```jsx
// Button.jsx
import styles from './Button.module.css';

function Button({ variant = 'primary', disabled, children }) {
  return (
    <button 
      className={styles[variant]}
      disabled={disabled}
    >
      {children}
    </button>
  );
}

export default Button;
```

### Conditional Classes

```jsx
import styles from './Card.module.css';

function Card({ featured, active }) {
  // Method 1: Template literal
  const className = `
    ${styles.card}
    ${featured ? styles.featured : ''}
    ${active ? styles.active : ''}
  `.trim();
  
  // Method 2: Array join
  const className2 = [
    styles.card,
    featured && styles.featured,
    active && styles.active,
  ].filter(Boolean).join(' ');
  
  // Method 3: classnames library
  import classNames from 'classnames';
  const className3 = classNames(
    styles.card,
    {
      [styles.featured]: featured,
      [styles.active]: active,
    }
  );
  
  return <div className={className}>Card</div>;
}
```

### With classnames/clsx

```bash
npm install classnames
# or
npm install clsx  # Smaller alternative
```

```jsx
import styles from './Component.module.css';
import classNames from 'classnames';

function Component({ primary, large, disabled }) {
  return (
    <button
      className={classNames(
        styles.button,
        {
          [styles.primary]: primary,
          [styles.large]: large,
          [styles.disabled]: disabled,
        }
      )}
    >
      Click me
    </button>
  );
}
```

### Accessing Nested Classes

```css
/* Component.module.css */
.container { }
.container .title { }
```

```jsx
// Won't work - .title is not exposed
<h2 className={styles.title}>Title</h2>

// Make title a top-level class
.title { }

// Then nest in CSS if needed
.container .title { }
```

### TypeScript Support

```typescript
// Button.module.css.d.ts (auto-generated)
declare const styles: {
  readonly button: string;
  readonly primary: string;
  readonly secondary: string;
  readonly disabled: string;
};
export default styles;
```

```tsx
// Button.tsx
import styles from './Button.module.css';

// TypeScript knows available classes
<button className={styles.button}>  // ✓ OK
<button className={styles.invalid}> // ✗ Error
```

Enable auto-generation:

```bash
npm install -D typescript-plugin-css-modules
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "plugins": [{ "name": "typescript-plugin-css-modules" }]
  }
}
```

## Usage with Vue

### Vue 3 (SFC)

```vue
<template>
  <div :class="$style.container">
    <h1 :class="$style.title">Hello</h1>
    <button :class="[$style.button, $style.primary]">
      Click me
    </button>
  </div>
</template>

<style module>
.container {
  padding: 20px;
}

.title {
  font-size: 2rem;
}

.button {
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
}

.primary {
  composes: button;
  background: blue;
  color: white;
}
</style>
```

### Named Modules

```vue
<template>
  <div :class="classes.container">
    <button :class="classes.button">Click</button>
  </div>
</template>

<style module="classes">
.container { }
.button { }
</style>
```

### External Module Files

```vue
<script>
import styles from './Component.module.css';

export default {
  computed: {
    classes() {
      return styles;
    }
  }
};
</script>

<template>
  <div :class="classes.container">
    Content
  </div>
</template>
```

### Conditional Classes

```vue
<template>
  <button 
    :class="[
      $style.button,
      { [$style.active]: isActive },
      { [$style.disabled]: isDisabled }
    ]"
  >
    Click me
  </button>
</template>

<style module>
.button { }
.active { }
.disabled { }
</style>
```

## Best Practices

### 1. One Component, One Module

```
components/
├── Button/
│   ├── Button.jsx
│   ├── Button.module.css
│   └── Button.test.js
├── Card/
│   ├── Card.jsx
│   ├── Card.module.css
│   └── Card.test.js
```

### 2. Shared Styles via Composition

```css
/* shared/base.module.css */
.resetButton {
  background: none;
  border: none;
  padding: 0;
  cursor: pointer;
}

.truncate {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
```

```css
/* Button.module.css */
.button {
  composes: resetButton from '../shared/base.module.css';
  padding: 10px 20px;
  background: blue;
}
```

### 3. Use @value for Constants

```css
/* constants.module.css */
@value colors: "./colors.module.css";
@value spacing: "./spacing.module.css";
@value primaryColor from colors;
@value basePadding from spacing;
```

### 4. Avoid Deep Nesting

```css
/* Bad */
.container .header .nav .item .link { }

/* Good - flat classes */
.container { }
.header { }
.navItem { }
.navLink { }
```

### 5. Mix Global and Local Strategically

```css
/* Component.module.css */

/* Local for component-specific */
.card {
  background: white;
  border-radius: 8px;
}

/* Global for third-party integration */
:global {
  .card :global(.react-datepicker) {
    z-index: 1000;
  }
}
```

### 6. Naming Conventions

```css
/* Clear, descriptive names */
.userCard { }
.userCardHeader { }
.userCardTitle { }
.userCardAvatar { }

/* State modifiers */
.userCardActive { }
.userCardDisabled { }

/* Variant modifiers */
.userCardCompact { }
.userCardFeatured { }
```

## Common Patterns

### Theme Switching

```css
/* Component.module.css */
@value lightBg: #ffffff;
@value darkBg: #1a1a1a;

.container {
  background: lightBg;
  color: #333;
}

.containerDark {
  composes: container;
  background: darkBg;
  color: #fff;
}
```

```jsx
function Component({ theme }) {
  const containerClass = theme === 'dark' 
    ? styles.containerDark 
    : styles.container;
  
  return <div className={containerClass}>Content</div>;
}
```

### Responsive Styles

```css
/* Component.module.css */
.grid {
  display: grid;
  gap: 20px;
  grid-template-columns: 1fr;
}

@media (min-width: 768px) {
  .grid {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (min-width: 1024px) {
  .grid {
    grid-template-columns: repeat(3, 1fr);
  }
}
```

### Animation

```css
/* Component.module.css */
@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.card {
  animation: fadeIn 0.3s ease-out;
}

/* keyframes are also scoped automatically */
```

## Limitations and Workarounds

### 1. Can't Use Dynamic Class Names

```jsx
// Won't work
const variant = 'primary';
<button className={styles.button-${variant}}>

// Workaround: Map object
const variants = {
  primary: styles.buttonPrimary,
  secondary: styles.buttonSecondary,
};
<button className={variants[variant]}>
```

### 2. Third-Party Component Integration

```jsx
// Problem: Third-party expects string class name
<ThirdPartyComponent className={styles.card} />
// Passes: "Card_card__a1b2c"

// Workaround: Component should accept and apply
function ThirdPartyWrapper({ className }) {
  return <div className={className}>...</div>;
}
```

### 3. Global Styles Override

```css
/* Component.module.css */
.button {
  background: blue;
}

/* global.css */
button {
  background: red !important; /* Overrides module */
}
```

Workaround: Increase specificity or use !important in modules.

## Migration Strategy

### From Traditional CSS

1. **Rename files**: `Component.css` → `Component.module.css`

2. **Update imports**:
```jsx
// Before
import './Component.css';
<div className="card">

// After
import styles from './Component.module.css';
<div className={styles.card}>
```

3. **Handle global styles**:
```css
/* Keep global styles in separate file */
/* global.css */
body { }
a { }

/* Convert component styles to modules */
/* Component.module.css */
.container { }
```

4. **Gradual migration**: Migrate component by component

## Performance Considerations

### Build-Time Optimization

CSS Modules are processed at build time:
- No runtime overhead
- Classes hashed and optimized
- Dead code elimination possible

### Bundle Size

```
Traditional: All CSS loaded upfront
Modules: Can code-split CSS per component
```

```javascript
// Webpack: CSS code-splitting
import(/* webpackChunkName: "dashboard" */ './Dashboard')
  .then(module => {
    // Dashboard.module.css loaded with component
  });
```

### Comparison

| Approach | Runtime Cost | Bundle Size | Flexibility |
|----------|--------------|-------------|-------------|
| CSS Modules | None | Small | Medium |
| CSS-in-JS | High | Medium-Large | High |
| Traditional CSS | None | Large | Low |
| Utility-First | None | Small | Medium |

## Conclusion

CSS Modules provide a sweet spot between traditional CSS and CSS-in-JS:

**Advantages:**
- Automatic scoping (no naming collisions)
- No runtime overhead
- Works with existing CSS knowledge
- Excellent tooling support
- Composable styles
- Type-safe (with TypeScript)

**Trade-offs:**
- Build step required
- Less dynamic than CSS-in-JS
- Requires component framework integration
- Learning curve for composition patterns

**Best For:**
- Component-based applications (React, Vue)
- Teams comfortable with CSS
- Projects needing scoped styles without runtime cost
- Design systems with shared styles

CSS Modules excel when you want the benefits of scoped styling without the complexity or runtime cost of CSS-in-JS, making them an excellent choice for modern component-based applications.
