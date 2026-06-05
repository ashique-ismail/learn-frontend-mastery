# Streaming SSR Internals

## The Idea

**In plain English:** Streaming SSR is a way for a web server to send a webpage to your browser piece by piece, as each piece gets ready, instead of making you wait for the entire page to be built before sending anything. "SSR" means the server builds the HTML (the page's structure) before your browser receives it, rather than having your browser build it from scratch.

**Real-world analogy:** Think of a restaurant kitchen sending dishes to your table as each one finishes cooking, rather than holding everything until the full order is done. You get your soup right away, then your salad a minute later, then the main course when it's ready.

- The kitchen = the server rendering your React components
- Each dish = a section of the webpage (e.g., the header, the comments, the sidebar)
- The waiter delivering each dish as it's ready = the stream sending HTML chunks to the browser
- You eating each dish as it arrives = the browser displaying and making each section interactive without waiting for the rest

---

## Overview

Streaming Server-Side Rendering (SSR) allows React to send HTML to the client progressively as it's generated, rather than waiting for the entire page to render. Combined with Suspense boundaries and selective hydration, streaming SSR dramatically improves Time to First Byte (TTFB) and user-perceived performance.

This guide explores streaming rendering mechanics, Suspense integration, selective hydration, and how React coordinates server and client rendering.

## Traditional SSR vs Streaming SSR

Traditional SSR has a waterfall problem:

```javascript
// Example 1: Traditional SSR (React 17 and earlier)
// Server:
async function handler(req, res) {
  // 1. Fetch ALL data (blocks)
  const user = await fetchUser();
  const posts = await fetchPosts();
  const comments = await fetchComments();
  
  // 2. Render ENTIRE tree (blocks)
  const html = ReactDOMServer.renderToString(
    <App user={user} posts={posts} comments={comments} />
  );
  
  // 3. Send complete HTML
  res.send(`
    <!DOCTYPE html>
    <html>
      <body>
        <div id="root">${html}</div>
        <script src="/bundle.js"></script>
      </body>
    </html>
  `);
  
  // User sees content only after ALL data fetched and ALL HTML rendered
}

// Example 2: Streaming SSR (React 18)
// Server:
async function handler(req, res) {
  const { pipe } = ReactDOMServer.renderToPipeableStream(
    <App />,
    {
      bootstrapScripts: ['/bundle.js'],
      onShellReady() {
        // 1. Send shell immediately (fast)
        res.setHeader('Content-Type', 'text/html');
        pipe(res);
      }
    }
  );
  
  // 2. Stream content as it becomes ready
  // User sees shell immediately, content streams in
}

// Timeline comparison:
// Traditional: 0ms--------------------3000ms (show complete page)
// Streaming:   0ms--(shell)--500ms--(comments)--1000ms--(posts)
```

## renderToPipeableStream

The core streaming API:

```javascript
// Example 3: Basic streaming setup
import { renderToPipeableStream } from 'react-dom/server';

function App() {
  return (
    <html>
      <body>
        <nav>Navigation (fast)</nav>
        <Suspense fallback={<Spinner />}>
          <SlowContent />
        </Suspense>
      </body>
    </html>
  );
}

async function SlowContent() {
  const data = await fetchSlowData(); // 2 seconds
  return <div>{data}</div>;
}

// Server:
const { pipe } = renderToPipeableStream(<App />, {
  bootstrapScripts: ['/client.js'],
  
  onShellReady() {
    // Shell ready (fast parts)
    res.statusCode = 200;
    res.setHeader('Content-Type', 'text/html');
    pipe(res);
  },
  
  onAllReady() {
    // Everything ready (including Suspense content)
    // For SEO-critical pages, wait for this
  },
  
  onShellError(error) {
    // Shell failed to render
    res.statusCode = 500;
    res.send('Error');
  },
  
  onError(error) {
    // Component error
    console.error(error);
  }
});

// Output sent to client:
// 1. Initial HTML (immediately):
/*
<html>
  <body>
    <nav>Navigation</nav>
    <template id="B:0"></template>
    <div hidden id="S:0"><div>Spinner</div></div>
  </body>
</html>
<script src="/client.js"></script>
*/

// 2. Later HTML (when SlowContent ready):
/*
<script>
  $RC = function(b,c,e) {
    // Replace template with content
  };
  $RC("B:0", "S:1")
</script>
<div hidden id="S:1"><div>Loaded Content</div></div>
*/
```

## Suspense Boundaries as Stream Segments

Suspense boundaries control streaming granularity:

```javascript
// Example 4: Multiple Suspense boundaries
function Page() {
  return (
    <html>
      <body>
        <Header />
        
        <Suspense fallback={<SidebarSkeleton />}>
          <Sidebar />
        </Suspense>
        
        <Suspense fallback={<ContentSkeleton />}>
          <MainContent />
        </Suspense>
        
        <Suspense fallback={<CommentsSkeleton />}>
          <Comments />
        </Suspense>
        
        <Footer />
      </body>
    </html>
  );
}

// Streaming timeline:
// t=0ms:    Send <Header />, fallbacks, <Footer />
// t=100ms:  <Sidebar /> ready → stream replacement
// t=500ms:  <MainContent /> ready → stream replacement
// t=1200ms: <Comments /> ready → stream replacement

// Each boundary is independent
// Fast components don't wait for slow ones

// Example 5: Nested Suspense boundaries
function Dashboard() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <PageLayout>
        <Suspense fallback={<WidgetSkeleton />}>
          <Widget1 />
        </Suspense>
        
        <Suspense fallback={<WidgetSkeleton />}>
          <Widget2 />
        </Suspense>
      </PageLayout>
    </Suspense>
  );
}

// If PageLayout is slow:
// - Outer Suspense shows PageSkeleton
// - Inner Suspense boundaries don't matter yet
//
// If PageLayout is fast but widgets slow:
// - PageLayout renders
// - Shows WidgetSkeleton for each widget
// - Widgets stream in independently
```

## Selective Hydration

Streaming SSR enables selective hydration—hydrating parts of the page as they become interactive:

```javascript
// Example 6: Selective hydration process
function App() {
  return (
    <div>
      <Header />
      
      <Suspense fallback={<Loading />}>
        <Comments />
      </Suspense>
      
      <Suspense fallback={<Loading />}>
        <Sidebar />
      </Suspense>
    </div>
  );
}

// Server sends:
// 1. Header HTML (immediately)
// 2. Fallback HTML for Comments and Sidebar
// 3. Client starts hydrating Header
// 4. Comments HTML arrives → replaces fallback
// 5. Client hydrates Comments
// 6. Sidebar HTML arrives → replaces fallback
// 7. Client hydrates Sidebar

// KEY: Hydration happens progressively
// User can interact with Header while Comments still loading

// Example 7: Hydration prioritization
// Client implementation:
function hydrateRoot(container, children) {
  const root = createRoot(container, { hydrate: true });
  
  // Start hydration
  root.render(children);
  
  // Hydrate boundaries as they become ready
  scheduleMicrotask(() => {
    hydrateNextBoundary();
  });
}

function hydrateNextBoundary() {
  const boundary = findNextHydratableBoundary();
  
  if (boundary) {
    hydrateBoundary(boundary);
    scheduleMicrotask(hydrateNextBoundary);
  }
}

// User interaction prioritizes hydration
function onUserInteraction(element) {
  const boundary = findBoundaryForElement(element);
  
  if (boundary && !boundary.hydrated) {
    // Prioritize this boundary
    hydrateBoundary(boundary);
  }
}
```

## Stream Protocol

Understanding the HTML and script coordination:

```javascript
// Example 8: Stream format
// Initial chunk (shell):
<html>
<body>
  <div id="root">
    <h1>Page Title</h1>
    <!--$?--><template id="B:0"></template><!--/$-->
  </div>
  <script src="/client.js" async></script>
</body>
</html>

// Comments mark Suspense boundaries:
// <!--$?--> = Suspense boundary start
// <!--/$--> = Suspense boundary end
// <template id="B:0"> = Placeholder

// Later chunk (content ready):
<div hidden id="S:0">
  <div>Actual Content</div>
</div>
<script>
$RC("B:0","S:0")
</script>

// $RC function (sent in bootstrap):
function $RC(boundaryId, contentId) {
  // Find placeholder
  const template = document.getElementById(boundaryId);
  
  // Find content
  const content = document.getElementById(contentId);
  
  // Replace placeholder with content
  const parent = template.parentNode;
  content.hidden = false;
  parent.insertBefore(content, template);
  template.remove();
}

// Example 9: Multiple content chunks
// Can stream multiple pieces for one boundary:
<script>$RX("B:0", "S:0", 0)</script>
<div hidden id="S:0">Chunk 1</div>

<script>$RX("B:0", "S:1", 1)</script>
<div hidden id="S:1">Chunk 2</div>

<script>$RC("B:0")</script>
// Signals boundary complete
```

## Error Handling in Streams

```javascript
// Example 10: Error boundaries with streaming
function App() {
  return (
    <html>
      <body>
        <ErrorBoundary fallback={<Error />}>
          <Suspense fallback={<Loading />}>
            <ComponentThatMayError />
          </Suspense>
        </ErrorBoundary>
      </body>
    </html>
  );
}

// If ComponentThatMayError throws:
// 1. Server catches error in Suspense boundary
// 2. Streams error HTML:
<div hidden id="E:0">
  <div>Error occurred</div>
</div>
<script>
$RE("B:0", "E:0", {
  message: "Error message",
  stack: "..." // In dev only
})
</script>

// 3. Client shows error UI
// 4. ErrorBoundary on client catches during hydration

// Example 11: Shell errors vs boundary errors
const { pipe } = renderToPipeableStream(<App />, {
  onShellError(error) {
    // Shell failed - nothing sent yet
    res.statusCode = 500;
    res.send(`
      <html>
        <body>
          <h1>Server Error</h1>
          <p>${error.message}</p>
        </body>
      </html>
    `);
  },
  
  onError(error) {
    // Boundary error - shell already sent
    // Error handled by ErrorBoundary
    console.error('Boundary error:', error);
  }
});
```

## Streaming Strategies

```javascript
// Example 12: Wait for important content
const { pipe } = renderToPipeableStream(<App />, {
  onAllReady() {
    // Wait for ALL content (including Suspense)
    // Use for:
    // - SEO-critical pages
    // - Social media crawlers
    // - Print-to-PDF
    pipe(res);
  }
});

// Example 13: Stream immediately
const { pipe } = renderToPipeableStream(<App />, {
  onShellReady() {
    // Start streaming as soon as possible
    // Use for:
    // - Interactive apps
    // - Logged-in experiences
    // - Better perceived performance
    pipe(res);
  }
});

// Example 14: Hybrid approach
const { pipe } = renderToPipeableStream(<App />, {
  onShellReady() {
    if (isBot(req)) {
      // For bots, wait for all content
      return;
    }
    
    // For users, stream immediately
    pipe(res);
  },
  
  onAllReady() {
    // Bots get complete page
    pipe(res);
  }
});
```

## Data Fetching Patterns

```javascript
// Example 15: Server Component pattern
// Server Component (RSC)
async function Posts() {
  const posts = await fetchPosts();
  
  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}

function Page() {
  return (
    <Suspense fallback={<PostsSkeleton />}>
      <Posts />
    </Suspense>
  );
}

// Posts component suspends while fetching
// Suspense boundary allows streaming

// Example 16: Parallel data fetching
function Dashboard() {
  return (
    <>
      <Suspense fallback={<UserSkeleton />}>
        <UserWidget />
      </Suspense>
      
      <Suspense fallback={<StatsSkeleton />}>
        <StatsWidget />
      </Suspense>
      
      <Suspense fallback={<ActivitySkeleton />}>
        <ActivityWidget />
      </Suspense>
    </>
  );
}

// All three widgets fetch in parallel
// Stream in as they complete, in any order

// Example 17: Sequential data fetching (anti-pattern)
async function BadNesting() {
  return (
    <Suspense fallback={<Loading />}>
      <Level1>
        <Suspense fallback={<Loading />}>
          <Level2>
            <Suspense fallback={<Loading />}>
              <Level3 />
            </Suspense>
          </Level2>
        </Suspense>
      </Level1>
    </Suspense>
  );
}

// Creates waterfall:
// Level1 loads → then Level2 loads → then Level3 loads
// Each waits for previous

// Better: Flatten and parallelize
async function GoodParallel() {
  return (
    <>
      <Suspense fallback={<Loading1 />}>
        <Level1 />
      </Suspense>
      <Suspense fallback={<Loading2 />}>
        <Level2 />
      </Suspense>
      <Suspense fallback={<Loading3 />}>
        <Level3 />
      </Suspense>
    </>
  );
}

// All three load in parallel
```

## Client-Side Coordination

```javascript
// Example 18: Hydration queue
// Client maintains hydration queue
const hydrationQueue = [];
let isHydrating = false;

function scheduleHydration(boundary) {
  hydrationQueue.push(boundary);
  
  if (!isHydrating) {
    isHydrating = true;
    scheduleCallback(flushHydrationQueue);
  }
}

function flushHydrationQueue() {
  while (hydrationQueue.length > 0) {
    const boundary = hydrationQueue.shift();
    hydrateBoundary(boundary);
    
    // Yield to browser every few boundaries
    if (shouldYield()) {
      scheduleCallback(flushHydrationQueue);
      return;
    }
  }
  
  isHydrating = false;
}

// Example 19: User interaction priority
document.addEventListener('click', (e) => {
  const target = e.target;
  const boundary = findBoundaryForElement(target);
  
  if (boundary && !boundary.hydrated) {
    // Move to front of queue
    hydrationQueue.unshift(boundary);
    flushHydrationQueue();
  }
}, true); // Capture phase

// User clicks → immediately hydrate that section
// Other sections hydrate in background
```

## Performance Characteristics

```javascript
// Example 20: Measuring streaming performance
const { pipe } = renderToPipeableStream(<App />, {
  onShellReady() {
    console.log('Time to First Byte:', Date.now() - startTime);
    pipe(res);
  },
  
  onAllReady() {
    console.log('Time to Complete:', Date.now() - startTime);
  }
});

// Metrics:
// TTFB (Time to First Byte):
//   Traditional SSR: Wait for all data + render
//   Streaming SSR: Send shell immediately (much faster)
//
// FCP (First Contentful Paint):
//   Traditional SSR: Same as TTFB
//   Streaming SSR: Shell paints quickly
//
// TTI (Time to Interactive):
//   Traditional SSR: After full hydration
//   Streaming SSR: Progressive (parts interactive earlier)
```

## Common Misconceptions

1. **"Streaming SSR is always faster"** - False. For simple pages or when all data is fast, traditional SSR can be simpler. Streaming shines with slow data sources.

2. **"You need React Server Components for streaming"** - False. Streaming SSR works with regular React. RSC and streaming complement each other but are separate features.

3. **"All HTML must stream"** - False. You can use onAllReady to wait for complete HTML, useful for SEO-critical pages or bots.

4. **"Suspense is only for data fetching"** - False. Suspense creates stream boundaries and enables code splitting, lazy loading, and any async operation.

5. **"Selective hydration works automatically"** - Mostly true. React handles it, but you need proper Suspense boundaries and should avoid blocking the main thread during hydration.

## Performance Implications

1. **TTFB Improvement** - Streaming can reduce TTFB from seconds to milliseconds by sending the shell immediately.

2. **Progressive Enhancement** - Users see and interact with content as it arrives, improving perceived performance.

3. **Network Utilization** - Streaming uses the network connection more efficiently, sending data as it's ready rather than in one large chunk.

4. **Memory Usage** - Server memory usage is more consistent—no need to buffer entire HTML before sending.

## Interview Questions

1. **Q: How does streaming SSR differ from traditional SSR?**
   A: Traditional SSR waits for all data and renders the complete tree before sending HTML. Streaming SSR sends the shell immediately and streams content as it becomes ready, dramatically improving TTFB and user-perceived performance.

2. **Q: What role do Suspense boundaries play in streaming?**
   A: Suspense boundaries mark stream segments. React sends fallbacks immediately in the shell, then streams actual content when ready, replacing fallbacks with inline scripts. Each boundary is independent, allowing parallel rendering.

3. **Q: How does selective hydration work?**
   A: As streamed HTML chunks arrive, React schedules hydration for those sections. Hydration happens progressively rather than all at once. User interactions prioritize hydration—clicked sections hydrate first, keeping the app responsive.

4. **Q: When would you use onShellReady vs onAllReady?**
   A: onShellReady streams immediately for faster user experience, good for interactive apps. onAllReady waits for all content, important for SEO-critical pages and search engine crawlers that need complete HTML.

5. **Q: How does streaming handle errors?**
   A: Shell errors trigger onShellError before any HTML is sent. Boundary errors after shell is sent are caught by ErrorBoundary components and streamed as error UI. Clients receive error replacement scripts.

6. **Q: What is the streaming protocol between server and client?**
   A: The server sends HTML with template placeholders marked by comments. As content becomes ready, it sends hidden divs with content and inline scripts ($RC function calls) that replace placeholders, progressively building the page.

7. **Q: How does streaming affect hydration performance?**
   A: Streaming enables progressive hydration—parts of the page hydrate as HTML arrives rather than waiting for the entire page. This reduces time to interactive for above-the-fold content and allows user interaction with hydrated sections while others load.

8. **Q: Can you stream without Suspense?**
   A: Technically no—Suspense boundaries are required to mark where streaming can happen. Without them, React must wait for the entire tree. Suspense is the mechanism that enables async content and stream segmentation.

## Key Takeaways

1. Streaming SSR sends HTML progressively as it's generated, not all at once
2. Suspense boundaries mark independent stream segments
3. The shell sends immediately with fallbacks, content streams in later
4. Selective hydration hydrates sections progressively as HTML arrives
5. User interactions prioritize hydration of clicked sections
6. onShellReady for fast UX, onAllReady for SEO
7. Inline scripts coordinate content replacement on the client
8. Streaming dramatically improves TTFB and perceived performance

## Resources

- [React 18 Streaming SSR](https://react.dev/reference/react-dom/server/renderToPipeableStream)
- [Streaming Server Rendering with Suspense](https://github.com/reactwg/react-18/discussions/37)
- [New Suspense SSR Architecture](https://github.com/reactwg/react-18/discussions/130)
- [Next.js Streaming](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming)
- [React Server Rendering APIs](https://react.dev/reference/react-dom/server)
