# Radix UI

## The Idea

**In plain English:** Radix UI is a collection of ready-made building blocks (called "primitives") for websites that handle all the tricky interactive behavior — like opening a dropdown menu or a pop-up dialog — without imposing any visual style, so you can make them look however you want.

**Real-world analogy:** Imagine buying flat-pack furniture from a store that sells just the mechanism — the hinges, drawer slides, and locking joints — but no wood panels or paint. You assemble the working parts yourself and then choose your own materials and finish to match your home's style.
- The hinges and drawer slides = Radix primitives (the functional behavior and accessibility logic)
- The wood panels and paint = your CSS or Tailwind styles (the visual appearance)
- Your finished furniture piece = the complete, styled component your users see and interact with

---

## Introduction

Radix UI is an open-source component library optimized for fast development, easy maintenance, and accessibility. Unlike styled component libraries, Radix provides unstyled, accessible primitives that serve as the foundation for your design system. It focuses on functionality, accessibility, and developer experience while leaving styling entirely up to you.

## Installation and Setup

### Basic Installation

```bash
# Install individual primitives as needed
npm install @radix-ui/react-dialog
npm install @radix-ui/react-dropdown-menu
npm install @radix-ui/react-select
npm install @radix-ui/react-tooltip

# or with yarn
yarn add @radix-ui/react-dialog @radix-ui/react-dropdown-menu

# or with pnpm
pnpm add @radix-ui/react-dialog
```

### Common Primitives Package

```bash
# Install commonly used primitives
npm install @radix-ui/react-dialog \
  @radix-ui/react-dropdown-menu \
  @radix-ui/react-popover \
  @radix-ui/react-tooltip \
  @radix-ui/react-select \
  @radix-ui/react-tabs \
  @radix-ui/react-accordion \
  @radix-ui/react-alert-dialog \
  @radix-ui/react-checkbox \
  @radix-ui/react-radio-group \
  @radix-ui/react-switch
```

### Setup with Styling (Tailwind CSS Example)

```bash
npm install tailwindcss @tailwindcss/forms
npx tailwindcss init
```

```tsx
// tailwind.config.js
module.exports = {
  content: [
    './pages/**/*.{js,ts,jsx,tsx,mdx}',
    './components/**/*.{js,ts,jsx,tsx,mdx}',
    './app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {},
  },
  plugins: [require('@tailwindcss/forms')],
};
```

## Core Primitives

### Dialog (Modal)

```tsx
import * as Dialog from '@radix-ui/react-dialog';
import { Cross2Icon } from '@radix-ui/react-icons';
import './dialog.css';

function DialogDemo() {
  return (
    <Dialog.Root>
      <Dialog.Trigger asChild>
        <button className="button">Open Dialog</button>
      </Dialog.Trigger>
      <Dialog.Portal>
        <Dialog.Overlay className="dialog-overlay" />
        <Dialog.Content className="dialog-content">
          <Dialog.Title className="dialog-title">
            Edit Profile
          </Dialog.Title>
          <Dialog.Description className="dialog-description">
            Make changes to your profile here. Click save when you're done.
          </Dialog.Description>
          
          <fieldset className="fieldset">
            <label className="label" htmlFor="name">
              Name
            </label>
            <input
              className="input"
              id="name"
              defaultValue="John Doe"
            />
          </fieldset>
          
          <fieldset className="fieldset">
            <label className="label" htmlFor="username">
              Username
            </label>
            <input
              className="input"
              id="username"
              defaultValue="@johndoe"
            />
          </fieldset>

          <div className="dialog-actions">
            <Dialog.Close asChild>
              <button className="button button-green">
                Save changes
              </button>
            </Dialog.Close>
          </div>
          
          <Dialog.Close asChild>
            <button className="icon-button" aria-label="Close">
              <Cross2Icon />
            </button>
          </Dialog.Close>
        </Dialog.Content>
      </Dialog.Portal>
    </Dialog.Root>
  );
}

export default DialogDemo;
```

