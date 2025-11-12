# Monitoring & Observability

## Question 21: Comprehensive Monitoring Solution

### 📋 Question Statement

Implement a complete observability stack for Emirates NBD microservices using CloudWatch, X-Ray, Prometheus, and Grafana. Include distributed tracing, metrics collection, log aggregation, and alerting.

---

### 🏗️ Observability Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│         EMIRATES NBD OBSERVABILITY & MONITORING ARCHITECTURE                │
└────────────────────────────────────────────────────────────────────────────┘

    METRICS COLLECTION           DISTRIBUTED TRACING
    ──────────────────          ────────────────────
    
    ┌──────────────┐            ┌──────────────┐
    │  CloudWatch  │            │    X-Ray     │
    │   Metrics    │            │  Trace Map   │
    │              │            │              │
    │  • ECS       │            │  • Spans     │
    │  • Lambda    │            │  • Segments  │
    │  • RDS       │            │  • Latency   │
    └──────┬───────┘            └──────┬───────┘
           │                           │
           └───────────┬───────────────┘
                       │
                       ▼
              ┌────────────────┐
              │   Grafana      │
              │   Dashboards   │
              └────────┬───────┘
                       │
                       ▼
              ┌────────────────┐
              │  CloudWatch    │
              │    Alarms      │
              └────────┬───────┘
                       │
                       ▼
              ┌────────────────┐
              │      SNS       │
              │    Alerts      │
              └────────────────┘
    
    LOG AGGREGATION
    ───────────────
    
    ┌──────────────┐
    │ CloudWatch   │
    │    Logs      │
    │              │
    │ • Service    │
    │   Logs       │
    │ • Access     │
    │   Logs       │
    │ • Error      │
    │   Logs       │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │   Lambda     │
    │  Log Parser  │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │ Elasticsearch│
    │  / OpenSearch│
    └──────────────┘
```

---

### 🔧 Monitoring Stack (CDK)

```typescript
// infrastructure/cdk/monitoring-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as cloudwatch from 'aws-cdk-lib/aws-cloudwatch';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as subscriptions from 'aws-cdk-lib/aws-sns-subscriptions';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as logs from 'aws-cdk-lib/aws-logs';
import * as ecs from 'aws-cdk-lib/aws-ecs';

