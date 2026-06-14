# Command Palette Accessibility

## The Idea

**In plain English:** A command palette is a modal search overlay — triggered by a keyboard shortcut like `Cmd+K` — that lets users type a query and instantly see a filtered list of actions, pages, or settings they can execute. Think VS Code's command palette or GitHub's jump-to-file dialog. The challenge: it looks simple (an input box + a list), but it sits at the intersection of four ARIA patterns — dialog, search, listbox, and live region — and getting those wrong means keyboard and screen-reader users either can't open it, can't navigate it, or never hear the results.

**Real-world analogy:** Think of a hotel concierge desk.

- The **concierge desk itself** is a dialog: it appears on demand, blocks access to the rest of the lobby until you are done, and the lobby is still there when you leave.
- The **"How can I help you?" prompt** is the search input: the concierge listens to whatever you say.
- The **list of suggestions the concierge reads back to you** ("I found three restaurants, a spa, and a taxi service") is the live region: it announces results the moment they change.
- **Each suggestion on the card in front of you** is a listbox option: you point at one to select it.
- When you leave, the concierge **escorts you back to exactly where you were standing** in the lobby — that is focus restoration to the trigger.

The key insight: a command palette is not a `<select>` and not a search form. It is a dialog that *contains* a combobox pattern, and every layer of that nesting carries its own ARIA contract.

---

## Learning Objectives

- Understand the trade-off between `role="dialog"` and `role="search"` as the palette root
- Know when `role="combobox"` belongs on the input versus `role="searchbox"`
- Correctly wire `role="listbox"` and `role="option"` for keyboard-navigable results
- Announce result counts without spamming the screen reader using `aria-live="polite"`
- Restore focus precisely to the trigger element that opened the palette
- Write a keyboard contract document that QA and design can test against
- Avoid the common mistakes: wrong live region placement, missing `aria-activedescendant`, broken focus restoration on route change

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Show/hide the overlay | ✅ with `visibility` + `opacity` | — |
| Filter the results list | ❌ | CSS cannot read input value or re-render DOM nodes |
| Track keyboard-selected option | ❌ | CSS has no concept of "active descendant" |
| Announce result count to screen reader | ❌ | `aria-live` content must be mutated by JS |
| Trap focus inside the dialog | ❌ | Requires `focusin` listeners and `querySelectorAll` |
| Restore focus to trigger on close | ❌ | Requires saving a reference before opening |
| Toggle `aria-expanded` on trigger | ❌ | ARIA state is an attribute; only JS can toggle it |
| Prevent background scroll | ❌ | `overflow: hidden` on `body` must be added/removed by JS |
| Close on backdrop click | ❌ | CSS has no click-outside detection |

**Conclusion:** CSS owns the visual layer (overlay backdrop, enter/exit animation, highlighted option background). JS owns everything else: filtering, keyboard navigation, ARIA state, focus management, and live announcements.

---

## HTML & CSS Foundation

### The Structure

