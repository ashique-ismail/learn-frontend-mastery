# HTTP Error Handling

## The Idea

**In plain English:** HTTP error handling is the process of detecting when something goes wrong with a network request (like asking a server for data and getting no response, or getting told "not allowed") and then deciding what to do next instead of just crashing or showing a blank screen. "HTTP" is just the set of rules computers use to talk to each other over the internet, and "error handling" means writing code that gracefully deals with those failures.

**Real-world analogy:** Imagine you order a pizza over the phone. Sometimes the line is dead (no connection), sometimes they say "we don't deliver to your area" (access denied), and sometimes they say "sorry, that pizza is off the menu" (not found). A good customer doesn't just hang up and stare at the wall — they ask again, order something else, or call a different restaurant.

- The phone call = the HTTP request sent to a server
- The pizza shop's reply = the server's response (or error code)
- Asking again = the retry logic in the code
- Ordering something else = returning a fallback value when the request fails
- Calling a different restaurant = switching to a backup endpoint

---

## Overview

Robust error handling is critical for production Angular applications. HTTP errors can occur due to network failures, server errors, invalid requests, or client-side issues. Angular's HttpClient provides comprehensive error handling capabilities through RxJS operators, the `HttpErrorResponse` type, and global error handling strategies.

Effective error handling includes distinguishing between client and server errors, implementing retry logic for transient failures, providing user-friendly error messages, logging errors for debugging, and gracefully degrading functionality when services are unavailable.

## Core Concepts

### 1. HttpErrorResponse

The `HttpErrorResponse` class represents HTTP error responses with detailed error information.

**HttpErrorResponse Structure:**

```typescript
// error-types.ts
import { HttpErrorResponse } from '@angular/common/http';

interface HttpErrorResponse {
  // HTTP status code (404, 500, etc.)
  status: number;
  
  // Status text ('Not Found', 'Internal Server Error', etc.)
  statusText: string;
  
  // Request URL
  url: string | null;
  
  // Whether error is client-side (network error) or server-side
  error: any | ErrorEvent;
  
  // Full HTTP headers
  headers: HttpHeaders;
  
  // Error message
  message: string;
  
  // Name of the error
  name: string;
}

// user.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

interface User {
  id: number;
  name: string;
  email: string;
}

@Injectable({
  providedIn: 'root'
})
export class UserService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com/users';

  getUser(id: number): Observable<User> {
    return this.http.get<User>(`${this.apiUrl}/${id}`).pipe(
      catchError((error: HttpErrorResponse) => {
        // Check if it's a client-side error
        if (error.error instanceof ErrorEvent) {
          // Client-side or network error
          console.error('Client-side error:', error.error.message);
        } else {
          // Server-side error
          console.error(
            `Server error ${error.status}: ${error.message}`,
            `Body: ${JSON.stringify(error.error)}`
          );
        }

        // Return error observable
        return throwError(() => error);
      })
    );
  }

  // Detailed error analysis
  analyzeError(error: HttpErrorResponse): void {
    console.log('Error Status:', error.status);
    console.log('Status Text:', error.statusText);
    console.log('URL:', error.url);
    console.log('Headers:', error.headers);
    console.log('Error Body:', error.error);
    console.log('Message:', error.message);

    // Check specific status codes
    switch (error.status) {
      case 0:
        console.log('Network error or CORS issue');
        break;
      case 400:
        console.log('Bad request - validation error');
        break;
      case 401:
        console.log('Unauthorized - authentication required');
        break;
      case 403:
        console.log('Forbidden - insufficient permissions');
        break;
      case 404:
        console.log('Not found - resource does not exist');
        break;
      case 500:
        console.log('Internal server error');
        break;
      case 503:
        console.log('Service unavailable');
        break;
    }
  }
}
```

### 2. catchError Operator

The `catchError` operator intercepts errors in the Observable stream and handles them.

**catchError Examples:**

