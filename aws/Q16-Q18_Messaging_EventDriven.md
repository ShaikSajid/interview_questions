# AWS Interview Questions: Messaging & Event-Driven Architecture (Q16-Q18)

## Question 16: Explain Amazon SQS (Simple Queue Service) and messaging patterns

### 📋 Answer

**Amazon SQS** is a fully managed message queuing service that enables you to decouple and scale microservices, distributed systems, and serverless applications.

### SQS Queue Types:

| Feature | Standard Queue | FIFO Queue |
|---------|---------------|------------|
| **Ordering** | Best-effort ordering | Strict ordering |
| **Throughput** | Nearly unlimited | 300 TPS (batch: 3000 TPS) |
| **Delivery** | At-least-once | Exactly-once |
| **Deduplication** | No | Yes (5-minute window) |
| **Use Case** | High throughput | Order-critical apps |

### Key Concepts:

```
SQS Architecture
├── Producer (Sends messages)
├── Queue (Stores messages)
├── Consumer (Processes messages)
├── Visibility Timeout (Hide message during processing)
├── Dead Letter Queue (Failed messages)
└── Message Attributes (Metadata)
```

### Complete SQS Implementation:

```javascript
// sqs-manager.js
import {
  SQSClient,
  CreateQueueCommand,
  SendMessageCommand,
  SendMessageBatchCommand,
  ReceiveMessageCommand,
  DeleteMessageCommand,
  DeleteMessageBatchCommand,
  GetQueueAttributesCommand,
  PurgeQueueCommand,
  SetQueueAttributesCommand
} from '@aws-sdk/client-sqs';

const sqsClient = new SQSClient({ region: 'us-east-1' });

// 1. Create Standard Queue
async function createStandardQueue(queueName) {
  const command = new CreateQueueCommand({
    QueueName: queueName,
    Attributes: {
      'VisibilityTimeout': '30',  // seconds
      'MessageRetentionPeriod': '345600',  // 4 days (max: 14 days)
      'ReceiveMessageWaitTimeSeconds': '20',  // Long polling
      'DelaySeconds': '0',  // Delivery delay
      'MaximumMessageSize': '262144'  // 256 KB
    },
    tags: {
      'Environment': 'Production',
      'Application': 'OrderProcessing'
    }
  });
  
  try {
    const response = await sqsClient.send(command);
    console.log('Queue created:', response.QueueUrl);
    return response.QueueUrl;
  } catch (error) {
    console.error('Failed to create queue:', error);
    throw error;
  }
}

// 2. Create FIFO Queue
async function createFIFOQueue(queueName) {
  const command = new CreateQueueCommand({
    QueueName: `${queueName}.fifo`,
    Attributes: {
      'FifoQueue': 'true',
      'ContentBasedDeduplication': 'true',  // Auto-deduplication
      'VisibilityTimeout': '30',
      'MessageRetentionPeriod': '345600'
    }
  });
  
  const response = await sqsClient.send(command);
  console.log('FIFO Queue created:', response.QueueUrl);
  return response.QueueUrl;
}

// 3. Send Message
async function sendMessage(queueUrl, messageBody, attributes = {}) {
  const command = new SendMessageCommand({
    QueueUrl: queueUrl,
    MessageBody: JSON.stringify(messageBody),
    MessageAttributes: Object.entries(attributes).reduce((acc, [key, value]) => {
      acc[key] = {
        DataType: 'String',
        StringValue: String(value)
      };
      return acc;
    }, {}),
    DelaySeconds: 0
  });
  
  try {
    const response = await sqsClient.send(command);
    console.log('Message sent:', response.MessageId);
    return response;
  } catch (error) {
    console.error('Failed to send message:', error);
    throw error;
  }
}

// 4. Send Message to FIFO Queue
async function sendMessageFIFO(queueUrl, messageBody, groupId, deduplicationId) {
  const command = new SendMessageCommand({
    QueueUrl: queueUrl,
    MessageBody: JSON.stringify(messageBody),
    MessageGroupId: groupId,  // Required for FIFO
    MessageDeduplicationId: deduplicationId  // Optional if ContentBasedDeduplication enabled
  });
  
  const response = await sqsClient.send(command);
  console.log('FIFO message sent:', response.MessageId);
  return response;
}

// 5. Send Batch Messages
async function sendMessageBatch(queueUrl, messages) {
  const entries = messages.map((msg, index) => ({
    Id: `msg-${index}`,
    MessageBody: JSON.stringify(msg),
    DelaySeconds: 0
  }));
  
  const command = new SendMessageBatchCommand({
    QueueUrl: queueUrl,
    Entries: entries
  });
  
  try {
    const response = await sqsClient.send(command);
    console.log(`Sent ${response.Successful.length} messages`);
    
    if (response.Failed && response.Failed.length > 0) {
      console.error('Failed messages:', response.Failed);
    }
    
    return response;
  } catch (error) {
    console.error('Batch send failed:', error);
    throw error;
  }
}

// 6. Receive Messages
async function receiveMessages(queueUrl, maxMessages = 10) {
  const command = new ReceiveMessageCommand({
    QueueUrl: queueUrl,
    MaxNumberOfMessages: maxMessages,  // 1-10
    WaitTimeSeconds: 20,  // Long polling (0-20)
    MessageAttributeNames: ['All'],
    AttributeNames: ['All'],
    VisibilityTimeout: 30
  });
  
  try {
    const response = await sqsClient.send(command);
    const messages = response.Messages || [];
    
    console.log(`Received ${messages.length} messages`);
    
    return messages.map(msg => ({
      messageId: msg.MessageId,
      receiptHandle: msg.ReceiptHandle,
      body: JSON.parse(msg.Body),
      attributes: msg.MessageAttributes,
      sentTimestamp: msg.Attributes?.SentTimestamp
    }));
  } catch (error) {
    console.error('Failed to receive messages:', error);
    throw error;
  }
}

// 7. Delete Message (after processing)
async function deleteMessage(queueUrl, receiptHandle) {
  const command = new DeleteMessageCommand({
    QueueUrl: queueUrl,
    ReceiptHandle: receiptHandle
  });
  
  try {
    await sqsClient.send(command);
    console.log('Message deleted');
  } catch (error) {
    console.error('Failed to delete message:', error);
    throw error;
  }
}

// 8. Delete Batch Messages
async function deleteMessageBatch(queueUrl, receiptHandles) {
  const entries = receiptHandles.map((handle, index) => ({
    Id: `msg-${index}`,
    ReceiptHandle: handle
  }));
  
  const command = new DeleteMessageBatchCommand({
    QueueUrl: queueUrl,
    Entries: entries
  });
  
  const response = await sqsClient.send(command);
  console.log(`Deleted ${response.Successful.length} messages`);
  return response;
}

// 9. Get Queue Attributes
async function getQueueAttributes(queueUrl) {
  const command = new GetQueueAttributesCommand({
    QueueUrl: queueUrl,
    AttributeNames: ['All']
  });
  
  const response = await sqsClient.send(command);
  
  return {
    approximateMessages: response.Attributes.ApproximateNumberOfMessages,
    approximateNotVisible: response.Attributes.ApproximateNumberOfMessagesNotVisible,
    approximateDelayed: response.Attributes.ApproximateNumberOfMessagesDelayed,
    createdTimestamp: response.Attributes.CreatedTimestamp,
    queueArn: response.Attributes.QueueArn
  };
}

// 10. Setup Dead Letter Queue
async function setupDeadLetterQueue(mainQueueUrl, dlqArn, maxReceiveCount = 3) {
  const redrivePolicy = {
    deadLetterTargetArn: dlqArn,
    maxReceiveCount: maxReceiveCount
  };
  
  const command = new SetQueueAttributesCommand({
    QueueUrl: mainQueueUrl,
    Attributes: {
      'RedrivePolicy': JSON.stringify(redrivePolicy)
    }
  });
  
  await sqsClient.send(command);
  console.log('Dead letter queue configured');
}
```

