# Mocking with MSW (Mock Service Worker)

## The Idea

**In plain English:** MSW (Mock Service Worker) is a tool that pretends to be a real server during testing, so your app can make network requests and get back fake responses — without ever hitting the internet. A "mock" means a fake stand-in, and a "service worker" is a small background program the browser runs alongside your app.

**Real-world analogy:** Imagine a flight simulator used to train pilots. The cockpit looks and behaves exactly like a real plane, but the "sky," turbulence, and emergencies are all fake and controlled by the instructor.

- The simulator cockpit = your React app making real fetch/API calls
- The instructor controlling fake weather and responses = MSW intercepting requests and returning pre-set data
- The pilot not knowing it's fake = your tests running just like they would against a real server

---

## Overview

Mock Service Worker (MSW) is a modern API mocking library that intercepts requests at the network level. Unlike traditional mocking that patches fetch/axios, MSW works by intercepting requests in a Service Worker (browser) or Node.js server, providing network-agnostic, realistic API mocking.

## Table of Contents

1. [MSW Setup](#msw-setup)
2. [REST API Mocking](#rest-api-mocking)
3. [Testing Loading States](#testing-loading-states)
4. [Testing Error States](#testing-error-states)
5. [GraphQL Mocking](#graphql-mocking)
6. [Network-Agnostic Tests](#network-agnostic-tests)
7. [Common Mistakes](#common-mistakes)
8. [Best Practices](#best-practices)
9. [Interview Questions](#interview-questions)

## MSW Setup

### Installation

```bash
npm install msw --save-dev
```

### Node.js Setup (Tests)

```typescript
// src/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
```

```typescript
// src/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('/api/users', () => {
    return HttpResponse.json([
      { id: 1, name: 'John Doe' },
      { id: 2, name: 'Jane Smith' },
    ]);
  }),
];
```

```typescript
// src/setupTests.ts
import { server } from './mocks/server';

// Establish API mocking before all tests
beforeAll(() => server.listen());

// Reset handlers after each test
afterEach(() => server.resetHandlers());

// Clean up after all tests
afterAll(() => server.close());
```

### Browser Setup (Development)

```typescript
// src/mocks/browser.ts
import { setupWorker } from 'msw/browser';
import { handlers } from './handlers';

export const worker = setupWorker(...handlers);
```

```typescript
// src/main.tsx
if (process.env.NODE_ENV === 'development') {
  const { worker } = await import('./mocks/browser');
  worker.start();
}
```

## REST API Mocking

### Basic GET Handler

```typescript
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';

const server = setupServer(
  http.get('/api/posts', () => {
    return HttpResponse.json([
      { id: 1, title: 'First Post' },
      { id: 2, title: 'Second Post' },
    ]);
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test('fetches posts', async () => {
  render(<PostList />);
  
  expect(await screen.findByText('First Post')).toBeInTheDocument();
  expect(screen.getByText('Second Post')).toBeInTheDocument();
});
```

### POST Handler

```typescript
http.post('/api/posts', async ({ request }) => {
  const newPost = await request.json();
  
  return HttpResponse.json(
    { id: 3, ...newPost },
    { status: 201 }
  );
});

test('creates new post', async () => {
  const user = userEvent.setup();
  render(<CreatePost />);
  
  await user.type(screen.getByLabelText('Title'), 'New Post');
  await user.type(screen.getByLabelText('Content'), 'Post content');
  await user.click(screen.getByRole('button', { name: 'Create' }));
  
  expect(await screen.findByText('Post created!')).toBeInTheDocument();
});
```

### Dynamic Route Parameters

```typescript
http.get('/api/users/:id', ({ params }) => {
  const { id } = params;
  
  const users = {
    '1': { id: '1', name: 'John Doe', email: 'john@example.com' },
    '2': { id: '2', name: 'Jane Smith', email: 'jane@example.com' },
  };
  
  const user = users[id as keyof typeof users];
  
  if (!user) {
    return new HttpResponse(null, { status: 404 });
  }
  
  return HttpResponse.json(user);
});

test('shows user profile', async () => {
  render(<UserProfile userId="1" />);
  
  expect(await screen.findByText('John Doe')).toBeInTheDocument();
  expect(screen.getByText('john@example.com')).toBeInTheDocument();
});

test('shows 404 for invalid user', async () => {
  render(<UserProfile userId="999" />);
  
  expect(await screen.findByText('User not found')).toBeInTheDocument();
});
```

### Query Parameters

```typescript
http.get('/api/search', ({ request }) => {
  const url = new URL(request.url);
  const query = url.searchParams.get('q');
  const page = url.searchParams.get('page') || '1';
  
  return HttpResponse.json({
    results: [
      { id: 1, title: `Result for "${query}"` },
    ],
    page: parseInt(page, 10),
    total: 1,
  });
});

test('searches with query params', async () => {
  render(<Search />);
  
  const input = screen.getByRole('textbox');
  await userEvent.type(input, 'react');
  await userEvent.click(screen.getByRole('button', { name: 'Search' }));
  
  expect(await screen.findByText('Result for "react"')).toBeInTheDocument();
});
```

## Testing Loading States

### Delayed Response

```typescript
import { delay, http, HttpResponse } from 'msw';

http.get('/api/data', async () => {
  await delay(2000); // 2 second delay
  
  return HttpResponse.json({ data: 'Loaded' });
});

test('shows loading state', async () => {
  render(<DataComponent />);
  
  // Loading state appears
  expect(screen.getByText('Loading...')).toBeInTheDocument();
  
  // Wait for data
  expect(await screen.findByText('Loaded')).toBeInTheDocument();
  
  // Loading state disappears
  expect(screen.queryByText('Loading...')).not.toBeInTheDocument();
});
```

### Infinite Loading (Timeout)

```typescript
http.get('/api/slow', () => delay('infinite'));

test('handles timeout', async () => {
  render(<SlowComponent />);
  
  expect(screen.getByText('Loading...')).toBeInTheDocument();
  
  // After timeout, should show error
  expect(await screen.findByText('Request timed out', {}, { timeout: 6000 }))
    .toBeInTheDocument();
});
```

## Testing Error States

### 404 Not Found

```typescript
http.get('/api/post/:id', ({ params }) => {
  const { id } = params;
  
  if (id === '999') {
    return new HttpResponse(null, { status: 404 });
  }
  
  return HttpResponse.json({ id, title: 'Post' });
});

test('shows 404 error', async () => {
  render(<Post id="999" />);
  
  expect(await screen.findByText('Post not found')).toBeInTheDocument();
});
```

### 500 Server Error

```typescript
http.get('/api/data', () => {
  return HttpResponse.json(
    { error: 'Internal Server Error' },
    { status: 500 }
  );
});

test('handles server error', async () => {
  render(<DataComponent />);
  
  expect(await screen.findByText('Failed to load data')).toBeInTheDocument();
});
```

### Network Error

```typescript
import { HttpResponse } from 'msw';

http.get('/api/data', () => {
  return HttpResponse.error();
});

test('handles network error', async () => {
  render(<DataComponent />);
  
  expect(await screen.findByText('Network error. Please check your connection.'))
    .toBeInTheDocument();
});
```

### Runtime Handler Override

```typescript
test('handles error after initial success', async () => {
  const { rerender } = render(<DataComponent />);
  
  // First request succeeds
  expect(await screen.findByText('Data loaded')).toBeInTheDocument();
  
  // Override handler for next request
  server.use(
    http.get('/api/data', () => {
      return HttpResponse.json({ error: 'Error' }, { status: 500 });
    })
  );
  
  // Trigger refetch
  await userEvent.click(screen.getByRole('button', { name: 'Refresh' }));
  
  // Now shows error
  expect(await screen.findByText('Failed to load data')).toBeInTheDocument();
});
```

## GraphQL Mocking

### GraphQL Setup

```typescript
import { graphql, HttpResponse } from 'msw';

const server = setupServer(
  graphql.query('GetUser', ({ query, variables }) => {
    return HttpResponse.json({
      data: {
        user: {
          id: variables.id,
          name: 'John Doe',
          email: 'john@example.com',
        },
      },
    });
  }),
  
  graphql.mutation('CreatePost', ({ query, variables }) => {
    return HttpResponse.json({
      data: {
        createPost: {
          id: '1',
          title: variables.title,
          content: variables.content,
        },
      },
    });
  })
);
```

### GraphQL Query Test

```typescript
test('fetches user via GraphQL', async () => {
  render(<UserProfile userId="1" />);
  
  expect(await screen.findByText('John Doe')).toBeInTheDocument();
  expect(screen.getByText('john@example.com')).toBeInTheDocument();
});
```

### GraphQL Mutation Test

```typescript
test('creates post via GraphQL mutation', async () => {
  const user = userEvent.setup();
  render(<CreatePost />);
  
  await user.type(screen.getByLabelText('Title'), 'New Post');
  await user.type(screen.getByLabelText('Content'), 'Content here');
  await user.click(screen.getByRole('button', { name: 'Create' }));
  
  expect(await screen.findByText('Post created!')).toBeInTheDocument();
});
```

### GraphQL Errors

```typescript
graphql.query('GetUser', () => {
  return HttpResponse.json({
    errors: [
      {
        message: 'User not found',
        extensions: { code: 'NOT_FOUND' },
      },
    ],
  });
});

test('handles GraphQL errors', async () => {
  render(<UserProfile userId="999" />);
  
  expect(await screen.findByText('User not found')).toBeInTheDocument();
});
```

## Network-Agnostic Tests

### Works with Any HTTP Client

```typescript
// Works with fetch
async function fetchUsers() {
  const res = await fetch('/api/users');
  return res.json();
}

// Works with axios
import axios from 'axios';
async function fetchUsersAxios() {
  const { data } = await axios.get('/api/users');
  return data;
}

// Works with react-query
function useUsers() {
  return useQuery({
    queryKey: ['users'],
    queryFn: () => fetch('/api/users').then(r => r.json()),
  });
}

// Same MSW handler works for all!
http.get('/api/users', () => {
  return HttpResponse.json([
    { id: 1, name: 'John' },
  ]);
});
```

### No Mocking Dependencies

```typescript
// ❌ Old way: Mock axios
jest.mock('axios');

test('old way', async () => {
  (axios.get as jest.Mock).mockResolvedValue({ data: [] });
  // Test...
});

// ✅ New way: MSW intercepts at network level
test('new way', async () => {
  // MSW handler already set up
  // No mocking needed!
  render(<Component />);
  // Test...
});
```

## Common Mistakes

### 1. Not Resetting Handlers

```typescript
// ❌ WRONG: Handlers leak between tests
afterEach(() => {
  // Missing server.resetHandlers()
});

// ✅ CORRECT: Reset after each test
afterEach(() => {
  server.resetHandlers();
});
```

### 2. Using Wrong HTTP Method

```typescript
// ❌ WRONG: Component uses POST, handler uses GET
http.get('/api/login', () => { /* ... */ });

// Component does:
// fetch('/api/login', { method: 'POST' })

// ✅ CORRECT: Match HTTP method
http.post('/api/login', () => { /* ... */ });
```

### 3. Not Awaiting Async Tests

```typescript
// ❌ WRONG: Not waiting for async update
test('wrong', () => {
  render(<AsyncComponent />);
  expect(screen.getByText('Loaded')).toBeInTheDocument(); // Fails!
});

// ✅ CORRECT: Wait for element
test('correct', async () => {
  render(<AsyncComponent />);
  expect(await screen.findByText('Loaded')).toBeInTheDocument();
});
```

## Best Practices

### 1. Centralize Handlers

```typescript
// src/mocks/handlers.ts
export const handlers = [
  http.get('/api/users', () => { /* ... */ }),
  http.get('/api/posts', () => { /* ... */ }),
];

// src/mocks/server.ts
export const server = setupServer(...handlers);
```

### 2. Use Realistic Data

```typescript
// ✅ GOOD: Realistic response
http.get('/api/users', () => {
  return HttpResponse.json({
    users: [
      { id: 1, name: 'John Doe', email: 'john@example.com', role: 'admin' },
    ],
    total: 1,
    page: 1,
    perPage: 10,
  });
});
```

### 3. Test All States

```typescript
test('success state', async () => { /* ... */ });
test('loading state', async () => { /* ... */ });
test('error state', async () => { /* ... */ });
test('empty state', async () => { /* ... */ });
```

## Interview Questions

### Q1: What is MSW and how does it work?

**Answer:** MSW intercepts requests at the network level using a Service Worker (browser) or Node.js server. It doesn't patch fetch/axios, making tests network-agnostic and more realistic.

### Q2: What are the advantages of MSW over mocking fetch/axios?

**Answer:** MSW is network-agnostic (works with any HTTP client), doesn't require mocking dependencies, works in both tests and browser, and provides more realistic mocking since it intercepts actual network requests.

### Q3: How do you test error states with MSW?

**Answer:** Return HttpResponse with error status codes (404, 500), use HttpResponse.error() for network errors, or return GraphQL errors in the response body.

### Q4: Why should you call server.resetHandlers() after each test?

**Answer:** To prevent handler overrides from leaking between tests. If one test overrides a handler with server.use(), resetHandlers() restores the original handlers.

### Q5: Can MSW mock GraphQL APIs?

**Answer:** Yes, MSW provides graphql.query() and graphql.mutation() handlers for mocking GraphQL operations, supporting both queries and mutations with full error handling.

## Key Takeaways

1. MSW intercepts requests at the network level
2. Works with any HTTP client (fetch, axios, react-query)
3. No need to mock fetch/axios in tests
4. Use delay() to test loading states
5. Return error status codes to test error states
6. Always call server.resetHandlers() after each test
7. Supports both REST and GraphQL APIs
8. Override handlers per-test with server.use()
9. Works in both Node (tests) and browser (development)
10. More realistic than traditional mocking approaches

## Resources

- [MSW Documentation](https://mswjs.io)
- [MSW Examples](https://github.com/mswjs/examples)
- [Testing with MSW](https://kentcdodds.com/blog/stop-mocking-fetch)
- [MSW Best Practices](https://mswjs.io/docs/best-practices)
