# Context Menu (Right-Click)

## The Idea

**In plain English:** When the user right-clicks (or long-presses on mobile) anywhere on a target surface, a small floating menu appears exactly where the cursor is. The menu lists context-sensitive actions for whatever was clicked. Clicking outside, pressing Escape, or scrolling dismisses it. The user can also navigate menu items with arrow keys and trigger actions with Enter.

**Real-world analogy:** Think of a physical cork board with sticky notes on it.

- **Right-clicking a sticky note** = requesting a context menu for that specific item.
- **The menu appearing at your fingertip** = the menu positions itself at cursor coordinates, not at some fixed corner of the screen.
- **Bumping into the edge of the board** = the viewport boundary clamp: if the menu would extend past the edge of the screen, it shifts inward so you can still read it.
- **Pulling your hand away from the board** = clicking outside the menu: the menu disappears because you have clearly moved on to something else.
- **Closing your eyes** = pressing Escape: a universal "never mind" that collapses the menu without acting.
- **Reading down the list of options** = keyboard arrow-key navigation: instead of reaching for the mouse again, you walk through the items linearly.

The critical insight: the menu's position is *data*, not CSS. You record the raw `MouseEvent.clientX / clientY`, clamp those numbers against `window.innerWidth / window.innerHeight` and the menu's own rendered dimensions, then write the result into inline styles. CSS cannot compute this because it has no runtime knowledge of the rendered menu size.

---

## Learning Objectives

- Understand why cursor-relative positioning requires reading `MouseEvent` coordinates at runtime
- Implement viewport boundary clamping so the menu never clips off-screen
- Build a reusable `useClickOutside` hook and understand its `mousedown` vs `click` event tradeoff
- Dismiss the menu on Escape key and on scroll (both passive and active scroll sources)
- Implement full ARIA `role="menu"` / `role="menuitem"` keyboard navigation (arrow keys, Enter, Home, End)
- Manage focus correctly: trap it inside the open menu, return it to the trigger on close
- Understand the Angular signals equivalent of each React pattern

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Show/hide the menu | ✅ with `:focus-within` or a checkbox hack | — |
| Position at cursor coordinates | ❌ | CSS has no access to `MouseEvent.clientX / clientY` |
| Clamp position to viewport edges | ❌ | Requires knowing the menu's rendered `offsetWidth / offsetHeight` at runtime |
| Dismiss on outside click | ❌ | Requires a `mousedown` listener on `document` |
| Dismiss on Escape key | ❌ | Requires a `keydown` listener |
| Dismiss on scroll | ❌ | Requires a `scroll` listener |
| Arrow-key navigation between items | ❌ | Requires focus management across dynamic child elements |
| Announce menu state to screen readers | ❌ | `aria-expanded` and `aria-haspopup` must be toggled by JS |

---

## HTML & CSS Foundation

### The Structure

