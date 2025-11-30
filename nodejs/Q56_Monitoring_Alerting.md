# Q56: Monitoring - Alerting & SLA Management

## 📋 Summary
This question covers **intelligent alerting strategies**, SLA/SLO monitoring, incident management, and on-call best practices for production Node.js banking applications. Learn to build effective alerting systems that catch real issues without causing alert fatigue.

**Key Topics**:
- Alerting fundamentals and anti-patterns
- SLIs, SLOs, and SLAs
- Alert routing and escalation
- PagerDuty and incident management
- Error budgets
- Alert fatigue prevention
- Synthetic monitoring
- Anomaly detection
- Runbook automation
- Post-mortem analysis

**Banking Use Cases**:
- Transaction failure alerts
- Payment processing SLAs
- Account access latency monitoring
- Fraud detection alerts
- Database connection alerts
- API gateway health
- Compliance violation notifications
- Customer impact assessment

---

## 🎯 Understanding Alerting Fundamentals

### Alert Hierarchy

**Effective alerting** requires understanding severity levels and when to page humans vs. log for later review.

```
┌─────────────────────────────────────────────────────────────┐
│              Alert Severity Pyramid                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│                      ▲ CRITICAL                              │
│                     ███ (P0)                                 │
│                    █████                                     │
│                   ███████  Page immediately                  │
│                  █████████ Customer-impacting                │
│                                                              │
│              ▲ HIGH (P1)                                     │
│             ███████████████                                  │
│            █████████████████                                 │
│           ███████████████████ Escalate if unresolved        │
│          █████████████████████ Degraded performance         │
│                                                              │
│        ▲ MEDIUM (P2)                                         │
│       █████████████████████████                              │
│      ███████████████████████████                             │
│     █████████████████████████████ Business hours alert      │
│    ███████████████████████████████ Track and plan fix       │
│                                                              │
│  ▲ LOW (P3)                                                  │
│ ███████████████████████████████████                          │
│█████████████████████████████████████ Log and review         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### SLI, SLO, SLA Framework

| Concept | Definition | Banking Example |
|---------|------------|-----------------|
| **SLI** (Service Level Indicator) | Quantitative measure of service level | 99.5% of transfers complete in <3s |
| **SLO** (Service Level Objective) | Target value or range for SLI | ≥99.9% transfer success rate |
| **SLA** (Service Level Agreement) | Business contract with consequences | 99.95% uptime or 10% refund |
| **Error Budget** | Allowed failure rate (100% - SLO) | 0.1% failures = 43 minutes/month |

### Alert Quality Metrics

```typescript
// Key metrics for alert effectiveness
interface AlertMetrics {
  precision: number;     // True alerts / All alerts (avoid false positives)
  recall: number;        // True alerts / All incidents (catch real issues)
  timeToDetect: number;  // Incident start → Alert fired
  timeToResolve: number; // Alert fired → Incident resolved
  falsePositiveRate: number; // False alarms / All alerts
}

// Target: Precision > 90%, Recall > 95%
```

---

## 💡 Example 1: Complete Alerting System with PagerDuty

Production-ready alerting infrastructure for banking API.

### Project Structure

```
banking-api/
├── src/
│   ├── monitoring/
│   │   ├── alerts/
│   │   │   ├── AlertManager.ts
│   │   │   ├── PagerDutyClient.ts
│   │   │   └── SlackClient.ts
│   │   ├── slo/
│   │   │   ├── SLOTracker.ts
│   │   │   └── ErrorBudget.ts
│   │   └── health/
│   │       └── HealthCheck.ts
│   ├── services/
│   └── index.ts
├── config/
│   ├── alerts.yaml
│   └── slo.yaml
└── runbooks/
    ├── high-error-rate.md
    └── slow-response.md
```

### 1. Alert Manager

```typescript
// src/monitoring/alerts/AlertManager.ts

import { PagerDutyClient } from './PagerDutyClient';
import { SlackClient } from './SlackClient';

