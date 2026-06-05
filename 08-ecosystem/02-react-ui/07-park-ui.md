# Park UI

## The Idea

**In plain English:** Park UI is a set of ready-made website building blocks (like buttons, menus, and pop-up dialogs) that you copy directly into your own project and can edit however you want. It handles all the complicated keyboard and screen-reader behaviour automatically, so you can focus on making things look the way you want.

**Real-world analogy:** Imagine buying a flat-pack furniture kit from a store. The kit comes with pre-cut, pre-finished wooden panels (the styled components), the hardware that makes drawers slide and doors open correctly (the built-in keyboard and accessibility behaviour), and an instruction sheet showing how the pieces connect (the component API). Once you bring the kit home it is yours — you can repaint it, cut it down, or add new shelves without asking the store's permission.
- The pre-cut panels = the styled component files the CLI copies into your project
- The drawer slides and door hinges = the Zag.js state machines that handle keyboard navigation and ARIA automatically
- Taking the kit home = owning the source code permanently so you can change anything

---

## What It Is

Park UI is a **styled component library** built directly on top of Ark UI primitives. It is **not** a standalone headless library — it is the styled layer in a three-tier stack:

```
Zag.js (state machines)
  └─ Ark UI (headless components — logic, ARIA, keyboard nav)
       └─ Park UI (styled components — Panda CSS recipes)
            └─ Your app
```

You get the accessibility and behavioral correctness of Ark UI's state machines, pre-styled with Panda CSS, and — crucially — you **own the component source code**. Park UI generates files into your project rather than locking them behind a package import.

---

## The Stack Explained

### Zag.js
State machine library for UI interactions. Defines every possible state transition for a component (e.g., an Accordion can be `idle`, `open`, `closing`). Framework-agnostic — the same machine runs in React, Vue, and Solid.

### Ark UI
Headless component library that wraps Zag.js machines in framework-specific components. Provides: focus management, ARIA attributes, keyboard navigation, composable part sub-components. No styles.

### Park UI
Takes Ark UI components and adds Panda CSS **recipes** (slot recipes) for styling. Exposes the same component API with pre-built visual style. You copy the component source into your project and own it.

---

## Multi-Framework Support

Park UI works across React, Solid, and Vue with the same component API. The component logic is identical — only the import path differs.

```tsx
// React
import { Button } from '~/components/ui/button';

// Solid
import { Button } from '~/components/ui/button';

// Vue
import { Button } from '~/components/ui/button';
```

The generated source adapts to your framework at install time. This is possible because Ark UI already provides uniform APIs across frameworks via Zag.js.

---

## Copy-Paste Philosophy

Like shadcn/ui, Park UI is **not** an npm package you import components from. The CLI copies component source files into your own codebase. You own the code permanently.

```
Traditional library:         npm install park-ui → import { Button } from 'park-ui'
                             (can't customise internals, dependent on upstream)

Park UI / shadcn approach:   CLI copies button.tsx → you own components/ui/button.tsx
                             (modify freely, no upstream coupling)
```

Consequence: you manually apply bug fixes or improvements from new Park UI versions — but you never fight the library to make a component match your design.

---

## Getting Started

### 1. Install Panda CSS

Park UI requires Panda CSS as its style engine.

```bash
npm install -D @pandacss/dev
npx panda init --postcss
```

### 2. Install Ark UI peer dependency

```bash
npm install @ark-ui/react   # React
npm install @ark-ui/solid   # Solid
npm install @ark-ui/vue     # Vue
```

### 3. Install the Park UI CLI

```bash
npm install -D @park-ui/cli
```

### 4. Initialise Park UI in panda.config.ts

```ts
// panda.config.ts
import { defineConfig } from '@pandacss/dev';
import { createPreset } from '@park-ui/panda-preset';

export default defineConfig({
  preflight: true,
  presets: [
    '@pandacss/preset-base',
    createPreset({
      accentColor: 'iris',  // primary brand colour
      grayColor: 'sand',    // neutral grey scale
      borderRadius: 'md',   // global border radius scale
    }),
  ],
  include: ['./src/**/*.{ts,tsx,js,jsx}'],
  outdir: 'styled-system',
});
```

### 5. Add components via CLI

