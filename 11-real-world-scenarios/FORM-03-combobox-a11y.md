# Combobox Accessibility Deep Dive

## The Idea

**In plain English:** A combobox is a text input fused with a popup list. The user types to filter a set of options, uses arrow keys to highlight a choice, presses Enter to confirm it, and presses Escape to abandon the popup and return to typing. Unlike a native `<select>`, a combobox lets users type arbitrary text AND pick from suggestions — making it the right control for city search, tag pickers, command palettes, and any "type to find" scenario. Because it is not a native HTML element, every interaction — keyboard navigation, focus management, screen-reader state — must be wired by hand using the ARIA combobox pattern.

**Real-world analogy:** Think of a concierge at a hotel desk.

- The **text input** is your mouth — you say "I'm looking for a restaurant…"
- The **popup list** is the concierge's notepad that appears on the desk — a filtered shortlist based on what you just said.
- **Typing more** narrows the notepad to fewer options.
- **Pressing the down arrow** means the concierge moves a finger down the list to highlight the next option — your voice (focus) stays with the concierge, not the notepad.
- **Pressing Enter** means you say "that one" — the concierge writes the choice on your key card and puts the notepad away.
- **Pressing Escape** means "never mind" — the notepad disappears and you can keep talking (typing) freely.
- The critical insight: **focus never physically moves to the list**. The combobox fakes the appearance of list focus using `aria-activedescendant` while DOM focus stays on the input at all times.

---

## Learning Objectives

- Understand the full ARIA combobox pattern: `role="combobox"`, `aria-expanded`, `role="listbox"`, `role="option"`, `aria-activedescendant`
- Know when to use `aria-activedescendant` versus roving `tabindex` and why combobox favors the former
- Implement the complete keyboard contract: type to filter, Arrow keys to navigate, Enter to select, Escape to dismiss
- Handle `aria-selected` correctly on options and announce the active option to screen readers
- Understand why mobile screen readers (TalkBack, VoiceOver iOS) behave differently and how to compensate
- Build a fully accessible combobox in React with TypeScript hooks
- Build the equivalent in Angular with signals and standalone components
- Recognize the production pitfalls that make comboboxes one of the most frequently broken ARIA patterns

---

## Why CSS Alone Isn't Enough

A `<select>` gives you filtering and keyboard navigation for free. A combobox abandons native controls in exchange for custom styling and richer interaction — which means every accessibility affordance must be rebuilt.

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Show/hide the popup list | ✅ with `:focus` + sibling selector | — |
| Filter options as the user types | ❌ | CSS cannot read input value or hide matching items |
| Track which option is currently highlighted | ❌ | CSS has no concept of `aria-activedescendant` |
| Announce the highlighted option to screen readers | ❌ | Requires `aria-activedescendant` attribute update via JS |
| Select an option on Enter | ❌ | CSS cannot intercept keyboard events |
| Close on Escape and restore prior value | ❌ | Requires JS state for prior input value |
| Close when focus moves outside the widget | ❌ | Requires `focusout`/`blur` listeners |
| Mark the selected option with `aria-selected="true"` | ❌ | Attribute must be toggled by JS |
| Scroll the list so the active option stays visible | ❌ | Requires `scrollIntoView` or manual scroll calculation |
| Position the popup relative to the input | ⚠️ | CSS can position but cannot flip to top when near viewport bottom |

**Conclusion:** CSS can style the popup's shape and the highlighted option's background. JS owns every interactive, stateful, and accessible behavior.

---

## HTML & CSS Foundation

### The Structure

