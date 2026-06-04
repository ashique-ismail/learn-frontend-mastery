# Accessible Single-Page Applications (SPAs)

## Table of Contents
- [Introduction](#introduction)
- [Why SPA Accessibility is Challenging](#why-spa-accessibility-is-challenging)
- [Client-Side Routing](#client-side-routing)
- [Focus Management](#focus-management)
- [Announcements and Live Regions](#announcements-and-live-regions)
- [Loading States](#loading-states)
- [Common Patterns](#common-patterns)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Single-Page Applications (SPAs) provide fluid, app-like experiences by updating content dynamically without full page reloads. However, this dynamic behavior creates unique accessibility challenges that don't exist in traditional multi-page websites. Screen reader users, keyboard users, and those relying on assistive technologies need special considerations to ensure SPAs are fully accessible.

```
Traditional Page vs SPA Navigation
===================================

Traditional Multi-Page App:
User clicks link → Full page reload → Browser handles:
  ├─ Focus moves to top
  ├─ Screen reader announces page
  ├─ Scroll position resets
  └─ Browser history updated

Single-Page App:
User clicks link → JS intercepts → Developer must handle:
  ├─ Focus management ⚠
  ├─ Announce navigation ⚠
  ├─ Scroll position ⚠
  └─ History state ⚠

⚠ = Requires manual implementation
```

According to WebAIM's screen reader survey, 67% of users find dynamic content updates problematic when not properly announced.

## Why SPA Accessibility is Challenging

### Problems Unique to SPAs

```typescript
// React: Common SPA accessibility issues

// ❌ PROBLEM 1: No page reload announcement
function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
      </Routes>
    </BrowserRouter>
  );
}
// Screen reader users aren't notified of route changes!

// ❌ PROBLEM 2: Lost focus on navigation
function Nav() {
  return (
    <nav>
      <Link to="/">Home</Link>
      <Link to="/about">About</Link>
    </nav>
  );
}
// Focus stays on clicked link, not on new content

// ❌ PROBLEM 3: Unannounced loading states
function DataComponent() {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    fetch('/api/data').then(res => res.json()).then(setData);
  }, []);
  
  return data ? <div>{data}</div> : <div>Loading...</div>;
}
// Screen readers may not announce loading state

// ❌ PROBLEM 4: Scroll position not reset
// User navigates to new page but scroll stays at bottom
```

### What Needs to be Handled

1. **Route Change Announcements**: Notify users that content has changed
2. **Focus Management**: Move focus appropriately after navigation
3. **Live Regions**: Announce dynamic content updates
4. **Loading States**: Communicate async operations
5. **Scroll Behavior**: Reset or maintain scroll position
6. **Browser History**: Update for back/forward button support

## Client-Side Routing

### Announcing Route Changes

```typescript
// React: Route change announcement with React Router
import { useLocation } from 'react-router-dom';
import { useEffect, useRef } from 'react';

const RouteAnnouncer: React.FC = () => {
  const location = useLocation();
  const [announcement, setAnnouncement] = React.useState('');
  const previousLocation = useRef('');

  useEffect(() => {
    if (previousLocation.current !== location.pathname) {
      // Determine page title or announcement
      const pageTitle = document.title || 'Page loaded';
      
      setAnnouncement(`Navigated to ${pageTitle}`);
      previousLocation.current = location.pathname;
    }
  }, [location]);

  return (
    <div
      role="status"
      aria-live="polite"
      aria-atomic="true"
      className="sr-only"
    >
      {announcement}
    </div>
  );
};

// Usage in App
function App() {
  return (
    <BrowserRouter>
      <RouteAnnouncer />
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/products" element={<Products />} />
      </Routes>
    </BrowserRouter>
  );
}
```

```typescript
// Angular: Route change announcement
import { Component, OnInit } from '@angular/core';
import { Router, NavigationEnd } from '@angular/router';
import { filter } from 'rxjs/operators';

@Component({
  selector: 'app-route-announcer',
  template: `
    <div
      role="status"
      aria-live="polite"
      aria-atomic="true"
      class="sr-only"
    >
      {{ announcement }}
    </div>
  `,
  styles: [`
    .sr-only {
      position: absolute;
      width: 1px;
      height: 1px;
      padding: 0;
      margin: -1px;
      overflow: hidden;
      clip: rect(0, 0, 0, 0);
      white-space: nowrap;
      border-width: 0;
    }
  `]
})
export class RouteAnnouncerComponent implements OnInit {
  announcement = '';

  constructor(private router: Router) {}

  ngOnInit() {
    this.router.events
      .pipe(filter(event => event instanceof NavigationEnd))
      .subscribe(() => {
        const title = this.getPageTitle();
        this.announcement = `Navigated to ${title}`;
      });
  }

  private getPageTitle(): string {
    // Extract title from route data or document title
    return document.title || 'New page';
  }
}
```

### Skip Links for SPAs

```typescript
// React: Skip link with dynamic main content
const Layout: React.FC = ({ children }) => {
  const mainRef = useRef<HTMLElement>(null);

  const skipToMain = (e: React.MouseEvent) => {
    e.preventDefault();
    mainRef.current?.focus();
  };

  return (
    <>
      <a href="#main-content" className="skip-link" onClick={skipToMain}>
        Skip to main content
      </a>

      <header>
        <nav>
          <Link to="/">Home</Link>
          <Link to="/about">About</Link>
          <Link to="/contact">Contact</Link>
        </nav>
      </header>

      <main id="main-content" ref={mainRef} tabIndex={-1}>
        {children}
      </main>

      <footer>
        <p>&copy; 2024 Company</p>
      </footer>
    </>
  );
};

// CSS
const styles = `
  .skip-link {
    position: absolute;
    top: -40px;
    left: 0;
    background: #000;
    color: #fff;
    padding: 8px;
    text-decoration: none;
    z-index: 1000;
  }

  .skip-link:focus {
    top: 0;
  }

  main:focus {
    outline: none;
  }
`;
```

## Focus Management

### Focus Management on Route Change

```typescript
// React: Comprehensive focus management
import { useEffect, useRef } from 'react';
import { useLocation } from 'react-router-dom';

const useFocusManagement = () => {
  const location = useLocation();
  const mainRef = useRef<HTMLElement>(null);
  const previousPathRef = useRef('');

  useEffect(() => {
    // Only run on route change, not initial mount
    if (previousPathRef.current && previousPathref.current !== location.pathname) {
      // Strategy 1: Focus main heading
      const heading = mainRef.current?.querySelector('h1');
      if (heading) {
        (heading as HTMLElement).setAttribute('tabindex', '-1');
        (heading as HTMLElement).focus();
        return;
      }

      // Strategy 2: Focus main container
      if (mainRef.current) {
        mainRef.current.focus();
        return;
      }

      // Strategy 3: Focus body (last resort)
      document.body.setAttribute('tabindex', '-1');
      document.body.focus();
      document.body.removeAttribute('tabindex');
    }

    previousPathRef.current = location.pathname;
  }, [location]);

  return mainRef;
};

// Usage
const Page: React.FC = () => {
  const mainRef = useFocusManagement();

  return (
    <main ref={mainRef} tabIndex={-1}>
      <h1>Page Title</h1>
      <p>Page content...</p>
    </main>
  );
};
```

```typescript
// React: Focus management for modals and dialogs
const Modal: React.FC<{
  isOpen: boolean;
  onClose: () => void;
}> = ({ isOpen, onClose, children }) => {
  const modalRef = useRef<HTMLDivElement>(null);
  const previousFocusRef = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (isOpen) {
      // Store current focus
      previousFocusRef.current = document.activeElement as HTMLElement;

      // Focus modal
      setTimeout(() => {
        const focusable = modalRef.current?.querySelector<HTMLElement>(
          'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
        );
        focusable?.focus() || modalRef.current?.focus();
      }, 0);
    } else {
      // Restore focus
      previousFocusRef.current?.focus();
    }
  }, [isOpen]);

  if (!isOpen) return null;

  return (
    <div
      ref={modalRef}
      role="dialog"
      aria-modal="true"
      tabIndex={-1}
      onKeyDown={(e) => {
        if (e.key === 'Escape') onClose();
      }}
    >
      {children}
    </div>
  );
};
```

### Focus Trap

```typescript
// React: Focus trap implementation
const useFocusTrap = (ref: React.RefObject<HTMLElement>, isActive: boolean) => {
  useEffect(() => {
    if (!isActive || !ref.current) return;

    const element = ref.current;
    const focusableElements = element.querySelectorAll<HTMLElement>(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );

    const firstElement = focusableElements[0];
    const lastElement = focusableElements[focusableElements.length - 1];

    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return;

      if (e.shiftKey) {
        if (document.activeElement === firstElement) {
          e.preventDefault();
          lastElement.focus();
        }
      } else {
        if (document.activeElement === lastElement) {
          e.preventDefault();
          firstElement.focus();
        }
      }
    };

    element.addEventListener('keydown', handleKeyDown);
    return () => element.removeEventListener('keydown', handleKeyDown);
  }, [ref, isActive]);
};

// Usage
const Modal: React.FC = () => {
  const [isOpen, setIsOpen] = useState(false);
  const modalRef = useRef<HTMLDivElement>(null);
  
  useFocusTrap(modalRef, isOpen);

  return isOpen ? (
    <div ref={modalRef} role="dialog" aria-modal="true">
      {/* Modal content */}
    </div>
  ) : null;
};
```

## Announcements and Live Regions

### Live Region Types

```typescript
// React: Different live region types
const LiveRegionExamples: React.FC = () => {
  const [politeMessage, setPoliteMessage] = useState('');
  const [assertiveMessage, setAssertiveMessage] = useState('');
  const [statusMessage, setStatusMessage] = useState('');

  return (
    <div>
      {/* Polite: Waits for user to finish */}
      <div role="status" aria-live="polite" aria-atomic="true">
        {politeMessage}
      </div>

      {/* Assertive: Interrupts immediately */}
      <div role="alert" aria-live="assertive" aria-atomic="true">
        {assertiveMessage}
      </div>

      {/* Status: Same as polite */}
      <div role="status" aria-atomic="true">
        {statusMessage}
      </div>

      {/* Log: Maintains history */}
      <div role="log" aria-live="polite" aria-atomic="false">
        <div>Item 1 added</div>
        <div>Item 2 added</div>
      </div>

      <button onClick={() => setPoliteMessage('Changes saved')}>
        Save (Polite announcement)
      </button>

      <button onClick={() => setAssertiveMessage('Error: Save failed!')}>
        Trigger Error (Assertive announcement)
      </button>

      <button onClick={() => setStatusMessage('Loading...')}>
        Load Data (Status announcement)
      </button>
    </div>
  );
};
```

### Announcer Service

```typescript
// React: Reusable announcer hook
import { useState, useCallback } from 'react';

type AnnouncementType = 'polite' | 'assertive';

interface Announcement {
  message: string;
  type: AnnouncementType;
}

const useAnnouncer = () => {
  const [announcement, setAnnouncement] = useState<Announcement | null>(null);

  const announce = useCallback((message: string, type: AnnouncementType = 'polite') => {
    // Clear previous announcement
    setAnnouncement(null);

    // Set new announcement after brief delay (ensures screen reader picks it up)
    setTimeout(() => {
      setAnnouncement({ message, type });
    }, 100);
  }, []);

  const Announcer = () => (
    <>
      <div
        role="status"
        aria-live="polite"
        aria-atomic="true"
        className="sr-only"
      >
        {announcement?.type === 'polite' ? announcement.message : ''}
      </div>
      <div
        role="alert"
        aria-live="assertive"
        aria-atomic="true"
        className="sr-only"
      >
        {announcement?.type === 'assertive' ? announcement.message : ''}
      </div>
    </>
  );

  return { announce, Announcer };
};

// Usage
const DataTable: React.FC = () => {
  const { announce, Announcer } = useAnnouncer();
  const [data, setData] = useState([]);
  const [loading, setLoading] = useState(false);

  const loadData = async () => {
    setLoading(true);
    announce('Loading data, please wait', 'polite');

    try {
      const response = await fetch('/api/data');
      const newData = await response.json();
      setData(newData);
      announce(`${newData.length} items loaded`, 'polite');
    } catch (error) {
      announce('Error loading data', 'assertive');
    } finally {
      setLoading(false);
    }
  };

  return (
    <>
      <Announcer />
      <button onClick={loadData} disabled={loading}>
        Load Data
      </button>
      <table>
        {/* Table content */}
      </table>
    </>
  );
};
```

```typescript
// Angular: Announcer service
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';

interface Announcement {
  message: string;
  type: 'polite' | 'assertive';
}

@Injectable({
  providedIn: 'root'
})
export class AnnouncerService {
  private announcement$ = new BehaviorSubject<Announcement | null>(null);
  
  getAnnouncement() {
    return this.announcement$.asObservable();
  }

  announce(message: string, type: 'polite' | 'assertive' = 'polite') {
    // Clear previous announcement
    this.announcement$.next(null);

    // Set new announcement
    setTimeout(() => {
      this.announcement$.next({ message, type });
    }, 100);
  }
}

// Announcer Component
@Component({
  selector: 'app-announcer',
  template: `
    <div
      role="status"
      aria-live="polite"
      aria-atomic="true"
      class="sr-only"
    >
      {{ (announcement$ | async)?.type === 'polite' ? (announcement$ | async)?.message : '' }}
    </div>
    <div
      role="alert"
      aria-live="assertive"
      aria-atomic="true"
      class="sr-only"
    >
      {{ (announcement$ | async)?.type === 'assertive' ? (announcement$ | async)?.message : '' }}
    </div>
  `
})
export class AnnouncerComponent {
  announcement$ = this.announcer.getAnnouncement();

  constructor(private announcer: AnnouncerService) {}
}

// Usage in component
@Component({
  selector: 'app-data-table',
  template: `
    <button (click)="loadData()" [disabled]="loading">
      Load Data
    </button>
    <table>
      <!-- Table content -->
    </table>
  `
})
export class DataTableComponent {
  data: any[] = [];
  loading = false;

  constructor(private announcer: AnnouncerService) {}

  async loadData() {
    this.loading = true;
    this.announcer.announce('Loading data, please wait', 'polite');

    try {
      const response = await fetch('/api/data');
      this.data = await response.json();
      this.announcer.announce(`${this.data.length} items loaded`, 'polite');
    } catch (error) {
      this.announcer.announce('Error loading data', 'assertive');
    } finally {
      this.loading = false;
    }
  }
}
```

## Loading States

### Accessible Loading Indicators

```typescript
// React: Various loading patterns
const LoadingPatterns: React.FC = () => {
  const [loading1, setLoading1] = useState(false);
  const [loading2, setLoading2] = useState(false);

  return (
    <div>
      {/* Pattern 1: Simple status */}
      <div>
        <button onClick={() => setLoading1(true)} disabled={loading1}>
          Load Data
        </button>
        {loading1 && (
          <div role="status" aria-live="polite">
            Loading data, please wait...
          </div>
        )}
      </div>

      {/* Pattern 2: Progress bar */}
      <div>
        <button onClick={() => setLoading2(true)} disabled={loading2}>
          Upload File
        </button>
        {loading2 && (
          <div
            role="progressbar"
            aria-valuenow={50}
            aria-valuemin={0}
            aria-valuemax={100}
            aria-label="Upload progress"
          >
            <div className="progress-bar" style={{ width: '50%' }}>
              50%
            </div>
          </div>
        )}
      </div>

      {/* Pattern 3: Spinner with status */}
      <div>
        {loading1 && (
          <div role="status" aria-live="polite" aria-label="Loading">
            <div className="spinner" aria-hidden="true"></div>
            <span className="sr-only">Loading, please wait...</span>
          </div>
        )}
      </div>
    </div>
  );
};
```

```typescript
// React: Skeleton screens with announcements
const SkeletonLoader: React.FC = () => {
  const [loading, setLoading] = useState(true);
  const [data, setData] = useState(null);

  useEffect(() => {
    setTimeout(() => {
      setData({ title: 'Article Title', content: 'Content...' });
      setLoading(false);
    }, 2000);
  }, []);

  return (
    <div>
      {/* Screen reader announcement */}
      <div role="status" aria-live="polite" aria-atomic="true">
        {loading ? 'Loading article...' : 'Article loaded'}
      </div>

      {loading ? (
        // Skeleton UI (aria-hidden from screen readers)
        <div aria-hidden="true" className="skeleton">
          <div className="skeleton-title"></div>
          <div className="skeleton-text"></div>
          <div className="skeleton-text"></div>
        </div>
      ) : (
        // Actual content
        <article>
          <h1>{data.title}</h1>
          <p>{data.content}</p>
        </article>
      )}
    </div>
  );
};
```

## Common Patterns

### Infinite Scroll

```typescript
// React: Accessible infinite scroll
const InfiniteScroll: React.FC = () => {
  const [items, setItems] = useState<string[]>([]);
  const [loading, setLoading] = useState(false);
  const [hasMore, setHasMore] = useState(true);
  const { announce, Announcer } = useAnnouncer();

  const loadMore = async () => {
    if (loading || !hasMore) return;

    setLoading(true);
    announce('Loading more items', 'polite');

    try {
      const newItems = await fetchItems();
      setItems([...items, ...newItems]);
      announce(`${newItems.length} more items loaded. Total: ${items.length + newItems.length}`, 'polite');
      
      if (newItems.length === 0) {
        setHasMore(false);
        announce('No more items to load', 'polite');
      }
    } catch (error) {
      announce('Error loading items', 'assertive');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div>
      <Announcer />
      
      <h1>Items List</h1>
      
      <ul>
        {items.map((item, index) => (
          <li key={index}>{item}</li>
        ))}
      </ul>

      {hasMore && (
        <button
          onClick={loadMore}
          disabled={loading}
          aria-label={loading ? 'Loading more items' : 'Load more items'}
        >
          {loading ? 'Loading...' : 'Load More'}
        </button>
      )}

      {!hasMore && (
        <div role="status">
          End of list. {items.length} items total.
        </div>
      )}
    </div>
  );
};
```

### Search with Live Results

```typescript
// React: Accessible live search
const LiveSearch: React.FC = () => {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<string[]>([]);
  const [loading, setLoading] = useState(false);
  const debounceTimeout = useRef<NodeJS.Timeout>();

  useEffect(() => {
    if (!query) {
      setResults([]);
      return;
    }

    if (debounceTimeout.current) {
      clearTimeout(debounceTimeout.current);
    }

    debounceTimeout.current = setTimeout(async () => {
      setLoading(true);
      const searchResults = await searchAPI(query);
      setResults(searchResults);
      setLoading(false);
    }, 300);

    return () => {
      if (debounceTimeout.current) {
        clearTimeout(debounceTimeout.current);
      }
    };
  }, [query]);

  return (
    <div>
      <label htmlFor="search">Search</label>
      <input
        id="search"
        type="search"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        aria-controls="search-results"
        aria-describedby="search-description"
      />
      <div id="search-description" className="sr-only">
        Results will appear below as you type
      </div>

      <div
        id="search-results"
        role="region"
        aria-live="polite"
        aria-atomic="false"
      >
        {loading && (
          <div role="status">Searching...</div>
        )}
        
        {!loading && results.length > 0 && (
          <div role="status" className="sr-only">
            {results.length} results found
          </div>
        )}

        {!loading && query && results.length === 0 && (
          <div role="status">No results found</div>
        )}

        <ul>
          {results.map((result, index) => (
            <li key={index}>
              <a href={`/item/${index}`}>{result}</a>
            </li>
          ))}
        </ul>
      </div>
    </div>
  );
};
```

## Common Mistakes

### 1. No Route Change Announcement

```typescript
// ❌ WRONG: Silent navigation
<BrowserRouter>
  <Routes>
    <Route path="/" element={<Home />} />
    <Route path="/about" element={<About />} />
  </Routes>
</BrowserRouter>

// ✅ CORRECT: Announced navigation
<BrowserRouter>
  <RouteAnnouncer />
  <Routes>
    <Route path="/" element={<Home />} />
    <Route path="/about" element={<About />} />
  </Routes>
</BrowserRouter>
```

### 2. Lost Focus After Actions

```typescript
// ❌ WRONG: Focus not managed after deletion
const deleteItem = (id: string) => {
  setItems(items.filter(item => item.id !== id));
  // Focus is lost!
};

// ✅ CORRECT: Focus moves to next item or container
const deleteItem = (id: string, index: number) => {
  setItems(items.filter(item => item.id !== id));
  
  // Focus next item, previous item, or container
  setTimeout(() => {
    const nextItem = document.querySelector(`[data-index="${index}"]`);
    const prevItem = document.querySelector(`[data-index="${index - 1}"]`);
    const container = document.querySelector('[role="list"]');
    
    (nextItem || prevItem || container)?.focus();
  }, 0);
};
```

### 3. Unannounced Errors

```typescript
// ❌ WRONG: Silent error
const [error, setError] = useState('');

return error && <div className="error">{error}</div>;

// ✅ CORRECT: Announced error
return error && (
  <div role="alert" aria-live="assertive" className="error">
    {error}
  </div>
);
```

## Best Practices

### 1. Progressive Enhancement

```typescript
// Start with accessible baseline, enhance with JS
const ProgressiveForm: React.FC = () => {
  const [clientSideValidation, setClientSideValidation] = useState(false);

  useEffect(() => {
    // Enable client-side features only when JS loads
    setClientSideValidation(true);
  }, []);

  return (
    <form action="/submit" method="POST">
      {/* Form works without JS */}
      <input type="email" name="email" required />
      
      {clientSideValidation && (
        <div role="status" aria-live="polite">
          {/* Client-side validation feedback */}
        </div>
      )}
      
      <button type="submit">Submit</button>
    </form>
  );
};
```

### 2. Test with Real Assistive Technology

```typescript
// Testing checklist for SPAs
const SPATestingChecklist = `
✓ Test with screen reader (NVDA, JAWS, VoiceOver)
✓ Navigate with keyboard only
✓ Check focus indicators visible
✓ Verify route changes announced
✓ Test loading states announced
✓ Check error messages announced
✓ Verify focus management
✓ Test skip links
✓ Check live regions
✓ Test with browser zoom
`;
```

## Interview Questions

### Junior Level
1. **What makes SPAs different from traditional websites in terms of accessibility?**
   - No automatic page reload announcements
   - Manual focus management required
   - Dynamic content needs announcement
   - Browser history handling

2. **What is a live region?**
   - Area that announces dynamic changes
   - Uses aria-live attribute
   - Types: polite, assertive, off
   - Doesn't require focus

3. **Why is focus management important in SPAs?**
   - Keyboard users need to know where they are
   - Screen readers follow focus
   - Prevents user disorientation
   - Required after route changes

### Mid Level
4. **How would you announce a route change to screen reader users?**
   - Use live region with role="status"
   - Announce page title
   - Place announcement in DOM
   - Use aria-live="polite"

5. **Explain the difference between aria-live="polite" and "assertive"**
   - Polite: Waits for user to finish
   - Assertive: Interrupts immediately
   - Polite for status updates
   - Assertive for errors/alerts

6. **How do you handle focus after deleting an item?**
   - Focus next item if available
   - Else focus previous item
   - Else focus container
   - Announce deletion

### Senior Level
7. **Design a comprehensive accessibility strategy for a large SPA**
   - Route announcement system
   - Global focus management
   - Announcer service
   - Loading state patterns
   - Error handling strategy
   - Testing automation
   - Documentation

8. **How would you make infinite scroll accessible?**
   - Load more button (no auto-load)
   - Announce items loaded
   - Maintain focus position
   - Keyboard accessible
   - Announce total count
   - "Load more" button

9. **Explain trade-offs between client-side and server-side rendering for accessibility**
   - SSR: Better initial load, automatic announcements
   - CSR: Better interactivity, requires manual work
   - SSR: SEO benefits
   - CSR: Complex focus management
   - Hybrid approaches available

10. **How would you test SPA accessibility automatically in CI/CD?**
    - axe-core integration
    - Lighthouse CI
    - pa11y in tests
    - Custom assertions
    - Screen reader testing tools
    - Visual regression tests

## Key Takeaways

1. **Announce route changes** - screen readers need notification
2. **Manage focus appropriately** - move focus on navigation
3. **Use live regions** - for dynamic content updates
4. **Announce loading states** - keep users informed
5. **Test with real assistive tech** - not just automated tools
6. **Handle errors accessibly** - use role="alert"
7. **Progressive enhancement** - work without JS
8. **Document patterns** - for team consistency
9. **Focus trap modals** - prevent focus escape
10. **Test keyboard navigation** - all functionality accessible

## Resources

### Libraries
- [React Router Announcer](https://github.com/remix-run/react-router/discussions/9663)
- [Reach Router](https://reach.tech/router/) - Built-in announcements
- [@angular/router](https://angular.io/api/router) - Angular routing

### Tools
- [axe DevTools](https://www.deque.com/axe/devtools/)
- [NVDA](https://www.nvaccess.org/) - Screen reader testing
- [Lighthouse](https://developers.google.com/web/tools/lighthouse)

### Articles
- [Accessible Client-Side Routing](https://www.gatsbyjs.com/blog/2019-07-11-user-testing-accessible-client-routing/)
- [Single Page Apps Routers are Broken](https://marcy sutton.com/single-page-apps-routers-are-broken)
- [Making SPAs Accessible](https://www.smashingmagazine.com/2015/05/client-rendered-accessibility/)

### Documentation
- [WAI-ARIA Authoring Practices](https://www.w3.org/WAI/ARIA/apg/)
- [WCAG 2.1 Guidelines](https://www.w3.org/WAI/WCAG21/quickref/)
- [MDN: ARIA Live Regions](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/ARIA_Live_Regions)
