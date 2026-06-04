# MSW (Mock Service Worker)

## Overview

Mock Service Worker (MSW) is an API mocking library that uses Service Worker API to intercept requests on the network level. It enables consistent API mocking across development, testing, and debugging environments. MSW works in both browser and Node.js, providing a seamless mocking experience without changing application code.

## Installation and Setup

```bash
# Install MSW
npm install -D msw

# Initialize browser service worker
npx msw init public/ --save
```

### Setup for Testing (Node.js)

```javascript
// src/mocks/server.js
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
```

```javascript
// src/test/setup.js
import { beforeAll, afterEach, afterAll } from 'vitest';
import { server } from '../mocks/server';

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

### Setup for Browser Development

```javascript
// src/mocks/browser.js
import { setupWorker } from 'msw/browser';
import { handlers } from './handlers';

export const worker = setupWorker(...handlers);
```

```javascript
// src/main.jsx (Development only)
import { worker } from './mocks/browser';

if (import.meta.env.DEV) {
  worker.start({
    onUnhandledRequest: 'bypass',
  });
}
```

## Core Concepts

### Request Handlers

```javascript
// src/mocks/handlers.js
import { http, HttpResponse } from 'msw';

export const handlers = [
  // GET request
  http.get('/api/users', () => {
    return HttpResponse.json([
      { id: 1, name: 'John Doe', email: 'john@example.com' },
      { id: 2, name: 'Jane Doe', email: 'jane@example.com' },
    ]);
  }),

  // GET request with params
  http.get('/api/users/:id', ({ params }) => {
    const { id } = params;
    return HttpResponse.json({
      id: Number(id),
      name: 'John Doe',
      email: 'john@example.com',
    });
  }),

  // POST request
  http.post('/api/users', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json(
      {
        id: 3,
        ...body,
      },
      { status: 201 }
    );
  }),

  // PUT request
  http.put('/api/users/:id', async ({ params, request }) => {
    const { id } = params;
    const body = await request.json();
    return HttpResponse.json({
      id: Number(id),
      ...body,
    });
  }),

  // DELETE request
  http.delete('/api/users/:id', ({ params }) => {
    return new HttpResponse(null, { status: 204 });
  }),

  // PATCH request
  http.patch('/api/users/:id', async ({ params, request }) => {
    const { id } = params;
    const body = await request.json();
    return HttpResponse.json({
      id: Number(id),
      ...body,
    });
  }),
];
```

### Response Types

```javascript
import { http, HttpResponse } from 'msw';

export const handlers = [
  // JSON response
  http.get('/api/data', () => {
    return HttpResponse.json({ message: 'Success' });
  }),

  // Text response
  http.get('/api/text', () => {
    return HttpResponse.text('Plain text response');
  }),

  // XML response
  http.get('/api/xml', () => {
    return HttpResponse.xml('<root><message>Hello</message></root>');
  }),

  // Custom response
  http.get('/api/custom', () => {
    return new HttpResponse('Custom body', {
      status: 200,
      statusText: 'OK',
      headers: {
        'Content-Type': 'application/json',
        'X-Custom-Header': 'value',
      },
    });
  }),

  // Binary response
  http.get('/api/file', () => {
    const buffer = new ArrayBuffer(8);
    return HttpResponse.arrayBuffer(buffer);
  }),

  // Redirect response
  http.get('/api/redirect', () => {
    return HttpResponse.redirect('/api/new-location', 302);
  }),

  // Empty response
  http.delete('/api/users/:id', () => {
    return new HttpResponse(null, { status: 204 });
  }),
];
```

### Request Matching

```javascript
import { http, HttpResponse } from 'msw';