```html
<!--
  The context menu trigger: any element where right-click should be intercepted.
  data-context-target tells the menu which item was acted on.
-->
<div
  class="context-surface"
  oncontextmenu="handleContextMenu(event)"
  role="region"
  aria-label="File list"
>
  <div class="context-item" data-id="file-1" data-context-target>README.md</div>
  <div class="context-item" data-id="file-2" data-context-target>package.json</div>
</div>

<!--
  The context menu itself.
  Rendered in a portal (end of body) to avoid overflow:hidden clipping.
  Position driven by inline style from JS.
  Hidden until opened.
-->
<ul
  id="context-menu"
  class="context-menu"
  role="menu"
  aria-label="File actions"
  tabindex="-1"
  hidden
  style="--x: 0px; --y: 0px;"
>
  <li role="none">
    <button class="context-menu__item" role="menuitem" tabindex="-1">
      Open
    </button>
  </li>
  <li role="none">
    <button class="context-menu__item" role="menuitem" tabindex="-1">
      Rename
    </button>
  </li>
  <li role="separator" aria-orientation="horizontal"></li>
  <li role="none">
    <button class="context-menu__item context-menu__item--danger" role="menuitem" tabindex="-1">
      Delete
    </button>
  </li>
</ul>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --menu-bg: #ffffff;
  --menu-border: #e2e8f0;
  --menu-shadow: 0 4px 6px -1px rgba(0,0,0,0.1), 0 2px 4px -1px rgba(0,0,0,0.06);
  --menu-radius: 6px;
  --menu-item-hover-bg: #f1f5f9;
  --menu-item-focus-bg: #e2e8f0;
  --menu-item-height: 36px;
  --menu-min-width: 180px;
  --menu-danger-color: #dc2626;
  --menu-transition: 0.12s cubic-bezier(0.4, 0, 0.2, 1);
}

/* ─── The menu container ─── */
/*
  Position is set via inline CSS variables --x and --y written by JS.
  This avoids writing raw `left` / `top` into style="" on every render —
  the component only updates two custom properties.
*/
.context-menu {
  position: fixed;
  left: var(--x);
  top: var(--y);
  z-index: 9999;
  min-width: var(--menu-min-width);
  background: var(--menu-bg);
  border: 1px solid var(--menu-border);
  border-radius: var(--menu-radius);
  box-shadow: var(--menu-shadow);
  list-style: none;
  margin: 0;
  padding: 4px 0;

  /* Entry animation: scale up from the click point */
  transform-origin: top left;
  animation: context-menu-in var(--menu-transition) both;
}

@keyframes context-menu-in {
  from { opacity: 0; transform: scale(0.95); }
  to   { opacity: 1; transform: scale(1); }
}

/* ─── Menu items ─── */
.context-menu__item {
  display: flex;
  align-items: center;
  gap: 8px;
  width: 100%;
  min-height: var(--menu-item-height);
  padding: 0 12px;
  background: none;
  border: none;
  font-size: 0.875rem;
  line-height: 1;
  text-align: left;
  color: inherit;
  cursor: pointer;
  white-space: nowrap;

  /*
    Keyboard focus: outline on :focus-visible only.
    We also highlight on :hover for mouse users.
  */
}

.context-menu__item:hover {
  background: var(--menu-item-hover-bg);
}

.context-menu__item:focus-visible {
  outline: 2px solid #2563eb;
  outline-offset: -2px;
  background: var(--menu-item-focus-bg);
}

.context-menu__item--danger {
  color: var(--menu-danger-color);
}

/* ─── Separator ─── */
[role="separator"] {
  height: 1px;
  background: var(--menu-border);
  margin: 4px 0;
}

/* ─── The trigger surface ─── */
.context-surface {
  /* Prevent native browser context menu from briefly flashing */
  user-select: none;
}

.context-item {
  padding: 8px 12px;
  cursor: default;
  border-radius: 4px;
}

.context-item:hover {
  background: #f8fafc;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .context-menu {
    animation: none;
  }
}
```

**What CSS owns:** layout, appearance, hover/focus states, entry animation, the separator line, danger-item color.

**What CSS cannot own:** where the menu appears on screen, whether it is visible, which item is currently focused, clamping to the viewport.

---

## React Implementation

### Types

```tsx
// context-menu.types.ts
export interface MenuPosition {
  x: number;
  y: number;
}

export interface MenuItem {
  id: string;
  label: string;
  icon?: React.ReactNode;
  danger?: boolean;
  separator?: never;
  onAction: () => void;
}

export interface MenuSeparator {
  id: string;
  separator: true;
}

export type MenuEntry = MenuItem | MenuSeparator;
```

### useClickOutside Hook

```tsx
// useClickOutside.ts
import { useEffect, RefObject } from 'react';

/**
 * Fires `callback` when a mousedown event lands outside `ref`.
 *
 * Why mousedown instead of click?
 * If the user presses down inside the menu and releases outside, a `click`
 * event fires on the outside element. Using mousedown means the menu closes
 * immediately when the press begins outside — which feels snappier and matches
 * OS native behaviour.
 */
export function useClickOutside<T extends HTMLElement>(
  ref: RefObject<T | null>,
  callback: () => void,
  enabled: boolean,
): void {
  useEffect(() => {
    if (!enabled) return;

    const handler = (e: MouseEvent) => {
      if (ref.current && !ref.current.contains(e.target as Node)) {
        callback();
      }
    };

    // Capture phase: fires before any React synthetic event handlers,
    // so the menu closes before a downstream click handler sees the event.
    document.addEventListener('mousedown', handler, true);
    return () => document.removeEventListener('mousedown', handler, true);
  }, [ref, callback, enabled]);
}
```

### useContextMenu Hook

