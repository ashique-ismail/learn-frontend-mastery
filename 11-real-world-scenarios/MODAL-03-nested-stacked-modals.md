# Nested / Stacked Modals

## The Idea

**In plain English:** Sometimes one modal needs to spawn another — a form opens a confirmation dialog, or a settings panel opens a date picker. The challenge is not visual stacking but behavioural stacking: each new modal must trap focus, own the Escape key, sit above all previous modals, and when it closes the focus must return not to the page but to the specific element that opened that modal — even if several modals are already open underneath.

**Real-world analogy:** Think of a stack of filing trays on a desk.

- Each **tray** is a modal. You can only work with the tray on **top**.
- When you place a new tray on the stack, you put down whatever you were holding and give your full attention to the new tray. The old trays still exist — they are just temporarily inaccessible.
- When you are done with the top tray you **remove it**, pick up whatever you were holding before you placed it (your focus), and continue working on the tray below.
- The **desk surface** = the main page. You cannot reach it until all trays are cleared.
- The **semi-transparent sheet under each tray** = the per-modal backdrop. Each tray brings its own sheet so the visual depth cue stacks too.
- **The rule about only working on the top tray** = the focus trap. You cannot Tab into a modal that is buried under another one.

The key insight: stacked modals are a *runtime call stack*. Each modal pushes onto the stack with its own context (the element that triggered it) and pops off cleanly, restoring exactly the state that existed before it appeared.

---

## Learning Objectives

- Understand why a single global modal manager breaks when modals nest
- Implement a focus stack (array of previously focused elements) so focus restores correctly through multiple levels
- Manage z-index with a counter so each modal is always on top of its predecessor
- Layer backdrops independently — each modal owns its own semi-transparent overlay
- Enforce the rule that Escape only dismisses the topmost modal
- Avoid common pitfalls: z-index overflow, scroll lock compounding, ARIA hidden leaking to active modals

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Stack multiple modals visually | ⚠️ static `z-index` values only | CSS cannot dynamically assign z-index based on open order |
| Know which modal is "topmost" | ❌ | CSS has no concept of a stack or array |
| Trap focus inside the topmost modal only | ❌ | Requires JS `keydown` listeners and DOM queries |
| Restore focus to the correct opener on close | ❌ | Requires storing a reference to `document.activeElement` before opening |
| Bind Escape to only the topmost modal | ❌ | CSS has no `keydown` handling |
| Prevent interaction with modals below | ❌ | `inert` attribute or `aria-hidden` must be toggled by JS |
| Lock body scroll exactly once (not once per modal) | ❌ | Requires a reference-counted lock in JS |
| Animate each modal in/out independently | ⚠️ | CSS animates, but JS must add/remove the trigger class per modal |

**Conclusion:** CSS handles the visual shape, backdrop colour, and entrance animation of each modal. JS owns the stack, the focus lifecycle, z-index assignment, Escape routing, and the `inert`/`aria-hidden` state of buried modals.

---

## HTML & CSS Foundation

### The Structure

