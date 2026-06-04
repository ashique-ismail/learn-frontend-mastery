# SOLID Principles

## Overview

SOLID is a set of five object-oriented design principles that help developers create more maintainable, flexible, and scalable software. While originally formulated for backend systems, these principles apply powerfully to frontend development, particularly in component architecture, state management, and application structure.

## The Five Principles

```
S - Single Responsibility Principle
O - Open/Closed Principle
L - Liskov Substitution Principle
I - Interface Segregation Principle
D - Dependency Inversion Principle
```

## Single Responsibility Principle (SRP)

**A class/component should have only one reason to change.**

### React Example

```tsx
// ❌ BAD: Component has multiple responsibilities
export const UserDashboard: React.FC = () => {
  const [user, setUser] = useState<User | null>(null);
  const [orders, setOrders] = useState<Order[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Fetching data
    fetch('/api/user')
      .then(res => res.json())
      .then(setUser);

    fetch('/api/orders')
      .then(res => res.json())
      .then(setOrders);
  }, []);

  // Validation logic
  const validateUser = (user: User) => {
    return user.email.includes('@') && user.age >= 18;
  };

  // Business logic
  const calculateTotalSpent = () => {
    return orders.reduce((sum, order) => sum + order.total, 0);
  };

  // Rendering
  return (
    <div>
      <header>
        <h1>{user?.name}</h1>
        <p>{user?.email}</p>
      </header>
      <section>
        <h2>Orders</h2>
        {orders.map(order => (
          <div key={order.id}>
            <span>{order.id}</span>
            <span>${order.total}</span>
          </div>
        ))}
      </section>
      <footer>
        <p>Total Spent: ${calculateTotalSpent()}</p>
      </footer>
    </div>
  );
};

// Reasons to change:
// 1. API endpoints change
// 2. Validation rules change
// 3. Business logic changes
// 4. UI layout changes
// 5. Data structure changes

// ✅ GOOD: Each component/hook has single responsibility
// 1. Data fetching
export const useUser = () => {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/user')
      .then(res => res.json())
      .then(setUser)
      .finally(() => setLoading(false));
  }, []);

  return { user, loading };
};

export const useOrders = () => {
  const [orders, setOrders] = useState<Order[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/orders')
      .then(res => res.json())
      .then(setOrders)
      .finally(() => setLoading(false));
  }, []);

  return { orders, loading };
};

// 2. Business logic
export const useOrderAnalytics = (orders: Order[]) => {
  const totalSpent = useMemo(() => {
    return orders.reduce((sum, order) => sum + order.total, 0);
  }, [orders]);

  return { totalSpent };
};

// 3. Validation
export const userValidator = {
  validateEmail: (email: string) => email.includes('@'),
  validateAge: (age: number) => age >= 18,
  validateUser: (user: User) => {
    return userValidator.validateEmail(user.email) && 
           userValidator.validateAge(user.age);
  }
};

// 4. Presentational components
export const UserHeader: React.FC<{ user: User }> = ({ user }) => (
  <header>
    <h1>{user.name}</h1>
    <p>{user.email}</p>
  </header>
);

export const OrderList: React.FC<{ orders: Order[] }> = ({ orders }) => (
  <section>
    <h2>Orders</h2>
    {orders.map(order => (
      <OrderItem key={order.id} order={order} />
    ))}
  </section>
);

export const OrderItem: React.FC<{ order: Order }> = ({ order }) => (
  <div>
    <span>{order.id}</span>
    <span>${order.total}</span>
  </div>
);

// 5. Composition
export const UserDashboard: React.FC = () => {
  const { user, loading: userLoading } = useUser();
  const { orders, loading: ordersLoading } = useOrders();
  const { totalSpent } = useOrderAnalytics(orders);

  if (userLoading || ordersLoading) return <LoadingSpinner />;
  if (!user) return <ErrorMessage />;

  return (
    <div>
      <UserHeader user={user} />
      <OrderList orders={orders} />
      <footer>
        <p>Total Spent: ${totalSpent}</p>
      </footer>
    </div>
  );
};

// Each piece has one reason to change
```

### Angular Example

