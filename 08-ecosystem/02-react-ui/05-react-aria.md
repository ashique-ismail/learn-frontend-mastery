# React Aria

## The Idea

**In plain English:** React Aria is a toolkit of ready-made "behaviors" for building website buttons, menus, and forms that anyone can use — including people who navigate with a keyboard or a screen reader (a program that reads the screen aloud). Instead of giving you pre-styled components, it gives you the invisible rules (like "press Enter to activate a button") and you design how things look yourself.

**Real-world analogy:** Imagine a car manufacturer that sells you a fully engineered engine, steering system, and safety features — but lets you design the car body however you want. You get all the mechanical rules that make a car work correctly, without being locked into one style.

- The engine and steering system = React Aria's hooks (the behavior and accessibility logic)
- The car body and paint = your custom HTML and CSS (the visual design you create)
- The safety standards (seatbelts, airbags) = WAI-ARIA guidelines (the rules that make the UI usable for everyone, including disabled users)

---

## Introduction

React Aria is a library of React Hooks that provides accessible UI primitives for your design system. Created by Adobe, it offers over 40 hooks that handle behavior, accessibility, internationalization, and interactions while leaving DOM structure and styling completely up to you. React Aria focuses on providing the best accessibility support possible, following WAI-ARIA guidelines precisely.

## Installation and Setup

### Basic Installation

```bash
npm install react-aria

# or with yarn
yarn add react-aria

# or with pnpm
pnpm add react-aria
```

### Individual Package Installation

```bash
# Install specific hooks packages
npm install @react-aria/button
npm install @react-aria/dialog
npm install @react-aria/menu
npm install @react-aria/overlays
```

### With React Stately (State Management)

```bash
# React Stately manages component state
npm install react-stately

# or individual packages
npm install @react-stately/toggle
npm install @react-stately/select
```

## Core Hooks

### useButton

```tsx
import { useButton } from '@react-aria/button';
import { useRef } from 'react';

interface ButtonProps {
  children: React.ReactNode;
  onPress?: () => void;
  isDisabled?: boolean;
}

function Button(props: ButtonProps) {
  const ref = useRef<HTMLButtonElement>(null);
  const { buttonProps } = useButton(props, ref);

  return (
    <button
      {...buttonProps}
      ref={ref}
      className="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700 disabled:opacity-50 disabled:cursor-not-allowed"
    >
      {props.children}
    </button>
  );
}

// Usage
function App() {
  return (
    <div className="space-x-4">
      <Button onPress={() => console.log('Pressed!')}>
        Click Me
      </Button>
      <Button isDisabled>Disabled</Button>
    </div>
  );
}

export default Button;
```

### useCheckbox

```tsx
import { useCheckbox } from '@react-aria/checkbox';
import { useToggleState } from '@react-stately/toggle';
import { useRef } from 'react';
import { VisuallyHidden } from '@react-aria/visually-hidden';

interface CheckboxProps {
  children: React.ReactNode;
  isSelected?: boolean;
  onChange?: (isSelected: boolean) => void;
  isDisabled?: boolean;
}

function Checkbox(props: CheckboxProps) {
  const state = useToggleState(props);
  const ref = useRef<HTMLInputElement>(null);
  const { inputProps } = useCheckbox(props, state, ref);

  return (
    <label className="flex items-center gap-2 cursor-pointer">
      <VisuallyHidden>
        <input {...inputProps} ref={ref} />
      </VisuallyHidden>
      <div
        className={`w-5 h-5 border-2 rounded flex items-center justify-center ${
          state.isSelected
            ? 'bg-blue-600 border-blue-600'
            : 'border-gray-400'
        } ${props.isDisabled ? 'opacity-50 cursor-not-allowed' : ''}`}
      >
        {state.isSelected && (
          <svg
            className="w-3 h-3 text-white"
            fill="none"
            stroke="currentColor"
            viewBox="0 0 24 24"
          >
            <path
              strokeLinecap="round"
              strokeLinejoin="round"
              strokeWidth={2}
              d="M5 13l4 4L19 7"
            />
          </svg>
        )}
      </div>
      <span className={props.isDisabled ? 'opacity-50' : ''}>
        {props.children}
      </span>
    </label>
  );
}

export default Checkbox;
```