```html
<!--
  Each modal is rendered independently.
  data-modal-id is used by JS to identify the modal in the stack.
  The backdrop is a sibling of the modal panel, not a parent,
  so each modal + backdrop pair can be managed independently.
-->

<!-- Modal level 1 -->
<div class="modal-backdrop" data-modal-backdrop="modal-1" aria-hidden="true"></div>
<div
  class="modal"
  id="modal-1"
  role="dialog"
  aria-modal="true"
  aria-labelledby="modal-1-title"
  aria-describedby="modal-1-desc"
>
  <div class="modal__inner">
    <h2 id="modal-1-title" class="modal__title">Edit Settings</h2>
    <p id="modal-1-desc" class="modal__desc">Adjust your account preferences below.</p>

    <div class="modal__body">
      <!-- content -->
      <button class="btn" data-opens="modal-2">Choose date…</button>
    </div>

    <div class="modal__footer">
      <button class="btn btn--ghost" data-modal-close="modal-1">Cancel</button>
      <button class="btn btn--primary">Save</button>
    </div>
  </div>
</div>

<!-- Modal level 2 — spawned by modal 1 -->
<div class="modal-backdrop" data-modal-backdrop="modal-2" aria-hidden="true"></div>
<div
  class="modal"
  id="modal-2"
  role="dialog"
  aria-modal="true"
  aria-labelledby="modal-2-title"
>
  <div class="modal__inner">
    <h2 id="modal-2-title" class="modal__title">Pick a Date</h2>
    <!-- date picker content -->
    <button class="btn btn--ghost" data-modal-close="modal-2">Close</button>
  </div>
</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --modal-base-z:       1000;   /* z-index of the first backdrop */
  --modal-z-step:       10;     /* each new modal pair gets +10 */
  --modal-backdrop-bg:  rgba(0, 0, 0, 0.45);
  --modal-bg:           #ffffff;
  --modal-radius:       8px;
  --modal-shadow:       0 24px 64px rgba(0, 0, 0, 0.24);
  --modal-transition:   0.22s cubic-bezier(0.4, 0, 0.2, 1);
  --modal-max-width:    560px;
}

/* ─── Backdrop ─── */
/*
  Each backdrop has its z-index set inline by JS:
  style="z-index: calc(var(--modal-base-z) + (index * var(--modal-z-step)))"
  The CSS here only handles the visual + animation.
*/
.modal-backdrop {
  position: fixed;
  inset: 0;
  background: var(--modal-backdrop-bg);
  opacity: 0;
  visibility: hidden;
  transition: opacity var(--modal-transition), visibility var(--modal-transition);
  /* z-index set by JS */
}

.modal-backdrop.is-visible {
  opacity: 1;
  visibility: visible;
}

/* Successive backdrops stack, but the page dims only once visually.
   Each backdrop adds a little more dimming, which is usually desirable
   (deeper stack = darker screen). Cap it with a max-opacity override if needed. */

/* ─── Modal panel ─── */
.modal {
  position: fixed;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  pointer-events: none;   /* backdrop handles clicks; inner panel re-enables */
  opacity: 0;
  visibility: hidden;
  transition: opacity var(--modal-transition), visibility var(--modal-transition);
  /* z-index set by JS — always one above its paired backdrop */
}

.modal.is-open {
  opacity: 1;
  visibility: visible;
  pointer-events: auto;
}

/* Entrance animation: scale up from 96% */
.modal__inner {
  background: var(--modal-bg);
  border-radius: var(--modal-radius);
  box-shadow: var(--modal-shadow);
  width: 100%;
  max-width: var(--modal-max-width);
  max-height: 90dvh;
  overflow-y: auto;
  padding: 2rem;
  transform: scale(0.96) translateY(8px);
  transition: transform var(--modal-transition);
}

.modal.is-open .modal__inner {
  transform: scale(1) translateY(0);
}

/* ─── Modal structure ─── */
.modal__title {
  font-size: 1.25rem;
  font-weight: 600;
  margin: 0 0 0.5rem;
}

.modal__desc {
  color: #666;
  margin: 0 0 1.5rem;
}

.modal__footer {
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
  margin-top: 1.5rem;
  padding-top: 1rem;
  border-top: 1px solid #eee;
}

/* ─── Buttons ─── */
.btn {
  padding: 0.5rem 1.25rem;
  border-radius: 4px;
  border: 1px solid transparent;
  font-size: 0.9375rem;
  cursor: pointer;
  min-height: 40px;
}
.btn--primary { background: #0066cc; color: #fff; border-color: #0066cc; }
.btn--ghost   { background: transparent; color: #333; border-color: #ccc; }

/* ─── Body scroll lock (applied once, reference-counted by JS) ─── */
body.modal-open {
  overflow: hidden;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .modal-backdrop,
  .modal,
  .modal__inner {
    transition: none;
  }
  .modal__inner {
    transform: none;
  }
}
```

**What CSS owns:** each modal's visual shape, the backdrop fade, the entrance scale animation, scroll lock class, reduced-motion fallback.