```typescript
// ❌ BAD: Service with multiple responsibilities
@Injectable()
export class UserService {
  constructor(private http: HttpClient) {}

  // Data access
  getUser(id: string): Observable<User> {
    return this.http.get<User>(`/api/users/${id}`);
  }

  // Validation
  validateUser(user: User): boolean {
    return user.email.includes('@') && user.age >= 18;
  }

  // Business logic
  calculateUserLevel(user: User): string {
    if (user.points > 1000) return 'Gold';
    if (user.points > 500) return 'Silver';
    return 'Bronze';
  }

  // Formatting
  formatUserName(user: User): string {
    return `${user.firstName} ${user.lastName}`;
  }
}

// ✅ GOOD: Separated concerns
// 1. Data access
@Injectable()
export class UserRepository {
  constructor(private http: HttpClient) {}

  getUser(id: string): Observable<User> {
    return this.http.get<User>(`/api/users/${id}`);
  }

  updateUser(user: User): Observable<User> {
    return this.http.put<User>(`/api/users/${user.id}`, user);
  }
}

// 2. Validation
@Injectable()
export class UserValidator {
  validateEmail(email: string): boolean {
    return email.includes('@');
  }

  validateAge(age: number): boolean {
    return age >= 18;
  }

  validateUser(user: User): boolean {
    return this.validateEmail(user.email) && this.validateAge(user.age);
  }
}

// 3. Business logic
@Injectable()
export class UserLevelService {
  calculateLevel(points: number): string {
    if (points > 1000) return 'Gold';
    if (points > 500) return 'Silver';
    return 'Bronze';
  }
}

// 4. Formatting
@Pipe({ name: 'userName' })
export class UserNamePipe implements PipeTransform {
  transform(user: User): string {
    return `${user.firstName} ${user.lastName}`;
  }
}
```

## Open/Closed Principle (OCP)

**Software entities should be open for extension but closed for modification.**

### React Example

```tsx
// ❌ BAD: Must modify component to add new button types
interface ButtonProps {
  type: 'primary' | 'secondary' | 'danger';
  onClick: () => void;
  children: React.ReactNode;
}

export const Button: React.FC<ButtonProps> = ({ type, onClick, children }) => {
  let className = 'btn';

  if (type === 'primary') {
    className += ' btn-primary';
  } else if (type === 'secondary') {
    className += ' btn-secondary';
  } else if (type === 'danger') {
    className += ' btn-danger';
  }
  // Need to add more if/else for each new type

  return (
    <button className={className} onClick={onClick}>
      {children}
    </button>
  );
};

// ✅ GOOD: Open for extension, closed for modification
interface ButtonProps {
  variant?: string;
  onClick: () => void;
  children: React.ReactNode;
  className?: string;
}

export const Button: React.FC<ButtonProps> = ({ 
  variant = 'primary', 
  onClick, 
  children,
  className = ''
}) => {
  return (
    <button 
      className={`btn btn-${variant} ${className}`} 
      onClick={onClick}
    >
      {children}
    </button>
  );
};

// Extend without modifying
export const PrimaryButton: React.FC<Omit<ButtonProps, 'variant'>> = (props) => (
  <Button variant="primary" {...props} />
);

export const DangerButton: React.FC<Omit<ButtonProps, 'variant'>> = (props) => (
  <Button variant="danger" {...props} />
);

export const SuccessButton: React.FC<Omit<ButtonProps, 'variant'>> = (props) => (
  <Button variant="success" {...props} />
);

// Or with composition
export const IconButton: React.FC<ButtonProps & { icon: React.ReactNode }> = ({
  icon,
  children,
  ...props
}) => (
  <Button {...props}>
    {icon}
    {children}
  </Button>
);
```

### Strategy Pattern for Extensibility

