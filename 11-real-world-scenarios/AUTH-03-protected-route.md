# Protected Route with Loading State

## The Idea

**In plain English:** Some pages in your app should only be visible to authenticated users. A protected route is a wrapper that checks whether the current user is logged in before rendering the page. If they are not logged in, the app redirects them to the login page. If auth state is still being determined (e.g., an async token check on first load), the route shows nothing, a skeleton, or a spinner — not a flash of the protected content and not a premature redirect.

**Real-world analogy:** Think of a hotel keycard system.

- The **hotel door** is the protected route — it does not open until a valid keycard is presented.
- The **keycard reader blinking amber** is the loading state — the reader is scanning, you do not yet know if the card is valid. The door stays closed.
- The **keycard reader flashing red** is the unauthenticated state — the card is invalid, you are redirected to the front desk (the login page).
- The **keycard reader flashing green** is the authenticated state — the door opens and you see the room (the protected page).
- The **"intended destination" slip** the front desk gives you is the `redirectTo` / `returnUrl` param stored before the redirect — so after you authenticate, you are taken directly back to the room you tried to enter, not dropped in the lobby.
- **White-flash / flicker** is what happens if the door swings open for a fraction of a second before the reader finishes scanning — it is a UX and security anti-pattern that protected route implementations must specifically prevent.

The key insight: auth state is async. Until you know for certain whether the user is authenticated or not, you must not render the protected content, and you must not redirect to login either.

---

## Learning Objectives

- Understand the three distinct auth states: `loading`, `authenticated`, `unauthenticated`
- Store the intended URL before redirect so the user lands back on it after login
- Prevent white-flash / flicker by holding render until auth state resolves
- Choose between showing nothing, a skeleton, or a spinner during the loading state
- Implement the React Router v6 pattern: component guard vs. `loader` guard
- Implement the Angular pattern: functional `canActivate` guard with signals
- Know the difference between a token-presence check (synchronous) and a token-validity check (async, requires a network call)

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Hide protected content while auth state resolves | ⚠️ `visibility: hidden` | Content is still in the DOM — screen readers and view-source expose it |
| Redirect to `/login` if unauthenticated | ❌ | CSS cannot trigger navigation |
| Read a JWT from `localStorage` or a cookie | ❌ | CSS has no access to storage APIs |
| Verify token expiry or call an `/auth/me` endpoint | ❌ | CSS cannot make network requests |
| Store the intended URL as a query param before redirecting | ❌ | Requires JS URL manipulation |
| Conditionally render a skeleton vs. the real page | ❌ | CSS cannot branch on async state |
| Restore scroll position after auth redirect round-trip | ❌ | Requires JS `history` / `scrollTo` |

**Conclusion:** CSS can only style the skeleton or spinner that JS decides to show. Every meaningful decision — whether to render, whether to redirect, what to redirect to — requires JavaScript.

---

## HTML & CSS Foundation

### Skeleton Loader (the right "loading" UI)

A skeleton avoids layout shift when auth resolves and content appears. It mirrors the shape of the protected page without exposing real data.

```html
<!-- Shown while auth state is resolving -->
<div class="skeleton-page" aria-hidden="true" role="presentation">
  <div class="skeleton-header"></div>
  <div class="skeleton-body">
    <div class="skeleton-line skeleton-line--wide"></div>
    <div class="skeleton-line skeleton-line--medium"></div>
    <div class="skeleton-line skeleton-line--narrow"></div>
  </div>
</div>

<!-- Screen reader gets context without seeing skeleton -->
<p class="sr-only" aria-live="polite">Loading, please wait.</p>
```

