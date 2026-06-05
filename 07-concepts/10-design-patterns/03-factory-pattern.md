# Factory Pattern

## The Idea

**In plain English:** A factory pattern is a way of writing code so that instead of building things yourself directly, you ask a special "factory" function to build the right thing for you based on what you need. Think of it like telling a vending machine what you want — you don't build the drink yourself, you just press a button and the machine figures out which one to give you.

**Real-world analogy:** Imagine a car rental counter at an airport. You walk up and say "I need a small car" or "I need an SUV." The rental agent picks the right car from the lot and hands you the keys — you never have to go find the car yourself or know exactly where it is parked.

- The rental counter = the factory (the one place you go to get what you need)
- The types of car (small, SUV, van) = the different classes or objects the factory can create
- Saying "I need a small car" = passing a parameter to the factory method
- The car you drive away in = the object the factory returns to your code

---

## Overview

The Factory pattern is a creational design pattern that provides an interface for creating objects without specifying their exact classes. Instead of calling constructors directly, you call a factory method that decides which class to instantiate based on input parameters or application context.

In frontend development, factories are commonly used for creating components, services, validators, and adapters, promoting loose coupling and making code more maintainable and testable.

## Core Concepts

### 1. Factory Pattern Structure

```
┌─────────────────────────────────────────┐
│            Creator (Factory)            │
├─────────────────────────────────────────┤
│  + factoryMethod(): Product            │
│  + businessMethod(): void              │
└─────────────────┬───────────────────────┘
                  │ creates
                  ↓
┌─────────────────────────────────────────┐
│             Product Interface           │
├─────────────────────────────────────────┤
│  + operation(): void                   │
└─────────────┬───────────────────────────┘
              │
     ┌────────┴────────┐
     ↓                 ↓
┌──────────┐    ┌──────────┐
│ ConcreteA │    │ ConcreteB│
└──────────┘    └──────────┘
```

### 2. Factory Pattern Types

- **Simple Factory**: Single factory method choosing between classes
- **Factory Method**: Subclasses decide which class to instantiate
- **Abstract Factory**: Family of related objects without concrete classes

## Implementation Patterns

### Simple Factory

```typescript
// button.types.ts
export interface Button {
  render(): HTMLElement;
  onClick(handler: () => void): void;
}

export class PrimaryButton implements Button {
  render(): HTMLElement {
    const button = document.createElement('button');
    button.className = 'btn btn-primary';
    button.textContent = 'Primary';
    return button;
  }

  onClick(handler: () => void): void {
    // Attach handler
  }
}

export class SecondaryButton implements Button {
  render(): HTMLElement {
    const button = document.createElement('button');
    button.className = 'btn btn-secondary';
    button.textContent = 'Secondary';
    return button;
  }

  onClick(handler: () => void): void {
    // Attach handler
  }
}

export class DangerButton implements Button {
  render(): HTMLElement {
    const button = document.createElement('button');
    button.className = 'btn btn-danger';
    button.textContent = 'Danger';
    return button;
  }

  onClick(handler: () => void): void {
    // Attach handler
  }
}

// button-factory.ts
export type ButtonType = 'primary' | 'secondary' | 'danger';

export class ButtonFactory {
  static createButton(type: ButtonType): Button {
    switch (type) {
      case 'primary':
        return new PrimaryButton();
      case 'secondary':
        return new SecondaryButton();
      case 'danger':
        return new DangerButton();
      default:
        throw new Error(`Unknown button type: ${type}`);
    }
  }
}

// Usage
const button = ButtonFactory.createButton('primary');
const element = button.render();
document.body.appendChild(element);
```

### React Component Factory

