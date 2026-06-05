# Request/Response Typing

## The Idea

**In plain English:** When your app asks a server for information, request/response typing means you tell the code exactly what shape the data will be — like labeling every box in a warehouse so you always know what's inside. A "type" is just a blueprint that describes what fields a piece of data has, so the coding tool can warn you before you make a mistake.

**Real-world analogy:** Imagine ordering a meal at a restaurant using a printed order form. The form has specific boxes for starter, main, and dessert — you can only write valid options in each box, and the kitchen receives a form where every field is clearly labeled.

- The order form template = the TypeScript interface (defines what fields exist)
- Filling in the form = creating a typed request object
- The meal the kitchen sends back = the typed HTTP response
- The printed labels on each box = the type annotations that prevent mistakes

---

## Overview

TypeScript's type system provides powerful guarantees for HTTP operations in Angular. Request/response typing ensures compile-time safety, better IDE support, and reduces runtime errors. Angular's HttpClient is built with generics, allowing you to specify exact types for requests and responses throughout the HTTP pipeline.

Proper typing extends beyond simple response bodies to include generic wrappers, discriminated unions for different response states, typed HTTP responses with headers and status codes, and type-safe interceptor chains that preserve and transform types through the request/response lifecycle.

## Core Concepts

### 1. Basic Response Typing

The HttpClient accepts a generic type parameter that defines the response body type.

**Simple Response Types:**

```typescript
// user.types.ts
export interface User {
  id: number;
  name: string;
  email: string;
  role: 'admin' | 'user' | 'guest';
  createdAt: string;
  updatedAt: string;
}

export interface CreateUserDto {
  name: string;
  email: string;
  password: string;
  role?: 'user' | 'guest';
}

export interface UpdateUserDto {
  name?: string;
  email?: string;
  role?: 'admin' | 'user' | 'guest';
}

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

  // Typed GET - returns Observable<User>
  getUser(id: number): Observable<User> {
    return this.http.get<User>(`${this.apiUrl}/${id}`);
  }

  // Typed POST - accepts CreateUserDto, returns Observable<User>
  createUser(dto: CreateUserDto): Observable<User> {
    return this.http.post<User>(this.apiUrl, dto);
  }

  // Typed PUT - accepts UpdateUserDto, returns Observable<User>
  updateUser(id: number, dto: UpdateUserDto): Observable<User> {
    return this.http.put<User>(`${this.apiUrl}/${id}`, dto);
  }

  // Typed DELETE - returns Observable<void>
  deleteUser(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`);
  }
}
```

### 2. Generic API Response Wrappers

Most APIs wrap responses in a consistent structure. Create generic types to handle this pattern.

**Generic Wrapper Types:**

```typescript
// api-response.types.ts
export interface ApiResponse<T> {
  success: boolean;
  data: T;
  message: string;
  timestamp: number;
}

export interface ApiError {
  success: false;
  error: {
    code: string;
    message: string;
    details?: Record<string, any>;
  };
  timestamp: number;
}

export interface PaginatedResponse<T> {
  items: T[];
  pagination: {
    page: number;
    pageSize: number;
    totalItems: number;
    totalPages: number;
    hasNext: boolean;
    hasPrevious: boolean;
  };
}

export interface ListResponse<T> {
  data: T[];
  count: number;
  offset: number;
  limit: number;
}

// product.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

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

  // Get single product with wrapper
  getProduct(id: number): Observable<Product> {
    return this.http
      .get<ApiResponse<Product>>(`${this.apiUrl}/products/${id}`)
      .pipe(
        map(response => response.data)
      );
  }

  // Get paginated products
  getProducts(page: number, pageSize: number): Observable<PaginatedResponse<Product>> {
    return this.http.get<PaginatedResponse<Product>>(
      `${this.apiUrl}/products`,
      { params: { page: page.toString(), pageSize: pageSize.toString() } }
    );
  }

  // Get list with count
  searchProducts(query: string): Observable<ListResponse<Product>> {
    return this.http.get<ListResponse<Product>>(
      `${this.apiUrl}/products/search`,
      { params: { q: query } }
    );
  }

  // Create product with wrapper
  createProduct(product: Omit<Product, 'id'>): Observable<Product> {
    return this.http
      .post<ApiResponse<Product>>(`${this.apiUrl}/products`, product)
      .pipe(
        map(response => response.data)
      );
  }
}
```

### 3. HttpResponse Type

Use `HttpResponse<T>` to access full response information including headers and status codes.

**Full Response Typing:**

```typescript
// http-response.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpResponse, HttpHeaders } from '@angular/common/http';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

