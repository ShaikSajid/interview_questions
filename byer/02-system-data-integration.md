# System & Data Integration - Interview Preparation

## Top 7 Interview Questions with Examples

### 1. Explain different integration patterns and when to use each

**Answer:**
Integration patterns define how systems communicate and exchange data.

**Example:**

```javascript
// 1. REQUEST-RESPONSE PATTERN (Synchronous)
// Use: Real-time data needed, low latency required
async function getUserProfile(userId) {
  const response = await fetch(`https://api.users.com/users/${userId}`);
  return await response.json();
}

// 2. PUBLISH-SUBSCRIBE PATTERN (Asynchronous)
// Use: One-to-many communication, decoupling systems
const AWS = require('aws-sdk');
const sns = new AWS.SNS();

async function publishUserCreatedEvent(user) {
  await sns.publish({
    TopicArn: 'arn:aws:sns:us-east-1:123456789:user-events',
    Message: JSON.stringify({
      eventType: 'USER_CREATED',
      userId: user.id,
      timestamp: new Date().toISOString()
    })
  }).promise();
}

// 3. MESSAGE QUEUE PATTERN (Asynchronous, Guaranteed Delivery)
// Use: Reliable processing, rate limiting, decoupling
const sqs = new AWS.SQS();

async function queueOrderProcessing(order) {
  await sqs.sendMessage({
    QueueUrl: 'https://sqs.us-east-1.amazonaws.com/123456789/orders',
    MessageBody: JSON.stringify(order),
    MessageAttributes: {
      priority: {
        DataType: 'String',
        StringValue: order.isPremium ? 'high' : 'normal'
      }
    }
  }).promise();
}

// 4. API GATEWAY PATTERN
// Use: Centralized entry point, authentication, rate limiting
const apiGateway = {
  authenticate: async (req, res, next) => {
    const token = req.headers.authorization;
    const user = await verifyToken(token);
    req.user = user;
    next();
  },
  
  rateLimit: rateLimit({
    windowMs: 60000,
    max: 100
  }),
  
  route: (req) => {
    const services = {
      '/users': 'user-service.internal',
      '/orders': 'order-service.internal',
      '/products': 'product-service.internal'
    };
    return services[req.path];
  }
};

// 5. EVENT SOURCING PATTERN
// Use: Audit trail, temporal queries, replay capability
class OrderEventStore {
  async appendEvent(orderId, event) {
    await dynamodb.putItem({
      TableName: 'order-events',
      Item: {
        orderId,
        eventId: uuid(),
        eventType: event.type,
        eventData: event.data,
        timestamp: Date.now()
      }
    });
  }
  
  async getOrderState(orderId) {
    const events = await this.getEvents(orderId);
    return events.reduce((state, event) => {
      return applyEvent(state, event);
    }, {});
  }
}
```

---

### 2. How do you handle data transformation between systems with different schemas?

**Answer:**
Data transformation ensures compatibility between different system formats.

**Example:**

```javascript
// Source System: Legacy CRM
const legacyCustomer = {
  cust_id: "12345",
  f_name: "John",
  l_name: "Doe",
  email_addr: "john@example.com",
  created_dt: "2024-01-15",
  status_cd: "A"
};

// Target System: Modern Customer Service
// Expected format:
// {
//   id: string,
//   fullName: string,
//   contactInfo: { email: string },
//   metadata: { createdAt: ISO8601, isActive: boolean }
// }

// Transformation Layer
class CustomerTransformer {
  static toModernFormat(legacyCustomer) {
    return {
      id: legacyCustomer.cust_id,
      fullName: `${legacyCustomer.f_name} ${legacyCustomer.l_name}`,
      contactInfo: {
        email: legacyCustomer.email_addr
      },
      metadata: {
        createdAt: new Date(legacyCustomer.created_dt).toISOString(),
        isActive: legacyCustomer.status_cd === 'A'
      }
    };
  }
  
  static toLegacyFormat(modernCustomer) {
    const [f_name, ...lastNameParts] = modernCustomer.fullName.split(' ');
    return {
      cust_id: modernCustomer.id,
      f_name: f_name,
      l_name: lastNameParts.join(' '),
      email_addr: modernCustomer.contactInfo.email,
      created_dt: modernCustomer.metadata.createdAt.split('T')[0],
      status_cd: modernCustomer.metadata.isActive ? 'A' : 'I'
    };
  }
}

// Advanced: Schema-based transformation with validation
const Ajv = require('ajv');
const ajv = new Ajv();

