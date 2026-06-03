# Hydration Mismatch — Causes & Fixes

## What Is Hydration?

After a server renders HTML and sends it to the browser, React **hydrates** it — attaches event listeners and reconciles the server-rendered DOM with what the client would render. If they differ, React throws a hydration error and re-renders from scratch (bad for performance and UX).

---

## Common Causes

### 1. Non-Deterministic Values

```jsx
// ❌ Different value on server vs client
function Timestamp() {
  return <span>{new Date().toLocaleTimeString()}</span>;
}

// ❌ Random values
function Avatar() {
  return <img src={`/avatars/${Math.random()}.png`} />;
}
```

### 2. Browser-Only APIs Accessed During Render

```jsx
// ❌ window is undefined on the server
function ThemeProvider() {
  const theme = window.matchMedia('(prefers-color-scheme: dark)').matches
    ? 'dark' : 'light';
  return <div className={theme}>...</div>;
}
```

### 3. Locale / Timezone Differences

```jsx
// ❌ Server (UTC) vs browser (user's timezone)
function Price({ amount }) {
  return <span>{new Intl.NumberFormat().format(amount)}</span>;
}
```

### 4. Browser Extensions Modifying the DOM

Extensions may inject elements (translation tools, ad blockers, password managers) into the HTML before React hydrates. This is usually outside your control — suppress with `suppressHydrationWarning` on affected elements.

### 5. Invalid HTML Nesting

```jsx
// ❌ <p> cannot contain <div> — browsers auto-correct the HTML
function Component() {
  return <p><div>Content</div></p>;
}
```

---

## Fixes

### `suppressHydrationWarning` (Opt-Out)

```jsx
// For content that legitimately differs (e.g., timestamps, extensions)
<time suppressHydrationWarning dateTime={iso}>
  {formattedTime}
</time>
```

Only suppresses the warning for that element — doesn't fix the mismatch.

### `useEffect` for Client-Only Rendering

```jsx
function ClientOnly({ children, fallback = null }) {
  const [mounted, setMounted] = useState(false);
  useEffect(() => setMounted(true), []);
  return mounted ? children : fallback;
}

// Usage
<ClientOnly fallback={<Skeleton />}>
  <ThemeAwareComponent />
</ClientOnly>
```

Server renders `fallback`, client renders the real component after mount.

### Next.js `dynamic` with `ssr: false`

```jsx
import dynamic from 'next/dynamic';

const Chart = dynamic(() => import('./Chart'), {
  ssr: false,
  loading: () => <Skeleton />,
});
```

The component is excluded from SSR entirely — rendered only on the client.

### Consistent Server/Client State

```jsx
// ❌ Don't read from browser APIs during render
function Component() {
  const [theme, setTheme] = useState('light');

  useEffect(() => {
    // Read browser API after mount, not during render
    setTheme(window.matchMedia('(prefers-dark)').matches ? 'dark' : 'light');
  }, []);

  return <div className={theme}>...</div>;
}
```

### Pass Locale/Timezone from Server

```tsx
// Server Component
export default async function Page() {
  const locale = headers().get('accept-language')?.split(',')[0] ?? 'en-US';
  return <ClientComponent locale={locale} />;
}
```

---

## Debugging Hydration Errors

React 18 logs the specific text that differs:

```
Warning: Text content did not match.
  Server: "12:30 PM"
  Client: "12:31 PM"
```

Approach:
1. Check the component mentioned in the stack trace
2. Look for `Date`, `Math.random()`, `window`, `localStorage` calls in render
3. Check for browser extension interference (test in incognito)
4. Validate HTML nesting with browser DevTools

---

## Common Interview Questions

**Q: What happens when a hydration mismatch occurs in React 18?**
React logs a warning and **re-renders the entire client tree from scratch** (discarding the server HTML). This means a flash of content and double-rendering cost.

**Q: Is hydration mismatch always a problem?**
For text that changes frequently (timestamps), it's acceptable to use `suppressHydrationWarning`. For structural mismatches (different elements), it's always a problem that should be fixed.

**Q: How does Next.js handle hydration for server-rendered pages?**
Next.js sends the server HTML with a `__NEXT_DATA__` script tag containing the initial props. React hydrates the HTML using this data. Mismatches occur when the client would render differently with the same data.
