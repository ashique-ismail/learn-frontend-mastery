# React Compiler

## The Idea

**In plain English:** React Compiler is a tool that reads your code before it runs and automatically adds performance tricks (called memoization) so your app doesn't redo work it already did. Think of it as a smart editor that rewrites your code to be faster, without you having to do it yourself.

**Real-world analogy:** Imagine a chef who writes a meal-prep checklist before a busy dinner service. Instead of re-chopping the same vegetables every time a dish is ordered, the checklist reminds the kitchen: "if we already chopped onions and nothing changed, use those — don't chop again."

- The checklist = the compiler analyzing your code before it runs
- The chopped vegetables = the already-calculated result stored in memory (memoized value)
- The rule "only re-chop if the recipe changed" = the compiler's logic for deciding when to reuse or recompute a value

---

## Overview

React Compiler (previously called "React Forget") is a build-time optimizing compiler that automatically memoizes React components and hooks. It eliminates the need to manually write `useMemo`, `useCallback`, and `React.memo` for most cases — the compiler infers where memoization is necessary and inserts it at compile time.

It shipped in React 19 and is available as an opt-in for React 17+. Meta has run it in production on Instagram.com since 2023.

---

## The Problem It Solves

Without the compiler, developers must manually prevent unnecessary re-renders:

```jsx
// Before React Compiler — manual memoization
function ProductList({ products, filters, onSelect }) {
  // Must remember to memoize every derived value
  const filtered = useMemo(
    () => products.filter(p => p.category === filters.category),
    [products, filters.category]
  );

  // Must memoize callbacks to prevent child re-renders
  const handleSelect = useCallback(
    (id) => onSelect(id),
    [onSelect]
  );

  return (
    // Must wrap in React.memo to prevent re-renders on parent updates
    <MemoizedList items={filtered} onSelect={handleSelect} />
  );
}

const MemoizedList = React.memo(function List({ items, onSelect }) {
  return items.map(item => (
    <Item key={item.id} item={item} onSelect={onSelect} />
  ));
});
```

This is fragile — miss a dependency, add the wrong one, or forget a `React.memo`, and you get either stale data or excessive re-renders.

---

## What the Compiler Does

The compiler analyzes your component's code statically and inserts memoization where it's safe and beneficial. The output is equivalent to manually written `useMemo`/`useCallback`, but generated automatically.

```jsx
// You write this:
function ProductList({ products, filters, onSelect }) {
  const filtered = products.filter(p => p.category === filters.category);

  return <List items={filtered} onSelect={onSelect} />;
}

// Compiler outputs (conceptually):
function ProductList({ products, filters, onSelect }) {
  const $ = useMemoCache(3);

  let filtered;
  if ($[0] !== products || $[1] !== filters.category) {
    filtered = products.filter(p => p.category === filters.category);
    $[0] = products;
    $[1] = filters.category;
    $[2] = filtered;
  } else {
    filtered = $[2];
  }

  return <List items={filtered} onSelect={onSelect} />;
}
```

The compiler uses a fine-grained cache keyed on the specific inputs that each computation depends on — not component-level memoization like `React.memo`, but expression-level memoization.

---

## Setup

```bash
npm install -D babel-plugin-react-compiler
# or via eslint plugin
npm install -D eslint-plugin-react-compiler
```

```javascript
// babel.config.js
module.exports = {
  plugins: [
    ['babel-plugin-react-compiler', {
      // Target React 17/18 with the runtime package
      // (React 19 has it built-in)
      runtimeModule: 'react-compiler-runtime',
    }],
  ],
};

// vite.config.ts (with @vitejs/plugin-react)
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [
    react({
      babel: {
        plugins: [['babel-plugin-react-compiler']],
      },
    }),
  ],
});

// Next.js (13.5.1+)
// next.config.js
const nextConfig = {
  experimental: {
    reactCompiler: true,
  },
};
```

For React 17/18 (without built-in compiler runtime):
```bash
npm install react-compiler-runtime
```

---

## Rules the Compiler Enforces

The compiler works **only** with code that follows the Rules of React. It detects violations and skips memoizing components it can't safely optimize.

**What breaks the compiler:**

```jsx
// ❌ Mutating props or state
function Bad({ items }) {
  items.push('new');  // mutation — compiler won't optimize this component
  return <List items={items} />;
}

// ❌ Calling hooks conditionally
function Bad({ show }) {
  if (show) {
    const data = useFetch('/api'); // conditional hook
  }
}

// ❌ Reading ref.current during render
function Bad({ ref }) {
  return <div>{ref.current}</div>; // ref reads during render break memoization
}
```

