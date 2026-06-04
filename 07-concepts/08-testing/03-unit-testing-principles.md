# Unit Testing Principles

## Overview

Unit testing forms the foundation of a robust testing strategy. A unit test verifies a single unit of code (function, method, or class) in isolation from its dependencies. Understanding core principles like the AAA pattern, test isolation, determinism, and speed helps developers write maintainable, reliable tests that catch bugs early and serve as living documentation.

## Core Principles

### 1. Arrange-Act-Assert (AAA) Pattern

The AAA pattern provides a clear structure for every test:

```typescript
// ✅ GOOD: Clear AAA structure
describe('calculateDiscount', () => {
  it('should apply 20% discount for premium members', () => {
    // Arrange - Set up test data and dependencies
    const user = { membership: 'premium' };
    const price = 100;
    
    // Act - Execute the code under test
    const result = calculateDiscount(user, price);
    
    // Assert - Verify the outcome
    expect(result).toBe(80);
  });
});
```

```typescript
// ❌ BAD: Mixed responsibilities, unclear structure
describe('calculateDiscount', () => {
  it('should work correctly', () => {
    const result = calculateDiscount({ membership: 'premium' }, 100);
    expect(result).toBe(80);
    expect(calculateDiscount({ membership: 'basic' }, 100)).toBe(95);
    expect(calculateDiscount({ membership: 'guest' }, 100)).toBe(100);
  });
});
```

**React Component Example:**

```tsx
// Component to test
interface ButtonProps {
  label: string;
  onClick: () => void;
  disabled?: boolean;
}

export const Button: React.FC<ButtonProps> = ({ label, onClick, disabled }) => {
  return (
    <button onClick={onClick} disabled={disabled}>
      {label}
    </button>
  );
};

// Test with AAA pattern
import { render, screen, fireEvent } from '@testing-library/react';

describe('Button', () => {
  it('should call onClick handler when clicked', () => {
    // Arrange
    const handleClick = jest.fn();
    const buttonLabel = 'Submit';
    
    render(<Button label={buttonLabel} onClick={handleClick} />);
    
    // Act
    const button = screen.getByRole('button', { name: buttonLabel });
    fireEvent.click(button);
    
    // Assert
    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it('should not call onClick when disabled', () => {
    // Arrange
    const handleClick = jest.fn();
    render(<Button label="Submit" onClick={handleClick} disabled />);
    
    // Act
    const button = screen.getByRole('button');
    fireEvent.click(button);
    
    // Assert
    expect(handleClick).not.toHaveBeenCalled();
  });
});
```

**Angular Service Example:**

```typescript
// Service to test
@Injectable()
export class UserService {
  constructor(private http: HttpClient) {}
  
  getUser(id: string): Observable<User> {
    return this.http.get<User>(`/api/users/${id}`);
  }
  
  updateUser(user: User): Observable<User> {
    return this.http.put<User>(`/api/users/${user.id}`, user);
  }
}

// Test with AAA pattern
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

  it('should fetch user by id', () => {
    // Arrange
    const mockUser: User = { id: '123', name: 'John Doe', email: 'john@example.com' };
    const userId = '123';
    
    // Act
    service.getUser(userId).subscribe(user => {
      // Assert
      expect(user).toEqual(mockUser);
    });
    
    // Assert (HTTP expectation)
    const req = httpMock.expectOne(`/api/users/${userId}`);
    expect(req.request.method).toBe('GET');
    req.flush(mockUser);
  });
  
  afterEach(() => {
    httpMock.verify();
  });
});
```

### 2. Test Isolation

Each test should be completely independent of other tests:

```typescript
// ❌ BAD: Tests depend on shared state
describe('ShoppingCart', () => {
  const cart = new ShoppingCart(); // Shared instance
  
  it('should add items', () => {
    cart.addItem({ id: '1', name: 'Book', price: 10 });
    expect(cart.getItemCount()).toBe(1);
  });
  
  it('should calculate total', () => {
    // This test depends on the previous test running first!
    expect(cart.getTotal()).toBe(10);
  });
});

// ✅ GOOD: Each test creates its own instance
describe('ShoppingCart', () => {
  let cart: ShoppingCart;
  
  beforeEach(() => {
    cart = new ShoppingCart(); // Fresh instance for each test
  });
  
  it('should add items', () => {
    cart.addItem({ id: '1', name: 'Book', price: 10 });
    expect(cart.getItemCount()).toBe(1);
  });
  
  it('should calculate total', () => {
    cart.addItem({ id: '1', name: 'Book', price: 10 });
    cart.addItem({ id: '2', name: 'Pen', price: 2 });
    expect(cart.getTotal()).toBe(12);
  });
});
```

**React Component Isolation:**

```tsx
// ❌ BAD: DOM pollution between tests
describe('TodoList', () => {
  it('should render todos', () => {
    const { container } = render(<TodoList todos={[{ id: 1, text: 'Task 1' }]} />);
    // Container remains in DOM
  });
  
  it('should handle empty list', () => {
    // Previous test's DOM might interfere
    render(<TodoList todos={[]} />);
  });
});

// ✅ GOOD: Clean up between tests
describe('TodoList', () => {
  afterEach(() => {
    cleanup(); // React Testing Library auto-cleanup
  });
  
  it('should render todos', () => {
    render(<TodoList todos={[{ id: 1, text: 'Task 1' }]} />);
    expect(screen.getByText('Task 1')).toBeInTheDocument();
  });
  
  it('should handle empty list', () => {
    render(<TodoList todos={[]} />);
    expect(screen.getByText('No todos')).toBeInTheDocument();
  });
});
```

### 3. Deterministic Tests

Tests must produce the same result every time they run:

```typescript
// ❌ BAD: Non-deterministic (depends on current time)
export function isBusinessDay(): boolean {
  const day = new Date().getDay();
  return day >= 1 && day <= 5;
}

it('should return true on business days', () => {
  expect(isBusinessDay()).toBe(true); // Fails on weekends!
});

// ✅ GOOD: Inject dependencies for determinism
export function isBusinessDay(date: Date = new Date()): boolean {
  const day = date.getDay();
  return day >= 1 && day <= 5;
}

it('should return true on business days', () => {
  const monday = new Date('2025-01-27'); // Monday
  expect(isBusinessDay(monday)).toBe(true);
});

it('should return false on weekends', () => {
  const saturday = new Date('2025-01-25'); // Saturday
  expect(isBusinessDay(saturday)).toBe(false);
});
```

**Mocking Time in React:**

```tsx
// Component that uses time
export const ExpirationTimer: React.FC<{ expiresAt: Date }> = ({ expiresAt }) => {
  const [timeLeft, setTimeLeft] = useState('');
  
  useEffect(() => {
    const updateTimer = () => {
      const now = Date.now();
      const diff = expiresAt.getTime() - now;
      setTimeLeft(diff > 0 ? `${Math.floor(diff / 1000)}s` : 'Expired');
    };
    
    updateTimer();
    const interval = setInterval(updateTimer, 1000);
    return () => clearInterval(interval);
  }, [expiresAt]);
  
  return <div>Time left: {timeLeft}</div>;
};

// ✅ GOOD: Mock time for deterministic tests
describe('ExpirationTimer', () => {
  beforeEach(() => {
    jest.useFakeTimers();
    jest.setSystemTime(new Date('2025-01-27T12:00:00Z'));
  });
  
  afterEach(() => {
    jest.useRealTimers();
  });
  
  it('should show time remaining', () => {
    const expiresAt = new Date('2025-01-27T12:01:00Z'); // 60 seconds from now
    render(<ExpirationTimer expiresAt={expiresAt} />);
    
    expect(screen.getByText('Time left: 60s')).toBeInTheDocument();
  });
  
  it('should update every second', () => {
    const expiresAt = new Date('2025-01-27T12:00:30Z'); // 30 seconds from now
    render(<ExpirationTimer expiresAt={expiresAt} />);
    
    expect(screen.getByText('Time left: 30s')).toBeInTheDocument();
    
    act(() => {
      jest.advanceTimersByTime(5000); // Advance 5 seconds
    });
    
    expect(screen.getByText('Time left: 25s')).toBeInTheDocument();
  });
});
```

