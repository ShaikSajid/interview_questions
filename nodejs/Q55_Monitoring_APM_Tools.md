# Q55: Monitoring - APM Tools (New Relic, DataDog)

## 📋 Summary
This question covers **Application Performance Monitoring (APM)** with industry-leading tools like New Relic, DataDog, and Prometheus. Learn to instrument Node.js banking applications for comprehensive observability - tracking performance, errors, custom business metrics, and distributed traces.

**Key Topics**:
- APM fundamentals and concepts
- New Relic integration and custom instrumentation
- DataDog APM and tracing
- Prometheus metrics and Grafana dashboards
- Custom metrics for banking applications
- Distributed tracing across microservices
- Error tracking and alerting
- Performance profiling
- Real User Monitoring (RUM)
- Log aggregation and analysis

**Banking Use Cases**:
- Transaction success/failure rates
- Payment processing latency
- Account balance query performance
- Authentication response times
- Database query optimization
- API endpoint monitoring
- Fraud detection metrics
- Compliance audit trails

---

## 🎯 Understanding APM (Application Performance Monitoring)

### What is APM?

**APM** provides real-time insights into application performance, user experience, and business metrics through automated instrumentation and custom tracking.

```
┌─────────────────────────────────────────────────────────────┐
│              APM Architecture Overview                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Application Layer                                          │
│  ┌────────────────────────────────────────────┐            │
│  │  Node.js Banking API                       │            │
│  │  - Express routes                          │            │
│  │  - Business logic                          │            │
│  │  - Database queries                        │            │
│  └───────────┬────────────────────────────────┘            │
│              │ APM Agent                                     │
│              │ (New Relic/DataDog)                          │
│              ▼                                               │
│  ┌────────────────────────────────────────────┐            │
│  │  Data Collection                           │            │
│  │  - Transactions                            │            │
│  │  - Errors                                  │            │
│  │  - Custom metrics                          │            │
│  │  - Distributed traces                      │            │
│  │  - Logs                                    │            │
│  └───────────┬────────────────────────────────┘            │
│              │ Agent sends data                             │
│              ▼                                               │
│  ┌────────────────────────────────────────────┐            │
│  │  APM Backend (Cloud)                       │            │
│  │  - Data aggregation                        │            │
│  │  - Anomaly detection                       │            │
│  │  - Alerting engine                         │            │
│  │  - Storage & indexing                      │            │
│  └───────────┬────────────────────────────────┘            │
│              │                                               │
│              ▼                                               │
│  ┌────────────────────────────────────────────┐            │
│  │  Visualization & Analysis                  │            │
│  │  - Dashboards                              │            │
│  │  - Alerts                                  │            │
│  │  - Reports                                 │            │
│  │  - Service maps                            │            │
│  └────────────────────────────────────────────┘            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Key APM Metrics

| Metric | Description | Banking Example |
|--------|-------------|-----------------|
| **Throughput** | Requests per minute | Transactions/min |
| **Response Time** | Average request duration | Payment processing time |
| **Error Rate** | Failed requests percentage | Failed transfers % |
| **Apdex Score** | User satisfaction index | Customer satisfaction |
| **Database Time** | DB query duration | Account lookup time |
| **External Calls** | Third-party API time | Payment gateway latency |
| **Memory Usage** | Heap size and GC | Application stability |
| **CPU Usage** | Processor utilization | Server load |

### APM Tool Comparison

| Feature | New Relic | DataDog | Prometheus + Grafana |
|---------|-----------|---------|---------------------|
| **Auto-instrumentation** | Excellent | Excellent | Manual |
| **Distributed Tracing** | Yes | Yes | Limited |
| **Custom Metrics** | Yes | Yes | Yes (native) |
| **Log Management** | Yes | Yes | No (use Loki) |
| **Alerting** | Advanced | Advanced | Basic (Alertmanager) |
| **Cost** | High | High | Free (self-hosted) |
| **Learning Curve** | Low | Low | Medium |

---

## 💡 Example 1: New Relic APM Integration

Complete New Relic integration with custom banking metrics.

### Installation

```bash
npm install newrelic
npm install @newrelic/native-metrics
```

### Project Structure

```
banking-api/
├── newrelic.js              # New Relic config
├── src/
│   ├── monitoring/
│   │   ├── newrelic.ts      # Custom instrumentation
│   │   └── metrics.ts       # Business metrics
│   ├── middleware/
│   │   └── monitoring.ts    # Request tracking
│   ├── services/
│   │   ├── AccountService.ts
│   │   └── TransactionService.ts
│   └── index.ts
├── package.json
└── .env
```

### 1. New Relic Configuration

```javascript
// newrelic.js (must be at project root)

