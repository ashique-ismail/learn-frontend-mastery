# System Design: Autocomplete/Typeahead

## The Idea

**In plain English:** Autocomplete is the feature that shows you a dropdown list of suggestions while you type — like when you start typing "pizza" into Google and it instantly shows you "pizza near me" and "pizza recipes" before you finish. The system predicts what you want and offers options so you don't have to type the whole thing.

**Real-world analogy:** Imagine you walk into a library and tell the librarian the first few words of a book title. The librarian has a huge index-card box sorted alphabetically, so they instantly flip to that section and read out the closest matches — without searching the entire collection every time.

- The librarian's index-card box = the cached list of suggestions stored in memory
- The moment you pause speaking = the debounce delay (waiting for you to stop typing before searching)
- The librarian reading out matching cards = the dropdown list of results shown on screen

---

## Overview

Design a scalable autocomplete system with debouncing, caching, keyboard navigation, accessibility, and fuzzy matching. This document covers architecture, implementation strategies, and optimization techniques for building a production-ready search autocomplete.

## Requirements Analysis

### Functional Requirements

1. Show suggestions as user types
2. Keyboard navigation (arrow keys, enter, escape)
3. Highlight matching text
4. Recent searches and popular suggestions
5. Categories/grouping of results
6. Rich content (avatars, descriptions)
7. Mobile and desktop support

### Non-Functional Requirements

1. Response time < 100ms
2. Debounce input (300ms)
3. Support 10M+ search queries/day
4. Handle slow/unreliable networks
5. Accessible (WCAG 2.1 AA)
6. Works offline with cached results
7. < 50KB bundle size for component

### Scale Considerations

- 10M daily searches
- 100K suggestions in database
- Average 5 characters per search
- 50 requests/second peak
- 3-10 suggestions per query

## Architecture

