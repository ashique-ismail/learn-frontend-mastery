# Constructor Injection

## The Idea

**In plain English:** Constructor injection is a way of giving a piece of code (called a "class") everything it needs to do its job by handing those things in at the moment it is created, instead of letting the code go find them on its own. A "class" is just a blueprint for building a reusable piece of a program.

**Real-world analogy:** When a chef is hired at a restaurant, the restaurant manager hands them a knife kit, a recipe book, and the keys to the pantry on their very first day — the chef never has to scavenge for those tools themselves.

- The chef = the class (a piece of code with a job to do)
- The knife kit, recipe book, and pantry keys = the dependencies (the tools the class needs)
- The manager handing everything over on day one = the constructor (the moment of creation where all tools are provided)

---

## Overview

Constructor injection is a dependency injection pattern where dependencies are provided through a class constructor. It's the most explicit and recommended form of dependency injection because it makes dependencies visible, enforces required dependencies at compile time, and promotes immutability.

## Core Concepts

### What is Constructor Injection?

Constructor injection passes dependencies as constructor parameters:

```typescript
// Bad - tight coupling
class UserService {
  private api = new ApiClient(); // Hard-coded dependency
  
  async getUser(id: string) {
    return this.api.get(`/users/${id}`);
  }
}

// Good - constructor injection
class UserService {
  constructor(private api: ApiClient) {} // Dependency injected
  
  async getUser(id: string) {
    return this.api.get(`/users/${id}`);
  }
}

// Usage
const apiClient = new ApiClient();
const userService = new UserService(apiClient);
```

### Benefits of Constructor Injection

1. **Explicit Dependencies**: All dependencies are visible in the constructor signature
2. **Compile-Time Safety**: Missing dependencies cause compilation errors
3. **Immutability**: Dependencies can be marked as readonly/final
4. **Testability**: Easy to inject mock dependencies
5. **Single Responsibility**: Forces you to think about dependencies

## React Examples

### Basic Constructor Injection Pattern

```typescript
// services/UserService.ts
interface IApiClient {
  get<T>(url: string): Promise<T>;
  post<T>(url: string, data: any): Promise<T>;
}

interface ICache {
  get<T>(key: string): T | null;
  set<T>(key: string, value: T): void;
}

class UserService {
  constructor(
    private readonly api: IApiClient,
    private readonly cache: ICache
  ) {}
  
  async getUser(id: string) {
    // Try cache first
    const cached = this.cache.get<User>(`user:${id}`);
    if (cached) return cached;
    
    // Fetch from API
    const user = await this.api.get<User>(`/users/${id}`);
    this.cache.set(`user:${id}`, user);
    return user;
  }
  
  async updateUser(id: string, data: Partial<User>) {
    const user = await this.api.post<User>(`/users/${id}`, data);
    this.cache.set(`user:${id}`, user); // Update cache
    return user;
  }
}
```

### React Context for Constructor Injection

```typescript
// contexts/ServiceContext.tsx
import { createContext, useContext, ReactNode } from 'react';

class ApiClient implements IApiClient {
  async get<T>(url: string): Promise<T> {
    const response = await fetch(url);
    return response.json();
  }
  
  async post<T>(url: string, data: any): Promise<T> {
    const response = await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    });
    return response.json();
  }
}

class InMemoryCache implements ICache {
  private storage = new Map<string, any>();
  
  get<T>(key: string): T | null {
    return this.storage.get(key) ?? null;
  }
  
  set<T>(key: string, value: T): void {
    this.storage.set(key, value);
  }
}

interface Services {
  userService: UserService;
  apiClient: IApiClient;
  cache: ICache;
}

const ServiceContext = createContext<Services | null>(null);

export function ServiceProvider({ children }: { children: ReactNode }) {
  // Create dependencies
  const apiClient = useMemo(() => new ApiClient(), []);
  const cache = useMemo(() => new InMemoryCache(), []);
  
  // Inject dependencies via constructor
  const userService = useMemo(
    () => new UserService(apiClient, cache),
    [apiClient, cache]
  );
  
  const services = useMemo(
    () => ({ userService, apiClient, cache }),
    [userService, apiClient, cache]
  );
  
  return (
    <ServiceContext.Provider value={services}>
      {children}
    </ServiceContext.Provider>
  );
}

export function useServices() {
  const context = useContext(ServiceContext);
  if (!context) {
    throw new Error('useServices must be used within ServiceProvider');
  }
  return context;
}
```

