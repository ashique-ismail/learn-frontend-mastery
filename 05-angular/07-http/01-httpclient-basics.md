# HttpClient Basics

## The Idea

**In plain English:** HttpClient is Angular's built-in tool for sending requests to a server and getting data back over the internet. Think of it as a messenger your app sends out to ask a website's database for information, then waits for the reply.

**Real-world analogy:** Imagine you are at a restaurant. You tell the waiter what you want, the waiter walks to the kitchen, the kitchen prepares the food, and the waiter brings it back to your table.

- The waiter = `HttpClient` (carries your request and brings back the response)
- Your order = the HTTP request (GET, POST, etc.) sent to the server
- The kitchen = the backend server (processes the request and prepares the data)

---

## Overview

Angular's `HttpClient` is a powerful, RxJS-based HTTP client for making HTTP requests. It provides a simplified API for performing common HTTP operations, automatic JSON parsing, request and response interception, and comprehensive testing support through mocking.

The `HttpClient` is built on top of the browser's `XMLHttpRequest` interface and returns Observables, making it a perfect fit for reactive programming patterns. It's part of `@angular/common/http` and must be provided before use.

## Core Concepts

### 1. HttpClient Service Injection

The `HttpClient` service must be provided at the application level using the `provideHttpClient()` function (standalone apps) or `HttpClientModule` (module-based apps).

**Standalone App Configuration:**

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideHttpClient, withInterceptors, withFetch } from '@angular/common/http';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, {
  providers: [
    provideHttpClient(
      withInterceptors([]), // Add interceptors here
      withFetch() // Use Fetch API instead of XMLHttpRequest
    )
  ]
});
```

**Module-Based App Configuration:**

```typescript
// app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { HttpClientModule } from '@angular/common/http';
import { AppComponent } from './app.component';

@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    HttpClientModule // Import once at root level
  ],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

**Service Injection:**

```typescript
// user.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class UserService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com/users';

  // Or use constructor injection
  // constructor(private http: HttpClient) {}

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl);
  }
}

interface User {
  id: number;
  name: string;
  email: string;
}
```

### 2. GET Requests

GET requests retrieve data from the server. The response is automatically parsed as JSON by default.

**Basic GET Request:**

```typescript
// product.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
}

@Injectable({
  providedIn: 'root'
})
export class ProductService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com';

  // Get all products
  getProducts(): Observable<Product[]> {
    return this.http.get<Product[]>(`${this.apiUrl}/products`);
  }

  // Get single product by ID
  getProduct(id: number): Observable<Product> {
    return this.http.get<Product>(`${this.apiUrl}/products/${id}`);
  }

  // Get with query parameters
  searchProducts(term: string, category?: string): Observable<Product[]> {
    const params: any = { search: term };
    if (category) {
      params.category = category;
    }
    return this.http.get<Product[]>(`${this.apiUrl}/products`, { params });
  }
}
```

**Component Usage:**

```typescript
// product-list.component.ts
import { Component, OnInit, inject } from '@angular/core';
import { ProductService } from './product.service';

@Component({
  selector: 'app-product-list',
  template: `
    <div *ngFor="let product of products">
      <h3>{{ product.name }}</h3>
      <p>Price: {{ product.price | currency }}</p>
    </div>
  `
})
export class ProductListComponent implements OnInit {
  private productService = inject(ProductService);
  products: Product[] = [];

  ngOnInit() {
    this.productService.getProducts().subscribe({
      next: (products) => {
        this.products = products;
      },
      error: (error) => {
        console.error('Error fetching products:', error);
      }
    });
  }
}
```

### 3. POST Requests

POST requests send data to the server to create new resources.

**POST Request Examples:**