export enum AlertSeverity {
  CRITICAL = 'CRITICAL',  // P0 - Page immediately
  HIGH = 'HIGH',          // P1 - Escalate
  MEDIUM = 'MEDIUM',      // P2 - Business hours
  LOW = 'LOW'             // P3 - Log only
}

export interface Alert {
  id: string;
  severity: AlertSeverity;
  title: string;
  description: string;
  service: string;
  timestamp: Date;
  metadata: Record<string, any>;
  runbookUrl?: string;
  affectedCustomers?: number;
}

export interface AlertRule {
  id: string;
  name: string;
  condition: (metrics: any) => boolean;
  severity: AlertSeverity;
  cooldown: number;  // Minutes before re-alerting
  threshold: {
    value: number;
    duration: number;  // Sustained for X minutes
  };
  escalation?: {
    timeout: number;  // Minutes before escalation
    to: AlertSeverity;
  };
}

export class AlertManager {
  private pagerDuty: PagerDutyClient;
  private slack: SlackClient;
  private activeAlerts: Map<string, Alert> = new Map();
  private lastAlertTime: Map<string, Date> = new Map();
  private alertRules: AlertRule[] = [];

  constructor(
    pagerDutyApiKey: string,
    slackWebhookUrl: string
  ) {
    this.pagerDuty = new PagerDutyClient(pagerDutyApiKey);
    this.slack = new SlackClient(slackWebhookUrl);
    this.loadAlertRules();
  }

  /**
   * Trigger an alert
   */
  async trigger(alert: Alert): Promise<void> {
    const alertKey = `${alert.service}:${alert.title}`;

    // Check cooldown period
    if (this.isInCooldown(alertKey, alert.severity)) {
      console.log(`Alert ${alertKey} is in cooldown period, skipping`);
      return;
    }

    // Store alert
    this.activeAlerts.set(alert.id, alert);
    this.lastAlertTime.set(alertKey, new Date());

    // Route based on severity
    switch (alert.severity) {
      case AlertSeverity.CRITICAL:
        await this.handleCriticalAlert(alert);
        break;

      case AlertSeverity.HIGH:
        await this.handleHighAlert(alert);
        break;

      case AlertSeverity.MEDIUM:
        await this.handleMediumAlert(alert);
        break;

      case AlertSeverity.LOW:
        await this.handleLowAlert(alert);
        break;
    }

    console.log(`Alert triggered: ${alert.title} (${alert.severity})`);
  }

  /**
   * Resolve an alert
   */
  async resolve(alertId: string, resolution: string): Promise<void> {
    const alert = this.activeAlerts.get(alertId);

    if (!alert) {
      console.log(`Alert ${alertId} not found`);
      return;
    }

    // Notify PagerDuty
    if (alert.severity === AlertSeverity.CRITICAL || alert.severity === AlertSeverity.HIGH) {
      await this.pagerDuty.resolveIncident(alertId, resolution);
    }

    // Notify Slack
    await this.slack.sendMessage({
      text: `✅ Alert Resolved: ${alert.title}`,
      blocks: [
        {
          type: 'section',
          text: {
            type: 'mrkdwn',
            text: `*Alert Resolved*\n\n*Title:* ${alert.title}\n*Service:* ${alert.service}\n*Resolution:* ${resolution}`,
          },
        },
      ],
    });

    this.activeAlerts.delete(alertId);
    console.log(`Alert resolved: ${alert.title}`);
  }