export class MonitoringStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // ============================================
    // SNS TOPIC FOR ALERTS
    // ============================================
    const alertTopic = new sns.Topic(this, 'AlertTopic', {
      topicName: 'banking-alerts',
      displayName: 'Emirates NBD Banking Alerts'
    });

    alertTopic.addSubscription(new subscriptions.EmailSubscription('devops@emiratesnbd.com'));
    alertTopic.addSubscription(new subscriptions.SmsSubscription('+971501234567'));

    // ============================================
    // CLOUDWATCH DASHBOARD
    // ============================================
    const dashboard = new cloudwatch.Dashboard(this, 'BankingDashboard', {
      dashboardName: 'Emirates-NBD-Services'
    });

    // API Gateway Metrics
    const apiRequests = new cloudwatch.Metric({
      namespace: 'AWS/ApiGateway',
      metricName: 'Count',
      statistic: 'Sum',
      period: cdk.Duration.minutes(5)
    });

    const api4xxErrors = new cloudwatch.Metric({
      namespace: 'AWS/ApiGateway',
      metricName: '4XXError',
      statistic: 'Sum',
      period: cdk.Duration.minutes(5)
    });

    const api5xxErrors = new cloudwatch.Metric({
      namespace: 'AWS/ApiGateway',
      metricName: '5XXError',
      statistic: 'Sum',
      period: cdk.Duration.minutes(5)
    });

    const apiLatency = new cloudwatch.Metric({
      namespace: 'AWS/ApiGateway',
      metricName: 'Latency',
      statistic: 'Average',
      period: cdk.Duration.minutes(5)
    });

    dashboard.addWidgets(
      new cloudwatch.GraphWidget({
        title: 'API Gateway Requests',
        left: [apiRequests],
        width: 12
      }),
      new cloudwatch.GraphWidget({
        title: 'API Gateway Errors',
        left: [api4xxErrors, api5xxErrors],
        width: 12
      }),
      new cloudwatch.GraphWidget({
        title: 'API Gateway Latency',
        left: [apiLatency],
        width: 24
      })
    );

    // Lambda Metrics
    const lambdaInvocations = new cloudwatch.Metric({
      namespace: 'AWS/Lambda',
      metricName: 'Invocations',
      statistic: 'Sum',
      period: cdk.Duration.minutes(5)
    });

    const lambdaErrors = new cloudwatch.Metric({
      namespace: 'AWS/Lambda',
      metricName: 'Errors',
      statistic: 'Sum',
      period: cdk.Duration.minutes(5)
    });

    const lambdaDuration = new cloudwatch.Metric({
      namespace: 'AWS/Lambda',
      metricName: 'Duration',
      statistic: 'Average',
      period: cdk.Duration.minutes(5)
    });

    const lambdaThrottles = new cloudwatch.Metric({
      namespace: 'AWS/Lambda',
      metricName: 'Throttles',
      statistic: 'Sum',
      period: cdk.Duration.minutes(5)
    });

    dashboard.addWidgets(
      new cloudwatch.GraphWidget({
        title: 'Lambda Invocations',
        left: [lambdaInvocations],
        width: 12
      }),
      new cloudwatch.GraphWidget({
        title: 'Lambda Errors & Throttles',
        left: [lambdaErrors, lambdaThrottles],
        width: 12
      }),
      new cloudwatch.GraphWidget({
        title: 'Lambda Duration',
        left: [lambdaDuration],
        width: 24
      })
    );

    // DynamoDB Metrics
    const dynamoReadCapacity = new cloudwatch.Metric({
      namespace: 'AWS/DynamoDB',
      metricName: 'ConsumedReadCapacityUnits',
      statistic: 'Sum',
      period: cdk.Duration.minutes(5)
    });

    const dynamoWriteCapacity = new cloudwatch.Metric({
      namespace: 'AWS/DynamoDB',
      metricName: 'ConsumedWriteCapacityUnits',
      statistic: 'Sum',
      period: cdk.Duration.minutes(5)
    });

    dashboard.addWidgets(
      new cloudwatch.GraphWidget({
        title: 'DynamoDB Capacity',
        left: [dynamoReadCapacity],
        right: [dynamoWriteCapacity],
        width: 24
      })
    );

    // ============================================
    // CLOUDWATCH ALARMS
    // ============================================

    // High Error Rate Alarm
    const highErrorAlarm = new cloudwatch.Alarm(this, 'HighErrorRateAlarm', {
      metric: api5xxErrors,
      threshold: 10,
      evaluationPeriods: 2,
      datapointsToAlarm: 2,
      comparisonOperator: cloudwatch.ComparisonOperator.GREATER_THAN_THRESHOLD,
      treatMissingData: cloudwatch.TreatMissingData.NOT_BREACHING
    });

    highErrorAlarm.addAlarmAction(new cw_actions.SnsAction(alertTopic));

    // High Latency Alarm
    const highLatencyAlarm = new cloudwatch.Alarm(this, 'HighLatencyAlarm', {
      metric: apiLatency,
      threshold: 1000, // 1 second
      evaluationPeriods: 3,
      comparisonOperator: cloudwatch.ComparisonOperator.GREATER_THAN_THRESHOLD
    });

    highLatencyAlarm.addAlarmAction(new cw_actions.SnsAction(alertTopic));

    // Lambda Error Rate Alarm
    const lambdaErrorAlarm = new cloudwatch.Alarm(this, 'LambdaErrorAlarm', {
      metric: lambdaErrors,
      threshold: 5,
      evaluationPeriods: 2,
      comparisonOperator: cloudwatch.ComparisonOperator.GREATER_THAN_THRESHOLD
    });

    lambdaErrorAlarm.addAlarmAction(new cw_actions.SnsAction(alertTopic));

    // Database Connection Alarm
    const dbConnectionAlarm = new cloudwatch.Alarm(this, 'DBConnectionAlarm', {
      metric: new cloudwatch.Metric({
        namespace: 'AWS/RDS',
        metricName: 'DatabaseConnections',
        statistic: 'Average'
      }),
      threshold: 80,
      evaluationPeriods: 2,
      comparisonOperator: cloudwatch.ComparisonOperator.GREATER_THAN_THRESHOLD
    });

    dbConnectionAlarm.addAlarmAction(new cw_actions.SnsAction(alertTopic));

    // ============================================
    // CUSTOM METRICS
    // ============================================
    
    // Business Metrics Lambda
    const metricsCollector = new lambda.Function(this, 'MetricsCollector', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromInline(`
        const { CloudWatchClient, PutMetricDataCommand } = require('@aws-sdk/client-cloudwatch');
        const cloudwatch = new CloudWatchClient({});
        
        exports.handler = async (event) => {
          await cloudwatch.send(new PutMetricDataCommand({
            Namespace: 'BankingApplication',
            MetricData: [{
              MetricName: 'TransactionCount',
              Value: event.transactionCount || 0,
              Unit: 'Count'
            }]
          }));
        };
      `)
    });

    new cdk.CfnOutput(this, 'DashboardURL', {
      value: `https://console.aws.amazon.com/cloudwatch/home?region=${this.region}#dashboards:name=${dashboard.dashboardName}`
    });
  }
}
```

### 📊 X-Ray Distributed Tracing

```javascript
// services/account-service/src/middleware/xray-middleware.js
const AWSXRay = require('aws-xray-sdk-core');
const AWS = AWSXRay.captureAWS(require('aws-sdk'));
const http = AWSXRay.captureHTTPs(require('http'));
const https = AWSXRay.captureHTTPs(require('https'));