```typescript
// ❌ BAD: Must modify class for new payment methods
class PaymentProcessor {
  processPayment(method: string, amount: number): void {
    if (method === 'creditCard') {
      // Credit card logic
      console.log(`Processing ${amount} via credit card`);
    } else if (method === 'paypal') {
      // PayPal logic
      console.log(`Processing ${amount} via PayPal`);
    } else if (method === 'crypto') {
      // Crypto logic
      console.log(`Processing ${amount} via crypto`);
    }
    // Need to add more conditions for each new method
  }
}

// ✅ GOOD: Open for extension via strategy pattern
interface PaymentStrategy {
  process(amount: number): Promise<PaymentResult>;
}

class CreditCardPayment implements PaymentStrategy {
  async process(amount: number): Promise<PaymentResult> {
    console.log(`Processing ${amount} via credit card`);
    return { success: true, transactionId: 'cc-123' };
  }
}

class PayPalPayment implements PaymentStrategy {
  async process(amount: number): Promise<PaymentResult> {
    console.log(`Processing ${amount} via PayPal`);
    return { success: true, transactionId: 'pp-456' };
  }
}

class CryptoPayment implements PaymentStrategy {
  async process(amount: number): Promise<PaymentResult> {
    console.log(`Processing ${amount} via crypto`);
    return { success: true, transactionId: 'crypto-789' };
  }
}

// Processor doesn't need to change when adding new methods
class PaymentProcessor {
  constructor(private strategy: PaymentStrategy) {}

  async processPayment(amount: number): Promise<PaymentResult> {
    return this.strategy.process(amount);
  }

  setStrategy(strategy: PaymentStrategy): void {
    this.strategy = strategy;
  }
}

// Usage
const processor = new PaymentProcessor(new CreditCardPayment());
await processor.processPayment(100);

// Easy to add new payment method without modifying existing code
class ApplePayPayment implements PaymentStrategy {
  async process(amount: number): Promise<PaymentResult> {
    console.log(`Processing ${amount} via Apple Pay`);
    return { success: true, transactionId: 'apple-999' };
  }
}

processor.setStrategy(new ApplePayPayment());
```

## Liskov Substitution Principle (LSP)

**Derived classes must be substitutable for their base classes.**

### React Example

```tsx
// ❌ BAD: Violates LSP - LoadingButton can't fully replace Button
interface ButtonProps {
  onClick: () => void;
  children: React.ReactNode;
}

export const Button: React.FC<ButtonProps> = ({ onClick, children }) => {
  return <button onClick={onClick}>{children}</button>;
};

// This component requires additional props, breaking substitution
export const LoadingButton: React.FC<ButtonProps & { isLoading: boolean }> = ({
  onClick,
  children,
  isLoading
}) => {
  if (isLoading) {
    return <button disabled>Loading...</button>;
  }
  return <button onClick={onClick}>{children}</button>;
};

// Can't substitute LoadingButton for Button without changing call sites
// <Button onClick={handleClick}>Save</Button>
// <LoadingButton onClick={handleClick}>Save</LoadingButton> // Error: isLoading missing

// ✅ GOOD: Both buttons share same interface
interface ButtonProps {
  onClick: () => void;
  children: React.ReactNode;
  disabled?: boolean;
}

export const Button: React.FC<ButtonProps> = ({ 
  onClick, 
  children,
  disabled = false 
}) => {
  return (
    <button onClick={onClick} disabled={disabled}>
      {children}
    </button>
  );
};

export const LoadingButton: React.FC<ButtonProps & { isLoading?: boolean }> = ({
  onClick,
  children,
  disabled = false,
  isLoading = false
}) => {
  return (
    <button onClick={onClick} disabled={disabled || isLoading}>
      {isLoading ? 'Loading...' : children}
    </button>
  );
};

// Both can be used interchangeably
const MyButton = someCondition ? Button : LoadingButton;
<MyButton onClick={handleClick} disabled={!canSave}>Save</MyButton>
```

### Inheritance Example

```typescript
// ❌ BAD: Rectangle-Square problem (classic LSP violation)
class Rectangle {
  constructor(protected width: number, protected height: number) {}

  setWidth(width: number): void {
    this.width = width;
  }

  setHeight(height: number): void {
    this.height = height;
  }

  getArea(): number {
    return this.width * this.height;
  }
}

class Square extends Rectangle {
  constructor(size: number) {
    super(size, size);
  }

  // Violates LSP - changes behavior unexpectedly
  setWidth(width: number): void {
    this.width = width;
    this.height = width; // Side effect!
  }

  setHeight(height: number): void {
    this.width = height; // Side effect!
    this.height = height;
  }
}

function testRectangle(rect: Rectangle) {
  rect.setWidth(5);
  rect.setHeight(4);
  console.assert(rect.getArea() === 20); // Fails for Square!
}

testRectangle(new Rectangle(0, 0)); // ✅ Works
testRectangle(new Square(0)); // ❌ Fails assertion (getArea returns 16, not 20)

// ✅ GOOD: Composition over inheritance
interface Shape {
  getArea(): number;
}

class Rectangle implements Shape {
  constructor(private width: number, private height: number) {}

  setWidth(width: number): void {
    this.width = width;
  }

  setHeight(height: number): void {
    this.height = height;
  }

  getArea(): number {
    return this.width * this.height;
  }
}

class Square implements Shape {
  constructor(private size: number) {}

  setSize(size: number): void {
    this.size = size;
  }

  getArea(): number {
    return this.size * this.size;
  }
}

// Now each shape has appropriate interface
function printArea(shape: Shape) {
  console.log(`Area: ${shape.getArea()}`);
}

printArea(new Rectangle(5, 4)); // ✅ Works
printArea(new Square(5)); // ✅ Works
```

