# Decorator Pattern

## The Idea

**In plain English:** The Decorator pattern is a way to add new abilities to an existing object by wrapping it in another object, without changing the original. Think of it like adding features on top of something without rewriting it from scratch.

**Real-world analogy:** Ordering a coffee at a cafe. You start with a plain black coffee, then the barista can add extras one at a time — milk, sugar, whipped cream — each addition wraps the previous drink and changes its price and description:

- The plain black coffee = the original object (the base component)
- Each add-on (milk, sugar, whip) = a decorator that wraps the coffee and adds new behaviour
- The final drink with all extras = the decorated object with stacked features

---

## Overview

The Decorator pattern is a structural design pattern that allows behavior to be added to individual objects dynamically without affecting other objects from the same class. It provides a flexible alternative to subclassing for extending functionality by wrapping objects with decorator objects that contain the additional behavior.

## Core Concepts

### Decorator Pattern Structure

```
┌──────────────────┐
│   Component      │
│   «interface»    │
├──────────────────┤
│ + operation()    │
└────────┬─────────┘
         △
    ┌────┴────┐
    │         │
┌───┴────────┐│
│  Concrete  ││
│ Component  ││
└────────────┘│
              │
       ┌──────┴──────────┐
       │   Decorator     │
       │  «abstract»     │
       ├─────────────────┤
       │ - component     │◆────┐
       │ + operation()   │     │
       └────────┬────────┘     │
                △               │
          ┌─────┴─────┐         │
          │           │         │
  ┌───────┴────┐ ┌────┴──────┐ │
  │ConcreteA   │ │ConcreteB  │ │
  │Decorator   │ │Decorator  │ │
  └────────────┘ └───────────┘ │
         wraps ─────────────────┘
```

### Key Components

1. **Component**: Interface for objects that can have responsibilities added
2. **ConcreteComponent**: Object to which additional responsibilities can be attached
3. **Decorator**: Abstract decorator maintaining reference to Component
4. **ConcreteDecorator**: Adds responsibilities to the component

## Implementation Patterns

### Basic Decorator Pattern

```typescript
// Component interface
interface Coffee {
  cost(): number;
  description(): string;
}

// Concrete component
class SimpleCoffee implements Coffee {
  cost(): number {
    return 2.0;
  }

  description(): string {
    return 'Simple coffee';
  }
}

// Base decorator
abstract class CoffeeDecorator implements Coffee {
  constructor(protected coffee: Coffee) {}

  cost(): number {
    return this.coffee.cost();
  }

  description(): string {
    return this.coffee.description();
  }
}

// Concrete decorators
class MilkDecorator extends CoffeeDecorator {
  cost(): number {
    return this.coffee.cost() + 0.5;
  }

  description(): string {
    return `${this.coffee.description()}, milk`;
  }
}

class SugarDecorator extends CoffeeDecorator {
  cost(): number {
    return this.coffee.cost() + 0.25;
  }

  description(): string {
    return `${this.coffee.description()}, sugar`;
  }
}

class WhipDecorator extends CoffeeDecorator {
  cost(): number {
    return this.coffee.cost() + 0.75;
  }

  description(): string {
    return `${this.coffee.description()}, whip`;
  }
}

class CaramelDecorator extends CoffeeDecorator {
  cost(): number {
    return this.coffee.cost() + 1.0;
  }

  description(): string {
    return `${this.coffee.description()}, caramel`;
  }
}

// Usage - Build coffee with decorators
let coffee: Coffee = new SimpleCoffee();
console.log(`${coffee.description()} - $${coffee.cost()}`);
// Simple coffee - $2

coffee = new MilkDecorator(coffee);
console.log(`${coffee.description()} - $${coffee.cost()}`);
// Simple coffee, milk - $2.5

coffee = new SugarDecorator(coffee);
console.log(`${coffee.description()} - $${coffee.cost()}`);
// Simple coffee, milk, sugar - $2.75

coffee = new WhipDecorator(coffee);
console.log(`${coffee.description()} - $${coffee.cost()}`);
// Simple coffee, milk, sugar, whip - $3.5

// Create complex coffee in one go
const fancyCoffee = new CaramelDecorator(
  new WhipDecorator(
    new MilkDecorator(
      new SimpleCoffee()
    )
  )
);
console.log(`${fancyCoffee.description()} - $${fancyCoffee.cost()}`);
// Simple coffee, milk, whip, caramel - $4.25
```

