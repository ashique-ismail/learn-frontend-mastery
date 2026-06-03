# 📚 Comprehensive React & Angular Learning Path — Detailed Curriculum

> **Format:** Hierarchical folder structure. Each leaf is a discrete learning item an agent can generate content for.
> **Audience:** Experienced web developer targeting senior-level interviews.
> **Convention:** `📁` = folder, `📄` = topic to generate content for.

---

## 📁 `01-html/`

### 📁 `01-html/01-fundamentals/`
- 📄 Document structure, DOCTYPE, rendering modes (quirks vs standards)
- 📄 Semantic elements & document outline (`header`, `main`, `article`, `section`, `nav`, `aside`, `footer`)
- 📄 Headings hierarchy and accessibility tree
- 📄 Block vs inline vs inline-block vs replaced elements
- 📄 Global attributes (`id`, `class`, `data-*`, `tabindex`, `hidden`, `contenteditable`)

### 📁 `01-html/02-forms/`
- 📄 Input types (text, number, email, date, file, range, color)
- 📄 Native validation (`required`, `pattern`, `min`, `max`, `step`, `:valid`, `:invalid`)
- 📄 `FormData` API and form submission lifecycle
- 📄 `<label>` association, fieldset/legend, accessibility
- 📄 Autocomplete, autofill, autofocus semantics
- 📄 File uploads (single, multiple, drag-drop)

### 📁 `01-html/03-media-and-graphics/`
- 📄 `<img>` with `srcset`, `sizes`, `loading="lazy"`, `decoding`
- 📄 `<picture>` element and art direction
- 📄 `<video>`, `<audio>`, tracks, captions
- 📄 `<canvas>` vs `<svg>` — when to use which
- 📄 SVG basics (viewBox, paths, symbols, sprites)

### 📁 `01-html/04-metadata-and-performance/`
- 📄 `<head>` essentials: charset, viewport, title, description
- 📄 Open Graph & Twitter Card meta tags
- 📄 `<link rel>` variants: preload, prefetch, preconnect, dns-prefetch, modulepreload
- 📄 Resource hints and critical rendering path
- 📄 `<script>` attributes: `async`, `defer`, `type="module"`, `nomodule`

### 📁 `01-html/05-accessibility/`
- 📄 ARIA roles, states, properties (when to use, when NOT to use)
- 📄 Landmark roles and screen reader navigation
- 📄 Focus management and `tabindex`
- 📄 Live regions (`aria-live`, `role="status"`, `role="alert"`)
- 📄 WCAG 2.2 conformance levels
- 📄 Keyboard navigation patterns

### 📁 `01-html/06-advanced/`
- 📄 Web Components: Custom Elements, Shadow DOM, slots, templates
- 📄 `<dialog>` element and the top layer
- 📄 Popover API
- 📄 IntersectionObserver, ResizeObserver, MutationObserver
- 📄 Drag-and-drop API

---

## 📁 `02-css/`

### 📁 `02-css/01-fundamentals/`
- 📄 Selectors, specificity, cascade, inheritance
- 📄 Box model (content-box vs border-box), margin collapsing
- 📄 Stacking contexts and `z-index` rules
- 📄 Containing blocks and positioning schemes
- 📄 Units: px, em, rem, %, vw/vh, dvh/svh/lvh, ch, ex, fr

### 📁 `02-css/02-layout/`
- 📄 Flexbox: main/cross axis, flex-grow/shrink/basis, alignment
- 📄 CSS Grid: explicit vs implicit, `grid-template`, named lines/areas
- 📄 Subgrid
- 📄 Multi-column layout
- 📄 Position: static, relative, absolute, fixed, sticky

### 📁 `02-css/03-modern-features/`
- 📄 Custom properties (CSS variables) and scoping
- 📄 `calc()`, `min()`, `max()`, `clamp()`
- 📄 `:has()` parent selector
- 📄 `:is()` and `:where()` (specificity differences)
- 📄 Container queries (`@container`)
- 📄 Logical properties (inline/block start/end)
- 📄 Cascade layers (`@layer`)
- 📄 Nesting (native CSS nesting)
- 📄 `color-mix()`, `oklch`, modern color spaces

### 📁 `02-css/04-styling/`
- 📄 Typography (font stacks, web fonts, `font-display`, variable fonts)
- 📄 Transitions and timing functions
- 📄 Keyframe animations and `@keyframes`
- 📄 Transforms (2D and 3D)
- 📄 Filters, backdrop-filter, mix-blend-mode
- 📄 Gradients (linear, radial, conic)
- 📄 Pseudo-elements and pseudo-classes

