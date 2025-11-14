# SQS & SNS Setup - Serverless Order Processing Microservice

## 📋 Overview

This guide covers the complete setup of Amazon SQS (Simple Queue Service) and SNS (Simple Notification Service) for asynchronous message processing and event-driven notifications in the serverless microservice.

---

## 🎯 Messaging Architecture Goals

- **Decoupling**: Separate producers from consumers
- **Reliability**: Guaranteed message delivery with DLQ
- **Scalability**: Handle millions of messages per day
- **Ordering**: FIFO queues for ordered processing
- **Fan-out**: SNS topics for multiple subscribers

---

## 1️⃣ SQS Architecture Design

### Queue Types

```javascript
const queueArchitecture = {
  orderProcessingQueue: {
    type: 'FIFO',
    purpose: 'Process orders in sequence per customer',
    features: [
      'Exactly-once processing',
      'Content-based deduplication',
      'Message group ID for ordering'
    ],
    consumers: ['order-processor Lambda']
  },
  
  inventoryUpdateQueue: {
    type: 'Standard',
    purpose: 'Update inventory levels',
    features: [
      'High throughput',
      'At-least-once delivery',
      'Best-effort ordering'
    ],
    consumers: ['inventory-updater Lambda']
  },
  
  deadLetterQueues: {
    orderDLQ: {
      purpose: 'Failed order processing messages',
      retention: '14 days',
      alarmThreshold: 1
    },
    inventoryDLQ: {
      purpose: 'Failed inventory updates',
      retention: '14 days',
      alarmThreshold: 1
    }
  }
};
```

### Message Flow Diagram

```
┌─────────────┐
│ API Handler │
│   Lambda    │
└──────┬──────┘
       │
       │ SendMessage
       ▼
┌──────────────────────────────────┐
│  Order Processing Queue (FIFO)   │
│  - MessageGroupId: customerId    │
│  - DeduplicationId: orderId      │
│  - MaxReceiveCount: 3            │
└──────┬───────────────────────────┘
       │
       │ Event Source Mapping
       │ BatchSize: 10
       ▼
┌──────────────┐
│    Order     │
│  Processor   │
│   Lambda     │
└──────┬───────┘
       │
       ├─→ Success: DeleteMessage
       │
       └─→ Failure: Return to queue
           After 3 attempts → DLQ
```

---

## 2️⃣ CloudFormation Template - SQS Queues

Create file: `infrastructure/cloudformation/sqs-stack.yaml`

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'SQS Queues for Serverless Order Processing'

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - staging
      - production

Mappings:
  QueueSettings:
    dev:
      VisibilityTimeout: 300
      MessageRetentionPeriod: 345600  # 4 days
      MaxReceiveCount: 3
    staging:
      VisibilityTimeout: 300
      MessageRetentionPeriod: 864000  # 10 days
      MaxReceiveCount: 3
    production:
      VisibilityTimeout: 300
      MessageRetentionPeriod: 1209600  # 14 days
      MaxReceiveCount: 3

