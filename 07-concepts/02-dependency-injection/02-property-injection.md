# Property Injection

## The Idea

**In plain English:** Property injection is a way to give an object the tools it needs (called dependencies) after the object has already been created, by setting them directly on the object like filling in blanks on a form. Unlike handing everything over at the start, you can add or swap out these tools at any point during the object's life.

**Real-world analogy:** Imagine you buy a basic smartphone out of the box. The phone works on its own, but you can slot in optional accessories after purchase — a SIM card, a microSD card, or a phone case — and you can even swap those accessories out later without buying a new phone.

- The smartphone = the object (class instance) created without all accessories upfront
- The accessories (SIM card, microSD card) = the dependencies set via properties after creation
- Swapping the SIM card for a different carrier = changing a dependency at runtime

---

## Overview

Property injection (also called setter injection) is a dependency injection pattern where dependencies are provided through public properties or setter methods after object construction. Unlike constructor injection, property injection allows for optional dependencies and runtime dependency changes, but sacrifices some compile-time safety and immutability guarantees.

## Core Concepts

### What is Property Injection?

Property injection sets dependencies after object creation:

```typescript
// Constructor injection (for comparison)
class UserService {
  constructor(private api: ApiClient) {}
}
const service = new UserService(apiClient);

// Property injection
class UserService {
  api!: ApiClient; // Set after construction
}
const service = new UserService();
service.api = apiClient; // Inject via property
```

### Constructor vs Property Injection

```
Constructor Injection:
┌─────────────┐
│ new Service(│
│   dep1,     │ ← Dependencies required at construction
│   dep2      │
│ )           │
└─────────────┘

Property Injection:
┌─────────────┐
│ new Service()│ ← Object created first
└──────┬──────┘
       │
       ▼
┌──────────────┐
│ service.dep1 │ ← Dependencies set after
│ service.dep2 │
└──────────────┘
```

### When to Use Property Injection

1. **Optional Dependencies**: Features that aren't required
2. **Runtime Configuration**: Dependencies that change during execution
3. **Circular Dependencies**: Breaking circular reference chains
4. **Framework Integration**: Some frameworks prefer property injection
5. **Lazy Initialization**: Defer dependency creation until needed

## React Examples

### Basic Property Injection

```typescript
// services/UserService.ts
interface ILogger {
  log(message: string): void;
  error(message: string): void;
}

class UserService {
  private _api!: ApiClient;
  private _cache!: ICache;
  logger?: ILogger; // Optional dependency
  
  // Required dependencies via setters
  set api(value: ApiClient) {
    if (!value) throw new Error('ApiClient is required');
    this._api = value;
  }
  
  get api(): ApiClient {
    if (!this._api) throw new Error('ApiClient not initialized');
    return this._api;
  }
  
  set cache(value: ICache) {
    if (!value) throw new Error('Cache is required');
    this._cache = value;
  }
  
  get cache(): ICache {
    if (!this._cache) throw new Error('Cache not initialized');
    return this._cache;
  }
  
  async getUser(id: string): Promise<User> {
    this.logger?.log(`Fetching user ${id}`);
    
    const cached = this.cache.get<User>(`user:${id}`);
    if (cached) {
      this.logger?.log(`Cache hit for user ${id}`);
      return cached;
    }
    
    const user = await this.api.get<User>(`/users/${id}`);
    this.cache.set(`user:${id}`, user);
    return user;
  }
}

// Usage
const userService = new UserService();
userService.api = new ApiClient();
userService.cache = new InMemoryCache();
userService.logger = new ConsoleLogger(); // Optional
```

### Property Injection with React Context

