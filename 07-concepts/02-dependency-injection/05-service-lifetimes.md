# Service Lifetimes

## The Idea

**In plain English:** A service lifetime controls how many copies of a helper tool your app creates and how long each copy sticks around. Think of it as deciding whether everyone in town shares one copy of a book, each person gets their own copy, or each reading group gets one copy to share among themselves.

**Real-world analogy:** Imagine a town library system managing copies of a popular reference book:
- The single shared reference copy at the front desk (never leaves) = Singleton — one instance used by the whole application
- Photocopies handed out and thrown away after each use = Transient — a fresh instance created every time one is needed
- One copy loaned to each book club for the duration of their meeting = Scoped — one instance shared within a defined boundary (like a request or component tree), then released

---

## Overview

Service lifetimes define how long a service instance lives and how many instances are created. Understanding lifetimes is crucial for managing resources, memory, state sharing, and preventing bugs. The three main lifetimes are: Singleton (one instance for entire application), Transient (new instance every time), and Scoped (one instance per scope/request).

## Core Concepts

### The Three Lifetimes

```
Singleton: One instance for entire application
┌─────────────────────────────────────┐
│         Application                 │
│  ┌────────────────────────────┐    │
│  │   Single Service Instance   │    │
│  └────────────────────────────┘    │
│         Used by all components      │
└─────────────────────────────────────┘

Transient: New instance every time
┌─────────────────────────────────────┐
│         Application                 │
│  ┌──────┐  ┌──────┐  ┌──────┐     │
│  │Inst 1│  │Inst 2│  │Inst 3│     │
│  └──────┘  └──────┘  └──────┘     │
│    Comp A    Comp B    Comp C      │
└─────────────────────────────────────┘

Scoped: One instance per scope (request/session)
┌─────────────────────────────────────┐
│         Application                 │
│  ┌──────────────┐  ┌──────────────┐│
│  │  Scope 1     │  │  Scope 2     ││
│  │  ┌────────┐  │  │  ┌────────┐ ││
│  │  │Instance│  │  │  │Instance│ ││
│  │  └────────┘  │  │  └────────┘ ││
│  └──────────────┘  └──────────────┘│
└─────────────────────────────────────┘
```

### Lifetime Characteristics

| Lifetime  | Instances | State | Memory | Use Case |
|-----------|-----------|-------|--------|----------|
| Singleton | 1 | Shared | Low | Config, cache, connection pools |
| Transient | Many | Isolated | High | Stateless operations, utilities |
| Scoped | Per scope | Shared in scope | Medium | Request context, transactions |

## React Examples

### Singleton Pattern

```typescript
// services/ApiClient.ts
class ApiClient {
  private static instance: ApiClient;
  private baseUrl: string;
  
  private constructor() {
    this.baseUrl = process.env.REACT_APP_API_URL || '';
    console.log('ApiClient instance created');
  }
  
  static getInstance(): ApiClient {
    if (!ApiClient.instance) {
      ApiClient.instance = new ApiClient();
    }
    return ApiClient.instance;
  }
  
  async get<T>(endpoint: string): Promise<T> {
    const response = await fetch(`${this.baseUrl}${endpoint}`);
    return response.json();
  }
}

// Usage - always returns same instance
const api1 = ApiClient.getInstance();
const api2 = ApiClient.getInstance();
console.log(api1 === api2); // true
```

### Singleton with React Context

```typescript
// contexts/SingletonContext.tsx
import { createContext, useContext, ReactNode, useMemo } from 'react';

// Singleton services
class ConfigService {
  private config = {
    apiUrl: process.env.REACT_APP_API_URL,
    apiKey: process.env.REACT_APP_API_KEY
  };
  
  get(key: string): any {
    return this.config[key];
  }
}

class CacheService {
  private cache = new Map<string, any>();
  
  get<T>(key: string): T | null {
    return this.cache.get(key) ?? null;
  }
  
  set<T>(key: string, value: T): void {
    this.cache.set(key, value);
  }
  
  clear(): void {
    this.cache.clear();
  }
}

class LoggerService {
  log(message: string): void {
    console.log(`[${new Date().toISOString()}] ${message}`);
  }
  
  error(message: string, error?: Error): void {
    console.error(`[${new Date().toISOString()}] ${message}`, error);
  }
}

// Create singleton instances OUTSIDE component
const singletonConfig = new ConfigService();
const singletonCache = new CacheService();
const singletonLogger = new LoggerService();

interface Singletons {
  config: ConfigService;
  cache: CacheService;
  logger: LoggerService;
}

const SingletonContext = createContext<Singletons | null>(null);

export function SingletonProvider({ children }: { children: ReactNode }) {
  // useMemo ensures these stay the same across re-renders
  const services = useMemo(() => ({
    config: singletonConfig,
    cache: singletonCache,
    logger: singletonLogger
  }), []); // Empty deps = created once
  
  return (
    <SingletonContext.Provider value={services}>
      {children}
    </SingletonContext.Provider>
  );
}

export function useSingletons() {
  const context = useContext(SingletonContext);
  if (!context) {
    throw new Error('useSingletons must be used within SingletonProvider');
  }
  return context;
}
```

