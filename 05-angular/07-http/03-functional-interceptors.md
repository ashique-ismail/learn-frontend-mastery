# Functional Interceptors

## The Idea

**In plain English:** A functional interceptor is a function that automatically runs on every web request your app makes or receives — letting you do things like add a password (token) to every request, or check every response for errors, without repeating that code everywhere.

**Real-world analogy:** Think of a hotel concierge desk that every guest must pass through when leaving or returning. The concierge checks your keycard, stamps your pass, and notes the time — every single time, automatically.

- The concierge desk = the interceptor function
- The guest's pass being stamped = the HTTP request being modified (e.g., a token added)
- Every guest passing through = every HTTP request/response going through the interceptor chain

---

## Overview

Functional interceptors, introduced in Angular 15, provide a modern, lightweight approach to intercepting HTTP requests and responses. Unlike class-based interceptors that require dependency injection through constructors, functional interceptors are simple functions that compose elegantly and integrate seamlessly with Angular's standalone APIs.

Functional interceptors use the `HttpInterceptorFn` type and are registered via `provideHttpClient(withInterceptors([...]))`. They receive the request and a `next` handler function, allowing you to modify requests, transform responses, handle errors, and coordinate behavior across the interceptor chain using `HttpContext`.

## Core Concepts

### 1. HttpInterceptorFn Signature

The `HttpInterceptorFn` is a function type that defines the interceptor contract.

**Basic Interceptor Structure:**

```typescript
// logging.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';

export const loggingInterceptor: HttpInterceptorFn = (req, next) => {
  console.log('Request URL:', req.url);
  console.log('Request method:', req.method);
  
  const startTime = Date.now();
  
  return next(req).pipe(
    tap({
      next: (event) => {
        if (event.type === HttpEventType.Response) {
          const elapsed = Date.now() - startTime;
          console.log(`Response received in ${elapsed}ms`);
        }
      },
      error: (error) => {
        console.error('Request failed:', error);
      }
    })
  );
};

// Type signature
type HttpInterceptorFn = (
  req: HttpRequest<unknown>,
  next: HttpHandlerFn
) => Observable<HttpEvent<unknown>>;

type HttpHandlerFn = (
  req: HttpRequest<unknown>
) => Observable<HttpEvent<unknown>>;
```

**Registration with provideHttpClient:**

```typescript
// main.ts (Standalone App)
import { bootstrapApplication } from '@angular/platform-browser';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { loggingInterceptor } from './interceptors/logging.interceptor';
import { authInterceptor } from './interceptors/auth.interceptor';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, {
  providers: [
    provideHttpClient(
      withInterceptors([
        loggingInterceptor,
        authInterceptor,
        // Interceptors execute in order
      ])
    )
  ]
});

// app.config.ts (Alternative standalone approach)
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient, withInterceptors } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([
        loggingInterceptor,
        authInterceptor
      ])
    )
  ]
};
```

### 2. Request Modification

Functional interceptors can modify requests by cloning them with updated properties.

**Request Modification Examples:**

```typescript
// auth-token.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { AuthService } from '../services/auth.service';

export const authTokenInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.getToken();

  // Skip authentication for certain URLs
  if (req.url.includes('/public/') || req.url.includes('/auth/login')) {
    return next(req);
  }

  // Clone request and add authorization header
  if (token) {
    const authReq = req.clone({
      setHeaders: {
        Authorization: `Bearer ${token}`
      }
    });
    return next(authReq);
  }

  return next(req);
};

// api-prefix.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { environment } from '../environments/environment';

export const apiPrefixInterceptor: HttpInterceptorFn = (req, next) => {
  // Add API base URL to relative paths
  if (!req.url.startsWith('http')) {
    const apiReq = req.clone({
      url: `${environment.apiUrl}${req.url}`
    });
    return next(apiReq);
  }

  return next(req);
};

// headers.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';

export const headersInterceptor: HttpInterceptorFn = (req, next) => {
  // Add common headers
  const modifiedReq = req.clone({
    setHeaders: {
      'Content-Type': 'application/json',
      'Accept': 'application/json',
      'X-API-Version': '1.0',
      'X-Request-ID': generateRequestId()
    }
  });

  return next(modifiedReq);
};

function generateRequestId(): string {
  return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
}

// cache-control.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';

export const cacheControlInterceptor: HttpInterceptorFn = (req, next) => {
  // Add cache control for GET requests
  if (req.method === 'GET') {
    const cachedReq = req.clone({
      setHeaders: {
        'Cache-Control': 'no-cache',
        'Pragma': 'no-cache'
      }
    });
    return next(cachedReq);
  }

  return next(req);
};
```