```html
<!-- Trigger button — keyboard shortcut hint is decorative, not the accessible name -->
<button
  class="palette-trigger"
  aria-label="Open command palette"
  aria-keyshortcuts="Meta+K"
  aria-haspopup="dialog"
  aria-expanded="false"
  id="palette-trigger"
>
  <span aria-hidden="true">⌘K</span>
  <span class="palette-trigger__label">Search</span>
</button>

<!-- Backdrop -->
<div class="palette-backdrop" aria-hidden="true" id="palette-backdrop"></div>

<!--
  Root: role="dialog" wins over role="search" here.
  The palette IS a dialog first — it traps focus, has a close action,
  and must return focus on dismiss. role="search" is a landmark for
  a search *section* of the page, not a modal overlay.
  We nest aria-label to name the dialog.
-->
<div
  id="command-palette"
  class="palette"
  role="dialog"
  aria-label="Command palette"
  aria-modal="true"
  hidden
>
  <!-- Search input
       role="combobox" signals: "this input controls a popup list".
       aria-controls points to the listbox.
       aria-activedescendant tracks which option is visually highlighted.
       aria-autocomplete="list" = filtering happens, not inline completion. -->
  <div class="palette__search">
    <label for="palette-input" class="sr-only">Search commands</label>
    <input
      id="palette-input"
      class="palette__input"
      type="text"
      role="combobox"
      aria-label="Search commands"
      aria-autocomplete="list"
      aria-expanded="false"
      aria-controls="palette-listbox"
      aria-activedescendant=""
      autocomplete="off"
      spellcheck="false"
      placeholder="Search commands, files, settings…"
    />
  </div>

  <!-- Live region: announces result count.
       Placed OUTSIDE the listbox so screen readers don't
       re-read the entire list when the count updates.
       aria-live="polite" = waits for the user to finish typing.
       aria-atomic="true" = read the whole string, not just the diff. -->
  <p
    id="palette-status"
    class="sr-only"
    role="status"
    aria-live="polite"
    aria-atomic="true"
  ></p>

  <!-- Results list -->
  <ul
    id="palette-listbox"
    class="palette__results"
    role="listbox"
    aria-label="Results"
  >
    <!-- Each option:
         id is required — aria-activedescendant points to it.
         aria-selected="true" marks the currently highlighted option.
         aria-describedby can point to a group label if results are grouped. -->
    <li
      id="palette-option-0"
      class="palette__option"
      role="option"
      aria-selected="false"
    >
      <span class="palette__option-icon" aria-hidden="true">⚙️</span>
      <span class="palette__option-label">Open Settings</span>
      <kbd class="palette__option-shortcut" aria-hidden="true">⌘,</kbd>
    </li>
  </ul>

  <!-- Empty state (conditionally rendered) -->
  <p class="palette__empty" hidden aria-live="polite">
    No results for "<span id="palette-empty-query"></span>"
  </p>
</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --palette-max-width: 640px;
  --palette-radius: 12px;
  --palette-shadow: 0 24px 64px rgba(0, 0, 0, 0.35);
  --palette-bg: #1e1e2e;
  --palette-input-bg: #2a2a3d;
  --palette-border: #3d3d56;
  --palette-option-hover: #2e2e45;
  --palette-option-selected: #3c3c5a;
  --palette-text: #e0e0f0;
  --palette-text-dim: #888;
  --palette-accent: #7c6af7;
  --palette-transition: 0.18s cubic-bezier(0.4, 0, 0.2, 1);
}

/* ─── Backdrop ─── */
.palette-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.6);
  backdrop-filter: blur(4px);
  opacity: 0;
  visibility: hidden;
  transition: opacity var(--palette-transition), visibility var(--palette-transition);
  z-index: 900;
}

.palette-backdrop.is-visible {
  opacity: 1;
  visibility: visible;
}

/* ─── Dialog container ─── */
.palette {
  position: fixed;
  top: 15%;
  left: 50%;
  transform: translateX(-50%) translateY(-12px);
  width: min(var(--palette-max-width), calc(100vw - 2rem));
  max-height: 70vh;
  display: flex;
  flex-direction: column;
  background: var(--palette-bg);
  border: 1px solid var(--palette-border);
  border-radius: var(--palette-radius);
  box-shadow: var(--palette-shadow);
  opacity: 0;
  visibility: hidden;
  transition: opacity var(--palette-transition),
              transform var(--palette-transition),
              visibility var(--palette-transition);
  z-index: 901;
  overflow: hidden;
}

.palette.is-open {
  opacity: 1;
  visibility: visible;
  transform: translateX(-50%) translateY(0);
}

/* ─── Search area ─── */
.palette__search {
  display: flex;
  align-items: center;
  padding: 0 1rem;
  border-bottom: 1px solid var(--palette-border);
  flex-shrink: 0;
}

.palette__input {
  flex: 1;
  height: 56px;
  background: none;
  border: none;
  outline: none;
  color: var(--palette-text);
  font-size: 1.1rem;
  caret-color: var(--palette-accent);
}

.palette__input::placeholder {
  color: var(--palette-text-dim);
}

/* ─── Results list ─── */
.palette__results {
  overflow-y: auto;
  max-height: calc(70vh - 57px);  /* total height minus input row */
  padding: 0.5rem 0;
  margin: 0;
  list-style: none;
  scroll-padding: 0.5rem;  /* keeps options from hiding behind scrollbar edge */
}

/* ─── Option ─── */
.palette__option {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.6rem 1rem;
  cursor: pointer;
  color: var(--palette-text);
  border-radius: 6px;
  margin: 0 0.375rem;
  /* No :hover rule — keyboard selection drives background, not mouse hover */
}

/* JS adds aria-selected="true"; CSS reads it */
.palette__option[aria-selected="true"] {
  background: var(--palette-option-selected);
  outline: 2px solid var(--palette-accent);
  outline-offset: -2px;
}

.palette__option:hover {
  background: var(--palette-option-hover);
}

.palette__option-label {
  flex: 1;
  font-size: 0.95rem;
}

.palette__option-shortcut {
  font-size: 0.75rem;
  color: var(--palette-text-dim);
  background: rgba(255, 255, 255, 0.08);
  padding: 2px 6px;
  border-radius: 4px;
}

/* ─── Empty state ─── */
.palette__empty {
  padding: 2rem;
  text-align: center;
  color: var(--palette-text-dim);
  font-size: 0.9rem;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .palette,
  .palette-backdrop {
    transition: none;
  }
}
```

