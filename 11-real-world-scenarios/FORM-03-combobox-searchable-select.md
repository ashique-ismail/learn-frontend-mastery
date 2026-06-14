# Combobox / Searchable Select

## The Idea

**In plain English:** A combobox is a text input that doubles as a dropdown selector. The user types to filter a list of options; they can pick one from the list or (sometimes) type a free-form value. It combines a `<input>` with a listbox panel, with full keyboard navigation between them.

**Real-world analogy:** Think of an airport self-check-in kiosk with a destination search.

- The **text field** is where you start typing "New Y…"
- The **dropdown list** is the suggestions panel that appears below — "New York JFK", "New York LGA", "Newark EWR"
- **Typing** filters the list in real time
- **Arrow keys** let you move through suggestions without losing your place in the text box
- **Enter or click** selects a suggestion and fills the input
- **Escape** dismisses the dropdown and returns focus to the input
- The kiosk also fetches flights from a live system — the options are **async**, not hardcoded

The key insight: this is NOT a `<select>` element dressed up. It is a custom widget built from an `<input>` + a `<ul>` that communicates its relationship to assistive technology entirely through ARIA.

---

## Learning Objectives

- Understand the ARIA combobox pattern (role, aria-expanded, aria-autocomplete, aria-activedescendant)
- Implement keyboard navigation without moving DOM focus away from the input
- Handle async option loading with debounce, loading state, and empty-state feedback
- Design a headless `useCombobox` hook that is UI-agnostic
- Avoid the common traps: NVDA/JAWS quirks with aria-activedescendant, option ID collisions

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Show/hide the dropdown panel | ✅ with `:focus-within` | — |
| Filter options as user types | ❌ | CSS cannot read input value |
| Highlight the focused option | ❌ | CSS doesn't know which option is "active" via keyboard |
| aria-activedescendant on input | ❌ | ARIA attributes require JS to update |
| Debounce async fetch | ❌ | Purely JS concern |
| Close on outside click | ❌ | Requires `mousedown` listener on document |
| Announce option count to screen readers | ❌ | aria-live updates require JS |

---

## HTML & CSS Foundation

```html
<div class="combobox-wrapper">
  <label for="destination-input" id="destination-label">Destination</label>

  <div class="combobox-input-wrapper">
    <input
      id="destination-input"
      type="text"
      role="combobox"
      aria-label="Destination"
      aria-autocomplete="list"
      aria-expanded="false"
      aria-controls="destination-listbox"
      aria-activedescendant=""
      autocomplete="off"
      class="combobox-input"
    />
    <!-- Loading spinner, injected by JS when fetching -->
    <span class="combobox-spinner" aria-hidden="true" hidden></span>
  </div>

  <!-- The options panel -->
  <ul
    id="destination-listbox"
    role="listbox"
    aria-label="Destinations"
    class="combobox-listbox"
    hidden
  >
    <!-- Options injected by JS -->
    <!-- <li role="option" id="opt-jfk" aria-selected="false">New York JFK</li> -->
  </ul>

  <!-- Screen reader status (option count, loading, no results) -->
  <div class="sr-only" aria-live="polite" id="combobox-status"></div>
</div>
```

