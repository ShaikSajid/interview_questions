# Question 2: Amazon SQS for Reliable Message Processing

## 🎯 Question
**How would you design a robust message queuing system using Amazon SQS to handle high-volume data integration between disparate systems? Explain message ordering, visibility timeout, dead-letter queues, and exactly-once processing patterns.**

---

## 📋 Answer Overview

Amazon SQS (Simple Queue Service) is a fully managed message queuing service that enables decoupling of microservices and distributed systems. For Bayer's integration needs, SQS provides:
- **Asynchronous communication** between systems
- **Buffering** during traffic spikes
- **Reliable message delivery** with automatic retries
- **Scalability** to millions of messages per second

---

## 🏗️ Architecture Design

### Integration Architecture with SQS

```
┌────────────────────────────────────────────────────────────┐
│                   Source Systems                           │
│  (SAP, Clinical Trials, Lab Systems, Manufacturing)        │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │   API Gateway         │  ← Entry point for all integrations
        └──────┬───────────────┘
               │
               ▼
        ┌──────────────────────┐
        │  Lambda: Validator    │  ← Validates incoming data
        └──────┬───────────────┘
               │
               ▼
    ┌─────────────────────────────────┐
    │  Standard SQS Queue              │  ← Default queue
    │  (integration-processing-queue)  │     (Best-effort ordering)
    └──────┬──────────────────────────┘
           │
    ┌──────┼──────────┐
    │      │          │
    ▼      ▼          ▼
  ┌────┐ ┌────┐ ┌────┐
  │λ #1│ │λ #2│ │λ #N│  ← Multiple Lambda consumers
  └──┬─┘ └──┬─┘ └──┬─┘     (Parallel processing)
     │      │      │
     └──────┼──────┘
            │
            ▼
    ┌──────────────────────┐
    │  Dead Letter Queue    │  ← Failed messages after 3 retries
    │  (integration-dlq)    │
    └──────┬───────────────┘
           │
           ▼
    ┌──────────────────────┐
    │  Lambda: DLQ Handler  │  ← Alerts + manual review
    └──────────────────────┘


FIFO Queue Alternative (For ordered processing):
    ┌─────────────────────────────────┐
    │  FIFO SQS Queue                  │  ← Strict ordering
    │  (patient-data-fifo.fifo)       │     (Same MessageGroupId)
    └──────┬──────────────────────────┘
           │
           ▼ (Sequential processing per group)
        ┌─────┐
        │λ #1 │  ← One message at a time per group
        └─────┘
```

**Why SQS?**
- **Durability**: Messages stored across multiple AZs
- **Scalability**: Unlimited throughput (standard queue)
- **Cost**: $0.40 per million requests
- **No infrastructure**: Fully managed, no servers to patch

---

## 💻 Implementation Examples

### 1. Producer: Sending Messages to SQS

