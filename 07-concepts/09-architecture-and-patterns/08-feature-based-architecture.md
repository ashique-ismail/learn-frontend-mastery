# Feature-Based Architecture

## The Idea

**In plain English:** Feature-based architecture is a way of organizing a codebase by grouping all the code for one specific thing the app can do (a "feature") into its own folder, instead of spreading it across separate folders for each type of file. A "feature" is just a distinct capability of your app, like a shopping cart or a user profile.

**Real-world analogy:** Think of a school where each subject has its own dedicated classroom that contains everything needed for that class — the textbooks, the worksheets, the teacher's materials, and the supplies — all in one room. Compare that to a school where all textbooks are stored in one giant room, all worksheets in another, and all supplies in a third, so every teacher has to run to multiple rooms to get what they need.

- The subject classroom = a feature folder (e.g., `shopping-cart/`)
- The textbooks, worksheets, and supplies inside that room = the components, logic, and data files for that feature
- The giant shared supply closet = the `shared/` folder for things every class genuinely needs

---

## Table of Contents
1. [Introduction](#introduction)
2. [Feature vs Layer Organization](#feature-vs-layer-organization)
3. [Feature Structure](#feature-structure)
4. [Implementation Strategies](#implementation-strategies)
5. [Implementation Examples](#implementation-examples)
6. [Common Mistakes](#common-mistakes)
7. [Best Practices](#best-practices)
8. [When to Use/Not to Use](#when-to-usenot-to-use)
9. [Interview Questions](#interview-questions)
10. [Key Takeaways](#key-takeaways)
11. [Resources](#resources)

## Introduction

Feature-Based Architecture (also called Feature-Sliced Architecture or Vertical Slice Architecture) organizes code around business features or capabilities rather than technical layers. Each feature contains all the code needed to implement that feature, including UI components, business logic, and data access.

This approach improves modularity, team scalability, and code discoverability by keeping related code together.

## Feature vs Layer Organization

### Layer-Based Structure (Traditional)

```
src/
├── components/
│   ├── UserProfile.tsx
│   ├── UserList.tsx
│   ├── ProductCard.tsx
│   └── ProductList.tsx
├── services/
│   ├── userService.ts
│   └── productService.ts
├── repositories/
│   ├── userRepository.ts
│   └── productRepository.ts
└── models/
    ├── User.ts
    └── Product.ts
```

Problems:
- Related code scattered across folders
- Hard to find all code for a feature
- Changes require touching multiple folders
- Teams conflict on shared folders

### Feature-Based Structure

```
src/
├── features/
│   ├── user-profile/
│   │   ├── components/
│   │   │   ├── UserProfile.tsx
│   │   │   └── UserAvatar.tsx
│   │   ├── hooks/
│   │   │   └── useUserProfile.ts
│   │   ├── services/
│   │   │   └── userProfileService.ts
│   │   ├── types.ts
│   │   └── index.ts
│   ├── user-list/
│   │   ├── components/
│   │   │   ├── UserList.tsx
│   │   │   └── UserListItem.tsx
│   │   ├── hooks/
│   │   │   └── useUserList.ts
│   │   ├── services/
│   │   │   └── userListService.ts
│   │   └── index.ts
│   └── product-catalog/
│       ├── components/
│       ├── hooks/
│       ├── services/
│       └── index.ts
└── shared/
    ├── api/
    ├── utils/
    └── types/
```

Benefits:
- All feature code in one place
- Easy to find and modify features
- Teams can own entire features
- Clear feature boundaries

## Feature Structure

### Basic Feature Structure

```
feature-name/
├── components/          # Feature-specific UI components
│   ├── FeatureMain.tsx
│   ├── FeatureForm.tsx
│   └── FeatureList.tsx
├── hooks/              # Feature-specific hooks
│   ├── useFeature.ts
│   └── useFeatureForm.ts
├── services/           # Business logic for feature
│   └── featureService.ts
├── api/               # API calls for feature
│   └── featureApi.ts
├── store/             # State management for feature
│   ├── featureSlice.ts
│   └── featureSelectors.ts
├── types.ts           # Feature-specific types
├── constants.ts       # Feature-specific constants
└── index.ts           # Public exports
```

### Advanced Feature Structure

```
feature-name/
├── ui/                # Presentation layer
│   ├── components/
│   ├── pages/
│   └── styles/
├── model/             # Business logic
│   ├── services/
│   ├── entities/
│   └── validators/
├── api/               # Data access
│   ├── endpoints/
│   ├── transforms/
│   └── types/
├── lib/               # Feature utilities
│   ├── helpers/
│   └── constants/
└── index.ts
```

## Implementation Strategies

### Strategy 1: Simple Feature Modules (React)

```typescript
// ============================================
// features/todo-list/types.ts
// ============================================

export interface Todo {
  id: string;
  text: string;
  completed: boolean;
  createdAt: Date;
}

export interface TodoFormData {
  text: string;
}

// ============================================
// features/todo-list/api/todoApi.ts
// ============================================

export class TodoApi {
  async fetchTodos(): Promise<Todo[]> {
    const response = await fetch('/api/todos');
    return response.json();
  }

  async createTodo(text: string): Promise<Todo> {
    const response = await fetch('/api/todos', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ text })
    });
    return response.json();
  }

  async updateTodo(id: string, updates: Partial<Todo>): Promise<Todo> {
    const response = await fetch(`/api/todos/${id}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(updates)
    });
    return response.json();
  }

  async deleteTodo(id: string): Promise<void> {
    await fetch(`/api/todos/${id}`, { method: 'DELETE' });
  }
}

// ============================================
// features/todo-list/services/todoService.ts
// ============================================

export class TodoService {
  constructor(private api: TodoApi) {}

  async getTodos(): Promise<Todo[]> {
    const todos = await this.api.fetchTodos();
    return todos.sort((a, b) => 
      b.createdAt.getTime() - a.createdAt.getTime()
    );
  }

  async addTodo(text: string): Promise<Todo> {
    if (!text.trim()) {
      throw new Error('Todo text cannot be empty');
    }

    return await this.api.createTodo(text.trim());
  }

  async toggleTodo(todo: Todo): Promise<Todo> {
    return await this.api.updateTodo(todo.id, {
      completed: !todo.completed
    });
  }

  async removeTodo(id: string): Promise<void> {
    await this.api.deleteTodo(id);
  }

  getActiveTodos(todos: Todo[]): Todo[] {
    return todos.filter(todo => !todo.completed);
  }

  getCompletedTodos(todos: Todo[]): Todo[] {
    return todos.filter(todo => todo.completed);
  }
}

// ============================================
// features/todo-list/hooks/useTodoList.ts
// ============================================

export function useTodoList() {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const api = useMemo(() => new TodoApi(), []);
  const service = useMemo(() => new TodoService(api), [api]);

  const loadTodos = useCallback(async () => {
    setLoading(true);
    setError(null);
    try {
      const todos = await service.getTodos();
      setTodos(todos);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  }, [service]);

  const addTodo = useCallback(async (text: string) => {
    try {
      const newTodo = await service.addTodo(text);
      setTodos(prev => [newTodo, ...prev]);
    } catch (err) {
      setError(err.message);
      throw err;
    }
  }, [service]);

  const toggleTodo = useCallback(async (todo: Todo) => {
    try {
      const updatedTodo = await service.toggleTodo(todo);
      setTodos(prev =>
        prev.map(t => t.id === updatedTodo.id ? updatedTodo : t)
      );
    } catch (err) {
      setError(err.message);
    }
  }, [service]);

  const removeTodo = useCallback(async (id: string) => {
    try {
      await service.removeTodo(id);
      setTodos(prev => prev.filter(t => t.id !== id));
    } catch (err) {
      setError(err.message);
    }
  }, [service]);

  useEffect(() => {
    loadTodos();
  }, [loadTodos]);

  const activeTodos = useMemo(
    () => service.getActiveTodos(todos),
    [todos, service]
  );

  const completedTodos = useMemo(
    () => service.getCompletedTodos(todos),
    [todos, service]
  );

  return {
    todos,
    activeTodos,
    completedTodos,
    loading,
    error,
    addTodo,
    toggleTodo,
    removeTodo,
    refresh: loadTodos
  };
}

// ============================================
// features/todo-list/components/TodoList.tsx
// ============================================

export function TodoList() {
  const {
    todos,
    activeTodos,
    completedTodos,
    loading,
    error,
    toggleTodo,
    removeTodo
  } = useTodoList();

  if (loading) return <div>Loading todos...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <div className="todo-list">
      <div className="todo-section">
        <h3>Active ({activeTodos.length})</h3>
        {activeTodos.map(todo => (
          <TodoItem
            key={todo.id}
            todo={todo}
            onToggle={() => toggleTodo(todo)}
            onDelete={() => removeTodo(todo.id)}
          />
        ))}
      </div>

      <div className="todo-section">
        <h3>Completed ({completedTodos.length})</h3>
        {completedTodos.map(todo => (
          <TodoItem
            key={todo.id}
            todo={todo}
            onToggle={() => toggleTodo(todo)}
            onDelete={() => removeTodo(todo.id)}
          />
        ))}
      </div>
    </div>
  );
}

// ============================================
// features/todo-list/components/TodoItem.tsx
// ============================================

interface TodoItemProps {
  todo: Todo;
  onToggle: () => void;
  onDelete: () => void;
}

export function TodoItem({ todo, onToggle, onDelete }: TodoItemProps) {
  return (
    <div className={`todo-item ${todo.completed ? 'completed' : ''}`}>
      <input
        type="checkbox"
        checked={todo.completed}
        onChange={onToggle}
      />
      <span className="todo-text">{todo.text}</span>
      <button onClick={onDelete} className="delete-button">
        Delete
      </button>
    </div>
  );
}

// ============================================
// features/todo-list/components/TodoForm.tsx
// ============================================

interface TodoFormProps {
  onSubmit: (text: string) => Promise<void>;
}

export function TodoForm({ onSubmit }: TodoFormProps) {
  const [text, setText] = useState('');
  const [submitting, setSubmitting] = useState(false);

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    if (!text.trim()) return;

    setSubmitting(true);
    try {
      await onSubmit(text);
      setText('');
    } catch (error) {
      alert(`Error: ${error.message}`);
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="todo-form">
      <input
        type="text"
        value={text}
        onChange={e => setText(e.target.value)}
        placeholder="What needs to be done?"
        disabled={submitting}
      />
      <button type="submit" disabled={submitting || !text.trim()}>
        {submitting ? 'Adding...' : 'Add Todo'}
      </button>
    </form>
  );
}

// ============================================
// features/todo-list/components/TodoPage.tsx
// ============================================

export function TodoPage() {
  const { addTodo, ...todoListProps } = useTodoList();

  return (
    <div className="todo-page">
      <h1>My Todos</h1>
      <TodoForm onSubmit={addTodo} />
      <TodoList {...todoListProps} />
    </div>
  );
}

// ============================================
// features/todo-list/index.ts
// ============================================

// Public API of the feature
export { TodoPage } from './components/TodoPage';
export { useTodoList } from './hooks/useTodoList';
export type { Todo, TodoFormData } from './types';

// Internal components are not exported
// They can only be used within the feature
```

### Strategy 2: Feature Modules with Redux (React)

```typescript
// ============================================
// features/shopping-cart/store/cartSlice.ts
// ============================================

interface CartItem {
  productId: string;
  name: string;
  price: number;
  quantity: number;
}

interface CartState {
  items: CartItem[];
  loading: boolean;
  error: string | null;
}

const initialState: CartState = {
  items: [],
  loading: false,
  error: null
};

export const cartSlice = createSlice({
  name: 'cart',
  initialState,
  reducers: {
    addItem: (state, action: PayloadAction<CartItem>) => {
      const existingItem = state.items.find(
        item => item.productId === action.payload.productId
      );

      if (existingItem) {
        existingItem.quantity += action.payload.quantity;
      } else {
        state.items.push(action.payload);
      }
    },

    removeItem: (state, action: PayloadAction<string>) => {
      state.items = state.items.filter(
        item => item.productId !== action.payload
      );
    },

    updateQuantity: (
      state,
      action: PayloadAction<{ productId: string; quantity: number }>
    ) => {
      const item = state.items.find(
        item => item.productId === action.payload.productId
      );
      if (item) {
        item.quantity = action.payload.quantity;
      }
    },

    clearCart: (state) => {
      state.items = [];
    }
  }
});

export const { addItem, removeItem, updateQuantity, clearCart } = cartSlice.actions;

// ============================================
// features/shopping-cart/store/cartSelectors.ts
// ============================================

export const selectCartItems = (state: RootState) => state.cart.items;

export const selectCartTotal = createSelector(
  [selectCartItems],
  (items) => items.reduce((sum, item) => sum + item.price * item.quantity, 0)
);

export const selectCartItemCount = createSelector(
  [selectCartItems],
  (items) => items.reduce((sum, item) => sum + item.quantity, 0)
);

export const selectCartItemById = (productId: string) =>
  createSelector([selectCartItems], (items) =>
    items.find(item => item.productId === productId)
  );

// ============================================
// features/shopping-cart/hooks/useCart.ts
// ============================================

export function useCart() {
  const dispatch = useDispatch();
  const items = useSelector(selectCartItems);
  const total = useSelector(selectCartTotal);
  const itemCount = useSelector(selectCartItemCount);

  const addToCart = useCallback(
    (product: { id: string; name: string; price: number }, quantity: number = 1) => {
      dispatch(
        addItem({
          productId: product.id,
          name: product.name,
          price: product.price,
          quantity
        })
      );
    },
    [dispatch]
  );

  const removeFromCart = useCallback(
    (productId: string) => {
      dispatch(removeItem(productId));
    },
    [dispatch]
  );

  const updateItemQuantity = useCallback(
    (productId: string, quantity: number) => {
      if (quantity <= 0) {
        dispatch(removeItem(productId));
      } else {
        dispatch(updateQuantity({ productId, quantity }));
      }
    },
    [dispatch]
  );

  const clear = useCallback(() => {
    dispatch(clearCart());
  }, [dispatch]);

  return {
    items,
    total,
    itemCount,
    addToCart,
    removeFromCart,
    updateItemQuantity,
    clear
  };
}

// ============================================
// features/shopping-cart/components/CartButton.tsx
// ============================================

export function CartButton() {
  const { itemCount } = useCart();
  const navigate = useNavigate();

  return (
    <button
      className="cart-button"
      onClick={() => navigate('/cart')}
    >
      🛒 Cart ({itemCount})
    </button>
  );
}

// ============================================
// features/shopping-cart/components/CartPage.tsx
// ============================================

export function CartPage() {
  const { items, total, updateItemQuantity, removeFromCart, clear } = useCart();
  const navigate = useNavigate();

  if (items.length === 0) {
    return (
      <div className="cart-empty">
        <h2>Your cart is empty</h2>
        <button onClick={() => navigate('/products')}>
          Continue Shopping
        </button>
      </div>
    );
  }

  return (
    <div className="cart-page">
      <h1>Shopping Cart</h1>

      <div className="cart-items">
        {items.map(item => (
          <div key={item.productId} className="cart-item">
            <div className="item-details">
              <h3>{item.name}</h3>
              <p className="price">${item.price.toFixed(2)}</p>
            </div>

            <div className="item-quantity">
              <button
                onClick={() => updateItemQuantity(item.productId, item.quantity - 1)}
              >
                -
              </button>
              <span>{item.quantity}</span>
              <button
                onClick={() => updateItemQuantity(item.productId, item.quantity + 1)}
              >
                +
              </button>
            </div>

            <div className="item-total">
              ${(item.price * item.quantity).toFixed(2)}
            </div>

            <button
              className="remove-button"
              onClick={() => removeFromCart(item.productId)}
            >
              Remove
            </button>
          </div>
        ))}
      </div>

      <div className="cart-summary">
        <div className="total">
          <strong>Total:</strong> ${total.toFixed(2)}
        </div>
        <button className="checkout-button" onClick={() => navigate('/checkout')}>
          Proceed to Checkout
        </button>
        <button className="clear-button" onClick={clear}>
          Clear Cart
        </button>
      </div>
    </div>
  );
}

// ============================================
// features/shopping-cart/index.ts
// ============================================

export { CartButton } from './components/CartButton';
export { CartPage } from './components/CartPage';
export { useCart } from './hooks/useCart';
export { cartSlice } from './store/cartSlice';
export * from './store/cartSelectors';
```

### Strategy 3: Feature Modules in Angular

```typescript
// ============================================
// features/user-management/user-management.module.ts
// ============================================

@NgModule({
  declarations: [
    UserListComponent,
    UserDetailComponent,
    UserFormComponent,
    UserCardComponent
  ],
  imports: [
    CommonModule,
    ReactiveFormsModule,
    RouterModule.forChild([
      { path: '', component: UserListComponent },
      { path: ':id', component: UserDetailComponent },
      { path: ':id/edit', component: UserFormComponent }
    ])
  ],
  providers: [
    UserService,
    UserRepository,
    UserValidator
  ]
})
export class UserManagementModule {}

// ============================================
// features/user-management/services/user.service.ts
// ============================================

interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user';
  active: boolean;
}

@Injectable()
export class UserService {
  constructor(
    private repository: UserRepository,
    private validator: UserValidator
  ) {}

  getUsers(): Observable<User[]> {
    return this.repository.findAll().pipe(
      map(users => users.sort((a, b) => a.name.localeCompare(b.name)))
    );
  }

  getUser(id: string): Observable<User> {
    return this.repository.findById(id).pipe(
      tap(user => {
        if (!user) {
          throw new Error('User not found');
        }
      })
    );
  }

  createUser(userData: Omit<User, 'id'>): Observable<User> {
    const errors = this.validator.validate(userData);
    if (errors.length > 0) {
      return throwError(() => new Error(errors.join(', ')));
    }

    return this.repository.create(userData);
  }

  updateUser(id: string, updates: Partial<User>): Observable<User> {
    return this.repository.update(id, updates);
  }

  deleteUser(id: string): Observable<void> {
    return this.repository.delete(id);
  }

  getActiveUsers(): Observable<User[]> {
    return this.getUsers().pipe(
      map(users => users.filter(user => user.active))
    );
  }

  getAdminUsers(): Observable<User[]> {
    return this.getUsers().pipe(
      map(users => users.filter(user => user.role === 'admin'))
    );
  }
}

// ============================================
// features/user-management/repositories/user.repository.ts
// ============================================

@Injectable()
export class UserRepository {
  constructor(private http: HttpClient) {}

  findAll(): Observable<User[]> {
    return this.http.get<User[]>('/api/users');
  }

  findById(id: string): Observable<User> {
    return this.http.get<User>(`/api/users/${id}`);
  }

  create(user: Omit<User, 'id'>): Observable<User> {
    return this.http.post<User>('/api/users', user);
  }

  update(id: string, updates: Partial<User>): Observable<User> {
    return this.http.patch<User>(`/api/users/${id}`, updates);
  }

  delete(id: string): Observable<void> {
    return this.http.delete<void>(`/api/users/${id}`);
  }
}

// ============================================
// features/user-management/validators/user.validator.ts
// ============================================

@Injectable()
export class UserValidator {
  validate(user: Partial<User>): string[] {
    const errors: string[] = [];

    if (!user.name || user.name.trim().length === 0) {
      errors.push('Name is required');
    }

    if (!user.email || !this.isValidEmail(user.email)) {
      errors.push('Valid email is required');
    }

    if (!user.role || !['admin', 'user'].includes(user.role)) {
      errors.push('Valid role is required');
    }

    return errors;
  }

  private isValidEmail(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }
}

// ============================================
// features/user-management/components/user-list.component.ts
// ============================================

@Component({
  selector: 'app-user-list',
  template: `
    <div class="user-list">
      <div class="header">
        <h1>Users</h1>
        <button routerLink="/users/new">Add User</button>
      </div>

      <div class="filters">
        <button (click)="filterType = 'all'" [class.active]="filterType === 'all'">
          All
        </button>
        <button (click)="filterType = 'active'" [class.active]="filterType === 'active'">
          Active
        </button>
        <button (click)="filterType = 'admin'" [class.active]="filterType === 'admin'">
          Admins
        </button>
      </div>

      <div *ngIf="loading$ | async">Loading...</div>
      <div *ngIf="error$ | async as error" class="error">{{ error }}</div>

      <div class="users" *ngIf="!(loading$ | async) && !(error$ | async)">
        <app-user-card
          *ngFor="let user of filteredUsers$ | async"
          [user]="user"
          (edit)="handleEdit($event)"
          (delete)="handleDelete($event)"
        ></app-user-card>
      </div>
    </div>
  `
})
export class UserListComponent implements OnInit {
  users$!: Observable<User[]>;
  filteredUsers$!: Observable<User[]>;
  loading$ = new BehaviorSubject(true);
  error$ = new BehaviorSubject<string | null>(null);
  filterType: 'all' | 'active' | 'admin' = 'all';

  constructor(
    private userService: UserService,
    private router: Router
  ) {}

  ngOnInit(): void {
    this.loadUsers();
  }

  private loadUsers(): void {
    this.loading$.next(true);
    this.error$.next(null);

    this.users$ = this.userService.getUsers().pipe(
      tap(() => this.loading$.next(false)),
      catchError(error => {
        this.error$.next(error.message);
        this.loading$.next(false);
        return of([]);
      }),
      shareReplay(1)
    );

    this.updateFilteredUsers();
  }

  private updateFilteredUsers(): void {
    switch (this.filterType) {
      case 'active':
        this.filteredUsers$ = this.userService.getActiveUsers();
        break;
      case 'admin':
        this.filteredUsers$ = this.userService.getAdminUsers();
        break;
      default:
        this.filteredUsers$ = this.users$;
    }
  }

  handleEdit(userId: string): void {
    this.router.navigate(['/users', userId, 'edit']);
  }

  handleDelete(userId: string): void {
    if (confirm('Are you sure you want to delete this user?')) {
      this.userService.deleteUser(userId).subscribe({
        next: () => this.loadUsers(),
        error: (error) => alert(`Error: ${error.message}`)
      });
    }
  }
}

// ============================================
// features/user-management/components/user-card.component.ts
// ============================================

@Component({
  selector: 'app-user-card',
  template: `
    <div class="user-card" [class.inactive]="!user.active">
      <div class="user-info">
        <h3>{{ user.name }}</h3>
        <p>{{ user.email }}</p>
        <span class="role-badge">{{ user.role }}</span>
      </div>
      <div class="user-actions">
        <button (click)="edit.emit(user.id)">Edit</button>
        <button (click)="delete.emit(user.id)" class="danger">Delete</button>
      </div>
    </div>
  `
})
export class UserCardComponent {
  @Input() user!: User;
  @Output() edit = new EventEmitter<string>();
  @Output() delete = new EventEmitter<string>();
}

// ============================================
// features/user-management/index.ts
// ============================================

export { UserManagementModule } from './user-management.module';
export { UserService } from './services/user.service';
export type { User } from './services/user.service';
```

## Common Mistakes

### 1. Creating Too Many Shared Dependencies

```typescript
// BAD: Features heavily dependent on shared code
// features/user-profile/components/UserProfile.tsx
import { Card } from '@shared/components/Card';
import { Button } from '@shared/components/Button';
import { formatDate } from '@shared/utils/date';
import { useAuth } from '@shared/hooks/useAuth';
import { API_BASE } from '@shared/constants';

// Too many shared dependencies defeat the purpose

// GOOD: Feature is mostly self-contained
// features/user-profile/components/UserProfile.tsx
import { formatDate } from '../utils/formatDate';
import { UserCard } from './UserCard';
import { EditButton } from './EditButton';

// Only truly shared utilities imported from shared
import { useAuth } from '@shared/auth';
```

### 2. Mixing Feature Concerns

```typescript
// BAD: Features directly importing from each other
// features/orders/services/orderService.ts
import { UserService } from '../../users/services/userService'; // Cross-feature import!

// GOOD: Features communicate through shared layer or events
// features/orders/services/orderService.ts
import { EventBus } from '@shared/events';

class OrderService {
  async createOrder(userId: string) {
    // Create order
    const order = await this.repository.create();
    
    // Notify other features through event
    EventBus.publish('order.created', { orderId: order.id, userId });
  }
}

// features/users/services/userActivityService.ts
import { EventBus } from '@shared/events';

EventBus.subscribe('order.created', (data) => {
  // Update user activity
});
```

### 3. Inconsistent Feature Structure

```typescript
// BAD: Each feature organized differently
features/
├── user-profile/
│   ├── UserProfile.tsx        # Components at root
│   └── userService.ts
├── orders/
│   ├── components/           # Components in folder
│   └── services/
└── products/
    ├── ui/                   # Different naming
    └── logic/

// GOOD: Consistent structure across features
features/
├── user-profile/
│   ├── components/
│   ├── hooks/
│   └── services/
├── orders/
│   ├── components/
│   ├── hooks/
│   └── services/
└── products/
    ├── components/
    ├── hooks/
    └── services/
```

## Best Practices

### 1. Define Clear Feature Boundaries

```typescript
// Each feature exports a clear public API

// features/authentication/index.ts
export { LoginPage } from './components/LoginPage';
export { useAuth } from './hooks/useAuth';
export { AuthProvider } from './context/AuthContext';
export type { User, AuthState } from './types';

// Don't export internal components
// LoginForm, LoginButton stay private
```

### 2. Use Shared Layer for Common Code

```typescript
// shared/api/client.ts
export class ApiClient {
  async get<T>(url: string): Promise<T> {
    const response = await fetch(url);
    return response.json();
  }
}

// shared/hooks/useAsync.ts
export function useAsync<T>(asyncFn: () => Promise<T>) {
  // Reusable async hook
}

// Features use shared utilities
import { ApiClient } from '@shared/api/client';
import { useAsync } from '@shared/hooks/useAsync';
```

### 3. Keep Features Independent

```typescript
// Features should work independently

// features/todo-list/index.ts
export function TodoListFeature() {
  return (
    <TodoProvider>
      <TodoPage />
    </TodoProvider>
  );
}

// App can use feature without knowing internals
function App() {
  return (
    <Router>
      <Route path="/todos" element={<TodoListFeature />} />
    </Router>
  );
}
```

### 4. Use Feature Flags for Optional Features

```typescript
// shared/config/features.ts
export const FEATURE_FLAGS = {
  userProfiles: true,
  darkMode: true,
  advancedSearch: false
};

// App.tsx
function App() {
  return (
    <Router>
      {FEATURE_FLAGS.userProfiles && (
        <Route path="/profile" element={<UserProfileFeature />} />
      )}
      {FEATURE_FLAGS.advancedSearch && (
        <Route path="/search" element={<AdvancedSearchFeature />} />
      )}
    </Router>
  );
}
```

## When to Use/Not to Use

### Use Feature-Based Architecture When:

1. **Large Applications**: Many distinct features
2. **Multiple Teams**: Teams can own features
3. **Frequent Changes**: Features change independently
4. **Modular Requirements**: Features may be enabled/disabled

```typescript
// Good fit: E-commerce platform with distinct features
features/
├── product-catalog/
├── shopping-cart/
├── checkout/
├── user-account/
├── order-history/
└── wishlist/
```

### Don't Use When:

1. **Small Applications**: Overhead not worth it
2. **Tightly Coupled Features**: Everything depends on everything
3. **Shared UI Library**: Building component library
4. **Simple CRUD**: Basic data entry application

```typescript
// Poor fit: Simple blog
src/
├── components/    # Few components, layer-based is fine
├── pages/
└── api/
```

## Interview Questions

### Junior Level

**Q: What is feature-based architecture?**

A: An architectural pattern that organizes code around business features rather than technical layers. Each feature contains all the code needed to implement that capability including UI, logic, and data access.

**Q: What goes in the shared folder?**

A: Truly reusable code that multiple features need, such as API clients, common utilities, UI components used everywhere, and shared types/constants.

### Mid Level

**Q: How do you handle communication between features?**

A: Use a shared event bus or message broker for loose coupling, put shared data in a global store, or create a shared service layer that features can depend on. Avoid direct feature-to-feature imports.

```typescript
// Event-based communication
EventBus.publish('user.updated', { userId });

// Shared store
const user = useSelector(state => state.user);

// Shared service
const user = await UserService.getUser(userId);
```

**Q: How do you prevent circular dependencies between features?**

A: Keep features independent, use events for cross-feature communication, extract shared code to the shared layer, and establish clear dependency rules (features can depend on shared, but not on other features).

### Senior Level

**Q: How do you gradually migrate from layer-based to feature-based architecture?**

A: Start by identifying discrete features, create feature folders alongside existing structure, gradually move code from layer folders to feature folders, establish conventions and patterns, update imports incrementally, and eventually remove empty layer folders.

```typescript
// Phase 1: Hybrid structure
src/
├── components/      # Legacy
├── services/        # Legacy
└── features/        # New
    └── user-profile/

// Phase 2: Mostly migrated
src/
├── components/      # Only shared components
└── features/        # Most code here
    ├── user-profile/
    ├── products/
    └── orders/

// Phase 3: Complete
src/
├── shared/          # Renamed from components
└── features/        # All feature code
```

**Q: How do you handle cross-cutting concerns like authentication in feature-based architecture?**

A: Implement in shared layer as a service/context that features consume, use HOCs or middleware for route protection, or implement as a separate "core" feature that other features depend on.

```typescript
// Shared auth
// shared/auth/AuthProvider.tsx
export function AuthProvider({ children }) {
  // Auth implementation
}

// shared/auth/useAuth.ts
export function useAuth() {
  // Auth hook
}

// Features use shared auth
import { useAuth } from '@shared/auth';

function UserProfile() {
  const { user } = useAuth();
  // Use auth
}
```

## Key Takeaways

1. Feature-based architecture organizes code by business feature
2. Each feature is self-contained with its own components, logic, and data access
3. Related code stays together, improving discoverability and maintainability
4. Teams can own and develop features independently
5. Shared code goes in a shared layer, not duplicated across features
6. Features should not directly depend on other features
7. Use events or shared services for cross-feature communication
8. Consistent structure across features improves navigation
9. Better for large applications with many distinct features
10. Enables feature flags and modular deployment

## Resources

### Articles
- Feature-Sliced Design: https://feature-sliced.design/
- Vertical Slice Architecture by Jimmy Bogard
- Modular Monoliths by Simon Brown

### Books
- "Micro Frontends in Action" by Michael Geers
- "Building Microservices" by Sam Newman (architectural concepts apply)

### Examples
- Feature-Sliced Design examples: https://github.com/feature-sliced
- Nx Monorepo: https://nx.dev/ (supports feature organization)
- Angular Style Guide: Feature modules pattern

### Tools
- Nx: Monorepo tool with feature module support
- Lerna: Managing JavaScript projects with multiple packages
- Dependency Cruiser: Validate architectural rules
