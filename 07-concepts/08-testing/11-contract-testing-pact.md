# Contract Testing and Consumer-Driven Contracts (Pact)

## Overview

Contract testing verifies that two services (a consumer and a provider) can communicate correctly — without requiring both to be running at the same time. Consumer-Driven Contract (CDC) testing inverts the typical approach: the **consumer** defines the contract (what it expects from the provider), and the **provider** verifies it can meet that contract. This is the pattern Pact implements.

At staff/principal engineer level, contract testing is a standard tool for scaling teams — enabling multiple frontend and backend teams to evolve their APIs independently without breaking each other.

---

## Why Contract Testing Exists

### The Problem at Scale

With multiple frontend teams (web, mobile, BFF) consuming a single backend:

```
Web Frontend  ──→ ┐
Mobile App    ──→ ├──→ Orders API  ──→ Database
Admin Panel   ──→ ┘
```

How do you know a backend API change won't break the web frontend? Options:

1. **Integration tests against a running environment** — slow, expensive, environment stability issues
2. **End-to-end tests** — very slow, brittle, catches issues late
3. **Hope and manual coordination** — doesn't scale
4. **Contract testing** — fast, runs in CI, catches breaking changes before deployment

### Contract vs Integration vs E2E

| Test type | Scope | Speed | Environment needed | Catches |
|---|---|---|---|---|
| Unit | One function/component | <1ms | None | Logic bugs |
| Contract | API boundary (consumer ↔ provider) | <100ms | None | Breaking API changes |
| Integration | Two real services talking | ~seconds | Both services | Real communication bugs |
| E2E | Full user journey | ~minutes | Full stack | User-facing regressions |

Contract testing fills the gap between unit tests and integration tests — it's fast like unit tests but catches API contract violations like integration tests.

---

## Consumer-Driven Contracts: The Flow

```
┌──────────────────────────────────────────────────────────────────┐
│ 1. Consumer writes a test describing what it NEEDS from the API  │
│    └─ Pact generates a contract (JSON file) from the test        │
├──────────────────────────────────────────────────────────────────┤
│ 2. Contract is published to Pact Broker (or shared in repo)      │
├──────────────────────────────────────────────────────────────────┤
│ 3. Provider runs the contract against its real implementation     │
│    └─ Pact replays the interactions and checks the responses     │
├──────────────────────────────────────────────────────────────────┤
│ 4. CI gates deployment on contract verification                  │
│    └─ Backend can't deploy if it breaks a consumer contract      │
└──────────────────────────────────────────────────────────────────┘
```

---

## Writing Consumer Tests (Pact JS)

```bash
npm install -D @pact-foundation/pact
```