## Interface Segregation Principle (ISP)

**Clients should not depend on interfaces they don't use.**

### React Example

```tsx
// ❌ BAD: Fat interface with many optional properties
interface UserCardProps {
  user: User;
  showAvatar?: boolean;
  showEmail?: boolean;
  showPhone?: boolean;
  showAddress?: boolean;
  showOrders?: boolean;
  showPurchaseHistory?: boolean;
  onEdit?: () => void;
  onDelete?: () => void;
  onSendMessage?: () => void;
  onViewProfile?: () => void;
}

export const UserCard: React.FC<UserCardProps> = (props) => {
  // Component forced to handle many properties it might not need
  return <div>...</div>;
};

// Every user needs to know about all these options
<UserCard 
  user={user}
  showAvatar={true}
  showEmail={true}
  showPhone={false}
  showAddress={false}
  // ... many more props
/>

// ✅ GOOD: Segregated interfaces
interface BasicUserCardProps {
  user: User;
}

interface UserActionsProps {
  onEdit: () => void;
  onDelete: () => void;
}

interface UserContactProps {
  showEmail?: boolean;
  showPhone?: boolean;
}

// Small, focused components
export const UserAvatar: React.FC<{ user: User }> = ({ user }) => (
  <img src={user.avatar} alt={user.name} />
);

export const UserInfo: React.FC<{ user: User }> = ({ user }) => (
  <div>
    <h3>{user.name}</h3>
    <p>{user.bio}</p>
  </div>
);

export const UserContact: React.FC<{ user: User } & UserContactProps> = ({
  user,
  showEmail = true,
  showPhone = true
}) => (
  <div>
    {showEmail && <p>{user.email}</p>}
    {showPhone && <p>{user.phone}</p>}
  </div>
);

export const UserActions: React.FC<UserActionsProps> = ({ onEdit, onDelete }) => (
  <div>
    <button onClick={onEdit}>Edit</button>
    <button onClick={onDelete}>Delete</button>
  </div>
);

// Compose as needed
export const SimpleUserCard: React.FC<BasicUserCardProps> = ({ user }) => (
  <div>
    <UserAvatar user={user} />
    <UserInfo user={user} />
  </div>
);

export const DetailedUserCard: React.FC<
  BasicUserCardProps & UserContactProps & UserActionsProps
> = ({ user, showEmail, showPhone, onEdit, onDelete }) => (
  <div>
    <UserAvatar user={user} />
    <UserInfo user={user} />
    <UserContact user={user} showEmail={showEmail} showPhone={showPhone} />
    <UserActions onEdit={onEdit} onDelete={onDelete} />
  </div>
);
```

### Angular Service Example

```typescript
// ❌ BAD: Monolithic service interface
interface DataService {
  // User operations
  getUser(id: string): Observable<User>;
  updateUser(user: User): Observable<User>;
  deleteUser(id: string): Observable<void>;
  
  // Order operations
  getOrders(userId: string): Observable<Order[]>;
  createOrder(order: Order): Observable<Order>;
  
  // Analytics operations
  trackEvent(event: string): void;
  getAnalytics(): Observable<Analytics>;
  
  // Notification operations
  sendEmail(to: string, subject: string): Observable<void>;
  sendSMS(to: string, message: string): Observable<void>;
}

// Components forced to depend on entire interface
class UserProfileComponent {
  constructor(private dataService: DataService) {}
  // Only needs user operations but depends on everything
}

// ✅ GOOD: Segregated service interfaces
interface UserService {
  getUser(id: string): Observable<User>;
  updateUser(user: User): Observable<User>;
  deleteUser(id: string): Observable<void>;
}

interface OrderService {
  getOrders(userId: string): Observable<Order[]>;
  createOrder(order: Order): Observable<Order>;
}

interface AnalyticsService {
  trackEvent(event: string): void;
  getAnalytics(): Observable<Analytics>;
}

interface NotificationService {
  sendEmail(to: string, subject: string): Observable<void>;
  sendSMS(to: string, message: string): Observable<void>;
}

// Components depend only on what they need
@Component({ selector: 'user-profile' })
class UserProfileComponent {
  constructor(
    private userService: UserService,
    private analyticsService: AnalyticsService
  ) {}
  // Depends only on required services
}

@Component({ selector: 'order-list' })
class OrderListComponent {
  constructor(private orderService: OrderService) {}
  // Only depends on order operations
}
```

