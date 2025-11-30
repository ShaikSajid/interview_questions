# Q36: Logging - Structured Logging & Log Aggregation

## 📋 Summary
Structured logging uses consistent, machine-readable formats (JSON) to enable powerful search, filtering, and analysis. This guide covers **JSON logging**, **ELK stack** (Elasticsearch, Logstash, Kibana), log correlation, distributed tracing, and complete banking examples for centralized audit systems.

---

## 🎯 What You'll Learn
- **Structured Logging**: JSON format, key-value pairs, consistent schema
- **Log Aggregation**: ELK stack, centralized logging, log shipping
- **Correlation IDs**: Track requests across microservices
- **Distributed Tracing**: OpenTelemetry, trace context propagation
- **Search & Analytics**: Kibana dashboards, log queries, alerting
- **Banking Examples**: Multi-service payment tracking, compliance reporting

---

## 📖 Comprehensive Explanation

### Structured vs Unstructured Logging

**Unstructured (Plain Text)** ❌:
```
2024-01-15 10:30:22 User alice transferred $500 to bob
```

Problems:
- Hard to parse programmatically
- Can't filter by amount or user
- Difficult to aggregate statistics

**Structured (JSON)** ✅:
```json
{
  "timestamp": "2024-01-15T10:30:22.000Z",
  "level": "info",
  "event": "transfer_completed",
  "userId": "alice",
  "recipient": "bob",
  "amount": 500,
  "currency": "USD",
  "transactionId": "TXN-123"
}
```

Benefits:
- Easy to parse and query
- Filter by any field
- Aggregate statistics
- Consistent format across services

---

## 📦 ELK Stack Overview

### Components

1. **Elasticsearch** 🔍
   - Distributed search and analytics engine
   - Stores logs as JSON documents
   - Full-text search capabilities
   - Aggregations and analytics

2. **Logstash** 📥
   - Log collection and processing pipeline
   - Parse, transform, enrich logs
   - Send to Elasticsearch

3. **Kibana** 📊
   - Visualization and dashboard tool
   - Search logs with Kibana Query Language (KQL)
   - Create charts, alerts, reports

4. **Filebeat** 📦 (Bonus)
   - Lightweight log shipper
   - Reads log files and sends to Logstash/Elasticsearch
   - Low resource usage

### Architecture

```
Node.js App → Winston Logger → Log Files → Filebeat → Logstash → Elasticsearch ← Kibana
```

Alternatively:
```
Node.js App → Winston HTTP Transport → Logstash → Elasticsearch ← Kibana
```

---

## 🔗 Correlation IDs

### What Are Correlation IDs?

**Correlation ID** (also called Request ID, Trace ID) is a unique identifier attached to every request, allowing you to track it across multiple services and log entries.

### Why Use Correlation IDs?

1. **Track requests** across microservices
2. **Debug distributed systems**
3. **Measure end-to-end latency**
4. **Group related logs** together

### Implementation

```javascript
const { v4: uuidv4 } = require('uuid');

// Middleware to generate correlation ID
function correlationMiddleware(req, res, next) {
  req.correlationId = req.get('X-Correlation-ID') || uuidv4();
  res.set('X-Correlation-ID', req.correlationId);
  next();
}

// Include in all logs
logger.info('Processing request', {
  correlationId: req.correlationId,
  method: req.method,
  path: req.path
});
```

---

## 📝 Example 1: Structured Logging with ELK Integration

### Winston Configuration for ELK