**What CSS owns:** dialog appearance/disappearance animation, option highlight via `aria-selected`, backdrop blur, reduced-motion fallback.

**What CSS cannot own:** filtering the results list, tracking which option is highlighted, announcing result counts, focus management, closing on Escape or backdrop click.

---

## React Implementation

### Types and Data

```tsx
// command-palette.types.ts
export interface Command {
  id: string;
  label: string;
  description?: string;
  shortcut?: string;
  icon?: string;
  group?: string;
  action: () => void;
}
```

### The Hook — All Palette State

```tsx
// useCommandPalette.ts
import {
  useState, useCallback, useRef, useEffect, useMemo
} from 'react';
import type { Command } from './command-palette.types';

export function useCommandPalette(commands: Command[]) {
  const [isOpen, setIsOpen]       = useState(false);
  const [query, setQuery]         = useState('');
  const [activeIndex, setActive]  = useState(-1);

  // The element that triggered the open — restored on close
  const triggerRef  = useRef<HTMLElement | null>(null);
  const inputRef    = useRef<HTMLInputElement>(null);

  // Filter commands synchronously; for large sets, debounce or use a worker
  const results = useMemo(() => {
    if (!query.trim()) return commands;
    const q = query.toLowerCase();
    return commands.filter(
      c =>
        c.label.toLowerCase().includes(q) ||
        c.description?.toLowerCase().includes(q)
    );
  }, [query, commands]);

  const open = useCallback((trigger?: HTMLElement) => {
    triggerRef.current = trigger ?? (document.activeElement as HTMLElement);
    setIsOpen(true);
    setQuery('');
    setActive(-1);
  }, []);

  const close = useCallback(() => {
    setIsOpen(false);
    // Restore focus AFTER the dialog is gone from the accessibility tree
    // Use rAF so the hidden attribute is applied first
    requestAnimationFrame(() => {
      triggerRef.current?.focus();
    });
  }, []);

  const select = useCallback(
    (index: number) => {
      const cmd = results[index];
      if (!cmd) return;
      close();
      // Execute after focus has restored — avoids action firing in wrong context
      setTimeout(() => cmd.action(), 0);
    },
    [results, close]
  );

  // Keyboard contract — documented at the bottom of this file
  const handleKeyDown = useCallback(
    (e: React.KeyboardEvent) => {
      switch (e.key) {
        case 'ArrowDown': {
          e.preventDefault();
          setActive(i => Math.min(i + 1, results.length - 1));
          break;
        }
        case 'ArrowUp': {
          e.preventDefault();
          setActive(i => Math.max(i - 1, 0));
          break;
        }
        case 'Home': {
          e.preventDefault();
          setActive(0);
          break;
        }
        case 'End': {
          e.preventDefault();
          setActive(results.length - 1);
          break;
        }
        case 'Enter': {
          e.preventDefault();
          if (activeIndex >= 0) select(activeIndex);
          break;
        }
        case 'Escape': {
          e.preventDefault();
          close();
          break;
        }
      }
    },
    [results.length, activeIndex, select, close]
  );

  // Focus input when palette opens
  useEffect(() => {
    if (isOpen) {
      requestAnimationFrame(() => inputRef.current?.focus());
    }
  }, [isOpen]);

  // Global Cmd+K / Ctrl+K shortcut
  useEffect(() => {
    const onKey = (e: KeyboardEvent) => {
      if ((e.metaKey || e.ctrlKey) && e.key === 'k') {
        e.preventDefault();
        isOpen ? close() : open();
      }
    };
    document.addEventListener('keydown', onKey);
    return () => document.removeEventListener('keydown', onKey);
  }, [isOpen, open, close]);

  // Scroll active option into view
  useEffect(() => {
    if (activeIndex < 0) return;
    const el = document.getElementById(`palette-option-${activeIndex}`);
    el?.scrollIntoView({ block: 'nearest' });
  }, [activeIndex]);

  return {
    isOpen, query, setQuery, results, activeIndex,
    open, close, select, handleKeyDown, inputRef
  };
}
```

