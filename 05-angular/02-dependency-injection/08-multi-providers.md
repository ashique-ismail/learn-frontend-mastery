# Multi-Providers in Angular

## The Idea

**In plain English:** Multi-providers let you register many different services or values under one shared label (called a token), and Angular collects all of them into a list so every registered item gets used. Instead of the last one overwriting the others, they all show up together.

**Real-world analogy:** Think of a school talent show where students sign up on a shared sheet under the label "Acts for Tonight." Every student who writes their name is added to the list — the organiser does not cross out the previous names. On show night, every act on the list performs.

- The label "Acts for Tonight" = the injection token (the shared key all providers register under)
- Each student signing up = a provider registering with `multi: true`
- The full signup sheet = the array Angular injects into the consumer
- The show organiser reading the list = the service or component that receives and runs all the collected values

---

## Table of Contents

- [Introduction](#introduction)
- [Understanding Multi-Providers](#understanding-multi-providers)
- [Built-in Multi-Providers](#built-in-multi-providers)
- [Creating Custom Multi-Providers](#creating-custom-multi-providers)
- [Use Cases and Patterns](#use-cases-and-patterns)
- [Practical Examples](#practical-examples)
- [Best Practices](#best-practices)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Resources](#resources)

## Introduction

Multi-providers allow you to register multiple values for the same injection token. Unlike regular providers where the last one wins, multi-providers accumulate values in an array. This is essential for extensible systems like HTTP interceptors, validators, and plugin architectures.

Multi-providers enable a plugin-style architecture where multiple parts of your application can contribute to a single concern without knowing about each other.

## Understanding Multi-Providers

### Regular Provider Behavior

```typescript
// Regular providers - last one wins
@NgModule({
  providers: [
    { provide: API_URL, useValue: 'http://dev.api.com' },
    { provide: API_URL, useValue: 'http://prod.api.com' }  // This one is used
  ]
})
```

### Multi-Provider Behavior

```typescript
// Multi-providers - all values collected in array
@NgModule({
  providers: [
    { provide: PLUGINS, useValue: 'Plugin A', multi: true },
    { provide: PLUGINS, useValue: 'Plugin B', multi: true }  // Both are used
  ]
})

// Injected as array
constructor(@Inject(PLUGINS) plugins: string[]) {
  console.log(plugins);  // ['Plugin A', 'Plugin B']
}
```

## Built-in Multi-Providers

### HTTP_INTERCEPTORS

The most common built-in multi-provider:

```typescript
import { HTTP_INTERCEPTORS, HttpInterceptor } from '@angular/common/http';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler) {
    const authReq = req.clone({
      headers: req.headers.set('Authorization', 'Bearer token')
    });
    return next.handle(authReq);
  }
}

@Injectable()
export class LoggingInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler) {
    console.log('Request:', req.url);
    return next.handle(req);
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
      multi: true  // Both interceptors will be used
    }
  ]
})
```

### APP_INITIALIZER

For application initialization:

```typescript
import { APP_INITIALIZER } from '@angular/core';

export function initConfig(config: ConfigService): () => Promise<void> {
  return () => config.load();
}

export function initAuth(auth: AuthService): () => Promise<void> {
  return () => auth.initialize();
}

@NgModule({
  providers: [
    {
      provide: APP_INITIALIZER,
      useFactory: initConfig,
      deps: [ConfigService],
      multi: true
    },
    {
      provide: APP_INITIALIZER,
      useFactory: initAuth,
      deps: [AuthService],
      multi: true  // Both initializers run
    }
  ]
})
```

### NG_VALIDATORS

For custom form validators:

```typescript
import { NG_VALIDATORS, Validator, AbstractControl } from '@angular/forms';

@Directive({
  selector: '[appEmailValidator]',
  providers: [
    {
      provide: NG_VALIDATORS,
      useExisting: EmailValidatorDirective,
      multi: true
    }
  ]
})
export class EmailValidatorDirective implements Validator {
  validate(control: AbstractControl): ValidationErrors | null {
    const email = control.value;
    if (!email) return null;
    
    const valid = /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
    return valid ? null : { email: true };
  }
}
```

### NG_VALUE_ACCESSOR

For custom form controls:

```typescript
import { NG_VALUE_ACCESSOR, ControlValueAccessor } from '@angular/forms';

@Component({
  selector: 'app-custom-input',
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => CustomInputComponent),
      multi: true
    }
  ]
})
export class CustomInputComponent implements ControlValueAccessor {
  // Implementation
}
```

## Creating Custom Multi-Providers

### Basic Custom Multi-Provider

```typescript
// Define token
export interface Plugin {
  name: string;
  execute(): void;
}

export const APP_PLUGINS = new InjectionToken<Plugin[]>('app.plugins');

// Create plugins
@Injectable()
export class AnalyticsPlugin implements Plugin {
  name = 'Analytics';
  
  execute(): void {
    console.log('Analytics plugin initialized');
  }
}

@Injectable()
export class LoggingPlugin implements Plugin {
  name = 'Logging';
  
  execute(): void {
    console.log('Logging plugin initialized');
  }
}

// Register plugins
@NgModule({
  providers: [
    {
      provide: APP_PLUGINS,
      useClass: AnalyticsPlugin,
      multi: true
    },
    {
      provide: APP_PLUGINS,
      useClass: LoggingPlugin,
      multi: true
    }
  ]
})

// Use plugins
@Injectable({ providedIn: 'root' })
export class PluginManager {
  constructor(@Inject(APP_PLUGINS) private plugins: Plugin[]) {
    this.initializePlugins();
  }
  
  private initializePlugins(): void {
    this.plugins.forEach(plugin => {
      console.log(`Initializing ${plugin.name}`);
      plugin.execute();
    });
  }
}
```

### Multi-Provider with Values

```typescript
export const ENVIRONMENT_VALIDATORS = new InjectionToken<Array<(env: any) => boolean>>(
  'environment.validators'
);

// Define validators
function hasApiUrl(env: any): boolean {
  return !!env.apiUrl;
}

function hasValidTimeout(env: any): boolean {
  return env.timeout > 0 && env.timeout < 60000;
}

// Register validators
@NgModule({
  providers: [
    {
      provide: ENVIRONMENT_VALIDATORS,
      useValue: hasApiUrl,
      multi: true
    },
    {
      provide: ENVIRONMENT_VALIDATORS,
      useValue: hasValidTimeout,
      multi: true
    }
  ]
})

// Use validators
@Injectable({ providedIn: 'root' })
export class EnvironmentValidator {
  constructor(
    @Inject(ENVIRONMENT_VALIDATORS) 
    private validators: Array<(env: any) => boolean>
  ) {}
  
  validate(env: any): boolean {
    return this.validators.every(validator => validator(env));
  }
}
```

## Use Cases and Patterns

### Plugin Architecture

```typescript
// plugin.interface.ts
export interface AppPlugin {
  readonly name: string;
  readonly version: string;
  initialize(): void;
  destroy?(): void;
}

export const APP_PLUGINS = new InjectionToken<AppPlugin[]>('app.plugins');

// notification.plugin.ts
@Injectable()
export class NotificationPlugin implements AppPlugin {
  name = 'Notification Plugin';
  version = '1.0.0';
  
  constructor(private notificationService: NotificationService) {}
  
  initialize(): void {
    this.notificationService.initialize();
    console.log(`${this.name} initialized`);
  }
  
  destroy(): void {
    this.notificationService.cleanup();
  }
}

// analytics.plugin.ts
@Injectable()
export class AnalyticsPlugin implements AppPlugin {
  name = 'Analytics Plugin';
  version = '2.1.0';
  
  constructor(private analytics: AnalyticsService) {}
  
  initialize(): void {
    this.analytics.setup();
    console.log(`${this.name} initialized`);
  }
}

// plugin-manager.service.ts
@Injectable({ providedIn: 'root' })
export class PluginManager implements OnDestroy {
  constructor(@Inject(APP_PLUGINS) private plugins: AppPlugin[]) {
    this.initializeAll();
  }
  
  private initializeAll(): void {
    console.log(`Initializing ${this.plugins.length} plugins...`);
    this.plugins.forEach(plugin => {
      try {
        plugin.initialize();
      } catch (error) {
        console.error(`Failed to initialize ${plugin.name}:`, error);
      }
    });
  }
  
  ngOnDestroy(): void {
    this.plugins.forEach(plugin => {
      if (plugin.destroy) {
        try {
          plugin.destroy();
        } catch (error) {
          console.error(`Failed to destroy ${plugin.name}:`, error);
        }
      }
    });
  }
  
  getPlugin(name: string): AppPlugin | undefined {
    return this.plugins.find(p => p.name === name);
  }
  
  getAllPlugins(): AppPlugin[] {
    return [...this.plugins];
  }
}

// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    {
      provide: APP_PLUGINS,
      useClass: NotificationPlugin,
      multi: true
    },
    {
      provide: APP_PLUGINS,
      useClass: AnalyticsPlugin,
      multi: true
    }
  ]
};
```

### Event Listeners

```typescript
// event-listener.interface.ts
export interface EventListener<T = any> {
  handle(event: T): void;
}

export const USER_LOGIN_LISTENERS = new InjectionToken<EventListener[]>(
  'user.login.listeners'
);

// analytics-listener.ts
@Injectable()
export class AnalyticsLoginListener implements EventListener<User> {
  constructor(private analytics: AnalyticsService) {}
  
  handle(user: User): void {
    this.analytics.trackEvent('user_login', {
      userId: user.id,
      timestamp: new Date()
    });
  }
}

// notification-listener.ts
@Injectable()
export class NotificationLoginListener implements EventListener<User> {
  constructor(private notifications: NotificationService) {}
  
  handle(user: User): void {
    this.notifications.show(`Welcome back, ${user.name}!`);
  }
}

// event-dispatcher.service.ts
@Injectable({ providedIn: 'root' })
export class EventDispatcher {
  constructor(
    @Inject(USER_LOGIN_LISTENERS) 
    private loginListeners: EventListener<User>[]
  ) {}
  
  dispatchUserLogin(user: User): void {
    this.loginListeners.forEach(listener => {
      try {
        listener.handle(user);
      } catch (error) {
        console.error('Event listener failed:', error);
      }
    });
  }
}

// Register listeners
@NgModule({
  providers: [
    {
      provide: USER_LOGIN_LISTENERS,
      useClass: AnalyticsLoginListener,
      multi: true
    },
    {
      provide: USER_LOGIN_LISTENERS,
      useClass: NotificationLoginListener,
      multi: true
    }
  ]
})
```

### Route Guards Chain

```typescript
export const ADMIN_GUARDS = new InjectionToken<CanActivateFn[]>('admin.guards');

// Auth guard
export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  return auth.isAuthenticated();
};

// Role guard
export const roleGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  return auth.hasRole('admin');
};

// Feature flag guard
export const featureFlagGuard: CanActivateFn = () => {
  const features = inject(FeatureService);
  return features.isEnabled('admin-panel');
};

// Composite guard
export const compositeAdminGuard: CanActivateFn = () => {
  const guards = inject(ADMIN_GUARDS);
  
  for (const guard of guards) {
    const result = guard();
    if (!result) {
      return false;
    }
  }
  
  return true;
};

// Configuration
bootstrapApplication(AppComponent, {
  providers: [
    {
      provide: ADMIN_GUARDS,
      useValue: authGuard,
      multi: true
    },
    {
      provide: ADMIN_GUARDS,
      useValue: roleGuard,
      multi: true
    },
    {
      provide: ADMIN_GUARDS,
      useValue: featureFlagGuard,
      multi: true
    }
  ]
});

// Routes
const routes: Routes = [
  {
    path: 'admin',
    canActivate: [compositeAdminGuard],
    loadChildren: () => import('./admin/admin.routes')
  }
];
```

### Data Transformers Pipeline

```typescript
export interface DataTransformer<T = any> {
  transform(data: T): T;
}

export const DATA_TRANSFORMERS = new InjectionToken<DataTransformer[]>(
  'data.transformers'
);

// Trim transformer
@Injectable()
export class TrimTransformer implements DataTransformer<any> {
  transform(data: any): any {
    if (typeof data === 'string') {
      return data.trim();
    }
    if (typeof data === 'object') {
      return Object.keys(data).reduce((acc, key) => {
        acc[key] = this.transform(data[key]);
        return acc;
      }, {} as any);
    }
    return data;
  }
}

// Lowercase transformer
@Injectable()
export class LowercaseTransformer implements DataTransformer<any> {
  transform(data: any): any {
    if (typeof data === 'string') {
      return data.toLowerCase();
    }
    if (typeof data === 'object') {
      return Object.keys(data).reduce((acc, key) => {
        acc[key] = this.transform(data[key]);
        return acc;
      }, {} as any);
    }
    return data;
  }
}

// Transformer pipeline
@Injectable({ providedIn: 'root' })
export class DataPipeline {
  constructor(
    @Inject(DATA_TRANSFORMERS) 
    private transformers: DataTransformer[]
  ) {}
  
  process<T>(data: T): T {
    return this.transformers.reduce(
      (result, transformer) => transformer.transform(result),
      data
    );
  }
}

// Register transformers
@NgModule({
  providers: [
    {
      provide: DATA_TRANSFORMERS,
      useClass: TrimTransformer,
      multi: true
    },
    {
      provide: DATA_TRANSFORMERS,
      useClass: LowercaseTransformer,
      multi: true
    }
  ]
})
```

## Practical Examples

### Example 1: Middleware System

```typescript
// middleware.interface.ts
export interface Middleware<T = any> {
  execute(context: T, next: () => void): void;
}

export const REQUEST_MIDDLEWARE = new InjectionToken<Middleware[]>(
  'request.middleware'
);

// logging.middleware.ts
@Injectable()
export class LoggingMiddleware implements Middleware {
  execute(context: any, next: () => void): void {
    console.log('Before request:', context);
    next();
    console.log('After request:', context);
  }
}

// auth.middleware.ts
@Injectable()
export class AuthMiddleware implements Middleware {
  constructor(private auth: AuthService) {}
  
  execute(context: any, next: () => void): void {
    if (!this.auth.isAuthenticated()) {
      throw new Error('Unauthorized');
    }
    context.userId = this.auth.getUserId();
    next();
  }
}

// validation.middleware.ts
@Injectable()
export class ValidationMiddleware implements Middleware {
  execute(context: any, next: () => void): void {
    if (!this.validate(context)) {
      throw new Error('Validation failed');
    }
    next();
  }
  
  private validate(context: any): boolean {
    // Validation logic
    return true;
  }
}

// middleware.service.ts
@Injectable({ providedIn: 'root' })
export class MiddlewareService {
  constructor(
    @Inject(REQUEST_MIDDLEWARE) 
    private middleware: Middleware[]
  ) {}
  
  async execute(context: any): Promise<void> {
    let index = 0;
    
    const next = () => {
      if (index < this.middleware.length) {
        const current = this.middleware[index++];
        current.execute(context, next);
      }
    };
    
    next();
  }
}

// Registration
bootstrapApplication(AppComponent, {
  providers: [
    {
      provide: REQUEST_MIDDLEWARE,
      useClass: LoggingMiddleware,
      multi: true
    },
    {
      provide: REQUEST_MIDDLEWARE,
      useClass: AuthMiddleware,
      multi: true
    },
    {
      provide: REQUEST_MIDDLEWARE,
      useClass: ValidationMiddleware,
      multi: true
    }
  ]
});
```

### Example 2: Validation Rules System

```typescript
// validation-rule.interface.ts
export interface ValidationRule {
  name: string;
  message: string;
  validate(value: any, context?: any): boolean;
}

export const FORM_VALIDATION_RULES = new InjectionToken<ValidationRule[]>(
  'form.validation.rules'
);

// required.rule.ts
export const requiredRule: ValidationRule = {
  name: 'required',
  message: 'This field is required',
  validate(value: any): boolean {
    return value != null && value !== '';
  }
};

// email.rule.ts
export const emailRule: ValidationRule = {
  name: 'email',
  message: 'Invalid email format',
  validate(value: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value);
  }
};

// min-length.rule.ts
export const minLengthRule: ValidationRule = {
  name: 'minLength',
  message: 'Value is too short',
  validate(value: string, context: { minLength: number }): boolean {
    return value && value.length >= context.minLength;
  }
};

// validator.service.ts
@Injectable({ providedIn: 'root' })
export class ValidatorService {
  private rulesMap: Map<string, ValidationRule>;
  
  constructor(
    @Inject(FORM_VALIDATION_RULES) 
    private rules: ValidationRule[]
  ) {
    this.rulesMap = new Map(
      rules.map(rule => [rule.name, rule])
    );
  }
  
  validate(ruleName: string, value: any, context?: any): ValidationResult {
    const rule = this.rulesMap.get(ruleName);
    
    if (!rule) {
      throw new Error(`Validation rule '${ruleName}' not found`);
    }
    
    const valid = rule.validate(value, context);
    
    return {
      valid,
      message: valid ? '' : rule.message
    };
  }
  
  validateMultiple(
    rules: Array<{ name: string; context?: any }>,
    value: any
  ): ValidationResult[] {
    return rules.map(({ name, context }) => 
      this.validate(name, value, context)
    );
  }
}

// Registration
@NgModule({
  providers: [
    {
      provide: FORM_VALIDATION_RULES,
      useValue: requiredRule,
      multi: true
    },
    {
      provide: FORM_VALIDATION_RULES,
      useValue: emailRule,
      multi: true
    },
    {
      provide: FORM_VALIDATION_RULES,
      useValue: minLengthRule,
      multi: true
    }
  ]
})
```

## Best Practices

### 1. Always Use multi: true

```typescript
// Good: Explicitly set multi: true
{
  provide: HTTP_INTERCEPTORS,
  useClass: AuthInterceptor,
  multi: true
}

// Bad: Forgetting multi: true
{
  provide: HTTP_INTERCEPTORS,
  useClass: AuthInterceptor  // Replaces all other interceptors!
}
```

### 2. Document Multi-Provider Contracts

```typescript
/**
 * Token for registering application plugins.
 * 
 * Plugins must implement the AppPlugin interface and will be
 * initialized in registration order during app startup.
 * 
 * @example
 * {
 *   provide: APP_PLUGINS,
 *   useClass: MyPlugin,
 *   multi: true
 * }
 */
export const APP_PLUGINS = new InjectionToken<AppPlugin[]>('app.plugins');
```

### 3. Handle Errors in Multi-Provider Consumers

```typescript
// Good: Error handling for each item
@Injectable({ providedIn: 'root' })
export class PluginManager {
  constructor(@Inject(APP_PLUGINS) private plugins: Plugin[]) {
    this.plugins.forEach(plugin => {
      try {
        plugin.initialize();
      } catch (error) {
        console.error(`Failed to initialize plugin: ${plugin.name}`, error);
      }
    });
  }
}

// Bad: No error handling
@Injectable({ providedIn: 'root' })
export class PluginManager {
  constructor(@Inject(APP_PLUGINS) private plugins: Plugin[]) {
    this.plugins.forEach(plugin => plugin.initialize());  // One failure breaks all
  }
}
```

### 4. Use Interfaces for Multi-Provider Values

```typescript
// Good: Interface-based
export interface Validator {
  validate(value: any): boolean;
}

export const VALIDATORS = new InjectionToken<Validator[]>('validators');

// Bad: Any type
export const VALIDATORS = new InjectionToken<any[]>('validators');
```

## Common Mistakes

### 1. Forgetting multi: true

```typescript
// Wrong: Both registered without multi: true
{
  provide: HTTP_INTERCEPTORS,
  useClass: AuthInterceptor
},
{
  provide: HTTP_INTERCEPTORS,
  useClass: LoggingInterceptor  // This replaces AuthInterceptor!
}

// Correct: Use multi: true
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

### 2. Mixing multi and non-multi

```typescript
// Wrong: Mixing multi and non-multi for same token
{
  provide: PLUGINS,
  useClass: PluginA,
  multi: true
},
{
  provide: PLUGINS,
  useClass: PluginB  // Missing multi: true - causes error!
}

// Correct: Consistent multi: true
{
  provide: PLUGINS,
  useClass: PluginA,
  multi: true
},
{
  provide: PLUGINS,
  useClass: PluginB,
  multi: true
}
```

### 3. Assuming Order

```typescript
// Don't rely on execution order
@Injectable({ providedIn: 'root' })
export class Manager {
  constructor(@Inject(PLUGINS) private plugins: Plugin[]) {
    // Order may not be guaranteed across modules
    this.plugins[0].initialize();  // Risky!
  }
}

// Better: Treat as unordered set
@Injectable({ providedIn: 'root' })
export class Manager {
  constructor(@Inject(PLUGINS) private plugins: Plugin[]) {
    this.plugins.forEach(plugin => plugin.initialize());
  }
}
```

### 4. Not Handling Empty Arrays

```typescript
// Wrong: Assumes plugins exist
constructor(@Inject(PLUGINS) private plugins: Plugin[]) {
  this.plugins.forEach(plugin => plugin.initialize());  // What if empty?
}

// Correct: Check for empty
constructor(@Inject(PLUGINS) private plugins: Plugin[]) {
  if (this.plugins.length > 0) {
    this.plugins.forEach(plugin => plugin.initialize());
  }
}
```

## Interview Questions

### Basic Questions

**Q1: What is a multi-provider in Angular?**

A: A multi-provider allows multiple values to be registered for the same injection token. With multi: true, values are collected in an array instead of replacing each other.

**Q2: What built-in Angular tokens use multi-providers?**

A: HTTP_INTERCEPTORS, APP_INITIALIZER, NG_VALIDATORS, and NG_VALUE_ACCESSOR are common built-in multi-providers.

**Q3: What happens if you forget multi: true?**

A: Without multi: true, each provider replaces the previous one, and only the last registered value will be available.

### Intermediate Questions

**Q4: How do you inject multi-provider values?**

A: Multi-provider values are injected as an array. Use @Inject() decorator with the token: `constructor(@Inject(TOKEN) values: Type[])`.

**Q5: Can you mix different provider types with multi-providers?**

A: Yes, you can use useClass, useValue, useFactory, or useExisting with multi: true. All values are collected in an array.

**Q6: What are common use cases for custom multi-providers?**

A: Plugin systems, event listeners, middleware chains, validation rules, data transformers, and any extensible system where multiple components contribute to a single concern.

### Advanced Questions

**Q7: How does the order of multi-providers work?**

A: Multi-providers are collected in registration order within a module, but cross-module order isn't guaranteed. Don't rely on specific ordering.

**Q8: How would you implement a plugin system using multi-providers?**

A: Create an interface for plugins, define a multi-provider token, register plugins with multi: true, inject the array, and manage plugin lifecycle in a service.

**Q9: What are the performance implications of multi-providers?**

A: All registered providers are instantiated (for useClass) or evaluated (for useFactory) when the token is first injected. Consider lazy initialization if plugins are expensive.

**Q10: How do you handle optional multi-providers?**

A: Use @Optional() decorator: `@Optional() @Inject(TOKEN) values: Type[] | null`. This allows the injection to succeed even if no providers are registered.

## Resources

### Official Documentation

- [Multi-Providers](https://angular.dev/guide/di/dependency-injection-providers#multi-providers)
- [HTTP Interceptors](https://angular.dev/guide/http/interceptors)

### Articles

- [Understanding Multi-Providers](https://blog.angular.io/multi-providers-explained)
- [Plugin Architecture with DI](https://medium.com/angular-in-depth/plugin-architecture)

### Videos

- [Advanced DI Patterns](https://www.youtube.com/watch?v=advanced-di)
- [Multi-Providers Deep Dive](https://www.youtube.com/watch?v=multi-providers)

### Community

- [Angular Discord](https://discord.gg/angular)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/angular-di)
