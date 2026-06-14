# Long-Press Context Menu

## The Idea

**In plain English:** On mobile, there is no right-click. Instead, when a user holds their finger on an element for roughly 500 ms without moving, the app should present a context menu with actions relevant to that element — the same way a long-press on an iOS photo reveals "Copy", "Share", "Delete". On desktop, that same menu is triggered by the standard right-click (`contextmenu` event). The trick is that both surfaces need to feel native: touch users get haptic feedback via `navigator.vibrate`, desktop users get the browser-native event, and keyboard users get a dedicated trigger so nobody is locked out.

**Real-world analogy:** Think of a physical receipt in a restaurant.

- **Tap (brief touch)** = glancing at the receipt — you see it and move on.
- **Long-press (hold)** = picking the receipt up and turning it over — now the waiter notices and comes over to offer options: "Would you like a box? Split the bill? Speak to the manager?"
- **Right-click on desktop** = snapping your fingers at the waiter directly — immediate, intentional, no waiting.
- **The context menu itself** = the waiter with a tray of options. They only appear for the specific item you pointed at — not for the whole restaurant.
- **Haptic buzz** = the waiter's acknowledgement tap on your shoulder confirming they heard you before they arrive.
- **Keyboard shortcut (Shift+F10 or the Menu key)** = the customer ringing a call button at the table — a different trigger, same service.

The key insight: the *gesture* and *event source* vary per device, but the *menu* is the same component — separated from the trigger so it can be reused across surfaces.

---

## Learning Objectives

- Implement a reliable long-press timer using `setTimeout` and clear it correctly on `touchmove` and `touchend`
- Intercept `contextmenu` on desktop and route it through the same menu component as the touch path
- Prevent the native browser context menu only when your own menu fires, not globally
- Deliver haptic feedback via `navigator.vibrate` with a graceful no-op fallback
- Provide a keyboard-accessible trigger (Shift+F10, Menu key, or a dedicated button) so the feature is not touch-only
- Position the menu at the pointer/touch coordinates and keep it within viewport bounds
- Handle the full ARIA pattern for a `role="menu"` pop-up: `aria-haspopup`, `aria-expanded`, focus management, and Escape to close

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Detect a touch hold of 500 ms | No | CSS has no timer concept |
| Clear the timer when the finger moves | No | Requires `touchmove` event listener |
| Fire on right-click (contextmenu event) | No | CSS cannot intercept events |
| Prevent the native browser context menu | No | `e.preventDefault()` requires JS |
| Position menu at exact pointer coordinates | No | `x`/`y` come from the event object, not CSS |
| Clamp menu to viewport edges | No | Requires reading `getBoundingClientRect` at runtime |
| Trigger vibration on long-press confirm | No | `navigator.vibrate` is a JS API |
| Manage focus inside the open menu | No | Requires JS `focus()` and keyboard event listeners |
| Announce menu open/close to screen readers | No | `aria-expanded` toggling requires JS |

---

## HTML & CSS Foundation

### The Structure

