# LogRocket

## The Idea

**In plain English:** LogRocket is a tool that silently records everything a user does inside your web app — their clicks, what data was sent back and forth, and any errors that appeared — so that when something goes wrong, developers can rewind and watch exactly what happened like a video playback.

**Real-world analogy:** Imagine a bank installs security cameras that record every customer interaction at the counter, along with the teller's screen and the transaction logs. When a dispute arises, the manager can rewind the footage to see exactly what the customer clicked, what the system returned, and where things went wrong:

- The security camera footage = the session replay (recording DOM changes and user actions)
- The transaction log on the teller's screen = the network requests and Redux/NgRx state changes
- The manager reviewing the footage = the developer debugging a production bug in the LogRocket dashboard

---

## What It Does

LogRocket records user sessions as video-like replays, captures network requests, console logs, Redux/NgRx state changes, and performance data. It lets you reproduce and debug production bugs by watching exactly what the user experienced.

---

## React Setup

```bash
npm install logrocket
```

```ts
// main.tsx or index.tsx
import LogRocket from 'logrocket';

LogRocket.init('your-app/project-id', {
  network: {
    requestSanitizer: (request) => {
      // Don't record Authorization headers
      if (request.headers['Authorization']) {
        request.headers['Authorization'] = '[REDACTED]';
      }
      // Don't record password field in request bodies
      if (request.body) {
        const body = JSON.parse(request.body);
        if (body.password) body.password = '[REDACTED]';
        request.body = JSON.stringify(body);
      }
      return request;
    },
    responseSanitizer: (response) => {
      // Don't record sensitive response data
      if (response.body) {
        const body = JSON.parse(response.body);
        if (body.creditCard) body.creditCard = '[REDACTED]';
        response.body = JSON.stringify(body);
      }
      return response;
    },
  },
});
```

---

## Angular Setup

```ts
// app.config.ts or main.ts
import LogRocket from 'logrocket';

LogRocket.init('your-app/project-id');

// Capture Angular errors
@Injectable({ providedIn: 'root' })
export class LogRocketErrorHandler implements ErrorHandler {
  handleError(error: any): void {
    LogRocket.captureException(error);
    console.error(error);
  }
}

// providers:
{ provide: ErrorHandler, useClass: LogRocketErrorHandler }
```

---

## User Identification

```ts
// After login — link sessions to users
LogRocket.identify('user-123', {
  name: 'Alice Smith',
  email: 'alice@example.com',
  // Custom fields
  subscriptionPlan: 'pro',
  company: 'Acme Corp',
  // Note: don't include PII you don't want recorded
});
```

After identifying, all sessions for that user are grouped together. You can search for "all sessions from alice@example.com" in the dashboard.

---

## Redux State Inspection

```ts
import LogRocket from 'logrocket';
import createReactPlugin from 'logrocket-react';
import createReduxMiddleware from 'logrocket-redux';

const logRocketMiddleware = createReduxMiddleware(LogRocket);

const store = configureStore({
  reducer: rootReducer,
  middleware: (getDefault) => getDefault().concat(logRocketMiddleware),
});

// Each Redux action and state change is recorded in the LogRocket session
// You can inspect state at any point during the replay
```

---

## NgRx State Inspection

```ts
// Custom meta-reducer to send NgRx actions to LogRocket
import LogRocket from 'logrocket';

export function logRocketMetaReducer(reducer: ActionReducer<AppState>): ActionReducer<AppState> {
  return (state, action) => {
    const nextState = reducer(state, action);
    LogRocket.log({
      type: 'NGRX_ACTION',
      action: action.type,
      state: nextState,
    });
    return nextState;
  };
}

// In store providers:
provideStore(reducers, {
  metaReducers: [logRocketMetaReducer],
})
```

---

## Manual Logging

```ts
// Log custom events (visible in session timeline)
LogRocket.log('User clicked checkout');
LogRocket.info('Payment method selected', { method: 'credit-card' });
LogRocket.warn('Slow network detected', { latency: '2000ms' });
LogRocket.error('Payment failed', { code: 'card_declined' });

// Capture exceptions manually
try {
  await processPayment();
} catch (error) {
  LogRocket.captureException(error, {
    tags: { section: 'checkout' },
    extra: { orderId: order.id },
  });
}

// Track custom events (for analytics)
LogRocket.track('Add to Cart', {
  productId: product.id,
  price: product.price,
  quantity: 1,
});
```

---

## Sentry Integration

Combining LogRocket session URLs with Sentry error context:

```ts
import LogRocket from 'logrocket';
import * as Sentry from '@sentry/react';

LogRocket.getSessionURL((sessionURL) => {
  Sentry.getCurrentScope().setExtra('logRocketSession', sessionURL);
});

// Now every Sentry error includes a link to the LogRocket session replay
```

---

## Privacy Controls

```ts
LogRocket.init('app/project', {
  dom: {
    // Mask specific inputs (passwords are auto-masked)
    inputSanitizer: true,   // mask all inputs
    // Or per-element:
    // Add data-private attribute to HTML: <div data-private>sensitive</div>
  },
});
```

CSS approach — style elements as invisible to LogRocket:
```css
/* These elements are blocked in LogRocket replay */
.payment-info { visibility: hidden; }
.pii-data { privacy: hidden; }
```

---

## Comparison with Sentry

| | LogRocket | Sentry |
|---|---|---|
| Session replay | Core feature | Add-on feature |
| Error tracking | Basic | Core feature (deep stack traces) |
| Performance monitoring | Basic | Detailed (transactions, spans) |
| State inspection | Redux/NgRx built-in | Manual |
| Network recording | Full request/response | Breadcrumbs only |
| Best for | Debugging UX bugs, CS support | Error monitoring, performance |
| Pricing | Per sessions | Per events |

---

## Common Interview Questions

**Q: What does session replay actually record?**
DOM mutations (recorded as a virtual DOM diff stream), network requests/responses, console logs, user events (clicks, inputs, scrolls), and optionally Redux/NgRx state. It reconstructs the session without storing screenshots.

**Q: How do you prevent recording sensitive data?**
The `data-private` HTML attribute, CSS `visibility: hidden`, input sanitizer callbacks, request/response sanitizers, and explicit masking of elements containing PII. LogRocket auto-masks `type="password"` inputs.

**Q: When would you choose LogRocket over Sentry?**
When user-reported bugs are hard to reproduce (session replay eliminates the "I can't reproduce it" problem), for customer support workflows (support agents can watch exactly what the user did), or when debugging complex UI interactions.
