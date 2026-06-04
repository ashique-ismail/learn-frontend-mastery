# Dependency Injection Frameworks

## Overview

Dependency Injection (DI) frameworks automate the creation and management of dependencies, removing the need for manual wiring. They provide features like automatic resolution, lifetime management, and configuration. Major frameworks include Angular's built-in DI system, InversifyJS for TypeScript, TSyringe for lightweight DI, and Awilix for Node.js applications.

## Core Concepts

### What DI Frameworks Provide

```
Manual DI:
┌──────────────┐
│  Developer   │
│   manually   │ ← Create instances
│   creates    │ ← Wire dependencies
│  and wires   │ ← Manage lifetimes
└──────────────┘

DI Framework:
┌──────────────┐
│   Register   │ ← Declare dependencies
│  services    │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Framework   │ ← Automatic resolution
│  resolves    │ ← Lifetime management
│dependencies  │ ← Circular detection
└──────────────┘
```

### Framework Comparison

| Framework | Type | Size | Learning Curve | Angular Integration |
|-----------|------|------|----------------|---------------------|
| Angular DI | Built-in | Medium | Medium | Native |
| InversifyJS | Full-featured | Large | High | External |
| TSyringe | Lightweight | Small | Low | External |
| Awilix | Node-focused | Medium | Medium | External |

## Angular Built-in DI

### Basic Registration

```typescript
// services/user.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';

// @Injectable decorator registers with DI system
@Injectable({
  providedIn: 'root' // Singleton registration
})
export class UserService {
  constructor(private http: HttpClient) {}
  
  getUser(id: string): Observable<User> {
    return this.http.get<User>(`/api/users/${id}`);
  }
}

// Component automatically gets injected service
@Component({
  selector: 'app-user-profile'
})
export class UserProfileComponent {
  constructor(private userService: UserService) {
    // Angular's DI system automatically provides UserService
  }
}
```

### Provider Configurations

```typescript
// app.module.ts
import { NgModule } from '@angular/core';

@NgModule({
  providers: [
    // 1. Class provider (most common)
    UserService,
    
    // 2. Class provider with token
    { provide: UserService, useClass: UserService },
    
    // 3. Alternative implementation
    { provide: UserService, useClass: MockUserService },
    
    // 4. Value provider
    { provide: 'API_URL', useValue: 'https://api.example.com' },
    
    // 5. Factory provider
    {
      provide: UserService,
      useFactory: (http: HttpClient, config: ConfigService) => {
        return new UserService(http, config.getApiUrl());
      },
      deps: [HttpClient, ConfigService]
    },
    
    // 6. Existing provider (alias)
    { provide: 'UserServiceAlias', useExisting: UserService }
  ]
})
export class AppModule {}
```

### InjectionToken for Non-Class Dependencies

```typescript
// tokens/api-config.token.ts
import { InjectionToken } from '@angular/core';

export interface ApiConfig {
  baseUrl: string;
  timeout: number;
  retries: number;
}

export const API_CONFIG = new InjectionToken<ApiConfig>('api.config');

// app.module.ts
@NgModule({
  providers: [
    {
      provide: API_CONFIG,
      useValue: {
        baseUrl: 'https://api.example.com',
        timeout: 5000,
        retries: 3
      }
    }
  ]
})
export class AppModule {}

// services/api.service.ts
@Injectable({
  providedIn: 'root'
})
export class ApiService {
  constructor(@Inject(API_CONFIG) private config: ApiConfig) {
    console.log('API URL:', this.config.baseUrl);
  }
}
```

### Hierarchical Injection

```typescript
// Root level - singleton
@Injectable({
  providedIn: 'root'
})
export class GlobalService {}

// Module level - one instance per module
@NgModule({
  providers: [ModuleScopedService]
})
export class FeatureModule {}

// Component level - one instance per component
@Component({
  selector: 'app-user-card',
  providers: [ComponentScopedService]
})
export class UserCardComponent {
  constructor(
    private global: GlobalService,         // From root
    private module: ModuleScopedService,   // From module
    private component: ComponentScopedService // From component
  ) {}
}

// Hierarchical resolution:
// ┌─────────────────┐
// │   Root Injector │ ← GlobalService
// └────────┬────────┘
//          │
// ┌────────▼─────────┐
// │ Module Injector  │ ← ModuleScopedService
// └────────┬─────────┘
//          │
// ┌────────▼──────────┐
// │Component Injector │ ← ComponentScopedService
// └───────────────────┘
```