Resources:
  # ========================================
  # Dead Letter Queues
  # ========================================
  OrderProcessingDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub '${Environment}-order-processing-dlq.fifo'
      FifoQueue: true
      MessageRetentionPeriod: 1209600  # 14 days
      KmsMasterKeyId: !ImportValue
        'Fn::Sub': '${Environment}-SQS-KMS-Key-ID'
      Tags:
        - Key: Environment
          Value: !Ref Environment
        - Key: Type
          Value: DeadLetterQueue

  InventoryUpdateDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub '${Environment}-inventory-update-dlq'
      MessageRetentionPeriod: 1209600  # 14 days
      KmsMasterKeyId: !ImportValue
        'Fn::Sub': '${Environment}-SQS-KMS-Key-ID'
      Tags:
        - Key: Environment
          Value: !Ref Environment
        - Key: Type
          Value: DeadLetterQueue

  # ========================================
  # Order Processing Queue (FIFO)
  # ========================================
  OrderProcessingQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub '${Environment}-order-processing.fifo'
      FifoQueue: true
      ContentBasedDeduplication: true
      DeduplicationScope: messageGroup
      FifoThroughputLimit: perMessageGroupId
      
      # Timeouts
      VisibilityTimeout: !FindInMap [QueueSettings, !Ref Environment, VisibilityTimeout]
      MessageRetentionPeriod: !FindInMap [QueueSettings, !Ref Environment, MessageRetentionPeriod]
      ReceiveMessageWaitTimeSeconds: 20  # Long polling
      
      # Dead Letter Queue
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt OrderProcessingDLQ.Arn
        maxReceiveCount: !FindInMap [QueueSettings, !Ref Environment, MaxReceiveCount]
      
      # Encryption
      KmsMasterKeyId: !ImportValue
        'Fn::Sub': '${Environment}-SQS-KMS-Key-ID'
      KmsDataKeyReusePeriodSeconds: 300
      
      Tags:
        - Key: Environment
          Value: !Ref Environment
        - Key: Service
          Value: order-processing

  # Queue Policy for Lambda
  OrderProcessingQueuePolicy:
    Type: AWS::SQS::QueuePolicy
    Properties:
      Queues:
        - !Ref OrderProcessingQueue
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:role/${Environment}-api-handler-role'
            Action:
              - sqs:SendMessage
              - sqs:GetQueueAttributes
            Resource: !GetAtt OrderProcessingQueue.Arn
          
          - Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:role/${Environment}-order-processor-role'
            Action:
              - sqs:ReceiveMessage
              - sqs:DeleteMessage
              - sqs:GetQueueAttributes
              - sqs:ChangeMessageVisibility
            Resource: !GetAtt OrderProcessingQueue.Arn

  # ========================================
  # Inventory Update Queue (Standard)
  # ========================================
  InventoryUpdateQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub '${Environment}-inventory-update'
      
      # Timeouts
      VisibilityTimeout: 180  # 3 minutes
      MessageRetentionPeriod: !FindInMap [QueueSettings, !Ref Environment, MessageRetentionPeriod]
      ReceiveMessageWaitTimeSeconds: 20
      
      # Dead Letter Queue
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt InventoryUpdateDLQ.Arn
        maxReceiveCount: !FindInMap [QueueSettings, !Ref Environment, MaxReceiveCount]
      
      # Encryption
      KmsMasterKeyId: !ImportValue
        'Fn::Sub': '${Environment}-SQS-KMS-Key-ID'
      
      Tags:
        - Key: Environment
          Value: !Ref Environment
        - Key: Service
          Value: inventory-management

  InventoryUpdateQueuePolicy:
    Type: AWS::SQS::QueuePolicy
    Properties:
      Queues:
        - !Ref InventoryUpdateQueue
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:role/${Environment}-order-processor-role'
            Action:
              - sqs:SendMessage
            Resource: !GetAtt InventoryUpdateQueue.Arn
          
          - Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:role/${Environment}-inventory-updater-role'
            Action:
              - sqs:ReceiveMessage
              - sqs:DeleteMessage
              - sqs:GetQueueAttributes
              - sqs:ChangeMessageVisibility
            Resource: !GetAtt InventoryUpdateQueue.Arn

  # ========================================
  # CloudWatch Alarms - DLQ Monitoring
  # ========================================
  OrderDLQAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${Environment}-order-dlq-messages'
      AlarmDescription: Alert when messages appear in Order DLQ
      MetricName: ApproximateNumberOfMessagesVisible
      Namespace: AWS/SQS
      Statistic: Sum
      Period: 300
      EvaluationPeriods: 1
      Threshold: 1
      ComparisonOperator: GreaterThanOrEqualToThreshold
      Dimensions:
        - Name: QueueName
          Value: !GetAtt OrderProcessingDLQ.QueueName
      AlarmActions:
        - !Ref AlertTopic
      TreatMissingData: notBreaching

  InventoryDLQAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${Environment}-inventory-dlq-messages'
      AlarmDescription: Alert when messages appear in Inventory DLQ
      MetricName: ApproximateNumberOfMessagesVisible
      Namespace: AWS/SQS
      Statistic: Sum
      Period: 300
      EvaluationPeriods: 1
      Threshold: 1
      ComparisonOperator: GreaterThanOrEqualToThreshold
      Dimensions:
        - Name: QueueName
          Value: !GetAtt InventoryUpdateDLQ.QueueName
      AlarmActions:
        - !Ref AlertTopic
      TreatMissingData: notBreaching

  # ========================================
  # SNS Topic for Alerts
  # ========================================
  AlertTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: !Sub '${Environment}-sqs-alerts'
      DisplayName: SQS Alert Notifications
      KmsMasterKeyId: !ImportValue
        'Fn::Sub': '${Environment}-SQS-KMS-Key-ID'
      Tags:
        - Key: Environment
          Value: !Ref Environment

  AlertTopicSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      Protocol: email
      TopicArn: !Ref AlertTopic
      Endpoint: ops-team@example.com  # Replace with your email

