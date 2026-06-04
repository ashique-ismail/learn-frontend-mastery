# Remix / React Router 7

## What It Is

Remix (merged into React Router 7 as of 2024) is a full-stack React framework that embraces web fundamentals: HTTP, forms, progressive enhancement. Data loading and mutations are colocated with routes.

---

## Core Mental Model

Every route can have:
- **loader** — loads data for GET requests (runs before render)
- **action** — handles form submissions / mutations (POST/PUT/DELETE)
- **component** — renders the UI with the loaded data

---

## Loaders

```tsx
// routes/users.$userId.tsx
import { type LoaderFunctionArgs } from '@remix-run/node';
import { useLoaderData } from '@remix-run/react';

export async function loader({ params }: LoaderFunctionArgs) {
  const user = await db.findUser(params.userId);
  if (!user) throw new Response('Not Found', { status: 404 });
  return json(user); // json() helper sets Content-Type header
}

export default function UserProfile() {
  const user = useLoaderData<typeof loader>(); // typed from loader
  return <div>{user.name}</div>;
}
```

---

## Actions

```tsx
import { type ActionFunctionArgs, redirect } from '@remix-run/node';
import { Form, useActionData } from '@remix-run/react';

export async function action({ request, params }: ActionFunctionArgs) {
  const formData = await request.formData();
  const name = formData.get('name') as string;

  const errors: Record<string, string> = {};
  if (!name) errors.name = 'Name is required';
  if (Object.keys(errors).length > 0) return json({ errors }, { status: 400 });

  await db.updateUser(params.userId, { name });
  return redirect(`/users/${params.userId}`);
}

export default function EditUser() {
  const actionData = useActionData<typeof action>();

  return (
    <Form method="post">
      <input name="name" />
      {actionData?.errors?.name && <p>{actionData.errors.name}</p>}
      <button type="submit">Save</button>
    </Form>
  );
}
```

---

## `<Form>` — Progressive Enhancement

The `<Form>` component works without JavaScript enabled (falls back to native form submission) and with JavaScript (intercepts and uses fetch):

```tsx
import { Form, useNavigation } from '@remix-run/react';

function EditForm() {
  const navigation = useNavigation();
  const isSubmitting = navigation.state === 'submitting';

  return (
    <Form method="post">
      <input name="title" />
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Saving...' : 'Save'}
      </button>
    </Form>
  );
}
```

No `useState` or `useEffect` needed — the framework manages form state.

---

## `useFetcher` — Background Requests

For mutations that don't cause navigation:

```tsx
function LikeButton({ postId }: { postId: string }) {
  const fetcher = useFetcher<typeof action>();
  const isOptimistic = fetcher.state !== 'idle';

  return (
    <fetcher.Form method="post" action={`/posts/${postId}/like`}>
      <button type="submit">
        {isOptimistic ? '❤️ (saving)' : '🤍 Like'}
      </button>
    </fetcher.Form>
  );
}
```

---

## Nested Routes and Nested Data Loading

Routes nest, and their loaders run concurrently:

```
routes/
├── _layout.tsx         → parent layout with <Outlet />
├── _layout.users.tsx   → /users (nested under layout)
└── _layout.users.$id.tsx → /users/:id (nested under users)
```

```tsx
// _layout.users.tsx
export async function loader() {
  return json({ users: await db.getUsers() }); // runs in parallel with child loader
}

// _layout.users.$id.tsx
export async function loader({ params }: LoaderFunctionArgs) {
  return json({ user: await db.getUser(params.id) }); // runs simultaneously
}
```

Remix orchestrates parallel data loading across the nested route tree.

---

## Error Handling

Each route can have its own error boundary:

```tsx
// routes/users.$userId.tsx
export function ErrorBoundary() {
  const error = useRouteError();

  if (isRouteErrorResponse(error)) {
    return <div>HTTP {error.status}: {error.data}</div>;
  }
  return <div>Unexpected error</div>;
}
```

Errors are isolated — a broken widget doesn't crash the page.

---

## Deferred Data (Streaming)

```tsx
import { defer } from '@remix-run/node';
import { Await, useLoaderData } from '@remix-run/react';

export async function loader() {
  return defer({
    user: await db.getUser(),           // await this — needed for initial render
    recommendations: db.getRecommendations(), // don't await — stream later
  });
}

export default function Page() {
  const { user, recommendations } = useLoaderData<typeof loader>();
  return (
    <div>
      <h1>{user.name}</h1>
      <Suspense fallback={<Skeleton />}>
        <Await resolve={recommendations}>
          {(recs) => <RecommendationList items={recs} />}
        </Await>
      </Suspense>
    </div>
  );
}
```

---

## Comparison with Next.js App Router

| | Remix / RR7 | Next.js App Router |
|---|---|---|
| Data loading | Loaders (route-level) | RSC async functions (component-level) |
| Mutations | Actions + Form | Server Actions |
| Routing | File-based | File-based |
| Progressive enhancement | First-class | Optional |
| Caching | HTTP cache, no client cache | Extensive built-in caching |
| Streaming | defer() | Suspense + streaming |
| Rendering | SSR first | RSC + SSR + SSG |
| Learning curve | Lower (web fundamentals) | Higher (RSC mental model) |

---

## Common Interview Questions

**Q: Why does Remix use `<Form>` instead of `<form>`?**
Remix's `<Form>` intercepts submissions with JavaScript and uses fetch, giving the UX benefits of SPA (no full reload, pending states). But when JS fails to load, it falls back to native form submission — progressive enhancement by default.

**Q: How is Remix different from the "old" React Router?**
React Router 6 was purely client-side routing. Remix added server-side capabilities (loaders, actions, SSR). React Router 7 merges Remix's full-stack features directly into React Router — they're now the same project.
