# Angular DI vs React Context: A Comparative Analysis

## Overview

Angular ships a full-featured hierarchical dependency injection system as a first-class framework primitive. React has no built-in DI system — Context is a value-propagation mechanism that developers press into service as a lightweight DI substitute. Understanding both systems deeply clarifies when React Context is sufficient, when it breaks down, and what patterns bridge the gap. This guide compares the two side-by-side across architecture, scoping, testing, and real-world scenarios.

## Mental Model Comparison

```
Angular DI:
  ┌─────────────────────────────────────────────────────┐
  │  NullInjector (root)                                │
  │    └─ PlatformInjector                              │
  │         └─ AppInjector (providedIn:'root' singletons)
  │              └─ FeatureModuleInjector               │
  │                   └─ ComponentInjector              │
  │                        └─ ChildComponentInjector    │
  └─────────────────────────────────────────────────────┘

  - Injectors form a tree mirroring the module/component hierarchy
  - Child injector checks parent if token not found locally
  - One instance per injector level (true singletons at root)

React Context:
  ┌─────────────────────────────────────────────────────┐
  │  <AppContext.Provider value={appDeps}>               │
  │    <FeatureContext.Provider value={featureDeps}>     │
  │      <ComponentTree />                               │
  │    </FeatureContext.Provider>                        │
  │  </AppContext.Provider>                              │
  └─────────────────────────────────────────────────────┘

  - Context is a named channel through the component tree
  - Nearest Provider wins
  - No automatic instantiation — you manage object creation
```

## Provider Registration

### Angular

```typescript
// 1. Root singleton — available everywhere, tree-shakable
@Injectable({ providedIn: 'root' })
export class UserService {
  private http = inject(HttpClient);
  getUser(id: string) { return this.http.get<User>(`/users/${id}`); }
}

// 2. Module-scoped — new instance per module
@NgModule({
  providers: [{ provide: UserService, useClass: UserService }]
})
export class AdminModule {}

// 3. Component-scoped — new instance per component tree
@Component({
  selector: 'app-form',
  providers: [{ provide: FormService, useClass: FormService }],
  template: '...',
})
export class FormComponent {}

// 4. Factory provider — complex construction logic
@NgModule({
  providers: [
    {
      provide: AnalyticsService,
      useFactory: (config: AppConfig, logger: LoggerService) => {
        return config.isProd
          ? new ProductionAnalytics(config.analyticsKey, logger)
          : new NoopAnalytics();
      },
      deps: [APP_CONFIG, LoggerService],
    },
  ],
})
export class AppModule {}

// 5. Value provider — for configuration
@NgModule({
  providers: [
    { provide: API_URL, useValue: environment.apiUrl },
  ],
})
export class CoreModule {}
```

### React Context (Equivalent Patterns)

```typescript
// 1. Root singleton equivalent
const UserServiceContext = createContext<UserService | null>(null);

// You create and manage the instance:
function AppProviders({ children }: { children: ReactNode }) {
  // useMemo ensures one instance across re-renders (equivalent to root singleton)
  const userService = useMemo(() => new UserService(apiBaseUrl), [apiBaseUrl]);

  return (
    <UserServiceContext.Provider value={userService}>
      {children}
    </UserServiceContext.Provider>
  );
}

// 2. Scoped instance — component subtree
function AdminPanel({ children }: { children: ReactNode }) {
  // New instance scoped to AdminPanel's subtree
  const adminService = useMemo(() => new AdminService(), []);
  return (
    <AdminServiceContext.Provider value={adminService}>
      {children}
    </AdminServiceContext.Provider>
  );
}

// 3. Factory equivalent
function AnalyticsProvider({ children }: { children: ReactNode }) {
  const config = useAppConfig();
  const logger = useLogger();

  const analytics = useMemo(
    () => (config.isProd ? new ProductionAnalytics(config.key, logger) : new NoopAnalytics()),
    [config.isProd, config.key, logger]
  );

  return (
    <AnalyticsContext.Provider value={analytics}>
      {children}
    </AnalyticsContext.Provider>
  );
}
```

## Service-to-Service Injection

This is where Angular's system shines compared to React Context:

### Angular: Automatic Dependency Graph