```typescript
// contexts/ServiceContext.tsx
import { createContext, useContext, ReactNode, useMemo } from 'react';

interface ServiceConfig {
  apiUrl: string;
  cacheEnabled: boolean;
  loggingEnabled: boolean;
}

class ConfigurableUserService {
  api?: ApiClient;
  cache?: ICache;
  logger?: ILogger;
  
  private config: ServiceConfig;
  
  constructor(config: ServiceConfig) {
    this.config = config;
  }
  
  // Lazy initialization via property getter
  private get apiClient(): ApiClient {
    if (!this.api) {
      this.api = new ApiClient(this.config.apiUrl);
    }
    return this.api;
  }
  
  async getUser(id: string): Promise<User> {
    this.logger?.log(`Fetching user ${id}`);
    
    if (this.config.cacheEnabled && this.cache) {
      const cached = this.cache.get<User>(`user:${id}`);
      if (cached) return cached;
    }
    
    const user = await this.apiClient.get<User>(`/users/${id}`);
    
    if (this.config.cacheEnabled && this.cache) {
      this.cache.set(`user:${id}`, user);
    }
    
    return user;
  }
  
  // Allow runtime dependency changes
  setLogger(logger: ILogger) {
    this.logger = logger;
  }
  
  setCache(cache: ICache) {
    this.cache = cache;
  }
}

const ServiceContext = createContext<ConfigurableUserService | null>(null);

export function ServiceProvider({ 
  children,
  config 
}: { 
  children: ReactNode;
  config: ServiceConfig;
}) {
  const service = useMemo(() => {
    const svc = new ConfigurableUserService(config);
    
    // Inject optional dependencies
    if (config.cacheEnabled) {
      svc.cache = new InMemoryCache();
    }
    
    if (config.loggingEnabled) {
      svc.logger = new ConsoleLogger();
    }
    
    return svc;
  }, [config]);
  
  return (
    <ServiceContext.Provider value={service}>
      {children}
    </ServiceContext.Provider>
  );
}

export function useUserService() {
  const context = useContext(ServiceContext);
  if (!context) {
    throw new Error('useUserService must be used within ServiceProvider');
  }
  return context;
}
```

### Runtime Dependency Changes

```typescript
// components/DebugPanel.tsx
function DebugPanel() {
  const userService = useUserService();
  const [loggingEnabled, setLoggingEnabled] = useState(false);
  
  useEffect(() => {
    if (loggingEnabled) {
      // Enable logging at runtime
      userService.setLogger(new VerboseLogger());
    } else {
      // Disable logging
      userService.setLogger(undefined);
    }
  }, [loggingEnabled, userService]);
  
  return (
    <div className="debug-panel">
      <label>
        <input
          type="checkbox"
          checked={loggingEnabled}
          onChange={(e) => setLoggingEnabled(e.target.checked)}
        />
        Enable Verbose Logging
      </label>
    </div>
  );
}
```

### Plugin System with Property Injection

```typescript
// services/PluginService.ts
interface IPlugin {
  name: string;
  initialize(): void;
  execute(data: any): any;
}

class AnalyticsPlugin implements IPlugin {
  name = 'analytics';
  
  initialize() {
    console.log('Analytics plugin initialized');
  }
  
  execute(data: any) {
    console.log('Track event:', data);
  }
}

class LoggingPlugin implements IPlugin {
  name = 'logging';
  
  initialize() {
    console.log('Logging plugin initialized');
  }
  
  execute(data: any) {
    console.log('Log:', data);
  }
}

class PluginManager {
  private plugins: Map<string, IPlugin> = new Map();
  
  // Add plugin via property-like method
  addPlugin(plugin: IPlugin) {
    plugin.initialize();
    this.plugins.set(plugin.name, plugin);
  }
  
  removePlugin(name: string) {
    this.plugins.delete(name);
  }
  
  executePlugins(data: any) {
    this.plugins.forEach(plugin => {
      try {
        plugin.execute(data);
      } catch (error) {
        console.error(`Plugin ${plugin.name} failed:`, error);
      }
    });
  }
  
  hasPlugin(name: string): boolean {
    return this.plugins.has(name);
  }
}

// Usage in React
function App() {
  const pluginManager = useMemo(() => new PluginManager(), []);
  
  useEffect(() => {
    // Add plugins at runtime
    pluginManager.addPlugin(new AnalyticsPlugin());
    pluginManager.addPlugin(new LoggingPlugin());
    
    return () => {
      // Cleanup plugins
      pluginManager.removePlugin('analytics');
      pluginManager.removePlugin('logging');
    };
  }, [pluginManager]);
  
  const handleAction = () => {
    pluginManager.executePlugins({ action: 'button_click' });
  };
  
  return <button onClick={handleAction}>Click Me</button>;
}
```

### Fluent Interface with Property Injection

