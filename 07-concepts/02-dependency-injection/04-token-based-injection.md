# Token-Based Injection

## The Idea

**In plain English:** Token-based injection is a way to give your code named "claim tickets" that it can hand in to receive a specific value or service — like a coat check where the ticket (token) is what you use to claim your coat, not the coat itself. The token is just a unique label that the system uses to look up and deliver exactly the right thing, whether that thing is a URL string, a config object, or a service.

**Real-world analogy:** Imagine a hotel luggage storage desk. You drop off your bag, and the clerk hands you a numbered ticket. Later, any guest who holds ticket #42 can walk up and claim that exact bag — they don't need to describe the bag, they just present the ticket.

- The numbered ticket = the injection token (a unique identifier, not the value itself)
- The bag stored behind the desk = the value or service being injected (e.g., an API URL, a config object)
- The hotel guest presenting the ticket = the component or service asking for the dependency

---

## Overview

Token-based injection solves a fundamental problem with class-based DI: you can't inject multiple implementations of the same interface, non-class values (strings, numbers, configs), or third-party services that you don't own. An injection token is a unique identifier used as the key in the dependency container — decoupled from both the provider and the consumer. This guide covers Angular's `InjectionToken`, React Context as a primitive token system, and patterns for making token-based injection type-safe and testable.

## The Problem with Class Tokens

```
Class-based injection maps class → instance:

  provide: HttpClient  →  instance of HttpClient

Problems:
  1. What if you need TWO different HTTP clients (different base URLs)?
  2. What if you want to inject a plain object, string, or factory function?
  3. What if the class comes from a third-party library you can't extend?
  4. What if you want to inject by interface/contract, not concrete class?

Token-based injection maps token → anything:

  provide: API_BASE_URL_TOKEN  →  "https://api.example.com"
  provide: FEATURE_FLAGS_TOKEN →  { darkMode: true, betaFeatures: false }
  provide: HTTP_CLIENT_TOKEN   →  new HttpClient({ baseUrl: '...' })
```

## Angular: InjectionToken

### Creating and Providing Tokens

```typescript
import { InjectionToken, Provider } from '@angular/core';

// ✅ Type-safe token creation
// Generic parameter T declares the shape of what's injected
export const API_URL = new InjectionToken<string>('API_URL');

export const FEATURE_FLAGS = new InjectionToken<FeatureFlags>('FEATURE_FLAGS', {
  // Optional: factory for default value (makes token tree-shakable)
  providedIn: 'root',
  factory: () => ({ darkMode: false, betaFeatures: false }),
});

interface FeatureFlags {
  darkMode: boolean;
  betaFeatures: boolean;
  experimentalUI: boolean;
}

// Providing a token in a module or component
@NgModule({
  providers: [
    { provide: API_URL, useValue: 'https://api.example.com' },
    {
      provide: FEATURE_FLAGS,
      useValue: {
        darkMode: true,
        betaFeatures: false,
        experimentalUI: false,
      },
    },
  ],
})
export class AppModule {}
```

### Injecting Tokens

```typescript
import { Component, inject, Inject } from '@angular/core';

// Modern style: inject() function (Angular 14+)
@Component({
  selector: 'app-api-service',
  template: '',
})
export class ApiService {
  private readonly apiUrl = inject(API_URL);
  private readonly flags = inject(FEATURE_FLAGS);

  getData() {
    return fetch(`${this.apiUrl}/data`);
  }
}

// Legacy style: @Inject decorator
@Component({ selector: 'app-legacy', template: '' })
export class LegacyComponent {
  constructor(
    @Inject(API_URL) private apiUrl: string,
    @Inject(FEATURE_FLAGS) private flags: FeatureFlags
  ) {}
}
```

### useFactory — Computed Token Values

```typescript
// Token that depends on other injected values
export const ANALYTICS_CONFIG = new InjectionToken<AnalyticsConfig>(
  'ANALYTICS_CONFIG'
);

const analyticsProviders: Provider[] = [
  {
    provide: ANALYTICS_CONFIG,
    useFactory: (apiUrl: string, flags: FeatureFlags) => ({
      endpoint: `${apiUrl}/analytics`,
      enabled: flags.betaFeatures,
      sessionTimeout: 30 * 60 * 1000,
    }),
    deps: [API_URL, FEATURE_FLAGS], // declare dependencies
  },
];
```

