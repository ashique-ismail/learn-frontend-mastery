# Implement Modal with Focus Trap

## The Idea

**In plain English:** A modal is a pop-up window that appears on top of a page and demands your attention before you can do anything else. A focus trap is the rule that makes your keyboard (Tab key) stay inside that pop-up — so you can't accidentally wander back into the content hidden behind it.

**Real-world analogy:** Imagine you walk into a bank and a security door locks behind you in the entrance booth. You must complete the check-in process (show ID, sign in) before the inner door opens and you can walk into the bank. You can move between the ID scanner, the pen, and the sign-in screen, but you cannot leave the booth or access anything behind you until you are done.

- The locked entrance booth = the modal dialog
- The ID scanner, pen, and sign-in screen = the focusable elements (buttons, inputs) inside the modal
- The rule that stops you leaving the booth = the focus trap (Tab cycling only within the modal)

---

## The Focus Trap Algorithm

When a modal is open, Tab and Shift+Tab must cycle through focusable elements *within the modal* only.

```ts
const FOCUSABLE_SELECTORS = [
  'a[href]',
  'button:not([disabled])',
  'input:not([disabled])',
  'select:not([disabled])',
  'textarea:not([disabled])',
  '[tabindex]:not([tabindex="-1"])',
  '[contenteditable="true"]',
].join(', ');

function getFocusableElements(container: HTMLElement): HTMLElement[] {
  return Array.from(container.querySelectorAll<HTMLElement>(FOCUSABLE_SELECTORS))
    .filter(el => !el.closest('[hidden]'));
}

function trapFocus(container: HTMLElement) {
  function handleKeyDown(e: KeyboardEvent) {
    if (e.key !== 'Tab') return;

    const focusable = getFocusableElements(container);
    if (focusable.length === 0) return;

    const first = focusable[0];
    const last = focusable[focusable.length - 1];

    if (e.shiftKey) {
      if (document.activeElement === first) {
        e.preventDefault();
        last.focus();
      }
    } else {
      if (document.activeElement === last) {
        e.preventDefault();
        first.focus();
      }
    }
  }

  container.addEventListener('keydown', handleKeyDown);
  return () => container.removeEventListener('keydown', handleKeyDown);
}
```

---

## Complete React Modal

```tsx
interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  title: string;
  children: ReactNode;
}

function Modal({ isOpen, onClose, title, children }: ModalProps) {
  const modalRef = useRef<HTMLDivElement>(null);
  const triggerRef = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (isOpen) {
      // Store the element that opened the modal
      triggerRef.current = document.activeElement as HTMLElement;

      // Focus the modal itself (or first focusable element)
      const modal = modalRef.current;
      if (modal) {
        const firstFocusable = getFocusableElements(modal)[0];
        (firstFocusable ?? modal).focus();

        // Set up focus trap
        return trapFocus(modal);
      }
    } else {
      // Return focus to trigger element when modal closes
      triggerRef.current?.focus();
    }
  }, [isOpen]);

  // Close on Escape key
  useEffect(() => {
    function handleEscape(e: KeyboardEvent) {
      if (e.key === 'Escape' && isOpen) onClose();
    }
    document.addEventListener('keydown', handleEscape);
    return () => document.removeEventListener('keydown', handleEscape);
  }, [isOpen, onClose]);

  // Scroll lock
  useEffect(() => {
    if (isOpen) {
      document.body.style.overflow = 'hidden';
      return () => { document.body.style.overflow = ''; };
    }
  }, [isOpen]);

  if (!isOpen) return null;

  return createPortal(
    <>
      {/* Backdrop */}
      <div
        className="modal-backdrop"
        onClick={onClose}
        aria-hidden="true"
      />

      {/* Dialog */}
      <div
        ref={modalRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby="modal-title"
        className="modal"
        tabIndex={-1}  // makes modal focusable as a fallback
      >
        <div className="modal-header">
          <h2 id="modal-title">{title}</h2>
          <button
            onClick={onClose}
            aria-label="Close dialog"
            className="modal-close"
          >
            ×
          </button>
        </div>
        <div className="modal-body">
          {children}
        </div>
      </div>
    </>,
    document.body
  );
}
```

---

## CSS

```css
.modal-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  z-index: 100;
}

.modal {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 101;
  background: white;
  border-radius: 8px;
  padding: 24px;
  max-width: 500px;
  width: 90vw;
  max-height: 85vh;
  overflow-y: auto;
  outline: none; /* suppress focus ring on the container itself */
}

@media (prefers-reduced-motion: no-preference) {
  .modal {
    animation: slideIn 200ms ease-out;
  }
}
```

---

## Using Radix UI (Production Alternative)

```tsx
import * as Dialog from '@radix-ui/react-dialog';

function MyModal({ open, onOpenChange }) {
  return (
    <Dialog.Root open={open} onOpenChange={onOpenChange}>
      <Dialog.Portal>
        <Dialog.Overlay className="modal-backdrop" />
        <Dialog.Content className="modal">
          <Dialog.Title>My Modal</Dialog.Title>
          <Dialog.Description>...</Dialog.Description>
          <Dialog.Close asChild>
            <button aria-label="Close">×</button>
          </Dialog.Close>
        </Dialog.Content>
      </Dialog.Portal>
    </Dialog.Root>
  );
}
// Radix handles: focus trap, scroll lock, Escape key, ARIA attributes, return focus
```

---

## Common Interview Questions

**Q: Why use `createPortal`?**
Renders the modal as a direct child of `<body>`, avoiding stacking context issues (`z-index` problems with parent elements that have `transform`, `filter`, or `opacity`).

**Q: What ARIA attributes does a modal need?**
`role="dialog"`, `aria-modal="true"` (tells screen readers content behind is inert), `aria-labelledby` pointing to the modal title.

**Q: What happens to background content when the modal is open?**
Add `aria-hidden="true"` to the main app root (not the modal) so screen readers can't navigate behind the modal. The `inert` attribute (modern) also handles keyboard navigation blocking.
