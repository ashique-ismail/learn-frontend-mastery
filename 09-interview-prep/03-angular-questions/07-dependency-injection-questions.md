# Angular Dependency Injection Interview Questions

## The Idea

**In plain English:** Dependency Injection is a way of giving a piece of code the tools it needs to do its job, instead of making it go fetch those tools itself. Think of a "dependency" as anything a class relies on to work, and "injection" as the act of handing that thing over from the outside.

**Real-world analogy:** Imagine a chef at a restaurant. Instead of growing their own vegetables, raising animals, and milling their own flour, the chef simply opens the kitchen door and the ingredients are delivered by a supplier each morning.

- The chef = the Angular component or service that needs something to function
- The ingredients = the dependencies (other services, config values, etc.)
- The supplier delivering ingredients = Angular's injector, which creates and hands over the right dependency
- The restaurant's ordering system = the provider configuration, which tells the injector what to deliver and how to prepare it

---

## Table of Contents
- [Core Concepts](#core-concepts)
- [Common Interview Questions](#common-interview-questions)
- [Advanced Questions](#advanced-questions)
- [What Interviewers Look For](#what-interviewers-look-for)
- [Red Flags to Avoid](#red-flags-to-avoid)
- [Key Takeaways](#key-takeaways)

## Core Concepts

Dependency Injection (DI) is a fundamental design pattern in Angular that allows components and services to declare their dependencies rather than creating them. Angular's DI system is one of its most powerful features, enabling modularity, testability, and reusability.

### DI Fundamentals

**Key Principles:**
- Dependencies are injected, not created
- Promotes loose coupling
- Facilitates testing through mocking
- Supports hierarchical dependency trees
- Enables singleton and scoped instances

### Provider Types

Angular offers multiple ways to provide dependencies:
- **useClass**: Provide a class
- **useValue**: Provide a static value
- **useFactory**: Use a factory function
- **useExisting**: Alias an existing provider

## Common Interview Questions

### Q1: Explain Angular's Dependency Injection system and how it works.

**What the interviewer wants to know:**
- Understanding of core DI concepts
- Knowledge of how Angular resolves dependencies
- Awareness of the injector hierarchy

**Strong Answer:**

Angular's Dependency Injection is a design pattern where a class receives its dependencies from external sources rather than creating them itself. Angular's DI framework provides dependencies to classes upon instantiation.

**How DI Works in Angular:**

```typescript
// 1. Define an injectable service
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root' // Makes service available application-wide
})
export class UserService {
  private users: User[] = [];

  getUsers(): User[] {
    return this.users;
  }

  addUser(user: User): void {
    this.users.push(user);
  }
}

// 2. Inject the service into a component
import { Component } from '@angular/core';
import { UserService } from './user.service';

@Component({
  selector: 'app-user-list',
  template: `
    <div *ngFor="let user of users">
      {{ user.name }}
    </div>
  `
})
export class UserListComponent {
  users: User[] = [];

  // Angular automatically injects UserService
  constructor(private userService: UserService) {
    this.users = this.userService.getUsers();
  }
}
```

**The DI Process:**

```typescript
// Step 1: Angular reads the constructor parameters
constructor(private userService: UserService) {}

// Step 2: Angular looks for a provider that can create UserService
// It searches up the injector hierarchy:
// - Component injector
// - Parent component injectors
// - Module injector
// - Root injector

// Step 3: Angular creates or retrieves the UserService instance
// If providedIn: 'root', it's a singleton across the app

// Step 4: Angular injects the instance into the component
```

**Injector Hierarchy:**

```typescript
// Root Module (providedIn: 'root')
@Injectable({ providedIn: 'root' })
export class GlobalService {}

// Feature Module
@NgModule({
  providers: [FeatureService] // Module-level provider
})
export class FeatureModule {}

// Component
@Component({
  selector: 'app-my-component',
  providers: [ComponentService] // Component-level provider
})
export class MyComponent {
  constructor(
    private globalService: GlobalService,      // From root
    private featureService: FeatureService,    // From module
    private componentService: ComponentService // From component
  ) {}
}
```

**Resolution Strategy:**

```typescript
// Angular searches for providers in this order:
// 1. Component's own providers
// 2. Parent component's providers
// 3. Parent's parent providers (and so on)
// 4. Module providers
// 5. Root providers

@Component({
  selector: 'app-child',
  providers: [
    { provide: DataService, useClass: ChildDataService }
  ]
})
export class ChildComponent {
  constructor(private dataService: DataService) {
    // Gets ChildDataService instance
  }
}

@Component({
  selector: 'app-parent',
  template: '<app-child></app-child>',
  providers: [
    { provide: DataService, useClass: ParentDataService }
  ]
})
export class ParentComponent {
  constructor(private dataService: DataService) {
    // Gets ParentDataService instance
  }
}
```

**providedIn Options:**

```typescript
// 1. providedIn: 'root' - Application-wide singleton
@Injectable({ providedIn: 'root' })
export class AppConfigService {}

// 2. providedIn: 'any' - New instance per lazy-loaded module
@Injectable({ providedIn: 'any' })
export class AnalyticsService {}

// 3. providedIn: Module - Scoped to specific module
@Injectable({ providedIn: FeatureModule })
export class FeatureSpecificService {}

// 4. No providedIn - Must be provided explicitly
@Injectable()
export class ManuallyProvidedService {}
```

**Practical Example:**

```typescript
// Service with dependencies
@Injectable({ providedIn: 'root' })
export class AuthService {
  constructor(
    private http: HttpClient,
    private storage: StorageService
  ) {}

  login(credentials: Credentials): Observable<User> {
    return this.http.post<User>('/api/login', credentials).pipe(
      tap(user => this.storage.setItem('user', user))
    );
  }
}

// Component using the service
@Component({
  selector: 'app-login',
  template: `
    <form (submit)="onSubmit()">
      <input [(ngModel)]="email" type="email">
      <input [(ngModel)]="password" type="password">
      <button type="submit">Login</button>
    </form>
  `
})
export class LoginComponent {
  email = '';
  password = '';

  constructor(
    private authService: AuthService,
    private router: Router
  ) {}

  onSubmit(): void {
    this.authService.login({ email: this.email, password: this.password })
      .subscribe(user => {
        this.router.navigate(['/dashboard']);
      });
  }
}
```

**Follow-up Questions:**

1. **What's the difference between providedIn: 'root' and module providers?**
   - providedIn: 'root': Tree-shakeable, singleton, loaded with app
   - Module providers: Not tree-shakeable, tied to module lifecycle
   - providedIn is preferred in modern Angular

2. **How does Angular handle circular dependencies?**
   - Generally throws error at compile time
   - Can use forwardRef() as workaround
   - Better solution: refactor to remove circular dependency

3. **What happens if Angular can't find a provider?**
   ```typescript
   // Throws NullInjectorError
   constructor(private unknownService: UnknownService) {}
   // Error: NullInjectorError: No provider for UnknownService!
   
   // Use @Optional() to handle gracefully
   constructor(@Optional() private unknownService?: UnknownService) {
     if (this.unknownService) {
       // Service is available
     }
   }
   ```

---

### Q2: Explain the different provider types (useClass, useValue, useFactory, useExisting) and when to use each.

**What the interviewer wants to know:**
- Deep understanding of provider configuration
- Ability to choose appropriate provider type
- Knowledge of advanced DI patterns

**Strong Answer:**

Angular provides four primary ways to configure providers, each serving different use cases. Understanding when to use each is crucial for flexible application architecture.

**1. useClass - Provide a Class:**

```typescript
// Basic usage - provide a class
providers: [
  { provide: LoggerService, useClass: LoggerService }
  // Shorthand: just LoggerService
]

// Common use case: Provide different implementations
export abstract class DataService {
  abstract getData(): Observable<Data[]>;
}

@Injectable()
export class ApiDataService implements DataService {
  constructor(private http: HttpClient) {}
  
  getData(): Observable<Data[]> {
    return this.http.get<Data[]>('/api/data');
  }
}

@Injectable()
export class MockDataService implements DataService {
  getData(): Observable<Data[]> {
    return of([
      { id: 1, name: 'Mock Data 1' },
      { id: 2, name: 'Mock Data 2' }
    ]);
  }
}

// In module or component
@NgModule({
  providers: [
    {
      provide: DataService,
      useClass: environment.production ? ApiDataService : MockDataService
    }
  ]
})
export class AppModule {}

// Components depend on abstraction, not implementation
@Component({
  selector: 'app-data-list'
})
export class DataListComponent {
  constructor(private dataService: DataService) {
    // Gets ApiDataService in prod, MockDataService in dev
  }
}
```

**2. useValue - Provide a Static Value:**

```typescript
// Configuration objects
export const APP_CONFIG = {
  apiUrl: 'https://api.example.com',
  timeout: 5000,
  retryAttempts: 3
};

providers: [
  { provide: 'APP_CONFIG', useValue: APP_CONFIG }
]

// Inject with @Inject decorator
constructor(@Inject('APP_CONFIG') private config: any) {
  console.log(this.config.apiUrl);
}

// Better: Use InjectionToken for type safety
export const APP_CONFIG = new InjectionToken<AppConfig>('app.config');

export interface AppConfig {
  apiUrl: string;
  timeout: number;
  retryAttempts: number;
}

providers: [
  {
    provide: APP_CONFIG,
    useValue: {
      apiUrl: 'https://api.example.com',
      timeout: 5000,
      retryAttempts: 3
    }
  }
]

// Type-safe injection
constructor(@Inject(APP_CONFIG) private config: AppConfig) {
  console.log(this.config.apiUrl); // TypeScript knows the type
}
```

**Use cases for useValue:**
```typescript
// 1. Feature flags
providers: [
  { provide: 'ENABLE_BETA_FEATURES', useValue: true }
]

// 2. Environment configuration
providers: [
  { provide: 'ENVIRONMENT', useValue: environment }
]

// 3. Constants
providers: [
  { provide: 'API_VERSION', useValue: 'v2' },
  { provide: 'MAX_ITEMS_PER_PAGE', useValue: 50 }
]

// 4. Mock data for testing
providers: [
  {
    provide: 'MOCK_USERS',
    useValue: [
      { id: 1, name: 'John' },
      { id: 2, name: 'Jane' }
    ]
  }
]
```

**3. useFactory - Create with Factory Function:**

```typescript
// Basic factory
function loggerFactory(): LoggerService {
  return new LoggerService();
}

providers: [
  { provide: LoggerService, useFactory: loggerFactory }
]

// Factory with dependencies
function authServiceFactory(
  http: HttpClient,
  config: AppConfig
): AuthService {
  return new AuthService(http, config.apiUrl);
}

providers: [
  {
    provide: AuthService,
    useFactory: authServiceFactory,
    deps: [HttpClient, APP_CONFIG] // Inject these into factory
  }
]
```

**Advanced factory patterns:**

```typescript
// 1. Conditional instantiation
export function dataServiceFactory(
  http: HttpClient,
  environment: Environment
): DataService {
  if (environment.useMockData) {
    return new MockDataService();
  } else if (environment.useCache) {
    return new CachedDataService(http);
  } else {
    return new ApiDataService(http);
  }
}

providers: [
  {
    provide: DataService,
    useFactory: dataServiceFactory,
    deps: [HttpClient, ENVIRONMENT]
  }
]

// 2. Complex initialization
export function storageFactory(): StorageService {
  const storage = new StorageService();
  
  // Initialize with data from localStorage
  const savedData = localStorage.getItem('app-state');
  if (savedData) {
    storage.loadState(JSON.parse(savedData));
  }
  
  // Set up auto-save
  storage.changes$.subscribe(state => {
    localStorage.setItem('app-state', JSON.stringify(state));
  });
  
  return storage;
}

// 3. Third-party library integration
export function analyticsFactory(
  config: AppConfig
): AnalyticsService {
  // Initialize third-party SDK
  thirdPartySDK.init({
    apiKey: config.analyticsKey,
    debug: !config.production
  });
  
  return new AnalyticsService(thirdPartySDK);
}

// 4. Multi-provider factory
export function httpInterceptorFactory(
  auth: AuthService,
  logger: LoggerService
): HttpInterceptor[] {
  return [
    new AuthInterceptor(auth),
    new LoggingInterceptor(logger),
    new ErrorInterceptor()
  ];
}
```

**4. useExisting - Alias an Existing Provider:**

```typescript
// Basic aliasing
@Injectable({ providedIn: 'root' })
export class LoggerService {
  log(message: string): void {
    console.log(message);
  }
}

providers: [
  LoggerService,
  { provide: 'LOGGER', useExisting: LoggerService }
]

// Now both work:
constructor(private logger: LoggerService) {}
constructor(@Inject('LOGGER') private logger: LoggerService) {}
```

**Common use cases:**

```typescript
// 1. Interface and implementation
export abstract class DataStorage {
  abstract save(data: any): void;
  abstract load(): any;
}

@Injectable({ providedIn: 'root' })
export class LocalStorageService implements DataStorage {
  save(data: any): void {
    localStorage.setItem('data', JSON.stringify(data));
  }
  
  load(): any {
    return JSON.parse(localStorage.getItem('data') || '{}');
  }
}

providers: [
  LocalStorageService,
  { provide: DataStorage, useExisting: LocalStorageService }
]

// Components can depend on either
constructor(private storage: DataStorage) {} // Abstraction
constructor(private storage: LocalStorageService) {} // Concrete

// 2. Multiple interfaces for same service
@Injectable({ providedIn: 'root' })
export class UserService implements AuthProvider, UserRepository {
  // Implements both interfaces
}

providers: [
  UserService,
  { provide: AuthProvider, useExisting: UserService },
  { provide: UserRepository, useExisting: UserService }
]

// Different components can inject different interfaces
@Component({})
export class LoginComponent {
  constructor(private auth: AuthProvider) {}
}

@Component({})
export class UserListComponent {
  constructor(private users: UserRepository) {}
}

// 3. Legacy compatibility
@Injectable({ providedIn: 'root' })
export class NewApiService {
  // New implementation
}

providers: [
  NewApiService,
  { provide: 'OldApiService', useExisting: NewApiService }
]

// Old code still works
constructor(@Inject('OldApiService') private api: any) {}
```

**Comparison Table:**

| Provider Type | Use When | Example |
|--------------|----------|---------|
| **useClass** | Providing class instances, swapping implementations | Dev vs prod services |
| **useValue** | Static values, configuration objects | API URLs, feature flags |
| **useFactory** | Complex initialization, conditional creation | Third-party SDK setup |
| **useExisting** | Creating aliases, multiple interfaces | Interface abstraction |

**Real-World Combined Example:**

```typescript
// Configuration
export const API_CONFIG = new InjectionToken<ApiConfig>('api.config');

// Abstract service
export abstract class DataService {
  abstract getData(): Observable<Data[]>;
}

// Implementations
@Injectable()
export class ProductionDataService implements DataService {
  constructor(
    @Inject(API_CONFIG) private config: ApiConfig,
    private http: HttpClient
  ) {}
  
  getData(): Observable<Data[]> {
    return this.http.get<Data[]>(`${this.config.url}/data`);
  }
}

@Injectable()
export class DevelopmentDataService implements DataService {
  getData(): Observable<Data[]> {
    return of(MOCK_DATA);
  }
}

// Factory for conditional creation
export function dataServiceFactory(
  http: HttpClient,
  config: ApiConfig
): DataService {
  if (config.useMock) {
    return new DevelopmentDataService();
  }
  return new ProductionDataService(config, http);
}

// Module configuration
@NgModule({
  providers: [
    // useValue for configuration
    {
      provide: API_CONFIG,
      useValue: {
        url: environment.apiUrl,
        useMock: !environment.production
      }
    },
    
    // useFactory for conditional instantiation
    {
      provide: DataService,
      useFactory: dataServiceFactory,
      deps: [HttpClient, API_CONFIG]
    },
    
    // useExisting for alias
    {
      provide: 'LEGACY_DATA_SERVICE',
      useExisting: DataService
    }
  ]
})
export class AppModule {}
```

**Follow-up Questions:**

1. **When would you use useFactory instead of useClass?**
   - Complex initialization logic
   - Conditional instantiation
   - Need to call methods during creation
   - Third-party library integration
   - When dependencies need processing before injection

2. **What's the difference between useValue and useFactory for static data?**
   - useValue: For already-computed static values
   - useFactory: When value needs computation or has dependencies
   - useValue is simpler and more performant for constants

3. **Can you chain providers? For example, alias an alias?**
   ```typescript
   providers: [
     ServiceA,
     { provide: ServiceB, useExisting: ServiceA },
     { provide: ServiceC, useExisting: ServiceB } // Yes, this works
   ]
   // ServiceC → ServiceB → ServiceA (all resolve to same instance)
   ```

---

### Q3: Explain hierarchical dependency injection and how to control provider scope.

**What the interviewer wants to know:**
- Understanding of Angular's injector tree
- Knowledge of provider scope and lifetime
- Ability to architect multi-level dependency structures

**Strong Answer:**

Hierarchical DI is one of Angular's most powerful features. Angular creates a tree of injectors that parallels the component tree, allowing fine-grained control over service scope and lifetime.

**The Injector Hierarchy:**

```typescript
// Application structure
Platform Injector (platform-browser)
    ↓
Root Injector (providedIn: 'root')
    ↓
Module Injectors (lazy-loaded modules)
    ↓
Component Injectors
    ↓
Child Component Injectors
    ↓
Child's Child Component Injectors
```

**How Resolution Works:**

```typescript
@Injectable({ providedIn: 'root' })
export class GlobalService {
  id = 'global';
}

@NgModule({
  providers: [
    { provide: ModuleService, useClass: ModuleService }
  ]
})
export class FeatureModule {}

@Component({
  selector: 'app-parent',
  providers: [
    { provide: ComponentService, useClass: ComponentService }
  ]
})
export class ParentComponent {
  constructor(
    private globalService: GlobalService,      // Found in root injector
    private moduleService: ModuleService,      // Found in module injector
    private componentService: ComponentService // Found in component injector
  ) {}
}

@Component({
  selector: 'app-child'
})
export class ChildComponent {
  constructor(
    private globalService: GlobalService,      // Resolved from root
    private moduleService: ModuleService,      // Resolved from module
    private componentService: ComponentService // Walks up to parent component
  ) {
    // All three services resolved by walking up the injector tree
  }
}
```

**Provider Scopes:**

**1. Root Scope (Application-Wide Singleton):**

```typescript
@Injectable({ providedIn: 'root' })
export class AuthService {
  private currentUser: User | null = null;
  
  // Single instance shared across entire app
  login(credentials: Credentials): Observable<User> {
    return this.http.post<User>('/api/login', credentials).pipe(
      tap(user => this.currentUser = user)
    );
  }
  
  getCurrentUser(): User | null {
    return this.currentUser;
  }
}

// Every component gets the same instance
@Component({ selector: 'app-header' })
export class HeaderComponent {
  constructor(private auth: AuthService) {
    console.log(auth.getCurrentUser()); // Same instance
  }
}

@Component({ selector: 'app-sidebar' })
export class SidebarComponent {
  constructor(private auth: AuthService) {
    console.log(auth.getCurrentUser()); // Same instance
  }
}
```

**2. Module Scope (Lazy-Loaded Modules):**

```typescript
// Each lazy-loaded module gets its own instance
@Injectable({ providedIn: 'any' })
export class FeatureStateService {
  private state = {};
  
  setState(key: string, value: any): void {
    this.state[key] = value;
  }
}

// Admin module has its own instance
@NgModule({
  imports: [CommonModule, RouterModule]
})
export class AdminModule {}

// User module has its own instance (separate from admin)
@NgModule({
  imports: [CommonModule, RouterModule]
})
export class UserModule {}
```

**3. Component Scope (Component and Children):**

```typescript
@Injectable()
export class FormStateService {
  private formData: any = {};
  
  setField(name: string, value: any): void {
    this.formData[name] = value;
  }
  
  getFormData(): any {
    return this.formData;
  }
  
  reset(): void {
    this.formData = {};
  }
}

@Component({
  selector: 'app-user-form',
  providers: [FormStateService], // New instance for this component tree
  template: `
    <app-form-step1></app-form-step1>
    <app-form-step2></app-form-step2>
    <app-form-step3></app-form-step3>
  `
})
export class UserFormComponent {
  constructor(private formState: FormStateService) {}
  
  ngOnDestroy(): void {
    // Service instance destroyed when component destroyed
    this.formState.reset();
  }
}

// All child components share the same FormStateService instance
@Component({ selector: 'app-form-step1' })
export class FormStep1Component {
  constructor(private formState: FormStateService) {
    // Same instance as parent
  }
}
```

**Controlling Scope:**

**Pattern 1: Shared State Across Component Tree:**

```typescript
// Service provided at parent level
@Injectable()
export class WizardStateService {
  currentStep = 0;
  formData: any = {};
  
  nextStep(): void {
    this.currentStep++;
  }
  
  previousStep(): void {
    this.currentStep--;
  }
}

@Component({
  selector: 'app-wizard',
  providers: [WizardStateService], // Scoped to wizard and children
  template: `
    <app-wizard-step1 *ngIf="wizardState.currentStep === 0"></app-wizard-step1>
    <app-wizard-step2 *ngIf="wizardState.currentStep === 1"></app-wizard-step2>
    <app-wizard-step3 *ngIf="wizardState.currentStep === 2"></app-wizard-step3>
  `
})
export class WizardComponent {
  constructor(public wizardState: WizardStateService) {}
}

// Each wizard instance has its own state
@Component({
  template: `
    <app-wizard></app-wizard>
    <app-wizard></app-wizard>
  `
})
export class ParentComponent {
  // Two wizards, two separate WizardStateService instances
}
```

**Pattern 2: Overriding Parent Providers:**

```typescript
@Injectable()
export class ThemeService {
  theme = 'light';
}

@Component({
  selector: 'app-root',
  providers: [ThemeService],
  template: `
    <div [class]="theme.theme">
      <app-main-content></app-main-content>
      <app-dark-sidebar></app-dark-sidebar>
    </div>
  `
})
export class AppComponent {
  constructor(public theme: ThemeService) {
    // Gets light theme
  }
}

@Component({
  selector: 'app-dark-sidebar',
  providers: [
    { provide: ThemeService, useValue: { theme: 'dark' } }
  ],
  template: `<div [class]="theme.theme">Dark sidebar content</div>`
})
export class DarkSidebarComponent {
  constructor(public theme: ThemeService) {
    // Gets dark theme (overridden)
  }
}
```

**Pattern 3: @Optional and @SkipSelf:**

```typescript
@Injectable()
export class LoggerService {
  logs: string[] = [];
  
  constructor(@Optional() @SkipSelf() private parentLogger?: LoggerService) {
    // If parent logger exists, inherit its logs
    if (parentLogger) {
      this.logs = [...parentLogger.logs];
    }
  }
  
  log(message: string): void {
    this.logs.push(message);
    
    // Also log to parent if exists
    if (this.parentLogger) {
      this.parentLogger.log(`[Child] ${message}`);
    }
  }
}

@Component({
  selector: 'app-parent',
  providers: [LoggerService]
})
export class ParentComponent {
  constructor(private logger: LoggerService) {
    this.logger.log('Parent initialized');
  }
}

@Component({
  selector: 'app-child',
  providers: [LoggerService]
})
export class ChildComponent {
  constructor(private logger: LoggerService) {
    // Has access to its own logger AND parent logger
    this.logger.log('Child initialized');
  }
}
```

**Pattern 4: @Host Decorator:**

```typescript
@Injectable()
export class FormContext {
  formId: string;
  
  constructor() {
    this.formId = `form-${Math.random()}`;
  }
}

@Component({
  selector: 'app-custom-form',
  providers: [FormContext],
  template: `
    <app-form-field name="username"></app-form-field>
    <app-form-field name="email"></app-form-field>
  `
})
export class CustomFormComponent {}

@Component({
  selector: 'app-form-field',
  template: `<input [id]="formContext.formId + '-' + name">`
})
export class FormFieldComponent {
  @Input() name: string;
  
  constructor(
    @Host() private formContext: FormContext
  ) {
    // @Host() limits search to host component
    // Won't look beyond CustomFormComponent
  }
}
```

**Real-World Example: Multi-Level Shopping Cart:**

```typescript
// Root-level service (app-wide cart)
@Injectable({ providedIn: 'root' })
export class CartService {
  private items: CartItem[] = [];
  
  addItem(item: CartItem): void {
    this.items.push(item);
  }
  
  getItems(): CartItem[] {
    return this.items;
  }
  
  getTotalPrice(): number {
    return this.items.reduce((sum, item) => sum + item.price, 0);
  }
}

// Component-level service (temporary "quote" cart)
@Injectable()
export class QuoteCartService {
  private items: CartItem[] = [];
  
  constructor(private cartService: CartService) {}
  
  addItem(item: CartItem): void {
    this.items.push(item);
  }
  
  confirmQuote(): void {
    // Move items to main cart
    this.items.forEach(item => this.cartService.addItem(item));
    this.items = [];
  }
  
  cancelQuote(): void {
    this.items = [];
  }
}

@Component({
  selector: 'app-quote-builder',
  providers: [QuoteCartService], // Scoped to this component
  template: `
    <div>
      <app-product-list></app-product-list>
      <button (click)="confirmQuote()">Confirm Quote</button>
      <button (click)="cancelQuote()">Cancel</button>
    </div>
  `
})
export class QuoteBuilderComponent {
  constructor(
    private quoteCart: QuoteCartService,
    private mainCart: CartService
  ) {
    // Has access to both scoped quote cart and global main cart
  }
  
  confirmQuote(): void {
    this.quoteCart.confirmQuote();
  }
  
  cancelQuote(): void {
    this.quoteCart.cancelQuote();
  }
}
```

**Follow-up Questions:**

1. **What's the lifecycle of a component-scoped service?**
   - Created when component is instantiated
   - Destroyed when component is destroyed
   - Shared with all child components
   - Not shared with sibling components

2. **How do you prevent a service from being provided multiple times?**
   ```typescript
   // Use providedIn: 'root' instead of providers array
   @Injectable({ providedIn: 'root' })
   export class SingletonService {}
   
   // Avoid this (creates multiple instances):
   @NgModule({
     providers: [SingletonService] // Each module gets own instance
   })
   ```

3. **Can a child component use @SkipSelf() to skip its own injector?**
   ```typescript
   @Component({
     selector: 'app-child',
     providers: [MyService]
   })
   export class ChildComponent {
     constructor(
       @SkipSelf() private myService: MyService
     ) {
       // Skips own providers, looks in parent
     }
   }
   ```

---

### Q4: What are InjectionTokens and when should you use them?

**What the interviewer wants to know:**
- Understanding of type-safe dependency injection
- Knowledge of advanced DI patterns
- Ability to design flexible dependency systems

**Strong Answer:**

InjectionTokens provide type-safe dependency injection for non-class dependencies like configuration objects, strings, or functions. They solve the problem of injecting primitive values or objects without using string tokens.

**The Problem InjectionTokens Solve:**

```typescript
// ❌ BAD: Using string tokens (no type safety)
providers: [
  { provide: 'API_URL', useValue: 'https://api.example.com' }
]

constructor(@Inject('API_URL') private apiUrl: any) {
  // No type checking
  // Easy to mistype
  // No IDE autocomplete
}

// ✅ GOOD: Using InjectionToken
export const API_URL = new InjectionToken<string>('api.url');

providers: [
  { provide: API_URL, useValue: 'https://api.example.com' }
]

constructor(@Inject(API_URL) private apiUrl: string) {
  // Type-safe
  // IDE support
  // Compile-time errors
}
```

**Creating InjectionTokens:**

```typescript
// Simple token
export const MAX_RETRIES = new InjectionToken<number>('max.retries');

// Token with interface
export interface AppConfig {
  apiUrl: string;
  timeout: number;
  enableLogging: boolean;
}

export const APP_CONFIG = new InjectionToken<AppConfig>('app.config');

// Token with default value
export const DEFAULT_TIMEOUT = new InjectionToken<number>('default.timeout', {
  providedIn: 'root',
  factory: () => 5000
});

// Token with factory that has dependencies
export const AUTH_CONFIG = new InjectionToken<AuthConfig>('auth.config', {
  providedIn: 'root',
  factory: () => {
    const environment = inject(ENVIRONMENT);
    return {
      clientId: environment.authClientId,
      redirectUri: environment.authRedirectUri
    };
  }
});
```

**Common Use Cases:**

**1. Configuration Objects:**

```typescript
export interface DatabaseConfig {
  host: string;
  port: number;
  database: string;
  ssl: boolean;
}

export const DB_CONFIG = new InjectionToken<DatabaseConfig>('db.config');

// Provide in module
@NgModule({
  providers: [
    {
      provide: DB_CONFIG,
      useValue: {
        host: 'localhost',
        port: 5432,
        database: 'myapp',
        ssl: false
      }
    }
  ]
})
export class AppModule {}

// Use in service
@Injectable({ providedIn: 'root' })
export class DatabaseService {
  constructor(@Inject(DB_CONFIG) private config: DatabaseConfig) {
    console.log(`Connecting to ${this.config.host}:${this.config.port}`);
  }
}
```

**2. Feature Flags:**

```typescript
export interface FeatureFlags {
  enableNewUI: boolean;
  enableBetaFeatures: boolean;
  enableAnalytics: boolean;
}

export const FEATURE_FLAGS = new InjectionToken<FeatureFlags>('feature.flags', {
  providedIn: 'root',
  factory: () => ({
    enableNewUI: true,
    enableBetaFeatures: false,
    enableAnalytics: true
  })
});

// Use in component
@Component({
  selector: 'app-dashboard',
  template: `
    <app-new-dashboard *ngIf="flags.enableNewUI"></app-new-dashboard>
    <app-old-dashboard *ngIf="!flags.enableNewUI"></app-old-dashboard>
  `
})
export class DashboardComponent {
  constructor(@Inject(FEATURE_FLAGS) public flags: FeatureFlags) {}
}
```

**3. Multi-Providers:**

```typescript
// Token for multiple HTTP interceptors
export const HTTP_INTERCEPTORS = new InjectionToken<HttpInterceptor[]>(
  'http.interceptors'
);

// Register multiple interceptors
@NgModule({
  providers: [
    { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true },
    { provide: HTTP_INTERCEPTORS, useClass: LoggingInterceptor, multi: true },
    { provide: HTTP_INTERCEPTORS, useClass: ErrorInterceptor, multi: true }
  ]
})
export class AppModule {}

// Angular collects all interceptors into an array
@Injectable()
export class HttpService {
  constructor(
    @Inject(HTTP_INTERCEPTORS) private interceptors: HttpInterceptor[]
  ) {
    console.log(`Registered ${this.interceptors.length} interceptors`);
  }
}
```

**4. Plugin Systems:**

```typescript
export interface Plugin {
  name: string;
  initialize(): void;
  destroy(): void;
}

export const PLUGINS = new InjectionToken<Plugin[]>('app.plugins');

// Define plugins
@Injectable()
export class AnalyticsPlugin implements Plugin {
  name = 'Analytics';
  initialize(): void { console.log('Analytics initialized'); }
  destroy(): void { console.log('Analytics destroyed'); }
}

@Injectable()
export class NotificationPlugin implements Plugin {
  name = 'Notifications';
  initialize(): void { console.log('Notifications initialized'); }
  destroy(): void { console.log('Notifications destroyed'); }
}

// Register plugins
@NgModule({
  providers: [
    { provide: PLUGINS, useClass: AnalyticsPlugin, multi: true },
    { provide: PLUGINS, useClass: NotificationPlugin, multi: true }
  ]
})
export class AppModule {}

// Plugin manager
@Injectable({ providedIn: 'root' })
export class PluginManager {
  constructor(@Inject(PLUGINS) private plugins: Plugin[]) {
    this.initializePlugins();
  }
  
  private initializePlugins(): void {
    this.plugins.forEach(plugin => plugin.initialize());
  }
  
  ngOnDestroy(): void {
    this.plugins.forEach(plugin => plugin.destroy());
  }
}
```

**5. Factory Functions:**

```typescript
export type LoggerFactory = (context: string) => Logger;

export const LOGGER_FACTORY = new InjectionToken<LoggerFactory>(
  'logger.factory',
  {
    providedIn: 'root',
    factory: () => {
      const globalLogger = inject(GlobalLoggerService);
      return (context: string) => globalLogger.createLogger(context);
    }
  }
);

// Use factory
@Injectable()
export class UserService {
  private logger: Logger;
  
  constructor(@Inject(LOGGER_FACTORY) createLogger: LoggerFactory) {
    this.logger = createLogger('UserService');
  }
  
  getUsers(): void {
    this.logger.log('Getting users...');
  }
}
```

**Advanced Patterns:**

**Pattern 1: Optional Configuration:**

```typescript
export const OPTIONAL_CONFIG = new InjectionToken<AppConfig | null>(
  'optional.config',
  {
    providedIn: 'root',
    factory: () => null
  }
);

@Injectable()
export class ConfigurableService {
  constructor(
    @Inject(OPTIONAL_CONFIG) private config: AppConfig | null
  ) {
    if (this.config) {
      console.log('Using custom config');
    } else {
      console.log('Using default config');
    }
  }
}
```

**Pattern 2: Environment-Specific Tokens:**

```typescript
export const ENVIRONMENT = new InjectionToken<Environment>('environment');

export const API_URL = new InjectionToken<string>('api.url', {
  providedIn: 'root',
  factory: () => {
    const env = inject(ENVIRONMENT);
    return env.production
      ? 'https://api.production.com'
      : 'https://api.staging.com';
  }
});

// Provide environment
@NgModule({
  providers: [
    { provide: ENVIRONMENT, useValue: environment }
  ]
})
export class AppModule {}
```

**Pattern 3: Validator Functions:**

```typescript
export type ValidatorFn = (value: any) => boolean;

export const CUSTOM_VALIDATORS = new InjectionToken<ValidatorFn[]>(
  'custom.validators'
);

@NgModule({
  providers: [
    {
      provide: CUSTOM_VALIDATORS,
      useValue: (value) => value.length >= 8,
      multi: true
    },
    {
      provide: CUSTOM_VALIDATORS,
      useValue: (value) => /[A-Z]/.test(value),
      multi: true
    }
  ]
})
export class AppModule {}

@Injectable()
export class ValidationService {
  constructor(
    @Inject(CUSTOM_VALIDATORS) private validators: ValidatorFn[]
  ) {}
  
  validate(value: any): boolean {
    return this.validators.every(validator => validator(value));
  }
}
```

**Migration from String Tokens:**

```typescript
// Before (string tokens)
providers: [
  { provide: 'API_KEY', useValue: 'abc123' },
  { provide: 'API_SECRET', useValue: 'secret' }
]

constructor(
  @Inject('API_KEY') private apiKey: any,
  @Inject('API_SECRET') private apiSecret: any
) {}

// After (InjectionTokens)
export const API_KEY = new InjectionToken<string>('api.key');
export const API_SECRET = new InjectionToken<string>('api.secret');

providers: [
  { provide: API_KEY, useValue: 'abc123' },
  { provide: API_SECRET, useValue: 'secret' }
]

constructor(
  @Inject(API_KEY) private apiKey: string,
  @Inject(API_SECRET) private apiSecret: string
) {}
```

**Follow-up Questions:**

1. **What's the difference between InjectionToken and string tokens?**
   - InjectionToken: Type-safe, better IDE support, compile-time checking
   - String tokens: No type safety, runtime only, prone to typos
   - Always prefer InjectionToken

2. **Can you use InjectionToken with tree-shaking?**
   - Yes, when using factory with providedIn
   - Unused tokens are removed from bundle
   - More efficient than module providers

3. **How do you test services that use InjectionTokens?**
   ```typescript
   TestBed.configureTestingModule({
     providers: [
       MyService,
       { provide: MY_TOKEN, useValue: mockValue }
     ]
   });
   
   const service = TestBed.inject(MyService);
   ```

---

## Advanced Questions

### Q5: Explain multi-providers and give real-world use cases.

**What the interviewer wants to know:**
- Understanding of advanced DI patterns
- Knowledge of extensible architecture
- Ability to design plugin-like systems

**Strong Answer:**

Multi-providers allow multiple values to be associated with a single injection token. Angular collects all providers for that token into an array, enabling plugin architectures and extensible systems.

**Basic Multi-Provider:**

```typescript
export const GREETING = new InjectionToken<string[]>('greeting');

@NgModule({
  providers: [
    { provide: GREETING, useValue: 'Hello', multi: true },
    { provide: GREETING, useValue: 'Bonjour', multi: true },
    { provide: GREETING, useValue: 'Hola', multi: true }
  ]
})
export class AppModule {}

@Injectable()
export class GreetingService {
  constructor(@Inject(GREETING) private greetings: string[]) {
    console.log(this.greetings); // ['Hello', 'Bonjour', 'Hola']
  }
  
  getRandomGreeting(): string {
    return this.greetings[Math.floor(Math.random() * this.greetings.length)];
  }
}
```

**Real-World Use Cases:**

**1. HTTP Interceptors:**

```typescript
// Angular's built-in multi-provider
export const HTTP_INTERCEPTORS = new InjectionToken<HttpInterceptor[]>(
  'HttpInterceptors'
);

// Register multiple interceptors
@NgModule({
  providers: [
    {
      provide: HTTP_INTERCEPTORS,
      useClass: AuthInterceptor,
      multi: true
    },
    {
      provide: HTTP_INTERCEPTORS,
      useClass: LoggingInterceptor,
      multi: true
    },
    {
      provide: HTTP_INTERCEPTORS,
      useClass: CachingInterceptor,
      multi: true
    }
  ]
})
export class AppModule {}

// Interceptors run in registration order
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const authReq = req.clone({
      headers: req.headers.set('Authorization', `Bearer ${this.getToken()}`)
    });
    return next.handle(authReq);
  }
}
```

**2. Route Guards:**

```typescript
export const APP_INITIALIZERS = new InjectionToken<InitializerFn[]>(
  'app.initializers'
);

export type InitializerFn = () => Observable<any> | Promise<any>;

@NgModule({
  providers: [
    {
      provide: APP_INITIALIZERS,
      useFactory: (config: ConfigService) => () => config.load(),
      deps: [ConfigService],
      multi: true
    },
    {
      provide: APP_INITIALIZERS,
      useFactory: (auth: AuthService) => () => auth.initialize(),
      deps: [AuthService],
      multi: true
    },
    {
      provide: APP_INITIALIZERS,
      useFactory: (i18n: I18nService) => () => i18n.loadTranslations(),
      deps: [I18nService],
      multi: true
    }
  ]
})
export class AppModule {
  constructor(
    @Inject(APP_INITIALIZERS) private initializers: InitializerFn[]
  ) {
    this.runInitializers();
  }
  
  private async runInitializers(): Promise<void> {
    for (const initializer of this.initializers) {
      await initializer();
    }
  }
}
```

**3. Validation System:**

```typescript
export interface Validator {
  name: string;
  validate(value: any): boolean;
  message: string;
}

export const VALIDATORS = new InjectionToken<Validator[]>('validators');

// Email validator
export class EmailValidator implements Validator {
  name = 'email';
  message = 'Invalid email format';
  
  validate(value: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value);
  }
}

// Password strength validator
export class PasswordValidator implements Validator {
  name = 'password';
  message = 'Password must be at least 8 characters with uppercase and number';
  
  validate(value: string): boolean {
    return value.length >= 8 &&
           /[A-Z]/.test(value) &&
           /[0-9]/.test(value);
  }
}

// Register validators
@NgModule({
  providers: [
    { provide: VALIDATORS, useClass: EmailValidator, multi: true },
    { provide: VALIDATORS, useClass: PasswordValidator, multi: true }
  ]
})
export class ValidationModule {}

// Validation service
@Injectable()
export class ValidationService {
  constructor(@Inject(VALIDATORS) private validators: Validator[]) {}
  
  validate(type: string, value: any): { valid: boolean; message?: string } {
    const validator = this.validators.find(v => v.name === type);
    
    if (!validator) {
      return { valid: true };
    }
    
    const valid = validator.validate(value);
    return {
      valid,
      message: valid ? undefined : validator.message
    };
  }
}
```

**4. Plugin Architecture:**

```typescript
export interface AppPlugin {
  name: string;
  version: string;
  initialize(): void;
  destroy(): void;
}

export const APP_PLUGINS = new InjectionToken<AppPlugin[]>('app.plugins');

// Analytics plugin
@Injectable()
export class AnalyticsPlugin implements AppPlugin {
  name = 'Analytics';
  version = '1.0.0';
  
  initialize(): void {
    console.log('Analytics plugin initialized');
    // Initialize analytics SDK
  }
  
  destroy(): void {
    console.log('Analytics plugin destroyed');
  }
}

// Notification plugin
@Injectable()
export class NotificationPlugin implements AppPlugin {
  name = 'Notifications';
  version = '1.0.0';
  
  constructor(private http: HttpClient) {}
  
  initialize(): void {
    console.log('Notification plugin initialized');
    // Set up notification listener
  }
  
  destroy(): void {
    console.log('Notification plugin destroyed');
  }
}

// Feature module registering plugins
@NgModule({
  providers: [
    { provide: APP_PLUGINS, useClass: AnalyticsPlugin, multi: true },
    { provide: APP_PLUGINS, useClass: NotificationPlugin, multi: true }
  ]
})
export class PluginsModule {}

// Plugin manager
@Injectable({ providedIn: 'root' })
export class PluginManager implements OnDestroy {
  constructor(@Inject(APP_PLUGINS) private plugins: AppPlugin[]) {
    this.initializePlugins();
  }
  
  private initializePlugins(): void {
    this.plugins.forEach(plugin => {
      console.log(`Initializing ${plugin.name} v${plugin.version}`);
      plugin.initialize();
    });
  }
  
  ngOnDestroy(): void {
    this.plugins.forEach(plugin => plugin.destroy());
  }
  
  getPlugin(name: string): AppPlugin | undefined {
    return this.plugins.find(plugin => plugin.name === name);
  }
}
```

**5. Event Bus System:**

```typescript
export interface EventHandler<T = any> {
  event: string;
  handle(data: T): void;
}

export const EVENT_HANDLERS = new InjectionToken<EventHandler[]>(
  'event.handlers'
);

// User login handler
@Injectable()
export class UserLoginHandler implements EventHandler {
  event = 'user:login';
  
  constructor(
    private analytics: AnalyticsService,
    private notifications: NotificationService
  ) {}
  
  handle(data: { user: User }): void {
    this.analytics.track('user_logged_in', { userId: data.user.id });
    this.notifications.show('Welcome back!');
  }
}

// User logout handler
@Injectable()
export class UserLogoutHandler implements EventHandler {
  event = 'user:logout';
  
  constructor(private cache: CacheService) {}
  
  handle(data: { user: User }): void {
    this.cache.clear();
  }
}

// Register handlers
@NgModule({
  providers: [
    { provide: EVENT_HANDLERS, useClass: UserLoginHandler, multi: true },
    { provide: EVENT_HANDLERS, useClass: UserLogoutHandler, multi: true }
  ]
})
export class EventModule {}

// Event bus
@Injectable({ providedIn: 'root' })
export class EventBus {
  constructor(@Inject(EVENT_HANDLERS) private handlers: EventHandler[]) {}
  
  emit(event: string, data?: any): void {
    const matchingHandlers = this.handlers.filter(h => h.event === event);
    matchingHandlers.forEach(handler => handler.handle(data));
  }
}
```

**6. Middleware Pipeline:**

```typescript
export interface Middleware<T = any> {
  process(context: T, next: () => void): void;
}

export const REQUEST_MIDDLEWARE = new InjectionToken<Middleware[]>(
  'request.middleware'
);

// Logging middleware
@Injectable()
export class LoggingMiddleware implements Middleware {
  process(context: any, next: () => void): void {
    console.log('Request started:', context);
    next();
    console.log('Request completed:', context);
  }
}

// Auth middleware
@Injectable()
export class AuthMiddleware implements Middleware {
  constructor(private auth: AuthService) {}
  
  process(context: any, next: () => void): void {
    if (this.auth.isAuthenticated()) {
      next();
    } else {
      throw new Error('Unauthorized');
    }
  }
}

// Validation middleware
@Injectable()
export class ValidationMiddleware implements Middleware {
  process(context: any, next: () => void): void {
    if (this.isValid(context)) {
      next();
    } else {
      throw new Error('Invalid request');
    }
  }
  
  private isValid(context: any): boolean {
    // Validation logic
    return true;
  }
}

// Register middleware
@NgModule({
  providers: [
    { provide: REQUEST_MIDDLEWARE, useClass: LoggingMiddleware, multi: true },
    { provide: REQUEST_MIDDLEWARE, useClass: AuthMiddleware, multi: true },
    { provide: REQUEST_MIDDLEWARE, useClass: ValidationMiddleware, multi: true }
  ]
})
export class MiddlewareModule {}

// Middleware runner
@Injectable()
export class MiddlewareRunner {
  constructor(
    @Inject(REQUEST_MIDDLEWARE) private middleware: Middleware[]
  ) {}
  
  run(context: any): void {
    let index = 0;
    
    const next = () => {
      if (index < this.middleware.length) {
        const current = this.middleware[index++];
        current.process(context, next);
      }
    };
    
    next();
  }
}
```

**Follow-up Questions:**

1. **What happens if you forget multi: true?**
   - Last provider wins (overwrites previous ones)
   - You get single value instead of array
   - Common mistake that breaks plugin systems

2. **Can multi-providers have different provider types?**
   ```typescript
   providers: [
     { provide: TOKEN, useValue: 'value1', multi: true },
     { provide: TOKEN, useClass: Service1, multi: true },
     { provide: TOKEN, useFactory: factory1, multi: true }
   ]
   // Yes! All types can be mixed
   ```

3. **How do you remove a multi-provider?**
   - Can't remove directly
   - Provide empty array or filter at injection
   - Design pattern: use conditional providers

---

## What Interviewers Look For

### Strong Signals

1. **Deep Understanding**: Explains DI concepts clearly with examples
2. **Real Experience**: Shares actual use cases from projects
3. **Best Practices**: Knows when to use each provider type
4. **Architecture Skills**: Can design hierarchical dependency structures
5. **Testing Knowledge**: Understands how DI facilitates testing
6. **Performance Awareness**: Discusses singleton vs scoped services
7. **Modern Patterns**: Familiar with providedIn and InjectionTokens

### Question Examples They'll Ask

- "How would you share state between sibling components?"
- "Design a plugin system using DI"
- "How do you test a service with many dependencies?"
- "When would you use component-level providers?"
- "Explain your experience with hierarchical injection"

## Red Flags to Avoid

1. **"DI is just for services"**: Shows limited understanding
2. **"Always use providedIn: 'root'"**: Lacks nuance
3. **String tokens over InjectionTokens**: Outdated practices
4. **Can't explain injector hierarchy**: Fundamental knowledge gap
5. **"Never used factory providers"**: Limited experience
6. **Confusion about scope and lifetime**: Conceptual misunderstanding
7. **"Multi-providers? Never used them"**: Hasn't built extensible systems

## Key Takeaways

### Essential Concepts

1. **DI Fundamentals**
   - Dependencies are injected, not created
   - Injector hierarchy mirrors component tree
   - Enables testing and modularity

2. **Provider Types**
   - useClass: Standard class providers
   - useValue: Static values and config
   - useFactory: Complex initialization
   - useExisting: Aliases and interfaces

3. **Hierarchical Injection**
   - Root: App-wide singletons
   - Module: Lazy-loaded module scope
   - Component: Component tree scope

4. **InjectionTokens**
   - Type-safe non-class dependencies
   - Better than string tokens
   - Supports tree-shaking

5. **Multi-Providers**
   - Multiple values for one token
   - Plugin architectures
   - Extensible systems

### Best Practices

1. Prefer `providedIn: 'root'` for services
2. Use InjectionTokens for configuration
3. Component providers for scoped state
4. Factory providers for conditional logic
5. Multi-providers for extensibility
6. @Injectable() on all services
7. Test with mock providers

### Interview Success Tips

1. **Start simple**: Explain basic DI first
2. **Progress to advanced**: Show hierarchical knowledge
3. **Use real examples**: Share project experiences
4. **Discuss trade-offs**: No solution fits all cases
5. **Show testing knowledge**: DI enables testability
6. **Ask clarifying questions**: Understand the scenario
7. **Be honest**: Admit knowledge gaps

### Quick Reference

```typescript
// Service with providedIn
@Injectable({ providedIn: 'root' })

// Provider types
{ provide: Token, useClass: Class }
{ provide: Token, useValue: value }
{ provide: Token, useFactory: factory, deps: [Deps] }
{ provide: Token, useExisting: ExistingToken }

// Multi-provider
{ provide: Token, useClass: Class, multi: true }

// InjectionToken
const TOKEN = new InjectionToken<Type>('description');

// Decorators
@Optional() // Allow null
@SkipSelf() // Skip current injector
@Self() // Only current injector
@Host() // Stop at host component
```

Remember: DI is about inverting control and enabling loose coupling. Master these concepts to build maintainable, testable Angular applications.
