# Provider Types in Angular

## The Idea

**In plain English:** A provider type tells Angular exactly how to create or find a piece of code (called a "service") when another part of the app needs it — like a set of instructions that says "make a new one," "use this ready-made value," "look up the one we already have," or "run this custom recipe."

**Real-world analogy:** Imagine a school cafeteria that serves meals in four different ways. A student (component) just says "I need lunch," and the cafeteria (Angular's injector) follows one of four instructions to fulfill that request:
- The "cook a fresh meal" card (useClass) = create a new instance of a class
- The "here's a pre-made sandwich" card (useValue) = use a fixed, ready-made value
- The "go to the other counter, it's the same food" card (useExisting) = point to an already-registered provider
- The "follow this custom recipe" card (useFactory) = run a factory function with special logic to build the result

---

## Table of Contents
- [Introduction](#introduction)
- [Understanding Providers](#understanding-providers)
- [useClass Provider](#useclass-provider)
- [useValue Provider](#usevalue-provider)
- [useExisting Provider](#useexisting-provider)
- [useFactory Provider](#usefactory-provider)
- [Class Provider Shorthand](#class-provider-shorthand)
- [Comparison of Provider Types](#comparison-of-provider-types)
- [Practical Examples](#practical-examples)
- [Best Practices](#best-practices)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Resources](#resources)

## Introduction

Angular's dependency injection system supports multiple provider types, each serving different use cases. Understanding when and how to use each provider type is essential for creating flexible, testable, and maintainable applications.

Providers tell Angular's injector how to obtain or create dependencies. The four main provider types are: useClass, useValue, useExisting, and useFactory.

## Understanding Providers

A provider is a recipe for creating dependencies. The basic provider structure:

```typescript
{
  provide: Token,        // What to provide
  useClass: Class        // How to provide it
}
```

## useClass Provider

Creates a new instance of the specified class when requested.

### Basic Usage

```typescript
// Service class
@Injectable()
export class LoggerService {
  log(message: string): void {
    console.log(message);
  }
}

// Provider using useClass
@NgModule({
  providers: [
    { provide: LoggerService, useClass: LoggerService }
  ]
})
export class AppModule {}

// Shorthand (same as above)
@NgModule({
  providers: [LoggerService]  // Angular automatically uses useClass
})
export class AppModule {}
```

### Providing Alternative Implementation

```typescript
// Base class or interface
export abstract class Logger {
  abstract log(message: string): void;
  abstract error(message: string): void;
}

// Development implementation
@Injectable()
export class ConsoleLogger extends Logger {
  log(message: string): void {
    console.log('[LOG]', message);
  }
  
  error(message: string): void {
    console.error('[ERROR]', message);
  }
}

// Production implementation
@Injectable()
export class RemoteLogger extends Logger {
  constructor(private http: HttpClient) {
    super();
  }
  
  log(message: string): void {
    this.http.post('/api/logs', { level: 'log', message }).subscribe();
  }
  
  error(message: string): void {
    this.http.post('/api/logs', { level: 'error', message }).subscribe();
  }
}

// Provide different implementation based on environment
@NgModule({
  providers: [
    {
      provide: Logger,
      useClass: environment.production ? RemoteLogger : ConsoleLogger
    }
  ]
})
export class AppModule {}

// Components use abstract class
@Component({
  selector: 'app-data'
})
export class DataComponent {
  constructor(private logger: Logger) {
    this.logger.log('Component initialized');
  }
}
```

### Testing with useClass

```typescript
// Mock implementation for testing
@Injectable()
export class MockLogger extends Logger {
  logs: string[] = [];
  errors: string[] = [];
  
  log(message: string): void {
    this.logs.push(message);
  }
  
  error(message: string): void {
    this.errors.push(message);
  }
}

// Test configuration
TestBed.configureTestingModule({
  providers: [
    DataComponent,
    { provide: Logger, useClass: MockLogger }
  ]
});
```

## useValue Provider

Provides a pre-existing value or object.

### Primitive Values

```typescript
// API configuration
const API_URL = new InjectionToken<string>('api.url');

@NgModule({
  providers: [
    { provide: API_URL, useValue: 'https://api.example.com' }
  ]
})

// Usage
@Injectable({ providedIn: 'root' })
export class ApiService {
  constructor(@Inject(API_URL) private apiUrl: string) {}
}
```

### Configuration Objects

```typescript
export interface AppConfig {
  apiUrl: string;
  timeout: number;
  retries: number;
}

const APP_CONFIG = new InjectionToken<AppConfig>('app.config');

const config: AppConfig = {
  apiUrl: 'https://api.example.com',
  timeout: 30000,
  retries: 3
};

@NgModule({
  providers: [
    { provide: APP_CONFIG, useValue: config }
  ]
})
```

### Mock Objects for Testing

```typescript
const mockHttpClient = {
  get: jasmine.createSpy('get').and.returnValue(of({ data: 'test' })),
  post: jasmine.createSpy('post').and.returnValue(of({ success: true }))
};

TestBed.configureTestingModule({
  providers: [
    { provide: HttpClient, useValue: mockHttpClient }
  ]
});
```

### Browser APIs

```typescript
const WINDOW = new InjectionToken<Window>('window');
const LOCAL_STORAGE = new InjectionToken<Storage>('localStorage');

@NgModule({
  providers: [
    { provide: WINDOW, useValue: window },
    { provide: LOCAL_STORAGE, useValue: localStorage }
  ]
})

// Usage
@Injectable({ providedIn: 'root' })
export class StorageService {
  constructor(@Inject(LOCAL_STORAGE) private storage: Storage) {}
  
  set(key: string, value: string): void {
    this.storage.setItem(key, value);
  }
}
```

## useExisting Provider

Creates an alias for an existing provider.

### Basic Aliasing

```typescript
@Injectable({ providedIn: 'root' })
export class LoggerService {
  log(message: string): void {
    console.log(message);
  }
}

// Create alias
const LOGGER = new InjectionToken<LoggerService>('logger');

@NgModule({
  providers: [
    { provide: LOGGER, useExisting: LoggerService }
  ]
})

// Both inject the same instance
constructor(
  private logger1: LoggerService,
  @Inject(LOGGER) private logger2: LoggerService
) {
  console.log(logger1 === logger2);  // true - same instance
}
```

### Migration Pattern

```typescript
// Old service
@Injectable({ providedIn: 'root' })
export class OldDataService {
  getData(): Observable<any[]> {
    return this.http.get<any[]>('/api/data');
  }
}

// New service with better name
@Injectable({ providedIn: 'root' })
export class DataService {
  getData(): Observable<any[]> {
    return this.http.get<any[]>('/api/data');
  }
}

// Alias old to new for backward compatibility
@NgModule({
  providers: [
    { provide: OldDataService, useExisting: DataService }
  ]
})

// Both work, same instance
constructor(
  private newService: DataService,
  private oldService: OldDataService
) {
  console.log(newService === oldService);  // true
}
```

### Interface Implementation

```typescript
export interface Validator {
  validate(value: any): boolean;
}

@Injectable({ providedIn: 'root' })
export class EmailValidator implements Validator {
  validate(value: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value);
  }
}

const EMAIL_VALIDATOR = new InjectionToken<Validator>('email.validator');

@NgModule({
  providers: [
    EmailValidator,
    { provide: EMAIL_VALIDATOR, useExisting: EmailValidator }
  ]
})
```

## useFactory Provider

Uses a factory function to create the dependency.

### Basic Factory

```typescript
function loggerFactory(): Logger {
  return environment.production 
    ? new RemoteLogger() 
    : new ConsoleLogger();
}

@NgModule({
  providers: [
    { provide: Logger, useFactory: loggerFactory }
  ]
})
```

### Factory with Dependencies

```typescript
function apiServiceFactory(
  http: HttpClient,
  config: ConfigService
): ApiService {
  const baseUrl = config.get('apiUrl');
  return new ApiService(http, baseUrl);
}

@NgModule({
  providers: [
    {
      provide: ApiService,
      useFactory: apiServiceFactory,
      deps: [HttpClient, ConfigService]  // Dependencies injected into factory
    }
  ]
})
```

### Conditional Factory

```typescript
function storageFactory(platformId: Object): Storage {
  if (isPlatformBrowser(platformId)) {
    return window.localStorage;
  } else {
    // Server-side storage implementation
    return new Map() as any;
  }
}

const STORAGE = new InjectionToken<Storage>('storage');

@NgModule({
  providers: [
    {
      provide: STORAGE,
      useFactory: storageFactory,
      deps: [PLATFORM_ID]
    }
  ]
})
```

### Complex Initialization

```typescript
function webSocketFactory(
  config: ConfigService,
  auth: AuthService
): WebSocketSubject<any> {
  const url = config.get('wsUrl');
  const token = auth.getToken();
  
  return webSocket({
    url: `${url}?token=${token}`,
    deserializer: (e) => JSON.parse(e.data),
    serializer: (value) => JSON.stringify(value)
  });
}

const WEB_SOCKET = new InjectionToken<WebSocketSubject<any>>('websocket');

@NgModule({
  providers: [
    {
      provide: WEB_SOCKET,
      useFactory: webSocketFactory,
      deps: [ConfigService, AuthService]
    }
  ]
})
```

### Factory with inject() Function

```typescript
const CURRENT_USER = new InjectionToken<Observable<User | null>>('current.user', {
  factory: () => {
    const authService = inject(AuthService);
    return authService.currentUser$;
  }
});
```

## Class Provider Shorthand

Angular provides convenient shorthand syntax:

```typescript
// Full syntax
@NgModule({
  providers: [
    { provide: UserService, useClass: UserService }
  ]
})

// Shorthand (equivalent)
@NgModule({
  providers: [UserService]
})

// Multiple services
@NgModule({
  providers: [
    UserService,
    DataService,
    ApiService
  ]
})
```

## Comparison of Provider Types

### When to Use Each

```typescript
// useClass: Different implementations
{
  provide: Logger,
  useClass: environment.production ? ProductionLogger : DevLogger
}

// useValue: Static configuration
{
  provide: API_URL,
  useValue: 'https://api.example.com'
}

// useExisting: Create alias
{
  provide: OldService,
  useExisting: NewService
}

// useFactory: Complex creation logic
{
  provide: Connection,
  useFactory: createConnection,
  deps: [Config, Auth]
}
```

### Performance Considerations

```typescript
// useClass: New instance created for each injection context
{ provide: Logger, useClass: Logger }  // Memory: High

// useValue: Single object shared
{ provide: CONFIG, useValue: config }  // Memory: Low

// useExisting: References existing singleton
{ provide: LoggerAlias, useExisting: Logger }  // Memory: Low

// useFactory: Depends on factory implementation
{
  provide: Service,
  useFactory: () => new Service()  // Memory: High (new instance)
}

{
  provide: Service,
  useFactory: () => singletonInstance  // Memory: Low (shared)
}
```

## Practical Examples

### Example 1: Multi-Environment Logger

```typescript
// logger.interface.ts
export interface ILogger {
  debug(message: string, ...args: any[]): void;
  info(message: string, ...args: any[]): void;
  warn(message: string, ...args: any[]): void;
  error(message: string, ...args: any[]): void;
}

// console-logger.ts
@Injectable()
export class ConsoleLogger implements ILogger {
  debug(message: string, ...args: any[]): void {
    console.debug(message, ...args);
  }
  
  info(message: string, ...args: any[]): void {
    console.info(message, ...args);
  }
  
  warn(message: string, ...args: any[]): void {
    console.warn(message, ...args);
  }
  
  error(message: string, ...args: any[]): void {
    console.error(message, ...args);
  }
}

// remote-logger.ts
@Injectable()
export class RemoteLogger implements ILogger {
  constructor(private http: HttpClient) {}
  
  private send(level: string, message: string, args: any[]): void {
    this.http.post('/api/logs', {
      level,
      message,
      data: args,
      timestamp: new Date().toISOString()
    }).subscribe();
  }
  
  debug(message: string, ...args: any[]): void {
    this.send('debug', message, args);
  }
  
  info(message: string, ...args: any[]): void {
    this.send('info', message, args);
  }
  
  warn(message: string, ...args: any[]): void {
    this.send('warn', message, args);
  }
  
  error(message: string, ...args: any[]): void {
    this.send('error', message, args);
  }
}

// logger.token.ts
export const LOGGER = new InjectionToken<ILogger>('logger');

// app.config.ts
function loggerFactory(http: HttpClient): ILogger {
  if (environment.production) {
    return new RemoteLogger(http);
  } else {
    return new ConsoleLogger();
  }
}

export const appConfig: ApplicationConfig = {
  providers: [
    {
      provide: LOGGER,
      useFactory: loggerFactory,
      deps: [HttpClient]
    }
  ]
};
```

### Example 2: Feature Toggle System

```typescript
// feature-toggle.interface.ts
export interface FeatureToggle {
  isEnabled(feature: string): boolean;
}

// static-toggle.service.ts
@Injectable()
export class StaticToggleService implements FeatureToggle {
  private features: Record<string, boolean>;
  
  constructor(@Inject(FEATURE_CONFIG) config: Record<string, boolean>) {
    this.features = config;
  }
  
  isEnabled(feature: string): boolean {
    return this.features[feature] || false;
  }
}

// dynamic-toggle.service.ts
@Injectable()
export class DynamicToggleService implements FeatureToggle {
  private features$ = new BehaviorSubject<Record<string, boolean>>({});
  
  constructor(private http: HttpClient) {
    this.loadFeatures();
  }
  
  private loadFeatures(): void {
    this.http.get<Record<string, boolean>>('/api/features')
      .subscribe(features => this.features$.next(features));
  }
  
  isEnabled(feature: string): boolean {
    return this.features$.value[feature] || false;
  }
}

// Configuration
const FEATURE_TOGGLE = new InjectionToken<FeatureToggle>('feature.toggle');
const FEATURE_CONFIG = new InjectionToken<Record<string, boolean>>('feature.config');

bootstrapApplication(AppComponent, {
  providers: [
    {
      provide: FEATURE_CONFIG,
      useValue: {
        newDashboard: true,
        betaFeatures: false
      }
    },
    {
      provide: FEATURE_TOGGLE,
      useClass: environment.production 
        ? DynamicToggleService 
        : StaticToggleService
    }
  ]
});
```

### Example 3: Database Connection Factory

```typescript
// database.interface.ts
export interface DatabaseConnection {
  query<T>(sql: string, params?: any[]): Promise<T>;
  close(): Promise<void>;
}

// postgres.connection.ts
export class PostgresConnection implements DatabaseConnection {
  constructor(private config: DbConfig) {}
  
  async query<T>(sql: string, params?: any[]): Promise<T> {
    // PostgreSQL implementation
    return {} as T;
  }
  
  async close(): Promise<void> {
    // Close connection
  }
}

// mysql.connection.ts
export class MySQLConnection implements DatabaseConnection {
  constructor(private config: DbConfig) {}
  
  async query<T>(sql: string, params?: any[]): Promise<T> {
    // MySQL implementation
    return {} as T;
  }
  
  async close(): Promise<void> {
    // Close connection
  }
}

// Factory
function databaseFactory(config: ConfigService): DatabaseConnection {
  const dbConfig = config.get('database');
  
  switch (dbConfig.type) {
    case 'postgres':
      return new PostgresConnection(dbConfig);
    case 'mysql':
      return new MySQLConnection(dbConfig);
    default:
      throw new Error(`Unsupported database type: ${dbConfig.type}`);
  }
}

const DATABASE = new InjectionToken<DatabaseConnection>('database');

@NgModule({
  providers: [
    {
      provide: DATABASE,
      useFactory: databaseFactory,
      deps: [ConfigService]
    }
  ]
})
```

## Best Practices

### 1. Choose the Right Provider Type

```typescript
// Static config → useValue
{ provide: API_URL, useValue: 'https://api.example.com' }

// Polymorphism → useClass
{ provide: Logger, useClass: ProductionLogger }

// Aliasing → useExisting
{ provide: LegacyService, useExisting: ModernService }

// Complex logic → useFactory
{ provide: Connection, useFactory: createConnection, deps: [Config] }
```

### 2. Use Abstract Classes for Polymorphism

```typescript
// Good: Abstract class with implementations
export abstract class DataStore {
  abstract save(data: any): Promise<void>;
  abstract load(id: string): Promise<any>;
}

{ provide: DataStore, useClass: LocalStorageStore }

// Avoid: Using concrete class as interface
{ provide: ConcreteService, useClass: MockService }
```

### 3. Type-Safe Injection Tokens

```typescript
// Good: Type-safe
const API_URL = new InjectionToken<string>('api.url');
{ provide: API_URL, useValue: 'https://api.example.com' }

// Avoid: String tokens
{ provide: 'API_URL', useValue: 'https://api.example.com' }
```

### 4. Document Factory Functions

```typescript
/**
 * Creates a logger instance based on environment.
 * 
 * @param http - HttpClient for remote logging
 * @param config - Configuration service
 * @returns ConsoleLogger for dev, RemoteLogger for production
 */
function createLogger(http: HttpClient, config: ConfigService): Logger {
  return environment.production
    ? new RemoteLogger(http, config.get('logEndpoint'))
    : new ConsoleLogger();
}
```

## Common Mistakes

### 1. Using useValue for Classes

```typescript
// Wrong: useValue with class
{ provide: Logger, useValue: Logger }  // Error!

// Correct: useClass
{ provide: Logger, useClass: Logger }
```

### 2. Forgetting deps in useFactory

```typescript
// Wrong: Dependencies not specified
{
  provide: ApiService,
  useFactory: (http) => new ApiService(http)  // Where does http come from?
}

// Correct: Specify dependencies
{
  provide: ApiService,
  useFactory: (http: HttpClient) => new ApiService(http),
  deps: [HttpClient]
}
```

### 3. useExisting vs useClass Confusion

```typescript
// useClass creates new instance
{
  provide: LoggerAlias,
  useClass: Logger  // New instance!
}

// useExisting references existing singleton
{
  provide: LoggerAlias,
  useExisting: Logger  // Same instance as Logger
}
```

### 4. Mutable useValue Objects

```typescript
// Wrong: Mutable configuration
const config = { apiUrl: 'http://localhost' };
{ provide: CONFIG, useValue: config }

// Later: config.apiUrl = 'http://production' // Mutates shared object!

// Better: Freeze object
const config = Object.freeze({ apiUrl: 'http://localhost' });
{ provide: CONFIG, useValue: config }
```

## Interview Questions

### Basic Questions

**Q1: What are the four main provider types in Angular?**

A: useClass (provides a class instance), useValue (provides a value), useExisting (creates an alias), and useFactory (uses a factory function).

**Q2: When should you use useValue instead of useClass?**

A: Use useValue for static configuration, constants, pre-existing objects, or when you don't want Angular to instantiate a class.

**Q3: What is the difference between useClass and useExisting?**

A: useClass creates a new instance, while useExisting creates an alias to an existing provider, sharing the same instance.

### Intermediate Questions

**Q4: How do you specify dependencies for a useFactory provider?**

A: Use the deps array to specify which services should be injected into the factory function: `{ useFactory: factoryFn, deps: [Service1, Service2] }`.

**Q5: Can you use multiple provider types for the same token?**

A: No, the last provider wins unless you use multi: true for multi-providers. Each token can only have one provider type at a time.

**Q6: When would you use useFactory instead of useClass?**

A: Use useFactory when you need conditional logic, complex initialization, dependencies that aren't services, or runtime configuration.

### Advanced Questions

**Q7: How do providers interact with the injector hierarchy?**

A: Providers are registered at specific injector levels (platform, root, module, component). The injector searches up the hierarchy to find providers, with child injectors overriding parent providers.

**Q8: What are the performance implications of different provider types?**

A: useValue has the lowest overhead (single object). useClass and useFactory create instances (higher memory). useExisting has low overhead (reference to existing instance).

**Q9: How would you migrate from one service to another using providers?**

A: Use useExisting to alias the old service name to the new service, allowing gradual migration: `{ provide: OldService, useExisting: NewService }`.

**Q10: Can factory functions be async?**

A: No, factory functions are synchronous. For async initialization, use APP_INITIALIZER with a factory that returns a Promise, or handle async operations inside the created service.

## Resources

### Official Documentation
- [Dependency Injection Providers](https://angular.dev/guide/di/dependency-injection-providers)
- [DI in Action](https://angular.dev/guide/di/di-in-action)

### Articles
- [Provider Types Explained](https://blog.angular.io/provider-types)
- [Advanced DI Patterns](https://medium.com/angular-in-depth/provider-patterns)

### Videos
- [Understanding Angular Providers](https://www.youtube.com/watch?v=providers-guide)

### Books
- "Angular Development with TypeScript" - DI Chapter
- "ng-book: The Complete Guide to Angular" - Providers Section