### Transient Pattern

```typescript
// services/TransientServices.ts

// Transient service - new instance every time
class DataProcessor {
  private id = Math.random();
  
  constructor() {
    console.log(`DataProcessor ${this.id} created`);
  }
  
  process(data: any): any {
    return {
      ...data,
      processed: true,
      processedBy: this.id
    };
  }
}

// Factory for creating transient instances
class ServiceFactory {
  createDataProcessor(): DataProcessor {
    return new DataProcessor(); // New instance
  }
  
  createValidator(): DataValidator {
    return new DataValidator(); // New instance
  }
}

// React hook for transient services
function useTransientService<T>(factory: () => T): T {
  // Create new instance on every render
  return useMemo(() => factory(), []);
  
  // Or for truly transient (new instance every render):
  // return factory();
}

// Usage in component
function DataProcessingComponent() {
  // New processor instance for this component instance
  const processor = useTransientService(() => new DataProcessor());
  
  const handleProcess = (data: any) => {
    const result = processor.process(data);
    console.log('Processed:', result);
  };
  
  return <button onClick={() => handleProcess({ value: 42 })}>Process</button>;
}
```

### Scoped Pattern (Per-Component Scope)

```typescript
// contexts/ScopedContext.tsx

// Scoped service - one instance per scope
class UserSessionService {
  private sessionId: string;
  private state: Record<string, any> = {};
  
  constructor() {
    this.sessionId = Math.random().toString(36);
    console.log(`UserSession ${this.sessionId} created`);
  }
  
  getSessionId(): string {
    return this.sessionId;
  }
  
  setState(key: string, value: any): void {
    this.state[key] = value;
  }
  
  getState(key: string): any {
    return this.state[key];
  }
}

const ScopedContext = createContext<UserSessionService | null>(null);

// Each ScopedProvider creates its own instance
export function ScopedProvider({ children }: { children: ReactNode }) {
  // New instance for this scope
  const session = useMemo(() => new UserSessionService(), []);
  
  return (
    <ScopedContext.Provider value={session}>
      {children}
    </ScopedContext.Provider>
  );
}

export function useScopedSession() {
  const context = useContext(ScopedContext);
  if (!context) {
    throw new Error('useScopedSession must be used within ScopedProvider');
  }
  return context;
}

// Usage - different scopes get different instances
function App() {
  return (
    <div>
      <ScopedProvider>
        <Section name="Section 1" />
      </ScopedProvider>
      
      <ScopedProvider>
        <Section name="Section 2" />
      </ScopedProvider>
    </div>
  );
}

function Section({ name }: { name: string }) {
  const session = useScopedSession();
  
  return (
    <div>
      <h2>{name}</h2>
      <p>Session ID: {session.getSessionId()}</p>
      <ChildComponent />
    </div>
  );
}

function ChildComponent() {
  const session = useScopedSession(); // Same instance as parent in scope
  
  return <p>Child sees same session: {session.getSessionId()}</p>;
}
```

### Request-Scoped Pattern

