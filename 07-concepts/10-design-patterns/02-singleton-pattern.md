# Singleton Pattern

## Overview

The Singleton pattern ensures that a class has only one instance throughout the application lifecycle and provides a global point of access to that instance. While one of the most well-known design patterns, it's also one of the most controversial due to its potential for misuse and the problems it can introduce.

In frontend development, Singletons are commonly used for services like API clients, configuration managers, logging services, and state stores, but should be used judiciously.

## Core Concepts

### 1. Singleton Structure

```
┌────────────────────────────────────────┐
│          Singleton Class                │
├────────────────────────────────────────┤
│  - static instance: Singleton          │
│  - private constructor()               │
├────────────────────────────────────────┤
│  + static getInstance(): Singleton     │
│  + businessMethod(): void              │
└────────────────────────────────────────┘

Client → getInstance() → returns same instance
```

### 2. Key Characteristics

- **Single Instance**: Only one instance exists
- **Global Access**: Available throughout the application
- **Lazy Initialization**: Created when first needed (optionally)
- **Private Constructor**: Prevents direct instantiation
- **Static Access Point**: Class method to get instance

## Implementation Patterns

### Basic Singleton (TypeScript)

```typescript
// singleton.ts
export class Singleton {
  private static instance: Singleton;
  private data: string[] = [];

  // Private constructor prevents direct instantiation
  private constructor() {
    console.log('Singleton instance created');
  }

  // Static method to get the single instance
  public static getInstance(): Singleton {
    if (!Singleton.instance) {
      Singleton.instance = new Singleton();
    }
    return Singleton.instance;
  }

  // Business methods
  public addData(value: string): void {
    this.data.push(value);
  }

  public getData(): string[] {
    return [...this.data];
  }

  public clearData(): void {
    this.data = [];
  }
}

// Usage
const singleton1 = Singleton.getInstance();
const singleton2 = Singleton.getInstance();

console.log(singleton1 === singleton2); // true

singleton1.addData('test');
console.log(singleton2.getData()); // ['test'] - same instance!
```

### Thread-Safe Singleton (JavaScript)

```typescript
// thread-safe-singleton.ts
export class ThreadSafeSingleton {
  private static instance: ThreadSafeSingleton | null = null;
  private static creating = false;

  private constructor() {}

  public static async getInstance(): Promise<ThreadSafeSingleton> {
    if (ThreadSafeSingleton.instance) {
      return ThreadSafeSingleton.instance;
    }

    // Prevent race conditions
    if (ThreadSafeSingleton.creating) {
      // Wait for creation to complete
      while (ThreadSafeSingleton.creating) {
        await new Promise(resolve => setTimeout(resolve, 10));
      }
      return ThreadSafeSingleton.instance!;
    }

    ThreadSafeSingleton.creating = true;
    
    try {
      // Simulate async initialization
      await new Promise(resolve => setTimeout(resolve, 100));
      ThreadSafeSingleton.instance = new ThreadSafeSingleton();
      return ThreadSafeSingleton.instance;
    } finally {
      ThreadSafeSingleton.creating = false;
    }
  }
}
```

### Configuration Manager Singleton

