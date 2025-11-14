 # Architecture Design - Serverless Order Processing Microservice

## 🎨 System Architecture Overview

### High-Level Architecture Diagram

```
                                    ┌─────────────────────────────────┐
                                    │   External Users/Systems        │
                                    │   (Web, Mobile, 3rd Party)      │
                                    └──────────────┬──────────────────┘
                                                   │
                                                   │ HTTPS
                                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                              AWS CLOUD (VPC)                                  │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                          API Gateway (REST)                          │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐  │   │
│  │  │ /orders    │  │ /orders/id │  │ /health    │  │ /metrics   │  │   │
│  │  │ POST       │  │ GET/PUT    │  │ GET        │  │ GET        │  │   │
│  │  └────────────┘  └────────────┘  └────────────┘  └────────────┘  │   │
│  │                                                                     │   │
│  │  Features: Throttling, Caching, API Keys, CORS, Request Validation│   │
│  └──────────────────┬──────────────────────────────────────────────┬─┘   │
│                     │                                                │      │
│                     │                                                │      │
│  ┌──────────────────▼──────────────────┐    ┌─────────────────────▼─┐   │
│  │   Lambda: API Handler                │    │  Lambda: Health Check  │   │
│  │   ┌──────────────────────────────┐  │    │  - System status       │   │
│  │   │ - Validate request           │  │    │  - Service health      │   │
│  │   │ - Check inventory            │  │    │  - Dependencies check  │   │
│  │   │ - Save to DynamoDB           │  │    └────────────────────────┘   │
│  │   │ - Send to SQS                │  │                                  │
│  │   │ - Return response            │  │                                  │
│  │   └──────────────────────────────┘  │                                  │
│  └──────────┬────────────────┬──────────┘                                  │
│             │                │                                              │
│             │                │                                              │
│     ┌───────▼─────┐   ┌─────▼──────────┐                                  │
│     │  DynamoDB   │   │  SQS Queue     │                                  │
│     │  Orders     │   │  (FIFO)        │                                  │
│     │  Table      │   │                │                                  │
│     └─────────────┘   │  Features:     │                                  │
│                       │  - Dedup       │                                  │
│                       │  - DLQ         │                                  │
│                       │  - Visibility  │                                  │
│                       └────────┬────────┘                                  │
│                                │                                            │
│                                │ Event Source Mapping                      │
│                                │                                            │
│                       ┌────────▼────────┐                                  │
│                       │  Lambda:        │                                  │
│                       │  Order          │                                  │
│                       │  Processor      │                                  │
│                       │  ┌───────────┐  │                                  │
│                       │  │1.Validate │  │                                  │
│                       │  │2.Process  │  │                                  │
│                       │  │3.Update   │  │                                  │
│                       │  │4.Notify   │  │                                  │
│                       │  └───────────┘  │                                  │
│                       └──┬────────┬──┬──┘                                  │
│                          │        │  │                                     │
│            ┌─────────────┘        │  └──────────────┐                     │
│            │                      │                 │                     │
│     ┌──────▼──────┐      ┌────────▼────────┐  ┌────▼──────────┐         │
│     │  Lambda:    │      │  DynamoDB:      │  │  SNS Topic    │         │
│     │  Inventory  │      │  Orders Table   │  │  Notifications│         │
│     │  Updater    │      │  (Update)       │  │               │         │
│     └──────┬──────┘      └─────────────────┘  └───────┬───────┘         │
│            │                                           │                  │
│     ┌──────▼──────┐                            ┌───────▼───────┐         │
│     │  DynamoDB:  │                            │  Lambda:      │         │
│     │  Inventory  │                            │  Notification │         │
│     │  Table      │                            │  Handler      │         │
│     └─────────────┘                            └───────┬───────┘         │
│                                                        │                  │
│                                                 ┌──────▼──────┐          │
│                                                 │   SNS       │          │
│                                                 │   - Email   │          │
│                                                 │   - SMS     │          │
│                                                 │   - Push    │          │
│                                                 └─────────────┘          │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Monitoring & Logging Layer                    │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │   │
│  │  │ CloudWatch   │  │  X-Ray       │  │ CloudWatch   │          │   │
│  │  │ Logs         │  │  Tracing     │  │ Alarms       │          │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 🏛️ Detailed Component Architecture

### 1. API Gateway Layer

```javascript
/**
 * API Gateway Configuration
 */
