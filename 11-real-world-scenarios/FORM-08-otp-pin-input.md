# OTP / PIN Input

## The Idea

**In plain English:** An OTP (One-Time Password) input is a row of single-character boxes — one digit per box — where typing in one box automatically moves focus to the next, and pressing Backspace moves focus back. The entire row represents one logical value even though it is rendered as many inputs.

**Real-world analogy:** Think of the PIN pad at an ATM.

- There are **4 separate slots**, but you are entering **one 4-digit number**
- You press **3** — it fills slot 1 and the cursor jumps to slot 2 automatically
- You press **Backspace** — the last digit is cleared and the cursor jumps back to slot 1
- You can **paste** "3829" and all four slots fill instantly
- The full code is only "submitted" once all slots are filled — the ATM doesn't try to verify "3" alone
- If you accidentally type a letter, the slot stays empty (only digits allowed)

The key insight: each `<input>` is a controlled box that holds exactly one character, but the component exposes a single `value` string to the parent. The ref array — not the DOM's natural tab order — controls focus movement.

---

## Learning Objectives

- Build a controlled multi-input component with `useImperativeHandle` for a clean external API
- Implement auto-advance on valid input, and backspace-to-previous behaviour
- Handle paste events: split the pasted string across all inputs
- Expose the component as a single `<PinInput value onChange />` interface to parents
- Make the group accessible as a single logical input

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Render N boxes side by side | ✅ via flexbox | — |
| Auto-advance focus to next box | ❌ | Focus management is JS |
| Move focus back on Backspace | ❌ | Backspace key handling is JS |
| Split pasted text across boxes | ❌ | `paste` event is JS |
| Validate input (digits only) | ❌ | Input filtering is JS |
| Announce "3 of 6 digits entered" to screen readers | ❌ | aria-live updates are JS |

---

## HTML & CSS Foundation

```html
<!-- The entire row is grouped as a single widget -->
<fieldset class="pin-group" role="group" aria-labelledby="pin-label">
  <legend id="pin-label">Enter your 6-digit verification code</legend>

  <div class="pin-inputs">
    <input
      class="pin-box"
      type="text"
      inputmode="numeric"
      pattern="[0-9]*"
      maxlength="1"
      autocomplete="one-time-code"
      aria-label="Digit 1 of 6"
      value=""
    />
    <!-- Repeated × 6 -->
  </div>

  <!-- Live region for screen reader progress -->
  <div class="sr-only" aria-live="polite" id="pin-status"></div>
</fieldset>
```

```css
:root {
  --pin-size: 52px;
  --pin-gap: 10px;
  --pin-border: 2px solid #ccc;
  --pin-focus-color: #1a73e8;
  --pin-filled-border: #333;
  --pin-error: #d93025;
}

.pin-group {
  border: none;
  padding: 0;
  margin: 0;
}

.pin-group legend {
  font-size: 0.9rem;
  color: #555;
  margin-bottom: 12px;
}

.pin-inputs {
  display: flex;
  gap: var(--pin-gap);
}

.pin-box {
  width: var(--pin-size);
  height: var(--pin-size);
  border: var(--pin-border);
  border-radius: 8px;
  text-align: center;
  font-size: 1.5rem;
  font-weight: 600;
  caret-color: transparent;   /* hide caret — the whole char replaces on type */
  transition: border-color 0.15s;
  -moz-appearance: textfield; /* remove number spinners in Firefox */
}

.pin-box::-webkit-outer-spin-button,
.pin-box::-webkit-inner-spin-button { display: none; }

.pin-box:focus {
  outline: none;
  border-color: var(--pin-focus-color);
  box-shadow: 0 0 0 3px rgba(26, 115, 232, 0.2);
}

.pin-box.is-filled {
  border-color: var(--pin-filled-border);
}

.pin-box.is-error {
  border-color: var(--pin-error);
}

/* Separator between groups (e.g. 3-3 split for OTP) */
.pin-sep {
  display: flex;
  align-items: center;
  color: #ccc;
  font-size: 1.5rem;
  user-select: none;
}
```

---

## React Implementation

### `useImperativeHandle` API