### The Component

```tsx
// CommandPalette.tsx
import { useCallback } from 'react';
import { useCommandPalette } from './useCommandPalette';
import type { Command } from './command-palette.types';

interface Props {
  commands: Command[];
}

export function CommandPalette({ commands }: Props) {
  const {
    isOpen, query, setQuery, results, activeIndex,
    open, close, select, handleKeyDown, inputRef
  } = useCommandPalette(commands);

  const activeId = activeIndex >= 0 ? `palette-option-${activeIndex}` : undefined;

  // Result count announcement — update AFTER results render
  // Using a separate string so aria-atomic reads the full sentence
  const statusText = isOpen
    ? results.length === 0
      ? `No results for "${query}"`
      : `${results.length} result${results.length !== 1 ? 's' : ''}`
    : '';

  return (
    <>
      {/* Global trigger button */}
      <button
        className="palette-trigger"
        aria-label="Open command palette"
        aria-keyshortcuts="Meta+K"
        aria-haspopup="dialog"
        aria-expanded={isOpen}
        onClick={() => open()}
      >
        <span aria-hidden="true">⌘K</span>
        <span className="palette-trigger__label">Search</span>
      </button>

      {/* Backdrop */}
      <div
        className={`palette-backdrop ${isOpen ? 'is-visible' : ''}`}
        aria-hidden="true"
        onClick={close}
      />

      {/* Dialog */}
      <div
        id="command-palette"
        className={`palette ${isOpen ? 'is-open' : ''}`}
        role="dialog"
        aria-label="Command palette"
        aria-modal="true"
        hidden={!isOpen}
      >
        {/* Search input */}
        <div className="palette__search">
          <label htmlFor="palette-input" className="sr-only">
            Search commands
          </label>
          <input
            ref={inputRef}
            id="palette-input"
            className="palette__input"
            type="text"
            role="combobox"
            aria-label="Search commands"
            aria-autocomplete="list"
            aria-expanded={results.length > 0}
            aria-controls="palette-listbox"
            aria-activedescendant={activeId}
            autoComplete="off"
            spellCheck={false}
            placeholder="Search commands, files, settings…"
            value={query}
            onChange={e => setQuery(e.target.value)}
            onKeyDown={handleKeyDown}
          />
        </div>

        {/* Live region — outside the listbox so it doesn't re-read results */}
        <p
          id="palette-status"
          className="sr-only"
          role="status"
          aria-live="polite"
          aria-atomic="true"
        >
          {statusText}
        </p>

        {/* Results */}
        <ul
          id="palette-listbox"
          className="palette__results"
          role="listbox"
          aria-label="Results"
        >
          {results.map((cmd, i) => (
            <li
              key={cmd.id}
              id={`palette-option-${i}`}
              className="palette__option"
              role="option"
              aria-selected={i === activeIndex}
              onClick={() => select(i)}
              // onMouseEnter previews but does not commit
              onMouseEnter={() => {/* intentionally empty */}}
            >
              {cmd.icon && (
                <span className="palette__option-icon" aria-hidden="true">
                  {cmd.icon}
                </span>
              )}
              <span className="palette__option-label">{cmd.label}</span>
              {cmd.shortcut && (
                <kbd className="palette__option-shortcut" aria-hidden="true">
                  {cmd.shortcut}
                </kbd>
              )}
            </li>
          ))}
        </ul>

        {/* Empty state */}
        {query && results.length === 0 && (
          <p className="palette__empty">
            No results for &ldquo;{query}&rdquo;
          </p>
        )}
      </div>
    </>
  );
}
```

---

## Angular Implementation

### Service

