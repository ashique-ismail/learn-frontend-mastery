# Production Failure: Hydration Failures and SSR Cascades

## The Idea

**In plain English:** When a website loads, the server first sends a "painted picture" of the page (HTML), and then the browser's JavaScript takes over to make it interactive. Hydration is that handover moment — and a hydration failure is when the JavaScript arrives and finds the picture looks different from what it expected, causing glitches or blank screens.

**Real-world analogy:** Imagine a stage crew sets up a theatre set before the audience arrives (the server renders HTML). Then the live actors walk out to perform (JavaScript "hydrates" the page). If the actors find the props have been moved — the chair is on the left instead of the right — they freeze up or improvise badly. That mismatch is the failure.

- The stage crew setting up the set = the server generating the initial HTML
- The actors walking out = the JavaScript/React code running in the browser
- The misplaced props = values that differ between server and browser (like a clock showing a different time)

---

## Overview

Hydration is the process of React attaching event listeners to server-rendered HTML. When the client-rendered React tree doesn't match what the server sent, React must reconcile the mismatch — and the failure modes range from silent wrong UI to complete white screens. These bugs are notoriously hard to reproduce because they only manifest in production SSR environments.

---

## Failure 1: Non-Deterministic Rendering

### The Bug

```tsx
// Server renders at 14:00:01 UTC
// Client hydrates at 14:00:02 UTC — different output → mismatch

function LiveTimestamp() {
  return <span>{new Date().toLocaleTimeString()}</span>;
  // Server:  "2:00:01 PM"
  // Client:  "2:00:02 PM"
  // React:   Hydration mismatch warning. Replaces server content with client.
  // User:    Sees flash of wrong content.
}

// Random IDs
function Modal() {
  const id = Math.random().toString(36).slice(2); // different every render
  return <div aria-labelledby={id}>...</div>;
}

// Browser-only APIs
function UserAgent() {
  return <p>{navigator.userAgent}</p>; // doesn't exist on server → crash
}
```

### The Fix

```tsx
// Timestamps: useEffect to update after hydration
function LiveTimestamp() {
  const [time, setTime] = useState<string | null>(null);

  useEffect(() => {
    setTime(new Date().toLocaleTimeString());
    const id = setInterval(() => setTime(new Date().toLocaleTimeString()), 1000);
    return () => clearInterval(id);
  }, []);

  // null on server + first render → no mismatch; updates after hydration
  return <span>{time ?? '--:--:--'}</span>;
}

// Stable IDs: useId hook (React 18+)
function Modal() {
  const id = useId(); // deterministic, same on server and client
  return <div aria-labelledby={id}>...</div>;
}

// Browser-only: suppressHydrationWarning or conditional render
function UserAgent() {
  const [ua, setUa] = useState('');
  useEffect(() => setUa(navigator.userAgent), []);
  return <p suppressHydrationWarning>{ua}</p>;
}
```

---

## Failure 2: Auth State Causing Full-Page Mismatch

### The Bug

```tsx
// Server: user is unauthenticated (no cookie during SSR)
// Client: user has a session cookie → isAuthenticated = true

export default function Header() {
  const { user } = useAuth(); // server: null, client: { name: 'Alice' }
  return (
    <header>
      {user ? <UserMenu user={user} /> : <LoginButton />}
    </header>
  );
}
```

Server sends `<LoginButton>`. Client hydrates and sees `<UserMenu>`. React throws a hydration error. In React 18: silently replaces the tree. In React 17: throws "Expected server HTML to contain a matching element."

### The Fix: Dehydrate Auth-Sensitive Sections

**Option A: Render auth-sensitive UI only on client**
```tsx
function Header() {
  const [mounted, setMounted] = useState(false);
  useEffect(() => setMounted(true), []);

  return (
    <header>
      <Logo />
      {mounted ? <AuthSection /> : <AuthPlaceholder />}
    </header>
  );
}
```

**Option B: Pass auth state from server (Next.js)**
```tsx
// Next.js App Router: server component reads auth cookie
import { cookies } from 'next/headers';

export default async function Header() {
  const session = await getSession(cookies());
  // Server and client now agree on initial state
  return (
    <header>
      {session ? <UserMenu user={session.user} /> : <LoginButton />}
    </header>
  );
}
```

**Option C: Suppress mismatch for known client-only subtrees**
```tsx
<div suppressHydrationWarning>
  {isClient && <ClientOnlyWidget />}
</div>
```

---

## Failure 3: Third-Party Script Mutating the DOM

### The Bug

```html
<!-- Server renders: -->
<div id="root"><h1>Hello</h1><p>Content</p></div>

<!-- Third-party chat widget script runs before React hydrates: -->
<script>
  // Chat widget injects: <div class="chat-bubble">...</div> into #root
</script>

<!-- React hydrates and sees: -->
<div id="root"><h1>Hello</h1><p>Content</p><div class="chat-bubble">...</div></div>

<!-- Mismatch! React expected 2 children, found 3. -->
<!-- Result: React removes the chat bubble or throws. Chat widget breaks. -->
```

### The Fix

```html
<!-- Load third-party scripts AFTER React hydrates -->
<script>
  window.addEventListener('load', () => {
    const script = document.createElement('script');
    script.src = 'https://widget.example.com/chat.js';
    document.body.appendChild(script);
  });
</script>

<!-- Or: render third-party widgets into a Portal outside #root -->
```

