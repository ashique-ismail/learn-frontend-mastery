# vanilla-extract

## The Idea

**In plain English:** vanilla-extract is a tool that lets you write your website's visual styles (colors, sizes, spacing) using TypeScript — a programming language — and then automatically converts them into regular CSS (the standard language browsers use to style web pages) before your site ever loads, so there is no extra work happening while visitors use your site.

**Real-world analogy:** Imagine a costume designer who sketches every outfit on paper before the show starts. Each sketch is labeled and filed away. When a performer needs a costume, they just grab the labeled item from the rack — no sewing happens during the live performance.

- The sketch = the style written in a `.css.ts` TypeScript file
- The labeled costume on the rack = the plain CSS class name produced at build time
- Grabbing the costume during the show = the component using the class name string at runtime (no extra work needed)

---

## What It Is

vanilla-extract is a **zero-runtime** CSS-in-TypeScript library. Styles are written in `.css.ts` files, compiled to plain CSS at build time — no runtime style injection, no stylesheet in JS bundles.

Think: CSS Modules with TypeScript type-checking + constraint-based design tokens.

---

## Core API: `style()`

```ts
// styles.css.ts
import { style } from '@vanilla-extract/css';

export const button = style({
  backgroundColor: 'blue',
  color: 'white',
  padding: '8px 16px',
  borderRadius: '4px',
  ':hover': {
    backgroundColor: 'darkblue',
  },
  '@media': {
    '(max-width: 768px)': {
      padding: '6px 12px',
    },
  },
});
```

```tsx
// Button.tsx
import { button } from './styles.css';

function Button({ children }) {
  return <button className={button}>{children}</button>;
}
```

The `button` variable is a **class name string** at runtime — the CSS is emitted to a `.css` file at build time.

---

## CSS Variables (Contract/Create)

```ts
// theme.css.ts
import { createTheme, createThemeContract } from '@vanilla-extract/css';

// Contract: defines the shape (nullable = theme must fill these in)
const themeVars = createThemeContract({
  color: {
    primary: null,
    secondary: null,
    background: null,
    text: null,
  },
  spacing: {
    small: null,
    medium: null,
    large: null,
  },
});

// Theme 1: light
export const lightTheme = createTheme(themeVars, {
  color: {
    primary: '#3b82f6',
    secondary: '#64748b',
    background: '#ffffff',
    text: '#111827',
  },
  spacing: {
    small: '8px',
    medium: '16px',
    large: '32px',
  },
});

// Theme 2: dark
export const darkTheme = createTheme(themeVars, {
  color: {
    primary: '#60a5fa',
    secondary: '#94a3b8',
    background: '#111827',
    text: '#f9fafb',
  },
  spacing: { small: '8px', medium: '16px', large: '32px' },
});

// Use theme variables in styles
export const card = style({
  backgroundColor: themeVars.color.background,
  color: themeVars.color.text,
  padding: themeVars.spacing.medium,
});
```

---

## `styleVariants()` — Multiple Variants

```ts
import { styleVariants } from '@vanilla-extract/css';

const base = style({ padding: '8px 16px', borderRadius: '4px' });

export const variants = styleVariants({
  primary: [base, { backgroundColor: '#3b82f6', color: 'white' }],
  secondary: [base, { backgroundColor: '#f1f5f9', color: '#111827' }],
  ghost: [base, { backgroundColor: 'transparent', border: '1px solid currentColor' }],
});

// Usage
function Button({ variant = 'primary', children }) {
  return <button className={variants[variant]}>{children}</button>;
}
```

---

## `recipe()` — Compound Variants (sprinkles alternative)

```ts
import { recipe } from '@vanilla-extract/recipes';

export const button = recipe({
  base: {
    display: 'inline-flex',
    alignItems: 'center',
    borderRadius: '4px',
    fontWeight: '600',
    transition: 'background-color 0.2s',
  },
  variants: {
    color: {
      primary: { backgroundColor: '#3b82f6', color: 'white' },
      secondary: { backgroundColor: '#f1f5f9', color: '#111827' },
    },
    size: {
      sm: { padding: '4px 8px', fontSize: '14px' },
      md: { padding: '8px 16px', fontSize: '16px' },
      lg: { padding: '12px 24px', fontSize: '18px' },
    },
  },
  compoundVariants: [
    {
      variants: { color: 'primary', size: 'lg' },
      style: { boxShadow: '0 4px 12px rgba(59, 130, 246, 0.4)' },
    },
  ],
  defaultVariants: {
    color: 'primary',
    size: 'md',
  },
});

// Usage
function Button({ color, size, children }) {
  return <button className={button({ color, size })}>{children}</button>;
}
// button({ color: 'primary', size: 'lg' }) → class name string
```

---

## `sprinkles()` — Constraint-Based Utility Classes

```ts
import { defineProperties, createSprinkles } from '@vanilla-extract/sprinkles';

const properties = defineProperties({
  properties: {
    display: ['none', 'flex', 'block', 'inline'],
    flexDirection: ['row', 'column'],
    padding: {
      small: '4px',
      medium: '8px',
      large: '16px',
    },
    color: {
      primary: '#3b82f6',
      text: '#111827',
    },
    gap: {
      small: '8px',
      medium: '16px',
    },
  },
  // Responsive conditions
  conditions: {
    mobile: {},
    tablet: { '@media': 'screen and (min-width: 768px)' },
    desktop: { '@media': 'screen and (min-width: 1024px)' },
  },
  defaultCondition: 'mobile',
});

export const sprinkles = createSprinkles(properties);

// Usage
<div className={sprinkles({
  display: { mobile: 'block', tablet: 'flex' },
  padding: 'medium',
  gap: 'small',
})}>
```

---

## Build Setup

```bash
npm install -D @vanilla-extract/css @vanilla-extract/vite-plugin
```

```ts
// vite.config.ts
import { vanillaExtractPlugin } from '@vanilla-extract/vite-plugin';

export default defineConfig({
  plugins: [vanillaExtractPlugin()],
});
```

---

## Comparison

| | vanilla-extract | Emotion/styled-components | CSS Modules | Tailwind |
|---|---|---|---|---|
| Runtime overhead | Zero | Runtime injection | Zero | Zero |
| TypeScript | Full type safety | Good | Module types | Class strings |
| Dynamic styles | Via CSS vars | Direct JS | Via vars | Via CVA |
| Bundle | CSS file only | JS + CSS | CSS file | CSS utility file |

---

## Common Interview Questions

**Q: Why "zero runtime"?**
The `.css.ts` files are processed at build time to produce `.css` files. The exported variables are just class name strings. No StyleSheet.inject, no `<style>` tag manipulation at runtime — just plain CSS delivered as a file.

**Q: How do you handle truly dynamic styles (unknown at build time)?**
Use CSS custom properties. Define them at build time (`vars.myVar`), set them at runtime via `element.style.setProperty('--my-var', dynamicValue)`. The CSS logic stays at build time; only the values change at runtime.
