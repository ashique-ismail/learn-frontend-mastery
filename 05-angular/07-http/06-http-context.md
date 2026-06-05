# HTTP Context API

## The Idea

**In plain English:** HTTP Context is a way to attach hidden sticky notes to a web request so that different parts of your app (called interceptors) can read those notes and decide how to behave — without changing the actual request being sent. An interceptor is a piece of code that runs automatically every time your app sends or receives data over the internet.

**Real-world analogy:** Imagine you drop off a package at a shipping counter and hand the clerk a separate instruction card that says "fragile - handle with care, require signature, no weekend delivery." The card never goes inside the box or on the label, but every handler along the way reads it and adjusts how they treat the package.

- The instruction card = the `HttpContext` object
- Each specific instruction on the card (e.g. "fragile") = an `HttpContextToken`
- Each shipping handler reading the card = an HTTP interceptor checking `req.context.get(TOKEN)`

---

## Overview

The HTTP Context API, introduced in Angular 12, provides a type-safe mechanism for passing metadata through the HTTP request/response pipeline without modifying the request itself. It allows interceptors to coordinate behavior, share data, and make decisions based on context tokens rather than inspecting URLs or headers.

`HttpContext` uses `HttpContextToken` keys to store and retrieve values, similar to Angular's injection tokens. This approach enables clean separation of concerns, type safety, and flexible configuration of HTTP behavior on a per-request basis.

## Core Concepts

### 1. HttpContextToken

`HttpContextToken` is a typed key used to store and retrieve values from `HttpContext`.

**Basic Token Definition:**

```typescript
// http-context-tokens.ts
import { HttpContextToken } from '@angular/common/http';

// Boolean token with default value
export const SKIP_AUTH = new HttpContextToken<boolean>(() => false);

// Number token
export const RETRY_COUNT = new HttpContextToken<number>(() => 3);

// String token
export const CACHE_KEY = new HttpContextToken<string>(() => '');

// Object token
export const METADATA = new HttpContextToken<{
  source: string;
  timestamp: number;
}>(() => ({ source: 'default', timestamp: Date.now() }));

// Complex type token
interface RequestConfig {
  showLoader?: boolean;
  timeout?: number;
  cacheDuration?: number;
  errorHandler?: 'global' | 'silent' | 'custom';
}

export const REQUEST_CONFIG = new HttpContextToken<RequestConfig>(() => ({
  showLoader: true,
  timeout: 30000,
  cacheDuration: 0,
  errorHandler: 'global'
}));

// Type-safe token
export const CUSTOM_ERROR_HANDLER = new HttpContextToken<
  (error: HttpErrorResponse) => void
>(() => () => {});

// Array token
export const ALLOWED_STATUS_CODES = new HttpContextToken<number[]>(() => [200, 201]);
```

### 2. Creating and Using HttpContext

Create `HttpContext` instances and set token values when making HTTP requests.

**HttpContext Usage:**

```typescript
// data.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpContext } from '@angular/common/http';
import { Observable } from 'rxjs';
import { SKIP_AUTH, RETRY_COUNT, REQUEST_CONFIG } from './http-context-tokens';

interface User {
  id: number;
  name: string;
  email: string;
}

@Injectable({
  providedIn: 'root'
})
export class DataService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com';

  // Simple context usage
  getPublicData(): Observable<any> {
    return this.http.get(`${this.apiUrl}/public/data`, {
      context: new HttpContext().set(SKIP_AUTH, true)
    });
  }

  // Multiple context values
  getCriticalData(): Observable<any> {
    return this.http.get(`${this.apiUrl}/critical`, {
      context: new HttpContext()
        .set(RETRY_COUNT, 5)
        .set(REQUEST_CONFIG, {
          showLoader: true,
          timeout: 60000,
          errorHandler: 'global'
        })
    });
  }

  // Context with chaining
  getSilentData(): Observable<any> {
    const context = new HttpContext()
      .set(SKIP_AUTH, false)
      .set(REQUEST_CONFIG, {
        showLoader: false,
        errorHandler: 'silent'
      });

    return this.http.get(`${this.apiUrl}/silent`, { context });
  }

  // Reusable context
  private getCachedContext(cacheKey: string): HttpContext {
    return new HttpContext()
      .set(USE_CACHE, true)
      .set(CACHE_KEY, cacheKey)
      .set(REQUEST_CONFIG, {
        showLoader: false,
        cacheDuration: 300000 // 5 minutes
      });
  }

  getCachedUsers(): Observable<User[]> {
    return this.http.get<User[]>(`${this.apiUrl}/users`, {
      context: this.getCachedContext('users-list')
    });
  }
}
```