**Handling Random Values:**

```typescript
// ❌ BAD: Non-deterministic random values
export function generateId(): string {
  return `id_${Math.random().toString(36).substr(2, 9)}`;
}

it('should generate unique ids', () => {
  const id = generateId();
  expect(id).toBe('id_???'); // Can't predict the value
});

// ✅ GOOD: Inject random generator
export function generateId(random: () => number = Math.random): string {
  return `id_${random().toString(36).substr(2, 9)}`;
}

it('should generate id with provided random function', () => {
  const mockRandom = jest.fn(() => 0.5);
  const id = generateId(mockRandom);
  
  expect(mockRandom).toHaveBeenCalled();
  expect(id).toBe('id_k0000000'); // Predictable with fixed random
});

// Alternative: Mock Math.random
it('should generate id', () => {
  jest.spyOn(Math, 'random').mockReturnValue(0.5);
  const id = generateId();
  
  expect(id).toBe('id_k0000000');
  
  (Math.random as jest.Mock).mockRestore();
});
```

### 4. Fast Tests

Unit tests should run in milliseconds:

```typescript
// ❌ BAD: Slow test with real delays
it('should retry failed requests', async () => {
  const result = await retryWithBackoff(() => fetchData(), 3);
  expect(result).toBeDefined();
  // Takes seconds to complete due to real delays
}, 10000);

// ✅ GOOD: Fast test with mocked delays
it('should retry failed requests', async () => {
  jest.useFakeTimers();
  
  let attempt = 0;
  const mockFetch = jest.fn(() => {
    attempt++;
    if (attempt < 3) return Promise.reject(new Error('Failed'));
    return Promise.resolve({ data: 'success' });
  });
  
  const promise = retryWithBackoff(mockFetch, 3);
  
  // Fast-forward through delays
  await jest.runAllTimersAsync();
  
  const result = await promise;
  expect(result).toEqual({ data: 'success' });
  expect(mockFetch).toHaveBeenCalledTimes(3);
  
  jest.useRealTimers();
});
```

**Angular Testing with Faked Async:**

```typescript
@Component({
  selector: 'app-debounced-search',
  template: `
    <input (input)="onSearch($event)" />
    <div *ngIf="results">{{ results }}</div>
  `
})
export class DebouncedSearchComponent {
  results: string = '';
  private searchSubject = new Subject<string>();
  
  constructor() {
    this.searchSubject
      .pipe(debounceTime(500))
      .subscribe(query => {
        this.results = `Results for: ${query}`;
      });
  }
  
  onSearch(event: Event): void {
    const query = (event.target as HTMLInputElement).value;
    this.searchSubject.next(query);
  }
}

// ✅ GOOD: Fast test with fakeAsync
describe('DebouncedSearchComponent', () => {
  let component: DebouncedSearchComponent;
  let fixture: ComponentFixture<DebouncedSearchComponent>;
  
  beforeEach(() => {
    TestBed.configureTestingModule({
      declarations: [DebouncedSearchComponent]
    });
    
    fixture = TestBed.createComponent(DebouncedSearchComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });
  
  it('should debounce search input', fakeAsync(() => {
    const input = fixture.nativeElement.querySelector('input');
    
    // Type multiple times quickly
    input.value = 'a';
    input.dispatchEvent(new Event('input'));
    
    tick(100);
    input.value = 'ab';
    input.dispatchEvent(new Event('input'));
    
    tick(100);
    input.value = 'abc';
    input.dispatchEvent(new Event('input'));
    
    // Results not shown yet
    fixture.detectChanges();
    expect(component.results).toBe('');
    
    // Fast-forward past debounce time
    tick(500);
    fixture.detectChanges();
    
    expect(component.results).toBe('Results for: abc');
  }));
});
```

## Test Structure and Organization

### Describe Blocks for Grouping

