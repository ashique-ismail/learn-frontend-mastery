# Testing with Dependency Injection

## The Idea

**In plain English:** When you write code, some parts depend on other parts (like a light switch depending on wiring). Testing with dependency injection means you can swap out those real parts for fake, controllable stand-ins during tests, so you can check that your code works without needing everything else to be set up and running.

**Real-world analogy:** Imagine a flight simulator used to train pilots. Instead of putting a trainee in a real plane with real passengers and real weather, you plug in a simulator cockpit that mimics everything — controls, instruments, even fake engine sounds. The trainee (the code being tested) behaves exactly as it would in a real flight, but the environment is completely under your control.

- The trainee pilot = the component or service being tested
- The real aircraft = the actual dependency (live API, database, external service)
- The flight simulator cockpit = the test double (mock, stub, or fake) injected in place of the real dependency
- The flight instructor controlling the simulator = the test, which sets up scenarios and checks the pilot's responses

---

## Overview

Dependency Injection makes testing easier by allowing you to replace real dependencies with test doubles (mocks, stubs, spies, fakes). This enables isolated unit testing, controlled test environments, and verification of interactions between components. Proper DI testing practices lead to reliable, fast, and maintainable test suites.

## Core Concepts

### Test Doubles Hierarchy

```
Test Doubles:
┌────────────────────────────────────┐
│          Test Double               │
└────────────┬───────────────────────┘
             │
    ┌────────┴────────┬────────┬─────────┐
    │                 │        │         │
┌───▼───┐     ┌──────▼──┐  ┌──▼──┐  ┌──▼──┐
│ Dummy │     │  Stub   │  │Spy  │  │Fake │
└───────┘     └─────────┘  └─────┘  └─────┘
                    │
                ┌───▼───┐
                │ Mock  │
                └───────┘

Dummy:  Passed but never used
Stub:   Returns predefined responses
Spy:    Records how it was called
Mock:   Verifies expectations
Fake:   Simplified working implementation
```

### Test Double Types

| Type | Purpose | Verification | State |
|------|---------|--------------|-------|
| Dummy | Fulfill parameter list | None | No state |
| Stub | Provide test data | None | Returns values |
| Spy | Record interactions | After execution | Tracks calls |
| Mock | Verify behavior | During/after | Has expectations |
| Fake | Realistic implementation | None | Working logic |

## React Testing Examples

### Basic Mock Injection

```typescript
// services/UserService.ts
export interface IUserService {
  getUser(id: string): Promise<User>;
  updateUser(id: string, data: Partial<User>): Promise<User>;
}

export class UserService implements IUserService {
  constructor(private api: ApiClient) {}
  
  async getUser(id: string): Promise<User> {
    return this.api.get<User>(`/users/${id}`);
  }
  
  async updateUser(id: string, data: Partial<User>): Promise<User> {
    return this.api.post<User>(`/users/${id}`, data);
  }
}

// components/UserProfile.tsx
export function UserProfile({ 
  userId, 
  userService 
}: { 
  userId: string; 
  userService: IUserService; 
}) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    userService.getUser(userId)
      .then(setUser)
      .finally(() => setLoading(false));
  }, [userId, userService]);
  
  if (loading) return <div>Loading...</div>;
  if (!user) return <div>User not found</div>;
  
  return <div>{user.name}</div>;
}

// __tests__/UserProfile.test.tsx
import { render, screen, waitFor } from '@testing-library/react';

describe('UserProfile', () => {
  it('should display user name', async () => {
    // Create mock service
    const mockUserService: IUserService = {
      getUser: jest.fn().mockResolvedValue({
        id: '1',
        name: 'John Doe',
        email: 'john@example.com'
      }),
      updateUser: jest.fn()
    };
    
    // Inject mock via props
    render(<UserProfile userId="1" userService={mockUserService} />);
    
    await waitFor(() => {
      expect(screen.getByText('John Doe')).toBeInTheDocument();
    });
    
    // Verify interaction
    expect(mockUserService.getUser).toHaveBeenCalledWith('1');
    expect(mockUserService.getUser).toHaveBeenCalledTimes(1);
  });
  
  it('should handle errors', async () => {
    const mockUserService: IUserService = {
      getUser: jest.fn().mockRejectedValue(new Error('API Error')),
      updateUser: jest.fn()
    };
    
    render(<UserProfile userId="1" userService={mockUserService} />);
    
    await waitFor(() => {
      expect(screen.getByText('User not found')).toBeInTheDocument();
    });
  });
});
```