```typescript
// Core autocomplete service with debouncing
@Injectable({ providedIn: 'root' })
export class AutocompleteService {
  private readonly apiUrl = '/api/search/autocomplete';
  private cache = new Map<string, Suggestion[]>();

  constructor(
    private http: HttpClient,
    private cacheService: CacheService
  ) {}

  search(query: string, options?: SearchOptions): Observable<Suggestion[]> {
    // Check cache first
    const cacheKey = this.getCacheKey(query, options);
    const cached = this.cache.get(cacheKey);
    
    if (cached) {
      return of(cached);
    }

    // Fetch from API
    return this.http.get<Suggestion[]>(this.apiUrl, {
      params: {
        q: query,
        limit: options?.limit || 10,
        ...options
      }
    }).pipe(
      tap(results => this.cache.set(cacheKey, results)),
      catchError(() => of([]))
    );
  }

  private getCacheKey(query: string, options?: SearchOptions): string {
    return `${query}-${JSON.stringify(options || {})}`;
  }
}

// Autocomplete component with full features
@Component({
  selector: 'app-autocomplete',
  standalone: true,
  template: `
    <div class="autocomplete-container" 
         [class.open]="showSuggestions">
      
      <!-- Input field -->
      <input
        #input
        type="text"
        class="autocomplete-input"
        [attr.aria-label]="label"
        [attr.aria-expanded]="showSuggestions"
        [attr.aria-controls]="listId"
        [attr.aria-activedescendant]="activeDescendantId"
        role="combobox"
        autocomplete="off"
        [placeholder]="placeholder"
        [(ngModel)]="query"
        (input)="onInput($event)"
        (keydown)="onKeyDown($event)"
        (focus)="onFocus()"
        (blur)="onBlur()">

      <!-- Clear button -->
      <button
        *ngIf="query"
        class="clear-button"
        aria-label="Clear search"
        (click)="clear()">
        ×
      </button>

      <!-- Loading indicator -->
      <div class="loading-indicator" *ngIf="loading">
        <app-spinner size="small"></app-spinner>
      </div>

      <!-- Suggestions dropdown -->
      <ul
        *ngIf="showSuggestions && suggestions.length > 0"
        [id]="listId"
        class="suggestions-list"
        role="listbox">
        
        <!-- Recent searches -->
        <li *ngIf="recentSearches.length > 0" class="section-header">
          Recent Searches
        </li>
        <li
          *ngFor="let recent of recentSearches; let i = index"
          [id]="getOptionId(i)"
          class="suggestion-item recent"
          [class.active]="activeIndex === i"
          role="option"
          [attr.aria-selected]="activeIndex === i"
          (mousedown)="selectSuggestion(recent)"
          (mouseenter)="activeIndex = i">
          
          <span class="icon">🕒</span>
          <span class="text" [innerHTML]="highlight(recent.text, query)"></span>
          <button class="remove-recent" (click)="removeRecent(recent, $event)">×</button>
        </li>

        <!-- Divider -->
        <li *ngIf="recentSearches.length > 0" class="divider"></li>

        <!-- Search suggestions -->
        <li
          *ngFor="let suggestion of suggestions; let i = index"
          [id]="getOptionId(i + recentSearches.length)"
          class="suggestion-item"
          [class.active]="activeIndex === (i + recentSearches.length)"
          role="option"
          [attr.aria-selected]="activeIndex === (i + recentSearches.length)"
          (mousedown)="selectSuggestion(suggestion)"
          (mouseenter)="activeIndex = i + recentSearches.length">
          
          <img
            *ngIf="suggestion.avatar"
            [src]="suggestion.avatar"
            [alt]="suggestion.text"
            class="avatar">
          
          <div class="suggestion-content">
            <span class="text" [innerHTML]="highlight(suggestion.text, query)"></span>
            <span *ngIf="suggestion.description" class="description">
              {{ suggestion.description }}
            </span>
          </div>

          <span *ngIf="suggestion.category" class="category">
            {{ suggestion.category }}
          </span>
        </li>
      </ul>

      <!-- No results -->
      <div 
        *ngIf="showSuggestions && !loading && suggestions.length === 0 && query"
        class="no-results">
        No results found for "{{ query }}"
      </div>
    </div>
  `,
  styles: [`
    .autocomplete-container {
      position: relative;
      width: 100%;
    }

    .autocomplete-input {
      width: 100%;
      padding: 12px 40px 12px 16px;
      font-size: 16px;
      border: 2px solid #e0e0e0;
      border-radius: 8px;
      transition: border-color 0.2s;
    }

    .autocomplete-input:focus {
      outline: none;
      border-color: #1976d2;
    }

    .suggestions-list {
      position: absolute;
      top: calc(100% + 4px);
      left: 0;
      right: 0;
      max-height: 400px;
      overflow-y: auto;
      background: white;
      border: 1px solid #e0e0e0;
      border-radius: 8px;
      box-shadow: 0 4px 12px rgba(0,0,0,0.1);
      list-style: none;
      padding: 0;
      margin: 0;
      z-index: 1000;
    }

    .suggestion-item {
      padding: 12px 16px;
      cursor: pointer;
      display: flex;
      align-items: center;
      gap: 12px;
      transition: background-color 0.2s;
    }

    .suggestion-item:hover,
    .suggestion-item.active {
      background-color: #f5f5f5;
    }

    .suggestion-item .text mark {
      background-color: #fff59d;
      font-weight: 600;
    }

    .clear-button {
      position: absolute;
      right: 12px;
      top: 50%;
      transform: translateY(-50%);
      background: none;
      border: none;
      font-size: 24px;
      cursor: pointer;
      color: #666;
    }
  `]
})
export class AutocompleteComponent implements OnInit, OnDestroy {
  @Input() placeholder = 'Search...';
  @Input() label = 'Search';
  @Input() minLength = 2;
  @Input() debounceTime = 300;
  @Input() maxSuggestions = 10;
  @Output() selected = new EventEmitter<Suggestion>();

  @ViewChild('input') inputElement!: ElementRef<HTMLInputElement>;

  query = '';
  suggestions: Suggestion[] = [];
  recentSearches: Suggestion[] = [];
  showSuggestions = false;
  loading = false;
  activeIndex = -1;

  listId = `suggestions-${Math.random().toString(36).substr(2, 9)}`;
  
  private destroy$ = new Subject<void>();
  private searchSubject = new Subject<string>();

  constructor(
    private autocompleteService: AutocompleteService,
    private recentSearchesService: RecentSearchesService
  ) {}

  ngOnInit() {
    // Setup debounced search
    this.searchSubject.pipe(
      debounceTime(this.debounceTime),
      distinctUntilChanged(),
      filter(query => query.length >= this.minLength),
      tap(() => this.loading = true),
      switchMap(query => this.autocompleteService.search(query, {
        limit: this.maxSuggestions
      })),
      takeUntil(this.destroy$)
    ).subscribe(suggestions => {
      this.suggestions = suggestions;
      this.loading = false;
      this.showSuggestions = true;
    });

    // Load recent searches
    this.recentSearches = this.recentSearchesService.getRecent(5);
  }

  onInput(event: Event) {
    const target = event.target as HTMLInputElement;
    const query = target.value;

    if (query.length < this.minLength) {
      this.suggestions = [];
      this.showSuggestions = false;
      this.loading = false;
      return;
    }

    this.searchSubject.next(query);
  }

  onKeyDown(event: KeyboardEvent) {
    switch (event.key) {
      case 'ArrowDown':
        event.preventDefault();
        this.moveSelection(1);
        break;
      case 'ArrowUp':
        event.preventDefault();
        this.moveSelection(-1);
        break;
      case 'Enter':
        event.preventDefault();
        if (this.activeIndex >= 0) {
          const allSuggestions = [...this.recentSearches, ...this.suggestions];
          this.selectSuggestion(allSuggestions[this.activeIndex]);
        }
        break;
      case 'Escape':
        this.showSuggestions = false;
        this.activeIndex = -1;
        break;
    }
  }

  moveSelection(direction: number) {
    const totalItems = this.recentSearches.length + this.suggestions.length;
    
    if (totalItems === 0) return;

    this.activeIndex += direction;

    if (this.activeIndex < 0) {
      this.activeIndex = totalItems - 1;
    } else if (this.activeIndex >= totalItems) {
      this.activeIndex = 0;
    }

    // Scroll into view
    const optionElement = document.getElementById(this.getOptionId(this.activeIndex));
    optionElement?.scrollIntoView({ block: 'nearest' });
  }

  selectSuggestion(suggestion: Suggestion) {
    this.query = suggestion.text;
    this.showSuggestions = false;
    this.activeIndex = -1;
    
    this.recentSearchesService.add(suggestion);
    this.selected.emit(suggestion);
  }

  onFocus() {
    if (this.query.length >= this.minLength || this.recentSearches.length > 0) {
      this.showSuggestions = true;
    }
  }

  onBlur() {
    // Delay to allow click events on suggestions
    setTimeout(() => {
      this.showSuggestions = false;
      this.activeIndex = -1;
    }, 200);
  }

  clear() {
    this.query = '';
    this.suggestions = [];
    this.showSuggestions = false;
    this.activeIndex = -1;
    this.inputElement.nativeElement.focus();
  }

  removeRecent(recent: Suggestion, event: Event) {
    event.stopPropagation();
    this.recentSearchesService.remove(recent);
    this.recentSearches = this.recentSearchesService.getRecent(5);
  }

  highlight(text: string, query: string): string {
    if (!query) return text;
    
    const regex = new RegExp(`(${this.escapeRegex(query)})`, 'gi');
    return text.replace(regex, '<mark>$1</mark>');
  }

  private escapeRegex(text: string): string {
    return text.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
  }

  getOptionId(index: number): string {
    return `${this.listId}-option-${index}`;
  }

  get activeDescendantId(): string {
    return this.activeIndex >= 0 ? this.getOptionId(this.activeIndex) : '';
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// Recent searches service
@Injectable({ providedIn: 'root' })
export class RecentSearchesService {
  private readonly storageKey = 'recent_searches';
  private readonly maxRecent = 10;

  getRecent(limit?: number): Suggestion[] {
    const stored = localStorage.getItem(this.storageKey);
    const recent = stored ? JSON.parse(stored) : [];
    return limit ? recent.slice(0, limit) : recent;
  }

  add(suggestion: Suggestion) {
    let recent = this.getRecent();
    
    // Remove duplicate
    recent = recent.filter(r => r.text !== suggestion.text);
    
    // Add to beginning
    recent.unshift(suggestion);
    
    // Limit size
    recent = recent.slice(0, this.maxRecent);
    
    localStorage.setItem(this.storageKey, JSON.stringify(recent));
  }

  remove(suggestion: Suggestion) {
    let recent = this.getRecent();
    recent = recent.filter(r => r.text !== suggestion.text);
    localStorage.setItem(this.storageKey, JSON.stringify(recent));
  }

  clear() {
    localStorage.removeItem(this.storageKey);
  }
}

// Fuzzy matching implementation
@Injectable({ providedIn: 'root' })
export class FuzzySearchService {
  search(query: string, items: string[]): string[] {
    return items
      .map(item => ({
        item,
        score: this.calculateScore(query, item)
      }))
      .filter(result => result.score > 0)
      .sort((a, b) => b.score - a.score)
      .map(result => result.item);
  }

  private calculateScore(query: string, target: string): number {
    query = query.toLowerCase();
    target = target.toLowerCase();

    // Exact match
    if (target === query) return 100;

    // Starts with query
    if (target.startsWith(query)) return 90;

    // Contains query
    if (target.includes(query)) return 80;

    // Fuzzy match
    let score = 0;
    let queryIndex = 0;

    for (let i = 0; i < target.length && queryIndex < query.length; i++) {
      if (target[i] === query[queryIndex]) {
        score += 1;
        queryIndex++;
      }
    }

    // All characters matched
    if (queryIndex === query.length) {
      return 50 + (score / target.length) * 20;
    }

    return 0;
  }
}
```