```css
/* ─── Screen-reader-only utility ─── */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

/* ─── Skeleton tokens ─── */
:root {
  --skeleton-base:      #e0e0e0;
  --skeleton-highlight: #f5f5f5;
  --skeleton-radius:    6px;
  --skeleton-duration:  1.6s;
}

/* ─── Shimmer animation ─── */
@keyframes skeleton-shimmer {
  0%   { background-position: -400px 0; }
  100% { background-position:  400px 0; }
}

.skeleton-header,
.skeleton-line {
  border-radius: var(--skeleton-radius);
  background: linear-gradient(
    90deg,
    var(--skeleton-base) 25%,
    var(--skeleton-highlight) 50%,
    var(--skeleton-base) 75%
  );
  background-size: 800px 100%;
  animation: skeleton-shimmer var(--skeleton-duration) infinite linear;
}

/* ─── Skeleton shapes ─── */
.skeleton-header {
  height: 48px;
  width: 100%;
  margin-bottom: 2rem;
}

.skeleton-line {
  height: 18px;
  margin-bottom: 1rem;
}

.skeleton-line--wide   { width: 100%; }
.skeleton-line--medium { width: 65%; }
.skeleton-line--narrow { width: 40%; }

.skeleton-body {
  padding: 1.5rem;
}

/* ─── Fade-in for real content after auth resolves ─── */
@keyframes auth-fade-in {
  from { opacity: 0; }
  to   { opacity: 1; }
}

.auth-resolved {
  animation: auth-fade-in 0.2s ease-out;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .skeleton-header,
  .skeleton-line,
  .auth-resolved {
    animation: none;
  }
}
```

**Why `aria-hidden="true"` on the skeleton?** The skeleton is decorative scaffolding. Screen readers do not benefit from hearing "loading shimmer bar". The separate `aria-live` region gives them a meaningful message instead.

---

## React Implementation

### Auth Context and Hook

```tsx
// auth/AuthContext.tsx
import {
  createContext, useContext, useEffect, useState,
  type ReactNode
} from 'react';

type AuthStatus = 'loading' | 'authenticated' | 'unauthenticated';

interface User {
  id: string;
  email: string;
  roles: string[];
}

interface AuthState {
  status: AuthStatus;
  user: User | null;
}

interface AuthContextValue extends AuthState {
  signOut: () => Promise<void>;
}

const AuthContext = createContext<AuthContextValue | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [state, setState] = useState<AuthState>({
    status: 'loading',
    user: null,
  });

  useEffect(() => {
    let cancelled = false;

    async function resolveAuth() {
      try {
        // Replace with your real session/token check.
        // This must be async — even a localStorage read followed by a
        // token-expiry check is async in practice.
        const res = await fetch('/api/auth/me', { credentials: 'include' });

        if (cancelled) return;

        if (res.ok) {
          const user: User = await res.json();
          setState({ status: 'authenticated', user });
        } else {
          setState({ status: 'unauthenticated', user: null });
        }
      } catch {
        if (!cancelled) {
          setState({ status: 'unauthenticated', user: null });
        }
      }
    }

    resolveAuth();
    return () => { cancelled = true; };
  }, []);

  const signOut = async () => {
    await fetch('/api/auth/signout', { method: 'POST', credentials: 'include' });
    setState({ status: 'unauthenticated', user: null });
  };

  return (
    <AuthContext.Provider value={{ ...state, signOut }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth(): AuthContextValue {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth must be used within AuthProvider');
  return ctx;
}
```

### Protected Route — Component Guard Pattern

```tsx
// auth/ProtectedRoute.tsx
import { Navigate, useLocation, Outlet } from 'react-router-dom';
import { useAuth } from './AuthContext';
import { PageSkeleton } from '../components/PageSkeleton';

interface ProtectedRouteProps {
  /** Optional role required to access the route. */
  requiredRole?: string;
  /** Custom fallback shown during the loading phase. Defaults to PageSkeleton. */
  loadingFallback?: React.ReactNode;
}

export function ProtectedRoute({
  requiredRole,
  loadingFallback = <PageSkeleton />,
}: ProtectedRouteProps) {
  const { status, user } = useAuth();
  const location = useLocation();

  // Phase 1: auth state not yet resolved — show nothing or skeleton
  // NEVER redirect here — you would incorrectly kick out authenticated users
  // whose session just hasn't loaded yet.
  if (status === 'loading') {
    return <>{loadingFallback}</>;
  }

  // Phase 2: not authenticated — redirect to login, preserving intended URL
  if (status === 'unauthenticated') {
    // `state` is the React Router way; `?returnUrl` is the query-param way.
    // React Router `state` is cleaner but is lost on hard refresh.
    // For hard-refresh safety, encode the path in a query param instead.
    const returnUrl = encodeURIComponent(location.pathname + location.search);
    return <Navigate to={`/login?returnUrl=${returnUrl}`} replace />;
  }

  // Phase 3: authenticated but wrong role
  if (requiredRole && !user?.roles.includes(requiredRole)) {
    return <Navigate to="/forbidden" replace />;
  }

  // Phase 4: authenticated — render the protected page
  return (
    <div className="auth-resolved">
      <Outlet />
    </div>
  );
}
```

