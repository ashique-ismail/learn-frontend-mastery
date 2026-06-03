# Integration Testing

## Overview

Integration testing verifies that multiple units of code work correctly together. While unit tests focus on isolated components, integration tests validate the interactions between components, services, APIs, and external systems. These tests catch bugs that emerge from component integration, data flow issues, and interface mismatches that unit tests miss.

## What is Integration Testing?

Integration testing sits between unit tests and end-to-end tests in the testing pyramid:

```
        /\
       /  \  E2E Tests (Few)
      /____\
     /      \
    / Integration \ (Some)
   /____________\
  /              \
 /   Unit Tests   \ (Many)
/__________________\
```

**Integration tests verify:**
- Component interactions
- Data flow between modules
- API integrations
- Database operations
- External service communications
- State management flows

## Component Integration Testing

### React Component Integration

```tsx
// Components to integrate
interface User {
  id: string;
  name: string;
  email: string;
}

// Child component
export const UserCard: React.FC<{ user: User; onDelete: (id: string) => void }> = ({
  user,
  onDelete
}) => {
  return (
    <div className="user-card">
      <h3>{user.name}</h3>
      <p>{user.email}</p>
      <button onClick={() => onDelete(user.id)}>Delete</button>
    </div>
  );
};

// Parent component
export const UserList: React.FC = () => {
  const [users, setUsers] = useState<User[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    fetchUsers()
      .then(setUsers)
      .catch(err => setError(err.message))
      .finally(() => setLoading(false));
  }, []);

  const handleDelete = async (id: string) => {
    try {
      await deleteUser(id);
      setUsers(users.filter(u => u.id !== id));
    } catch (err) {
      setError('Failed to delete user');
    }
  };

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <div className="user-list">
      {users.map(user => (
        <UserCard key={user.id} user={user} onDelete={handleDelete} />
      ))}
    </div>
  );
};

// ✅ Integration test - tests parent and child together
describe('UserList Integration', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('should fetch and display users', async () => {
    // Arrange
    const mockUsers: User[] = [
      { id: '1', name: 'John Doe', email: 'john@example.com' },
      { id: '2', name: 'Jane Smith', email: 'jane@example.com' }
    ];

    (fetchUsers as jest.Mock).mockResolvedValue(mockUsers);

    // Act
    render(<UserList />);

    // Assert - loading state
    expect(screen.getByText('Loading...')).toBeInTheDocument();

    // Wait for users to load
    await waitFor(() => {
      expect(screen.getByText('John Doe')).toBeInTheDocument();
      expect(screen.getByText('Jane Smith')).toBeInTheDocument();
    });
  });

  it('should delete user when delete button clicked', async () => {
    // Arrange
    const mockUsers: User[] = [
      { id: '1', name: 'John Doe', email: 'john@example.com' },
      { id: '2', name: 'Jane Smith', email: 'jane@example.com' }
    ];

    (fetchUsers as jest.Mock).mockResolvedValue(mockUsers);
    (deleteUser as jest.Mock).mockResolvedValue(undefined);

    render(<UserList />);

    await waitFor(() => {
      expect(screen.getByText('John Doe')).toBeInTheDocument();
    });

    // Act - click delete on first user
    const deleteButtons = screen.getAllByText('Delete');
    fireEvent.click(deleteButtons[0]);

    // Assert - user removed from list
    await waitFor(() => {
      expect(screen.queryByText('John Doe')).not.toBeInTheDocument();
      expect(screen.getByText('Jane Smith')).toBeInTheDocument();
    });

    expect(deleteUser).toHaveBeenCalledWith('1');
  });

  it('should handle delete errors', async () => {
    // Arrange
    const mockUsers: User[] = [
      { id: '1', name: 'John Doe', email: 'john@example.com' }
    ];

    (fetchUsers as jest.Mock).mockResolvedValue(mockUsers);
    (deleteUser as jest.Mock).mockRejectedValue(new Error('Server error'));

    render(<UserList />);

    await waitFor(() => {
      expect(screen.getByText('John Doe')).toBeInTheDocument();
    });

    // Act
    const deleteButton = screen.getByText('Delete');
    fireEvent.click(deleteButton);

    // Assert - error message shown, user still in list
    await waitFor(() => {
      expect(screen.getByText('Error: Failed to delete user')).toBeInTheDocument();
      expect(screen.getByText('John Doe')).toBeInTheDocument();
    });
  });

  it('should display error on fetch failure', async () => {
    // Arrange
    (fetchUsers as jest.Mock).mockRejectedValue(new Error('Network error'));

    // Act
    render(<UserList />);

    // Assert
    await waitFor(() => {
      expect(screen.getByText(/Error: Network error/)).toBeInTheDocument();
    });
  });
});
```