**What CSS cannot own:** which modal is on top, z-index values at runtime, focus trapping, Escape routing, restoring focus through the stack.

---

## React Implementation

### The Modal Stack Context

```tsx
// modal-stack.context.tsx
import {
  createContext, useContext, useCallback,
  useRef, useState, useEffect, ReactNode
} from 'react';

interface ModalEntry {
  id: string;
  opener: HTMLElement | null;   // element that had focus when this modal opened
}

interface ModalStackContextValue {
  push: (id: string) => void;
  pop: () => void;
  isTop: (id: string) => boolean;
  zIndexFor: (id: string) => { backdrop: number; modal: number };
  stackSize: number;
}

const BASE_Z    = 1000;
const Z_STEP    = 10;

const ModalStackContext = createContext<ModalStackContextValue | null>(null);

export function ModalStackProvider({ children }: { children: ReactNode }) {
  const [stack, setStack] = useState<ModalEntry[]>([]);
  const lockCount = useRef(0);

  const push = useCallback((id: string) => {
    const opener = document.activeElement as HTMLElement | null;
    setStack(prev => [...prev, { id, opener }]);

    // Reference-counted scroll lock
    lockCount.current++;
    if (lockCount.current === 1) {
      document.body.classList.add('modal-open');
    }
  }, []);

  const pop = useCallback(() => {
    setStack(prev => {
      const entry = prev[prev.length - 1];
      // Restore focus to whatever opened this modal
      if (entry?.opener && typeof entry.opener.focus === 'function') {
        // Defer so the closing animation doesn't fight focus
        setTimeout(() => entry.opener!.focus(), 0);
      }
      return prev.slice(0, -1);
    });

    lockCount.current = Math.max(0, lockCount.current - 1);
    if (lockCount.current === 0) {
      document.body.classList.remove('modal-open');
    }
  }, []);

  const isTop = useCallback(
    (id: string) => {
      return stack.length > 0 && stack[stack.length - 1].id === id;
    },
    [stack]
  );

  const zIndexFor = useCallback(
    (id: string) => {
      const idx = stack.findIndex(e => e.id === id);
      const base = BASE_Z + idx * Z_STEP;
      return { backdrop: base, modal: base + 1 };
    },
    [stack]
  );

  return (
    <ModalStackContext.Provider
      value={{ push, pop, isTop, zIndexFor, stackSize: stack.length }}
    >
      {children}
    </ModalStackContext.Provider>
  );
}

export function useModalStack() {
  const ctx = useContext(ModalStackContext);
  if (!ctx) throw new Error('useModalStack must be used inside ModalStackProvider');
  return ctx;
}
```

### The useModal Hook

```tsx
// use-modal.ts
import { useState, useCallback } from 'react';
import { useModalStack } from './modal-stack.context';

export function useModal(id: string) {
  const [isOpen, setIsOpen] = useState(false);
  const { push, pop, isTop, zIndexFor } = useModalStack();

  const open = useCallback(() => {
    push(id);
    setIsOpen(true);
  }, [id, push]);

  const close = useCallback(() => {
    pop();
    setIsOpen(false);
  }, [pop]);

  return {
    isOpen,
    open,
    close,
    isTop: isTop(id),
    zIndex: zIndexFor(id),
  };
}
```

### The Modal Component

