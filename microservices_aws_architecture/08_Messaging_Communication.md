# Messaging & Communication Patterns

## Question 15: SQS/SNS Async Messaging Patterns

### 📋 Question Statement

Implement asynchronous messaging patterns for Emirates NBD using Amazon SQS and SNS. Include fan-out, dead letter queues, message deduplication, FIFO queues, and integration with Lambda for event processing.

---

### 🏗️ Async Messaging Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│               EMIRATES NBD ASYNC MESSAGING ARCHITECTURE                     │
└────────────────────────────────────────────────────────────────────────────┘

                          PATTERN 1: FAN-OUT
                          ──────────────────
                     
                     ┌──────────────────┐
                     │  Transaction     │
                     │  Service         │
                     └────────┬─────────┘
                              │
                              ▼
                     ┌──────────────────┐
                     │   SNS Topic      │
                     │  (Transaction    │
                     │   Events)        │
                     └────────┬─────────┘
                              │
                ┌─────────────┼─────────────┐
                │             │             │
                ▼             ▼             ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ SQS Queue│  │ SQS Queue│  │ SQS Queue│
        │ (Email)  │  │ (SMS)    │  │ (Fraud)  │
        └────┬─────┘  └────┬─────┘  └────┬─────┘
             │             │             │
             ▼             ▼             ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ Lambda   │  │ Lambda   │  │ Lambda   │
        │ (Email)  │  │ (SMS)    │  │ (Fraud)  │
        └──────────┘  └──────────┘  └──────────┘

                     PATTERN 2: WORK QUEUE
                     ─────────────────────
                     
        ┌──────────────────┐
        │  API Gateway     │
        └────────┬─────────┘
                 │
                 ▼
        ┌──────────────────┐
        │  SQS FIFO Queue  │
        │  (Payment Orders)│
        │  • Deduplication │
        │  • Ordering      │
        └────────┬─────────┘
                 │
        ┌────────┴────────┐
        │                 │
        ▼                 ▼
   ┌──────────┐      ┌──────────┐
   │ Lambda   │      │ Lambda   │
   │ Worker 1 │      │ Worker 2 │
   └────┬─────┘      └────┬─────┘
        │                 │
        └────────┬────────┘
                 │
                 ▼
        ┌──────────────────┐
        │  DynamoDB        │
        │  (Payments)      │
        └──────────────────┘

                     PATTERN 3: DLQ & RETRY
                     ──────────────────────
                     
        ┌──────────────────┐
        │  SQS Main Queue  │
        └────────┬─────────┘
                 │
                 ▼
        ┌──────────────────┐
        │  Lambda Consumer │
        │  (3 retries)     │
        └────────┬─────────┘
                 │
          ┌──────┴──────┐
          │ Failure     │
          ▼             │
   ┌──────────────┐     │
   │  Dead Letter │     │
   │  Queue       │     │
   └──────┬───────┘     │
          │             │
          ▼             │
   ┌──────────────┐     │
   │  Lambda      │     │
   │  (Alert)     │     │
   └──────────────┘     │
                        │
          ┌─────────────┘
          │ Success
          ▼
   ┌──────────────┐
   │  Processing  │
   │  Complete    │
   └──────────────┘
```

---

### 🔧 SQS/SNS Infrastructure (CDK)

```typescript
// infrastructure/cdk/messaging-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as sqs from 'aws-cdk-lib/aws-sqs';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as subscriptions from 'aws-cdk-lib/aws-sns-subscriptions';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as lambdaEventSources from 'aws-cdk-lib/aws-lambda-event-sources';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

