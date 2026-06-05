# Dependency Injection Implementation: Injector Tree and Resolution

## The Idea

**In plain English:** Dependency injection is a way for parts of a program to ask for the tools they need instead of building those tools themselves. Angular acts like a supply room manager that keeps track of every tool available and hands the right one to whoever asks for it.

**Real-world analogy:** Imagine a restaurant kitchen where a chef does not shop for ingredients personally. Instead, the chef submits a request list to a supply manager, who fetches everything from the storeroom and delivers it. If the main storeroom does not have something, the manager checks the walk-in cooler, then the dry-goods pantry, going up the chain until the item is found.

- The chef = a component or service that needs things to work
- The request list = the constructor parameters (dependencies declared in code)
- The supply manager = Angular's injector (the system that finds and delivers dependencies)
- The storeroom hierarchy (main room, cooler, pantry) = the injector tree (component injector, module injector, root injector)

---

## Overview

Angular's Dependency Injection (DI) system is one of the most sophisticated and powerful features of the framework. Unlike simple service locator patterns, Angular implements a hierarchical injector tree that mirrors the component tree, enabling precise control over dependency scope, lifetime, and resolution.

Understanding how Angular's DI works internally reveals the mechanisms behind provider resolution, injection tokens, hierarchical injectors, and tree-shakable providers - knowledge essential for building scalable, maintainable Angular applications.

## Injector Architecture

### Hierarchical Structure

```typescript
// Angular creates a tree of injectors
// Root Injector (Platform)
//   └── Module Injector (AppModule)
//       ├── Component Injector (AppComponent)
//       │   ├── Component Injector (HeaderComponent)
//       │   └── Component Injector (ContentComponent)
//       │       └── Component Injector (ItemComponent)
//       └── Component Injector (FooterComponent)

// Each injector can have its own providers
// Resolution walks up the tree until found
```

### Injector Implementation

```typescript
// Simplified injector implementation
class Injector {
  private providers = new Map<any, Provider>();
  private instances = new Map<any, any>();
  parent: Injector | null;
  
  constructor(providers: Provider[], parent: Injector | null = null) {
    this.parent = parent;
    
    // Register providers
    for (const provider of providers) {
      this.providers.set(provider.provide, provider);
    }
  }
  
  get<T>(token: Type<T> | InjectionToken<T>, notFoundValue?: T): T {
    // Check if already instantiated
    if (this.instances.has(token)) {
      return this.instances.get(token);
    }
    
    // Find provider
    const provider = this.providers.get(token);
    
    if (provider) {
      // Create instance
      const instance = this.createInstance(provider);
      
      // Cache if singleton
      if (!provider.multi) {
        this.instances.set(token, instance);
      }
      
      return instance;
    }
    
    // Try parent injector
    if (this.parent) {
      return this.parent.get(token, notFoundValue);
    }
    
    // Not found
    if (notFoundValue !== undefined) {
      return notFoundValue;
    }
    
    throw new Error(`No provider for ${token}`);
  }
  
  private createInstance(provider: Provider): any {
    if (provider.useClass) {
      return this.instantiateClass(provider.useClass);
    }
    if (provider.useFactory) {
      return provider.useFactory();
    }
    if (provider.useValue !== undefined) {
      return provider.useValue;
    }
    if (provider.useExisting) {
      return this.get(provider.useExisting);
    }
  }
  
  private instantiateClass(cls: Type<any>): any {
    // Get constructor dependencies
    const deps = this.getDependencies(cls);
    
    // Resolve dependencies
    const resolvedDeps = deps.map(dep => this.get(dep));
    
    // Create instance
    return new cls(...resolvedDeps);
  }
  
  private getDependencies(cls: Type<any>): any[] {
    // Extract from decorator metadata
    return cls.ɵfac?.deps || [];
  }
}
```

## Provider Types

### Class Providers

