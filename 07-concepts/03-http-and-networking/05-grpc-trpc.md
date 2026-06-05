# gRPC and tRPC

## The Idea

**In plain English:** gRPC and tRPC are two ways for different parts of a software system to call functions on each other over a network, as if they were in the same program. gRPC is a high-speed communication method used between separate services (even written in different languages), while tRPC lets the front-end and back-end of a TypeScript website share types so mistakes are caught before the code even runs.

**Real-world analogy:** Imagine a busy restaurant where the waiter (front-end) needs to get food from the kitchen (back-end). With gRPC, the waiter and kitchen use a strict, laminated order form (the `.proto` schema) — every field is numbered and predefined, orders travel as compact shorthand codes (protobuf), and the system works even if the waiter speaks English and the kitchen speaks Spanish. With tRPC, the waiter and kitchen work in the same small diner where everyone speaks the same language, so the waiter can just shout the order and automatically knows exactly what will come back, no form needed.

- The laminated order form = the `.proto` schema (the shared contract)
- The compact shorthand codes = protobuf binary encoding
- The waiter shouting in the same language = TypeScript type sharing between client and server
- The kitchen = the server (back-end)
- The waiter = the client (front-end)

---

## Overview

REST is not the only way to build APIs. gRPC uses Protocol Buffers and HTTP/2 to deliver high-performance, strongly-typed, bi-directional streaming RPC. tRPC takes a different approach — it skips the network protocol entirely for full-stack TypeScript monorepos, sharing types between server and client with zero code generation. This guide covers when each fits, how they differ from REST and GraphQL, and practical implementation patterns.

## gRPC Fundamentals

```
gRPC Architecture:

  Client                    Server
  ┌─────────────┐           ┌─────────────┐
  │  Stub (gen) │──HTTP/2──▶│  Handler    │
  │             │◀──────────│  (gen)      │
  └─────────────┘           └─────────────┘
       ▲                         ▲
       │                         │
  protobuf encode           protobuf decode
       │                         │
  .proto schema ────────────────▶│
  (shared contract)
```

gRPC defines services and messages in `.proto` files. Code generators produce client stubs and server interfaces in any supported language. The binary protobuf encoding is 3-10x smaller and faster to parse than JSON.

## Protocol Buffers (protobuf)

```protobuf
// user.proto
syntax = "proto3";

package users.v1;

option go_package = "github.com/example/users/v1";

// Message definitions
message User {
  string id = 1;
  string email = 2;
  string display_name = 3;
  int64 created_at = 4;        // Unix timestamp
  UserRole role = 5;
  repeated string tag_ids = 6; // repeated = array
}

enum UserRole {
  USER_ROLE_UNSPECIFIED = 0;  // proto3: first value must be 0
  USER_ROLE_VIEWER = 1;
  USER_ROLE_EDITOR = 2;
  USER_ROLE_ADMIN = 3;
}

message GetUserRequest {
  string user_id = 1;
}

message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;  // cursor-based pagination
  string filter = 3;      // optional filter expression
}

message ListUsersResponse {
  repeated User users = 1;
  string next_page_token = 2;
}

// Service definition
service UserService {
  // Unary RPC — classic request/response
  rpc GetUser(GetUserRequest) returns (User);

  // Server streaming — server sends multiple responses
  rpc ListUsers(ListUsersRequest) returns (stream ListUsersResponse);

  // Client streaming — client sends multiple requests
  rpc BulkCreateUsers(stream User) returns (BulkCreateUsersResponse);

  // Bidirectional streaming
  rpc SyncUsers(stream SyncRequest) returns (stream SyncResponse);
}
```

## Four RPC Types