### useTextField

```tsx
import { useTextField } from '@react-aria/textfield';
import { useRef } from 'react';

interface TextFieldProps {
  label: string;
  description?: string;
  errorMessage?: string;
  value?: string;
  onChange?: (value: string) => void;
  isRequired?: boolean;
  isDisabled?: boolean;
  type?: 'text' | 'email' | 'password' | 'tel' | 'url';
}

function TextField(props: TextFieldProps) {
  const ref = useRef<HTMLInputElement>(null);
  const { labelProps, inputProps, descriptionProps, errorMessageProps } =
    useTextField(props, ref);

  return (
    <div className="flex flex-col gap-1">
      <label {...labelProps} className="font-medium text-sm text-gray-700">
        {props.label}
        {props.isRequired && <span className="text-red-500 ml-1">*</span>}
      </label>
      <input
        {...inputProps}
        ref={ref}
        className="px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500 disabled:bg-gray-100 disabled:cursor-not-allowed"
      />
      {props.description && (
        <div {...descriptionProps} className="text-sm text-gray-500">
          {props.description}
        </div>
      )}
      {props.errorMessage && (
        <div {...errorMessageProps} className="text-sm text-red-600">
          {props.errorMessage}
        </div>
      )}
    </div>
  );
}

export default TextField;
```

### useSelect (Dropdown)

```tsx
import { useSelect } from '@react-aria/select';
import { useSelectState } from '@react-stately/select';
import { useButton } from '@react-aria/button';
import { useListBox, useOption } from '@react-aria/listbox';
import { useOverlay, DismissButton } from '@react-aria/overlays';
import { FocusScope } from '@react-aria/focus';
import { useRef } from 'react';

interface SelectProps {
  label: string;
  items: Array<{ key: string; label: string }>;
  selectedKey?: string;
  onSelectionChange?: (key: string) => void;
}

function Select(props: SelectProps) {
  const state = useSelectState(props);
  const ref = useRef<HTMLButtonElement>(null);
  const { labelProps, triggerProps, valueProps, menuProps } = useSelect(
    props,
    state,
    ref
  );

  return (
    <div className="relative inline-block">
      <div className="flex flex-col gap-1">
        <label {...labelProps} className="font-medium text-sm text-gray-700">
          {props.label}
        </label>
        <button
          {...triggerProps}
          ref={ref}
          className="px-3 py-2 border border-gray-300 rounded-md bg-white text-left flex justify-between items-center min-w-[200px]"
        >
          <span {...valueProps}>
            {state.selectedItem?.label || 'Select an option'}
          </span>
          <span aria-hidden="true">▼</span>
        </button>
      </div>
      {state.isOpen && (
        <Popover onClose={state.close}>
          <ListBox {...menuProps} state={state} />
        </Popover>
      )}
    </div>
  );
}

function Popover({ children, onClose }: any) {
  const ref = useRef<HTMLDivElement>(null);
  const { overlayProps } = useOverlay(
    { onClose, isOpen: true, isDismissable: true },
    ref
  );

  return (
    <FocusScope restoreFocus>
      <div
        {...overlayProps}
        ref={ref}
        className="absolute z-10 mt-1 w-full bg-white border border-gray-300 rounded-md shadow-lg max-h-60 overflow-auto"
      >
        <DismissButton onDismiss={onClose} />
        {children}
        <DismissButton onDismiss={onClose} />
      </div>
    </FocusScope>
  );
}

function ListBox({ state, ...props }: any) {
  const ref = useRef<HTMLUListElement>(null);
  const { listBoxProps } = useListBox(props, state, ref);

  return (
    <ul {...listBoxProps} ref={ref} className="outline-none">
      {[...state.collection].map((item) => (
        <Option key={item.key} item={item} state={state} />
      ))}
    </ul>
  );
}

function Option({ item, state }: any) {
  const ref = useRef<HTMLLIElement>(null);
  const { optionProps, isSelected, isFocused } = useOption(
    { key: item.key },
    state,
    ref
  );

  return (
    <li
      {...optionProps}
      ref={ref}
      className={`px-3 py-2 cursor-pointer outline-none ${
        isFocused ? 'bg-blue-100' : ''
      } ${isSelected ? 'bg-blue-500 text-white' : ''}`}
    >
      {item.rendered}
    </li>
  );
}

export default Select;
```