### Optional and Multi-Providers

```typescript
// Optional injection
@Injectable()
export class AnalyticsService {
  constructor(@Optional() private logger?: LoggerService) {
    if (this.logger) {
      this.logger.log('Analytics initialized');
    }
  }
}

// Multi-providers
export const HTTP_INTERCEPTORS = new InjectionToken<HttpInterceptor[]>('http.interceptors');

@NgModule({
  providers: [
    { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true },
    { provide: HTTP_INTERCEPTORS, useClass: LoggingInterceptor, multi: true },
    { provide: HTTP_INTERCEPTORS, useClass: ErrorInterceptor, multi: true }
  ]
})
export class AppModule {}

// All interceptors injected as array
@Injectable()
export class HttpService {
  constructor(@Inject(HTTP_INTERCEPTORS) private interceptors: HttpInterceptor[]) {
    console.log(`Loaded ${this.interceptors.length} interceptors`);
  }
}
```

### Self and SkipSelf

```typescript
// @Self - only look in current injector
@Component({
  selector: 'app-child',
  providers: [LocalService]
})
export class ChildComponent {
  constructor(@Self() private local: LocalService) {
    // Only gets LocalService from this component's injector
    // Throws error if not provided in this component
  }
}

// @SkipSelf - skip current injector
@Component({
  selector: 'app-child'
})
export class ChildComponent {
  constructor(@SkipSelf() private parent: ParentService) {
    // Gets ParentService from parent injector
    // Skips current component's injector
  }
}

// @Host - only look up to host component
@Directive({
  selector: '[appHighlight]'
})
export class HighlightDirective {
  constructor(@Host() private component: HostComponent) {
    // Only searches up to the host component
  }
}
```

## InversifyJS

### Setup and Configuration

```typescript
// inversify.config.ts
import { Container } from 'inversify';
import 'reflect-metadata';

// Service identifiers
const TYPES = {
  ApiClient: Symbol.for('ApiClient'),
  UserService: Symbol.for('UserService'),
  CacheService: Symbol.for('CacheService'),
  LoggerService: Symbol.for('LoggerService')
};

// Service interfaces
interface IApiClient {
  get<T>(url: string): Promise<T>;
  post<T>(url: string, data: any): Promise<T>;
}

interface IUserService {
  getUser(id: string): Promise<User>;
}

interface ICacheService {
  get<T>(key: string): T | null;
  set<T>(key: string, value: T): void;
}

interface ILoggerService {
  log(message: string): void;
  error(message: string, error?: Error): void;
}

// Implementations
@injectable()
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

@injectable()
class CacheService implements ICacheService {
  private cache = new Map<string, any>();
  
  get<T>(key: string): T | null {
    return this.cache.get(key) ?? null;
  }
  
  set<T>(key: string, value: T): void {
    this.cache.set(key, value);
  }
}

@injectable()
class LoggerService implements ILoggerService {
  log(message: string): void {
    console.log(`[LOG] ${message}`);
  }
  
  error(message: string, error?: Error): void {
    console.error(`[ERROR] ${message}`, error);
  }
}

@injectable()
class UserService implements IUserService {
  constructor(
    @inject(TYPES.ApiClient) private api: IApiClient,
    @inject(TYPES.CacheService) private cache: ICacheService,
    @inject(TYPES.LoggerService) private logger: ILoggerService
  ) {}
  
  async getUser(id: string): Promise<User> {
    this.logger.log(`Fetching user ${id}`);
    
    const cached = this.cache.get<User>(`user:${id}`);
    if (cached) {
      this.logger.log(`Cache hit for user ${id}`);
      return cached;
    }
    
    const user = await this.api.get<User>(`/api/users/${id}`);
    this.cache.set(`user:${id}`, user);
    return user;
  }
}

// Create and configure container
const container = new Container();

// Bind services
container.bind<IApiClient>(TYPES.ApiClient).to(ApiClient).inSingletonScope();
container.bind<ICacheService>(TYPES.CacheService).to(CacheService).inSingletonScope();
container.bind<ILoggerService>(TYPES.LoggerService).to(LoggerService).inSingletonScope();
container.bind<IUserService>(TYPES.UserService).to(UserService).inTransientScope();

export { container, TYPES };
```