```css
/* dialog.css */
.dialog-overlay {
  background-color: rgba(0, 0, 0, 0.5);
  position: fixed;
  inset: 0;
  animation: overlayShow 150ms cubic-bezier(0.16, 1, 0.3, 1);
}

.dialog-content {
  background-color: white;
  border-radius: 6px;
  box-shadow: hsl(206 22% 7% / 35%) 0px 10px 38px -10px,
    hsl(206 22% 7% / 20%) 0px 10px 20px -15px;
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 90vw;
  max-width: 450px;
  max-height: 85vh;
  padding: 25px;
  animation: contentShow 150ms cubic-bezier(0.16, 1, 0.3, 1);
}

.dialog-content:focus {
  outline: none;
}

.dialog-title {
  margin: 0;
  font-weight: 500;
  font-size: 17px;
}

.dialog-description {
  margin: 10px 0 20px;
  color: #666;
  font-size: 15px;
  line-height: 1.5;
}

@keyframes overlayShow {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

@keyframes contentShow {
  from {
    opacity: 0;
    transform: translate(-50%, -48%) scale(0.96);
  }
  to {
    opacity: 1;
    transform: translate(-50%, -50%) scale(1);
  }
}
```

### Dropdown Menu

```tsx
import * as DropdownMenu from '@radix-ui/react-dropdown-menu';
import {
  HamburgerMenuIcon,
  DotFilledIcon,
  CheckIcon,
  ChevronRightIcon,
} from '@radix-ui/react-icons';

function DropdownMenuDemo() {
  return (
    <DropdownMenu.Root>
      <DropdownMenu.Trigger asChild>
        <button className="icon-button" aria-label="Menu">
          <HamburgerMenuIcon />
        </button>
      </DropdownMenu.Trigger>

      <DropdownMenu.Portal>
        <DropdownMenu.Content className="dropdown-content" sideOffset={5}>
          <DropdownMenu.Item className="dropdown-item">
            New Tab <div className="right-slot">⌘+T</div>
          </DropdownMenu.Item>
          <DropdownMenu.Item className="dropdown-item">
            New Window <div className="right-slot">⌘+N</div>
          </DropdownMenu.Item>
          <DropdownMenu.Item className="dropdown-item" disabled>
            New Private Window <div className="right-slot">⇧+⌘+N</div>
          </DropdownMenu.Item>
          
          <DropdownMenu.Sub>
            <DropdownMenu.SubTrigger className="dropdown-item">
              More Tools
              <div className="right-slot">
                <ChevronRightIcon />
              </div>
            </DropdownMenu.SubTrigger>
            <DropdownMenu.Portal>
              <DropdownMenu.SubContent className="dropdown-content" sideOffset={2}>
                <DropdownMenu.Item className="dropdown-item">
                  Save Page As… <div className="right-slot">⌘+S</div>
                </DropdownMenu.Item>
                <DropdownMenu.Item className="dropdown-item">
                  Create Shortcut…
                </DropdownMenu.Item>
                <DropdownMenu.Item className="dropdown-item">
                  Name Window…
                </DropdownMenu.Item>
                <DropdownMenu.Separator className="dropdown-separator" />
                <DropdownMenu.Item className="dropdown-item">
                  Developer Tools
                </DropdownMenu.Item>
              </DropdownMenu.SubContent>
            </DropdownMenu.Portal>
          </DropdownMenu.Sub>

          <DropdownMenu.Separator className="dropdown-separator" />

          <DropdownMenu.CheckboxItem
            className="dropdown-checkbox-item"
            checked={true}
          >
            <DropdownMenu.ItemIndicator className="dropdown-item-indicator">
              <CheckIcon />
            </DropdownMenu.ItemIndicator>
            Show Bookmarks <div className="right-slot">⌘+B</div>
          </DropdownMenu.CheckboxItem>
          <DropdownMenu.CheckboxItem className="dropdown-checkbox-item">
            <DropdownMenu.ItemIndicator className="dropdown-item-indicator">
              <CheckIcon />
            </DropdownMenu.ItemIndicator>
            Show Full URLs
          </DropdownMenu.CheckboxItem>

          <DropdownMenu.Separator className="dropdown-separator" />

          <DropdownMenu.Label className="dropdown-label">
            People
          </DropdownMenu.Label>
          <DropdownMenu.RadioGroup value="pedro">
            <DropdownMenu.RadioItem className="dropdown-radio-item" value="pedro">
              <DropdownMenu.ItemIndicator className="dropdown-item-indicator">
                <DotFilledIcon />
              </DropdownMenu.ItemIndicator>
              Pedro Duarte
            </DropdownMenu.RadioItem>
            <DropdownMenu.RadioItem className="dropdown-radio-item" value="colm">
              <DropdownMenu.ItemIndicator className="dropdown-item-indicator">
                <DotFilledIcon />
              </DropdownMenu.ItemIndicator>
              Colm Tuite
            </DropdownMenu.RadioItem>
          </DropdownMenu.RadioGroup>

          <DropdownMenu.Arrow className="dropdown-arrow" />
        </DropdownMenu.Content>
      </DropdownMenu.Portal>
    </DropdownMenu.Root>
  );
}

export default DropdownMenuDemo;
```