```javascript
// src/config/logger.js

const winston = require('winston');
const DailyRotateFile = require('winston-daily-rotate-file');
const path = require('path');

// JSON format with additional metadata
const elkFormat = winston.format.combine(
  winston.format.timestamp({ format: 'YYYY-MM-DDTHH:mm:ss.SSSZ' }),
  winston.format.errors({ stack: true }),
  winston.format.metadata({ fillExcept: ['message', 'level', 'timestamp'] }),
  winston.format.json()
);

// Create logger
const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: elkFormat,
  defaultMeta: {
    service: process.env.SERVICE_NAME || 'banking-api',
    environment: process.env.NODE_ENV || 'development',
    host: require('os').hostname(),
    pid: process.pid
  },
  transports: [
    // Console for development
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      )
    }),

    // JSON log files for Filebeat
    new DailyRotateFile({
      filename: path.join('logs', 'app-%DATE%.json'),
      datePattern: 'YYYY-MM-DD',
      maxSize: '100m',
      maxFiles: '14d',
      format: elkFormat
    }),

    // HTTP transport to Logstash (optional)
    new winston.transports.Http({
      host: process.env.LOGSTASH_HOST || 'localhost',
      port: process.env.LOGSTASH_PORT || 5000,
      path: '/logs',
      format: elkFormat
    })
  ]
});

// Child loggers for different modules
logger.payment = logger.child({ module: 'payment' });
logger.auth = logger.child({ module: 'auth' });
logger.account = logger.child({ module: 'account' });

module.exports = logger;
```

### Correlation ID Middleware

```javascript
// src/middleware/correlation.js

const { v4: uuidv4 } = require('uuid');
const { AsyncLocalStorage } = require('async_hooks');

// Store correlation ID per request
const asyncLocalStorage = new AsyncLocalStorage();

function correlationMiddleware(req, res, next) {
  const correlationId = req.get('X-Correlation-ID') || uuidv4();
  
  // Set in response header
  res.set('X-Correlation-ID', correlationId);
  
  // Store in async context
  asyncLocalStorage.run({ correlationId }, () => {
    req.correlationId = correlationId;
    next();
  });
}

function getCorrelationId() {
  const store = asyncLocalStorage.getStore();
  return store?.correlationId || 'no-correlation-id';
}

module.exports = { correlationMiddleware, getCorrelationId };
```

### Enhanced Logger with Correlation

```javascript
// src/utils/contextLogger.js

const logger = require('../config/logger');
const { getCorrelationId } = require('../middleware/correlation');

class ContextLogger {
  constructor(baseLogger) {
    this.baseLogger = baseLogger;
  }

  _addContext(metadata = {}) {
    return {
      ...metadata,
      correlationId: getCorrelationId(),
      timestamp: new Date().toISOString()
    };
  }

  info(message, metadata) {
    this.baseLogger.info(message, this._addContext(metadata));
  }

  error(message, metadata) {
    this.baseLogger.error(message, this._addContext(metadata));
  }

  warn(message, metadata) {
    this.baseLogger.warn(message, this._addContext(metadata));
  }

  debug(message, metadata) {
    this.baseLogger.debug(message, this._addContext(metadata));
  }

  // Specific event loggers
  logEvent(eventName, metadata) {
    this.info(`Event: ${eventName}`, {
      ...metadata,
      eventType: eventName
    });
  }
}

module.exports = new ContextLogger(logger);
```

### Banking Audit Logger with Correlation