### 📁 `02-css/05-architecture/`
- 📄 BEM methodology
- 📄 ITCSS / SMACSS / OOCSS
- 📄 Utility-first (Tailwind philosophy)
- 📄 CSS Modules
- 📄 CSS-in-JS (Emotion, styled-components) — tradeoffs
- 📄 Atomic CSS engines (UnoCSS, Tailwind JIT)

### 📁 `02-css/06-responsive-and-a11y/`
- 📄 Mobile-first vs desktop-first
- 📄 Media queries (range syntax, feature queries `@supports`)
- 📄 `prefers-color-scheme`, `prefers-reduced-motion`, `prefers-contrast`
- 📄 Fluid typography (`clamp`)
- 📄 Focus-visible and accessible focus indicators

---

## 📁 `03-javascript/`

### 📁 `03-javascript/01-language-basics/`
- 📄 `var` vs `let` vs `const` (hoisting, TDZ)
- 📄 Primitives vs objects, value vs reference
- 📄 Type coercion, `==` vs `===`, truthy/falsy
- 📄 Operators (logical, nullish, optional chaining, spread/rest)
- 📄 Control flow and labeled statements
- 📄 Strict mode

### 📁 `03-javascript/02-functions/`
- 📄 Function declarations vs expressions vs arrow functions
- 📄 Parameters: default, rest, destructured
- 📄 `this` binding rules (4 rules) and arrow functions
- 📄 `call`, `apply`, `bind`
- 📄 IIFEs, higher-order functions, currying, partial application
- 📄 Pure functions and side effects

### 📁 `03-javascript/03-objects-and-types/`
- 📄 Object creation patterns (literal, `Object.create`, classes, factories)
- 📄 Property descriptors (`Object.defineProperty`, getters/setters)
- 📄 `Object.freeze`, `seal`, `preventExtensions`
- 📄 Classes, inheritance, `super`, private fields (`#`)
- 📄 Symbols and well-known symbols
- 📄 `Map`, `Set`, `WeakMap`, `WeakSet`, `WeakRef`

### 📁 `03-javascript/04-async/`
- 📄 Callbacks and callback hell
- 📄 Promises (states, chaining, error handling)
- 📄 `Promise.all`, `allSettled`, `race`, `any`
- 📄 async/await (error handling, parallel vs sequential)
- 📄 AbortController and cancellation
- 📄 Generators and async iterators
- 📄 Top-level await

### 📁 `03-javascript/05-modules-and-tooling/`
- 📄 ESM vs CommonJS (semantics, live bindings)
- 📄 Dynamic `import()` and code splitting
- 📄 Tree shaking prerequisites
- 📄 Package managers: npm, pnpm, yarn (lockfiles, workspaces)
- 📄 Bundlers: Vite, Webpack, esbuild, Rollup, Turbopack
- 📄 Transpilers: Babel, SWC, TSC

### 📁 `03-javascript/06-typescript/`
- 📄 Basic types, unions, intersections, literals
- 📄 Generics
- 📄 Utility types (`Partial`, `Pick`, `Omit`, `Record`, `ReturnType`, etc.)
- 📄 Conditional types, mapped types, template literal types
- 📄 Type narrowing, type guards, discriminated unions
- 📄 `unknown` vs `any` vs `never`
- 📄 Declaration files and module augmentation
- 📄 `tsconfig.json` essentials

### 📁 `03-javascript/07-browser-apis/`
- 📄 DOM traversal and manipulation
- 📄 Event model (bubbling, capturing, delegation, passive listeners)
- 📄 Custom events
- 📄 Fetch API, Headers, Request, Response
- 📄 Storage: localStorage, sessionStorage, IndexedDB, Cookies
- 📄 Web Workers, Service Workers, SharedArrayBuffer
- 📄 WebSockets, Server-Sent Events
- 📄 History API and navigation
- 📄 Clipboard, Notifications, Geolocation, File System Access

---

## 📁 `04-react/`

### 📁 `04-react/01-core/`
- 📄 JSX → `createElement` → ReactElement
- 📄 Function components and props
- 📄 Composition: `children`, render props, component slots
- 📄 Conditional rendering patterns
- 📄 Lists and keys (stable identity rules)
- 📄 Controlled vs uncontrolled components
- 📄 Fragments, Portals, StrictMode