```typescript
// Generated TypeScript client (using @grpc/grpc-js + ts-proto or connect-es)

// 1. Unary — like a regular function call
const user = await userClient.getUser({ userId: 'u1' });
console.log(user.displayName);

// 2. Server streaming — server pushes a stream of responses
const stream = userClient.listUsers({ pageSize: 100 });
for await (const response of stream) {
  console.log(`Received ${response.users.length} users`);
}

// 3. Client streaming — client pushes multiple requests, server responds once
const call = userClient.bulkCreateUsers();
for (const user of usersToCreate) {
  call.write(user);
}
const result = await call.closeAndReceive();
console.log(`Created ${result.count} users`);

// 4. Bidirectional streaming — both sides stream simultaneously
const channel = userClient.syncUsers();
channel.write({ action: 'SUBSCRIBE', filter: 'role=admin' });
for await (const event of channel) {
  if (event.type === 'USER_UPDATED') {
    updateLocalCache(event.user);
  }
}
```

## gRPC in Node.js / TypeScript

```typescript
// server.ts — using @connectrpc/connect (modern gRPC-compatible)
import { ConnectRouter } from '@connectrpc/connect';
import { fastifyConnectPlugin } from '@connectrpc/connect-fastify';
import Fastify from 'fastify';
import { UserService } from './gen/user_connect.js';
import { User, GetUserRequest } from './gen/user_pb.js';

const routes = (router: ConnectRouter) =>
  router.service(UserService, {
    async getUser(req: GetUserRequest): Promise<User> {
      const user = await db.users.findById(req.userId);
      if (!user) throw new ConnectError('User not found', Code.NotFound);
      return new User({
        id: user.id,
        email: user.email,
        displayName: user.name,
        createdAt: BigInt(user.createdAt.getTime()),
        role: toProtoRole(user.role),
      });
    },

    async *listUsers(req: ListUsersRequest): AsyncIterable<ListUsersResponse> {
      let cursor = req.pageToken || undefined;
      while (true) {
        const page = await db.users.paginate({ cursor, limit: req.pageSize });
        yield new ListUsersResponse({
          users: page.items.map(toProtoUser),
          nextPageToken: page.nextCursor ?? '',
        });
        if (!page.nextCursor) break;
        cursor = page.nextCursor;
      }
    },
  });

const server = Fastify();
await server.register(fastifyConnectPlugin, { routes });
await server.listen({ port: 8080 });
```

```typescript
// client.ts
import { createClient } from '@connectrpc/connect';
import { createConnectTransport } from '@connectrpc/connect-web';
import { UserService } from './gen/user_connect.js';

const transport = createConnectTransport({
  baseUrl: 'https://api.example.com',
});

const client = createClient(UserService, transport);

// Type-safe call — autocomplete on request fields and response
const user = await client.getUser({ userId: 'u1' });
//     ^? User  — fully typed from .proto

// Stream
for await (const response of client.listUsers({ pageSize: 50 })) {
  response.users.forEach(displayUser);
}
```

## tRPC: End-to-End Type Safety Without Code Generation

tRPC doesn't use protobuf or HTTP/2. It works by sharing TypeScript types directly between server router definitions and client calls — the type flows at build time through module resolution, not at runtime through schemas.

```
tRPC type flow:

  Server router definition
  ┌───────────────────────────────┐
  │ const appRouter = router({    │
  │   getUser: query()            │
  │     .input(z.object({...}))   │
  │     .output(UserSchema)       │  ← TypeScript type inferred here
  │     .query(async ({input})    │
  │       => db.users.find(...)   │
  │   )                           │
  │ })                            │
  │                               │
  │ export type AppRouter =       │  ← exported type
  │   typeof appRouter            │
  └──────────────┬────────────────┘
                 │  type import (not runtime import)
                 ▼
  Client usage
  ┌───────────────────────────────┐
  │ import type { AppRouter }     │
  │   from '../server/router'     │
  │                               │
  │ const user =                  │
  │   await trpc.getUser.query({  │  ← fully typed, autocomplete works
  │     userId: 'u1'              │
  │   });                         │
  │ //  ^? User                   │  ← return type inferred
  └───────────────────────────────┘
```

## tRPC Setup (Next.js App Router)