### Angular Component Integration

```typescript
// Child component
@Component({
  selector: 'app-product-card',
  template: `
    <div class="product-card">
      <h3>{{ product.name }}</h3>
      <p>{{ product.price | currency }}</p>
      <button (click)="addToCart.emit(product)">Add to Cart</button>
    </div>
  `
})
export class ProductCardComponent {
  @Input() product!: Product;
  @Output() addToCart = new EventEmitter<Product>();
}

// Parent component
@Component({
  selector: 'app-product-list',
  template: `
    <div class="product-list">
      <div *ngIf="loading">Loading...</div>
      <div *ngIf="error" class="error">{{ error }}</div>
      <app-product-card
        *ngFor="let product of products"
        [product]="product"
        (addToCart)="handleAddToCart($event)"
      ></app-product-card>
      <div *ngIf="cartMessage" class="success">{{ cartMessage }}</div>
    </div>
  `
})
export class ProductListComponent implements OnInit {
  products: Product[] = [];
  loading = true;
  error = '';
  cartMessage = '';

  constructor(
    private productService: ProductService,
    private cartService: CartService
  ) {}

  ngOnInit(): void {
    this.productService.getProducts().subscribe({
      next: products => {
        this.products = products;
        this.loading = false;
      },
      error: err => {
        this.error = err.message;
        this.loading = false;
      }
    });
  }

  handleAddToCart(product: Product): void {
    this.cartService.addItem(product).subscribe({
      next: () => {
        this.cartMessage = `${product.name} added to cart`;
        setTimeout(() => this.cartMessage = '', 3000);
      },
      error: err => {
        this.error = `Failed to add ${product.name}`;
      }
    });
  }
}

// ✅ Integration test
describe('ProductList Integration', () => {
  let component: ProductListComponent;
  let fixture: ComponentFixture<ProductListComponent>;
  let productService: jasmine.SpyObj<ProductService>;
  let cartService: jasmine.SpyObj<CartService>;

  beforeEach(() => {
    const productServiceSpy = jasmine.createSpyObj('ProductService', ['getProducts']);
    const cartServiceSpy = jasmine.createSpyObj('CartService', ['addItem']);

    TestBed.configureTestingModule({
      declarations: [ProductListComponent, ProductCardComponent],
      providers: [
        { provide: ProductService, useValue: productServiceSpy },
        { provide: CartService, useValue: cartServiceSpy }
      ]
    });

    fixture = TestBed.createComponent(ProductListComponent);
    component = fixture.componentInstance;
    productService = TestBed.inject(ProductService) as jasmine.SpyObj<ProductService>;
    cartService = TestBed.inject(CartService) as jasmine.SpyObj<CartService>;
  });

  it('should fetch and display products', fakeAsync(() => {
    // Arrange
    const mockProducts: Product[] = [
      { id: '1', name: 'Product 1', price: 10 },
      { id: '2', name: 'Product 2', price: 20 }
    ];

    productService.getProducts.and.returnValue(of(mockProducts));

    // Act
    fixture.detectChanges(); // Triggers ngOnInit

    // Assert - loading state
    expect(fixture.nativeElement.textContent).toContain('Loading...');

    tick(); // Complete async operations
    fixture.detectChanges();

    // Products displayed
    expect(fixture.nativeElement.textContent).toContain('Product 1');
    expect(fixture.nativeElement.textContent).toContain('Product 2');
    expect(fixture.nativeElement.textContent).not.toContain('Loading...');
  }));

  it('should add product to cart when button clicked', fakeAsync(() => {
    // Arrange
    const mockProducts: Product[] = [
      { id: '1', name: 'Product 1', price: 10 }
    ];

    productService.getProducts.and.returnValue(of(mockProducts));
    cartService.addItem.and.returnValue(of(undefined));

    fixture.detectChanges();
    tick();
    fixture.detectChanges();

    // Act - click add to cart button
    const button = fixture.nativeElement.querySelector('button');
    button.click();
    tick();
    fixture.detectChanges();

    // Assert
    expect(cartService.addItem).toHaveBeenCalledWith(mockProducts[0]);
    expect(fixture.nativeElement.textContent).toContain('Product 1 added to cart');
  }));

  it('should handle product fetch errors', fakeAsync(() => {
    // Arrange
    productService.getProducts.and.returnValue(
      throwError(() => new Error('Network error'))
    );

    // Act
    fixture.detectChanges();
    tick();
    fixture.detectChanges();

    // Assert
    expect(fixture.nativeElement.textContent).toContain('Network error');
    expect(component.loading).toBe(false);
  }));
});
```