### TypeScript Decorators (Experimental Feature)

```typescript
// Class decorator
function sealed(constructor: Function) {
  Object.seal(constructor);
  Object.seal(constructor.prototype);
}

@sealed
class BugReport {
  type = 'report';
  title: string;

  constructor(title: string) {
    this.title = title;
  }
}

// Method decorator
function log(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const originalMethod = descriptor.value;

  descriptor.value = function(...args: any[]) {
    console.log(`Calling ${propertyKey} with`, args);
    const result = originalMethod.apply(this, args);
    console.log(`Result:`, result);
    return result;
  };

  return descriptor;
}

// Property decorator
function readonly(target: any, propertyKey: string) {
  Object.defineProperty(target, propertyKey, {
    writable: false
  });
}

// Parameter decorator
function required(target: any, propertyKey: string, parameterIndex: number) {
  const existingRequiredParams: number[] = 
    Reflect.getOwnMetadata('required', target, propertyKey) || [];
  existingRequiredParams.push(parameterIndex);
  Reflect.defineMetadata('required', existingRequiredParams, target, propertyKey);
}

// Example usage
class Calculator {
  @readonly
  version = '1.0.0';

  @log
  add(a: number, b: number): number {
    return a + b;
  }

  @log
  multiply(@required a: number, @required b: number): number {
    return a * b;
  }
}

const calc = new Calculator();
calc.add(5, 3);        // Logs method call and result
calc.multiply(4, 7);   // Logs method call and result
```

### Performance Monitoring Decorator

```typescript
// Performance decorator
function measure(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
) {
  const originalMethod = descriptor.value;

  descriptor.value = async function(...args: any[]) {
    const start = performance.now();
    
    try {
      const result = await originalMethod.apply(this, args);
      const end = performance.now();
      const duration = end - start;
      
      console.log(`⏱️ ${propertyKey} took ${duration.toFixed(2)}ms`);
      
      return result;
    } catch (error) {
      const end = performance.now();
      const duration = end - start;
      
      console.log(`❌ ${propertyKey} failed after ${duration.toFixed(2)}ms`);
      throw error;
    }
  };

  return descriptor;
}

// Caching decorator
function memoize(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
) {
  const originalMethod = descriptor.value;
  const cache = new Map<string, any>();

  descriptor.value = function(...args: any[]) {
    const key = JSON.stringify(args);
    
    if (cache.has(key)) {
      console.log(`📦 Cache hit for ${propertyKey}`);
      return cache.get(key);
    }
    
    console.log(`🔍 Cache miss for ${propertyKey}`);
    const result = originalMethod.apply(this, args);
    cache.set(key, result);
    
    return result;
  };

  return descriptor;
}

// Retry decorator
function retry(maxAttempts: number = 3, delay: number = 1000) {
  return function(
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    const originalMethod = descriptor.value;

    descriptor.value = async function(...args: any[]) {
      let lastError: Error;
      
      for (let attempt = 1; attempt <= maxAttempts; attempt++) {
        try {
          return await originalMethod.apply(this, args);
        } catch (error) {
          lastError = error as Error;
          console.log(`Attempt ${attempt}/${maxAttempts} failed`);
          
          if (attempt < maxAttempts) {
            await new Promise(resolve => setTimeout(resolve, delay));
          }
        }
      }
      
      throw lastError!;
    };

    return descriptor;
  };
}

// Usage
class DataService {
  @measure
  @memoize
  async fetchUserData(userId: string): Promise<any> {
    // Simulate API call
    await new Promise(resolve => setTimeout(resolve, 1000));
    return { id: userId, name: 'John Doe' };
  }

  @measure
  @retry(3, 500)
  async unreliableOperation(): Promise<string> {
    // Simulate 50% failure rate
    if (Math.random() < 0.5) {
      throw new Error('Operation failed');
    }
    return 'Success';
  }

  @measure
  calculateFibonacci(n: number): number {
    if (n <= 1) return n;
    return this.calculateFibonacci(n - 1) + this.calculateFibonacci(n - 2);
  }
}

// Test the decorators
const service = new DataService();
await service.fetchUserData('123'); // Cache miss, takes ~1000ms
await service.fetchUserData('123'); // Cache hit, instant
await service.unreliableOperation(); // May retry on failure
```

