# IoC Containers and Dependency Inversion

## Overview

Inversion of Control (IoC) containers are frameworks that manage object creation, lifecycle, and dependency resolution automatically. They implement the Dependency Inversion Principle (DIP), where high-level modules don't depend on low-level modules but both depend on abstractions. IoC containers reduce boilerplate, improve testability, and enable loose coupling by inverting the traditional control flow where objects create their own dependencies.

## Core Concepts

### Dependency Inversion Principle

```
┌──────────────────────────────────────────────────────────┐
│        Traditional Dependency vs Inverted Dependency      │
└──────────────────────────────────────────────────────────┘

TRADITIONAL (High-level depends on Low-level)
┌─────────────────┐
│  OrderService   │
│   (High-level)  │
└────────┬────────┘
         │ depends on
         ▼
┌─────────────────┐
│ MySQLRepository │
│   (Low-level)   │
└─────────────────┘

Problem: OrderService tightly coupled to MySQL implementation


DEPENDENCY INVERSION (Both depend on abstraction)
┌─────────────────┐
│  OrderService   │────────┐
│   (High-level)  │        │
└────────┬────────┘        │
         │                 │ both depend on
         ▼                 ▼
    ┌────────────────────────┐
    │ IOrderRepository       │
    │    (Abstraction)       │
    └────────────────────────┘
                 ▲
                 │ implements
                 │
    ┌────────────┴───────────┐
    │   MySQLOrderRepository │
    │      (Low-level)       │
    └────────────────────────┘

Benefit: OrderService decoupled, can swap implementations
```

### IoC Container Flow

```
┌──────────────────────────────────────────────────────────┐
│              IoC Container Workflow                       │
└──────────────────────────────────────────────────────────┘

1. REGISTRATION PHASE
   container.register(IUserService, UserService)
   container.register(IDatabase, PostgresDatabase)

2. RESOLUTION PHASE
   Application requests: IUserService
                            │
                            ▼
                     IoC Container
                            │
                   ┌────────┴────────┐
                   │ Check Registry  │
                   └────────┬────────┘
                            │
                   ┌────────▼────────┐
                   │  Create/Retrieve│
                   │   UserService   │
                   └────────┬────────┘
                            │
                   ┌────────▼────────┐
                   │ Inject deps:    │
                   │  - IDatabase    │
                   └────────┬────────┘
                            │
                            ▼
                    Return instance
```

## Building a Simple IoC Container

### Basic Container Implementation

```typescript
type Constructor<T = any> = new (...args: any[]) => T;
type Factory<T = any> = () => T;

class SimpleContainer {
  private services = new Map<any, any>();
  private singletons = new Map<any, any>();

  // Register a service
  register<T>(token: any, implementation: Constructor<T> | Factory<T>) {
    this.services.set(token, implementation);
  }

  // Register a singleton
  registerSingleton<T>(token: any, implementation: Constructor<T> | Factory<T>) {
    this.services.set(token, implementation);
    // Mark as singleton
    this.singletons.set(token, null);
  }

  // Resolve a dependency
  resolve<T>(token: any): T {
    // Check if singleton already exists
    if (this.singletons.has(token)) {
      const existing = this.singletons.get(token);
      if (existing) return existing;
    }

    const implementation = this.services.get(token);
    
    if (!implementation) {
      throw new Error(`No provider found for ${token}`);
    }

    let instance: T;

    if (typeof implementation === 'function') {
      // Check if it's a class constructor
      if (implementation.prototype) {
        instance = new implementation();
      } else {
        // It's a factory function
        instance = implementation();
      }
    } else {
      instance = implementation;
    }

    // Store singleton
    if (this.singletons.has(token)) {
      this.singletons.set(token, instance);
    }

    return instance;
  }

  // Clear all registrations
  clear() {
    this.services.clear();
    this.singletons.clear();
  }
}

// Usage
interface ILogger {
  log(message: string): void;
}

class ConsoleLogger implements ILogger {
  log(message: string) {
    console.log(message);
  }
}

interface IUserService {
  getUser(id: string): any;
}

class UserService implements IUserService {
  constructor(private logger: ILogger) {}

  getUser(id: string) {
    this.logger.log(`Fetching user ${id}`);
    return { id, name: 'John' };
  }
}

// Setup
const container = new SimpleContainer();
container.registerSingleton('ILogger', ConsoleLogger);
container.register('IUserService', () => {
  const logger = container.resolve<ILogger>('ILogger');
  return new UserService(logger);
});

// Usage
const userService = container.resolve<IUserService>('IUserService');
userService.getUser('123');
```