### 3. Response Transformation

Interceptors can transform responses using RxJS operators.

**Response Transformation Examples:**

```typescript
// response-unwrapper.interceptor.ts
import { HttpInterceptorFn, HttpResponse } from '@angular/common/http';
import { map } from 'rxjs/operators';

interface ApiWrapper<T> {
  success: boolean;
  data: T;
  message: string;
}

export const responseUnwrapperInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    map(event => {
      // Only process successful responses
      if (event instanceof HttpResponse && event.body) {
        const body = event.body as ApiWrapper<any>;
        
        // Unwrap if response has wrapper structure
        if (body && typeof body === 'object' && 'data' in body) {
          return event.clone({ body: body.data });
        }
      }
      return event;
    })
  );
};

// timestamp.interceptor.ts
import { HttpInterceptorFn, HttpResponse } from '@angular/common/http';
import { map } from 'rxjs/operators';

export const timestampInterceptor: HttpInterceptorFn = (req, next) => {
  const requestTime = Date.now();

  return next(req).pipe(
    map(event => {
      if (event instanceof HttpResponse) {
        const responseTime = Date.now();
        const elapsed = responseTime - requestTime;

        // Add custom header with elapsed time
        return event.clone({
          headers: event.headers.set('X-Response-Time', elapsed.toString())
        });
      }
      return event;
    })
  );
};

// date-parser.interceptor.ts
import { HttpInterceptorFn, HttpResponse } from '@angular/common/http';
import { map } from 'rxjs/operators';

export const dateParserInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    map(event => {
      if (event instanceof HttpResponse && event.body) {
        const body = parseDates(event.body);
        return event.clone({ body });
      }
      return event;
    })
  );
};

function parseDates(obj: any): any {
  if (obj === null || obj === undefined) return obj;
  
  if (typeof obj === 'string') {
    // Check if string is ISO date format
    const isoDateRegex = /^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}/;
    if (isoDateRegex.test(obj)) {
      return new Date(obj);
    }
  }
  
  if (Array.isArray(obj)) {
    return obj.map(item => parseDates(item));
  }
  
  if (typeof obj === 'object') {
    const result: any = {};
    for (const key in obj) {
      result[key] = parseDates(obj[key]);
    }
    return result;
  }
  
  return obj;
}
```

### 4. Error Handling

Functional interceptors can catch and handle HTTP errors globally.

**Error Handling Examples:**

```typescript
// error-handler.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { catchError, throwError } from 'rxjs';
import { inject } from '@angular/core';
import { Router } from '@angular/router';
import { NotificationService } from '../services/notification.service';

export const errorHandlerInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);
  const notificationService = inject(NotificationService);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      let errorMessage = 'An error occurred';

      if (error.error instanceof ErrorEvent) {
        // Client-side error
        errorMessage = `Error: ${error.error.message}`;
      } else {
        // Server-side error
        switch (error.status) {
          case 400:
            errorMessage = 'Bad request. Please check your input.';
            break;
          case 401:
            errorMessage = 'Unauthorized. Please login.';
            router.navigate(['/login']);
            break;
          case 403:
            errorMessage = 'Forbidden. You do not have permission.';
            break;
          case 404:
            errorMessage = 'Resource not found.';
            break;
          case 500:
            errorMessage = 'Server error. Please try again later.';
            break;
          default:
            errorMessage = `Error: ${error.status} - ${error.message}`;
        }
      }

      notificationService.error(errorMessage);
      return throwError(() => error);
    })
  );
};

// retry.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { retry, timer } from 'rxjs';
import { retryWhen, mergeMap, throwError } from 'rxjs';

export const retryInterceptor: HttpInterceptorFn = (req, next) => {
  const maxRetries = 3;
  const delayMs = 1000;

  return next(req).pipe(
    retryWhen(errors =>
      errors.pipe(
        mergeMap((error: HttpErrorResponse, index) => {
          // Only retry on specific status codes
          const retryableStatuses = [408, 429, 500, 502, 503, 504];
          
          if (
            index < maxRetries &&
            retryableStatuses.includes(error.status)
          ) {
            console.log(`Retrying request (${index + 1}/${maxRetries})...`);
            return timer(delayMs * (index + 1)); // Exponential backoff
          }
          
          return throwError(() => error);
        })
      )
    )
  );
};

// offline-handler.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { catchError, throwError } from 'rxjs';
import { inject } from '@angular/core';
import { OfflineService } from '../services/offline.service';

export const offlineHandlerInterceptor: HttpInterceptorFn = (req, next) => {
  const offlineService = inject(OfflineService);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (!navigator.onLine || error.status === 0) {
        offlineService.queueRequest(req);
        offlineService.showOfflineMessage();
        return throwError(() => new Error('You are offline. Request queued.'));
      }
      return throwError(() => error);
    })
  );
};
```

