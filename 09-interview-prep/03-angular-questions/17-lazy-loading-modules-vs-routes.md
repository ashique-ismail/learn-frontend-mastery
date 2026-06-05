# Lazy Loading: NgModules vs Standalone Routes

## The Idea

**In plain English:** Lazy loading means your app only downloads the code for a page when someone actually visits that page, instead of loading everything at once when the app first opens. Think of it like only fetching the ingredients for a recipe right before you cook it, rather than buying every ingredient for every meal at the start of the week.

**Real-world analogy:** Imagine a big department store that keeps most of its stock in a warehouse out back. When you walk in, only the front entrance and a few popular displays are set up. If you head to the electronics section, a staff member quickly brings out the electronics inventory from the warehouse right then.

- The front entrance displays = the initial app bundle (what loads immediately)
- The warehouse sections = lazy-loaded route chunks (code kept separate until needed)
- A customer walking to electronics = the user navigating to a route
- Staff bringing out the inventory = the browser downloading that route's chunk on demand

---

## Why Lazy Load?

Split the JavaScript bundle so users only download code for the routes they visit. The checkout module shouldn't be in the initial bundle if the user is just browsing the homepage.

---

## NgModule Lazy Loading (Classic)

```ts
// app-routing.module.ts
const routes: Routes = [
  {
    path: 'checkout',
    loadChildren: () => import('./checkout/checkout.module')
      .then(m => m.CheckoutModule)
  }
];

// checkout.module.ts
@NgModule({
  declarations: [CheckoutComponent, CartSummaryComponent],
  imports: [RouterModule.forChild([
    { path: '', component: CheckoutComponent },
    { path: 'confirm', component: OrderConfirmationComponent },
  ])],
})
export class CheckoutModule {}
```

A separate webpack chunk is created for `CheckoutModule` and all its dependencies.

---

## Standalone Component Lazy Loading (Modern)

### Single Component

```ts
// routes.ts
const routes: Routes = [
  {
    path: 'checkout',
    loadComponent: () => import('./checkout/checkout.component')
      .then(c => c.CheckoutComponent)
  }
];
```

### Lazy Route File (Replaces NgModule)

```ts
// app.routes.ts
const routes: Routes = [
  {
    path: 'checkout',
    loadChildren: () => import('./checkout/checkout.routes')
      .then(r => r.CHECKOUT_ROUTES)
  }
];

// checkout/checkout.routes.ts
export const CHECKOUT_ROUTES: Routes = [
  {
    path: '',
    component: CheckoutComponent,
    providers: [CheckoutService], // route-scoped providers (no NgModule needed)
  },
  {
    path: 'confirm',
    component: OrderConfirmationComponent,
  }
];
```

---

## Feature Comparison

| | NgModule | Standalone |
|---|---|---|
| Boilerplate | High (module class required) | Low (just a routes array) |
| Providers | Module.providers | Route `providers` array |
| Bundle splitting | Per module | Per component or per route file |
| Migration effort | Existing code | New projects or migrated code |

---

## Preloading Strategies

By default, lazy routes load only when navigated to. Preloading downloads them in the background after the app loads:

```ts
// Preload all lazy routes after app loads
RouterModule.forRoot(routes, {
  preloadingStrategy: PreloadAllModules
})

// Or standalone:
provideRouter(routes, withPreloading(PreloadAllModules))
```

**Custom preloading strategy:**
```ts
@Injectable({ providedIn: 'root' })
export class SelectivePreloading implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    return route.data?.['preload'] ? load() : EMPTY;
  }
}

// Mark routes to preload:
{ path: 'dashboard', loadChildren: ..., data: { preload: true } }
```

---

## Impact on Bundle Size

```
Without lazy loading:
main.js = app + auth + checkout + admin + reports = 1.2 MB

With lazy loading:
main.js = app + auth = 200 KB  (initial)
checkout.chunk.js = 150 KB    (loaded on navigation)
admin.chunk.js = 400 KB       (loaded only if user visits admin)
```

---

## Common Interview Questions

**Q: Can you use `loadComponent` and `loadChildren` together?**
Yes. `loadComponent` lazily loads a single component. `loadChildren` lazily loads a set of child routes (from either an NgModule or a routes array). You can use `loadChildren` pointing to a routes file that itself uses `loadComponent` for its routes.

**Q: How does Angular know where to split the bundle?**
The dynamic `import()` expression is the signal to webpack/esbuild to create a separate chunk. Any `import()` inside `loadComponent` or `loadChildren` creates a new chunk at build time.

**Q: What's the difference between `forRoot` and `forChild`?**
`RouterModule.forRoot()` sets up the root router (once, in the app module/bootstrap). `RouterModule.forChild()` registers routes without setting up a new router instance. With standalone routing (`provideRouter()`), this distinction goes away.

**Q: Does `NgModule` lazy loading still work in Angular 17+?**
Yes. NgModules are still fully supported and not deprecated. The standalone approach is the new default and preferred for new code, but NgModule lazy loading continues to work.