```typescript
// Most common: class providers
@Injectable({ providedIn: 'root' })
export class UserService {
  getUsers() {
    return ['Alice', 'Bob'];
  }
}

// Provider syntax
providers: [
  // Shorthand
  UserService,
  
  // Equivalent longhand
  { provide: UserService, useClass: UserService },
  
  // Different implementation
  { provide: UserService, useClass: MockUserService }
]

// Generated factory (simplified)
UserService.ɵfac = function(t) {
  return new (t || UserService)();
};

UserService.ɵprov = ɵɵdefineInjectable({
  token: UserService,
  factory: UserService.ɵfac,
  providedIn: 'root'
});
```

### Value Providers

```typescript
// Provide concrete values
export const API_URL = new InjectionToken<string>('API_URL');

providers: [
  { provide: API_URL, useValue: 'https://api.example.com' }
]

// Usage
@Injectable()
export class ApiService {
  constructor(@Inject(API_URL) private apiUrl: string) {
    console.log(this.apiUrl); // 'https://api.example.com'
  }
}

// Common use cases
const CONFIG = {
  apiUrl: 'https://api.example.com',
  timeout: 5000,
  retries: 3
};

providers: [
  { provide: 'APP_CONFIG', useValue: CONFIG }
]
```

### Factory Providers

```typescript
// Dynamic value creation
export function loggerFactory(isDev: boolean): Logger {
  return isDev ? new DevLogger() : new ProdLogger();
}

providers: [
  { 
    provide: Logger,
    useFactory: loggerFactory,
    deps: [IS_DEV]  // Dependencies for factory
  }
]

// Complex factory with multiple dependencies
export function apiServiceFactory(
  http: HttpClient,
  config: AppConfig,
  logger: Logger
): ApiService {
  const service = new ApiService(http, config.apiUrl);
  service.setLogger(logger);
  return service;
}

providers: [
  {
    provide: ApiService,
    useFactory: apiServiceFactory,
    deps: [HttpClient, AppConfig, Logger]
  }
]
```

### Existing Providers

```typescript
// Alias one provider to another
@Injectable()
export class DataService {
  getData() { return [1, 2, 3]; }
}

providers: [
  DataService,
  // Alias: NewDataService uses DataService instance
  { provide: 'NewDataService', useExisting: DataService }
]

// Common pattern: interface + implementation
export abstract class StorageService {
  abstract get(key: string): any;
  abstract set(key: string, value: any): void;
}

@Injectable()
export class LocalStorageService implements StorageService {
  get(key: string) { return localStorage.getItem(key); }
  set(key: string, value: any) { localStorage.setItem(key, value); }
}

providers: [
  LocalStorageService,
  { provide: StorageService, useExisting: LocalStorageService }
]
```

## Resolution Algorithm

### Dependency Resolution

```typescript
// Resolution algorithm (simplified)
class DependencyResolver {
  resolve<T>(
    token: Type<T> | InjectionToken<T>,
    injector: Injector,
    flags: InjectFlags = InjectFlags.Default
  ): T {
    // 1. Check self injector
    if (!(flags & InjectFlags.SkipSelf)) {
      const instance = injector.instances.get(token);
      if (instance !== undefined) {
        return instance;
      }
    }
    
    // 2. Walk up the injector tree
    let currentInjector: Injector | null = injector.parent;
    
    while (currentInjector) {
      const instance = currentInjector.instances.get(token);
      if (instance !== undefined) {
        return instance;
      }
      
      // Stop at host?
      if (flags & InjectFlags.Host && this.isHostBoundary(currentInjector)) {
        break;
      }
      
      currentInjector = currentInjector.parent;
    }
    
    // 3. Not found - optional or throw
    if (flags & InjectFlags.Optional) {
      return null as any;
    }
    
    throw new Error(`No provider for ${token}`);
  }
  
  private isHostBoundary(injector: Injector): boolean {
    // Check if this injector is marked as host
    return injector.flags & InjectorFlags.Host;
  }
}

// Usage in component
@Component({
  template: '...'
})
export class MyComponent {
  constructor(
    // Default: start from this injector, walk up
    private service: UserService,
    
    // Skip this injector, start from parent
    @SkipSelf() private parentService: UserService,
    
    // Don't throw if not found
    @Optional() private optionalService: AnalyticsService,
    
    // Stop at host component
    @Host() private hostService: LayoutService
  ) {}
}
```