### 5. Request/Response Chain

Multiple interceptors form a chain, executing in registration order.

**Chain Composition Examples:**

```typescript
// main.ts - Interceptor chain order matters
import { provideHttpClient, withInterceptors } from '@angular/common/http';

bootstrapApplication(AppComponent, {
  providers: [
    provideHttpClient(
      withInterceptors([
        // 1. First: Log outgoing request
        loggingInterceptor,
        
        // 2. Add API prefix to URLs
        apiPrefixInterceptor,
        
        // 3. Add authentication token
        authTokenInterceptor,
        
        // 4. Add common headers
        headersInterceptor,
        
        // 5. Handle caching
        cacheInterceptor,
        
        // 6. Handle errors
        errorHandlerInterceptor,
        
        // 7. Retry failed requests
        retryInterceptor,
        
        // 8. Parse response dates
        dateParserInterceptor,
        
        // 9. Unwrap API responses
        responseUnwrapperInterceptor,
        
        // 10. Log final response
        responseLoggingInterceptor
      ])
    )
  ]
});

// chain-demo.interceptor.ts - Understanding execution flow
import { HttpInterceptorFn } from '@angular/common/http';
import { tap, finalize } from 'rxjs/operators';

export const firstInterceptor: HttpInterceptorFn = (req, next) => {
  console.log('1. First interceptor - Request');
  
  return next(req).pipe(
    tap(() => console.log('1. First interceptor - Response')),
    finalize(() => console.log('1. First interceptor - Finalize'))
  );
};

export const secondInterceptor: HttpInterceptorFn = (req, next) => {
  console.log('2. Second interceptor - Request');
  
  return next(req).pipe(
    tap(() => console.log('2. Second interceptor - Response')),
    finalize(() => console.log('2. Second interceptor - Finalize'))
  );
};

// Execution order:
// 1. First interceptor - Request
// 2. Second interceptor - Request
// HTTP Request sent
// HTTP Response received
// 2. Second interceptor - Response
// 1. First interceptor - Response
// 2. Second interceptor - Finalize
// 1. First interceptor - Finalize
```

### 6. HttpContext for Metadata

`HttpContext` allows passing metadata between interceptors without modifying the request.

**HttpContext Examples:**

