# Question 6: Distributed Systems Patterns in AWS

## 🎯 Question
**How would you design a resilient distributed system for Bayer's integration platform? Discuss patterns like Circuit Breaker, Retry with Exponential Backoff, Idempotency, Event Sourcing, CQRS, Saga, and how to handle distributed transactions, eventual consistency, and failure scenarios.**

---

## 📋 Answer Overview

Distributed systems introduce challenges:
- **Network failures**: Services can become unavailable
- **Partial failures**: Some operations succeed, others fail
- **Eventual consistency**: Data not immediately consistent across services
- **Distributed transactions**: Coordinating updates across multiple databases

Key patterns for resilience:
- **Circuit Breaker**: Stop calling failing services
- **Retry + Backoff**: Automatically retry failed operations
- **Idempotency**: Safe to retry operations multiple times
- **Event Sourcing**: Store all changes as events
- **CQRS**: Separate read and write models
- **Saga**: Manage distributed transactions

---

## 🏗️ Distributed Architecture

```
┌────────────────────────────────────────────────────────────┐
│              Bayer Integration Platform                    │
│         (Distributed Microservices)                        │
└────────────────────────────────────────────────────────────┘

┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Patient    │     │  Clinical   │     │  Mfg        │
│  Service    │     │  Service    │     │  Service    │
├─────────────┤     ├─────────────┤     ├─────────────┤
│ DynamoDB    │     │ RDS         │     │ DynamoDB    │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                    ┌──────▼──────┐
                    │  EventBridge│
                    │  (Event Bus)│
                    └──────┬──────┘
                           │
       ┌───────────────────┼───────────────────┐
       │                   │                   │
       ▼                   ▼                   ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Notification│     │ Audit       │     │ Analytics   │
│ Service     │     │ Service     │     │ Service     │
└─────────────┘     └─────────────┘     └─────────────┘

Patterns Applied:
✓ Circuit Breaker: Fail fast when service down
✓ Retry + Backoff: Automatic retry with delays
✓ Idempotency: Duplicate-safe operations
✓ Event-Driven: Asynchronous communication
✓ CQRS: Separate read/write data stores
```

---

## 💻 Implementation Examples

### 1. Circuit Breaker Pattern