### Message Processing Worker:

```javascript
// sqs-worker.js
class SQSWorker {
  constructor(queueUrl, handler, options = {}) {
    this.queueUrl = queueUrl;
    this.handler = handler;
    this.pollingInterval = options.pollingInterval || 1000;
    this.maxMessages = options.maxMessages || 10;
    this.visibilityTimeout = options.visibilityTimeout || 30;
    this.isRunning = false;
  }
  
  // Start polling for messages
  async start() {
    this.isRunning = true;
    console.log('Worker started, polling for messages...');
    
    while (this.isRunning) {
      try {
        await this.poll();
      } catch (error) {
        console.error('Polling error:', error);
        await this.sleep(5000);  // Wait 5s on error
      }
    }
  }
  
  // Stop worker
  stop() {
    this.isRunning = false;
    console.log('Worker stopped');
  }
  
  // Poll for messages
  async poll() {
    const messages = await receiveMessages(this.queueUrl, this.maxMessages);
    
    if (messages.length === 0) {
      return;
    }
    
    // Process messages in parallel
    const promises = messages.map(msg => this.processMessage(msg));
    await Promise.allSettled(promises);
  }
  
  // Process individual message
  async processMessage(message) {
    console.log('Processing message:', message.messageId);
    
    try {
      // Execute handler
      await this.handler(message.body, message);
      
      // Delete message on success
      await deleteMessage(this.queueUrl, message.receiptHandle);
      console.log('Message processed successfully:', message.messageId);
      
    } catch (error) {
      console.error('Failed to process message:', message.messageId, error);
      // Message will become visible again after visibility timeout
      // and will be retried or sent to DLQ
    }
  }
  
  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Usage
const orderHandler = async (orderData, message) => {
  console.log('Processing order:', orderData.orderId);
  
  // Simulate order processing
  await processOrder(orderData);
  await updateInventory(orderData);
  await sendConfirmation(orderData);
  
  console.log('Order completed:', orderData.orderId);
};

const worker = new SQSWorker(
  'https://sqs.us-east-1.amazonaws.com/123456789012/OrderQueue',
  orderHandler,
  {
    maxMessages: 10,
    visibilityTimeout: 60,
    pollingInterval: 1000
  }
);

// Start worker
worker.start();

// Graceful shutdown
process.on('SIGTERM', () => {
  console.log('Received SIGTERM, shutting down gracefully...');
  worker.stop();
});
```