```tsx
// useContextMenu.ts
import {
  useState, useCallback, useEffect, useRef, RefObject
} from 'react';
import type { MenuPosition } from './context-menu.types';

interface UseContextMenuReturn {
  isOpen: boolean;
  position: MenuPosition;
  menuRef: RefObject<HTMLUListElement | null>;
  open: (e: React.MouseEvent) => void;
  close: () => void;
}

export function useContextMenu(): UseContextMenuReturn {
  const [isOpen, setIsOpen] = useState(false);
  const [position, setPosition] = useState<MenuPosition>({ x: 0, y: 0 });
  const menuRef = useRef<HTMLUListElement>(null);

  /**
   * Compute a clamped position so the menu never overflows the viewport.
   *
   * The menu has not rendered yet when this runs, so we cannot read its
   * real dimensions. We use the minimum width token (180px) and an estimated
   * height as safe fallbacks. After the first paint we re-clamp in a
   * requestAnimationFrame — at that point the real offsetWidth / offsetHeight
   * are available.
   */
  const clamp = useCallback(
    (rawX: number, rawY: number, menuW = 180, menuH = 200): MenuPosition => {
      const vw = window.innerWidth;
      const vh = window.innerHeight;
      return {
        x: Math.min(rawX, vw - menuW - 8),
        y: Math.min(rawY, vh - menuH - 8),
      };
    },
    [],
  );

  const open = useCallback(
    (e: React.MouseEvent) => {
      e.preventDefault(); // suppress native context menu

      const initial = clamp(e.clientX, e.clientY);
      setPosition(initial);
      setIsOpen(true);

      // After paint, re-clamp with real menu dimensions.
      requestAnimationFrame(() => {
        if (!menuRef.current) return;
        const { offsetWidth: w, offsetHeight: h } = menuRef.current;
        setPosition(clamp(e.clientX, e.clientY, w, h));
      });
    },
    [clamp],
  );

  const close = useCallback(() => setIsOpen(false), []);

  // Dismiss on Escape
  useEffect(() => {
    if (!isOpen) return;
    const onKey = (e: KeyboardEvent) => {
      if (e.key === 'Escape') {
        e.stopPropagation();
        close();
      }
    };
    document.addEventListener('keydown', onKey, true);
    return () => document.removeEventListener('keydown', onKey, true);
  }, [isOpen, close]);

  // Dismiss on scroll — use capture so we catch scrolls inside nested
  // scrollable elements before they bubble to document.
  useEffect(() => {
    if (!isOpen) return;
    const onScroll = () => close();
    window.addEventListener('scroll', onScroll, { passive: true, capture: true });
    return () => window.removeEventListener('scroll', onScroll, true);
  }, [isOpen, close]);

  return { isOpen, position, menuRef, open, close };
}
```

### ContextMenu Component