```typescript
// services/QueryBuilder.ts
class QueryBuilder {
  private filters: Array<{ field: string; value: any }> = [];
  private sortField?: string;
  private sortDirection?: 'asc' | 'desc';
  private limitValue?: number;
  
  // Fluent setters
  filter(field: string, value: any): this {
    this.filters.push({ field, value });
    return this;
  }
  
  sort(field: string, direction: 'asc' | 'desc' = 'asc'): this {
    this.sortField = field;
    this.sortDirection = direction;
    return this;
  }
  
  limit(value: number): this {
    this.limitValue = value;
    return this;
  }
  
  build(): string {
    let query = 'SELECT * FROM users';
    
    if (this.filters.length > 0) {
      const conditions = this.filters
        .map(f => `${f.field} = '${f.value}'`)
        .join(' AND ');
      query += ` WHERE ${conditions}`;
    }
    
    if (this.sortField) {
      query += ` ORDER BY ${this.sortField} ${this.sortDirection}`;
    }
    
    if (this.limitValue) {
      query += ` LIMIT ${this.limitValue}`;
    }
    
    return query;
  }
}

// Usage
function UserList() {
  const [users, setUsers] = useState<User[]>([]);
  
  useEffect(() => {
    const query = new QueryBuilder()
      .filter('status', 'active')
      .filter('role', 'admin')
      .sort('created_at', 'desc')
      .limit(10)
      .build();
    
    console.log(query);
    // SELECT * FROM users WHERE status = 'active' AND role = 'admin' 
    // ORDER BY created_at desc LIMIT 10
  }, []);
  
  return <div>{/* Render users */}</div>;
}
```

## Angular Examples

### Basic Property Injection

```typescript
// services/configurable-api.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Injectable({
  providedIn: 'root'
})
export class ConfigurableApiService {
  private _baseUrl: string = 'https://api.example.com';
  private _timeout: number = 5000;
  logger?: ILogger; // Optional dependency
  
  // Property setters
  set baseUrl(url: string) {
    this._baseUrl = url;
  }
  
  get baseUrl(): string {
    return this._baseUrl;
  }
  
  set timeout(ms: number) {
    this._timeout = ms;
  }
  
  get timeout(): number {
    return this._timeout;
  }
  
  constructor(private http: HttpClient) {}
  
  get<T>(endpoint: string): Observable<T> {
    this.logger?.log(`GET ${this._baseUrl}${endpoint}`);
    
    return this.http.get<T>(`${this._baseUrl}${endpoint}`, {
      // Use timeout property
      // Note: Actual timeout implementation would use RxJS timeout operator
    });
  }
}
```

### @Input() as Property Injection

```typescript
// components/user-card/user-card.component.ts
import { Component, Input, OnChanges, SimpleChanges } from '@angular/core';

// Services injected as properties via @Input()
@Component({
  selector: 'app-user-card',
  template: `
    <div class="user-card">
      <h3>{{ user?.name }}</h3>
      <button (click)="refresh()">Refresh</button>
    </div>
  `
})
export class UserCardComponent implements OnChanges {
  @Input() userId!: string;
  @Input() userService!: UserService; // Property injection via @Input
  @Input() logger?: ILogger; // Optional
  
  user: User | null = null;
  
  ngOnChanges(changes: SimpleChanges) {
    if (changes['userId'] || changes['userService']) {
      this.loadUser();
    }
  }
  
  private loadUser() {
    if (!this.userService) {
      console.error('UserService not provided');
      return;
    }
    
    this.logger?.log(`Loading user ${this.userId}`);
    
    this.userService.getUser(this.userId).subscribe({
      next: (user) => {
        this.user = user;
        this.logger?.log(`Loaded user: ${user.name}`);
      },
      error: (error) => {
        this.logger?.error('Failed to load user', error);
      }
    });
  }
  
  refresh() {
    this.loadUser();
  }
}

// Usage in parent component
@Component({
  template: `
    <app-user-card
      [userId]="userId"
      [userService]="userService"
      [logger]="logger">
    </app-user-card>
  `
})
export class ParentComponent {
  userId = '123';
  
  constructor(
    public userService: UserService,
    public logger: LoggerService
  ) {}
}
```

### Runtime Configuration with Property Injection

```typescript
// services/feature-flag.service.ts
@Injectable({
  providedIn: 'root'
})
export class FeatureFlagService {
  private flags = new Map<string, boolean>();
  
  setFlag(name: string, enabled: boolean) {
    this.flags.set(name, enabled);
  }
  
  isEnabled(name: string): boolean {
    return this.flags.get(name) ?? false;
  }
}

// services/user.service.ts
@Injectable({
  providedIn: 'root'
})
export class UserService {
  // Optional dependency
  private _featureFlags?: FeatureFlagService;
  
  set featureFlags(service: FeatureFlagService) {
    this._featureFlags = service;
  }
  
  constructor(
    private http: HttpClient,
    private cache: CacheService
  ) {}
  
  getUser(id: string): Observable<User> {
    // Use new API if feature flag is enabled
    if (this._featureFlags?.isEnabled('use-new-api')) {
      return this.http.get<User>(`/v2/users/${id}`);
    }
    
    // Fall back to old API
    return this.http.get<User>(`/v1/users/${id}`);
  }
}

// app.component.ts
@Component({
  selector: 'app-root',
  template: '<router-outlet></router-outlet>'
})
export class AppComponent implements OnInit {
  constructor(
    private userService: UserService,
    private featureFlags: FeatureFlagService
  ) {}
  
  ngOnInit() {
    // Configure feature flags at runtime
    this.featureFlags.setFlag('use-new-api', true);
    
    // Inject into service
    this.userService.featureFlags = this.featureFlags;
  }
}
```

