# Astro and React Islands

## Overview

Astro is a web framework designed for building content-rich websites with the Islands Architecture pattern. It ships zero JavaScript by default and allows you to hydrate interactive components (islands) only when needed. Astro supports React, Vue, Svelte, and other frameworks simultaneously.

## Table of Contents

1. [Islands Architecture Philosophy](#islands-architecture-philosophy)
2. [Client Directives](#client-directives)
3. [React Integration](#react-integration)
4. [Passing Props and Shared State](#passing-props-and-shared-state)
5. [Islands Patterns](#islands-patterns)
6. [Astro vs Next.js](#astro-vs-nextjs)
7. [Common Mistakes](#common-mistakes)
8. [Best Practices](#best-practices)
9. [Interview Questions](#interview-questions)

## Islands Architecture Philosophy

Islands Architecture delivers static HTML with isolated interactive "islands" that hydrate independently.

### Traditional SSR (React, Next.js)

```typescript
// Traditional approach: entire page hydrates
// page.tsx
export default function Page() {
  return (
    <div>
      <Header /> {/* Static, but hydrated */}
      <Navigation /> {/* Static, but hydrated */}
      <Article /> {/* Static, but hydrated */}
      <Sidebar /> {/* Interactive, needs hydration */}
      <Comments /> {/* Interactive, needs hydration */}
      <Footer /> {/* Static, but hydrated */}
    </div>
  );
}

// Result: ~200KB JavaScript shipped for the entire page
// All components hydrate even if most are static
```

### Islands Architecture (Astro)

```astro
---
// src/pages/index.astro
import Header from '../components/Header.astro';
import Navigation from '../components/Navigation.astro';
import Article from '../components/Article.astro';
import Sidebar from '../components/Sidebar.jsx';
import Comments from '../components/Comments.jsx';
import Footer from '../components/Footer.astro';
---

<html>
  <body>
    <!-- Static HTML, no JavaScript -->
    <Header />
    <Navigation />
    <Article />
    
    <!-- Island: hydrates on client -->
    <Sidebar client:load />
    
    <!-- Island: hydrates when visible -->
    <Comments client:visible />
    
    <!-- Static HTML, no JavaScript -->
    <Footer />
  </body>
</html>

<!-- Result: ~50KB JavaScript shipped (only islands)
     Only Sidebar and Comments hydrate
     Header, Navigation, Article, Footer are pure HTML -->
```

### Benefits of Islands

```typescript
// Performance comparison

// Traditional SPA/SSR:
// - Initial bundle: 200-500KB
// - Time to Interactive: 3-5s
// - Hydration cost: All components
// - Best for: Highly interactive apps

// Islands Architecture:
// - Initial bundle: 10-50KB (only islands)
// - Time to Interactive: <1s
// - Hydration cost: Only interactive components
// - Best for: Content-heavy sites with selective interactivity

// Example: Blog post page
// Traditional: 300KB (entire React app)
// Islands: 20KB (just comment form + like buttons)
// Result: 93% smaller bundle, 15x faster load
```

## Client Directives

Astro provides client directives to control when and how islands hydrate.

### client:load

```astro
---
// src/pages/dashboard.astro
import Counter from '../components/Counter.jsx';
---

<!-- Hydrates immediately on page load -->
<Counter client:load />
```

```jsx
// src/components/Counter.jsx
import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
    </div>
  );
}
```

### client:idle

```astro
---
// src/pages/home.astro
import ChatWidget from '../components/ChatWidget.jsx';
---

<!-- Hydrates when browser is idle (requestIdleCallback) -->
<!-- Perfect for low-priority widgets -->
<ChatWidget client:idle />
```

```jsx
// src/components/ChatWidget.jsx
import { useState } from 'react';

export default function ChatWidget() {
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <div className="chat-widget">
      <button onClick={() => setIsOpen(!isOpen)}>
        Chat with us
      </button>
      {isOpen && (
        <div className="chat-window">
          <p>How can we help?</p>
        </div>
      )}
    </div>
  );
}
```

### client:visible

```astro
---
// src/pages/article.astro
import Comments from '../components/Comments.jsx';
import RelatedArticles from '../components/RelatedArticles.jsx';
---

<article>
  <h1>My Article</h1>
  <p>Content...</p>
</article>

<!-- Hydrates when element enters viewport (Intersection Observer) -->
<Comments client:visible />
<RelatedArticles client:visible />
```

```jsx
// src/components/Comments.jsx
import { useState, useEffect } from 'react';

export default function Comments() {
  const [comments, setComments] = useState([]);
  
  useEffect(() => {
    // Only fetches when component becomes visible
    fetch('/api/comments')
      .then(res => res.json())
      .then(setComments);
  }, []);
  
  return (
    <div>
      <h2>Comments</h2>
      {comments.map(comment => (
        <div key={comment.id}>
          <p>{comment.text}</p>
        </div>
      ))}
    </div>
  );
}
```

### client:media

```astro
---
// src/pages/product.astro
import MobileMenu from '../components/MobileMenu.jsx';
import DesktopSidebar from '../components/DesktopSidebar.jsx';
---

<!-- Hydrates only on mobile devices -->
<MobileMenu client:media="(max-width: 768px)" />

<!-- Hydrates only on desktop -->
<DesktopSidebar client:media="(min-width: 769px)" />
```

```jsx
// src/components/MobileMenu.jsx
import { useState } from 'react';

export default function MobileMenu() {
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <nav className="mobile-menu">
      <button onClick={() => setIsOpen(!isOpen)}>
        Menu
      </button>
      {isOpen && (
        <ul>
          <li><a href="/about">About</a></li>
          <li><a href="/products">Products</a></li>
          <li><a href="/contact">Contact</a></li>
        </ul>
      )}
    </nav>
  );
}
```

### client:only

```astro
---
// src/pages/app.astro
import Chart from '../components/Chart.jsx';
---

<!-- Skip SSR, only render on client -->
<!-- Useful for browser-only libraries (e.g., Chart.js) -->
<Chart client:only="react" />
```

```jsx
// src/components/Chart.jsx
import { useEffect, useRef } from 'react';

export default function Chart({ data }) {
  const canvasRef = useRef(null);
  
  useEffect(() => {
    // Browser-only API
    import('chart.js').then(({ Chart }) => {
      new Chart(canvasRef.current, {
        type: 'bar',
        data: data,
      });
    });
  }, [data]);
  
  return <canvas ref={canvasRef} />;
}
```

### No Directive (Default)

```astro
---
// src/pages/index.astro
import StaticCard from '../components/StaticCard.jsx';
---

<!-- No directive = no hydration -->
<!-- Renders to static HTML only -->
<StaticCard title="Welcome" content="This is pure HTML" />
```

```jsx
// src/components/StaticCard.jsx
export default function StaticCard({ title, content }) {
  // No useState, no useEffect - just props
  // Rendered to HTML at build time
  return (
    <div className="card">
      <h2>{title}</h2>
      <p>{content}</p>
    </div>
  );
}
```

## React Integration

### Setting Up React with Astro

```bash
# Install Astro with React
npm create astro@latest
npx astro add react
```

```typescript
// astro.config.mjs
import { defineConfig } from 'astro/config';
import react from '@astrojs/react';

export default defineConfig({
  integrations: [react()],
});
```

### Using React Components

```astro
---
// src/pages/index.astro
import ReactButton from '../components/ReactButton.jsx';
import ReactCounter from '../components/ReactCounter.jsx';
---

<html>
  <body>
    <h1>Welcome to Astro</h1>
    
    <!-- Static React component (no JS) -->
    <ReactButton label="Learn More" />
    
    <!-- Interactive React component -->
    <ReactCounter client:load />
  </body>
</html>
```

```jsx
// src/components/ReactButton.jsx
export default function ReactButton({ label }) {
  // This renders to static HTML
  // No onClick will work without client directive
  return (
    <button className="btn">
      {label}
    </button>
  );
}

// src/components/ReactCounter.jsx
import { useState } from 'react';

export default function ReactCounter() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
    </div>
  );
}
```

### Mixing Astro and React

```astro
---
// src/pages/blog.astro
import Layout from '../layouts/Layout.astro';
import SearchBar from '../components/SearchBar.jsx';

const posts = await fetch('https://api.example.com/posts').then(r => r.json());
---

<Layout title="Blog">
  <!-- Astro component (static) -->
  <header>
    <h1>Blog</h1>
  </header>
  
  <!-- React component (interactive) -->
  <SearchBar client:load posts={posts} />
  
  <!-- Astro loop (static) -->
  <div class="posts">
    {posts.map(post => (
      <article>
        <h2>{post.title}</h2>
        <p>{post.excerpt}</p>
      </article>
    ))}
  </div>
</Layout>
```

```jsx
// src/components/SearchBar.jsx
import { useState } from 'react';

export default function SearchBar({ posts }) {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState(posts);
  
  const handleSearch = (e) => {
    const value = e.target.value;
    setQuery(value);
    
    const filtered = posts.filter(post =>
      post.title.toLowerCase().includes(value.toLowerCase())
    );
    setResults(filtered);
  };
  
  return (
    <div>
      <input
        type="search"
        value={query}
        onChange={handleSearch}
        placeholder="Search posts..."
      />
      
      <div className="results">
        {results.map(post => (
          <div key={post.id}>
            <h3>{post.title}</h3>
            <p>{post.excerpt}</p>
          </div>
        ))}
      </div>
    </div>
  );
}
```

## Passing Props and Shared State

### Passing Props to Islands

```astro
---
// src/pages/product.astro
import AddToCart from '../components/AddToCart.jsx';
import ReviewForm from '../components/ReviewForm.jsx';

const product = {
  id: '123',
  name: 'Awesome Product',
  price: 99.99,
};
---

<!-- Pass props to islands -->
<AddToCart 
  client:load 
  productId={product.id}
  price={product.price}
/>

<ReviewForm 
  client:visible 
  productId={product.id}
  productName={product.name}
/>
```

```jsx
// src/components/AddToCart.jsx
import { useState } from 'react';

export default function AddToCart({ productId, price }) {
  const [quantity, setQuantity] = useState(1);
  const [isAdding, setIsAdding] = useState(false);
  
  const addToCart = async () => {
    setIsAdding(true);
    await fetch('/api/cart', {
      method: 'POST',
      body: JSON.stringify({ productId, quantity }),
    });
    setIsAdding(false);
  };
  
  return (
    <div>
      <p>Price: ${price * quantity}</p>
      <input
        type="number"
        value={quantity}
        onChange={(e) => setQuantity(parseInt(e.target.value))}
        min="1"
      />
      <button onClick={addToCart} disabled={isAdding}>
        {isAdding ? 'Adding...' : 'Add to Cart'}
      </button>
    </div>
  );
}
```

### Sharing State Between Islands (Nano Stores)

```bash
npm install nanostores @nanostores/react
```

```typescript
// src/stores/cartStore.ts
import { atom } from 'nanostores';

export const cartItems = atom<Array<{ id: string; quantity: number }>>([]);

export function addToCart(productId: string, quantity: number) {
  const current = cartItems.get();
  const existing = current.find(item => item.id === productId);
  
  if (existing) {
    cartItems.set(
      current.map(item =>
        item.id === productId
          ? { ...item, quantity: item.quantity + quantity }
          : item
      )
    );
  } else {
    cartItems.set([...current, { id: productId, quantity }]);
  }
}

export function removeFromCart(productId: string) {
  cartItems.set(cartItems.get().filter(item => item.id !== productId));
}
```

```astro
---
// src/pages/shop.astro
import AddToCartButton from '../components/AddToCartButton.jsx';
import CartSummary from '../components/CartSummary.jsx';
---

<html>
  <body>
    <!-- Multiple islands sharing state -->
    <header>
      <CartSummary client:load />
    </header>
    
    <main>
      <div class="product">
        <h2>Product A</h2>
        <AddToCartButton client:load productId="a" />
      </div>
      
      <div class="product">
        <h2>Product B</h2>
        <AddToCartButton client:load productId="b" />
      </div>
    </main>
  </body>
</html>
```

```jsx
// src/components/AddToCartButton.jsx
import { addToCart } from '../stores/cartStore';

export default function AddToCartButton({ productId }) {
  return (
    <button onClick={() => addToCart(productId, 1)}>
      Add to Cart
    </button>
  );
}

// src/components/CartSummary.jsx
import { useStore } from '@nanostores/react';
import { cartItems } from '../stores/cartStore';

export default function CartSummary() {
  const items = useStore(cartItems);
  const total = items.reduce((sum, item) => sum + item.quantity, 0);
  
  return (
    <div className="cart-summary">
      <span>Cart: {total} items</span>
    </div>
  );
}
```

### Using Context API (Within Islands)

```jsx
// src/components/ShoppingCart.jsx
import { createContext, useContext, useState } from 'react';

const CartContext = createContext();

export function CartProvider({ children }) {
  const [items, setItems] = useState([]);
  
  const addItem = (product) => {
    setItems([...items, product]);
  };
  
  const removeItem = (id) => {
    setItems(items.filter(item => item.id !== id));
  };
  
  return (
    <CartContext.Provider value={{ items, addItem, removeItem }}>
      {children}
    </CartContext.Provider>
  );
}

export function useCart() {
  return useContext(CartContext);
}

// Usage in Astro
```

```astro
---
// src/pages/shop.astro
import { CartProvider } from '../components/ShoppingCart.jsx';
import ProductList from '../components/ProductList.jsx';
import CartSidebar from '../components/CartSidebar.jsx';
---

<!-- Wrap multiple components in a single island -->
<CartProvider client:load>
  <ProductList />
  <CartSidebar />
</CartProvider>
```

## Islands Patterns

### Progressive Enhancement Pattern

```astro
---
// src/pages/newsletter.astro
import NewsletterForm from '../components/NewsletterForm.jsx';
---

<!-- Form works without JavaScript (submits to server) -->
<!-- Enhanced with React when loaded -->
<NewsletterForm client:idle />
```

```jsx
// src/components/NewsletterForm.jsx
import { useState } from 'react';

export default function NewsletterForm() {
  const [email, setEmail] = useState('');
  const [status, setStatus] = useState('idle');
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    setStatus('loading');
    
    try {
      await fetch('/api/newsletter', {
        method: 'POST',
        body: JSON.stringify({ email }),
      });
      setStatus('success');
    } catch (error) {
      setStatus('error');
    }
  };
  
  return (
    <form onSubmit={handleSubmit} action="/api/newsletter" method="post">
      <input
        type="email"
        name="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        required
      />
      <button type="submit" disabled={status === 'loading'}>
        {status === 'loading' ? 'Subscribing...' : 'Subscribe'}
      </button>
      {status === 'success' && <p>Subscribed!</p>}
      {status === 'error' && <p>Failed to subscribe</p>}
    </form>
  );
}
```

### Lazy Loading Pattern

```astro
---
// src/pages/docs.astro
import TableOfContents from '../components/TableOfContents.jsx';
import CodePlayground from '../components/CodePlayground.jsx';
---

<article>
  <h1>Documentation</h1>
  
  <!-- Load immediately (navigation) -->
  <TableOfContents client:load />
  
  <div class="content">
    <!-- Content here -->
  </div>
  
  <!-- Load when visible (below fold) -->
  <CodePlayground client:visible />
</article>
```

### Conditional Loading Pattern

```astro
---
// src/pages/home.astro
import AdminPanel from '../components/AdminPanel.jsx';

const user = Astro.locals.user;
const isAdmin = user?.role === 'admin';
---

<html>
  <body>
    <h1>Welcome</h1>
    
    <!-- Only load island for admins -->
    {isAdmin && (
      <AdminPanel client:load />
    )}
  </body>
</html>
```

## Astro vs Next.js

### When to Use Astro

```typescript
// Astro is BEST for:
// - Content-heavy sites (blogs, documentation, marketing)
// - Sites with minimal interactivity
// - When you need multiple frameworks (React + Vue + Svelte)
// - When bundle size is critical
// - Static site generation with selective hydration

// Example: Blog
// Pages: 100
// Interactivity: Search bar, comment forms, like buttons
// Result with Astro: 10-20KB per page
// Result with Next.js: 100-200KB per page
```

```astro
---
// src/pages/blog/[slug].astro
import Layout from '../../layouts/Layout.astro';
import SearchBar from '../../components/SearchBar.jsx';
import LikeButton from '../../components/LikeButton.jsx';
import CommentForm from '../../components/CommentForm.jsx';

export async function getStaticPaths() {
  const posts = await fetch('https://api.example.com/posts').then(r => r.json());
  return posts.map(post => ({
    params: { slug: post.slug },
    props: { post },
  }));
}

const { post } = Astro.props;
---

<Layout title={post.title}>
  <!-- Static header -->
  <header>
    <h1>{post.title}</h1>
    <time>{post.date}</time>
  </header>
  
  <!-- Interactive search (loads idle) -->
  <SearchBar client:idle />
  
  <!-- Static content -->
  <article set:html={post.content} />
  
  <!-- Interactive like button (loads immediately) -->
  <LikeButton client:load postId={post.id} />
  
  <!-- Interactive comments (loads when visible) -->
  <CommentForm client:visible postId={post.id} />
</Layout>

<!-- Result: 15KB JavaScript (only islands) -->
```

### When to Use Next.js

```typescript
// Next.js is BEST for:
// - Highly interactive applications (dashboards, tools)
// - When you need server-side rendering (SSR)
// - Complex routing and data fetching
// - When everything needs to be interactive
// - API routes and full-stack features

// Example: Dashboard
// Pages: 20
// Interactivity: Everything is interactive
// Result with Next.js: 150-250KB per page (acceptable)
// Result with Astro: 150KB + overhead (no benefit)
```

```typescript
// app/dashboard/page.tsx (Next.js)
'use client';

import { useState, useEffect } from 'react';
import { Chart } from './Chart';
import { DataTable } from './DataTable';
import { FilterPanel } from './FilterPanel';
import { RealTimeUpdates } from './RealTimeUpdates';

export default function Dashboard() {
  const [data, setData] = useState([]);
  const [filters, setFilters] = useState({});
  
  useEffect(() => {
    // Everything is interactive
    fetchDashboardData(filters).then(setData);
  }, [filters]);
  
  return (
    <div>
      <FilterPanel onFilterChange={setFilters} />
      <Chart data={data} />
      <DataTable data={data} />
      <RealTimeUpdates />
    </div>
  );
}

// Result: 200KB JavaScript (everything interactive, that's fine)
```

### Feature Comparison

```typescript
// Feature comparison

// Astro:
// ✅ Zero JS by default
// ✅ Multi-framework support
// ✅ Smallest bundle sizes
// ✅ Best for content sites
// ✅ Islands architecture
// ❌ No SSR (unless using adapter)
// ❌ Limited routing features
// ❌ No built-in API routes

// Next.js:
// ✅ Full-stack framework
// ✅ SSR and ISR
// ✅ API routes
// ✅ Advanced routing
// ✅ Best for apps
// ❌ Ships more JavaScript
// ❌ Everything hydrates by default
// ❌ Single framework (React only)
```

## Common Mistakes

### 1. Over-Hydrating

```astro
<!-- WRONG: Hydrating everything -->
<Header client:load />
<Navigation client:load />
<Article client:load />
<Sidebar client:load />
<Footer client:load />

<!-- CORRECT: Only hydrate interactive components -->
<Header />
<Navigation />
<Article />
<Sidebar client:load /> <!-- Only this needs JS -->
<Footer />
```

### 2. Using Wrong Client Directive

```astro
<!-- WRONG: Loading everything immediately -->
<Comments client:load />
<RelatedPosts client:load />
<Newsletter client:load />

<!-- CORRECT: Use appropriate directives -->
<Comments client:visible /> <!-- Below fold -->
<RelatedPosts /> <!-- Static, no JS needed -->
<Newsletter client:idle /> <!-- Low priority -->
```

### 3. Not Using Astro Components

```astro
<!-- WRONG: Using React for static content -->
<ReactHeader client:load />
<ReactArticle client:load />

<!-- CORRECT: Use Astro components for static content -->
<Header />
<Article />
```

## Best Practices

### 1. Default to Static

```astro
<!-- Start with static Astro components -->
<Card title="Welcome" />

<!-- Add interactivity only when needed -->
<InteractiveCard client:visible title="Features" />
```

### 2. Choose Right Directive

```astro
<!-- client:load - Critical, above fold -->
<Navigation client:load />

<!-- client:idle - Non-critical, low priority -->
<ChatWidget client:idle />

<!-- client:visible - Below fold -->
<Comments client:visible />

<!-- client:media - Responsive -->
<MobileMenu client:media="(max-width: 768px)" />
```

### 3. Share State Efficiently

```typescript
// Use nano stores for cross-island state
import { atom } from 'nanostores';

export const userStore = atom({ name: '', email: '' });
```

## Interview Questions

### Q1: What is Islands Architecture?

**Answer:** Islands Architecture delivers static HTML with isolated interactive components (islands) that hydrate independently. Only interactive parts ship JavaScript, while static content remains HTML, resulting in faster loads and better performance.

### Q2: When should you use Astro vs Next.js?

**Answer:** Use Astro for content-heavy sites with selective interactivity (blogs, docs, marketing). Use Next.js for highly interactive applications (dashboards, tools, SPAs) where most of the page needs JavaScript.

### Q3: What's the difference between client:load and client:visible?

**Answer:** client:load hydrates immediately on page load. client:visible hydrates when the component enters the viewport using Intersection Observer. Use client:visible for below-the-fold content to improve initial load performance.

### Q4: How do you share state between islands?

**Answer:** Use nano stores (@nanostores/react) which provide a lightweight, framework-agnostic state management solution that works across multiple islands without requiring them to be wrapped in a provider.

### Q5: Can you use multiple frameworks in Astro?

**Answer:** Yes, Astro supports React, Vue, Svelte, Solid, Preact, and others simultaneously. You can mix frameworks in the same project, even on the same page, since each island is isolated.

## Key Takeaways

1. Islands Architecture ships less JavaScript by default
2. Only interactive components (islands) hydrate
3. client:load, client:idle, client:visible, client:media, client:only control hydration
4. Astro is best for content-heavy sites with selective interactivity
5. Next.js is best for highly interactive applications
6. Use nano stores to share state between islands
7. Default to static Astro components, add React when needed
8. Choose appropriate client directive based on component priority
9. Astro supports multiple frameworks simultaneously
10. Islands can use full React features (hooks, context) when hydrated

## Resources

- [Astro Documentation](https://docs.astro.build)
- [Islands Architecture](https://docs.astro.build/en/concepts/islands/)
- [Astro React Integration](https://docs.astro.build/en/guides/integrations-guide/react/)
- [Nano Stores](https://github.com/nanostores/nanostores)
- [Astro vs Next.js](https://docs.astro.build/en/guides/migrate-to-astro/from-nextjs/)
