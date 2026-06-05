# Whiteboard: Architecture Discussion

## The Idea

**In plain English:** Architecture discussion is when you explain how you would organize and connect all the different parts of a software system — like deciding how many rooms a building needs, what each room is for, and how people move between them — before you start building.

**Real-world analogy:** Imagine planning a new shopping mall. Before construction begins, you sit down with the owners and draw a map: where the food court goes, how many entrances there are, which shops sit near each other, and how delivery trucks reach the back of each store. Every choice has a reason ("the food court goes in the middle so foot traffic passes all stores") and a trade-off ("a bigger food court means fewer retail spots").

- The mall blueprint = the system architecture diagram
- Each section of the mall (food court, clothing stores, parking) = each software service or module
- The corridors and escalators connecting sections = the APIs and data flows between services
- The mall management deciding which shops get prime spots = the team making trade-off decisions about performance, cost, and complexity

---

## Overview

Master discussing architecture decisions, trade-offs, scalability considerations, technology choices, and balancing competing concerns like performance vs maintainability during whiteboard interviews.

## Architecture Discussion Framework

### Step 1: Clarify Requirements
- Functional requirements (what the system does)
- Non-functional requirements (scalability, performance, security)
- Constraints (budget, timeline, team skills)
- Current state vs future state

### Step 2: Identify Components
- Break system into logical components
- Define responsibilities
- Identify interfaces between components
- Consider data flow

### Step 3: Discuss Trade-offs
- Performance vs Complexity
- Consistency vs Availability
- Flexibility vs Simplicity
- Cost vs Quality

### Step 4: Justify Decisions
- Explain reasoning
- Reference industry patterns
- Consider alternatives
- Discuss mitigation strategies

## Example 1: E-commerce Frontend Architecture

### Requirements

**Functional:**
- Product catalog browsing
- Shopping cart
- Checkout flow
- User authentication
- Order history
- Real-time inventory updates

**Non-Functional:**
- Support 10K concurrent users
- Page load < 2 seconds
- 99.9% uptime
- Mobile-responsive
- SEO-friendly

**Constraints:**
- 3-month timeline
- Team of 5 developers (React experience)
- Limited DevOps resources
- Must support older browsers (last 2 versions)

### Architecture Design

```
┌────────────────────────────────────────────────────────────┐
│                    Client Application                       │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────┐      │
│  │   Product   │  │  Shopping    │  │    User     │      │
│  │   Catalog   │  │     Cart     │  │   Profile   │      │
│  │   Module    │  │    Module    │  │   Module    │      │
│  └─────────────┘  └──────────────┘  └─────────────┘      │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │           State Management (Redux Toolkit)            │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │              API Client Layer                         │ │
│  │  (Axios + React Query for caching)                    │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                             │
└────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────┐
│                    API Gateway                              │
│            (Rate limiting, Auth, Routing)                   │
└────────────────────────────────────────────────────────────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ Product  │  │   Cart   │  │   User   │
        │ Service  │  │ Service  │  │ Service  │
        └──────────┘  └──────────┘  └──────────┘
```

### Technology Decisions

#### 1. Frontend Framework: React

**Decision:** Use React with TypeScript

**Rationale:**
- Team has React experience (minimize learning curve)
- Large ecosystem and community support
- Excellent performance with Virtual DOM
- Strong TypeScript support
- Component reusability

**Alternatives Considered:**
- **Vue.js:** 
  - Pros: Easier learning curve, built-in state management
  - Cons: Team lacks experience, smaller ecosystem
  - Verdict: Not chosen due to team skills

- **Angular:**
  - Pros: Batteries-included, strong TypeScript support
  - Cons: Steeper learning curve, more boilerplate
  - Verdict: Not chosen due to timeline constraints

**Trade-offs:**
- React requires more architectural decisions (state management, routing)
- Need to assemble ecosystem (React Router, Redux, etc.)
- More flexibility but less opinionated than Angular

#### 2. State Management: Redux Toolkit

**Decision:** Redux Toolkit for global state, React Query for server state

**Rationale:**
- Redux Toolkit reduces boilerplate compared to plain Redux
- Predictable state updates
- Excellent DevTools for debugging
- React Query handles API caching and synchronization
- Separation of client state vs server state

**Alternatives:**
- **Context API:**
  - Pros: Built-in, simple
  - Cons: Performance issues with frequent updates, no DevTools
  - Verdict: Too limited for complex e-commerce state

- **MobX:**
  - Pros: Less boilerplate, reactive
  - Cons: Team unfamiliar, less predictable
  - Verdict: Learning curve doesn't justify benefits

- **Zustand:**
  - Pros: Simple, minimal boilerplate
  - Cons: Less mature ecosystem, smaller community
  - Verdict: Too new for production project

**Trade-offs:**
- Redux adds complexity and boilerplate
- React Query increases bundle size
- Need to learn Redux patterns
- Benefits: Predictability, debugging, time-travel debugging

#### 3. Styling: CSS Modules + Tailwind CSS

**Decision:** Tailwind CSS for utility classes, CSS Modules for component-specific styles