```typescript
// orders.pact.spec.ts (consumer test — runs on the frontend/BFF)
import { PactV3, MatchersV3 } from '@pact-foundation/pact';
import path from 'path';
import { OrdersClient } from './orders-client';

const { like, eachLike, regex, integer } = MatchersV3;

const provider = new PactV3({
  consumer: 'WebFrontend',
  provider: 'OrdersAPI',
  dir:      path.resolve(process.cwd(), 'pacts'), // where to write contract files
  port:     8080,
});

describe('Orders API contract', () => {
  describe('GET /orders/:id', () => {
    it('returns an order when it exists', async () => {
      // Define what the consumer will send and what it expects back
      await provider
        .given('order 42 exists')                  // provider state
        .uponReceiving('a request for order 42')   // interaction description
        .withRequest({
          method: 'GET',
          path:   '/orders/42',
          headers: { Accept: 'application/json' },
        })
        .willRespondWith({
          status:  200,
          headers: { 'Content-Type': 'application/json' },
          body: {
            id:     integer(42),                     // must be an integer
            status: like('pending'),                 // must have a 'status' string field
            total:  like(99.99),                     // must have a numeric 'total'
            items:  eachLike({                       // must be a non-empty array
              productId: like('prod-1'),
              quantity:  integer(2),
              price:     like(49.99),
            }),
          },
        })
        .executeTest(async (mockServer) => {
          // The client runs against Pact's mock server, not the real API
          const client = new OrdersClient(mockServer.url);
          const order  = await client.getOrder(42);

          // Assertions on the client (not the API)
          expect(order.id).toBe(42);
          expect(order.status).toBeDefined();
          expect(order.items).toHaveLength(1);
        });
    });

    it('returns 404 when order does not exist', async () => {
      await provider
        .given('order 999 does not exist')
        .uponReceiving('a request for a non-existent order')
        .withRequest({ method: 'GET', path: '/orders/999' })
        .willRespondWith({ status: 404, body: { error: like('Not found') } })
        .executeTest(async (mockServer) => {
          const client = new OrdersClient(mockServer.url);
          await expect(client.getOrder(999)).rejects.toThrow('Not found');
        });
    });
  });

  describe('POST /orders', () => {
    it('creates an order', async () => {
      await provider
        .given('user u1 has items in cart')
        .uponReceiving('a request to create an order')
        .withRequest({
          method: 'POST',
          path:   '/orders',
          headers: { 'Content-Type': 'application/json' },
          body: {
            userId: like('u1'),
            items:  eachLike({ productId: like('prod-1'), quantity: integer(1) }),
          },
        })
        .willRespondWith({
          status:  201,
          body:    { id: integer(), status: like('pending') },
        })
        .executeTest(async (mockServer) => {
          const client = new OrdersClient(mockServer.url);
          const order  = await client.createOrder({
            userId: 'u1',
            items: [{ productId: 'prod-1', quantity: 1 }],
          });
          expect(order.id).toBeDefined();
        });
    });
  });
});
```

After running this test, Pact writes a JSON contract file:

```json
// pacts/WebFrontend-OrdersAPI.json (generated — commit to repo or upload to broker)
{
  "consumer": { "name": "WebFrontend" },
  "provider": { "name": "OrdersAPI" },
  "interactions": [
    {
      "description": "a request for order 42",
      "providerStates": [{ "name": "order 42 exists" }],
      "request": { "method": "GET", "path": "/orders/42" },
      "response": {
        "status": 200,
        "body": { "id": 42, "status": "pending", "total": 99.99, "items": [...] }
      }
    }
  ]
}
```

---

## Matchers: Flexible vs Exact

Exact matching is too brittle for contract tests. Pact matchers specify structure and type:

```typescript
import { MatchersV3 } from '@pact-foundation/pact';
const { like, eachLike, integer, decimal, string, boolean, regex, datetime } = MatchersV3;

like('hello')          // must be present and type-match (string), value irrelevant
integer(42)            // must be an integer
decimal(9.99)          // must be a decimal number
string('any')          // must be a string
boolean(true)          // must be a boolean
eachLike({ id: 1 })    // must be a non-empty array of objects matching { id: integer }
regex('active|pending', 'active') // must match the regex
datetime("yyyy-MM-dd'T'HH:mm:ss", '2024-01-01T00:00:00') // must be a valid datetime
```

Use `like()` for most fields — you care about shape, not exact values.

---

## Provider Verification

The provider (backend team) pulls the contract and verifies their API satisfies it:

```typescript
// orders-api.pact.spec.ts (provider test — runs on the backend)
import { PactV3 } from '@pact-foundation/pact';
import { startServer } from './server';

const provider = new PactV3({
  consumer:              'WebFrontend',
  provider:              'OrdersAPI',
  pactUrls:              ['./pacts/WebFrontend-OrdersAPI.json'], // or fetch from broker
  providerBaseUrl:       'http://localhost:3000',
  stateHandlers: {
    // Set up test data for each provider state
    'order 42 exists': async () => {
      await db.orders.insert({ id: 42, status: 'pending', total: 99.99,
        items: [{ productId: 'prod-1', quantity: 2, price: 49.99 }] });
    },
    'order 999 does not exist': async () => {
      await db.orders.deleteAll();
    },
    'user u1 has items in cart': async () => {
      await db.users.insert({ id: 'u1' });
    },
  },
});

describe('Provider verification', () => {
  let server: Server;

  beforeAll(async () => { server = await startServer(3000); });
  afterAll(() => server.close());

  it('satisfies the WebFrontend contract', () =>
    provider.verifyProvider()
  );
});
```

