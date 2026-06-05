# Next.js Route Handlers

## The Idea

**In plain English:** A route handler is a special function you write on the server side of your website that listens for incoming requests at a specific web address (called a "route") and decides what data to send back. Think of it as the part of your website that handles questions like "give me the list of users" or "save this new message."

**Real-world analogy:** Imagine a restaurant where customers place orders at specific windows — one window for takeout, one for catering, one for complaints. Each window has a dedicated staff member who listens for requests, looks up what is needed, and hands back the right response.

- The restaurant window address (e.g., "Window 3 - Takeout") = the route URL (e.g., `/api/orders`)
- The type of request the customer makes (order food, cancel order, update order) = the HTTP method (GET, POST, DELETE)
- The staff member who handles the request and prepares the response = the route handler function

---

## Overview

Route Handlers allow you to create custom request handlers for API routes using the Web Request and Response APIs. They replace API Routes from the Pages Router and provide a more powerful, flexible way to handle HTTP requests.

## Table of Contents

1. [Basic Route Handlers](#basic-route-handlers)
2. [HTTP Methods](#http-methods)
3. [NextRequest and NextResponse](#nextrequest-and-nextresponse)
4. [Middleware](#middleware)
5. [Edge Runtime vs Node.js Runtime](#edge-runtime-vs-nodejs-runtime)
6. [Streaming Responses](#streaming-responses)
7. [Route Handler Patterns](#route-handler-patterns)
8. [Common Mistakes](#common-mistakes)
9. [Best Practices](#best-practices)
10. [Interview Questions](#interview-questions)

## Basic Route Handlers

Route handlers are defined in `route.ts` files within the `app` directory.

### Simple GET Handler

```typescript
// app/api/hello/route.ts
import { NextResponse } from 'next/server';

export async function GET() {
  return NextResponse.json({ message: 'Hello, World!' });
}
```

### Reading URL Parameters

```typescript
// app/api/users/[id]/route.ts
import { NextResponse } from 'next/server';

export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  const { id } = params;
  
  const user = await fetch(`https://api.example.com/users/${id}`)
    .then(res => res.json());
  
  return NextResponse.json(user);
}
```

### Reading Search Params

```typescript
// app/api/search/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const query = searchParams.get('q');
  const page = searchParams.get('page') || '1';
  const limit = searchParams.get('limit') || '10';
  
  const results = await fetch(
    `https://api.example.com/search?q=${query}&page=${page}&limit=${limit}`
  ).then(res => res.json());
  
  return NextResponse.json(results);
}

// Usage: GET /api/search?q=javascript&page=2&limit=20
```

## HTTP Methods

Route handlers support all HTTP methods: GET, POST, PUT, PATCH, DELETE, HEAD, and OPTIONS.

### POST Handler

```typescript
// app/api/posts/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function POST(request: NextRequest) {
  const body = await request.json();
  
  // Validate input
  if (!body.title || !body.content) {
    return NextResponse.json(
      { error: 'Title and content are required' },
      { status: 400 }
    );
  }
  
  // Create post
  const post = await fetch('https://api.example.com/posts', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  }).then(res => res.json());
  
  return NextResponse.json(post, { status: 201 });
}
```

### PUT Handler

```typescript
// app/api/posts/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function PUT(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const { id } = params;
  const body = await request.json();
  
  const updatedPost = await fetch(`https://api.example.com/posts/${id}`, {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  }).then(res => res.json());
  
  return NextResponse.json(updatedPost);
}
```

### PATCH Handler

```typescript
// app/api/users/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function PATCH(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const { id } = params;
  const body = await request.json();
  
  // Partial update
  const user = await fetch(`https://api.example.com/users/${id}`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  }).then(res => res.json());
  
  return NextResponse.json(user);
}
```

### DELETE Handler

```typescript
// app/api/posts/[id]/route.ts
import { NextResponse } from 'next/server';

export async function DELETE(
  request: Request,
  { params }: { params: { id: string } }
) {
  const { id } = params;
  
  await fetch(`https://api.example.com/posts/${id}`, {
    method: 'DELETE',
  });
  
  return NextResponse.json({ success: true }, { status: 204 });
}
```

### Multiple Methods in One File

```typescript
// app/api/todos/route.ts
import { NextRequest, NextResponse } from 'next/server';

// GET /api/todos
export async function GET(request: NextRequest) {
  const todos = await fetch('https://api.example.com/todos')
    .then(res => res.json());
  
  return NextResponse.json(todos);
}

// POST /api/todos
export async function POST(request: NextRequest) {
  const body = await request.json();
  
  const todo = await fetch('https://api.example.com/todos', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  }).then(res => res.json());
  
  return NextResponse.json(todo, { status: 201 });
}

// app/api/todos/[id]/route.ts
export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  const todo = await fetch(`https://api.example.com/todos/${params.id}`)
    .then(res => res.json());
  
  return NextResponse.json(todo);
}

export async function PUT(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const body = await request.json();
  
  const todo = await fetch(`https://api.example.com/todos/${params.id}`, {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  }).then(res => res.json());
  
  return NextResponse.json(todo);
}

export async function DELETE(
  request: Request,
  { params }: { params: { id: string } }
) {
  await fetch(`https://api.example.com/todos/${params.id}`, {
    method: 'DELETE',
  });
  
  return new NextResponse(null, { status: 204 });
}
```

## NextRequest and NextResponse

### NextRequest Features

```typescript
// app/api/debug/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET(request: NextRequest) {
  // URL information
  const url = request.nextUrl;
  const pathname = url.pathname;
  const searchParams = url.searchParams;
  
  // Headers
  const userAgent = request.headers.get('user-agent');
  const referer = request.headers.get('referer');
  const authorization = request.headers.get('authorization');
  
  // Cookies
  const token = request.cookies.get('token');
  const allCookies = request.cookies.getAll();
  
  // Geo information (available on Edge runtime)
  const geo = request.geo;
  const ip = request.ip;
  
  return NextResponse.json({
    url: {
      pathname,
      searchParams: Object.fromEntries(searchParams),
    },
    headers: {
      userAgent,
      referer,
      authorization,
    },
    cookies: {
      token: token?.value,
      all: allCookies,
    },
    geo,
    ip,
  });
}
```

### NextResponse Features

```typescript
// app/api/login/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function POST(request: NextRequest) {
  const { email, password } = await request.json();
  
  // Authenticate user
  const user = await authenticateUser(email, password);
  
  if (!user) {
    return NextResponse.json(
      { error: 'Invalid credentials' },
      { status: 401 }
    );
  }
  
  const response = NextResponse.json({ user });
  
  // Set cookie
  response.cookies.set({
    name: 'token',
    value: user.token,
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 60 * 60 * 24 * 7, // 1 week
  });
  
  // Set headers
  response.headers.set('X-User-Id', user.id);
  response.headers.set('Cache-Control', 'no-store');
  
  return response;
}