### React Higher-Order Components (Decorators)

```typescript
// withAuth HOC
import React, { ComponentType, useEffect, useState } from 'react';
import { useNavigate } from 'react-router-dom';

interface WithAuthProps {
  isAuthenticated?: boolean;
}

function withAuth<P extends object>(
  Component: ComponentType<P>
): ComponentType<P & WithAuthProps> {
  return function AuthenticatedComponent(props: P & WithAuthProps) {
    const navigate = useNavigate();
    const [isChecking, setIsChecking] = useState(true);

    useEffect(() => {
      const checkAuth = async () => {
        const token = localStorage.getItem('authToken');
        
        if (!token) {
          navigate('/login');
        } else {
          setIsChecking(false);
        }
      };

      checkAuth();
    }, [navigate]);

    if (isChecking) {
      return <div>Checking authentication...</div>;
    }

    return <Component {...props} isAuthenticated={true} />;
  };
}

// withLoading HOC
interface WithLoadingProps {
  isLoading: boolean;
}

function withLoading<P extends object>(
  Component: ComponentType<P>
): ComponentType<P & WithLoadingProps> {
  return function LoadingComponent({ isLoading, ...props }: P & WithLoadingProps) {
    if (isLoading) {
      return (
        <div className="loading-spinner">
          <span>Loading...</span>
        </div>
      );
    }

    return <Component {...(props as P)} />;
  };
}

// withErrorBoundary HOC
interface ErrorBoundaryState {
  hasError: boolean;
  error?: Error;
}

function withErrorBoundary<P extends object>(
  Component: ComponentType<P>
): ComponentType<P> {
  return class ErrorBoundary extends React.Component<P, ErrorBoundaryState> {
    constructor(props: P) {
      super(props);
      this.state = { hasError: false };
    }

    static getDerivedStateFromError(error: Error): ErrorBoundaryState {
      return { hasError: true, error };
    }

    componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
      console.error('Error caught by boundary:', error, errorInfo);
    }

    render() {
      if (this.state.hasError) {
        return (
          <div className="error-boundary">
            <h2>Something went wrong</h2>
            <p>{this.state.error?.message}</p>
            <button onClick={() => this.setState({ hasError: false })}>
              Try again
            </button>
          </div>
        );
      }

      return <Component {...this.props} />;
    }
  };
}

// withAnalytics HOC
function withAnalytics<P extends object>(
  Component: ComponentType<P>,
  eventName: string
): ComponentType<P> {
  return function AnalyticsComponent(props: P) {
    useEffect(() => {
      console.log(`Analytics: Component ${eventName} mounted`);
      
      return () => {
        console.log(`Analytics: Component ${eventName} unmounted`);
      };
    }, []);

    return <Component {...props} />;
  };
}

// Compose multiple HOCs
function compose<P extends object>(
  ...hocs: Array<(component: ComponentType<any>) => ComponentType<any>>
) {
  return (Component: ComponentType<P>) =>
    hocs.reduceRight((acc, hoc) => hoc(acc), Component);
}

// Base component
interface DashboardProps {
  userId: string;
}

const Dashboard: React.FC<DashboardProps> = ({ userId }) => {
  return (
    <div className="dashboard">
      <h1>Dashboard for {userId}</h1>
    </div>
  );
};

// Apply multiple decorators
const EnhancedDashboard = compose<DashboardProps>(
  withAuth,
  withLoading,
  withErrorBoundary,
  (Component) => withAnalytics(Component, 'Dashboard')
)(Dashboard);

// Usage
export const DashboardPage: React.FC = () => {
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    setTimeout(() => setIsLoading(false), 1000);
  }, []);

  return <EnhancedDashboard userId="123" isLoading={isLoading} />;
};
```

### Angular Decorators

