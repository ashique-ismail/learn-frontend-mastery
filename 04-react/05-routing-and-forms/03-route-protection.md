# Route Protection and Authentication Guards

## The Idea

**In plain English:** Route protection is a way to lock certain pages of a website so only the right people can see them — like a dashboard that only shows up after you've logged in. If you try to visit a locked page without permission, the app automatically sends you somewhere else (usually a login page).

**Real-world analogy:** Think of a movie theater where some screening rooms are for ticket-holders only. A staff member stands at the door and checks your ticket before you can enter.

- The staff member at the door = the route loader (the code that runs before the page loads)
- Your movie ticket = the user's login session or authentication token
- Being turned away and directed to the ticket booth = the redirect to the login page
- The screening room itself = the protected page/route

---

## Table of Contents

1. [Introduction](#introduction)
2. [Basic Route Protection](#basic-route-protection)
3. [Loader-Based Protection](#loader-based-protection)
4. [Role-Based Access Control](#role-based-access-control)
5. [Redirect Patterns](#redirect-patterns)
6. [Protected Layout Routes](#protected-layout-routes)
7. [Session Management](#session-management)
8. [Advanced Patterns](#advanced-patterns)
9. [Common Mistakes](#common-mistakes)
10. [Best Practices](#best-practices)
11. [Interview Questions](#interview-questions)
12. [Resources](#resources)

## Introduction

Route protection ensures that users can only access pages they're authorized to view. With React Router's data APIs, authentication checks happen in loaders before rendering, preventing flash of unauthorized content and improving security.

## Basic Route Protection

### Component-Based Protection (Legacy Pattern)

```typescript
import { Navigate, useLocation } from 'react-router-dom';

interface ProtectedRouteProps {
  children: React.ReactNode;
}

// ❌ Not recommended - causes flash of content
function ProtectedRoute({ children }: ProtectedRouteProps) {
  const { user, loading } = useAuth();
  const location = useLocation();
  
  if (loading) {
    return <LoadingSpinner />;
  }
  
  if (!user) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }
  
  return <>{children}</>;
}

// Usage
<Route
  path="/dashboard"
  element={
    <ProtectedRoute>
      <Dashboard />
    </ProtectedRoute>
  }
/>
```

### Modern Loader-Based Protection

```typescript
import { redirect, LoaderFunctionArgs } from 'react-router-dom';

// ✅ Recommended - checks before rendering
async function protectedLoader({ request }: LoaderFunctionArgs) {
  const user = await getCurrentUser();
  
  if (!user) {
    const url = new URL(request.url);
    const params = new URLSearchParams();
    params.set('from', url.pathname);
    return redirect(`/login?${params.toString()}`);
  }
  
  return user;
}

const router = createBrowserRouter([
  {
    path: '/dashboard',
    element: <Dashboard />,
    loader: protectedLoader
  }
]);
```

### Auth Service

```typescript
// services/auth.ts
interface User {
  id: string;
  email: string;
  name: string;
  roles: string[];
}

class AuthService {
  private user: User | null = null;
  
  async getCurrentUser(): Promise<User | null> {
    if (this.user) return this.user;
    
    try {
      const response = await fetch('/api/auth/me', {
        credentials: 'include'
      });
      
      if (!response.ok) return null;
      
      this.user = await response.json();
      return this.user;
    } catch {
      return null;
    }
  }
  
  async login(email: string, password: string): Promise<User> {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password }),
      credentials: 'include'
    });
    
    if (!response.ok) {
      throw new Error('Login failed');
    }
    
    this.user = await response.json();
    return this.user;
  }
  
  async logout(): Promise<void> {
    await fetch('/api/auth/logout', {
      method: 'POST',
      credentials: 'include'
    });
    this.user = null;
  }
  
  clearCache() {
    this.user = null;
  }
}

export const authService = new AuthService();
```

## Loader-Based Protection

### Simple Auth Check

```typescript
async function requireAuth({ request }: LoaderFunctionArgs) {
  const user = await authService.getCurrentUser();
  
  if (!user) {
    throw redirect('/login');
  }
  
  return user;
}

// Apply to routes
const router = createBrowserRouter([
  {
    path: '/dashboard',
    element: <Dashboard />,
    loader: requireAuth
  },
  {
    path: '/profile',
    element: <Profile />,
    loader: requireAuth
  }
]);
```

### Auth Check with Redirect Destination

```typescript
function requireAuth({ request }: LoaderFunctionArgs) {
  const user = await authService.getCurrentUser();
  
  if (!user) {
    const url = new URL(request.url);
    const searchParams = new URLSearchParams();
    searchParams.set('redirectTo', url.pathname + url.search);
    
    throw redirect(`/login?${searchParams.toString()}`);
  }
  
  return user;
}

// Login action - redirect after login
async function loginAction({ request }: ActionFunctionArgs) {
  const formData = await request.formData();
  const email = formData.get('email') as string;
  const password = formData.get('password') as string;
  
  const user = await authService.login(email, password);
  
  // Get redirect destination
  const url = new URL(request.url);
  const redirectTo = url.searchParams.get('redirectTo') || '/dashboard';
  
  return redirect(redirectTo);
}
```

### Combining Auth Check with Data Loading

```typescript
async function dashboardLoader({ request }: LoaderFunctionArgs) {
  // Check auth first
  const user = await requireAuth({ request } as LoaderFunctionArgs);
  
  // If we get here, user is authenticated
  const [stats, recentActivity] = await Promise.all([
    fetch('/api/dashboard/stats').then(r => r.json()),
    fetch('/api/dashboard/activity').then(r => r.json())
  ]);
  
  return { user, stats, recentActivity };
}

function Dashboard() {
  const { user, stats, recentActivity } = useLoaderData() as DashboardData;
  
  return (
    <div>
      <h1>Welcome, {user.name}</h1>
      <Stats data={stats} />
      <ActivityFeed items={recentActivity} />
    </div>
  );
}
```

## Role-Based Access Control

### Basic RBAC

```typescript
type Role = 'user' | 'admin' | 'moderator';

interface User {
  id: string;
  roles: Role[];
}

function requireRole(allowedRoles: Role[]) {
  return async ({ request }: LoaderFunctionArgs) => {
    const user = await authService.getCurrentUser();
    
    if (!user) {
      throw redirect('/login');
    }
    
    const hasRequiredRole = allowedRoles.some(role => 
      user.roles.includes(role)
    );
    
    if (!hasRequiredRole) {
      throw new Response('Forbidden', { status: 403 });
    }
    
    return user;
  };
}

// Usage
const router = createBrowserRouter([
  {
    path: '/admin',
    element: <AdminPanel />,
    loader: requireRole(['admin']),
    errorElement: <ForbiddenPage />
  },
  {
    path: '/moderate',
    element: <ModerationPanel />,
    loader: requireRole(['admin', 'moderator'])
  }
]);
```

### Permission-Based Access Control

```typescript
type Permission = 
  | 'users:read'
  | 'users:write'
  | 'posts:read'
  | 'posts:write'
  | 'posts:delete';

interface User {
  id: string;
  permissions: Permission[];
}

function requirePermission(requiredPermissions: Permission[]) {
  return async ({ request }: LoaderFunctionArgs) => {
    const user = await authService.getCurrentUser();
    
    if (!user) {
      throw redirect('/login');
    }
    
    const hasAllPermissions = requiredPermissions.every(permission =>
      user.permissions.includes(permission)
    );
    
    if (!hasAllPermissions) {
      throw json(
        { message: 'Insufficient permissions' },
        { status: 403 }
      );
    }
    
    return user;
  };
}

// Usage
const router = createBrowserRouter([
  {
    path: '/users/:id/edit',
    element: <EditUser />,
    loader: requirePermission(['users:write'])
  },
  {
    path: '/posts/:id/delete',
    action: requirePermission(['posts:delete'])
  }
]);
```

### Resource-Based Authorization

```typescript
interface Post {
  id: string;
  authorId: string;
  title: string;
}

async function postLoader({ params, request }: LoaderFunctionArgs) {
  const user = await requireAuth({ request } as LoaderFunctionArgs);
  
  const post = await fetch(`/api/posts/${params.id}`)
    .then(r => r.json()) as Post;
  
  // Check if user can edit this specific post
  const canEdit = 
    post.authorId === user.id || 
    user.roles.includes('admin');
  
  return { post, canEdit };
}

function EditPost() {
  const { post, canEdit } = useLoaderData() as { 
    post: Post; 
    canEdit: boolean 
  };
  
  if (!canEdit) {
    return (
      <div>
        <h1>Access Denied</h1>
        <p>You don't have permission to edit this post.</p>
      </div>
    );
  }
  
  return <PostEditor post={post} />;
}
```

### Composable Authorization

```typescript
// Authorization helpers
type AuthCheck = (user: User, context?: any) => boolean;

const authChecks = {
  isAuthenticated: (user: User) => !!user,
  
  hasRole: (role: Role) => (user: User) => 
    user.roles.includes(role),
  
  hasPermission: (permission: Permission) => (user: User) =>
    user.permissions.includes(permission),
  
  isOwner: (resourceOwnerId: string) => (user: User) =>
    user.id === resourceOwnerId,
  
  or: (...checks: AuthCheck[]) => (user: User, context?: any) =>
    checks.some(check => check(user, context)),
  
  and: (...checks: AuthCheck[]) => (user: User, context?: any) =>
    checks.every(check => check(user, context))
};

// Create reusable authorization loader
function authorize(check: AuthCheck, options: { redirectTo?: string } = {}) {
  return async ({ request }: LoaderFunctionArgs) => {
    const user = await authService.getCurrentUser();
    
    if (!user) {
      throw redirect(options.redirectTo || '/login');
    }
    
    if (!check(user)) {
      throw new Response('Forbidden', { status: 403 });
    }
    
    return user;
  };
}

// Usage - complex authorization rules
const canEditPost = authChecks.or(
  authChecks.hasRole('admin'),
  authChecks.and(
    authChecks.hasPermission('posts:write'),
    authChecks.isOwner('post-author-id')
  )
);

const router = createBrowserRouter([
  {
    path: '/posts/:id/edit',
    element: <EditPost />,
    loader: authorize(canEditPost)
  }
]);
```

## Redirect Patterns

### Login Redirect

```typescript
// Login page
function Login() {
  const [searchParams] = useSearchParams();
  const redirectTo = searchParams.get('redirectTo') || '/dashboard';
  
  return (
    <Form method="post" action={`/login?redirectTo=${redirectTo}`}>
      <input type="email" name="email" required />
      <input type="password" name="password" required />
      <button type="submit">Login</button>
    </Form>
  );
}

// Login action
async function loginAction({ request }: ActionFunctionArgs) {
  const formData = await request.formData();
  const email = formData.get('email') as string;
  const password = formData.get('password') as string;
  
  try {
    await authService.login(email, password);
    
    const url = new URL(request.url);
    const redirectTo = url.searchParams.get('redirectTo') || '/dashboard';
    
    return redirect(redirectTo);
  } catch (error) {
    return { error: 'Invalid credentials' };
  }
}
```

### Logout Redirect

```typescript
async function logoutAction({ request }: ActionFunctionArgs) {
  await authService.logout();
  
  const url = new URL(request.url);
  const redirectTo = url.searchParams.get('redirectTo') || '/';
  
  return redirect(redirectTo);
}

function Header() {
  const submit = useSubmit();
  
  const handleLogout = () => {
    submit(null, { 
      method: 'post', 
      action: '/logout?redirectTo=/' 
    });
  };
  
  return (
    <header>
      <button onClick={handleLogout}>Logout</button>
    </header>
  );
}
```

### Conditional Redirects

```typescript
async function rootLoader({ request }: LoaderFunctionArgs) {
  const user = await authService.getCurrentUser();
  const url = new URL(request.url);
  
  // Redirect authenticated users away from public pages
  if (user && url.pathname === '/') {
    return redirect('/dashboard');
  }
  
  // Redirect unauthenticated users to login
  if (!user && url.pathname.startsWith('/app')) {
    return redirect('/login');
  }
  
  return { user };
}
```

## Protected Layout Routes

### Auth Layout Pattern

```typescript
// Layout that protects all child routes
async function authLayoutLoader({ request }: LoaderFunctionArgs) {
  const user = await authService.getCurrentUser();
  
  if (!user) {
    const url = new URL(request.url);
    throw redirect(`/login?redirectTo=${url.pathname}`);
  }
  
  // Load user-specific data for layout
  const notifications = await fetch('/api/notifications')
    .then(r => r.json());
  
  return { user, notifications };
}

function AuthLayout() {
  const { user, notifications } = useLoaderData() as AuthLayoutData;
  
  return (
    <div className="app-layout">
      <Header user={user} notifications={notifications} />
      <Sidebar />
      <main>
        <Outlet />
      </main>
    </div>
  );
}

// Route configuration
const router = createBrowserRouter([
  {
    path: '/',
    element: <Root />,
    children: [
      {
        path: 'app',
        element: <AuthLayout />,
        loader: authLayoutLoader,
        children: [
          {
            index: true,
            element: <Dashboard />
          },
          {
            path: 'profile',
            element: <Profile />
          },
          {
            path: 'settings',
            element: <Settings />
          }
        ]
      }
    ]
  }
]);
```

### Multiple Auth Levels

```typescript
const router = createBrowserRouter([
  {
    path: '/',
    element: <Root />,
    children: [
      // Public routes
      {
        path: 'login',
        element: <Login />,
        action: loginAction
      },
      
      // Authenticated routes
      {
        element: <AuthLayout />,
        loader: requireAuth,
        children: [
          {
            path: 'dashboard',
            element: <Dashboard />
          },
          {
            path: 'profile',
            element: <Profile />
          },
          
          // Admin-only routes (nested protection)
          {
            element: <AdminLayout />,
            loader: requireRole(['admin']),
            children: [
              {
                path: 'admin/users',
                element: <UserManagement />
              },
              {
                path: 'admin/settings',
                element: <AdminSettings />
              }
            ]
          }
        ]
      }
    ]
  }
]);
```

## Session Management

### Session Validation

```typescript
interface Session {
  userId: string;
  expiresAt: number;
}

async function validateSession(): Promise<User | null> {
  try {
    const response = await fetch('/api/auth/session', {
      credentials: 'include'
    });
    
    if (!response.ok) {
      return null;
    }
    
    const session: Session = await response.json();
    
    // Check expiration
    if (session.expiresAt < Date.now()) {
      await authService.logout();
      return null;
    }
    
    // Extend session
    await fetch('/api/auth/session/extend', {
      method: 'POST',
      credentials: 'include'
    });
    
    return authService.getCurrentUser();
  } catch {
    return null;
  }
}

// Use in protected loader
async function protectedLoader({ request }: LoaderFunctionArgs) {
  const user = await validateSession();
  
  if (!user) {
    throw redirect('/login?session=expired');
  }
  
  return user;
}
```

### Refresh Token Pattern

```typescript
class AuthService {
  private accessToken: string | null = null;
  private refreshToken: string | null = null;
  
  async fetchWithAuth(url: string, options: RequestInit = {}) {
    // Add access token
    const headers = new Headers(options.headers);
    if (this.accessToken) {
      headers.set('Authorization', `Bearer ${this.accessToken}`);
    }
    
    let response = await fetch(url, { ...options, headers });
    
    // If unauthorized, try refreshing token
    if (response.status === 401 && this.refreshToken) {
      const refreshed = await this.refreshAccessToken();
      
      if (refreshed) {
        // Retry with new token
        headers.set('Authorization', `Bearer ${this.accessToken}`);
        response = await fetch(url, { ...options, headers });
      } else {
        // Refresh failed - logout
        await this.logout();
        throw new Error('Session expired');
      }
    }
    
    return response;
  }
  
  private async refreshAccessToken(): Promise<boolean> {
    try {
      const response = await fetch('/api/auth/refresh', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ refreshToken: this.refreshToken })
      });
      
      if (!response.ok) return false;
      
      const data = await response.json();
      this.accessToken = data.accessToken;
      this.refreshToken = data.refreshToken;
      
      return true;
    } catch {
      return false;
    }
  }
}
```

### Inactivity Timeout

```typescript
function useInactivityTimeout(timeoutMs: number = 30 * 60 * 1000) {
  const navigate = useNavigate();
  const submit = useSubmit();
  
  useEffect(() => {
    let timeout: NodeJS.Timeout;
    
    const resetTimer = () => {
      clearTimeout(timeout);
      timeout = setTimeout(() => {
        submit(null, { method: 'post', action: '/logout' });
        navigate('/login?reason=inactivity');
      }, timeoutMs);
    };
    
    // Reset on user activity
    const events = ['mousedown', 'keydown', 'scroll', 'touchstart'];
    events.forEach(event => {
      document.addEventListener(event, resetTimer);
    });
    
    resetTimer();
    
    return () => {
      clearTimeout(timeout);
      events.forEach(event => {
        document.removeEventListener(event, resetTimer);
      });
    };
  }, [timeoutMs, navigate, submit]);
}

function AuthLayout() {
  useInactivityTimeout(30 * 60 * 1000); // 30 minutes
  
  return (
    <div>
      <Outlet />
    </div>
  );
}
```

## Advanced Patterns

### Route Guards with Middleware Pattern

```typescript
type RouteMiddleware = (
  args: LoaderFunctionArgs
) => Promise<Response | User | null>;

function composeMiddleware(...middlewares: RouteMiddleware[]) {
  return async (args: LoaderFunctionArgs) => {
    for (const middleware of middlewares) {
      const result = await middleware(args);
      
      // If middleware returns Response, stop and return it
      if (result instanceof Response) {
        return result;
      }
    }
    
    // All middleware passed
    return null;
  };
}

// Middleware functions
const authMiddleware: RouteMiddleware = async ({ request }) => {
  const user = await authService.getCurrentUser();
  if (!user) {
    return redirect('/login');
  }
  return user;
};

const roleMiddleware = (role: Role): RouteMiddleware => {
  return async ({ request }) => {
    const user = await authService.getCurrentUser();
    if (!user || !user.roles.includes(role)) {
      throw new Response('Forbidden', { status: 403 });
    }
    return user;
  };
};

// Compose middleware
const adminLoader = composeMiddleware(
  authMiddleware,
  roleMiddleware('admin')
);

// Use in route
const router = createBrowserRouter([
  {
    path: '/admin',
    element: <AdminPanel />,
    loader: async (args) => {
      await adminLoader(args);
      // Load admin data
      return fetchAdminData();
    }
  }
]);
```

### Conditional Route Registration

```typescript
function createRoutes(user: User | null): RouteObject[] {
  const baseRoutes: RouteObject[] = [
    {
      path: '/',
      element: <Home />
    },
    {
      path: '/about',
      element: <About />
    }
  ];
  
  if (user) {
    baseRoutes.push({
      path: '/dashboard',
      element: <Dashboard />,
      loader: dashboardLoader
    });
  }
  
  if (user?.roles.includes('admin')) {
    baseRoutes.push({
      path: '/admin',
      element: <AdminPanel />,
      loader: adminLoader
    });
  }
  
  return baseRoutes;
}

// Dynamic router creation
function App() {
  const [user, setUser] = useState<User | null>(null);
  const [router, setRouter] = useState<ReturnType<typeof createBrowserRouter>>();
  
  useEffect(() => {
    authService.getCurrentUser().then(user => {
      setUser(user);
      setRouter(createBrowserRouter(createRoutes(user)));
    });
  }, []);
  
  if (!router) return <Loading />;
  
  return <RouterProvider router={router} />;
}
```

## Common Mistakes

### 1. Flash of Unauthorized Content

```typescript
// ❌ Bad - component renders before redirect
function ProtectedRoute({ children }) {
  const { user } = useAuth();
  
  if (!user) {
    return <Navigate to="/login" />;
  }
  
  return children; // Children already rendered!
}

// ✅ Good - loader prevents rendering
async function protectedLoader({ request }: LoaderFunctionArgs) {
  const user = await authService.getCurrentUser();
  
  if (!user) {
    throw redirect('/login');
  }
  
  return user;
}
```

### 2. Not Handling Loading States

```typescript
// ❌ Bad - synchronous check
function Dashboard() {
  const user = authService.user; // Might be null during load
  return <div>Welcome {user.name}</div>; // Crashes!
}

// ✅ Good - loader ensures user exists
async function dashboardLoader() {
  return await requireAuth(/* args */);
}

function Dashboard() {
  const user = useLoaderData() as User; // Guaranteed to exist
  return <div>Welcome {user.name}</div>;
}
```

### 3. Not Clearing Auth State on Logout

```typescript
// ❌ Bad - stale data
async function logout() {
  await fetch('/api/logout', { method: 'POST' });
  // authService.user still set!
}

// ✅ Good - clear all auth state
async function logoutAction() {
  await authService.logout(); // Clears internal state
  return redirect('/login');
}
```

### 4. Checking Auth in Every Component

```typescript
// ❌ Bad - repeated checks
function Header() {
  const { user } = useAuth();
  if (!user) return null;
  // ...
}

function Sidebar() {
  const { user } = useAuth();
  if (!user) return null;
  // ...
}

// ✅ Good - check once in layout loader
async function authLayoutLoader() {
  return await requireAuth(/* args */);
}

function AuthLayout() {
  const user = useLoaderData();
  return (
    <>
      <Header user={user} />
      <Sidebar user={user} />
    </>
  );
}
```

## Best Practices

### 1. Centralize Auth Logic

```typescript
// utils/auth.ts - single source of truth
export async function requireAuth(args: LoaderFunctionArgs) {
  const user = await authService.getCurrentUser();
  
  if (!user) {
    const url = new URL(args.request.url);
    throw redirect(`/login?from=${url.pathname}`);
  }
  
  return user;
}

export function requireRole(roles: Role[]) {
  return async (args: LoaderFunctionArgs) => {
    const user = await requireAuth(args);
    
    if (!roles.some(role => user.roles.includes(role))) {
      throw new Response('Forbidden', { status: 403 });
    }
    
    return user;
  };
}
```

### 2. Use TypeScript for Type Safety

```typescript
interface AuthUser {
  id: string;
  roles: Role[];
  permissions: Permission[];
}

type ProtectedLoader<T> = (
  args: LoaderFunctionArgs,
  user: AuthUser
) => Promise<T>;

function withAuth<T>(loader: ProtectedLoader<T>) {
  return async (args: LoaderFunctionArgs) => {
    const user = await requireAuth(args);
    return loader(args, user as AuthUser);
  };
}

// Usage - type-safe
const dashboardLoader = withAuth(async (args, user) => {
  // user is guaranteed to be AuthUser
  const data = await fetchDashboardData(user.id);
  return data;
});
```

### 3. Handle Errors Gracefully

```typescript
function ErrorBoundary() {
  const error = useRouteError();
  
  if (isRouteErrorResponse(error)) {
    if (error.status === 403) {
      return (
        <div>
          <h1>Access Denied</h1>
          <p>You don't have permission to view this page.</p>
          <Link to="/">Go Home</Link>
        </div>
      );
    }
  }
  
  return <GenericError />;
}

const router = createBrowserRouter([
  {
    path: '/admin',
    element: <AdminPanel />,
    loader: requireRole(['admin']),
    errorElement: <ErrorBoundary />
  }
]);
```

### 4. Test Authorization Logic

```typescript
describe('requireAuth', () => {
  it('redirects when not authenticated', async () => {
    vi.spyOn(authService, 'getCurrentUser').mockResolvedValue(null);
    
    const args = {
      request: new Request('http://localhost/dashboard')
    } as LoaderFunctionArgs;
    
    await expect(requireAuth(args)).rejects.toThrow(redirect);
  });
  
  it('returns user when authenticated', async () => {
    const mockUser = { id: '1', name: 'Test' };
    vi.spyOn(authService, 'getCurrentUser').mockResolvedValue(mockUser);
    
    const args = {
      request: new Request('http://localhost/dashboard')
    } as LoaderFunctionArgs;
    
    const result = await requireAuth(args);
    expect(result).toEqual(mockUser);
  });
});
```

## Interview Questions

### Q1: Why is loader-based auth protection better than component-based?

**Answer:**
- No flash of unauthorized content
- Prevents rendering until auth is verified
- Works with JavaScript disabled (server-side)
- Cleaner error handling
- Integrates with React Router's data flow
- Parallel data loading with auth check

### Q2: How would you implement role-based access control?

**Answer:** Create a higher-order function that:
1. Checks authentication
2. Verifies user has required role(s)
3. Returns 403 if unauthorized
4. Returns user if authorized
5. Apply to route loaders

Can be extended to permission-based or resource-based checks.

### Q3: How do you handle session expiration?

**Answer:**
- Validate session in loaders
- Check expiration timestamp
- Redirect to login on expiration
- Optionally refresh access tokens
- Clear auth state on logout
- Handle inactivity timeout

### Q4: What's the best way to share user data across protected routes?

**Answer:** Use a parent layout route with a loader:
- Layout loader checks auth and loads user
- Child routes access via `useRouteLoaderData`
- Avoids redundant auth checks
- Provides consistent user object
- Enables context provider pattern

## Resources

### Official Documentation
- React Router Authentication: https://reactrouter.com/en/main/start/tutorial#adding-authentication
- Remix Auth Patterns: https://remix.run/docs/en/main/guides/authentication

### Libraries
- Auth.js: https://authjs.dev/
- Clerk: https://clerk.com/
- Supabase Auth: https://supabase.com/docs/guides/auth

### Articles
- Route Protection Patterns: https://kentcdodds.com/blog/authentication-in-react-applications
- Session Management: https://www.npmjs.com/package/remix-auth

---

**Next Steps:**
- Implement loader-based auth protection
- Build RBAC system
- Add session management
- Create reusable auth utilities