async function authenticateUser(email: string, password: string) {
  // Authentication logic
  return { id: '1', email, token: 'jwt-token' };
}
```

### Setting Multiple Cookies

```typescript
// app/api/auth/route.ts
import { NextResponse } from 'next/server';

export async function POST(request: Request) {
  const { email, password } = await request.json();
  
  const { accessToken, refreshToken, user } = await login(email, password);
  
  const response = NextResponse.json({ user });
  
  // Set access token
  response.cookies.set('accessToken', accessToken, {
    httpOnly: true,
    secure: true,
    sameSite: 'strict',
    maxAge: 15 * 60, // 15 minutes
  });
  
  // Set refresh token
  response.cookies.set('refreshToken', refreshToken, {
    httpOnly: true,
    secure: true,
    sameSite: 'strict',
    maxAge: 60 * 60 * 24 * 30, // 30 days
  });
  
  return response;
}

async function login(email: string, password: string) {
  return {
    accessToken: 'access-token',
    refreshToken: 'refresh-token',
    user: { id: '1', email },
  };
}
```

### Redirects

```typescript
// app/api/redirect/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const destination = searchParams.get('to');
  
  if (!destination) {
    return NextResponse.json(
      { error: 'Destination required' },
      { status: 400 }
    );
  }
  
  // Redirect
  return NextResponse.redirect(new URL(destination, request.url));
}

// Usage: GET /api/redirect?to=/dashboard
```

### Rewrites

```typescript
// app/api/proxy/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const apiPath = searchParams.get('path');
  
  // Rewrite to external API
  return NextResponse.rewrite(
    new URL(`https://external-api.com${apiPath}`, request.url)
  );
}
```

## Middleware

Middleware runs before route handlers and can modify requests/responses.

### Basic Middleware

```typescript
// middleware.ts
import { NextRequest, NextResponse } from 'next/server';