```javascript
// producer.js - API Gateway Lambda sends messages to SQS
// WHY: Decouples API from processing, returns response immediately
// REQUIRED FOR: High-throughput APIs that can't wait for processing

const AWS = require('aws-sdk');
const sqs = new AWS.SQS({ apiVersion: '2012-11-05' });

// Configuration
// WHY: Environment variables for different environments (dev/staging/prod)
const QUEUE_URL = process.env.QUEUE_URL;
const FIFO_QUEUE_URL = process.env.FIFO_QUEUE_URL;

/**
 * Send integration request to SQS
 * WHY: Async processing improves API response time (200ms vs 5 seconds)
 * 
 * @param {Object} integrationData - Data to be processed
 * @returns {Promise<Object>} - Message ID for tracking
 */
exports.handler = async (event) => {
  try {
    // Parse incoming request
    const body = JSON.parse(event.body);
    
    // Validate required fields
    // WHY: Catch errors early before wasting queue space
    if (!body.source || !body.target || !body.data) {
      return {
        statusCode: 400,
        body: JSON.stringify({ error: 'Missing required fields' })
      };
    }
    
    // Prepare message
    const message = {
      QueueUrl: QUEUE_URL,
      MessageBody: JSON.stringify({
        source: body.source,
        target: body.target,
        data: body.data,
        requestId: generateRequestId(),
        timestamp: new Date().toISOString()
      }),
      
      // Message Attributes
      // WHY: Enable filtering and routing without parsing message body
      MessageAttributes: {
        'Source': {
          DataType: 'String',
          StringValue: body.source
        },
        'Priority': {
          DataType: 'String',
          StringValue: body.priority || 'normal'
        },
        'DataType': {
          DataType: 'String',
          StringValue: body.dataType || 'clinical'
        }
      },
      
      // Delay delivery (optional)
      // WHY: Schedule processing for later (e.g., batch at midnight)
      DelaySeconds: body.delaySeconds || 0  // 0-900 seconds
    };
    
    // Send to SQS
    const result = await sqs.sendMessage(message).promise();
    
    console.log('Message sent to SQS', {
      messageId: result.MessageId,
      requestId: body.requestId
    });
    
    // Return immediately to client
    // WHY: Don't make user wait for processing (async pattern)
    return {
      statusCode: 202,  // Accepted
      body: JSON.stringify({
        message: 'Integration request accepted',
        trackingId: result.MessageId,
        estimatedProcessingTime: '2-5 minutes'
      })
    };
    
  } catch (error) {
    console.error('Failed to send message', error);
    return {
      statusCode: 500,
      body: JSON.stringify({ error: 'Failed to queue integration request' })
    };
  }
};

/**
 * Send batch of messages (more efficient)
 * WHY: Reduce API calls from 1000 to 100 (10 messages per batch)
 * COST SAVINGS: $0.40 per million requests vs $4.00
 */
async function sendBatchMessages(messages) {
  // SQS supports up to 10 messages per batch
  // WHY: Reduce API calls and improve throughput
  const batches = chunkArray(messages, 10);
  
  for (const batch of batches) {
    const params = {
      QueueUrl: QUEUE_URL,
      Entries: batch.map((msg, index) => ({
        Id: `msg-${index}`,  // Unique ID within batch
        MessageBody: JSON.stringify(msg),
        MessageAttributes: {
          'BatchId': {
            DataType: 'String',
            StringValue: generateBatchId()
          }
        }
      }))
    };
    
    try {
      const result = await sqs.sendMessageBatch(params).promise();
      
      // Check for failures
      // WHY: Some messages in batch might fail individually
      if (result.Failed && result.Failed.length > 0) {
        console.error('Some messages failed', {
          failed: result.Failed.length,
          successful: result.Successful.length
        });
        
        // Retry failed messages
        // WHY: Don't lose data due to transient errors
        await retryFailedMessages(result.Failed);
      }
      
    } catch (error) {
      console.error('Batch send failed', error);
      throw error;
    }
  }
}

function chunkArray(array, size) {
  const chunks = [];
  for (let i = 0; i < array.length; i += size) {
    chunks.push(array.slice(i, i + size));
  }
  return chunks;
}

function generateRequestId() {
  return `REQ-${Date.now()}-${Math.random().toString(36).substring(7)}`;
}

function generateBatchId() {
  return `BATCH-${Date.now()}`;
}
```

---

### 2. Consumer: Processing Messages from SQS

