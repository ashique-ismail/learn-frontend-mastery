# History API and Navigation

## Overview

The History API enables JavaScript to manipulate the browser's session history—the stack of pages visited in a tab—without triggering full page reloads. This is the foundation of client-side routing in every major SPA framework (React Router, Vue Router, Angular Router). Understanding the History API's state model, the popstate event, and how it relates to the newer Navigation API is critical for senior interviews in frontend engineering.

## The Browser Session History

Each browser tab maintains an ordered stack of history entries. Each entry has:
- A URL
- A serializable state object (optional, set via the History API)
- A scroll position (managed by the browser)

The user can navigate this stack via the browser's Back/Forward buttons.

## Core History API

### Reading History State

```javascript
// Number of entries in the session history stack for this tab
history.length;

// State object associated with the current history entry
history.state; // null if none was set

// Current URL (read-only, use pushState/replaceState to change)
location.href;
location.pathname; // '/products/42'
location.search;   // '?page=2&sort=asc'
location.hash;     // '#reviews'
```

### pushState — Add a New Entry

```javascript
// history.pushState(state, title, url)
// state: serializable object (< 640KB in some browsers)
// title: ignored by most browsers (pass '' or a string)
// url: new URL — must be same origin; triggers no network request

history.pushState(
  { page: 2, filters: { category: 'books' } },
  '',
  '/products?page=2&category=books'
);

// After: back button goes to previous URL
// history.length increases by 1
// popstate does NOT fire for pushState calls
```

### replaceState — Replace Current Entry

```javascript
// Same signature as pushState but replaces the current entry
// Does not change history.length
// Useful for redirects, correcting URLs, updating state without back-button entry

history.replaceState(
  { page: 1 },
  '',
  '/products'
);
```

### Back, Forward, Go

```javascript
history.back();     // go to previous entry (same as browser Back)
history.forward();  // go to next entry
history.go(-1);     // same as back()
history.go(1);      // same as forward()
history.go(-3);     // jump 3 entries back
history.go(0);      // reload the page
```

## The popstate Event

`popstate` fires when the user navigates the history stack (Back/Forward buttons, or `history.go()`). It does NOT fire for `pushState` or `replaceState` calls.

```javascript
window.addEventListener('popstate', (event) => {
  // event.state is the state object associated with the new (current) entry
  const state = event.state;
  console.log('Navigated to state:', state);
  renderPage(location.pathname, state);
});

// Note: popstate fires with event.state === null if the entry
// was created by normal browser navigation (not via pushState)
```

### Building a Simple Client-Side Router

```javascript
class Router {
  #routes = new Map();

  route(pattern, handler) {
    this.#routes.set(pattern, handler);
    return this;
  }

  navigate(url, state = {}) {
    history.pushState({ ...state, url }, '', url);
    this.#dispatch(url, state);
  }

  start() {
    window.addEventListener('popstate', (e) => {
      this.#dispatch(location.pathname, e.state ?? {});
    });

    // Handle initial page load
    this.#dispatch(location.pathname, history.state ?? {});

    // Intercept link clicks
    document.addEventListener('click', (e) => {
      const link = e.target.closest('a[href]');
      if (!link) return;
      const href = link.getAttribute('href');
      if (href.startsWith('/') && !link.target) {
        e.preventDefault();
        this.navigate(href);
      }
    });
  }

  #dispatch(path, state) {
    for (const [pattern, handler] of this.#routes) {
      const match = this.#match(pattern, path);
      if (match) {
        handler({ params: match, state, path });
        return;
      }
    }
    this.#routes.get('*')?.(({ path, state }));
  }

  #match(pattern, path) {
    const patternParts = pattern.split('/');
    const pathParts = path.split('/');
    if (patternParts.length !== pathParts.length) return null;
    const params = {};
    for (let i = 0; i < patternParts.length; i++) {
      if (patternParts[i].startsWith(':')) {
        params[patternParts[i].slice(1)] = pathParts[i];
      } else if (patternParts[i] !== pathParts[i]) {
        return null;
      }
    }
    return params;
  }
}

const router = new Router();
router
  .route('/', ({ state }) => renderHome())
  .route('/products', ({ state }) => renderProducts(state))
  .route('/products/:id', ({ params }) => renderProduct(params.id))
  .route('*', () => render404())
  .start();
```

## URL Manipulation

### URLSearchParams

