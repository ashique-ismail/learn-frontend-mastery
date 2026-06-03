# Authentication Strategies

## Overview

Authentication is the process of verifying the identity of a user or system. Modern web applications use various authentication strategies, each with different security characteristics, use cases, and trade-offs. The most common strategies include JWT (JSON Web Tokens), session-based authentication, OAuth 2.0, and refresh token mechanisms.

## Authentication Flow Comparison

```
Session-Based Authentication:
┌────────┐                           ┌────────┐
│ Client │                           │ Server │
└───┬────┘                           └───┬────┘
    │ 1. Login (username/password)      │
    │──────────────────────────────────>│
    │                                    │ Create session
    │ 2. Session ID (cookie)             │ Store in database
    │<──────────────────────────────────│
    │                                    │
    │ 3. Request + Session ID            │
    │──────────────────────────────────>│
    │                                    │ Lookup session
    │ 4. Response                        │ Validate
    │<──────────────────────────────────│

JWT Authentication:
┌────────┐                           ┌────────┐
│ Client │                           │ Server │
└───┬────┘                           └───┬────┘
    │ 1. Login (username/password)      │
    │──────────────────────────────────>│
    │                                    │ Generate JWT
    │ 2. JWT Token                       │ Sign token
    │<──────────────────────────────────│
    │                                    │
    │ 3. Request + JWT (Authorization)   │
    │──────────────────────────────────>│
    │                                    │ Verify signature
    │ 4. Response                        │ Decode payload
    │<──────────────────────────────────│
```

## JWT (JSON Web Tokens)

JWT is a compact, self-contained token format that encodes user information and is signed to prevent tampering.

### JWT Structure

```
JWT Token Structure:
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Header  │.│ Payload  │.│ Signature│
└──────────┘  └──────────┘  └──────────┘

Header (Base64):
{
  "alg": "HS256",
  "typ": "JWT"
}

Payload (Base64):
{
  "sub": "user123",
  "name": "John Doe",
  "iat": 1516239022,
  "exp": 1516242622,
  "roles": ["user", "admin"]
}

Signature:
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret
)
```

### JWT Implementation (React)

```typescript
// services/auth.service.ts
import jwtDecode from 'jwt-decode';

interface JWTPayload {
  sub: string;
  name: string;
  email: string;
  roles: string[];
  iat: number;
  exp: number;
}

class AuthService {
  private readonly TOKEN_KEY = 'access_token';
  private readonly REFRESH_TOKEN_KEY = 'refresh_token';

  async login(email: string, password: string): Promise<void> {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });

    if (!response.ok) {
      throw new Error('Login failed');
    }

    const { accessToken, refreshToken } = await response.json();
    
    this.setAccessToken(accessToken);
    this.setRefreshToken(refreshToken);
  }

  logout(): void {
    localStorage.removeItem(this.TOKEN_KEY);
    localStorage.removeItem(this.REFRESH_TOKEN_KEY);
  }

  setAccessToken(token: string): void {
    localStorage.setItem(this.TOKEN_KEY, token);
  }

  getAccessToken(): string | null {
    return localStorage.getItem(this.TOKEN_KEY);
  }

  setRefreshToken(token: string): void {
    localStorage.setItem(this.REFRESH_TOKEN_KEY, token);
  }

  getRefreshToken(): string | null {
    return localStorage.getItem(this.REFRESH_TOKEN_KEY);
  }

  decodeToken(token: string): JWTPayload | null {
    try {
      return jwtDecode<JWTPayload>(token);
    } catch (error) {
      console.error('Failed to decode token:', error);
      return null;
    }
  }

  isTokenExpired(token: string): boolean {
    const decoded = this.decodeToken(token);
    if (!decoded) return true;

    const now = Date.now() / 1000;
    return decoded.exp < now;
  }

  isAuthenticated(): boolean {
    const token = this.getAccessToken();
    if (!token) return false;

    return !this.isTokenExpired(token);
  }

  getCurrentUser(): JWTPayload | null {
    const token = this.getAccessToken();
    if (!token) return null;

    return this.decodeToken(token);
  }

  hasRole(role: string): boolean {
    const user = this.getCurrentUser();
    return user?.roles.includes(role) ?? false;
  }

  async refreshAccessToken(): Promise<string | null> {
    const refreshToken = this.getRefreshToken();
    if (!refreshToken) return null;

    try {
      const response = await fetch('/api/auth/refresh', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ refreshToken })
      });

      if (!response.ok) {
        this.logout();
        return null;
      }

      const { accessToken } = await response.json();
      this.setAccessToken(accessToken);
      return accessToken;
    } catch (error) {
      console.error('Token refresh failed:', error);
      this.logout();
      return null;
    }
  }
}

export const authService = new AuthService();
```