### Setter Injection with Validation

```typescript
// services/validated-config.service.ts
@Injectable({
  providedIn: 'root'
})
export class ValidatedConfigService {
  private _apiKey: string = '';
  private _environment: 'dev' | 'prod' = 'dev';
  
  set apiKey(key: string) {
    if (!key || key.length < 32) {
      throw new Error('API key must be at least 32 characters');
    }
    this._apiKey = key;
  }
  
  get apiKey(): string {
    if (!this._apiKey) {
      throw new Error('API key not configured');
    }
    return this._apiKey;
  }
  
  set environment(env: 'dev' | 'prod') {
    if (env !== 'dev' && env !== 'prod') {
      throw new Error('Environment must be dev or prod');
    }
    this._environment = env;
  }
  
  get environment(): 'dev' | 'prod' {
    return this._environment;
  }
  
  get apiUrl(): string {
    return this._environment === 'prod'
      ? 'https://api.example.com'
      : 'https://api-dev.example.com';
  }
}
```

### Method Injection Pattern

```typescript
// services/notification.service.ts
@Injectable({
  providedIn: 'root'
})
export class NotificationService {
  private transports: INotificationTransport[] = [];
  
  // Method injection - add transports dynamically
  addTransport(transport: INotificationTransport) {
    this.transports.push(transport);
  }
  
  removeTransport(transport: INotificationTransport) {
    const index = this.transports.indexOf(transport);
    if (index > -1) {
      this.transports.splice(index, 1);
    }
  }
  
  async send(message: string) {
    const results = await Promise.allSettled(
      this.transports.map(t => t.send(message))
    );
    
    results.forEach((result, index) => {
      if (result.status === 'rejected') {
        console.error(
          `Transport ${this.transports[index].name} failed:`,
          result.reason
        );
      }
    });
  }
}

// Transports
interface INotificationTransport {
  name: string;
  send(message: string): Promise<void>;
}

class EmailTransport implements INotificationTransport {
  name = 'email';
  async send(message: string) {
    console.log('Sending email:', message);
  }
}

class SMSTransport implements INotificationTransport {
  name = 'sms';
  async send(message: string) {
    console.log('Sending SMS:', message);
  }
}

class PushTransport implements INotificationTransport {
  name = 'push';
  async send(message: string) {
    console.log('Sending push:', message);
  }
}

// Usage
@Component({
  selector: 'app-notification-settings',
  template: `
    <div>
      <label>
        <input type="checkbox" [(ngModel)]="emailEnabled" 
               (change)="updateTransports()">
        Email
      </label>
      <label>
        <input type="checkbox" [(ngModel)]="smsEnabled"
               (change)="updateTransports()">
        SMS
      </label>
      <label>
        <input type="checkbox" [(ngModel)]="pushEnabled"
               (change)="updateTransports()">
        Push
      </label>
    </div>
  `
})
export class NotificationSettingsComponent {
  emailEnabled = true;
  smsEnabled = false;
  pushEnabled = true;
  
  private emailTransport = new EmailTransport();
  private smsTransport = new SMSTransport();
  private pushTransport = new PushTransport();
  
  constructor(private notificationService: NotificationService) {
    this.updateTransports();
  }
  
  updateTransports() {
    // Clear all transports
    this.notificationService.removeTransport(this.emailTransport);
    this.notificationService.removeTransport(this.smsTransport);
    this.notificationService.removeTransport(this.pushTransport);
    
    // Add enabled transports
    if (this.emailEnabled) {
      this.notificationService.addTransport(this.emailTransport);
    }
    if (this.smsEnabled) {
      this.notificationService.addTransport(this.smsTransport);
    }
    if (this.pushEnabled) {
      this.notificationService.addTransport(this.pushTransport);
    }
  }
}
```