```typescript
// auth.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpHeaders } from '@angular/common/http';
import { Observable } from 'rxjs';

interface LoginCredentials {
  email: string;
  password: string;
}

interface AuthResponse {
  token: string;
  user: {
    id: number;
    name: string;
    email: string;
  };
}

@Injectable({
  providedIn: 'root'
})
export class AuthService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com';

  login(credentials: LoginCredentials): Observable<AuthResponse> {
    return this.http.post<AuthResponse>(
      `${this.apiUrl}/auth/login`,
      credentials
    );
  }

  register(userData: any): Observable<AuthResponse> {
    const headers = new HttpHeaders({
      'Content-Type': 'application/json'
    });

    return this.http.post<AuthResponse>(
      `${this.apiUrl}/auth/register`,
      userData,
      { headers }
    );
  }

  // POST with custom options
  submitForm(formData: any): Observable<any> {
    return this.http.post(
      `${this.apiUrl}/forms/submit`,
      formData,
      {
        headers: new HttpHeaders({
          'X-Custom-Header': 'custom-value'
        }),
        reportProgress: true, // For tracking upload progress
        observe: 'events' // Get all events, not just response
      }
    );
  }
}
```

### 4. PUT Requests

PUT requests update existing resources on the server.

**PUT Request Examples:**

```typescript
// user.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

interface User {
  id: number;
  name: string;
  email: string;
  role: string;
}

@Injectable({
  providedIn: 'root'
})
export class UserService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com/users';

  // Full update (replace entire resource)
  updateUser(userId: number, user: User): Observable<User> {
    return this.http.put<User>(
      `${this.apiUrl}/${userId}`,
      user
    );
  }

  // Partial update with PATCH
  patchUser(userId: number, updates: Partial<User>): Observable<User> {
    return this.http.patch<User>(
      `${this.apiUrl}/${userId}`,
      updates
    );
  }

  // Update with optimistic response
  updateUserOptimistic(userId: number, user: User): Observable<User> {
    return this.http.put<User>(
      `${this.apiUrl}/${userId}`,
      user,
      {
        headers: { 'X-Optimistic-Update': 'true' }
      }
    );
  }
}
```

### 5. DELETE Requests

DELETE requests remove resources from the server.

**DELETE Request Examples:**

```typescript
// task.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpResponse } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class TaskService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com/tasks';

  // Simple delete
  deleteTask(taskId: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${taskId}`);
  }

  // Delete with response body
  deleteTaskWithResponse(taskId: number): Observable<{ message: string }> {
    return this.http.delete<{ message: string }>(
      `${this.apiUrl}/${taskId}`
    );
  }

  // Delete with full HTTP response
  deleteTaskFullResponse(taskId: number): Observable<HttpResponse<any>> {
    return this.http.delete(
      `${this.apiUrl}/${taskId}`,
      { observe: 'response' }
    );
  }

  // Bulk delete
  deleteTasks(taskIds: number[]): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}`, {
      body: { ids: taskIds }
    });
  }
}
```

### 6. Typing Responses

TypeScript generics ensure type safety for HTTP responses.

**Response Typing Examples:**

```typescript
// api-types.ts
export interface ApiResponse<T> {
  data: T;
  message: string;
  status: number;
}

export interface PaginatedResponse<T> {
  items: T[];
  total: number;
  page: number;
  pageSize: number;
  hasMore: boolean;
}

export interface User {
  id: number;
  name: string;
  email: string;
  createdAt: string;
}

export interface CreateUserDto {
  name: string;
  email: string;
  password: string;
}

// data.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class DataService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com';

  // Typed response with wrapper
  getUsers(): Observable<User[]> {
    return this.http
      .get<ApiResponse<User[]>>(`${this.apiUrl}/users`)
      .pipe(
        map(response => response.data)
      );
  }

  // Paginated response
  getUsersPaginated(page: number, pageSize: number): Observable<PaginatedResponse<User>> {
    return this.http.get<PaginatedResponse<User>>(
      `${this.apiUrl}/users`,
      { params: { page: page.toString(), pageSize: pageSize.toString() } }
    );
  }

  // Create with typed request/response
  createUser(dto: CreateUserDto): Observable<User> {
    return this.http
      .post<ApiResponse<User>>(`${this.apiUrl}/users`, dto)
      .pipe(
        map(response => response.data)
      );
  }
}
```

### 7. Observables and Subscriptions

HttpClient methods return cold Observables that must be subscribed to execute.

**Observable Patterns:**

