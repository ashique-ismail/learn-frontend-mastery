# providedIn: 'root' vs Module-Level Providers

## The Injector Hierarchy

Angular has a tree of injectors:

```
Platform Injector (providedIn: 'platform')
  └── Root Environment Injector (providedIn: 'root')
        └── Route/Module Injectors
              └── Element Injectors (component/directive providers)
```

---

## `providedIn: 'root'` (Recommended Default)

```ts
@Injectable({
  providedIn: 'root',  // singleton for the entire app
})
export class UserService {
  private users = signal<User[]>([]);
}
```

**Characteristics:**
- Single instance shared across the entire application
- **Tree-shakable** — if nothing injects it, it's excluded from the bundle
- Available anywhere without importing a module
- Created lazily on first injection

---

## `providedIn: 'platform'`

```ts
@Injectable({
  providedIn: 'platform',
})
export class SharedConfigService {}
```

Shared across multiple Angular apps bootstrapped on the same page. Rare in typical SPAs, useful for micro-frontend scenarios with multiple Angular apps.

---

## `providedIn: 'any'`

```ts
@Injectable({
  providedIn: 'any',
})
export class TransientService {}
```

Creates a **new instance per lazy-loaded module** (and one for the root injector). Each lazy module gets its own instance. Useful for services that should be isolated per feature.

---

## Module-Level Providers (Legacy Pattern)

```ts
@NgModule({
  providers: [FeatureService]  // NOT tree-shakable
})
export class FeatureModule {}
```

**Problems:**
- Not tree-shakable — included in bundle even if unused
- Creates a new instance per module (can be surprising)
- If FeatureModule is eagerly loaded, FeatureService is in the root injector

---

## Component-Level Providers (Element Injector)

```ts
@Component({
  selector: 'app-form',
  providers: [FormStateService], // new instance per component instance
})
export class FormComponent {}
```

**Use case:** Services that must be isolated per component instance (e.g., a form state service for a reusable form widget).

**Destroyed:** When the component is destroyed, so is the service instance.

---

## Route-Level Providers (Standalone)

```ts
const routes: Routes = [
  {
    path: 'checkout',
    providers: [CheckoutService], // scoped to this route and children
    loadComponent: () => import('./checkout.component'),
  }
];
```

Creates a new environment injector for the route. Service is available to the component and all its children, destroyed when navigating away.

---

## Provider Types

```ts
// useClass — most common, creates instance
providers: [{ provide: LogService, useClass: ProductionLogService }]

// useValue — provides a static value
providers: [{ provide: API_URL, useValue: 'https://api.example.com' }]

// useFactory — creates instance with custom logic
providers: [{
  provide: AuthService,
  useFactory: (config: Config) => new AuthService(config.authUrl),
  deps: [Config]
}]

// useExisting — alias one token to another
providers: [{ provide: NewService, useExisting: OldService }]
```

---

## Common Interview Questions

**Q: What's the main reason to prefer `providedIn: 'root'` over `providers` in NgModule?**
Tree-shaking. `providedIn: 'root'` is tree-shakable: if nothing injects the service, it's excluded from the bundle. NgModule `providers` are always included when the module is loaded.

**Q: Can you have multiple instances of a `providedIn: 'root'` service?**
Normally no — it's a singleton. But if a lazy-loaded module also provides the same service (either via `providers` in the NgModule or `providedIn: 'any'`), that module gets its own instance in its injector.

**Q: When would you use component-level providers?**
When you need a fresh instance per component (e.g., a form wizard where each wizard instance needs isolated state), or when you want the service to be destroyed with the component.

**Q: What's an `InjectionToken`?**
A way to inject non-class values (strings, configs, functions) without using a class as the injection key:
```ts
const API_URL = new InjectionToken<string>('API_URL');
providers: [{ provide: API_URL, useValue: 'https://api.example.com' }]
// Usage:
constructor(@Inject(API_URL) private apiUrl: string) {}
// Or with inject():
const apiUrl = inject(API_URL);
```