```typescript
// contexts/RequestContext.tsx

// Request-scoped service
class RequestService {
  private requestId: string;
  private startTime: number;
  private metadata: Record<string, any> = {};
  
  constructor() {
    this.requestId = `req_${Date.now()}_${Math.random()}`;
    this.startTime = Date.now();
  }
  
  getRequestId(): string {
    return this.requestId;
  }
  
  getDuration(): number {
    return Date.now() - this.startTime;
  }
  
  setMetadata(key: string, value: any): void {
    this.metadata[key] = value;
  }
  
  getMetadata(): Record<string, any> {
    return { ...this.metadata };
  }
}

const RequestContext = createContext<RequestService | null>(null);

export function RequestProvider({ children }: { children: ReactNode }) {
  const request = useMemo(() => new RequestService(), []);
  
  useEffect(() => {
    return () => {
      console.log(`Request ${request.getRequestId()} completed in ${request.getDuration()}ms`);
    };
  }, [request]);
  
  return (
    <RequestContext.Provider value={request}>
      {children}
    </RequestContext.Provider>
  );
}

export function useRequest() {
  const context = useContext(RequestContext);
  if (!context) {
    throw new Error('useRequest must be used within RequestProvider');
  }
  return context;
}

// Usage - wrap each "request" in provider
function UserProfilePage({ userId }: { userId: string }) {
  return (
    <RequestProvider>
      <UserProfileContent userId={userId} />
    </RequestProvider>
  );
}

function UserProfileContent({ userId }: { userId: string }) {
  const request = useRequest();
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    request.setMetadata('userId', userId);
    request.setMetadata('action', 'loadUser');
    
    // Load user...
  }, [userId, request]);
  
  return (
    <div>
      <p>Request ID: {request.getRequestId()}</p>
      {/* Render user */}
    </div>
  );
}
```

### Comparing Lifetimes

```typescript
// services/LifetimeDemo.ts

// Singleton - shared state
class SingletonCounter {
  private static instance: SingletonCounter;
  private count = 0;
  
  private constructor() {}
  
  static getInstance() {
    if (!this.instance) {
      this.instance = new SingletonCounter();
    }
    return this.instance;
  }
  
  increment() {
    this.count++;
  }
  
  getCount() {
    return this.count;
  }
}

// Transient - isolated state
class TransientCounter {
  private count = 0;
  
  increment() {
    this.count++;
  }
  
  getCount() {
    return this.count;
  }
}

// Demo component
function LifetimeDemo() {
  const singleton = SingletonCounter.getInstance();
  const transient = useMemo(() => new TransientCounter(), []);
  
  const [, forceUpdate] = useReducer(x => x + 1, 0);
  
  return (
    <div>
      <h2>Singleton Counter</h2>
      <p>Count: {singleton.getCount()}</p>
      <button onClick={() => {
        singleton.increment();
        forceUpdate();
      }}>Increment Singleton</button>
      
      <h2>Transient Counter</h2>
      <p>Count: {transient.getCount()}</p>
      <button onClick={() => {
        transient.increment();
        forceUpdate();
      }}>Increment Transient</button>
      
      <hr />
      
      <OtherComponent />
    </div>
  );
}

function OtherComponent() {
  const singleton = SingletonCounter.getInstance(); // Same instance!
  const transient = useMemo(() => new TransientCounter(), []); // Different instance!
  
  return (
    <div>
      <p>Other component sees singleton count: {singleton.getCount()}</p>
      <p>Other component transient count: {transient.getCount()}</p>
    </div>
  );
}
```

## Angular Examples

### Singleton Services

```typescript
// services/singleton.service.ts

// providedIn: 'root' = Singleton
@Injectable({
  providedIn: 'root' // Single instance for entire app
})
export class ConfigService {
  private config = {
    apiUrl: environment.apiUrl,
    apiKey: environment.apiKey
  };
  
  constructor() {
    console.log('ConfigService instance created');
  }
  
  get<T>(key: string): T {
    return this.config[key];
  }
  
  set(key: string, value: any): void {
    this.config[key] = value;
  }
}

@Injectable({
  providedIn: 'root'
})
export class CacheService {
  private cache = new Map<string, any>();
  
  constructor() {
    console.log('CacheService instance created');
  }
  
  get<T>(key: string): T | null {
    return this.cache.get(key) ?? null;
  }
  
  set<T>(key: string, value: T): void {
    this.cache.set(key, value);
  }
}

// Usage - same instance everywhere
@Component({
  selector: 'app-component-a'
})
export class ComponentA {
  constructor(private config: ConfigService) {
    // ConfigService instance created once
  }
}

@Component({
  selector: 'app-component-b'
})
export class ComponentB {
  constructor(private config: ConfigService) {
    // Same ConfigService instance injected
  }
}
```

### Transient Services