```typescript
// ✅ GOOD: Logical grouping with describe blocks
describe('UserValidator', () => {
  describe('validateEmail', () => {
    it('should accept valid email addresses', () => {
      expect(validateEmail('user@example.com')).toBe(true);
    });
    
    it('should reject emails without @', () => {
      expect(validateEmail('userexample.com')).toBe(false);
    });
    
    it('should reject emails without domain', () => {
      expect(validateEmail('user@')).toBe(false);
    });
  });
  
  describe('validatePassword', () => {
    it('should require minimum length', () => {
      expect(validatePassword('abc')).toBe(false);
      expect(validatePassword('abcd1234')).toBe(true);
    });
    
    it('should require at least one number', () => {
      expect(validatePassword('abcdefgh')).toBe(false);
      expect(validatePassword('abcdefg1')).toBe(true);
    });
  });
});
```

### Setup and Teardown

```typescript
describe('DatabaseService', () => {
  let service: DatabaseService;
  let connection: MockConnection;
  
  // Runs once before all tests in this describe block
  beforeAll(() => {
    connection = createMockConnection();
  });
  
  // Runs before each test
  beforeEach(() => {
    service = new DatabaseService(connection);
    connection.clear(); // Reset state
  });
  
  // Runs after each test
  afterEach(() => {
    jest.clearAllMocks();
  });
  
  // Runs once after all tests
  afterAll(() => {
    connection.close();
  });
  
  it('should insert records', async () => {
    await service.insert('users', { name: 'John' });
    expect(connection.records.users).toHaveLength(1);
  });
  
  it('should query records', async () => {
    await service.insert('users', { name: 'John' });
    const results = await service.query('users', { name: 'John' });
    expect(results).toHaveLength(1);
  });
});
```

## Single Responsibility Per Test

```typescript
// ❌ BAD: Testing multiple things in one test
it('should handle user registration', async () => {
  const user = await registerUser({ email: 'test@example.com', password: 'pass123' });
  expect(user.id).toBeDefined();
  expect(user.email).toBe('test@example.com');
  expect(user.createdAt).toBeInstanceOf(Date);
  
  const foundUser = await findUserById(user.id);
  expect(foundUser).toBeDefined();
  
  const loginResult = await login('test@example.com', 'pass123');
  expect(loginResult.success).toBe(true);
});

// ✅ GOOD: Each test has a single responsibility
describe('User Registration', () => {
  it('should create user with generated id', async () => {
    const user = await registerUser({ email: 'test@example.com', password: 'pass123' });
    expect(user.id).toBeDefined();
  });
  
  it('should store email correctly', async () => {
    const user = await registerUser({ email: 'test@example.com', password: 'pass123' });
    expect(user.email).toBe('test@example.com');
  });
  
  it('should set creation timestamp', async () => {
    const user = await registerUser({ email: 'test@example.com', password: 'pass123' });
    expect(user.createdAt).toBeInstanceOf(Date);
  });
});

describe('User Retrieval', () => {
  it('should find user by id', async () => {
    const user = await registerUser({ email: 'test@example.com', password: 'pass123' });
    const foundUser = await findUserById(user.id);
    expect(foundUser).toBeDefined();
  });
});

describe('User Authentication', () => {
  it('should login with valid credentials', async () => {
    await registerUser({ email: 'test@example.com', password: 'pass123' });
    const result = await login('test@example.com', 'pass123');
    expect(result.success).toBe(true);
  });
});
```

## Test Naming Conventions

```typescript
// ✅ GOOD: Descriptive test names
describe('calculateShipping', () => {
  it('should return 0 for orders over $100', () => {});
  it('should return flat rate for domestic orders', () => {});
  it('should calculate weight-based rate for international orders', () => {});
  it('should throw error for negative weights', () => {});
});

// Alternative: Given-When-Then style
describe('calculateShipping', () => {
  it('given order over $100, when calculating shipping, then returns 0', () => {});
  it('given domestic order, when calculating shipping, then returns flat rate', () => {});
});

// ❌ BAD: Vague test names
describe('calculateShipping', () => {
  it('should work correctly', () => {});
  it('test case 1', () => {});
  it('should calculate', () => {});
});
```

## Testing Pure Functions

Pure functions are easiest to test:

```typescript
// Pure function - same input always produces same output
export function calculateTax(amount: number, rate: number): number {
  return amount * rate;
}

describe('calculateTax', () => {
  it('should calculate tax correctly', () => {
    expect(calculateTax(100, 0.1)).toBe(10);
    expect(calculateTax(200, 0.15)).toBe(30);
    expect(calculateTax(0, 0.1)).toBe(0);
  });
  
  it('should handle decimal amounts', () => {
    expect(calculateTax(99.99, 0.08)).toBeCloseTo(8.00, 2);
  });
});

// More complex pure function
export function formatCurrency(amount: number, currency: string = 'USD'): string {
  const formatter = new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency
  });
  return formatter.format(amount);
}

describe('formatCurrency', () => {
  it('should format USD correctly', () => {
    expect(formatCurrency(1234.56)).toBe('$1,234.56');
  });
  
  it('should format EUR correctly', () => {
    expect(formatCurrency(1234.56, 'EUR')).toBe('€1,234.56');
  });
  
  it('should handle zero', () => {
    expect(formatCurrency(0)).toBe('$0.00');
  });
});
```

## Testing with Dependencies

### Constructor Injection (React)

```tsx
// Service with dependencies
export class ApiService {
  constructor(private httpClient: HttpClient) {}
  
  async getUsers(): Promise<User[]> {
    return this.httpClient.get('/api/users');
  }
}

// Component using service
export const UserList: React.FC<{ apiService: ApiService }> = ({ apiService }) => {
  const [users, setUsers] = useState<User[]>([]);
  
  useEffect(() => {
    apiService.getUsers().then(setUsers);
  }, [apiService]);
  
  return (
    <ul>
      {users.map(user => <li key={user.id}>{user.name}</li>)}
    </ul>
  );
};

// Test with mock service
describe('UserList', () => {
  it('should render users from API', async () => {
    // Arrange
    const mockUsers: User[] = [
      { id: '1', name: 'John' },
      { id: '2', name: 'Jane' }
    ];
    
    const mockApiService = {
      getUsers: jest.fn().mockResolvedValue(mockUsers)
    } as any;
    
    // Act
    render(<UserList apiService={mockApiService} />);
    
    // Assert
    await waitFor(() => {
      expect(screen.getByText('John')).toBeInTheDocument();
      expect(screen.getByText('Jane')).toBeInTheDocument();
    });
  });
});
```

### Angular Dependency Injection

```typescript
@Injectable()
export class UserService {
  constructor(private http: HttpClient) {}
  
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>('/api/users');
  }
}

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
  
  it('should fetch users', () => {
    const mockUsers: User[] = [
      { id: '1', name: 'John' },
      { id: '2', name: 'Jane' }
    ];
    
    service.getUsers().subscribe(users => {
      expect(users).toEqual(mockUsers);
    });
    
    const req = httpMock.expectOne('/api/users');
    expect(req.request.method).toBe('GET');
    req.flush(mockUsers);
  });
  
  afterEach(() => {
    httpMock.verify();
  });
});
```

## Testing Edge Cases and Error Conditions

```typescript
export function divide(a: number, b: number): number {
  if (b === 0) {
    throw new Error('Division by zero');
  }
  return a / b;
}

describe('divide', () => {
  // Happy path
  it('should divide two positive numbers', () => {
    expect(divide(10, 2)).toBe(5);
  });
  
  // Edge cases
  it('should handle zero dividend', () => {
    expect(divide(0, 5)).toBe(0);
  });
  
  it('should handle negative numbers', () => {
    expect(divide(-10, 2)).toBe(-5);
    expect(divide(10, -2)).toBe(-5);
    expect(divide(-10, -2)).toBe(5);
  });
  
  it('should handle decimal results', () => {
    expect(divide(5, 2)).toBe(2.5);
  });
  
  // Error conditions
  it('should throw error on division by zero', () => {
    expect(() => divide(10, 0)).toThrow('Division by zero');
  });
});

// React component error boundary testing
export class ErrorBoundary extends React.Component<
  { children: React.ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false };
  
  static getDerivedStateFromError() {
    return { hasError: true };
  }
  
  render() {
    if (this.state.hasError) {
      return <div>Something went wrong</div>;
    }
    return this.props.children;
  }
}

const ProblematicComponent = () => {
  throw new Error('Test error');
};

describe('ErrorBoundary', () => {
  it('should catch and display error', () => {
    // Suppress console.error for this test
    const spy = jest.spyOn(console, 'error').mockImplementation();
    
    render(
      <ErrorBoundary>
        <ProblematicComponent />
      </ErrorBoundary>
    );
    
    expect(screen.getByText('Something went wrong')).toBeInTheDocument();
    spy.mockRestore();
  });
});
```

