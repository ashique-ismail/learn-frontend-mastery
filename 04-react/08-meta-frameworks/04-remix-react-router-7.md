# Remix and React Router 7

## The Idea

**In plain English:** Remix (and its newer form, React Router 7) is a tool for building websites where the server and the browser work closely together — the server fetches your data and handles form submissions, while the browser displays everything and makes it feel fast and interactive. Think of it as a complete kit for building a web page that works even before JavaScript loads.

**Real-world analogy:** Imagine ordering food at a restaurant. You (the browser) sit at a table and look at a menu. A waiter (the loader) goes to the kitchen and brings back exactly the dishes you need. When you want to send something back or place a new order, you call the waiter again (the action), who takes your request to the kitchen and comes back with a result.

- The waiter fetching dishes = the loader (gets data from the server before the page renders)
- Placing a new order or sending food back = the action (handles form submissions and data changes)
- The table where you sit and see everything = the route (the page tied to a specific URL)

---

## Overview

Remix is a full-stack web framework that emphasizes web fundamentals, progressive enhancement, and excellent user experiences. React Router 7 represents the evolution of Remix, where the Remix framework has been extracted into React Router itself, unifying the ecosystem.

## Table of Contents

1. [File-Based Routing](#file-based-routing)
2. [Loaders and Actions](#loaders-and-actions)
3. [Data Hooks](#data-hooks)
4. [Form and Progressive Enhancement](#form-and-progressive-enhancement)
5. [Deferred Data and Streaming](#deferred-data-and-streaming)
6. [Error Boundaries](#error-boundaries)
7. [React Router 7](#react-router-7)
8. [Common Mistakes](#common-mistakes)
9. [Best Practices](#best-practices)
10. [Interview Questions](#interview-questions)

## File-Based Routing

Remix uses file-system based routing conventions in the `app/routes` directory.

### Basic Routes

```typescript
// app/routes/_index.tsx
export default function Index() {
  return <h1>Home Page</h1>;
}

// app/routes/about.tsx
export default function About() {
  return <h1>About Us</h1>;
}

// app/routes/contact.tsx
export default function Contact() {
  return <h1>Contact Us</h1>;
}
```

### Dynamic Routes

```typescript
// app/routes/posts.$slug.tsx
import { json, LoaderFunctionArgs } from '@remix-run/node';
import { useLoaderData } from '@remix-run/react';

export async function loader({ params }: LoaderFunctionArgs) {
  const post = await getPost(params.slug);
  return json({ post });
}

export default function Post() {
  const { post } = useLoaderData<typeof loader>();
  
  return (
    <article>
      <h1>{post.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </article>
  );
}

async function getPost(slug: string | undefined) {
  return {
    title: 'My Post',
    content: '<p>Post content</p>',
  };
}
```

### Nested Routes

```typescript
// app/routes/dashboard.tsx (parent layout)
import { Outlet } from '@remix-run/react';

export default function Dashboard() {
  return (
    <div>
      <nav>
        <a href="/dashboard">Overview</a>
        <a href="/dashboard/settings">Settings</a>
        <a href="/dashboard/profile">Profile</a>
      </nav>
      <main>
        <Outlet /> {/* Child routes render here */}
      </main>
    </div>
  );
}

// app/routes/dashboard._index.tsx
export default function DashboardIndex() {
  return <h1>Dashboard Overview</h1>;
}

// app/routes/dashboard.settings.tsx
export default function Settings() {
  return <h1>Settings</h1>;
}

// app/routes/dashboard.profile.tsx
export default function Profile() {
  return <h1>Profile</h1>;
}
```

### Pathless Routes

```typescript
// app/routes/__auth.tsx (pathless layout, note the __)
import { Outlet } from '@remix-run/react';

export default function AuthLayout() {
  return (
    <div className="auth-container">
      <div className="auth-card">
        <Outlet />
      </div>
    </div>
  );
}

// app/routes/__auth.login.tsx
export default function Login() {
  return <h1>Login</h1>;
}

// app/routes/__auth.register.tsx
export default function Register() {
  return <h1>Register</h1>;
}
```

## Loaders and Actions

Loaders fetch data on GET requests. Actions handle mutations (POST, PUT, DELETE).

### Basic Loader

```typescript
// app/routes/users.tsx
import { json, LoaderFunctionArgs } from '@remix-run/node';
import { useLoaderData } from '@remix-run/react';

export async function loader({ request }: LoaderFunctionArgs) {
  const url = new URL(request.url);
  const search = url.searchParams.get('search') || '';
  
  const users = await fetchUsers(search);
  
  return json({ users, search });
}

export default function Users() {
  const { users, search } = useLoaderData<typeof loader>();
  
  return (
    <div>
      <h1>Users</h1>
      <form method="get">
        <input name="search" defaultValue={search} />
        <button type="submit">Search</button>
      </form>
      <ul>
        {users.map(user => (
          <li key={user.id}>{user.name}</li>
        ))}
      </ul>
    </div>
  );
}

async function fetchUsers(search: string) {
  return [
    { id: 1, name: 'Alice' },
    { id: 2, name: 'Bob' },
  ].filter(u => u.name.toLowerCase().includes(search.toLowerCase()));
}
```

### Loader with Authentication

```typescript
// app/routes/dashboard.tsx
import { json, LoaderFunctionArgs, redirect } from '@remix-run/node';
import { useLoaderData } from '@remix-run/react';
import { getSession } from '~/session.server';

export async function loader({ request }: LoaderFunctionArgs) {
  const session = await getSession(request.headers.get('Cookie'));
  const userId = session.get('userId');
  
  if (!userId) {
    throw redirect('/login');
  }
  
  const user = await getUserById(userId);
  const stats = await getUserStats(userId);
  
  return json({ user, stats });
}

export default function Dashboard() {
  const { user, stats } = useLoaderData<typeof loader>();
  
  return (
    <div>
      <h1>Welcome, {user.name}</h1>
      <p>Total Posts: {stats.posts}</p>
      <p>Total Comments: {stats.comments}</p>
    </div>
  );
}

async function getUserById(id: string) {
  return { id, name: 'John Doe', email: 'john@example.com' };
}

async function getUserStats(id: string) {
  return { posts: 10, comments: 25 };
}
```

### Basic Action

```typescript
// app/routes/posts.new.tsx
import { json, ActionFunctionArgs, redirect } from '@remix-run/node';
import { useActionData, Form } from '@remix-run/react';

export async function action({ request }: ActionFunctionArgs) {
  const formData = await request.formData();
  const title = formData.get('title');
  const content = formData.get('content');
  
  // Validation
  const errors: Record<string, string> = {};
  if (!title) errors.title = 'Title is required';
  if (!content) errors.content = 'Content is required';
  
  if (Object.keys(errors).length > 0) {
    return json({ errors }, { status: 400 });
  }
  
  // Create post
  const post = await createPost({ title: String(title), content: String(content) });
  
  return redirect(`/posts/${post.slug}`);
}

export default function NewPost() {
  const actionData = useActionData<typeof action>();
  
  return (
    <Form method="post">
      <div>
        <label htmlFor="title">Title</label>
        <input type="text" id="title" name="title" />
        {actionData?.errors?.title && (
          <p className="error">{actionData.errors.title}</p>
        )}
      </div>
      
      <div>
        <label htmlFor="content">Content</label>
        <textarea id="content" name="content" />
        {actionData?.errors?.content && (
          <p className="error">{actionData.errors.content}</p>
        )}
      </div>
      
      <button type="submit">Create Post</button>
    </Form>
  );
}

async function createPost(data: { title: string; content: string }) {
  return { id: '1', slug: 'my-post', ...data };
}
```

### Action with Multiple Intents

```typescript
// app/routes/posts.$id.tsx
import { json, ActionFunctionArgs, redirect } from '@remix-run/node';
import { useLoaderData, Form } from '@remix-run/react';

export async function loader({ params }: LoaderFunctionArgs) {
  const post = await getPost(params.id);
  return json({ post });
}

export async function action({ request, params }: ActionFunctionArgs) {
  const formData = await request.formData();
  const intent = formData.get('intent');
  
  switch (intent) {
    case 'delete':
      await deletePost(params.id);
      return redirect('/posts');
      
    case 'publish':
      await publishPost(params.id);
      return json({ success: true });
      
    case 'archive':
      await archivePost(params.id);
      return json({ success: true });
      
    default:
      return json({ error: 'Unknown intent' }, { status: 400 });
  }
}

export default function Post() {
  const { post } = useLoaderData<typeof loader>();
  
  return (
    <div>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
      
      <div className="actions">
        <Form method="post">
          <button name="intent" value="publish">Publish</button>
        </Form>
        
        <Form method="post">
          <button name="intent" value="archive">Archive</button>
        </Form>
        
        <Form method="post" onSubmit={(e) => {
          if (!confirm('Are you sure?')) e.preventDefault();
        }}>
          <button name="intent" value="delete">Delete</button>
        </Form>
      </div>
    </div>
  );
}

async function getPost(id: string | undefined) {
  return { id, title: 'My Post', content: 'Content here' };
}

async function deletePost(id: string | undefined) {}
async function publishPost(id: string | undefined) {}
async function archivePost(id: string | undefined) {}
```

## Data Hooks

### useLoaderData

```typescript
// app/routes/products.$id.tsx
import { json, LoaderFunctionArgs } from '@remix-run/node';
import { useLoaderData } from '@remix-run/react';

export async function loader({ params }: LoaderFunctionArgs) {
  const product = await fetchProduct(params.id);
  const reviews = await fetchReviews(params.id);
  
  return json({ product, reviews });
}

export default function Product() {
  const { product, reviews } = useLoaderData<typeof loader>();
  
  return (
    <div>
      <h1>{product.name}</h1>
      <p>${product.price}</p>
      
      <h2>Reviews ({reviews.length})</h2>
      {reviews.map(review => (
        <div key={review.id}>
          <p>{review.rating} stars</p>
          <p>{review.comment}</p>
        </div>
      ))}
    </div>
  );
}

async function fetchProduct(id: string | undefined) {
  return { id, name: 'Product', price: 99.99 };
}

async function fetchReviews(id: string | undefined) {
  return [
    { id: 1, rating: 5, comment: 'Great!' },
    { id: 2, rating: 4, comment: 'Good' },
  ];
}
```

### useActionData

```typescript
// app/routes/contact.tsx
import { json, ActionFunctionArgs } from '@remix-run/node';
import { useActionData, Form } from '@remix-run/react';

export async function action({ request }: ActionFunctionArgs) {
  const formData = await request.formData();
  const email = formData.get('email');
  const message = formData.get('message');
  
  try {
    await sendEmail({ email: String(email), message: String(message) });
    return json({ success: true, message: 'Message sent!' });
  } catch (error) {
    return json({ success: false, error: 'Failed to send message' }, { status: 500 });
  }
}

export default function Contact() {
  const actionData = useActionData<typeof action>();
  
  return (
    <div>
      <h1>Contact Us</h1>
      
      {actionData?.success && (
        <p className="success">{actionData.message}</p>
      )}
      
      {actionData?.error && (
        <p className="error">{actionData.error}</p>
      )}
      
      <Form method="post">
        <input type="email" name="email" required />
        <textarea name="message" required />
        <button type="submit">Send</button>
      </Form>
    </div>
  );
}

async function sendEmail(data: { email: string; message: string }) {
  console.log('Sending email:', data);
}
```

### useFetcher

```typescript
// app/routes/todos.tsx
import { json, ActionFunctionArgs } from '@remix-run/node';
import { useLoaderData, useFetcher } from '@remix-run/react';

export async function loader() {
  const todos = await getTodos();
  return json({ todos });
}

export async function action({ request }: ActionFunctionArgs) {
  const formData = await request.formData();
  const intent = formData.get('intent');
  
  if (intent === 'toggle') {
    const id = formData.get('id');
    await toggleTodo(String(id));
    return json({ success: true });
  }
  
  if (intent === 'create') {
    const title = formData.get('title');
    const todo = await createTodo(String(title));
    return json({ todo });
  }
  
  return json({ error: 'Invalid intent' }, { status: 400 });
}

export default function Todos() {
  const { todos } = useLoaderData<typeof loader>();
  const fetcher = useFetcher();
  
  return (
    <div>
      <h1>Todos</h1>
      
      {/* Add todo - doesn't cause navigation */}
      <fetcher.Form method="post">
        <input type="text" name="title" placeholder="New todo..." />
        <button name="intent" value="create">Add</button>
      </fetcher.Form>
      
      {/* Todo list */}
      {todos.map(todo => (
        <TodoItem key={todo.id} todo={todo} />
      ))}
    </div>
  );
}

function TodoItem({ todo }: { todo: any }) {
  const fetcher = useFetcher();
  
  // Optimistic UI
  const isToggling = fetcher.formData?.get('id') === todo.id;
  const completed = isToggling 
    ? !todo.completed 
    : todo.completed;
  
  return (
    <fetcher.Form method="post" style={{ opacity: isToggling ? 0.5 : 1 }}>
      <input type="hidden" name="id" value={todo.id} />
      <input
        type="checkbox"
        checked={completed}
        onChange={(e) => {
          fetcher.submit(e.currentTarget.form);
        }}
      />
      <button name="intent" value="toggle" style={{ display: 'none' }} />
      <span style={{ textDecoration: completed ? 'line-through' : 'none' }}>
        {todo.title}
      </span>
    </fetcher.Form>
  );
}

async function getTodos() {
  return [
    { id: '1', title: 'Learn Remix', completed: true },
    { id: '2', title: 'Build app', completed: false },
  ];
}

async function toggleTodo(id: string) {}
async function createTodo(title: string) {
  return { id: Date.now().toString(), title, completed: false };
}
```

### useNavigation

```typescript
// app/root.tsx
import { Outlet, useNavigation } from '@remix-run/react';

export default function App() {
  const navigation = useNavigation();
  
  const isNavigating = navigation.state === 'loading';
  const isSubmitting = navigation.state === 'submitting';
  
  return (
    <html>
      <body>
        {isNavigating && <div className="loading-bar">Loading...</div>}
        {isSubmitting && <div className="loading-bar">Saving...</div>}
        
        <Outlet />
      </body>
    </html>
  );
}
```

## Form and Progressive Enhancement

### Basic Form

```typescript
// app/routes/newsletter.tsx
import { json, ActionFunctionArgs } from '@remix-run/node';
import { Form, useActionData } from '@remix-run/react';

export async function action({ request }: ActionFunctionArgs) {
  const formData = await request.formData();
  const email = formData.get('email');
  
  if (!email || !String(email).includes('@')) {
    return json({ error: 'Invalid email' }, { status: 400 });
  }
  
  await subscribeToNewsletter(String(email));
  
  return json({ success: true });
}

export default function Newsletter() {
  const actionData = useActionData<typeof action>();
  
  return (
    <div>
      <h2>Subscribe to Newsletter</h2>
      
      {/* Works without JavaScript! */}
      <Form method="post">
        <input type="email" name="email" required />
        <button type="submit">Subscribe</button>
      </Form>
      
      {actionData?.success && <p>Subscribed!</p>}
      {actionData?.error && <p className="error">{actionData.error}</p>}
    </div>
  );
}

async function subscribeToNewsletter(email: string) {
  console.log('Subscribed:', email);
}
```

### Form with Replace

```typescript
// app/routes/search.tsx
import { json, LoaderFunctionArgs } from '@remix-run/node';
import { useLoaderData, Form } from '@remix-run/react';

export async function loader({ request }: LoaderFunctionArgs) {
  const url = new URL(request.url);
  const q = url.searchParams.get('q') || '';
  
  const results = await search(q);
  
  return json({ results, q });
}

export default function Search() {
  const { results, q } = useLoaderData<typeof loader>();
  
  return (
    <div>
      <h1>Search</h1>
      
      {/* replace=true doesn't push to history stack */}
      <Form method="get" replace>
        <input type="search" name="q" defaultValue={q} />
        <button type="submit">Search</button>
      </Form>
      
      <ul>
        {results.map(result => (
          <li key={result.id}>{result.title}</li>
        ))}
      </ul>
    </div>
  );
}

async function search(query: string) {
  return [
    { id: 1, title: 'Result 1' },
    { id: 2, title: 'Result 2' },
  ];
}
```

### Auto-Submit Form

```typescript
// app/routes/products.tsx
import { json, LoaderFunctionArgs } from '@remix-run/node';
import { useLoaderData, useSubmit } from '@remix-run/react';

export async function loader({ request }: LoaderFunctionArgs) {
  const url = new URL(request.url);
  const category = url.searchParams.get('category') || 'all';
  const sortBy = url.searchParams.get('sortBy') || 'name';
  
  const products = await fetchProducts({ category, sortBy });
  
  return json({ products, category, sortBy });
}

export default function Products() {
  const { products, category, sortBy } = useLoaderData<typeof loader>();
  const submit = useSubmit();
  
  return (
    <div>
      <h1>Products</h1>
      
      <form
        onChange={(e) => {
          // Auto-submit on change
          submit(e.currentTarget, { replace: true });
        }}
      >
        <select name="category" defaultValue={category}>
          <option value="all">All</option>
          <option value="electronics">Electronics</option>
          <option value="clothing">Clothing</option>
        </select>
        
        <select name="sortBy" defaultValue={sortBy}>
          <option value="name">Name</option>
          <option value="price">Price</option>
          <option value="rating">Rating</option>
        </select>
      </form>
      
      <ul>
        {products.map(product => (
          <li key={product.id}>
            {product.name} - ${product.price}
          </li>
        ))}
      </ul>
    </div>
  );
}

async function fetchProducts(filters: { category: string; sortBy: string }) {
  return [
    { id: 1, name: 'Product A', price: 29.99 },
    { id: 2, name: 'Product B', price: 49.99 },
  ];
}
```

## Deferred Data and Streaming

### Basic Defer

```typescript
// app/routes/dashboard.tsx
import { defer, LoaderFunctionArgs } from '@remix-run/node';
import { Await, useLoaderData } from '@remix-run/react';
import { Suspense } from 'react';

export async function loader({ request }: LoaderFunctionArgs) {
  // Fast data - await immediately
  const user = await getUser(request);
  
  // Slow data - don't await, stream later
  const analyticsPromise = getAnalytics();
  const recentActivityPromise = getRecentActivity();
  
  return defer({
    user,
    analytics: analyticsPromise,
    recentActivity: recentActivityPromise,
  });
}

export default function Dashboard() {
  const data = useLoaderData<typeof loader>();
  
  return (
    <div>
      {/* Render immediately */}
      <h1>Welcome, {data.user.name}</h1>
      
      {/* Stream in when ready */}
      <Suspense fallback={<div>Loading analytics...</div>}>
        <Await resolve={data.analytics}>
          {(analytics) => (
            <div>
              <h2>Analytics</h2>
              <p>Views: {analytics.views}</p>
              <p>Clicks: {analytics.clicks}</p>
            </div>
          )}
        </Await>
      </Suspense>
      
      <Suspense fallback={<div>Loading activity...</div>}>
        <Await resolve={data.recentActivity}>
          {(activity) => (
            <div>
              <h2>Recent Activity</h2>
              <ul>
                {activity.map((item: any) => (
                  <li key={item.id}>{item.description}</li>
                ))}
              </ul>
            </div>
          )}
        </Await>
      </Suspense>
    </div>
  );
}

async function getUser(request: Request) {
  return { id: '1', name: 'John Doe' };
}

async function getAnalytics() {
  await new Promise(resolve => setTimeout(resolve, 2000));
  return { views: 1000, clicks: 500 };
}

async function getRecentActivity() {
  await new Promise(resolve => setTimeout(resolve, 3000));
  return [
    { id: 1, description: 'Logged in' },
    { id: 2, description: 'Updated profile' },
  ];
}
```

### Defer with Error Handling

```typescript
// app/routes/profile.tsx
import { defer, LoaderFunctionArgs } from '@remix-run/node';
import { Await, useLoaderData } from '@remix-run/react';
import { Suspense } from 'react';

export async function loader({ params }: LoaderFunctionArgs) {
  const userPromise = getUser(params.id);
  const postsPromise = getUserPosts(params.id);
  
  return defer({
    user: userPromise,
    posts: postsPromise,
  });
}

export default function Profile() {
  const data = useLoaderData<typeof loader>();
  
  return (
    <div>
      <Suspense fallback={<div>Loading user...</div>}>
        <Await resolve={data.user} errorElement={<div>Failed to load user</div>}>
          {(user) => (
            <div>
              <h1>{user.name}</h1>
              <p>{user.bio}</p>
            </div>
          )}
        </Await>
      </Suspense>
      
      <Suspense fallback={<div>Loading posts...</div>}>
        <Await resolve={data.posts} errorElement={<div>Failed to load posts</div>}>
          {(posts) => (
            <ul>
              {posts.map((post: any) => (
                <li key={post.id}>{post.title}</li>
              ))}
            </ul>
          )}
        </Await>
      </Suspense>
    </div>
  );
}

async function getUser(id: string | undefined) {
  await new Promise(resolve => setTimeout(resolve, 1000));
  return { id, name: 'John Doe', bio: 'Software developer' };
}

async function getUserPosts(id: string | undefined) {
  await new Promise(resolve => setTimeout(resolve, 2000));
  return [
    { id: 1, title: 'First Post' },
    { id: 2, title: 'Second Post' },
  ];
}
```

## Error Boundaries

### Route-Level Error Boundary

```typescript
// app/routes/posts.$id.tsx
import { json, LoaderFunctionArgs } from '@remix-run/node';
import { useLoaderData, useRouteError, isRouteErrorResponse } from '@remix-run/react';

export async function loader({ params }: LoaderFunctionArgs) {
  const post = await getPost(params.id);
  
  if (!post) {
    throw new Response('Not Found', { status: 404 });
  }
  
  return json({ post });
}

export default function Post() {
  const { post } = useLoaderData<typeof loader>();
  
  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </article>
  );
}

export function ErrorBoundary() {
  const error = useRouteError();
  
  if (isRouteErrorResponse(error)) {
    return (
      <div>
        <h1>{error.status} {error.statusText}</h1>
        <p>{error.data}</p>
      </div>
    );
  }
  
  return (
    <div>
      <h1>Error</h1>
      <p>Something went wrong!</p>
    </div>
  );
}

async function getPost(id: string | undefined) {
  if (id === 'missing') {
    return null;
  }
  return { id, title: 'My Post', content: 'Content here' };
}
```

### Nested Error Boundaries

```typescript
// app/routes/dashboard.tsx
import { Outlet } from '@remix-run/react';

export default function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      <nav>
        <a href="/dashboard/settings">Settings</a>
        <a href="/dashboard/billing">Billing</a>
      </nav>
      <main>
        <Outlet />
      </main>
    </div>
  );
}

export function ErrorBoundary() {
  return (
    <div>
      <h1>Dashboard Error</h1>
      <p>Something went wrong with the dashboard.</p>
    </div>
  );
}

// app/routes/dashboard.billing.tsx
export async function loader() {
  throw new Error('Billing service unavailable');
}

export default function Billing() {
  return <div>Billing Info</div>;
}

// This error boundary is more specific
export function ErrorBoundary() {
  return (
    <div>
      <h2>Billing Error</h2>
      <p>Unable to load billing information. Please try again later.</p>
    </div>
  );
}
```

## React Router 7

React Router 7 is the evolution of Remix, where Remix's framework features have been extracted into React Router itself.

### Migration from Remix to React Router 7

```typescript
// Before (Remix)
// app/routes/posts.$id.tsx
import { json, LoaderFunctionArgs } from '@remix-run/node';
import { useLoaderData } from '@remix-run/react';

export async function loader({ params }: LoaderFunctionArgs) {
  const post = await getPost(params.id);
  return json({ post });
}

export default function Post() {
  const { post } = useLoaderData<typeof loader>();
  return <article>{post.title}</article>;
}

// After (React Router 7)
// app/routes/posts.$id.tsx
import { LoaderFunctionArgs } from 'react-router-dom';
import { useLoaderData } from 'react-router-dom';

export async function loader({ params }: LoaderFunctionArgs) {
  const post = await getPost(params.id);
  return { post }; // Can return plain objects now
}

export default function Post() {
  const { post } = useLoaderData<typeof loader>();
  return <article>{post.title}</article>;
}

async function getPost(id: string | undefined) {
  return { id, title: 'Post Title' };
}
```

### Key Differences

```typescript
// React Router 7 simplifications:

// 1. No more json() helper needed
export async function loader() {
  return { data: 'value' }; // Instead of json({ data: 'value' })
}

// 2. Actions return plain objects
export async function action({ request }: ActionFunctionArgs) {
  const formData = await request.formData();
  return { success: true }; // Instead of json({ success: true })
}

// 3. Simplified error handling
export function ErrorBoundary() {
  const error = useRouteError();
  return <div>Error: {error.message}</div>;
}

// 4. TypeScript improvements
// Better type inference from loaders/actions
const data = useLoaderData<typeof loader>(); // Fully typed!
```

## Common Mistakes

### 1. Not Using Form Component

```typescript
// WRONG: Regular form loses progressive enhancement
export default function Contact() {
  return (
    <form method="post" action="/api/contact">
      <input name="email" />
      <button>Submit</button>
    </form>
  );
}

// CORRECT: Remix Form provides progressive enhancement
import { Form } from '@remix-run/react';

export default function Contact() {
  return (
    <Form method="post">
      <input name="email" />
      <button>Submit</button>
    </Form>
  );
}
```

### 2. Over-fetching in Loaders

```typescript
// WRONG: Loading too much data
export async function loader() {
  const allUsers = await getAllUsers(); // 10,000 users!
  const allPosts = await getAllPosts(); // 100,000 posts!
  return json({ allUsers, allPosts });
}

// CORRECT: Load only what's needed
export async function loader({ request }: LoaderFunctionArgs) {
  const url = new URL(request.url);
  const page = parseInt(url.searchParams.get('page') || '1', 10);
  
  const users = await getUsers({ page, limit: 20 });
  return json({ users });
}
```

### 3. Not Using useFetcher for Non-Navigation Actions

```typescript
// WRONG: Using Form causes navigation
import { Form } from '@remix-run/react';

export default function TodoList({ todos }: { todos: any[] }) {
  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          <Form method="post">
            <input type="hidden" name="id" value={todo.id} />
            <button name="intent" value="delete">Delete</button>
          </Form>
        </li>
      ))}
    </ul>
  );
}

// CORRECT: Use useFetcher to avoid navigation
import { useFetcher } from '@remix-run/react';

export default function TodoList({ todos }: { todos: any[] }) {
  return (
    <ul>
      {todos.map(todo => (
        <TodoItem key={todo.id} todo={todo} />
      ))}
    </ul>
  );
}

function TodoItem({ todo }: { todo: any }) {
  const fetcher = useFetcher();
  
  return (
    <li>
      <fetcher.Form method="post">
        <input type="hidden" name="id" value={todo.id} />
        <button name="intent" value="delete">Delete</button>
      </fetcher.Form>
    </li>
  );
}
```

## Best Practices

### 1. Use Defer for Slow Data

```typescript
export async function loader() {
  // Fast data - await
  const critical = await getCriticalData();
  
  // Slow data - defer
  const slowData = getSlowData();
  
  return defer({
    critical,
    slow: slowData,
  });
}
```

### 2. Leverage Progressive Enhancement

```typescript
// Works without JavaScript
import { Form } from '@remix-run/react';

export default function Subscribe() {
  return (
    <Form method="post">
      <input type="email" name="email" required />
      <button>Subscribe</button>
    </Form>
  );
}
```

### 3. Use useFetcher for Concurrent Mutations

```typescript
function LikeButton({ postId }: { postId: string }) {
  const fetcher = useFetcher();
  
  // Optimistic UI
  const liked = fetcher.formData 
    ? fetcher.formData.get('liked') === 'true'
    : false;
  
  return (
    <fetcher.Form method="post" action="/api/like">
      <input type="hidden" name="postId" value={postId} />
      <input type="hidden" name="liked" value={(!liked).toString()} />
      <button type="submit">
        {liked ? 'Unlike' : 'Like'}
      </button>
    </fetcher.Form>
  );
}
```

## Interview Questions

### Q1: What's the difference between loader and action?

**Answer:** Loaders run on GET requests to fetch data before rendering. Actions run on POST/PUT/PATCH/DELETE to handle mutations. Loaders return data to render, actions typically return redirect or validation errors.

### Q2: When should you use useFetcher vs Form?

**Answer:** Use Form when the action should navigate (e.g., creating a post navigates to the post page). Use useFetcher when you want to perform actions without navigation (e.g., liking a post, marking a todo complete).

### Q3: How does defer work?

**Answer:** defer allows you to return promises from loaders without awaiting them. The page renders with the resolved data immediately, and deferred promises stream in and render when ready using Suspense boundaries.

### Q4: What's the difference between Remix and React Router 7?

**Answer:** React Router 7 is the evolution of Remix, where Remix's framework features (loaders, actions, defer, etc.) have been extracted into React Router. Remix will continue as a layer on top of React Router with additional features like server runtime adapters.

### Q5: How do nested error boundaries work?

**Answer:** Error boundaries catch errors in their route segment and children. Child route error boundaries catch errors first. If a child doesn't have an error boundary, the error bubbles up to the parent's boundary.

## Key Takeaways

1. File-based routing uses file naming conventions in app/routes
2. Loaders fetch data on GET, actions handle mutations
3. useLoaderData, useActionData, useFetcher are core data hooks
4. Form component provides progressive enhancement
5. defer enables streaming slow data while rendering fast data
6. useFetcher enables concurrent mutations without navigation
7. Error boundaries provide granular error handling per route
8. React Router 7 is the evolution of Remix framework
9. Progressive enhancement means apps work without JavaScript
10. Optimistic UI improves perceived performance

## Resources

- [Remix Documentation](https://remix.run/docs)
- [React Router Documentation](https://reactrouter.com)
- [Remix vs Next.js](https://remix.run/blog/remix-vs-next)
- [Progressive Enhancement](https://remix.run/docs/en/main/discussion/progressive-enhancement)
- [React Router 7 Announcement](https://remix.run/blog/react-router-v7)
