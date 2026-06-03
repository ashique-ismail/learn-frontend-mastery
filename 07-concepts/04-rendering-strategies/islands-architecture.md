# Islands Architecture

## Overview

Islands Architecture is a rendering pattern that delivers primarily static HTML with small, isolated regions of interactivity ("islands" of dynamic content). Unlike traditional SPAs where the entire page is hydrated with JavaScript, Islands Architecture only hydrates the interactive components, dramatically reducing JavaScript bundle size and improving performance.

## How Islands Architecture Works

```
Traditional SPA:
┌────────────────────────────────────────────────────┐
│  Entire Page is Interactive (Heavy JavaScript)     │
├────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────┐ │
│  │  Header (static, but still hydrated)         │ │
│  ├──────────────────────────────────────────────┤ │
│  │  Navigation (static, but still hydrated)     │ │
│  ├──────────────────────────────────────────────┤ │
│  │  Content (static, but still hydrated)        │ │
│  ├──────────────────────────────────────────────┤ │
│  │  Interactive Widget (needs JS)               │ │
│  ├──────────────────────────────────────────────┤ │
│  │  Footer (static, but still hydrated)         │ │
│  └──────────────────────────────────────────────┘ │
│  All components load JS, even if not interactive  │
└────────────────────────────────────────────────────┘

Islands Architecture:
┌────────────────────────────────────────────────────┐
│  Mostly Static HTML (Minimal JavaScript)           │
├────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────┐ │
│  │  Header (pure HTML, no JS)                   │ │
│  ├──────────────────────────────────────────────┤ │
│  │  Navigation (pure HTML, no JS)               │ │
│  ├──────────────────────────────────────────────┤ │
│  │  Content (pure HTML, no JS)                  │ │
│  ├──────────────────────────────────────────────┤ │
│  │  🏝️ ISLAND: Interactive Widget (has JS)      │ │
│  ├──────────────────────────────────────────────┤ │
│  │  Footer (pure HTML, no JS)                   │ │
│  └──────────────────────────────────────────────┘ │
│  Only islands load JS and hydrate                 │
└────────────────────────────────────────────────────┘

Page Structure:
┌─────────────────────────────────────────────────┐
│  Static HTML                                     │
│  ┌─────────────────────────────────────────┐   │
│  │ Static Header                            │   │
│  └─────────────────────────────────────────┘   │
│                                                  │
│  ┌─────────────────────────────────────────┐   │
│  │ Static Content                           │   │
│  │  ┌───────────────────────────────────┐  │   │
│  │  │ 🏝️ Island 1: Search (React)      │  │   │
│  │  │ - Hydrates independently          │  │   │
│  │  │ - Loads own JS bundle             │  │   │
│  │  └───────────────────────────────────┘  │   │
│  │                                          │   │
│  │  More static content...                 │   │
│  │                                          │   │
│  │  ┌───────────────────────────────────┐  │   │
│  │  │ 🏝️ Island 2: Counter (Vue)       │  │   │
│  │  │ - Independent hydration           │  │   │
│  │  │ - Different framework OK          │  │   │
│  │  └───────────────────────────────────┘  │   │
│  └─────────────────────────────────────────┘   │
│                                                  │
│  ┌─────────────────────────────────────────┐   │
│  │ Static Footer                            │   │
│  └─────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

## Astro Islands Implementation

### Basic Island Component

```astro
---
// pages/index.astro
import Counter from '../components/Counter.jsx';
import SearchBox from '../components/SearchBox.vue';
import Newsletter from '../components/Newsletter.svelte';
---

<html>
  <head>
    <title>Islands Architecture Demo</title>
  </head>
  <body>
    <!-- Static HTML (no JS) -->
    <header>
      <h1>My Website</h1>
      <nav>
        <a href="/">Home</a>
        <a href="/about">About</a>
      </nav>
    </header>

    <main>
      <!-- Static content -->
      <h2>Welcome</h2>
      <p>This is static HTML content.</p>

      <!-- Island 1: Interactive counter (client:load) -->
      <Counter client:load />

      <!-- More static content -->
      <article>
        <h3>Blog Post</h3>
        <p>Static blog content here...</p>
      </article>

      <!-- Island 2: Search (client:idle) -->
      <SearchBox client:idle />

      <!-- Island 3: Newsletter (client:visible) -->
      <Newsletter client:visible />
    </main>

    <!-- Static footer -->
    <footer>
      <p>&copy; 2024 My Website</p>
    </footer>
  </body>