### useExisting — Aliasing Tokens

```typescript
// Make multiple tokens resolve to the same instance
export const LEGACY_HTTP = new InjectionToken<HttpClient>('LEGACY_HTTP');
export const MODERN_HTTP  = new InjectionToken<HttpClient>('MODERN_HTTP');

providers: [
  { provide: MODERN_HTTP, useClass: HttpClient },
  { provide: LEGACY_HTTP, useExisting: MODERN_HTTP }, // alias — same instance
]
```

### Multi-Providers with the Same Token

```typescript
// Collect multiple implementations under one token
export const HTTP_INTERCEPTORS = new InjectionToken<HttpInterceptor[]>(
  'HTTP_INTERCEPTORS'
);

providers: [
  { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor,   multi: true },
  { provide: HTTP_INTERCEPTORS, useClass: LoggingInterceptor, multi: true },
  { provide: HTTP_INTERCEPTORS, useClass: RetryInterceptor,   multi: true },
]

// Injection: receives an array
@Injectable()
export class HttpService {
  private interceptors = inject(HTTP_INTERCEPTORS); // HttpInterceptor[]
}
```

### Scoped Tokens — Feature Module Override

```typescript
// Override a root-level token at component tree level
@Component({
  selector: 'app-admin-panel',
  template: '<app-api-service></app-api-service>',
  providers: [
    // Provide a different API URL for all descendants
    { provide: API_URL, useValue: 'https://admin-api.example.com' },
  ],
})
export class AdminPanelComponent {}

// ApiService inside AdminPanelComponent's subtree gets the admin URL
// ApiService elsewhere gets the root URL
```

## React Context as Token-Based Injection

React's Context API is conceptually equivalent to token-based injection: the Context object is the token, and `Provider` + `value` is the provider registration.

```typescript
import { createContext, useContext, ReactNode } from 'react';

// The context object IS the token
interface ApiConfig {
  baseUrl: string;
  timeout: number;
  headers: Record<string, string>;
}

// createContext creates the token — never pass null, use a proper default or sentinel
const ApiConfigContext = createContext<ApiConfig | null>(null);

// Custom hook that reads the token — fails loudly if not provided
function useApiConfig(): ApiConfig {
  const config = useContext(ApiConfigContext);
  if (config === null) {
    throw new Error('useApiConfig must be used inside <ApiConfigProvider>');
  }
  return config;
}

// Provider (analogous to Angular's providers array)
function ApiConfigProvider({ children, config }: { children: ReactNode; config: ApiConfig }) {
  return (
    <ApiConfigContext.Provider value={config}>
      {children}
    </ApiConfigContext.Provider>
  );
}

// Consumer
function ApiService() {
  const { baseUrl, headers } = useApiConfig();

  async function fetchUsers() {
    return fetch(`${baseUrl}/users`, { headers });
  }

  return null; // service component or just a hook pattern
}
```

### Multiple Implementations via Tokens

```typescript
// Use different tokens for different implementations of the same interface
interface Logger {
  log(message: string, data?: unknown): void;
  error(message: string, error?: Error): void;
}

const DevLoggerContext  = createContext<Logger | null>(null);
const ProdLoggerContext = createContext<Logger | null>(null);

const consoleLogger: Logger = {
  log: (msg, data) => console.log(msg, data),
  error: (msg, err) => console.error(msg, err),
};

const sentryLogger: Logger = {
  log: (msg, data) => Sentry.addBreadcrumb({ message: msg, data }),
  error: (msg, err) => Sentry.captureException(err, { extra: { message: msg } }),
};

// Or: one token, different values at different tree levels
const LoggerContext = createContext<Logger>(consoleLogger); // default

function ProductionApp({ children }: { children: ReactNode }) {
  return (
    <LoggerContext.Provider value={sentryLogger}>
      {children}
    </LoggerContext.Provider>
  );
}

function DevelopmentApp({ children }: { children: ReactNode }) {
  // Uses default (consoleLogger) — no Provider needed
  return <>{children}</>;
}
```