```typescript
// hooks/useAuth.ts
import { useState, useEffect, useCallback, createContext, useContext } from 'react';
import { authService } from '../services/auth.service';

interface User {
  id: string;
  name: string;
  email: string;
  roles: string[];
}

interface AuthContextType {
  user: User | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  hasRole: (role: string) => boolean;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export const AuthProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    const initAuth = () => {
      const currentUser = authService.getCurrentUser();
      if (currentUser) {
        setUser({
          id: currentUser.sub,
          name: currentUser.name,
          email: currentUser.email,
          roles: currentUser.roles
        });
      }
      setIsLoading(false);
    };

    initAuth();
  }, []);

  const login = useCallback(async (email: string, password: string) => {
    await authService.login(email, password);
    const currentUser = authService.getCurrentUser();
    if (currentUser) {
      setUser({
        id: currentUser.sub,
        name: currentUser.name,
        email: currentUser.email,
        roles: currentUser.roles
      });
    }
  }, []);

  const logout = useCallback(() => {
    authService.logout();
    setUser(null);
  }, []);

  const hasRole = useCallback((role: string) => {
    return authService.hasRole(role);
  }, []);

  return (
    <AuthContext.Provider
      value={{
        user,
        isAuthenticated: !!user,
        isLoading,
        login,
        logout,
        hasRole
      }}
    >
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
};
```

```typescript
// components/LoginForm.tsx
import React, { useState } from 'react';
import { useAuth } from '../hooks/useAuth';
import { useNavigate } from 'react-router-dom';

const LoginForm: React.FC = () => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  
  const { login } = useAuth();
  const navigate = useNavigate();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError('');
    setIsLoading(true);

    try {
      await login(email, password);
      navigate('/dashboard');
    } catch (err) {
      setError('Invalid email or password');
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <h2>Login</h2>
      
      {error && <div className="error">{error}</div>}
      
      <div>
        <label htmlFor="email">Email</label>
        <input
          id="email"
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          required
        />
      </div>

      <div>
        <label htmlFor="password">Password</label>
        <input
          id="password"
          type="password"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          required
        />
      </div>

      <button type="submit" disabled={isLoading}>
        {isLoading ? 'Logging in...' : 'Login'}
      </button>
    </form>
  );
};

export default LoginForm;
```

### HTTP Interceptor with JWT (React)

```typescript
// utils/http-interceptor.ts
import { authService } from '../services/auth.service';

export const fetchWithAuth = async (
  url: string,
  options: RequestInit = {}
): Promise<Response> => {
  // Add access token to headers
  const token = authService.getAccessToken();
  const headers = {
    ...options.headers,
    ...(token && { Authorization: `Bearer ${token}` })
  };

  let response = await fetch(url, { ...options, headers });

  // Handle 401 - try to refresh token
  if (response.status === 401) {
    const newToken = await authService.refreshAccessToken();
    
    if (newToken) {
      // Retry request with new token
      response = await fetch(url, {
        ...options,
        headers: {
          ...options.headers,
          Authorization: `Bearer ${newToken}`
        }
      });
    } else {
      // Refresh failed, redirect to login
      window.location.href = '/login';
    }
  }

  return response;
};
```

