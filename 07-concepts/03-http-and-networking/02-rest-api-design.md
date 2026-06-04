# REST API Design

## Overview

REST (Representational State Transfer) is an architectural style for designing networked applications. It uses HTTP methods to perform CRUD operations on resources, treats everything as a resource with a unique identifier (URL), and leverages HTTP status codes for communication. Proper REST API design creates intuitive, scalable, and maintainable APIs.

## Core Concepts

### REST Principles

```
REST Constraints:
┌──────────────────────────────────────┐
│  1. Client-Server Separation         │
│  2. Stateless Communication          │
│  3. Cacheable Responses              │
│  4. Uniform Interface                │
│  5. Layered System                   │
│  6. Code on Demand (optional)        │
└──────────────────────────────────────┘

Resource-Based:
┌────────────┐
│  Resource  │ ← Noun, not verb
│   /users   │
│  /products │
└────────────┘
      │
      ▼
┌────────────┐
│   Method   │ ← HTTP verbs
│  GET/POST  │
│  PUT/DELETE│
└────────────┘
```

### HTTP Methods and CRUD

| HTTP Method | CRUD Operation | Idempotent | Safe | Usage |
|-------------|----------------|------------|------|-------|
| GET | Read | Yes | Yes | Retrieve resource(s) |
| POST | Create | No | No | Create new resource |
| PUT | Update | Yes | No | Replace entire resource |
| PATCH | Update | No | No | Partial update |
| DELETE | Delete | Yes | No | Remove resource |
| HEAD | Read metadata | Yes | Yes | Get headers only |
| OPTIONS | Discover methods | Yes | Yes | CORS preflight |

### Status Code Categories

```
1xx: Informational
├── 100 Continue
└── 101 Switching Protocols

2xx: Success
├── 200 OK
├── 201 Created
├── 202 Accepted
├── 204 No Content
└── 206 Partial Content

3xx: Redirection
├── 301 Moved Permanently
├── 302 Found
├── 304 Not Modified
└── 307 Temporary Redirect

4xx: Client Error
├── 400 Bad Request
├── 401 Unauthorized
├── 403 Forbidden
├── 404 Not Found
├── 409 Conflict
├── 422 Unprocessable Entity
└── 429 Too Many Requests

5xx: Server Error
├── 500 Internal Server Error
├── 502 Bad Gateway
├── 503 Service Unavailable
└── 504 Gateway Timeout
```

## Resource Modeling

### Resource Naming Conventions

```typescript
// Good - plural nouns
GET    /users              // List all users
GET    /users/123          // Get specific user
POST   /users              // Create new user
PUT    /users/123          // Update user
DELETE /users/123          // Delete user

// Good - nested resources (relationships)
GET    /users/123/posts           // User's posts
GET    /users/123/posts/456       // Specific post by user
POST   /users/123/posts           // Create post for user

// Good - collections and filters
GET    /users?role=admin           // Filter users
GET    /posts?author=123&status=published  // Multiple filters
GET    /products?sort=price&order=desc     // Sorting

// Bad - verbs in URLs
POST   /createUser          // ❌ Use POST /users
GET    /getUser/123         // ❌ Use GET /users/123
POST   /deleteUser/123      // ❌ Use DELETE /users/123

// Bad - actions as resources
POST   /users/123/activate  // ❌ Consider PUT /users/123 with status
POST   /users/123/disable   // ❌ Use PATCH /users/123 { "active": false }

// Acceptable - for operations that don't fit CRUD
POST   /users/123/send-verification-email
POST   /orders/123/cancel
POST   /payments/123/refund
```

## React Examples

### Complete REST Client