```typescript
// config-manager.ts
interface AppConfig {
  apiUrl: string;
  apiKey: string;
  environment: 'development' | 'staging' | 'production';
  features: {
    analytics: boolean;
    darkMode: boolean;
    notifications: boolean;
  };
}

export class ConfigManager {
  private static instance: ConfigManager;
  private config: AppConfig | null = null;
  private initialized = false;

  private constructor() {}

  public static getInstance(): ConfigManager {
    if (!ConfigManager.instance) {
      ConfigManager.instance = new ConfigManager();
    }
    return ConfigManager.instance;
  }

  public async initialize(configPath?: string): Promise<void> {
    if (this.initialized) {
      console.warn('ConfigManager already initialized');
      return;
    }

    try {
      // Load configuration from file or API
      const response = await fetch(configPath || '/config.json');
      this.config = await response.json();
      this.initialized = true;
      console.log('Configuration loaded successfully');
    } catch (error) {
      console.error('Failed to load configuration:', error);
      throw error;
    }
  }

  public get<K extends keyof AppConfig>(key: K): AppConfig[K] {
    if (!this.initialized || !this.config) {
      throw new Error('ConfigManager not initialized');
    }
    return this.config[key];
  }

  public getFeatureFlag(feature: keyof AppConfig['features']): boolean {
    return this.get('features')[feature];
  }

  public isProduction(): boolean {
    return this.get('environment') === 'production';
  }

  public reset(): void {
    this.config = null;
    this.initialized = false;
  }
}

// Usage
const config = ConfigManager.getInstance();
await config.initialize();

const apiUrl = config.get('apiUrl');
const analyticsEnabled = config.getFeatureFlag('analytics');

if (config.isProduction()) {
  console.log('Running in production mode');
}
```

### API Client Singleton

```typescript
// api-client.ts
interface RequestConfig {
  method?: 'GET' | 'POST' | 'PUT' | 'DELETE' | 'PATCH';
  headers?: Record<string, string>;
  body?: any;
}

export class APIClient {
  private static instance: APIClient;
  private baseURL: string;
  private defaultHeaders: Record<string, string>;
  private interceptors: {
    request: Array<(config: RequestConfig) => RequestConfig>;
    response: Array<(response: Response) => Response | Promise<Response>>;
  };

  private constructor(baseURL: string) {
    this.baseURL = baseURL;
    this.defaultHeaders = {
      'Content-Type': 'application/json'
    };
    this.interceptors = {
      request: [],
      response: []
    };
  }

  public static getInstance(baseURL?: string): APIClient {
    if (!APIClient.instance) {
      if (!baseURL) {
        throw new Error('Base URL required for first initialization');
      }
      APIClient.instance = new APIClient(baseURL);
    }
    return APIClient.instance;
  }

  public setAuthToken(token: string): void {
    this.defaultHeaders['Authorization'] = `Bearer ${token}`;
  }

  public clearAuthToken(): void {
    delete this.defaultHeaders['Authorization'];
  }

  public addRequestInterceptor(
    interceptor: (config: RequestConfig) => RequestConfig
  ): void {
    this.interceptors.request.push(interceptor);
  }

  public addResponseInterceptor(
    interceptor: (response: Response) => Response | Promise<Response>
  ): void {
    this.interceptors.response.push(interceptor);
  }

  public async request<T>(
    endpoint: string,
    config: RequestConfig = {}
  ): Promise<T> {
    // Apply request interceptors
    let finalConfig = { ...config };
    for (const interceptor of this.interceptors.request) {
      finalConfig = interceptor(finalConfig);
    }

    const url = `${this.baseURL}${endpoint}`;
    const options: RequestInit = {
      method: finalConfig.method || 'GET',
      headers: {
        ...this.defaultHeaders,
        ...finalConfig.headers
      }
    };

    if (finalConfig.body) {
      options.body = JSON.stringify(finalConfig.body);
    }

    let response = await fetch(url, options);

    // Apply response interceptors
    for (const interceptor of this.interceptors.response) {
      response = await interceptor(response);
    }

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    return await response.json();
  }

  public get<T>(endpoint: string): Promise<T> {
    return this.request<T>(endpoint, { method: 'GET' });
  }

  public post<T>(endpoint: string, body: any): Promise<T> {
    return this.request<T>(endpoint, { method: 'POST', body });
  }

  public put<T>(endpoint: string, body: any): Promise<T> {
    return this.request<T>(endpoint, { method: 'PUT', body });
  }

  public delete<T>(endpoint: string): Promise<T> {
    return this.request<T>(endpoint, { method: 'DELETE' });
  }
}

// Usage
const api = APIClient.getInstance('https://api.example.com');

// Add auth token
api.setAuthToken('your-token-here');

// Add request logging interceptor
api.addRequestInterceptor((config) => {
  console.log('Request:', config);
  return config;
});

// Add error handling interceptor
api.addResponseInterceptor(async (response) => {
  if (response.status === 401) {
    // Handle unauthorized
    console.error('Unauthorized - redirecting to login');
  }
  return response;
});

// Make requests
const users = await api.get<User[]>('/users');
const newUser = await api.post<User>('/users', { name: 'John' });
```

