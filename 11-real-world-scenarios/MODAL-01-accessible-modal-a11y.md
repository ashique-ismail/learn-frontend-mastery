# Accessible Modal Dialog (A11y Deep Dive)

## The Idea

**In plain English:** A modal dialog is a window that appears on top of the page content and forces the user to interact with it before returning to the page. Done correctly, it blocks all interaction with the background — mouse clicks, keyboard navigation, and screen reader virtual cursor movement — until the user explicitly closes it. Done incorrectly, keyboard users fall through to background content, screen reader users get lost outside the dialog, and the page shifts 15px when a scrollbar disappears.

**Real-world analogy:** Think of a modal like a security door at a bank vault.

- The **vault room** is the modal itself — once the door opens, you are locked inside until you finish your business and press the exit button.
- The **rest of the bank** is the background page — still visible through the glass, but you cannot touch anything in it while inside the vault.
- The **security guard at the door** is the focus trap — every time you try to walk out without permission (Tab past the last button), they redirect you back to the first control inside.
- The **"Vault Occupied" sign on the outside door** is `aria-modal="true"` — it tells screen reader users not to wander in through the wall (virtual cursor navigation).
- **Returning the guard's keycard when you leave** is returning focus to the trigger button — the system restores exactly where you were before you entered.

The key insight: visual containment (the overlay) is not the same as functional containment. You must enforce containment at three independent layers — keyboard focus, screen reader cursor, and pointer events.

---

## Learning Objectives

- Implement a robust focus trap using Tab/Shift+Tab interception with correct focusable element detection
- Understand why `aria-modal` alone is not sufficient and which screen readers ignore it
- Apply scroll lock without causing layout shift by measuring and compensating scrollbar width
- Use the `inert` attribute as a modern complement (not replacement) for aria-hidden on background content
- Return focus precisely to the trigger element that opened the modal
- Handle edge cases: nested modals, dynamically added focusable elements, modals opened programmatically

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Visual overlay covering the background | ✅ | — |
| Centering the dialog on screen | ✅ | — |
| Entrance/exit animation | ✅ | — |
| Prevent keyboard Tab from leaving the dialog | ❌ | CSS cannot intercept or redirect keyboard events |
| Prevent screen reader virtual cursor from leaving the dialog | ❌ | Requires `aria-modal`, `aria-hidden`, or `inert` toggled by JS |
| Remove background content from tab order | ❌ | CSS `pointer-events: none` does not affect focus, only pointer |
| Lock body scroll without layout shift | ❌ | Scrollbar compensation requires measuring `window.innerWidth - document.documentElement.clientWidth` in JS |
| Return focus to trigger on close | ❌ | Requires JS to store a reference to `document.activeElement` before opening |
| Close on Escape key | ❌ | Requires a `keydown` event listener |
| Announce "dialog" role and label to screen readers on open | ❌ | Requires `role="dialog"` + `aria-labelledby` and programmatic focus move into the dialog |

**Conclusion:** CSS owns the visual layer — overlay, centering, animation, sizing. JS owns the functional containment layer — focus trap, scroll lock, ARIA state, and focus restoration.

---

## HTML & CSS Foundation

### The Structure