export class MessagingStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // ============================================
    // 1. DEAD LETTER QUEUES
    // ============================================
    const emailDLQ = new sqs.Queue(this, 'EmailDLQ', {
      queueName: 'email-notifications-dlq',
      retentionPeriod: cdk.Duration.days(14)
    });

    const smsDLQ = new sqs.Queue(this, 'SmsDLQ', {
      queueName: 'sms-notifications-dlq',
      retentionPeriod: cdk.Duration.days(14)
    });

    const paymentDLQ = new sqs.Queue(this, 'PaymentDLQ', {
      queueName: 'payment-processing-dlq',
      retentionPeriod: cdk.Duration.days(14)
    });

    // ============================================
    // 2. STANDARD QUEUES
    // ============================================

    // Email notification queue
    const emailQueue = new sqs.Queue(this, 'EmailQueue', {
      queueName: 'email-notifications',
      visibilityTimeout: cdk.Duration.seconds(300),
      receiveMessageWaitTime: cdk.Duration.seconds(20), // Long polling
      deadLetterQueue: {
        queue: emailDLQ,
        maxReceiveCount: 3
      }
    });

    // SMS notification queue
    const smsQueue = new sqs.Queue(this, 'SmsQueue', {
      queueName: 'sms-notifications',
      visibilityTimeout: cdk.Duration.seconds(60),
      receiveMessageWaitTime: cdk.Duration.seconds(20),
      deadLetterQueue: {
        queue: smsDLQ,
        maxReceiveCount: 3
      }
    });

    // Fraud detection queue
    const fraudQueue = new sqs.Queue(this, 'FraudQueue', {
      queueName: 'fraud-detection',
      visibilityTimeout: cdk.Duration.seconds(120),
      receiveMessageWaitTime: cdk.Duration.seconds(20)
    });

    // ============================================
    // 3. FIFO QUEUES (for ordering)
    // ============================================

    const paymentFifoQueue = new sqs.Queue(this, 'PaymentFifoQueue', {
      queueName: 'payment-processing.fifo',
      fifo: true,
      contentBasedDeduplication: true,
      deduplicationScope: sqs.DeduplicationScope.MESSAGE_GROUP,
      fifoThroughputLimit: sqs.FifoThroughputLimit.PER_MESSAGE_GROUP_ID,
      visibilityTimeout: cdk.Duration.seconds(300),
      deadLetterQueue: {
        queue: paymentDLQ,
        maxReceiveCount: 3
      }
    });

    const accountUpdateFifoQueue = new sqs.Queue(this, 'AccountUpdateFifoQueue', {
      queueName: 'account-updates.fifo',
      fifo: true,
      contentBasedDeduplication: true,
      visibilityTimeout: cdk.Duration.seconds(180)
    });

    // ============================================
    // 4. SNS TOPICS
    // ============================================

    // Transaction events topic (fan-out)
    const transactionTopic = new sns.Topic(this, 'TransactionTopic', {
      topicName: 'transaction-events',
      displayName: 'Banking Transaction Events'
    });

    // Account events topic
    const accountTopic = new sns.Topic(this, 'AccountTopic', {
      topicName: 'account-events',
      displayName: 'Banking Account Events'
    });

    // Alert topic (for critical events)
    const alertTopic = new sns.Topic(this, 'AlertTopic', {
      topicName: 'critical-alerts',
      displayName: 'Critical Banking Alerts'
    });

    // ============================================
    // 5. SNS SUBSCRIPTIONS (Fan-out pattern)
    // ============================================

    // Transaction events → Email queue
    transactionTopic.addSubscription(
      new subscriptions.SqsSubscription(emailQueue, {
        rawMessageDelivery: true,
        filterPolicy: {
          transactionType: sns.SubscriptionFilter.stringFilter({
            allowlist: ['TRANSFER', 'PAYMENT']
          }),
          amount: sns.SubscriptionFilter.numericFilter({
            greaterThan: 1000
          })
        }
      })
    );

    // Transaction events → SMS queue
    transactionTopic.addSubscription(
      new subscriptions.SqsSubscription(smsQueue, {
        rawMessageDelivery: true,
        filterPolicy: {
          urgency: sns.SubscriptionFilter.stringFilter({
            allowlist: ['HIGH', 'CRITICAL']
          })
        }
      })
    );

    // Transaction events → Fraud detection queue
    transactionTopic.addSubscription(
      new subscriptions.SqsSubscription(fraudQueue, {
        rawMessageDelivery: true
      })
    );

    // ============================================
    // 6. LAMBDA FUNCTIONS
    // ============================================

    // Email processor
    const emailLambda = new lambda.Function(this, 'EmailProcessor', {
      functionName: 'email-notification-processor',
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambdas/email-processor'),
      timeout: cdk.Duration.seconds(30),
      reservedConcurrentExecutions: 50,
      environment: {
        SES_REGION: 'us-east-1',
        FROM_EMAIL: 'noreply@emiratesnbd.com'
      }
    });

    emailLambda.addEventSource(
      new lambdaEventSources.SqsEventSource(emailQueue, {
        batchSize: 10,
        maxBatchingWindow: cdk.Duration.seconds(5),
        reportBatchItemFailures: true
      })
    );

    // SMS processor
    const smsLambda = new lambda.Function(this, 'SmsProcessor', {
      functionName: 'sms-notification-processor',
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambdas/sms-processor'),
      timeout: cdk.Duration.seconds(30),
      environment: {
        SNS_REGION: 'us-east-1'
      }
    });

    smsLambda.addEventSource(
      new lambdaEventSources.SqsEventSource(smsQueue, {
        batchSize: 10,
        reportBatchItemFailures: true
      })
    );

    // Fraud detection processor
    const fraudLambda = new lambda.Function(this, 'FraudProcessor', {
      functionName: 'fraud-detection-processor',
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambdas/fraud-processor'),
      timeout: cdk.Duration.seconds(60),
      memorySize: 1024
    });

    fraudLambda.addEventSource(
      new lambdaEventSources.SqsEventSource(fraudQueue, {
        batchSize: 5,
        reportBatchItemFailures: true
      })
    );

    // Payment processor (FIFO)
    const paymentLambda = new lambda.Function(this, 'PaymentProcessor', {
      functionName: 'payment-fifo-processor',
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambdas/payment-processor'),
      timeout: cdk.Duration.seconds(120),
      reservedConcurrentExecutions: 10 // Limit concurrency for FIFO
    });

    paymentLambda.addEventSource(
      new lambdaEventSources.SqsEventSource(paymentFifoQueue, {
        batchSize: 1, // Process one at a time for strict ordering
        reportBatchItemFailures: true
      })
    );

    // DLQ monitor (alerts on DLQ messages)
    const dlqMonitorLambda = new lambda.Function(this, 'DlqMonitor', {
      functionName: 'dlq-monitor',
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambdas/dlq-monitor'),
      timeout: cdk.Duration.seconds(30),
      environment: {
        ALERT_TOPIC_ARN: alertTopic.topicArn
      }
    });

    dlqMonitorLambda.addEventSource(
      new lambdaEventSources.SqsEventSource(emailDLQ, {
        batchSize: 10
      })
    );

    // Grant permissions
    alertTopic.grantPublish(dlqMonitorLambda);

    // ============================================
    // 7. DYNAMODB TABLES
    // ============================================

    const paymentsTable = new dynamodb.Table(this, 'PaymentsTable', {
      tableName: 'banking-payments',
      partitionKey: { name: 'paymentId', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      stream: dynamodb.StreamViewType.NEW_AND_OLD_IMAGES
    });

    paymentsTable.grantReadWriteData(paymentLambda);

    // ============================================
    // 8. OUTPUTS
    // ============================================

    new cdk.CfnOutput(this, 'TransactionTopicArn', {
      value: transactionTopic.topicArn,
      description: 'Transaction Events Topic ARN'
    });

    new cdk.CfnOutput(this, 'PaymentQueueUrl', {
      value: paymentFifoQueue.queueUrl,
      description: 'Payment Processing FIFO Queue URL'
    });
  }
}
```

---

### 💻 Message Publisher

```javascript
// publishers/transaction-publisher.js
const { SNSClient, PublishCommand } = require('@aws-sdk/client-sns');
const { SQSClient, SendMessageCommand } = require('@aws-sdk/client-sqs');