```css
:root {
  --combo-border: 1px solid #ccc;
  --combo-radius: 6px;
  --combo-shadow: 0 4px 16px rgba(0, 0, 0, 0.12);
  --combo-highlight: #e8f0fe;
  --combo-selected: #1a73e8;
}

.combobox-wrapper {
  position: relative;
  display: inline-flex;
  flex-direction: column;
  gap: 4px;
  width: 320px;
}

.combobox-input-wrapper {
  position: relative;
  display: flex;
  align-items: center;
}

.combobox-input {
  width: 100%;
  padding: 10px 40px 10px 12px;
  border: var(--combo-border);
  border-radius: var(--combo-radius);
  font-size: 1rem;
  outline: none;
}

.combobox-input:focus {
  border-color: var(--combo-selected);
  box-shadow: 0 0 0 3px rgba(26, 115, 232, 0.2);
}

.combobox-listbox {
  position: absolute;
  top: calc(100% + 4px);
  left: 0;
  right: 0;
  max-height: 280px;
  overflow-y: auto;
  background: #fff;
  border: var(--combo-border);
  border-radius: var(--combo-radius);
  box-shadow: var(--combo-shadow);
  list-style: none;
  margin: 0;
  padding: 4px 0;
  z-index: 200;
}

.combobox-listbox[hidden] { display: none; }

.combobox-option {
  padding: 10px 14px;
  cursor: pointer;
  font-size: 0.95rem;
}

.combobox-option[aria-selected="true"],
.combobox-option.is-active {
  background: var(--combo-highlight);
  color: var(--combo-selected);
}

.combobox-option:hover {
  background: #f5f5f5;
}

.combobox-spinner {
  position: absolute;
  right: 10px;
  width: 18px;
  height: 18px;
  border: 2px solid #ccc;
  border-top-color: var(--combo-selected);
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
}

@keyframes spin { to { transform: rotate(360deg); } }
```

---

## React Implementation

### The Hook

```tsx
// useCombobox.ts
import { useState, useRef, useCallback, useEffect, useId } from 'react';

export interface ComboboxOption {
  value: string;
  label: string;
}

interface UseComboboxOptions {
  fetchOptions: (query: string) => Promise<ComboboxOption[]>;
  debounceMs?: number;
  onSelect?: (option: ComboboxOption) => void;
}

export function useCombobox({
  fetchOptions,
  debounceMs = 250,
  onSelect,
}: UseComboboxOptions) {
  const instanceId           = useId();
  const [inputValue, setInputValue] = useState('');
  const [options, setOptions]       = useState<ComboboxOption[]>([]);
  const [isOpen, setIsOpen]         = useState(false);
  const [activeIndex, setActiveIndex] = useState(-1);
  const [isLoading, setIsLoading]   = useState(false);
  const [selectedOption, setSelectedOption] = useState<ComboboxOption | null>(null);

  const debounceRef = useRef<ReturnType<typeof setTimeout>>();
  const inputRef    = useRef<HTMLInputElement>(null);
  const listboxRef  = useRef<HTMLUListElement>(null);

  // Stable option ID generator
  const getOptionId = useCallback(
    (index: number) => `${instanceId}-opt-${index}`,
    [instanceId]
  );

  const listboxId = `${instanceId}-listbox`;

  // Fetch options with debounce
  const handleInputChange = useCallback(
    (value: string) => {
      setInputValue(value);
      setActiveIndex(-1);
      clearTimeout(debounceRef.current);

      if (!value.trim()) {
        setOptions([]);
        setIsOpen(false);
        return;
      }

      debounceRef.current = setTimeout(async () => {
        setIsLoading(true);
        try {
          const results = await fetchOptions(value);
          setOptions(results);
          setIsOpen(results.length > 0);
        } finally {
          setIsLoading(false);
        }
      }, debounceMs);
    },
    [fetchOptions, debounceMs]
  );

  const selectOption = useCallback(
    (option: ComboboxOption) => {
      setInputValue(option.label);
      setSelectedOption(option);
      setIsOpen(false);
      setActiveIndex(-1);
      onSelect?.(option);
    },
    [onSelect]
  );

  const handleKeyDown = useCallback(
    (e: React.KeyboardEvent<HTMLInputElement>) => {
      switch (e.key) {
        case 'ArrowDown':
          e.preventDefault();
          setActiveIndex(i => Math.min(i + 1, options.length - 1));
          break;
        case 'ArrowUp':
          e.preventDefault();
          setActiveIndex(i => Math.max(i - 1, -1));
          break;
        case 'Enter':
          if (activeIndex >= 0 && options[activeIndex]) {
            e.preventDefault();
            selectOption(options[activeIndex]);
          }
          break;
        case 'Escape':
          setIsOpen(false);
          setActiveIndex(-1);
          break;
        case 'Tab':
          setIsOpen(false);
          break;
      }
    },
    [options, activeIndex, selectOption]
  );

  // Close on outside click
  useEffect(() => {
    const handler = (e: MouseEvent) => {
      if (
        !inputRef.current?.closest('.combobox-wrapper')?.contains(e.target as Node)
      ) {
        setIsOpen(false);
      }
    };
    document.addEventListener('mousedown', handler);
    return () => document.removeEventListener('mousedown', handler);
  }, []);

  // Scroll active option into view
  useEffect(() => {
    if (activeIndex < 0 || !listboxRef.current) return;
    const el = listboxRef.current.querySelector<HTMLElement>(
      `[id="${getOptionId(activeIndex)}"]`
    );
    el?.scrollIntoView({ block: 'nearest' });
  }, [activeIndex, getOptionId]);

  return {
    inputValue,
    options,
    isOpen,
    activeIndex,
    isLoading,
    selectedOption,
    inputRef,
    listboxRef,
    listboxId,
    getOptionId,
    handleInputChange,
    handleKeyDown,
    selectOption,
  };
}
```

