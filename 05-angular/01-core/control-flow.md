# Angular Control Flow

## Learning Objectives
- Master the new @if, @else, and @else if control flow syntax (Angular 17+)
- Implement @for loops with track expressions for optimal performance
- Use @switch statements for complex conditional rendering
- Leverage @defer for lazy loading and performance optimization
- Understand @let for template-scoped variable declarations
- Migrate from legacy structural directives (*ngIf, *ngFor, *ngSwitch)
- Optimize rendering performance with proper track functions
- Debug common control flow patterns and errors

## Introduction

Angular 17 introduced a revolutionary built-in control flow syntax that replaces the traditional structural directives (*ngIf, *ngFor, *ngSwitch) with a more intuitive, performant, and type-safe approach. This new syntax uses `@` prefixed blocks that are processed directly by the Angular compiler, resulting in better performance, improved type checking, and a more familiar programming experience.

The new control flow syntax brings several advantages: better runtime performance through compiler optimizations, improved type narrowing in templates, more readable code without asterisk prefixes, and built-in features like @defer for progressive loading. This modernization aligns Angular's template syntax more closely with JavaScript control flow while maintaining the framework's reactive nature.

## @if Control Flow

The @if block replaces *ngIf with a more intuitive syntax that supports inline else conditions and better type narrowing.

### Basic @if Usage

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-conditional-rendering',
  standalone: true,
  template: `
    <div>
      <!-- Simple @if -->
      @if (isLoggedIn) {
        <p>Welcome back, {{ username }}!</p>
      }

      <!-- @if with @else -->
      @if (hasPermission) {
        <button>Edit Profile</button>
      } @else {
        <p>You don't have permission to edit</p>
      }

      <!-- @if with @else if -->
      @if (status === 'loading') {
        <p>Loading...</p>
      } @else if (status === 'error') {
        <p>Error occurred: {{ errorMessage }}</p>
      } @else if (status === 'success') {
        <p>Data loaded successfully!</p>
      } @else {
        <p>Ready to load data</p>
      }

      <!-- Nested @if blocks -->
      @if (user) {
        <div class="user-card">
          <h3>{{ user.name }}</h3>
          @if (user.email) {
            <p>Email: {{ user.email }}</p>
          }
          @if (user.isPremium) {
            <span class="badge">Premium Member</span>
          }
        </div>
      }
    </div>
  `
})
export class ConditionalRenderingComponent {
  isLoggedIn = true;
  username = 'John Doe';
  hasPermission = false;
  status: 'idle' | 'loading' | 'error' | 'success' = 'idle';
  errorMessage = '';
  
  user = {
    name: 'Jane Smith',
    email: 'jane@example.com',
    isPremium: true
  };
}
```

### Type Narrowing with @if

One of the powerful features of @if is automatic type narrowing:

```typescript
import { Component } from '@angular/core';

interface User {
  id: number;
  name: string;
  email: string;
}

@Component({
  selector: 'app-type-narrowing',
  standalone: true,
  template: `
    <div>
      <!-- Type narrowing: user is User inside @if block -->
      @if (user) {
        <div>
          <!-- TypeScript knows user is not null here -->
          <h3>{{ user.name }}</h3>
          <p>{{ user.email }}</p>
          <button (click)="logUserId(user.id)">Show ID</button>
        </div>
      } @else {
        <p>No user found</p>
      }

      <!-- Type narrowing with type guards -->
      @if (data && isUser(data)) {
        <!-- TypeScript knows data is User here -->
        <p>User: {{ data.name }}</p>
      }

      <!-- Discriminated unions -->
      @if (result.type === 'success') {
        <!-- TypeScript knows result has 'data' property -->
        <p>Success: {{ result.data }}</p>
      } @else if (result.type === 'error') {
        <!-- TypeScript knows result has 'error' property -->
        <p>Error: {{ result.error }}</p>
      }

      <!-- Array length check with type narrowing -->
      @if (items.length > 0) {
        <!-- Safe to access array elements -->
        <p>First item: {{ items[0] }}</p>
      }
    </div>
  `
})
export class TypeNarrowingComponent {
  user: User | null = {
    id: 1,
    name: 'Alice Johnson',
    email: 'alice@example.com'
  };

  data: User | { other: string } = {
    id: 2,
    name: 'Bob Smith',
    email: 'bob@example.com'
  };

  result: 
    | { type: 'success'; data: string }
    | { type: 'error'; error: string }
    | { type: 'loading' } = { type: 'success', data: 'Hello' };

  items: string[] = ['Item 1', 'Item 2'];

  isUser(value: any): value is User {
    return value && typeof value.name === 'string';
  }

  logUserId(id: number): void {
    console.log('User ID:', id);
  }
}
```

### Complex Conditional Logic

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-complex-conditions',
  standalone: true,
  template: `
    <div>
      <!-- Multiple conditions -->
      @if (isAuthenticated && hasRole('admin')) {
        <button>Admin Panel</button>
      }

      <!-- Complex expressions -->
      @if (age >= 18 && age <= 65 && isEmployed) {
        <p>Eligible for standard plan</p>
      }

      <!-- Logical OR conditions -->
      @if (isOwner || isEditor || hasPermission('write')) {
        <button>Edit Document</button>
      }

      <!-- Negation -->
      @if (!isDeleted && !isArchived) {
        <div class="active-item">
          <h3>{{ title }}</h3>
        </div>
      }

      <!-- Comparison operators -->
      @if (score > 90) {
        <span class="grade-a">Grade: A</span>
      } @else if (score > 80) {
        <span class="grade-b">Grade: B</span>
      } @else if (score > 70) {
        <span class="grade-c">Grade: C</span>
      } @else {
        <span class="grade-f">Grade: F</span>
      }

      <!-- Ternary-like pattern with @if/@else -->
      @if (temperature > 30) {
        <span>🔥 Hot</span>
      } @else if (temperature > 20) {
        <span>☀️ Warm</span>
      } @else if (temperature > 10) {
        <span>🌤️ Cool</span>
      } @else {
        <span>❄️ Cold</span>
      }

      <!-- Combining with method calls -->
      @if (canUserEditPost(currentUser, post)) {
        <button (click)="editPost()">Edit Post</button>
      }

      <!-- Nullish checks -->
      @if (data ?? false) {
        <p>Data exists: {{ data }}</p>
      }
    </div>
  `
})
export class ComplexConditionsComponent {
  isAuthenticated = true;
  age = 30;
  isEmployed = true;
  isOwner = false;
  isEditor = true;
  isDeleted = false;
  isArchived = false;
  title = 'Active Document';
  score = 85;
  temperature = 25;
  data: string | null = 'Sample data';

  currentUser = { id: 1, role: 'editor' };
  post = { authorId: 1, isLocked: false };

  hasRole(role: string): boolean {
    return this.currentUser.role === role;
  }

  hasPermission(permission: string): boolean {
    return permission === 'write' && this.isEditor;
  }

  canUserEditPost(user: any, post: any): boolean {
    return user.id === post.authorId && !post.isLocked;
  }

  editPost(): void {
    console.log('Editing post...');
  }
}
```

## @for Control Flow

The @for block replaces *ngFor with mandatory track expressions for better performance and change detection.

### Basic @for Usage