```html
<!--
  The ARIA combobox pattern (APG 1.2):
  - The INPUT has role="combobox" and owns the listbox via aria-controls
  - aria-expanded reflects whether the popup is open
  - aria-activedescendant points to the id of the currently highlighted option
  - The popup UL has role="listbox"
  - Each LI has role="option" and a unique id
-->
<div class="combobox-wrapper">
  <label for="city-input" id="city-label">City</label>

  <div class="combobox-control">
    <input
      id="city-input"
      type="text"
      role="combobox"
      autocomplete="off"
      autocorrect="off"
      autocapitalize="none"
      spellcheck="false"
      aria-labelledby="city-label"
      aria-autocomplete="list"
      aria-expanded="false"
      aria-controls="city-listbox"
      aria-activedescendant=""
      class="combobox-input"
    />
    <!-- Toggle button: optional but helps pointer users -->
    <button
      class="combobox-toggle"
      tabindex="-1"
      aria-label="Show suggestions"
      aria-expanded="false"
      aria-controls="city-listbox"
    >
      <span class="combobox-caret" aria-hidden="true">▾</span>
    </button>
  </div>

  <ul
    id="city-listbox"
    role="listbox"
    aria-labelledby="city-label"
    class="combobox-listbox"
    hidden
  >
    <!-- Populated by JS -->
    <li
      id="city-opt-0"
      role="option"
      aria-selected="false"
      class="combobox-option"
    >
      New York
    </li>
    <li
      id="city-opt-1"
      role="option"
      aria-selected="false"
      class="combobox-option combobox-option--active"
    >
      Newark
    </li>
  </ul>
</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --cb-border: 1px solid #767676;
  --cb-border-focus: 2px solid #0057b8;
  --cb-radius: 4px;
  --cb-option-height: 40px;
  --cb-active-bg: #0057b8;
  --cb-active-color: #ffffff;
  --cb-hover-bg: #e8f0fe;
  --cb-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  --cb-max-height: 280px;
}

/* ─── Wrapper ─── */
.combobox-wrapper {
  position: relative;
  display: inline-flex;
  flex-direction: column;
  gap: 4px;
  width: 320px;
}

/* ─── Control row (input + toggle button) ─── */
.combobox-control {
  display: flex;
  align-items: stretch;
  border: var(--cb-border);
  border-radius: var(--cb-radius);
  background: #fff;
  overflow: hidden;   /* clip the button edge within the rounded border */
}

.combobox-control:focus-within {
  outline: var(--cb-border-focus);
  outline-offset: 2px;
}

.combobox-input {
  flex: 1;
  min-height: 44px;
  padding: 0 0.75rem;
  border: none;
  background: transparent;
  font-size: 1rem;
  color: inherit;
  outline: none;      /* outline is on .combobox-control via :focus-within */
}

.combobox-toggle {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 36px;
  flex-shrink: 0;
  background: none;
  border: none;
  border-left: var(--cb-border);
  cursor: pointer;
  color: #555;
}

.combobox-toggle:hover {
  background: var(--cb-hover-bg);
}

/* Rotate caret when expanded */
.combobox-input[aria-expanded="true"] ~ .combobox-toggle .combobox-caret,
.combobox-toggle[aria-expanded="true"] .combobox-caret {
  display: inline-block;
  transform: rotate(180deg);
}

/* ─── Popup listbox ─── */
.combobox-listbox {
  position: absolute;
  top: calc(100% + 4px);
  left: 0;
  right: 0;
  max-height: var(--cb-max-height);
  overflow-y: auto;
  background: #fff;
  border: var(--cb-border);
  border-radius: var(--cb-radius);
  box-shadow: var(--cb-shadow);
  list-style: none;
  margin: 0;
  padding: 4px 0;
  z-index: 500;
  /* scroll-snap keeps active option fully visible on keyboard nav */
  scroll-snap-type: y mandatory;
}

.combobox-listbox[hidden] {
  display: none;
}

/* ─── Options ─── */
.combobox-option {
  display: flex;
  align-items: center;
  min-height: var(--cb-option-height);
  padding: 0 0.75rem;
  font-size: 1rem;
  cursor: pointer;
  scroll-snap-align: nearest;
}

.combobox-option:hover {
  background: var(--cb-hover-bg);
}

/* Active option (aria-activedescendant target) */
.combobox-option--active {
  background: var(--cb-active-bg);
  color: var(--cb-active-color);
}

/* ─── Empty state ─── */
.combobox-empty {
  padding: 0.75rem;
  color: #767676;
  font-size: 0.875rem;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .combobox-caret {
    transition: none;
  }
}
```

**What CSS owns:** border, focus outline via `:focus-within`, listbox positioning, option highlight background, caret rotation, scroll containment.

**What CSS cannot own:** which option is active, filtering, keyboard handling, `aria-activedescendant` updates, `aria-expanded` toggling, scroll-into-view on keyboard navigation.

---

## React Implementation

### The Hook — Combobox State