```typescript
// product.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { Observable, throwError, of, EMPTY } from 'rxjs';
import { catchError, retry } from 'rxjs/operators';

interface Product {
  id: number;
  name: string;
  price: number;
}

@Injectable({
  providedIn: 'root'
})
export class ProductService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com/products';

  // Pattern 1: Re-throw error after logging
  getProduct(id: number): Observable<Product> {
    return this.http.get<Product>(`${this.apiUrl}/${id}`).pipe(
      catchError((error: HttpErrorResponse) => {
        console.error('Error fetching product:', error);
        return throwError(() => error);
      })
    );
  }

  // Pattern 2: Return fallback value
  getProductWithFallback(id: number): Observable<Product | null> {
    return this.http.get<Product>(`${this.apiUrl}/${id}`).pipe(
      catchError((error: HttpErrorResponse) => {
        console.error('Product not found, returning null');
        return of(null);
      })
    );
  }

  // Pattern 3: Return default object
  getProductWithDefault(id: number): Observable<Product> {
    return this.http.get<Product>(`${this.apiUrl}/${id}`).pipe(
      catchError((error: HttpErrorResponse) => {
        console.error('Using default product');
        return of({
          id: 0,
          name: 'Default Product',
          price: 0
        });
      })
    );
  }

  // Pattern 4: Return empty array for lists
  getProducts(): Observable<Product[]> {
    return this.http.get<Product[]>(this.apiUrl).pipe(
      catchError((error: HttpErrorResponse) => {
        console.error('Failed to load products');
        return of([]);
      })
    );
  }

  // Pattern 5: Complete the stream without emitting
  deleteProduct(id: number): Observable<never> {
    return this.http.delete(`${this.apiUrl}/${id}`).pipe(
      catchError((error: HttpErrorResponse) => {
        console.error('Delete failed silently');
        return EMPTY;
      })
    );
  }

  // Pattern 6: Transform error into custom error
  createProduct(product: Omit<Product, 'id'>): Observable<Product> {
    return this.http.post<Product>(this.apiUrl, product).pipe(
      catchError((error: HttpErrorResponse) => {
        const customError = new Error(
          `Failed to create product: ${error.message}`
        );
        return throwError(() => customError);
      })
    );
  }
}
```

### 3. Retry Strategies

Implement retry logic for transient failures using RxJS retry operators.

**Retry Examples:**

```typescript
// retry.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { Observable, throwError, timer } from 'rxjs';
import { retry, retryWhen, mergeMap, finalize, tap } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class RetryService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com';

  // Simple retry - retry 3 times immediately
  getWithSimpleRetry<T>(url: string): Observable<T> {
    return this.http.get<T>(url).pipe(
      retry(3),
      catchError((error: HttpErrorResponse) => {
        console.error('Failed after 3 retries');
        return throwError(() => error);
      })
    );
  }

  // Retry with delay
  getWithDelayedRetry<T>(url: string): Observable<T> {
    return this.http.get<T>(url).pipe(
      retryWhen(errors =>
        errors.pipe(
          mergeMap((error, index) => {
            const retryAttempt = index + 1;
            
            // Max 3 retries
            if (retryAttempt > 3) {
              return throwError(() => error);
            }
            
            console.log(`Retry attempt ${retryAttempt} in 1 second...`);
            return timer(1000);
          })
        )
      )
    );
  }

  // Exponential backoff retry
  getWithExponentialBackoff<T>(url: string): Observable<T> {
    return this.http.get<T>(url).pipe(
      retryWhen(errors =>
        errors.pipe(
          mergeMap((error: HttpErrorResponse, index) => {
            const retryAttempt = index + 1;
            const maxRetries = 5;
            
            if (retryAttempt > maxRetries) {
              return throwError(() => error);
            }
            
            // Exponential backoff: 1s, 2s, 4s, 8s, 16s
            const delayMs = Math.pow(2, index) * 1000;
            console.log(
              `Retry attempt ${retryAttempt}/${maxRetries} in ${delayMs}ms`
            );
            
            return timer(delayMs);
          })
        )
      )
    );
  }

  // Conditional retry - only retry on specific errors
  getWithConditionalRetry<T>(url: string): Observable<T> {
    return this.http.get<T>(url).pipe(
      retryWhen(errors =>
        errors.pipe(
          mergeMap((error: HttpErrorResponse, index) => {
            const retryAttempt = index + 1;
            const maxRetries = 3;

            // Only retry on specific status codes
            const retryableStatuses = [408, 429, 500, 502, 503, 504];
            
            if (
              retryAttempt > maxRetries ||
              !retryableStatuses.includes(error.status)
            ) {
              console.error('Not retrying:', error.status);
              return throwError(() => error);
            }

            const delayMs = retryAttempt * 1000;
            console.log(
              `Retrying ${error.status} error (attempt ${retryAttempt})`
            );
            
            return timer(delayMs);
          })
        )
      )
    );
  }

  // Retry with jitter to prevent thundering herd
  getWithJitter<T>(url: string): Observable<T> {
    return this.http.get<T>(url).pipe(
      retryWhen(errors =>
        errors.pipe(
          mergeMap((error: HttpErrorResponse, index) => {
            const retryAttempt = index + 1;
            const maxRetries = 5;
            
            if (retryAttempt > maxRetries) {
              return throwError(() => error);
            }

            // Base delay with random jitter
            const baseDelay = Math.pow(2, index) * 1000;
            const jitter = Math.random() * 1000;
            const delayMs = baseDelay + jitter;

            console.log(`Retry ${retryAttempt} in ${delayMs.toFixed(0)}ms`);
            return timer(delayMs);
          }),
          finalize(() => console.log('Retry sequence complete'))
        )
      )
    );
  }

  // Retry with progress callback
  getWithRetryProgress<T>(
    url: string,
    onRetry?: (attempt: number, delay: number) => void
  ): Observable<T> {
    return this.http.get<T>(url).pipe(
      retryWhen(errors =>
        errors.pipe(
          mergeMap((error: HttpErrorResponse, index) => {
            const retryAttempt = index + 1;
            const maxRetries = 3;
            
            if (retryAttempt > maxRetries) {
              return throwError(() => error);
            }

            const delayMs = retryAttempt * 1000;
            
            if (onRetry) {
              onRetry(retryAttempt, delayMs);
            }

            return timer(delayMs);
          })
        )
      )
    );
  }
}

// Component usage
@Component({
  selector: 'app-data',
  template: `
    <div *ngIf="retryAttempt > 0">
      Retrying... (Attempt {{ retryAttempt }})
    </div>
  `
})
export class DataComponent implements OnInit {
  private retryService = inject(RetryService);
  retryAttempt = 0;

  ngOnInit() {
    this.retryService
      .getWithRetryProgress<any>(
        '/api/data',
        (attempt, delay) => {
          this.retryAttempt = attempt;
          console.log(`Retry ${attempt} in ${delay}ms`);
        }
      )
      .subscribe({
        next: (data) => console.log('Success:', data),
        error: (error) => console.error('Failed after retries:', error)
      });
  }
}
```