```typescript
// hooks/useApi.ts
import { useState, useCallback } from 'react';
import { fetchWithAuth } from '../utils/http-interceptor';

export const useApi = <T>() => {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  const execute = useCallback(async (url: string, options?: RequestInit) => {
    setLoading(true);
    setError(null);

    try {
      const response = await fetchWithAuth(url, options);
      
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      const result = await response.json();
      setData(result);
      return result;
    } catch (err) {
      const error = err instanceof Error ? err : new Error('Unknown error');
      setError(error);
      throw error;
    } finally {
      setLoading(false);
    }
  }, []);

  return { data, loading, error, execute };
};
```

## JWT (Angular)

```typescript
// services/auth.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { BehaviorSubject, Observable } from 'rxjs';
import { map, tap } from 'rxjs/operators';
import jwtDecode from 'jwt-decode';

interface JWTPayload {
  sub: string;
  name: string;
  email: string;
  roles: string[];
  exp: number;
}

interface LoginResponse {
  accessToken: string;
  refreshToken: string;
}

@Injectable({
  providedIn: 'root'
})
export class AuthService {
  private readonly TOKEN_KEY = 'access_token';
  private readonly REFRESH_TOKEN_KEY = 'refresh_token';
  private currentUserSubject = new BehaviorSubject<JWTPayload | null>(null);
  public currentUser$ = this.currentUserSubject.asObservable();

  constructor(private http: HttpClient) {
    this.initializeAuth();
  }

  private initializeAuth() {
    const token = this.getAccessToken();
    if (token && !this.isTokenExpired(token)) {
      const user = this.decodeToken(token);
      this.currentUserSubject.next(user);
    }
  }

  login(email: string, password: string): Observable<void> {
    return this.http.post<LoginResponse>('/api/auth/login', { email, password })
      .pipe(
        tap(response => {
          this.setAccessToken(response.accessToken);
          this.setRefreshToken(response.refreshToken);
          
          const user = this.decodeToken(response.accessToken);
          this.currentUserSubject.next(user);
        }),
        map(() => void 0)
      );
  }

  logout() {
    localStorage.removeItem(this.TOKEN_KEY);
    localStorage.removeItem(this.REFRESH_TOKEN_KEY);
    this.currentUserSubject.next(null);
  }

  getAccessToken(): string | null {
    return localStorage.getItem(this.TOKEN_KEY);
  }

  setAccessToken(token: string) {
    localStorage.setItem(this.TOKEN_KEY, token);
  }

  getRefreshToken(): string | null {
    return localStorage.getItem(this.REFRESH_TOKEN_KEY);
  }

  setRefreshToken(token: string) {
    localStorage.setItem(this.REFRESH_TOKEN_KEY, token);
  }

  decodeToken(token: string): JWTPayload | null {
    try {
      return jwtDecode<JWTPayload>(token);
    } catch {
      return null;
    }
  }

  isTokenExpired(token: string): boolean {
    const decoded = this.decodeToken(token);
    if (!decoded) return true;

    return decoded.exp < Date.now() / 1000;
  }

  isAuthenticated(): boolean {
    const token = this.getAccessToken();
    return token ? !this.isTokenExpired(token) : false;
  }

  hasRole(role: string): boolean {
    const user = this.currentUserSubject.value;
    return user?.roles.includes(role) ?? false;
  }

  refreshAccessToken(): Observable<string> {
    const refreshToken = this.getRefreshToken();
    
    return this.http.post<{ accessToken: string }>('/api/auth/refresh', { refreshToken })
      .pipe(
        tap(response => {
          this.setAccessToken(response.accessToken);
        }),
        map(response => response.accessToken)
      );
  }
}
```