### Popover

```tsx
import * as Popover from '@radix-ui/react-popover';
import { MixerHorizontalIcon, Cross2Icon } from '@radix-ui/react-icons';

function PopoverDemo() {
  return (
    <Popover.Root>
      <Popover.Trigger asChild>
        <button className="icon-button" aria-label="Update dimensions">
          <MixerHorizontalIcon />
        </button>
      </Popover.Trigger>
      <Popover.Portal>
        <Popover.Content className="popover-content" sideOffset={5}>
          <div style={{ display: 'flex', flexDirection: 'column', gap: 10 }}>
            <p className="text" style={{ marginBottom: 10 }}>
              Dimensions
            </p>
            <fieldset className="fieldset">
              <label className="label" htmlFor="width">
                Width
              </label>
              <input className="input" id="width" defaultValue="100%" />
            </fieldset>
            <fieldset className="fieldset">
              <label className="label" htmlFor="maxWidth">
                Max. width
              </label>
              <input className="input" id="maxWidth" defaultValue="300px" />
            </fieldset>
            <fieldset className="fieldset">
              <label className="label" htmlFor="height">
                Height
              </label>
              <input className="input" id="height" defaultValue="25px" />
            </fieldset>
            <fieldset className="fieldset">
              <label className="label" htmlFor="maxHeight">
                Max. height
              </label>
              <input className="input" id="maxHeight" defaultValue="none" />
            </fieldset>
          </div>
          <Popover.Close className="popover-close" aria-label="Close">
            <Cross2Icon />
          </Popover.Close>
          <Popover.Arrow className="popover-arrow" />
        </Popover.Content>
      </Popover.Portal>
    </Popover.Root>
  );
}

export default PopoverDemo;
```

### Tooltip

```tsx
import * as Tooltip from '@radix-ui/react-tooltip';
import { PlusIcon } from '@radix-ui/react-icons';

function TooltipDemo() {
  return (
    <Tooltip.Provider>
      <Tooltip.Root>
        <Tooltip.Trigger asChild>
          <button className="icon-button">
            <PlusIcon />
          </button>
        </Tooltip.Trigger>
        <Tooltip.Portal>
          <Tooltip.Content className="tooltip-content" sideOffset={5}>
            Add to library
            <Tooltip.Arrow className="tooltip-arrow" />
          </Tooltip.Content>
        </Tooltip.Portal>
      </Tooltip.Root>
    </Tooltip.Provider>
  );
}

export default TooltipDemo;
```

### Select