### Skeleton Component

```tsx
// components/PageSkeleton.tsx
export function PageSkeleton() {
  return (
    <>
      <div className="skeleton-page" aria-hidden="true" role="presentation">
        <div className="skeleton-header" />
        <div className="skeleton-body">
          <div className="skeleton-line skeleton-line--wide" />
          <div className="skeleton-line skeleton-line--medium" />
          <div className="skeleton-line skeleton-line--narrow" />
        </div>
      </div>
      <p className="sr-only" aria-live="polite">Loading, please wait.</p>
    </>
  );
}
```

### Router Configuration

```tsx
// main.tsx / router.tsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';
import { AuthProvider } from './auth/AuthContext';
import { ProtectedRoute } from './auth/ProtectedRoute';
import { Dashboard } from './pages/Dashboard';
import { AdminPanel } from './pages/AdminPanel';
import { Login } from './pages/Login';

const router = createBrowserRouter([
  { path: '/login', element: <Login /> },

  // All children share the same auth guard
  {
    element: <ProtectedRoute />,
    children: [
      { path: '/dashboard', element: <Dashboard /> },
      { path: '/settings',  element: <Settings /> },
    ],
  },

  // Role-scoped sub-tree
  {
    element: <ProtectedRoute requiredRole="admin" />,
    children: [
      { path: '/admin', element: <AdminPanel /> },
    ],
  },
]);

export function App() {
  return (
    <AuthProvider>
      <RouterProvider router={router} />
    </AuthProvider>
  );
}
```

### Returning to the Intended URL After Login

```tsx
// pages/Login.tsx
import { useNavigate, useSearchParams } from 'react-router-dom';
import { useAuth } from '../auth/AuthContext';

export function Login() {
  const navigate      = useNavigate();
  const [params]      = useSearchParams();
  const { status }    = useAuth();

  // If the user navigates to /login while already authenticated, bounce them.
  useEffect(() => {
    if (status === 'authenticated') {
      const returnUrl = params.get('returnUrl');
      // Validate: only redirect to relative paths — never to an external domain.
      const destination = returnUrl && returnUrl.startsWith('/') 
        ? decodeURIComponent(returnUrl)
        : '/dashboard';
      navigate(destination, { replace: true });
    }
  }, [status, navigate, params]);

  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const data = new FormData(e.currentTarget);
    // ...call sign-in API...
    // AuthContext re-fetches /api/auth/me and sets status → 'authenticated'
    // The useEffect above will then fire and redirect.
  }

  return (
    <form onSubmit={handleSubmit}>
      <input name="email"    type="email"    required />
      <input name="password" type="password" required />
      <button type="submit">Sign in</button>
    </form>
  );
}
```

### React Router `loader` Guard (Alternative Pattern)

```tsx
// auth/authLoader.ts
// Use this when you want the guard to run BEFORE the route component renders,
// rather than inside the component tree. Eliminates the loading-flash entirely
// because React Router suspends the render until the loader resolves.

import { redirect } from 'react-router-dom';
import { getAuthState } from './authState'; // your singleton auth state module

export async function requireAuth({ request }: { request: Request }) {
  const { user } = await getAuthState();   // resolves the async check

  if (!user) {
    const url    = new URL(request.url);
    const returnUrl = encodeURIComponent(url.pathname + url.search);
    throw redirect(`/login?returnUrl=${returnUrl}`);
  }

  return { user };   // available via useLoaderData() in the component
}

// In the router:
// { path: '/dashboard', loader: requireAuth, element: <Dashboard /> }
```