### Producer-Consumer Pattern:

```javascript
// order-service.js - Producer
class OrderService {
  constructor(queueUrl) {
    this.queueUrl = queueUrl;
  }
  
  async createOrder(orderData) {
    // Save to database
    const order = await db.createOrder(orderData);
    
    // Send to queue for async processing
    await sendMessage(this.queueUrl, {
      orderId: order.id,
      customerId: order.customerId,
      items: order.items,
      totalAmount: order.totalAmount,
      timestamp: new Date().toISOString()
    }, {
      'orderType': order.type,
      'priority': order.priority
    });
    
    console.log('Order queued for processing:', order.id);
    
    return order;
  }
  
  async createOrderBatch(orders) {
    const messages = orders.map(order => ({
      orderId: order.id,
      customerId: order.customerId,
      items: order.items,
      totalAmount: order.totalAmount
    }));
    
    await sendMessageBatch(this.queueUrl, messages);
    console.log(`Queued ${orders.length} orders`);
  }
}

// fulfillment-service.js - Consumer
class FulfillmentService {
  async processOrder(orderData) {
    console.log('Fulfilling order:', orderData.orderId);
    
    try {
      // Check inventory
      const inventory = await this.checkInventory(orderData.items);
      
      if (!inventory.available) {
        throw new Error('Insufficient inventory');
      }
      
      // Reserve items
      await this.reserveItems(orderData.items);
      
      // Process payment
      await this.processPayment(orderData);
      
      // Create shipment
      await this.createShipment(orderData);
      
      // Send notifications
      await this.sendNotifications(orderData);
      
      console.log('Order fulfilled successfully:', orderData.orderId);
      
    } catch (error) {
      console.error('Order fulfillment failed:', orderData.orderId, error);
      
      // Send to DLQ or retry queue
      await this.handleFailure(orderData, error);
      
      throw error;
    }
  }
  
  async checkInventory(items) {
    // Implementation
    return { available: true };
  }
  
  async reserveItems(items) {
    // Implementation
  }
  
  async processPayment(orderData) {
    // Implementation
  }
  
  async createShipment(orderData) {
    // Implementation
  }
  
  async sendNotifications(orderData) {
    // Implementation
  }
  
  async handleFailure(orderData, error) {
    // Log to monitoring system, send to DLQ, etc.
  }
}
```

### Best Practices:

1. ✅ Use long polling (WaitTimeSeconds=20) to reduce API calls
2. ✅ Implement idempotent message processing
3. ✅ Configure dead letter queues
4. ✅ Use batch operations for efficiency
5. ✅ Set appropriate visibility timeout
6. ✅ Delete messages after successful processing
7. ✅ Use FIFO queues for ordering requirements
8. ✅ Monitor queue depth with CloudWatch
9. ✅ Implement exponential backoff for retries
10. ✅ Use message attributes for filtering

---

## Question 17: Explain Amazon SNS (Simple Notification Service) and pub/sub pattern

### 📋 Answer

**Amazon SNS** is a fully managed pub/sub messaging service for application-to-application (A2A) and application-to-person (A2P) communication.

### SNS Architecture:

```
SNS Topic
├── Publishers (Send messages)
├── Subscribers
│   ├── SQS Queue
│   ├── Lambda Function
│   ├── HTTP/HTTPS Endpoint
│   ├── Email
│   ├── SMS
│   └── Mobile Push
└── Message Filtering
```

### Complete SNS Implementation:

```javascript
// sns-manager.js
import {
  SNSClient,
  CreateTopicCommand,
  SubscribeCommand,
  PublishCommand,
  PublishBatchCommand,
  SetSubscriptionAttributesCommand,
  ListSubscriptionsByTopicCommand,
  DeleteTopicCommand,
  UnsubscribeCommand
} from '@aws-sdk/client-sns';

const snsClient = new SNSClient({ region: 'us-east-1' });

// 1. Create Topic
async function createTopic(topicName, attributes = {}) {
  const command = new CreateTopicCommand({
    Name: topicName,
    Attributes: attributes,
    Tags: [
      { Key: 'Environment', Value: 'Production' },
      { Key: 'Application', Value: 'NotificationService' }
    ]
  });
  
  try {
    const response = await snsClient.send(command);
    console.log('Topic created:', response.TopicArn);
    return response.TopicArn;
  } catch (error) {
    console.error('Failed to create topic:', error);
    throw error;
  }
}

// 2. Create FIFO Topic
async function createFIFOTopic(topicName) {
  return await createTopic(`${topicName}.fifo`, {
    'FifoTopic': 'true',
    'ContentBasedDeduplication': 'true'
  });
}

// 3. Subscribe (SQS)
async function subscribeQueueToTopic(topicArn, queueArn) {
  const command = new SubscribeCommand({
    TopicArn: topicArn,
    Protocol: 'sqs',
    Endpoint: queueArn,
    ReturnSubscriptionArn: true
  });
  
  const response = await snsClient.send(command);
  console.log('Queue subscribed:', response.SubscriptionArn);
  return response.SubscriptionArn;
}

// 4. Subscribe (Lambda)
async function subscribeLambdaToTopic(topicArn, lambdaArn) {
  const command = new SubscribeCommand({
    TopicArn: topicArn,
    Protocol: 'lambda',
    Endpoint: lambdaArn,
    ReturnSubscriptionArn: true
  });
  
  const response = await snsClient.send(command);
  console.log('Lambda subscribed:', response.SubscriptionArn);
  return response.SubscriptionArn;
}

// 5. Subscribe (Email)
async function subscribeEmailToTopic(topicArn, email) {
  const command = new SubscribeCommand({
    TopicArn: topicArn,
    Protocol: 'email',
    Endpoint: email,
    ReturnSubscriptionArn: true
  });
  
  const response = await snsClient.send(command);
  console.log('Email subscription pending confirmation:', email);
  return response.SubscriptionArn;
}

// 6. Subscribe (HTTP/HTTPS)
async function subscribeHttpToTopic(topicArn, url) {
  const command = new SubscribeCommand({
    TopicArn: topicArn,
    Protocol: 'https',
    Endpoint: url,
    ReturnSubscriptionArn: true
  });
  
  const response = await snsClient.send(command);
  console.log('HTTP endpoint subscribed:', url);
  return response.SubscriptionArn;
}

// 7. Publish Message
async function publishMessage(topicArn, message, subject, attributes = {}) {
  const command = new PublishCommand({
    TopicArn: topicArn,
    Message: JSON.stringify(message),
    Subject: subject,
    MessageAttributes: Object.entries(attributes).reduce((acc, [key, value]) => {
      acc[key] = {
        DataType: 'String',
        StringValue: String(value)
      };
      return acc;
    }, {})
  });
  
  try {
    const response = await snsClient.send(command);
    console.log('Message published:', response.MessageId);
    return response;
  } catch (error) {
    console.error('Failed to publish message:', error);
    throw error;
  }
}

// 8. Publish with Message Structure (different content per protocol)
async function publishMessageWithStructure(topicArn, messages) {
  const messageStructure = {
    default: messages.default || 'Default message',
    email: messages.email || messages.default,
    sqs: JSON.stringify(messages.sqs || messages.default),
    lambda: JSON.stringify(messages.lambda || messages.default)
  };
  
  const command = new PublishCommand({
    TopicArn: topicArn,
    Message: JSON.stringify(messageStructure),
    MessageStructure: 'json'
  });
  
  const response = await snsClient.send(command);
  console.log('Structured message published:', response.MessageId);
  return response;
}

// 9. Publish Batch
async function publishBatch(topicArn, messages) {
  const entries = messages.map((msg, index) => ({
    Id: `msg-${index}`,
    Message: JSON.stringify(msg.body),
    Subject: msg.subject,
    MessageAttributes: msg.attributes
  }));
  
  const command = new PublishBatchCommand({
    TopicArn: topicArn,
    PublishBatchRequestEntries: entries
  });
  
  const response = await snsClient.send(command);
  console.log(`Published ${response.Successful.length} messages`);
  
  if (response.Failed && response.Failed.length > 0) {
    console.error('Failed messages:', response.Failed);
  }
  
  return response;
}

// 10. Set Subscription Filter Policy
async function setFilterPolicy(subscriptionArn, filterPolicy) {
  const command = new SetSubscriptionAttributesCommand({
    SubscriptionArn: subscriptionArn,
    AttributeName: 'FilterPolicy',
    AttributeValue: JSON.stringify(filterPolicy)
  });
  
  await snsClient.send(command);
  console.log('Filter policy set');
}

// 11. List Subscriptions
async function listSubscriptions(topicArn) {
  const command = new ListSubscriptionsByTopicCommand({
    TopicArn: topicArn
  });
  
  const response = await snsClient.send(command);
  
  return response.Subscriptions.map(sub => ({
    subscriptionArn: sub.SubscriptionArn,
    protocol: sub.Protocol,
    endpoint: sub.Endpoint
  }));
}
```

