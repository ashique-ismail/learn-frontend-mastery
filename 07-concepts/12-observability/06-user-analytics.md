# User Analytics

## The Idea

**In plain English:** User analytics is the practice of recording what people do on a website or app — like which buttons they click, which pages they visit, and where they stop and leave — so the team building it can understand what is working and what is not.

**Real-world analogy:** Imagine a store manager watching a security camera feed and keeping a tally sheet. Each time a shopper picks up a product, pauses at a shelf, or walks out without buying, the manager notes it down to figure out what layout changes would help more people reach the checkout.

- The security camera = the analytics script embedded in the app
- A shopper picking up a product = a user clicking a button or visiting a page (an "event")
- The tally sheet = the analytics dashboard where counts and patterns are displayed

---

## Overview

User analytics involves tracking, measuring, and analyzing user behavior to understand how people interact with your application. By collecting data on page views, events, user flows, and conversions, teams can make data-driven decisions to improve user experience, optimize features, and increase business metrics. Modern analytics platforms provide real-time insights, segmentation, funnel analysis, and cohort tracking to help product teams understand what users do and why.

## Table of Contents
- [Analytics Fundamentals](#analytics-fundamentals)
- [Google Analytics](#google-analytics)
- [Mixpanel](#mixpanel)
- [Event Tracking](#event-tracking)
- [Custom Events](#custom-events)
- [Funnels and Conversion Tracking](#funnels-and-conversion-tracking)
- [User Journeys](#user-journeys)
- [Segmentation](#segmentation)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use/Not to Use](#when-to-usenot-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Analytics Fundamentals

### Key Metrics

```javascript
// Common analytics metrics
const analyticsMetrics = {
  // Engagement metrics
  engagement: {
    pageViews: 50000,                    // Total page views
    uniquePageViews: 35000,              // Unique page views
    sessions: 15000,                     // User sessions
    avgSessionDuration: 180,             // Average session duration (seconds)
    bounceRate: 45,                      // Percentage of single-page sessions
    pagesPerSession: 3.2,                // Average pages viewed per session
  },
  
  // User metrics
  users: {
    totalUsers: 10000,                   // Total registered users
    activeUsers: 5000,                   // Active users (period)
    newUsers: 1000,                      // New users (period)
    returningUsers: 4000,                // Returning users
    dailyActiveUsers: 500,               // DAU
    monthlyActiveUsers: 5000,            // MAU
    stickinessRatio: 0.1,               // DAU/MAU ratio
  },
  
  // Conversion metrics
  conversion: {
    conversionRate: 5.5,                 // Percentage who complete goal
    goalCompletions: 275,                // Total goal completions
    goalValue: 27500,                    // Total goal value ($)
    transactionRevenue: 50000,           // E-commerce revenue
    averageOrderValue: 99.50,            // Average transaction value
  },
  
  // Retention metrics
  retention: {
    day1Retention: 45,                   // % returning day 1
    day7Retention: 28,                   // % returning day 7
    day30Retention: 15,                  // % returning day 30
    churnRate: 5,                        // % users who left
  },
};
```

### Analytics Data Model

```typescript
// analytics-types.ts
interface AnalyticsEvent {
  name: string;                          // Event name
  timestamp: number;                     // When it occurred
  userId?: string;                       // Identified user
  sessionId: string;                     // Session identifier
  properties?: Record<string, any>;      // Custom properties
  context?: AnalyticsContext;            // Environment context
}

interface AnalyticsContext {
  page: {
    url: string;
    path: string;
    title: string;
    referrer: string;
  };
  user: {
    id?: string;
    anonymousId: string;
    traits?: Record<string, any>;
  };
  device: {
    type: 'mobile' | 'tablet' | 'desktop';
    os: string;
    browser: string;
    screenWidth: number;
    screenHeight: number;
  };
  location: {
    country?: string;
    city?: string;
    region?: string;
    timezone?: string;
  };
  campaign?: {
    source?: string;
    medium?: string;
    name?: string;
    term?: string;
    content?: string;
  };
}

interface PageView {
  page: string;
  title: string;
  url: string;
  referrer: string;
  timestamp: number;
}

interface UserProfile {
  userId: string;
  traits: {
    email?: string;
    name?: string;
    createdAt?: string;
    plan?: string;
    [key: string]: any;
  };
}
```

## Google Analytics

### GA4 Setup (React)

```typescript
// ga4-setup.ts
import ReactGA from 'react-ga4';

export function initializeGA(measurementId: string) {
  ReactGA.initialize(measurementId, {
    gaOptions: {
      send_page_view: false, // We'll manually track page views
    },
  });
}

// Track page view
export function trackPageView(path: string, title?: string) {
  ReactGA.send({
    hitType: 'pageview',
    page: path,
    title: title || document.title,
  });
}

// Track event
export function trackEvent(
  category: string,
  action: string,
  label?: string,
  value?: number
) {
  ReactGA.event({
    category,
    action,
    label,
    value,
  });
}

// Set user properties
export function setUserProperties(properties: Record<string, any>) {
  ReactGA.set(properties);
}

// Track custom dimension
export function setCustomDimension(index: number, value: string) {
  ReactGA.set({ [`dimension${index}`]: value });
}

// Track timing
export function trackTiming(
  category: string,
  variable: string,
  value: number,
  label?: string
) {
  ReactGA.send({
    hitType: 'timing',
    timingCategory: category,
    timingVar: variable,
    timingValue: value,
    timingLabel: label,
  });
}

// App.tsx
import { useEffect } from 'react';
import { useLocation } from 'react-router-dom';
import { initializeGA, trackPageView } from './ga4-setup';

function App() {
  const location = useLocation();
  
  useEffect(() => {
    // Initialize GA on app load
    initializeGA(process.env.REACT_APP_GA_MEASUREMENT_ID || '');
  }, []);
  
  useEffect(() => {
    // Track page view on route change
    trackPageView(location.pathname);
  }, [location]);
  
  return <Routes />;
}

// Component usage
import { trackEvent } from './ga4-setup';

const CheckoutButton: React.FC = () => {
  const handleCheckout = () => {
    trackEvent('Ecommerce', 'Begin Checkout', 'Cart Value', cart.total);
    // Proceed with checkout
  };
  
  return <button onClick={handleCheckout}>Checkout</button>;
};
```

### GA4 E-commerce Tracking

```typescript
// ga4-ecommerce.ts
import ReactGA from 'react-ga4';

interface Product {
  item_id: string;
  item_name: string;
  price: number;
  quantity: number;
  item_category?: string;
  item_brand?: string;
}

// View item
export function trackViewItem(product: Product) {
  ReactGA.event('view_item', {
    items: [product],
    value: product.price,
    currency: 'USD',
  });
}

// Add to cart
export function trackAddToCart(product: Product) {
  ReactGA.event('add_to_cart', {
    items: [product],
    value: product.price * product.quantity,
    currency: 'USD',
  });
}

// Remove from cart
export function trackRemoveFromCart(product: Product) {
  ReactGA.event('remove_from_cart', {
    items: [product],
    value: product.price * product.quantity,
    currency: 'USD',
  });
}

// Begin checkout
export function trackBeginCheckout(items: Product[], total: number) {
  ReactGA.event('begin_checkout', {
    items,
    value: total,
    currency: 'USD',
  });
}

// Purchase
export function trackPurchase(
  transactionId: string,
  items: Product[],
  total: number,
  shipping?: number,
  tax?: number
) {
  ReactGA.event('purchase', {
    transaction_id: transactionId,
    value: total,
    currency: 'USD',
    tax: tax || 0,
    shipping: shipping || 0,
    items,
  });
}

// Usage
const ProductPage: React.FC<{ product: Product }> = ({ product }) => {
  useEffect(() => {
    trackViewItem(product);
  }, [product]);
  
  const handleAddToCart = () => {
    trackAddToCart(product);
    // Add to cart logic
  };
  
  return <button onClick={handleAddToCart}>Add to Cart</button>;
};
```

### Angular Google Analytics

```typescript
// analytics.service.ts
import { Injectable } from '@angular/core';
import { NavigationEnd, Router } from '@angular/router';
import { filter } from 'rxjs/operators';

declare let gtag: Function;

@Injectable({
  providedIn: 'root'
})
export class AnalyticsService {
  constructor(private router: Router) {
    this.initializePageTracking();
  }
  
  initializePageTracking(): void {
    this.router.events.pipe(
      filter(event => event instanceof NavigationEnd)
    ).subscribe((event: any) => {
      this.trackPageView(event.urlAfterRedirects);
    });
  }
  
  trackPageView(path: string): void {
    gtag('config', 'GA_MEASUREMENT_ID', {
      page_path: path
    });
  }
  
  trackEvent(
    action: string,
    category: string,
    label?: string,
    value?: number
  ): void {
    gtag('event', action, {
      event_category: category,
      event_label: label,
      value: value
    });
  }
  
  setUserId(userId: string): void {
    gtag('config', 'GA_MEASUREMENT_ID', {
      user_id: userId
    });
  }
  
  setUserProperties(properties: Record<string, any>): void {
    gtag('set', 'user_properties', properties);
  }
}

// Usage in component
import { Component } from '@angular/core';
import { AnalyticsService } from './analytics.service';

@Component({
  selector: 'app-product',
  templateUrl: './product.component.html'
})
export class ProductComponent {
  constructor(private analytics: AnalyticsService) {}
  
  addToCart(product: Product): void {
    this.analytics.trackEvent(
      'add_to_cart',
      'Ecommerce',
      product.name,
      product.price
    );
    
    // Add to cart logic
  }
}
```

## Mixpanel

### Mixpanel Setup

```typescript
// mixpanel-setup.ts
import mixpanel from 'mixpanel-browser';

export function initializeMixpanel(token: string) {
  mixpanel.init(token, {
    debug: process.env.NODE_ENV === 'development',
    track_pageview: true,
    persistence: 'localStorage',
  });
}

// Identify user
export function identifyUser(userId: string, properties?: Record<string, any>) {
  mixpanel.identify(userId);
  
  if (properties) {
    mixpanel.people.set(properties);
  }
}

// Track event
export function trackEvent(eventName: string, properties?: Record<string, any>) {
  mixpanel.track(eventName, properties);
}

// Set user properties
export function setUserProperties(properties: Record<string, any>) {
  mixpanel.people.set(properties);
}

// Increment property
export function incrementProperty(property: string, value: number = 1) {
  mixpanel.people.increment(property, value);
}

// Track revenue
export function trackRevenue(amount: number, properties?: Record<string, any>) {
  mixpanel.people.track_charge(amount, properties);
}

// Create alias (for anonymous to identified user)
export function createAlias(userId: string) {
  mixpanel.alias(userId);
}

// Reset (on logout)
export function reset() {
  mixpanel.reset();
}

// App initialization
import { useEffect } from 'react';
import { initializeMixpanel, identifyUser } from './mixpanel-setup';

function App() {
  useEffect(() => {
    initializeMixpanel(process.env.REACT_APP_MIXPANEL_TOKEN || '');
    
    // Identify user if logged in
    const user = getCurrentUser();
    if (user) {
      identifyUser(user.id, {
        $email: user.email,
        $name: user.name,
        $created: user.createdAt,
        plan: user.plan,
      });
    }
  }, []);
  
  return <Routes />;
}
```

### Mixpanel Event Tracking

```typescript
// mixpanel-events.ts
import { trackEvent, incrementProperty } from './mixpanel-setup';

// User signup
export function trackSignup(method: string) {
  trackEvent('Sign Up', {
    method,                                // 'email', 'google', 'facebook'
    timestamp: new Date().toISOString(),
  });
}

// Feature usage
export function trackFeatureUsage(feature: string, metadata?: Record<string, any>) {
  trackEvent('Feature Used', {
    feature,
    ...metadata,
  });
  
  incrementProperty(`${feature}_usage_count`);
}

// Content interaction
export function trackContentView(contentType: string, contentId: string) {
  trackEvent('Content Viewed', {
    content_type: contentType,
    content_id: contentId,
  });
}

// Search
export function trackSearch(query: string, resultsCount: number) {
  trackEvent('Search Performed', {
    query,
    results_count: resultsCount,
    has_results: resultsCount > 0,
  });
}

// Form submission
export function trackFormSubmission(formName: string, success: boolean) {
  trackEvent('Form Submitted', {
    form_name: formName,
    success,
  });
}

// Error
export function trackError(errorType: string, errorMessage: string) {
  trackEvent('Error Occurred', {
    error_type: errorType,
    error_message: errorMessage,
    page: window.location.pathname,
  });
}

// Usage in components
const SearchComponent: React.FC = () => {
  const handleSearch = async (query: string) => {
    const results = await searchAPI(query);
    trackSearch(query, results.length);
    setResults(results);
  };
  
  return <SearchInput onSearch={handleSearch} />;
};

const FeatureComponent: React.FC = () => {
  const handleExport = () => {
    trackFeatureUsage('data_export', {
      format: 'CSV',
      records: data.length,
    });
    
    exportData(data);
  };
  
  return <button onClick={handleExport}>Export</button>;
};
```

## Event Tracking

### Event Tracking Architecture

```typescript
// analytics-manager.ts
interface AnalyticsProvider {
  name: string;
  trackEvent(name: string, properties?: Record<string, any>): void;
  trackPageView(path: string, properties?: Record<string, any>): void;
  identifyUser(userId: string, traits?: Record<string, any>): void;
}

class AnalyticsManager {
  private providers: AnalyticsProvider[] = [];
  private eventQueue: Array<{ name: string; properties: any }> = [];
  private isInitialized = false;
  
  addProvider(provider: AnalyticsProvider): void {
    this.providers.push(provider);
  }
  
  initialize(): void {
    this.isInitialized = true;
    
    // Flush queued events
    this.eventQueue.forEach(event => {
      this.trackEvent(event.name, event.properties);
    });
    this.eventQueue = [];
  }
  
  trackEvent(name: string, properties?: Record<string, any>): void {
    if (!this.isInitialized) {
      // Queue events until initialized
      this.eventQueue.push({ name, properties });
      return;
    }
    
    // Send to all providers
    this.providers.forEach(provider => {
      try {
        provider.trackEvent(name, properties);
      } catch (error) {
        console.error(`Failed to track event with ${provider.name}:`, error);
      }
    });
  }
  
  trackPageView(path: string, properties?: Record<string, any>): void {
    this.providers.forEach(provider => {
      try {
        provider.trackPageView(path, properties);
      } catch (error) {
        console.error(`Failed to track page view with ${provider.name}:`, error);
      }
    });
  }
  
  identifyUser(userId: string, traits?: Record<string, any>): void {
    this.providers.forEach(provider => {
      try {
        provider.identifyUser(userId, traits);
      } catch (error) {
        console.error(`Failed to identify user with ${provider.name}:`, error);
      }
    });
  }
}

// Singleton instance
export const analytics = new AnalyticsManager();

// Google Analytics provider
class GAProvider implements AnalyticsProvider {
  name = 'Google Analytics';
  
  trackEvent(name: string, properties?: Record<string, any>): void {
    ReactGA.event({ action: name, ...properties });
  }
  
  trackPageView(path: string, properties?: Record<string, any>): void {
    ReactGA.send({ hitType: 'pageview', page: path, ...properties });
  }
  
  identifyUser(userId: string, traits?: Record<string, any>): void {
    ReactGA.set({ userId, ...traits });
  }
}

// Mixpanel provider
class MixpanelProvider implements AnalyticsProvider {
  name = 'Mixpanel';
  
  trackEvent(name: string, properties?: Record<string, any>): void {
    mixpanel.track(name, properties);
  }
  
  trackPageView(path: string, properties?: Record<string, any>): void {
    mixpanel.track('Page Viewed', { page: path, ...properties });
  }
  
  identifyUser(userId: string, traits?: Record<string, any>): void {
    mixpanel.identify(userId);
    if (traits) mixpanel.people.set(traits);
  }
}

// Initialize
analytics.addProvider(new GAProvider());
analytics.addProvider(new MixpanelProvider());
analytics.initialize();

// Usage
import { analytics } from './analytics-manager';

analytics.trackEvent('Button Clicked', {
  button: 'Sign Up',
  location: 'Header',
});
```

## Custom Events

### Event Taxonomy

```typescript
// event-definitions.ts
// Define a consistent event taxonomy

export const Events = {
  // User events
  USER_SIGNED_UP: 'User Signed Up',
  USER_LOGGED_IN: 'User Logged In',
  USER_LOGGED_OUT: 'User Logged Out',
  USER_UPDATED_PROFILE: 'User Updated Profile',
  
  // Navigation events
  PAGE_VIEWED: 'Page Viewed',
  LINK_CLICKED: 'Link Clicked',
  
  // Feature events
  FEATURE_VIEWED: 'Feature Viewed',
  FEATURE_USED: 'Feature Used',
  FEATURE_COMPLETED: 'Feature Completed',
  
  // E-commerce events
  PRODUCT_VIEWED: 'Product Viewed',
  PRODUCT_ADDED_TO_CART: 'Product Added to Cart',
  PRODUCT_REMOVED_FROM_CART: 'Product Removed from Cart',
  CHECKOUT_STARTED: 'Checkout Started',
  CHECKOUT_COMPLETED: 'Checkout Completed',
  PAYMENT_INFO_ENTERED: 'Payment Info Entered',
  PURCHASE_COMPLETED: 'Purchase Completed',
  
  // Content events
  CONTENT_SEARCHED: 'Content Searched',
  CONTENT_FILTERED: 'Content Filtered',
  CONTENT_SHARED: 'Content Shared',
  CONTENT_DOWNLOADED: 'Content Downloaded',
  
  // Engagement events
  VIDEO_PLAYED: 'Video Played',
  VIDEO_COMPLETED: 'Video Completed',
  FORM_SUBMITTED: 'Form Submitted',
  FILE_UPLOADED: 'File Uploaded',
  
  // Error events
  ERROR_OCCURRED: 'Error Occurred',
  ERROR_RECOVERED: 'Error Recovered',
} as const;

// Event properties interfaces
export interface UserSignedUpProperties {
  method: 'email' | 'google' | 'facebook' | 'apple';
  referrer?: string;
}

export interface ProductViewedProperties {
  product_id: string;
  product_name: string;
  category: string;
  price: number;
  currency: string;
}

export interface CheckoutStartedProperties {
  cart_value: number;
  item_count: number;
  currency: string;
  items: Array<{
    product_id: string;
    product_name: string;
    quantity: number;
    price: number;
  }>;
}

// Type-safe event tracking
export function trackTypedEvent<T extends keyof typeof Events>(
  event: T,
  properties?: Record<string, any>
): void {
  analytics.trackEvent(Events[event], properties);
}

// Usage with type safety
trackTypedEvent('USER_SIGNED_UP', {
  method: 'email',
  referrer: document.referrer,
} as UserSignedUpProperties);
```

### Higher-Order Component for Analytics

```typescript
// withAnalytics.tsx
import React, { useEffect } from 'react';
import { analytics } from './analytics-manager';

interface WithAnalyticsProps {
  trackOnMount?: {
    event: string;
    properties?: Record<string, any>;
  };
  trackOnUnmount?: {
    event: string;
    properties?: Record<string, any>;
  };
}

export function withAnalytics<P extends object>(
  Component: React.ComponentType<P>,
  config: WithAnalyticsProps
) {
  return function AnalyticsWrapper(props: P) {
    useEffect(() => {
      if (config.trackOnMount) {
        analytics.trackEvent(
          config.trackOnMount.event,
          config.trackOnMount.properties
        );
      }
      
      return () => {
        if (config.trackOnUnmount) {
          analytics.trackEvent(
            config.trackOnUnmount.event,
            config.trackOnUnmount.properties
          );
        }
      };
    }, []);
    
    return <Component {...props} />;
  };
}

// Usage
const DashboardPage = () => <div>Dashboard</div>;

export default withAnalytics(DashboardPage, {
  trackOnMount: {
    event: 'Page Viewed',
    properties: { page: 'Dashboard' },
  },
  trackOnUnmount: {
    event: 'Page Left',
    properties: { page: 'Dashboard' },
  },
});
```

## Funnels and Conversion Tracking

### Funnel Definition

```typescript
// funnel-tracker.ts
interface FunnelStep {
  name: string;
  event: string;
  required: boolean;
}

class FunnelTracker {
  private funnels: Map<string, FunnelStep[]> = new Map();
  private userProgress: Map<string, Set<string>> = new Map();
  
  defineFunnel(funnelName: string, steps: FunnelStep[]): void {
    this.funnels.set(funnelName, steps);
  }
  
  trackStep(funnelName: string, stepEvent: string, userId: string): void {
    const steps = this.funnels.get(funnelName);
    if (!steps) return;
    
    const step = steps.find(s => s.event === stepEvent);
    if (!step) return;
    
    // Track user progress
    const progressKey = `${userId}:${funnelName}`;
    if (!this.userProgress.has(progressKey)) {
      this.userProgress.set(progressKey, new Set());
    }
    
    this.userProgress.get(progressKey)!.add(stepEvent);
    
    // Track event
    analytics.trackEvent(stepEvent, {
      funnel: funnelName,
      step: step.name,
      step_number: steps.indexOf(step) + 1,
      total_steps: steps.length,
    });
    
    // Check if funnel completed
    const progress = this.userProgress.get(progressKey)!;
    const allSteps = steps.map(s => s.event);
    const completed = allSteps.every(event => progress.has(event));
    
    if (completed) {
      analytics.trackEvent('Funnel Completed', {
        funnel: funnelName,
      });
      
      // Clear progress
      this.userProgress.delete(progressKey);
    }
  }
  
  getFunnelProgress(funnelName: string, userId: string): {
    completed: string[];
    remaining: string[];
    completion_rate: number;
  } {
    const steps = this.funnels.get(funnelName);
    if (!steps) return { completed: [], remaining: [], completion_rate: 0 };
    
    const progressKey = `${userId}:${funnelName}`;
    const progress = this.userProgress.get(progressKey) || new Set();
    
    const completed = steps.filter(s => progress.has(s.event)).map(s => s.name);
    const remaining = steps.filter(s => !progress.has(s.event)).map(s => s.name);
    
    return {
      completed,
      remaining,
      completion_rate: (completed.length / steps.length) * 100,
    };
  }
}

export const funnelTracker = new FunnelTracker();

// Define checkout funnel
funnelTracker.defineFunnel('checkout', [
  { name: 'View Cart', event: 'Cart Viewed', required: true },
  { name: 'Enter Shipping', event: 'Shipping Info Entered', required: true },
  { name: 'Enter Payment', event: 'Payment Info Entered', required: true },
  { name: 'Complete Purchase', event: 'Purchase Completed', required: true },
]);

// Define signup funnel
funnelTracker.defineFunnel('signup', [
  { name: 'Visit Signup', event: 'Signup Page Viewed', required: true },
  { name: 'Enter Email', event: 'Email Entered', required: true },
  { name: 'Verify Email', event: 'Email Verified', required: true },
  { name: 'Complete Profile', event: 'Profile Completed', required: true },
]);

// Usage
const CheckoutFlow: React.FC = () => {
  const userId = useCurrentUserId();
  
  const handleShippingSubmit = () => {
    funnelTracker.trackStep('checkout', 'Shipping Info Entered', userId);
    // Continue to payment
  };
  
  const handlePaymentSubmit = () => {
    funnelTracker.trackStep('checkout', 'Payment Info Entered', userId);
    // Process payment
  };
  
  const handlePurchaseComplete = () => {
    funnelTracker.trackStep('checkout', 'Purchase Completed', userId);
    // Show confirmation
  };
  
  return <div>Checkout</div>;
};
```

### Conversion Rate Optimization

```typescript
// conversion-tracker.ts
class ConversionTracker {
  trackConversion(
    goalName: string,
    value?: number,
    metadata?: Record<string, any>
  ): void {
    analytics.trackEvent('Goal Completed', {
      goal: goalName,
      value,
      ...metadata,
    });
    
    // Also track in provider-specific format
    if (typeof gtag !== 'undefined') {
      gtag('event', 'conversion', {
        send_to: `GA_MEASUREMENT_ID/${goalName}`,
        value,
        currency: 'USD',
      });
    }
  }
  
  trackMicroConversion(action: string, metadata?: Record<string, any>): void {
    analytics.trackEvent('Micro Conversion', {
      action,
      ...metadata,
    });
  }
}

export const conversionTracker = new ConversionTracker();

// Track macro conversions (primary goals)
conversionTracker.trackConversion('Purchase', 99.99, {
  product_id: '12345',
  category: 'Electronics',
});

conversionTracker.trackConversion('Signup', undefined, {
  plan: 'Premium',
});

// Track micro conversions (secondary goals)
conversionTracker.trackMicroConversion('Newsletter Signup', {
  source: 'Footer',
});

conversionTracker.trackMicroConversion('Demo Request', {
  company_size: '50-100',
});
```

## User Journeys

### Journey Mapping

```typescript
// journey-tracker.ts
interface JourneyEvent {
  event: string;
  timestamp: number;
  properties?: Record<string, any>;
}

class JourneyTracker {
  private journeys: Map<string, JourneyEvent[]> = new Map();
  
  startJourney(userId: string): void {
    this.journeys.set(userId, []);
  }
  
  addEvent(
    userId: string,
    event: string,
    properties?: Record<string, any>
  ): void {
    if (!this.journeys.has(userId)) {
      this.startJourney(userId);
    }
    
    const journey = this.journeys.get(userId)!;
    journey.push({
      event,
      timestamp: Date.now(),
      properties,
    });
    
    // Keep last 50 events
    if (journey.length > 50) {
      journey.shift();
    }
  }
  
  getJourney(userId: string): JourneyEvent[] {
    return this.journeys.get(userId) || [];
  }
  
  analyzeJourney(userId: string): {
    duration: number;
    eventCount: number;
    uniqueEvents: number;
    path: string[];
  } {
    const journey = this.getJourney(userId);
    
    if (journey.length === 0) {
      return {
        duration: 0,
        eventCount: 0,
        uniqueEvents: 0,
        path: [],
      };
    }
    
    const first = journey[0];
    const last = journey[journey.length - 1];
    const duration = last.timestamp - first.timestamp;
    
    const uniqueEvents = new Set(journey.map(e => e.event)).size;
    const path = journey.map(e => e.event);
    
    return {
      duration,
      eventCount: journey.length,
      uniqueEvents,
      path,
    };
  }
  
  findCommonPaths(minSupport: number = 5): string[][] {
    // Find common event sequences across all users
    const pathCounts = new Map<string, number>();
    
    this.journeys.forEach(journey => {
      const path = journey.map(e => e.event).join(' → ');
      pathCounts.set(path, (pathCounts.get(path) || 0) + 1);
    });
    
    // Filter by minimum support
    return Array.from(pathCounts.entries())
      .filter(([_, count]) => count >= minSupport)
      .sort((a, b) => b[1] - a[1])
      .map(([path, _]) => path.split(' → '));
  }
}

export const journeyTracker = new JourneyTracker();

// Track user journey
const App: React.FC = () => {
  const userId = useCurrentUserId();
  
  useEffect(() => {
    if (userId) {
      journeyTracker.startJourney(userId);
    }
  }, [userId]);
  
  // Track events automatically
  useEffect(() => {
    const handleEvent = (event: Event) => {
      if (userId) {
        journeyTracker.addEvent(userId, event.type, {
          target: (event.target as HTMLElement).tagName,
        });
      }
    };
    
    window.addEventListener('click', handleEvent);
    window.addEventListener('submit', handleEvent);
    
    return () => {
      window.removeEventListener('click', handleEvent);
      window.removeEventListener('submit', handleEvent);
    };
  }, [userId]);
  
  return <Routes />;
};
```

## Segmentation

### User Segmentation

```typescript
// segmentation.ts
interface Segment {
  name: string;
  condition: (user: User) => boolean;
}

class SegmentationManager {
  private segments: Segment[] = [];
  
  defineSegment(name: string, condition: (user: User) => boolean): void {
    this.segments.push({ name, condition });
  }
  
  getUserSegments(user: User): string[] {
    return this.segments
      .filter(segment => segment.condition(user))
      .map(segment => segment.name);
  }
  
  trackEventWithSegments(
    user: User,
    event: string,
    properties?: Record<string, any>
  ): void {
    const segments = this.getUserSegments(user);
    
    analytics.trackEvent(event, {
      ...properties,
      user_segments: segments,
    });
  }
}

export const segmentation = new SegmentationManager();

// Define segments
segmentation.defineSegment('New User', (user) => {
  const daysSinceSignup = (Date.now() - user.createdAt) / (1000 * 60 * 60 * 24);
  return daysSinceSignup < 7;
});

segmentation.defineSegment('Power User', (user) => {
  return user.monthlyActiveTime > 10 * 60 * 60; // 10 hours
});

segmentation.defineSegment('Premium', (user) => {
  return user.plan === 'premium' || user.plan === 'enterprise';
});

segmentation.defineSegment('At Risk', (user) => {
  const daysSinceLastActive = (Date.now() - user.lastActiveAt) / (1000 * 60 * 60 * 24);
  return daysSinceLastActive > 14 && daysSinceLastActive < 30;
});

segmentation.defineSegment('High Value', (user) => {
  return user.lifetimeValue > 1000;
});

// Usage
const user = getCurrentUser();
const segments = segmentation.getUserSegments(user);

console.log('User segments:', segments);
// ['Power User', 'Premium', 'High Value']

// Track events with segments
segmentation.trackEventWithSegments(user, 'Feature Used', {
  feature: 'Advanced Analytics',
});
```

## Common Mistakes

### 1. Tracking Too Much

```typescript
// Bad: Track everything (noisy, expensive)
document.addEventListener('mousemove', (e) => {
  analytics.trackEvent('Mouse Moved', { x: e.clientX, y: e.clientY });
});

// Good: Track meaningful interactions
button.addEventListener('click', () => {
  analytics.trackEvent('CTA Clicked', {
    button: 'Sign Up',
    location: 'Hero Section',
  });
});
```

### 2. Inconsistent Event Names

```typescript
// Bad: Inconsistent naming
analytics.trackEvent('user_signup');
analytics.trackEvent('UserLogin');
analytics.trackEvent('User Logged Out');

// Good: Consistent naming convention
analytics.trackEvent('User Signed Up');
analytics.trackEvent('User Logged In');
analytics.trackEvent('User Logged Out');
```

### 3. Missing Context

```typescript
// Bad: No context
analytics.trackEvent('Button Clicked');

// Good: Rich context
analytics.trackEvent('Button Clicked', {
  button: 'Add to Cart',
  product_id: '12345',
  product_name: 'Wireless Mouse',
  price: 29.99,
  location: 'Product Page',
});
```

### 4. Not Respecting Privacy

```typescript
// Bad: Track PII without consent
analytics.trackEvent('User Signed Up', {
  email: user.email,
  phone: user.phone,
  ssn: user.ssn, // NEVER!
});

// Good: Hash or pseudonymize PII
analytics.trackEvent('User Signed Up', {
  user_id: user.id, // Use ID, not email
  plan: user.plan,
  referrer: document.referrer,
});
```

### 5. Blocking UI with Analytics

```typescript
// Bad: Synchronous, blocks UI
function handleClick() {
  analytics.trackEvent('Button Clicked'); // Blocks if slow
  processAction();
}

// Good: Asynchronous, non-blocking
async function handleClick() {
  // Fire and forget
  analytics.trackEvent('Button Clicked');
  processAction(); // Don't wait
}
```

## Best Practices

### 1. Implement Event Taxonomy

```typescript
// Create a documented event taxonomy
/**
 * Event Naming Convention:
 * - Use Object-Action format: "Product Viewed", "Cart Emptied"
 * - Use Title Case: "User Signed Up" not "user_signed_up"
 * - Be specific: "Payment Failed" not "Error"
 * - Use past tense: "Item Purchased" not "Purchase Item"
 */

export const EventTaxonomy = {
  // User lifecycle
  USER_SIGNED_UP: 'User Signed Up',
  USER_LOGGED_IN: 'User Logged In',
  USER_LOGGED_OUT: 'User Logged Out',
  USER_DELETED_ACCOUNT: 'User Deleted Account',
  
  // Product interaction
  PRODUCT_VIEWED: 'Product Viewed',
  PRODUCT_COMPARED: 'Product Compared',
  PRODUCT_REVIEWED: 'Product Reviewed',
  
  // Purchasing
  CART_ITEM_ADDED: 'Cart Item Added',
  CART_ITEM_REMOVED: 'Cart Item Removed',
  CHECKOUT_STARTED: 'Checkout Started',
  PAYMENT_INFO_ENTERED: 'Payment Info Entered',
  ORDER_COMPLETED: 'Order Completed',
  ORDER_CANCELED: 'Order Canceled',
};
```

### 2. Use Batching for Performance

```typescript
// analytics-queue.ts
class AnalyticsQueue {
  private queue: Array<{ event: string; properties: any }> = [];
  private flushInterval = 5000; // 5 seconds
  private maxQueueSize = 20;
  
  constructor() {
    setInterval(() => this.flush(), this.flushInterval);
  }
  
  enqueue(event: string, properties?: Record<string, any>): void {
    this.queue.push({ event, properties });
    
    if (this.queue.length >= this.maxQueueSize) {
      this.flush();
    }
  }
  
  flush(): void {
    if (this.queue.length === 0) return;
    
    const events = [...this.queue];
    this.queue = [];
    
    // Batch send to analytics provider
    this.sendBatch(events);
  }
  
  private sendBatch(events: Array<{ event: string; properties: any }>): void {
    // Send to analytics API as batch
    fetch('/api/analytics/batch', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ events }),
      keepalive: true, // Important for page unload
    });
  }
}

export const analyticsQueue = new AnalyticsQueue();
```

### 3. A/B Test Tracking

```typescript
// ab-test-tracker.ts
interface ABTest {
  name: string;
  variant: string;
}

class ABTestTracker {
  private activeTests: Map<string, string> = new Map();
  
  assignVariant(testName: string, variant: string): void {
    this.activeTests.set(testName, variant);
    
    // Track assignment
    analytics.trackEvent('AB Test Assigned', {
      test: testName,
      variant,
    });
    
    // Set as user property
    analytics.setUserProperties({
      [`ab_test_${testName}`]: variant,
    });
  }
  
  trackConversion(testName: string, metadata?: Record<string, any>): void {
    const variant = this.activeTests.get(testName);
    
    if (!variant) {
      console.warn(`No active variant for test: ${testName}`);
      return;
    }
    
    analytics.trackEvent('AB Test Conversion', {
      test: testName,
      variant,
      ...metadata,
    });
  }
  
  getActiveTests(): ABTest[] {
    return Array.from(this.activeTests.entries()).map(([name, variant]) => ({
      name,
      variant,
    }));
  }
}

export const abTestTracker = new ABTestTracker();

// Usage
const ButtonVariant: React.FC = () => {
  useEffect(() => {
    // Assign variant (usually done by feature flag service)
    const variant = Math.random() < 0.5 ? 'control' : 'treatment';
    abTestTracker.assignVariant('button_color_test', variant);
  }, []);
  
  const handleClick = () => {
    // Track conversion
    abTestTracker.trackConversion('button_color_test', {
      action: 'signup_clicked',
    });
    
    // Proceed with action
    signUp();
  };
  
  return <button onClick={handleClick}>Sign Up</button>;
};
```

### 4. Cohort Analysis

```typescript
// cohort-tracker.ts
class CohortTracker {
  trackCohort(userId: string, cohortDate: string): void {
    analytics.setUserProperties({
      cohort: cohortDate, // e.g., '2024-06-15'
    });
  }
  
  trackRetention(userId: string, dayNumber: number): void {
    analytics.trackEvent('User Returned', {
      day: dayNumber,
      is_retained: true,
    });
  }
}

export const cohortTracker = new CohortTracker();

// Track cohort on signup
function handleSignup(user: User) {
  const cohortDate = new Date().toISOString().split('T')[0];
  cohortTracker.trackCohort(user.id, cohortDate);
}

// Track retention
function trackDailyActive(user: User) {
  const signupDate = new Date(user.createdAt);
  const now = new Date();
  const daysSinceSignup = Math.floor(
    (now.getTime() - signupDate.getTime()) / (1000 * 60 * 60 * 24)
  );
  
  cohortTracker.trackRetention(user.id, daysSinceSignup);
}
```

### 5. GDPR Compliance

```typescript
// gdpr-compliance.ts
class GDPRCompliance {
  private consentGiven = false;
  private queuedEvents: Array<{ event: string; properties: any }> = [];
  
  requestConsent(): Promise<boolean> {
    return new Promise((resolve) => {
      // Show consent banner
      showConsentBanner({
        onAccept: () => {
          this.consentGiven = true;
          this.flushQueuedEvents();
          resolve(true);
        },
        onReject: () => {
          this.consentGiven = false;
          this.queuedEvents = [];
          resolve(false);
        },
      });
    });
  }
  
  trackEvent(event: string, properties?: Record<string, any>): void {
    if (!this.consentGiven) {
      // Queue event until consent given
      this.queuedEvents.push({ event, properties });
      return;
    }
    
    analytics.trackEvent(event, properties);
  }
  
  private flushQueuedEvents(): void {
    this.queuedEvents.forEach(({ event, properties }) => {
      analytics.trackEvent(event, properties);
    });
    this.queuedEvents = [];
  }
  
  optOut(): void {
    this.consentGiven = false;
    this.queuedEvents = [];
    
    // Disable tracking
    if (typeof gtag !== 'undefined') {
      gtag('consent', 'update', {
        analytics_storage: 'denied',
      });
    }
    
    if (typeof mixpanel !== 'undefined') {
      mixpanel.opt_out_tracking();
    }
  }
}

export const gdprCompliance = new GDPRCompliance();

// Request consent on app load
gdprCompliance.requestConsent();
```

## When to Use/Not to Use

### When to Use Analytics

1. **Product Decisions**: Data-driven feature development
2. **User Behavior Understanding**: How users interact with your app
3. **Conversion Optimization**: Improve signup/purchase flows
4. **Performance Monitoring**: Track load times and errors
5. **A/B Testing**: Measure experiment results
6. **Retention Analysis**: Understand why users leave
7. **Marketing Attribution**: Track campaign effectiveness
8. **Feature Adoption**: See which features are used

### When Analytics May Be Overkill

1. **Internal Tools**: Small user base
2. **Prototypes**: Early development, frequent changes
3. **Privacy-Critical Applications**: Healthcare, finance (need special compliance)
4. **Static Websites**: No user interaction
5. **Offline-First Apps**: Limited connectivity
6. **Simple Landing Pages**: One call-to-action
7. **Budget Constraints**: Premium analytics tools expensive

## Interview Questions

### 1. What's the difference between Google Analytics and Mixpanel?

**Answer**: Google Analytics focuses on website traffic and marketing metrics (page views, sessions, acquisition channels). Mixpanel focuses on product analytics and user behavior (events, funnels, retention, cohorts). GA is better for marketing and SEO. Mixpanel is better for product teams understanding user actions. GA tracks sessions, Mixpanel tracks users over time.

### 2. How do you track a multi-step checkout funnel?

**Answer**: Define funnel steps (view cart, enter shipping, enter payment, complete purchase). Track each step as a distinct event with consistent properties (cart_value, items, step_number). Use funnel analysis tools to visualize drop-off rates. Identify where users abandon the process. Implement recovery mechanisms (email cart abandonment, simplify problematic steps).

### 3. What is cohort analysis and why is it useful?

**Answer**: Cohort analysis groups users by signup date (or other shared characteristic) and tracks their behavior over time. Shows retention rates (Day 1, Day 7, Day 30). Helps identify if product improvements increase retention for new cohorts vs old cohorts. Useful for measuring impact of changes, understanding user lifecycle, and predicting churn.

### 4. How would you implement privacy-compliant analytics?

**Answer**: Request consent before tracking (GDPR). Anonymize IP addresses. Don't track PII without consent. Use cookie-less tracking where possible. Provide opt-out mechanism. Hash sensitive identifiers. Implement data retention policies (auto-delete old data). Use privacy-focused analytics (Plausible, Fathom) or self-host. Document in privacy policy.

### 5. What events should you track for a SaaS application?

**Answer**: User lifecycle (signup, login, logout, delete account). Feature usage (which features, how often). Onboarding completion. Subscription events (trial start, upgrade, downgrade, cancel). Key actions (create project, invite teammate, export data). Errors and issues. Time-to-value metrics. Engagement (DAU, MAU, session duration).

### 6. How do you measure feature adoption?

**Answer**: Track feature discovery (how many users see it), usage (how many use it), frequency (how often), and depth (how many actions within feature). Calculate adoption rate (users who used / users who could use). Track over time to see growth. Segment by user type. Compare to other features. Use in-app surveys to understand why users don't adopt.

### 7. What is event-driven architecture for analytics?

**Answer**: System where analytics events are first-class entities. Events are emitted by application, captured by event bus, and routed to multiple destinations (analytics, data warehouse, real-time processing). Decouples tracking from analytics providers. Enables data ownership and flexibility. Common pattern: app → Segment/RudderStack → GA/Mixpanel/warehouse.

### 8. How would you track user engagement for a mobile-first web app?

**Answer**: Use RUM (Real User Monitoring) for performance. Track touch interactions, not just clicks. Monitor offline usage (service worker events). Track app install prompts. Use session replay carefully (mobile data concerns). Focus on mobile-specific metrics (page load on 3G, scroll depth on small screens). Segment by device type. Consider native app analytics SDKs if PWA.

## Key Takeaways

1. **Define clear event taxonomy**: Consistent naming and structure
2. **Track user journeys, not just events**: Understand context and flow
3. **Implement funnels for critical paths**: Identify drop-off points
4. **Use segmentation for insights**: Different users behave differently
5. **Respect user privacy**: GDPR compliance, consent management
6. **Don't track everything**: Focus on actionable metrics
7. **Batch events for performance**: Don't block UI with analytics
8. **Combine multiple analytics tools**: Marketing (GA) + Product (Mixpanel)
9. **Use analytics for decisions**: Data-driven product development
10. **Monitor analytics health**: Ensure tracking is working correctly

## Resources

### Analytics Platforms
- [Google Analytics 4](https://analytics.google.com/)
- [Mixpanel](https://mixpanel.com/)
- [Amplitude](https://amplitude.com/)
- [Heap](https://heap.io/)
- [PostHog](https://posthog.com/) - Open-source

### Privacy-Focused
- [Plausible](https://plausible.io/)
- [Fathom](https://usefathom.com/)
- [Umami](https://umami.is/)

### Event Infrastructure
- [Segment](https://segment.com/)
- [RudderStack](https://rudderstack.com/)
- [Snowplow](https://snowplow.io/)

### Documentation
- [GA4 Documentation](https://developers.google.com/analytics/devguides/collection/ga4)
- [Mixpanel Documentation](https://developer.mixpanel.com/)
- [Segment Spec](https://segment.com/docs/connections/spec/)

### Articles
- [Event Taxonomy Guide](https://segment.com/academy/collecting-data/naming-conventions-for-clean-data/)
- [Product Analytics Guide](https://mixpanel.com/blog/product-analytics-guide/)
- [GDPR Compliance for Analytics](https://gdpr.eu/cookies/)