  /**
   * Handle CRITICAL alerts (P0)
   */
  private async handleCriticalAlert(alert: Alert): Promise<void> {
    // Page on-call engineer immediately
    await this.pagerDuty.createIncident({
      title: `[CRITICAL] ${alert.title}`,
      description: alert.description,
      severity: 'critical',
      service: alert.service,
      urgency: 'high',
      metadata: alert.metadata,
    });

    // Send to Slack #incidents channel
    await this.slack.sendMessage({
      text: `🚨 CRITICAL ALERT: ${alert.title}`,
      blocks: [
        {
          type: 'header',
          text: {
            type: 'plain_text',
            text: '🚨 CRITICAL ALERT',
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
              text: `*Severity:*\nCRITICAL`,
            },
            {
              type: 'mrkdwn',
              text: `*Affected Customers:*\n${alert.affectedCustomers || 'Unknown'}`,
            },
            {
              type: 'mrkdwn',
              text: `*Time:*\n${alert.timestamp.toISOString()}`,
            },
          ],
        },
        {
          type: 'section',
          text: {
            type: 'mrkdwn',
            text: `*Description:*\n${alert.description}`,
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
              url: alert.runbookUrl || 'https://runbooks.example.com',
            },
          ],
        },
      ],
      channel: '#incidents',
    });

    // Auto-create incident ticket
    await this.createIncidentTicket(alert);
  }

  /**
   * Handle HIGH alerts (P1)
   */
  private async handleHighAlert(alert: Alert): Promise<void> {
    // Create PagerDuty incident with medium urgency
    await this.pagerDuty.createIncident({
      title: `[HIGH] ${alert.title}`,
      description: alert.description,
      severity: 'error',
      service: alert.service,
      urgency: 'high',
      metadata: alert.metadata,
    });

    // Send to Slack
    await this.slack.sendMessage({
      text: `⚠️ HIGH ALERT: ${alert.title}`,
      blocks: [
        {
          type: 'section',
          text: {
            type: 'mrkdwn',
            text: `*⚠️ HIGH ALERT*\n\n*Service:* ${alert.service}\n*Description:* ${alert.description}`,
          },
        },
      ],
      channel: '#alerts',
    });
  }

  /**
   * Handle MEDIUM alerts (P2)
   */
  private async handleMediumAlert(alert: Alert): Promise<void> {
    // Only notify during business hours
    if (this.isBusinessHours()) {
      await this.slack.sendMessage({
        text: `ℹ️ Medium Alert: ${alert.title}`,
        blocks: [
          {
            type: 'section',
            text: {
              type: 'mrkdwn',
              text: `*ℹ️ Medium Alert*\n\n*Service:* ${alert.service}\n*Description:* ${alert.description}`,
            },
          },
        ],
        channel: '#monitoring',
      });
    }

    // Always log for review
    console.log(`[MEDIUM ALERT] ${alert.title}:`, alert.metadata);
  }

  /**
   * Handle LOW alerts (P3)
   */
  private async handleLowAlert(alert: Alert): Promise<void> {
    // Log only
    console.log(`[LOW ALERT] ${alert.title}:`, alert.metadata);
  }

  /**
   * Check if alert is in cooldown
   */
  private isInCooldown(alertKey: string, severity: AlertSeverity): boolean {
    const lastAlert = this.lastAlertTime.get(alertKey);
    if (!lastAlert) return false;

    const cooldownMinutes = this.getCooldownPeriod(severity);
    const cooldownMs = cooldownMinutes * 60 * 1000;
    const timeSinceLastAlert = Date.now() - lastAlert.getTime();

    return timeSinceLastAlert < cooldownMs;
  }

  /**
   * Get cooldown period based on severity
   */
  private getCooldownPeriod(severity: AlertSeverity): number {
    switch (severity) {
      case AlertSeverity.CRITICAL:
        return 5;   // 5 minutes
      case AlertSeverity.HIGH:
        return 15;  // 15 minutes
      case AlertSeverity.MEDIUM:
        return 60;  // 1 hour
      case AlertSeverity.LOW:
        return 240; // 4 hours
    }
  }

  /**
   * Check if current time is business hours
   */
  private isBusinessHours(): boolean {
    const now = new Date();
    const hour = now.getHours();
    const day = now.getDay();

    // Monday-Friday, 9 AM - 6 PM
    return day >= 1 && day <= 5 && hour >= 9 && hour < 18;
  }

  /**
   * Create incident ticket (integration with JIRA/ServiceNow)
   */
  private async createIncidentTicket(alert: Alert): Promise<void> {
    // Implement integration with ticketing system
    console.log(`Creating incident ticket for: ${alert.title}`);
  }

  /**
   * Load alert rules from configuration
   */
  private loadAlertRules(): void {
    this.alertRules = [
      {
        id: 'high-error-rate',
        name: 'High Error Rate',
        condition: (metrics) => metrics.errorRate > 5,
        severity: AlertSeverity.CRITICAL,
        cooldown: 5,
        threshold: {
          value: 5,
          duration: 5,
        },
      },
      {
        id: 'slow-response',
        name: 'Slow Response Time',
        condition: (metrics) => metrics.p95ResponseTime > 3000,
        severity: AlertSeverity.HIGH,
        cooldown: 15,
        threshold: {
          value: 3000,
          duration: 10,
        },
      },
      // Add more rules...
    ];
  }

  /**
   * Get active alerts
   */
  getActiveAlerts(): Alert[] {
    return Array.from(this.activeAlerts.values());
  }
}
```

### 2. PagerDuty Client

```typescript
// src/monitoring/alerts/PagerDutyClient.ts