```javascript
// Parse query string
const params = new URLSearchParams(location.search);
params.get('page');       // '2'
params.getAll('tag');     // ['js', 'css'] (multiple values)
params.has('filter');     // boolean
params.set('page', '3');  // replace
params.append('tag', 'html'); // add without replacing
params.delete('filter');
params.toString();        // 'page=3&tag=js&tag=css&tag=html'

// Update URL without reload
const newParams = new URLSearchParams(location.search);
newParams.set('sort', 'price');
history.pushState({}, '', `${location.pathname}?${newParams}`);
```

### URL Constructor

```javascript
const url = new URL('https://example.com/products?page=2#reviews');

url.protocol;  // 'https:'
url.host;      // 'example.com'
url.hostname;  // 'example.com'
url.port;      // ''
url.pathname;  // '/products'
url.search;    // '?page=2'
url.searchParams; // URLSearchParams instance
url.hash;      // '#reviews'
url.href;      // full URL string

// Relative URL resolution
const base = 'https://example.com/app/dashboard';
new URL('../settings', base).href; // 'https://example.com/app/settings'
```

## Hash-Based Routing (Legacy)

Before the History API was widely supported, SPAs used the URL hash (`#`) to store navigation state. Hash changes do not cause page reloads and the `hashchange` event fires.

```javascript
// Old approach: hash routing
window.addEventListener('hashchange', (event) => {
  const hash = location.hash.slice(1); // remove '#'
  renderRoute(hash);
});

// Navigate
location.hash = 'products/42';
// URL becomes: https://example.com/#products/42

// Drawbacks:
// - Hash is not sent to the server (URL is not canonical)
// - SEO is harder
// - Server cannot serve the correct initial HTML for deep links
```

## The Navigation API (Modern)

The Navigation API (`window.navigation`) is a more capable successor to the History API, available in Chromium-based browsers. It provides intercept, cancellation, and a typed event model.

```javascript
// Feature detection
if ('navigation' in window) {
  navigation.addEventListener('navigate', (event) => {
    const url = new URL(event.destination.url);

    // Only handle same-origin navigations
    if (url.origin !== location.origin) return;

    event.intercept({
      async handler() {
        const html = await fetchPage(url.pathname);
        document.querySelector('#content').innerHTML = html;
        document.title = extractTitle(html);
      },
    });
  });
}

// Navigate programmatically
navigation.navigate('/products/42', { state: { from: 'catalog' } });
navigation.back();
navigation.forward();

// Current entry
navigation.currentEntry.url;
navigation.currentEntry.key;
navigation.currentEntry.id;
navigation.currentEntry.getState();

// Traverse to any entry by key
navigation.traverseTo(entry.key);
```

### Navigation API vs. History API

```javascript
// Navigation API advantages:
// - Intercepts ALL navigations (links, back/forward, form submissions, navigation.navigate)
// - Cancellation support via event.preventDefault()
// - Built-in async handler with loading state
// - event.destination replaces the awkward state restoration
// - No need to intercept all <a> clicks manually
```

## Scroll Restoration

```javascript
// Browser can automatically restore scroll position on popstate
history.scrollRestoration = 'auto';    // default — browser manages scroll
history.scrollRestoration = 'manual';  // you manage scroll position

// Manual scroll restoration
window.addEventListener('popstate', (e) => {
  const scroll = e.state?.scroll;
  if (scroll) {
    window.scrollTo(scroll.x, scroll.y);
  }
});

// Save scroll before navigating away
function navigate(url, state = {}) {
  history.pushState(
    { ...state, scroll: { x: scrollX, y: scrollY } },
    '',
    url
  );
}
```

## Comparison Table

| Feature | Hash Routing | History API | Navigation API |
|---------|-------------|-------------|----------------|
| URL format | `/app#/route` | `/route` | `/route` |
| Server-side routing needed | No | Yes | Yes |
| SEO friendly | Partial | Yes | Yes |
| popstate on push | No | No | N/A (intercept) |
| Intercept all navigations | No | No | Yes |
| Cancellation | No | No | Yes |
| Back/Forward control | hashchange | popstate | navigate event |
| Browser support | Universal | Universal | Chromium only (2023+) |

## Best Practices

### 1. Always Pair pushState with Server-Side Routing

```javascript
// The server must return the SPA shell for all known routes
// Otherwise, a hard refresh on /products/42 returns 404
// Express example:
app.get('*', (req, res) => res.sendFile(path.join(__dirname, 'dist/index.html')));
```

### 2. Keep State Objects Serializable and Small