export const handlers = [
  // Query parameters
  http.get('/api/users', ({ request }) => {
    const url = new URL(request.url);
    const page = url.searchParams.get('page');
    const limit = url.searchParams.get('limit');

    return HttpResponse.json({
      page: Number(page) || 1,
      limit: Number(limit) || 10,
      users: [/* ... */],
    });
  }),

  // Request headers
  http.get('/api/protected', ({ request }) => {
    const auth = request.headers.get('Authorization');

    if (!auth || !auth.startsWith('Bearer ')) {
      return HttpResponse.json(
        { error: 'Unauthorized' },
        { status: 401 }
      );
    }

    return HttpResponse.json({ data: 'Protected data' });
  }),

  // Request body
  http.post('/api/login', async ({ request }) => {
    const { email, password } = await request.json();

    if (email === 'admin@test.com' && password === 'admin123') {
      return HttpResponse.json({
        token: 'mock-jwt-token',
        user: { email, role: 'admin' },
      });
    }

    return HttpResponse.json(
      { error: 'Invalid credentials' },
      { status: 401 }
    );
  }),

  // Cookies
  http.get('/api/session', ({ cookies }) => {
    const sessionId = cookies.get('sessionId');

    if (!sessionId) {
      return HttpResponse.json(
        { error: 'No session' },
        { status: 401 }
      );
    }

    return HttpResponse.json({ sessionId });
  }),
];
```

## Advanced Patterns

### Conditional Responses

```javascript
import { http, HttpResponse, delay } from 'msw';

export const handlers = [
  // Delayed response
  http.get('/api/slow', async () => {
    await delay(2000); // 2 second delay
    return HttpResponse.json({ data: 'Slow response' });
  }),

  // Random delay
  http.get('/api/variable', async () => {
    await delay(); // Random delay
    return HttpResponse.json({ data: 'Variable response' });
  }),

  // Conditional status
  http.get('/api/flaky', ({ request }) => {
    const shouldFail = Math.random() > 0.5;

    if (shouldFail) {
      return HttpResponse.json(
        { error: 'Server error' },
        { status: 500 }
      );
    }

    return HttpResponse.json({ data: 'Success' });
  }),

  // Request count based
  (() => {
    let requestCount = 0;

    return http.get('/api/retry', () => {
      requestCount++;

      if (requestCount < 3) {
        return HttpResponse.json(
          { error: 'Temporary error' },
          { status: 503 }
        );
      }

      return HttpResponse.json({ data: 'Success after retries' });
    });
  })(),
];
```

### Dynamic Responses

```javascript
import { http, HttpResponse } from 'msw';

// In-memory database
const users = [
  { id: 1, name: 'John Doe', email: 'john@example.com' },
  { id: 2, name: 'Jane Doe', email: 'jane@example.com' },
];

let nextId = 3;

export const handlers = [
  // List with pagination
  http.get('/api/users', ({ request }) => {
    const url = new URL(request.url);
    const page = Number(url.searchParams.get('page')) || 1;
    const limit = Number(url.searchParams.get('limit')) || 10;

    const start = (page - 1) * limit;
    const end = start + limit;
    const paginatedUsers = users.slice(start, end);

    return HttpResponse.json({
      users: paginatedUsers,
      total: users.length,
      page,
      limit,
    });
  }),

  // Create
  http.post('/api/users', async ({ request }) => {
    const body = await request.json();
    const newUser = { id: nextId++, ...body };
    users.push(newUser);

    return HttpResponse.json(newUser, { status: 201 });
  }),

  // Read
  http.get('/api/users/:id', ({ params }) => {
    const { id } = params;
    const user = users.find((u) => u.id === Number(id));

    if (!user) {
      return HttpResponse.json(
        { error: 'User not found' },
        { status: 404 }
      );
    }

    return HttpResponse.json(user);
  }),

  // Update
  http.put('/api/users/:id', async ({ params, request }) => {
    const { id } = params;
    const body = await request.json();
    const index = users.findIndex((u) => u.id === Number(id));

    if (index === -1) {
      return HttpResponse.json(
        { error: 'User not found' },
        { status: 404 }
      );
    }

    users[index] = { ...users[index], ...body };
    return HttpResponse.json(users[index]);
  }),

  // Delete
  http.delete('/api/users/:id', ({ params }) => {
    const { id } = params;
    const index = users.findIndex((u) => u.id === Number(id));

    if (index === -1) {
      return HttpResponse.json(
        { error: 'User not found' },
        { status: 404 }
      );
    }

    users.splice(index, 1);
    return new HttpResponse(null, { status: 204 });
  }),
];
```

### GraphQL Mocking

```javascript
import { graphql, HttpResponse } from 'msw';