```html
<!-- The trigger button (must be a real button) -->
<button
  class="modal-trigger"
  aria-haspopup="dialog"
  data-modal-target="confirm-dialog"
>
  Delete account
</button>

<!--
  The backdrop: separates visual overlay from the dialog element.
  aria-hidden keeps it invisible to screen readers.
-->
<div class="modal-backdrop" aria-hidden="true" hidden></div>

<!--
  The dialog element.
  role="dialog" — native <dialog> element also works (see React section).
  aria-modal="true" — hints to screen readers to ignore background content.
  aria-labelledby — points to the visible heading inside.
  aria-describedby — points to the body text (optional but recommended).
  tabindex="-1" — makes the dialog itself programmatically focusable so we can
                  move focus into it on open; removes it from natural tab order.
-->
<div
  id="confirm-dialog"
  role="dialog"
  aria-modal="true"
  aria-labelledby="dialog-title"
  aria-describedby="dialog-desc"
  tabindex="-1"
  class="modal-dialog"
  hidden
>
  <div class="modal-dialog__inner">
    <h2 id="dialog-title" class="modal-dialog__title">Delete account</h2>
    <p id="dialog-desc" class="modal-dialog__body">
      This action cannot be undone. All your data will be permanently removed.
    </p>
    <div class="modal-dialog__footer">
      <button class="btn btn--danger" data-modal-confirm>Delete</button>
      <button class="btn btn--ghost" data-modal-cancel>Cancel</button>
    </div>
    <!-- Close button in top-right corner -->
    <button
      class="modal-dialog__close"
      aria-label="Close dialog"
      data-modal-cancel
    >
      <span aria-hidden="true">&times;</span>
    </button>
  </div>
</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --modal-overlay-bg:  rgba(0, 0, 0, 0.6);
  --modal-bg:          #ffffff;
  --modal-radius:      8px;
  --modal-shadow:      0 20px 60px rgba(0, 0, 0, 0.3);
  --modal-transition:  0.2s cubic-bezier(0.4, 0, 0.2, 1);
  --modal-max-width:   480px;
  --modal-padding:     1.5rem;
}

/* ─── Backdrop ─── */
.modal-backdrop {
  position: fixed;
  inset: 0;
  background: var(--modal-overlay-bg);
  z-index: 900;
  opacity: 0;
  transition: opacity var(--modal-transition);
}

.modal-backdrop.is-visible {
  opacity: 1;
}

/* ─── Dialog container: centers the dialog ─── */
/*
  We position the dialog independently from the backdrop so that
  clicks on the backdrop can close the modal without the dialog
  itself capturing the click.
*/
.modal-dialog {
  position: fixed;
  inset: 0;
  z-index: 901;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 1rem;           /* breathing room on small screens */
}

/* ─── The actual box ─── */
.modal-dialog__inner {
  position: relative;
  background: var(--modal-bg);
  border-radius: var(--modal-radius);
  box-shadow: var(--modal-shadow);
  width: 100%;
  max-width: var(--modal-max-width);
  padding: var(--modal-padding);
  transform: translateY(12px);
  opacity: 0;
  transition:
    transform var(--modal-transition),
    opacity   var(--modal-transition);
}

.modal-dialog.is-open .modal-dialog__inner {
  transform: translateY(0);
  opacity: 1;
}

/* ─── Inner layout ─── */
.modal-dialog__title {
  margin: 0 0 0.75rem;
  font-size: 1.25rem;
  font-weight: 600;
  padding-right: 2rem;   /* space for close button */
}

.modal-dialog__body {
  margin: 0 0 1.5rem;
  color: #555;
  line-height: 1.6;
}

.modal-dialog__footer {
  display: flex;
  gap: 0.75rem;
  justify-content: flex-end;
  flex-wrap: wrap;
}

/* ─── Close button ─── */
.modal-dialog__close {
  position: absolute;
  top: 1rem;
  right: 1rem;
  width: 36px;
  height: 36px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: none;
  border: none;
  border-radius: 50%;
  font-size: 1.25rem;
  cursor: pointer;
  color: #666;
  transition: background 0.15s;
}
.modal-dialog__close:hover { background: #f0f0f0; }

/* ─── Buttons ─── */
.btn {
  padding: 0.6rem 1.25rem;
  border-radius: 6px;
  font-size: 1rem;
  cursor: pointer;
  border: 2px solid transparent;
  min-height: 44px;        /* accessible tap target */
}
.btn--danger { background: #c0392b; color: #fff; border-color: #c0392b; }
.btn--ghost  { background: transparent; color: #333; border-color: #ccc; }

/* ─── Scroll lock: applied by JS to <html> ─── */
/*
  We apply to <html> instead of <body> so that the scrollbar
  width compensation on <body> via padding-right is not overridden
  by anything that sets body margin/padding.
*/
html.modal-open {
  overflow: hidden;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .modal-backdrop,
  .modal-dialog__inner {
    transition: none;
  }
}
```

**What CSS owns:** overlay opacity, dialog centering, entrance animation, close button positioning, scroll lock rule.