### 4. Global Error Handler

Create a global error handler for centralized error processing.

**Global Error Handler:**

```typescript
// global-error-handler.ts
import { ErrorHandler, Injectable, inject, Injector } from '@angular/core';
import { HttpErrorResponse } from '@angular/common/http';
import { Router } from '@angular/router';
import { ErrorService } from './error.service';
import { LoggingService } from './logging.service';
import { NotificationService } from './notification.service';

@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  private injector = inject(Injector);

  handleError(error: Error | HttpErrorResponse): void {
    const errorService = this.injector.get(ErrorService);
    const loggingService = this.injector.get(LoggingService);
    const notificationService = this.injector.get(NotificationService);
    const router = this.injector.get(Router);

    let message: string;
    let stackTrace: string | undefined;

    if (error instanceof HttpErrorResponse) {
      // Server error
      message = this.getServerErrorMessage(error);
      stackTrace = error.message;
      
      // Handle specific HTTP errors
      this.handleHttpError(error, router, notificationService);
    } else {
      // Client error
      message = this.getClientErrorMessage(error);
      stackTrace = error.stack;
    }

    // Log error
    loggingService.logError(message, stackTrace);

    // Store error for error page
    errorService.setError(error);

    // Show notification
    notificationService.error(message);

    // Log to console in development
    if (!environment.production) {
      console.error('Global error handler caught:', error);
    }
  }

  private getServerErrorMessage(error: HttpErrorResponse): string {
    if (!navigator.onLine) {
      return 'No internet connection';
    }

    if (error.error?.message) {
      return error.error.message;
    }

    return `Server error: ${error.status} - ${error.statusText}`;
  }

  private getClientErrorMessage(error: Error): string {
    return error.message || 'An unexpected error occurred';
  }

  private handleHttpError(
    error: HttpErrorResponse,
    router: Router,
    notificationService: NotificationService
  ): void {
    switch (error.status) {
      case 401:
        notificationService.error('Please log in to continue');
        router.navigate(['/login']);
        break;
      case 403:
        notificationService.error('Access denied');
        router.navigate(['/forbidden']);
        break;
      case 404:
        notificationService.error('Resource not found');
        break;
      case 500:
        notificationService.error('Server error. Please try again later.');
        break;
      case 503:
        notificationService.error('Service temporarily unavailable');
        break;
    }
  }
}

// error.service.ts - Store error state
import { Injectable, signal } from '@angular/core';

@Injectable({
  providedIn: 'root'
})
export class ErrorService {
  private errorSignal = signal<Error | null>(null);
  error = this.errorSignal.asReadonly();

  setError(error: Error): void {
    this.errorSignal.set(error);
  }

  clearError(): void {
    this.errorSignal.set(null);
  }
}

// main.ts - Register global error handler
import { bootstrapApplication } from '@angular/platform-browser';
import { ErrorHandler } from '@angular/core';
import { GlobalErrorHandler } from './app/services/global-error-handler';

bootstrapApplication(AppComponent, {
  providers: [
    { provide: ErrorHandler, useClass: GlobalErrorHandler }
  ]
});
```