The loader pattern is the cleanest flicker-prevention technique in React Router v6 — but it requires that your auth check is expressible as a standalone async function, not tied to React context.

---

## Angular Implementation

### Auth Service with Signals

```typescript
// auth/auth.service.ts
import { Injectable, signal, computed, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';

export type AuthStatus = 'loading' | 'authenticated' | 'unauthenticated';

export interface User {
  id: string;
  email: string;
  roles: string[];
}

@Injectable({ providedIn: 'root' })
export class AuthService {
  private http = inject(HttpClient);

  private _status = signal<AuthStatus>('loading');
  private _user   = signal<User | null>(null);

  readonly status          = this._status.asReadonly();
  readonly user            = this._user.asReadonly();
  readonly isAuthenticated = computed(() => this._status() === 'authenticated');

  constructor() {
    this.resolveSession();
  }

  private async resolveSession(): Promise<void> {
    try {
      const user = await firstValueFrom(
        this.http.get<User>('/api/auth/me', { withCredentials: true })
      );
      this._user.set(user);
      this._status.set('authenticated');
    } catch {
      this._user.set(null);
      this._status.set('unauthenticated');
    }
  }

  async signOut(): Promise<void> {
    await firstValueFrom(
      this.http.post('/api/auth/signout', {}, { withCredentials: true })
    );
    this._user.set(null);
    this._status.set('unauthenticated');
  }
}
```

### Functional `canActivate` Guard

```typescript
// auth/auth.guard.ts
import { inject } from '@angular/core';
import { Router, type CanActivateFn } from '@angular/router';
import { toObservable } from '@angular/core/rxjs-interop';
import { filter, map, take } from 'rxjs/operators';
import { AuthService } from './auth.service';

export const authGuard: CanActivateFn = (route, state) => {
  const auth   = inject(AuthService);
  const router = inject(Router);

  // Wait until auth status is no longer 'loading', then make the decision once.
  return toObservable(auth.status).pipe(
    filter(status => status !== 'loading'),   // skip the loading phase
    take(1),                                   // complete after the first resolved value
    map(status => {
      if (status === 'authenticated') {
        return true;
      }

      // Store intended URL in query params before redirecting
      const returnUrl = state.url;
      return router.createUrlTree(['/login'], {
        queryParams: { returnUrl },
      });
    })
  );
};

// Role-scoped variant
export function roleGuard(requiredRole: string): CanActivateFn {
  return (_route, state) => {
    const auth   = inject(AuthService);
    const router = inject(Router);

    return toObservable(auth.status).pipe(
      filter(s => s !== 'loading'),
      take(1),
      map(status => {
        if (status !== 'authenticated') {
          return router.createUrlTree(['/login'], {
            queryParams: { returnUrl: state.url },
          });
        }
        const user = auth.user();
        if (!user?.roles.includes(requiredRole)) {
          return router.createUrlTree(['/forbidden']);
        }
        return true;
      })
    );
  };
}
```

### Router Configuration (Angular)

```typescript
// app.routes.ts
import { Routes } from '@angular/router';
import { authGuard, roleGuard } from './auth/auth.guard';
import { DashboardComponent } from './pages/dashboard.component';
import { AdminPanelComponent } from './pages/admin-panel.component';
import { LoginComponent } from './pages/login.component';

export const routes: Routes = [
  { path: 'login', component: LoginComponent },

  {
    path: 'dashboard',
    component: DashboardComponent,
    canActivate: [authGuard],
  },

  {
    path: 'admin',
    component: AdminPanelComponent,
    canActivate: [roleGuard('admin')],
  },

  { path: '**', redirectTo: 'dashboard' },
];
```

### Skeleton During Route Resolution

Angular's guard runs before the component is created, so the page is blank while the guard's observable resolves. Use a global loading indicator tied to Angular Router events, or the `withNavigationErrorHandler` / `withRouterConfig` APIs.