```typescript
// notification.types.ts
export interface NotificationProps {
  message: string;
  onClose: () => void;
}

export type NotificationType = 'success' | 'error' | 'warning' | 'info';

// notification-components.tsx
export const SuccessNotification: React.FC<NotificationProps> = ({ 
  message, 
  onClose 
}) => (
  <div className="notification success">
    <span className="icon">✓</span>
    <span className="message">{message}</span>
    <button onClick={onClose}>×</button>
  </div>
);

export const ErrorNotification: React.FC<NotificationProps> = ({ 
  message, 
  onClose 
}) => (
  <div className="notification error">
    <span className="icon">✗</span>
    <span className="message">{message}</span>
    <button onClick={onClose}>×</button>
  </div>
);

export const WarningNotification: React.FC<NotificationProps> = ({ 
  message, 
  onClose 
}) => (
  <div className="notification warning">
    <span className="icon">⚠</span>
    <span className="message">{message}</span>
    <button onClick={onClose}>×</button>
  </div>
);

export const InfoNotification: React.FC<NotificationProps> = ({ 
  message, 
  onClose 
}) => (
  <div className="notification info">
    <span className="icon">ℹ</span>
    <span className="message">{message}</span>
    <button onClick={onClose}>×</button>
  </div>
);

// notification-factory.tsx
export class NotificationFactory {
  static create(
    type: NotificationType, 
    props: NotificationProps
  ): React.ReactElement {
    switch (type) {
      case 'success':
        return <SuccessNotification {...props} />;
      case 'error':
        return <ErrorNotification {...props} />;
      case 'warning':
        return <WarningNotification {...props} />;
      case 'info':
        return <InfoNotification {...props} />;
      default:
        throw new Error(`Unknown notification type: ${type}`);
    }
  }
}

// Usage in component
function App() {
  const [notifications, setNotifications] = useState<Array<{
    id: string;
    type: NotificationType;
    message: string;
  }>>([]);

  const addNotification = (type: NotificationType, message: string) => {
    const id = crypto.randomUUID();
    setNotifications(prev => [...prev, { id, type, message }]);
  };

  const removeNotification = (id: string) => {
    setNotifications(prev => prev.filter(n => n.id !== id));
  };

  return (
    <div>
      <button onClick={() => addNotification('success', 'Operation successful!')}>
        Show Success
      </button>
      
      <div className="notifications">
        {notifications.map(({ id, type, message }) => (
          <div key={id}>
            {NotificationFactory.create(type, {
              message,
              onClose: () => removeNotification(id)
            })}
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Validator Factory

```typescript
// validator.interface.ts
export interface Validator<T = any> {
  validate(value: T): ValidationResult;
}

export interface ValidationResult {
  valid: boolean;
  errors: string[];
}

// validators.ts
export class EmailValidator implements Validator<string> {
  validate(value: string): ValidationResult {
    const valid = /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value);
    return {
      valid,
      errors: valid ? [] : ['Invalid email format']
    };
  }
}

export class PasswordValidator implements Validator<string> {
  constructor(private minLength = 8) {}

  validate(value: string): ValidationResult {
    const errors: string[] = [];

    if (value.length < this.minLength) {
      errors.push(`Password must be at least ${this.minLength} characters`);
    }
    if (!/[A-Z]/.test(value)) {
      errors.push('Password must contain uppercase letter');
    }
    if (!/[a-z]/.test(value)) {
      errors.push('Password must contain lowercase letter');
    }
    if (!/[0-9]/.test(value)) {
      errors.push('Password must contain number');
    }

    return {
      valid: errors.length === 0,
      errors
    };
  }
}

export class URLValidator implements Validator<string> {
  validate(value: string): ValidationResult {
    try {
      new URL(value);
      return { valid: true, errors: [] };
    } catch {
      return { valid: false, errors: ['Invalid URL format'] };
    }
  }
}

export class PhoneValidator implements Validator<string> {
  validate(value: string): ValidationResult {
    const valid = /^\+?[1-9]\d{1,14}$/.test(value);
    return {
      valid,
      errors: valid ? [] : ['Invalid phone number format']
    };
  }
}