const snsClient = new SNSClient({ region: 'us-east-1' });
const sqsClient = new SQSClient({ region: 'us-east-1' });

class MessagePublisher {
  constructor() {
    this.transactionTopicArn = process.env.TRANSACTION_TOPIC_ARN;
    this.paymentQueueUrl = process.env.PAYMENT_QUEUE_URL;
  }

  // Publish to SNS (fan-out pattern)
  async publishTransactionEvent(transaction) {
    const message = {
      transactionId: transaction.transactionId,
      fromAccount: transaction.fromAccount,
      toAccount: transaction.toAccount,
      amount: transaction.amount,
      currency: transaction.currency,
      transactionType: transaction.transactionType,
      timestamp: new Date().toISOString(),
      urgency: transaction.amount > 10000 ? 'HIGH' : 'NORMAL'
    };

    const response = await snsClient.send(new PublishCommand({
      TopicArn: this.transactionTopicArn,
      Message: JSON.stringify(message),
      MessageAttributes: {
        transactionType: {
          DataType: 'String',
          StringValue: transaction.transactionType
        },
        amount: {
          DataType: 'Number',
          StringValue: transaction.amount.toString()
        },
        urgency: {
          DataType: 'String',
          StringValue: message.urgency
        }
      }
    }));

    console.log('Transaction event published:', response.MessageId);
    return response.MessageId;
  }

  // Send to SQS FIFO (ordering guaranteed)
  async sendPaymentOrder(payment) {
    const message = {
      paymentId: payment.paymentId,
      fromAccount: payment.fromAccount,
      toAccount: payment.toAccount,
      amount: payment.amount,
      currency: payment.currency,
      timestamp: new Date().toISOString()
    };

    const response = await sqsClient.send(new SendMessageCommand({
      QueueUrl: this.paymentQueueUrl,
      MessageBody: JSON.stringify(message),
      MessageGroupId: payment.fromAccount, // Orders per account
      MessageDeduplicationId: payment.paymentId, // Deduplication
      MessageAttributes: {
        priority: {
          DataType: 'String',
          StringValue: payment.priority || 'NORMAL'
        }
      }
    }));

    console.log('Payment order sent:', response.MessageId);
    return response.MessageId;
  }

  // Batch send to SQS (better performance)
  async sendBatch(queueUrl, messages) {
    const entries = messages.map((msg, index) => ({
      Id: index.toString(),
      MessageBody: JSON.stringify(msg),
      DelaySeconds: msg.delaySeconds || 0
    }));

    // Split into batches of 10 (SQS limit)
    const batches = [];
    for (let i = 0; i < entries.length; i += 10) {
      batches.push(entries.slice(i, i + 10));
    }

    const results = [];
    for (const batch of batches) {
      const response = await sqsClient.send(new SendMessageBatchCommand({
        QueueUrl: queueUrl,
        Entries: batch
      }));
      results.push(response);
    }

    return results;
  }
}

module.exports = MessagePublisher;

// Example usage
/*
const publisher = new MessagePublisher();

// Publish transaction event (fan-out to email, SMS, fraud detection)
await publisher.publishTransactionEvent({
  transactionId: 'TXN-123456',
  fromAccount: 'ACC-12345',
  toAccount: 'ACC-67890',
  amount: 15000,
  currency: 'AED',
  transactionType: 'TRANSFER'
});

// Send payment order (FIFO queue for ordering)
await publisher.sendPaymentOrder({
  paymentId: 'PAY-789012',
  fromAccount: 'ACC-12345',
  toAccount: 'ACC-67890',
  amount: 5000,
  currency: 'AED',
  priority: 'HIGH'
});
*/
```

---

### 📥 Message Consumers (Lambda)

```javascript
// lambdas/email-processor/index.js
const { SESClient, SendEmailCommand } = require('@aws-sdk/client-ses');

const sesClient = new SESClient({ region: process.env.SES_REGION });

exports.handler = async (event) => {
  console.log('Received SQS event:', JSON.stringify(event, null, 2));

  const batchItemFailures = [];

  for (const record of event.Records) {
    try {
      const transaction = JSON.parse(record.body);
      
      await sendTransactionEmail(transaction);
      
      console.log('Email sent for transaction:', transaction.transactionId);
    } catch (error) {
      console.error('Error processing record:', error);
      
      // Report failed item for retry
      batchItemFailures.push({
        itemIdentifier: record.messageId
      });
    }
  }

  // Return failed items (SQS will retry these)
  return { batchItemFailures };
};

async function sendTransactionEmail(transaction) {
  const customerEmail = await getCustomerEmail(transaction.fromAccount);

  await sesClient.send(new SendEmailCommand({
    Source: process.env.FROM_EMAIL,
    Destination: {
      ToAddresses: [customerEmail]
    },
    Message: {
      Subject: {
        Data: `Transaction Confirmation - ${transaction.transactionId}`
      },
      Body: {
        Html: {
          Data: `
            <h2>Transaction Completed</h2>
            <p><strong>Transaction ID:</strong> ${transaction.transactionId}</p>
            <p><strong>Amount:</strong> ${transaction.currency} ${transaction.amount}</p>
            <p><strong>From:</strong> ${transaction.fromAccount}</p>
            <p><strong>To:</strong> ${transaction.toAccount}</p>
            <p><strong>Date:</strong> ${new Date(transaction.timestamp).toLocaleString()}</p>
          `
        }
      }
    }
  }));
}

