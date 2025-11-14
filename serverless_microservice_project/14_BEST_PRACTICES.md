# Best Practices - Serverless Order Processing Microservice

## 📋 Overview

Production-ready best practices for building, deploying, and operating serverless microservices on AWS.

---

## 1️⃣ Lambda Function Best Practices

### Memory & Performance Optimization

```typescript
// Optimize cold starts
// ❌ BAD: Initialize inside handler
export const handler = async (event: any) => {
  const docClient = new DynamoDB.DocumentClient();  // Created every invocation
  return await processOrder(docClient, event);
};

// ✅ GOOD: Initialize outside handler
const docClient = new DynamoDB.DocumentClient();
const sqs = new SQS();

export const handler = async (event: any) => {
  return await processOrder(docClient, event);
};
```

### Right-Sizing Memory

```powershell
# Use AWS Lambda Power Tuning
# https://github.com/alexcasalboni/aws-lambda-power-tuning

aws lambda invoke `
    --function-name order-processing-dev-api `
    --payload '{"test": "payload"}' `
    response.json

# Test different memory configurations
$memoryConfigs = @(256, 512, 1024, 1536, 2048)
foreach ($memory in $memoryConfigs) {
    aws lambda update-function-configuration `
        --function-name order-processing-dev-api `
        --memory-size $memory
    
    # Run performance test
    # Measure: duration, cost, latency
}
```

### Error Handling

```typescript
// Comprehensive error handling
import { Logger } from '../utils/logger';
const logger = new Logger({ function: 'orderProcessor' });

export const handler = async (event: SQSEvent) => {
  const batchItemFailures = [];
  
  for (const record of event.Records) {
    try {
      await processRecord(record);
    } catch (error: any) {
      logger.error('Failed to process record', error, {
        messageId: record.messageId,
        body: record.body
      });
      
      // Determine if error is retryable
      if (isRetryable(error)) {
        batchItemFailures.push({ itemIdentifier: record.messageId });
      } else {
        // Send to DLQ or handle permanently
        await sendToDLQ(record, error);
      }
    }
  }
  
  return { batchItemFailures };
};

function isRetryable(error: Error): boolean {
  const retryableErrors = [
    'ProvisionedThroughputExceededException',
    'ServiceUnavailable',
    'ThrottlingException'
  ];
  return retryableErrors.includes(error.name);
}
```

### Environment Variables

```yaml
# serverless.yml - Structured environment variables
provider:
  environment:
    # System
    STAGE: ${self:provider.stage}
    AWS_NODEJS_CONNECTION_REUSE_ENABLED: '1'
    
    # DynamoDB
    ORDERS_TABLE: !Ref OrdersTable
    INVENTORY_TABLE: !Ref InventoryTable
    
    # Queues
    ORDER_QUEUE_URL: !Ref OrderQueue
    INVENTORY_QUEUE_URL: !Ref InventoryQueue
    
    # Topics
    NOTIFICATION_TOPIC_ARN: !Ref NotificationTopic
    
    # Feature Flags
    ENABLE_FRAUD_CHECK: ${self:custom.stages.${self:provider.stage}.enableFraudCheck}
    
    # Timeouts
    PAYMENT_TIMEOUT_MS: '5000'
    INVENTORY_TIMEOUT_MS: '3000'
```

### Lambda Layers

```yaml
# serverless.yml
layers:
  commonLibs:
    path: layers/common
    name: ${self:service}-common-libs
    description: Shared utilities and dependencies
    compatibleRuntimes:
      - nodejs18.x

functions:
  api:
    handler: dist/handlers/api.handler
    layers:
      - !Ref CommonLibsLambdaLayer
```

```
# Directory structure
layers/
  common/
    nodejs/
      node_modules/
        aws-sdk/
        uuid/
        joi/
      utils/
        logger.ts
        validator.ts
```

---

## 2️⃣ API Gateway Best Practices

### Request Validation

```yaml
# CloudFormation
OrderRequestModel:
  Type: AWS::ApiGateway::Model
  Properties:
    RestApiId: !Ref ApiGateway
    Name: OrderRequest
    ContentType: application/json
    Schema:
      $schema: http://json-schema.org/draft-04/schema#
      type: object
      required:
        - customerId
        - items
      properties:
        customerId:
          type: string
          pattern: ^CUST-[A-Z0-9]+$
        items:
          type: array
          minItems: 1
          items:
            type: object
            required:
              - productId
              - quantity
              - price
            properties:
              productId:
                type: string
              quantity:
                type: integer
                minimum: 1
              price:
                type: number
                minimum: 0

