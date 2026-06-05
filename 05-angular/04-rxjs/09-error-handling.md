# RxJS Error Handling

## The Idea

**In plain English:** Error handling in RxJS is a way to decide what your app should do when something goes wrong while waiting for data — like a failed internet request — so the app can recover gracefully instead of crashing. An "observable" is just a stream of data your app listens to over time.

**Real-world analogy:** Imagine you order a pizza and the delivery driver calls to say they got lost. You have options: ask them to try again (retry), accept a backup pizza from the freezer (fallback value), or cancel the whole order and tell someone (re-throw the error).

- The delivery driver getting lost = an error occurring in the observable stream
- Asking the driver to try again = the `retry` operator
- Accepting the freezer pizza = the `catchError` operator returning a fallback value
- Cancelling and notifying someone = re-throwing the error with `throwError`

---

## Table of Contents

- [Introduction](#introduction)
- [catchError](#catcherror)
- [retry](#retry)
- [retryWhen](#retrywhen)
- [Error Handling Strategies](#error-handling-strategies)
- [throwError](#throwerror)
- [EMPTY and of](#empty-and-of)
- [Error Recovery Patterns](#error-recovery-patterns)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Error handling in RxJS is crucial for building resilient Angular applications. Unlike synchronous code where you can use try-catch blocks, observables require specific operators to handle errors gracefully. Understanding these operators and patterns will help you build applications that recover from failures, provide meaningful feedback to users, and maintain stable operation.

Errors in observables are terminal events - once an error occurs, the observable stream completes and stops emitting values. This makes proper error handling essential for long-lived streams like user inputs, WebSocket connections, and polling operations.

## catchError

The `catchError` operator intercepts errors in the observable chain and allows you to recover by returning a new observable. This prevents the error from propagating to subscribers and keeps the stream alive.

### Basic Usage

```typescript
import { catchError } from 'rxjs/operators';
import { of } from 'rxjs';

@Injectable()
export class ProductService {
  constructor(private http: HttpClient) {}

  getProduct(id: string): Observable<Product> {
    return this.http.get<Product>(`/api/products/${id}`).pipe(
      catchError(error => {
        console.error('Failed to load product:', error);
        // Return fallback value
        return of(null);
      })
    );
  }
}
```

### Returning Default Values

```typescript
@Injectable()
export class UserPreferencesService {
  constructor(private http: HttpClient) {}

  private defaultPreferences: UserPreferences = {
    theme: 'light',
    language: 'en',
    notifications: true
  };

  loadPreferences(userId: string): Observable<UserPreferences> {
    return this.http.get<UserPreferences>(`/api/users/${userId}/preferences`).pipe(
      catchError(error => {
        console.warn('Using default preferences:', error.message);
        return of(this.defaultPreferences);
      })
    );
  }
}
```

### Re-throwing Errors

```typescript
@Injectable()
export class AuthService {
  constructor(
    private http: HttpClient,
    private notificationService: NotificationService
  ) {}

  login(credentials: Credentials): Observable<User> {
    return this.http.post<User>('/api/auth/login', credentials).pipe(
      catchError(error => {
        // Log error
        console.error('Login failed:', error);
        
        // Show user-friendly message
        this.notificationService.showError('Login failed. Please try again.');
        
        // Re-throw for component to handle
        return throwError(() => new Error('Authentication failed'));
      })
    );
  }
}
```

### Different Error Types

```typescript
interface ApiError {
  status: number;
  message: string;
  code: string;
}

@Injectable()
export class DataService {
  constructor(private http: HttpClient) {}

  getData(id: string): Observable<Data> {
    return this.http.get<Data>(`/api/data/${id}`).pipe(
      catchError((error: HttpErrorResponse) => {
        if (error.status === 404) {
          console.log('Data not found, using empty state');
          return of(this.getEmptyData());
        }
        
        if (error.status === 403) {
          console.error('Access denied');
          return throwError(() => new Error('You do not have permission to access this data'));
        }
        
        if (error.status === 0) {
          console.error('Network error');
          return throwError(() => new Error('Network connection failed'));
        }
        
        // Generic error
        return throwError(() => new Error('An unexpected error occurred'));
      })
    );
  }
}
```

### Accessing Error Details

```typescript
@Component({
  selector: 'app-data-viewer'
})
export class DataViewerComponent {
  private dataService = inject(DataService);
  
  errorMessage$ = new BehaviorSubject<string>('');
  
  data$ = this.dataService.getData().pipe(
    catchError(error => {
      // Access error details
      const errorMsg = error instanceof HttpErrorResponse
        ? `HTTP ${error.status}: ${error.message}`
        : error.message || 'Unknown error occurred';
      
      this.errorMessage$.next(errorMsg);
      
      // Return empty array to continue stream
      return of([]);
    })
  );
}
```

## retry

The `retry` operator resubscribes to the source observable when an error occurs, attempting the operation again.

### Basic Retry

```typescript
@Injectable()
export class NetworkService {
  constructor(private http: HttpClient) {}

  fetchData(url: string): Observable<any> {
    return this.http.get(url).pipe(
      retry(3), // Retry up to 3 times
      catchError(error => {
        console.error('Failed after 3 retries:', error);
        return of(null);
      })
    );
  }
}
```

### Retry with Configuration (RxJS 7+)

```typescript
import { retry, RetryConfig } from 'rxjs';

@Injectable()
export class ResilientHttpService {
  constructor(private http: HttpClient) {}

  getData<T>(url: string): Observable<T> {
    const retryConfig: RetryConfig = {
      count: 3,           // Number of retry attempts
      delay: 1000,        // Delay between retries (ms)
      resetOnSuccess: true // Reset retry count on success
    };

    return this.http.get<T>(url).pipe(
      retry(retryConfig),
      catchError(error => {
        console.error(`Failed to fetch from ${url} after retries:`, error);
        return throwError(() => error);
      })
    );
  }
}
```

### Exponential Backoff

```typescript
import { timer } from 'rxjs';
import { retryWhen, mergeMap, tap } from 'rxjs/operators';

@Injectable()
export class SmartRetryService {
  constructor(private http: HttpClient) {}

  getDataWithBackoff<T>(url: string, maxRetries: number = 3): Observable<T> {
    let retryCount = 0;

    return this.http.get<T>(url).pipe(
      retryWhen(errors =>
        errors.pipe(
          mergeMap(error => {
            retryCount++;
            
            if (retryCount > maxRetries) {
              return throwError(() => error);
            }
            
            const delayMs = Math.pow(2, retryCount) * 1000; // 2s, 4s, 8s
            console.log(`Retry ${retryCount}/${maxRetries} after ${delayMs}ms`);
            
            return timer(delayMs);
          })
        )
      ),
      catchError(error => {
        console.error('All retries exhausted:', error);
        return throwError(() => error);
      })
    );
  }
}
```

### Conditional Retry

```typescript
@Injectable()
export class ConditionalRetryService {
  constructor(private http: HttpClient) {}

  fetchData(url: string): Observable<any> {
    return this.http.get(url).pipe(
      retryWhen(errors =>
        errors.pipe(
          mergeMap((error, index) => {
            // Only retry on specific errors
            if (this.shouldRetry(error) && index < 3) {
              const delay = (index + 1) * 1000;
              console.log(`Retrying in ${delay}ms (attempt ${index + 1})`);
              return timer(delay);
            }
            
            // Don't retry on 4xx errors (except 429)
            return throwError(() => error);
          })
        )
      )
    );
  }

  private shouldRetry(error: any): boolean {
    if (!(error instanceof HttpErrorResponse)) {
      return true; // Retry network errors
    }
    
    const status = error.status;
    
    // Retry on server errors and rate limiting
    if (status >= 500 || status === 429) {
      return true;
    }
    
    // Don't retry on client errors
    if (status >= 400 && status < 500) {
      return false;
    }
    
    // Retry on timeout and network errors
    return status === 0 || status === 408;
  }
}
```

## retryWhen

The `retryWhen` operator provides more control over retry logic, allowing you to implement custom retry strategies.

### Retry with Delay

```typescript
import { retryWhen, delay, take, concat, throwError } from 'rxjs/operators';

@Injectable()
export class DelayedRetryService {
  constructor(private http: HttpClient) {}

  getData(url: string): Observable<any> {
    return this.http.get(url).pipe(
      retryWhen(errors =>
        errors.pipe(
          delay(2000),  // Wait 2 seconds between retries
          take(3),      // Retry 3 times
          concat(throwError(() => new Error('Max retries reached')))
        )
      )
    );
  }
}
```

### Advanced Retry Strategy

```typescript
interface RetryStrategy {
  maxRetries: number;
  scalingDuration: number;
  excludedStatusCodes: number[];
}

@Injectable()
export class AdvancedRetryService {
  constructor(private http: HttpClient) {}

  private defaultStrategy: RetryStrategy = {
    maxRetries: 3,
    scalingDuration: 1000,
    excludedStatusCodes: [400, 401, 403, 404]
  };

  getWithStrategy<T>(
    url: string,
    strategy: RetryStrategy = this.defaultStrategy
  ): Observable<T> {
    return this.http.get<T>(url).pipe(
      retryWhen(errors =>
        errors.pipe(
          mergeMap((error, attempt) => {
            const retryAttempt = attempt + 1;

            // Check if we should retry
            if (
              retryAttempt > strategy.maxRetries ||
              this.isExcludedError(error, strategy.excludedStatusCodes)
            ) {
              return throwError(() => error);
            }

            const delayDuration = retryAttempt * strategy.scalingDuration;
            
            console.log(
              `Retry attempt ${retryAttempt}/${strategy.maxRetries} after ${delayDuration}ms`
            );

            return timer(delayDuration);
          })
        )
      )
    );
  }

  private isExcludedError(error: any, excludedCodes: number[]): boolean {
    return error instanceof HttpErrorResponse &&
      excludedCodes.includes(error.status);
  }
}
```

## Error Handling Strategies

### Strategy 1: Local Error Handling

```typescript
@Component({
  selector: 'app-product-details'
})
export class ProductDetailsComponent {
  private route = inject(ActivatedRoute);
  private productService = inject(ProductService);
  
  errorMessage = signal<string>('');
  
  product$ = this.route.params.pipe(
    switchMap(params =>
      this.productService.getProduct(params['id']).pipe(
        catchError(error => {
          this.errorMessage.set('Failed to load product');
          return of(null);
        })
      )
    )
  );
}
```

### Strategy 2: Global Error Handler

```typescript
@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  constructor(
    private notificationService: NotificationService,
    private loggingService: LoggingService
  ) {}

  handleError(error: Error | HttpErrorResponse): void {
    if (error instanceof HttpErrorResponse) {
      // Server or network error
      if (!navigator.onLine) {
        this.notificationService.showError('No internet connection');
      } else {
        this.notificationService.showError(
          this.getServerErrorMessage(error)
        );
      }
    } else {
      // Client-side error
      this.notificationService.showError('An error occurred');
    }

    // Log all errors
    this.loggingService.logError(error);
  }

  private getServerErrorMessage(error: HttpErrorResponse): string {
    switch (error.status) {
      case 404:
        return 'Resource not found';
      case 403:
        return 'Access denied';
      case 500:
        return 'Internal server error';
      default:
        return 'An error occurred';
    }
  }
}

// Provide in app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    { provide: ErrorHandler, useClass: GlobalErrorHandler }
  ]
};
```

### Strategy 3: HTTP Interceptor Error Handling

```typescript
@Injectable()
export class ErrorInterceptor implements HttpInterceptor {
  constructor(
    private notificationService: NotificationService,
    private authService: AuthService,
    private router: Router
  ) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    return next.handle(req).pipe(
      catchError((error: HttpErrorResponse) => {
        // Handle specific error statuses globally
        switch (error.status) {
          case 401:
            // Unauthorized - redirect to login
            this.authService.logout();
            this.router.navigate(['/login']);
            this.notificationService.showError('Session expired. Please login again.');
            break;

          case 403:
            // Forbidden
            this.notificationService.showError('Access denied');
            this.router.navigate(['/']);
            break;

          case 429:
            // Rate limited
            this.notificationService.showWarning('Too many requests. Please slow down.');
            break;

          case 500:
          case 502:
          case 503:
            // Server errors
            this.notificationService.showError('Server error. Please try again later.');
            break;

          case 0:
            // Network error
            this.notificationService.showError('Network error. Please check your connection.');
            break;
        }

        return throwError(() => error);
      })
    );
  }
}
```

### Strategy 4: Graceful Degradation

```typescript
@Injectable()
export class DataService {
  constructor(
    private http: HttpClient,
    private cacheService: CacheService
  ) {}

  getData(id: string): Observable<Data> {
    return this.http.get<Data>(`/api/data/${id}`).pipe(
      tap(data => this.cacheService.set(id, data)),
      catchError(error => {
        console.warn('API failed, trying cache:', error);
        
        const cached = this.cacheService.get(id);
        if (cached) {
          console.log('Returning cached data');
          return of(cached);
        }
        
        console.error('No cached data available');
        return throwError(() => error);
      })
    );
  }
}
```

### Strategy 5: Partial Error Handling

```typescript
@Injectable()
export class BatchDataService {
  constructor(private http: HttpClient) {}

  loadMultipleResources(ids: string[]): Observable<ResourceResult[]> {
    const requests = ids.map(id =>
      this.http.get<Resource>(`/api/resources/${id}`).pipe(
        map(resource => ({ id, success: true, data: resource, error: null })),
        catchError(error => of({ id, success: false, data: null, error: error.message }))
      )
    );

    return forkJoin(requests).pipe(
      map(results => {
        const successful = results.filter(r => r.success);
        const failed = results.filter(r => !r.success);
        
        console.log(`Loaded ${successful.length}/${ids.length} resources`);
        if (failed.length > 0) {
          console.warn('Failed to load:', failed);
        }
        
        return results;
      })
    );
  }
}
```

## throwError

Create an observable that immediately errors.

### Basic Usage

```typescript
import { throwError } from 'rxjs';

@Injectable()
export class ValidationService {
  validateAndFetch(id: string): Observable<Data> {
    if (!id || id.trim() === '') {
      return throwError(() => new Error('ID is required'));
    }

    if (id.length < 3) {
      return throwError(() => new Error('ID must be at least 3 characters'));
    }

    return this.http.get<Data>(`/api/data/${id}`);
  }
}
```

### Custom Error Types

```typescript
class ValidationError extends Error {
  constructor(message: string, public field: string) {
    super(message);
    this.name = 'ValidationError';
  }
}

class AuthorizationError extends Error {
  constructor(message: string, public requiredRole: string) {
    super(message);
    this.name = 'AuthorizationError';
  }
}

@Injectable()
export class SecureDataService {
  constructor(
    private http: HttpClient,
    private authService: AuthService
  ) {}

  getSecureData(id: string): Observable<SecureData> {
    const user = this.authService.getCurrentUser();

    if (!user) {
      return throwError(() => new AuthorizationError(
        'User not authenticated',
        'user'
      ));
    }

    if (!user.roles.includes('admin')) {
      return throwError(() => new AuthorizationError(
        'Admin role required',
        'admin'
      ));
    }

    return this.http.get<SecureData>(`/api/secure/${id}`);
  }
}
```

## EMPTY and of

Use `EMPTY` to complete without emitting values, or `of` to emit specific values then complete.

### Using EMPTY

```typescript
import { EMPTY } from 'rxjs';

@Component({
  selector: 'app-conditional-loader'
})
export class ConditionalLoaderComponent {
  private dataService = inject(DataService);
  private authService = inject(AuthService);

  data$ = this.authService.isAuthenticated$.pipe(
    switchMap(isAuthenticated => {
      if (!isAuthenticated) {
        console.log('User not authenticated, skipping data load');
        return EMPTY; // Complete without emitting
      }
      return this.dataService.getData();
    })
  );
}
```

### Using of

```typescript
import { of } from 'rxjs';

@Injectable()
export class CacheFirstService {
  constructor(
    private http: HttpClient,
    private cache: CacheService
  ) {}

  getData(id: string): Observable<Data> {
    const cached = this.cache.get(id);
    
    if (cached) {
      console.log('Returning cached data');
      return of(cached); // Emit cached value immediately
    }

    return this.http.get<Data>(`/api/data/${id}`).pipe(
      tap(data => this.cache.set(id, data))
    );
  }
}
```

## Error Recovery Patterns

### Pattern 1: Retry Then Fallback

```typescript
@Injectable()
export class ResilientService {
  constructor(private http: HttpClient) {}

  getData(id: string): Observable<Data> {
    return this.http.get<Data>(`/api/data/${id}`).pipe(
      retry({
        count: 2,
        delay: 1000
      }),
      catchError(error => {
        console.warn('API failed, using fallback');
        return this.getFallbackData(id);
      })
    );
  }

  private getFallbackData(id: string): Observable<Data> {
    return of({
      id,
      name: 'Fallback Data',
      status: 'offline'
    });
  }
}
```

### Pattern 2: Continue on Error

```typescript
@Component({
  selector: 'app-live-feed'
})
export class LiveFeedComponent {
  private feedService = inject(FeedService);

  // Keep stream alive even if individual items fail
  feed$ = this.feedService.feedUpdates$.pipe(
    mergeMap(item =>
      this.processItem(item).pipe(
        catchError(error => {
          console.error('Failed to process item:', error);
          return EMPTY; // Skip failed item, continue stream
        })
      )
    )
  );

  private processItem(item: FeedItem): Observable<ProcessedItem> {
    return this.feedService.process(item);
  }
}
```

### Pattern 3: Circuit Breaker

```typescript
@Injectable()
export class CircuitBreakerService {
  private failureCount = 0;
  private readonly threshold = 3;
  private readonly resetTimeout = 60000; // 1 minute
  private circuitOpen = false;

  constructor(private http: HttpClient) {}

  getData(url: string): Observable<any> {
    if (this.circuitOpen) {
      return throwError(() => 
        new Error('Circuit breaker is open. Service temporarily unavailable.')
      );
    }

    return this.http.get(url).pipe(
      tap(() => this.onSuccess()),
      catchError(error => this.onError(error))
    );
  }

  private onSuccess(): void {
    this.failureCount = 0;
  }

  private onError(error: any): Observable<never> {
    this.failureCount++;

    if (this.failureCount >= this.threshold) {
      this.openCircuit();
    }

    return throwError(() => error);
  }

  private openCircuit(): void {
    this.circuitOpen = true;
    console.warn('Circuit breaker opened');

    setTimeout(() => {
      this.circuitOpen = false;
      this.failureCount = 0;
      console.log('Circuit breaker reset');
    }, this.resetTimeout);
  }
}
```

## Common Mistakes

### 1. Not Catching Errors

```typescript
// WRONG: Error propagates to subscriber
this.http.get('/api/data').subscribe(data => {
  this.data = data;
});

// RIGHT: Handle errors
this.http.get('/api/data').pipe(
  catchError(error => {
    console.error('Failed:', error);
    return of(null);
  })
).subscribe(data => {
  this.data = data;
});
```

### 2. Catching Too Early

```typescript
// WRONG: Can't retry if error is caught
this.http.get('/api/data').pipe(
  catchError(() => of(null)),
  retry(3) // Won't retry - error already caught!
)

// RIGHT: Retry before catching
this.http.get('/api/data').pipe(
  retry(3),
  catchError(() => of(null))
)
```

### 3. Not Re-throwing When Needed

```typescript
// WRONG: Component doesn't know about error
service.pipe(
  catchError(error => {
    console.error(error);
    return of(null); // Swallows error
  })
)

// RIGHT: Re-throw if component should handle it
service.pipe(
  catchError(error => {
    this.logError(error);
    return throwError(() => error); // Component can handle
  })
)
```

### 4. Infinite Retry Loops

```typescript
// WRONG: Will retry forever
this.http.get('/api/data').pipe(
  retryWhen(errors => errors.pipe(delay(1000)))
)

// RIGHT: Limit retries
this.http.get('/api/data').pipe(
  retryWhen(errors =>
    errors.pipe(
      delay(1000),
      take(3)
    )
  )
)
```

## Best Practices

### 1. Layer Error Handling

```typescript
// Service layer - handle generic errors
@Injectable()
export class DataService {
  getData(): Observable<Data> {
    return this.http.get<Data>('/api/data').pipe(
      retry(2),
      catchError(error => {
        this.logError(error);
        return throwError(() => error);
      })
    );
  }
}

// Component layer - handle UI-specific concerns
@Component({})
export class DataComponent {
  data$ = this.dataService.getData().pipe(
    catchError(error => {
      this.showUserMessage('Failed to load data');
      return of(null);
    })
  );
}
```

### 2. Provide Context in Errors

```typescript
class DetailedError extends Error {
  constructor(
    message: string,
    public context: {
      url: string;
      params: any;
      timestamp: Date;
    }
  ) {
    super(message);
  }
}

catchError((error: HttpErrorResponse) => {
  return throwError(() => new DetailedError(
    'Data fetch failed',
    {
      url: error.url || '',
      params: { id },
      timestamp: new Date()
    }
  ));
})
```

### 3. Use Type Guards

```typescript
function isHttpError(error: any): error is HttpErrorResponse {
  return error instanceof HttpErrorResponse;
}

catchError(error => {
  if (isHttpError(error)) {
    // TypeScript knows error is HttpErrorResponse
    console.log('Status:', error.status);
  }
  return throwError(() => error);
})
```

## Interview Questions

### Q1: What's the difference between catchError and retry?

**Answer:**
- **catchError**: Handles errors by returning a new observable. Use to provide fallback values or transform errors.
- **retry**: Resubscribes to source observable on error. Use to automatically attempt failed operations again.

They work together: retry first, then catchError as last resort.

### Q2: Why must catchError return an observable?

**Answer:** Because observables are lazy and composable. Returning an observable allows the stream to continue, lets subscribers receive values, and maintains the reactive chain. Use `of()`, `throwError()`, or `EMPTY` based on desired behavior.

### Q3: How do you prevent memory leaks with error handling?

**Answer:** Use `takeUntilDestroyed()` or `takeUntil()` before error operators to ensure cleanup even if errors occur. Errors complete the stream, but you still need to unsubscribe from outer streams in components.

## Key Takeaways

1. **catchError** - Handle errors by returning new observable
2. **retry** - Automatically retry failed operations
3. **retryWhen** - Custom retry logic with conditions
4. **throwError** - Create error observable
5. Errors are terminal - they complete the stream
6. Order matters: retry before catchError
7. Layer error handling (service + component)
8. Always provide user feedback on errors
9. Use exponential backoff for retries
10. Consider circuit breaker pattern for failing services

## Resources

- [RxJS Documentation - Error Handling](https://rxjs.dev/guide/operators#error-handling-operators)
- [Angular HTTP Error Handling](https://angular.io/guide/http#error-handling)
- [Learn RxJS - Error Handling](https://www.learnrxjs.io/learn-rxjs/operators/error_handling)
- [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html)