export function middleware(request: NextRequest) {
  // Add custom header
  const response = NextResponse.next();
  response.headers.set('X-Custom-Header', 'my-value');
  
  return response;
}

// Configure matcher
export const config = {
  matcher: '/api/:path*',
};
```

### Authentication Middleware

```typescript
// middleware.ts
import { NextRequest, NextResponse } from 'next/server';

export function middleware(request: NextRequest) {
  const token = request.cookies.get('token');
  
  // Protect API routes
  if (request.nextUrl.pathname.startsWith('/api/protected')) {
    if (!token) {
      return NextResponse.json(
        { error: 'Unauthorized' },
        { status: 401 }
      );
    }
    
    // Verify token (simplified)
    const isValid = verifyToken(token.value);
    if (!isValid) {
      return NextResponse.json(
        { error: 'Invalid token' },
        { status: 401 }
      );
    }
  }
  
  return NextResponse.next();
}

export const config = {
  matcher: '/api/:path*',
};

function verifyToken(token: string): boolean {
  // Token verification logic
  return true;
}
```

### Redirect Based on Auth

```typescript
// middleware.ts
import { NextRequest, NextResponse } from 'next/server';

export function middleware(request: NextRequest) {
  const isAuthenticated = request.cookies.has('token');
  const isAuthPage = request.nextUrl.pathname.startsWith('/auth');
  const isProtectedPage = request.nextUrl.pathname.startsWith('/dashboard');
  
  // Redirect to login if accessing protected page without auth
  if (isProtectedPage && !isAuthenticated) {
    const loginUrl = new URL('/auth/login', request.url);
    loginUrl.searchParams.set('from', request.nextUrl.pathname);
    return NextResponse.redirect(loginUrl);
  }
  
  // Redirect to dashboard if authenticated user tries to access auth pages
  if (isAuthPage && isAuthenticated) {
    return NextResponse.redirect(new URL('/dashboard', request.url));
  }
  
  return NextResponse.next();
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
};
```

### Rate Limiting Middleware

```typescript
// middleware.ts
import { NextRequest, NextResponse } from 'next/server';

const rateLimit = new Map<string, { count: number; resetAt: number }>();

export function middleware(request: NextRequest) {
  const ip = request.ip || 'unknown';
  const now = Date.now();
  
  // Get or create rate limit entry
  let entry = rateLimit.get(ip);
  
  if (!entry || now > entry.resetAt) {
    entry = { count: 0, resetAt: now + 60000 }; // 1 minute window
    rateLimit.set(ip, entry);
  }
  
  entry.count++;
  
  // Check if rate limit exceeded (10 requests per minute)
  if (entry.count > 10) {
    return NextResponse.json(
      { error: 'Too many requests' },
      { status: 429 }
    );
  }
  
  const response = NextResponse.next();
  response.headers.set('X-RateLimit-Limit', '10');
  response.headers.set('X-RateLimit-Remaining', String(10 - entry.count));
  
  return response;
}

export const config = {
  matcher: '/api/:path*',
};
```

### A/B Testing Middleware

```typescript
// middleware.ts
import { NextRequest, NextResponse } from 'next/server';

export function middleware(request: NextRequest) {
  // Check if user has a variant cookie
  let variant = request.cookies.get('ab-test-variant');
  
  if (!variant) {
    // Assign random variant
    variant = { name: 'ab-test-variant', value: Math.random() < 0.5 ? 'A' : 'B' };
  }
  
  const response = NextResponse.next();
  
  // Set variant cookie
  response.cookies.set(variant.name, variant.value, {
    maxAge: 60 * 60 * 24 * 30, // 30 days
  });
  
  // Add variant to header for route handlers
  response.headers.set('X-Variant', variant.value);
  
  return response;
}

export const config = {
  matcher: '/:path*',
};
```

## Edge Runtime vs Node.js Runtime

### Node.js Runtime (Default)

```typescript
// app/api/node/route.ts
import { NextResponse } from 'next/server';
import fs from 'fs';
import path from 'path';

// Default runtime is Node.js
export async function GET() {
  // Can use Node.js APIs
  const filePath = path.join(process.cwd(), 'data.json');
  const data = fs.readFileSync(filePath, 'utf-8');
  
  return NextResponse.json(JSON.parse(data));
}
```

### Edge Runtime

```typescript
// app/api/edge/route.ts
import { NextResponse } from 'next/server';

