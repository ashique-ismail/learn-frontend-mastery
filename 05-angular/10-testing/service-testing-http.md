# Service Testing with HTTP in Angular

## Table of Contents
- [Introduction](#introduction)
- [HttpTestingController Fundamentals](#httptestingcontroller-fundamentals)
- [Testing GET Requests](#testing-get-requests)
- [Testing POST/PUT/DELETE Requests](#testing-postputdelete-requests)
- [Testing Error Scenarios](#testing-error-scenarios)
- [Testing Request Headers and Parameters](#testing-request-headers-and-parameters)
- [Testing Concurrent Requests](#testing-concurrent-requests)
- [Advanced HTTP Testing Patterns](#advanced-http-testing-patterns)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Testing Angular services that make HTTP requests is crucial for ensuring application reliability. The `HttpTestingController` from `@angular/common/http/testing` provides a robust API for mocking HTTP requests, controlling responses, and verifying that services interact correctly with backend APIs without making actual network calls.

This guide covers comprehensive patterns for testing HTTP services, from basic GET requests to complex error handling and concurrent request scenarios.

## HttpTestingController Fundamentals

### Basic Setup

```typescript
import { TestBed } from '@angular/core/testing';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';
import { UserService } from './user.service';

describe('UserService', () => {
  let service: UserService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [UserService]
    });

    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    // Verify that no unmatched requests are outstanding
    httpMock.verify();
  });

  it('should be created', () => {
    expect(service).toBeTruthy();
  });
});
```

### Service Under Test

```typescript
import { Injectable } from '@angular/core';
import { HttpClient, HttpHeaders, HttpParams } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, retry } from 'rxjs/operators';

export interface User {
  id: number;
  name: string;
  email: string;
  role: string;
}

@Injectable({
  providedIn: 'root'
})
export class UserService {
  private apiUrl = 'https://api.example.com/users';

  constructor(private http: HttpClient) {}

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl);
  }

  getUserById(id: number): Observable<User> {
    return this.http.get<User>(`${this.apiUrl}/${id}`);
  }

  createUser(user: Partial<User>): Observable<User> {
    return this.http.post<User>(this.apiUrl, user);
  }

  updateUser(id: number, user: Partial<User>): Observable<User> {
    return this.http.put<User>(`${this.apiUrl}/${id}`, user);
  }

  deleteUser(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`);
  }

  searchUsers(query: string): Observable<User[]> {
    const params = new HttpParams().set('q', query);
    return this.http.get<User[]>(`${this.apiUrl}/search`, { params });
  }
}
```

## Testing GET Requests

### Basic GET Request Test

```typescript
it('should fetch all users', () => {
  const mockUsers: User[] = [
    { id: 1, name: 'John Doe', email: 'john@example.com', role: 'admin' },
    { id: 2, name: 'Jane Smith', email: 'jane@example.com', role: 'user' }
  ];

  service.getUsers().subscribe(users => {
    expect(users).toEqual(mockUsers);
    expect(users.length).toBe(2);
  });

  // Expect a single GET request to the API URL
  const req = httpMock.expectOne('https://api.example.com/users');
  
  // Assert that the request is a GET request
  expect(req.request.method).toBe('GET');
  
  // Respond with mock data
  req.flush(mockUsers);
});
```

### Testing GET with Path Parameters

```typescript
it('should fetch a user by id', () => {
  const mockUser: User = {
    id: 1,
    name: 'John Doe',
    email: 'john@example.com',
    role: 'admin'
  };

  service.getUserById(1).subscribe(user => {
    expect(user).toEqual(mockUser);
    expect(user.id).toBe(1);
  });

  const req = httpMock.expectOne('https://api.example.com/users/1');
  expect(req.request.method).toBe('GET');
  req.flush(mockUser);
});
```

### Testing GET with Query Parameters

```typescript
it('should search users with query parameters', () => {
  const mockResults: User[] = [
    { id: 1, name: 'John Doe', email: 'john@example.com', role: 'admin' }
  ];

  service.searchUsers('john').subscribe(users => {
    expect(users).toEqual(mockResults);
    expect(users.length).toBe(1);
  });

  const req = httpMock.expectOne(request => 
    request.url === 'https://api.example.com/users/search' &&
    request.params.get('q') === 'john'
  );

  expect(req.request.method).toBe('GET');
  expect(req.request.params.get('q')).toBe('john');
  req.flush(mockResults);
});
```

## Testing POST/PUT/DELETE Requests

### Testing POST Requests

```typescript
it('should create a new user', () => {
  const newUser: Partial<User> = {
    name: 'New User',
    email: 'new@example.com',
    role: 'user'
  };

  const createdUser: User = {
    id: 3,
    ...newUser as User
  };

  service.createUser(newUser).subscribe(user => {
    expect(user).toEqual(createdUser);
    expect(user.id).toBe(3);
  });

  const req = httpMock.expectOne('https://api.example.com/users');
  expect(req.request.method).toBe('POST');
  expect(req.request.body).toEqual(newUser);
  
  req.flush(createdUser);
});
```

### Testing PUT Requests

```typescript
it('should update an existing user', () => {
  const userId = 1;
  const updates: Partial<User> = {
    name: 'Updated Name',
    email: 'updated@example.com'
  };

  const updatedUser: User = {
    id: userId,
    name: 'Updated Name',
    email: 'updated@example.com',
    role: 'admin'
  };

  service.updateUser(userId, updates).subscribe(user => {
    expect(user).toEqual(updatedUser);
    expect(user.name).toBe('Updated Name');
  });

  const req = httpMock.expectOne(`https://api.example.com/users/${userId}`);
  expect(req.request.method).toBe('PUT');
  expect(req.request.body).toEqual(updates);
  
  req.flush(updatedUser);
});
```

### Testing DELETE Requests

```typescript
it('should delete a user', () => {
  const userId = 1;

  service.deleteUser(userId).subscribe(response => {
    expect(response).toBeUndefined();
  });

  const req = httpMock.expectOne(`https://api.example.com/users/${userId}`);
  expect(req.request.method).toBe('DELETE');
  
  // DELETE often returns empty response
  req.flush(null);
});
```

## Testing Error Scenarios

### Testing 404 Not Found

```typescript
it('should handle 404 error when user not found', () => {
  const userId = 999;

  service.getUserById(userId).subscribe(
    () => fail('should have failed with 404 error'),
    (error) => {
      expect(error.status).toBe(404);
      expect(error.statusText).toBe('Not Found');
    }
  );

  const req = httpMock.expectOne(`https://api.example.com/users/${userId}`);
  
  req.flush('User not found', {
    status: 404,
    statusText: 'Not Found'
  });
});
```

### Testing 500 Server Error

```typescript
it('should handle 500 server error', () => {
  service.getUsers().subscribe(
    () => fail('should have failed with 500 error'),
    (error) => {
      expect(error.status).toBe(500);
      expect(error.error).toBe('Internal server error');
    }
  );

  const req = httpMock.expectOne('https://api.example.com/users');
  
  req.flush('Internal server error', {
    status: 500,
    statusText: 'Internal Server Error'
  });
});
```

### Testing Network Errors

```typescript
it('should handle network error', () => {
  service.getUsers().subscribe(
    () => fail('should have failed with network error'),
    (error) => {
      expect(error.error.type).toBe('error');
    }
  );

  const req = httpMock.expectOne('https://api.example.com/users');
  
  // Simulate a network error
  req.error(new ProgressEvent('error'));
});
```

### Testing Service with Error Handling

```typescript
// Enhanced service with error handling
@Injectable({
  providedIn: 'root'
})
export class UserServiceWithErrorHandling {
  private apiUrl = 'https://api.example.com/users';

  constructor(private http: HttpClient) {}

  getUsersWithRetry(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl).pipe(
      retry(3),
      catchError(this.handleError)
    );
  }

  private handleError(error: any): Observable<never> {
    console.error('An error occurred:', error);
    return throwError(() => new Error('Something went wrong'));
  }
}

// Test
it('should retry failed requests 3 times', () => {
  let callCount = 0;

  service.getUsersWithRetry().subscribe(
    () => fail('should have failed after retries'),
    (error) => {
      expect(error.message).toBe('Something went wrong');
      expect(callCount).toBe(4); // Initial + 3 retries
    }
  );

  // Expect 4 requests total (1 initial + 3 retries)
  for (let i = 0; i < 4; i++) {
    const req = httpMock.expectOne('https://api.example.com/users');
    callCount++;
    req.flush('Error', { status: 500, statusText: 'Server Error' });
  }
});
```

## Testing Request Headers and Parameters

### Testing Custom Headers

```typescript
// Service with authentication headers
@Injectable({
  providedIn: 'root'
})
export class AuthUserService {
  private apiUrl = 'https://api.example.com/users';

  constructor(private http: HttpClient) {}

  getUsersWithAuth(token: string): Observable<User[]> {
    const headers = new HttpHeaders({
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    });

    return this.http.get<User[]>(this.apiUrl, { headers });
  }
}

// Test
it('should send request with authorization header', () => {
  const token = 'test-token-123';
  const mockUsers: User[] = [
    { id: 1, name: 'John', email: 'john@example.com', role: 'admin' }
  ];

  service.getUsersWithAuth(token).subscribe(users => {
    expect(users).toEqual(mockUsers);
  });

  const req = httpMock.expectOne('https://api.example.com/users');
  
  expect(req.request.headers.get('Authorization')).toBe(`Bearer ${token}`);
  expect(req.request.headers.get('Content-Type')).toBe('application/json');
  
  req.flush(mockUsers);
});
```

### Testing Multiple Query Parameters

```typescript
// Service with complex filtering
@Injectable({
  providedIn: 'root'
})
export class FilteredUserService {
  private apiUrl = 'https://api.example.com/users';

  constructor(private http: HttpClient) {}

  getFilteredUsers(role: string, page: number, limit: number): Observable<User[]> {
    const params = new HttpParams()
      .set('role', role)
      .set('page', page.toString())
      .set('limit', limit.toString());

    return this.http.get<User[]>(this.apiUrl, { params });
  }
}

// Test
it('should send request with multiple query parameters', () => {
  const mockUsers: User[] = [
    { id: 1, name: 'Admin User', email: 'admin@example.com', role: 'admin' }
  ];

  service.getFilteredUsers('admin', 1, 10).subscribe(users => {
    expect(users).toEqual(mockUsers);
  });

  const req = httpMock.expectOne(request => {
    return request.url === 'https://api.example.com/users' &&
           request.params.get('role') === 'admin' &&
           request.params.get('page') === '1' &&
           request.params.get('limit') === '10';
  });

  expect(req.request.method).toBe('GET');
  expect(req.request.params.keys().length).toBe(3);
  
  req.flush(mockUsers);
});
```

## Testing Concurrent Requests

### Testing Multiple Simultaneous Requests

```typescript
it('should handle multiple concurrent requests', () => {
  const mockUser1: User = { id: 1, name: 'User 1', email: 'user1@example.com', role: 'user' };
  const mockUser2: User = { id: 2, name: 'User 2', email: 'user2@example.com', role: 'admin' };

  let user1Result: User | undefined;
  let user2Result: User | undefined;

  // Make two concurrent requests
  service.getUserById(1).subscribe(user => user1Result = user);
  service.getUserById(2).subscribe(user => user2Result = user);

  // Both requests should be pending
  const requests = httpMock.match(req => req.url.startsWith('https://api.example.com/users/'));
  expect(requests.length).toBe(2);

  // Respond to first request
  const req1 = httpMock.expectOne('https://api.example.com/users/1');
  req1.flush(mockUser1);

  // Respond to second request
  const req2 = httpMock.expectOne('https://api.example.com/users/2');
  req2.flush(mockUser2);

  // Verify both completed
  expect(user1Result).toEqual(mockUser1);
  expect(user2Result).toEqual(mockUser2);
});
```

### Testing Request Cancellation

```typescript
it('should handle cancelled requests', () => {
  const subscription = service.getUsers().subscribe();

  // Cancel the subscription before responding
  subscription.unsubscribe();

  const req = httpMock.expectOne('https://api.example.com/users');
  
  // Even though we respond, the subscriber won't receive it
  req.flush([]);
  
  // Verify the request was made but cancelled
  expect(req.cancelled).toBe(false); // Request itself wasn't cancelled
  expect(subscription.closed).toBe(true); // But subscription was closed
});
```

## Advanced HTTP Testing Patterns

### Testing with RxJS Operators

```typescript
// Service with data transformation
@Injectable({
  providedIn: 'root'
})
export class TransformUserService {
  private apiUrl = 'https://api.example.com/users';

  constructor(private http: HttpClient) {}

  getUserNames(): Observable<string[]> {
    return this.http.get<User[]>(this.apiUrl).pipe(
      map(users => users.map(user => user.name)),
      tap(names => console.log('User names:', names))
    );
  }

  getActiveUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl).pipe(
      map(users => users.filter(user => user.role !== 'inactive')),
      shareReplay(1)
    );
  }
}

// Test
it('should transform users to names array', () => {
  const mockUsers: User[] = [
    { id: 1, name: 'John', email: 'john@example.com', role: 'admin' },
    { id: 2, name: 'Jane', email: 'jane@example.com', role: 'user' }
  ];

  service.getUserNames().subscribe(names => {
    expect(names).toEqual(['John', 'Jane']);
    expect(names.length).toBe(2);
  });

  const req = httpMock.expectOne('https://api.example.com/users');
  req.flush(mockUsers);
});
```

### Testing Caching Mechanisms

```typescript
// Service with caching
@Injectable({
  providedIn: 'root'
})
export class CachedUserService {
  private apiUrl = 'https://api.example.com/users';
  private cache$ = new BehaviorSubject<User[] | null>(null);

  constructor(private http: HttpClient) {}

  getUsers(forceRefresh = false): Observable<User[]> {
    if (!forceRefresh && this.cache$.value) {
      return this.cache$.asObservable().pipe(
        filter(users => users !== null),
        map(users => users!)
      );
    }

    return this.http.get<User[]>(this.apiUrl).pipe(
      tap(users => this.cache$.next(users))
    );
  }

  clearCache(): void {
    this.cache$.next(null);
  }
}

// Test
it('should cache users and not make duplicate requests', () => {
  const mockUsers: User[] = [
    { id: 1, name: 'John', email: 'john@example.com', role: 'admin' }
  ];

  // First call - should hit the API
  service.getUsers().subscribe(users => {
    expect(users).toEqual(mockUsers);
  });

  const req = httpMock.expectOne('https://api.example.com/users');
  req.flush(mockUsers);

  // Second call - should use cache, no HTTP request
  service.getUsers().subscribe(users => {
    expect(users).toEqual(mockUsers);
  });

  // Verify no additional requests were made
  httpMock.expectNone('https://api.example.com/users');
});

it('should refresh cache when forceRefresh is true', () => {
  const mockUsers: User[] = [
    { id: 1, name: 'John', email: 'john@example.com', role: 'admin' }
  ];

  const updatedUsers: User[] = [
    { id: 1, name: 'John Updated', email: 'john@example.com', role: 'admin' }
  ];

  // First call
  service.getUsers().subscribe();
  httpMock.expectOne('https://api.example.com/users').flush(mockUsers);

  // Force refresh
  service.getUsers(true).subscribe(users => {
    expect(users).toEqual(updatedUsers);
  });

  const req = httpMock.expectOne('https://api.example.com/users');
  req.flush(updatedUsers);
});
```

### Testing File Upload

```typescript
// Service with file upload
@Injectable({
  providedIn: 'root'
})
export class FileUploadService {
  private apiUrl = 'https://api.example.com/upload';

  constructor(private http: HttpClient) {}

  uploadFile(file: File): Observable<{ url: string }> {
    const formData = new FormData();
    formData.append('file', file);

    return this.http.post<{ url: string }>(this.apiUrl, formData);
  }
}

// Test
it('should upload file with FormData', () => {
  const mockFile = new File(['content'], 'test.txt', { type: 'text/plain' });
  const mockResponse = { url: 'https://cdn.example.com/test.txt' };

  service.uploadFile(mockFile).subscribe(response => {
    expect(response).toEqual(mockResponse);
  });

  const req = httpMock.expectOne('https://api.example.com/upload');
  expect(req.request.method).toBe('POST');
  expect(req.request.body instanceof FormData).toBe(true);
  
  req.flush(mockResponse);
});
```

### Testing Request Interceptors

```typescript
// Interceptor
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  constructor(private authService: AuthService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = this.authService.getToken();
    
    if (token) {
      req = req.clone({
        setHeaders: { Authorization: `Bearer ${token}` }
      });
    }

    return next.handle(req);
  }
}

// Test
describe('AuthInterceptor', () => {
  let service: UserService;
  let httpMock: HttpTestingController;
  let authService: jasmine.SpyObj<AuthService>;

  beforeEach(() => {
    const authSpy = jasmine.createSpyObj('AuthService', ['getToken']);

    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [
        UserService,
        { provide: AuthService, useValue: authSpy },
        { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true }
      ]
    });

    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpTestingController);
    authService = TestBed.inject(AuthService) as jasmine.SpyObj<AuthService>;
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('should add Authorization header when token exists', () => {
    authService.getToken.and.returnValue('test-token');

    service.getUsers().subscribe();

    const req = httpMock.expectOne('https://api.example.com/users');
    expect(req.request.headers.get('Authorization')).toBe('Bearer test-token');
    
    req.flush([]);
  });

  it('should not add Authorization header when no token', () => {
    authService.getToken.and.returnValue(null);

    service.getUsers().subscribe();

    const req = httpMock.expectOne('https://api.example.com/users');
    expect(req.request.headers.has('Authorization')).toBe(false);
    
    req.flush([]);
  });
});
```

## Common Mistakes

### 1. Forgetting httpMock.verify()

```typescript
// BAD: Missing verify in afterEach
afterEach(() => {
  // Nothing here - unmatched requests won't be caught
});