# ========================================
# Outputs
# ========================================
Outputs:
  OrderProcessingQueueUrl:
    Description: Order Processing Queue URL
    Value: !Ref OrderProcessingQueue
    Export:
      Name: !Sub '${Environment}-OrderProcessingQueue-URL'

  OrderProcessingQueueArn:
    Description: Order Processing Queue ARN
    Value: !GetAtt OrderProcessingQueue.Arn
    Export:
      Name: !Sub '${Environment}-OrderProcessingQueue-ARN'

  OrderProcessingDLQUrl:
    Description: Order Processing DLQ URL
    Value: !Ref OrderProcessingDLQ
    Export:
      Name: !Sub '${Environment}-OrderProcessingDLQ-URL'

  InventoryUpdateQueueUrl:
    Description: Inventory Update Queue URL
    Value: !Ref InventoryUpdateQueue
    Export:
      Name: !Sub '${Environment}-InventoryUpdateQueue-URL'

  InventoryUpdateQueueArn:
    Description: Inventory Update Queue ARN
    Value: !GetAtt InventoryUpdateQueue.Arn
    Export:
      Name: !Sub '${Environment}-InventoryUpdateQueue-ARN'

  AlertTopicArn:
    Description: Alert Topic ARN
    Value: !Ref AlertTopic
    Export:
      Name: !Sub '${Environment}-AlertTopic-ARN'
```

---

## 3️⃣ SNS Topics Configuration

Create file: `infrastructure/cloudformation/sns-stack.yaml`

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'SNS Topics for Serverless Order Processing'

Parameters:
  Environment:
    Type: String
    Default: dev

Resources:
  # ========================================
  # Order Notifications Topic
  # ========================================
  OrderNotificationsTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: !Sub '${Environment}-order-notifications'
      DisplayName: Order Processing Notifications
      KmsMasterKeyId: !ImportValue
        'Fn::Sub': '${Environment}-SQS-KMS-Key-ID'
      Tags:
        - Key: Environment
          Value: !Ref Environment

  # Email Subscription for Admins
  AdminEmailSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      Protocol: email
      TopicArn: !Ref OrderNotificationsTopic
      Endpoint: admin@example.com
      FilterPolicy:
        orderStatus:
          - CANCELLED
          - FAILED
        priority:
          - HIGH

  # Lambda Subscription
  NotificationHandlerSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      Protocol: lambda
      TopicArn: !Ref OrderNotificationsTopic
      Endpoint: !Sub 'arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:${Environment}-notification-handler'

  # SQS Subscription for Analytics
  AnalyticsQueueSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      Protocol: sqs
      TopicArn: !Ref OrderNotificationsTopic
      Endpoint: !GetAtt AnalyticsQueue.Arn
      RawMessageDelivery: true
      FilterPolicy:
        eventType:
          - ORDER_COMPLETED
          - ORDER_CANCELLED

  # ========================================
  # Analytics Queue
  # ========================================
  AnalyticsQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub '${Environment}-analytics-queue'
      MessageRetentionPeriod: 1209600
      VisibilityTimeout: 300

  AnalyticsQueuePolicy:
    Type: AWS::SQS::QueuePolicy
    Properties:
      Queues:
        - !Ref AnalyticsQueue
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: sns.amazonaws.com
            Action:
              - sqs:SendMessage
            Resource: !GetAtt AnalyticsQueue.Arn
            Condition:
              ArnEquals:
                aws:SourceArn: !Ref OrderNotificationsTopic

  # ========================================
  # Topic Policy
  # ========================================
  OrderNotificationsTopicPolicy:
    Type: AWS::SNS::TopicPolicy
    Properties:
      Topics:
        - !Ref OrderNotificationsTopic
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:role/${Environment}-order-processor-role'
            Action:
              - SNS:Publish
            Resource: !Ref OrderNotificationsTopic

Outputs:
  OrderNotificationsTopicArn:
    Description: Order Notifications Topic ARN
    Value: !Ref OrderNotificationsTopic
    Export:
      Name: !Sub '${Environment}-OrderNotificationsTopic-ARN'

  AnalyticsQueueUrl:
    Description: Analytics Queue URL
    Value: !Ref AnalyticsQueue
    Export:
      Name: !Sub '${Environment}-AnalyticsQueue-URL'
```