```typescript
// Angular resolves the entire graph automatically
@Injectable({ providedIn: 'root' })
export class AuthService {
  private token: string | null = null;
  setToken(t: string) { this.token = t; }
  getToken() { return this.token; }
}

@Injectable({ providedIn: 'root' })
export class HttpService {
  private auth = inject(AuthService);  // automatically injected
  
  get<T>(url: string) {
    return fetch(url, {
      headers: { Authorization: `Bearer ${this.auth.getToken()}` }
    });
  }
}

@Injectable({ providedIn: 'root' })
export class UserService {
  private http = inject(HttpService);  // automatically injected
  
  getProfile() { return this.http.get<Profile>('/me'); }
}

// Component just declares what it needs:
@Component({ ... })
export class ProfileComponent {
  private userService = inject(UserService);
  // Angular resolves the entire AuthService → HttpService → UserService chain
}
```

### React: Manual Wiring

```typescript
// React requires you to wire everything explicitly
function buildDependencies(config: AppConfig) {
  const authService = new AuthService();
  const httpService = new HttpService(authService); // manual injection
  const userService = new UserService(httpService); // manual injection

  return { authService, httpService, userService };
}

function AppProviders({ children, config }: { children: ReactNode; config: AppConfig }) {
  const services = useMemo(() => buildDependencies(config), [config]);

  return (
    <AuthContext.Provider value={services.authService}>
      <HttpContext.Provider value={services.httpService}>
        <UserContext.Provider value={services.userService}>
          {children}
        </UserContext.Provider>
      </HttpContext.Provider>
    </AuthContext.Provider>
  );
}

// Context nesting becomes a "Provider Hell" with many services
// Usually solved by a single "ServicesContext" with all deps as an object
```

## Lifetime and Scoping

```
                Angular DI          │  React Context
─────────────────────────────────── │ ─────────────────────────────
Singleton (app lifetime)     ✓      │  ✓ (via useMemo at root)
Module-scoped singleton      ✓      │  ✓ (Provider in feature tree)
Component-scoped singleton   ✓      │  ✓ (Provider in component)
Request-scoped (SSR)         ✓      │  Manual wiring per request
Transient (new per inject)   ✓      │  Not supported natively
Destroy lifecycle hook       ✓      │  useEffect cleanup only
```

### Angular Component Lifecycle Integration

```typescript
// Angular services can implement OnDestroy
@Injectable()
export class WebSocketService implements OnDestroy {
  private socket: WebSocket;

  constructor() {
    this.socket = new WebSocket('wss://example.com');
  }

  ngOnDestroy() {
    // Automatically called when the component that provides this service is destroyed
    this.socket.close();
  }
}

@Component({
  selector: 'app-chat',
  providers: [WebSocketService], // component-scoped, destroyed with component
  template: '<div>Chat</div>',
})
export class ChatComponent {}
```

```typescript
// React equivalent — must wire cleanup manually
function useWebSocket(url: string) {
  const [socket, setSocket] = useState<WebSocket | null>(null);

  useEffect(() => {
    const ws = new WebSocket(url);
    setSocket(ws);
    return () => ws.close(); // cleanup on unmount
  }, [url]);

  return socket;
}

// Or via Context
function WebSocketProvider({ children, url }: { children: ReactNode; url: string }) {
  const socketRef = useRef<WebSocket | null>(null);

  useEffect(() => {
    socketRef.current = new WebSocket(url);
    return () => socketRef.current?.close();
  }, [url]);

  return (
    <WebSocketContext.Provider value={socketRef}>
      {children}
    </WebSocketContext.Provider>
  );
}
```

## Interceptors and Middleware

### Angular: First-Class Interceptor Pattern

```typescript
// HttpInterceptors modify every HTTP request/response in the chain
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  private auth = inject(AuthService);

  intercept(req: HttpRequest<unknown>, next: HttpHandler) {
    const token = this.auth.getToken();
    if (!token) return next.handle(req);

    const authReq = req.clone({
      headers: req.headers.set('Authorization', `Bearer ${token}`)
    });
    return next.handle(authReq);
  }
}

// Register once, applies to all HttpClient calls
providers: [
  { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true },
  { provide: HTTP_INTERCEPTORS, useClass: LoggingInterceptor, multi: true },
]
```