## API Integration Testing

### Testing HTTP Requests (React)

```typescript
// API client
export class ApiClient {
  constructor(private baseUrl: string) {}

  async get<T>(endpoint: string): Promise<T> {
    const response = await fetch(`${this.baseUrl}${endpoint}`);
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }
    return response.json();
  }

  async post<T>(endpoint: string, data: any): Promise<T> {
    const response = await fetch(`${this.baseUrl}${endpoint}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    });
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }
    return response.json();
  }
}

// Service using API client
export class UserService {
  constructor(private apiClient: ApiClient) {}

  async createUser(userData: CreateUserRequest): Promise<User> {
    const user = await this.apiClient.post<User>('/users', userData);
    return user;
  }

  async getUserById(id: string): Promise<User> {
    return this.apiClient.get<User>(`/users/${id}`);
  }

  async updateUserProfile(id: string, data: UpdateProfileRequest): Promise<User> {
    return this.apiClient.post<User>(`/users/${id}/profile`, data);
  }
}

// ✅ Integration test with real HTTP mocking
describe('UserService API Integration', () => {
  let apiClient: ApiClient;
  let userService: UserService;

  beforeEach(() => {
    apiClient = new ApiClient('https://api.example.com');
    userService = new UserService(apiClient);

    // Mock fetch globally
    global.fetch = jest.fn();
  });

  afterEach(() => {
    jest.restoreAllMocks();
  });

  it('should create user via API', async () => {
    // Arrange
    const createRequest: CreateUserRequest = {
      name: 'John Doe',
      email: 'john@example.com'
    };

    const expectedUser: User = {
      id: '123',
      ...createRequest
    };

    (global.fetch as jest.Mock).mockResolvedValue({
      ok: true,
      json: async () => expectedUser
    });

    // Act
    const user = await userService.createUser(createRequest);

    // Assert
    expect(user).toEqual(expectedUser);
    expect(global.fetch).toHaveBeenCalledWith(
      'https://api.example.com/users',
      expect.objectContaining({
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(createRequest)
      })
    );
  });

  it('should handle API errors', async () => {
    // Arrange
    (global.fetch as jest.Mock).mockResolvedValue({
      ok: false,
      status: 500,
      statusText: 'Internal Server Error'
    });

    // Act & Assert
    await expect(userService.getUserById('123')).rejects.toThrow(
      'HTTP 500: Internal Server Error'
    );
  });

  it('should chain multiple API calls', async () => {
    // Arrange
    const userId = '123';
    const user: User = { id: userId, name: 'John', email: 'john@example.com' };
    const updateData = { bio: 'Software developer' };
    const updatedUser = { ...user, bio: 'Software developer' };

    (global.fetch as jest.Mock)
      .mockResolvedValueOnce({
        ok: true,
        json: async () => user
      })
      .mockResolvedValueOnce({
        ok: true,
        json: async () => updatedUser
      });

    // Act
    const fetchedUser = await userService.getUserById(userId);
    const updated = await userService.updateUserProfile(userId, updateData);

    // Assert
    expect(fetchedUser).toEqual(user);
    expect(updated).toEqual(updatedUser);
    expect(global.fetch).toHaveBeenCalledTimes(2);
  });
});
```

### Testing with MSW (Mock Service Worker)

```typescript
import { rest } from 'msw';
import { setupServer } from 'msw/node';