const legacySchema = {
  type: 'object',
  required: ['cust_id', 'f_name', 'l_name', 'email_addr'],
  properties: {
    cust_id: { type: 'string' },
    f_name: { type: 'string' },
    l_name: { type: 'string' },
    email_addr: { type: 'string', format: 'email' }
  }
};

const modernSchema = {
  type: 'object',
  required: ['id', 'fullName', 'contactInfo'],
  properties: {
    id: { type: 'string' },
    fullName: { type: 'string' },
    contactInfo: {
      type: 'object',
      properties: {
        email: { type: 'string', format: 'email' }
      }
    }
  }
};

class ValidatingTransformer {
  constructor() {
    this.validateLegacy = ajv.compile(legacySchema);
    this.validateModern = ajv.compile(modernSchema);
  }
  
  transform(legacyData) {
    // Validate input
    if (!this.validateLegacy(legacyData)) {
      throw new Error(`Invalid legacy format: ${JSON.stringify(this.validateLegacy.errors)}`);
    }
    
    // Transform
    const modernData = CustomerTransformer.toModernFormat(legacyData);
    
    // Validate output
    if (!this.validateModern(modernData)) {
      throw new Error(`Invalid modern format: ${JSON.stringify(this.validateModern.errors)}`);
    }
    
    return modernData;
  }
}
```

---

### 3. How do you ensure data consistency across distributed systems?

**Answer:**
Maintaining consistency in distributed systems requires careful design patterns.

**Example:**

```javascript
// 1. TWO-PHASE COMMIT (Strong Consistency)
class TwoPhaseCommitCoordinator {
  async executeTransaction(operations) {
    const transactionId = uuid();
    
    // Phase 1: Prepare
    const prepareResults = await Promise.all(
      operations.map(op => op.prepare(transactionId))
    );
    
    if (prepareResults.every(result => result.success)) {
      // Phase 2: Commit
      await Promise.all(
        operations.map(op => op.commit(transactionId))
      );
      return { success: true };
    } else {
      // Rollback
      await Promise.all(
        operations.map(op => op.rollback(transactionId))
      );
      return { success: false, error: 'Transaction aborted' };
    }
  }
}

// 2. SAGA PATTERN (Eventual Consistency)
class OrderSaga {
  async createOrder(orderData) {
    const sagaId = uuid();
    
    try {
      // Step 1: Reserve inventory
      const inventory = await this.reserveInventory(orderData, sagaId);
      
      // Step 2: Process payment
      const payment = await this.processPayment(orderData, sagaId);
      
      // Step 3: Create shipment
      const shipment = await this.createShipment(orderData, sagaId);
      
      return { success: true, orderId: sagaId };
      
    } catch (error) {
      // Compensating transactions (reverse order)
      await this.compensate(sagaId, error.step);
      throw error;
    }
  }
  
  async compensate(sagaId, failedStep) {
    if (failedStep >= 3) {
      await this.cancelShipment(sagaId);
    }
    if (failedStep >= 2) {
      await this.refundPayment(sagaId);
    }
    if (failedStep >= 1) {
      await this.releaseInventory(sagaId);
    }
  }
}

// 3. IDEMPOTENCY (Prevent duplicate processing)
class IdempotentOrderProcessor {
  constructor() {
    this.processedOrders = new Map(); // In production: use Redis/DynamoDB
  }
  
  async processOrder(orderId, orderData) {
    // Check if already processed
    const existing = this.processedOrders.get(orderId);
    if (existing) {
      console.log(`Order ${orderId} already processed`);
      return existing.result;
    }
    
    // Process order
    const result = await this.executeOrderProcessing(orderData);
    
    // Store result
    this.processedOrders.set(orderId, {
      result,
      timestamp: Date.now()
    });
    
    return result;
  }
}

// 4. OPTIMISTIC LOCKING (Detect conflicts)
class OptimisticLockingService {
  async updateProduct(productId, updates) {
    const product = await this.getProduct(productId);
    const currentVersion = product.version;
    
    const result = await dynamodb.updateItem({
      TableName: 'products',
      Key: { id: productId },
      UpdateExpression: 'SET price = :price, version = :newVersion',
      ConditionExpression: 'version = :currentVersion',
      ExpressionAttributeValues: {
        ':price': updates.price,
        ':currentVersion': currentVersion,
        ':newVersion': currentVersion + 1
      }
    });
    
    // If condition fails, version mismatch detected
    return result;
  }
}
```

---

### 4. How do you implement data validation in integration pipelines?

**Answer:**
Multi-layer validation ensures data quality and prevents errors.

**Example:**

```javascript
// Validation Framework
class DataValidator {
  constructor() {
    this.rules = [];
  }
  