### useDialog and useModal

```tsx
import { useDialog } from '@react-aria/dialog';
import { useOverlay, useModal } from '@react-aria/overlays';
import { useButton } from '@react-aria/button';
import { FocusScope } from '@react-aria/focus';
import { useRef, useState } from 'react';

interface DialogProps {
  title: string;
  children: React.ReactNode;
  onClose: () => void;
}

function Dialog({ title, children, onClose }: DialogProps) {
  const ref = useRef<HTMLDivElement>(null);
  const { overlayProps, underlayProps } = useOverlay(
    { onClose, isOpen: true, isDismissable: true },
    ref
  );

  const { modalProps } = useModal();
  const { dialogProps, titleProps } = useDialog({}, ref);

  const closeButtonRef = useRef<HTMLButtonElement>(null);
  const { buttonProps: closeButtonProps } = useButton(
    { onPress: onClose },
    closeButtonRef
  );

  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center">
      {/* Backdrop */}
      <div
        {...underlayProps}
        className="fixed inset-0 bg-black bg-opacity-50"
      />

      {/* Dialog */}
      <FocusScope contain restoreFocus autoFocus>
        <div
          {...overlayProps}
          {...dialogProps}
          {...modalProps}
          ref={ref}
          className="relative bg-white rounded-lg shadow-xl max-w-md w-full p-6 z-50"
        >
          <h3 {...titleProps} className="text-xl font-semibold mb-4">
            {title}
          </h3>
          <div className="mb-6">{children}</div>
          <button
            {...closeButtonProps}
            ref={closeButtonRef}
            className="absolute top-4 right-4 text-gray-400 hover:text-gray-600"
          >
            ✕
          </button>
        </div>
      </FocusScope>
    </div>
  );
}

// Usage
function App() {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <div>
      <button
        onClick={() => setIsOpen(true)}
        className="px-4 py-2 bg-blue-600 text-white rounded"
      >
        Open Dialog
      </button>

      {isOpen && (
        <Dialog title="Confirmation" onClose={() => setIsOpen(false)}>
          <p className="mb-4">Are you sure you want to proceed?</p>
          <div className="flex gap-2 justify-end">
            <button
              onClick={() => setIsOpen(false)}
              className="px-4 py-2 border border-gray-300 rounded"
            >
              Cancel
            </button>
            <button
              onClick={() => {
                console.log('Confirmed!');
                setIsOpen(false);
              }}
              className="px-4 py-2 bg-blue-600 text-white rounded"
            >
              Confirm
            </button>
          </div>
        </Dialog>
      )}
    </div>
  );
}

export default Dialog;
```

### useMenu (Context Menu)