### Using InversifyJS in React

```typescript
// contexts/InversifyContext.tsx
import { createContext, useContext, ReactNode } from 'react';
import { Container } from 'inversify';

const InversifyContext = createContext<Container | null>(null);

export function InversifyProvider({ 
  children, 
  container 
}: { 
  children: ReactNode; 
  container: Container;
}) {
  return (
    <InversifyContext.Provider value={container}>
      {children}
    </InversifyContext.Provider>
  );
}

export function useInjection<T>(identifier: symbol): T {
  const container = useContext(InversifyContext);
  if (!container) {
    throw new Error('useInjection must be used within InversifyProvider');
  }
  return container.get<T>(identifier);
}

// App.tsx
import { container } from './inversify.config';

function App() {
  return (
    <InversifyProvider container={container}>
      <UserProfile userId="123" />
    </InversifyProvider>
  );
}

// components/UserProfile.tsx
function UserProfile({ userId }: { userId: string }) {
  const userService = useInjection<IUserService>(TYPES.UserService);
  const logger = useInjection<ILoggerService>(TYPES.LoggerService);
  
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    logger.log(`Loading user ${userId}`);
    userService.getUser(userId).then(setUser);
  }, [userId, userService, logger]);
  
  return user ? <div>{user.name}</div> : <div>Loading...</div>;
}
```

### Advanced InversifyJS Features

```typescript
// Named bindings
container.bind<ILogger>(TYPES.Logger)
  .to(ConsoleLogger)
  .whenTargetNamed('console');

container.bind<ILogger>(TYPES.Logger)
  .to(FileLogger)
  .whenTargetNamed('file');

@injectable()
class MultiLoggerService {
  constructor(
    @inject(TYPES.Logger) @named('console') private consoleLogger: ILogger,
    @inject(TYPES.Logger) @named('file') private fileLogger: ILogger
  ) {}
}

// Conditional bindings
container.bind<IApiClient>(TYPES.ApiClient)
  .to(MockApiClient)
  .when((request) => process.env.NODE_ENV === 'test');

container.bind<IApiClient>(TYPES.ApiClient)
  .to(RealApiClient)
  .when((request) => process.env.NODE_ENV === 'production');

// Dynamic value injection
container.bind<string>(TYPES.ApiUrl)
  .toDynamicValue(() => process.env.REACT_APP_API_URL || 'http://localhost:3000');

// Factory pattern
container.bind<interfaces.Factory<IUserService>>(TYPES.UserServiceFactory)
  .toFactory<IUserService>((context) => {
    return () => {
      return context.container.get<IUserService>(TYPES.UserService);
    };
  });

// Provider pattern
container.bind<IUserService>(TYPES.UserService)
  .toProvider<IUserService>((context) => {
    return async () => {
      const api = context.container.get<IApiClient>(TYPES.ApiClient);
      return new UserService(api);
    };
  });
```

## TSyringe

### Setup and Basic Usage

```typescript
// Install: npm install tsyringe reflect-metadata

// main.tsx
import 'reflect-metadata';
import { container } from 'tsyringe';

// services/ApiClient.ts
import { injectable } from 'tsyringe';

@injectable()
export class ApiClient {
  async get<T>(url: string): Promise<T> {
    const response = await fetch(url);
    return response.json();
  }
}

// services/UserService.ts
@injectable()
export class UserService {
  constructor(private apiClient: ApiClient) {}
  
  async getUser(id: string): Promise<User> {
    return this.apiClient.get<User>(`/api/users/${id}`);
  }
}

// Usage in component
function UserProfile({ userId }: { userId: string }) {
  const userService = container.resolve(UserService);
  
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    userService.getUser(userId).then(setUser);
  }, [userId]);
  
  return user ? <div>{user.name}</div> : <div>Loading...</div>;
}
```

