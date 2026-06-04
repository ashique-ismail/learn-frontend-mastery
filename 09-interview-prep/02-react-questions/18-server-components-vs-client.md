# Server Components vs Client Components

## The Core Distinction

| | Server Components (RSC) | Client Components |
|---|---|---|
| Runs on | Server only | Browser (and optionally server for SSR) |
| Bundle size | Not included in JS bundle | Included in JS bundle |
| Hooks | Cannot use | Can use |
| Event handlers | Cannot have | Can have |
| Direct DB/filesystem access | Yes | No |
| `use client` directive | Not needed | Required at top of file |

---

## Server Components

Default in Next.js App Router. Run on the server, never shipped to the browser.

```tsx
// app/users/page.tsx — Server Component (no 'use client')
import { db } from '@/lib/db';

export default async function UsersPage() {
  const users = await db.query('SELECT * FROM users'); // direct DB access
  return (
    <ul>
      {users.map(u => <li key={u.id}>{u.name}</li>)}
    </ul>
  );
}
```

**What RSC CANNOT do:**
- `useState`, `useEffect`, or any hooks
- Event handlers (`onClick`, `onChange`)
- Browser APIs (`window`, `document`, `localStorage`)
- Context consumers (but can create providers)
- Class components

**What RSC CAN do:**
- `async/await` directly in the component
- Access server-side resources (DB, filesystem, env vars)
- Import server-only modules
- Pass data down as props to Client Components

---

## Client Components

Opt in with `'use client'` at the top of the file. Everything in the file (and its imports) is treated as client code.

```tsx
'use client';

import { useState } from 'react';

export function Counter({ initialCount }: { initialCount: number }) {
  const [count, setCount] = useState(initialCount);
  return (
    <button onClick={() => setCount(c => c + 1)}>
      Count: {count}
    </button>
  );
}
```

`'use client'` marks a **boundary** — this file and everything it imports is client code.

---

## Composition Rules

RSC can import Client Components (passes them as serialized props or children):

```tsx
// ServerPage.tsx — Server Component
import { Counter } from './Counter';  // Counter is 'use client'

export default async function Page() {
  const initialCount = await db.getCount();
  return (
    <main>
      <h1>Dashboard</h1>
      <Counter initialCount={initialCount} />  {/* props serialized */}
    </main>
  );
}
```

**Client Components CANNOT import Server Components** — server code would end up in the client bundle. Pass RSC as `children` props instead:

```tsx
// Correct: RSC as children prop
// layout.tsx (Server Component)
export default function Layout({ children }) {
  return <ClientShell>{children}</ClientShell>;
}

// ClientShell.tsx ('use client')
export function ClientShell({ children }) {
  return <div className="shell">{children}</div>; // children rendered by server
}
```

---

## Props Must Be Serializable

Props passed from Server to Client Components must be serializable to JSON (no functions, class instances, or Dates in their object form):

```tsx
// ✅ OK
<ClientComponent count={42} label="hello" items={['a', 'b']} />

// ❌ Cannot pass
<ClientComponent fn={() => {}} date={new Date()} />
```

---

## When to Use Which

**Use Server Components for:**
- Data fetching and database queries
- Accessing secrets/env vars
- Large dependencies you don't want in the bundle
- Static or rarely-changing content

**Use Client Components for:**
- Interactivity (click handlers, form inputs)
- State and effects (`useState`, `useEffect`)
- Browser APIs
- Third-party libraries that use hooks/events

---

## Common Interview Questions

**Q: Are Server Components the same as SSR?**
No. SSR renders React components on the server to HTML and then hydrates. RSC renders to a special format (RSC payload) and is never hydrated — it has no client-side representation. RSC and SSR are orthogonal and can be combined.

**Q: What's the RSC payload?**
A serialized tree format (not HTML, not JSON) that describes the rendered server component output, references to client component boundaries, and props. Next.js streams this to the browser.

**Q: Can RSC reduce bundle size?**
Significantly. A Server Component that uses a 500 KB parsing library contributes 0 bytes to the JS bundle. Only Client Components add to the bundle.

**Q: Can Server Components use React Context?**
They cannot consume context (Context requires a client-side provider). They can import and render context providers that wrap client subtrees.
