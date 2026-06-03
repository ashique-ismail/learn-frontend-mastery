# Service Locator Pattern

## Overview

The Service Locator pattern provides a central registry where components can retrieve their dependencies rather than having them injected. While superficially similar to Dependency Injection, it's often considered an anti-pattern because it hides dependencies, making code harder to test and understand. However, understanding this pattern is important as it appears in legacy code and has specific use cases where it may be appropriate despite its drawbacks.

## Core Concepts

### Service Locator vs Dependency Injection

```
┌──────────────────────────────────────────────────────────┐
│      Service Locator vs Dependency Injection             │
└──────────────────────────────────────────────────────────┘

SERVICE LOCATOR (Pull Dependencies)
┌─────────────────────┐
│   UserService       │
│                     │
│ constructor() {     │
│   this.logger =     │
│     locator.get()───┼──┐
│ }                   │  │
└─────────────────────┘  │
                         ▼
                 ┌──────────────┐
                 │    Service   │
                 │    Locator   │
                 └──────────────┘

Problems:
- Hidden dependencies
- Hard to test
- Runtime errors

DEPENDENCY INJECTION (Push Dependencies)
┌─────────────────────┐
│   UserService       │◄───── Dependencies injected
│                     │
│ constructor(        │
│   logger: Logger    │
│ ) {}                │
└─────────────────────┘

Benefits:
- Explicit dependencies
- Easy to test
- Compile-time safety
```

## Basic Service Locator Implementation

### Simple Service Locator

```typescript
class ServiceLocator {
  private static services = new Map<string, any>();

  static register<T>(name: string, instance: T): void {
    this.services.set(name, instance);
  }

  static get<T>(name: string): T {
    const service = this.services.get(name);
    
    if (!service) {
      throw new Error(`Service "${name}" not found in locator`);
    }
    
    return service;
  }

  static has(name: string): boolean {
    return this.services.has(name);
  }

  static clear(): void {
    this.services.clear();
  }
}

// Logger service
class Logger {
  log(message: string) {
    console.log(`[LOG] ${message}`);
  }
}

// Database service
class Database {
  query(sql: string) {
    console.log(`Executing: ${sql}`);
    return [];
  }
}

// User service using service locator
class UserService {
  private logger: Logger;
  private db: Database;

  constructor() {
    // Pull dependencies from locator
    this.logger = ServiceLocator.get<Logger>('Logger');
    this.db = ServiceLocator.get<Database>('Database');
  }

  getUser(id: string) {
    this.logger.log(`Fetching user ${id}`);
    return this.db.query(`SELECT * FROM users WHERE id = '${id}'`);
  }
}

// Setup
ServiceLocator.register('Logger', new Logger());
ServiceLocator.register('Database', new Database());

// Usage
const userService = new UserService();
userService.getUser('123');
```

### Type-Safe Service Locator

```typescript
// Service tokens with types
const SERVICE_TOKENS = {
  Logger: Symbol.for('Logger'),
  Database: Symbol.for('Database'),
  Cache: Symbol.for('Cache')
} as const;

interface ServiceMap {
  [SERVICE_TOKENS.Logger]: Logger;
  [SERVICE_TOKENS.Database]: Database;
  [SERVICE_TOKENS.Cache]: Cache;
}

class TypedServiceLocator {
  private services = new Map<symbol, any>();

  register<K extends keyof ServiceMap>(
    token: K,
    instance: ServiceMap[K]
  ): void {
    this.services.set(token as symbol, instance);
  }

  get<K extends keyof ServiceMap>(token: K): ServiceMap[K] {
    const service = this.services.get(token as symbol);
    
    if (!service) {
      throw new Error(`Service for token ${String(token)} not found`);
    }
    
    return service;
  }

  has<K extends keyof ServiceMap>(token: K): boolean {
    return this.services.has(token as symbol);
  }
}

// Usage with type safety
const locator = new TypedServiceLocator();
locator.register(SERVICE_TOKENS.Logger, new Logger());

// Type-safe retrieval
const logger = locator.get(SERVICE_TOKENS.Logger); // Type: Logger
```

### Lazy Service Locator

