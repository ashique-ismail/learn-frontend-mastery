# Next.js App Router

## Overview

The App Router (introduced in Next.js 13, stable in 14) is a React Server Components-based routing system that colocates data fetching with components. It replaces the Pages Router's `getServerSideProps`/`getStaticProps` model.

---

## File Conventions

```
app/
├── layout.tsx              → root layout (wraps all pages)
├── page.tsx                → / (home page)
├── loading.tsx             → loading UI for the route
├── error.tsx               → error boundary for the route
├── not-found.tsx           → 404 for the route
├── global-error.tsx        → root-level error boundary
├── route.ts                → API route handler
├── users/
│   ├── layout.tsx          → /users layout (wraps users pages)
│   ├── page.tsx            → /users page
│   └── [id]/
│       ├── page.tsx        → /users/:id page
│       └── edit/
│           └── page.tsx    → /users/:id/edit page
└── (marketing)/            → route group (no URL segment)
    ├── about/page.tsx      → /about
    └── pricing/page.tsx    → /pricing
```

---

## Server Components (Default)

```tsx
// app/users/[id]/page.tsx — Server Component by default
interface PageProps {
  params: { id: string };
  searchParams: { tab?: string };
}

export default async function UserPage({ params, searchParams }: PageProps) {
  // Direct DB access — runs on server, never sent to client
  const user = await db.users.findById(params.id);

  if (!user) notFound();  // renders not-found.tsx

  return (
    <div>
      <h1>{user.name}</h1>
      <UserTabs userId={user.id} activeTab={searchParams.tab} />
    </div>
  );
}

// SEO metadata
export async function generateMetadata({ params }: PageProps) {
  const user = await db.users.findById(params.id);
  return {
    title: `${user.name} — Profile`,
    description: user.bio,
  };
}
```

---

## Layouts

```tsx
// app/layout.tsx — root layout (wraps everything)
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <NavigationBar />
        <main>{children}</main>
        <Footer />
      </body>
    </html>
  );
}

// app/users/layout.tsx — nested layout (wraps all /users/* pages)
export default function UsersLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="users-layout">
      <UsersSidebar />
      <div className="content">{children}</div>
    </div>
  );
}
```

---

## Parallel Routes

Render multiple pages in the same layout simultaneously:

```
app/
├── layout.tsx
├── @team/           → named slot
│   └── page.tsx
└── @analytics/      → named slot
    └── page.tsx
```

```tsx
// app/layout.tsx — receives named slots as props
export default function Layout({
  children,
  team,     // @team slot
  analytics // @analytics slot
}: {
  children: React.ReactNode;
  team: React.ReactNode;
  analytics: React.ReactNode;
}) {
  return (
    <div>
      {children}
      <div className="panels">
        {team}       {/* rendered independently */}
        {analytics}  {/* rendered independently */}
      </div>
    </div>
  );
}
```

Use case: dashboard with independent loading sections, tab UIs.

---

## Intercepting Routes

Show a modal overlay while keeping the background page, but show full page on direct navigation:

```
app/
├── photos/
│   └── [id]/page.tsx    → /photos/1 (full page)
└── (.)photos/           → intercepts /photos/* navigations
    └── [id]/page.tsx    → shows as modal when navigating from feed
```

```
(.)  — intercept from same level
(..) — intercept from one level up
(..)(..) — two levels up
(...)    — from root
```

---

## Server Actions

```tsx
'use server'; // can be at top of file or on the function

async function createUser(formData: FormData) {
  'use server';

  const name = formData.get('name') as string;
  const user = await db.users.create({ name });

  revalidatePath('/users'); // invalidate cached route
  redirect(`/users/${user.id}`);
}

// Usage in Server Component
export default function Page() {
  return (
    <form action={createUser}>
      <input name="name" />
      <button type="submit">Create</button>
    </form>
  );
}

// Or in Client Component
'use client';
import { createUser } from './actions';

export function UserForm() {
  const [state, formAction] = useFormState(createUser, null);
  return <form action={formAction}>...</form>;
}
```

---

## Route Handlers

```ts
// app/api/users/route.ts
import { NextResponse } from 'next/server';

export async function GET(request: Request) {
  const users = await db.users.findAll();
  return NextResponse.json(users);
}

export async function POST(request: Request) {
  const body = await request.json();
  const user = await db.users.create(body);
  return NextResponse.json(user, { status: 201 });
}
```

---

## `loading.tsx` — Instant Loading States

```tsx
// app/users/loading.tsx
// Shown while the page.tsx async function is resolving
export default function UsersLoading() {
  return <UserListSkeleton />;
}
```

React Suspense + streaming SSR: the layout HTML streams immediately, the loading skeleton shows while data loads.

---

## Common Interview Questions

**Q: What's the difference between `layout.tsx` and `template.tsx`?**
`layout.tsx` persists across navigations (state is preserved). `template.tsx` re-mounts on every navigation (fresh state, new instance). Use `template` when you need fresh state on each visit (e.g., analytics page views).

**Q: How do you share data between layout and page?**
Not via props (layouts don't receive page props). Use: React Context (client-side), `cookies()` / `headers()` (server-side), or fetch the same data in both (Next.js deduplicates identical `fetch` calls within a render pass).

**Q: What's the `cache()` function?**
`import { cache } from 'react'` memoizes a function's return value within a single render pass. Useful to share expensive queries between a layout and a page component without calling the DB twice.