```typescript
// app.component.ts
import {
  Component, inject, signal
} from '@angular/core';
import { Router, NavigationStart, NavigationEnd, NavigationCancel, NavigationError } from '@angular/router';
import { toSignal } from '@angular/core/rxjs-interop';
import { map } from 'rxjs/operators';
import { AuthService } from './auth/auth.service';

@Component({
  selector: 'app-root',
  standalone: true,
  template: `
    <!-- Auth-level loading: show skeleton until auth resolves for the first time -->
    @if (authService.status() === 'loading') {
      <app-page-skeleton />
    } @else {
      <!-- Navigation-level loading: show skeleton during subsequent guard checks -->
      @if (isNavigating()) {
        <app-page-skeleton />
      }
      <router-outlet />
    }
  `,
})
export class AppComponent {
  authService = inject(AuthService);

  private router = inject(Router);

  // Derive a boolean signal from Router events
  isNavigating = toSignal(
    this.router.events.pipe(
      map(event => {
        if (event instanceof NavigationStart)                  return true;
        if (event instanceof NavigationEnd)                    return false;
        if (event instanceof NavigationCancel)                 return false;
        if (event instanceof NavigationError)                  return false;
        return null;  // other events — no change
      }),
      // filter out nulls so signal only changes on meaningful events
      // (toSignal initial value is false)
    ),
    { initialValue: false }
  );
}
```

### Login Component — Return to Intended URL

```typescript
// pages/login.component.ts
import { Component, inject, effect } from '@angular/core';
import { Router, ActivatedRoute } from '@angular/router';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { AuthService } from '../auth/auth.service';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';

@Component({
  selector: 'app-login',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="submit()">
      <input formControlName="email"    type="email"    required />
      <input formControlName="password" type="password" required />
      <button type="submit" [disabled]="form.invalid || submitting">Sign in</button>
      @if (error) { <p role="alert">{{ error }}</p> }
    </form>
  `,
})
export class LoginComponent {
  private fb      = inject(FormBuilder);
  private auth    = inject(AuthService);
  private router  = inject(Router);
  private route   = inject(ActivatedRoute);
  private http    = inject(HttpClient);

  form = this.fb.group({
    email:    ['', [Validators.required, Validators.email]],
    password: ['', Validators.required],
  });

  submitting = false;
  error: string | null = null;

  constructor() {
    // Redirect away if already authenticated
    effect(() => {
      if (this.auth.isAuthenticated()) {
        const returnUrl = this.route.snapshot.queryParamMap.get('returnUrl');
        const destination = this.isSafeUrl(returnUrl) ? returnUrl! : '/dashboard';
        this.router.navigateByUrl(destination, { replaceUrl: true });
      }
    });
  }

  async submit() {
    if (this.form.invalid) return;
    this.submitting = true;
    this.error = null;

    try {
      const { email, password } = this.form.getRawValue();
      await firstValueFrom(
        this.http.post('/api/auth/login', { email, password }, { withCredentials: true })
      );
      // AuthService constructor calls resolveSession — call it again after login
      // OR emit an event / update the signal directly here.
    } catch (e: unknown) {
      this.error = 'Invalid credentials. Please try again.';
    } finally {
      this.submitting = false;
    }
  }