### Event-Driven Architecture Example:

```javascript
// event-publisher.js
class EventPublisher {
  constructor(topicArn) {
    this.topicArn = topicArn;
  }
  
  async publishEvent(eventType, eventData) {
    const event = {
      eventType: eventType,
      timestamp: new Date().toISOString(),
      data: eventData
    };
    
    await publishMessage(
      this.topicArn,
      event,
      `Event: ${eventType}`,
      {
        'eventType': eventType,
        'source': 'order-service',
        'priority': eventData.priority || 'normal'
      }
    );
    
    console.log(`Event published: ${eventType}`);
  }
  
  async publishOrderCreated(order) {
    await this.publishEvent('ORDER_CREATED', {
      orderId: order.id,
      customerId: order.customerId,
      totalAmount: order.totalAmount,
      items: order.items
    });
  }
  
  async publishOrderShipped(order) {
    await this.publishEvent('ORDER_SHIPPED', {
      orderId: order.id,
      trackingNumber: order.trackingNumber,
      shippedAt: new Date().toISOString()
    });
  }
  
  async publishPaymentProcessed(payment) {
    await this.publishEvent('PAYMENT_PROCESSED', {
      paymentId: payment.id,
      orderId: payment.orderId,
      amount: payment.amount,
      status: payment.status
    });
  }
}

// Lambda subscriber for email notifications
export const emailNotificationHandler = async (event) => {
  for (const record of event.Records) {
    const snsMessage = JSON.parse(record.Sns.Message);
    const eventType = record.Sns.MessageAttributes.eventType.Value;
    
    console.log('Processing event:', eventType);
    
    switch (eventType) {
      case 'ORDER_CREATED':
        await sendOrderConfirmationEmail(snsMessage.data);
        break;
      case 'ORDER_SHIPPED':
        await sendShippingNotificationEmail(snsMessage.data);
        break;
      case 'PAYMENT_PROCESSED':
        await sendPaymentReceiptEmail(snsMessage.data);
        break;
    }
  }
};

// Lambda subscriber for inventory updates
export const inventoryUpdateHandler = async (event) => {
  for (const record of event.Records) {
    const snsMessage = JSON.parse(record.Sns.Message);
    const eventType = record.Sns.MessageAttributes.eventType.Value;
    
    if (eventType === 'ORDER_CREATED') {
      await updateInventory(snsMessage.data);
    }
  }
};

// Lambda subscriber for analytics
export const analyticsHandler = async (event) => {
  for (const record of event.Records) {
    const snsMessage = JSON.parse(record.Sns.Message);
    await logEventToAnalytics(snsMessage);
  }
};
```

### Message Filtering:

```javascript
// Setup filtered subscriptions
async function setupFilteredSubscriptions(topicArn) {
  // Email notifications only for high-priority orders
  const emailSub = await subscribeEmailToTopic(topicArn, 'urgent@example.com');
  await setFilterPolicy(emailSub, {
    priority: ['high', 'urgent'],
    eventType: ['ORDER_CREATED']
  });
  
  // Analytics receives all events
  const analyticsSub = await subscribeLambdaToTopic(
    topicArn,
    'arn:aws:lambda:us-east-1:123456789012:function:AnalyticsHandler'
  );
  // No filter - receives everything
  
  // Inventory only for order events
  const inventorySub = await subscribeLambdaToTopic(
    topicArn,
    'arn:aws:lambda:us-east-1:123456789012:function:InventoryHandler'
  );
  await setFilterPolicy(inventorySub, {
    eventType: ['ORDER_CREATED', 'ORDER_CANCELLED']
  });
  
  console.log('Filtered subscriptions configured');
}
```