```javascript
// src/services/auditLogger.js

const contextLogger = require('../utils/contextLogger');
const crypto = require('crypto');

class AuditLogger {
  /**
   * Log payment initiation
   */
  logPaymentInitiated(paymentData) {
    contextLogger.logEvent('PAYMENT_INITIATED', {
      paymentId: paymentData.paymentId,
      fromAccount: this.maskAccount(paymentData.fromAccount),
      toAccount: this.maskAccount(paymentData.toAccount),
      amount: paymentData.amount,
      currency: paymentData.currency,
      userId: paymentData.userId,
      paymentMethod: paymentData.method,
      auditLevel: 'HIGH'
    });
  }

  /**
   * Log payment completed
   */
  logPaymentCompleted(paymentData, processingTime) {
    contextLogger.logEvent('PAYMENT_COMPLETED', {
      paymentId: paymentData.paymentId,
      amount: paymentData.amount,
      currency: paymentData.currency,
      processingTime: `${processingTime}ms`,
      status: 'SUCCESS',
      auditLevel: 'HIGH'
    });
  }

  /**
   * Log payment validation
   */
  logPaymentValidation(paymentId, validationResult) {
    contextLogger.logEvent('PAYMENT_VALIDATED', {
      paymentId,
      validationStatus: validationResult.isValid ? 'PASSED' : 'FAILED',
      validationErrors: validationResult.errors || [],
      validationDuration: validationResult.duration,
      auditLevel: validationResult.isValid ? 'MEDIUM' : 'HIGH'
    });
  }

  /**
   * Log fraud check
   */
  logFraudCheck(paymentId, fraudScore, riskLevel) {
    contextLogger.logEvent('FRAUD_CHECK_COMPLETED', {
      paymentId,
      fraudScore,
      riskLevel,
      action: riskLevel === 'HIGH' ? 'BLOCKED' : 'ALLOWED',
      auditLevel: 'CRITICAL'
    });
  }

  /**
   * Log balance check
   */
  logBalanceCheck(accountId, requestedAmount, availableBalance) {
    const hasSufficientFunds = availableBalance >= requestedAmount;
    
    contextLogger.logEvent('BALANCE_CHECKED', {
      accountId: this.maskAccount(accountId),
      requestedAmount,
      availableBalance,
      hasSufficientFunds,
      auditLevel: 'MEDIUM'
    });
  }

  /**
   * Log account access
   */
  logAccountAccess(userId, accountId, action, ipAddress) {
    contextLogger.logEvent('ACCOUNT_ACCESS', {
      userId,
      accountId: this.maskAccount(accountId),
      action,
      ipAddress,
      userAgent: this.getUserAgent(),
      auditLevel: 'LOW'
    });
  }

  /**
   * Log compliance event
   */
  logComplianceEvent(eventType, details) {
    contextLogger.logEvent('COMPLIANCE_EVENT', {
      complianceEventType: eventType,
      ...details,
      auditLevel: 'CRITICAL',
      requiresReview: true
    });
  }

  /**
   * Mask sensitive account information
   */
  maskAccount(accountNumber) {
    if (!accountNumber) return '****';
    const visible = accountNumber.slice(-4);
    return `****${visible}`;
  }

  getUserAgent() {
    // Get from async context if available
    return 'User-Agent-String';
  }
}

module.exports = new AuditLogger();
```

### Multi-Service Payment Flow