```typescript
import { Component } from '@angular/core';

interface Todo {
  id: number;
  title: string;
  completed: boolean;
}

@Component({
  selector: 'app-list-rendering',
  standalone: true,
  template: `
    <div>
      <!-- Basic @for with track -->
      <ul>
        @for (item of items; track item) {
          <li>{{ item }}</li>
        }
      </ul>

      <!-- @for with index -->
      <ul>
        @for (user of users; track user.id; let i = $index) {
          <li>{{ i + 1 }}. {{ user.name }}</li>
        }
      </ul>

      <!-- @for with multiple context variables -->
      <ul>
        @for (todo of todos; track todo.id; let idx = $index; let isFirst = $first; let isLast = $last) {
          <li 
            [class.first]="isFirst"
            [class.last]="isLast">
            {{ idx }}. {{ todo.title }}
            @if (todo.completed) {
              <span>✓</span>
            }
          </li>
        }
      </ul>

      <!-- @for with @empty block -->
      <ul>
        @for (product of products; track product.id) {
          <li>{{ product.name }} - ${{ product.price }}</li>
        } @empty {
          <li>No products available</li>
        }
      </ul>

      <!-- Nested @for loops -->
      @for (category of categories; track category.id) {
        <div class="category">
          <h3>{{ category.name }}</h3>
          <ul>
            @for (item of category.items; track item.id) {
              <li>{{ item.name }}</li>
            }
          </ul>
        </div>
      }
    </div>
  `,
  styles: [`
    .first { font-weight: bold; }
    .last { font-style: italic; }
    .category { margin-bottom: 20px; }
  `]
})
export class ListRenderingComponent {
  items = ['Apple', 'Banana', 'Cherry', 'Date'];

  users = [
    { id: 1, name: 'Alice' },
    { id: 2, name: 'Bob' },
    { id: 3, name: 'Charlie' }
  ];

  todos: Todo[] = [
    { id: 1, title: 'Learn Angular', completed: true },
    { id: 2, title: 'Build app', completed: false },
    { id: 3, title: 'Deploy', completed: false }
  ];

  products: Array<{ id: number; name: string; price: number }> = [];

  categories = [
    {
      id: 1,
      name: 'Fruits',
      items: [
        { id: 11, name: 'Apple' },
        { id: 12, name: 'Banana' }
      ]
    },
    {
      id: 2,
      name: 'Vegetables',
      items: [
        { id: 21, name: 'Carrot' },
        { id: 22, name: 'Broccoli' }
      ]
    }
  ];
}
```

### Track Expressions and Performance

The track expression is mandatory in @for and crucial for performance:

```typescript
import { Component } from '@angular/core';

interface User {
  id: number;
  name: string;
  email: string;
}

@Component({
  selector: 'app-track-performance',
  standalone: true,
  template: `
    <div>
      <!-- ✅ BEST: Track by unique ID -->
      <ul>
        @for (user of users; track user.id) {
          <li>{{ user.name }} ({{ user.email }})</li>
        }
      </ul>

      <!-- ✅ GOOD: Track by identity when items are primitives -->
      <ul>
        @for (name of names; track name) {
          <li>{{ name }}</li>
        }
      </ul>

      <!-- ✅ GOOD: Track by index when order matters -->
      <ul>
        @for (item of dynamicItems; track $index) {
          <li>Position {{ $index }}: {{ item }}</li>
        }
      </ul>

      <!-- ✅ GOOD: Track by composite key -->
      <ul>
        @for (order of orders; track order.customerId + '-' + order.orderId) {
          <li>Order #{{ order.orderId }} for Customer #{{ order.customerId }}</li>
        }
      </ul>

      <!-- ❌ AVOID: Track by index for dynamic lists with IDs -->
      <!-- This defeats the purpose of efficient change detection -->
      <!--
      @for (user of users; track $index) {
        <li>{{ user.name }}</li>
      }
      -->

      <!-- Performance comparison demo -->
      <div>
        <button (click)="shuffleUsers()">Shuffle Users</button>
        <button (click)="addUser()">Add User</button>
        <button (click)="removeUser()">Remove First User</button>
        
        <h3>Tracked by ID (efficient):</h3>
        <ul>
          @for (user of users; track user.id) {
            <li>{{ user.name }} - {{ user.email }}</li>
          }
        </ul>
      </div>
    </div>
  `
})
export class TrackPerformanceComponent {
  users: User[] = [
    { id: 1, name: 'Alice', email: 'alice@example.com' },
    { id: 2, name: 'Bob', email: 'bob@example.com' },
    { id: 3, name: 'Charlie', email: 'charlie@example.com' }
  ];

  names = ['John', 'Jane', 'Jack'];
  
  dynamicItems = ['First', 'Second', 'Third'];

  orders = [
    { customerId: 101, orderId: 1001 },
    { customerId: 102, orderId: 1002 },
    { customerId: 101, orderId: 1003 }
  ];

  private nextId = 4;

  shuffleUsers(): void {
    // Shuffle array
    this.users = [...this.users].sort(() => Math.random() - 0.5);
    console.log('Users shuffled - DOM efficiently updated with track by id');
  }

  addUser(): void {
    this.users.push({
      id: this.nextId++,
      name: `User ${this.nextId}`,
      email: `user${this.nextId}@example.com`
    });
  }

  removeUser(): void {
    this.users.shift();
  }
}
```

### Context Variables in @for

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-for-context',
  standalone: true,
  template: `
    <div>
      <!-- All available context variables -->
      <ul>
        @for (item of items; track item.id; 
          let idx = $index;
          let count = $count;
          let isFirst = $first;
          let isLast = $last;
          let isEven = $even;
          let isOdd = $odd;
        ) {
          <li 
            [class.even]="isEven"
            [class.odd]="isOdd"
            [class.first]="isFirst"
            [class.last]="isLast">
            Item {{ idx + 1 }} of {{ count }}: {{ item.name }}
          </li>
        }
      </ul>

      <!-- Using context for conditional styling -->
      <div class="table">
        @for (row of tableData; track row.id; let i = $index; let isOdd = $odd) {
          <div 
            class="row"
            [class.striped]="isOdd">
            <span class="cell">{{ i + 1 }}</span>
            <span class="cell">{{ row.name }}</span>
            <span class="cell">{{ row.value }}</span>
          </div>
        }
      </div>

      <!-- First and last styling -->
      @for (step of steps; track step.id; let isFirst = $first; let isLast = $last) {
        <div class="step">
          @if (!isFirst) {
            <div class="connector"></div>
          }
          <div class="step-content">{{ step.title }}</div>
          @if (!isLast) {
            <div class="arrow">→</div>
          }
        </div>
      }

      <!-- Progress indicator using $index and $count -->
      @for (task of tasks; track task.id; let idx = $index; let total = $count) {
        <div class="task">
          <span>{{ task.name }}</span>
          <span class="progress">{{ idx + 1 }}/{{ total }}</span>
        </div>
      }
    </div>
  `,
  styles: [`
    .even { background-color: #f0f0f0; }
    .odd { background-color: #ffffff; }
    .first { border-top: 2px solid #3498db; }
    .last { border-bottom: 2px solid #3498db; }
    .row.striped { background-color: #f9f9f9; }
    .step { display: flex; align-items: center; }
    .connector { width: 20px; height: 2px; background: #ddd; }
    .arrow { margin: 0 10px; }
  `]
})
export class ForContextComponent {
  items = [
    { id: 1, name: 'Item One' },
    { id: 2, name: 'Item Two' },
    { id: 3, name: 'Item Three' },
    { id: 4, name: 'Item Four' }
  ];