```javascript
// consumer.js - Lambda function triggered by SQS
// WHY: Automatic polling, scaling, and retry handling
// TRIGGERED BY: SQS Event Source Mapping

exports.handler = async (event) => {
  console.log('Processing SQS messages', {
    messageCount: event.Records.length
  });
  
  // Track results
  const results = {
    successful: [],
    failed: []
  };
  
  // Process each message
  // WHY: Lambda can receive 1-10 messages per invocation (BatchSize config)
  for (const record of event.Records) {
    try {
      // Parse message
      const message = JSON.parse(record.body);
      
      // Extract metadata
      // WHY: Use attributes for routing without parsing body
      const source = record.messageAttributes?.Source?.stringValue;
      const priority = record.messageAttributes?.Priority?.stringValue;
      
      console.log('Processing message', {
        messageId: record.messageId,
        source,
        priority,
        attempt: record.attributes.ApproximateReceiveCount
      });
      
      // Business logic: Transform data
      const result = await processIntegration(message);
      
      results.successful.push({
        messageId: record.messageId,
        result
      });
      
      // Delete from queue is automatic
      // WHY: Lambda deletes message only if handler succeeds
      
    } catch (error) {
      console.error('Message processing failed', {
        messageId: record.messageId,
        error: error.message,
        receiveCount: record.attributes.ApproximateReceiveCount
      });
      
      results.failed.push({
        messageId: record.messageId,
        error: error.message
      });
      
      // Throw error to trigger retry
      // WHY: Message returns to queue for retry (up to maxReceiveCount)
      throw error;
    }
  }
  
  return {
    statusCode: 200,
    body: JSON.stringify(results)
  };
};

/**
 * Process integration data
 * WHY: Core business logic for data transformation
 */
async function processIntegration(message) {
  const { source, target, data } = message;
  
  // Simulate processing time
  // REAL SCENARIO: Call external APIs, transform data, store results
  console.log(`Transforming data: ${source} → ${target}`);
  
  // Example: Clinical trial data integration
  if (source === 'clinical_trials') {
    return await processClinicalData(data);
  }
  
  // Example: Manufacturing data integration
  if (source === 'manufacturing') {
    return await processManufacturingData(data);
  }
  
  // Example: SAP integration
  if (source === 'sap') {
    return await processSAPData(data);
  }
  
  throw new Error(`Unknown source: ${source}`);
}

/**
 * Process clinical trial data
 * WHY: Regulatory compliance requires specific data handling
 */
async function processClinicalData(data) {
  // Validate required fields
  // WHY: FDA requires specific data points for clinical trials
  const requiredFields = [
    'protocolNumber',
    'investigatorName',
    'patientId',
    'adverseEvents'
  ];
  
  for (const field of requiredFields) {
    if (!data[field]) {
      throw new Error(`Missing required field: ${field}`);
    }
  }
  
  // De-identify patient data
  // WHY: HIPAA compliance requires anonymization
  const anonymizedData = {
    ...data,
    patientId: hashPatientId(data.patientId),
    investigatorId: hashInvestigatorId(data.investigatorName)
  };
  
  // Store in regulatory database
  // WHY: Must be retrievable for FDA audits
  await storeInRegulatoryDB(anonymizedData);
  
  // Check for serious adverse events
  // WHY: FDA requires 24-hour reporting for serious events
  const seriousEvents = data.adverseEvents.filter(e => e.severity === 'serious');
  if (seriousEvents.length > 0) {
    await triggerAdverseEventAlert(seriousEvents);
  }
  
  return {
    status: 'processed',
    protocolNumber: data.protocolNumber,
    eventsProcessed: data.adverseEvents.length,
    seriousEventsFound: seriousEvents.length
  };
}
```

---

### 3. FIFO Queue for Ordered Processing