```bash
# Add individual components
npx @park-ui/cli components add button
npx @park-ui/cli components add dialog
npx @park-ui/cli components add select
npx @park-ui/cli components add date-picker

# Add all available components at once
npx @park-ui/cli components add --all
```

Each command generates a file like `src/components/ui/button.tsx` that you own entirely.

---

## Project Structure After Setup

```
src/
  components/
    ui/
      button.tsx        ← generated by CLI, yours to edit
      dialog.tsx
      select.tsx
      tabs.tsx
      ...
styled-system/          ← generated by Panda CSS (do not edit)
  css/
  patterns/
  recipes/
panda.config.ts         ← Park UI preset + your custom tokens
```

---

## Generated Component Anatomy

```tsx
// src/components/ui/button.tsx (generated — you own this)
import { ark } from '@ark-ui/react/factory';
import { type VariantProps, cva } from '../../styled-system/css';
import { styled } from '../../styled-system/jsx';

// The recipe is defined here in the generated file
// OR it references a recipe from your panda.config.ts
export const buttonRecipe = cva({
  base: {
    alignItems: 'center',
    borderRadius: 'l2',
    cursor: 'pointer',
    display: 'inline-flex',
    fontWeight: 'semibold',
    justifyContent: 'center',
    transitionDuration: 'normal',
    transitionProperty: 'background, border-color, color, box-shadow',
    transitionTimingFunction: 'default',
    userSelect: 'none',
    verticalAlign: 'middle',
    whiteSpace: 'nowrap',
    _disabled: { opacity: 0.4, cursor: 'not-allowed' },
    _focusVisible: { outline: '2px solid', outlineColor: 'colorPalette.default', outlineOffset: '2px' },
  },
  variants: {
    variant: {
      solid: {
        background: 'colorPalette.default',
        color: 'colorPalette.fg',
        _hover: { background: 'colorPalette.emphasized' },
        _active: { background: 'colorPalette.emphasized' },
      },
      outline: {
        borderWidth: '1px',
        borderColor: 'colorPalette.default',
        color: 'colorPalette.default',
        _hover: { background: 'colorPalette.a2' },
      },
      ghost: {
        color: 'colorPalette.default',
        _hover: { background: 'colorPalette.a2' },
      },
      link: {
        color: 'colorPalette.default',
        _hover: { textDecoration: 'underline' },
      },
    },
    size: {
      sm: { h: '8', minW: '8', textStyle: 'sm', px: '3', gap: '1' },
      md: { h: '10', minW: '10', textStyle: 'sm', px: '4', gap: '2' },
      lg: { h: '12', minW: '12', textStyle: 'md', px: '5', gap: '2' },
      xl: { h: '14', minW: '14', textStyle: 'md', px: '6', gap: '2' },
    },
  },
  defaultVariants: { variant: 'solid', size: 'md' },
});

export type ButtonVariantProps = VariantProps<typeof buttonRecipe>;
export const Button = styled(ark.button, buttonRecipe);
```

Usage is clean and type-safe:

```tsx
import { Button } from '~/components/ui/button';

function App() {
  return (
    <div>
      <Button variant="solid" size="md">Save changes</Button>
      <Button variant="outline" size="sm">Cancel</Button>
      <Button variant="ghost" size="lg" disabled>Delete</Button>
    </div>
  );
}
```

---

## Component Catalogue

Park UI covers the components you need for production apps:

| Category | Components |
|---|---|
| Layout | Accordion, Collapsible, Splitter |
| Overlay | Dialog, Drawer, Popover, Tooltip, Hover Card |
| Forms | Checkbox, Radio Group, Select, Combobox, Switch, Slider, Pin Input, Number Input, Textarea, File Upload |
| Date/Time | Date Picker, Date Range Picker |
| Navigation | Tabs, Pagination, Breadcrumb, Menu |
| Feedback | Alert, Progress, Skeleton, Spinner, Toast |
| Display | Avatar, Badge, Card, Code, Kbd, Separator, Table, Tag |

---

## Panda CSS Integration: Theming

### Semantic Tokens

Park UI's preset registers semantic tokens that map to CSS custom properties. You extend them in `panda.config.ts`:

```ts
// panda.config.ts
import { defineConfig } from '@pandacss/dev';
import { createPreset } from '@park-ui/panda-preset';

export default defineConfig({
  presets: [
    '@pandacss/preset-base',
    createPreset({ accentColor: 'violet', grayColor: 'slate', borderRadius: 'sm' }),
  ],
  theme: {
    extend: {
      tokens: {
        fonts: {
          body: { value: 'Inter, sans-serif' },
          heading: { value: '"Cal Sans", sans-serif' },
        },
      },
      semanticTokens: {
        colors: {
          // Override the brand colour for a specific semantic role
          brand: {
            default: { value: { base: '#5B21B6', _dark: '#7C3AED' } },
            emphasized: { value: { base: '#4C1D95', _dark: '#6D28D9' } },
          },
        },
      },
    },
  },
  include: ['./src/**/*.{ts,tsx}'],
  outdir: 'styled-system',
});
```

### Color Palettes

Park UI uses Panda CSS's `colorPalette` mechanism. Components use `colorPalette.default`, `colorPalette.fg`, etc. Setting `accentColor` wires these tokens automatically.

Available accent colours: `amber`, `blue`, `bronze`, `brown`, `crimson`, `cyan`, `gold`, `grass`, `green`, `indigo`, `iris`, `jade`, `lime`, `mauve`, `mint`, `olive`, `orange`, `pink`, `plum`, `purple`, `red`, `ruby`, `sage`, `sand`, `sky`, `slate`, `teal`, `tomato`, `violet`, `yellow`.

---

## Customising Component Recipes

Because you own the component source, you can modify the recipe directly in the generated file. But for design-system-level overrides, the idiomatic approach is to add or extend slot recipes in `panda.config.ts`.

### Example: Extending the Button Recipe

```ts
// panda.config.ts — add a new variant without touching the generated file
import { defineConfig } from '@pandacss/dev';
import { createPreset } from '@park-ui/panda-preset';

export default defineConfig({
  presets: ['@pandacss/preset-base', createPreset({ accentColor: 'iris' })],
  theme: {
    extend: {
      recipes: {
        button: {
          variants: {
            variant: {
              // New variant — add to existing variants
              danger: {
                background: 'red.9',
                color: 'white',
                _hover: { background: 'red.10' },
              },
            },
          },
        },
      },
    },
  },
  include: ['./src/**/*.{ts,tsx}'],
  outdir: 'styled-system',
});
```

### Example: Modifying the Dialog Slot Recipe

Dialog is a multi-part component (backdrop, positioner, content, title, description, close trigger). Panda calls these **slot recipes**.

```ts
// panda.config.ts
theme: {
  extend: {
    slotRecipes: {
      dialog: {
        slots: ['backdrop', 'positioner', 'content', 'title', 'description', 'closeTrigger'],
        base: {
          content: {
            // Override content base styles
            borderRadius: 'xl',
            boxShadow: '2xl',
            padding: '8',
          },
        },
      },
    },
  },
},
```

---

## Accessibility: What You Get for Free

Park UI components inherit full accessibility from Ark UI, which is powered by Zag.js state machines. This means the component logic (and its ARIA behaviour) is deterministic.

### What Ark UI (and therefore Park UI) handles automatically:

| Component | Automatic Behaviour |
|---|---|
| Dialog | Focus trap, Escape to close, scroll lock, `role="dialog"`, `aria-modal`, `aria-labelledby` |
| Select | `role="listbox"`, `aria-selected`, `aria-expanded`, arrow key navigation, type-ahead search |
| Tabs | `role="tablist"`, `role="tab"`, `role="tabpanel"`, arrow key navigation, `aria-selected` |
| Accordion | `aria-expanded`, `aria-controls`, `aria-labelledby`, Enter/Space to toggle |
| Combobox | Full WAI-ARIA combobox pattern: input, listbox, options, live region |
| Date Picker | Complex keyboard grid navigation, ARIA grid roles, screen reader announcements |

### Keyboard navigation patterns (Zag.js state machines enforce these):

```
Dialog:
  Tab / Shift+Tab  → cycle focus within trap
  Escape           → close

Select / Combobox:
  Arrow Down/Up    → navigate options
  Enter / Space    → select highlighted option
  Escape           → close listbox
  Home / End       → first / last option
  Type chars       → type-ahead filter

Tabs:
  Arrow Left/Right → move between tabs
  Home / End       → first / last tab
  Enter / Space    → activate focused tab

Accordion:
  Enter / Space    → expand / collapse item
  Arrow Down/Up    → move between headers
```