async function getCustomerEmail(accountId) {
  // Query database for customer email
  return 'customer@example.com'; // Placeholder
}
```

```javascript
// lambdas/sms-processor/index.js
const { SNSClient, PublishCommand } = require('@aws-sdk/client-sns');

const snsClient = new SNSClient({ region: process.env.SNS_REGION });

exports.handler = async (event) => {
  console.log('Received SQS event:', JSON.stringify(event, null, 2));

  const batchItemFailures = [];

  for (const record of event.Records) {
    try {
      const transaction = JSON.parse(record.body);
      
      await sendTransactionSms(transaction);
      
      console.log('SMS sent for transaction:', transaction.transactionId);
    } catch (error) {
      console.error('Error processing record:', error);
      
      batchItemFailures.push({
        itemIdentifier: record.messageId
      });
    }
  }

  return { batchItemFailures };
};

async function sendTransactionSms(transaction) {
  const phoneNumber = await getCustomerPhone(transaction.fromAccount);

  await snsClient.send(new PublishCommand({
    PhoneNumber: phoneNumber,
    Message: `Emirates NBD: Transaction ${transaction.transactionId} of ${transaction.currency} ${transaction.amount} completed successfully. From: ${transaction.fromAccount} To: ${transaction.toAccount}`
  }));
}

async function getCustomerPhone(accountId) {
  // Query database for phone number
  return '+971501234567'; // Placeholder
}
```

```javascript
// lambdas/fraud-processor/index.js
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, PutCommand } = require('@aws-sdk/lib-dynamodb');

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

exports.handler = async (event) => {
  console.log('Received SQS event:', JSON.stringify(event, null, 2));

  const batchItemFailures = [];

  for (const record of event.Records) {
    try {
      const transaction = JSON.parse(record.body);
      
      const fraudScore = await analyzeFraud(transaction);
      
      if (fraudScore > 80) {
        await blockTransaction(transaction, fraudScore);
      }
      
      console.log('Fraud analysis completed:', transaction.transactionId, 'Score:', fraudScore);
    } catch (error) {
      console.error('Error processing record:', error);
      
      batchItemFailures.push({
        itemIdentifier: record.messageId
      });
    }
  }

  return { batchItemFailures };
};

async function analyzeFraud(transaction) {
  // Fraud detection logic
  let score = 0;

  // Check amount
  if (transaction.amount > 50000) score += 30;
  else if (transaction.amount > 20000) score += 15;

  // Check velocity (transactions per hour)
  const recentCount = await getRecentTransactionCount(transaction.fromAccount);
  if (recentCount > 10) score += 40;
  else if (recentCount > 5) score += 20;

  // Check location (placeholder)
  // const location = await getTransactionLocation(transaction);
  // if (location.country !== 'AE') score += 20;

  return score;
}

async function blockTransaction(transaction, fraudScore) {
  await docClient.send(new PutCommand({
    TableName: 'fraud-alerts',
    Item: {
      alertId: `ALERT-${Date.now()}`,
      transactionId: transaction.transactionId,
      fraudScore,
      status: 'BLOCKED',
      timestamp: new Date().toISOString()
    }
  }));

  console.log('Transaction blocked:', transaction.transactionId);
}

async function getRecentTransactionCount(accountId) {
  // Query recent transactions
  return 3; // Placeholder
}
```

```javascript
// lambdas/payment-processor/index.js
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, PutCommand, UpdateCommand } = require('@aws-sdk/lib-dynamodb');

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

exports.handler = async (event) => {
  console.log('Received FIFO SQS event:', JSON.stringify(event, null, 2));

  // FIFO queues: process one message at a time for strict ordering
  const record = event.Records[0];
  
  try {
    const payment = JSON.parse(record.body);
    
    await processPayment(payment);
    
    console.log('Payment processed:', payment.paymentId);
    
    return { batchItemFailures: [] };
  } catch (error) {
    console.error('Error processing payment:', error);
    
    // Return failure (will be retried or moved to DLQ)
    return {
      batchItemFailures: [{
        itemIdentifier: record.messageId
      }]
    };
  }
};

async function processPayment(payment) {
  // Store payment
  await docClient.send(new PutCommand({
    TableName: 'banking-payments',
    Item: {
      paymentId: payment.paymentId,
      fromAccount: payment.fromAccount,
      toAccount: payment.toAccount,
      amount: payment.amount,
      currency: payment.currency,
      status: 'PROCESSING',
      timestamp: payment.timestamp
    }
  }));

  // Simulate payment processing
  await sleep(2000);

  // Update status
  await docClient.send(new UpdateCommand({
    TableName: 'banking-payments',
    Key: { paymentId: payment.paymentId },
    UpdateExpression: 'SET #status = :status, processedAt = :processedAt',
    ExpressionAttributeNames: {
      '#status': 'status'
    },
    ExpressionAttributeValues: {
      ':status': 'COMPLETED',
      ':processedAt': new Date().toISOString()
    }
  }));
}

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}
```

```javascript
// lambdas/dlq-monitor/index.js
const { SNSClient, PublishCommand } = require('@aws-sdk/client-sns');

const snsClient = new SNSClient({ region: 'us-east-1' });

exports.handler = async (event) => {
  console.log('DLQ message received:', JSON.stringify(event, null, 2));

  for (const record of event.Records) {
    try {
      const failedMessage = JSON.parse(record.body);
      
      await sendAlert(failedMessage, record);
      
      console.log('Alert sent for failed message:', record.messageId);
    } catch (error) {
      console.error('Error processing DLQ record:', error);
    }
  }

  return { statusCode: 200 };
};