**What CSS cannot own:** when to add/remove classes, scrollbar width compensation, focus trap, ARIA toggling, `inert` on background, focus restoration.

---

## React Implementation

### The `useModal` Hook — All Accessibility Logic in One Place

```tsx
// useModal.ts
import { useCallback, useEffect, useRef, useState } from 'react';

// All focusable element selectors per ARIA spec
const FOCUSABLE =
  'a[href], button:not([disabled]), input:not([disabled]), ' +
  'select:not([disabled]), textarea:not([disabled]), ' +
  '[tabindex]:not([tabindex="-1"]), details > summary';

interface UseModalOptions {
  onClose?: () => void;
}

export function useModal({ onClose }: UseModalOptions = {}) {
  const [isOpen, setIsOpen] = useState(false);

  // Store the element that triggered the modal
  const triggerRef = useRef<HTMLElement | null>(null);
  // Ref to the dialog element itself
  const dialogRef  = useRef<HTMLDivElement>(null);

  // ── Scrollbar width compensation ──────────────────────────────────────────
  // Without this, hiding body overflow removes the scrollbar (11-17px on
  // most OSes) and the page shifts left, causing a jarring layout shift.
  const lockScroll = useCallback(() => {
    const scrollbarWidth =
      window.innerWidth - document.documentElement.clientWidth;

    document.documentElement.classList.add('modal-open');

    // Compensate for the removed scrollbar so nothing shifts
    if (scrollbarWidth > 0) {
      document.body.style.paddingRight = `${scrollbarWidth}px`;
      // Also compensate any fixed headers/toolbars
      document
        .querySelectorAll<HTMLElement>('[data-fixed-header]')
        .forEach(el => {
          el.style.paddingRight = `${scrollbarWidth}px`;
        });
    }
  }, []);

  const unlockScroll = useCallback(() => {
    document.documentElement.classList.remove('modal-open');
    document.body.style.paddingRight = '';
    document
      .querySelectorAll<HTMLElement>('[data-fixed-header]')
      .forEach(el => {
        el.style.paddingRight = '';
      });
  }, []);

  // ── inert on background ───────────────────────────────────────────────────
  // aria-modal alone is ignored by NVDA+Firefox and JAWS in browse mode.
  // Setting inert on background elements is the only fully reliable method.
  const applyInert = useCallback((dialogEl: HTMLElement) => {
    Array.from(document.body.children).forEach(child => {
      if (!dialogEl.contains(child) && child !== dialogEl) {
        (child as HTMLElement).setAttribute('inert', '');
        (child as HTMLElement).setAttribute('aria-hidden', 'true');
      }
    });
  }, []);

  const removeInert = useCallback(() => {
    document.querySelectorAll('[inert]').forEach(el => {
      el.removeAttribute('inert');
      el.removeAttribute('aria-hidden');
    });
  }, []);

  // ── Open / Close ──────────────────────────────────────────────────────────
  const open = useCallback(() => {
    // Capture the currently focused element BEFORE opening
    triggerRef.current = document.activeElement as HTMLElement;
    setIsOpen(true);
  }, []);

  const close = useCallback(() => {
    setIsOpen(false);
    onClose?.();
  }, [onClose]);

  // ── Effects: run after state change ──────────────────────────────────────
  useEffect(() => {
    const dialog = dialogRef.current;
    if (!dialog) return;

    if (isOpen) {
      lockScroll();
      applyInert(dialog);

      // Move focus INTO the dialog. We focus the dialog element itself
      // (tabindex="-1") so the screen reader announces the role and label.
      // Then natural tab order starts from the first focusable child.
      requestAnimationFrame(() => dialog.focus());
    } else {
      unlockScroll();
      removeInert();
      // Return focus to the element that triggered the modal
      triggerRef.current?.focus();
      triggerRef.current = null;
    }

    return () => {
      // Safety cleanup if component unmounts while open
      unlockScroll();
      removeInert();
    };
  }, [isOpen, lockScroll, unlockScroll, applyInert, removeInert]);

  // ── Focus Trap ────────────────────────────────────────────────────────────
  useEffect(() => {
    if (!isOpen || !dialogRef.current) return;

    const trap = (e: KeyboardEvent) => {
      if (e.key === 'Escape') {
        e.stopPropagation();
        close();
        return;
      }

      if (e.key !== 'Tab') return;

      const focusable = Array.from(
        dialogRef.current!.querySelectorAll<HTMLElement>(FOCUSABLE)
      ).filter(el => !el.closest('[inert]'));   // skip inert subtrees

      if (focusable.length === 0) {
        e.preventDefault();
        return;
      }

      const first = focusable[0];
      const last  = focusable[focusable.length - 1];

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
    };

    // Capture phase so we intercept before any component handler
    document.addEventListener('keydown', trap, true);
    return () => document.removeEventListener('keydown', trap, true);
  }, [isOpen, close]);

  return { isOpen, open, close, dialogRef };
}
```

