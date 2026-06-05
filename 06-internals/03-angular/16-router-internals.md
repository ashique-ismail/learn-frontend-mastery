# Router Internals

## The Idea

**In plain English:** The Angular Router is the system that decides which page (screen) to show when you type a URL or click a link in an Angular app. It reads the web address, figures out which piece of your app matches it, runs any security checks, fetches needed data, and then displays the right content.

**Real-world analogy:** Think of a large museum with a front desk, security checkpoints, and tour guides. When you arrive and say "I want to see the Egyptian exhibit," a specific process kicks in before you ever set foot in that room.

- The museum directory (route configuration) = the list of URL patterns and which component to show for each
- The front desk clerk checking your ticket = a canActivate guard deciding if you are allowed to visit that route
- The tour guide fetching brochures before the tour starts = a resolver loading data before the component appears
- The floor number and room sign you follow = the URL path that Angular matches to find the right component

---

## Table of Contents

- [Introduction](#introduction)
- [Navigation Cycle](#navigation-cycle)
- [Route Matching Algorithm](#route-matching-algorithm)
- [Guards Execution Order](#guards-execution-order)
- [Route Resolution](#route-resolution)
- [URL Serialization](#url-serialization)
- [Router State Management](#router-state-management)
- [Common Misconceptions](#common-misconceptions)
- [Performance Implications](#performance-implications)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

The Angular Router is a sophisticated navigation system that manages application state based on URL changes. Understanding its internals reveals how routing decisions are made, guards are executed, and components are loaded.

Key concepts:
- **Navigation cycle**: Multi-phase process from URL change to component rendering
- **Route matching**: Algorithm to find matching route configurations
- **Guard execution**: Order and timing of navigation guards
- **State management**: Router state tree and snapshots
- **URL handling**: Parsing, serialization, and synchronization

## Navigation Cycle

### Complete Navigation Flow

The router navigation follows a precise sequence:

```typescript
// Navigation phases
enum NavigationPhase {
  ApplyRedirects,
  Recognize,
  RunGuards,
  ResolveData,
  ActivateRoutes,
  Complete
}

// Simplified navigation cycle
class NavigationCycle {
  async navigate(url: string): Promise<boolean> {
    console.log('Starting navigation to:', url);

    try {
      // Phase 1: Apply redirects
      const redirectedUrl = await this.applyRedirects(url);
      console.log('After redirects:', redirectedUrl);

      // Phase 2: Recognize routes
      const recognized = await this.recognizeRoutes(redirectedUrl);
      console.log('Recognized routes:', recognized);

      // Phase 3: Run guards
      const guardsResult = await this.runGuards(recognized);
      if (guardsResult !== true) {
        console.log('Navigation blocked by guards');
        return false;
      }

      // Phase 4: Resolve data
      const resolvedData = await this.resolveData(recognized);
      console.log('Resolved data:', resolvedData);

      // Phase 5: Activate routes
      await this.activateRoutes(recognized, resolvedData);
      console.log('Routes activated');

      // Phase 6: Complete
      this.completeNavigation();
      return true;

    } catch (error) {
      console.error('Navigation failed:', error);
      return false;
    }
  }

  private async applyRedirects(url: string): Promise<string> {
    // Apply redirect configurations
    return url;
  }

  private async recognizeRoutes(url: string): Promise<RouterStateSnapshot> {
    // Match URL to routes
    return null as any;
  }

  private async runGuards(state: RouterStateSnapshot): Promise<boolean | UrlTree> {
    // Execute guards
    return true;
  }

  private async resolveData(state: RouterStateSnapshot): Promise<any> {
    // Resolve route data
    return {};
  }

  private async activateRoutes(state: RouterStateSnapshot, data: any): Promise<void> {
    // Activate components
  }

  private completeNavigation(): void {
    // Finalize navigation
  }
}
```

### Navigation Events

Tracking navigation progress:

```typescript
import { Router, NavigationStart, NavigationEnd, NavigationCancel, NavigationError } from '@angular/router';

@Injectable()
export class NavigationTracker {
  constructor(private router: Router) {
    this.trackNavigation();
  }

  private trackNavigation(): void {
    this.router.events.subscribe(event => {
      if (event instanceof NavigationStart) {
        console.log('Navigation started:', {
          id: event.id,
          url: event.url,
          navigationTrigger: event.navigationTrigger,
          restoredState: event.restoredState
        });
      }

      if (event instanceof RoutesRecognized) {
        console.log('Routes recognized:', {
          id: event.id,
          url: event.url,
          urlAfterRedirects: event.urlAfterRedirects,
          state: event.state
        });
      }

      if (event instanceof GuardsCheckStart) {
        console.log('Guards check started:', {
          id: event.id,
          url: event.url,
          urlAfterRedirects: event.urlAfterRedirects,
          state: event.state
        });
      }

      if (event instanceof GuardsCheckEnd) {
        console.log('Guards check ended:', {
          id: event.id,
          url: event.url,
          urlAfterRedirects: event.urlAfterRedirects,
          state: event.state,
          shouldActivate: event.shouldActivate
        });
      }

      if (event instanceof ResolveStart) {
        console.log('Resolve started:', {
          id: event.id,
          url: event.url,
          urlAfterRedirects: event.urlAfterRedirects,
          state: event.state
        });
      }

      if (event instanceof ResolveEnd) {
        console.log('Resolve ended:', {
          id: event.id,
          url: event.url,
          urlAfterRedirects: event.urlAfterRedirects,
          state: event.state
        });
      }

      if (event instanceof NavigationEnd) {
        console.log('Navigation ended successfully:', {
          id: event.id,
          url: event.url,
          urlAfterRedirects: event.urlAfterRedirects
        });
      }

      if (event instanceof NavigationCancel) {
        console.log('Navigation cancelled:', {
          id: event.id,
          url: event.url,
          reason: event.reason
        });
      }

      if (event instanceof NavigationError) {
        console.error('Navigation error:', {
          id: event.id,
          url: event.url,
          error: event.error
        });
      }
    });
  }
}
```

### Imperativ Navigation

Different ways to trigger navigation:

```typescript
@Component({
  selector: 'app-navigation-demo',
  template: `
    <button (click)="navigateAbsolute()">Absolute Navigation</button>
    <button (click)="navigateRelative()">Relative Navigation</button>
    <button (click)="navigateWithExtras()">Navigate with Extras</button>
    <button (click)="navigateByUrl()">Navigate by URL</button>
  `
})
export class NavigationDemoComponent {
  constructor(
    private router: Router,
    private route: ActivatedRoute
  ) {}

  // Absolute navigation
  navigateAbsolute(): void {
    this.router.navigate(['/products', 123, 'details']);
    // Result: /products/123/details
  }

  // Relative navigation
  navigateRelative(): void {
    this.router.navigate(['../sibling'], {
      relativeTo: this.route
    });
    // If current route is /products/123/details
    // Result: /products/123/sibling
  }

  // Navigation with query params and fragment
  navigateWithExtras(): void {
    this.router.navigate(['/search'], {
      queryParams: { q: 'angular', category: 'books' },
      fragment: 'results',
      queryParamsHandling: 'merge', // or 'preserve'
      replaceUrl: false,
      state: { customData: 'value' }
    });
    // Result: /search?q=angular&category=books#results
  }

  // Navigate by URL string
  navigateByUrl(): void {
    this.router.navigateByUrl('/products/123/details?sort=price#reviews');
  }

  // Navigate with UrlTree
  navigateWithUrlTree(): void {
    const urlTree = this.router.createUrlTree(
      ['/products', 123],
      {
        queryParams: { view: 'grid' },
        fragment: 'top'
      }
    );
    this.router.navigateByUrl(urlTree);
  }
}
```

## Route Matching Algorithm

### URL Matching Process

How the router matches URLs to routes:

```typescript
// Route configuration
const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'products', component: ProductListComponent },
  { path: 'products/:id', component: ProductDetailComponent },
  { path: 'products/:id/reviews', component: ReviewsComponent },
  { path: 'categories/:category', component: CategoryComponent },
  { path: '**', component: NotFoundComponent }
];

// Simplified route matcher
class RouteMatcher {
  match(url: string, routes: Routes): Route | null {
    const segments = this.parseUrl(url);
    
    for (const route of routes) {
      const match = this.matchRoute(segments, route);
      if (match) {
        return match;
      }
    }
    
    return null;
  }

  private matchRoute(segments: UrlSegment[], route: Route): Route | null {
    const pathSegments = route.path?.split('/') || [];

    // Check if number of segments match
    if (pathSegments.length !== segments.length) {
      // Check for wildcard
      if (route.path === '**') {
        return route;
      }
      return null;
    }

    // Match each segment
    for (let i = 0; i < pathSegments.length; i++) {
      const pathSeg = pathSegments[i];
      const urlSeg = segments[i];

      if (pathSeg.startsWith(':')) {
        // Parameter segment - always matches
        continue;
      }

      if (pathSeg !== urlSeg.path) {
        // Static segment doesn't match
        return null;
      }
    }

    return route;
  }

  private parseUrl(url: string): UrlSegment[] {
    return url.split('/').filter(s => s).map(path => ({ path }));
  }
}

// Usage example
const matcher = new RouteMatcher();
console.log(matcher.match('/products/123', routes));
// Matches: { path: 'products/:id', component: ProductDetailComponent }
```

### Custom Matchers

Implementing custom route matching:

```typescript
// Custom matcher for locale-specific routes
export function localeMatcher(
  segments: UrlSegment[],
  group: UrlSegmentGroup,
  route: Route
): UrlMatchResult | null {
  const localePattern = /^(en|es|fr|de)$/;

  if (segments.length === 0) {
    return null;
  }

  const locale = segments[0].path;

  if (!localePattern.test(locale)) {
    return null;
  }

  // Match: consume first segment as locale
  return {
    consumed: [segments[0]],
    posParams: {
      locale: new UrlSegment(locale, {})
    }
  };
}

// Route configuration with custom matcher
const routes: Routes = [
  {
    matcher: localeMatcher,
    component: LocalizedComponent,
    children: [
      { path: 'products', component: ProductsComponent },
      { path: 'about', component: AboutComponent }
    ]
  }
];

// Matches:
// /en/products -> locale='en', route='products'
// /es/about -> locale='es', route='about'
// /invalid/products -> no match
```

### Path Matching Strategies

Different matching strategies:

```typescript
const routes: Routes = [
  // Default: 'prefix' matching
  {
    path: 'admin',
    component: AdminComponent,
    children: [
      { path: 'users', component: UsersComponent },
      { path: 'settings', component: SettingsComponent }
    ]
  },
  // Matches:
  // /admin
  // /admin/users
  // /admin/settings

  // Full path matching
  {
    path: 'search',
    component: SearchComponent,
    pathMatch: 'full'
  },
  // Matches only: /search
  // Does NOT match: /search/results

  // Empty path with redirect
  {
    path: '',
    redirectTo: '/home',
    pathMatch: 'full'
  },
  // Without 'full', would redirect all routes!

  // Wildcard (must be last)
  {
    path: '**',
    component: NotFoundComponent
  }
];
```

### Route Priority

Understanding route matching priority:

```typescript
// Route order matters!
const routes: Routes = [
  // Specific routes first
  { path: 'products/new', component: NewProductComponent },
  { path: 'products/featured', component: FeaturedProductsComponent },
  
  // Then parameterized routes
  { path: 'products/:id', component: ProductDetailComponent },
  
  // Then wildcards
  { path: '**', component: NotFoundComponent }
];

// WRONG ORDER - will never reach 'new' or 'featured'
const wrongRoutes: Routes = [
  { path: 'products/:id', component: ProductDetailComponent }, // Matches 'new' and 'featured'!
  { path: 'products/new', component: NewProductComponent },    // Never reached
  { path: 'products/featured', component: FeaturedProductsComponent } // Never reached
];
```

## Guards Execution Order

### Guard Types and Execution

Understanding guard execution sequence:

```typescript
// Navigation from /home to /admin/users

// Guard execution order:
// 1. canDeactivate (current route)
// 2. canLoad (lazy loaded module)
// 3. canActivate (new route, parent to child)
// 4. canActivateChild (child routes)
// 5. resolve (data resolution)

const routes: Routes = [
  {
    path: 'admin',
    canActivate: [AdminGuard],           // 3. Check parent
    canActivateChild: [AdminChildGuard], // 4. Check before children
    children: [
      {
        path: 'users',
        component: UsersComponent,
        canActivate: [UsersGuard],       // 3. Check this route
        canDeactivate: [UnsavedChangesGuard], // 1. Check when leaving
        resolve: {
          users: UsersResolver           // 5. Resolve data
        }
      }
    ]
  },
  {
    path: 'dashboard',
    loadChildren: () => import('./dashboard/dashboard.module'),
    canLoad: [AuthGuard]                 // 2. Check before loading
  }
];
```

### CanActivate Guard

Controlling route activation:

```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}

  canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): Observable<boolean | UrlTree> | Promise<boolean | UrlTree> | boolean | UrlTree {
    console.log('AuthGuard checking:', state.url);

    return this.authService.isAuthenticated().pipe(
      map(isAuth => {
        if (isAuth) {
          return true;
        }

        // Redirect to login with return URL
        return this.router.createUrlTree(['/login'], {
          queryParams: { returnUrl: state.url }
        });
      })
    );
  }
}

// Multiple guards on one route
{
  path: 'admin',
  canActivate: [AuthGuard, RoleGuard, SubscriptionGuard],
  component: AdminComponent
}
// All guards must return true for navigation to proceed
// Short-circuits on first false/UrlTree
```

### CanDeactivate Guard

Preventing navigation away:

```typescript
export interface CanComponentDeactivate {
  canDeactivate(): Observable<boolean> | Promise<boolean> | boolean;
}

@Injectable()
export class UnsavedChangesGuard implements CanDeactivate<CanComponentDeactivate> {
  canDeactivate(
    component: CanComponentDeactivate,
    currentRoute: ActivatedRouteSnapshot,
    currentState: RouterStateSnapshot,
    nextState?: RouterStateSnapshot
  ): Observable<boolean> | Promise<boolean> | boolean {
    console.log('Checking for unsaved changes');
    console.log('Current:', currentState.url);
    console.log('Next:', nextState?.url);

    return component.canDeactivate?.() ?? true;
  }
}

// Component implementation
@Component({
  selector: 'app-form-editor',
  template: `
    <form>
      <input [(ngModel)]="formData.name" />
      <button (click)="save()">Save</button>
    </form>
  `
})
export class FormEditorComponent implements CanComponentDeactivate {
  formData = { name: '' };
  private originalData = { name: '' };

  canDeactivate(): boolean | Observable<boolean> {
    if (this.hasUnsavedChanges()) {
      return confirm('You have unsaved changes. Do you really want to leave?');
    }
    return true;
  }

  private hasUnsavedChanges(): boolean {
    return JSON.stringify(this.formData) !== JSON.stringify(this.originalData);
  }

  save(): void {
    this.originalData = { ...this.formData };
  }
}
```

### CanLoad vs CanActivate

Understanding the difference:

```typescript
// CanLoad: Prevents lazy module from loading
@Injectable()
export class AuthGuard implements CanLoad {
  constructor(private authService: AuthService) {}

  canLoad(
    route: Route,
    segments: UrlSegment[]
  ): Observable<boolean> | Promise<boolean> | boolean {
    console.log('CanLoad: Checking before loading module');
    
    // Module code is NOT downloaded if this returns false
    return this.authService.isAuthenticated();
  }
}

// CanActivate: Module loads, but route doesn't activate
@Injectable()
export class RoleGuard implements CanActivate {
  constructor(private authService: AuthService) {}

  canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): Observable<boolean> | Promise<boolean> | boolean {
    console.log('CanActivate: Checking after module loaded');
    
    // Module code IS downloaded even if this returns false
    return this.authService.hasRole('admin');
  }
}

// Usage
const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module'),
    canLoad: [AuthGuard],      // Prevents module download
    canActivate: [RoleGuard]   // Checks after module loads
  }
];

// Best practice: Use CanLoad for initial authentication check
// Use CanActivate for role/permission checks
```

### Resolve Guards

Pre-fetching data before activation:

```typescript
@Injectable()
export class UserResolver implements Resolve<User> {
  constructor(private userService: UserService) {}

  resolve(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): Observable<User> | Promise<User> | User {
    const userId = route.paramMap.get('id')!;
    
    console.log('Resolving user data for:', userId);

    return this.userService.getUser(userId).pipe(
      catchError(error => {
        console.error('Failed to resolve user:', error);
        // Return empty user or redirect
        return of(null as any);
      })
    );
  }
}

// Multiple resolvers
const routes: Routes = [
  {
    path: 'users/:id',
    component: UserProfileComponent,
    resolve: {
      user: UserResolver,
      posts: UserPostsResolver,
      stats: UserStatsResolver
    }
  }
];

// Component receives resolved data
@Component({
  selector: 'app-user-profile',
  template: `
    <h1>{{ user.name }}</h1>
    <p>Posts: {{ posts.length }}</p>
    <p>Followers: {{ stats.followers }}</p>
  `
})
export class UserProfileComponent implements OnInit {
  user!: User;
  posts!: Post[];
  stats!: UserStats;

  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    // Data is already resolved and available
    this.route.data.subscribe(data => {
      this.user = data['user'];
      this.posts = data['posts'];
      this.stats = data['stats'];
    });
  }
}
```

### Guard Execution Flow

Complete guard execution sequence:

```typescript
class GuardExecutor {
  async executeGuards(
    currentState: RouterStateSnapshot,
    nextState: RouterStateSnapshot
  ): Promise<boolean | UrlTree> {
    // 1. CanDeactivate guards (current routes)
    const canDeactivate = await this.runCanDeactivateGuards(currentState);
    if (canDeactivate !== true) {
      return canDeactivate;
    }

    // 2. CanLoad guards (lazy modules)
    const canLoad = await this.runCanLoadGuards(nextState);
    if (canLoad !== true) {
      return canLoad;
    }

    // 3. CanActivate guards (parent to child)
    const canActivate = await this.runCanActivateGuards(nextState);
    if (canActivate !== true) {
      return canActivate;
    }

    // 4. CanActivateChild guards
    const canActivateChild = await this.runCanActivateChildGuards(nextState);
    if (canActivateChild !== true) {
      return canActivateChild;
    }

    return true;
  }

  private async runCanDeactivateGuards(
    state: RouterStateSnapshot
  ): Promise<boolean | UrlTree> {
    // Execute all canDeactivate guards
    // Short-circuit on first false/UrlTree
    return true;
  }

  private async runCanLoadGuards(
    state: RouterStateSnapshot
  ): Promise<boolean | UrlTree> {
    // Execute canLoad for lazy modules
    return true;
  }

  private async runCanActivateGuards(
    state: RouterStateSnapshot
  ): Promise<boolean | UrlTree> {
    // Execute from root to leaf
    return true;
  }

  private async runCanActivateChildGuards(
    state: RouterStateSnapshot
  ): Promise<boolean | UrlTree> {
    // Execute for child routes
    return true;
  }
}
```

## Route Resolution

### Data Resolvers

Pre-loading route data:

```typescript
// Complex resolver with dependencies
@Injectable()
export class ProductDetailsResolver implements Resolve<ProductDetails> {
  constructor(
    private productService: ProductService,
    private reviewService: ReviewService,
    private inventoryService: InventoryService
  ) {}

  resolve(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): Observable<ProductDetails> {
    const productId = route.paramMap.get('id')!;

    // Parallel data loading
    return forkJoin({
      product: this.productService.getProduct(productId),
      reviews: this.reviewService.getReviews(productId),
      inventory: this.inventoryService.getInventory(productId)
    }).pipe(
      map(({ product, reviews, inventory }) => ({
        product,
        reviews,
        inventory,
        averageRating: this.calculateAverage(reviews)
      })),
      catchError(error => {
        console.error('Resolution failed:', error);
        // Fallback or redirect
        return of(null as any);
      })
    );
  }

  private calculateAverage(reviews: Review[]): number {
    if (reviews.length === 0) return 0;
    const sum = reviews.reduce((acc, r) => acc + r.rating, 0);
    return sum / reviews.length;
  }
}
```

### Route Data

Static and dynamic route data:

```typescript
const routes: Routes = [
  {
    path: 'products',
    component: ProductsComponent,
    data: {
      title: 'Products',
      breadcrumb: 'Products',
      roles: ['user', 'admin']
    }
  },
  {
    path: 'products/:id',
    component: ProductDetailComponent,
    data: {
      title: 'Product Details',
      animation: 'detail'
    },
    resolve: {
      product: ProductResolver
    }
  }
];

// Accessing route data
@Component({
  selector: 'app-products',
  template: `<h1>{{ title }}</h1>`
})
export class ProductsComponent implements OnInit {
  title = '';

  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    // Access static data
    this.route.data.subscribe(data => {
      this.title = data['title'];
      console.log('Breadcrumb:', data['breadcrumb']);
      console.log('Required roles:', data['roles']);
    });

    // Access resolved data
    this.route.data.subscribe(data => {
      const product = data['product'];
      console.log('Resolved product:', product);
    });

    // Access parent route data
    let parent = this.route.parent;
    while (parent) {
      parent.data.subscribe(data => {
        console.log('Parent data:', data);
      });
      parent = parent.parent;
    }
  }
}
```

## URL Serialization

### URL Parsing

How URLs are parsed:

```typescript
class UrlParser {
  parse(url: string): UrlTree {
    // Parse: /products/123;color=red?sort=price&order=asc#reviews

    const [pathAndMatrix, queryAndFragment] = url.split('?');
    const [path, fragment] = (queryAndFragment || '').split('#');

    // Parse segments and matrix params
    const segments = this.parseSegments(pathAndMatrix);

    // Parse query params
    const queryParams = this.parseQueryParams(path);

    return {
      root: new UrlSegmentGroup(segments, {}),
      queryParams,
      fragment: fragment || null
    };
  }

  private parseSegments(path: string): UrlSegment[] {
    return path.split('/').filter(s => s).map(segment => {
      // Split matrix params: products;color=red;size=large
      const [path, ...matrixParts] = segment.split(';');
      
      const parameters: { [key: string]: string } = {};
      matrixParts.forEach(part => {
        const [key, value] = part.split('=');
        parameters[key] = value;
      });

      return new UrlSegment(path, parameters);
    });
  }

  private parseQueryParams(query: string): Params {
    if (!query) return {};

    const params: Params = {};
    query.split('&').forEach(part => {
      const [key, value] = part.split('=');
      params[decodeURIComponent(key)] = decodeURIComponent(value);
    });

    return params;
  }
}
```

### Matrix Parameters

Using matrix parameters:

```typescript
@Component({
  selector: 'app-product-list',
  template: `
    <div *ngFor="let product of products">
      <!-- Navigate with matrix params -->
      <a [routerLink]="['/products', product.id, {color: 'red', size: 'large'}]">
        {{ product.name }}
      </a>
    </div>
  `
})
export class ProductListComponent {
  products = [/* ... */];

  navigateWithMatrixParams(): void {
    // Programmatic navigation with matrix params
    this.router.navigate([
      '/products',
      123,
      { color: 'red', size: 'large' }
    ]);
    // Result: /products/123;color=red;size=large
  }
}

// Reading matrix params
@Component({
  selector: 'app-product-detail',
  template: `
    <h1>Product {{ productId }}</h1>
    <p>Color: {{ color }}</p>
    <p>Size: {{ size }}</p>
  `
})
export class ProductDetailComponent implements OnInit {
  productId = '';
  color = '';
  size = '';

  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    this.route.paramMap.subscribe(params => {
      this.productId = params.get('id')!;
      this.color = params.get('color') || 'default';
      this.size = params.get('size') || 'medium';
    });
  }
}

// Matrix params vs Query params:
// Matrix params: /products/123;color=red    (scoped to segment)
// Query params:  /products/123?color=red    (global to URL)
```

### Custom URL Serializer

Implementing custom serialization:

```typescript
@Injectable()
export class CustomUrlSerializer implements UrlSerializer {
  parse(url: string): UrlTree {
    // Custom parsing logic
    // Example: Handle legacy URL format
    console.log('Parsing URL:', url);
    
    // Delegate to default serializer
    const defaultSerializer = new DefaultUrlSerializer();
    return defaultSerializer.parse(url);
  }

  serialize(tree: UrlTree): string {
    // Custom serialization logic
    // Example: Add custom formatting
    console.log('Serializing UrlTree:', tree);
    
    const defaultSerializer = new DefaultUrlSerializer();
    let url = defaultSerializer.serialize(tree);

    // Custom modifications
    // Example: Lowercase all URLs
    url = url.toLowerCase();

    return url;
  }
}

// Provide custom serializer
@NgModule({
  providers: [
    { provide: UrlSerializer, useClass: CustomUrlSerializer }
  ]
})
export class AppModule {}
```

## Router State Management

### Router State Tree

Understanding the state structure:

```typescript
// Router state snapshot
interface RouterStateSnapshot {
  url: string;
  root: ActivatedRouteSnapshot;
}

// Activated route snapshot
interface ActivatedRouteSnapshot {
  url: UrlSegment[];
  params: Params;
  queryParams: Params;
  fragment: string | null;
  data: Data;
  outlet: string;
  component: Type<any> | string | null;
  routeConfig: Route | null;
  root: ActivatedRouteSnapshot;
  parent: ActivatedRouteSnapshot | null;
  firstChild: ActivatedRouteSnapshot | null;
  children: ActivatedRouteSnapshot[];
  pathFromRoot: ActivatedRouteSnapshot[];
}

// Accessing router state
@Injectable()
export class RouterStateService {
  constructor(private router: Router) {
    this.logRouterState();
  }

  private logRouterState(): void {
    this.router.events.pipe(
      filter(event => event instanceof NavigationEnd)
    ).subscribe(() => {
      const state = this.router.routerState;
      const snapshot = state.snapshot;

      console.log('Current URL:', snapshot.url);
      console.log('Root route:', snapshot.root);
      this.logRouteTree(snapshot.root, 0);
    });
  }

  private logRouteTree(route: ActivatedRouteSnapshot, depth: number): void {
    const indent = '  '.repeat(depth);
    console.log(`${indent}Route:`, {
      path: route.routeConfig?.path,
      component: route.component,
      params: route.params,
      data: route.data
    });

    route.children.forEach(child => {
      this.logRouteTree(child, depth + 1);
    });
  }
}
```

### Route Parameters

Accessing route parameters:

```typescript
@Component({
  selector: 'app-demo',
  template: `
    <p>User ID: {{ userId }}</p>
    <p>Tab: {{ tab }}</p>
    <p>Query: {{ searchQuery }}</p>
    <p>Fragment: {{ fragment }}</p>
  `
})
export class ParameterDemoComponent implements OnInit {
  userId = '';
  tab = '';
  searchQuery = '';
  fragment = '';

  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    // URL: /users/123/posts;tab=recent?search=angular#comments

    // Path parameters
    this.route.paramMap.subscribe(params => {
      this.userId = params.get('id') || '';
      console.log('All params:', params.keys.map(k => `${k}=${params.get(k)}`));
    });

    // Matrix parameters
    this.route.paramMap.subscribe(params => {
      this.tab = params.get('tab') || 'all';
    });

    // Query parameters
    this.route.queryParamMap.subscribe(params => {
      this.searchQuery = params.get('search') || '';
      console.log('All query params:', params.keys.map(k => `${k}=${params.get(k)}`));
    });

    // Fragment
    this.route.fragment.subscribe(fragment => {
      this.fragment = fragment || '';
      if (fragment) {
        this.scrollToFragment(fragment);
      }
    });

    // Combined snapshot (not reactive)
    const snapshot = this.route.snapshot;
    console.log('Snapshot params:', snapshot.params);
    console.log('Snapshot query:', snapshot.queryParams);
    console.log('Snapshot fragment:', snapshot.fragment);
  }

  private scrollToFragment(fragment: string): void {
    const element = document.getElementById(fragment);
    element?.scrollIntoView({ behavior: 'smooth' });
  }
}
```

## Common Misconceptions

### Misconception 1: "Guards can be async without observables"

**Reality**: Guards must return Observable, Promise, boolean, or UrlTree. Plain async functions need to return Promise.

```typescript
// WRONG - Returns Promise<Observable<boolean>>
async canActivate(): Observable<boolean> {
  return this.authService.isAuthenticated();
}

// CORRECT - Returns Observable
canActivate(): Observable<boolean> {
  return this.authService.isAuthenticated();
}

// OR - Returns Promise
async canActivate(): Promise<boolean> {
  return firstValueFrom(this.authService.isAuthenticated());
}
```

### Misconception 2: "Route order doesn't matter"

**Reality**: Router matches first route that fits, so order is critical.

```typescript
// WRONG - specific routes after generic
const wrongRoutes: Routes = [
  { path: 'products/:id', component: DetailComponent },
  { path: 'products/new', component: NewComponent } // Never reached!
];

// CORRECT - specific routes first
const correctRoutes: Routes = [
  { path: 'products/new', component: NewComponent },
  { path: 'products/:id', component: DetailComponent }
];
```

### Misconception 3: "CanDeactivate runs before navigation starts"

**Reality**: CanDeactivate runs during navigation, after user initiates it.

```typescript
// This doesn't prevent the navigation from starting
// It runs DURING navigation and can cancel it
@Injectable()
export class UnsavedChangesGuard implements CanDeactivate<any> {
  canDeactivate(component: any): boolean {
    // Navigation already started when this runs
    return component.hasUnsavedChanges() 
      ? confirm('Leave without saving?')
      : true;
  }
}
```

## Performance Implications

### Lazy Loading Impact

Optimizing with lazy routes:

```typescript
// Before: Everything in main bundle
const routes: Routes = [
  { path: 'home', component: HomeComponent },
  { path: 'products', component: ProductsComponent },
  { path: 'admin', component: AdminComponent }
];
// Result: main.js = 2 MB

// After: Lazy loading
const routes: Routes = [
  { path: 'home', component: HomeComponent },
  {
    path: 'products',
    loadChildren: () => import('./products/products.module')
      .then(m => m.ProductsModule)
  },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module')
      .then(m => m.AdminModule)
  }
];
// Result:
// main.js = 500 KB (75% reduction!)
// products.js = 800 KB (loaded on demand)
// admin.js = 700 KB (loaded on demand)
```

### Preloading Strategies

Optimizing lazy module loading:

```typescript
// Custom preloading strategy
@Injectable()
export class CustomPreloadingStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    // Preload if route has preload:true in data
    if (route.data && route.data['preload']) {
      console.log('Preloading:', route.path);
      return load();
    }
    
    return of(null);
  }
}

// Route configuration
const routes: Routes = [
  {
    path: 'products',
    loadChildren: () => import('./products/products.module'),
    data: { preload: true } // Will preload
  },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module')
    // Will NOT preload
  }
];

// Module configuration
@NgModule({
  imports: [
    RouterModule.forRoot(routes, {
      preloadingStrategy: CustomPreloadingStrategy
    })
  ],
  providers: [CustomPreloadingStrategy]
})
export class AppModule {}
```

### Guard Performance

Optimizing guard execution:

```typescript
// BAD - Sequential guard execution
@Injectable()
export class SlowGuard implements CanActivate {
  async canActivate(): Promise<boolean> {
    await this.check1(); // 500ms
    await this.check2(); // 500ms
    await this.check3(); // 500ms
    return true; // Total: 1500ms
  }
}

// GOOD - Parallel guard execution
@Injectable()
export class FastGuard implements CanActivate {
  canActivate(): Observable<boolean> {
    return forkJoin({
      check1: this.check1(),
      check2: this.check2(),
      check3: this.check3()
    }).pipe(
      map(results => 
        results.check1 && results.check2 && results.check3
      )
    );
    // Total: 500ms (parallel execution)
  }
}
```

## Interview Questions

### Question 1: Explain the complete navigation cycle in Angular Router.

**Answer**: Navigation follows these phases:

1. **ApplyRedirects**: Process redirect configurations
2. **Recognize**: Match URL to route configuration
3. **RunGuards**: Execute navigation guards (canDeactivate → canLoad → canActivate → canActivateChild)
4. **ResolveData**: Run resolvers to fetch data
5. **ActivateRoutes**: Create/update component instances
6. **Complete**: Finalize navigation and update browser URL

Each phase can cancel navigation if it fails.

### Question 2: What's the difference between CanLoad and CanActivate guards?

**Answer**:

**CanLoad**:
- Runs before lazy module loads
- Prevents module code from downloading
- Can't access route parameters
- Good for authentication checks

**CanActivate**:
- Runs after module loads
- Module code already downloaded
- Can access route parameters
- Good for authorization/role checks

Use CanLoad for initial auth, CanActivate for permissions.

### Question 3: How does the router match URLs to routes?

**Answer**: The router:

1. Parses URL into segments
2. Iterates routes in order (first match wins)
3. Compares each segment:
   - Static segments must match exactly
   - `:param` segments match any value
   - `**` wildcard matches remaining path
4. Returns first matching route

Route order matters: specific routes before generic, wildcard last.

### Question 4: What's the execution order of multiple guards?

**Answer**: For navigation from `/a` to `/b`:

1. CanDeactivate (component at `/a`)
2. CanLoad (if `/b` is lazy loaded)
3. CanActivate (parent routes first, then children)
4. CanActivateChild (child routes)
5. Resolve (data resolvers)

All guards must return true/UrlTree. First false/UrlTree cancels navigation.

### Question 5: How do you pass data between routes?

**Answer**: Multiple methods:

1. **Route parameters**: `/products/:id`
2. **Query parameters**: `?search=term`
3. **Route data**: Static config
4. **Router state**: `state: { data: value }`
5. **Resolvers**: Pre-fetched data
6. **Services**: Shared state

```typescript
// Navigate with state
this.router.navigate(['/details'], {
  state: { product: selectedProduct }
});

// Read state
const state = this.router.getCurrentNavigation()?.extras.state;
```

### Question 6: What are matrix parameters and when should you use them?

**Answer**: Matrix parameters are segment-scoped parameters:

```typescript
// Matrix params: /products/123;color=red;size=large
// Query params:  /products/123?color=red&size=large
```

**Use matrix params when**:
- Parameters are specific to a segment
- Multiple sibling outlets need different params
- You want cleaner URLs for pagination/filtering

**Use query params when**:
- Parameters apply globally (search, filters)
- Sharing URLs with params

### Question 7: How do resolvers affect navigation performance?

**Answer**: Resolvers delay navigation until data loads:

**Pros**:
- Component receives data immediately
- No loading state needed
- Consistent user experience

**Cons**:
- Navigation feels slower
- User waits before seeing anything
- All resolvers must complete

**Optimization**:
- Use parallel resolvers (forkJoin)
- Load only essential data
- Consider loading states instead

```typescript
// Fast: parallel resolvers
resolve: {
  user: UserResolver,
  posts: PostsResolver
}
// Both load simultaneously

// Slow: dependent resolvers
// Load posts after user data arrives
```

### Question 8: How do you implement a custom route matching strategy?

**Answer**: Implement a custom matcher:

```typescript
export function customMatcher(
  segments: UrlSegment[],
  group: UrlSegmentGroup,
  route: Route
): UrlMatchResult | null {
  // Custom logic to match segments
  if (matchesCustomPattern(segments)) {
    return {
      consumed: segments,
      posParams: {
        param: new UrlSegment('value', {})
      }
    };
  }
  return null;
}

// Use in route config
{
  matcher: customMatcher,
  component: MyComponent
}
```

Use cases: locale prefixes, legacy URL formats, complex patterns.

## Key Takeaways

1. **Navigation is multi-phase**: ApplyRedirects → Recognize → Guards → Resolve → Activate → Complete
2. **Route order matters**: First match wins, so specific routes before generic
3. **Guards execute in order**: canDeactivate → canLoad → canActivate → canActivateChild → resolve
4. **CanLoad prevents downloads**: Use for auth, CanActivate for permissions
5. **Resolvers block navigation**: Use sparingly, prefer loading states for better UX
6. **Multiple parameter types**: Path params, matrix params, query params, fragment
7. **Lazy loading improves performance**: Dramatically reduces initial bundle size
8. **Router state is hierarchical**: Tree structure matches route configuration

## Resources

### Official Documentation
- [Angular Router Guide](https://angular.dev/guide/routing)
- [Router API](https://angular.dev/api/router/Router)
- [Route Guards](https://angular.dev/guide/routing/common-router-tasks#preventing-unauthorized-access)

### Articles & Tutorials
- "Angular Router In Depth" by Victor Savkin
- "Understanding Angular Router" by Thoughtram
- "Router Navigation Cycle" by Angular University

### Videos
- Angular Router Deep Dive
- Navigation Guards Explained
- Lazy Loading Best Practices

### Books
- "Router Chapter" in ng-book
- "Angular Router" by Victor Savkin

### Source Code
- [Angular Router Implementation](https://github.com/angular/angular/tree/main/packages/router)
- [Router Tests](https://github.com/angular/angular/tree/main/packages/router/test)