```typescript
// http-context-tokens.ts
import { HttpContext, HttpContextToken } from '@angular/common/http';

// Define context tokens
export const SKIP_AUTH = new HttpContextToken<boolean>(() => false);
export const USE_CACHE = new HttpContextToken<boolean>(() => false);
export const RETRY_COUNT = new HttpContextToken<number>(() => 3);
export const SHOW_LOADER = new HttpContextToken<boolean>(() => true);
export const CUSTOM_ERROR_HANDLER = new HttpContextToken<boolean>(() => false);

// auth.interceptor.ts - Using context
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { SKIP_AUTH } from './http-context-tokens';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  // Check if authentication should be skipped
  if (req.context.get(SKIP_AUTH)) {
    return next(req);
  }

  const authService = inject(AuthService);
  const token = authService.getToken();

  if (token) {
    const authReq = req.clone({
      setHeaders: { Authorization: `Bearer ${token}` }
    });
    return next(authReq);
  }

  return next(req);
};

// cache.interceptor.ts - Using context
import { HttpInterceptorFn, HttpResponse } from '@angular/common/http';
import { inject } from '@angular/core';
import { of, tap, shareReplay } from 'rxjs';
import { USE_CACHE } from './http-context-tokens';

const cache = new Map<string, HttpResponse<any>>();

export const cacheInterceptor: HttpInterceptorFn = (req, next) => {
  // Only cache GET requests with USE_CACHE context
  if (req.method !== 'GET' || !req.context.get(USE_CACHE)) {
    return next(req);
  }

  const cachedResponse = cache.get(req.url);
  if (cachedResponse) {
    console.log('Returning cached response for:', req.url);
    return of(cachedResponse);
  }

  return next(req).pipe(
    tap(event => {
      if (event instanceof HttpResponse) {
        cache.set(req.url, event);
      }
    })
  );
};

// loader.interceptor.ts - Using context
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { finalize } from 'rxjs/operators';
import { LoaderService } from '../services/loader.service';
import { SHOW_LOADER } from './http-context-tokens';

export const loaderInterceptor: HttpInterceptorFn = (req, next) => {
  const loaderService = inject(LoaderService);
  
  if (req.context.get(SHOW_LOADER)) {
    loaderService.show();
  }

  return next(req).pipe(
    finalize(() => {
      if (req.context.get(SHOW_LOADER)) {
        loaderService.hide();
      }
    })
  );
};

// Service usage with context
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpContext } from '@angular/common/http';
import { SKIP_AUTH, USE_CACHE, SHOW_LOADER } from './http-context-tokens';

@Injectable({
  providedIn: 'root'
})
export class DataService {
  private http = inject(HttpClient);

  // Public endpoint - skip authentication
  getPublicData() {
    return this.http.get('/api/public/data', {
      context: new HttpContext().set(SKIP_AUTH, true)
    });
  }

  // Cached endpoint
  getCachedData() {
    return this.http.get('/api/cached-data', {
      context: new HttpContext().set(USE_CACHE, true)
    });
  }

  // Silent request without loader
  getSilentData() {
    return this.http.get('/api/silent', {
      context: new HttpContext().set(SHOW_LOADER, false)
    });
  }

  // Multiple context values
  getSpecialData() {
    return this.http.get('/api/special', {
      context: new HttpContext()
        .set(SKIP_AUTH, true)
        .set(USE_CACHE, true)
        .set(SHOW_LOADER, false)
    });
  }
}
```

### 7. Service Injection in Interceptors

Functional interceptors use the `inject()` function for dependency injection.

**Service Injection Examples:**

```typescript
// service-injection.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { Router } from '@angular/router';
import { AuthService } from '../services/auth.service';
import { LoggerService } from '../services/logger.service';
import { ConfigService } from '../services/config.service';

export const multiServiceInterceptor: HttpInterceptorFn = (req, next) => {
  // Inject multiple services
  const authService = inject(AuthService);
  const logger = inject(LoggerService);
  const config = inject(ConfigService);
  const router = inject(Router);

  // Use services
  const token = authService.getToken();
  const apiUrl = config.get('apiUrl');
  
  logger.log('Intercepting request:', req.url);

  if (!token && req.url.startsWith(apiUrl)) {
    logger.warn('No token found, redirecting to login');
    router.navigate(['/login']);
    return throwError(() => new Error('Authentication required'));
  }

  const authReq = req.clone({
    setHeaders: { Authorization: `Bearer ${token}` }
  });

  return next(authReq).pipe(
    tap(() => logger.log('Request completed:', req.url))
  );
};

// Store injection example
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { Store } from '@ngrx/store';
import { selectAuthToken } from '../store/auth.selectors';
import { take, switchMap } from 'rxjs/operators';

export const storeInterceptor: HttpInterceptorFn = (req, next) => {
  const store = inject(Store);

  // Get token from store
  return store.select(selectAuthToken).pipe(
    take(1),
    switchMap(token => {
      if (token) {
        const authReq = req.clone({
          setHeaders: { Authorization: `Bearer ${token}` }
        });
        return next(authReq);
      }
      return next(req);
    })
  );
};
```