**Rationale:**
- Tailwind enables rapid development
- CSS Modules prevent style conflicts
- Hybrid approach balances flexibility and speed
- Good DX with autocomplete

**Alternatives:**
- **Styled-components:**
  - Pros: Dynamic styles, theme support
  - Cons: Runtime performance cost, larger bundle
  - Verdict: Performance concerns for high-traffic site

- **Plain CSS:**
  - Pros: No additional dependencies
  - Cons: Naming conflicts, no utilities
  - Verdict: Too slow for tight timeline

**Trade-offs:**
- Tailwind increases HTML verbosity
- Learning curve for team
- Build step required
- Benefits: Fast development, consistent design system

#### 4. Routing: React Router v6

**Decision:** React Router v6 with code splitting

**Rationale:**
- Industry standard
- Code splitting reduces initial bundle
- Supports nested routes
- Good TypeScript support

**Implementation:**
```typescript
// Route configuration with code splitting
const routes = [
  {
    path: '/',
    element: <Layout />,
    children: [
      {
        index: true,
        element: <HomePage />,
      },
      {
        path: 'products',
        lazy: () => import('./pages/Products'),
      },
      {
        path: 'products/:id',
        lazy: () => import('./pages/ProductDetail'),
      },
      {
        path: 'cart',
        lazy: () => import('./pages/Cart'),
      },
      {
        path: 'checkout',
        lazy: () => import('./pages/Checkout'),
        loader: authLoader, // Protected route
      },
    ],
  },
];
```

### Architecture Decisions

#### Micro-Frontend vs Monolith

**Decision:** Start with monolithic frontend, design for future micro-frontend migration

**Rationale:**
- **Current State:** Team size (5) and timeline (3 months) don't justify micro-frontend complexity
- **Future State:** Design modules with clear boundaries to enable future split
- **Migration Path:** Use module federation when team grows to 15+

**Architecture for Future Migration:**
```typescript
// Clear module boundaries
/src
  /modules
    /catalog
      /components
      /hooks
      /store
      /api
      index.ts  // Public API
    /cart
    /checkout
    /user
```

**Trade-offs:**
- Monolith is simpler initially
- Risk of tight coupling if not careful
- Mitigation: Enforce module boundaries with ESLint rules

#### Server-Side Rendering (SSR)

**Decision:** Implement SSR with Next.js for SEO and performance

**Rationale:**
- **SEO:** Product pages must be crawlable
- **Performance:** Faster First Contentful Paint
- **User Experience:** Better perceived performance

**Implementation Strategy:**
```typescript
// pages/products/[id].tsx
export async function getServerSideProps(context) {
  const { id } = context.params;
  const product = await fetchProduct(id);

  return {
    props: {
      product,
    },
  };
}

// pages/products/index.tsx
export async function getStaticProps() {
  const products = await fetchProducts();

  return {
    props: {
      products,
    },
    revalidate: 60, // ISR: Revalidate every 60 seconds
  };
}
```

**Alternatives:**
- **Client-Side Rendering (CSR):**
  - Pros: Simpler, no Node.js server
  - Cons: Poor SEO, slower initial load
  - Verdict: Not acceptable for e-commerce

- **Static Site Generation (SSG):**
  - Pros: Best performance, no server
  - Cons: Can't handle dynamic data
  - Verdict: Use for static pages only

**Trade-offs:**
- SSR adds infrastructure complexity
- Requires Node.js server
- More complex debugging
- Benefits: Better SEO, faster perceived performance

#### Data Fetching Strategy

**Decision:** React Query for client-side data fetching and caching

**Rationale:**
- Automatic caching and revalidation
- Optimistic updates
- Request deduplication
- Retry logic and error handling

**Implementation:**
```typescript
// API client with React Query
function useProduct(id: string) {
  return useQuery({
    queryKey: ['product', id],
    queryFn: () => fetchProduct(id),
    staleTime: 5 * 60 * 1000, // 5 minutes
    cacheTime: 10 * 60 * 1000, // 10 minutes
  });
}

// Optimistic updates for cart
function useAddToCart() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: addToCart,
    onMutate: async (newItem) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['cart'] });

      // Snapshot previous value
      const previousCart = queryClient.getQueryData(['cart']);

      // Optimistically update
      queryClient.setQueryData(['cart'], (old: any) => ({
        ...old,
        items: [...old.items, newItem],
      }));

      return { previousCart };
    },
    onError: (err, newItem, context) => {
      // Rollback on error
      queryClient.setQueryData(['cart'], context?.previousCart);
    },
    onSettled: () => {
      // Refetch after mutation
      queryClient.invalidateQueries({ queryKey: ['cart'] });
    },
  });
}
```

### Performance Optimization Strategy

#### 1. Code Splitting

```typescript
// Route-level splitting
const ProductDetail = lazy(() => import('./pages/ProductDetail'));
const Cart = lazy(() => import('./pages/Cart'));

// Component-level splitting
const ProductReviews = lazy(() => import('./components/ProductReviews'));

// Dynamic import for heavy libraries
async function loadCharts() {
  const { Chart } = await import('chart.js');
  return Chart;
}
```

#### 2. Image Optimization