## Parameterized Tests

```typescript
// Using test.each for multiple similar tests
describe('isPalindrome', () => {
  test.each([
    ['racecar', true],
    ['hello', false],
    ['A man a plan a canal Panama', true],
    ['', true],
    ['a', true]
  ])('isPalindrome("%s") should return %s', (input, expected) => {
    expect(isPalindrome(input)).toBe(expected);
  });
});

// Object-based parameterized tests
describe('validateAge', () => {
  test.each([
    { age: 0, expected: false, description: 'zero age' },
    { age: -5, expected: false, description: 'negative age' },
    { age: 18, expected: true, description: 'legal age' },
    { age: 150, expected: false, description: 'unrealistic age' }
  ])('should handle $description', ({ age, expected }) => {
    expect(validateAge(age)).toBe(expected);
  });
});
```

## Testing Asynchronous Code

```typescript
// Promise-based async function
export async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  if (!response.ok) {
    throw new Error('User not found');
  }
  return response.json();
}

describe('fetchUser', () => {
  beforeEach(() => {
    global.fetch = jest.fn();
  });
  
  it('should fetch and return user', async () => {
    const mockUser: User = { id: '123', name: 'John' };
    (global.fetch as jest.Mock).mockResolvedValue({
      ok: true,
      json: async () => mockUser
    });
    
    const user = await fetchUser('123');
    
    expect(user).toEqual(mockUser);
    expect(global.fetch).toHaveBeenCalledWith('/api/users/123');
  });
  
  it('should throw error when user not found', async () => {
    (global.fetch as jest.Mock).mockResolvedValue({
      ok: false
    });
    
    await expect(fetchUser('999')).rejects.toThrow('User not found');
  });
});

// Callback-based async code
export function debounce<T extends (...args: any[]) => void>(
  func: T,
  delay: number
): T {
  let timeoutId: NodeJS.Timeout;
  
  return ((...args: any[]) => {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => func(...args), delay);
  }) as T;
}

describe('debounce', () => {
  beforeEach(() => {
    jest.useFakeTimers();
  });
  
  afterEach(() => {
    jest.useRealTimers();
  });
  
  it('should delay function execution', () => {
    const mockFn = jest.fn();
    const debouncedFn = debounce(mockFn, 1000);
    
    debouncedFn();
    expect(mockFn).not.toHaveBeenCalled();
    
    jest.advanceTimersByTime(999);
    expect(mockFn).not.toHaveBeenCalled();
    
    jest.advanceTimersByTime(1);
    expect(mockFn).toHaveBeenCalledTimes(1);
  });
  
  it('should cancel previous calls', () => {
    const mockFn = jest.fn();
    const debouncedFn = debounce(mockFn, 1000);
    
    debouncedFn();
    jest.advanceTimersByTime(500);
    debouncedFn();
    jest.advanceTimersByTime(500);
    
    expect(mockFn).not.toHaveBeenCalled();
    
    jest.advanceTimersByTime(500);
    expect(mockFn).toHaveBeenCalledTimes(1);
  });
});
```

## Common Mistakes

### 1. Testing Implementation Details

```typescript
// ❌ BAD: Testing internal state
it('should update internal counter', () => {
  const component = new Counter();
  component.increment();
  expect(component['_count']).toBe(1); // Testing private field
});

// ✅ GOOD: Testing public behavior
it('should increment count', () => {
  const component = new Counter();
  component.increment();
  expect(component.getCount()).toBe(1);
});
```

### 2. Brittle Tests with Too Much Specificity