```typescript
// command-palette.service.ts
import { Injectable, signal, computed } from '@angular/core';
import { Command } from './command-palette.types';

@Injectable({ providedIn: 'root' })
export class CommandPaletteService {
  private _isOpen  = signal(false);
  private _query   = signal('');
  private _active  = signal(-1);
  private _cmds    = signal<Command[]>([]);
  private _trigger: HTMLElement | null = null;

  readonly isOpen  = this._isOpen.asReadonly();
  readonly query   = this._query.asReadonly();
  readonly active  = this._active.asReadonly();

  readonly results = computed(() => {
    const q = this._query().toLowerCase().trim();
    const all = this._cmds();
    if (!q) return all;
    return all.filter(
      c =>
        c.label.toLowerCase().includes(q) ||
        c.description?.toLowerCase().includes(q)
    );
  });

  readonly statusText = computed(() => {
    if (!this._isOpen()) return '';
    const count = this.results().length;
    const q = this._query();
    if (count === 0 && q) return `No results for "${q}"`;
    return `${count} result${count !== 1 ? 's' : ''}`;
  });

  readonly activeId = computed(() => {
    const i = this._active();
    return i >= 0 ? `palette-option-${i}` : '';
  });

  register(commands: Command[]) {
    this._cmds.set(commands);
  }

  open(trigger?: HTMLElement) {
    this._trigger = trigger ?? (document.activeElement as HTMLElement);
    this._isOpen.set(true);
    this._query.set('');
    this._active.set(-1);
  }

  close() {
    this._isOpen.set(false);
    requestAnimationFrame(() => this._trigger?.focus());
  }

  setQuery(q: string) {
    this._query.set(q);
    this._active.set(-1);   // reset selection on new query
  }

  moveDown() {
    this._active.update(i => Math.min(i + 1, this.results().length - 1));
  }

  moveUp() {
    this._active.update(i => Math.max(i - 1, 0));
  }

  selectActive() {
    const cmd = this.results()[this._active()];
    if (!cmd) return;
    this.close();
    setTimeout(() => cmd.action(), 0);
  }
}
```

### Component

