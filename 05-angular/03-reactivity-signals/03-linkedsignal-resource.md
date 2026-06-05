# linkedSignal and resource in Angular

## The Idea

**In plain English:** `linkedSignal` and `resource` are tools in Angular that help your app automatically update its data in smart ways — `linkedSignal` lets a value start by copying from another value but still be changed on its own, while `resource` handles fetching data from the internet and keeps track of whether that data is still loading, has arrived, or hit an error.

**Real-world analogy:** Imagine a whiteboard in a classroom. The teacher writes the day's agenda on the board (the source), and a student copies it into their notebook (the linked copy). The student can cross things out or add personal notes independently, but if the teacher erases the board and writes a brand new agenda, the student starts fresh with a clean copy again. Meanwhile, a school librarian is placing a book order online — everyone can see a sign saying "Order Pending", then "Books Arrived", or "Order Failed" depending on what happens.

- The teacher's agenda on the whiteboard = the source signal
- The student's notebook copy = the `linkedSignal` (starts as a copy, can be edited independently, resets when the source is fully replaced)
- The librarian's book order process = the `resource` (fetches data asynchronously with visible loading, success, and error states)

---

## Table of Contents

- [Introduction](#introduction)
- [linkedSignal() Function](#linkedsignal-function)
- [resource() Function](#resource-function)
- [Async Data Management](#async-data-management)
- [Error Handling](#error-handling)
- [Loading States](#loading-states)
- [Advanced Patterns](#advanced-patterns)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Angular 19 introduces two powerful primitives for managing reactive state: `linkedSignal()` and `resource()`. These functions build on the foundation of signals to provide more sophisticated state management patterns.

- **linkedSignal()**: Creates a writable signal that derives its initial value from another signal but can be independently updated
- **resource()**: Manages asynchronous data loading with built-in loading, error, and success states

Both primitives address common patterns in Angular applications, making complex state management more declarative and easier to reason about.

## linkedSignal() Function

### Basic Usage

`linkedSignal()` creates a signal that starts with a computed value but can be modified independently. It automatically recomputes when its source changes, unless manually overridden.

```typescript
import { Component, signal, linkedSignal } from '@angular/core';

@Component({
  selector: 'app-linked-basic',
  template: `
    <div>
      <p>Source: {{ source() }}</p>
      <p>Linked: {{ linked() }}</p>
      <button (click)="updateSource()">Update Source</button>
      <button (click)="updateLinked()">Update Linked</button>
      <button (click)="reset()">Reset</button>
    </div>
  `
})
export class LinkedBasicComponent {
  source = signal(10);
  
  // Linked signal derives initial value from source
  linked = linkedSignal(() => this.source() * 2);
  
  updateSource(): void {
    this.source.update(n => n + 1);
    // linked automatically updates to source * 2
  }
  
  updateLinked(): void {
    this.linked.set(100);
    // linked is now independent from source
    // Further source changes don't affect linked until reset
  }
  
  reset(): void {
    // Manually reset to follow source again
    this.linked.set(this.source() * 2);
  }
}
```

### Form Input Pattern

```typescript
import { Component, signal, linkedSignal } from '@angular/core';

interface User {
  id: number;
  name: string;
  email: string;
}

@Component({
  selector: 'app-user-form',
  template: `
    <div>
      <h3>Current User</h3>
      <p>{{ currentUser().name }} ({{ currentUser().email }})</p>
      
      <h3>Edit Form</h3>
      <input [(ngModel)]="formName" (ngModelChange)="editedName.set(formName)">
      <input [(ngModel)]="formEmail" (ngModelChange)="editedEmail.set(formEmail)">
      
      <button (click)="save()">Save</button>
      <button (click)="cancel()">Cancel</button>
      <button (click)="loadNextUser()">Load Next User</button>
      
      <p *ngIf="hasChanges()">Unsaved changes!</p>
    </div>
  `
})
export class UserFormComponent {
  currentUser = signal<User>({
    id: 1,
    name: 'John Doe',
    email: 'john@example.com'
  });
  
  formName = 'John Doe';
  formEmail = 'john@example.com';
  
  // Linked signals automatically reset when currentUser changes
  editedName = linkedSignal(() => this.currentUser().name);
  editedEmail = linkedSignal(() => this.currentUser().email);
  
  hasChanges = computed(() => {
    const user = this.currentUser();
    return this.editedName() !== user.name || 
           this.editedEmail() !== user.email;
  });
  
  save(): void {
    this.currentUser.update(user => ({
      ...user,
      name: this.editedName(),
      email: this.editedEmail()
    }));
  }
  
  cancel(): void {
    // Reset to current user values
    const user = this.currentUser();
    this.editedName.set(user.name);
    this.editedEmail.set(user.email);
    this.formName = user.name;
    this.formEmail = user.email;
  }
  
  loadNextUser(): void {
    // Simulate loading new user
    this.currentUser.set({
      id: 2,
      name: 'Jane Smith',
      email: 'jane@example.com'
    });
    // editedName and editedEmail automatically reset!
    this.formName = this.editedName();
    this.formEmail = this.editedEmail();
  }
}
```

### Default Value Pattern

```typescript
import { Component, signal, linkedSignal, computed } from '@angular/core';

@Component({
  selector: 'app-default-value',
  template: `
    <div>
      <label>
        <input type="checkbox" [(ngModel)]="useDefaultValue" (change)="useDefault.set(useDefaultValue)">
        Use Default
      </label>
      
      <input 
        [disabled]="useDefault()"
        [(ngModel)]="customValue"
        (ngModelChange)="value.set(customValue)">
      
      <p>Current Value: {{ value() }}</p>
      <p>Default Value: {{ defaultValue() }}</p>
      
      <button (click)="updateDefault()">Update Default</button>
    </div>
  `
})
export class DefaultValueComponent {
  useDefaultValue = true;
  customValue = '';
  
  defaultValue = signal('Default Text');
  useDefault = signal(true);
  
  // Value links to default when useDefault is true
  value = linkedSignal(() => 
    this.useDefault() ? this.defaultValue() : ''
  );
  
  updateDefault(): void {
    this.defaultValue.set('New Default Text');
    // If useDefault is true, value automatically updates
  }
}
```

### Cascading Updates Pattern

```typescript
import { Component, signal, linkedSignal } from '@angular/core';

interface Product {
  id: number;
  name: string;
  category: string;
}

interface Category {
  id: string;
  name: string;
  products: Product[];
}

@Component({
  selector: 'app-cascading',
  template: `
    <div>
      <select [(ngModel)]="categoryIdValue" (change)="selectedCategoryId.set(categoryIdValue)">
        <option *ngFor="let cat of categories" [value]="cat.id">
          {{ cat.name }}
        </option>
      </select>
      
      <select [(ngModel)]="productIdValue" (change)="selectedProductId.set(+productIdValue)">
        <option *ngFor="let product of availableProducts()" [value]="product.id">
          {{ product.name }}
        </option>
      </select>
      
      <p>Selected: {{ selectedProduct()?.name }}</p>
    </div>
  `
})
export class CascadingComponent {
  categoryIdValue = '';
  productIdValue = 0;
  
  categories: Category[] = [
    {
      id: 'electronics',
      name: 'Electronics',
      products: [
        { id: 1, name: 'Laptop', category: 'electronics' },
        { id: 2, name: 'Phone', category: 'electronics' }
      ]
    },
    {
      id: 'furniture',
      name: 'Furniture',
      products: [
        { id: 3, name: 'Desk', category: 'furniture' },
        { id: 4, name: 'Chair', category: 'furniture' }
      ]
    }
  ];
  
  selectedCategoryId = signal('electronics');
  
  availableProducts = computed(() => {
    const catId = this.selectedCategoryId();
    return this.categories.find(c => c.id === catId)?.products ?? [];
  });
  
  // Product selection resets when category changes
  selectedProductId = linkedSignal(() => {
    const products = this.availableProducts();
    return products[0]?.id ?? 0;
  });
  
  selectedProduct = computed(() => {
    const products = this.availableProducts();
    return products.find(p => p.id === this.selectedProductId());
  });
  
  constructor() {
    this.categoryIdValue = this.selectedCategoryId();
    this.productIdValue = this.selectedProductId();
  }
}
```

### Synchronized State Pattern

```typescript
import { Component, signal, linkedSignal, effect } from '@angular/core';

@Component({
  selector: 'app-synchronized',
  template: `
    <div>
      <h3>Server Value: {{ serverValue() }}</h3>
      
      <input [(ngModel)]="localValueInput" (ngModelChange)="localValue.set(localValueInput)">
      <p>Local Value: {{ localValue() }}</p>
      
      <button (click)="sync()">Sync</button>
      <button (click)="simulateServerUpdate()">Simulate Server Update</button>
      
      <p *ngIf="isDirty()">Local changes not synced</p>
    </div>
  `
})
export class SynchronizedComponent {
  localValueInput = '';
  
  // Simulated server state
  serverValue = signal('Server Data v1');
  
  // Local state linked to server state
  localValue = linkedSignal(() => this.serverValue());
  
  isDirty = computed(() => 
    this.localValue() !== this.serverValue()
  );
  
  constructor() {
    this.localValueInput = this.localValue();
    
    // Auto-sync when server value changes
    effect(() => {
      const server = this.serverValue();
      if (!this.isDirty()) {
        this.localValue.set(server);
        this.localValueInput = server;
      }
    });
  }
  
  sync(): void {
    // Simulate sending to server
    this.serverValue.set(this.localValue());
  }
  
  simulateServerUpdate(): void {
    this.serverValue.set('Server Data v2');
    // If no local changes, localValue automatically updates
  }
}
```

## resource() Function

### Basic Resource Usage

```typescript
import { Component, signal, resource } from '@angular/core';
import { inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';

interface User {
  id: number;
  name: string;
  email: string;
}

@Component({
  selector: 'app-basic-resource',
  template: `
    <div>
      <button (click)="userId.set(userId() + 1)">Load Next User</button>
      
      @if (userResource.isLoading()) {
        <p>Loading user...</p>
      }
      
      @if (userResource.error()) {
        <p class="error">Error: {{ userResource.error()?.message }}</p>
      }
      
      @if (userResource.value(); as user) {
        <div class="user-card">
          <h3>{{ user.name }}</h3>
          <p>{{ user.email }}</p>
        </div>
      }
      
      <button 
        (click)="userResource.reload()"
        [disabled]="userResource.isLoading()">
        Reload
      </button>
    </div>
  `
})
export class BasicResourceComponent {
  private http = inject(HttpClient);
  
  userId = signal(1);
  
  // Resource automatically loads when userId changes
  userResource = resource({
    request: () => ({ id: this.userId() }),
    loader: ({ request }) => {
      return this.http.get<User>(`/api/users/${request.id}`);
    }
  });
}
```

### Resource with Dependencies

```typescript
import { Component, signal, resource, computed } from '@angular/core';
import { inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';

interface Product {
  id: number;
  name: string;
  price: number;
}

@Component({
  selector: 'app-resource-dependencies',
  template: `
    <div>
      <input [(ngModel)]="categoryValue" (ngModelChange)="category.set(categoryValue)">
      <input [(ngModel)]="searchValue" (ngModelChange)="searchTerm.set(searchValue)">
      
      <label>
        <input type="checkbox" [(ngModel)]="inStockValue" (change)="inStockOnly.set(inStockValue)">
        In Stock Only
      </label>
      
      @if (productsResource.isLoading()) {
        <p>Loading products...</p>
      }
      
      @if (productsResource.value(); as products) {
        <div *ngFor="let product of products">
          {{ product.name }} - ${{ product.price }}
        </div>
        <p>{{ products.length }} products found</p>
      }
    </div>
  `
})
export class ResourceDependenciesComponent {
  private http = inject(HttpClient);
  
  categoryValue = '';
  searchValue = '';
  inStockValue = false;
  
  category = signal('');
  searchTerm = signal('');
  inStockOnly = signal(false);
  
  // Resource depends on multiple signals
  productsResource = resource({
    request: () => ({
      category: this.category(),
      search: this.searchTerm(),
      inStock: this.inStockOnly()
    }),
    loader: ({ request }) => {
      const params: any = {};
      if (request.category) params.category = request.category;
      if (request.search) params.search = request.search;
      if (request.inStock) params.inStock = true;
      
      return this.http.get<Product[]>('/api/products', { params });
    }
  });
  
  constructor() {
    this.categoryValue = this.category();
    this.searchValue = this.searchTerm();
    this.inStockValue = this.inStockOnly();
  }
}
```

### Resource with Debouncing

```typescript
import { Component, signal, resource } from '@angular/core';
import { inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { debounceTime, of } from 'rxjs';

@Component({
  selector: 'app-debounced-resource',
  template: `
    <div>
      <input 
        [(ngModel)]="searchValue"
        (ngModelChange)="debouncedSearch.set(searchValue)"
        placeholder="Search...">
      
      @if (searchResource.isLoading()) {
        <p>Searching...</p>
      }
      
      @if (searchResource.value(); as results) {
        <div *ngFor="let result of results">
          {{ result.name }}
        </div>
        <p *ngIf="results.length === 0">No results found</p>
      }
    </div>
  `
})
export class DebouncedResourceComponent {
  private http = inject(HttpClient);
  
  searchValue = '';
  debouncedSearch = signal('');
  
  searchResource = resource({
    request: () => {
      const term = this.debouncedSearch();
      // Don't load if search term is too short
      return term.length >= 3 ? { term } : null;
    },
    loader: ({ request }) => {
      if (!request) {
        return of([]);
      }
      
      return this.http.get<any[]>(`/api/search?q=${request.term}`).pipe(
        debounceTime(300)
      );
    }
  });
}
```

## Async Data Management

### Pagination Resource

```typescript
import { Component, signal, resource, computed } from '@angular/core';
import { inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';

interface PaginatedResponse<T> {
  items: T[];
  page: number;
  pageSize: number;
  total: number;
}

interface User {
  id: number;
  name: string;
}

@Component({
  selector: 'app-pagination',
  template: `
    <div>
      @if (usersResource.isLoading()) {
        <p>Loading...</p>
      }
      
      @if (usersResource.value(); as response) {
        <div *ngFor="let user of response.items">
          {{ user.name }}
        </div>
        
        <div class="pagination">
          <button 
            (click)="previousPage()"
            [disabled]="page() === 1 || usersResource.isLoading()">
            Previous
          </button>
          
          <span>Page {{ page() }} of {{ totalPages() }}</span>
          
          <button 
            (click)="nextPage()"
            [disabled]="page() >= totalPages() || usersResource.isLoading()">
            Next
          </button>
        </div>
      }
    </div>
  `
})
export class PaginationComponent {
  private http = inject(HttpClient);
  
  page = signal(1);
  pageSize = signal(10);
  
  usersResource = resource({
    request: () => ({
      page: this.page(),
      pageSize: this.pageSize()
    }),
    loader: ({ request }) => {
      return this.http.get<PaginatedResponse<User>>('/api/users', {
        params: {
          page: request.page,
          pageSize: request.pageSize
        }
      });
    }
  });
  
  totalPages = computed(() => {
    const response = this.usersResource.value();
    if (!response) return 1;
    return Math.ceil(response.total / response.pageSize);
  });
  
  nextPage(): void {
    this.page.update(p => p + 1);
  }
  
  previousPage(): void {
    this.page.update(p => Math.max(1, p - 1));
  }
}
```

### Infinite Scroll Resource

```typescript
import { Component, signal, resource } from '@angular/core';
import { inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';

interface Post {
  id: number;
  title: string;
  content: string;
}

@Component({
  selector: 'app-infinite-scroll',
  template: `
    <div 
      class="feed"
      (scroll)="onScroll($event)">
      
      @if (postsResource.value(); as posts) {
        <div *ngFor="let post of allPosts()" class="post">
          <h3>{{ post.title }}</h3>
          <p>{{ post.content }}</p>
        </div>
      }
      
      @if (postsResource.isLoading()) {
        <p>Loading more...</p>
      }
      
      @if (hasMore()) {
        <button (click)="loadMore()">Load More</button>
      }
    </div>
  `
})
export class InfiniteScrollComponent {
  private http = inject(HttpClient);
  
  page = signal(1);
  allPosts = signal<Post[]>([]);
  hasMore = signal(true);
  
  postsResource = resource({
    request: () => ({ page: this.page() }),
    loader: ({ request }) => {
      return this.http.get<Post[]>('/api/posts', {
        params: { page: request.page, limit: 20 }
      });
    }
  });
  
  constructor() {
    effect(() => {
      const newPosts = this.postsResource.value();
      if (newPosts) {
        this.allPosts.update(posts => [...posts, ...newPosts]);
        this.hasMore.set(newPosts.length === 20);
      }
    });
  }
  
  loadMore(): void {
    this.page.update(p => p + 1);
  }
  
  onScroll(event: Event): void {
    const element = event.target as HTMLElement;
    const atBottom = element.scrollHeight - element.scrollTop === element.clientHeight;
    
    if (atBottom && this.hasMore() && !this.postsResource.isLoading()) {
      this.loadMore();
    }
  }
}
```

### Dependent Resources

```typescript
import { Component, signal, resource, computed } from '@angular/core';
import { inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';

interface User {
  id: number;
  name: string;
}

interface Post {
  id: number;
  userId: number;
  title: string;
}

@Component({
  selector: 'app-dependent-resources',
  template: `
    <div>
      @if (userResource.value(); as user) {
        <h2>{{ user.name }}'s Posts</h2>
        
        @if (postsResource.isLoading()) {
          <p>Loading posts...</p>
        }
        
        @if (postsResource.value(); as posts) {
          <div *ngFor="let post of posts">
            {{ post.title }}
          </div>
        }
      }
      
      <button (click)="userId.set(userId() + 1)">Next User</button>
    </div>
  `
})
export class DependentResourcesComponent {
  private http = inject(HttpClient);
  
  userId = signal(1);
  
  // First resource: Load user
  userResource = resource({
    request: () => ({ id: this.userId() }),
    loader: ({ request }) => {
      return this.http.get<User>(`/api/users/${request.id}`);
    }
  });
  
  // Second resource: Load user's posts (depends on first)
  postsResource = resource({
    request: () => {
      const user = this.userResource.value();
      return user ? { userId: user.id } : null;
    },
    loader: ({ request }) => {
      if (!request) return of([]);
      
      return this.http.get<Post[]>('/api/posts', {
        params: { userId: request.userId }
      });
    }
  });
}
```

## Error Handling

### Retry Logic

```typescript
import { Component, signal, resource } from '@angular/core';
import { inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { retry, catchError, of } from 'rxjs';

@Component({
  selector: 'app-retry',
  template: `
    <div>
      @if (dataResource.isLoading()) {
        <p>Loading... (Attempt {{ retryCount() }})</p>
      }
      
      @if (dataResource.error(); as error) {
        <div class="error">
          <p>Error: {{ error.message }}</p>
          <p>Failed after {{ maxRetries }} attempts</p>
          <button (click)="dataResource.reload()">Try Again</button>
        </div>
      }
      
      @if (dataResource.value(); as data) {
        <div>{{ data | json }}</div>
      }
    </div>
  `
})
export class RetryComponent {
  private http = inject(HttpClient);
  
  maxRetries = 3;
  retryCount = signal(0);
  
  dataResource = resource({
    request: () => ({}),
    loader: () => {
      this.retryCount.set(0);
      
      return this.http.get('/api/data').pipe(
        retry({
          count: this.maxRetries,
          delay: (error, retryCount) => {
            this.retryCount.set(retryCount);
            return of(null).pipe(delay(1000 * retryCount));
          }
        }),
        catchError(error => {
          throw new Error(`Failed after ${this.maxRetries} attempts: ${error.message}`);
        })
      );
    }
  });
}
```

### Fallback Data

```typescript
import { Component, signal, resource, computed } from '@angular/core';
import { inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { catchError, of } from 'rxjs';

interface Config {
  theme: string;
  language: string;
}

@Component({
  selector: 'app-fallback',
  template: `
    <div>
      @if (configResource.error()) {
        <p class="warning">Using default configuration</p>
      }
      
      <div>
        <p>Theme: {{ config().theme }}</p>
        <p>Language: {{ config().language }}</p>
      </div>
    </div>
  `
})
export class FallbackComponent {
  private http = inject(HttpClient);
  
  defaultConfig: Config = {
    theme: 'light',
    language: 'en'
  };
  
  configResource = resource({
    request: () => ({}),
    loader: () => {
      return this.http.get<Config>('/api/config').pipe(
        catchError(error => {
          console.error('Failed to load config:', error);
          return of(this.defaultConfig);
        })
      );
    }
  });
  
  config = computed(() => 
    this.configResource.value() ?? this.defaultConfig
  );
}
```

### Error Boundaries

```typescript
import { Component, signal, resource } from '@angular/core';
import { inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Component({
  selector: 'app-error-boundary',
  template: `
    <div>
      @if (dataResource.isLoading()) {
        <app-skeleton-loader />
      } @else if (dataResource.error(); as error) {
        <app-error-display 
          [error]="error"
          (retry)="dataResource.reload()" />
      } @else if (dataResource.value(); as data) {
        <app-data-display [data]="data" />
      }
    </div>
  `
})
export class ErrorBoundaryComponent {
  private http = inject(HttpClient);
  
  dataResource = resource({
    request: () => ({}),
    loader: () => this.http.get('/api/data')
  });
}

@Component({
  selector: 'app-error-display',
  template: `
    <div class="error-container">
      <h3>Oops! Something went wrong</h3>
      <p>{{ error.message }}</p>
      <button (click)="retry.emit()">Try Again</button>
      <button (click)="reportError()">Report Issue</button>
    </div>
  `
})
export class ErrorDisplayComponent {
  @Input() error!: Error;
  @Output() retry = new EventEmitter<void>();
  
  reportError(): void {
    // Send error to logging service
    console.error('Reported error:', this.error);
  }
}
```

## Loading States

### Skeleton Loaders

```typescript
import { Component, signal, resource } from '@angular/core';
import { inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';

interface Article {
  id: number;
  title: string;
  content: string;
  author: string;
}

@Component({
  selector: 'app-skeleton',
  template: `
    <div>
      @if (articleResource.isLoading()) {
        <div class="skeleton">
          <div class="skeleton-title"></div>
          <div class="skeleton-author"></div>
          <div class="skeleton-content"></div>
          <div class="skeleton-content"></div>
          <div class="skeleton-content"></div>
        </div>
      } @else if (articleResource.value(); as article) {
        <article>
          <h1>{{ article.title }}</h1>
          <p class="author">By {{ article.author }}</p>
          <div>{{ article.content }}</div>
        </article>
      }
    </div>
  `,
  styles: [`
    .skeleton > div {
      background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
      background-size: 200% 100%;
      animation: loading 1.5s infinite;
    }
    
    .skeleton-title {
      height: 32px;
      width: 70%;
      margin-bottom: 16px;
    }
    
    .skeleton-author {
      height: 16px;
      width: 30%;
      margin-bottom: 24px;
    }
    
    .skeleton-content {
      height: 16px;
      width: 100%;
      margin-bottom: 8px;
    }
    
    @keyframes loading {
      0% { background-position: 200% 0; }
      100% { background-position: -200% 0; }
    }
  `]
})
export class SkeletonComponent {
  private http = inject(HttpClient);
  
  articleId = signal(1);
  
  articleResource = resource({
    request: () => ({ id: this.articleId() }),
    loader: ({ request }) => {
      return this.http.get<Article>(`/api/articles/${request.id}`);
    }
  });
}
```

### Progress Indicators

```typescript
import { Component, signal, resource, computed } from '@angular/core';
import { inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Component({
  selector: 'app-progress',
  template: `
    <div>
      <div class="progress-container">
        @if (uploadResource.isLoading()) {
          <div class="progress-bar">
            <div 
              class="progress-fill" 
              [style.width.%]="progress()">
            </div>
          </div>
          <p>Uploading... {{ progress() }}%</p>
        }
      </div>
      
      @if (uploadResource.value(); as result) {
        <p>Upload complete! {{ result.filename }}</p>
      }
      
      <input 
        type="file" 
        (change)="onFileSelect($event)"
        [disabled]="uploadResource.isLoading()">
    </div>
  `
})
export class ProgressComponent {
  private http = inject(HttpClient);
  
  selectedFile = signal<File | null>(null);
  progress = signal(0);
  
  uploadResource = resource({
    request: () => {
      const file = this.selectedFile();
      return file ? { file } : null;
    },
    loader: ({ request }) => {
      if (!request) return of(null);
      
      const formData = new FormData();
      formData.append('file', request.file);
      
      return this.http.post('/api/upload', formData, {
        reportProgress: true,
        observe: 'events'
      }).pipe(
        tap(event => {
          if (event.type === HttpEventType.UploadProgress && event.total) {
            this.progress.set(Math.round(100 * event.loaded / event.total));
          }
        }),
        filter(event => event.type === HttpEventType.Response),
        map(event => (event as HttpResponse<any>).body)
      );
    }
  });
  
  onFileSelect(event: Event): void {
    const input = event.target as HTMLInputElement;
    if (input.files?.length) {
      this.selectedFile.set(input.files[0]);
      this.progress.set(0);
    }
  }
}
```

## Advanced Patterns

### Optimistic Updates

```typescript
import { Component, signal, resource } from '@angular/core';
import { inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';

interface Todo {
  id: number;
  text: string;
  completed: boolean;
}

@Component({
  selector: 'app-optimistic',
  template: `
    <div>
      @if (todosResource.value(); as todos) {
        <div *ngFor="let todo of displayTodos()">
          <input 
            type="checkbox"
            [checked]="todo.completed"
            (change)="toggleTodo(todo)"
            [disabled]="pendingToggle().has(todo.id)">
          <span [class.completed]="todo.completed">
            {{ todo.text }}
          </span>
          @if (pendingToggle().has(todo.id)) {
            <span class="spinner"></span>
          }
        </div>
      }
    </div>
  `
})
export class OptimisticComponent {
  private http = inject(HttpClient);
  
  pendingToggle = signal(new Set<number>());
  optimisticUpdates = signal<Map<number, Partial<Todo>>>(new Map());
  
  todosResource = resource({
    request: () => ({}),
    loader: () => this.http.get<Todo[]>('/api/todos')
  });
  
  displayTodos = computed(() => {
    const todos = this.todosResource.value() ?? [];
    const updates = this.optimisticUpdates();
    
    return todos.map(todo => {
      const update = updates.get(todo.id);
      return update ? { ...todo, ...update } : todo;
    });
  });
  
  toggleTodo(todo: Todo): void {
    const newState = !todo.completed;
    
    // Optimistic update
    this.optimisticUpdates.update(map => {
      const newMap = new Map(map);
      newMap.set(todo.id, { completed: newState });
      return newMap;
    });
    
    this.pendingToggle.update(set => new Set(set).add(todo.id));
    
    // Actual API call
    this.http.patch(`/api/todos/${todo.id}`, { completed: newState })
      .subscribe({
        next: () => {
          // Remove optimistic update on success
          this.optimisticUpdates.update(map => {
            const newMap = new Map(map);
            newMap.delete(todo.id);
            return newMap;
          });
          this.pendingToggle.update(set => {
            const newSet = new Set(set);
            newSet.delete(todo.id);
            return newSet;
          });
          this.todosResource.reload();
        },
        error: (error) => {
          // Revert optimistic update on error
          this.optimisticUpdates.update(map => {
            const newMap = new Map(map);
            newMap.delete(todo.id);
            return newMap;
          });
          this.pendingToggle.update(set => {
            const newSet = new Set(set);
            newSet.delete(todo.id);
            return newSet;
          });
          alert('Failed to update todo');
        }
      });
  }
}
```

### Polling Resource

```typescript
import { Component, signal, resource, effect } from '@angular/core';
import { inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { interval } from 'rxjs';

interface Status {
  state: 'pending' | 'processing' | 'complete';
  progress: number;
}

@Component({
  selector: 'app-polling',
  template: `
    <div>
      @if (statusResource.value(); as status) {
        <div>
          <p>State: {{ status.state }}</p>
          <p>Progress: {{ status.progress }}%</p>
          
          @if (status.state === 'complete') {
            <p>Processing complete!</p>
          }
        </div>
      }
      
      <button (click)="startPolling()">Start Job</button>
      <button (click)="stopPolling()">Stop Polling</button>
    </div>
  `
})
export class PollingComponent {
  private http = inject(HttpClient);
  
  jobId = signal<number | null>(null);
  polling = signal(false);
  
  statusResource = resource({
    request: () => {
      const id = this.jobId();
      return id ? { id } : null;
    },
    loader: ({ request }) => {
      if (!request) return of(null);
      return this.http.get<Status>(`/api/jobs/${request.id}/status`);
    }
  });
  
  constructor() {
    effect((onCleanup) => {
      if (!this.polling()) return;
      
      const subscription = interval(2000).subscribe(() => {
        const status = this.statusResource.value();
        if (status?.state === 'complete') {
          this.stopPolling();
        } else {
          this.statusResource.reload();
        }
      });
      
      onCleanup(() => subscription.unsubscribe());
    });
  }
  
  startPolling(): void {
    // Start job
    this.http.post<{ id: number }>('/api/jobs', {}).subscribe(response => {
      this.jobId.set(response.id);
      this.polling.set(true);
    });
  }
  
  stopPolling(): void {
    this.polling.set(false);
  }
}
```

## Common Mistakes

### Mistake 1: Mutating linkedSignal Source

```typescript
// BAD: Mutating the linked value
const source = signal({ count: 0 });
const linked = linkedSignal(() => source());

linked().count++; // Don't mutate!

// GOOD: Immutable updates
linked.update(val => ({ count: val.count + 1 }));
```

### Mistake 2: Not Handling Null Requests in resource

```typescript
// BAD: resource loader doesn't handle null request
const userId = signal<number | null>(null);

const userResource = resource({
  request: () => ({ id: userId() }), // id might be null!
  loader: ({ request }) => {
    return http.get(`/api/users/${request.id}`); // Error if null!
  }
});

// GOOD: Handle null requests
const userResource = resource({
  request: () => {
    const id = userId();
    return id ? { id } : null;
  },
  loader: ({ request }) => {
    if (!request) return of(null);
    return http.get(`/api/users/${request.id}`);
  }
});
```

### Mistake 3: Forgetting to Check Loading State

```typescript
// BAD: Not checking loading state
@Component({
  template: `
    <div *ngFor="let item of dataResource.value()">
      {{ item.name }}
    </div>
  `
})
export class BadComponent {
  // Might be undefined while loading!
}

// GOOD: Check loading state
@Component({
  template: `
    @if (dataResource.isLoading()) {
      <p>Loading...</p>
    } @else if (dataResource.value(); as data) {
      <div *ngFor="let item of data">
        {{ item.name }}
      </div>
    }
  `
})
export class GoodComponent {}
```

### Mistake 4: Not Handling Errors

```typescript
// BAD: Ignoring errors
@Component({
  template: `
    @if (dataResource.value(); as data) {
      {{ data }}
    }
  `
})
export class BadComponent {}

// GOOD: Handle errors
@Component({
  template: `
    @if (dataResource.error(); as error) {
      <p class="error">{{ error.message }}</p>
      <button (click)="dataResource.reload()">Retry</button>
    } @else if (dataResource.value(); as data) {
      {{ data }}
    }
  `
})
export class GoodComponent {}
```

### Mistake 5: Unnecessary Resource Dependencies

```typescript
// BAD: Too many dependencies cause unnecessary reloads
const filter = signal('');
const sortOrder = signal('asc');
const page = signal(1);

const dataResource = resource({
  request: () => ({
    filter: filter(),
    sort: sortOrder(),
    page: page()
  }),
  loader: ({ request }) => {
    // Reloads on ANY change, even unrelated ones
  }
});

// GOOD: Group related dependencies
const searchParams = computed(() => ({
  filter: filter(),
  sort: sortOrder()
}));

const dataResource = resource({
  request: () => ({
    ...searchParams(),
    page: page()
  }),
  loader: ({ request }) => {
    // More controlled reloading
  }
});
```

## Best Practices

### 1. Use linkedSignal for Form Synchronization

```typescript
const serverData = signal({ name: 'John', age: 30 });
const formData = linkedSignal(() => ({ ...serverData() }));

// Form automatically resets when server data changes
```

### 2. Always Handle All Resource States

```typescript
@Component({
  template: `
    @if (resource.isLoading()) {
      <app-loading />
    } @else if (resource.error(); as error) {
      <app-error [error]="error" />
    } @else if (resource.value(); as data) {
      <app-content [data]="data" />
    } @else {
      <app-empty />
    }
  `
})
export class ResourceComponent {}
```

### 3. Use Computed for Derived Resource State

```typescript
const dataResource = resource({...});

const hasData = computed(() => 
  dataResource.value()?.length > 0
);

const isEmpty = computed(() => 
  !dataResource.isLoading() && 
  !dataResource.error() && 
  dataResource.value()?.length === 0
);
```

### 4. Implement Request Deduplication

```typescript
const lastRequest = signal<any>(null);

const dataResource = resource({
  request: () => {
    const newRequest = { id: userId() };
    if (JSON.stringify(newRequest) === JSON.stringify(lastRequest())) {
      return null; // Skip duplicate request
    }
    lastRequest.set(newRequest);
    return newRequest;
  },
  loader: ({ request }) => {
    if (!request) return of(null);
    return http.get(`/api/data/${request.id}`);
  }
});
```

### 5. Cleanup Resources Properly

```typescript
@Component({...})
export class ResourceComponent implements OnDestroy {
  dataResource = resource({...});
  
  ngOnDestroy(): void {
    // Resources auto-cleanup, but manual cleanup if needed
    this.dataResource.destroy();
  }
}
```

## Interview Questions

### Q1: What is the difference between linkedSignal and computed?

**Answer:** `linkedSignal()` creates a writable signal that derives its initial value from another signal but can be independently modified. `computed()` creates a read-only signal that always reflects the computed value of its dependencies. Use linkedSignal for values that start computed but can be overridden, like form defaults.

### Q2: When should you use resource instead of manual HTTP calls?

**Answer:** Use `resource()` when you need automatic loading state management, error handling, and reactive data loading based on signal dependencies. It's ideal for data that should reload when dependencies change, with built-in loading/error/success states.

### Q3: How does resource handle concurrent requests?

**Answer:** When a resource's request signal changes while a previous request is pending, Angular automatically cancels the previous request and starts the new one, preventing race conditions and ensuring only the latest request's data is used.

### Q4: What are the main states of a resource?

**Answer:** A resource has four main states: loading (isLoading()), value (successful data), error (error()), and idle (no request made). You should handle all these states in your template for a good user experience.

### Q5: How do linkedSignals help with form management?

**Answer:** linkedSignals automatically reset form values when the source data changes, which is perfect for edit forms that should reset when loading a different entity, while still allowing independent modifications during editing.

## Key Takeaways

1. linkedSignal creates writable signals that derive from other signals
2. resource provides declarative async data management
3. Resources automatically handle loading, error, and success states
4. linkedSignal is perfect for form synchronization patterns
5. Resources cancel previous requests when dependencies change
6. Always handle all resource states in templates
7. Use computed for derived resource state
8. linkedSignals reset when source signals change
9. Resources enable reactive data loading patterns
10. Both primitives simplify complex state management scenarios

## Resources

- [Angular Signals Guide](https://angular.dev/guide/signals)
- [linkedSignal API Documentation](https://angular.dev/api/core/linkedSignal)
- [resource API Documentation](https://angular.dev/api/core/resource)
- [Async Data Management with Signals](https://blog.angular.io/async-data-signals)
- [Advanced Signal Patterns](https://angular.dev/guide/signals/advanced)