```typescript
type ServiceFactory<T> = () => T;

class LazyServiceLocator {
  private factories = new Map<string, ServiceFactory<any>>();
  private instances = new Map<string, any>();

  registerFactory<T>(name: string, factory: ServiceFactory<T>): void {
    this.factories.set(name, factory);
  }

  registerSingleton<T>(name: string, factory: ServiceFactory<T>): void {
    this.factories.set(name, factory);
    // Mark as singleton
    this.instances.set(name, undefined);
  }

  get<T>(name: string): T {
    // Return existing singleton
    if (this.instances.has(name)) {
      let instance = this.instances.get(name);
      
      if (instance === undefined) {
        // Lazy create singleton
        const factory = this.factories.get(name);
        if (!factory) throw new Error(`Factory for "${name}" not found`);
        
        instance = factory();
        this.instances.set(name, instance);
      }
      
      return instance;
    }

    // Create transient instance
    const factory = this.factories.get(name);
    if (!factory) throw new Error(`Factory for "${name}" not found`);
    
    return factory();
  }
}

// Usage
const locator = new LazyServiceLocator();

locator.registerSingleton('Database', () => {
  console.log('Creating Database instance');
  return new Database();
});

locator.registerFactory('UserService', () => {
  const db = locator.get<Database>('Database');
  return new UserService(db);
});

// Database created only when first accessed
const service1 = locator.get<UserService>('UserService');
const service2 = locator.get<UserService>('UserService');
// Both services share the same Database instance
```

## React Service Locator Pattern

### Context-Based Service Locator

```typescript
import React, { createContext, useContext, ReactNode } from 'react';

interface Services {
  logger: Logger;
  api: ApiClient;
  analytics: Analytics;
}

const ServicesContext = createContext<Services | null>(null);

export const ServiceProvider: React.FC<{
  services: Services;
  children: ReactNode;
}> = ({ services, children }) => {
  return (
    <ServicesContext.Provider value={services}>
      {children}
    </ServicesContext.Provider>
  );
};

// Hook to access services (Service Locator)
export function useService<K extends keyof Services>(
  serviceName: K
): Services[K] {
  const services = useContext(ServicesContext);
  
  if (!services) {
    throw new Error('Services not provided');
  }
  
  return services[serviceName];
}

// Component using service locator
function UserProfile({ userId }: { userId: string }) {
  const api = useService('api');
  const logger = useService('logger');
  const [user, setUser] = React.useState(null);

  React.useEffect(() => {
    logger.log(`Loading user ${userId}`);
    api.getUser(userId).then(setUser);
  }, [userId, api, logger]);

  return <div>{user?.name}</div>;
}

// App setup
const services: Services = {
  logger: new Logger(),
  api: new ApiClient(),
  analytics: new Analytics()
};

function App() {
  return (
    <ServiceProvider services={services}>
      <UserProfile userId="123" />
    </ServiceProvider>
  );
}
```

### React with Global Service Locator

```typescript
// Global service locator (anti-pattern, but common)
class GlobalServices {
  private static instance: GlobalServices;
  private services = new Map<string, any>();

  private constructor() {}

  static getInstance(): GlobalServices {
    if (!GlobalServices.instance) {
      GlobalServices.instance = new GlobalServices();
    }
    return GlobalServices.instance;
  }

  register<T>(name: string, service: T): void {
    this.services.set(name, service);
  }

  get<T>(name: string): T {
    const service = this.services.get(name);
    if (!service) {
      throw new Error(`Service "${name}" not registered`);
    }
    return service;
  }
}

// Hook to access global services
function useGlobalService<T>(serviceName: string): T {
  return React.useMemo(
    () => GlobalServices.getInstance().get<T>(serviceName),
    [serviceName]
  );
}

// Component
function TodoList() {
  const todoService = useGlobalService<TodoService>('TodoService');
  const [todos, setTodos] = React.useState([]);

  React.useEffect(() => {
    todoService.getTodos().then(setTodos);
  }, [todoService]);

  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>{todo.text}</li>
      ))}
    </ul>
  );
}

// Setup before rendering
GlobalServices.getInstance().register('TodoService', new TodoService());
```

## Angular Service Locator (Injector)

### Using Angular Injector as Service Locator