### Testing with React Context

```typescript
// contexts/ServiceContext.tsx
export const ServiceContext = createContext<{
  userService: IUserService;
} | null>(null);

export function useServices() {
  const context = useContext(ServiceContext);
  if (!context) throw new Error('useServices must be within provider');
  return context;
}

// components/UserProfile.tsx
export function UserProfile({ userId }: { userId: string }) {
  const { userService } = useServices(); // Get from context
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    userService.getUser(userId).then(setUser);
  }, [userId, userService]);
  
  return user ? <div>{user.name}</div> : <div>Loading...</div>;
}

// __tests__/UserProfile.test.tsx
function renderWithServices(
  ui: React.ReactElement,
  services: { userService: IUserService }
) {
  return render(
    <ServiceContext.Provider value={services}>
      {ui}
    </ServiceContext.Provider>
  );
}

describe('UserProfile', () => {
  it('should display user', async () => {
    const mockUserService: IUserService = {
      getUser: jest.fn().mockResolvedValue({
        id: '1',
        name: 'John Doe'
      }),
      updateUser: jest.fn()
    };
    
    renderWithServices(
      <UserProfile userId="1" />,
      { userService: mockUserService }
    );
    
    await waitFor(() => {
      expect(screen.getByText('John Doe')).toBeInTheDocument();
    });
  });
});
```

### Spy Pattern

```typescript
// Create spy that tracks calls
class SpyUserService implements IUserService {
  calls: Array<{ method: string; args: any[] }> = [];
  
  async getUser(id: string): Promise<User> {
    this.calls.push({ method: 'getUser', args: [id] });
    return {
      id,
      name: 'Test User',
      email: 'test@example.com'
    };
  }
  
  async updateUser(id: string, data: Partial<User>): Promise<User> {
    this.calls.push({ method: 'updateUser', args: [id, data] });
    return { id, ...data } as User;
  }
  
  wasCalledWith(method: string, ...args: any[]): boolean {
    return this.calls.some(
      call => call.method === method && 
              JSON.stringify(call.args) === JSON.stringify(args)
    );
  }
  
  getCallCount(method: string): number {
    return this.calls.filter(call => call.method === method).length;
  }
}

// Test with spy
describe('UserProfile with Spy', () => {
  it('should track service calls', async () => {
    const spy = new SpyUserService();
    
    render(<UserProfile userId="123" userService={spy} />);
    
    await waitFor(() => {
      expect(screen.getByText('Test User')).toBeInTheDocument();
    });
    
    // Verify calls
    expect(spy.wasCalledWith('getUser', '123')).toBe(true);
    expect(spy.getCallCount('getUser')).toBe(1);
  });
});
```

### Stub Pattern

```typescript
// Stub returns predefined data
class StubUserService implements IUserService {
  private users: Map<string, User> = new Map([
    ['1', { id: '1', name: 'Alice', email: 'alice@example.com' }],
    ['2', { id: '2', name: 'Bob', email: 'bob@example.com' }]
  ]);
  
  async getUser(id: string): Promise<User> {
    const user = this.users.get(id);
    if (!user) throw new Error('User not found');
    return user;
  }
  
  async updateUser(id: string, data: Partial<User>): Promise<User> {
    const user = this.users.get(id);
    if (!user) throw new Error('User not found');
    const updated = { ...user, ...data };
    this.users.set(id, updated);
    return updated;
  }
}

// Test with stub
describe('UserProfile with Stub', () => {
  it('should display stubbed user', async () => {
    const stub = new StubUserService();
    
    render(<UserProfile userId="1" userService={stub} />);
    
    await waitFor(() => {
      expect(screen.getByText('Alice')).toBeInTheDocument();
    });
  });
  
  it('should handle missing user', async () => {
    const stub = new StubUserService();
    
    render(<UserProfile userId="999" userService={stub} />);
    
    await waitFor(() => {
      expect(screen.getByText('User not found')).toBeInTheDocument();
    });
  });
});
```