// validator-factory.ts
export type ValidatorType = 'email' | 'password' | 'url' | 'phone';

export class ValidatorFactory {
  static create(type: ValidatorType, options?: any): Validator {
    switch (type) {
      case 'email':
        return new EmailValidator();
      case 'password':
        return new PasswordValidator(options?.minLength);
      case 'url':
        return new URLValidator();
      case 'phone':
        return new PhoneValidator();
      default:
        throw new Error(`Unknown validator type: ${type}`);
    }
  }
}

// Usage
const emailValidator = ValidatorFactory.create('email');
const result = emailValidator.validate('user@example.com');

if (!result.valid) {
  console.error(result.errors);
}

const passwordValidator = ValidatorFactory.create('password', { minLength: 10 });
const pwResult = passwordValidator.validate('weak');
console.log(pwResult.errors); // Lists all validation errors
```

### HTTP Client Factory

```typescript
// http-client.interface.ts
export interface HTTPClient {
  get<T>(url: string, config?: RequestConfig): Promise<T>;
  post<T>(url: string, data: any, config?: RequestConfig): Promise<T>;
  put<T>(url: string, data: any, config?: RequestConfig): Promise<T>;
  delete<T>(url: string, config?: RequestConfig): Promise<T>;
}

export interface RequestConfig {
  headers?: Record<string, string>;
  timeout?: number;
}

// fetch-client.ts
export class FetchClient implements HTTPClient {
  constructor(private baseURL: string) {}

  async get<T>(url: string, config?: RequestConfig): Promise<T> {
    const response = await fetch(this.baseURL + url, {
      method: 'GET',
      headers: config?.headers,
      signal: this.createTimeout(config?.timeout)
    });
    return response.json();
  }

  async post<T>(url: string, data: any, config?: RequestConfig): Promise<T> {
    const response = await fetch(this.baseURL + url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', ...config?.headers },
      body: JSON.stringify(data),
      signal: this.createTimeout(config?.timeout)
    });
    return response.json();
  }

  async put<T>(url: string, data: any, config?: RequestConfig): Promise<T> {
    const response = await fetch(this.baseURL + url, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json', ...config?.headers },
      body: JSON.stringify(data),
      signal: this.createTimeout(config?.timeout)
    });
    return response.json();
  }

  async delete<T>(url: string, config?: RequestConfig): Promise<T> {
    const response = await fetch(this.baseURL + url, {
      method: 'DELETE',
      headers: config?.headers,
      signal: this.createTimeout(config?.timeout)
    });
    return response.json();
  }

  private createTimeout(timeout?: number): AbortSignal | undefined {
    if (!timeout) return undefined;
    const controller = new AbortController();
    setTimeout(() => controller.abort(), timeout);
    return controller.signal;
  }
}

// axios-client.ts
import axios, { AxiosInstance } from 'axios';

export class AxiosClient implements HTTPClient {
  private client: AxiosInstance;

  constructor(baseURL: string) {
    this.client = axios.create({ baseURL });
  }

  async get<T>(url: string, config?: RequestConfig): Promise<T> {
    const response = await this.client.get(url, {
      headers: config?.headers,
      timeout: config?.timeout
    });
    return response.data;
  }

  async post<T>(url: string, data: any, config?: RequestConfig): Promise<T> {
    const response = await this.client.post(url, data, {
      headers: config?.headers,
      timeout: config?.timeout
    });
    return response.data;
  }

  async put<T>(url: string, data: any, config?: RequestConfig): Promise<T> {
    const response = await this.client.put(url, data, {
      headers: config?.headers,
      timeout: config?.timeout
    });
    return response.data;
  }

  async delete<T>(url: string, config?: RequestConfig): Promise<T> {
    const response = await this.client.delete(url, {
      headers: config?.headers,
      timeout: config?.timeout
    });
    return response.data;
  }
}

// http-client-factory.ts
export type ClientType = 'fetch' | 'axios';