---

## 4️⃣ Deploy SQS & SNS Stacks

```powershell
# Deploy SQS stack
aws cloudformation create-stack `
  --stack-name dev-sqs-stack `
  --template-body file://infrastructure/cloudformation/sqs-stack.yaml `
  --parameters ParameterKey=Environment,ParameterValue=dev `
  --region us-east-1

# Deploy SNS stack
aws cloudformation create-stack `
  --stack-name dev-sns-stack `
  --template-body file://infrastructure/cloudformation/sns-stack.yaml `
  --parameters ParameterKey=Environment,ParameterValue=dev `
  --region us-east-1

# Wait for completion
aws cloudformation wait stack-create-complete --stack-name dev-sqs-stack
aws cloudformation wait stack-create-complete --stack-name dev-sns-stack

# Get outputs
aws cloudformation describe-stacks --stack-name dev-sqs-stack --query 'Stacks[0].Outputs'
aws cloudformation describe-stacks --stack-name dev-sns-stack --query 'Stacks[0].Outputs'
```

---

## 5️⃣ SQS Operations - TypeScript

### Send Message to Queue

```typescript
// src/services/queue.service.ts
import { SQS } from 'aws-sdk';
import { v4 as uuidv4 } from 'uuid';

const sqs = new SQS({ region: process.env.AWS_REGION });

export interface OrderMessage {
  orderId: string;
  customerId: string;
  orderData: any;
  timestamp: string;
  retryCount?: number;
}

export async function sendOrderToQueue(
  orderMessage: OrderMessage
): Promise<string> {
  const params: SQS.SendMessageRequest = {
    QueueUrl: process.env.ORDER_QUEUE_URL!,
    MessageBody: JSON.stringify(orderMessage),
    MessageGroupId: orderMessage.customerId,  // FIFO ordering per customer
    MessageDeduplicationId: orderMessage.orderId,  // Prevent duplicates
    MessageAttributes: {
      orderId: {
        DataType: 'String',
        StringValue: orderMessage.orderId
      },
      customerId: {
        DataType: 'String',
        StringValue: orderMessage.customerId
      },
      timestamp: {
        DataType: 'Number',
        StringValue: Date.now().toString()
      }
    }
  };

  try {
    const result = await sqs.sendMessage(params).promise();
    console.log(`Message sent to queue: ${result.MessageId}`);
    return result.MessageId!;
  } catch (error) {
    console.error('Error sending message to queue:', error);
    throw error;
  }
}

export async function sendBatchToQueue(
  messages: OrderMessage[]
): Promise<void> {
  const entries: SQS.SendMessageBatchRequestEntry[] = messages.map((msg, index) => ({
    Id: index.toString(),
    MessageBody: JSON.stringify(msg),
    MessageGroupId: msg.customerId,
    MessageDeduplicationId: msg.orderId
  }));

  const params: SQS.SendMessageBatchRequest = {
    QueueUrl: process.env.ORDER_QUEUE_URL!,
    Entries: entries
  };

  try {
    const result = await sqs.sendMessageBatch(params).promise();
    console.log(`Batch sent: ${result.Successful?.length} successful, ${result.Failed?.length} failed`);
    
    if (result.Failed && result.Failed.length > 0) {
      console.error('Failed messages:', result.Failed);
    }
  } catch (error) {
    console.error('Error sending batch to queue:', error);
    throw error;
  }
}
```

### Process Messages from Queue

```typescript
// src/handlers/order-processor.ts
import { SQSEvent, SQSRecord, Context } from 'aws-lambda';
import { SQS } from 'aws-sdk';

const sqs = new SQS();

export async function handler(event: SQSEvent, context: Context) {
  console.log(`Processing ${event.Records.length} messages`);

  const batchItemFailures: { itemIdentifier: string }[] = [];

  for (const record of event.Records) {
    try {
      await processMessage(record);
    } catch (error) {
      console.error(`Error processing message ${record.messageId}:`, error);
      // Add to batch item failures for partial batch failure handling
      batchItemFailures.push({ itemIdentifier: record.messageId });
    }
  }

  // Return failed message IDs for SQS to retry
  return {
    batchItemFailures
  };
}

async function processMessage(record: SQSRecord): Promise<void> {
  const message: OrderMessage = JSON.parse(record.body);
  
  console.log(`Processing order: ${message.orderId}`);
  
  // Business logic here
  // 1. Validate order
  // 2. Check inventory
  // 3. Update order status
  // 4. Send notification
  
  // If error occurs, throw exception to trigger retry
}
```

