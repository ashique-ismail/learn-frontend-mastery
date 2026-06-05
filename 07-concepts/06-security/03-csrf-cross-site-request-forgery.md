# CSRF (Cross-Site Request Forgery)

## The Idea

**In plain English:** CSRF is a type of attack where a malicious website tricks your browser into secretly sending a request to another website you are already logged into — performing actions you never intended, like transferring money or changing your password. Your browser is the unwitting messenger carrying your login credentials to do the attacker's bidding.

**Real-world analogy:** Imagine you are at work and you left your employee badge on your desk. A sneaky co-worker grabs your badge, walks to the supply room, and signs out expensive equipment in your name — all while you are sitting at your desk totally unaware. The supply room trusts the badge, not the person holding it.
- The badge = your session cookie (the credential the website trusts)
- The sneaky co-worker = the malicious website (evil.com)
- The supply room = the legitimate website (bank.com) that accepts the request
- Signing out equipment in your name = the forged request (e.g., transferring money from your account)

---

## Table of Contents
- [Introduction](#introduction)
- [How CSRF Attacks Work](#how-csrf-attacks-work)
- [CSRF Attack Examples](#csrf-attack-examples)
- [CSRF Prevention Strategies](#csrf-prevention-strategies)
- [CSRF Tokens](#csrf-tokens)
- [SameSite Cookies](#samesite-cookies)
- [Double-Submit Cookie Pattern](#double-submit-cookie-pattern)
- [Implementation Examples](#implementation-examples)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use/Not to Use](#when-to-usenot-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

**Cross-Site Request Forgery (CSRF)** is an attack that forces an authenticated user to execute unwanted actions on a web application in which they're currently authenticated. CSRF attacks specifically target state-changing requests, not theft of data, since the attacker has no way to see the response to the forged request.

### Why CSRF is Dangerous

```
┌─────────────────────────────────────────────────────────────┐
│                    CSRF Attack Flow                          │
└─────────────────────────────────────────────────────────────┘

1. User logs into legitimate site (bank.com)
   ┌──────────┐                    ┌──────────┐
   │  User    │ ───── Login ─────> │ Bank.com │
   └──────────┘                    └──────────┘
        │                                │
        │ <───── Session Cookie ─────────┤
        │                                │

2. User visits malicious site while still logged in
   ┌──────────┐                    ┌──────────┐
   │  User    │ ───── Visit ─────> │ Evil.com │
   └──────────┘                    └──────────┘
        │                                │
        │ <───── Malicious HTML ─────────┤
        │                                │

3. Malicious site triggers request to bank.com
   ┌──────────┐                    ┌──────────┐
   │  User    │                    │ Bank.com │
   │ Browser  │ ── Transfer Money ─> │ (thinks │
   │          │  (with valid cookie)│ it's     │
   │          │                     │ legit!)  │
   └──────────┘                    └──────────┘
```

## How CSRF Attacks Work

CSRF exploits the trust that a site has in the user's browser. If a user is authenticated to a site, the browser automatically includes credentials (cookies, HTTP authentication) with every request to that site.

### Attack Prerequisites

1. **User must be authenticated** - The victim must have an active session with the target site
2. **Request must be forgeable** - The attacker must be able to construct a valid request
3. **No unpredictable parameters** - The request parameters must be known or guessable

### Attack Anatomy

```typescript
// Vulnerable endpoint (no CSRF protection)
@Post('/api/transfer')
async transferMoney(
  @Body() body: { to: string; amount: number },
  @Req() req: Request
) {
  // Session cookie automatically included by browser
  const userId = req.session.userId;
  
  // No CSRF validation - VULNERABLE!
  return this.bankService.transfer(userId, body.to, body.amount);
}
```

## CSRF Attack Examples

### Example 1: Hidden Form Auto-Submit

```html
<!-- Malicious page hosted on evil.com -->
<!DOCTYPE html>
<html>
<head>
  <title>You won a prize!</title>
</head>
<body>
  <h1>Congratulations! Click to claim your prize!</h1>
  
  <!-- Hidden form that submits automatically -->
  <form id="csrf-form" action="https://bank.com/api/transfer" method="POST">
    <input type="hidden" name="to" value="attacker-account-123" />
    <input type="hidden" name="amount" value="10000" />
  </form>
  
  <script>
    // Auto-submit when page loads
    document.getElementById('csrf-form').submit();
  </script>
</body>
</html>
```

### Example 2: Image Tag Attack

```html
<!-- Malicious page using GET request vulnerability -->
<!DOCTYPE html>
<html>
<body>
  <h1>Cute Cat Pictures!</h1>
  
  <!-- This triggers a GET request with cookies -->
  <img src="https://bank.com/api/transfer?to=attacker&amount=10000" 
       style="display:none" />
  
  <!-- Even more sneaky: 0x0 pixel -->
  <img src="https://social.com/api/follow?user=attacker" 
       width="0" height="0" />
</body>
</html>
```

### Example 3: AJAX/Fetch Attack

```html
<!-- Modern CSRF using fetch API -->
<!DOCTYPE html>
<html>
<body>
  <script>
    // This works if the API doesn't check Origin/Referer
    fetch('https://bank.com/api/transfer', {
      method: 'POST',
      credentials: 'include', // Include cookies
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        to: 'attacker-account',
        amount: 10000
      })
    });
  </script>
</body>
</html>
```

### Example 4: Link-Based Attack

```html
<!-- Social engineering attack -->
<!DOCTYPE html>
<html>
<body>
  <h1>Important Security Update!</h1>
  <p>Click here to verify your account:</p>
  
  <!-- GET request triggered by click -->
  <a href="https://bank.com/api/delete-account?confirm=yes">
    Verify Account
  </a>
</body>
</html>
```

## CSRF Prevention Strategies

```
┌─────────────────────────────────────────────────────────────┐
│              CSRF Defense Layers                             │
└─────────────────────────────────────────────────────────────┘

Layer 1: SameSite Cookies (Primary Defense)
   ├── Strict: No cross-site requests
   ├── Lax: Only safe HTTP methods (GET, HEAD, OPTIONS)
   └── None: All requests (requires Secure flag)

Layer 2: CSRF Tokens (Token-Based Defense)
   ├── Synchronizer Token Pattern
   ├── Double-Submit Cookie Pattern
   └── Encrypted Token Pattern

Layer 3: Origin/Referer Validation
   ├── Check Origin header
   ├── Check Referer header
   └── Reject mismatches

Layer 4: Custom Headers
   ├── Require custom header (X-Requested-With)
   └── CORS preflight for cross-origin

Layer 5: User Interaction
   ├── Re-authentication for sensitive actions
   ├── CAPTCHA challenges
   └── Confirmation dialogs
```

## CSRF Tokens

### Synchronizer Token Pattern

The most common CSRF defense mechanism.

```typescript
// Backend: Generate and validate CSRF tokens
import { randomBytes } from 'crypto';

class CSRFProtection {
  private tokens = new Map<string, { token: string; expires: number }>();

  generateToken(sessionId: string): string {
    const token = randomBytes(32).toString('hex');
    const expires = Date.now() + 3600000; // 1 hour
    
    this.tokens.set(sessionId, { token, expires });
    return token;
  }

  validateToken(sessionId: string, token: string): boolean {
    const stored = this.tokens.get(sessionId);
    
    if (!stored) {
      return false;
    }
    
    if (stored.expires < Date.now()) {
      this.tokens.delete(sessionId);
      return false;
    }
    
    return stored.token === token;
  }

  deleteToken(sessionId: string): void {
    this.tokens.delete(sessionId);
  }
}

// Express middleware
const csrfProtection = new CSRFProtection();

app.use((req, res, next) => {
  if (req.method === 'GET') {
    // Generate token for GET requests
    const token = csrfProtection.generateToken(req.sessionID);
    res.locals.csrfToken = token;
    next();
  } else {
    // Validate token for state-changing requests
    const token = req.headers['x-csrf-token'] || req.body._csrf;
    
    if (!csrfProtection.validateToken(req.sessionID, token)) {
      return res.status(403).json({ error: 'Invalid CSRF token' });
    }
    
    next();
  }
});
```

### React Implementation with CSRF Tokens

```typescript
// React: CSRF token management
import { createContext, useContext, useState, useEffect } from 'react';

interface CSRFContextType {
  token: string | null;
  refreshToken: () => Promise<void>;
}

const CSRFContext = createContext<CSRFContextType | undefined>(undefined);

export const CSRFProvider: React.FC<{ children: React.ReactNode }> = ({ 
  children 
}) => {
  const [token, setToken] = useState<string | null>(null);

  const fetchToken = async () => {
    try {
      const response = await fetch('/api/csrf-token', {
        credentials: 'include'
      });
      const data = await response.json();
      setToken(data.csrfToken);
    } catch (error) {
      console.error('Failed to fetch CSRF token:', error);
    }
  };

  useEffect(() => {
    fetchToken();
  }, []);

  return (
    <CSRFContext.Provider value={{ token, refreshToken: fetchToken }}>
      {children}
    </CSRFContext.Provider>
  );
};

export const useCSRF = () => {
  const context = useContext(CSRFContext);
  if (!context) {
    throw new Error('useCSRF must be used within CSRFProvider');
  }
  return context;
};

// Custom fetch hook with CSRF protection
export const useSecureFetch = () => {
  const { token, refreshToken } = useCSRF();

  const secureFetch = async (
    url: string, 
    options: RequestInit = {}
  ): Promise<Response> => {
    const headers = new Headers(options.headers);
    
    // Add CSRF token to all non-GET requests
    if (token && options.method !== 'GET') {
      headers.set('X-CSRF-Token', token);
    }

    try {
      const response = await fetch(url, {
        ...options,
        headers,
        credentials: 'include' // Include cookies
      });

      // If CSRF token is invalid, refresh and retry
      if (response.status === 403) {
        await refreshToken();
        headers.set('X-CSRF-Token', token!);
        return fetch(url, { ...options, headers, credentials: 'include' });
      }

      return response;
    } catch (error) {
      throw error;
    }
  };

  return secureFetch;
};

// Usage in component
const TransferMoney: React.FC = () => {
  const secureFetch = useSecureFetch();
  const [loading, setLoading] = useState(false);

  const handleTransfer = async (to: string, amount: number) => {
    setLoading(true);
    try {
      const response = await secureFetch('/api/transfer', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ to, amount })
      });

      if (response.ok) {
        alert('Transfer successful!');
      }
    } catch (error) {
      console.error('Transfer failed:', error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div>
      <button onClick={() => handleTransfer('account-123', 100)}>
        Transfer $100
      </button>
    </div>
  );
};
```

### Angular Implementation with CSRF Tokens

```typescript
// Angular: HTTP Interceptor for CSRF
import { Injectable } from '@angular/core';
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpErrorResponse
} from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, switchMap } from 'rxjs/operators';

@Injectable()
export class CsrfInterceptor implements HttpInterceptor {
  constructor(private csrfService: CsrfService) {}

  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    // Skip GET, HEAD, OPTIONS requests
    if (['GET', 'HEAD', 'OPTIONS'].includes(request.method)) {
      return next.handle(request);
    }

    // Add CSRF token to state-changing requests
    const token = this.csrfService.getToken();
    
    if (token) {
      request = request.clone({
        setHeaders: {
          'X-CSRF-Token': token
        },
        withCredentials: true
      });
    }

    return next.handle(request).pipe(
      catchError((error: HttpErrorResponse) => {
        // If CSRF token is invalid, refresh and retry
        if (error.status === 403 && error.error?.message === 'Invalid CSRF token') {
          return this.csrfService.refreshToken().pipe(
            switchMap(newToken => {
              const retryRequest = request.clone({
                setHeaders: {
                  'X-CSRF-Token': newToken
                }
              });
              return next.handle(retryRequest);
            })
          );
        }
        return throwError(() => error);
      })
    );
  }
}

// CSRF Service
@Injectable({
  providedIn: 'root'
})
export class CsrfService {
  private csrfToken: string | null = null;

  constructor(private http: HttpClient) {
    this.initToken();
  }

  private async initToken(): Promise<void> {
    try {
      const response = await this.http.get<{ csrfToken: string }>(
        '/api/csrf-token',
        { withCredentials: true }
      ).toPromise();
      this.csrfToken = response?.csrfToken || null;
    } catch (error) {
      console.error('Failed to initialize CSRF token:', error);
    }
  }

  getToken(): string | null {
    return this.csrfToken;
  }

  refreshToken(): Observable<string> {
    return this.http.get<{ csrfToken: string }>(
      '/api/csrf-token',
      { withCredentials: true }
    ).pipe(
      map(response => {
        this.csrfToken = response.csrfToken;
        return response.csrfToken;
      })
    );
  }
}

// Module configuration
@NgModule({
  providers: [
    {
      provide: HTTP_INTERCEPTORS,
      useClass: CsrfInterceptor,
      multi: true
    }
  ]
})
export class AppModule {}

// Usage in component
@Component({
  selector: 'app-transfer',
  template: `
    <button (click)="transfer()" [disabled]="loading">
      Transfer Money
    </button>
  `
})
export class TransferComponent {
  loading = false;

  constructor(private http: HttpClient) {}

  transfer(): void {
    this.loading = true;
    this.http.post('/api/transfer', {
      to: 'account-123',
      amount: 100
    }, {
      withCredentials: true
    }).subscribe({
      next: () => alert('Transfer successful!'),
      error: (error) => console.error('Transfer failed:', error),
      complete: () => this.loading = false
    });
  }
}
```

## SameSite Cookies

The `SameSite` cookie attribute is a powerful defense against CSRF attacks.

### SameSite Values

```typescript
// Backend: Setting SameSite cookies
import express from 'express';
import session from 'express-session';

const app = express();

// SameSite=Strict: Strongest protection
app.use(session({
  secret: 'your-secret-key',
  cookie: {
    httpOnly: true,
    secure: true, // HTTPS only
    sameSite: 'strict', // No cross-site requests at all
    maxAge: 3600000 // 1 hour
  }
}));

// SameSite=Lax: Balanced approach (DEFAULT in modern browsers)
app.use(session({
  secret: 'your-secret-key',
  cookie: {
    httpOnly: true,
    secure: true,
    sameSite: 'lax', // Allows safe cross-site navigation
    maxAge: 3600000
  }
}));

// SameSite=None: No CSRF protection (requires Secure)
app.use(session({
  secret: 'your-secret-key',
  cookie: {
    httpOnly: true,
    secure: true, // REQUIRED with SameSite=None
    sameSite: 'none', // Allow all cross-site requests
    maxAge: 3600000
  }
}));
```

### SameSite Behavior Comparison

```
┌─────────────────────────────────────────────────────────────┐
│           SameSite Cookie Behavior                           │
└─────────────────────────────────────────────────────────────┘

Scenario: User on evil.com, making request to bank.com

┌──────────────┬─────────┬──────┬───────┐
│   Action     │ Strict  │ Lax  │ None  │
├──────────────┼─────────┼──────┼───────┤
│ Link (GET)   │   ✗     │  ✓   │   ✓   │
│ Form (POST)  │   ✗     │  ✗   │   ✓   │
│ AJAX (POST)  │   ✗     │  ✗   │   ✓   │
│ Image src    │   ✗     │  ✗   │   ✓   │
│ iframe       │   ✗     │  ✗   │   ✓   │
└──────────────┴─────────┴──────┴───────┘

✓ = Cookie sent
✗ = Cookie NOT sent
```

### React: Checking SameSite Configuration

```typescript
// React: Component to test SameSite behavior
import { useState, useEffect } from 'react';

const CookieSecurityCheck: React.FC = () => {
  const [cookieInfo, setCookieInfo] = useState<{
    hasCookies: boolean;
    sameSiteSupport: boolean;
    cookies: string[];
  }>({
    hasCookies: false,
    sameSiteSupport: false,
    cookies: []
  });

  useEffect(() => {
    // Check if browser supports SameSite
    const sameSiteSupport = 'cookieStore' in window;
    
    // Parse cookies (limited by httpOnly flag)
    const cookies = document.cookie.split(';').map(c => c.trim());
    
    setCookieInfo({
      hasCookies: cookies.length > 0,
      sameSiteSupport,
      cookies
    });
  }, []);

  const testCrossSiteRequest = async () => {
    try {
      // This will fail with SameSite=Strict or Lax
      const response = await fetch('https://external-api.com/test', {
        method: 'POST',
        credentials: 'include'
      });
      console.log('Cross-site request succeeded:', response);
    } catch (error) {
      console.log('Cross-site request blocked:', error);
    }
  };

  return (
    <div>
      <h2>Cookie Security Status</h2>
      <p>SameSite Support: {cookieInfo.sameSiteSupport ? '✓' : '✗'}</p>
      <p>Has Cookies: {cookieInfo.hasCookies ? '✓' : '✗'}</p>
      <button onClick={testCrossSiteRequest}>
        Test Cross-Site Request
      </button>
    </div>
  );
};
```

## Double-Submit Cookie Pattern

An alternative to synchronizer tokens that doesn't require server-side session state.

```typescript
// Backend: Double-Submit Cookie implementation
import { randomBytes } from 'crypto';
import express from 'express';
import cookieParser from 'cookie-parser';

const app = express();
app.use(cookieParser());
app.use(express.json());

// Middleware to set CSRF cookie
app.use((req, res, next) => {
  if (!req.cookies.csrfToken) {
    const token = randomBytes(32).toString('hex');
    res.cookie('csrfToken', token, {
      httpOnly: false, // Must be readable by JavaScript
      secure: true,
      sameSite: 'strict',
      maxAge: 3600000
    });
  }
  next();
});

// Middleware to validate CSRF token
const validateCSRF = (req, res, next) => {
  if (['GET', 'HEAD', 'OPTIONS'].includes(req.method)) {
    return next();
  }

  const cookieToken = req.cookies.csrfToken;
  const headerToken = req.headers['x-csrf-token'];

  if (!cookieToken || !headerToken || cookieToken !== headerToken) {
    return res.status(403).json({ error: 'CSRF token validation failed' });
  }

  next();
};

app.use(validateCSRF);

// Protected endpoint
app.post('/api/transfer', (req, res) => {
  const { to, amount } = req.body;
  // Process transfer...
  res.json({ success: true });
});
```

### React: Double-Submit Cookie Client

```typescript
// React: Reading and submitting CSRF cookie
const getCSRFTokenFromCookie = (): string | null => {
  const match = document.cookie.match(/csrfToken=([^;]+)/);
  return match ? match[1] : null;
};

const DoubleSubmitExample: React.FC = () => {
  const handleSecureRequest = async () => {
    const csrfToken = getCSRFTokenFromCookie();
    
    if (!csrfToken) {
      console.error('CSRF token not found in cookies');
      return;
    }

    try {
      const response = await fetch('/api/transfer', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-CSRF-Token': csrfToken // Submit token in header
        },
        credentials: 'include', // Include cookies
        body: JSON.stringify({
          to: 'account-123',
          amount: 100
        })
      });

      if (response.ok) {
        console.log('Transfer successful!');
      }
    } catch (error) {
      console.error('Request failed:', error);
    }
  };

  return (
    <button onClick={handleSecureRequest}>
      Make Secure Transfer
    </button>
  );
};
```

### Angular: Double-Submit Cookie Implementation

```typescript
// Angular: HTTP Interceptor for double-submit
import { Injectable } from '@angular/core';
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent
} from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable()
export class DoubleSubmitInterceptor implements HttpInterceptor {
  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    // Skip safe methods
    if (['GET', 'HEAD', 'OPTIONS'].includes(request.method)) {
      return next.handle(request);
    }

    // Read CSRF token from cookie
    const csrfToken = this.getCookieValue('csrfToken');
    
    if (csrfToken) {
      // Add token to header
      request = request.clone({
        setHeaders: {
          'X-CSRF-Token': csrfToken
        },
        withCredentials: true
      });
    }

    return next.handle(request);
  }

  private getCookieValue(name: string): string | null {
    const match = document.cookie.match(
      new RegExp('(^|;\\s*)(' + name + ')=([^;]*)')
    );
    return match ? decodeURIComponent(match[3]) : null;
  }
}
```

## Common Mistakes

### 1. Using GET Requests for State Changes

```typescript
// ❌ BAD: GET request for state-changing operation
@Get('/delete-account')
deleteAccount(@Session() session) {
  return this.userService.deleteAccount(session.userId);
}

// ✓ GOOD: POST/DELETE for state changes
@Post('/delete-account')
@UseGuards(CsrfGuard)
deleteAccount(@Session() session) {
  return this.userService.deleteAccount(session.userId);
}
```

### 2. CSRF Token in URL

```typescript
// ❌ BAD: Token in URL (visible in logs, referrer headers)
fetch(`/api/transfer?csrf=${token}&to=account&amount=100`);

// ✓ GOOD: Token in header or body
fetch('/api/transfer', {
  method: 'POST',
  headers: {
    'X-CSRF-Token': token
  },
  body: JSON.stringify({ to: 'account', amount: 100 })
});
```

### 3. Validating Only Origin Header

```typescript
// ❌ BAD: Only checking Origin (can be missing)
const validateOrigin = (req, res, next) => {
  if (req.headers.origin !== 'https://myapp.com') {
    return res.status(403).send('Forbidden');
  }
  next();
};

// ✓ GOOD: Multiple validation layers
const validateRequest = (req, res, next) => {
  // Layer 1: SameSite cookies
  // Layer 2: CSRF token
  const token = req.headers['x-csrf-token'];
  if (!validateCSRFToken(req.session, token)) {
    return res.status(403).send('Invalid CSRF token');
  }
  
  // Layer 3: Origin/Referer check as backup
  const origin = req.headers.origin || req.headers.referer;
  if (origin && !origin.startsWith('https://myapp.com')) {
    return res.status(403).send('Invalid origin');
  }
  
  next();
};
```

### 4. Not Protecting API Endpoints

```typescript
// ❌ BAD: Assuming API endpoints are safe
app.post('/api/data', (req, res) => {
  // No CSRF protection
  processData(req.body);
});

// ✓ GOOD: Protect all state-changing endpoints
app.post('/api/data', csrfProtection, (req, res) => {
  processData(req.body);
});
```

### 5. SameSite=None Without Secure

```typescript
// ❌ BAD: SameSite=None without Secure flag
res.cookie('session', sessionId, {
  sameSite: 'none' // Rejected by browsers without Secure
});

// ✓ GOOD: SameSite=None with Secure
res.cookie('session', sessionId, {
  sameSite: 'none',
  secure: true, // HTTPS required
  httpOnly: true
});
```

## Best Practices

### 1. Defense in Depth

```typescript
// Implement multiple CSRF protections
class CSRFDefense {
  // Layer 1: SameSite cookies
  private setCookie(res: Response, name: string, value: string) {
    res.cookie(name, value, {
      httpOnly: true,
      secure: true,
      sameSite: 'strict'
    });
  }

  // Layer 2: CSRF token validation
  validateToken(req: Request): boolean {
    const sessionToken = req.session.csrfToken;
    const requestToken = req.headers['x-csrf-token'];
    return sessionToken === requestToken;
  }

  // Layer 3: Origin validation
  validateOrigin(req: Request): boolean {
    const origin = req.headers.origin;
    const allowedOrigins = ['https://myapp.com', 'https://app.myapp.com'];
    return origin ? allowedOrigins.includes(origin) : true;
  }

  // Layer 4: Custom request header
  validateCustomHeader(req: Request): boolean {
    return req.headers['x-requested-with'] === 'XMLHttpRequest';
  }

  // Combined validation
  isRequestSecure(req: Request): boolean {
    return (
      this.validateToken(req) &&
      this.validateOrigin(req) &&
      this.validateCustomHeader(req)
    );
  }
}
```

### 2. Use Framework Built-in Protection

```typescript
// Express with csurf middleware
import csrf from 'csurf';

const csrfProtection = csrf({ 
  cookie: {
    httpOnly: true,
    secure: true,
    sameSite: 'strict'
  }
});

app.use(csrfProtection);

app.get('/form', (req, res) => {
  res.render('form', { csrfToken: req.csrfToken() });
});

// Angular has built-in CSRF protection
@NgModule({
  imports: [
    HttpClientXsrfModule.withOptions({
      cookieName: 'XSRF-TOKEN',
      headerName: 'X-XSRF-TOKEN'
    })
  ]
})
export class AppModule {}
```

### 3. Token Rotation

```typescript
// Rotate CSRF tokens after sensitive operations
class TokenRotation {
  async performSensitiveAction(req: Request, res: Response) {
    // Validate current token
    if (!this.validateToken(req)) {
      return res.status(403).send('Invalid token');
    }

    // Perform action
    await this.sensitiveOperation(req.body);

    // Generate new token
    const newToken = this.generateToken();
    req.session.csrfToken = newToken;

    res.json({ 
      success: true, 
      csrfToken: newToken // Return new token
    });
  }
}
```

### 4. Logging and Monitoring

```typescript
// Monitor CSRF attempts
class CSRFMonitoring {
  logCSRFAttempt(req: Request, reason: string) {
    console.error('CSRF attempt detected:', {
      timestamp: new Date().toISOString(),
      ip: req.ip,
      userAgent: req.headers['user-agent'],
      url: req.url,
      method: req.method,
      origin: req.headers.origin,
      referer: req.headers.referer,
      reason
    });

    // Alert security team for repeated attempts
    this.checkForRepeatedAttempts(req.ip);
  }

  private async checkForRepeatedAttempts(ip: string) {
    const attempts = await this.getRecentAttempts(ip);
    if (attempts > 5) {
      this.alertSecurityTeam(ip);
      this.blockIP(ip);
    }
  }
}
```

## When to Use/Not to Use

### When to Use CSRF Protection

1. **Authentication-based applications** - Any app with user sessions
2. **State-changing operations** - POST, PUT, DELETE, PATCH requests
3. **Financial transactions** - Banking, e-commerce, payment processing
4. **Account management** - Password changes, profile updates, account deletion
5. **Social actions** - Posting, following, liking, commenting

### When CSRF Protection May Not Be Needed

1. **Stateless APIs with token authentication** - JWT in Authorization header (not cookies)
2. **Read-only operations** - GET requests that don't change state
3. **Public APIs** - Endpoints designed for cross-origin access
4. **GraphQL with proper authentication** - If using header-based auth

### CSRF Protection Decision Tree

```
┌─────────────────────────────────────────────────────────────┐
│          Does the endpoint change state?                     │
│                                                              │
│                 Yes                    No                    │
│                  │                      │                    │
│                  ▼                      ▼                    │
│     Does it use cookie-based      CSRF not needed           │
│     authentication?                                          │
│                  │                                           │
│         Yes             No                                   │
│          │               │                                   │
│          ▼               ▼                                   │
│   IMPLEMENT CSRF    Header-based auth                        │
│   PROTECTION        (probably safe, but                      │
│                     validate Origin)                         │
└─────────────────────────────────────────────────────────────┘
```

## Interview Questions

### Q1: What is CSRF and how does it differ from XSS?

**Answer:** CSRF (Cross-Site Request Forgery) is an attack that forces an authenticated user to execute unwanted actions on a web application where they're authenticated. It exploits the trust that a site has in the user's browser.

**Key differences from XSS:**
- **CSRF**: Exploits trust in user's browser, performs actions as the user
- **XSS**: Exploits trust in the server, injects malicious code

CSRF doesn't steal data (attacker can't see response), while XSS can steal sensitive information. CSRF requires the user to be authenticated, XSS doesn't necessarily require authentication.

### Q2: Explain how SameSite cookies prevent CSRF attacks.

**Answer:** SameSite cookies prevent CSRF by controlling when cookies are sent in cross-site requests:

- **SameSite=Strict**: Cookies never sent in cross-site requests. Most secure but breaks some legitimate flows (OAuth redirects, payment gateways).
- **SameSite=Lax**: Cookies sent only for "safe" cross-site requests (GET from top-level navigation). Balances security and usability. Default in modern browsers.
- **SameSite=None**: Cookies sent in all cross-site requests. Requires Secure flag. Used for legitimate cross-site scenarios (embedded content, API calls).

SameSite=Strict or Lax prevents attackers from making authenticated requests from malicious sites because the session cookie won't be included.

### Q3: What is the double-submit cookie pattern?

**Answer:** Double-submit cookie pattern is a CSRF defense where:
1. Server sets a random token in a cookie (NOT httpOnly)
2. Client reads the token from cookie and sends it in request header/body
3. Server validates that cookie value matches header/body value

**Advantages:**
- No server-side session state needed
- Stateless, scales well

**How it works:** Attacker can't read cookies from victim's browser (Same-Origin Policy), so they can't get the token to submit in the header. The attack fails because cookie value won't match header value.

**Limitation:** Vulnerable if attacker can set cookies for your domain (subdomain attacks, cookie tossing).

### Q4: Why is it important to use POST instead of GET for state-changing operations?

**Answer:** State-changing operations should use POST/PUT/DELETE because:

1. **CSRF Protection**: SameSite=Lax allows GET in cross-site navigation, but blocks POST
2. **Browser Behavior**: GET can be triggered by images, links, browser prefetch - POST requires form submission or JavaScript
3. **Semantic Correctness**: HTTP semantics define GET as safe and idempotent
4. **Log Exposure**: GET parameters appear in URLs, which are logged everywhere (proxy logs, browser history, referrer headers)
5. **Caching**: GET responses may be cached, exposing sensitive operations

Example: `<img src="https://bank.com/transfer?to=attacker&amount=1000">` works if transfer uses GET, but not with POST.

### Q5: How do you protect against CSRF in a single-page application (SPA)?

**Answer:** CSRF protection in SPAs:

**1. Use custom headers:**
```typescript
fetch('/api/data', {
  method: 'POST',
  headers: {
    'X-Requested-With': 'XMLHttpRequest',
    'X-CSRF-Token': token
  }
});
```
Cross-origin requests with custom headers trigger CORS preflight, which attackers can't force.

**2. Implement CSRF token exchange:**
- Fetch token on app initialization
- Include in all state-changing requests
- Refresh periodically

**3. Use SameSite cookies:**
- Set SameSite=Strict or Lax for session cookies
- Prevents cross-site request forgery automatically

**4. Validate Origin/Referer headers:**
- Server-side validation as additional layer
- Reject requests from unexpected origins

**Best practice:** Combine multiple approaches (defense in depth).

### Q6: What are the security implications of SameSite=None?

**Answer:** SameSite=None security implications:

**Risks:**
- No CSRF protection from SameSite mechanism
- Cookies sent on all cross-site requests
- Increases attack surface

**Requirements:**
- MUST use Secure flag (HTTPS only)
- Should implement alternative CSRF protections (tokens, Origin validation)

**When necessary:**
- Cross-site embedded content (iframes)
- Third-party integrations
- OAuth/SAML flows
- Cross-domain API calls

**Mitigation:**
```typescript
res.cookie('session', value, {
  sameSite: 'none',
  secure: true, // Required
  httpOnly: true,
  // Add CSRF token validation
});
```

Always combine with CSRF tokens and Origin validation when using SameSite=None.

### Q7: How do you handle CSRF protection for mobile apps?

**Answer:** Mobile apps typically don't need traditional CSRF protection because:

**Why different:**
- Don't use cookie-based sessions (use tokens in headers)
- Can't be tricked by malicious websites
- Attacker can't force user's app to make requests

**Recommended approach:**
```typescript
// Use token-based authentication
const token = await getStoredToken();

fetch('https://api.example.com/data', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify(data)
});
```

**Still implement:**
- Origin/Referer validation (prevent direct API abuse)
- Rate limiting (prevent brute force)
- Certificate pinning (prevent MITM)
- Token expiration and refresh

CSRF protection is needed if mobile app uses WebViews with cookie-based auth.

### Q8: Explain a scenario where Origin and Referer headers could be missing and how to handle it.

**Answer:** Headers can be missing in:

**Scenarios:**
1. **Privacy-focused browsers** - Block Referer header
2. **Browser extensions** - Strip headers
3. **Corporate proxies** - Remove headers
4. **HTTPS→HTTP transition** - Referer stripped (by spec)
5. **Direct navigation** - No Referer (typed URL, bookmark)

**Handling strategy:**
```typescript
const validateRequest = (req) => {
  const origin = req.headers.origin;
  const referer = req.headers.referer;
  
  // If headers present, validate them
  if (origin || referer) {
    const source = origin || referer;
    if (!isAllowedSource(source)) {
      return false; // Reject suspicious origin
    }
  }
  
  // If headers missing, rely on CSRF token
  // Never reject based solely on missing headers
  return validateCSRFToken(req);
};
```

**Best practice:** Use Origin/Referer as additional security layer, not primary defense. Always implement CSRF tokens as main protection.

## Key Takeaways

1. **CSRF exploits browser trust** - Attackers leverage automatic cookie inclusion in cross-site requests to forge authenticated requests
2. **Use SameSite cookies** - SameSite=Strict or Lax provides strong CSRF protection with minimal implementation effort
3. **Implement CSRF tokens** - Synchronizer token pattern or double-submit cookies add critical defense layer
4. **Never use GET for state changes** - POST/PUT/DELETE prevent simple CSRF attacks via images, links, and browser prefetch
5. **Defense in depth** - Combine multiple protections: SameSite cookies + CSRF tokens + Origin validation + custom headers
6. **Token-based auth is safer** - JWT in Authorization headers (not cookies) naturally resistant to CSRF attacks
7. **Validate Origin and Referer** - Secondary defense but don't rely solely on these headers as they can be missing
8. **SameSite=None requires Secure** - Only use with HTTPS and implement additional CSRF protections
9. **Protect all state-changing endpoints** - API endpoints, form submissions, AJAX requests all need protection
10. **Monitor and log CSRF attempts** - Detect attack patterns, block malicious IPs, alert security team for investigation

## Resources

### Official Documentation
- [OWASP CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [MDN: SameSite Cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite)
- [RFC 6749: The OAuth 2.0 Authorization Framework](https://tools.ietf.org/html/rfc6749)

### Libraries and Tools
- [csurf (Express CSRF middleware)](https://github.com/expressjs/csurf)
- [Django CSRF Protection](https://docs.djangoproject.com/en/stable/ref/csrf/)
- [Angular CSRF Protection](https://angular.io/guide/http#security-xsrf-protection)

### Articles and Tutorials
- [Understanding CSRF Attacks](https://www.acunetix.com/websitesecurity/csrf-attacks/)
- [SameSite Cookie Changes](https://web.dev/samesite-cookies-explained/)
- [Token-Based Authentication Best Practices](https://auth0.com/blog/token-based-authentication-made-easy/)

### Testing Tools
- [Burp Suite CSRF PoC Generator](https://portswigger.net/burp/documentation/desktop/tools/csrf-poc-generator)
- [OWASP ZAP](https://www.zaproxy.org/)
- [CSP Evaluator](https://csp-evaluator.withgoogle.com/)