### The Component

```tsx
// Combobox.tsx
import { useCombobox, ComboboxOption } from './useCombobox';

interface Props {
  label: string;
  placeholder?: string;
  fetchOptions: (query: string) => Promise<ComboboxOption[]>;
  onSelect?: (option: ComboboxOption) => void;
}

export function Combobox({ label, placeholder, fetchOptions, onSelect }: Props) {
  const {
    inputValue, options, isOpen, activeIndex, isLoading,
    inputRef, listboxRef, listboxId, getOptionId,
    handleInputChange, handleKeyDown, selectOption,
  } = useCombobox({ fetchOptions, onSelect });

  const activeDescendant =
    activeIndex >= 0 ? getOptionId(activeIndex) : undefined;

  return (
    <div className="combobox-wrapper">
      <label htmlFor="combobox-input">{label}</label>
      <div className="combobox-input-wrapper">
        <input
          id="combobox-input"
          ref={inputRef}
          type="text"
          role="combobox"
          aria-autocomplete="list"
          aria-expanded={isOpen}
          aria-controls={listboxId}
          aria-activedescendant={activeDescendant}
          autoComplete="off"
          className="combobox-input"
          placeholder={placeholder}
          value={inputValue}
          onChange={e => handleInputChange(e.target.value)}
          onKeyDown={handleKeyDown}
        />
        {isLoading && (
          <span className="combobox-spinner" aria-hidden="true" />
        )}
      </div>

      {/* Screen reader status */}
      <div className="sr-only" aria-live="polite">
        {isLoading
          ? 'Loading options…'
          : isOpen
          ? `${options.length} option${options.length !== 1 ? 's' : ''} available`
          : !isLoading && inputValue && !isOpen
          ? 'No results found'
          : ''}
      </div>

      {isOpen && (
        <ul
          ref={listboxRef}
          id={listboxId}
          role="listbox"
          aria-label={label}
          className="combobox-listbox"
        >
          {options.map((opt, i) => (
            <li
              key={opt.value}
              id={getOptionId(i)}
              role="option"
              aria-selected={i === activeIndex}
              className={`combobox-option${i === activeIndex ? ' is-active' : ''}`}
              onMouseDown={e => {
                // mousedown fires before blur; prevent closing before click
                e.preventDefault();
                selectOption(opt);
              }}
            >
              {opt.label}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}

// Usage example:
// async function searchCities(q: string): Promise<ComboboxOption[]> {
//   const res = await fetch(`/api/cities?q=${encodeURIComponent(q)}`);
//   return res.json();
// }
// <Combobox label="Destination" fetchOptions={searchCities} />
```

---

## Angular Implementation

