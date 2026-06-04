# Feature Flags and Progressive Rollout

## Overview

Feature flags (also called feature toggles) are a deployment technique that separates code deployment from feature activation. You ship code with a new feature hidden behind a flag, then activate it for specific users or percentages without deploying again. This enables continuous deployment, canary releases, A/B testing, instant rollbacks, and kill switches. This guide covers flag patterns, major tools (LaunchDarkly, GrowthBook), targeting rules, and the architectural considerations for a flag system.

## Why Feature Flags

```
Without feature flags:
  Code complete → branch merge → deploy → all users see feature
  Problem: you can't test at scale before full rollout
  Problem: rolling back requires a revert commit and redeploy (slow!)

With feature flags:
  Code complete → merge → deploy (feature OFF) → test in production
                                → enable for 1% → monitor
                                → enable for 10% → monitor
                                → enable for 100% → or rollback instantly

Benefits:
  ✓ Dark launch: ship code before activating it
  ✓ Canary release: test with a small percentage of real users
  ✓ Kill switch: disable broken features without deploying
  ✓ A/B testing: measure impact of changes with real users
  ✓ Beta programs: enable for specific users/teams
  ✓ Ops flags: disable expensive features under load
```

## Flag Types

```
Release flags (temporary):
  Enable a new feature for gradual rollout
  Should be removed after 100% rollout
  Lifetime: days to weeks

Experiment flags (temporary):
  A/B test — measure which variant performs better
  Removed after experiment concludes
  Lifetime: days to weeks

Ops flags (semi-permanent):
  Kill switch for performance-intensive features
  Example: "disable real-time sync under high load"
  Lifetime: indefinite (but rarely changes)

Permission flags (permanent):
  Entitlement-based access control
  Example: "premium feature enabled for paid users"
  Lifetime: indefinite (part of the product)
```

## Simple In-App Flag Implementation

Before reaching for a third-party service, understand what a flag system is at its core:

```typescript
// Minimal feature flag system
interface FeatureFlags {
  newCheckoutFlow: boolean;
  experimentalSearch: boolean;
  premiumDashboard: boolean;
}

// Simplest: environment variable flags (build-time, not runtime)
const flags: FeatureFlags = {
  newCheckoutFlow: import.meta.env.VITE_FF_NEW_CHECKOUT === 'true',
  experimentalSearch: import.meta.env.VITE_FF_SEARCH === 'true',
  premiumDashboard: import.meta.env.VITE_FF_PREMIUM === 'true',
};

// Usage
if (flags.newCheckoutFlow) {
  return <NewCheckoutFlow />;
} else {
  return <LegacyCheckoutFlow />;
}
```

Build-time flags require a rebuild to toggle — fine for development, insufficient for production.

## Runtime Flag System

```typescript
// Runtime flags fetched from a server — togglable without deployment

interface FlagDefinition {
  key: string;
  defaultValue: boolean | string | number;
  description: string;
}

class FeatureFlagClient {
  private flags: Map<string, boolean | string | number> = new Map();
  private userId: string;

  constructor(userId: string) {
    this.userId = userId;
  }

  async initialize(): Promise<void> {
    // Fetch flags from your flag service on app startup
    const response = await fetch(`/api/flags?userId=${this.userId}`);
    const data = await response.json();
    for (const [key, value] of Object.entries(data)) {
      this.flags.set(key, value as boolean | string | number);
    }
  }

  isEnabled(key: string, defaultValue = false): boolean {
    return (this.flags.get(key) as boolean) ?? defaultValue;
  }

  getVariant(key: string, defaultValue: string): string {
    return (this.flags.get(key) as string) ?? defaultValue;
  }

  getNumber(key: string, defaultValue: number): number {
    return (this.flags.get(key) as number) ?? defaultValue;
  }
}

// React context integration
const FeatureFlagContext = createContext<FeatureFlagClient | null>(null);

export function FeatureFlagProvider({
  children,
  userId,
}: {
  children: ReactNode;
  userId: string;
}) {
  const client = useMemo(() => new FeatureFlagClient(userId), [userId]);
  const [ready, setReady] = useState(false);

  useEffect(() => {
    client.initialize().then(() => setReady(true));
  }, [client]);

  if (!ready) return <LoadingScreen />;
  return (
    <FeatureFlagContext.Provider value={client}>
      {children}
    </FeatureFlagContext.Provider>
  );
}

export function useFlag(key: string, defaultValue: boolean = false): boolean {
  const client = useContext(FeatureFlagContext);
  return client?.isEnabled(key, defaultValue) ?? defaultValue;
}
```