```typescript
import { Component, Injector } from '@angular/core';
import { UserService } from './user.service';

@Component({
  selector: 'app-user-profile',
  template: `<div>{{ user?.name }}</div>`
})
export class UserProfileComponent {
  user: any;

  // Service Locator pattern (not recommended)
  constructor(private injector: Injector) {
    const userService = this.injector.get(UserService);
    userService.getUser('123').subscribe(user => {
      this.user = user;
    });
  }
}

// Better: Use DI directly (recommended)
@Component({
  selector: 'app-user-profile',
  template: `<div>{{ user?.name }}</div>`
})
export class UserProfileBetterComponent {
  user: any;

  constructor(private userService: UserService) {
    this.userService.getUser('123').subscribe(user => {
      this.user = user;
    });
  }
}
```

### Dynamic Service Resolution

```typescript
import { Injector, InjectionToken } from '@angular/core';

// Use case: Dynamic feature loading
interface FeatureService {
  name: string;
  execute(): void;
}

const FEATURE_SERVICES = new InjectionToken<Map<string, FeatureService>>(
  'FEATURE_SERVICES'
);

@Injectable()
export class FeatureManager {
  private featureServices: Map<string, FeatureService>;

  constructor(private injector: Injector) {
    // Valid use of injector when features are dynamic
    this.featureServices = this.injector.get(FEATURE_SERVICES);
  }

  executeFeature(featureName: string): void {
    const feature = this.featureServices.get(featureName);
    
    if (feature) {
      feature.execute();
    } else {
      console.warn(`Feature "${featureName}" not found`);
    }
  }

  hasFeature(featureName: string): boolean {
    return this.featureServices.has(featureName);
  }
}
```

## Problems with Service Locator

### Hidden Dependencies

```typescript
// ❌ Problem: Hidden dependencies
class OrderService {
  private logger: Logger;
  private db: Database;
  private email: EmailService;
  private payment: PaymentService;

  constructor() {
    // Dependencies hidden - not visible in signature
    this.logger = ServiceLocator.get('Logger');
    this.db = ServiceLocator.get('Database');
    this.email = ServiceLocator.get('EmailService');
    this.payment = ServiceLocator.get('PaymentService');
  }

  processOrder(order: Order) {
    // Uses all 4 dependencies, but not obvious from constructor
  }
}

// ✅ Solution: Explicit DI makes dependencies visible
class OrderService {
  constructor(
    private logger: Logger,
    private db: Database,
    private email: EmailService,
    private payment: PaymentService
  ) {
    // Dependencies explicit and visible
  }

  processOrder(order: Order) {
    // Clear which dependencies are used
  }
}
```

### Testing Difficulties

```typescript
// ❌ Problem: Hard to test with Service Locator
describe('UserService', () => {
  it('should fetch user', () => {
    // Must set up global service locator state
    ServiceLocator.clear();
    ServiceLocator.register('Logger', mockLogger);
    ServiceLocator.register('Database', mockDb);
    
    const service = new UserService();
    // Test...
  });

  afterEach(() => {
    // Must clean up global state
    ServiceLocator.clear();
  });
});

// ✅ Solution: Easy to test with DI
describe('UserService', () => {
  it('should fetch user', () => {
    // Just pass mocks directly
    const service = new UserService(mockLogger, mockDb);
    // Test...
  });
  // No global state to manage
});
```

### Runtime Errors

```typescript
// ❌ Problem: Runtime errors for missing services
class UserService {
  constructor() {
    // Throws at runtime if Logger not registered
    this.logger = ServiceLocator.get('Logger');
  }
}

// Forgot to register Logger
// ServiceLocator.register('Logger', new Logger());

const service = new UserService(); // Runtime error!

// ✅ Solution: Compile-time safety with DI
class UserService {
  constructor(private logger: Logger) {}
  // TypeScript ensures Logger is provided
}
```

### Tight Coupling to Locator

```typescript
// ❌ Problem: Coupled to service locator
class UserService {
  constructor() {
    this.logger = ServiceLocator.get('Logger');
  }
}

// Can't use UserService without ServiceLocator

// ✅ Solution: No coupling with DI
class UserService {
  constructor(private logger: Logger) {}
}

// Can instantiate anywhere with any logger
const service = new UserService(new ConsoleLogger());
```

