# Lambda Functions - Serverless Order Processing Microservice

## 📋 Overview

This guide provides complete TypeScript implementations for all Lambda functions in the serverless order processing microservice, including error handling, retries, logging, and best practices.

---

## 🎯 Lambda Functions Architecture

### Function Overview

```javascript
const lambdaFunctions = {
  apiHandler: {
    trigger: 'API Gateway',
    purpose: 'Handle HTTP requests',
    runtime: 'nodejs18.x',
    memory: '512 MB',
    timeout: '30 seconds'
  },
  orderProcessor: {
    trigger: 'SQS (Order Queue)',
    purpose: 'Process orders asynchronously',
    runtime: 'nodejs18.x',
    memory: '1024 MB',
    timeout: '60 seconds'
  },
  inventoryUpdater: {
    trigger: 'SQS (Inventory Queue)',
    purpose: 'Update inventory levels',
    runtime: 'nodejs18.x',
    memory: '512 MB',
    timeout: '30 seconds'
  },
  notificationHandler: {
    trigger: 'SNS (Notification Topic)',
    purpose: 'Send customer notifications',
    runtime: 'nodejs18.x',
    memory: '256 MB',
    timeout: '30 seconds'
  }
};
```

---

## 1️⃣ Project Structure

```
src/
├── handlers/
│   ├── api-handler.ts
│   ├── order-processor.ts
│   ├── inventory-updater.ts
│   └── notification-handler.ts
├── services/
│   ├── order.service.ts
│   ├── inventory.service.ts
│   ├── queue.service.ts
│   └── notification.service.ts
├── models/
│   ├── order.model.ts
│   ├── inventory.model.ts
│   └── notification.model.ts
├── utils/
│   ├── dynamodb.ts
│   ├── logger.ts
│   ├── validator.ts
│   └── error-handler.ts
└── layers/
    └── common/
        └── nodejs/
            └── node_modules/
```

---

## 2️⃣ Lambda Layer - Common Dependencies

### Layer Structure

```typescript
// src/layers/common/nodejs/package.json
{
  "name": "common-layer",
  "version": "1.0.0",
  "dependencies": {
    "aws-sdk": "^2.1490.0",
    "uuid": "^9.0.1",
    "joi": "^17.11.0",
    "moment": "^2.29.4"
  }
}
```

### Build Layer

```powershell
# Create layer directory
New-Item -Path "src/layers/common/nodejs" -ItemType Directory -Force

# Install dependencies
cd src/layers/common/nodejs
npm install

# Create layer zip
cd ..
Compress-Archive -Path nodejs -DestinationPath common-layer.zip

# Upload to Lambda
aws lambda publish-layer-version `
  --layer-name dev-common-layer `
  --zip-file fileb://common-layer.zip `
  --compatible-runtimes nodejs18.x `
  --region us-east-1
```

---

## 3️⃣ Utility Functions

### Logger

```typescript
// src/utils/logger.ts
export enum LogLevel {
  DEBUG = 'DEBUG',
  INFO = 'INFO',
  WARN = 'WARN',
  ERROR = 'ERROR'
}

export class Logger {
  private context: string;

  constructor(context: string) {
    this.context = context;
  }

  private log(level: LogLevel, message: string, meta?: any): void {
    const logEntry = {
      timestamp: new Date().toISOString(),
      level,
      context: this.context,
      message,
      requestId: process.env.AWS_REQUEST_ID,
      ...(meta && { meta })
    };

    console.log(JSON.stringify(logEntry));
  }

  debug(message: string, meta?: any): void {
    if (process.env.LOG_LEVEL === 'DEBUG') {
      this.log(LogLevel.DEBUG, message, meta);
    }
  }

  info(message: string, meta?: any): void {
    this.log(LogLevel.INFO, message, meta);
  }

  warn(message: string, meta?: any): void {
    this.log(LogLevel.WARN, message, meta);
  }

  error(message: string, error?: Error, meta?: any): void {
    this.log(LogLevel.ERROR, message, {
      ...meta,
      error: error ? {
        name: error.name,
        message: error.message,
        stack: error.stack
      } : undefined
    });
  }
}
```

### Validator