### Token-Based Config Injection Pattern

```typescript
// Comprehensive example: environment config injection
interface AppConfig {
  api: { baseUrl: string; version: string };
  auth: { domain: string; clientId: string };
  features: { darkMode: boolean; analytics: boolean };
}

const AppConfigContext = createContext<AppConfig | null>(null);

// Strict hook — throws if token not provided
export function useAppConfig(): AppConfig {
  const config = useContext(AppConfigContext);
  if (!config) throw new Error('AppConfig not provided');
  return config;
}

// Convenience derived hooks
export function useApiBaseUrl(): string {
  return useAppConfig().api.baseUrl;
}

export function useFeatureFlag(flag: keyof AppConfig['features']): boolean {
  return useAppConfig().features[flag];
}

// Root injection point
function App() {
  const config: AppConfig = {
    api: { baseUrl: process.env.NEXT_PUBLIC_API_URL!, version: 'v2' },
    auth: { domain: process.env.NEXT_PUBLIC_AUTH_DOMAIN!, clientId: '...' },
    features: { darkMode: true, analytics: process.env.NODE_ENV === 'production' },
  };

  return (
    <AppConfigContext.Provider value={config}>
      <Router />
    </AppConfigContext.Provider>
  );
}
```

## When to Use Token-Based Injection Over Class Tokens

```
Use token-based injection when:

┌────────────────────────────────────────────────────────────────┐
│ Scenario                          │ Token or Class?            │
│───────────────────────────────────│────────────────────────────│
│ Injecting a string/number/boolean │ Token                      │
│ Injecting a config object         │ Token                      │
│ Multiple impls of same interface  │ Token (+ multi: true)      │
│ Third-party class (no constructor)│ Token (useFactory/useValue)│
│ Feature flags / env variables     │ Token                      │
│ Your own service with no deps     │ Class token (simpler)      │
│ Service that needs injected deps  │ Class token (auto-DI)      │
│ Abstract base class pattern       │ Token (prevents abstract   │
│                                   │ class instantiation)       │
└────────────────────────────────────────────────────────────────┘
```

## Testing with Token-Based Injection

### Angular Testing

```typescript
import { TestBed } from '@angular/core/testing';

describe('ApiService', () => {
  const mockApiUrl = 'http://localhost:3000';
  const mockFlags: FeatureFlags = {
    darkMode: false,
    betaFeatures: true,
    experimentalUI: false,
  };

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        ApiService,
        // Inject test values via tokens — no need to mock entire classes
        { provide: API_URL, useValue: mockApiUrl },
        { provide: FEATURE_FLAGS, useValue: mockFlags },
      ],
    });
  });

  it('uses the injected API URL', () => {
    const service = TestBed.inject(ApiService);
    // service.apiUrl === 'http://localhost:3000'
  });
});
```

### React Testing

```typescript
import { render, screen } from '@testing-library/react';

function renderWithConfig(ui: React.ReactElement, config: Partial<AppConfig> = {}) {
  const testConfig: AppConfig = {
    api: { baseUrl: 'http://localhost:3000', version: 'v1' },
    auth: { domain: 'test.auth0.com', clientId: 'test-client' },
    features: { darkMode: false, analytics: false },
    ...config,
  };

  return render(
    <AppConfigContext.Provider value={testConfig}>
      {ui}
    </AppConfigContext.Provider>
  );
}

test('displays feature when flag enabled', () => {
  renderWithConfig(<FeatureComponent />, {
    features: { darkMode: false, analytics: true },
  });
  expect(screen.getByTestId('analytics-panel')).toBeInTheDocument();
});
```

## Common Mistakes

### 1. Using Strings as Tokens (Type Unsafe)