### The Modal Component

```tsx
// Modal.tsx
import { useEffect } from 'react';
import { createPortal } from 'react-dom';

interface ModalProps {
  isOpen:     boolean;
  onClose:    () => void;
  title:      string;
  titleId:    string;
  descId?:    string;
  children:   React.ReactNode;
}

export function Modal({
  isOpen, onClose, title, titleId, descId, children
}: ModalProps) {
  if (!isOpen) return null;

  // Render into document.body via portal to avoid stacking context issues
  return createPortal(
    <>
      <div
        className={`modal-backdrop ${isOpen ? 'is-visible' : ''}`}
        aria-hidden="true"
        onClick={onClose}
      />
      <div
        role="dialog"
        aria-modal="true"
        aria-labelledby={titleId}
        aria-describedby={descId}
        tabIndex={-1}
        className={`modal-dialog ${isOpen ? 'is-open' : ''}`}
        // Stop click bubbling so clicking inside doesn't close the modal
        onClick={e => e.stopPropagation()}
      >
        <div className="modal-dialog__inner">
          <h2 id={titleId} className="modal-dialog__title">{title}</h2>
          {children}
          <button
            className="modal-dialog__close"
            aria-label="Close dialog"
            onClick={onClose}
          >
            <span aria-hidden="true">&times;</span>
          </button>
        </div>
      </div>
    </>,
    document.body
  );
}
```

### Wiring It Together

```tsx
// DeleteAccountButton.tsx
import { useModal } from './useModal';
import { Modal }    from './Modal';

export function DeleteAccountButton() {
  const { isOpen, open, close, dialogRef } = useModal();

  return (
    <>
      <button
        className="modal-trigger"
        aria-haspopup="dialog"
        onClick={open}
      >
        Delete account
      </button>

      <Modal
        isOpen={isOpen}
        onClose={close}
        title="Delete account"
        titleId="delete-dialog-title"
        descId="delete-dialog-desc"
      >
        {/*
          We attach dialogRef here instead of on the outer Modal wrapper
          so the focus trap and inert logic target the correct element.
          Pass ref via forwardRef if you keep Modal in a separate file.
        */}
        <p id="delete-dialog-desc" className="modal-dialog__body">
          This action cannot be undone. All your data will be permanently removed.
        </p>
        <div className="modal-dialog__footer">
          <button className="btn btn--danger" onClick={() => { /* confirm */ close(); }}>
            Delete
          </button>
          <button className="btn btn--ghost" onClick={close}>
            Cancel
          </button>
        </div>
      </Modal>
    </>
  );
}
```

---

## Angular Implementation

### The Modal Service