### Using Injected Services in Components

```typescript
// components/UserProfile.tsx
function UserProfile({ userId }: { userId: string }) {
  const { userService } = useServices();
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    let cancelled = false;
    
    async function loadUser() {
      try {
        setLoading(true);
        const data = await userService.getUser(userId);
        if (!cancelled) {
          setUser(data);
        }
      } catch (error) {
        console.error('Failed to load user:', error);
      } finally {
        if (!cancelled) {
          setLoading(false);
        }
      }
    }
    
    loadUser();
    
    return () => {
      cancelled = true;
    };
  }, [userId, userService]);
  
  const handleUpdate = async (data: Partial<User>) => {
    try {
      const updated = await userService.updateUser(userId, data);
      setUser(updated);
    } catch (error) {
      console.error('Failed to update user:', error);
    }
  };
  
  if (loading) return <div>Loading...</div>;
  if (!user) return <div>User not found</div>;
  
  return (
    <div>
      <h1>{user.name}</h1>
      <button onClick={() => handleUpdate({ name: 'New Name' })}>
        Update Name
      </button>
    </div>
  );
}
```

### Multiple Dependencies

```typescript
// services/NotificationService.ts
interface ILogger {
  log(message: string): void;
  error(message: string, error?: Error): void;
}

interface IAnalytics {
  track(event: string, properties?: Record<string, any>): void;
}

class NotificationService {
  constructor(
    private readonly api: IApiClient,
    private readonly logger: ILogger,
    private readonly analytics: IAnalytics
  ) {
    this.logger.log('NotificationService initialized');
  }
  
  async sendNotification(userId: string, message: string) {
    try {
      this.logger.log(`Sending notification to user ${userId}`);
      
      const result = await this.api.post('/notifications', {
        userId,
        message,
        timestamp: Date.now()
      });
      
      this.analytics.track('notification_sent', {
        userId,
        messageLength: message.length
      });
      
      return result;
    } catch (error) {
      this.logger.error('Failed to send notification', error as Error);
      throw error;
    }
  }
}

// Implementation classes
class ConsoleLogger implements ILogger {
  log(message: string): void {
    console.log(`[LOG] ${message}`);
  }
  
  error(message: string, error?: Error): void {
    console.error(`[ERROR] ${message}`, error);
  }
}

class GoogleAnalytics implements IAnalytics {
  track(event: string, properties?: Record<string, any>): void {
    // @ts-ignore
    if (window.gtag) {
      // @ts-ignore
      window.gtag('event', event, properties);
    }
  }
}
```

### Factory Pattern with Constructor Injection

```typescript
// factories/ServiceFactory.ts
class ServiceFactory {
  private static apiClient: IApiClient;
  private static cache: ICache;
  private static logger: ILogger;
  private static analytics: IAnalytics;
  
  static initialize() {
    this.apiClient = new ApiClient();
    this.cache = new InMemoryCache();
    this.logger = new ConsoleLogger();
    this.analytics = new GoogleAnalytics();
  }
  
  static createUserService(): UserService {
    return new UserService(this.apiClient, this.cache);
  }
  
  static createNotificationService(): NotificationService {
    return new NotificationService(
      this.apiClient,
      this.logger,
      this.analytics
    );
  }
}

// Usage in React
function App() {
  const services = useMemo(() => {
    ServiceFactory.initialize();
    return {
      userService: ServiceFactory.createUserService(),
      notificationService: ServiceFactory.createNotificationService()
    };
  }, []);
  
  return (
    <ServiceContext.Provider value={services}>
      <Router />
    </ServiceContext.Provider>
  );
}
```