```javascript
// Good — plain data, small
history.pushState({ productId: 42, page: 2 }, '', url);

// Bad — large or non-serializable data fails silently or throws
history.pushState({ products: massiveArray }, '', url); // may exceed size limit
history.pushState({ el: domElement }, '', url);         // DataCloneError
```

### 3. Handle the Initial Page Load

```javascript
// popstate does not fire on load — dispatch initial route manually
window.addEventListener('DOMContentLoaded', () => {
  router.dispatch(location.pathname, history.state);
});
```

### 4. replaceState for Redirects and State Corrections

```javascript
// User goes to /login but is already authenticated
if (isAuthenticated()) {
  history.replaceState({}, '', '/dashboard'); // no back-button entry for /login
}
```

## Interview Questions

### Q1: What is the difference between pushState and replaceState?

**Answer:** Both change the current URL without triggering a page reload. `pushState` adds a new entry to the session history stack (so the user can navigate back to the previous URL), while `replaceState` modifies the current entry in place without increasing `history.length`. Use `replaceState` for redirects, search query updates, and any navigation that should not create a back button entry.

### Q2: Does popstate fire when pushState is called?

**Answer:** No. `popstate` only fires when the user (or `history.go()`) navigates the history stack—Back button, Forward button, or programmatic `go(n)`. `pushState` and `replaceState` do not fire `popstate`. Your router must call its own render function after invoking `pushState`, then handle `popstate` to re-render when the user goes back.

### Q3: How do SPAs handle direct URL access (deep linking)?

**Answer:** When a user navigates directly to `/products/42`, the browser sends an HTTP request to the server. For SPAs, the server must return the application shell HTML for any valid route—not a 404. A common pattern is a catch-all route that returns `index.html`. The SPA then reads `location.pathname` on startup and renders the correct view.

### Q4: What is the difference between the History API and the Navigation API?

**Answer:** The History API (`history.pushState`, `popstate`) is the established standard but requires manual interception of all link clicks and has no unified way to intercept form submissions or programmatic navigations. The Navigation API (`window.navigation`) intercepts all navigations—links, forms, `navigation.navigate()`—in a single `navigate` event that can be cancelled and handled asynchronously. It also provides a `destination` entry rather than requiring reconstruction of state from `location` and `history.state`. The Navigation API is currently Chromium-only.

### Q5: What are the security constraints on pushState URLs?

**Answer:** The new URL passed to `pushState` must be same-origin. Attempting to push a URL from a different origin throws a `SecurityError`. The path and query string can be anything—the browser does not validate that the path actually exists on the server. The URL is updated in the address bar, history, and used for subsequent navigation, but no network request is made.

## Common Pitfalls

### 1. Forgetting popstate Does Not Fire on pushState

```javascript
// Wrong: only popstate listener, never fires after navigate()
function navigate(url) {
  history.pushState({}, '', url);
  // render() not called — page does not update!
}

// Correct
function navigate(url, state = {}) {
  history.pushState(state, '', url);
  render(url, state); // call render directly after pushState
}
window.addEventListener('popstate', (e) => render(location.pathname, e.state));
```

### 2. Not Handling the Initial Load

```javascript
// popstate fires for back/forward, not on page load
// Without this, the SPA renders blank on hard refresh or direct URL entry
router.dispatch(location.pathname, history.state ?? {}); // call on DOMContentLoaded
```

### 3. Storing Non-Serializable Data in State

```javascript
// Throws DataCloneError — DOM nodes, functions, class instances not allowed
history.pushState({ handler: () => {} }, '', '/next');

// Store IDs or plain data, look up the rest in application state
history.pushState({ id: '42' }, '', '/products/42');
```

### 4. Multiple popstate Listeners Accumulating

```javascript
// Re-registering on every navigation adds duplicate listeners
function navigate(url) {
  history.pushState({}, '', url);
  window.addEventListener('popstate', render); // adds a new listener each time!
}

// Register once on startup
window.addEventListener('popstate', render);
```

## Resources

- [MDN: History API](https://developer.mozilla.org/en-US/docs/Web/API/History_API)
- [MDN: history.pushState](https://developer.mozilla.org/en-US/docs/Web/API/History/pushState)
- [MDN: popstate event](https://developer.mozilla.org/en-US/docs/Web/API/Window/popstate_event)
- [MDN: URLSearchParams](https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams)
- [MDN: Navigation API](https://developer.mozilla.org/en-US/docs/Web/API/Navigation_API)
- [web.dev: Modern client-side routing — the Navigation API](https://developer.chrome.com/docs/web-platform/navigation-api/)