## LaunchDarkly

LaunchDarkly is the market-leading feature flag platform with SDKs for every platform:

```bash
npm install launchdarkly-react-client-sdk
```

```typescript
// app.tsx — initialize LaunchDarkly
import { withLDProvider } from 'launchdarkly-react-client-sdk';

const App = withLDProvider({
  clientSideID: process.env.REACT_APP_LD_CLIENT_ID!,
  context: {
    kind: 'user',
    key: user.id,              // unique per user
    email: user.email,
    name: user.name,
    custom: {
      plan: user.subscriptionPlan,  // custom attributes for targeting rules
      country: user.country,
      company: user.companyId,
    },
  },
  options: {
    streaming: true,  // real-time flag updates without polling
  },
})(MainApp);

// Using flags in components
import { useFlags, useLDClient } from 'launchdarkly-react-client-sdk';

function CheckoutPage() {
  const { newCheckoutFlow, experimentSearchV2 } = useFlags();
  // Flags are typed — IDE autocomplete works with TS codegen

  return newCheckoutFlow ? <NewCheckout /> : <LegacyCheckout />;
}

// Tracking flag-related metrics
function ProductPage({ productId }: { productId: string }) {
  const { recommendationsAlgorithm } = useFlags();
  const ldClient = useLDClient();

  const handlePurchase = () => {
    // Track experiment outcome
    ldClient?.track('purchase_completed', {
      algorithm: recommendationsAlgorithm,
      productId,
    });
  };
}
```

### LaunchDarkly Targeting Rules

```
LaunchDarkly UI targeting:

Flag: new-checkout-flow
Default variation: false (OFF)

Targeting rules (evaluated in order):
  1. Internal QA users (user.email ends with @company.com) → true
  2. Beta users (user.custom.betaUser === true) → true
  3. Canary rollout (20% of remaining users) → true
  4. Default → false (OFF for everyone else)

Rollout strategy:
  Week 1: 5% → monitor error rates, conversion
  Week 2: 25% → confirm metrics improving
  Week 3: 75% → near-full rollout
  Week 4: 100% → remove flag code
```

## GrowthBook — Open Source Alternative

```bash
npm install @growthbook/growthbook-react
```

```typescript
// GrowthBook — open source, self-hostable
import { GrowthBook, GrowthBookProvider, useFeatureIsOn, useFeatureValue } from '@growthbook/growthbook-react';

const gb = new GrowthBook({
  apiHost: 'https://cdn.growthbook.io',
  clientKey: process.env.REACT_APP_GB_KEY,
  // Track experiment exposures for analysis
  trackingCallback: (experiment, result) => {
    analytics.track('experiment_viewed', {
      experiment_id: experiment.key,
      variation_id: result.variationId,
    });
  },
});

// Initialize with user context
await gb.loadFeatures();
gb.setAttributes({
  id: user.id,
  email: user.email,
  plan: user.plan,
  country: user.country,
});

// React usage
function App() {
  return (
    <GrowthBookProvider growthbook={gb}>
      <Router />
    </GrowthBookProvider>
  );
}

function CheckoutButton() {
  const newFlow = useFeatureIsOn('new-checkout-flow');
  const buttonVariant = useFeatureValue('checkout-button-variant', 'primary');

  return (
    <Button variant={buttonVariant}>
      {newFlow ? 'Proceed to Checkout' : 'Checkout'}
    </Button>
  );
}
```