### Advanced Container with Metadata

```typescript
import 'reflect-metadata';

const INJECT_METADATA_KEY = Symbol('INJECT_KEY');

// Decorator for dependency injection
function Injectable() {
  return function <T extends { new(...args: any[]): {} }>(constructor: T) {
    return constructor;
  };
}

function Inject(token: any) {
  return function(target: any, propertyKey: string | symbol, parameterIndex: number) {
    const existingInjectedParams = Reflect.getOwnMetadata(
      INJECT_METADATA_KEY,
      target
    ) || [];
    
    existingInjectedParams[parameterIndex] = token;
    
    Reflect.defineMetadata(
      INJECT_METADATA_KEY,
      existingInjectedParams,
      target
    );
  };
}

class Container {
  private services = new Map<any, any>();
  private instances = new Map<any, any>();

  register<T>(token: any, implementation: Constructor<T>, singleton = false) {
    this.services.set(token, { implementation, singleton });
  }

  resolve<T>(token: any): T {
    const service = this.services.get(token);
    
    if (!service) {
      throw new Error(`No provider found for ${String(token)}`);
    }

    // Return existing singleton
    if (service.singleton && this.instances.has(token)) {
      return this.instances.get(token);
    }

    // Get constructor parameter types
    const paramTypes = Reflect.getMetadata('design:paramtypes', service.implementation) || [];
    
    // Get injected parameters
    const injectedParams = Reflect.getOwnMetadata(
      INJECT_METADATA_KEY,
      service.implementation
    ) || [];

    // Resolve dependencies
    const dependencies = paramTypes.map((paramType: any, index: number) => {
      const injectedToken = injectedParams[index];
      return this.resolve(injectedToken || paramType);
    });

    const instance = new service.implementation(...dependencies);

    // Store singleton
    if (service.singleton) {
      this.instances.set(token, instance);
    }

    return instance;
  }
}

// Usage
interface IDatabase {
  query(sql: string): any;
}

@Injectable()
class Database implements IDatabase {
  query(sql: string) {
    console.log(`Executing: ${sql}`);
    return [];
  }
}

interface IUserRepository {
  findById(id: string): any;
}

@Injectable()
class UserRepository implements IUserRepository {
  constructor(@Inject('IDatabase') private db: IDatabase) {}

  findById(id: string) {
    return this.db.query(`SELECT * FROM users WHERE id = '${id}'`);
  }
}

@Injectable()
class UserService {
  constructor(@Inject('IUserRepository') private repo: IUserRepository) {}

  getUser(id: string) {
    return this.repo.findById(id);
  }
}

// Setup
const container = new Container();
container.register('IDatabase', Database, true); // Singleton
container.register('IUserRepository', UserRepository);
container.register('IUserService', UserService);

const userService = container.resolve<UserService>('IUserService');
```

## TypeScript with InversifyJS

### Basic InversifyJS Setup