export const handlers = [
  // Query
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

  // Mutation
  graphql.mutation('CreateUser', ({ query, variables }) => {
    return HttpResponse.json({
      data: {
        createUser: {
          id: 3,
          ...variables.input,
        },
      },
    });
  }),

  // Error response
  graphql.query('GetPosts', () => {
    return HttpResponse.json({
      errors: [
        {
          message: 'Not authenticated',
          extensions: { code: 'UNAUTHENTICATED' },
        },
      ],
    });
  }),
];
```

## Testing with MSW

### Basic Test Example

```typescript
import { render, screen, waitFor } from '@testing-library/react';
import { server } from './mocks/server';
import { http, HttpResponse } from 'msw';
import { UserList } from './UserList';

describe('UserList', () => {
  it('displays users from API', async () => {
    render(<UserList />);

    expect(screen.getByText('Loading...')).toBeInTheDocument();

    expect(await screen.findByText('John Doe')).toBeInTheDocument();
    expect(await screen.findByText('Jane Doe')).toBeInTheDocument();
  });

  it('displays error message on failure', async () => {
    server.use(
      http.get('/api/users', () => {
        return HttpResponse.json(
          { error: 'Server error' },
          { status: 500 }
        );
      })
    );

    render(<UserList />);

    expect(await screen.findByText(/failed to load/i)).toBeInTheDocument();
  });

  it('handles empty user list', async () => {
    server.use(
      http.get('/api/users', () => {
        return HttpResponse.json([]);
      })
    );

    render(<UserList />);

    expect(await screen.findByText('No users found')).toBeInTheDocument();
  });
});
```

### Runtime Handler Override

```typescript
import { server } from './mocks/server';
import { http, HttpResponse } from 'msw';

describe('Runtime overrides', () => {
  it('overrides handler for single test', async () => {
    server.use(
      http.get('/api/users', () => {
        return HttpResponse.json([
          { id: 999, name: 'Test User', email: 'test@example.com' },
        ]);
      })
    );

    // Test with overridden handler
  });

  it('resets to default handlers', async () => {
    // Handler is reset after previous test
    // Uses original handler from handlers.js
  });
});
```

### Test Scenarios

```typescript
import { server } from './mocks/server';
import { http, HttpResponse, delay } from 'msw';

describe('API scenarios', () => {
  it('handles slow network', async () => {
    server.use(
      http.get('/api/users', async () => {
        await delay(3000);
        return HttpResponse.json([]);
      })
    );

    render(<UserList />);
    expect(screen.getByText('Loading...')).toBeInTheDocument();
  });

  it('handles network error', async () => {
    server.use(
      http.get('/api/users', () => {
        return HttpResponse.error();
      })
    );

    render(<UserList />);
    expect(await screen.findByText(/network error/i)).toBeInTheDocument();
  });

  it('handles timeout', async () => {
    server.use(
      http.get('/api/users', async () => {
        await delay('infinite');
      })
    );

    render(<UserList />);
    // Test timeout handling
  });
});
```

## Real-World Patterns

### Authentication Flow

```javascript
import { http, HttpResponse } from 'msw';

let authToken = null;