</html>
```

### Client Directives

```astro
---
// Different hydration strategies
import HeavyComponent from './HeavyComponent.jsx';
import LightComponent from './LightComponent.jsx';
import MediaPlayer from './MediaPlayer.jsx';
import ChatWidget from './ChatWidget.jsx';
---

<!-- No hydration: rendered to static HTML only -->
<HeavyComponent />

<!-- Hydrate immediately on page load -->
<LightComponent client:load />

<!-- Hydrate when browser is idle -->
<MediaPlayer client:idle />

<!-- Hydrate when component is visible -->
<ChatWidget client:visible />

<!-- Hydrate on media query match -->
<MobileMenu client:media="(max-width: 768px)" />

<!-- Never hydrate, but run once on page load -->
<Analytics client:only="react" />
```

### React Island Component

```tsx
// components/Counter.tsx
import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div className="counter-island">
      <h3>Interactive Counter</h3>
      <p>Current count: {count}</p>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
      <button onClick={() => setCount(count - 1)}>
        Decrement
      </button>
      <button onClick={() => setCount(0)}>
        Reset
      </button>
    </div>
  );
}
```

### Vue Island Component

```vue
<!-- components/SearchBox.vue -->
<template>
  <div class="search-island">
    <input
      v-model="query"
      @input="handleSearch"
      placeholder="Search..."
      type="text"
    />
    <ul v-if="results.length > 0">
      <li v-for="result in results" :key="result.id">
        <a :href="result.url">{{ result.title }}</a>
      </li>
    </ul>
    <p v-else-if="query">No results found</p>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue';

interface SearchResult {
  id: string;
  title: string;
  url: string;
}

const query = ref('');
const results = ref<SearchResult[]>([]);

const handleSearch = async () => {
  if (query.value.length < 3) {
    results.value = [];
    return;
  }

  try {
    const response = await fetch(`/api/search?q=${query.value}`);
    results.value = await response.json();
  } catch (error) {
    console.error('Search failed:', error);
    results.value = [];
  }
};
</script>
```

## Custom Islands Implementation

### Island Component System

```typescript
// lib/island-manager.ts
interface Island {
  id: string;
  component: string;
  props: Record<string, any>;
  strategy: 'load' | 'idle' | 'visible' | 'media';
  mediaQuery?: string;
  hydrated: boolean;
}

class IslandManager {
  private islands: Map<string, Island> = new Map();
  private observer: IntersectionObserver;
  private idleCallback: number | null = null;

  constructor() {
    // Observer for visible strategy
    this.observer = new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting) {
            const islandId = entry.target.id;
            this.hydrateIsland(islandId);
          }
        });
      },
      { rootMargin: '50px' }
    );

    // Setup idle callback for idle strategy
    if ('requestIdleCallback' in window) {
      this.idleCallback = requestIdleCallback(() => {
        this.hydrateIdleIslands();
      });
    }
  }

  registerIsland(island: Island): void {
    this.islands.set(island.id, island);

    switch (island.strategy) {
      case 'load':
        this.hydrateIsland(island.id);
        break;

      case 'idle':
        // Will be handled by idle callback
        break;

      case 'visible':
        const element = document.getElementById(island.id);
        if (element) {
          this.observer.observe(element);
        }
        break;

      case 'media':
        if (island.mediaQuery) {
          const mediaQuery = window.matchMedia(island.mediaQuery);
          if (mediaQuery.matches) {
            this.hydrateIsland(island.id);
          }
          mediaQuery.addEventListener('change', (e) => {
            if (e.matches) {
              this.hydrateIsland(island.id);
            }
          });
        }
        break;
    }
  }

  async hydrateIsland(id: string): Promise<void> {
    const island = this.islands.get(id);
    if (!island || island.hydrated) return;

    console.log(`Hydrating island: ${id}`);

    try {
      // Dynamic import of component
      const module = await import(`../islands/${island.component}.js`);
      const Component = module.default;

      // Get container element
      const container = document.getElementById(id);
      if (!container) {
        throw new Error(`Container not found: ${id}`);
      }

      // Hydrate based on framework
      await this.hydrateComponent(Component, container, island.props);

      island.hydrated = true;

      // Cleanup observer if used
      this.observer.unobserve(container);

      console.log(`Island hydrated: ${id}`);
    } catch (error) {
      console.error(`Failed to hydrate island ${id}:`, error);
    }
  }

  private async hydrateComponent(
    Component: any,
    container: HTMLElement,
    props: Record<string, any>
  ): Promise<void> {
    const framework = container.dataset.framework;

    switch (framework) {
      case 'react':
        const { hydrateRoot } = await import('react-dom/client');
        const React = await import('react');
        hydrateRoot(container, React.createElement(Component, props));
        break;

      case 'vue':
        const { createApp } = await import('vue');
        const app = createApp(Component, props);
        app.mount(container);
        break;

      case 'svelte':
        new Component({
          target: container,
          hydrate: true,
          props,
        });
        break;

      default:
        throw new Error(`Unknown framework: ${framework}`);
    }
  }

  private hydrateIdleIslands(): void {
    this.islands.forEach((island, id) => {
      if (island.strategy === 'idle' && !island.hydrated) {
        this.hydrateIsland(id);
      }
    });
  }

  getStats() {
    const total = this.islands.size;
    const hydrated = Array.from(this.islands.values()).filter(
      (i) => i.hydrated
    ).length;

    return {
      total,
      hydrated,
      pending: total - hydrated,
      hydratedPercentage: (hydrated / total) * 100,
    };
  }
}