### 3. Reading Context in Interceptors

Interceptors read context values to conditionally apply logic.

**Context-Aware Interceptors:**

```typescript
// auth.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { SKIP_AUTH } from './http-context-tokens';
import { AuthService } from '../services/auth.service';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  // Read context value
  const skipAuth = req.context.get(SKIP_AUTH);

  if (skipAuth) {
    console.log('Skipping authentication for:', req.url);
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

// retry.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { retry, retryWhen, mergeMap, throwError, timer } from 'rxjs';
import { RETRY_COUNT } from './http-context-tokens';

export const retryInterceptor: HttpInterceptorFn = (req, next) => {
  const maxRetries = req.context.get(RETRY_COUNT);

  return next(req).pipe(
    retryWhen(errors =>
      errors.pipe(
        mergeMap((error: HttpErrorResponse, index) => {
          if (index >= maxRetries) {
            return throwError(() => error);
          }

          // Only retry on specific status codes
          const retryableStatuses = [408, 429, 500, 502, 503, 504];
          if (!retryableStatuses.includes(error.status)) {
            return throwError(() => error);
          }

          console.log(`Retry attempt ${index + 1}/${maxRetries}`);
          return timer(Math.pow(2, index) * 1000);
        })
      )
    )
  );
};

// loader.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { finalize } from 'rxjs/operators';
import { LoaderService } from '../services/loader.service';
import { REQUEST_CONFIG } from './http-context-tokens';

export const loaderInterceptor: HttpInterceptorFn = (req, next) => {
  const config = req.context.get(REQUEST_CONFIG);
  const loaderService = inject(LoaderService);

  if (config.showLoader) {
    loaderService.show();
  }

  return next(req).pipe(
    finalize(() => {
      if (config.showLoader) {
        loaderService.hide();
      }
    })
  );
};

// timeout.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { timeout } from 'rxjs/operators';
import { REQUEST_CONFIG } from './http-context-tokens';

export const timeoutInterceptor: HttpInterceptorFn = (req, next) => {
  const config = req.context.get(REQUEST_CONFIG);
  
  return next(req).pipe(
    timeout(config.timeout || 30000)
  );
};
```

### 4. Caching with Context

Use context to control caching behavior per request.

**Cache Interceptor with Context:**