import axios from 'axios';

export interface PagerDutyIncident {
  title: string;
  description: string;
  severity: 'critical' | 'error' | 'warning' | 'info';
  service: string;
  urgency: 'high' | 'low';
  metadata?: Record<string, any>;
}

export class PagerDutyClient {
  private apiKey: string;
  private baseUrl = 'https://api.pagerduty.com';

  constructor(apiKey: string) {
    this.apiKey = apiKey;
  }

  /**
   * Create PagerDuty incident
   */
  async createIncident(incident: PagerDutyIncident): Promise<string> {
    try {
      const response = await axios.post(
        `${this.baseUrl}/incidents`,
        {
          incident: {
            type: 'incident',
            title: incident.title,
            body: {
              type: 'incident_body',
              details: incident.description,
            },
            urgency: incident.urgency,
            incident_key: `${incident.service}-${Date.now()}`,
          },
        },
        {
          headers: {
            'Authorization': `Token token=${this.apiKey}`,
            'Content-Type': 'application/json',
            'Accept': 'application/vnd.pagerduty+json;version=2',
          },
        }
      );

      const incidentId = response.data.incident.id;
      console.log(`PagerDuty incident created: ${incidentId}`);
      return incidentId;
    } catch (error) {
      console.error('Failed to create PagerDuty incident:', error);
      throw error;
    }
  }

  /**
   * Resolve PagerDuty incident
   */
  async resolveIncident(incidentId: string, resolution: string): Promise<void> {
    try {
      await axios.put(
        `${this.baseUrl}/incidents/${incidentId}`,
        {
          incident: {
            type: 'incident',
            status: 'resolved',
            resolution: resolution,
          },
        },
        {
          headers: {
            'Authorization': `Token token=${this.apiKey}`,
            'Content-Type': 'application/json',
            'Accept': 'application/vnd.pagerduty+json;version=2',
          },
        }
      );

      console.log(`PagerDuty incident resolved: ${incidentId}`);
    } catch (error) {
      console.error('Failed to resolve PagerDuty incident:', error);
      throw error;
    }
  }

  /**
   * Trigger event (simpler alternative to incidents)
   */
  async triggerEvent(
    routingKey: string,
    payload: {
      summary: string;
      severity: 'critical' | 'error' | 'warning' | 'info';
      source: string;
      customDetails?: Record<string, any>;
    }
  ): Promise<void> {
    try {
      await axios.post('https://events.pagerduty.com/v2/enqueue', {
        routing_key: routingKey,
        event_action: 'trigger',
        payload: {
          summary: payload.summary,
          severity: payload.severity,
          source: payload.source,
          custom_details: payload.customDetails,
        },
      });

      console.log('PagerDuty event triggered');
    } catch (error) {
      console.error('Failed to trigger PagerDuty event:', error);
      throw error;
    }
  }
}
```

### 3. SLO Tracker

```typescript
// src/monitoring/slo/SLOTracker.ts

export interface SLO {
  id: string;
  name: string;
  target: number;        // Target percentage (e.g., 99.9)
  window: number;        // Time window in days (e.g., 30)
  type: 'availability' | 'latency' | 'quality';
  threshold?: number;    // For latency SLOs (e.g., 3000ms)
}