'use strict';

/**
 * New Relic agent configuration
 * 
 * See node_modules/newrelic/lib/config/default.js for complete list of options
 */
exports.config = {
  /**
   * Application name (displayed in New Relic UI)
   */
  app_name: ['Banking API Production'],

  /**
   * License key from New Relic account
   */
  license_key: process.env.NEW_RELIC_LICENSE_KEY,

  /**
   * Logging configuration
   */
  logging: {
    level: 'info',
    filepath: 'stdout',
  },

  /**
   * Allow all data to be sent to New Relic
   */
  allow_all_headers: true,

  /**
   * Custom attributes for all transactions
   */
  attributes: {
    exclude: [
      'request.headers.cookie',
      'request.headers.authorization',
      'request.headers.proxyAuthorization',
      'request.headers.setCookie*',
      'request.headers.x*',
      'response.headers.cookie',
      'response.headers.authorization',
      'response.headers.proxyAuthorization',
      'response.headers.setCookie*',
      'response.headers.x*',
    ],
  },

  /**
   * Transaction tracer configuration
   */
  transaction_tracer: {
    enabled: true,
    transaction_threshold: 'apdex_f',  // Trace slow transactions
    record_sql: 'obfuscated',          // Obfuscate SQL queries
    explain_threshold: 500,             // Explain queries > 500ms
  },

  /**
   * Error collector configuration
   */
  error_collector: {
    enabled: true,
    ignore_status_codes: [404],        // Don't report 404s
    capture_events: true,
    max_event_samples_stored: 100,
  },

  /**
   * Distributed tracing
   */
  distributed_tracing: {
    enabled: true,
  },

  /**
   * Slow SQL statements
   */
  slow_sql: {
    enabled: true,
  },

  /**
   * Custom instrumentation
   */
  custom_insights_events: {
    enabled: true,
    max_samples_stored: 10000,
  },

  /**
   * Application logging (forward logs to New Relic)
   */
  application_logging: {
    enabled: true,
    forwarding: {
      enabled: true,
      max_samples_stored: 10000,
    },
    metrics: {
      enabled: true,
    },
    local_decorating: {
      enabled: true,
    },
  },
};
```

### 2. Custom Instrumentation

```typescript
// src/monitoring/newrelic.ts

import newrelic from 'newrelic';

/**
 * Banking-specific custom metrics
 */
export class NewRelicMonitoring {
  /**
   * Record custom metric
   */
  static recordMetric(name: string, value: number): void {
    newrelic.recordMetric(name, value);
  }

  /**
   * Record transaction success
   */
  static recordTransaction(
    type: string,
    amount: number,
    duration: number,
    success: boolean
  ): void {
    // Custom event
    newrelic.recordCustomEvent('BankingTransaction', {
      transactionType: type,
      amount: amount,
      duration: duration,
      success: success,
      timestamp: Date.now(),
    });

    // Custom metrics
    this.recordMetric(`Custom/Banking/Transaction/${type}/Count`, 1);
    this.recordMetric(`Custom/Banking/Transaction/${type}/Amount`, amount);
    this.recordMetric(`Custom/Banking/Transaction/${type}/Duration`, duration);

    if (!success) {
      this.recordMetric(`Custom/Banking/Transaction/${type}/Errors`, 1);
    }
  }