// Opt-in to Edge runtime
export const runtime = 'edge';

export async function GET() {
  // No Node.js APIs available
  // Much faster cold starts
  // Globally distributed
  
  const data = await fetch('https://api.example.com/data')
    .then(res => res.json());
  
  return NextResponse.json(data);
}
```

### Edge with Geolocation

```typescript
// app/api/geo/route.ts
import { NextRequest, NextResponse } from 'next/server';

export const runtime = 'edge';

export async function GET(request: NextRequest) {
  const geo = request.geo;
  
  return NextResponse.json({
    country: geo?.country,
    region: geo?.region,
    city: geo?.city,
    latitude: geo?.latitude,
    longitude: geo?.longitude,
  });
}
```

### When to Use Each Runtime

```typescript
// Use Node.js runtime when:
// - Need access to Node.js APIs (fs, crypto, etc.)
// - Using native Node modules
// - Heavy computation
// - Database connections

// app/api/upload/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { writeFile } from 'fs/promises';

export async function POST(request: NextRequest) {
  const formData = await request.formData();
  const file = formData.get('file') as File;
  
  const bytes = await file.arrayBuffer();
  const buffer = Buffer.from(bytes);
  
  await writeFile(`./uploads/${file.name}`, buffer);
  
  return NextResponse.json({ success: true });
}

// Use Edge runtime when:
// - Need fast cold starts
// - Simple request/response
// - Geolocation-based logic
// - A/B testing, redirects, rewrites

// app/api/personalize/route.ts
import { NextRequest, NextResponse } from 'next/server';

export const runtime = 'edge';

export async function GET(request: NextRequest) {
  const country = request.geo?.country || 'US';
  
  const content = await fetch(
    `https://api.example.com/content?country=${country}`
  ).then(res => res.json());
  
  return NextResponse.json(content);
}
```

## Streaming Responses

### Streaming Text

```typescript
// app/api/stream/route.ts
export const runtime = 'edge';

export async function GET() {
  const encoder = new TextEncoder();
  
  const stream = new ReadableStream({
    async start(controller) {
      const messages = ['Hello', 'World', 'from', 'streaming', 'API'];
      
      for (const message of messages) {
        controller.enqueue(encoder.encode(message + '\n'));
        await new Promise(resolve => setTimeout(resolve, 1000));
      }
      
      controller.close();
    },
  });
  
  return new Response(stream, {
    headers: {
      'Content-Type': 'text/plain',
      'Transfer-Encoding': 'chunked',
    },
  });
}
```

### Streaming JSON

```typescript
// app/api/stream-json/route.ts
export const runtime = 'edge';

export async function GET() {
  const encoder = new TextEncoder();
  
  const stream = new ReadableStream({
    async start(controller) {
      const items = [
        { id: 1, name: 'Item 1' },
        { id: 2, name: 'Item 2' },
        { id: 3, name: 'Item 3' },
      ];
      
      controller.enqueue(encoder.encode('['));
      
      for (let i = 0; i < items.length; i++) {
        const json = JSON.stringify(items[i]);
        controller.enqueue(encoder.encode(json));
        
        if (i < items.length - 1) {
          controller.enqueue(encoder.encode(','));
        }
        
        await new Promise(resolve => setTimeout(resolve, 1000));
      }
      
      controller.enqueue(encoder.encode(']'));
      controller.close();
    },
  });
  
  return new Response(stream, {
    headers: {
      'Content-Type': 'application/json',
      'Transfer-Encoding': 'chunked',
    },
  });
}
```

### Streaming from External API

```typescript
// app/api/proxy-stream/route.ts
export const runtime = 'edge';

export async function GET() {
  const response = await fetch('https://api.example.com/stream');
  
  // Pipe the stream through
  return new Response(response.body, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  });
}
```

### Server-Sent Events (SSE)

```typescript
// app/api/sse/route.ts
export const runtime = 'edge';

export async function GET() {
  const encoder = new TextEncoder();
  
  const stream = new ReadableStream({
    async start(controller) {
      let id = 0;
      
      const interval = setInterval(() => {
        const data = {
          id: ++id,
          time: new Date().toISOString(),
          message: 'Server update',
        };
        
        const event = `data: ${JSON.stringify(data)}\n\n`;
        controller.enqueue(encoder.encode(event));
        
        if (id >= 10) {
          clearInterval(interval);
          controller.close();
        }
      }, 1000);
    },
  });
  
  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  });
}