## Advanced Features

### Throttling vs Debouncing

```typescript
// Debouncing (recommended): Wait for pause in typing
const debouncedSearch$ = searchInput$.pipe(
  debounceTime(300),
  distinctUntilChanged()
);

// Throttling: Limit request frequency
const throttledSearch$ = searchInput$.pipe(
  throttleTime(300),
  distinctUntilChanged()
);

// Hybrid: Combine both
const hybridSearch$ = searchInput$.pipe(
  debounceTime(300),
  throttleTime(1000),
  distinctUntilChanged()
);
```

### Caching Strategies

```typescript
// LRU Cache implementation
class LRUCache<K, V> {
  private cache = new Map<K, V>();
  private maxSize: number;

  constructor(maxSize: number = 100) {
    this.maxSize = maxSize;
  }

  get(key: K): V | undefined {
    if (!this.cache.has(key)) return undefined;
    
    // Move to end (most recently used)
    const value = this.cache.get(key)!;
    this.cache.delete(key);
    this.cache.set(key, value);
    
    return value;
  }

  set(key: K, value: V): void {
    // Remove if exists
    if (this.cache.has(key)) {
      this.cache.delete(key);
    }

    // Add to end
    this.cache.set(key, value);

    // Remove oldest if over capacity
    if (this.cache.size > this.maxSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
  }

  clear(): void {
    this.cache.clear();
  }
}
```