  /**
   * Record account operation
   */
  static recordAccountOperation(
    operation: string,
    duration: number,
    success: boolean
  ): void {
    newrelic.recordCustomEvent('AccountOperation', {
      operation,
      duration,
      success,
      timestamp: Date.now(),
    });

    this.recordMetric(`Custom/Banking/Account/${operation}/Duration`, duration);

    if (!success) {
      this.recordMetric(`Custom/Banking/Account/${operation}/Errors`, 1);
    }
  }

  /**
   * Record database query performance
   */
  static recordDatabaseQuery(
    query: string,
    duration: number,
    rowCount: number
  ): void {
    newrelic.recordCustomEvent('DatabaseQuery', {
      query: query.substring(0, 100),  // Truncate for privacy
      duration,
      rowCount,
      timestamp: Date.now(),
    });

    this.recordMetric('Custom/Database/Query/Duration', duration);
    this.recordMetric('Custom/Database/Query/RowCount', rowCount);

    // Alert on slow queries
    if (duration > 1000) {
      newrelic.noticeError(new Error(`Slow query detected: ${duration}ms`), {
        query: query.substring(0, 100),
        duration,
      });
    }
  }

  /**
   * Record business metric
   */
  static recordBusinessMetric(
    metricName: string,
    value: number,
    attributes?: Record<string, any>
  ): void {
    newrelic.recordCustomEvent('BusinessMetric', {
      metricName,
      value,
      ...attributes,
      timestamp: Date.now(),
    });

    this.recordMetric(`Custom/Business/${metricName}`, value);
  }

  /**
   * Add custom attributes to current transaction
   */
  static addCustomAttributes(attributes: Record<string, any>): void {
    Object.entries(attributes).forEach(([key, value]) => {
      newrelic.addCustomAttribute(key, value);
    });
  }

  /**
   * Start background transaction (for async jobs)
   */
  static startBackgroundTransaction(
    name: string,
    group: string,
    callback: () => Promise<void>
  ): Promise<void> {
    return new Promise((resolve, reject) => {
      newrelic.startBackgroundTransaction(name, group, async () => {
        try {
          await callback();
          resolve();
        } catch (error) {
          newrelic.noticeError(error as Error);
          reject(error);
        } finally {
          newrelic.endTransaction();
        }
      });
    });
  }

  /**
   * Set transaction name (useful for dynamic routes)
   */
  static setTransactionName(name: string): void {
    newrelic.setTransactionName(name);
  }

  /**
   * Report error with context
   */
  static reportError(
    error: Error,
    attributes?: Record<string, any>
  ): void {
    newrelic.noticeError(error, attributes);
  }
}
```

### 3. Monitoring Middleware

```typescript
// src/middleware/monitoring.ts

import { Request, Response, NextFunction } from 'express';
import newrelic from 'newrelic';
import { NewRelicMonitoring } from '../monitoring/newrelic';

/**
 * Request tracking middleware
 */
export function requestMonitoring(
  req: Request,
  res: Response,
  next: NextFunction
): void {
  const startTime = Date.now();

  // Add custom attributes to transaction
  NewRelicMonitoring.addCustomAttributes({
    userId: req.headers['x-user-id'] || 'anonymous',
    requestId: req.headers['x-request-id'],
    userAgent: req.headers['user-agent'],
    clientIp: req.ip,
  });

  // Set meaningful transaction name
  const transactionName = `${req.method} ${req.route?.path || req.path}`;
  NewRelicMonitoring.setTransactionName(transactionName);

  // Capture response
  const originalSend = res.send;
  res.send = function (data: any): Response {
    const duration = Date.now() - startTime;

    // Record request metrics
    NewRelicMonitoring.recordMetric(
      `Custom/API/Request/${req.method}/${req.route?.path || 'unknown'}/Duration`,
      duration
    );

    NewRelicMonitoring.recordCustomEvent('ApiRequest', {
      method: req.method,
      path: req.path,
      statusCode: res.statusCode,
      duration,
      success: res.statusCode < 400,
    });

    return originalSend.call(this, data);
  };

  next();
}