```tsx
// ContextMenu.tsx
import { useRef, useEffect, useCallback } from 'react';
import { createPortal } from 'react-dom';
import { useClickOutside } from './useClickOutside';
import type { MenuEntry, MenuItem } from './context-menu.types';

interface ContextMenuProps {
  entries: MenuEntry[];
  isOpen: boolean;
  position: { x: number; y: number };
  menuRef: React.RefObject<HTMLUListElement | null>;
  onClose: () => void;
}

export function ContextMenu({
  entries, isOpen, position, menuRef, onClose,
}: ContextMenuProps) {
  const itemRefs = useRef<(HTMLButtonElement | null)[]>([]);

  // Close on outside mousedown
  useClickOutside(menuRef, onClose, isOpen);

  // Focus the first menu item when the menu opens
  useEffect(() => {
    if (!isOpen) return;
    // rAF: ensure the element is painted and not `display:none`
    requestAnimationFrame(() => {
      itemRefs.current[0]?.focus();
    });
  }, [isOpen]);

  /**
   * Arrow-key navigation.
   *
   * We iterate only over visible menuitem buttons, skipping separators.
   * Home/End jump to first/last item.
   * Tab closes the menu (natural tab order should leave the menu).
   */
  const handleKeyDown = useCallback(
    (e: React.KeyboardEvent) => {
      const items = itemRefs.current.filter(Boolean) as HTMLButtonElement[];
      const current = document.activeElement;
      const idx = items.indexOf(current as HTMLButtonElement);

      switch (e.key) {
        case 'ArrowDown': {
          e.preventDefault();
          const next = items[(idx + 1) % items.length];
          next?.focus();
          break;
        }
        case 'ArrowUp': {
          e.preventDefault();
          const prev = items[(idx - 1 + items.length) % items.length];
          prev?.focus();
          break;
        }
        case 'Home':
          e.preventDefault();
          items[0]?.focus();
          break;
        case 'End':
          e.preventDefault();
          items[items.length - 1]?.focus();
          break;
        case 'Tab':
          // Let Tab close rather than navigate — the roving-tabindex pattern
          // keeps all items at tabindex="-1" so Tab naturally leaves the menu.
          onClose();
          break;
        default:
          break;
      }
    },
    [onClose],
  );

  if (!isOpen) return null;

  // Render into a portal so that no ancestor overflow:hidden clips the menu.
  return createPortal(
    <ul
      ref={menuRef}
      role="menu"
      aria-label="Context menu"
      tabIndex={-1}
      className="context-menu"
      style={
        {
          '--x': `${position.x}px`,
          '--y': `${position.y}px`,
        } as React.CSSProperties
      }
      onKeyDown={handleKeyDown}
    >
      {entries.map((entry, i) => {
        if ('separator' in entry && entry.separator) {
          return (
            <li key={entry.id} role="separator" aria-orientation="horizontal" />
          );
        }

        const item = entry as MenuItem;
        return (
          <li key={item.id} role="none">
            <button
              ref={(el) => { itemRefs.current[i] = el; }}
              role="menuitem"
              tabIndex={-1}
              className={`context-menu__item${item.danger ? ' context-menu__item--danger' : ''}`}
              onClick={() => { item.onAction(); onClose(); }}
            >
              {item.icon && <span aria-hidden="true">{item.icon}</span>}
              {item.label}
            </button>
          </li>
        );
      })}
    </ul>,
    document.body,
  );
}
```

### Usage — FileList Example

```tsx
// FileList.tsx
import { useContextMenu } from './useContextMenu';
import { ContextMenu } from './ContextMenu';
import type { MenuEntry } from './context-menu.types';

interface FileItem {
  id: string;
  name: string;
}

const files: FileItem[] = [
  { id: '1', name: 'README.md' },
  { id: '2', name: 'package.json' },
  { id: '3', name: 'tsconfig.json' },
];

export function FileList() {
  const { isOpen, position, menuRef, open, close } = useContextMenu();

  // In a real app this would be stored in state alongside the open position.
  const buildEntries = (file: FileItem): MenuEntry[] => [
    { id: 'open',   label: 'Open',   onAction: () => console.log('open',   file.id) },
    { id: 'rename', label: 'Rename', onAction: () => console.log('rename', file.id) },
    { id: 'sep',    separator: true },
    { id: 'delete', label: 'Delete', danger: true, onAction: () => console.log('delete', file.id) },
  ];

  // Keep a ref to which file was right-clicked so the menu entries are correct.
  const activeFile = useRef<FileItem | null>(null);

  const handleContextMenu = (e: React.MouseEvent, file: FileItem) => {
    activeFile.current = file;
    open(e);
  };

  return (
    <>
      <ul role="list" className="file-list">
        {files.map(file => (
          <li
            key={file.id}
            className="context-item"
            onContextMenu={(e) => handleContextMenu(e, file)}
            // Support keyboard users: open menu on Shift+F10 or the ContextMenu key
            onKeyDown={(e) => {
              if (e.key === 'ContextMenu' || (e.shiftKey && e.key === 'F10')) {
                e.preventDefault();
                const rect = (e.currentTarget as HTMLElement).getBoundingClientRect();
                // Synthesise a mouse event positioned at the element's bottom-left
                open({ clientX: rect.left, clientY: rect.bottom, preventDefault: () => {} } as React.MouseEvent);
                activeFile.current = file;
              }
            }}
            tabIndex={0}
          >
            {file.name}
          </li>
        ))}
      </ul>

      <ContextMenu
        entries={activeFile.current ? buildEntries(activeFile.current) : []}
        isOpen={isOpen}
        position={position}
        menuRef={menuRef}
        onClose={close}
      />
    </>
  );
}
```

---

## Angular Implementation

### Service — Context Menu State