  addRule(rule) {
    this.rules.push(rule);
    return this;
  }
  
  async validate(data) {
    const errors = [];
    
    for (const rule of this.rules) {
      const result = await rule.validate(data);
      if (!result.valid) {
        errors.push({
          rule: rule.name,
          message: result.message,
          field: result.field
        });
      }
    }
    
    return {
      valid: errors.length === 0,
      errors
    };
  }
}

// Validation Rules
const requiredFieldRule = {
  name: 'RequiredField',
  validate: (data) => {
    const requiredFields = ['id', 'email', 'name'];
    for (const field of requiredFields) {
      if (!data[field]) {
        return {
          valid: false,
          message: `${field} is required`,
          field
        };
      }
    }
    return { valid: true };
  }
};

const emailFormatRule = {
  name: 'EmailFormat',
  validate: (data) => {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(data.email)) {
      return {
        valid: false,
        message: 'Invalid email format',
        field: 'email'
      };
    }
    return { valid: true };
  }
};

const businessRule = {
  name: 'AgeRestriction',
  validate: async (data) => {
    if (data.age < 18) {
      return {
        valid: false,
        message: 'User must be 18 or older',
        field: 'age'
      };
    }
    return { valid: true };
  }
};

// Usage in integration pipeline
class IntegrationPipeline {
  async processData(rawData) {
    // Stage 1: Schema validation
    const validator = new DataValidator()
      .addRule(requiredFieldRule)
      .addRule(emailFormatRule)
      .addRule(businessRule);
    
    const validation = await validator.validate(rawData);
    
    if (!validation.valid) {
      await this.logValidationErrors(validation.errors);
      throw new ValidationError('Data validation failed', validation.errors);
    }
    
    // Stage 2: Data enrichment
    const enrichedData = await this.enrichData(rawData);
    
    // Stage 3: Cross-system validation
    await this.validateAgainstExternalSystem(enrichedData);
    
    // Stage 4: Processing
    return await this.processValidData(enrichedData);
  }
  
  async validateAgainstExternalSystem(data) {
    // Check if user already exists
    const existingUser = await fetch(`https://api.users.com/check/${data.email}`);
    if (existingUser.exists) {
      throw new ValidationError('User already exists');
    }
  }
}
```

---

### 5. How do you handle real-time data synchronization between systems?

**Answer:**
Real-time sync requires event-driven architecture and conflict resolution.

**Example:**

```javascript
// Change Data Capture (CDC) Pattern
class DatabaseChangeStream {
  constructor(database) {
    this.database = database;
    this.subscribers = [];
  }
  
  // Monitor database changes
  async startWatching(collection) {
    const changeStream = this.database
      .collection(collection)
      .watch({ fullDocument: 'updateLookup' });
    
    changeStream.on('change', async (change) => {
      const event = {
        operation: change.operationType,
        documentId: change.documentKey._id,
        fullDocument: change.fullDocument,
        timestamp: new Date()
      };
      
      await this.notifySubscribers(event);
    });
  }
  
  subscribe(handler) {
    this.subscribers.push(handler);
  }
  
  async notifySubscribers(event) {
    await Promise.all(
      this.subscribers.map(handler => handler(event))
    );
  }
}

// Real-time Synchronization Service
class SyncService {
  constructor() {
    this.syncQueue = new AWS.SQS();
    this.conflictResolver = new ConflictResolver();
  }
  
  async syncToTargetSystem(event) {
    try {
      // Get current state from target
      const targetState = await this.getTargetState(event.documentId);
      
      // Detect conflicts
      if (this.hasConflict(event, targetState)) {
        const resolved = await this.conflictResolver.resolve(event, targetState);
        await this.applyChanges(resolved);
      } else {
        await this.applyChanges(event);
      }
      
      // Update sync metadata
      await this.updateSyncMetadata(event.documentId, {
        lastSynced: Date.now(),
        version: event.version
      });
      
    } catch (error) {
      // Queue for retry
      await this.queueForRetry(event, error);
    }
  }
}

// Conflict Resolution Strategies
class ConflictResolver {
  async resolve(sourceEvent, targetState) {
    // Strategy 1: Last Write Wins
    if (sourceEvent.timestamp > targetState.lastModified) {
      return sourceEvent.fullDocument;
    }
    
    // Strategy 2: Merge changes
    return this.mergeChanges(sourceEvent.fullDocument, targetState.data);
  }
  
  mergeChanges(source, target) {
    // Custom merge logic
    return {
      ...target,
      ...source,
      _conflictResolved: true,
      _mergedAt: Date.now()
    };
  }
}

