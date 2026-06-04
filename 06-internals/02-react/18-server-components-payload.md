# Server Components Payload

## Overview

React Server Components (RSC) represent a fundamental shift in React's architecture, enabling components to run exclusively on the server. Understanding the RSC wire format, serialization protocol, client reference system, and payload structure reveals how React sends component trees from server to client while maintaining interactivity.

This guide explores RSC internals: the serialization format, how client and server components interact, streaming, and the journey from server-side React tree to client-side hydration.

## RSC Architecture

Server Components run only on the server, never in the browser:

```javascript
// Example 1: Server vs Client Components
// ServerComponent.server.js (or 'use server')
async function ServerComponent() {
  // Runs on server only
  const data = await db.query('SELECT * FROM users');
  const apiKey = process.env.API_KEY; // Safe - never sent to client
  
  return (
    <div>
      <h1>Users</h1>
      {data.map(user => (
        <UserCard key={user.id} user={user} />
      ))}
    </div>
  );
}

// ClientComponent.client.js (or 'use client')
'use client';

function ClientComponent({ initialCount }) {
  // Runs on client (and server for SSR)
  const [count, setCount] = useState(initialCount);
  
  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
}

// Server Component can import Client Component
import ClientComponent from './ClientComponent.client';

async function Page() {
  const data = await fetchData();
  
  return (
    <div>
      <h1>Page</h1>
      {/* Client Component boundary */}
      <ClientComponent initialCount={data.count} />
    </div>
  );
}
```

## RSC Wire Format

RSC uses a special serialization format to send component trees:

```javascript
// Example 2: RSC payload structure
// Server renders:
<div>
  <h1>Hello</h1>
  <ClientButton count={5} />
</div>

// Becomes payload:
M1:{"id":"./ClientButton.client.js","chunks":["client1"],"name":"default"}
J0:["$","div",null,{"children":[["$","h1",null,{"children":"Hello"}],["$","@1",null,{"count":5}]]}]

// Format explanation:
// M1: Module reference (Client Component)
// J0: JSON tree with special tokens
// "$": React element marker
// "@1": Reference to module M1
```

### Payload Components

```javascript
// Example 3: Detailed payload structure
// Module references (M lines):
M1:{"id":"./Button.client.js","chunks":["chunk1"],"name":"default"}
M2:{"id":"./Input.client.js","chunks":["chunk2"],"name":"Input"}

// Tree structure (J lines):
J0:["$","div",null,{
  "className":"container",
  "children":[
    ["$","@1",null,{"label":"Click"}],
    ["$","@2",null,{"value":"test"}]
  ]
}]

// Element format: ["$", type, key, props]
// "$" = React element
// type = "div" (host) or "@1" (client component reference)
// key = null or string
// props = object with properties

// Example 4: Serializing different types
// Host element:
<div className="test">Hello</div>
// → ["$","div",null,{"className":"test","children":"Hello"}]

// Client Component reference:
<ClientButton onClick={handler} />
// → ["$","@1",null,{"onClick":{"$$F":"f1"}}]
// $$F = function reference

// Fragment:
<>
  <div>One</div>
  <div>Two</div>
</>
// → ["$","$F",null,{"children":[["$","div",...],["$","div",...]]}]
// $F = Fragment symbol

// Suspense:
<Suspense fallback={<Loading />}>
  <Content />
</Suspense>
// → ["$","$S",null,{"fallback":["$","@2",null,{}],"children":["$","@3",null,{}]}]
// $S = Suspense symbol
```

## Serialization Protocol

React serializes server component output into a streamable format:

```javascript
// Example 5: Serialization process
async function renderServerComponent(Component, props) {
  // 1. Execute server component
  const element = await Component(props);
  
  // 2. Walk the element tree
  const serialized = await serializeElement(element);
  
  // 3. Return payload
  return serialized;
}

async function serializeElement(element) {
  if (typeof element === 'string' || typeof element === 'number') {
    return element;
  }
  
  if (Array.isArray(element)) {
    return Promise.all(element.map(serializeElement));
  }
  
  if (isReactElement(element)) {
    const { type, key, props } = element;
    
    if (typeof type === 'string') {
      // Host element (div, span, etc.)
      return [
        '$',
        type,
        key,
        await serializeProps(props)
      ];
    }
    
    if (isClientComponent(type)) {
      // Client Component - serialize as reference
      const moduleId = getModuleId(type);
      return [
        '$',
        `@${moduleId}`,
        key,
        await serializeProps(props)
      ];
    }
    
    if (isServerComponent(type)) {
      // Server Component - render and serialize result
      const result = await type(props);
      return serializeElement(result);
    }
  }
  
  throw new Error('Unsupported element type');
}

// Example 6: Prop serialization
async function serializeProps(props) {
  const serialized = {};
  
  for (const [key, value] of Object.entries(props)) {
    if (key === 'children') {
      serialized.children = await serializeElement(value);
    } else if (typeof value === 'function') {
      // Functions need special handling
      serialized[key] = { $$F: registerFunction(value) };
    } else if (React.isValidElement(value)) {
      serialized[key] = await serializeElement(value);
    } else if (isSerializable(value)) {
      serialized[key] = value;
    } else {
      throw new Error(`Cannot serialize prop: ${key}`);
    }
  }
  
  return serialized;
}
```

## Client References

Client Components are represented as module references:

```javascript
// Example 7: Client Component registration
// During build, bundler marks Client Components
// ClientButton.client.js
'use client';

export default function ClientButton({ label, onClick }) {
  return <button onClick={onClick}>{label}</button>;
}

// Becomes module reference:
{
  $$typeof: Symbol.for('react.client.reference'),
  $$id: './ClientButton.client.js',
  $$async: false
}

// Example 8: How client references work
// Server-side:
function ServerPage() {
  return (
    <div>
      <ClientButton label="Click" onClick={serverAction} />
    </div>
  );
}

// Serializes to:
M1:{"id":"./ClientButton.client.js","chunks":["client-bundle"],"name":"default"}
J0:["$","div",null,{"children":["$","@1",null,{"label":"Click","onClick":{"$$F":"f1"}}]}]

// Client-side:
// 1. Load client-bundle (contains ClientButton code)
// 2. Parse payload
// 3. Replace @1 with actual ClientButton component
// 4. Render interactive tree

// Example 9: Nested client components
'use client';

function ParentClient() {
  return (
    <div>
      <ChildClient />
    </div>
  );
}

function ChildClient() {
  return <button>Child</button>;
}

// IMPORTANT: ChildClient is part of ParentClient bundle
// Not a separate reference in payload
// Once you cross client boundary, everything is client code
```

## Server Actions

Functions can be serialized and called from the client:

```javascript
// Example 10: Server Action
// actions.js
'use server';

export async function saveData(formData) {
  // Runs on server only
  const data = Object.fromEntries(formData);
  await db.insert(data);
  revalidatePath('/dashboard');
}

// Example 11: Server Action serialization
// Client Component using Server Action
'use client';

import { saveData } from './actions';

function Form() {
  return (
    <form action={saveData}>
      <input name="title" />
      <button type="submit">Save</button>
    </form>
  );
}

// Payload includes action reference:
M1:{"id":"./actions.js","chunks":["server-actions"],"name":"saveData"}
J0:["$","form",null,{"action":{"$$F":"@1"},"children":[...]}]

// When form submits:
// 1. Client serializes form data
// 2. POSTs to server with action ID
// 3. Server executes saveData
// 4. Returns updated RSC payload
// 5. Client merges updates

// Example 12: Server Action with closure
async function ServerComponent() {
  const userId = await getCurrentUserId();
  
  // Server Action with closure
  async function updateUser(formData) {
    'use server';
    // Has access to userId from closure
    await db.update(userId, formData);
  }
  
  return (
    <form action={updateUser}>
      <input name="name" />
      <button>Update</button>
    </form>
  );
}

// Closure data encrypted and embedded in action reference
```

## Streaming RSC Payload

RSC payloads stream progressively:

```javascript
// Example 13: Streaming structure
// Initial chunk:
M1:{"id":"./ClientNav.client.js","chunks":["nav"],"name":"default"}
J0:["$","div",null,{"children":[["$","@1",null,{}],"$L1"]}]

// $L1 = placeholder for lazy chunk
// Suspense boundary

// Later chunk (when data ready):
J1:["$","div",null,{"children":"Loaded content"}]

// Client receives:
// 1. Initial chunk with placeholder
// 2. Shows Suspense fallback
// 3. Later chunk arrives
// 4. Replaces placeholder with content

// Example 14: Progressive rendering
async function Page() {
  return (
    <div>
      <Header />
      {/* This loads first */}
      
      <Suspense fallback={<Skeleton />}>
        <SlowContent />
        {/* This streams in later */}
      </Suspense>
    </div>
  );
}

async function SlowContent() {
  const data = await slowDataFetch(); // 2 seconds
  return <div>{data}</div>;
}

// Timeline:
// t=0ms: Send Header + Skeleton
// t=2000ms: Send SlowContent
// User sees Header immediately, content pops in later
```

## Deserialization on Client

Client parses and reconstructs the tree:

```javascript
// Example 15: Client-side deserialization (simplified)
function parseRSCPayload(payload) {
  const modules = new Map();
  const chunks = new Map();
  
  const lines = payload.split('\n');
  
  for (const line of lines) {
    if (line.startsWith('M')) {
      // Module reference
      const [id, data] = line.slice(1).split(':');
      modules.set(id, JSON.parse(data));
    } else if (line.startsWith('J')) {
      // JSON tree
      const [id, data] = line.slice(1).split(':');
      chunks.set(id, JSON.parse(data, reviver));
    }
  }
  
  return { modules, chunks };
}

function reviver(key, value) {
  if (Array.isArray(value) && value[0] === '$') {
    // React element
    const [, type, key, props] = value;
    
    if (type.startsWith('@')) {
      // Client Component reference
      const moduleId = type.slice(1);
      const module = loadClientModule(moduleId);
      return React.createElement(module, { ...props, key });
    }
    
    return React.createElement(type, { ...props, key });
  }
  
  if (typeof value === 'object' && value?.$$F) {
    // Function reference
    return createServerActionProxy(value.$$F);
  }
  
  return value;
}

// Example 16: Lazy chunk resolution
function resolveLazyChunk(chunkId) {
  if (chunks.has(chunkId)) {
    return chunks.get(chunkId);
  }
  
  // Return promise that resolves when chunk arrives
  return new Promise(resolve => {
    pendingChunks.set(chunkId, resolve);
  });
}

// When chunk arrives:
function handleChunk(chunkId, data) {
  chunks.set(chunkId, data);
  
  if (pendingChunks.has(chunkId)) {
    pendingChunks.get(chunkId)(data);
    pendingChunks.delete(chunkId);
  }
}
```

## Client Boundaries

Understanding what crosses the client boundary:

```javascript
// Example 17: What can't be serialized
// Server Component
async function ServerComponent() {
  const data = await fetchData();
  
  // CAN serialize:
  const validProps = {
    string: 'hello',
    number: 42,
    boolean: true,
    null: null,
    array: [1, 2, 3],
    object: { a: 1, b: 2 },
    date: new Date(), // Serialized as ISO string
    reactElement: <div>Hello</div>
  };
  
  // CANNOT serialize:
  const invalidProps = {
    function: () => {}, // Unless Server Action
    symbol: Symbol('test'),
    map: new Map(),
    set: new Set(),
    class: class MyClass {},
    domNode: document.getElementById('test'),
    error: new Error('test')
  };
  
  return <ClientComponent {...validProps} />;
}

// Example 18: Serialization boundaries
// ✅ GOOD: Serialize data, pass to client
async function GoodPattern() {
  const posts = await db.posts.findMany();
  
  return (
    <ClientList
      posts={posts.map(p => ({
        id: p.id,
        title: p.title,
        author: p.author.name
      }))}
    />
  );
}

// ❌ BAD: Can't pass database connection
async function BadPattern() {
  const db = connectToDatabase();
  
  return (
    <ClientComponent db={db} />
    // Error: Cannot serialize database connection
  );
}
```

## Optimizations