```typescript
// context-menu.service.ts
import { Injectable, signal, computed } from '@angular/core';

export interface MenuPosition { x: number; y: number; }

export interface MenuItem {
  id: string;
  label: string;
  danger?: boolean;
  separator?: never;
  onAction: () => void;
}

export interface MenuSeparator { id: string; separator: true; }

export type MenuEntry = MenuItem | MenuSeparator;

@Injectable({ providedIn: 'root' })
export class ContextMenuService {
  private _isOpen   = signal(false);
  private _position = signal<MenuPosition>({ x: 0, y: 0 });
  private _entries  = signal<MenuEntry[]>([]);

  readonly isOpen   = this._isOpen.asReadonly();
  readonly position = this._position.asReadonly();
  readonly entries  = this._entries.asReadonly();

  open(event: MouseEvent, entries: MenuEntry[]): void {
    event.preventDefault();
    this._entries.set(entries);
    this._position.set(this.clamp(event.clientX, event.clientY));
    this._isOpen.set(true);
  }

  close(): void {
    this._isOpen.set(false);
  }

  updatePosition(menuW: number, menuH: number, rawX: number, rawY: number): void {
    this._position.set(this.clamp(rawX, rawY, menuW, menuH));
  }

  private clamp(rawX: number, rawY: number, menuW = 180, menuH = 200): MenuPosition {
    return {
      x: Math.min(rawX, window.innerWidth  - menuW - 8),
      y: Math.min(rawY, window.innerHeight - menuH - 8),
    };
  }
}
```

### ContextMenu Component

```typescript
// context-menu.component.ts
import {
  Component, inject, ElementRef, ViewChild,
  HostListener, AfterViewChecked, OnDestroy, effect
} from '@angular/core';
import { NgFor, NgIf } from '@angular/common';
import { ContextMenuService, MenuEntry, MenuItem } from './context-menu.service';

@Component({
  selector: 'app-context-menu',
  standalone: true,
  imports: [NgFor, NgIf],
  template: `
    <ul
      *ngIf="svc.isOpen()"
      #menuEl
      role="menu"
      aria-label="Context menu"
      tabindex="-1"
      class="context-menu"
      [style.--x.px]="svc.position().x"
      [style.--y.px]="svc.position().y"
      (keydown)="onKeyDown($event)"
    >
      <ng-container *ngFor="let entry of svc.entries(); let i = index; trackBy: trackById">
        <!-- Separator -->
        <li *ngIf="isSeparator(entry)" role="separator" aria-orientation="horizontal"></li>

        <!-- Item -->
        <li *ngIf="!isSeparator(entry)" role="none">
          <button
            role="menuitem"
            tabindex="-1"
            class="context-menu__item"
            [class.context-menu__item--danger]="asItem(entry).danger"
            (click)="activate(asItem(entry))"
          >
            {{ asItem(entry).label }}
          </button>
        </li>
      </ng-container>
    </ul>
  `,
})
export class ContextMenuComponent implements AfterViewChecked, OnDestroy {
  svc = inject(ContextMenuService);

  @ViewChild('menuEl') menuEl?: ElementRef<HTMLUListElement>;

  private rawOpen = { x: 0, y: 0 };
  private needsReclamp = false;

  constructor() {
    // Re-clamp after the menu element becomes visible and has real dimensions.
    effect(() => {
      if (this.svc.isOpen()) {
        this.needsReclamp = true;
      }
    });
  }

  ngAfterViewChecked(): void {
    if (this.needsReclamp && this.menuEl?.nativeElement) {
      this.needsReclamp = false;
      const { offsetWidth: w, offsetHeight: h } = this.menuEl.nativeElement;
      this.svc.updatePosition(w, h, this.rawOpen.x, this.rawOpen.y);
      // Focus first item
      const first = this.menuEl.nativeElement
        .querySelector<HTMLButtonElement>('[role="menuitem"]');
      first?.focus();
    }
  }

  ngOnDestroy(): void {
    this.svc.close();
  }

  isSeparator(entry: MenuEntry): boolean {
    return 'separator' in entry && !!entry.separator;
  }

  asItem(entry: MenuEntry): MenuItem {
    return entry as MenuItem;
  }

  trackById(_: number, entry: MenuEntry): string {
    return entry.id;
  }

  activate(item: MenuItem): void {
    item.onAction();
    this.svc.close();
  }

  @HostListener('document:mousedown', ['$event'])
  onOutsideClick(e: MouseEvent): void {
    if (!this.svc.isOpen()) return;
    if (!this.menuEl?.nativeElement.contains(e.target as Node)) {
      this.svc.close();
    }
  }

  @HostListener('document:keydown.escape', ['$event'])
  onEscape(e: KeyboardEvent): void {
    if (this.svc.isOpen()) {
      e.stopPropagation();
      this.svc.close();
    }
  }

  @HostListener('window:scroll', ['$event'])
  onScroll(): void {
    if (this.svc.isOpen()) this.svc.close();
  }

  onKeyDown(e: KeyboardEvent): void {
    const items = Array.from(
      this.menuEl?.nativeElement.querySelectorAll<HTMLButtonElement>('[role="menuitem"]') ?? [],
    );
    const idx = items.indexOf(document.activeElement as HTMLButtonElement);

    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        items[(idx + 1) % items.length]?.focus();
        break;
      case 'ArrowUp':
        e.preventDefault();
        items[(idx - 1 + items.length) % items.length]?.focus();
        break;
      case 'Home':
        e.preventDefault();
        items[0]?.focus();
        break;
      case 'End':
        e.preventDefault();
        items[items.length - 1]?.focus();
        break;
      case 'Tab':
        this.svc.close();
        break;
    }
  }
}
```