```tsx
// Modal.tsx
import {
  useRef, useEffect, ReactNode, KeyboardEvent as ReactKeyboardEvent
} from 'react';
import { useModalStack } from './modal-stack.context';

interface ModalProps {
  id: string;
  isOpen: boolean;
  onClose: () => void;
  title: string;
  description?: string;
  children: ReactNode;
  footer?: ReactNode;
}

export function Modal({
  id, isOpen, onClose, title, description, children, footer
}: ModalProps) {
  const { isTop, zIndexFor } = useModalStack();
  const { backdrop: backdropZ, modal: modalZ } = zIndexFor(id);
  const dialogRef = useRef<HTMLDivElement>(null);

  // Focus trap — only active when this modal is the topmost
  useEffect(() => {
    if (!isOpen || !isTop(id) || !dialogRef.current) return;

    const dialog = dialogRef.current;
    const focusable = dialog.querySelectorAll<HTMLElement>(
      'a[href], button:not([disabled]), input:not([disabled]), ' +
      'select:not([disabled]), textarea:not([disabled]), [tabindex]:not([tabindex="-1"])'
    );
    const first = focusable[0];
    const last  = focusable[focusable.length - 1];

    first?.focus();

    const trapTab = (e: globalThis.KeyboardEvent) => {
      if (e.key !== 'Tab') return;
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

    // Escape — only fire if this modal is on top
    const trapEscape = (e: globalThis.KeyboardEvent) => {
      if (e.key === 'Escape' && isTop(id)) {
        e.stopPropagation();   // prevent Escape from bubbling to a buried modal
        onClose();
      }
    };

    document.addEventListener('keydown', trapTab);
    document.addEventListener('keydown', trapEscape, { capture: true });
    return () => {
      document.removeEventListener('keydown', trapTab);
      document.removeEventListener('keydown', trapEscape, { capture: true });
    };
  }, [isOpen, id, isTop, onClose]);

  // Mark buried modals as inert so Tab cannot reach them
  useEffect(() => {
    if (!dialogRef.current) return;
    const buried = isOpen && !isTop(id);
    dialogRef.current.inert = buried;
    dialogRef.current.setAttribute('aria-hidden', String(buried));
  }, [isOpen, id, isTop]);

  if (!isOpen) return null;

  return (
    <>
      {/* Per-modal backdrop */}
      <div
        className={`modal-backdrop ${isOpen ? 'is-visible' : ''}`}
        style={{ zIndex: backdropZ }}
        aria-hidden="true"
        onClick={isTop(id) ? onClose : undefined}
      />

      {/* Modal panel */}
      <div
        ref={dialogRef}
        id={id}
        className={`modal ${isOpen ? 'is-open' : ''}`}
        role="dialog"
        aria-modal="true"
        aria-labelledby={`${id}-title`}
        aria-describedby={description ? `${id}-desc` : undefined}
        style={{ zIndex: modalZ }}
      >
        <div className="modal__inner">
          <h2 id={`${id}-title`} className="modal__title">{title}</h2>
          {description && (
            <p id={`${id}-desc`} className="modal__desc">{description}</p>
          )}
          <div className="modal__body">{children}</div>
          {footer && <div className="modal__footer">{footer}</div>}
        </div>
      </div>
    </>
  );
}
```

### Wiring it Together

```tsx
// App.tsx — demonstrates two levels of nesting
import { ModalStackProvider } from './modal-stack.context';
import { useModal } from './use-modal';
import { Modal } from './Modal';

function DatePickerModal() {
  const datePicker = useModal('date-picker-modal');

  return (
    <>
      <button className="btn btn--ghost" onClick={datePicker.open}>
        Choose date…
      </button>

      <Modal
        id="date-picker-modal"
        isOpen={datePicker.isOpen}
        onClose={datePicker.close}
        title="Pick a Date"
        footer={
          <button className="btn btn--primary" onClick={datePicker.close}>
            Confirm
          </button>
        }
      >
        <p>Date picker content here.</p>
      </Modal>
    </>
  );
}

function SettingsModal() {
  const settings = useModal('settings-modal');

  return (
    <>
      <button className="btn btn--primary" onClick={settings.open}>
        Open Settings
      </button>

      <Modal
        id="settings-modal"
        isOpen={settings.isOpen}
        onClose={settings.close}
        title="Edit Settings"
        description="Adjust your account preferences below."
        footer={
          <>
            <button className="btn btn--ghost" onClick={settings.close}>Cancel</button>
            <button className="btn btn--primary">Save</button>
          </>
        }
      >
        {/* DatePickerModal is nested *inside* SettingsModal's children */}
        <DatePickerModal />
      </Modal>
    </>
  );
}

export function App() {
  return (
    <ModalStackProvider>
      <SettingsModal />
    </ModalStackProvider>
  );
}
```