### Injection Flags

```typescript
enum InjectFlags {
  Default = 0,      // Normal resolution
  Optional = 1,     // Return null if not found
  SkipSelf = 2,     // Skip current injector
  Self = 4,         // Only check current injector
  Host = 8          // Stop at host boundary
}

// Examples
@Component({
  providers: [ServiceA]
})
export class ParentComponent {
  constructor(private service: ServiceA) {
    // Gets instance from this component's injector
  }
}

@Component({
  selector: 'app-child'
})
export class ChildComponent {
  constructor(
    // Gets parent's ServiceA
    private service: ServiceA,
    
    // Skips self, gets parent's ServiceA
    @SkipSelf() private parentService: ServiceA,
    
    // Only checks ChildComponent injector (would throw)
    // @Self() private selfService: ServiceA,
    
    // Returns null if not found
    @Optional() private optionalService: ServiceB
  ) {}
}
```

## Injection Tokens

### String Tokens (Deprecated)

```typescript
// Old way - not type-safe
providers: [
  { provide: 'API_URL', useValue: 'https://api.example.com' }
]

constructor(@Inject('API_URL') apiUrl: string) {}
// Problems: no type checking, easy to typo
```

### InjectionToken

```typescript
// Modern way - type-safe
export const API_URL = new InjectionToken<string>('api.url', {
  providedIn: 'root',
  factory: () => 'https://api.example.com'
});

// Usage with type safety
constructor(@Inject(API_URL) private apiUrl: string) {
  // apiUrl is string, enforced by TypeScript
}

// With description for debugging
export const LOG_LEVEL = new InjectionToken<'debug' | 'info' | 'error'>(
  'log.level',
  {
    providedIn: 'root',
    factory: () => environment.production ? 'error' : 'debug'
  }
);
```

### Abstract Classes as Tokens

```typescript
// Abstract class as interface
export abstract class Logger {
  abstract log(message: string): void;
  abstract error(error: Error): void;
}

// Implementation
@Injectable()
export class ConsoleLogger extends Logger {
  log(message: string) {
    console.log(message);
  }
  
  error(error: Error) {
    console.error(error);
  }
}

// Provider
providers: [
  { provide: Logger, useClass: ConsoleLogger }
]

// Injection
constructor(private logger: Logger) {
  // Gets ConsoleLogger instance
  this.logger.log('Hello');
}
```

## Hierarchical Injection

### Component-Level Providers

```typescript
// Root-level service (singleton)
@Injectable({ providedIn: 'root' })
export class GlobalService {
  private data = 'global';
}

// Component-level service (new instance per component)
@Injectable()
export class InstanceService {
  private data = 'instance';
}

@Component({
  selector: 'app-parent',
  providers: [InstanceService], // New instance
  template: `
    <app-child></app-child>
    <app-child></app-child>
  `
})
export class ParentComponent {
  constructor(
    private global: GlobalService,      // Same instance everywhere
    private instance: InstanceService   // Unique to ParentComponent
  ) {}
}

@Component({
  selector: 'app-child'
})
export class ChildComponent {
  constructor(
    private global: GlobalService,      // Same instance as parent
    private instance: InstanceService   // Inherited from parent
  ) {}
}

// Result:
// - GlobalService: 1 instance shared by all
// - InstanceService: 1 instance shared by parent + 2 children
```

### Module-Level Providers

```typescript
// Feature module
@NgModule({
  providers: [FeatureService]
})
export class FeatureModule {}

// Service scoped to module
@Injectable()
export class FeatureService {
  constructor() {
    console.log('FeatureService created');
  }
}

// Lazy-loaded module gets its own injector
const routes = [
  {
    path: 'feature',
    loadChildren: () => import('./feature/feature.module')
      .then(m => m.FeatureModule)
  }
];

// Each lazy module gets own FeatureService instance
// Eagerly loaded modules share root injector
```