// GOOD: Always verify
afterEach(() => {
  httpMock.verify(); // Catches unmatched requests
});
```

### 2. Not Handling Async Expectations

```typescript
// BAD: Expectations might not run
it('should fetch users', () => {
  service.getUsers().subscribe(users => {
    expect(users.length).toBe(2); // Might not execute
  });
  
  const req = httpMock.expectOne(apiUrl);
  req.flush(mockUsers);
  // Test completes before subscription callback
});

// GOOD: Use done callback or fakeAsync
it('should fetch users', (done) => {
  service.getUsers().subscribe(users => {
    expect(users.length).toBe(2);
    done(); // Ensures expectations run
  });
  
  const req = httpMock.expectOne(apiUrl);
  req.flush(mockUsers);
});
```

### 3. Incorrect Error Testing

```typescript
// BAD: Test passes even if no error occurs
it('should handle error', () => {
  service.getUsers().subscribe(
    () => {}, // Empty success handler
    (error) => {
      expect(error.status).toBe(404);
    }
  );
  
  const req = httpMock.expectOne(apiUrl);
  req.flush([], { status: 200, statusText: 'OK' }); // Success, not error!
});

// GOOD: Use fail() in success handler
it('should handle error', () => {
  service.getUsers().subscribe(
    () => fail('should have failed with 404'),
    (error) => {
      expect(error.status).toBe(404);
    }
  );
  
  const req = httpMock.expectOne(apiUrl);
  req.flush('Not Found', { status: 404, statusText: 'Not Found' });
});
```

### 4. Testing with Real HttpClient

```typescript
// BAD: Using real HttpClient
TestBed.configureTestingModule({
  imports: [HttpClientModule], // Real HTTP calls!
  providers: [UserService]
});