### React: Manual Middleware

```typescript
// Must build interceptor chain manually
type Interceptor = (req: RequestInit, next: typeof fetch) => Promise<Response>;

function createHttpClient(interceptors: Interceptor[]) {
  return async function(url: string, options: RequestInit = {}) {
    let index = 0;

    async function next(req: RequestInit): Promise<Response> {
      if (index < interceptors.length) {
        return interceptors[index++](req, next);
      }
      return fetch(url, req);
    }

    return next(options);
  };
}

const authInterceptor: Interceptor = async (req, next) => {
  const token = getToken(); // from some auth context
  return next({ ...req, headers: { ...req.headers, Authorization: `Bearer ${token}` } });
};

const http = createHttpClient([authInterceptor, loggingInterceptor]);

// Provide through context
const HttpContext = createContext(http);
```

## Where React Context Falls Short

```
Limitation                       │ Angular DI Alternative
─────────────────────────────────│─────────────────────────────────
Automatic dependency graph       │ @Injectable deps are auto-resolved
Circular dependency detection    │ Framework detects at startup
Lazy module loading + DI         │ @Injectable({ providedIn: LazyModule })
Transient / per-request scope    │ useClass + non-root providers
Service destroy lifecycle        │ ngOnDestroy on services
Compile-time DI graph validation │ Ivy compiler checks at build time
Interceptor chains               │ HTTP_INTERCEPTORS multi-token
Tree-shaking unused providers    │ providedIn: 'root' with no consumers
```

## Using InversifyJS in React

For React apps requiring true DI, InversifyJS provides an Angular-like container:

```typescript
import 'reflect-metadata';
import { injectable, inject, Container } from 'inversify';

// Symbols as tokens (equivalent to InjectionToken)
const TYPES = {
  Logger: Symbol.for('Logger'),
  HttpService: Symbol.for('HttpService'),
  UserService: Symbol.for('UserService'),
};

@injectable()
class HttpService {
  constructor(@inject(TYPES.Logger) private logger: Logger) {}

  async get<T>(url: string): Promise<T> {
    this.logger.log(`GET ${url}`);
    const res = await fetch(url);
    return res.json();
  }
}

@injectable()
class UserService {
  constructor(@inject(TYPES.HttpService) private http: HttpService) {}
  getProfile() { return this.http.get<Profile>('/me'); }
}

// Build container
const container = new Container();
container.bind<Logger>(TYPES.Logger).toConstantValue(consoleLogger);
container.bind<HttpService>(TYPES.HttpService).to(HttpService).inSingletonScope();
container.bind<UserService>(TYPES.UserService).to(UserService).inSingletonScope();

// React integration via Context
const DIContext = createContext<Container | null>(null);

function DIProvider({ children }: { children: ReactNode }) {
  return <DIContext.Provider value={container}>{children}</DIContext.Provider>;
}

function useDependency<T>(token: symbol): T {
  const container = useContext(DIContext);
  if (!container) throw new Error('DIProvider required');
  return container.get<T>(token);
}

// Usage
function ProfilePage() {
  const userService = useDependency<UserService>(TYPES.UserService);
  // ...
}
```

## Testing Comparison

```typescript
// Angular: TestBed gives full DI control
describe('UserService', () => {
  let service: UserService;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        UserService,
        { provide: HttpService, useClass: MockHttpService },
        { provide: API_URL, useValue: 'http://localhost' },
      ],
    });
    service = TestBed.inject(UserService);
  });
});

// React: Wrap in provider with test values
function renderWithDeps(ui: ReactElement, overrides: Partial<Services> = {}) {
  const testServices: Services = {
    userService: { getProfile: jest.fn() },
    httpService: { get: jest.fn() },
    ...overrides,
  };
  return render(
    <ServicesContext.Provider value={testServices}>
      {ui}
    </ServicesContext.Provider>
  );
}
```

## Decision Guide

```
Question: Which DI mechanism should I use?

Is the project Angular?
  → Use Angular DI (InjectionToken, providers, @Injectable)

Is the project React with simple needs?
  → Use Context (1-3 services, no complex dependency graph)

Does your React app have:
  - 5+ interdependent services?
  - Need for transient/request scoping?
  - Interceptor pipelines?
  - Circular dep detection requirements?
  → Consider InversifyJS or tsyringe

Does the "service" just hold shared state?
  → Use Zustand / Redux / Jotai instead — they're the right tool
```