### Fan-Out Pattern (SNS + SQS):

```javascript
// Setup fan-out architecture
async function setupFanOut() {
  // Create SNS topic
  const topicArn = await createTopic('OrderEvents');
  
  // Create SQS queues for different services
  const emailQueueUrl = await createStandardQueue('EmailNotifications');
  const inventoryQueueUrl = await createStandardQueue('InventoryUpdates');
  const analyticsQueueUrl = await createStandardQueue('Analytics');
  
  // Get queue ARNs
  const emailQueueArn = await getQueueArn(emailQueueUrl);
  const inventoryQueueArn = await getQueueArn(inventoryQueueUrl);
  const analyticsQueueArn = await getQueueArn(analyticsQueueUrl);
  
  // Subscribe queues to topic
  await subscribeQueueToTopic(topicArn, emailQueueArn);
  await subscribeQueueToTopic(topicArn, inventoryQueueArn);
  await subscribeQueueToTopic(topicArn, analyticsQueueArn);
  
  console.log('Fan-out architecture configured');
  
  // Now when you publish to SNS, all queues receive the message
  await publishMessage(topicArn, {
    orderId: 'order-123',
    event: 'ORDER_CREATED'
  }, 'New Order');
}
```

### Best Practices:

1. ✅ Use message filtering to reduce unnecessary processing
2. ✅ Implement idempotent subscribers
3. ✅ Use dead letter queues for failed deliveries
4. ✅ Enable encryption at rest
5. ✅ Use message attributes for metadata
6. ✅ Monitor with CloudWatch metrics
7. ✅ Use FIFO topics for ordered messages
8. ✅ Implement retry logic in subscribers
9. ✅ Tag topics for cost tracking
10. ✅ Use SNS+SQS for reliable fan-out

---

## Question 18: Explain Amazon EventBridge for event-driven architectures

### 📋 Answer

**Amazon EventBridge** is a serverless event bus service that makes it easy to connect applications using events from AWS services, custom applications, and SaaS applications.

### EventBridge Components:

```
EventBridge
├── Event Bus (Default, Custom, Partner)
├── Rules (Pattern matching & routing)
├── Targets (Where events are sent)
│   ├── Lambda Functions
│   ├── Step Functions
│   ├── SQS Queues
│   ├── SNS Topics
│   └── 20+ AWS Services
├── Event Patterns (Filter events)
└── Scheduled Events (Cron expressions)
```

### Complete EventBridge Implementation:

```javascript
// eventbridge-manager.js
import {
  EventBridgeClient,
  CreateEventBusCommand,
  PutRuleCommand,
  PutTargetsCommand,
  PutEventsCommand,
  ListRulesCommand,
  DescribeRuleCommand,
  EnableRuleCommand,
  DisableRuleCommand,
  DeleteRuleCommand
} from '@aws-sdk/client-eventbridge';

const eventBridgeClient = new EventBridgeClient({ region: 'us-east-1' });

// 1. Create Custom Event Bus
async function createEventBus(name) {
  const command = new CreateEventBusCommand({
    Name: name,
    Tags: [
      { Key: 'Environment', Value: 'Production' }
    ]
  });
  
  try {
    const response = await eventBridgeClient.send(command);
    console.log('Event bus created:', response.EventBusArn);
    return response.EventBusArn;
  } catch (error) {
    console.error('Failed to create event bus:', error);
    throw error;
  }
}

// 2. Create Rule with Event Pattern
async function createEventRule(ruleName, eventBusName, eventPattern) {
  const command = new PutRuleCommand({
    Name: ruleName,
    EventBusName: eventBusName,
    Description: 'Route events to appropriate targets',
    State: 'ENABLED',
    EventPattern: JSON.stringify(eventPattern)
  });
  
  const response = await eventBridgeClient.send(command);
  console.log('Rule created:', response.RuleArn);
  return response.RuleArn;
}

// 3. Create Scheduled Rule (Cron)
async function createScheduledRule(ruleName, schedule) {
  const command = new PutRuleCommand({
    Name: ruleName,
    Description: 'Scheduled event rule',
    State: 'ENABLED',
    ScheduleExpression: schedule  // 'rate(5 minutes)' or 'cron(0 12 * * ? *)'
  });
  
  const response = await eventBridgeClient.send(command);
  console.log('Scheduled rule created:', response.RuleArn);
  return response.RuleArn;
}

// 4. Add Targets to Rule
async function addTargets(ruleName, targets) {
  const command = new PutTargetsCommand({
    Rule: ruleName,
    Targets: targets
  });
  
  try {
    const response = await eventBridgeClient.send(command);
    console.log('Targets added:', response.FailedEntryCount === 0);
    return response;
  } catch (error) {
    console.error('Failed to add targets:', error);
    throw error;
  }
}

// 5. Put Custom Events
async function putEvents(events) {
  const entries = events.map(event => ({
    Time: new Date(),
    Source: event.source,
    DetailType: event.detailType,
    Detail: JSON.stringify(event.detail),
    EventBusName: event.eventBusName || 'default',
    Resources: event.resources || []
  }));
  
  const command = new PutEventsCommand({
    Entries: entries
  });
  
  try {
    const response = await eventBridgeClient.send(command);
    
    if (response.FailedEntryCount > 0) {
      console.error('Failed entries:', response.Entries.filter(e => e.ErrorCode));
    } else {
      console.log(`Published ${entries.length} events`);
    }
    
    return response;
  } catch (error) {
    console.error('Failed to put events:', error);
    throw error;
  }
}

// 6. Enable/Disable Rule
async function toggleRule(ruleName, enable) {
  const command = enable 
    ? new EnableRuleCommand({ Name: ruleName })
    : new DisableRuleCommand({ Name: ruleName });
  
  await eventBridgeClient.send(command);
  console.log(`Rule ${enable ? 'enabled' : 'disabled'}:`, ruleName);
}

// 7. List Rules
async function listRules(eventBusName = 'default') {
  const command = new ListRulesCommand({
    EventBusName: eventBusName
  });
  
  const response = await eventBridgeClient.send(command);
  
  return response.Rules.map(rule => ({
    name: rule.Name,
    state: rule.State,
    description: rule.Description,
    eventPattern: rule.EventPattern
  }));
}
```

### Complete Event-Driven System:

```javascript
// event-driven-order-system.js

// 1. Setup EventBridge Infrastructure
async function setupOrderEventSystem() {
  // Create custom event bus
  const eventBusArn = await createEventBus('OrderProcessing');
  
  // Event pattern for order created
  const orderCreatedPattern = {
    source: ['order.service'],
    'detail-type': ['Order Created']
  };
  
  // Create rule for order created events
  const orderCreatedRule = await createEventRule(
    'OrderCreatedRule',
    'OrderProcessing',
    orderCreatedPattern
  );
  
  // Add targets: Lambda, SQS, Step Functions
  await addTargets('OrderCreatedRule', [
    {
      Id: '1',
      Arn: 'arn:aws:lambda:us-east-1:123456789012:function:ProcessOrder',
      RetryPolicy: {
        MaximumRetryAttempts: 2,
        MaximumEventAge: 3600
      }
    },
    {
      Id: '2',
      Arn: 'arn:aws:sqs:us-east-1:123456789012:OrderQueue'
    },
    {
      Id: '3',
      Arn: 'arn:aws:states:us-east-1:123456789012:stateMachine:OrderWorkflow',
      RoleArn: 'arn:aws:iam::123456789012:role/EventBridgeStepFunctionsRole'
    }
  ]);
  
  // Setup scheduled inventory check (every hour)
  const inventoryCheckRule = await createScheduledRule(
    'InventoryCheckSchedule',
    'rate(1 hour)'
  );
  
  await addTargets('InventoryCheckSchedule', [
    {
      Id: '1',
      Arn: 'arn:aws:lambda:us-east-1:123456789012:function:CheckInventory'
    }
  ]);
  
  console.log('Event-driven order system configured');
}

// 2. Event Publisher
class OrderEventPublisher {
  constructor(eventBusName = 'OrderProcessing') {
    this.eventBusName = eventBusName;
  }
  
  async publishOrderCreated(order) {
    await putEvents([
      {
        source: 'order.service',
        detailType: 'Order Created',
        detail: {
          orderId: order.id,
          customerId: order.customerId,
          items: order.items,
          totalAmount: order.totalAmount,
          status: 'PENDING',
          createdAt: new Date().toISOString()
        },
        eventBusName: this.eventBusName
      }
    ]);
    
    console.log('Order created event published:', order.id);
  }
  
  async publishOrderShipped(order) {
    await putEvents([
      {
        source: 'fulfillment.service',
        detailType: 'Order Shipped',
        detail: {
          orderId: order.id,
          trackingNumber: order.trackingNumber,
          carrier: order.carrier,
          shippedAt: new Date().toISOString()
        },
        eventBusName: this.eventBusName
      }
    ]);
  }
  
  async publishPaymentFailed(payment) {
    await putEvents([
      {
        source: 'payment.service',
        detailType: 'Payment Failed',
        detail: {
          orderId: payment.orderId,
          reason: payment.failureReason,
          amount: payment.amount,
          failedAt: new Date().toISOString()
        },
        eventBusName: this.eventBusName
      }
    ]);
  }
}

// 3. Lambda Event Handlers
export const orderCreatedHandler = async (event) => {
  console.log('Received OrderCreated event:', JSON.stringify(event, null, 2));
  
  const orderDetails = event.detail;
  
  // Process order
  await validateOrder(orderDetails);
  await checkInventory(orderDetails);
  await reserveItems(orderDetails);
  
  console.log('Order processing initiated:', orderDetails.orderId);
};

export const orderShippedHandler = async (event) => {
  console.log('Received OrderShipped event:', JSON.stringify(event, null, 2));
  
  const shipmentDetails = event.detail;
  
  // Send notification
  await sendShippingNotification(shipmentDetails);
  await updateOrderStatus(shipmentDetails.orderId, 'SHIPPED');
  
  console.log('Shipping notification sent:', shipmentDetails.orderId);
};

export const paymentFailedHandler = async (event) => {
  console.log('Received PaymentFailed event:', JSON.stringify(event, null, 2));
  
  const paymentDetails = event.detail;
  
  // Handle failure
  await cancelOrder(paymentDetails.orderId);
  await sendPaymentFailureEmail(paymentDetails);
  await logPaymentFailure(paymentDetails);
  
  console.log('Payment failure handled:', paymentDetails.orderId);
};
```