```tsx
import * as Select from '@radix-ui/react-select';
import {
  CheckIcon,
  ChevronDownIcon,
  ChevronUpIcon,
} from '@radix-ui/react-icons';

const SelectItem = React.forwardRef<
  HTMLDivElement,
  Select.SelectItemProps
>(({ children, ...props }, forwardedRef) => {
  return (
    <Select.Item className="select-item" {...props} ref={forwardedRef}>
      <Select.ItemText>{children}</Select.ItemText>
      <Select.ItemIndicator className="select-item-indicator">
        <CheckIcon />
      </Select.ItemIndicator>
    </Select.Item>
  );
});

function SelectDemo() {
  return (
    <Select.Root>
      <Select.Trigger className="select-trigger" aria-label="Food">
        <Select.Value placeholder="Select a fruit…" />
        <Select.Icon className="select-icon">
          <ChevronDownIcon />
        </Select.Icon>
      </Select.Trigger>
      <Select.Portal>
        <Select.Content className="select-content">
          <Select.ScrollUpButton className="select-scroll-button">
            <ChevronUpIcon />
          </Select.ScrollUpButton>
          <Select.Viewport className="select-viewport">
            <Select.Group>
              <Select.Label className="select-label">Fruits</Select.Label>
              <SelectItem value="apple">Apple</SelectItem>
              <SelectItem value="banana">Banana</SelectItem>
              <SelectItem value="blueberry">Blueberry</SelectItem>
              <SelectItem value="grapes">Grapes</SelectItem>
              <SelectItem value="pineapple">Pineapple</SelectItem>
            </Select.Group>

            <Select.Separator className="select-separator" />

            <Select.Group>
              <Select.Label className="select-label">Vegetables</Select.Label>
              <SelectItem value="aubergine">Aubergine</SelectItem>
              <SelectItem value="broccoli">Broccoli</SelectItem>
              <SelectItem value="carrot" disabled>
                Carrot
              </SelectItem>
              <SelectItem value="courgette">Courgette</SelectItem>
              <SelectItem value="leek">Leek</SelectItem>
            </Select.Group>
          </Select.Viewport>
          <Select.ScrollDownButton className="select-scroll-button">
            <ChevronDownIcon />
          </Select.ScrollDownButton>
        </Select.Content>
      </Select.Portal>
    </Select.Root>
  );
}

export default SelectDemo;
```

### Tabs

```tsx
import * as Tabs from '@radix-ui/react-tabs';

function TabsDemo() {
  return (
    <Tabs.Root className="tabs-root" defaultValue="tab1">
      <Tabs.List className="tabs-list" aria-label="Manage your account">
        <Tabs.Trigger className="tabs-trigger" value="tab1">
          Account
        </Tabs.Trigger>
        <Tabs.Trigger className="tabs-trigger" value="tab2">
          Password
        </Tabs.Trigger>
      </Tabs.List>
      <Tabs.Content className="tabs-content" value="tab1">
        <p className="text">
          Make changes to your account here. Click save when you're done.
        </p>
        <fieldset className="fieldset">
          <label className="label" htmlFor="name">
            Name
          </label>
          <input className="input" id="name" defaultValue="Pedro Duarte" />
        </fieldset>
        <fieldset className="fieldset">
          <label className="label" htmlFor="username">
            Username
          </label>
          <input className="input" id="username" defaultValue="@peduarte" />
        </fieldset>
        <div style={{ display: 'flex', marginTop: 20, justifyContent: 'flex-end' }}>
          <button className="button button-green">Save changes</button>
        </div>
      </Tabs.Content>
      <Tabs.Content className="tabs-content" value="tab2">
        <p className="text">
          Change your password here. After saving, you'll be logged out.
        </p>
        <fieldset className="fieldset">
          <label className="label" htmlFor="currentPassword">
            Current password
          </label>
          <input className="input" id="currentPassword" type="password" />
        </fieldset>
        <fieldset className="fieldset">
          <label className="label" htmlFor="newPassword">
            New password
          </label>
          <input className="input" id="newPassword" type="password" />
        </fieldset>
        <fieldset className="fieldset">
          <label className="label" htmlFor="confirmPassword">
            Confirm password
          </label>
          <input className="input" id="confirmPassword" type="password" />
        </fieldset>
        <div style={{ display: 'flex', marginTop: 20, justifyContent: 'flex-end' }}>
          <button className="button button-green">Change password</button>
        </div>
      </Tabs.Content>
    </Tabs.Root>
  );
}

export default TabsDemo;
```

### Accordion

