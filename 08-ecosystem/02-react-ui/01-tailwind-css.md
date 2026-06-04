# Tailwind CSS v3/v4

## What Is It

Tailwind CSS is a utility-first CSS framework. Instead of writing CSS classes with semantic names, you compose small utility classes directly in HTML/JSX.

---

## Core Concept: Utilities

```html
<!-- Without Tailwind -->
<button class="btn btn-primary btn-large">Submit</button>

<!-- With Tailwind — describe the appearance directly -->
<button class="bg-blue-600 hover:bg-blue-700 text-white px-6 py-3 rounded-lg font-semibold text-lg transition-colors">
  Submit
</button>
```

---

## JIT (Just-In-Time) Compiler

Since Tailwind v2.2 (now default): generates only the CSS classes you actually use, scanning your source files:

```js
// tailwind.config.ts
export default {
  content: [
    './src/**/*.{ts,tsx}',
    './components/**/*.{ts,tsx}',
  ],
  // ↑ JIT scans these files and generates CSS for classes found
};
```

Result: production CSS is typically 5-20 KB instead of the full utility bundle (~4 MB).

---

## Configuration

```ts
// tailwind.config.ts
import type { Config } from 'tailwindcss';

const config: Config = {
  content: ['./src/**/*.{ts,tsx,html}'],
  darkMode: 'class', // or 'media'
  theme: {
    extend: {  // extend (don't replace) default theme
      colors: {
        brand: {
          50: '#eff6ff',
          500: '#3b82f6',
          900: '#1e3a8a',
        },
      },
      fontFamily: {
        sans: ['Inter', 'sans-serif'],
      },
      spacing: {
        18: '4.5rem',
        128: '32rem',
      },
      animation: {
        'spin-slow': 'spin 3s linear infinite',
      },
    },
  },
  plugins: [
    require('@tailwindcss/typography'),  // prose styles
    require('@tailwindcss/forms'),        // form reset
    require('@tailwindcss/aspect-ratio'),
  ],
};
```

---

## Custom Utilities and Components

```css
/* globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Custom component (avoids duplication for repeated patterns) */
@layer components {
  .btn {
    @apply inline-flex items-center justify-center rounded-md font-medium transition-colors;
    @apply focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring;
    @apply disabled:pointer-events-none disabled:opacity-50;
  }
}

/* Custom utility */
@layer utilities {
  .text-balance {
    text-wrap: balance;
  }
  .scrollbar-hide {
    -ms-overflow-style: none;
    scrollbar-width: none;
    &::-webkit-scrollbar { display: none; }
  }
}
```

---

## Dark Mode

```html
<!-- With darkMode: 'class' -->
<html class="dark">
  <body>
    <p class="text-gray-900 dark:text-gray-100">
      Light mode: dark text. Dark mode: light text.
    </p>
  </body>
</html>
```

```tsx
// Toggle in React
function ThemeToggle() {
  const [isDark, setIsDark] = useState(false);

  useEffect(() => {
    document.documentElement.classList.toggle('dark', isDark);
  }, [isDark]);
}
```

---

## Responsive Design

Mobile-first breakpoints (apply at `sm:` and up):

```html
<div class="
  grid grid-cols-1        /* mobile: 1 column */
  sm:grid-cols-2          /* ≥640px: 2 columns */
  md:grid-cols-3          /* ≥768px: 3 columns */
  lg:grid-cols-4          /* ≥1024px: 4 columns */
  gap-4
">
```

---

## Tailwind v4 New Features (2025)

```css
/* CSS-first configuration — no tailwind.config.ts needed */
@import "tailwindcss";

@theme {
  --color-brand-500: #3b82f6;
  --font-family-sans: "Inter", sans-serif;
  --spacing-18: 4.5rem;
}

/* Use CSS variables in utilities: text-[--color-brand-500] */
```

Key changes in v4:
- Built on Lightning CSS (Rust-based) — 10x faster builds
- CSS-first configuration with `@theme`
- Automatic content detection (no `content` array needed in many cases)
- Container queries built-in (`@container`, `@sm:`, `@lg:`)
- Cascade layers by default

---

## Common Interview Questions

**Q: How does Tailwind avoid style conflicts between components?**
Each utility class does one thing (`p-4` is always `padding: 1rem`). There's no cascade conflict — specificity is always `(0,1,0)` for a class. `tailwind-merge` handles conflicts when you need to override.

**Q: What's the difference between `extend` and overriding in the theme config?**
`theme.extend.colors` adds to the default color palette. `theme.colors` replaces it entirely (losing `blue`, `red`, etc.). Always use `extend` unless you intentionally want to replace the defaults.

**Q: How do you handle Tailwind in a component library?**
Either: (1) Publish components as source code (shadcn/ui approach), or (2) set up Tailwind in the library with a prefix (`tw-`) to avoid conflicts with consumers' Tailwind config.
