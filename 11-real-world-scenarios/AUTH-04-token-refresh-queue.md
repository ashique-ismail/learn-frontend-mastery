# Token Refresh with Request Queue

## The Idea

**In plain English:** When your access token expires, the API returns a 401. If you have ten concurrent requests all firing at the same moment, all ten will receive a 401 simultaneously. A naive implementation would try to refresh the token ten times in parallel, causing race conditions, duplicate refresh calls, and potential logout loops. The correct pattern: the first 401 triggers exactly one refresh call; every other 401 that arrives while that refresh is in-flight joins a queue and waits; once the refresh succeeds, all queued requests are replayed with the new token. If the refresh itself fails, every queued request is rejected and the user is logged out.

**Real-world analogy:** Think of a security desk at a corporate building.

- Your **ID badge** is the access token — it expires at the end of the day.
- The **401 response** is the turnstile refusing your badge.
- The **token refresh call** is one employee going to the front desk to renew their badge.
- The **request queue** is every other employee who was also denied — they wait in a line by the turnstile rather than all rushing the front desk at once.
- The **refresh success** is the employee returning with a master key that unlocks the turnstile for everyone waiting.
- The **refresh failure** is the front desk saying "this person is no longer an employee" — the building calls security and escorts everyone out (logout).

The key insight: the refresh is a shared resource. The first requestor owns the renewal process; everyone else subscribes to its outcome.

---

## Learning Objectives

- Understand why concurrent 401s require a queuing strategy rather than independent refresh calls
- Implement the single in-flight refresh promise pattern shared across interceptors
- Build an Axios request interceptor that handles the full refresh + retry lifecycle
- Build the equivalent Angular HttpClient interceptor using RxJS
- Handle the logout-on-failure path cleanly so no queued requests hang forever
- Recognise the production edge cases: infinite retry loops, token races on page load, refresh token rotation

---

## Why CSS Alone Isn't Enough

This topic is entirely in the HTTP / JavaScript layer. There is no CSS component. Skip to the implementation sections below.

---

## HTML & CSS Foundation

There is no HTML or CSS specific to this pattern. The token refresh queue lives entirely in the network layer — an HTTP interceptor that sits between your application code and the server. Application code calls `api.get('/users')` exactly as before; the interceptor handles expiry and retry invisibly.

The one UI concern is a loading or lockout state while refresh is in progress. A simple approach:

```html
<!-- Show a non-interactive overlay while a critical refresh is happening -->
<div
  id="auth-refresh-overlay"
  role="status"
  aria-live="polite"
  aria-label="Refreshing your session, please wait"
  hidden
>
  <span class="spinner" aria-hidden="true"></span>
  <p>Refreshing session...</p>
</div>
```

```css
#auth-refresh-overlay {
  position: fixed;
  inset: 0;
  background: rgba(255, 255, 255, 0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: column;
  gap: 1rem;
  z-index: 9999;
}

#auth-refresh-overlay[hidden] {
  display: none;
}

.spinner {
  width: 40px;
  height: 40px;
  border: 3px solid #ddd;
  border-top-color: #0066cc;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

@media (prefers-reduced-motion: reduce) {
  .spinner {
    animation: none;
    border-top-color: #0066cc;
  }
}
```

In practice most applications skip this overlay and let individual request loading states handle UX — the queue resolves fast enough that the user rarely notices.

---

## React Implementation

### Token Storage and Types

```typescript
// auth/types.ts
export interface TokenPair {
  accessToken: string;
  refreshToken: string;
}

export interface QueueEntry {
  resolve: (token: string) => void;
  reject: (error: unknown) => void;
}
```

### The Core Interceptor Module

