# React Router v6/v7

## Overview

React Router is the standard routing solution for React applications. v6 introduced a complete rewrite with nested routes and data loading built in. v7 (React Router 7 / Remix v3) merges Remix's full-stack features into React Router.

---

## Core Setup (v6)

```tsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';

const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <ErrorPage />,
    children: [
      { index: true, element: <HomePage /> },
      {
        path: 'users',
        element: <UsersLayout />,
        loader: usersLoader,     // load data before rendering
        children: [
          { index: true, element: <UsersList /> },
          { path: ':userId', element: <UserDetail />, loader: userLoader },
        ],
      },
    ],
  },
]);

function App() {
  return <RouterProvider router={router} />;
}
```

---

## Loaders (Data Loading)

Loaders run before the component renders — no loading state needed in the component:

```ts
// loader function
export async function userLoader({ params }: LoaderFunctionArgs) {
  const user = await fetchUser(params.userId);
  if (!user) throw new Response('Not Found', { status: 404 });
  return user;
}

// Component uses the data synchronously
function UserDetail() {
  const user = useLoaderData() as User;
  return <div>{user.name}</div>;
}
```

All parallel loaders run concurrently — parent and child loaders execute simultaneously.

---

## Actions (Mutations)

Actions handle form submissions and mutations:

```ts
export async function createUserAction({ request }: ActionFunctionArgs) {
  const formData = await request.formData();
  const name = formData.get('name') as string;

  try {
    const user = await api.createUser({ name });
    return redirect(`/users/${user.id}`);
  } catch (error) {
    return { error: 'Failed to create user' };
  }
}

function CreateUser() {
  const actionData = useActionData() as { error?: string };
  const navigation = useNavigation();
  const isSubmitting = navigation.state === 'submitting';

  return (
    <Form method="post">
      <input name="name" />
      {actionData?.error && <p>{actionData.error}</p>}
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Creating...' : 'Create'}
      </button>
    </Form>
  );
}
```

---

## Key Hooks

```tsx
// Navigation
const navigate = useNavigate();
navigate('/users'); navigate(-1); navigate('/users', { replace: true });

// URL parameters
const { userId } = useParams<{ userId: string }>();

// Query string
const [searchParams, setSearchParams] = useSearchParams();
const query = searchParams.get('q');

// Location
const location = useLocation(); // { pathname, search, hash, state }

// Navigation state (for pending UI)
const navigation = useNavigation();
// navigation.state: 'idle' | 'loading' | 'submitting'
```

---

## Nested Routes and Layout Routes

```tsx
// Layout route: renders Outlet for children
function UsersLayout() {
  return (
    <div>
      <UsersSidebar />
      <Outlet />  {/* renders matched child route */}
    </div>
  );
}

// Routes config
{
  path: 'users',
  element: <UsersLayout />,  // renders for all /users/* routes
  children: [
    { index: true, element: <UsersList /> },     // /users
    { path: ':id', element: <UserDetail /> },     // /users/123
    { path: 'new', element: <CreateUser /> },     // /users/new
  ]
}
```

---

## Route Protection

```ts
// Loader-based guard (recommended)
export async function protectedLoader({ request }: LoaderFunctionArgs) {
  const user = await getUser();
  if (!user) {
    throw redirect(`/login?from=${new URL(request.url).pathname}`);
  }
  return user;
}

// Component-based guard (alternative)
function RequireAuth({ children }: { children: ReactNode }) {
  const { user } = useAuth();
  const location = useLocation();
  if (!user) return <Navigate to="/login" state={{ from: location }} replace />;
  return children;
}
```

---

## Comparison with TanStack Router

| | React Router v7 | TanStack Router |
|---|---|---|
| Type safety | Basic | Full end-to-end TypeScript |
| Search params | `useSearchParams` (string-based) | Fully typed search params |
| File-based routing | Optional (Vite plugin) | Core feature |
| Maturity | Battle-tested | Newer, growing adoption |
| Bundle size | ~13 KB | ~13 KB |
| Data loading | Loaders/actions | Loaders with caching |

---

## Common Interview Questions

**Q: What's the difference between index routes and layout routes?**
Index routes (`{ index: true }`) match the parent path exactly with no additional path. Layout routes render an `<Outlet>` for child routes — they provide shared UI for a group of routes.

**Q: How do concurrent loaders work?**
When navigating to a nested route, React Router runs all matching loaders concurrently (not sequentially). A page with 3 nested levels runs all 3 loaders in parallel, only blocking render until all complete.

**Q: When should you use `<Link>` vs `<Form>`?**
`<Link>` for navigation (GET requests that load a page). `<Form>` for mutations (POST/PUT/DELETE that trigger actions). `<Form>` integrates with React Router's action system and navigation state.