```typescript
// Custom validator decorator
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

export function MinAge(minAge: number): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    if (!control.value) {
      return null;
    }

    const birthDate = new Date(control.value);
    const today = new Date();
    const age = today.getFullYear() - birthDate.getFullYear();

    return age >= minAge ? null : { minAge: { minAge, actualAge: age } };
  };
}

// Component decorator factory
import { Component } from '@angular/core';

export function LogLifecycle() {
  return function<T extends { new(...args: any[]): {} }>(constructor: T) {
    const original = constructor.prototype.ngOnInit;
    const originalDestroy = constructor.prototype.ngOnDestroy;

    constructor.prototype.ngOnInit = function() {
      console.log(`${constructor.name} initialized`);
      if (original) {
        original.apply(this);
      }
    };

    constructor.prototype.ngOnDestroy = function() {
      console.log(`${constructor.name} destroyed`);
      if (originalDestroy) {
        originalDestroy.apply(this);
      }
    };

    return constructor;
  };
}

// Method timing decorator
export function LogExecutionTime() {
  return function(
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    const originalMethod = descriptor.value;

    descriptor.value = async function(...args: any[]) {
      const start = performance.now();
      const result = await originalMethod.apply(this, args);
      const end = performance.now();
      
      console.log(`${propertyKey} executed in ${(end - start).toFixed(2)}ms`);
      
      return result;
    };

    return descriptor;
  };
}

// Usage in Angular component
@Component({
  selector: 'app-user-profile',
  template: `
    <form [formGroup]="userForm">
      <input formControlName="birthDate" type="date" />
      <div *ngIf="userForm.get('birthDate')?.errors?.['minAge']">
        Must be at least 18 years old
      </div>
      <button (click)="saveProfile()">Save</button>
    </form>
  `
})
@LogLifecycle()
export class UserProfileComponent {
  userForm = new FormGroup({
    birthDate: new FormControl('', [MinAge(18)])
  });

  @LogExecutionTime()
  async saveProfile(): Promise<void> {
    await new Promise(resolve => setTimeout(resolve, 500));
    console.log('Profile saved');
  }
}
```

### Function Composition Decorators

```typescript
// Function decorator utilities
type AnyFunction = (...args: any[]) => any;

// Before decorator
function before(beforeFn: AnyFunction) {
  return function(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;

    descriptor.value = function(...args: any[]) {
      beforeFn.apply(this, args);
      return originalMethod.apply(this, args);
    };

    return descriptor;
  };
}

// After decorator
function after(afterFn: AnyFunction) {
  return function(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;

    descriptor.value = function(...args: any[]) {
      const result = originalMethod.apply(this, args);
      afterFn.apply(this, [result, ...args]);
      return result;
    };

    return descriptor;
  };
}

// Around decorator
function around(wrapper: (original: AnyFunction) => AnyFunction) {
  return function(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    descriptor.value = wrapper(descriptor.value);
    return descriptor;
  };
}

// Throttle decorator
function throttle(limit: number) {
  return function(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;
    let timeout: NodeJS.Timeout | null = null;

    descriptor.value = function(...args: any[]) {
      if (!timeout) {
        originalMethod.apply(this, args);
        timeout = setTimeout(() => {
          timeout = null;
        }, limit);
      }
    };

    return descriptor;
  };
}

// Debounce decorator
function debounce(delay: number) {
  return function(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;
    let timeout: NodeJS.Timeout;

    descriptor.value = function(...args: any[]) {
      clearTimeout(timeout);
      timeout = setTimeout(() => {
        originalMethod.apply(this, args);
      }, delay);
    };

    return descriptor;
  };
}

// Usage examples
class SearchComponent {
  @before(function() {
    console.log('Validating search...');
  })
  @after(function(result: any) {
    console.log('Search completed with result:', result);
  })
  search(query: string): string[] {
    console.log('Searching for:', query);
    return ['result1', 'result2'];
  }

  @throttle(1000)
  onScroll(event: Event): void {
    console.log('Scroll event:', event);
  }

  @debounce(500)
  onSearchInput(value: string): void {
    console.log('Search input changed:', value);
  }

  @around((original) => {
    return function(...args: any[]) {
      console.log('Before execution');
      const result = original.apply(this, args);
      console.log('After execution');
      return result;
    };
  })
  complexOperation(data: any): any {
    console.log('Executing complex operation');
    return data;
  }
}
```

### API Request Decorator

