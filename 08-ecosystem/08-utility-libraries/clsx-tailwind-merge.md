# clsx + tailwind-merge

## The Problem They Solve

When building components with utility-first CSS (Tailwind), you need to:
1. Conditionally apply classes (`clsx`)
2. Handle conflicts when the same property appears multiple times (`tailwind-merge`)

---

## `clsx` — Conditional Class Composition

```ts
import clsx from 'clsx';

// String
clsx('foo', 'bar');          // 'foo bar'

// Conditional: false/null/undefined values are excluded
clsx('foo', isActive && 'active');     // 'foo' or 'foo active'
clsx({ active: isActive, disabled: isDisabled }); // 'active' (only truthy keys)

// Array support
clsx(['foo', 'bar', isActive && 'active']); // 'foo bar active'

// Mixed
clsx(
  'base-class',
  { 'is-active': isActive, 'is-disabled': isDisabled },
  ['extra-class', condition && 'conditional'],
  undefined, null, false  // ignored
);
```

Similar to the older `classnames` package (nearly identical API).

---

## `tailwind-merge` — Conflict Resolution

Without `tailwind-merge`, passing conflicting Tailwind classes results in both being in the class list. CSS specificity decides the winner (usually the first-declared in the CSS file, NOT the order in the class string).

```html
<!-- ❌ Both classes in the list: which wins depends on CSS source order, not your intention -->
<div class="p-4 p-8">...</div>
<!-- p-4 and p-8 both apply — browser picks based on stylesheet order (usually p-4) -->

<!-- ✅ tailwind-merge: resolves to only p-8 -->
import { twMerge } from 'tailwind-merge';
twMerge('p-4 p-8')  // → 'p-8'
```

`tailwind-merge` understands Tailwind's class semantics and keeps only the last class for each CSS property:

```ts
import { twMerge } from 'tailwind-merge';

twMerge('p-4 p-8');            // 'p-8' (both set padding — last wins)
twMerge('px-4 p-8');           // 'p-8' (p-8 is more specific, overrides px-4)
twMerge('text-sm text-lg');    // 'text-lg'
twMerge('bg-red-500 bg-blue-500'); // 'bg-blue-500'
twMerge('w-8 w-12 w-4');       // 'w-4' (last wins)

// Non-conflicting classes are kept
twMerge('p-4 m-4');            // 'p-4 m-4' (different properties)
twMerge('bg-red-500 text-white'); // 'bg-red-500 text-white'
```

---

## The `cn` Utility (Industry Standard)

The standard pattern in most Tailwind projects (used by shadcn/ui, etc.):

```ts
// lib/utils.ts
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

```tsx
// Usage in components
function Button({ variant, className, disabled, children, ...props }: ButtonProps) {
  return (
    <button
      className={cn(
        // Base classes
        'px-4 py-2 rounded-md font-medium transition-colors',
        // Variant-dependent classes
        variant === 'primary' && 'bg-blue-600 text-white hover:bg-blue-700',
        variant === 'ghost' && 'bg-transparent hover:bg-gray-100',
        // State-dependent
        disabled && 'opacity-50 cursor-not-allowed',
        // Consumer override — tailwind-merge ensures this wins over conflicting base classes
        className
      )}
      disabled={disabled}
      {...props}
    >
      {children}
    </button>
  );
}

// Consumer can override padding
<Button className="px-8">Wide Button</Button>
// Result: 'py-2 rounded-md font-medium transition-colors bg-blue-600 text-white hover:bg-blue-700 px-8'
// (px-4 from base is removed, px-8 from className wins)
```

---

## Custom Tailwind Classes Configuration

If you extend Tailwind with custom utilities, tell `tailwind-merge` about them:

```ts
import { extendTailwindMerge } from 'tailwind-merge';

const twMerge = extendTailwindMerge({
  extend: {
    classGroups: {
      'text-shadow': ['text-shadow-sm', 'text-shadow-md', 'text-shadow-lg'],
    },
  },
});

// Now these conflict correctly
twMerge('text-shadow-sm text-shadow-lg'); // 'text-shadow-lg'
```

---

## Performance Note

`tailwind-merge` builds a class group map and LRU cache internally. For components that render frequently, memoize the result if the class string doesn't change:

```tsx
// The cn() call itself is fast, but for heavily-rendered components:
const classes = useMemo(
  () => cn('base-class', variant === 'primary' && 'primary-class'),
  [variant]
);
```

---

## Common Interview Questions

**Q: Why use both `clsx` and `tailwind-merge` instead of just one?**
`clsx` doesn't understand Tailwind semantics — it just joins strings. It would output `p-4 p-8` without resolving the conflict. `tailwind-merge` alone doesn't handle conditional classes elegantly. Together: `clsx` handles conditions, `twMerge` handles conflicts.

**Q: Why does `tailwind-merge` need to understand Tailwind specifically?**
Regular string deduplication would keep only unique strings — but `p-4` and `p-8` are different strings that set the same CSS property. `tailwind-merge` has knowledge of Tailwind's naming conventions to group conflicting classes.

**Q: Does `tailwind-merge` work with Tailwind v4?**
`tailwind-merge` v3+ includes support for Tailwind CSS v4.