export class HTTPClientFactory {
  static create(type: ClientType, baseURL: string): HTTPClient {
    switch (type) {
      case 'fetch':
        return new FetchClient(baseURL);
      case 'axios':
        return new AxiosClient(baseURL);
      default:
        throw new Error(`Unknown client type: ${type}`);
    }
  }
}

// Usage
const client = HTTPClientFactory.create('fetch', 'https://api.example.com');
const users = await client.get<User[]>('/users');

// Easy to switch implementation
const axiosClient = HTTPClientFactory.create('axios', 'https://api.example.com');
```

### Abstract Factory Pattern

```typescript
// ui-theme.interface.ts
export interface Button {
  render(): HTMLElement;
}

export interface Input {
  render(): HTMLElement;
}

export interface Card {
  render(): HTMLElement;
}

export interface UIThemeFactory {
  createButton(text: string): Button;
  createInput(placeholder: string): Input;
  createCard(title: string, content: string): Card;
}

// light-theme.ts
class LightButton implements Button {
  constructor(private text: string) {}
  
  render(): HTMLElement {
    const button = document.createElement('button');
    button.textContent = this.text;
    button.style.backgroundColor = '#ffffff';
    button.style.color = '#000000';
    button.style.border = '1px solid #cccccc';
    return button;
  }
}

class LightInput implements Input {
  constructor(private placeholder: string) {}
  
  render(): HTMLElement {
    const input = document.createElement('input');
    input.placeholder = this.placeholder;
    input.style.backgroundColor = '#ffffff';
    input.style.color = '#000000';
    input.style.border = '1px solid #cccccc';
    return input;
  }
}

class LightCard implements Card {
  constructor(private title: string, private content: string) {}
  
  render(): HTMLElement {
    const card = document.createElement('div');
    card.style.backgroundColor = '#ffffff';
    card.style.border = '1px solid #e0e0e0';
    card.innerHTML = `
      <h3 style="color: #000000">${this.title}</h3>
      <p style="color: #333333">${this.content}</p>
    `;
    return card;
  }
}

export class LightThemeFactory implements UIThemeFactory {
  createButton(text: string): Button {
    return new LightButton(text);
  }

  createInput(placeholder: string): Input {
    return new LightInput(placeholder);
  }

  createCard(title: string, content: string): Card {
    return new LightCard(title, content);
  }
}

// dark-theme.ts
class DarkButton implements Button {
  constructor(private text: string) {}
  
  render(): HTMLElement {
    const button = document.createElement('button');
    button.textContent = this.text;
    button.style.backgroundColor = '#2d2d2d';
    button.style.color = '#ffffff';
    button.style.border = '1px solid #404040';
    return button;
  }
}

class DarkInput implements Input {
  constructor(private placeholder: string) {}
  
  render(): HTMLElement {
    const input = document.createElement('input');
    input.placeholder = this.placeholder;
    input.style.backgroundColor = '#2d2d2d';
    input.style.color = '#ffffff';
    input.style.border = '1px solid #404040';
    return input;
  }
}

class DarkCard implements Card {
  constructor(private title: string, private content: string) {}
  
  render(): HTMLElement {
    const card = document.createElement('div');
    card.style.backgroundColor = '#1e1e1e';
    card.style.border = '1px solid #404040';
    card.innerHTML = `
      <h3 style="color: #ffffff">${this.title}</h3>
      <p style="color: #cccccc">${this.content}</p>
    `;
    return card;
  }
}

export class DarkThemeFactory implements UIThemeFactory {
  createButton(text: string): Button {
    return new DarkButton(text);
  }

  createInput(placeholder: string): Input {
    return new DarkInput(placeholder);
  }

  createCard(title: string, content: string): Card {
    return new DarkCard(title, content);
  }
}

// theme-factory.ts
export type Theme = 'light' | 'dark';