export const islandManager = new IslandManager();

// Auto-initialize islands from DOM
if (typeof window !== 'undefined') {
  document.addEventListener('DOMContentLoaded', () => {
    const islandElements = document.querySelectorAll('[data-island]');
    
    islandElements.forEach((element) => {
      const id = element.id;
      const component = element.dataset.island!;
      const strategy = (element.dataset.strategy || 'load') as Island['strategy'];
      const props = JSON.parse(element.dataset.props || '{}');
      const mediaQuery = element.dataset.media;

      islandManager.registerIsland({
        id,
        component,
        props,
        strategy,
        mediaQuery,
        hydrated: false,
      });
    });
  });
}
```

### Server-Side Island Rendering

```typescript
// lib/island-renderer.ts
import { renderToString } from 'react-dom/server';
import React from 'react';

interface IslandRenderOptions {
  component: React.ComponentType<any>;
  props: Record<string, any>;
  strategy: 'load' | 'idle' | 'visible' | 'media';
  framework: 'react' | 'vue' | 'svelte';
  mediaQuery?: string;
}

export function renderIsland(options: IslandRenderOptions): string {
  const { component, props, strategy, framework, mediaQuery } = options;
  const islandId = generateIslandId();

  // Render component to static HTML
  let html = '';
  
  switch (framework) {
    case 'react':
      html = renderToString(React.createElement(component, props));
      break;
    // Handle other frameworks...
  }

  // Wrap in island container with metadata
  return `
    <div
      id="${islandId}"
      data-island="${component.name}"
      data-framework="${framework}"
      data-strategy="${strategy}"
      data-props='${JSON.stringify(props)}'
      ${mediaQuery ? `data-media="${mediaQuery}"` : ''}
    >
      ${html}
    </div>
  `;
}

function generateIslandId(): string {
  return `island-${Math.random().toString(36).substring(2, 11)}`;
}
```

## Next.js Islands Pattern

```tsx
// app/page.tsx
import dynamic from 'next/dynamic';

// Static component (no client JS)
function StaticHeader() {
  return (
    <header>
      <h1>My Website</h1>
      <nav>
        <a href="/">Home</a>
        <a href="/about">About</a>
      </nav>
    </header>
  );
}

// Dynamic islands (client-side JS only when needed)
const Counter = dynamic(() => import('./Counter'), {
  ssr: false, // Client-only
  loading: () => <div>Loading counter...</div>,
});

const SearchBox = dynamic(() => import('./SearchBox'), {
  ssr: false,
  loading: () => <div>Loading search...</div>,
});