```tsx
import * as Accordion from '@radix-ui/react-accordion';
import { ChevronDownIcon } from '@radix-ui/react-icons';
import React from 'react';

const AccordionItem = React.forwardRef<
  HTMLDivElement,
  Accordion.AccordionItemProps
>(({ children, ...props }, forwardedRef) => (
  <Accordion.Item
    className="accordion-item"
    {...props}
    ref={forwardedRef}
  >
    {children}
  </Accordion.Item>
));

const AccordionTrigger = React.forwardRef<
  HTMLButtonElement,
  Accordion.AccordionTriggerProps
>(({ children, ...props }, forwardedRef) => (
  <Accordion.Header className="accordion-header">
    <Accordion.Trigger
      className="accordion-trigger"
      {...props}
      ref={forwardedRef}
    >
      {children}
      <ChevronDownIcon className="accordion-chevron" aria-hidden />
    </Accordion.Trigger>
  </Accordion.Header>
));

const AccordionContent = React.forwardRef<
  HTMLDivElement,
  Accordion.AccordionContentProps
>(({ children, ...props }, forwardedRef) => (
  <Accordion.Content
    className="accordion-content"
    {...props}
    ref={forwardedRef}
  >
    <div className="accordion-content-text">{children}</div>
  </Accordion.Content>
));

function AccordionDemo() {
  return (
    <Accordion.Root
      className="accordion-root"
      type="single"
      defaultValue="item-1"
      collapsible
    >
      <AccordionItem value="item-1">
        <AccordionTrigger>Is it accessible?</AccordionTrigger>
        <AccordionContent>
          Yes. It adheres to the WAI-ARIA design pattern.
        </AccordionContent>
      </AccordionItem>

      <AccordionItem value="item-2">
        <AccordionTrigger>Is it unstyled?</AccordionTrigger>
        <AccordionContent>
          Yes. It's unstyled by default, giving you freedom over the look and
          feel.
        </AccordionContent>
      </AccordionItem>

      <AccordionItem value="item-3">
        <AccordionTrigger>Can it be animated?</AccordionTrigger>
        <AccordionContent>
          Yes! You can animate the Accordion with CSS or JavaScript animation
          libraries.
        </AccordionContent>
      </AccordionItem>
    </Accordion.Root>
  );
}

export default AccordionDemo;
```

### Form Controls

```tsx
import * as Checkbox from '@radix-ui/react-checkbox';
import * as RadioGroup from '@radix-ui/react-radio-group';
import * as Switch from '@radix-ui/react-switch';
import { CheckIcon } from '@radix-ui/react-icons';

function FormControls() {
  return (
    <div style={{ display: 'flex', flexDirection: 'column', gap: 20 }}>
      {/* Checkbox */}
      <div style={{ display: 'flex', alignItems: 'center' }}>
        <Checkbox.Root className="checkbox-root" defaultChecked id="c1">
          <Checkbox.Indicator className="checkbox-indicator">
            <CheckIcon />
          </Checkbox.Indicator>
        </Checkbox.Root>
        <label className="label" htmlFor="c1" style={{ paddingLeft: 15 }}>
          Accept terms and conditions.
        </label>
      </div>

      {/* Radio Group */}
      <form>
        <RadioGroup.Root
          className="radio-group-root"
          defaultValue="default"
          aria-label="View density"
        >
          <div style={{ display: 'flex', alignItems: 'center' }}>
            <RadioGroup.Item className="radio-group-item" value="default" id="r1">
              <RadioGroup.Indicator className="radio-group-indicator" />
            </RadioGroup.Item>
            <label className="label" htmlFor="r1" style={{ paddingLeft: 15 }}>
              Default
            </label>
          </div>
          <div style={{ display: 'flex', alignItems: 'center' }}>
            <RadioGroup.Item className="radio-group-item" value="comfortable" id="r2">
              <RadioGroup.Indicator className="radio-group-indicator" />
            </RadioGroup.Item>
            <label className="label" htmlFor="r2" style={{ paddingLeft: 15 }}>
              Comfortable
            </label>
          </div>
          <div style={{ display: 'flex', alignItems: 'center' }}>
            <RadioGroup.Item className="radio-group-item" value="compact" id="r3">
              <RadioGroup.Indicator className="radio-group-indicator" />
            </RadioGroup.Item>
            <label className="label" htmlFor="r3" style={{ paddingLeft: 15 }}>
              Compact
            </label>
          </div>
        </RadioGroup.Root>
      </form>

      {/* Switch */}
      <div style={{ display: 'flex', alignItems: 'center' }}>
        <label className="label" htmlFor="airplane-mode" style={{ paddingRight: 15 }}>
          Airplane mode
        </label>
        <Switch.Root className="switch-root" id="airplane-mode">
          <Switch.Thumb className="switch-thumb" />
        </Switch.Root>
      </div>
    </div>
  );
}

export default FormControls;
```