You do not wire these up. They work because the underlying Zag.js state machine defines all valid state transitions.

---

## Example: Dialog

```tsx
// src/components/ui/dialog.tsx (generated by CLI — simplified for illustration)
import { Dialog as ArkDialog } from '@ark-ui/react/dialog';
import { styled } from '../../styled-system/jsx';
import { dialog } from '../../styled-system/recipes'; // slot recipe

const { withProvider, withContext } = createStyleContext(dialog);

export const DialogRoot = withProvider(ArkDialog.Root);
export const DialogBackdrop = withContext(styled(ArkDialog.Backdrop), 'backdrop');
export const DialogContent = withContext(styled(ArkDialog.Content), 'content');
export const DialogTitle = withContext(styled(ArkDialog.Title), 'title');
export const DialogDescription = withContext(styled(ArkDialog.Description), 'description');
export const DialogCloseTrigger = withContext(styled(ArkDialog.CloseTrigger), 'closeTrigger');
export const DialogTrigger = ArkDialog.Trigger;
export const DialogPositioner = styled(ArkDialog.Positioner);
```

Usage in your app:

```tsx
import {
  DialogRoot,
  DialogBackdrop,
  DialogContent,
  DialogTitle,
  DialogDescription,
  DialogCloseTrigger,
  DialogTrigger,
  DialogPositioner,
} from '~/components/ui/dialog';
import { Button } from '~/components/ui/button';
import { XIcon } from 'lucide-react';

function DeleteConfirmDialog({ onConfirm }: { onConfirm: () => void }) {
  return (
    <DialogRoot>
      <DialogTrigger asChild>
        <Button variant="outline" colorPalette="red">Delete account</Button>
      </DialogTrigger>

      <DialogBackdrop />   {/* styled backdrop, ARIA handled automatically */}

      <DialogPositioner>
        <DialogContent>
          <DialogTitle>Are you sure?</DialogTitle>
          <DialogDescription>
            This action is permanent and cannot be undone.
          </DialogDescription>

          <div style={{ display: 'flex', gap: '12px', marginTop: '24px', justifyContent: 'flex-end' }}>
            <DialogCloseTrigger asChild>
              <Button variant="outline">Cancel</Button>
            </DialogCloseTrigger>
            <DialogCloseTrigger asChild>
              <Button colorPalette="red" onClick={onConfirm}>Delete</Button>
            </DialogCloseTrigger>
          </div>

          <DialogCloseTrigger asChild position="absolute" top="4" right="4">
            <Button variant="ghost" size="sm" aria-label="Close">
              <XIcon />
            </Button>
          </DialogCloseTrigger>
        </DialogContent>
      </DialogPositioner>
    </DialogRoot>
  );
}
```

---

## Example: Select

```tsx
import {
  SelectRoot,
  SelectControl,
  SelectTrigger,
  SelectValueText,
  SelectIndicator,
  SelectPositioner,
  SelectContent,
  SelectItemGroup,
  SelectItem,
  SelectItemText,
  SelectItemIndicator,
  SelectLabel,
} from '~/components/ui/select';
import { createListCollection } from '@ark-ui/react/select';
import { CheckIcon, ChevronsUpDownIcon } from 'lucide-react';

const frameworks = createListCollection({
  items: [
    { label: 'React', value: 'react' },
    { label: 'Angular', value: 'angular' },
    { label: 'Vue', value: 'vue' },
    { label: 'Svelte', value: 'svelte' },
  ],
});

function FrameworkPicker() {
  return (
    <SelectRoot collection={frameworks} width="200px">
      <SelectLabel>Framework</SelectLabel>
      <SelectControl>
        <SelectTrigger>
          <SelectValueText placeholder="Select framework" />
          <SelectIndicator>
            <ChevronsUpDownIcon />
          </SelectIndicator>
        </SelectTrigger>
      </SelectControl>

      <SelectPositioner>
        <SelectContent>
          <SelectItemGroup>
            {frameworks.items.map((item) => (
              <SelectItem key={item.value} item={item}>
                <SelectItemText>{item.label}</SelectItemText>
                <SelectItemIndicator>
                  <CheckIcon />
                </SelectItemIndicator>
              </SelectItem>
            ))}
          </SelectItemGroup>
        </SelectContent>
      </SelectPositioner>
    </SelectRoot>
  );
}
```