## Common Mistakes

### 1. Using Context for High-Frequency Updates

```typescript
// ❌ Bad: putting frequently-changing values in Context
// Every useContext consumer re-renders on ANY context value change
const AppContext = createContext({
  user: null,
  theme: 'dark',
  notifications: [],  // updated every 5 seconds!
  currentRoute: '/',  // changes on every navigation!
});

// ✅ Good: split contexts by update frequency
const ThemeContext     = createContext('dark');        // rarely changes
const UserContext      = createContext<User | null>(null); // changes on login/logout
const NotifyContext    = createContext<Notification[]>([]); // frequent — separate context
```

### 2. Missing Memoization on Context Values

```typescript
// ❌ Bad: new object reference on every Parent render
function Parent() {
  const value = { user: currentUser, updateUser: setUser }; // new ref every render
  return <UserContext.Provider value={value}>{children}</UserContext.Provider>;
}

// ✅ Good: memoize the context value
function Parent() {
  const value = useMemo(
    () => ({ user: currentUser, updateUser: setUser }),
    [currentUser, setUser]
  );
  return <UserContext.Provider value={value}>{children}</UserContext.Provider>;
}
```

### 3. Treating Context as a React-Only Concept

Angular developers moving to React sometimes underuse Context (missing the DI parallel) and React developers moving to Angular sometimes overuse `providedIn: 'root'` (creating unnecessary singletons).

## Interview Questions

### 1. What is the fundamental architectural difference between Angular DI and React Context?

**Answer**: Angular DI is a hierarchical inversion-of-control container: it manages object creation, lifetime, and dependency resolution automatically. You declare what a class needs (`inject(UserService)`) and the framework finds, creates, or retrieves the instance from the injector tree. React Context is a value propagation mechanism: it passes a value from a Provider down to any consuming component without prop drilling. React has no automatic instantiation — you create the service objects yourself and place them in a Provider. Angular DI is pull-based (framework controls creation); React Context is push-based (you push values down the tree).

### 2. When does React Context break down as a DI mechanism?

**Answer**: Context breaks down in several scenarios: (1) **Complex dependency graphs** — when ServiceA depends on ServiceB which depends on ServiceC, you must manually wire the chain; Angular does this automatically. (2) **Performance with high-frequency updates** — every context consumer re-renders when context value changes; splitting contexts mitigates but doesn't eliminate this. (3) **Transient/request scoping** — React has no equivalent to creating a new service instance per request (important for SSR). (4) **Interceptor pipelines** — HTTP interceptors require manual middleware chain construction. (5) **Circular dependency detection** — Angular detects at compile time; React will just throw a runtime error or infinite loop.

### 3. How would you implement a singleton service pattern in React that's equivalent to Angular's `providedIn: 'root'`?

**Answer**: Wrap the service creation in `useMemo` at the root Provider, or (for true module-level singletons) create the instance outside React's render cycle as a module-level constant. The `useMemo` approach scopes the instance to the Provider's lifetime — if the Provider unmounts and remounts, you get a new instance (equivalent to component-scoped in Angular). For a true app-lifetime singleton, create the instance at module evaluation time: `const userService = new UserService(httpService);` and provide it via context. This is a "poor man's `providedIn: 'root'`" — it works but loses Angular's tree-shaking benefit (Angular can detect unused `providedIn: 'root'` services at build time).

### 4. What would make you choose InversifyJS over raw React Context for dependency injection?

**Answer**: Choose InversifyJS when: (a) you have 5+ services with complex interdependencies that would create deeply nested Providers or a God-object services bag; (b) you need transient scope (new instance every injection) — Context always gives you the same instance; (c) you want compile-time DI graph validation via TypeScript decorators; (d) you need circular dependency detection; (e) the team is familiar with Angular-style DI and moving to React. The cost is `reflect-metadata` polyfill, decorator-based syntax (requires experimentalDecorators), and framework coupling. For simpler React apps, the cognitive overhead of InversifyJS outweighs the benefits — plain Context + custom hooks is usually sufficient.