```typescript
// auth/axiosInterceptor.ts
import axios, {
  AxiosInstance,
  AxiosError,
  InternalAxiosRequestConfig,
} from 'axios';
import type { QueueEntry, TokenPair } from './types';

// ─── Module-level state ────────────────────────────────────────────────────
// These live outside React — they are NOT in useState.
// They must survive across multiple concurrent requests.

let isRefreshing = false;
let failedQueue: QueueEntry[] = [];

function processQueue(error: unknown, token: string | null): void {
  failedQueue.forEach(({ resolve, reject }) => {
    if (error) {
      reject(error);
    } else {
      resolve(token!);
    }
  });
  failedQueue = [];
}

// ─── Token accessors ──────────────────────────────────────────────────────
// Swap these for your actual storage mechanism (localStorage, memory, etc.)

function getAccessToken(): string | null {
  return localStorage.getItem('accessToken');
}

function getRefreshToken(): string | null {
  return localStorage.getItem('refreshToken');
}

function saveTokens(pair: TokenPair): void {
  localStorage.setItem('accessToken', pair.accessToken);
  localStorage.setItem('refreshToken', pair.refreshToken);
}

function clearTokens(): void {
  localStorage.removeItem('accessToken');
  localStorage.removeItem('refreshToken');
}

// ─── Axios instance ────────────────────────────────────────────────────────

export const apiClient: AxiosInstance = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 10_000,
});

// ─── Request interceptor: attach access token ─────────────────────────────

apiClient.interceptors.request.use(
  (config: InternalAxiosRequestConfig) => {
    const token = getAccessToken();
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// ─── Response interceptor: handle 401 ─────────────────────────────────────

apiClient.interceptors.response.use(
  // Pass successful responses straight through
  (response) => response,

  async (error: AxiosError) => {
    const originalRequest = error.config as InternalAxiosRequestConfig & {
      _retry?: boolean;
    };

    // Only intercept 401s — let everything else bubble up normally
    if (error.response?.status !== 401) {
      return Promise.reject(error);
    }

    // Prevent infinite retry: if this request already had one retry, give up
    if (originalRequest._retry) {
      return Promise.reject(error);
    }

    // ── Case 1: A refresh is already in progress ───────────────────────────
    // Queue this request and return a promise that will be resolved or
    // rejected once the in-flight refresh completes.
    if (isRefreshing) {
      return new Promise<string>((resolve, reject) => {
        failedQueue.push({ resolve, reject });
      })
        .then((newToken) => {
          originalRequest.headers.Authorization = `Bearer ${newToken}`;
          return apiClient(originalRequest);
        })
        .catch((err) => Promise.reject(err));
    }

    // ── Case 2: This is the first 401 — own the refresh ───────────────────
    originalRequest._retry = true;
    isRefreshing = true;

    const refreshToken = getRefreshToken();

    if (!refreshToken) {
      // No refresh token at all — log out immediately
      isRefreshing = false;
      processQueue(new Error('No refresh token'), null);
      clearTokens();
      window.dispatchEvent(new CustomEvent('auth:logout'));
      return Promise.reject(error);
    }

    try {
      // Call the refresh endpoint directly with plain axios (not apiClient)
      // to avoid this interceptor catching a 401 from the refresh call itself
      const { data } = await axios.post<TokenPair>(
        `${import.meta.env.VITE_API_BASE_URL}/auth/refresh`,
        { refreshToken },
        { timeout: 10_000 }
      );

      saveTokens(data);
      const newAccessToken = data.accessToken;

      // Resolve all queued requests with the new token
      processQueue(null, newAccessToken);

      // Retry the original request that triggered the refresh
      originalRequest.headers.Authorization = `Bearer ${newAccessToken}`;
      return apiClient(originalRequest);
    } catch (refreshError) {
      // Refresh failed — reject all queued requests and log out
      processQueue(refreshError, null);
      clearTokens();
      window.dispatchEvent(new CustomEvent('auth:logout'));
      return Promise.reject(refreshError);
    } finally {
      isRefreshing = false;
    }
  }
);
```

### React Hook — Respond to Logout Events