```tsx
// useCombobox.ts
import { useState, useCallback, useRef, useId } from 'react';

export interface ComboboxOption {
  value: string;
  label: string;
}

interface UseComboboxOptions {
  options: ComboboxOption[];
  onSelect?: (option: ComboboxOption) => void;
}

export function useCombobox({ options, onSelect }: UseComboboxOptions) {
  const listboxId  = useId();
  const optionIdPrefix = `${listboxId}-opt`;

  const [inputValue, setInputValue]     = useState('');
  const [isOpen, setIsOpen]             = useState(false);
  const [activeIndex, setActiveIndex]   = useState<number>(-1);
  const [selectedValue, setSelectedValue] = useState<string | null>(null);

  // Filtered options derived from inputValue
  const filtered = options.filter(o =>
    o.label.toLowerCase().includes(inputValue.toLowerCase())
  );

  const activeDescendant =
    activeIndex >= 0 && filtered[activeIndex]
      ? `${optionIdPrefix}-${activeIndex}`
      : undefined;

  const open = useCallback(() => {
    setIsOpen(true);
    setActiveIndex(-1);
  }, []);

  const close = useCallback(() => {
    setIsOpen(false);
    setActiveIndex(-1);
  }, []);

  const selectOption = useCallback((option: ComboboxOption) => {
    setInputValue(option.label);
    setSelectedValue(option.value);
    onSelect?.(option);
    close();
  }, [onSelect, close]);

  const handleInputChange = useCallback((value: string) => {
    setInputValue(value);
    setIsOpen(true);
    setActiveIndex(-1);       // reset active when the filter changes
    setSelectedValue(null);   // clear selection on new input
  }, []);

  const handleKeyDown = useCallback((e: React.KeyboardEvent) => {
    switch (e.key) {
      case 'ArrowDown': {
        e.preventDefault();
        if (!isOpen) { open(); return; }
        setActiveIndex(i => Math.min(i + 1, filtered.length - 1));
        break;
      }
      case 'ArrowUp': {
        e.preventDefault();
        if (!isOpen) return;
        setActiveIndex(i => Math.max(i - 1, -1));
        break;
      }
      case 'Enter': {
        if (isOpen && activeIndex >= 0 && filtered[activeIndex]) {
          e.preventDefault();
          selectOption(filtered[activeIndex]);
        }
        break;
      }
      case 'Escape': {
        e.preventDefault();
        if (isOpen) {
          close();
        } else {
          // Second Escape clears input (matches browser native behaviour)
          setInputValue('');
          setSelectedValue(null);
        }
        break;
      }
      case 'Tab': {
        // Accept the active option on Tab (common UX convention)
        if (isOpen && activeIndex >= 0 && filtered[activeIndex]) {
          selectOption(filtered[activeIndex]);
        } else {
          close();
        }
        break;
      }
      case 'Home': {
        if (isOpen) { e.preventDefault(); setActiveIndex(0); }
        break;
      }
      case 'End': {
        if (isOpen) { e.preventDefault(); setActiveIndex(filtered.length - 1); }
        break;
      }
    }
  }, [isOpen, activeIndex, filtered, open, close, selectOption]);

  return {
    inputValue,
    isOpen,
    activeIndex,
    activeDescendant,
    filtered,
    selectedValue,
    listboxId,
    optionIdPrefix,
    handleInputChange,
    handleKeyDown,
    open,
    close,
    selectOption,
  };
}
```

### The Component