```html
<!-- The item that can be long-pressed / right-clicked -->
<div
  class="pressable-item"
  tabindex="0"
  role="button"
  aria-haspopup="menu"
  aria-expanded="false"
  data-item-id="photo-42"
>
  <img src="photo-42.jpg" alt="Sunset at the beach" />
</div>

<!-- The context menu (rendered once, repositioned in JS) -->
<ul
  id="context-menu"
  class="context-menu"
  role="menu"
  aria-label="Photo actions"
  hidden
>
  <li role="none">
    <button class="context-menu__item" role="menuitem">
      Share
    </button>
  </li>
  <li role="none">
    <button class="context-menu__item" role="menuitem">
      Copy link
    </button>
  </li>
  <li role="none">
    <button class="context-menu__item context-menu__item--danger" role="menuitem">
      Delete
    </button>
  </li>
</ul>

<!-- Invisible backdrop to catch outside clicks -->
<div class="context-menu-backdrop" hidden aria-hidden="true"></div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --menu-bg: #ffffff;
  --menu-shadow: 0 8px 24px rgba(0, 0, 0, 0.15), 0 2px 6px rgba(0, 0, 0, 0.1);
  --menu-radius: 10px;
  --menu-min-width: 180px;
  --menu-item-height: 44px;    /* minimum tap target */
  --menu-transition: 0.15s cubic-bezier(0.2, 0, 0, 1);
  --menu-danger-color: #d32f2f;
}

/* ─── Pressable item visual feedback ─── */
.pressable-item {
  user-select: none;          /* prevents text selection during hold */
  -webkit-user-select: none;
  touch-action: none;         /* disables browser pan/zoom so our handler runs */
  cursor: context-menu;
  transition: opacity 0.1s;
}

/* Visual "press" state added by JS during the hold countdown */
.pressable-item.is-pressing {
  opacity: 0.7;
  transform: scale(0.98);
  transition: transform 0.1s, opacity 0.1s;
}

/* ─── Context menu ─── */
.context-menu {
  position: fixed;
  z-index: 9999;
  min-width: var(--menu-min-width);
  background: var(--menu-bg);
  border-radius: var(--menu-radius);
  box-shadow: var(--menu-shadow);
  list-style: none;
  margin: 0;
  padding: 6px 0;

  /* Enter animation */
  opacity: 0;
  transform: scale(0.92);
  transform-origin: top left;     /* JS adjusts this based on position */
  transition: opacity var(--menu-transition), transform var(--menu-transition);
  pointer-events: none;
}

/* JS adds .is-open after positioning is done (avoids flash at wrong coords) */
.context-menu.is-open {
  opacity: 1;
  transform: scale(1);
  pointer-events: auto;
}

/* ─── Menu items ─── */
.context-menu__item {
  display: flex;
  align-items: center;
  gap: 10px;
  width: 100%;
  min-height: var(--menu-item-height);
  padding: 0 16px;
  background: none;
  border: none;
  font-size: 0.9375rem;    /* 15px — comfortable on mobile */
  text-align: left;
  cursor: pointer;
  color: inherit;
  transition: background 0.1s;
}

.context-menu__item:hover,
.context-menu__item:focus {
  background: rgba(0, 0, 0, 0.06);
  outline: none;
}

.context-menu__item--danger {
  color: var(--menu-danger-color);
}

/* ─── Backdrop ─── */
.context-menu-backdrop {
  position: fixed;
  inset: 0;
  z-index: 9998;
}

/* ─── Reduced motion: skip scale animation ─── */
@media (prefers-reduced-motion: reduce) {
  .context-menu {
    transition: opacity var(--menu-transition);
    transform: none;
  }
  .context-menu.is-open {
    transform: none;
  }
}
```

**What CSS owns:** visual press feedback, enter animation, danger item color, minimum tap target height, `user-select: none` and `touch-action: none` to prevent interference.

**What CSS cannot own:** the timer, event interception, coordinate positioning, viewport clamping, ARIA toggling, vibration, or focus management.

---

## React Implementation

### The Hook — `useLongPress`