```typescript
// article.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, Subject, BehaviorSubject, shareReplay, tap } from 'rxjs';

interface Article {
  id: number;
  title: string;
  content: string;
}

@Injectable({
  providedIn: 'root'
})
export class ArticleService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com/articles';

  // Cache articles with shareReplay
  private articlesCache$: Observable<Article[]> | null = null;

  getArticles(): Observable<Article[]> {
    if (!this.articlesCache$) {
      this.articlesCache$ = this.http.get<Article[]>(this.apiUrl).pipe(
        shareReplay(1) // Cache the last emission
      );
    }
    return this.articlesCache$;
  }

  // Invalidate cache
  invalidateCache(): void {
    this.articlesCache$ = null;
  }

  // Subject-based state management
  private articlesSubject = new BehaviorSubject<Article[]>([]);
  articles$ = this.articlesSubject.asObservable();

  loadArticles(): void {
    this.http.get<Article[]>(this.apiUrl).subscribe({
      next: (articles) => this.articlesSubject.next(articles),
      error: (error) => console.error('Error loading articles:', error)
    });
  }

  addArticle(article: Article): void {
    this.http.post<Article>(this.apiUrl, article)
      .pipe(
        tap(newArticle => {
          const current = this.articlesSubject.value;
          this.articlesSubject.next([...current, newArticle]);
        })
      )
      .subscribe();
  }
}
```

### 8. Custom Headers

Headers can be set for authentication, content negotiation, or custom metadata.

**Header Configuration:**

```typescript
// api.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpHeaders, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class ApiService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com';

  // Method 1: HttpHeaders object
  getWithHeaders(): Observable<any> {
    const headers = new HttpHeaders({
      'Authorization': 'Bearer token123',
      'Content-Type': 'application/json',
      'X-Custom-Header': 'custom-value'
    });

    return this.http.get(`${this.apiUrl}/data`, { headers });
  }

  // Method 2: Append headers
  getWithAppendedHeaders(): Observable<any> {
    let headers = new HttpHeaders();
    headers = headers.append('Authorization', 'Bearer token123');
    headers = headers.append('X-Request-ID', this.generateRequestId());

    return this.http.get(`${this.apiUrl}/data`, { headers });
  }

  // Method 3: Set headers
  getWithSetHeaders(): Observable<any> {
    let headers = new HttpHeaders();
    headers = headers.set('Authorization', 'Bearer token123');

    return this.http.get(`${this.apiUrl}/data`, { headers });
  }

  // Headers with query parameters
  searchWithHeadersAndParams(query: string): Observable<any> {
    const headers = new HttpHeaders({
      'Authorization': 'Bearer token123'
    });

    const params = new HttpParams()
      .set('q', query)
      .set('limit', '10')
      .set('sort', 'date');

    return this.http.get(`${this.apiUrl}/search`, { headers, params });
  }

  private generateRequestId(): string {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
}
```

### 9. Query Parameters

Query parameters are passed via the `params` option using `HttpParams` or object literals.

**Query Parameter Examples:**

```typescript
// search.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';

interface SearchFilters {
  category?: string;
  minPrice?: number;
  maxPrice?: number;
  sortBy?: string;
  order?: 'asc' | 'desc';
}

@Injectable({
  providedIn: 'root'
})
export class SearchService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com';

  // Method 1: Object literal (simple)
  search(query: string): Observable<any[]> {
    return this.http.get<any[]>(`${this.apiUrl}/search`, {
      params: { q: query, limit: '20' }
    });
  }

  // Method 2: HttpParams (complex)
  advancedSearch(query: string, filters: SearchFilters): Observable<any[]> {
    let params = new HttpParams().set('q', query);

    if (filters.category) {
      params = params.set('category', filters.category);
    }
    if (filters.minPrice !== undefined) {
      params = params.set('minPrice', filters.minPrice.toString());
    }
    if (filters.maxPrice !== undefined) {
      params = params.set('maxPrice', filters.maxPrice.toString());
    }
    if (filters.sortBy) {
      params = params.set('sortBy', filters.sortBy);
    }
    if (filters.order) {
      params = params.set('order', filters.order);
    }

    return this.http.get<any[]>(`${this.apiUrl}/search`, { params });
  }

  // Method 3: fromObject
  searchWithObject(searchParams: Record<string, any>): Observable<any[]> {
    const params = new HttpParams({ fromObject: searchParams });
    return this.http.get<any[]>(`${this.apiUrl}/search`, { params });
  }

  // Array parameters
  searchMultipleCategories(categories: string[]): Observable<any[]> {
    let params = new HttpParams();
    categories.forEach(cat => {
      params = params.append('category', cat);
    });
    // Results in: ?category=electronics&category=books&category=sports

    return this.http.get<any[]>(`${this.apiUrl}/search`, { params });
  }
}
```

