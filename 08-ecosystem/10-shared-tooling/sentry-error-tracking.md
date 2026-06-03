# Sentry

## What It Does

Sentry captures errors, performance issues, and user session data from production applications. It provides stack traces, breadcrumbs (what the user did before the error), and context to help you reproduce and fix bugs.

---

## React Setup

```bash
npm install @sentry/react
```

```tsx
// main.tsx
import * as Sentry from '@sentry/react';

Sentry.init({
  dsn: 'https://your-dsn@sentry.io/project-id',
  environment: process.env.NODE_ENV,
  release: process.env.VITE_APP_VERSION, // link errors to code releases

  // Performance monitoring
  tracesSampleRate: 0.1, // sample 10% of transactions
  profilesSampleRate: 0.1,

  // Replays (session replay)
  replaysSessionSampleRate: 0.1,    // 10% of all sessions
  replaysOnErrorSampleRate: 1.0,    // 100% of sessions with errors

  integrations: [
    Sentry.browserTracingIntegration(),
    Sentry.replayIntegration({
      maskAllText: false, // mask sensitive text
      blockAllMedia: false,
    }),
  ],
});
```

---

## Error Boundary Integration

```tsx
import * as Sentry from '@sentry/react';

function App() {
  return (
    <Sentry.ErrorBoundary
      fallback={({ error, resetError }) => (
        <div>
          <h2>Something went wrong</h2>
          <p>{error.message}</p>
          <button onClick={resetError}>Try Again</button>
        </div>
      )}
      beforeCapture={(scope) => {
        scope.setTag('section', 'main-app');
      }}
    >
      <MainApp />
    </Sentry.ErrorBoundary>
  );
}
```

---

## Angular Setup

```bash
npm install @sentry/angular
```

```ts
// app.config.ts
import * as Sentry from '@sentry/angular';

Sentry.init({
  dsn: 'https://your-dsn@sentry.io/project-id',
  integrations: [
    Sentry.browserTracingIntegration(),
  ],
  tracesSampleRate: 0.1,
});

// app.config.ts providers
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    {
      provide: ErrorHandler,
      useValue: Sentry.createErrorHandler({ showDialog: false }),
    },
    {
      provide: Sentry.TraceService,
      deps: [Router],
    },
    {
      provide: APP_INITIALIZER,
      useFactory: () => () => {},
      deps: [Sentry.TraceService],
      multi: true,
    },
  ],
};
```

---

## Manual Error Capture

```ts
// Capture an exception
try {
  await riskyOperation();
} catch (error) {
  Sentry.captureException(error, {
    tags: { section: 'checkout' },
    extra: { orderId: order.id, userId: user.id },
    level: 'error',
  });
}

// Add user context
Sentry.setUser({
  id: user.id,
  email: user.email,
  username: user.username,
});

// Add custom context
Sentry.setContext('subscription', {
  plan: 'pro',
  billingPeriod: 'monthly',
});

// Add breadcrumb (trail leading to error)
Sentry.addBreadcrumb({
  category: 'ui.click',
  message: 'User clicked checkout button',
  level: 'info',
  data: { cartItems: cart.length },
});
```

---

## Performance Monitoring

```ts
// Manually trace a transaction
const transaction = Sentry.startInactiveSpan({
  name: 'process-order',
  op: 'task',
});

const span = transaction.startChild({
  op: 'db.query',
  description: 'SELECT * FROM orders',
});
await queryDatabase();
span.finish();

await processPayment();
transaction.finish();
```

React integration auto-tracks route changes and component renders via `browserTracingIntegration()`.

---

## Source Maps

Essential for readable stack traces in minified production code:

```bash
# Upload source maps with release
npx sentry-cli releases new $VERSION
npx sentry-cli releases files $VERSION upload-sourcemaps ./dist --rewrite
npx sentry-cli releases finalize $VERSION
npx sentry-cli releases deploys $VERSION new -e production
```

```js
// vite.config.ts
import { sentryVitePlugin } from '@sentry/vite-plugin';

export default defineConfig({
  plugins: [
    react(),
    sentryVitePlugin({
      org: 'my-org',
      project: 'my-project',
      authToken: process.env.SENTRY_AUTH_TOKEN,
    }),
  ],
  build: {
    sourcemap: true, // generate source maps
  },
});
```

---

## Alerting Rules

In Sentry dashboard:
- **Issue alerts**: Notify on first occurrence, spike in frequency, or regression
- **Metric alerts**: Notify when error rate > threshold (e.g., > 5% of sessions have errors)
- **Notification channels**: Slack, email, PagerDuty, GitHub Issues

---

## Sampling Configuration

```ts
// Dynamic sampling based on context
tracesSampler: (samplingContext) => {
  // Always sample health check transactions
  if (samplingContext.transactionContext?.name === '/api/health') return 0;

  // Sample 100% of checkout transactions (high business value)
  if (samplingContext.transactionContext?.name?.includes('checkout')) return 1.0;

  // Default: 10%
  return 0.1;
},
```

---

## Common Interview Questions

**Q: What are breadcrumbs in Sentry?**
A chronological log of events leading up to an error — navigation, clicks, network requests, console logs. When an error occurs, the breadcrumb trail helps you understand exactly what the user was doing.

**Q: What's session replay?**
A video-like recording of user interactions. When an error occurs with session replay enabled, you can watch exactly what the user did. PII (passwords, credit cards) are automatically masked.

**Q: How do you prevent Sentry from capturing sensitive data?**
Use `beforeSend` callback to scrub PII from events, enable `maskAllText` in replay settings, configure `denyUrls` to skip tracking errors from third-party scripts, and set up PII scrubbing rules in the Sentry dashboard.
