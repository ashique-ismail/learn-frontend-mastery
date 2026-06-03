# Datadog Real User Monitoring (RUM)

## What It Is

Datadog RUM captures real user interactions, performance metrics, and errors from production frontend applications. It correlates frontend performance with backend traces, giving end-to-end visibility in a single platform — especially powerful for teams already using Datadog for backend monitoring.

---

## React Setup

```bash
npm install @datadog/browser-rum @datadog/browser-logs
```

```ts
// main.tsx
import { datadogRum } from '@datadog/browser-rum';

datadogRum.init({
  applicationId: 'your-application-id',
  clientToken: 'pub-your-client-token',
  site: 'datadoghq.com',        // or 'datadoghq.eu' for EU
  service: 'my-react-app',
  env: process.env.NODE_ENV,
  version: process.env.VITE_APP_VERSION,

  // Sampling
  sessionSampleRate: 100,        // % of sessions to track
  sessionReplaySampleRate: 20,   // % of sessions to record replay

  // Performance tracking
  trackUserInteractions: true,   // track clicks, inputs
  trackResources: true,          // track network requests
  trackLongTasks: true,          // track long JS tasks (>50ms)
  defaultPrivacyLevel: 'mask-user-input', // privacy control
});

datadogRum.startSessionReplayRecording();
```

---

## Angular Setup

```ts
// main.ts
import { datadogRum } from '@datadog/browser-rum';

datadogRum.init({
  applicationId: 'your-app-id',
  clientToken: 'pub-your-token',
  site: 'datadoghq.com',
  service: 'my-angular-app',
  env: environment.name,
  version: environment.version,
  sessionSampleRate: 100,
  sessionReplaySampleRate: 20,
  trackUserInteractions: true,
  trackResources: true,
});
```

---

## User Context

```ts
// After login — link sessions to users
datadogRum.setUser({
  id: user.id,
  name: user.displayName,
  email: user.email,
  plan: user.subscriptionPlan,
  // Don't include passwords or sensitive data
});

// Clear on logout
datadogRum.removeUser();
```

---

## Manual Error Capture

```ts
// Capture custom errors
try {
  await processPayment(order);
} catch (error) {
  datadogRum.addError(error, {
    source: 'checkout',
    orderId: order.id,
    paymentMethod: order.paymentMethod,
  });
  throw error;
}

// React Error Boundary integration
class ErrorBoundary extends React.Component {
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    datadogRum.addError(error, { componentStack: info.componentStack });
  }
}
```

---

## Custom Actions and Timing

```ts
// Track custom user actions (beyond auto-tracked clicks)
datadogRum.addAction('checkout_initiated', {
  cart_value: cart.total,
  item_count: cart.items.length,
  coupon_applied: !!cart.coupon,
});

// Track feature usage
datadogRum.addAction('feature_used', { feature: 'advanced_filter', filter_type: 'date_range' });

// Custom performance timing
datadogRum.startDurationVital('initial_data_load');
await loadDashboardData();
datadogRum.stopDurationVital('initial_data_load');
```

---

## Integration with Datadog Logs

```ts
import { datadogLogs } from '@datadog/browser-logs';

datadogLogs.init({
  clientToken: 'pub-your-token',
  site: 'datadoghq.com',
  service: 'my-app',
  env: 'production',
  forwardErrorsToLogs: true,      // capture console.error() calls
  sessionSampleRate: 100,
});

// Custom logs
datadogLogs.logger.info('User completed onboarding', { step: 'profile_setup', userId });
datadogLogs.logger.warn('Slow API response', { endpoint: '/api/reports', duration: 3200 });
datadogLogs.logger.error('Payment failed', { error: error.message, code: error.code });
```

---

## Distributed Tracing

When combined with Datadog APM (backend), RUM automatically correlates frontend requests with backend traces:

```ts
datadogRum.init({
  ...
  allowedTracingUrls: [
    'https://api.myapp.com',                              // exact URL
    /https:\/\/api\.myapp\.com/,                          // regex
    (url) => url.startsWith('https://api.myapp.com'),     // function
  ],
});
```

This injects `x-datadog-trace-id` headers into fetch/XHR requests. In the Datadog dashboard, clicking a slow frontend API call shows the full backend trace.

---

## Comparison with Sentry and LogRocket

| | Datadog RUM | Sentry | LogRocket |
|---|---|---|---|
| Session replay | Yes | Add-on | Core feature |
| Error tracking | Yes | Core feature | Basic |
| Backend correlation | Deep (if using Datadog APM) | Limited | No |
| Performance monitoring | Core Web Vitals + custom | Transactions | Basic |
| Log management | Integrated | No | Session logs |
| Infrastructure monitoring | Integrated with Datadog | Separate | No |
| Pricing model | Per session | Per event | Per session |
| Best for | Teams using Datadog stack | Error-focused | UX debugging |

---

## Common Interview Questions

**Q: What makes Datadog RUM different from Sentry?**
Datadog RUM is best when your organization already uses Datadog for backend infrastructure monitoring. The key advantage is end-to-end tracing — you can see a slow frontend API call and immediately view the corresponding backend trace, database queries, and server logs in the same platform.

**Q: What is session replay in Datadog RUM?**
A video-like recording of user DOM interactions (not actual video — reconstructed from DOM mutations). Used to reproduce bugs, understand UX patterns, and diagnose performance issues. PII is masked based on the `defaultPrivacyLevel` setting.

**Q: How do you control costs?**
`sessionSampleRate` and `sessionReplaySampleRate` control what percentage of sessions are tracked. Set `sessionReplaySampleRate` to a lower value (5-20%) since replays are the most expensive. Use `startSessionReplayRecording()` conditionally to only record sessions that encounter errors.