```typescript
import { Container, injectable, inject } from 'inversify';
import 'reflect-metadata';

// Define service identifiers
const TYPES = {
  Logger: Symbol.for('Logger'),
  Database: Symbol.for('Database'),
  UserRepository: Symbol.for('UserRepository'),
  UserService: Symbol.for('UserService')
};

// Interfaces
interface Logger {
  log(message: string): void;
}

interface Database {
  query(sql: string): Promise<any>;
}

interface UserRepository {
  findById(id: string): Promise<any>;
  save(user: any): Promise<void>;
}

interface UserService {
  getUser(id: string): Promise<any>;
  createUser(name: string, email: string): Promise<any>;
}

// Implementations
@injectable()
class ConsoleLogger implements Logger {
  log(message: string) {
    console.log(`[LOG] ${message}`);
  }
}

@injectable()
class PostgresDatabase implements Database {
  async query(sql: string) {
    console.log(`Executing: ${sql}`);
    return [];
  }
}

@injectable()
class UserRepositoryImpl implements UserRepository {
  constructor(
    @inject(TYPES.Database) private db: Database,
    @inject(TYPES.Logger) private logger: Logger
  ) {}

  async findById(id: string) {
    this.logger.log(`Finding user ${id}`);
    return this.db.query(`SELECT * FROM users WHERE id = '${id}'`);
  }

  async save(user: any) {
    this.logger.log(`Saving user ${user.id}`);
    await this.db.query(`INSERT INTO users ...`);
  }
}

@injectable()
class UserServiceImpl implements UserService {
  constructor(
    @inject(TYPES.UserRepository) private repo: UserRepository,
    @inject(TYPES.Logger) private logger: Logger
  ) {}

  async getUser(id: string) {
    this.logger.log(`Getting user ${id}`);
    return this.repo.findById(id);
  }

  async createUser(name: string, email: string) {
    const user = { id: Date.now().toString(), name, email };
    await this.repo.save(user);
    return user;
  }
}

// Container configuration
const container = new Container();
container.bind<Logger>(TYPES.Logger).to(ConsoleLogger).inSingletonScope();
container.bind<Database>(TYPES.Database).to(PostgresDatabase).inSingletonScope();
container.bind<UserRepository>(TYPES.UserRepository).to(UserRepositoryImpl);
container.bind<UserService>(TYPES.UserService).to(UserServiceImpl);

// Usage
const userService = container.get<UserService>(TYPES.UserService);
await userService.createUser('John', 'john@example.com');
```

### InversifyJS with React

```typescript
import React, { useContext, createContext } from 'react';
import { Container, injectable, inject } from 'inversify';

// Container context
const ContainerContext = createContext<Container | null>(null);

export const ContainerProvider: React.FC<{ container: Container }> = ({
  container,
  children
}) => {
  return (
    <ContainerContext.Provider value={container}>
      {children}
    </ContainerContext.Provider>
  );
};

// Hook to resolve dependencies
export function useInjection<T>(identifier: symbol): T {
  const container = useContext(ContainerContext);
  if (!container) {
    throw new Error('No container provided');
  }
  return container.get<T>(identifier);
}

// Services
interface TodoService {
  getTodos(): Promise<any[]>;
  addTodo(text: string): Promise<void>;
}

@injectable()
class TodoServiceImpl implements TodoService {
  constructor(@inject(TYPES.Logger) private logger: Logger) {}

  async getTodos() {
    this.logger.log('Fetching todos');
    return [];
  }

  async addTodo(text: string) {
    this.logger.log(`Adding todo: ${text}`);
  }
}

// Component using injection
function TodoList() {
  const todoService = useInjection<TodoService>(TYPES.TodoService);
  const [todos, setTodos] = React.useState([]);

  React.useEffect(() => {
    todoService.getTodos().then(setTodos);
  }, [todoService]);

  const handleAdd = async (text: string) => {
    await todoService.addTodo(text);
    const updated = await todoService.getTodos();
    setTodos(updated);
  };

  return (
    <div>
      {todos.map(todo => (
        <div key={todo.id}>{todo.text}</div>
      ))}
      <button onClick={() => handleAdd('New todo')}>Add</button>
    </div>
  );
}

// App setup
const container = new Container();
container.bind<Logger>(TYPES.Logger).to(ConsoleLogger).inSingletonScope();
container.bind<TodoService>(TYPES.TodoService).to(TodoServiceImpl);

function App() {
  return (
    <ContainerProvider container={container}>
      <TodoList />
    </ContainerProvider>
  );
}
```