### 8. Conditional Interceptor Logic

Implement conditional logic to apply interceptor behavior selectively.

**Conditional Logic Examples:**

```typescript
// conditional.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';

export const conditionalInterceptor: HttpInterceptorFn = (req, next) => {
  // Condition 1: URL-based
  if (req.url.includes('/api/v1/')) {
    return next(addV1Headers(req));
  }

  // Condition 2: Method-based
  if (req.method === 'POST' || req.method === 'PUT') {
    return next(addTimestamp(req));
  }

  // Condition 3: Header-based
  if (req.headers.has('X-Custom-Processing')) {
    return next(processCustom(req));
  }

  // Default: pass through unchanged
  return next(req);
};

function addV1Headers(req: HttpRequest<any>): HttpRequest<any> {
  return req.clone({
    setHeaders: { 'X-API-Version': '1.0' }
  });
}

function addTimestamp(req: HttpRequest<any>): HttpRequest<any> {
  return req.clone({
    setHeaders: { 'X-Timestamp': Date.now().toString() }
  });
}

function processCustom(req: HttpRequest<any>): HttpRequest<any> {
  return req.clone({
    setHeaders: { 'X-Processing': 'enabled' }
  });
}

// environment-based.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { environment } from '../environments/environment';

export const environmentInterceptor: HttpInterceptorFn = (req, next) => {
  if (environment.production) {
    // Production: strict error handling
    return next(req).pipe(
      catchError(error => {
        // Log to external service
        logToExternalService(error);
        return throwError(() => error);
      })
    );
  } else {
    // Development: verbose logging
    console.log('DEV Request:', req.method, req.url);
    return next(req).pipe(
      tap(event => {
        if (event instanceof HttpResponse) {
          console.log('DEV Response:', event.status, event.body);
        }
      })
    );
  }
};
```

### 9. Performance Monitoring

Track request performance and timing with interceptors.

**Performance Monitoring Examples:**

```typescript
// performance.interceptor.ts
import { HttpInterceptorFn, HttpResponse } from '@angular/common/http';
import { tap } from 'rxjs/operators';
import { inject } from '@angular/core';
import { PerformanceService } from '../services/performance.service';

export const performanceInterceptor: HttpInterceptorFn = (req, next) => {
  const performanceService = inject(PerformanceService);
  const startTime = performance.now();
  const requestId = generateRequestId();

  return next(req).pipe(
    tap({
      next: (event) => {
        if (event instanceof HttpResponse) {
          const endTime = performance.now();
          const duration = endTime - startTime;

          // Track metrics
          performanceService.trackRequest({
            id: requestId,
            url: req.url,
            method: req.method,
            status: event.status,
            duration,
            timestamp: Date.now(),
            size: calculateResponseSize(event)
          });

          // Warn on slow requests
          if (duration > 3000) {
            console.warn(`Slow request detected: ${req.url} took ${duration}ms`);
          }
        }
      },
      error: (error) => {
        const endTime = performance.now();
        const duration = endTime - startTime;

        performanceService.trackError({
          id: requestId,
          url: req.url,
          method: req.method,
          error: error.message,
          duration,
          timestamp: Date.now()
        });
      }
    })
  );
};

function calculateResponseSize(response: HttpResponse<any>): number {
  const bodySize = JSON.stringify(response.body).length;
  const headerSize = response.headers.keys().reduce(
    (size, key) => size + key.length + (response.headers.get(key)?.length || 0),
    0
  );
  return bodySize + headerSize;
}

function generateRequestId(): string {
  return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
}
```

### 10. Testing Functional Interceptors

Functional interceptors are easy to test as pure functions.

**Testing Examples:**