```tsx
// auth/useAuthLogout.ts
import { useEffect } from 'react';
import { useNavigate } from 'react-router-dom';

/**
 * Listens for the custom 'auth:logout' event dispatched by the interceptor
 * and redirects the user to the login page.
 */
export function useAuthLogout(): void {
  const navigate = useNavigate();

  useEffect(() => {
    const handler = () => {
      navigate('/login', { replace: true, state: { reason: 'session_expired' } });
    };

    window.addEventListener('auth:logout', handler);
    return () => window.removeEventListener('auth:logout', handler);
  }, [navigate]);
}
```

### Wiring It Into the App

```tsx
// App.tsx
import { useAuthLogout } from './auth/useAuthLogout';
import { apiClient } from './auth/axiosInterceptor';

export function App() {
  // One instance at the root handles logout events from anywhere
  useAuthLogout();

  return <RouterOutlet />;
}

// Usage in any component or data-fetching layer:
async function fetchDashboardData() {
  // No token logic here — the interceptor handles it transparently
  const { data } = await apiClient.get('/dashboard');
  return data;
}
```

---

## Angular Implementation

### Token Service

```typescript
// auth/token.service.ts
import { Injectable, signal } from '@angular/core';
import type { TokenPair } from './types';

@Injectable({ providedIn: 'root' })
export class TokenService {
  private _accessToken  = signal<string | null>(null);
  private _refreshToken = signal<string | null>(null);

  readonly accessToken  = this._accessToken.asReadonly();
  readonly refreshToken = this._refreshToken.asReadonly();

  constructor() {
    // Rehydrate from storage on startup
    this._accessToken.set(localStorage.getItem('accessToken'));
    this._refreshToken.set(localStorage.getItem('refreshToken'));
  }

  save(pair: TokenPair): void {
    this._accessToken.set(pair.accessToken);
    this._refreshToken.set(pair.refreshToken);
    localStorage.setItem('accessToken', pair.accessToken);
    localStorage.setItem('refreshToken', pair.refreshToken);
  }

  clear(): void {
    this._accessToken.set(null);
    this._refreshToken.set(null);
    localStorage.removeItem('accessToken');
    localStorage.removeItem('refreshToken');
  }

  hasTokens(): boolean {
    return !!this._accessToken() && !!this._refreshToken();
  }
}
```

### Auth Interceptor (Angular HttpClient)

