# Ark UI

## What It Is

Ark UI is a **headless component library** built on top of Zag.js (a state machine-based DOM library). It provides accessible, unstyled components that work across React, Solid, Vue, and Svelte with a consistent API.

Created by the team behind Chakra UI. Think of it as the "brain" layer — logic, ARIA, keyboard navigation, state — without visual styling.

---

## Key Differentiators

- **State machine-based**: Components powered by Zag.js state machines — predictable, testable behavior
- **Multi-framework**: Same component behavior across React, Vue, Solid, Svelte
- **Headless**: Zero styling — integrate with any CSS approach (Tailwind, CSS Modules, styled-components)
- **Composable**: Each component exposes granular parts for full control

---

## Installation (React)

```bash
npm install @ark-ui/react
```

---

## Example: Dialog

```tsx
import { Dialog } from '@ark-ui/react';

function ConfirmDialog({ onConfirm }) {
  return (
    <Dialog.Root>
      <Dialog.Trigger asChild>
        <button>Delete Account</button>
      </Dialog.Trigger>

      <Dialog.Backdrop className="fixed inset-0 bg-black/50" />

      <Dialog.Positioner className="fixed inset-0 flex items-center justify-center">
        <Dialog.Content className="bg-white rounded-lg p-6 max-w-md w-full shadow-xl">
          <Dialog.Title className="text-lg font-semibold">
            Delete Account?
          </Dialog.Title>
          <Dialog.Description className="text-gray-600 mt-2">
            This action is permanent and cannot be undone.
          </Dialog.Description>

          <div className="flex gap-3 mt-6 justify-end">
            <Dialog.CloseTrigger asChild>
              <button className="px-4 py-2 border rounded">Cancel</button>
            </Dialog.CloseTrigger>
            <Dialog.CloseTrigger asChild>
              <button
                onClick={onConfirm}
                className="px-4 py-2 bg-red-600 text-white rounded"
              >
                Delete
              </button>
            </Dialog.CloseTrigger>
          </div>
        </Dialog.Content>
      </Dialog.Positioner>
    </Dialog.Root>
  );
}
```

Ark handles: focus trap, Escape key, scroll lock, ARIA attributes, backdrop click-to-close. You handle: all the CSS.

---

## Example: Combobox (Complex Accessible Component)

```tsx
import { Combobox, createListCollection } from '@ark-ui/react';

const frameworks = createListCollection({
  items: [
    { label: 'React', value: 'react' },
    { label: 'Angular', value: 'angular' },
    { label: 'Vue', value: 'vue' },
  ],
});

function FrameworkSelect() {
  return (
    <Combobox.Root collection={frameworks} onValueChange={({ value }) => console.log(value)}>
      <Combobox.Label className="block text-sm font-medium mb-1">
        Framework
      </Combobox.Label>

      <Combobox.Control className="flex border rounded">
        <Combobox.Input
          className="flex-1 px-3 py-2 outline-none"
          placeholder="Select a framework..."
        />
        <Combobox.Trigger asChild>
          <button className="px-3 border-l">▼</button>
        </Combobox.Trigger>
      </Combobox.Control>

      <Combobox.Positioner>
        <Combobox.Content className="bg-white border rounded shadow-lg mt-1">
          <Combobox.List>
            {frameworks.items.map(item => (
              <Combobox.Item
                key={item.value}
                item={item}
                className="px-3 py-2 cursor-pointer hover:bg-gray-100 data-[highlighted]:bg-blue-50"
              >
                <Combobox.ItemText>{item.label}</Combobox.ItemText>
              </Combobox.Item>
            ))}
          </Combobox.List>
        </Combobox.Content>
      </Combobox.Positioner>
    </Combobox.Root>
  );
}
```

---

## `asChild` Pattern

The `asChild` prop (from Radix UI, adopted by Ark) merges the component's props and event handlers onto the child element instead of rendering its own element:

```tsx
// Without asChild: renders <button><a href="/link">Link</a></button> (invalid HTML)
<Dialog.Trigger>
  <a href="/link">Link</a>
</Dialog.Trigger>

// With asChild: renders <a href="/link">Link</a> with dialog trigger behavior
<Dialog.Trigger asChild>
  <a href="/link">Link</a>
</Dialog.Trigger>
```

---

## Data Attributes for Styling

Ark exposes state via data attributes for CSS styling:

```css
/* Style based on component state */
[data-highlighted] { background: #eff6ff; }
[data-selected] { background: #dbeafe; font-weight: 600; }
[data-disabled] { opacity: 0.5; cursor: not-allowed; }
[data-open] { display: block; }
[data-state="open"] .chevron { transform: rotate(180deg); }
```

---

## Comparison with Radix UI

| | Ark UI | Radix UI |
|---|---|---|
| Framework support | React, Solid, Vue, Svelte | React only |
| State machines | Zag.js (explicit state) | Internal hooks |
| Components | 35+ | 30+ |
| Date/time pickers | Yes (complex) | No |
| TypeScript | Excellent | Excellent |
| Bundle | Per-package | Per-package |

Both are strong choices. Prefer Ark if you need Vue/Solid/Svelte or complex date pickers. Prefer Radix if React-only and you value its more widespread adoption.

---

## Common Interview Questions

**Q: What's Zag.js and why does Ark UI use state machines?**
Zag.js is a DOM-agnostic state machine library for UI components. State machines make component behavior explicit and predictable — every state transition is defined. This means the same behavioral logic can power React, Vue, and Solid components without rewriting it per framework.

**Q: When would you choose Ark UI over shadcn/ui?**
Shadcn/ui is React-only and built on Radix. Ark UI works across React, Vue, Solid, and Svelte with consistent API. Choose Ark for multi-framework design systems or when you need components like DatePicker/TimePicker that Radix doesn't provide.