```typescript
// src/utils/validator.ts
import Joi from 'joi';

export const orderSchema = Joi.object({
  customerId: Joi.string().required(),
  items: Joi.array().items(
    Joi.object({
      productId: Joi.string().required(),
      quantity: Joi.number().integer().min(1).required(),
      unitPrice: Joi.number().positive().required()
    })
  ).min(1).required(),
  shippingAddress: Joi.object({
    street: Joi.string().required(),
    city: Joi.string().required(),
    state: Joi.string().required(),
    zipCode: Joi.string().required(),
    country: Joi.string().required()
  }).required(),
  billingAddress: Joi.object({
    street: Joi.string().required(),
    city: Joi.string().required(),
    state: Joi.string().required(),
    zipCode: Joi.string().required(),
    country: Joi.string().required()
  }).required()
});

export function validateOrder(data: any): { error?: string; value?: any } {
  const { error, value } = orderSchema.validate(data, { abortEarly: false });
  
  if (error) {
    return {
      error: error.details.map(d => d.message).join(', ')
    };
  }
  
  return { value };
}
```

### Error Handler

```typescript
// src/utils/error-handler.ts
import { Logger } from './logger';

const logger = new Logger('ErrorHandler');

export class AppError extends Error {
  constructor(
    public statusCode: number,
    public message: string,
    public isOperational: boolean = true
  ) {
    super(message);
    Object.setPrototypeOf(this, AppError.prototype);
  }
}

export function handleError(error: any): {
  statusCode: number;
  body: string;
} {
  logger.error('Error occurred', error);

  if (error instanceof AppError) {
    return {
      statusCode: error.statusCode,
      body: JSON.stringify({
        error: error.message
      })
    };
  }

  // Unknown errors
  return {
    statusCode: 500,
    body: JSON.stringify({
      error: 'Internal server error'
    })
  };
}
```

---

## 4️⃣ API Handler Lambda

### Complete Implementation

```typescript
// src/handlers/api-handler.ts
import { APIGatewayProxyEvent, APIGatewayProxyResult, Context } from 'aws-lambda';
import { Logger } from '../utils/logger';
import { validateOrder } from '../utils/validator';
import { handleError, AppError } from '../utils/error-handler';
import { createOrder, getOrderById, getOrdersByCustomer } from '../services/order.service';
import { sendOrderToQueue } from '../services/queue.service';

const logger = new Logger('ApiHandler');

export async function handler(
  event: APIGatewayProxyEvent,
  context: Context
): Promise<APIGatewayProxyResult> {
  logger.info('API request received', {
    method: event.httpMethod,
    path: event.path,
    requestId: context.requestId
  });

  try {
    const method = event.httpMethod;
    const path = event.path;

    // Route requests
    if (method === 'POST' && path === '/orders') {
      return await handleCreateOrder(event);
    } else if (method === 'GET' && path.startsWith('/orders/')) {
      return await handleGetOrder(event);
    } else if (method === 'GET' && path === '/orders') {
      return await handleListOrders(event);
    } else if (method === 'GET' && path === '/health') {
      return handleHealthCheck();
    }

    throw new AppError(404, 'Not Found');
  } catch (error) {
    return handleError(error);
  }
}

async function handleCreateOrder(event: APIGatewayProxyEvent): Promise<APIGatewayProxyResult> {
  // Parse request body
  if (!event.body) {
    throw new AppError(400, 'Request body is required');
  }

  const orderData = JSON.parse(event.body);

  // Validate request
  const { error, value } = validateOrder(orderData);
  if (error) {
    throw new AppError(400, `Validation error: ${error}`);
  }

  logger.info('Creating order', { customerId: value.customerId });

  // Calculate total amount
  const totalAmount = value.items.reduce(
    (sum: number, item: any) => sum + (item.quantity * item.unitPrice),
    0
  );

  // Create order in DynamoDB
  const order = await createOrder({
    ...value,
    totalAmount,
    currency: 'USD'
  });

  logger.info('Order created', { orderId: order.orderId });

  // Send to SQS for async processing
  await sendOrderToQueue({
    orderId: order.orderId,
    customerId: order.customerId,
    orderData: order,
    timestamp: new Date().toISOString()
  });

  logger.info('Order sent to processing queue', { orderId: order.orderId });

  return {
    statusCode: 201,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify({
      message: 'Order created successfully',
      order: {
        orderId: order.orderId,
        orderStatus: order.orderStatus,
        totalAmount: order.totalAmount,
        createdAt: order.createdAt
      }
    })
  };
}

async function handleGetOrder(event: APIGatewayProxyEvent): Promise<APIGatewayProxyResult> {
  const orderId = event.pathParameters?.orderId;
  
  if (!orderId) {
    throw new AppError(400, 'Order ID is required');
  }

  logger.info('Fetching order', { orderId });

  const order = await getOrderById(orderId);

  if (!order) {
    throw new AppError(404, 'Order not found');
  }

  return {
    statusCode: 200,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify({ order })
  };
}

async function handleListOrders(event: APIGatewayProxyEvent): Promise<APIGatewayProxyResult> {
  const customerId = event.queryStringParameters?.customerId;
  
  if (!customerId) {
    throw new AppError(400, 'Customer ID is required');
  }

  const limit = parseInt(event.queryStringParameters?.limit || '20');
  const lastKey = event.queryStringParameters?.nextToken 
    ? JSON.parse(Buffer.from(event.queryStringParameters.nextToken, 'base64').toString())
    : undefined;

  logger.info('Fetching orders', { customerId, limit });

  const { orders, lastEvaluatedKey } = await getOrdersByCustomer(customerId, limit, lastKey);

  const nextToken = lastEvaluatedKey 
    ? Buffer.from(JSON.stringify(lastEvaluatedKey)).toString('base64')
    : undefined;

  return {
    statusCode: 200,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify({
      orders,
      nextToken,
      count: orders.length
    })
  };
}

function handleHealthCheck(): APIGatewayProxyResult {
  return {
    statusCode: 200,
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      status: 'healthy',
      timestamp: new Date().toISOString(),
      service: 'order-processing-api'
    })
  };
}
```