/**
 * Error tracking middleware
 */
export function errorMonitoring(
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
): void {
  // Report error to New Relic with context
  NewRelicMonitoring.reportError(err, {
    method: req.method,
    path: req.path,
    userId: req.headers['x-user-id'],
    requestId: req.headers['x-request-id'],
    body: JSON.stringify(req.body).substring(0, 1000),  // Truncate
  });

  // Record error metric
  NewRelicMonitoring.recordMetric(
    `Custom/API/Errors/${req.method}/${req.route?.path || 'unknown'}`,
    1
  );

  next(err);
}
```

### 4. Instrumented Service

```typescript
// src/services/TransactionService.ts

import { Pool } from 'pg';
import { NewRelicMonitoring } from '../monitoring/newrelic';

export class TransactionService {
  constructor(private pool: Pool) {}

  /**
   * Transfer funds with monitoring
   */
  async transfer(
    fromAccountId: string,
    toAccountId: string,
    amount: number
  ): Promise<any> {
    const startTime = Date.now();
    let success = false;

    try {
      const client = await this.pool.connect();

      try {
        await client.query('BEGIN');

        // Record custom attributes
        NewRelicMonitoring.addCustomAttributes({
          fromAccountId,
          toAccountId,
          amount,
          operation: 'transfer',
        });

        // Debit from account
        const debitQuery = `
          UPDATE accounts 
          SET balance = balance - $1 
          WHERE id = $2 
          RETURNING balance
        `;
        const debitStart = Date.now();
        const debitResult = await client.query(debitQuery, [amount, fromAccountId]);
        NewRelicMonitoring.recordDatabaseQuery(
          'UPDATE accounts (debit)',
          Date.now() - debitStart,
          debitResult.rowCount
        );

        if (debitResult.rows[0].balance < 0) {
          throw new Error('Insufficient funds');
        }

        // Credit to account
        const creditQuery = `
          UPDATE accounts 
          SET balance = balance + $1 
          WHERE id = $2
        `;
        const creditStart = Date.now();
        const creditResult = await client.query(creditQuery, [amount, toAccountId]);
        NewRelicMonitoring.recordDatabaseQuery(
          'UPDATE accounts (credit)',
          Date.now() - creditStart,
          creditResult.rowCount
        );

        await client.query('COMMIT');
        success = true;

        // Record business metrics
        NewRelicMonitoring.recordBusinessMetric('TransferAmount', amount);
        NewRelicMonitoring.recordBusinessMetric('TransferCount', 1);

        return { success: true, transactionId: 'TXN123' };
      } catch (error) {
        await client.query('ROLLBACK');
        throw error;
      } finally {
        client.release();
      }
    } catch (error) {
      NewRelicMonitoring.reportError(error as Error, {
        fromAccountId,
        toAccountId,
        amount,
      });
      throw error;
    } finally {
      const duration = Date.now() - startTime;

      // Record transaction metrics
      NewRelicMonitoring.recordTransaction('TRANSFER', amount, duration, success);

      // Alert on slow transfers
      if (duration > 3000) {
        NewRelicMonitoring.reportError(
          new Error(`Slow transfer detected: ${duration}ms`),
          { fromAccountId, toAccountId, amount, duration }
        );
      }
    }
  }
}
```

### 5. Application Entry Point

```typescript
// src/index.ts

// ⚠️ IMPORTANT: New Relic must be required FIRST
import newrelic from 'newrelic';