```typescript
// Use Next.js Image component
<Image
  src={product.image}
  alt={product.name}
  width={400}
  height={400}
  loading="lazy"
  quality={85}
  formats={['image/webp', 'image/avif']}
/>

// Or implement custom optimization
function OptimizedImage({ src, alt }: ImageProps) {
  const srcSet = `
    ${src}?w=400&q=85 400w,
    ${src}?w=800&q=85 800w,
    ${src}?w=1200&q=85 1200w
  `;

  return (
    <picture>
      <source srcSet={srcSet} type="image/webp" />
      <img src={src} alt={alt} loading="lazy" />
    </picture>
  );
}
```

#### 3. Caching Strategy

```typescript
// Service Worker for offline support
// sw.js
const CACHE_NAME = 'ecommerce-v1';
const urlsToCache = [
  '/',
  '/products',
  '/static/css/main.css',
  '/static/js/main.js',
];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => {
      return cache.addAll(urlsToCache);
    })
  );
});

// Cache-first strategy for static assets
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((response) => {
      return response || fetch(event.request);
    })
  );
});
```

## Example 2: Monolith vs Microservices Discussion

### Scenario

Design architecture for a dashboard application that will grow from 5 to 50 features over 2 years. Team will grow from 10 to 50 developers.

### Decision Matrix

| Factor | Monolith | Microservices | Decision |
|--------|----------|---------------|----------|
| **Team Size** | Current: 10 (good) | Current: too complex | Start monolith |
| **Future Team** | Future: 50 (problematic) | Future: 50 (good) | Plan migration |
| **Deployment** | Simple | Complex | Start simple |
| **Technology** | One stack | Polyglot | Start unified |
| **Scaling** | Vertical only | Independent | Start vertical |
| **Development** | Faster initially | Slower initially | Start monolith |

### Architecture Evolution

**Phase 1 (Months 1-6): Modular Monolith**
```
Frontend (React SPA)
    ↓
API Gateway
    ↓
Monolith Backend (clear module boundaries)
```

**Phase 2 (Months 7-12): Extract First Service**
```
Frontend
    ↓
API Gateway
    ↓  ↓
Monolith  Auth Service (first extraction)
```

**Phase 3 (Months 13-24): Microservices**
```
Frontend (Module Federation)
    ↓
API Gateway
    ↓  ↓  ↓  ↓
Service1  Service2  Service3  Service4
```

### Justification

**Why Start with Monolith:**
- Faster initial development
- Simpler deployment and operations
- Easier to refactor early
- Lower infrastructure costs
- Team can focus on business features

**Why Prepare for Microservices:**
- Eventual team scaling requires it
- Enable independent deployments
- Allow technology diversity
- Better fault isolation

**Migration Strategy:**
- Design with clear module boundaries
- Use dependency injection
- Implement API versioning
- Extract services when team hits 20+ developers
- Start with most independent modules

## Key Takeaways

1. **Requirements First:** Always start by clarifying functional and non-functional requirements and constraints.

2. **Trade-offs are Inevitable:** Every decision has trade-offs - explicitly discuss pros/cons of each option.

3. **Team Context Matters:** Consider team size, skills, and timeline when choosing technologies.

4. **Plan for Evolution:** Design for current needs but enable future scaling and migration.

5. **Performance Budget:** Set and enforce performance budgets (bundle size, load time, etc.).

6. **Document Decisions:** Use ADRs (Architecture Decision Records) to capture reasoning.

7. **Gradual Migration:** Prefer incremental changes over big-bang migrations.

8. **Measure Everything:** Implement monitoring and analytics to validate architectural decisions.

9. **Start Simple:** Choose simpler solutions initially, add complexity only when needed.

10. **Communication is Key:** Articulate reasoning clearly, consider alternatives, justify with data.

## Whiteboard Discussion Tips

### Do:
- Ask clarifying questions about requirements
- Draw diagrams to visualize architecture
- Discuss multiple alternatives
- Explain trade-offs explicitly
- Consider future scalability
- Mention monitoring and observability
- Discuss testing strategy
- Reference industry patterns (MVC, BFF, etc.)
- Be honest about trade-offs
- Think out loud

### Don't:
- Jump to solutions without understanding requirements
- Ignore constraints (team size, timeline, budget)
- Present one solution as "perfect"
- Ignore operational concerns
- Forget about testing and monitoring
- Dismiss interviewer's suggestions
- Get defensive about decisions
- Over-engineer for current needs
- Ignore team skills and experience
- Forget about developer experience

## Common Architecture Patterns

### Backend for Frontend (BFF)

```
Mobile App ──→ Mobile BFF ──┐
                             ├──→ Microservices
Web App ────→ Web BFF ──────┘
```

**When to Use:** Different clients need different data shapes

### Event-Driven Architecture

```
Service A ──→ Event Bus ──→ Service B
                      ├───→ Service C
                      └───→ Service D
```

**When to Use:** Loose coupling, async processing, high scalability

### CQRS (Command Query Responsibility Segregation)

```
Commands ──→ Write Model ──→ Database
                              │
Queries ───→ Read Model ──────┘
```

**When to Use:** Different optimization needs for reads vs writes
