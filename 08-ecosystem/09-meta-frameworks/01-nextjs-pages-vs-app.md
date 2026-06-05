# Next.js Pages Router vs App Router

## The Idea

**In plain English:** Next.js is a tool for building websites, and it gives you two different ways to organize the pages of your site — an older, familiar way (Pages Router) and a newer, more powerful way (App Router). Choosing between them is about deciding which system of rules your project follows for turning files into web pages.

**Real-world analogy:** Imagine a hotel where every room is a destination guests can visit. The old hotel (Pages Router) requires every room key to be collected from a single front desk, and a staff member must personally hand you what you need before you enter. The new hotel (App Router) lets you walk straight into any room and grab what you need yourself, with some rooms being staff-only and others open to guests.

- The hotel rooms = the pages/routes of your website
- Collecting keys at the front desk = `getServerSideProps`/`getStaticProps` fetching data before the page loads
- Walking in and grabbing things yourself = async Server Components fetching data directly inside the component
- Staff-only rooms = Server Components (run on the server, never sent to the browser)
- Rooms open to guests = Client Components (marked with `'use client'`, run in the browser)

---

## Overview

Next.js has two routing systems:
- **Pages Router** (`/pages` directory) — original system, stable since Next.js 9
- **App Router** (`/app` directory) — new system based on React Server Components, stable since Next.js 13.4

Both can coexist in the same project during migration.

---

## File Structure Comparison

```
Pages Router          App Router
pages/                app/
├── index.tsx         ├── page.tsx         (/ route)
├── about.tsx         ├── about/
├── users/            │   └── page.tsx
│   ├── index.tsx     ├── users/
│   └── [id].tsx      │   ├── page.tsx
├── api/              │   └── [id]/
│   └── users.ts      │       └── page.tsx
└── _app.tsx          ├── layout.tsx       (root layout)
                      └── api/users/
                          └── route.ts
```

---

## Data Fetching Comparison

### Pages Router

```tsx
// pages/users/[id].tsx
export async function getServerSideProps(context: GetServerSidePropsContext) {
  const user = await fetchUser(context.params.id);
  return { props: { user } };
}

// Or static generation
export async function getStaticProps(context: GetStaticPropsContext) {
  const user = await fetchUser(context.params.id);
  return { props: { user }, revalidate: 60 };
}

export async function getStaticPaths() {
  const users = await fetchAllUsers();
  return { paths: users.map(u => ({ params: { id: u.id } })), fallback: 'blocking' };
}

export default function UserPage({ user }: { user: User }) {
  return <div>{user.name}</div>;
}
```

### App Router

```tsx
// app/users/[id]/page.tsx
export default async function UserPage({ params }: { params: { id: string } }) {
  const user = await fetchUser(params.id); // directly in the component
  return <div>{user.name}</div>;
}

// ISR equivalent
export const revalidate = 60;

// Static generation equivalent
export async function generateStaticParams() {
  const users = await fetchAllUsers();
  return users.map(u => ({ id: u.id }));
}
```

---

## Key Architectural Differences

| Aspect | Pages Router | App Router |
|---|---|---|
| Default rendering | SSR/SSG via `getServerSideProps`/`getStaticProps` | Server Components (RSC) |
| Client components | All components are client-capable | Explicit `'use client'` |
| Data fetching | Per-page functions | Anywhere in server components |
| Layouts | `_app.tsx` + `_document.tsx` | Nested `layout.tsx` files |
| Colocated loading | Not built-in | `loading.tsx` (Suspense) |
| Error handling | Custom `_error.tsx` | Per-route `error.tsx` |
| Middleware | `middleware.ts` | `middleware.ts` (same) |
| React Context | Works everywhere | Only in `'use client'` |

---

## API Routes Comparison

```ts
// Pages Router: pages/api/users.ts
import type { NextApiRequest, NextApiResponse } from 'next';

export default function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method === 'GET') {
    res.status(200).json({ users: [] });
  } else {
    res.setHeader('Allow', ['GET']);
    res.status(405).end('Method Not Allowed');
  }
}

// App Router: app/api/users/route.ts
import { NextResponse } from 'next/server';

export async function GET() {
  return NextResponse.json({ users: [] });
}

export async function POST(request: Request) {
  const body = await request.json();
  return NextResponse.json(body, { status: 201 });
}
```

---

## Coexistence During Migration

```
my-app/
├── pages/         ← Pages Router routes (still work)
│   └── legacy/
│       └── old-page.tsx
├── app/           ← App Router routes (new)
│   ├── layout.tsx
│   └── new-feature/
│       └── page.tsx
└── next.config.ts
```

Pages Router and App Router coexist. Same path can't exist in both — App Router takes precedence.

---

## When to Use Pages Router

1. **Existing codebase** that hasn't been migrated
2. **Heavy reliance on `getServerSideProps`** patterns your team knows well
3. **Third-party libraries** that use Context heavily (they need `'use client'`)
4. **Time pressure** — App Router has a steeper learning curve

---

## Migration Checklist

```
□ Identify routes to migrate (start with simple static pages)
□ Move page files from pages/ to app/
□ Replace getStaticProps/getServerSideProps with async Server Components
□ Add 'use client' to components that use hooks, state, or event handlers
□ Replace _app.tsx wrapper with layout.tsx
□ Replace API routes with Route Handlers
□ Update React Context providers to be Client Components
□ Test: check for hydration mismatches, loading states, error handling
```

---

## Common Interview Questions

**Q: Can `getServerSideProps` and React Server Components do the same thing?**
Similar outcomes but different mechanisms. `getServerSideProps` runs on the server and passes serialized props to a client component. RSC runs on the server and streams its output directly — no explicit props/serialization, and the component itself is never sent to the browser.

**Q: Why do we need `'use client'` in the App Router?**
App Router defaults to Server Components. `'use client'` marks a file as a client boundary — this file and all its imports are bundled for the browser. Without it, the entire component tree would be server-only.

**Q: Is the Pages Router deprecated?**
No. Vercel has committed to maintaining it. But new features are built for the App Router, and the Next.js team recommends App Router for new projects.