// Express middleware
const xrayMiddleware = AWSXRay.express.openSegment('AccountService');

// Custom subsegments for business logic
async function traceFunction(name, fn) {
  const segment = AWSXRay.getSegment();
  const subsegment = segment.addNewSubsegment(name);
  
  try {
    const result = await fn();
    subsegment.close();
    return result;
  } catch (error) {
    subsegment.addError(error);
    subsegment.close();
    throw error;
  }
}

// Add annotations (indexed)
function addAnnotation(key, value) {
  const segment = AWSXRay.getSegment();
  if (segment) {
    segment.addAnnotation(key, value);
  }
}

// Add metadata (not indexed)
function addMetadata(key, value) {
  const segment = AWSXRay.getSegment();
  if (segment) {
    segment.addMetadata(key, value);
  }
}

module.exports = {
  xrayMiddleware,
  traceFunction,
  addAnnotation,
  addMetadata,
  closeSegment: AWSXRay.express.closeSegment()
};
```

```javascript
// Usage in Express app
const express = require('express');
const { xrayMiddleware, closeSegment, traceFunction, addAnnotation } = require('./middleware/xray-middleware');

const app = express();

// Apply X-Ray middleware
app.use(xrayMiddleware);

app.post('/accounts', async (req, res) => {
  addAnnotation('customerId', req.body.customerId);
  addAnnotation('accountType', req.body.accountType);
  
  const account = await traceFunction('CreateAccount', async () => {
    return await accountService.create(req.body);
  });
  
  res.json(account);
});

app.use(closeSegment);
```

### 📈 Custom Metrics Publisher

```javascript
// services/shared/src/metrics-publisher.js
const { CloudWatchClient, PutMetricDataCommand } = require('@aws-sdk/client-cloudwatch');

class MetricsPublisher {
  constructor() {
    this.client = new CloudWatchClient({});
    this.namespace = 'BankingApplication';
    this.buffer = [];
    this.bufferSize = 20; // CloudWatch limit
  }

  async publishMetric(metricName, value, unit = 'Count', dimensions = {}) {
    const metric = {
      MetricName: metricName,
      Value: value,
      Unit: unit,
      Timestamp: new Date(),
      Dimensions: Object.entries(dimensions).map(([Name, Value]) => ({ Name, Value }))
    };

    this.buffer.push(metric);

    if (this.buffer.length >= this.bufferSize) {
      await this.flush();
    }
  }

  async flush() {
    if (this.buffer.length === 0) return;

    try {
      await this.client.send(new PutMetricDataCommand({
        Namespace: this.namespace,
        MetricData: this.buffer
      }));
      
      this.buffer = [];
    } catch (error) {
      console.error('Failed to publish metrics:', error);
    }
  }

  // Business metrics
  async recordTransaction(type, amount, status) {
    await this.publishMetric('TransactionCount', 1, 'Count', {
      TransactionType: type,
      Status: status
    });

    await this.publishMetric('TransactionAmount', amount, 'None', {
      TransactionType: type
    });
  }

  async recordAPICall(endpoint, method, statusCode, duration) {
    await this.publishMetric('APICall', 1, 'Count', {
      Endpoint: endpoint,
      Method: method,
      StatusCode: statusCode.toString()
    });

    await this.publishMetric('APILatency', duration, 'Milliseconds', {
      Endpoint: endpoint
    });
  }

  async recordError(errorType, service) {
    await this.publishMetric('ErrorCount', 1, 'Count', {
      ErrorType: errorType,
      Service: service
    });
  }
}