## Advanced Patterns

### Composed Components

```tsx
import * as Dialog from '@radix-ui/react-dialog';
import * as Form from '@radix-ui/react-form';

function ComposedDialog() {
  return (
    <Dialog.Root>
      <Dialog.Trigger asChild>
        <button>Create Account</button>
      </Dialog.Trigger>
      <Dialog.Portal>
        <Dialog.Overlay className="dialog-overlay" />
        <Dialog.Content className="dialog-content">
          <Dialog.Title>Create Account</Dialog.Title>
          <Form.Root>
            <Form.Field name="email">
              <Form.Label>Email</Form.Label>
              <Form.Control type="email" required />
              <Form.Message match="valueMissing">
                Please enter your email
              </Form.Message>
              <Form.Message match="typeMismatch">
                Please provide a valid email
              </Form.Message>
            </Form.Field>
            
            <Form.Field name="password">
              <Form.Label>Password</Form.Label>
              <Form.Control type="password" required />
              <Form.Message match="valueMissing">
                Please enter a password
              </Form.Message>
            </Form.Field>
            
            <Form.Submit asChild>
              <button>Create Account</button>
            </Form.Submit>
          </Form.Root>
        </Dialog.Content>
      </Dialog.Portal>
    </Dialog.Root>
  );
}
```

### Custom Styling with CSS Modules

```tsx
// Button.module.css
.button {
  all: unset;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: 4px;
  padding: 0 15px;
  font-size: 15px;
  line-height: 1;
  font-weight: 500;
  height: 35px;
  background-color: white;
  color: var(--violet11);
  box-shadow: 0 2px 10px var(--blackA7);
}

.button:hover {
  background-color: var(--mauve3);
}

.button:focus {
  box-shadow: 0 0 0 2px black;
}

// Button.tsx
import styles from './Button.module.css';

export function Button({ children, ...props }) {
  return (
    <button className={styles.button} {...props}>
      {children}
    </button>
  );
}
```

### Controlled Components

```tsx
import { useState } from 'react';
import * as Switch from '@radix-ui/react-switch';

function ControlledSwitch() {
  const [checked, setChecked] = useState(false);

  return (
    <div>
      <Switch.Root
        checked={checked}
        onCheckedChange={setChecked}
        className="switch-root"
      >
        <Switch.Thumb className="switch-thumb" />
      </Switch.Root>
      <p>Switch is {checked ? 'on' : 'off'}</p>
    </div>
  );
}
```

## Common Mistakes

1. **Forgetting asChild prop**: Required to merge props with custom components
```tsx
// Bad - Creates extra wrapper
<Dialog.Trigger>
  <button>Open</button>
</Dialog.Trigger>

// Good - Merges props into button
<Dialog.Trigger asChild>
  <button>Open</button>
</Dialog.Trigger>
```

2. **Not using Portal**: Overlays need Portal for proper rendering
```tsx
// Bad - May have z-index issues
<Dialog.Content>...</Dialog.Content>

// Good - Renders at document root
<Dialog.Portal>
  <Dialog.Content>...</Dialog.Content>
</Dialog.Portal>
```

3. **Missing Tooltip.Provider**: Required for tooltip functionality
```tsx
// Bad - Won't work
<Tooltip.Root>...</Tooltip.Root>

// Good
<Tooltip.Provider>
  <Tooltip.Root>...</Tooltip.Root>
</Tooltip.Provider>
```