### Fake Implementation

```typescript
// Fake has real logic but simplified
class FakeUserService implements IUserService {
  private users: User[] = [];
  private nextId = 1;
  
  async getUser(id: string): Promise<User> {
    await this.delay(10); // Simulate network delay
    const user = this.users.find(u => u.id === id);
    if (!user) throw new Error('User not found');
    return { ...user };
  }
  
  async updateUser(id: string, data: Partial<User>): Promise<User> {
    await this.delay(10);
    const index = this.users.findIndex(u => u.id === id);
    if (index === -1) throw new Error('User not found');
    
    this.users[index] = { ...this.users[index], ...data };
    return { ...this.users[index] };
  }
  
  async createUser(data: Omit<User, 'id'>): Promise<User> {
    await this.delay(10);
    const user = { ...data, id: String(this.nextId++) };
    this.users.push(user);
    return { ...user };
  }
  
  private delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
  
  // Test helpers
  reset() {
    this.users = [];
    this.nextId = 1;
  }
  
  seed(users: User[]) {
    this.users = [...users];
    this.nextId = Math.max(...users.map(u => parseInt(u.id))) + 1;
  }
}

// Test with fake
describe('UserProfile with Fake', () => {
  let fake: FakeUserService;
  
  beforeEach(() => {
    fake = new FakeUserService();
    fake.seed([
      { id: '1', name: 'Alice', email: 'alice@example.com' }
    ]);
  });
  
  it('should work with realistic fake', async () => {
    render(<UserProfile userId="1" userService={fake} />);
    
    await waitFor(() => {
      expect(screen.getByText('Alice')).toBeInTheDocument();
    });
  });
  
  it('should update user', async () => {
    const updated = await fake.updateUser('1', { name: 'Alice Smith' });
    expect(updated.name).toBe('Alice Smith');
    
    const retrieved = await fake.getUser('1');
    expect(retrieved.name).toBe('Alice Smith');
  });
});
```

## Angular Testing Examples

### Basic Service Testing

```typescript
// services/user.service.ts
@Injectable({
  providedIn: 'root'
})
export class UserService {
  constructor(private http: HttpClient) {}
  
  getUser(id: string): Observable<User> {
    return this.http.get<User>(`/api/users/${id}`);
  }
  
  updateUser(id: string, data: Partial<User>): Observable<User> {
    return this.http.put<User>(`/api/users/${id}`, data);
  }
}

// user.service.spec.ts
describe('UserService', () => {
  let service: UserService;
  let httpMock: HttpTestingController;
  
  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [UserService]
    });
    
    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpTestingController);
  });
  
  afterEach(() => {
    httpMock.verify(); // Verify no outstanding requests
  });
  
  it('should fetch user', (done) => {
    const mockUser = { id: '1', name: 'John', email: 'john@example.com' };
    
    service.getUser('1').subscribe(user => {
      expect(user).toEqual(mockUser);
      done();
    });
    
    const req = httpMock.expectOne('/api/users/1');
    expect(req.request.method).toBe('GET');
    req.flush(mockUser);
  });
  
  it('should handle error', (done) => {
    service.getUser('1').subscribe({
      next: () => fail('should have failed'),
      error: (error) => {
        expect(error.status).toBe(404);
        done();
      }
    });
    
    const req = httpMock.expectOne('/api/users/1');
    req.flush('User not found', { status: 404, statusText: 'Not Found' });
  });
});
```

### Component Testing with Mock Services