// GOOD: Use HttpClientTestingModule
TestBed.configureTestingModule({
  imports: [HttpClientTestingModule], // Mocked HTTP
  providers: [UserService]
});
```

### 5. Not Matching Request Predicates Correctly

```typescript
// BAD: Too generic matching
const req = httpMock.expectOne(req => req.url.includes('users'));
// This might match unintended requests

// GOOD: Specific matching
const req = httpMock.expectOne(req => 
  req.url === 'https://api.example.com/users' &&
  req.method === 'GET' &&
  req.params.get('role') === 'admin'
);
```

## Best Practices

### 1. Use Descriptive Test Names

```typescript
// GOOD: Clear and descriptive
it('should return empty array when no users exist', () => {});
it('should include authorization header with valid token', () => {});
it('should retry request 3 times on network failure', () => {});
```

### 2. Test Both Success and Failure Paths

```typescript
describe('getUserById', () => {
  it('should return user when found', () => {
    // Test success case
  });

  it('should return 404 error when user not found', () => {
    // Test error case
  });

  it('should handle network timeout', () => {
    // Test network failure
  });
});
```

### 3. Verify Request Details

```typescript
it('should send correct request', () => {
  service.createUser(newUser).subscribe();

  const req = httpMock.expectOne(apiUrl);
  
  // Verify method
  expect(req.request.method).toBe('POST');
  
  // Verify body
  expect(req.request.body).toEqual(newUser);
  
  // Verify headers
  expect(req.request.headers.get('Content-Type')).toBe('application/json');
  
  req.flush(mockResponse);
});
```

### 4. Use Test Fixtures

```typescript
// test-fixtures.ts
export const TEST_USERS: User[] = [
  { id: 1, name: 'John Doe', email: 'john@example.com', role: 'admin' },
  { id: 2, name: 'Jane Smith', email: 'jane@example.com', role: 'user' }
];

