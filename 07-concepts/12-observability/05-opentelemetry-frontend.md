# OpenTelemetry for Frontend

## The Idea

**In plain English:** OpenTelemetry is a universal recording system for software — it tracks exactly what your app does, step by step, across every service involved, so you can replay and measure any action later. A "trace" is one complete recording of a single action (like a button click), and a "span" is one timestamped step within that trace.

**Real-world analogy:** Imagine a postal package with a barcode that gets scanned at every stop — the warehouse, the truck, the regional hub, and finally your door. Each scan creates a log entry tied to the same tracking number, so you can see the full journey in one place.

- The tracking number = the traceId (one ID that follows the request everywhere)
- Each scan at a checkpoint = a span (a timed record of one step in the journey)
- The tracking number printed on the label = the traceparent header (what gets passed between services so they all know they belong to the same trip)

---

## Overview

OpenTelemetry (OTel) is a vendor-neutral observability framework for collecting traces, metrics, and logs. In the frontend, it extends the same distributed tracing standard used by backend services — meaning a user interaction in the browser can produce a trace that spans the React component, the fetch request, and the Node.js API handler, all as a single correlated unit. This guide covers OTel concepts applied to the browser, instrumenting fetch and XHR, exporting to backends, and correlating frontend spans with server-side traces.

## OpenTelemetry Concepts

```
OpenTelemetry data model:

Trace: a distributed request's complete journey
  └── Span 1: "PageLoad"
        ├── Span 2: "fetch /api/products"
        │     └── Span 3: "http.server products.list" (server-side)
        │           └── Span 4: "db.query SELECT products" (database)
        └── Span 5: "React render ProductList"

Span attributes (metadata on a span):
  http.method = "GET"
  http.url = "/api/products"
  http.status_code = 200
  user.id = "u1"
  component = "ProductList"

Context propagation (linking client and server spans):
  Browser sends traceparent header:
  traceparent: 00-{traceId}-{spanId}-01
  Server reads it, creates child spans in same trace
  → One trace, spanning browser + server + database
```

## Installing OpenTelemetry for the Web

```bash
npm install \
  @opentelemetry/sdk-trace-web \
  @opentelemetry/api \
  @opentelemetry/instrumentation-fetch \
  @opentelemetry/instrumentation-xml-http-request \
  @opentelemetry/instrumentation-document-load \
  @opentelemetry/context-zone \
  @opentelemetry/exporter-trace-otlp-http
```

## Basic Setup

```typescript
// otel/setup.ts — Initialize OpenTelemetry
import { WebTracerProvider } from '@opentelemetry/sdk-trace-web';
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-base';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { registerInstrumentations } from '@opentelemetry/instrumentation';
import { FetchInstrumentation } from '@opentelemetry/instrumentation-fetch';
import { XMLHttpRequestInstrumentation } from '@opentelemetry/instrumentation-xml-http-request';
import { DocumentLoadInstrumentation } from '@opentelemetry/instrumentation-document-load';
import { UserInteractionInstrumentation } from '@opentelemetry/instrumentation-user-interaction';
import { ZoneContextManager } from '@opentelemetry/context-zone';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';
import { Resource } from '@opentelemetry/resources';

// Initialize provider
const provider = new WebTracerProvider({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'my-frontend-app',
    [SemanticResourceAttributes.SERVICE_VERSION]: import.meta.env.VITE_VERSION,
    'deployment.environment': import.meta.env.MODE,
  }),
});

// Configure exporter — sends spans to OTel Collector or directly to backend
const exporter = new OTLPTraceExporter({
  url: 'https://otel-collector.example.com/v1/traces',
  headers: {
    'Authorization': `Bearer ${import.meta.env.VITE_OTEL_TOKEN}`,
  },
});

// BatchSpanProcessor buffers spans and sends in batches
provider.addSpanProcessor(
  new BatchSpanProcessor(exporter, {
    maxQueueSize: 100,
    scheduledDelayMillis: 5000, // flush every 5s
    maxExportBatchSize: 50,
  })
);

// ZoneContextManager maintains context across async operations
provider.register({
  contextManager: new ZoneContextManager(),
});

// Auto-instrument fetch, XHR, document load, and user interactions
registerInstrumentations({
  instrumentations: [
    new DocumentLoadInstrumentation(),     // page load performance
    new FetchInstrumentation({
      propagateTraceHeaderCorsUrls: [
        /https:\/\/api\.example\.com\/.*/,  // add traceparent header to these URLs
      ],
      ignoreUrls: [
        /analytics\.example\.com/,           // don't trace analytics calls
        /sentry\.io/,                         // don't trace error reporting
      ],
    }),
    new XMLHttpRequestInstrumentation({
      propagateTraceHeaderCorsUrls: [
        /https:\/\/api\.example\.com\/.*/,
      ],
    }),
    new UserInteractionInstrumentation({
      eventNames: ['click', 'submit'],       // auto-instrument these DOM events
    }),
  ],
});

export const tracer = provider.getTracer('frontend');
```

## Creating Manual Spans

Auto-instrumentation covers network calls. Manual spans cover application logic:

```typescript
import { trace, context, SpanStatusCode } from '@opentelemetry/api';
import { tracer } from './otel/setup';

// Basic span
async function loadUserData(userId: string): Promise<User> {
  return tracer.startActiveSpan('loadUserData', async (span) => {
    span.setAttribute('user.id', userId);

    try {
      const user = await userRepository.findById(userId);
      span.setAttribute('user.found', true);
      span.setStatus({ code: SpanStatusCode.OK });
      return user;
    } catch (err) {
      span.recordException(err as Error);
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: (err as Error).message,
      });
      throw err;
    } finally {
      span.end(); // ALWAYS end the span
    }
  });
}

// Nested spans — child spans inherit parent context
async function checkoutWorkflow(cartId: string): Promise<Order> {
  return tracer.startActiveSpan('checkout', async (parentSpan) => {
    parentSpan.setAttribute('cart.id', cartId);

    try {
      // These spans are automatic children of 'checkout'
      const order = await tracer.startActiveSpan('createOrder', async (span) => {
        span.setAttribute('cart.id', cartId);
        const result = await orderApi.create(cartId);
        span.setAttribute('order.id', result.id);
        span.end();
        return result;
      });

      await tracer.startActiveSpan('processPayment', async (span) => {
        span.setAttribute('order.id', order.id);
        span.setAttribute('payment.amount', order.total);
        await paymentApi.charge(order.id);
        span.end();
      });

      parentSpan.setAttribute('order.id', order.id);
      parentSpan.setStatus({ code: SpanStatusCode.OK });
      return order;
    } catch (err) {
      parentSpan.recordException(err as Error);
      parentSpan.setStatus({ code: SpanStatusCode.ERROR });
      throw err;
    } finally {
      parentSpan.end();
    }
  });
}
```

## React Integration

```typescript
// Instrument React component lifecycle
import { useEffect, useRef } from 'react';
import { trace, SpanStatusCode } from '@opentelemetry/api';

function useSpan(componentName: string) {
  const tracer = trace.getTracer('react-components');
  const spanRef = useRef<ReturnType<typeof tracer.startSpan> | null>(null);

  useEffect(() => {
    const span = tracer.startSpan(`${componentName}.mount`);
    spanRef.current = span;

    return () => {
      spanRef.current?.end();
      spanRef.current = null;
    };
  }, [componentName, tracer]);

  return spanRef;
}

// Instrument async data loading with tracing
function useTracedQuery<T>(
  queryKey: string,
  queryFn: () => Promise<T>
) {
  const tracer = trace.getTracer('react-query');

  return useQuery({
    queryKey: [queryKey],
    queryFn: () =>
      tracer.startActiveSpan(`query.${queryKey}`, async (span) => {
        try {
          const result = await queryFn();
          span.setStatus({ code: SpanStatusCode.OK });
          return result;
        } catch (err) {
          span.recordException(err as Error);
          span.setStatus({ code: SpanStatusCode.ERROR });
          throw err;
        } finally {
          span.end();
        }
      }),
  });
}

// Usage
function ProductList() {
  const { data } = useTracedQuery('products', () => productApi.list());
  // Creates span: query.products
  // If it calls fetch /api/products → auto-instrumented child span
  // Both appear in the same trace
}
```

## Context Propagation to Server

This is the most powerful OTel feature for frontend — linking browser spans to server spans:

```typescript
// When FetchInstrumentation propagates trace context, it adds:
// traceparent: 00-{traceId}-{spanId}-01
// to fetch requests matching propagateTraceHeaderCorsUrls

// Server (Node.js Express) reads this and creates child spans:
import { trace, context, propagation } from '@opentelemetry/api';
import { NodeSDK } from '@opentelemetry/sdk-node';
import { ExpressInstrumentation } from '@opentelemetry/instrumentation-express';

// Express middleware to extract context from incoming requests
app.use((req, res, next) => {
  const parentContext = propagation.extract(context.active(), req.headers);
  context.with(parentContext, () => {
    next();
  });
});

app.get('/api/products', async (req, res) => {
  // This span is automatically a child of the browser span that sent the request
  const span = trace.getActiveSpan();
  span?.setAttribute('products.query.category', req.query.category);

  const products = await productService.list(req.query);
  res.json(products);
});
```

```
Resulting trace (visible in Jaeger/Zipkin/Grafana Tempo):

► 320ms: checkout [browser]
    ► 45ms: fetch POST /api/orders [browser - fetch auto-instrumented]
        ► 40ms: POST /api/orders [server - Express auto-instrumented]
            ► 12ms: orders.create [server - manual]
                ► 10ms: db.insert orders [server - database auto-instrumented]
    ► 60ms: fetch POST /api/payments [browser]
        ► 55ms: POST /api/payments [server]
            ► 45ms: stripe.charges.create [server]
```

## Sampling

In production, you shouldn't trace 100% of requests:

```typescript
import { TraceIdRatioBasedSampler, ParentBasedSampler } from '@opentelemetry/sdk-trace-base';

// Sample 10% of traces
const sampler = new ParentBasedSampler({
  root: new TraceIdRatioBasedSampler(0.1), // 10% sample rate
});

const provider = new WebTracerProvider({
  sampler,
  resource: new Resource({ ... }),
});

// Or: sample 100% in development, 10% in production
const sampleRate = import.meta.env.MODE === 'development' ? 1.0 : 0.1;
const sampler = new ParentBasedSampler({
  root: new TraceIdRatioBasedSampler(sampleRate),
});
```

## Exporting Options

```typescript
// Option 1: OTel Collector (recommended for production)
// Decouples your app from backend choice
const exporter = new OTLPTraceExporter({
  url: 'https://otel-collector.internal/v1/traces',
  // Collector fans out to Jaeger, Zipkin, Datadog, etc.
});

// Option 2: Jaeger directly
import { JaegerExporter } from '@opentelemetry/exporter-jaeger';
const exporter = new JaegerExporter({
  host: 'jaeger.internal',
  port: 6832,
});

// Option 3: Zipkin
import { ZipkinExporter } from '@opentelemetry/exporter-zipkin';
const exporter = new ZipkinExporter({
  url: 'https://zipkin.internal/api/v2/spans',
});

// Option 4: Datadog (via OTel Collector with datadog exporter)
// Option 5: Grafana Tempo (via OTLP HTTP)
// Option 6: Honeycomb (OTLP-compatible)
```

## Common Mistakes

### 1. Not Ending Spans

```typescript
// ❌ Span that never ends — memory leak and missing from traces
const span = tracer.startSpan('doWork');
await doWork(); // if this throws, span never ends!

// ✅ Always use try/finally or startActiveSpan callback
await tracer.startActiveSpan('doWork', async (span) => {
  try {
    return await doWork();
  } finally {
    span.end(); // always runs
  }
});
```

### 2. Adding Sensitive Data as Span Attributes

```typescript
// ❌ PII in span attributes — visible to anyone with trace access
span.setAttribute('user.password', password);
span.setAttribute('payment.card.number', cardNumber);

// ✅ Only add non-sensitive identifiers
span.setAttribute('user.id', userId);
span.setAttribute('payment.method', 'card'); // type, not number
span.setAttribute('request.id', requestId);
```

### 3. Tracing Third-Party Requests Without CORS Configuration

```typescript
// ❌ Adding traceparent header to third-party requests causes CORS preflight failures
new FetchInstrumentation({
  propagateTraceHeaderCorsUrls: [/.*$/], // matches EVERYTHING including 3rd party
})

// ✅ Only propagate to YOUR own services
new FetchInstrumentation({
  propagateTraceHeaderCorsUrls: [
    /https:\/\/api\.myapp\.com\/.*/,
    /https:\/\/internal-service\.myapp\.com\/.*/,
  ],
})
```

## Interview Questions

### 1. What is OpenTelemetry and how does it differ from a specific monitoring tool like Datadog?

**Answer:** OpenTelemetry is a vendor-neutral open standard for collecting observability data (traces, metrics, logs) with SDKs for every major language. It's the instrumentation layer — how you collect data. Datadog, Jaeger, Honeycomb, Grafana Tempo are the backends — where you store and query the data. OTel sends data in a standard format (OTLP) that any compatible backend can receive. This means you instrument your code once with OTel and can switch from Jaeger to Datadog without reinstrumentation. The OTel Collector is a proxy layer that receives OTel data and fans it out to one or many backends.

### 2. What is context propagation and why does it matter for frontend tracing?

**Answer:** Context propagation is the mechanism for linking spans across service boundaries. The browser creates a trace with a `traceId` and starts a span with a `spanId`. When it makes a fetch request to the API, it injects these IDs into the `traceparent` request header. The server reads this header, creates a new span as a child of the browser span, and all subsequent operations (database queries, downstream service calls) become children of that span. The result is a single trace that spans from the user's click in the browser all the way through the API and database. Without propagation, you have disconnected browser and server traces with no way to link them.

### 3. How do you decide what to instrument manually vs what to leave to auto-instrumentation?

**Answer:** Auto-instrumentation covers network calls (fetch, XHR), page load events, and DOM interactions — these are table stakes and should always be enabled. Manual instrumentation covers application-level semantics that auto-instrumentation can't infer: complex business workflows (checkout with multiple steps), expensive data transformations, long-running component operations, and cache operations. The guideline: instrument anything that takes >10ms or that could fail in a way you'd want to diagnose. Each manual span should answer "what is this request doing and did it succeed?" Add attributes that will help you filter and diagnose (user ID, entity IDs, relevant parameters) — not raw data values or PII.

### 4. What are the performance implications of adding OTel to the frontend?

**Answer:** The SDK adds ~50-100KB to the bundle (gzipped: ~20-30KB). Auto-instrumentation wraps fetch/XHR with a thin proxy — the overhead per call is <1ms. The `BatchSpanProcessor` buffers spans and sends them in batches (every 5s or when the buffer fills), so there's no per-request network overhead. The main performance risk is if you instrument very hot paths (every keystroke, every render) — these create thousands of spans per minute, overwhelming the exporter and adding overhead. Sample at 10-20% for high-traffic sites to control volume. Always test the SDK's impact on your specific Lighthouse score and TBT before shipping to production.