## Dependency Inversion Principle (DIP)

**Depend on abstractions, not concretions.**

### React Example with Dependency Injection

```tsx
// ❌ BAD: Component depends on concrete implementation
export const UserList: React.FC = () => {
  const [users, setUsers] = useState<User[]>([]);

  useEffect(() => {
    // Directly depends on fetch API and specific endpoint
    fetch('https://api.example.com/users')
      .then(res => res.json())
      .then(setUsers);
  }, []);

  return (
    <ul>
      {users.map(user => <li key={user.id}>{user.name}</li>)}
    </ul>
  );
};

// Hard to test, can't swap implementations

// ✅ GOOD: Depend on abstraction (interface)
interface UserRepository {
  getUsers(): Promise<User[]>;
}

// Concrete implementation
class ApiUserRepository implements UserRepository {
  async getUsers(): Promise<User[]> {
    const response = await fetch('https://api.example.com/users');
    return response.json();
  }
}

// Mock implementation for testing
class MockUserRepository implements UserRepository {
  async getUsers(): Promise<User[]> {
    return [
      { id: '1', name: 'John Doe' },
      { id: '2', name: 'Jane Smith' }
    ];
  }
}

// Component depends on abstraction
export const UserList: React.FC<{ repository: UserRepository }> = ({ repository }) => {
  const [users, setUsers] = useState<User[]>([]);

  useEffect(() => {
    repository.getUsers().then(setUsers);
  }, [repository]);

  return (
    <ul>
      {users.map(user => <li key={user.id}>{user.name}</li>)}
    </ul>
  );
};

// Inject dependency
const apiRepository = new ApiUserRepository();
<UserList repository={apiRepository} />

// Easy to test with mock
const mockRepository = new MockUserRepository();
<UserList repository={mockRepository} />

// Or use Context for dependency injection
const RepositoryContext = createContext<UserRepository | null>(null);

export const RepositoryProvider: React.FC<{
  repository: UserRepository;
  children: React.ReactNode;
}> = ({ repository, children }) => (
  <RepositoryContext.Provider value={repository}>
    {children}
  </RepositoryContext.Provider>
);

export const useRepository = () => {
  const context = useContext(RepositoryContext);
  if (!context) throw new Error('useRepository must be within RepositoryProvider');
  return context;
};

// Component gets repository from context
export const UserListWithContext: React.FC = () => {
  const repository = useRepository();
  const [users, setUsers] = useState<User[]>([]);

  useEffect(() => {
    repository.getUsers().then(setUsers);
  }, [repository]);

  return (
    <ul>
      {users.map(user => <li key={user.id}>{user.name}</li>)}
    </ul>
  );
};

// App composition
<RepositoryProvider repository={new ApiUserRepository()}>
  <UserListWithContext />
</RepositoryProvider>
```

### Angular Dependency Injection

```typescript
// ❌ BAD: Concrete dependency
@Component({ selector: 'user-list' })
export class UserListComponent implements OnInit {
  users: User[] = [];

  ngOnInit() {
    // Concrete dependency on HttpClient and URL
    this.http.get<User[]>('https://api.example.com/users')
      .subscribe(users => this.users = users);
  }

  constructor(private http: HttpClient) {}
}

// ✅ GOOD: Depend on abstraction
// Define abstraction
export abstract class UserRepository {
  abstract getUsers(): Observable<User[]>;
  abstract getUser(id: string): Observable<User>;
}

// Concrete implementation
@Injectable()
export class ApiUserRepository extends UserRepository {
  constructor(private http: HttpClient) {
    super();
  }

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>('https://api.example.com/users');
  }

  getUser(id: string): Observable<User> {
    return this.http.get<User>(`https://api.example.com/users/${id}`);
  }
}