  private isSafeUrl(url: string | null): boolean {
    // Only allow relative paths to prevent open-redirect attacks
    return !!url && url.startsWith('/') && !url.startsWith('//');
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Skeleton is hidden from screen readers | `aria-hidden="true"` on the skeleton container |
| Loading state is announced to screen readers | `aria-live="polite"` with "Loading, please wait." text |
| Login form error is announced immediately | `role="alert"` on the error paragraph (implicitly `aria-live="assertive"`) |
| Login form fields have associated labels | `<label for="...">` or `aria-label` on each input |
| Submit button is disabled during submission | `[disabled]` prevents double-submit, informs screen reader |
| Forbidden / error pages have a descriptive `<h1>` | Gives screen reader users context after navigation |
| Focus is managed after auth redirect round-trip | React Router `<ScrollRestoration />` or Angular `scrollPositionRestoration: 'enabled'` |
| Skeleton animation respects `prefers-reduced-motion` | `@media (prefers-reduced-motion)` disables shimmer |
| Page title updates on each protected route | `<title>` or Angular `Title.setTitle()` — important for tab context |
| Login page is not indexable if SSR | `<meta name="robots" content="noindex">` (not an a11y concern, but often paired) |

---

## Production Pitfalls

**1. Redirecting to `/login` during the loading phase**
The most common bug. If your guard checks `user === null` without first checking whether auth is still loading, authenticated users whose session resolves asynchronously are kicked to the login page on every hard refresh. The fix is a three-way branch: `loading` → show skeleton, `unauthenticated` → redirect, `authenticated` → render. Never conflate `null` (not loaded) with `null` (not logged in).

**2. Open redirect via `returnUrl`**
If you redirect to whatever URL is in the query param, an attacker can craft a link like `/login?returnUrl=https://evil.com` and, after login, the user is redirected to an external site. Always validate that `returnUrl` starts with `/` and does not start with `//` (protocol-relative). Reject anything else and fall back to a safe default.

**3. White flash before redirect fires**
If your component renders the protected content for a single frame before the `useEffect` / `effect()` fires and redirects, the user briefly sees the protected page. The fix is to return `null` or a skeleton during the loading phase and to structure your conditional so the redirect case is evaluated before any JSX that renders protected content. The React Router `loader` guard eliminates this entirely.

**4. Storing `returnUrl` in `localStorage` instead of a query param**
`localStorage` is shared across all tabs. If the user has two tabs open and is redirected in one, the second tab overwrites the stored URL. Query params are scoped to the specific navigation, making them more reliable.

**5. Auth context not initialized before the router renders**
In React, wrapping `<RouterProvider>` inside `<AuthProvider>` is correct. Getting it backwards means the first render of every protected route has no auth context, causing a thrown error instead of a graceful loading state. In Angular, the `AuthService` is `providedIn: 'root'` and its constructor kicks off `resolveSession` immediately — but if the guard is evaluated before the service is injected, signals read during route resolution may not yet exist.

**6. Skeleton that looks nothing like the real page**
A skeleton that is wildly different in height and layout from the authenticated page causes a jarring layout shift when auth resolves. The skeleton should mirror the real page's approximate layout — same number of sections, same rough heights. Use a `min-height` on the page wrapper to anchor the skeleton and prevent collapse.

**7. Re-fetching auth state on every navigation**
Calling `/api/auth/me` inside the guard itself (rather than relying on a shared service/context) means every navigation triggers a network request. Cache the resolved user in a singleton service or context. Only re-validate on explicit events: browser focus, a `401` response from any API call, or a configurable interval.

---

## Interview Angle

**Q: "How would you implement a protected route that does not flash the protected content while auth state is resolving?"**

The key is recognizing that there are three states — `loading`, `authenticated`, `unauthenticated` — not two. While `status === 'loading'`, the guard renders a skeleton (or nothing at all) and neither shows the content nor redirects. Only once the async check resolves does the guard branch: redirect to `/login` with `returnUrl` encoded in the query string, or render the protected content wrapped in a fade-in. In React Router v6, the `loader` function pattern is even cleaner because the route simply does not render until the loader's promise resolves, eliminating the flicker entirely. In Angular, the `canActivate` guard returns an observable filtered to the first non-loading status value, so the router waits for auth to resolve before mounting the component.

**Follow-up: "What is an open redirect and how does protected-route code introduce one?"**

An open redirect happens when your login page reads a `returnUrl` from query params and blindly redirects to it after authentication. An attacker sends a phishing link like `https://yourapp.com/login?returnUrl=https://evil.com/steal-credentials`. After the user logs in on your real domain, they are silently forwarded to the attacker's page. The fix is a single validation function: accept the redirect only if the URL is a relative path (`url.startsWith('/') && !url.startsWith('//')`). Any other value falls back to a safe default like `/dashboard`. This check belongs in the login component, not in the guard.