```typescript
// api/rest-client.ts
interface RequestConfig extends RequestInit {
  params?: Record<string, string | number | boolean>;
}

class RestClient {
  constructor(private baseUrl: string) {}
  
  private buildUrl(endpoint: string, params?: Record<string, any>): string {
    const url = new URL(endpoint, this.baseUrl);
    
    if (params) {
      Object.entries(params).forEach(([key, value]) => {
        if (value !== undefined && value !== null) {
          url.searchParams.append(key, String(value));
        }
      });
    }
    
    return url.toString();
  }
  
  private async request<T>(
    endpoint: string,
    config: RequestConfig = {}
  ): Promise<T> {
    const { params, ...fetchConfig } = config;
    const url = this.buildUrl(endpoint, params);
    
    const response = await fetch(url, {
      ...fetchConfig,
      headers: {
        'Content-Type': 'application/json',
        ...fetchConfig.headers
      }
    });
    
    if (!response.ok) {
      throw await this.handleError(response);
    }
    
    // Handle 204 No Content
    if (response.status === 204) {
      return null as T;
    }
    
    return response.json();
  }
  
  private async handleError(response: Response): Promise<Error> {
    const contentType = response.headers.get('content-type');
    
    if (contentType?.includes('application/json')) {
      const error = await response.json();
      return new Error(error.message || response.statusText);
    }
    
    return new Error(response.statusText);
  }
  
  // GET - retrieve resource(s)
  async get<T>(endpoint: string, params?: Record<string, any>): Promise<T> {
    return this.request<T>(endpoint, { method: 'GET', params });
  }
  
  // POST - create resource
  async post<T>(endpoint: string, data?: any): Promise<T> {
    return this.request<T>(endpoint, {
      method: 'POST',
      body: JSON.stringify(data)
    });
  }
  
  // PUT - replace resource
  async put<T>(endpoint: string, data: any): Promise<T> {
    return this.request<T>(endpoint, {
      method: 'PUT',
      body: JSON.stringify(data)
    });
  }
  
  // PATCH - partial update
  async patch<T>(endpoint: string, data: any): Promise<T> {
    return this.request<T>(endpoint, {
      method: 'PATCH',
      body: JSON.stringify(data)
    });
  }
  
  // DELETE - remove resource
  async delete<T = void>(endpoint: string): Promise<T> {
    return this.request<T>(endpoint, { method: 'DELETE' });
  }
  
  // HEAD - get headers only
  async head(endpoint: string): Promise<Headers> {
    const url = this.buildUrl(endpoint);
    const response = await fetch(url, { method: 'HEAD' });
    return response.headers;
  }
}

export const api = new RestClient(process.env.REACT_APP_API_URL || '');
```

### User Resource API

```typescript
// api/users.api.ts
export interface User {
  id: string;
  name: string;
  email: string;
  role: 'user' | 'admin';
  createdAt: string;
  updatedAt: string;
}

export interface CreateUserDto {
  name: string;
  email: string;
  password: string;
  role?: 'user' | 'admin';
}

export interface UpdateUserDto {
  name?: string;
  email?: string;
  role?: 'user' | 'admin';
}

export interface ListUsersParams {
  page?: number;
  limit?: number;
  role?: 'user' | 'admin';
  search?: string;
  sort?: 'name' | 'email' | 'createdAt';
  order?: 'asc' | 'desc';
}

export interface ListResponse<T> {
  data: T[];
  total: number;
  page: number;
  pageSize: number;
}

export const usersApi = {
  // GET /users - List users with filtering and pagination
  list: (params?: ListUsersParams): Promise<ListResponse<User>> => {
    return api.get<ListResponse<User>>('/users', params);
  },
  
  // GET /users/:id - Get single user
  get: (id: string): Promise<User> => {
    return api.get<User>(`/users/${id}`);
  },
  
  // POST /users - Create new user
  create: (data: CreateUserDto): Promise<User> => {
    return api.post<User>('/users', data);
  },
  
  // PUT /users/:id - Replace user (all fields)
  replace: (id: string, data: User): Promise<User> => {
    return api.put<User>(`/users/${id}`, data);
  },
  
  // PATCH /users/:id - Update user (partial)
  update: (id: string, data: UpdateUserDto): Promise<User> => {
    return api.patch<User>(`/users/${id}`, data);
  },
  
  // DELETE /users/:id - Delete user
  delete: (id: string): Promise<void> => {
    return api.delete(`/users/${id}`);
  }
};

// Usage in component
function UserList() {
  const [users, setUsers] = useState<User[]>([]);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    const loadUsers = async () => {
      try {
        const response = await usersApi.list({
          role: 'admin',
          sort: 'name',
          order: 'asc',
          page: 1,
          limit: 20
        });
        setUsers(response.data);
      } catch (error) {
        console.error('Failed to load users:', error);
      } finally {
        setLoading(false);
      }
    };
    
    loadUsers();
  }, []);
  
  const handleDelete = async (id: string) => {
    try {
      await usersApi.delete(id);
      setUsers(users.filter(u => u.id !== id));
    } catch (error) {
      console.error('Failed to delete user:', error);
    }
  };
  
  if (loading) return <div>Loading...</div>;
  
  return (
    <div>
      {users.map(user => (
        <div key={user.id}>
          <span>{user.name}</span>
          <button onClick={() => handleDelete(user.id)}>Delete</button>
        </div>
      ))}
    </div>
  );
}
```

