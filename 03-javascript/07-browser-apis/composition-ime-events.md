# Composition Events and IME Handling

## Overview

Composition events fire when a user is composing text using an Input Method Editor (IME) — the software that converts sequences of keystrokes into characters for languages like Chinese, Japanese, Korean (CJK), Arabic, Hindi, and others that have more characters than keyboard keys. Missing these events is the most common hidden bug in search-as-you-type, autocomplete, and form validation built by developers who only type in Latin-script languages.

---

## The Problem

For Latin-script input (English, French, etc.), each `keydown` maps directly to a character. You can trigger search on every keystroke:

```javascript
// Works fine for English:
input.addEventListener('input', (e) => {
  search(e.target.value); // fire on every character
});
```

For Japanese input, the user types a phonetic sequence (romaji), which the IME converts to kanji/hiragana. During composition:

```
User types:  n → i → h → o → n → g → o
IME shows:   に → にほ → にほん → にほんご  (intermediate, uncommitted text)
User presses Enter/Space to commit: "日本語"
```

If you call `search()` on every `input` event during this, you're searching for `n`, `ni`, `nih`, `niho`... — garbage intermediate strings. The search fires 7 times for a 3-character word.

---

## The Three Composition Events

| Event | When | `event.data` |
|---|---|---|
| `compositionstart` | User starts IME composition (first keystroke into IME) | `""` |
| `compositionupdate` | Each keystroke updates the composition | Current intermediate string |
| `compositionend` | User commits the composition (Enter/Space/click) | Final committed string |

```javascript
input.addEventListener('compositionstart', (e) => {
  console.log('IME composition started');
  // e.data: "" (empty at start)
});

input.addEventListener('compositionupdate', (e) => {
  console.log('Composing:', e.data);
  // e.data: "ni", "nih", "niho", "nihon", "nihong", "nihongo"
});

input.addEventListener('compositionend', (e) => {
  console.log('Committed:', e.data);
  // e.data: "日本語" (the final committed text)
});
```

---

## The Fix: Composition Guard

The standard pattern is to track whether composition is active and skip processing during it:

```javascript
let isComposing = false;

input.addEventListener('compositionstart', () => { isComposing = true; });
input.addEventListener('compositionend', (e) => {
  isComposing = false;
  // Process the committed value now
  handleInput(input.value);
});

input.addEventListener('input', (e) => {
  if (isComposing) return; // skip during IME composition
  handleInput(e.target.value);
});
```

### Using `event.isComposing`

`InputEvent` and `KeyboardEvent` have an `isComposing` property — `true` if fired during a composition session. This is cleaner but has slightly different semantics:

```javascript
input.addEventListener('input', (e) => {
  if (e.isComposing) return; // built-in flag
  handleInput(e.target.value);
});

input.addEventListener('keydown', (e) => {
  if (e.isComposing) return; // e.g. don't submit form on Enter during IME
  if (e.key === 'Enter') submitForm();
});
```

**Note:** On some browsers, `compositionend` fires *before* the final `input` event. On others, it fires after. The `isComposing` flag on the `input` event is therefore more reliable than the boolean you set manually.

---

## Form Submission with Enter Key

A critical edge case: pressing Enter to commit an IME composition should NOT submit the form. It's the most common CJK input bug:

```javascript
// Bug: Enter submits form mid-composition
input.addEventListener('keydown', (e) => {
  if (e.key === 'Enter') submitSearch(); // ← fires during IME commit too!
});

// Fix: check isComposing
input.addEventListener('keydown', (e) => {
  if (e.key === 'Enter' && !e.isComposing) {
    submitSearch();
  }
});
```

---

## Search-as-You-Type: Full Correct Implementation

```typescript
function createSearchInput(input: HTMLInputElement, onSearch: (q: string) => void) {
  let debounceTimer: number;

  function handleSearch(value: string) {
    clearTimeout(debounceTimer);
    debounceTimer = window.setTimeout(() => onSearch(value), 300);
  }

  // Debounced search on every non-composition input
  input.addEventListener('input', (e) => {
    if ((e as InputEvent).isComposing) return;
    handleSearch(input.value);
  });

  // Fire immediately when composition commits
  input.addEventListener('compositionend', () => {
    clearTimeout(debounceTimer); // cancel pending debounce
    onSearch(input.value);       // search with committed value immediately
  });

  // Submit on Enter — but not during composition
  input.addEventListener('keydown', (e) => {
    if (e.key === 'Enter' && !e.isComposing) {
      clearTimeout(debounceTimer);
      onSearch(input.value);
    }
  });
}
```

---

## React Implementation

React's synthetic events have a `nativeEvent.isComposing` property, but React itself does not fire `onChange` during composition in some older versions. In React 16+, `onChange` fires on every `input` event. Guard with `isComposing`:

```tsx
import { useState, useRef } from 'react';

function SearchInput({ onSearch }: { onSearch: (q: string) => void }) {
  const [value, setValue] = useState('');
  const isComposing = useRef(false);

  return (
    <input
      value={value}
      onChange={(e) => {
        setValue(e.target.value);
        if (!isComposing.current) {
          onSearch(e.target.value);
        }
      }}
      onCompositionStart={() => { isComposing.current = true; }}
      onCompositionEnd={(e) => {
        isComposing.current = false;
        onSearch((e.target as HTMLInputElement).value);
      }}
      onKeyDown={(e) => {
        if (e.key === 'Enter' && !e.nativeEvent.isComposing) {
          onSearch(value);
        }
      }}
    />
  );
}
```

---

## Angular Implementation

```typescript
import { Component, Output, EventEmitter } from '@angular/core';

@Component({
  selector: 'app-search-input',
  standalone: true,
  template: `
    <input
      type="search"
      [value]="query"
      (compositionstart)="onCompositionStart()"
      (compositionend)="onCompositionEnd($event)"
      (input)="onInput($event)"
      (keydown.enter)="onEnter($event)"
    />
  `
})
export class SearchInputComponent {
  @Output() search = new EventEmitter<string>();

  query = '';
  private composing = false;

  onCompositionStart() {
    this.composing = true;
  }

  onCompositionEnd(e: CompositionEvent) {
    this.composing = false;
    this.query = (e.target as HTMLInputElement).value;
    this.search.emit(this.query);
  }

  onInput(e: Event) {
    const input = e as InputEvent;
    if (input.isComposing || this.composing) return;
    this.query = (e.target as HTMLInputElement).value;
    this.search.emit(this.query);
  }

  onEnter(e: KeyboardEvent) {
    if (e.isComposing) return; // don't submit on IME Enter
    this.search.emit(this.query);
  }
}
```

---

## contenteditable and Rich-Text Editors

Composition events are especially important in contenteditable editors:

```javascript
const editor = document.querySelector('[contenteditable]');
let isComposing = false;

editor.addEventListener('compositionstart', () => { isComposing = true; });
editor.addEventListener('compositionend', () => {
  isComposing = false;
  syncContent(); // update your state model after composition
});

// Mutation observer to track content changes
const observer = new MutationObserver(() => {
  if (!isComposing) syncContent(); // skip during composition
});
observer.observe(editor, { subtree: true, characterData: true, childList: true });
```

Editors like Prosemirror and Tiptap handle this internally — but understanding why is important for debugging edge cases.

---

## Virtual Keyboard and Mobile IME

On mobile, every soft keyboard uses composition for many languages. The same events fire:
- Predictive text suggestions commit via `compositionend`
- Autocorrect replacements fire `compositionend` followed by `input`
- Voice input can fire composition events

Always test search/autocomplete on a real Android device with a CJK keyboard, or use Chrome DevTools device mode with a language that uses IME.

---

## Browser Behavior Differences

| Behavior | Chrome | Firefox | Safari |
|---|---|---|---|
| `input` fires during composition | Yes | Yes | Yes |
| `compositionend` fires before final `input` | Yes | No (fires after) | Yes |
| `e.isComposing` on `input` event | Yes | Yes | Yes |
| `e.isComposing` on `compositionend` | `false` | `false` | `false` |

The last row is important: `e.isComposing` is already `false` on the `compositionend` event itself — the composition is already done. Don't check `e.isComposing` inside `compositionend`; it's always `false` there.

---

## Interview Questions

**Q: What is an IME and why do composition events matter?**
A: An Input Method Editor is software that converts keystroke sequences to characters for languages with more characters than keyboard keys (Chinese, Japanese, Korean, Arabic, etc.). During composition, the user types intermediate phonetic characters that get replaced with final characters on commit. Without composition event guards, search-as-you-type fires on every intermediate keystroke — producing garbage queries and unnecessary API calls.

**Q: What's the correct way to detect when to trigger a search in an input that handles IME?**
A: Listen to `input` and skip if `event.isComposing` is true. Also listen to `compositionend` and fire the search immediately on commit (bypassing the debounce delay). And guard `keydown` Enter handling with `!e.isComposing` to avoid form submission when Enter is pressed to commit IME input.

**Q: Why is `compositionend` sometimes fired before the final `input` event and sometimes after?**
A: It's a browser implementation difference. Chrome fires `compositionend` then `input`; Firefox fires them in the opposite order. This is why relying on `event.isComposing` on the `input` event is more reliable than tracking state manually — the browser knows the correct value at the time of each event regardless of ordering.