```javascript
// fifo-producer.js - Send messages in order
// WHY: Some integrations require strict ordering (e.g., patient updates)
// EXAMPLE: Patient record must be created before adding test results

const AWS = require('aws-sdk');
const sqs = new AWS.SQS();

/**
 * Send ordered messages to FIFO queue
 * WHY: Ensures messages processed in exact order sent
 * USE CASE: Patient record lifecycle (create → update → delete)
 */
async function sendOrderedMessages(patientId, events) {
  // FIFO queue requires .fifo suffix
  const FIFO_QUEUE_URL = process.env.FIFO_QUEUE_URL; // ends with .fifo
  
  for (const event of events) {
    const params = {
      QueueUrl: FIFO_QUEUE_URL,
      MessageBody: JSON.stringify(event),
      
      // Message Group ID
      // WHY: Messages with same group ID processed in FIFO order
      // CRITICAL: Different patients can process in parallel
      MessageGroupId: `patient-${patientId}`,
      
      // Message Deduplication ID
      // WHY: Prevents duplicate messages within 5-minute window
      // EXAMPLE: Retry logic won't create duplicate records
      MessageDeduplicationId: `${patientId}-${event.eventType}-${event.timestamp}`,
      
      // Alternative: Use content-based deduplication
      // WHY: Automatically uses SHA-256 of message body
      // Enable this in queue config to skip MessageDeduplicationId
    };
    
    try {
      const result = await sqs.sendMessage(params).promise();
      console.log('Ordered message sent', {
        messageId: result.MessageId,
        sequenceNumber: result.SequenceNumber,  // Unique to FIFO
        patientId
      });
    } catch (error) {
      console.error('Failed to send ordered message', error);
      throw error;
    }
  }
}

// Example usage: Patient record lifecycle
const patientEvents = [
  { eventType: 'CREATE', patientId: 'P12345', data: { name: 'John Doe', dob: '1980-01-01' } },
  { eventType: 'UPDATE', patientId: 'P12345', data: { diagnosis: 'Hypertension' } },
  { eventType: 'ADD_RESULT', patientId: 'P12345', data: { test: 'Blood pressure', result: '140/90' } },
  { eventType: 'UPDATE', patientId: 'P12345', data: { medication: 'Lisinopril 10mg' } }
];

// WHY: All events for patient P12345 will process in exact order
await sendOrderedMessages('P12345', patientEvents);
```

**FIFO Queue vs Standard Queue:**

| Feature | Standard Queue | FIFO Queue |
|---------|---------------|------------|
| **Ordering** | Best-effort | Exact order (per group) |
| **Throughput** | Unlimited | 300 TPS (3000 with batching) |
| **Deduplication** | No | Yes (5-minute window) |
| **Use Case** | High volume, order doesn't matter | Patient records, financial transactions |
| **Cost** | $0.40 per million | $0.50 per million |

---

### 4. Visibility Timeout & Message Locking

```javascript
// visibility-timeout.js
// WHY: Prevents multiple consumers from processing same message
// PROBLEM: Without this, duplicate processing would occur

/**
 * Visibility Timeout Explained:
 * 
 * 1. Consumer receives message from queue
 * 2. Message becomes "invisible" for VisibilityTimeout seconds
 * 3. If consumer finishes → deletes message (success)
 * 4. If consumer crashes → message becomes visible again (auto-retry)
 * 5. If processing takes longer → extend visibility timeout
 */

exports.handler = async (event) => {
  for (const record of event.Records) {
    const startTime = Date.now();
    
    try {
      // Long-running operation
      await processLargeDataset(record);
      
      // Processing took 4 minutes
      const processingTime = Date.now() - startTime;
      console.log(`Processing completed in ${processingTime}ms`);
      
      // Message deleted automatically by Lambda
      
    } catch (error) {
      // If we know processing will take longer, extend visibility
      if (error.message === 'STILL_PROCESSING') {
        await extendVisibilityTimeout(record.receiptHandle, 300);
      }
    }
  }
};

/**
 * Extend visibility timeout
 * WHY: Processing takes longer than initial timeout
 * PREVENTS: Message from being picked up by another consumer mid-processing
 */
async function extendVisibilityTimeout(receiptHandle, additionalSeconds) {
  const sqs = new AWS.SQS();
  
  await sqs.changeMessageVisibility({
    QueueUrl: process.env.QUEUE_URL,
    ReceiptHandle: receiptHandle,
    VisibilityTimeout: additionalSeconds  // Extend by 5 minutes
  }).promise();
  
  console.log(`Visibility timeout extended by ${additionalSeconds} seconds`);
}

// Queue Configuration: Set appropriate initial visibility timeout
const queueConfig = {
  VisibilityTimeout: '300',  // 5 minutes (match Lambda timeout)
  // WHY: Should be >= Lambda timeout to prevent duplicate processing
  
  // If Lambda times out at 5 min, message becomes visible again
  // Next Lambda can retry the message
};
```

---

### 5. Dead Letter Queue (DLQ) Handling