module.exports = new MetricsPublisher();
```

### 🎓 Interview Discussion Points - Q21

**Q1: What are the pillars of observability?**

**A**:
- **Logs**: What happened
- **Metrics**: How much/how many
- **Traces**: Where time was spent

**Q2: What metrics should you monitor?**

**A**:
- **RED**: Rate, Errors, Duration
- **USE**: Utilization, Saturation, Errors
- **Business**: Transactions, revenue, users

**Q3: How do you reduce alert fatigue?**

**A**:
- Set appropriate thresholds
- Use composite alarms
- Implement escalation policies
- Review and tune regularly

---

## Question 22: Log Aggregation & Analysis

### 📋 Question Statement

Implement log aggregation for Emirates NBD using CloudWatch Logs Insights, Elasticsearch/OpenSearch, and structured logging. Include log parsing, searching, and correlation with distributed traces.

---

### 🏗️ Logging Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│              EMIRATES NBD LOG AGGREGATION ARCHITECTURE                      │
└────────────────────────────────────────────────────────────────────────────┘

    APPLICATION LOGS              LOG PROCESSING
    ────────────────             ──────────────
    
    ┌──────────────┐             ┌──────────────┐
    │     ECS      │────────────►│  CloudWatch  │
    │   Services   │   stdout    │     Logs     │
    └──────────────┘             └──────┬───────┘
                                        │
    ┌──────────────┐                    │
    │   Lambda     │────────────────────┤
    │  Functions   │                    │
    └──────────────┘                    │
                                        │
                                        ▼
                                 ┌──────────────┐
                                 │  Lambda      │
                                 │  Subscriber  │
                                 └──────┬───────┘
                                        │
                                        ▼
                                 ┌──────────────┐
                                 │ OpenSearch   │
                                 │   Cluster    │
                                 └──────┬───────┘
                                        │
                                        ▼
                                 ┌──────────────┐
                                 │   Kibana     │
                                 │  Dashboards  │
                                 └──────────────┘
```

---

### 📝 Structured Logging

```javascript
// services/shared/src/logger.js
const winston = require('winston');
const AWSXRay = require('aws-xray-sdk-core');

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: {
    service: process.env.SERVICE_NAME || 'unknown',
    environment: process.env.ENVIRONMENT || 'development',
    version: process.env.APP_VERSION || '1.0.0'
  },
  transports: [
    new winston.transports.Console()
  ]
});

// Add trace ID from X-Ray
logger.addTraceId = () => {
  const segment = AWSXRay.getSegment();
  if (segment) {
    return {
      traceId: segment.trace_id,
      segmentId: segment.id
    };
  }
  return {};
};

// Structured log methods
logger.logRequest = (req) => {
  logger.info('HTTP Request', {
    ...logger.addTraceId(),
    method: req.method,
    path: req.path,
    query: req.query,
    ip: req.ip,
    userAgent: req.get('user-agent'),
    userId: req.user?.id
  });
};

logger.logResponse = (req, res, duration) => {
  logger.info('HTTP Response', {
    ...logger.addTraceId(),
    method: req.method,
    path: req.path,
    statusCode: res.statusCode,
    duration: `${duration}ms`,
    userId: req.user?.id
  });
};

logger.logTransaction = (transaction) => {
  logger.info('Transaction', {
    ...logger.addTraceId(),
    transactionId: transaction.id,
    type: transaction.type,
    amount: transaction.amount,
    fromAccount: transaction.fromAccount,
    toAccount: transaction.toAccount,
    status: transaction.status
  });
};

logger.logError = (error, context = {}) => {
  logger.error('Error occurred', {
    ...logger.addTraceId(),
    error: {
      name: error.name,
      message: error.message,
      stack: error.stack
    },
    ...context
  });
};

module.exports = logger;
```

### 🔍 CloudWatch Logs Insights Queries

```sql
-- Find all errors in the last hour
fields @timestamp, @message, error.message, error.stack
| filter @message like /ERROR/
| sort @timestamp desc
| limit 100

-- Transaction latency analysis
fields @timestamp, transactionId, type, duration
| filter type = "Transaction"
| stats avg(duration), max(duration), min(duration) by type

-- API endpoint performance
fields @timestamp, method, path, statusCode, duration
| filter method = "POST" and path like /accounts/
| stats count(), avg(duration), percentile(duration, 95) by path

-- Error rate by service
fields @timestamp, service, error.name
| filter ispresent(error)
| stats count() as errorCount by service, error.name
| sort errorCount desc

-- User activity tracking
fields @timestamp, userId, method, path
| filter ispresent(userId)
| stats count() as requestCount by userId
| sort requestCount desc
| limit 20
```

### 🎓 Interview Discussion Points - Q22

**Q1: What is structured logging?**

**A**:
- JSON format logs
- Consistent field names
- Easy to parse and query
- Better than free-text logs

**Q2: How do you correlate logs across services?**

**A**:
- **Trace ID**: From X-Ray or custom
- **Correlation ID**: Generated at entry point
- **Session ID**: For user sessions
- **Request ID**: Unique per request

**Q3: What should you log?**

**A**:
- **DO**: Errors, user actions, transactions, security events
- **DON'T**: Passwords, credit cards, PII

---

**End of File 11**