## Angular Examples

### Basic Constructor Injection

```typescript
// services/api-client.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class ApiClientService {
  constructor(private http: HttpClient) {}
  
  get<T>(url: string): Observable<T> {
    return this.http.get<T>(url);
  }
  
  post<T>(url: string, data: any): Observable<T> {
    return this.http.post<T>(url, data);
  }
}

// services/cache.service.ts
@Injectable({
  providedIn: 'root'
})
export class CacheService {
  private storage = new Map<string, any>();
  
  get<T>(key: string): T | null {
    return this.storage.get(key) ?? null;
  }
  
  set<T>(key: string, value: T): void {
    this.storage.set(key, value);
  }
  
  clear(): void {
    this.storage.clear();
  }
}

// services/user.service.ts
@Injectable({
  providedIn: 'root'
})
export class UserService {
  // Constructor injection - Angular automatically provides dependencies
  constructor(
    private apiClient: ApiClientService,
    private cache: CacheService
  ) {}
  
  getUser(id: string): Observable<User> {
    const cached = this.cache.get<User>(`user:${id}`);
    if (cached) {
      return of(cached);
    }
    
    return this.apiClient.get<User>(`/users/${id}`).pipe(
      tap(user => this.cache.set(`user:${id}`, user))
    );
  }
  
  updateUser(id: string, data: Partial<User>): Observable<User> {
    return this.apiClient.post<User>(`/users/${id}`, data).pipe(
      tap(user => this.cache.set(`user:${id}`, user))
    );
  }
}
```

### Multiple Dependencies in Angular

```typescript
// services/logger.service.ts
@Injectable({
  providedIn: 'root'
})
export class LoggerService {
  log(message: string): void {
    console.log(`[LOG] ${new Date().toISOString()} - ${message}`);
  }
  
  error(message: string, error?: Error): void {
    console.error(`[ERROR] ${new Date().toISOString()} - ${message}`, error);
  }
  
  warn(message: string): void {
    console.warn(`[WARN] ${new Date().toISOString()} - ${message}`);
  }
}

// services/analytics.service.ts
@Injectable({
  providedIn: 'root'
})
export class AnalyticsService {
  track(event: string, properties?: Record<string, any>): void {
    // Send to analytics service
    console.log('Track event:', event, properties);
  }
}

// services/notification.service.ts
@Injectable({
  providedIn: 'root'
})
export class NotificationService {
  // Multiple dependencies injected via constructor
  constructor(
    private apiClient: ApiClientService,
    private logger: LoggerService,
    private analytics: AnalyticsService
  ) {
    this.logger.log('NotificationService initialized');
  }
  
  sendNotification(userId: string, message: string): Observable<any> {
    this.logger.log(`Sending notification to user ${userId}`);
    
    return this.apiClient.post('/notifications', {
      userId,
      message,
      timestamp: Date.now()
    }).pipe(
      tap(() => {
        this.analytics.track('notification_sent', {
          userId,
          messageLength: message.length
        });
      }),
      catchError(error => {
        this.logger.error('Failed to send notification', error);
        return throwError(() => error);
      })
    );
  }
}
```

### Using Services in Components