```typescript
// HTTP decorators
interface RequestConfig {
  baseURL?: string;
  timeout?: number;
  headers?: Record<string, string>;
}

function Get(path: string, config?: RequestConfig) {
  return function(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    descriptor.value = async function(...args: any[]) {
      const url = `${config?.baseURL || ''}${path}`;
      const response = await fetch(url, {
        method: 'GET',
        headers: config?.headers,
      });
      
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      
      return await response.json();
    };

    return descriptor;
  };
}

function Post(path: string, config?: RequestConfig) {
  return function(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;

    descriptor.value = async function(...args: any[]) {
      const data = await originalMethod.apply(this, args);
      const url = `${config?.baseURL || ''}${path}`;
      
      const response = await fetch(url, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          ...config?.headers,
        },
        body: JSON.stringify(data),
      });
      
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      
      return await response.json();
    };

    return descriptor;
  };
}

function Cache(ttl: number = 60000) {
  const cache = new Map<string, { data: any; timestamp: number }>();

  return function(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;

    descriptor.value = async function(...args: any[]) {
      const key = `${propertyKey}:${JSON.stringify(args)}`;
      const cached = cache.get(key);
      
      if (cached && Date.now() - cached.timestamp < ttl) {
        console.log('Returning cached result');
        return cached.data;
      }
      
      const result = await originalMethod.apply(this, args);
      cache.set(key, { data: result, timestamp: Date.now() });
      
      return result;
    };

    return descriptor;
  };
}

// Usage
class UserApiService {
  @Get('/api/users', { baseURL: 'https://api.example.com' })
  @Cache(30000)
  async getUsers(): Promise<any[]> {
    // Implementation replaced by decorator
    return [];
  }

  @Post('/api/users', { baseURL: 'https://api.example.com' })
  async createUser(userData: any): Promise<any> {
    return userData; // Prepared by method, sent by decorator
  }

  @Get('/api/users/:id')
  @Cache(60000)
  async getUserById(id: string): Promise<any> {
    return {};
  }
}
```

### Validation Decorator

```typescript
// Validation decorators
function ValidateArgs(schema: any) {
  return function(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;

    descriptor.value = function(...args: any[]) {
      // Simple validation example
      const errors: string[] = [];

      schema.forEach((validator: any, index: number) => {
        const arg = args[index];
        
        if (validator.required && (arg === undefined || arg === null)) {
          errors.push(`Argument ${index} is required`);
        }
        
        if (validator.type && typeof arg !== validator.type) {
          errors.push(`Argument ${index} must be of type ${validator.type}`);
        }
        
        if (validator.min !== undefined && arg < validator.min) {
          errors.push(`Argument ${index} must be at least ${validator.min}`);
        }
        
        if (validator.max !== undefined && arg > validator.max) {
          errors.push(`Argument ${index} must be at most ${validator.max}`);
        }
      });

      if (errors.length > 0) {
        throw new Error(`Validation failed: ${errors.join(', ')}`);
      }

      return originalMethod.apply(this, args);
    };

    return descriptor;
  };
}

function ValidateReturn(validator: (value: any) => boolean, errorMessage: string) {
  return function(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;

    descriptor.value = function(...args: any[]) {
      const result = originalMethod.apply(this, args);
      
      if (!validator(result)) {
        throw new Error(errorMessage);
      }
      
      return result;
    };

    return descriptor;
  };
}

// Usage
class MathOperations {
  @ValidateArgs([
    { required: true, type: 'number', min: 0 },
    { required: true, type: 'number', min: 0 }
  ])
  @ValidateReturn(
    (result) => result >= 0,
    'Result must be non-negative'
  )
  add(a: number, b: number): number {
    return a + b;
  }

  @ValidateArgs([
    { required: true, type: 'number', min: 0 },
    { required: true, type: 'number', min: 1 }
  ])
  divide(a: number, b: number): number {
    return a / b;
  }
}

const math = new MathOperations();
console.log(math.add(5, 3)); // 8
// math.add(-5, 3); // Throws validation error
// math.divide(10, 0); // Throws validation error
```

## Common Mistakes

### 1. Too Many Decorator Layers

```typescript
// BAD - Over-decorated, hard to debug
const result = new DecoratorA(
  new DecoratorB(
    new DecoratorC(
      new DecoratorD(
        new DecoratorE(
          new BaseComponent()
        )
      )
    )
  )
);

// GOOD - Reasonable decorator chain
const result = new LoggingDecorator(
  new CachingDecorator(
    new BaseComponent()
  )
);
```

### 2. Breaking Interface Contract

```typescript
// BAD - Decorator changes interface
class BadDecorator extends BaseDecorator {
  // Adds new method not in interface
  newMethod() { }
}

// GOOD - Maintains interface
class GoodDecorator extends BaseDecorator {
  operation() {
    // Extends existing method
    super.operation();
    // Additional behavior
  }
}
```