### 📁 `04-react/02-hooks/`
- 📄 `useState` (functional updates, lazy init)
- 📄 `useReducer` (when to prefer over useState)
- 📄 `useEffect` (lifecycle mapping, cleanup, dependency rules)
- 📄 `useLayoutEffect` vs `useEffect`
- 📄 `useMemo` (and when it's an anti-pattern)
- 📄 `useCallback` (referential stability)
- 📄 `useRef` (DOM, mutable values, instance variables)
- 📄 `useContext` (and re-render pitfalls)
- 📄 `useImperativeHandle` + `forwardRef`
- 📄 `useTransition` and `useDeferredValue`
- 📄 `useId`
- 📄 `useSyncExternalStore`
- 📄 `use` hook (React 19)
- 📄 Custom hooks — composition patterns and naming conventions
- 📄 Rules of hooks (and why they exist)

### 📁 `04-react/03-patterns/`
- 📄 Container/Presentational components
- 📄 Compound components
- 📄 Render props
- 📄 Higher-Order Components (HOCs)
- 📄 Provider pattern
- 📄 State reducer pattern
- 📄 Controlled props pattern
- 📄 Component composition over prop drilling

### 📁 `04-react/04-data-and-effects/`
- 📄 Data fetching with `useEffect` (and its problems)
- 📄 Race conditions and cancellation
- 📄 Suspense for data fetching
- 📄 Error boundaries
- 📄 Optimistic updates

### 📁 `04-react/05-routing-and-forms/`
- 📄 React Router v6+ (data routers, loaders, actions)
- 📄 Nested routes, layout routes, index routes
- 📄 Route protection / guards
- 📄 React Hook Form architecture
- 📄 Form validation strategies (Zod + RHF)

### 📁 `04-react/06-performance/`
- 📄 Reconciliation and keys
- 📄 `React.memo` and shallow comparison
- 📄 Code splitting with `React.lazy` and Suspense
- 📄 Virtualization (TanStack Virtual, react-window)
- 📄 React DevTools Profiler
- 📄 Avoiding re-renders (context splitting, selectors)

### 📁 `04-react/07-react-18-19/`
- 📄 Concurrent rendering model
- 📄 Automatic batching
- 📄 Server Components (RSC)
- 📄 Server Actions
- 📄 Streaming SSR with Suspense
- 📄 `useOptimistic`, `useFormStatus`, `useActionState`

### 📁 `04-react/08-meta-frameworks/`
- 📄 Next.js App Router (layouts, loading, error, parallel routes, intercepting routes)
- 📄 Next.js rendering modes (SSG, SSR, ISR, PPR)
- 📄 Next.js route handlers and middleware
- 📄 Remix / React Router 7 (loaders, actions, nested data)
- 📄 Astro with React islands

### 📁 `04-react/09-testing/`
- 📄 React Testing Library philosophy
- 📄 User-event vs fire-event
- 📄 Mocking modules and network (MSW)
- 📄 Testing hooks
- 📄 Snapshot testing (and its pitfalls)
- 📄 E2E with Playwright/Cypress

---

## 📁 `05-angular/`

### 📁 `05-angular/01-core/`
- 📄 Angular architecture overview (modules vs standalone)
- 📄 Components, templates, decorators
- 📄 `@Input`, `@Output`, `EventEmitter`
- 📄 Required inputs, input transforms, signal inputs
- 📄 Template syntax (interpolation, property/event/two-way binding)
- 📄 New control flow: `@if`, `@for`, `@switch`, `@defer`, `@let`
- 📄 Lifecycle hooks (`ngOnInit`, `ngOnChanges`, `ngOnDestroy`, etc.)
- 📄 Standalone components and `bootstrapApplication`

### 📁 `05-angular/02-dependency-injection/`
- 📄 Injectors hierarchy (root, platform, element)
- 📄 `providedIn: 'root' | 'platform' | 'any'`
- 📄 `inject()` function vs constructor injection
- 📄 Injection tokens (`InjectionToken`)
- 📄 Provider types: useClass, useValue, useFactory, useExisting
- 📄 Multi-providers
- 📄 Optional, Self, SkipSelf, Host decorators
- 📄 Tree-shakable providers

### 📁 `05-angular/03-reactivity-signals/`
- 📄 `signal()`, `computed()`, `effect()`
- 📄 `linkedSignal()` and `resource()`
- 📄 Signal inputs/outputs/queries
- 📄 `toSignal` and `toObservable` (RxJS interop)
- 📄 Zone.js vs Zoneless change detection
- 📄 `OnPush` change detection strategy
- 📄 `ChangeDetectorRef` (markForCheck, detectChanges)

### 📁 `05-angular/04-rxjs/`
- 📄 Observable, Observer, Subscription, Subject
- 📄 BehaviorSubject, ReplaySubject, AsyncSubject
- 📄 Creation operators (`of`, `from`, `fromEvent`, `interval`, `timer`)
- 📄 Pipeable operators overview
- 📄 Flattening operators: `switchMap`, `mergeMap`, `concatMap`, `exhaustMap`
- 📄 Combination: `combineLatest`, `forkJoin`, `withLatestFrom`, `zip`
- 📄 Filtering: `filter`, `debounceTime`, `distinctUntilChanged`, `take`, `takeUntil`
- 📄 Error handling: `catchError`, `retry`, `retryWhen`
- 📄 Multicasting: `share`, `shareReplay`
- 📄 Hot vs cold observables
- 📄 Unsubscription patterns (`takeUntilDestroyed`)
- 📄 Schedulers

### 📁 `05-angular/05-forms/`
- 📄 Template-driven forms
- 📄 Reactive forms: `FormGroup`, `FormControl`, `FormArray`
- 📄 Typed forms
- 📄 Built-in validators
- 📄 Custom synchronous validators
- 📄 Custom async validators
- 📄 Cross-field validation
- 📄 ControlValueAccessor (custom form controls)

### 📁 `05-angular/06-routing/`
- 📄 Route configuration and lazy loading
- 📄 Router outlet and navigation
- 📄 Route parameters, query params, fragments
- 📄 Guards: `canActivate`, `canMatch`, `canDeactivate`, `canActivateChild`
- 📄 Functional guards (modern approach)
- 📄 Resolvers
- 📄 Router events
- 📄 Preloading strategies

### 📁 `05-angular/07-http/`
- 📄 `HttpClient` basics
- 📄 Request/response typing
- 📄 Functional HTTP interceptors
- 📄 Error handling
- 📄 `HttpContext` and metadata
- 📄 File upload/download with progress
- 📄 Caching strategies

### 📁 `05-angular/08-advanced-templates/`
- 📄 `ng-template`, `ng-container`, `TemplateRef`
- 📄 Content projection: single-slot, multi-slot, conditional
- 📄 Structural directives (custom)
- 📄 Attribute directives (custom)
- 📄 Host bindings and host listeners
- 📄 Pipes (pure vs impure), custom pipes
- 📄 Dynamic component loading (`ViewContainerRef`)

### 📁 `05-angular/09-ssr-and-performance/`
- 📄 Angular Universal (SSR)
- 📄 Non-destructive hydration
- 📄 `@defer` blocks and deferred loading
- 📄 OnPush + Signals optimization
- 📄 Bundle optimization, lazy modules → lazy routes
- 📄 `trackBy` / `@for` track

### 📁 `05-angular/10-testing/`
- 📄 TestBed configuration
- 📄 Component testing
- 📄 Service testing with `HttpTestingController`
- 📄 Marble testing for RxJS
- 📄 Spectator library
- 📄 E2E with Cypress/Playwright

---

## 📁 `06-internals/` *(Deep "How It Works" Folder)*

### 📁 `06-internals/01-javascript-engine/`
- 📄 Execution context and lexical environment
- 📄 Call stack and stack frames
- 📄 Heap memory and object allocation
- 📄 Closure mechanics (scope chain in detail)
- 📄 Prototype chain resolution algorithm
- 📄 Event loop: tasks, microtasks, render steps
- 📄 `queueMicrotask`, `requestAnimationFrame`, `requestIdleCallback`
- 📄 Garbage collection (mark-and-sweep, generational GC)
- 📄 Memory leaks: closures, listeners, detached DOM
- 📄 V8 pipeline: Parser → Ignition (bytecode) → TurboFan (JIT)
- 📄 Hidden classes and inline caches
- 📄 Proxy and Reflect (foundation for reactivity)
- 📄 ESM resolution algorithm and live bindings

### 📁 `06-internals/02-react/`
- 📄 Virtual DOM vs Fiber architecture
- 📄 Fiber node structure (type, props, alternate, return, child, sibling)
- 📄 Reconciliation algorithm (diffing rules)
- 📄 Render phase vs Commit phase
- 📄 Double buffering (current vs work-in-progress tree)
- 📄 Hooks implementation (linked list per fiber)
- 📄 Why the "Rules of Hooks" exist (implementation reason)
- 📄 Concurrent rendering and time-slicing
- 📄 Lanes model and priorities
- 📄 Scheduler package (cooperative scheduling)
- 📄 Batching mechanics (legacy vs automatic)
- 📄 Suspense internals (throwing promises, fallbacks)
- 📄 Server Components serialization (RSC payload)
- 📄 Hydration and selective hydration
- 📄 `useSyncExternalStore` and tearing prevention

### 📁 `06-internals/03-angular/`
- 📄 Change Detection algorithm (top-down tree traversal)
- 📄 Zone.js patching of async APIs (how it works)
- 📄 OnPush strategy internals
- 📄 Ivy renderer (instruction-based templates)
- 📄 AOT compilation pipeline
- 📄 Template compiler output structure
- 📄 Dependency Injection internals (injector trees)
- 📄 Element injector vs environment injector
- 📄 Signals implementation (push-pull reactive graph)
- 📄 Glitch-free propagation in signals
- 📄 Zoneless change detection (signal-driven)
- 📄 Hydration (incremental and non-destructive)
- 📄 `@defer` block compilation
- 📄 RxJS internals: schedulers, lift, operator composition

### 📁 `06-internals/04-browser/`
- 📄 Critical rendering path
- 📄 Parse → DOM/CSSOM → Render tree → Layout → Paint → Composite
- 📄 Reflow vs Repaint vs Compositing
- 📄 Browser process model (renderer, GPU, network)
- 📄 HTTP/1.1 vs HTTP/2 vs HTTP/3
- 📄 Same-origin policy and CORS
- 📄 Cookies, SameSite, CSRF
- 📄 Content Security Policy (CSP)

---

## 📁 `07-concepts/` *(High-Level Cross-Cutting Concepts)*

### 📁 `07-concepts/01-state-management/`
- 📄 State categories: server, client, URL, form, UI, ephemeral
- 📄 Local vs lifted vs global state — decision criteria
- 📄 Flux architecture and unidirectional data flow
- 📄 Redux pattern (actions, reducers, store, middleware)
- 📄 Atom-based state (Jotai, Recoil)
- 📄 Proxy-based state (Valtio, MobX)
- 📄 Selector pattern and memoization
- 📄 Immutability and structural sharing
- 📄 Normalization of relational state
- 📄 Server state vs client state separation
- 📄 Optimistic updates and rollback
- 📄 Persistence and rehydration

### 📁 `07-concepts/02-dependency-injection/`
- 📄 What DI solves (coupling, testability)
- 📄 Service locator vs DI container
- 📄 Inversion of Control (IoC)
- 📄 Constructor vs setter vs interface injection
- 📄 Singleton vs transient vs scoped lifetimes
- 📄 DI in Angular vs DI in React (Context as poor-man's DI)
- 📄 Token-based injection

### 📁 `07-concepts/03-http-and-networking/`
- 📄 HTTP methods and semantics
- 📄 Status codes (categories and common ones)
- 📄 Headers (request, response, common ones)
- 📄 REST principles and constraints
- 📄 GraphQL fundamentals (queries, mutations, subscriptions)
- 📄 gRPC and tRPC concepts
- 📄 WebSockets vs SSE vs long polling
- 📄 Caching: ETag, Cache-Control, stale-while-revalidate
- 📄 CORS preflight and credentials
- 📄 Authentication: cookies, JWT, OAuth2, OIDC, session
- 📄 Rate limiting and retries with backoff

### 📁 `07-concepts/04-rendering-strategies/`
- 📄 CSR (Client-Side Rendering)
- 📄 SSR (Server-Side Rendering)
- 📄 SSG (Static Site Generation)
- 📄 ISR (Incremental Static Regeneration)
- 📄 PPR (Partial Prerendering)
- 📄 Streaming SSR
- 📄 Edge rendering
- 📄 Islands architecture
- 📄 Resumability (Qwik concept)
- 📄 When to choose what (decision matrix)

### 📁 `07-concepts/05-performance/`
- 📄 Core Web Vitals (LCP, INP, CLS) — what they measure
- 📄 TTFB, FCP, TTI, TBT
- 📄 Critical rendering path optimization
- 📄 Code splitting strategies
- 📄 Lazy loading (routes, components, images)
- 📄 Preloading, prefetching, preconnect
- 📄 Tree shaking and dead code elimination
- 📄 Bundle analysis (source-map-explorer, webpack-bundle-analyzer)
- 📄 Image optimization (formats, sizing, CDN)
- 📄 Font optimization (`font-display`, subsetting)
- 📄 Memoization and computational caching
- 📄 Virtualization for long lists
- 📄 Debouncing vs throttling
- 📄 Memory profiling

### 📁 `07-concepts/06-security/`
- 📄 XSS (stored, reflected, DOM-based) and prevention
- 📄 CSRF and mitigation strategies
- 📄 CSP (Content Security Policy) directives
- 📄 Clickjacking and `X-Frame-Options`
- 📄 SQL injection awareness (for full-stack)
- 📄 Prototype pollution
- 📄 Subresource Integrity (SRI)
- 📄 Secure cookie flags (HttpOnly, Secure, SameSite)
- 📄 OWASP Top 10
- 📄 Dependency vulnerability scanning
- 📄 Secrets management

### 📁 `07-concepts/07-accessibility/`
- 📄 WCAG principles (POUR)
- 📄 Semantic HTML first, ARIA second
- 📄 Keyboard navigation and focus management
- 📄 Focus trapping (modals, menus)
- 📄 Color contrast and visual accessibility
- 📄 Screen reader testing (NVDA, VoiceOver, JAWS)
- 📄 Reduced motion preferences
- 📄 Internationalization (i18n) and RTL support

### 📁 `07-concepts/08-testing/`
- 📄 Testing pyramid (unit, integration, E2E)
- 📄 Testing trophy (Kent C. Dodds model)
- 📄 Test doubles: stubs, spies, mocks, fakes
- 📄 Snapshot testing
- 📄 Property-based testing
- 📄 Visual regression testing
- 📄 Accessibility testing (axe-core)
- 📄 Code coverage (and its limits)
- 📄 TDD vs BDD

### 📁 `07-concepts/09-architecture-and-patterns/`
- 📄 SOLID principles in frontend
- 📄 DRY, KISS, YAGNI
- 📄 Component-driven development
- 📄 Atomic Design (atoms, molecules, organisms)
- 📄 Feature-Sliced Design
- 📄 Domain-Driven Design (frontend application)
- 📄 Micro-frontends (Module Federation, single-spa)
- 📄 Monorepo architecture (Nx, Turborepo)
- 📄 BFF (Backend for Frontend) pattern
- 📄 Hexagonal architecture in frontend
- 📄 Event-driven UI (event bus, pub-sub)

### 📁 `07-concepts/10-design-patterns/`
- 📄 Singleton, Factory, Builder (creational)
- 📄 Observer, Strategy, Command (behavioral)
- 📄 Adapter, Decorator, Facade, Proxy (structural)
- 📄 Module pattern
- 📄 Mediator pattern
- 📄 Repository pattern
- 📄 Saga pattern (async workflows)

### 📁 `07-concepts/11-build-and-deploy/`
- 📄 CI/CD pipelines
- 📄 Semantic versioning and changesets
- 📄 Containerization (Docker basics for frontend)
- 📄 Edge deployment (Vercel, Cloudflare Workers)
- 📄 CDN strategies
- 📄 Feature flags and progressive rollout
- 📄 Environment management
- 📄 Source maps in production

### 📁 `07-concepts/12-observability/`
- 📄 Error tracking (Sentry)
- 📄 Real User Monitoring (RUM)
- 📄 Synthetic monitoring
- 📄 OpenTelemetry for frontend
- 📄 Structured logging
- 📄 Performance budgets

---

## 📁 `08-ecosystem/` *(Libraries & Tools — What to Learn)*

### 📁 `08-ecosystem/01-react-libraries/`

#### 📁 `client-state/`
- 📄 Zustand (API, middleware, slices, persist)
- 📄 Jotai (atoms, derived atoms, async atoms)
- 📄 Redux Toolkit (createSlice, createAsyncThunk, RTK Query)
- 📄 Valtio (proxy-based)
- 📄 XState (state machines)

#### 📁 `server-state/`
- 📄 TanStack Query (queries, mutations, invalidation, infinite, prefetching)
- 📄 SWR
- 📄 RTK Query
- 📄 Apollo Client (GraphQL)

#### 📁 `routing/`
- 📄 React Router v6/v7
- 📄 TanStack Router (type-safe routing)

#### 📁 `forms-and-validation/`
- 📄 React Hook Form
- 📄 TanStack Form
- 📄 Zod
- 📄 Yup / Valibot

#### 📁 `ui-libraries/`
- 📄 shadcn/ui (philosophy: copy-paste, not install)
- 📄 Radix UI primitives
- 📄 Headless UI
- 📄 MUI / Material-UI
- 📄 Mantine
- 📄 Chakra UI / Park UI
- 📄 Ark UI

#### 📁 `styling/`
- 📄 Tailwind CSS (and Tailwind v4 features)
- 📄 Emotion / styled-components
- 📄 vanilla-extract
- 📄 CVA (class-variance-authority)
- 📄 clsx / tailwind-merge

#### 📁 `animation/`
- 📄 Framer Motion / Motion
- 📄 React Spring
- 📄 GSAP with React
- 📄 Auto-Animate

#### 📁 `tables-data-viz/`
- 📄 TanStack Table
- 📄 AG Grid
- 📄 Recharts / Visx / Nivo

#### 📁 `testing/`
- 📄 Vitest / Jest
- 📄 React Testing Library
- 📄 Playwright
- 📄 MSW (Mock Service Worker)
- 📄 Storybook

#### 📁 `meta-frameworks/`
- 📄 Next.js
- 📄 Remix / React Router 7
- 📄 Astro
- 📄 TanStack Start

### 📁 `08-ecosystem/02-angular-libraries/`

#### 📁 `state/`
- 📄 NgRx (Store, Effects, Entity, Router-Store)
- 📄 NgRx Signals (SignalStore)
- 📄 NgRx ComponentStore
- 📄 NGXS
- 📄 Akita / Elf
- 📄 RxAngular

#### 📁 `ui-libraries/`
- 📄 Angular Material + CDK
- 📄 PrimeNG
- 📄 Taiga UI
- 📄 Nebular
- 📄 ng-bootstrap
- 📄 Spartan/ui (shadcn for Angular)

#### 📁 `data-and-http/`
- 📄 Apollo Angular (GraphQL)
- 📄 `@ngneat/query` (TanStack Query port)
- 📄 `@tanstack/angular-query`

#### 📁 `forms/`
- 📄 Angular Reactive Forms (built-in deep dive)
- 📄 `@ngneat/reactive-forms`
- 📄 ngx-formly (dynamic forms)

#### 📁 `utilities/`
- 📄 ngxtension (signal utilities)
- 📄 `@ngneat/until-destroyed`
- 📄 `@angular/cdk` (Overlay, Portal, A11y, DragDrop, Virtual Scroll)

#### 📁 `animation/`
- 📄 `@angular/animations`
- 📄 GSAP with Angular
- 📄 Motion One

#### 📁 `testing/`
- 📄 Jest (preferred over Karma)
- 📄 Spectator
- 📄 Cypress / Playwright
- 📄 Marble testing (jest-marbles)

### 📁 `08-ecosystem/03-shared-tooling/`
- 📄 Vite, Webpack, esbuild, Turbopack, Rspack
- 📄 Nx, Turborepo (monorepos)
- 📄 ESLint, Prettier, Biome
- 📄 Husky, lint-staged, commitlint
- 📄 Changesets, semantic-release
- 📄 Storybook
- 📄 Sentry, LogRocket, Datadog RUM

---

## 📁 `09-interview-prep/`

### 📁 `09-interview-prep/01-javascript-questions/`
- 📄 Explain the event loop with a code example
- 📄 What's the difference between `==` and `===`?
- 📄 Closures: explain with a use case
- 📄 `this` binding in different contexts
- 📄 Promises vs async/await vs callbacks
- 📄 Difference between `map`, `mergeMap`, `switchMap`, `concatMap`
- 📄 Debounce vs throttle (implement both)
- 📄 Deep clone vs shallow clone (implement deep clone)
- 📄 Currying (implement curry function)
- 📄 Implement `Promise.all` from scratch
- 📄 Implement `bind` polyfill
- 📄 Memoization (implement)
- 📄 Event delegation explained
- 📄 Hoisting and TDZ
- 📄 Prototype vs class inheritance
- 📄 ESM vs CJS differences
- 📄 What is a generator? When to use?
- 📄 Explain WeakMap/WeakSet use cases

### 📁 `09-interview-prep/02-react-questions/`
- 📄 Virtual DOM vs Real DOM vs Fiber
- 📄 Reconciliation and the role of keys
- 📄 Why don't hooks work inside conditionals?
- 📄 `useMemo` vs `useCallback` vs `React.memo`
- 📄 `useEffect` vs `useLayoutEffect`
- 📄 How does Context cause re-renders?
- 📄 What is Suspense and how does it work internally?
- 📄 Server Components vs Client Components
- 📄 SSR vs SSG vs ISR
- 📄 Error boundaries — implement one
- 📄 Custom hook design — rules and naming
- 📄 How does React batch updates?
- 📄 What is hydration mismatch and how to fix?
- 📄 Controlled vs uncontrolled — when to use which
- 📄 Higher-Order Component vs Custom Hook
- 📄 Render props vs hooks
- 📄 How would you debug a slow React app?
- 📄 Concurrent rendering — what changed?

### 📁 `09-interview-prep/03-angular-questions/`
- 📄 Difference between Angular modules and standalone components
- 📄 Change detection: how does it work?
- 📄 OnPush vs Default — when to use OnPush?
- 📄 Zone.js — what does it do?
- 📄 Signals vs Observables — when to use what?
- 📄 `switchMap` vs `mergeMap` vs `concatMap` vs `exhaustMap`
- 📄 Hot vs cold observables
- 📄 DI hierarchy — explain with example
- 📄 `providedIn: 'root'` vs module-level providers
- 📄 Template-driven vs Reactive forms
- 📄 Pure vs impure pipes
- 📄 Structural vs attribute directives
- 📄 `ng-template` vs `ng-container` vs `ng-content`
- 📄 Route guards — types and use cases
- 📄 Lazy loading: modules vs routes
- 📄 AOT vs JIT compilation
- 📄 SSR with Angular Universal
- 📄 How do signals avoid the need for Zone.js?
- 📄 ControlValueAccessor — when and how

### 📁 `09-interview-prep/04-system-design-frontend/`
- 📄 Design Twitter feed (infinite scroll, virtualization, optimistic updates)
- 📄 Design Google Docs (CRDT, OT, real-time sync)
- 📄 Design a chat app (WebSocket, presence, delivery receipts)
- 📄 Design an autocomplete (debouncing, caching, ranking)
- 📄 Design a video streaming UI (HLS/DASH, buffering)
- 📄 Design a design system (tokens, theming, primitives)
- 📄 Design a multi-tenant SaaS frontend
- 📄 Design an offline-first PWA
- 📄 Design a dashboard with real-time updates
- 📄 Design image gallery with lazy loading and prefetching
- 📄 Design a notifications system
- 📄 Design micro-frontends architecture
- 📄 Design a file upload (resumable, chunked, parallel)

### 📁 `09-interview-prep/05-coding-challenges/`
- 📄 Implement a debounce hook
- 📄 Implement an infinite scroll component
- 📄 Implement a virtualized list
- 📄 Implement a typeahead/autocomplete
- 📄 Implement a modal with focus trap
- 📄 Implement an accessible accordion
- 📄 Implement drag-and-drop reordering
- 📄 Implement a tab component (with keyboard nav)
- 📄 Implement a star rating component
- 📄 Implement undo/redo
- 📄 Implement a polling hook with backoff
- 📄 Implement a cancellable fetch hook
- 📄 Implement a state machine for a form wizard
- 📄 Build a tic-tac-toe / connect-four
- 📄 Build a stopwatch / countdown timer

### 📁 `09-interview-prep/06-css-questions/`
- 📄 Specificity calculation
- 📄 Flexbox vs Grid — when to use which
- 📄 Stacking context creation rules
- 📄 Center an element (5 ways)
- 📄 Sticky vs fixed positioning
- 📄 `em` vs `rem`
- 📄 BEM vs utility-first tradeoffs
- 📄 CSS containment
- 📄 Container queries vs media queries

### 📁 `09-interview-prep/07-behavioral-and-architectural/`
- 📄 Tell me about a time you optimized performance
- 📄 How do you decide between two libraries?
- 📄 How do you onboard onto a legacy codebase?
- 📄 How do you handle technical debt?
- 📄 Describe a difficult bug you fixed
- 📄 How do you design a component API?
- 📄 How do you approach accessibility?
- 📄 How do you mentor junior developers?

---

## 📁 `10-topics-bonus/` *(Advanced & Trending)*

- 📄 Web Components interop with React/Angular
- 📄 WASM and frontend integration
- 📄 PWAs and Service Workers (caching strategies)
- 📄 WebGPU / WebGL basics
- 📄 AI-assisted UI (streaming LLM responses, optimistic rendering)
- 📄 Real-time collaboration (CRDTs, Yjs, Automerge)
- 📄 Local-first software architecture
- 📄 Edge computing and frontend
- 📄 Qwik and resumability
- 📄 Solid.js (signals done differently — useful contrast)
- 📄 HTMX and hypermedia-driven UIs
- 📄 Module Federation in production

---

## 🎯 Suggested Generation Order for the Agent

1. `01-html` → `02-css` → `03-javascript` (foundation, fast pass)
2. `06-internals/01-javascript-engine` (deep JS *before* frameworks)
3. `04-react` (core → hooks → patterns → advanced)
4. `06-internals/02-react`
5. `05-angular` (core → DI → signals → RxJS → advanced)
6. `06-internals/03-angular`
7. `07-concepts` (cross-cutting; can be done in parallel)
8. `08-ecosystem` (reference material, generate per library)
9. `06-internals/04-browser`
10. `09-interview-prep` (last; benefits from everything above)
11. `10-topics-bonus` (optional / continuous learning)

---

**Total leaf topics:** ~450 discrete items. Each is small enough for an agent to generate focused, deep content (1–3 pages) without ballooning scope.