# Focus Trapping

## Table of Contents
- [Why Focus Trapping Is Required](#why-focus-trapping-is-required)
- [The Focus Trap Algorithm](#the-focus-trap-algorithm)
- [Full Vanilla JavaScript Implementation](#full-vanilla-javascript-implementation)
- [React Hook Implementation](#react-hook-implementation)
- [The `inert` Attribute — Declarative Alternative](#the-inert-attribute--declarative-alternative)
- [`aria-modal="true"` and Screen Reader Implications](#aria-modaltrue-and-screen-reader-implications)
- [Returning Focus to the Trigger Element](#returning-focus-to-the-trigger-element)
- [Nested Focus Traps](#nested-focus-traps)
- [Libraries](#libraries)
- [Common Pitfalls](#common-pitfalls)
- [WCAG 2.2 Success Criteria](#wcag-22-success-criteria)
- [Interview Q&A](#interview-qa)

---

## Why Focus Trapping Is Required

When a modal, dialog, drawer, or dropdown menu opens, keyboard and screen reader users need all focusable interaction confined to that layer. Without a focus trap:

1. **Keyboard users Tab past the overlay** into underlying page content that is visually hidden or covered — they cannot see where focus went and cannot interact meaningfully.
2. **Screen reader virtual cursor** (in Browse/Reading mode) can read content behind the dialog, breaking the conversational model that the dialog is the current context.
3. **Escape routes are confusing** — pressing Shift+Tab from the first element in the dialog sends focus to the browser chrome or the last focusable element on the page, not back around inside the dialog.

### Components That Require a Focus Trap

| Component | Reason |
|---|---|
| Modal dialog | Blocks all underlying content; WCAG 2.1.2 |
| Drawer / side panel | Same as modal; user must explicitly dismiss |
| Confirmation alert dialog | Requires a response before proceeding |
| Toast with actions | If it has interactive elements, trap is needed |
| Mega-menu / dropdown | Trap while menu is open; release on Escape or selection |
| Date picker popup | Contains its own navigation; trap keeps context |

### What Focus Trapping Does NOT Mean

Focus trapping does **not** mean the user can never leave. The user must always be able to dismiss the component (via Escape, a close button, or a commit action) which releases the trap and returns focus. WCAG 2.1.2 (No Keyboard Trap) prohibits trapping focus in a way that has **no documented exit mechanism**.

---

## The Focus Trap Algorithm

### Step 1 — Collect All Focusable Elements

The selector must cover every element that can receive keyboard focus in the natural tab order:

```
button:not([disabled])
[href]
input:not([disabled])
select:not([disabled])
textarea:not([disabled])
[tabindex]:not([tabindex="-1"])
details > summary
audio[controls]
video[controls]
```

Elements are also **excluded** if they are:
- `display: none` or `visibility: hidden`
- Inside a `details` element that is closed (the `summary` is focusable, children are not)
- `inert` (covered in a later section)

### Step 2 — Identify Boundary Nodes

```
firstFocusable = focusableElements[0]
lastFocusable  = focusableElements[focusableElements.length - 1]
```

### Step 3 — Intercept Tab and Shift+Tab at the Boundaries

```
Tab pressed AND focus is on lastFocusable  → preventDefault(), focus(firstFocusable)
Shift+Tab pressed AND focus is on firstFocusable → preventDefault(), focus(lastFocusable)
```

All other keypresses pass through normally.

### Step 4 — Initial Focus Placement

On activation, move focus inside the trap immediately. The recommended targets in priority order:

1. An element with `autofocus` in the markup
2. The first interactive element (confirm button, first input)
3. The dialog container itself (`tabindex="-1"` on the `<div role="dialog">`)

Do **not** focus the close button as the very first element — it gives no context about the dialog content.

### Visualized Flow

```
 Page content  │  Focus Trap (Modal)
───────────────┼──────────────────────
 [nav link]    │  [Close ×]  ← lastFocusable wraps here
 [nav link]    │  [Heading]
               │  [Input 1]
               │  [Input 2]
               │  [Cancel]
 [page btn]    │  [Submit]   ← firstFocusable wraps here
```

When Tab is pressed on `[Submit]`, focus moves to `[Close ×]` instead of `[page btn]`.
When Shift+Tab is pressed on `[Close ×]`, focus moves to `[Submit]` instead of `[nav link]`.

---

## Full Vanilla JavaScript Implementation

This is a production-quality, dependency-free focus trap that handles edge cases including dynamically added elements and disabled elements that become enabled.

```javascript
// focus-trap.js — Zero-dependency focus trap

const FOCUSABLE_SELECTOR = [
  'a[href]',
  'area[href]',
  'input:not([disabled]):not([type="hidden"])',
  'select:not([disabled])',
  'textarea:not([disabled])',
  'button:not([disabled])',
  'iframe',
  'object',
  'embed',
  '[contenteditable]',
  '[tabindex]:not([tabindex="-1"])',
  'details > summary',
  'audio[controls]',
  'video[controls]',
].join(', ');

/**
 * Returns all visible, focusable elements inside a container in DOM order.
 * Filters out elements hidden via CSS or the `inert` attribute.
 */
function getFocusableElements(container) {
  const candidates = Array.from(container.querySelectorAll(FOCUSABLE_SELECTOR));

  return candidates.filter((el) => {
    // Exclude elements with display:none / visibility:hidden
    if (!el.offsetParent && el.offsetWidth === 0 && el.offsetHeight === 0) {
      // offsetParent is null for fixed elements too, check more carefully
      const style = window.getComputedStyle(el);
      if (style.display === 'none' || style.visibility === 'hidden') return false;
    }

    // Exclude elements inside a closed <details>
    let ancestor = el.parentElement;
    while (ancestor && ancestor !== container) {
      if (ancestor.tagName === 'DETAILS' && !ancestor.open) return false;
      ancestor = ancestor.parentElement;
    }

    // Exclude inert elements and their descendants
    if (el.closest('[inert]')) return false;

    return true;
  });
}

/**
 * Creates and activates a focus trap on `container`.
 * Returns a deactivate() function.
 *
 * @param {HTMLElement} container  — The element to trap focus within
 * @param {Object}      options
 * @param {HTMLElement} [options.initialFocus]  — Element to focus on activation
 * @param {boolean}     [options.escapeDeactivates=true]
 * @param {Function}    [options.onDeactivate]
 */
function createFocusTrap(container, options = {}) {
  const {
    initialFocus,
    escapeDeactivates = true,
    onDeactivate,
  } = options;

  let active = false;

  function getFirstFocusable() {
    const els = getFocusableElements(container);
    return els[0] || container;
  }

  function getLastFocusable() {
    const els = getFocusableElements(container);
    return els[els.length - 1] || container;
  }

  function handleKeyDown(event) {
    if (!active) return;

    if (escapeDeactivates && event.key === 'Escape') {
      event.preventDefault();
      deactivate();
      return;
    }

    if (event.key !== 'Tab') return;

    const focusable = getFocusableElements(container);
    if (focusable.length === 0) {
      event.preventDefault();
      return;
    }

    const first = focusable[0];
    const last = focusable[focusable.length - 1];
    const focused = document.activeElement;

    if (event.shiftKey) {
      // Shift+Tab: moving backwards
      if (focused === first || focused === container) {
        event.preventDefault();
        last.focus();
      }
    } else {
      // Tab: moving forwards
      if (focused === last || focused === container) {
        event.preventDefault();
        first.focus();
      }
    }
  }

  // Prevent clicks outside the container from stealing focus
  function handleFocusIn(event) {
    if (!active) return;
    if (!container.contains(event.target)) {
      event.preventDefault();
      getFirstFocusable().focus();
    }
  }

  function activate() {
    if (active) return;
    active = true;

    document.addEventListener('keydown', handleKeyDown, true); // capture phase
    document.addEventListener('focusin', handleFocusIn, true);

    // Set initial focus
    const target = initialFocus || getFirstFocusable();
    // Defer to next tick so the element is fully rendered
    requestAnimationFrame(() => {
      target.focus({ preventScroll: false });
    });
  }

  function deactivate() {
    if (!active) return;
    active = false;

    document.removeEventListener('keydown', handleKeyDown, true);
    document.removeEventListener('focusin', handleFocusIn, true);

    if (typeof onDeactivate === 'function') {
      onDeactivate();
    }
  }

  return { activate, deactivate };
}

// ─── Usage Example ────────────────────────────────────────────────────────────

const modal = document.getElementById('my-modal');
const triggerBtn = document.getElementById('open-modal-btn');
let trap = null;

triggerBtn.addEventListener('click', () => {
  modal.removeAttribute('hidden');
  trap = createFocusTrap(modal, {
    escapeDeactivates: true,
    onDeactivate: closeModal,
  });
  trap.activate();
});

function closeModal() {
  if (trap) {
    trap.deactivate();
    trap = null;
  }
  modal.setAttribute('hidden', '');
  triggerBtn.focus(); // return focus to trigger
}

document.getElementById('modal-close').addEventListener('click', closeModal);
```

### Key Design Decisions

- **Capture phase listeners** (`addEventListener(..., true)`) intercept keyboard events before they bubble, preventing a child's `stopPropagation` from breaking the trap.
- **`focusin` guard** handles programmatic focus theft (e.g., an auto-playing video inside an iframe).
- **`requestAnimationFrame` for initial focus** avoids focus races during CSS transitions.
- **Re-querying focusable elements on each keydown** handles dynamically added/removed content inside the trap (e.g., multi-step wizard where step 2 reveals new inputs).

---

## React Hook Implementation

```typescript
// useFocusTrap.ts
import { useRef, useEffect, useCallback, RefObject } from 'react';

const FOCUSABLE_SELECTOR = [
  'a[href]',
  'area[href]',
  'input:not([disabled]):not([type="hidden"])',
  'select:not([disabled])',
  'textarea:not([disabled])',
  'button:not([disabled])',
  'iframe',
  '[contenteditable]',
  '[tabindex]:not([tabindex="-1"])',
  'details > summary',
].join(', ');

function getFocusableElements(container: HTMLElement): HTMLElement[] {
  return Array.from(
    container.querySelectorAll<HTMLElement>(FOCUSABLE_SELECTOR)
  ).filter((el) => {
    const style = window.getComputedStyle(el);
    return (
      style.display !== 'none' &&
      style.visibility !== 'hidden' &&
      !el.closest('[inert]')
    );
  });
}

interface UseFocusTrapOptions {
  /** Whether the trap is currently active */
  active: boolean;
  /** Element to focus on activation; defaults to first focusable element */
  initialFocusRef?: RefObject<HTMLElement>;
  /** Called when Escape is pressed */
  onEscape?: () => void;
}

/**
 * Traps keyboard focus inside `containerRef` when `active` is true.
 * Restores focus to the previously focused element on deactivation.
 */
export function useFocusTrap<T extends HTMLElement>({
  active,
  initialFocusRef,
  onEscape,
}: UseFocusTrapOptions): RefObject<T> {
  const containerRef = useRef<T>(null);
  const previousFocusRef = useRef<HTMLElement | null>(null);

  const handleKeyDown = useCallback(
    (event: KeyboardEvent) => {
      const container = containerRef.current;
      if (!container) return;

      if (event.key === 'Escape') {
        event.preventDefault();
        onEscape?.();
        return;
      }

      if (event.key !== 'Tab') return;

      const focusable = getFocusableElements(container);
      if (focusable.length === 0) {
        event.preventDefault();
        return;
      }

      const first = focusable[0];
      const last = focusable[focusable.length - 1];
      const focused = document.activeElement as HTMLElement;

      if (event.shiftKey) {
        if (focused === first || focused === container) {
          event.preventDefault();
          last.focus();
        }
      } else {
        if (focused === last || focused === container) {
          event.preventDefault();
          first.focus();
        }
      }
    },
    [onEscape]
  );

  const handleFocusIn = useCallback((event: FocusEvent) => {
    const container = containerRef.current;
    if (!container) return;
    if (!container.contains(event.target as Node)) {
      const focusable = getFocusableElements(container);
      focusable[0]?.focus();
    }
  }, []);

  useEffect(() => {
    if (!active) return;

    // Capture previously focused element before trap activates
    previousFocusRef.current = document.activeElement as HTMLElement;

    document.addEventListener('keydown', handleKeyDown, true);
    document.addEventListener('focusin', handleFocusIn, true);

    // Set initial focus
    requestAnimationFrame(() => {
      const target =
        initialFocusRef?.current ??
        (containerRef.current
          ? getFocusableElements(containerRef.current)[0]
          : null) ??
        containerRef.current;
      target?.focus({ preventScroll: false });
    });

    return () => {
      document.removeEventListener('keydown', handleKeyDown, true);
      document.removeEventListener('focusin', handleFocusIn, true);

      // Return focus to wherever the user was before the trap opened
      previousFocusRef.current?.focus({ preventScroll: true });
    };
  }, [active, handleKeyDown, handleFocusIn, initialFocusRef]);

  return containerRef;
}
```

```typescript
// Modal.tsx — consuming the hook
import React, { useRef } from 'react';
import { createPortal } from 'react-dom';
import { useFocusTrap } from './useFocusTrap';

interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  title: string;
  children: React.ReactNode;
}

export const Modal: React.FC<ModalProps> = ({
  isOpen,
  onClose,
  title,
  children,
}) => {
  const closeBtnRef = useRef<HTMLButtonElement>(null);

  // containerRef is typed as RefObject<HTMLDivElement>
  const containerRef = useFocusTrap<HTMLDivElement>({
    active: isOpen,
    onEscape: onClose,
    // Optionally focus a specific element — e.g., the primary action button
    // initialFocusRef: confirmBtnRef,
  });

  if (!isOpen) return null;

  return createPortal(
    // Overlay click closes the modal — a common UX affordance
    <div
      className="modal-overlay"
      role="presentation"
      onClick={onClose}
      style={{
        position: 'fixed',
        inset: 0,
        background: 'rgba(0,0,0,0.5)',
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center',
        zIndex: 1000,
      }}
    >
      {/* Stop clicks inside the dialog from closing the modal */}
      <div
        ref={containerRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby="modal-title"
        tabIndex={-1}
        onClick={(e) => e.stopPropagation()}
        style={{
          background: '#fff',
          borderRadius: 8,
          padding: 24,
          minWidth: 320,
          maxWidth: '90vw',
          outline: 'none',
        }}
      >
        <div style={{ display: 'flex', justifyContent: 'space-between', marginBottom: 16 }}>
          <h2 id="modal-title" style={{ margin: 0 }}>
            {title}
          </h2>
          <button
            ref={closeBtnRef}
            onClick={onClose}
            aria-label="Close dialog"
          >
            ×
          </button>
        </div>

        <div>{children}</div>

        <div style={{ marginTop: 24, display: 'flex', gap: 8 }}>
          <button onClick={onClose}>Cancel</button>
          <button onClick={onClose}>Confirm</button>
        </div>
      </div>
    </div>,
    document.body
  );
};
```

---

## The `inert` Attribute — Declarative Alternative

### How It Works

The HTML `inert` attribute was standardized in HTML 5.1 and broadly shipped in 2022–2023. When applied to an element:

- All focusable descendants become **unfocusable** (removed from the tab order)
- The element and its descendants are **excluded from the accessibility tree** (screen readers cannot navigate into them)
- User events (click, keypress, pointer) are **not dispatched** to inert descendants
- `document.querySelector` and `getElementById` still find inert elements — `inert` is a focus/interaction guard, not a DOM filter

```html
<!-- inert applied to the page background while a modal is open -->
<div id="app-content" inert>
  <!-- All links, buttons, inputs here: unfocusable, unannounced -->
</div>

<div id="modal" role="dialog" aria-modal="true" aria-labelledby="modal-title">
  <!-- Focus naturally stays here because everything else is inert -->
</div>
```

### Using `inert` in React

```typescript
// InertModal.tsx — using inert as the trapping mechanism

import React, { useEffect, useRef } from 'react';
import { createPortal } from 'react-dom';

interface InertModalProps {
  isOpen: boolean;
  onClose: () => void;
  title: string;
  children: React.ReactNode;
}

export const InertModal: React.FC<InertModalProps> = ({
  isOpen,
  onClose,
  title,
  children,
}) => {
  const modalRef = useRef<HTMLDivElement>(null);
  const previousFocusRef = useRef<HTMLElement | null>(null);
  const appRoot = document.getElementById('root'); // or your app shell element

  useEffect(() => {
    if (isOpen) {
      previousFocusRef.current = document.activeElement as HTMLElement;

      // Make everything behind the modal inert
      appRoot?.setAttribute('inert', '');

      // Set initial focus inside modal
      requestAnimationFrame(() => {
        const firstFocusable = modalRef.current?.querySelector<HTMLElement>(
          'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
        );
        firstFocusable?.focus();
      });
    } else {
      // Remove inert when modal closes
      appRoot?.removeAttribute('inert');

      // Restore previous focus
      previousFocusRef.current?.focus({ preventScroll: true });
    }

    return () => {
      // Cleanup in case component unmounts while open
      appRoot?.removeAttribute('inert');
    };
  }, [isOpen, appRoot]);

  // Escape key handling
  useEffect(() => {
    if (!isOpen) return;
    const handleEscape = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose();
    };
    document.addEventListener('keydown', handleEscape);
    return () => document.removeEventListener('keydown', handleEscape);
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return createPortal(
    <div
      style={{ position: 'fixed', inset: 0, zIndex: 1000 }}
      onClick={onClose}
      role="presentation"
    >
      <div
        ref={modalRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby="inert-modal-title"
        onClick={(e) => e.stopPropagation()}
        style={{ /* ... styles */ }}
      >
        <h2 id="inert-modal-title">{title}</h2>
        {children}
        <button onClick={onClose}>Close</button>
      </div>
    </div>,
    document.body
  );
};
```

### Browser Support (as of mid-2025)

| Browser | Version | Notes |
|---|---|---|
| Chrome / Edge | 102+ | Full support |
| Firefox | 112+ | Full support |
| Safari | 15.5+ | Full support |
| iOS Safari | 15.5+ | Full support |
| Samsung Internet | 20+ | Full support |

Global support is ~96%+ as of 2025. A polyfill exists: `wicg-inert` on npm.

### `inert` vs Keydown-based Trapping — Trade-offs

| Aspect | `inert` approach | Keydown approach |
|---|---|---|
| Screen reader | Blocks SR virtual cursor from background | Does NOT block SR virtual cursor |
| Implementation | 2 lines (`setAttribute`/`removeAttribute`) | ~50 lines of careful JS |
| Portal content | Portals outside `#root` must also be inerted | Trap container naturally covers portals |
| Nested modals | Each level must manage inert carefully | Nested trap stack patterns are well-tested |
| Dynamic content | Browser handles new focusable children | Must re-query on mutation |
| Browser support | ~96% (2025) | 100% |

**Best practice**: Use `inert` on background content **and** a keydown-based trap inside the dialog. This gives you both SR virtual cursor blocking (`inert`) and robust Tab cycling (keydown handler).

---

## `aria-modal="true"` and Screen Reader Implications

### What `aria-modal` Does

`aria-modal="true"` on a `role="dialog"` element is an **ARIA property** that signals to screen readers: "treat this dialog as if everything outside it were removed from the accessibility tree."

```html
<div role="dialog" aria-modal="true" aria-labelledby="dlg-title">
  <h2 id="dlg-title">Confirm Delete</h2>
  <!-- ... -->
</div>
```

### What `aria-modal` Does NOT Do

**`aria-modal="true"` is NOT a focus trap.** It does not:

- Prevent Tab from moving focus outside the dialog
- Prevent keyboard users from leaving the dialog
- Apply any visual containment
- Work reliably in all screen reader + browser combinations

### Browser/Screen Reader Reality (critical for interviews)

| Screen Reader + Browser | `aria-modal` behavior |
|---|---|
| NVDA + Chrome | Respected — virtual cursor confined |
| JAWS + Chrome | Respected |
| VoiceOver + Safari | Partially respected — inconsistent in older versions |
| VoiceOver + iOS | Generally respected |
| Narrator + Edge | Respected |
| NVDA + Firefox | Sometimes ignored, depends on NVDA version |

**The correct pattern is: `aria-modal="true"` + actual focus trap + `inert` on background.**

They serve different axes:
- `aria-modal` → screen reader virtual/browse mode cursor
- Focus trap (keydown) → Tab key focus cycle
- `inert` → both SR cursor AND Tab, for background elements

```typescript
// Correct combination
<div
  role="dialog"
  aria-modal="true"           // ← SR virtual cursor hint
  aria-labelledby="title-id"
  tabIndex={-1}               // ← allows programmatic focus on container
  ref={trapRef}               // ← keydown trap applied here
>
```

---

## Returning Focus to the Trigger Element

### Why It Matters

When a modal closes, focus must return to the element that triggered it. If it does not:

- Keyboard users lose their place in the document — they have no idea where focus landed
- Screen reader users hear either silence or an unexpected context announcement
- The user must Tab from the top of the page to find their position again

This is required by **WCAG 2.4.3 Focus Order** — focus order must be meaningful and predictable.

### Storing and Restoring Focus

```typescript
// Pattern: store before opening, restore on close
function useReturnFocus(isOpen: boolean) {
  const triggerRef = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (isOpen) {
      // Capture the trigger the instant the overlay opens
      triggerRef.current = document.activeElement as HTMLElement;
    } else {
      // Restore focus when overlay closes
      triggerRef.current?.focus({ preventScroll: true });
      triggerRef.current = null;
    }
  }, [isOpen]);
}
```

### Edge Cases

```typescript
// Edge case 1: Trigger element was removed from the DOM while modal was open
function safeRestoreFocus(trigger: HTMLElement | null, fallback: HTMLElement) {
  if (trigger && document.body.contains(trigger)) {
    trigger.focus({ preventScroll: true });
  } else {
    // Fallback: first focusable in main content, or body
    fallback.focus({ preventScroll: true });
  }
}

// Edge case 2: Trigger was disabled while modal was open (e.g., form submitted)
function focusFirstEnabled(candidates: HTMLElement[]) {
  const target = candidates.find(
    (el) => !(el as HTMLButtonElement).disabled && !el.getAttribute('aria-disabled')
  );
  target?.focus();
}

// Edge case 3: SPA route change closes the modal (trigger is on previous route)
// → Focus the main heading of the new page
useEffect(() => {
  if (modalClosedDueToNavigation) {
    document.querySelector<HTMLElement>('main h1, [role="main"] h1')?.focus();
  }
}, [location.pathname]);
```

### Drawer Close Button Placement

Always place the close button **at the end** of the drawer's DOM order (or at least reachable without excessive tabbing), but **initially focus** the first meaningful interactive element. The close button should be focusable, not the first thing announced.

---

## Nested Focus Traps

When a modal opens a second modal (or a drawer opens a confirmation dialog), you need a **trap stack**.

### The Stack Pattern

```typescript
// focusTrapStack.ts — manages a LIFO stack of active traps

const trapStack: Array<() => void> = []; // each entry is a deactivate fn

export function pushTrap(deactivateFn: () => void): void {
  trapStack.push(deactivateFn);
}

export function popTrap(): void {
  const deactivate = trapStack.pop();
  deactivate?.();
}

export function peekTrap(): (() => void) | undefined {
  return trapStack[trapStack.length - 1];
}
```

```typescript
// useFocusTrapStack.ts — a hook that auto-manages the stack

import { useEffect } from 'react';
import { pushTrap, popTrap } from './focusTrapStack';

export function useFocusTrapStack(isActive: boolean, onDeactivate: () => void) {
  useEffect(() => {
    if (!isActive) return;

    pushTrap(onDeactivate);

    return () => {
      // When this trap is cleaned up, pop it from the stack
      popTrap();
    };
  }, [isActive, onDeactivate]);
}
```

### Example: Modal Within a Modal

```typescript
// OuterModal.tsx
const OuterModal: React.FC<{ isOpen: boolean; onClose: () => void }> = ({
  isOpen,
  onClose,
}) => {
  const [innerOpen, setInnerOpen] = React.useState(false);
  const outerTrapRef = useFocusTrap<HTMLDivElement>({ active: isOpen && !innerOpen });

  return isOpen ? (
    <div ref={outerTrapRef} role="dialog" aria-modal="true" aria-labelledby="outer-title">
      <h2 id="outer-title">Outer Modal</h2>
      <button onClick={() => setInnerOpen(true)}>Open Inner Modal</button>
      <button onClick={onClose}>Close Outer</button>

      <InnerModal
        isOpen={innerOpen}
        onClose={() => setInnerOpen(false)}
      />
    </div>
  ) : null;
};

// InnerModal.tsx
const InnerModal: React.FC<{ isOpen: boolean; onClose: () => void }> = ({
  isOpen,
  onClose,
}) => {
  // Inner trap is active; outer trap is paused by { active: isOpen && !innerOpen }
  const innerTrapRef = useFocusTrap<HTMLDivElement>({ active: isOpen, onEscape: onClose });

  return isOpen ? (
    <div ref={innerTrapRef} role="dialog" aria-modal="true" aria-labelledby="inner-title">
      <h2 id="inner-title">Confirm Action</h2>
      <button onClick={onClose}>Cancel</button>
      <button onClick={onClose}>Confirm</button>
    </div>
  ) : null;
};
```

### Key Rules for Nested Traps

1. **Only one trap is active at a time** — the outermost visible dialog pauses when a new one opens.
2. **Closing the inner trap reactivates the outer trap** — focus returns to the trigger inside the outer modal.
3. **`inert` must be layered** — when inner opens, outer gets `inert`; when inner closes, outer's `inert` is removed.
4. **Escape key stack** — pressing Escape should close the innermost trap, not all of them.

```typescript
// Escape handler for nested traps: only close the topmost
document.addEventListener('keydown', (e) => {
  if (e.key === 'Escape') {
    e.preventDefault();
    const topDeactivate = peekTrap();
    topDeactivate?.(); // closes innermost trap only
  }
});
```

---

## Libraries

### `focus-trap` npm package

The canonical low-level JS library. Framework-agnostic.

```bash
npm install focus-trap
```

```javascript
import { createFocusTrap } from 'focus-trap';

const trap = createFocusTrap('#modal', {
  initialFocus: '#modal-heading',
  escapeDeactivates: true,
  returnFocusOnDeactivate: true,
  allowOutsideClick: false, // prevent clicks outside from stealing focus
  checkCanFocusInitialElement: (el) => !el.closest('[aria-hidden]'),
});

openModalButton.addEventListener('click', () => {
  modal.removeAttribute('hidden');
  trap.activate();
});

closeModalButton.addEventListener('click', () => {
  trap.deactivate();
  modal.setAttribute('hidden', '');
});
```

```typescript
// focus-trap-react wrapper
import FocusTrap from 'focus-trap-react';

const Modal: React.FC<{ isOpen: boolean; onClose: () => void }> = ({
  isOpen,
  onClose,
  children,
}) => {
  if (!isOpen) return null;

  return (
    <FocusTrap
      active={isOpen}
      focusTrapOptions={{
        initialFocus: false, // focus container, not first element
        escapeDeactivates: true,
        onDeactivate: onClose,
        allowOutsideClick: true,
        returnFocusOnDeactivate: true,
      }}
    >
      <div role="dialog" aria-modal="true" aria-labelledby="modal-title" tabIndex={-1}>
        <h2 id="modal-title">Modal Title</h2>
        {children}
        <button onClick={onClose}>Close</button>
      </div>
    </FocusTrap>
  );
};
```

### `@radix-ui/react-dialog`

Radix UI ships a fully accessible, headless dialog that handles focus trapping, `aria-modal`, scroll lock, and focus return out of the box.

```bash
npm install @radix-ui/react-dialog
```

```typescript
import * as Dialog from '@radix-ui/react-dialog';

// Radix handles all a11y requirements internally.
// FocusTrap is built into Dialog.Content via @radix-ui/react-focus-guards.

const ConfirmDialog: React.FC<{
  open: boolean;
  onOpenChange: (open: boolean) => void;
}> = ({ open, onOpenChange }) => {
  return (
    <Dialog.Root open={open} onOpenChange={onOpenChange}>
      <Dialog.Trigger asChild>
        <button>Open Dialog</button>
      </Dialog.Trigger>

      <Dialog.Portal>
        {/* Overlay is outside the portal so Radix can manage z-index cleanly */}
        <Dialog.Overlay className="dialog-overlay" />

        <Dialog.Content
          className="dialog-content"
          // Radix auto-focuses the first interactive element
          // onOpenAutoFocus — override initial focus
          // onCloseAutoFocus — override return focus target
          onOpenAutoFocus={(e) => {
            // e.preventDefault(); // prevent default auto-focus
            // customRef.current?.focus();
          }}
          onCloseAutoFocus={(e) => {
            // e.preventDefault(); // prevent return focus (rarely needed)
          }}
        >
          <Dialog.Title>Are you sure?</Dialog.Title>
          <Dialog.Description>This action cannot be undone.</Dialog.Description>

          <div style={{ display: 'flex', gap: 8, marginTop: 16 }}>
            <Dialog.Close asChild>
              <button>Cancel</button>
            </Dialog.Close>
            <button onClick={() => { /* do action */ onOpenChange(false); }}>
              Confirm
            </button>
          </div>

          <Dialog.Close asChild>
            <button aria-label="Close">×</button>
          </Dialog.Close>
        </Dialog.Content>
      </Dialog.Portal>
    </Dialog.Root>
  );
};
```

**What Radix handles automatically:**
- Focus trap via `@radix-ui/react-focus-scope`
- `aria-modal="true"` on `Dialog.Content`
- `aria-labelledby` wired to `Dialog.Title`
- `aria-describedby` wired to `Dialog.Description`
- Return focus to `Dialog.Trigger` on close
- Escape key closes the dialog
- Body scroll lock

### `@angular/cdk/a11y` FocusTrap

Angular CDK provides the `FocusTrap` service and the `cdkTrapFocus` directive.

```bash
npm install @angular/cdk
```

```typescript
// dialog.component.ts — using the FocusTrap class directly
import { Component, ElementRef, Input, OnDestroy, OnInit, ViewChild } from '@angular/core';
import { FocusTrap, FocusTrapFactory } from '@angular/cdk/a11y';

@Component({
  selector: 'app-dialog',
  template: `
    <div
      *ngIf="isOpen"
      #dialogContainer
      role="dialog"
      aria-modal="true"
      [attr.aria-labelledby]="titleId"
      class="dialog"
    >
      <h2 [attr.id]="titleId">{{ title }}</h2>
      <ng-content></ng-content>
      <button (click)="close()">Close</button>
    </div>
  `,
})
export class DialogComponent implements OnInit, OnDestroy {
  @Input() isOpen = false;
  @Input() title = '';
  @ViewChild('dialogContainer') dialogContainer!: ElementRef<HTMLElement>;

  titleId = `dialog-title-${Math.random().toString(36).slice(2)}`;

  private focusTrap: FocusTrap | null = null;
  private previousFocusedElement: HTMLElement | null = null;

  constructor(private focusTrapFactory: FocusTrapFactory) {}

  ngOnInit() {
    if (this.isOpen) this.activateTrap();
  }

  ngOnChanges() {
    if (this.isOpen) {
      this.activateTrap();
    } else {
      this.deactivateTrap();
    }
  }

  private activateTrap() {
    this.previousFocusedElement = document.activeElement as HTMLElement;

    // CDK creates and activates a focus trap on the container element
    this.focusTrap = this.focusTrapFactory.create(
      this.dialogContainer.nativeElement
    );
    // focusInitialElementWhenReady returns a promise that resolves once
    // the initial focus element is focusable (handles lazy rendering)
    this.focusTrap.focusInitialElementWhenReady();
  }

  private deactivateTrap() {
    this.focusTrap?.destroy();
    this.focusTrap = null;
    this.previousFocusedElement?.focus({ preventScroll: true });
  }

  close() {
    this.deactivateTrap();
  }

  ngOnDestroy() {
    this.deactivateTrap();
  }
}
```

```typescript
// Simpler Angular approach: cdkTrapFocus directive
import { A11yModule } from '@angular/cdk/a11y';

// In your module:
// imports: [A11yModule]

@Component({
  selector: 'app-simple-dialog',
  template: `
    <div
      *ngIf="isOpen"
      cdkTrapFocus
      [cdkTrapFocusAutoCapture]="true"
      role="dialog"
      aria-modal="true"
      [attr.aria-labelledby]="titleId"
    >
      <h2 [attr.id]="titleId">{{ title }}</h2>
      <ng-content></ng-content>
      <button (click)="close()">Close</button>
    </div>
  `,
})
export class SimpleDialogComponent {
  @Input() isOpen = false;
  @Input() title = '';
  titleId = `dlg-${Math.random().toString(36).slice(2)}`;

  @Output() closed = new EventEmitter<void>();

  close() {
    this.closed.emit();
  }
}
```

**`cdkTrapFocusAutoCapture="true"`**: CDK automatically focuses the first tabbable element on init and restores focus when the directive is destroyed.

### Angular CDK FocusMonitor

For more granular control, `FocusMonitor` tracks the origin of focus events (keyboard, mouse, touch, program):

```typescript
import { FocusMonitor } from '@angular/cdk/a11y';

@Component({
  selector: 'app-focused-button',
  template: `<button #btn>Click me</button>`,
})
export class FocusedButtonComponent implements AfterViewInit, OnDestroy {
  @ViewChild('btn') button!: ElementRef<HTMLElement>;

  constructor(private focusMonitor: FocusMonitor) {}

  ngAfterViewInit() {
    this.focusMonitor.monitor(this.button).subscribe((origin) => {
      // origin: 'keyboard' | 'mouse' | 'touch' | 'program' | null
      if (origin === 'keyboard') {
        console.log('Keyboard user — show prominent focus ring');
      }
    });
  }

  ngOnDestroy() {
    this.focusMonitor.stopMonitoring(this.button);
  }
}
```

---

## Common Pitfalls

### 1. iframes

An `<iframe>` is focusable as a single unit, but once focus enters the iframe's document, keyboard events are dispatched to the iframe's `document`, not the parent. Your `keydown` listener on the parent document **does not fire** while focus is inside an iframe.

```typescript
// Problem: Tab inside iframe escapes your trap
const handleKeyDown = (event: KeyboardEvent) => {
  // This never fires for Tab presses inside the iframe's document!
};

// Mitigation 1: Make the iframe inert if it's decorative or non-critical
iframeEl.setAttribute('inert', '');

// Mitigation 2: If the iframe must be interactive, communicate via postMessage
// Parent → listens for Tab on the iframe wrapper, not inside it.

// Mitigation 3: tabIndex="-1" on iframe to exclude it from the trap entirely
// (if user doesn't need to Tab into it)
<iframe src="..." tabIndex="-1" aria-hidden="true" />

// Mitigation 4: focus-trap library handles this with sentinel elements
// surrounding the iframe that catch Tab bleed-through
```

### 2. Shadow DOM

Custom elements using Shadow DOM create a separate focus tree. `querySelectorAll` does not pierce shadow roots.

```typescript
// getFocusableElements() misses shadow DOM children
// ❌ This does NOT traverse shadow roots:
container.querySelectorAll(FOCUSABLE_SELECTOR);

// ✅ Walk shadow roots explicitly:
function getFocusableDeep(container: HTMLElement): HTMLElement[] {
  const results: HTMLElement[] = [];

  function walk(root: Element | ShadowRoot) {
    const children = Array.from(root.children) as Element[];
    for (const child of children) {
      if (child.matches?.(FOCUSABLE_SELECTOR)) {
        results.push(child as HTMLElement);
      }
      // Recurse into shadow root if it exists
      if ((child as HTMLElement).shadowRoot) {
        walk((child as HTMLElement).shadowRoot!);
      } else {
        walk(child);
      }
    }
  }

  walk(container);
  return results;
}

// Note: shadow roots with `mode: 'closed'` cannot be walked
// In that case, rely on the custom element itself being focusable
// as a single tab stop and designing the custom element to manage
// internal keyboard navigation
```

### 3. Dynamically Rendered Content

A multi-step wizard or accordion inside a modal can change which elements are focusable. A trap that cached the element list at activation time will break.

```typescript
// ❌ Cached list — breaks when content changes
useEffect(() => {
  const focusable = getFocusableElements(container); // cached!
  // ...
}, [isActive]); // never re-runs when content changes

// ✅ Re-query on every keydown — always fresh
const handleKeyDown = (event: KeyboardEvent) => {
  if (event.key !== 'Tab') return;
  const focusable = getFocusableElements(container); // live query
  // ...
};
```

### 4. React Portals

When a modal is rendered via `ReactDOM.createPortal(children, document.body)`, the dialog is inside the focus trap's container in the **React tree**, but in the **DOM tree** it's a sibling of `#root`. This affects `inert` management:

```typescript
// ❌ Inert on #root won't affect portal content since portal is outside #root
appRoot.setAttribute('inert', '');
// The portal itself is NOT inerted!

// ✅ Also inert any portal containers that appear behind the modal
const portalContainers = document.querySelectorAll('[data-portal-root]');
portalContainers.forEach((el) => {
  if (el !== modalPortalEl) el.setAttribute('inert', '');
});

// ✅ Or use the focus-trap library which handles portals via sentinel nodes
```

### 5. Transition / Animation Timing

CSS transitions can delay when elements are visible and therefore focusable. Setting initial focus before the transition completes means the element may not yet be scrolled into view or fully interactive.

```typescript
// ❌ Focus set synchronously — element may be transitioning
useEffect(() => {
  if (isOpen) firstFocusable?.focus(); // fires before CSS transition completes
}, [isOpen]);

// ✅ Defer focus until after transition using requestAnimationFrame or transitionend
useEffect(() => {
  if (!isOpen) return;

  const el = containerRef.current;
  const handleTransitionEnd = () => {
    const firstFocusable = getFocusableElements(el!)[0];
    firstFocusable?.focus();
    el!.removeEventListener('transitionend', handleTransitionEnd);
  };

  // If no transition, requestAnimationFrame is enough
  el?.addEventListener('transitionend', handleTransitionEnd);
  requestAnimationFrame(() => {
    // Fallback if no transitionend fires (no CSS transition)
    if (document.activeElement === document.body) {
      getFocusableElements(el!)[0]?.focus();
    }
  });
}, [isOpen]);
```

### 6. `tabindex` Attribute Mutation

An element can have its `tabindex` changed (e.g., -1 → 0) programmatically after the trap activates:

```typescript
// ❌ Element had tabindex="-1" at trap creation, added to tab order later
button.setAttribute('tabindex', '0'); // now it's in the tab order
// But if you cached focusable list, this element is missed

// ✅ Re-query on every Tab keydown (already covered by the re-query pattern above)
// ✅ OR use MutationObserver to invalidate the cache
const observer = new MutationObserver(() => {
  focusableCache = null; // invalidate
});
observer.observe(container, { attributeFilter: ['tabindex', 'disabled', 'inert'], subtree: true });
```

---

## WCAG 2.2 Success Criteria

### 2.1.1 Keyboard (Level A)

> All functionality of the content is operable through a keyboard interface without requiring specific timings for individual keystrokes, except where the underlying function requires input that depends on the path of the user's movement and not just the endpoints.

**What this means for focus trapping:**
- The dialog must be fully operable by keyboard: open, interact with all controls, and close.
- No functionality inside the modal can be mouse-only.
- The Tab cycle must reach every interactive control.

**How to comply:**
- Ensure all interactive elements inside the trap are in the focusable elements query.
- Never rely on hover for secondary actions inside the modal.

### 2.1.2 No Keyboard Trap (Level A)

> If keyboard focus can be moved to a component of the page using a keyboard interface, then focus can be moved away from that component using only a keyboard interface, and, if it requires more than unmodified arrow or Tab keys, the user is advised of the method for moving focus away.

**What this means for focus trapping:**
- You MUST provide a documented exit mechanism.
- Escape key to close is the standard exit mechanism — always document it.
- If Escape is not available, the close button must be reachable and clearly labeled.

**How to comply:**
```typescript
// Always handle Escape
document.addEventListener('keydown', (e) => {
  if (e.key === 'Escape' && trapIsActive) {
    closeDialog();
  }
});

// Announce to screen reader users how to close (in addition to close button)
<div role="dialog" aria-describedby="dialog-close-hint">
  <p id="dialog-close-hint" className="sr-only">
    Press Escape or activate the Close button to dismiss this dialog.
  </p>
  ...
</div>
```

### 2.4.3 Focus Order (Level A)

> If a Web page can be navigated sequentially and the navigation sequences affect meaning or operation, focusable components receive focus in an order that preserves meaning and operation.

**What this means for focus trapping:**
- When the dialog opens, focus must move into the dialog immediately — not stay on the trigger.
- When the dialog closes, focus must return to the trigger element (or the next logical location).
- The Tab order inside the dialog must follow the visual/logical reading order.
- Focus must never land on a non-interactive element during navigation (unless it has `tabindex="-1"` for programmatic placement only).

**How to comply:**
```typescript
// Focus order inside dialog should match visual layout (DOM order = visual order)
// Avoid using position:absolute to visually reorder elements that are in a
// different DOM order — this creates a mismatch between visual and keyboard order

// ❌ Visual order: [Input] [Cancel] [Submit]
//    DOM order:    [Submit] [Input] [Cancel]
// Tab order follows DOM, creating a confusing mismatch.

// ✅ Ensure DOM order matches visual order
<form>
  <input name="name" />
  <div class="actions">
    <button type="button">Cancel</button>
    <button type="submit">Submit</button>
  </div>
</form>
```

---

## Interview Q&A

### Q1: Explain what a focus trap is and why modals need one.

**A:** A focus trap is a keyboard interaction pattern where Tab and Shift+Tab key navigation is confined to a specific container — usually a modal dialog or overlay — while it is open. Modals need focus traps because they overlay the entire page visually, but without a trap, keyboard users can Tab out of the modal into the underlying content they cannot see or interact with. This breaks their navigation context and violates WCAG 2.4.3 (Focus Order). The trap cycles focus between the first and last focusable elements inside the container, and provides an exit mechanism (typically Escape) that closes the modal and returns focus to the trigger.

---

### Q2: Walk me through the focus trap algorithm at the code level.

**A:** The algorithm has four steps:

1. **Collect focusable elements**: Query the container for all elements that accept keyboard focus (`button`, `a[href]`, `input`, `select`, `textarea`, `[tabindex]:not([tabindex="-1"])`, `details > summary`). Filter out elements hidden via CSS (`display:none`, `visibility:hidden`) and elements inside closed `<details>` blocks.

2. **Identify boundaries**: `first = focusable[0]`, `last = focusable[last]`.

3. **Intercept Tab/Shift+Tab**: Add a `keydown` listener (capture phase). When `Tab` is pressed and `document.activeElement === last`, call `preventDefault()` and `first.focus()`. When `Shift+Tab` is pressed and `document.activeElement === first`, call `preventDefault()` and `last.focus()`.

4. **Handle initial focus and cleanup**: On activation, call `first.focus()` (or a specified initial focus target). On deactivation, remove the listener and call `previousFocus.focus()` to restore the trigger.

One important subtlety: the focusable list should be re-queried on **every keydown**, not cached, because dynamically rendered content (multi-step forms, accordions) can add or remove focusable elements.

---

### Q3: What is the `inert` attribute and how does it differ from a keydown-based focus trap?

**A:** The `inert` HTML attribute, when applied to an element, makes all of its descendants non-focusable, excluded from the accessibility tree, and non-interactive. Unlike a keydown trap which only intercepts Tab navigation, `inert` also blocks screen reader virtual cursor navigation (Browse mode in NVDA/JAWS, VO reading cursor), which `aria-modal` only hints at but does not guarantee.

The difference in implementation is significant: a keydown trap requires ~50 lines of careful JS to handle edge cases; `inert` requires two lines: `appRoot.setAttribute('inert', '')` on open and `appRoot.removeAttribute('inert')` on close.

However, `inert` doesn't replace a keydown trap entirely for modals. Content rendered via React portals outside the app root is NOT covered by `inert` on the root. Also, `inert` alone does not enforce Tab cycling inside the modal — it just prevents Tab from reaching outside content. The recommended pattern is to combine both: `inert` on the background for maximum SR compatibility, and a keydown trap inside the dialog for robust Tab cycling.

---

### Q4: `aria-modal="true"` is on my dialog — do I still need a focus trap?

**A:** Yes, absolutely. `aria-modal="true"` is a screen reader hint, not a focus containment mechanism. It tells screen readers to treat the dialog as if the rest of the page doesn't exist — but it only affects the SR virtual cursor (Browse mode), not actual keyboard Tab focus. A keyboard user (with or without a screen reader) can still Tab out of the dialog into the underlying page content unless a programmatic focus trap is active. Additionally, `aria-modal` support is inconsistent across screen reader and browser combinations — older VoiceOver versions do not fully respect it, and some NVDA versions ignore it in certain contexts. The correct implementation is: `role="dialog"` + `aria-modal="true"` + a real focus trap (keydown-based or `inert`-based).

---

### Q5: Why does focus return to the trigger element matter, and what happens if you get it wrong?

**A:** WCAG 2.4.3 requires that focus order preserves meaning and operation. When a dialog closes, the user's position in the page must be preserved. If focus is dropped (goes to `document.body` or to a completely unrelated element), a keyboard-only user has to Tab from their current position (wherever the browser decided to land) to get back to where they were. For a page with 50+ focusable elements before the trigger, that's 50+ Tab presses. A screen reader user will hear the entire page context change abruptly.

The fix is always storing `document.activeElement` immediately before or the instant after the overlay opens, then calling `storedElement.focus()` the moment the overlay closes. Edge cases include: the trigger being removed from the DOM while the modal was open (fall back to the closest meaningful element), the trigger being disabled (fall back to the next interactive sibling), or a route change causing the trigger to be on a previous page (focus the `<h1>` of the new page).

---

### Q6: How do you handle focus traps with nested modals?

**A:** The key principle is that only one trap should be active at a time. The outer trap is **paused** (not destroyed) when the inner modal opens, and **reactivated** when the inner modal closes.

The implementation uses a LIFO stack of deactivate functions. When a new trap activates, it pushes itself onto the stack. Pressing Escape calls `pop()` on the stack — which closes only the innermost trap and reactivates the one below it. Critically, the inner modal should also apply `inert` to the outer modal (not just the page background), so screen readers don't read behind the inner modal.

---

### Q7: What are the WCAG success criteria that focus trapping directly supports?

**A:**

- **2.1.1 Keyboard (Level A)**: All modal functionality must be keyboard operable. The trap ensures Tab reaches every interactive control inside the modal.
- **2.1.2 No Keyboard Trap (Level A)**: Paradoxically, this SC *requires* that there is a documented escape mechanism from the trap. Implementing Escape to close satisfies this. A trap without an exit violates 2.1.2.
- **2.4.3 Focus Order (Level A)**: Focus must move into the dialog on open (not stay on the trigger) and return to the trigger on close. The Tab cycle inside must follow logical order.

---

### Q8: What are the hardest edge cases in focus trap implementations?

**A:**

1. **iframes**: The parent `document`'s `keydown` listener does not fire when focus is inside an iframe's document. The focus-trap library handles this by placing invisible sentinel elements before and after the iframe that intercept Tab bleed-through.

2. **Shadow DOM**: `querySelectorAll` does not traverse shadow roots. Custom elements may contain focusable content that is invisible to a standard focusable-elements query. You must walk the shadow DOM tree recursively, but `mode: 'closed'` roots are inaccessible.

3. **Dynamically rendered content**: If a multi-step form inside the modal replaces step 1 inputs with step 2 inputs, a cached focusable list is stale. Always re-query on each keydown event.

4. **React portals outside the app root**: `inert` applied to `#root` does not cover portal content appended to `document.body`. You must explicitly track and inert all background portal containers.

5. **CSS transition timing**: Setting focus synchronously on open fires before the transition completes, potentially focusing a not-yet-visible element. Use `transitionend` event or `requestAnimationFrame` to defer focus placement.

---

### Q9: Compare the three major library options for focus trapping in React/Angular.

**A:**

| Library | Approach | When to Use |
|---|---|---|
| `focus-trap` + `focus-trap-react` | Low-level, handles iframes/shadow DOM, sentinels, pausing, stacking | When you need full control and have complex requirements |
| `@radix-ui/react-dialog` | Opinionated, headless component; handles trap + ARIA + scroll lock as a unit | New React projects; want a11y correct out of the box with styling freedom |
| `@angular/cdk/a11y` FocusTrap | Angular-idiomatic, integrates with CDK's Live Announcer and FocusMonitor | Angular projects; Material Design ecosystem |

For greenfield React projects, Radix UI is the best default because it ships a semantically complete, specification-compliant dialog pattern. For existing codebases or components that aren't dialogs (drawers, tooltips with complex content), `focus-trap-react` gives you the trap behavior as a drop-in. In Angular, `cdkTrapFocus` directive with `cdkTrapFocusAutoCapture` is the idiomatic choice and handles focus restoration automatically.

---

### Q10: What does WCAG 2.1.2 "No Keyboard Trap" say, and how is it compatible with intentional focus trapping in modals?

**A:** WCAG 2.1.2 states that if a keyboard user can move focus into a component, they must be able to move focus *away* using only standard keys (Tab, arrow keys, Escape), or if non-standard keys are required, the user must be informed of the method.

This is not a contradiction with focus trapping — it is precisely why every focus trap must implement an **exit mechanism**. A modal that traps focus but provides no way to exit (no close button, no Escape key, no submit that closes it) *would* violate 2.1.2. A modal that traps focus but has a clearly labeled Close button and responds to Escape is fully compliant.

The specification's intent is to prevent focus from becoming stuck in a component with no escape route — for example, a Flash object or a custom widget that captures all keyboard events and has no way to exit. Intentional modal trapping with a documented exit (Escape key, Close button) satisfies the criterion.

---

## Key Takeaways

1. A focus trap cycles Tab/Shift+Tab between the first and last focusable elements inside a container — implement it with a capture-phase `keydown` listener, not a bubble-phase one.
2. Always re-query focusable elements on each keydown — never cache the list — to handle dynamic content changes inside the modal.
3. `aria-modal="true"` is a screen reader hint, not a focus trap. You need both.
4. The `inert` attribute is a powerful declarative alternative that also blocks SR virtual cursor; combine it with a keydown trap for maximum compatibility.
5. Always store the trigger element's reference before opening the overlay and restore focus to it on close — this is required by WCAG 2.4.3.
6. Nested traps require a LIFO stack; Escape should close only the innermost trap.
7. `@radix-ui/react-dialog` and `@angular/cdk/a11y` handle the full focus management lifecycle; prefer them over hand-rolled solutions unless you have special constraints.
8. WCAG 2.1.2 (No Keyboard Trap) requires an exit mechanism, not the absence of trapping — Escape + close button satisfies it.
9. iframes, shadow DOM, and React portals are the three hardest edge cases and each requires specific handling that most hand-rolled traps miss.
10. Focus trap behaviour should be verified with a real keyboard (unplug the mouse) and with NVDA+Chrome or VoiceOver+Safari to confirm both Tab containment and SR virtual cursor containment.

## Resources

### Specifications and Standards
- [WCAG 2.2 — 2.1.1 Keyboard](https://www.w3.org/WAI/WCAG22/Understanding/keyboard.html)
- [WCAG 2.2 — 2.1.2 No Keyboard Trap](https://www.w3.org/WAI/WCAG22/Understanding/no-keyboard-trap.html)
- [WCAG 2.2 — 2.4.3 Focus Order](https://www.w3.org/WAI/WCAG22/Understanding/focus-order.html)
- [ARIA Authoring Practices: Modal Dialog Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/dialog-modal/)
- [HTML `inert` spec — WHATWG](https://html.spec.whatwg.org/multipage/interaction.html#inert)

### Libraries
- [focus-trap on GitHub](https://github.com/focus-trap/focus-trap)
- [focus-trap-react](https://github.com/focus-trap/focus-trap-react)
- [Radix UI Dialog](https://www.radix-ui.com/primitives/docs/components/dialog)
- [Angular CDK a11y](https://material.angular.io/cdk/a11y/overview)
- [wicg-inert polyfill](https://github.com/WICG/inert)

### Articles
- [MDN — `inert` attribute](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/inert)
- [web.dev — Making content inert](https://developer.chrome.com/docs/css-ui/inert)
- [Smashing Magazine — Accessible Modal Windows](https://www.smashingmagazine.com/2021/07/accessible-dialog-from-scratch/)
- [Hidde de Vries — aria-modal and virtual cursor](https://hidde.blog/using-javascript-to-trap-focus-in-an-element/)