### 3. Order Dependencies

```typescript
// BAD - Order matters but not documented
@CacheDecorator()
@ValidationDecorator()
@LoggingDecorator()
method() { } // Logging happens before validation!

// GOOD - Clear ordering or order-independent
@LoggingDecorator()
@ValidationDecorator()
@CacheDecorator()
method() { } // Log, then validate, then cache
```

## Best Practices

1. **Maintain single responsibility** - each decorator should do one thing
2. **Preserve the component interface** - decorators should be transparent
3. **Make decorators composable** - should work together without conflicts
4. **Document decorator order dependencies** when they exist
5. **Use descriptive names** that clearly indicate purpose
6. **Consider performance impact** of decorator chains
7. **Test decorators independently** before composing
8. **Provide configuration options** for flexibility
9. **Handle errors gracefully** in decorators
10. **Use TypeScript generics** for type safety

## When to Use Decorator Pattern

### Good Use Cases

- Adding logging, caching, or monitoring to existing methods
- Authentication and authorization checks
- Validation and transformation of inputs/outputs
- Rate limiting and throttling
- Error handling and retry logic
- UI component enhancement (HOCs in React)
- Cross-cutting concerns (AOP)

### When to Avoid

- Core functionality (use inheritance or composition)
- Simple modifications (direct implementation may be clearer)
- Performance-critical code (decorator overhead)
- When subclassing is more appropriate

## Interview Questions

### Q1: What's the difference between Decorator pattern and inheritance?

**Answer:** Decorator adds behavior dynamically at runtime by wrapping objects, while inheritance adds behavior statically at compile time. Decorators are more flexible and allow combining behaviors.

### Q2: How do TypeScript decorators differ from the classic Decorator pattern?

**Answer:** TypeScript decorators are a language feature (syntactic sugar) that modifies classes/methods at declaration time. Classic Decorator pattern wraps objects at runtime. TypeScript decorators are applied during class definition, while pattern decorators wrap instances.

### Q3: What are the performance implications of using decorators?

**Answer:** Each decorator adds a layer of function calls, which has overhead. Deep decorator chains can impact performance. Mitigation: cache decorator results, use decorators judiciously, profile performance.

### Q4: How do you compose multiple decorators effectively?

**Answer:** Use a compose function or apply decorators in correct order. Consider order dependencies (e.g., validation before caching). Document order requirements.

### Q5: What's the difference between Decorator and Proxy patterns?

**Answer:**
- **Decorator**: Adds new functionality while preserving interface
- **Proxy**: Controls access to the object, may restrict or delay operations

### Q6: How would you implement a decorator for rate limiting?

**Answer:** (See throttle/debounce decorators above) Track call times, reject calls exceeding rate limit, optionally queue delayed calls.

### Q7: Can decorators modify the return type of a function?

**Answer:** No, TypeScript decorators cannot change method signatures. They can modify behavior but must preserve the type contract.

### Q8: How do React HOCs relate to the Decorator pattern?

**Answer:** HOCs are an implementation of the Decorator pattern - they wrap components to add behavior while preserving the component interface.

## Key Takeaways

1. Decorator pattern adds behavior dynamically without modifying original object
2. Provides flexible alternative to subclassing for extending functionality
3. Decorators can be stacked to combine multiple behaviors
4. TypeScript/JavaScript decorators are experimental language features
5. React HOCs implement decorator pattern for components
6. Maintains single responsibility principle through composition
7. Order of decorators matters when there are dependencies
8. Performance overhead increases with decorator depth
9. Excellent for cross-cutting concerns (logging, caching, auth)
10. Should preserve original component interface

## Resources

- [TypeScript Decorators Documentation](https://www.typescriptlang.org/docs/handbook/decorators.html)
- [TC39 Decorator Proposal](https://github.com/tc39/proposal-decorators)
- [Refactoring Guru: Decorator Pattern](https://refactoring.guru/design-patterns/decorator)
- [React Higher-Order Components](https://react.dev/reference/react/Component#static-getderivedstatefromprops)
- [Angular Decorators](https://angular.io/guide/glossary#decorator)
- [Gang of Four Design Patterns](https://en.wikipedia.org/wiki/Design_Patterns)
