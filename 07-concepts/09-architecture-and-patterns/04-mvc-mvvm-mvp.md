# MVC, MVVM, and MVP Patterns

## Table of Contents
1. [Introduction](#introduction)
2. [MVC (Model-View-Controller)](#mvc-model-view-controller)
3. [MVVM (Model-View-ViewModel)](#mvvm-model-view-viewmodel)
4. [MVP (Model-View-Presenter)](#mvp-model-view-presenter)
5. [Comparison](#comparison)
6. [Implementation Examples](#implementation-examples)
7. [Common Mistakes](#common-mistakes)
8. [Best Practices](#best-practices)
9. [When to Use](#when-to-use)
10. [Interview Questions](#interview-questions)
11. [Key Takeaways](#key-takeaways)
12. [Resources](#resources)

## Introduction

MVC, MVVM, and MVP are architectural patterns that separate concerns in user interface applications. They all aim to separate presentation logic from business logic, but differ in how they achieve this separation and how components interact.

## MVC (Model-View-Controller)

### Architecture

```
┌─────────────────────────────────────────────────┐
│                                                  │
│  ┌────────┐                    ┌──────────┐    │
│  │  View  │◄──────────────────│Controller│    │
│  └────────┘                    └──────────┘    │
│      │                              │           │
│      │                              │           │
│      │                              ▼           │
│      │                         ┌────────┐      │
│      └────────────────────────▶│ Model  │      │
│                                 └────────┘      │
│                                                  │
│  User Input → Controller → Model → View         │
└─────────────────────────────────────────────────┘
```

### Components

**Model**: Contains data and business logic
**View**: Displays data to user
**Controller**: Handles user input and updates model

### React Implementation

```typescript
// ============================================
// MODEL
// ============================================

class TodoModel {
  private todos: Todo[] = [];
  private listeners: Array<() => void> = [];

  addTodo(text: string): void {
    const todo: Todo = {
      id: Date.now().toString(),
      text,
      completed: false
    };
    this.todos.push(todo);
    this.notifyListeners();
  }

  toggleTodo(id: string): void {
    const todo = this.todos.find(t => t.id === id);
    if (todo) {
      todo.completed = !todo.completed;
      this.notifyListeners();
    }
  }

  deleteTodo(id: string): void {
    this.todos = this.todos.filter(t => t.id !== id);
    this.notifyListeners();
  }

  getTodos(): Todo[] {
    return [...this.todos];
  }

  subscribe(listener: () => void): () => void {
    this.listeners.push(listener);
    return () => {
      this.listeners = this.listeners.filter(l => l !== listener);
    };
  }

  private notifyListeners(): void {
    this.listeners.forEach(listener => listener());
  }
}

// ============================================
// CONTROLLER
// ============================================

class TodoController {
  constructor(private model: TodoModel) {}

  handleAddTodo(text: string): void {
    if (text.trim()) {
      this.model.addTodo(text.trim());
    }
  }

  handleToggleTodo(id: string): void {
    this.model.toggleTodo(id);
  }

  handleDeleteTodo(id: string): void {
    this.model.deleteTodo(id);
  }

  getTodos(): Todo[] {
    return this.model.getTodos();
  }
}

// ============================================
// VIEW
// ============================================

const TodoContext = createContext<TodoController | null>(null);

function TodoApp() {
  const model = useMemo(() => new TodoModel(), []);
  const controller = useMemo(() => new TodoController(model), [model]);

  return (
    <TodoContext.Provider value={controller}>
      <div className="todo-app">
        <TodoInput />
        <TodoList />
      </div>
    </TodoContext.Provider>
  );
}

function TodoInput() {
  const controller = useContext(TodoContext);
  const [text, setText] = useState('');

  const handleSubmit = (e: FormEvent) => {
    e.preventDefault();
    if (controller) {
      controller.handleAddTodo(text);
      setText('');
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={text}
        onChange={e => setText(e.target.value)}
        placeholder="Add todo..."
      />
      <button type="submit">Add</button>
    </form>
  );
}

function TodoList() {
  const controller = useContext(TodoContext);
  const [todos, setTodos] = useState<Todo[]>([]);

  useEffect(() => {
    if (!controller) return;

    const updateTodos = () => {
      setTodos(controller.getTodos());
    };

    updateTodos();
    // Subscribe to model changes
    const unsubscribe = controller.model.subscribe(updateTodos);
    return unsubscribe;
  }, [controller]);

  if (!controller) return null;

  return (
    <ul className="todo-list">
      {todos.map(todo => (
        <TodoItem
          key={todo.id}
          todo={todo}
          onToggle={() => controller.handleToggleTodo(todo.id)}
          onDelete={() => controller.handleDeleteTodo(todo.id)}
        />
      ))}
    </ul>
  );
}

function TodoItem({ todo, onToggle, onDelete }: TodoItemProps) {
  return (
    <li className={todo.completed ? 'completed' : ''}>
      <input
        type="checkbox"
        checked={todo.completed}
        onChange={onToggle}
      />
      <span>{todo.text}</span>
      <button onClick={onDelete}>Delete</button>
    </li>
  );
}
```

### Angular Implementation

```typescript
// ============================================
// MODEL (Service)
// ============================================

interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

@Injectable({ providedIn: 'root' })
export class TodoModel {
  private todosSubject = new BehaviorSubject<Todo[]>([]);
  todos$ = this.todosSubject.asObservable();

  addTodo(text: string): void {
    const todo: Todo = {
      id: Date.now().toString(),
      text,
      completed: false
    };
    const currentTodos = this.todosSubject.value;
    this.todosSubject.next([...currentTodos, todo]);
  }

  toggleTodo(id: string): void {
    const todos = this.todosSubject.value.map(todo =>
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    );
    this.todosSubject.next(todos);
  }

  deleteTodo(id: string): void {
    const todos = this.todosSubject.value.filter(todo => todo.id !== id);
    this.todosSubject.next(todos);
  }

  getTodos(): Todo[] {
    return this.todosSubject.value;
  }
}

// ============================================
// CONTROLLER (Service)
// ============================================

@Injectable({ providedIn: 'root' })
export class TodoController {
  constructor(private model: TodoModel) {}

  handleAddTodo(text: string): void {
    if (text.trim()) {
      this.model.addTodo(text.trim());
    }
  }

  handleToggleTodo(id: string): void {
    this.model.toggleTodo(id);
  }

  handleDeleteTodo(id: string): void {
    this.model.deleteTodo(id);
  }

  getTodos$(): Observable<Todo[]> {
    return this.model.todos$;
  }
}

// ============================================
// VIEW (Component)
// ============================================

@Component({
  selector: 'app-todo',
  template: `
    <div class="todo-app">
      <form (ngSubmit)="handleAddTodo()">
        <input
          [(ngModel)]="todoText"
          placeholder="Add todo..."
          name="todoText"
        />
        <button type="submit">Add</button>
      </form>

      <ul class="todo-list">
        <li *ngFor="let todo of todos$ | async"
            [class.completed]="todo.completed">
          <input
            type="checkbox"
            [checked]="todo.completed"
            (change)="handleToggleTodo(todo.id)"
          />
          <span>{{ todo.text }}</span>
          <button (click)="handleDeleteTodo(todo.id)">Delete</button>
        </li>
      </ul>
    </div>
  `,
  styles: [`
    .completed { text-decoration: line-through; }
  `]
})
export class TodoComponent {
  todoText = '';
  todos$ = this.controller.getTodos$();

  constructor(private controller: TodoController) {}

  handleAddTodo(): void {
    this.controller.handleAddTodo(this.todoText);
    this.todoText = '';
  }

  handleToggleTodo(id: string): void {
    this.controller.handleToggleTodo(id);
  }

  handleDeleteTodo(id: string): void {
    this.controller.handleDeleteTodo(id);
  }
}
```

## MVVM (Model-View-ViewModel)

### Architecture

```
┌─────────────────────────────────────────────────┐
│                                                  │
│  ┌────────┐         ┌──────────────┐            │
│  │  View  │◄───────▶│  ViewModel   │            │
│  └────────┘         └──────────────┘            │
│      │                     │                     │
│      │  Data Binding       │                     │
│      │                     ▼                     │
│      │                ┌────────┐                 │
│      └───────────────▶│ Model  │                 │
│                        └────────┘                 │
│                                                  │
│  Two-way binding between View and ViewModel     │
└─────────────────────────────────────────────────┘
```

### Components

**Model**: Business logic and data
**View**: UI elements
**ViewModel**: Presentation logic, exposes data for view, handles view logic

### React Implementation with Hooks

```typescript
// ============================================
// MODEL
// ============================================

class UserModel {
  async fetchUser(id: string): Promise<User> {
    const response = await fetch(`/api/users/${id}`);
    return response.json();
  }

  async updateUser(user: User): Promise<User> {
    const response = await fetch(`/api/users/${user.id}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(user)
    });
    return response.json();
  }
}

// ============================================
// VIEWMODEL (Custom Hook)
// ============================================

interface UserViewModel {
  user: User | null;
  loading: boolean;
  error: string | null;
  firstName: string;
  lastName: string;
  email: string;
  isValid: boolean;
  isDirty: boolean;
  setFirstName: (value: string) => void;
  setLastName: (value: string) => void;
  setEmail: (value: string) => void;
  save: () => Promise<void>;
  reset: () => void;
}

function useUserViewModel(userId: string): UserViewModel {
  const model = useMemo(() => new UserModel(), []);
  
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  
  const [firstName, setFirstName] = useState('');
  const [lastName, setLastName] = useState('');
  const [email, setEmail] = useState('');
  
  const [isDirty, setIsDirty] = useState(false);

  // Load user
  useEffect(() => {
    let mounted = true;
    
    setLoading(true);
    model.fetchUser(userId)
      .then(user => {
        if (mounted) {
          setUser(user);
          setFirstName(user.firstName);
          setLastName(user.lastName);
          setEmail(user.email);
          setError(null);
        }
      })
      .catch(err => {
        if (mounted) {
          setError(err.message);
        }
      })
      .finally(() => {
        if (mounted) {
          setLoading(false);
        }
      });
    
    return () => { mounted = false; };
  }, [userId, model]);

  // Track changes
  useEffect(() => {
    if (user) {
      setIsDirty(
        firstName !== user.firstName ||
        lastName !== user.lastName ||
        email !== user.email
      );
    }
  }, [firstName, lastName, email, user]);

  // Validation
  const isValid = useMemo(() => {
    return (
      firstName.trim().length > 0 &&
      lastName.trim().length > 0 &&
      /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)
    );
  }, [firstName, lastName, email]);

  // Save
  const save = useCallback(async () => {
    if (!user || !isValid) return;

    setLoading(true);
    try {
      const updatedUser = await model.updateUser({
        ...user,
        firstName,
        lastName,
        email
      });
      setUser(updatedUser);
      setIsDirty(false);
      setError(null);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  }, [user, firstName, lastName, email, isValid, model]);

  // Reset
  const reset = useCallback(() => {
    if (user) {
      setFirstName(user.firstName);
      setLastName(user.lastName);
      setEmail(user.email);
      setIsDirty(false);
    }
  }, [user]);

  return {
    user,
    loading,
    error,
    firstName,
    lastName,
    email,
    isValid,
    isDirty,
    setFirstName,
    setLastName,
    setEmail,
    save,
    reset
  };
}

// ============================================
// VIEW
// ============================================

function UserProfileEditor({ userId }: { userId: string }) {
  const vm = useUserViewModel(userId);

  if (vm.loading && !vm.user) {
    return <div>Loading...</div>;
  }

  if (vm.error && !vm.user) {
    return <div>Error: {vm.error}</div>;
  }

  return (
    <div className="user-profile-editor">
      <h2>Edit Profile</h2>
      
      <div className="form-field">
        <label>First Name</label>
        <input
          value={vm.firstName}
          onChange={e => vm.setFirstName(e.target.value)}
        />
      </div>

      <div className="form-field">
        <label>Last Name</label>
        <input
          value={vm.lastName}
          onChange={e => vm.setLastName(e.target.value)}
        />
      </div>

      <div className="form-field">
        <label>Email</label>
        <input
          type="email"
          value={vm.email}
          onChange={e => vm.setEmail(e.target.value)}
        />
      </div>

      {vm.error && <div className="error">{vm.error}</div>}

      <div className="actions">
        <button
          onClick={vm.save}
          disabled={!vm.isValid || !vm.isDirty || vm.loading}
        >
          {vm.loading ? 'Saving...' : 'Save'}
        </button>
        <button
          onClick={vm.reset}
          disabled={!vm.isDirty || vm.loading}
        >
          Reset
        </button>
      </div>
    </div>
  );
}
```

### Angular Implementation (MVVM is natural fit)

```typescript
// ============================================
// MODEL (Service)
// ============================================

@Injectable({ providedIn: 'root' })
export class UserModel {
  constructor(private http: HttpClient) {}

  fetchUser(id: string): Observable<User> {
    return this.http.get<User>(`/api/users/${id}`);
  }

  updateUser(user: User): Observable<User> {
    return this.http.put<User>(`/api/users/${user.id}`, user);
  }
}

// ============================================
// VIEWMODEL (Component Class)
// ============================================

@Component({
  selector: 'app-user-profile-editor',
  template: `
    <div class="user-profile-editor" *ngIf="vm$ | async as vm">
      <div *ngIf="vm.loading && !vm.user">Loading...</div>
      <div *ngIf="vm.error && !vm.user">Error: {{ vm.error }}</div>

      <form [formGroup]="userForm" (ngSubmit)="save()" *ngIf="vm.user">
        <h2>Edit Profile</h2>

        <div class="form-field">
          <label>First Name</label>
          <input formControlName="firstName" />
          <div *ngIf="userForm.get('firstName')?.invalid && userForm.get('firstName')?.touched"
               class="error">
            First name is required
          </div>
        </div>

        <div class="form-field">
          <label>Last Name</label>
          <input formControlName="lastName" />
          <div *ngIf="userForm.get('lastName')?.invalid && userForm.get('lastName')?.touched"
               class="error">
            Last name is required
          </div>
        </div>

        <div class="form-field">
          <label>Email</label>
          <input type="email" formControlName="email" />
          <div *ngIf="userForm.get('email')?.invalid && userForm.get('email')?.touched"
               class="error">
            Valid email is required
          </div>
        </div>

        <div *ngIf="vm.error" class="error">{{ vm.error }}</div>

        <div class="actions">
          <button
            type="submit"
            [disabled]="!userForm.valid || !userForm.dirty || vm.loading">
            {{ vm.loading ? 'Saving...' : 'Save' }}
          </button>
          <button
            type="button"
            (click)="reset()"
            [disabled]="!userForm.dirty || vm.loading">
            Reset
          </button>
        </div>
      </form>
    </div>
  `
})
export class UserProfileEditorComponent implements OnInit {
  @Input() userId!: string;

  userForm = this.fb.group({
    firstName: ['', Validators.required],
    lastName: ['', Validators.required],
    email: ['', [Validators.required, Validators.email]]
  });

  private userSubject = new BehaviorSubject<User | null>(null);
  private loadingSubject = new BehaviorSubject<boolean>(false);
  private errorSubject = new BehaviorSubject<string | null>(null);

  vm$ = combineLatest([
    this.userSubject,
    this.loadingSubject,
    this.errorSubject
  ]).pipe(
    map(([user, loading, error]) => ({ user, loading, error }))
  );

  constructor(
    private model: UserModel,
    private fb: FormBuilder
  ) {}

  ngOnInit(): void {
    this.loadUser();
  }

  private loadUser(): void {
    this.loadingSubject.next(true);
    
    this.model.fetchUser(this.userId).subscribe({
      next: (user) => {
        this.userSubject.next(user);
        this.userForm.patchValue({
          firstName: user.firstName,
          lastName: user.lastName,
          email: user.email
        });
        this.errorSubject.next(null);
        this.loadingSubject.next(false);
      },
      error: (error) => {
        this.errorSubject.next(error.message);
        this.loadingSubject.next(false);
      }
    });
  }

  save(): void {
    if (!this.userForm.valid) return;

    const user = this.userSubject.value;
    if (!user) return;

    this.loadingSubject.next(true);

    const updatedUser = {
      ...user,
      firstName: this.userForm.value.firstName!,
      lastName: this.userForm.value.lastName!,
      email: this.userForm.value.email!
    };

    this.model.updateUser(updatedUser).subscribe({
      next: (user) => {
        this.userSubject.next(user);
        this.userForm.markAsPristine();
        this.errorSubject.next(null);
        this.loadingSubject.next(false);
      },
      error: (error) => {
        this.errorSubject.next(error.message);
        this.loadingSubject.next(false);
      }
    });
  }

  reset(): void {
    const user = this.userSubject.value;
    if (user) {
      this.userForm.patchValue({
        firstName: user.firstName,
        lastName: user.lastName,
        email: user.email
      });
      this.userForm.markAsPristine();
    }
  }
}
```

## MVP (Model-View-Presenter)

### Architecture

```
┌─────────────────────────────────────────────────┐
│                                                  │
│  ┌────────┐         ┌──────────────┐            │
│  │  View  │◄───────▶│  Presenter   │            │
│  └────────┘         └──────────────┘            │
│      │                     │                     │
│      │                     │                     │
│      │                     ▼                     │
│      │                ┌────────┐                 │
│      └───────────────▶│ Model  │                 │
│                        └────────┘                 │
│                                                  │
│  Presenter mediates between View and Model      │
└─────────────────────────────────────────────────┘
```

### Components

**Model**: Data and business logic
**View**: Passive display, delegates events to presenter
**Presenter**: Contains presentation logic, updates view

### React Implementation

```typescript
// ============================================
// MODEL
// ============================================

class ProductModel {
  async fetchProducts(): Promise<Product[]> {
    const response = await fetch('/api/products');
    return response.json();
  }

  async searchProducts(query: string): Promise<Product[]> {
    const response = await fetch(`/api/products/search?q=${query}`);
    return response.json();
  }
}

// ============================================
// VIEW INTERFACE
// ============================================

interface IProductListView {
  showProducts(products: Product[]): void;
  showLoading(): void;
  showError(message: string): void;
  getSearchQuery(): string;
}

// ============================================
// PRESENTER
// ============================================

class ProductListPresenter {
  private view: IProductListView;
  private model: ProductModel;

  constructor(view: IProductListView, model: ProductModel) {
    this.view = view;
    this.model = model;
  }

  async loadProducts(): Promise<void> {
    try {
      this.view.showLoading();
      const products = await this.model.fetchProducts();
      this.view.showProducts(products);
    } catch (error) {
      this.view.showError(error.message);
    }
  }

  async handleSearch(): Promise<void> {
    try {
      const query = this.view.getSearchQuery();
      if (!query.trim()) {
        await this.loadProducts();
        return;
      }

      this.view.showLoading();
      const products = await this.model.searchProducts(query);
      this.view.showProducts(products);
    } catch (error) {
      this.view.showError(error.message);
    }
  }

  formatPrice(price: number): string {
    return `$${price.toFixed(2)}`;
  }

  formatAvailability(inStock: boolean): string {
    return inStock ? 'In Stock' : 'Out of Stock';
  }
}

// ============================================
// VIEW IMPLEMENTATION
// ============================================

function ProductListView() {
  const [products, setProducts] = useState<Product[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [searchQuery, setSearchQuery] = useState('');

  // View implements interface
  const view: IProductListView = useMemo(() => ({
    showProducts: (products: Product[]) => {
      setProducts(products);
      setLoading(false);
      setError(null);
    },
    showLoading: () => {
      setLoading(true);
      setError(null);
    },
    showError: (message: string) => {
      setError(message);
      setLoading(false);
    },
    getSearchQuery: () => searchQuery
  }), [searchQuery]);

  // Create presenter
  const presenter = useMemo(() => {
    const model = new ProductModel();
    return new ProductListPresenter(view, model);
  }, [view]);

  // Load initial data
  useEffect(() => {
    presenter.loadProducts();
  }, [presenter]);

  const handleSearch = () => {
    presenter.handleSearch();
  };

  if (loading) {
    return <div>Loading...</div>;
  }

  if (error) {
    return <div>Error: {error}</div>;
  }

  return (
    <div className="product-list">
      <div className="search-bar">
        <input
          value={searchQuery}
          onChange={e => setSearchQuery(e.target.value)}
          placeholder="Search products..."
        />
        <button onClick={handleSearch}>Search</button>
      </div>

      <div className="products">
        {products.map(product => (
          <div key={product.id} className="product-card">
            <h3>{product.name}</h3>
            <p>{presenter.formatPrice(product.price)}</p>
            <p>{presenter.formatAvailability(product.inStock)}</p>
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Angular Implementation

```typescript
// ============================================
// MODEL
// ============================================

@Injectable({ providedIn: 'root' })
export class ProductModel {
  constructor(private http: HttpClient) {}

  fetchProducts(): Observable<Product[]> {
    return this.http.get<Product[]>('/api/products');
  }

  searchProducts(query: string): Observable<Product[]> {
    return this.http.get<Product[]>(`/api/products/search?q=${query}`);
  }
}

// ============================================
// VIEW INTERFACE
// ============================================

export interface IProductListView {
  showProducts(products: Product[]): void;
  showLoading(): void;
  showError(message: string): void;
  getSearchQuery(): string;
}

// ============================================
// PRESENTER
// ============================================

@Injectable()
export class ProductListPresenter {
  private view!: IProductListView;

  constructor(private model: ProductModel) {}

  attachView(view: IProductListView): void {
    this.view = view;
  }

  loadProducts(): void {
    this.view.showLoading();
    
    this.model.fetchProducts().subscribe({
      next: (products) => this.view.showProducts(products),
      error: (error) => this.view.showError(error.message)
    });
  }

  handleSearch(): void {
    const query = this.view.getSearchQuery();
    
    if (!query.trim()) {
      this.loadProducts();
      return;
    }

    this.view.showLoading();
    
    this.model.searchProducts(query).subscribe({
      next: (products) => this.view.showProducts(products),
      error: (error) => this.view.showError(error.message)
    });
  }

  formatPrice(price: number): string {
    return `$${price.toFixed(2)}`;
  }

  formatAvailability(inStock: boolean): string {
    return inStock ? 'In Stock' : 'Out of Stock';
  }
}

// ============================================
// VIEW
// ============================================

@Component({
  selector: 'app-product-list',
  template: `
    <div class="product-list">
      <div class="search-bar">
        <input
          [(ngModel)]="searchQuery"
          placeholder="Search products..."
        />
        <button (click)="handleSearch()">Search</button>
      </div>

      <div *ngIf="loading">Loading...</div>
      <div *ngIf="error" class="error">Error: {{ error }}</div>

      <div class="products" *ngIf="!loading && !error">
        <div *ngFor="let product of products" class="product-card">
          <h3>{{ product.name }}</h3>
          <p>{{ presenter.formatPrice(product.price) }}</p>
          <p>{{ presenter.formatAvailability(product.inStock) }}</p>
        </div>
      </div>
    </div>
  `,
  providers: [ProductListPresenter]
})
export class ProductListComponent implements OnInit, IProductListView {
  products: Product[] = [];
  loading = false;
  error: string | null = null;
  searchQuery = '';

  constructor(public presenter: ProductListPresenter) {
    this.presenter.attachView(this);
  }

  ngOnInit(): void {
    this.presenter.loadProducts();
  }

  // IProductListView implementation
  showProducts(products: Product[]): void {
    this.products = products;
    this.loading = false;
    this.error = null;
  }

  showLoading(): void {
    this.loading = true;
    this.error = null;
  }

  showError(message: string): void {
    this.error = message;
    this.loading = false;
  }

  getSearchQuery(): string {
    return this.searchQuery;
  }

  handleSearch(): void {
    this.presenter.handleSearch();
  }
}
```

## Comparison

### Communication Flow

```
MVC:
User → Controller → Model
Model → View (updates)

MVVM:
User → View → ViewModel (two-way binding)
ViewModel ↔ Model

MVP:
User → View → Presenter
Presenter → Model
Presenter → View (explicit updates)
```

### Testability

```typescript
// MVC: Test controller
describe('TodoController', () => {
  it('should add todo', () => {
    const model = new TodoModel();
    const controller = new TodoController(model);
    
    controller.handleAddTodo('Test');
    
    expect(model.getTodos()).toHaveLength(1);
  });
});

// MVVM: Test ViewModel
describe('useUserViewModel', () => {
  it('should validate form', () => {
    const { result } = renderHook(() => useUserViewModel('user-1'));
    
    act(() => {
      result.current.setEmail('invalid');
    });
    
    expect(result.current.isValid).toBe(false);
  });
});

// MVP: Test Presenter
describe('ProductListPresenter', () => {
  it('should load products', async () => {
    const mockView = {
      showProducts: jest.fn(),
      showLoading: jest.fn(),
      showError: jest.fn(),
      getSearchQuery: jest.fn()
    };
    const mockModel = {
      fetchProducts: jest.fn().mockResolvedValue([{ id: '1' }])
    };
    const presenter = new ProductListPresenter(mockView, mockModel);
    
    await presenter.loadProducts();
    
    expect(mockView.showLoading).toHaveBeenCalled();
    expect(mockView.showProducts).toHaveBeenCalledWith([{ id: '1' }]);
  });
});
```

### Comparison Table

| Aspect | MVC | MVVM | MVP |
|--------|-----|------|-----|
| View Knowledge | View knows Model | View knows ViewModel | View knows only interface |
| View Intelligence | Can query Model | Binds to ViewModel | Very passive |
| Testing | Hard to test View | Easy with ViewModels | Easy with mock views |
| Data Binding | Manual | Two-way | Manual |
| Best For | Simple apps | Data-driven UIs | Complex logic |
| Learning Curve | Easy | Medium | Medium |

## Common Mistakes

### 1. Tight Coupling in MVC

```typescript
// BAD: View directly manipulates model
function TodoView() {
  const [todos, setTodos] = useState([]);
  
  const addTodo = (text: string) => {
    // View shouldn't know model implementation!
    const newTodo = { id: Date.now(), text };
    setTodos([...todos, newTodo]);
  };
}

// GOOD: Controller mediates
function TodoView() {
  const controller = useContext(TodoContext);
  
  const addTodo = (text: string) => {
    controller.handleAddTodo(text); // Controller handles it
  };
}
```

### 2. Business Logic in ViewModel

```typescript
// BAD: Business logic in ViewModel
function useProductViewModel() {
  const calculateDiscount = (price: number) => {
    // Complex business logic doesn't belong here!
    if (price > 1000) return price * 0.2;
    if (price > 500) return price * 0.1;
    return 0;
  };
}

// GOOD: Business logic in Model
class ProductModel {
  calculateDiscount(price: number): number {
    // Business rules in model
    if (price > 1000) return price * 0.2;
    if (price > 500) return price * 0.1;
    return 0;
  }
}

function useProductViewModel() {
  const model = new ProductModel();
  const getDiscountedPrice = (price: number) => {
    return price - model.calculateDiscount(price);
  };
}
```

### 3. Smart Views in MVP

```typescript
// BAD: View contains logic
@Component({...})
export class ProductComponent {
  handleAddToCart(product: Product): void {
    // Logic in view!
    if (product.inStock && product.price > 0) {
      this.cartService.add(product);
    }
  }
}

// GOOD: Presenter contains logic
export class ProductPresenter {
  handleAddToCart(product: Product): void {
    if (!this.canAddToCart(product)) {
      this.view.showError('Cannot add product to cart');
      return;
    }
    this.model.addToCart(product);
    this.view.showSuccess('Added to cart');
  }
  
  private canAddToCart(product: Product): boolean {
    return product.inStock && product.price > 0;
  }
}
```

## Best Practices

### 1. Keep Views Thin

```typescript
// View should only handle presentation
function UserProfile({ userId }: Props) {
  const vm = useUserViewModel(userId);
  
  // No logic, just rendering
  return (
    <div>
      <h1>{vm.fullName}</h1>
      <p>{vm.email}</p>
      <button onClick={vm.save} disabled={!vm.canSave}>
        Save
      </button>
    </div>
  );
}
```

### 2. Use Interfaces for Abstraction

```typescript
// Define view interface for testing
interface IView {
  showData(data: Data[]): void;
  showError(message: string): void;
}

// Presenter works with interface
class Presenter {
  constructor(private view: IView) {}
  
  loadData(): void {
    // Implementation
  }
}

// Easy to test with mock
const mockView: IView = {
  showData: jest.fn(),
  showError: jest.fn()
};
```

### 3. Separate Concerns Clearly

```typescript
// Model: Business logic only
class OrderModel {
  calculateTotal(items: Item[]): number {
    return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }
}

// ViewModel: Presentation logic
function useOrderViewModel(orderId: string) {
  const model = new OrderModel();
  
  const formatTotal = (items: Item[]): string => {
    const total = model.calculateTotal(items);
    return `$${total.toFixed(2)}`;
  };
  
  return { formatTotal };
}

// View: Rendering only
function OrderSummary({ orderId }: Props) {
  const vm = useOrderViewModel(orderId);
  return <div>{vm.formatTotal(items)}</div>;
}
```

## When to Use

### Use MVC When:
- Building server-rendered applications
- Simple applications with straightforward flow
- Team familiar with traditional MVC pattern
- Need clear separation of routing logic

### Use MVVM When:
- Building data-driven UIs
- Need two-way data binding
- Complex forms with validation
- Angular applications (natural fit)
- Real-time data updates

### Use MVP When:
- Need highly testable presentation layer
- Complex presentation logic
- Multiple views sharing same logic
- Legacy applications requiring refactoring

## Interview Questions

### Junior Level

**Q: What is the main difference between MVC and MVVM?**

A: In MVC, the Controller handles user input and updates the Model, then the View observes the Model. In MVVM, the View is bound to the ViewModel through data binding, creating a more direct connection between UI and presentation logic.

**Q: What role does the Presenter play in MVP?**

A: The Presenter acts as a mediator between the View and Model. It receives events from the View, interacts with the Model, and explicitly tells the View what to display.

### Mid Level

**Q: How does testability differ between these patterns?**

A: MVP offers the best testability because the View is completely passive and defined by an interface, making it easy to mock. MVVM is also highly testable through ViewModels. MVC is hardest to test because Views often have direct Model knowledge.

**Q: When would you choose MVP over MVVM in Angular?**

A: Choose MVP when you have complex presentation logic that needs to be thoroughly tested, when you want complete separation between View and logic, or when building component libraries that need to support multiple view implementations.

### Senior Level

**Q: How do you handle navigation in each pattern?**

A: In MVC, Controllers typically handle navigation. In MVVM, ViewModels can emit navigation events that the View framework handles. In MVP, Presenters coordinate navigation by telling Views to navigate, keeping navigation logic testable.

```typescript
// MVC
class ProductController {
  handleProductClick(id: string): void {
    this.router.navigate(['products', id]);
  }
}

// MVVM
function useProductViewModel() {
  const navigate = useNavigate();
  
  const selectProduct = (id: string) => {
    navigate(`/products/${id}`);
  };
  
  return { selectProduct };
}

// MVP
interface IProductView {
  navigateToProduct(id: string): void;
}

class ProductPresenter {
  handleProductClick(id: string): void {
    this.view.navigateToProduct(id);
  }
}
```

**Q: How do you manage state across multiple features using these patterns?**

A: Use a centralized state management solution (Redux, NgRx) that works alongside these patterns. Models/Services can interact with the store, while ViewModels/Presenters subscribe to slices of state and transform it for their views.

## Key Takeaways

1. All three patterns separate presentation from business logic
2. MVC: Controller mediates between View and Model
3. MVVM: View binds to ViewModel, ViewModel updates Model
4. MVP: Presenter mediates, View is passive with interface
5. MVVM works best with two-way binding (Angular, Vue)
6. MVP offers best testability through passive views
7. MVC is simplest but least testable
8. Choose based on: testability needs, framework support, team experience
9. All patterns improve maintainability over spaghetti code
10. Don't mix patterns within same feature

## Resources

### Books
- "Patterns of Enterprise Application Architecture" by Martin Fowler
- "Design Patterns: Elements of Reusable Object-Oriented Software"

### Articles
- GUI Architectures by Martin Fowler
- MVC vs MVP vs MVVM by Microsoft

### Frameworks
- Angular: Built-in MVVM support
- React: Flexible, supports all patterns
- Vue: MVVM-inspired architecture

### Examples
- TodoMVC: Same app in all patterns
- Angular Documentation: MVVM examples
- React Testing Library: Component testing patterns
