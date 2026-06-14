# Accessible Modal Dialog

## The Idea

**In plain English:** A modal dialog is a UI panel that appears on top of the page and demands the user's attention. While it is open, the rest of the page is inaccessible — users can only interact with the modal's content. When the modal closes, the user's focus returns to wherever it was before.

**Real-world analogy:** Think of a bank teller's window with a privacy screen.

- When you step up to the window, a **privacy screen slides across** the counter — you can only interact with the teller in front of you; the other tellers behind the screen still exist but are off-limits
- The **teller's window** = the modal content (the thing you're supposed to interact with)
- The **privacy screen** = the backdrop overlay (visually dims and blocks the rest of the page)
- The **"transaction complete — window closes"** = the modal closing on confirm
- When the transaction is done, you step back to **exactly where you were standing** in the queue — that is focus returning to the trigger element
- If you get frustrated and walk away, the privacy screen retracts — that is pressing Escape to dismiss

The critical insight: a modal is not just a visually centered `<div>`. Accessibility requires a focus trap (Tab cycles only inside), scroll lock (the page behind doesn't scroll), and ARIA dialog semantics so screen readers know they are inside a constrained context.

---

## Learning Objectives

- Use the `<dialog>` element vs a custom `role="dialog"` `<div>` approach (trade-offs)
- Render the modal in a portal so CSS `overflow:hidden` ancestors don't clip it
- Implement a robust focus trap with Tab and Shift+Tab handling
- Lock body scroll without losing scroll position on iOS Safari
- Return focus to the trigger element when the modal closes
- Set correct ARIA: `role="dialog"`, `aria-modal`, `aria-labelledby`, `aria-describedby`

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Center the modal visually | ✅ via `position: fixed` + transforms | — |
| Dim the backdrop | ✅ via a fixed overlay | — |
| Show/hide the modal | ✅ via `display: none` toggle | — |
| Trap Tab focus inside modal | ❌ | Focus is a DOM/JS concept |
| Lock body scroll on iOS | ❌ | iOS ignores `overflow:hidden` on body |
| Return focus to trigger on close | ❌ | JS must store and restore focus |
| Announce "dialog" to screen readers | ❌ | ARIA attributes need JS to set them |
| Portal render to `<body>` | ❌ | DOM insertion requires JS |

---

## HTML & CSS Foundation

```html
<!-- Trigger button (lives anywhere in the page) -->
<button id="open-dialog-btn" aria-haspopup="dialog">
  Open settings
</button>

<!-- Portal target — modal rendered here by JS -->
<div id="modal-root"></div>

<!-- Modal structure (rendered inside #modal-root via portal) -->
<div class="modal-backdrop" aria-hidden="true"></div>
<div
  role="dialog"
  aria-modal="true"
  aria-labelledby="modal-title"
  aria-describedby="modal-desc"
  class="modal"
  tabindex="-1"
>
  <div class="modal-inner">
    <header class="modal-header">
      <h2 id="modal-title" class="modal-title">Settings</h2>
      <button
        class="modal-close"
        aria-label="Close settings dialog"
      >
        <span aria-hidden="true">✕</span>
      </button>
    </header>

    <div class="modal-body">
      <p id="modal-desc">Update your notification and privacy preferences.</p>
      <!-- Modal content -->
    </div>

    <footer class="modal-footer">
      <button class="btn btn-secondary">Cancel</button>
      <button class="btn btn-primary">Save changes</button>
    </footer>
  </div>
</div>
```

```css
:root {
  --modal-z: 500;
  --modal-max-width: 560px;
  --modal-radius: 12px;
  --modal-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  --backdrop-bg: rgba(0, 0, 0, 0.5);
  --modal-transition: 0.2s ease;
}

/* ── Backdrop ── */
.modal-backdrop {
  position: fixed;
  inset: 0;
  background: var(--backdrop-bg);
  z-index: var(--modal-z);
  /* Animation handled by parent class */
}

/* ── Modal container ── */
.modal {
  position: fixed;
  inset: 0;
  z-index: calc(var(--modal-z) + 1);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 16px;
}

/* ── Modal inner panel ── */
.modal-inner {
  background: #fff;
  border-radius: var(--modal-radius);
  box-shadow: var(--modal-shadow);
  width: 100%;
  max-width: var(--modal-max-width);
  max-height: calc(100dvh - 32px);
  display: flex;
  flex-direction: column;
  overflow: hidden;

  /* Entrance animation */
  animation: modal-in var(--modal-transition) both;
}

@keyframes modal-in {
  from { opacity: 0; transform: scale(0.96) translateY(8px); }
  to   { opacity: 1; transform: scale(1)    translateY(0);    }
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 20px 24px 16px;
  border-bottom: 1px solid #eee;
}

.modal-title {
  margin: 0;
  font-size: 1.2rem;
}

.modal-close {
  width: 36px; height: 36px;
  border-radius: 50%;
  border: none;
  background: none;
  font-size: 1rem;
  cursor: pointer;
  display: flex; align-items: center; justify-content: center;
}
.modal-close:hover { background: #f5f5f5; }
.modal-close:focus { outline: 2px solid #1a73e8; outline-offset: 2px; }

.modal-body {
  padding: 20px 24px;
  overflow-y: auto;
  flex: 1;
}

.modal-footer {
  padding: 16px 24px 20px;
  border-top: 1px solid #eee;
  display: flex;
  justify-content: flex-end;
  gap: 12px;
}

/* ── Scroll lock class (added to body by JS) ── */
body.modal-open {
  overflow: hidden;
}

/* ── Reduced motion ── */
@media (prefers-reduced-motion: reduce) {
  .modal-inner { animation: none; }
}
```

---

## React Implementation

### `useFocusTrap` Hook

```tsx
// useFocusTrap.ts
import { useEffect, RefObject } from 'react';

const FOCUSABLE_SELECTORS = [
  'a[href]',
  'button:not([disabled])',
  'input:not([disabled])',
  'select:not([disabled])',
  'textarea:not([disabled])',
  '[tabindex]:not([tabindex="-1"])',
].join(', ');

export function useFocusTrap(containerRef: RefObject<HTMLElement>, active: boolean) {
  useEffect(() => {
    if (!active || !containerRef.current) return;

    const container = containerRef.current;
    const getFocusable = () =>
      Array.from(container.querySelectorAll<HTMLElement>(FOCUSABLE_SELECTORS));

    // Focus the first focusable element
    getFocusable()[0]?.focus();

    const handler = (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return;

      const focusable = getFocusable();
      const first = focusable[0];
      const last  = focusable[focusable.length - 1];

      if (!focusable.length) { e.preventDefault(); return; }

      if (e.shiftKey) {
        if (document.activeElement === first) {
          e.preventDefault();
          last?.focus();
        }
      } else {
        if (document.activeElement === last) {
          e.preventDefault();
          first?.focus();
        }
      }
    };

    document.addEventListener('keydown', handler);
    return () => document.removeEventListener('keydown', handler);
  }, [active, containerRef]);
}
```

### `useScrollLock` Hook

```tsx
// useScrollLock.ts
import { useEffect } from 'react';

export function useScrollLock(active: boolean) {
  useEffect(() => {
    if (!active) return;

    const scrollY = window.scrollY;
    const originalStyle = document.body.style.cssText;

    document.body.style.cssText = `
      overflow: hidden;
      position: fixed;
      top: -${scrollY}px;
      left: 0;
      right: 0;
    `;

    return () => {
      document.body.style.cssText = originalStyle;
      window.scrollTo(0, scrollY);
    };
  }, [active]);
}
```

### Modal Component

```tsx
// Modal.tsx
import {
  useRef, useEffect, useCallback, ReactNode,
  KeyboardEvent
} from 'react';
import { createPortal } from 'react-dom';
import { useFocusTrap } from './useFocusTrap';
import { useScrollLock } from './useScrollLock';

interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  title: string;
  description?: string;
  children: ReactNode;
  footer?: ReactNode;
  closeOnBackdropClick?: boolean;
}

export function Modal({
  isOpen,
  onClose,
  title,
  description,
  children,
  footer,
  closeOnBackdropClick = true,
}: ModalProps) {
  const dialogRef    = useRef<HTMLDivElement>(null);
  const triggerRef   = useRef<Element | null>(null);

  // Store the element that opened the modal
  useEffect(() => {
    if (isOpen) {
      triggerRef.current = document.activeElement;
    } else {
      // Return focus on close
      (triggerRef.current as HTMLElement)?.focus();
    }
  }, [isOpen]);

  useFocusTrap(dialogRef, isOpen);
  useScrollLock(isOpen);

  const handleKeyDown = useCallback(
    (e: KeyboardEvent<HTMLDivElement>) => {
      if (e.key === 'Escape') onClose();
    },
    [onClose]
  );

  const handleBackdropClick = useCallback(() => {
    if (closeOnBackdropClick) onClose();
  }, [closeOnBackdropClick, onClose]);

  if (!isOpen) return null;

  return createPortal(
    <>
      {/* Backdrop */}
      <div
        className="modal-backdrop"
        aria-hidden="true"
        onClick={handleBackdropClick}
      />

      {/* Dialog */}
      <div
        ref={dialogRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby="modal-title"
        aria-describedby={description ? 'modal-desc' : undefined}
        className="modal"
        onKeyDown={handleKeyDown}
        tabIndex={-1}
      >
        <div
          className="modal-inner"
          onClick={e => e.stopPropagation()} // prevent backdrop click from bubbling
        >
          <header className="modal-header">
            <h2 id="modal-title" className="modal-title">{title}</h2>
            <button
              className="modal-close"
              aria-label={`Close ${title} dialog`}
              onClick={onClose}
            >
              <span aria-hidden="true">✕</span>
            </button>
          </header>

          <div className="modal-body">
            {description && (
              <p id="modal-desc">{description}</p>
            )}
            {children}
          </div>

          {footer && (
            <footer className="modal-footer">{footer}</footer>
          )}
        </div>
      </div>
    </>,
    document.body
  );
}
```

### Usage

```tsx
// SettingsPage.tsx
import { useState } from 'react';
import { Modal } from './Modal';

export function SettingsPage() {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <>
      <button aria-haspopup="dialog" onClick={() => setIsOpen(true)}>
        Open settings
      </button>

      <Modal
        isOpen={isOpen}
        onClose={() => setIsOpen(false)}
        title="Settings"
        description="Update your notification and privacy preferences."
        footer={
          <>
            <button className="btn btn-secondary" onClick={() => setIsOpen(false)}>
              Cancel
            </button>
            <button className="btn btn-primary">Save changes</button>
          </>
        }
      >
        <p>Modal content goes here.</p>
      </Modal>
    </>
  );
}
```

---

## Angular Implementation

```typescript
// modal.component.ts
import {
  Component, Input, Output, EventEmitter,
  ElementRef, AfterViewInit, OnDestroy,
  HostListener, ViewChild, signal, effect
} from '@angular/core';
import { NgIf } from '@angular/common';

@Component({
  selector: 'app-modal',
  standalone: true,
  imports: [NgIf],
  template: `
    <ng-container *ngIf="isOpen">
      <!-- Teleport to body via Angular CDK Overlay would be ideal;
           without CDK, we use high z-index -->
      <div class="modal-backdrop" aria-hidden="true" (click)="onBackdropClick()"></div>
      <div
        #dialogEl
        role="dialog"
        aria-modal="true"
        [attr.aria-labelledby]="titleId"
        class="modal"
        (keydown.escape)="close.emit()"
      >
        <div class="modal-inner" (click)="$event.stopPropagation()">
          <header class="modal-header">
            <h2 [id]="titleId" class="modal-title">{{ title }}</h2>
            <button class="modal-close" [attr.aria-label]="'Close ' + title" (click)="close.emit()">
              <span aria-hidden="true">✕</span>
            </button>
          </header>
          <div class="modal-body">
            <ng-content></ng-content>
          </div>
          <footer class="modal-footer" *ngIf="hasFooter">
            <ng-content select="[modal-footer]"></ng-content>
          </footer>
        </div>
      </div>
    </ng-container>
  `,
})
export class ModalComponent implements AfterViewInit, OnDestroy {
  @Input() title = '';
  @Input() isOpen = false;
  @Input() hasFooter = false;
  @Input() closeOnBackdropClick = true;
  @Output() close = new EventEmitter<void>();
  @ViewChild('dialogEl') dialogEl?: ElementRef<HTMLElement>;

  readonly titleId = `modal-title-${Math.random().toString(36).slice(2)}`;

  private previouslyFocused: HTMLElement | null = null;
  private scrollY = 0;

  ngAfterViewInit() {
    if (this.isOpen) this.onOpen();
  }

  ngOnDestroy() { this.onClose(); }

  private onOpen() {
    this.previouslyFocused = document.activeElement as HTMLElement;
    this.scrollY = window.scrollY;
    document.body.style.cssText = `overflow:hidden;position:fixed;top:-${this.scrollY}px;left:0;right:0;`;

    setTimeout(() => {
      const focusable = this.dialogEl?.nativeElement.querySelectorAll<HTMLElement>(
        'a[href],button:not([disabled]),input:not([disabled]),[tabindex]:not([tabindex="-1"])'
      );
      focusable?.[0]?.focus();
    });
  }

  private onClose() {
    document.body.style.cssText = '';
    window.scrollTo(0, this.scrollY);
    this.previouslyFocused?.focus();
  }

  onBackdropClick() {
    if (this.closeOnBackdropClick) this.close.emit();
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| `role="dialog"` on the container | Required ARIA role |
| `aria-modal="true"` | Prevents virtual cursor from leaving (NVDA/JAWS) |
| `aria-labelledby` pointing to title `id` | Screen reader announces dialog name on open |
| `aria-describedby` for context text | Optional but helpful for complex dialogs |
| Focus moves to first interactive element on open | User knows where they are immediately |
| Tab / Shift+Tab cycle stays inside | Focus trap prevents leaving the dialog |
| Escape closes the dialog | Standard keyboard contract |
| Backdrop click closes (configurable) | Mouse user convenience |
| Focus returns to trigger element on close | User is back where they were |
| Body scroll locked while open | Prevents disorienting background scroll |
| Rendered via portal to `<body>` | Avoids `overflow:hidden` clipping from ancestors |

---

## Production Pitfalls

**1. iOS Safari scrolls the background through the overlay**
`overflow: hidden` on `body` does not stop iOS from scrolling the page behind the modal. Fix: set `position: fixed; top: -${scrollY}px` on the body when the modal opens, store `scrollY` first, and restore it with `window.scrollTo(0, scrollY)` on close.

**2. `<dialog>` element vs `role="dialog"` div**
The native `<dialog>` element has `.showModal()` which provides a built-in focus trap and backdrop. However, it has inconsistent animation support (the `::backdrop` pseudo-element is hard to animate), and `.close()` / `.returnValue` patterns differ from React state. The custom `role="dialog"` approach in this file gives full control but requires manual focus trap + scroll lock.

**3. Focus trap breaks with dynamically added content**
If the modal body loads content asynchronously (e.g. a form that fetches options), new focusable elements appear after the trap is initialized. Fix: query focusable elements inside the trap handler on each Tab event (not once on open).

**4. `aria-modal` is not universally supported**
Older versions of NVDA + Firefox do not fully respect `aria-modal="true"` and allow virtual cursor to leave the dialog. Fix: complement `aria-modal` by setting `aria-hidden="true"` on all other major sections of the page while the modal is open (the "inert" pattern). The `inert` HTML attribute is now widely supported for this.

**5. Multiple modals using the same `id` for title**
If two modals are mounted simultaneously (e.g. in a wizard), both might have `id="modal-title"`, causing invalid HTML and ARIA pointing to the wrong element. Fix: generate unique IDs per instance with `useId()` (React) or a random suffix.

---

## Interview Angle

**Q: "What is a focus trap, and why does a modal need one?"**

A focus trap is a JavaScript pattern that intercepts Tab and Shift+Tab key events to keep keyboard focus cycling within a defined container. Modals need a focus trap because, without one, a keyboard user pressing Tab can leave the modal and interact with the disabled background content — a confusing and inaccessible experience. The trap queries all focusable elements in the container, then redirects focus from the last element back to the first (and vice versa for Shift+Tab).

**Q: "What is the difference between `display:none` and `aria-hidden='true'`?"**

`display: none` removes the element from both visual rendering and the accessibility tree. `aria-hidden="true"` removes it from the accessibility tree only — it remains visually rendered. For modals, use `aria-hidden="true"` on the backdrop and on background content while keeping the modal visible. Never use `aria-hidden="true"` on focusable elements while they have focus.

**Q: "Why render the modal in a portal?"**

Any ancestor with `overflow: hidden`, `transform`, or `filter` creates a new stacking context and clips or repositions `position: fixed` children. By rendering the modal directly as a child of `<body>` (via portal), you guarantee it is never clipped and always stacks correctly regardless of where in the component tree the trigger lives.