  tableData = [
    { id: 1, name: 'Row 1', value: 100 },
    { id: 2, name: 'Row 2', value: 200 },
    { id: 3, name: 'Row 3', value: 300 }
  ];

  steps = [
    { id: 1, title: 'Step 1: Initialize' },
    { id: 2, title: 'Step 2: Process' },
    { id: 3, title: 'Step 3: Complete' }
  ];

  tasks = [
    { id: 1, name: 'Task A' },
    { id: 2, name: 'Task B' },
    { id: 3, name: 'Task C' }
  ];
}
```

## @switch Control Flow

The @switch block provides a cleaner alternative to multiple @if/@else if chains:

```typescript
import { Component } from '@angular/core';

type UserRole = 'admin' | 'editor' | 'viewer' | 'guest';
type Status = 'idle' | 'loading' | 'success' | 'error';

@Component({
  selector: 'app-switch-statement',
  standalone: true,
  template: `
    <div>
      <!-- Basic @switch -->
      @switch (userRole) {
        @case ('admin') {
          <div class="admin-panel">
            <h2>Admin Dashboard</h2>
            <button>Manage Users</button>
            <button>System Settings</button>
          </div>
        }
        @case ('editor') {
          <div class="editor-panel">
            <h2>Editor Dashboard</h2>
            <button>Create Content</button>
            <button>Edit Content</button>
          </div>
        }
        @case ('viewer') {
          <div class="viewer-panel">
            <h2>Viewer Dashboard</h2>
            <p>Read-only access</p>
          </div>
        }
        @default {
          <div class="guest-panel">
            <h2>Guest Access</h2>
            <p>Please log in for full access</p>
          </div>
        }
      }

      <!-- @switch with status handling -->
      @switch (status) {
        @case ('loading') {
          <div class="loading">
            <div class="spinner"></div>
            <p>Loading data...</p>
          </div>
        }
        @case ('success') {
          <div class="success">
            <h3>✓ Success!</h3>
            <p>{{ data }}</p>
          </div>
        }
        @case ('error') {
          <div class="error">
            <h3>✗ Error</h3>
            <p>{{ errorMessage }}</p>
            <button (click)="retry()">Retry</button>
          </div>
        }
        @case ('idle') {
          <button (click)="loadData()">Load Data</button>
        }
      }

      <!-- @switch with theme selection -->
      @switch (theme) {
        @case ('dark') {
          <div class="theme-dark">
            <h3>🌙 Dark Theme</h3>
            <p>Easy on the eyes</p>
          </div>
        }
        @case ('light') {
          <div class="theme-light">
            <h3>☀️ Light Theme</h3>
            <p>Classic look</p>
          </div>
        }
        @case ('auto') {
          <div class="theme-auto">
            <h3>🔄 Auto Theme</h3>
            <p>Follows system preference</p>
          </div>
        }
        @default {
          <div>Using default theme</div>
        }
      }

      <!-- Multiple cases leading to same block (workaround) -->
      @switch (dayType) {
        @case ('monday') {
          <p>{{ getWeekdayMessage() }}</p>
        }
        @case ('tuesday') {
          <p>{{ getWeekdayMessage() }}</p>
        }
        @case ('wednesday') {
          <p>{{ getWeekdayMessage() }}</p>
        }
        @case ('thursday') {
          <p>{{ getWeekdayMessage() }}</p>
        }
        @case ('friday') {
          <p>{{ getWeekdayMessage() }}</p>
        }
        @case ('saturday') {
          <p>{{ getWeekendMessage() }}</p>
        }
        @case ('sunday') {
          <p>{{ getWeekendMessage() }}</p>
        }
      }

      <!-- Nested @switch -->
      @switch (category) {
        @case ('electronics') {
          <h3>Electronics</h3>
          @switch (subcategory) {
            @case ('phones') {
              <p>Browse smartphones</p>
            }
            @case ('laptops') {
              <p>Browse laptops</p>
            }
            @default {
              <p>Browse all electronics</p>
            }
          }
        }
        @case ('clothing') {
          <h3>Clothing</h3>
          @switch (subcategory) {
            @case ('mens') {
              <p>Men's clothing</p>
            }
            @case ('womens') {
              <p>Women's clothing</p>
            }
            @default {
              <p>Browse all clothing</p>
            }
          }
        }
        @default {
          <p>Browse all categories</p>
        }
      }
    </div>
  `,
  styles: [`
    .admin-panel, .editor-panel, .viewer-panel, .guest-panel {
      padding: 20px;
      border-radius: 8px;
      margin: 10px 0;
    }
    .admin-panel { background-color: #e74c3c; color: white; }
    .editor-panel { background-color: #3498db; color: white; }
    .viewer-panel { background-color: #95a5a6; color: white; }
    .loading { text-align: center; }
    .success { color: #27ae60; }
    .error { color: #e74c3c; }
    .theme-dark { background: #2c3e50; color: white; padding: 20px; }
    .theme-light { background: #ecf0f1; padding: 20px; }
  `]
})
export class SwitchStatementComponent {
  userRole: UserRole = 'editor';
  status: Status = 'idle';
  data = '';
  errorMessage = '';
  theme: 'dark' | 'light' | 'auto' = 'dark';
  dayType = 'monday';
  category = 'electronics';
  subcategory = 'phones';

  loadData(): void {
    this.status = 'loading';
    setTimeout(() => {
      if (Math.random() > 0.3) {
        this.status = 'success';
        this.data = 'Data loaded successfully!';
      } else {
        this.status = 'error';
        this.errorMessage = 'Failed to load data';
      }
    }, 1000);
  }

  retry(): void {
    this.loadData();
  }

  getWeekdayMessage(): string {
    return 'It\'s a weekday - time to work!';
  }