---

## 5️⃣ Order Processor Lambda

```typescript
// src/handlers/order-processor.ts
import { SQSEvent, SQSRecord, Context } from 'aws-lambda';
import { Logger } from '../utils/logger';
import { updateOrderStatus } from '../services/order.service';
import { checkInventory, reserveInventory } from '../services/inventory.service';
import { publishOrderNotification } from '../services/notification.service';

const logger = new Logger('OrderProcessor');

export async function handler(event: SQSEvent, context: Context) {
  logger.info('Processing SQS messages', {
    messageCount: event.Records.length,
    requestId: context.requestId
  });

  const batchItemFailures: { itemIdentifier: string }[] = [];

  for (const record of event.Records) {
    try {
      await processOrder(record);
    } catch (error) {
      logger.error('Failed to process order', error as Error, {
        messageId: record.messageId
      });
      batchItemFailures.push({ itemIdentifier: record.messageId });
    }
  }

  logger.info('Batch processing complete', {
    total: event.Records.length,
    failures: batchItemFailures.length
  });

  // Return failed items for partial batch failure handling
  return { batchItemFailures };
}

async function processOrder(record: SQSRecord): Promise<void> {
  const message = JSON.parse(record.body);
  const { orderId, customerId, orderData } = message;

  logger.info('Processing order', { orderId, customerId });

  try {
    // Step 1: Validate order
    if (!orderId || !orderData) {
      throw new Error('Invalid order message');
    }

    // Step 2: Check inventory for all items
    const inventoryChecks = await Promise.all(
      orderData.items.map(async (item: any) => {
        const inventory = await checkInventory(item.productId);
        
        if (!inventory) {
          throw new Error(`Product not found: ${item.productId}`);
        }
        
        if (inventory.availableQuantity < item.quantity) {
          throw new Error(
            `Insufficient inventory for ${item.productId}: ` +
            `required ${item.quantity}, available ${inventory.availableQuantity}`
          );
        }
        
        return { productId: item.productId, quantity: item.quantity, inventory };
      })
    );

    logger.info('Inventory check passed', { orderId });

    // Step 3: Reserve inventory
    for (const { productId, quantity, inventory } of inventoryChecks) {
      await reserveInventory(productId, quantity, inventory.version);
      logger.info('Inventory reserved', { orderId, productId, quantity });
    }

    // Step 4: Update order status to PROCESSING
    await updateOrderStatus(orderId, 'PROCESSING', orderData.version);
    logger.info('Order status updated to PROCESSING', { orderId });

    // Step 5: Simulate payment processing
    await simulatePaymentProcessing(orderId);

    // Step 6: Update order status to COMPLETED
    await updateOrderStatus(orderId, 'COMPLETED', orderData.version + 1);
    logger.info('Order status updated to COMPLETED', { orderId });

    // Step 7: Send notification
    await publishOrderNotification({
      orderId,
      customerId,
      orderStatus: 'COMPLETED',
      eventType: 'ORDER_COMPLETED',
      timestamp: new Date().toISOString(),
      metadata: {
        totalAmount: orderData.totalAmount,
        itemCount: orderData.items.length
      }
    });

    logger.info('Order processed successfully', { orderId });
  } catch (error) {
    logger.error('Order processing failed', error as Error, { orderId });

    // Update order status to FAILED
    try {
      await updateOrderStatus(orderId, 'FAILED', orderData.version);
      
      // Send failure notification
      await publishOrderNotification({
        orderId,
        customerId,
        orderStatus: 'FAILED',
        eventType: 'ORDER_FAILED',
        timestamp: new Date().toISOString(),
        metadata: {
          error: (error as Error).message
        }
      });
    } catch (updateError) {
      logger.error('Failed to update order status', updateError as Error, { orderId });
    }

    throw error;
  }
}

async function simulatePaymentProcessing(orderId: string): Promise<void> {
  // Simulate payment gateway call
  logger.info('Processing payment', { orderId });
  
  return new Promise((resolve) => {
    setTimeout(() => {
      logger.info('Payment processed', { orderId });
      resolve();
    }, 1000);
  });
}
```