// user.service.spec.ts
import { TEST_USERS } from './test-fixtures';

it('should fetch users', () => {
  service.getUsers().subscribe(users => {
    expect(users).toEqual(TEST_USERS);
  });
  
  const req = httpMock.expectOne(apiUrl);
  req.flush(TEST_USERS);
});
```

### 5. Test Edge Cases

```typescript
it('should handle empty response array', () => {
  service.getUsers().subscribe(users => {
    expect(users).toEqual([]);
    expect(users.length).toBe(0);
  });
  
  const req = httpMock.expectOne(apiUrl);
  req.flush([]);
});

it('should handle malformed response data', () => {
  service.getUsers().subscribe(
    () => fail('should have failed'),
    (error) => {
      expect(error).toBeDefined();
    }
  );
  
  const req = httpMock.expectOne(apiUrl);
  req.flush('invalid json', { status: 200, statusText: 'OK' });
});
```

## Interview Questions

### 1. What is HttpTestingController and why is it used?

**Answer:** `HttpTestingController` is a testing utility from `@angular/common/http/testing` that allows you to mock and assert HTTP requests in Angular tests. It intercepts HTTP requests made by `HttpClient`, allowing you to:

- Mock HTTP responses without making real network calls
- Verify that specific requests were made with correct URLs, methods, headers, and bodies
- Simulate error conditions and edge cases
- Test request/response transformations
- Ensure no unexpected requests are made using `verify()`

It's essential for unit testing services that depend on HTTP communication.

### 2. Explain the difference between expectOne() and match() methods.

**Answer:**

**expectOne()**: Expects exactly one matching request. Throws an error if zero or multiple requests match. Returns a single `TestRequest`.

```typescript
const req = httpMock.expectOne('https://api.example.com/users');
```

**match()**: Returns an array of all matching requests. Doesn't throw an error if zero or multiple requests match.

```typescript
const reqs = httpMock.match('https://api.example.com/users');
expect(reqs.length).toBe(2);
```

Use `expectOne()` for single requests and `match()` when you expect multiple requests or need to count requests.

### 3. How do you test HTTP error scenarios?

**Answer:** Use `flush()` with error status codes or `error()` for network errors:

```typescript
// HTTP error (e.g., 404)
it('should handle 404 error', () => {
  service.getUser(999).subscribe(
    () => fail('should have failed'),
    (error) => {
      expect(error.status).toBe(404);
    }
  );
  
  const req = httpMock.expectOne(`${apiUrl}/999`);
  req.flush('Not found', { status: 404, statusText: 'Not Found' });
});