export default function HomePage() {
  return (
    <div>
      {/* Static HTML */}
      <StaticHeader />

      <main>
        {/* Static content */}
        <section>
          <h2>Welcome</h2>
          <p>This is static content without JavaScript.</p>
        </section>

        {/* Interactive island */}
        <section>
          <h3>Try our counter</h3>
          <Counter />
        </section>

        {/* More static content */}
        <article>
          <h3>Latest News</h3>
          <p>Static news content...</p>
        </article>

        {/* Another interactive island */}
        <section>
          <h3>Search</h3>
          <SearchBox />
        </section>
      </main>

      {/* Static footer */}
      <footer>
        <p>&copy; 2024 My Website</p>
      </footer>
    </div>
  );
}
```

## Angular Lazy Islands

```typescript
// app.component.ts
import { Component, ViewChild, ViewContainerRef } from '@angular/core';

@Component({
  selector: 'app-root',
  template: `
    <header>
      <h1>My Website</h1>
      <nav>
        <a routerLink="/">Home</a>
        <a routerLink="/about">About</a>
      </nav>
    </header>

    <main>
      <!-- Static content -->
      <section>
        <h2>Welcome</h2>
        <p>This is static content.</p>
      </section>

      <!-- Island container -->
      <section>
        <h3>Interactive Widget</h3>
        <div #islandContainer></div>
        <button (click)="loadIsland()" *ngIf="!islandLoaded">
          Load Interactive Component
        </button>
      </section>

      <!-- More static content -->
      <article>
        <h3>Blog Post</h3>
        <p>Static blog content...</p>
      </article>
    </main>

    <footer>
      <p>&copy; 2024 My Website</p>
    </footer>
  `
})
export class AppComponent {
  @ViewChild('islandContainer', { read: ViewContainerRef })
  container!: ViewContainerRef;

  islandLoaded = false;

  async loadIsland(): Promise<void> {
    // Lazy load the module
    const { CounterComponent } = await import('./islands/counter.component');

    // Create component dynamically
    this.container.clear();
    this.container.createComponent(CounterComponent);

    this.islandLoaded = true;
  }

  ngAfterViewInit(): void {
    // Optionally auto-load based on visibility
    const observer = new IntersectionObserver((entries) => {
      if (entries[0].isIntersecting && !this.islandLoaded) {
        this.loadIsland();
      }
    });

    const element = this.container.element.nativeElement;
    observer.observe(element);
  }
}
```

## Partial Hydration Strategies

### Lazy Hydration on Interaction

```typescript
// components/LazyIsland.tsx
import { useState, useEffect, useRef } from 'react';
import type { ComponentType } from 'react';

interface LazyIslandProps {
  component: () => Promise<{ default: ComponentType<any> }>;
  props?: Record<string, any>;
  trigger?: 'click' | 'hover' | 'focus';
}

export function LazyIsland({
  component,
  props = {},
  trigger = 'click',
}: LazyIslandProps) {
  const [Component, setComponent] = useState<ComponentType<any> | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const containerRef = useRef<HTMLDivElement>(null);

  const loadComponent = async () => {
    if (Component || isLoading) return;

    setIsLoading(true);
    try {
      const module = await component();
      setComponent(() => module.default);
    } catch (error) {
      console.error('Failed to load component:', error);
    } finally {
      setIsLoading(false);
    }
  };

  useEffect(() => {
    const container = containerRef.current;
    if (!container) return;

    const handleTrigger = () => {
      loadComponent();
    };

    switch (trigger) {
      case 'click':
        container.addEventListener('click', handleTrigger, { once: true });
        return () => container.removeEventListener('click', handleTrigger);

      case 'hover':
        container.addEventListener('mouseenter', handleTrigger, { once: true });
        return () => container.removeEventListener('mouseenter', handleTrigger);

      case 'focus':
        container.addEventListener('focus', handleTrigger, { once: true });
        return () => container.removeEventListener('focus', handleTrigger);
    }
  }, [trigger]);

  return (
    <div ref={containerRef} className="lazy-island">
      {Component ? (
        <Component {...props} />
      ) : isLoading ? (
        <div>Loading...</div>
      ) : (
        <div className="lazy-island-placeholder">
          Click to load interactive content
        </div>
      )}
    </div>
  );
}