export interface SLOStatus {
  slo: SLO;
  current: number;       // Current achievement
  errorBudget: number;   // Remaining error budget
  errorBudgetSpent: number; // Spent error budget %
  isHealthy: boolean;
  projectedAchievement: number; // Projected for end of window
}

export class SLOTracker {
  private slos: Map<string, SLO> = new Map();
  private measurements: Map<string, number[]> = new Map();

  constructor() {
    this.initializeSLOs();
  }

  /**
   * Initialize SLOs for banking API
   */
  private initializeSLOs(): void {
    const slos: SLO[] = [
      {
        id: 'transaction-success-rate',
        name: 'Transaction Success Rate',
        target: 99.9,
        window: 30,
        type: 'availability',
      },
      {
        id: 'api-response-time',
        name: 'API Response Time P95',
        target: 95,
        window: 30,
        type: 'latency',
        threshold: 3000,  // 95% of requests < 3000ms
      },
      {
        id: 'payment-processing-success',
        name: 'Payment Processing Success',
        target: 99.95,
        window: 30,
        type: 'availability',
      },
      {
        id: 'account-query-latency',
        name: 'Account Query Latency P99',
        target: 99,
        window: 30,
        type: 'latency',
        threshold: 500,   // 99% of queries < 500ms
      },
    ];

    slos.forEach(slo => this.slos.set(slo.id, slo));
  }

  /**
   * Record SLO measurement
   */
  recordMeasurement(sloId: string, success: boolean): void {
    if (!this.measurements.has(sloId)) {
      this.measurements.set(sloId, []);
    }

    const measurements = this.measurements.get(sloId)!;
    measurements.push(success ? 1 : 0);

    // Keep only last window worth of data
    const maxMeasurements = 100000; // Adjust based on request rate
    if (measurements.length > maxMeasurements) {
      measurements.shift();
    }
  }

  /**
   * Get SLO status
   */
  getSLOStatus(sloId: string): SLOStatus | null {
    const slo = this.slos.get(sloId);
    if (!slo) return null;

    const measurements = this.measurements.get(sloId) || [];
    if (measurements.length === 0) {
      return {
        slo,
        current: 100,
        errorBudget: 100,
        errorBudgetSpent: 0,
        isHealthy: true,
        projectedAchievement: 100,
      };
    }

    // Calculate current achievement
    const successCount = measurements.filter(m => m === 1).length;
    const current = (successCount / measurements.length) * 100;

    // Calculate error budget
    const allowedFailures = ((100 - slo.target) / 100) * measurements.length;
    const actualFailures = measurements.length - successCount;
    const errorBudgetRemaining = Math.max(0, allowedFailures - actualFailures);
    const errorBudget = (errorBudgetRemaining / allowedFailures) * 100;
    const errorBudgetSpent = 100 - errorBudget;

    // Determine health
    const isHealthy = current >= slo.target && errorBudget > 10;

    // Project achievement (simple linear projection)
    const projectedAchievement = current;

    return {
      slo,
      current,
      errorBudget,
      errorBudgetSpent,
      isHealthy,
      projectedAchievement,
    };
  }

  /**
   * Get all SLO statuses
   */
  getAllSLOStatuses(): SLOStatus[] {
    return Array.from(this.slos.keys())
      .map(id => this.getSLOStatus(id))
      .filter((status): status is SLOStatus => status !== null);
  }

  /**
   * Check if error budget allows new feature deployment
   */
  canDeploy(sloId: string, minimumBudget: number = 20): boolean {
    const status = this.getSLOStatus(sloId);
    return status ? status.errorBudget >= minimumBudget : false;
  }

  /**
   * Calculate burn rate (how fast error budget is depleting)
   */
  calculateBurnRate(sloId: string, windowHours: number = 1): number {
    const measurements = this.measurements.get(sloId) || [];
    const slo = this.slos.get(sloId);

    if (!slo || measurements.length === 0) return 0;

    // Get recent measurements (last hour)
    const recentMeasurements = measurements.slice(-1000); // Adjust based on rate
    const failures = recentMeasurements.filter(m => m === 0).length;
    const failureRate = failures / recentMeasurements.length;

    // Expected failure rate for SLO
    const expectedFailureRate = (100 - slo.target) / 100;

    // Burn rate = actual / expected
    return failureRate / expectedFailureRate;
  }
}
```

### 4. Integrated Monitoring Service

```typescript
// src/monitoring/MonitoringService.ts

