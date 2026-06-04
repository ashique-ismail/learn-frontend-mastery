# TanStack Router

## Overview

TanStack Router provides end-to-end type safety for routing — the type system knows every route, path parameter, and search parameter. It's the most type-safe routing solution in the React ecosystem.

---

## Key Differentiator: Full Type Safety

```tsx
// Route definition includes type information
const userRoute = createRoute({
  getParentRoute: () => usersRoute,
  path: '$userId',
  validateSearch: (search) => ({
    tab: z.enum(['overview', 'activity']).default('overview').parse(search.tab),
  }),
  loader: ({ params }) => fetchUser(params.userId), // params.userId is typed string
});

// Usage — fully typed
function UserDetail() {
  const { userId } = userRoute.useParams(); // typed string
  const { tab } = userRoute.useSearch();    // typed 'overview' | 'activity'
  const user = userRoute.useLoaderData();   // typed from loader return
}

// Navigation — type-checked at compile time
<Link to="/users/$userId" params={{ userId: user.id }} search={{ tab: 'activity' }} />
// TypeScript error if: wrong path, missing params, wrong search param types
```

---

## Setup

```tsx
import { createRouter, RouterProvider } from '@tanstack/react-router';

// File-based routing (recommended)
// src/routes/users.$userId.tsx automatically becomes /users/:userId route

// Or manual route tree
const routeTree = rootRoute.addChildren([
  indexRoute,
  usersRoute.addChildren([userRoute]),
]);

const router = createRouter({ routeTree });

// Type-safe router instance
declare module '@tanstack/react-router' {
  interface Register {
    router: typeof router;
  }
}

function App() {
  return <RouterProvider router={router} />;
}
```

---

## File-Based Routing

```
src/routes/
├── __root.tsx          → root layout (/)
├── index.tsx           → /
├── users/
│   ├── index.tsx       → /users
│   └── $userId.tsx     → /users/:userId
├── _auth.tsx           → layout route (no path)
└── _auth.dashboard.tsx → /dashboard (under _auth layout)
```

```tsx
// src/routes/users/$userId.tsx
import { createFileRoute } from '@tanstack/react-router';

export const Route = createFileRoute('/users/$userId')({
  loader: async ({ params }) => {
    return await fetchUser(params.userId);
  },
  component: UserDetail,
});

function UserDetail() {
  const user = Route.useLoaderData();
  const { userId } = Route.useParams();
  return <div>{user.name}</div>;
}
```

---

## Search Params with Validation

```tsx
import { z } from 'zod';

const searchSchema = z.object({
  query: z.string().default(''),
  page: z.number().int().min(1).default(1),
  sortBy: z.enum(['name', 'date', 'relevance']).default('relevance'),
});

export const Route = createFileRoute('/search')({
  validateSearch: searchSchema,
  component: SearchPage,
});

function SearchPage() {
  const { query, page, sortBy } = Route.useSearch();
  const navigate = useNavigate({ from: Route.fullPath });

  return (
    <input
      value={query}
      onChange={(e) =>
        navigate({ search: (prev) => ({ ...prev, query: e.target.value, page: 1 }) })
      }
    />
  );
}
```

---

## Loaders with Caching

```tsx
export const Route = createFileRoute('/users/$userId')({
  loader: async ({ params, context }) => {
    // context: { queryClient } — inject dependencies
    await context.queryClient.ensureQueryData({
      queryKey: ['user', params.userId],
      queryFn: () => fetchUser(params.userId),
    });
  },
  // loaderDeps: re-run loader when search params change
  loaderDeps: ({ search: { tab } }) => ({ tab }),
  staleTime: 10_000, // cache for 10 seconds
});
```

---

## Route Context

Pass data through the route tree without prop drilling:

```tsx
const rootRoute = createRootRouteWithContext<{
  queryClient: QueryClient;
  auth: AuthContext;
}>()({
  component: Root,
});

const router = createRouter({
  routeTree,
  context: { queryClient, auth: getAuth() },
});

// Access in any loader
loader: async ({ context }) => {
  const user = await context.auth.getUser();
}
```

---

## Comparison with React Router

| | TanStack Router | React Router v7 |
|---|---|---|
| Search param types | Full schema validation | String-based |
| Route type safety | End-to-end TypeScript | Basic |
| File-based routing | Built-in (Vite plugin) | Optional |
| Loaders | With caching (staleTime) | One-time execution |
| Bundle | ~13 KB | ~13 KB |
| Ecosystem | Smaller, growing | Huge, mature |

---

## Common Interview Questions

**Q: How does TanStack Router's type safety differ from React Router's?**
TanStack Router generates a route tree type from your route definitions. Every `<Link>`, `navigate()`, and `useParams()` call is checked against the route tree — TypeScript errors if you use wrong paths, miss required params, or pass wrong search param types. React Router has basic typing but doesn't catch wrong route paths at compile time.

**Q: When would you choose TanStack Router over React Router?**
When type safety for navigation and search params is critical — especially for large codebases where route changes cause bugs. Also when you want search params to behave like first-class state (validated, parsed, typed) rather than raw strings.