```typescript
// interceptors/auth.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpErrorResponse
} from '@angular/common/http';
import { Observable, throwError, BehaviorSubject } from 'rxjs';
import { catchError, switchMap, filter, take } from 'rxjs/operators';
import { AuthService } from '../services/auth.service';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  private isRefreshing = false;
  private refreshTokenSubject = new BehaviorSubject<string | null>(null);

  constructor(private authService: AuthService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // Add token to request
    const token = this.authService.getAccessToken();
    if (token) {
      req = this.addToken(req, token);
    }

    return next.handle(req).pipe(
      catchError(error => {
        if (error instanceof HttpErrorResponse && error.status === 401) {
          return this.handle401Error(req, next);
        }
        return throwError(() => error);
      })
    );
  }

  private addToken(req: HttpRequest<any>, token: string): HttpRequest<any> {
    return req.clone({
      setHeaders: {
        Authorization: `Bearer ${token}`
      }
    });
  }

  private handle401Error(
    req: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    if (!this.isRefreshing) {
      this.isRefreshing = true;
      this.refreshTokenSubject.next(null);

      return this.authService.refreshAccessToken().pipe(
        switchMap(token => {
          this.isRefreshing = false;
          this.refreshTokenSubject.next(token);
          return next.handle(this.addToken(req, token));
        }),
        catchError(error => {
          this.isRefreshing = false;
          this.authService.logout();
          return throwError(() => error);
        })
      );
    } else {
      // Wait for token refresh to complete
      return this.refreshTokenSubject.pipe(
        filter(token => token !== null),
        take(1),
        switchMap(token => next.handle(this.addToken(req, token!)))
      );
    }
  }
}
```

```typescript
// guards/auth.guard.ts
import { Injectable } from '@angular/core';
import { CanActivate, Router, ActivatedRouteSnapshot } from '@angular/router';
import { AuthService } from '../services/auth.service';

@Injectable({
  providedIn: 'root'
})
export class AuthGuard implements CanActivate {
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}

  canActivate(route: ActivatedRouteSnapshot): boolean {
    if (!this.authService.isAuthenticated()) {
      this.router.navigate(['/login']);
      return false;
    }

    // Check role-based access
    const requiredRoles = route.data['roles'] as string[];
    if (requiredRoles) {
      const hasRole = requiredRoles.some(role => this.authService.hasRole(role));
      if (!hasRole) {
        this.router.navigate(['/unauthorized']);
        return false;
      }
    }

    return true;
  }
}
```

## Session-Based Authentication

```typescript
// server/session-auth-server.ts
import express from 'express';
import session from 'express-session';
import RedisStore from 'connect-redis';
import Redis from 'ioredis';
import bcrypt from 'bcrypt';

const app = express();
const redis = new Redis();

app.use(express.json());

// Session configuration
app.use(
  session({
    store: new RedisStore({ client: redis }),
    secret: process.env.SESSION_SECRET!,
    resave: false,
    saveUninitialized: false,
    cookie: {
      secure: process.env.NODE_ENV === 'production', // HTTPS only in production
      httpOnly: true, // Prevent XSS
      maxAge: 24 * 60 * 60 * 1000, // 24 hours
      sameSite: 'strict' // CSRF protection
    }
  })
);

// Extend session type
declare module 'express-session' {
  interface SessionData {
    userId: string;
    email: string;
  }
}

// Login endpoint
app.post('/api/auth/login', async (req, res) => {
  const { email, password } = req.body;

  // Find user in database
  const user = await findUserByEmail(email);
  if (!user) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // Verify password
  const valid = await bcrypt.compare(password, user.passwordHash);
  if (!valid) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // Create session
  req.session.userId = user.id;
  req.session.email = user.email;

  res.json({ user: { id: user.id, email: user.email, name: user.name } });
});

// Logout endpoint
app.post('/api/auth/logout', (req, res) => {
  req.session.destroy((err) => {
    if (err) {
      return res.status(500).json({ error: 'Logout failed' });
    }
    res.clearCookie('connect.sid');
    res.json({ message: 'Logged out' });
  });
});

// Protected route
app.get('/api/user/profile', (req, res) => {
  if (!req.session.userId) {
    return res.status(401).json({ error: 'Not authenticated' });
  }

  // Return user data
  res.json({ userId: req.session.userId, email: req.session.email });
});

// Auth middleware
const requireAuth = (req: express.Request, res: express.Response, next: express.NextFunction) => {
  if (!req.session.userId) {
    return res.status(401).json({ error: 'Not authenticated' });
  }
  next();
};

app.get('/api/protected', requireAuth, (req, res) => {
  res.json({ message: 'Protected data' });
});
```