## Angular Dependency Injection

### Angular Built-in DI

```typescript
// Service with dependencies
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Injectable({
  providedIn: 'root' // Singleton across app
})
export class LoggerService {
  log(message: string) {
    console.log(`[${new Date().toISOString()}] ${message}`);
  }
}

@Injectable({
  providedIn: 'root'
})
export class UserService {
  constructor(
    private http: HttpClient,
    private logger: LoggerService
  ) {}

  getUsers() {
    this.logger.log('Fetching users');
    return this.http.get('/api/users');
  }
}

// Component with DI
import { Component, OnInit } from '@angular/core';

@Component({
  selector: 'app-users',
  template: `
    <div *ngFor="let user of users$ | async">
      {{ user.name }}
    </div>
  `
})
export class UsersComponent implements OnInit {
  users$;

  constructor(private userService: UserService) {}

  ngOnInit() {
    this.users$ = this.userService.getUsers();
  }
}
```

### Angular with Injection Tokens

```typescript
import { InjectionToken, Inject } from '@angular/core';

// Define injection token
export const API_URL = new InjectionToken<string>('API_URL');

// Use in service
@Injectable()
export class DataService {
  constructor(@Inject(API_URL) private apiUrl: string) {}

  fetchData() {
    return this.http.get(`${this.apiUrl}/data`);
  }
}

// Provide value
@NgModule({
  providers: [
    { provide: API_URL, useValue: 'https://api.example.com' }
  ]
})
export class AppModule {}

// Interface injection token
interface Config {
  apiUrl: string;
  timeout: number;
}

export const APP_CONFIG = new InjectionToken<Config>('APP_CONFIG');

// Factory provider
@NgModule({
  providers: [
    {
      provide: APP_CONFIG,
      useFactory: () => ({
        apiUrl: environment.apiUrl,
        timeout: 5000
      })
    }
  ]
})
export class AppModule {}
```

### Angular Hierarchical Injection

```typescript
// Module-level provider
@NgModule({
  providers: [
    { provide: ThemeService, useClass: DefaultThemeService }
  ]
})
export class AppModule {}

// Component-level provider (overrides module)
@Component({
  selector: 'app-admin',
  providers: [
    { provide: ThemeService, useClass: AdminThemeService }
  ],
  template: `<div>Admin section with different theme</div>`
})
export class AdminComponent {}

// Optional dependency
@Injectable()
export class AnalyticsService {
  constructor(@Optional() private tracker: TrackerService) {}

  track(event: string) {
    if (this.tracker) {
      this.tracker.track(event);
    }
  }
}

// Self vs SkipSelf
@Component({
  providers: [LoggerService]
})
export class ParentComponent {
  constructor(@Self() private logger: LoggerService) {
    // Gets LoggerService from this component
  }
}

@Component({
  selector: 'app-child'
})
export class ChildComponent {
  constructor(@SkipSelf() private logger: LoggerService) {
    // Skips this component, gets from parent
  }
}
```

## Testing with IoC

### Mock Dependencies in Tests

```typescript
import { Container } from 'inversify';

describe('UserService', () => {
  let container: Container;
  let userService: UserService;
  let mockRepo: jest.Mocked<UserRepository>;
  let mockLogger: jest.Mocked<Logger>;

  beforeEach(() => {
    // Create mocks
    mockRepo = {
      findById: jest.fn(),
      save: jest.fn()
    };

    mockLogger = {
      log: jest.fn()
    };

    // Setup container with mocks
    container = new Container();
    container.bind<UserRepository>(TYPES.UserRepository).toConstantValue(mockRepo);
    container.bind<Logger>(TYPES.Logger).toConstantValue(mockLogger);
    container.bind<UserService>(TYPES.UserService).to(UserServiceImpl);

    userService = container.get<UserService>(TYPES.UserService);
  });

  it('should fetch user by id', async () => {
    const mockUser = { id: '123', name: 'John' };
    mockRepo.findById.mockResolvedValue(mockUser);

    const result = await userService.getUser('123');

    expect(result).toEqual(mockUser);
    expect(mockRepo.findById).toHaveBeenCalledWith('123');
    expect(mockLogger.log).toHaveBeenCalledWith('Getting user 123');
  });
});
```