// Define API handlers
const handlers = [
  rest.get('https://api.example.com/users/:id', (req, res, ctx) => {
    const { id } = req.params;
    return res(
      ctx.json({
        id,
        name: 'John Doe',
        email: 'john@example.com'
      })
    );
  }),

  rest.post('https://api.example.com/users', async (req, res, ctx) => {
    const body = await req.json();
    return res(
      ctx.status(201),
      ctx.json({
        id: '123',
        ...body
      })
    );
  }),

  rest.post('https://api.example.com/users/:id/profile', async (req, res, ctx) => {
    const { id } = req.params;
    const body = await req.json();
    return res(
      ctx.json({
        id,
        name: 'John Doe',
        email: 'john@example.com',
        ...body
      })
    );
  })
];

const server = setupServer(...handlers);

// ✅ Integration test with MSW
describe('UserService with MSW', () => {
  let apiClient: ApiClient;
  let userService: UserService;

  beforeAll(() => server.listen());
  afterEach(() => server.resetHandlers());
  afterAll(() => server.close());

  beforeEach(() => {
    apiClient = new ApiClient('https://api.example.com');
    userService = new UserService(apiClient);
  });

  it('should create and fetch user', async () => {
    // Act - create user
    const created = await userService.createUser({
      name: 'John Doe',
      email: 'john@example.com'
    });

    // Assert
    expect(created.id).toBe('123');
    expect(created.name).toBe('John Doe');

    // Act - fetch user
    const fetched = await userService.getUserById('123');

    // Assert
    expect(fetched).toEqual({
      id: '123',
      name: 'John Doe',
      email: 'john@example.com'
    });
  });

  it('should handle network errors', async () => {
    // Arrange - override handler to return error
    server.use(
      rest.get('https://api.example.com/users/:id', (req, res, ctx) => {
        return res(ctx.status(500));
      })
    );

    // Act & Assert
    await expect(userService.getUserById('123')).rejects.toThrow();
  });
});
```

## State Management Integration

### Redux Integration Testing

```typescript
// Actions
export const fetchUsersRequest = () => ({ type: 'FETCH_USERS_REQUEST' });
export const fetchUsersSuccess = (users: User[]) => ({
  type: 'FETCH_USERS_SUCCESS',
  payload: users
});
export const fetchUsersFailure = (error: string) => ({
  type: 'FETCH_USERS_FAILURE',
  payload: error
});

// Thunk action
export const fetchUsers = () => async (dispatch: Dispatch) => {
  dispatch(fetchUsersRequest());
  try {
    const users = await api.getUsers();
    dispatch(fetchUsersSuccess(users));
  } catch (error) {
    dispatch(fetchUsersFailure(error.message));
  }
};

// Reducer
interface UsersState {
  data: User[];
  loading: boolean;
  error: string | null;
}

const initialState: UsersState = {
  data: [],
  loading: false,
  error: null
};

export function usersReducer(state = initialState, action: any): UsersState {
  switch (action.type) {
    case 'FETCH_USERS_REQUEST':
      return { ...state, loading: true, error: null };
    case 'FETCH_USERS_SUCCESS':
      return { ...state, loading: false, data: action.payload };
    case 'FETCH_USERS_FAILURE':
      return { ...state, loading: false, error: action.payload };
    default:
      return state;
  }
}

// ✅ Integration test - actions + reducer + async
describe('Users Redux Integration', () => {
  it('should handle successful user fetch', async () => {
    // Arrange
    const mockUsers: User[] = [
      { id: '1', name: 'John' },
      { id: '2', name: 'Jane' }
    ];

    jest.spyOn(api, 'getUsers').mockResolvedValue(mockUsers);

    const store = configureStore({
      reducer: { users: usersReducer }
    });

    // Act
    await store.dispatch(fetchUsers() as any);

    // Assert
    const state = store.getState().users;
    expect(state.data).toEqual(mockUsers);
    expect(state.loading).toBe(false);
    expect(state.error).toBeNull();
  });

  it('should handle failed user fetch', async () => {
    // Arrange
    jest.spyOn(api, 'getUsers').mockRejectedValue(new Error('Network error'));

    const store = configureStore({
      reducer: { users: usersReducer }
    });

    // Act
    await store.dispatch(fetchUsers() as any);

    // Assert
    const state = store.getState().users;
    expect(state.data).toEqual([]);
    expect(state.loading).toBe(false);
    expect(state.error).toBe('Network error');
  });

  it('should set loading state during fetch', () => {
    // Arrange
    const store = configureStore({
      reducer: { users: usersReducer }
    });

    // Act
    store.dispatch(fetchUsersRequest());

    // Assert
    const state = store.getState().users;
    expect(state.loading).toBe(true);
    expect(state.error).toBeNull();
  });
});
```

### React Context Integration

```tsx
// Context
interface AuthContextValue {
  user: User | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  isAuthenticated: boolean;
}