  getWeekendMessage(): string {
    return 'It\'s the weekend - time to relax!';
  }
}
```

## @defer Control Flow

The @defer block enables lazy loading and performance optimization through progressive hydration:

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-defer-loading',
  standalone: true,
  template: `
    <div>
      <!-- Basic @defer - loads when browser is idle -->
      @defer {
        <app-heavy-component></app-heavy-component>
      } @placeholder {
        <div class="placeholder">Loading...</div>
      }

      <!-- @defer on viewport - loads when element enters viewport -->
      <h2>Scroll down to load image</h2>
      <div style="height: 1000px;"></div>
      @defer (on viewport) {
        <img src="large-image.jpg" alt="Large image">
        <app-image-gallery></app-image-gallery>
      } @placeholder {
        <div class="placeholder-image">📷 Image will load when visible</div>
      } @loading (minimum 500ms) {
        <div class="spinner">Loading image...</div>
      }

      <!-- @defer on interaction - loads when user interacts -->
      @defer (on interaction) {
        <app-comments-section></app-comments-section>
      } @placeholder {
        <button>Click to load comments</button>
      }

      <!-- @defer on hover -->
      @defer (on hover) {
        <app-tooltip-content></app-tooltip-content>
      } @placeholder {
        <span>Hover for more info</span>
      }

      <!-- @defer on immediate - loads immediately but non-blocking -->
      @defer (on immediate) {
        <app-analytics-tracker></app-analytics-tracker>
      }

      <!-- @defer on timer -->
      @defer (on timer(5000ms)) {
        <app-promotional-banner></app-promotional-banner>
      } @placeholder {
        <div>Banner will appear in 5 seconds...</div>
      }

      <!-- @defer with prefetch strategies -->
      @defer (on interaction; prefetch on idle) {
        <app-video-player></app-video-player>
      } @placeholder {
        <div class="video-placeholder">
          <button>Click to play video</button>
          <p>Video will be prefetched when browser is idle</p>
        </div>
      }

      <!-- Multiple conditions with OR -->
      @defer (on hover, interaction) {
        <app-dropdown-menu></app-dropdown-menu>
      } @placeholder {
        <button>Hover or click for menu</button>
      }

      <!-- Complete example with all blocks -->
      @defer (on viewport; prefetch on idle) {
        <app-data-table [data]="tableData"></app-data-table>
      } @loading (minimum 1s) {
        <div class="table-loading">
          <div class="spinner"></div>
          <p>Loading table...</p>
        </div>
      } @placeholder (minimum 500ms) {
        <div class="table-placeholder">
          <p>Table will load when scrolled into view</p>
          <p>Prefetching will start when browser is idle</p>
        </div>
      } @error {
        <div class="error-state">
          <p>Failed to load table</p>
          <button (click)="reloadTable()">Retry</button>
        </div>
      }

      <!-- Conditional @defer based on feature flags -->
      @defer (on idle; when enableLazyLoading) {
        <app-optional-feature></app-optional-feature>
      } @placeholder {
        @if (enableLazyLoading) {
          <p>Feature will load soon...</p>
        } @else {
          <p>Feature is disabled</p>
        }
      }

      <!-- Nested @defer blocks -->
      @defer (on viewport) {
        <div class="section">
          <h3>Main Content</h3>
          @defer (on interaction) {
            <app-nested-content></app-nested-content>
          } @placeholder {
            <button>Load nested content</button>
          }
        </div>
      } @placeholder {
        <div class="section-placeholder">
          Scroll to load this section
        </div>
      }
    </div>
  `,
  styles: [`
    .placeholder, .placeholder-image, .table-placeholder {
      padding: 20px;
      border: 2px dashed #ddd;
      text-align: center;
      border-radius: 8px;
      margin: 10px 0;
    }
    .spinner {
      border: 3px solid #f3f3f3;
      border-top: 3px solid #3498db;
      border-radius: 50%;
      width: 40px;
      height: 40px;
      animation: spin 1s linear infinite;
      margin: 0 auto;
    }
    @keyframes spin {
      0% { transform: rotate(0deg); }
      100% { transform: rotate(360deg); }
    }
    .error-state {
      background-color: #fee;
      padding: 20px;
      border-radius: 8px;
      color: #c00;
    }
  `]
})
export class DeferLoadingComponent {
  tableData = Array.from({ length: 100 }, (_, i) => ({
    id: i + 1,
    name: `Item ${i + 1}`,
    value: Math.random() * 1000
  }));

  enableLazyLoading = true;

  reloadTable(): void {
    console.log('Reloading table...');
    // Reload logic
  }
}
```

### @defer Triggers and Strategies

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-defer-strategies',
  standalone: true,
  template: `
    <div>
      <!-- 1. idle - loads when browser is idle (default) -->
      @defer {
        <app-non-critical-content></app-non-critical-content>
      }

      <!-- 2. viewport - loads when enters viewport -->
      @defer (on viewport) {
        <app-below-fold-content></app-below-fold-content>
      }

      <!-- 3. viewport with custom element reference -->
      <div #trigger></div>
      @defer (on viewport(trigger)) {
        <app-content-near-trigger></app-content-near-trigger>
      }

      <!-- 4. interaction - loads on click, keydown, etc -->
      @defer (on interaction) {
        <app-modal-dialog></app-modal-dialog>
      } @placeholder {
        <button>Open Dialog</button>
      }

      <!-- 5. interaction with element reference -->
      <button #loadBtn>Load Content</button>
      @defer (on interaction(loadBtn)) {
        <app-button-triggered-content></app-button-triggered-content>
      }

      <!-- 6. hover - loads on mouseenter/focusin -->
      @defer (on hover) {
        <app-hover-card></app-hover-card>
      } @placeholder {
        <div class="hover-trigger">Hover me</div>
      }

      <!-- 7. hover with element reference -->
      <div #hoverZone class="hover-zone">Hover Zone</div>
      @defer (on hover(hoverZone)) {
        <app-hover-content></app-hover-content>
      }

      <!-- 8. immediate - loads immediately but async -->
      @defer (on immediate) {
        <app-critical-immediate></app-critical-immediate>
      }

      <!-- 9. timer - loads after specified time -->
      @defer (on timer(3000ms)) {
        <app-delayed-content></app-delayed-content>
      }

      <!-- 10. Prefetch strategies -->
      
      <!-- Prefetch on idle -->
      @defer (on interaction; prefetch on idle) {
        <app-prefetch-idle></app-prefetch-idle>
      }

      <!-- Prefetch on timer -->
      @defer (on interaction; prefetch on timer(5s)) {
        <app-prefetch-timer></app-prefetch-timer>
      }

      <!-- Prefetch on hover -->
      @defer (on interaction; prefetch on hover) {
        <app-prefetch-hover></app-prefetch-hover>
      }

      <!-- Prefetch immediately -->
      @defer (on viewport; prefetch on immediate) {
        <app-prefetch-immediate></app-prefetch-immediate>
      }

      <!-- 11. Multiple triggers (OR logic) -->
      @defer (on hover, interaction, timer(10s)) {
        <app-multiple-triggers></app-multiple-triggers>
      } @placeholder {
        <p>Hover, click, or wait 10 seconds</p>
      }

      <!-- 12. Conditional with when -->
      @defer (when showFeature && userIsAuthenticated) {
        <app-conditional-feature></app-conditional-feature>
      }

      <!-- 13. Combined conditions and triggers -->
      @defer (on viewport; when dataLoaded) {
        <app-data-visualization [data]="visualData"></app-data-visualization>
      } @placeholder {
        @if (!dataLoaded) {
          <p>Waiting for data...</p>
        } @else {
          <p>Scroll down to view visualization</p>
        }
      }
    </div>
  `
})
export class DeferStrategiesComponent {
  showFeature = true;
  userIsAuthenticated = true;
  dataLoaded = false;
  visualData: any[] = [];

