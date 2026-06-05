# Error Tracking

## The Idea

**In plain English:** Error tracking is a system that automatically notices when something goes wrong in an app that real people are using, records exactly what happened, and alerts the developers so they can fix it. Think of it as a security camera for bugs — instead of finding out about problems from angry users, the system catches and reports them the moment they occur.

**Real-world analogy:** Imagine a large hotel with hundreds of rooms. The hotel manager cannot personally walk into every room to check if something is broken, so the hotel installs a call button in each room. When a guest presses it, the front desk logs the room number, the time, what the guest reported, and which staff member was on duty — then sends a maintenance alert.

- The guest pressing the call button = an error thrown in your code
- The front desk log = the error tracking dashboard (e.g., Sentry)
- The room number and details the guest provides = the stack trace and context captured with the error
- The maintenance alert sent to staff = the notification or alert fired to the development team

---

## Overview

Error tracking is the practice of automatically detecting, capturing, and monitoring errors in production applications. Modern error tracking tools like Sentry provide real-time error monitoring, detailed stack traces, context about when and how errors occur, and insights into which errors impact the most users. Effective error tracking helps teams identify and fix issues quickly, improving application reliability and user experience.

## Table of Contents
- [Error Tracking Fundamentals](#error-tracking-fundamentals)
- [Sentry Integration](#sentry-integration)
- [Error Boundaries](#error-boundaries)
- [Source Maps](#source-maps)
- [Error Context](#error-context)
- [Alerting and Notifications](#alerting-and-notifications)
- [Error Grouping](#error-grouping)
- [Release Tracking](#release-tracking)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use/Not to Use](#when-to-usenot-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Error Tracking Fundamentals

### Why Error Tracking Matters

```
Traditional Logging          vs          Error Tracking
──────────────────                      ──────────────

❌ Scattered in log files               ✅ Centralized dashboard
❌ Hard to search/filter                ✅ Powerful search & filters
❌ No automatic grouping                ✅ Smart error grouping
❌ Manual log analysis                  ✅ Automatic analysis
❌ No user impact metrics               ✅ User & session tracking
❌ No release correlation               ✅ Release tracking
❌ Missing context                      ✅ Rich context capture
```

### Core Concepts

```javascript
// What error tracking captures
const errorEvent = {
  // Basic error information
  message: 'Cannot read property "length" of undefined',
  type: 'TypeError',
  stack: 'at UserProfile.render (UserProfile.jsx:45:12)...',
  
  // Context
  user: {
    id: '12345',
    email: 'user@example.com',
    username: 'johndoe',
  },
  
  // Request information
  request: {
    url: 'https://example.com/profile',
    method: 'GET',
    headers: { ... },
    query: { tab: 'settings' },
  },
  
  // Environment
  environment: 'production',
  release: 'v1.2.3',
  platform: 'javascript',
  
  // Browser/device information
  browser: {
    name: 'Chrome',
    version: '120.0.0',
  },
  os: {
    name: 'Windows',
    version: '10',
  },
  device: {
    type: 'desktop',
    brand: 'Dell',
  },
  
  // Breadcrumbs (events leading to error)
  breadcrumbs: [
    { type: 'navigation', message: 'Navigated to /profile' },
    { type: 'http', message: 'GET /api/user/12345' },
    { type: 'user', message: 'Clicked "Edit Profile"' },
  ],
  
  // Tags for filtering
  tags: {
    feature: 'user-profile',
    team: 'frontend',
  },
  
  // Additional context
  extra: {
    componentState: { ... },
    propsData: { ... },
  },
};
```

## Sentry Integration

### Node.js/Express Setup

```javascript
// sentry-setup.js
const Sentry = require('@sentry/node');
const Tracing = require('@sentry/tracing');

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  
  // Environment
  environment: process.env.NODE_ENV || 'development',
  
  // Release tracking
  release: `my-app@${process.env.npm_package_version}`,
  
  // Performance monitoring
  tracesSampleRate: 0.1, // 10% of transactions
  
  // Integrations
  integrations: [
    new Sentry.Integrations.Http({ tracing: true }),
    new Tracing.Integrations.Express({ app }),
    new Tracing.Integrations.Postgres(),
    new Tracing.Integrations.Mongo(),
  ],
  
  // Before send hook (modify or filter events)
  beforeSend(event, hint) {
    // Don't send errors from bots
    if (event.request?.headers?.['user-agent']?.includes('bot')) {
      return null;
    }
    
    // Redact sensitive data
    if (event.request?.headers?.authorization) {
      event.request.headers.authorization = '[Redacted]';
    }
    
    // Add custom tags
    event.tags = {
      ...event.tags,
      server: require('os').hostname(),
    };
    
    return event;
  },
  
  // Ignore certain errors
  ignoreErrors: [
    'NetworkError',
    'AbortError',
    /timeout of \d+ms exceeded/,
  ],
});

// Express middleware
const express = require('express');
const app = express();

// RequestHandler creates a separate execution context using domains
app.use(Sentry.Handlers.requestHandler());

// TracingHandler creates a trace for every incoming request
app.use(Sentry.Handlers.tracingHandler());

// Routes
app.get('/api/users/:id', async (req, res) => {
  try {
    const user = await getUser(req.params.id);
    res.json(user);
  } catch (error) {
    // Manually capture exception with additional context
    Sentry.captureException(error, {
      tags: {
        feature: 'user-api',
      },
      extra: {
        userId: req.params.id,
        requestId: req.id,
      },
    });
    
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Error handler (must be before other error middleware)
app.use(Sentry.Handlers.errorHandler());

// Optional fallback error handler
app.use((err, req, res, next) => {
  res.status(500).json({
    error: 'Internal server error',
    id: res.sentry, // Error ID from Sentry
  });
});

app.listen(3000);

// Graceful shutdown
process.on('SIGTERM', () => {
  Sentry.close(2000).then(() => {
    process.exit(0);
  });
});
```

### React Setup

```typescript
// sentry-config.ts
import * as Sentry from '@sentry/react';
import { BrowserTracing } from '@sentry/tracing';

Sentry.init({
  dsn: process.env.REACT_APP_SENTRY_DSN,
  
  environment: process.env.NODE_ENV,
  release: process.env.REACT_APP_VERSION,
  
  // Performance monitoring
  integrations: [
    new BrowserTracing({
      // Trace all navigation
      routingInstrumentation: Sentry.reactRouterV6Instrumentation(
        React.useEffect,
        useLocation,
        useNavigationType,
        createRoutesFromChildren,
        matchRoutes
      ),
    }),
  ],
  
  tracesSampleRate: 0.1,
  
  // Session replay (record user sessions leading to errors)
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
  
  // Breadcrumbs
  beforeBreadcrumb(breadcrumb, hint) {
    // Filter out noisy breadcrumbs
    if (breadcrumb.category === 'console' && breadcrumb.level === 'debug') {
      return null;
    }
    
    // Redact sensitive data from breadcrumbs
    if (breadcrumb.category === 'fetch' && breadcrumb.data?.url?.includes('password')) {
      breadcrumb.data.url = '[Redacted]';
    }
    
    return breadcrumb;
  },
  
  beforeSend(event, hint) {
    // Attach user feedback
    if (event.exception) {
      Sentry.showReportDialog({ eventId: event.event_id });
    }
    
    return event;
  },
  
  ignoreErrors: [
    // Browser extensions
    'top.GLOBALS',
    'chrome-extension://',
    'moz-extension://',
    // Network errors
    'Network request failed',
    'Failed to fetch',
    // Third-party scripts
    /gtag/i,
    /google-analytics/i,
  ],
});

// Identify user (call after authentication)
export function identifyUser(user: { id: string; email: string; username: string }) {
  Sentry.setUser({
    id: user.id,
    email: user.email,
    username: user.username,
  });
}

// Clear user on logout
export function clearUser() {
  Sentry.setUser(null);
}

// Add custom context
export function setCustomContext(context: Record<string, any>) {
  Sentry.setContext('custom', context);
}

// Add tags
export function setTags(tags: Record<string, string>) {
  Sentry.setTags(tags);
}

// Manually capture exception
export function captureError(error: Error, context?: Record<string, any>) {
  Sentry.captureException(error, {
    extra: context,
  });
}

// Capture message
export function captureMessage(message: string, level: Sentry.SeverityLevel = 'info') {
  Sentry.captureMessage(message, level);
}

// Add breadcrumb
export function addBreadcrumb(breadcrumb: Sentry.Breadcrumb) {
  Sentry.addBreadcrumb(breadcrumb);
}
```

```tsx
// App.tsx
import React from 'react';
import * as Sentry from '@sentry/react';
import { BrowserRouter } from 'react-router-dom';

// Wrap App with Sentry profiler
const App = () => {
  return (
    <Sentry.ErrorBoundary
      fallback={<ErrorFallback />}
      showDialog
      beforeCapture={(scope) => {
        scope.setTag('location', 'app-root');
      }}
    >
      <BrowserRouter>
        <Routes>
          {/* Your routes */}
        </Routes>
      </BrowserRouter>
    </Sentry.ErrorBoundary>
  );
};

// Error fallback component
const ErrorFallback: React.FC = () => {
  return (
    <div className="error-fallback">
      <h1>Oops! Something went wrong</h1>
      <p>We've been notified and are working on a fix.</p>
      <button onClick={() => window.location.reload()}>
        Reload page
      </button>
    </div>
  );
};

export default Sentry.withProfiler(App);

// Using in components
const UserProfile: React.FC = () => {
  const handleAction = async () => {
    // Add breadcrumb for user action
    Sentry.addBreadcrumb({
      category: 'user-action',
      message: 'User clicked save button',
      level: 'info',
    });
    
    try {
      await saveProfile();
    } catch (error) {
      // Capture with additional context
      Sentry.captureException(error, {
        tags: {
          feature: 'user-profile',
        },
        extra: {
          action: 'save-profile',
          timestamp: new Date().toISOString(),
        },
      });
    }
  };
  
  return <div>User Profile</div>;
};
```

### Angular Setup

```typescript
// sentry.config.ts
import * as Sentry from '@sentry/angular-ivy';
import { ErrorHandler, Injectable, InjectionToken } from '@angular/core';
import { Router } from '@angular/router';

export const SENTRY_CONFIG = new InjectionToken<string>('SENTRY_CONFIG');

export function initSentry() {
  Sentry.init({
    dsn: environment.sentryDsn,
    environment: environment.production ? 'production' : 'development',
    release: environment.version,
    
    integrations: [
      new Sentry.BrowserTracing({
        routingInstrumentation: Sentry.routingInstrumentation,
      }),
      new Sentry.Replay(),
    ],
    
    tracesSampleRate: 0.1,
    replaysSessionSampleRate: 0.1,
    replaysOnErrorSampleRate: 1.0,
  });
}

// Custom error handler
@Injectable()
export class SentryErrorHandler implements ErrorHandler {
  constructor() {}
  
  handleError(error: any): void {
    // Log to console in development
    if (!environment.production) {
      console.error(error);
    }
    
    // Extract useful information
    const chunkFailedMessage = /Loading chunk [\d]+ failed/;
    
    if (chunkFailedMessage.test(error.message)) {
      // Handle chunk loading errors (suggest reload)
      Sentry.captureMessage('Chunk loading failed', 'warning');
    } else {
      Sentry.captureException(error);
    }
  }
}

// app.module.ts
import { APP_INITIALIZER, ErrorHandler, NgModule } from '@angular/core';
import { Router } from '@angular/router';
import * as Sentry from '@sentry/angular-ivy';

@NgModule({
  providers: [
    {
      provide: ErrorHandler,
      useClass: SentryErrorHandler,
    },
    {
      provide: Sentry.TraceService,
      deps: [Router],
    },
    {
      provide: APP_INITIALIZER,
      useFactory: () => initSentry,
      multi: true,
    },
  ],
})
export class AppModule {}

// sentry.service.ts
import { Injectable } from '@angular/core';
import * as Sentry from '@sentry/angular-ivy';

@Injectable({
  providedIn: 'root'
})
export class SentryService {
  setUser(user: { id: string; email: string; username: string }): void {
    Sentry.setUser({
      id: user.id,
      email: user.email,
      username: user.username,
    });
  }
  
  clearUser(): void {
    Sentry.setUser(null);
  }
  
  captureException(error: Error, context?: Record<string, any>): void {
    Sentry.captureException(error, {
      extra: context,
    });
  }
  
  captureMessage(message: string, level: Sentry.SeverityLevel = 'info'): void {
    Sentry.captureMessage(message, level);
  }
  
  addBreadcrumb(breadcrumb: Sentry.Breadcrumb): void {
    Sentry.addBreadcrumb(breadcrumb);
  }
  
  setTag(key: string, value: string): void {
    Sentry.setTag(key, value);
  }
  
  setContext(name: string, context: Record<string, any>): void {
    Sentry.setContext(name, context);
  }
}

// Usage in component
import { Component, OnInit } from '@angular/core';
import { SentryService } from './sentry.service';

@Component({
  selector: 'app-user-profile',
  templateUrl: './user-profile.component.html'
})
export class UserProfileComponent implements OnInit {
  constructor(private sentry: SentryService) {}
  
  ngOnInit(): void {
    this.sentry.addBreadcrumb({
      category: 'navigation',
      message: 'Navigated to user profile',
      level: 'info',
    });
  }
  
  async saveProfile(): Promise<void> {
    try {
      this.sentry.addBreadcrumb({
        category: 'user-action',
        message: 'Clicked save button',
        level: 'info',
      });
      
      await this.userService.saveProfile();
    } catch (error) {
      this.sentry.captureException(error as Error, {
        feature: 'user-profile',
        action: 'save',
      });
    }
  }
}
```

## Error Boundaries

### React Error Boundaries

```typescript
// ErrorBoundary.tsx
import React, { Component, ErrorInfo, ReactNode } from 'react';
import * as Sentry from '@sentry/react';

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = {
      hasError: false,
      error: null,
    };
  }
  
  static getDerivedStateFromError(error: Error): State {
    return {
      hasError: true,
      error,
    };
  }
  
  componentDidCatch(error: Error, errorInfo: ErrorInfo): void {
    // Log error to console in development
    if (process.env.NODE_ENV === 'development') {
      console.error('Error boundary caught error:', error, errorInfo);
    }
    
    // Send to Sentry
    Sentry.captureException(error, {
      contexts: {
        react: {
          componentStack: errorInfo.componentStack,
        },
      },
    });
    
    // Call custom error handler
    this.props.onError?.(error, errorInfo);
  }
  
  resetError = (): void => {
    this.setState({
      hasError: false,
      error: null,
    });
  };
  
  render(): ReactNode {
    if (this.state.hasError) {
      // Custom fallback UI
      if (this.props.fallback) {
        return this.props.fallback;
      }
      
      // Default fallback UI
      return (
        <div className="error-boundary-fallback">
          <h2>Something went wrong</h2>
          <details>
            <summary>Error details</summary>
            <pre>{this.state.error?.toString()}</pre>
          </details>
          <button onClick={this.resetError}>Try again</button>
        </div>
      );
    }
    
    return this.props.children;
  }
}

export default ErrorBoundary;

// Usage
import ErrorBoundary from './ErrorBoundary';

const App = () => {
  return (
    <ErrorBoundary
      fallback={<ErrorFallback />}
      onError={(error, errorInfo) => {
        console.log('Error caught:', error);
        // Additional error handling
      }}
    >
      <Routes>
        <Route path="/" element={<Home />} />
        <Route
          path="/profile"
          element={
            <ErrorBoundary fallback={<ProfileErrorFallback />}>
              <UserProfile />
            </ErrorBoundary>
          }
        />
      </Routes>
    </ErrorBoundary>
  );
};
```

### Async Error Boundaries

```typescript
// AsyncErrorBoundary.tsx
import React, { Component, ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback?: (error: Error, retry: () => void) => ReactNode;
}

interface State {
  error: Error | null;
}

class AsyncErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { error: null };
  }
  
  static getDerivedStateFromError(error: Error): State {
    return { error };
  }
  
  componentDidCatch(error: Error, errorInfo: React.ErrorInfo): void {
    // Log error
    console.error('Async error caught:', error, errorInfo);
  }
  
  resetError = (): void => {
    this.setState({ error: null });
  };
  
  render(): ReactNode {
    if (this.state.error) {
      if (this.props.fallback) {
        return this.props.fallback(this.state.error, this.resetError);
      }
      
      return (
        <div>
          <p>Failed to load component</p>
          <button onClick={this.resetError}>Retry</button>
        </div>
      );
    }
    
    return this.props.children;
  }
}

// Usage with Suspense
const LazyComponent = React.lazy(() => import('./LazyComponent'));

const App = () => {
  return (
    <AsyncErrorBoundary
      fallback={(error, retry) => (
        <div>
          <p>Failed to load: {error.message}</p>
          <button onClick={retry}>Retry</button>
        </div>
      )}
    >
      <Suspense fallback={<Loading />}>
        <LazyComponent />
      </Suspense>
    </AsyncErrorBoundary>
  );
};
```

### Angular Error Handling

```typescript
// global-error-handler.ts
import { ErrorHandler, Injectable, Injector } from '@angular/core';
import { HttpErrorResponse } from '@angular/common/http';
import { SentryService } from './sentry.service';

@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  constructor(private injector: Injector) {}
  
  handleError(error: Error | HttpErrorResponse): void {
    const sentry = this.injector.get(SentryService);
    
    // Handle different error types
    if (error instanceof HttpErrorResponse) {
      // Server error
      console.error('HTTP Error:', error);
      
      sentry.captureException(error, {
        type: 'http-error',
        status: error.status,
        url: error.url,
      });
    } else {
      // Client error
      console.error('Client Error:', error);
      
      sentry.captureException(error, {
        type: 'client-error',
      });
    }
    
    // Show user-friendly message
    this.showErrorToUser(error);
  }
  
  private showErrorToUser(error: Error | HttpErrorResponse): void {
    // Implement notification service
    // this.notificationService.showError('An error occurred');
  }
}

// app.module.ts
@NgModule({
  providers: [
    {
      provide: ErrorHandler,
      useClass: GlobalErrorHandler,
    },
  ],
})
export class AppModule {}
```

## Source Maps

### Generating Source Maps

```javascript
// webpack.config.js
module.exports = {
  mode: 'production',
  
  // Generate source maps
  devtool: 'source-map', // or 'hidden-source-map' for security
  
  output: {
    filename: '[name].[contenthash].js',
    sourceMapFilename: '[name].[contenthash].js.map',
  },
  
  plugins: [
    // Upload source maps to Sentry
    new SentryWebpackPlugin({
      authToken: process.env.SENTRY_AUTH_TOKEN,
      org: 'my-org',
      project: 'my-project',
      
      // Include source maps
      include: './dist',
      
      // Ignore node_modules
      ignore: ['node_modules'],
      
      // Delete source maps after upload (security)
      deleteAfterCompile: true,
      
      // Set release
      release: process.env.REACT_APP_VERSION,
    }),
  ],
};
```

```json
// package.json scripts
{
  "scripts": {
    "build": "webpack --mode production",
    "build:sentry": "webpack --mode production && sentry-cli releases files $npm_package_version upload-sourcemaps ./dist --url-prefix '~/static/js/'",
    "release": "npm run build:sentry && sentry-cli releases finalize $npm_package_version"
  }
}
```

### Sentry CLI Source Map Upload

```bash
# Install Sentry CLI
npm install -g @sentry/cli

# Create release
sentry-cli releases new "my-app@1.2.3"

# Upload source maps
sentry-cli releases files "my-app@1.2.3" upload-sourcemaps ./dist \
  --url-prefix '~/static/js/' \
  --validate

# Finalize release
sentry-cli releases finalize "my-app@1.2.3"

# Set deploy
sentry-cli releases deploys "my-app@1.2.3" new \
  --env production \
  --name "Production Deploy"
```

### Source Map Security

```javascript
// webpack.config.js
module.exports = {
  // Use hidden-source-map to not expose source maps to users
  devtool: 'hidden-source-map',
  
  plugins: [
    new SentryWebpackPlugin({
      // Upload to Sentry
      include: './dist',
      
      // Delete source maps after upload
      deleteAfterCompile: true,
      
      // Or move to separate directory
      // cleanArtifacts: true,
    }),
  ],
};

// Alternatively, serve source maps only to Sentry
// nginx.conf
location ~ \.map$ {
  # Deny public access
  deny all;
  
  # Allow Sentry IPs
  allow 34.120.195.249;
  allow 35.184.238.160;
}
```

## Error Context

### Adding Custom Context

```typescript
// context-manager.ts
import * as Sentry from '@sentry/react';

export class ErrorContextManager {
  static setUserContext(user: {
    id: string;
    email: string;
    username: string;
    role: string;
  }): void {
    Sentry.setUser({
      id: user.id,
      email: user.email,
      username: user.username,
    });
    
    Sentry.setTag('user_role', user.role);
  }
  
  static setFeatureContext(feature: string, enabled: boolean): void {
    Sentry.setTag(`feature_${feature}`, enabled.toString());
  }
  
  static setBusinessContext(context: {
    organizationId?: string;
    subscriptionTier?: string;
    trialStatus?: string;
  }): void {
    Sentry.setContext('business', context);
  }
  
  static setPerformanceContext(metrics: {
    loadTime?: number;
    firstPaint?: number;
    domContentLoaded?: number;
  }): void {
    Sentry.setContext('performance', metrics);
  }
  
  static addBreadcrumb(category: string, message: string, data?: Record<string, any>): void {
    Sentry.addBreadcrumb({
      category,
      message,
      level: 'info',
      data,
      timestamp: Date.now() / 1000,
    });
  }
}

// Usage in React
import { ErrorContextManager } from './context-manager';

const App = () => {
  useEffect(() => {
    // Set user context after authentication
    const user = getCurrentUser();
    if (user) {
      ErrorContextManager.setUserContext(user);
    }
    
    // Track performance metrics
    if (window.performance) {
      const timing = window.performance.timing;
      ErrorContextManager.setPerformanceContext({
        loadTime: timing.loadEventEnd - timing.navigationStart,
        firstPaint: timing.responseStart - timing.navigationStart,
        domContentLoaded: timing.domContentLoadedEventEnd - timing.navigationStart,
      });
    }
  }, []);
  
  return <Routes />;
};
```

### Request Context

```javascript
// Express middleware for request context
const Sentry = require('@sentry/node');

app.use((req, res, next) => {
  // Add request context to all errors in this request
  Sentry.configureScope((scope) => {
    scope.setContext('request', {
      method: req.method,
      url: req.url,
      query: req.query,
      ip: req.ip,
      userAgent: req.get('user-agent'),
      referrer: req.get('referrer'),
    });
    
    scope.setTag('route', req.route?.path || 'unknown');
    scope.setTag('method', req.method);
    
    // Add user if authenticated
    if (req.user) {
      scope.setUser({
        id: req.user.id,
        email: req.user.email,
        username: req.user.username,
      });
    }
    
    // Add correlation ID
    if (req.correlationId) {
      scope.setTag('correlation_id', req.correlationId);
    }
  });
  
  next();
});
```

## Alerting and Notifications

### Alert Rules

```javascript
// Sentry alert rules configuration (via API)
const alertRule = {
  name: 'High Error Rate',
  environment: 'production',
  
  // Trigger conditions
  conditions: [
    {
      id: 'sentry.rules.conditions.event_frequency.EventFrequencyCondition',
      interval: '5m',
      value: 100, // More than 100 errors
    },
  ],
  
  // Filters
  filters: [
    {
      id: 'sentry.rules.filters.issue_occurrences.IssueOccurrencesFilter',
      value: 10, // At least 10 occurrences
    },
  ],
  
  // Actions
  actions: [
    {
      id: 'sentry.integrations.slack.notify_action.SlackNotifyServiceAction',
      workspace: 'my-workspace',
      channel: '#alerts',
      tags: 'environment,level',
    },
    {
      id: 'sentry.mail.actions.NotifyEmailAction',
      targetType: 'Team',
      targetIdentifier: 'engineering',
    },
    {
      id: 'sentry.integrations.pagerduty.notify_action.PagerDutyNotifyServiceAction',
      service: 'production-api',
      severity: 'critical',
    },
  ],
  
  // Frequency
  frequency: 5, // Minutes
  actionMatch: 'all', // or 'any'
};
```

### Slack Integration

```javascript
// Slack webhook notification
const sendSlackAlert = async (error) => {
  const webhook = process.env.SLACK_WEBHOOK_URL;
  
  const message = {
    text: 'Error Alert',
    blocks: [
      {
        type: 'header',
        text: {
          type: 'plain_text',
          text: '🚨 Production Error Alert',
        },
      },
      {
        type: 'section',
        fields: [
          {
            type: 'mrkdwn',
            text: `*Error:*\n${error.message}`,
          },
          {
            type: 'mrkdwn',
            text: `*Environment:*\n${error.environment}`,
          },
          {
            type: 'mrkdwn',
            text: `*Count:*\n${error.count} occurrences`,
          },
          {
            type: 'mrkdwn',
            text: `*Users Affected:*\n${error.userCount}`,
          },
        ],
      },
      {
        type: 'section',
        text: {
          type: 'mrkdwn',
          text: `*<${error.url}|View in Sentry>*`,
        },
      },
    ],
  };
  
  await fetch(webhook, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(message),
  });
};
```

## Error Grouping

### Custom Fingerprinting

```javascript
// Custom fingerprinting for better error grouping
Sentry.init({
  dsn: process.env.SENTRY_DSN,
  
  beforeSend(event, hint) {
    // Custom fingerprinting
    if (event.exception?.values?.[0]) {
      const error = event.exception.values[0];
      
      // Group by error message pattern
      if (error.value?.includes('timeout')) {
        event.fingerprint = ['timeout-error', error.type];
      }
      
      // Group by endpoint for API errors
      if (event.request?.url) {
        const url = new URL(event.request.url);
        event.fingerprint = ['api-error', url.pathname, error.type];
      }
      
      // Group by component for React errors
      if (error.stacktrace?.frames) {
        const topFrame = error.stacktrace.frames[error.stacktrace.frames.length - 1];
        if (topFrame?.filename?.includes('components/')) {
          event.fingerprint = ['component-error', topFrame.filename, error.type];
        }
      }
    }
    
    return event;
  },
});
```

### Issue Management

```javascript
// Sentry API integration for issue management
const SentryAPI = require('@sentry/api');

class IssueManager {
  constructor(apiToken, org, project) {
    this.client = new SentryAPI(apiToken);
    this.org = org;
    this.project = project;
  }
  
  async resolveIssue(issueId) {
    await this.client.put(`/issues/${issueId}/`, {
      status: 'resolved',
    });
  }
  
  async ignoreIssue(issueId) {
    await this.client.put(`/issues/${issueId}/`, {
      status: 'ignored',
    });
  }
  
  async assignIssue(issueId, userId) {
    await this.client.put(`/issues/${issueId}/`, {
      assignedTo: userId,
    });
  }
  
  async addComment(issueId, comment) {
    await this.client.post(`/issues/${issueId}/comments/`, {
      data: { text: comment },
    });
  }
  
  async getIssueStats(issueId) {
    const stats = await this.client.get(`/issues/${issueId}/stats/`);
    return stats;
  }
}
```

## Release Tracking

### Creating Releases

```javascript
// Create release programmatically
const Sentry = require('@sentry/node');

async function createRelease(version) {
  const release = {
    version,
    projects: ['my-project'],
  };
  
  // Create release
  await Sentry.createRelease(release);
  
  // Associate commits
  await Sentry.setCommits(version, {
    auto: true, // Automatically determine commits
  });
  
  // Or specify commits manually
  await Sentry.setCommits(version, {
    commits: [
      {
        id: 'abc123',
        repository: 'my-org/my-repo',
        author_email: 'dev@example.com',
        author_name: 'Developer',
        message: 'Fix bug in user profile',
        timestamp: '2024-06-15T10:30:00Z',
      },
    ],
  });
  
  // Deploy
  await Sentry.createDeploy(version, {
    environment: 'production',
    name: 'Production Deploy',
    url: 'https://example.com',
  });
}

// Usage in CI/CD
createRelease(process.env.VERSION);
```

### Release Health Monitoring

```javascript
// Track session health
import * as Sentry from '@sentry/react';

// Initialize with session tracking
Sentry.init({
  dsn: process.env.SENTRY_DSN,
  release: process.env.VERSION,
  
  // Track sessions
  autoSessionTracking: true,
  
  // Session replay
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
});

// Manually start/end session
Sentry.startSession();
Sentry.endSession();

// Track custom session data
Sentry.configureScope((scope) => {
  scope.setExtra('session_metadata', {
    duration: sessionDuration,
    actionsPerformed: actionCount,
  });
});
```

## Common Mistakes

### 1. Not Filtering Sensitive Data

```javascript
// Bad: Sending passwords to error tracker
logger.error('Login failed', { 
  username: 'user@example.com', 
  password: 'secret123' // NEVER!
});

// Good: Filter sensitive data
Sentry.init({
  beforeSend(event) {
    // Remove sensitive fields
    if (event.request?.data) {
      delete event.request.data.password;
      delete event.request.data.creditCard;
    }
    return event;
  },
});
```

### 2. Catching and Ignoring Errors

```javascript
// Bad: Silent failures
try {
  await fetchData();
} catch (error) {
  // Error silently ignored!
}

// Good: Track errors even if handled
try {
  await fetchData();
} catch (error) {
  Sentry.captureException(error, {
    level: 'warning',
    tags: { handled: true },
  });
  // Show user-friendly message
}
```

### 3. Too Much Noise

```javascript
// Bad: Tracking every validation error
if (!email.includes('@')) {
  Sentry.captureMessage('Invalid email format'); // Too noisy!
}

// Good: Only track unexpected errors
try {
  await complexOperation();
} catch (error) {
  if (error instanceof ValidationError) {
    // Handle validation errors gracefully
    showValidationMessage(error);
  } else {
    // Track unexpected errors
    Sentry.captureException(error);
  }
}
```

### 4. Missing Context

```javascript
// Bad: Vague error without context
Sentry.captureException(error);

// Good: Rich context
Sentry.captureException(error, {
  tags: {
    feature: 'checkout',
    step: 'payment',
  },
  extra: {
    userId: user.id,
    cartValue: cart.total,
    paymentMethod: payment.method,
  },
});
```

### 5. Not Using Source Maps

```javascript
// Bad: Minified stack traces (useless!)
Error: Cannot read property 'length' of undefined
  at t.render (bundle.min.js:1:45678)

// Good: With source maps
Error: Cannot read property 'length' of undefined
  at UserProfile.render (UserProfile.jsx:45:12)
```

## Best Practices

### 1. Set Appropriate Sample Rates

```javascript
Sentry.init({
  dsn: process.env.SENTRY_DSN,
  
  // Sample 100% of errors
  sampleRate: 1.0,
  
  // Sample 10% of transactions (performance)
  tracesSampleRate: 0.1,
  
  // Sample 10% of normal sessions, 100% of error sessions
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
});
```

### 2. Implement Error Prioritization

```javascript
Sentry.init({
  beforeSend(event, hint) {
    // Prioritize based on impact
    const originalException = hint.originalException;
    
    if (originalException?.name === 'ChunkLoadError') {
      // Lower priority for chunk load errors
      event.level = 'warning';
    } else if (event.request?.url?.includes('/api/payment')) {
      // High priority for payment errors
      event.level = 'error';
      event.tags = { ...event.tags, critical: 'true' };
    }
    
    return event;
  },
});
```

### 3. Use Breadcrumbs Effectively

```javascript
// Track user journey
Sentry.addBreadcrumb({
  category: 'auth',
  message: 'User logged in',
  level: 'info',
});

Sentry.addBreadcrumb({
  category: 'navigation',
  message: 'Navigated to checkout',
  level: 'info',
});

Sentry.addBreadcrumb({
  category: 'ui',
  message: 'Clicked payment button',
  level: 'info',
  data: {
    amount: 99.99,
    currency: 'USD',
  },
});

// When error occurs, breadcrumbs show the path that led to it
```

### 4. Monitor Error Trends

```javascript
// Create custom dashboards to track:
// - Error rate over time
// - Top errors by frequency
// - Errors by user/browser/device
// - Release comparison
// - Error distribution by feature

// Use Sentry's Discover feature or API
const getErrorTrends = async () => {
  const response = await fetch('https://sentry.io/api/0/organizations/my-org/events/', {
    headers: {
      Authorization: `Bearer ${SENTRY_TOKEN}`,
    },
    body: JSON.stringify({
      query: 'level:error',
      statsPeriod: '7d',
      field: ['count()', 'user.count()'],
      orderby: '-count()',
    }),
  });
  
  return response.json();
};
```

### 5. Integrate with Development Workflow

```yaml
# .github/workflows/pr-check.yml
name: PR Error Check

on: pull_request

jobs:
  check-errors:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Check for new errors in Sentry
        run: |
          # Compare error rates before/after PR
          npx @sentry/cli releases compare $BASE_VERSION $HEAD_VERSION
      
      - name: Comment on PR if errors increased
        if: failure()
        uses: actions/github-script@v6
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              body: '⚠️ Error rate increased in this PR. Check Sentry for details.'
            })
```

## When to Use/Not to Use

### When to Use Error Tracking

1. **Production Applications**: Monitor real user errors
2. **High-Traffic Applications**: Impossible to manually monitor logs
3. **Distributed Systems**: Errors across multiple services
4. **Customer-Facing Applications**: Impact on user experience
5. **Critical Business Logic**: Payment, authentication, data processing
6. **Multi-Platform Applications**: Web, mobile, desktop
7. **Third-Party Integrations**: Track integration failures
8. **Performance Monitoring**: Identify slow transactions

### When Error Tracking May Be Overkill

1. **Internal Tools**: Small user base, low criticality
2. **Prototype/MVP**: Early development, frequent changes
3. **Static Websites**: No dynamic logic or user interaction
4. **Budget Constraints**: Cost of monitoring service
5. **Simple Scripts**: One-off utilities, cron jobs
6. **Localhost Development**: Use browser console instead
7. **Legacy Systems**: No budget for modernization

## Interview Questions

### 1. What is the difference between logging and error tracking?

**Answer**: Logging captures all events (info, debug, errors) sequentially in log files, useful for debugging. Error tracking specifically monitors errors with automatic grouping, impact analysis (users affected), stack traces with source maps, breadcrumbs showing user actions leading to errors, and alerting. Error tracking provides structured error data optimized for triaging and fixing issues, while logs are general event records.

### 2. How do source maps help with error tracking?

**Answer**: Source maps map minified/transpiled production code back to original source code. Without them, stack traces show minified variable names and line numbers, making debugging nearly impossible. With source maps, error tracking tools display readable stack traces with actual file names, function names, and line numbers from your original source code. Source maps should be uploaded to error tracking services securely and not exposed publicly.

### 3. What are error boundaries in React and why are they important?

**Answer**: Error boundaries are React components that catch JavaScript errors in their child component tree, log errors, and display fallback UI instead of crashing the entire app. They catch errors during rendering, in lifecycle methods, and in constructors. Important because: prevents entire app crashes, provides better UX with fallback UI, allows granular error handling per feature, and integrates with error tracking services for automatic reporting.

### 4. How would you prioritize which errors to fix first?

**Answer**: Prioritize by: 1) User impact (users affected, frequency), 2) Severity (crashes vs minor issues), 3) Business impact (payment, authentication vs cosmetic), 4) New vs regression (new errors in recent releases), 5) Ease of fix (quick wins vs complex refactors). Use metrics like "users affected × frequency × severity" to calculate priority scores. Critical path errors (checkout, signup) always take precedence.

### 5. What information should be included in error context?

**Answer**: User context (ID, email, role), request context (URL, method, headers), environment (browser, OS, device), application state (Redux/Vuex state snapshot), custom context (feature flags, A/B test variants), breadcrumbs (user actions leading to error), release version, tags for filtering (feature, team), and performance metrics. Balance detail with privacy - never include passwords, tokens, or PII.

### 6. How do you prevent sensitive data from being sent to error tracking services?

**Answer**: Use `beforeSend` hooks to filter sensitive fields (passwords, tokens, credit cards), configure data scrubbing rules in error tracking service, redact sensitive URL parameters, implement allow-lists for what data to send, use environment variables for secrets (not hardcoded), sanitize request bodies, and regularly audit what data is being captured. Consider using encryption for truly sensitive contexts.

### 7. What are breadcrumbs and how do they help debug errors?

**Answer**: Breadcrumbs are a trail of events leading up to an error (navigation, HTTP requests, user clicks, console logs, state changes). They provide context about what the user was doing before the error occurred. Essential for reproducing issues: "User clicked checkout → API call to /payment failed → Error: Invalid card". Most error tracking tools automatically capture common breadcrumbs but custom breadcrumbs can be added for domain-specific events.

### 8. How would you implement error tracking in a microservices architecture?

**Answer**: Use distributed tracing with correlation IDs across all services. Each service reports errors to centralized error tracking with correlation ID. Include service name, version, and host in error context. Use tags to filter by service. Implement circuit breakers to prevent cascade failures. Use service mesh (Istio) for automatic trace propagation. Set up cross-service error dashboards. Alert on error rate spikes across services.

## Key Takeaways

1. **Error tracking is essential for production applications**: Provides visibility into real user issues
2. **Use source maps for meaningful stack traces**: Upload securely, don't expose publicly
3. **Implement error boundaries**: Prevent cascading failures and provide fallback UI
4. **Add rich context to errors**: User info, breadcrumbs, environment details
5. **Filter sensitive data**: Never send passwords, tokens, or PII
6. **Set up intelligent alerting**: High error rates, new errors, critical errors
7. **Use error grouping**: Automatic grouping by stack trace, custom fingerprinting
8. **Track releases**: Correlate errors with deployments
9. **Prioritize by impact**: Users affected and business criticality
10. **Integrate with workflow**: PR checks, Slack notifications, issue trackers

## Resources

### Error Tracking Services
- [Sentry](https://sentry.io/) - Open-source error tracking
- [Rollbar](https://rollbar.com/) - Real-time error monitoring
- [Bugsnag](https://www.bugsnag.com/) - Error monitoring and reporting
- [Airbrake](https://airbrake.io/) - Error tracking and performance monitoring
- [TrackJS](https://trackjs.com/) - JavaScript error monitoring

### Documentation
- [Sentry Documentation](https://docs.sentry.io/)
- [React Error Boundaries](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary)
- [Source Maps Specification](https://sourcemaps.info/spec.html)

### Tools
- [@sentry/webpack-plugin](https://www.npmjs.com/package/@sentry/webpack-plugin)
- [@sentry/cli](https://docs.sentry.io/product/cli/)
- [sentry-javascript](https://github.com/getsentry/sentry-javascript)

### Articles
- [Error Handling in JavaScript](https://javascript.info/error-handling)
- [Best Practices for Error Handling](https://www.toptal.com/nodejs/node-js-error-handling)
- [Source Maps: A Beginner's Guide](https://blog.logrocket.com/source-maps-a-beginners-guide/)