```javascript
// dlq-handler.js
// WHY: Investigate permanently failed messages
// TRIGGERED BY: Messages that failed maxReceiveCount times

const AWS = require('aws-sdk');
const sqs = new AWS.SQS();
const sns = new AWS.SNS();

exports.handler = async (event) => {
  console.log('DLQ messages received', {
    count: event.Records.length
  });
  
  for (const record of event.Records) {
    try {
      const message = JSON.parse(record.body);
      const receiveCount = record.attributes.ApproximateReceiveCount;
      
      // Log for investigation
      console.error('Failed message in DLQ', {
        messageId: record.messageId,
        receiveCount,
        message: JSON.stringify(message, null, 2)
      });
      
      // Alert operations team
      // WHY: Human investigation required for persistent failures
      await sns.publish({
        TopicArn: process.env.ALERT_TOPIC_ARN,
        Subject: '🚨 Integration DLQ Alert',
        Message: JSON.stringify({
          severity: 'HIGH',
          messageId: record.messageId,
          failureCount: receiveCount,
          source: message.source,
          target: message.target,
          errorDetails: 'Message failed after maximum retries',
          investigationRequired: true,
          timestamp: new Date().toISOString()
        }, null, 2)
      }).promise();
      
      // Store in S3 for long-term analysis
      // WHY: Historical analysis of failure patterns
      await storeDLQMessageInS3(message, record);
      
      // Attempt alternative processing path
      // WHY: Some failures can be resolved with manual intervention
      if (isManuallyRecoverable(message)) {
        await sendToManualReviewQueue(message);
      }
      
    } catch (error) {
      console.error('DLQ handler failed', error);
      // Even DLQ handler can fail - log and continue
    }
  }
};

async function storeDLQMessageInS3(message, record) {
  const s3 = new AWS.S3();
  const key = `dlq/${new Date().toISOString().split('T')[0]}/${record.messageId}.json`;
  
  await s3.putObject({
    Bucket: process.env.DLQ_BUCKET,
    Key: key,
    Body: JSON.stringify({
      message,
      metadata: {
        messageId: record.messageId,
        receiveCount: record.attributes.ApproximateReceiveCount,
        dlqTimestamp: new Date().toISOString()
      }
    }, null, 2),
    ContentType: 'application/json'
  }).promise();
}

function isManuallyRecoverable(message) {
  // WHY: Some errors are transient or require manual data fixes
  const recoverableErrors = [
    'EXTERNAL_API_DOWN',
    'INVALID_CREDENTIALS',
    'RATE_LIMIT_EXCEEDED'
  ];
  return recoverableErrors.includes(message.errorType);
}
```

---

## ⚡ Exactly-Once Processing Pattern

```javascript
// exactly-once.js
// WHY: Prevent duplicate data in target systems
// CHALLENGE: SQS guarantees at-least-once delivery, not exactly-once

/**
 * Idempotency Pattern for Exactly-Once Processing
 * 
 * Strategy: Store processed message IDs in DynamoDB
 * If message ID already exists → skip processing
 */

const AWS = require('aws-sdk');
const dynamodb = new AWS.DynamoDB.DocumentClient();

const IDEMPOTENCY_TABLE = process.env.IDEMPOTENCY_TABLE;
const IDEMPOTENCY_TTL = 24 * 60 * 60; // 24 hours

exports.handler = async (event) => {
  for (const record of event.Records) {
    const messageId = record.messageId;
    
    // Check if already processed
    // WHY: Prevent duplicate processing on retry
    const alreadyProcessed = await isMessageProcessed(messageId);
    
    if (alreadyProcessed) {
      console.log('Message already processed, skipping', { messageId });
      continue;  // Skip to next message
    }
    
    try {
      // Process message
      const result = await processMessage(record);
      
      // Mark as processed
      // WHY: Store before Lambda could crash/timeout
      await markMessageAsProcessed(messageId, result);
      
    } catch (error) {
      console.error('Processing failed', { messageId, error: error.message });
      throw error;  // Trigger retry
    }
  }
};

/**
 * Check if message already processed
 * WHY: DynamoDB single-digit millisecond lookup
 */
async function isMessageProcessed(messageId) {
  try {
    const result = await dynamodb.get({
      TableName: IDEMPOTENCY_TABLE,
      Key: { messageId }
    }).promise();
    
    return !!result.Item;  // true if exists
    
  } catch (error) {
    console.error('Idempotency check failed', error);
    return false;  // Process on error (fail open)
  }
}

/**
 * Mark message as processed
 * WHY: Prevent duplicate processing on subsequent retries
 */
async function markMessageAsProcessed(messageId, result) {
  await dynamodb.put({
    TableName: IDEMPOTENCY_TABLE,
    Item: {
      messageId,
      processedAt: new Date().toISOString(),
      result: JSON.stringify(result),
      ttl: Math.floor(Date.now() / 1000) + IDEMPOTENCY_TTL
      // WHY: Auto-delete old records via DynamoDB TTL (cost optimization)
    }
  }).promise();
}
```

