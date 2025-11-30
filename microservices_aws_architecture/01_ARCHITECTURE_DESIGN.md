# Architecture Design - Enterprise Microservices on AWS

## 🎨 System Architecture Overview

### High-Level Multi-Service Architecture

```
                    ┌─────────────────────────────────────────────────────┐
                    │         External Users & Applications               │
                    │    (Web Apps, Mobile Apps, Third-party Systems)     │
                    └──────────────────────┬──────────────────────────────┘
                                           │
                                           │ HTTPS/WSS
                                           ▼
┌────────────────────────────────────────────────────────────────────────────────────┐
│                                Route 53 (DNS)                                       │
│                    Global Traffic Management & Health Checks                        │
└──────────────────────────────────┬─────────────────────────────────────────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │                             │
                    ▼                             ▼
         ┌──────────────────┐          ┌──────────────────┐
         │   CloudFront     │          │   CloudFront     │
         │   (Primary)      │          │   (Failover)     │
         │   CDN + WAF      │          │   CDN + WAF      │
         └────────┬─────────┘          └────────┬─────────┘
                  │                             │
                  └──────────────┬──────────────┘
                                 │
┌────────────────────────────────┼────────────────────────────────────────────────────┐
│                          AWS CLOUD (Multi-Region)                                   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │                    Application Load Balancer (ALB)                           │  │
│  │          - SSL Termination  - Health Checks  - Target Groups                │  │
│  └───────────────────────────────┬─────────────────────────────────────────────┘  │
│                                  │                                                 │
│  ┌───────────────────────────────┴─────────────────────────────────────────────┐  │
│  │                         API Gateway (REST + WebSocket)                       │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │  │
│  │  │  /users  │  │ /products│  │  /orders │  │ /payments│  │/analytics│    │  │
│  │  │  POST    │  │  GET     │  │  POST    │  │  POST    │  │  GET     │    │  │
│  │  │  GET     │  │  PUT     │  │  GET     │  │  GET     │  │  POST    │    │  │
│  │  │  PUT     │  │  DELETE  │  │  PUT     │  │          │  │          │    │  │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │  │
│  │                                                                             │  │
│  │  Features: Throttling, Caching, API Keys, JWT Auth, Request Validation     │  │
│  └─────┬──────────┬──────────┬──────────┬──────────┬──────────────────────────┘  │
│        │          │          │          │          │                              │
│        │          │          │          │          │                              │
│  ┌─────▼──────────▼──────────▼──────────▼──────────▼─────────────────┐          │
│  │                    Service Mesh (AWS App Mesh)                      │          │
│  │     - Service Discovery  - Traffic Routing  - Circuit Breaking     │          │
│  │     - Retry Logic  - Timeout Management  - mTLS Encryption         │          │
│  └─────┬──────────┬──────────┬──────────┬──────────┬─────────────────┘          │
│        │          │          │          │          │                              │
│        │          │          │          │          │                              │
│  ┌─────▼─────┐ ┌──▼──────┐ ┌─▼──────┐ ┌─▼──────┐ ┌─▼────────┐                  │
│  │User       │ │Product  │ │Order   │ │Payment │ │Analytics │                  │
│  │Service    │ │Service  │ │Service │ │Service │ │Service   │                  │
│  │           │ │         │ │        │ │        │ │          │                  │
│  │Features:  │ │Features:│ │Features│ │Features│ │Features: │                  │
│  │-Auth      │ │-Catalog │ │-Create │ │-Process│ │-Tracking │                  │
│  │-Profile   │ │-Search  │ │-Status │ │-Refund │ │-Reports  │                  │
│  │-Prefs     │ │-Inv Mgmt│ │-History│ │-Webhook│ │-Metrics  │                  │
│  │           │ │         │ │        │ │        │ │          │                  │
│  │Deployment:│ │Deploy:  │ │Deploy: │ │Deploy: │ │Deploy:   │                  │
│  │ECS Fargate│ │Lambda   │ │EKS     │ │Lambda  │ │Kinesis   │                  │
│  │           │ │         │ │        │ │        │ │+Lambda   │                  │
│  └─────┬─────┘ └──┬──────┘ └─┬──────┘ └─┬──────┘ └─┬────────┘                  │
│        │          │           │          │          │                            │
│        │          │           │          │          │                            │
│  ┌─────▼──────────▼───────────▼──────────▼──────────▼──────────┐                │
│  │              Database Layer (Multi-Database Pattern)          │                │
│  │                                                                │                │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │                │
│  │  │RDS       │  │DynamoDB  │  │RDS       │  │DynamoDB  │    │                │
│  │  │PostgreSQL│  │(NoSQL)   │  │MySQL     │  │(NoSQL)   │    │                │
│  │  │          │  │          │  │          │  │          │    │                │
│  │  │Users DB  │  │Products  │  │Orders DB │  │Payments  │    │                │
│  │  │          │  │Catalog   │  │          │  │Txns      │    │                │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │                │
│  │                                                                │                │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │                │
│  │  │ElastiCache│ │ElastiCache│ │S3 Data   │                   │                │
│  │  │Redis     │  │Memcached │  │Lake      │                   │                │
│  │  │Sessions  │  │Query     │  │Analytics │                   │                │
│  │  └──────────┘  └──────────┘  └──────────┘                   │                │
│  └────────────────────────────────────────────────────────────┘                │
│                                                                                   │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │              Event-Driven Communication Layer                              │  │
│  │                                                                             │  │
│  │  ┌────────────────┐    ┌────────────────┐    ┌────────────────┐          │  │
│  │  │  EventBridge   │    │  SQS Queues    │    │  SNS Topics    │          │  │
│  │  │  (Event Bus)   │    │  (Async Jobs)  │    │  (Pub/Sub)     │          │  │
│  │  │                │    │                │    │                │          │  │
│  │  │ - Order Events │    │ - Email Queue  │    │ - Notif Topic  │          │  │
│  │  │ - User Events  │    │ - Report Queue │    │ - Alert Topic  │          │  │
│  │  │ - Payment Evt  │    │ - Export Queue │    │ - Webhook Topic│          │  │
│  │  └────────────────┘    └────────────────┘    └────────────────┘          │  │
│  │                                                                             │  │
│  │  ┌────────────────┐    ┌────────────────┐    ┌────────────────┐          │  │
│  │  │ Step Functions │    │ Kinesis Data   │    │ API Gateway    │          │  │
│  │  │ (Workflows)    │    │ Streams        │    │ WebSocket      │          │  │
│  │  │                │    │ (Real-time)    │    │ (Real-time)    │          │  │
│  │  │ - Order Flow   │    │ - Click Stream │    │ - Live Updates │          │  │
│  │  │ - Approval Flow│    │ - Log Stream   │    │ - Chat         │          │  │
│  │  └────────────────┘    └────────────────┘    └────────────────┘          │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                   │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │              Observability & Monitoring Layer                              │  │
│  │                                                                             │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐         │  │
│  │  │CloudWatch  │  │  X-Ray     │  │CloudWatch  │  │Elasticsearch│         │  │
│  │  │Logs        │  │ Tracing    │  │ Alarms     │  │(ELK Stack) │         │  │
│  │  └────────────┘  └────────────┘  └────────────┘  └────────────┘         │  │
│  │                                                                             │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐                          │  │
│  │  │Prometheus  │  │  Grafana   │  │OpenTelemetry│                         │  │
│  │  │Metrics     │  │ Dashboards │  │(OTEL)      │                          │  │
│  │  └────────────┘  └────────────┘  └────────────┘                          │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                   │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                    CI/CD & DevOps Pipeline                                 │  │
│  │                                                                             │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐         │  │
│  │  │CodeCommit/ │→ │CodeBuild   │→ │CodeDeploy  │→ │CloudFormation│        │  │
│  │  │GitHub      │  │(Build)     │  │(Deploy)    │  │/Terraform  │         │  │
│  │  └────────────┘  └────────────┘  └────────────┘  └────────────┘         │  │
│  │                                                                             │  │
│  │  Deployment Strategies: Blue-Green, Canary, Rolling Updates               │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                   │
└───────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🏛️ Microservices Architecture - Detailed View

### Service Breakdown

```
┌────────────────────────────────────────────────────────────────────────┐
│                         Microservices Ecosystem                         │
└────────────────────────────────────────────────────────────────────────┘