// Client-side usage:
// const eventSource = new EventSource('/api/sse');
// eventSource.onmessage = (event) => {
//   const data = JSON.parse(event.data);
//   console.log(data);
// };
```

## Route Handler Patterns

### CORS Configuration

```typescript
// app/api/cors/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function OPTIONS(request: NextRequest) {
  return new NextResponse(null, {
    status: 200,
    headers: {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type, Authorization',
    },
  });
}

export async function GET(request: NextRequest) {
  const data = { message: 'CORS enabled' };
  
  return NextResponse.json(data, {
    headers: {
      'Access-Control-Allow-Origin': '*',
    },
  });
}
```

### Error Handling

```typescript
// app/api/users/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  try {
    const { id } = params;
    
    if (!id || isNaN(Number(id))) {
      return NextResponse.json(
        { error: 'Invalid user ID' },
        { status: 400 }
      );
    }
    
    const user = await fetch(`https://api.example.com/users/${id}`)
      .then(async res => {
        if (!res.ok) {
          throw new Error(`User not found: ${res.statusText}`);
        }
        return res.json();
      });
    
    return NextResponse.json(user);
  } catch (error) {
    console.error('Error fetching user:', error);
    
    return NextResponse.json(
      { error: error instanceof Error ? error.message : 'Internal server error' },
      { status: 500 }
    );
  }
}
```

### Pagination

```typescript
// app/api/posts/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const page = parseInt(searchParams.get('page') || '1', 10);
  const limit = parseInt(searchParams.get('limit') || '10', 10);
  const offset = (page - 1) * limit;
  
  const response = await fetch(
    `https://api.example.com/posts?limit=${limit}&offset=${offset}`
  ).then(res => res.json());
  
  const { posts, total } = response;
  const totalPages = Math.ceil(total / limit);
  
  return NextResponse.json({
    posts,
    pagination: {
      page,
      limit,
      total,
      totalPages,
      hasNext: page < totalPages,
      hasPrev: page > 1,
    },
  });
}
```

### File Upload

```typescript
// app/api/upload/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { writeFile } from 'fs/promises';
import { join } from 'path';

export async function POST(request: NextRequest) {
  try {
    const formData = await request.formData();
    const file = formData.get('file') as File;
    
    if (!file) {
      return NextResponse.json(
        { error: 'No file provided' },
        { status: 400 }
      );
    }
    
    const bytes = await file.arrayBuffer();
    const buffer = Buffer.from(bytes);
    
    const uploadDir = join(process.cwd(), 'public', 'uploads');
    const filePath = join(uploadDir, file.name);
    
    await writeFile(filePath, buffer);
    
    return NextResponse.json({
      success: true,
      filename: file.name,
      size: file.size,
      url: `/uploads/${file.name}`,
    });
  } catch (error) {
    return NextResponse.json(
      { error: 'Upload failed' },
      { status: 500 }
    );
  }
}
```

### Webhooks

```typescript
// app/api/webhooks/stripe/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { headers } from 'next/headers';

export async function POST(request: NextRequest) {
  const body = await request.text();
  const signature = headers().get('stripe-signature');
  
  if (!signature) {
    return NextResponse.json(
      { error: 'No signature' },
      { status: 400 }
    );
  }
  
  try {
    // Verify webhook signature
    const event = verifyStripeWebhook(body, signature);
    
    // Handle different event types
    switch (event.type) {
      case 'payment_intent.succeeded':
        await handlePaymentSuccess(event.data);
        break;
      case 'payment_intent.failed':
        await handlePaymentFailure(event.data);
        break;
      default:
        console.log(`Unhandled event type: ${event.type}`);
    }
    
    return NextResponse.json({ received: true });
  } catch (error) {
    return NextResponse.json(
      { error: 'Webhook verification failed' },
      { status: 400 }
    );
  }
}

function verifyStripeWebhook(body: string, signature: string) {
  // Stripe webhook verification logic
  return { type: 'payment_intent.succeeded', data: {} };
}

async function handlePaymentSuccess(data: any) {
  console.log('Payment succeeded:', data);
}

