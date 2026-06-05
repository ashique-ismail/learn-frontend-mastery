# Implement a Typeahead / Autocomplete

## The Idea

**In plain English:** A typeahead (also called autocomplete) is a search box that shows you a list of matching suggestions while you type, updating instantly with each new letter so you can pick an option without finishing the full word.

**Real-world analogy:** Think of a librarian assistant standing next to a card catalog. You say the first word of a book title out loud, and the assistant immediately pulls out a short stack of matching cards for you to pick from — then you say another word and they swap in an even smaller stack.

- The words you speak = the characters you type into the input box
- The assistant listening and pulling cards = the debounced fetch function that requests matching results from the server
- The stack of cards shown to you = the dropdown list of suggestions rendered on screen

---

## Core Requirements

1. Show suggestions as the user types (debounced)
2. Keyboard navigation through results (↑↓ arrows, Enter to select, Escape to close)
3. Accessible (ARIA combobox pattern)
4. Handle loading/error states
5. Avoid race conditions (cancel stale requests)

---

## Custom Hook

```ts
function useAutocomplete<T>({
  fetchSuggestions,
  debounceMs = 300,
  minLength = 1,
}: {
  fetchSuggestions: (query: string) => Promise<T[]>;
  debounceMs?: number;
  minLength?: number;
}) {
  const [query, setQuery] = useState('');
  const [suggestions, setSuggestions] = useState<T[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  const [activeIndex, setActiveIndex] = useState(-1);
  const [isOpen, setIsOpen] = useState(false);

  useEffect(() => {
    if (query.length < minLength) {
      setSuggestions([]);
      setIsOpen(false);
      return;
    }

    const controller = new AbortController();
    const timer = setTimeout(async () => {
      setLoading(true);
      setError(null);
      try {
        const results = await fetchSuggestions(query);
        if (!controller.signal.aborted) {
          setSuggestions(results);
          setActiveIndex(-1);
          setIsOpen(results.length > 0);
        }
      } catch (err) {
        if (!controller.signal.aborted) {
          setError(err as Error);
          setSuggestions([]);
        }
      } finally {
        if (!controller.signal.aborted) setLoading(false);
      }
    }, debounceMs);

    return () => {
      controller.abort();
      clearTimeout(timer);
    };
  }, [query, debounceMs, minLength, fetchSuggestions]);

  function handleKeyDown(e: React.KeyboardEvent) {
    if (!isOpen) return;

    if (e.key === 'ArrowDown') {
      e.preventDefault();
      setActiveIndex(i => Math.min(i + 1, suggestions.length - 1));
    } else if (e.key === 'ArrowUp') {
      e.preventDefault();
      setActiveIndex(i => Math.max(i - 1, -1));
    } else if (e.key === 'Escape') {
      setIsOpen(false);
      setActiveIndex(-1);
    }
  }

  function selectSuggestion(index: number) {
    setIsOpen(false);
    setActiveIndex(-1);
    return suggestions[index];
  }

  return {
    query, setQuery,
    suggestions, loading, error,
    activeIndex, isOpen, setIsOpen,
    handleKeyDown, selectSuggestion,
  };
}
```

---

## Component with ARIA Combobox Pattern

```tsx
interface Country { code: string; name: string; }

function CountrySearch() {
  const inputRef = useRef<HTMLInputElement>(null);
  const listboxId = useId();
  const [selectedCountry, setSelectedCountry] = useState<Country | null>(null);

  const {
    query, setQuery,
    suggestions, loading, error,
    activeIndex, isOpen,
    handleKeyDown, selectSuggestion,
  } = useAutocomplete<Country>({
    fetchSuggestions: (q) => api.searchCountries(q),
    debounceMs: 300,
    minLength: 1,
  });

  function handleSelect(index: number) {
    const country = selectSuggestion(index);
    setSelectedCountry(country);
    setQuery(country.name);
    inputRef.current?.focus();
  }

  return (
    <div style={{ position: 'relative' }}>
      {/* Input — role="combobox" signals autocomplete to screen readers */}
      <input
        ref={inputRef}
        role="combobox"
        aria-haspopup="listbox"
        aria-expanded={isOpen}
        aria-controls={listboxId}
        aria-activedescendant={
          activeIndex >= 0 ? `option-${activeIndex}` : undefined
        }
        aria-autocomplete="list"
        value={query}
        onChange={e => { setQuery(e.target.value); setSelectedCountry(null); }}
        onKeyDown={(e) => {
          handleKeyDown(e);
          if (e.key === 'Enter' && activeIndex >= 0) handleSelect(activeIndex);
        }}
        placeholder="Search countries..."
      />

      {loading && <span aria-live="polite" className="sr-only">Searching...</span>}
      {error && <span role="alert">{error.message}</span>}

      {/* Dropdown list */}
      {isOpen && (
        <ul
          id={listboxId}
          role="listbox"
          aria-label="Countries"
          style={{ position: 'absolute', top: '100%', left: 0, right: 0, zIndex: 10 }}
        >
          {suggestions.map((country, index) => (
            <li
              key={country.code}
              id={`option-${index}`}
              role="option"
              aria-selected={index === activeIndex}
              onMouseDown={(e) => { e.preventDefault(); handleSelect(index); }}
              style={{ background: index === activeIndex ? '#e0e7ff' : 'white' }}
            >
              {country.name}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

---

## Key Implementation Details

### Race Condition Prevention

The `AbortController` in the hook cancels stale requests — if the user types quickly, only the latest request's results are used. The debounce timer also ensures requests aren't fired for every keypress.

### `onMouseDown` vs `onClick` for List Items

Using `onClick` on list items causes a bug: the input's `onBlur` fires before `onClick`, closing the dropdown before the click registers. `onMouseDown` fires before `onBlur`, so `e.preventDefault()` prevents the blur while the selection is processed.

### `aria-activedescendant`

Points to the ID of the currently highlighted option. Screen readers announce the highlighted item as the user arrows through — without this, keyboard navigation is inaccessible.

---

## Common Interview Questions

**Q: How do you prevent the dropdown from closing when clicking a suggestion?**
`e.preventDefault()` in `onMouseDown` prevents the input from losing focus (which triggers `onBlur`). The `onClick` approach fails here because blur fires first.

**Q: How would you add caching to avoid repeated requests?**
Store results in a `Map<string, T[]>` keyed by query string. Check cache first before fetching. Optionally add TTL — cache entries expire after N milliseconds.

**Q: What's the ARIA combobox pattern?**
`role="combobox"` on the input, `aria-haspopup="listbox"`, `aria-expanded`, `aria-controls` pointing to the listbox ID, `aria-activedescendant` pointing to the active option ID, and `aria-autocomplete="list"` to indicate filtering behavior.