```typescript
// modal.service.ts
import { Injectable, signal } from '@angular/core';

const FOCUSABLE =
  'a[href], button:not([disabled]), input:not([disabled]), ' +
  'select:not([disabled]), textarea:not([disabled]), ' +
  '[tabindex]:not([tabindex="-1"])';

@Injectable({ providedIn: 'root' })
export class ModalService {
  private _isOpen  = signal(false);
  private _trigger = signal<HTMLElement | null>(null);

  readonly isOpen = this._isOpen.asReadonly();

  open(triggerEl: HTMLElement) {
    this._trigger.set(triggerEl);
    this._isOpen.set(true);
    this.lockScroll();
  }

  close() {
    this._isOpen.set(false);
    this.unlockScroll();
    // Return focus after the next microtask so the DOM has updated
    const trigger = this._trigger();
    Promise.resolve().then(() => trigger?.focus());
    this._trigger.set(null);
  }

  applyInert(dialogEl: HTMLElement) {
    Array.from(document.body.children).forEach(child => {
      if (child !== dialogEl && !dialogEl.contains(child)) {
        (child as HTMLElement).setAttribute('inert', '');
        (child as HTMLElement).setAttribute('aria-hidden', 'true');
      }
    });
  }

  removeInert() {
    document.querySelectorAll('[inert]').forEach(el => {
      el.removeAttribute('inert');
      el.removeAttribute('aria-hidden');
    });
  }

  private lockScroll() {
    const w = window.innerWidth - document.documentElement.clientWidth;
    document.documentElement.classList.add('modal-open');
    if (w > 0) document.body.style.paddingRight = `${w}px`;
  }

  private unlockScroll() {
    document.documentElement.classList.remove('modal-open');
    document.body.style.paddingRight = '';
  }
}
```

### The Modal Component

```typescript
// modal.component.ts
import {
  Component, Input, Output, EventEmitter,
  ElementRef, OnChanges, SimpleChanges,
  HostListener, inject, AfterViewInit
} from '@angular/core';
import { NgIf } from '@angular/common';
import { ModalService } from './modal.service';

const FOCUSABLE =
  'a[href], button:not([disabled]), input:not([disabled]), ' +
  'select:not([disabled]), textarea:not([disabled]), ' +
  '[tabindex]:not([tabindex="-1"])';

@Component({
  selector: 'app-modal',
  standalone: true,
  imports: [NgIf],
  template: `
    <ng-container *ngIf="isOpen">
      <!-- Backdrop -->
      <div
        class="modal-backdrop is-visible"
        aria-hidden="true"
        (click)="handleClose()"
      ></div>

      <!-- Dialog -->
      <div
        #dialogEl
        role="dialog"
        aria-modal="true"
        [attr.aria-labelledby]="titleId"
        [attr.aria-describedby]="descId || null"
        tabindex="-1"
        class="modal-dialog is-open"
        (click)="$event.stopPropagation()"
      >
        <div class="modal-dialog__inner">
          <h2 [id]="titleId" class="modal-dialog__title">{{ title }}</h2>
          <ng-content></ng-content>
          <button
            class="modal-dialog__close"
            aria-label="Close dialog"
            (click)="handleClose()"
          >
            <span aria-hidden="true">&times;</span>
          </button>
        </div>
      </div>
    </ng-container>
  `,
})
export class ModalComponent implements OnChanges, AfterViewInit {
  @Input()  isOpen  = false;
  @Input()  title   = '';
  @Input()  titleId = 'modal-title';
  @Input()  descId?: string;
  @Output() closed  = new EventEmitter<void>();

  private el      = inject(ElementRef);
  private service = inject(ModalService);
  private dialogEl: HTMLElement | null = null;

  ngAfterViewInit() {
    // dialogEl is only in the DOM when isOpen; we re-query on changes
  }

  ngOnChanges(changes: SimpleChanges) {
    if (!changes['isOpen']) return;

    if (this.isOpen) {
      // Wait one tick for NgIf to render the dialog
      Promise.resolve().then(() => {
        this.dialogEl = this.el.nativeElement.querySelector('[role="dialog"]');
        if (this.dialogEl) {
          this.service.applyInert(this.dialogEl);
          this.dialogEl.focus();
        }
      });
    } else {
      this.service.removeInert();
    }
  }

  handleClose() {
    this.closed.emit();
  }

  @HostListener('document:keydown', ['$event'])
  onKeydown(e: KeyboardEvent) {
    if (!this.isOpen) return;

    if (e.key === 'Escape') {
      e.stopPropagation();
      this.handleClose();
      return;
    }

    if (e.key !== 'Tab' || !this.dialogEl) return;

    const focusable = Array.from(
      this.dialogEl.querySelectorAll<HTMLElement>(FOCUSABLE)
    );

    if (!focusable.length) { e.preventDefault(); return; }

    const first = focusable[0];
    const last  = focusable[focusable.length - 1];

    if (e.shiftKey && document.activeElement === first) {
      e.preventDefault(); last.focus();
    } else if (!e.shiftKey && document.activeElement === last) {
      e.preventDefault(); first.focus();
    }
  }
}
```

