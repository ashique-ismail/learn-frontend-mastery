# Astro with React Islands

## The Idea

**In plain English:** Astro is a tool for building websites that sends almost no JavaScript to the browser by default, making pages load super fast. The few parts of a page that actually need to be interactive (like a button that counts clicks) are called "islands" — small, self-contained zones that load their own JavaScript independently.

**Real-world analogy:** Imagine a newspaper that is mostly plain printed text, but has a few scratch-off lottery tickets stuck to certain sections. The newspaper itself needs no batteries or electronics — you just read it. Only the scratch-off tickets are "active" and need interaction.

- The printed newspaper = the static HTML page (no JavaScript needed)
- Each scratch-off ticket = a React island (interactive, loads its own JavaScript)
- Peeling off just one ticket = hydrating only that specific island, leaving the rest of the page untouched

---

## What Astro Is

Astro is a web framework for building content-heavy websites. It ships zero JavaScript by default — HTML is generated at build time. React (or Vue/Svelte/etc.) components can be included as "islands" — isolated interactive components that hydrate independently.

---

## Islands Architecture

```
┌─────────────────────────────────────────┐
│  Astro Page (static HTML, no JS)        │
│                                          │
│  ┌──────────┐  ┌───────────────────┐    │
│  │ Static   │  │ React Island      │    │
│  │ content  │  │ (loads JS only for│    │
│  │ (HTML)   │  │  this component)  │    │
│  └──────────┘  └───────────────────┘    │
│                                          │
│  ┌─────────────────┐                    │
│  │ Another Island  │                    │
│  │ (lazy loaded)   │                    │
│  └─────────────────┘                    │
└─────────────────────────────────────────┘
```

Static content ships as HTML. Islands ship with only the JavaScript needed for that component.

---

## React Integration

```bash
npx astro add react
# Installs @astrojs/react and configures astro.config.mjs
```

```js
// astro.config.mjs
import { defineConfig } from 'astro/config';
import react from '@astrojs/react';

export default defineConfig({
  integrations: [react()],
});
```

---

## Client Directives (Hydration Strategies)

```astro
---
// src/pages/index.astro
import StaticHeader from '../components/StaticHeader.astro';
import InteractiveCounter from '../components/Counter.tsx';  // React component
import HeavyChart from '../components/Chart.tsx';
---

<!-- No directive: renders to HTML, no JavaScript shipped -->
<StaticHeader />

<!-- client:load: hydrate immediately on page load -->
<InteractiveCounter client:load />

<!-- client:idle: hydrate when browser is idle (requestIdleCallback) -->
<HeavyChart client:idle />

<!-- client:visible: hydrate when entering viewport (IntersectionObserver) -->
<LazySection client:visible />

<!-- client:media: hydrate only when media query matches -->
<MobileMenu client:media="(max-width: 768px)" />

<!-- client:only: skip SSR entirely, render only on client -->
<ClientOnlyComponent client:only="react" />
```

---

## React Component in Astro

```tsx
// src/components/Counter.tsx — regular React component
import { useState } from 'react';

export default function Counter({ initialCount = 0 }: { initialCount?: number }) {
  const [count, setCount] = useState(initialCount);
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}
```

```astro
---
// src/pages/demo.astro
import Counter from '../components/Counter.tsx';

const initialCount = 5; // this is server-side data
---

<html>
  <body>
    <h1>Interactive Demo</h1>
    <!-- Pass props from Astro to React, hydrate on load -->
    <Counter initialCount={initialCount} client:load />
  </body>
</html>
```

---

## Content Collections

```ts
// src/content/config.ts
import { defineCollection, z } from 'astro:content';

const blog = defineCollection({
  type: 'content', // markdown/MDX files
  schema: z.object({
    title: z.string(),
    pubDate: z.date(),
    description: z.string(),
    tags: z.array(z.string()).default([]),
    image: z.string().optional(),
  }),
});

export const collections = { blog };
```

```astro
---
// src/pages/blog/[slug].astro
import { getCollection, getEntry } from 'astro:content';

const { slug } = Astro.params;
const entry = await getEntry('blog', slug);
const { Content } = await entry.render();
---

<h1>{entry.data.title}</h1>
<Content />  <!-- renders the markdown as HTML -->
```

---

## Rendering Modes

```js
// astro.config.mjs
export default defineConfig({
  output: 'static',    // SSG (default) — all pages at build time
  // output: 'server', // SSR — all pages on demand
  // output: 'hybrid', // mix — opt-in SSR per page
});
```

```astro
---
// Force a page to SSR in hybrid mode
export const prerender = false;

// Access request object (SSR only)
const user = await getUser(Astro.request.headers.get('cookie'));
---
```

---

## View Transitions

```astro
---
import { ViewTransitions } from 'astro:transitions';
---
<head>
  <ViewTransitions />  <!-- enables smooth page transitions -->
</head>

<!-- Custom transition on a specific element -->
<img
  src="/hero.jpg"
  transition:name="hero-image"  <!-- matches between pages -->
  transition:animate="fade"
/>
```

---

## When to Choose Astro

**Astro excels at:**
- Documentation sites (like Astro's own docs)
- Marketing sites, landing pages
- Blogs and content-heavy sites
- Sites where SEO and performance are critical

**Astro is not ideal for:**
- Highly interactive SPAs (use Next.js or Remix)
- Real-time applications
- Apps that are mostly authenticated/dynamic content

---

## Common Interview Questions

**Q: How does partial hydration differ from Next.js SSR?**
Next.js SSR renders the full page on the server then hydrates the entire page. Astro ships zero JS by default and only hydrates specific interactive islands. Result: much less JavaScript, faster TTI for content sites.

**Q: Can you mix React and Vue components in the same Astro page?**
Yes — each island is isolated and can use a different framework. Install both `@astrojs/react` and `@astrojs/vue` integrations.

**Q: What's `client:only` for?**
Components that use browser APIs (localStorage, window) and can't be server-rendered. With `client:only="react"`, Astro skips SSR entirely for that component.