  ngOnInit(): void {
    // Simulate data loading
    setTimeout(() => {
      this.dataLoaded = true;
      this.visualData = [/* data */];
    }, 2000);
  }
}
```

## @let for Template Variables

The @let declaration creates template-scoped variables for reusing expressions:

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-template-variables',
  standalone: true,
  template: `
    <div>
      <!-- Basic @let usage -->
      @let userName = user.firstName + ' ' + user.lastName;
      <h1>Welcome, {{ userName }}!</h1>
      <p>Profile for {{ userName }}</p>

      <!-- @let with complex expressions -->
      @let fullAddress = user.address.street + ', ' + user.address.city + ', ' + user.address.country;
      <p>Address: {{ fullAddress }}</p>
      <button (click)="copyToClipboard(fullAddress)">Copy Address</button>

      <!-- @let for avoiding repeated calculations -->
      @let discountedPrice = product.price * (1 - product.discount);
      @let tax = discountedPrice * 0.1;
      @let finalPrice = discountedPrice + tax;
      
      <div class="price-breakdown">
        <p>Original: ${{ product.price }}</p>
        <p>After {{ product.discount * 100 }}% discount: ${{ discountedPrice }}</p>
        <p>Tax (10%): ${{ tax }}</p>
        <p><strong>Final Price: ${{ finalPrice }}</strong></p>
      </div>

      <!-- @let with safe navigation -->
      @let displayName = user?.profile?.displayName ?? user?.email ?? 'Anonymous';
      <p>Display Name: {{ displayName }}</p>

      <!-- @let with array methods -->
      @let activeUsers = users.filter(u => u.isActive);
      @let userCount = activeUsers.length;
      <p>Active Users: {{ userCount }}</p>
      <ul>
        @for (user of activeUsers; track user.id) {
          <li>{{ user.name }}</li>
        }
      </ul>

      <!-- @let in conditional blocks -->
      @let isEligible = user.age >= 18 && user.hasAccount;
      @if (isEligible) {
        <button>Proceed to Checkout</button>
      } @else {
        <p>You must be 18+ with an account</p>
      }

      <!-- @let with async data -->
      @let currentData = dataStream | async;
      @if (currentData) {
        @let processedData = transformData(currentData);
        <app-data-display [data]="processedData"></app-data-display>
      }

      <!-- Multiple @let declarations -->
      @let totalItems = cart.items.length;
      @let subtotal = cart.items.reduce((sum, item) => sum + item.price, 0);
      @let shipping = subtotal > 100 ? 0 : 10;
      @let total = subtotal + shipping;

      <div class="cart-summary">
        <p>Items: {{ totalItems }}</p>
        <p>Subtotal: ${{ subtotal }}</p>
        <p>Shipping: ${{ shipping }}</p>
        <p><strong>Total: ${{ total }}</strong></p>
      </div>

      <!-- @let scope is limited to the block -->
      @if (showDetails) {
        @let detailInfo = getDetailedInfo();
        <div>{{ detailInfo }}</div>
      }
      <!-- detailInfo is not accessible here -->

      <!-- @let with template reference variables -->
      <input #searchInput>
      @let searchQuery = searchInput.value.toLowerCase().trim();
      @let searchResults = items.filter(item => 
        item.name.toLowerCase().includes(searchQuery)
      );
      <p>Found {{ searchResults.length }} results</p>
    </div>
  `
})
export class TemplateVariablesComponent {
  user = {
    firstName: 'John',
    lastName: 'Doe',
    age: 25,
    hasAccount: true,
    email: 'john@example.com',
    address: {
      street: '123 Main St',
      city: 'New York',
      country: 'USA'
    },
    profile: {
      displayName: 'Johnny'
    }
  };

  product = {
    price: 100,
    discount: 0.2
  };

  users = [
    { id: 1, name: 'Alice', isActive: true },
    { id: 2, name: 'Bob', isActive: false },
    { id: 3, name: 'Charlie', isActive: true }
  ];

  cart = {
    items: [
      { id: 1, name: 'Item 1', price: 50 },
      { id: 2, name: 'Item 2', price: 75 }
    ]
  };

  dataStream: any; // Observable
  showDetails = true;
  items: any[] = [];

  copyToClipboard(text: string): void {
    navigator.clipboard.writeText(text);
  }

  transformData(data: any): any {
    return data;
  }

  getDetailedInfo(): string {
    return 'Detailed information';
  }
}
```

## Migration from Legacy Structural Directives

### Migrating from *ngIf to @if

```typescript
// BEFORE (Angular 16 and earlier)
@Component({
  template: `
    <!-- Simple *ngIf -->
    <div *ngIf="isVisible">Content</div>

    <!-- *ngIf with else -->
    <div *ngIf="hasData; else noData">
      Has data
    </div>
    <ng-template #noData>
      No data
    </ng-template>

    <!-- *ngIf with as syntax -->
    <div *ngIf="user$ | async as user">
      {{ user.name }}
    </div>

    <!-- *ngIf with then/else -->
    <div *ngIf="condition; then thenBlock else elseBlock"></div>
    <ng-template #thenBlock>Then content</ng-template>
    <ng-template #elseBlock>Else content</ng-template>
  `
})

// AFTER (Angular 17+)
@Component({
  template: `
    <!-- Simple @if -->
    @if (isVisible) {
      <div>Content</div>
    }

    <!-- @if with @else -->
    @if (hasData) {
      <div>Has data</div>
    } @else {
      <div>No data</div>
    }

    <!-- @if with @let for async -->
    @let user = user$ | async;
    @if (user) {
      <div>{{ user.name }}</div>
    }

    <!-- @if with @else -->
    @if (condition) {
      <div>Then content</div>
    } @else {
      <div>Else content</div>
    }
  `
})
```

### Migrating from *ngFor to @for

```typescript
// BEFORE (Angular 16 and earlier)
@Component({
  template: `
    <!-- Basic *ngFor -->
    <div *ngFor="let item of items">
      {{ item }}
    </div>

    <!-- *ngFor with trackBy -->
    <div *ngFor="let item of items; trackBy: trackByFn">
      {{ item.name }}
    </div>

    <!-- *ngFor with index -->
    <div *ngFor="let item of items; let i = index">
      {{ i }}. {{ item }}
    </div>

    <!-- *ngFor with multiple locals -->
    <div *ngFor="let item of items; let i = index; let isFirst = first; let isLast = last">
      {{ i }}. {{ item }} (First: {{ isFirst }}, Last: {{ isLast }})
    </div>
  `
})
export class OldComponent {
  items = [/*...*/];

  trackByFn(index: number, item: any): any {
    return item.id;
  }
}

// AFTER (Angular 17+)
@Component({
  template: `
    <!-- Basic @for -->
    @for (item of items; track item) {
      <div>{{ item }}</div>
    }

    <!-- @for with track expression -->
    @for (item of items; track item.id) {
      <div>{{ item.name }}</div>
    }

    <!-- @for with index -->
    @for (item of items; track item; let i = $index) {
      <div>{{ i }}. {{ item }}</div>
    }

    <!-- @for with multiple context variables -->
    @for (item of items; track item; let i = $index; let isFirst = $first; let isLast = $last) {
      <div>{{ i }}. {{ item }} (First: {{ isFirst }}, Last: {{ isLast }})</div>
    }

    <!-- @for with @empty -->
    @for (item of items; track item.id) {
      <div>{{ item.name }}</div>
    } @empty {
      <div>No items</div>
    }
  `
})
export class NewComponent {
  items = [/*...*/];
  // No need for separate trackByFn method
}
```

### Migrating from *ngSwitch to @switch

```typescript
// BEFORE (Angular 16 and earlier)
@Component({
  template: `
    <div [ngSwitch]="status">
      <div *ngSwitchCase="'loading'">Loading...</div>
      <div *ngSwitchCase="'success'">Success!</div>
      <div *ngSwitchCase="'error'">Error occurred</div>
      <div *ngSwitchDefault>Unknown status</div>
    </div>
  `
})

// AFTER (Angular 17+)
@Component({
  template: `
    @switch (status) {
      @case ('loading') {
        <div>Loading...</div>
      }
      @case ('success') {
        <div>Success!</div>
      }
      @case ('error') {
        <div>Error occurred</div>
      }
      @default {
        <div>Unknown status</div>
      }
    }
  `
})
```

## Common Mistakes

### Mistake 1: Forgetting track in @for

❌ **Wrong:**
```typescript
@Component({
  template: `
    <!-- Error: track expression is required -->
    @for (item of items) {
      <div>{{ item }}</div>
    }
  `
})
```