```typescript
// services/transient.service.ts

// No providedIn = Transient (when provided in component)
@Injectable()
export class DataProcessor {
  private id = Math.random();
  
  constructor() {
    console.log(`DataProcessor ${this.id} created`);
  }
  
  process(data: any): any {
    return {
      ...data,
      processed: true,
      processedBy: this.id
    };
  }
}

// Provide in component = new instance per component
@Component({
  selector: 'app-data-display',
  providers: [DataProcessor] // New instance for each component
})
export class DataDisplayComponent {
  constructor(private processor: DataProcessor) {
    // New DataProcessor instance
  }
  
  processData(data: any) {
    return this.processor.process(data);
  }
}

// Each component gets its own instance
@Component({
  template: `
    <app-data-display></app-data-display>
    <app-data-display></app-data-display>
  `
})
export class ParentComponent {
  // Two DataDisplayComponents = two DataProcessor instances
}
```

### Scoped Services

```typescript
// services/scoped.service.ts

// Service provided at specific scope
@Injectable()
export class UserSessionService {
  private sessionId = Math.random().toString(36);
  private state = new Map<string, any>();
  
  constructor() {
    console.log(`UserSessionService ${this.sessionId} created`);
  }
  
  getSessionId(): string {
    return this.sessionId;
  }
  
  setState(key: string, value: any): void {
    this.state.set(key, value);
  }
  
  getState(key: string): any {
    return this.state.get(key);
  }
}

// Provided at module level = one instance per module
@NgModule({
  providers: [UserSessionService] // Scoped to this module
})
export class FeatureModule {}

// Provided at component level = one instance per component tree
@Component({
  selector: 'app-user-section',
  providers: [UserSessionService], // Scoped to this component and children
  template: `
    <div>
      <p>Session: {{ session.getSessionId() }}</p>
      <app-child></app-child>
    </div>
  `
})
export class UserSectionComponent {
  constructor(public session: UserSessionService) {}
}

@Component({
  selector: 'app-child'
})
export class ChildComponent {
  constructor(public session: UserSessionService) {
    // Same instance as parent UserSectionComponent
  }
}
```

### Request-Scoped with HTTP Interceptor

```typescript
// services/request-context.service.ts
@Injectable({
  providedIn: 'root'
})
export class RequestContextService {
  private contexts = new Map<string, RequestContext>();
  
  createContext(): string {
    const id = `req_${Date.now()}_${Math.random()}`;
    this.contexts.set(id, {
      id,
      startTime: Date.now(),
      metadata: {}
    });
    return id;
  }
  
  getContext(id: string): RequestContext | undefined {
    return this.contexts.get(id);
  }
  
  destroyContext(id: string): void {
    const context = this.contexts.get(id);
    if (context) {
      console.log(`Request ${id} took ${Date.now() - context.startTime}ms`);
      this.contexts.delete(id);
    }
  }
}

// interceptors/request-scope.interceptor.ts
@Injectable()
export class RequestScopeInterceptor implements HttpInterceptor {
  constructor(private requestContext: RequestContextService) {}
  
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const contextId = this.requestContext.createContext();
    
    // Add context ID to request headers
    const clonedReq = req.clone({
      setHeaders: {
        'X-Request-Context': contextId
      }
    });
    
    return next.handle(clonedReq).pipe(
      finalize(() => {
        this.requestContext.destroyContext(contextId);
      })
    );
  }
}
```

### Lifetime Configuration

```typescript
// app.module.ts
@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule],
  providers: [
    // Singleton - one instance for app
    { provide: ConfigService, useClass: ConfigService },
    
    // Transient - factory creates new instance each time
    { 
      provide: DataProcessor, 
      useFactory: () => new DataProcessor() 
    },
    
    // Scoped - different instance per injector
    { 
      provide: UserSessionService, 
      useClass: UserSessionService 
    }
  ]
})
export class AppModule {}

// feature.module.ts
@NgModule({
  providers: [
    // This module gets its own UserSessionService instance
    UserSessionService
  ]
})
export class FeatureModule {}
```

### Lifetime with InjectionToken

```typescript
// tokens/lifetime-tokens.ts
export const SINGLETON_TOKEN = new InjectionToken<CacheService>('singleton.cache', {
  providedIn: 'root',
  factory: () => new CacheService() // Created once
});

export const TRANSIENT_TOKEN = new InjectionToken<DataProcessor>('transient.processor');

// app.module.ts
@NgModule({
  providers: [
    {
      provide: TRANSIENT_TOKEN,
      useFactory: () => new DataProcessor(), // New instance each time
      deps: []
    }
  ]
})
export class AppModule {}

// component.ts
@Component({
  selector: 'app-example'
})
export class ExampleComponent {
  constructor(
    @Inject(SINGLETON_TOKEN) private cache: CacheService, // Same instance
    @Inject(TRANSIENT_TOKEN) private processor: DataProcessor // New instance
  ) {}
}
```