```tsx
import { useMenu, useMenuItem, useMenuTrigger } from '@react-aria/menu';
import { useMenuTriggerState } from '@react-stately/menu';
import { useTreeState } from '@react-stately/tree';
import { useButton } from '@react-aria/button';
import { useOverlay } from '@react-aria/overlays';
import { FocusScope } from '@react-aria/focus';
import { useRef } from 'react';

function MenuButton(props: any) {
  const state = useMenuTriggerState(props);
  const ref = useRef<HTMLButtonElement>(null);
  const { menuTriggerProps, menuProps } = useMenuTrigger({}, state, ref);

  const { buttonProps } = useButton(menuTriggerProps, ref);

  return (
    <div className="relative inline-block">
      <button
        {...buttonProps}
        ref={ref}
        className="px-4 py-2 bg-gray-200 rounded hover:bg-gray-300"
      >
        {props.label}
        <span aria-hidden="true" className="ml-2">
          ▼
        </span>
      </button>
      {state.isOpen && (
        <Popover onClose={state.close}>
          <Menu {...menuProps} {...props} onClose={state.close} />
        </Popover>
      )}
    </div>
  );
}

function Menu(props: any) {
  const state = useTreeState(props);
  const ref = useRef<HTMLUListElement>(null);
  const { menuProps } = useMenu(props, state, ref);

  return (
    <ul
      {...menuProps}
      ref={ref}
      className="outline-none py-2"
    >
      {[...state.collection].map((item) => (
        <MenuItem key={item.key} item={item} state={state} onClose={props.onClose} />
      ))}
    </ul>
  );
}

function MenuItem({ item, state, onClose }: any) {
  const ref = useRef<HTMLLIElement>(null);
  const { menuItemProps, isFocused } = useMenuItem(
    {
      key: item.key,
      onAction: () => {
        item.props.onAction?.();
        onClose();
      },
    },
    state,
    ref
  );

  return (
    <li
      {...menuItemProps}
      ref={ref}
      className={`px-4 py-2 cursor-pointer outline-none ${
        isFocused ? 'bg-blue-100' : ''
      }`}
    >
      {item.rendered}
    </li>
  );
}

function Popover({ children, onClose }: any) {
  const ref = useRef<HTMLDivElement>(null);
  const { overlayProps } = useOverlay(
    { onClose, isOpen: true, isDismissable: true },
    ref
  );

  return (
    <FocusScope restoreFocus>
      <div
        {...overlayProps}
        ref={ref}
        className="absolute z-10 mt-1 bg-white border border-gray-300 rounded-md shadow-lg min-w-[200px]"
      >
        {children}
      </div>
    </FocusScope>
  );
}

// Usage with React Stately
import { Item } from '@react-stately/collections';

function App() {
  return (
    <MenuButton label="Actions">
      <Item key="new" onAction={() => console.log('New')}>
        New
      </Item>
      <Item key="open" onAction={() => console.log('Open')}>
        Open
      </Item>
      <Item key="close" onAction={() => console.log('Close')}>
        Close
      </Item>
    </MenuButton>
  );
}

export default MenuButton;
```

### useFocusRing

```tsx
import { useFocusRing } from '@react-aria/focus';
import { useButton } from '@react-aria/button';
import { useRef } from 'react';

function FocusableButton({ children }: { children: React.ReactNode }) {
  const ref = useRef<HTMLButtonElement>(null);
  const { buttonProps } = useButton({ onPress: () => {} }, ref);
  const { isFocusVisible, focusProps } = useFocusRing();

  return (
    <button
      {...buttonProps}
      {...focusProps}
      ref={ref}
      className={`px-4 py-2 rounded ${
        isFocusVisible
          ? 'ring-2 ring-blue-500 ring-offset-2'
          : ''
      } bg-blue-600 text-white hover:bg-blue-700`}
    >
      {children}
    </button>
  );
}

export default FocusableButton;
```

### useTooltip

```tsx
import { useTooltip, useTooltipTrigger } from '@react-aria/tooltip';
import { useTooltipTriggerState } from '@react-stately/tooltip';
import { useRef } from 'react';

interface TooltipProps {
  children: React.ReactNode;
  content: string;
}

function Tooltip({ children, content }: TooltipProps) {
  const state = useTooltipTriggerState({});
  const ref = useRef<HTMLButtonElement>(null);
  const { triggerProps, tooltipProps } = useTooltipTrigger({}, state, ref);

  return (
    <>
      <button {...triggerProps} ref={ref} className="relative">
        {children}
      </button>
      {state.isOpen && (
        <TooltipContent {...tooltipProps}>{content}</TooltipContent>
      )}
    </>
  );
}

function TooltipContent({ children, ...props }: any) {
  const { tooltipProps } = useTooltip(props);

  return (
    <div
      {...tooltipProps}
      className="absolute z-50 px-2 py-1 text-sm bg-gray-900 text-white rounded shadow-lg -top-8 left-1/2 transform -translate-x-1/2 whitespace-nowrap"
    >
      {children}
    </div>
  );
}

export default Tooltip;
```

