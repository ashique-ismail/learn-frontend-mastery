# Debounced Search with Cancellation

## The Idea

**In plain English:** The user types in a search box. You want to fire an API request only after they pause typing — not on every keystroke. But there is a second, deeper problem: if the user types "re", then "rea", then "reac", three requests fly out. The "rea" response might arrive *after* the "reac" response (network is unpredictable), making the UI show stale results for "rea" even though the user has already typed "reac". This is a race condition, and debouncing alone does not fix it.

**Real-world analogy:** Imagine you shout search terms to a librarian across a large hall.

- **Debouncing** is waiting until you stop shouting for two seconds before the librarian starts walking to the shelves. No point walking for every half-word.
- **Cancellation** is the librarian throwing away any previous sticky note if you give them a new one. Even if the previous errand runner comes back first with the wrong books, the librarian ignores them because the sticky note was discarded.
- **AbortController** is that "discard the sticky note" mechanism — it tells the in-flight network request "do not bother delivering your results, they are no longer wanted."
- **`useRef` for the latest request ID** is the librarian's way of knowing which sticky note is current.

---

## Learning Objectives

- Understand why debouncing alone does not prevent stale results (race conditions)
- Use `AbortController` and `signal` to cancel in-flight `fetch` requests
- Use `useRef` to track the latest request and ignore outdated responses
- Implement correct pending / error / empty states in the UI
- Replicate the same pattern in Angular with RxJS `switchMap` (which cancels automatically)
- Avoid the most common pitfalls: missing cleanup, re-triggering on unmount, error swallowing

---

## Why CSS Alone Isn't Enough

Not applicable — this is a pure JavaScript/framework concern. CSS only owns the visual loading spinner.

---

## HTML & CSS Foundation

### Loading and State Styles

```html
<div class="search-wrapper">
  <input
    type="search"
    class="search-input"
    placeholder="Search..."
    aria-label="Search"
    aria-controls="search-results"
    aria-autocomplete="list"
  />
  <!-- Spinner shown during fetch -->
  <span class="search-spinner" aria-hidden="true" hidden></span>
</div>

<div
  id="search-results"
  role="listbox"
  aria-label="Search results"
  aria-live="polite"
  aria-busy="false"
>
  <!-- Results injected here -->
</div>
```

```css
:root {
  --search-radius: 6px;
  --search-border: #d1d5db;
  --search-focus-ring: #3b82f6;
  --result-hover: #f3f4f6;
}

.search-wrapper {
  position: relative;
  display: inline-flex;
  align-items: center;
  width: 100%;
  max-width: 480px;
}

.search-input {
  width: 100%;
  padding: 0.625rem 2.5rem 0.625rem 0.875rem;
  border: 1px solid var(--search-border);
  border-radius: var(--search-radius);
  font-size: 1rem;
  outline: none;
  transition: border-color 0.15s, box-shadow 0.15s;
}

.search-input:focus {
  border-color: var(--search-focus-ring);
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.25);
}

/* Spinner positioned inside the input on the right */
.search-spinner {
  position: absolute;
  right: 0.75rem;
  width: 16px;
  height: 16px;
  border: 2px solid #e5e7eb;
  border-top-color: #6b7280;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
}

@keyframes spin { to { transform: rotate(360deg); } }

/* Result list */
#search-results {
  margin-top: 4px;
  border: 1px solid var(--search-border);
  border-radius: var(--search-radius);
  background: #fff;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
  overflow: hidden;
}

[role="option"] {
  padding: 0.625rem 0.875rem;
  cursor: pointer;
  border-bottom: 1px solid #f3f4f6;
}

[role="option"]:hover,
[role="option"][aria-selected="true"] {
  background: var(--result-hover);
}

.search-status {
  padding: 0.75rem 0.875rem;
  color: #6b7280;
  font-size: 0.875rem;
}
```

---

## React Implementation

### The Hook