```javascript
// src/services/paymentService.js

const auditLogger = require('./auditLogger');
const contextLogger = require('../utils/contextLogger');
const axios = require('axios');
const { getCorrelationId } = require('../middleware/correlation');

class PaymentService {
  async processPayment(paymentData) {
    const startTime = Date.now();
    const paymentId = `PAY-${Date.now()}-${Math.random().toString(36).substr(2, 9).toUpperCase()}`;
    
    const enrichedPaymentData = { ...paymentData, paymentId };

    try {
      // 1. Log initiation
      auditLogger.logPaymentInitiated(enrichedPaymentData);

      // 2. Validate payment
      contextLogger.info('Starting payment validation', { paymentId });
      const validationResult = await this.validatePayment(paymentId, paymentData);
      auditLogger.logPaymentValidation(paymentId, validationResult);

      if (!validationResult.isValid) {
        throw new Error(`Validation failed: ${validationResult.errors.join(', ')}`);
      }

      // 3. Fraud check (call fraud service)
      contextLogger.info('Starting fraud check', { paymentId });
      const fraudResult = await this.checkFraud(paymentId, paymentData);
      auditLogger.logFraudCheck(paymentId, fraudResult.score, fraudResult.riskLevel);

      if (fraudResult.riskLevel === 'HIGH') {
        throw new Error('Payment blocked due to fraud risk');
      }

      // 4. Check balance (call account service)
      contextLogger.info('Checking account balance', { paymentId });
      const balance = await this.checkBalance(paymentData.fromAccount);
      auditLogger.logBalanceCheck(paymentData.fromAccount, paymentData.amount, balance);

      if (balance < paymentData.amount) {
        throw new Error('Insufficient funds');
      }

      // 5. Execute transfer (call ledger service)
      contextLogger.info('Executing transfer', { paymentId });
      await this.executeTransfer(paymentData);

      // 6. Log completion
      const processingTime = Date.now() - startTime;
      auditLogger.logPaymentCompleted(enrichedPaymentData, processingTime);

      return {
        success: true,
        paymentId,
        processingTime
      };

    } catch (error) {
      contextLogger.error('Payment failed', {
        paymentId,
        error: error.message,
        stack: error.stack
      });
      throw error;
    }
  }

  async validatePayment(paymentId, paymentData) {
    const startTime = Date.now();
    const errors = [];

    if (!paymentData.amount || paymentData.amount <= 0) {
      errors.push('Invalid amount');
    }

    if (!paymentData.fromAccount || !paymentData.toAccount) {
      errors.push('Missing account information');
    }

    return {
      isValid: errors.length === 0,
      errors,
      duration: Date.now() - startTime
    };
  }

  async checkFraud(paymentId, paymentData) {
    // Call fraud detection service
    const correlationId = getCorrelationId();
    
    const response = await axios.post(
      'http://fraud-service:3001/check',
      paymentData,
      {
        headers: {
          'X-Correlation-ID': correlationId
        }
      }
    );

    return response.data; // { score: 0.2, riskLevel: 'LOW' }
  }

  async checkBalance(accountId) {
    const correlationId = getCorrelationId();
    
    const response = await axios.get(
      `http://account-service:3002/accounts/${accountId}/balance`,
      {
        headers: {
          'X-Correlation-ID': correlationId
        }
      }
    );

    return response.data.balance;
  }

  async executeTransfer(paymentData) {
    const correlationId = getCorrelationId();
    
    await axios.post(
      'http://ledger-service:3003/transfer',
      paymentData,
      {
        headers: {
          'X-Correlation-ID': correlationId
        }
      }
    );
  }
}

module.exports = new PaymentService();
```

### Express Application

```javascript
// src/app.js

const express = require('express');
const { correlationMiddleware } = require('./middleware/correlation');
const contextLogger = require('./utils/contextLogger');
const paymentService = require('./services/paymentService');

const app = express();

app.use(express.json());
app.use(correlationMiddleware);

// Payment endpoint
app.post('/api/payments', async (req, res) => {
  try {
    const paymentData = {
      fromAccount: req.body.fromAccount,
      toAccount: req.body.toAccount,
      amount: req.body.amount,
      currency: req.body.currency || 'USD',
      method: req.body.method || 'TRANSFER',
      userId: req.user?.id || 'guest'
    };

    const result = await paymentService.processPayment(paymentData);
    res.status(201).json(result);

  } catch (error) {
    contextLogger.error('Payment endpoint error', { error: error.message });
    res.status(400).json({ error: error.message });
  }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  contextLogger.info('Payment service started', { port: PORT });
});

module.exports = app;
```

### Sample Structured Logs (JSON)

```json
{
  "level": "info",
  "message": "Event: PAYMENT_INITIATED",
  "correlationId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "timestamp": "2024-01-15T10:30:22.123Z",
  "service": "banking-api",
  "environment": "production",
  "host": "payment-service-01",
  "pid": 1234,
  "module": "payment",
  "eventType": "PAYMENT_INITIATED",
  "paymentId": "PAY-1705315822-ABC123",
  "fromAccount": "****6789",
  "toAccount": "****1234",
  "amount": 500,
  "currency": "USD",
  "userId": "user-456",
  "paymentMethod": "TRANSFER",
  "auditLevel": "HIGH"
}

{
  "level": "info",
  "message": "Starting fraud check",
  "correlationId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "timestamp": "2024-01-15T10:30:22.234Z",
  "service": "banking-api",
  "paymentId": "PAY-1705315822-ABC123"
}