```typescript
// cache-context-tokens.ts
import { HttpContextToken, HttpResponse } from '@angular/common/http';

export const USE_CACHE = new HttpContextToken<boolean>(() => false);
export const CACHE_KEY = new HttpContextToken<string>(() => '');
export const CACHE_TTL = new HttpContextToken<number>(() => 300000); // 5 minutes
export const FORCE_REFRESH = new HttpContextToken<boolean>(() => false);

// cache.interceptor.ts
import { HttpInterceptorFn, HttpResponse } from '@angular/common/http';
import { of, tap } from 'rxjs';
import { USE_CACHE, CACHE_KEY, CACHE_TTL, FORCE_REFRESH } from './cache-context-tokens';

interface CacheEntry {
  response: HttpResponse<any>;
  timestamp: number;
}

const cache = new Map<string, CacheEntry>();

export const cacheInterceptor: HttpInterceptorFn = (req, next) => {
  // Only cache GET requests
  if (req.method !== 'GET') {
    return next(req);
  }

  const useCache = req.context.get(USE_CACHE);
  if (!useCache) {
    return next(req);
  }

  const forceRefresh = req.context.get(FORCE_REFRESH);
  const cacheKey = req.context.get(CACHE_KEY) || req.urlWithParams;
  const ttl = req.context.get(CACHE_TTL);

  // Check cache if not forcing refresh
  if (!forceRefresh) {
    const cached = cache.get(cacheKey);
    if (cached) {
      const age = Date.now() - cached.timestamp;
      if (age < ttl) {
        console.log(`Cache HIT for ${cacheKey} (age: ${age}ms)`);
        return of(cached.response);
      } else {
        console.log(`Cache EXPIRED for ${cacheKey}`);
        cache.delete(cacheKey);
      }
    }
  }

  console.log(`Cache MISS for ${cacheKey}`);
  return next(req).pipe(
    tap(event => {
      if (event instanceof HttpResponse) {
        cache.set(cacheKey, {
          response: event,
          timestamp: Date.now()
        });
      }
    })
  );
};

// Service usage
@Injectable({
  providedIn: 'root'
})
export class ProductService {
  private http = inject(HttpClient);

  getProducts(forceRefresh = false): Observable<Product[]> {
    return this.http.get<Product[]>('/api/products', {
      context: new HttpContext()
        .set(USE_CACHE, true)
        .set(CACHE_KEY, 'products-list')
        .set(CACHE_TTL, 600000) // 10 minutes
        .set(FORCE_REFRESH, forceRefresh)
    });
  }

  getProduct(id: number): Observable<Product> {
    return this.http.get<Product>(`/api/products/${id}`, {
      context: new HttpContext()
        .set(USE_CACHE, true)
        .set(CACHE_KEY, `product-${id}`)
        .set(CACHE_TTL, 300000) // 5 minutes
    });
  }
}
```

### 5. Metadata Passing Between Interceptors

Use context to pass data between interceptors in the chain.

**Inter-Interceptor Communication:**

```typescript
// metadata-tokens.ts
import { HttpContextToken } from '@angular/common/http';

export const REQUEST_ID = new HttpContextToken<string>(() => '');
export const START_TIME = new HttpContextToken<number>(() => 0);
export const REQUEST_METADATA = new HttpContextToken<{
  requestId: string;
  startTime: number;
  userId?: number;
  source: string;
}>(() => ({
  requestId: '',
  startTime: 0,
  source: 'unknown'
}));

// request-id.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { REQUEST_ID, START_TIME } from './metadata-tokens';

export const requestIdInterceptor: HttpInterceptorFn = (req, next) => {
  // Generate request ID
  const requestId = `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  const startTime = Date.now();

  // Clone request with updated context
  const contextReq = req.clone({
    context: req.context
      .set(REQUEST_ID, requestId)
      .set(START_TIME, startTime),
    setHeaders: {
      'X-Request-ID': requestId
    }
  });

  return next(contextReq);
};

// logging.interceptor.ts
import { HttpInterceptorFn, HttpResponse } from '@angular/common/http';
import { tap } from 'rxjs/operators';
import { REQUEST_ID, START_TIME } from './metadata-tokens';

export const loggingInterceptor: HttpInterceptorFn = (req, next) => {
  // Read metadata set by previous interceptor
  const requestId = req.context.get(REQUEST_ID);
  const startTime = req.context.get(START_TIME);

  console.log(`[${requestId}] Request: ${req.method} ${req.url}`);

  return next(req).pipe(
    tap(event => {
      if (event instanceof HttpResponse) {
        const elapsed = Date.now() - startTime;
        console.log(`[${requestId}] Response: ${event.status} (${elapsed}ms)`);
      }
    })
  );
};

// analytics.interceptor.ts
import { HttpInterceptorFn, HttpResponse, HttpErrorResponse } from '@angular/common/http';
import { tap, catchError } from 'rxjs/operators';
import { inject } from '@angular/core';
import { REQUEST_ID, START_TIME } from './metadata-tokens';
import { AnalyticsService } from '../services/analytics.service';

