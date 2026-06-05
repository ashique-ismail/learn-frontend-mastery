# Route Parameters in Angular

## The Idea

**In plain English:** Route parameters are pieces of information you can embed directly inside a web address (URL) so a page knows which specific item to show — for example, which user profile or which product detail to display. A "parameter" here just means a named slot in the URL that holds a changeable value.

**Real-world analogy:** Think of a library's shelf system where every book has a unique call number written on its spine, like "FIC-0042". When a librarian looks up "FIC-0042", they go straight to that exact book. The URL works the same way: instead of walking shelves, the app reads the number from the address and fetches the right content.

- The shelf section label (FIC) = the fixed part of the route path (e.g., `/users/`)
- The call number on the spine (0042) = the route parameter value (e.g., `42` in `/users/42`)
- The librarian looking up the book = the Angular component reading `ActivatedRoute` to fetch data

---

## Table of Contents

- [Introduction](#introduction)
- [Route Parameters](#route-parameters)
- [Query Parameters](#query-parameters)
- [Fragments](#fragments)
- [ActivatedRoute Overview](#activatedroute-overview)
- [Snapshot vs Observable](#snapshot-vs-observable)
- [Parameter Maps](#parameter-maps)
- [Matrix URL Notation](#matrix-url-notation)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Route parameters allow you to pass data through URLs in Angular applications. Understanding the different types of parameters and how to access them is crucial for building dynamic, data-driven applications.

## Route Parameters

### Basic Route Parameters

```typescript
// app.routes.ts
import { Routes } from '@angular/router';
import { UserDetailComponent } from './user-detail.component';

export const routes: Routes = [
  { path: 'users/:id', component: UserDetailComponent },
  { path: 'posts/:postId/comments/:commentId', component: CommentDetailComponent },
  { path: 'products/:category/:id', component: ProductDetailComponent }
];

// user-detail.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute } from '@angular/router';

@Component({
  selector: 'app-user-detail',
  standalone: true,
  template: `<div>User ID: {{ userId }}</div>`
})
export class UserDetailComponent implements OnInit {
  userId: string | null = null;

  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    // Using snapshot for one-time read
    this.userId = this.route.snapshot.paramMap.get('id');
  }
}
```

### Multiple Route Parameters

```typescript
// product-detail.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute } from '@angular/router';

@Component({
  selector: 'app-product-detail',
  standalone: true,
  template: `
    <div>
      <h2>Product Details</h2>
      <p>Category: {{ category }}</p>
      <p>Product ID: {{ productId }}</p>
    </div>
  `
})
export class ProductDetailComponent implements OnInit {
  category: string | null = null;
  productId: string | null = null;

  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    const params = this.route.snapshot.paramMap;
    this.category = params.get('category');
    this.productId = params.get('id');
  }
}
```

### Accessing All Parameters at Once

```typescript
// route-params-example.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute, Params } from '@angular/router';

@Component({
  selector: 'app-route-params-example',
  standalone: true,
  template: `<pre>{{ allParams | json }}</pre>`
})
export class RouteParamsExampleComponent implements OnInit {
  allParams: Params = {};

  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    // Get all parameters as object
    this.allParams = this.route.snapshot.params;
    
    // Or get all keys
    const paramMap = this.route.snapshot.paramMap;
    const keys = paramMap.keys;
    
    keys.forEach(key => {
      console.log(`${key}: ${paramMap.get(key)}`);
    });
  }
}
```

### Type-Safe Route Parameters

```typescript
// route-params.types.ts
export interface UserRouteParams {
  id: string;
}

export interface ProductRouteParams {
  category: string;
  id: string;
}

// user-detail.component.ts
import { Component, OnInit, signal } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { UserRouteParams } from './route-params.types';

@Component({
  selector: 'app-user-detail',
  standalone: true,
  template: `<div>User ID: {{ userId() }}</div>`
})
export class UserDetailComponent implements OnInit {
  userId = signal<string>('');

  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    const params = this.route.snapshot.paramMap;
    const id = params.get('id');
    
    if (id) {
      this.userId.set(id);
      this.loadUserData(id);
    }
  }

  private loadUserData(id: string): void {
    // Type-safe ID usage
    const numericId = parseInt(id, 10);
    // Fetch user data
  }
}
```

## Query Parameters

### Basic Query Parameters

```typescript
// search.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute } from '@angular/router';

@Component({
  selector: 'app-search',
  standalone: true,
  template: `
    <div>
      <h2>Search Results</h2>
      <p>Query: {{ searchQuery }}</p>
      <p>Category: {{ category }}</p>
      <p>Page: {{ page }}</p>
    </div>
  `
})
export class SearchComponent implements OnInit {
  searchQuery: string | null = null;
  category: string | null = null;
  page: number = 1;

  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    // Access query parameters
    const queryParams = this.route.snapshot.queryParamMap;
    
    this.searchQuery = queryParams.get('q');
    this.category = queryParams.get('category');
    
    const pageParam = queryParams.get('page');
    this.page = pageParam ? parseInt(pageParam, 10) : 1;
  }
}
```

### Setting Query Parameters

```typescript
// product-list.component.ts
import { Component } from '@angular/core';
import { Router } from '@angular/router';
import { RouterLink } from '@angular/router';

@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [RouterLink],
  template: `
    <div>
      <!-- Declarative with RouterLink -->
      <a 
        routerLink="/products"
        [queryParams]="{ category: 'electronics', sort: 'price' }">
        Electronics
      </a>
      
      <!-- Programmatic -->
      <button (click)="filterProducts('books')">Books</button>
      <button (click)="sortBy('date')">Sort by Date</button>
    </div>
  `
})
export class ProductListComponent {
  constructor(private router: Router) {}

  filterProducts(category: string): void {
    this.router.navigate(['/products'], {
      queryParams: { category }
    });
  }

  sortBy(sort: string): void {
    this.router.navigate([], {
      queryParams: { sort },
      queryParamsHandling: 'merge' // Preserve other params
    });
  }
}
```

### Query Parameter Handling

```typescript
// filter.component.ts
import { Component, OnInit } from '@angular/core';
import { Router, ActivatedRoute } from '@angular/router';

@Component({
  selector: 'app-filter',
  standalone: true,
  template: `
    <div class="filters">
      <button (click)="addFilter('new')">Add Filter</button>
      <button (click)="replaceFilters()">Replace All</button>
      <button (click)="clearFilters()">Clear</button>
    </div>
  `
})
export class FilterComponent implements OnInit {
  constructor(
    private router: Router,
    private route: ActivatedRoute
  ) {}

  ngOnInit(): void {
    // Read current query params
    console.log(this.route.snapshot.queryParams);
  }

  // Merge with existing params
  addFilter(filter: string): void {
    this.router.navigate([], {
      relativeTo: this.route,
      queryParams: { filter },
      queryParamsHandling: 'merge'
    });
  }

  // Replace all params
  replaceFilters(): void {
    this.router.navigate([], {
      relativeTo: this.route,
      queryParams: { filter: 'all', page: 1 }
      // Default behavior replaces all params
    });
  }

  // Clear all query params
  clearFilters(): void {
    this.router.navigate([], {
      relativeTo: this.route,
      queryParams: {}
    });
  }
}
```

### Complex Query Parameters

```typescript
// advanced-search.component.ts
import { Component, OnInit } from '@angular/core';
import { Router, ActivatedRoute } from '@angular/router';

interface SearchFilters {
  query: string;
  categories: string[];
  priceMin: number;
  priceMax: number;
  sortBy: string;
  page: number;
}

@Component({
  selector: 'app-advanced-search',
  standalone: true,
  template: `<div>Advanced Search</div>`
})
export class AdvancedSearchComponent implements OnInit {
  filters: SearchFilters = {
    query: '',
    categories: [],
    priceMin: 0,
    priceMax: 1000,
    sortBy: 'relevance',
    page: 1
  };

  constructor(
    private router: Router,
    private route: ActivatedRoute
  ) {}

  ngOnInit(): void {
    this.parseQueryParams();
  }

  parseQueryParams(): void {
    const params = this.route.snapshot.queryParamMap;
    
    this.filters = {
      query: params.get('q') || '',
      categories: params.getAll('category'),
      priceMin: parseInt(params.get('minPrice') || '0', 10),
      priceMax: parseInt(params.get('maxPrice') || '1000', 10),
      sortBy: params.get('sort') || 'relevance',
      page: parseInt(params.get('page') || '1', 10)
    };
  }

  applyFilters(filters: Partial<SearchFilters>): void {
    const queryParams: any = {
      q: filters.query,
      category: filters.categories,
      minPrice: filters.priceMin,
      maxPrice: filters.priceMax,
      sort: filters.sortBy,
      page: filters.page
    };

    this.router.navigate([], {
      relativeTo: this.route,
      queryParams,
      queryParamsHandling: 'merge'
    });
  }
}
```

## Fragments

### Using URL Fragments

```typescript
// documentation.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';
import { RouterLink } from '@angular/router';

@Component({
  selector: 'app-documentation',
  standalone: true,
  imports: [RouterLink],
  template: `
    <nav>
      <a routerLink="/docs" fragment="intro">Introduction</a>
      <a routerLink="/docs" fragment="getting-started">Getting Started</a>
      <a routerLink="/docs" fragment="advanced">Advanced</a>
    </nav>
    
    <div>
      <section id="intro">Introduction content</section>
      <section id="getting-started">Getting Started content</section>
      <section id="advanced">Advanced content</section>
    </div>
    
    <button (click)="scrollToSection('advanced')">Jump to Advanced</button>
  `
})
export class DocumentationComponent implements OnInit {
  constructor(
    private route: ActivatedRoute,
    private router: Router
  ) {}

  ngOnInit(): void {
    // Read fragment
    const fragment = this.route.snapshot.fragment;
    if (fragment) {
      this.scrollToElement(fragment);
    }
  }

  scrollToSection(section: string): void {
    this.router.navigate([], {
      relativeTo: this.route,
      fragment: section
    });
    
    this.scrollToElement(section);
  }

  private scrollToElement(elementId: string): void {
    setTimeout(() => {
      const element = document.getElementById(elementId);
      element?.scrollIntoView({ behavior: 'smooth', block: 'start' });
    }, 100);
  }
}
```

### Fragment with Query Parameters

```typescript
// article.component.ts
import { Component } from '@angular/core';
import { Router } from '@angular/router';
import { RouterLink } from '@angular/router';

@Component({
  selector: 'app-article',
  standalone: true,
  imports: [RouterLink],
  template: `
    <a 
      routerLink="/articles/123"
      [queryParams]="{ highlight: 'true' }"
      fragment="comments">
      Jump to Comments
    </a>
    
    <button (click)="navigateToSection()">
      Navigate to Section
    </button>
  `
})
export class ArticleComponent {
  constructor(private router: Router) {}

  navigateToSection(): void {
    this.router.navigate(['/articles', 123], {
      queryParams: { highlight: 'true', theme: 'dark' },
      fragment: 'comments'
    });
  }
}
```

## ActivatedRoute Overview

### ActivatedRoute Properties

```typescript
// activated-route-demo.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute, Data, Params } from '@angular/router';
import { Observable } from 'rxjs';

@Component({
  selector: 'app-activated-route-demo',
  standalone: true,
  template: `<div>Route Information Demo</div>`
})
export class ActivatedRouteDemoComponent implements OnInit {
  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    // URL segments
    console.log('URL:', this.route.snapshot.url);
    
    // Route parameters (Observable)
    this.route.params.subscribe(params => {
      console.log('Params changed:', params);
    });
    
    // Query parameters (Observable)
    this.route.queryParams.subscribe(queryParams => {
      console.log('Query params changed:', queryParams);
    });
    
    // Fragment (Observable)
    this.route.fragment.subscribe(fragment => {
      console.log('Fragment changed:', fragment);
    });
    
    // Route data
    this.route.data.subscribe(data => {
      console.log('Route data:', data);
    });
    
    // Combined observable
    this.route.paramMap.subscribe(paramMap => {
      console.log('Param map:', paramMap);
    });
    
    // Parent route
    console.log('Parent route:', this.route.parent);
    
    // Child routes
    console.log('Children:', this.route.children);
    
    // First child
    console.log('First child:', this.route.firstChild);
  }
}
```

### Accessing Parent Route Parameters

```typescript
// app.routes.ts
export const routes: Routes = [
  {
    path: 'users/:userId',
    component: UserComponent,
    children: [
      { path: 'posts/:postId', component: PostComponent }
    ]
  }
];

// post.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute } from '@angular/router';

@Component({
  selector: 'app-post',
  standalone: true,
  template: `
    <div>
      <p>User ID: {{ userId }}</p>
      <p>Post ID: {{ postId }}</p>
    </div>
  `
})
export class PostComponent implements OnInit {
  userId: string | null = null;
  postId: string | null = null;

  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    // Current route param
    this.postId = this.route.snapshot.paramMap.get('postId');
    
    // Parent route param
    this.userId = this.route.parent?.snapshot.paramMap.get('userId') || null;
  }
}
```

## Snapshot vs Observable

### When to Use Snapshot

```typescript
// product-detail-snapshot.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute } from '@angular/router';

// Use snapshot when:
// 1. Component is destroyed and recreated on navigation
// 2. You only need the initial value
// 3. Route params won't change while component is active

@Component({
  selector: 'app-product-detail-snapshot',
  standalone: true,
  template: `<div>Product: {{ productId }}</div>`
})
export class ProductDetailSnapshotComponent implements OnInit {
  productId: string | null = null;

  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    // One-time read - perfect for this use case
    this.productId = this.route.snapshot.paramMap.get('id');
    this.loadProduct(this.productId);
  }

  private loadProduct(id: string | null): void {
    // Load product data
  }
}
```

### When to Use Observable

```typescript
// search-results.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { Subject, takeUntil } from 'rxjs';

// Use observable when:
// 1. Component is reused on navigation
// 2. Route params can change without destroying component
// 3. Need to react to parameter changes

@Component({
  selector: 'app-search-results',
  standalone: true,
  template: `
    <div>
      <h2>Search Results for "{{ searchQuery }}"</h2>
      <div>Page: {{ currentPage }}</div>
    </div>
  `
})
export class SearchResultsComponent implements OnInit, OnDestroy {
  searchQuery: string = '';
  currentPage: number = 1;
  private destroy$ = new Subject<void>();

  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    // Subscribe to query param changes
    this.route.queryParams
      .pipe(takeUntil(this.destroy$))
      .subscribe(params => {
        this.searchQuery = params['q'] || '';
        this.currentPage = parseInt(params['page'] || '1', 10);
        this.performSearch();
      });
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }

  private performSearch(): void {
    console.log(`Searching for "${this.searchQuery}" on page ${this.currentPage}`);
    // Perform search
  }
}
```

### Observable with Signal Integration

```typescript
// modern-search.component.ts
import { Component, OnInit, signal } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { toSignal } from '@angular/core/rxjs-interop';
import { map } from 'rxjs/operators';

@Component({
  selector: 'app-modern-search',
  standalone: true,
  template: `
    <div>
      <h2>Searching: {{ searchQuery() }}</h2>
      <p>Category: {{ category() }}</p>
      <p>Page: {{ page() }}</p>
    </div>
  `
})
export class ModernSearchComponent {
  // Convert observables to signals
  private queryParams = toSignal(this.route.queryParams, { initialValue: {} });
  
  searchQuery = computed(() => this.queryParams()['q'] || '');
  category = computed(() => this.queryParams()['category'] || 'all');
  page = computed(() => parseInt(this.queryParams()['page'] || '1', 10));

  constructor(private route: ActivatedRoute) {
    // Signals automatically track changes
    effect(() => {
      console.log('Search updated:', this.searchQuery(), this.page());
      this.performSearch();
    });
  }

  private performSearch(): void {
    // Search logic using signal values
  }
}
```

### Combining Multiple Observables

```typescript
// combined-params.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { combineLatest } from 'rxjs';
import { map } from 'rxjs/operators';

interface RouteState {
  userId: string | null;
  queryParams: any;
  fragment: string | null;
}

@Component({
  selector: 'app-combined-params',
  standalone: true,
  template: `<div>Combined Route State</div>`
})
export class CombinedParamsComponent implements OnInit {
  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    // Combine multiple route observables
    combineLatest([
      this.route.paramMap,
      this.route.queryParamMap,
      this.route.fragment
    ])
    .pipe(
      map(([paramMap, queryParamMap, fragment]) => ({
        userId: paramMap.get('id'),
        queryParams: this.convertToObject(queryParamMap),
        fragment
      }))
    )
    .subscribe((state: RouteState) => {
      console.log('Complete route state:', state);
      this.updateView(state);
    });
  }

  private convertToObject(paramMap: any): any {
    const obj: any = {};
    paramMap.keys.forEach((key: string) => {
      obj[key] = paramMap.get(key);
    });
    return obj;
  }

  private updateView(state: RouteState): void {
    // Update component based on complete state
  }
}
```

## Parameter Maps

### ParamMap API

```typescript
// param-map-demo.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute, ParamMap } from '@angular/router';

@Component({
  selector: 'app-param-map-demo',
  standalone: true,
  template: `<div>ParamMap API Demo</div>`
})
export class ParamMapDemoComponent implements OnInit {
  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    const paramMap: ParamMap = this.route.snapshot.paramMap;
    
    // Get single parameter
    const id = paramMap.get('id');
    console.log('ID:', id);
    
    // Get all values for a parameter (useful for multi-select)
    const tags = paramMap.getAll('tag');
    console.log('Tags:', tags);
    
    // Check if parameter exists
    const hasCategory = paramMap.has('category');
    console.log('Has category:', hasCategory);
    
    // Get all parameter keys
    const keys = paramMap.keys;
    console.log('All keys:', keys);
    
    // Iterate over all parameters
    keys.forEach(key => {
      console.log(`${key}: ${paramMap.get(key)}`);
    });
  }
}
```

### Type-Safe ParamMap Helper

```typescript
// route-param.helper.ts
import { ParamMap } from '@angular/router';

export class RouteParamHelper {
  static getString(paramMap: ParamMap, key: string, defaultValue = ''): string {
    return paramMap.get(key) || defaultValue;
  }

  static getNumber(paramMap: ParamMap, key: string, defaultValue = 0): number {
    const value = paramMap.get(key);
    return value ? parseInt(value, 10) : defaultValue;
  }

  static getBoolean(paramMap: ParamMap, key: string, defaultValue = false): boolean {
    const value = paramMap.get(key);
    return value ? value === 'true' : defaultValue;
  }

  static getArray(paramMap: ParamMap, key: string): string[] {
    return paramMap.getAll(key);
  }

  static getDate(paramMap: ParamMap, key: string): Date | null {
    const value = paramMap.get(key);
    return value ? new Date(value) : null;
  }
}

// Usage
@Component({})
export class ExampleComponent implements OnInit {
  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    const paramMap = this.route.snapshot.paramMap;
    
    const id = RouteParamHelper.getNumber(paramMap, 'id');
    const active = RouteParamHelper.getBoolean(paramMap, 'active');
    const tags = RouteParamHelper.getArray(paramMap, 'tag');
    const date = RouteParamHelper.getDate(paramMap, 'date');
  }
}
```

## Matrix URL Notation

### Using Matrix Parameters

```typescript
// matrix-params.component.ts
import { Component } from '@angular/core';
import { Router } from '@angular/router';
import { RouterLink } from '@angular/router';

// Matrix notation: /products;color=red;size=large/123
// vs Query params: /products/123?color=red&size=large

@Component({
  selector: 'app-matrix-params',
  standalone: true,
  imports: [RouterLink],
  template: `
    <div>
      <!-- Matrix parameters with RouterLink -->
      <a [routerLink]="['/products', {color: 'red', size: 'large'}, 123]">
        Red Large Product
      </a>
      
      <!-- Programmatic -->
      <button (click)="navigateWithMatrix()">Navigate</button>
    </div>
  `
})
export class MatrixParamsComponent {
  constructor(private router: Router) {}

  navigateWithMatrix(): void {
    // Navigate with matrix parameters
    this.router.navigate([
      '/products',
      { color: 'blue', size: 'medium' },
      '456'
    ]);
  }
}

// Reading matrix parameters
@Component({})
export class ProductComponent implements OnInit {
  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    // Read matrix parameters
    const paramMap = this.route.snapshot.paramMap;
    const color = paramMap.get('color');
    const size = paramMap.get('size');
    const id = paramMap.get('id');
    
    console.log('Color:', color);
    console.log('Size:', size);
    console.log('ID:', id);
  }
}
```

## Common Mistakes

### 1. Using Snapshot with Reusable Components

```typescript
// WRONG - Won't update when navigating from /users/1 to /users/2
@Component({})
export class WrongUserComponent implements OnInit {
  constructor(private route: ActivatedRoute) {}
  
  ngOnInit(): void {
    const id = this.route.snapshot.paramMap.get('id');
    this.loadUser(id); // Only loads once!
  }
}

// CORRECT - Reacts to parameter changes
@Component({})
export class CorrectUserComponent implements OnInit {
  constructor(private route: ActivatedRoute) {}
  
  ngOnInit(): void {
    this.route.paramMap.subscribe(params => {
      const id = params.get('id');
      this.loadUser(id); // Loads on every parameter change
    });
  }
}
```

### 2. Forgetting to Unsubscribe

```typescript
// WRONG - Memory leak
@Component({})
export class WrongComponent implements OnInit {
  ngOnInit(): void {
    this.route.queryParams.subscribe(params => {
      // Handle params
    });
  }
}

// CORRECT - Proper cleanup
@Component({})
export class CorrectComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  
  ngOnInit(): void {
    this.route.queryParams
      .pipe(takeUntil(this.destroy$))
      .subscribe(params => {
        // Handle params
      });
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### 3. Not Handling Null Parameters

```typescript
// WRONG - Potential null errors
@Component({})
export class WrongComponent {
  ngOnInit(): void {
    const id = this.route.snapshot.paramMap.get('id');
    const numericId = parseInt(id, 10); // Error if id is null!
  }
}

// CORRECT - Null handling
@Component({})
export class CorrectComponent {
  ngOnInit(): void {
    const id = this.route.snapshot.paramMap.get('id');
    if (id) {
      const numericId = parseInt(id, 10);
      this.loadData(numericId);
    } else {
      this.handleMissingId();
    }
  }
}
```

### 4. Mixing Query Params Handling Modes

```typescript
// WRONG - Inconsistent behavior
@Component({})
export class WrongComponent {
  addFilter(): void {
    this.router.navigate([], {
      queryParams: { filter: 'new' }
      // Replaces all params
    });
  }
  
  addSort(): void {
    this.router.navigate([], {
      queryParams: { sort: 'date' },
      queryParamsHandling: 'merge'
      // Merges with existing
    });
  }
}

// CORRECT - Consistent approach
@Component({})
export class CorrectComponent {
  updateQueryParams(params: any): void {
    this.router.navigate([], {
      queryParams: params,
      queryParamsHandling: 'merge' // Always merge
    });
  }
}
```

## Best Practices

### 1. Create Route Parameter Services

```typescript
// route-params.service.ts
@Injectable({ providedIn: 'root' })
export class RouteParamsService {
  constructor(private route: ActivatedRoute) {}

  getRequiredParam(key: string): string {
    const value = this.route.snapshot.paramMap.get(key);
    if (!value) {
      throw new Error(`Required parameter '${key}' not found`);
    }
    return value;
  }

  getOptionalParam(key: string, defaultValue = ''): string {
    return this.route.snapshot.paramMap.get(key) || defaultValue;
  }

  getNumericParam(key: string, defaultValue = 0): number {
    const value = this.route.snapshot.paramMap.get(key);
    return value ? parseInt(value, 10) : defaultValue;
  }

  watchParam(key: string): Observable<string | null> {
    return this.route.paramMap.pipe(
      map(params => params.get(key))
    );
  }
}
```

### 2. Use Signals with Route Parameters

```typescript
// modern-route-params.component.ts
import { Component, effect } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { toSignal } from '@angular/core/rxjs-interop';
import { map } from 'rxjs/operators';

@Component({
  selector: 'app-modern-route-params',
  standalone: true,
  template: `<div>User {{ userId() }}</div>`
})
export class ModernRouteParamsComponent {
  userId = toSignal(
    this.route.paramMap.pipe(map(params => params.get('id'))),
    { initialValue: null }
  );

  constructor(private route: ActivatedRoute) {
    effect(() => {
      const id = this.userId();
      if (id) {
        this.loadUser(id);
      }
    });
  }

  private loadUser(id: string): void {
    console.log('Loading user:', id);
  }
}
```

### 3. Validate Parameters

```typescript
// validated-route.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';

@Component({
  selector: 'app-validated-route',
  standalone: true,
  template: `<div>Validated Route</div>`
})
export class ValidatedRouteComponent implements OnInit {
  constructor(
    private route: ActivatedRoute,
    private router: Router
  ) {}

  ngOnInit(): void {
    const id = this.route.snapshot.paramMap.get('id');
    
    if (!this.isValidId(id)) {
      this.router.navigate(['/404']);
      return;
    }

    this.loadData(id!);
  }

  private isValidId(id: string | null): boolean {
    if (!id) return false;
    const numericId = parseInt(id, 10);
    return !isNaN(numericId) && numericId > 0;
  }

  private loadData(id: string): void {
    // Load data
  }
}
```

### 4. Centralize Query Parameter Logic

```typescript
// search-params.service.ts
@Injectable({ providedIn: 'root' })
export class SearchParamsService {
  private defaultParams = {
    query: '',
    page: 1,
    limit: 20,
    sort: 'relevance'
  };

  constructor(
    private router: Router,
    private route: ActivatedRoute
  ) {}

  getSearchParams(): typeof this.defaultParams {
    const params = this.route.snapshot.queryParams;
    return {
      query: params['q'] || this.defaultParams.query,
      page: parseInt(params['page'] || this.defaultParams.page, 10),
      limit: parseInt(params['limit'] || this.defaultParams.limit, 10),
      sort: params['sort'] || this.defaultParams.sort
    };
  }

  updateSearchParams(params: Partial<typeof this.defaultParams>): void {
    this.router.navigate([], {
      relativeTo: this.route,
      queryParams: params,
      queryParamsHandling: 'merge'
    });
  }

  resetSearchParams(): void {
    this.router.navigate([], {
      relativeTo: this.route,
      queryParams: this.defaultParams
    });
  }
}
```

## Interview Questions

### Q1: What's the difference between route parameters and query parameters?

**Answer:** Route parameters are part of the route path (`/users/:id`) and are typically required for the route to function. Query parameters are optional additions to the URL (`?search=term&page=1`) used for filtering, sorting, or optional data.

### Q2: When should you use snapshot vs observable for route parameters?

**Answer:** Use snapshot when the component is destroyed and recreated on navigation (different route configuration). Use observable when the same component instance is reused for different parameter values (same route, different params).

### Q3: How do you access parent route parameters in a child component?

**Answer:** Access through `this.route.parent?.snapshot.paramMap` for snapshot or `this.route.parent?.paramMap` for observable access.

### Q4: What are matrix parameters and when would you use them?

**Answer:** Matrix parameters use semicolon notation (`/products;color=red;size=large/123`) and are scoped to a specific URL segment. Use them when parameters are specific to one route segment rather than the entire URL.

### Q5: How do you handle multiple values for the same query parameter?

**Answer:** Use `getAll()` method: `paramMap.getAll('tag')` returns an array of all values for the 'tag' parameter.

### Q6: What's the purpose of queryParamsHandling?

**Answer:** It controls how new query parameters interact with existing ones: 'merge' combines them, 'preserve' keeps existing ones, and default ('') replaces all.

### Q7: How do you pass complex data through routes?

**Answer:** Use navigation state: `router.navigate([path], { state: { data } })`. Access via `getCurrentNavigation()?.extras.state`.

### Q8: How do you prevent memory leaks with route parameter subscriptions?

**Answer:** Use takeUntil pattern with a destroy$ Subject, or use toSignal() for signal-based reactive programming.

## Key Takeaways

1. **Route parameters** are part of the path, **query parameters** are optional additions
2. **Snapshot** for one-time reads, **observable** for reactive updates
3. **ParamMap** provides type-safe parameter access with helper methods
4. **Always handle null** parameter values defensively
5. **Unsubscribe** from parameter observables to prevent memory leaks
6. **Matrix parameters** scope to URL segments, query params to entire URL
7. **queryParamsHandling** controls parameter merging behavior
8. **Navigation state** passes data without exposing in URL
9. **toSignal()** modernizes parameter handling with signals
10. **Validate parameters** before using them to prevent errors

## Resources

- [Angular Router Documentation](https://angular.io/guide/router)
- [ActivatedRoute API](https://angular.io/api/router/ActivatedRoute)
- [ParamMap API](https://angular.io/api/router/ParamMap)
- [Router Navigate Options](https://angular.io/api/router/NavigationExtras)
- [Route Parameters Guide](https://angular.io/guide/router#route-parameters)
- [Query Parameters Guide](https://angular.io/guide/router#query-parameters)
- [RxJS takeUntil Pattern](https://rxjs.dev/api/operators/takeUntil)