```typescript
// server/trpc.ts — core initialization
import { initTRPC, TRPCError } from '@trpc/server';
import { ZodError } from 'zod';
import superjson from 'superjson';

// Context — what every procedure receives
interface Context {
  session: Session | null;
  db: PrismaClient;
}

const t = initTRPC.context<Context>().create({
  transformer: superjson,  // enables Date, Map, Set in responses
  errorFormatter({ shape, error }) {
    return {
      ...shape,
      data: {
        ...shape.data,
        zodError:
          error.cause instanceof ZodError ? error.cause.flatten() : null,
      },
    };
  },
});

// Reusable middleware
const enforceAuth = t.middleware(({ ctx, next }) => {
  if (!ctx.session?.user) {
    throw new TRPCError({ code: 'UNAUTHORIZED' });
  }
  return next({
    ctx: { ...ctx, user: ctx.session.user }, // narrows type
  });
});

export const router = t.router;
export const publicProcedure = t.procedure;
export const protectedProcedure = t.procedure.use(enforceAuth);
```

```typescript
// server/routers/users.ts
import { z } from 'zod';
import { router, publicProcedure, protectedProcedure } from '../trpc';

export const usersRouter = router({
  getById: publicProcedure
    .input(z.object({ id: z.string().cuid() }))
    .query(async ({ input, ctx }) => {
      const user = await ctx.db.user.findUniqueOrThrow({
        where: { id: input.id },
        select: { id: true, name: true, email: true, role: true },
      });
      return user; // TypeScript infers return type
    }),

  list: protectedProcedure
    .input(
      z.object({
        cursor: z.string().optional(),
        limit: z.number().min(1).max(100).default(20),
      })
    )
    .query(async ({ input, ctx }) => {
      const users = await ctx.db.user.findMany({
        take: input.limit + 1,
        cursor: input.cursor ? { id: input.cursor } : undefined,
        orderBy: { createdAt: 'desc' },
      });
      const nextCursor =
        users.length > input.limit ? users.pop()!.id : undefined;
      return { users, nextCursor };
    }),

  updateProfile: protectedProcedure
    .input(
      z.object({
        name: z.string().min(1).max(100),
        bio: z.string().max(500).optional(),
      })
    )
    .mutation(async ({ input, ctx }) => {
      return ctx.db.user.update({
        where: { id: ctx.user.id },
        data: input,
      });
    }),
});
```

```typescript
// server/root.ts
import { router } from './trpc';
import { usersRouter } from './routers/users';
import { postsRouter } from './routers/posts';

export const appRouter = router({
  users: usersRouter,
  posts: postsRouter,
});

export type AppRouter = typeof appRouter;
```

```typescript
// app/trpc-client.ts
import { createTRPCReact } from '@trpc/react-query';
import type { AppRouter } from '../server/root';

export const trpc = createTRPCReact<AppRouter>();

// Usage in React component — full type safety, no generated code
function UserProfile({ userId }: { userId: string }) {
  const { data, isLoading } = trpc.users.getById.useQuery({ id: userId });
  //            ^? { id: string; name: string; email: string; role: Role }

  const updateMutation = trpc.users.updateProfile.useMutation();

  const handleSave = (name: string) => {
    updateMutation.mutate({ name });
  };

  if (isLoading) return <Spinner />;
  return <div>{data?.name}</div>;
}
```

## gRPC vs tRPC vs REST vs GraphQL

```
Decision matrix:

┌─────────────────┬─────────────┬──────────────┬──────────────┬──────────────┐
│                 │    REST     │   GraphQL    │    gRPC      │    tRPC      │
├─────────────────┼─────────────┼──────────────┼──────────────┼──────────────┤
│ Type safety     │ Manual/OAS  │ Codegen      │ Codegen      │ Native TS    │
│ Transport       │ HTTP/1.1    │ HTTP/1.1     │ HTTP/2       │ HTTP/1.1     │
│ Payload format  │ JSON        │ JSON         │ protobuf     │ JSON/superjson│
│ Streaming       │ SSE/WS      │ Subscriptions│ Native       │ WS/SSE       │
│ Browser support │ Full        │ Full         │ Limited*     │ Full         │
│ Cross-language  │ Full        │ Full         │ Full         │ TS only      │
│ Schema def      │ Optional    │ Required     │ Required     │ Inferred     │
│ Code generation │ Optional    │ Yes          │ Yes          │ None         │
│ Bundle impact   │ None        │ Client lib   │ Large        │ Small        │
│ Best for        │ Public APIs │ Flex queries │ Microservices│ TS monorepos │
└─────────────────┴─────────────┴──────────────┴──────────────┴──────────────┘

* gRPC-Web requires a proxy (Envoy/Connect) for browsers
```