---

## Example: Tabs

```tsx
import {
  TabsRoot,
  TabsList,
  TabsTrigger,
  TabsContent,
  TabsIndicator,
} from '~/components/ui/tabs';

function AccountTabs() {
  return (
    <TabsRoot defaultValue="profile">
      <TabsList>
        <TabsTrigger value="profile">Profile</TabsTrigger>
        <TabsTrigger value="billing">Billing</TabsTrigger>
        <TabsTrigger value="security">Security</TabsTrigger>
        <TabsIndicator />  {/* animated underline indicator */}
      </TabsList>

      <TabsContent value="profile">
        <p>Edit your profile details here.</p>
      </TabsContent>
      <TabsContent value="billing">
        <p>Manage payment methods and invoices.</p>
      </TabsContent>
      <TabsContent value="security">
        <p>Change password and two-factor authentication.</p>
      </TabsContent>
    </TabsRoot>
  );
}
```

---

## Example: Toast

Park UI uses Ark UI's toast API with the `@zag-js/toast` machine.

```tsx
// Step 1: Set up the Toaster — usually in a layout component
import { Toaster, createToaster } from '~/components/ui/toaster';

export const toaster = createToaster({
  placement: 'top-end',
  max: 3,
});

function RootLayout({ children }) {
  return (
    <>
      {children}
      <Toaster toaster={toaster} />
    </>
  );
}

// Step 2: Trigger toasts anywhere
import { toaster } from '~/lib/toaster';

function SaveButton() {
  return (
    <Button
      onClick={() => {
        toaster.success({
          title: 'Saved',
          description: 'Your changes have been saved.',
        });
      }}
    >
      Save
    </Button>
  );
}
```

---

## Comparison with shadcn/ui

Both share the same **copy-paste philosophy** — you own the component source. The differences are in the primitive layer and style engine.

| | Park UI | shadcn/ui |
|---|---|---|
| Primitive layer | Ark UI (Zag.js state machines) | Radix UI |
| Style engine | Panda CSS (recipes, tokens) | Tailwind CSS (utility classes) |
| Framework support | React, Solid, Vue | React only |
| Theming | Semantic tokens in panda.config.ts | CSS variables in globals.css |
| Date/Time pickers | Yes (from Ark UI) | No (Radix lacks these) |
| Community size | Smaller, newer | Much larger, dominant in React ecosystem |
| Styling approach | Type-safe style objects, recipes | String-based class names (CVA + cn()) |
| Build output | Atomic CSS (Panda) | Tailwind utility classes |
| TypeScript DX | Excellent (style props are typed) | Good (CVA provides types) |

**Same idea, different stack.** Choose Park UI when you are already on Panda CSS or want multi-framework support. Choose shadcn/ui when you are on Tailwind CSS and React-only.

---

## Comparison with Raw Radix UI + Tailwind

Using Radix UI directly with Tailwind is what shadcn/ui wraps into a copy-paste library. Park UI does the same for Ark UI + Panda CSS.

| | Raw Ark UI + Panda CSS | Park UI |
|---|---|---|
| What you write | Full component markup + styles from scratch | Copy component, modify recipe |
| Time investment | High (build every component yourself) | Low (components pre-built, just tweak) |
| Control | Maximum | High (you own the source) |
| Consistency | Manual (your discipline) | Built-in (shared recipes) |
| Component parts | All Ark parts exposed | Pre-assembled with styled parts |

Park UI sits between "roll your own from headless primitives" and "full styled library you can't touch". It is the styled scaffolding on top of Ark, which is itself the behavioral scaffolding on top of Zag.

---

## When to Prefer Park UI

**Reach for Park UI when:**

- Your project already uses **Panda CSS** — Park UI's preset integrates cleanly, no extra tooling.
- You want **Ark UI primitives** specifically — multi-framework support, Zag.js state machines, complex components like DatePicker or TimePicker that Radix UI does not offer.
- Your team works across **React and Vue** (or Solid) and needs a consistent component API and visual language.
- You want the **copy-paste ownership model** but do not want Tailwind in your stack.
- You prefer **type-safe style props** over string-based class names.
- You need complex interactions (file upload, date range picker, OTP pin input) that a simpler headless library does not cover.