**What the compiler handles:**

```jsx
// ✓ Derived values
const total = items.reduce((sum, item) => sum + item.price, 0);

// ✓ Event handlers
const handleClick = (id) => dispatch({ type: 'SELECT', id });

// ✓ JSX expressions
const header = <Header title={title} subtitle={subtitle} />;

// ✓ Conditional rendering (returns only, not hooks)
if (loading) return <Spinner />;
```

---

## ESLint Plugin: `eslint-plugin-react-compiler`

The plugin flags code that would prevent the compiler from optimizing a component:

```bash
npm install -D eslint-plugin-react-compiler
```

```javascript
// .eslintrc.js
module.exports = {
  plugins: ['react-compiler'],
  rules: {
    'react-compiler/react-compiler': 'error',
  },
};
```

Running this before enabling the compiler shows exactly which components need to be fixed first.

---

## Opt-Out Escape Hatch

When a component genuinely can't be compiled (third-party code, intentional mutation), opt it out with a directive:

```jsx
function ThirdPartyWrapper() {
  'use no memo'; // tells compiler to skip this function

  // ... code that intentionally breaks rules
}
```

---

## What Changes (and What Doesn't)

**You no longer need:**
- `React.memo` wrapping — the compiler memoizes at the JSX element level
- `useMemo` for derived values in renders
- `useCallback` for stable function references passed to children

**You still need:**
- `useRef` for mutable values that shouldn't trigger re-renders
- `useEffect` for side effects
- `useState` / `useReducer` for state
- Manual `useMemo` for **very expensive computations** — the compiler is conservative and won't always memoize everything
- `useDeferredValue` / `useTransition` for concurrency control — these are about scheduling, not memoization

---

## Compiler vs Manual Memoization

| Scenario | Manual | Compiler |
|---|---|---|
| Simple derived values | `useMemo` required | Automatic |
| Callbacks passed to children | `useCallback` required | Automatic |
| Component re-render prevention | `React.memo` required | Automatic |
| Stale closure bugs | You can introduce them | Compiler catches them |
| Complex side-effect dependencies | `useEffect` dep array | No change |
| Intentional mutation | Works (but defeats memoization) | Must opt out |
| Context consumers | Still re-render on context change | No change — use selectors |

---

## Current Status (2024–2025)

- **React 19**: Compiler is available as opt-in, runtime built-in
- **React 17/18**: Works with `react-compiler-runtime` polyfill
- **Meta**: Running in production on Instagram, facebook.com
- **Next.js 15**: First-class support via `experimental.reactCompiler`
- **Not yet**: Not enabled by default in create-react-app or Vite templates — it's still opt-in

---

## Verifying Compiler Output

React DevTools shows a `Memo ✨` badge on components the compiler has optimized.

```bash
# Check what the compiler would do without modifying your code
npx react-compiler-healthcheck

# Output:
# Successfully compiled 87 out of 90 components.
# 3 components skipped (see --verbose for details)
```

---

## Interview Questions

**Q: What is React Compiler and why was it created?**  
A: React Compiler is a build-time Babel/SWC plugin that automatically inserts `useMemo`/`useCallback`-equivalent memoization into components. It was created because manual memoization is error-prone — developers forget it, over-apply it, or introduce stale closure bugs. The compiler statically analyzes data flow and inserts correct memoization automatically.

**Q: Does React Compiler replace `useMemo` and `useCallback` entirely?**  
A: For most cases, yes. The compiler infers the same memoization a developer would write manually. You still need `useMemo` for computations that are genuinely expensive and the compiler didn't memoize, and `useCallback` is effectively redundant. `useEffect`, `useState`, `useRef` and concurrency hooks (`useTransition`, `useDeferredValue`) are unaffected.

**Q: What happens to components the compiler can't optimize?**  
A: The compiler skips them silently (unless you use the ESLint plugin, which flags them). Components that mutate props/state, call hooks conditionally, or read `ref.current` during render are excluded. You can also explicitly opt out with `'use no memo'`.

**Q: Is React Compiler safe to enable on an existing codebase?**  
A: Generally yes, if your code follows the Rules of React. Run `eslint-plugin-react-compiler` first to identify violations. The compiler won't silently break correct code — it either optimizes it correctly or skips it. The main risk is surfacing bugs that were hidden by accidental re-renders.