export const handlers = [
  // Login
  http.post('/api/auth/login', async ({ request }) => {
    const { email, password } = await request.json();

    if (email === 'admin@test.com' && password === 'admin123') {
      authToken = 'mock-jwt-token-' + Date.now();
      return HttpResponse.json({
        token: authToken,
        user: { email, role: 'admin' },
      });
    }

    return HttpResponse.json(
      { error: 'Invalid credentials' },
      { status: 401 }
    );
  }),

  // Protected endpoint
  http.get('/api/profile', ({ request }) => {
    const auth = request.headers.get('Authorization');

    if (!auth || auth !== `Bearer ${authToken}`) {
      return HttpResponse.json(
        { error: 'Unauthorized' },
        { status: 401 }
      );
    }

    return HttpResponse.json({
      email: 'admin@test.com',
      name: 'Admin User',
      role: 'admin',
    });
  }),

  // Logout
  http.post('/api/auth/logout', () => {
    authToken = null;
    return new HttpResponse(null, { status: 204 });
  }),
];
```

### File Upload

```javascript
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.post('/api/upload', async ({ request }) => {
    const formData = await request.formData();
    const file = formData.get('file');

    if (!file) {
      return HttpResponse.json(
        { error: 'No file provided' },
        { status: 400 }
      );
    }

    return HttpResponse.json({
      id: 'file-123',
      name: file.name,
      size: file.size,
      type: file.type,
      url: `https://example.com/files/file-123`,
    });
  }),
];
```

### Pagination and Filtering

```javascript
import { http, HttpResponse } from 'msw';

const products = Array.from({ length: 100 }, (_, i) => ({
  id: i + 1,
  name: `Product ${i + 1}`,
  price: Math.random() * 1000,
  category: ['Electronics', 'Clothing', 'Food'][i % 3],
}));

export const handlers = [
  http.get('/api/products', ({ request }) => {
    const url = new URL(request.url);
    const page = Number(url.searchParams.get('page')) || 1;
    const limit = Number(url.searchParams.get('limit')) || 20;
    const category = url.searchParams.get('category');
    const sort = url.searchParams.get('sort');

    let filtered = category
      ? products.filter((p) => p.category === category)
      : products;

    if (sort === 'price-asc') {
      filtered.sort((a, b) => a.price - b.price);
    } else if (sort === 'price-desc') {
      filtered.sort((a, b) => b.price - a.price);
    }

    const start = (page - 1) * limit;
    const end = start + limit;
    const paginated = filtered.slice(start, end);

    return HttpResponse.json({
      products: paginated,
      total: filtered.length,
      page,
      limit,
      hasMore: end < filtered.length,
    });
  }),
];
```

## Common Mistakes

1. **Not resetting handlers**: Forgetting to reset handlers between tests causing pollution.

2. **Improper handler order**: More specific handlers should come before generic ones.

3. **Not awaiting responses**: Missing async/await in request handlers.

4. **Hardcoding URLs**: Not using baseURL or environment variables.

5. **Over-mocking**: Mocking too much, missing integration issues.

6. **Not testing error states**: Only testing success paths.

7. **Ignoring request validation**: Not validating request body/headers in handlers.

8. **Memory leaks**: Not cleaning up in-memory data between tests.

9. **Synchronous delays**: Using `setTimeout` instead of MSW's `delay`.

10. **Not handling unhandled requests**: Missing configuration for unexpected requests.

## Best Practices

1. **Organize handlers**: Group related handlers by domain/feature.

2. **Use TypeScript**: Type request/response for better safety.

3. **Reset between tests**: Use `server.resetHandlers()` in `afterEach`.

4. **Test error scenarios**: Mock failures, timeouts, network errors.

5. **Validate requests**: Check request body, headers, params in handlers.

6. **Use realistic data**: Mock responses that match actual API structure.

7. **Leverage delays**: Test loading states with `delay()`.

8. **Override for specific tests**: Use `server.use()` for test-specific mocking.

9. **Configure unhandled requests**: Set appropriate `onUnhandledRequest` behavior.

10. **Document scenarios**: Comment complex handler logic and scenarios.

## When to Use MSW

**Use MSW when:**
- Need consistent mocking across environments
- Want network-level interception
- Building comprehensive test suites
- Developing without backend
- Need realistic API mocking
- Want to mock third-party APIs
- Testing error scenarios

**Consider alternatives when:**
- Need simple function mocks (use vi.fn/jest.fn)
- Want component-level mocking
- Building unit tests only
- Need performance testing
- Want to test actual API

## Interview Questions

### Q1: How does MSW work under the hood?

**Answer**: MSW uses Service Worker API in browsers and http/https module interception in Node.js to intercept network requests at the network level. In browsers, it registers a service worker that intercepts fetch requests before they reach the network. In Node.js, it patches the native http modules. This allows MSW to mock requests without modifying application code or importing mock utilities in components.

### Q2: What's the advantage of MSW over traditional mocking?

**Answer**: MSW intercepts requests at the network level, meaning: 1) No changes to application code, 2) Works with any request library (fetch, axios, etc.), 3) Same mocks work in tests and browser development, 4) More realistic than function mocks, 5) Tests actual request/response cycle, 6) Supports all HTTP methods and features like headers, cookies, redirects.

### Q3: How do you override handlers for specific tests?

**Answer**: Use `server.use()` to override handlers for specific tests:

```javascript
test('overrides handler', () => {
  server.use(
    http.get('/api/users', () => {
      return HttpResponse.json({ error: 'Error' }, { status: 500 });
    })
  );
  // Test with overridden handler
});
// Handler resets in afterEach with server.resetHandlers()
```

### Q4: How do you test authentication with MSW?

**Answer**: Store auth state in handler closure and validate in protected endpoints:

```javascript
let token = null;