### Event Patterns Examples:

```javascript
// 1. Match specific source and detail-type
const basicPattern = {
  source: ['order.service'],
  'detail-type': ['Order Created']
};

// 2. Match multiple values
const multiplePattern = {
  source: ['order.service', 'inventory.service'],
  'detail-type': ['Order Created', 'Inventory Updated']
};

// 3. Match with prefix
const prefixPattern = {
  source: [{ prefix: 'order.' }],
  'detail-type': [{ prefix: 'Order' }]
};

// 4. Match nested fields
const nestedPattern = {
  source: ['order.service'],
  detail: {
    status: ['PENDING', 'PROCESSING'],
    totalAmount: [{ numeric: ['>', 100] }]
  }
};

// 5. Complex matching
const complexPattern = {
  source: ['order.service'],
  'detail-type': ['Order Created'],
  detail: {
    customerId: [{ exists: true }],
    items: {
      quantity: [{ numeric: ['>', 1] }]
    },
    priority: ['high', 'urgent']
  }
};

// 6. Match AWS service events
const ec2Pattern = {
  source: ['aws.ec2'],
  'detail-type': ['EC2 Instance State-change Notification'],
  detail: {
    state: ['terminated']
  }
};
```

### Best Practices:

1. ✅ Use custom event buses for application isolation
2. ✅ Implement detailed event patterns for precise routing
3. ✅ Add retry policies to targets
4. ✅ Use dead letter queues for failed events
5. ✅ Enable event archiving for compliance
6. ✅ Monitor rule metrics with CloudWatch
7. ✅ Use schema registry for event validation
8. ✅ Implement event versioning
9. ✅ Tag rules and buses for organization
10. ✅ Use EventBridge Pipes for transformations

---

## Key Takeaways

### Question 16 (SQS):
- Fully managed message queuing service
- Standard (high throughput) vs FIFO (ordered, exactly-once)
- Visibility timeout prevents duplicate processing
- Dead letter queues for failed messages
- Long polling reduces costs

### Question 17 (SNS):
- Pub/sub messaging for fan-out patterns
- Multiple protocol support (SQS, Lambda, Email, HTTP)
- Message filtering with filter policies
- SNS+SQS fan-out for reliable delivery
- FIFO topics for ordered notifications

### Question 18 (EventBridge):
- Serverless event bus for event-driven architectures
- Event pattern matching for routing
- 20+ AWS service integrations
- Scheduled events with cron expressions
- Schema registry for event validation
- Custom event buses for multi-tenant isolation
