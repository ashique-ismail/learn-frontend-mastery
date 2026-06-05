# Route Resolvers in Angular

## The Idea

**In plain English:** A route resolver is a helper that goes and fetches the data a page needs before that page even opens, so when you arrive the information is already there waiting for you. Think of it like a waiter who takes your order and brings your food to the table before you even sit down.

**Real-world analogy:** Imagine checking into a hotel. Before you walk into your room, the concierge prepares it: the bed is made, the towels are ready, and your welcome snack is on the table. You never see an empty, unprepared room.

- The concierge preparing the room = the resolver fetching data from the server
- The prepared room = the resolved data available when the component loads
- You walking into the ready room = the Angular component rendering with all data already present

---

## Table of Contents

- [Introduction](#introduction)
- [Basic Resolvers](#basic-resolvers)
- [ResolveFn Type](#resolvefn-type)
- [Async Data Resolution](#async-data-resolution)
- [Error Handling](#error-handling)
- [Resolve Guards](#resolve-guards)
- [Pre-fetching Strategies](#pre-fetching-strategies)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Route resolvers pre-fetch data before a route is activated, ensuring all required data is available before the component renders. This prevents displaying empty or loading states and provides a smoother user experience.

## Basic Resolvers

### Functional Resolver (Modern Approach)

```typescript
// product.resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { ProductService } from './product.service';
import { Product } from './product.model';

export const productResolver: ResolveFn<Product> = (route, state) => {
  const productService = inject(ProductService);
  const productId = route.paramMap.get('id');
  
  if (!productId) {
    throw new Error('Product ID is required');
  }

  return productService.getProduct(productId);
};

// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: 'products/:id',
    component: ProductDetailComponent,
    resolve: {
      product: productResolver
    }
  }
];

// product-detail.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { Product } from './product.model';

@Component({
  selector: 'app-product-detail',
  standalone: true,
  template: `
    <div *ngIf="product">
      <h2>{{ product.name }}</h2>
      <p>{{ product.description }}</p>
      <p>Price: {{ product.price | currency }}</p>
    </div>
  `
})
export class ProductDetailComponent implements OnInit {
  product!: Product;

  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    // Data is already resolved
    this.product = this.route.snapshot.data['product'];
  }
}
```

### Multiple Resolvers

```typescript
// user-detail.resolver.ts
export const userResolver: ResolveFn<User> = (route, state) => {
  const userService = inject(UserService);
  const userId = route.paramMap.get('id')!;
  return userService.getUser(userId);
};

// user-posts.resolver.ts
export const userPostsResolver: ResolveFn<Post[]> = (route, state) => {
  const postService = inject(PostService);
  const userId = route.paramMap.get('id')!;
  return postService.getUserPosts(userId);
};

// user-comments.resolver.ts
export const userCommentsResolver: ResolveFn<Comment[]> = (route, state) => {
  const commentService = inject(CommentService);
  const userId = route.paramMap.get('id')!;
  return commentService.getUserComments(userId);
};

// app.routes.ts
export const routes: Routes = [
  {
    path: 'users/:id',
    component: UserProfileComponent,
    resolve: {
      user: userResolver,
      posts: userPostsResolver,
      comments: userCommentsResolver
    }
  }
];

// user-profile.component.ts
@Component({
  selector: 'app-user-profile',
  standalone: true,
  template: `
    <div class="profile">
      <h2>{{ user.name }}</h2>
      <div class="posts">
        <h3>Posts ({{ posts.length }})</h3>
      </div>
      <div class="comments">
        <h3>Comments ({{ comments.length }})</h3>
      </div>
    </div>
  `
})
export class UserProfileComponent implements OnInit {
  user!: User;
  posts!: Post[];
  comments!: Comment[];

  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    const data = this.route.snapshot.data;
    this.user = data['user'];
    this.posts = data['posts'];
    this.comments = data['comments'];
  }
}
```

### Observable-Based Resolver Access

```typescript
// reactive-component.component.ts
import { Component } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { map, Observable } from 'rxjs';

@Component({
  selector: 'app-reactive-component',
  standalone: true,
  template: `
    <div *ngIf="product$ | async as product">
      <h2>{{ product.name }}</h2>
      <p>{{ product.description }}</p>
    </div>
  `
})
export class ReactiveComponent {
  product$: Observable<Product>;

  constructor(private route: ActivatedRoute) {
    // React to data changes (useful for route reuse)
    this.product$ = this.route.data.pipe(
      map(data => data['product'])
    );
  }
}
```

## ResolveFn Type

### Type-Safe Resolvers

```typescript
// typed-resolvers.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';

// Simple type
export const stringResolver: ResolveFn<string> = (route, state) => {
  return 'Hello World';
};

// Complex type
export interface ProductDetails {
  product: Product;
  reviews: Review[];
  relatedProducts: Product[];
}

export const productDetailsResolver: ResolveFn<ProductDetails> = (route, state) => {
  const productService = inject(ProductService);
  const productId = route.paramMap.get('id')!;

  return productService.getProductDetails(productId);
};

// Union types
export const dataResolver: ResolveFn<Product | null> = (route, state) => {
  const productService = inject(ProductService);
  const productId = route.paramMap.get('id');

  if (!productId) {
    return null;
  }

  return productService.getProduct(productId);
};

// Array types
export const categoriesResolver: ResolveFn<Category[]> = (route, state) => {
  const categoryService = inject(CategoryService);
  return categoryService.getAllCategories();
};
```

### Generic Resolver Factory

```typescript
// resolver-factory.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { Observable } from 'rxjs';

export interface EntityService<T> {
  getById(id: string): Observable<T>;
  getAll(): Observable<T[]>;
}

export function createEntityResolver<T>(
  serviceToken: any,
  paramName = 'id'
): ResolveFn<T> {
  return (route, state) => {
    const service: EntityService<T> = inject(serviceToken);
    const id = route.paramMap.get(paramName);

    if (!id) {
      throw new Error(`${paramName} parameter is required`);
    }

    return service.getById(id);
  };
}

export function createListResolver<T>(
  serviceToken: any
): ResolveFn<T[]> {
  return (route, state) => {
    const service: EntityService<T> = inject(serviceToken);
    return service.getAll();
  };
}

// Usage
export const productResolver = createEntityResolver<Product>(ProductService);
export const userResolver = createEntityResolver<User>(UserService);
export const productsResolver = createListResolver<Product>(ProductService);
```

## Async Data Resolution

### Observable Resolvers

```typescript
// async-resolvers.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { Observable } from 'rxjs';
import { map, catchError } from 'rxjs/operators';

export const asyncProductResolver: ResolveFn<Product> = (route, state) => {
  const productService = inject(ProductService);
  const productId = route.paramMap.get('id')!;

  return productService.getProduct(productId);
};

// product.service.ts
@Injectable({ providedIn: 'root' })
export class ProductService {
  constructor(private http: HttpClient) {}

  getProduct(id: string): Observable<Product> {
    return this.http.get<Product>(`/api/products/${id}`);
  }
}
```

### Promise Resolvers

```typescript
// promise-resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';

export const promiseResolver: ResolveFn<Product> = async (route, state) => {
  const productService = inject(ProductService);
  const productId = route.paramMap.get('id')!;

  try {
    const product = await productService.getProductAsync(productId);
    return product;
  } catch (error) {
    console.error('Failed to load product:', error);
    throw error;
  }
};

// product.service.ts
@Injectable({ providedIn: 'root' })
export class ProductService {
  constructor(private http: HttpClient) {}

  async getProductAsync(id: string): Promise<Product> {
    return firstValueFrom(
      this.http.get<Product>(`/api/products/${id}`)
    );
  }
}
```

### Combining Multiple Async Operations

```typescript
// combined-resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { forkJoin, map } from 'rxjs';

export interface DashboardData {
  user: User;
  stats: Statistics;
  notifications: Notification[];
  recentActivity: Activity[];
}

export const dashboardResolver: ResolveFn<DashboardData> = (route, state) => {
  const userService = inject(UserService);
  const statsService = inject(StatsService);
  const notificationService = inject(NotificationService);
  const activityService = inject(ActivityService);

  return forkJoin({
    user: userService.getCurrentUser(),
    stats: statsService.getUserStats(),
    notifications: notificationService.getUnread(),
    recentActivity: activityService.getRecent(10)
  });
};
```

### Sequential Data Loading

```typescript
// sequential-resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { switchMap, map } from 'rxjs';

export interface OrderWithDetails {
  order: Order;
  customer: Customer;
  items: OrderItem[];
}

export const orderDetailsResolver: ResolveFn<OrderWithDetails> = (route, state) => {
  const orderService = inject(OrderService);
  const customerService = inject(CustomerService);
  const orderId = route.paramMap.get('id')!;

  return orderService.getOrder(orderId).pipe(
    switchMap(order =>
      forkJoin({
        order: of(order),
        customer: customerService.getCustomer(order.customerId),
        items: orderService.getOrderItems(orderId)
      })
    )
  );
};
```

## Error Handling

### Basic Error Handling

```typescript
// error-handling.resolver.ts
import { inject } from '@angular/core';
import { ResolveFn, Router } from '@angular/router';
import { catchError, of } from 'rxjs';

export const safeProductResolver: ResolveFn<Product | null> = (route, state) => {
  const productService = inject(ProductService);
  const router = inject(Router);
  const productId = route.paramMap.get('id')!;

  return productService.getProduct(productId).pipe(
    catchError(error => {
      console.error('Failed to load product:', error);
      router.navigate(['/products']);
      return of(null);
    })
  );
};
```

### Redirect on Error

```typescript
// redirect-on-error.resolver.ts
import { inject } from '@angular/core';
import { ResolveFn, Router } from '@angular/router';
import { catchError, EMPTY } from 'rxjs';

export const redirectOnErrorResolver: ResolveFn<Product> = (route, state) => {
  const productService = inject(ProductService);
  const router = inject(Router);
  const productId = route.paramMap.get('id')!;

  return productService.getProduct(productId).pipe(
    catchError(error => {
      if (error.status === 404) {
        router.navigate(['/404']);
      } else if (error.status === 403) {
        router.navigate(['/unauthorized']);
      } else {
        router.navigate(['/error'], {
          queryParams: { message: error.message }
        });
      }
      return EMPTY; // Cancel navigation
    })
  );
};
```

### Retry Logic in Resolvers

```typescript
// retry-resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { retry, catchError, throwError } from 'rxjs';

export const retryResolver: ResolveFn<Product> = (route, state) => {
  const productService = inject(ProductService);
  const productId = route.paramMap.get('id')!;

  return productService.getProduct(productId).pipe(
    retry({
      count: 3,
      delay: 1000
    }),
    catchError(error => {
      console.error('Failed after 3 retries:', error);
      return throwError(() => error);
    })
  );
};
```

### Custom Error Response

```typescript
// custom-error-resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { catchError, of } from 'rxjs';

export interface ResolverResult<T> {
  data?: T;
  error?: {
    message: string;
    code: string;
  };
}

export const customErrorResolver: ResolveFn<ResolverResult<Product>> = (route, state) => {
  const productService = inject(ProductService);
  const productId = route.paramMap.get('id')!;

  return productService.getProduct(productId).pipe(
    map(product => ({ data: product })),
    catchError(error => of({
      error: {
        message: error.message || 'Failed to load product',
        code: error.status?.toString() || 'UNKNOWN'
      }
    }))
  );
};

// Component usage
@Component({})
export class ProductComponent implements OnInit {
  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    const result: ResolverResult<Product> = this.route.snapshot.data['product'];
    
    if (result.error) {
      console.error('Error:', result.error.message);
      // Handle error
    } else if (result.data) {
      // Use data
      console.log('Product:', result.data);
    }
  }
}
```

## Resolve Guards

### Using Resolvers as Guards

```typescript
// auth-resolve.guard.ts
import { inject } from '@angular/core';
import { ResolveFn, Router } from '@angular/router';
import { catchError, map } from 'rxjs';

export const authResolveGuard: ResolveFn<User | null> = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  return authService.getCurrentUser().pipe(
    map(user => {
      if (!user) {
        router.navigate(['/login']);
        return null;
      }
      return user;
    }),
    catchError(() => {
      router.navigate(['/login']);
      return of(null);
    })
  );
};

// app.routes.ts
export const routes: Routes = [
  {
    path: 'profile',
    component: ProfileComponent,
    resolve: {
      user: authResolveGuard
    }
  }
];
```

### Conditional Resolution

```typescript
// conditional-resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { of } from 'rxjs';

export const conditionalResolver: ResolveFn<Product | null> = (route, state) => {
  const productService = inject(ProductService);
  const cacheService = inject(CacheService);
  const productId = route.paramMap.get('id')!;

  // Check cache first
  const cached = cacheService.get<Product>(`product-${productId}`);
  if (cached) {
    console.log('Using cached product');
    return of(cached);
  }

  // Fetch from API
  return productService.getProduct(productId).pipe(
    tap(product => cacheService.set(`product-${productId}`, product, 300000))
  );
};
```

## Pre-fetching Strategies

### Prefetch Related Data

```typescript
// prefetch-resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { forkJoin, map } from 'rxjs';

export interface ArticlePageData {
  article: Article;
  author: Author;
  relatedArticles: Article[];
  comments: Comment[];
  categories: Category[];
}

export const articlePageResolver: ResolveFn<ArticlePageData> = (route, state) => {
  const articleService = inject(ArticleService);
  const authorService = inject(AuthorService);
  const commentService = inject(CommentService);
  const categoryService = inject(CategoryService);
  const articleId = route.paramMap.get('id')!;

  return articleService.getArticle(articleId).pipe(
    switchMap(article =>
      forkJoin({
        article: of(article),
        author: authorService.getAuthor(article.authorId),
        relatedArticles: articleService.getRelated(article.id, 5),
        comments: commentService.getArticleComments(article.id),
        categories: categoryService.getArticleCategories(article.id)
      })
    )
  );
};
```

### Lazy Data Loading Strategy

```typescript
// lazy-resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { of } from 'rxjs';

export interface MinimalData {
  essential: EssentialData;
  lazyLoader: () => Observable<AdditionalData>;
}

export const lazyDataResolver: ResolveFn<MinimalData> = (route, state) => {
  const dataService = inject(DataService);
  const id = route.paramMap.get('id')!;

  // Load only essential data immediately
  return dataService.getEssential(id).pipe(
    map(essential => ({
      essential,
      // Provide function to load additional data later
      lazyLoader: () => dataService.getAdditional(id)
    }))
  );
};

// Component usage
@Component({})
export class LazyLoadComponent implements OnInit {
  minimalData!: MinimalData;
  additionalData?: AdditionalData;

  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    this.minimalData = this.route.snapshot.data['data'];
  }

  loadMore(): void {
    this.minimalData.lazyLoader().subscribe(data => {
      this.additionalData = data;
    });
  }
}
```

### Pagination Resolver

```typescript
// pagination-resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';

export interface PaginatedData<T> {
  items: T[];
  total: number;
  page: number;
  pageSize: number;
  hasNext: boolean;
  hasPrevious: boolean;
}

export const paginatedProductsResolver: ResolveFn<PaginatedData<Product>> = (route, state) => {
  const productService = inject(ProductService);
  
  const page = parseInt(route.queryParamMap.get('page') || '1', 10);
  const pageSize = parseInt(route.queryParamMap.get('pageSize') || '20', 10);
  const category = route.queryParamMap.get('category');

  return productService.getProducts({
    page,
    pageSize,
    category: category || undefined
  });
};
```

### Background Data Refresh

```typescript
// refresh-resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { shareReplay, tap, timer, switchMap, startWith } from 'rxjs';

export const autoRefreshResolver: ResolveFn<DashboardStats> = (route, state) => {
  const statsService = inject(StatsService);

  // Load immediately, then refresh every 30 seconds
  return timer(0, 30000).pipe(
    switchMap(() => statsService.getStats()),
    tap(stats => console.log('Stats refreshed:', stats)),
    shareReplay(1)
  );
};
```

## Common Mistakes

### 1. Not Handling Errors

```typescript
// WRONG - No error handling
export const wrongResolver: ResolveFn<Product> = (route, state) => {
  const service = inject(ProductService);
  return service.getProduct(route.paramMap.get('id')!);
  // If this fails, navigation hangs!
};

// CORRECT - Proper error handling
export const correctResolver: ResolveFn<Product | null> = (route, state) => {
  const service = inject(ProductService);
  const router = inject(Router);
  
  return service.getProduct(route.paramMap.get('id')!).pipe(
    catchError(error => {
      console.error('Resolver error:', error);
      router.navigate(['/error']);
      return of(null);
    })
  );
};
```

### 2. Over-fetching Data

```typescript
// WRONG - Loading too much data upfront
export const wrongResolver: ResolveFn<HugeDataset> = (route, state) => {
  const service = inject(DataService);
  return service.loadEverything(); // Slow!
};

// CORRECT - Load only what's needed
export const correctResolver: ResolveFn<EssentialData> = (route, state) => {
  const service = inject(DataService);
  return service.loadEssentials();
};
```

### 3. Not Using Type Safety

```typescript
// WRONG - No type safety
export const wrongResolver: ResolveFn<any> = (route, state) => {
  return inject(ProductService).getProduct('123');
};

// CORRECT - Type-safe
export const correctResolver: ResolveFn<Product> = (route, state) => {
  return inject(ProductService).getProduct('123');
};
```

### 4. Blocking Navigation Too Long

```typescript
// WRONG - Multiple sequential requests block navigation
export const wrongResolver: ResolveFn<ComplexData> = (route, state) => {
  const service = inject(DataService);
  
  return service.getFirst().pipe(
    switchMap(first => service.getSecond(first.id)),
    switchMap(second => service.getThird(second.id))
  );
  // Navigation blocked until all complete!
};

// CORRECT - Load minimal data, lazy-load rest
export const correctResolver: ResolveFn<MinimalData> = (route, state) => {
  const service = inject(DataService);
  return service.getFirst(); // Only critical data
};
```

## Best Practices

### 1. Keep Resolvers Focused

```typescript
// Good: Single responsibility
export const productResolver: ResolveFn<Product> = (route, state) => {
  const service = inject(ProductService);
  return service.getProduct(route.paramMap.get('id')!);
};

export const reviewsResolver: ResolveFn<Review[]> = (route, state) => {
  const service = inject(ReviewService);
  return service.getProductReviews(route.paramMap.get('id')!);
};

// Use multiple resolvers instead of one doing everything
export const routes: Routes = [
  {
    path: 'products/:id',
    component: ProductComponent,
    resolve: {
      product: productResolver,
      reviews: reviewsResolver
    }
  }
];
```

### 2. Use Caching

```typescript
// cached-resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { shareReplay } from 'rxjs';

const cache = new Map<string, Observable<any>>();

export function createCachedResolver<T>(
  fetchFn: (id: string) => Observable<T>,
  cacheKeyPrefix: string
): ResolveFn<T> {
  return (route, state) => {
    const id = route.paramMap.get('id')!;
    const cacheKey = `${cacheKeyPrefix}-${id}`;

    if (!cache.has(cacheKey)) {
      cache.set(cacheKey, fetchFn(id).pipe(shareReplay(1)));
    }

    return cache.get(cacheKey)!;
  };
}
```

### 3. Provide Loading Feedback

```typescript
// loading-resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { tap, finalize } from 'rxjs';

export const loadingResolver: ResolveFn<Product> = (route, state) => {
  const productService = inject(ProductService);
  const loadingService = inject(LoadingService);
  const productId = route.paramMap.get('id')!;

  loadingService.show('Loading product...');

  return productService.getProduct(productId).pipe(
    finalize(() => loadingService.hide())
  );
};
```

### 4. Document Resolver Dependencies

```typescript
/**
 * Resolves product data before activating the product detail route.
 * 
 * @requires ProductService - For fetching product data
 * @param id - Product ID from route parameter
 * @returns Observable<Product> - Resolved product data
 * 
 * @throws Will redirect to /404 if product not found
 * @throws Will redirect to /error on server error
 */
export const documentedProductResolver: ResolveFn<Product> = (route, state) => {
  const productService = inject(ProductService);
  const router = inject(Router);
  const productId = route.paramMap.get('id')!;

  return productService.getProduct(productId).pipe(
    catchError(error => {
      if (error.status === 404) {
        router.navigate(['/404']);
      } else {
        router.navigate(['/error']);
      }
      return EMPTY;
    })
  );
};
```

## Interview Questions

### Q1: What is the purpose of route resolvers?

**Answer:** Resolvers pre-fetch data before a route activates, ensuring all required data is available before the component renders. This prevents showing loading states or empty components and provides a smoother UX.

### Q2: What's the difference between resolvers and loading data in ngOnInit?

**Answer:** Resolvers fetch data before route activation and navigation completes. ngOnInit fetches data after the component is created, requiring loading states. Resolvers guarantee data availability when the component initializes.

### Q3: How do you handle errors in resolvers?

**Answer:** Use RxJS catchError operator to handle errors. Either return a default value, redirect to an error page, or return EMPTY to cancel navigation. Always handle errors to prevent hanging navigation.

### Q4: Can resolvers return synchronous data?

**Answer:** Yes, resolvers can return synchronous values, Promises, or Observables. Angular handles all types automatically.

### Q5: How do you access resolved data in a component?

**Answer:** Access via `ActivatedRoute.snapshot.data['keyName']` for one-time access, or subscribe to `ActivatedRoute.data` observable for reactive updates.

### Q6: Should you always use resolvers for data fetching?

**Answer:** No. Use resolvers for critical data needed before rendering. For secondary data, progressive loading, or when you want to show loading states, fetch in the component.

### Q7: How do multiple resolvers execute?

**Answer:** Multiple resolvers execute in parallel using forkJoin internally. Navigation completes only after all resolvers finish or one fails.

### Q8: Can you redirect in a resolver?

**Answer:** Yes, inject Router and use createUrlTree() or navigate(). Return EMPTY to cancel the original navigation after redirecting.

## Key Takeaways

1. **Resolvers pre-fetch data** before route activation
2. **Use ResolveFn** for functional resolvers (Angular 14+)
3. **Always handle errors** to prevent hanging navigation
4. **Multiple resolvers** run in parallel automatically
5. **Access data** via ActivatedRoute.snapshot.data or ActivatedRoute.data observable
6. **Type-safe resolvers** prevent runtime errors
7. **Keep resolvers focused** on single responsibility
8. **Consider caching** for frequently accessed data
9. **Don't over-fetch** - load only essential data
10. **Provide feedback** during long-running resolutions

## Resources

- [Angular Resolvers Guide](https://angular.io/guide/router#resolve-pre-fetching-component-data)
- [ResolveFn API](https://angular.io/api/router/ResolveFn)
- [ActivatedRoute API](https://angular.io/api/router/ActivatedRoute)
- [Router Resolve Documentation](https://angular.io/api/router/Resolve)
- [RxJS Error Handling](https://rxjs.dev/api/operators/catchError)
- [Route Resolver Tutorial](https://angular.io/guide/router-tutorial-toh#milestone-4-crisis-center-feature)