```typescript
// components/user-profile/user-profile.component.ts
import { Component, Input, OnInit } from '@angular/core';

@Component({
  selector: 'app-user-profile',
  template: `
    <div *ngIf="loading">Loading...</div>
    <div *ngIf="error">{{ error }}</div>
    <div *ngIf="user">
      <h1>{{ user.name }}</h1>
      <p>{{ user.email }}</p>
      <button (click)="updateName()">Update Name</button>
    </div>
  `
})
export class UserProfileComponent implements OnInit {
  @Input() userId!: string;
  user: User | null = null;
  loading = true;
  error: string | null = null;
  
  // Dependencies injected via constructor
  constructor(
    private userService: UserService,
    private notificationService: NotificationService,
    private logger: LoggerService
  ) {}
  
  ngOnInit() {
    this.loadUser();
  }
  
  private loadUser() {
    this.loading = true;
    this.error = null;
    
    this.userService.getUser(this.userId).subscribe({
      next: (user) => {
        this.user = user;
        this.loading = false;
        this.logger.log(`Loaded user: ${user.name}`);
      },
      error: (error) => {
        this.error = 'Failed to load user';
        this.loading = false;
        this.logger.error('Failed to load user', error);
      }
    });
  }
  
  updateName() {
    if (!this.user) return;
    
    this.userService.updateUser(this.userId, { name: 'New Name' })
      .subscribe({
        next: (user) => {
          this.user = user;
          this.notificationService.sendNotification(
            this.userId,
            'Your name has been updated'
          ).subscribe();
        },
        error: (error) => {
          this.logger.error('Failed to update user', error);
        }
      });
  }
}
```

### Interface-Based Injection with InjectionToken

```typescript
// interfaces/storage.interface.ts
export interface IStorage {
  getItem(key: string): string | null;
  setItem(key: string, value: string): void;
  removeItem(key: string): void;
  clear(): void;
}

// Create injection token
export const STORAGE_TOKEN = new InjectionToken<IStorage>('storage');

// implementations/local-storage.service.ts
@Injectable()
export class LocalStorageService implements IStorage {
  getItem(key: string): string | null {
    return localStorage.getItem(key);
  }
  
  setItem(key: string, value: string): void {
    localStorage.setItem(key, value);
  }
  
  removeItem(key: string): void {
    localStorage.removeItem(key);
  }
  
  clear(): void {
    localStorage.clear();
  }
}

// implementations/session-storage.service.ts
@Injectable()
export class SessionStorageService implements IStorage {
  getItem(key: string): string | null {
    return sessionStorage.getItem(key);
  }
  
  setItem(key: string, value: string): void {
    sessionStorage.setItem(key, value);
  }
  
  removeItem(key: string): void {
    sessionStorage.removeItem(key);
  }
  
  clear(): void {
    sessionStorage.clear();
  }
}

// app.module.ts
@NgModule({
  providers: [
    // Provide LocalStorage implementation
    { provide: STORAGE_TOKEN, useClass: LocalStorageService }
    // Or switch to SessionStorage
    // { provide: STORAGE_TOKEN, useClass: SessionStorageService }
  ]
})
export class AppModule {}

// services/auth.service.ts
@Injectable({
  providedIn: 'root'
})
export class AuthService {
  // Inject interface via token
  constructor(@Inject(STORAGE_TOKEN) private storage: IStorage) {}
  
  saveToken(token: string): void {
    this.storage.setItem('auth_token', token);
  }
  
  getToken(): string | null {
    return this.storage.getItem('auth_token');
  }
  
  clearToken(): void {
    this.storage.removeItem('auth_token');
  }
}
```

## Testing with Constructor Injection

### React Testing