### 5. HTTP Error Interceptor

Create an interceptor for centralized HTTP error handling.

**Error Interceptor:**

```typescript
// http-error.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { inject } from '@angular/core';
import { catchError, throwError } from 'rxjs';
import { Router } from '@angular/router';
import { NotificationService } from '../services/notification.service';
import { AuthService } from '../services/auth.service';

export const httpErrorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);
  const notificationService = inject(NotificationService);
  const authService = inject(AuthService);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      let errorMessage = 'An error occurred';

      if (error.error instanceof ErrorEvent) {
        // Client-side error
        errorMessage = `Client error: ${error.error.message}`;
      } else {
        // Server-side error
        errorMessage = getServerErrorMessage(error);
        handleServerError(error, router, authService, notificationService);
      }

      // Log error details
      console.error('HTTP Error:', {
        status: error.status,
        message: errorMessage,
        url: error.url,
        error: error.error
      });

      return throwError(() => error);
    })
  );
};

function getServerErrorMessage(error: HttpErrorResponse): string {
  // Check for custom error message from API
  if (error.error?.message) {
    return error.error.message;
  }

  // Default messages by status code
  switch (error.status) {
    case 0:
      return 'Unable to connect to server';
    case 400:
      return 'Invalid request';
    case 401:
      return 'Authentication required';
    case 403:
      return 'Access forbidden';
    case 404:
      return 'Resource not found';
    case 408:
      return 'Request timeout';
    case 422:
      return 'Validation failed';
    case 429:
      return 'Too many requests';
    case 500:
      return 'Internal server error';
    case 502:
      return 'Bad gateway';
    case 503:
      return 'Service unavailable';
    case 504:
      return 'Gateway timeout';
    default:
      return `Server error: ${error.status}`;
  }
}

function handleServerError(
  error: HttpErrorResponse,
  router: Router,
  authService: AuthService,
  notificationService: NotificationService
): void {
  switch (error.status) {
    case 401:
      // Clear authentication and redirect to login
      authService.logout();
      router.navigate(['/login'], {
        queryParams: { returnUrl: router.url }
      });
      notificationService.warning('Please log in to continue');
      break;

    case 403:
      // Redirect to forbidden page
      router.navigate(['/forbidden']);
      break;

    case 404:
      // Optionally redirect to not found page
      if (!req.url.includes('/api/')) {
        router.navigate(['/not-found']);
      }
      break;

    case 500:
    case 502:
    case 503:
    case 504:
      // Show error notification
      notificationService.error('Server is experiencing issues. Please try again later.');
      break;
  }
}
```

### 6. User-Friendly Error Messages

Transform technical errors into user-friendly messages.

**Error Message Service:**