## Advanced Patterns

### Form with Validation

```tsx
import { useForm } from '@react-aria/form';
import TextField from './TextField';
import { useState } from 'react';

function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [errors, setErrors] = useState<Record<string, string>>({});

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    
    const newErrors: Record<string, string> = {};
    
    if (!email) {
      newErrors.email = 'Email is required';
    } else if (!/\S+@\S+\.\S+/.test(email)) {
      newErrors.email = 'Email is invalid';
    }
    
    if (!password) {
      newErrors.password = 'Password is required';
    } else if (password.length < 8) {
      newErrors.password = 'Password must be at least 8 characters';
    }
    
    setErrors(newErrors);
    
    if (Object.keys(newErrors).length === 0) {
      console.log('Form submitted:', { email, password });
    }
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4 max-w-md">
      <TextField
        label="Email"
        type="email"
        value={email}
        onChange={setEmail}
        errorMessage={errors.email}
        isRequired
      />
      <TextField
        label="Password"
        type="password"
        value={password}
        onChange={setPassword}
        errorMessage={errors.password}
        isRequired
      />
      <button
        type="submit"
        className="w-full px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700"
      >
        Sign In
      </button>
    </form>
  );
}

export default LoginForm;
```

### Accessible Data Table

```tsx
import { useTable, useTableCell, useTableRow } from '@react-aria/table';
import { useTableState } from '@react-stately/table';

function DataTable({ columns, rows }: any) {
  const state = useTableState({ children: null });
  const ref = useRef<HTMLTableElement>(null);
  const { gridProps } = useTable({}, state, ref);

  return (
    <table {...gridProps} ref={ref} className="w-full border-collapse">
      <thead>
        <tr>
          {columns.map((column: any) => (
            <th key={column.key} className="border p-2 bg-gray-100">
              {column.name}
            </th>
          ))}
        </tr>
      </thead>
      <tbody>
        {rows.map((row: any, rowIndex: number) => (
          <TableRow key={rowIndex} row={row} columns={columns} />
        ))}
      </tbody>
    </table>
  );
}

function TableRow({ row, columns }: any) {
  const ref = useRef<HTMLTableRowElement>(null);
  const { rowProps } = useTableRow({ node: row }, ref);

  return (
    <tr {...rowProps} ref={ref} className="hover:bg-gray-50">
      {columns.map((column: any) => (
        <TableCell key={column.key} value={row[column.key]} />
      ))}
    </tr>
  );
}

function TableCell({ value }: any) {
  const ref = useRef<HTMLTableCellElement>(null);
  const { gridCellProps } = useTableCell({}, ref);

  return (
    <td {...gridCellProps} ref={ref} className="border p-2">
      {value}
    </td>
  );
}

export default DataTable;
```

## Common Mistakes

1. **Not using React Stately**: State management hooks are separate
```tsx
// Bad - Missing state management
const { buttonProps } = useButton(props, ref);

// Good - Include state when needed
const state = useToggleState(props);
const { buttonProps } = useButton(props, ref);
```

2. **Forgetting ref forwarding**: Hooks require refs for DOM manipulation
```tsx
// Bad
const { buttonProps } = useButton(props);

// Good
const ref = useRef<HTMLButtonElement>(null);
const { buttonProps } = useButton(props, ref);
```

3. **Not spreading props correctly**: Props must be spread onto elements
```tsx
// Bad
<button onClick={buttonProps.onClick}>

// Good
<button {...buttonProps}>
```

4. **Missing VisuallyHidden**: Native inputs need to be hidden accessibly
```tsx
// Bad
<input style={{ display: 'none' }} />

// Good
<VisuallyHidden>
  <input {...inputProps} />
</VisuallyHidden>
```