### Usage in a Parent Component

```typescript
// confirm-delete.component.ts
import { Component, inject } from '@angular/core';
import { NgIf } from '@angular/common';
import { ModalComponent } from './modal.component';
import { ModalService }   from './modal.service';

@Component({
  selector: 'app-confirm-delete',
  standalone: true,
  imports: [NgIf, ModalComponent],
  template: `
    <button
      #triggerBtn
      aria-haspopup="dialog"
      (click)="openModal(triggerBtn)"
    >
      Delete account
    </button>

    <app-modal
      [isOpen]="isOpen"
      title="Delete account"
      titleId="delete-title"
      descId="delete-desc"
      (closed)="closeModal()"
    >
      <p id="delete-desc" class="modal-dialog__body">
        This action cannot be undone.
      </p>
      <div class="modal-dialog__footer">
        <button class="btn btn--danger" (click)="confirm()">Delete</button>
        <button class="btn btn--ghost"  (click)="closeModal()">Cancel</button>
      </div>
    </app-modal>
  `,
})
export class ConfirmDeleteComponent {
  isOpen = false;
  private service = inject(ModalService);

  openModal(trigger: HTMLElement) {
    this.isOpen = true;
    this.service.open(trigger);
  }

  closeModal() {
    this.isOpen = false;
    this.service.close();
  }

  confirm() {
    // perform delete...
    this.closeModal();
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Dialog has `role="dialog"` or uses `<dialog>` element | `role="dialog"` on the container div; `<dialog>` is equivalent and adds built-in close-on-backdrop-click in some browsers |
| Dialog has accessible name | `aria-labelledby` pointing to the visible `<h2>` inside |
| Dialog has accessible description | `aria-describedby` pointing to the body paragraph (optional but improves context for screen reader users) |
| `aria-modal="true"` declared | Informs screen readers supporting it (VoiceOver, Narrator) to restrict virtual cursor to the dialog |
| Background content receives `inert` attribute | Prevents NVDA and JAWS in browse mode from reading background content, which `aria-modal` alone fails to block |
| Background content receives `aria-hidden="true"` | Belt-and-suspenders with `inert`; ensures older screen readers also ignore background |
| Focus moves into dialog on open | Dialog element (tabindex="-1") receives `focus()` via `requestAnimationFrame` / `Promise.resolve()` after render |
| Focus is trapped with Tab and Shift+Tab | `keydown` listener in capture phase intercepts Tab at document level; wraps from last to first and first to last |
| Escape key closes the dialog | Same `keydown` listener checks for `Escape` |
| Focus returns to trigger on close | Trigger element stored in a ref before opening; `.focus()` called after close |
| Scroll lock applied without layout shift | `overflow: hidden` on `<html>` + `padding-right` equal to scrollbar width on `<body>` |
| Fixed headers compensated | Elements with `[data-fixed-header]` also receive matching `padding-right` |
| `prefers-reduced-motion` respected | CSS media query disables all modal transitions |
| Minimum tap target sizes enforced | 44px minimum height on all interactive elements inside the dialog |

---

## Production Pitfalls

**1. `aria-modal` is not enough on its own**
NVDA in browse mode and JAWS in virtual cursor mode ignore `aria-modal="true"` and will let the user's virtual cursor roam freely into the background content. The only reliable solution is setting the `inert` attribute on every sibling of the dialog at the `body` level. `inert` removes the element from the accessibility tree, prevents focus, and disables pointer events — all in one attribute. Polyfill with `wicg-inert` for older browsers, but as of 2024 all major browsers support it natively.

**2. Scrollbar layout shift**
When you set `overflow: hidden` on `html` or `body`, the scrollbar (typically 11-17px wide on Windows, 0px on macOS overlay scrollbars) disappears, and everything shifts left. Measure the gap before locking: `window.innerWidth - document.documentElement.clientWidth`. Apply that value as `padding-right` to the body and to any `position: fixed` elements (headers, toasts, floating buttons). Remove it on unlock. On macOS with overlay scrollbars the gap is 0 so no compensation is needed — the measurement handles this automatically.

**3. Focus trap breaks when dynamic content is added inside the modal**
If the modal contains a form where fields appear and disappear based on user input, the cached `focusable` array in a one-time setup becomes stale. Fix: query focusable elements fresh on every `keydown` event inside the trap handler, not once at setup time. This is slightly less efficient but ensures the trap always reflects the current DOM.

**4. Mounting the modal inside the component tree causes stacking context issues**
If the modal's parent has `transform`, `filter`, `will-change`, or `isolation: isolate` applied, `position: fixed` inside it no longer fixes to the viewport — it fixes to the nearest stacking context ancestor. Fix: always render the modal into `document.body` via a React portal (`createPortal`) or Angular CDK `Overlay`. This also makes it easier to apply `inert` to siblings, since the dialog is a direct child of `body`.

**5. Returning focus to the wrong element when the trigger was destroyed**
If the button that opened the modal is conditionally rendered and removed from the DOM while the modal is open (e.g., a list row that gets deleted), the stored `triggerRef` points to a detached node. Calling `.focus()` on it silently fails. Fix: after calling `.focus()`, check that `document.activeElement` changed. If it did not, fall back to `document.body` or the nearest logical container like the main content landmark.

**6. Nested modals double-apply inert**
If a second modal opens while the first is open, calling `applyInert` again will mark the first modal as inert since it is now a sibling of the second. Fix: maintain a stack of open modals. Only the topmost modal should apply `inert`; opening a second modal should add `inert` to the first modal (so it is properly blocked) but track that state so closing the second modal restores the first modal to active without clearing inert on unrelated background elements.

**7. iOS Safari momentum scroll bleeds through the overlay**
Setting `overflow: hidden` on `<html>` still allows `-webkit-overflow-scrolling: touch` momentum scrolling on `<body>` in older iOS versions. Fix: in addition to `overflow: hidden`, set `position: fixed` and `width: 100%` on `<body>`. Store `window.scrollY` before applying and restore with `window.scrollTo(0, savedY)` on unlock to prevent the page jumping to the top.

---

## Interview Angle

**Q: "Walk me through how you would implement an accessible modal dialog from scratch."**

A complete answer covers four distinct layers. First, the ARIA layer: `role="dialog"`, `aria-modal="true"`, `aria-labelledby` pointing to the visible heading, and `tabindex="-1"` on the dialog so focus can be programmatically moved into it on open. Second, the focus layer: store `document.activeElement` as the trigger reference before opening, move focus into the dialog with `requestAnimationFrame(() => dialog.focus())`, implement a Tab/Shift+Tab trap by querying all focusable descendants fresh on each keydown, and return focus to the trigger on close. Third, the background-isolation layer: `aria-modal` alone does not stop NVDA or JAWS virtual cursor navigation — you must also apply the `inert` attribute to every sibling of the dialog at the body level. Fourth, the scroll layer: measure scrollbar width before locking (`window.innerWidth - document.documentElement.clientWidth`), set `overflow: hidden` on `<html>`, compensate with `padding-right` on body and any fixed headers, then fully reverse all of this on close.

**Follow-up: "Why is `aria-modal` not sufficient and what exactly does `inert` do differently?"**

`aria-modal="true"` is a semantic hint that well-behaved screen readers use to restrict their virtual reading cursor to the dialog, but it is not enforced at the DOM level. NVDA in browse mode and JAWS in virtual cursor mode read it but do not always respect it — especially when the user uses arrow keys rather than Tab to navigate. The `inert` attribute, on the other hand, is enforced by the browser itself: it removes the element from the accessibility tree (equivalent to `aria-hidden`), removes all its descendants from the tab order (equivalent to setting `tabindex="-1"` on everything inside), and disables all pointer events. Because it works at the browser platform level rather than the screen reader heuristic level, it is reliable across all assistive technology combinations. The correct implementation uses both: `aria-modal` for screen readers that respect it, and `inert` on siblings for those that do not.