### Nested Resources

```typescript
// api/posts.api.ts
export interface Post {
  id: string;
  userId: string;
  title: string;
  content: string;
  status: 'draft' | 'published';
  createdAt: string;
}

export interface Comment {
  id: string;
  postId: string;
  userId: string;
  text: string;
  createdAt: string;
}

export const postsApi = {
  // User's posts
  listUserPosts: (userId: string): Promise<Post[]> => {
    return api.get<Post[]>(`/users/${userId}/posts`);
  },
  
  getUserPost: (userId: string, postId: string): Promise<Post> => {
    return api.get<Post>(`/users/${userId}/posts/${postId}`);
  },
  
  createUserPost: (userId: string, data: Partial<Post>): Promise<Post> => {
    return api.post<Post>(`/users/${userId}/posts`, data);
  },
  
  // Post comments (nested two levels)
  listPostComments: (postId: string): Promise<Comment[]> => {
    return api.get<Comment[]>(`/posts/${postId}/comments`);
  },
  
  createPostComment: (postId: string, data: Partial<Comment>): Promise<Comment> => {
    return api.post<Comment>(`/posts/${postId}/comments`, data);
  },
  
  // Alternative: non-nested (when relationship is not hierarchical)
  listAllPosts: (params?: { userId?: string; status?: string }): Promise<Post[]> => {
    return api.get<Post[]>('/posts', params);
  }
};

// Usage
function UserPosts({ userId }: { userId: string }) {
  const [posts, setPosts] = useState<Post[]>([]);
  
  useEffect(() => {
    postsApi.listUserPosts(userId).then(setPosts);
  }, [userId]);
  
  const handleCreatePost = async (title: string, content: string) => {
    const newPost = await postsApi.createUserPost(userId, {
      title,
      content,
      status: 'draft'
    });
    setPosts([...posts, newPost]);
  };
  
  return (
    <div>
      {posts.map(post => (
        <PostItem key={post.id} post={post} />
      ))}
    </div>
  );
}
```

### Error Handling

```typescript
// api/errors.ts
export class ApiError extends Error {
  constructor(
    message: string,
    public status: number,
    public code?: string,
    public details?: any
  ) {
    super(message);
    this.name = 'ApiError';
  }
  
  static isBadRequest(error: unknown): error is ApiError {
    return error instanceof ApiError && error.status === 400;
  }
  
  static isUnauthorized(error: unknown): error is ApiError {
    return error instanceof ApiError && error.status === 401;
  }
  
  static isForbidden(error: unknown): error is ApiError {
    return error instanceof ApiError && error.status === 403;
  }
  
  static isNotFound(error: unknown): error is ApiError {
    return error instanceof ApiError && error.status === 404;
  }
  
  static isConflict(error: unknown): error is ApiError {
    return error instanceof ApiError && error.status === 409;
  }
  
  static isServerError(error: unknown): error is ApiError {
    return error instanceof ApiError && error.status >= 500;
  }
}

// Enhanced rest client with proper error handling
class RestClient {
  private async handleError(response: Response): Promise<never> {
    const contentType = response.headers.get('content-type');
    let errorData: any = {};
    
    if (contentType?.includes('application/json')) {
      errorData = await response.json();
    }
    
    throw new ApiError(
      errorData.message || response.statusText,
      response.status,
      errorData.code,
      errorData.details
    );
  }
  
  // ... rest of implementation
}

// Usage with error handling
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  const [error, setError] = useState<string | null>(null);
  
  useEffect(() => {
    const loadUser = async () => {
      try {
        const data = await usersApi.get(userId);
        setUser(data);
        setError(null);
      } catch (err) {
        if (ApiError.isNotFound(err)) {
          setError('User not found');
        } else if (ApiError.isUnauthorized(err)) {
          setError('Please log in to view this profile');
        } else if (ApiError.isServerError(err)) {
          setError('Server error. Please try again later.');
        } else {
          setError('An unexpected error occurred');
        }
      }
    };
    
    loadUser();
  }, [userId]);
  
  if (error) return <div className="error">{error}</div>;
  if (!user) return <div>Loading...</div>;
  
  return <div>{user.name}</div>;
}
```