const AuthContext = createContext<AuthContextValue | undefined>(undefined);

export const AuthProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [user, setUser] = useState<User | null>(null);

  const login = async (email: string, password: string) => {
    const user = await authApi.login(email, password);
    setUser(user);
    localStorage.setItem('token', user.token);
  };

  const logout = () => {
    setUser(null);
    localStorage.removeItem('token');
  };

  const value = {
    user,
    login,
    logout,
    isAuthenticated: !!user
  };

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
};

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within AuthProvider');
  return context;
};

// Component using context
export const LoginForm: React.FC = () => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const { login } = useAuth();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    await login(email, password);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={email}
        onChange={e => setEmail(e.target.value)}
        placeholder="Email"
      />
      <input
        type="password"
        value={password}
        onChange={e => setPassword(e.target.value)}
        placeholder="Password"
      />
      <button type="submit">Login</button>
    </form>
  );
};

// ✅ Integration test - Context + Component
describe('Auth Integration', () => {
  beforeEach(() => {
    localStorage.clear();
    jest.clearAllMocks();
  });

  it('should login user and update context', async () => {
    // Arrange
    const mockUser: User = {
      id: '123',
      name: 'John',
      email: 'john@example.com',
      token: 'fake-token'
    };

    jest.spyOn(authApi, 'login').mockResolvedValue(mockUser);

    render(
      <AuthProvider>
        <LoginForm />
      </AuthProvider>
    );

    // Act
    fireEvent.change(screen.getByPlaceholderText('Email'), {
      target: { value: 'john@example.com' }
    });
    fireEvent.change(screen.getByPlaceholderText('Password'), {
      target: { value: 'password123' }
    });
    fireEvent.click(screen.getByText('Login'));

    // Assert
    await waitFor(() => {
      expect(authApi.login).toHaveBeenCalledWith('john@example.com', 'password123');
      expect(localStorage.getItem('token')).toBe('fake-token');
    });
  });

  it('should provide auth state to nested components', async () => {
    // Test component that uses auth context
    const TestComponent = () => {
      const { isAuthenticated, user } = useAuth();
      return (
        <div>
          {isAuthenticated ? <div>Welcome {user?.name}</div> : <div>Not logged in</div>}
        </div>
      );
    };

    const mockUser: User = {
      id: '123',
      name: 'John',
      email: 'john@example.com',
      token: 'fake-token'
    };

    jest.spyOn(authApi, 'login').mockResolvedValue(mockUser);

    render(
      <AuthProvider>
        <TestComponent />
        <LoginForm />
      </AuthProvider>
    );

    // Initially not authenticated
    expect(screen.getByText('Not logged in')).toBeInTheDocument();

    // Login
    fireEvent.change(screen.getByPlaceholderText('Email'), {
      target: { value: 'john@example.com' }
    });
    fireEvent.change(screen.getByPlaceholderText('Password'), {
      target: { value: 'password123' }
    });
    fireEvent.click(screen.getByText('Login'));

    // Assert - authenticated state propagates
    await waitFor(() => {
      expect(screen.getByText('Welcome John')).toBeInTheDocument();
    });
  });
});
```

## Database Integration Testing

```typescript
// Database service
export class UserRepository {
  constructor(private db: Database) {}

  async create(user: CreateUserDTO): Promise<User> {
    const result = await this.db.query(
      'INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *',
      [user.name, user.email]
    );
    return result.rows[0];
  }

  async findById(id: string): Promise<User | null> {
    const result = await this.db.query(
      'SELECT * FROM users WHERE id = $1',
      [id]
    );
    return result.rows[0] || null;
  }

  async update(id: string, data: UpdateUserDTO): Promise<User> {
    const result = await this.db.query(
      'UPDATE users SET name = $1, email = $2 WHERE id = $3 RETURNING *',
      [data.name, data.email, id]
    );
    return result.rows[0];
  }