```typescript
// command-palette.component.ts
import {
  Component, inject, HostListener, ElementRef,
  OnInit, OnDestroy, effect, ViewChild
} from '@angular/core';
import { NgFor, NgIf } from '@angular/common';
import { CommandPaletteService } from './command-palette.service';

@Component({
  selector: 'app-command-palette',
  standalone: true,
  imports: [NgFor, NgIf],
  template: `
    <!-- Trigger -->
    <button
      class="palette-trigger"
      aria-label="Open command palette"
      aria-keyshortcuts="Meta+K"
      aria-haspopup="dialog"
      [attr.aria-expanded]="svc.isOpen()"
      (click)="svc.open()"
    >
      <span aria-hidden="true">⌘K</span>
      <span class="palette-trigger__label">Search</span>
    </button>

    <!-- Backdrop -->
    <div
      class="palette-backdrop"
      [class.is-visible]="svc.isOpen()"
      aria-hidden="true"
      (click)="svc.close()"
    ></div>

    <!-- Dialog -->
    <div
      #paletteEl
      id="command-palette"
      class="palette"
      [class.is-open]="svc.isOpen()"
      role="dialog"
      aria-label="Command palette"
      aria-modal="true"
      [attr.hidden]="svc.isOpen() ? null : true"
    >
      <!-- Input -->
      <div class="palette__search">
        <label for="palette-input" class="sr-only">Search commands</label>
        <input
          #inputEl
          id="palette-input"
          class="palette__input"
          type="text"
          role="combobox"
          aria-label="Search commands"
          aria-autocomplete="list"
          [attr.aria-expanded]="svc.results().length > 0"
          aria-controls="palette-listbox"
          [attr.aria-activedescendant]="svc.activeId()"
          autocomplete="off"
          spellcheck="false"
          placeholder="Search commands, files, settings…"
          [value]="svc.query()"
          (input)="svc.setQuery($any($event.target).value)"
          (keydown)="onKeyDown($event)"
        />
      </div>

      <!-- Live region -->
      <p
        class="sr-only"
        role="status"
        aria-live="polite"
        aria-atomic="true"
      >{{ svc.statusText() }}</p>

      <!-- Results -->
      <ul
        id="palette-listbox"
        class="palette__results"
        role="listbox"
        aria-label="Results"
      >
        <li
          *ngFor="let cmd of svc.results(); let i = index; trackBy: trackById"
          [id]="'palette-option-' + i"
          class="palette__option"
          role="option"
          [attr.aria-selected]="i === svc.active()"
          (click)="selectAt(i)"
        >
          <span *ngIf="cmd.icon" class="palette__option-icon" aria-hidden="true">
            {{ cmd.icon }}
          </span>
          <span class="palette__option-label">{{ cmd.label }}</span>
          <kbd *ngIf="cmd.shortcut" class="palette__option-shortcut" aria-hidden="true">
            {{ cmd.shortcut }}
          </kbd>
        </li>
      </ul>

      <!-- Empty state -->
      <p
        *ngIf="svc.query() && svc.results().length === 0"
        class="palette__empty"
      >
        No results for "{{ svc.query() }}"
      </p>
    </div>
  `,
})
export class CommandPaletteComponent implements OnInit, OnDestroy {
  svc = inject(CommandPaletteService);

  @ViewChild('inputEl') inputEl!: ElementRef<HTMLInputElement>;

  constructor() {
    effect(() => {
      if (this.svc.isOpen()) {
        requestAnimationFrame(() => this.inputEl?.nativeElement.focus());
      }
    });

    // Scroll active option into view
    effect(() => {
      const i = this.svc.active();
      if (i < 0) return;
      const el = document.getElementById(`palette-option-${i}`);
      el?.scrollIntoView({ block: 'nearest' });
    });
  }

  ngOnInit() {}
  ngOnDestroy() {}

  // Global shortcut
  @HostListener('document:keydown', ['$event'])
  onGlobal(e: KeyboardEvent) {
    if ((e.metaKey || e.ctrlKey) && e.key === 'k') {
      e.preventDefault();
      this.svc.isOpen() ? this.svc.close() : this.svc.open();
    }
  }

  onKeyDown(e: KeyboardEvent) {
    switch (e.key) {
      case 'ArrowDown': e.preventDefault(); this.svc.moveDown(); break;
      case 'ArrowUp':   e.preventDefault(); this.svc.moveUp();   break;
      case 'Home':      e.preventDefault(); this.svc['_active'].set(0); break;
      case 'End': {
        e.preventDefault();
        this.svc['_active'].set(this.svc.results().length - 1);
        break;
      }
      case 'Enter':  e.preventDefault(); this.svc.selectActive(); break;
      case 'Escape': e.preventDefault(); this.svc.close();        break;
    }
  }

  selectAt(i: number) {
    this.svc['_active'].set(i);
    this.svc.selectActive();
  }

  trackById(_: number, cmd: { id: string }) {
    return cmd.id;
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Dialog role on root container | `role="dialog"` with `aria-modal="true"` and `aria-label` |
| Input is a combobox, not a plain text field | `role="combobox"` + `aria-autocomplete="list"` + `aria-controls` pointing to listbox |
| Results use listbox + option roles | `role="listbox"` on `<ul>`, `role="option"` on each `<li>` |
| Active option signalled without moving focus | `aria-activedescendant` on the input, updated on arrow keys |
| `aria-selected` tracks highlighted option | Set to `"true"` on active item; CSS reads attribute for highlight |
| Result count announced without reading all items | Separate `role="status"` + `aria-live="polite"` outside the listbox |
| Live region is atomic | `aria-atomic="true"` prevents partial reads during rapid typing |
| Escape closes the palette | `keydown` handler; does not require focus to be on the backdrop |
| Focus moves to input on open | `requestAnimationFrame` after `hidden` attribute is removed |
| Focus restored to trigger on close | Trigger element reference saved before open; restored via `rAF` |
| Trigger button declares shortcut | `aria-keyshortcuts="Meta+K"` on the trigger button |
| Trigger button has `aria-haspopup="dialog"` | Tells screen reader that activating opens a dialog |
| Background content inert while open | `aria-modal="true"` + `hidden` prevents background traversal in supporting browsers |
| Keyboard shortcut display is decorative | `⌘K` span has `aria-hidden="true"` |
| Empty state announced | Rendered into the DOM; screen reader reads it as part of dialog content |

---

## Production Pitfalls

**1. Placing the live region inside the listbox**
`aria-live` regions inside `role="listbox"` are ignored by some screen readers because listbox has its own announcement contract. The result count is never spoken. Fix: place the `role="status"` element as a sibling of the listbox, not a child of it.

**2. Using `aria-live="assertive"` for result counts**
Assertive interrupts whatever the screen reader is saying mid-sentence. For a palette that filters on every keystroke, this creates a cacophony of interrupted announcements. Fix: use `aria-live="polite"` always. If you need urgency (e.g. a critical error state), assertive is appropriate, but not for result counts.

**3. Forgetting `aria-activedescendant` — relying on `aria-selected` alone**
`aria-selected` marks which option is selected, but without `aria-activedescendant` on the input, a screen reader in browse mode does not know which option corresponds to the current keyboard position. VoiceOver on Safari will be silent as the user presses arrow keys. Fix: always keep `aria-activedescendant` in sync with `activeIndex` on the combobox input.

**4. Focus restoration race condition**
If you call `trigger.focus()` synchronously inside the close handler, the dialog may still be in the accessibility tree when focus moves. AT software then announces the dialog content again. Fix: wrap the focus restoration in `requestAnimationFrame` so the `hidden` attribute is applied in the same paint frame before focus shifts.

**5. `aria-modal` is not a focus trap**
`aria-modal="true"` tells AT software to treat background content as inert — but only in supporting browsers (Chrome + NVDA, Safari + VoiceOver handle this well; older combinations do not). Tab key still moves focus outside on unsupported combinations. Fix: always implement a programmatic focus trap with `querySelectorAll` and wrap-around Tab handling, regardless of `aria-modal`.

**6. IDs shift when results re-render**
If you use index-based IDs (`palette-option-0`) and the results list re-renders completely on each keystroke, `aria-activedescendant` may briefly point to the wrong element. Fix: use stable IDs tied to command `id` (e.g. `palette-option-${cmd.id}`) and update `aria-activedescendant` after the DOM has updated, not during the state setter.

**7. Missing keyboard contract documentation**
Designers change the palette UI without knowing which keys are contractual. Undocumented keyboard behaviour gets removed in the next sprint. Fix: keep a keyboard contract comment in the component file and a corresponding QA checklist — this is the contract:

```
Keyboard Contract — Command Palette
────────────────────────────────────
Cmd/Ctrl+K     Open or close the palette (global, works anywhere on page)
Escape         Close the palette; focus returns to trigger
ArrowDown      Move selection to next result; wraps at end: no (stops)
ArrowUp        Move selection to previous result; wraps at start: no (stops)
Home           Move selection to first result
End            Move selection to last result
Enter          Execute the highlighted command and close
Tab            Moves focus out of dialog (intentional; no tab-trap inside palette)
Shift+Tab      Moves focus backwards; if on input, stays on input (no items behind it)
Click result   Same as Enter on that result
Click backdrop Close the palette; focus returns to trigger
```

---

## Interview Angle

**Q: "Should a command palette use `role="dialog"` or `role="search"` on its container?"**

`role="dialog"` is the right choice for any command palette that: appears in a modal overlay, traps focus (or manages it with `aria-modal`), and returns focus on close. `role="search"` is a landmark role for a search section that is permanently part of the page layout — like a site-wide search bar in the header. Using `role="search"` on a modal overlay means the screen reader announces "search landmark" and does not treat the overlay as blocking background content, so the user can still Tab into the page behind it. The practical consequence: a screen-reader user hears "search" and navigates past your overlay. `role="dialog"` with `aria-modal="true"` gives the correct semantics and, in supporting AT, hides the background. You can still add `aria-label="Search commands"` to the input inside the dialog for search semantics without misusing the landmark role.

**Follow-up: "Why can't you just use `aria-live="assertive"` to make the result count announcement more immediate?"**

Because a command palette filters on every keystroke. With `aria-live="assertive"`, each intermediate result count — "12 results", "8 results", "3 results", "1 result" — interrupts the screen reader mid-word. The user cannot finish typing before AT reads "12 results" over their input echo. `aria-live="polite"` waits for the user to pause, then announces the stable count. The experience goes from noise to a single, useful update. Assertive is the right choice only when the announcement is truly urgent and time-sensitive, such as a session-timeout warning or a form submission error that blocks progress.