interface User {
  id: number;
  name: string;
  email: string;
}

interface ResponseMetadata {
  etag: string;
  lastModified: string;
  rateLimit: {
    limit: number;
    remaining: number;
    reset: number;
  };
}

@Injectable({
  providedIn: 'root'
})
export class HttpResponseService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com';

  // Get full response
  getUserWithMetadata(id: number): Observable<HttpResponse<User>> {
    return this.http.get<User>(
      `${this.apiUrl}/users/${id}`,
      { observe: 'response' }
    );
    // Returns HttpResponse<User> with status, headers, body
  }

  // Extract headers and body
  getUserWithHeaders(id: number): Observable<{ user: User; metadata: ResponseMetadata }> {
    return this.http
      .get<User>(`${this.apiUrl}/users/${id}`, { observe: 'response' })
      .pipe(
        map(response => ({
          user: response.body!,
          metadata: this.extractMetadata(response.headers)
        }))
      );
  }

  // Check response status
  checkUserExists(id: number): Observable<boolean> {
    return this.http
      .get<User>(`${this.apiUrl}/users/${id}`, { observe: 'response' })
      .pipe(
        map(response => response.status === 200),
        catchError(() => of(false))
      );
  }

  // Conditional request with ETag
  getUserIfModified(id: number, etag: string): Observable<User | null> {
    const headers = new HttpHeaders({ 'If-None-Match': etag });

    return this.http
      .get<User>(`${this.apiUrl}/users/${id}`, { 
        headers, 
        observe: 'response' 
      })
      .pipe(
        map(response => {
          if (response.status === 304) {
            return null; // Not modified
          }
          return response.body;
        })
      );
  }

  private extractMetadata(headers: HttpHeaders): ResponseMetadata {
    return {
      etag: headers.get('ETag') || '',
      lastModified: headers.get('Last-Modified') || '',
      rateLimit: {
        limit: parseInt(headers.get('X-RateLimit-Limit') || '0'),
        remaining: parseInt(headers.get('X-RateLimit-Remaining') || '0'),
        reset: parseInt(headers.get('X-RateLimit-Reset') || '0')
      }
    };
  }
}
```

### 4. Discriminated Unions for Response States

Use discriminated unions to represent different response states type-safely.

**State-Based Response Types:**

```typescript
// api-state.types.ts
export type ApiState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: string };

export type DataResult<T, E = string> =
  | { success: true; data: T }
  | { success: false; error: E };

export type AsyncData<T> =
  | { state: 'pending' }
  | { state: 'resolved'; value: T }
  | { state: 'rejected'; reason: Error };

// state.service.ts
import { Injectable, inject, signal } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { catchError, map, startWith } from 'rxjs/operators';
import { of, Observable } from 'rxjs';

interface Product {
  id: number;
  name: string;
  price: number;
}

@Injectable({
  providedIn: 'root'
})
export class StateService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com';

  // Signal-based state management
  productsState = signal<ApiState<Product[]>>({ status: 'idle' });

  loadProducts(): void {
    this.productsState.set({ status: 'loading' });

    this.http.get<Product[]>(`${this.apiUrl}/products`)
      .subscribe({
        next: (data) => this.productsState.set({ status: 'success', data }),
        error: (error: HttpErrorResponse) => 
          this.productsState.set({ status: 'error', error: error.message })
      });
  }

  // Observable-based state stream
  getProductsStream(): Observable<ApiState<Product[]>> {
    return this.http.get<Product[]>(`${this.apiUrl}/products`).pipe(
      map((data): ApiState<Product[]> => ({ status: 'success', data })),
      catchError((error): Observable<ApiState<Product[]>> => 
        of({ status: 'error', error: error.message })
      ),
      startWith({ status: 'loading' } as ApiState<Product[]>)
    );
  }

  // Result-based API call
  createProduct(product: Omit<Product, 'id'>): Observable<DataResult<Product>> {
    return this.http
      .post<Product>(`${this.apiUrl}/products`, product)
      .pipe(
        map((data): DataResult<Product> => ({ success: true, data })),
        catchError((error: HttpErrorResponse): Observable<DataResult<Product>> =>
          of({ success: false, error: error.message })
        )
      );
  }

  // Async data pattern
  getProductAsync(id: number): Observable<AsyncData<Product>> {
    return this.http.get<Product>(`${this.apiUrl}/products/${id}`).pipe(
      map((value): AsyncData<Product> => ({ state: 'resolved', value })),
      catchError((reason): Observable<AsyncData<Product>> =>
        of({ state: 'rejected', reason })
      ),
      startWith({ state: 'pending' } as AsyncData<Product>)
    );
  }
}