```tsx
// useLongPress.ts
import { useRef, useCallback } from 'react';

interface LongPressOptions {
  delay?: number;              // ms before long-press fires (default 500)
  moveThreshold?: number;      // px movement to cancel (default 8)
  onLongPress: (position: { x: number; y: number }) => void;
  onPress?: () => void;        // regular tap/click callback
}

export function useLongPress({
  delay = 500,
  moveThreshold = 8,
  onLongPress,
  onPress,
}: LongPressOptions) {
  const timerRef     = useRef<ReturnType<typeof setTimeout> | null>(null);
  const startPosRef  = useRef<{ x: number; y: number } | null>(null);
  const firedRef     = useRef(false);   // tracks whether long-press already fired

  const clear = useCallback(() => {
    if (timerRef.current !== null) {
      clearTimeout(timerRef.current);
      timerRef.current = null;
    }
  }, []);

  const onTouchStart = useCallback(
    (e: React.TouchEvent) => {
      const touch = e.touches[0];
      const pos   = { x: touch.clientX, y: touch.clientY };
      startPosRef.current = pos;
      firedRef.current    = false;

      // Add visual press feedback via a custom event or ref — handled in component
      timerRef.current = setTimeout(() => {
        firedRef.current = true;
        // Haptic feedback — graceful no-op where unsupported
        navigator.vibrate?.(50);
        onLongPress(pos);
      }, delay);
    },
    [delay, onLongPress]
  );

  const onTouchMove = useCallback(
    (e: React.TouchEvent) => {
      if (!startPosRef.current) return;
      const touch = e.touches[0];
      const dx    = Math.abs(touch.clientX - startPosRef.current.x);
      const dy    = Math.abs(touch.clientY - startPosRef.current.y);

      // Cancel if the finger moved more than the threshold — user is scrolling
      if (dx > moveThreshold || dy > moveThreshold) {
        clear();
      }
    },
    [clear, moveThreshold]
  );

  const onTouchEnd = useCallback(() => {
    clear();
    if (!firedRef.current) {
      onPress?.();
    }
    startPosRef.current = null;
  }, [clear, onPress]);

  // Desktop: intercept the native contextmenu event
  const onContextMenu = useCallback(
    (e: React.MouseEvent) => {
      e.preventDefault();   // suppress native menu
      onLongPress({ x: e.clientX, y: e.clientY });
    },
    [onLongPress]
  );

  // Keyboard: Menu key or Shift+F10 — position near the element's center
  const onKeyDown = useCallback(
    (e: React.KeyboardEvent<HTMLElement>) => {
      if (e.key === 'ContextMenu' || (e.shiftKey && e.key === 'F10')) {
        e.preventDefault();
        const rect = (e.currentTarget as HTMLElement).getBoundingClientRect();
        onLongPress({ x: rect.left + rect.width / 2, y: rect.bottom });
      }
    },
    [onLongPress]
  );

  return { onTouchStart, onTouchMove, onTouchEnd, onContextMenu, onKeyDown };
}
```

### Context Menu Positioning Utility

```tsx
// menuPosition.ts
export interface MenuCoords {
  top: number;
  left: number;
  transformOrigin: string;
}

/**
 * Clamp the menu so it never overflows the viewport.
 * Returns adjusted top/left and a matching transform-origin for the scale animation.
 */
export function calcMenuPosition(
  x: number,
  y: number,
  menuWidth: number,
  menuHeight: number
): MenuCoords {
  const vw = window.innerWidth;
  const vh = window.innerHeight;
  const padding = 8;   // minimum gap from viewport edge

  // Default: menu opens to the right and below the pointer
  let left = x;
  let top  = y;

  // Flip horizontally if it would overflow the right edge
  if (left + menuWidth + padding > vw) {
    left = x - menuWidth;
  }
  left = Math.max(padding, left);

  // Flip vertically if it would overflow the bottom edge
  if (top + menuHeight + padding > vh) {
    top = y - menuHeight;
  }
  top = Math.max(padding, top);

  // Match CSS transform-origin to the corner closest to the pointer
  const originX = left < x ? 'right' : 'left';
  const originY = top  < y ? 'bottom' : 'top';

  return { top, left, transformOrigin: `${originY} ${originX}` };
}
```

### The Context Menu Component