export class ThemeFactory {
  static create(theme: Theme): UIThemeFactory {
    switch (theme) {
      case 'light':
        return new LightThemeFactory();
      case 'dark':
        return new DarkThemeFactory();
      default:
        throw new Error(`Unknown theme: ${theme}`);
    }
  }
}

// Usage
const theme = ThemeFactory.create('dark');
const button = theme.createButton('Click me');
const input = theme.createInput('Enter text');
const card = theme.createCard('Title', 'Content');

document.body.appendChild(button.render());
document.body.appendChild(input.render());
document.body.appendChild(card.render());
```

### Angular Factory Service

```typescript
// logger.interface.ts
export interface Logger {
  log(message: string): void;
  error(message: string): void;
}

// console-logger.ts
export class ConsoleLogger implements Logger {
  log(message: string): void {
    console.log(`[LOG] ${message}`);
  }

  error(message: string): void {
    console.error(`[ERROR] ${message}`);
  }
}

// remote-logger.ts
export class RemoteLogger implements Logger {
  constructor(private apiUrl: string) {}

  log(message: string): void {
    fetch(`${this.apiUrl}/logs`, {
      method: 'POST',
      body: JSON.stringify({ level: 'log', message })
    });
  }

  error(message: string): void {
    fetch(`${this.apiUrl}/logs`, {
      method: 'POST',
      body: JSON.stringify({ level: 'error', message })
    });
  }
}

// logger.factory.ts
import { Injectable } from '@angular/core';
import { environment } from '../environments/environment';

@Injectable({
  providedIn: 'root'
})
export class LoggerFactory {
  create(): Logger {
    if (environment.production) {
      return new RemoteLogger(environment.logApiUrl);
    } else {
      return new ConsoleLogger();
    }
  }
}

// Component usage
@Component({
  selector: 'app-user-list',
  template: `<div>{{ users.length }} users</div>`
})
export class UserListComponent implements OnInit {
  private logger: Logger;
  users: User[] = [];

  constructor(
    private loggerFactory: LoggerFactory,
    private userService: UserService
  ) {
    this.logger = this.loggerFactory.create();
  }

  ngOnInit(): void {
    this.logger.log('UserListComponent initialized');
    
    this.userService.getUsers().subscribe({
      next: users => {
        this.users = users;
        this.logger.log(`Loaded ${users.length} users`);
      },
      error: error => {
        this.logger.error(`Failed to load users: ${error.message}`);
      }
    });
  }
}
```

### Dynamic Component Factory (React)

```typescript
// chart.types.ts
export interface ChartProps {
  data: number[];
  width: number;
  height: number;
}

export type ChartType = 'bar' | 'line' | 'pie' | 'scatter';

// chart-components.tsx
export const BarChart: React.FC<ChartProps> = ({ data, width, height }) => (
  <svg width={width} height={height}>
    {data.map((value, index) => (
      <rect
        key={index}
        x={index * 50}
        y={height - value}
        width={40}
        height={value}
        fill="blue"
      />
    ))}
  </svg>
);

export const LineChart: React.FC<ChartProps> = ({ data, width, height }) => {
  const points = data.map((value, index) => 
    `${index * 50},${height - value}`
  ).join(' ');

  return (
    <svg width={width} height={height}>
      <polyline
        points={points}
        fill="none"
        stroke="blue"
        strokeWidth="2"
      />
    </svg>
  );
};

export const PieChart: React.FC<ChartProps> = ({ data, width, height }) => {
  const total = data.reduce((sum, val) => sum + val, 0);
  let currentAngle = 0;

  return (
    <svg width={width} height={height}>
      {data.map((value, index) => {
        const angle = (value / total) * 360;
        const slice = (
          <circle
            key={index}
            cx={width / 2}
            cy={height / 2}
            r={Math.min(width, height) / 3}
            fill={`hsl(${currentAngle}, 70%, 50%)`}
          />
        );
        currentAngle += angle;
        return slice;
      })}
    </svg>
  );
};