### Angular Testing with DI

```typescript
import { TestBed } from '@angular/core/testing';
import { HttpClientTestingModule } from '@angular/common/http/testing';

describe('UserService', () => {
  let service: UserService;
  let mockLogger: jasmine.SpyObj<LoggerService>;

  beforeEach(() => {
    // Create mock
    mockLogger = jasmine.createSpyObj('LoggerService', ['log']);

    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [
        UserService,
        { provide: LoggerService, useValue: mockLogger }
      ]
    });

    service = TestBed.inject(UserService);
  });

  it('should log when fetching users', () => {
    service.getUsers().subscribe();
    expect(mockLogger.log).toHaveBeenCalledWith('Fetching users');
  });
});
```

## Common Mistakes

### 1. Circular Dependencies

```typescript
// ❌ Wrong - circular dependency
@injectable()
class ServiceA {
  constructor(@inject(TYPES.ServiceB) private b: ServiceB) {}
}

@injectable()
class ServiceB {
  constructor(@inject(TYPES.ServiceA) private a: ServiceA) {}
}

// ✅ Correct - use interface or event bus to break cycle
interface IEventBus {
  emit(event: string, data: any): void;
  on(event: string, handler: Function): void;
}

@injectable()
class ServiceA {
  constructor(@inject(TYPES.EventBus) private events: IEventBus) {}
  
  doSomething() {
    this.events.emit('something-happened', { data: 'value' });
  }
}

@injectable()
class ServiceB {
  constructor(@inject(TYPES.EventBus) private events: IEventBus) {
    this.events.on('something-happened', this.handleEvent);
  }
}
```

### 2. Not Using Interfaces

```typescript
// ❌ Wrong - depending on concrete implementation
@injectable()
class UserService {
  constructor(@inject(MySQLDatabase) private db: MySQLDatabase) {}
}

// ✅ Correct - depend on abstraction
interface IDatabase {
  query(sql: string): Promise<any>;
}

@injectable()
class UserService {
  constructor(@inject(TYPES.Database) private db: IDatabase) {}
}
```

### 3. Forgetting Singleton Scope

```typescript
// ❌ Wrong - new instance each time
container.bind<Database>(TYPES.Database).to(Database);

// ✅ Correct - singleton for shared resources
container.bind<Database>(TYPES.Database).to(Database).inSingletonScope();
```

### 4. Using Container as Service Locator

```typescript
// ❌ Wrong - service locator anti-pattern
class UserService {
  constructor(private container: Container) {}
  
  getUser(id: string) {
    const repo = this.container.get<UserRepository>(TYPES.UserRepository);
    return repo.findById(id);
  }
}

// ✅ Correct - inject dependencies directly
class UserService {
  constructor(@inject(TYPES.UserRepository) private repo: UserRepository) {}
  
  getUser(id: string) {
    return this.repo.findById(id);
  }
}
```

## Best Practices

### 1. Program to Interfaces

```typescript
// Define interface
interface IEmailService {
  send(to: string, subject: string, body: string): Promise<void>;
}

// Multiple implementations
@injectable()
class SendGridEmailService implements IEmailService {
  async send(to: string, subject: string, body: string) {
    // SendGrid implementation
  }
}

@injectable()
class MockEmailService implements IEmailService {
  async send(to: string, subject: string, body: string) {
    console.log(`Mock: Sending email to ${to}`);
  }
}

// Bind based on environment
if (process.env.NODE_ENV === 'production') {
  container.bind<IEmailService>(TYPES.EmailService).to(SendGridEmailService);
} else {
  container.bind<IEmailService>(TYPES.EmailService).to(MockEmailService);
}
```