## When Service Locator Might Be Acceptable

### Plugin Systems

```typescript
// Acceptable: Dynamic plugin loading
class PluginManager {
  private plugins = new Map<string, Plugin>();

  registerPlugin(name: string, plugin: Plugin): void {
    this.plugins.set(name, plugin);
  }

  getPlugin(name: string): Plugin | undefined {
    return this.plugins.get(name);
  }

  executePlugin(name: string, data: any): any {
    const plugin = this.plugins.get(name);
    
    if (plugin) {
      return plugin.execute(data);
    }
    
    throw new Error(`Plugin "${name}" not found`);
  }
}

// Plugins registered at runtime
pluginManager.registerPlugin('pdf-export', new PdfExportPlugin());
pluginManager.registerPlugin('excel-export', new ExcelExportPlugin());

// Dynamic execution based on user choice
const format = userChoice; // 'pdf-export' or 'excel-export'
pluginManager.executePlugin(format, reportData);
```

### Legacy Code Integration

```typescript
// Acceptable: Bridging legacy code that uses global state
class LegacyBridge {
  static getLegacyService(name: string): any {
    // Access legacy global services
    return (window as any).LegacyServices[name];
  }
}

// Modern code can wrap legacy access
class ModernService {
  private legacyAuth: any;

  constructor() {
    // Only place service locator is used
    this.legacyAuth = LegacyBridge.getLegacyService('auth');
  }

  async login(credentials: Credentials): Promise<User> {
    // Use legacy auth but return modern types
    const result = await this.legacyAuth.login(credentials);
    return this.convertLegacyUser(result);
  }
}
```

### Framework Integration

```typescript
// Acceptable: Framework requires service locator pattern
// (e.g., some older frameworks, WordPress, etc.)

// Framework's service locator
declare const FrameworkServices: {
  get(name: string): any;
};

// Wrapper to isolate framework coupling
class FrameworkServiceBridge {
  static createUserService(): UserService {
    const logger = FrameworkServices.get('logger');
    const db = FrameworkServices.get('database');
    return new UserService(logger, db);
  }
}

// Application code uses DI
class UserService {
  constructor(
    private logger: Logger,
    private db: Database
  ) {}
}

// Only bridge uses service locator
const userService = FrameworkServiceBridge.createUserService();
```

## Migrating from Service Locator to DI

### Step 1: Make Dependencies Explicit

```typescript
// Before
class UserService {
  constructor() {
    this.logger = ServiceLocator.get('Logger');
    this.db = ServiceLocator.get('Database');
  }
}

// After
class UserService {
  constructor(
    private logger: Logger = ServiceLocator.get('Logger'),
    private db: Database = ServiceLocator.get('Database')
  ) {}
}

// Now dependencies are visible, locator usage is explicit
```

### Step 2: Remove Default Parameters

```typescript
// Remove service locator from constructor
class UserService {
  constructor(
    private logger: Logger,
    private db: Database
  ) {}
}

// Update instantiation sites
const userService = new UserService(
  ServiceLocator.get('Logger'),
  ServiceLocator.get('Database')
);
```

### Step 3: Use DI Container

```typescript
// Setup DI container
container.register('Logger', Logger);
container.register('Database', Database);
container.register('UserService', UserService, ['Logger', 'Database']);

// Let container manage dependencies
const userService = container.resolve('UserService');
```

## Common Mistakes

### 1. Using Service Locator for Everything

```typescript
// ❌ Wrong - service locator for all dependencies
class MyComponent {
  constructor() {
    this.serviceA = locator.get('A');
    this.serviceB = locator.get('B');
    this.serviceC = locator.get('C');
  }
}

// ✅ Correct - use DI
class MyComponent {
  constructor(
    private serviceA: ServiceA,
    private serviceB: ServiceB,
    private serviceC: ServiceC
  ) {}
}
```

### 2. Global Mutable State

```typescript
// ❌ Wrong - global mutable locator
ServiceLocator.register('Logger', new ConsoleLogger());
// Somewhere else...
ServiceLocator.register('Logger', new FileLogger()); // Overwrites!

// ✅ Correct - immutable container or error on duplicate
container.register('Logger', ConsoleLogger);
container.register('Logger', FileLogger); // Throws error
```