---

## 6️⃣ SNS Operations - TypeScript

### Publish to SNS Topic

```typescript
// src/services/notification.service.ts
import { SNS } from 'aws-sdk';

const sns = new SNS({ region: process.env.AWS_REGION });

export interface OrderNotification {
  orderId: string;
  customerId: string;
  orderStatus: string;
  eventType: string;
  timestamp: string;
  metadata?: any;
}

export async function publishOrderNotification(
  notification: OrderNotification
): Promise<string> {
  const params: SNS.PublishInput = {
    TopicArn: process.env.NOTIFICATION_TOPIC_ARN!,
    Message: JSON.stringify(notification),
    Subject: `Order ${notification.orderStatus}: ${notification.orderId}`,
    MessageAttributes: {
      orderId: {
        DataType: 'String',
        StringValue: notification.orderId
      },
      orderStatus: {
        DataType: 'String',
        StringValue: notification.orderStatus
      },
      eventType: {
        DataType: 'String',
        StringValue: notification.eventType
      },
      priority: {
        DataType: 'String',
        StringValue: notification.orderStatus === 'FAILED' ? 'HIGH' : 'NORMAL'
      }
    }
  };

  try {
    const result = await sns.publish(params).promise();
    console.log(`Notification published: ${result.MessageId}`);
    return result.MessageId!;
  } catch (error) {
    console.error('Error publishing notification:', error);
    throw error;
  }
}

export async function publishBatchNotifications(
  notifications: OrderNotification[]
): Promise<void> {
  const promises = notifications.map(notification =>
    publishOrderNotification(notification)
  );

  try {
    await Promise.all(promises);
    console.log(`Published ${notifications.length} notifications`);
  } catch (error) {
    console.error('Error publishing batch notifications:', error);
    throw error;
  }
}
```

---

## 7️⃣ DLQ Handling & Retry Logic

### Redrive Messages from DLQ

```typescript
// scripts/redrive-dlq-messages.ts
import { SQS } from 'aws-sdk';

const sqs = new SQS({ region: 'us-east-1' });

async function redriveMessages(
  dlqUrl: string,
  targetQueueUrl: string,
  maxMessages: number = 10
): Promise<void> {
  console.log(`Redriving messages from DLQ to target queue...`);

  let processedCount = 0;

  while (processedCount < maxMessages) {
    // Receive messages from DLQ
    const receiveResult = await sqs.receiveMessage({
      QueueUrl: dlqUrl,
      MaxNumberOfMessages: 10,
      WaitTimeSeconds: 5
    }).promise();

    if (!receiveResult.Messages || receiveResult.Messages.length === 0) {
      console.log('No more messages in DLQ');
      break;
    }

    // Send to target queue
    for (const message of receiveResult.Messages) {
      try {
        await sqs.sendMessage({
          QueueUrl: targetQueueUrl,
          MessageBody: message.Body!,
          MessageAttributes: message.MessageAttributes
        }).promise();

        // Delete from DLQ
        await sqs.deleteMessage({
          QueueUrl: dlqUrl,
          ReceiptHandle: message.ReceiptHandle!
        }).promise();

        processedCount++;
        console.log(`✅ Redriven message ${processedCount}`);
      } catch (error) {
        console.error(`❌ Error redriving message:`, error);
      }
    }
  }

  console.log(`✨ Completed: ${processedCount} messages redriven`);
}

// Usage
const dlqUrl = process.env.ORDER_DLQ_URL!;
const targetQueueUrl = process.env.ORDER_QUEUE_URL!;
redriveMessages(dlqUrl, targetQueueUrl, 100);
```

---

## 8️⃣ Message Filtering with SNS

### Filter Policy Examples

```typescript
// Filter policy for email subscriptions
const emailFilterPolicy = {
  orderStatus: ['COMPLETED', 'CANCELLED', 'FAILED'],
  priority: ['HIGH'],
  totalAmount: [{ numeric: ['>=', 1000] }]
};

// Filter policy for analytics
const analyticsFilterPolicy = {
  eventType: [
    'ORDER_CREATED',
    'ORDER_COMPLETED',
    'ORDER_CANCELLED'
  ]
};

// Apply filter policy
async function updateSubscriptionFilter(
  subscriptionArn: string,
  filterPolicy: any
): Promise<void> {
  const sns = new SNS();
  
  await sns.setSubscriptionAttributes({
    SubscriptionArn: subscriptionArn,
    AttributeName: 'FilterPolicy',
    AttributeValue: JSON.stringify(filterPolicy)
  }).promise();
  
  console.log('Filter policy updated');
}
```