async function handlePaymentFailure(data: any) {
  console.log('Payment failed:', data);
}
```

## Common Mistakes

### 1. Not Handling Errors

```typescript
// WRONG: No error handling
export async function GET() {
  const data = await fetch('https://api.example.com/data').then(res => res.json());
  return NextResponse.json(data);
}

// CORRECT: Proper error handling
export async function GET() {
  try {
    const response = await fetch('https://api.example.com/data');
    
    if (!response.ok) {
      return NextResponse.json(
        { error: 'Failed to fetch data' },
        { status: response.status }
      );
    }
    
    const data = await response.json();
    return NextResponse.json(data);
  } catch (error) {
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    );
  }
}
```

### 2. Using Node.js APIs in Edge Runtime

```typescript
// WRONG: Using fs in Edge runtime
export const runtime = 'edge';

export async function GET() {
  const fs = require('fs'); // Error!
  const data = fs.readFileSync('data.json');
  return NextResponse.json(data);
}

// CORRECT: Use Node.js runtime for fs
export async function GET() {
  const fs = require('fs');
  const data = fs.readFileSync('data.json');
  return NextResponse.json(data);
}
```

### 3. Not Setting Proper Content-Type

```typescript
// WRONG: Returning text without content-type
export async function GET() {
  return new Response('Hello');
}

// CORRECT: Set content-type
export async function GET() {
  return new Response('Hello', {
    headers: { 'Content-Type': 'text/plain' },
  });
}
```

## Best Practices

### 1. Use Appropriate Runtime

```typescript
// Edge for simple, fast responses
export const runtime = 'edge';

export async function GET() {
  return NextResponse.json({ status: 'ok' });
}

// Node.js for complex operations
export async function POST(request: NextRequest) {
  const formData = await request.formData();
  // Process file upload
}
```

### 2. Validate Input

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

export async function POST(request: NextRequest) {
  const body = await request.json();
  
  const result = schema.safeParse(body);
  
  if (!result.success) {
    return NextResponse.json(
      { error: result.error.flatten() },
      { status: 400 }
    );
  }
  
  // Process valid data
  const { email, password } = result.data;
  // ...
}
```

### 3. Use Middleware for Common Logic

```typescript
// middleware.ts
export function middleware(request: NextRequest) {
  // Authentication, logging, etc.
}

// route.ts - keep handlers focused
export async function GET() {
  // Business logic only
}
```

## Interview Questions

### Q1: What's the difference between Route Handlers and API Routes?

**Answer:** Route Handlers are the App Router's replacement for API Routes. They use Web APIs (Request/Response) instead of Node.js APIs (req/res), support streaming, and can run on Edge runtime for faster cold starts.

### Q2: When should you use Edge runtime vs Node.js runtime?

**Answer:** Use Edge runtime for simple, fast responses that don't need Node.js APIs (geolocation, A/B testing, redirects). Use Node.js runtime for file system access, native modules, database connections, or complex operations.

### Q3: How do you handle authentication in route handlers?

**Answer:** Use middleware to check authentication for protected routes. Middleware can read cookies/headers, verify tokens, and return 401 responses or redirect to login. Individual route handlers can also check authentication if needed.

### Q4: How do you implement streaming responses?

**Answer:** Create a ReadableStream and return it in a Response. Use a controller to enqueue chunks of data. Common for SSE, large datasets, or AI-generated content.

### Q5: What are the benefits of using NextRequest/NextResponse?

**Answer:** They extend the standard Request/Response APIs with convenient methods for cookies, headers, URL manipulation, and Next.js-specific features like rewrites and redirects.

## Key Takeaways

1. Route handlers use `route.ts` files in the app directory
2. Support all HTTP methods: GET, POST, PUT, PATCH, DELETE, etc.
3. NextRequest provides cookies, geo, IP, and enhanced URL access
4. NextResponse simplifies setting cookies, headers, and responses
5. Middleware runs before handlers for cross-cutting concerns
6. Edge runtime is fast but limited; Node.js runtime has full APIs
7. Streaming enables progressive responses and SSE
8. Always handle errors and validate input
9. Use appropriate runtime based on requirements
10. Middleware is ideal for authentication, logging, and rate limiting

## Resources

- [Next.js Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)
- [Next.js Middleware](https://nextjs.org/docs/app/building-your-application/routing/middleware)
- [Edge Runtime](https://nextjs.org/docs/app/api-reference/edge)
- [Web APIs](https://developer.mozilla.org/en-US/docs/Web/API)
