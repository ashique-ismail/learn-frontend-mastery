# Spectator Testing Library for Angular

## Table of Contents
- [Introduction](#introduction)
- [Installation and Setup](#installation-and-setup)
- [Basic Component Testing](#basic-component-testing)
- [Testing Services](#testing-services)
- [Testing Directives](#testing-directives)
- [Testing Pipes](#testing-pipes)
- [HTTP Testing with Spectator](#http-testing-with-spectator)
- [Advanced Testing Patterns](#advanced-testing-patterns)
- [Testing Routing](#testing-routing)
- [Custom Matchers and Helpers](#custom-matchers-and-helpers)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Spectator is a powerful testing library from ngneat that dramatically simplifies Angular testing by reducing boilerplate and providing intuitive APIs. It wraps Angular's TestBed with a cleaner, more expressive interface, making tests easier to write, read, and maintain.

Traditional Angular testing requires verbose TestBed configuration, manual change detection, and complex DOM queries. Spectator streamlines this process with factories, automatic change detection, jQuery-like selectors, and built-in mocking utilities.

## Installation and Setup

### Installation

```bash
npm install @ngneat/spectator --save-dev
# or
yarn add @ngneat/spectator --dev
```

### Basic Configuration

```typescript
// spectator.config.ts (optional global configuration)
import { SpectatorOptions } from '@ngneat/spectator';

export const defaultSpectatorOptions: SpectatorOptions<any> = {
  detectChanges: true,
  shallow: false
};
```

### Importing Spectator

```typescript
import { Spectator, createComponentFactory } from '@ngneat/spectator';
import { createServiceFactory, SpectatorService } from '@ngneat/spectator';
import { createDirectiveFactory, SpectatorDirective } from '@ngneat/spectator';
import { createHttpFactory, SpectatorHttp } from '@ngneat/spectator';
```

## Basic Component Testing

### Simple Component

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `
    <div class="counter">
      <h2>{{ count }}</h2>
      <button class="increment" (click)="increment()">+</button>
      <button class="decrement" (click)="decrement()">-</button>
      <button class="reset" (click)="reset()">Reset</button>
    </div>
  `
})
export class CounterComponent {
  count = 0;

  increment(): void {
    this.count++;
  }

  decrement(): void {
    this.count--;
  }

  reset(): void {
    this.count = 0;
  }
}
```

### Testing with Spectator

```typescript
import { Spectator, createComponentFactory } from '@ngneat/spectator';
import { CounterComponent } from './counter.component';

describe('CounterComponent', () => {
  let spectator: Spectator<CounterComponent>;
  const createComponent = createComponentFactory(CounterComponent);

  beforeEach(() => {
    spectator = createComponent();
  });

  it('should create', () => {
    expect(spectator.component).toBeTruthy();
  });

  it('should initialize count to 0', () => {
    expect(spectator.component.count).toBe(0);
    expect(spectator.query('h2')).toHaveText('0');
  });

  it('should increment count', () => {
    spectator.click('.increment');
    expect(spectator.component.count).toBe(1);
    expect(spectator.query('h2')).toHaveText('1');
  });

  it('should decrement count', () => {
    spectator.click('.decrement');
    expect(spectator.component.count).toBe(-1);
    expect(spectator.query('h2')).toHaveText('-1');
  });

  it('should reset count', () => {
    spectator.component.count = 5;
    spectator.detectChanges();
    spectator.click('.reset');
    expect(spectator.component.count).toBe(0);
  });
});
```

### Component with Inputs and Outputs

```typescript
@Component({
  selector: 'app-user-card',
  template: `
    <div class="user-card">
      <h3>{{ user.name }}</h3>
      <p>{{ user.email }}</p>
      <button class="delete" (click)="onDelete()">Delete</button>
    </div>
  `
})
export class UserCardComponent {
  @Input() user!: { name: string; email: string };
  @Output() delete = new EventEmitter<void>();

  onDelete(): void {
    this.delete.emit();
  }
}

// Test
describe('UserCardComponent', () => {
  let spectator: Spectator<UserCardComponent>;
  const createComponent = createComponentFactory(UserCardComponent);

  it('should display user information', () => {
    spectator = createComponent({
      props: {
        user: { name: 'John Doe', email: 'john@example.com' }
      }
    });

    expect(spectator.query('h3')).toHaveText('John Doe');
    expect(spectator.query('p')).toHaveText('john@example.com');
  });

  it('should emit delete event when button clicked', () => {
    spectator = createComponent({
      props: {
        user: { name: 'John Doe', email: 'john@example.com' }
      }
    });

    let emitted = false;
    spectator.output('delete').subscribe(() => emitted = true);

    spectator.click('.delete');
    expect(emitted).toBe(true);
  });

  it('should emit delete event using spy', () => {
    spectator = createComponent({
      props: {
        user: { name: 'John Doe', email: 'john@example.com' }
      }
    });

    const deleteSpy = spyOn(spectator.component.delete, 'emit');
    spectator.click('.delete');
    expect(deleteSpy).toHaveBeenCalled();
  });
});
```

### Testing with Dependencies

```typescript
@Injectable({ providedIn: 'root' })
export class UserService {
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>('/api/users');
  }
}

@Component({
  selector: 'app-user-list',
  template: `
    <div class="user-list">
      <div class="user" *ngFor="let user of users">
        {{ user.name }}
      </div>
    </div>
  `
})
export class UserListComponent implements OnInit {
  users: User[] = [];

  constructor(private userService: UserService) {}

  ngOnInit(): void {
    this.userService.getUsers().subscribe(users => {
      this.users = users;
    });
  }
}

// Test
describe('UserListComponent', () => {
  let spectator: Spectator<UserListComponent>;
  const createComponent = createComponentFactory({
    component: UserListComponent,
    mocks: [UserService]
  });

  beforeEach(() => {
    spectator = createComponent();
  });

  it('should load users on init', () => {
    const mockUsers = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Jane' }
    ];

    const userService = spectator.inject(UserService);
    userService.getUsers.and.returnValue(of(mockUsers));

    spectator.component.ngOnInit();

    expect(spectator.component.users).toEqual(mockUsers);
    expect(spectator.queryAll('.user')).toHaveLength(2);
  });
});
```

## Testing Services

### Basic Service Testing

```typescript
@Injectable({ providedIn: 'root' })
export class CalculatorService {
  add(a: number, b: number): number {
    return a + b;
  }

  subtract(a: number, b: number): number {
    return a - b;
  }

  multiply(a: number, b: number): number {
    return a * b;
  }
}

// Test
import { SpectatorService, createServiceFactory } from '@ngneat/spectator';

describe('CalculatorService', () => {
  let spectator: SpectatorService<CalculatorService>;
  const createService = createServiceFactory(CalculatorService);

  beforeEach(() => {
    spectator = createService();
  });

  it('should add numbers', () => {
    expect(spectator.service.add(2, 3)).toBe(5);
  });

  it('should subtract numbers', () => {
    expect(spectator.service.subtract(5, 3)).toBe(2);
  });

  it('should multiply numbers', () => {
    expect(spectator.service.multiply(4, 3)).toBe(12);
  });
});
```

### Service with Dependencies

```typescript
@Injectable({ providedIn: 'root' })
export class AuthService {
  constructor(private http: HttpClient) {}

  login(username: string, password: string): Observable<{ token: string }> {
    return this.http.post<{ token: string }>('/api/login', { username, password });
  }

  logout(): void {
    localStorage.removeItem('token');
  }
}

// Test
import { createServiceFactory, SpectatorService } from '@ngneat/spectator';
import { HttpClientTestingModule } from '@angular/common/http/testing';

describe('AuthService', () => {
  let spectator: SpectatorService<AuthService>;
  const createService = createServiceFactory({
    service: AuthService,
    imports: [HttpClientTestingModule]
  });

  beforeEach(() => {
    spectator = createService();
  });

  it('should login', () => {
    const http = spectator.inject(HttpClient);
    spyOn(http, 'post').and.returnValue(of({ token: 'test-token' }));

    spectator.service.login('user', 'pass').subscribe(result => {
      expect(result.token).toBe('test-token');
    });

    expect(http.post).toHaveBeenCalledWith('/api/login', {
      username: 'user',
      password: 'pass'
    });
  });

  it('should remove token on logout', () => {
    spyOn(localStorage, 'removeItem');
    spectator.service.logout();
    expect(localStorage.removeItem).toHaveBeenCalledWith('token');
  });
});
```

### Testing with Mock Services

```typescript
describe('UserListComponent', () => {
  let spectator: Spectator<UserListComponent>;
  const createComponent = createComponentFactory({
    component: UserListComponent,
    providers: [
      {
        provide: UserService,
        useValue: {
          getUsers: () => of([
            { id: 1, name: 'John' },
            { id: 2, name: 'Jane' }
          ])
        }
      }
    ]
  });

  beforeEach(() => {
    spectator = createComponent();
  });

  it('should use mocked service', () => {
    expect(spectator.component.users.length).toBe(2);
  });
});
```

## Testing Directives

### Simple Directive

```typescript
@Directive({
  selector: '[appHighlight]'
})
export class HighlightDirective {
  @Input() appHighlight = 'yellow';

  constructor(private el: ElementRef) {}

  @HostListener('mouseenter') onMouseEnter() {
    this.highlight(this.appHighlight);
  }

  @HostListener('mouseleave') onMouseLeave() {
    this.highlight('');
  }

  private highlight(color: string) {
    this.el.nativeElement.style.backgroundColor = color;
  }
}

// Test
import { SpectatorDirective, createDirectiveFactory } from '@ngneat/spectator';

describe('HighlightDirective', () => {
  let spectator: SpectatorDirective<HighlightDirective>;
  const createDirective = createDirectiveFactory(HighlightDirective);

  it('should change background color on mouse enter', () => {
    spectator = createDirective(
      `<div appHighlight="red">Test</div>`
    );

    spectator.dispatchMouseEvent(spectator.element, 'mouseenter');
    expect(spectator.element).toHaveStyle({ backgroundColor: 'red' });

    spectator.dispatchMouseEvent(spectator.element, 'mouseleave');
    expect(spectator.element).toHaveStyle({ backgroundColor: '' });
  });

  it('should use default color', () => {
    spectator = createDirective(
      `<div appHighlight>Test</div>`
    );

    spectator.dispatchMouseEvent(spectator.element, 'mouseenter');
    expect(spectator.element).toHaveStyle({ backgroundColor: 'yellow' });
  });
});
```

### Structural Directive

```typescript
@Directive({
  selector: '[appUnless]'
})
export class UnlessDirective {
  private hasView = false;

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef
  ) {}

  @Input() set appUnless(condition: boolean) {
    if (!condition && !this.hasView) {
      this.viewContainer.createEmbeddedView(this.templateRef);
      this.hasView = true;
    } else if (condition && this.hasView) {
      this.viewContainer.clear();
      this.hasView = false;
    }
  }
}

// Test
describe('UnlessDirective', () => {
  let spectator: SpectatorDirective<UnlessDirective>;
  const createDirective = createDirectiveFactory(UnlessDirective);

  it('should show content when condition is false', () => {
    spectator = createDirective(
      `<div *appUnless="false">Content</div>`
    );

    expect(spectator.element).toHaveText('Content');
  });

  it('should hide content when condition is true', () => {
    spectator = createDirective(
      `<div *appUnless="true">Content</div>`
    );

    expect(spectator.element).not.toHaveText('Content');
  });
});
```

## Testing Pipes

### Basic Pipe

```typescript
@Pipe({ name: 'truncate' })
export class TruncatePipe implements PipeTransform {
  transform(value: string, limit: number = 10, ellipsis: string = '...'): string {
    if (!value) return '';
    if (value.length <= limit) return value;
    return value.substring(0, limit) + ellipsis;
  }
}

// Test
import { SpectatorPipe, createPipeFactory } from '@ngneat/spectator';

describe('TruncatePipe', () => {
  let spectator: SpectatorPipe<TruncatePipe>;
  const createPipe = createPipeFactory(TruncatePipe);

  it('should truncate long text', () => {
    spectator = createPipe(`{{ 'Hello World' | truncate:5 }}`);
    expect(spectator.element).toHaveText('Hello...');
  });

  it('should not truncate short text', () => {
    spectator = createPipe(`{{ 'Hi' | truncate:10 }}`);
    expect(spectator.element).toHaveText('Hi');
  });

  it('should use custom ellipsis', () => {
    spectator = createPipe(`{{ 'Hello World' | truncate:5:'***' }}`);
    expect(spectator.element).toHaveText('Hello***');
  });

  it('should handle empty string', () => {
    spectator = createPipe(`{{ '' | truncate }}`);
    expect(spectator.element).toHaveText('');
  });
});
```

## HTTP Testing with Spectator

### Service with HTTP

```typescript
@Injectable({ providedIn: 'root' })
export class TodoService {
  private apiUrl = '/api/todos';

  constructor(private http: HttpClient) {}

  getTodos(): Observable<Todo[]> {
    return this.http.get<Todo[]>(this.apiUrl);
  }

  createTodo(todo: Partial<Todo>): Observable<Todo> {
    return this.http.post<Todo>(this.apiUrl, todo);
  }

  updateTodo(id: number, todo: Partial<Todo>): Observable<Todo> {
    return this.http.put<Todo>(`${this.apiUrl}/${id}`, todo);
  }

  deleteTodo(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`);
  }
}

// Test
import { SpectatorHttp, createHttpFactory } from '@ngneat/spectator';

describe('TodoService', () => {
  let spectator: SpectatorHttp<TodoService>;
  const createHttp = createHttpFactory(TodoService);

  beforeEach(() => {
    spectator = createHttp();
  });

  it('should get todos', () => {
    const mockTodos = [
      { id: 1, title: 'Todo 1', completed: false },
      { id: 2, title: 'Todo 2', completed: true }
    ];

    spectator.service.getTodos().subscribe(todos => {
      expect(todos).toEqual(mockTodos);
    });

    spectator.expectOne('/api/todos', 'GET').flush(mockTodos);
  });

  it('should create todo', () => {
    const newTodo = { title: 'New Todo', completed: false };
    const createdTodo = { id: 3, ...newTodo };

    spectator.service.createTodo(newTodo).subscribe(todo => {
      expect(todo).toEqual(createdTodo);
    });

    const req = spectator.expectOne('/api/todos', 'POST');
    expect(req.request.body).toEqual(newTodo);
    req.flush(createdTodo);
  });

  it('should update todo', () => {
    const updates = { completed: true };
    const updatedTodo = { id: 1, title: 'Todo 1', completed: true };

    spectator.service.updateTodo(1, updates).subscribe(todo => {
      expect(todo).toEqual(updatedTodo);
    });

    const req = spectator.expectOne('/api/todos/1', 'PUT');
    expect(req.request.body).toEqual(updates);
    req.flush(updatedTodo);
  });

  it('should delete todo', () => {
    spectator.service.deleteTodo(1).subscribe();

    spectator.expectOne('/api/todos/1', 'DELETE').flush(null);
  });

  it('should handle 404 error', () => {
    spectator.service.getTodos().subscribe(
      () => fail('should have failed'),
      error => {
        expect(error.status).toBe(404);
      }
    );

    spectator.expectOne('/api/todos', 'GET').flush('Not found', {
      status: 404,
      statusText: 'Not Found'
    });
  });
});
```

## Advanced Testing Patterns

### Testing with Reactive Forms

```typescript
@Component({
  selector: 'app-login',
  template: `
    <form [formGroup]="loginForm" (ngSubmit)="onSubmit()">
      <input formControlName="username" placeholder="Username">
      <input formControlName="password" type="password" placeholder="Password">
      <button type="submit" [disabled]="loginForm.invalid">Login</button>
      <div class="error" *ngIf="loginForm.get('username')?.hasError('required')">
        Username is required
      </div>
    </form>
  `
})
export class LoginComponent {
  loginForm = this.fb.group({
    username: ['', Validators.required],
    password: ['', [Validators.required, Validators.minLength(6)]]
  });

  constructor(private fb: FormBuilder) {}

  onSubmit(): void {
    if (this.loginForm.valid) {
      console.log(this.loginForm.value);
    }
  }
}

// Test
describe('LoginComponent', () => {
  let spectator: Spectator<LoginComponent>;
  const createComponent = createComponentFactory({
    component: LoginComponent,
    imports: [ReactiveFormsModule]
  });

  beforeEach(() => {
    spectator = createComponent();
  });

  it('should create form with two controls', () => {
    expect(spectator.component.loginForm.contains('username')).toBe(true);
    expect(spectator.component.loginForm.contains('password')).toBe(true);
  });

  it('should make username required', () => {
    const control = spectator.component.loginForm.get('username');
    control?.setValue('');
    expect(control?.hasError('required')).toBe(true);
  });

  it('should disable submit button when form is invalid', () => {
    expect(spectator.query('button')).toBeDisabled();
  });

  it('should enable submit button when form is valid', () => {
    spectator.typeInElement('testuser', 'input[formControlName="username"]');
    spectator.typeInElement('password123', 'input[formControlName="password"]');

    expect(spectator.query('button')).not.toBeDisabled();
  });

  it('should show error message for required username', () => {
    const usernameInput = spectator.query('input[formControlName="username"]');
    spectator.focus(usernameInput);
    spectator.blur(usernameInput);

    spectator.detectChanges();

    expect(spectator.query('.error')).toHaveText('Username is required');
  });
});
```

### Testing Components with Child Components

```typescript
@Component({
  selector: 'app-child',
  template: `<div>{{ message }}</div>`
})
export class ChildComponent {
  @Input() message!: string;
}

@Component({
  selector: 'app-parent',
  template: `
    <div class="parent">
      <app-child [message]="parentMessage"></app-child>
    </div>
  `
})
export class ParentComponent {
  parentMessage = 'Hello from parent';
}

// Test with real child
describe('ParentComponent with real child', () => {
  let spectator: Spectator<ParentComponent>;
  const createComponent = createComponentFactory({
    component: ParentComponent,
    declarations: [ChildComponent]
  });

  beforeEach(() => {
    spectator = createComponent();
  });

  it('should pass message to child', () => {
    expect(spectator.query('app-child div')).toHaveText('Hello from parent');
  });
});

// Test with shallow rendering (mocked child)
describe('ParentComponent with mocked child', () => {
  let spectator: Spectator<ParentComponent>;
  const createComponent = createComponentFactory({
    component: ParentComponent,
    shallow: true
  });

  beforeEach(() => {
    spectator = createComponent();
  });

  it('should render without child implementation', () => {
    expect(spectator.component).toBeTruthy();
  });
});
```

### Testing Async Operations

```typescript
@Component({
  selector: 'app-async-data',
  template: `
    <div class="loading" *ngIf="loading">Loading...</div>
    <div class="data" *ngIf="!loading">{{ data }}</div>
  `
})
export class AsyncDataComponent implements OnInit {
  loading = false;
  data = '';

  constructor(private dataService: DataService) {}

  ngOnInit(): void {
    this.loading = true;
    this.dataService.getData().subscribe(result => {
      this.data = result;
      this.loading = false;
    });
  }
}

// Test
describe('AsyncDataComponent', () => {
  let spectator: Spectator<AsyncDataComponent>;
  const createComponent = createComponentFactory({
    component: AsyncDataComponent,
    mocks: [DataService]
  });

  it('should show loading state', () => {
    const dataService = createComponent.inject(DataService);
    dataService.getData.and.returnValue(new Subject());

    spectator = createComponent();

    expect(spectator.query('.loading')).toExist();
    expect(spectator.query('.data')).not.toExist();
  });

  it('should show data after loading', fakeAsync(() => {
    const dataService = createComponent.inject(DataService);
    dataService.getData.and.returnValue(of('Test Data').pipe(delay(100)));

    spectator = createComponent();

    tick(100);
    spectator.detectChanges();

    expect(spectator.query('.loading')).not.toExist();
    expect(spectator.query('.data')).toHaveText('Test Data');
  }));
});
```

## Testing Routing

### Component with Router Navigation

```typescript
@Component({
  selector: 'app-navigation',
  template: `
    <button class="home" (click)="goHome()">Home</button>
    <button class="profile" (click)="goToProfile(123)">Profile</button>
  `
})
export class NavigationComponent {
  constructor(private router: Router) {}

  goHome(): void {
    this.router.navigate(['/home']);
  }

  goToProfile(id: number): void {
    this.router.navigate(['/profile', id]);
  }
}

// Test
import { Router } from '@angular/router';

describe('NavigationComponent', () => {
  let spectator: Spectator<NavigationComponent>;
  const createComponent = createComponentFactory({
    component: NavigationComponent,
    mocks: [Router]
  });

  beforeEach(() => {
    spectator = createComponent();
  });

  it('should navigate to home', () => {
    const router = spectator.inject(Router);
    spectator.click('.home');

    expect(router.navigate).toHaveBeenCalledWith(['/home']);
  });

  it('should navigate to profile with id', () => {
    const router = spectator.inject(Router);
    spectator.click('.profile');

    expect(router.navigate).toHaveBeenCalledWith(['/profile', 123]);
  });
});
```

## Custom Matchers and Helpers

### Using Built-in Matchers

```typescript
it('should use custom matchers', () => {
  spectator = createComponent();

  // Element existence
  expect(spectator.query('h1')).toExist();
  expect(spectator.query('.missing')).not.toExist();

  // Text content
  expect(spectator.query('h1')).toHaveText('Title');
  expect(spectator.query('h1')).toContainText('Tit');

  // Classes
  expect(spectator.query('.button')).toHaveClass('active');
  expect(spectator.query('.button')).toHaveClass(['btn', 'primary']);

  // Attributes
  expect(spectator.query('input')).toHaveAttribute('type', 'text');
  expect(spectator.query('input')).toHaveAttribute({ type: 'text', name: 'username' });

  // Properties
  expect(spectator.query('input')).toHaveProperty('disabled', false);

  // Value
  expect(spectator.query('input')).toHaveValue('test');

  // Styles
  expect(spectator.query('.colored')).toHaveStyle({ color: 'red' });

  // Disabled state
  expect(spectator.query('button')).toBeDisabled();
  expect(spectator.query('input')).not.toBeDisabled();

  // Visibility
  expect(spectator.query('.visible')).toBeVisible();
  expect(spectator.query('.hidden')).toBeHidden();
});
```

### Query Methods

```typescript
it('should use query methods', () => {
  spectator = createComponent();

  // Single element
  const element = spectator.query('.item');
  const button = spectator.query<HTMLButtonElement>('button');

  // Multiple elements
  const items = spectator.queryAll('.item');
  expect(items).toHaveLength(3);

  // By directive
  const directive = spectator.query(MyDirective);

  // Last element
  const lastItem = spectator.queryLast('.item');

  // Host element
  const host = spectator.element;

  // Debug element
  const debugElement = spectator.debugElement;
});
```

## Common Mistakes

### 1. Not Using createComponentFactory

```typescript
// BAD: Using TestBed directly
beforeEach(() => {
  TestBed.configureTestingModule({
    declarations: [MyComponent],
    imports: [CommonModule]
  });
  fixture = TestBed.createComponent(MyComponent);
  component = fixture.componentInstance;
});

// GOOD: Using Spectator factory
let spectator: Spectator<MyComponent>;
const createComponent = createComponentFactory({
  component: MyComponent,
  imports: [CommonModule]
});

beforeEach(() => spectator = createComponent());
```

### 2. Manual Change Detection

```typescript
// BAD: Manual change detection
component.value = 'test';
fixture.detectChanges(); // Verbose

// GOOD: Spectator handles it
spectator.setInput('value', 'test'); // Auto-detects changes
```

### 3. Complex DOM Queries

```typescript
// BAD: Native queries
const element = fixture.debugElement.query(By.css('.item')).nativeElement;

// GOOD: Spectator shorthand
const element = spectator.query('.item');
```

### 4. Not Using Mocks Parameter

```typescript
// BAD: Manual mocking
providers: [
  { provide: UserService, useValue: jasmine.createSpyObj('UserService', ['getUser']) }
]

// GOOD: Use mocks array
const createComponent = createComponentFactory({
  component: MyComponent,
  mocks: [UserService]
});
```

### 5. Not Testing Emissions

```typescript
// BAD: Not subscribing to outputs
spectator.click('.button');
// How do we know the output emitted?

// GOOD: Subscribe to output
spectator.output('save').subscribe(data => {
  expect(data).toEqual(expectedData);
});
spectator.click('.button');
```

## Best Practices

### 1. Use Factories Consistently

```typescript
// GOOD: One factory per test suite
describe('MyComponent', () => {
  let spectator: Spectator<MyComponent>;
  const createComponent = createComponentFactory(MyComponent);

  beforeEach(() => spectator = createComponent());
});
```

### 2. Leverage Type Safety

```typescript
// GOOD: Type your spectators
let spectator: Spectator<MyComponent>;
let serviceSpectator: SpectatorService<MyService>;
let httpSpectator: SpectatorHttp<MyHttpService>;
```

### 3. Use Props for Input Testing

```typescript
// GOOD: Clear input testing
spectator = createComponent({
  props: {
    title: 'Test Title',
    items: [1, 2, 3]
  }
});
```

### 4. Group Related Tests

```typescript
describe('UserComponent', () => {
  describe('when user is logged in', () => {
    beforeEach(() => {
      spectator = createComponent({
        props: { user: mockUser }
      });
    });

    it('should display user name', () => {});
    it('should show logout button', () => {});
  });

  describe('when user is logged out', () => {
    beforeEach(() => {
      spectator = createComponent({
        props: { user: null }
      });
    });

    it('should show login button', () => {});
  });
});
```

### 5. Test User Interactions

```typescript
// GOOD: Simulate real user behavior
it('should handle form submission', () => {
  spectator.typeInElement('username', 'input[name="username"]');
  spectator.typeInElement('password', 'input[name="password"]');
  spectator.click('button[type="submit"]');

  expect(spectator.component.submitted).toBe(true);
});
```

## Interview Questions

### 1. What is Spectator and what problems does it solve?

**Answer:** Spectator is a testing library for Angular that simplifies and reduces boilerplate in tests. It solves several problems:

- **Verbose TestBed configuration**: Wraps TestBed with cleaner API
- **Manual change detection**: Automatically triggers change detection
- **Complex DOM queries**: Provides jQuery-like selectors
- **Repetitive mocking**: Built-in `mocks` parameter auto-creates spies
- **Readability**: More intuitive and expressive test syntax
- **Type safety**: Full TypeScript support with proper typing

### 2. How does createComponentFactory work?

**Answer:** `createComponentFactory` creates a factory function that configures and instantiates components for testing:

```typescript
const createComponent = createComponentFactory({
  component: MyComponent,
  imports: [CommonModule],
  providers: [MyService],
  mocks: [HttpClient],
  shallow: false,
  detectChanges: true
});

const spectator = createComponent({
  props: { title: 'Test' },
  providers: [/* override providers */]
});
```

It returns a factory that can be called with optional overrides, creating a configured Spectator instance with component, fixture, and testing utilities.

### 3. What's the difference between shallow and deep rendering?

**Answer:**

**Deep rendering** (default): Renders component with all child components fully initialized.

```typescript
const createComponent = createComponentFactory({
  component: ParentComponent,
  declarations: [ChildComponent] // Child fully rendered
});
```

**Shallow rendering**: Renders only the component being tested; child components are mocked.

```typescript
const createComponent = createComponentFactory({
  component: ParentComponent,
  shallow: true // Child components mocked
});
```

Use shallow for focused unit tests; use deep for integration tests.

### 4. How do you test component inputs and outputs with Spectator?

**Answer:**

**Inputs:**
```typescript
// Set on creation
spectator = createComponent({
  props: { title: 'Test' }
});

// Set after creation
spectator.setInput('title', 'New Title');
```

**Outputs:**
```typescript
// Subscribe to output
spectator.output('save').subscribe(data => {
  expect(data).toEqual(expected);
});

// Or use spy
spyOn(spectator.component.save, 'emit');
spectator.click('.save-button');
expect(spectator.component.save.emit).toHaveBeenCalled();
```

### 5. How does the mocks parameter work?

**Answer:** The `mocks` parameter automatically creates Jasmine spy objects for services:

```typescript
const createComponent = createComponentFactory({
  component: MyComponent,
  mocks: [UserService, DataService]
});

// UserService and DataService are automatically mocked
const userService = spectator.inject(UserService);
userService.getUser.and.returnValue(of(mockUser));
```

Spectator creates spies for all methods, saving manual mock creation.

### 6. What are the advantages of SpectatorHttp?

**Answer:** `SpectatorHttp` provides a cleaner API for HTTP testing:

```typescript
let spectator: SpectatorHttp<MyService>;
const createHttp = createHttpFactory(MyService);

spectator.service.getData().subscribe(/* ... */);

// Clean expectation API
spectator.expectOne('/api/data', 'GET').flush(mockData);
```

Advantages:
- Cleaner syntax than HttpTestingController
- Automatic TestBed configuration with HttpClientTestingModule
- Type-safe HTTP method specification
- Integrated with Spectator ecosystem

### 7. How do you test directives with Spectator?

**Answer:** Use `createDirectiveFactory`:

```typescript
const createDirective = createDirectiveFactory(MyDirective);

const spectator = createDirective(
  `<div myDirective="value">Content</div>`
);

// Access directive instance
const directive = spectator.directive;

// Access element
expect(spectator.element).toHaveClass('applied');

// Trigger events
spectator.dispatchMouseEvent(spectator.element, 'click');
```

### 8. What custom matchers does Spectator provide?

**Answer:** Spectator provides many custom matchers:
- `toExist()` / `toBeVisible()` / `toBeHidden()`
- `toHaveText()` / `toContainText()`
- `toHaveClass()` / `toHaveAttribute()` / `toHaveProperty()`
- `toHaveValue()` / `toBeDisabled()` / `toBeChecked()`
- `toHaveStyle()` / `toBeEmpty()`

These make assertions more readable and expressive than vanilla Jasmine.

### 9. How do you test async operations with Spectator?

**Answer:** Use Angular's async utilities with Spectator:

```typescript
it('should handle async', fakeAsync(() => {
  const service = spectator.inject(DataService);
  service.getData.and.returnValue(of('data').pipe(delay(100)));

  spectator.component.loadData();

  tick(100);
  spectator.detectChanges();

  expect(spectator.component.data).toBe('data');
}));
```

Spectator works seamlessly with `fakeAsync`, `tick`, and `flush`.

### 10. When should you use Spectator vs vanilla Angular testing?

**Answer:** Use Spectator when:
- Writing new tests (less boilerplate)
- Team prefers cleaner syntax
- Want faster test writing
- Need better readability
- Testing multiple components/services

Use vanilla Angular testing when:
- Existing test suite (migration cost)
- Team unfamiliar with Spectator
- Need maximum control over TestBed
- Following strict Angular style guidelines

Spectator is generally recommended for new projects.

## Key Takeaways

1. **Spectator reduces boilerplate** by wrapping TestBed with cleaner, more intuitive APIs
2. **Use createComponentFactory** to create reusable test factories with consistent configuration
3. **Automatic change detection** eliminates manual detectChanges() calls in most cases
4. **jQuery-like selectors** make DOM querying simple and readable
5. **Built-in mocks parameter** automatically creates Jasmine spies for services
6. **Custom matchers** like toHaveText, toExist, and toBeDisabled improve test readability
7. **Type-safe** with full TypeScript support for components, services, and directives
8. **Specialized factories** for components, services, directives, pipes, and HTTP testing
9. **Shallow rendering** option for focused unit tests without child component complexity
10. **Compatible with existing Angular testing utilities** like fakeAsync, tick, and flush

## Resources

### Official Documentation
- [Spectator GitHub Repository](https://github.com/ngneat/spectator)
- [Spectator Documentation](https://ngneat.github.io/spectator/)
- [Angular Testing Guide](https://angular.dev/guide/testing)

### Articles and Tutorials
- [Better Angular Testing with Spectator](https://netbasal.com/spectator-2c8ad6e23523)
- [Simplifying Angular Tests with Spectator](https://blog.nrwl.io/spectator-for-angular-testing-5f28b7a4a96c)

### Video Tutorials
- [Spectator Tutorial Series](https://www.youtube.com/results?search_query=angular+spectator+testing)
- [Angular Testing Best Practices](https://www.youtube.com/watch?v=BumgayeUC08)

### Related Libraries
- [@ngneat/spectator-jest](https://github.com/ngneat/spectator#jest) - Jest integration
- [@ngneat/spectator/jest](https://github.com/ngneat/spectator#jest) - Jest-specific utilities

### Community
- [Spectator Issues & Discussions](https://github.com/ngneat/spectator/issues)
- [Stack Overflow - Spectator Tag](https://stackoverflow.com/questions/tagged/spectator)