## Key Takeaways

1. **Debounce Input**: Use 300ms debounce to reduce API calls while maintaining responsiveness.

2. **Keyboard Navigation**: Implement full keyboard support (arrows, enter, escape) for accessibility and UX.

3. **ARIA Attributes**: Use proper ARIA roles (combobox, listbox, option) and attributes for screen readers.

4. **Caching Strategy**: Implement multi-level caching (memory, localStorage) to reduce API calls and improve performance.

5. **Recent Searches**: Show recent searches when input is empty for better UX and engagement.

6. **Highlight Matches**: Visually highlight matching text in suggestions for better scannability.

7. **Loading States**: Show loading indicator during search to provide feedback on long-running queries.

8. **Error Handling**: Gracefully handle API failures, network errors, and empty results.

9. **Mobile Optimization**: Ensure touch-friendly target sizes (44px minimum) and appropriate font sizes.

10. **Performance**: Use virtual scrolling for large result sets and lazy load suggestion details.

## Red Flags to Avoid

- Not debouncing input (too many API calls)
- Missing keyboard navigation
- Poor accessibility (missing ARIA attributes)
- No caching strategy
- Blocking UI during search
- Not handling edge cases (empty query, special characters)
- Poor mobile experience
- Missing loading/error states
- Not escaping HTML in suggestions
- Memory leaks from unsubscribed observables

## Trade-offs

**Client-side vs Server-side Filtering:**

- Client: Faster, less server load, limited dataset
- Server: Handles large datasets, more flexible, higher latency

**Prefix vs Fuzzy Matching:**

- Prefix: Faster, simpler, less flexible
- Fuzzy: Better UX, more complex, slower

**Dropdown vs Inline Suggestions:**

- Dropdown: More space, better for many results
- Inline: Faster to select, limited space

## Interview Talking Points

- Explain debouncing vs throttling trade-offs
- Discuss caching invalidation strategies
- Describe accessibility implementation
- Explain fuzzy matching algorithms
- Discuss API optimization techniques
- Describe keyboard navigation implementation
- Explain error handling and retry logic
- Discuss mobile vs desktop differences
- Describe performance optimization approaches
- Explain testing strategies for autocomplete