```typescript
// auth.interceptor.spec.ts
import { TestBed } from '@angular/core/testing';
import { HttpRequest, HttpHandler, HttpEvent, HttpResponse } from '@angular/common/http';
import { Observable, of } from 'rxjs';
import { authInterceptor } from './auth.interceptor';
import { AuthService } from '../services/auth.service';

describe('authInterceptor', () => {
  let mockAuthService: jasmine.SpyObj<AuthService>;

  beforeEach(() => {
    mockAuthService = jasmine.createSpyObj('AuthService', ['getToken']);
    
    TestBed.configureTestingModule({
      providers: [
        { provide: AuthService, useValue: mockAuthService }
      ]
    });
  });

  it('should add authorization header when token exists', (done) => {
    const token = 'test-token';
    mockAuthService.getToken.and.returnValue(token);

    const request = new HttpRequest('GET', '/api/users');
    const next: HttpHandler = {
      handle: (req: HttpRequest<any>) => {
        expect(req.headers.get('Authorization')).toBe(`Bearer ${token}`);
        return of(new HttpResponse({ status: 200 }));
      }
    };

    TestBed.runInInjectionContext(() => {
      authInterceptor(request, next.handle).subscribe(() => {
        done();
      });
    });
  });

  it('should not add header when token is missing', (done) => {
    mockAuthService.getToken.and.returnValue(null);

    const request = new HttpRequest('GET', '/api/users');
    const next: HttpHandler = {
      handle: (req: HttpRequest<any>) => {
        expect(req.headers.has('Authorization')).toBe(false);
        return of(new HttpResponse({ status: 200 }));
      }
    };

    TestBed.runInInjectionContext(() => {
      authInterceptor(request, next.handle).subscribe(() => {
        done();
      });
    });
  });
});

// cache.interceptor.spec.ts
import { TestBed } from '@angular/core/testing';
import { HttpRequest, HttpResponse, HttpContext } from '@angular/common/http';
import { of } from 'rxjs';
import { cacheInterceptor } from './cache.interceptor';
import { USE_CACHE } from './http-context-tokens';

describe('cacheInterceptor', () => {
  it('should cache GET requests when USE_CACHE is true', (done) => {
    const request = new HttpRequest('GET', '/api/data', {
      context: new HttpContext().set(USE_CACHE, true)
    });

    let callCount = 0;
    const mockResponse = new HttpResponse({ body: { data: 'test' }, status: 200 });
    
    const next = (req: HttpRequest<any>) => {
      callCount++;
      return of(mockResponse);
    };

    TestBed.runInInjectionContext(() => {
      // First call
      cacheInterceptor(request, next).subscribe(() => {
        // Second call (should use cache)
        cacheInterceptor(request, next).subscribe(() => {
          expect(callCount).toBe(1); // Only called once
          done();
        });
      });
    });
  });
});
```

## Common Mistakes

### 1. Mutating Request Directly

```typescript
// WRONG: Cannot mutate request
export const wrongInterceptor: HttpInterceptorFn = (req, next) => {
  req.headers.set('Authorization', 'token'); // Error!
  return next(req);
};

// CORRECT: Clone the request
export const correctInterceptor: HttpInterceptorFn = (req, next) => {
  const authReq = req.clone({
    setHeaders: { Authorization: 'token' }
  });
  return next(authReq);
};
```

### 2. Not Returning Observable

```typescript
// WRONG: Missing return
export const wrongInterceptor: HttpInterceptorFn = (req, next) => {
  next(req); // Doesn't return!
};

// CORRECT: Return the observable
export const correctInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req);
};
```

### 3. Incorrect Chain Order

```typescript
// WRONG: Auth after API prefix
provideHttpClient(
  withInterceptors([
    authInterceptor,      // Adds auth to relative URL
    apiPrefixInterceptor  // Then adds prefix - auth on wrong URL!
  ])
)

// CORRECT: API prefix before auth
provideHttpClient(
  withInterceptors([
    apiPrefixInterceptor, // First make URL absolute
    authInterceptor       // Then add auth to correct URL
  ])
)
```

### 4. Forgetting to Call next()

```typescript
// WRONG: Doesn't call next()
export const wrongInterceptor: HttpInterceptorFn = (req, next) => {
  console.log('Request:', req.url);
  return of(new HttpResponse({ body: {} })); // Chain broken!
};

// CORRECT: Always call next()
export const correctInterceptor: HttpInterceptorFn = (req, next) => {
  console.log('Request:', req.url);
  return next(req); // Continue chain
};
```