async function sendAlert(failedMessage, record) {
  await snsClient.send(new PublishCommand({
    TopicArn: process.env.ALERT_TOPIC_ARN,
    Subject: 'DLQ Alert: Message Processing Failed',
    Message: JSON.stringify({
      messageId: record.messageId,
      receiveCount: record.attributes.ApproximateReceiveCount,
      failedMessage,
      timestamp: new Date().toISOString()
    }, null, 2)
  }));
}
```

---

### 🎓 Interview Discussion Points - Q15

**Q1: What's the difference between SQS Standard and FIFO queues?**

**A**:
- **Standard**: Best-effort ordering, at-least-once delivery, unlimited throughput
- **FIFO**: Strict ordering, exactly-once processing, 300 TPS (3000 with batching)

**Q2: How do you handle message deduplication?**

**A**:
- **FIFO**: Content-based or message deduplication ID (5-minute window)
- **Standard**: Implement idempotency keys in application
- Store processed message IDs in DynamoDB with TTL

**Q3: When to use SNS vs SQS vs EventBridge?**

**A**:
- **SNS**: Simple pub/sub, fan-out, mobile push, SMS/email
- **SQS**: Work queues, decoupling, buffering, retry logic
- **EventBridge**: Complex routing, schema validation, cross-account

**Q4: How do you implement message priority in SQS?**

**A**:
- Create separate queues for priorities (high, normal, low)
- Lambda polls high-priority queue first
- Use weighted polling or dedicated workers

**Q5: What are DLQ best practices?**

**A**:
- Set maxReceiveCount to 3-5
- Monitor DLQ depth with CloudWatch alarms
- Implement automated alerts
- Periodically replay or investigate messages
- Set appropriate retention (14 days)

---

## Question 16: gRPC Implementation with Protocol Buffers

### 📋 Question Statement

Implement high-performance inter-service communication for Emirates NBD using gRPC with Protocol Buffers. Include service definitions, streaming RPCs, error handling, load balancing, and integration with Cloud Map.

---

### 🏗️ gRPC Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                  EMIRATES NBD gRPC ARCHITECTURE                             │
└────────────────────────────────────────────────────────────────────────────┘

                    CLIENT-SERVER COMMUNICATION
                    ───────────────────────────
                    
        ┌──────────────────┐                  ┌──────────────────┐
        │  Payment Service │                  │ Account Service  │
        │  (gRPC Client)   │                  │  (gRPC Server)   │
        └────────┬─────────┘                  └────────┬─────────┘
                 │                                      │
                 │  1. Unary RPC                       │
                 │  GetAccount(accountId)              │
                 ├─────────────────────────────────────>│
                 │                                      │
                 │  2. Response                         │
                 │<─────────────────────────────────────┤
                 │     Account{balance, ...}            │
                 │                                      │
                 │  3. Server Streaming                 │
                 │  StreamTransactions(accountId)       │
                 ├─────────────────────────────────────>│
                 │                                      │
                 │  4. Stream Response                  │
                 │<─────────────────────────────────────┤
                 │     Transaction{...}                 │
                 │<─────────────────────────────────────┤
                 │     Transaction{...}                 │
                 │<─────────────────────────────────────┤
                 │     [End Stream]                     │

                    BIDIRECTIONAL STREAMING
                    ───────────────────────
                    
        ┌──────────────────┐                  ┌──────────────────┐
        │  Trading Service │                  │  Price Service   │
        └────────┬─────────┘                  └────────┬─────────┘
                 │                                      │
                 │  Subscribe(symbol)                   │
                 ├─────────────────────────────────────>│
                 │                                      │
                 │  Price Update                        │
                 │<─────────────────────────────────────┤
                 │                                      │
                 │  Subscribe(another symbol)           │
                 ├─────────────────────────────────────>│
                 │                                      │
                 │  Price Update (both symbols)         │
                 │<─────────────────────────────────────┤
                 │<─────────────────────────────────────┤

                    LOAD BALANCING
                    ──────────────
                    
                    ┌─────────────┐
                    │  gRPC Client│
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │  Cloud Map  │
                    │  (Service   │
                    │  Discovery) │
                    └──────┬──────┘
                           │
                ┌──────────┼──────────┐
                │          │          │
                ▼          ▼          ▼
         ┌──────────┐┌──────────┐┌──────────┐
         │ Server 1 ││ Server 2 ││ Server 3 │
         └──────────┘└──────────┘└──────────┘
```

---

### 📝 Protocol Buffer Definitions