```typescript
// components/user-profile/user-profile.component.ts
@Component({
  selector: 'app-user-profile',
  template: `
    <div *ngIf="loading">Loading...</div>
    <div *ngIf="user">
      <h1>{{ user.name }}</h1>
      <p>{{ user.email }}</p>
    </div>
  `
})
export class UserProfileComponent implements OnInit {
  @Input() userId!: string;
  user: User | null = null;
  loading = true;
  
  constructor(private userService: UserService) {}
  
  ngOnInit() {
    this.userService.getUser(this.userId).subscribe({
      next: (user) => {
        this.user = user;
        this.loading = false;
      },
      error: () => {
        this.loading = false;
      }
    });
  }
}

// user-profile.component.spec.ts
describe('UserProfileComponent', () => {
  let component: UserProfileComponent;
  let fixture: ComponentFixture<UserProfileComponent>;
  let mockUserService: jasmine.SpyObj<UserService>;
  
  beforeEach(() => {
    // Create mock service
    mockUserService = jasmine.createSpyObj('UserService', ['getUser', 'updateUser']);
    
    TestBed.configureTestingModule({
      declarations: [UserProfileComponent],
      providers: [
        // Inject mock service
        { provide: UserService, useValue: mockUserService }
      ]
    });
    
    fixture = TestBed.createComponent(UserProfileComponent);
    component = fixture.componentInstance;
  });
  
  it('should display user', () => {
    const mockUser = { id: '1', name: 'John', email: 'john@example.com' };
    mockUserService.getUser.and.returnValue(of(mockUser));
    
    component.userId = '1';
    fixture.detectChanges();
    
    expect(component.loading).toBe(false);
    expect(component.user).toEqual(mockUser);
    expect(mockUserService.getUser).toHaveBeenCalledWith('1');
  });
  
  it('should handle error', () => {
    mockUserService.getUser.and.returnValue(
      throwError(() => new Error('API Error'))
    );
    
    component.userId = '1';
    fixture.detectChanges();
    
    expect(component.loading).toBe(false);
    expect(component.user).toBeNull();
  });
});
```

### Testing with Fake Service

```typescript
// testing/fake-user.service.ts
export class FakeUserService {
  private users = new Map<string, User>([
    ['1', { id: '1', name: 'Alice', email: 'alice@example.com' }],
    ['2', { id: '2', name: 'Bob', email: 'bob@example.com' }]
  ]);
  
  getUser(id: string): Observable<User> {
    const user = this.users.get(id);
    return user 
      ? of(user).pipe(delay(10))
      : throwError(() => new Error('User not found'));
  }
  
  updateUser(id: string, data: Partial<User>): Observable<User> {
    const user = this.users.get(id);
    if (!user) {
      return throwError(() => new Error('User not found'));
    }
    const updated = { ...user, ...data };
    this.users.set(id, updated);
    return of(updated).pipe(delay(10));
  }
  
  addUser(user: User): void {
    this.users.set(user.id, user);
  }
}

// user-profile.component.spec.ts
describe('UserProfileComponent with Fake', () => {
  let component: UserProfileComponent;
  let fixture: ComponentFixture<UserProfileComponent>;
  let fakeService: FakeUserService;
  
  beforeEach(() => {
    fakeService = new FakeUserService();
    
    TestBed.configureTestingModule({
      declarations: [UserProfileComponent],
      providers: [
        { provide: UserService, useValue: fakeService }
      ]
    });
    
    fixture = TestBed.createComponent(UserProfileComponent);
    component = fixture.componentInstance;
  });
  
  it('should work with fake service', fakeAsync(() => {
    component.userId = '1';
    fixture.detectChanges();
    tick(10); // Advance time for delay
    
    expect(component.user?.name).toBe('Alice');
  }));
});
```

### Testing with Dependency Override