---

## 9️⃣ Monitoring & Metrics

### CloudWatch Metrics to Monitor

```javascript
const metricsToMonitor = {
  sqs: [
    'ApproximateNumberOfMessagesVisible',     // Messages in queue
    'ApproximateNumberOfMessagesNotVisible',  // In-flight messages
    'ApproximateAgeOfOldestMessage',          // Queue processing lag
    'NumberOfMessagesSent',                   // Throughput
    'NumberOfMessagesReceived',
    'NumberOfMessagesDeleted',
    'NumberOfEmptyReceives'                   // Polling efficiency
  ],
  sns: [
    'NumberOfMessagesPublished',
    'NumberOfNotificationsDelivered',
    'NumberOfNotificationsFailed'
  ]
};
```

### Create Custom Dashboard

```yaml
# Add to CloudFormation
SQSDashboard:
  Type: AWS::CloudWatch::Dashboard
  Properties:
    DashboardName: !Sub '${Environment}-sqs-monitoring'
    DashboardBody: !Sub |
      {
        "widgets": [
          {
            "type": "metric",
            "properties": {
              "metrics": [
                ["AWS/SQS", "ApproximateNumberOfMessagesVisible", {"stat": "Average"}],
                [".", "ApproximateAgeOfOldestMessage", {"stat": "Maximum"}]
              ],
              "period": 300,
              "stat": "Average",
              "region": "${AWS::Region}",
              "title": "Queue Depth & Age"
            }
          }
        ]
      }
```

---

## 🎯 Best Practices

### DO ✅

1. **Use FIFO for ordering**: When sequence matters
2. **Enable long polling**: Set ReceiveMessageWaitTimeSeconds to 20
3. **Set appropriate visibility timeout**: Match Lambda timeout + buffer
4. **Implement DLQ**: Always configure dead letter queues
5. **Use batch operations**: SendMessageBatch for efficiency
6. **Monitor DLQ depth**: Alert on any messages in DLQ
7. **Enable encryption**: Use KMS for sensitive data
8. **Use message attributes**: For filtering and routing
9. **Implement exponential backoff**: For retries
10. **Test failure scenarios**: Ensure DLQ works correctly

### DON'T ❌

1. **Don't ignore DLQ messages**: Monitor and handle failures
2. **Don't use short visibility timeout**: Causes duplicate processing
3. **Don't poll without long polling**: Wastes requests and costs
4. **Don't send large messages**: Keep under 256KB
5. **Don't skip message deduplication**: Use deduplication IDs
6. **Don't ignore message age**: Monitor processing lag
7. **Don't use standard queues when order matters**: Use FIFO
8. **Don't forget SNS filter policies**: Reduce unnecessary invocations
9. **Don't skip error handling**: Always handle exceptions
10. **Don't ignore costs**: Monitor usage and optimize

---

## 📊 Cost Analysis

```javascript
const sqsSnsCosts = {
  sqs: {
    fifo: {
      requests: '$0.50 per million after free tier',
      example: '10M requests/month = $5.00'
    },
    standard: {
      requests: '$0.40 per million after free tier',
      example: '10M requests/month = $4.00'
    },
    freeTier: '1 million requests/month'
  },
  sns: {
    publishRequests: '$0.50 per million',
    emailNotifications: '$2.00 per 100,000',
    smsNotifications: 'Varies by region ($0.00645 US)',
    example: {
      publish: '1M publishes = $0.50',
      email: '10K emails = $0.20',
      total: '$0.70/month'
    }
  },
  estimated: {
    dev: '$5-10/month',
    staging: '$20-30/month',
    production: '$50-100/month'
  }
};
```

---

## 🎯 Next Steps

SQS & SNS setup complete! You now have:

- ✅ FIFO queue for ordered processing
- ✅ Standard queue for high throughput
- ✅ Dead letter queues configured
- ✅ SNS topics for notifications
- ✅ Message filtering enabled
- ✅ CloudWatch alarms for DLQ
- ✅ TypeScript SDK operations
- ✅ Batch operations support

**Ready to proceed?**

👉 [Lambda Functions Implementation](./07_LAMBDA_FUNCTIONS.md)

---

**Last Updated**: November 12, 2025  
**Version**: 1.0.0