// Usage
function Page() {
  return (
    <LazyIsland
      component={() => import('./HeavyWidget')}
      props={{ userId: '123' }}
      trigger="click"
    />
  );
}
```

### Priority-Based Hydration

```typescript
// lib/priority-hydration.ts
interface HydrationTask {
  id: string;
  priority: 'high' | 'medium' | 'low';
  hydrate: () => Promise<void>;
  hydrated: boolean;
}

class PriorityHydrationScheduler {
  private tasks: HydrationTask[] = [];
  private isProcessing = false;

  addTask(task: HydrationTask): void {
    this.tasks.push(task);
    this.tasks.sort((a, b) => {
      const priorityMap = { high: 0, medium: 1, low: 2 };
      return priorityMap[a.priority] - priorityMap[b.priority];
    });

    if (!this.isProcessing) {
      this.processTasks();
    }
  }

  private async processTasks(): Promise<void> {
    this.isProcessing = true;

    while (this.tasks.length > 0) {
      const task = this.tasks.shift();
      if (!task || task.hydrated) continue;

      try {
        await task.hydrate();
        task.hydrated = true;
        console.log(`Hydrated island: ${task.id} (${task.priority})`);

        // Yield to browser between tasks
        await this.yieldToBrowser();
      } catch (error) {
        console.error(`Failed to hydrate ${task.id}:`, error);
      }
    }

    this.isProcessing = false;
  }

  private yieldToBrowser(): Promise<void> {
    return new Promise((resolve) => {
      if ('scheduler' in window && 'yield' in (window as any).scheduler) {
        (window as any).scheduler.yield().then(resolve);
      } else {
        setTimeout(resolve, 0);
      }
    });
  }
}

export const hydrationScheduler = new PriorityHydrationScheduler();
```

## Performance Monitoring

### Island Performance Metrics

```typescript
// lib/island-metrics.ts
interface IslandMetric {
  id: string;
  component: string;
  strategy: string;
  loadTime: number;
  hydrationTime: number;
  jsSize: number;
  timestamp: number;
}

class IslandPerformanceMonitor {
  private metrics: IslandMetric[] = [];

  startTracking(id: string): number {
    return performance.now();
  }

  recordHydration(
    id: string,
    component: string,
    strategy: string,
    startTime: number,
    jsSize: number
  ): void {
    const endTime = performance.now();
    const hydrationTime = endTime - startTime;

    const metric: IslandMetric = {
      id,
      component,
      strategy,
      loadTime: endTime,
      hydrationTime,
      jsSize,
      timestamp: Date.now(),
    };

    this.metrics.push(metric);

    // Send to analytics
    this.sendToAnalytics(metric);

    console.log(`Island ${id} hydrated in ${hydrationTime.toFixed(2)}ms`);
  }

  getStats() {
    const totalHydrationTime = this.metrics.reduce(
      (sum, m) => sum + m.hydrationTime,
      0
    );
    const totalJsSize = this.metrics.reduce((sum, m) => sum + m.jsSize, 0);
    const avgHydrationTime = totalHydrationTime / this.metrics.length;

    return {
      totalIslands: this.metrics.length,
      totalHydrationTime,
      avgHydrationTime,
      totalJsSize,
      byStrategy: this.groupByStrategy(),
    };
  }

  private groupByStrategy() {
    const grouped: Record<string, IslandMetric[]> = {};

    this.metrics.forEach((metric) => {
      if (!grouped[metric.strategy]) {
        grouped[metric.strategy] = [];
      }
      grouped[metric.strategy].push(metric);
    });

    return Object.entries(grouped).map(([strategy, metrics]) => ({
      strategy,
      count: metrics.length,
      avgHydrationTime:
        metrics.reduce((sum, m) => sum + m.hydrationTime, 0) / metrics.length,
      totalJsSize: metrics.reduce((sum, m) => sum + m.jsSize, 0),
    }));
  }

  private sendToAnalytics(metric: IslandMetric): void {
    if (typeof window !== 'undefined' && window.gtag) {
      window.gtag('event', 'island_hydration', {
        island_id: metric.id,
        component: metric.component,
        strategy: metric.strategy,
        hydration_time: metric.hydrationTime,
        js_size: metric.jsSize,
      });
    }
  }
}

export const islandMonitor = new IslandPerformanceMonitor();
```

## Common Mistakes

### 1. Over-Using Islands

```typescript
// BAD: Making everything an island
<Island component={Header} />
<Island component={Navigation} />
<Island component={Content} />
<Island component={Footer} />
// Too much overhead!