// Bi-directional Sync
class BidirectionalSync {
  constructor(systemA, systemB) {
    this.systemA = systemA;
    this.systemB = systemB;
    this.syncLog = new Map();
  }
  
  async initialize() {
    // Watch changes in both systems
    this.systemA.onChange(change => this.syncAtoB(change));
    this.systemB.onChange(change => this.syncBtoA(change));
  }
  
  async syncAtoB(change) {
    // Prevent sync loops
    if (this.isFromSync(change)) return;
    
    this.markAsSync(change);
    await this.systemB.update(change);
  }
  
  async syncBtoA(change) {
    if (this.isFromSync(change)) return;
    
    this.markAsSync(change);
    await this.systemA.update(change);
  }
  
  isFromSync(change) {
    return this.syncLog.has(change.id) &&
           Date.now() - this.syncLog.get(change.id) < 5000;
  }
  
  markAsSync(change) {
    this.syncLog.set(change.id, Date.now());
  }
}
```

---

### 6. How do you implement retry logic and error handling in integrations?

**Answer:**
Robust error handling ensures reliability in distributed systems.

**Example:**

```javascript
// Exponential Backoff Retry
class RetryHandler {
  async executeWithRetry(operation, options = {}) {
    const {
      maxRetries = 3,
      initialDelay = 1000,
      maxDelay = 30000,
      backoffMultiplier = 2,
      retryableErrors = ['TIMEOUT', 'RATE_LIMIT', '503']
    } = options;
    
    let lastError;
    
    for (let attempt = 0; attempt <= maxRetries; attempt++) {
      try {
        return await operation();
      } catch (error) {
        lastError = error;
        
        // Check if error is retryable
        if (!this.isRetryable(error, retryableErrors)) {
          throw error;
        }
        
        // Check if we have more retries
        if (attempt === maxRetries) {
          throw new Error(`Max retries (${maxRetries}) exceeded: ${error.message}`);
        }
        
        // Calculate delay with exponential backoff
        const delay = Math.min(
          initialDelay * Math.pow(backoffMultiplier, attempt),
          maxDelay
        );
        
        // Add jitter to prevent thundering herd
        const jitter = Math.random() * 0.3 * delay;
        const finalDelay = delay + jitter;
        
        console.log(`Retry attempt ${attempt + 1} after ${finalDelay}ms`);
        await this.sleep(finalDelay);
      }
    }
    
    throw lastError;
  }
  
  isRetryable(error, retryableErrors) {
    return retryableErrors.some(code => 
      error.code === code || 
      error.message.includes(code)
    );
  }
  
  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Circuit Breaker Pattern
class CircuitBreaker {
  constructor(options = {}) {
    this.failureThreshold = options.failureThreshold || 5;
    this.resetTimeout = options.resetTimeout || 60000;
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.failures = 0;
    this.nextAttempt = Date.now();
  }
  
  async execute(operation) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN');
      }
      this.state = 'HALF_OPEN';
    }
    
    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  onSuccess() {
    this.failures = 0;
    this.state = 'CLOSED';
  }
  
  onFailure() {
    this.failures++;
    if (this.failures >= this.failureThreshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.resetTimeout;
      console.log(`Circuit breaker opened for ${this.resetTimeout}ms`);
    }
  }
}

// Dead Letter Queue (DLQ) Handler
class DLQHandler {
  constructor() {
    this.sqs = new AWS.SQS();
    this.mainQueueUrl = process.env.MAIN_QUEUE_URL;
    this.dlqUrl = process.env.DLQ_URL;
  }
  
  async processMessage(message) {
    try {
      await this.handleMessage(message);
      await this.deleteMessage(message);
    } catch (error) {
      const retryCount = message.attributes.ApproximateReceiveCount;
      
      if (retryCount >= 3) {
        // Move to DLQ
        await this.moveToDLQ(message, error);
        await this.deleteMessage(message);
      } else {
        // Let it return to queue for retry
        console.log(`Message will be retried (attempt ${retryCount})`);
      }
    }
  }
  
  async moveToDLQ(message, error) {
    await this.sqs.sendMessage({
      QueueUrl: this.dlqUrl,
      MessageBody: message.body,
      MessageAttributes: {
        originalMessageId: {
          DataType: 'String',
          StringValue: message.messageId
        },
        errorMessage: {
          DataType: 'String',
          StringValue: error.message
        },
        failedAt: {
          DataType: 'String',
          StringValue: new Date().toISOString()
        }
      }
    }).promise();
  }
}
```

---

### 7. How do you monitor and debug integration pipelines in production?

**Answer:**
Comprehensive monitoring and logging are essential for production systems.

**Example:**

```javascript
// Distributed Tracing
const AWSXRay = require('aws-xray-sdk-core');
const AWS = AWSXRay.captureAWS(require('aws-sdk'));