import express from 'express';
import { Pool } from 'pg';
import { requestMonitoring, errorMonitoring } from './middleware/monitoring';
import { TransactionService } from './services/TransactionService';

const app = express();
const PORT = process.env.PORT || 3000;

// Middleware
app.use(express.json());
app.use(requestMonitoring);

// Database
const pool = new Pool({
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT || '5432'),
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
});

// Services
const transactionService = new TransactionService(pool);

// Routes
app.post('/api/transfer', async (req, res, next) => {
  try {
    const result = await transactionService.transfer(
      req.body.fromAccountId,
      req.body.toAccountId,
      req.body.amount
    );
    res.json(result);
  } catch (error) {
    next(error);
  }
});

// Error handling
app.use(errorMonitoring);
app.use((err: Error, req: any, res: any, next: any) => {
  res.status(500).json({ error: err.message });
});

app.listen(PORT, () => {
  console.log(`🏦 Banking API running on port ${PORT}`);
  console.log('📊 New Relic monitoring enabled');
});
```

### package.json

```json
{
  "name": "banking-api-monitored",
  "version": "1.0.0",
  "scripts": {
    "start": "node -r newrelic dist/index.js",
    "dev": "ts-node-dev -r newrelic src/index.ts"
  },
  "dependencies": {
    "express": "^4.18.2",
    "newrelic": "^11.0.0",
    "@newrelic/native-metrics": "^9.0.0",
    "pg": "^8.11.0"
  }
}
```

---

## 💡 Example 2: DataDog APM Integration

Complete DataDog integration with distributed tracing.

### Installation

```bash
npm install dd-trace
```

### 1. DataDog Configuration

```typescript
// src/monitoring/datadog.ts

import tracer from 'dd-trace';

/**
 * Initialize DataDog tracer
 */
export function initializeDataDog(): void {
  tracer.init({
    service: 'banking-api',
    env: process.env.NODE_ENV || 'development',
    version: process.env.APP_VERSION || '1.0.0',
    
    // Logging
    logInjection: true,
    debug: process.env.DD_DEBUG === 'true',
    
    // Profiling
    profiling: true,
    runtimeMetrics: true,
    
    // Sampling (100% in dev, 50% in prod)
    sampleRate: process.env.NODE_ENV === 'production' ? 0.5 : 1.0,
    
    // Tags
    tags: {
      region: process.env.AWS_REGION || 'us-east-1',
      cluster: process.env.CLUSTER_NAME || 'default',
    },
  });

  console.log('📊 DataDog APM initialized');
}

/**
 * Custom DataDog metrics
 */
export class DataDogMetrics {
  private static client = tracer;

  /**
   * Increment counter
   */
  static increment(metric: string, tags?: Record<string, string>): void {
    this.client.dogstatsd.increment(metric, 1, tags);
  }

  /**
   * Record gauge value
   */
  static gauge(metric: string, value: number, tags?: Record<string, string>): void {
    this.client.dogstatsd.gauge(metric, value, tags);
  }

  /**
   * Record histogram (timing)
   */
  static histogram(metric: string, value: number, tags?: Record<string, string>): void {
    this.client.dogstatsd.histogram(metric, value, tags);
  }

  /**
   * Record distribution
   */
  static distribution(metric: string, value: number, tags?: Record<string, string>): void {
    this.client.dogstatsd.distribution(metric, value, tags);
  }

  /**
   * Record transaction
   */
  static recordTransaction(
    type: string,
    amount: number,
    duration: number,
    success: boolean
  ): void {
    const tags = {
      transaction_type: type,
      success: success.toString(),
    };

    this.increment('banking.transaction.count', tags);
    this.histogram('banking.transaction.duration', duration, tags);
    this.distribution('banking.transaction.amount', amount, tags);

    if (!success) {
      this.increment('banking.transaction.error', tags);
    }
  }

