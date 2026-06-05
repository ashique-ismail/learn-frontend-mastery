# Spartan/ui

## The Idea

**In plain English:** Spartan/ui is a free collection of ready-made building blocks for Angular apps — things like buttons, forms, and dialogs — where instead of installing a locked-down library, you copy the actual code into your project so you can change anything you want. Angular is a JavaScript framework, which is a set of tools that helps developers build websites.

**Real-world analogy:** Imagine buying a flat-pack furniture kit (like IKEA) where every piece arrives pre-cut and pre-painted, but you get the raw boards and paint too — so you can sand them down and repaint them any color you like. Then consider that some boards handle the structure (holding weight) while other boards handle the looks (the finish and color).

- The structural boards = the `brain` layer (handles behavior, keyboard controls, and accessibility — no colors)
- The painted finish boards = the `helm` layer (adds Tailwind CSS styling on top of the brain)
- Getting the raw boards to customize freely = copy-pasting the source code into your own project

---

## What It Is

Spartan/ui is Angular's equivalent of shadcn/ui — an open-source collection of Angular component source code that you copy into your project. It uses the same "brain + helm" (headless + styled) architecture.

- **brain**: Headless, unstyled, accessible Angular directives/components (behavior only)
- **helm**: Styled layer using Tailwind CSS that wraps the brain components

---

## Architecture

```
shadcn/ui (React)          Spartan/ui (Angular)
--------------------------  --------------------------
Radix UI primitives     →   @spartan-ng/brain/* (headless directives)
CVA + Tailwind styles   →   hlm* components (Tailwind-styled wrappers)
cn() utility            →   hlm class helpers
```

---

## Setup

```bash
# With Nx
npx nx add @spartan-ng/nx-plugin

# Manual setup
npm install @spartan-ng/ui-core tailwindcss

# Add a component
npx nx g @spartan-ng/nx-plugin:ui --name=button --directory=libs/ui
# Or with Angular CLI:
npx @analogjs/create-analog spartan-component -- --name button
```

---

## Using a Component

```ts
// After adding: generates libs/ui/ui-button/
// Contains: hlm-button.directive.ts (the styled layer)

import { HlmButtonDirective } from '@spartan-ng/ui-button-helm';

@Component({
  imports: [HlmButtonDirective],
  template: `
    <button hlmBtn>Default Button</button>
    <button hlmBtn variant="destructive">Delete</button>
    <button hlmBtn variant="outline" size="sm">Small</button>
    <button hlmBtn [disabled]="isLoading()">
      @if (isLoading()) { Loading... } @else { Submit }
    </button>
  `,
})
export class MyComponent {
  isLoading = signal(false);
}
```

---

## Dialog Example

```ts
import { BrnDialogContentDirective, BrnDialogTriggerDirective } from '@spartan-ng/brain/dialog';
import {
  HlmDialogComponent,
  HlmDialogContentComponent,
  HlmDialogDescriptionDirective,
  HlmDialogFooterComponent,
  HlmDialogHeaderComponent,
  HlmDialogTitleDirective,
} from '@spartan-ng/ui-dialog-helm';

@Component({
  imports: [
    BrnDialogTriggerDirective,
    BrnDialogContentDirective,
    HlmDialogComponent,
    HlmDialogContentComponent,
    HlmDialogHeaderComponent,
    HlmDialogFooterComponent,
    HlmDialogTitleDirective,
    HlmDialogDescriptionDirective,
    HlmButtonDirective,
  ],
  template: `
    <hlm-dialog>
      <button brnDialogTrigger hlmBtn>Open Dialog</button>

      <hlm-dialog-content *brnDialogContent="let ctx">
        <hlm-dialog-header>
          <h3 hlmDialogTitle>Confirm Deletion</h3>
          <p hlmDialogDescription>This action cannot be undone.</p>
        </hlm-dialog-header>

        <hlm-dialog-footer>
          <button hlmBtn variant="outline" (click)="ctx.close()">Cancel</button>
          <button hlmBtn variant="destructive" (click)="onConfirm(); ctx.close()">
            Delete
          </button>
        </hlm-dialog-footer>
      </hlm-dialog-content>
    </hlm-dialog>
  `,
})
export class ConfirmDialog {
  onConfirm() { /* handle deletion */ }
}
```

---

## Available Components

As of 2024, Spartan/ui includes:
- Accordion, Alert, AlertDialog, AspectRatio
- Avatar, Badge, Button, Card
- Checkbox, Combobox, Command, DataTable
- DatePicker, Dialog, Form, Input
- Label, Menubar, NavigationMenu, Pagination
- Popover, Progress, RadioGroup, ScrollArea
- Select, Separator, Sheet, Skeleton
- Slider, Switch, Table, Tabs, Textarea
- Toast, Toggle, Tooltip

---

## Theming

Same CSS custom properties approach as shadcn/ui:

```css
/* globals.css */
:root {
  --background: 0 0% 100%;
  --foreground: 222.2 84% 4.9%;
  --primary: 221.2 83.2% 53.3%;
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

---

## Comparison with Angular Material

| | Spartan/ui | Angular Material |
|---|---|---|
| Styling | Tailwind (copy-paste) | Material Design (CSS) |
| Customization | Full (you own the code) | Limited by Material spec |
| Bundle size | Per-component (small) | Full Material bundle |
| Accessibility | Via brain (headless primitives) | Built-in |
| Design system | Neutral (customize freely) | Material Design |
| Maturity | Newer, growing | Stable, mature |

---

## Common Interview Questions

**Q: What's the brain/helm separation?**
`brain` packages contain the logic, ARIA, keyboard navigation, and state — no styling. `helm` packages import the brain and add Tailwind classes. You can use brain components with your own styling if you don't want the helm's Tailwind classes.

**Q: Why copy source code instead of using npm packages?**
Same philosophy as shadcn/ui: you own the code and can modify it without fighting a library's opinions. No breaking change upgrades. The tradeoff: you must manually pull upstream fixes.

**Q: Does Spartan/ui work with Angular SSR?**
Yes. The brain components are built to be SSR-compatible. Server-side rendering works the same as any other Angular standalone component.