class TracedIntegrationService {
  async processOrder(orderId) {
    const segment = AWSXRay.getSegment();
    const subsegment = segment.addNewSubsegment('ProcessOrder');
    
    try {
      subsegment.addAnnotation('orderId', orderId);
      subsegment.addMetadata('orderData', { id: orderId });
      
      // Step 1: Fetch order
      await this.traceOperation('FetchOrder', async () => {
        return await this.fetchOrder(orderId);
      });
      
      // Step 2: Validate order
      await this.traceOperation('ValidateOrder', async () => {
        return await this.validateOrder(orderId);
      });
      
      // Step 3: Process payment
      await this.traceOperation('ProcessPayment', async () => {
        return await this.processPayment(orderId);
      });
      
      subsegment.close();
    } catch (error) {
      subsegment.addError(error);
      subsegment.close();
      throw error;
    }
  }
  
  async traceOperation(name, operation) {
    const subsegment = AWSXRay.getSegment().addNewSubsegment(name);
    try {
      const result = await operation();
      subsegment.close();
      return result;
    } catch (error) {
      subsegment.addError(error);
      subsegment.close();
      throw error;
    }
  }
}

// Structured Logging
class IntegrationLogger {
  log(level, message, context = {}) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      level,
      message,
      context: {
        ...context,
        requestId: context.requestId || this.generateRequestId(),
        service: 'integration-service',
        environment: process.env.NODE_ENV
      }
    };
    
    console.log(JSON.stringify(logEntry));
    
    // Send to CloudWatch
    this.sendToCloudWatch(logEntry);
  }
  
  info(message, context) {
    this.log('INFO', message, context);
  }
  
  error(message, error, context) {
    this.log('ERROR', message, {
      ...context,
      error: {
        message: error.message,
        stack: error.stack,
        code: error.code
      }
    });
  }
  
  async sendToCloudWatch(logEntry) {
    const cloudwatchlogs = new AWS.CloudWatchLogs();
    await cloudwatchlogs.putLogEvents({
      logGroupName: '/aws/integration-service',
      logStreamName: `${process.env.NODE_ENV}-${new Date().toISOString().split('T')[0]}`,
      logEvents: [{
        message: JSON.stringify(logEntry),
        timestamp: Date.now()
      }]
    }).promise();
  }
}

// Metrics and Monitoring
class IntegrationMetrics {
  constructor() {
    this.cloudwatch = new AWS.CloudWatch();
  }
  
  async recordMetric(metricName, value, unit = 'Count') {
    await this.cloudwatch.putMetricData({
      Namespace: 'IntegrationService',
      MetricData: [{
        MetricName: metricName,
        Value: value,
        Unit: unit,
        Timestamp: new Date(),
        Dimensions: [
          {
            Name: 'Environment',
            Value: process.env.NODE_ENV
          }
        ]
      }]
    }).promise();
  }
  
  async recordProcessingTime(operationName, startTime) {
    const duration = Date.now() - startTime;
    await this.recordMetric(`${operationName}Duration`, duration, 'Milliseconds');
  }
  
  async recordSuccess(operationName) {
    await this.recordMetric(`${operationName}Success`, 1);
  }
  
  async recordFailure(operationName, error) {
    await this.recordMetric(`${operationName}Failure`, 1);
    await this.recordMetric(`${operationName}Error_${error.code}`, 1);
  }
}

// Health Check Endpoint
class HealthCheck {
  async checkHealth() {
    const checks = {
      database: await this.checkDatabase(),
      queue: await this.checkQueue(),
      externalApi: await this.checkExternalAPI(),
      disk: await this.checkDiskSpace()
    };
    
    const isHealthy = Object.values(checks).every(check => check.healthy);
    
    return {
      status: isHealthy ? 'healthy' : 'unhealthy',
      timestamp: new Date().toISOString(),
      checks
    };
  }
  
  async checkDatabase() {
    try {
      await database.query('SELECT 1');
      return { healthy: true, latency: 5 };
    } catch (error) {
      return { healthy: false, error: error.message };
    }
  }
}
```

---

## Key Points to Emphasize:
- Experience with ETL pipelines
- Data quality and validation strategies
- Handling schema evolution
- Real-time vs batch processing trade-offs
- Monitoring and alerting strategies
- Disaster recovery and rollback procedures