```javascript
// circuit-breaker.js
// WHY: Prevent cascading failures in distributed systems
// EXAMPLE: If SAP system down, don't keep trying (fail fast)

class CircuitBreaker {
  constructor(options = {}) {
    this.failureThreshold = options.failureThreshold || 5;  // Failures before opening
    this.successThreshold = options.successThreshold || 2;  // Successes to close
    this.timeout = options.timeout || 60000;  // 60 seconds in OPEN state
    
    this.state = 'CLOSED';  // CLOSED, OPEN, HALF_OPEN
    this.failureCount = 0;
    this.successCount = 0;
    this.nextAttempt = Date.now();
    
    // WHY: Track circuit breaker state for monitoring
    this.stateChangeCallbacks = [];
  }
  
  /**
   * Execute function with circuit breaker protection
   * WHY: Automatically fail fast when service unhealthy
   */
  async execute(fn) {
    // Check if circuit is OPEN
    // WHY: Don't waste time calling failing service
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        const error = new Error('Circuit breaker is OPEN');
        error.circuitBreakerState = 'OPEN';
        throw error;
      }
      
      // Try again after timeout (HALF_OPEN state)
      // WHY: Periodically check if service recovered
      this.state = 'HALF_OPEN';
      this.notifyStateChange('HALF_OPEN');
      console.log('Circuit breaker entering HALF_OPEN state');
    }
    
    try {
      const result = await fn();
      this.onSuccess();
      return result;
      
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  /**
   * Handle successful execution
   * WHY: Track successes to close circuit
   */
  onSuccess() {
    this.failureCount = 0;
    
    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      
      // Close circuit after threshold successes
      // WHY: Service recovered, resume normal operation
      if (this.successCount >= this.successThreshold) {
        this.state = 'CLOSED';
        this.successCount = 0;
        this.notifyStateChange('CLOSED');
        console.log('Circuit breaker CLOSED (service recovered)');
      }
    }
  }
  
  /**
   * Handle failed execution
   * WHY: Track failures to open circuit
   */
  onFailure() {
    this.failureCount++;
    
    // Open circuit after threshold failures
    // WHY: Stop calling failing service
    if (this.failureCount >= this.failureThreshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.timeout;
      this.notifyStateChange('OPEN');
      console.error('Circuit breaker OPEN (too many failures)', {
        failureCount: this.failureCount,
        nextAttempt: new Date(this.nextAttempt)
      });
    }
  }
  
  /**
   * Notify listeners of state changes
   * WHY: CloudWatch alarms, dashboards
   */
  notifyStateChange(newState) {
    this.stateChangeCallbacks.forEach(callback => {
      callback(newState);
    });
  }
  
  /**
   * Register state change listener
   * WHY: Monitor circuit breaker state
   */
  onStateChange(callback) {
    this.stateChangeCallbacks.push(callback);
  }
}

// Usage Example: SAP Integration with Circuit Breaker
const sapCircuitBreaker = new CircuitBreaker({
  failureThreshold: 5,   // Open after 5 failures
  successThreshold: 2,   // Close after 2 successes
  timeout: 60000         // Try again after 60 seconds
});

// Monitor circuit breaker state
sapCircuitBreaker.onStateChange((state) => {
  console.log('SAP circuit breaker state changed', { state });
  
  // Send CloudWatch metric
  // WHY: Alert on-call engineer when circuit opens
  const cloudwatch = new AWS.CloudWatch();
  cloudwatch.putMetricData({
    Namespace: 'Integration/CircuitBreaker',
    MetricData: [{
      MetricName: 'CircuitBreakerState',
      Value: state === 'OPEN' ? 1 : 0,
      Dimensions: [
        { Name: 'Service', Value: 'SAP' }
      ]
    }]
  }).promise();
});

/**
 * Call SAP API with circuit breaker
 * WHY: Fail fast if SAP down, don't waste Lambda time
 */
async function callSAPAPI(endpoint, data) {
  try {
    return await sapCircuitBreaker.execute(async () => {
      const response = await fetch(`https://sap.bayer.com/${endpoint}`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${process.env.SAP_API_KEY}`
        },
        body: JSON.stringify(data),
        timeout: 5000  // 5 second timeout
      });
      
      if (!response.ok) {
        throw new Error(`SAP API error: ${response.status}`);
      }
      
      return await response.json();
    });
    
  } catch (error) {
    if (error.circuitBreakerState === 'OPEN') {
      console.log('SAP circuit breaker is OPEN, skipping call');
      // WHY: Return cached data or fallback response
      return getCachedSAPData(endpoint);
    }
    
    throw error;
  }
}
```

---

### 2. Retry with Exponential Backoff

```javascript
// retry-with-backoff.js
// WHY: Automatically retry transient failures
// TRANSIENT FAILURES: Network timeouts, rate limits, temporary unavailability

/**
 * Retry function with exponential backoff
 * WHY: Give service time to recover between retries
 */
async function retryWithBackoff(fn, options = {}) {
  const maxRetries = options.maxRetries || 3;
  const baseDelay = options.baseDelay || 1000;  // 1 second
  const maxDelay = options.maxDelay || 30000;   // 30 seconds
  const factor = options.factor || 2;            // Exponential factor
  const jitter = options.jitter !== false;       // Random jitter
  
  let lastError;
  
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
      
    } catch (error) {
      lastError = error;
      
      // Don't retry on certain errors
      // WHY: No point retrying 400 Bad Request or 401 Unauthorized
      if (isNonRetryableError(error)) {
        throw error;
      }
      
      // Last attempt, give up
      if (attempt === maxRetries) {
        console.error('All retries exhausted', {
          attempts: attempt + 1,
          error: error.message
        });
        break;
      }
      
      // Calculate delay: baseDelay * factor^attempt
      // EXAMPLE: 1s, 2s, 4s, 8s, 16s
      let delay = Math.min(baseDelay * Math.pow(factor, attempt), maxDelay);
      
      // Add jitter to prevent thundering herd
      // WHY: If 1000 Lambdas retry at same time, overwhelm service
      if (jitter) {
        delay = delay * (0.5 + Math.random() * 0.5);
      }
      
      console.log('Retrying after delay', {
        attempt: attempt + 1,
        maxRetries,
        delayMs: Math.round(delay),
        error: error.message
      });
      
      await sleep(delay);
    }
  }
  
  throw lastError;
}