```typescript
// auth/auth.interceptor.ts
import {
  HttpInterceptorFn,
  HttpRequest,
  HttpHandlerFn,
  HttpErrorResponse,
  HttpEvent,
} from '@angular/common/http';
import {
  inject,
  PLATFORM_ID,
} from '@angular/core';
import { isPlatformBrowser } from '@angular/common';
import {
  Observable,
  throwError,
  BehaviorSubject,
  from,
} from 'rxjs';
import {
  catchError,
  filter,
  take,
  switchMap,
  finalize,
} from 'rxjs/operators';
import { TokenService } from './token.service';
import { AuthService } from './auth.service';
import type { TokenPair } from './types';

// ─── Module-level refresh state (lives outside DI) ───────────────────────
// BehaviorSubject<null> = refresh in progress
// BehaviorSubject<string> = refresh done, value is new access token
let isRefreshing = false;
const refreshTokenSubject = new BehaviorSubject<string | null>(null);

function addAuthHeader(
  req: HttpRequest<unknown>,
  token: string
): HttpRequest<unknown> {
  return req.clone({
    setHeaders: { Authorization: `Bearer ${token}` },
  });
}

export const authInterceptor: HttpInterceptorFn = (
  req: HttpRequest<unknown>,
  next: HttpHandlerFn
): Observable<HttpEvent<unknown>> => {
  const tokenService = inject(TokenService);
  const authService  = inject(AuthService);
  const platformId   = inject(PLATFORM_ID);

  // Skip on the server (SSR) — no tokens available
  if (!isPlatformBrowser(platformId)) {
    return next(req);
  }

  // Skip the refresh endpoint itself to avoid interceptor loops
  if (req.url.includes('/auth/refresh')) {
    return next(req);
  }

  // Attach current access token
  const accessToken = tokenService.accessToken();
  const authReq = accessToken ? addAuthHeader(req, accessToken) : req;

  return next(authReq).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status !== 401) {
        return throwError(() => error);
      }

      // ── Case 1: Refresh already in progress ──────────────────────────
      // Wait for the BehaviorSubject to emit a non-null value (new token),
      // then retry this request once.
      if (isRefreshing) {
        return refreshTokenSubject.pipe(
          filter((token): token is string => token !== null),
          take(1),
          switchMap((newToken) => next(addAuthHeader(req, newToken)))
        );
      }

      // ── Case 2: Own the refresh ───────────────────────────────────────
      isRefreshing = true;
      refreshTokenSubject.next(null); // signal: refresh in flight

      const refreshToken = tokenService.refreshToken();

      if (!refreshToken) {
        isRefreshing = false;
        tokenService.clear();
        authService.logout();
        return throwError(() => error);
      }

      return from(authService.refreshTokens(refreshToken)).pipe(
        switchMap((pair: TokenPair) => {
          tokenService.save(pair);
          refreshTokenSubject.next(pair.accessToken); // unblock the queue
          return next(addAuthHeader(req, pair.accessToken));
        }),
        catchError((refreshError) => {
          tokenService.clear();
          authService.logout();
          return throwError(() => refreshError);
        }),
        finalize(() => {
          isRefreshing = false;
        })
      );
    })
  );
};
```

### Auth Service

```typescript
// auth/auth.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Router } from '@angular/router';
import { firstValueFrom } from 'rxjs';
import { environment } from '../environments/environment';
import type { TokenPair } from './types';

@Injectable({ providedIn: 'root' })
export class AuthService {
  private http   = inject(HttpClient);
  private router = inject(Router);

  async refreshTokens(refreshToken: string): Promise<TokenPair> {
    // Uses plain HttpClient — the interceptor skips /auth/refresh URLs
    return firstValueFrom(
      this.http.post<TokenPair>(`${environment.apiUrl}/auth/refresh`, {
        refreshToken,
      })
    );
  }

  logout(): void {
    // Navigate to login with a reason flag so the UI can show a message
    this.router.navigate(['/login'], {
      replaceUrl: true,
      state: { reason: 'session_expired' },
    });
  }
}
```

### Registering the Interceptor

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { provideRouter } from '@angular/router';
import { authInterceptor } from './auth/auth.interceptor';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(withInterceptors([authInterceptor])),
  ],
};
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| User is not silently logged out without feedback | `auth:logout` event / `router.navigate` includes `reason: 'session_expired'`; login page reads state and shows a banner |
| Session expiry banner is announced to screen readers | Banner rendered with `role="alert"` or `aria-live="assertive"` so it interrupts immediately |
| Refresh overlay (if shown) is non-interactive | `pointer-events: none` on overlay; `aria-hidden="true"` on spinner so screen readers skip the decoration |
| Refresh overlay status announced | Container uses `role="status"` and `aria-live="polite"` |
| Loading states during queued request retry are preserved | Individual request loading states remain active until their Promise/Observable resolves — no extra work needed |
| Logout redirect does not leave orphaned focus | `router.navigate` with `replaceUrl: true` causes a full page transition; login page sets focus to heading on mount |
| Error messages after refresh failure are descriptive | Catch block in consuming code receives the original error object; UI should say "Your session expired. Please log in again." not a generic error |

---

## Production Pitfalls

**1. Interceptor intercepts the refresh call itself, causing an infinite 401 loop**
If your refresh endpoint also returns a 401 (e.g. the refresh token is invalid), the interceptor will try to refresh again, trigger another 401, and recurse indefinitely. Fix: explicitly skip the refresh URL in your interceptor (`if (req.url.includes('/auth/refresh')) return next(req)`) and mark the original request with `_retry = true` so a second 401 on the same request is passed through directly.