✅ **Correct:**
```typescript
@Component({
  template: `
    @for (item of items; track item.id) {
      <div>{{ item }}</div>
    }
    
    <!-- Or for primitives -->
    @for (item of items; track item) {
      <div>{{ item }}</div>
    }
  `
})
```

### Mistake 2: Using $index for tracking dynamic lists

❌ **Wrong:**
```typescript
@Component({
  template: `
    <!-- Inefficient: causes unnecessary re-renders on reorder -->
    @for (user of users; track $index) {
      <div>{{ user.name }}</div>
    }
  `
})
```

✅ **Correct:**
```typescript
@Component({
  template: `
    <!-- Efficient: tracks by unique ID -->
    @for (user of users; track user.id) {
      <div>{{ user.name }}</div>
    }
  `
})
```

### Mistake 3: Missing @empty with conditional rendering

❌ **Wrong:**
```typescript
@Component({
  template: `
    @if (items.length === 0) {
      <p>No items</p>
    }
    @for (item of items; track item.id) {
      <div>{{ item.name }}</div>
    }
  `
})
```

✅ **Correct:**
```typescript
@Component({
  template: `
    @for (item of items; track item.id) {
      <div>{{ item.name }}</div>
    } @empty {
      <p>No items</p>
    }
  `
})
```

### Mistake 4: Overusing @defer on critical content

❌ **Wrong:**
```typescript
@Component({
  template: `
    <!-- Critical content shouldn't be deferred -->
    @defer (on idle) {
      <h1>{{ pageTitle }}</h1>
      <p>{{ criticalMessage }}</p>
    }
  `
})
```

✅ **Correct:**
```typescript
@Component({
  template: `
    <!-- Critical content renders immediately -->
    <h1>{{ pageTitle }}</h1>
    <p>{{ criticalMessage }}</p>
    
    <!-- Only defer non-critical content -->
    @defer (on viewport) {
      <app-comments-section></app-comments-section>
    }
  `
})
```

### Mistake 5: Missing minimum time in @loading blocks

❌ **Wrong:**
```typescript
@Component({
  template: `
    @defer (on viewport) {
      <app-content></app-content>
    } @loading {
      <div class="spinner"></div>
    }
  `
})
// Loading spinner might flash briefly causing poor UX
```

✅ **Correct:**
```typescript
@Component({
  template: `
    @defer (on viewport) {
      <app-content></app-content>
    } @loading (minimum 500ms) {
      <div class="spinner"></div>
    }
  `
})
```

## Best Practices

### 1. Always Use track in @for

✅ **Good:**
```typescript
@Component({
  template: `
    <!-- Track by unique ID -->
    @for (item of items; track item.id) {
      <div>{{ item.name }}</div>
    }
    
    <!-- Track by identity for primitives -->
    @for (tag of tags; track tag) {
      <span>{{ tag }}</span>
    }
  `
})
```

### 2. Prefer @if over [hidden]

✅ **Good:**
```typescript
@Component({
  template: `
    <!-- Doesn't render in DOM when false -->
    @if (shouldShow) {
      <app-expensive-component></app-expensive-component>
    }
  `
})
```

❌ **Avoid:**
```typescript
@Component({
  template: `
    <!-- Always in DOM, just hidden -->
    <app-expensive-component [hidden]="!shouldShow"></app-expensive-component>
  `
})
```

### 3. Use @empty for Better UX

✅ **Good:**
```typescript
@Component({
  template: `
    @for (item of items; track item.id) {
      <div>{{ item.name }}</div>
    } @empty {
      <div class="empty-state">
        <p>No items to display</p>
        <button (click)="addItem()">Add First Item</button>
      </div>
    }
  `
})
```

### 4. Leverage @defer for Performance

✅ **Good:**
```typescript
@Component({
  template: `
    <!-- Defer below-the-fold content -->
    @defer (on viewport; prefetch on idle) {
      <app-comments></app-comments>
    } @loading (minimum 500ms) {
      <div class="skeleton-loader"></div>
    } @placeholder {
      <div class="comments-placeholder"></div>
    }
  `
})
```

### 5. Use @let for Complex Expressions

✅ **Good:**
```typescript
@Component({
  template: `
    @let filteredItems = items.filter(i => i.isActive);
    @let total = filteredItems.length;
    
    <p>Showing {{ total }} active items</p>
    @for (item of filteredItems; track item.id) {
      <div>{{ item.name }}</div>
    }
  `
})
```

### 6. Prefer @switch for Multiple Conditions

✅ **Good:**
```typescript
@Component({
  template: `
    @switch (userRole) {
      @case ('admin') { <app-admin-panel></app-admin-panel> }
      @case ('editor') { <app-editor-panel></app-editor-panel> }
      @case ('viewer') { <app-viewer-panel></app-viewer-panel> }
      @default { <app-guest-panel></app-guest-panel> }
    }
  `
})
```

❌ **Avoid:**
```typescript
@Component({
  template: `
    @if (userRole === 'admin') {
      <app-admin-panel></app-admin-panel>
    } @else if (userRole === 'editor') {
      <app-editor-panel></app-editor-panel>
    } @else if (userRole === 'viewer') {
      <app-viewer-panel></app-viewer-panel>
    } @else {
      <app-guest-panel></app-guest-panel>
    }
  `
})
```

### 7. Use Appropriate @defer Triggers

✅ **Good:**
```typescript
@Component({
  template: `
    <!-- Viewport for below-the-fold content -->
    @defer (on viewport) {
      <app-footer></app-footer>
    }
    
    <!-- Interaction for modals -->
    @defer (on interaction) {
      <app-modal></app-modal>
    } @placeholder {
      <button>Open Modal</button>
    }
    
    <!-- Idle for non-critical analytics -->
    @defer (on idle) {
      <app-analytics></app-analytics>
    }
  `
})
```

## Interview Questions

**Q1: What are the advantages of the new control flow syntax (@if, @for, @switch) over traditional structural directives?**

**A:** The new control flow syntax offers several advantages:

**1. Better Performance:**
- Processed directly by the compiler, not at runtime
- More efficient change detection
- Smaller bundle sizes
- Better tree-shaking

**2. Improved Type Safety:**
- Better type narrowing in @if blocks
- TypeScript understands the conditions better
- Reduced need for type assertions

**3. Mandatory track Expressions:**
- @for requires track, preventing common performance mistakes
- No need for separate trackBy functions
- Clearer tracking intent

**4. More Intuitive Syntax:**
- Resembles JavaScript control flow
- No asterisk prefix confusion
- Cleaner nested structures
- Easier for developers coming from other frameworks

**5. Built-in @empty:**
- @for includes @empty block for empty arrays
- No need for separate *ngIf checks
- Better UX patterns

**6. Native @defer Support:**
- Built-in lazy loading
- Multiple trigger strategies
- Prefetch capabilities
- Loading/placeholder/error states

```typescript
// OLD WAY
<div *ngIf="items.length > 0; else noItems">
  <div *ngFor="let item of items; trackBy: trackByFn">
    {{ item.name }}
  </div>
</div>
<ng-template #noItems>No items</ng-template>

// NEW WAY (cleaner, more performant)
@for (item of items; track item.id) {
  <div>{{ item.name }}</div>
} @empty {
  <div>No items</div>
}
```

**Q2: Why is the track expression mandatory in @for, and how do you choose the right tracking strategy?**

