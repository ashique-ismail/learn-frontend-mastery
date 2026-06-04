# Alerting and On-Call

## Overview

Alerting and on-call management are critical components of production operations that ensure rapid response to incidents. Effective alerting systems notify the right people at the right time with actionable information, while on-call processes ensure 24/7 coverage and clear incident response procedures. This includes alert design, PagerDuty integration, SLOs/SLIs, runbooks, incident management, escalation policies, and on-call rotation strategies.

## Table of Contents
- [Alert Design Principles](#alert-design-principles)
- [Service Level Objectives (SLOs) and Indicators (SLIs)](#service-level-objectives-slos-and-indicators-slis)
- [PagerDuty Integration](#pagerduty-integration)
- [Runbooks](#runbooks)
- [Incident Response](#incident-response)
- [Escalation Policies](#escalation-policies)
- [On-Call Rotation](#on-call-rotation)
- [Alert Fatigue Prevention](#alert-fatigue-prevention)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use/Not to Use](#when-to-usenot-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Alert Design Principles

### The Five Golden Rules of Alerting

```javascript
// 1. Alert on symptoms, not causes
// Bad: Alert on high CPU usage
if (cpuUsage > 80) {
  alert('High CPU usage'); // Not actionable - might be normal
}

// Good: Alert on user-facing impact
if (responseTime > SLO && errorRate > threshold) {
  alert('Service degradation affecting users', {
    responseTime,
    errorRate,
    affectedEndpoints: ['/api/users', '/api/orders'],
  });
}

// 2. Alert on what matters (SLI breaches)
// Bad: Alert on everything
alert('Disk usage at 60%'); // Not urgent

// Good: Alert on SLO violations
if (availability < 99.9) {
  alert('SLO violation: Availability below target', {
    current: availability,
    target: 99.9,
    impact: 'High',
  });
}

// 3. Every alert must be actionable
// Bad: Vague alert
alert('Something is wrong');

// Good: Specific, actionable alert
alert('Database connection pool exhausted', {
  action: 'Scale up connection pool or investigate slow queries',
  runbook: 'https://wiki.company.com/runbooks/db-pool-exhausted',
  currentPool: 95,
  maxPool: 100,
});

// 4. Alert severity must be accurate
enum AlertSeverity {
  INFO = 'info',           // FYI, no action needed
  WARNING = 'warning',     // Investigate during business hours
  ERROR = 'error',         // Fix within 4 hours
  CRITICAL = 'critical',   // Page immediately, fix ASAP
}

// 5. Provide context and next steps
interface Alert {
  title: string;
  severity: AlertSeverity;
  description: string;
  impact: string;
  affectedServices: string[];
  metrics: Record<string, number>;
  runbook: string;
  suggestedActions: string[];
}
```

### Alert Structure

```typescript
// alert-manager.ts
interface AlertContext {
  // Identification
  alertId: string;
  alertName: string;
  timestamp: number;
  
  // Severity and impact
  severity: 'info' | 'warning' | 'error' | 'critical';
  priority: 'P1' | 'P2' | 'P3' | 'P4';
  impact: {
    affectedUsers?: number;
    affectedServices: string[];
    businessImpact: string;
  };
  
  // Context
  service: string;
  environment: 'production' | 'staging' | 'development';
  region?: string;
  cluster?: string;
  
  // Metrics
  metrics: Record<string, number>;
  threshold: number;
  currentValue: number;
  
  // Response
  runbookUrl: string;
  suggestedActions: string[];
  dashboardUrl?: string;
  logQuery?: string;
  
  // Attribution
  owner: string;
  oncallTeam: string;
}

class AlertManager {
  async sendAlert(alert: AlertContext): Promise<void> {
    // Validate alert before sending
    this.validateAlert(alert);
    
    // Deduplicate
    if (await this.isDuplicate(alert)) {
      console.log(`Suppressing duplicate alert: ${alert.alertName}`);
      return;
    }
    
    // Enrich with additional context
    const enrichedAlert = await this.enrichAlert(alert);
    
    // Route to appropriate channel(s)
    await this.routeAlert(enrichedAlert);
    
    // Store in incident database
    await this.storeAlert(enrichedAlert);
    
    // Track alert metrics
    this.trackAlertMetrics(enrichedAlert);
  }
  
  private validateAlert(alert: AlertContext): void {
    if (!alert.runbookUrl) {
      throw new Error('Alert must include runbook URL');
    }
    
    if (alert.suggestedActions.length === 0) {
      throw new Error('Alert must include suggested actions');
    }
    
    if (alert.severity === 'critical' && !alert.oncallTeam) {
      throw new Error('Critical alerts must specify oncall team');
    }
  }
  
  private async isDuplicate(alert: AlertContext): Promise<boolean> {
    // Check if same alert fired in last 5 minutes
    const recentAlerts = await this.getRecentAlerts(5);
    return recentAlerts.some(a => 
      a.alertName === alert.alertName &&
      a.service === alert.service &&
      Math.abs(a.currentValue - alert.currentValue) < alert.threshold * 0.1
    );
  }
  
  private async enrichAlert(alert: AlertContext): Promise<AlertContext> {
    // Add dashboard links
    alert.dashboardUrl = `https://grafana.company.com/d/${alert.service}`;
    
    // Add log query
    alert.logQuery = `service:${alert.service} level:error timestamp:[now-5m TO now]`;
    
    // Calculate affected users
    if (!alert.impact.affectedUsers) {
      alert.impact.affectedUsers = await this.estimateAffectedUsers(alert);
    }
    
    return alert;
  }
  
  private async routeAlert(alert: AlertContext): Promise<void> {
    switch (alert.severity) {
      case 'critical':
        await this.sendToPagerDuty(alert);
        await this.sendToSlack(alert, '#incidents');
        break;
      case 'error':
        await this.sendToSlack(alert, '#alerts');
        await this.sendEmail(alert);
        break;
      case 'warning':
        await this.sendToSlack(alert, '#monitoring');
        break;
      case 'info':
        await this.logAlert(alert);
        break;
    }
  }
  
  private async sendToPagerDuty(alert: AlertContext): Promise<void> {
    // Implementation in PagerDuty section
  }
  
  private async sendToSlack(alert: AlertContext, channel: string): Promise<void> {
    // Implementation below
  }
  
  private async sendEmail(alert: AlertContext): Promise<void> {
    // Implementation below
  }
  
  private async logAlert(alert: AlertContext): Promise<void> {
    console.log('[ALERT]', alert);
  }
  
  private async storeAlert(alert: AlertContext): Promise<void> {
    // Store in database for analysis
  }
  
  private trackAlertMetrics(alert: AlertContext): void {
    // Track alert volume, MTTA, MTTR
  }
  
  private async getRecentAlerts(minutes: number): Promise<AlertContext[]> {
    // Fetch recent alerts from database
    return [];
  }
  
  private async estimateAffectedUsers(alert: AlertContext): Promise<number> {
    // Query metrics to estimate user impact
    return 0;
  }
}

export const alertManager = new AlertManager();
```

## Service Level Objectives (SLOs) and Indicators (SLIs)

### Defining SLIs and SLOs

```typescript
// slo-manager.ts
interface SLI {
  name: string;
  description: string;
  query: string;
  unit: 'percentage' | 'milliseconds' | 'count';
}

interface SLO {
  name: string;
  description: string;
  sli: SLI;
  target: number;
  window: '1h' | '24h' | '7d' | '30d';
  alertThreshold: number; // Alert when remaining error budget < this %
}

class SLOManager {
  private slos: Map<string, SLO> = new Map();
  private errorBudgets: Map<string, number> = new Map();
  
  defineSLO(slo: SLO): void {
    this.slos.set(slo.name, slo);
    
    // Calculate initial error budget
    this.errorBudgets.set(slo.name, 100 - slo.target);
  }
  
  async checkSLO(sloName: string): Promise<{
    current: number;
    target: number;
    isViolated: boolean;
    errorBudgetRemaining: number;
    shouldAlert: boolean;
  }> {
    const slo = this.slos.get(sloName);
    if (!slo) {
      throw new Error(`SLO ${sloName} not found`);
    }
    
    // Query current SLI value
    const current = await this.querySLI(slo.sli);
    
    // Calculate error budget
    const errorBudget = 100 - slo.target;
    const errorSpent = 100 - current;
    const errorBudgetRemaining = ((errorBudget - errorSpent) / errorBudget) * 100;
    
    // Check if should alert
    const shouldAlert = errorBudgetRemaining < slo.alertThreshold;
    
    return {
      current,
      target: slo.target,
      isViolated: current < slo.target,
      errorBudgetRemaining,
      shouldAlert,
    };
  }
  
  private async querySLI(sli: SLI): Promise<number> {
    // Query monitoring system for SLI value
    // This would integrate with Prometheus, Datadog, etc.
    return 99.5; // Placeholder
  }
  
  async burnRateAlert(sloName: string): Promise<void> {
    // Fast burn rate: 2% error budget consumed in 1 hour
    // Slow burn rate: 10% error budget consumed in 6 hours
    
    const slo = this.slos.get(sloName);
    if (!slo) return;
    
    const burnRate = await this.calculateBurnRate(slo, '1h');
    
    if (burnRate > 14.4) {
      // At this rate, entire error budget will be consumed in < 2 hours
      alertManager.sendAlert({
        alertId: `slo-burn-rate-${sloName}`,
        alertName: 'Fast SLO Burn Rate',
        timestamp: Date.now(),
        severity: 'critical',
        priority: 'P1',
        service: slo.name,
        environment: 'production',
        impact: {
          affectedServices: [slo.name],
          businessImpact: 'Rapid error budget depletion',
        },
        metrics: {
          burnRate,
          errorBudgetRemaining: await this.getErrorBudget(sloName),
        },
        threshold: 14.4,
        currentValue: burnRate,
        runbookUrl: `https://wiki.company.com/runbooks/slo-burn-rate`,
        suggestedActions: [
          'Investigate recent deployments or traffic changes',
          'Check error logs for patterns',
          'Consider rolling back recent changes',
        ],
        owner: 'sre-team',
        oncallTeam: 'platform',
      });
    }
  }
  
  private async calculateBurnRate(slo: SLO, window: string): Promise<number> {
    // Calculate how fast error budget is being consumed
    // Burn rate = (errors in window) / (allowable errors in window)
    return 1.0; // Placeholder
  }
  
  private async getErrorBudget(sloName: string): Promise<number> {
    return this.errorBudgets.get(sloName) || 0;
  }
}

export const sloManager = new SLOManager();

// Define SLOs
sloManager.defineSLO({
  name: 'api-availability',
  description: 'API availability SLO',
  sli: {
    name: 'availability',
    description: 'Percentage of successful requests',
    query: 'sum(rate(http_requests_total{status!~"5.."}[5m])) / sum(rate(http_requests_total[5m]))',
    unit: 'percentage',
  },
  target: 99.9, // 99.9% availability
  window: '30d',
  alertThreshold: 10, // Alert when <10% error budget remaining
});

sloManager.defineSLO({
  name: 'api-latency',
  description: 'API latency SLO',
  sli: {
    name: 'p95-latency',
    description: '95th percentile response time',
    query: 'histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))',
    unit: 'milliseconds',
  },
  target: 200, // p95 < 200ms
  window: '24h',
  alertThreshold: 20,
});

// Check SLOs periodically
setInterval(async () => {
  const availabilityStatus = await sloManager.checkSLO('api-availability');
  
  if (availabilityStatus.shouldAlert) {
    alertManager.sendAlert({
      alertId: 'slo-violation-availability',
      alertName: 'SLO Violation: API Availability',
      timestamp: Date.now(),
      severity: 'error',
      priority: 'P2',
      service: 'api',
      environment: 'production',
      impact: {
        affectedServices: ['api'],
        businessImpact: 'Reduced reliability, potential customer impact',
      },
      metrics: {
        current: availabilityStatus.current,
        target: availabilityStatus.target,
        errorBudgetRemaining: availabilityStatus.errorBudgetRemaining,
      },
      threshold: availabilityStatus.target,
      currentValue: availabilityStatus.current,
      runbookUrl: 'https://wiki.company.com/runbooks/slo-availability',
      suggestedActions: [
        'Check error logs for patterns',
        'Verify infrastructure health',
        'Review recent deployments',
      ],
      owner: 'backend-team',
      oncallTeam: 'platform',
    });
  }
}, 60000); // Check every minute
```

## PagerDuty Integration

### PagerDuty Setup

```typescript
// pagerduty-client.ts
import axios from 'axios';

interface PagerDutyEvent {
  routing_key: string;
  event_action: 'trigger' | 'acknowledge' | 'resolve';
  dedup_key?: string;
  payload: {
    summary: string;
    severity: 'critical' | 'error' | 'warning' | 'info';
    source: string;
    timestamp?: string;
    component?: string;
    group?: string;
    class?: string;
    custom_details?: Record<string, any>;
  };
  links?: Array<{
    href: string;
    text: string;
  }>;
  images?: Array<{
    src: string;
    href?: string;
    alt?: string;
  }>;
}

class PagerDutyClient {
  private apiKey: string;
  private routingKey: string;
  
  constructor(apiKey: string, routingKey: string) {
    this.apiKey = apiKey;
    this.routingKey = routingKey;
  }
  
  async triggerIncident(alert: AlertContext): Promise<string> {
    const event: PagerDutyEvent = {
      routing_key: this.routingKey,
      event_action: 'trigger',
      dedup_key: alert.alertId,
      payload: {
        summary: alert.alertName,
        severity: this.mapSeverity(alert.severity),
        source: alert.service,
        timestamp: new Date(alert.timestamp).toISOString(),
        component: alert.service,
        group: alert.environment,
        custom_details: {
          metrics: alert.metrics,
          affectedUsers: alert.impact.affectedUsers,
          runbook: alert.runbookUrl,
          suggestedActions: alert.suggestedActions,
        },
      },
      links: [
        {
          href: alert.runbookUrl,
          text: 'Runbook',
        },
        {
          href: alert.dashboardUrl || '',
          text: 'Dashboard',
        },
      ],
    };
    
    const response = await axios.post(
      'https://events.pagerduty.com/v2/enqueue',
      event
    );
    
    return response.data.dedup_key;
  }
  
  async acknowledgeIncident(dedupKey: string): Promise<void> {
    await axios.post('https://events.pagerduty.com/v2/enqueue', {
      routing_key: this.routingKey,
      event_action: 'acknowledge',
      dedup_key: dedupKey,
    });
  }
  
  async resolveIncident(dedupKey: string): Promise<void> {
    await axios.post('https://events.pagerduty.com/v2/enqueue', {
      routing_key: this.routingKey,
      event_action: 'resolve',
      dedup_key: dedupKey,
    });
  }
  
  private mapSeverity(severity: string): 'critical' | 'error' | 'warning' | 'info' {
    const mapping: Record<string, 'critical' | 'error' | 'warning' | 'info'> = {
      critical: 'critical',
      error: 'error',
      warning: 'warning',
      info: 'info',
    };
    return mapping[severity] || 'error';
  }
  
  async createIncident(incident: {
    title: string;
    service_id: string;
    urgency: 'high' | 'low';
    body: {
      type: 'incident_body';
      details: string;
    };
  }): Promise<string> {
    const response = await axios.post(
      'https://api.pagerduty.com/incidents',
      { incident },
      {
        headers: {
          'Authorization': `Token token=${this.apiKey}`,
          'Content-Type': 'application/json',
          'Accept': 'application/vnd.pagerduty+json;version=2',
        },
      }
    );
    
    return response.data.incident.id;
  }
  
  async getOnCallUser(scheduleId: string): Promise<{
    id: string;
    name: string;
    email: string;
  }> {
    const response = await axios.get(
      `https://api.pagerduty.com/schedules/${scheduleId}/users`,
      {
        headers: {
          'Authorization': `Token token=${this.apiKey}`,
          'Accept': 'application/vnd.pagerduty+json;version=2',
        },
      }
    );
    
    const user = response.data.users[0];
    return {
      id: user.id,
      name: user.name,
      email: user.email,
    };
  }
}

export const pagerDutyClient = new PagerDutyClient(
  process.env.PAGERDUTY_API_KEY || '',
  process.env.PAGERDUTY_ROUTING_KEY || ''
);

// Usage
alertManager.sendToPagerDuty = async (alert: AlertContext) => {
  try {
    const dedupKey = await pagerDutyClient.triggerIncident(alert);
    console.log(`PagerDuty incident created: ${dedupKey}`);
  } catch (error) {
    console.error('Failed to create PagerDuty incident:', error);
  }
};
```

### Slack Integration

```typescript
// slack-client.ts
interface SlackMessage {
  text: string;
  blocks?: any[];
  thread_ts?: string;
}

class SlackClient {
  private webhookUrl: string;
  
  constructor(webhookUrl: string) {
    this.webhookUrl = webhookUrl;
  }
  
  async sendAlert(alert: AlertContext, channel: string): Promise<void> {
    const message = this.formatAlertMessage(alert);
    
    await axios.post(this.webhookUrl, message);
  }
  
  private formatAlertMessage(alert: AlertContext): SlackMessage {
    const emoji = this.getSeverityEmoji(alert.severity);
    const color = this.getSeverityColor(alert.severity);
    
    return {
      text: `${emoji} ${alert.alertName}`,
      blocks: [
        {
          type: 'header',
          text: {
            type: 'plain_text',
            text: `${emoji} ${alert.alertName}`,
          },
        },
        {
          type: 'section',
          fields: [
            {
              type: 'mrkdwn',
              text: `*Service:*\n${alert.service}`,
            },
            {
              type: 'mrkdwn',
              text: `*Environment:*\n${alert.environment}`,
            },
            {
              type: 'mrkdwn',
              text: `*Severity:*\n${alert.severity.toUpperCase()}`,
            },
            {
              type: 'mrkdwn',
              text: `*Affected Users:*\n${alert.impact.affectedUsers || 'Unknown'}`,
            },
          ],
        },
        {
          type: 'section',
          text: {
            type: 'mrkdwn',
            text: `*Impact:* ${alert.impact.businessImpact}`,
          },
        },
        {
          type: 'section',
          text: {
            type: 'mrkdwn',
            text: `*Current Value:* ${alert.currentValue}\n*Threshold:* ${alert.threshold}`,
          },
        },
        {
          type: 'section',
          text: {
            type: 'mrkdwn',
            text: `*Suggested Actions:*\n${alert.suggestedActions.map(a => `• ${a}`).join('\n')}`,
          },
        },
        {
          type: 'actions',
          elements: [
            {
              type: 'button',
              text: {
                type: 'plain_text',
                text: 'View Runbook',
              },
              url: alert.runbookUrl,
              style: 'primary',
            },
            {
              type: 'button',
              text: {
                type: 'plain_text',
                text: 'View Dashboard',
              },
              url: alert.dashboardUrl,
            },
            {
              type: 'button',
              text: {
                type: 'plain_text',
                text: 'Acknowledge',
              },
              action_id: 'acknowledge_alert',
              value: alert.alertId,
            },
          ],
        },
      ],
    };
  }
  
  private getSeverityEmoji(severity: string): string {
    const emojis: Record<string, string> = {
      critical: '🚨',
      error: '❌',
      warning: '⚠️',
      info: 'ℹ️',
    };
    return emojis[severity] || '📢';
  }
  
  private getSeverityColor(severity: string): string {
    const colors: Record<string, string> = {
      critical: '#FF0000',
      error: '#FF6B6B',
      warning: '#FFA500',
      info: '#4A90E2',
    };
    return colors[severity] || '#808080';
  }
}

export const slackClient = new SlackClient(
  process.env.SLACK_WEBHOOK_URL || ''
);

// Usage
alertManager.sendToSlack = async (alert: AlertContext, channel: string) => {
  try {
    await slackClient.sendAlert(alert, channel);
  } catch (error) {
    console.error('Failed to send Slack alert:', error);
  }
};
```

## Runbooks

### Runbook Structure

```markdown
# Runbook: High Error Rate

## Overview
This runbook covers response procedures for elevated error rates in the API service.

## Severity: P2 (Error)

## Impact
- Reduced user experience
- Potential data loss for failed requests
- Customer complaints

## Symptoms
- Error rate > 1%
- Alert: "High API Error Rate"
- Increased 5xx responses
- Elevated latency

## Diagnosis

### Step 1: Check Current Error Rate
```bash
# Query Prometheus
rate(http_requests_total{status=~"5.."}[5m])

# Or Datadog
avg:requests.errors{service:api}.as_rate()
```

### Step 2: Identify Error Patterns
```bash
# Check error logs
kubectl logs -l app=api --tail=100 | grep ERROR

# Check error distribution by endpoint
SELECT endpoint, COUNT(*) as error_count
FROM error_logs
WHERE timestamp > NOW() - INTERVAL '5 minutes'
GROUP BY endpoint
ORDER BY error_count DESC;
```

### Step 3: Check Dependencies
- Database: Check connection pool usage, slow queries
- Cache: Check Redis connection, hit rate
- External APIs: Check third-party service status

## Resolution Steps

### Immediate Actions (< 5 minutes)
1. Check if recent deployment occurred
   ```bash
   kubectl rollout history deployment/api
   ```

2. If recent deployment, consider rollback
   ```bash
   kubectl rollout undo deployment/api
   ```

3. Check if auto-scaling is working
   ```bash
   kubectl get hpa
   ```

### Short-term Actions (< 30 minutes)
1. Analyze error logs for root cause
2. Scale up if resource constrained
   ```bash
   kubectl scale deployment/api --replicas=10
   ```

3. Enable circuit breakers if external dependency failing
4. Increase timeouts if seeing timeout errors

### Long-term Actions
1. Fix root cause identified in logs
2. Add monitoring for detected failure mode
3. Update this runbook with new findings

## Escalation
- If error rate > 5%: Escalate to senior engineer
- If error rate > 10%: Escalate to engineering manager
- If unresolved in 30 minutes: Escalate to on-call lead

## Communication
- Post update in #incidents Slack channel every 15 minutes
- Update status page if customer-facing impact
- Notify customer support if significant impact

## Related Resources
- Dashboard: https://grafana.company.com/d/api-health
- Logs: https://kibana.company.com/app/logs?service=api
- Recent incidents: https://pagerduty.com/incidents?service=api

## Post-Incident
- Create post-mortem document
- Update this runbook with learnings
- Create follow-up tasks for improvements
```

### Runbook Management

```typescript
// runbook-manager.ts
interface Runbook {
  id: string;
  title: string;
  service: string;
  severity: 'P1' | 'P2' | 'P3' | 'P4';
  url: string;
  tags: string[];
  lastUpdated: Date;
  owner: string;
}

class RunbookManager {
  private runbooks: Map<string, Runbook> = new Map();
  
  registerRunbook(runbook: Runbook): void {
    this.runbooks.set(runbook.id, runbook);
  }
  
  findRunbook(alertName: string, service: string): Runbook | undefined {
    // Find relevant runbook based on alert and service
    return Array.from(this.runbooks.values()).find(
      rb => rb.service === service || rb.tags.includes(alertName)
    );
  }
  
  getRunbookUrl(alertName: string, service: string): string {
    const runbook = this.findRunbook(alertName, service);
    return runbook?.url || 'https://wiki.company.com/runbooks/general';
  }
  
  validateRunbooks(): {
    total: number;
    stale: number;
    missing: string[];
  } {
    const total = this.runbooks.size;
    const thirtyDaysAgo = Date.now() - (30 * 24 * 60 * 60 * 1000);
    
    const stale = Array.from(this.runbooks.values()).filter(
      rb => rb.lastUpdated.getTime() < thirtyDaysAgo
    ).length;
    
    // Check for missing runbooks for critical services
    const criticalServices = ['api', 'database', 'payment', 'auth'];
    const missing = criticalServices.filter(
      service => !Array.from(this.runbooks.values()).some(rb => rb.service === service)
    );
    
    return { total, stale, missing };
  }
}

export const runbookManager = new RunbookManager();

// Register runbooks
runbookManager.registerRunbook({
  id: 'high-error-rate',
  title: 'High Error Rate',
  service: 'api',
  severity: 'P2',
  url: 'https://wiki.company.com/runbooks/high-error-rate',
  tags: ['error', 'availability', '5xx'],
  lastUpdated: new Date('2024-06-01'),
  owner: 'backend-team',
});
```

## Incident Response

### Incident Management System

```typescript
// incident-manager.ts
interface Incident {
  id: string;
  title: string;
  severity: 'P1' | 'P2' | 'P3' | 'P4';
  status: 'investigating' | 'identified' | 'monitoring' | 'resolved';
  startTime: number;
  endTime?: number;
  impact: {
    affectedUsers: number;
    affectedServices: string[];
    businessImpact: string;
  };
  timeline: Array<{
    timestamp: number;
    action: string;
    author: string;
  }>;
  assignee: string;
  responders: string[];
  postMortemUrl?: string;
}

class IncidentManager {
  private incidents: Map<string, Incident> = new Map();
  
  async createIncident(alert: AlertContext): Promise<Incident> {
    const incident: Incident = {
      id: `INC-${Date.now()}`,
      title: alert.alertName,
      severity: this.mapAlertPriority(alert.priority),
      status: 'investigating',
      startTime: Date.now(),
      impact: alert.impact,
      timeline: [
        {
          timestamp: Date.now(),
          action: 'Incident created from alert',
          author: 'system',
        },
      ],
      assignee: await this.getOnCallEngineer(alert.oncallTeam),
      responders: [],
    };
    
    this.incidents.set(incident.id, incident);
    
    // Notify team
    await this.notifyIncidentCreated(incident);
    
    // Create PagerDuty incident
    await pagerDutyClient.createIncident({
      title: incident.title,
      service_id: process.env.PAGERDUTY_SERVICE_ID || '',
      urgency: incident.severity === 'P1' ? 'high' : 'low',
      body: {
        type: 'incident_body',
        details: JSON.stringify(alert, null, 2),
      },
    });
    
    return incident;
  }
  
  async updateStatus(
    incidentId: string,
    status: Incident['status'],
    author: string,
    notes?: string
  ): Promise<void> {
    const incident = this.incidents.get(incidentId);
    if (!incident) {
      throw new Error(`Incident ${incidentId} not found`);
    }
    
    incident.status = status;
    incident.timeline.push({
      timestamp: Date.now(),
      action: `Status updated to ${status}${notes ? `: ${notes}` : ''}`,
      author,
    });
    
    // Notify team of status change
    await this.notifyStatusChange(incident);
    
    // Auto-resolve after monitoring period
    if (status === 'monitoring') {
      setTimeout(() => {
        this.resolveIncident(incidentId, author);
      }, 30 * 60 * 1000); // 30 minutes
    }
  }
  
  async resolveIncident(incidentId: string, author: string): Promise<void> {
    const incident = this.incidents.get(incidentId);
    if (!incident) {
      throw new Error(`Incident ${incidentId} not found`);
    }
    
    incident.status = 'resolved';
    incident.endTime = Date.now();
    incident.timeline.push({
      timestamp: Date.now(),
      action: 'Incident resolved',
      author,
    });
    
    // Calculate metrics
    const duration = (incident.endTime - incident.startTime) / 1000 / 60; // minutes
    const mtta = this.calculateMTTA(incident);
    const mttr = duration;
    
    console.log(`Incident ${incidentId} resolved:`, {
      duration: `${duration.toFixed(2)} minutes`,
      mtta: `${mtta.toFixed(2)} minutes`,
      mttr: `${mttr.toFixed(2)} minutes`,
    });
    
    // Notify team
    await this.notifyIncidentResolved(incident);
    
    // Create post-mortem if P1/P2
    if (incident.severity === 'P1' || incident.severity === 'P2') {
      await this.createPostMortemTemplate(incident);
    }
  }
  
  private calculateMTTA(incident: Incident): number {
    // Mean Time To Acknowledge
    const acknowledgeAction = incident.timeline.find(
      entry => entry.action.includes('Incident acknowledged')
    );
    
    if (!acknowledgeAction) {
      return 0;
    }
    
    return (acknowledgeAction.timestamp - incident.startTime) / 1000 / 60;
  }
  
  private mapAlertPriority(priority: string): 'P1' | 'P2' | 'P3' | 'P4' {
    const mapping: Record<string, 'P1' | 'P2' | 'P3' | 'P4'> = {
      P1: 'P1',
      P2: 'P2',
      P3: 'P3',
      P4: 'P4',
    };
    return mapping[priority] || 'P3';
  }
  
  private async getOnCallEngineer(team: string): Promise<string> {
    // Get current on-call engineer from PagerDuty
    const scheduleId = process.env[`PAGERDUTY_SCHEDULE_${team.toUpperCase()}`];
    if (scheduleId) {
      const user = await pagerDutyClient.getOnCallUser(scheduleId);
      return user.email;
    }
    return 'unassigned';
  }
  
  private async notifyIncidentCreated(incident: Incident): Promise<void> {
    await slackClient.sendAlert(
      {
        alertId: incident.id,
        alertName: `New Incident: ${incident.title}`,
        timestamp: incident.startTime,
        severity: incident.severity === 'P1' ? 'critical' : 'error',
        priority: incident.severity,
        service: incident.impact.affectedServices[0] || 'unknown',
        environment: 'production',
        impact: incident.impact,
        metrics: {},
        threshold: 0,
        currentValue: 0,
        runbookUrl: '',
        suggestedActions: ['Check incident timeline for updates'],
        owner: incident.assignee,
        oncallTeam: '',
      },
      '#incidents'
    );
  }
  
  private async notifyStatusChange(incident: Incident): Promise<void> {
    // Send update to Slack
  }
  
  private async notifyIncidentResolved(incident: Incident): Promise<void> {
    // Send resolution notification
  }
  
  private async createPostMortemTemplate(incident: Incident): Promise<void> {
    // Create post-mortem document template
    const postMortem = `
# Post-Mortem: ${incident.title}

## Incident Summary
- **Incident ID:** ${incident.id}
- **Severity:** ${incident.severity}
- **Duration:** ${((incident.endTime! - incident.startTime) / 1000 / 60).toFixed(2)} minutes
- **Affected Users:** ${incident.impact.affectedUsers}
- **Business Impact:** ${incident.impact.businessImpact}

## Timeline
${incident.timeline.map(entry => 
  `- ${new Date(entry.timestamp).toISOString()}: ${entry.action} (${entry.author})`
).join('\n')}

## Root Cause
[To be filled]

## Resolution
[To be filled]

## Action Items
- [ ] [Action item 1]
- [ ] [Action item 2]

## Lessons Learned
[To be filled]
    `;
    
    console.log('Post-mortem template created:', postMortem);
  }
}

export const incidentManager = new IncidentManager();
```

## Escalation Policies

### Escalation Configuration

```typescript
// escalation-manager.ts
interface EscalationLevel {
  level: number;
  responders: string[];
  timeout: number; // minutes
  notificationChannels: ('email' | 'sms' | 'phone' | 'slack')[];
}

interface EscalationPolicy {
  name: string;
  service: string;
  levels: EscalationLevel[];
}

class EscalationManager {
  private policies: Map<string, EscalationPolicy> = new Map();
  
  definePolicy(policy: EscalationPolicy): void {
    this.policies.set(policy.name, policy);
  }
  
  async escalate(incidentId: string, policyName: string): Promise<void> {
    const policy = this.policies.get(policyName);
    if (!policy) {
      throw new Error(`Escalation policy ${policyName} not found`);
    }
    
    for (const level of policy.levels) {
      console.log(`Escalating to level ${level.level}:`, level.responders);
      
      // Notify responders
      await this.notifyResponders(incidentId, level);
      
      // Wait for acknowledgment or timeout
      const acknowledged = await this.waitForAcknowledgment(
        incidentId,
        level.timeout * 60 * 1000
      );
      
      if (acknowledged) {
        console.log(`Incident ${incidentId} acknowledged at level ${level.level}`);
        break;
      }
      
      console.log(`Level ${level.level} timeout, escalating to next level`);
    }
  }
  
  private async notifyResponders(
    incidentId: string,
    level: EscalationLevel
  ): Promise<void> {
    for (const channel of level.notificationChannels) {
      for (const responder of level.responders) {
        await this.sendNotification(incidentId, responder, channel);
      }
    }
  }
  
  private async sendNotification(
    incidentId: string,
    responder: string,
    channel: 'email' | 'sms' | 'phone' | 'slack'
  ): Promise<void> {
    const incident = incidentManager.incidents.get(incidentId);
    if (!incident) return;
    
    switch (channel) {
      case 'email':
        // Send email
        break;
      case 'sms':
        // Send SMS via Twilio
        break;
      case 'phone':
        // Make phone call via PagerDuty
        break;
      case 'slack':
        // Send DM via Slack
        break;
    }
  }
  
  private async waitForAcknowledgment(
    incidentId: string,
    timeout: number
  ): Promise<boolean> {
    // Poll for acknowledgment with timeout
    const startTime = Date.now();
    
    while (Date.now() - startTime < timeout) {
      const incident = incidentManager.incidents.get(incidentId);
      if (incident && incident.status !== 'investigating') {
        return true;
      }
      
      await new Promise(resolve => setTimeout(resolve, 5000)); // Check every 5 seconds
    }
    
    return false;
  }
}

export const escalationManager = new EscalationManager();

// Define escalation policies
escalationManager.definePolicy({
  name: 'api-escalation',
  service: 'api',
  levels: [
    {
      level: 1,
      responders: ['oncall-primary@company.com'],
      timeout: 5,
      notificationChannels: ['slack', 'sms'],
    },
    {
      level: 2,
      responders: ['oncall-primary@company.com', 'oncall-backup@company.com'],
      timeout: 10,
      notificationChannels: ['sms', 'phone'],
    },
    {
      level: 3,
      responders: ['engineering-manager@company.com', 'vp-engineering@company.com'],
      timeout: 15,
      notificationChannels: ['phone', 'email'],
    },
  ],
});
```

## On-Call Rotation

### On-Call Schedule Management

```typescript
// oncall-manager.ts
interface OnCallShift {
  id: string;
  team: string;
  engineer: string;
  startTime: Date;
  endTime: Date;
  isBackup: boolean;
}

interface OnCallRotation {
  team: string;
  rotationType: 'weekly' | 'biweekly' | 'daily';
  engineers: string[];
  startDate: Date;
}

class OnCallManager {
  private rotations: Map<string, OnCallRotation> = new Map();
  private shifts: OnCallShift[] = [];
  
  defineRotation(rotation: OnCallRotation): void {
    this.rotations.set(rotation.team, rotation);
    this.generateShifts(rotation);
  }
  
  private generateShifts(rotation: OnCallRotation): void {
    const daysPerShift = rotation.rotationType === 'weekly' ? 7 : 
                         rotation.rotationType === 'biweekly' ? 14 : 1;
    
    let currentDate = new Date(rotation.startDate);
    let engineerIndex = 0;
    
    // Generate shifts for next 3 months
    const endDate = new Date(currentDate);
    endDate.setMonth(endDate.getMonth() + 3);
    
    while (currentDate < endDate) {
      const shiftEnd = new Date(currentDate);
      shiftEnd.setDate(shiftEnd.getDate() + daysPerShift);
      
      // Primary on-call
      this.shifts.push({
        id: `shift-${Date.now()}-${engineerIndex}`,
        team: rotation.team,
        engineer: rotation.engineers[engineerIndex],
        startTime: new Date(currentDate),
        endTime: new Date(shiftEnd),
        isBackup: false,
      });
      
      // Backup on-call (next person in rotation)
      const backupIndex = (engineerIndex + 1) % rotation.engineers.length;
      this.shifts.push({
        id: `shift-backup-${Date.now()}-${engineerIndex}`,
        team: rotation.team,
        engineer: rotation.engineers[backupIndex],
        startTime: new Date(currentDate),
        endTime: new Date(shiftEnd),
        isBackup: true,
      });
      
      currentDate = shiftEnd;
      engineerIndex = (engineerIndex + 1) % rotation.engineers.length;
    }
  }
  
  getCurrentOnCall(team: string): {
    primary: string;
    backup: string;
  } | null {
    const now = Date.now();
    
    const primary = this.shifts.find(shift => 
      shift.team === team &&
      !shift.isBackup &&
      shift.startTime.getTime() <= now &&
      shift.endTime.getTime() > now
    );
    
    const backup = this.shifts.find(shift => 
      shift.team === team &&
      shift.isBackup &&
      shift.startTime.getTime() <= now &&
      shift.endTime.getTime() > now
    );
    
    if (!primary || !backup) {
      return null;
    }
    
    return {
      primary: primary.engineer,
      backup: backup.engineer,
    };
  }
  
  getUpcomingShifts(engineer: string, days: number = 30): OnCallShift[] {
    const now = Date.now();
    const endTime = now + (days * 24 * 60 * 60 * 1000);
    
    return this.shifts.filter(shift =>
      shift.engineer === engineer &&
      shift.startTime.getTime() >= now &&
      shift.startTime.getTime() <= endTime
    );
  }
  
  swapShift(shiftId: string, newEngineer: string, reason: string): void {
    const shift = this.shifts.find(s => s.id === shiftId);
    if (!shift) {
      throw new Error(`Shift ${shiftId} not found`);
    }
    
    const oldEngineer = shift.engineer;
    shift.engineer = newEngineer;
    
    console.log(`Shift swapped: ${oldEngineer} -> ${newEngineer} (${reason})`);
    
    // Notify team
    this.notifyShiftSwap(shift, oldEngineer, newEngineer, reason);
  }
  
  private notifyShiftSwap(
    shift: OnCallShift,
    oldEngineer: string,
    newEngineer: string,
    reason: string
  ): void {
    // Send notification to team
    slackClient.sendAlert(
      {
        alertId: `shift-swap-${shift.id}`,
        alertName: 'On-Call Shift Swap',
        timestamp: Date.now(),
        severity: 'info',
        priority: 'P4',
        service: shift.team,
        environment: 'production',
        impact: {
          affectedServices: [shift.team],
          businessImpact: `On-call coverage maintained`,
        },
        metrics: {},
        threshold: 0,
        currentValue: 0,
        runbookUrl: '',
        suggestedActions: [],
        owner: newEngineer,
        oncallTeam: shift.team,
      },
      '#oncall'
    );
  }
  
  getOnCallStats(engineer: string): {
    totalShifts: number;
    totalIncidents: number;
    avgIncidentsPerShift: number;
    avgResponseTime: number;
  } {
    // Calculate on-call statistics
    return {
      totalShifts: 0,
      totalIncidents: 0,
      avgIncidentsPerShift: 0,
      avgResponseTime: 0,
    };
  }
}

export const onCallManager = new OnCallManager();

// Define rotation
onCallManager.defineRotation({
  team: 'platform',
  rotationType: 'weekly',
  engineers: [
    'alice@company.com',
    'bob@company.com',
    'charlie@company.com',
    'diana@company.com',
  ],
  startDate: new Date('2024-06-01'),
});

// Get current on-call
const currentOnCall = onCallManager.getCurrentOnCall('platform');
console.log('Current on-call:', currentOnCall);
```

## Alert Fatigue Prevention

### Alert Noise Reduction

```typescript
// alert-noise-reduction.ts
interface AlertMetrics {
  alertName: string;
  fireCount: number;
  acknowledgeCount: number;
  resolveCount: number;
  avgResolutionTime: number;
  actionableRate: number; // % that led to action
}

class AlertFatiguePrevention {
  private alertHistory: Map<string, AlertMetrics> = new Map();
  
  async analyzeAlerts(days: number = 30): Promise<{
    noisyAlerts: string[];
    unusedAlerts: string[];
    suggestions: string[];
  }> {
    const metrics = Array.from(this.alertHistory.values());
    
    // Identify noisy alerts (high fire count, low action rate)
    const noisyAlerts = metrics
      .filter(m => m.fireCount > 100 && m.actionableRate < 0.3)
      .map(m => m.alertName);
    
    // Identify unused alerts (never acknowledged)
    const unusedAlerts = metrics
      .filter(m => m.acknowledgeCount === 0 && m.fireCount > 10)
      .map(m => m.alertName);
    
    const suggestions: string[] = [];
    
    // Generate suggestions
    noisyAlerts.forEach(alert => {
      suggestions.push(`Increase threshold for "${alert}" to reduce noise`);
    });
    
    unusedAlerts.forEach(alert => {
      suggestions.push(`Consider removing "${alert}" - never acknowledged`);
    });
    
    return { noisyAlerts, unusedAlerts, suggestions };
  }
  
  shouldSuppress(alertName: string): boolean {
    const metrics = this.alertHistory.get(alertName);
    if (!metrics) return false;
    
    // Suppress if fired > 10 times in last hour with no acknowledgments
    const recentFires = metrics.fireCount;
    const recentAcks = metrics.acknowledgeCount;
    
    if (recentFires > 10 && recentAcks === 0) {
      console.warn(`Suppressing noisy alert: ${alertName}`);
      return true;
    }
    
    return false;
  }
  
  async reviewAlerts(): Promise<void> {
    const analysis = await this.analyzeAlerts();
    
    console.log('Alert Analysis:');
    console.log(`Noisy alerts: ${analysis.noisyAlerts.length}`);
    console.log(`Unused alerts: ${analysis.unusedAlerts.length}`);
    console.log('Suggestions:');
    analysis.suggestions.forEach(s => console.log(`- ${s}`));
  }
}

export const alertFatiguePrevention = new AlertFatiguePrevention();

// Schedule weekly review
setInterval(() => {
  alertFatiguePrevention.reviewAlerts();
}, 7 * 24 * 60 * 60 * 1000); // Weekly
```

## Common Mistakes

### 1. Alert on Everything

```typescript
// Bad: Too many alerts
if (diskUsage > 60) {
  alert('Disk usage high'); // Not urgent
}

// Good: Alert on critical thresholds only
if (diskUsage > 90) {
  alert('Critical: Disk usage >90%', {
    action: 'Clean up logs or scale storage',
  });
}
```

### 2. Missing Runbook URLs

```typescript
// Bad: No guidance
alertManager.sendAlert({
  alertName: 'High error rate',
  // No runbook!
});

// Good: Always include runbook
alertManager.sendAlert({
  alertName: 'High error rate',
  runbookUrl: 'https://wiki.company.com/runbooks/high-error-rate',
  suggestedActions: ['Check logs', 'Verify deployment'],
});
```

### 3. Alert Storms

```typescript
// Bad: Send alert for each error
errors.forEach(error => {
  alert('Error occurred', error); // 1000 alerts!
});

// Good: Aggregate and alert once
if (errorRate > threshold) {
  alert('High error rate', {
    errorCount: errors.length,
    sampleErrors: errors.slice(0, 5),
  });
}
```

### 4. No Context in Alerts

```typescript
// Bad: Vague alert
alert('Service down');

// Good: Rich context
alert('API service degraded', {
  affectedEndpoints: ['/api/users', '/api/orders'],
  errorRate: '5.2%',
  affectedUsers: 1500,
  dashboard: 'https://grafana.company.com/d/api',
});
```

### 5. Wrong Severity

```typescript
// Bad: Everything is critical
alert('Log file rotated', { severity: 'critical' });

// Good: Appropriate severity
alert('Log file rotated', { severity: 'info' });
alert('SLO violation', { severity: 'error' });
alert('Service outage', { severity: 'critical' });
```

## Best Practices

### 1. Alert on User Impact

```typescript
// Alert on symptoms users experience
if (checkoutSuccessRate < 95) {
  alert('Checkout failures affecting users', {
    successRate: checkoutSuccessRate,
    affectedUsers: estimatedAffectedUsers,
    revenueImpact: estimatedRevenueLoss,
  });
}
```

### 2. Implement Alert Grouping

```typescript
// Group related alerts
const alertGroups = new Map();

function sendAlertWithGrouping(alert: AlertContext): void {
  const groupKey = `${alert.service}-${alert.alertName}`;
  
  if (alertGroups.has(groupKey)) {
    // Update existing alert instead of creating new one
    alertGroups.get(groupKey).count++;
  } else {
    alertGroups.set(groupKey, { ...alert, count: 1 });
    alertManager.sendAlert(alert);
  }
}
```

### 3. Regular Alert Review

```typescript
// Schedule monthly alert review
setInterval(async () => {
  const report = await generateAlertReport();
  
  console.log('Alert Health Report:');
  console.log(`Total alerts: ${report.total}`);
  console.log(`Alert fatigue score: ${report.fatigueScore}/100`);
  console.log(`Top noisy alerts:`, report.topNoisy);
  console.log(`Unused alerts:`, report.unused);
}, 30 * 24 * 60 * 60 * 1000);
```

### 4. Test Alerts Regularly

```typescript
// Test alert pipeline
async function testAlertPipeline(): Promise<void> {
  await alertManager.sendAlert({
    alertId: 'test-alert',
    alertName: 'Test Alert - Please Acknowledge',
    timestamp: Date.now(),
    severity: 'info',
    priority: 'P4',
    service: 'test',
    environment: 'production',
    impact: {
      affectedServices: ['test'],
      businessImpact: 'None - this is a test',
    },
    metrics: {},
    threshold: 0,
    currentValue: 0,
    runbookUrl: 'https://wiki.company.com/runbooks/test',
    suggestedActions: ['Acknowledge this test alert'],
    owner: 'sre-team',
    oncallTeam: 'platform',
  });
}

// Run test weekly
setInterval(testAlertPipeline, 7 * 24 * 60 * 60 * 1000);
```

### 5. Document On-Call Expectations

```markdown
# On-Call Expectations

## Responsibilities
- Respond to pages within 5 minutes
- Acknowledge incidents within 15 minutes
- Provide status updates every 30 minutes during active incidents
- Write post-mortem for P1/P2 incidents

## Tools Access
- PagerDuty mobile app
- VPN access
- Production system access
- Slack #incidents channel

## Escalation
- If unable to resolve in 30 minutes, escalate to senior engineer
- If impacting customers, notify customer support immediately
- If database-related, escalate to database team

## Handoff
- Update incident status before end of shift
- Brief incoming on-call engineer on active incidents
- Document any ongoing investigations
```

## When to Use/Not to Use

### When to Implement On-Call

1. **24/7 Services**: User-facing applications requiring high availability
2. **SLA Commitments**: Contractual uptime guarantees
3. **Mission-Critical Systems**: Payment, healthcare, infrastructure
4. **High-Traffic Applications**: Millions of users
5. **Distributed Teams**: Global coverage needed
6. **Regulatory Requirements**: Compliance mandates
7. **Growing Organizations**: More than 5-10 engineers
8. **Customer Expectations**: Users expect instant fixes

### When On-Call May Not Be Needed

1. **Internal Tools**: Office hours support sufficient
2. **Small Teams**: Less than 5 engineers
3. **Low-Traffic Sites**: Minimal user impact
4. **Batch Processing**: Next-day resolution acceptable
5. **Read-Only Services**: No data loss risk
6. **Development Environments**: Non-production
7. **Graceful Degradation**: System handles failures well
8. **Budget Constraints**: Can't support on-call compensation

## Interview Questions

### 1. What makes a good alert?

**Answer**: A good alert is actionable (clear what to do), has appropriate severity (not everything is critical), includes context (metrics, affected users), links to runbook, fires when users are impacted (symptoms not causes), and is not noisy (aggregated, deduplicated). Every alert should answer: what happened, why it matters, what to do.

### 2. Explain the difference between SLI, SLO, and SLA.

**Answer**: SLI (Service Level Indicator) is a measurement of service quality (e.g., latency, availability). SLO (Service Level Objective) is the target for that SLI (e.g., 99.9% availability). SLA (Service Level Agreement) is a contract with customers, often with penalties if SLO is breached. Example: SLI = availability, SLO = 99.9%, SLA = refund if < 99.5%.

### 3. How do you prevent alert fatigue?

**Answer**: Alert on symptoms not causes, set appropriate thresholds (not too sensitive), implement alert grouping/deduplication, suppress flapping alerts, regular alert review to remove noisy/unused alerts, use proper severity levels, make alerts actionable with runbooks, and track alert metrics (fire rate, acknowledge rate) to identify problems.

### 4. What should be included in a runbook?

**Answer**: Overview, severity, impact, symptoms, diagnosis steps (commands to run, logs to check), resolution steps (immediate, short-term, long-term), escalation procedures, communication guidelines, related resources (dashboard, logs), and post-incident tasks. Runbooks should be step-by-step guides that any on-call engineer can follow.

### 5. How do you calculate error budget?

**Answer**: Error budget = 100% - SLO target. For 99.9% SLO, error budget is 0.1%. In 30-day month, that's 43 minutes of downtime. Track error budget consumption: (actual failures / total requests) compared to allowed failures. Alert when error budget is being consumed too fast (burn rate) or when remaining budget is low.

### 6. What's the difference between MTTA and MTTR?

**Answer**: MTTA (Mean Time To Acknowledge) measures how long until someone acknowledges the alert. MTTR (Mean Time To Resolve/Recovery) measures total time from alert to resolution. Both are important incident metrics. Low MTTA shows good on-call responsiveness. Low MTTR shows effective troubleshooting and incident response.

### 7. How would you design an escalation policy?

**Answer**: Multi-level escalation with timeouts: Level 1 (primary on-call, 5 min timeout), Level 2 (backup on-call, 10 min timeout), Level 3 (manager/senior engineer, 15 min timeout). Use multiple notification channels (Slack, SMS, phone) with increasing urgency. Ensure someone is always available. Test escalation policy regularly. Document clearly who gets notified at each level.

### 8. How do you ensure fair on-call rotation?

**Answer**: Rotate on-call duties evenly among team members, provide adequate compensation (time off, pay), limit on-call duration (1 week shifts), provide backup on-call, allow shift swaps with approval, track on-call metrics per person (incident count, response time), schedule strategically (avoid holidays, events), and ensure proper training for all on-call engineers.

## Key Takeaways

1. **Alert on symptoms, not causes**: User-facing impact matters
2. **Every alert must be actionable**: Include runbook and suggested actions
3. **Use SLOs to define what matters**: Alert on SLO violations
4. **Implement proper escalation**: Multi-level with timeouts
5. **Prevent alert fatigue**: Regular review, appropriate thresholds
6. **Maintain runbooks**: Up-to-date, step-by-step guides
7. **Track incident metrics**: MTTA, MTTR, error budget
8. **Fair on-call rotation**: Equitable distribution, proper compensation
9. **Test alerting regularly**: Ensure pipeline works
10. **Post-incident reviews**: Learn from every incident

## Resources

### Alerting Platforms
- [PagerDuty](https://www.pagerduty.com/) - Incident management
- [Opsgenie](https://www.atlassian.com/software/opsgenie) - On-call management
- [VictorOps](https://victorops.com/) - Incident response
- [AlertManager](https://prometheus.io/docs/alerting/latest/alertmanager/) - Prometheus alerting

### Documentation
- [Google SRE Book - Monitoring](https://sre.google/sre-book/monitoring-distributed-systems/)
- [PagerDuty Incident Response](https://response.pagerduty.com/)
- [Atlassian Incident Management](https://www.atlassian.com/incident-management)

### Tools
- [prometheus/alertmanager](https://github.com/prometheus/alertmanager)
- [grafana/oncall](https://github.com/grafana/oncall) - Open-source

### Articles
- [Alert Design Best Practices](https://www.pagerduty.com/resources/learn/alert-design/)
- [My Philosophy on Alerting](https://docs.google.com/document/d/199PqyG3UsyXlwieHaqbGiWVa8eMWi8zzAn0YfcApr8Q)
- [SLO Best Practices](https://cloud.google.com/blog/products/devops-sre/sre-fundamentals-sli-vs-slo-vs-sla)