## Canary Releases

```
Canary release flow:

Deploy v2 to 5% of users:
  ┌─────────────────────────────────────────────┐
  │  Traffic split:                              │
  │  95% ──→ v1 (current stable)                │
  │   5% ──→ v2 (canary)                        │
  └─────────────────────────────────────────────┘

Monitor canary metrics:
  - Error rate (should not increase significantly)
  - Latency p95 (should not degrade)
  - Conversion rate (should not decrease)
  - Specific business metrics (add-to-cart, checkout completion)

Decision:
  Metrics OK → increase to 25% → 75% → 100%
  Metrics bad → kill switch → 0% instantly (no deploy needed)
```

### Sticky Sessions (Consistent Experience)

```typescript
// Users should consistently see the same variant — not flip-flop
// LaunchDarkly and GrowthBook handle this via user key bucketing
// The same userId always maps to the same bucket

// Manual implementation: hash userId into bucket
function assignVariant(userId: string, flagKey: string, variants: string[]): string {
  // Deterministic hash: same userId + flagKey → same variant every time
  const hash = murmurhash3_32_gc(`${flagKey}:${userId}`, 0);
  const bucket = (hash >>> 0) / 4294967296; // normalize to 0-1
  const index = Math.floor(bucket * variants.length);
  return variants[index];
}

// Result: user u1 always gets 'variant-a', user u2 always gets 'variant-b'
// No session storage required — deterministic from user ID
```

## Kill Switches

```typescript
// Operational kill switch pattern
// Kill switches are flags that default to ON, turned OFF to disable

function useKillSwitch(feature: string): boolean {
  const { [feature]: disabled } = useFlags();
  return !(disabled as boolean); // flag OFF means feature disabled
}

// Usage
function RealTimeCollaboration() {
  const realtimeEnabled = useKillSwitch('disable-realtime-collaboration');

  if (!realtimeEnabled) {
    return <StaticViewMode />;
  }
  return <CollaborativeEditor />;
}

// When real-time collaboration causes DB overload:
// Toggle 'disable-realtime-collaboration' = true in LaunchDarkly
// All users immediately fall back to static view — zero deploy
```

## Flag-Driven A/B Testing

```typescript
// Experiment: does longer copy on CTA improve conversion?
function HeroSection() {
  const ctaVariant = useFlags().ctaCopyExperiment; // 'control' | 'verbose'
  const ldClient = useLDClient();

  const handleCTAClick = useCallback(() => {
    // Track conversion event
    ldClient?.track('cta_clicked', { variant: ctaVariant });
    navigate('/signup');
  }, [ctaVariant]);

  return (
    <section>
      <h1>Transform your workflow</h1>
      <Button onClick={handleCTAClick}>
        {ctaVariant === 'verbose'
          ? 'Start your free 14-day trial — no credit card required'
          : 'Get started free'}
      </Button>
    </section>
  );
}

// After experiment: analyze CTR and downstream conversion
// Winner becomes the default; flag and losing variant removed
```

## Flag Hygiene — Avoiding Flag Debt

```typescript
// Flag debt: abandoned flags accumulate as dead code

// ❌ Old flag that should have been removed months ago
function PaymentForm() {
  const { newPaymentProvider } = useFlags();
  if (newPaymentProvider) {
    return <StripePaymentForm />;  // 100% of users get this now
  }
  return <LegacyPaymentForm />;   // Dead code — nobody uses this path
}

// ✅ After 100% rollout: remove the flag and the old code
function PaymentForm() {
  return <StripePaymentForm />;
}

// Best practices:
// 1. Set flag expiry dates in your flag service
// 2. Track flag age in monitoring
// 3. Create tech debt tickets when a flag goes to 100%
// 4. Review flags in every sprint — remove old ones
// 5. Lint rule: flag keys not in allowlist fail CI
```

## Common Mistakes