CreateOrderMethod:
  Type: AWS::ApiGateway::Method
  Properties:
    RequestValidatorId: !Ref RequestValidator
    RequestModels:
      application/json: !Ref OrderRequestModel
```

### Throttling Strategy

```yaml
# Stage-specific throttling
ApiGatewayStage:
  Type: AWS::ApiGateway::Stage
  Properties:
    StageName: ${Stage}
    RestApiId: !Ref ApiGateway
    MethodSettings:
      - ResourcePath: /orders
        HttpMethod: POST
        ThrottlingBurstLimit: 1000
        ThrottlingRateLimit: 500
      - ResourcePath: /orders/*
        HttpMethod: GET
        ThrottlingBurstLimit: 2000
        ThrottlingRateLimit: 1000
```

### Usage Plans & API Keys

```yaml
UsagePlan:
  Type: AWS::ApiGateway::UsagePlan
  Properties:
    UsagePlanName: ${Stage}-standard-plan
    Quota:
      Limit: 100000
      Period: MONTH
    Throttle:
      BurstLimit: 500
      RateLimit: 100
    ApiStages:
      - ApiId: !Ref ApiGateway
        Stage: !Ref ApiGatewayStage

ApiKey:
  Type: AWS::ApiGateway::ApiKey
  Properties:
    Name: ${Stage}-partner-api-key
    Enabled: true

UsagePlanKey:
  Type: AWS::ApiGateway::UsagePlanKey
  Properties:
    KeyId: !Ref ApiKey
    KeyType: API_KEY
    UsagePlanId: !Ref UsagePlan
```

### CORS Configuration

```typescript
// Standardized CORS headers
export function corsHeaders(origin?: string): Record<string, string> {
  const allowedOrigins = [
    'https://app.example.com',
    'https://staging.example.com'
  ];
  
  const allowOrigin = allowedOrigins.includes(origin || '')
    ? origin!
    : allowedOrigins[0];
  
  return {
    'Access-Control-Allow-Origin': allowOrigin,
    'Access-Control-Allow-Credentials': 'true',
    'Access-Control-Allow-Methods': 'GET,POST,PUT,DELETE,OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Correlation-ID',
    'Access-Control-Max-Age': '3600'
  };
}
```

---

## 3️⃣ DynamoDB Best Practices

### Single Table Design

```typescript
// Single table for related entities
interface BaseEntity {
  PK: string;       // Partition Key
  SK: string;       // Sort Key
  GSI1PK?: string;  // Global Secondary Index 1 PK
  GSI1SK?: string;  // Global Secondary Index 1 SK
  EntityType: string;
}

// Order entity
const order: BaseEntity = {
  PK: 'ORDER#ORDER-123',
  SK: 'METADATA',
  GSI1PK: 'CUSTOMER#CUST-123',
  GSI1SK: 'ORDER#2024-11-12',
  EntityType: 'Order',
  // ... other attributes
};

// Order item
const orderItem: BaseEntity = {
  PK: 'ORDER#ORDER-123',
  SK: 'ITEM#1',
  EntityType: 'OrderItem',
  // ... other attributes
};

// Customer entity
const customer: BaseEntity = {
  PK: 'CUSTOMER#CUST-123',
  SK: 'METADATA',
  GSI1PK: 'EMAIL#customer@example.com',
  GSI1SK: 'CUSTOMER',
  EntityType: 'Customer'
};
```

### Optimistic Locking

```typescript
// Prevent concurrent updates
async function updateInventory(
  productId: string,
  quantityChange: number
): Promise<void> {
  let retries = 3;
  
  while (retries > 0) {
    try {
      // Get current version
      const result = await docClient.get({
        TableName: 'inventory',
        Key: { productId }
      }).promise();
      
      const currentVersion = result.Item?.version || 0;
      const currentQuantity = result.Item?.quantity || 0;
      
      // Update with version check
      await docClient.update({
        TableName: 'inventory',
        Key: { productId },
        UpdateExpression: 'SET quantity = :newQty, version = :newVersion',
        ConditionExpression: 'version = :currentVersion',
        ExpressionAttributeValues: {
          ':newQty': currentQuantity + quantityChange,
          ':newVersion': currentVersion + 1,
          ':currentVersion': currentVersion
        }
      }).promise();
      
      return;
    } catch (error: any) {
      if (error.code === 'ConditionalCheckFailedException') {
        retries--;
        await sleep(100 * (4 - retries));  // Exponential backoff
      } else {
        throw error;
      }
    }
  }
  
  throw new Error('Failed to update after retries');
}
```

### Batch Operations

```typescript
// Efficient batch reads
async function getMultipleOrders(orderIds: string[]): Promise<any[]> {
  const batchSize = 100;  // DynamoDB limit
  const batches = [];
  
  for (let i = 0; i < orderIds.length; i += batchSize) {
    const batch = orderIds.slice(i, i + batchSize);
    batches.push(batch);
  }
  
  const results = await Promise.all(
    batches.map(batch =>
      docClient.batchGet({
        RequestItems: {
          'orders': {
            Keys: batch.map(id => ({ orderId: id })),
            ProjectionExpression: 'orderId, customerId, status, total'
          }
        }
      }).promise()
    )
  );
  
  return results.flatMap(r => r.Responses?.orders || []);
}
```

### GSI Overloading

```yaml
# Single GSI for multiple access patterns
OrdersTable:
  Type: AWS::DynamoDB::Table
  Properties:
    GlobalSecondaryIndexes:
      - IndexName: GSI1
        KeySchema:
          - AttributeName: GSI1PK
            KeyType: HASH
          - AttributeName: GSI1SK
            KeyType: RANGE
        Projection:
          ProjectionType: ALL
```

```typescript
// Multiple query patterns with single GSI
// Pattern 1: Orders by customer
const customerOrders = await docClient.query({
  TableName: 'orders',
  IndexName: 'GSI1',
  KeyConditionExpression: 'GSI1PK = :customerId',
  ExpressionAttributeValues: {
    ':customerId': 'CUSTOMER#CUST-123'
  }
}).promise();

// Pattern 2: Orders by status
const pendingOrders = await docClient.query({
  TableName: 'orders',
  IndexName: 'GSI1',
  KeyConditionExpression: 'GSI1PK = :status',
  ExpressionAttributeValues: {
    ':status': 'STATUS#PENDING'
  }
}).promise();
```

---

## 4️⃣ Security Best Practices

### Secrets Management

```typescript
// src/utils/secrets.ts
import { SecretsManager } from 'aws-sdk';

const secretsManager = new SecretsManager();
const cache: Map<string, { value: any; expiry: number }> = new Map();

export async function getSecret(secretName: string): Promise<any> {
  // Check cache (TTL: 5 minutes)
  const cached = cache.get(secretName);
  if (cached && cached.expiry > Date.now()) {
    return cached.value;
  }
  
  // Fetch from Secrets Manager
  const result = await secretsManager.getSecretValue({
    SecretId: secretName
  }).promise();
  
  const value = JSON.parse(result.SecretString || '{}');
  
  // Cache with TTL
  cache.set(secretName, {
    value,
    expiry: Date.now() + (5 * 60 * 1000)
  });
  
  return value;
}
```

### Least Privilege IAM

```yaml
# Function-specific IAM roles
ApiLambdaRole:
  Type: AWS::IAM::Role
  Properties:
    AssumeRolePolicyDocument:
      Version: '2012-10-17'
      Statement:
        - Effect: Allow
          Principal:
            Service: lambda.amazonaws.com
          Action: sts:AssumeRole
    ManagedPolicyArns:
      - arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole
    Policies:
      - PolicyName: ApiLambdaPolicy
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            # Only what this function needs
            - Effect: Allow
              Action:
                - dynamodb:PutItem
                - dynamodb:GetItem
              Resource: !GetAtt OrdersTable.Arn
            - Effect: Allow
              Action:
                - sqs:SendMessage
              Resource: !GetAtt OrderQueue.Arn
```

### Input Validation

```typescript
// src/utils/validator.ts
import Joi from 'joi';

const orderSchema = Joi.object({
  customerId: Joi.string()
    .pattern(/^CUST-[A-Z0-9]+$/)
    .required(),
  items: Joi.array()
    .items(Joi.object({
      productId: Joi.string().required(),
      quantity: Joi.number().integer().min(1).max(100).required(),
      price: Joi.number().positive().precision(2).required()
    }))
    .min(1)
    .max(50)
    .required(),
  notes: Joi.string().max(500).optional()
});

export function validateOrder(data: any): { error?: string; value?: any } {
  const { error, value } = orderSchema.validate(data, {
    abortEarly: false,
    stripUnknown: true
  });
  
  if (error) {
    return {
      error: error.details.map(d => d.message).join(', ')
    };
  }
  
  return { value };
}
```

### Encryption

```yaml
# CloudFormation - Encryption at rest
OrdersTable:
  Type: AWS::DynamoDB::Table
  Properties:
    SSESpecification:
      SSEEnabled: true
      SSEType: KMS
      KMSMasterKeyId: !Ref DataEncryptionKey

OrderQueue:
  Type: AWS::SQS::Queue
  Properties:
    KmsMasterKeyId: !Ref QueueEncryptionKey
    KmsDataKeyReusePeriodSeconds: 300

S3Bucket:
  Type: AWS::S3::Bucket
  Properties:
    BucketEncryption:
      ServerSideEncryptionConfiguration:
        - ServerSideEncryptionByDefault:
            SSEAlgorithm: aws:kms
            KMSMasterKeyID: !Ref S3EncryptionKey
```

---

## 5️⃣ Monitoring & Observability

### Custom Metrics

```typescript
// src/utils/metrics.ts
import { CloudWatch } from 'aws-sdk';

const cloudwatch = new CloudWatch();

export class Metrics {
  private namespace: string = 'OrderProcessing';
  
  async publishMetric(
    metricName: string,
    value: number,
    unit: string = 'Count',
    dimensions?: Array<{ Name: string; Value: string }>
  ): Promise<void> {
    await cloudwatch.putMetricData({
      Namespace: this.namespace,
      MetricData: [{
        MetricName: metricName,
        Value: value,
        Unit: unit,
        Timestamp: new Date(),
        Dimensions: dimensions || []
      }]
    }).promise();
  }
  
  async publishOrderCreated(orderId: string, total: number): Promise<void> {
    await Promise.all([
      this.publishMetric('OrderCreated', 1, 'Count'),
      this.publishMetric('OrderValue', total, 'None', [
        { Name: 'OrderId', Value: orderId }
      ])
    ]);
  }
}
```

### Structured Logging

```typescript
// Consistent log format
interface LogEntry {
  level: 'DEBUG' | 'INFO' | 'WARN' | 'ERROR';
  message: string;
  timestamp: string;
  correlationId: string;
  service: string;
  stage: string;
  metadata?: Record<string, any>;
}

export class Logger {
  log(level: string, message: string, metadata?: any) {
    const entry: LogEntry = {
      level: level as any,
      message,
      timestamp: new Date().toISOString(),
      correlationId: CorrelationContext.get(),
      service: 'order-processing',
      stage: process.env.STAGE || 'dev',
      metadata
    };
    
    console.log(JSON.stringify(entry));
  }
}
```

---

## 6️⃣ Testing Best Practices

### Unit Test Coverage

```typescript
// tests/unit/orderService.test.ts
describe('OrderService', () => {
  describe('createOrder', () => {
    it('should create order with valid data', async () => { /* ... */ });
    it('should throw error for invalid customer ID', async () => { /* ... */ });
    it('should calculate total correctly', async () => { /* ... */ });
    it('should handle DynamoDB errors', async () => { /* ... */ });
  });
  
  describe('updateOrderStatus', () => {
    it('should update status successfully', async () => { /* ... */ });
    it('should prevent invalid status transitions', async () => { /* ... */ });
    it('should handle concurrent updates', async () => { /* ... */ });
  });
});
```

### Integration Tests

```typescript
// Test complete workflows
it('should process order end-to-end', async () => {
  // 1. Create order
  const order = await createOrder(testData);
  
  // 2. Verify in DynamoDB
  const dbOrder = await getOrderFromDB(order.orderId);
  expect(dbOrder.status).toBe('PENDING');
  
  // 3. Process message
  await processOrderMessage(order.orderId);
  
  // 4. Verify final state
  const finalOrder = await getOrderFromDB(order.orderId);
  expect(finalOrder.status).toBe('COMPLETED');
});
```

---

## 🎯 Production Readiness Checklist

### Infrastructure ✅
- [ ] Multi-AZ deployment
- [ ] VPC with private subnets
- [ ] VPC endpoints configured
- [ ] Auto-scaling enabled
- [ ] Backup and disaster recovery
- [ ] Resource tagging complete

### Security ✅
- [ ] Secrets in Secrets Manager
- [ ] KMS encryption enabled
- [ ] IAM least privilege
- [ ] CloudTrail logging
- [ ] GuardDuty enabled
- [ ] WAF rules configured

### Monitoring ✅
- [ ] CloudWatch dashboards
- [ ] Alarms for critical metrics
- [ ] X-Ray tracing enabled
- [ ] Custom business metrics
- [ ] Log aggregation
- [ ] PagerDuty integration

### Development ✅
- [ ] Unit test coverage >80%
- [ ] Integration tests
- [ ] E2E smoke tests
- [ ] Load testing completed
- [ ] Security scanning
- [ ] Code review process

### Operations ✅
- [ ] Runbooks documented
- [ ] Incident response plan
- [ ] Rollback procedures
- [ ] On-call rotation
- [ ] SLA defined
- [ ] Cost budget alerts

---

**Next:** [Cost Optimization](./15_COST_OPTIMIZATION.md)

---

**Last Updated**: November 12, 2025