```typescript
// Complex service with multiple dependencies
@Injectable()
export class ComplexService {
  constructor(
    private api: ApiService,
    private cache: CacheService,
    private logger: LoggerService
  ) {}
  
  async processData(data: any): Promise<any> {
    this.logger.log('Processing data');
    const cached = this.cache.get('data');
    if (cached) return cached;
    
    const result = await this.api.post('/process', data);
    this.cache.set('data', result);
    return result;
  }
}

// complex.service.spec.ts
describe('ComplexService', () => {
  let service: ComplexService;
  let mockApi: jasmine.SpyObj<ApiService>;
  let mockCache: jasmine.SpyObj<CacheService>;
  let mockLogger: jasmine.SpyObj<LoggerService>;
  
  beforeEach(() => {
    mockApi = jasmine.createSpyObj('ApiService', ['get', 'post']);
    mockCache = jasmine.createSpyObj('CacheService', ['get', 'set']);
    mockLogger = jasmine.createSpyObj('LoggerService', ['log', 'error']);
    
    TestBed.configureTestingModule({
      providers: [
        ComplexService,
        { provide: ApiService, useValue: mockApi },
        { provide: CacheService, useValue: mockCache },
        { provide: LoggerService, useValue: mockLogger }
      ]
    });
    
    service = TestBed.inject(ComplexService);
  });
  
  it('should use cache when available', async () => {
    const cachedData = { result: 'cached' };
    mockCache.get.and.returnValue(cachedData);
    
    const result = await service.processData({ input: 'test' });
    
    expect(result).toEqual(cachedData);
    expect(mockCache.get).toHaveBeenCalledWith('data');
    expect(mockApi.post).not.toHaveBeenCalled();
  });
  
  it('should fetch from API when not cached', async () => {
    const apiData = { result: 'from-api' };
    mockCache.get.and.returnValue(null);
    mockApi.post.and.returnValue(Promise.resolve(apiData));
    
    const result = await service.processData({ input: 'test' });
    
    expect(result).toEqual(apiData);
    expect(mockApi.post).toHaveBeenCalled();
    expect(mockCache.set).toHaveBeenCalledWith('data', apiData);
  });
});
```

## Test Utilities and Helpers

### Mock Factory

```typescript
// testing/mock-factory.ts
export class MockFactory {
  static createMockUserService(): jest.Mocked<IUserService> {
    return {
      getUser: jest.fn(),
      updateUser: jest.fn(),
      deleteUser: jest.fn(),
      listUsers: jest.fn()
    };
  }
  
  static createMockApiClient(): jest.Mocked<IApiClient> {
    return {
      get: jest.fn(),
      post: jest.fn(),
      put: jest.fn(),
      delete: jest.fn()
    };
  }
  
  static createMockLogger(): jest.Mocked<ILogger> {
    return {
      log: jest.fn(),
      error: jest.fn(),
      warn: jest.fn(),
      debug: jest.fn()
    };
  }
}

// Usage in tests
describe('MyComponent', () => {
  it('should work', () => {
    const mockUserService = MockFactory.createMockUserService();
    mockUserService.getUser.mockResolvedValue({ id: '1', name: 'John' });
    
    // Use mock...
  });
});
```

### Test Data Builders

```typescript
// testing/builders/user.builder.ts
export class UserBuilder {
  private user: Partial<User> = {
    id: '1',
    name: 'Test User',
    email: 'test@example.com',
    role: 'user'
  };
  
  withId(id: string): this {
    this.user.id = id;
    return this;
  }
  
  withName(name: string): this {
    this.user.name = name;
    return this;
  }
  
  withEmail(email: string): this {
    this.user.email = email;
    return this;
  }
  
  withRole(role: string): this {
    this.user.role = role;
    return this;
  }
  
  build(): User {
    return this.user as User;
  }
}

// Usage
describe('UserProfile', () => {
  it('should display admin badge', () => {
    const admin = new UserBuilder()
      .withName('Admin User')
      .withRole('admin')
      .build();
    
    const mockService = MockFactory.createMockUserService();
    mockService.getUser.mockResolvedValue(admin);
    
    render(<UserProfile userId="1" userService={mockService} />);
    
    expect(screen.getByText('Admin User')).toBeInTheDocument();
    expect(screen.getByText('Admin')).toBeInTheDocument();
  });
});
```

### Custom Render Function