## OAuth 2.0

```typescript
// services/oauth.service.ts
class OAuth2Service {
  private readonly CLIENT_ID = process.env.REACT_APP_OAUTH_CLIENT_ID!;
  private readonly REDIRECT_URI = process.env.REACT_APP_OAUTH_REDIRECT_URI!;
  private readonly AUTHORIZATION_ENDPOINT = 'https://oauth.provider.com/authorize';
  private readonly TOKEN_ENDPOINT = 'https://oauth.provider.com/token';

  // Step 1: Redirect to authorization server
  initiateAuthFlow() {
    const state = this.generateRandomState();
    const codeVerifier = this.generateCodeVerifier();
    const codeChallenge = this.generateCodeChallenge(codeVerifier);

    // Store for later
    sessionStorage.setItem('oauth_state', state);
    sessionStorage.setItem('code_verifier', codeVerifier);

    const params = new URLSearchParams({
      response_type: 'code',
      client_id: this.CLIENT_ID,
      redirect_uri: this.REDIRECT_URI,
      scope: 'openid profile email',
      state,
      code_challenge: codeChallenge,
      code_challenge_method: 'S256'
    });

    window.location.href = `${this.AUTHORIZATION_ENDPOINT}?${params}`;
  }

  // Step 2: Handle callback with authorization code
  async handleCallback(code: string, state: string): Promise<void> {
    const storedState = sessionStorage.getItem('oauth_state');
    const codeVerifier = sessionStorage.getItem('code_verifier');

    if (state !== storedState) {
      throw new Error('Invalid state parameter');
    }

    // Exchange code for tokens
    const response = await fetch(this.TOKEN_ENDPOINT, {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: new URLSearchParams({
        grant_type: 'authorization_code',
        code,
        redirect_uri: this.REDIRECT_URI,
        client_id: this.CLIENT_ID,
        code_verifier: codeVerifier!
      })
    });

    const tokens = await response.json();
    
    // Store tokens
    localStorage.setItem('access_token', tokens.access_token);
    localStorage.setItem('refresh_token', tokens.refresh_token);
    
    // Clean up
    sessionStorage.removeItem('oauth_state');
    sessionStorage.removeItem('code_verifier');
  }

  private generateRandomState(): string {
    const array = new Uint8Array(32);
    crypto.getRandomValues(array);
    return Array.from(array, byte => byte.toString(16).padStart(2, '0')).join('');
  }

  private generateCodeVerifier(): string {
    const array = new Uint8Array(32);
    crypto.getRandomValues(array);
    return btoa(String.fromCharCode(...array))
      .replace(/\+/g, '-')
      .replace(/\//g, '_')
      .replace(/=/g, '');
  }

  private async generateCodeChallenge(verifier: string): Promise<string> {
    const encoder = new TextEncoder();
    const data = encoder.encode(verifier);
    const hash = await crypto.subtle.digest('SHA-256', data);
    return btoa(String.fromCharCode(...new Uint8Array(hash)))
      .replace(/\+/g, '-')
      .replace(/\//g, '_')
      .replace(/=/g, '');
  }
}

export const oauthService = new OAuth2Service();
```

## Refresh Token Rotation