```typescript
// ❌ BAD: Overly specific assertion
it('should render user card', () => {
  render(<UserCard user={{ name: 'John', age: 30 }} />);
  expect(container.innerHTML).toBe('<div class="card"><h2>John</h2><p>30</p></div>');
});

// ✅ GOOD: Test behavior, not exact HTML
it('should render user card', () => {
  render(<UserCard user={{ name: 'John', age: 30 }} />);
  expect(screen.getByRole('heading', { name: 'John' })).toBeInTheDocument();
  expect(screen.getByText('30')).toBeInTheDocument();
});
```

### 3. Not Cleaning Up Side Effects

```typescript
// ❌ BAD: Side effects leak between tests
describe('LocalStorage', () => {
  it('should save data', () => {
    localStorage.setItem('key', 'value');
    expect(localStorage.getItem('key')).toBe('value');
  });
  
  it('should return null for missing keys', () => {
    // Fails because previous test left data
    expect(localStorage.getItem('key')).toBeNull();
  });
});

// ✅ GOOD: Clean up after each test
describe('LocalStorage', () => {
  afterEach(() => {
    localStorage.clear();
  });
  
  it('should save data', () => {
    localStorage.setItem('key', 'value');
    expect(localStorage.getItem('key')).toBe('value');
  });
  
  it('should return null for missing keys', () => {
    expect(localStorage.getItem('key')).toBeNull();
  });
});
```

## Best Practices

1. **Follow AAA Pattern**: Clearly separate Arrange, Act, and Assert sections
2. **One Assertion Per Concept**: Test one logical concept per test
3. **Make Tests Independent**: Each test should run in isolation
4. **Use Descriptive Names**: Test names should describe what's being tested
5. **Test Behavior, Not Implementation**: Focus on public API, not internals
6. **Keep Tests Fast**: Unit tests should run in milliseconds
7. **Avoid Logic in Tests**: Tests should be simple and linear
8. **Use Setup/Teardown**: DRY principle applies to test code too
9. **Test Edge Cases**: Cover boundary conditions and error paths
10. **Make Tests Deterministic**: No random values, no time dependencies

## When to Use Unit Tests

**Use unit tests when:**
- Testing pure functions and utilities
- Testing business logic in isolation
- Testing individual React components
- Testing Angular services and pipes
- Testing class methods
- You need fast feedback during development

**Don't rely only on unit tests for:**
- Testing component integration
- Testing data flow between components
- Testing API integrations
- Testing user workflows
- Testing browser-specific behavior

## Interview Questions

1. **What is the AAA pattern in unit testing?**
   - Arrange: Set up test data and dependencies
   - Act: Execute the code under test
   - Assert: Verify the expected outcome

2. **Why should unit tests be deterministic?**
   - Tests must produce the same result every time they run. Non-deterministic tests (random values, time-dependent, order-dependent) lead to flaky tests that reduce confidence.

3. **What's the difference between beforeEach and beforeAll?**
   - beforeEach runs before every test (use for fresh state)
   - beforeAll runs once before all tests (use for expensive setup)

4. **How do you test asynchronous code?**
   - Use async/await in test functions
   - Return promises from tests
   - Use done callback (older style)
   - Use fake timers for time-based async code

5. **What makes a good unit test?**
   - Fast, isolated, deterministic, tests one thing, clear name, tests behavior not implementation

## Key Takeaways

1. Unit tests verify individual units of code in isolation
2. AAA pattern provides clear test structure
3. Tests must be independent and deterministic
4. Fast tests enable rapid feedback loops
5. Test behavior, not implementation details
6. Clean up side effects between tests
7. Use mocks/stubs to isolate units from dependencies
8. Edge cases and error conditions are as important as happy paths
9. Good test names serve as documentation
10. Unit tests form the base of the testing pyramid

## Resources

- **Testing Library Documentation**: https://testing-library.com/docs/
- **Jest Documentation**: https://jestjs.io/docs/getting-started
- **Jasmine Documentation**: https://jasmine.github.io/
- **Angular Testing Guide**: https://angular.io/guide/testing
- **Kent C. Dodds - Testing Blog**: https://kentcdodds.com/blog?q=testing
- **"Growing Object-Oriented Software, Guided by Tests"** by Steve Freeman
- **"Unit Testing Principles, Practices, and Patterns"** by Vladimir Khorikov