### ViewProviders

```typescript
// ViewProviders only available to view (not content children)
@Component({
  selector: 'app-parent',
  providers: [ServiceA],        // Available to view + content
  viewProviders: [ServiceB],     // Only available to view
  template: `
    <app-view-child></app-view-child>  <!-- Gets ServiceA + ServiceB -->
    <ng-content></ng-content>          <!-- Projects content -->
  `
})
export class ParentComponent {}

// Usage
@Component({
  template: `
    <app-parent>
      <app-content-child></app-content-child>  <!-- Gets ServiceA, NOT ServiceB -->
    </app-parent>
  `
})
export class AppComponent {}
```

## Tree-Shakable Providers

### providedIn Syntax

```typescript
// Old way - not tree-shakable
@Injectable()
export class OldService {}

@NgModule({
  providers: [OldService]
})
export class AppModule {}

// New way - tree-shakable
@Injectable({ providedIn: 'root' })
export class NewService {}

// No need to add to providers array
// Automatically included if used
// Removed if never imported
```

### How It Works

```typescript
// Generated code for tree-shakable service
@Injectable({ providedIn: 'root' })
export class UserService {
  getUsers() { return []; }
}

// Compiles to:
class UserService {
  getUsers() { return []; }
  
  static ɵfac = function UserService_Factory(t) {
    return new (t || UserService)();
  };
  
  static ɵprov = ɵɵdefineInjectable({
    token: UserService,
    factory: UserService.ɵfac,
    providedIn: 'root'  // Registered at root
  });
}

// If UserService is never injected anywhere:
// - Factory code is removed
// - Service code is removed
// - Registration is removed
// Complete tree-shaking!
```

### providedIn with Modules

```typescript
// Scope to specific module
@Injectable({ providedIn: FeatureModule })
export class FeatureService {}

// Only available within FeatureModule
// Tree-shaken if FeatureModule not imported

// Multiple modules
@Injectable({ providedIn: 'root' })
export class SharedService {}

// Available everywhere, but single instance
```

## Advanced DI Patterns

### Multi Providers

```typescript
// Multiple values for same token
export const PLUGIN = new InjectionToken<Plugin>('plugin');

@Injectable()
export class AnalyticsPlugin implements Plugin {
  initialize() { console.log('Analytics initialized'); }
}

@Injectable()
export class LoggingPlugin implements Plugin {
  initialize() { console.log('Logging initialized'); }
}

providers: [
  { provide: PLUGIN, useClass: AnalyticsPlugin, multi: true },
  { provide: PLUGIN, useClass: LoggingPlugin, multi: true }
]

// Injection returns array
constructor(@Inject(PLUGIN) private plugins: Plugin[]) {
  plugins.forEach(plugin => plugin.initialize());
  // Output:
  // Analytics initialized
  // Logging initialized
}
```

### Optional Dependencies

```typescript
@Injectable()
export class ServiceWithOptional {
  constructor(
    @Optional() private analytics: AnalyticsService,
    @Optional() private logger: LoggerService
  ) {
    // Use if available
    if (this.analytics) {
      this.analytics.track('Service created');
    }
    
    if (this.logger) {
      this.logger.log('Service initialized');
    }
  }
}

// Works even if AnalyticsService or LoggerService not provided
```

### Self vs SkipSelf

```typescript
@Component({
  selector: 'app-parent',
  providers: [DataService],
  template: '<app-child></app-child>'
})
export class ParentComponent {
  constructor(private data: DataService) {
    // Gets instance from this component
  }
}

@Component({
  selector: 'app-child',
  providers: [DataService]  // Override parent
})
export class ChildComponent {
  constructor(
    // Gets instance from ChildComponent injector
    @Self() private data: DataService,
    
    // Gets instance from ParentComponent injector
    @SkipSelf() private parentData: DataService
  ) {}
}
```

### Forward References