```typescript
// __tests__/UserService.test.ts
class MockApiClient implements IApiClient {
  private responses = new Map<string, any>();
  
  mockResponse(url: string, data: any) {
    this.responses.set(url, data);
  }
  
  async get<T>(url: string): Promise<T> {
    const response = this.responses.get(url);
    if (!response) throw new Error(`No mock for ${url}`);
    return response;
  }
  
  async post<T>(url: string, data: any): Promise<T> {
    return data as T;
  }
}

class MockCache implements ICache {
  private storage = new Map<string, any>();
  
  get<T>(key: string): T | null {
    return this.storage.get(key) ?? null;
  }
  
  set<T>(key: string, value: T): void {
    this.storage.set(key, value);
  }
  
  getSpy() {
    return jest.spyOn(this, 'get');
  }
  
  setSpy() {
    return jest.spyOn(this, 'set');
  }
}

describe('UserService', () => {
  let apiClient: MockApiClient;
  let cache: MockCache;
  let userService: UserService;
  
  beforeEach(() => {
    // Inject mock dependencies
    apiClient = new MockApiClient();
    cache = new MockCache();
    userService = new UserService(apiClient, cache);
  });
  
  it('should fetch user from API and cache it', async () => {
    const mockUser = { id: '1', name: 'John' };
    apiClient.mockResponse('/users/1', mockUser);
    
    const getSpy = cache.getSpy();
    const setSpy = cache.setSpy();
    
    const user = await userService.getUser('1');
    
    expect(user).toEqual(mockUser);
    expect(getSpy).toHaveBeenCalledWith('user:1');
    expect(setSpy).toHaveBeenCalledWith('user:1', mockUser);
  });
  
  it('should return cached user if available', async () => {
    const mockUser = { id: '1', name: 'John' };
    cache.set('user:1', mockUser);
    
    const user = await userService.getUser('1');
    
    expect(user).toEqual(mockUser);
    // API should not be called
  });
});
```

### Angular Testing

```typescript
// user.service.spec.ts
describe('UserService', () => {
  let service: UserService;
  let httpMock: HttpTestingController;
  let cacheService: CacheService;
  
  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [
        UserService,
        CacheService
      ]
    });
    
    // Angular automatically injects dependencies
    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpTestingController);
    cacheService = TestBed.inject(CacheService);
  });
  
  afterEach(() => {
    httpMock.verify();
  });
  
  it('should fetch user from API', (done) => {
    const mockUser = { id: '1', name: 'John' };
    
    service.getUser('1').subscribe(user => {
      expect(user).toEqual(mockUser);
      done();
    });
    
    const req = httpMock.expectOne('/users/1');
    expect(req.request.method).toBe('GET');
    req.flush(mockUser);
  });
  
  it('should use cached user', (done) => {
    const mockUser = { id: '1', name: 'John' };
    spyOn(cacheService, 'get').and.returnValue(mockUser);
    
    service.getUser('1').subscribe(user => {
      expect(user).toEqual(mockUser);
      done();
    });
    
    // No HTTP request should be made
    httpMock.expectNone('/users/1');
  });
});

// Testing with mock services
describe('UserProfileComponent', () => {
  let component: UserProfileComponent;
  let fixture: ComponentFixture<UserProfileComponent>;
  let mockUserService: jasmine.SpyObj<UserService>;
  
  beforeEach(() => {
    // Create mock service
    mockUserService = jasmine.createSpyObj('UserService', ['getUser', 'updateUser']);
    
    TestBed.configureTestingModule({
      declarations: [UserProfileComponent],
      providers: [
        // Inject mock service
        { provide: UserService, useValue: mockUserService }
      ]
    });
    
    fixture = TestBed.createComponent(UserProfileComponent);
    component = fixture.componentInstance;
  });
  
  it('should load user on init', () => {
    const mockUser = { id: '1', name: 'John' };
    mockUserService.getUser.and.returnValue(of(mockUser));
    
    component.userId = '1';
    component.ngOnInit();
    
    expect(mockUserService.getUser).toHaveBeenCalledWith('1');
    expect(component.user).toEqual(mockUser);
  });
});
```

## Advanced Patterns

### Optional Dependencies

```typescript
// React
class AnalyticsService {
  constructor(
    private readonly tracker?: IAnalytics // Optional dependency
  ) {}
  
  trackEvent(event: string, properties?: Record<string, any>) {
    if (this.tracker) {
      this.tracker.track(event, properties);
    } else {
      console.log('Analytics not configured, event:', event);
    }
  }
}

// Angular
@Injectable({
  providedIn: 'root'
})
export class AnalyticsService {
  constructor(
    @Optional() private tracker?: IAnalytics
  ) {}
  
  trackEvent(event: string, properties?: Record<string, any>) {
    if (this.tracker) {
      this.tracker.track(event, properties);
    }
  }
}
```