import { AlertManager, Alert, AlertSeverity } from './alerts/AlertManager';
import { SLOTracker } from './slo/SLOTracker';

export class MonitoringService {
  private alertManager: AlertManager;
  private sloTracker: SLOTracker;
  private metrics: Map<string, number[]> = new Map();

  constructor(
    pagerDutyApiKey: string,
    slackWebhookUrl: string
  ) {
    this.alertManager = new AlertManager(pagerDutyApiKey, slackWebhookUrl);
    this.sloTracker = new SLOTracker();
    this.startPeriodicChecks();
  }

  /**
   * Record transaction result
   */
  recordTransaction(type: string, success: boolean, duration: number): void {
    // Record for SLO tracking
    this.sloTracker.recordMeasurement('transaction-success-rate', success);

    // Record latency SLO
    const latencySLO = duration < 3000;
    this.sloTracker.recordMeasurement('api-response-time', latencySLO);

    // Check for alerts
    if (!success) {
      this.checkErrorRateAlert();
    }

    if (duration > 5000) {
      this.checkLatencyAlert(duration);
    }
  }

  /**
   * Check error rate and alert if threshold exceeded
   */
  private async checkErrorRateAlert(): Promise<void> {
    const status = this.sloTracker.getSLOStatus('transaction-success-rate');

    if (status && status.current < status.slo.target) {
      await this.alertManager.trigger({
        id: `error-rate-${Date.now()}`,
        severity: AlertSeverity.CRITICAL,
        title: 'High Transaction Failure Rate',
        description: `Transaction success rate is ${status.current.toFixed(2)}%, below target of ${status.slo.target}%`,
        service: 'banking-api',
        timestamp: new Date(),
        metadata: {
          current: status.current,
          target: status.slo.target,
          errorBudgetRemaining: status.errorBudget,
        },
        runbookUrl: 'https://runbooks.example.com/high-error-rate',
        affectedCustomers: 1000,
      });
    }
  }

  /**
   * Check latency and alert if threshold exceeded
   */
  private async checkLatencyAlert(duration: number): Promise<void> {
    await this.alertManager.trigger({
      id: `latency-${Date.now()}`,
      severity: AlertSeverity.HIGH,
      title: 'High API Latency Detected',
      description: `API response time is ${duration}ms, significantly above threshold`,
      service: 'banking-api',
      timestamp: new Date(),
      metadata: {
        duration,
        threshold: 3000,
      },
      runbookUrl: 'https://runbooks.example.com/high-latency',
    });
  }

  /**
   * Start periodic health checks
   */
  private startPeriodicChecks(): void {
    setInterval(() => {
      this.checkSLOHealth();
      this.checkErrorBudget();
    }, 60000); // Every minute
  }

  /**
   * Check overall SLO health
   */
  private async checkSLOHealth(): Promise<void> {
    const statuses = this.sloTracker.getAllSLOStatuses();

    for (const status of statuses) {
      if (!status.isHealthy) {
        await this.alertManager.trigger({
          id: `slo-breach-${status.slo.id}-${Date.now()}`,
          severity: AlertSeverity.HIGH,
          title: `SLO Breach: ${status.slo.name}`,
          description: `${status.slo.name} is at ${status.current.toFixed(2)}%, below target of ${status.slo.target}%`,
          service: 'banking-api',
          timestamp: new Date(),
          metadata: {
            sloId: status.slo.id,
            current: status.current,
            target: status.slo.target,
            errorBudgetRemaining: status.errorBudget,
          },
        });
      }
    }
  }