## Memory Implications

### Singleton Memory Profile

```typescript
// Single instance - low memory overhead
class ConfigService {
  private config = { /* configuration data */ };
  // Only one instance in memory
}

// Memory usage: O(1) - constant
// Good for: Shared resources, configuration, caches
```

### Transient Memory Profile

```typescript
// Many instances - high memory overhead
class DataProcessor {
  private buffer = new Array(1000); // Each instance has buffer
  
  process(data: any) {
    // Processing logic
  }
}

// If created 100 times: 100 * buffer size in memory
// Memory usage: O(n) - linear with instance count
// Good for: Stateless operations, temporary processing
```

### Scoped Memory Profile

```typescript
// One instance per scope - medium overhead
class UserSessionService {
  private state = new Map<string, any>();
  
  // Each scope (request/component tree) has one instance
}

// Memory usage: O(s) where s = number of active scopes
// Good for: Request context, isolated feature state
```

### Memory Leak Example

```typescript
// Bad - singleton holding references to transient objects
class BadSingletonService {
  private listeners: Array<EventListener> = [];
  
  addListener(listener: EventListener) {
    this.listeners.push(listener); // Memory leak!
  }
  
  // If listeners are from components that unmount,
  // they'll never be garbage collected
}

// Good - clean up references
class GoodSingletonService {
  private listeners = new Set<EventListener>();
  
  addListener(listener: EventListener) {
    this.listeners.add(listener);
  }
  
  removeListener(listener: EventListener) {
    this.listeners.delete(listener);
  }
}

// Component cleanup
function MyComponent() {
  const service = useSingleton();
  
  useEffect(() => {
    const listener = () => {};
    service.addListener(listener);
    
    return () => {
      service.removeListener(listener); // Clean up!
    };
  }, [service]);
}
```

## Testing Different Lifetimes

### Testing Singleton

```typescript
// singleton.service.spec.ts
describe('CacheService (Singleton)', () => {
  it('should return same instance', () => {
    const instance1 = CacheService.getInstance();
    const instance2 = CacheService.getInstance();
    
    expect(instance1).toBe(instance2);
  });
  
  it('should share state across instances', () => {
    const instance1 = CacheService.getInstance();
    const instance2 = CacheService.getInstance();
    
    instance1.set('key', 'value');
    expect(instance2.get('key')).toBe('value');
  });
});
```

### Testing Transient

```typescript
// transient.service.spec.ts
describe('DataProcessor (Transient)', () => {
  it('should create new instances', () => {
    const factory = new ServiceFactory();
    const instance1 = factory.createDataProcessor();
    const instance2 = factory.createDataProcessor();
    
    expect(instance1).not.toBe(instance2);
  });
  
  it('should have isolated state', () => {
    const factory = new ServiceFactory();
    const instance1 = factory.createDataProcessor();
    const instance2 = factory.createDataProcessor();
    
    const result1 = instance1.process({ value: 1 });
    const result2 = instance2.process({ value: 2 });
    
    expect(result1.processedBy).not.toBe(result2.processedBy);
  });
});
```

### Testing Scoped

```typescript
// scoped.service.spec.ts
describe('UserSessionService (Scoped)', () => {
  it('should create instance per scope', () => {
    TestBed.configureTestingModule({
      providers: [UserSessionService]
    });
    
    const instance1 = TestBed.inject(UserSessionService);
    
    // Create new testing module = new scope
    TestBed.resetTestingModule();
    TestBed.configureTestingModule({
      providers: [UserSessionService]
    });
    
    const instance2 = TestBed.inject(UserSessionService);
    
    expect(instance1).not.toBe(instance2);
  });
});
```

## Common Mistakes

### 1. Using Singleton for Stateful Request Data

```typescript
// Bad - singleton with request-specific state
@Injectable({
  providedIn: 'root' // Singleton!
})
export class BadRequestService {
  private currentUserId: string = ''; // Shared across all requests!
  
  setUserId(id: string) {
    this.currentUserId = id; // Race condition!
  }
}

// Good - scoped or pass as parameter
@Injectable()
export class GoodRequestService {
  processRequest(userId: string) {
    // User ID passed as parameter, not stored
  }
}
```

### 2. Memory Leaks with Singletons