### TSyringe Registration

```typescript
// di-setup.ts
import { container, Lifecycle } from 'tsyringe';

// Singleton (default)
container.registerSingleton(ApiClient);

// Transient
container.register(
  'DataProcessor',
  { useClass: DataProcessor },
  { lifecycle: Lifecycle.Transient }
);

// Resolution scoped
container.register(
  'RequestContext',
  { useClass: RequestContext },
  { lifecycle: Lifecycle.ResolutionScoped }
);

// Interface with token
export const IUserService = Symbol('IUserService');

container.register<IUserService>(
  IUserService,
  { useClass: UserService }
);

// Factory registration
container.register('UserServiceFactory', {
  useFactory: (dependencyContainer) => {
    const api = dependencyContainer.resolve(ApiClient);
    return new UserService(api);
  }
});

// Value registration
container.register('ApiConfig', {
  useValue: {
    baseUrl: 'https://api.example.com',
    timeout: 5000
  }
});
```

### TSyringe with React Hooks

```typescript
// hooks/useDI.ts
import { container } from 'tsyringe';
import { useRef } from 'react';

export function useDI<T>(token: InjectionToken<T>): T {
  const instanceRef = useRef<T>();
  
  if (!instanceRef.current) {
    instanceRef.current = container.resolve(token);
  }
  
  return instanceRef.current;
}

export function useTransientDI<T>(token: InjectionToken<T>): T {
  // Create new instance on every render
  return container.resolve(token);
}

// Usage
function MyComponent() {
  const userService = useDI(UserService);
  const logger = useDI('LoggerService');
  
  // Use services...
}
```

### TSyringe Scoped Containers

```typescript
// contexts/DIContext.tsx
import { DependencyContainer, container as rootContainer } from 'tsyringe';
import { createContext, useContext, ReactNode, useMemo } from 'react';

const DIContext = createContext<DependencyContainer>(rootContainer);

export function ScopedDIProvider({ children }: { children: ReactNode }) {
  // Create child container for this scope
  const scopedContainer = useMemo(() => {
    return rootContainer.createChildContainer();
  }, []);
  
  useEffect(() => {
    // Register scope-specific services
    scopedContainer.register('ScopeId', {
      useValue: Math.random().toString(36)
    });
    
    return () => {
      // Cleanup if needed
      scopedContainer.reset();
    };
  }, [scopedContainer]);
  
  return (
    <DIContext.Provider value={scopedContainer}>
      {children}
    </DIContext.Provider>
  );
}

export function useScopedDI<T>(token: InjectionToken<T>): T {
  const container = useContext(DIContext);
  return container.resolve(token);
}
```

## Awilix (Node.js)

### Setup and Configuration

```typescript
// container.ts
import { createContainer, asClass, asFunction, asValue, Lifetime } from 'awilix';

// Create container
const container = createContainer();

// Register with different lifetimes
container.register({
  // Singleton
  apiClient: asClass(ApiClient).singleton(),
  
  // Transient
  dataProcessor: asClass(DataProcessor).transient(),
  
  // Scoped (per request)
  requestContext: asClass(RequestContext).scoped(),
  
  // Function factory
  userServiceFactory: asFunction(({ apiClient, cache }) => {
    return new UserService(apiClient, cache);
  }).singleton(),
  
  // Value
  config: asValue({
    apiUrl: process.env.API_URL,
    port: process.env.PORT || 3000
  })
});

export { container };
```

### Awilix with Express

```typescript
// server.ts
import express from 'express';
import { scopePerRequest } from 'awilix-express';
import { container } from './container';

const app = express();

// Create scoped container per request
app.use(scopePerRequest(container));

// Middleware to access scoped services
app.use((req, res, next) => {
  // Each request gets its own scope
  req.container.register({
    requestId: asValue(Math.random().toString(36))
  });
  next();
});

// Routes
app.get('/users/:id', async (req, res) => {
  // Resolve service from request-scoped container
  const userService = req.container.resolve<UserService>('userService');
  const requestId = req.container.resolve<string>('requestId');
  
  console.log(`Request ${requestId} for user ${req.params.id}`);
  
  try {
    const user = await userService.getUser(req.params.id);
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: 'Failed to fetch user' });
  }
});

app.listen(3000);
```