## When to Choose Each

### Choose gRPC when:
- High-throughput microservice-to-microservice communication
- Polyglot services (Go backend, Python ML service, Node API gateway)
- Bidirectional streaming is a first-class requirement (chat, live collaboration)
- Binary efficiency matters at scale (high-frequency financial data, telemetry)
- Strong contract enforcement between teams via `.proto` schema registry

### Choose tRPC when:
- Full-stack TypeScript monorepo (Next.js, Remix, SvelteKit)
- Rapid development where type safety without codegen is the priority
- Small to medium teams where the frontend and backend are owned together
- You want React Query integration out of the box
- Avoiding a separate schema language is a priority

### Choose GraphQL when:
- Clients need flexible, self-describing query composition
- Multiple clients (mobile, web, third-party) with different data needs
- Federation across multiple backend services under one graph

### Stick with REST when:
- Public API consumed by external developers
- Team expertise/preference is strong
- Existing REST infrastructure (caching layers, CDNs, API gateways)

## tRPC Subscriptions (Real-time)

```typescript
// Server — subscription procedure
import { observable } from '@trpc/server/observable';

export const notificationsRouter = router({
  onNew: protectedProcedure
    .input(z.object({ types: z.array(z.string()).optional() }))
    .subscription(({ input, ctx }) => {
      return observable<Notification>((emit) => {
        const handler = (notification: Notification) => {
          if (!input.types || input.types.includes(notification.type)) {
            emit.next(notification);
          }
        };
        // Subscribe to an event emitter / Redis pub-sub
        eventBus.on(`user:${ctx.user.id}:notification`, handler);
        return () => eventBus.off(`user:${ctx.user.id}:notification`, handler);
      });
    }),
});

// Client
function NotificationBell() {
  trpc.notifications.onNew.useSubscription(
    { types: ['MENTION', 'REPLY'] },
    {
      onData(notification) {
        toast(notification.title);
        queryClient.invalidateQueries(['notifications']);
      },
    }
  );
}
```

## gRPC Error Handling

```typescript
import { ConnectError, Code } from '@connectrpc/connect';

// Server — structured error codes
async getUser(req: GetUserRequest): Promise<User> {
  if (!req.userId) {
    throw new ConnectError('user_id is required', Code.InvalidArgument);
  }
  const user = await db.users.findById(req.userId);
  if (!user) {
    throw new ConnectError(`User ${req.userId} not found`, Code.NotFound);
  }
  if (!caller.canView(user)) {
    throw new ConnectError('Insufficient permissions', Code.PermissionDenied);
  }
  return toProtoUser(user);
}

// Client — typed error handling
try {
  const user = await client.getUser({ userId });
} catch (err) {
  if (err instanceof ConnectError) {
    switch (err.code) {
      case Code.NotFound:
        showNotFoundUI();
        break;
      case Code.PermissionDenied:
        redirectToLogin();
        break;
      default:
        reportError(err);
    }
  }
}
```

## Common Mistakes

### 1. Using tRPC Across Separate Deployments

```typescript
// ❌ tRPC across separate repos or services
// server lives at api.example.com (Node)
// client lives at app.example.com (React)
// You can't import server types into the client without sharing the package

// ✅ tRPC is designed for monorepos
// Use REST or gRPC for cross-service boundaries
// Use tRPC when server and client are in the same repo
```

### 2. Forgetting gRPC-Web Proxy for Browsers

```
# ❌ Direct gRPC from browser — HTTP/2 framing not supported
Browser → Port 50051 (raw gRPC) → FAILS

# ✅ Use Connect protocol or gRPC-Web proxy
Browser → Connect/gRPC-Web → Envoy/Connect proxy → gRPC service
```