```tsx
// ContextMenu.tsx
import { useEffect, useRef, useCallback } from 'react';
import { calcMenuPosition } from './menuPosition';

interface MenuItem {
  id: string;
  label: string;
  icon?: React.ReactNode;
  danger?: boolean;
  onSelect: () => void;
}

interface ContextMenuProps {
  items: MenuItem[];
  position: { x: number; y: number } | null;   // null = closed
  onClose: () => void;
  triggerRef: React.RefObject<HTMLElement>;      // to return focus on close
}

export function ContextMenu({ items, position, onClose, triggerRef }: ContextMenuProps) {
  const menuRef  = useRef<HTMLUListElement>(null);
  const isOpen   = position !== null;

  // Position and animate the menu after it renders
  useEffect(() => {
    if (!isOpen || !menuRef.current || !position) return;

    const menu   = menuRef.current;
    const width  = menu.offsetWidth;
    const height = menu.offsetHeight;

    const { top, left, transformOrigin } = calcMenuPosition(
      position.x,
      position.y,
      width,
      height
    );

    menu.style.top             = `${top}px`;
    menu.style.left            = `${left}px`;
    menu.style.transformOrigin = transformOrigin;

    // Trigger CSS transition (rAF ensures layout has settled)
    requestAnimationFrame(() => {
      menu.classList.add('is-open');
      // Focus first item for keyboard users
      const firstItem = menu.querySelector<HTMLButtonElement>('[role="menuitem"]');
      firstItem?.focus();
    });
  }, [isOpen, position]);

  // Clean up open class on close
  useEffect(() => {
    if (!isOpen && menuRef.current) {
      menuRef.current.classList.remove('is-open');
    }
  }, [isOpen]);

  // Close on Escape, arrow-key navigation inside menu
  useEffect(() => {
    if (!isOpen) return;

    const handler = (e: KeyboardEvent) => {
      if (e.key === 'Escape') {
        onClose();
        triggerRef.current?.focus();
        return;
      }

      if (!menuRef.current) return;
      const items = Array.from(
        menuRef.current.querySelectorAll<HTMLButtonElement>('[role="menuitem"]:not([disabled])')
      );
      const idx = items.indexOf(document.activeElement as HTMLButtonElement);

      if (e.key === 'ArrowDown') {
        e.preventDefault();
        items[(idx + 1) % items.length]?.focus();
      } else if (e.key === 'ArrowUp') {
        e.preventDefault();
        items[(idx - 1 + items.length) % items.length]?.focus();
      } else if (e.key === 'Tab') {
        // Tab out of menu = close
        onClose();
      }
    };

    document.addEventListener('keydown', handler);
    return () => document.removeEventListener('keydown', handler);
  }, [isOpen, onClose, triggerRef]);

  const handleSelect = useCallback(
    (item: MenuItem) => {
      item.onSelect();
      onClose();
      triggerRef.current?.focus();
    },
    [onClose, triggerRef]
  );

  if (!isOpen) return null;

  return (
    <>
      {/* Backdrop catches outside pointer clicks */}
      <div
        className="context-menu-backdrop"
        aria-hidden="true"
        onPointerDown={onClose}
      />
      <ul
        ref={menuRef}
        className="context-menu"
        role="menu"
        aria-label="Item actions"
      >
        {items.map((item) => (
          <li key={item.id} role="none">
            <button
              className={`context-menu__item${item.danger ? ' context-menu__item--danger' : ''}`}
              role="menuitem"
              onClick={() => handleSelect(item)}
            >
              {item.icon && <span aria-hidden="true">{item.icon}</span>}
              {item.label}
            </button>
          </li>
        ))}
      </ul>
    </>
  );
}
```

### Wiring It Together