┌─────────────────────┐
│  1. User Service    │
├─────────────────────┤
│ Responsibilities:   │
│ - User registration │
│ - Authentication    │
│ - Profile management│
│ - Authorization     │
│ - Session mgmt      │
│                     │
│ Tech Stack:         │
│ - ECS Fargate       │
│ - Node.js           │
│ - RDS PostgreSQL    │
│ - Redis (sessions)  │
│ - Cognito (auth)    │
│                     │
│ APIs:               │
│ POST /users/signup  │
│ POST /users/login   │
│ GET  /users/{id}    │
│ PUT  /users/{id}    │
│ DELETE /users/{id}  │
└─────────────────────┘

┌─────────────────────┐
│  2. Product Service │
├─────────────────────┤
│ Responsibilities:   │
│ - Product catalog   │
│ - Inventory mgmt    │
│ - Search & filter   │
│ - Price management  │
│ - Categories        │
│                     │
│ Tech Stack:         │
│ - AWS Lambda        │
│ - Python            │
│ - DynamoDB          │
│ - ElastiCache       │
│ - OpenSearch        │
│                     │
│ APIs:               │
│ GET  /products      │
│ GET  /products/{id} │
│ POST /products      │
│ PUT  /products/{id} │
│ DELETE /products/{id}│
└─────────────────────┘

┌─────────────────────┐
│  3. Order Service   │
├─────────────────────┤
│ Responsibilities:   │
│ - Order creation    │
│ - Order tracking    │
│ - Order history     │
│ - Order cancellation│
│ - Status updates    │
│                     │
│ Tech Stack:         │
│ - EKS (Kubernetes)  │
│ - Java Spring Boot  │
│ - RDS MySQL         │
│ - SQS queues        │
│ - Step Functions    │
│                     │
│ APIs:               │
│ POST /orders        │
│ GET  /orders/{id}   │
│ PUT  /orders/{id}   │
│ GET  /orders/user/{id}│
│ DELETE /orders/{id} │
└─────────────────────┘

┌─────────────────────┐
│  4. Payment Service │
├─────────────────────┤
│ Responsibilities:   │
│ - Payment processing│
│ - Transaction mgmt  │
│ - Refunds           │
│ - Payment methods   │
│ - Fraud detection   │
│                     │
│ Tech Stack:         │
│ - AWS Lambda        │
│ - Node.js           │
│ - DynamoDB          │
│ - Stripe/PayPal API │
│ - SQS (async)       │
│                     │
│ APIs:               │
│ POST /payments      │
│ GET  /payments/{id} │
│ POST /payments/refund│
│ GET  /payments/status│
└─────────────────────┘

┌─────────────────────┐
│  5. Notification    │
│     Service         │
├─────────────────────┤
│ Responsibilities:   │
│ - Email notifications│
│ - SMS notifications │
│ - Push notifications│
│ - Notification prefs│
│ - Templates         │
│                     │
│ Tech Stack:         │
│ - AWS Lambda        │
│ - Python            │
│ - SNS               │
│ - SES (email)       │
│ - DynamoDB          │
│                     │
│ Triggers:           │
│ - SNS subscriptions │
│ - EventBridge rules │
│ - SQS queues        │
└─────────────────────┘