```typescript
// error-message.service.ts
import { Injectable } from '@angular/core';
import { HttpErrorResponse } from '@angular/common/http';

interface ErrorMessages {
  [key: string]: string;
}

@Injectable({
  providedIn: 'root'
})
export class ErrorMessageService {
  private readonly defaultMessages: ErrorMessages = {
    'network': 'Unable to connect. Please check your internet connection.',
    'timeout': 'Request timed out. Please try again.',
    'server': 'Server error. Please try again later.',
    'notFound': 'The requested resource was not found.',
    'unauthorized': 'You need to log in to access this resource.',
    'forbidden': 'You do not have permission to access this resource.',
    'validation': 'Please check your input and try again.',
    'unknown': 'An unexpected error occurred. Please try again.'
  };

  getUserFriendlyMessage(error: HttpErrorResponse | Error): string {
    if (error instanceof HttpErrorResponse) {
      return this.getHttpErrorMessage(error);
    }
    return this.getClientErrorMessage(error);
  }

  private getHttpErrorMessage(error: HttpErrorResponse): string {
    // Network error
    if (!navigator.onLine || error.status === 0) {
      return this.defaultMessages['network'];
    }

    // Check for API-provided error message
    if (error.error?.message && typeof error.error.message === 'string') {
      return error.error.message;
    }

    // Status code based messages
    switch (error.status) {
      case 400:
        return this.getValidationMessage(error);
      case 401:
        return this.defaultMessages['unauthorized'];
      case 403:
        return this.defaultMessages['forbidden'];
      case 404:
        return this.defaultMessages['notFound'];
      case 408:
        return this.defaultMessages['timeout'];
      case 422:
        return this.getValidationMessage(error);
      case 500:
      case 502:
      case 503:
      case 504:
        return this.defaultMessages['server'];
      default:
        return this.defaultMessages['unknown'];
    }
  }

  private getValidationMessage(error: HttpErrorResponse): string {
    if (error.error?.errors && Array.isArray(error.error.errors)) {
      // Format validation errors
      const errors = error.error.errors
        .map((e: any) => e.message || e)
        .join(', ');
      return `Validation failed: ${errors}`;
    }

    if (error.error?.validationErrors) {
      // Format field-level validation errors
      const errors = Object.entries(error.error.validationErrors)
        .map(([field, messages]) => `${field}: ${messages}`)
        .join(', ');
      return errors;
    }

    return this.defaultMessages['validation'];
  }

  private getClientErrorMessage(error: Error): string {
    if (error.message) {
      return error.message;
    }
    return this.defaultMessages['unknown'];
  }

  // Get detailed technical message for logging
  getTechnicalMessage(error: HttpErrorResponse | Error): string {
    if (error instanceof HttpErrorResponse) {
      return `HTTP ${error.status}: ${error.message} | URL: ${error.url}`;
    }
    return `${error.name}: ${error.message}\n${error.stack}`;
  }
}

// Usage in component
@Component({
  selector: 'app-user-profile',
  template: `
    <div *ngIf="errorMessage" class="error-banner">
      {{ errorMessage }}
    </div>
  `
})
export class UserProfileComponent {
  private userService = inject(UserService);
  private errorMessageService = inject(ErrorMessageService);
  errorMessage: string | null = null;

  loadProfile(userId: number): void {
    this.userService.getUser(userId).subscribe({
      next: (user) => {
        this.errorMessage = null;
        // Process user
      },
      error: (error) => {
        this.errorMessage = this.errorMessageService.getUserFriendlyMessage(error);
      }
    });
  }
}
```

### 7. Error Logging

Implement comprehensive error logging for debugging and monitoring.

**Error Logging Service:**

```typescript
// logging.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { environment } from '../environments/environment';

interface ErrorLog {
  timestamp: number;
  level: 'error' | 'warning' | 'info';
  message: string;
  stack?: string;
  url?: string;
  userAgent?: string;
  userId?: number;
  context?: Record<string, any>;
}

@Injectable({
  providedIn: 'root'
})
export class LoggingService {
  private http = inject(HttpClient);
  private logs: ErrorLog[] = [];
  private maxLocalLogs = 100;

  logError(message: string, stack?: string, context?: Record<string, any>): void {
    const errorLog: ErrorLog = {
      timestamp: Date.now(),
      level: 'error',
      message,
      stack,
      url: window.location.href,
      userAgent: navigator.userAgent,
      context
    };

    this.addLog(errorLog);
    
    if (environment.production) {
      this.sendToServer(errorLog);
    } else {
      console.error('Error logged:', errorLog);
    }
  }

  logHttpError(error: HttpErrorResponse, context?: Record<string, any>): void {
    const errorLog: ErrorLog = {
      timestamp: Date.now(),
      level: 'error',
      message: `HTTP ${error.status}: ${error.message}`,
      url: error.url || undefined,
      userAgent: navigator.userAgent,
      context: {
        ...context,
        status: error.status,
        statusText: error.statusText,
        errorBody: error.error
      }
    };

    this.addLog(errorLog);
    
    if (environment.production) {
      this.sendToServer(errorLog);
    }
  }

  logWarning(message: string, context?: Record<string, any>): void {
    const warningLog: ErrorLog = {
      timestamp: Date.now(),
      level: 'warning',
      message,
      context
    };

    this.addLog(warningLog);
    console.warn('Warning logged:', warningLog);
  }

  private addLog(log: ErrorLog): void {
    this.logs.unshift(log);
    
    // Keep only recent logs in memory
    if (this.logs.length > this.maxLocalLogs) {
      this.logs.pop();
    }

    // Persist to localStorage
    this.saveToLocalStorage();
  }

  private saveToLocalStorage(): void {
    try {
      localStorage.setItem('error_logs', JSON.stringify(this.logs.slice(0, 20)));
    } catch (error) {
      console.error('Failed to save logs to localStorage:', error);
    }
  }

  private sendToServer(log: ErrorLog): void {
    // Send to logging endpoint (don't handle errors to avoid recursion)
    this.http.post(`${environment.apiUrl}/logs`, log).subscribe({
      error: (error) => console.error('Failed to send log to server:', error)
    });
  }

  getLogs(): ErrorLog[] {
    return [...this.logs];
  }

  clearLogs(): void {
    this.logs = [];
    localStorage.removeItem('error_logs');
  }

  exportLogs(): string {
    return JSON.stringify(this.logs, null, 2);
  }
}
```