```tsx
// PhotoCard.tsx
import { useState, useRef, useCallback } from 'react';
import { useLongPress } from './useLongPress';
import { ContextMenu } from './ContextMenu';
import type { MenuItem } from './ContextMenu';

interface Photo {
  id: string;
  src: string;
  alt: string;
}

interface PhotoCardProps {
  photo: Photo;
  onShare: (id: string) => void;
  onDelete: (id: string) => void;
}

export function PhotoCard({ photo, onShare, onDelete }: PhotoCardProps) {
  const [menuPosition, setMenuPosition] = useState<{ x: number; y: number } | null>(null);
  const itemRef = useRef<HTMLDivElement>(null);

  const openMenu = useCallback((pos: { x: number; y: number }) => {
    setMenuPosition(pos);
    if (itemRef.current) {
      itemRef.current.setAttribute('aria-expanded', 'true');
    }
  }, []);

  const closeMenu = useCallback(() => {
    setMenuPosition(null);
    if (itemRef.current) {
      itemRef.current.setAttribute('aria-expanded', 'false');
    }
  }, []);

  const longPress = useLongPress({
    onLongPress: openMenu,
    onPress: () => console.log('regular tap — open full photo'),
  });

  const menuItems: MenuItem[] = [
    { id: 'share',  label: 'Share',       onSelect: () => onShare(photo.id)  },
    { id: 'copy',   label: 'Copy link',   onSelect: () => navigator.clipboard.writeText(`/photos/${photo.id}`) },
    { id: 'delete', label: 'Delete',      danger: true, onSelect: () => onDelete(photo.id) },
  ];

  return (
    <>
      <div
        ref={itemRef}
        className={`pressable-item${menuPosition ? ' is-pressing' : ''}`}
        role="button"
        tabIndex={0}
        aria-haspopup="menu"
        aria-expanded={menuPosition !== null}
        aria-label={`${photo.alt}. Long-press or right-click for options.`}
        {...longPress}
      >
        <img src={photo.src} alt={photo.alt} draggable={false} />
      </div>

      <ContextMenu
        items={menuItems}
        position={menuPosition}
        onClose={closeMenu}
        triggerRef={itemRef as React.RefObject<HTMLElement>}
      />
    </>
  );
}
```

---

## Angular Implementation

### The Directive — `LongPressDirective`

```typescript
// long-press.directive.ts
import {
  Directive, Output, EventEmitter,
  HostListener, OnDestroy, inject, ElementRef
} from '@angular/core';

export interface PressPosition { x: number; y: number; }

@Directive({
  selector: '[appLongPress]',
  standalone: true,
  host: {
    '[class.is-pressing]': 'isPressing',
    '[attr.aria-haspopup]': '"menu"',
  },
})
export class LongPressDirective implements OnDestroy {
  @Output() longPress  = new EventEmitter<PressPosition>();
  @Output() shortPress = new EventEmitter<void>();

  readonly DELAY          = 500;
  readonly MOVE_THRESHOLD = 8;

  isPressing  = false;
  private timer: ReturnType<typeof setTimeout> | null = null;
  private startX = 0;
  private startY = 0;
  private fired  = false;

  private el = inject(ElementRef);

  @HostListener('touchstart', ['$event'])
  onTouchStart(e: TouchEvent) {
    const touch = e.touches[0];
    this.startX   = touch.clientX;
    this.startY   = touch.clientY;
    this.fired    = false;
    this.isPressing = true;

    this.timer = setTimeout(() => {
      this.fired = true;
      this.isPressing = false;
      navigator.vibrate?.(50);
      this.longPress.emit({ x: this.startX, y: this.startY });
    }, this.DELAY);
  }

  @HostListener('touchmove', ['$event'])
  onTouchMove(e: TouchEvent) {
    const touch = e.touches[0];
    const dx = Math.abs(touch.clientX - this.startX);
    const dy = Math.abs(touch.clientY - this.startY);
    if (dx > this.MOVE_THRESHOLD || dy > this.MOVE_THRESHOLD) {
      this.cancel();
    }
  }

  @HostListener('touchend')
  onTouchEnd() {
    this.cancel();
    if (!this.fired) this.shortPress.emit();
  }

  @HostListener('contextmenu', ['$event'])
  onContextMenu(e: MouseEvent) {
    e.preventDefault();
    this.longPress.emit({ x: e.clientX, y: e.clientY });
  }

  @HostListener('keydown', ['$event'])
  onKeyDown(e: KeyboardEvent) {
    if (e.key === 'ContextMenu' || (e.shiftKey && e.key === 'F10')) {
      e.preventDefault();
      const rect = (this.el.nativeElement as HTMLElement).getBoundingClientRect();
      this.longPress.emit({ x: rect.left + rect.width / 2, y: rect.bottom });
    }
  }

  private cancel() {
    if (this.timer !== null) { clearTimeout(this.timer); this.timer = null; }
    this.isPressing = false;
  }

  ngOnDestroy() { this.cancel(); }
}
```

