# Frontend Engineering Mastery: React & Angular from Internals to Interview

A senior-level reference curriculum covering the full stack of modern frontend engineering — from HTML semantics and CSS internals to React Fiber, Angular signals, browser rendering pipelines, and system design. Built for experienced engineers who want to go deeper, not broader.

---

## What You Get

**800+ deep-dive documents** across 10 sections. Every file targets the questions a senior engineer actually gets asked — not just what an API does, but *why* it works that way, *when* to use it, and *what breaks* if you use it wrong. Each document includes working code examples, internal mechanics where relevant, and a model interview Q&A section.

---

## What's Inside

### [01. HTML](./01-html/)
Beyond tags — semantic document structure, accessibility tree, ARIA patterns, the Intersection/Resize/Mutation Observer APIs, Web Components, the Popover and Dialog APIs, and how browsers actually build the DOM.

### [02. CSS](./02-css/)
Layout systems (Flexbox, Grid, Subgrid), stacking contexts, the modern CSS feature set (`:has()`, container queries, cascade layers, native nesting, `oklch`), CSS architecture (BEM, utility-first, CSS Modules, CSS-in-JS trade-offs), and responsive/accessible design patterns.

### [03. JavaScript](./03-javascript/)
The language in depth — hoisting, TDZ, closure mechanics, prototype chain, `this` binding rules, the async model (callbacks → Promises → async/await → generators), ESM vs CommonJS live bindings, tree shaking prerequisites, and the full TypeScript type system including conditional types, mapped types, and declaration files.

### [04. React](./04-react/)
Every hook explained with implementation-level detail, all major composition patterns (compound components, render props, HOCs, state reducer), React 18/19 features (concurrent rendering, automatic batching, Server Components, Server Actions, streaming SSR), meta-frameworks (Next.js App Router, Remix, Astro), and testing with React Testing Library and MSW.

### [05. Angular](./05-angular/)
Modern Angular top to bottom — standalone components, the new control flow syntax, signal-based reactivity (`signal()`, `computed()`, `effect()`, `linkedSignal()`, `resource()`), the full RxJS operator catalogue, typed reactive forms, functional guards and interceptors, `@defer` blocks, non-destructive hydration, and zoneless change detection.

### [06. Internals](./06-internals/)
The section most resources skip. How V8 actually compiles JavaScript (Ignition → TurboFan, hidden classes, inline caches), React's Fiber architecture (fiber node structure, lanes model, double buffering, hooks as a linked list, Suspense internals), Angular's Ivy renderer and signal push-pull graph, and the browser's full rendering pipeline (critical path, reflow vs repaint vs composite, process model, HTTP/1.1 vs HTTP/2 vs HTTP/3).

### [07. Concepts](./07-concepts/)
Framework-agnostic engineering depth — state management patterns (Flux, Redux, atom-based, proxy-based), rendering strategies (CSR → SSR → SSG → ISR → PPR → streaming → resumability), Core Web Vitals and performance optimisation, security (XSS, CSRF, CSP, OWASP Top 10, prototype pollution), accessibility (WCAG, focus trapping, screen readers, i18n/RTL), testing strategy (pyramid vs trophy, property-based testing, visual regression), architecture patterns (SOLID, DDD, micro-frontends, BFF, hexagonal, Feature-Sliced Design), all GoF design patterns applied to frontend, CI/CD, and observability.

### [08. Ecosystem](./08-ecosystem/)
Practical guides for the libraries you'll actually use:

**React:** Zustand, Jotai, Redux Toolkit, Valtio, XState, TanStack Query, SWR, RTK Query, Apollo Client, React Router, TanStack Router, React Hook Form, Zod, shadcn/ui, Radix UI, MUI, Mantine, Tailwind CSS, Framer Motion, React Spring, TanStack Table, AG Grid, Recharts, Vitest, Playwright, MSW, Storybook, Next.js, Remix, Astro, TanStack Start

**Angular:** NgRx (Store, Effects, Entity, SignalStore), NGXS, Angular Material + CDK, PrimeNG, Taiga UI, Spectator, ngxtension, ngx-formly, TanStack Query for Angular

**Shared:** Vite, Webpack, esbuild, Nx, Turborepo, ESLint, Prettier, Biome, Sentry, Datadog

### [09. Interview Prep](./09-interview-prep/)
Structured preparation across six question categories — JavaScript fundamentals (implement `Promise.all`, debounce, memoization, deep clone, `bind`), React deep dives (Fiber, hooks, Server Components, hydration), Angular internals (change detection, signals vs observables, DI hierarchy), CSS mechanics, 13 frontend system design problems (Twitter feed, Google Docs, autocomplete, chat app, file upload, micro-frontends), 18 coding challenges with full implementations, and behavioral questions.

### [10. Bonus Topics](./10-topics-bonus/)
Web Components and framework interop, WebAssembly (Rust and C++ to Wasm), Progressive Web Apps, WebGPU/WebGL/Three.js, and emerging patterns: AI-assisted UI, CRDTs (Yjs/Automerge), local-first architecture, Qwik resumability, Solid.js fine-grained reactivity, HTMX, Module Federation, WebRTC, View Transitions API, CSS Houdini, and the Temporal API.

### [11. Real-World Scenarios](./11-real-world-scenarios/)

End-to-end walkthroughs for 70+ complex UI problems — multi-level responsive navigation, virtualized tables, real-time feeds, file upload pipelines, mobile-specific patterns, and more. Each scenario covers the CSS/HTML foundation, the React implementation, and the Angular implementation, with accessibility checklists and production pitfalls. Prefixed by category: `NAV-`, `LAYOUT-`, `FORM-`, `DATA-`, `MODAL-`, `PERF-`, `A11Y-`, `ANIM-`, `AUTH-`, `UPLOAD-`, `RT-`, `MOBILE-`.

---

## Suggested Reading Order

**If you're preparing for senior interviews:**
1. `06-internals/` — this is where most candidates are weak
2. `07-concepts/` — cross-cutting patterns examiners love
3. Your primary framework section (`04-react/` or `05-angular/`)
4. `09-interview-prep/04-system-design/` + `05-coding-challenges/`

**If you're building framework depth:**
1. `01-html/` → `02-css/` → `03-javascript/` (foundation pass)
2. `04-react/` or `05-angular/` (core → hooks/DI → patterns → performance)
3. `06-internals/` for the matching framework
4. `08-ecosystem/` for the library landscape

**If you're filling specific gaps:**
Go directly to the relevant section — every document is self-contained.

---

## Content Format

Every document is written at senior engineer depth. Expect:
- Mechanism-level explanations (not just "what" but "how" and "why")
- Working, typed code examples throughout
- Comparison tables for similar APIs or patterns
- Pitfalls and anti-patterns called out explicitly
- Model interview Q&A at the end of each file