### 2. Use Symbols for Tokens

```typescript
// Good - prevents collisions
const TYPES = {
  Logger: Symbol.for('Logger'),
  Database: Symbol.for('Database')
};
```

### 3. Configure Container in One Place

```typescript
// container.config.ts
export function configureContainer() {
  const container = new Container();
  
  // Infrastructure
  container.bind<Logger>(TYPES.Logger).to(ConsoleLogger).inSingletonScope();
  container.bind<Database>(TYPES.Database).to(PostgresDatabase).inSingletonScope();
  
  // Repositories
  container.bind<UserRepository>(TYPES.UserRepository).to(UserRepositoryImpl);
  
  // Services
  container.bind<UserService>(TYPES.UserService).to(UserServiceImpl);
  
  return container;
}
```

### 4. Document Dependencies

```typescript
/**
 * UserService handles user-related business logic
 * 
 * Dependencies:
 * - UserRepository: Data access for users
 * - Logger: Logging service
 * - EmailService: Send user notifications
 */
@injectable()
class UserService {
  constructor(
    @inject(TYPES.UserRepository) private repo: UserRepository,
    @inject(TYPES.Logger) private logger: Logger,
    @inject(TYPES.EmailService) private email: EmailService
  ) {}
}
```

## Interview Questions

### Q1: What is the Dependency Inversion Principle?
**Answer**: DIP states that high-level modules should not depend on low-level modules; both should depend on abstractions. Abstractions should not depend on details; details should depend on abstractions. This inverts traditional dependency flow, enabling loose coupling and easier testing through dependency injection.

### Q2: How do IoC containers improve testability?
**Answer**: IoC containers enable easy substitution of dependencies with mocks/stubs in tests. You register test doubles instead of real implementations, and the container injects them automatically. This avoids manual wiring and ensures consistent dependency resolution across tests.

### Q3: What's the difference between IoC and Dependency Injection?
**Answer**: IoC is a broader principle where framework controls flow rather than application. DI is a specific IoC pattern for providing dependencies. IoC containers implement DI by managing object creation and dependency resolution. IoC encompasses other patterns like event-driven architecture, but DI is most common in frontend.

### Q4: When should you use Singleton scope?
**Answer**: Use singleton for stateful services shared across application (database connections, configuration, caches, loggers). Avoid for services holding request-specific state. In frontend, most services can be singletons. Use transient scope for stateful, request-specific, or expensive-to-create objects.

### Q5: What are circular dependencies and how do you fix them?
**Answer**: Circular dependencies occur when A depends on B and B depends on A, creating a cycle. Fix by introducing an abstraction/interface both depend on, using events/pub-sub to decouple, lazy-loading one dependency, or restructuring to eliminate the cycle. IoC containers typically detect and error on circular deps.

## Key Takeaways

1. IoC containers automate dependency management and object creation
2. Dependency Inversion Principle decouples high-level from low-level modules
3. Program to interfaces, not implementations
4. Use singleton scope for shared, stateless services
5. Avoid circular dependencies through abstraction
6. IoC improves testability by enabling easy mock injection
7. Configure container in centralized location
8. Use symbols/tokens to prevent naming collisions
9. Don't use container as service locator anti-pattern
10. Angular has built-in DI; React requires libraries like InversifyJS

## Resources

- [InversifyJS Documentation](https://inversify.io/)
- [TSyringe (TypeScript DI)](https://github.com/microsoft/tsyringe)
- [Angular Dependency Injection](https://angular.io/guide/dependency-injection)
- [Dependency Inversion Principle](https://en.wikipedia.org/wiki/Dependency_inversion_principle)
- [Martin Fowler on IoC](https://martinfowler.com/articles/injection.html)
- [SOLID Principles](https://en.wikipedia.org/wiki/SOLID)