// GOOD: Only interactive parts
<Header /> {/* Static HTML */}
<Navigation /> {/* Static HTML */}
<Content>
  <Island component={InteractiveWidget} />
</Content>
<Footer /> {/* Static HTML */}
```

### 2. Wrong Hydration Strategy

```typescript
// BAD: Loading heavy component immediately
<HeavyChart client:load />

// GOOD: Load when visible
<HeavyChart client:visible />

// Or load when idle
<HeavyChart client:idle />
```

### 3. Island Communication Overhead

```typescript
// BAD: Islands trying to communicate
// Island 1
<Counter client:load />

// Island 2 (needs counter value)
<Display client:load /> // Can't access Counter state!

// GOOD: Single island for related functionality
<CounterWithDisplay client:load />
```

## Best Practices

### 1. Keep Islands Small and Focused

```typescript
// Each island should be self-contained
function ProductIsland({ productId }: { productId: string }) {
  return (
    <div className="product-island">
      <AddToCart productId={productId} />
      <WishlistButton productId={productId} />
      <ShareButton productId={productId} />
    </div>
  );
}
```

### 2. Use Appropriate Hydration Strategies

```typescript
// Immediate: Critical interactivity
<SearchBox client:load />

// Idle: Nice-to-have features
<NewsletterSignup client:idle />

// Visible: Below-the-fold content
<Comments client:visible />

// Media query: Responsive components
<MobileMenu client:media="(max-width: 768px)" />
```

### 3. Optimize Island Bundle Size

```typescript
// Use dynamic imports within islands
function HeavyIsland() {
  const loadModule = async () => {
    const { HeavyModule } = await import('./heavy-module');
    // Use module
  };

  return <button onClick={loadModule}>Load Feature</button>;
}
```

## When to Use Islands Architecture

### Ideal Use Cases

1. **Content-heavy websites** - Blogs, documentation, marketing sites
2. **E-commerce product pages** - Mostly static with few interactive elements
3. **Landing pages** - Minimal interactivity needed
4. **News sites** - Static articles with some interactive widgets
5. **Portfolio sites** - Showcase work with occasional interactivity

### Not Recommended For

1. **Highly interactive SPAs** - Too much interactivity overhead
2. **Real-time applications** - Dashboards, collaboration tools
3. **Complex state management** - Islands can't easily share state
4. **Frequent cross-component communication** - Islands are isolated

## Interview Questions

1. **What is Islands Architecture?**
   - Pattern with mostly static HTML and isolated interactive regions
   - Only islands load JavaScript and hydrate
   - Reduces JS bundle size dramatically

2. **How do islands differ from traditional SPAs?**
   - SPAs: entire page is interactive/hydrated
   - Islands: only specific components hydrate
   - Islands: much less JavaScript sent to client

3. **What are the benefits of Islands Architecture?**
   - Minimal JavaScript
   - Fast initial load
   - Better Core Web Vitals
   - Progressive enhancement

4. **How do islands communicate with each other?**
   - They typically don't (isolated by design)
   - Can use URL params, localStorage, or custom events
   - Better to combine related functionality in one island

5. **What hydration strategies are available?**
   - load: immediate hydration
   - idle: hydrate when browser idle
   - visible: hydrate when scrolled into view
   - media: hydrate on media query match

## Key Takeaways

1. Islands Architecture delivers minimal JavaScript
2. Only interactive components ("islands") hydrate
3. Most content remains static HTML
4. Choose appropriate hydration strategy per island
5. Keep islands small and self-contained
6. Islands can't easily share state
7. Great for content-heavy sites
8. Not suitable for highly interactive apps
9. Astro is the leading islands framework
10. Can implement in any framework with effort

## Resources

- [Islands Architecture](https://jasonformat.com/islands-architecture/)
- [Astro Islands](https://docs.astro.build/en/concepts/islands/)
- [Partial Hydration](https://www.patterns.dev/posts/progressive-hydration/)
- [The Islands Architecture](https://www.patterns.dev/posts/islands-architecture/)
- [Building with Islands](https://dev.to/this-is-learning/why-efficient-hydration-in-javascript-frameworks-is-so-challenging-1ca3)