### Logger Singleton

```typescript
// logger.ts
type LogLevel = 'debug' | 'info' | 'warn' | 'error';

interface LogEntry {
  level: LogLevel;
  message: string;
  timestamp: Date;
  context?: any;
}

export class Logger {
  private static instance: Logger;
  private logs: LogEntry[] = [];
  private minLevel: LogLevel = 'info';
  private maxLogs = 1000;

  private levelPriority: Record<LogLevel, number> = {
    debug: 0,
    info: 1,
    warn: 2,
    error: 3
  };

  private constructor() {}

  public static getInstance(): Logger {
    if (!Logger.instance) {
      Logger.instance = new Logger();
    }
    return Logger.instance;
  }

  public setMinLevel(level: LogLevel): void {
    this.minLevel = level;
  }

  public setMaxLogs(max: number): void {
    this.maxLogs = max;
  }

  private shouldLog(level: LogLevel): boolean {
    return this.levelPriority[level] >= this.levelPriority[this.minLevel];
  }

  private log(level: LogLevel, message: string, context?: any): void {
    if (!this.shouldLog(level)) {
      return;
    }

    const entry: LogEntry = {
      level,
      message,
      timestamp: new Date(),
      context
    };

    this.logs.push(entry);

    // Trim logs if exceeded max
    if (this.logs.length > this.maxLogs) {
      this.logs.shift();
    }

    // Console output
    const timestamp = entry.timestamp.toISOString();
    const logMessage = `[${timestamp}] ${level.toUpperCase()}: ${message}`;
    
    switch (level) {
      case 'debug':
        console.debug(logMessage, context);
        break;
      case 'info':
        console.info(logMessage, context);
        break;
      case 'warn':
        console.warn(logMessage, context);
        break;
      case 'error':
        console.error(logMessage, context);
        break;
    }
  }

  public debug(message: string, context?: any): void {
    this.log('debug', message, context);
  }

  public info(message: string, context?: any): void {
    this.log('info', message, context);
  }

  public warn(message: string, context?: any): void {
    this.log('warn', message, context);
  }

  public error(message: string, context?: any): void {
    this.log('error', message, context);
  }

  public getLogs(level?: LogLevel): LogEntry[] {
    if (level) {
      return this.logs.filter(log => log.level === level);
    }
    return [...this.logs];
  }

  public clearLogs(): void {
    this.logs = [];
  }

  public exportLogs(): string {
    return JSON.stringify(this.logs, null, 2);
  }
}

// Usage
const logger = Logger.getInstance();
logger.setMinLevel('debug');

logger.debug('Application started');
logger.info('User logged in', { userId: 123 });
logger.warn('API rate limit approaching', { remaining: 10 });
logger.error('Failed to load data', { error: 'Network timeout' });

// Export logs
const logsJSON = logger.exportLogs();
```

### React Context Alternative to Singleton

```typescript
// Better approach: Use React Context instead of Singleton

// api-context.tsx
import React, { createContext, useContext, ReactNode } from 'react';

class APIService {
  private baseURL: string;

  constructor(baseURL: string) {
    this.baseURL = baseURL;
  }

  async get<T>(endpoint: string): Promise<T> {
    const response = await fetch(`${this.baseURL}${endpoint}`);
    return response.json();
  }

  async post<T>(endpoint: string, body: any): Promise<T> {
    const response = await fetch(`${this.baseURL}${endpoint}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body)
    });
    return response.json();
  }
}