```typescript
// combobox.component.ts
import {
  Component, Input, Output, EventEmitter,
  signal, computed, inject, ElementRef,
  HostListener, OnDestroy, ViewChild
} from '@angular/core';
import { FormsModule } from '@angular/forms';
import { NgFor, NgIf } from '@angular/common';

export interface ComboboxOption { value: string; label: string; }

@Component({
  selector: 'app-combobox',
  standalone: true,
  imports: [FormsModule, NgFor, NgIf],
  template: `
    <div class="combobox-wrapper" #wrapperEl>
      <label [for]="inputId">{{ label }}</label>

      <div class="combobox-input-wrapper">
        <input
          [id]="inputId"
          #inputEl
          type="text"
          role="combobox"
          aria-autocomplete="list"
          [attr.aria-expanded]="isOpen()"
          [attr.aria-controls]="listboxId"
          [attr.aria-activedescendant]="activeDescendant()"
          autocomplete="off"
          class="combobox-input"
          [placeholder]="placeholder"
          [(ngModel)]="inputValue"
          (ngModelChange)="onInputChange($event)"
          (keydown)="onKeyDown($event)"
        />
        <span *ngIf="isLoading()" class="combobox-spinner" aria-hidden="true"></span>
      </div>

      <div class="sr-only" aria-live="polite">{{ statusMessage() }}</div>

      <ul
        *ngIf="isOpen()"
        [id]="listboxId"
        #listboxEl
        role="listbox"
        [attr.aria-label]="label"
        class="combobox-listbox"
      >
        <li
          *ngFor="let opt of options(); let i = index; trackBy: trackByValue"
          [id]="getOptionId(i)"
          role="option"
          [attr.aria-selected]="i === activeIndex()"
          [class.is-active]="i === activeIndex()"
          class="combobox-option"
          (mousedown)="$event.preventDefault(); selectOption(opt)"
        >
          {{ opt.label }}
        </li>
      </ul>
    </div>
  `,
})
export class ComboboxComponent implements OnDestroy {
  @Input() label = '';
  @Input() placeholder = '';
  @Input() fetchOptions!: (q: string) => Promise<ComboboxOption[]>;
  @Output() selected = new EventEmitter<ComboboxOption>();
  @ViewChild('listboxEl') listboxEl?: ElementRef<HTMLUListElement>;
  @ViewChild('wrapperEl') wrapperEl!: ElementRef<HTMLDivElement>;

  readonly inputId  = `combobox-${Math.random().toString(36).slice(2)}`;
  readonly listboxId = `${this.inputId}-listbox`;

  inputValue = '';
  options    = signal<ComboboxOption[]>([]);
  isOpen     = signal(false);
  activeIndex = signal(-1);
  isLoading  = signal(false);

  activeDescendant = computed(() =>
    this.activeIndex() >= 0 ? this.getOptionId(this.activeIndex()) : null
  );

  statusMessage = computed(() => {
    if (this.isLoading()) return 'Loading options…';
    if (this.isOpen()) return `${this.options().length} option(s) available`;
    if (this.inputValue && !this.isOpen()) return 'No results found';
    return '';
  });

  private debounceTimer?: ReturnType<typeof setTimeout>;

  getOptionId(i: number) { return `${this.inputId}-opt-${i}`; }
  trackByValue(_: number, opt: ComboboxOption) { return opt.value; }

  onInputChange(value: string) {
    this.activeIndex.set(-1);
    clearTimeout(this.debounceTimer);
    if (!value.trim()) { this.options.set([]); this.isOpen.set(false); return; }

    this.debounceTimer = setTimeout(async () => {
      this.isLoading.set(true);
      try {
        const results = await this.fetchOptions(value);
        this.options.set(results);
        this.isOpen.set(results.length > 0);
      } finally {
        this.isLoading.set(false);
      }
    }, 250);
  }

  onKeyDown(e: KeyboardEvent) {
    const opts = this.options();
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        this.activeIndex.update(i => Math.min(i + 1, opts.length - 1));
        this.scrollActiveIntoView();
        break;
      case 'ArrowUp':
        e.preventDefault();
        this.activeIndex.update(i => Math.max(i - 1, -1));
        this.scrollActiveIntoView();
        break;
      case 'Enter':
        if (this.activeIndex() >= 0) {
          e.preventDefault();
          this.selectOption(opts[this.activeIndex()]);
        }
        break;
      case 'Escape':
        this.isOpen.set(false);
        this.activeIndex.set(-1);
        break;
    }
  }

  selectOption(opt: ComboboxOption) {
    this.inputValue = opt.label;
    this.isOpen.set(false);
    this.activeIndex.set(-1);
    this.selected.emit(opt);
  }

  @HostListener('document:mousedown', ['$event'])
  onOutsideClick(e: MouseEvent) {
    if (!this.wrapperEl.nativeElement.contains(e.target as Node)) {
      this.isOpen.set(false);
    }
  }

  private scrollActiveIntoView() {
    const i = this.activeIndex();
    if (i < 0 || !this.listboxEl) return;
    const el = this.listboxEl.nativeElement.querySelector<HTMLElement>(
      `#${this.getOptionId(i)}`
    );
    el?.scrollIntoView({ block: 'nearest' });
  }

  ngOnDestroy() { clearTimeout(this.debounceTimer); }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| `<input>` has `role="combobox"` | Required per ARIA 1.2 combobox pattern |
