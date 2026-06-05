# Feature Flags

## The Idea

**In plain English:** A feature flag is like a light switch built into your software — it lets you turn a new feature on or off without having to re-release the whole app. "Deploying" means pushing new code to the live app; a feature flag means that code is already there, just hidden until you flip the switch.

**Real-world analogy:** Imagine a restaurant that secretly installs a brand-new menu item in the kitchen but keeps it off the printed menu. They quietly offer it to a handful of regulars first to get feedback, then slowly start mentioning it to more customers, and only put it on the official menu once they are confident it works well.

- The kitchen already having the dish = the code already deployed to production
- The "off the menu" status = the feature flag turned off for most users
- Offering it to a handful of regulars first = a gradual rollout to a small percentage of users
- The owner flipping a switch to add it to the menu = toggling the flag on for everyone

---

## Overview

Feature flags (also called feature toggles or feature switches) are a software development technique that allows teams to enable or disable features without deploying new code. They provide control over feature rollouts, enable A/B testing, support gradual releases, and offer kill switches for problematic features. Modern feature flag systems support targeting, scheduling, and experimentation, making them essential for continuous delivery and risk management.

## Table of Contents

- [Feature Flag Fundamentals](#feature-flag-fundamentals)
- [Types of Feature Flags](#types-of-feature-flags)
- [LaunchDarkly Integration](#launchdarkly-integration)
- [Gradual Rollouts](#gradual-rollouts)
- [A/B Testing](#ab-testing)
- [Kill Switches](#kill-switches)
- [Targeting and Segmentation](#targeting-and-segmentation)
- [Feature Flag Management](#feature-flag-management)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use/Not to Use](#when-to-usenot-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Feature Flag Fundamentals

### Why Feature Flags Matter

```
Traditional Deployment          vs        Feature Flag Deployment
──────────────────────                   ───────────────────────

Deploy → All users see change            Deploy → Flag off by default
                                                 ↓
❌ Risk: Bug affects everyone            Enable for 10% → Test
❌ Rollback requires redeploy            Enable for 50% → Monitor
❌ Can't target specific users           Enable for 100% → Full rollout
❌ No gradual rollout                    
                                         ✅ Risk: Bug affects 10%
                                         ✅ Rollback: Toggle flag off
                                         ✅ Target: Beta users, employees
                                         ✅ Gradual: 10% → 50% → 100%
```

### Basic Feature Flag Implementation

```typescript
// feature-flags.ts
interface FeatureFlags {
  newDashboard: boolean;
  darkMode: boolean;
  advancedAnalytics: boolean;
  experimentalSearch: boolean;
}

class FeatureFlagManager {
  private flags: Map<string, boolean> = new Map();
  
  constructor(initialFlags: Partial<FeatureFlags> = {}) {
    Object.entries(initialFlags).forEach(([key, value]) => {
      this.flags.set(key, value);
    });
  }
  
  isEnabled(flagName: string): boolean {
    return this.flags.get(flagName) || false;
  }
  
  enable(flagName: string): void {
    this.flags.set(flagName, true);
  }
  
  disable(flagName: string): void {
    this.flags.set(flagName, false);
  }
  
  getAllFlags(): Record<string, boolean> {
    return Object.fromEntries(this.flags);
  }
}

export const featureFlags = new FeatureFlagManager({
  newDashboard: false,
  darkMode: true,
  advancedAnalytics: false,
  experimentalSearch: false,
});

// Usage in React
import { featureFlags } from './feature-flags';

const Dashboard: React.FC = () => {
  const showNewDashboard = featureFlags.isEnabled('newDashboard');
  
  if (showNewDashboard) {
    return <NewDashboard />;
  }
  
  return <OldDashboard />;
};

// Usage in Angular
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root'
})
export class FeatureFlagService {
  isEnabled(flagName: string): boolean {
    return featureFlags.isEnabled(flagName);
  }
}
```

## Types of Feature Flags

### Release Toggles

```typescript
// Release toggles: Hide incomplete features
// Lifespan: Short (days to weeks)
// Remove after feature is fully released

const releaseFlags = {
  // New feature in development
  redesignedCheckout: false,
  
  // Feature ready for beta testing
  newPaymentFlow: true,
  
  // Feature ready for full release
  improvedSearch: true,
};

// Component
const CheckoutPage: React.FC = () => {
  if (featureFlags.isEnabled('redesignedCheckout')) {
    return <NewCheckoutFlow />;
  }
  
  return <LegacyCheckoutFlow />;
};
```

### Experiment Toggles

```typescript
// Experiment toggles: A/B testing
// Lifespan: Medium (weeks to months)
// Remove after experiment concludes

interface ExperimentConfig {
  name: string;
  variants: string[];
  weights: number[];
}

class ExperimentManager {
  private experiments: Map<string, ExperimentConfig> = new Map();
  private userAssignments: Map<string, Map<string, string>> = new Map();
  
  defineExperiment(config: ExperimentConfig): void {
    this.experiments.set(config.name, config);
  }
  
  getVariant(userId: string, experimentName: string): string {
    // Check if user already assigned
    if (!this.userAssignments.has(userId)) {
      this.userAssignments.set(userId, new Map());
    }
    
    const userAssignments = this.userAssignments.get(userId)!;
    if (userAssignments.has(experimentName)) {
      return userAssignments.get(experimentName)!;
    }
    
    // Assign variant
    const experiment = this.experiments.get(experimentName);
    if (!experiment) {
      return 'control';
    }
    
    const variant = this.assignVariant(experiment);
    userAssignments.set(experimentName, variant);
    
    return variant;
  }
  
  private assignVariant(experiment: ExperimentConfig): string {
    const random = Math.random();
    let cumulative = 0;
    
    for (let i = 0; i < experiment.variants.length; i++) {
      cumulative += experiment.weights[i];
      if (random < cumulative) {
        return experiment.variants[i];
      }
    }
    
    return experiment.variants[0];
  }
}

export const experimentManager = new ExperimentManager();

// Define experiment
experimentManager.defineExperiment({
  name: 'button_color',
  variants: ['blue', 'green', 'red'],
  weights: [0.33, 0.33, 0.34],
});

// Usage
const SignupButton: React.FC = () => {
  const userId = useCurrentUserId();
  const variant = experimentManager.getVariant(userId, 'button_color');
  
  return (
    <button style={{ backgroundColor: variant }}>
      Sign Up
    </button>
  );
};
```

### Ops Toggles

```typescript
// Ops toggles: Operational control (kill switches, circuit breakers)
// Lifespan: Long (permanent)

interface OpsToggle {
  name: string;
  enabled: boolean;
  reason?: string;
  disabledAt?: number;
}

class OpsToggleManager {
  private toggles: Map<string, OpsToggle> = new Map();
  
  defineToggle(name: string): void {
    this.toggles.set(name, {
      name,
      enabled: true,
    });
  }
  
  isEnabled(name: string): boolean {
    const toggle = this.toggles.get(name);
    return toggle ? toggle.enabled : true;
  }
  
  disable(name: string, reason: string): void {
    const toggle = this.toggles.get(name);
    if (toggle) {
      toggle.enabled = false;
      toggle.reason = reason;
      toggle.disabledAt = Date.now();
      
      console.warn(`[OPS TOGGLE] Disabled: ${name} - ${reason}`);
      
      // Alert team
      this.alertTeam(name, reason);
    }
  }
  
  enable(name: string): void {
    const toggle = this.toggles.get(name);
    if (toggle) {
      toggle.enabled = true;
      toggle.reason = undefined;
      toggle.disabledAt = undefined;
    }
  }
  
  private alertTeam(name: string, reason: string): void {
    // Send to Slack, PagerDuty, etc.
  }
}

export const opsToggles = new OpsToggleManager();

// Define ops toggles
opsToggles.defineToggle('external_api_enabled');
opsToggles.defineToggle('email_notifications_enabled');
opsToggles.defineToggle('background_jobs_enabled');

// Usage
async function processPayment() {
  if (!opsToggles.isEnabled('external_api_enabled')) {
    throw new Error('Payment processing temporarily disabled');
  }
  
  // Process payment
}

// Kill switch
if (apiErrorRate > 10) {
  opsToggles.disable('external_api_enabled', 'High error rate detected');
}
```

### Permission Toggles

```typescript
// Permission toggles: User-based access control
// Lifespan: Long (permanent)

interface User {
  id: string;
  role: 'admin' | 'user' | 'guest';
  plan: 'free' | 'pro' | 'enterprise';
  betaFeatures: string[];
}

class PermissionToggleManager {
  canAccess(user: User, feature: string): boolean {
    // Admin always has access
    if (user.role === 'admin') {
      return true;
    }
    
    // Check plan-based access
    if (this.requiresPlan(feature, user.plan)) {
      return true;
    }
    
    // Check beta access
    if (user.betaFeatures.includes(feature)) {
      return true;
    }
    
    return false;
  }
  
  private requiresPlan(feature: string, plan: string): boolean {
    const featurePlans: Record<string, string[]> = {
      advancedAnalytics: ['pro', 'enterprise'],
      apiAccess: ['pro', 'enterprise'],
      whiteLabel: ['enterprise'],
      basicReports: ['free', 'pro', 'enterprise'],
    };
    
    return featurePlans[feature]?.includes(plan) || false;
  }
}

export const permissionToggles = new PermissionToggleManager();

// Usage
const AnalyticsPage: React.FC = () => {
  const user = useCurrentUser();
  const hasAccess = permissionToggles.canAccess(user, 'advancedAnalytics');
  
  if (!hasAccess) {
    return <UpgradePrompt feature="Advanced Analytics" />;
  }
  
  return <AdvancedAnalytics />;
};
```

## LaunchDarkly Integration

### LaunchDarkly Setup (React)

```typescript
// launchdarkly-setup.ts
import { asyncWithLDProvider } from 'launchdarkly-react-client-sdk';

export async function initializeLaunchDarkly(clientSideId: string, user?: any) {
  const LDProvider = await asyncWithLDProvider({
    clientSideID: clientSideId,
    context: user ? {
      kind: 'user',
      key: user.id,
      email: user.email,
      name: user.name,
      custom: {
        plan: user.plan,
        role: user.role,
      },
    } : {
      kind: 'user',
      key: 'anonymous',
      anonymous: true,
    },
    options: {
      bootstrap: 'localStorage', // Cache flags
    },
  });
  
  return LDProvider;
}

// App.tsx
import React, { useEffect, useState } from 'react';
import { initializeLaunchDarkly } from './launchdarkly-setup';

function App() {
  const [LDProvider, setLDProvider] = useState<any>(null);
  
  useEffect(() => {
    const user = getCurrentUser();
    
    initializeLaunchDarkly(
      process.env.REACT_APP_LD_CLIENT_ID || '',
      user
    ).then(setLDProvider);
  }, []);
  
  if (!LDProvider) {
    return <Loading />;
  }
  
  return (
    <LDProvider>
      <Routes />
    </LDProvider>
  );
}

// Component usage
import { useFlags } from 'launchdarkly-react-client-sdk';

const Dashboard: React.FC = () => {
  const flags = useFlags();
  
  return (
    <div>
      {flags.newDashboard ? <NewDashboard /> : <OldDashboard />}
      {flags.darkMode && <DarkModeToggle />}
      {flags.advancedAnalytics && <AnalyticsPanel />}
    </div>
  );
};

// Hook for specific flag
import { useLDClient } from 'launchdarkly-react-client-sdk';

function useFeatureFlag(flagKey: string, defaultValue: boolean = false): boolean {
  const ldClient = useLDClient();
  const [flagValue, setFlagValue] = useState(defaultValue);
  
  useEffect(() => {
    if (!ldClient) return;
    
    // Get initial value
    setFlagValue(ldClient.variation(flagKey, defaultValue));
    
    // Subscribe to changes
    const handleChange = (current: boolean) => {
      setFlagValue(current);
    };
    
    ldClient.on(`change:${flagKey}`, handleChange);
    
    return () => {
      ldClient.off(`change:${flagKey}`, handleChange);
    };
  }, [ldClient, flagKey, defaultValue]);
  
  return flagValue;
}

// Usage
const NewFeature: React.FC = () => {
  const enabled = useFeatureFlag('newFeature', false);
  
  if (!enabled) {
    return null;
  }
  
  return <div>New Feature Content</div>;
};
```

### LaunchDarkly Server-Side (Node.js)

```javascript
// launchdarkly-server.js
const LaunchDarkly = require('@launchdarkly/node-server-sdk');

const ldClient = LaunchDarkly.init(process.env.LD_SDK_KEY);

async function waitForLDClient() {
  await ldClient.waitForInitialization();
  console.log('LaunchDarkly initialized');
}

waitForLDClient();

// Evaluate flag for user
async function getFlagValue(flagKey, user, defaultValue = false) {
  try {
    const value = await ldClient.variation(flagKey, user, defaultValue);
    return value;
  } catch (error) {
    console.error(`Error evaluating flag ${flagKey}:`, error);
    return defaultValue;
  }
}

// Express middleware
function featureFlagMiddleware(req, res, next) {
  req.featureFlags = {
    isEnabled: async (flagKey, defaultValue = false) => {
      const user = {
        key: req.user?.id || 'anonymous',
        email: req.user?.email,
        custom: {
          plan: req.user?.plan,
          role: req.user?.role,
        },
      };
      
      return await getFlagValue(flagKey, user, defaultValue);
    },
  };
  
  next();
}

app.use(featureFlagMiddleware);

// Usage in route
app.get('/api/analytics', async (req, res) => {
  const hasAccess = await req.featureFlags.isEnabled('advancedAnalytics');
  
  if (!hasAccess) {
    return res.status(403).json({ error: 'Feature not available' });
  }
  
  // Return analytics data
});

// Graceful shutdown
process.on('SIGTERM', async () => {
  await ldClient.flush();
  ldClient.close();
});

module.exports = { ldClient, getFlagValue };
```

## Gradual Rollouts

### Percentage Rollout

```typescript
// percentage-rollout.ts
class PercentageRollout {
  private rolloutPercentages: Map<string, number> = new Map();
  
  setRolloutPercentage(feature: string, percentage: number): void {
    this.rolloutPercentages.set(feature, percentage);
  }
  
  isEnabledForUser(feature: string, userId: string): boolean {
    const percentage = this.rolloutPercentages.get(feature) || 0;
    
    if (percentage === 0) return false;
    if (percentage === 100) return true;
    
    // Consistent hashing for stable assignment
    const hash = this.hashUserId(userId);
    return hash < percentage;
  }
  
  private hashUserId(userId: string): number {
    // Simple hash function (use better hash in production)
    let hash = 0;
    for (let i = 0; i < userId.length; i++) {
      hash = ((hash << 5) - hash) + userId.charCodeAt(i);
      hash = hash & hash; // Convert to 32bit integer
    }
    
    // Return value between 0-100
    return Math.abs(hash) % 100;
  }
}

export const percentageRollout = new PercentageRollout();

// Gradual rollout schedule
async function performGradualRollout(feature: string) {
  // Day 1: 10%
  percentageRollout.setRolloutPercentage(feature, 10);
  await sleep(24 * 60 * 60 * 1000);
  
  // Day 2: 25%
  percentageRollout.setRolloutPercentage(feature, 25);
  await sleep(24 * 60 * 60 * 1000);
  
  // Day 3: 50%
  percentageRollout.setRolloutPercentage(feature, 50);
  await sleep(24 * 60 * 60 * 1000);
  
  // Day 4: 100%
  percentageRollout.setRolloutPercentage(feature, 100);
}

// Usage
const NewFeature: React.FC = () => {
  const userId = useCurrentUserId();
  const enabled = percentageRollout.isEnabledForUser('newFeature', userId);
  
  if (!enabled) {
    return <OldFeature />;
  }
  
  return <NewFeature />;
};
```

### Ring-Based Rollout

```typescript
// ring-rollout.ts
enum RolloutRing {
  Internal = 'internal',      // Employees
  Canary = 'canary',          // 1% of users
  EarlyAdopters = 'early',    // 10% of users
  General = 'general',        // 100% of users
}

interface RolloutConfig {
  feature: string;
  currentRing: RolloutRing;
  schedule: {
    ring: RolloutRing;
    date: Date;
  }[];
}

class RingRollout {
  private configs: Map<string, RolloutConfig> = new Map();
  private internalUsers: Set<string> = new Set();
  private canaryUsers: Set<string> = new Set();
  
  configureRollout(config: RolloutConfig): void {
    this.configs.set(config.feature, config);
  }
  
  isEnabledForUser(feature: string, user: {
    id: string;
    isInternal: boolean;
    isCanary: boolean;
  }): boolean {
    const config = this.configs.get(feature);
    if (!config) return false;
    
    switch (config.currentRing) {
      case RolloutRing.Internal:
        return user.isInternal;
      
      case RolloutRing.Canary:
        return user.isInternal || user.isCanary;
      
      case RolloutRing.EarlyAdopters:
        return user.isInternal || user.isCanary || this.isEarlyAdopter(user.id);
      
      case RolloutRing.General:
        return true;
      
      default:
        return false;
    }
  }
  
  private isEarlyAdopter(userId: string): boolean {
    // Use consistent hashing for stable 10% assignment
    const hash = this.hashUserId(userId);
    return hash < 10;
  }
  
  private hashUserId(userId: string): number {
    let hash = 0;
    for (let i = 0; i < userId.length; i++) {
      hash = ((hash << 5) - hash) + userId.charCodeAt(i);
      hash = hash & hash;
    }
    return Math.abs(hash) % 100;
  }
}

export const ringRollout = new RingRollout();

// Configure rollout
ringRollout.configureRollout({
  feature: 'newDashboard',
  currentRing: RolloutRing.Internal,
  schedule: [
    { ring: RolloutRing.Internal, date: new Date('2024-06-01') },
    { ring: RolloutRing.Canary, date: new Date('2024-06-03') },
    { ring: RolloutRing.EarlyAdopters, date: new Date('2024-06-07') },
    { ring: RolloutRing.General, date: new Date('2024-06-14') },
  ],
});
```

## A/B Testing

### A/B Test Implementation

```typescript
// ab-test.ts
interface ABTestConfig {
  name: string;
  variants: {
    name: string;
    weight: number;
  }[];
  targetingRules?: {
    attribute: string;
    operator: 'equals' | 'contains' | 'greaterThan';
    value: any;
  }[];
}

class ABTestManager {
  private tests: Map<string, ABTestConfig> = new Map();
  private assignments: Map<string, Map<string, string>> = new Map();
  
  defineTest(config: ABTestConfig): void {
    // Validate weights sum to 1
    const totalWeight = config.variants.reduce((sum, v) => sum + v.weight, 0);
    if (Math.abs(totalWeight - 1) > 0.001) {
      throw new Error('Variant weights must sum to 1');
    }
    
    this.tests.set(config.name, config);
  }
  
  getVariant(testName: string, userId: string, userAttributes?: Record<string, any>): string {
    const test = this.tests.get(testName);
    if (!test) {
      return 'control';
    }
    
    // Check targeting rules
    if (test.targetingRules && userAttributes) {
      const matches = test.targetingRules.every(rule => 
        this.evaluateRule(rule, userAttributes)
      );
      
      if (!matches) {
        return 'control';
      }
    }
    
    // Get or assign variant
    if (!this.assignments.has(userId)) {
      this.assignments.set(userId, new Map());
    }
    
    const userAssignments = this.assignments.get(userId)!;
    
    if (userAssignments.has(testName)) {
      return userAssignments.get(testName)!;
    }
    
    // Assign variant based on weights
    const variant = this.assignVariant(test, userId);
    userAssignments.set(testName, variant);
    
    // Track assignment
    analytics.trackEvent('AB Test Assigned', {
      test: testName,
      variant,
      userId,
    });
    
    return variant;
  }
  
  private assignVariant(test: ABTestConfig, userId: string): string {
    const hash = this.hashString(`${userId}:${test.name}`);
    const random = hash / 0xFFFFFFFF; // Normalize to 0-1
    
    let cumulative = 0;
    for (const variant of test.variants) {
      cumulative += variant.weight;
      if (random < cumulative) {
        return variant.name;
      }
    }
    
    return test.variants[0].name;
  }
  
  private hashString(str: string): number {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i);
      hash = hash & hash;
    }
    return Math.abs(hash);
  }
  
  private evaluateRule(rule: any, attributes: Record<string, any>): boolean {
    const value = attributes[rule.attribute];
    
    switch (rule.operator) {
      case 'equals':
        return value === rule.value;
      case 'contains':
        return String(value).includes(rule.value);
      case 'greaterThan':
        return Number(value) > Number(rule.value);
      default:
        return false;
    }
  }
  
  trackConversion(testName: string, userId: string, value?: number): void {
    const variant = this.assignments.get(userId)?.get(testName);
    
    if (!variant) {
      console.warn(`No variant found for user ${userId} in test ${testName}`);
      return;
    }
    
    analytics.trackEvent('AB Test Conversion', {
      test: testName,
      variant,
      userId,
      value,
    });
  }
}

export const abTestManager = new ABTestManager();

// Define A/B test
abTestManager.defineTest({
  name: 'checkout_button_color',
  variants: [
    { name: 'control', weight: 0.5 },
    { name: 'blue', weight: 0.25 },
    { name: 'green', weight: 0.25 },
  ],
  targetingRules: [
    { attribute: 'plan', operator: 'equals', value: 'free' },
  ],
});

// Usage in component
const CheckoutButton: React.FC = () => {
  const userId = useCurrentUserId();
  const user = useCurrentUser();
  
  const variant = abTestManager.getVariant(
    'checkout_button_color',
    userId,
    { plan: user.plan }
  );
  
  const handleCheckout = () => {
    // Track conversion
    abTestManager.trackConversion(
      'checkout_button_color',
      userId,
      cart.total
    );
    
    // Proceed with checkout
    processCheckout();
  };
  
  return (
    <button
      style={{ backgroundColor: variant === 'blue' ? 'blue' : variant === 'green' ? 'green' : 'gray' }}
      onClick={handleCheckout}
    >
      Checkout
    </button>
  );
};
```

## Kill Switches

### Emergency Kill Switch

```typescript
// kill-switch.ts
interface KillSwitch {
  name: string;
  enabled: boolean;
  disabledAt?: number;
  disabledBy?: string;
  reason?: string;
}

class KillSwitchManager {
  private switches: Map<string, KillSwitch> = new Map();
  private listeners: Map<string, Set<(enabled: boolean) => void>> = new Map();
  
  register(name: string): void {
    this.switches.set(name, {
      name,
      enabled: true,
    });
  }
  
  isEnabled(name: string): boolean {
    const killSwitch = this.switches.get(name);
    return killSwitch ? killSwitch.enabled : false;
  }
  
  disable(name: string, reason: string, disabledBy: string): void {
    const killSwitch = this.switches.get(name);
    if (!killSwitch) {
      throw new Error(`Kill switch ${name} not found`);
    }
    
    killSwitch.enabled = false;
    killSwitch.disabledAt = Date.now();
    killSwitch.disabledBy = disabledBy;
    killSwitch.reason = reason;
    
    console.error(`[KILL SWITCH] Disabled: ${name} - ${reason}`);
    
    // Notify listeners
    this.notifyListeners(name, false);
    
    // Alert team
    this.alertTeam(name, reason, disabledBy);
    
    // Log to monitoring
    this.logToMonitoring(name, 'disabled', reason);
  }
  
  enable(name: string): void {
    const killSwitch = this.switches.get(name);
    if (!killSwitch) {
      throw new Error(`Kill switch ${name} not found`);
    }
    
    killSwitch.enabled = true;
    killSwitch.disabledAt = undefined;
    killSwitch.disabledBy = undefined;
    killSwitch.reason = undefined;
    
    console.log(`[KILL SWITCH] Enabled: ${name}`);
    
    // Notify listeners
    this.notifyListeners(name, true);
    
    // Log to monitoring
    this.logToMonitoring(name, 'enabled');
  }
  
  subscribe(name: string, callback: (enabled: boolean) => void): () => void {
    if (!this.listeners.has(name)) {
      this.listeners.set(name, new Set());
    }
    
    this.listeners.get(name)!.add(callback);
    
    // Return unsubscribe function
    return () => {
      this.listeners.get(name)?.delete(callback);
    };
  }
  
  private notifyListeners(name: string, enabled: boolean): void {
    const listeners = this.listeners.get(name);
    if (listeners) {
      listeners.forEach(callback => callback(enabled));
    }
  }
  
  private alertTeam(name: string, reason: string, disabledBy: string): void {
    // Send to Slack, PagerDuty, etc.
    const message = `🚨 KILL SWITCH ACTIVATED\n\nFeature: ${name}\nReason: ${reason}\nDisabled by: ${disabledBy}\nTime: ${new Date().toISOString()}`;
    
    // Implementation depends on alerting system
  }
  
  private logToMonitoring(name: string, action: string, reason?: string): void {
    // Send to APM/logging system
    analytics.trackEvent('Kill Switch Action', {
      switch: name,
      action,
      reason,
    });
  }
  
  getStatus(): Record<string, KillSwitch> {
    return Object.fromEntries(this.switches);
  }
}

export const killSwitchManager = new KillSwitchManager();

// Register kill switches
killSwitchManager.register('external_payment_api');
killSwitchManager.register('email_notifications');
killSwitchManager.register('background_processing');
killSwitchManager.register('real_time_updates');

// Usage with auto-kill based on error rate
let errorCount = 0;
let requestCount = 0;

async function processPayment() {
  requestCount++;
  
  if (!killSwitchManager.isEnabled('external_payment_api')) {
    throw new Error('Payment processing is temporarily disabled');
  }
  
  try {
    await externalPaymentAPI.process();
  } catch (error) {
    errorCount++;
    
    const errorRate = errorCount / requestCount;
    
    // Auto-kill if error rate > 10%
    if (errorRate > 0.1 && requestCount > 10) {
      killSwitchManager.disable(
        'external_payment_api',
        `High error rate detected: ${(errorRate * 100).toFixed(2)}%`,
        'auto-kill'
      );
    }
    
    throw error;
  }
}

// React hook for kill switch
function useKillSwitch(name: string): boolean {
  const [enabled, setEnabled] = useState(
    killSwitchManager.isEnabled(name)
  );
  
  useEffect(() => {
    return killSwitchManager.subscribe(name, setEnabled);
  }, [name]);
  
  return enabled;
}

// Usage in component
const PaymentForm: React.FC = () => {
  const paymentEnabled = useKillSwitch('external_payment_api');
  
  if (!paymentEnabled) {
    return (
      <Alert severity="warning">
        Payment processing is temporarily unavailable. Please try again later.
      </Alert>
    );
  }
  
  return <PaymentFormFields />;
};
```

## Targeting and Segmentation

### User Targeting

```typescript
// targeting.ts
interface TargetingRule {
  attribute: string;
  operator: 'equals' | 'notEquals' | 'contains' | 'greaterThan' | 'lessThan' | 'in' | 'notIn';
  value: any;
}

interface TargetingConfig {
  feature: string;
  rules: TargetingRule[];
  matchAll: boolean; // AND vs OR
}

class TargetingManager {
  private configs: Map<string, TargetingConfig> = new Map();
  
  configureTargeting(config: TargetingConfig): void {
    this.configs.set(config.feature, config);
  }
  
  isTargeted(feature: string, user: Record<string, any>): boolean {
    const config = this.configs.get(feature);
    if (!config) return true; // No targeting = available to all
    
    if (config.matchAll) {
      // AND: all rules must match
      return config.rules.every(rule => this.evaluateRule(rule, user));
    } else {
      // OR: at least one rule must match
      return config.rules.some(rule => this.evaluateRule(rule, user));
    }
  }
  
  private evaluateRule(rule: TargetingRule, user: Record<string, any>): boolean {
    const userValue = user[rule.attribute];
    
    switch (rule.operator) {
      case 'equals':
        return userValue === rule.value;
      
      case 'notEquals':
        return userValue !== rule.value;
      
      case 'contains':
        return String(userValue).includes(rule.value);
      
      case 'greaterThan':
        return Number(userValue) > Number(rule.value);
      
      case 'lessThan':
        return Number(userValue) < Number(rule.value);
      
      case 'in':
        return Array.isArray(rule.value) && rule.value.includes(userValue);
      
      case 'notIn':
        return Array.isArray(rule.value) && !rule.value.includes(userValue);
      
      default:
        return false;
    }
  }
}

export const targetingManager = new TargetingManager();

// Target premium users
targetingManager.configureTargeting({
  feature: 'advancedAnalytics',
  matchAll: false, // OR
  rules: [
    { attribute: 'plan', operator: 'in', value: ['pro', 'enterprise'] },
    { attribute: 'role', operator: 'equals', value: 'admin' },
  ],
});

// Target beta testers in specific regions
targetingManager.configureTargeting({
  feature: 'newUI',
  matchAll: true, // AND
  rules: [
    { attribute: 'betaTester', operator: 'equals', value: true },
    { attribute: 'region', operator: 'in', value: ['US', 'CA', 'UK'] },
  ],
});

// Usage
const AdvancedAnalytics: React.FC = () => {
  const user = useCurrentUser();
  const hasAccess = targetingManager.isTargeted('advancedAnalytics', user);
  
  if (!hasAccess) {
    return <UpgradePrompt />;
  }
  
  return <AnalyticsContent />;
};
```

## Feature Flag Management

### Feature Flag Dashboard

```typescript
// flag-dashboard.ts
interface FlagMetrics {
  name: string;
  enabled: boolean;
  rolloutPercentage?: number;
  usersAffected: number;
  lastModified: Date;
  modifiedBy: string;
  type: 'release' | 'experiment' | 'ops' | 'permission';
  lifecycle: 'temporary' | 'permanent';
}

class FlagDashboard {
  private flags: Map<string, FlagMetrics> = new Map();
  
  registerFlag(metrics: FlagMetrics): void {
    this.flags.set(metrics.name, metrics);
  }
  
  getAllFlags(): FlagMetrics[] {
    return Array.from(this.flags.values());
  }
  
  getStaleFlags(daysSinceModified: number = 90): FlagMetrics[] {
    const threshold = Date.now() - (daysSinceModified * 24 * 60 * 60 * 1000);
    
    return this.getAllFlags().filter(flag => 
      flag.lifecycle === 'temporary' &&
      flag.lastModified.getTime() < threshold
    );
  }
  
  getFlagsByType(type: FlagMetrics['type']): FlagMetrics[] {
    return this.getAllFlags().filter(flag => flag.type === type);
  }
  
  generateReport(): {
    total: number;
    byType: Record<string, number>;
    stale: number;
    enabled: number;
  } {
    const flags = this.getAllFlags();
    
    return {
      total: flags.length,
      byType: {
        release: flags.filter(f => f.type === 'release').length,
        experiment: flags.filter(f => f.type === 'experiment').length,
        ops: flags.filter(f => f.type === 'ops').length,
        permission: flags.filter(f => f.type === 'permission').length,
      },
      stale: this.getStaleFlags().length,
      enabled: flags.filter(f => f.enabled).length,
    };
  }
}

export const flagDashboard = new FlagDashboard();

// Register flags
flagDashboard.registerFlag({
  name: 'newDashboard',
  enabled: true,
  rolloutPercentage: 50,
  usersAffected: 5000,
  lastModified: new Date('2024-06-01'),
  modifiedBy: 'john@example.com',
  type: 'release',
  lifecycle: 'temporary',
});
```

### Automated Flag Cleanup

```typescript
// flag-cleanup.ts
class FlagCleanup {
  async suggestFlagsForRemoval(): Promise<string[]> {
    const suggestions: string[] = [];
    
    // Find flags that are 100% rolled out for >30 days
    const fullyRolledOut = flagDashboard.getAllFlags().filter(flag => {
      if (flag.type !== 'release') return false;
      
      const daysSinceModified = (Date.now() - flag.lastModified.getTime()) / (1000 * 60 * 60 * 24);
      return flag.rolloutPercentage === 100 && daysSinceModified > 30;
    });
    
    suggestions.push(...fullyRolledOut.map(f => f.name));
    
    // Find experiment flags for concluded experiments
    const concludedExperiments = flagDashboard.getFlagsByType('experiment').filter(flag => {
      const daysSinceModified = (Date.now() - flag.lastModified.getTime()) / (1000 * 60 * 60 * 24);
      return daysSinceModified > 60; // Experiments typically run 30-60 days
    });
    
    suggestions.push(...concludedExperiments.map(f => f.name));
    
    return suggestions;
  }
  
  async generateRemovalPlan(flagName: string): Promise<{
    flag: string;
    steps: string[];
    affectedFiles: string[];
    testingRequired: string[];
  }> {
    // Analyze code to find flag usage
    const affectedFiles = await this.findFlagUsage(flagName);
    
    return {
      flag: flagName,
      steps: [
        'Verify flag is fully rolled out to 100%',
        'Ensure no recent changes to flag configuration',
        'Create feature branch for flag removal',
        'Replace flag checks with default behavior',
        'Remove flag from feature flag system',
        'Run full test suite',
        'Deploy and monitor',
      ],
      affectedFiles,
      testingRequired: [
        'Unit tests for affected components',
        'Integration tests for feature',
        'E2E tests covering user flows',
        'Performance tests if applicable',
      ],
    };
  }
  
  private async findFlagUsage(flagName: string): Promise<string[]> {
    // Search codebase for flag usage
    // This would use grep or AST analysis in real implementation
    return [];
  }
}

export const flagCleanup = new FlagCleanup();
```

## Common Mistakes

### 1. Not Cleaning Up Old Flags

```typescript
// Bad: Accumulating technical debt
if (featureFlags.isEnabled('oldFeature2022')) {
  // Feature that's been 100% for 2 years
}

// Good: Remove flags after full rollout
// Just use the new code directly
// Old feature flag check removed after 30 days at 100%
```

### 2. Complex Flag Logic

```typescript
// Bad: Nested flag checks
if (featureFlags.isEnabled('featureA')) {
  if (featureFlags.isEnabled('featureB')) {
    if (user.plan === 'premium') {
      return <ComplexComponent />;
    }
  }
}

// Good: Single combined flag or targeting
if (targetingManager.isTargeted('combinedFeature', user)) {
  return <ComplexComponent />;
}
```

### 3. Missing Default Values

```typescript
// Bad: No fallback
const enabled = featureFlags.isEnabled('newFeature'); // undefined if flag doesn't exist

// Good: Explicit default
const enabled = featureFlags.isEnabled('newFeature', false); // Default to false
```

### 4. Not Tracking Flag Usage

```typescript
// Bad: No analytics
if (featureFlags.isEnabled('newFeature')) {
  return <NewFeature />;
}

// Good: Track exposure
if (featureFlags.isEnabled('newFeature')) {
  analytics.trackEvent('Feature Flag Exposed', {
    flag: 'newFeature',
    variant: 'enabled',
  });
  return <NewFeature />;
}
```

### 5. Synchronous Flag Evaluation

```typescript
// Bad: Blocking UI while fetching flags
const flags = await fetchFeatureFlags(); // Blocks render

// Good: Show default while loading
const { flags, loading } = useFeatureFlags();

if (loading) {
  return <DefaultView />;
}

return flags.newFeature ? <NewFeature /> : <OldFeature />;
```

## Best Practices

### 1. Use Descriptive Flag Names

```typescript
// Bad: Vague names
const flag1 = featureFlags.isEnabled('new');
const flag2 = featureFlags.isEnabled('test');

// Good: Clear, descriptive names
const hasRedesignedCheckout = featureFlags.isEnabled('redesignedCheckoutFlow');
const hasAdvancedSearch = featureFlags.isEnabled('advancedSearchAlgorithm');
```

### 2. Document Flag Lifecycle

```typescript
/**
 * Feature Flag: redesignedCheckout
 * 
 * Created: 2024-06-01
 * Created by: john@example.com
 * Type: Release toggle
 * Lifecycle: Temporary (remove after 100% rollout)
 * 
 * Rollout Plan:
 * - 2024-06-01: 10% (canary)
 * - 2024-06-05: 50% (general rollout)
 * - 2024-06-10: 100% (full availability)
 * - 2024-07-10: Remove flag (30 days after 100%)
 * 
 * Dependencies: None
 * Metrics: conversion_rate, cart_abandonment
 */
const hasRedesignedCheckout = featureFlags.isEnabled('redesignedCheckout');
```

### 3. Implement Flag Expiration

```typescript
// flag-expiration.ts
interface ExpiringFlag {
  name: string;
  expiresAt: Date;
  action: 'warn' | 'disable' | 'remove';
}

class FlagExpiration {
  private expiringFlags: Map<string, ExpiringFlag> = new Map();
  
  setExpiration(config: ExpiringFlag): void {
    this.expiringFlags.set(config.name, config);
  }
  
  checkExpiration(flagName: string): void {
    const config = this.expiringFlags.get(flagName);
    if (!config) return;
    
    const now = Date.now();
    if (now > config.expiresAt.getTime()) {
      switch (config.action) {
        case 'warn':
          console.warn(`Flag ${flagName} has expired. Please review.`);
          break;
        case 'disable':
          featureFlags.disable(flagName);
          console.warn(`Flag ${flagName} automatically disabled due to expiration.`);
          break;
        case 'remove':
          throw new Error(`Flag ${flagName} has expired and should be removed from code.`);
      }
    }
  }
}

export const flagExpiration = new FlagExpiration();

// Set expiration
flagExpiration.setExpiration({
  name: 'redesignedCheckout',
  expiresAt: new Date('2024-07-10'),
  action: 'warn',
});
```

### 4. Use Type-Safe Flags

```typescript
// feature-flags-typed.ts
export enum FeatureFlag {
  NewDashboard = 'newDashboard',
  DarkMode = 'darkMode',
  AdvancedAnalytics = 'advancedAnalytics',
  ExperimentalSearch = 'experimentalSearch',
}

class TypeSafeFeatureFlags {
  isEnabled(flag: FeatureFlag): boolean {
    return featureFlags.isEnabled(flag);
  }
}

export const typeSafeFlags = new TypeSafeFeatureFlags();

// Usage with type safety
const enabled = typeSafeFlags.isEnabled(FeatureFlag.NewDashboard);
```

### 5. Test Both Code Paths

```typescript
// feature.test.tsx
describe('Dashboard', () => {
  it('renders new dashboard when flag enabled', () => {
    featureFlags.enable('newDashboard');
    
    const { getByText } = render(<Dashboard />);
    expect(getByText('New Dashboard')).toBeInTheDocument();
  });
  
  it('renders old dashboard when flag disabled', () => {
    featureFlags.disable('newDashboard');
    
    const { getByText } = render(<Dashboard />);
    expect(getByText('Old Dashboard')).toBeInTheDocument();
  });
});
```

## When to Use/Not to Use

### When to Use Feature Flags

1. **Gradual Rollouts**: Release features to subset of users
2. **A/B Testing**: Experiment with variations
3. **Kill Switches**: Emergency disable of problematic features
4. **Trunk-Based Development**: Merge incomplete features
5. **Beta Programs**: Early access for select users
6. **Operational Control**: Circuit breakers, rate limiting
7. **Permission Control**: Feature access by plan/role
8. **Dark Launches**: Deploy without exposing to users

### When Not to Use Feature Flags

1. **Simple Static Content**: No need for flags on static pages
2. **Database Schema Changes**: Use migrations instead
3. **Security Fixes**: Deploy immediately, don't flag
4. **Configuration**: Use environment variables instead
5. **A/B Tests on Every Element**: Too many flags = complexity
6. **Long-Term Branches**: Flags are not a substitute for proper branching
7. **UI Themes**: Use proper theming system
8. **Permanent Behavior**: If it should always work this way, no flag needed

## Interview Questions

### 1. What's the difference between feature flags and feature branches?

**Answer**: Feature branches isolate code changes in version control until ready to merge. Feature flags allow merging incomplete features to main branch, hidden behind flags. Flags enable continuous deployment with features turned off, gradual rollouts, and instant rollbacks. Feature branches require code deployment to change; flags can be toggled in production instantly. Use trunk-based development with flags for faster integration and reduced merge conflicts.

### 2. How do you implement a gradual rollout with feature flags?

**Answer**: Start with percentage-based targeting (10% → 25% → 50% → 100%). Use consistent hashing to ensure same users always get same experience. Monitor metrics at each stage (error rates, performance, conversion). Implement kill switch for instant rollback. Consider ring-based rollout (internal → canary → early adopters → general). Use LaunchDarkly or similar for sophisticated targeting and gradual percentage increases.

### 3. What are the different types of feature flags?

**Answer**: Release toggles (hide incomplete features, temporary), experiment toggles (A/B testing, medium-term), ops toggles (kill switches, circuit breakers, permanent), permission toggles (plan-based access, permanent). Each has different lifecycle and should be managed accordingly. Release toggles should be removed after full rollout. Ops toggles stay permanent for operational control.

### 4. How do you prevent feature flag technical debt?

**Answer**: Set expiration dates on flags. Document when to remove flags. Automate detection of fully-rolled-out flags. Create tickets for flag removal. Review flags quarterly. Use linting tools to detect stale flags. Remove flags 30 days after 100% rollout. Track flag age in dashboard. Make flag removal part of definition of done.

### 5. How would you implement a kill switch for a critical feature?

**Answer**: Create ops toggle with immediate effect. Implement real-time flag updates (websockets or polling). Add monitoring to auto-trigger based on error rate/latency. Provide admin UI for manual toggle. Alert team when triggered. Log all state changes. Implement graceful degradation (fallback behavior). Test kill switch regularly. Document recovery procedures.

### 6. What's the difference between client-side and server-side feature flags?

**Answer**: Client-side flags evaluated in browser (JavaScript), suitable for UI changes, fast evaluation, exposed in network requests. Server-side flags evaluated on backend, suitable for API behavior, database queries, secure (not exposed), can use server-side user attributes. Use both: client-side for UI, server-side for business logic and sensitive features.

### 7. How do you handle feature flag consistency in distributed systems?

**Answer**: Use centralized feature flag service with caching. Implement consistent hashing for user assignments. Cache flag values with short TTL (5-60 seconds). Propagate flag changes via pub/sub. Handle flag evaluation failures with defaults. Use distributed tracing with flag context. Log flag states in requests for debugging. Consider eventual consistency acceptable for most flags.

### 8. How would you implement multivariate testing with feature flags?

**Answer**: Define experiment with multiple variants and weights. Assign users to variants using consistent hashing. Track exposure and conversion events. Calculate statistical significance. Segment by user attributes. Support targeting rules. Provide winner selection and gradual rollout of winning variant. Integrate with analytics platform. Use tools like LaunchDarkly, Optimizely, or build custom system.

## Key Takeaways

1. **Feature flags enable risk-free deployments**: Deploy code, enable features gradually
2. **Different flag types have different lifecycles**: Remove release toggles, keep ops toggles
3. **Clean up flags regularly**: Technical debt accumulates quickly
4. **Use consistent hashing for assignments**: Same users get same experience
5. **Implement kill switches for critical features**: Instant rollback capability
6. **Combine flags with monitoring**: Track metrics for each variant
7. **Use feature flag services**: LaunchDarkly, Split.io provide sophisticated targeting
8. **Test both code paths**: Ensure flags work when on and off
9. **Document flag purpose and lifecycle**: When created, when to remove
10. **Gradual rollouts reduce risk**: 10% → 50% → 100% with monitoring

## Resources

### Feature Flag Platforms
- [LaunchDarkly](https://launchdarkly.com/) - Enterprise feature management
- [Split.io](https://www.split.io/) - Feature flags and experimentation
- [Optimizely](https://www.optimizely.com/) - Experimentation platform
- [Unleash](https://www.getunleash.io/) - Open-source feature toggles
- [Flagsmith](https://www.flagsmith.com/) - Open-source feature flags

### Documentation
- [LaunchDarkly Docs](https://docs.launchdarkly.com/)
- [Feature Toggles (Martin Fowler)](https://martinfowler.com/articles/feature-toggles.html)
- [LaunchDarkly React SDK](https://docs.launchdarkly.com/sdk/client-side/react)

### Tools
- [unleash/unleash](https://github.com/Unleash/unleash) - Open-source alternative
- [flagsmith/flagsmith](https://github.com/Flagsmith/flagsmith) - Open-source flags

### Articles
- [Feature Flags Best Practices](https://launchdarkly.com/blog/dos-and-donts-of-feature-flags/)
- [Kill Your Feature Flags](https://www.split.io/blog/when-to-kill-feature-flags/)
- [Feature Flag Driven Development](https://trunkbaseddevelopment.com/feature-flags/)