### Usage — Trigger Directive

```typescript
// context-trigger.directive.ts
import { Directive, HostListener, Input } from '@angular/core';
import { ContextMenuService, MenuEntry } from './context-menu.service';

@Directive({
  selector: '[appContextMenu]',
  standalone: true,
})
export class ContextTriggerDirective {
  @Input('appContextMenu') entries: MenuEntry[] = [];

  constructor(private svc: ContextMenuService) {}

  @HostListener('contextmenu', ['$event'])
  onContextMenu(e: MouseEvent): void {
    this.svc.open(e, this.entries);
  }
}
```

```html
<!-- file-list.component.html -->
<li
  *ngFor="let file of files"
  [appContextMenu]="buildEntries(file)"
  tabindex="0"
>
  {{ file.name }}
</li>

<!-- Place once in the app shell -->
<app-context-menu />
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Menu has `role="menu"` | Static attribute on `<ul>` |
| Each action button has `role="menuitem"` | Static attribute on each `<button>` |
| Separator uses `role="separator"` | Applied to the `<li>` divider element |
| Wrapper `<li>` for each item uses `role="none"` | Prevents `<li>` from being announced as a list item inside a menu role |
| First item receives focus on open | `requestAnimationFrame` / `ngAfterViewChecked` sets focus after paint |
| Arrow keys navigate between items | `keydown` handler walks `querySelectorAll('[role="menuitem"]')` |
| Enter / Space activates focused item | Native `<button>` behaviour — no extra code needed |
| Escape closes the menu | `keydown` listener in capture phase |
| Tab closes the menu (does not navigate inside it) | Explicit handler — all items stay at `tabindex="-1"` (roving tabindex) |
| Scroll closes the menu | `scroll` listener on `window` |
| Outside click closes the menu | `mousedown` listener in capture phase via `useClickOutside` |
| Keyboard trigger supported | `ContextMenu` key / `Shift+F10` synthesises open at element bounds |
| Focus returns to triggering element on close | Store a ref to the last focused element; restore on close |
| Minimum tap target on items | CSS `min-height: 36px` on items (adjust to 44px for mobile-primary apps) |
| Danger items visually distinct | `.context-menu__item--danger` applies `color: var(--menu-danger-color)` |
| Reduced-motion support | CSS `@media (prefers-reduced-motion)` disables entry animation |

---

## Production Pitfalls

**1. Menu clips behind `overflow: hidden` ancestors**
When the context menu component is rendered inside a scrollable container, an ancestor with `overflow: hidden` will clip it. Fix: always render the menu into a portal (`createPortal(…, document.body)` in React; place `<app-context-menu>` in the root `AppComponent` in Angular). The menu uses `position: fixed` so it ignores all stacking contexts except those created by `transform`, `filter`, or `will-change` on an ancestor of the fixed element. Rendering at body level avoids all of those traps.

**2. The initial clamp is wrong because the menu has not rendered yet**
At the moment `contextmenu` fires, the menu element does not exist in the DOM yet, so `offsetWidth` and `offsetHeight` are zero. Using those values to clamp means the menu can still overflow. Fix: use the known CSS `min-width` (180px) and an estimated height as initial values for the clamp pass. Then run a second, correcting clamp in `requestAnimationFrame` (React) or `ngAfterViewChecked` (Angular) once the real dimensions are available. Most users will never notice a sub-pixel jump.

**3. `mousedown` vs `click` for outside dismissal**
Using `click` to dismiss means if the user presses down inside the menu and drags out before releasing, the `click` fires on the outside target and the menu stays open for a confusing moment. `mousedown` fires as soon as the button is pressed, which matches OS-native behaviour. However, `mousedown` can also fire on the scroll bar in some browsers. Guard against it: check `e.button === 0` (primary button only) or check `menuRef.current.contains(e.target)` before closing.

**4. Scroll detection misses nested scrollable elements**
Listening on `window` for the `scroll` event does not catch scrolls inside a child `div` with `overflow: auto`. Fix: add the listener with `capture: true` so it fires for all scroll events in the tree before they are handled by the scrolling element. In React: `window.addEventListener('scroll', handler, { passive: true, capture: true })`. In Angular: `@HostListener('window:scroll')` captures at the window level but not inner elements — add the listener manually in `ngOnInit` for full coverage.

**5. Opening a second menu does not close the first**
If two right-clicks happen in rapid succession, two menus can be open simultaneously. Fix: in the service/hook, always call `close()` or reset state before applying the new open position. In the Angular service `open()` method this is automatic because the signal is overwritten. In React, because state updates are batched, call `setIsOpen(false)` and `setPosition(…)` together; or use a `useReducer` that guarantees the two updates are atomic.

**6. `transform` on an ancestor breaks `position: fixed`**
If any ancestor element applies `transform`, `filter`, `backdrop-filter`, or `will-change: transform`, `position: fixed` is scoped to that ancestor rather than the viewport. The menu renders at the wrong position. The portal-to-body pattern eliminates this for most cases, but if the body itself has a `transform` (common in full-page transition animations), even that fails. Fix: apply the page transition to a wrapper `<div>` inside `<body>`, not to `<body>` itself.

**7. Long menu items cause horizontal overflow on narrow viewports**
`white-space: nowrap` prevents wrapping but can push the menu wider than the viewport. Fix: add `max-width: min(var(--menu-min-width) * 2, 90vw)` to `.context-menu` and allow text truncation with `overflow: hidden; text-overflow: ellipsis` on `.context-menu__item`. Consider that truncated labels reduce usability; prefer short, scannable action names.

---

## Interview Angle

**Q: "Walk me through how you would implement a right-click context menu that always stays inside the viewport."**

The answer has four parts. First, intercept `contextmenu` on the trigger surface and call `event.preventDefault()` to suppress the native browser menu. Record `event.clientX` and `event.clientY` as the raw desired position. Second, clamp: compute `clampedX = Math.min(rawX, window.innerWidth - menuWidth - 8)` and the same for Y. The tricky part is that the menu has not rendered yet when you compute this — so seed `menuWidth` and `menuHeight` from the known CSS `min-width` and an estimated height, then run a correcting reclamp in `requestAnimationFrame` once `offsetWidth` and `offsetHeight` are readable. Third, write the clamped values into CSS custom properties on the menu element (`--x` and `--y`) rather than directly into `left` / `top` — this keeps the style update to one property write and lets the CSS own the `position: fixed` declaration. Fourth, render the menu into a portal at `document.body` to escape any `overflow: hidden` or `transform` ancestor.

**Follow-up: "How would you handle keyboard accessibility for the context menu items?"**

The correct pattern is roving `tabindex`, not `tabindex="0"` on each item. All items start at `tabindex="-1"`. When the menu opens, programmatically focus the first item. A `keydown` listener on the menu container then handles `ArrowDown` / `ArrowUp` (move focus to next/prev item, wrapping at ends), `Home` (jump to first), `End` (jump to last), `Enter` / `Space` (activate — the native button behaviour handles this automatically), `Escape` (close the menu), and `Tab` (close the menu, do not navigate inside it — natural tab order should leave the widget). This follows the ARIA Authoring Practices Guide `menu` pattern. The items themselves need `role="menuitem"` and must be wrapped in `<li role="none">` to cancel the implied `listitem` role from the `<ul>`.