4. **Improper forwarding refs**: Custom components need ref forwarding
```tsx
// Bad
const SelectItem = ({ children }) => (
  <Select.Item>{children}</Select.Item>
);

// Good
const SelectItem = React.forwardRef((props, ref) => (
  <Select.Item ref={ref} {...props} />
));
```

5. **Not handling keyboard navigation**: Always test with keyboard
```tsx
// Always ensure focusable elements and proper tab order
<Dialog.Close asChild>
  <button>Close</button>
</Dialog.Close>
```

## Best Practices

1. **Use Radix with a styling solution**: Tailwind, CSS Modules, or styled-components
2. **Leverage composition**: Build complex components from primitives
3. **Test accessibility**: Use keyboard navigation and screen readers
4. **Implement proper focus management**: Use asChild and ref forwarding
5. **Use Portal for overlays**: Prevents z-index and positioning issues
6. **Follow ARIA patterns**: Radix implements them correctly
7. **Customize with data attributes**: Use data-state for styling
8. **Keep styling separate**: Don't mix logic with styles
9. **Use TypeScript**: Excellent type definitions provided
10. **Read the documentation**: Each primitive has specific usage patterns

## When to Use Radix UI

### Use Radix UI When:
- You want full control over styling
- Accessibility is a top priority
- Building a custom design system
- You prefer headless/unstyled components
- Need maximum flexibility
- Working with Tailwind CSS or custom CSS
- Want minimal bundle size for used components
- Building reusable component libraries

### Consider Alternatives When:
- You need pre-styled components (use Chakra UI or Mantine)
- Time to market is critical (use complete libraries)
- Team lacks design resources (use opinionated libraries)
- Need enterprise-ready components out of the box (use Ant Design)
- Prefer all-in-one solutions (use Material UI)

## Interview Questions

1. **Q: What makes Radix UI different from other component libraries?**
   A: Radix provides unstyled, accessible primitives focused on functionality rather than aesthetics. You have complete control over styling while getting robust accessibility, keyboard navigation, and ARIA patterns out of the box.

2. **Q: Why is the asChild prop important?**
   A: asChild uses Radix's Slot component to merge props into child elements instead of creating wrapper divs. This prevents extra DOM nodes and allows proper semantic HTML structure.

3. **Q: How does Radix ensure accessibility?**
   A: Radix implements WAI-ARIA design patterns, manages focus automatically, provides keyboard navigation, uses proper ARIA attributes, and supports screen readers. Each primitive follows accessibility guidelines.

4. **Q: What is the Portal component used for?**
   A: Portal renders children at the end of document.body (or custom container), solving z-index and overflow issues for overlays like dialogs, tooltips, and dropdowns.

5. **Q: How do you style Radix components?**
   A: Use className prop with any styling solution (Tailwind, CSS Modules, styled-components), or use data attributes like data-state for conditional styling based on component state.

## Key Takeaways

1. Radix UI provides unstyled, accessible primitives for complete styling control
2. Accessibility and keyboard navigation are built-in and robust
3. The asChild prop enables proper component composition without extra wrappers
4. Portal component solves overlay rendering and positioning issues
5. Each primitive follows WAI-ARIA design patterns correctly
6. TypeScript support is excellent with full type definitions
7. Bundle size is minimal as you only import what you use
8. Perfect foundation for building custom design systems
9. Works excellently with Tailwind CSS and other styling solutions
10. Requires more setup than styled libraries but provides maximum flexibility

## Resources

- **Official Documentation**: https://www.radix-ui.com/docs/primitives/overview/introduction
- **GitHub Repository**: https://github.com/radix-ui/primitives
- **Component Demos**: https://www.radix-ui.com/docs/primitives/components
- **Radix Icons**: https://www.radix-ui.com/icons
- **Radix Colors**: https://www.radix-ui.com/colors
- **Radix Themes**: https://www.radix-ui.com/themes/docs/overview/getting-started
- **shadcn/ui** (Built on Radix): https://ui.shadcn.com/
- **Radix with Tailwind**: https://www.radix-ui.com/docs/primitives/guides/styling
- **Accessibility Docs**: https://www.radix-ui.com/docs/primitives/overview/accessibility
- **Discord Community**: https://discord.com/invite/7Xb99uG