```tsx
// PinInput.tsx
import {
  useRef, useState, useCallback, forwardRef,
  useImperativeHandle, useEffect, ClipboardEvent,
  KeyboardEvent, ChangeEvent
} from 'react';

export interface PinInputHandle {
  focus: () => void;
  clear: () => void;
  getValue: () => string;
}

interface PinInputProps {
  length?: number;
  onChange?: (value: string) => void;
  onComplete?: (value: string) => void;
  disabled?: boolean;
  autoFocus?: boolean;
  mask?: boolean;          // show • instead of digits
  separator?: number;      // insert a visual gap every N digits (e.g. 3 for "3-3" OTP)
}

export const PinInput = forwardRef<PinInputHandle, PinInputProps>(function PinInput(
  {
    length = 6,
    onChange,
    onComplete,
    disabled = false,
    autoFocus = false,
    mask = false,
    separator,
  },
  ref
) {
  const [values, setValues] = useState<string[]>(Array(length).fill(''));
  const inputRefs = useRef<(HTMLInputElement | null)[]>([]);

  // Expose imperative API to parent
  useImperativeHandle(ref, () => ({
    focus: () => inputRefs.current[0]?.focus(),
    clear: () => {
      setValues(Array(length).fill(''));
      inputRefs.current[0]?.focus();
    },
    getValue: () => values.join(''),
  }));

  // Auto-focus first input on mount
  useEffect(() => {
    if (autoFocus) inputRefs.current[0]?.focus();
  }, [autoFocus]);

  const focusAt = useCallback((index: number) => {
    const el = inputRefs.current[Math.max(0, Math.min(index, length - 1))];
    el?.focus();
    el?.select();
  }, [length]);

  const updateValues = useCallback(
    (next: string[]) => {
      setValues(next);
      const full = next.join('');
      onChange?.(full);
      if (next.every(v => v !== '')) onComplete?.(full);
    },
    [onChange, onComplete]
  );

  const handleChange = useCallback(
    (e: ChangeEvent<HTMLInputElement>, index: number) => {
      const raw = e.target.value.replace(/\D/g, ''); // digits only
      if (!raw) return;

      const digit = raw[raw.length - 1]; // take last digit if multiple pasted via change
      const next = [...values];
      next[index] = digit;
      updateValues(next);

      // Advance focus
      if (index < length - 1) focusAt(index + 1);
    },
    [values, length, focusAt, updateValues]
  );

  const handleKeyDown = useCallback(
    (e: KeyboardEvent<HTMLInputElement>, index: number) => {
      if (e.key === 'Backspace') {
        e.preventDefault();
        const next = [...values];
        if (next[index]) {
          // Clear current box
          next[index] = '';
          updateValues(next);
        } else if (index > 0) {
          // Current box already empty — go back and clear previous
          next[index - 1] = '';
          updateValues(next);
          focusAt(index - 1);
        }
      } else if (e.key === 'ArrowLeft') {
        e.preventDefault();
        focusAt(index - 1);
      } else if (e.key === 'ArrowRight') {
        e.preventDefault();
        focusAt(index + 1);
      } else if (e.key === 'Delete') {
        e.preventDefault();
        const next = [...values];
        next[index] = '';
        updateValues(next);
      }
    },
    [values, focusAt, updateValues]
  );

  const handlePaste = useCallback(
    (e: ClipboardEvent<HTMLInputElement>, index: number) => {
      e.preventDefault();
      const pasted = e.clipboardData.getData('text').replace(/\D/g, '');
      if (!pasted) return;

      const next = [...values];
      let cursor = index;
      for (let i = 0; i < pasted.length && cursor < length; i++, cursor++) {
        next[cursor] = pasted[i];
      }
      updateValues(next);
      // Focus the box after the last pasted digit
      focusAt(Math.min(cursor, length - 1));
    },
    [values, length, focusAt, updateValues]
  );

  const filledCount = values.filter(Boolean).length;

  return (
    <fieldset className="pin-group" disabled={disabled}>
      <legend>Enter your {length}-digit verification code</legend>

      <div className="pin-inputs" role="group" aria-label={`${length}-digit code`}>
        {values.map((val, i) => (
          <input
            key={i}
            ref={el => { inputRefs.current[i] = el; }}
            className={`pin-box${val ? ' is-filled' : ''}`}
            type={mask ? 'password' : 'text'}
            inputMode="numeric"
            pattern="[0-9]*"
            maxLength={1}
            autoComplete={i === 0 ? 'one-time-code' : 'off'}
            aria-label={`Digit ${i + 1} of ${length}`}
            value={val}
            disabled={disabled}
            onChange={e => handleChange(e, i)}
            onKeyDown={e => handleKeyDown(e, i)}
            onPaste={e => handlePaste(e, i)}
            onFocus={e => e.target.select()}
          />
        ))}
      </div>

      <div className="sr-only" aria-live="polite">
        {filledCount > 0 && filledCount < length
          ? `${filledCount} of ${length} digits entered`
          : filledCount === length
          ? 'All digits entered'
          : ''}
      </div>
    </fieldset>
  );
});
```