5. **Improper overlay structure**: Overlays need proper focus management
```tsx
// Bad
<div>{content}</div>

// Good
<FocusScope restoreFocus>
  <div {...overlayProps}>{content}</div>
</FocusScope>
```

## Best Practices

1. **Use with React Stately**: Combine hooks for complete functionality
2. **Test with keyboard**: Verify Tab, Enter, Escape, Arrow keys
3. **Test with screen readers**: Use NVDA, JAWS, or VoiceOver
4. **Implement focus management**: Use FocusScope for overlays
5. **Handle internationalization**: React Aria supports i18n
6. **Use TypeScript**: Excellent type definitions provided
7. **Follow ARIA patterns**: Trust the hooks' implementations
8. **Separate concerns**: Keep behavior and styling independent
9. **Test mobile interactions**: Touch targets and gestures
10. **Read the documentation**: Each hook has specific requirements

## When to Use React Aria

### Use React Aria When:
- Accessibility is critical (Adobe uses it for their products)
- Building a completely custom design system
- Need maximum control over DOM structure and styling
- Want best-in-class accessibility support
- Require internationalization support
- Building component libraries for others
- Need fine-grained control over behavior
- Want to follow WAI-ARIA patterns precisely

### Consider Alternatives When:
- Need pre-styled components (use Chakra or Mantine)
- Want faster development (use complete libraries)
- Don't need enterprise-level accessibility
- Prefer higher-level abstractions (use Radix or Headless UI)
- Working with Tailwind specifically (use Headless UI)

## Interview Questions

1. **Q: What makes React Aria different from other accessibility libraries?**
   A: React Aria provides hooks for behavior and accessibility without any DOM structure or styling. It's built by Adobe with enterprise-grade accessibility, following WAI-ARIA patterns exactly, and includes internationalization support.

2. **Q: Why separate React Aria and React Stately?**
   A: React Aria handles behavior and accessibility while React Stately manages component state. This separation allows using each independently and promotes better code organization with single responsibility principle.

3. **Q: How does React Aria ensure accessibility?**
   A: It implements WAI-ARIA patterns precisely, manages focus automatically, handles keyboard navigation, provides proper ARIA attributes, supports screen readers, and includes internationalization for global accessibility.

4. **Q: What is the VisuallyHidden component used for?**
   A: VisuallyHidden hides elements visually while keeping them accessible to screen readers. It's used for native form inputs that are styled with custom UI, ensuring screen reader users can still interact with actual form controls.

5. **Q: How do you handle overlays with React Aria?**
   A: Use useOverlay hook for behavior, useModal for modal dialogs, FocusScope for focus management, and DismissButton for keyboard dismissal. This ensures proper focus trapping, restoration, and accessible dismissal.

## Key Takeaways

1. React Aria provides enterprise-grade accessibility via React Hooks
2. Separates behavior from presentation for maximum flexibility
3. Works with React Stately for complete component functionality
4. Follows WAI-ARIA design patterns precisely
5. Built and used by Adobe for their products
6. Includes internationalization support out of the box
7. Requires more setup than complete libraries but provides best accessibility
8. Perfect for building accessible component libraries
9. Excellent TypeScript support throughout
10. Best choice when accessibility is non-negotiable

## Resources

- **Official Documentation**: https://react-spectrum.adobe.com/react-aria/
- **GitHub Repository**: https://github.com/adobe/react-spectrum
- **React Stately**: https://react-spectrum.adobe.com/react-stately/
- **Component Examples**: https://react-spectrum.adobe.com/react-aria/examples/
- **Internationalization**: https://react-spectrum.adobe.com/react-aria/internationalization.html
- **WAI-ARIA Patterns**: https://www.w3.org/WAI/ARIA/apg/
- **Discord Community**: https://discord.gg/S9qsGuF
- **Blog**: https://react-spectrum.adobe.com/blog/
- **React Spectrum** (Complete library): https://react-spectrum.adobe.com/react-spectrum/
- **Storybook Examples**: https://react-spectrum.adobe.com/storybook/