const apiGatewayConfig = {
  name: 'order-processing-api',
  description: 'RESTful API for order management',
  
  // API Definition
  resources: {
    '/orders': {
      methods: {
        POST: {
          integration: 'Lambda (api-handler)',
          authorization: 'AWS_IAM or Cognito',
          requestValidation: true,
          throttling: {
            rateLimit: 1000,    // requests per second
            burstLimit: 2000
          }
        },
        GET: {
          integration: 'Lambda (api-handler)',
          queryParameters: ['limit', 'nextToken', 'status'],
          caching: {
            enabled: true,
            ttl: 300  // 5 minutes
          }
        }
      }
    },
    '/orders/{orderId}': {
      methods: {
        GET: {
          integration: 'Lambda (api-handler)',
          pathParameters: ['orderId'],
          caching: {
            enabled: true,
            ttl: 60
          }
        },
        PUT: {
          integration: 'Lambda (api-handler)',
          authorization: 'AWS_IAM'
        },
        DELETE: {
          integration: 'Lambda (api-handler)',
          authorization: 'AWS_IAM'
        }
      }
    }
  },
  
  // Stage-specific settings
  stages: {
    dev: {
      throttling: { rateLimit: 100, burstLimit: 200 },
      caching: false,
      logging: 'INFO'
    },
    staging: {
      throttling: { rateLimit: 500, burstLimit: 1000 },
      caching: true,
      logging: 'INFO'
    },
    production: {
      throttling: { rateLimit: 1000, burstLimit: 2000 },
      caching: true,
      logging: 'ERROR',
      wafEnabled: true
    }
  }
};
```

### 2. Lambda Functions Architecture

#### 2.1 API Handler Lambda

```javascript
/**
 * API Handler Lambda - Entry point for all API requests
 */
const apiHandlerArchitecture = {
  name: 'api-handler',
  runtime: 'nodejs18.x',
  memorySize: 512,  // MB
  timeout: 30,       // seconds
  
  layers: [
    'common-libs-layer',   // AWS SDK, lodash, moment
    'validation-layer'     // joi, ajv schemas
  ],
  
  environment: {
    ORDERS_TABLE: '${STAGE}-orders',
    INVENTORY_TABLE: '${STAGE}-inventory',
    ORDER_QUEUE_URL: '${ORDER_QUEUE_URL}',
    LOG_LEVEL: '${STAGE === "prod" ? "ERROR" : "INFO"}'
  },
  
  iamRole: {
    statements: [
      {
        Effect: 'Allow',
        Action: [
          'dynamodb:GetItem',
          'dynamodb:PutItem',
          'dynamodb:Query',
          'dynamodb:Scan'
        ],
        Resource: [
          'arn:aws:dynamodb:*:*:table/${STAGE}-orders',
          'arn:aws:dynamodb:*:*:table/${STAGE}-inventory'
        ]
      },
      {
        Effect: 'Allow',
        Action: ['sqs:SendMessage'],
        Resource: '${ORDER_QUEUE_ARN}'
      }
    ]
  },
  
  // Concurrency settings
  concurrency: {
    reserved: null,        // null = unreserved
    provisioned: {
      dev: 0,
      staging: 2,
      production: 10
    }
  },
  
  // VPC Configuration (if needed)
  vpc: {
    securityGroupIds: ['${LAMBDA_SECURITY_GROUP}'],
    subnetIds: ['${PRIVATE_SUBNET_1}', '${PRIVATE_SUBNET_2}']
  }
};
```

#### 2.2 Order Processor Lambda

```javascript
/**
 * Order Processor Lambda - Processes orders from SQS queue
 */