### 8. Graceful Degradation

Implement fallback strategies when HTTP requests fail.

**Graceful Degradation Patterns:**

```typescript
// data.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { Observable, of, throwError } from 'rxjs';
import { catchError, map, timeout } from 'rxjs/operators';

interface CacheEntry<T> {
  data: T;
  timestamp: number;
}

@Injectable({
  providedIn: 'root'
})
export class DataService {
  private http = inject(HttpClient);
  private cache = new Map<string, CacheEntry<any>>();
  private readonly CACHE_TTL = 5 * 60 * 1000; // 5 minutes

  // Pattern 1: Fallback to cached data
  getDataWithCache<T>(url: string): Observable<T> {
    const cached = this.getCached<T>(url);
    
    return this.http.get<T>(url).pipe(
      timeout(5000),
      map(data => {
        this.setCache(url, data);
        return data;
      }),
      catchError((error: HttpErrorResponse) => {
        if (cached) {
          console.warn('Using cached data due to error:', error.status);
          return of(cached);
        }
        return throwError(() => error);
      })
    );
  }

  // Pattern 2: Fallback to default data
  getDataWithDefault<T>(url: string, defaultValue: T): Observable<T> {
    return this.http.get<T>(url).pipe(
      timeout(5000),
      catchError((error: HttpErrorResponse) => {
        console.warn('Using default data due to error:', error.status);
        return of(defaultValue);
      })
    );
  }

  // Pattern 3: Fallback to alternative endpoint
  getDataWithFallback<T>(primaryUrl: string, fallbackUrl: string): Observable<T> {
    return this.http.get<T>(primaryUrl).pipe(
      timeout(3000),
      catchError((error: HttpErrorResponse) => {
        console.warn('Primary endpoint failed, trying fallback');
        return this.http.get<T>(fallbackUrl).pipe(
          catchError((fallbackError: HttpErrorResponse) => {
            console.error('Both endpoints failed');
            return throwError(() => fallbackError);
          })
        );
      })
    );
  }

  // Pattern 4: Offline queue
  private offlineQueue: Array<{ url: string; method: string; body?: any }> = [];

  postWithOfflineSupport<T>(url: string, body: any): Observable<T> {
    if (!navigator.onLine) {
      this.queueForLater(url, 'POST', body);
      return of(null as T);
    }

    return this.http.post<T>(url, body).pipe(
      catchError((error: HttpErrorResponse) => {
        if (error.status === 0) {
          // Network error - queue for later
          this.queueForLater(url, 'POST', body);
          return of(null as T);
        }
        return throwError(() => error);
      })
    );
  }

  private queueForLater(url: string, method: string, body?: any): void {
    this.offlineQueue.push({ url, method, body });
    console.log('Request queued for when online');
  }

  processOfflineQueue(): void {
    if (!navigator.onLine) return;

    while (this.offlineQueue.length > 0) {
      const request = this.offlineQueue.shift();
      if (request) {
        this.http.request(request.method, request.url, { body: request.body })
          .subscribe({
            next: () => console.log('Queued request processed'),
            error: (error) => console.error('Queued request failed:', error)
          });
      }
    }
  }

  private getCached<T>(key: string): T | null {
    const entry = this.cache.get(key);
    if (!entry) return null;

    const isExpired = Date.now() - entry.timestamp > this.CACHE_TTL;
    if (isExpired) {
      this.cache.delete(key);
      return null;
    }

    return entry.data;
  }

  private setCache<T>(key: string, data: T): void {
    this.cache.set(key, { data, timestamp: Date.now() });
  }
}
```

### 9. Error Boundary Component

Create an error boundary component to catch rendering errors.

**Error Boundary Component:**