```javascript
// Example 19: Deduplication
// If same client component used multiple times:
<div>
  <ClientButton label="One" />
  <ClientButton label="Two" />
  <ClientButton label="Three" />
</div>

// Module reference sent once:
M1:{"id":"./ClientButton.client.js","chunks":["btn"],"name":"default"}
J0:["$","div",null,{"children":[
  ["$","@1",null,{"label":"One"}],
  ["$","@1",null,{"label":"Two"}],
  ["$","@1",null,{"label":"Three"}]
]}]

// @1 reused, not duplicated

// Example 20: Chunk splitting
// Large tree split into chunks:
J0:["$","div",null,{"children":["$L1","$L2","$L3"]}]
// Three lazy chunks loaded independently

// Enables:
// - Progressive rendering
// - Parallel loading
// - Better caching
```

## Common Misconceptions

1. **"Server Components never re-render"** - False. They re-render on navigation or when refetched, but on the server.

2. **"You can't use state in Server Components"** - True. Server Components are async functions that run once per request, not interactive components.

3. **"Client Components can't import Server Components"** - True. Once you cross to client, everything must be client code.

4. **"RSC makes your app faster automatically"** - Not always. It enables optimizations (zero client JS for some components, data fetching close to source) but requires proper architecture.

5. **"All components should be Server Components"** - False. Interactive features need Client Components. Use Server Components for data fetching and layout, Client Components for interactivity.

## Performance Implications

1. **Payload Size** - Serialized tree is compact but grows with complexity. Large trees increase initial load time.

2. **Streaming Benefits** - Suspense boundaries enable progressive rendering, improving perceived performance.

3. **Client Bundle Size** - Server Components have zero client JavaScript, dramatically reducing bundle size for non-interactive content.

4. **Refetching Cost** - Navigating triggers server-side re-render. Fast for simple components, slow for complex data fetching.

## Interview Questions

1. **Q: What is the RSC wire format?**
   A: RSC uses a line-based serialization format. Module references (M lines) define Client Components. JSON trees (J lines) describe the React tree structure using special tokens like "$" for elements and "@N" for client component references.

2. **Q: How do Server and Client Components interact?**
   A: Server Components run on the server and can render Client Components, passing serializable props. Client Components act as boundaries—everything inside must be client code. Server Components can't be imported by Client Components.

3. **Q: What can be passed as props to Client Components?**
   A: Serializable data: strings, numbers, booleans, arrays, plain objects, dates (as ISO strings), and React elements. Functions must be Server Actions. Cannot pass class instances, DOM nodes, symbols, or non-serializable objects.

4. **Q: How do Server Actions work?**
   A: Server Actions are functions marked with 'use server' that can be called from the client. They're serialized as references in the payload. When called, the client sends a POST with the action ID and arguments, the server executes it, and returns an updated RSC payload.

5. **Q: How does RSC streaming work?**
   A: The server sends the payload in chunks as components resolve. Suspense boundaries create natural split points. Initial chunks render immediately, later chunks stream in as data becomes available, progressively updating the UI.

6. **Q: Why can't Client Components import Server Components?**
   A: Client Components run in the browser and need to be bundled. Server Components run on the server with access to databases and file systems. There's no way to bundle server-only code for the browser. The boundary must go one direction: server → client.

7. **Q: How does RSC reduce bundle size?**
   A: Server Components don't get bundled for the client—they execute on the server and send rendered output. Only Client Components and their dependencies get bundled. For content-heavy pages with minimal interactivity, this dramatically reduces JavaScript sent to the browser.

8. **Q: What is the $$typeof symbol in RSC payload?**
   A: Like regular React elements, RSC uses $$typeof symbols to identify element types for security and deserialization. Client references use Symbol.for('react.client.reference'), preventing injection attacks and ensuring only valid components are rendered.

## Key Takeaways

1. RSC uses a line-based serialization format with module references and JSON trees
2. Server Components render on the server, Client Components render on both
3. Client Components act as boundaries—everything inside must be client code
4. Only serializable props can cross the server-client boundary
5. Server Actions enable server function calls from client code
6. Streaming enables progressive rendering through Suspense boundaries
7. Server Components have zero client JavaScript, reducing bundle size
8. The payload format is optimized for streaming and deduplication

## Resources

- [React Server Components RFC](https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md)
- [RSC from Scratch](https://github.com/reactwg/server-components/discussions/5)
- [Understanding React Server Components](https://www.joshwcomeau.com/react/server-components/)
- [Next.js App Router](https://nextjs.org/docs/app)
- [React Server Components Demo](https://github.com/reactjs/server-components-demo)