const orderProcessorArchitecture = {
  name: 'order-processor',
  runtime: 'nodejs18.x',
  memorySize: 1024,
  timeout: 60,
  
  // SQS Event Source Mapping
  eventSourceMapping: {
    eventSourceArn: '${ORDER_QUEUE_ARN}',
    batchSize: 10,              // Process 10 messages at a time
    maxBatchingWindowSeconds: 5, // Wait up to 5 seconds to fill batch
    functionResponseTypes: ['ReportBatchItemFailures'],  // Partial batch failure
    enabled: true
  },
  
  // Business Logic Flow
  processingSteps: [
    '1. Parse SQS message',
    '2. Validate order data',
    '3. Check inventory availability',
    '4. Update order status in DynamoDB',
    '5. Trigger inventory update',
    '6. Send notification via SNS',
    '7. Handle errors and retries'
  ],
  
  // Error Handling
  errorHandling: {
    retryAttempts: 3,
    retryDelaySeconds: [60, 300, 900],  // 1min, 5min, 15min
    deadLetterQueue: '${ORDER_DLQ_URL}'
  },
  
  // Performance Optimization
  optimization: {
    connectionPooling: true,     // Reuse DB connections
    warmup: {
      enabled: true,
      schedule: 'rate(5 minutes)'  // Keep warm in production
    }
  }
};
```

#### 2.3 Inventory Updater Lambda

```javascript
/**
 * Inventory Updater Lambda - Updates inventory levels
 */
const inventoryUpdaterArchitecture = {
  name: 'inventory-updater',
  runtime: 'nodejs18.x',
  memorySize: 512,
  timeout: 30,
  
  triggers: [
    {
      type: 'SQS',
      queueArn: '${INVENTORY_QUEUE_ARN}',
      batchSize: 10
    }
  ],
  
  // Atomic operations for inventory
  operations: {
    reserveInventory: {
      dynamoOperation: 'UpdateItem',
      conditionExpression: 'available_quantity >= :quantity',
      updateExpression: 'SET reserved_quantity = reserved_quantity + :quantity'
    },
    releaseInventory: {
      dynamoOperation: 'UpdateItem',
      updateExpression: 'SET available_quantity = available_quantity + :quantity'
    }
  }
};
```

#### 2.4 Notification Handler Lambda

```javascript
/**
 * Notification Handler Lambda - Sends notifications via SNS
 */