**A:** The track expression is mandatory because it directly impacts Angular's change detection performance and prevents a common source of bugs.

**Why Mandatory:**
1. **Performance**: Angular uses track to identify which items changed
2. **Prevents Mistakes**: Developers can't forget to specify tracking
3. **Explicit Intent**: Makes performance considerations obvious
4. **Optimization**: Compiler can optimize based on tracking strategy

**Choosing Track Strategy:**

**1. Track by Unique ID (Best):**
```typescript
@for (user of users; track user.id) {
  <div>{{ user.name }}</div>
}
```
Use when: Items have unique identifiers
Benefits: Most efficient, handles reordering well

**2. Track by Identity (Good for Primitives):**
```typescript
@for (tag of tags; track tag) {
  <span>{{ tag }}</span>
}
```
Use when: Items are primitives (strings, numbers)
Benefits: Simple, works for immutable data

**3. Track by Index (Use Rarely):**
```typescript
@for (item of items; track $index) {
  <div>{{ item }}</div>
}
```
Use when: Array never reorders, items have no ID
Warning: Poor performance on reorder/insert/delete

**4. Track by Composite Key:**
```typescript
@for (order of orders; track order.customerId + '-' + order.orderId) {
  <div>Order #{{ order.orderId }}</div>
}
```
Use when: No single unique identifier exists

**Performance Impact Example:**
```typescript
// ❌ Bad: track by index with reordering
users = [{ id: 1, name: 'Alice' }, { id: 2, name: 'Bob' }];
@for (user of users; track $index) { ... }
// Shuffle array -> All DOM nodes recreated

// ✅ Good: track by ID
@for (user of users; track user.id) { ... }
// Shuffle array -> DOM nodes simply reordered
```

**Q3: How does @defer work, and what are its different trigger strategies?**

**A:** @defer enables lazy loading of components and templates based on various triggers, improving initial load performance.

**How @defer Works:**
1. **Placeholder Phase**: Shows optional placeholder content
2. **Trigger**: Waits for specified condition
3. **Loading Phase**: Optional loading state while fetching
4. **Content Phase**: Renders deferred content
5. **Error Phase**: Optional error state if loading fails

**Trigger Strategies:**

**1. idle (default):**
```typescript
@defer {
  <app-heavy-component></app-heavy-component>
}
```
Loads when browser is idle (requestIdleCallback)

**2. viewport:**
```typescript
@defer (on viewport) {
  <app-below-fold></app-below-fold>
}
```
Loads when element enters viewport (IntersectionObserver)

**3. interaction:**
```typescript
@defer (on interaction) {
  <app-modal></app-modal>
} @placeholder {
  <button>Open Modal</button>
}
```
Loads on click, keydown, or other interactions

**4. hover:**
```typescript
@defer (on hover) {
  <app-tooltip></app-tooltip>
}
```
Loads on mouseenter or focusin

**5. immediate:**
```typescript
@defer (on immediate) {
  <app-async-content></app-async-content>
}
```
Loads immediately but asynchronously (non-blocking)

**6. timer:**
```typescript
@defer (on timer(5000ms)) {
  <app-banner></app-banner>
}
```
Loads after specified time

**7. when (conditional):**
```typescript
@defer (when dataReady && userLoggedIn) {
  <app-dashboard></app-dashboard>
}
```
Loads when expression becomes truthy

**Prefetch Strategies:**
```typescript
// Prefetch on idle, load on interaction
@defer (on interaction; prefetch on idle) {
  <app-video-player></app-video-player>
}

// Prefetch ahead of trigger
@defer (on viewport; prefetch on hover) {
  <app-comments></app-comments>
}
```

**Complete Example:**
```typescript
@defer (on viewport; prefetch on idle) {
  <app-data-table [data]="data"></app-data-table>
} @loading (minimum 500ms) {
  <div class="spinner">Loading...</div>
} @placeholder (minimum 500ms) {
  <div class="skeleton"></div>
} @error {
  <div>Failed to load table</div>
}
```

**Q4: What is @let and how does it improve template readability and performance?**

**A:** @let creates template-scoped variables that store expression results, improving both readability and performance.

**Benefits:**

**1. Avoid Repeated Calculations:**
```typescript
// ❌ Without @let - calculated 3 times
<p>Price: ${{ product.price * (1 - discount) }}</p>
<p>Tax: ${{ (product.price * (1 - discount)) * 0.1 }}</p>
<p>Total: ${{ (product.price * (1 - discount)) * 1.1 }}</p>

// ✅ With @let - calculated once
@let finalPrice = product.price * (1 - discount);
<p>Price: ${{ finalPrice }}</p>
<p>Tax: ${{ finalPrice * 0.1 }}</p>
<p>Total: ${{ finalPrice * 1.1 }}</p>
```

**2. Simplify Complex Expressions:**
```typescript
// ❌ Hard to read
@if ((user?.profile?.settings?.notifications?.email ?? false) && (user?.subscriptions?.includes('premium') ?? false)) {
  <app-premium-notifications></app-premium-notifications>
}

// ✅ Clear with @let
@let emailEnabled = user?.profile?.settings?.notifications?.email ?? false;
@let isPremium = user?.subscriptions?.includes('premium') ?? false;
@if (emailEnabled && isPremium) {
  <app-premium-notifications></app-premium-notifications>
}
```

**3. Store Async Results:**
```typescript
@let currentUser = userStream | async;
@if (currentUser) {
  <p>Welcome, {{ currentUser.name }}!</p>
  <app-user-menu [user]="currentUser"></app-user-menu>
  <app-notifications [userId]="currentUser.id"></app-notifications>
}
// Async pipe runs once, value reused 3 times
```

**4. Improve Type Narrowing:**
```typescript
@let data = apiResponse?.data;
@if (data) {
  // TypeScript knows data is not null/undefined here
  <p>{{ data.title }}</p>
  <p>{{ data.description }}</p>
}
```

**5. Scope Management:**
```typescript
@if (showSection) {
  @let sectionData = getComplexData();
  <app-section [data]="sectionData"></app-section>
  // sectionData only exists within this @if block
}
// sectionData is not accessible here
```

**Performance Impact:**
- Expressions evaluated once per change detection cycle
- Results cached in template variable
- Reduces function calls and computations
- Particularly beneficial with expensive operations

**Limitations:**
- Block-scoped (not accessible outside its block)
- Cannot be reassigned
- Must be declared before use

