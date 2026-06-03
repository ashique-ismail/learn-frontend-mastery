# Session Management and Logout Complexity

## Overview

"User logs out. Clear the token." That's what it looks like on the surface. In a real production app, a correct logout involves: multiple tabs, active WebSocket connections, in-flight network requests, service worker caches, client-side state, and cross-tab notification. Getting any one wrong means the user is either not fully logged out (security risk) or loses in-progress work (UX failure).

---

## The Complete Logout Checklist

```
On logout:
  1. Revoke the session on the server
  2. Clear auth tokens (localStorage, sessionStorage, cookies)
  3. Cancel in-flight requests
  4. Close WebSocket connections
  5. Clear client-side cache (React Query, Apollo, Redux)
  6. Clear sensitive state from memory
  7. Notify other tabs
  8. Clear service worker cache (if caching auth'd responses)
  9. Redirect to login page
```

---

## Token Storage and Logout

### HttpOnly Cookies

The most secure option. Tokens are not accessible to JavaScript — only the browser sends them automatically.

```
// Server sets:
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict; Path=/

// Logout:
POST /auth/logout
  → Server: DELETE FROM sessions WHERE token = 'abc123'
  → Server response: Set-Cookie: session=; Max-Age=0  (clears the cookie)
  → Browser: cookie removed, subsequent requests unauthenticated
```

```javascript
// Client-side: you can't clear HttpOnly cookies — the server must do it
async function logout() {
  await fetch('/auth/logout', { method: 'POST', credentials: 'include' });
  // Server response will clear the cookie
  // Now clean up client state
  clearClientState();
}
```

### localStorage Tokens (JWT)

Accessible to JavaScript. Must be cleared explicitly.

```javascript
function clearAuthStorage() {
  localStorage.removeItem('access_token');
  localStorage.removeItem('refresh_token');
  sessionStorage.clear(); // belt and suspenders
}
```

**The silent bug:** `localStorage.clear()` clears everything, including non-auth data (preferences, draft content). Use specific key removal unless you intend to nuke everything.

---

## Cancelling In-Flight Requests

When the user logs out mid-navigation, there may be pending fetch calls. If they complete after logout, they may write to a now-cleared state or trigger errors:

```typescript
class RequestManager {
  private controllers = new Map<string, AbortController>();

  fetch(key: string, url: string, options?: RequestInit): Promise<Response> {
    // Cancel any existing request with this key
    this.controllers.get(key)?.abort();

    const controller = new AbortController();
    this.controllers.set(key, controller);

    return fetch(url, { ...options, signal: controller.signal })
      .finally(() => this.controllers.delete(key));
  }

  cancelAll(): void {
    this.controllers.forEach(controller => controller.abort());
    this.controllers.clear();
  }
}

export const requestManager = new RequestManager();

// On logout:
requestManager.cancelAll();
```

---

## Closing WebSocket Connections

WebSocket connections survive navigation and are not automatically closed on logout. A logged-out user whose WebSocket is still open will continue receiving real-time events intended for the authenticated user.

```typescript
class WebSocketManager {
  private ws: WebSocket | null = null;
  private subscriptions = new Set<() => void>(); // cleanup callbacks

  connect(url: string): void {
    this.ws = new WebSocket(url);
    // ... setup handlers
  }

  disconnect(reason = 'logout'): void {
    if (this.ws) {
      this.ws.close(1000, reason); // 1000 = normal closure
      this.ws = null;
    }

    // Run all subscription cleanup callbacks
    this.subscriptions.forEach(cleanup => cleanup());
    this.subscriptions.clear();
  }
}

// On logout:
wsManager.disconnect('logout');
```

---

## Clearing the Client-Side Cache

### React Query

```typescript
async function logout() {
  await api.logout();

  // Option A: clear all cached data
  queryClient.clear();

  // Option B: clear only sensitive queries, keep public data
  queryClient.removeQueries({ queryKey: ['user'] });
  queryClient.removeQueries({ queryKey: ['orders'] });
  queryClient.removeQueries({ queryKey: ['profile'] });

  // Reset all queries to unfetched state (triggers refetch on next mount)
  queryClient.resetQueries();
}
```

### Apollo Client (GraphQL)

```typescript
async function logout() {
  await api.logout();
  await apolloClient.clearStore();     // clears cache, doesn't refetch
  // or
  await apolloClient.resetStore();     // clears cache AND refetches active queries
}
```

### Redux

```typescript
// Add a root reducer that handles logout
const rootReducer = (state: RootState | undefined, action: Action) => {
  if (action.type === 'auth/logout') {
    // Return initial state — effectively clears all slices
    return appReducer(undefined, action);
  }
  return appReducer(state, action);
};
```

---

## Multi-Tab Notification

```typescript
// Tab A: user clicks logout
async function logout() {
  await api.logout();
  clearClientState();

  // Notify all other tabs
  localStorage.setItem('logout-event', Date.now().toString());
  localStorage.removeItem('logout-event'); // immediately — just need the event to fire

  redirectToLogin();
}

// All tabs: listen for the logout signal
window.addEventListener('storage', (event) => {
  if (event.key === 'logout-event') {
    clearClientState();
    redirectToLogin();
  }
});
```