  /**
   * Record business metric
   */
  static recordBusinessMetric(
    name: string,
    value: number,
    tags?: Record<string, string>
  ): void {
    this.distribution(`banking.business.${name}`, value, tags);
  }
}

/**
 * Custom tracing helpers
 */
export class DataDogTracing {
  /**
   * Create custom span
   */
  static async trace<T>(
    operationName: string,
    operation: (span: any) => Promise<T>,
    tags?: Record<string, any>
  ): Promise<T> {
    const span = tracer.startSpan(operationName, {
      tags: {
        ...tags,
        'resource.name': operationName,
      },
    });

    try {
      const result = await operation(span);
      span.setTag('status', 'success');
      return result;
    } catch (error) {
      span.setTag('error', true);
      span.setTag('error.message', (error as Error).message);
      span.setTag('error.stack', (error as Error).stack);
      throw error;
    } finally {
      span.finish();
    }
  }

  /**
   * Add tags to current span
   */
  static addTags(tags: Record<string, any>): void {
    const span = tracer.scope().active();
    if (span) {
      Object.entries(tags).forEach(([key, value]) => {
        span.setTag(key, value);
      });
    }
  }

  /**
   * Get trace ID (for log correlation)
   */
  static getTraceId(): string {
    const span = tracer.scope().active();
    return span ? span.context().toTraceId() : '';
  }
}
```

### 2. Instrumented Service with DataDog

```typescript
// src/services/PaymentService.ts

import { DataDogMetrics, DataDogTracing } from '../monitoring/datadog';

export class PaymentService {
  /**
   * Process payment with distributed tracing
   */
  async processPayment(
    accountId: string,
    amount: number,
    method: string
  ): Promise<any> {
    return await DataDogTracing.trace(
      'payment.process',
      async (span) => {
        const startTime = Date.now();
        let success = false;

        try {
          // Add custom tags
          DataDogTracing.addTags({
            'payment.account_id': accountId,
            'payment.amount': amount,
            'payment.method': method,
          });

          // Step 1: Validate account
          await this.validateAccount(accountId);
          span.setTag('validation', 'passed');

          // Step 2: Check fraud
          const fraudScore = await this.checkFraud(accountId, amount);
          span.setTag('fraud_score', fraudScore);

          if (fraudScore > 0.8) {
            throw new Error('High fraud risk detected');
          }

          // Step 3: Process with payment gateway
          const gatewayResponse = await this.callPaymentGateway(amount, method);
          span.setTag('gateway_response', gatewayResponse.status);

          // Step 4: Update account
          await this.updateAccountBalance(accountId, amount);

          success = true;
          return { success: true, transactionId: gatewayResponse.transactionId };
        } catch (error) {
          span.setTag('error.type', (error as Error).name);
          throw error;
        } finally {
          const duration = Date.now() - startTime;

          // Record metrics
          DataDogMetrics.recordTransaction(method, amount, duration, success);
          DataDogMetrics.recordBusinessMetric('payment_volume', amount, {
            method,
            success: success.toString(),
          });
        }
      },
      { 'service.name': 'payment-service' }
    );
  }

  private async validateAccount(accountId: string): Promise<void> {
    return await DataDogTracing.trace('payment.validate_account', async (span) => {
      span.setTag('account_id', accountId);
      // Validation logic
      await this.delay(50);
    });
  }

  private async checkFraud(accountId: string, amount: number): Promise<number> {
    return await DataDogTracing.trace('payment.check_fraud', async (span) => {
      span.setTag('account_id', accountId);
      span.setTag('amount', amount);
      // Fraud check logic
      await this.delay(100);
      return Math.random();
    });
  }

  private async callPaymentGateway(amount: number, method: string): Promise<any> {
    return await DataDogTracing.trace('payment.gateway_call', async (span) => {
      span.setTag('amount', amount);
      span.setTag('method', method);
      // External API call
      await this.delay(200);
      return { status: 'success', transactionId: 'TXN123' };
    });
  }