---

## 6️⃣ Inventory Updater Lambda

```typescript
// src/handlers/inventory-updater.ts
import { SQSEvent, Context } from 'aws-lambda';
import { Logger } from '../utils/logger';
import { updateInventoryQuantity } from '../services/inventory.service';

const logger = new Logger('InventoryUpdater');

export async function handler(event: SQSEvent, context: Context) {
  logger.info('Processing inventory updates', {
    messageCount: event.Records.length,
    requestId: context.requestId
  });

  const batchItemFailures: { itemIdentifier: string }[] = [];

  for (const record of event.Records) {
    try {
      const message = JSON.parse(record.body);
      await updateInventory(message);
    } catch (error) {
      logger.error('Failed to update inventory', error as Error, {
        messageId: record.messageId
      });
      batchItemFailures.push({ itemIdentifier: record.messageId });
    }
  }

  return { batchItemFailures };
}

async function updateInventory(message: any): Promise<void> {
  const { productId, quantity, operation } = message;

  logger.info('Updating inventory', { productId, quantity, operation });

  if (operation === 'RESERVE') {
    // Already handled in order-processor
    logger.debug('Inventory reservation', { productId, quantity });
  } else if (operation === 'RELEASE') {
    await updateInventoryQuantity(productId, quantity, 'ADD');
    logger.info('Inventory released', { productId, quantity });
  } else if (operation === 'DEDUCT') {
    await updateInventoryQuantity(productId, quantity, 'SUBTRACT');
    logger.info('Inventory deducted', { productId, quantity });
  }
}
```

---

## 7️⃣ Notification Handler Lambda