```protobuf
// protos/account.proto
syntax = "proto3";

package banking;

option go_package = "github.com/emiratesnbd/proto/banking";
option java_package = "com.emiratesnbd.banking";

import "google/protobuf/timestamp.proto";

// Account Service
service AccountService {
  // Unary RPC: Get single account
  rpc GetAccount(GetAccountRequest) returns (Account);
  
  // Unary RPC: Create account
  rpc CreateAccount(CreateAccountRequest) returns (Account);
  
  // Unary RPC: Update balance
  rpc UpdateBalance(UpdateBalanceRequest) returns (Account);
  
  // Server streaming: Stream transactions for an account
  rpc StreamTransactions(StreamTransactionsRequest) returns (stream Transaction);
  
  // Client streaming: Batch account creation
  rpc BatchCreateAccounts(stream CreateAccountRequest) returns (BatchCreateResponse);
  
  // Bidirectional streaming: Real-time balance updates
  rpc SubscribeToBalanceUpdates(stream SubscribeRequest) returns (stream BalanceUpdate);
}

message GetAccountRequest {
  string account_id = 1;
}

message CreateAccountRequest {
  string customer_id = 1;
  AccountType account_type = 2;
  string currency = 3;
  double initial_balance = 4;
}

message UpdateBalanceRequest {
  string account_id = 1;
  double amount = 2;
  BalanceOperation operation = 3;
}

message StreamTransactionsRequest {
  string account_id = 1;
  google.protobuf.Timestamp start_date = 2;
  google.protobuf.Timestamp end_date = 3;
}

message SubscribeRequest {
  string account_id = 1;
}

message BatchCreateResponse {
  int32 created_count = 1;
  repeated string account_ids = 2;
}

message Account {
  string account_id = 1;
  string customer_id = 2;
  AccountType account_type = 3;
  string currency = 4;
  double balance = 5;
  AccountStatus status = 6;
  google.protobuf.Timestamp created_at = 7;
  google.protobuf.Timestamp updated_at = 8;
}

message Transaction {
  string transaction_id = 1;
  string from_account = 2;
  string to_account = 3;
  double amount = 4;
  string currency = 5;
  TransactionType transaction_type = 6;
  TransactionStatus status = 7;
  google.protobuf.Timestamp timestamp = 8;
}

message BalanceUpdate {
  string account_id = 1;
  double balance = 2;
  double previous_balance = 3;
  google.protobuf.Timestamp timestamp = 4;
}

enum AccountType {
  SAVINGS = 0;
  CURRENT = 1;
  FIXED_DEPOSIT = 2;
  CREDIT_CARD = 3;
}

enum AccountStatus {
  ACTIVE = 0;
  INACTIVE = 1;
  FROZEN = 2;
  CLOSED = 3;
}

enum TransactionType {
  TRANSFER = 0;
  PAYMENT = 1;
  WITHDRAWAL = 2;
  DEPOSIT = 3;
}

enum TransactionStatus {
  PENDING = 0;
  COMPLETED = 1;
  FAILED = 2;
  CANCELLED = 3;
}

enum BalanceOperation {
  CREDIT = 0;
  DEBIT = 1;
}
```

```protobuf
// protos/payment.proto
syntax = "proto3";

package banking;

import "google/protobuf/timestamp.proto";

service PaymentService {
  rpc ProcessPayment(PaymentRequest) returns (PaymentResponse);
  rpc GetPaymentStatus(PaymentStatusRequest) returns (PaymentStatus);
  rpc StreamPayments(StreamPaymentsRequest) returns (stream Payment);
}

message PaymentRequest {
  string payment_id = 1;
  string from_account = 2;
  string to_account = 3;
  double amount = 4;
  string currency = 5;
  string description = 6;
}

message PaymentResponse {
  string payment_id = 1;
  PaymentStatus status = 2;
  string transaction_id = 3;
}

message PaymentStatusRequest {
  string payment_id = 1;
}

message PaymentStatus {
  string payment_id = 1;
  string status = 2;
  google.protobuf.Timestamp created_at = 3;
  google.protobuf.Timestamp updated_at = 4;
}

message StreamPaymentsRequest {
  string account_id = 1;
}

message Payment {
  string payment_id = 1;
  string from_account = 2;
  string to_account = 3;
  double amount = 4;
  string currency = 5;
  google.protobuf.Timestamp timestamp = 6;
}
```

---

### 🖥️ gRPC Server Implementation (Node.js)