  /**
   * Check error budget and warn if depleting fast
   */
  private async checkErrorBudget(): Promise<void> {
    const statuses = this.sloTracker.getAllSLOStatuses();

    for (const status of statuses) {
      const burnRate = this.sloTracker.calculateBurnRate(status.slo.id);

      // Alert if burning budget > 2x expected rate
      if (burnRate > 2.0) {
        await this.alertManager.trigger({
          id: `error-budget-burn-${status.slo.id}-${Date.now()}`,
          severity: AlertSeverity.MEDIUM,
          title: `High Error Budget Burn Rate: ${status.slo.name}`,
          description: `Error budget is burning at ${burnRate.toFixed(2)}x expected rate`,
          service: 'banking-api',
          timestamp: new Date(),
          metadata: {
            sloId: status.slo.id,
            burnRate,
            errorBudgetRemaining: status.errorBudget,
          },
        });
      }
    }
  }

  /**
   * Get monitoring dashboard data
   */
  getDashboardData(): any {
    return {
      slos: this.sloTracker.getAllSLOStatuses(),
      activeAlerts: this.alertManager.getActiveAlerts(),
      timestamp: new Date(),
    };
  }
}
```

### 5. Usage in Application

```typescript
// src/index.ts

import express from 'express';
import { MonitoringService } from './monitoring/MonitoringService';

const app = express();
const PORT = process.env.PORT || 3000;

// Initialize monitoring
const monitoring = new MonitoringService(
  process.env.PAGERDUTY_API_KEY!,
  process.env.SLACK_WEBHOOK_URL!
);

app.use(express.json());

// Monitored endpoint
app.post('/api/transfer', async (req, res) => {
  const startTime = Date.now();
  let success = false;

  try {
    // Process transfer
    await processTransfer(req.body);
    success = true;
    res.json({ success: true });
  } catch (error) {
    res.status(500).json({ error: (error as Error).message });
  } finally {
    const duration = Date.now() - startTime;
    monitoring.recordTransaction('TRANSFER', success, duration);
  }
});

// Dashboard endpoint
app.get('/monitoring/dashboard', (req, res) => {
  res.json(monitoring.getDashboardData());
});

app.listen(PORT, () => {
  console.log(`🏦 Banking API running on port ${PORT}`);
  console.log('📊 Monitoring and alerting enabled');
});

async function processTransfer(data: any): Promise<void> {
  // Transfer logic
  await new Promise(resolve => setTimeout(resolve, 100));
}
```

---

## 🎯 Best Practices

### 1. Alert on Symptoms, Not Causes

```typescript
// ✅ GOOD: Alert on user-facing issue
{
  alert: "High API Error Rate",
  condition: "error_rate > 5% for 5 minutes",
  impact: "Customers cannot complete transactions"
}

// ❌ BAD: Alert on internal metric without context
{
  alert: "High CPU Usage",
  condition: "cpu > 80%",
  impact: "Unknown - may or may not affect users"
}
```

### 2. Include Runbooks in Alerts

```markdown
# Runbook: High Error Rate

## Symptom
Transaction error rate > 5% for 5+ minutes

## Impact
Customers unable to complete transfers

## Diagnosis
1. Check error logs: `kubectl logs -l app=banking-api --tail=100`
2. Check database connectivity: `curl https://api.banking.com/health/db`
3. Check external services: Check payment gateway status page

## Resolution
### If database issue:
- Restart connection pool: `kubectl rollout restart deployment/banking-api`
- Check RDS metrics in AWS console

### If payment gateway issue:
- Switch to backup gateway: `kubectl set env deployment/banking-api GATEWAY=backup`
- Contact gateway support

## Escalation
If not resolved in 15 minutes, escalate to @platform-team
```

### 3. Implement Alert Deduplication

```typescript
// Deduplicate similar alerts
class AlertDeduplicator {
  private recentAlerts: Map<string, Date> = new Map();