const APIContext = createContext<APIService | null>(null);

export const APIProvider: React.FC<{
  baseURL: string;
  children: ReactNode;
}> = ({ baseURL, children }) => {
  const api = React.useMemo(() => new APIService(baseURL), [baseURL]);

  return (
    <APIContext.Provider value={api}>
      {children}
    </APIContext.Provider>
  );
};

export const useAPI = (): APIService => {
  const context = useContext(APIContext);
  if (!context) {
    throw new Error('useAPI must be used within APIProvider');
  }
  return context;
};

// App.tsx
function App() {
  return (
    <APIProvider baseURL="https://api.example.com">
      <UserList />
    </APIProvider>
  );
}

// Component usage
function UserList() {
  const api = useAPI();
  const [users, setUsers] = useState([]);

  useEffect(() => {
    api.get('/users').then(setUsers);
  }, [api]);

  return <div>{/* render users */}</div>;
}
```

### Angular Service Singleton

```typescript
// Angular services are singletons by default when provided in root

// api.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root' // This makes it a singleton
})
export class ApiService {
  private baseURL = 'https://api.example.com';

  constructor(private http: HttpClient) {}

  get<T>(endpoint: string): Observable<T> {
    return this.http.get<T>(`${this.baseURL}${endpoint}`);
  }

  post<T>(endpoint: string, body: any): Observable<T> {
    return this.http.post<T>(`${this.baseURL}${endpoint}`, body);
  }
}

// Component usage
@Component({
  selector: 'app-user-list',
  template: `
    <div *ngFor="let user of users$ | async">
      {{ user.name }}
    </div>
  `
})
export class UserListComponent {
  users$: Observable<User[]>;

  constructor(private api: ApiService) {
    this.users$ = this.api.get<User[]>('/users');
  }
}
```

### Module-Scoped Singleton

```typescript
// module-singleton.ts
class SingletonService {
  private data: Map<string, any> = new Map();

  getData(key: string): any {
    return this.data.get(key);
  }

  setData(key: string, value: any): void {
    this.data.set(key, value);
  }
}

// Export single instance (module-scoped)
export const singletonService = new SingletonService();

// Usage in any file
import { singletonService } from './module-singleton';

singletonService.setData('user', { name: 'John' });
const user = singletonService.getData('user');
```

### Resettable Singleton (Testing)

```typescript
// resettable-singleton.ts
export class ResettableSingleton {
  private static instance: ResettableSingleton | null = null;
  private data: string[] = [];

  private constructor() {}

  public static getInstance(): ResettableSingleton {
    if (!ResettableSingleton.instance) {
      ResettableSingleton.instance = new ResettableSingleton();
    }
    return ResettableSingleton.instance;
  }

  // For testing purposes only
  public static reset(): void {
    ResettableSingleton.instance = null;
  }

  public addData(value: string): void {
    this.data.push(value);
  }

  public getData(): string[] {
    return [...this.data];
  }
}

// test.spec.ts
describe('ResettableSingleton', () => {
  afterEach(() => {
    ResettableSingleton.reset(); // Clean state between tests
  });

  it('should maintain state within test', () => {
    const instance = ResettableSingleton.getInstance();
    instance.addData('test');
    expect(instance.getData()).toEqual(['test']);
  });

  it('should have clean state in new test', () => {
    const instance = ResettableSingleton.getInstance();
    expect(instance.getData()).toEqual([]); // Fresh instance
  });
});
```

### Registry Pattern (Multiple Named Singletons)

```typescript
// registry-pattern.ts
class Registry<T> {
  private instances = new Map<string, T>();

  register(name: string, instance: T): void {
    if (this.instances.has(name)) {
      throw new Error(`Instance '${name}' already registered`);
    }
    this.instances.set(name, instance);
  }