// Component usage with type narrowing
@Component({
  selector: 'app-products',
  template: `
    @switch (productsState().status) {
      @case ('idle') {
        <button (click)="loadProducts()">Load Products</button>
      }
      @case ('loading') {
        <p>Loading products...</p>
      }
      @case ('success') {
        <div *ngFor="let product of productsState().data">
          {{ product.name }} - {{ product.price | currency }}
        </div>
      }
      @case ('error') {
        <p class="error">Error: {{ productsState().error }}</p>
      }
    }
  `
})
export class ProductsComponent {
  private stateService = inject(StateService);
  productsState = this.stateService.productsState;

  loadProducts(): void {
    this.stateService.loadProducts();
  }
}
```

### 5. Type-Safe Interceptor Chains

Interceptors can preserve and transform types throughout the request/response pipeline.

**Typed Interceptors:**

```typescript
// auth-token.interceptor.ts
import { HttpInterceptorFn, HttpRequest } from '@angular/common/http';
import { inject } from '@angular/core';

// Type-safe request cloning
export const authTokenInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).getToken();

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

// response-wrapper.interceptor.ts
import { HttpInterceptorFn, HttpResponse } from '@angular/common/http';
import { map } from 'rxjs/operators';

interface ApiWrapper<T> {
  data: T;
  meta: {
    timestamp: number;
    version: string;
  };
}

// Unwrap API responses
export const responseUnwrapperInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    map(event => {
      if (event instanceof HttpResponse && event.body) {
        // Check if response is wrapped
        const body = event.body as any;
        if (body && typeof body === 'object' && 'data' in body) {
          // Unwrap and return just the data
          return event.clone({
            body: body.data
          });
        }
      }
      return event;
    })
  );
};

// typed-error.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { catchError, throwError } from 'rxjs';

export interface TypedApiError {
  code: string;
  message: string;
  details?: Record<string, any>;
  timestamp: number;
}

export const typedErrorInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      const typedError: TypedApiError = {
        code: error.error?.code || 'UNKNOWN_ERROR',
        message: error.error?.message || error.message,
        details: error.error?.details,
        timestamp: Date.now()
      };
      return throwError(() => typedError);
    })
  );
};

// Type preservation through chain
export const interceptorChain: HttpInterceptorFn[] = [
  authTokenInterceptor,
  responseUnwrapperInterceptor,
  typedErrorInterceptor
];
```

### 6. Request Body Typing

Type request bodies to ensure correct data structure is sent to the API.

**Request Typing Examples:**

```typescript
// request.types.ts
export interface CreatePostRequest {
  title: string;
  content: string;
  authorId: number;
  tags: string[];
  published?: boolean;
}

export interface UpdatePostRequest {
  title?: string;
  content?: string;
  tags?: string[];
  published?: boolean;
}

export interface BulkDeleteRequest {
  ids: number[];
  reason?: string;
}

export interface SearchRequest {
  query: string;
  filters: {
    category?: string;
    minPrice?: number;
    maxPrice?: number;
    inStock?: boolean;
  };
  sort?: {
    field: string;
    order: 'asc' | 'desc';
  };
  pagination: {
    page: number;
    pageSize: number;
  };
}

// post.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

interface Post {
  id: number;
  title: string;
  content: string;
  authorId: number;
  tags: string[];
  published: boolean;
  createdAt: string;
  updatedAt: string;
}

