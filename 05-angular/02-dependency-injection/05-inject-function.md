# inject() Function in Angular

## The Idea

**In plain English:** The `inject()` function is a way for a piece of code (like a webpage section) to ask Angular's built-in system for a tool or service it needs, without having to list all those needs in a special setup method called a constructor. Angular acts like a supplier that hands out the right tool whenever you ask.

**Real-world analogy:** Imagine you work at a restaurant kitchen. Instead of picking up every tool you need at the start of your shift and carrying them all to your station (constructor injection), you can just reach to a shared tool rack on the wall and grab exactly what you need, right when you need it.

- The shared tool rack on the wall = Angular's dependency injection system
- Each tool (knife, spatula, tongs) = a service or dependency your code needs
- Reaching out and grabbing a tool = calling `inject(SomeService)`

---

## Table of Contents

- [Introduction](#introduction)
- [Understanding inject()](#understanding-inject)
- [Basic Usage](#basic-usage)
- [Injection Context](#injection-context)
- [Use Cases and Patterns](#use-cases-and-patterns)
- [Functional Guards and Resolvers](#functional-guards-and-resolvers)
- [Custom Injection Functions](#custom-injection-functions)
- [Comparison with Constructor Injection](#comparison-with-constructor-injection)
- [Practical Examples](#practical-examples)
- [Best Practices](#best-practices)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Resources](#resources)

## Introduction

The `inject()` function, introduced in Angular 14, provides a new way to inject dependencies without using constructor injection. This function-based approach enables more flexible dependency injection patterns, especially useful for functional programming, composable utilities, and modern Angular features like functional guards and resolvers.

`inject()` represents a shift towards a more functional programming style in Angular, making code more concise and enabling new patterns previously difficult or impossible with constructor injection.

## Understanding inject()

The `inject()` function retrieves a dependency from the current injection context:

```typescript
import { inject } from '@angular/core';

// Old way (constructor injection)
class MyComponent {
  constructor(private http: HttpClient) {}
}

// New way (inject function)
class MyComponent {
  private http = inject(HttpClient);
}
```

### Function Signature

```typescript
function inject<T>(
  token: Type<T> | InjectionToken<T>,
  flags?: InjectFlags
): T;

// InjectFlags options
enum InjectFlags {
  Default = 0,
  Optional = 1 << 0,     // @Optional()
  SkipSelf = 1 << 1,     // @SkipSelf()
  Self = 1 << 2,         // @Self()
  Host = 1 << 3          // @Host()
}
```

### How It Works

```typescript
// inject() must be called in an injection context
@Component({
  selector: 'app-example'
})
export class ExampleComponent {
  // ✓ Valid: Called during class field initialization
  private http = inject(HttpClient);
  
  constructor() {
    // ✓ Valid: Called in constructor
    const router = inject(Router);
  }
  
  ngOnInit() {
    // ✗ Invalid: Not in injection context
    const service = inject(MyService); // Error!
  }
}
```

## Basic Usage

### Simple Injection

```typescript
import { Component, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Router } from '@angular/router';

@Component({
  selector: 'app-user-list',
  template: `
    <div *ngFor="let user of users$ | async">
      {{ user.name }}
    </div>
  `
})
export class UserListComponent {
  private http = inject(HttpClient);
  private router = inject(Router);
  
  users$ = this.http.get<User[]>('/api/users');
  
  navigateToUser(id: string): void {
    this.router.navigate(['/users', id]);
  }
}
```

### With Options

```typescript
import { Component, inject, InjectFlags } from '@angular/core';

@Component({
  selector: 'app-flexible'
})
export class FlexibleComponent {
  // Optional dependency
  private analytics = inject(AnalyticsService, { optional: true });
  
  // Skip self injector
  private parentConfig = inject(ConfigService, { skipSelf: true });
  
  // Self only
  private localService = inject(LocalService, { self: true });
  
  // Multiple flags
  private optionalParent = inject(
    ParentService, 
    { optional: true, skipSelf: true }
  );
  
  constructor() {
    if (this.analytics) {
      this.analytics.trackPageView();
    }
  }
}
```

### In Standalone Components

```typescript
import { Component, inject } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-standalone',
  standalone: true,
  imports: [CommonModule],
  template: `<div>{{ data$ | async }}</div>`
})
export class StandaloneComponent {
  private dataService = inject(DataService);
  data$ = this.dataService.getData();
}
```

## Injection Context

### Valid Injection Contexts

```typescript
// 1. Class field initializers
export class MyComponent {
  private service = inject(MyService);  // ✓ Valid
}

// 2. Constructor
export class MyComponent {
  private service: MyService;
  
  constructor() {
    this.service = inject(MyService);  // ✓ Valid
  }
}

// 3. Factory functions
export const myFactory = () => {
  const service = inject(MyService);  // ✓ Valid
  return new MyClass(service);
};

// 4. Field initializers with factory
export class MyComponent {
  data = this.loadData();
  
  private loadData() {
    const http = inject(HttpClient);  // ✓ Valid (called during initialization)
    return http.get('/api/data');
  }
}

// 5. Functional guards/resolvers
export const authGuard = () => {
  const auth = inject(AuthService);  // ✓ Valid
  return auth.isAuthenticated();
};
```

### Invalid Injection Contexts

```typescript
export class MyComponent {
  ngOnInit() {
    // ✗ Invalid: Lifecycle hooks are not injection contexts
    const service = inject(MyService); // Error!
  }
  
  onClick() {
    // ✗ Invalid: Event handlers are not injection contexts
    const service = inject(MyService); // Error!
  }
  
  async loadData() {
    // ✗ Invalid: Async methods lose injection context
    await somePromise();
    const service = inject(MyService); // Error!
  }
}
```

### runInInjectionContext

Use `runInInjectionContext` to create an injection context:

```typescript
import { inject, runInInjectionContext, Injector } from '@angular/core';

export class MyComponent {
  constructor(private injector: Injector) {}
  
  ngOnInit() {
    // Create injection context
    runInInjectionContext(this.injector, () => {
      const service = inject(MyService);  // ✓ Valid now
      service.doSomething();
    });
  }
  
  async loadData() {
    await fetch('/api/data');
    
    // Use injection context after async operation
    runInInjectionContext(this.injector, () => {
      const logger = inject(LoggerService);
      logger.log('Data loaded');
    });
  }
}
```

## Use Cases and Patterns

### Composable Utilities

```typescript
// Reusable injection utilities
export function injectLocalStorage() {
  const platformId = inject(PLATFORM_ID);
  
  if (isPlatformBrowser(platformId)) {
    return window.localStorage;
  }
  
  return {
    getItem: () => null,
    setItem: () => {},
    removeItem: () => {},
    clear: () => {}
  };
}

export function injectQueryParams() {
  const route = inject(ActivatedRoute);
  return route.queryParams;
}

export function injectRouteParam(key: string) {
  const route = inject(ActivatedRoute);
  return route.params.pipe(
    map(params => params[key])
  );
}

// Usage in components
@Component({
  selector: 'app-user-detail'
})
export class UserDetailComponent {
  private localStorage = injectLocalStorage();
  private userId$ = injectRouteParam('id');
  
  ngOnInit() {
    this.userId$.subscribe(id => {
      this.localStorage.setItem('lastViewedUser', id);
    });
  }
}
```

### Base Class Pattern

```typescript
// Base class with common dependencies
export abstract class BaseComponent {
  protected router = inject(Router);
  protected route = inject(ActivatedRoute);
  protected toastr = inject(ToastrService);
  
  navigateBack(): void {
    this.router.navigate(['../'], { relativeTo: this.route });
  }
  
  showSuccess(message: string): void {
    this.toastr.success(message);
  }
  
  showError(message: string): void {
    this.toastr.error(message);
  }
}

// Derived components automatically have these dependencies
@Component({
  selector: 'app-user-form'
})
export class UserFormComponent extends BaseComponent {
  private userService = inject(UserService);
  
  saveUser(user: User): void {
    this.userService.save(user).subscribe({
      next: () => {
        this.showSuccess('User saved successfully');
        this.navigateBack();
      },
      error: () => this.showError('Failed to save user')
    });
  }
}
```

### Conditional Injection

```typescript
export function injectAuthService() {
  const config = inject(ConfigService);
  
  if (config.get('useOAuth')) {
    return inject(OAuthService);
  } else {
    return inject(BasicAuthService);
  }
}

@Component({
  selector: 'app-login'
})
export class LoginComponent {
  private auth = injectAuthService();
  
  login(credentials: Credentials): void {
    this.auth.login(credentials).subscribe();
  }
}
```

### Injection with Transformation

```typescript
export function injectUserSettings() {
  const http = inject(HttpClient);
  const userId = inject(CURRENT_USER_ID);
  
  return http.get<Settings>(`/api/users/${userId}/settings`).pipe(
    map(settings => ({
      ...settings,
      theme: settings.theme || 'default',
      language: settings.language || 'en'
    }))
  );
}

@Component({
  selector: 'app-settings'
})
export class SettingsComponent {
  settings$ = injectUserSettings();
}
```

## Functional Guards and Resolvers

### Functional Guard

```typescript
// auth.guard.ts
import { inject } from '@angular/core';
import { Router } from '@angular/router';
import { AuthService } from './auth.service';

export const authGuard = () => {
  const authService = inject(AuthService);
  const router = inject(Router);
  
  if (authService.isLoggedIn()) {
    return true;
  }
  
  return router.createUrlTree(['/login']);
};

// Usage in routes
const routes: Routes = [
  {
    path: 'dashboard',
    component: DashboardComponent,
    canActivate: [authGuard]
  }
];
```

### Functional Guard with Parameters

```typescript
export function roleGuard(allowedRoles: string[]) {
  return () => {
    const authService = inject(AuthService);
    const router = inject(Router);
    
    const userRole = authService.getUserRole();
    
    if (allowedRoles.includes(userRole)) {
      return true;
    }
    
    return router.createUrlTree(['/unauthorized']);
  };
}

// Usage
const routes: Routes = [
  {
    path: 'admin',
    component: AdminComponent,
    canActivate: [roleGuard(['admin', 'superadmin'])]
  }
];
```

### Functional Resolver

```typescript
// user.resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { UserService } from './user.service';

export const userResolver: ResolveFn<User> = (route, state) => {
  const userService = inject(UserService);
  const userId = route.paramMap.get('id')!;
  
  return userService.getUser(userId);
};

// With error handling
export const safeUserResolver: ResolveFn<User | null> = (route, state) => {
  const userService = inject(UserService);
  const router = inject(Router);
  const userId = route.paramMap.get('id')!;
  
  return userService.getUser(userId).pipe(
    catchError(error => {
      console.error('Failed to load user:', error);
      router.navigate(['/users']);
      return of(null);
    })
  );
};

// Usage
const routes: Routes = [
  {
    path: 'users/:id',
    component: UserDetailComponent,
    resolve: {
      user: userResolver
    }
  }
];
```

### CanDeactivate Guard

```typescript
export interface CanComponentDeactivate {
  canDeactivate: () => boolean | Observable<boolean>;
}

export const unsavedChangesGuard = (): boolean | Observable<boolean> => {
  const component = inject(CanComponentDeactivate);
  
  return component.canDeactivate ? component.canDeactivate() : true;
};

// Component implementation
@Component({
  selector: 'app-form'
})
export class FormComponent implements CanComponentDeactivate {
  hasUnsavedChanges = false;
  
  canDeactivate(): boolean {
    if (this.hasUnsavedChanges) {
      return confirm('You have unsaved changes. Do you want to leave?');
    }
    return true;
  }
}
```

## Custom Injection Functions

### Query Parameter Injection

```typescript
export function injectQueryParam<T = string>(
  key: string,
  transform?: (value: string) => T
): Observable<T | null> {
  const route = inject(ActivatedRoute);
  
  return route.queryParamMap.pipe(
    map(params => {
      const value = params.get(key);
      if (!value) return null;
      return transform ? transform(value) : (value as unknown as T);
    })
  );
}

// Usage
@Component({
  selector: 'app-search'
})
export class SearchComponent {
  searchTerm$ = injectQueryParam('q');
  page$ = injectQueryParam('page', Number);
  sortBy$ = injectQueryParam<SortOption>('sort');
}
```

### Storage Injection

```typescript
export function injectStorage(type: 'local' | 'session' = 'local') {
  const platformId = inject(PLATFORM_ID);
  
  if (!isPlatformBrowser(platformId)) {
    return createMockStorage();
  }
  
  const storage = type === 'local' ? localStorage : sessionStorage;
  
  return {
    get: <T>(key: string): T | null => {
      const item = storage.getItem(key);
      return item ? JSON.parse(item) : null;
    },
    set: <T>(key: string, value: T): void => {
      storage.setItem(key, JSON.stringify(value));
    },
    remove: (key: string): void => {
      storage.removeItem(key);
    },
    clear: (): void => {
      storage.clear();
    }
  };
}

// Usage
@Component({
  selector: 'app-preferences'
})
export class PreferencesComponent {
  private storage = injectStorage('local');
  
  savePreference(key: string, value: any): void {
    this.storage.set(key, value);
  }
  
  loadPreference<T>(key: string): T | null {
    return this.storage.get<T>(key);
  }
}
```

### Destroy Signal

```typescript
export function injectDestroy$() {
  const subject = new Subject<void>();
  const destroyRef = inject(DestroyRef);
  
  destroyRef.onDestroy(() => {
    subject.next();
    subject.complete();
  });
  
  return subject.asObservable();
}

// Usage
@Component({
  selector: 'app-data'
})
export class DataComponent {
  private destroy$ = injectDestroy$();
  private dataService = inject(DataService);
  
  ngOnInit() {
    this.dataService.getData()
      .pipe(takeUntil(this.destroy$))
      .subscribe(data => console.log(data));
  }
}
```

### Media Query Injection

```typescript
export function injectMediaQuery(query: string): Signal<boolean> {
  const platformId = inject(PLATFORM_ID);
  
  if (!isPlatformBrowser(platformId)) {
    return signal(false);
  }
  
  const matches = signal(window.matchMedia(query).matches);
  const mediaQuery = window.matchMedia(query);
  
  const listener = (e: MediaQueryListEvent) => matches.set(e.matches);
  mediaQuery.addEventListener('change', listener);
  
  const destroyRef = inject(DestroyRef);
  destroyRef.onDestroy(() => {
    mediaQuery.removeEventListener('change', listener);
  });
  
  return matches.asReadonly();
}

// Usage
@Component({
  selector: 'app-responsive'
})
export class ResponsiveComponent {
  isMobile = injectMediaQuery('(max-width: 768px)');
  isTablet = injectMediaQuery('(min-width: 769px) and (max-width: 1024px)');
  isDesktop = injectMediaQuery('(min-width: 1025px)');
}
```

## Comparison with Constructor Injection

### Constructor Injection

```typescript
@Component({
  selector: 'app-user'
})
export class UserComponent {
  constructor(
    private http: HttpClient,
    private router: Router,
    private route: ActivatedRoute,
    @Optional() private analytics: AnalyticsService,
    @SkipSelf() private parentConfig: ConfigService
  ) {}
}
```

### inject() Function

```typescript
@Component({
  selector: 'app-user'
})
export class UserComponent {
  private http = inject(HttpClient);
  private router = inject(Router);
  private route = inject(ActivatedRoute);
  private analytics = inject(AnalyticsService, { optional: true });
  private parentConfig = inject(ConfigService, { skipSelf: true });
}
```

### Advantages of inject()

1. **No constructor boilerplate**
2. **Easier inheritance** (no super() calls needed)
3. **Composable utilities**
4. **Functional programming patterns**
5. **Conditional injection**

```typescript
// Conditional injection with inject()
export class MyComponent {
  private logger = environment.production
    ? inject(ProductionLogger)
    : inject(DebugLogger);
}

// Difficult with constructor injection
export class MyComponent {
  private logger: Logger;
  
  constructor(
    @Inject(ProductionLogger) prodLogger: ProductionLogger,
    @Inject(DebugLogger) debugLogger: DebugLogger
  ) {
    this.logger = environment.production ? prodLogger : debugLogger;
  }
}
```

### When to Use Each

```typescript
// Use constructor injection when:
// - You prefer traditional OOP style
// - You're working with existing code
// - You want explicit constructor parameters

// Use inject() when:
// - Creating functional guards/resolvers
// - Building composable utilities
// - Working with inheritance
// - Preferring functional programming style
```

## Practical Examples

### Example 1: Feature Flag Service

```typescript
// feature-flags.ts
export function injectFeatureFlag(flag: string): Signal<boolean> {
  const http = inject(HttpClient);
  const flagsSignal = signal(false);
  
  http.get<Record<string, boolean>>('/api/feature-flags')
    .subscribe(flags => {
      flagsSignal.set(flags[flag] || false);
    });
  
  return flagsSignal.asReadonly();
}

// Usage
@Component({
  selector: 'app-dashboard'
})
export class DashboardComponent {
  showNewFeature = injectFeatureFlag('new-dashboard');
  enableBetaMode = injectFeatureFlag('beta-features');
}
```

### Example 2: Pagination Utility

```typescript
export interface PaginationConfig {
  pageSize: number;
  pageSizeOptions: number[];
}

export function injectPagination(config: PaginationConfig) {
  const route = inject(ActivatedRoute);
  const router = inject(Router);
  
  const currentPage = signal(1);
  const pageSize = signal(config.pageSize);
  
  // Sync with query params
  route.queryParams.subscribe(params => {
    currentPage.set(Number(params['page']) || 1);
    pageSize.set(Number(params['pageSize']) || config.pageSize);
  });
  
  const setPage = (page: number) => {
    router.navigate([], {
      relativeTo: route,
      queryParams: { page },
      queryParamsHandling: 'merge'
    });
  };
  
  const setPageSize = (size: number) => {
    router.navigate([], {
      relativeTo: route,
      queryParams: { pageSize: size, page: 1 },
      queryParamsHandling: 'merge'
    });
  };
  
  return {
    currentPage: currentPage.asReadonly(),
    pageSize: pageSize.asReadonly(),
    setPage,
    setPageSize,
    pageSizeOptions: config.pageSizeOptions
  };
}

// Usage
@Component({
  selector: 'app-user-list'
})
export class UserListComponent {
  pagination = injectPagination({
    pageSize: 10,
    pageSizeOptions: [10, 25, 50, 100]
  });
  
  private http = inject(HttpClient);
  
  users$ = toObservable(this.pagination.currentPage).pipe(
    switchMap(page => 
      this.http.get(`/api/users?page=${page}&size=${this.pagination.pageSize()}`)
    )
  );
}
```

### Example 3: Form Utilities

```typescript
export function injectForm<T>(
  initialValue: T,
  validators?: ValidatorFn[]
) {
  const fb = inject(FormBuilder);
  const form = fb.group(initialValue as any, { validators });
  
  const value = toSignal(
    form.valueChanges.pipe(startWith(initialValue)),
    { initialValue }
  );
  
  const valid = toSignal(
    form.statusChanges.pipe(
      startWith(form.status),
      map(status => status === 'VALID')
    ),
    { initialValue: false }
  );
  
  return {
    form,
    value,
    valid,
    reset: () => form.reset(initialValue),
    patchValue: (value: Partial<T>) => form.patchValue(value as any),
    markAllAsTouched: () => form.markAllAsTouched()
  };
}

// Usage
@Component({
  selector: 'app-user-form'
})
export class UserFormComponent {
  formUtils = injectForm(
    { name: '', email: '', age: null },
    [Validators.required]
  );
  
  onSubmit() {
    if (this.formUtils.valid()) {
      console.log('Submitting:', this.formUtils.value());
    } else {
      this.formUtils.markAllAsTouched();
    }
  }
}
```

## Best Practices

### 1. Use inject() in Appropriate Contexts

```typescript
// Good: Field initializers
export class MyComponent {
  private service = inject(MyService);
}

// Good: Constructor
export class MyComponent {
  private service: MyService;
  
  constructor() {
    this.service = inject(MyService);
  }
}

// Bad: Lifecycle hooks
export class MyComponent {
  ngOnInit() {
    const service = inject(MyService); // Error!
  }
}
```

### 2. Create Reusable Injection Utilities

```typescript
// Good: Reusable utility
export function injectCurrentUser() {
  const auth = inject(AuthService);
  return auth.currentUser$;
}

// Use across multiple components
export class ComponentA {
  user$ = injectCurrentUser();
}

export class ComponentB {
  user$ = injectCurrentUser();
}
```

### 3. Handle Optional Dependencies Properly

```typescript
// Good: Explicit optional handling
export class MyComponent {
  private analytics = inject(AnalyticsService, { optional: true });
  
  trackEvent(event: string) {
    this.analytics?.track(event);
  }
}

// Bad: Assuming presence
export class MyComponent {
  private analytics = inject(AnalyticsService);
  
  trackEvent(event: string) {
    this.analytics.track(event); // May crash if not provided
  }
}
```

### 4. Document Custom Injection Functions

```typescript
/**
 * Injects route parameter as an observable.
 * 
 * @param key - The route parameter key
 * @returns Observable of the parameter value
 * 
 * @example
 * const userId$ = injectRouteParam('id');
 */
export function injectRouteParam(key: string): Observable<string | null> {
  const route = inject(ActivatedRoute);
  return route.paramMap.pipe(map(params => params.get(key)));
}
```

## Common Mistakes

### 1. Using inject() Outside Injection Context

```typescript
// Wrong
export class MyComponent {
  ngOnInit() {
    const service = inject(MyService); // Error!
  }
}

// Correct
export class MyComponent {
  private service = inject(MyService);
  
  ngOnInit() {
    this.service.doSomething();
  }
}
```

### 2. Not Handling Optional Dependencies

```typescript
// Wrong
const service = inject(OptionalService, { optional: true });
service.method(); // May crash!

// Correct
const service = inject(OptionalService, { optional: true });
service?.method();
```

### 3. Forgetting Cleanup

```typescript
// Wrong: No cleanup
export function injectInterval(ms: number) {
  const interval$ = interval(ms);
  return interval$; // Subscription never cleaned up
}

// Correct: With cleanup
export function injectInterval(ms: number) {
  const destroyRef = inject(DestroyRef);
  const interval$ = interval(ms);
  
  const subscription = interval$.subscribe();
  destroyRef.onDestroy(() => subscription.unsubscribe());
  
  return interval$;
}
```

### 4. Mixing Patterns Inconsistently

```typescript
// Inconsistent
export class MyComponent {
  private http = inject(HttpClient); // inject()
  
  constructor(private router: Router) {} // constructor injection
}

// Better: Choose one approach
export class MyComponent {
  private http = inject(HttpClient);
  private router = inject(Router);
}
```

## Interview Questions

### Basic Questions

**Q1: What is the inject() function in Angular?**

A: inject() is a function that retrieves dependencies from the current injection context without using constructor injection. It must be called during class initialization or in an injection context.

**Q2: When can you use the inject() function?**

A: During class field initialization, in constructors, in factory functions, and in functional guards/resolvers. It cannot be used in lifecycle hooks or event handlers without runInInjectionContext.

**Q3: What are the advantages of inject() over constructor injection?**

A: Simpler syntax, easier inheritance (no super() needed), enables functional programming patterns, allows conditional injection, and makes creating composable utilities easier.

### Intermediate Questions

**Q4: How do you handle optional dependencies with inject()?**

A: Use the optional flag: `inject(Service, { optional: true })`, which returns undefined if the service isn't available, similar to @Optional() decorator.

**Q5: What is runInInjectionContext and when would you use it?**

A: runInInjectionContext creates an injection context, allowing inject() to be used in places like lifecycle hooks or async operations where it normally wouldn't work.

**Q6: How do functional guards use inject()?**

A: Functional guards are arrow functions that use inject() to retrieve dependencies like Router and services, making them more concise than class-based guards.

### Advanced Questions

**Q7: How would you create a reusable injection utility for route parameters?**

A: Create a function that uses inject() to get ActivatedRoute, then extracts and transforms route parameters, returning an observable or signal that components can consume.

**Q8: What are the limitations of inject() compared to constructor injection?**

A: inject() must be called in an injection context, can't be used in lifecycle hooks without workarounds, and may be less familiar to developers from other DI frameworks.

**Q9: How do you implement conditional injection based on environment?**

A: Use inject() in a conditional expression or ternary operator based on environment variables or configuration, injecting different services based on the condition.

**Q10: Can inject() be used with custom injection tokens?**

A: Yes, inject() works with any injection token including InjectionToken instances, just like constructor injection: `inject(MY_CUSTOM_TOKEN)`.

## Resources

### Official Documentation

- [inject() Function](https://angular.dev/api/core/inject)
- [Dependency Injection Guide](https://angular.dev/guide/di)

### Articles and Tutorials

- [Modern DI with inject()](https://blog.angular.io/inject-function-guide)
- [Functional Guards and Resolvers](https://angular.io/guide/router-tutorial)

### Video Tutorials

- [Understanding inject() Function](https://www.youtube.com/watch?v=inject-function)
- [Building Composable Utilities](https://www.youtube.com/watch?v=composable-utils)

### Community Resources

- [Angular Discord](https://discord.gg/angular) - Community support
- [Stack Overflow](https://stackoverflow.com/questions/tagged/angular) - Q&A