### Awilix Auto-Loading

```typescript
// container.ts
import { createContainer, asClass, Lifetime } from 'awilix';
import { loadControllers, loadServices } from 'awilix-express';

const container = createContainer();

// Auto-load services from directory
container.loadModules([
  'services/**/*.ts',
  'repositories/**/*.ts'
], {
  formatName: 'camelCase',
  resolverOptions: {
    lifetime: Lifetime.SINGLETON,
    register: asClass
  }
});

// Load controllers
loadControllers(container, 'controllers/**/*.ts');

export { container };
```

## Framework Comparison Example

```typescript
// Same service, different frameworks

// 1. Angular
@Injectable({ providedIn: 'root' })
export class UserService {
  constructor(private http: HttpClient) {}
}

// Component
@Component({})
export class MyComponent {
  constructor(private userService: UserService) {}
}

// 2. InversifyJS
@injectable()
class UserService {
  constructor(@inject(TYPES.HttpClient) private http: IHttpClient) {}
}

// Usage
const userService = container.get<UserService>(TYPES.UserService);

// 3. TSyringe
@injectable()
class UserService {
  constructor(private http: HttpClient) {}
}

// Usage
const userService = container.resolve(UserService);

// 4. Manual (React Context)
class UserService {
  constructor(private http: HttpClient) {}
}

const ServiceContext = createContext<UserService | null>(null);

function Provider({ children }) {
  const service = useMemo(() => new UserService(new HttpClient()), []);
  return <ServiceContext.Provider value={service}>{children}</ServiceContext.Provider>;
}

function useUserService() {
  return useContext(ServiceContext);
}
```

## Testing with DI Frameworks

### Angular Testing

```typescript
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
  
  it('should fetch user', () => {
    const mockUser = { id: '1', name: 'John' };
    
    service.getUser('1').subscribe(user => {
      expect(user).toEqual(mockUser);
    });
    
    const req = httpMock.expectOne('/api/users/1');
    req.flush(mockUser);
  });
});
```

### InversifyJS Testing

```typescript
describe('UserService', () => {
  let container: Container;
  let mockApi: IApiClient;
  
  beforeEach(() => {
    mockApi = {
      get: jest.fn(),
      post: jest.fn()
    };
    
    container = new Container();
    container.bind<IApiClient>(TYPES.ApiClient).toConstantValue(mockApi);
    container.bind<IUserService>(TYPES.UserService).to(UserService);
  });
  
  it('should fetch user', async () => {
    const mockUser = { id: '1', name: 'John' };
    (mockApi.get as jest.Mock).mockResolvedValue(mockUser);
    
    const service = container.get<IUserService>(TYPES.UserService);
    const user = await service.getUser('1');
    
    expect(user).toEqual(mockUser);
  });
});
```

### TSyringe Testing

```typescript
describe('UserService', () => {
  let mockApi: ApiClient;
  
  beforeEach(() => {
    mockApi = {
      get: jest.fn()
    } as any;
    
    container.clearInstances();
    container.register(ApiClient, { useValue: mockApi });
  });
  
  it('should fetch user', async () => {
    const mockUser = { id: '1', name: 'John' };
    (mockApi.get as jest.Mock).mockResolvedValue(mockUser);
    
    const service = container.resolve(UserService);
    const user = await service.getUser('1');
    
    expect(user).toEqual(mockUser);
  });
});
```

## Common Mistakes

### 1. Forgetting reflect-metadata

```typescript
// Bad - missing reflect-metadata
import { injectable } from 'inversify';

@injectable()
class MyService {} // Runtime error!

// Good
import 'reflect-metadata'; // At app entry point
import { injectable } from 'inversify';

@injectable()
class MyService {} // Works
```

### 2. Circular Dependencies