```tsx
// Combobox.tsx
import { useRef, useEffect } from 'react';
import { useCombobox, ComboboxOption } from './useCombobox';

interface Props {
  label: string;
  options: ComboboxOption[];
  onSelect?: (option: ComboboxOption) => void;
  placeholder?: string;
}

export function Combobox({ label, options, onSelect, placeholder }: Props) {
  const {
    inputValue, isOpen, activeIndex, activeDescendant,
    filtered, listboxId, optionIdPrefix,
    handleInputChange, handleKeyDown, open, close, selectOption,
  } = useCombobox({ options, onSelect });

  const inputRef  = useRef<HTMLInputElement>(null);
  const listRef   = useRef<HTMLUListElement>(null);
  const labelId   = `${listboxId}-label`;

  // Scroll active option into view when navigating by keyboard
  useEffect(() => {
    if (!listRef.current || activeIndex < 0) return;
    const activeEl = listRef.current.querySelector<HTMLElement>(
      `[id="${optionIdPrefix}-${activeIndex}"]`
    );
    activeEl?.scrollIntoView({ block: 'nearest' });
  }, [activeIndex, optionIdPrefix]);

  // Close when focus moves outside the widget
  const handleBlur = (e: React.FocusEvent) => {
    // relatedTarget is the element receiving focus
    const wrapper = e.currentTarget as HTMLElement;
    if (!wrapper.contains(e.relatedTarget as Node)) {
      close();
    }
  };

  return (
    <div
      className="combobox-wrapper"
      onBlur={handleBlur}
    >
      <label id={labelId} htmlFor={`${listboxId}-input`}>
        {label}
      </label>

      <div className="combobox-control">
        <input
          ref={inputRef}
          id={`${listboxId}-input`}
          type="text"
          role="combobox"
          autoComplete="off"
          autoCorrect="off"
          autoCapitalize="none"
          spellCheck={false}
          placeholder={placeholder}
          aria-labelledby={labelId}
          aria-autocomplete="list"
          aria-expanded={isOpen}
          aria-controls={listboxId}
          aria-activedescendant={activeDescendant}
          className="combobox-input"
          value={inputValue}
          onChange={e => handleInputChange(e.target.value)}
          onKeyDown={handleKeyDown}
          onFocus={() => { if (inputValue) open(); }}
        />
        <button
          tabIndex={-1}
          className="combobox-toggle"
          aria-label={isOpen ? 'Hide suggestions' : 'Show suggestions'}
          aria-expanded={isOpen}
          aria-controls={listboxId}
          onClick={() => {
            if (isOpen) { close(); } else { open(); inputRef.current?.focus(); }
          }}
        >
          <span className="combobox-caret" aria-hidden="true">▾</span>
        </button>
      </div>

      <ul
        ref={listRef}
        id={listboxId}
        role="listbox"
        aria-labelledby={labelId}
        className="combobox-listbox"
        hidden={!isOpen}
      >
        {filtered.length === 0 ? (
          // role="option" with aria-disabled prevents screen readers
          // from reading "0 options" and then silence
          <li role="option" aria-disabled="true" className="combobox-empty">
            No results for &ldquo;{inputValue}&rdquo;
          </li>
        ) : (
          filtered.map((opt, i) => (
            <li
              key={opt.value}
              id={`${optionIdPrefix}-${i}`}
              role="option"
              aria-selected={i === activeIndex}
              className={`combobox-option ${i === activeIndex ? 'combobox-option--active' : ''}`}
              onMouseDown={e => {
                // Prevent input blur before we can handle the click
                e.preventDefault();
                selectOption(opt);
              }}
            >
              {opt.label}
            </li>
          ))
        )}
      </ul>
    </div>
  );
}
```

### Usage

```tsx
// App.tsx
const CITIES: ComboboxOption[] = [
  { value: 'nyc',    label: 'New York'       },
  { value: 'la',     label: 'Los Angeles'    },
  { value: 'chi',    label: 'Chicago'        },
  { value: 'hou',    label: 'Houston'        },
  { value: 'phx',    label: 'Phoenix'        },
  { value: 'phi',    label: 'Philadelphia'   },
];

function App() {
  return (
    <Combobox
      label="City"
      options={CITIES}
      placeholder="Search cities…"
      onSelect={opt => console.log('Selected:', opt)}
    />
  );
}
```

---

## Angular Implementation

### Model & Service

```typescript
// combobox.model.ts
export interface ComboboxOption {
  value: string;
  label: string;
}
```