Pact replays every interaction from the contract against the real running provider, applying state handlers before each one, and checks that the responses match the expected structure.

---

## Pact Broker

The Pact Broker is a service that stores contracts and verification results, enabling teams to work independently:

```bash
# Consumer: publish contract after test run
npx pact-broker publish \
  ./pacts \
  --broker-base-url https://broker.pact.io \
  --consumer-app-version $(git rev-parse HEAD) \
  --branch main

# Provider: fetch all consumer contracts for this provider
npx pact-broker can-i-deploy \
  --pacticipant OrdersAPI \
  --version $(git rev-parse HEAD) \
  --to-environment production \
  --broker-base-url https://broker.pact.io
```

`can-i-deploy` answers: "Given all known consumer contracts, is it safe to deploy this provider version to production?" If any consumer contract fails verification, it blocks the deployment.

---

## When to Use Contract Testing

**Good fit:**
- Microservices / multiple teams working on separate services
- REST or GraphQL APIs between frontend and backend
- Independent deployment cadences
- Multiple consumers of the same API (web, mobile, partner)

**Not a good fit:**
- Monoliths where frontend and backend deploy together (just use integration tests)
- Third-party APIs you don't control (can't verify the provider)
- Highly dynamic APIs where schema changes constantly (contracts become noise)
- Database-layer testing (use integration tests)

---

## Contract Testing vs Schema Validation (OpenAPI/GraphQL)

| | Contract Testing (Pact) | Schema Validation (OpenAPI/GraphQL) |
|---|---|---|
| Who defines | Consumer drives | Provider publishes |
| What it verifies | Consumer gets what it needs | Provider's output matches its schema |
| When | In CI per service | On deploy / at request time |
| Catches | Breaking changes for *known* consumers | Invalid provider responses |
| Best for | Multiple consumers, independent deploy | Single consumer or public APIs |

They complement each other — use schema validation to ensure your API conforms to its own spec, and contract testing to ensure each consumer's specific needs are met.

---

## Interview Questions

**Q: What is consumer-driven contract testing and how does it differ from integration testing?**
A: In CDC testing, the consumer writes a test describing what it *needs* from the provider API — not what the full API does. Pact generates a contract from that test and the provider verifies it. Integration testing requires both services running; contract testing runs each side independently. This makes it fast (runs in unit-test time), stable (no shared environment), and precise (catches exactly the breaking changes that affect known consumers).

**Q: Why is "like()" used instead of exact value matching in Pact?**
A: Contract tests verify *structure and type*, not exact values. A consumer cares that `order.id` is an integer and `order.status` is a string — not that it's exactly `42` or `"pending"`. Exact matching makes contracts too brittle: the provider test fails if test data differs from the exact values in the contract, even when the API is functionally correct.

**Q: What is `can-i-deploy` and how does it enable independent deployments?**
A: `can-i-deploy` queries the Pact Broker for all consumer contracts and their verification results. It answers: "Is it safe to deploy this provider version to this environment?" If any consumer contract is unverified or failing, it blocks the deployment. This lets teams deploy independently — the backend knows it can release without checking manually with every consumer team.

**Q: How does contract testing scale to many consumers?**
A: Each consumer publishes its own contract. The provider verifies all of them. A Pact Broker aggregates the results. When a backend team makes a change, running `can-i-deploy` instantly shows whether any consumer would break. New consumers add their contracts without any coordination — the provider's CI picks them up automatically on the next run.