```tsx
// useDebounceSearch.ts
import { useState, useEffect, useRef, useCallback } from 'react';

interface SearchResult {
  id: string;
  title: string;
  description: string;
}

interface SearchState {
  results: SearchResult[];
  isLoading: boolean;
  error: string | null;
  query: string;
}

const DEBOUNCE_MS = 300;

async function fetchResults(
  query: string,
  signal: AbortSignal
): Promise<SearchResult[]> {
  const res = await fetch(`/api/search?q=${encodeURIComponent(query)}`, { signal });
  if (!res.ok) throw new Error(`Search failed: ${res.status}`);
  return res.json();
}

export function useDebounceSearch() {
  const [state, setState] = useState<SearchState>({
    results: [],
    isLoading: false,
    error: null,
    query: '',
  });

  // Holds the AbortController for the in-flight request.
  // useRef keeps a stable reference across renders without triggering re-renders.
  const abortControllerRef = useRef<AbortController | null>(null);
  const debounceTimerRef = useRef<ReturnType<typeof setTimeout> | null>(null);

  const search = useCallback((rawQuery: string) => {
    const query = rawQuery.trim();
    setState(prev => ({ ...prev, query: rawQuery }));

    // Clear any pending debounce timer
    if (debounceTimerRef.current) {
      clearTimeout(debounceTimerRef.current);
    }

    // Cancel any in-flight request immediately when the user types
    if (abortControllerRef.current) {
      abortControllerRef.current.abort();
      abortControllerRef.current = null;
    }

    if (!query) {
      setState(prev => ({ ...prev, results: [], isLoading: false, error: null }));
      return;
    }

    setState(prev => ({ ...prev, isLoading: true, error: null }));

    debounceTimerRef.current = setTimeout(async () => {
      // Create a fresh controller for this request
      const controller = new AbortController();
      abortControllerRef.current = controller;

      try {
        const results = await fetchResults(query, controller.signal);
        // Only update state if this controller is still the current one
        // (i.e. no newer request has overwritten abortControllerRef)
        if (abortControllerRef.current === controller) {
          setState(prev => ({ ...prev, results, isLoading: false }));
        }
      } catch (err) {
        if (err instanceof Error && err.name === 'AbortError') {
          // Request was intentionally cancelled — do nothing
          return;
        }
        if (abortControllerRef.current === controller) {
          setState(prev => ({
            ...prev,
            error: err instanceof Error ? err.message : 'Unknown error',
            isLoading: false,
          }));
        }
      }
    }, DEBOUNCE_MS);
  }, []);

  // Cleanup on unmount: cancel any pending timer and in-flight request
  useEffect(() => {
    return () => {
      if (debounceTimerRef.current) clearTimeout(debounceTimerRef.current);
      if (abortControllerRef.current) abortControllerRef.current.abort();
    };
  }, []);

  return { ...state, search };
}
```

### The Component

```tsx
// SearchBox.tsx
import { useId, useRef } from 'react';
import { useDebounceSearch } from './useDebounceSearch';

export function SearchBox() {
  const { results, isLoading, error, query, search } = useDebounceSearch();
  const listboxId = useId();
  const inputRef = useRef<HTMLInputElement>(null);

  const isEmpty = !isLoading && !error && query.trim() && results.length === 0;

  return (
    <div style={{ position: 'relative', maxWidth: 480 }}>
      <div className="search-wrapper">
        <input
          ref={inputRef}
          type="search"
          className="search-input"
          placeholder="Search..."
          aria-label="Search"
          aria-controls={listboxId}
          aria-autocomplete="list"
          aria-busy={isLoading}
          onChange={e => search(e.target.value)}
        />
        {isLoading && (
          <span className="search-spinner" aria-hidden="true" />
        )}
      </div>

      {(results.length > 0 || isEmpty || error) && (
        <div
          id={listboxId}
          role="listbox"
          aria-label="Search results"
          aria-live="polite"
          aria-busy={isLoading}
        >
          {error && (
            <p className="search-status" role="alert">{error}</p>
          )}
          {isEmpty && (
            <p className="search-status">No results for "{query}"</p>
          )}
          {results.map(r => (
            <div
              key={r.id}
              role="option"
              aria-selected="false"
              tabIndex={0}
              onClick={() => console.log('Selected:', r.id)}
              onKeyDown={e => e.key === 'Enter' && console.log('Selected:', r.id)}
            >
              <strong>{r.title}</strong>
              <p style={{ margin: 0, fontSize: '0.875rem', color: '#6b7280' }}>
                {r.description}
              </p>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

---

## Angular Implementation

Angular's RxJS `switchMap` operator handles cancellation automatically — it unsubscribes from the previous inner observable when a new value arrives, making it the idiomatic equivalent of `AbortController` + debounce.

### Service

```typescript
// search.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

export interface SearchResult {
  id: string;
  title: string;
  description: string;
}

@Injectable({ providedIn: 'root' })
export class SearchService {
  private http = inject(HttpClient);

  search(query: string): Observable<SearchResult[]> {
    return this.http.get<SearchResult[]>(`/api/search?q=${encodeURIComponent(query)}`);
  }
}
```

### Component

```typescript
// search-box.component.ts
import {
  Component, OnInit, OnDestroy, signal, inject
} from '@angular/core';
import { FormControl, ReactiveFormsModule } from '@angular/forms';
import { AsyncPipe, NgFor, NgIf } from '@angular/common';
import {
  Subject, switchMap, debounceTime, distinctUntilChanged,
  catchError, of, tap, startWith, map
} from 'rxjs';
import { takeUntilDestroyed, DestroyRef } from '@angular/core/rxjs-interop';
import { SearchService, SearchResult } from './search.service';