/**
 * Check if error is retryable
 * WHY: Don't waste retries on permanent failures
 */
function isNonRetryableError(error) {
  const nonRetryableCodes = [400, 401, 403, 404, 422];
  
  // HTTP status codes
  if (error.statusCode && nonRetryableCodes.includes(error.statusCode)) {
    return true;
  }
  
  // AWS SDK error codes
  const nonRetryableAWSErrors = [
    'ValidationException',
    'AccessDeniedException',
    'InvalidParameterException'
  ];
  
  if (error.code && nonRetryableAWSErrors.includes(error.code)) {
    return true;
  }
  
  return false;
}

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// Usage Example: Retry DynamoDB operation
async function updatePatientRecord(patientId, updates) {
  const dynamodb = new AWS.DynamoDB.DocumentClient();
  
  return await retryWithBackoff(async () => {
    await dynamodb.update({
      TableName: process.env.PATIENTS_TABLE,
      Key: { patientId },
      UpdateExpression: 'SET #status = :status, #updatedAt = :updatedAt',
      ExpressionAttributeNames: {
        '#status': 'status',
        '#updatedAt': 'updatedAt'
      },
      ExpressionAttributeValues: {
        ':status': updates.status,
        ':updatedAt': new Date().toISOString()
      }
    }).promise();
  }, {
    maxRetries: 3,
    baseDelay: 1000,
    factor: 2
  });
}
```

---

### 3. Idempotency Pattern

```javascript
// idempotency.js
// WHY: Safe to retry operations multiple times
// CRITICAL: Prevent duplicate charges, duplicate records

const AWS = require('aws-sdk');
const dynamodb = new AWS.DynamoDB.DocumentClient();

/**
 * Idempotent operation handler
 * WHY: Duplicate requests return same result (don't execute twice)
 * EXAMPLE: If API call retried, don't create duplicate patient record
 */
class IdempotencyHandler {
  constructor(tableName) {
    this.tableName = tableName;
    this.ttl = 24 * 60 * 60;  // 24 hours
  }
  
  /**
   * Execute operation with idempotency guarantee
   * WHY: Check if operation already executed
   */
  async execute(idempotencyKey, operation) {
    // Check if operation already executed
    // WHY: Return cached result if duplicate request
    const existing = await this.getIdempotencyRecord(idempotencyKey);
    
    if (existing) {
      if (existing.status === 'IN_PROGRESS') {
        // Operation still in progress, wait
        // WHY: Concurrent requests for same operation
        throw new Error('Operation in progress');
      }
      
      if (existing.status === 'COMPLETED') {
        console.log('Returning cached result (idempotent)', {
          idempotencyKey
        });
        return existing.result;
      }
    }
    
    // Mark operation as IN_PROGRESS
    // WHY: Prevent concurrent execution
    await this.setIdempotencyRecord(idempotencyKey, 'IN_PROGRESS', null);
    
    try {
      // Execute operation
      const result = await operation();
      
      // Mark as COMPLETED with result
      // WHY: Cache result for future duplicate requests
      await this.setIdempotencyRecord(idempotencyKey, 'COMPLETED', result);
      
      return result;
      
    } catch (error) {
      // Mark as FAILED
      // WHY: Allow retry after failure
      await this.setIdempotencyRecord(idempotencyKey, 'FAILED', {
        error: error.message
      });
      
      throw error;
    }
  }
  