// chart-factory.tsx
const chartComponents: Record<ChartType, React.ComponentType<ChartProps>> = {
  bar: BarChart,
  line: LineChart,
  pie: PieChart,
  scatter: BarChart // Placeholder
};

export class ChartFactory {
  static create(type: ChartType, props: ChartProps): React.ReactElement {
    const ChartComponent = chartComponents[type];
    
    if (!ChartComponent) {
      throw new Error(`Unknown chart type: ${type}`);
    }

    return <ChartComponent {...props} />;
  }
}

// Usage
function Dashboard() {
  const [chartType, setChartType] = useState<ChartType>('bar');
  const data = [10, 20, 30, 40, 50];

  return (
    <div>
      <select value={chartType} onChange={e => setChartType(e.target.value as ChartType)}>
        <option value="bar">Bar Chart</option>
        <option value="line">Line Chart</option>
        <option value="pie">Pie Chart</option>
      </select>

      {ChartFactory.create(chartType, {
        data,
        width: 400,
        height: 300
      })}
    </div>
  );
}
```

### Registry-Based Factory

```typescript
// registry-factory.ts
type Constructor<T> = new (...args: any[]) => T;

export class RegistryFactory<T> {
  private registry = new Map<string, Constructor<T>>();

  register(name: string, constructor: Constructor<T>): void {
    if (this.registry.has(name)) {
      throw new Error(`Type '${name}' already registered`);
    }
    this.registry.set(name, constructor);
  }

  create(name: string, ...args: any[]): T {
    const Constructor = this.registry.get(name);
    
    if (!Constructor) {
      throw new Error(`Type '${name}' not registered`);
    }

    return new Constructor(...args);
  }

  isRegistered(name: string): boolean {
    return this.registry.has(name);
  }

  getRegisteredTypes(): string[] {
    return Array.from(this.registry.keys());
  }
}

// Usage: Payment processor factory
interface PaymentProcessor {
  process(amount: number): Promise<void>;
}

class StripeProcessor implements PaymentProcessor {
  async process(amount: number): Promise<void> {
    console.log(`Processing $${amount} with Stripe`);
  }
}

class PayPalProcessor implements PaymentProcessor {
  async process(amount: number): Promise<void> {
    console.log(`Processing $${amount} with PayPal`);
  }
}

const paymentFactory = new RegistryFactory<PaymentProcessor>();
paymentFactory.register('stripe', StripeProcessor);
paymentFactory.register('paypal', PayPalProcessor);

const processor = paymentFactory.create('stripe');
await processor.process(99.99);
```

### Parameterized Factory

```typescript
// modal-factory.ts
export interface ModalConfig {
  title: string;
  content: string;
  width?: number;
  height?: number;
  closable?: boolean;
  footer?: React.ReactNode;
}

export class ModalFactory {
  static createModal(config: ModalConfig): React.ReactElement {
    const {
      title,
      content,
      width = 500,
      height = 300,
      closable = true,
      footer
    } = config;

    return (
      <div className="modal" style={{ width, height }}>
        <div className="modal-header">
          <h2>{title}</h2>
          {closable && <button className="close">×</button>}
        </div>
        <div className="modal-body">
          {content}
        </div>
        {footer && <div className="modal-footer">{footer}</div>}
      </div>
    );
  }

  static createConfirmModal(
    message: string,
    onConfirm: () => void,
    onCancel: () => void
  ): React.ReactElement {
    return this.createModal({
      title: 'Confirm',
      content: message,
      footer: (
        <>
          <button onClick={onCancel}>Cancel</button>
          <button onClick={onConfirm}>Confirm</button>
        </>
      )
    });
  }

  static createAlertModal(message: string, onClose: () => void): React.ReactElement {
    return this.createModal({
      title: 'Alert',
      content: message,
      footer: <button onClick={onClose}>OK</button>
    });
  }
}