export const analyticsInterceptor: HttpInterceptorFn = (req, next) => {
  const analytics = inject(AnalyticsService);
  const requestId = req.context.get(REQUEST_ID);
  const startTime = req.context.get(START_TIME);

  return next(req).pipe(
    tap(event => {
      if (event instanceof HttpResponse) {
        analytics.trackHttpRequest({
          requestId,
          method: req.method,
          url: req.url,
          status: event.status,
          duration: Date.now() - startTime,
          success: true
        });
      }
    }),
    catchError((error: HttpErrorResponse) => {
      analytics.trackHttpRequest({
        requestId,
        method: req.method,
        url: req.url,
        status: error.status,
        duration: Date.now() - startTime,
        success: false,
        error: error.message
      });
      throw error;
    })
  );
};
```

### 6. Custom Error Handling with Context

Use context to specify custom error handlers per request.

**Context-Based Error Handling:**

```typescript
// error-handler-tokens.ts
import { HttpContextToken, HttpErrorResponse } from '@angular/common/http';

export type ErrorHandlerType = 'default' | 'silent' | 'custom' | 'notification';

export const ERROR_HANDLER_TYPE = new HttpContextToken<ErrorHandlerType>(() => 'default');

export const CUSTOM_ERROR_HANDLER = new HttpContextToken<
  (error: HttpErrorResponse) => void
>(() => () => {});

export const ERROR_MESSAGE_OVERRIDE = new HttpContextToken<string>(() => '');

// error-handler.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { catchError, throwError } from 'rxjs';
import { inject } from '@angular/core';
import { 
  ERROR_HANDLER_TYPE, 
  CUSTOM_ERROR_HANDLER, 
  ERROR_MESSAGE_OVERRIDE 
} from './error-handler-tokens';
import { NotificationService } from '../services/notification.service';
import { LoggingService } from '../services/logging.service';

export const errorHandlerInterceptor: HttpInterceptorFn = (req, next) => {
  const notificationService = inject(NotificationService);
  const loggingService = inject(LoggingService);
  
  const handlerType = req.context.get(ERROR_HANDLER_TYPE);
  const customHandler = req.context.get(CUSTOM_ERROR_HANDLER);
  const messageOverride = req.context.get(ERROR_MESSAGE_OVERRIDE);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      switch (handlerType) {
        case 'silent':
          // Only log, don't show to user
          loggingService.logError(error);
          break;

        case 'custom':
          // Use custom handler
          customHandler(error);
          break;

        case 'notification':
          // Show notification with override or default message
          const message = messageOverride || getErrorMessage(error);
          notificationService.error(message);
          loggingService.logError(error);
          break;

        case 'default':
        default:
          // Default error handling
          loggingService.logError(error);
          console.error('HTTP Error:', error);
          break;
      }

      return throwError(() => error);
    })
  );
};

function getErrorMessage(error: HttpErrorResponse): string {
  if (error.status === 0) {
    return 'Network error. Please check your connection.';
  }
  return error.error?.message || `Error ${error.status}: ${error.statusText}`;
}

// Service usage
@Injectable({
  providedIn: 'root'
})
export class UserService {
  private http = inject(HttpClient);

  // Silent error handling
  checkUserExists(email: string): Observable<boolean> {
    return this.http.get<boolean>(`/api/users/exists?email=${email}`, {
      context: new HttpContext().set(ERROR_HANDLER_TYPE, 'silent')
    });
  }

  // Custom error handler
  deleteUser(id: number, onError: (error: HttpErrorResponse) => void): Observable<void> {
    return this.http.delete<void>(`/api/users/${id}`, {
      context: new HttpContext()
        .set(ERROR_HANDLER_TYPE, 'custom')
        .set(CUSTOM_ERROR_HANDLER, onError)
    });
  }