```typescript
// services/token-rotation.service.ts
class TokenRotationService {
  private refreshPromise: Promise<string> | null = null;

  async getValidAccessToken(): Promise<string> {
    const accessToken = localStorage.getItem('access_token');
    
    if (!accessToken || this.isTokenExpired(accessToken)) {
      return this.refreshAccessToken();
    }

    return accessToken;
  }

  private async refreshAccessToken(): Promise<string> {
    // Prevent multiple simultaneous refresh requests
    if (this.refreshPromise) {
      return this.refreshPromise;
    }

    this.refreshPromise = this.performTokenRefresh();

    try {
      const newToken = await this.refreshPromise;
      return newToken;
    } finally {
      this.refreshPromise = null;
    }
  }

  private async performTokenRefresh(): Promise<string> {
    const refreshToken = localStorage.getItem('refresh_token');
    
    if (!refreshToken) {
      throw new Error('No refresh token available');
    }

    const response = await fetch('/api/auth/refresh', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ refreshToken })
    });

    if (!response.ok) {
      localStorage.removeItem('access_token');
      localStorage.removeItem('refresh_token');
      window.location.href = '/login';
      throw new Error('Token refresh failed');
    }

    const { accessToken, refreshToken: newRefreshToken } = await response.json();
    
    // Store new tokens (refresh token rotation)
    localStorage.setItem('access_token', accessToken);
    localStorage.setItem('refresh_token', newRefreshToken);

    return accessToken;
  }

  private isTokenExpired(token: string): boolean {
    try {
      const payload = JSON.parse(atob(token.split('.')[1]));
      return payload.exp < Date.now() / 1000;
    } catch {
      return true;
    }
  }
}

export const tokenRotationService = new TokenRotationService();
```

## Common Mistakes

### Mistake 1: Storing JWT in localStorage (XSS Vulnerability)

```typescript
// BAD: localStorage is vulnerable to XSS
localStorage.setItem('token', jwt);

// BETTER: Use httpOnly cookies (server-side)
res.cookie('token', jwt, {
  httpOnly: true,
  secure: true,
  sameSite: 'strict'
});

// BEST: Combine short-lived JWT in memory + refresh token in httpOnly cookie
```

### Mistake 2: Not Implementing Token Expiration

```typescript
// BAD: Tokens never expire
const token = jwt.sign({ userId: user.id }, secret);

// GOOD: Set expiration
const token = jwt.sign(
  { userId: user.id },
  secret,
  { expiresIn: '15m' }
);
```

### Mistake 3: Not Validating JWT Signature

```typescript
// BAD: Trusting JWT without verification
const payload = JSON.parse(atob(token.split('.')[1]));

// GOOD: Always verify signature
const payload = jwt.verify(token, secret);
```

## Best Practices

1. Use short-lived access tokens (15 minutes) with refresh tokens
2. Store tokens in httpOnly cookies to prevent XSS
3. Implement CSRF protection with SameSite cookies
4. Use HTTPS in production
5. Implement token rotation for refresh tokens
6. Add rate limiting to authentication endpoints
7. Use strong secrets and rotate them regularly
8. Implement proper password hashing (bcrypt, argon2)
9. Add account lockout after failed attempts
10. Log authentication events for security auditing

## Key Takeaways

1. JWT is stateless and scales well across servers
2. Sessions require server-side storage but are more secure
3. OAuth 2.0 enables third-party authentication
4. Refresh tokens should rotate on each use
5. Always use HTTPS for authentication
6. httpOnly cookies prevent XSS attacks
7. Implement proper token expiration
8. Use PKCE for OAuth in public clients

## Interview Questions

1. What is the difference between authentication and authorization?
2. Explain how JWT works and its structure
3. What are the security concerns with storing JWT in localStorage?
4. How does refresh token rotation work?
5. What is OAuth 2.0 and when would you use it?
6. Explain the difference between session-based and token-based auth
7. What is PKCE and why is it important?
8. How do you prevent CSRF attacks?
9. What is the difference between httpOnly and secure cookies?
10. How do you handle token expiration in SPAs?

## Resources

- JWT.io Documentation
- OAuth 2.0 Specification
- OWASP Authentication Cheat Sheet
- RFC 7519 (JWT)
- Refresh Token Best Practices
- PKCE RFC 7636
