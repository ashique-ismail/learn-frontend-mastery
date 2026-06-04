# Angular Element Injector vs Environment Injector

## Table of Contents
- [Overview](#overview)
- [The Two Injector Trees](#the-two-injector-trees)
- [Environment Injector](#environment-injector)
- [Element Injector](#element-injector)
- [Resolution Order](#resolution-order)
- [providedIn Explained](#providedin-explained)
- [Hierarchical Injection Patterns](#hierarchical-injection-patterns)
- [Lazy Loading and Injector Boundaries](#lazy-loading-and-injector-boundaries)
- [inject() Function and Context Rules](#inject-function-and-context-rules)
- [Common Pitfalls](#common-pitfalls)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Overview

Angular's dependency injection system consists of two parallel injector trees that are walked in a specific order to resolve tokens. Understanding this dual-tree architecture is essential for reasoning about service scoping, lifetime, and isolation.

- **Environment Injector tree**: mirrors the application's module/route hierarchy. Created at application bootstrap and at each lazy-loaded route. Services provided here are singletons within their environment scope.
- **Element Injector tree**: mirrors the component/directive tree in the DOM. Created per component or directive instance. Services provided here are scoped to that component's subtree.

When Angular resolves a token, it starts in the element injector tree and walks up toward the root. Only when it reaches the top of the element tree without finding the token does it switch to the environment injector tree.

## The Two Injector Trees

```
Environment Injector Tree (module/route hierarchy)
─────────────────────────────────────────────────
ApplicationRef (root environment injector)
   │  NullInjector (throws if not found)
   │
   ├── Root EnvironmentInjector
   │     providers from bootstrapApplication() / AppModule
   │     all services with providedIn: 'root'
   │
   ├── Feature EnvironmentInjector  (lazy route: /admin)
   │     providers from { providers: [...] } in route config
   │
   └── Feature EnvironmentInjector  (lazy route: /shop)
         providers from loadChildren route config


Element Injector Tree (component/directive tree)
────────────────────────────────────────────────
AppComponent  (injector: providers from @Component.providers)
   │
   ├── NavComponent  (injector: [])
   │
   └── RouterOutlet
         │
         └── AdminComponent  (injector: providers from @Component.providers)
               │
               └── UserListComponent  (injector: [TableService])
```

## Environment Injector

The `EnvironmentInjector` class is Angular's public API for the environment injector tree. It is used for:
- Platform-level providers (HttpClient, Router)
- Application-wide singletons (`providedIn: 'root'`)
- Route-level singletons (lazy module providers)

```typescript
import { EnvironmentInjector, createEnvironmentInjector, inject, ENVIRONMENT_INITIALIZER } from '@angular/core';

// Reading the current environment injector inside a component
@Component({ selector: 'app-root', template: '' })
export class AppComponent {
  private envInjector = inject(EnvironmentInjector);

  runOutsideAngular() {
    // Create a child environment injector with additional providers
    const childInjector = createEnvironmentInjector(
      [{ provide: AnalyticsToken, useValue: new AnalyticsService() }],
      this.envInjector, // parent
    );

    // Use it to get a service that isn't in the component tree
    const analytics = childInjector.get(AnalyticsToken);
    analytics.track('event');

    // Clean up when done
    childInjector.destroy();
  }
}
```

### Platform Injector

Above the root environment injector sits the platform injector, which holds platform-wide singletons:

```typescript
import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';

// Creates the platform injector (one per browser tab)
// All ApplicationRef instances in this tab share this injector
platformBrowserDynamic().bootstrapModule(AppModule);

// platformBrowserDynamic providers:
// - DomSanitizer
// - PlatformLocation
// - DOCUMENT token
```

## Element Injector

Every component and directive creates an element injector. Even if a component declares no providers, it still has an element injector (which is empty and immediately delegates to its parent).

```typescript
@Component({
  selector: 'app-data-table',
  template: `...`,
  providers: [
    // TableService is scoped to DataTableComponent and its children
    TableService,
    // Override a root-provided service for this subtree only
    { provide: HttpClient, useValue: MockHttpClient },
  ],
  viewProviders: [
    // RowHighlighter is only available to the component's OWN template
    // NOT available to content projected into <ng-content>
    RowHighlighter,
  ],
})
export class DataTableComponent {}
```

### providers vs viewProviders

```
Component template:
┌─────────────────────────────────────────────────────────┐
│  DataTableComponent                                      │
│  providers: [TableService]   viewProviders: [Highlighter]│
│                                                          │
│  ┌─────────────────────────────────────────────┐        │
│  │  View children (own template):              │        │
│  │  Can inject: TableService ✓  Highlighter ✓  │        │
│  └─────────────────────────────────────────────┘        │
│  ┌─────────────────────────────────────────────┐        │
│  │  Content children (ng-content projection):  │        │
│  │  Can inject: TableService ✓  Highlighter ✗  │        │
│  └─────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────┘
```

```typescript
// viewProviders are visible to the component's own template view
// but NOT to content projected via <ng-content>.
// This is useful for preventing projected components from accidentally
// receiving services they shouldn't have.

@Component({
  selector: 'app-form-wrapper',
  template: `
    <form>
      <ng-content></ng-content> <!-- cannot inject FormContextService -->
      <app-submit-button></app-submit-button> <!-- CAN inject FormContextService -->
    </form>
  `,
  viewProviders: [FormContextService],
})
export class FormWrapperComponent {}
```

## Resolution Order

The full token resolution algorithm walks both trees:

```
Token lookup for T in Component C:
───────────────────────────────────────────────────────────────
Step 1: Walk the element injector tree upward
  C's element injector          → has T? → return
  C's parent element injector   → has T? → return
  ...
  AppComponent element injector → has T? → return

Step 2: Switch to environment injector tree
  Route EnvironmentInjector     → has T? → return
  Root EnvironmentInjector      → has T? → return
  Platform Injector             → has T? → return
  NullInjector                  → throw InjectionError (or return undefined with @Optional)
```

```typescript
// Demonstrate resolution order with a concrete example

@Injectable()
class LogService {
  name = 'root';
}

@Component({
  selector: 'app-parent',
  template: '<app-child></app-child>',
  providers: [{ provide: LogService, useValue: { name: 'parent' } }],
})
class ParentComponent {
  log = inject(LogService); // resolves to { name: 'parent' }
}

@Component({
  selector: 'app-child',
  template: '',
  // No providers — element injector is empty
})
class ChildComponent {
  log = inject(LogService); // walks up: child (miss) → parent (hit!) → { name: 'parent' }
}

@Component({
  selector: 'app-sibling',
  template: '',
  // No providers
})
class SiblingComponent {
  // Not a descendant of ParentComponent
  log = inject(LogService); // walks up all the way → root environment injector → { name: 'root' }
}
```

### @Self, @SkipSelf, @Host, @Optional

These decorators control which part of the injector tree is searched:

```typescript
import { Self, SkipSelf, Host, Optional, inject } from '@angular/core';

@Component({ selector: 'app-demo', template: '', providers: [LogService] })
class DemoComponent {
  // @Self: only look in THIS component's element injector
  // Throws if not found (unless combined with @Optional)
  private selfLog = inject(LogService, { self: true });

  // @SkipSelf: skip this component's element injector, start at parent
  // Useful for providers that need to reach past their own declaration
  private parentLog = inject(LogService, { skipSelf: true });

  // @Optional: return null instead of throwing if not found
  private maybeLog = inject(LogService, { optional: true }); // LogService | null

  // @Host: only look in the element injector tree up to (and including)
  // the host component of the current view. Does not cross component boundaries.
  private hostLog = inject(LogService, { host: true });
}

// Combining: skip own injector, accept null if not found in parents
class MyDirective {
  private parentService = inject(MyService, { skipSelf: true, optional: true });
}
```

### The @Host Boundary

```
Component A (providers: [LogService])
  ↓
Component B (no providers)
  → @Host search: B's injector → A's injector (stops here — A is host boundary)

Component B's directive uses @Host → finds LogService in A ✓

If LogService was only in AppComponent (above A):
  Component B's directive uses @Host → not found in B or A → null (with @Optional)
```

## providedIn Explained

`providedIn` in `@Injectable` is tree-shaking-friendly scoping: it tells Angular which injector should provide the service without explicitly listing it in `providers[]`.

```typescript
// providedIn: 'root' — provided in the root EnvironmentInjector
// One singleton for the entire application
// Tree-shaken if never injected
@Injectable({ providedIn: 'root' })
class UserService {}

// providedIn: 'platform' — shared across multiple Angular apps on the same page
// (rare; mostly for micro-frontend scenarios)
@Injectable({ providedIn: 'platform' })
class PlatformLogger {}

// providedIn: 'any' — each lazy-loaded module gets its own instance
// (deprecated in newer Angular; use scope-level providers instead)
@Injectable({ providedIn: 'any' })
class ModalService {}

// providedIn: SomeModule — provided in that module's injector (not root)
// Only available to components declared in SomeModule
// NOT tree-shaken
@Injectable({ providedIn: SomeFeatureModule })
class FeatureSpecificService {}

// No providedIn — must be listed in providers[] explicitly
// Good for services that are intentionally component-scoped
@Injectable()
class ComponentScopedService {}
```

### Tree Shaking with providedIn

```typescript
// providedIn: 'root' enables tree-shaking:
// If AnalyticsService is never inject()ed, the bundler removes it.
// If it's in providers: [AnalyticsService], it's always included.

// BAD for tree-shaking (always bundled):
@NgModule({
  providers: [AnalyticsService], // included in bundle even if unused
})
class AppModule {}

// GOOD for tree-shaking:
@Injectable({ providedIn: 'root' })
class AnalyticsService {} // removed from bundle if never injected
```

## Hierarchical Injection Patterns

### Scoped Services (Component-Level Singleton)

```typescript
// Use case: Each instance of DataGridComponent needs its own SortService
// All children within one DataGridComponent share that instance
// Two DataGridComponents get two separate SortService instances

@Injectable() // No providedIn — must be explicitly provided
class SortService {
  private _sortColumn = '';
  sort(column: string) { this._sortColumn = column; }
  get sortColumn() { return this._sortColumn; }
}

@Component({
  selector: 'app-data-grid',
  template: `
    <app-column-header [column]="'name'"></app-column-header>
    <app-column-header [column]="'date'"></app-column-header>
    <app-grid-body></app-grid-body>
  `,
  providers: [SortService], // One instance per DataGridComponent
})
class DataGridComponent {}

@Component({ selector: 'app-column-header', template: '' })
class ColumnHeaderComponent {
  // Gets the SortService from the nearest DataGridComponent ancestor
  private sort = inject(SortService);
}
```

### Override for Testing

```typescript
// Override a root service in a specific component subtree
// Useful for isolated testing without mocking the entire DI system

@Component({
  selector: 'app-feature',
  template: '<app-child></app-child>',
  providers: [
    {
      provide: HttpClient,
      useValue: new FakeHttpClient({ users: [{ id: 1, name: 'Test' }] }),
    },
  ],
})
class FeatureComponent {} // Children use the fake HttpClient
```

### Multi-Providers

```typescript
import { InjectionToken } from '@angular/core';

export const VALIDATORS = new InjectionToken<Validator[]>('Form Validators');

@NgModule({
  providers: [
    { provide: VALIDATORS, useClass: RequiredValidator, multi: true },
    { provide: VALIDATORS, useClass: EmailValidator, multi: true },
    { provide: VALIDATORS, useClass: MinLengthValidator, multi: true },
  ],
})
class FormModule {}

// Inject all validators as an array
class FormField {
  private validators = inject(VALIDATORS); // Validator[]
}
```

## Lazy Loading and Injector Boundaries

Lazy-loaded routes create a new `EnvironmentInjector` as a child of the root. Services provided in the lazy route (via `providers` in the route config or `NgModule`) are only available within that lazy scope.

```typescript
// app.routes.ts
export const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.routes').then(m => m.ADMIN_ROUTES),
    providers: [
      // AdminService is only available inside the /admin route subtree
      AdminService,
      // Even if AdminService has providedIn: 'root', this override takes priority
      // for the admin environment injector scope
    ],
  },
];

// admin/admin.routes.ts
export const ADMIN_ROUTES: Routes = [
  {
    path: '',
    component: AdminShellComponent,
    providers: [AdminDataService], // Scoped to this route and children
    children: [
      { path: 'users', component: UserManagementComponent },
      { path: 'settings', component: SettingsComponent },
    ],
  },
];
```

```
Injector hierarchy after lazy load:

Root EnvironmentInjector (AdminService NOT here unless providedIn:'root')
  │
  └── Admin EnvironmentInjector (AdminService IS here)
        ├── AdminShellComponent element injector
        │     └── UserManagementComponent element injector
        └── SettingsComponent element injector
```

## inject() Function and Context Rules

The `inject()` function is the modern alternative to constructor injection. It can only be called in specific contexts:

```typescript
// Valid contexts for inject():
// 1. Constructor body of a class being injected (component, service, directive, pipe)
// 2. Field initializers
// 3. Factory functions passed to InjectionToken or providers
// 4. Functions called synchronously from one of the above

@Injectable()
class UserService {
  // Field initializer context ✓
  private http = inject(HttpClient);
  private router = inject(Router);

  constructor() {
    // Constructor context ✓
    const auth = inject(AuthService);
  }
}

// Helper function called during field initialization ✓
function createApiClient() {
  const http = inject(HttpClient);
  const baseUrl = inject(BASE_URL_TOKEN);
  return new ApiClient(http, baseUrl);
}

@Component({ /* ... */ })
class MyComponent {
  private api = createApiClient(); // ✓ called during field init
}

// INVALID: inject() outside injection context
class SomeClass {
  getData() {
    // BAD: inject() called outside construction context
    const service = inject(DataService); // throws NG0203
  }
}

// Use runInInjectionContext for dynamic injection
class MyService {
  private injector = inject(Injector);

  doSomethingLater() {
    setTimeout(() => {
      // ✓ runInInjectionContext creates a valid injection context
      runInInjectionContext(this.injector, () => {
        const service = inject(SomeService);
        service.doWork();
      });
    }, 1000);
  }
}
```

## Common Pitfalls

### Pitfall 1: Multiple Instances Due to Duplicate Providers

```typescript
// A common bug in lazy-loaded applications:
// If AuthService is in both root providers AND a lazy module's providers,
// you get TWO instances — one in each environment injector.

// Root app
@Injectable({ providedIn: 'root' })
class AuthService { token = 'root-token'; }

// Lazy module (DON'T DO THIS)
@NgModule({
  providers: [AuthService], // Creates a second instance in the lazy module scope!
})
class AdminModule {}

@Component({ selector: 'app-admin', template: '' })
class AdminComponent {
  auth = inject(AuthService);
  // Gets the AdminModule-scoped instance, not the root one
  // auth.token is undefined (fresh instance) instead of the logged-in token
}
```

### Pitfall 2: Forgetting That Element Injectors Are Per-Instance

```typescript
// If you have a service in @Component.providers, EVERY component instance
// gets its own service instance. This is usually intentional, but can
// cause bugs if you expect sharing.

@Component({
  selector: 'app-cart-item',
  providers: [CartCalculatorService], // NEW instance per <app-cart-item>!
})
class CartItemComponent {}

// Each <app-cart-item> in the list has its own CartCalculatorService
// They don't share state. If you need shared state, provide it in the parent.
```

### Pitfall 3: inject() in ngOnInit

```typescript
// BAD: inject() outside constructor/field initializer
@Component({ selector: 'app-demo', template: '' })
class DemoComponent implements OnInit {
  ngOnInit() {
    // throws NG0203: inject() must be called from an injection context
    const service = inject(MyService);
  }
}

// GOOD: Inject in constructor or as field
@Component({ selector: 'app-demo', template: '' })
class DemoComponent implements OnInit {
  private service = inject(MyService); // field initializer ✓

  ngOnInit() {
    this.service.doWork(); // use it in lifecycle hooks
  }
}
```

### Pitfall 4: viewProviders Breaking Content Projection

```typescript
// If a projected component needs a service from the parent,
// use providers not viewProviders

@Component({
  selector: 'app-modal',
  template: '<ng-content></ng-content>',
  viewProviders: [ModalService], // Content-projected children CANNOT inject this!
  // providers: [ModalService],  // This would work for projected children
})
class ModalComponent {}

// ProjectedContent tries to inject ModalService but gets undefined or throws
@Component({ selector: 'app-projected-content', template: '' })
class ProjectedContent {
  modal = inject(ModalService, { optional: true }); // null — viewProviders not visible
}
```

## Interview Questions

### Question 1: What is the difference between the element injector tree and the environment injector tree?

**Answer**: The element injector tree mirrors the component/directive DOM tree — every component and directive has its own element injector. The environment injector tree mirrors the application's module/route structure — there's one root environment injector and one per lazy-loaded route. When Angular resolves a token, it first walks up the element injector tree (from the requesting component to the root component). If the token is not found there, it switches to the environment injector tree. This dual-tree design means component-provided services can override root services for a subtree, and lazy-loaded routes can have isolated service instances.

### Question 2: What is the difference between providers and viewProviders in @Component?

**Answer**: Both define services scoped to the component's element injector. The difference is visibility to content-projected children. Services in `providers[]` are visible to both the component's own template children AND components/directives projected via `<ng-content>`. Services in `viewProviders[]` are only visible to the component's own template — projected content cannot inject them. `viewProviders` is useful when you want to provide context to a component's own template without accidentally leaking it to external content that happens to be projected in.

### Question 3: How does Angular handle service scoping in lazy-loaded routes?

**Answer**: Each lazy-loaded route (using `loadChildren` or `loadComponent` with a `providers` array) creates a new `EnvironmentInjector` as a child of the root environment injector. Services provided in the route config are instantiated fresh for each lazy route environment — they are singletons within that route but independent from the root. This means a service with `providedIn: 'root'` is still a single instance; but if you also list it in a route's `providers`, the route gets its own instance. This is a common source of multiple-instance bugs.

### Question 4: When would you use @SkipSelf in Angular dependency injection?

**Answer**: `@SkipSelf` tells Angular to skip the current component's own element injector and start resolution from the parent. It's most commonly used when a provider needs to delegate to or extend the parent's implementation. For example, a custom `ErrorHandler` that wraps the parent's handler: `constructor(@SkipSelf() private parentHandler: ErrorHandler)`. Without `@SkipSelf`, the component would inject itself (circular). Another use case: a service that checks if a required parent service exists, without accidentally finding the version it provides itself.

### Question 5: What is the inject() function and how does it differ from constructor injection?

**Answer**: `inject()` is a function (Angular 14+) that resolves a token from the current injection context. It produces identical results to constructor injection but can be used in field initializers, making it more ergonomic for functional-style code and composable utilities. The key constraint is that `inject()` can only be called during the construction phase (constructor body, field initializers, or functions synchronously called from those contexts). It cannot be called in lifecycle hooks like `ngOnInit` or in event handlers. `runInInjectionContext(injector, fn)` is the escape hatch for calling `inject()` outside the normal construction phase.

## Key Takeaways

1. **Two parallel trees**: element injector (component/directive tree) and environment injector (module/route tree) — resolution walks element tree first, then environment tree

2. **Resolution order**: child element → parent element → ... → root component → route EnvironmentInjector → root EnvironmentInjector → platform injector → NullInjector (throws)

3. **providers vs viewProviders**: both create element-scoped services; viewProviders are invisible to `<ng-content>` projected children

4. **providedIn: 'root' is preferred**: enables tree-shaking, creates one singleton, no need to list in providers[]

5. **Lazy routes create EnvironmentInjector boundaries**: services provided at the route level are isolated from the root; beware of accidental multiple instances

6. **@Self / @SkipSelf / @Host / @Optional**: decorators (or inject() options) that control which part of the tree is searched

7. **inject() requires an injection context**: call it in constructors, field initializers, or factory functions — not in lifecycle hooks or event handlers

8. **Component-scoped services**: list in @Component.providers for per-instance singletons shared within that component's subtree

## Resources

- [Angular Docs: Dependency Injection Guide](https://angular.dev/guide/di)
- [Angular Docs: Hierarchical Injectors](https://angular.dev/guide/di/hierarchical-dependency-injection)
- [Angular source: injector.ts](https://github.com/angular/angular/blob/main/packages/core/src/di/injector.ts)
- [Angular source: r3_injector.ts (EnvironmentInjector)](https://github.com/angular/angular/blob/main/packages/core/src/di/r3_injector.ts)
- ["Angular DI deep dive" by Manfred Steyer](https://www.angulararchitects.io/en/blog/the-new-lightweight-injection-token-pattern/)
- [Angular injection context (NG0203 error)](https://angular.dev/errors/NG0203)