  shouldAlert(alertKey: string, cooldownMinutes: number): boolean {
    const lastAlert = this.recentAlerts.get(alertKey);
    
    if (!lastAlert) {
      this.recentAlerts.set(alertKey, new Date());
      return true;
    }

    const minutesSinceLastAlert = 
      (Date.now() - lastAlert.getTime()) / 1000 / 60;

    if (minutesSinceLastAlert >= cooldownMinutes) {
      this.recentAlerts.set(alertKey, new Date());
      return true;
    }

    return false;
  }
}
```

### 4. Use Error Budgets for Decision Making

```typescript
// Check error budget before deployment
async function canDeploy(service: string): Promise<boolean> {
  const errorBudget = await getErrorBudget(service);
  
  if (errorBudget < 10) {
    console.log('❌ Cannot deploy: Error budget too low');
    console.log('Focus on reliability before new features');
    return false;
  }
  
  console.log('✅ Error budget healthy, deployment approved');
  return true;
}
```

---

## 📚 Common Interview Questions

### Q1: How do you prevent alert fatigue?

**Answer**:
1. **High precision**: Only alert on actionable issues
2. **Proper severity**: Don't page for non-urgent issues
3. **Aggregation**: Group related alerts
4. **Cooldown periods**: Avoid repeated alerts
5. **Self-healing**: Auto-remediate when possible

### Q2: What's the difference between SLI, SLO, and SLA?

**Answer**:
- **SLI**: Measurement (what you measure) - "99.5% uptime"
- **SLO**: Target (what you aim for) - "≥99.9% uptime"
- **SLA**: Contract (what you promise) - "99.95% uptime or refund"

### Q3: How do you handle alert storms?

**Answer**:
1. **Correlation**: Group related alerts
2. **Root cause detection**: Identify primary failure
3. **Suppression**: Pause dependent alerts
4. **Escalation**: Page only once for storm
5. **Post-mortem**: Fix alerting rules after resolution

### Q4: What's a good error budget policy?

**Answer**:
```yaml
Error Budget Policy:
  - Budget > 50%: Deploy freely
  - Budget 25-50%: Deploy with caution
  - Budget 10-25%: Freeze non-critical deploys
  - Budget < 10%: Feature freeze, focus on reliability
```

### Q5: How do you measure alert quality?

**Answer**:
- **Precision**: True positives / (True positives + False positives)
- **Recall**: True positives / (True positives + False negatives)
- **MTTA**: Mean Time To Acknowledge
- **MTTR**: Mean Time To Resolve
- Target: Precision > 90%, Recall > 95%

---

## ✅ Summary & Key Takeaways

### Alerting Strategy Checklist

```yaml
✅ Alert Design:
  - [ ] Alert on symptoms, not causes
  - [ ] Include runbooks in alerts
  - [ ] Set appropriate severity levels
  - [ ] Add cooldown periods
  - [ ] Include affected customer count

✅ SLO Management:
  - [ ] Define clear SLIs
  - [ ] Set realistic SLO targets
  - [ ] Track error budgets
  - [ ] Review SLOs quarterly
  - [ ] Use error budgets for decisions

✅ Incident Management:
  - [ ] Integrate with PagerDuty
  - [ ] Route by severity
  - [ ] Auto-create tickets
  - [ ] Track MTTA and MTTR
  - [ ] Conduct post-mortems

✅ Alert Quality:
  - [ ] Monitor precision/recall
  - [ ] Regular alert tuning
  - [ ] Remove noisy alerts
  - [ ] Aggregate related alerts
  - [ ] Self-healing when possible
```

### Key Metrics

```
┌────────────────────────────────────────────────────┐
│         Banking API SLO Dashboard                  │
├────────────────────────────────────────────────────┤
│                                                     │
│ Transaction Success Rate:                          │
│  Current: 99.92% ✅ (Target: 99.9%)                │
│  Error Budget: 75% remaining                       │
│  Burn Rate: 0.8x (healthy)                         │
│                                                     │
│ API Response Time (P95):                           │
│  Current: 2,450ms ✅ (Target: <3,000ms)            │
│  Error Budget: 85% remaining                       │
│  Burn Rate: 0.5x (healthy)                         │
│                                                     │
│ Payment Processing Success:                        │
│  Current: 99.97% ✅ (Target: 99.95%)               │
│  Error Budget: 90% remaining                       │
│  Burn Rate: 0.3x (healthy)                         │
│                                                     │
└────────────────────────────────────────────────────┘
```

---

**Status**: ✅ Complete with production-ready alerting and SLO management for banking applications!