**Reach for shadcn/ui instead when:**

- You are on **Tailwind CSS** and React-only.
- You want the **largest community** and most third-party resources, tutorials, and ready-made blocks.
- Your team is already fluent in the `cn()` + CVA pattern.

---

## Trade-Offs

### Advantages

- Accessibility baked in via Zag.js state machines — the most rigorous behavioral model available.
- Multi-framework: same components in React, Vue, and Solid.
- Panda CSS provides zero-runtime CSS with build-time extraction — no style injection at runtime.
- Type-safe styling: style props are TypeScript objects, not strings.
- Complex components (DatePicker, DateRangePicker, Combobox, FileUpload) that many headless libs lack.

### Disadvantages

- **Smaller community**: fewer tutorials, blog posts, StackOverflow answers, and third-party component blocks compared to shadcn/ui.
- **Panda CSS learning curve**: semantic tokens, slot recipes, and `styled-system/` generated folder require understanding before you can customise confidently.
- **Build step required**: Panda CSS requires a build-time transform (PostCSS plugin). Cannot drop into a pure CDN or no-build setup.
- **Less ecosystem content**: shadcn/ui has a growing library of full-page templates and blocks; Park UI does not yet.
- **Newer project**: shadcn/ui has more production battle-testing and community feedback.

---

## Common Interview Questions

**Q: What is Park UI and how does it differ from shadcn/ui?**
Both are copy-paste component libraries — you run a CLI to add component source files to your project rather than importing from an npm package. The difference is the primitive and style layer: Park UI uses Ark UI (Zag.js state machines) and Panda CSS, while shadcn/ui uses Radix UI and Tailwind CSS. Park UI also supports React, Vue, and Solid; shadcn/ui is React-only.

**Q: Why would you use Park UI over just using Ark UI directly?**
Ark UI is headless — it provides logic, ARIA, and keyboard behaviour but zero styles. You would need to write all CSS yourself for every component. Park UI provides pre-built Panda CSS recipes for those components, so you start with a working, visually coherent UI and customise from there instead of starting from scratch.

**Q: How does theming work in Park UI?**
Via the `createPreset()` call in `panda.config.ts`. You select an `accentColor` (e.g. `violet`) and `grayColor` (e.g. `slate`) and Park UI's Panda preset wires up semantic tokens that map to CSS custom properties. All components reference these tokens via `colorPalette.*`, so changing the preset changes the entire UI colour scheme. For deeper customisation you extend `theme.extend.semanticTokens` or override slot recipes.

**Q: What is a slot recipe in Panda CSS?**
A slot recipe is a multi-part component recipe where each named slot gets its own style definitions. For example a `dialog` slot recipe has slots: `backdrop`, `positioner`, `content`, `title`, `description`, `closeTrigger`. Each slot can have `base` styles and `variant` styles. This replaces the pattern of exporting separate styled sub-components — everything for a component is collocated in one recipe definition.

**Q: How does accessibility work in Park UI — do you need to add ARIA manually?**
No. ARIA attributes, keyboard navigation, focus management, and screen reader behaviour are injected by Ark UI and Zag.js state machines. The state machine defines all valid state transitions for a component; Ark translates those states into the correct ARIA attributes at render time. You get correct accessibility for free as long as you use the component parts (DialogTitle, DialogDescription, etc.) in the intended structure.

**Q: When would you not use Park UI?**
When your project uses Tailwind CSS (use shadcn/ui instead), when you need maximum community resources and third-party templates, or when adding Panda CSS's build pipeline is not acceptable for the project constraints. Also avoid it if you need server-side rendering without a build step — Panda CSS requires PostCSS processing.

**Q: What does "you own the component source" actually mean in practice?**
The CLI writes the full component TypeScript source into your `src/components/ui/` directory. That file is a regular file in your git repo. You can change anything: rename parts, add props, alter the recipe, swap out the underlying Ark primitive. You are not constrained by what the library exports. The downside is you do not automatically receive upstream improvements — you need to manually re-run the CLI or diff against new versions to pull in changes.