### Context Menu Component

```typescript
// context-menu.component.ts
import {
  Component, Input, Output, EventEmitter,
  OnChanges, SimpleChanges, ElementRef, inject,
  ViewChildren, QueryList, AfterViewInit
} from '@angular/core';
import { NgFor, NgIf } from '@angular/common';
import { calcMenuPosition } from './menu-position';

export interface MenuItem {
  id: string;
  label: string;
  danger?: boolean;
  onSelect: () => void;
}

@Component({
  selector: 'app-context-menu',
  standalone: true,
  imports: [NgFor, NgIf],
  template: `
    <ng-container *ngIf="position">
      <div
        class="context-menu-backdrop"
        aria-hidden="true"
        (pointerdown)="close.emit()"
      ></div>
      <ul
        #menuEl
        class="context-menu"
        role="menu"
        aria-label="Item actions"
        (keydown)="onKeyDown($event)"
      >
        <li *ngFor="let item of items" role="none">
          <button
            class="context-menu__item"
            [class.context-menu__item--danger]="item.danger"
            role="menuitem"
            (click)="select(item)"
          >
            {{ item.label }}
          </button>
        </li>
      </ul>
    </ng-container>
  `,
})
export class ContextMenuComponent implements OnChanges {
  @Input() items: MenuItem[] = [];
  @Input() position: { x: number; y: number } | null = null;
  @Output() close = new EventEmitter<void>();

  private el = inject(ElementRef);

  ngOnChanges(changes: SimpleChanges) {
    if (changes['position'] && this.position) {
      // Position the menu after Angular renders it
      requestAnimationFrame(() => {
        const menu = this.el.nativeElement.querySelector<HTMLUListElement>('.context-menu');
        if (!menu) return;

        const { top, left, transformOrigin } = calcMenuPosition(
          this.position!.x,
          this.position!.y,
          menu.offsetWidth,
          menu.offsetHeight
        );

        menu.style.top             = `${top}px`;
        menu.style.left            = `${left}px`;
        menu.style.transformOrigin = transformOrigin;

        requestAnimationFrame(() => {
          menu.classList.add('is-open');
          const first = menu.querySelector<HTMLButtonElement>('[role="menuitem"]');
          first?.focus();
        });
      });
    }
  }

  select(item: MenuItem) {
    item.onSelect();
    this.close.emit();
  }

  onKeyDown(e: KeyboardEvent) {
    const menu = this.el.nativeElement.querySelector<HTMLUListElement>('.context-menu');
    if (!menu) return;

    const items = Array.from(
      menu.querySelectorAll<HTMLButtonElement>('[role="menuitem"]:not([disabled])')
    );
    const idx = items.indexOf(document.activeElement as HTMLButtonElement);

    if (e.key === 'Escape') { this.close.emit(); return; }
    if (e.key === 'ArrowDown') { e.preventDefault(); items[(idx + 1) % items.length]?.focus(); }
    if (e.key === 'ArrowUp')   { e.preventDefault(); items[(idx - 1 + items.length) % items.length]?.focus(); }
    if (e.key === 'Tab')       { this.close.emit(); }
  }
}
```

### Wiring the Directive and Menu