```javascript
// services/account-service/src/grpc-server.js
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');

// Load proto files
const PROTO_PATH = path.join(__dirname, '../../../protos/account.proto');
const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true
});

const accountProto = grpc.loadPackageDefinition(packageDefinition).banking;

// Database mock (replace with real DB)
const accountsDB = new Map();

// Unary RPC: Get Account
function getAccount(call, callback) {
  const { account_id } = call.request;
  
  console.log('GetAccount request:', account_id);
  
  const account = accountsDB.get(account_id);
  
  if (!account) {
    return callback({
      code: grpc.status.NOT_FOUND,
      message: `Account ${account_id} not found`
    });
  }
  
  callback(null, account);
}

// Unary RPC: Create Account
function createAccount(call, callback) {
  const { customer_id, account_type, currency, initial_balance } = call.request;
  
  console.log('CreateAccount request:', call.request);
  
  const account = {
    account_id: `ACC-${Date.now()}`,
    customer_id,
    account_type,
    currency,
    balance: initial_balance,
    status: 'ACTIVE',
    created_at: { seconds: Math.floor(Date.now() / 1000) },
    updated_at: { seconds: Math.floor(Date.now() / 1000) }
  };
  
  accountsDB.set(account.account_id, account);
  
  callback(null, account);
}

// Unary RPC: Update Balance
function updateBalance(call, callback) {
  const { account_id, amount, operation } = call.request;
  
  console.log('UpdateBalance request:', call.request);
  
  const account = accountsDB.get(account_id);
  
  if (!account) {
    return callback({
      code: grpc.status.NOT_FOUND,
      message: `Account ${account_id} not found`
    });
  }
  
  if (operation === 'CREDIT') {
    account.balance += amount;
  } else if (operation === 'DEBIT') {
    if (account.balance < amount) {
      return callback({
        code: grpc.status.FAILED_PRECONDITION,
        message: 'Insufficient funds'
      });
    }
    account.balance -= amount;
  }
  
  account.updated_at = { seconds: Math.floor(Date.now() / 1000) };
  
  callback(null, account);
}

// Server streaming: Stream Transactions
function streamTransactions(call) {
  const { account_id } = call.request;
  
  console.log('StreamTransactions request:', account_id);
  
  // Simulate streaming transactions
  const transactions = [
    {
      transaction_id: 'TXN-001',
      from_account: account_id,
      to_account: 'ACC-67890',
      amount: 1000,
      currency: 'AED',
      transaction_type: 'TRANSFER',
      status: 'COMPLETED',
      timestamp: { seconds: Math.floor(Date.now() / 1000) }
    },
    {
      transaction_id: 'TXN-002',
      from_account: 'ACC-11111',
      to_account: account_id,
      amount: 500,
      currency: 'AED',
      transaction_type: 'TRANSFER',
      status: 'COMPLETED',
      timestamp: { seconds: Math.floor(Date.now() / 1000) }
    }
  ];
  
  transactions.forEach((txn, index) => {
    setTimeout(() => {
      call.write(txn);
      
      if (index === transactions.length - 1) {
        call.end();
      }
    }, index * 1000); // Send one per second
  });
}

// Client streaming: Batch Create Accounts
function batchCreateAccounts(call, callback) {
  let createdCount = 0;
  const accountIds = [];
  
  call.on('data', (request) => {
    console.log('Batch create account:', request);
    
    const account = {
      account_id: `ACC-${Date.now()}-${createdCount}`,
      customer_id: request.customer_id,
      account_type: request.account_type,
      currency: request.currency,
      balance: request.initial_balance,
      status: 'ACTIVE',
      created_at: { seconds: Math.floor(Date.now() / 1000) },
      updated_at: { seconds: Math.floor(Date.now() / 1000) }
    };
    
    accountsDB.set(account.account_id, account);
    accountIds.push(account.account_id);
    createdCount++;
  });
  
  call.on('end', () => {
    callback(null, {
      created_count: createdCount,
      account_ids: accountIds
    });
  });
}

// Bidirectional streaming: Subscribe to Balance Updates
function subscribeToBalanceUpdates(call) {
  const subscriptions = new Map();
  
  call.on('data', (request) => {
    const { account_id } = request;
    console.log('Subscribe to balance updates:', account_id);
    
    subscriptions.set(account_id, true);
    
    // Simulate balance updates
    const interval = setInterval(() => {
      const account = accountsDB.get(account_id);
      
      if (account) {
        const update = {
          account_id,
          balance: account.balance,
          previous_balance: account.balance - 100,
          timestamp: { seconds: Math.floor(Date.now() / 1000) }
        };
        
        call.write(update);
      }
    }, 5000); // Update every 5 seconds
    
    call.on('end', () => {
      clearInterval(interval);
    });
  });
  
  call.on('end', () => {
    call.end();
  });
}

// Start gRPC server
function startServer() {
  const server = new grpc.Server();
  
  server.addService(accountProto.AccountService.service, {
    getAccount,
    createAccount,
    updateBalance,
    streamTransactions,
    batchCreateAccounts,
    subscribeToBalanceUpdates
  });
  
  const PORT = process.env.GRPC_PORT || 50051;
  
  server.bindAsync(
    `0.0.0.0:${PORT}`,
    grpc.ServerCredentials.createInsecure(),
    (error, port) => {
      if (error) {
        console.error('Server binding failed:', error);
        return;
      }
      
      console.log(`gRPC server started on port ${port}`);
      server.start();
    }
  );
}

// Graceful shutdown
process.on('SIGTERM', () => {
  console.log('SIGTERM received, shutting down gRPC server');
  process.exit(0);
});

startServer();
```

---

### 📱 gRPC Client Implementation

```javascript
// services/payment-service/src/grpc-client.js
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');

// Load proto files
const PROTO_PATH = path.join(__dirname, '../../../protos/account.proto');
const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true
});

const accountProto = grpc.loadPackageDefinition(packageDefinition).banking;

class AccountServiceClient {
  constructor(serverAddress) {
    this.client = new accountProto.AccountService(
      serverAddress || 'account-service.emiratesnbd.local:50051',
      grpc.credentials.createInsecure()
    );
  }

  // Unary call with deadline
  async getAccount(accountId) {
    return new Promise((resolve, reject) => {
      const deadline = new Date();
      deadline.setSeconds(deadline.getSeconds() + 5); // 5 second timeout
      
      this.client.getAccount(
        { account_id: accountId },
        { deadline: deadline.getTime() },
        (error, response) => {
          if (error) {
            console.error('GetAccount error:', error);
            return reject(error);
          }
          
          resolve(response);
        }
      );
    });
  }

  async createAccount(customerData) {
    return new Promise((resolve, reject) => {
      this.client.createAccount(customerData, (error, response) => {
        if (error) return reject(error);
        resolve(response);
      });
    });
  }

  async updateBalance(accountId, amount, operation) {
    return new Promise((resolve, reject) => {
      this.client.updateBalance(
        { account_id: accountId, amount, operation },
        (error, response) => {
          if (error) return reject(error);
          resolve(response);
        }
      );
    });
  }

  // Server streaming
  streamTransactions(accountId) {
    const call = this.client.streamTransactions({ account_id: accountId });
    
    const transactions = [];
    
    return new Promise((resolve, reject) => {
      call.on('data', (transaction) => {
        console.log('Received transaction:', transaction);
        transactions.push(transaction);
      });
      
      call.on('end', () => {
        console.log('Stream ended');
        resolve(transactions);
      });
      
      call.on('error', (error) => {
        console.error('Stream error:', error);
        reject(error);
      });
    });
  }

  // Client streaming
  async batchCreateAccounts(accounts) {
    return new Promise((resolve, reject) => {
      const call = this.client.batchCreateAccounts((error, response) => {
        if (error) return reject(error);
        resolve(response);
      });
      
      accounts.forEach(account => {
        call.write(account);
      });
      
      call.end();
    });
  }

  // Bidirectional streaming
  subscribeToBalanceUpdates(accountIds) {
    const call = this.client.subscribeToBalanceUpdates();
    
    // Subscribe to accounts
    accountIds.forEach(accountId => {
      call.write({ account_id: accountId });
    });
    
    // Listen for updates
    call.on('data', (update) => {
      console.log('Balance update:', update);
    });
    
    call.on('error', (error) => {
      console.error('Subscription error:', error);
    });
    
    return call; // Return for later cleanup
  }

  close() {
    grpc.closeClient(this.client);
  }
}

module.exports = AccountServiceClient;

// Example usage
/*
const client = new AccountServiceClient('localhost:50051');

// Unary call
const account = await client.getAccount('ACC-12345');
console.log('Account:', account);

// Server streaming
const transactions = await client.streamTransactions('ACC-12345');
console.log('Transactions:', transactions);

// Client streaming
const result = await client.batchCreateAccounts([
  { customer_id: 'CUST-001', account_type: 'SAVINGS', currency: 'AED', initial_balance: 1000 },
  { customer_id: 'CUST-002', account_type: 'CURRENT', currency: 'AED', initial_balance: 5000 }
]);
console.log('Batch result:', result);

// Bidirectional streaming
const subscription = client.subscribeToBalanceUpdates(['ACC-12345', 'ACC-67890']);

// Later: cleanup
setTimeout(() => {
  subscription.end();
  client.close();
}, 30000);
*/
```