@Injectable({
  providedIn: 'root'
})
export class PostService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com';

  // Typed request body
  createPost(request: CreatePostRequest): Observable<Post> {
    return this.http.post<Post>(`${this.apiUrl}/posts`, request);
  }

  // Partial update with typed request
  updatePost(id: number, request: UpdatePostRequest): Observable<Post> {
    return this.http.patch<Post>(`${this.apiUrl}/posts/${id}`, request);
  }

  // Bulk operation with typed request
  bulkDelete(request: BulkDeleteRequest): Observable<{ deleted: number }> {
    return this.http.post<{ deleted: number }>(
      `${this.apiUrl}/posts/bulk-delete`,
      request
    );
  }

  // Complex search with typed request
  search(request: SearchRequest): Observable<PaginatedResponse<Post>> {
    return this.http.post<PaginatedResponse<Post>>(
      `${this.apiUrl}/posts/search`,
      request
    );
  }
}
```

### 7. Generic Service Patterns

Create generic service base classes for consistent typing across resources.

**Generic Service Implementation:**

```typescript
// generic-api.service.ts
import { inject } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface BaseEntity {
  id: number | string;
}

export interface ListParams {
  page?: number;
  pageSize?: number;
  sort?: string;
  order?: 'asc' | 'desc';
}

export abstract class GenericApiService<
  T extends BaseEntity,
  TCreate = Omit<T, 'id'>,
  TUpdate = Partial<Omit<T, 'id'>>