```typescript
// src/handlers/notification-handler.ts
import { SNSEvent, Context } from 'aws-lambda';
import { Logger } from '../utils/logger';
import { SES } from 'aws-sdk';

const logger = new Logger('NotificationHandler');
const ses = new SES({ region: process.env.AWS_REGION });

export async function handler(event: SNSEvent, context: Context) {
  logger.info('Processing notifications', {
    recordCount: event.Records.length,
    requestId: context.requestId
  });

  for (const record of event.Records) {
    try {
      const message = JSON.parse(record.Sns.Message);
      await sendNotification(message);
    } catch (error) {
      logger.error('Failed to send notification', error as Error, {
        messageId: record.Sns.MessageId
      });
    }
  }
}

async function sendNotification(notification: any): Promise<void> {
  const { orderId, customerId, orderStatus, eventType, metadata } = notification;

  logger.info('Sending notification', { orderId, customerId, eventType });

  // Get customer email (in real app, fetch from customer service)
  const customerEmail = `customer-${customerId}@example.com`;

  // Prepare email
  const emailParams: SES.SendEmailRequest = {
    Source: process.env.FROM_EMAIL || 'noreply@example.com',
    Destination: {
      ToAddresses: [customerEmail]
    },
    Message: {
      Subject: {
        Data: getEmailSubject(orderStatus, orderId)
      },
      Body: {
        Html: {
          Data: getEmailBody(orderStatus, orderId, metadata)
        }
      }
    }
  };

  try {
    await ses.sendEmail(emailParams).promise();
    logger.info('Email sent successfully', { orderId, customerEmail });
  } catch (error) {
    logger.error('Failed to send email', error as Error, { orderId });
    throw error;
  }
}

function getEmailSubject(orderStatus: string, orderId: string): string {
  const subjects: Record<string, string> = {
    COMPLETED: `Order Confirmed - ${orderId}`,
    FAILED: `Order Failed - ${orderId}`,
    CANCELLED: `Order Cancelled - ${orderId}`,
    SHIPPED: `Order Shipped - ${orderId}`
  };
  
  return subjects[orderStatus] || `Order Update - ${orderId}`;
}

function getEmailBody(orderStatus: string, orderId: string, metadata: any): string {
  return `
    <html>
      <body>
        <h2>Order ${orderStatus}</h2>
        <p>Order ID: ${orderId}</p>
        <p>Status: ${orderStatus}</p>
        ${metadata?.totalAmount ? `<p>Total: $${metadata.totalAmount}</p>` : ''}
        <p>Thank you for your order!</p>
      </body>
    </html>
  `;
}
```

---

## 8️⃣ Serverless Framework Configuration

```yaml
# serverless.yml
service: order-processing

provider:
  name: aws
  runtime: nodejs18.x
  region: us-east-1
  stage: ${opt:stage, 'dev'}
  memorySize: 512
  timeout: 30
  
  environment:
    STAGE: ${self:provider.stage}
    ORDERS_TABLE: ${self:provider.stage}-orders
    INVENTORY_TABLE: ${self:provider.stage}-inventory
    ORDER_QUEUE_URL: !Ref OrderProcessingQueue
    NOTIFICATION_TOPIC_ARN: !Ref OrderNotificationsTopic
    LOG_LEVEL: ${self:custom.logLevel.${self:provider.stage}}
  
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:GetItem
            - dynamodb:PutItem
            - dynamodb:UpdateItem
            - dynamodb:Query
          Resource:
            - !GetAtt OrdersTable.Arn
            - !GetAtt InventoryTable.Arn
        
        - Effect: Allow
          Action:
            - sqs:SendMessage
            - sqs:ReceiveMessage
            - sqs:DeleteMessage
          Resource:
            - !GetAtt OrderProcessingQueue.Arn
        
        - Effect: Allow
          Action:
            - sns:Publish
          Resource:
            - !Ref OrderNotificationsTopic

custom:
  logLevel:
    dev: DEBUG
    staging: INFO
    production: ERROR

functions:
  apiHandler:
    handler: dist/handlers/api-handler.handler
    events:
      - http:
          path: /orders
          method: post
          cors: true
      - http:
          path: /orders
          method: get
          cors: true
      - http:
          path: /orders/{orderId}
          method: get
          cors: true
      - http:
          path: /health
          method: get
    layers:
      - !Ref CommonLayer

  orderProcessor:
    handler: dist/handlers/order-processor.handler
    memorySize: 1024
    timeout: 60
    events:
      - sqs:
          arn: !GetAtt OrderProcessingQueue.Arn
          batchSize: 10
          maximumBatchingWindowInSeconds: 5
          functionResponseType: ReportBatchItemFailures
    layers:
      - !Ref CommonLayer

  inventoryUpdater:
    handler: dist/handlers/inventory-updater.handler
    layers:
      - !Ref CommonLayer

  notificationHandler:
    handler: dist/handlers/notification-handler.handler
    memorySize: 256
    events:
      - sns:
          arn: !Ref OrderNotificationsTopic
    layers:
      - !Ref CommonLayer

layers:
  common:
    path: src/layers/common
    compatibleRuntimes:
      - nodejs18.x

resources:
  Resources:
    OrderProcessingQueue:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: ${self:provider.stage}-order-processing.fifo
        FifoQueue: true
        ContentBasedDeduplication: true

    OrderNotificationsTopic:
      Type: AWS::SNS::Topic
      Properties:
        TopicName: ${self:provider.stage}-order-notifications

    OrdersTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${self:provider.stage}-orders
        BillingMode: PAY_PER_REQUEST
        AttributeDefinitions:
          - AttributeName: orderId
            AttributeType: S
        KeySchema:
          - AttributeName: orderId
            KeyType: HASH

    InventoryTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${self:provider.stage}-inventory
        BillingMode: PAY_PER_REQUEST
        AttributeDefinitions:
          - AttributeName: productId
            AttributeType: S
        KeySchema:
          - AttributeName: productId
            KeyType: HASH

plugins:
  - serverless-plugin-typescript
  - serverless-offline
```