```typescript
// error-boundary.component.ts
import { Component, Input, ErrorHandler, OnInit } from '@angular/core';
import { ErrorService } from './error.service';

@Component({
  selector: 'app-error-boundary',
  template: `
    <div *ngIf="!hasError" class="content">
      <ng-content></ng-content>
    </div>
    
    <div *ngIf="hasError" class="error-container">
      <h2>Something went wrong</h2>
      <p>{{ errorMessage }}</p>
      
      <button (click)="retry()">Try Again</button>
      <button (click)="goHome()">Go Home</button>
      
      <details *ngIf="showDetails">
        <summary>Error Details</summary>
        <pre>{{ errorDetails }}</pre>
      </details>
    </div>
  `,
  styles: [`
    .error-container {
      padding: 20px;
      border: 1px solid #f44336;
      border-radius: 4px;
      background-color: #ffebee;
    }
    
    details {
      margin-top: 20px;
    }
    
    pre {
      background: #fff;
      padding: 10px;
      border-radius: 4px;
      overflow-x: auto;
    }
  `]
})
export class ErrorBoundaryComponent implements OnInit {
  @Input() showDetails = false;
  
  hasError = false;
  errorMessage = '';
  errorDetails = '';

  constructor(
    private errorService: ErrorService,
    private router: Router
  ) {}

  ngOnInit() {
    // Subscribe to error service
    this.errorService.error$.subscribe(error => {
      if (error) {
        this.handleError(error);
      }
    });
  }

  private handleError(error: Error): void {
    this.hasError = true;
    this.errorMessage = error.message || 'An unexpected error occurred';
    this.errorDetails = error.stack || '';
  }

  retry(): void {
    this.hasError = false;
    this.errorService.clearError();
    window.location.reload();
  }

  goHome(): void {
    this.hasError = false;
    this.errorService.clearError();
    this.router.navigate(['/']);
  }
}
```

### 10. Validation Error Handling

Handle validation errors with detailed field-level feedback.

**Validation Error Handling:**

```typescript
// form.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { FormGroup } from '@angular/forms';
import { catchError, throwError } from 'rxjs';

interface ValidationError {
  field: string;
  message: string;
}

interface ApiValidationError {
  status: 422;
  error: {
    message: string;
    errors: ValidationError[];
  };
}

@Injectable({
  providedIn: 'root'
})
export class FormService {
  private http = inject(HttpClient);

  submitForm<T>(url: string, formData: any, form?: FormGroup): Observable<T> {
    return this.http.post<T>(url, formData).pipe(
      catchError((error: HttpErrorResponse) => {
        if (error.status === 422 && form) {
          this.handleValidationErrors(error, form);
        }
        return throwError(() => error);
      })
    );
  }

  private handleValidationErrors(error: HttpErrorResponse, form: FormGroup): void {
    const validationErrors = error.error?.errors;
    
    if (Array.isArray(validationErrors)) {
      validationErrors.forEach((err: ValidationError) => {
        const control = form.get(err.field);
        if (control) {
          control.setErrors({
            serverError: err.message
          });
        }
      });
    } else if (typeof error.error?.errors === 'object') {
      // Handle object format: { email: ['Invalid email'], password: ['Too short'] }
      Object.entries(error.error.errors).forEach(([field, messages]) => {
        const control = form.get(field);
        if (control && Array.isArray(messages)) {
          control.setErrors({
            serverError: messages.join(', ')
          });
        }
      });
    }
  }
}

// Component usage
@Component({
  selector: 'app-user-form',
  template: `
    <form [formGroup]="userForm" (ngSubmit)="onSubmit()">
      <input formControlName="email" placeholder="Email">
      <div *ngIf="userForm.get('email')?.errors?.['serverError']" class="error">
        {{ userForm.get('email')?.errors?.['serverError'] }}
      </div>

      <input formControlName="password" type="password" placeholder="Password">
      <div *ngIf="userForm.get('password')?.errors?.['serverError']" class="error">
        {{ userForm.get('password')?.errors?.['serverError'] }}
      </div>

      <button type="submit">Submit</button>
    </form>
  `
})
export class UserFormComponent {
  private formService = inject(FormService);
  private fb = inject(FormBuilder);

  userForm = this.fb.group({
    email: [''],
    password: ['']
  });

  onSubmit(): void {
    if (this.userForm.valid) {
      this.formService.submitForm('/api/users', this.userForm.value, this.userForm)
        .subscribe({
          next: () => console.log('Form submitted successfully'),
          error: (error) => console.error('Form submission failed:', error)
        });
    }
  }
}
```

## Common Mistakes

### 1. Not Distinguishing Client vs Server Errors

```typescript
// WRONG: Treats all errors the same
catchError(error => {
  console.log('Error:', error.message);
})

// CORRECT: Check error type
catchError((error: HttpErrorResponse) => {
  if (error.error instanceof ErrorEvent) {
    // Client-side error
  } else {
    // Server-side error
  }
})
```

### 2. Swallowing Errors Silently