### 3. Neglecting proto Backward Compatibility

```protobuf
// ❌ Changing field numbers breaks existing clients
message User {
  string id = 2;     // was 1 — now deserialization breaks for old clients
  string email = 1;  // was 2
}

// ✅ Never change field numbers; only add new fields
message User {
  string id = 1;          // never change this
  string email = 2;       // never change this
  string display_name = 3; // new field — old clients ignore it safely
}
```

### 4. Overusing tRPC Mutations for Reads

```typescript
// ❌ Using mutation for a read — loses caching benefits
const result = trpc.users.getList.useMutation();
result.mutate({ page: 1 });

// ✅ Queries are cached, background-refetched, and shareable
const result = trpc.users.list.useQuery({ cursor: undefined });
```

### 5. Not Using superjson with tRPC

```typescript
// ❌ Without superjson, Dates become strings
const user = await trpc.users.getById.query({ id: 'u1' });
user.createdAt instanceof Date; // false — it's a string!

// ✅ Add superjson transformer to preserve types
const t = initTRPC.create({ transformer: superjson });
user.createdAt instanceof Date; // true
```

## Interview Questions

### 1. What are the four gRPC communication patterns and when would you use each?

**Answer:** Unary (one request, one response) is the most common — equivalent to a REST call. Server streaming sends multiple responses to a single request — useful for large dataset paging, live price feeds, or log tailing. Client streaming sends multiple requests and receives one response — useful for bulk uploads or aggregation. Bidirectional streaming allows both sides to send and receive concurrently — used for real-time collaboration, chat, or continuous sync. Most CRUD APIs use unary; streaming makes sense when latency or throughput of a single request/response cycle is the bottleneck.

### 2. How does tRPC achieve type safety without code generation?

**Answer:** tRPC leverages TypeScript's structural type system and module imports. The server exports `typeof appRouter` as a type-only export. The client imports that type with `import type { AppRouter }` — which has zero runtime cost. tRPC's generics thread the router type through client factory functions so every procedure call is typed end to end: input validation via Zod shapes the input type, and the resolver's return type becomes the inferred output type. No `.proto` compilation, no GraphQL codegen step — just TypeScript's built-in type inference.

### 3. When would you choose gRPC over tRPC?

**Answer:** gRPC when: the services are in different languages (Go, Python, Rust, Node — gRPC has official support for all); you need bidirectional binary streaming at high throughput; you want a strict contract enforced via a schema registry (`.proto` files checked in and versioned); or browser access is not required. tRPC when: the entire stack is TypeScript, lives in a monorepo, and the team wants native type safety without a codegen build step. gRPC's protobuf binary format and HTTP/2 multiplexing give it a measurable performance edge for high-frequency service-to-service calls.

### 4. What is the Connect protocol and why does it matter for gRPC in browsers?

**Answer:** Browsers cannot speak raw gRPC (which relies on HTTP/2 trailers and binary framing that `fetch` doesn't expose). The Connect protocol (from Buf) is a gRPC-compatible protocol that works over standard HTTP/1.1 or HTTP/2 with JSON or protobuf bodies, making it natively fetch-compatible. It's backward compatible — a Connect server also accepts gRPC and gRPC-Web requests. This means you can use `@connectrpc/connect-web` in the browser without a proxy, while still interoperating with gRPC backends from server-side code.

### 5. What are the tradeoffs between protobuf and JSON for API payloads?

**Answer:** Protobuf is binary — 3-10x smaller, significantly faster to encode/decode, and strongly schema-typed. JSON is human-readable, universally parseable without tooling, and flexible (schema optional). Protobuf requires a `.proto` schema and codegen step to use; adding a field without following backward-compatibility rules (never renumber fields) is a breaking change. JSON allows ad-hoc clients without schemas. For internal high-throughput services where both sides control the schema, protobuf wins on performance. For public APIs, webhooks, or debugging-heavy workflows, JSON's legibility is often worth the overhead.
