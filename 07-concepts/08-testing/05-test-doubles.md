# Test Doubles: Mocks, Stubs, Spies, and Fakes

## The Idea

**In plain English:** When you test your code, some parts depend on things you don't control — like a live website or a real database. A test double is a stand-in replacement for that real thing, so your tests can run quickly and reliably without depending on the outside world.

**Real-world analogy:** Imagine a movie director filming a car chase. Instead of using the lead actor (who could get hurt), they use a stunt double — someone who looks and acts like the actor for that scene. The director can then check whether the scene works without risking the real actor.

- The lead actor = the real dependency (e.g., a live API or database)
- The stunt double = the test double (e.g., a mock, stub, or fake)
- The director checking the scene = your automated test verifying the code works

---

## Table of Contents
- [Introduction](#introduction)
- [What are Test Doubles](#what-are-test-doubles)
- [Types of Test Doubles](#types-of-test-doubles)
- [Mocks](#mocks)
- [Stubs](#stubs)
- [Spies](#spies)
- [Fakes](#fakes)
- [When to Use Each](#when-to-use-each)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Test doubles are objects that stand in for real dependencies during testing. Just as stunt doubles replace actors in dangerous scenes, test doubles replace real objects to make testing easier, faster, and more reliable. Understanding the different types and when to use each is crucial for writing effective tests.

```
Test Double Hierarchy
=====================

Real Object (Production)
         |
    Test Double (Testing)
         |
    _____|_____________________
    |      |        |          |
  Dummy  Stub     Spy        Mock
           |                   |
         Fake            (Behavior Verification)
    (Functional)
```

The term "test double" comes from Gerard Meszaros's book "xUnit Test Patterns" and encompasses all types of pretend objects used in testing.

## What are Test Doubles

### The Problem They Solve

```typescript
// Real API call (problems in testing)
class UserService {
  async getUser(id: string): Promise<User> {
    const response = await fetch(`https://api.example.com/users/${id}`);
    return response.json();
  }
}

/*
Problems:
❌ Requires network access
❌ Slow (hundreds of milliseconds)
❌ External dependency (API might be down)
❌ Costs money (API calls)
❌ Hard to test error cases
❌ Non-deterministic (data changes)
*/

// Test double solution
class MockUserService {
  async getUser(id: string): Promise<User> {
    return { id, name: 'Test User', email: 'test@example.com' };
  }
}

/*
Benefits:
✓ No network needed
✓ Fast (microseconds)
✓ Always available
✓ Free
✓ Can simulate any scenario
✓ Deterministic
*/
```

## Types of Test Doubles

### Quick Reference

```typescript
/*
1. DUMMY
   - Passed but never used
   - Fills parameter lists
   - Example: null, undefined, empty object

2. STUB
   - Returns predetermined values
   - No behavior verification
   - Example: Mock API returning fixed data

3. SPY
   - Records information about calls
   - Allows verification after the fact
   - Example: Track how many times called

4. MOCK
   - Pre-programmed with expectations
   - Verifies behavior happened correctly
   - Example: Expect function called with specific args

5. FAKE
   - Working implementation
   - Simplified for testing
   - Example: In-memory database
*/
```

## Mocks

### Definition

Mocks are objects pre-programmed with expectations about how they should be called. They verify that expected interactions occur.

### Examples

```typescript
// React: Mocking with Jest
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { UserProfile } from './UserProfile';
import { analytics } from './analytics';

// Mock the analytics module
jest.mock('./analytics');

describe('UserProfile', () => {
  it('tracks profile view event', () => {
    const mockTrack = jest.fn();
    (analytics.track as jest.Mock) = mockTrack;

    render(<UserProfile userId="123" />);

    // Verify the mock was called correctly
    expect(mockTrack).toHaveBeenCalledWith('profile_viewed', {
      userId: '123'
    });
    expect(mockTrack).toHaveBeenCalledTimes(1);
  });

  it('tracks button click with correct data', async () => {
    const mockTrack = jest.fn();
    (analytics.track as jest.Mock) = mockTrack;

    render(<UserProfile userId="123" />);
    
    await userEvent.click(screen.getByRole('button', { name: 'Follow' }));

    expect(mockTrack).toHaveBeenCalledWith('follow_clicked', {
      userId: '123',
      timestamp: expect.any(Number)
    });
  });

  it('fails if analytics not called', () => {
    const mockTrack = jest.fn();
    (analytics.track as jest.Mock) = mockTrack;

    render(<UserProfile userId="123" />);

    // This would fail if track() wasn't called
    expect(mockTrack).toHaveBeenCalled();
  });
});
```

```typescript
// Angular: Mocking services with Jasmine
import { TestBed } from '@angular/core/testing';
import { UserComponent } from './user.component';
import { UserService } from './user.service';
import { of } from 'rxjs';

describe('UserComponent', () => {
  let component: UserComponent;
  let userService: jasmine.SpyObj<UserService>;

  beforeEach(() => {
    // Create mock service
    const userServiceMock = jasmine.createSpyObj('UserService', [
      'getUser',
      'updateUser',
      'deleteUser'
    ]);

    TestBed.configureTestingModule({
      declarations: [UserComponent],
      providers: [
        { provide: UserService, useValue: userServiceMock }
      ]
    });

    component = TestBed.createComponent(UserComponent).componentInstance;
    userService = TestBed.inject(UserService) as jasmine.SpyObj<UserService>;
  });

  it('calls updateUser with correct data', () => {
    const userData = { id: '1', name: 'Updated Name' };
    userService.updateUser.and.returnValue(of(userData));

    component.updateUser(userData);

    expect(userService.updateUser).toHaveBeenCalledWith(userData);
    expect(userService.updateUser).toHaveBeenCalledTimes(1);
  });

  it('calls deleteUser and handles response', () => {
    userService.deleteUser.and.returnValue(of({ success: true }));

    component.deleteUser('123');

    expect(userService.deleteUser).toHaveBeenCalledWith('123');
  });
});
```

### Advanced Mock Patterns

```typescript
// React: Mock with implementation
const mockFetch = jest.fn().mockImplementation((url: string) => {
  if (url.includes('/users')) {
    return Promise.resolve({
      json: () => Promise.resolve({ id: '1', name: 'John' })
    });
  }
  if (url.includes('/posts')) {
    return Promise.resolve({
      json: () => Promise.resolve([{ id: '1', title: 'Post 1' }])
    });
  }
  return Promise.reject(new Error('Not found'));
});

global.fetch = mockFetch as any;

// Mock with different return values
const mockGetUser = jest.fn()
  .mockResolvedValueOnce({ id: '1', name: 'First Call' })
  .mockResolvedValueOnce({ id: '2', name: 'Second Call' })
  .mockRejectedValueOnce(new Error('Third Call Fails'));

// Verify call order
expect(mockFn).toHaveBeenNthCalledWith(1, 'first arg');
expect(mockFn).toHaveBeenNthCalledWith(2, 'second arg');

// Verify last call
expect(mockFn).toHaveBeenLastCalledWith('last arg');
```

## Stubs

### Definition

Stubs provide predetermined responses to calls. They don't verify interactions, just return data.

### Examples

```typescript
// React: Stubbing API calls with MSW
import { rest } from 'msw';
import { setupServer } from 'msw/node';
import { render, screen, waitFor } from '@testing-library/react';
import { UserList } from './UserList';

// Setup MSW server with stubbed responses
const server = setupServer(
  rest.get('/api/users', (req, res, ctx) => {
    // Stub: Always returns same data
    return res(
      ctx.json([
        { id: '1', name: 'Alice', email: 'alice@example.com' },
        { id: '2', name: 'Bob', email: 'bob@example.com' }
      ])
    );
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('UserList', () => {
  it('displays users from API', async () => {
    render(<UserList />);

    await waitFor(() => {
      expect(screen.getByText('Alice')).toBeInTheDocument();
      expect(screen.getByText('Bob')).toBeInTheDocument();
    });
  });

  it('handles different stub data', async () => {
    // Override stub for this test
    server.use(
      rest.get('/api/users', (req, res, ctx) => {
        return res(ctx.json([
          { id: '3', name: 'Charlie', email: 'charlie@example.com' }
        ]));
      })
    );

    render(<UserList />);

    await waitFor(() => {
      expect(screen.getByText('Charlie')).toBeInTheDocument();
    });
  });

  it('handles error state', async () => {
    // Stub error response
    server.use(
      rest.get('/api/users', (req, res, ctx) => {
        return res(ctx.status(500), ctx.json({ error: 'Server error' }));
      })
    );

    render(<UserList />);

    expect(await screen.findByText('Error loading users')).toBeInTheDocument();
  });
});
```

```typescript
// Stubbing localStorage
const localStorageStub = (() => {
  let store: Record<string, string> = {};

  return {
    getItem: (key: string) => store[key] || null,
    setItem: (key: string, value: string) => {
      store[key] = value;
    },
    removeItem: (key: string) => {
      delete store[key];
    },
    clear: () => {
      store = {};
    }
  };
})();

Object.defineProperty(window, 'localStorage', {
  value: localStorageStub
});

// Now you can test code that uses localStorage
it('stores user preferences', () => {
  const component = new PreferencesComponent();
  component.savePreference('theme', 'dark');

  expect(localStorage.getItem('theme')).toBe('dark');
});
```

## Spies

### Definition

Spies record information about how they were called, allowing you to verify interactions after they happen.

### Examples

```typescript
// React: Using Jest spies
describe('Button Component', () => {
  it('calls onClick handler when clicked', async () => {
    const handleClick = jest.fn();
    render(<Button onClick={handleClick}>Click me</Button>);

    await userEvent.click(screen.getByRole('button'));

    // Spy recorded the call
    expect(handleClick).toHaveBeenCalled();
    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it('passes correct event to handler', async () => {
    const handleClick = jest.fn();
    render(<Button onClick={handleClick}>Click me</Button>);

    await userEvent.click(screen.getByRole('button'));

    // Spy recorded the arguments
    expect(handleClick).toHaveBeenCalledWith(
      expect.objectContaining({
        type: 'click',
        target: expect.any(HTMLButtonElement)
      })
    );
  });
});

// Spying on existing methods
const user = {
  name: 'John',
  greet() {
    return `Hello, ${this.name}`;
  }
};

const greetSpy = jest.spyOn(user, 'greet');

user.greet();

expect(greetSpy).toHaveBeenCalled();
expect(greetSpy).toHaveReturnedWith('Hello, John');

// Spy still calls real method unless mocked
greetSpy.mockImplementation(() => 'Mocked greeting');
```

```typescript
// Angular: Spying with Jasmine
describe('DataService', () => {
  let service: DataService;
  let httpClient: HttpClient;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [DataService]
    });

    service = TestBed.inject(DataService);
    httpClient = TestBed.inject(HttpClient);
  });

  it('calls HTTP GET with correct URL', () => {
    const spy = spyOn(httpClient, 'get').and.returnValue(of({ data: 'test' }));

    service.getData('123');

    expect(spy).toHaveBeenCalledWith('/api/data/123');
  });

  it('transforms response correctly', () => {
    spyOn(httpClient, 'get').and.returnValue(
      of({ raw: 'data' })
    );

    service.getData('123').subscribe(result => {
      expect(result).toEqual({ transformed: 'data' });
    });
  });
});
```

### Spy vs Mock

```typescript
// SPY: Verifies calls after they happen
const spy = jest.fn();
doSomething(spy);
expect(spy).toHaveBeenCalled(); // Check after the fact

// MOCK: Pre-programmed expectations
const mock = jest.fn();
expect(mock).not.toHaveBeenCalled(); // Expectation set upfront
doSomething(mock);
// Test fails if expectations not met
```

## Fakes

### Definition

Fakes are working implementations, but simplified for testing (e.g., in-memory database instead of real database).

### Examples

```typescript
// Fake implementation of a database
class FakeUserDatabase {
  private users: Map<string, User> = new Map();

  async create(user: User): Promise<User> {
    this.users.set(user.id, user);
    return user;
  }

  async findById(id: string): Promise<User | null> {
    return this.users.get(id) || null;
  }

  async update(id: string, data: Partial<User>): Promise<User | null> {
    const user = this.users.get(id);
    if (!user) return null;

    const updated = { ...user, ...data };
    this.users.set(id, updated);
    return updated;
  }

  async delete(id: string): Promise<boolean> {
    return this.users.delete(id);
  }

  async findAll(): Promise<User[]> {
    return Array.from(this.users.values());
  }

  // Test helpers
  clear() {
    this.users.clear();
  }

  size() {
    return this.users.size;
  }
}

// Usage in tests
describe('UserService with Fake Database', () => {
  let service: UserService;
  let fakeDb: FakeUserDatabase;

  beforeEach(() => {
    fakeDb = new FakeUserDatabase();
    service = new UserService(fakeDb);
  });

  afterEach(() => {
    fakeDb.clear();
  });

  it('creates user successfully', async () => {
    const user = await service.createUser({
      name: 'John',
      email: 'john@example.com'
    });

    expect(user.id).toBeDefined();
    expect(user.name).toBe('John');
    expect(fakeDb.size()).toBe(1);
  });

  it('finds user by id', async () => {
    const created = await service.createUser({
      name: 'John',
      email: 'john@example.com'
    });

    const found = await service.getUserById(created.id);

    expect(found).toEqual(created);
  });

  it('updates user', async () => {
    const user = await service.createUser({
      name: 'John',
      email: 'john@example.com'
    });

    await service.updateUser(user.id, { name: 'John Doe' });

    const updated = await service.getUserById(user.id);
    expect(updated?.name).toBe('John Doe');
  });
});
```

```typescript
// Fake HTTP client
class FakeHttpClient {
  private responses: Map<string, any> = new Map();
  private callHistory: Array<{ url: string; options: any }> = [];

  setResponse(url: string, response: any) {
    this.responses.set(url, response);
  }

  async get(url: string, options?: any): Promise<any> {
    this.callHistory.push({ url, options });

    if (this.responses.has(url)) {
      return this.responses.get(url);
    }

    throw new Error(`No response configured for ${url}`);
  }

  async post(url: string, data: any, options?: any): Promise<any> {
    this.callHistory.push({ url, options: { ...options, data } });

    if (this.responses.has(url)) {
      return this.responses.get(url);
    }

    return { success: true, data };
  }

  getCallHistory() {
    return this.callHistory;
  }

  clear() {
    this.responses.clear();
    this.callHistory = [];
  }
}

// Usage
const fakeHttp = new FakeHttpClient();
fakeHttp.setResponse('/api/users', [{ id: '1', name: 'John' }]);

const users = await fakeHttp.get('/api/users');
expect(users).toHaveLength(1);
```

## When to Use Each

### Decision Matrix

```typescript
/*
Use DUMMY when:
✓ Parameter required but not used
✓ Filling interface requirements
✓ Callback never called

Use STUB when:
✓ Need predetermined responses
✓ Testing different data scenarios
✓ No need to verify calls
✓ Simulating API responses

Use SPY when:
✓ Need to verify function was called
✓ Want to see call arguments
✓ Need call count
✓ Want real implementation to run

Use MOCK when:
✓ Behavior verification is important
✓ Need to ensure specific interactions
✓ Testing side effects
✓ Verifying integration points

Use FAKE when:
✓ Need working implementation
✓ Real dependency too complex/slow
✓ Testing business logic
✓ Multiple tests need same setup
*/
```

### Practical Examples

```typescript
// DUMMY: Not used, just fills parameter
function processUser(user: User, logger: Logger) {
  // logger never used in this code path
  return user.name.toUpperCase();
}

test('processes user name', () => {
  const result = processUser(
    { id: '1', name: 'john' },
    null as any // Dummy - never used
  );
  expect(result).toBe('JOHN');
});

// STUB: Returns predetermined data
test('displays user data', async () => {
  server.use(
    rest.get('/api/user', (req, res, ctx) => {
      return res(ctx.json({ name: 'John' })); // Stub
    })
  );

  render(<UserProfile />);
  expect(await screen.findByText('John')).toBeInTheDocument();
});

// SPY: Verifies call happened
test('logs error', () => {
  const consoleSpy = jest.spyOn(console, 'error');
  
  causeError();
  
  expect(consoleSpy).toHaveBeenCalledWith('Error occurred');
  consoleSpy.mockRestore();
});

// MOCK: Pre-programmed expectations
test('saves data to database', async () => {
  const mockSave = jest.fn().mockResolvedValue({ id: '1' });
  const db = { save: mockSave };

  await saveUser(db, { name: 'John' });

  expect(mockSave).toHaveBeenCalledWith({ name: 'John' });
});

// FAKE: Working implementation
test('user service operations', async () => {
  const fakeDb = new FakeDatabase();
  const service = new UserService(fakeDb);

  await service.createUser({ name: 'John' });
  const users = await service.getAllUsers();

  expect(users).toHaveLength(1);
});
```

## Common Mistakes

### 1. Over-Mocking

```typescript
// ❌ WRONG: Mocking everything
jest.mock('./utils');
jest.mock('./helpers');
jest.mock('./services');
jest.mock('./components/Button');

// Tests pass but app might be broken

// ✅ CORRECT: Mock only external dependencies
jest.mock('./api'); // External
// Use real utils, helpers, components
```

### 2. Testing Mock Instead of Code

```typescript
// ❌ WRONG: Testing the mock
const mockFn = jest.fn().mockReturnValue('test');
expect(mockFn()).toBe('test'); // This tests Jest, not your code!

// ✅ CORRECT: Test your code with mock
const mockGetUser = jest.fn().mockResolvedValue({ name: 'John' });
const result = await MyComponent({ getUser: mockGetUser });
expect(result).toContain('John'); // Tests your component
```

### 3. Not Resetting Mocks

```typescript
// ❌ WRONG: Mocks accumulate state
const mockFn = jest.fn();

test('first test', () => {
  callFunctionWith(mockFn);
  expect(mockFn).toHaveBeenCalledTimes(1);
});

test('second test', () => {
  callFunctionWith(mockFn);
  expect(mockFn).toHaveBeenCalledTimes(1); // Fails! Count is 2
});

// ✅ CORRECT: Reset between tests
afterEach(() => {
  jest.clearAllMocks();
  // or mockFn.mockClear();
});
```

### 4. Brittle Mocks

```typescript
// ❌ WRONG: Too specific
expect(mockFn).toHaveBeenCalledWith({
  id: '123',
  name: 'John',
  email: 'john@example.com',
  createdAt: '2024-01-15T10:30:00Z', // Will break if time changes!
  metadata: { /* ... */ }
});

// ✅ CORRECT: Focus on important parts
expect(mockFn).toHaveBeenCalledWith(
  expect.objectContaining({
    id: '123',
    name: 'John',
    createdAt: expect.any(String)
  })
);
```

## Best Practices

### 1. Prefer Spies Over Mocks

```typescript
// ✅ GOOD: Spy lets real code run
const spy = jest.spyOn(console, 'log');

myFunction();

expect(spy).toHaveBeenCalled();
spy.mockRestore();
```

### 2. Use Fakes for Complex Dependencies

```typescript
// ✅ GOOD: Fake database for integration tests
class FakeDatabase {
  // Simplified but functional implementation
}

// Use across many tests
const db = new FakeDatabase();
```

### 3. Mock at Module Boundaries

```typescript
// ✅ GOOD: Mock external API
jest.mock('./api');

// ✅ GOOD: Mock third-party library
jest.mock('axios');

// ❌ AVOID: Mocking internal modules excessively
```

### 4. Clear Mock Names

```typescript
// ❌ BAD
const mock1 = jest.fn();
const mock2 = jest.fn();

// ✅ GOOD
const mockSaveUser = jest.fn();
const mockDeleteUser = jest.fn();
```

## Interview Questions

### Junior Level
1. **What is a test double?**
   - Object that replaces real dependency in tests
   - Makes testing easier and faster
   - Examples: mocks, stubs, spies, fakes

2. **What's the difference between a mock and a stub?**
   - Mock: Verifies behavior (how called)
   - Stub: Returns data (what returns)
   - Mock: Expectations upfront
   - Stub: No verification

3. **When would you use a spy?**
   - Need to verify calls
   - Want real implementation to run
   - Recording call information
   - Testing callbacks

### Mid Level
4. **Explain the difference between spy, mock, and fake**
   - Spy: Records calls, real implementation
   - Mock: Pre-programmed expectations
   - Fake: Working simplified implementation
   - Each serves different purposes

5. **When should you create a fake vs using mocks?**
   - Fake: Complex logic, reused across tests
   - Mock: Simple interactions, one-off tests
   - Fake: Integration testing
   - Mock: Unit testing

6. **How do you prevent mocks from making tests brittle?**
   - Use expect.objectContaining()
   - Use expect.any()
   - Focus on important properties
   - Don't over-specify
   - Reset between tests

### Senior Level
7. **Design a test double strategy for a large application**
   - Mock external services (APIs)
   - Fake for databases
   - Spies for analytics/logging
   - Real implementations for business logic
   - Clear boundaries
   - Documentation

8. **How do you balance mocking vs real dependencies?**
   - Mock external only
   - Integration tests use real code
   - Unit tests can use more mocks
   - Consider test confidence
   - Measure maintenance cost
   - Use Testing Trophy approach

9. **What are the trade-offs of using fakes?**
   - Pros: Realistic, reusable, fast
   - Cons: Maintenance overhead, divergence risk
   - Need to keep in sync with real implementation
   - Good for complex dependencies

10. **How would you test code that's hard to mock?**
    - Refactor for testability
    - Dependency injection
    - Create abstractions
    - Use fakes
    - Integration tests
    - Consider architecture changes

## Key Takeaways

1. **Test doubles** replace real dependencies in tests
2. **Mocks** verify behavior and interactions
3. **Stubs** provide predetermined responses
4. **Spies** record call information
5. **Fakes** are simplified working implementations
6. **Mock external dependencies** only (APIs, third-party)
7. **Prefer real implementations** for your own code
8. **Reset mocks** between tests
9. **Don't over-mock** - reduces confidence
10. **Choose the right tool** for each situation

## Resources

### Documentation
- [Jest Mock Functions](https://jestjs.io/docs/mock-functions)
- [Jasmine Spies](https://jasmine.github.io/tutorials/your_first_suite#section-Spies)
- [Sinon.js](https://sinonjs.org/) - Standalone test doubles
- [MSW](https://mswjs.io/) - API mocking

### Articles
- [Test Doubles — Fakes, Mocks and Stubs](https://blog.pragmatists.com/test-doubles-fakes-mocks-and-stubs-1a7491dfa3da)
- [Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html) - Martin Fowler
- [xUnit Test Patterns](http://xunitpatterns.com/) - Gerard Meszaros

### Books
- *xUnit Test Patterns* by Gerard Meszaros
- *Growing Object-Oriented Software, Guided by Tests* by Freeman & Pryce
- *The Art of Unit Testing* by Roy Osherove