```typescript
// Circular dependency resolution
@Component({
  providers: [
    { 
      provide: ServiceA, 
      useClass: forwardRef(() => ServiceB)  // ServiceB not yet defined
    }
  ]
})
export class MyComponent {}

class ServiceB {}  // Defined later

// forwardRef delays resolution until runtime
```

## Common Misconceptions

### "Services are always singletons"

**Reality**: Depends on where provided:

```typescript
// Singleton - one instance app-wide
@Injectable({ providedIn: 'root' })
export class GlobalService {}

// New instance per component
@Component({
  providers: [PerComponentService]
})
export class MyComponent {}

// New instance per lazy module
@Injectable({ providedIn: LazyModule })
export class LazyService {}
```

### "Circular dependencies are impossible"

**Reality**: Angular detects and throws helpful errors:

```typescript
@Injectable()
export class ServiceA {
  constructor(private b: ServiceB) {}
}

@Injectable()
export class ServiceB {
  constructor(private a: ServiceA) {}
}

// Error: Circular dependency detected
// Use forwardRef or redesign
```

## Performance Implications

### Provider Lookup

```typescript
// Lookup performance
// Best: Direct class token
constructor(private service: MyService) {}
// O(1) lookup in injector map

// Good: InjectionToken
constructor(@Inject(MY_TOKEN) private value: any) {}
// O(1) lookup, but additional indirection

// Avoid: Deep hierarchies with many injectors
// O(n) where n = injector depth
```

### Memory Usage

```typescript
// providedIn: 'root' - Single instance
@Injectable({ providedIn: 'root' })
export class SingletonService {}
// Memory: ~constant

// Component providers - Multiple instances
@Component({
  providers: [PerComponentService]
})
// Memory: O(components)
```

## Interview Questions

### Q1: Explain Angular's hierarchical injector system.

**Answer**: Angular creates a tree of injectors that mirrors the application structure. There's a root injector (platform), module injectors, and component injectors for each component. When resolving a dependency, Angular starts at the requesting injector and walks up the tree until it finds a provider or reaches the root.

Example: If ChildComponent requests UserService, Angular checks ChildComponent's injector, then ParentComponent's injector, then module injector, then root injector. The first provider found wins. This allows different parts of the app to have different instances of the same service by providing it at different levels.

### Q2: What's the difference between providers and viewProviders?

**Answer**: Both provide services, but with different visibility:

**providers**: Available to the component's view AND projected content (ng-content)
**viewProviders**: Only available to the component's view, NOT to projected content

Example:
```typescript
@Component({
  providers: [ServiceA],
  viewProviders: [ServiceB],
  template: '<child-in-view></child-in-view><ng-content></ng-content>'
})
```

`child-in-view` gets both ServiceA and ServiceB. Content projected via ng-content only gets ServiceA. This prevents content children from depending on implementation details of the host component.

### Q3: How do tree-shakable providers work?

**Answer**: Tree-shakable providers use `providedIn` instead of module providers arrays:

```typescript
@Injectable({ providedIn: 'root' })
export class MyService {}
```

The service registers itself at the root injector. If the service is never injected anywhere, the entire service code (including its factory and registration) can be removed by the build optimizer. This works because the dependency flows from service to injector, not injector to service.

Traditional providers (`providers: [MyService]`) can't be tree-shaken because the module explicitly references the service, creating a hard dependency. Modern Angular favors providedIn for better tree-shaking.

### Q4: What are the different inject flags and when would you use them?

**Answer**: Inject flags modify dependency resolution:

**@Optional()**: Returns null if not found instead of throwing
```typescript
constructor(@Optional() logger?: Logger) {}
```

**@Self()**: Only looks in current injector, doesn't walk up
```typescript
constructor(@Self() service: DataService) {}
```

**@SkipSelf()**: Skips current injector, starts from parent
```typescript
constructor(@SkipSelf() parentService: DataService) {}
```

**@Host()**: Stops lookup at host component (useful with ng-content)
```typescript
constructor(@Host() hostService: LayoutService) {}
```

Use cases: @Optional for graceful degradation, @SkipSelf to access parent's instance, @Host to respect component boundaries, @Self to ensure local provision.