### 10. Request Options

HttpClient methods accept an options object for customizing requests.

**Request Options Examples:**

```typescript
// http-options.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpResponse, HttpEvent, HttpHeaders } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class HttpOptionsService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com';

  // Option 1: observe 'body' (default)
  getBody(): Observable<any> {
    return this.http.get(`${this.apiUrl}/data`);
    // Returns only the response body
  }

  // Option 2: observe 'response'
  getFullResponse(): Observable<HttpResponse<any>> {
    return this.http.get(`${this.apiUrl}/data`, {
      observe: 'response'
    });
    // Returns full HttpResponse with headers, status, etc.
  }

  // Option 3: observe 'events'
  getEvents(): Observable<HttpEvent<any>> {
    return this.http.get(`${this.apiUrl}/data`, {
      observe: 'events',
      reportProgress: true
    });
    // Returns all HTTP events (sent, progress, response)
  }

  // Response type options
  getText(): Observable<string> {
    return this.http.get(`${this.apiUrl}/text`, {
      responseType: 'text'
    });
  }

  getBlob(): Observable<Blob> {
    return this.http.get(`${this.apiUrl}/image`, {
      responseType: 'blob'
    });
  }

  getArrayBuffer(): Observable<ArrayBuffer> {
    return this.http.get(`${this.apiUrl}/binary`, {
      responseType: 'arraybuffer'
    });
  }

  // Combined options
  downloadFileWithProgress(): Observable<HttpEvent<Blob>> {
    return this.http.get(`${this.apiUrl}/download`, {
      responseType: 'blob',
      observe: 'events',
      reportProgress: true,
      headers: new HttpHeaders({
        'Accept': 'application/octet-stream'
      })
    });
  }

  // With credentials (cookies)
  getWithCredentials(): Observable<any> {
    return this.http.get(`${this.apiUrl}/secure`, {
      withCredentials: true // Send cookies with cross-origin requests
    });
  }
}
```

## Common Mistakes

### 1. Not Subscribing to Observables

```typescript
// WRONG: Request never executes
getUsers() {
  this.http.get<User[]>('/api/users'); // Does nothing!
}

// CORRECT: Subscribe to execute
getUsers() {
  this.http.get<User[]>('/api/users').subscribe({
    next: (users) => this.users = users,
    error: (error) => console.error(error)
  });
}
```

### 2. Multiple Subscriptions to Same Request

```typescript
// WRONG: Makes multiple HTTP requests
const users$ = this.http.get<User[]>('/api/users');
users$.subscribe(u => console.log(u));
users$.subscribe(u => this.users = u);
users$.subscribe(u => this.count = u.length);

// CORRECT: Share one request
const users$ = this.http.get<User[]>('/api/users').pipe(
  shareReplay(1)
);
users$.subscribe(u => console.log(u));
users$.subscribe(u => this.users = u);
users$.subscribe(u => this.count = u.length);
```

### 3. Forgetting to Unsubscribe

```typescript
// WRONG: Memory leak
ngOnInit() {
  this.http.get('/api/data').subscribe(data => this.data = data);
}

// CORRECT: Use takeUntilDestroyed or async pipe
private destroyRef = inject(DestroyRef);

ngOnInit() {
  this.http.get('/api/data')
    .pipe(takeUntilDestroyed(this.destroyRef))
    .subscribe(data => this.data = data);
}

// OR use async pipe in template
data$ = this.http.get('/api/data');
// Template: {{ data$ | async | json }}
```