┌─────────────────────┐
│  6. Analytics       │
│     Service         │
├─────────────────────┤
│ Responsibilities:   │
│ - Event tracking    │
│ - User behavior     │
│ - Sales analytics   │
│ - Real-time metrics │
│ - Reporting         │
│                     │
│ Tech Stack:         │
│ - Kinesis Streams   │
│ - Lambda (processing)│
│ - S3 Data Lake      │
│ - Athena (queries)  │
│ - QuickSight        │
│                     │
│ APIs:               │
│ POST /events        │
│ GET  /analytics/users│
│ GET  /analytics/sales│
│ GET  /analytics/reports│
└─────────────────────┘

┌─────────────────────┐
│  7. Inventory       │
│     Service         │
├─────────────────────┤
│ Responsibilities:   │
│ - Stock management  │
│ - Reservations      │
│ - Stock updates     │
│ - Low stock alerts  │
│ - Warehouse mgmt    │
│                     │
│ Tech Stack:         │
│ - AWS Lambda        │
│ - Python            │
│ - DynamoDB          │
│ - SQS               │
│ - EventBridge       │
│                     │
│ Events:             │
│ - InventoryReserved │
│ - InventoryReleased │
│ - LowStockAlert     │
│ - StockUpdated      │
└─────────────────────┘

┌─────────────────────┐
│  8. Shipping        │
│     Service         │
├─────────────────────┤
│ Responsibilities:   │
│ - Shipping rates    │
│ - Label generation  │
│ - Tracking          │
│ - Carrier integration│
│ - Delivery estimates│
│                     │
│ Tech Stack:         │
│ - ECS Fargate       │
│ - Node.js           │
│ - RDS PostgreSQL    │
│ - FedEx/UPS APIs    │
│ - SQS               │
│                     │
│ APIs:               │
│ GET  /shipping/rates│
│ POST /shipping/labels│
│ GET  /shipping/track│
└─────────────────────┘
```

---

## 🔄 Communication Patterns

### 1. Synchronous Communication (REST/gRPC)

```javascript
/**
 * REST API Communication Pattern
 * Used for: Request-response scenarios requiring immediate results
 */