### Dependency Graph Visualization

```
Constructor Injection Dependency Graph:

┌─────────────────────┐
│   AppComponent      │
└──────────┬──────────┘
           │
           ├──────────────────┬──────────────────┐
           │                  │                  │
    ┌──────▼──────┐    ┌──────▼──────┐   ┌──────▼──────┐
    │ UserService │    │AuthService  │   │CartService  │
    └──────┬──────┘    └──────┬──────┘   └──────┬──────┘
           │                  │                  │
           ├────────┬─────────┼──────────────────┘
           │        │         │
      ┌────▼───┐ ┌──▼────┐ ┌─▼────────┐
      │ApiClient│ │Cache  │ │Storage   │
      └─────────┘ └───────┘ └──────────┘

All dependencies flow downward through constructors
```

### Circular Dependency Prevention

```typescript
// Bad - circular dependency
class ServiceA {
  constructor(private serviceB: ServiceB) {}
}

class ServiceB {
  constructor(private serviceA: ServiceA) {} // Circular!
}

// Good - introduce interface or mediator
interface IServiceBOperations {
  doSomething(): void;
}

class ServiceA {
  private serviceB?: IServiceBOperations;
  
  setServiceB(serviceB: IServiceBOperations) {
    this.serviceB = serviceB;
  }
}

class ServiceB implements IServiceBOperations {
  constructor(private serviceA: ServiceA) {
    // Break circular dependency
    serviceA.setServiceB(this);
  }
  
  doSomething() {
    // Implementation
  }
}

// Better - use event emitter or mediator
class EventBus {
  private listeners = new Map<string, Function[]>();
  
  on(event: string, handler: Function) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, []);
    }
    this.listeners.get(event)!.push(handler);
  }
  
  emit(event: string, data?: any) {
    const handlers = this.listeners.get(event) || [];
    handlers.forEach(handler => handler(data));
  }
}

class ServiceA {
  constructor(private eventBus: EventBus) {
    this.eventBus.on('serviceb:action', this.handleAction);
  }
  
  handleAction = (data: any) => {
    // Handle event from ServiceB
  };
}

class ServiceB {
  constructor(private eventBus: EventBus) {}
  
  doAction() {
    this.eventBus.emit('serviceb:action', { /* data */ });
  }
}
```

## Common Mistakes

### 1. Hidden Dependencies

```typescript
// Bad - hidden dependency on global
class UserService {
  constructor(private api: ApiClient) {}
  
  async getUser(id: string) {
    // Hidden dependency on localStorage
    const cached = localStorage.getItem(`user:${id}`);
    if (cached) return JSON.parse(cached);
    
    return this.api.get(`/users/${id}`);
  }
}

// Good - explicit dependency
class UserService {
  constructor(
    private api: ApiClient,
    private storage: IStorage // Explicit
  ) {}
  
  async getUser(id: string) {
    const cached = this.storage.getItem(`user:${id}`);
    if (cached) return JSON.parse(cached);
    
    return this.api.get(`/users/${id}`);
  }
}
```

### 2. Too Many Dependencies

```typescript
// Bad - too many dependencies (violation of SRP)
class UserService {
  constructor(
    private api: ApiClient,
    private cache: ICache,
    private logger: ILogger,
    private analytics: IAnalytics,
    private storage: IStorage,
    private validator: IValidator,
    private formatter: IFormatter
  ) {}
}

// Good - split into smaller services
class UserService {
  constructor(
    private api: ApiClient,
    private cache: ICache
  ) {}
}

class UserAnalytics {
  constructor(
    private analytics: IAnalytics,
    private logger: ILogger
  ) {}
}
```

### 3. Mutating Dependencies