**BroadcastChannel (more explicit):**

```typescript
const logoutChannel = new BroadcastChannel('auth');

// Sender tab:
logoutChannel.postMessage({ type: 'LOGOUT' });

// All other tabs:
logoutChannel.onmessage = (event) => {
  if (event.data.type === 'LOGOUT') {
    clearClientState();
    redirectToLogin();
  }
};
```

---

## Service Worker Cache

If your service worker caches authenticated API responses, logout must clear those too:

```javascript
// service-worker.js: listen for logout message
self.addEventListener('message', async (event) => {
  if (event.data?.type === 'LOGOUT') {
    const cacheKeys = await caches.keys();
    await Promise.all(
      cacheKeys
        .filter(key => key.startsWith('api-cache'))
        .map(key => caches.delete(key))
    );
  }
});

// App: send message to service worker on logout
async function logout() {
  await api.logout();
  clearClientState();

  // Tell the service worker to clear its caches
  if ('serviceWorker' in navigator && navigator.serviceWorker.controller) {
    navigator.serviceWorker.controller.postMessage({ type: 'LOGOUT' });
  }

  redirectToLogin();
}
```

---

## The Full Logout Implementation

```typescript
async function performLogout(): Promise<void> {
  try {
    // 1. Revoke server session (best-effort — proceed even if it fails)
    await Promise.race([
      api.post('/auth/logout'),
      new Promise((_, reject) => setTimeout(() => reject(new Error('timeout')), 3000)),
    ]).catch(() => {}); // don't block logout if server call fails

    // 2. Cancel in-flight requests
    requestManager.cancelAll();

    // 3. Close WebSocket
    wsManager.disconnect('logout');

    // 4. Clear auth tokens
    localStorage.removeItem('access_token');
    localStorage.removeItem('refresh_token');
    document.cookie = 'session=; Max-Age=0; path=/'; // clear non-HttpOnly cookies

    // 5. Clear client cache
    queryClient.clear();

    // 6. Clear sensitive Redux state
    store.dispatch({ type: 'auth/logout' });

    // 7. Clear service worker cache
    navigator.serviceWorker?.controller?.postMessage({ type: 'LOGOUT' });

    // 8. Notify other tabs
    const logoutChannel = new BroadcastChannel('auth');
    logoutChannel.postMessage({ type: 'LOGOUT' });
    logoutChannel.close();

    // 9. Redirect
    window.location.href = '/login?reason=logout'; // hard navigation, clears JS state
  } catch (error) {
    // Always redirect even if something fails
    console.error('Logout error:', error);
    window.location.href = '/login?reason=logout';
  }
}
```

Note the `window.location.href` redirect (hard navigation) rather than `router.push('/login')`. A hard navigation destroys the JS context, clearing any in-memory state you might have missed.

---

## Handling the Logged-Out State (Expired Sessions)

Sessions expire. The user was authenticated when they loaded the page, but their session expires while they're using it. Every API call after that returns 401.

```typescript
// Axios interceptor: handle 401 globally
axios.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      // Try to refresh the token first
      try {
        await refreshToken();
        return axios.request(error.config); // retry the original request
      } catch {
        // Refresh also failed — session is truly expired
        performLogout();
      }
    }
    return Promise.reject(error);
  }
);
```

---

## Interview Questions

**Q: "User logs out. Walk me through everything that needs to happen."**
A: (1) Revoke the session server-side — the client-side cleanup is useless if the server still accepts the old token. (2) Clear auth tokens from storage. (3) Cancel in-flight requests — they'll 401 anyway but cancelling is cleaner. (4) Close WebSocket connections — they'd otherwise keep receiving real-time events. (5) Clear the client-side cache (React Query, Redux, Apollo) — a subsequent login by a different user must not see the previous user's data. (6) Notify other open tabs via BroadcastChannel. (7) Clear service worker cache if you're caching auth'd API responses. (8) Hard-navigate to login — not router.push, but window.location.href — to destroy the JS context entirely.

**Q: "What's the security risk if you only clear tokens client-side but don't revoke server-side?"**
A: If an attacker exfiltrated the token (XSS, network interception) before logout, they still hold a valid token. The user logged out on their device but the attacker's copy is still valid. Server-side revocation (deleting the session from a store, or adding the JWT to a blocklist) is the only way to invalidate a compromised token. This is why stateful sessions (database-backed) are more revocable than stateless JWTs — a JWT is valid until expiry regardless of server state.

**Q: "A user is on tab A and logs out. Tab B is still open and showing authenticated content. What happens and how do you fix it?"**
A: Without a fix: Tab B continues showing private data indefinitely. Tab B's requests will fail with 401 when the token is gone, but the already-rendered content stays visible. The fix: on logout in Tab A, post a message via BroadcastChannel (or trigger a localStorage `storage` event) that Tab B listens for. Tab B receives the logout event and performs its own cleanup — clears state, redirects to login. This ensures logout is always global, not just per-tab.
