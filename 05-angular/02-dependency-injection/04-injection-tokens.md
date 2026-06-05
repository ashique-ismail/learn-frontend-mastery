# Injection Tokens in Angular

## The Idea

**In plain English:** An injection token is a unique name tag you create so Angular can find and hand over a specific value (like a setting or configuration) to any part of your app that asks for it. Think of it as a labeled locker key — the label is the token, and the locker holds the value.

**Real-world analogy:** Imagine a hotel where guests can request specific amenities by calling the front desk with a room-service code. You dial code "42" to get extra towels — not "towels" as a word, because another guest might also say "towels" and mean something different. The unique numeric code removes all ambiguity.

- The room-service code (e.g., 42) = the InjectionToken (a unique identifier)
- The amenity delivered (extra towels) = the value or service provided to your component
- The front desk matching code to amenity = Angular's dependency injection system resolving the token to the correct provider

---

## Table of Contents

- [Introduction](#introduction)
- [Understanding Injection Tokens](#understanding-injection-tokens)
- [Built-in Injection Tokens](#built-in-injection-tokens)
- [Creating Custom Injection Tokens](#creating-custom-injection-tokens)
- [InjectionToken Class](#injectiontoken-class)
- [Token Types](#token-types)
- [Multi-Provider Tokens](#multi-provider-tokens)
- [Practical Examples](#practical-examples)
- [Best Practices](#best-practices)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Resources](#resources)

## Introduction

Injection tokens are unique identifiers used by Angular's dependency injection system to map dependencies to their providers. While classes can serve as their own tokens, InjectionToken provides a way to create tokens for non-class dependencies like primitives, functions, and configuration objects.

Understanding injection tokens is essential for creating flexible, type-safe, and maintainable Angular applications, especially when dealing with configuration, environment-specific values, and interfaces.

## Understanding Injection Tokens

### What is a Token?

A token is a unique identifier that the DI system uses to locate providers:

```typescript
// Class as token (most common)
@Injectable({ providedIn: 'root' })
export class UserService {}

// Class UserService is both the service and its token
constructor(private userService: UserService) {}

// Non-class values need explicit tokens
const API_URL = new InjectionToken<string>('api.url');

constructor(@Inject(API_URL) private apiUrl: string) {}
```

### Why Use Injection Tokens?

```typescript
// Problem: Can't use primitives as tokens
@NgModule({
  providers: [
    { provide: string, useValue: 'https://api.example.com' } // Error!
  ]
})

// Solution: Use InjectionToken
const API_URL = new InjectionToken<string>('api.url');

@NgModule({
  providers: [
    { provide: API_URL, useValue: 'https://api.example.com' } // Works!
  ]
})
```

## Built-in Injection Tokens

Angular provides many built-in injection tokens:

### Platform and Environment

```typescript
import { PLATFORM_ID, isPlatformBrowser } from '@angular/common';
import { DOCUMENT } from '@angular/common';
import { Inject } from '@angular/core';

@Component({
  selector: 'app-platform-aware'
})
export class PlatformAwareComponent {
  constructor(
    @Inject(PLATFORM_ID) private platformId: Object,
    @Inject(DOCUMENT) private document: Document
  ) {
    if (isPlatformBrowser(this.platformId)) {
      console.log('Running in browser');
      this.document.title = 'My App';
    }
  }
}
```

### Locale and Internationalization

```typescript
import { LOCALE_ID } from '@angular/core';

@Component({
  selector: 'app-localized'
})
export class LocalizedComponent {
  constructor(@Inject(LOCALE_ID) private locale: string) {
    console.log('Current locale:', locale); // e.g., 'en-US'
  }
}

// Configure in providers
@NgModule({
  providers: [
    { provide: LOCALE_ID, useValue: 'fr-FR' }
  ]
})
export class AppModule {}
```

### APP_INITIALIZER

```typescript
import { APP_INITIALIZER } from '@angular/core';

export function initializeApp(config: ConfigService): () => Promise<void> {
  return () => config.load();
}

@NgModule({
  providers: [
    {
      provide: APP_INITIALIZER,
      useFactory: initializeApp,
      deps: [ConfigService],
      multi: true  // Important: allows multiple initializers
    }
  ]
})
export class AppModule {}
```

### APP_BASE_HREF

```typescript
import { APP_BASE_HREF } from '@angular/common';

// Set base href programmatically
@NgModule({
  providers: [
    { provide: APP_BASE_HREF, useValue: '/my-app/' }
  ]
})
export class AppModule {}
```

## Creating Custom Injection Tokens

### Basic Token Creation

```typescript
import { InjectionToken } from '@angular/core';

// Create token with type safety
export const API_URL = new InjectionToken<string>('api.url');

// Provide value
@NgModule({
  providers: [
    { provide: API_URL, useValue: 'https://api.example.com' }
  ]
})
export class AppModule {}

// Inject in component
@Component({
  selector: 'app-api-client'
})
export class ApiClientComponent {
  constructor(@Inject(API_URL) private apiUrl: string) {
    console.log('API URL:', apiUrl);
  }
}
```

### Token with Default Value

```typescript
export const API_TIMEOUT = new InjectionToken<number>(
  'api.timeout',
  {
    providedIn: 'root',
    factory: () => 30000  // Default 30 seconds
  }
);

// Use default or override
@Component({
  selector: 'app-http-client'
})
export class HttpClientComponent {
  constructor(@Inject(API_TIMEOUT) private timeout: number) {
    console.log('Timeout:', timeout); // 30000
  }
}

// Override in specific module
@NgModule({
  providers: [
    { provide: API_TIMEOUT, useValue: 60000 }  // 60 seconds for this module
  ]
})
export class SlowFeatureModule {}
```

### Token for Configuration Objects

```typescript
export interface AppConfig {
  apiUrl: string;
  apiKey: string;
  environment: 'development' | 'production';
  features: {
    enableAnalytics: boolean;
    enableChat: boolean;
  };
}

export const APP_CONFIG = new InjectionToken<AppConfig>('app.config', {
  providedIn: 'root',
  factory: () => ({
    apiUrl: 'https://api.example.com',
    apiKey: 'default-key',
    environment: 'development',
    features: {
      enableAnalytics: false,
      enableChat: false
    }
  })
});

// Usage
@Injectable({ providedIn: 'root' })
export class ApiService {
  constructor(@Inject(APP_CONFIG) private config: AppConfig) {}
  
  makeRequest(endpoint: string): Observable<any> {
    return this.http.get(`${this.config.apiUrl}${endpoint}`, {
      headers: { 'X-API-Key': this.config.apiKey }
    });
  }
}
```

## InjectionToken Class

### Basic Syntax

```typescript
class InjectionToken<T> {
  constructor(
    protected _desc: string,
    options?: {
      providedIn?: Type<any> | 'root' | 'platform' | 'any' | null;
      factory: () => T;
    }
  )
}
```

### With Factory Function

```typescript
export const WINDOW = new InjectionToken<Window>(
  'window',
  {
    providedIn: 'root',
    factory: () => window
  }
);

// Safe for SSR
export const SAFE_WINDOW = new InjectionToken<Window | null>(
  'safe.window',
  {
    providedIn: 'root',
    factory: () => {
      if (typeof window !== 'undefined') {
        return window;
      }
      return null;
    }
  }
);
```

### With Dependencies

```typescript
export const CURRENT_USER = new InjectionToken<Observable<User | null>>(
  'current.user',
  {
    providedIn: 'root',
    factory: () => {
      const authService = inject(AuthService);
      return authService.currentUser$;
    }
  }
);

// Usage
@Component({
  selector: 'app-header'
})
export class HeaderComponent {
  user$ = inject(CURRENT_USER);
}
```

## Token Types

### String Tokens (Legacy - Not Recommended)

```typescript
// Old way - avoid this
@Injectable()
export class MyService {
  constructor(@Inject('API_URL') private apiUrl: string) {}
}

// No type safety!
// Provider:
{ provide: 'API_URL', useValue: 'https://api.example.com' }
```

### Class Tokens

```typescript
// Classes are their own tokens
@Injectable({ providedIn: 'root' })
export class UserService {
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>('/api/users');
  }
}

// No @Inject needed for class tokens
constructor(private userService: UserService) {}
```

### InjectionToken (Recommended for Non-Classes)

```typescript
// Type-safe tokens for non-class values
export const MAX_RETRIES = new InjectionToken<number>('max.retries');
export const ENABLE_LOGGING = new InjectionToken<boolean>('enable.logging');
export const FEATURE_FLAGS = new InjectionToken<string[]>('feature.flags');

// Type safety enforced
@NgModule({
  providers: [
    { provide: MAX_RETRIES, useValue: 3 },  // ✓ number
    { provide: ENABLE_LOGGING, useValue: true },  // ✓ boolean
    { provide: FEATURE_FLAGS, useValue: ['beta', 'experimental'] }  // ✓ string[]
  ]
})
```

### Abstract Class Tokens

```typescript
// Define interface as abstract class
export abstract class Logger {
  abstract log(message: string): void;
  abstract error(message: string): void;
}

// Concrete implementations
@Injectable()
export class ConsoleLogger implements Logger {
  log(message: string): void {
    console.log(message);
  }
  
  error(message: string): void {
    console.error(message);
  }
}

@Injectable()
export class RemoteLogger implements Logger {
  log(message: string): void {
    // Send to remote service
  }
  
  error(message: string): void {
    // Send error to remote service
  }
}

// Provide based on environment
@NgModule({
  providers: [
    {
      provide: Logger,
      useClass: environment.production ? RemoteLogger : ConsoleLogger
    }
  ]
})

// Inject using abstract class
constructor(private logger: Logger) {}
```

## Multi-Provider Tokens

### HTTP Interceptors Example

```typescript
import { HTTP_INTERCEPTORS } from '@angular/common/http';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler) {
    // Add auth token
  }
}

@Injectable()
export class LoggingInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler) {
    // Log requests
  }
}

@NgModule({
  providers: [
    {
      provide: HTTP_INTERCEPTORS,
      useClass: AuthInterceptor,
      multi: true  // Important!
    },
    {
      provide: HTTP_INTERCEPTORS,
      useClass: LoggingInterceptor,
      multi: true  // Adds to array, doesn't replace
    }
  ]
})
```

### Custom Multi-Provider Token

```typescript
export interface Plugin {
  name: string;
  initialize(): void;
}

export const APP_PLUGINS = new InjectionToken<Plugin[]>('app.plugins');

@Injectable()
export class AnalyticsPlugin implements Plugin {
  name = 'analytics';
  initialize(): void {
    console.log('Analytics initialized');
  }
}

@Injectable()
export class ChatPlugin implements Plugin {
  name = 'chat';
  initialize(): void {
    console.log('Chat initialized');
  }
}

@NgModule({
  providers: [
    { provide: APP_PLUGINS, useClass: AnalyticsPlugin, multi: true },
    { provide: APP_PLUGINS, useClass: ChatPlugin, multi: true }
  ]
})

// Use all plugins
@Injectable({ providedIn: 'root' })
export class PluginManager {
  constructor(@Inject(APP_PLUGINS) private plugins: Plugin[]) {
    this.initializeAll();
  }
  
  private initializeAll(): void {
    this.plugins.forEach(plugin => plugin.initialize());
  }
}
```

### APP_INITIALIZER with Multiple Functions

```typescript
export function loadConfig(config: ConfigService): () => Promise<void> {
  return () => config.load();
}

export function loadUser(auth: AuthService): () => Promise<void> {
  return () => auth.loadCurrentUser();
}

export function initializeAnalytics(analytics: AnalyticsService): () => void {
  return () => analytics.initialize();
}

@NgModule({
  providers: [
    {
      provide: APP_INITIALIZER,
      useFactory: loadConfig,
      deps: [ConfigService],
      multi: true
    },
    {
      provide: APP_INITIALIZER,
      useFactory: loadUser,
      deps: [AuthService],
      multi: true
    },
    {
      provide: APP_INITIALIZER,
      useFactory: initializeAnalytics,
      deps: [AnalyticsService],
      multi: true
    }
  ]
})
```

## Practical Examples

### Example 1: Environment Configuration

```typescript
// config.token.ts
export interface EnvironmentConfig {
  production: boolean;
  apiUrl: string;
  wsUrl: string;
  version: string;
}

export const ENVIRONMENT_CONFIG = new InjectionToken<EnvironmentConfig>(
  'environment.config',
  {
    providedIn: 'root',
    factory: () => ({
      production: false,
      apiUrl: 'http://localhost:3000',
      wsUrl: 'ws://localhost:3000',
      version: '1.0.0'
    })
  }
);

// main.ts
bootstrapApplication(AppComponent, {
  providers: [
    {
      provide: ENVIRONMENT_CONFIG,
      useValue: {
        production: true,
        apiUrl: 'https://api.production.com',
        wsUrl: 'wss://ws.production.com',
        version: '2.1.0'
      }
    }
  ]
});

// api.service.ts
@Injectable({ providedIn: 'root' })
export class ApiService {
  private baseUrl: string;
  
  constructor(
    @Inject(ENVIRONMENT_CONFIG) private config: EnvironmentConfig,
    private http: HttpClient
  ) {
    this.baseUrl = config.apiUrl;
  }
  
  get<T>(endpoint: string): Observable<T> {
    return this.http.get<T>(`${this.baseUrl}${endpoint}`);
  }
}
```

### Example 2: Feature Flags System

```typescript
// feature-flags.token.ts
export interface FeatureFlags {
  [key: string]: boolean;
}

export const FEATURE_FLAGS = new InjectionToken<FeatureFlags>(
  'feature.flags',
  {
    providedIn: 'root',
    factory: () => ({
      newDashboard: false,
      betaFeatures: false,
      experimentalUI: false
    })
  }
);

// feature-flag.service.ts
@Injectable({ providedIn: 'root' })
export class FeatureFlagService {
  private flags: FeatureFlags;
  
  constructor(@Inject(FEATURE_FLAGS) flags: FeatureFlags) {
    this.flags = flags;
  }
  
  isEnabled(flag: string): boolean {
    return this.flags[flag] || false;
  }
  
  enable(flag: string): void {
    this.flags[flag] = true;
  }
  
  disable(flag: string): void {
    this.flags[flag] = false;
  }
  
  getAllFlags(): FeatureFlags {
    return { ...this.flags };
  }
}

// Directive to conditionally show content
@Directive({
  selector: '[featureFlag]',
  standalone: true
})
export class FeatureFlagDirective {
  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef,
    private featureFlags: FeatureFlagService
  ) {}
  
  @Input() set featureFlag(flag: string) {
    if (this.featureFlags.isEnabled(flag)) {
      this.viewContainer.createEmbeddedView(this.templateRef);
    } else {
      this.viewContainer.clear();
    }
  }
}

// Usage
@Component({
  selector: 'app-dashboard',
  template: `
    <div *featureFlag="'newDashboard'">
      <h2>New Dashboard</h2>
    </div>
    
    <div *featureFlag="'betaFeatures'">
      <p>Beta features enabled</p>
    </div>
  `
})
export class DashboardComponent {}
```

### Example 3: Logging System with Levels

```typescript
// logger.token.ts
export enum LogLevel {
  Debug = 0,
  Info = 1,
  Warn = 2,
  Error = 3
}

export interface LoggerConfig {
  level: LogLevel;
  enableConsole: boolean;
  enableRemote: boolean;
  remoteEndpoint?: string;
}

export const LOGGER_CONFIG = new InjectionToken<LoggerConfig>(
  'logger.config',
  {
    providedIn: 'root',
    factory: () => ({
      level: LogLevel.Info,
      enableConsole: true,
      enableRemote: false
    })
  }
);

// logger.service.ts
@Injectable({ providedIn: 'root' })
export class LoggerService {
  constructor(
    @Inject(LOGGER_CONFIG) private config: LoggerConfig,
    private http: HttpClient
  ) {}
  
  debug(message: string, ...args: any[]): void {
    if (this.config.level <= LogLevel.Debug) {
      this.log('DEBUG', message, args);
    }
  }
  
  info(message: string, ...args: any[]): void {
    if (this.config.level <= LogLevel.Info) {
      this.log('INFO', message, args);
    }
  }
  
  warn(message: string, ...args: any[]): void {
    if (this.config.level <= LogLevel.Warn) {
      this.log('WARN', message, args);
    }
  }
  
  error(message: string, ...args: any[]): void {
    if (this.config.level <= LogLevel.Error) {
      this.log('ERROR', message, args);
    }
  }
  
  private log(level: string, message: string, args: any[]): void {
    const logEntry = {
      timestamp: new Date().toISOString(),
      level,
      message,
      data: args
    };
    
    if (this.config.enableConsole) {
      console[level.toLowerCase() as 'log'](message, ...args);
    }
    
    if (this.config.enableRemote && this.config.remoteEndpoint) {
      this.http.post(this.config.remoteEndpoint, logEntry).subscribe();
    }
  }
}

// Configure for production
bootstrapApplication(AppComponent, {
  providers: [
    {
      provide: LOGGER_CONFIG,
      useValue: {
        level: LogLevel.Warn,
        enableConsole: false,
        enableRemote: true,
        remoteEndpoint: 'https://logs.example.com/api/logs'
      }
    }
  ]
});
```

### Example 4: Validation Rules Token

```typescript
// validation.token.ts
export interface ValidationRule {
  name: string;
  validate: (value: any) => boolean;
  errorMessage: string;
}

export const VALIDATION_RULES = new InjectionToken<ValidationRule[]>(
  'validation.rules'
);

// validation-rules.ts
export const commonValidationRules: ValidationRule[] = [
  {
    name: 'required',
    validate: (value) => value != null && value !== '',
    errorMessage: 'This field is required'
  },
  {
    name: 'email',
    validate: (value) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value),
    errorMessage: 'Invalid email format'
  },
  {
    name: 'minLength',
    validate: (value) => value && value.length >= 3,
    errorMessage: 'Must be at least 3 characters'
  }
];

// Provide rules
@NgModule({
  providers: [
    {
      provide: VALIDATION_RULES,
      useValue: commonValidationRules
    }
  ]
})

// Use in validator service
@Injectable({ providedIn: 'root' })
export class CustomValidatorService {
  constructor(
    @Inject(VALIDATION_RULES) private rules: ValidationRule[]
  ) {}
  
  getValidator(ruleName: string): ValidatorFn {
    const rule = this.rules.find(r => r.name === ruleName);
    
    if (!rule) {
      throw new Error(`Validation rule '${ruleName}' not found`);
    }
    
    return (control: AbstractControl): ValidationErrors | null => {
      const valid = rule.validate(control.value);
      return valid ? null : { [ruleName]: rule.errorMessage };
    };
  }
}
```

## Best Practices

### 1. Use Descriptive Token Names

```typescript
// Good: Clear and descriptive
export const API_BASE_URL = new InjectionToken<string>('api.base.url');
export const MAX_RETRY_ATTEMPTS = new InjectionToken<number>('max.retry.attempts');

// Bad: Vague names
export const URL = new InjectionToken<string>('url');
export const MAX = new InjectionToken<number>('max');
```

### 2. Provide Type Information

```typescript
// Good: Type-safe
export const USER_CONFIG = new InjectionToken<UserConfig>('user.config');

// Bad: Untyped (loses type safety)
export const USER_CONFIG = new InjectionToken('user.config');
```

### 3. Use Interfaces for Complex Configurations

```typescript
// Define interface first
export interface DatabaseConfig {
  host: string;
  port: number;
  database: string;
  ssl: boolean;
}

// Create token with interface
export const DATABASE_CONFIG = new InjectionToken<DatabaseConfig>('db.config');

// Type-safe provider
@NgModule({
  providers: [
    {
      provide: DATABASE_CONFIG,
      useValue: {
        host: 'localhost',
        port: 5432,
        database: 'myapp',
        ssl: false
      }
    }
  ]
})
```

### 4. Document Tokens

```typescript
/**
 * Injection token for the API base URL.
 * 
 * This should be the full URL including protocol and domain,
 * without a trailing slash.
 * 
 * @example
 * { provide: API_BASE_URL, useValue: 'https://api.example.com' }
 */
export const API_BASE_URL = new InjectionToken<string>('api.base.url');
```

### 5. Group Related Tokens

```typescript
// api-tokens.ts
export const API_BASE_URL = new InjectionToken<string>('api.base.url');
export const API_KEY = new InjectionToken<string>('api.key');
export const API_TIMEOUT = new InjectionToken<number>('api.timeout');
export const API_RETRY_ATTEMPTS = new InjectionToken<number>('api.retry.attempts');

// feature-tokens.ts
export const ENABLE_ANALYTICS = new InjectionToken<boolean>('feature.analytics');
export const ENABLE_CHAT = new InjectionToken<boolean>('feature.chat');
export const ENABLE_NOTIFICATIONS = new InjectionToken<boolean>('feature.notifications');
```

## Common Mistakes

### 1. Using String Tokens Instead of InjectionToken

```typescript
// Wrong: No type safety
{ provide: 'API_URL', useValue: 'https://api.example.com' }
constructor(@Inject('API_URL') private apiUrl: string) {}

// Correct: Type-safe with InjectionToken
const API_URL = new InjectionToken<string>('api.url');
{ provide: API_URL, useValue: 'https://api.example.com' }
constructor(@Inject(API_URL) private apiUrl: string) {}
```

### 2. Forgetting multi: true for Multi-Providers

```typescript
// Wrong: Second provider replaces first
{
  provide: HTTP_INTERCEPTORS,
  useClass: AuthInterceptor
},
{
  provide: HTTP_INTERCEPTORS,
  useClass: LoggingInterceptor  // Replaces AuthInterceptor!
}

// Correct: Both interceptors are registered
{
  provide: HTTP_INTERCEPTORS,
  useClass: AuthInterceptor,
  multi: true
},
{
  provide: HTTP_INTERCEPTORS,
  useClass: LoggingInterceptor,
  multi: true
}
```

### 3. Not Providing Default Values

```typescript
// Wrong: No default, will throw error if not provided
export const API_TIMEOUT = new InjectionToken<number>('api.timeout');

// Correct: Has sensible default
export const API_TIMEOUT = new InjectionToken<number>('api.timeout', {
  providedIn: 'root',
  factory: () => 30000
});
```

### 4. Circular Dependencies with Tokens

```typescript
// Wrong: Circular dependency
export const SERVICE_A = new InjectionToken<ServiceA>('service.a', {
  factory: () => {
    const serviceB = inject(SERVICE_B);  // Depends on B
    return new ServiceA(serviceB);
  }
});

export const SERVICE_B = new InjectionToken<ServiceB>('service.b', {
  factory: () => {
    const serviceA = inject(SERVICE_A);  // Depends on A - circular!
    return new ServiceB(serviceA);
  }
});

// Correct: Break circular dependency
export const SERVICE_A = new InjectionToken<ServiceA>('service.a', {
  factory: () => new ServiceA()
});

export const SERVICE_B = new InjectionToken<ServiceB>('service.b', {
  factory: () => {
    const serviceA = inject(SERVICE_A);
    return new ServiceB(serviceA);
  }
});
```

## Interview Questions

### Basic Questions

**Q1: What is an InjectionToken in Angular?**

A: An InjectionToken is a unique identifier used to provide and inject non-class dependencies like primitives, objects, or functions in a type-safe way.

**Q2: Why can't you use primitive values directly as tokens?**

A: Primitive values like strings or numbers aren't unique identifiers. Multiple providers could use the same primitive value, causing conflicts. InjectionToken creates a unique token object.

**Q3: What is the difference between a class token and an InjectionToken?**

A: Class tokens use the class itself as the identifier and are automatically type-safe. InjectionToken is used for non-class values and requires explicit type parameters for type safety.

### Intermediate Questions

**Q4: What does the multi: true option do in providers?**

A: multi: true allows multiple providers for the same token, creating an array of values. Without it, later providers replace earlier ones. It's commonly used for HTTP_INTERCEPTORS and APP_INITIALIZER.

**Q5: How do you provide a default value for an InjectionToken?**

A: Use the factory option in the token's configuration: `new InjectionToken('token', { providedIn: 'root', factory: () => defaultValue })`.

**Q6: Can InjectionToken be tree-shaken?**

A: Yes, when using providedIn in the token configuration, tokens can be tree-shaken if not used, similar to services with providedIn: 'root'.

### Advanced Questions

**Q7: How would you create a token for an interface?**

A: Define an InjectionToken with the interface as the type parameter: `const MY_TOKEN = new InjectionToken<MyInterface>('my.token')`. Then provide a concrete implementation using the token.

**Q8: What are the use cases for abstract class tokens vs InjectionToken?**

A: Abstract class tokens work well for polymorphic services with multiple implementations. InjectionToken is better for configuration objects, primitives, or when you need more control over the token identity.

**Q9: How do you handle optional injection tokens?**

A: Use @Optional() decorator or inject function with optional flag: `inject(MY_TOKEN, { optional: true })`. This returns undefined if the token isn't provided instead of throwing an error.

**Q10: How would you create a hierarchical configuration system using tokens?**

A: Create tokens that check for parent injector values using @SkipSelf() and merge them with local values, implementing a cascading configuration pattern where child components can override parent values.

## Resources

### Official Documentation

- [InjectionToken API](https://angular.dev/api/core/InjectionToken)
- [Dependency Injection Guide](https://angular.dev/guide/di)

### Articles and Tutorials

- [Understanding Injection Tokens](https://blog.angular.io/injection-tokens-explained)
- [Advanced DI Patterns](https://angular.io/guide/dependency-injection-providers)

### Video Tutorials

- [Injection Tokens Deep Dive](https://www.youtube.com/watch?v=injection-tokens)
- [Custom Tokens in Angular](https://www.youtube.com/watch?v=custom-tokens)

### Community Resources

- [Angular Discord](https://discord.gg/angular)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/angular-di)