http.post('/api/login', async ({ request }) => {
  // Validate credentials, set token
  token = 'mock-token';
  return HttpResponse.json({ token });
});

http.get('/api/protected', ({ request }) => {
  if (request.headers.get('Authorization') !== `Bearer ${token}`) {
    return HttpResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }
  return HttpResponse.json({ data: 'protected' });
});
```

### Q5: How do you simulate slow network with MSW?

**Answer**: Use `delay()` function:

```javascript
import { http, HttpResponse, delay } from 'msw';

http.get('/api/data', async () => {
  await delay(2000); // Fixed 2s delay
  // Or await delay() for random delay
  return HttpResponse.json({ data: 'slow response' });
});
```

### Q6: How do you test GraphQL APIs with MSW?

**Answer**: Use GraphQL utilities from MSW:

```javascript
import { graphql, HttpResponse } from 'msw';

graphql.query('GetUser', ({ query, variables }) => {
  return HttpResponse.json({
    data: { user: { id: variables.id, name: 'John' } },
  });
});

graphql.mutation('CreateUser', ({ variables }) => {
  return HttpResponse.json({
    data: { createUser: { id: 1, ...variables.input } },
  });
});
```

### Q7: How do you handle dynamic data and stateful mocking?

**Answer**: Create in-memory data store within handlers:

```javascript
const users = [];
let nextId = 1;

http.post('/api/users', async ({ request }) => {
  const user = { id: nextId++, ...await request.json() };
  users.push(user);
  return HttpResponse.json(user, { status: 201 });
});

http.get('/api/users', () => {
  return HttpResponse.json(users);
});
```

Remember to reset state in test setup/teardown.

## Key Takeaways

1. MSW intercepts network requests at the network level using Service Workers (browser) and module patches (Node.js).

2. Handlers define request patterns and responses using http/graphql utilities with full HTTP feature support.

3. Setup requires server configuration for tests and worker configuration for browser development.

4. Runtime handler override with `server.use()` enables test-specific mocking without affecting other tests.

5. Realistic testing includes delays (`delay()`), error responses, and network failures for comprehensive coverage.

6. Stateful mocking uses in-memory data structures within handler closures for dynamic CRUD operations.

7. GraphQL support through dedicated utilities handles queries, mutations, and subscriptions.

8. Authentication flows store tokens in handler scope and validate headers in protected endpoints.

9. Request validation in handlers ensures application sends correct data structure and headers.

10. MSW provides consistent mocking across development, testing, and debugging without changing application code.

## Resources

- [Official Documentation](https://mswjs.io/)
- [Getting Started](https://mswjs.io/docs/getting-started)
- [API Reference](https://mswjs.io/docs/api)
- [Recipes](https://mswjs.io/docs/recipes)
- [GitHub Repository](https://github.com/mswjs/msw)
- [Examples](https://github.com/mswjs/examples)