  // Custom error message
  createUser(user: any): Observable<User> {
    return this.http.post<User>('/api/users', user, {
      context: new HttpContext()
        .set(ERROR_HANDLER_TYPE, 'notification')
        .set(ERROR_MESSAGE_OVERRIDE, 'Failed to create user. Please try again.')
    });
  }
}
```

### 7. Token-Based Context Configuration

Create configuration presets using context tokens.

**Context Configuration Presets:**

```typescript
// context-presets.ts
import { HttpContext } from '@angular/common/http';
import {
  SKIP_AUTH,
  USE_CACHE,
  RETRY_COUNT,
  REQUEST_CONFIG,
  ERROR_HANDLER_TYPE
} from './http-context-tokens';

export class HttpContextPresets {
  // Public endpoint (no auth, cached)
  static public(): HttpContext {
    return new HttpContext()
      .set(SKIP_AUTH, true)
      .set(USE_CACHE, true)
      .set(REQUEST_CONFIG, {
        showLoader: false,
        errorHandler: 'silent'
      });
  }

  // Critical operation (auth, retry, show loader)
  static critical(): HttpContext {
    return new HttpContext()
      .set(SKIP_AUTH, false)
      .set(RETRY_COUNT, 5)
      .set(REQUEST_CONFIG, {
        showLoader: true,
        timeout: 60000,
        errorHandler: 'global'
      });
  }

  // Background task (no loader, silent errors)
  static background(): HttpContext {
    return new HttpContext()
      .set(REQUEST_CONFIG, {
        showLoader: false,
        errorHandler: 'silent'
      })
      .set(RETRY_COUNT, 3);
  }

  // Cached data with TTL
  static cached(key: string, ttl = 300000): HttpContext {
    return new HttpContext()
      .set(USE_CACHE, true)
      .set(CACHE_KEY, key)
      .set(CACHE_TTL, ttl)
      .set(REQUEST_CONFIG, {
        showLoader: false
      });
  }

  // Real-time data (no cache, show loader)
  static realtime(): HttpContext {
    return new HttpContext()
      .set(USE_CACHE, false)
      .set(REQUEST_CONFIG, {
        showLoader: true,
        timeout: 10000
      });
  }

  // Custom preset builder
  static custom(options: {
    skipAuth?: boolean;
    useCache?: boolean;
    retryCount?: number;
    showLoader?: boolean;
    errorHandler?: 'default' | 'silent' | 'notification';
  }): HttpContext {
    const context = new HttpContext();

    if (options.skipAuth !== undefined) {
      context.set(SKIP_AUTH, options.skipAuth);
    }
    if (options.useCache !== undefined) {
      context.set(USE_CACHE, options.useCache);
    }
    if (options.retryCount !== undefined) {
      context.set(RETRY_COUNT, options.retryCount);
    }

    context.set(REQUEST_CONFIG, {
      showLoader: options.showLoader ?? true,
      errorHandler: options.errorHandler ?? 'default'
    });

    return context;
  }
}

// Service usage
@Injectable({
  providedIn: 'root'
})
export class DataService {
  private http = inject(HttpClient);

  getPublicData(): Observable<any> {
    return this.http.get('/api/public/data', {
      context: HttpContextPresets.public()
    });
  }

  getCriticalData(): Observable<any> {
    return this.http.get('/api/critical', {
      context: HttpContextPresets.critical()
    });
  }

  syncBackgroundData(): Observable<any> {
    return this.http.post('/api/sync', {}, {
      context: HttpContextPresets.background()
    });
  }

  getCachedUsers(): Observable<User[]> {
    return this.http.get<User[]>('/api/users', {
      context: HttpContextPresets.cached('users-list', 600000)
    });
  }

  getRealtimeData(): Observable<any> {
    return this.http.get('/api/realtime', {
      context: HttpContextPresets.realtime()
    });
  }