@Component({
  selector: 'app-search-box',
  standalone: true,
  imports: [ReactiveFormsModule, AsyncPipe, NgFor, NgIf],
  template: `
    <div class="search-wrapper">
      <input
        type="search"
        class="search-input"
        placeholder="Search..."
        aria-label="Search"
        [formControl]="searchControl"
        [attr.aria-busy]="isLoading()"
      />
      @if (isLoading()) {
        <span class="search-spinner" aria-hidden="true"></span>
      }
    </div>

    <div role="listbox" aria-label="Search results" aria-live="polite">
      @if (error()) {
        <p class="search-status" role="alert">{{ error() }}</p>
      }
      @if (isEmpty()) {
        <p class="search-status">No results for "{{ searchControl.value }}"</p>
      }
      @for (result of results(); track result.id) {
        <div role="option" aria-selected="false" tabindex="0">
          <strong>{{ result.title }}</strong>
          <p>{{ result.description }}</p>
        </div>
      }
    </div>
  `,
})
export class SearchBoxComponent implements OnInit {
  private searchService = inject(SearchService);
  private destroyRef = inject(DestroyRef);

  searchControl = new FormControl('', { nonNullable: true });

  results  = signal<SearchResult[]>([]);
  isLoading = signal(false);
  error    = signal<string | null>(null);
  isEmpty  = signal(false);

  ngOnInit() {
    this.searchControl.valueChanges.pipe(
      debounceTime(300),           // wait 300ms after last keystroke
      distinctUntilChanged(),      // skip if same value
      tap(query => {
        this.isLoading.set(!!query.trim());
        this.error.set(null);
        this.isEmpty.set(false);
        if (!query.trim()) this.results.set([]);
      }),
      // switchMap CANCELS the previous inner observable when a new value arrives
      // This is the RxJS equivalent of AbortController
      switchMap(query => {
        if (!query.trim()) return of([]);
        return this.searchService.search(query).pipe(
          catchError(err => {
            this.error.set(err.message ?? 'Search failed');
            return of([] as SearchResult[]);
          })
        );
      }),
      takeUntilDestroyed(this.destroyRef),
    ).subscribe(results => {
      this.isLoading.set(false);
      this.results.set(results);
      this.isEmpty.set(
        results.length === 0 && !!this.searchControl.value.trim() && !this.error()
      );
    });
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Input has `aria-label` | `aria-label="Search"` on the `<input>` |
| Input announces its loading state | `aria-busy` toggled during fetch |
| Input is linked to result list | `aria-controls` pointing to listbox `id` |
| Input declares autocomplete behaviour | `aria-autocomplete="list"` |
| Result list uses `role="listbox"` | Semantically correct for combobox pattern |
| Results announced to screen readers | `aria-live="polite"` on result container |
| Error announced immediately | `role="alert"` on error paragraph |
| Empty-state is communicated | Visible text inside the live region |
| Keyboard navigation inside results | `tabIndex={0}` and `onKeyDown` Enter handler |

---

## Production Pitfalls

**1. Debounce without cancellation still produces stale results**
A 300ms debounce only reduces the number of requests — it cannot guarantee order. Always pair debounce with `AbortController` (React) or `switchMap` (Angular).

**2. Not aborting on unmount**
If the component unmounts while a request is in flight (e.g. the user navigates away), the `setState` call in the `.then()` block will fire on an unmounted component. The cleanup `useEffect` that calls `abort()` prevents this.

**3. Swallowing non-abort errors**
`AbortError` should be silently ignored. All other errors should surface to the user. The `if (err.name === 'AbortError') return;` guard is essential.

**4. `distinctUntilChanged` is missing**
Without it, pressing Space then Backspace (returning to the original value) re-fires the same query. `distinctUntilChanged()` in RxJS or a manual comparison in React avoids redundant requests.

**5. Race condition with `useRef` comparison**
The pattern `if (abortControllerRef.current === controller)` is the correct guard. An alternative is a counter (`requestIdRef`), incrementing on each new request and checking in the callback that the counter hasn't moved.

**6. Overly aggressive debounce delays hurt perceived performance**
300ms is the sweet spot for search. Values above 500ms make the UI feel sluggish. Values below 150ms cancel and re-issue so quickly that the cancellation overhead outweighs the benefit.

---

## Interview Angle

**Q: "A user types quickly in a search box. Request for 'rea' comes back after request for 'reac'. What do you do?"**

Strong answer: This is a race condition. Debouncing reduces how many requests fire but does not guarantee delivery order. The solution is `AbortController`: on every new keystroke (or after the debounce fires), call `previousController.abort()` before creating a new one. In the fetch callback, additionally guard with `if (abortControllerRef.current === thisController)` before updating state. In Angular, `switchMap` cancels the previous inner `Observable` automatically — it is purpose-built for this.

**Follow-up: "Why use `useRef` instead of `useState` for the AbortController?"**

`useRef` is synchronous and does not trigger a re-render. `AbortController` is implementation detail — it is not display state. Storing it in `useState` would cause a spurious re-render on every keystroke, potentially re-triggering the effect. `useRef` gives a stable mutable container across renders with zero render cost.

**Follow-up: "What happens if the component unmounts while the request is in flight?"**

Without cleanup, `setState` fires on an unmounted component, producing a React warning and potential memory leak. The cleanup function in `useEffect` calls `controller.abort()` and `clearTimeout()`, preventing both the state update and the request from completing. Angular's `takeUntilDestroyed(destroyRef)` achieves the same result by completing the observable stream on destroy.