## Testing with Property Injection

### React Testing

```typescript
// __tests__/ConfigurableUserService.test.ts
describe('ConfigurableUserService', () => {
  let service: ConfigurableUserService;
  let mockApi: jest.Mocked<ApiClient>;
  let mockCache: jest.Mocked<ICache>;
  let mockLogger: jest.Mocked<ILogger>;
  
  beforeEach(() => {
    const config = {
      apiUrl: 'https://test.com',
      cacheEnabled: true,
      loggingEnabled: true
    };
    
    service = new ConfigurableUserService(config);
    
    // Inject mock dependencies via properties
    mockApi = {
      get: jest.fn(),
      post: jest.fn()
    } as any;
    
    mockCache = {
      get: jest.fn(),
      set: jest.fn()
    } as any;
    
    mockLogger = {
      log: jest.fn(),
      error: jest.fn()
    } as any;
    
    service.api = mockApi;
    service.cache = mockCache;
    service.logger = mockLogger;
  });
  
  it('should use injected logger', async () => {
    mockCache.get.mockReturnValue(null);
    mockApi.get.mockResolvedValue({ id: '1', name: 'John' });
    
    await service.getUser('1');
    
    expect(mockLogger.log).toHaveBeenCalledWith('Fetching user 1');
  });
  
  it('should work without optional logger', async () => {
    service.logger = undefined; // Remove optional dependency
    
    mockCache.get.mockReturnValue(null);
    mockApi.get.mockResolvedValue({ id: '1', name: 'John' });
    
    // Should not throw
    await expect(service.getUser('1')).resolves.toBeTruthy();
  });
  
  it('should allow runtime logger changes', async () => {
    const newLogger = {
      log: jest.fn(),
      error: jest.fn()
    };
    
    service.setLogger(newLogger);
    
    mockCache.get.mockReturnValue(null);
    mockApi.get.mockResolvedValue({ id: '1', name: 'John' });
    
    await service.getUser('1');
    
    expect(newLogger.log).toHaveBeenCalled();
    expect(mockLogger.log).not.toHaveBeenCalled();
  });
});
```

### Angular Testing

```typescript
// user-card.component.spec.ts
describe('UserCardComponent', () => {
  let component: UserCardComponent;
  let fixture: ComponentFixture<UserCardComponent>;
  let mockUserService: jasmine.SpyObj<UserService>;
  let mockLogger: jasmine.SpyObj<ILogger>;
  
  beforeEach(() => {
    mockUserService = jasmine.createSpyObj('UserService', ['getUser']);
    mockLogger = jasmine.createSpyObj('ILogger', ['log', 'error']);
    
    TestBed.configureTestingModule({
      declarations: [UserCardComponent]
    });
    
    fixture = TestBed.createComponent(UserCardComponent);
    component = fixture.componentInstance;
    
    // Inject via properties
    component.userId = '123';
    component.userService = mockUserService;
    component.logger = mockLogger;
  });
  
  it('should load user on init', () => {
    const mockUser = { id: '123', name: 'John' };
    mockUserService.getUser.and.returnValue(of(mockUser));
    
    component.ngOnChanges({
      userId: new SimpleChange(null, '123', true)
    });
    
    expect(mockUserService.getUser).toHaveBeenCalledWith('123');
    expect(component.user).toEqual(mockUser);
  });
  
  it('should work without optional logger', () => {
    component.logger = undefined; // Remove optional dependency
    
    const mockUser = { id: '123', name: 'John' };
    mockUserService.getUser.and.returnValue(of(mockUser));
    
    // Should not throw
    expect(() => {
      component.ngOnChanges({
        userId: new SimpleChange(null, '123', true)
      });
    }).not.toThrow();
  });
});
```

## Common Mistakes

### 1. Using Property Injection for Required Dependencies

```typescript
// Bad - required dependency via property
class UserService {
  api!: ApiClient; // May be undefined!
  
  async getUser(id: string) {
    // Runtime error if api not set
    return this.api.get(`/users/${id}`);
  }
}

// Good - required dependency via constructor
class UserService {
  constructor(private api: ApiClient) {}
  
  async getUser(id: string) {
    // Guaranteed to have api
    return this.api.get(`/users/${id}`);
  }
}
```

### 2. No Validation in Setters

```typescript
// Bad - no validation
class ConfigService {
  set apiKey(key: string) {
    this._apiKey = key;
  }
}

// Good - validate in setter
class ConfigService {
  private _apiKey: string = '';
  
  set apiKey(key: string) {
    if (!key || key.length < 32) {
      throw new Error('Invalid API key');
    }
    this._apiKey = key;
  }
}
```

