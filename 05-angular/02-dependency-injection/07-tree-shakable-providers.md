# Tree-Shakable Providers in Angular

## The Idea

**In plain English:** Tree-shakable providers are a way to write Angular services (reusable pieces of code that do specific jobs in your app) so that if a service is never actually used, the build tool automatically leaves it out of the final app file — keeping the app smaller and faster to load.

**Real-world analogy:** Think of a restaurant that prints a menu with every dish they could possibly make, but only actually preps and cooks a dish when a customer orders it. A traditional kitchen stocks every ingredient upfront (bigger cost, more waste), while a smart kitchen only buys what gets ordered.

- The menu = the list of all services you have written
- A customer ordering a dish = a component or other service actually using (injecting) a service
- The kitchen only prepping ordered dishes = the build tool only including used services in the final bundle
- Dishes never ordered being left out = unused services being "tree-shaken" (removed) from the app

---

## Table of Contents

- [Introduction](#introduction)
- [Understanding Tree-Shaking](#understanding-tree-shaking)
- [providedIn Syntax](#providedin-syntax)
- [Tree-Shakable Provider Types](#tree-shakable-provider-types)
- [Optimization Strategies](#optimization-strategies)
- [Circular Dependencies](#circular-dependencies)
- [Migration from Module Providers](#migration-from-module-providers)
- [Advanced Patterns](#advanced-patterns)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Tree-shakable providers are Angular's mechanism for creating services that can be removed from the final bundle if they're never used. This optimization is crucial for modern Angular applications, significantly reducing bundle sizes and improving application performance.

Traditional module-based providers are always included in the bundle, even if never used. Tree-shakable providers use the `providedIn` syntax to reverse this relationship, allowing the build optimizer to eliminate unused services.

## Understanding Tree-Shaking

### What is Tree-Shaking?

Tree-shaking is a dead-code elimination technique used by modern JavaScript bundlers. It analyzes your code to determine which exports are actually used and removes unused code from the final bundle.

```typescript
// Without tree-shaking (traditional providers)
@NgModule({
  providers: [
    ServiceA,  // Always included, even if unused
    ServiceB,  // Always included, even if unused
    ServiceC   // Always included, even if unused
  ]
})
export class AppModule {}

// With tree-shaking (providedIn)
@Injectable({ providedIn: 'root' })
export class ServiceA {
  // Only included if used anywhere in the app
}

@Injectable({ providedIn: 'root' })
export class ServiceB {
  // Only included if used anywhere in the app
}

@Injectable({ providedIn: 'root' })
export class ServiceC {
  // Only included if used anywhere in the app
}
```

### How Angular Implements Tree-Shaking

```typescript
/*
Traditional approach (NOT tree-shakable):
  Module → declares providers → Services are imported
  Bundle includes: Module + ALL services (even unused ones)

Tree-shakable approach:
  Service → declares its own provider → Injected where needed
  Bundle includes: Only used services
  
The key difference: Dependency direction is reversed!
*/

// Service declares where it should be provided
@Injectable({
  providedIn: 'root' // Service knows about injection, not the module
})
export class TreeShakableService {
  constructor() {
    console.log('Service instantiated');
  }
}

// Module doesn't need to know about service
@NgModule({
  // No providers array needed!
})
export class AppModule {}
```

### Bundle Size Impact

```typescript
// Example: Before tree-shaking
// services.ts (150kb total)
export class AnalyticsService {} // 30kb - Used
export class LoggingService {}   // 20kb - Used
export class DebugService {}     // 40kb - UNUSED
export class MetricsService {}   // 30kb - UNUSED
export class TracingService {}   // 30kb - UNUSED

// app.module.ts
@NgModule({
  providers: [
    AnalyticsService,
    LoggingService,
    DebugService,    // Included even though unused
    MetricsService,  // Included even though unused
    TracingService   // Included even though unused
  ]
})
export class AppModule {}
// Final bundle: 150kb

// After tree-shaking with providedIn
@Injectable({ providedIn: 'root' })
export class AnalyticsService {} // 30kb - Used, included

@Injectable({ providedIn: 'root' })
export class LoggingService {} // 20kb - Used, included

@Injectable({ providedIn: 'root' })
export class DebugService {} // 40kb - Unused, REMOVED

@Injectable({ providedIn: 'root' })
export class MetricsService {} // 30kb - Unused, REMOVED

@Injectable({ providedIn: 'root' })
export class TracingService {} // 30kb - Unused, REMOVED

// Final bundle: 50kb (70% reduction!)
```

## providedIn Syntax

### Basic Usage

```typescript
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root'
})
export class BasicService {
  getData(): string {
    return 'data';
  }
}

// Usage in component
@Component({
  selector: 'app-example',
  template: '<p>{{ data }}</p>'
})
export class ExampleComponent {
  data: string;
  
  constructor(private service: BasicService) {
    this.data = this.service.getData();
  }
}
```

### providedIn: 'root'

```typescript
// Provided in root injector - singleton for entire application
@Injectable({
  providedIn: 'root'
})
export class RootScopedService {
  private count = 0;
  
  increment(): void {
    this.count++;
  }
  
  getCount(): number {
    return this.count;
  }
}

// Same instance shared across all components
@Component({
  selector: 'app-component-a',
  template: '<button (click)="increment()">Increment: {{ count }}</button>'
})
export class ComponentA {
  constructor(private service: RootScopedService) {}
  
  increment(): void {
    this.service.increment();
  }
  
  get count(): number {
    return this.service.getCount();
  }
}

@Component({
  selector: 'app-component-b',
  template: '<p>Count: {{ count }}</p>'
})
export class ComponentB {
  constructor(private service: RootScopedService) {}
  
  get count(): number {
    return this.service.getCount(); // Same count as ComponentA
  }
}
```

### providedIn: 'platform'

```typescript
// Shared across multiple Angular applications on the same page
@Injectable({
  providedIn: 'platform'
})
export class PlatformService {
  private sharedState = new Map<string, any>();
  
  setState(key: string, value: any): void {
    this.sharedState.set(key, value);
  }
  
  getState(key: string): any {
    return this.sharedState.get(key);
  }
}

// Useful for micro-frontends or multiple Angular apps
// All apps share the same instance
```

### providedIn: 'any'

```typescript
// Creates a unique instance per lazy-loaded module
@Injectable({
  providedIn: 'any'
})
export class ModuleScopedService {
  private moduleData: any[] = [];
  
  addData(data: any): void {
    this.moduleData.push(data);
  }
  
  getData(): any[] {
    return this.moduleData;
  }
}

// Each lazy-loaded module gets its own instance
@NgModule({
  imports: [CommonModule, RouterModule.forChild(routes)]
})
export class FeatureAModule {}

@NgModule({
  imports: [CommonModule, RouterModule.forChild(routes)]
})
export class FeatureBModule {}

// FeatureAModule and FeatureBModule have separate instances
```

### providedIn with Module Reference

```typescript
// Provided only when specific module is loaded
@Injectable({
  providedIn: FeatureModule
})
export class FeatureService {
  getFeatureData(): string {
    return 'Feature data';
  }
}

@NgModule({
  imports: [CommonModule]
})
export class FeatureModule {}

// Service is only included if FeatureModule is imported
// Enables tree-shaking at module level
```

## Tree-Shakable Provider Types

### Simple Services

```typescript
@Injectable({
  providedIn: 'root'
})
export class SimpleService {
  private data: string[] = [];
  
  addData(item: string): void {
    this.data.push(item);
  }
  
  getData(): string[] {
    return [...this.data];
  }
}
```

### Services with Dependencies

```typescript
@Injectable({
  providedIn: 'root'
})
export class LoggerService {
  log(message: string): void {
    console.log(`[LOG] ${message}`);
  }
}

@Injectable({
  providedIn: 'root'
})
export class DataService {
  constructor(private logger: LoggerService) {}
  
  fetchData(): void {
    this.logger.log('Fetching data...');
    // Fetch logic
  }
}

// Both services are tree-shakable
// If DataService is unused, both are removed
// If DataService is used, both are included
```

### Factory Providers

```typescript
import { Injectable, InjectionToken } from '@angular/core';

export const API_CONFIG = new InjectionToken<ApiConfig>('api.config');

export interface ApiConfig {
  baseUrl: string;
  timeout: number;
}

@Injectable({
  providedIn: 'root',
  useFactory: () => ({
    baseUrl: environment.apiUrl,
    timeout: 30000
  })
})
export class ApiConfigService implements ApiConfig {
  baseUrl: string;
  timeout: number;
  
  constructor() {
    const config = this.createConfig();
    this.baseUrl = config.baseUrl;
    this.timeout = config.timeout;
  }
  
  private createConfig(): ApiConfig {
    return {
      baseUrl: environment.apiUrl,
      timeout: 30000
    };
  }
}
```

### Value Providers

```typescript
import { InjectionToken } from '@angular/core';

export interface AppSettings {
  theme: string;
  language: string;
}

export const APP_SETTINGS = new InjectionToken<AppSettings>('app.settings', {
  providedIn: 'root',
  factory: () => ({
    theme: 'light',
    language: 'en'
  })
});

// Usage
@Component({
  selector: 'app-settings',
  template: '<p>Theme: {{ settings.theme }}</p>'
})
export class SettingsComponent {
  constructor(@Inject(APP_SETTINGS) public settings: AppSettings) {}
}
```

### Class Providers

```typescript
export abstract class StorageService {
  abstract save(key: string, value: any): void;
  abstract load(key: string): any;
}

@Injectable()
export class LocalStorageService implements StorageService {
  save(key: string, value: any): void {
    localStorage.setItem(key, JSON.stringify(value));
  }
  
  load(key: string): any {
    const item = localStorage.getItem(key);
    return item ? JSON.parse(item) : null;
  }
}

@Injectable({
  providedIn: 'root',
  useClass: LocalStorageService
})
export class StorageServiceImpl extends StorageService {
  save(key: string, value: any): void {
    localStorage.setItem(key, JSON.stringify(value));
  }
  
  load(key: string): any {
    const item = localStorage.getItem(key);
    return item ? JSON.parse(item) : null;
  }
}
```

## Optimization Strategies

### Lazy Loading with Tree-Shakable Providers

```typescript
// feature.service.ts
@Injectable({
  providedIn: FeatureModule // Only loaded with feature
})
export class FeatureService {
  loadFeatureData(): Observable<any> {
    return this.http.get('/api/feature-data');
  }
}

// feature.module.ts
@NgModule({
  imports: [
    CommonModule,
    RouterModule.forChild([
      { path: '', component: FeatureComponent }
    ])
  ]
})
export class FeatureModule {}

// app-routing.module.ts
const routes: Routes = [
  {
    path: 'feature',
    loadChildren: () => import('./feature/feature.module').then(m => m.FeatureModule)
  }
];

// FeatureService is only included when feature route is accessed
```

### Conditional Service Loading

```typescript
@Injectable({
  providedIn: 'root',
  useFactory: () => {
    return environment.production
      ? new ProductionAnalyticsService()
      : new DevelopmentAnalyticsService();
  }
})
export class AnalyticsService {
  trackEvent(event: string): void {
    // Implementation
  }
}

@Injectable()
export class ProductionAnalyticsService extends AnalyticsService {
  override trackEvent(event: string): void {
    // Send to analytics server
  }
}

@Injectable()
export class DevelopmentAnalyticsService extends AnalyticsService {
  override trackEvent(event: string): void {
    // Log to console only
    console.log('Analytics:', event);
  }
}
```

### Module-Level Tree-Shaking

```typescript
// shared.module.ts
@NgModule({})
export class SharedModule {}

// Shared services are tree-shakable
@Injectable({
  providedIn: SharedModule
})
export class SharedService {
  // Only included if SharedModule is imported
}

// feature-a.module.ts
@NgModule({
  imports: [SharedModule] // SharedService included
})
export class FeatureAModule {}

// feature-b.module.ts
@NgModule({
  // No SharedModule import
})
export class FeatureBModule {} // SharedService NOT included
```

### Token-Based Tree-Shaking

```typescript
import { InjectionToken } from '@angular/core';

export const LOGGER = new InjectionToken<Logger>('logger', {
  providedIn: 'root',
  factory: () => new ConsoleLogger()
});

export interface Logger {
  log(message: string): void;
}

export class ConsoleLogger implements Logger {
  log(message: string): void {
    console.log(message);
  }
}

// Service using token
@Injectable({
  providedIn: 'root'
})
export class DataService {
  constructor(@Inject(LOGGER) private logger: Logger) {}
  
  fetchData(): void {
    this.logger.log('Fetching data');
  }
}

// If DataService is unused, both DataService and ConsoleLogger are removed
```

### Provider Scoping Strategy

```typescript
// Global singleton services
@Injectable({ providedIn: 'root' })
export class AuthService {}

@Injectable({ providedIn: 'root' })
export class UserService {}

// Feature-specific services
@Injectable({ providedIn: AdminModule })
export class AdminService {}

@Injectable({ providedIn: DashboardModule })
export class DashboardService {}

// Component-specific services (not tree-shakable but scoped)
@Component({
  selector: 'app-form',
  template: '<form></form>',
  providers: [FormStateService] // New instance per component
})
export class FormComponent {}
```

## Circular Dependencies

### Understanding Circular Dependencies

```typescript
// BAD: Circular dependency
// service-a.ts
@Injectable({ providedIn: 'root' })
export class ServiceA {
  constructor(private serviceB: ServiceB) {} // Depends on ServiceB
}

// service-b.ts
@Injectable({ providedIn: 'root' })
export class ServiceB {
  constructor(private serviceA: ServiceA) {} // Depends on ServiceA
}

// Results in: ERROR: NG0200: Circular dependency in DI detected
```

### Breaking Circular Dependencies with forwardRef

```typescript
import { Injectable, forwardRef, Inject } from '@angular/core';

// service-a.ts
@Injectable({ providedIn: 'root' })
export class ServiceA {
  constructor(
    @Inject(forwardRef(() => ServiceB)) private serviceB: ServiceB
  ) {}
  
  methodA(): void {
    this.serviceB.methodB();
  }
}

// service-b.ts
@Injectable({ providedIn: 'root' })
export class ServiceB {
  constructor(private serviceA: ServiceA) {}
  
  methodB(): void {
    console.log('Method B called');
  }
}
```

### Using Injection Token to Break Cycles

```typescript
import { InjectionToken } from '@angular/core';

// Define interface
export interface NotificationService {
  notify(message: string): void;
}

// Create token
export const NOTIFICATION_SERVICE = new InjectionToken<NotificationService>(
  'notification.service'
);

// service-a.ts
@Injectable({ providedIn: 'root' })
export class ServiceA {
  constructor(
    @Inject(NOTIFICATION_SERVICE) private notifications: NotificationService
  ) {}
  
  doWork(): void {
    this.notifications.notify('Work completed');
  }
}

// service-b.ts
@Injectable({
  providedIn: 'root',
  useExisting: forwardRef(() => NotificationServiceImpl)
})
export class NotificationServiceImpl implements NotificationService {
  constructor(private serviceA: ServiceA) {}
  
  notify(message: string): void {
    console.log(message);
  }
}
```

### Refactoring to Eliminate Circular Dependencies

```typescript
// BAD: Circular dependency
@Injectable({ providedIn: 'root' })
export class UserService {
  constructor(private authService: AuthService) {}
  
  getCurrentUser(): User {
    if (this.authService.isAuthenticated()) {
      // Get user
    }
  }
}

@Injectable({ providedIn: 'root' })
export class AuthService {
  constructor(private userService: UserService) {}
  
  login(credentials: Credentials): void {
    // After login, load user
    this.userService.getCurrentUser();
  }
}

// GOOD: Extract shared logic to third service
@Injectable({ providedIn: 'root' })
export class AuthStateService {
  private authenticated = false;
  private currentUser: User | null = null;
  
  setAuthenticated(value: boolean): void {
    this.authenticated = value;
  }
  
  isAuthenticated(): boolean {
    return this.authenticated;
  }
  
  setCurrentUser(user: User): void {
    this.currentUser = user;
  }
  
  getCurrentUser(): User | null {
    return this.currentUser;
  }
}

@Injectable({ providedIn: 'root' })
export class UserService {
  constructor(private authState: AuthStateService) {}
  
  getCurrentUser(): User | null {
    return this.authState.getCurrentUser();
  }
  
  loadUser(): void {
    // Load user and update state
    this.authState.setCurrentUser(user);
  }
}

@Injectable({ providedIn: 'root' })
export class AuthService {
  constructor(
    private authState: AuthStateService,
    private userService: UserService
  ) {}
  
  login(credentials: Credentials): void {
    // Login logic
    this.authState.setAuthenticated(true);
    this.userService.loadUser();
  }
}
```

### Using Event Bus Pattern

```typescript
import { Injectable } from '@angular/core';
import { Subject, Observable } from 'rxjs';
import { filter, map } from 'rxjs/operators';

export interface DomainEvent {
  type: string;
  payload: any;
}

@Injectable({ providedIn: 'root' })
export class EventBus {
  private events$ = new Subject<DomainEvent>();
  
  emit(event: DomainEvent): void {
    this.events$.next(event);
  }
  
  on(eventType: string): Observable<any> {
    return this.events$.pipe(
      filter(event => event.type === eventType),
      map(event => event.payload)
    );
  }
}

// service-a.ts
@Injectable({ providedIn: 'root' })
export class ServiceA {
  constructor(private eventBus: EventBus) {
    this.eventBus.on('USER_LOGGED_IN').subscribe(user => {
      console.log('User logged in:', user);
    });
  }
}

// service-b.ts
@Injectable({ providedIn: 'root' })
export class ServiceB {
  constructor(private eventBus: EventBus) {}
  
  login(user: User): void {
    // Login logic
    this.eventBus.emit({
      type: 'USER_LOGGED_IN',
      payload: user
    });
  }
}
```

## Migration from Module Providers

### Before: Module-Based Providers

```typescript
// OLD: app.module.ts
@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule],
  providers: [
    ServiceA,
    ServiceB,
    ServiceC,
    { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true }
  ],
  bootstrap: [AppComponent]
})
export class AppModule {}

// services not tree-shakable
export class ServiceA {}
export class ServiceB {}
export class ServiceC {}
```

### After: Tree-Shakable Providers

```typescript
// NEW: app.config.ts (standalone)
export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([authInterceptor])
    )
  ]
};

// services.ts - tree-shakable
@Injectable({ providedIn: 'root' })
export class ServiceA {}

@Injectable({ providedIn: 'root' })
export class ServiceB {}

@Injectable({ providedIn: 'root' })
export class ServiceC {}

// main.ts
bootstrapApplication(AppComponent, appConfig);
```

### Migration Steps

```typescript
// Step 1: Identify services in providers array
@NgModule({
  providers: [
    ServiceA,        // Migrate to providedIn
    ServiceB,        // Migrate to providedIn
    ConfigService,   // Migrate to providedIn with factory
    { provide: API_URL, useValue: 'https://api.example.com' } // Migrate to token
  ]
})
export class AppModule {}

// Step 2: Add providedIn to services
@Injectable({ providedIn: 'root' })
export class ServiceA {}

@Injectable({ providedIn: 'root' })
export class ServiceB {}

@Injectable({
  providedIn: 'root',
  useFactory: () => new ConfigService(environment.config)
})
export class ConfigService {}

// Step 3: Convert value providers to tokens
export const API_URL = new InjectionToken<string>('api.url', {
  providedIn: 'root',
  factory: () => 'https://api.example.com'
});

// Step 4: Remove from module providers
@NgModule({
  providers: [] // Empty!
})
export class AppModule {}
```

### Handling Complex Providers

```typescript
// OLD: Complex factory provider
@NgModule({
  providers: [
    {
      provide: DataService,
      useFactory: (http: HttpClient, config: ConfigService) => {
        return new DataService(http, config.getApiUrl());
      },
      deps: [HttpClient, ConfigService]
    }
  ]
})
export class AppModule {}

// NEW: Tree-shakable factory
@Injectable({
  providedIn: 'root',
  useFactory: (http: HttpClient, config: ConfigService) => {
    return new DataService(http, config.getApiUrl());
  },
  deps: [HttpClient, ConfigService]
})
export class DataService {
  constructor(
    private http: HttpClient,
    private apiUrl: string
  ) {}
}

// Or better: Use constructor injection
@Injectable({ providedIn: 'root' })
export class DataService {
  private apiUrl: string;
  
  constructor(
    private http: HttpClient,
    private config: ConfigService
  ) {
    this.apiUrl = this.config.getApiUrl();
  }
}
```

## Advanced Patterns

### Multi-Provider Pattern

```typescript
import { InjectionToken } from '@angular/core';

export interface Plugin {
  name: string;
  execute(): void;
}

export const PLUGINS = new InjectionToken<Plugin[]>('plugins', {
  providedIn: 'root',
  factory: () => []
});

@Injectable({ providedIn: 'root' })
export class PluginA implements Plugin {
  name = 'Plugin A';
  execute(): void {
    console.log('Plugin A executed');
  }
}

@Injectable({ providedIn: 'root' })
export class PluginB implements Plugin {
  name = 'Plugin B';
  execute(): void {
    console.log('Plugin B executed');
  }
}

// Register plugins
export const pluginProviders = [
  { provide: PLUGINS, useClass: PluginA, multi: true },
  { provide: PLUGINS, useClass: PluginB, multi: true }
];

// Use plugins
@Injectable({ providedIn: 'root' })
export class PluginManager {
  constructor(@Inject(PLUGINS) private plugins: Plugin[]) {}
  
  executeAll(): void {
    this.plugins.forEach(plugin => plugin.execute());
  }
}
```

### Scoped Service Factories

```typescript
import { Injectable, Injector } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class ServiceFactory {
  constructor(private injector: Injector) {}
  
  createScopedService<T>(serviceType: Type<T>): T {
    const childInjector = Injector.create({
      providers: [{ provide: serviceType }],
      parent: this.injector
    });
    return childInjector.get(serviceType);
  }
}

// Usage
@Component({
  selector: 'app-dynamic',
  template: '<p>Dynamic service</p>'
})
export class DynamicComponent {
  constructor(private factory: ServiceFactory) {
    const scopedService = this.factory.createScopedService(MyService);
    scopedService.doWork();
  }
}
```

### Conditional Provider Pattern

```typescript
import { InjectionToken, Provider } from '@angular/core';

export const FEATURE_FLAGS = new InjectionToken<FeatureFlags>('feature.flags', {
  providedIn: 'root',
  factory: () => ({
    enableNewUI: false,
    enableBetaFeatures: false
  })
});

export function provideFeatureService(flags: FeatureFlags): Provider[] {
  const providers: Provider[] = [];
  
  if (flags.enableNewUI) {
    providers.push({
      provide: UIService,
      useClass: NewUIService
    });
  } else {
    providers.push({
      provide: UIService,
      useClass: LegacyUIService
    });
  }
  
  return providers;
}

// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    ...provideFeatureService(environment.featureFlags)
  ]
};
```

## Common Mistakes

### Mistake 1: Forgetting providedIn

```typescript
// BAD: Not tree-shakable
@Injectable()
export class MyService {}

// Module must provide it
@NgModule({
  providers: [MyService] // Not tree-shakable
})
export class AppModule {}

// GOOD: Tree-shakable
@Injectable({ providedIn: 'root' })
export class MyService {}

// No module provider needed
```

### Mistake 2: Using Both providedIn and Module Providers

```typescript
// BAD: Redundant and confusing
@Injectable({ providedIn: 'root' })
export class MyService {}

@NgModule({
  providers: [MyService] // Unnecessary!
})
export class AppModule {}

// GOOD: Use only providedIn
@Injectable({ providedIn: 'root' })
export class MyService {}

@NgModule({
  // No providers needed
})
export class AppModule {}
```

### Mistake 3: Incorrect Module Reference

```typescript
// BAD: Module doesn't exist yet
@Injectable({ providedIn: FeatureModule })
export class FeatureService {}

export class FeatureModule {} // Not decorated with @NgModule

// GOOD: Proper module
@NgModule({})
export class FeatureModule {}

@Injectable({ providedIn: FeatureModule })
export class FeatureService {}
```

### Mistake 4: Not Handling Circular Dependencies

```typescript
// BAD: Creates circular dependency
@Injectable({ providedIn: 'root' })
export class ServiceA {
  constructor(private serviceB: ServiceB) {}
}

@Injectable({ providedIn: 'root' })
export class ServiceB {
  constructor(private serviceA: ServiceA) {}
}

// GOOD: Use forwardRef or refactor
@Injectable({ providedIn: 'root' })
export class ServiceA {
  constructor(
    @Inject(forwardRef(() => ServiceB)) private serviceB: ServiceB
  ) {}
}
```

### Mistake 5: Overusing 'any' Scope

```typescript
// BAD: Unnecessary use of 'any'
@Injectable({ providedIn: 'any' })
export class GlobalService {} // Should be 'root'

// GOOD: Use 'any' only for module-scoped services
@Injectable({ providedIn: 'any' })
export class ModuleScopedService {} // Different instance per lazy module

@Injectable({ providedIn: 'root' })
export class GlobalService {} // Single instance
```

## Best Practices

### 1. Default to providedIn: 'root'

```typescript
// Start with root for most services
@Injectable({ providedIn: 'root' })
export class ApiService {}

@Injectable({ providedIn: 'root' })
export class AuthService {}

@Injectable({ providedIn: 'root' })
export class UserService {}
```

### 2. Use Module Scope for Feature Services

```typescript
@NgModule({})
export class AdminModule {}

@Injectable({ providedIn: AdminModule })
export class AdminService {
  // Only loaded when AdminModule is imported
}
```

### 3. Leverage Tree-Shaking for Large Services

```typescript
// Heavy service with many dependencies
@Injectable({ providedIn: 'root' })
export class AnalyticsService {
  constructor(
    private http: HttpClient,
    private storage: StorageService,
    private tracking: TrackingService
  ) {}
  
  // If unused, entire service tree is removed
}
```

### 4. Use Injection Tokens for Configuration

```typescript
export const APP_CONFIG = new InjectionToken<AppConfig>('app.config', {
  providedIn: 'root',
  factory: () => ({
    apiUrl: environment.apiUrl,
    timeout: 30000
  })
});
```

### 5. Document Provider Scope

```typescript
/**
 * Global authentication service.
 * Singleton across entire application.
 * Tree-shakable if not used.
 */
@Injectable({ providedIn: 'root' })
export class AuthService {}

/**
 * Feature-specific data service.
 * Only loaded with FeatureModule.
 * Tree-shakable if feature not loaded.
 */
@Injectable({ providedIn: FeatureModule })
export class FeatureDataService {}
```

## Interview Questions

### Q1: What is a tree-shakable provider?

**Answer:** A tree-shakable provider is a service that uses the `providedIn` syntax in its `@Injectable()` decorator. This allows Angular's build optimizer to remove the service from the final bundle if it's never used, reducing bundle size.

### Q2: What's the difference between providedIn: 'root' and 'any'?

**Answer:** `providedIn: 'root'` creates a single instance shared across the entire application. `providedIn: 'any'` creates a unique instance for each lazy-loaded module that injects it, allowing module-scoped state.

### Q3: How do tree-shakable providers improve performance?

**Answer:** They reduce bundle size by allowing unused services to be eliminated during the build process. This results in faster initial load times and better overall application performance.

### Q4: Can you use providedIn with a specific module?

**Answer:** Yes, you can use `providedIn: SomeModule` to scope a service to a specific module. The service will only be included if that module is imported.

### Q5: How do you handle circular dependencies with tree-shakable providers?

**Answer:** Use `forwardRef()`, extract shared logic to a third service, use injection tokens, or employ an event bus pattern to break the circular dependency.

## Key Takeaways

1. Tree-shakable providers reduce bundle size by eliminating unused services
2. Use `providedIn: 'root'` for application-wide singletons
3. Use `providedIn: SomeModule` for feature-scoped services
4. Use `providedIn: 'any'` for module-specific instances
5. Tree-shaking works by reversing the dependency direction
6. Avoid circular dependencies with forwardRef or refactoring
7. Prefer providedIn over module providers array
8. InjectionTokens can also be tree-shakable
9. Factory providers can be tree-shakable with providedIn
10. Tree-shakable providers are essential for modern Angular optimization

## Resources

- [Angular Dependency Injection Guide](https://angular.dev/guide/di)
- [Tree-Shakable Providers](https://angular.dev/guide/di/lightweight-injection-tokens)
- [providedIn Documentation](https://angular.dev/api/core/Injectable#providedIn)
- [Angular Performance Best Practices](https://angular.dev/best-practices/runtime-performance)
- [Bundle Size Optimization](https://web.dev/angular-bundle-size/)