> {
  protected http = inject(HttpClient);
  protected abstract baseUrl: string;

  getAll(params?: ListParams): Observable<T[]> {
    let httpParams = new HttpParams();
    if (params) {
      Object.entries(params).forEach(([key, value]) => {
        if (value !== undefined) {
          httpParams = httpParams.set(key, value.toString());
        }
      });
    }

    return this.http.get<ApiResponse<T[]>>(this.baseUrl, { params: httpParams })
      .pipe(map(response => response.data));
  }

  getById(id: number | string): Observable<T> {
    return this.http.get<ApiResponse<T>>(`${this.baseUrl}/${id}`)
      .pipe(map(response => response.data));
  }

  create(item: TCreate): Observable<T> {
    return this.http.post<ApiResponse<T>>(this.baseUrl, item)
      .pipe(map(response => response.data));
  }

  update(id: number | string, item: TUpdate): Observable<T> {
    return this.http.patch<ApiResponse<T>>(`${this.baseUrl}/${id}`, item)
      .pipe(map(response => response.data));
  }

  delete(id: number | string): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/${id}`);
  }
}

// Concrete implementation
interface Product extends BaseEntity {
  id: number;
  name: string;
  price: number;
  category: string;
}

interface CreateProductDto {
  name: string;
  price: number;
  category: string;
}

interface UpdateProductDto {
  name?: string;
  price?: number;
  category?: string;
}

@Injectable({
  providedIn: 'root'
})
export class ProductService extends GenericApiService<
  Product,
  CreateProductDto,
  UpdateProductDto
> {
  protected baseUrl = 'https://api.example.com/products';

  // Add product-specific methods
  searchByCategory(category: string): Observable<Product[]> {
    return this.http.get<ApiResponse<Product[]>>(
      `${this.baseUrl}/category/${category}`
    ).pipe(map(response => response.data));
  }
}
```

### 8. Type Guards for Runtime Validation

Combine TypeScript types with runtime validation for robust type safety.

**Type Guards and Validation:**

```typescript
// type-guards.ts
export interface User {
  id: number;
  name: string;
  email: string;
  role: 'admin' | 'user';
}

export interface Admin extends User {
  role: 'admin';
  permissions: string[];
}

// Type guard functions
export function isUser(obj: any): obj is User {
  return (
    obj &&
    typeof obj.id === 'number' &&
    typeof obj.name === 'string' &&
    typeof obj.email === 'string' &&
    (obj.role === 'admin' || obj.role === 'user')
  );
}

export function isAdmin(user: User): user is Admin {
  return user.role === 'admin' && 'permissions' in user;
}

export function isApiResponse<T>(obj: any): obj is ApiResponse<T> {
  return (
    obj &&
    typeof obj.success === 'boolean' &&
    'data' in obj &&
    typeof obj.timestamp === 'number'
  );
}

// validation.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { map, catchError } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class ValidationService {
  constructor(private http: HttpClient) {}

  getUserWithValidation(id: number): Observable<User> {
    return this.http.get<User>(`/api/users/${id}`).pipe(
      map(response => {
        if (!isUser(response)) {
          throw new Error('Invalid user data received from API');
        }
        return response;
      }),
      catchError(error => {
        console.error('Validation error:', error);
        return throwError(() => error);
      })
    );
  }

  processUser(user: User): void {
    if (isAdmin(user)) {
      // TypeScript knows user is Admin here
      console.log('Admin permissions:', user.permissions);
    } else {
      // Regular user
      console.log('Regular user:', user.name);
    }
  }
}
```

### 9. Utility Types for HTTP Operations

Use TypeScript utility types to derive request/response types.

**Utility Type Examples:**

```typescript
// utility-types.ts
export interface Product {
  id: number;
  name: string;
  description: string;
  price: number;
  stock: number;
  createdAt: string;
  updatedAt: string;
}

// Derive create type (omit generated fields)
export type CreateProduct = Omit<Product, 'id' | 'createdAt' | 'updatedAt'>;

// Derive update type (all fields optional except id)
export type UpdateProduct = Partial<Omit<Product, 'id' | 'createdAt' | 'updatedAt'>>;

// Pick specific fields for list view
export type ProductListItem = Pick<Product, 'id' | 'name' | 'price'>;

// Required subset
export type ProductSummary = Required<Pick<Product, 'id' | 'name' | 'price'>>;

// Readonly for immutable data
export type ImmutableProduct = Readonly<Product>;

// Record for key-value mapping
export type ProductMap = Record<number, Product>;

// Extract/Exclude with unions
export type ProductRole = 'viewer' | 'editor' | 'admin' | 'owner';
export type RestrictedRole = Exclude<ProductRole, 'owner'>; // 'viewer' | 'editor' | 'admin'
export type AdminRole = Extract<ProductRole, 'admin' | 'owner'>; // 'admin' | 'owner'

// product-advanced.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class ProductAdvancedService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com/products';

  // Use derived types in methods
  create(product: CreateProduct): Observable<Product> {
    return this.http.post<Product>(this.apiUrl, product);
  }

  update(id: number, updates: UpdateProduct): Observable<Product> {
    return this.http.patch<Product>(`${this.apiUrl}/${id}`, updates);
  }

  getList(): Observable<ProductListItem[]> {
    return this.http.get<ProductListItem[]>(`${this.apiUrl}/list`);
  }

  getSummaries(): Observable<ProductSummary[]> {
    return this.http.get<ProductSummary[]>(`${this.apiUrl}/summaries`);
  }
}
```

### 10. Conditional Types for Advanced Scenarios

Use conditional types for complex type transformations in HTTP operations.

**Conditional Type Examples:**

```typescript
// conditional-types.ts
// Unwrap API response wrapper
export type Unwrap<T> = T extends ApiResponse<infer U> ? U : T;

// Make fields required based on condition
export type RequireFields<T, K extends keyof T> = T & Required<Pick<T, K>>;

// Async result wrapper
export type AsyncResult<T> = Promise<T> | Observable<T>;

// Conditional endpoint type
export type EndpointResponse<T extends string> =
  T extends 'user' ? User :
  T extends 'product' ? Product :
  T extends 'order' ? Order :
  never;

// Deep partial for nested objects
export type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

// api-client.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class ApiClientService {
  private http = inject(HttpClient);
  private baseUrl = 'https://api.example.com';

  // Generic endpoint fetcher with conditional typing
  get<T extends 'user' | 'product' | 'order'>(
    endpoint: T,
    id: number
  ): Observable<EndpointResponse<T>> {
    return this.http.get<EndpointResponse<T>>(`${this.baseUrl}/${endpoint}/${id}`);
  }

  // Unwrap response wrapper automatically
  getUnwrapped<T>(url: string): Observable<Unwrap<ApiResponse<T>>> {
    return this.http.get<ApiResponse<T>>(url).pipe(
      map(response => response.data as Unwrap<ApiResponse<T>>)
    );
  }

  // Deep partial update
  deepUpdate<T>(url: string, updates: DeepPartial<T>): Observable<T> {
    return this.http.patch<T>(url, updates);
  }
}
```

## Common Mistakes

### 1. Not Typing HTTP Responses

```typescript
// WRONG: Returns Observable<any>
getUser(id: number) {
  return this.http.get(`/api/users/${id}`);
}

// CORRECT: Type the response
getUser(id: number): Observable<User> {
  return this.http.get<User>(`/api/users/${id}`);
}
```

### 2. Incorrect Generic Usage

```typescript
// WRONG: Type doesn't match actual response
this.http.get<User>('/api/users') // Returns User[], not User

// CORRECT: Match type to actual response
this.http.get<User[]>('/api/users')
```

### 3. Ignoring Wrapped Responses

```typescript
// WRONG: Type doesn't account for wrapper
getUser(id: number): Observable<User> {
  return this.http.get<User>(`/api/users/${id}`);
  // But API returns { success: true, data: User }
}

// CORRECT: Type the wrapper and unwrap
getUser(id: number): Observable<User> {
  return this.http.get<ApiResponse<User>>(`/api/users/${id}`)
    .pipe(map(response => response.data));
}
```

### 4. Unsafe Type Assertions

```typescript
// WRONG: Unsafe assertion
const user = response as User; // No runtime check

// CORRECT: Use type guard
if (isUser(response)) {
  const user = response; // Safely typed
}
```

### 5. Missing Error Response Typing

```typescript
// WRONG: Untyped error
.catchError(error => {
  console.log(error.code); // error is any
})

// CORRECT: Type the error
.catchError((error: HttpErrorResponse) => {
  if (error.status === 404) {
    // Handle not found
  }
})
```

## Best Practices

### 1. Define Types in Separate Files

```typescript
// types/user.types.ts
export interface User { /* ... */ }
export interface CreateUserDto { /* ... */ }
export interface UpdateUserDto { /* ... */ }
```

### 2. Use Utility Types for DRY Types

```typescript
type CreateUser = Omit<User, 'id' | 'createdAt'>;
type UpdateUser = Partial<CreateUser>;
```

### 3. Type All HTTP Responses

```typescript
// Always specify the generic type
this.http.get<User[]>('/api/users')
this.http.post<User>('/api/users', data)
```

### 4. Create Generic Base Services

```typescript
abstract class BaseApiService<T> {
  getAll(): Observable<T[]> { /* ... */ }
  getById(id: number): Observable<T> { /* ... */ }
}
```

### 5. Use Discriminated Unions for States

```typescript
type LoadState<T> =
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: string };
```

## Interview Questions

**Q: How does HttpClient preserve type safety?**

A: HttpClient uses TypeScript generics. You specify the response type as a generic parameter: `http.get<User[]>()`. TypeScript then enforces this type throughout your code, but there's no runtime validation.

**Q: What's the difference between ApiResponse<User> and User?**

A: `ApiResponse<User>` is a generic wrapper type that contains a User plus metadata (success, message, etc.). `User` is just the entity. You need to map wrapped responses to extract the data.

**Q: How do you type an HTTP error response?**

A: Use `HttpErrorResponse` type from `@angular/common/http`:
```typescript
catchError((error: HttpErrorResponse) => {
  console.log(error.status, error.message);
})
```

**Q: What are discriminated unions good for in HTTP contexts?**

A: Representing different response states (loading, success, error) type-safely. TypeScript can narrow types based on a discriminant property like `status`.

**Q: How do you ensure runtime type safety with HttpClient?**

A: TypeScript types are compile-time only. For runtime validation, use type guard functions or validation libraries like Zod to verify response structure.

## Key Takeaways

1. Always specify generic type parameters for HttpClient methods: `http.get<Type>()`
2. Create separate DTOs for create/update operations vs. full entities
3. Use generic wrapper types like `ApiResponse<T>` for consistent API structures
4. Access full response metadata with `HttpResponse<T>` by setting `observe: 'response'`
5. Discriminated unions provide type-safe state management for async operations
6. TypeScript utility types (Omit, Pick, Partial) reduce type duplication
7. Type guards add runtime validation to compile-time types
8. Generic base services enforce consistent typing across resources
9. Interceptors can preserve and transform types through the request/response chain
10. Conditional types enable advanced type transformations for complex scenarios

## Resources

- [TypeScript Handbook - Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html)
- [TypeScript Utility Types](https://www.typescriptlang.org/docs/handbook/utility-types.html)
- [Angular HttpClient API](https://angular.io/api/common/http/HttpClient)
- [Type Guards and Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)
- [Discriminated Unions](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#discriminated-unions)