```typescript
// testing/test-utils.tsx
interface RenderOptions {
  userService?: IUserService;
  authService?: IAuthService;
  router?: Router;
}

export function renderWithProviders(
  ui: React.ReactElement,
  options: RenderOptions = {}
) {
  const {
    userService = MockFactory.createMockUserService(),
    authService = MockFactory.createMockAuthService(),
    router = createMemoryRouter()
  } = options;
  
  const AllProviders = ({ children }: { children: React.ReactNode }) => (
    <ServiceContext.Provider value={{ userService, authService }}>
      <RouterProvider router={router}>
        {children}
      </RouterProvider>
    </ServiceContext.Provider>
  );
  
  return {
    ...render(ui, { wrapper: AllProviders }),
    userService,
    authService,
    router
  };
}

// Usage
describe('UserProfile', () => {
  it('should render with custom services', async () => {
    const mockUserService = MockFactory.createMockUserService();
    mockUserService.getUser.mockResolvedValue({
      id: '1',
      name: 'John'
    });
    
    const { userService } = renderWithProviders(
      <UserProfile userId="1" />,
      { userService: mockUserService }
    );
    
    await waitFor(() => {
      expect(screen.getByText('John')).toBeInTheDocument();
    });
    
    expect(userService.getUser).toHaveBeenCalledWith('1');
  });
});
```

## Integration Testing

### Testing Service Integration

```typescript
// integration/user-workflow.test.ts
describe('User Workflow Integration', () => {
  let userService: UserService;
  let authService: AuthService;
  let apiClient: FakeApiClient;
  
  beforeEach(() => {
    apiClient = new FakeApiClient();
    userService = new UserService(apiClient);
    authService = new AuthService(apiClient);
  });
  
  it('should complete full user workflow', async () => {
    // 1. Login
    const authToken = await authService.login('user@example.com', 'password');
    expect(authToken).toBeDefined();
    
    // 2. Fetch user profile
    const user = await userService.getUser('1');
    expect(user.email).toBe('user@example.com');
    
    // 3. Update profile
    const updated = await userService.updateUser('1', { name: 'New Name' });
    expect(updated.name).toBe('New Name');
    
    // 4. Verify update persisted
    const retrieved = await userService.getUser('1');
    expect(retrieved.name).toBe('New Name');
    
    // 5. Logout
    await authService.logout();
    expect(authService.isAuthenticated()).toBe(false);
  });
});
```

### Component Integration Testing

```typescript
// integration/user-dashboard.test.tsx
describe('User Dashboard Integration', () => {
  it('should handle complete user interaction flow', async () => {
    const fakeUserService = new FakeUserService();
    fakeUserService.seed([
      { id: '1', name: 'Alice', email: 'alice@example.com' }
    ]);
    
    const { userService } = renderWithProviders(
      <UserDashboard />,
      { userService: fakeUserService }
    );
    
    // Wait for initial load
    await waitFor(() => {
      expect(screen.getByText('Alice')).toBeInTheDocument();
    });
    
    // Click edit button
    userEvent.click(screen.getByText('Edit'));
    
    // Change name
    const nameInput = screen.getByLabelText('Name');
    userEvent.clear(nameInput);
    userEvent.type(nameInput, 'Alice Smith');
    
    // Submit form
    userEvent.click(screen.getByText('Save'));
    
    // Verify update
    await waitFor(() => {
      expect(screen.getByText('Alice Smith')).toBeInTheDocument();
    });
    
    // Verify service was called
    const user = await fakeUserService.getUser('1');
    expect(user.name).toBe('Alice Smith');
  });
});
```

## Common Mistakes

### 1. Not Resetting Mocks

```typescript
// Bad - state leaks between tests
describe('UserService', () => {
  const mockApi = { get: jest.fn() };
  
  it('test 1', () => {
    mockApi.get.mockResolvedValue({ id: '1' });
    // Test...
  });
  
  it('test 2', () => {
    // mockApi.get still has mock from test 1!
  });
});

// Good - reset between tests
describe('UserService', () => {
  let mockApi: jest.Mocked<IApiClient>;
  
  beforeEach(() => {
    mockApi = {
      get: jest.fn(),
      post: jest.fn()
    };
  });
  
  it('test 1', () => {
    mockApi.get.mockResolvedValue({ id: '1' });
    // Test...
  });
  
  it('test 2', () => {
    // Fresh mock
  });
});
```

### 2. Over-Mocking