---

## 9️⃣ Deploy Lambda Functions

```powershell
# Build TypeScript
npm run build

# Deploy to dev
serverless deploy --stage dev

# Deploy specific function
serverless deploy function --function apiHandler --stage dev

# View logs
serverless logs --function apiHandler --stage dev --tail

# Invoke function locally
serverless invoke local --function apiHandler --path test/events/create-order.json

# Remove deployment
serverless remove --stage dev
```

---

## 🔟 Testing Lambda Functions

### Unit Tests

```typescript
// test/handlers/api-handler.test.ts
import { handler } from '../../src/handlers/api-handler';
import { APIGatewayProxyEvent, Context } from 'aws-lambda';

describe('API Handler', () => {
  it('should create order successfully', async () => {
    const event: Partial<APIGatewayProxyEvent> = {
      httpMethod: 'POST',
      path: '/orders',
      body: JSON.stringify({
        customerId: 'CUST-001',
        items: [
          {
            productId: 'PROD-001',
            quantity: 2,
            unitPrice: 100
          }
        ],
        shippingAddress: {
          street: '123 Main St',
          city: 'New York',
          state: 'NY',
          zipCode: '10001',
          country: 'USA'
        },
        billingAddress: {
          street: '123 Main St',
          city: 'New York',
          state: 'NY',
          zipCode: '10001',
          country: 'USA'
        }
      })
    };

    const context: Partial<Context> = {
      requestId: 'test-request-id'
    };

    const result = await handler(
      event as APIGatewayProxyEvent,
      context as Context
    );

    expect(result.statusCode).toBe(201);
    const body = JSON.parse(result.body);
    expect(body.order).toBeDefined();
    expect(body.order.orderId).toBeDefined();
  });
});
```

---

## 🎯 Best Practices

### DO ✅

1. **Use TypeScript**: Type safety and better IDE support
2. **Implement structured logging**: Use JSON format
3. **Handle errors gracefully**: Return proper HTTP status codes
4. **Use Lambda layers**: Share common dependencies
5. **Set appropriate timeouts**: Match function complexity
6. **Enable X-Ray tracing**: For debugging
7. **Use environment variables**: For configuration
8. **Implement health checks**: Monitor function health
9. **Use async/await**: For cleaner code
10. **Test thoroughly**: Unit and integration tests

### DON'T ❌

1. **Don't put secrets in code**: Use Secrets Manager
2. **Don't ignore cold starts**: Implement warmup if needed
3. **Don't use synchronous calls**: Use async patterns
4. **Don't over-allocate memory**: Right-size functions
5. **Don't ignore errors**: Always log and handle
6. **Don't use large dependencies**: Minimize package size
7. **Don't skip validation**: Validate all inputs
8. **Don't hardcode values**: Use environment variables
9. **Don't forget retries**: Implement exponential backoff
10. **Don't skip monitoring**: Use CloudWatch metrics

---

## 🎯 Next Steps

Lambda functions implementation complete! You now have:

- ✅ Complete TypeScript implementations
- ✅ Error handling and logging
- ✅ Input validation
- ✅ Async processing with SQS
- ✅ Notification handling with SNS
- ✅ Serverless Framework configuration
- ✅ Unit test examples

**Ready to proceed?**

👉 [API Gateway Setup](./08_API_GATEWAY_SETUP.md)

---

**Last Updated**: November 12, 2025  
**Version**: 1.0.0