  /**
   * Get idempotency record from DynamoDB
   */
  async getIdempotencyRecord(key) {
    try {
      const result = await dynamodb.get({
        TableName: this.tableName,
        Key: { idempotencyKey: key }
      }).promise();
      
      return result.Item;
      
    } catch (error) {
      console.error('Failed to get idempotency record', {
        key,
        error: error.message
      });
      return null;
    }
  }
  
  /**
   * Set idempotency record in DynamoDB
   */
  async setIdempotencyRecord(key, status, result) {
    await dynamodb.put({
      TableName: this.tableName,
      Item: {
        idempotencyKey: key,
        status,
        result,
        createdAt: new Date().toISOString(),
        ttl: Math.floor(Date.now() / 1000) + this.ttl
        // WHY: Auto-delete after 24 hours (save storage costs)
      }
    }).promise();
  }
}

// Usage Example: Idempotent patient creation
const idempotencyHandler = new IdempotencyHandler(process.env.IDEMPOTENCY_TABLE);

exports.handler = async (event) => {
  // Generate idempotency key from request
  // WHY: Same input = same idempotency key
  const idempotencyKey = event.headers['X-Idempotency-Key'] 
    || generateIdempotencyKey(event.body);
  
  console.log('Processing request with idempotency', {
    idempotencyKey
  });
  
  try {
    const result = await idempotencyHandler.execute(
      idempotencyKey,
      async () => {
        // Create patient record
        const patient = JSON.parse(event.body);
        
        await dynamodb.put({
          TableName: process.env.PATIENTS_TABLE,
          Item: {
            patientId: generatePatientId(),
            ...patient,
            createdAt: new Date().toISOString()
          }
        }).promise();
        
        return { patientId, message: 'Patient created' };
      }
    );
    
    return {
      statusCode: 200,
      body: JSON.stringify(result)
    };
    
  } catch (error) {
    console.error('Failed to create patient', {
      error: error.message
    });
    
    return {
      statusCode: 500,
      body: JSON.stringify({ error: error.message })
    };
  }
};

/**
 * Generate idempotency key from request body
 * WHY: Hash ensures same input = same key
 */
function generateIdempotencyKey(body) {
  const crypto = require('crypto');
  return crypto.createHash('sha256').update(body).digest('hex');
}
```

---

### 4. Saga Pattern (Distributed Transactions)

```javascript
// saga-pattern.js
// WHY: Coordinate distributed transactions across multiple services
// EXAMPLE: Order placement requires: inventory, payment, shipping

/**
 * Saga orchestrator
 * WHY: Manage multi-step transaction with compensating actions
 */
class SagaOrchestrator {
  constructor(steps) {
    this.steps = steps;  // Array of { execute, compensate }
  }
  
  /**
   * Execute saga with rollback on failure
   * WHY: Ensure eventual consistency across services
   */
  async execute(context) {
    const completedSteps = [];
    
    try {
      // Execute each step
      for (const step of this.steps) {
        console.log('Executing saga step', {
          step: step.name,
          context
        });
        
        await step.execute(context);
        completedSteps.push(step);
      }
      
      console.log('Saga completed successfully');
      return { success: true, context };
      
    } catch (error) {
      console.error('Saga failed, executing compensations', {
        error: error.message,
        completedSteps: completedSteps.length
      });
      
      // Execute compensating actions in reverse order
      // WHY: Rollback already completed steps
      for (const step of completedSteps.reverse()) {
        try {
          console.log('Compensating saga step', {
            step: step.name
          });
          
          await step.compensate(context);
          
        } catch (compensateError) {
          console.error('Compensation failed', {
            step: step.name,
            error: compensateError.message
          });
          // Continue with other compensations
        }
      }
      
      throw error;
    }
  }
}