## Angular Examples

### REST Service with HttpClient

```typescript
// services/rest.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams, HttpHeaders, HttpErrorResponse } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, retry } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class RestService {
  constructor(
    private http: HttpClient,
    @Inject('API_URL') private apiUrl: string
  ) {}
  
  private buildUrl(endpoint: string): string {
    return `${this.apiUrl}${endpoint}`;
  }
  
  private buildParams(params?: Record<string, any>): HttpParams {
    let httpParams = new HttpParams();
    
    if (params) {
      Object.entries(params).forEach(([key, value]) => {
        if (value !== undefined && value !== null) {
          httpParams = httpParams.append(key, String(value));
        }
      });
    }
    
    return httpParams;
  }
  
  private handleError(error: HttpErrorResponse): Observable<never> {
    let errorMessage = 'An error occurred';
    
    if (error.error instanceof ErrorEvent) {
      // Client-side error
      errorMessage = error.error.message;
    } else {
      // Server-side error
      errorMessage = error.error?.message || error.statusText;
    }
    
    console.error('API Error:', errorMessage);
    return throwError(() => new Error(errorMessage));
  }
  
  get<T>(endpoint: string, params?: Record<string, any>): Observable<T> {
    return this.http.get<T>(this.buildUrl(endpoint), {
      params: this.buildParams(params)
    }).pipe(
      retry(1),
      catchError(this.handleError)
    );
  }
  
  post<T>(endpoint: string, data?: any): Observable<T> {
    return this.http.post<T>(this.buildUrl(endpoint), data).pipe(
      catchError(this.handleError)
    );
  }
  
  put<T>(endpoint: string, data: any): Observable<T> {
    return this.http.put<T>(this.buildUrl(endpoint), data).pipe(
      catchError(this.handleError)
    );
  }
  
  patch<T>(endpoint: string, data: any): Observable<T> {
    return this.http.patch<T>(this.buildUrl(endpoint), data).pipe(
      catchError(this.handleError)
    );
  }
  
  delete<T = void>(endpoint: string): Observable<T> {
    return this.http.delete<T>(this.buildUrl(endpoint)).pipe(
      catchError(this.handleError)
    );
  }
}
```

### Resource Service

```typescript
// services/user.service.ts
@Injectable({
  providedIn: 'root'
})
export class UserService {
  private readonly endpoint = '/users';
  
  constructor(private rest: RestService) {}
  
  list(params?: ListUsersParams): Observable<ListResponse<User>> {
    return this.rest.get<ListResponse<User>>(this.endpoint, params);
  }
  
  get(id: string): Observable<User> {
    return this.rest.get<User>(`${this.endpoint}/${id}`);
  }
  
  create(data: CreateUserDto): Observable<User> {
    return this.rest.post<User>(this.endpoint, data);
  }
  
  update(id: string, data: UpdateUserDto): Observable<User> {
    return this.rest.patch<User>(`${this.endpoint}/${id}`, data);
  }
  
  delete(id: string): Observable<void> {
    return this.rest.delete(`${this.endpoint}/${id}`);
  }
}

// Component usage
@Component({
  selector: 'app-user-list',
  template: `
    <div *ngIf="loading">Loading...</div>
    <div *ngIf="error" class="error">{{ error }}</div>
    <div *ngFor="let user of users">
      <span>{{ user.name }}</span>
      <button (click)="deleteUser(user.id)">Delete</button>
    </div>
  `
})
export class UserListComponent implements OnInit {
  users: User[] = [];
  loading = true;
  error: string | null = null;
  
  constructor(private userService: UserService) {}
  
  ngOnInit() {
    this.loadUsers();
  }
  
  private loadUsers() {
    this.loading = true;
    this.error = null;
    
    this.userService.list({ role: 'admin', sort: 'name' }).subscribe({
      next: (response) => {
        this.users = response.data;
        this.loading = false;
      },
      error: (error) => {
        this.error = error.message;
        this.loading = false;
      }
    });
  }
  
  deleteUser(id: string) {
    if (!confirm('Are you sure?')) return;
    
    this.userService.delete(id).subscribe({
      next: () => {
        this.users = this.users.filter(u => u.id !== id);
      },
      error: (error) => {
        this.error = error.message;
      }
    });
  }
}
```