  customRequest(): Observable<any> {
    return this.http.get('/api/custom', {
      context: HttpContextPresets.custom({
        skipAuth: false,
        useCache: true,
        retryCount: 2,
        showLoader: false,
        errorHandler: 'silent'
      })
    });
  }
}
```

### 8. Debugging with Context

Use context for debugging and tracing requests.

**Debug Context:**

```typescript
// debug-tokens.ts
import { HttpContextToken } from '@angular/common/http';

export const DEBUG_REQUEST = new HttpContextToken<boolean>(() => false);
export const DEBUG_LABEL = new HttpContextToken<string>(() => '');
export const TRACE_REQUEST = new HttpContextToken<boolean>(() => false);

// debug.interceptor.ts
import { HttpInterceptorFn, HttpResponse, HttpEvent } from '@angular/common/http';
import { tap, finalize } from 'rxjs/operators';
import { DEBUG_REQUEST, DEBUG_LABEL, TRACE_REQUEST } from './debug-tokens';

export const debugInterceptor: HttpInterceptorFn = (req, next) => {
  const shouldDebug = req.context.get(DEBUG_REQUEST);
  const debugLabel = req.context.get(DEBUG_LABEL);
  const shouldTrace = req.context.get(TRACE_REQUEST);

  if (!shouldDebug && !shouldTrace) {
    return next(req);
  }

  const label = debugLabel || req.url;
  const startTime = performance.now();

  console.group(`🔍 ${label}`);
  console.log('Method:', req.method);
  console.log('URL:', req.urlWithParams);
  console.log('Headers:', req.headers.keys().map(k => `${k}: ${req.headers.get(k)}`));
  console.log('Body:', req.body);

  if (shouldTrace) {
    console.trace('Request stack trace');
  }

  return next(req).pipe(
    tap((event: HttpEvent<any>) => {
      if (event instanceof HttpResponse) {
        const elapsed = performance.now() - startTime;
        console.log('Status:', event.status);
        console.log('Response Headers:', 
          event.headers.keys().map(k => `${k}: ${event.headers.get(k)}`)
        );
        console.log('Response Body:', event.body);
        console.log(`⏱️ Duration: ${elapsed.toFixed(2)}ms`);
      }
    }),
    finalize(() => {
      console.groupEnd();
    })
  );
};

// Development helper service
@Injectable({
  providedIn: 'root'
})
export class DebugHttpService {
  private http = inject(HttpClient);

  // Wrapper for debugging requests
  debugGet<T>(url: string, label?: string): Observable<T> {
    return this.http.get<T>(url, {
      context: new HttpContext()
        .set(DEBUG_REQUEST, true)
        .set(DEBUG_LABEL, label || url)
    });
  }

  debugPost<T>(url: string, body: any, label?: string): Observable<T> {
    return this.http.post<T>(url, body, {
      context: new HttpContext()
        .set(DEBUG_REQUEST, true)
        .set(DEBUG_LABEL, label || url)
        .set(TRACE_REQUEST, true)
    });
  }
}
```

### 9. Performance Monitoring with Context

Track performance metrics using context metadata.

**Performance Context:**

```typescript
// performance-tokens.ts
import { HttpContextToken } from '@angular/common/http';

export const TRACK_PERFORMANCE = new HttpContextToken<boolean>(() => false);
export const PERFORMANCE_LABEL = new HttpContextToken<string>(() => '');
export const SLOW_REQUEST_THRESHOLD = new HttpContextToken<number>(() => 3000);

// performance.interceptor.ts
import { HttpInterceptorFn, HttpResponse } from '@angular/common/http';
import { tap, finalize } from 'rxjs/operators';
import { inject } from '@angular/core';
import { 
  TRACK_PERFORMANCE, 
  PERFORMANCE_LABEL, 
  SLOW_REQUEST_THRESHOLD 
} from './performance-tokens';
import { PerformanceService } from '../services/performance.service';