// Example: Clinical Trial Enrollment Saga
const enrollmentSaga = new SagaOrchestrator([
  {
    name: 'ReservePatientSlot',
    execute: async (context) => {
      // Reserve slot in trial
      const dynamodb = new AWS.DynamoDB.DocumentClient();
      await dynamodb.update({
        TableName: 'ClinicalTrials',
        Key: { trialId: context.trialId },
        UpdateExpression: 'SET #slots = #slots - :dec',
        ExpressionAttributeNames: { '#slots': 'availableSlots' },
        ExpressionAttributeValues: { ':dec': 1 },
        ConditionExpression: '#slots > :zero'
      }).promise();
    },
    compensate: async (context) => {
      // Release slot
      const dynamodb = new AWS.DynamoDB.DocumentClient();
      await dynamodb.update({
        TableName: 'ClinicalTrials',
        Key: { trialId: context.trialId },
        UpdateExpression: 'SET #slots = #slots + :inc',
        ExpressionAttributeNames: { '#slots': 'availableSlots' },
        ExpressionAttributeValues: { ':inc': 1 }
      }).promise();
    }
  },
  {
    name: 'CreatePatientRecord',
    execute: async (context) => {
      const dynamodb = new AWS.DynamoDB.DocumentClient();
      await dynamodb.put({
        TableName: 'TrialPatients',
        Item: {
          patientId: context.patientId,
          trialId: context.trialId,
          enrolledAt: new Date().toISOString()
        }
      }).promise();
    },
    compensate: async (context) => {
      const dynamodb = new AWS.DynamoDB.DocumentClient();
      await dynamodb.delete({
        TableName: 'TrialPatients',
        Key: {
          patientId: context.patientId,
          trialId: context.trialId
        }
      }).promise();
    }
  },
  {
    name: 'SendWelcomeEmail',
    execute: async (context) => {
      const ses = new AWS.SES();
      await ses.sendEmail({
        Source: 'trials@bayer.com',
        Destination: { ToAddresses: [context.patientEmail] },
        Message: {
          Subject: { Data: 'Welcome to Clinical Trial' },
          Body: { Text: { Data: 'You are enrolled!' } }
        }
      }).promise();
    },
    compensate: async (context) => {
      // Send cancellation email
      const ses = new AWS.SES();
      await ses.sendEmail({
        Source: 'trials@bayer.com',
        Destination: { ToAddresses: [context.patientEmail] },
        Message: {
          Subject: { Data: 'Enrollment Cancelled' },
          Body: { Text: { Data: 'Sorry, enrollment failed.' } }
        }
      }).promise();
    }
  }
]);

// Execute saga
try {
  await enrollmentSaga.execute({
    trialId: 'TRIAL001',
    patientId: 'PAT123',
    patientEmail: 'patient@example.com'
  });
} catch (error) {
  console.error('Enrollment failed', { error: error.message });
}
```

---

## 📊 Distributed Systems Patterns Summary

| Pattern | Purpose | Use Case |
|---------|---------|----------|
| **Circuit Breaker** | Fail fast when service down | SAP integration |
| **Retry + Backoff** | Automatically retry transient failures | Network timeouts |
| **Idempotency** | Safe to retry multiple times | Payment processing |
| **Event Sourcing** | Store all changes as events | Audit trail |
| **CQRS** | Separate read/write models | High-performance queries |
| **Saga** | Distributed transactions | Multi-service workflows |

---

## 🎯 Key Takeaways

### ✅ Do's
1. **Implement circuit breakers** for external services
2. **Use exponential backoff** for retries
3. **Make operations idempotent** (safe to retry)
4. **Use saga pattern** for distributed transactions
5. **Monitor distributed tracing** with X-Ray
6. **Design for eventual consistency**
7. **Use event-driven architecture**
8. **Implement graceful degradation**

### ❌ Don'ts
1. **Don't assume network is reliable**
2. **Don't use distributed transactions** (2PC, XA)
3. **Don't retry immediately** (use backoff)
4. **Don't forget compensating actions**
5. **Don't ignore partial failures**

---

**Estimated Reading Time**: 17-20 minutes  
**Difficulty Level**: ⭐⭐⭐⭐⭐ Expert  
**Prerequisites**: Distributed systems, AWS services  
**Bayer Job Alignment**: 100% - Core requirement