### HTTP Interceptor for REST

```typescript
// interceptors/rest.interceptor.ts
@Injectable()
export class RestInterceptor implements HttpInterceptor {
  constructor(private authService: AuthService) {}
  
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // Add authentication token
    const token = this.authService.getToken();
    let authReq = req;
    
    if (token) {
      authReq = req.clone({
        headers: req.headers.set('Authorization', `Bearer ${token}`)
      });
    }
    
    // Add common headers
    authReq = authReq.clone({
      headers: authReq.headers
        .set('Accept', 'application/json')
        .set('X-Requested-With', 'XMLHttpRequest')
    });
    
    // Handle request
    return next.handle(authReq).pipe(
      tap(event => {
        if (event instanceof HttpResponse) {
          console.log(`${req.method} ${req.url} - ${event.status}`);
        }
      }),
      catchError((error: HttpErrorResponse) => {
        if (error.status === 401) {
          this.authService.logout();
        }
        return throwError(() => error);
      })
    );
  }
}

// Register in module
@NgModule({
  providers: [
    {
      provide: HTTP_INTERCEPTORS,
      useClass: RestInterceptor,
      multi: true
    }
  ]
})
export class AppModule {}
```

## Advanced Patterns

### HATEOAS (Hypermedia)

```typescript
// Response with links
interface UserWithLinks {
  id: string;
  name: string;
  email: string;
  _links: {
    self: { href: string };
    posts: { href: string };
    avatar: { href: string };
    edit?: { href: string }; // Only if user can edit
    delete?: { href: string }; // Only if user can delete
  };
}

// Client follows links
async function getUserPosts(user: UserWithLinks): Promise<Post[]> {
  const postsUrl = user._links.posts.href;
  return api.get<Post[]>(postsUrl);
}

// Check capabilities
function canEditUser(user: UserWithLinks): boolean {
  return !!user._links.edit;
}
```

### Versioning

```typescript
// URL versioning
const apiV1 = new RestClient('https://api.example.com/v1');
const apiV2 = new RestClient('https://api.example.com/v2');

// Header versioning
class RestClient {
  async request<T>(endpoint: string, config: RequestConfig): Promise<T> {
    return fetch(endpoint, {
      ...config,
      headers: {
        ...config.headers,
        'Accept': 'application/vnd.api+json; version=2'
      }
    });
  }
}

// Content negotiation
headers: {
  'Accept': 'application/vnd.myapp.v2+json'
}
```

### Bulk Operations

```typescript
// Bulk create
interface BulkCreateResponse<T> {
  created: T[];
  errors: Array<{ index: number; error: string }>;
}

export const usersApi = {
  bulkCreate: (users: CreateUserDto[]): Promise<BulkCreateResponse<User>> => {
    return api.post<BulkCreateResponse<User>>('/users/bulk', { users });
  },
  
  bulkUpdate: (updates: Array<{ id: string; data: UpdateUserDto }>): Promise<User[]> => {
    return api.patch<User[]>('/users/bulk', { updates });
  },
  
  bulkDelete: (ids: string[]): Promise<void> => {
    return api.post('/users/bulk-delete', { ids });
  }
};
```

## Common Mistakes

### 1. Using Verbs in URLs

```typescript
// Bad
POST /createUser
GET  /getUsers
POST /updateUser/123

// Good
POST   /users
GET    /users
PUT    /users/123
```