| `aria-expanded` reflects listbox visibility | Toggled on open/close |
| `aria-controls` points to listbox `id` | Static attribute on input |
| `aria-autocomplete="list"` | Informs AT that a list filters as you type |
| `aria-activedescendant` tracks focused option | Updated on ArrowUp/Down without moving DOM focus |
| Each option has unique `id` | Required for `aria-activedescendant` to work |
| Options have `role="option"` inside `role="listbox"` | Correct ARIA ownership |
| `aria-selected` on each option | Reflects current keyboard highlight |
| `aria-live="polite"` status region | Announces count / loading / no results |
| `mousedown` (not `click`) to select | Prevents input blur before selection registers |
| Escape closes without selecting | Keyboard contract for combobox |

---

## Production Pitfalls

**1. `aria-activedescendant` requires stable, unique IDs**
If you render two comboboxes on the same page without namespaced IDs, NVDA/JAWS will point to the wrong option. Fix: use `useId()` (React 18+) or a per-instance random suffix in Angular.

**2. Blur fires before click on options**
If you use `onClick` to select an option, the input's `blur` event fires first and closes the listbox before the click registers. Fix: use `onMouseDown` + `e.preventDefault()` to prevent the blur entirely.

**3. Async race condition**
If the user types "ab" → "abc" quickly, the slower "ab" response may arrive after "abc" and overwrite the correct results. Fix: cancel the previous fetch (AbortController) or ignore stale results by comparing the query at response time.

**4. VoiceOver on iOS does not support `aria-activedescendant`**
VoiceOver iOS ignores `aria-activedescendant` entirely; it expects real focus movement. For iOS support, consider moving DOM focus to each option on ArrowDown (changing pattern slightly) or providing a native `<select>` fallback.

**5. Options panel obscured by `overflow: hidden` ancestors**
If the combobox lives inside a scrollable container with `overflow: hidden`, the listbox will be clipped. Fix: use a portal to render the listbox at the body level and use `getBoundingClientRect()` to position it.

**6. Empty-state UX gap**
When there are no results, closing the dropdown silently gives the user no feedback. Fix: show a "No results" option inside the listbox (with `aria-live` announcement), then close.

---

## Interview Angle

**Q: "How does a combobox differ from a `<select>` element, and why would you build a custom one?"**

A `<select>` has no text filtering, fixed OS-native styling, and cannot display rich content (avatars, metadata) inside options. A custom combobox trades native semantics for design freedom. You must recreate all ARIA roles and keyboard behaviour manually: `role="combobox"`, `role="listbox"`, `role="option"`, `aria-expanded`, `aria-activedescendant`, and the full keyboard contract (ArrowDown/Up, Enter, Escape).

**Q: "Why use `aria-activedescendant` instead of actually moving focus to each option?"**

Moving DOM focus to `<li>` elements would fire the input's `blur` event, close the listbox, and destroy the options in the DOM — a chicken-and-egg problem. `aria-activedescendant` lets the input retain DOM focus while telling assistive technology which option is "logically active". The visually highlighted option and the AT-announced option stay in sync without the input losing focus.

**Q: "How do you handle async options without showing stale data?"**

Use an AbortController: cancel the previous fetch when a new keystroke arrives. Alternatively, store a `queryVersion` counter and only apply results that match the latest version. Debouncing alone doesn't solve races — it only reduces their frequency.
