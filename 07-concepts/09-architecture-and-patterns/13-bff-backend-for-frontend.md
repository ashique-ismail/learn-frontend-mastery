# Backend for Frontend (BFF)

## The Idea

**In plain English:** A Backend for Frontend (BFF) is a small server-side helper built specifically for one type of app — like a website or a mobile app — that gathers information from many different places and hands it back in exactly the shape that app needs. Think of a "backend" as a behind-the-scenes computer that fetches and organises data, and a "frontend" as the visible screen the user actually sees.

**Real-world analogy:** Imagine a hotel concierge desk. A guest walks up and says "I need a taxi, dinner reservation, and theatre tickets." The concierge makes three separate calls to a taxi company, a restaurant, and a theatre — and then comes back with one tidy summary for the guest instead of making the guest call all three places themselves.

- The guest = the frontend app (web or mobile)
- The concierge = the BFF server
- The taxi company, restaurant, and theatre = the separate backend microservices (user service, orders service, etc.)

---

## Table of Contents
1. [Introduction and Origin](#introduction-and-origin)
2. [What Problem BFF Solves](#what-problem-bff-solves)
3. [Architecture Structure](#architecture-structure)
4. [Core Responsibilities](#core-responsibilities)
5. [BFF vs API Gateway](#bff-vs-api-gateway)
6. [BFF vs GraphQL](#bff-vs-graphql)
7. [Implementation Options](#implementation-options)
8. [Code Examples](#code-examples)
9. [Security Considerations](#security-considerations)
10. [Deployment Strategies](#deployment-strategies)
11. [When NOT to Use BFF](#when-not-to-use-bff)
12. [Interview Questions](#interview-questions)

---

## Introduction and Origin

The Backend for Frontend pattern was named and popularized by **Sam Newman** (author of *Building Microservices*) in a 2015 blog post. The pattern emerged from practical pain at companies like **SoundCloud** and **Netflix**, where teams discovered that a single general-purpose API serving every client type — web, mobile, third-party — accumulated incompatible compromises over time.

SoundCloud's engineering team documented how their mobile and web teams were constantly blocked by a shared API that could not satisfy both clients without over-fetching or under-fetching. The API became a negotiation bottleneck. Netflix faced a similar problem serving vastly different device profiles (smart TVs, phones, tablets, browsers) that needed wildly different data shapes from the same underlying microservices.

The insight: **a shared backend serving multiple frontend types is an implicit coupling point that slows every team it touches.** The solution is to give each client type its own thin backend layer, owned by the frontend team, that speaks the client's language.

---

## What Problem BFF Solves

### 1. Per-Client API Shape

Different clients have structurally different needs. A mobile home screen might need a single endpoint returning a unified feed object. A web dashboard needs granular endpoints for independent panel loading. A third-party API consumer needs stable, versioned contracts.

Without BFF, you end up with one of these bad outcomes:
- A bloated "kitchen sink" endpoint that returns everything for all clients, wasting bandwidth on every request.
- A proliferation of query parameters (`?format=mobile`, `?include=thumbnail`) that turn the API into an implicit DSL nobody documents.
- Constant backend team coordination tax every time a frontend team wants to change a screen.

BFF gives each client team a backend surface they own and can change on their own deployment cycle.

### 2. Payload Optimization

Mobile clients operate on constrained bandwidth and battery. Sending a 40-field user object when the mobile nav bar only needs `id`, `displayName`, and `avatarUrl` is wasteful. The BFF trims response payloads to exactly what the client renders, at the BFF layer, without requiring changes to any downstream microservice.

### 3. Authentication Aggregation

In a microservices architecture, multiple services may have different auth requirements: one uses JWT, another uses mutual TLS, a third expects an API key. The client should not need to know or manage these differences. The BFF holds the downstream credentials, presents a single auth surface to the client (typically a session cookie or a single token), and translates outbound calls to whatever auth scheme each downstream service requires.

This is especially important for server-side session management: the BFF can hold a session cookie that never leaves the server, forwarding access tokens downstream without ever exposing them to browser JavaScript.

### 4. Protocol Translation

Clients almost always speak HTTP/JSON. Downstream services may speak gRPC, GraphQL, AMQP, or Thrift. The BFF absorbs this impedance mismatch. The client talks plain REST or GraphQL to the BFF; the BFF translates to gRPC calls to service A, a message queue publish to service B, and a SQL query to service C — returning a single composed response.

---

## Architecture Structure

```
                    ┌─────────────────────────────────────┐
                    │         Downstream Services         │
                    │                                     │
                    │  ┌──────────┐  ┌──────────────┐    │
                    │  │ User Svc │  │  Orders Svc  │    │
                    │  │ (gRPC)   │  │  (REST/JSON) │    │
                    │  └────┬─────┘  └──────┬───────┘    │
                    │       │               │             │
                    │  ┌────┴───────────────┴───────┐     │
                    │  │      Product Svc (gRPC)    │     │
                    │  └────────────────────────────┘     │
                    └──────────────────────────────┬──────┘
                                                   │
              ┌────────────────────────────────────┼────────────────────────────────────┐
              │                                    │                                    │
    ┌─────────▼─────────┐              ┌───────────▼───────────┐         ┌─────────────▼──────────┐
    │    Web BFF        │              │     Mobile BFF        │         │   Third-Party BFF      │
    │                   │              │                       │         │   (Partner API)        │
    │ - Rich payloads   │              │ - Trimmed payloads    │         │ - Versioned contracts  │
    │ - Session cookies │              │ - Token-based auth    │         │ - API keys             │
    │ - Full field sets │              │ - Offline-friendly    │         │ - Rate limiting        │
    │ - SSR data needs  │              │   response shape      │         │ - Stable schema        │
    └─────────┬─────────┘              └───────────┬───────────┘         └─────────────┬──────────┘
              │                                    │                                   │
    ┌─────────▼─────────┐              ┌───────────▼───────────┐         ┌─────────────▼──────────┐
    │   React/Angular   │              │   iOS / Android App   │         │   Partner Integrations │
    │   Web App         │              │                       │         │                        │
    └───────────────────┘              └───────────────────────┘         └────────────────────────┘
```

**Key structural principle:** each BFF is owned by the team that builds the client it serves. The web team owns the web BFF. The mobile team owns the mobile BFF. They deploy independently.

---

## Core Responsibilities

### 1. Aggregating Downstream Microservices

The most common BFF operation is the "parallel fan-out and join": fire multiple downstream requests concurrently and compose their results into a single response the client can use in one render cycle.

```
Client Request: GET /api/dashboard
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
     User Service   Orders Svc   Analytics Svc
     (concurrent)  (concurrent)  (concurrent)
          │             │             │
          └─────────────┼─────────────┘
                        ▼
              Composed dashboard payload
                returned to client
```

This eliminates multiple round-trips from the client and removes the need to expose every microservice's URL to the browser.

### 2. Data Transformation and Reshaping

Downstream services are designed around domain models, not UI models. The BFF converts:
- Snake case to camelCase (or vice versa)
- Unix timestamps to ISO strings
- Nested domain objects to flattened view models
- Paginated cursor-based responses to offset-based pagination the UI framework expects
- Error codes and domain exceptions to user-facing message shapes

This prevents domain logic from leaking into the frontend bundle.

### 3. Auth Token Handling

The BFF is the natural trust boundary for authentication:
- Receives a session cookie or access token from the client
- Validates it (verifying JWT signature, or checking session store)
- Attaches the correct downstream credentials for each service call (forwarding an access token, injecting a service account key, etc.)
- Never exposes raw access tokens in the response body back to the browser

For OAuth flows, the BFF can implement the **Token Handler Pattern**: the browser holds only an HTTP-only, SameSite=Strict session cookie; the BFF manages the OAuth token exchange, refresh, and storage entirely server-side.

### 4. Rate Limiting Per Client

Different clients have different usage patterns. Mobile clients that poll aggressively need different rate limits than web clients making user-triggered requests. The BFF applies client-specific rate limiting, circuit breakers, and retry budgets — logic that would be inappropriate to enforce at the general API gateway level, which cannot know which client type is making a call.

---

## BFF vs API Gateway

This distinction is frequently muddled in interviews. They solve different problems and are not mutually exclusive.

| Concern | API Gateway | BFF |
|---|---|---|
| **Primary purpose** | Cross-cutting infrastructure | Client-specific data aggregation and shaping |
| **Who owns it** | Platform/infra team | Frontend team |
| **Number of instances** | One (or one per environment) | One per client type |
| **Client awareness** | None — routes blindly | Deep — knows the client's data needs |
| **Business logic** | None | Light: composition, transformation, auth forwarding |
| **Auth** | Token validation, mTLS termination | Token exchange, session management, forwarding |
| **Rate limiting** | Global rate limits | Per-client-type limits |
| **Response shaping** | No | Yes |

The typical deployment has **both**: an API gateway handling TLS termination, global auth token validation, DDoS protection, and request logging sits in front of the BFFs. Each BFF then handles client-specific orchestration against the downstream microservices.

```
Internet → API Gateway → Web BFF → [Downstream microservices]
                       → Mobile BFF → [Downstream microservices]
```

The API gateway does not know or care that Web BFF and Mobile BFF produce different response shapes. That is not its concern.

---

## BFF vs GraphQL

GraphQL is sometimes proposed as an alternative to BFF because both solve the over-fetching and under-fetching problem. The comparison is real but often oversimplified.

### When GraphQL can replace BFF

- You have multiple clients with truly varying, unpredictable data requirements.
- Your team has GraphQL expertise and tooling already in place.
- You want clients to self-serve their data shape without backend deployments.
- Your data model is genuinely graph-shaped (interconnected entities with many traversal patterns).

### When BFF is better than GraphQL

**Non-graph data**: REST microservices returning flat resource collections do not benefit from a graph layer. Wrapping REST APIs in GraphQL resolvers adds indirection without value.

**Tight coupling is acceptable**: When there is one web team owning both the React app and its BFF, tight coupling is a feature, not a bug. GraphQL's contract-first, schema-evolution discipline adds overhead when the consumer and producer are the same team.

**Simpler stack**: GraphQL demands schema definition, resolver authoring, N+1 query protection (DataLoader), persisted queries for performance, and a schema registry for governance. A BFF is a plain HTTP service with async/await — every senior engineer on the team already knows how to write and debug it.

**Security**: GraphQL's flexible query surface makes it harder to reason about worst-case query cost. A BFF exposes a fixed set of endpoints; you know exactly what each one does.

**Performance-sensitive aggregation**: GraphQL resolvers compose sequentially per field in naive implementations. A BFF can hand-optimize the concurrency pattern — running three downstream calls in parallel, racing two alternatives, or short-circuiting on a cache hit — in ways that are awkward to express in a resolver tree.

**The honest summary**: BFF and GraphQL solve overlapping but not identical problems. Large companies (GitHub, Shopify) use GraphQL because they have many disparate clients. Most product teams with two or three known client types are better served by a BFF.

---

## Implementation Options

### Option 1: Next.js Route Handlers (App Router)

Collocated with the frontend. No additional deployment. Runs as edge functions or serverless lambdas on Vercel/Cloudflare.

```
app/
  api/
    dashboard/
      route.ts   ← BFF logic lives here
  dashboard/
    page.tsx
```

Best for: web-only products, Vercel-deployed apps, teams that want zero infrastructure overhead.

### Option 2: Remix Loaders and Actions

Remix's `loader` function runs server-side on every route render, making it a natural BFF entry point. No separate routing layer needed.

```typescript
// routes/dashboard.tsx
export async function loader({ request }: LoaderFunctionArgs) {
  // This IS the BFF — runs on the server, composes downstream calls
  const [user, orders] = await Promise.all([
    fetchUser(request),
    fetchOrders(request),
  ]);
  return json({ user, orders });
}
```

Best for: full-stack Remix apps, teams that want the BFF as a first-class framework concept rather than an afterthought.

### Option 3: Dedicated Express / Fastify / Hono Service

A separate Node.js (or Deno/Bun) service deployed independently from the frontend. Full control over middleware, caching, circuit breakers, and deployment topology.

```
frontend-repo/     ← React or Angular SPA
web-bff/           ← Separate service, separate deploy
  src/
    routes/
    middleware/
    services/      ← Downstream API clients
  Dockerfile
```

Best for: Angular SPAs (which have no server-side rendering layer to colocate with), organizations that need separate scaling of the BFF, teams that want full infrastructure control.

**Hono** is increasingly popular for BFF use because it runs natively on Cloudflare Workers, Deno Deploy, Node, and Bun with the same codebase — matching the BFF's role as a thin intermediary.

---

## Code Examples

### Next.js Route Handler as BFF

This example shows a `/api/dashboard` BFF endpoint that aggregates a user profile and recent orders from two independent downstream services, handles auth forwarding, trims the payload to only what the UI needs, and sanitizes errors.

```typescript
// app/api/dashboard/route.ts
import { NextRequest, NextResponse } from "next/server";
import { getServerSession } from "next-auth";
import { authOptions } from "@/lib/auth";

// Downstream service clients — these would live in a services/ directory
async function fetchUserProfile(userId: string, accessToken: string) {
  const res = await fetch(
    `${process.env.USER_SERVICE_URL}/users/${userId}`,
    {
      headers: {
        Authorization: `Bearer ${accessToken}`,
        "X-Service-Client": "web-bff",
      },
      // Next.js fetch cache — revalidate every 60s
      next: { revalidate: 60 },
    }
  );

  if (!res.ok) {
    // Never forward raw downstream errors to the client
    throw new Error(`USER_SERVICE_ERROR:${res.status}`);
  }

  const raw = await res.json();

  // Transform: downstream uses snake_case, UI expects camelCase + only needed fields
  return {
    id: raw.user_id,
    displayName: raw.display_name,
    avatarUrl: raw.profile_image_url,
    plan: raw.subscription_tier,
  };
}

async function fetchRecentOrders(userId: string, accessToken: string) {
  const res = await fetch(
    `${process.env.ORDERS_SERVICE_URL}/orders?userId=${userId}&limit=5`,
    {
      headers: {
        Authorization: `Bearer ${accessToken}`,
        "X-Service-Client": "web-bff",
      },
      cache: "no-store", // Orders must be fresh
    }
  );

  if (!res.ok) {
    throw new Error(`ORDERS_SERVICE_ERROR:${res.status}`);
  }

  const raw = await res.json();

  // Reshape: downstream returns paginated cursor response, UI wants a flat array
  return raw.data.map((order: any) => ({
    id: order.order_id,
    status: order.fulfillment_status,
    total: order.price_cents / 100,         // cents → dollars at the BFF layer
    placedAt: new Date(order.created_at * 1000).toISOString(), // unix → ISO
    itemCount: order.line_items.length,
  }));
}

export async function GET(request: NextRequest) {
  // 1. Authenticate the client request — BFF is the trust boundary
  const session = await getServerSession(authOptions);

  if (!session?.user?.id || !session?.accessToken) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const { userId, accessToken } = session as {
    userId: string;
    accessToken: string;
  };

  try {
    // 2. Fan-out: call both downstream services concurrently
    const [user, recentOrders] = await Promise.all([
      fetchUserProfile(userId, accessToken),
      fetchRecentOrders(userId, accessToken),
    ]);

    // 3. Compose the response — exactly what the dashboard page needs, nothing more
    return NextResponse.json(
      {
        user,
        recentOrders,
        meta: {
          fetchedAt: new Date().toISOString(),
        },
      },
      {
        status: 200,
        headers: {
          // Client-level cache: fresh for 30s, stale-while-revalidate for 60s
          "Cache-Control": "private, max-age=30, stale-while-revalidate=60",
        },
      }
    );
  } catch (error) {
    // 4. Sanitize errors — never leak downstream service internals to the browser
    console.error("[web-bff] dashboard aggregation failed:", error);

    const message = error instanceof Error ? error.message : "Unknown error";

    // Map internal error codes to safe client-facing responses
    if (message.startsWith("USER_SERVICE_ERROR:503")) {
      return NextResponse.json(
        { error: "Profile service temporarily unavailable" },
        { status: 503 }
      );
    }

    return NextResponse.json(
      { error: "Failed to load dashboard data" },
      { status: 500 }
    );
  }
}
```

The React component consuming this endpoint makes a single fetch call and receives a fully composed, correctly shaped payload:

```typescript
// app/dashboard/page.tsx
export default async function DashboardPage() {
  // Server Component — fetches from our own BFF, not from the microservices directly
  const res = await fetch("/api/dashboard", { cache: "no-store" });

  if (!res.ok) {
    throw new Error("Failed to load dashboard");
  }

  const { user, recentOrders } = await res.json();

  return (
    <main>
      <UserHeader user={user} />
      <RecentOrders orders={recentOrders} />
    </main>
  );
}
```

### Dedicated Hono BFF for an Angular SPA

When the frontend is Angular (no SSR layer to colocate with), a standalone Hono service acts as the BFF:

```typescript
// web-bff/src/index.ts
import { Hono } from "hono";
import { cors } from "hono/cors";
import { bearerAuth } from "hono/bearer-auth";
import { rateLimiter } from "hono-rate-limiter";

const app = new Hono();

// CORS: only allow requests from the Angular app's origin
app.use(
  "/api/*",
  cors({
    origin: process.env.WEB_APP_ORIGIN!,
    credentials: true,
  })
);

// Rate limit: web clients get 120 requests/min
app.use(
  "/api/*",
  rateLimiter({
    windowMs: 60_000,
    limit: 120,
    keyGenerator: (c) => c.req.header("x-forwarded-for") ?? "unknown",
  })
);

app.get("/api/dashboard", async (c) => {
  const authHeader = c.req.header("Authorization");
  if (!authHeader?.startsWith("Bearer ")) {
    return c.json({ error: "Unauthorized" }, 401);
  }

  const token = authHeader.slice(7);

  // Validate token, extract userId
  const userId = await validateToken(token);
  if (!userId) return c.json({ error: "Unauthorized" }, 401);

  try {
    const [user, orders] = await Promise.all([
      fetchUserProfile(userId, token),
      fetchRecentOrders(userId, token),
    ]);

    return c.json({ user, orders });
  } catch (err) {
    console.error("[web-bff]", err);
    return c.json({ error: "Service unavailable" }, 503);
  }
});

export default app;
```

---

## Security Considerations

The BFF is a trust boundary. Getting this wrong creates serious vulnerabilities.

### Never Expose Raw Downstream Errors

Downstream error messages often contain internal hostnames, service names, stack traces, or database errors. Always catch and map to generic client-facing messages. Log the full error server-side.

```typescript
// WRONG — leaks internal topology
return NextResponse.json({ error: err.message }, { status: 500 });
// "connect ECONNREFUSED orders-service.internal:3001" — tells attackers your internal DNS

// RIGHT — safe, logged internally
console.error("[bff] upstream failure:", err);
return NextResponse.json({ error: "Service unavailable" }, { status: 503 });
```

### Session Token Storage: Token Handler Pattern

Store access tokens server-side in the BFF session, never in `localStorage` or as a readable cookie. The browser holds only an HTTP-only, Secure, SameSite=Strict session cookie. The BFF exchanges this for the access token on every request.

```
Browser              Web BFF              Auth Server
  │                    │                     │
  │  POST /auth/login  │                     │
  │──────────────────► │                     │
  │                    │  Authorization Code │
  │                    │────────────────────►│
  │                    │  Access + Refresh   │
  │                    │◄────────────────────│
  │                    │ (stored server-side)│
  │  Set-Cookie:       │                     │
  │  session=<opaque>  │                     │
  │◄──────────────────│                     │
  │                    │                     │
  │  GET /api/data     │                     │
  │  Cookie: session=  │                     │
  │──────────────────► │                     │
  │                    │  Bearer access_token│
  │                    │────────────────────►│ downstream
```

### Input Validation

The BFF must validate and sanitize all inputs before forwarding them downstream. If a client sends a malformed `userId`, validate it at the BFF rather than forwarding garbage to a microservice.

### CORS Policy

Configure CORS on the BFF to only accept requests from your frontend's known origins. A BFF that accepts requests from any origin defeats the trust boundary.

### Do Not Forward All Headers

Be explicit about which headers you forward to downstream services. Never blindly proxy all client headers — this can forward cookies, host headers, or other client data to internal services that should never see them.

---

## Deployment Strategies

### Colocation with the Frontend

**Next.js on Vercel / Remix on Fly.io**: The BFF route handlers deploy as part of the frontend application. No separate service, no separate infrastructure, no inter-service latency.

Advantages:
- Zero-latency BFF-to-frontend communication (same process or same edge node).
- Single deployment pipeline — frontend and BFF ship together.
- No CORS configuration needed between frontend and BFF.
- Shared environment variables and secrets.

Disadvantages:
- BFF logic scales with the frontend, not independently.
- Frontend team must manage all server-side code.
- Not suitable for Angular/Vue SPAs without an SSR framework.

### Separate Service

A standalone Node/Deno/Go service, Dockerized and deployed independently.

```
Frontend SPA (Cloudflare Pages) → Web BFF (Cloudflare Workers)
                                → [Downstream services]
```

Advantages:
- BFF can scale independently (BFF under heavy traffic does not scale the static frontend assets).
- Clear team ownership boundary when the org separates frontend and backend-for-frontend teams.
- Language flexibility — BFF can be Go or Rust if latency is critical.
- Independent deployment cadence.

Disadvantages:
- Cross-origin request from SPA to BFF requires CORS configuration.
- Additional infrastructure to manage, monitor, and secure.
- Extra network hop (small, but real).

### Edge Deployment

Deploying the BFF as a Cloudflare Worker or Vercel Edge Function runs the aggregation logic geographically close to the user, reducing latency on the client-to-BFF leg.

This is most valuable when downstream services are also distributed (using Cloudflare D1, R2, or Durable Objects). If downstream microservices are centralized in us-east-1, routing through an edge BFF in Tokyo to then hop back to Virginia adds latency rather than removing it — design accordingly.

---

## When NOT to Use BFF

**Simple applications with one client type**: If you have a single web app and no plans for mobile or third-party API consumers, a BFF adds a layer with no payoff. Build your API with the web client's needs in mind directly.

**Early-stage products**: When requirements are unclear and the team is small, a BFF is premature architecture. The indirection slows iteration. Start with a direct API and extract a BFF when you have two or more clients with genuinely divergent needs.

**Team lacks bandwidth**: A BFF is a service that needs to be deployed, monitored, and maintained. If the frontend team cannot own a Node service in production (no DevOps support, no observability tooling), the BFF becomes a reliability liability.

**Downstream services already have a well-designed aggregation layer**: If your backend team has already built a well-shaped composite API that works well for all clients, adding a BFF duplicates aggregation logic for no gain.

**Performance-critical, low-latency paths**: Every BFF hop adds network latency. For real-time or ultra-low-latency use cases (trading platforms, live video), evaluate whether the BFF is on a hot path and consider alternatives (WebSocket gateways, server-sent events direct from source).

---

## Interview Questions

**Q1: How does a BFF differ from an API Gateway, and when would you deploy both?**

An API gateway handles cross-cutting infrastructure concerns: TLS termination, global rate limiting, auth token validation, request logging, and DDoS protection. It is owned by a platform team and is client-agnostic — it routes requests without knowing anything about what data the client needs.

A BFF is a thin application layer owned by the frontend team that knows exactly what each client type needs. It aggregates multiple microservice calls, transforms payloads to match the UI's data model, and handles client-specific auth session management.

You deploy both when you have a microservices architecture with multiple client types. The API gateway sits at the edge and handles infrastructure-level concerns. Traffic then flows to the appropriate BFF, which handles orchestration. The gateway does not need to know what the BFF returns; the BFF does not need to concern itself with TLS or DDoS.

---

**Q2: Why might a team choose a BFF over GraphQL for aggregating multiple microservices?**

GraphQL solves a similar over-fetching/under-fetching problem, but it introduces schema definition, resolver implementation, N+1 query protection (DataLoader), schema governance, and a new query language the frontend team must adopt and the backend team must secure.

A BFF with plain HTTP/JSON endpoints has a near-zero learning curve, is trivially debuggable with curl or Postman, and allows fine-grained hand-optimization of concurrency patterns (parallel fan-out, short-circuit on cache hits) that are awkward in a resolver tree.

BFF is preferable when: the number of client types is small and known; the team does not already have GraphQL expertise; the data model is not genuinely graph-shaped; or when the tight coupling between the frontend team and their BFF is a feature (they co-deploy, they move together) rather than something to be avoided.

---

**Q3: Describe how you would implement the Token Handler Pattern in a Next.js BFF to prevent access token exposure in the browser.**

The Token Handler Pattern ensures that OAuth access tokens and refresh tokens never appear in browser JavaScript or readable cookies.

The Next.js BFF (Route Handlers) handles the OAuth authorization code flow entirely server-side: it receives the authorization code callback, exchanges it for tokens with the auth server, and stores the access token and refresh token in a server-side session store (Redis, encrypted database, or an encrypted Next.js iron-session cookie that is HTTP-only and Secure).

The browser receives only an opaque session identifier in an HTTP-only, Secure, SameSite=Strict cookie. On subsequent API requests, the browser sends this cookie; the BFF reads it, retrieves the access token from the session store, attaches it as a Bearer token to downstream service calls, and returns the composed response. The access token never appears in a response body, in `localStorage`, or in a JavaScript-readable cookie.

On token expiry, the BFF silently refreshes using the stored refresh token before forwarding the downstream call — the client is never exposed to token lifecycle management.

---

**Q4: Your company is building a product with a React web app, a React Native mobile app, and a public third-party API. How would you structure the BFF layer, and who owns each piece?**

Three separate BFFs, each owned by the team closest to that client:

- **Web BFF** (owned by web frontend team): deployed as Next.js Route Handlers on Vercel. Returns full payloads optimized for web layouts. Manages OAuth session cookies. Aggregates user, product, and analytics services for dashboard views. Can evolve at the web team's deployment cadence.

- **Mobile BFF** (owned by mobile team): deployed as a standalone Hono or Fastify service. Returns trimmed payloads for bandwidth efficiency. Handles push notification token registration. Implements aggressive client-specific rate limits. Exposes endpoints shaped around mobile screen flows, not domain resources.

- **Partner API BFF** (owned by a platform/API team): deployed as a separate service with strict versioning, OpenAPI documentation, and stable contract guarantees. Enforces API key authentication, per-partner rate limits, and audit logging. Changes to this surface require a deprecation process.

All three BFFs share the same downstream microservices. An API gateway sits in front of all three, handling TLS, global DDoS protection, and routing based on hostname or path prefix. The microservices do not know which BFF called them — they receive authenticated service-to-service calls regardless of origin.
