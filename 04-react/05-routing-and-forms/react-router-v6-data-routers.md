# React Router v6 Data Routers

## Table of Contents
1. [Introduction](#introduction)
2. [Data Router Overview](#data-router-overview)
3. [Route Loaders](#route-loaders)
4. [Route Actions](#route-actions)
5. [defer() and Await](#defer-and-await)
6. [Error Handling](#error-handling)
7. [Navigation State](#navigation-state)
8. [Advanced Patterns](#advanced-patterns)
9. [Common Mistakes](#common-mistakes)
10. [Best Practices](#best-practices)
11. [Interview Questions](#interview-questions)
12. [Resources](#resources)

## Introduction

React Router v6.4+ introduced data routers, a paradigm shift that brings data fetching, mutations, and error handling directly into the routing layer. This eliminates the need for useEffect-based data fetching and provides a more declarative, performant approach to building data-driven applications.

## Data Router Overview

### Traditional Router vs Data Router

```typescript
// Traditional approach (BrowserRouter)
import { BrowserRouter, Routes, Route } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/users/:id" element={<UserDetail />} />
      </Routes>
    </BrowserRouter>
  );
}

// Data router approach (createBrowserRouter)
import { createBrowserRouter, RouterProvider } from 'react-router-dom';

const router = createBrowserRouter([
  {
    path: '/',
    element: <Home />,
    loader: homeLoader,
    errorElement: <ErrorPage />
  },
  {
    path: '/users/:id',
    element: <UserDetail />,
    loader: userLoader,
    action: userAction,
    errorElement: <UserError />
  }
]);

function App() {
  return <RouterProvider router={router} />;
}
```

### Creating Data Routers

```typescript
import {
  createBrowserRouter,
  createMemoryRouter,
  createHashRouter
} from 'react-router-dom';

// Most common - HTML5 history API
const browserRouter = createBrowserRouter([
  // routes
]);

// For testing
const memoryRouter = createMemoryRouter([
  // routes
], {
  initialEntries: ['/users/1'],
  initialIndex: 0
});

// For legacy browser support
const hashRouter = createHashRouter([
  // routes
]);
```

### Route Configuration Object

```typescript
interface RouteObject {
  path?: string;
  index?: boolean;
  children?: RouteObject[];
  caseSensitive?: boolean;
  id?: string;
  loader?: LoaderFunction;
  action?: ActionFunction;
  element?: React.ReactNode;
  Component?: React.ComponentType;
  errorElement?: React.ReactNode;
  ErrorBoundary?: React.ComponentType;
  handle?: any;
  shouldRevalidate?: ShouldRevalidateFunction;
  lazy?: LazyRouteFunction<RouteObject>;
}
```

## Route Loaders

Loaders fetch data before the route component renders. They run in parallel with navigation, providing instant navigation and data loading.

### Basic Loader

```typescript
import { LoaderFunctionArgs, useLoaderData } from 'react-router-dom';

interface User {
  id: string;
  name: string;
  email: string;
}

// Loader function
async function userLoader({ params }: LoaderFunctionArgs): Promise<User> {
  const response = await fetch(`/api/users/${params.id}`);
  
  if (!response.ok) {
    throw new Response('User not found', { status: 404 });
  }
  
  return response.json();
}

// Component
function UserDetail() {
  const user = useLoaderData() as User;
  
  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}

// Route configuration
const router = createBrowserRouter([
  {
    path: '/users/:id',
    element: <UserDetail />,
    loader: userLoader
  }
]);
```

### Loader with Multiple Data Sources

```typescript
interface DashboardData {
  user: User;
  stats: Stats;
  recentActivity: Activity[];
}

async function dashboardLoader({ 
  request 
}: LoaderFunctionArgs): Promise<DashboardData> {
  const url = new URL(request.url);
  const period = url.searchParams.get('period') || '7d';
  
  // Parallel data fetching
  const [user, stats, activity] = await Promise.all([
    fetch('/api/user').then(r => r.json()),
    fetch(`/api/stats?period=${period}`).then(r => r.json()),
    fetch('/api/activity/recent').then(r => r.json())
  ]);
  
  return { user, stats, recentActivity: activity };
}

function Dashboard() {
  const { user, stats, recentActivity } = useLoaderData() as DashboardData;
  
  return (
    <div>
      <h1>Welcome, {user.name}</h1>
      <StatsGrid stats={stats} />
      <ActivityFeed activities={recentActivity} />
    </div>
  );
}
```

### Loader with Authentication

```typescript
import { redirect } from 'react-router-dom';

async function protectedLoader({ request }: LoaderFunctionArgs) {
  const user = await getAuthenticatedUser();
  
  if (!user) {
    const params = new URLSearchParams();
    params.set('from', new URL(request.url).pathname);
    return redirect('/login?' + params.toString());
  }
  
  return user;
}

// Combined loader
async function dashboardLoader({ request }: LoaderFunctionArgs) {
  // Check authentication first
  const user = await protectedLoader({ request } as LoaderFunctionArgs);
  
  if (user instanceof Response) {
    return user; // It's a redirect
  }
  
  // Fetch dashboard data
  const data = await fetch('/api/dashboard').then(r => r.json());
  
  return { user, ...data };
}
```

### Accessing Loader Data in Child Routes

```typescript
import { useRouteLoaderData } from 'react-router-dom';

// Parent route
const router = createBrowserRouter([
  {
    path: '/projects/:projectId',
    id: 'project', // Named route
    loader: projectLoader,
    element: <ProjectLayout />,
    children: [
      {
        index: true,
        element: <ProjectOverview />
      },
      {
        path: 'tasks',
        element: <ProjectTasks />
      }
    ]
  }
]);

// Parent loader
async function projectLoader({ params }: LoaderFunctionArgs) {
  return fetch(`/api/projects/${params.projectId}`).then(r => r.json());
}

// Child component accessing parent data
function ProjectTasks() {
  const project = useRouteLoaderData('project') as Project;
  
  return (
    <div>
      <h2>Tasks for {project.name}</h2>
      {/* ... */}
    </div>
  );
}
```

## Route Actions

Actions handle mutations (POST, PUT, DELETE) and automatically revalidate loaders after completion.

### Basic Action

```typescript
import { ActionFunctionArgs, Form, redirect } from 'react-router-dom';

// Action function
async function createUserAction({ request }: ActionFunctionArgs) {
  const formData = await request.formData();
  
  const newUser = {
    name: formData.get('name'),
    email: formData.get('email')
  };
  
  const response = await fetch('/api/users', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(newUser)
  });
  
  if (!response.ok) {
    return { error: 'Failed to create user' };
  }
  
  const user = await response.json();
  return redirect(`/users/${user.id}`);
}

// Component with Form
function CreateUser() {
  const actionData = useActionData() as { error?: string };
  
  return (
    <Form method="post">
      {actionData?.error && (
        <div className="error">{actionData.error}</div>
      )}
      
      <input type="text" name="name" required />
      <input type="email" name="email" required />
      <button type="submit">Create User</button>
    </Form>
  );
}

// Route
const router = createBrowserRouter([
  {
    path: '/users/new',
    element: <CreateUser />,
    action: createUserAction
  }
]);
```

### Action with Validation

```typescript
import { z } from 'zod';

const userSchema = z.object({
  name: z.string().min(2, 'Name must be at least 2 characters'),
  email: z.string().email('Invalid email address'),
  age: z.number().min(18, 'Must be 18 or older')
});

type ValidationErrors = Partial<Record<keyof z.infer<typeof userSchema>, string>>;

async function userAction({ request }: ActionFunctionArgs) {
  const formData = await request.formData();
  
  const data = {
    name: formData.get('name') as string,
    email: formData.get('email') as string,
    age: Number(formData.get('age'))
  };
  
  // Validate
  const result = userSchema.safeParse(data);
  
  if (!result.success) {
    const errors: ValidationErrors = {};
    result.error.issues.forEach(issue => {
      errors[issue.path[0] as keyof typeof data] = issue.message;
    });
    return { errors };
  }
  
  // Save user
  const response = await fetch('/api/users', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(result.data)
  });
  
  if (!response.ok) {
    return { errors: { _form: 'Failed to create user' } };
  }
  
  return redirect('/users');
}

function CreateUserForm() {
  const actionData = useActionData() as { 
    errors?: ValidationErrors & { _form?: string } 
  };
  const navigation = useNavigation();
  const isSubmitting = navigation.state === 'submitting';
  
  return (
    <Form method="post">
      {actionData?.errors?._form && (
        <div className="error">{actionData.errors._form}</div>
      )}
      
      <div>
        <input type="text" name="name" />
        {actionData?.errors?.name && (
          <span className="error">{actionData.errors.name}</span>
        )}
      </div>
      
      <div>
        <input type="email" name="email" />
        {actionData?.errors?.email && (
          <span className="error">{actionData.errors.email}</span>
        )}
      </div>
      
      <div>
        <input type="number" name="age" />
        {actionData?.errors?.age && (
          <span className="error">{actionData.errors.age}</span>
        )}
      </div>
      
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Creating...' : 'Create User'}
      </button>
    </Form>
  );
}
```

### Programmatic Form Submission

```typescript
import { useSubmit, useFetcher } from 'react-router-dom';

// Using useSubmit
function DeleteButton({ userId }: { userId: string }) {
  const submit = useSubmit();
  
  const handleDelete = () => {
    if (confirm('Are you sure?')) {
      submit(
        { userId },
        { method: 'delete', action: `/users/${userId}` }
      );
    }
  };
  
  return <button onClick={handleDelete}>Delete</button>;
}

// Using useFetcher (doesn't cause navigation)
function LikeButton({ postId, liked }: { postId: string; liked: boolean }) {
  const fetcher = useFetcher();
  
  const isLiking = fetcher.state !== 'idle';
  const currentLiked = fetcher.formData 
    ? fetcher.formData.get('liked') === 'true'
    : liked;
  
  return (
    <fetcher.Form method="post" action={`/posts/${postId}/like`}>
      <input type="hidden" name="liked" value={String(!currentLiked)} />
      <button type="submit" disabled={isLiking}>
        {currentLiked ? '❤️' : '🤍'}
      </button>
    </fetcher.Form>
  );
}
```

## defer() and Await

`defer()` enables streaming data and rendering critical content first while slower data loads.

### Basic defer Usage

```typescript
import { defer, Await } from 'react-router-dom';
import { Suspense } from 'react';

interface DashboardData {
  critical: Promise<CriticalData>;
  slow: Promise<SlowData>;
}

async function dashboardLoader(): Promise<DeferredData<DashboardData>> {
  // Start both requests
  const criticalPromise = fetch('/api/critical').then(r => r.json());
  const slowPromise = fetch('/api/slow').then(r => r.json());
  
  // Wait for critical, defer slow
  const critical = await criticalPromise;
  
  return defer({
    critical, // Resolved
    slow: slowPromise // Still pending
  });
}

function Dashboard() {
  const data = useLoaderData() as DashboardData;
  
  return (
    <div>
      {/* Renders immediately */}
      <CriticalSection data={data.critical} />
      
      {/* Renders when slow data resolves */}
      <Suspense fallback={<Spinner />}>
        <Await resolve={data.slow}>
          {(slowData) => <SlowSection data={slowData} />}
        </Await>
      </Suspense>
    </div>
  );
}
```

### defer with Error Handling

```typescript
function Dashboard() {
  const data = useLoaderData() as DashboardData;
  
  return (
    <div>
      <CriticalSection data={data.critical} />
      
      <Suspense fallback={<Spinner />}>
        <Await
          resolve={data.slow}
          errorElement={<ErrorMessage message="Failed to load data" />}
        >
          {(slowData) => <SlowSection data={slowData} />}
        </Await>
      </Suspense>
    </div>
  );
}
```

### Multiple Deferred Resources

```typescript
interface ProductPageData {
  product: Product; // Critical - wait
  reviews: Promise<Review[]>; // Defer
  recommendations: Promise<Product[]>; // Defer
  inventory: Promise<Inventory>; // Defer
}

async function productLoader({ params }: LoaderFunctionArgs) {
  const product = await fetch(`/api/products/${params.id}`)
    .then(r => r.json());
  
  return defer({
    product,
    reviews: fetch(`/api/products/${params.id}/reviews`)
      .then(r => r.json()),
    recommendations: fetch(`/api/products/${params.id}/recommendations`)
      .then(r => r.json()),
    inventory: fetch(`/api/products/${params.id}/inventory`)
      .then(r => r.json())
  });
}

function ProductPage() {
  const data = useLoaderData() as ProductPageData;
  
  return (
    <div>
      <ProductHero product={data.product} />
      
      <div className="grid">
        <Suspense fallback={<SkeletonReviews />}>
          <Await resolve={data.reviews}>
            {(reviews) => <ReviewsList reviews={reviews} />}
          </Await>
        </Suspense>
        
        <Suspense fallback={<SkeletonRecommendations />}>
          <Await resolve={data.recommendations}>
            {(recs) => <RecommendationsList products={recs} />}
          </Await>
        </Suspense>
        
        <Suspense fallback={<SkeletonInventory />}>
          <Await resolve={data.inventory}>
            {(inv) => <InventoryInfo inventory={inv} />}
          </Await>
        </Suspense>
      </div>
    </div>
  );
}
```

## Error Handling

### Route-Level Error Boundaries

```typescript
import { useRouteError, isRouteErrorResponse } from 'react-router-dom';

function ErrorPage() {
  const error = useRouteError();
  
  if (isRouteErrorResponse(error)) {
    // Thrown Response object
    return (
      <div>
        <h1>{error.status} {error.statusText}</h1>
        <p>{error.data}</p>
      </div>
    );
  }
  
  if (error instanceof Error) {
    return (
      <div>
        <h1>Oops! Unexpected Error</h1>
        <p>{error.message}</p>
        <pre>{error.stack}</pre>
      </div>
    );
  }
  
  return <div>Unknown error</div>;
}

const router = createBrowserRouter([
  {
    path: '/',
    element: <Root />,
    errorElement: <ErrorPage />,
    children: [
      // Child routes inherit error boundary
    ]
  }
]);
```

### Nested Error Boundaries

```typescript
const router = createBrowserRouter([
  {
    path: '/',
    element: <Root />,
    errorElement: <RootError />, // Catches all unhandled errors
    children: [
      {
        path: 'dashboard',
        element: <Dashboard />,
        loader: dashboardLoader,
        errorElement: <DashboardError />, // More specific error UI
        children: [
          {
            path: 'analytics',
            element: <Analytics />,
            loader: analyticsLoader,
            errorElement: <AnalyticsError /> // Even more specific
          }
        ]
      }
    ]
  }
]);
```

### Error Handling in Loaders

```typescript
async function userLoader({ params }: LoaderFunctionArgs) {
  const response = await fetch(`/api/users/${params.id}`);
  
  if (!response.ok) {
    // Throw Response - caught by errorElement
    throw new Response('User not found', { 
      status: 404,
      statusText: 'Not Found'
    });
  }
  
  const user = await response.json();
  
  if (!user.active) {
    // Throw custom error
    throw new Error('User account is inactive');
  }
  
  return user;
}

// With JSON data
async function projectLoader({ params }: LoaderFunctionArgs) {
  const response = await fetch(`/api/projects/${params.id}`);
  
  if (!response.ok) {
    throw json(
      { message: 'Project not found', code: 'PROJECT_NOT_FOUND' },
      { status: 404 }
    );
  }
  
  return response.json();
}
```

## Navigation State

### useNavigation Hook

```typescript
import { useNavigation } from 'react-router-dom';

function GlobalLoadingIndicator() {
  const navigation = useNavigation();
  
  // navigation.state: "idle" | "loading" | "submitting"
  if (navigation.state === 'loading') {
    return (
      <div className="loading-bar">
        <div className="loading-progress" />
      </div>
    );
  }
  
  return null;
}

// More detailed state
function NavigationState() {
  const navigation = useNavigation();
  
  return (
    <div>
      <p>State: {navigation.state}</p>
      <p>Location: {navigation.location?.pathname}</p>
      <p>Form Data: {navigation.formData?.get('username')}</p>
      <p>Form Action: {navigation.formAction}</p>
      <p>Form Method: {navigation.formMethod}</p>
    </div>
  );
}
```

### Optimistic UI

```typescript
function TodoList() {
  const todos = useLoaderData() as Todo[];
  const fetcher = useFetcher();
  
  // Optimistic UI: show todo as complete immediately
  const optimisticTodos = todos.map(todo => {
    if (fetcher.formData?.get('todoId') === todo.id) {
      return {
        ...todo,
        completed: fetcher.formData.get('completed') === 'true'
      };
    }
    return todo;
  });
  
  return (
    <ul>
      {optimisticTodos.map(todo => (
        <li key={todo.id}>
          <fetcher.Form method="post" action={`/todos/${todo.id}`}>
            <input 
              type="hidden" 
              name="todoId" 
              value={todo.id} 
            />
            <input 
              type="hidden" 
              name="completed" 
              value={String(!todo.completed)} 
            />
            <button type="submit">
              {todo.completed ? '✓' : '○'} {todo.title}
            </button>
          </fetcher.Form>
        </li>
      ))}
    </ul>
  );
}
```

## Advanced Patterns

### Revalidation Control

```typescript
import { ShouldRevalidateFunction } from 'react-router-dom';

const shouldRevalidate: ShouldRevalidateFunction = ({
  currentUrl,
  currentParams,
  nextUrl,
  nextParams,
  formMethod,
  formAction,
  defaultShouldRevalidate
}) => {
  // Only revalidate if ID changed
  if (currentParams.id !== nextParams.id) {
    return true;
  }
  
  // Don't revalidate for GET navigation
  if (formMethod === 'GET') {
    return false;
  }
  
  // Use default behavior for other cases
  return defaultShouldRevalidate;
};

const router = createBrowserRouter([
  {
    path: '/users/:id',
    element: <UserProfile />,
    loader: userLoader,
    shouldRevalidate
  }
]);
```

### Lazy Route Loading

```typescript
const router = createBrowserRouter([
  {
    path: '/admin',
    lazy: async () => {
      const { AdminLayout, loader } = await import('./routes/admin');
      return {
        Component: AdminLayout,
        loader
      };
    },
    children: [
      {
        path: 'users',
        lazy: () => import('./routes/admin/users')
      },
      {
        path: 'settings',
        lazy: () => import('./routes/admin/settings')
      }
    ]
  }
]);

// In route file
export function Component() {
  return <AdminUsers />;
}

export async function loader({ request }: LoaderFunctionArgs) {
  // loader code
}

export async function action({ request }: ActionFunctionArgs) {
  // action code
}
```

### Route Handle for Breadcrumbs

```typescript
const router = createBrowserRouter([
  {
    path: '/',
    element: <Root />,
    handle: {
      crumb: () => <Link to="/">Home</Link>
    },
    children: [
      {
        path: 'projects',
        element: <ProjectList />,
        handle: {
          crumb: () => <Link to="/projects">Projects</Link>
        },
        children: [
          {
            path: ':projectId',
            element: <ProjectDetail />,
            loader: projectLoader,
            handle: {
              crumb: (data: Project) => (
                <Link to={`/projects/${data.id}`}>{data.name}</Link>
              )
            }
          }
        ]
      }
    ]
  }
]);

function Breadcrumbs() {
  const matches = useMatches();
  const crumbs = matches
    .filter(match => match.handle?.crumb)
    .map(match => match.handle.crumb(match.data));
  
  return (
    <nav>
      {crumbs.map((crumb, index) => (
        <span key={index}>
          {crumb}
          {index < crumbs.length - 1 && ' / '}
        </span>
      ))}
    </nav>
  );
}
```

## Common Mistakes

### 1. Not Handling Loader Errors

```typescript
// ❌ Bad - no error handling
async function userLoader({ params }: LoaderFunctionArgs) {
  const response = await fetch(`/api/users/${params.id}`);
  return response.json(); // Fails silently if 404
}

// ✅ Good - proper error handling
async function userLoader({ params }: LoaderFunctionArgs) {
  const response = await fetch(`/api/users/${params.id}`);
  
  if (!response.ok) {
    throw new Response('User not found', { status: 404 });
  }
  
  return response.json();
}
```

### 2. Blocking on Non-Critical Data

```typescript
// ❌ Bad - waiting for slow data
async function loader() {
  const critical = await fetchCritical();
  const slow = await fetchSlow(); // Delays page render
  return { critical, slow };
}

// ✅ Good - defer slow data
async function loader() {
  const critical = await fetchCritical();
  return defer({
    critical,
    slow: fetchSlow() // Don't await
  });
}
```

### 3. Not Using Correct Form Method

```typescript
// ❌ Bad - wrong method for mutation
<Form method="get" action="/users/delete">
  <button type="submit">Delete</button>
</Form>

// ✅ Good - correct method
<Form method="delete" action="/users/123">
  <button type="submit">Delete</button>
</Form>
```

### 4. Forgetting to Return from Actions

```typescript
// ❌ Bad - no return value
async function action({ request }: ActionFunctionArgs) {
  const formData = await request.formData();
  await saveData(formData);
  // No return - stays on same page with no feedback
}

// ✅ Good - return redirect or data
async function action({ request }: ActionFunctionArgs) {
  const formData = await request.formData();
  await saveData(formData);
  return redirect('/success');
}
```

## Best Practices

### 1. Organize Route Configuration

```typescript
// routes/index.ts
import { RouteObject } from 'react-router-dom';
import { rootRoutes } from './root';
import { authRoutes } from './auth';
import { dashboardRoutes } from './dashboard';

export const routes: RouteObject[] = [
  ...rootRoutes,
  ...authRoutes,
  ...dashboardRoutes
];

// routes/dashboard.ts
export const dashboardRoutes: RouteObject[] = [
  {
    path: '/dashboard',
    element: <DashboardLayout />,
    loader: dashboardLayoutLoader,
    children: [
      {
        index: true,
        element: <DashboardHome />,
        loader: dashboardHomeLoader
      },
      // more routes
    ]
  }
];
```

### 2. Type-Safe Loaders

```typescript
// Create typed loader helper
function createTypedLoader<T>(
  loader: (args: LoaderFunctionArgs) => Promise<T>
) {
  return loader;
}

interface UserData {
  user: User;
  posts: Post[];
}

const userLoader = createTypedLoader<UserData>(async ({ params }) => {
  const [user, posts] = await Promise.all([
    fetchUser(params.id),
    fetchUserPosts(params.id)
  ]);
  
  return { user, posts };
});

function UserPage() {
  const data = useLoaderData() as UserData; // Type-safe
  return <div>{data.user.name}</div>;
}
```

### 3. Centralized Error Handling

```typescript
// utils/loaderHelpers.ts
export async function fetchWithErrorHandling(
  url: string,
  options?: RequestInit
) {
  const response = await fetch(url, options);
  
  if (!response.ok) {
    throw json(
      { message: 'Request failed', status: response.status },
      { status: response.status }
    );
  }
  
  return response.json();
}

// Use in loaders
async function userLoader({ params }: LoaderFunctionArgs) {
  return fetchWithErrorHandling(`/api/users/${params.id}`);
}
```

### 4. Loading States

```typescript
function Root() {
  const navigation = useNavigation();
  const isNavigating = navigation.state === 'loading';
  
  return (
    <div>
      <header>
        <nav className={isNavigating ? 'loading' : ''}>
          {/* navigation */}
        </nav>
      </header>
      
      <main className={isNavigating ? 'opacity-50' : ''}>
        <Outlet />
      </main>
      
      {isNavigating && <GlobalSpinner />}
    </div>
  );
}
```

## Interview Questions

### Q1: What's the difference between loader and useEffect for data fetching?

**Answer:** Loaders fetch data before rendering, preventing waterfalls and loading states. useEffect fetches after render, causing:
- Loading spinners
- Layout shifts
- Sequential requests (parent renders → child useEffect fires)

Loaders enable:
- Parallel data fetching (all route loaders run together)
- Navigation delay until data ready
- Error handling at route level
- Form actions that auto-revalidate

### Q2: When should you use defer()?

**Answer:** Use defer when:
- You have slow, non-critical data
- You want to render critical UI immediately
- Progressive loading improves UX

Don't use when:
- All data is fast (<100ms)
- All data is critical for initial render
- SEO requires complete data

### Q3: How do actions differ from traditional form handling?

**Answer:**
- Actions are route-level, not component-level
- Automatically revalidate loaders after completion
- Work with/without JavaScript (progressive enhancement)
- Integrate with navigation state
- Support optimistic UI patterns

### Q4: Explain the revalidation lifecycle.

**Answer:**
1. Action completes successfully
2. All loaders for current page revalidate (by default)
3. UI updates with fresh data
4. Can control with `shouldRevalidate` function
5. Fetchers can revalidate without navigation

## Resources

### Official Documentation
- React Router Docs - Data APIs: https://reactrouter.com/en/main/routers/picking-a-router
- Remix Philosophy (inspires RR): https://remix.run/docs/en/main/discussion/data-flow

### Tutorials
- React Router 6.4+ Tutorial: https://reactrouter.com/en/main/start/tutorial
- Data Loading Deep Dive: https://www.youtube.com/watch?v=95B8mnhzoCM

### Articles
- "When to Fetch" by Ryan Florence: https://www.youtube.com/watch?v=95B8mnhzoCM
- Remix Data Patterns: https://remix.run/docs/en/main/discussion/data-flow

### Tools
- React Router DevTools: https://github.com/remix-run/react-router-devtools

---

**Next Steps:**
- Practice converting useEffect-based apps to loaders
- Build form-heavy applications using actions
- Implement optimistic UI patterns
- Explore defer() for performance optimization