### 2. Wrong HTTP Methods

```typescript
// Bad - using GET for actions with side effects
GET /users/123/delete

// Good
DELETE /users/123

// Bad - using POST for updates
POST /users/123/update

// Good
PUT   /users/123  // Full update
PATCH /users/123  // Partial update
```

### 3. Inconsistent Status Codes

```typescript
// Bad - returning 200 for errors
fetch('/users/123').then(res => {
  // res.status === 200, but body has error
  const data = await res.json();
  if (data.error) { /* handle error */ }
});

// Good - proper status codes
fetch('/users/123').then(res => {
  if (!res.ok) {
    if (res.status === 404) {
      // User not found
    }
    throw new Error(res.statusText);
  }
  return res.json();
});
```

### 4. Not Using Query Parameters

```typescript
// Bad - parameters in URL segments
GET /users/filter/admin/sort/name

// Good - query parameters
GET /users?role=admin&sort=name
```

## Best Practices

### 1. Use Plural Nouns for Collections

```typescript
// Good
GET /users
GET /products
GET /orders

// Bad
GET /user
GET /product
```

### 2. Return Appropriate Status Codes

```typescript
// Success codes
200 OK          // GET, PUT, PATCH successful
201 Created     // POST successful
204 No Content  // DELETE successful

// Client error codes
400 Bad Request       // Invalid data
401 Unauthorized      // Not authenticated
403 Forbidden         // Not authorized
404 Not Found         // Resource doesn't exist
409 Conflict          // Duplicate/conflict
422 Unprocessable     // Validation failed

// Server error codes
500 Internal Error    // Server problem
503 Service Unavailable // Temporarily down
```

### 3. Provide Filtering and Pagination

```typescript
GET /users?page=1&limit=20
GET /users?role=admin&status=active
GET /users?sort=createdAt&order=desc
GET /users?search=john
```

### 4. Use Consistent Response Format

```typescript
// Success response
{
  "data": { /* resource */ },
  "meta": {
    "timestamp": "2024-01-01T00:00:00Z"
  }
}

// List response
{
  "data": [ /* resources */ ],
  "meta": {
    "total": 100,
    "page": 1,
    "pageSize": 20
  }
}

// Error response
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid email address",
    "details": {
      "field": "email",
      "value": "invalid"
    }
  }
}
```

## Interview Questions

### Q1: What are the key constraints of REST architecture?
**Answer**: 1) Client-server separation, 2) Stateless communication (each request contains all necessary information), 3) Cacheable responses, 4) Uniform interface (standard HTTP methods), 5) Layered system, 6) Code on demand (optional). These constraints promote scalability, simplicity, and reliability.

### Q2: What's the difference between PUT and PATCH?
**Answer**: PUT replaces the entire resource with the provided data (requires all fields). PATCH performs a partial update, only modifying specified fields. PUT is idempotent (multiple identical requests have the same effect), while PATCH may or may not be idempotent depending on implementation.

### Q3: When should you nest resources in URLs?
**Answer**: Nest resources when there's a clear parent-child relationship and the child resource doesn't make sense without the parent (e.g., /users/123/posts). Don't nest more than 2-3 levels deep. For resources that can exist independently or have multiple relationships, use query parameters instead.

### Q4: How do you handle errors in REST APIs?
**Answer**: Use appropriate HTTP status codes (4xx for client errors, 5xx for server errors), return consistent error response format with code, message, and details, log errors server-side, don't expose sensitive information in error messages, and provide actionable error messages for clients.

## Key Takeaways

1. REST is resource-based with standard HTTP methods
2. Use nouns (not verbs) in URLs
3. HTTP methods map to CRUD operations
4. Status codes communicate operation results
5. Resources should have plural names
6. Use query parameters for filtering and pagination
7. PUT replaces, PATCH updates partially
8. Proper error handling with status codes
9. Keep URLs simple and predictable
10. Document your API thoroughly

## Resources

- REST API Tutorial: https://restfulapi.net/
- HTTP Status Codes: https://httpstatuses.com/
- Roy Fielding's Dissertation: https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm
- Best Practices for REST API Design
- OpenAPI Specification: https://swagger.io/specification/