  async delete(id: string): Promise<void> {
    await this.db.query('DELETE FROM users WHERE id = $1', [id]);
  }
}

// ✅ Integration test with in-memory database
describe('UserRepository Integration', () => {
  let db: Database;
  let repository: UserRepository;

  beforeAll(async () => {
    // Use in-memory or test database
    db = await createTestDatabase();
    await db.query(`
      CREATE TABLE users (
        id SERIAL PRIMARY KEY,
        name VARCHAR(255) NOT NULL,
        email VARCHAR(255) UNIQUE NOT NULL
      )
    `);
  });

  afterAll(async () => {
    await db.close();
  });

  beforeEach(async () => {
    repository = new UserRepository(db);
    // Clear data before each test
    await db.query('DELETE FROM users');
  });

  it('should create and retrieve user', async () => {
    // Arrange
    const userData: CreateUserDTO = {
      name: 'John Doe',
      email: 'john@example.com'
    };

    // Act - create user
    const created = await repository.create(userData);

    // Assert
    expect(created.id).toBeDefined();
    expect(created.name).toBe(userData.name);
    expect(created.email).toBe(userData.email);

    // Act - retrieve user
    const retrieved = await repository.findById(created.id);

    // Assert
    expect(retrieved).toEqual(created);
  });

  it('should update user', async () => {
    // Arrange
    const created = await repository.create({
      name: 'John Doe',
      email: 'john@example.com'
    });

    // Act
    const updated = await repository.update(created.id, {
      name: 'Jane Doe',
      email: 'jane@example.com'
    });

    // Assert
    expect(updated.id).toBe(created.id);
    expect(updated.name).toBe('Jane Doe');
    expect(updated.email).toBe('jane@example.com');
  });

  it('should delete user', async () => {
    // Arrange
    const created = await repository.create({
      name: 'John Doe',
      email: 'john@example.com'
    });

    // Act
    await repository.delete(created.id);

    // Assert
    const retrieved = await repository.findById(created.id);
    expect(retrieved).toBeNull();
  });

  it('should enforce unique email constraint', async () => {
    // Arrange
    await repository.create({
      name: 'John Doe',
      email: 'john@example.com'
    });

    // Act & Assert
    await expect(
      repository.create({
        name: 'Jane Doe',
        email: 'john@example.com' // Duplicate email
      })
    ).rejects.toThrow();
  });
});
```

## Router Integration Testing

```tsx
// React Router integration
import { BrowserRouter, Routes, Route, useNavigate } from 'react-router-dom';

const Home: React.FC = () => {
  const navigate = useNavigate();
  return (
    <div>
      <h1>Home</h1>
      <button onClick={() => navigate('/profile')}>Go to Profile</button>
    </div>
  );
};

const Profile: React.FC = () => {
  const { user } = useAuth();
  return (
    <div>
      <h1>Profile</h1>
      <p>{user?.name}</p>
    </div>
  );
};

const App: React.FC = () => {
  return (
    <AuthProvider>
      <BrowserRouter>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/profile" element={<Profile />} />
        </Routes>
      </BrowserRouter>
    </AuthProvider>
  );
};

// ✅ Integration test with router
describe('App Routing Integration', () => {
  it('should navigate between pages', async () => {
    // Arrange
    const mockUser: User = {
      id: '123',
      name: 'John Doe',
      email: 'john@example.com',
      token: 'token'
    };

    jest.spyOn(authApi, 'login').mockResolvedValue(mockUser);

    render(<App />, { wrapper: MemoryRouter });

    // Assert - on home page
    expect(screen.getByText('Home')).toBeInTheDocument();

    // Act - navigate to profile
    fireEvent.click(screen.getByText('Go to Profile'));

    // Assert - on profile page
    await waitFor(() => {
      expect(screen.getByText('Profile')).toBeInTheDocument();
    });
  });
});
```

## Common Mistakes

### 1. Testing Too Many Layers

```typescript
// ❌ BAD: Integration test that's really an E2E test
it('should complete entire user journey', async () => {
  // Opens browser, navigates, clicks through entire flow
  // This should be an E2E test, not integration test
});