```typescript
// combobox.service.ts
import { Injectable, signal, computed } from '@angular/core';
import { ComboboxOption } from './combobox.model';

@Injectable()   // component-scoped — one instance per combobox
export class ComboboxService {
  private _allOptions  = signal<ComboboxOption[]>([]);
  private _inputValue  = signal('');
  private _isOpen      = signal(false);
  private _activeIndex = signal(-1);
  private _selected    = signal<ComboboxOption | null>(null);

  readonly inputValue  = this._inputValue.asReadonly();
  readonly isOpen      = this._isOpen.asReadonly();
  readonly activeIndex = this._activeIndex.asReadonly();
  readonly selected    = this._selected.asReadonly();

  readonly filtered = computed(() => {
    const q = this._inputValue().toLowerCase();
    return this._allOptions().filter(o => o.label.toLowerCase().includes(q));
  });

  readonly activeDescendant = computed(() => {
    const i = this._activeIndex();
    return i >= 0 ? `cb-opt-${i}` : null;
  });

  init(options: ComboboxOption[]) {
    this._allOptions.set(options);
  }

  setInput(value: string) {
    this._inputValue.set(value);
    this._isOpen.set(true);
    this._activeIndex.set(-1);
    this._selected.set(null);
  }

  open()  { this._isOpen.set(true);  this._activeIndex.set(-1); }
  close() { this._isOpen.set(false); this._activeIndex.set(-1); }

  moveDown() {
    const max = this.filtered().length - 1;
    this._activeIndex.update(i => Math.min(i + 1, max));
  }

  moveUp() {
    this._activeIndex.update(i => Math.max(i - 1, -1));
  }

  moveFirst() { this._activeIndex.set(0); }
  moveLast()  { this._activeIndex.set(this.filtered().length - 1); }

  selectActive() {
    const i   = this._activeIndex();
    const opt = this.filtered()[i];
    if (opt) this.selectOption(opt);
  }

  selectOption(opt: ComboboxOption) {
    this._inputValue.set(opt.label);
    this._selected.set(opt);
    this.close();
  }

  clearInput() {
    this._inputValue.set('');
    this._selected.set(null);
  }
}
```

### Component