### 3. Public Mutable Properties

```typescript
// Bad - public mutable property
class UserService {
  api: ApiClient = new ApiClient(); // Can be changed anywhere
}

// Good - private with getter/setter
class UserService {
  private _api: ApiClient = new ApiClient();
  
  set api(value: ApiClient) {
    if (!value) throw new Error('ApiClient required');
    this._api = value;
  }
  
  get api(): ApiClient {
    return this._api;
  }
}
```

### 4. Not Checking for Undefined

```typescript
// Bad - assuming optional dependency exists
class Service {
  logger?: ILogger;
  
  doSomething() {
    this.logger.log('Doing something'); // May crash!
  }
}

// Good - optional chaining
class Service {
  logger?: ILogger;
  
  doSomething() {
    this.logger?.log('Doing something'); // Safe
  }
}
```

## Best Practices

### 1. Use Constructor Injection for Required Dependencies

```typescript
// Required via constructor, optional via property
class UserService {
  logger?: ILogger; // Optional
  
  constructor(
    private readonly api: ApiClient, // Required
    private readonly cache: ICache   // Required
  ) {}
}
```

### 2. Validate in Setters

```typescript
class ConfigService {
  private _timeout: number = 5000;
  
  set timeout(ms: number) {
    if (ms < 0 || ms > 60000) {
      throw new Error('Timeout must be between 0 and 60000ms');
    }
    this._timeout = ms;
  }
}
```

### 3. Provide Defaults for Optional Dependencies

```typescript
class UserService {
  private _logger: ILogger = new NoOpLogger(); // Default
  
  set logger(logger: ILogger) {
    this._logger = logger;
  }
  
  async getUser(id: string) {
    this._logger.log(`Fetching user ${id}`); // Always safe
    // ...
  }
}

class NoOpLogger implements ILogger {
  log() {}
  error() {}
}
```

### 4. Document Property Injection

```typescript
/**
 * UserService manages user operations
 * 
 * @example
 * const service = new UserService(api, cache);
 * service.logger = new ConsoleLogger(); // Optional
 */
class UserService {
  /** Optional logger for debugging */
  logger?: ILogger;
  
  constructor(
    private api: ApiClient,
    private cache: ICache
  ) {}
}
```

## When to Use Property Injection

### Use When:
- Dependency is truly optional
- Need to change dependencies at runtime
- Implementing plugin systems
- Breaking circular dependencies
- Framework requires it (e.g., Angular @Input)

### Don't Use When:
- Dependency is required for functionality
- Want compile-time safety
- Object should be immutable after creation
- Testing doesn't require runtime changes

## Interview Questions

### Q1: What's the difference between constructor and property injection?
**Answer**: Constructor injection provides dependencies when creating an object, making them required and immutable. Property injection sets dependencies after construction via properties/setters, allowing optional dependencies and runtime changes but losing compile-time safety.

### Q2: When would you choose property injection over constructor injection?
**Answer**: Use property injection for: 1) Optional dependencies that aren't required, 2) Dependencies that need to change at runtime, 3) Plugin systems with dynamic components, 4) Breaking circular dependencies, 5) Framework requirements like Angular @Input.

### Q3: How do you ensure property-injected dependencies are valid?
**Answer**: 1) Validate in setter methods, 2) Use optional chaining when accessing, 3) Provide default implementations, 4) Check for undefined before use, 5) Document which properties are required vs optional.

### Q4: What are the risks of property injection?
**Answer**: Objects may be used in partially initialized states, no compile-time dependency checking, dependencies can be changed unexpectedly, harder to track what dependencies exist, and can lead to runtime errors if required properties aren't set.

## Key Takeaways

1. Property injection sets dependencies after object construction
2. Best for optional dependencies and runtime configuration
3. Loses compile-time safety compared to constructor injection
4. Allows changing dependencies during object lifetime
5. Use getters/setters for validation and encapsulation
6. Always validate property values in setters
7. Use optional chaining for truly optional dependencies
8. Prefer constructor injection for required dependencies
9. Good for plugin systems and feature flags
10. Document which properties are required vs optional

## Resources

- Martin Fowler's DI Patterns: https://martinfowler.com/articles/injection.html
- Angular Property Injection: https://angular.io/guide/dependency-injection
- TypeScript Property Decorators
- Setter Injection Pattern
- IoC Container Patterns