  private async updateAccountBalance(accountId: string, amount: number): Promise<void> {
    return await DataDogTracing.trace('payment.update_balance', async (span) => {
      span.setTag('account_id', accountId);
      span.setTag('amount', amount);
      // Database update
      await this.delay(75);
    });
  }

  private delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

### 3. Application with DataDog

```typescript
// src/index.ts

// ⚠️ IMPORTANT: Initialize DataDog FIRST
import { initializeDataDog } from './monitoring/datadog';
initializeDataDog();

import express from 'express';
import { PaymentService } from './services/PaymentService';
import { DataDogMetrics } from './monitoring/datadog';

const app = express();
const PORT = process.env.PORT || 3000;

app.use(express.json());

const paymentService = new PaymentService();

// Routes with automatic tracing
app.post('/api/payments', async (req, res, next) => {
  try {
    const result = await paymentService.processPayment(
      req.body.accountId,
      req.body.amount,
      req.body.method
    );
    res.json(result);
  } catch (error) {
    next(error);
  }
});

// Custom metrics endpoint
app.get('/metrics', (req, res) => {
  // Record custom gauge
  DataDogMetrics.gauge('banking.api.uptime', process.uptime(), {
    service: 'banking-api',
  });

  res.json({ status: 'ok', uptime: process.uptime() });
});

app.listen(PORT, () => {
  console.log(`🏦 Banking API running on port ${PORT}`);
});
```

---

## 🎯 Best Practices

### 1. Monitor Key Business Metrics

```typescript
// Track business-critical metrics
class BusinessMetrics {
  static recordDailyTransactionVolume(amount: number): void {
    newrelic.recordMetric('Custom/Business/DailyTransactionVolume', amount);
  }

  static recordCustomerSatisfaction(score: number): void {
    newrelic.recordMetric('Custom/Business/CustomerSatisfaction', score);
  }

  static recordRevenuePerTransaction(revenue: number): void {
    newrelic.recordMetric('Custom/Business/RevenuePerTransaction', revenue);
  }
}
```

### 2. Set Up Intelligent Alerting

```yaml
# New Relic Alert Policy (NRQL)
alerts:
  - name: "High Error Rate"
    condition: "SELECT percentage(count(*), WHERE error IS true) FROM Transaction WHERE appName = 'Banking API'"
    threshold: "> 5%"
    duration: "5 minutes"
    
  - name: "Slow Transaction Processing"
    condition: "SELECT percentile(duration, 95) FROM Transaction WHERE transactionType = 'TRANSFER'"
    threshold: "> 3 seconds"
    duration: "10 minutes"
    
  - name: "Low Transaction Success Rate"
    condition: "SELECT percentage(count(*), WHERE success = true) FROM BankingTransaction"
    threshold: "< 95%"
    duration: "5 minutes"
```

### 3. Use Distributed Tracing

```typescript
// Correlate logs with traces
import winston from 'winston';
import { DataDogTracing } from './monitoring/datadog';

const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
});

// Add trace ID to logs
function logWithTrace(message: string, level: string = 'info'): void {
  const traceId = DataDogTracing.getTraceId();
  logger.log(level, message, { dd: { trace_id: traceId } });
}
```

### 4. Profile Database Queries

```typescript
// Monitor slow database queries
async function executeQueryWithMonitoring<T>(
  query: string,
  params: any[]
): Promise<T> {
  const startTime = Date.now();
  
  try {
    const result = await pool.query(query, params);
    const duration = Date.now() - startTime;
    
    // Log slow queries
    if (duration > 500) {
      newrelic.noticeError(new Error('Slow query detected'), {
        query: query.substring(0, 100),
        duration,
        rowCount: result.rowCount,
      });
    }
    
    NewRelicMonitoring.recordDatabaseQuery(query, duration, result.rowCount);
    
    return result;
  } catch (error) {
    NewRelicMonitoring.reportError(error as Error, { query, params });
    throw error;
  }
}
```

---

## 📚 Common Interview Questions

### Q1: What's the difference between APM and logging?

**Answer**:
- **APM**: Performance monitoring, metrics, traces (How fast? How many?)
- **Logging**: Event records, debugging info (What happened? Why?)
- **Both**: Complementary - APM detects issues, logs explain them

### Q2: How do you monitor microservices?

**Answer**:
1. **Distributed tracing**: Track requests across services
2. **Service mesh**: Envoy/Istio for service-level metrics
3. **Unified dashboards**: Aggregate metrics from all services
4. **Correlation IDs**: Link related requests

### Q3: What is Apdex score?

**Answer**:
Application Performance Index - user satisfaction metric:
- **Satisfied**: Response time < T (target)
- **Tolerating**: Response time between T and 4T
- **Frustrated**: Response time > 4T
- **Score**: (Satisfied + 0.5 * Tolerating) / Total

### Q4: How do you handle high-cardinality metrics?

**Answer**:
1. **Aggregate**: Pre-aggregate before sending
2. **Sample**: Don't send every metric
3. **Tag wisely**: Avoid user IDs as tags
4. **Use distributions**: For percentiles

### Q5: What are SLIs, SLOs, and SLAs?

**Answer**:
- **SLI** (Service Level Indicator): Actual measurement (e.g., 99.5% uptime)
- **SLO** (Service Level Objective): Target value (e.g., ≥99.9% uptime)
- **SLA** (Service Level Agreement): Contract with penalties (e.g., 99.95% uptime or refund)

---

## ✅ Summary & Key Takeaways

### APM Setup Checklist

```yaml
✅ Installation:
  - [ ] Install APM agent (New Relic/DataDog)
  - [ ] Configure agent before app code
  - [ ] Set environment variables
  - [ ] Enable distributed tracing

✅ Instrumentation:
  - [ ] Auto-instrument Express/Fastify
  - [ ] Add custom transactions
  - [ ] Monitor database queries
  - [ ] Track external API calls
  - [ ] Monitor business metrics

✅ Alerting:
  - [ ] Error rate alerts
  - [ ] Performance degradation alerts
  - [ ] Business metric alerts
  - [ ] Infrastructure alerts
  - [ ] PagerDuty/Slack integration

✅ Dashboards:
  - [ ] Service overview dashboard
  - [ ] Database performance dashboard
  - [ ] Business metrics dashboard
  - [ ] Error tracking dashboard
  - [ ] SLO tracking dashboard
```

### Key Metrics to Monitor

```
┌──────────────────────────────────────────────────────┐
│         Banking API Monitoring Strategy              │
├──────────────────────────────────────────────────────┤
│                                                       │
│ Application Metrics:                                 │
│  ├─ Request rate (req/min)                           │
│  ├─ Response time (p50, p95, p99)                    │
│  ├─ Error rate (%)                                   │
│  └─ Apdex score                                      │
│                                                       │
│ Business Metrics:                                    │
│  ├─ Transaction volume ($)                           │
│  ├─ Transaction count (#)                            │
│  ├─ Success rate (%)                                 │
│  └─ Revenue per transaction ($)                      │
│                                                       │
│ Infrastructure Metrics:                              │
│  ├─ CPU usage (%)                                    │
│  ├─ Memory usage (MB)                                │
│  ├─ Disk I/O (ops/sec)                               │
│  └─ Network throughput (Mbps)                        │
│                                                       │
│ Database Metrics:                                    │
│  ├─ Query duration (ms)                              │
│  ├─ Connection pool usage (#)                        │
│  ├─ Slow queries (#)                                 │
│  └─ Deadlocks (#)                                    │
│                                                       │
└──────────────────────────────────────────────────────┘
```

---

**Status**: ✅ Complete with production-ready APM integration for comprehensive banking API monitoring!