### 3. Not Providing Test Doubles

```typescript
// ❌ Wrong - production services in tests
test('user service', () => {
  const service = new UserService();
  // Uses real ServiceLocator.get() calls
});

// ✅ Correct - mock dependencies
test('user service', () => {
  const mockLogger = { log: jest.fn() };
  const mockDb = { query: jest.fn() };
  const service = new UserService(mockLogger, mockDb);
});
```

## Best Practices (When Forced to Use It)

### 1. Limit Scope

```typescript
// Isolate service locator usage to factory functions
class ServiceFactory {
  static createUserService(): UserService {
    const logger = ServiceLocator.get<Logger>('Logger');
    const db = ServiceLocator.get<Database>('Database');
    return new UserService(logger, db);
  }
}

// Rest of code uses DI
const userService = ServiceFactory.createUserService();
```

### 2. Type Safety

```typescript
// Use TypeScript for type-safe retrieval
const SERVICES = {
  Logger: Symbol.for('Logger'),
  Database: Symbol.for('Database')
} as const;

interface ServiceMap {
  [SERVICES.Logger]: Logger;
  [SERVICES.Database]: Database;
}

class TypeSafeLocator {
  get<K extends keyof ServiceMap>(key: K): ServiceMap[K] {
    // Implementation
  }
}
```

### 3. Document Why

```typescript
/**
 * Uses service locator for plugin system.
 * Plugins are registered dynamically at runtime,
 * making traditional DI impractical.
 * 
 * @see PluginArchitecture.md for details
 */
class PluginManager {
  private locator = new ServiceLocator();
  
  // ...
}
```

## Interview Questions

### Q1: What is the Service Locator pattern?
**Answer**: Service Locator provides a central registry where components retrieve dependencies rather than having them injected. It's considered an anti-pattern because it hides dependencies, creates tight coupling to the locator, causes runtime errors for missing services, and makes testing harder compared to Dependency Injection.

### Q2: Why is Service Locator considered an anti-pattern?
**Answer**: It hides dependencies (not visible in constructor), couples code to the locator, causes runtime errors vs compile-time safety, makes testing harder (must mock global state), and reduces code clarity. DI solves all these issues by making dependencies explicit and injectable.

### Q3: When might Service Locator be acceptable?
**Answer**: For dynamic plugin systems, legacy code integration where refactoring isn't feasible, or when frameworks require it. Even then, isolate usage to factory methods or adapters, use type-safe implementations, and document why it's necessary. Prefer DI for new code.

### Q4: How does Service Locator differ from Dependency Injection?
**Answer**: Service Locator pulls dependencies (component requests them), DI pushes dependencies (injected from outside). Service Locator hides dependencies, DI makes them explicit. Service Locator has runtime errors, DI has compile-time safety. DI enables better testing and clearer code.

### Q5: How do you migrate from Service Locator to DI?
**Answer**: Make dependencies explicit as constructor parameters with default values using the locator. Update call sites to pass dependencies explicitly. Introduce DI container to manage instantiation. Remove service locator calls. Update tests to inject mocks. Refactor incrementally to avoid breaking changes.

## Key Takeaways

1. Service Locator is generally an anti-pattern in modern code
2. It hides dependencies and reduces code clarity
3. Testing is harder due to global state management
4. Runtime errors vs compile-time safety with DI
5. Tight coupling to the locator implementation
6. Acceptable for dynamic plugins or legacy integration
7. Isolate usage to factories if you must use it
8. Prefer Dependency Injection for explicit, testable code
9. Type safety helps mitigate some problems
10. Document clearly when and why you use it

## Resources

- [Martin Fowler on Service Locator](https://martinfowler.com/articles/injection.html#UsingAServiceLocator)
- [Service Locator is an Anti-Pattern](https://blog.ploeh.dk/2010/02/03/ServiceLocatorisanAnti-Pattern/)
- [Dependency Injection vs Service Locator](https://stackoverflow.com/questions/1557781/whats-the-difference-between-the-dependency-injection-and-service-locator-patte)
- [InversifyJS Service Locator Example](https://github.com/inversify/InversifyJS/blob/master/wiki/service_locator_pattern.md)