**2. `isRefreshing` is reset in the wrong place**
If `isRefreshing = false` is set inside the `try` block before queued requests have replayed, subsequent 401s on replayed requests will start a second refresh. Always reset `isRefreshing` in a `finally` block after the queue has been fully processed.

**3. Multiple browser tabs race on the same refresh token**
Each tab runs its own interceptor in separate JS contexts. If two tabs both hit 401 at the same time, both will call the refresh endpoint. If your server uses refresh token rotation (single-use tokens), one will succeed and one will get a 401 from the refresh endpoint — triggering logout. Fix: use the BroadcastChannel API or localStorage events to coordinate across tabs: one tab claims the refresh lock by writing a timestamp; others wait and read the new token from storage.

**4. Stored tokens in localStorage are accessible to XSS**
`localStorage` is readable by any JS on the page. For high-security applications, store the refresh token in an `HttpOnly` cookie (server-set, not readable by JS). The access token can live in memory (a module-level variable or signal) — it survives navigation but is lost on hard refresh, at which point the `HttpOnly` cookie is used to get a new pair on first API call.

**5. Queued requests replay with stale request bodies**
Axios clones the original config including the body. Angular `HttpRequest` is immutable by design — `req.clone()` copies the body. Verify this holds for multipart form data: `FormData` objects are mutable and passed by reference. If anything mutates the `FormData` between the original request and the retry, the retry sends corrupted data. Fix: avoid mutating request bodies after they are sent; use plain JSON objects where possible.

**6. The queue leaks memory if never drained**
If the refresh endpoint is unreachable (network outage) and your request timeout is long, `failedQueue` will grow unbounded. Fix: add a timeout to the refresh call itself and ensure `processQueue(error, null)` is always called in the catch block. Never leave the queue in a state where entries are added but never consumed.

**7. Token expiry detected too late causes UX flicker**
Waiting for a 401 means the request has already been sent, failed, and must be retried. A proactive approach: decode the JWT on the client, check `exp` before each request, and pre-emptively refresh if the token expires within the next 60 seconds. This eliminates the 401 round-trip entirely for normal usage.

---

## Interview Angle

**Q: "What happens if ten API requests all fire at the same moment and the access token has just expired?"**

All ten will receive 401 responses simultaneously. A naive implementation attaches a `.catch` to each that calls the refresh endpoint, resulting in ten concurrent refresh calls. This causes race conditions: the first refresh may succeed and invalidate the old refresh token (if rotation is enabled), causing the remaining nine refresh calls to fail with their own 401s, triggering nine logouts. The correct pattern uses a module-level boolean (`isRefreshing`) and a queue. The first 401 sets `isRefreshing = true` and owns the single refresh call. Every subsequent 401 that arrives while the flag is set pushes a `{ resolve, reject }` callback pair onto the queue rather than calling refresh again. When the refresh completes, `processQueue` iterates the array, calling either `resolve(newToken)` or `reject(error)` on every entry. Each queued request then retries with the new token or surfaces its error to the caller.

**Follow-up: "How do you handle this pattern across multiple browser tabs?"**

The single-tab queue works within one JS execution context, but separate tabs have separate module state. If two tabs both hit 401, both will attempt a refresh. With single-use refresh token rotation this results in one tab succeeding and one being logged out. The solution is cross-tab coordination via `BroadcastChannel` (modern browsers) or `localStorage` events (broader compatibility). One tab acquires a distributed lock by writing a `refreshing_until` timestamp to `localStorage`. Other tabs that detect an in-progress refresh wait on a `storage` event for the `accessToken` key to update, then read the new value and resume — they never call the refresh endpoint themselves. If the lock-holder tab is closed mid-refresh, the other tabs detect the stale timestamp and one of them claims the lock and retries.