---

### 🔧 Service Discovery Integration

```javascript
// clients/grpc-client-with-discovery.js
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const { ServiceDiscoveryClient } = require('@aws-sdk/client-servicediscovery');

const discoveryClient = new ServiceDiscoveryClient({ region: 'us-east-1' });

class LoadBalancedGrpcClient {
  constructor(serviceName, namespace) {
    this.serviceName = serviceName;
    this.namespace = namespace;
    this.clients = [];
    this.currentIndex = 0;
  }

  async initialize() {
    // Discover service instances
    const instances = await this.discoverInstances();
    
    // Create gRPC client for each instance
    const PROTO_PATH = path.join(__dirname, '../../../protos/account.proto');
    const packageDefinition = protoLoader.loadSync(PROTO_PATH);
    const accountProto = grpc.loadPackageDefinition(packageDefinition).banking;
    
    instances.forEach(instance => {
      const address = `${instance.ip}:${instance.port}`;
      const client = new accountProto.AccountService(
        address,
        grpc.credentials.createInsecure()
      );
      
      this.clients.push({ address, client, healthy: true });
    });
    
    console.log(`Initialized ${this.clients.length} gRPC clients`);
  }

  async discoverInstances() {
    const response = await discoveryClient.send(new DiscoverInstancesCommand({
      NamespaceName: this.namespace,
      ServiceName: this.serviceName,
      HealthStatus: 'HEALTHY'
    }));
    
    return response.Instances.map(inst => ({
      ip: inst.Attributes.AWS_INSTANCE_IPV4,
      port: parseInt(inst.Attributes.AWS_INSTANCE_PORT || '50051')
    }));
  }

  // Round-robin load balancing
  getClient() {
    if (this.clients.length === 0) {
      throw new Error('No healthy clients available');
    }
    
    const client = this.clients[this.currentIndex];
    this.currentIndex = (this.currentIndex + 1) % this.clients.length;
    
    return client.client;
  }

  async getAccount(accountId) {
    const client = this.getClient();
    
    return new Promise((resolve, reject) => {
      client.getAccount({ account_id: accountId }, (error, response) => {
        if (error) return reject(error);
        resolve(response);
      });
    });
  }

  // Refresh instances periodically
  startHealthCheck(intervalMs = 30000) {
    setInterval(async () => {
      try {
        await this.initialize();
      } catch (error) {
        console.error('Health check failed:', error);
      }
    }, intervalMs);
  }
}

module.exports = LoadBalancedGrpcClient;
```

---

### 🐳 Dockerfile with gRPC

```dockerfile
# Dockerfile
FROM node:20-alpine AS builder

WORKDIR /app

# Install protoc
RUN apk add --no-cache protobuf protobuf-dev

# Copy proto files
COPY protos/ /app/protos/

# Copy package files
COPY package*.json ./
RUN npm ci --production

# Copy source
COPY src/ /app/src/

FROM node:20-alpine

WORKDIR /app

# Copy from builder
COPY --from=builder /app /app

# Expose gRPC port
EXPOSE 50051

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD node -e "require('grpc-health-check')" || exit 1

CMD ["node", "src/grpc-server.js"]
```

---

### 🎓 Interview Discussion Points - Q16

**Q1: What are the advantages of gRPC over REST?**

**A**:
- **Performance**: Binary protocol (Protocol Buffers) vs JSON
- **Streaming**: Bidirectional streaming support
- **Type Safety**: Strong typing with .proto files
- **Code Generation**: Auto-generate client/server code
- **HTTP/2**: Multiplexing, server push

**Q2: When NOT to use gRPC?**

**A**:
- Browser clients (limited support, use gRPC-Web)
- Public APIs (REST is more accessible)
- Simple CRUD operations (REST is simpler)
- Debugging (binary format harder to inspect)

**Q3: How do you handle errors in gRPC?**

**A**:
- Use standard gRPC status codes (NOT_FOUND, INVALID_ARGUMENT, etc.)
- Include error details in metadata
- Implement retry logic with exponential backoff
- Use deadlines/timeouts for all calls

**Q4: How does gRPC load balancing work?**

**A**:
- **Client-side**: Client maintains connections to multiple servers
- **Proxy-based**: Use Envoy or NGINX as load balancer
- **Cloud Map**: Discover healthy instances dynamically
- **Round-robin**: Rotate requests across instances

**Q5: What is the difference between unary and streaming RPCs?**

**A**:
- **Unary**: Single request, single response (like REST)
- **Server streaming**: Single request, stream of responses
- **Client streaming**: Stream of requests, single response
- **Bidirectional**: Both client and server stream

---

**End of File 8**