### Q5: How does Angular resolve circular dependencies?

**Answer**: Angular detects circular dependencies at runtime and throws an error. There are two solutions:

**1. forwardRef()**: Delays resolution
```typescript
@Injectable()
class ServiceA {
  constructor(@Inject(forwardRef(() => ServiceB)) b: ServiceB) {}
}
class ServiceB {
  constructor(private a: ServiceA) {}
}
```

**2. Redesign**: Better approach - extract shared logic to third service or use interfaces to break the cycle.

Circular dependencies usually indicate poor architecture. The dependency graph should be acyclic (DAG). If you encounter them, refactor to decouple services or use event-based communication instead.

### Q6: What happens when you provide the same service at multiple levels?

**Answer**: Each injector can have its own instance, creating different instances at different levels:

```typescript
@Injectable({ providedIn: 'root' })
export class DataService {}

@Component({
  providers: [DataService]  // Overrides root
})
export class ChildComponent {
  constructor(private data: DataService) {
    // Gets new instance from ChildComponent injector
  }
}
```

Resolution stops at the first provider found when walking up the tree. Children inherit the parent's instance unless they override with their own provider. This is useful for component-scoped state, but be careful - it can cause confusion if components expect shared state but get different instances.

### Q7: How do multi providers work?

**Answer**: Multi providers allow multiple values for the same token, injected as an array:

```typescript
export const HTTP_INTERCEPTOR = new InjectionToken<HttpInterceptor>('interceptor');

providers: [
  { provide: HTTP_INTERCEPTOR, useClass: AuthInterceptor, multi: true },
  { provide: HTTP_INTERCEPTOR, useClass: LogInterceptor, multi: true }
]

constructor(@Inject(HTTP_INTERCEPTOR) interceptors: HttpInterceptor[]) {
  // interceptors = [AuthInterceptor, LogInterceptor]
}
```

All providers with `multi: true` for the same token are collected into an array. Order is preserved. This is how Angular implements extensible systems like HTTP interceptors, where multiple independent providers contribute to the same collection.

### Q8: What's the difference between useClass, useValue, useFactory, and useExisting?

**Answer**:

**useClass**: Creates instance of specified class
```typescript
{ provide: Logger, useClass: ConsoleLogger }
```

**useValue**: Provides a concrete value
```typescript
{ provide: API_URL, useValue: 'https://api.com' }
```

**useFactory**: Calls factory function with dependencies
```typescript
{ provide: Logger, useFactory: loggerFactory, deps: [Config] }
```

**useExisting**: Aliases to another token
```typescript
{ provide: 'ILogger', useExisting: Logger }
```

Use useClass for polymorphism, useValue for constants, useFactory for complex initialization, useExisting for aliasing/interface pattern. useFactory is most flexible but has overhead of function call on every injection.

## Key Takeaways

1. Angular's DI uses a hierarchical injector tree mirroring the component tree
2. Resolution walks up the tree from requester to root until provider is found
3. Tree-shakable providers (providedIn) enable automatic removal of unused code
4. Different provider types (useClass, useValue, useFactory, useExisting) serve different purposes
5. Injection flags (@Optional, @Self, @SkipSelf, @Host) modify resolution behavior
6. Multi providers collect multiple values for same token into an array
7. viewProviders restrict visibility to component's view (excludes content children)
8. Provider scope determines instance lifetime and sharing across components

## Resources

- [Dependency Injection in Angular](https://angular.dev/guide/di)
- [Hierarchical Injectors](https://angular.dev/guide/di/hierarchical-dependency-injection)
- [DI Providers](https://angular.dev/guide/di/dependency-injection-providers)
- [Tree-Shakable Providers](https://angular.dev/guide/di/lightweight-injection-tokens)
- [Injection Tokens](https://angular.dev/api/core/InjectionToken)
- [Angular DI Internals](https://blog.angularindepth.com/angular-dependency-injection-internals-5f829c8d16f4)
- [Provider Scope in Angular](https://blog.angular.io/providedIn-any-2018-e2b48fce9b63)