### 4. Incorrect Header Mutation

```typescript
// WRONG: HttpHeaders is immutable
const headers = new HttpHeaders();
headers.set('Authorization', 'Bearer token'); // Returns new instance!

// CORRECT: Reassign the result
let headers = new HttpHeaders();
headers = headers.set('Authorization', 'Bearer token');
```

### 5. Missing Error Handling

```typescript
// WRONG: Unhandled errors
this.http.get('/api/data').subscribe(data => this.data = data);

// CORRECT: Handle errors
this.http.get('/api/data').subscribe({
  next: (data) => this.data = data,
  error: (error) => this.handleError(error)
});
```

## Best Practices

### 1. Service Encapsulation

```typescript
// Encapsulate HTTP logic in services
@Injectable({ providedIn: 'root' })
export class UserService {
  private http = inject(HttpClient);
  private apiUrl = environment.apiUrl;

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(`${this.apiUrl}/users`);
  }
}
```

### 2. Type Safety

```typescript
// Always specify response types
this.http.get<User[]>('/api/users') // Typed as Observable<User[]>
  .subscribe(users => {
    // users is User[], not any
  });
```

### 3. Centralized Configuration

```typescript
// Use environment files for API URLs
export const environment = {
  production: false,
  apiUrl: 'https://api.example.com'
};

// In service
private apiUrl = environment.apiUrl;
```

### 4. Async Pipe for Subscriptions

```typescript
// Component
users$ = this.userService.getUsers();

// Template
<div *ngFor="let user of users$ | async">
  {{ user.name }}
</div>
```

### 5. Error Handling Strategy

```typescript
getUsers(): Observable<User[]> {
  return this.http.get<User[]>('/api/users').pipe(
    catchError(error => {
      console.error('Error fetching users:', error);
      return of([]); // Return empty array as fallback
    })
  );
}
```

## Interview Questions

**Q: What is the difference between HttpClient and HttpClientModule?**

A: `HttpClientModule` is the NgModule that provides the `HttpClient` service (used in module-based apps). `HttpClient` is the actual service for making HTTP requests. In standalone apps, use `provideHttpClient()` instead of importing the module.

**Q: Why does HttpClient return Observables instead of Promises?**

A: Observables provide cancellation, retry logic, multiple operators for transformation, and the ability to handle multiple values over time. They integrate seamlessly with RxJS for complex async operations.

**Q: How do you cancel an HTTP request?**

A: Unsubscribe from the Observable:

```typescript
const subscription = this.http.get('/api/data').subscribe();
subscription.unsubscribe(); // Cancels the request
```

**Q: What happens if you don't subscribe to an HttpClient Observable?**

A: Nothing. The HTTP request is never sent. HttpClient Observables are "cold" - they only execute when subscribed to.

**Q: How do you make multiple parallel HTTP requests?**

A: Use `forkJoin` to wait for all requests to complete:

```typescript
forkJoin({
  users: this.http.get('/api/users'),
  posts: this.http.get('/api/posts')
}).subscribe(({ users, posts }) => {
  // Both requests complete
});
```

## Key Takeaways

1. HttpClient must be provided via `provideHttpClient()` or `HttpClientModule` before injection
2. All HttpClient methods return cold Observables that must be subscribed to execute
3. Use TypeScript generics for type-safe responses: `http.get<User[]>()`
4. HttpHeaders and HttpParams are immutable - always reassign after modification
5. Prefer async pipe in templates to automatic subscription management
6. Encapsulate HTTP logic in services, not components
7. Always handle errors with catchError or error callback
8. Use shareReplay to prevent multiple executions of the same request
9. Query parameters can be passed as object literals or HttpParams
10. Custom headers are essential for authentication and API requirements

## Resources

- [Angular HttpClient Documentation](https://angular.io/guide/http)
- [HttpClient API Reference](https://angular.io/api/common/http/HttpClient)
- [RxJS Observable Guide](https://rxjs.dev/guide/observable)
- [Angular HTTP Guide (v18+)](https://angular.io/guide/understanding-communicating-with-http)
- [HTTP Testing Guide](https://angular.io/guide/http-test-requests)