---

## Angular Implementation

### The Modal Stack Service

```typescript
// modal-stack.service.ts
import { Injectable, signal, computed } from '@angular/core';

export interface ModalEntry {
  id: string;
  opener: HTMLElement | null;
}

@Injectable({ providedIn: 'root' })
export class ModalStackService {
  private static BASE_Z  = 1000;
  private static Z_STEP  = 10;
  private lockCount      = 0;

  private _stack = signal<ModalEntry[]>([]);
  readonly stack = this._stack.asReadonly();

  readonly topId = computed(() => {
    const s = this._stack();
    return s.length > 0 ? s[s.length - 1].id : null;
  });

  push(id: string): void {
    const opener = document.activeElement as HTMLElement | null;
    this._stack.update(s => [...s, { id, opener }]);
    this.lockCount++;
    if (this.lockCount === 1) {
      document.body.classList.add('modal-open');
    }
  }

  pop(): void {
    this._stack.update(s => {
      const entry = s[s.length - 1];
      if (entry?.opener) {
        setTimeout(() => entry.opener!.focus(), 0);
      }
      return s.slice(0, -1);
    });
    this.lockCount = Math.max(0, this.lockCount - 1);
    if (this.lockCount === 0) {
      document.body.classList.remove('modal-open');
    }
  }

  isTop(id: string): boolean {
    return this.topId() === id;
  }

  zIndexFor(id: string): { backdrop: number; modal: number } {
    const idx  = this._stack().findIndex(e => e.id === id);
    const base = ModalStackService.BASE_Z + Math.max(0, idx) * ModalStackService.Z_STEP;
    return { backdrop: base, modal: base + 1 };
  }
}
```

### The Modal Component

```typescript
// modal.component.ts
import {
  Component, Input, Output, EventEmitter, OnChanges, OnDestroy,
  SimpleChanges, ElementRef, inject, effect, ChangeDetectionStrategy
} from '@angular/core';
import { NgIf } from '@angular/common';
import { ModalStackService } from './modal-stack.service';

@Component({
  selector: 'app-modal',
  standalone: true,
  imports: [NgIf],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <ng-container *ngIf="isOpen">
      <!-- Per-modal backdrop -->
      <div
        class="modal-backdrop"
        [class.is-visible]="isOpen"
        [style.z-index]="zIndex.backdrop"
        aria-hidden="true"
        (click)="onBackdropClick()"
      ></div>

      <!-- Modal panel -->
      <div
        #dialogEl
        [id]="id"
        class="modal"
        [class.is-open]="isOpen"
        role="dialog"
        aria-modal="true"
        [attr.aria-labelledby]="id + '-title'"
        [style.z-index]="zIndex.modal"
      >
        <div class="modal__inner">
          <h2 [id]="id + '-title'" class="modal__title">{{ title }}</h2>
          <p *ngIf="description" class="modal__desc">{{ description }}</p>
          <div class="modal__body">
            <ng-content></ng-content>
          </div>
          <div class="modal__footer" *ngIf="footer">
            <ng-content select="[modalFooter]"></ng-content>
          </div>
        </div>
      </div>
    </ng-container>
  `,
})
export class ModalComponent implements OnChanges, OnDestroy {
  @Input({ required: true }) id!: string;
  @Input() isOpen = false;
  @Input() title  = '';
  @Input() description?: string;
  @Input() footer = false;
  @Output() closed = new EventEmitter<void>();

  private stackService = inject(ModalStackService);
  private el           = inject(ElementRef);
  private keydownHandler?: (e: KeyboardEvent) => void;

  get zIndex() { return this.stackService.zIndexFor(this.id); }

  ngOnChanges(changes: SimpleChanges): void {
    if (!changes['isOpen']) return;
    if (this.isOpen) {
      this.stackService.push(this.id);
      // Defer focus trap setup until the DOM has rendered
      setTimeout(() => this.setupFocusTrap(), 0);
    } else {
      this.teardownFocusTrap();
      this.stackService.pop();
    }
  }