// Network error
it('should handle network error', () => {
  service.getUsers().subscribe(
    () => fail('should have failed'),
    (error) => {
      expect(error.error.type).toBe('error');
    }
  );
  
  const req = httpMock.expectOne(apiUrl);
  req.error(new ProgressEvent('error'));
});
```

### 4. What does httpMock.verify() do and why is it important?

**Answer:** `httpMock.verify()` ensures that all expected HTTP requests have been made and responded to, and that no unexpected requests remain pending. It should be called in `afterEach()` to:

- Detect unhandled requests that could cause test interference
- Ensure tests properly mock all HTTP calls
- Catch bugs where services make unexpected API calls
- Prevent false positives from incomplete test setup

```typescript
afterEach(() => {
  httpMock.verify(); // Throws if unmatched requests exist
});
```

### 5. How do you test HTTP interceptors?

**Answer:** Configure the interceptor in TestBed providers with `multi: true`, then verify that requests are modified correctly:

```typescript
TestBed.configureTestingModule({
  imports: [HttpClientTestingModule],
  providers: [
    UserService,
    { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true }
  ]
});

it('should add auth header via interceptor', () => {
  service.getUsers().subscribe();
  
  const req = httpMock.expectOne(apiUrl);
  expect(req.request.headers.get('Authorization')).toBe('Bearer token');
  
  req.flush([]);
});
```

Test both request modification (outgoing) and response handling (incoming) if the interceptor transforms responses.

### 6. How do you test services with retry logic?

**Answer:** Make multiple requests and respond with errors to trigger retries:

```typescript
it('should retry 3 times on failure', () => {
  let attemptCount = 0;
  
  service.getUsersWithRetry().subscribe(
    () => fail('should have failed'),
    () => {
      expect(attemptCount).toBe(4); // 1 initial + 3 retries
    }
  );
  
  // Respond to all 4 attempts
  for (let i = 0; i < 4; i++) {
    const req = httpMock.expectOne(apiUrl);
    attemptCount++;
    req.flush('Error', { status: 500, statusText: 'Error' });
  }
});
```

### 7. How do you test request parameters and headers?

**Answer:** Access `request.params` and `request.headers` on the `TestRequest`:

```typescript
it('should send correct params and headers', () => {
  service.searchUsers('john', 'admin-token').subscribe();
  
  const req = httpMock.expectOne(request => {
    return request.url === apiUrl &&
           request.params.get('q') === 'john' &&
           request.headers.get('Authorization') === 'Bearer admin-token';
  });
  
  expect(req.request.params.get('q')).toBe('john');
  expect(req.request.headers.get('Authorization')).toBe('Bearer admin-token');
  
  req.flush([]);
});
```

### 8. How do you test concurrent HTTP requests?

**Answer:** Make multiple service calls, use `match()` to get all pending requests, then respond to each:

```typescript
it('should handle concurrent requests', () => {
  service.getUser(1).subscribe(user1 => expect(user1.id).toBe(1));
  service.getUser(2).subscribe(user2 => expect(user2.id).toBe(2));
  
  const reqs = httpMock.match(req => req.url.includes('/users/'));
  expect(reqs.length).toBe(2);
  
  reqs[0].flush({ id: 1, name: 'User 1' });
  reqs[1].flush({ id: 2, name: 'User 2' });
});
```

### 9. What are common mistakes when testing HTTP services?

**Answer:**
- Forgetting `httpMock.verify()` in `afterEach()`
- Not using `fail()` in success handlers when testing errors
- Using real `HttpClientModule` instead of `HttpClientTestingModule`
- Not handling async expectations properly
- Testing implementation details instead of behavior
- Not testing error scenarios
- Overly generic request matching that catches unintended requests

### 10. How do you test caching in HTTP services?

**Answer:** Make multiple calls and verify only one HTTP request is made:

```typescript
it('should cache and not make duplicate requests', () => {
  const mockData = [{ id: 1, name: 'User' }];
  
  // First call - should hit API
  service.getCachedUsers().subscribe(data => {
    expect(data).toEqual(mockData);
  });
  
  const req = httpMock.expectOne(apiUrl);
  req.flush(mockData);
  
  // Second call - should use cache
  service.getCachedUsers().subscribe(data => {
    expect(data).toEqual(mockData);
  });
  
  // Verify no additional request was made
  httpMock.expectNone(apiUrl);
});
```

## Key Takeaways

1. **Always use HttpClientTestingModule** instead of HttpClientModule to mock HTTP requests in tests
2. **Call httpMock.verify()** in afterEach() to ensure all expected requests are handled and no unexpected requests remain
3. **Use expectOne()** for single requests and match() for multiple or variable numbers of requests
4. **Test both success and error paths** including network errors, HTTP errors, and edge cases
5. **Verify request details** including method, URL, headers, body, and query parameters
6. **Use flush() with error status** for HTTP errors and error() for network failures
7. **Test interceptors** by configuring them in TestBed and verifying request/response modifications
8. **Handle retry logic** by responding to multiple sequential requests with errors
9. **Test caching mechanisms** by verifying subsequent calls don't trigger new HTTP requests
10. **Create test fixtures** for reusable mock data to keep tests DRY and maintainable

## Resources

### Official Documentation
- [Angular Testing Guide - HTTP](https://angular.dev/guide/testing/components-scenarios#testing-http-services)
- [HttpClientTestingModule API](https://angular.io/api/common/http/testing/HttpClientTestingModule)
- [HttpTestingController API](https://angular.io/api/common/http/testing/HttpTestingController)

### Articles and Tutorials
- [Testing Angular HTTP Services](https://www.digitalocean.com/community/tutorials/angular-testing-httpclient)
- [Angular HTTP Testing Guide](https://medium.com/@bencabanes/angular-http-testing-3dd8de9e4ff9)

### Tools
- [Jasmine](https://jasmine.github.io/) - Testing framework
- [Karma](https://karma-runner.github.io/) - Test runner
- [Jest](https://jestjs.io/) - Alternative testing framework

### Video Tutorials
- [Angular Testing Crash Course](https://www.youtube.com/watch?v=BumgayeUC08)
- [Testing HTTP Services in Angular](https://www.youtube.com/watch?v=x2mhYmJIkJ8)

### Books
- "Testing Angular Applications" by Corinna Cohn et al.
- "Angular Development with TypeScript" by Yakov Fain and Anton Moiseev