// ✅ GOOD: Integration test focused on specific integration
it('should integrate authentication with protected routes', async () => {
  // Tests specific integration point
});
```

### 2. Not Isolating External Dependencies

```typescript
// ❌ BAD: Hitting real external services
it('should fetch data from API', async () => {
  const data = await fetch('https://real-api.com/data'); // Real network call
  expect(data).toBeDefined();
});

// ✅ GOOD: Mock external services
it('should fetch data from API', async () => {
  server.use(
    rest.get('https://real-api.com/data', (req, res, ctx) => {
      return res(ctx.json({ data: 'mock' }));
    })
  );
  
  const data = await fetch('https://real-api.com/data');
  expect(data).toBeDefined();
});
```

### 3. Shared State Between Tests

```typescript
// ❌ BAD: Tests share database state
describe('UserService', () => {
  it('should create user', async () => {
    await userService.create({ name: 'John' });
  });
  
  it('should list users', async () => {
    const users = await userService.list();
    expect(users).toHaveLength(1); // Depends on previous test!
  });
});

// ✅ GOOD: Clean state between tests
describe('UserService', () => {
  beforeEach(async () => {
    await clearDatabase();
  });
  
  it('should create user', async () => {
    await userService.create({ name: 'John' });
    const users = await userService.list();
    expect(users).toHaveLength(1);
  });
  
  it('should list users', async () => {
    await userService.create({ name: 'John' });
    await userService.create({ name: 'Jane' });
    const users = await userService.list();
    expect(users).toHaveLength(2);
  });
});
```

## Best Practices

1. **Test Component Integration**: Verify parent and child components work together
2. **Mock External Services**: Use MSW or similar tools for API mocking
3. **Test Data Flow**: Verify data moves correctly through layers
4. **Use Test Databases**: In-memory or dedicated test databases for DB tests
5. **Clean Up Between Tests**: Reset state, clear mocks, clean database
6. **Test Error Paths**: Verify error handling across integration points
7. **Focus on Interfaces**: Test how components communicate, not internals
8. **Keep Tests Focused**: Test specific integrations, not entire systems
9. **Use Real Implementations**: Don't mock everything, test real integration
10. **Make Tests Readable**: Clear arrange-act-assert structure

## When to Use Integration Tests

**Use integration tests when:**
- Testing component interactions
- Verifying data flow between modules
- Testing API integrations
- Testing state management
- Testing routing and navigation
- Testing database operations
- Verifying error propagation

**Don't use integration tests for:**
- Testing single units in isolation (use unit tests)
- Testing complete user workflows (use E2E tests)
- Testing UI appearance (use visual regression tests)
- Testing every edge case (use unit tests)

## Interview Questions

1. **What's the difference between unit and integration tests?**
   - Unit tests verify individual components in isolation. Integration tests verify multiple components work correctly together.

2. **How do you handle external dependencies in integration tests?**
   - Mock external services (APIs, databases) using tools like MSW, test databases, or service mocks. Test real integration between internal components.

3. **What's the testing pyramid and where do integration tests fit?**
   - More unit tests (base), some integration tests (middle), few E2E tests (top). Integration tests balance coverage and speed.

4. **How do you test API integration without hitting real endpoints?**
   - Use Mock Service Worker (MSW), jest.mock, or dedicated mocking libraries to intercept and mock network requests.

5. **What makes a good integration test?**
   - Tests real integration points, focused scope, isolated from external dependencies, reliable, readable, and reasonably fast.

## Key Takeaways

1. Integration tests verify multiple units work together correctly
2. They sit between unit tests and E2E tests in the testing pyramid
3. Mock external dependencies but test real internal integrations
4. Test data flow and component communication
5. Clean up state between tests to ensure isolation
6. Focus on integration points and interfaces
7. Use tools like MSW for API mocking
8. Test both happy paths and error scenarios
9. Keep tests focused on specific integrations
10. Integration tests catch bugs that unit tests miss

## Resources

- **React Testing Library**: https://testing-library.com/docs/react-testing-library/intro
- **Angular Testing Guide**: https://angular.io/guide/testing
- **Mock Service Worker**: https://mswjs.io/
- **Testing JavaScript**: https://testingjavascript.com/
- **Martin Fowler - Integration Testing**: https://martinfowler.com/bliki/IntegrationTest.html
- **Kent C. Dodds - Testing Blog**: https://kentcdodds.com/blog