  onBackdropClick(): void {
    if (this.stackService.isTop(this.id)) {
      this.closed.emit();
    }
  }

  private setupFocusTrap(): void {
    const dialog = this.el.nativeElement.querySelector(`#${this.id}`);
    if (!dialog) return;

    const focusable: HTMLElement[] = Array.from(
      dialog.querySelectorAll(
        'a[href], button:not([disabled]), input:not([disabled]), ' +
        'select:not([disabled]), textarea:not([disabled]), [tabindex]:not([tabindex="-1"])'
      )
    );
    focusable[0]?.focus();

    const first = focusable[0];
    const last  = focusable[focusable.length - 1];

    this.keydownHandler = (e: KeyboardEvent) => {
      if (e.key === 'Escape' && this.stackService.isTop(this.id)) {
        e.stopPropagation();
        this.closed.emit();
        return;
      }
      if (e.key !== 'Tab') return;
      if (e.shiftKey) {
        if (document.activeElement === first) { e.preventDefault(); last?.focus(); }
      } else {
        if (document.activeElement === last) { e.preventDefault(); first?.focus(); }
      }
    };

    document.addEventListener('keydown', this.keydownHandler, { capture: true });
  }

  private teardownFocusTrap(): void {
    if (this.keydownHandler) {
      document.removeEventListener('keydown', this.keydownHandler, { capture: true });
      this.keydownHandler = undefined;
    }
  }

  ngOnDestroy(): void {
    if (this.isOpen) {
      this.teardownFocusTrap();
      this.stackService.pop();
    }
  }
}
```

### Using the Modal in a Parent Component

```typescript
// settings-page.component.ts
import { Component, signal } from '@angular/core';
import { ModalComponent } from './modal.component';