**Q5: How do you migrate from *ngIf/*ngFor/*ngSwitch to the new control flow syntax?**

**A:** Migration follows these patterns:

**Automated Migration:**
```bash
# Angular CLI provides automatic migration
ng generate @angular/core:control-flow
```

**Manual Migration Patterns:**

**1. *ngIf to @if:**
```typescript
// Before
<div *ngIf="condition">Content</div>
<div *ngIf="user$ | async as user">{{ user.name }}</div>
<div *ngIf="condition; else elseBlock">Then</div>
<ng-template #elseBlock>Else</ng-template>

// After
@if (condition) {
  <div>Content</div>
}

@let user = user$ | async;
@if (user) {
  <div>{{ user.name }}</div>
}

@if (condition) {
  <div>Then</div>
} @else {
  <div>Else</div>
}
```

**2. *ngFor to @for:**
```typescript
// Before
<div *ngFor="let item of items; trackBy: trackByFn">
  {{ item.name }}
</div>

<div *ngFor="let item of items; let i = index; let isFirst = first">
  {{ i }}. {{ item.name }}
</div>

// After
@for (item of items; track item.id) {
  <div>{{ item.name }}</div>
}

@for (item of items; track item.id; let i = $index; let isFirst = $first) {
  <div>{{ i }}. {{ item.name }}</div>
}
```

**3. *ngSwitch to @switch:**
```typescript
// Before
<div [ngSwitch]="status">
  <div *ngSwitchCase="'loading'">Loading...</div>
  <div *ngSwitchCase="'success'">Success!</div>
  <div *ngSwitchDefault>Default</div>
</div>

// After
@switch (status) {
  @case ('loading') {
    <div>Loading...</div>
  }
  @case ('success') {
    <div>Success!</div>
  }
  @default {
    <div>Default</div>
  }
}
```

**4. Complex Nested Structures:**
```typescript
// Before
<div *ngIf="items.length > 0; else noItems">
  <div *ngFor="let item of items; trackBy: trackByFn">
    <span *ngIf="item.isActive">{{ item.name }}</span>
  </div>
</div>
<ng-template #noItems>No items</ng-template>

// After
@for (item of items; track item.id) {
  @if (item.isActive) {
    <div>
      <span>{{ item.name }}</span>
    </div>
  }
} @empty {
  <div>No items</div>
}
```

**Migration Checklist:**
1. ✅ Run automated migration command
2. ✅ Replace trackBy functions with inline track expressions
3. ✅ Convert ng-template #refs to @else/@empty blocks
4. ✅ Update *ngIf with 'as' syntax to @let
5. ✅ Test all conditional logic thoroughly
6. ✅ Update unit tests (TestBed queries may change)
7. ✅ Consider adding @defer for performance wins
8. ✅ Remove unused trackBy methods from components

**Compatibility:**
- Both syntaxes work together during migration
- No breaking changes to component logic
- Can migrate incrementally per component
- Template syntax only changes

**Q6: What are the performance implications of @defer, and when should you use it?**

**A:** @defer significantly impacts performance by splitting bundles and delaying non-critical code loading.

**Performance Benefits:**

**1. Reduced Initial Bundle Size:**
```typescript
// Without @defer - all loaded upfront
<app-comments></app-comments>          // 50KB
<app-recommendations></app-recommendations>  // 80KB
<app-analytics></app-analytics>        // 30KB
// Total initial: 160KB

// With @defer - loaded on demand
@defer (on viewport) {
  <app-comments></app-comments>
}
@defer (on viewport) {
  <app-recommendations></app-recommendations>
}
@defer (on idle) {
  <app-analytics></app-analytics>
}
// Initial: ~0KB, loaded progressively
```

**2. Faster Initial Render:**
- Less JavaScript to parse and execute
- Faster Time to Interactive (TTI)
- Better Core Web Vitals scores
- Improved perceived performance

**3. Better Resource Utilization:**
- Code loaded when needed
- Reduced memory footprint initially
- Better cache utilization
- Network requests spread over time

**When to Use @defer:**

**✅ Use for:**

1. **Below-the-fold content:**
```typescript
@defer (on viewport) {
  <app-footer></app-footer>
  <app-comments-section></app-comments-section>
}
```

2. **Modal dialogs and overlays:**
```typescript
@defer (on interaction) {
  <app-settings-modal></app-settings-modal>
} @placeholder {
  <button>Open Settings</button>
}
```

3. **Heavy third-party widgets:**
```typescript
@defer (on idle) {
  <app-chat-widget></app-chat-widget>
  <app-analytics-tracker></app-analytics-tracker>
}
```

4. **Admin/rare features:**
```typescript
@defer (when user.isAdmin) {
  <app-admin-dashboard></app-admin-dashboard>
}
```

5. **Non-critical visualizations:**
```typescript
@defer (on viewport; prefetch on idle) {
  <app-data-charts [data]="chartData"></app-data-charts>
} @loading (minimum 500ms) {
  <app-chart-skeleton></app-chart-skeleton>
}
```

**❌ Don't use for:**

1. **Critical above-the-fold content:**
```typescript
// ❌ Bad - delays critical content
@defer (on idle) {
  <h1>{{ pageTitle }}</h1>
  <app-main-content></app-main-content>
}

// ✅ Good - render immediately
<h1>{{ pageTitle }}</h1>
<app-main-content></app-main-content>
```

2. **Small components:**
```typescript
// ❌ Bad - overhead not worth it
@defer (on viewport) {
  <span>{{ smallText }}</span>
}
```

3. **SEO-critical content:**
```typescript
// ❌ Bad - search engines may not see it
@defer (on interaction) {
  <article>{{ seoContent }}</article>
}
```

**Prefetch Strategy:**
```typescript
// Balance between performance and UX
@defer (on interaction; prefetch on hover) {
  <app-modal></app-modal>
}
// Modal code prefetched on hover, loaded on click
// Result: instant feel with delayed initial load
```

**Measuring Impact:**
```typescript
// Before @defer
Initial bundle: 500KB
Time to Interactive: 3.2s
First Contentful Paint: 1.8s

// After strategic @defer usage
Initial bundle: 250KB (-50%)
Time to Interactive: 1.6s (-50%)
First Contentful Paint: 1.2s (-33%)
```

**Best Practice Pattern:**
```typescript
<main>
  <!-- Critical: immediate render -->
  <app-header></app-header>
  <app-hero-section></app-hero-section>
  
  <!-- Important: prefetch, load on viewport -->
  @defer (on viewport; prefetch on idle) {
    <app-features></app-features>
  } @placeholder (minimum 200ms) {
    <app-features-skeleton></app-features-skeleton>
  }
  
  <!-- Secondary: load on viewport -->
  @defer (on viewport) {
    <app-testimonials></app-testimonials>
    <app-pricing></app-pricing>
  }
  
  <!-- Tertiary: load when idle -->
  @defer (on idle) {
    <app-footer></app-footer>
    <app-analytics></app-analytics>
  }
</main>
```

## Key Takeaways

- **@if** replaces *ngIf with cleaner syntax and better type narrowing
- **@for** requires mandatory track expressions for optimal performance
- **@switch** provides intuitive multi-condition branching
- **@defer** enables strategic lazy loading with multiple trigger strategies
- **@let** creates template-scoped variables for reusable expressions
- New control flow syntax is processed by the compiler for better performance
- Track expressions should use unique IDs when available, not $index
- @empty blocks provide built-in empty state handling
- @defer should be used for non-critical, below-the-fold, or interactive content
- Prefetch strategies balance performance with user experience
- Migration from legacy structural directives is straightforward and can be automated
- The new syntax aligns Angular templates with standard JavaScript control flow

## Resources

- [Angular Control Flow - Official Docs](https://angular.io/guide/control-flow)
- [@if Documentation](https://angular.io/api/core/@if)
- [@for Documentation](https://angular.io/api/core/@for)
- [@switch Documentation](https://angular.io/api/core/@switch)
- [@defer Documentation](https://angular.io/api/core/@defer)
- [Control Flow Migration Guide](https://angular.io/guide/control-flow#migration)
- [Angular 17 Control Flow - Blog Post](https://blog.angular.io/introducing-angular-v17-4d7033312e4b)
- [Performance Optimization with @defer](https://angular.io/guide/defer)
- [Track Function Best Practices](https://angular.io/guide/control-flow#track)
- [Angular Change Detection](https://angular.io/guide/change-detection)