```typescript
// Bad - singleton holding component references
class EventBus {
  private static instance: EventBus;
  private listeners = new Map<string, Function[]>();
  
  on(event: string, handler: Function) {
    // Handlers never removed = memory leak
    if (!this.listeners.has(event)) {
      this.listeners.set(event, []);
    }
    this.listeners.get(event)!.push(handler);
  }
}

// Good - provide cleanup method
class EventBus {
  private listeners = new Map<string, Set<Function>>();
  
  on(event: string, handler: Function) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(handler);
    
    // Return cleanup function
    return () => this.off(event, handler);
  }
  
  off(event: string, handler: Function) {
    this.listeners.get(event)?.delete(handler);
  }
}
```

### 3. Wrong Lifetime Choice

```typescript
// Bad - transient for expensive operation
@Component({
  providers: [ExpensiveDatabaseConnection] // New connection per component!
})
export class MyComponent {}

// Good - singleton for shared resources
@Injectable({
  providedIn: 'root' // One connection for app
})
export class DatabaseConnection {}
```

## Best Practices

### 1. Choose Appropriate Lifetime

```typescript
// Singleton: Shared resources, configuration
@Injectable({ providedIn: 'root' })
export class ConfigService {}

// Transient: Stateless utilities
@Injectable() // Provided in component
export class DataValidator {}

// Scoped: Request/session context
@Injectable() // Provided in module/component
export class RequestContext {}
```

### 2. Avoid State in Transient Services

```typescript
// Bad - state in transient service
class TransientCounter {
  private count = 0; // Lost when instance destroyed
  
  increment() {
    this.count++;
  }
}

// Good - stateless transient
class DataTransformer {
  transform(data: any): any {
    return { ...data, transformed: true };
  }
}
```

### 3. Clean Up Singleton Resources

```typescript
class WebSocketService {
  private connection?: WebSocket;
  
  connect() {
    this.connection = new WebSocket('ws://localhost');
  }
  
  disconnect() {
    this.connection?.close();
    this.connection = undefined;
  }
}
```

### 4. Document Lifetime

```typescript
/**
 * ConfigService - Application configuration
 * Lifetime: Singleton (providedIn: 'root')
 * State: Shared across application
 */
@Injectable({
  providedIn: 'root'
})
export class ConfigService {}
```

## When to Use Each Lifetime

### Use Singleton When:
- Sharing state across application
- Managing expensive resources (connections, caches)
- Configuration that doesn't change
- Coordinating between components

### Use Transient When:
- Service is stateless
- Each caller needs isolated instance
- Service is lightweight
- Avoiding shared state bugs

### Use Scoped When:
- Request-specific context
- Feature-specific state
- Transaction boundaries
- Isolated component trees

## Interview Questions

### Q1: What's the difference between singleton and transient lifetimes?
**Answer**: Singleton creates one instance for the entire application, sharing state across all consumers. Transient creates a new instance every time it's requested, providing isolation. Singletons use less memory but can have shared state issues, while transient services use more memory but avoid state conflicts.

### Q2: When would you use scoped lifetime over singleton?
**Answer**: Use scoped when you need state isolation per request, session, or component tree. For example, tracking request-specific data, managing transactions, or maintaining separate state for different features. Scoped provides isolation without the memory overhead of fully transient services.

### Q3: What memory implications do different lifetimes have?
**Answer**: Singleton: O(1) memory - one instance. Transient: O(n) memory - grows with usage. Scoped: O(s) memory where s is number of active scopes. Singletons are most memory-efficient but can cause memory leaks if holding references improperly. Transient services can use excessive memory if created frequently.

### Q4: How do you prevent memory leaks with singleton services?
**Answer**: 1) Provide cleanup methods to remove references, 2) Use WeakMap/WeakSet for storing references when possible, 3) Clean up subscriptions and event listeners, 4) Avoid storing references to transient objects, 5) Implement disposal patterns for resources.

## Key Takeaways

1. Three main lifetimes: Singleton, Transient, Scoped
2. Singleton: one instance, shared state, low memory
3. Transient: many instances, isolated state, high memory
4. Scoped: one per scope, balanced trade-off
5. Angular: providedIn: 'root' = Singleton
6. React: requires manual lifetime management
7. Choose lifetime based on state sharing needs
8. Memory leaks common with singletons
9. Document lifetime decisions
10. Test lifetime behavior explicitly

## Resources

- Angular Dependency Injection: https://angular.io/guide/dependency-injection
- Singleton Pattern: https://refactoring.guru/design-patterns/singleton
- Service Locator Pattern
- ASP.NET Core DI Lifetimes
- Memory Management in JavaScript