// Example: Order Service calling Payment Service
const orderToPaymentFlow = {
  scenario: 'Customer places order and pays',
  
  // Step 1: Order Service receives request
  orderService: {
    endpoint: 'POST /orders',
    handler: async (request) => {
      // Validate order
      const order = await validateOrder(request.body);
      
      // Step 2: Synchronously call Payment Service
      const paymentResponse = await fetch('https://payment-service/api/payments', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${serviceToken}`,
          'X-Request-ID': request.requestId
        },
        body: JSON.stringify({
          orderId: order.id,
          amount: order.totalAmount,
          currency: 'USD',
          paymentMethod: request.body.paymentMethod
        })
      });
      
      // Step 3: Handle payment response
      if (paymentResponse.ok) {
        const payment = await paymentResponse.json();
        order.paymentId = payment.id;
        order.status = 'PAID';
        await saveOrder(order);
        return { success: true, order };
      } else {
        order.status = 'PAYMENT_FAILED';
        await saveOrder(order);
        throw new Error('Payment processing failed');
      }
    }
  },
  
  // Timeout and retry configuration
  resilience: {
    timeout: 5000,        // 5 seconds
    retries: 3,
    backoff: 'exponential',
    circuitBreaker: {
      enabled: true,
      threshold: 5,       // Open circuit after 5 failures
      timeout: 30000      // Try again after 30 seconds
    }
  }
};
```

### 2. Asynchronous Communication (Event-Driven)

```javascript
/**
 * Event-Driven Communication Pattern
 * Used for: Loose coupling, eventual consistency, fire-and-forget
 */

// Example: Order Service emitting events
const eventDrivenFlow = {
  scenario: 'Order created, multiple services react',
  
  // Publisher: Order Service
  orderService: {
    handler: async (order) => {
      // Save order to database
      await dynamoDB.putItem({
        TableName: 'orders',
        Item: order
      });
      
      // Publish event to EventBridge
      await eventBridge.putEvents({
        Entries: [{
          Source: 'order.service',
          DetailType: 'OrderCreated',
          Detail: JSON.stringify({
            orderId: order.id,
            customerId: order.customerId,
            items: order.items,
            totalAmount: order.totalAmount,
            timestamp: new Date().toISOString()
          })
        }]
      });
      
      return { success: true, orderId: order.id };
    }
  },
  
  // Subscriber 1: Inventory Service
  inventoryService: {
    eventRule: {
      source: 'order.service',
      detailType: 'OrderCreated'
    },
    handler: async (event) => {
      // Reserve inventory for order items
      for (const item of event.detail.items) {
        await reserveInventory(item.productId, item.quantity);
      }
      
      // Emit inventory reserved event
      await eventBridge.putEvents({
        Entries: [{
          Source: 'inventory.service',
          DetailType: 'InventoryReserved',
          Detail: JSON.stringify({
            orderId: event.detail.orderId,
            items: event.detail.items
          })
        }]
      });
    }
  },
  
  // Subscriber 2: Notification Service
  notificationService: {
    eventRule: {
      source: 'order.service',
      detailType: 'OrderCreated'
    },
    handler: async (event) => {
      // Send order confirmation email
      await ses.sendEmail({
        to: event.detail.customerEmail,
        subject: 'Order Confirmation',
        template: 'order-confirmation',
        data: event.detail
      });
    }
  },
  
  // Subscriber 3: Analytics Service
  analyticsService: {
    eventRule: {
      source: 'order.service',
      detailType: 'OrderCreated'
    },
    handler: async (event) => {
      // Stream to Kinesis for real-time analytics
      await kinesis.putRecord({
        StreamName: 'order-events',
        Data: JSON.stringify(event.detail),
        PartitionKey: event.detail.customerId
      });
    }
  }
};
```

### 3. Message Queue Pattern (SQS)

```javascript
/**
 * Message Queue Pattern
 * Used for: Background jobs, async processing, load leveling
 */

const messageQueuePattern = {
  // Producer: Order Service
  producer: {
    handler: async (order) => {
      // Send message to SQS queue
      await sqs.sendMessage({
        QueueUrl: process.env.ORDER_PROCESSING_QUEUE_URL,
        MessageBody: JSON.stringify({
          orderId: order.id,
          action: 'PROCESS_ORDER',
          data: order
        }),
        MessageGroupId: order.customerId,  // FIFO ordering per customer
        MessageDeduplicationId: order.id
      });
    }
  },
  
  // Consumer: Order Processor Lambda
  consumer: {
    batchSize: 10,
    handler: async (event) => {
      const records = event.Records;
      const results = [];
      
      for (const record of records) {
        try {
          const message = JSON.parse(record.body);
          
          // Process order
          await processOrder(message.data);
          
          // Successfully processed
          results.push({
            itemIdentifier: record.messageId,
            status: 'success'
          });
        } catch (error) {
          console.error('Processing failed:', error);
          
          // This message will be retried or sent to DLQ
          results.push({
            itemIdentifier: record.messageId,
            status: 'failure'
          });
        }
      }
      
      // Return batch item failures for partial batch success
      return {
        batchItemFailures: results
          .filter(r => r.status === 'failure')
          .map(r => ({ itemIdentifier: r.itemIdentifier }))
      };
    }
  },
  
  // Queue Configuration
  queueConfig: {
    type: 'FIFO',
    visibilityTimeout: 300,
    messageRetentionPeriod: 1209600,  // 14 days
    maxReceiveCount: 3,
    deadLetterQueue: 'order-processing-dlq.fifo'
  }
};
```

### 4. Saga Pattern (Distributed Transactions)

```javascript
/**
 * Saga Pattern using Step Functions
 * Used for: Distributed transactions, compensating transactions
 */

const sagaPattern = {
  // Step Functions State Machine
  stateMachine: {
    Comment: 'Order Processing Saga',
    StartAt: 'ReserveInventory',
    States: {
      // Step 1: Reserve Inventory
      ReserveInventory: {
        Type: 'Task',
        Resource: 'arn:aws:lambda:region:account:function:reserve-inventory',
        Next: 'ProcessPayment',
        Catch: [{
          ErrorEquals: ['States.ALL'],
          Next: 'ReleaseInventory'
        }]
      },
      
      // Step 2: Process Payment
      ProcessPayment: {
        Type: 'Task',
        Resource: 'arn:aws:lambda:region:account:function:process-payment',
        Next: 'CreateShipment',
        Catch: [{
          ErrorEquals: ['States.ALL'],
          Next: 'RefundPayment'
        }]
      },
      
      // Step 3: Create Shipment
      CreateShipment: {
        Type: 'Task',
        Resource: 'arn:aws:lambda:region:account:function:create-shipment',
        Next: 'SendNotification',
        Catch: [{
          ErrorEquals: ['States.ALL'],
          Next: 'CancelShipment'
        }]
      },
      
      // Step 4: Send Notification
      SendNotification: {
        Type: 'Task',
        Resource: 'arn:aws:lambda:region:account:function:send-notification',
        Next: 'Success'
      },
      
      // Success State
      Success: {
        Type: 'Succeed'
      },
      
      // Compensating Transaction: Release Inventory
      ReleaseInventory: {
        Type: 'Task',
        Resource: 'arn:aws:lambda:region:account:function:release-inventory',
        Next: 'Failed'
      },
      
      // Compensating Transaction: Refund Payment
      RefundPayment: {
        Type: 'Task',
        Resource: 'arn:aws:lambda:region:account:function:refund-payment',
        Next: 'ReleaseInventory'
      },
      
      // Compensating Transaction: Cancel Shipment
      CancelShipment: {
        Type: 'Task',
        Resource: 'arn:aws:lambda:region:account:function:cancel-shipment',
        Next: 'RefundPayment'
      },
      
      // Failed State
      Failed: {
        Type: 'Fail',
        Cause: 'Order processing failed',
        Error: 'OrderProcessingError'
      }
    }
  }
};
```

---

## 🗄️ Database Architecture

### Database Per Service Pattern

```javascript
/**
 * Each microservice has its own database
 * Ensures loose coupling and independent scaling
 */

const databaseArchitecture = {
  // 1. User Service Database
  userService: {
    type: 'RDS PostgreSQL',
    engine: 'postgres',
    version: '15.3',
    instanceClass: 'db.t3.medium',
    multiAZ: true,
    
    schema: {
      users: {
        columns: {
          user_id: 'UUID PRIMARY KEY',
          email: 'VARCHAR(255) UNIQUE NOT NULL',
          password_hash: 'VARCHAR(255)',
          first_name: 'VARCHAR(100)',
          last_name: 'VARCHAR(100)',
          phone: 'VARCHAR(20)',
          created_at: 'TIMESTAMP DEFAULT NOW()',
          updated_at: 'TIMESTAMP DEFAULT NOW()',
          is_active: 'BOOLEAN DEFAULT TRUE'
        },
        indexes: [
          'CREATE INDEX idx_users_email ON users(email)',
          'CREATE INDEX idx_users_created_at ON users(created_at)'
        ]
      },
      user_profiles: {
        columns: {
          profile_id: 'UUID PRIMARY KEY',
          user_id: 'UUID REFERENCES users(user_id)',
          avatar_url: 'VARCHAR(500)',
          bio: 'TEXT',
          preferences: 'JSONB'
        }
      }
    },
    
    backup: {
      retentionPeriod: 30,  // days
      preferredBackupWindow: '03:00-04:00'
    }
  },
  
  // 2. Product Service Database
  productService: {
    type: 'DynamoDB',
    billingMode: 'PAY_PER_REQUEST',
    
    tables: {
      products: {
        partitionKey: 'product_id',
        attributes: {
          product_id: 'String',
          name: 'String',
          description: 'String',
          price: 'Number',
          category: 'String',
          images: 'List',
          specifications: 'Map',
          created_at: 'String',
          updated_at: 'String'
        },
        gsi: [
          {
            indexName: 'category-index',
            partitionKey: 'category',
            sortKey: 'created_at',
            projection: 'ALL'
          }
        ]
      }
    },
    
    // Caching Layer
    cache: {
      engine: 'ElastiCache Redis',
      nodeType: 'cache.t3.medium',
      numCacheNodes: 2,
      ttl: 3600  // 1 hour
    }
  },
  
  // 3. Order Service Database
  orderService: {
    type: 'RDS MySQL',
    engine: 'mysql',
    version: '8.0',
    instanceClass: 'db.r5.large',
    multiAZ: true,
    
    schema: {
      orders: {
        columns: {
          order_id: 'VARCHAR(50) PRIMARY KEY',
          customer_id: 'VARCHAR(50) NOT NULL',
          order_status: 'ENUM("PENDING","PROCESSING","SHIPPED","DELIVERED","CANCELLED")',
          total_amount: 'DECIMAL(10,2)',
          currency: 'VARCHAR(3)',
          created_at: 'TIMESTAMP DEFAULT CURRENT_TIMESTAMP',
          updated_at: 'TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP'
        },
        indexes: [
          'CREATE INDEX idx_orders_customer ON orders(customer_id)',
          'CREATE INDEX idx_orders_status ON orders(order_status)',
          'CREATE INDEX idx_orders_created ON orders(created_at)'
        ]
      },
      order_items: {
        columns: {
          item_id: 'INT AUTO_INCREMENT PRIMARY KEY',
          order_id: 'VARCHAR(50) NOT NULL',
          product_id: 'VARCHAR(50) NOT NULL',
          quantity: 'INT NOT NULL',
          unit_price: 'DECIMAL(10,2)',
          subtotal: 'DECIMAL(10,2)',
          'FOREIGN KEY (order_id) REFERENCES orders(order_id)': ''
        }
      }
    },
    
    readReplicas: 2
  },
  
  // 4. Payment Service Database
  paymentService: {
    type: 'DynamoDB',
    billingMode: 'PAY_PER_REQUEST',
    
    tables: {
      payments: {
        partitionKey: 'payment_id',
        sortKey: 'timestamp',
        attributes: {
          payment_id: 'String',
          order_id: 'String',
          customer_id: 'String',
          amount: 'Number',
          currency: 'String',
          payment_method: 'String',
          status: 'String',
          gateway_transaction_id: 'String',
          timestamp: 'Number'
        },
        gsi: [
          {
            indexName: 'order-index',
            partitionKey: 'order_id',
            sortKey: 'timestamp',
            projection: 'ALL'
          }
        ]
      }
    },
    
    // Point-in-time recovery
    pointInTimeRecovery: true
  }
};
```

---

## 🔐 Security Architecture

### Multi-Layer Security

```javascript
const securityArchitecture = {
  // Layer 1: Network Security
  networkSecurity: {
    vpc: {
      cidr: '10.0.0.0/16',
      subnets: {
        public: ['10.0.1.0/24', '10.0.2.0/24', '10.0.3.0/24'],
        private: ['10.0.11.0/24', '10.0.12.0/24', '10.0.13.0/24'],
        database: ['10.0.21.0/24', '10.0.22.0/24', '10.0.23.0/24']
      }
    },
    
    securityGroups: {
      alb: {
        inbound: [
          { port: 443, source: '0.0.0.0/0', description: 'HTTPS from internet' },
          { port: 80, source: '0.0.0.0/0', description: 'HTTP from internet' }
        ],
        outbound: [
          { port: 'all', destination: 'vpc', description: 'To VPC services' }
        ]
      },
      ecsServices: {
        inbound: [
          { port: 8080, source: 'alb-sg', description: 'From ALB' }
        ],
        outbound: [
          { port: 443, destination: '0.0.0.0/0', description: 'HTTPS to internet' },
          { port: 3306, destination: 'db-sg', description: 'To RDS' }
        ]
      },
      database: {
        inbound: [
          { port: 3306, source: 'ecs-sg', description: 'MySQL from ECS' },
          { port: 5432, source: 'ecs-sg', description: 'PostgreSQL from ECS' }
        ],
        outbound: []
      }
    },
    
    waf: {
      rules: [
        'AWS-AWSManagedRulesCommonRuleSet',
        'AWS-AWSManagedRulesKnownBadInputsRuleSet',
        'AWS-AWSManagedRulesSQLiRuleSet',
        'Custom-RateLimitRule'
      ]
    }
  },
  
  // Layer 2: Authentication & Authorization
  authentication: {
    cognito: {
      userPool: {
        name: 'microservices-user-pool',
        mfaConfiguration: 'OPTIONAL',
        passwordPolicy: {
          minimumLength: 12,
          requireUppercase: true,
          requireLowercase: true,
          requireNumbers: true,
          requireSymbols: true
        }
      },
      
      appClients: [
        {
          name: 'web-app',
          authFlows: ['USER_PASSWORD_AUTH', 'REFRESH_TOKEN_AUTH'],
          tokenValidity: {
            accessToken: 1,   // hour
            idToken: 1,       // hour
            refreshToken: 30  // days
          }
        },
        {
          name: 'mobile-app',
          authFlows: ['USER_PASSWORD_AUTH', 'REFRESH_TOKEN_AUTH']
        }
      ]
    },
    
    apiGateway: {
      authorizers: [
        {
          type: 'COGNITO_USER_POOLS',
          name: 'cognito-authorizer',
          userPoolArn: '${USER_POOL_ARN}'
        },
        {
          type: 'JWT',
          name: 'jwt-authorizer',
          jwtConfiguration: {
            issuer: 'https://cognito-idp.region.amazonaws.com/pool-id',
            audience: ['app-client-id']
          }
        }
      ]
    }
  },
  
  // Layer 3: Service-to-Service Security
  serviceSecurity: {
    iamRoles: {
      userService: {
        policies: [
          {
            Effect: 'Allow',
            Action: ['dynamodb:GetItem', 'dynamodb:PutItem', 'dynamodb:Query'],
            Resource: 'arn:aws:dynamodb:*:*:table/users'
          },
          {
            Effect: 'Allow',
            Action: ['secretsmanager:GetSecretValue'],
            Resource: 'arn:aws:secretsmanager:*:*:secret:db-credentials-*'
          }
        ]
      },
      orderService: {
        policies: [
          {
            Effect: 'Allow',
            Action: ['sqs:SendMessage', 'sqs:ReceiveMessage'],
            Resource: 'arn:aws:sqs:*:*:order-*'
          },
          {
            Effect: 'Allow',
            Action: ['events:PutEvents'],
            Resource: 'arn:aws:events:*:*:event-bus/default'
          }
        ]
      }
    },
    
    // mTLS for service mesh
    appMesh: {
      tls: {
        mode: 'STRICT',
        certificate: {
          acm: {
            certificateArn: '${ACM_CERTIFICATE_ARN}'
          }
        }
      }
    }
  },
  
  // Layer 4: Data Security
  dataSecurity: {
    encryption: {
      atRest: {
        rds: 'AWS_OWNED_KMS_KEY',
        dynamodb: 'AWS_OWNED_CMK',
        s3: 'AES256',
        ebs: 'aws/ebs'
      },
      inTransit: {
        minimum: 'TLS 1.2',
        preferred: 'TLS 1.3'
      }
    },
    
    secretsManagement: {
      secretsManager: {
        secrets: [
          'database-credentials',
          'api-keys',
          'oauth-secrets',
          'encryption-keys'
        ],
        rotation: {
          enabled: true,
          automaticallyAfter: 90  // days
        }
      }
    }
  },
  
  // Layer 5: Compliance & Auditing
  compliance: {
    cloudTrail: {
      enabled: true,
      multiRegion: true,
      includeGlobalEvents: true,
      s3Bucket: 'audit-logs-bucket',
      logFileValidation: true
    },
    
    config: {
      rules: [
        'encrypted-volumes',
        'rds-encryption-enabled',
        'iam-password-policy',
        's3-bucket-public-read-prohibited',
        'vpc-flow-logs-enabled'
      ]
    },
    
    guardDuty: {
      enabled: true,
      findingPublishing: 'SNS'
    }
  }
};
```

---

## 📊 Deployment Architecture

### Multi-Environment Strategy

```javascript
const deploymentArchitecture = {
  environments: {
    // Development Environment
    development: {
      region: 'us-east-1',
      accountId: '111111111111',
      
      compute: {
        ecs: {
          taskCount: 1,
          cpu: '256',
          memory: '512'
        },
        lambda: {
          memorySize: 512,
          timeout: 30,
          provisionedConcurrency: 0
        }
      },
      
      database: {
        rds: {
          instanceClass: 'db.t3.micro',
          multiAZ: false,
          backupRetention: 1
        },
        dynamodb: {
          billingMode: 'PAY_PER_REQUEST'
        }
      },
      
      monitoring: {
        logRetention: 7,
        detailedMonitoring: false
      }
    },
    
    // Staging Environment
    staging: {
      region: 'us-east-1',
      accountId: '222222222222',
      
      compute: {
        ecs: {
          taskCount: 2,
          cpu: '512',
          memory: '1024'
        },
        lambda: {
          memorySize: 1024,
          timeout: 60,
          provisionedConcurrency: 2
        }
      },
      
      database: {
        rds: {
          instanceClass: 'db.t3.small',
          multiAZ: true,
          backupRetention: 7,
          readReplicas: 1
        },
        dynamodb: {
          billingMode: 'PROVISIONED',
          readCapacity: 5,
          writeCapacity: 5,
          autoScaling: true
        }
      },
      
      monitoring: {
        logRetention: 30,
        detailedMonitoring: true
      }
    },
    
    // Production Environment
    production: {
      region: 'us-east-1',
      accountId: '333333333333',
      multiRegion: ['us-east-1', 'us-west-2', 'eu-west-1'],
      
      compute: {
        ecs: {
          taskCount: 10,
          cpu: '1024',
          memory: '2048',
          autoScaling: {
            minTasks: 5,
            maxTasks: 50,
            targetCPU: 70,
            targetMemory: 80
          }
        },
        lambda: {
          memorySize: 3008,
          timeout: 900,
          provisionedConcurrency: 10,
          reservedConcurrency: 100
        }
      },
      
      database: {
        rds: {
          instanceClass: 'db.r5.xlarge',
          multiAZ: true,
          backupRetention: 30,
          readReplicas: 3,
          performanceInsights: true
        },
        dynamodb: {
          billingMode: 'PROVISIONED',
          readCapacity: 100,
          writeCapacity: 50,
          autoScaling: {
            enabled: true,
            minCapacity: 50,
            maxCapacity: 10000,
            targetUtilization: 70
          },
          globalTables: true
        }
      },
      
      monitoring: {
        logRetention: 90,
        detailedMonitoring: true,
        xrayTracing: true
      },
      
      disaster Recovery: {
        rto: 3600,      // 1 hour
        rpo: 300,       // 5 minutes
        backupRegion: 'us-west-2'
      }
    }
  },
  
  // Deployment Strategies
  deploymentStrategies: {
    blueGreen: {
      services: ['user-service', 'order-service'],
      steps: [
        '1. Deploy green environment',
        '2. Run smoke tests',
        '3. Switch traffic to green',
        '4. Monitor metrics for 1 hour',
        '5. Terminate blue environment'
      ]
    },
    
    canary: {
      services: ['payment-service'],
      stages: [
        { percentage: 10, duration: 600 },   // 10% for 10 minutes
        { percentage: 25, duration: 600 },   // 25% for 10 minutes
        { percentage: 50, duration: 1200 },  // 50% for 20 minutes
        { percentage: 100, duration: 0 }     // 100%
      ],
      rollbackOnAlarm: true
    },
    
    rolling: {
      services: ['product-service', 'notification-service'],
      batchSize: 2,
      healthCheckGracePeriod: 300
    }
  }
};
```

---

## 📈 Monitoring & Observability

```javascript
const observabilityArchitecture = {
  // Three Pillars of Observability
  pillars: {
    // 1. Metrics
    metrics: {
      cloudWatch: {
        namespaces: [
          'AWS/ECS',
          'AWS/Lambda',
          'AWS/RDS',
          'AWS/DynamoDB',
          'Custom/Application'
        ],
        
        customMetrics: [
          {
            namespace: 'Custom/Application',
            metricName: 'OrdersProcessed',
            dimensions: [
              { Name: 'Service', Value: 'order-service' },
              { Name: 'Environment', Value: 'production' }
            ]
          },
          {
            namespace: 'Custom/Application',
            metricName: 'APILatency',
            dimensions: [
              { Name: 'Endpoint', Value: '/orders' },
              { Name: 'Method', Value: 'POST' }
            ]
          }
        ],
        
        dashboards: [
          {
            name: 'Service-Health-Dashboard',
            widgets: [
              'Service CPU/Memory',
              'Request Rate',
              'Error Rate',
              'Latency (p50, p95, p99)'
            ]
          },
          {
            name: 'Business-Metrics-Dashboard',
            widgets: [
              'Orders Per Minute',
              'Revenue Per Hour',
              'Active Users',
              'Conversion Rate'
            ]
          }
        ]
      },
      
      prometheus: {
        enabled: true,
        scrapeInterval: '30s',
        retention: '15d',
        exporters: [
          'node-exporter',
          'container-exporter'
        ]
      }
    },
    
    // 2. Logs
    logs: {
      cloudWatchLogs: {
        logGroups: [
          '/aws/lambda/user-service',
          '/aws/lambda/order-service',
          '/aws/ecs/user-service',
          '/aws/ecs/order-service',
          '/aws/apigateway/access-logs',
          '/aws/rds/audit-logs'
        ],
        
        retention: {
          dev: 7,
          staging: 30,
          production: 90
        },
        
        metricFilters: [
          {
            name: 'ErrorCount',
            pattern: '[level=ERROR]',
            metricNamespace: 'LogMetrics',
            metricName: 'Errors'
          }
        ]
      },
      
      structuredLogging: {
        format: 'JSON',
        fields: {
          timestamp: 'ISO8601',
          level: 'string',
          service: 'string',
          requestId: 'string',
          userId: 'string',
          orderId: 'string',
          message: 'string',
          duration: 'number',
          statusCode: 'number',
          error: 'object'
        }
      }
    },
    
    // 3. Traces
    traces: {
      xray: {
        enabled: true,
        samplingRate: {
          dev: 1.0,
          staging: 0.5,
          production: 0.1
        },
        
        segments: [
          'API Gateway',
          'Lambda',
          'DynamoDB',
          'RDS',
          'SQS',
          'SNS',
          'External HTTP'
        ],
        
        annotations: [
          'service',
          'environment',
          'userId',
          'orderId'
        ]
      },
      
      openTelemetry: {
        enabled: true,
        exporters: ['xray', 'jaeger'],
        propagators: ['tracecontext', 'baggage']
      }
    }
  },
  
  // Alerting Strategy
  alerting: {
    criticalAlerts: [
      {
        name: 'ServiceDown',
        metric: 'HealthCheckStatus',
        threshold: 0,
        evaluation: 2,
        action: 'PagerDuty + SNS'
      },
      {
        name: 'HighErrorRate',
        metric: 'ErrorRate',
        threshold: 5,  // percent
        evaluation: 2,
        action: 'PagerDuty + SNS'
      },
      {
        name: 'DatabaseConnectionPoolExhausted',
        metric: 'DatabaseConnectionsUsed',
        threshold: 95,  // percent
        action: 'PagerDuty + SNS'
      }
    ],
    
    warningAlerts: [
      {
        name: 'HighLatency',
        metric: 'ResponseTime',
        threshold: 1000,  // ms
        evaluation: 5,
        action: 'SNS'
      },
      {
        name: 'HighCPU',
        metric: 'CPUUtilization',
        threshold: 80,
        evaluation: 10,
        action: 'SNS'
      }
    ]
  }
};
```

---

## 🔄 CI/CD Pipeline Architecture

```javascript
const cicdArchitecture = {
  // Pipeline Stages
  pipeline: {
    source: {
      provider: 'GitHub',
      repository: 'org/microservices',
      branch: 'main',
      trigger: 'push'
    },
    
    stages: [
      // Stage 1: Build
      {
        name: 'Build',
        actions: [
          {
            name: 'CodeBuild',
            steps: [
              'Install dependencies',
              'Run linters (ESLint, Prettier)',
              'Run unit tests',
              'Generate code coverage report',
              'Build Docker images',
              'Scan for vulnerabilities (Snyk, Trivy)',
              'Push images to ECR'
            ],
            environment: {
              computeType: 'BUILD_GENERAL1_MEDIUM',
              image: 'aws/codebuild/standard:7.0',
              privilegedMode: true
            }
          }
        ]
      },
      
      // Stage 2: Test
      {
        name: 'Test',
        actions: [
          {
            name: 'IntegrationTests',
            steps: [
              'Deploy to test environment',
              'Run integration tests',
              'Run contract tests',
              'Run API tests (Postman/Newman)'
            ]
          },
          {
            name: 'SecurityTests',
            steps: [
              'SAST (Static Analysis)',
              'DAST (Dynamic Analysis)',
              'Dependency scanning',
              'Secrets scanning'
            ]
          }
        ]
      },
      
      // Stage 3: Deploy to Staging
      {
        name: 'DeployStaging',
        actions: [
          {
            name: 'DeployInfrastructure',
            tool: 'CloudFormation/Terraform',
            steps: [
              'Validate templates',
              'Create change sets',
              'Execute change sets'
            ]
          },
          {
            name: 'DeployApplications',
            steps: [
              'Update ECS task definitions',
              'Update Lambda functions',
              'Run database migrations',
              'Deploy with blue-green strategy'
            ]
          },
          {
            name: 'SmokeTests',
            steps: [
              'Health check endpoints',
              'Critical path tests',
              'Performance baseline'
            ]
          }
        ]
      },
      
      // Stage 4: Manual Approval
      {
        name: 'ManualApproval',
        approvers: ['team-lead@company.com', 'devops@company.com'],
        timeout: 3600  // 1 hour
      },
      
      // Stage 5: Deploy to Production
      {
        name: 'DeployProduction',
        actions: [
          {
            name: 'CanaryDeployment',
            strategy: 'Canary10Percent',
            steps: [
              'Deploy to 10% of traffic',
              'Monitor metrics for 10 minutes',
              'Deploy to 25% of traffic',
              'Monitor metrics for 10 minutes',
              'Deploy to 100% of traffic'
            ],
            rollbackOnAlarm: true
          }
        ]
      },
      
      // Stage 6: Post-Deployment
      {
        name: 'PostDeployment',
        actions: [
          {
            name: 'Verification',
            steps: [
              'Run smoke tests',
              'Verify metrics',
              'Check error rates',
              'Validate logs'
            ]
          },
          {
            name: 'Notifications',
            channels: ['Slack', 'Email', 'PagerDuty']
          }
        ]
      }
    ]
  },
  
  // Infrastructure as Code
  infrastructureAsCode: {
    tool: 'Terraform',
    modules: [
      'vpc',
      'security-groups',
      'ecs-cluster',
      'lambda-functions',
      'rds-databases',
      'dynamodb-tables',
      'api-gateway',
      'cloudwatch-alarms'
    ],
    
    state: {
      backend: 's3',
      bucket: 'terraform-state-bucket',
      dynamodbTable: 'terraform-state-lock',
      encryption: true
    }
  }
};
```

---

## 💡 Design Principles

### 1. **Single Responsibility Principle**
Each microservice is responsible for a single business capability.

### 2. **Loose Coupling**
Services communicate through well-defined APIs and events, not direct dependencies.

### 3. **High Cohesion**
Related functionality is grouped together within a service.

### 4. **Resilience**
Services handle failures gracefully with circuit breakers, retries, and fallbacks.

### 5. **Scalability**
Each service can scale independently based on its specific load.

### 6. **Observability**
Comprehensive monitoring, logging, and tracing for all services.

### 7. **Security**
Defense in depth with multiple security layers.

### 8. **Automation**
Automated deployment, testing, and remediation.

---

## 🎯 Non-Functional Requirements

```javascript
const nfr = {
  performance: {
    apiLatency: {
      p50: '< 100ms',
      p95: '< 200ms',
      p99: '< 500ms'
    },
    throughput: '10,000 requests/second (peak)',
    concurrency: '50,000 concurrent users'
  },
  
  availability: {
    uptime: '99.99% (< 52.56 minutes downtime/year)',
    mttr: '< 15 minutes',  // Mean Time To Recovery
    mtbf: '> 30 days'      // Mean Time Between Failures
  },
  
  scalability: {
    horizontal: 'Auto-scale to 1000+ instances',
    vertical: 'Scale individual services independently',
    data: 'Support 100TB+ of data'
  },
  
  security: {
    authentication: 'Multi-factor authentication',
    authorization: 'Role-based access control',
    encryption: 'At rest and in transit',
    compliance: 'SOC 2, ISO 27001, GDPR, PCI-DSS'
  },
  
  reliability: {
    dataLoss: 'Zero tolerance',
    rto: '< 1 hour',   // Recovery Time Objective
    rpo: '< 5 minutes', // Recovery Point Objective
    backup: 'Automated daily backups with 30-day retention'
  },
  
  maintainability: {
    codeQuality: 'Test coverage > 80%',
    documentation: 'API docs, architecture docs, runbooks',
    monitoring: 'Real-time dashboards and alerts',
    deployment: 'Zero-downtime deployments'
  }
};
```

---

**Next Document:** 👉 [02_PREREQUISITES_SETUP.md](./02_PREREQUISITES_SETUP.md)

---

**Last Updated**: November 15, 2025  
**Version**: 1.0.0  
**Document Owner**: Platform Architecture Team  
**Review Cycle**: Quarterly