const notificationHandlerArchitecture = {
  name: 'notification-handler',
  runtime: 'nodejs18.x',
  memorySize: 256,
  timeout: 30,
  
  triggers: [
    {
      type: 'SNS',
      topicArn: '${NOTIFICATION_TOPIC_ARN}'
    }
  ],
  
  notificationTypes: {
    orderConfirmation: {
      channels: ['email', 'sms'],
      template: 'order-confirmation-template'
    },
    orderShipped: {
      channels: ['email', 'push'],
      template: 'order-shipped-template'
    },
    orderDelivered: {
      channels: ['email'],
      template: 'order-delivered-template'
    }
  }
};
```

---

## 📊 Data Models

### DynamoDB Table Schemas

#### Orders Table

```javascript
const ordersTableSchema = {
  tableName: '${STAGE}-orders',
  
  // Primary Key
  partitionKey: {
    name: 'orderId',
    type: 'String',
    format: 'ORD-{timestamp}-{random}'
  },
  
  // Attributes
  attributes: {
    orderId: 'String',         // PK: ORD-1699804800000-ABC123
    customerId: 'String',      // GSI PK
    orderStatus: 'String',     // PENDING, PROCESSING, COMPLETED, CANCELLED
    items: 'List',             // Array of order items
    totalAmount: 'Number',
    currency: 'String',
    shippingAddress: 'Map',
    billingAddress: 'Map',
    createdAt: 'String',       // ISO 8601
    updatedAt: 'String',
    metadata: 'Map'
  },
  
  // Global Secondary Indexes
  gsi: [
    {
      indexName: 'customer-index',
      partitionKey: 'customerId',
      sortKey: 'createdAt',
      projection: 'ALL'
    },
    {
      indexName: 'status-index',
      partitionKey: 'orderStatus',
      sortKey: 'createdAt',
      projection: 'ALL'
    }
  ],
  
  // Capacity Settings
  billingMode: {
    dev: 'PAY_PER_REQUEST',
    staging: 'PROVISIONED',
    production: 'PROVISIONED'
  },
  
  provisionedThroughput: {
    staging: { readCapacity: 5, writeCapacity: 5 },
    production: { readCapacity: 100, writeCapacity: 50 }
  },
  
  // Auto Scaling
  autoScaling: {
    enabled: true,
    minCapacity: 5,
    maxCapacity: 1000,
    targetUtilization: 70
  },
  
  // Point-in-Time Recovery
  pointInTimeRecovery: {
    enabled: true
  },
  
  // TTL (Time to Live)
  ttl: {
    enabled: false  // Keep orders indefinitely
  }
};
```

#### Inventory Table

```javascript
const inventoryTableSchema = {
  tableName: '${STAGE}-inventory',
  
  // Primary Key
  partitionKey: {
    name: 'productId',
    type: 'String'
  },
  
  attributes: {
    productId: 'String',           // PK: PROD-12345
    productName: 'String',
    sku: 'String',
    availableQuantity: 'Number',   // Current available stock
    reservedQuantity: 'Number',    // Reserved but not fulfilled
    totalQuantity: 'Number',       // Total in warehouse
    reorderPoint: 'Number',        // When to reorder
    reorderQuantity: 'Number',     // How much to reorder
    lastUpdated: 'String',
    version: 'Number'              // Optimistic locking
  },
  
  // No GSI needed for this table
  gsi: [],
  
  billingMode: {
    dev: 'PAY_PER_REQUEST',
    staging: 'PROVISIONED',
    production: 'PROVISIONED'
  }
};
```

---

## 🔄 Message Queue Architecture

### SQS Queue Configuration

```javascript
const sqsQueueArchitecture = {
  // Main Order Processing Queue (FIFO)
  orderQueue: {
    name: '${STAGE}-order-processing.fifo',
    type: 'FIFO',
    
    settings: {
      messageRetentionPeriod: 1209600,     // 14 days
      visibilityTimeout: 300,               // 5 minutes
      receiveMessageWaitTime: 20,           // Long polling
      maxReceiveCount: 3,                   // After 3 retries, go to DLQ
      contentBasedDeduplication: true,      // Automatic dedup
      fifoThroughputLimit: 'perMessageGroupId'
    },
    
    // Dead Letter Queue
    deadLetterQueue: {
      name: '${STAGE}-order-processing-dlq.fifo',
      maxReceiveCount: 3
    },
    
    // Message Format
    messageStructure: {
      orderId: 'String',
      orderData: 'Object',
      timestamp: 'Number',
      retryCount: 'Number',
      messageGroupId: 'customerId'  // For FIFO ordering
    }
  },
  
  // Inventory Update Queue
  inventoryQueue: {
    name: '${STAGE}-inventory-update',
    type: 'Standard',  // Standard queue for high throughput
    
    settings: {
      visibilityTimeout: 180,
      maxReceiveCount: 3
    }
  }
};
```

---

## 📢 SNS Topic Architecture

```javascript
const snsTopicArchitecture = {
  // Main Notification Topic
  notificationTopic: {
    name: '${STAGE}-order-notifications',
    type: 'Standard',
    
    subscriptions: [
      {
        protocol: 'lambda',
        endpoint: '${NOTIFICATION_HANDLER_ARN}'
      },
      {
        protocol: 'email',
        endpoint: 'admin@company.com',
        filterPolicy: {
          orderStatus: ['COMPLETED', 'CANCELLED']
        }
      },
      {
        protocol: 'sqs',
        endpoint: '${ANALYTICS_QUEUE_ARN}',  // For analytics processing
        rawMessageDelivery: true
      }
    ],
    
    // Message Attributes
    messageAttributes: {
      orderId: 'String',
      customerId: 'String',
      orderStatus: 'String',
      notificationType: 'String',
      priority: 'String'
    }
  }
};
```

---

## 🌐 VPC & Networking Architecture

```javascript
const networkArchitecture = {
  vpc: {
    cidr: '10.0.0.0/16',
    name: '${STAGE}-order-processing-vpc',
    
    // Availability Zones
    azs: ['us-east-1a', 'us-east-1b', 'us-east-1c'],
    
    // Subnets
    subnets: {
      public: [
        { cidr: '10.0.1.0/24', az: 'us-east-1a' },  // NAT Gateway
        { cidr: '10.0.2.0/24', az: 'us-east-1b' },  // NAT Gateway
        { cidr: '10.0.3.0/24', az: 'us-east-1c' }   // NAT Gateway
      ],
      private: [
        { cidr: '10.0.11.0/24', az: 'us-east-1a' }, // Lambda functions
        { cidr: '10.0.12.0/24', az: 'us-east-1b' }, // Lambda functions
        { cidr: '10.0.13.0/24', az: 'us-east-1c' }  // Lambda functions
      ]
    },
    
    // NAT Gateways (for Lambda internet access)
    natGateways: {
      count: 3,  // One per AZ for high availability
      eipAllocation: true
    },
    
    // VPC Endpoints (for AWS services without internet)
    vpcEndpoints: [
      {
        service: 'dynamodb',
        type: 'Gateway'
      },
      {
        service: 's3',
        type: 'Gateway'
      },
      {
        service: 'sqs',
        type: 'Interface'
      },
      {
        service: 'sns',
        type: 'Interface'
      },
      {
        service: 'secretsmanager',
        type: 'Interface'
      },
      {
        service: 'kms',
        type: 'Interface'
      }
    ]
  },
  
  // Security Groups
  securityGroups: {
    lambda: {
      name: '${STAGE}-lambda-sg',
      inbound: [],  // No inbound needed
      outbound: [
        { protocol: 'tcp', port: 443, destination: '0.0.0.0/0', description: 'HTTPS to AWS services' }
      ]
    }
  }
};
```

---

## 🔐 Security Architecture

### IAM Roles & Policies

```javascript
const securityArchitecture = {
  // Lambda Execution Roles
  lambdaRoles: {
    apiHandler: {
      roleName: '${STAGE}-api-handler-role',
      policies: [
        'AWSLambdaVPCAccessExecutionRole',
        'AWSXRayDaemonWriteAccess',
        {
          custom: {
            dynamoDB: ['GetItem', 'PutItem', 'Query', 'Scan'],
            sqs: ['SendMessage'],
            cloudWatch: ['PutMetricData']
          }
        }
      ]
    },
    
    orderProcessor: {
      roleName: '${STAGE}-order-processor-role',
      policies: [
        'AWSLambdaSQSQueueExecutionRole',
        {
          custom: {
            dynamoDB: ['GetItem', 'PutItem', 'UpdateItem'],
            sns: ['Publish'],
            sqs: ['ReceiveMessage', 'DeleteMessage']
          }
        }
      ]
    }
  },
  
  // API Gateway Authorization
  apiAuthorization: {
    methods: [
      {
        type: 'AWS_IAM',
        description: 'Use IAM credentials for internal services'
      },
      {
        type: 'COGNITO_USER_POOLS',
        userPoolArn: '${COGNITO_USER_POOL_ARN}',
        description: 'Use Cognito for customer authentication'
      },
      {
        type: 'API_KEY',
        description: 'Use API keys for third-party integrations'
      }
    ]
  },
  
  // Encryption
  encryption: {
    atRest: {
      dynamoDB: 'AWS_OWNED_CMK',  // or CUSTOMER_MANAGED_CMK
      s3: 'AES256',
      sqs: 'AWS_MANAGED_CMK'
    },
    inTransit: {
      api: 'TLS 1.2+',
      internal: 'TLS 1.2+'
    }
  }
};
```

---

## 📈 Monitoring & Observability

```javascript
const monitoringArchitecture = {
  // CloudWatch Metrics
  metrics: {
    api: [
      'Count',           // Total requests
      '4XXError',        // Client errors
      '5XXError',        // Server errors
      'Latency',         // Response time
      'CacheHitCount',   // Cache effectiveness
      'CacheMissCount'
    ],
    lambda: [
      'Invocations',
      'Errors',
      'Duration',
      'Throttles',
      'ConcurrentExecutions',
      'DeadLetterErrors'
    ],
    dynamoDB: [
      'ConsumedReadCapacityUnits',
      'ConsumedWriteCapacityUnits',
      'UserErrors',
      'SystemErrors',
      'ThrottledRequests'
    ],
    sqs: [
      'ApproximateNumberOfMessagesVisible',
      'ApproximateAgeOfOldestMessage',
      'NumberOfMessagesSent',
      'NumberOfMessagesDeleted'
    ]
  },
  
  // CloudWatch Alarms
  alarms: [
    {
      name: 'API-High-Error-Rate',
      metric: 'API Gateway 5XXError',
      threshold: 10,
      period: 300,
      evaluationPeriods: 2,
      action: 'SNS notification + PagerDuty'
    },
    {
      name: 'Lambda-High-Error-Rate',
      metric: 'Lambda Errors',
      threshold: 5,
      period: 300,
      action: 'SNS notification'
    },
    {
      name: 'DLQ-Messages-Present',
      metric: 'SQS ApproximateNumberOfMessagesVisible',
      threshold: 1,
      action: 'SNS notification + Auto-remediation Lambda'
    }
  ],
  
  // X-Ray Tracing
  xray: {
    enabled: true,
    samplingRate: {
      dev: 1.0,        // 100% sampling
      staging: 0.5,    // 50% sampling
      production: 0.1  // 10% sampling
    }
  },
  
  // Structured Logging
  logging: {
    format: 'JSON',
    fields: [
      'timestamp',
      'level',
      'requestId',
      'orderId',
      'customerId',
      'message',
      'duration',
      'statusCode'
    ],
    retention: {
      dev: 7,        // days
      staging: 30,
      production: 90
    }
  }
};
```

---

## 🔄 Event Flow Sequence

### 1. Order Creation Flow

```
1. Customer → API Gateway POST /orders
2. API Gateway → Validate request schema
3. API Gateway → Lambda (api-handler)
4. Lambda → Validate business rules
5. Lambda → DynamoDB PutItem (orders table)
6. Lambda → SQS SendMessage (order queue)
7. Lambda → API Gateway → Customer (201 Created)
8. SQS → Lambda (order-processor) [async]
9. Lambda → DynamoDB GetItem (inventory table)
10. Lambda → DynamoDB UpdateItem (orders table - status)
11. Lambda → SNS Publish (notification topic)
12. SNS → Lambda (notification-handler)
13. Lambda → Send email/SMS to customer
```

### 2. Error Handling Flow

```
1. Lambda function encounters error
2. If retryable error → Return to SQS (retry after visibility timeout)
3. After maxReceiveCount (3) → Move to DLQ
4. CloudWatch Alarm triggered on DLQ messages
5. SNS notification sent to operations team
6. Auto-remediation Lambda (optional) attempts fix
7. Manual intervention if needed
```

---

## 💡 Design Decisions & Rationale

### Why FIFO Queue for Order Processing?
- **Guarantee**: Exactly-once processing per message group
- **Ordering**: Orders from same customer processed in sequence
- **Deduplication**: Automatic deduplication prevents duplicate orders

### Why DynamoDB Over RDS?
- **Scalability**: Seamless horizontal scaling
- **Performance**: Single-digit millisecond latency
- **Serverless**: No server management
- **Cost**: Pay-per-request model for variable workloads

### Why Lambda Over EC2/ECS?
- **Zero Administration**: No server management
- **Auto-scaling**: Automatic based on load
- **Cost**: Pay only for execution time
- **Integration**: Native integration with AWS services

### Why API Gateway Over ALB?
- **Features**: Built-in throttling, caching, API keys
- **Transformation**: Request/response mapping
- **Deployment**: Stage management built-in
- **WebSocket**: Support for real-time connections

---

## 🎯 Non-Functional Requirements

```javascript
const nfr = {
  performance: {
    apiLatency: '< 200ms (p95)',
    orderProcessing: '< 5 seconds',
    availability: '99.9%',
    throughput: '1000 requests/second (peak)'
  },
  
  scalability: {
    horizontal: 'Auto-scale to 10,000 concurrent requests',
    vertical: 'N/A (serverless)'
  },
  
  security: {
    authentication: 'Required for all endpoints',
    encryption: 'At rest and in transit',
    compliance: 'PCI-DSS, GDPR ready'
  },
  
  reliability: {
    dataLoss: 'Zero tolerance',
    rto: '< 1 hour',
    rpo: '< 5 minutes',
    backup: 'Daily automated backups'
  }
};
```

---

**Next Steps:**
👉 [Prerequisites Setup](./02_PREREQUISITES_SETUP.md)

---

**Last Updated**: November 12, 2025  
**Version**: 1.0.0