{
  "level": "info",
  "message": "Event: FRAUD_CHECK_COMPLETED",
  "correlationId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "timestamp": "2024-01-15T10:30:22.456Z",
  "service": "fraud-service",
  "eventType": "FRAUD_CHECK_COMPLETED",
  "paymentId": "PAY-1705315822-ABC123",
  "fraudScore": 0.15,
  "riskLevel": "LOW",
  "action": "ALLOWED",
  "auditLevel": "CRITICAL"
}

{
  "level": "info",
  "message": "Event: PAYMENT_COMPLETED",
  "correlationId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "timestamp": "2024-01-15T10:30:23.789Z",
  "service": "banking-api",
  "eventType": "PAYMENT_COMPLETED",
  "paymentId": "PAY-1705315822-ABC123",
  "amount": 500,
  "currency": "USD",
  "processingTime": "1666ms",
  "status": "SUCCESS",
  "auditLevel": "HIGH"
}
```

### Filebeat Configuration

```yaml
# filebeat.yml

filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /app/logs/*.json
    json.keys_under_root: true
    json.add_error_key: true

output.logstash:
  hosts: ["logstash:5044"]

processors:
  - add_cloud_metadata: ~
  - add_host_metadata: ~
```

### Logstash Configuration

```ruby
# logstash.conf

input {
  beats {
    port => 5044
  }
}

filter {
  # Parse JSON if not already parsed
  if [message] =~ /^\{/ {
    json {
      source => "message"
    }
  }

  # Add geoip for IP addresses
  if [ipAddress] {
    geoip {
      source => "ipAddress"
    }
  }

  # Parse timestamp
  date {
    match => [ "timestamp", "ISO8601" ]
    target => "@timestamp"
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "banking-logs-%{+YYYY.MM.dd}"
  }
}
```

### Kibana Queries (KQL)

```
# Find all payments for a specific correlation ID
correlationId: "a1b2c3d4-e5f6-7890-abcd-ef1234567890"

# Find failed payments
eventType: "PAYMENT_FAILED" AND level: "error"

# Find high-value transactions
amount > 10000

# Find fraud blocks
eventType: "FRAUD_CHECK_COMPLETED" AND action: "BLOCKED"

# Find slow payments
processingTime > 2000ms

# Compliance audit logs
auditLevel: "CRITICAL"
```

---

## 🎯 Key Takeaways

### Structured Logging Benefits

1. **Machine-readable format**
   - Easy to parse and analyze
   - Consistent schema across services
   - Powerful querying capabilities

2. **Correlation tracking**
   - Track requests across microservices
   - Debug distributed systems
   - Measure end-to-end performance

3. **Centralized visibility**
   - Single place to view all logs
   - Cross-service debugging
   - Real-time monitoring

4. **Compliance and auditing**
   - Complete audit trail
   - Track sensitive operations
   - Regulatory compliance (PCI-DSS, SOX)

### Best Practices

1. **Use correlation IDs**
   ```javascript
   // Always include correlation ID
   logger.info('Processing payment', {
     correlationId: req.correlationId,
     paymentId: 'PAY-123'
   });
   ```

2. **Consistent field names**
   ```javascript
   // ❌ Bad: Inconsistent
   { user_id: '123' }
   { userId: '456' }
   
   // ✅ Good: Consistent
   { userId: '123' }
   { userId: '456' }
   ```

3. **Include context**
   ```javascript
   logger.info('Payment processed', {
     paymentId,
     amount,
     duration,
     service: 'payment-service',
     environment: 'production'
   });
   ```

4. **Set retention policies**
   - Hot storage: 7 days (fast queries)
   - Warm storage: 30 days (slower queries)
   - Cold storage: 90 days (compliance)
   - Archive: 7 years (regulatory)

### ELK Stack Production Tips

- Use Filebeat (lightweight) instead of Logstash for log shipping
- Implement log sampling for high-volume services
- Use index lifecycle management (ILM) for retention
- Create Kibana dashboards for key metrics
- Set up alerts for critical errors
- Backup Elasticsearch indices regularly

---

**File**: `Q36_Logging_Structured_Logging.md`  
**Status**: ✅ Complete with structured JSON logging, ELK integration, correlation tracking, and multi-service banking audit system