```typescript
// combobox.component.ts
import {
  Component, Input, Output, EventEmitter, OnInit,
  inject, ElementRef, ViewChild, effect, HostListener
} from '@angular/core';
import { NgFor, NgIf } from '@angular/common';
import { ComboboxService } from './combobox.service';
import { ComboboxOption } from './combobox.model';

@Component({
  selector: 'app-combobox',
  standalone: true,
  imports: [NgFor, NgIf],
  providers: [ComboboxService],   // scoped per instance
  template: `
    <div class="combobox-wrapper" (focusout)="onFocusOut($event)">
      <label [id]="labelId" [for]="inputId">{{ label }}</label>

      <div class="combobox-control">
        <input
          #inputEl
          [id]="inputId"
          type="text"
          role="combobox"
          autocomplete="off"
          autocorrect="off"
          autocapitalize="none"
          spellcheck="false"
          [placeholder]="placeholder"
          [attr.aria-labelledby]="labelId"
          aria-autocomplete="list"
          [attr.aria-expanded]="cb.isOpen()"
          [attr.aria-controls]="listboxId"
          [attr.aria-activedescendant]="cb.activeDescendant()"
          class="combobox-input"
          [value]="cb.inputValue()"
          (input)="cb.setInput($any($event.target).value)"
          (keydown)="onKeyDown($event)"
          (focus)="onInputFocus()"
        />
        <button
          tabindex="-1"
          class="combobox-toggle"
          [attr.aria-label]="cb.isOpen() ? 'Hide suggestions' : 'Show suggestions'"
          [attr.aria-expanded]="cb.isOpen()"
          [attr.aria-controls]="listboxId"
          (click)="togglePopup()"
        >
          <span class="combobox-caret" aria-hidden="true">▾</span>
        </button>
      </div>

      <ul
        #listEl
        [id]="listboxId"
        role="listbox"
        [attr.aria-labelledby]="labelId"
        class="combobox-listbox"
        [hidden]="!cb.isOpen()"
      >
        <li
          *ngIf="cb.filtered().length === 0"
          role="option"
          aria-disabled="true"
          class="combobox-empty"
        >
          No results for &ldquo;{{ cb.inputValue() }}&rdquo;
        </li>
        <li
          *ngFor="let opt of cb.filtered(); let i = index; trackBy: trackByValue"
          [id]="'cb-opt-' + i"
          role="option"
          [attr.aria-selected]="i === cb.activeIndex()"
          [class.combobox-option--active]="i === cb.activeIndex()"
          class="combobox-option"
          (mousedown)="$event.preventDefault(); cb.selectOption(opt)"
        >
          {{ opt.label }}
        </li>
      </ul>
    </div>
  `,
})
export class ComboboxComponent implements OnInit {
  @Input() label       = '';
  @Input() options: ComboboxOption[] = [];
  @Input() placeholder = '';
  @Output() selected   = new EventEmitter<ComboboxOption>();

  @ViewChild('inputEl') inputEl!: ElementRef<HTMLInputElement>;
  @ViewChild('listEl')  listEl!:  ElementRef<HTMLUListElement>;

  cb = inject(ComboboxService);

  readonly labelId   = `cb-label-${Math.random().toString(36).slice(2)}`;
  readonly inputId   = `cb-input-${Math.random().toString(36).slice(2)}`;
  readonly listboxId = `cb-list-${Math.random().toString(36).slice(2)}`;

  constructor() {
    // Scroll active option into view and emit selection
    effect(() => {
      const i = this.cb.activeIndex();
      if (i < 0 || !this.listEl) return;
      const el = this.listEl.nativeElement.querySelector<HTMLElement>(`#cb-opt-${i}`);
      el?.scrollIntoView({ block: 'nearest' });
    });

    effect(() => {
      const sel = this.cb.selected();
      if (sel) this.selected.emit(sel);
    });
  }

  ngOnInit() {
    this.cb.init(this.options);
  }

  onInputFocus() {
    if (this.cb.inputValue()) this.cb.open();
  }

  onFocusOut(e: FocusEvent) {
    const wrapper = (e.currentTarget as HTMLElement);
    if (!wrapper.contains(e.relatedTarget as Node)) {
      this.cb.close();
    }
  }

  togglePopup() {
    if (this.cb.isOpen()) { this.cb.close(); }
    else { this.cb.open(); this.inputEl.nativeElement.focus(); }
  }

  onKeyDown(e: KeyboardEvent) {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        this.cb.isOpen() ? this.cb.moveDown() : this.cb.open();
        break;
      case 'ArrowUp':
        e.preventDefault();
        if (this.cb.isOpen()) this.cb.moveUp();
        break;
      case 'Enter':
        if (this.cb.isOpen() && this.cb.activeIndex() >= 0) {
          e.preventDefault();
          this.cb.selectActive();
        }
        break;
      case 'Escape':
        e.preventDefault();
        if (this.cb.isOpen()) { this.cb.close(); }
        else { this.cb.clearInput(); }
        break;
      case 'Tab':
        if (this.cb.isOpen() && this.cb.activeIndex() >= 0) {
          this.cb.selectActive();
        } else {
          this.cb.close();
        }
        break;
      case 'Home':
        if (this.cb.isOpen()) { e.preventDefault(); this.cb.moveFirst(); }
        break;
      case 'End':
        if (this.cb.isOpen()) { e.preventDefault(); this.cb.moveLast(); }
        break;
    }
  }

  trackByValue(_: number, opt: ComboboxOption) {
    return opt.value;
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Input has `role="combobox"` | Static attribute on the `<input>` |
| `aria-expanded` reflects popup state | Toggled by JS on every open/close |
| `aria-controls` points to listbox id | Static attribute; id must match the `<ul>` |
| Listbox has `role="listbox"` | Static attribute on the `<ul>` |
| Each option has `role="option"` and unique `id` | Generated as `${prefix}-${index}` |
| Active option announced on arrow key | `aria-activedescendant` updated to active option's `id` |
| Active option has `aria-selected="true"` | JS sets `true` on highlighted index, `false` on others |
| Focus stays on input throughout navigation | `aria-activedescendant` pattern — focus never moves to list |
| Selected option has `aria-selected="true"` after picking | The chosen `<li>` keeps `aria-selected="true"` until next interaction |
| Listbox hidden from AT when closed | `hidden` attribute removes it from accessibility tree |
| `aria-autocomplete="list"` declared | Tells AT that suggestions come from a list (not inline) |
| Empty state is not silent | An `aria-disabled` option announces "No results for X" |
| `mousedown` preventDefault on options | Prevents input blur before `selectOption` fires |
| Active option scrolled into view | `scrollIntoView({ block: 'nearest' })` on index change |
| Widget closes on outside click/focus | `focusout` checks if new focus target is inside wrapper |
| Minimum input tap target 44px height | Enforced in CSS via `min-height: 44px` |
| Reduced motion respected | `transition: none` under `prefers-reduced-motion` |

---

## Production Pitfalls

**1. Using `aria-activedescendant` without unique, stable option IDs**
`aria-activedescendant` must match the exact `id` attribute of a DOM element inside the listbox. If options are re-rendered with new keys on each filter update (e.g., React array index as key when items reorder), the IDs change and screen readers announce nothing. Fix: use `${listboxId}-opt-${index}` where the index is into the *filtered* array — stable within a single open session.

**2. Moving DOM focus to a list item (roving tabindex) instead of using `aria-activedescendant`**
Moving focus physically into the `role="listbox"` breaks the keyboard contract for a combobox: the user can no longer type to continue filtering. Roving tabindex is correct for `role="menu"` and standalone `role="listbox"`, but ARIA's combobox pattern specifies `aria-activedescendant` precisely to keep focus in the input. Never add `tabindex` to `role="option"` elements in a combobox.

**3. Setting `aria-selected` on options only when selected, not during keyboard navigation**
Many implementations toggle `aria-selected` only when the user presses Enter. But the spec requires `aria-selected="true"` on the *currently highlighted* option during keyboard navigation. Some screen readers (NVDA + Firefox in particular) rely on this to announce the highlighted item. Set `aria-selected` based on `activeIndex`, and update it on every arrow key.

**4. Forgetting `mousedown` preventDefault on option clicks**
When a user clicks an option, the input fires `blur` before `click`. This causes the listbox to close (via the `focusout` handler) before the click event resolves — the selection is swallowed silently. Fix: use `mousedown` instead of `click` on options, and call `e.preventDefault()` to block the blur.

**5. iOS VoiceOver ignores `aria-activedescendant` entirely**
On iOS, VoiceOver does not follow `aria-activedescendant` — it reads the input value but never announces the highlighted option. The workaround is to update the input's `value` itself to the highlighted option's label as the user arrows down (with an in-progress selection class), then restore the typed query if Escape is pressed. This is a pragmatic deviation from the spec for iOS compatibility.

**6. TalkBack (Android) in Browse mode intercepts arrow keys**
Android TalkBack in "reading mode" (swipe navigation) does not forward arrow keys to the web content. Users must switch to "linear navigation" mode manually. There is no CSS or ARIA fix for this. The mitigation is to ensure the options are also reachable by swipe — meaning the listbox must be in the accessibility tree and each `role="option"` must be individually focusable in TalkBack's virtual cursor mode. Do not hide them with `visibility: hidden` or `pointer-events: none`.

**7. Popup overflowing the viewport at the bottom of the page**
`position: absolute; top: 100%` positions the popup below the input. Near the viewport bottom, the list is clipped. Fix: measure available space below (`window.innerHeight - inputRect.bottom`) and flip to `bottom: 100%` if the list would overflow. This requires JS — CSS alone cannot conditionally flip the popup. Libraries like Floating UI handle this automatically.

**8. Screen readers double-announcing the option label**
When `aria-activedescendant` updates, some screen reader + browser combinations read the option content twice: once from the `aria-activedescendant` announcement and once because the option text visually changes. Fix: ensure the option element contains only plain text, no nested `aria-label` or `aria-describedby` that would duplicate the announcement.

---

## Interview Angle

**Q: "What is the difference between `aria-activedescendant` and roving tabindex, and why does the combobox pattern use one and not the other?"**

With roving tabindex, DOM focus physically moves to each item: the focused item has `tabindex="0"`, all others have `tabindex="-1"`. This is correct for standalone widgets like toolbars, menus, and tree views, where moving focus is exactly what you want. A combobox has a different contract: focus must stay on the `<input>` at all times so the user can type to continue filtering while arrow keys navigate the list. `aria-activedescendant` solves this by keeping focus on the input and pointing to the id of the visually highlighted option. The screen reader reads the referenced element's content without the browser moving caret focus. The trade-off is that `aria-activedescendant` has significantly worse support on iOS VoiceOver, requiring the iOS-specific workaround of mirroring the highlighted option label into the input value.

**Follow-up: "How would you debug a combobox where VoiceOver on Mac announces nothing when the user presses ArrowDown?"**

Start with the accessibility tree in browser DevTools — confirm the input has `role="combobox"`, `aria-controls` points to an existing element, and `aria-expanded="true"` when the list is open. Then check that `aria-activedescendant` on the input matches an `id` attribute on a visible `role="option"` element inside the referenced `role="listbox"`. The most common cause is the option `id` being set on a React/Angular wrapper `div` rather than the `<li>` itself, so the id exists in the DOM but the referenced element lacks `role="option"`. Also confirm the listbox is not hidden via `display:none` or the `hidden` attribute while the popup is open — VoiceOver ignores `aria-activedescendant` if the referenced element is not in the rendered accessibility tree.
