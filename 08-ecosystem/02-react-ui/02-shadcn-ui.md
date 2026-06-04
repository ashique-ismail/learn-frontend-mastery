# shadcn/ui

## What It Is (And What It Isn't)

shadcn/ui is **not** an npm package. It's a collection of component source files you copy into your own project using the CLI. You own the code — it's in your `components/ui/` folder.

Built on: **Radix UI primitives** (accessibility, keyboard navigation, ARIA) + **Tailwind CSS** (styling) + **CVA** (variants).

---

## The Philosophy

Traditional component library: `npm install @ui-library/react` → import component → can't easily customize.

shadcn/ui: Copy the source code into your project → modify freely → no dependency to upgrade.

```bash
# Add a component (copies source to your project)
npx shadcn-ui@latest add button
npx shadcn-ui@latest add dialog
npx shadcn-ui@latest add form

# Generated file: components/ui/button.tsx (you own this file)
```

---

## Generated Component Structure

```tsx
// components/ui/button.tsx (generated, fully yours to edit)
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '@/lib/utils';

const buttonVariants = cva(
  'inline-flex items-center justify-center whitespace-nowrap rounded-md text-sm font-medium ring-offset-background transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground hover:bg-destructive/90',
        outline: 'border border-input bg-background hover:bg-accent hover:text-accent-foreground',
        secondary: 'bg-secondary text-secondary-foreground hover:bg-secondary/80',
        ghost: 'hover:bg-accent hover:text-accent-foreground',
        link: 'text-primary underline-offset-4 hover:underline',
      },
      size: {
        default: 'h-10 px-4 py-2',
        sm: 'h-9 rounded-md px-3',
        lg: 'h-11 rounded-md px-8',
        icon: 'h-10 w-10',
      },
    },
    defaultVariants: { variant: 'default', size: 'default' },
  }
);

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean;
}

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, ...props }, ref) => {
    const Comp = asChild ? Slot : 'button';
    return (
      <Comp
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        {...props}
      />
    );
  }
);
```

---

## Theming with CSS Variables

shadcn/ui uses CSS custom properties for all colors — easy to theme:

```css
/* globals.css */
:root {
  --background: 0 0% 100%;
  --foreground: 222.2 84% 4.9%;
  --primary: 222.2 47.4% 11.2%;
  --primary-foreground: 210 40% 98%;
  --muted: 210 40% 96.1%;
  /* ... */
}

.dark {
  --background: 222.2 84% 4.9%;
  --foreground: 210 40% 98%;
  /* ... */
}
```

All components use `bg-primary`, `text-foreground`, etc. which map to these variables. Theming = changing the variable values.

---

## The `cn` Utility

```ts
// lib/utils.ts
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}

// Usage: cn('base-class', condition && 'conditional-class', props.className)
```

`clsx` handles conditional classes. `tailwind-merge` resolves Tailwind conflicts (e.g. `p-2 p-4` → keeps `p-4`).

---

## Setup

```bash
# New project with Next.js
npx create-next-app@latest my-app --typescript --tailwind
cd my-app
npx shadcn-ui@latest init

# Configure in components.json
```

```json
// components.json
{
  "style": "default",
  "rsc": true,
  "tailwind": { "config": "tailwind.config.ts", "css": "globals.css" },
  "aliases": { "components": "@/components", "utils": "@/lib/utils" }
}
```

---

## Common Interview Questions

**Q: Why copy source instead of using a package?**
You get full ownership. No breaking changes from upstream forcing you to update. You can modify the component to exactly fit your needs without fighting the library's opinion. The tradeoff: you must manually pull in any bug fixes or new features.

**Q: What's the `asChild` prop?**
Renders the component's logic on a different HTML element. `<Button asChild><a href="/link">Go</a></Button>` applies button styles to the `<a>` element using Radix's `Slot` component. Used to combine button behavior with link semantics.

**Q: How does shadcn/ui handle accessibility?**
It delegates to Radix UI primitives for complex interactive components (Dialog, Dropdown, Select). Radix handles ARIA attributes, keyboard navigation, focus management, and screen reader announcements.