### Usage with Imperative Handle

```tsx
// VerifyPage.tsx
import { useRef } from 'react';
import { PinInput, PinInputHandle } from './PinInput';

export function VerifyPage() {
  const pinRef = useRef<PinInputHandle>(null);

  const handleComplete = async (code: string) => {
    try {
      await verifyOTP(code);
    } catch {
      pinRef.current?.clear(); // clear on wrong code
    }
  };

  return (
    <form>
      <PinInput
        ref={pinRef}
        length={6}
        autoFocus
        onComplete={handleComplete}
        onChange={v => console.log('partial:', v)}
      />
      <button type="button" onClick={() => pinRef.current?.clear()}>
        Clear
      </button>
    </form>
  );
}
```

---

## Angular Implementation

```typescript
// pin-input.component.ts
import {
  Component, Input, Output, EventEmitter,
  ViewChildren, QueryList, ElementRef,
  OnInit, ChangeDetectionStrategy,
  signal, computed
} from '@angular/core';
import { NgFor } from '@angular/common';
import { FormsModule } from '@angular/forms';

@Component({
  selector: 'app-pin-input',
  standalone: true,
  imports: [NgFor, FormsModule],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <fieldset class="pin-group" [disabled]="disabled">
      <legend>Enter your {{ length }}-digit code</legend>

      <div class="pin-inputs">
        <input
          *ngFor="let _ of boxes; let i = index"
          #inputEl
          class="pin-box"
          [class.is-filled]="values()[i]"
          [type]="mask ? 'password' : 'text'"
          inputmode="numeric"
          pattern="[0-9]*"
          maxlength="1"
          [attr.autocomplete]="i === 0 ? 'one-time-code' : 'off'"
          [attr.aria-label]="'Digit ' + (i + 1) + ' of ' + length"
          [value]="values()[i]"
          (input)="onInput($event, i)"
          (keydown)="onKeyDown($event, i)"
          (paste)="onPaste($event, i)"
          (focus)="$any($event.target).select()"
        />
      </div>

      <div class="sr-only" aria-live="polite">{{ statusMessage() }}</div>
    </fieldset>
  `,
})
export class PinInputComponent implements OnInit {
  @Input() length = 6;
  @Input() disabled = false;
  @Input() mask = false;
  @Output() codeChange  = new EventEmitter<string>();
  @Output() codeComplete = new EventEmitter<string>();

  @ViewChildren('inputEl') inputEls!: QueryList<ElementRef<HTMLInputElement>>;

  values = signal<string[]>([]);
  boxes: undefined[] = [];

  statusMessage = computed(() => {
    const filled = this.values().filter(Boolean).length;
    if (filled === 0 || filled === this.length) return '';
    return `${filled} of ${this.length} digits entered`;
  });

  ngOnInit() {
    this.boxes = Array(this.length);
    this.values.set(Array(this.length).fill(''));
  }

  private focusAt(i: number) {
    const el = this.inputEls?.get(Math.max(0, Math.min(i, this.length - 1)));
    el?.nativeElement.focus();
    el?.nativeElement.select();
  }

  private updateValues(next: string[]) {
    this.values.set(next);
    const full = next.join('');
    this.codeChange.emit(full);
    if (next.every(v => v !== '')) this.codeComplete.emit(full);
  }

  onInput(e: Event, i: number) {
    const input = e.target as HTMLInputElement;
    const digit = input.value.replace(/\D/g, '').slice(-1);
    const next = [...this.values()];
    next[i] = digit;
    this.updateValues(next);
    input.value = digit; // force value (avoids stale display)
    if (digit && i < this.length - 1) this.focusAt(i + 1);
  }