@Component({
  selector: 'app-settings-page',
  standalone: true,
  imports: [ModalComponent],
  template: `
    <button class="btn btn--primary" (click)="settingsOpen.set(true)">
      Open Settings
    </button>

    <app-modal
      id="settings-modal"
      title="Edit Settings"
      description="Adjust your account preferences below."
      [isOpen]="settingsOpen()"
      (closed)="settingsOpen.set(false)"
    >
      <!-- Nested modal trigger lives inside the outer modal's content -->
      <button class="btn btn--ghost" (click)="dateOpen.set(true)">
        Choose date…
      </button>

      <ng-container modalFooter>
        <button class="btn btn--ghost" (click)="settingsOpen.set(false)">Cancel</button>
        <button class="btn btn--primary">Save</button>
      </ng-container>
    </app-modal>

    <!-- Second modal is a sibling in the DOM but a child in the stack -->
    <app-modal
      id="date-picker-modal"
      title="Pick a Date"
      [isOpen]="dateOpen()"
      (closed)="dateOpen.set(false)"
    >
      <p>Date picker content here.</p>
      <ng-container modalFooter>
        <button class="btn btn--primary" (click)="dateOpen.set(false)">Confirm</button>
      </ng-container>
    </app-modal>
  `,
})
export class SettingsPageComponent {
  settingsOpen = signal(false);
  dateOpen     = signal(false);
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Each modal has `role="dialog"` and `aria-modal="true"` | Tells screen readers this is a modal boundary |
| Each modal has `aria-labelledby` pointing to its visible title | Title announced when the modal receives focus |
| Backdrop click closes only the topmost modal | `isTop(id)` guard in click handler |
| Escape closes only the topmost modal | `stopPropagation` on the capture-phase listener prevents bubbling to buried modals |
| Focus is trapped inside the topmost modal | Tab/Shift+Tab cycle stays within the modal's focusable elements |
| Buried modals are marked `inert` and `aria-hidden="true"` | Prevents virtual cursor and keyboard from reaching them |
| Focus is saved before opening each modal | `document.activeElement` stored in the stack entry at push time |
| Focus is restored to the correct opener on close | Each stack entry's `opener` reference is focused on pop |
| Body scroll is locked once, not once per modal | Reference-counted `lockCount` — only first push adds the class, only last pop removes it |
| Minimum focusable target size | CSS `min-height: 40px` on buttons |
| Animation respects `prefers-reduced-motion` | `@media` query removes all transitions |

---

## Production Pitfalls

**1. z-index values collide with other page layers**
If your app already has a header at z-index 100 or a toast notification at z-index 9999, a hardcoded `BASE_Z = 1000` will either bury or be buried by those elements. Fix: audit every z-index in your codebase and choose a BASE_Z above the highest non-modal layer. Document the z-index budget in a design token file (e.g., `--z-modal-base: 1000`).

**2. Escape bubbles through the stack and closes all modals at once**
If you attach Escape listeners on `document` without stopping propagation, a single Escape keypress fires all registered listeners simultaneously. Fix: register the Escape listener in the `capture` phase (`{ capture: true }`) so the topmost modal's handler fires first and calls `stopPropagation()` before any buried modal's handler sees the event.

**3. Focus stack goes stale after re-renders**
In React, if a modal is closed and reopened, the `opener` captured at push time may point to a DOM node that was unmounted and remounted between the two opens. Fix: always read `document.activeElement` fresh at `push()` time — never cache it before the open is triggered.

**4. `inert` is not supported in older browsers**
`element.inert = true` has full support from Chrome 102 / Firefox 112 / Safari 15.5 onwards, but older mobile browsers (especially old Samsung Internet) silently ignore it. Fix: add a polyfill (`wicg-inert`) or fall back to setting `tabindex="-1"` on all focusable children of buried modals via JS.

**5. Body scroll unlocks too early when closing with animation**
If you remove `modal-open` from `body` at the moment `pop()` is called, the page jumps before the closing animation finishes. Fix: delay the scroll unlock by the same duration as the CSS transition (e.g., `setTimeout(() => document.body.classList.remove('modal-open'), 220)`). Alternatively, only unlock when the `transitionend` event fires on the last modal's backdrop.

**6. Multiple modals compound background dimming unexpectedly**
Each backdrop adds 45% opacity on top of the previous one. With three modals open, the background is nearly black. Fix: either cap the cumulative dimming by reducing `--modal-backdrop-bg` opacity for nested modals (stack index-aware styling), or render a single shared backdrop at the bottom of the stack and let nested modals skip adding their own.

**7. Screen reader virtual cursor bypasses `aria-hidden`**
Some screen readers ignore `aria-hidden` on container elements and will still read children. The `inert` attribute is more reliable because it also removes focusability. Use both: `aria-hidden="true"` for AT compatibility and `inert` for DOM-level enforcement.

---

## Interview Angle

**Q: "You have a form in a modal. When the user clicks a date field inside that form, a date picker opens in a second modal. How do you ensure Escape, focus, and z-index all work correctly across both modals?"**

Strong answer covers four points:
1. **Focus stack, not a single reference** — before opening each modal, capture `document.activeElement` and push it onto an array. When each modal closes, pop the array and restore focus to that element. With two levels, closing the date picker returns focus to the date button inside the form modal — not to the page.
2. **Escape routing via capture-phase stopPropagation** — register Escape listeners in the capture phase so the topmost modal's listener fires first and calls `e.stopPropagation()`. This prevents the single Escape keypress from cascading down to the form modal below.
3. **z-index counter, not hardcoded values** — each modal pair (backdrop + panel) receives `BASE_Z + (stackIndex * STEP)`. The stack service computes this at render time, so the opener always sits on top regardless of how many modals are open.
4. **Bury non-top modals with `inert` + `aria-hidden`** — when a second modal opens, the first is marked `inert` so keyboard navigation cannot reach it. This also satisfies the ARIA requirement that only one `aria-modal="true"` region is interactable at a time.

**Follow-up: "What breaks in production that a unit test wouldn't catch?"**
The background scroll-lock race condition (body scroll flashes before the animation finishes), the `inert` polyfill gap on older Android WebView, and cumulative backdrop dimming making the UI look broken with three+ layers open — these only surface during real browser testing or accessibility audits.