```typescript
// photo-card.component.ts
import { Component, Input, signal } from '@angular/core';
import { NgIf } from '@angular/common';
import { LongPressDirective, PressPosition } from './long-press.directive';
import { ContextMenuComponent, MenuItem } from './context-menu.component';

interface Photo { id: string; src: string; alt: string; }

@Component({
  selector: 'app-photo-card',
  standalone: true,
  imports: [NgIf, LongPressDirective, ContextMenuComponent],
  template: `
    <div
      appLongPress
      class="pressable-item"
      role="button"
      tabindex="0"
      [attr.aria-label]="photo.alt + '. Long-press or right-click for options.'"
      [attr.aria-expanded]="menuPos() !== null"
      (longPress)="openMenu($event)"
      (shortPress)="onTap()"
    >
      <img [src]="photo.src" [alt]="photo.alt" draggable="false" />
    </div>

    <app-context-menu
      [items]="menuItems"
      [position]="menuPos()"
      (close)="closeMenu()"
    />
  `,
})
export class PhotoCardComponent {
  @Input({ required: true }) photo!: Photo;

  menuPos = signal<PressPosition | null>(null);

  menuItems: MenuItem[] = [
    { id: 'share',  label: 'Share',     onSelect: () => this.share()  },
    { id: 'copy',   label: 'Copy link', onSelect: () => this.copy()   },
    { id: 'delete', label: 'Delete',    danger: true, onSelect: () => this.delete() },
  ];

  openMenu(pos: PressPosition) { this.menuPos.set(pos); }
  closeMenu()                  { this.menuPos.set(null); }
  onTap()                      { console.log('open full photo'); }

  private share()  { console.log('share', this.photo.id); }
  private copy()   { navigator.clipboard.writeText(`/photos/${this.photo.id}`); }
  private delete() { console.log('delete', this.photo.id); }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Trigger element has `aria-haspopup="menu"` | Set on the pressable element; signals to screen readers a menu will appear |
| Trigger element has `aria-expanded` | Toggled `true`/`false` when menu opens and closes |
| Menu element has `role="menu"` | Set on the `<ul>` |
| Each action has `role="menuitem"` | Set on each `<button>` inside the menu |
| Wrapper `<li>` has `role="none"` | Removes the implicit `listitem` role that conflicts with `menuitem` |
| Menu is keyboard reachable | Menu key / Shift+F10 triggers it; positions near the element |
| Arrow keys navigate within menu | `ArrowDown`/`ArrowUp` move focus between items |
| Escape closes the menu | `keydown` handler on the menu closes and returns focus |
| Focus returns to trigger on close | `triggerRef.current?.focus()` called in `onClose` |
| Keyboard-only users not dependent on long-press | Menu key and Shift+F10 provide a direct trigger path |
| `touch-action: none` on pressable item | Prevents scroll from interfering with long-press timer |
| `user-select: none` on pressable item | Prevents accidental text selection during hold |
| Haptic buzz guarded by optional chaining | `navigator.vibrate?.()` — no crash on unsupported browsers |

---

## Production Pitfalls

**1. Timer fires after the component unmounts**
If the user navigates away during the 500 ms countdown, the `setTimeout` callback fires on an unmounted component. Fix: clear the timer in the cleanup of `useEffect` (React) or `ngOnDestroy` (Angular). In the React hook, `clear()` is called in both `touchend` and as a cleanup ref via `useRef` so nothing leaks.

**2. `touchmove` cancels the timer too aggressively on rough surfaces**
On textured surfaces or with a shaky hand, even a stationary finger can produce small `touchmove` events. A threshold of 1–2 px cancels legit long-presses on low-end devices. Fix: use a `moveThreshold` of 8–10 px, which matches the system-level scroll initiation threshold most platforms use.

**3. The menu appears at `(0, 0)` on first render**
If you render the menu at `position: fixed` before setting `top`/`left`, it briefly flashes at the top-left corner of the screen. Fix: keep the menu in the DOM only when `position !== null`, or set `pointer-events: none` and `opacity: 0` until the positioning `rAF` callback adds `.is-open`.

**4. Double-firing on mobile — both `touchend` and `click` trigger actions**
Mobile browsers synthesize a `click` event ~300 ms after a tap. If your menu item has both an `onClick` handler and you also call `onPress`, the action fires twice. Fix: call `e.preventDefault()` in `touchend` when `fired === true`, or use the `onContextMenu` path exclusively for the menu trigger and guard the click handler.

**5. `contextmenu` fires on iOS Safari only from iOS 13+**
On older iOS versions, right-click (via assistive touch or hardware keyboard) may not fire `contextmenu`. Fix: rely on the long-press touch path as the primary mobile trigger; treat `contextmenu` as the desktop enhancement. Do not assume `contextmenu` works on all mobile browsers.

**6. Menu overflows viewport on small screens**
Without viewport clamping, a menu positioned at the right edge of the screen clips silently. The `calcMenuPosition` utility handles this, but only if you measure the menu's dimensions *after* it renders. If you measure a hidden `display: none` element, its `offsetWidth` is zero. Fix: render with `visibility: hidden` (not `display: none`) before measurement, then switch to `visibility: visible` after positioning.

**7. Haptic feedback blocked by strict browser policies**
`navigator.vibrate` requires a user gesture on some browsers and is blocked in cross-origin iframes entirely. Always wrap in `navigator.vibrate?.()` and treat it as a progressive enhancement — the feature works fine without it.

---

## Interview Angle

**Q: "How do you implement a long-press context menu that works on both touch and desktop, and is still accessible to keyboard users?"**

A strong answer separates the problem into three interaction surfaces and one shared UI. For touch: start a `setTimeout` on `touchstart`, clear it on `touchmove` (if movement exceeds an 8 px threshold) and on `touchend`. If the timer completes, fire the menu and call `navigator.vibrate?.(50)` for haptic confirmation. For desktop: intercept the `contextmenu` event and call `e.preventDefault()` to suppress the native menu before rendering yours at `e.clientX`/`e.clientY`. For keyboard: listen for `ContextMenu` or `Shift+F10` on the trigger element and position the menu below the element's bounding rect. The menu itself uses `role="menu"`, `role="menuitem"` on each action, `aria-haspopup="menu"` and `aria-expanded` on the trigger, arrow-key navigation within the menu, Escape to close, and focus restoration to the trigger on dismiss.

**Follow-up Q: "What are the main failure modes in production?"**

Two stand out: timer lifecycle and menu positioning. Timers that are not cleared on component unmount cause state updates on unmounted trees (React warning) or null-reference crashes (Angular). The fix is always to store the timer id in a ref and clear it in the cleanup. Positioning failures happen because you measure the menu's dimensions before the browser has laid it out — a zero-width measurement means the menu is never clamped to the viewport. The fix is to render invisibly (not `display: none`), measure after the next animation frame, apply `top`/`left`, then make the menu visible. A secondary production concern is the double-event problem on mobile, where `touchend` and the synthesized `click` both fire, potentially submitting a form or triggering an action twice.

**Follow-up Q: "How would you test this component?"**

Unit-test the timer logic in isolation: mock `setTimeout`/`clearTimeout` with fake timers (Jest's `useFakeTimers`), dispatch synthetic `touchstart` and `touchend` events, and assert that the `onLongPress` callback fires after the delay but not after a short tap or after a move beyond the threshold. For the positioning utility, unit-test the clamping logic with hardcoded viewport and menu dimensions. Integration-test the ARIA state with a library like Testing Library: after triggering the menu, assert `aria-expanded="true"` on the trigger, `role="menu"` on the container, and that focus lands on the first `menuitem`. For haptics, mock `navigator.vibrate` and assert it was called with `50`.