```typescript
// WRONG: Error disappears
catchError(error => {
  return of(null);
})

// CORRECT: Log and handle appropriately
catchError(error => {
  console.error('Error occurred:', error);
  this.notificationService.error('Failed to load data');
  return of(null);
})
```

### 3. Retrying Non-Idempotent Operations

```typescript
// WRONG: Retrying POST/PUT/DELETE
this.http.post('/api/users', data).pipe(
  retry(3) // Could create duplicate entries!
)

// CORRECT: Only retry GET requests
if (method === 'GET') {
  return this.http.get(url).pipe(retry(3));
}
```

### 4. Not Handling Specific Status Codes

```typescript
// WRONG: Generic error handling
catchError(error => {
  alert('Error occurred');
})

// CORRECT: Handle specific cases
catchError((error: HttpErrorResponse) => {
  switch (error.status) {
    case 401: /* Handle unauthorized */
    case 404: /* Handle not found */
    case 500: /* Handle server error */
  }
})
```

### 5. Infinite Retry Loops

```typescript
// WRONG: Retries forever
retryWhen(errors => errors.pipe(delay(1000)))

// CORRECT: Limit retry attempts
retryWhen(errors => errors.pipe(
  mergeMap((error, index) => 
    index < 3 ? timer(1000) : throwError(() => error)
  )
))
```

## Best Practices

### 1. Use Typed Error Responses

```typescript
interface ApiError {
  code: string;
  message: string;
  details?: any;
}

catchError((error: HttpErrorResponse) => {
  const apiError = error.error as ApiError;
  console.log(apiError.code, apiError.message);
})
```

### 2. Implement Exponential Backoff

```typescript
// Wait longer between each retry attempt
retryWhen(errors => errors.pipe(
  mergeMap((error, index) => timer(Math.pow(2, index) * 1000))
))
```

### 3. Log Errors for Debugging

```typescript
catchError((error: HttpErrorResponse) => {
  this.loggingService.logError(error);
  return throwError(() => error);
})
```

### 4. Provide User Feedback

```typescript
catchError((error: HttpErrorResponse) => {
  const message = this.errorMessageService.getUserFriendlyMessage(error);
  this.notificationService.error(message);
  return throwError(() => error);
})
```

### 5. Use Global Error Handler

```typescript
// Register a global error handler for uncaught errors
{ provide: ErrorHandler, useClass: GlobalErrorHandler }
```

## Interview Questions

**Q: What's the difference between client-side and server-side HTTP errors?**

A: Client-side errors occur in the browser (network failures, CORS issues) and have `error.error instanceof ErrorEvent`. Server-side errors come from the backend (400, 500, etc.) with status codes in `error.status`.

**Q: When should you retry HTTP requests?**

A: Only retry idempotent operations (GET, PUT with idempotency keys) on transient failures (408, 429, 500-504). Never retry non-idempotent operations (POST without idempotency) as they may execute multiple times.

**Q: How do you implement exponential backoff?**

A: Use `retryWhen` with increasing delays:

```typescript
retryWhen(errors => errors.pipe(
  mergeMap((error, index) => timer(Math.pow(2, index) * 1000))
))
```

**Q: What is the purpose of a global error handler?**

A: A global error handler catches all unhandled errors (HTTP and client-side), logs them, shows user notifications, and can redirect users or trigger recovery actions. It provides a centralized place for error handling logic.

**Q: How do you handle validation errors from the API?**

A: Check for 422 status, extract field-level errors from the response body, and apply them to the corresponding form controls using `control.setErrors()`.

## Key Takeaways

1. Use `HttpErrorResponse` to access detailed error information (status, message, url)
2. Distinguish between client-side (`error.error instanceof ErrorEvent`) and server-side errors
3. `catchError` operator intercepts errors and allows returning fallback values or re-throwing
4. Only retry idempotent operations (GET) on transient failures (408, 429, 500-504)
5. Implement exponential backoff with jitter to prevent thundering herd problems
6. Create a global error handler for centralized error processing and logging
7. Use error interceptors for consistent HTTP error handling across all requests
8. Transform technical errors into user-friendly messages for better UX
9. Implement graceful degradation with fallback strategies (cached data, defaults)
10. Log errors comprehensively for debugging and monitoring in production

## Resources

- [Angular Error Handling Guide](https://angular.io/guide/http-handle-errors)
- [HttpErrorResponse API](https://angular.io/api/common/http/HttpErrorResponse)
- [RxJS Error Handling](https://rxjs.dev/guide/error-handling)
- [Global Error Handler](https://angular.io/api/core/ErrorHandler)
- [HTTP Status Codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
