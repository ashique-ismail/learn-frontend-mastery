# CORS (Cross-Origin Resource Sharing) In-Depth

## The Idea

**In plain English:** CORS is a security rule built into web browsers that controls which websites are allowed to request data from a different website. Without it, any random website could secretly pull your private information from another site you are logged into.

**Real-world analogy:** Imagine a bank (the server) that only accepts phone calls from numbers on its approved caller list. When an unknown number calls asking for account details, the bank refuses and hangs up.
- The bank = the server hosting the API
- The approved caller list = the `Access-Control-Allow-Origin` list of trusted origins
- The unknown caller = a website from a different domain trying to fetch data

---

## Table of Contents
- [Introduction](#introduction)
- [Same-Origin Policy](#same-origin-policy)
- [CORS Mechanism](#cors-mechanism)
- [CORS Headers](#cors-headers)
- [Preflight Requests](#preflight-requests)
- [Credentials Mode](#credentials-mode)
- [CORS Security Implications](#cors-security-implications)
- [Implementation Examples](#implementation-examples)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use/Not to Use](#when-to-usenot-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

**CORS (Cross-Origin Resource Sharing)** is a security mechanism that allows web pages to make requests to a different domain than the one serving the web page. It relaxes the Same-Origin Policy in a controlled manner.

### Why CORS Matters

```
┌─────────────────────────────────────────────────────────────┐
│              Same-Origin vs Cross-Origin                     │
└─────────────────────────────────────────────────────────────┘

Same-Origin (Allowed by default):
   https://example.com/page
   → https://example.com/api/data
   ✓ Same protocol, domain, and port

Cross-Origin (Requires CORS):
   https://example.com/page
   → https://api.example.com/data
   ✗ Different subdomain

   https://example.com/page
   → https://example.com:3000/api
   ✗ Different port

   https://example.com/page
   → http://example.com/api
   ✗ Different protocol
```

## Same-Origin Policy

The Same-Origin Policy (SOP) is a critical security concept that restricts how documents or scripts from one origin can interact with resources from another origin.

### Origin Definition

```typescript
// Origin = Protocol + Domain + Port

interface Origin {
  protocol: string;  // http:// or https://
  domain: string;    // example.com
  port: number;      // 80, 443, 3000, etc.
}

// Examples
const origin1 = {
  protocol: 'https://',
  domain: 'example.com',
  port: 443
};

const origin2 = {
  protocol: 'https://',
  domain: 'api.example.com',
  port: 443
};

// Different origins!
console.log(isSameOrigin(origin1, origin2)); // false
```

### Same-Origin Policy Examples

```typescript
// Current page: https://example.com:443/page.html

const requests = [
  {
    url: 'https://example.com/api/data',
    sameOrigin: true,  // Same protocol, domain, port
    reason: 'Identical origin'
  },
  {
    url: 'https://example.com:443/api/data',
    sameOrigin: true,  // 443 is default for HTTPS
    reason: 'Explicit default port'
  },
  {
    url: 'http://example.com/api/data',
    sameOrigin: false,  // Different protocol
    reason: 'Protocol mismatch (https vs http)'
  },
  {
    url: 'https://api.example.com/data',
    sameOrigin: false,  // Different domain
    reason: 'Subdomain difference'
  },
  {
    url: 'https://example.com:3000/data',
    sameOrigin: false,  // Different port
    reason: 'Port mismatch (443 vs 3000)'
  },
  {
    url: 'https://example.org/data',
    sameOrigin: false,  // Different domain
    reason: 'Different top-level domain'
  }
];
```

### What SOP Restricts

```
┌─────────────────────────────────────────────────────────────┐
│         Same-Origin Policy Restrictions                      │
└─────────────────────────────────────────────────────────────┘

Cross-origin READS blocked:
  ✗ Fetch/XHR response data
  ✗ Canvas pixel data from cross-origin image
  ✗ iframe document access
  ✗ Cookie access

Cross-origin WRITES allowed (with caveats):
  ✓ Links (<a>)
  ✓ Redirects
  ✓ Form submissions
  ✓ Script includes (but execution in current context)

Cross-origin EMBEDS allowed:
  ✓ <script src="...">
  ✓ <link rel="stylesheet" href="...">
  ✓ <img src="...">
  ✓ <video> / <audio>
  ✓ <iframe src="..."> (with sandbox restrictions)
```

## CORS Mechanism

CORS uses HTTP headers to tell browsers to give a web application running at one origin access to selected resources from a different origin.

### Simple CORS Request Flow

```
┌─────────────────────────────────────────────────────────────┐
│              Simple CORS Request                             │
└─────────────────────────────────────────────────────────────┘

1. Browser makes cross-origin request
   ┌─────────┐                           ┌─────────┐
   │ Browser │ ──── GET /api/data ─────> │ Server  │
   │         │    Origin: https://app.com │         │
   └─────────┘                           └─────────┘

2. Server checks Origin and responds
   ┌─────────┐                           ┌─────────┐
   │ Browser │ <──── 200 OK ──────────── │ Server  │
   │         │    Access-Control-Allow-  │         │
   │         │    Origin: https://app.com│         │
   │         │    Response data           │         │
   └─────────┘                           └─────────┘

3. Browser checks CORS headers
   - Is Origin in Access-Control-Allow-Origin?
   - ✓ Yes: Expose response to JavaScript
   - ✗ No:  Block response (CORS error)
```

### Preflight CORS Request Flow

```
┌─────────────────────────────────────────────────────────────┐
│            Preflight CORS Request                            │
└─────────────────────────────────────────────────────────────┘

Triggered by:
  - Non-simple HTTP methods (PUT, DELETE, PATCH)
  - Custom headers
  - Content-Type other than form-data/urlencoded/text

1. Browser sends OPTIONS preflight
   ┌─────────┐                           ┌─────────┐
   │ Browser │ ──── OPTIONS /api/data ──>│ Server  │
   │         │    Origin: https://app.com │         │
   │         │    Access-Control-Request- │         │
   │         │    Method: DELETE          │         │
   │         │    Access-Control-Request- │         │
   │         │    Headers: Authorization  │         │
   └─────────┘                           └─────────┘

2. Server responds to preflight
   ┌─────────┐                           ┌─────────┐
   │ Browser │ <──── 204 No Content ───── │ Server  │
   │         │    Access-Control-Allow-   │         │
   │         │    Origin: https://app.com │         │
   │         │    Access-Control-Allow-   │         │
   │         │    Methods: GET, POST,     │         │
   │         │             DELETE          │         │
   │         │    Access-Control-Allow-   │         │
   │         │    Headers: Authorization  │         │
   │         │    Access-Control-Max-Age: │         │
   │         │             86400           │         │
   └─────────┘                           └─────────┘

3. Browser checks preflight response
   - Is method allowed?
   - Are headers allowed?
   - ✓ Yes: Send actual request
   - ✗ No:  Block request (CORS error)

4. Actual request (if preflight passed)
   ┌─────────┐                           ┌─────────┐
   │ Browser │ ──── DELETE /api/data ───>│ Server  │
   │         │    Origin: https://app.com │         │
   │         │    Authorization: Bearer..│         │
   └─────────┘                           └─────────┘

5. Server responds
   ┌─────────┐                           ┌─────────┐
   │ Browser │ <──── 200 OK ──────────── │ Server  │
   │         │    Access-Control-Allow-  │         │
   │         │    Origin: https://app.com│         │
   └─────────┘                           └─────────┘
```

### Simple vs Preflight Requests

```typescript
// Simple requests (no preflight)
const simpleRequests = {
  methods: ['GET', 'HEAD', 'POST'],
  headers: [
    'Accept',
    'Accept-Language',
    'Content-Language',
    'Content-Type' // Only if value is:
    // - application/x-www-form-urlencoded
    // - multipart/form-data
    // - text/plain
  ]
};

// Examples of simple requests
fetch('https://api.example.com/data'); // GET, no preflight

fetch('https://api.example.com/data', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded'
  },
  body: 'name=value'
}); // Simple POST, no preflight

// Preflight requests (requires OPTIONS)
fetch('https://api.example.com/data', {
  method: 'DELETE' // ← Non-simple method
}); // Preflight required

fetch('https://api.example.com/data', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json', // ← Non-simple Content-Type
    'Authorization': 'Bearer token' // ← Custom header
  },
  body: JSON.stringify({ data: 'value' })
}); // Preflight required
```

## CORS Headers

### Response Headers (Server → Browser)

```typescript
// CORS response headers
interface CORSResponseHeaders {
  // Required: Specifies allowed origin(s)
  'Access-Control-Allow-Origin': string;
  // Examples: 'https://example.com', '*'
  
  // Allowed HTTP methods
  'Access-Control-Allow-Methods'?: string;
  // Example: 'GET, POST, PUT, DELETE'
  
  // Allowed request headers
  'Access-Control-Allow-Headers'?: string;
  // Example: 'Content-Type, Authorization, X-Custom-Header'
  
  // Expose specific headers to JavaScript
  'Access-Control-Expose-Headers'?: string;
  // Example: 'X-Total-Count, X-Page-Number'
  
  // Allow credentials (cookies, auth headers)
  'Access-Control-Allow-Credentials'?: 'true';
  
  // Preflight cache duration (seconds)
  'Access-Control-Max-Age'?: string;
  // Example: '86400' (24 hours)
}
```

### Request Headers (Browser → Server)

```typescript
// CORS request headers (automatically set by browser)
interface CORSRequestHeaders {
  // Origin of the requesting page
  'Origin': string;
  // Example: 'https://example.com'
  
  // Preflight: Requested HTTP method
  'Access-Control-Request-Method'?: string;
  // Example: 'DELETE'
  
  // Preflight: Requested custom headers
  'Access-Control-Request-Headers'?: string;
  // Example: 'authorization, x-custom-header'
}
```

### Backend Implementation Examples

```typescript
// Express.js: Basic CORS configuration
import express from 'express';
import cors from 'cors';

const app = express();

// Option 1: Allow all origins (NOT recommended for production)
app.use(cors());

// Option 2: Specific origin
app.use(cors({
  origin: 'https://example.com'
}));

// Option 3: Multiple origins
const allowedOrigins = [
  'https://example.com',
  'https://app.example.com',
  'http://localhost:3000'
];

app.use(cors({
  origin: (origin, callback) => {
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  credentials: true, // Allow cookies
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  exposedHeaders: ['X-Total-Count'],
  maxAge: 86400 // 24 hours
}));

// Option 4: Manual CORS middleware
const manualCORS = (req: express.Request, res: express.Response, next: express.NextFunction) => {
  const origin = req.headers.origin;
  
  if (origin && allowedOrigins.includes(origin)) {
    res.setHeader('Access-Control-Allow-Origin', origin);
    res.setHeader('Access-Control-Allow-Credentials', 'true');
    res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
    res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
    res.setHeader('Access-Control-Expose-Headers', 'X-Total-Count');
  }
  
  // Handle preflight
  if (req.method === 'OPTIONS') {
    res.setHeader('Access-Control-Max-Age', '86400');
    return res.status(204).send();
  }
  
  next();
};

app.use(manualCORS);
```

### NestJS CORS Configuration

```typescript
// NestJS: CORS setup
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Option 1: Simple CORS
  app.enableCors();
  
  // Option 2: Configured CORS
  app.enableCors({
    origin: ['https://example.com', 'https://app.example.com'],
    credentials: true,
    methods: 'GET,HEAD,PUT,PATCH,POST,DELETE',
    allowedHeaders: 'Content-Type, Authorization',
    exposedHeaders: 'X-Total-Count, X-Page-Number',
    maxAge: 3600
  });
  
  // Option 3: Dynamic origin validation
  app.enableCors({
    origin: (origin, callback) => {
      const allowedOrigins = process.env.ALLOWED_ORIGINS?.split(',') || [];
      
      if (!origin || allowedOrigins.includes(origin)) {
        callback(null, true);
      } else {
        callback(new Error(`Origin ${origin} not allowed by CORS`));
      }
    },
    credentials: true
  });
  
  await app.listen(3000);
}
bootstrap();
```

## Preflight Requests

Preflight requests are OPTIONS requests that check if the actual request is safe to send.

### When Preflight is Triggered

```typescript
// React: Examples that trigger preflight

// ✗ Triggers preflight: Custom header
fetch('https://api.example.com/data', {
  headers: {
    'X-Custom-Header': 'value'
  }
});

// ✗ Triggers preflight: Content-Type application/json
fetch('https://api.example.com/data', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ data: 'value' })
});

// ✗ Triggers preflight: Non-simple method
fetch('https://api.example.com/data', {
  method: 'DELETE'
});

// ✓ No preflight: Simple request
fetch('https://api.example.com/data', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded'
  },
  body: 'name=value'
});
```

### Optimizing Preflight Requests

```typescript
// Backend: Increase preflight cache duration
app.options('*', cors({
  origin: 'https://example.com',
  maxAge: 86400, // 24 hours - reduces preflight requests
  credentials: true
}));

// Frontend: Batch requests to reduce preflights
class RequestBatcher {
  private queue: Array<{
    url: string;
    options: RequestInit;
    resolve: (value: Response) => void;
    reject: (reason: any) => void;
  }> = [];
  
  private timeout: NodeJS.Timeout | null = null;

  async fetch(url: string, options: RequestInit = {}): Promise<Response> {
    return new Promise((resolve, reject) => {
      this.queue.push({ url, options, resolve, reject });
      
      if (!this.timeout) {
        this.timeout = setTimeout(() => this.flush(), 50);
      }
    });
  }

  private async flush(): Promise<void> {
    const batch = [...this.queue];
    this.queue = [];
    this.timeout = null;

    // Single preflight for batched request
    const response = await fetch('https://api.example.com/batch', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(
        batch.map(req => ({
          url: req.url,
          method: req.options.method || 'GET'
        }))
      )
    });

    const results = await response.json();
    
    batch.forEach((req, index) => {
      req.resolve(new Response(JSON.stringify(results[index])));
    });
  }
}
```

### Angular: Handling Preflight

```typescript
// Angular: HTTP Interceptor for preflight optimization
import { Injectable } from '@angular/core';
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent
} from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable()
export class PreflightOptimizationInterceptor implements HttpInterceptor {
  // Cache preflight results
  private preflightCache = new Map<string, number>();

  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    const cacheKey = `${request.method}:${request.url}`;
    const cachedTime = this.preflightCache.get(cacheKey);

    // If preflight was recent, browser will use cached result
    if (cachedTime && Date.now() - cachedTime < 3600000) { // 1 hour
      console.log('Preflight cached for', cacheKey);
    } else {
      this.preflightCache.set(cacheKey, Date.now());
    }

    return next.handle(request);
  }
}

// Service with preflight-aware requests
@Injectable({
  providedIn: 'root'
})
export class ApiService {
  constructor(private http: HttpClient) {}

  // Avoid triggering unnecessary preflights
  getData(): Observable<any> {
    // Use simple request when possible (no preflight)
    return this.http.get('/api/data'); // Simple GET
  }

  // When preflight is unavoidable, make it count
  postData(data: any): Observable<any> {
    return this.http.post('/api/data', data, {
      headers: {
        'Content-Type': 'application/json',
        // Group custom headers to minimize preflight variations
        'X-Request-ID': generateRequestId()
      }
    });
  }
}
```

## Credentials Mode

Credentials mode controls whether cookies and authorization headers are sent with cross-origin requests.

```typescript
// Credentials modes
type CredentialsMode = 'omit' | 'same-origin' | 'include';

// omit: Never send credentials
fetch('https://api.example.com/data', {
  credentials: 'omit'
});

// same-origin: Send credentials only for same-origin (default)
fetch('https://api.example.com/data', {
  credentials: 'same-origin'
});

// include: Always send credentials (requires server support)
fetch('https://api.example.com/data', {
  credentials: 'include'
});
```

### Credentials with CORS

```typescript
// Backend: Allow credentials
app.use(cors({
  origin: 'https://example.com', // Cannot be '*' with credentials
  credentials: true // Required to accept credentials
}));

// Frontend: Send credentials
fetch('https://api.example.com/data', {
  method: 'POST',
  credentials: 'include', // Include cookies
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ data: 'value' })
});
```

### React: Credentials Management

```typescript
// React: Hook for authenticated requests
import { useState, useEffect } from 'react';

interface UseAuthFetchOptions extends RequestInit {
  skipAuth?: boolean;
}

const useAuthFetch = () => {
  const [isAuthenticated, setIsAuthenticated] = useState(false);

  useEffect(() => {
    // Check authentication status
    checkAuth();
  }, []);

  const checkAuth = async () => {
    try {
      const response = await fetch('https://api.example.com/auth/check', {
        credentials: 'include'
      });
      setIsAuthenticated(response.ok);
    } catch {
      setIsAuthenticated(false);
    }
  };

  const authFetch = async (
    url: string,
    options: UseAuthFetchOptions = {}
  ): Promise<Response> => {
    const { skipAuth, ...fetchOptions } = options;

    if (!skipAuth && !isAuthenticated) {
      throw new Error('Not authenticated');
    }

    return fetch(url, {
      ...fetchOptions,
      credentials: 'include', // Always include cookies
      headers: {
        'Content-Type': 'application/json',
        ...fetchOptions.headers
      }
    });
  };

  return { authFetch, isAuthenticated };
};

// Usage
const UserProfile: React.FC = () => {
  const { authFetch, isAuthenticated } = useAuthFetch();
  const [profile, setProfile] = useState(null);

  useEffect(() => {
    if (isAuthenticated) {
      loadProfile();
    }
  }, [isAuthenticated]);

  const loadProfile = async () => {
    try {
      const response = await authFetch('https://api.example.com/profile');
      const data = await response.json();
      setProfile(data);
    } catch (error) {
      console.error('Failed to load profile:', error);
    }
  };

  return (
    <div>
      {isAuthenticated ? (
        <div>Welcome, {profile?.name}</div>
      ) : (
        <div>Please log in</div>
      )}
    </div>
  );
};
```

### Angular: Credentials with Interceptor

```typescript
// Angular: Interceptor for credentials
@Injectable()
export class CredentialsInterceptor implements HttpInterceptor {
  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    // Add withCredentials for cross-origin requests
    if (this.isCrossOrigin(request.url)) {
      request = request.clone({
        withCredentials: true
      });
    }

    return next.handle(request);
  }

  private isCrossOrigin(url: string): boolean {
    const currentOrigin = window.location.origin;
    try {
      const requestOrigin = new URL(url).origin;
      return currentOrigin !== requestOrigin;
    } catch {
      return false; // Relative URL
    }
  }
}

// Module configuration
@NgModule({
  providers: [
    {
      provide: HTTP_INTERCEPTORS,
      useClass: CredentialsInterceptor,
      multi: true
    }
  ]
})
export class AppModule {}
```

## CORS Security Implications

### Common CORS Misconfigurations

```typescript
// ❌ DANGEROUS: Allow all origins with credentials
app.use(cors({
  origin: '*',
  credentials: true
}));
// Browser will reject this (spec violation)

// ❌ DANGEROUS: Echo Origin header
app.use((req, res, next) => {
  res.setHeader('Access-Control-Allow-Origin', req.headers.origin || '*');
  res.setHeader('Access-Control-Allow-Credentials', 'true');
  next();
});
// Allows ANY origin to make authenticated requests

// ❌ DANGEROUS: Allow null origin
app.use(cors({
  origin: (origin, callback) => {
    callback(null, true); // Accepts all origins including 'null'
  },
  credentials: true
}));
// Attackers can use sandboxed iframes with origin 'null'

// ✓ SECURE: Allowlist specific origins
const allowedOrigins = ['https://example.com', 'https://app.example.com'];

app.use(cors({
  origin: (origin, callback) => {
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  credentials: true
}));
```

### CORS as a Security Mechanism

```
┌─────────────────────────────────────────────────────────────┐
│        What CORS Does and Doesn't Protect                    │
└─────────────────────────────────────────────────────────────┘

CORS DOES protect:
  ✓ Prevents malicious websites from reading your API responses
  ✓ Blocks cross-origin requests from untrusted origins
  ✓ Protects against some forms of CSRF (when credentials involved)

CORS DOES NOT protect:
  ✗ Server-to-server requests (CORS is browser-enforced)
  ✗ Requests from tools like curl, Postman
  ✗ Requests if server is misconfigured
  ✗ Does not prevent request from being sent (only blocks response)

Important: CORS is not a substitute for proper authentication
           and authorization!
```

### CORS Bypass Techniques (Know Your Enemy)

```typescript
// Attack 1: Null origin exploitation
// Sandboxed iframes have origin 'null'
// <iframe sandbox="allow-scripts" src="data:text/html,...">

// Attack 2: Subdomain takeover
// If *.example.com is allowed, attacker can use compromised subdomain

// Attack 3: Server-side proxy
// Attacker's server proxies requests, bypassing browser CORS

// Defense: Server-side validation
class SecureCORSValidator {
  private allowedOrigins = [
    'https://example.com',
    'https://app.example.com'
  ];

  validateOrigin(origin: string | undefined): boolean {
    if (!origin) {
      // Reject missing origin for sensitive endpoints
      return false;
    }

    if (origin === 'null') {
      // Always reject null origin
      return false;
    }

    // Exact match only (no wildcards in production)
    return this.allowedOrigins.includes(origin);
  }

  validateRequest(req: express.Request): boolean {
    const origin = req.headers.origin;
    
    // Check origin
    if (!this.validateOrigin(origin)) {
      return false;
    }

    // Additional validations
    if (!this.validateReferer(req.headers.referer)) {
      return false;
    }

    if (!this.validateUserAgent(req.headers['user-agent'])) {
      return false;
    }

    return true;
  }

  private validateReferer(referer: string | undefined): boolean {
    // Ensure referer matches allowed origins
    if (!referer) return true; // Optional
    
    return this.allowedOrigins.some(origin => 
      referer.startsWith(origin)
    );
  }

  private validateUserAgent(ua: string | undefined): boolean {
    // Block suspicious user agents
    if (!ua) return false;
    
    const suspiciousPatterns = [
      /curl/i,
      /postman/i,
      /python-requests/i
    ];
    
    return !suspiciousPatterns.some(pattern => pattern.test(ua));
  }
}
```

## Common Mistakes

### 1. Wildcard Origin with Credentials

```typescript
// ❌ BAD: Spec violation
res.setHeader('Access-Control-Allow-Origin', '*');
res.setHeader('Access-Control-Allow-Credentials', 'true');
// Browser rejects this

// ✓ GOOD: Specific origin with credentials
res.setHeader('Access-Control-Allow-Origin', 'https://example.com');
res.setHeader('Access-Control-Allow-Credentials', 'true');
```

### 2. Not Handling Preflight OPTIONS Requests

```typescript
// ❌ BAD: Missing OPTIONS handler
app.post('/api/data', (req, res) => {
  res.json({ success: true });
});
// Preflight fails with 404

// ✓ GOOD: Handle OPTIONS
app.options('/api/data', cors());
app.post('/api/data', (req, res) => {
  res.json({ success: true });
});
```

### 3. Forgetting Exposed Headers

```typescript
// ❌ BAD: Custom headers not exposed
app.get('/api/data', (req, res) => {
  res.setHeader('X-Total-Count', '100');
  res.json([...]); // X-Total-Count not accessible to JavaScript
});

// ✓ GOOD: Expose custom headers
app.get('/api/data', cors({
  exposedHeaders: ['X-Total-Count']
}), (req, res) => {
  res.setHeader('X-Total-Count', '100');
  res.json([...]);
});

// Frontend can now access
const response = await fetch('https://api.example.com/data');
const totalCount = response.headers.get('X-Total-Count'); // Works!
```

### 4. Not Caching Preflight Responses

```typescript
// ❌ BAD: No preflight caching
app.options('*', cors());
// Preflight on every non-simple request

// ✓ GOOD: Cache preflight
app.options('*', cors({
  maxAge: 86400 // 24 hours
}));
// Reduces preflight requests
```

### 5. Incorrect Credentials Configuration

```typescript
// ❌ BAD: Mismatch between client and server
// Frontend
fetch('https://api.example.com/data', {
  credentials: 'include' // Trying to send cookies
});

// Backend
app.use(cors({
  origin: 'https://example.com'
  // Missing credentials: true
}));
// Cookies not accepted by browser

// ✓ GOOD: Matched configuration
// Frontend
fetch('https://api.example.com/data', {
  credentials: 'include'
});

// Backend
app.use(cors({
  origin: 'https://example.com',
  credentials: true // ← Required
}));
```

## Best Practices

### 1. Strict Origin Allowlist

```typescript
// Environment-based origin configuration
class OriginValidator {
  private allowedOrigins: string[];

  constructor() {
    this.allowedOrigins = this.loadAllowedOrigins();
  }

  private loadAllowedOrigins(): string[] {
    const envOrigins = process.env.ALLOWED_ORIGINS?.split(',') || [];
    
    // Default origins by environment
    const defaultOrigins = {
      production: ['https://example.com'],
      staging: ['https://staging.example.com', 'https://example.com'],
      development: ['http://localhost:3000', 'http://localhost:4200']
    };

    const env = process.env.NODE_ENV as keyof typeof defaultOrigins || 'development';
    return [...defaultOrigins[env], ...envOrigins];
  }

  isAllowed(origin: string | undefined): boolean {
    if (!origin) return false;
    return this.allowedOrigins.includes(origin);
  }

  getCORSOptions(): cors.CorsOptions {
    return {
      origin: (origin, callback) => {
        if (!origin || this.isAllowed(origin)) {
          callback(null, true);
        } else {
          callback(new Error(`Origin ${origin} not allowed by CORS`));
        }
      },
      credentials: true,
      maxAge: 86400
    };
  }
}

const originValidator = new OriginValidator();
app.use(cors(originValidator.getCORSOptions()));
```

### 2. Conditional CORS

```typescript
// Apply CORS only where needed
const publicRoutes = ['/api/public'];
const authenticatedRoutes = ['/api/user', '/api/data'];

// Public routes: More permissive CORS
app.use('/api/public', cors({
  origin: '*',
  methods: ['GET']
}));

// Authenticated routes: Strict CORS
app.use(authenticatedRoutes, cors({
  origin: ['https://example.com'],
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE']
}));
```

### 3. Security Headers with CORS

```typescript
// Combine CORS with other security headers
import helmet from 'helmet';

app.use(helmet());

app.use((req, res, next) => {
  // CORS headers
  const origin = req.headers.origin;
  if (origin && allowedOrigins.includes(origin)) {
    res.setHeader('Access-Control-Allow-Origin', origin);
    res.setHeader('Access-Control-Allow-Credentials', 'true');
  }

  // Additional security headers
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('X-XSS-Protection', '1; mode=block');
  res.setHeader(
    'Strict-Transport-Security',
    'max-age=31536000; includeSubDomains'
  );

  next();
});
```

### 4. Logging and Monitoring

```typescript
// Monitor CORS issues
class CORSMonitor {
  logCORSRequest(req: express.Request, allowed: boolean) {
    console.log({
      timestamp: new Date().toISOString(),
      origin: req.headers.origin,
      method: req.method,
      url: req.url,
      allowed,
      userAgent: req.headers['user-agent']
    });

    if (!allowed) {
      this.alertOnBlockedOrigin(req.headers.origin);
    }
  }

  private async alertOnBlockedOrigin(origin: string | undefined) {
    // Track blocked origins
    const count = await this.getBlockedCount(origin);
    
    if (count > 10) {
      // Alert security team
      this.sendAlert(`High volume of blocked requests from ${origin}`);
    }
  }
}

const corsMonitor = new CORSMonitor();

app.use((req, res, next) => {
  const origin = req.headers.origin;
  const allowed = !origin || allowedOrigins.includes(origin);
  
  corsMonitor.logCORSRequest(req, allowed);
  
  if (allowed && origin) {
    res.setHeader('Access-Control-Allow-Origin', origin);
    res.setHeader('Access-Control-Allow-Credentials', 'true');
  }
  
  next();
});
```

## When to Use/Not to Use

### When to Configure CORS

1. **Cross-origin API access** - Frontend and backend on different domains
2. **Microservices architecture** - Services on different subdomains
3. **CDN-hosted frontends** - Static site on CDN, API on separate domain
4. **Mobile app APIs** - Apps making requests to your API
5. **Third-party integrations** - Allowing specific partners to access your API

### When CORS is Not Needed

1. **Same-origin requests** - Frontend and backend on same domain
2. **Server-side only APIs** - APIs only accessed by other servers
3. **Internal services** - Services within private network
4. **Proxy pattern** - Using server-side proxy to forward requests

### CORS Alternatives

```typescript
// Alternative 1: Proxy pattern (avoid CORS entirely)
// Frontend calls own backend, which proxies to API
app.get('/api/proxy/*', async (req, res) => {
  const targetUrl = `https://external-api.com${req.params[0]}`;
  const response = await fetch(targetUrl);
  const data = await response.json();
  res.json(data);
});

// Alternative 2: JSONP (legacy, avoid if possible)
app.get('/api/jsonp', (req, res) => {
  const callback = req.query.callback;
  const data = { message: 'Hello' };
  res.send(`${callback}(${JSON.stringify(data)})`);
});

// Alternative 3: PostMessage for iframe communication
// Parent window
const iframe = document.getElementById('api-iframe');
iframe.contentWindow.postMessage({ action: 'getData' }, 'https://api.example.com');

window.addEventListener('message', (event) => {
  if (event.origin === 'https://api.example.com') {
    console.log('Data received:', event.data);
  }
});
```

## Interview Questions

### Q1: What is CORS and why is it needed?

**Answer:** CORS (Cross-Origin Resource Sharing) is a mechanism that allows browsers to make cross-origin requests in a controlled manner. It's needed because of the Same-Origin Policy (SOP), which restricts web pages from making requests to a different domain than the one serving the page.

**Without CORS:** Browser blocks cross-origin requests to protect users from malicious scripts stealing data from other sites.

**With CORS:** Server explicitly allows specific origins to access its resources by sending Access-Control-Allow-Origin headers.

**Example:** Frontend at https://app.com wants to call API at https://api.com. Without CORS headers from api.com, browser blocks the request. With proper CORS configuration, the request succeeds.

**Key point:** CORS is browser-enforced; it doesn't affect server-to-server requests or tools like curl.

### Q2: Explain the difference between simple and preflight CORS requests.

**Answer:**

**Simple requests:**
- Methods: GET, HEAD, POST
- Headers: Only standard headers like Accept, Content-Language
- Content-Type: Only application/x-www-form-urlencoded, multipart/form-data, or text/plain
- No preflight OPTIONS request
- Browser sends request directly

**Preflight requests:**
- Triggered by: Non-simple methods (PUT, DELETE, PATCH), custom headers, or Content-Type: application/json
- Browser sends OPTIONS request first
- Server responds with allowed methods, headers, origins
- If preflight succeeds, browser sends actual request
- Can be cached via Access-Control-Max-Age

**Example:**
```typescript
// Simple (no preflight)
fetch('/api', { method: 'GET' });

// Preflight required
fetch('/api', {
  method: 'DELETE',  // Non-simple method
  headers: { 'Authorization': 'Bearer token' }  // Custom header
});
```

### Q3: Why can't you use Access-Control-Allow-Origin: * with credentials?

**Answer:** This is a security restriction in the CORS specification. If credentials (cookies, Authorization headers) are included, the server must specify an exact origin, not a wildcard.

**Reason:** Wildcard with credentials would allow any website to make authenticated requests to your API and read the responses, completely defeating cross-origin security.

**Attack scenario if allowed:**
1. User logs into https://bank.com (gets session cookie)
2. User visits https://evil.com
3. Evil.com makes request to bank.com with credentials
4. If wildcard was allowed, evil.com could read all user's banking data

**Correct approach:**
```typescript
// Invalid
res.setHeader('Access-Control-Allow-Origin', '*');
res.setHeader('Access-Control-Allow-Credentials', 'true');

// Valid
res.setHeader('Access-Control-Allow-Origin', 'https://trusted-app.com');
res.setHeader('Access-Control-Allow-Credentials', 'true');
```

### Q4: What is a CORS preflight cache and how do you optimize it?

**Answer:** Preflight cache stores the results of OPTIONS requests so the browser doesn't need to send a preflight for every subsequent request.

**How it works:**
- Server sends Access-Control-Max-Age header (in seconds)
- Browser caches preflight response for that duration
- Within cache period, browser skips preflight for identical requests

**Optimization strategies:**

```typescript
// 1. Increase cache duration
app.options('*', cors({ maxAge: 86400 })); // 24 hours

// 2. Reduce preflight triggers
// Use simple requests when possible
fetch('/api', {
  method: 'POST',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' }
  // Simple request, no preflight
});

// 3. Batch requests
// Combine multiple operations into single endpoint

// 4. Use same headers consistently
// Preflight is cached per unique combination of origin, method, headers
```

**Caveat:** Browsers have max cache duration limits (typically 5-10 minutes regardless of Max-Age value).

### Q5: How do you securely configure CORS for a production API?

**Answer:** Secure CORS configuration checklist:

**1. Strict origin allowlist:**
```typescript
const allowedOrigins = [
  'https://app.example.com',
  'https://admin.example.com'
];

app.use(cors({
  origin: (origin, callback) => {
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  }
}));
```

**2. Enable credentials only when needed:**
```typescript
app.use(cors({
  origin: 'https://app.example.com',
  credentials: true  // Only if cookies/auth headers required
}));
```

**3. Restrict methods:**
```typescript
app.use(cors({
  methods: ['GET', 'POST', 'PUT', 'DELETE']  // Don't allow all
}));
```

**4. Expose only necessary headers:**
```typescript
app.use(cors({
  exposedHeaders: ['X-Total-Count']  // Only what frontend needs
}));
```

**5. Set reasonable preflight cache:**
```typescript
app.use(cors({
  maxAge: 3600  // 1 hour (balance between performance and flexibility)
}));
```

**6. Don't rely on CORS alone:**
- Implement proper authentication
- Validate Authorization headers
- Use rate limiting
- Log suspicious activity

### Q6: What are common CORS errors and how do you debug them?

**Answer:**

**Error 1:** "No 'Access-Control-Allow-Origin' header"
- **Cause:** Server not sending CORS headers
- **Fix:** Configure CORS middleware on server

**Error 2:** "Origin not allowed"
- **Cause:** Origin not in allowlist
- **Fix:** Add origin to allowedOrigins array
- **Debug:** Check request Origin header vs server allowlist

**Error 3:** "Preflight response invalid"
- **Cause:** OPTIONS handler missing or misconfigured
- **Fix:** Add OPTIONS handler with proper CORS headers
- **Debug:** Check OPTIONS response headers in DevTools

**Error 4:** "Credentials flag is 'true', but 'Access-Control-Allow-Credentials' is not"
- **Cause:** Client sending credentials but server not configured
- **Fix:** Add credentials: true to server CORS config

**Error 5:** "Wildcard '*' cannot be used when credentials flag is true"
- **Cause:** Using * with credentials
- **Fix:** Specify exact origin

**Debugging approach:**
1. Check browser DevTools Network tab
2. Look at OPTIONS request (preflight) first
3. Verify all required headers present
4. Check origin matches exactly (including protocol, port)
5. Test with curl to verify server-side config:
```bash
curl -H "Origin: https://example.com" \
     -H "Access-Control-Request-Method: POST" \
     -X OPTIONS https://api.example.com/endpoint
```

### Q7: How does CORS relate to CSRF protection?

**Answer:** CORS and CSRF are related but different security mechanisms:

**CORS:**
- Protects against unauthorized cross-origin data reading
- Browser-enforced
- Prevents malicious sites from reading API responses

**CSRF:**
- Protects against unauthorized state-changing actions
- Exploits browser's automatic credential inclusion
- Attacker can trigger request but can't read response

**Relationship:**
- CORS provides some CSRF protection when credentials are used (strict origin validation)
- But CORS alone is insufficient for CSRF protection
- CSRF protection needed even with proper CORS

**Example:**
```typescript
// CORS configured properly
app.use(cors({
  origin: 'https://app.example.com',
  credentials: true
}));

// Still vulnerable to CSRF if no CSRF token
app.post('/transfer', (req, res) => {
  // Attacker from evil.com can't read response (CORS blocks)
  // But can still trigger the request (CSRF attack succeeds)
  transferMoney(req.body.to, req.body.amount);
});

// Proper protection: CORS + CSRF tokens
app.post('/transfer', csrfProtection, (req, res) => {
  transferMoney(req.body.to, req.body.amount);
});
```

**Best practice:** Use both CORS (for origin validation) and CSRF tokens (for request authenticity).

### Q8: How do you handle CORS in a microservices architecture?

**Answer:** Strategies for CORS in microservices:

**Approach 1: API Gateway handles CORS**
```typescript
// API Gateway
app.use(cors({
  origin: ['https://app.example.com'],
  credentials: true
}));

// Forward to microservices
app.use('/api/users', proxy('http://user-service:3000'));
app.use('/api/orders', proxy('http://order-service:3000'));

// Microservices don't need CORS (internal communication)
```

**Approach 2: Each service handles CORS**
```typescript
// Shared CORS configuration
const corsConfig = {
  origin: process.env.ALLOWED_ORIGINS.split(','),
  credentials: true
};

// User Service
app.use(cors(corsConfig));

// Order Service  
app.use(cors(corsConfig));

// Ensures consistent CORS across services
```

**Approach 3: Service Mesh**
- Envoy/Istio handle CORS at infrastructure level
- Consistent policy across all services
- Centralized configuration

**Recommendations:**
- API Gateway approach for simplicity
- Centralized origin configuration
- Monitor CORS errors across services
- Document which services require CORS

## Key Takeaways

1. **CORS relaxes Same-Origin Policy** - Allows controlled cross-origin requests that would otherwise be blocked by browser security
2. **Server must opt-in** - CORS requires server to send Access-Control headers; browser enforces based on server's response
3. **Preflight for complex requests** - Non-simple requests trigger OPTIONS preflight to verify server allows the actual request
4. **Credentials require exact origin** - Cannot use wildcard (*) with credentials; must specify exact allowed origin
5. **CORS is browser-enforced only** - Server-to-server requests, curl, Postman ignore CORS; need server-side auth/validation
6. **Strict origin allowlisting** - Never echo Origin header or use wildcard with credentials; maintain explicit allowlist
7. **Expose headers explicitly** - Custom response headers only accessible if listed in Access-Control-Expose-Headers
8. **Cache preflight responses** - Use Access-Control-Max-Age to reduce preflight requests and improve performance
9. **CORS ≠ CSRF protection** - CORS prevents reading responses but doesn't prevent requests; still need CSRF tokens
10. **Monitor and log CORS issues** - Track blocked origins and frequent violations to detect attacks and configuration problems

## Resources

### Official Documentation
- [MDN: Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [W3C CORS Specification](https://www.w3.org/TR/cors/)
- [Fetch Standard](https://fetch.spec.whatwg.org/#http-cors-protocol)

### Libraries and Tools
- [cors (Express middleware)](https://www.npmjs.com/package/cors)
- [@nestjs/common](https://docs.nestjs.com/security/cors) - NestJS CORS
- [Test CORS](https://www.test-cors.org/) - CORS testing tool

### Articles and Guides
- [Understanding CORS](https://web.dev/cross-origin-resource-sharing/)
- [CORS in Action](https://livebook.manning.com/book/cors-in-action/)
- [CORS Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html#cross-origin-resource-sharing)

### Security Resources
- [OWASP: Cross-Origin Resource Sharing](https://owasp.org/www-community/attacks/CORS_OriginHeaderScrutiny)
- [PortSwigger: CORS Vulnerabilities](https://portswigger.net/web-security/cors)
- [Common CORS Misconfigurations](https://blog.detectify.com/2018/04/26/cors-misconfigurations-explained/)