```typescript
// Bad - mocking too much
it('should process data', () => {
  const mockValidator = jest.fn().mockReturnValue(true);
  const mockFormatter = jest.fn().mockReturnValue('formatted');
  const mockLogger = jest.fn();
  
  // Testing implementation details
});

// Good - test behavior
it('should process data', () => {
  const result = service.processData(input);
  expect(result).toEqual(expectedOutput);
});
```

### 3. Not Testing Error Cases

```typescript
// Bad - only happy path
it('should get user', async () => {
  mockService.getUser.mockResolvedValue({ id: '1' });
  const user = await service.getUser('1');
  expect(user.id).toBe('1');
});

// Good - test errors too
it('should handle API errors', async () => {
  mockService.getUser.mockRejectedValue(new Error('API Error'));
  
  await expect(service.getUser('1')).rejects.toThrow('API Error');
});
```

## Best Practices

### 1. Use Test Doubles Appropriately

```typescript
// Stub for queries (no side effects)
const stubUserService = {
  getUser: () => Promise.resolve({ id: '1', name: 'John' })
};

// Mock for commands (verify behavior)
const mockUserService = jest.fn();
mockUserService.updateUser = jest.fn();

// Call command
await service.updateUser('1', { name: 'Jane' });

// Verify it was called
expect(mockUserService.updateUser).toHaveBeenCalledWith('1', { name: 'Jane' });
```

### 2. Keep Tests Isolated

```typescript
// Each test should be independent
describe('UserService', () => {
  let service: UserService;
  let mockApi: jest.Mocked<IApiClient>;
  
  beforeEach(() => {
    // Fresh instances for each test
    mockApi = MockFactory.createMockApiClient();
    service = new UserService(mockApi);
  });
  
  // Tests are independent
});
```

### 3. Test Behavior, Not Implementation

```typescript
// Bad - testing implementation
it('should call api.get with correct parameters', () => {
  service.getUser('1');
  expect(mockApi.get).toHaveBeenCalledWith('/users/1');
});

// Good - testing behavior
it('should return user data', async () => {
  mockApi.get.mockResolvedValue({ id: '1', name: 'John' });
  const user = await service.getUser('1');
  expect(user.name).toBe('John');
});
```

### 4. Use Descriptive Test Names

```typescript
// Bad
it('works', () => {});
it('test user', () => {});

// Good
it('should return user when ID exists', () => {});
it('should throw error when user not found', () => {});
it('should cache user data after first fetch', () => {});
```

## Interview Questions

### Q1: What's the difference between a mock and a stub?
**Answer**: A stub provides predefined responses to method calls and is used for queries. A mock verifies that specific methods were called with specific arguments and is used for commands. Stubs focus on state, mocks focus on behavior verification.

### Q2: How does dependency injection make testing easier?
**Answer**: DI allows you to inject test doubles instead of real dependencies, enabling isolated unit testing, controlling test environment, verifying interactions, and testing edge cases without external dependencies like databases or APIs.

### Q3: When should you use a fake vs a mock?
**Answer**: Use a fake when you need realistic behavior for integration testing or complex scenarios (e.g., in-memory database). Use a mock for unit testing when you want to verify specific interactions and isolate the unit under test.

### Q4: How do you prevent test code from affecting other tests?
**Answer**: Reset/recreate mocks before each test, use fresh service instances, avoid shared state, clean up side effects in afterEach hooks, and ensure tests don't depend on execution order.

## Key Takeaways

1. DI enables easy injection of test doubles
2. Use appropriate test double for each scenario
3. Mocks verify behavior, stubs provide data
4. Fakes provide realistic test implementations
5. Reset mocks between tests
6. Test behavior, not implementation details
7. Create helper functions for common setups
8. Keep tests isolated and independent
9. Test both success and error cases
10. Use builders for complex test data

## Resources

- Jest Documentation: https://jestjs.io/
- Testing Library: https://testing-library.com/
- Angular Testing Guide: https://angular.io/guide/testing
- Martin Fowler's Test Doubles: https://martinfowler.com/bliki/TestDouble.html
- Mocks Aren't Stubs: https://martinfowler.com/articles/mocksArentStubs.html