// Usage
const confirmModal = ModalFactory.createConfirmModal(
  'Are you sure you want to delete this item?',
  () => console.log('Confirmed'),
  () => console.log('Cancelled')
);
```

## Common Mistakes

### 1. Factory Doing Too Much

```typescript
// BAD: Factory with business logic
class UserFactory {
  static async createUser(data: UserData): Promise<User> {
    // Validation
    if (!data.email) throw new Error('Email required');
    
    // Database check
    const existing = await db.users.findOne({ email: data.email });
    if (existing) throw new Error('Email exists');
    
    // Hash password
    const hashedPassword = await bcrypt.hash(data.password, 10);
    
    // Create user
    const user = new User({
      ...data,
      password: hashedPassword
    });
    
    // Save to database
    await user.save();
    
    // Send welcome email
    await emailService.send(user.email, 'Welcome!');
    
    return user;
  }
}

// GOOD: Factory just creates instances
class UserFactory {
  static createUser(data: UserData): User {
    return new User(data);
  }
}

// Business logic in service
class UserService {
  async registerUser(data: UserData): Promise<User> {
    await this.validateUserData(data);
    const user = UserFactory.createUser(data);
    await this.userRepository.save(user);
    await this.emailService.sendWelcome(user);
    return user;
  }
}
```

### 2. Not Using Interfaces

```typescript
// BAD: Factory returns concrete types
class PaymentFactory {
  static create(type: string): StripeProcessor | PayPalProcessor {
    if (type === 'stripe') return new StripeProcessor();
    return new PayPalProcessor();
  }
}

// Client code coupled to concrete types
const processor = PaymentFactory.create('stripe');
if (processor instanceof StripeProcessor) {
  processor.stripeSpecificMethod(); // Tight coupling!
}

// GOOD: Factory returns interface
interface PaymentProcessor {
  process(amount: number): Promise<void>;
}

class PaymentFactory {
  static create(type: string): PaymentProcessor {
    if (type === 'stripe') return new StripeProcessor();
    return new PayPalProcessor();
  }
}

// Client code works with interface
const processor = PaymentFactory.create('stripe');
await processor.process(100); // No coupling to concrete type
```

### 3. No Error Handling

```typescript
// BAD: No validation
class ComponentFactory {
  static create(type: string) {
    return new components[type](); // Crashes if type doesn't exist
  }
}

// GOOD: Proper error handling
class ComponentFactory {
  static create(type: string): Component {
    const ComponentClass = components[type];
    
    if (!ComponentClass) {
      throw new Error(
        `Unknown component type: ${type}. ` +
        `Available types: ${Object.keys(components).join(', ')}`
      );
    }

    try {
      return new ComponentClass();
    } catch (error) {
      throw new Error(`Failed to create component '${type}': ${error.message}`);
    }
  }
}
```

### 4. Overusing Factory

```typescript
// BAD: Factory for simple cases
class StringFactory {
  static createString(value: string): string {
    return new String(value).toString();
  }
}

const str = StringFactory.createString('hello'); // Overkill!

// GOOD: Direct instantiation
const str = 'hello';

// Use factory when:
// - Complex creation logic
// - Multiple implementations
// - Creation depends on runtime conditions
```

## Best Practices

### 1. Use TypeScript Generics

```typescript
class Factory<T> {
  private registry = new Map<string, new () => T>();

  register(name: string, constructor: new () => T): void {
    this.registry.set(name, constructor);
  }

  create(name: string): T {
    const Constructor = this.registry.get(name);
    if (!Constructor) {
      throw new Error(`Type '${name}' not registered`);
    }
    return new Constructor();
  }
}
```

### 2. Provide Default Options

```typescript
class ConfigurableFactory {
  static create(type: string, options: Partial<Options> = {}): Product {
    const defaults: Options = {
      width: 100,
      height: 100,
      color: 'blue'
    };

    const finalOptions = { ...defaults, ...options };
    return new Product(type, finalOptions);
  }
}
```

### 3. Document Factory Usage

```typescript
/**
 * Factory for creating notification components
 * 
 * @example
 * ```typescript
 * const notification = NotificationFactory.create('success', {
 *   message: 'Operation completed',
 *   onClose: () => console.log('Closed')
 * });
 * ```
 */