```typescript
// ❌ Bad: string key is not type-safe
providers: [{ provide: 'API_URL', useValue: 'https://example.com' }]

@Inject('API_URL') private url: string // no compile-time check

// ✅ Good: InjectionToken carries type information
export const API_URL = new InjectionToken<string>('API_URL');
```

### 2. Mutable Default Context Values

```typescript
// ❌ Bad: mutable default — consumers can accidentally modify shared config
const ConfigContext = createContext({ theme: 'dark', language: 'en' });

// ✅ Good: either use null default (forces explicit provision) or freeze
const ConfigContext = createContext<AppConfig | null>(null);
// or
const ConfigContext = createContext(Object.freeze({ theme: 'dark', language: 'en' }));
```

### 3. Token Collision in Large Apps

```typescript
// ❌ Bad: generic description string — collision risk in large teams
const TOKEN = new InjectionToken('http');       // which http? conflict possible
const TOKEN2 = new InjectionToken('http');      // same description, different token
// They are different tokens (object identity), but confusing

// ✅ Good: descriptive, namespaced description
const HTTP_CLIENT_TOKEN = new InjectionToken<HttpClient>('app/core:HttpClient');
const ADMIN_HTTP_CLIENT  = new InjectionToken<HttpClient>('app/admin:HttpClient');
```

### 4. Forgetting Optional vs Required Injection

```typescript
// ❌ Bad: will throw at runtime if token not provided
const value = inject(OPTIONAL_FEATURE_TOKEN); // throws NullInjectorError

// ✅ Good: mark optional injections
const value = inject(OPTIONAL_FEATURE_TOKEN, { optional: true }); // returns null
// or in React
const config = useContext(ConfigContext); // can be null — check before use
```

## Interview Questions

### 1. What is an injection token and why is it preferred over using a class as a DI key?

**Answer**: An injection token is an opaque identifier used as the key in a dependency injection container. Unlike class tokens, it doesn't require a class to exist — you can inject primitive values (strings, numbers), interfaces, config objects, or functions. Class tokens are limited because: (a) you can't inject an interface (TypeScript interfaces don't exist at runtime), (b) you can't have two different injections of the same class type, (c) third-party classes can't always be used directly as tokens. `InjectionToken<T>` is type-safe (the generic T tells the type system what will be injected) and uniquely identified by object identity, so two tokens with the same description string are still distinct.

### 2. How does Angular's InjectionToken differ from React Context in terms of DI semantics?

**Answer**: Angular's `InjectionToken` is a first-class DI primitive: tokens are registered in a hierarchical injector (root, module, component), the framework resolves them through the injector tree, and `multi: true` supports collecting multiple registrations. React Context is flatter — there's no injector hierarchy, just a Provider in the component tree. Context is essentially a global mutable value scoped to a subtree. Angular DI handles lazy loading, singleton scoping (`providedIn: 'root'`), and interceptor chains natively; React requires manual patterns for these. For complex enterprise DI scenarios, Angular's model is more powerful; for React, libraries like InversifyJS or tsyringe can add similar capabilities.

### 3. What is the `multi: true` flag in Angular providers and what problem does it solve?

**Answer**: `multi: true` tells Angular that multiple providers can register under the same token, and the injected value will be an array of all registered values. Without it, each `provide` for the same token overwrites the previous one. This is how Angular's built-in `HTTP_INTERCEPTORS` token works — multiple interceptors register independently, and the `HttpClient` receives them all as an array. It enables plugin-style architectures: library code defines a token, consumers register their implementations, and the service collects them all without any of the providers knowing about each other.

### 4. How would you test a component that depends on an injected token in both Angular and React?

**Answer**: In Angular's `TestBed`, you provide the token with a test value: `{ provide: MY_TOKEN, useValue: testValue }`. This overrides the real provider for the test module — no mocking frameworks needed. In React, you wrap the component under test in a context Provider with the test value: `render(<MyCtx.Provider value={testValue}><ComponentUnderTest /></MyCtx.Provider>)`. Both patterns enable testing the component in isolation with controlled input values. The key benefit of token-based injection is that you never need to mock class constructors or patch module imports — you simply provide a different value through the same channel.