### 1. Shipping Flag Evaluation Logic to the Browser

```typescript
// ❌ Business targeting logic client-side — users can inspect and override
const flags = {
  premiumFeature: user.plan === 'premium', // users can change plan in localStorage
};

// ✅ Evaluate targeting rules server-side; send result to client
// The server evaluates: "does this user's plan include premium features?"
// and sends back: { premiumFeature: true/false }
// Client only uses the result — never sees the rule
```

### 2. Using Feature Flags for Permanent Configuration

```typescript
// ❌ Flags for things that will never change
const { enableDarkMode } = useFlags(); // dark mode has been default for 2 years

// Flags = temporary; configuration = permanent
// If a flag hasn't been touched in 6 months, it's configuration — bake it in
```

### 3. Not Handling Flag Fetch Failures

```typescript
// ❌ App breaks if flag service is unavailable
const { newFeature } = useFlags();
// If flag service is down, useFlags() returns empty object → newFeature is undefined
// → runtime error or incorrect behavior

// ✅ Always provide sensible defaults
const newFeature = useFlags().newFeature ?? false; // fallback to safe default
```

### 4. Creating Flags Without Flag-Off Path

```typescript
// ❌ Flag with no old behavior to fall back to
function NewFeaturePage() {
  const { newFeature } = useFlags();
  if (!newFeature) return null; // flag-off = blank page!
}

// ✅ Always have a flag-off state
function NewFeaturePage() {
  const { newFeature } = useFlags();
  if (!newFeature) return <ComingSoonPage />; // graceful degradation
  return <NewFeatureContent />;
}
```

## Interview Questions

### 1. What is the difference between a feature flag and an environment variable?

**Answer:** An environment variable is baked into the build at deploy time — changing it requires a new build and deployment. A feature flag is evaluated at runtime, typically fetched from a flag service on each session or request. Feature flags can be toggled without deploying — a kill switch takes effect for all users within seconds of the toggle, not after a 5-minute deploy pipeline. Environment variables work for per-environment configuration (dev, staging, prod) but not for per-user targeting, gradual rollouts, or instant kill switches.

### 2. How do you ensure users get a consistent flag variant (don't flip-flop between variants)?

**Answer:** Flag SDKs (LaunchDarkly, GrowthBook) use deterministic bucketing — they hash the user's unique identifier (userId) with the flag key to produce a consistent bucket value. The same userId always maps to the same bucket, so the same user always gets the same variant regardless of which server or edge node evaluates the flag. This is called "sticky evaluation." The hash is deterministic and requires no storage — the same inputs always produce the same outputs. Without sticky bucketing, a user might see the control on one page load and the variant on the next, making experiment results invalid.

### 3. What is a kill switch and how does it differ from a feature flag?

**Answer:** A kill switch is a feature flag that defaults to enabled and is used to quickly disable a feature in production. The naming inversion is intentional: the "off" state means "disabled," which operators toggle when something breaks. Unlike release flags (which start disabled and are enabled gradually), kill switches start enabled — the feature is always on unless explicitly killed. They're operational flags for resilience: "disable real-time sync under high load," "disable the recommendations algorithm if it causes high latency," "disable uploads if object storage is degraded." The key value is instant effect without a deployment — the flag toggle propagates to all users in seconds.

### 4. What is flag debt and how do you prevent it?

**Answer:** Flag debt is the accumulation of old feature flags that are no longer serving their original purpose — a feature that's been fully rolled out but still has `if (flagEnabled)` branching in the code. It increases cognitive overhead (which branch is the real one?), risks bugs (the old branch gets out of date), and bloats the codebase. Prevention: (1) Set expiry dates on flags in the flag service; (2) Create a tech debt ticket immediately when a flag reaches 100% rollout, scheduled for the next sprint; (3) Lint rules that flag keys must be in an allowlist — expired keys fail CI; (4) Regular flag audit in sprint retrospectives; (5) Flag naming conventions that include the feature area and approximate year.