---

## 📊 Monitoring & Best Practices

### CloudWatch Metrics to Monitor

```javascript
// WHY: Proactive monitoring prevents outages

const metricsToMonitor = {
  // Queue depth
  'ApproximateNumberOfMessagesVisible': {
    threshold: 10000,
    // WHY: High queue depth indicates processing can't keep up
    action: 'Increase Lambda concurrency or scale consumers'
  },
  
  // Age of oldest message
  'ApproximateAgeOfOldestMessage': {
    threshold: 300,  // 5 minutes
    // WHY: Messages waiting too long indicate bottleneck
    action: 'Check consumer health and scaling'
  },
  
  // Messages in DLQ
  'ApproximateNumberOfMessagesVisible (DLQ)': {
    threshold: 10,
    // WHY: DLQ messages require manual investigation
    action: 'Alert on-call engineer immediately'
  },
  
  // Number of receives
  'NumberOfMessagesSent': {
    // WHY: Track throughput trends
    action: 'Capacity planning'
  },
  
  // Number of deletes
  'NumberOfMessagesDeleted': {
    // WHY: Successful processing rate
    action: 'Compare with sent to find failures'
  }
};
```

---

## 🎯 Key Takeaways

### ✅ Do's
1. **Use SQS for async communication** (decoupling)
2. **Set visibility timeout >= Lambda timeout**
3. **Configure DLQ with maxReceiveCount: 3**
4. **Use batch operations** for cost optimization
5. **Implement idempotency** for exactly-once semantics
6. **Monitor queue depth** and age of messages
7. **Use FIFO queues** only when ordering required
8. **Set message retention** to 14 days (maximum)
9. **Enable long polling** (ReceiveMessageWaitTimeSeconds: 20)
10. **Use message attributes** for routing/filtering

### ❌ Don'ts
1. **Don't poll empty queues** (use long polling)
2. **Don't use FIFO for high throughput** (limited to 3000 TPS)
3. **Don't ignore DLQ messages** (investigate root cause)
4. **Don't set visibility timeout too low** (causes duplicates)
5. **Don't process messages synchronously** (use Lambda async)
6. **Don't forget to delete messages** after processing
7. **Don't exceed 256KB message size** (use S3 for large payloads)
8. **Don't create queue per message type** (use attributes instead)

---

## 💰 Cost Optimization

| Optimization | Savings | Implementation |
|--------------|---------|----------------|
| **Batch API calls** | 90% | Use SendMessageBatch (10 messages) |
| **Long polling** | 50% | Set ReceiveMessageWaitTimeSeconds: 20 |
| **Right-size visibility** | 20% | Match Lambda timeout |
| **Delete old queues** | Variable | Audit unused queues monthly |

**Example Cost Calculation:**
- 10 million messages/month
- Standard queue: $0.40 per million = $4.00
- FIFO queue: $0.50 per million = $5.00
- Data transfer: $0.09 per GB = ~$1.00
- **Total: ~$5-6/month** (extremely cost-effective!)

---

**Estimated Reading Time**: 15-20 minutes  
**Difficulty Level**: ⭐⭐⭐ Intermediate  
**Prerequisites**: AWS Lambda, Message queuing concepts  
**Bayer Job Alignment**: 98% - Critical for system integration