```typescript
// Bad
@injectable()
class ServiceA {
  constructor(private serviceB: ServiceB) {}
}

@injectable()
class ServiceB {
  constructor(private serviceA: ServiceA) {} // Circular!
}

// Good - use interface or break dependency
interface IServiceB {
  doSomething(): void;
}

@injectable()
class ServiceA {
  private serviceB?: IServiceB;
  
  setServiceB(serviceB: IServiceB) {
    this.serviceB = serviceB;
  }
}
```

### 3. Wrong Lifetime

```typescript
// Bad - transient for expensive resource
container.register(DatabaseConnection, {
  useClass: DatabaseConnection
}, { lifecycle: Lifecycle.Transient }); // New connection every time!

// Good - singleton for shared resource
container.registerSingleton(DatabaseConnection);
```

## Best Practices

### 1. Use Interfaces

```typescript
// Define interface
interface IUserService {
  getUser(id: string): Promise<User>;
}

// Register with interface
container.register<IUserService>('IUserService', {
  useClass: UserService
});
```

### 2. Organize Registration

```typescript
// di/services.ts
export function registerServices(container: Container) {
  container.bind(TYPES.ApiClient).to(ApiClient).inSingletonScope();
  container.bind(TYPES.UserService).to(UserService).inTransientScope();
}

// di/repositories.ts
export function registerRepositories(container: Container) {
  container.bind(TYPES.UserRepo).to(UserRepository).inSingletonScope();
}

// di/index.ts
const container = new Container();
registerServices(container);
registerRepositories(container);
export { container };
```

### 3. Document Registrations

```typescript
/**
 * Application DI Container Configuration
 * 
 * Singletons:
 * - ApiClient: HTTP client
 * - CacheService: Application cache
 * 
 * Transient:
 * - DataProcessor: Stateless data processing
 */
```

## When to Use Each Framework

### Use Angular DI When:
- Building Angular applications (built-in)
- Need hierarchical injection
- Want framework integration

### Use InversifyJS When:
- Need full-featured DI in React/Vue
- Complex dependency graphs
- Advanced features (named bindings, etc.)

### Use TSyringe When:
- Want lightweight DI
- Simple to medium complexity
- React applications

### Use Awilix When:
- Building Node.js/Express apps
- Need per-request scoping
- Auto-loading modules

## Interview Questions

### Q1: What advantages do DI frameworks provide over manual DI?
**Answer**: Automatic dependency resolution, lifetime management, configuration in one place, circular dependency detection, easier testing with mocking, reduced boilerplate, and features like hierarchical injection and multi-providers.

### Q2: How does Angular's DI differ from InversifyJS?
**Answer**: Angular's DI is framework-integrated with hierarchical injectors, decorator-based registration, and tree-shakable providers. InversifyJS is standalone, requires explicit container configuration, uses symbol-based tokens, and provides more advanced features like named/tagged bindings.

### Q3: What is the purpose of InjectionToken in Angular?
**Answer**: InjectionToken creates unique tokens for registering non-class dependencies (primitives, objects, interfaces) in Angular's DI system. It prevents naming collisions and provides type safety for injected values.

### Q4: How do you prevent circular dependencies in DI frameworks?
**Answer**: Use interfaces/abstractions, lazy injection, setter injection for one dependency, event-based communication, or refactoring to remove the circular relationship. Most frameworks will throw errors when circular dependencies are detected.

## Key Takeaways

1. DI frameworks automate dependency management
2. Angular has built-in hierarchical DI system
3. InversifyJS provides full-featured DI for TypeScript
4. TSyringe is lightweight alternative
5. Awilix is best for Node.js/Express
6. All require reflect-metadata
7. Choose based on application needs
8. Properly configure lifetimes
9. Use interfaces for flexibility
10. Test by injecting mocks

## Resources

- Angular DI Guide: https://angular.io/guide/dependency-injection
- InversifyJS Documentation: https://inversify.io/
- TSyringe GitHub: https://github.com/microsoft/tsyringe
- Awilix Documentation: https://github.com/jeffijoe/awilix
- IoC Container Patterns
