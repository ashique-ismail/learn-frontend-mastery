# React Server Components (RSC)

## The Idea

**In plain English:** React Server Components are a way to build parts of a website entirely on the server (the computer that runs your app), so those parts never need to send any code to the visitor's browser. A "component" here just means a reusable chunk of your web page's UI, like a product card or a navigation bar.

**Real-world analogy:** Think of a restaurant kitchen. The chef (server) prepares a fully cooked meal and sends only the finished plate to your table — you never see the raw ingredients, the recipe, or the kitchen equipment. Compare that to a meal-kit delivery (client-side), where the box of raw ingredients is shipped to your house and you cook it yourself.

- The chef in the kitchen = the server running the React Server Component
- The finished plate sent to the table = the HTML/UI output sent to the browser
- The meal-kit box of raw ingredients shipped to you = the JavaScript bundle downloaded by the browser for a Client Component

---

## Table of Contents
- [Introduction](#introduction)
- [What are Server Components?](#what-are-server-components)
- [RSC Architecture](#rsc-architecture)
- [Client vs Server Components](#client-vs-server-components)
- ["use client" Directive](#use-client-directive)
- [Async Server Components](#async-server-components)
- [Component Boundaries](#component-boundaries)
- [RSC Payload Format](#rsc-payload-format)
- [When to Use RSC](#when-to-use-rsc)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

React Server Components (RSC) is a paradigm shift that allows components to render on the server, sending only the necessary UI to the client. Unlike traditional SSR, Server Components never re-render on the client and can directly access server-side resources like databases and filesystems.

## What are Server Components?

Server Components are React components that run only on the server. They execute during the build or request time and send a serialized representation to the client.

**Key Characteristics:**
- Zero JavaScript sent to client
- Direct access to server resources (databases, filesystems)
- Can use secret environment variables
- Never re-render on client
- Cannot use state, effects, or browser APIs
- Can be async and use await

**Traditional Component (Client):**
```typescript
// ❌ Client-side data fetching
'use client';

function ProductList() {
  const [products, setProducts] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/products')
      .then(res => res.json())
      .then(data => {
        setProducts(data);
        setLoading(false);
      });
  }, []);

  if (loading) return <Spinner />;

  return (
    <ul>
      {products.map(product => (
        <li key={product.id}>{product.name}</li>
      ))}
    </ul>
  );
}
```

**Server Component:**
```typescript
// ✅ Server Component (default in App Router)
async function ProductList() {
  // Fetch directly on server (no API route needed)
  const products = await db.product.findMany();

  return (
    <ul>
      {products.map(product => (
        <li key={product.id}>{product.name}</li>
      ))}
    </ul>
  );
}
```

## RSC Architecture

### How RSC Works

```
┌─────────────────────────────────────────────────────────┐
│                         SERVER                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. Server Component renders                            │
│     ├─ Fetch data from DB                              │
│     ├─ Access filesystem                               │
│     └─ Use secret env vars                             │
│                                                         │
│  2. Serialize to RSC payload                           │
│     ├─ Component tree structure                        │
│     ├─ Props data                                      │
│     └─ Placeholders for Client Components              │
│                                                         │
│  3. Send payload to client                             │
│     └─ Minimal, compressed format                      │
│                                                         │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                        CLIENT                           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  4. Receive RSC payload                                │
│                                                         │
│  5. Reconcile with Client Components                   │
│     ├─ Hydrate Client Components                      │
│     ├─ Compose with Server Component output           │
│     └─ Final UI                                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Data Flow Example

```typescript
// app/page.tsx (Server Component)
async function HomePage() {
  // Runs on server only
  const user = await getCurrentUser();
  const posts = await db.post.findMany({
    where: { authorId: user.id },
    include: { comments: true }
  });

  return (
    <main>
      <h1>Welcome {user.name}</h1>
      <PostList posts={posts} />
    </main>
  );
}

// Server Component: no JS sent to client
async function PostList({ posts }: { posts: Post[] }) {
  return (
    <div>
      {posts.map(post => (
        <PostCard key={post.id} post={post} />
      ))}
    </div>
  );
}

// Client Component: interactive, JS sent to client
'use client';

function PostCard({ post }: { post: Post }) {
  const [liked, setLiked] = useState(false);

  return (
    <article>
      <h2>{post.title}</h2>
      <p>{post.content}</p>
      <button onClick={() => setLiked(!liked)}>
        {liked ? '❤️' : '🤍'} Like
      </button>
      <CommentSection comments={post.comments} />
    </article>
  );
}
```

## Client vs Server Components

### Server Components (Default)

```typescript
// Server Component (no directive needed)
async function ServerComponent() {
  // ✅ Can do:
  const data = await fetch('https://api.example.com/data');
  const dbData = await db.query('SELECT * FROM users');
  const fileContent = await fs.readFile('data.json');
  const secret = process.env.SECRET_KEY;

  // ❌ Cannot do:
  // const [state, setState] = useState(0); // No hooks
  // useEffect(() => {}, []); // No effects
  // onClick={handleClick} // No event handlers
  // const ctx = useContext(MyContext); // No context (consumer)

  return <div>{/* JSX */}</div>;
}
```

### Client Components

```typescript
'use client';

// Client Component (needs directive)
function ClientComponent() {
  // ✅ Can do:
  const [state, setState] = useState(0);
  useEffect(() => {
    console.log('Mounted');
  }, []);
  
  const handleClick = () => {
    setState(s => s + 1);
  };

  const theme = useContext(ThemeContext);

  // ❌ Cannot do:
  // async function ClientComponent() {} // Cannot be async
  // const dbData = await db.query(); // No direct DB access
  // const secret = process.env.SECRET_KEY; // Exposed to client!

  return <button onClick={handleClick}>{state}</button>;
}
```

### Comparison Table

```typescript
interface ComponentCapabilities {
  serverComponents: {
    async: true;
    awaitInBody: true;
    databaseAccess: true;
    filesystemAccess: true;
    secretEnvVars: true;
    hooks: false;
    state: false;
    effects: false;
    eventHandlers: false;
    browserAPIs: false;
    bundleSize: 0; // Zero JS to client
  };
  
  clientComponents: {
    async: false;
    awaitInBody: false;
    databaseAccess: false;
    filesystemAccess: false;
    secretEnvVars: false; // Exposed to client!
    hooks: true;
    state: true;
    effects: true;
    eventHandlers: true;
    browserAPIs: true;
    bundleSize: number; // Adds JS to bundle
  };
}
```

## "use client" Directive

The `"use client"` directive marks a boundary between Server and Client Components.

### Basic Usage

```typescript
// components/Counter.tsx
'use client'; // Everything in this file is a Client Component

import { useState } from 'react';

export function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(c => c + 1)}>
      Count: {count}
    </button>
  );
}
```

### Boundary Rules

```typescript
// app/page.tsx (Server Component)
import { Counter } from '@/components/Counter'; // Client Component

async function Page() {
  const data = await fetchData(); // Server-only

  return (
    <div>
      {/* Server Component rendering */}
      <h1>{data.title}</h1>
      
      {/* Client Component boundary */}
      <Counter />
      
      {/* Back to Server Component */}
      <Footer />
    </div>
  );
}
```

### Propagation

```typescript
// components/Button.tsx
'use client';

// This file and ALL imports become Client Components
import { Icon } from './Icon'; // Also Client Component now
import { useTheme } from './theme'; // Also Client Component

export function Button({ children }: { children: React.ReactNode }) {
  const [clicked, setClicked] = useState(false);
  const theme = useTheme();

  return (
    <button onClick={() => setClicked(true)}>
      <Icon name="click" />
      {children}
    </button>
  );
}
```

### Granular Boundaries

```typescript
// ❌ BAD: Marking parent as client makes everything client
'use client';

import { Header } from './Header'; // Forced to be client
import { Footer } from './Footer'; // Forced to be client

function Layout({ children }) {
  const [sidebarOpen, setSidebarOpen] = useState(false);

  return (
    <>
      <Header /> {/* Could have been server component */}
      <Sidebar open={sidebarOpen} />
      <main>{children}</main>
      <Footer /> {/* Could have been server component */}
    </>
  );
}
```

```typescript
// ✅ GOOD: Only mark interactive parts as client
// Layout.tsx (Server Component)
import { Header } from './Header'; // Server Component
import { Sidebar } from './Sidebar'; // Client Component
import { Footer } from './Footer'; // Server Component

function Layout({ children }) {
  return (
    <>
      <Header /> {/* Server Component */}
      <Sidebar /> {/* Client Component */}
      <main>{children}</main>
      <Footer /> {/* Server Component */}
    </>
  );
}

// Sidebar.tsx
'use client'; // Only this needs to be client

function Sidebar() {
  const [open, setOpen] = useState(false);
  return <aside>{/* Interactive sidebar */}</aside>;
}
```

## Async Server Components

Server Components can be async and use `await` at the top level:

### Basic Async Pattern

```typescript
// app/products/page.tsx
async function ProductsPage() {
  // Multiple awaits in component body
  const categories = await db.category.findMany();
  const featured = await db.product.findMany({
    where: { featured: true }
  });

  return (
    <>
      <CategoryNav categories={categories} />
      <FeaturedProducts products={featured} />
    </>
  );
}
```

### Parallel Data Fetching

```typescript
async function DashboardPage() {
  // Fetch in parallel using Promise.all
  const [user, stats, recentActivity] = await Promise.all([
    db.user.findUnique({ where: { id: userId } }),
    db.stats.aggregate({ where: { userId } }),
    db.activity.findMany({ where: { userId }, take: 10 })
  ]);

  return (
    <Dashboard>
      <UserProfile user={user} />
      <StatsPanel stats={stats} />
      <ActivityFeed activity={recentActivity} />
    </Dashboard>
  );
}
```

### Sequential Data Fetching (When Needed)

```typescript
async function UserPostsPage({ userId }: { userId: string }) {
  // Sequential: need user before fetching posts
  const user = await db.user.findUnique({
    where: { id: userId }
  });

  if (!user) {
    notFound();
  }

  // Now fetch posts based on user data
  const posts = await db.post.findMany({
    where: {
      authorId: user.id,
      published: user.isPublic ? true : undefined
    }
  });

  return (
    <>
      <UserHeader user={user} />
      <PostList posts={posts} />
    </>
  );
}
```

### Error Handling in Async Components

```typescript
async function ProductPage({ id }: { id: string }) {
  try {
    const product = await db.product.findUnique({
      where: { id },
      include: { reviews: true }
    });

    if (!product) {
      notFound(); // Next.js helper
    }

    return <ProductDetail product={product} />;
  } catch (error) {
    console.error('Failed to load product:', error);
    throw error; // Caught by error boundary
  }
}
```

### Streaming with Suspense

```typescript
async function Page() {
  return (
    <>
      {/* Renders immediately */}
      <Header />

      {/* Suspends until data loads */}
      <Suspense fallback={<ProductsSkeleton />}>
        <Products />
      </Suspense>

      {/* Suspends independently */}
      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews />
      </Suspense>

      {/* Renders immediately */}
      <Footer />
    </>
  );
}

async function Products() {
  const products = await db.product.findMany();
  return <ProductList products={products} />;
}

async function Reviews() {
  const reviews = await db.review.findMany();
  return <ReviewList reviews={reviews} />;
}
```

## Component Boundaries

### Passing Server Components to Client Components

```typescript
// ❌ WRONG: Cannot import Server Component into Client Component
'use client';

import { ServerComponent } from './ServerComponent'; // Error!

function ClientComponent() {
  return <ServerComponent />; // Won't work
}
```

```typescript
// ✅ CORRECT: Pass Server Component as children/prop
// app/page.tsx (Server Component)
import { ClientWrapper } from './ClientWrapper';
import { ServerChild } from './ServerChild';

function Page() {
  return (
    <ClientWrapper>
      <ServerChild /> {/* Server Component passed as children */}
    </ClientWrapper>
  );
}

// ClientWrapper.tsx
'use client';

function ClientWrapper({ children }: { children: React.ReactNode }) {
  const [state, setState] = useState(false);
  
  return (
    <div>
      <button onClick={() => setState(!state)}>Toggle</button>
      {children} {/* Server Component rendered here */}
    </div>
  );
}
```

### Composition Patterns

```typescript
// Server Component
async function ProductPage({ id }: { id: string }) {
  const product = await db.product.findUnique({ where: { id } });

  return (
    <InteractiveLayout
      header={<ProductHeader product={product} />}
      sidebar={<ProductSpecs specs={product.specs} />}
      footer={<RelatedProducts category={product.category} />}
    >
      <ProductDetails product={product} />
    </InteractiveLayout>
  );
}

// Client Component
'use client';

function InteractiveLayout({
  header,
  sidebar,
  footer,
  children
}: {
  header: React.ReactNode;
  sidebar: React.ReactNode;
  footer: React.ReactNode;
  children: React.ReactNode;
}) {
  const [sidebarVisible, setSidebarVisible] = useState(true);

  return (
    <>
      {header}
      <div>
        {sidebarVisible && sidebar}
        <main>{children}</main>
      </div>
      {footer}
    </>
  );
}
```

### Context Boundaries

```typescript
// Server Component CANNOT consume context
// ❌ WRONG
'use server'; // Not a real directive, just for illustration

import { useContext } from 'react';
import { ThemeContext } from './theme';

function ServerComponent() {
  const theme = useContext(ThemeContext); // Error!
  return <div />;
}
```

```typescript
// ✅ CORRECT: Client Component consumes context
// app/layout.tsx (Server Component)
import { ThemeProvider } from './ThemeProvider';

function Layout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <ThemeProvider>
          {children} {/* Can contain Server Components */}
        </ThemeProvider>
      </body>
    </html>
  );
}

// ThemeProvider.tsx
'use client';

import { createContext, useState } from 'react';

export const ThemeContext = createContext<Theme | null>(null);

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<Theme>('light');

  return (
    <ThemeContext.Provider value={theme}>
      {children} {/* Server Components can be nested here */}
    </ThemeContext.Provider>
  );
}

// ThemedButton.tsx
'use client';

function ThemedButton() {
  const theme = useContext(ThemeContext); // Client Component consumes context
  return <button className={theme}>Click</button>;
}
```

## RSC Payload Format

The RSC payload is a specialized format that describes the component tree:

### Example Payload

```typescript
// Server Component
async function Page() {
  const user = await getUser();
  return (
    <div>
      <h1>Hello {user.name}</h1>
      <Counter initialCount={0} />
    </div>
  );
}

// Simplified RSC Payload:
{
  "type": "div",
  "props": null,
  "children": [
    {
      "type": "h1",
      "props": null,
      "children": ["Hello ", "John Doe"]
    },
    {
      "type": "@client/Counter",
      "props": { "initialCount": 0 },
      "children": null
    }
  ]
}
```

### Serialization Rules

```typescript
// ✅ Serializable props
<ClientComponent
  string="text"
  number={42}
  boolean={true}
  null={null}
  array={[1, 2, 3]}
  object={{ key: 'value' }}
  date={new Date().toISOString()} // Serialize dates as strings
/>

// ❌ Non-serializable props
<ClientComponent
  function={() => {}} // Functions cannot be serialized
  symbol={Symbol('key')} // Symbols cannot be serialized
  classInstance={new MyClass()} // Class instances cannot be serialized
  reactElement={<div />} // Elements need special handling
/>
```

### Server Actions in Payload

```typescript
// Server Component
async function Form() {
  async function submitForm(formData: FormData) {
    'use server'; // Server Action
    
    const name = formData.get('name');
    await db.user.create({ data: { name } });
  }

  return (
    <form action={submitForm}>
      <input name="name" />
      <button>Submit</button>
    </form>
  );
}

// RSC Payload includes server action reference:
{
  "type": "form",
  "props": {
    "action": "@server/submitForm:abc123" // Reference, not actual function
  },
  "children": [...]
}
```

## When to Use RSC

### Use Server Components When:

```typescript
// ✅ Data fetching
async function Products() {
  const products = await db.product.findMany();
  return <ProductGrid products={products} />;
}

// ✅ Accessing backend resources
async function FileViewer({ path }: { path: string }) {
  const content = await fs.readFile(path, 'utf-8');
  return <CodeBlock code={content} />;
}

// ✅ Using secret keys
async function Analytics() {
  const data = await fetch('https://api.service.com/analytics', {
    headers: { Authorization: `Bearer ${process.env.SECRET_KEY}` }
  });
  return <AnalyticsDashboard data={data} />;
}

// ✅ Large dependencies that don't need client interactivity
import { unified } from 'unified'; // Large library
import remarkParse from 'remark-parse';
import remarkHtml from 'remark-html';

async function MarkdownRenderer({ content }: { content: string }) {
  const html = await unified()
    .use(remarkParse)
    .use(remarkHtml)
    .process(content);
  
  return <div dangerouslySetInnerHTML={{ __html: String(html) }} />;
}
```

### Use Client Components When:

```typescript
// ✅ Interactivity and event handlers
'use client';

function Button() {
  return <button onClick={() => alert('Clicked!')}>Click</button>;
}

// ✅ State and lifecycle
'use client';

function Form() {
  const [value, setValue] = useState('');
  
  useEffect(() => {
    console.log('Value changed:', value);
  }, [value]);

  return <input value={value} onChange={e => setValue(e.target.value)} />;
}

// ✅ Browser APIs
'use client';

function GeolocationComponent() {
  const [location, setLocation] = useState<GeolocationPosition | null>(null);

  useEffect(() => {
    navigator.geolocation.getCurrentPosition(setLocation);
  }, []);

  return <div>{location && `${location.coords.latitude}, ${location.coords.longitude}`}</div>;
}

// ✅ Custom hooks
'use client';

function Counter() {
  const count = useCounter(0); // Custom hook
  return <div>{count}</div>;
}
```

### Decision Matrix

```typescript
interface ComponentDecision {
  serverComponent: {
    dataFetching: boolean;
    backendAccess: boolean;
    secretKeys: boolean;
    largeDependencies: boolean;
    noInteractivity: boolean;
    staticContent: boolean;
  };
  
  clientComponent: {
    useState: boolean;
    useEffect: boolean;
    eventHandlers: boolean;
    browserAPIs: boolean;
    customHooks: boolean;
    contextConsumer: boolean;
    animations: boolean;
    realTimeUpdates: boolean;
  };
}

// Examples of each:

// Server: Data-heavy, static
async function ProductCatalog() {
  const products = await db.product.findMany();
  return <ProductGrid products={products} />;
}

// Client: Interactive, dynamic
'use client';

function SearchBar() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  useEffect(() => {
    if (query) {
      searchProducts(query).then(setResults);
    }
  }, [query]);

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <SearchResults results={results} />
    </>
  );
}

// Hybrid: Server fetches, Client interacts
async function ProductPage({ id }: { id: string }) {
  const product = await db.product.findUnique({ where: { id } });

  return (
    <>
      <ProductInfo product={product} /> {/* Server Component */}
      <AddToCartButton product={product} /> {/* Client Component */}
      <ProductReviews productId={id} /> {/* Server Component */}
    </>
  );
}
```

## Common Mistakes

### 1. Making Everything a Client Component

```typescript
// ❌ BAD: Unnecessary "use client"
'use client';

async function Page() { // Can't be async!
  const data = await db.query(); // Can't access DB!
  return <div>{data}</div>;
}

// ✅ GOOD: Server Component by default
async function Page() {
  const data = await db.query();
  return <div>{data}</div>;
}
```

### 2. Importing Server Components into Client Components

```typescript
// ❌ BAD: Direct import
'use client';

import { ServerComponent } from './ServerComponent';

function ClientComponent() {
  return <ServerComponent />; // Error!
}

// ✅ GOOD: Pass as prop/children
// Parent (Server Component)
function Page() {
  return (
    <ClientWrapper>
      <ServerComponent />
    </ClientWrapper>
  );
}

// ClientWrapper.tsx
'use client';

function ClientWrapper({ children }: { children: React.ReactNode }) {
  return <div>{children}</div>;
}
```

### 3. Exposing Secrets to Client

```typescript
// ❌ BAD: Secret in Client Component
'use client';

function Analytics() {
  const apiKey = process.env.SECRET_API_KEY; // Exposed to client!
  
  fetch(`https://api.example.com/track?key=${apiKey}`); // Bad!
}

// ✅ GOOD: Secret in Server Component
async function Analytics() {
  const data = await fetch('https://api.example.com/track', {
    headers: { Authorization: `Bearer ${process.env.SECRET_API_KEY}` }
  });
  
  return <AnalyticsDashboard data={data} />;
}
```

## Best Practices

### 1. Default to Server Components

```typescript
// Start with Server Components by default
async function Page() {
  const data = await fetchData();
  
  return (
    <>
      <Header data={data} /> {/* Server Component */}
      <Content data={data} /> {/* Server Component */}
      <InteractiveWidget /> {/* Only this needs "use client" */}
      <Footer /> {/* Server Component */}
    </>
  );
}
```

### 2. Move "use client" Down the Tree

```typescript
// ❌ BAD: Client boundary too high
'use client';

function Layout({ children }) {
  const [sidebarOpen, setSidebarOpen] = useState(false);
  
  return (
    <>
      <Header /> {/* Forced to be client */}
      <Sidebar open={sidebarOpen} onToggle={setSidebarOpen} />
      <main>{children}</main>
      <Footer /> {/* Forced to be client */}
    </>
  );
}

// ✅ GOOD: Client boundary as low as possible
function Layout({ children }) {
  return (
    <>
      <Header /> {/* Server Component */}
      <SidebarWrapper /> {/* Only this is client */}
      <main>{children}</main>
      <Footer /> {/* Server Component */}
    </>
  );
}

// SidebarWrapper.tsx
'use client';

function SidebarWrapper() {
  const [open, setOpen] = useState(false);
  return <Sidebar open={open} onToggle={setOpen} />;
}
```

### 3. Use Composition for Flexibility

```typescript
// Good: Flexible, composable architecture
async function Page() {
  const user = await getUser();
  const posts = await getPosts(user.id);

  return (
    <InteractiveLayout
      header={<UserHeader user={user} />}
      sidebar={<Navigation />}
    >
      <PostList posts={posts} />
    </InteractiveLayout>
  );
}

'use client';

function InteractiveLayout({
  header,
  sidebar,
  children
}: {
  header: React.ReactNode;
  sidebar: React.ReactNode;
  children: React.ReactNode;
}) {
  const [sidebarVisible, setSidebarVisible] = useState(true);

  return (
    <>
      {header}
      <div className="layout">
        {sidebarVisible && <aside>{sidebar}</aside>}
        <main>{children}</main>
      </div>
    </>
  );
}
```

## Interview Questions

### Q1: What are React Server Components and how do they differ from Server-Side Rendering?

**Answer:** React Server Components (RSC) run only on the server and send a serialized representation to the client. Unlike SSR where components render to HTML on the server and then hydrate on the client, Server Components never re-render on the client and send zero JavaScript. SSR is about when/where initial HTML is generated; RSC is about which components run where.

### Q2: When should you use Server Components vs Client Components?

**Answer:** Use Server Components for data fetching, accessing backend resources, using secret keys, and static content. Use Client Components for interactivity (state, effects, event handlers), browser APIs, and anything requiring hydration. Default to Server Components and only add "use client" where interactivity is needed.

### Q3: Can you import a Server Component into a Client Component?

**Answer:** No, you cannot directly import a Server Component into a Client Component file. However, you can pass Server Components to Client Components as props (children or named props). This allows composition while maintaining the client/server boundary.

### Q4: How do Server Components access data without waterfalls?

**Answer:** Server Components can use parallel data fetching with Promise.all, or sequential fetching when dependencies exist. They can also use Suspense boundaries to stream content independently, allowing parts of the page to render while others are still loading.

### Q5: What is the RSC payload and what can be serialized?

**Answer:** The RSC payload is a specialized format that describes the component tree sent from server to client. It can serialize primitives, arrays, objects, and references to Server Actions, but cannot serialize functions, symbols, class instances, or most non-plain objects. Data must be JSON-serializable.

## Key Takeaways

1. **Server Components are the default** - In App Router, components are Server Components unless marked with "use client"
2. **Zero JavaScript for Server Components** - They send only serialized output, not code
3. **Can be async** - Server Components can use await at the top level
4. **Direct backend access** - Can access databases, filesystems, and use secret keys
5. **Client boundary is one-way** - Server → Client (via props), never Client → Server (via import)
6. **Composition over inheritance** - Pass Server Components as children/props to Client Components
7. **Move "use client" down** - Keep client boundaries as low in the tree as possible
8. **Not a replacement for SSR** - RSC and SSR work together; RSC runs during SSR

## Resources

### Official Documentation
- [Server Components RFC](https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md)
- [Next.js App Router Documentation](https://nextjs.org/docs/app)
- [React Server Components in Next.js](https://nextjs.org/docs/app/building-your-application/rendering/server-components)

### Articles and Talks
- "Introducing Zero-Bundle-Size React Server Components" - React Team
- "Server Components: The Future of React" - Dan Abramov
- "Making Sense of React Server Components" - Josh Comeau

### Examples
- [Next.js RSC Examples](https://github.com/vercel/next.js/tree/canary/examples)
- [Server Components Demo](https://github.com/reactjs/server-components-demo)

---

*Last Updated: 2026-05 - React 19 current*