  get(name: string): T {
    const instance = this.instances.get(name);
    if (!instance) {
      throw new Error(`Instance '${name}' not found`);
    }
    return instance;
  }

  has(name: string): boolean {
    return this.instances.has(name);
  }

  unregister(name: string): void {
    this.instances.delete(name);
  }

  clear(): void {
    this.instances.clear();
  }
}

// Usage: Multiple API clients
interface APIClient {
  get(endpoint: string): Promise<any>;
}

const apiRegistry = new Registry<APIClient>();

apiRegistry.register('mainAPI', new APIService('https://api.main.com'));
apiRegistry.register('authAPI', new APIService('https://api.auth.com'));
apiRegistry.register('paymentAPI', new APIService('https://api.payment.com'));

const mainAPI = apiRegistry.get('mainAPI');
const authAPI = apiRegistry.get('authAPI');
```

## Common Mistakes

### 1. Global State Abuse

```typescript
// BAD: Using Singleton for all state
class GlobalState {
  private static instance: GlobalState;
  public user: User | null = null;
  public cart: CartItem[] = [];
  public preferences: UserPreferences = {};
  // 50 more properties...

  static getInstance() { return this.instance || (this.instance = new GlobalState()); }
}

// GOOD: Use proper state management
// Use Redux, MobX, or Context for application state
// Use Singleton only for services, not state
```

### 2. Hidden Dependencies

```typescript
// BAD: Hidden dependency on Singleton
class UserService {
  getUser(id: string) {
    // Hidden dependency - hard to test
    const api = APIClient.getInstance();
    return api.get(`/users/${id}`);
  }
}

// GOOD: Explicit dependency injection
class UserService {
  constructor(private api: APIClient) {}

  getUser(id: string) {
    return this.api.get(`/users/${id}`);
  }
}

// Easy to test with mock
const mockAPI = { get: jest.fn() };
const service = new UserService(mockAPI);
```

### 3. Not Thread-Safe

```typescript
// BAD: Race condition possible
class Singleton {
  private static instance: Singleton;

  static getInstance() {
    if (!this.instance) {
      // Multiple calls could create multiple instances
      this.instance = new Singleton();
    }
    return this.instance;
  }
}

// GOOD: Use eager initialization or locking
class Singleton {
  private static instance = new Singleton(); // Created immediately

  private constructor() {}

  static getInstance() {
    return this.instance;
  }
}
```

### 4. Difficulty Testing

```typescript
// BAD: Hard to test due to global state
class Cache {
  private static instance: Cache;
  private data = new Map();

  static getInstance() { /* ... */ }

  set(key: string, value: any) {
    this.data.set(key, value);
  }
}

// Tests interfere with each other
test('test 1', () => {
  Cache.getInstance().set('key', 'value1');
  // ...
});

test('test 2', () => {
  // Still has data from test 1!
  Cache.getInstance().set('key', 'value2');
});

// GOOD: Make resettable for tests
class Cache {
  // ... same as before
  
  static reset() {
    this.instance = null;
  }
}

afterEach(() => {
  Cache.reset();
});
```

## Best Practices

### 1. Prefer Dependency Injection

```typescript
// Instead of Singleton, use DI

// Service
class EmailService {
  send(to: string, message: string) {
    // Send email
  }
}

// Component receives service
class UserRegistration {
  constructor(private emailService: EmailService) {}

  register(user: User) {
    this.emailService.send(user.email, 'Welcome!');
  }
}

// Easy to test
const mockEmail = { send: jest.fn() };
const registration = new UserRegistration(mockEmail);
```

### 2. Use for Stateless Services

```typescript
// GOOD: Singleton for stateless service
class ValidationService {
  private static instance: ValidationService;

  static getInstance() { /* ... */ }

  validateEmail(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }

  validatePassword(password: string): boolean {
    return password.length >= 8;
  }
}
```

### 3. Document Singleton Usage

```typescript
/**
 * APIClient Singleton
 * 
 * Provides global access to API client with shared configuration.
 * This is a Singleton to ensure:
 * - Single connection pool
 * - Shared authentication state
 * - Consistent interceptors
 * 
 * @example
 * const api = APIClient.getInstance('https://api.example.com');
 * api.setAuthToken(token);
 * const data = await api.get('/users');
 */
export class APIClient {
  // Implementation
}
```

### 4. Consider Alternatives First

```typescript
// Before using Singleton, consider:

// 1. Module-scoped instance
export const logger = new Logger();

// 2. React Context
const APIContext = createContext<APIClient>(null);

// 3. Angular Service
@Injectable({ providedIn: 'root' })

// 4. Dependency Injection
constructor(private logger: Logger)

// 5. Factory function
export function createAPIClient() {
  return new APIClient();
}
```

## When to Use Singleton

### Use When

1. **Exactly one instance needed**: Connection pools, cache managers
2. **Global coordination**: Event bus, configuration manager
3. **Resource management**: File system access, database connections
4. **Logging**: Centralized logging service
5. **Shared state**: Feature flags, application settings

### Don't Use When

1. **Testing is priority**: Hard to mock and test
2. **Multiple instances may be needed**: Different configurations
3. **State should be scoped**: Component-level or module-level
4. **Better alternatives exist**: Context, DI, module exports
5. **Just sharing code**: Regular class or functions work

## Interview Questions

### Q1: Why is Singleton considered an anti-pattern?

Answer: Singletons introduce global state, making code harder to test and reason about. They hide dependencies, violate single responsibility principle (managing lifecycle + business logic), and make unit testing difficult. They're often a sign that better patterns like dependency injection should be used.

### Q2: How do you make a Singleton thread-safe in JavaScript?

Answer: JavaScript is single-threaded, but async operations can create race conditions. Use eager initialization (create instance immediately), implement locking with flags and async/await, or use module-scoped instances which are naturally singleton.

### Q3: What's the difference between Singleton and static class?

Answer: Singleton is an instance (can implement interfaces, pass around). Static class is just namespaced functions (cannot be instantiated, passed, or implement interfaces). Singleton provides more flexibility but similar global access issues.

### Q4: How do you test code that uses Singletons?

Answer: Make Singletons resettable for tests, use dependency injection to pass Singleton in, mock the Singleton's methods, or use test-specific instances. Better: avoid Singletons and use DI instead.

### Q5: When is Singleton actually the right choice?

Answer: When you genuinely need exactly one instance (hardware access, connection pool), need lazy initialization with global access, or managing shared resources. But first consider if Context, module exports, or DI would work better.

## Key Takeaways

1. **Singleton ensures single instance** with global access point
2. **Use private constructor** to prevent direct instantiation
3. **Lazy vs eager initialization** - choose based on needs
4. **Consider alternatives first** - Context, DI, module exports
5. **Makes testing harder** - hidden dependencies, global state
6. **Good for stateless services** - validation, formatting, logging
7. **Bad for application state** - use proper state management
8. **Document why it's a Singleton** - justify the pattern choice
9. **Provide reset mechanism** for testing purposes
10. **Thread-safety matters** even in JavaScript with async code

## Resources

### Documentation
- [Singleton Pattern (Refactoring Guru)](https://refactoring.guru/design-patterns/singleton)
- [Singleton Pattern (JavaScript)](https://www.patterns.dev/posts/singleton-pattern)
- [Why Singletons are Controversial](https://testing.googleblog.com/2008/08/root-cause-of-singletons.html)

### Articles
- "Singletons: Solve Problems or Create Them?" - Martin Fowler
- "Dependency Injection vs Singleton" - Clean Code
- "Testing with Singletons" - Unit Testing Best Practices

### Alternatives
- React Context API
- Angular Dependency Injection
- Module-scoped exports
- Factory Pattern
- Service Locator Pattern