### 5. Improper Error Handling

```typescript
// WRONG: Swallows error
export const wrongInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    catchError(error => {
      console.error(error);
      return of(null); // Hides error from caller!
    })
  );
};

// CORRECT: Re-throw or handle appropriately
export const correctInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    catchError(error => {
      console.error(error);
      return throwError(() => error); // Propagate error
    })
  );
};
```

## Best Practices

### 1. Keep Interceptors Focused

```typescript
// Each interceptor should have a single responsibility
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  // Only handles authentication
};

export const loggingInterceptor: HttpInterceptorFn = (req, next) => {
  // Only handles logging
};
```

### 2. Use HttpContext for Configuration

```typescript
// Define tokens for inter-interceptor communication
export const SKIP_LOADER = new HttpContextToken(() => false);
export const CACHE_TTL = new HttpContextToken(() => 300);
```

### 3. Order Interceptors Carefully

```typescript
// Request processing: top to bottom
// Response processing: bottom to top
withInterceptors([
  loggingInterceptor,      // 1st in, last out
  apiPrefixInterceptor,    // 2nd in, 2nd-to-last out
  authInterceptor          // Last in, first out
])
```

### 4. Handle Errors Gracefully

```typescript
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      // Log error
      // Show user notification
      // Return fallback or re-throw
      return throwError(() => error);
    })
  );
};
```

### 5. Test Interceptors

```typescript
// Write unit tests for interceptor logic
it('should add auth header', () => {
  // Test interceptor behavior in isolation
});
```

## Interview Questions

**Q: What's the difference between functional and class-based interceptors?**

A: Functional interceptors are simple functions with `HttpInterceptorFn` type, registered with `withInterceptors([])`. Class-based interceptors implement `HttpInterceptor` interface, require `@Injectable()`, and are registered via `HTTP_INTERCEPTORS` token. Functional interceptors are lighter, more composable, and the preferred approach in modern Angular.

**Q: How do interceptors execute in a chain?**

A: Interceptors execute in registration order for requests (top to bottom), then in reverse order for responses (bottom to top). Each interceptor calls `next(req)` to pass control to the next interceptor.

**Q: What is HttpContext used for?**

A: `HttpContext` passes metadata between interceptors without modifying the request itself. It uses `HttpContextToken` keys to store values that interceptors can read to conditionally apply logic.

**Q: Can you skip an interceptor for specific requests?**

A: Yes, use `HttpContext` with a token to signal skipping:

```typescript
http.get('/api/data', {
  context: new HttpContext().set(SKIP_AUTH, true)
});
```

**Q: How do you inject services into functional interceptors?**

A: Use the `inject()` function:

```typescript
export const myInterceptor: HttpInterceptorFn = (req, next) => {
  const service = inject(MyService);
  // Use service
};
```

## Key Takeaways

1. Functional interceptors are functions with `HttpInterceptorFn` signature: `(req, next) => Observable`
2. Register with `provideHttpClient(withInterceptors([...]))` in standalone apps
3. Always clone requests before modification - they are immutable
4. Interceptors execute in order for requests, reverse order for responses
5. Use `HttpContext` and `HttpContextToken` to pass metadata between interceptors
6. Inject services using the `inject()` function within interceptor body
7. Always call `next(req)` to continue the interceptor chain
8. Handle errors with `catchError` and decide whether to re-throw or return fallback
9. Keep interceptors focused on single responsibilities for maintainability
10. Order matters - place URL modification before authentication, logging first/last

## Resources

- [Angular Functional Interceptors Guide](https://angular.io/guide/http-interceptors#functional-interceptors)
- [HttpInterceptorFn API](https://angular.io/api/common/http/HttpInterceptorFn)
- [HttpContext Documentation](https://angular.io/api/common/http/HttpContext)
- [Migration from Class to Functional Interceptors](https://angular.io/guide/http-interceptors#migrating-from-class-based-interceptors)
- [Testing HTTP Requests](https://angular.io/guide/http-test-requests)