```typescript
// Bad - modifying injected dependency
class UserService {
  constructor(private config: AppConfig) {
    this.config.apiUrl = 'https://new-url.com'; // Mutation!
  }
}

// Good - treat as immutable
class UserService {
  constructor(private readonly config: AppConfig) {
    // Use config, don't modify
    const url = this.config.apiUrl;
  }
}
```

## Best Practices

### 1. Depend on Abstractions

```typescript
// Bad - depend on concrete class
class UserService {
  constructor(private api: ApiClient) {}
}

// Good - depend on interface
interface IApiClient {
  get<T>(url: string): Promise<T>;
}

class UserService {
  constructor(private api: IApiClient) {}
}
```

### 2. Use Readonly Modifiers

```typescript
class UserService {
  constructor(
    private readonly api: IApiClient,
    private readonly cache: ICache
  ) {}
  // Dependencies cannot be reassigned
}
```

### 3. Validate Dependencies

```typescript
class UserService {
  constructor(
    private readonly api: IApiClient,
    private readonly cache: ICache
  ) {
    if (!api) throw new Error('ApiClient is required');
    if (!cache) throw new Error('Cache is required');
  }
}
```

### 4. Document Dependencies

```typescript
/**
 * UserService manages user data operations
 * 
 * @param api - HTTP client for API requests
 * @param cache - Cache for storing user data
 * @param logger - Optional logger for debugging
 */
class UserService {
  constructor(
    private readonly api: IApiClient,
    private readonly cache: ICache,
    private readonly logger?: ILogger
  ) {}
}
```

## When to Use Constructor Injection

### Use When:
- Dependencies are required for the class to function
- You want compile-time dependency checking
- You want to enforce immutability
- Testing with mock dependencies is important
- Dependencies won't change during object lifetime

### Don't Use When:
- Dependencies are truly optional
- You need to change dependencies at runtime
- Working with legacy code that uses other patterns
- Creating simple data classes with no behavior

## Interview Questions

### Q1: What are the advantages of constructor injection over property injection?
**Answer**: Constructor injection makes dependencies explicit and required, provides compile-time safety, allows marking dependencies as readonly/immutable, and ensures the object is fully initialized when constructed. Property injection can lead to objects being used in partially initialized states.

### Q2: How do you handle circular dependencies with constructor injection?
**Answer**: Circular dependencies indicate a design problem. Solutions include: 1) Introducing an interface/abstraction, 2) Using a mediator/event bus, 3) Setter injection for one dependency, 4) Refactoring to remove the circular dependency by extracting shared logic.

### Q3: What's the difference between constructor injection in React and Angular?
**Answer**: Angular has a built-in DI container that automatically resolves and injects dependencies based on constructor parameters. React doesn't have built-in DI, so you typically use Context API, higher-order components, or external DI libraries. Angular's approach is more explicit and framework-integrated.

### Q4: How do you test code that uses constructor injection?
**Answer**: Create mock implementations of the dependencies and inject them through the constructor. This is straightforward because all dependencies are explicit. You can use testing frameworks' spy/mock utilities to create test doubles.

## Key Takeaways

1. Constructor injection makes dependencies explicit and required
2. It provides compile-time safety and enables immutability
3. All dependencies are visible in the constructor signature
4. It's the most testable form of dependency injection
5. Angular has built-in constructor injection via DI container
6. React requires Context API or DI libraries for constructor injection
7. Depend on abstractions (interfaces) not concrete implementations
8. Too many constructor parameters indicate SRP violation
9. Prevents objects from being used in partially initialized state
10. Enables easy swapping of implementations for testing

## Resources

- Martin Fowler's Dependency Injection: https://martinfowler.com/articles/injection.html
- Angular Dependency Injection: https://angular.io/guide/dependency-injection
- Inversion of Control Containers and DI Pattern
- SOLID Principles - Dependency Inversion Principle
- TypeScript Handbook - Classes and Interfaces