export const performanceInterceptor: HttpInterceptorFn = (req, next) => {
  const shouldTrack = req.context.get(TRACK_PERFORMANCE);
  
  if (!shouldTrack) {
    return next(req);
  }

  const performanceService = inject(PerformanceService);
  const label = req.context.get(PERFORMANCE_LABEL) || req.url;
  const slowThreshold = req.context.get(SLOW_REQUEST_THRESHOLD);
  
  const startTime = performance.now();
  const markStart = `http-start-${label}`;
  const markEnd = `http-end-${label}`;
  
  performance.mark(markStart);

  return next(req).pipe(
    tap(event => {
      if (event instanceof HttpResponse) {
        performance.mark(markEnd);
        performance.measure(label, markStart, markEnd);
        
        const endTime = performance.now();
        const duration = endTime - startTime;

        performanceService.recordHttpMetric({
          label,
          url: req.url,
          method: req.method,
          status: event.status,
          duration,
          timestamp: Date.now()
        });

        if (duration > slowThreshold) {
          console.warn(`⚠️ Slow request detected: ${label} took ${duration.toFixed(2)}ms`);
          performanceService.recordSlowRequest({
            label,
            url: req.url,
            duration
          });
        }
      }
    }),
    finalize(() => {
      performance.clearMarks(markStart);
      performance.clearMarks(markEnd);
      performance.clearMeasures(label);
    })
  );
};
```

### 10. Testing with Context

Test HTTP requests with specific context configurations.

**Testing Context:**

```typescript
// data.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';
import { HttpContext } from '@angular/common/http';
import { DataService } from './data.service';
import { SKIP_AUTH, USE_CACHE } from './http-context-tokens';

describe('DataService', () => {
  let service: DataService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [DataService]
    });

    service = TestBed.inject(DataService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('should set SKIP_AUTH context for public data', () => {
    service.getPublicData().subscribe();

    const req = httpMock.expectOne('/api/public/data');
    
    // Verify context was set
    expect(req.request.context.get(SKIP_AUTH)).toBe(true);
    
    req.flush({ data: 'test' });
  });

  it('should set USE_CACHE context for cached requests', () => {
    service.getCachedData().subscribe();

    const req = httpMock.expectOne('/api/cached');
    
    expect(req.request.context.get(USE_CACHE)).toBe(true);
    
    req.flush({ data: 'cached' });
  });

  it('should not set context for regular requests', () => {
    service.getRegularData().subscribe();

    const req = httpMock.expectOne('/api/data');
    
    expect(req.request.context.get(SKIP_AUTH)).toBe(false);
    expect(req.request.context.get(USE_CACHE)).toBe(false);
    
    req.flush({ data: 'regular' });
  });
});

// Interceptor testing with context
describe('authInterceptor', () => {
  it('should skip authentication when SKIP_AUTH is true', (done) => {
    const request = new HttpRequest('GET', '/api/public', {
      context: new HttpContext().set(SKIP_AUTH, true)
    });

    const next: HttpHandler = {
      handle: (req: HttpRequest<any>) => {
        expect(req.headers.has('Authorization')).toBe(false);
        return of(new HttpResponse({ status: 200 }));
      }
    };

    TestBed.runInInjectionContext(() => {
      authInterceptor(request, next.handle).subscribe(() => done());
    });
  });
});
```

## Common Mistakes

### 1. Not Providing Default Values

```typescript
// WRONG: No default value
export const MY_TOKEN = new HttpContextToken<string>();

// CORRECT: Always provide default
export const MY_TOKEN = new HttpContextToken<string>(() => '');
```

### 2. Mutating Context

```typescript
// WRONG: Cannot mutate context
const context = new HttpContext();
context.set(TOKEN, value); // Returns new context!

// CORRECT: Chain or reassign
const context = new HttpContext()
  .set(TOKEN1, value1)
  .set(TOKEN2, value2);
```

### 3. Using Wrong Token Type

```typescript
// WRONG: Type mismatch
const token = new HttpContextToken<boolean>(() => false);
context.set(token, 'string'); // Type error!

// CORRECT: Match types
context.set(token, true);
```

### 4. Not Reading Context in Interceptor

```typescript
// WRONG: Hardcoded behavior
export const interceptor: HttpInterceptorFn = (req, next) => {
  // Always does the same thing
  return next(req);
};