  onKeyDown(e: KeyboardEvent, i: number) {
    const next = [...this.values()];
    if (e.key === 'Backspace') {
      e.preventDefault();
      if (next[i]) { next[i] = ''; this.updateValues(next); }
      else if (i > 0) { next[i - 1] = ''; this.updateValues(next); this.focusAt(i - 1); }
    } else if (e.key === 'ArrowLeft') { e.preventDefault(); this.focusAt(i - 1); }
    else if (e.key === 'ArrowRight') { e.preventDefault(); this.focusAt(i + 1); }
  }

  onPaste(e: ClipboardEvent, startIndex: number) {
    e.preventDefault();
    const text = e.clipboardData?.getData('text').replace(/\D/g, '') ?? '';
    const next = [...this.values()];
    let cursor = startIndex;
    for (let i = 0; i < text.length && cursor < this.length; i++, cursor++) {
      next[cursor] = text[i];
    }
    this.updateValues(next);
    this.focusAt(Math.min(cursor, this.length - 1));
  }

  clear() { this.values.set(Array(this.length).fill('')); this.focusAt(0); }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Group wrapped in `<fieldset>` + `<legend>` | Announced as a group by screen readers |
| Each input has `aria-label="Digit N of M"` | Screen reader announces position on focus |
| `aria-live="polite"` progress region | Announces "3 of 6 digits entered" as user types |
| `inputmode="numeric"` | Shows numeric keypad on mobile |
| `autocomplete="one-time-code"` on first input | Enables SMS autofill on Android/iOS |
| `onFocus` → select all | Replaces digit cleanly without double-tap |
| `type="password"` mask option | For sensitive PINs that should not be visible |
| Backspace navigates to previous box | Expected keyboard behaviour |
| Arrow keys move between boxes | Keyboard discoverability |
| Paste splits across boxes | Prevents copy-paste friction |

---

## Production Pitfalls

**1. Android's virtual keyboard fires `keydown` inconsistently**
On Android, `keydown` events for Backspace often have `e.key === 'Unidentified'` or don't fire at all. Fix: use the `input` event instead of `keydown` for character input. For Backspace detection on Android, compare `e.inputType === 'deleteContentBackward'` in the `input` event handler.

**2. `value` and `onChange` fight on controlled inputs**
Setting `value` on an input and also handling `onChange` means React controls the input. If you set `value` inside `onChange` but the parent hasn't re-rendered yet, the input can appear to "bounce back". Fix: always derive displayed value from state, never from `e.target.value` after the fact.

**3. `autocomplete="one-time-code"` only works on the first input**
The browser's SMS autofill fills the first field with the entire code. If you intercept the `change` event and expect only one character, the autofill of "123456" into box 1 will fail. Fix: in `onChange`, check if `e.target.value.length > 1` and treat it as a paste event.

**4. Screen readers read each box as a separate input**
Some users find navigating 6 separate inputs confusing. For high-accessibility contexts, consider a single `<input maxlength="6">` with custom visual rendering over it (the "fake boxes" pattern using a background-image or CSS grid overlay).

**5. Paste on iOS requires the `onPaste` handler on every input**
iOS Safari fires `paste` on whichever input is focused at the moment of paste, not necessarily the first one. Ensure every input in the row has the paste handler, and that handler distributes from the current `startIndex` forward.

---

## Interview Angle

**Q: "How would you expose a `clear()` method on a PIN input component from a parent?"**

Use `useImperativeHandle` (React) or a `ViewChild` + public method (Angular). The parent gets a `ref` / reference to the component instance and calls `ref.current.clear()` / `pinComponent.clear()`. This is the imperative escape hatch for "parent needs to trigger child behaviour", which is the right pattern for OTP clear-on-error scenarios — avoiding prop-driven side effects.

**Q: "Why not just use a single `<input maxlength='6'>`?"**

A single input is simpler and accessible by default. But it cannot show individual-character visual boxes (card-style UX), cannot animate each digit as it fills, and cannot easily prevent non-digit input per character position. The multi-input approach is chosen for UX fidelity, not technical necessity — you trade accessibility simplicity for richer visual feedback.

**Q: "How does paste work across multiple inputs?"**

Handle the `paste` event on each input. On paste, prevent the default, read `clipboardData.getData('text')`, strip non-digits, and distribute characters sequentially starting from the currently focused input's index. Focus the last filled box or the first empty box after distribution.
