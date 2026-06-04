# CVA (class-variance-authority)

## What It Does

CVA provides a type-safe API for defining component variants. It returns a function that takes variant props and returns the appropriate class string.

Solves the "ternary hell" problem in utility-first CSS:

```tsx
// Without CVA — gets messy fast
<button className={`
  px-4 py-2 rounded font-medium
  ${variant === 'primary' ? 'bg-blue-600 text-white hover:bg-blue-700' :
    variant === 'secondary' ? 'bg-gray-100 text-gray-900 hover:bg-gray-200' :
    'bg-transparent hover:bg-gray-100'}
  ${size === 'sm' ? 'text-sm h-8' : size === 'lg' ? 'text-lg h-12' : 'h-10'}
`}>
```

---

## Basic Usage

```tsx
import { cva, type VariantProps } from 'class-variance-authority';

const buttonVariants = cva(
  // Base classes (always applied)
  'inline-flex items-center justify-center rounded-md font-medium transition-colors disabled:opacity-50',
  {
    variants: {
      variant: {
        primary: 'bg-blue-600 text-white hover:bg-blue-700',
        secondary: 'bg-gray-100 text-gray-900 hover:bg-gray-200',
        ghost: 'hover:bg-gray-100',
        destructive: 'bg-red-600 text-white hover:bg-red-700',
        outline: 'border border-gray-300 bg-transparent hover:bg-gray-50',
      },
      size: {
        sm: 'h-8 px-3 text-sm',
        md: 'h-10 px-4',
        lg: 'h-12 px-6 text-lg',
        icon: 'h-10 w-10',
      },
    },
    defaultVariants: {
      variant: 'primary',
      size: 'md',
    },
  }
);

// TypeScript interface automatically inferred
interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {}

function Button({ variant, size, className, ...props }: ButtonProps) {
  return (
    <button
      className={buttonVariants({ variant, size, className })}
      {...props}
    />
  );
}

// Usage
<Button variant="secondary" size="sm">Cancel</Button>
<Button variant="destructive">Delete</Button>
<Button>Submit</Button>  {/* uses defaultVariants */}
```

---

## Compound Variants

Apply classes when multiple variants combine:

```tsx
const alertVariants = cva(
  'flex items-start gap-3 rounded-lg p-4',
  {
    variants: {
      variant: {
        info: 'bg-blue-50 text-blue-900',
        success: 'bg-green-50 text-green-900',
        warning: 'bg-yellow-50 text-yellow-900',
        error: 'bg-red-50 text-red-900',
      },
      withIcon: {
        true: 'pl-12',  // extra left padding when icon is present
        false: '',
      },
    },
    compoundVariants: [
      // Extra styles when both conditions match
      {
        variant: 'error',
        withIcon: true,
        class: 'border border-red-200',
      },
    ],
    defaultVariants: {
      variant: 'info',
      withIcon: false,
    },
  }
);
```

---

## Combining with `tailwind-merge` and `clsx`

```ts
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

**Why both?**
- `clsx` handles conditional class values and arrays
- `tailwind-merge` resolves Tailwind conflicts (e.g., `p-2` + `p-4` → keeps only `p-4`)

Without `tailwind-merge`, passing `className="p-8"` to a button with default `px-4 py-2` would have both in the class list, and the original would win (CSS order, not the override):

```tsx
// ✅ tailwind-merge ensures className prop overrides defaults
function Button({ className, ...props }: ButtonProps) {
  return (
    <button
      className={cn(buttonVariants({ ...props }), className)}
      {...props}
    />
  );
}

<Button className="px-8">Wide Button</Button>
// Without twMerge: 'px-4 py-2 ... px-8' — px-4 wins (comes first in CSS)
// With twMerge: 'py-2 ... px-8' — conflicts resolved, px-8 wins
```

---

## TypeScript Inference

```ts
// VariantProps extracts the variant types
type ButtonVariants = VariantProps<typeof buttonVariants>;
// {
//   variant?: 'primary' | 'secondary' | 'ghost' | 'destructive' | 'outline' | null | undefined
//   size?: 'sm' | 'md' | 'lg' | 'icon' | null | undefined
// }

// Use in component props
interface ButtonProps extends ButtonVariants {
  children: React.ReactNode;
  onClick?: () => void;
}
```

---

## Common Interview Questions

**Q: What does `className` in the CVA call do?**
CVA accepts an optional `class`/`className` key in the options object that gets appended after the variant classes. This allows consumers to add extra classes: `buttonVariants({ variant: 'primary', className: 'w-full' })`.

**Q: Can you use CVA without Tailwind?**
Yes — CVA just concatenates strings. It works with any CSS-in-class approach (CSS Modules with `clsx`, inline CSS classes, etc.).

**Q: What's the alternative to CVA?**
`class-variance-authority` vs `cva` from `@vanilla-extract/recipes` (for vanilla-extract), vs manually using `clsx` with if/else chains. For shadcn/ui-style projects, CVA is the standard.