export class NotificationFactory {
  // Implementation
}
```

### 4. Make Factories Extensible

```typescript
// Allow registering custom types
export class ExtensibleFactory {
  private static customTypes = new Map<string, Constructor>();

  static registerType(name: string, constructor: Constructor): void {
    this.customTypes.set(name, constructor);
  }

  static create(type: string): Product {
    // Check built-in types
    if (type === 'typeA') return new TypeA();
    
    // Check custom types
    const CustomType = this.customTypes.get(type);
    if (CustomType) return new CustomType();

    throw new Error(`Unknown type: ${type}`);
  }
}
```

## When to Use Factory Pattern

### Use When

1. **Multiple Implementations**: Several classes implementing same interface
2. **Complex Creation**: Object creation requires complex logic
3. **Runtime Decision**: Type to create determined at runtime
4. **Hide Complexity**: Abstract away creation details from client
5. **Centralized Creation**: Want single place for object creation logic

### Don't Use When

1. **Simple Construction**: Object creation is straightforward
2. **Single Implementation**: Only one class, no variations
3. **No Abstraction Needed**: Clients can depend on concrete classes
4. **Overkill**: Pattern adds unnecessary complexity
5. **Performance Critical**: Factory adds minimal overhead but matters

## Interview Questions

### Q1: What's the difference between Factory Method and Abstract Factory?

Answer: Factory Method defines interface for creating one type of object, letting subclasses decide which class to instantiate. Abstract Factory provides interface for creating families of related objects without specifying concrete classes. Factory Method uses inheritance; Abstract Factory uses composition.

### Q2: When should you use Factory pattern vs direct instantiation?

Answer: Use Factory when: multiple implementations exist, creation logic is complex, type is determined at runtime, or you need to hide concrete classes. Use direct instantiation when construction is simple and coupling to concrete class is acceptable.

### Q3: How do you make a factory extensible?

Answer: Use a registry pattern allowing registration of new types at runtime, accept constructor functions as parameters, use plugin architecture, or allow subclassing the factory to add new types.

### Q4: What's the relationship between Factory and Dependency Injection?

Answer: Factory creates objects; DI provides them to dependents. Factories can be injected into classes that need to create objects dynamically. DI containers often use factories internally. They solve different problems but complement each other.

### Q5: How do you test code that uses factories?

Answer: Mock the factory to return test doubles, use dependency injection to provide test factories, create factory methods that accept dependencies, or use test-specific factory implementations.

## Key Takeaways

1. **Factory pattern abstracts object creation** from client code
2. **Use interfaces** to decouple from concrete implementations
3. **Three types**: Simple Factory, Factory Method, Abstract Factory
4. **Centralizes creation logic** making it easier to maintain
5. **Enables polymorphism** - return different types from same interface
6. **Makes code testable** by allowing mock implementations
7. **Don't overuse** - simple cases don't need factories
8. **Registry pattern** allows runtime extensibility
9. **Combine with other patterns** like Singleton or Builder
10. **Document what factories create** and when to use them

## Resources

### Documentation
- [Factory Pattern (Refactoring Guru)](https://refactoring.guru/design-patterns/factory-method)
- [Abstract Factory Pattern](https://refactoring.guru/design-patterns/abstract-factory)
- [Factory Pattern (JavaScript)](https://www.patterns.dev/posts/factory-pattern)

### Articles
- "Factory vs Constructor" - Design Patterns
- "When to Use Factory Pattern" - Clean Code
- "Factory Pattern in React" - Component Design

### Examples
- React Component Factories
- Angular Service Factories
- Payment Processor Factories
- UI Theme Factories