// Mock implementation
@Injectable()
export class MockUserRepository extends UserRepository {
  getUsers(): Observable<User[]> {
    return of([
      { id: '1', name: 'John Doe' },
      { id: '2', name: 'Jane Smith' }
    ]);
  }

  getUser(id: string): Observable<User> {
    return of({ id, name: 'Mock User' });
  }
}

// Component depends on abstraction
@Component({
  selector: 'user-list',
  template: `
    <ul>
      <li *ngFor="let user of users">{{ user.name }}</li>
    </ul>
  `
})
export class UserListComponent implements OnInit {
  users: User[] = [];

  // Depends on abstraction, not concrete class
  constructor(private userRepository: UserRepository) {}

  ngOnInit() {
    this.userRepository.getUsers()
      .subscribe(users => this.users = users);
  }
}

// Configure in module
@NgModule({
  providers: [
    {
      provide: UserRepository,
      useClass: environment.production ? ApiUserRepository : MockUserRepository
    }
  ]
})
export class AppModule {}
```

## SOLID Benefits in Frontend

```typescript
// With SOLID principles:

// 1. Easier to test
const mockRepository = new MockUserRepository();
const component = new UserList({ repository: mockRepository });

// 2. Easier to extend
class CachedUserRepository implements UserRepository {
  constructor(private inner: UserRepository) {}
  
  async getUsers(): Promise<User[]> {
    const cached = localStorage.getItem('users');
    if (cached) return JSON.parse(cached);
    
    const users = await this.inner.getUsers();
    localStorage.setItem('users', JSON.stringify(users));
    return users;
  }
}

// 3. Easier to maintain
// Each component/service has single responsibility
// Changes don't ripple through codebase

// 4. Easier to understand
// Small, focused components with clear purpose
// Dependencies explicitly declared

// 5. More flexible
// Can swap implementations without changing consumers
```

## Common Violations and Fixes

### Violation: God Component

```tsx
// ❌ Violates SRP
const Dashboard = () => {
  // Fetching, business logic, validation, rendering all in one
  return <div>Everything</div>;
};

// ✅ Fix: Separate concerns
const Dashboard = () => {
  const { user } = useUser();
  const { orders } = useOrders();
  return (
    <>
      <UserHeader user={user} />
      <OrderList orders={orders} />
    </>
  );
};
```

### Violation: Tight Coupling

```tsx
// ❌ Violates DIP
const Component = () => {
  const data = fetch('/api/data'); // Concrete dependency
};

// ✅ Fix: Inject dependency
const Component = ({ repository }) => {
  const data = repository.getData(); // Abstract dependency
};
```

## Interview Questions

1. **Explain the Single Responsibility Principle**
   - A class/component should have only one reason to change. Each unit should do one thing well.

2. **How does Open/Closed Principle apply to React components?**
   - Components should be extendable through composition and props without modifying their code.

3. **What is the Liskov Substitution Principle?**
   - Subtypes must be substitutable for their base types without altering program correctness.

4. **Why is Interface Segregation important?**
   - Components shouldn't depend on interfaces they don't use. Smaller, focused interfaces are better than large, monolithic ones.

5. **How do you apply Dependency Inversion in frontend?**
   - Depend on abstractions (interfaces) rather than concrete implementations. Inject dependencies rather than creating them.

## Key Takeaways

1. SRP: One component, one responsibility
2. OCP: Extend through composition, not modification
3. LSP: Subtypes must be substitutable for base types
4. ISP: Small, focused interfaces over large ones
5. DIP: Depend on abstractions, inject dependencies
6. SOLID makes code more maintainable and testable
7. Apply pragmatically - not every function needs all principles
8. Composition over inheritance
9. Dependency injection enables testing and flexibility
10. Small, focused components are easier to understand

## Resources

- **Clean Code** by Robert C. Martin
- **SOLID Principles** by Uncle Bob: https://blog.cleancoder.com/
- **React Patterns**: https://reactpatterns.com/
- **Angular Architecture**: https://angular.io/guide/architecture
- **Refactoring Guru - SOLID**: https://refactoring.guru/design-patterns/solid-principles