```tsx
// Mount third-party widgets in a container outside React's root
function ChatWidget() {
  const containerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    // Initialize widget in its own div, not inside React's root
    initChatWidget(containerRef.current);
    return () => destroyChatWidget();
  }, []);

  return createPortal(<div ref={containerRef} />, document.getElementById('chat-root')!);
}
```

---

## Failure 4: CSS-in-JS Style Flash

### The Bug

```tsx
// Emotion/styled-components: generates styles at runtime
const Title = styled.h1`
  color: ${props => props.theme.primary};
  font-size: 2rem;
`;
```

Server generates styles and injects them into the HTML. But on the client, the CSS-in-JS library regenerates styles before applying them. For a moment (~50–200ms on slow devices), the page renders with server styles, then re-renders with client styles, causing a visible flash of unstyled or differently-styled content.

**Specific failure**: Server uses `dark` theme, client reads user preference and switches to `light` — full-page style flash.

### The Fix: Critical CSS Extraction

```tsx
// Next.js + Emotion: extract critical CSS into <head> during SSR
import createEmotionServer from '@emotion/server/create-instance';
import createCache from '@emotion/cache';

export function getServerSideProps() {
  const cache = createCache({ key: 'css' });
  const { extractCriticalToChunks, constructStyleTagsFromChunks } =
    createEmotionServer(cache);

  const html = renderToString(
    <CacheProvider value={cache}>
      <App />
    </CacheProvider>
  );

  const chunks = extractCriticalToChunks(html);
  const styles = constructStyleTagsFromChunks(chunks);

  // styles are injected into <head> — client picks them up, no flash
}
```

```tsx
// Better: move away from runtime CSS-in-JS for SSR apps
// Use CSS Modules, vanilla-extract, or Tailwind — no runtime, no mismatch
```

---

## Failure 5: Hydration Order Mismatch in Concurrent Mode

### The Bug

React 18 Concurrent Mode can interrupt and restart rendering. If a component throws during render (not during hydration), React retries — but the second render may produce different output if it depends on external mutable state read during render.

```tsx
function AnalyticsProvider({ children }) {
  // Reading from a mutable singleton during render
  const sessionId = analyticsSDK.getSessionId(); // mutable — can change between retries

  return (
    <AnalyticsContext.Provider value={{ sessionId }}>
      {children}
    </AnalyticsContext.Provider>
  );
}
```

React 18 may call this component twice (Strict Mode does this intentionally in development). If `getSessionId()` returns different values, downstream components get inconsistent props.

### The Fix: Read External State in Effects or useSyncExternalStore

```tsx
function AnalyticsProvider({ children }) {
  // useSyncExternalStore: safe for concurrent mode, consistent between renders
  const sessionId = useSyncExternalStore(
    analyticsSDK.subscribe,  // subscribe to changes
    analyticsSDK.getSessionId, // get current value
    () => 'server-session',    // server snapshot
  );

  return (
    <AnalyticsContext.Provider value={{ sessionId }}>
      {children}
    </AnalyticsContext.Provider>
  );
}
```

---

## Diagnosing Hydration Errors in Production

### Development

React 18 logs specific hydration error messages in the console:
```
Warning: Text content did not match.
  Server: "Login"
  Client: "Log In"
```

Use `--trace-warnings` in Node.js to get stack traces.

### Production

Hydration errors in production are often silent (React replaces the tree without throwing). Use:

```typescript
// Custom error boundary to catch hydration failures
class HydrationBoundary extends React.Component {
  static getDerivedStateFromError(error: Error) {
    if (error.message.includes('hydrat')) {
      // Log to Sentry with the component tree
      Sentry.captureException(error, { tags: { type: 'hydration' } });
    }
    return { hasError: true };
  }
}
```

Or: monitor your Sentry/Datadog for errors matching `hydration` or `did not match`.

---

## Interview Questions

**Q: Your SSR app shows a flash of incorrect content on load. How do you debug it?**
A: This is a hydration mismatch. I'd check: (1) Any `Math.random()`, `Date.now()`, or browser-only APIs used during render — these produce different values on server vs client; (2) Auth state — if the component renders differently based on authentication, the server (unauthenticated) and client (authenticated) will mismatch; (3) Locale/timezone — `toLocaleString()` may differ between server timezone and user timezone; (4) Third-party scripts that mutate the DOM before React hydrates.

**Q: How does `suppressHydrationWarning` work and when should you use it?**
A: It tells React to skip the attribute/content comparison for that specific element and its text content during hydration — the server value is kept but no warning/error is thrown when the client value differs. Use it for genuinely client-specific values that legitimately differ: timestamps, user-specific data that can't be SSR'd, third-party script output. Don't use it to silence legitimate mismatch bugs — it hides real problems.

**Q: Why does CSS-in-JS cause a style flash in SSR apps?**
A: The server generates CSS and injects it into `<style>` tags in the HTML. On the client, the CSS-in-JS runtime regenerates class names and styles. If the generated class names differ between server and client renders (due to insertion order, randomness in class name generation, or theme differences), there's a brief period where server styles are applied before client styles replace them. The fix is to extract critical CSS on the server and include it inline so the client runtime picks it up rather than regenerating.