// CORRECT: Check context
export const interceptor: HttpInterceptorFn = (req, next) => {
  if (req.context.get(MY_TOKEN)) {
    // Conditional behavior
  }
  return next(req);
};
```

### 5. Forgetting to Pass Context

```typescript
// WRONG: Context not used
this.http.get('/api/data'); // Uses defaults

// CORRECT: Pass context
this.http.get('/api/data', {
  context: new HttpContext().set(TOKEN, value)
});
```

## Best Practices

### 1. Define Tokens in Separate File

```typescript
// http-context-tokens.ts
export const SKIP_AUTH = new HttpContextToken<boolean>(() => false);
export const USE_CACHE = new HttpContextToken<boolean>(() => false);
```

### 2. Use Type-Safe Tokens

```typescript
// Typed token with interface
interface RequestConfig {
  showLoader: boolean;
  timeout: number;
}

export const REQUEST_CONFIG = new HttpContextToken<RequestConfig>(
  () => ({ showLoader: true, timeout: 30000 })
);
```

### 3. Create Context Presets

```typescript
// Reusable context configurations
export class HttpContextPresets {
  static public() { /* ... */ }
  static critical() { /* ... */ }
}
```

### 4. Document Token Purpose

```typescript
/**
 * Skips authentication for public endpoints.
 * Default: false (authentication required)
 */
export const SKIP_AUTH = new HttpContextToken<boolean>(() => false);
```

### 5. Check Context in Interceptors

```typescript
// Always check context before applying logic
export const myInterceptor: HttpInterceptorFn = (req, next) => {
  const shouldApply = req.context.get(MY_TOKEN);
  if (!shouldApply) {
    return next(req);
  }
  // Apply logic
};
```

## Interview Questions

**Q: What is HttpContext used for?**

A: `HttpContext` passes metadata through the HTTP request/response pipeline using typed tokens. It allows interceptors to coordinate behavior without modifying the request or inspecting URLs/headers.

**Q: How do HttpContextTokens work?**

A: `HttpContextToken<T>` is a typed key for storing values in `HttpContext`. You create tokens with default values: `new HttpContextToken<T>(() => defaultValue)`, then set/get values using `context.set(token, value)` and `context.get(token)`.

**Q: Can you modify HttpContext after creating it?**

A: No, `HttpContext` is immutable. Methods like `set()` return a new context instance. Always chain calls or reassign: `context = context.set(TOKEN, value)`.

**Q: How do interceptors read context values?**

A: Interceptors access context via `req.context.get(TOKEN)` to read values set when making the HTTP request, allowing conditional logic based on context.

**Q: What's the advantage of HttpContext over custom headers?**

A: Context is type-safe, doesn't pollute HTTP headers, stays internal to Angular, supports complex types (objects, functions), and provides cleaner separation of concerns.

## Key Takeaways

1. `HttpContext` passes metadata through HTTP pipeline without modifying requests
2. `HttpContextToken<T>` provides type-safe keys with default value factories
3. Create context with `new HttpContext().set(TOKEN, value)` when making requests
4. Interceptors read context via `req.context.get(TOKEN)` for conditional logic
5. Context is immutable - `set()` returns new instance, use chaining
6. Use context for authentication skipping, caching, retry configuration, error handling
7. Define tokens in separate files with descriptive names and documentation
8. Create reusable context presets for common request patterns
9. Context enables inter-interceptor communication without side effects
10. Test context values in HTTP tests using HttpTestingController

## Resources

- [Angular HttpContext Documentation](https://angular.io/api/common/http/HttpContext)
- [HttpContextToken API](https://angular.io/api/common/http/HttpContextToken)
- [HTTP Interceptors with Context](https://angular.io/guide/http-interceptors#http-context)
- [Testing HTTP Requests](https://angular.io/guide/http-test-requests)
- [Angular HTTP Guide](https://angular.io/guide/understanding-communicating-with-http)
