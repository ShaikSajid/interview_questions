# DynamoDB Setup - Serverless Order Processing Microservice

## 📋 Overview

This guide covers the complete DynamoDB setup for the serverless microservice, including table design, indexes, auto-scaling, backup strategies, and CloudFormation templates.

---

## 🎯 Database Design Goals

- **High Performance**: Single-digit millisecond latency
- **Scalability**: Handle millions of requests per day
- **Cost-Efficient**: Pay-per-request for variable workloads
- **Reliable**: Point-in-time recovery and automated backups
- **Flexible**: Support multiple access patterns

---

## 1️⃣ Table Design - Orders Table

### Schema Design

```javascript
const ordersTableDesign = {
  tableName: 'orders',
  
  // Primary Key
  primaryKey: {
    partitionKey: {
      name: 'orderId',
      type: 'String',
      format: 'ORD-{timestamp}-{random}',
      example: 'ORD-1699804800000-A1B2C3'
    }
  },
  
  // Attributes
  attributes: {
    orderId: 'String',              // PK
    customerId: 'String',           // GSI1-PK
    orderStatus: 'String',          // GSI2-PK (PENDING, PROCESSING, COMPLETED, CANCELLED, FAILED)
    orderDate: 'String',            // ISO 8601 timestamp
    items: [                        // Array of order items
      {
        productId: 'String',
        productName: 'String',
        quantity: 'Number',
        unitPrice: 'Number',
        totalPrice: 'Number'
      }
    ],
    totalAmount: 'Number',          // Sum of all items
    currency: 'String',             // USD, EUR, etc.
    shippingAddress: {
      street: 'String',
      city: 'String',
      state: 'String',
      zipCode: 'String',
      country: 'String'
    },
    billingAddress: {
      street: 'String',
      city: 'String',
      state: 'String',
      zipCode: 'String',
      country: 'String'
    },
    paymentMethod: 'String',        // credit_card, paypal, etc.
    paymentStatus: 'String',        // pending, paid, failed
    createdAt: 'String',            // ISO 8601
    updatedAt: 'String',            // ISO 8601
    version: 'Number',              // For optimistic locking
    metadata: {                     // Flexible metadata
      source: 'String',             // web, mobile, api
      userAgent: 'String',
      ipAddress: 'String'
    }
  },
  
  // Access Patterns
  accessPatterns: [
    '1. Get order by orderId (PK)',
    '2. Get all orders for a customer (GSI1: customerId)',
    '3. Get orders by status (GSI2: orderStatus)',
    '4. Get orders by date range (GSI1: customerId + orderDate)',
    '5. Get recent orders (GSI2: orderStatus + orderDate)'
  ]
};
```

---

## 2️⃣ Table Design - Inventory Table

### Schema Design

```javascript
const inventoryTableDesign = {
  tableName: 'inventory',
  
  // Primary Key
  primaryKey: {
    partitionKey: {
      name: 'productId',
      type: 'String',
      format: 'PROD-{category}-{id}',
      example: 'PROD-ELECTRONICS-12345'
    }
  },
  
  // Attributes
  attributes: {
    productId: 'String',            // PK
    productName: 'String',
    sku: 'String',                  // Stock Keeping Unit
    category: 'String',
    availableQuantity: 'Number',    // Current available stock
    reservedQuantity: 'Number',     // Reserved but not fulfilled
    totalQuantity: 'Number',        // Total in warehouse
    reorderPoint: 'Number',         // When to reorder
    reorderQuantity: 'Number',      // How much to reorder
    unitCost: 'Number',
    unitPrice: 'Number',
    supplier: {
      supplierId: 'String',
      supplierName: 'String',
      contactEmail: 'String'
    },
    lastRestocked: 'String',        // ISO 8601
    createdAt: 'String',
    updatedAt: 'String',
    version: 'Number'               // For optimistic locking
  },
  
  // Access Patterns
  accessPatterns: [
    '1. Get product by productId (PK)',
    '2. Check availability before order',
    '3. Update quantity atomically',
    '4. Find low stock items (Scan with filter)'
  ]
};
```

---

## 3️⃣ CloudFormation Template - DynamoDB Tables

Create file: `infrastructure/cloudformation/dynamodb-stack.yaml`

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'DynamoDB Tables for Serverless Order Processing'

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - staging
      - production

Mappings:
  CapacitySettings:
    dev:
      BillingMode: PAY_PER_REQUEST
      ReadCapacity: 0
      WriteCapacity: 0
    staging:
      BillingMode: PROVISIONED
      ReadCapacity: 5
      WriteCapacity: 5
    production:
      BillingMode: PROVISIONED
      ReadCapacity: 100
      WriteCapacity: 50

Resources:
  # ========================================
  # Orders Table
  # ========================================
  OrdersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: !Sub '${Environment}-orders'
      BillingMode: !FindInMap [CapacitySettings, !Ref Environment, BillingMode]
      
      AttributeDefinitions:
        - AttributeName: orderId
          AttributeType: S
        - AttributeName: customerId
          AttributeType: S
        - AttributeName: orderStatus
          AttributeType: S
        - AttributeName: orderDate
          AttributeType: S
      
      KeySchema:
        - AttributeName: orderId
          KeyType: HASH
      
      # Global Secondary Indexes
      GlobalSecondaryIndexes:
        # GSI1: Query orders by customer
        - IndexName: customer-index
          KeySchema:
            - AttributeName: customerId
              KeyType: HASH
            - AttributeName: orderDate
              KeyType: RANGE
          Projection:
            ProjectionType: ALL
          ProvisionedThroughput:
            ReadCapacityUnits: !If
              - IsProvisioned
              - !FindInMap [CapacitySettings, !Ref Environment, ReadCapacity]
              - !Ref AWS::NoValue
            WriteCapacityUnits: !If
              - IsProvisioned
              - !FindInMap [CapacitySettings, !Ref Environment, WriteCapacity]
              - !Ref AWS::NoValue
        
        # GSI2: Query orders by status
        - IndexName: status-index
          KeySchema:
            - AttributeName: orderStatus
              KeyType: HASH
            - AttributeName: orderDate
              KeyType: RANGE
          Projection:
            ProjectionType: ALL
          ProvisionedThroughput:
            ReadCapacityUnits: !If
              - IsProvisioned
              - !FindInMap [CapacitySettings, !Ref Environment, ReadCapacity]
              - !Ref AWS::NoValue
            WriteCapacityUnits: !If
              - IsProvisioned
              - !FindInMap [CapacitySettings, !Ref Environment, WriteCapacity]
              - !Ref AWS::NoValue
      
      # Provisioned Throughput (only for PROVISIONED mode)
      ProvisionedThroughput: !If
        - IsProvisioned
        - ReadCapacityUnits: !FindInMap [CapacitySettings, !Ref Environment, ReadCapacity]
          WriteCapacityUnits: !FindInMap [CapacitySettings, !Ref Environment, WriteCapacity]
        - !Ref AWS::NoValue
      
      # Point-in-Time Recovery
      PointInTimeRecoverySpecification:
        PointInTimeRecoveryEnabled: !If
          - IsProduction
          - true
          - false
      
      # Server-Side Encryption
      SSESpecification:
        SSEEnabled: true
        SSEType: KMS
        KMSMasterKeyId: !ImportValue
          'Fn::Sub': '${Environment}-DynamoDB-KMS-Key-ID'
      
      # Stream Specification (for CDC - Change Data Capture)
      StreamSpecification:
        StreamViewType: NEW_AND_OLD_IMAGES
      
      # Time to Live
      TimeToLiveSpecification:
        AttributeName: ttl
        Enabled: false  # Set to true if you want automatic deletion
      
      # Tags
      Tags:
        - Key: Environment
          Value: !Ref Environment
        - Key: Service
          Value: order-processing
        - Key: ManagedBy
          Value: CloudFormation

  # ========================================
  # Orders Table Auto Scaling (Production only)
  # ========================================
  OrdersTableReadScalingTarget:
    Type: AWS::ApplicationAutoScaling::ScalableTarget
    Condition: IsProductionProvisioned
    Properties:
      ServiceNamespace: dynamodb
      ResourceId: !Sub 'table/${OrdersTable}'
      ScalableDimension: dynamodb:table:ReadCapacityUnits
      MinCapacity: 5
      MaxCapacity: 1000
      RoleARN: !Sub 'arn:aws:iam::${AWS::AccountId}:role/aws-service-role/dynamodb.application-autoscaling.amazonaws.com/AWSServiceRoleForApplicationAutoScaling_DynamoDBTable'

  OrdersTableReadScalingPolicy:
    Type: AWS::ApplicationAutoScaling::ScalingPolicy
    Condition: IsProductionProvisioned
    Properties:
      PolicyName: ReadAutoScalingPolicy
      PolicyType: TargetTrackingScaling
      ScalingTargetId: !Ref OrdersTableReadScalingTarget
      TargetTrackingScalingPolicyConfiguration:
        TargetValue: 70.0
        PredefinedMetricSpecification:
          PredefinedMetricType: DynamoDBReadCapacityUtilization

  OrdersTableWriteScalingTarget:
    Type: AWS::ApplicationAutoScaling::ScalableTarget
    Condition: IsProductionProvisioned
    Properties:
      ServiceNamespace: dynamodb
      ResourceId: !Sub 'table/${OrdersTable}'
      ScalableDimension: dynamodb:table:WriteCapacityUnits
      MinCapacity: 5
      MaxCapacity: 1000
      RoleARN: !Sub 'arn:aws:iam::${AWS::AccountId}:role/aws-service-role/dynamodb.application-autoscaling.amazonaws.com/AWSServiceRoleForApplicationAutoScaling_DynamoDBTable'

  OrdersTableWriteScalingPolicy:
    Type: AWS::ApplicationAutoScaling::ScalingPolicy
    Condition: IsProductionProvisioned
    Properties:
      PolicyName: WriteAutoScalingPolicy
      PolicyType: TargetTrackingScaling
      ScalingTargetId: !Ref OrdersTableWriteScalingTarget
      TargetTrackingScalingPolicyConfiguration:
        TargetValue: 70.0
        PredefinedMetricSpecification:
          PredefinedMetricType: DynamoDBWriteCapacityUtilization

  # ========================================
  # Inventory Table
  # ========================================
  InventoryTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: !Sub '${Environment}-inventory'
      BillingMode: !FindInMap [CapacitySettings, !Ref Environment, BillingMode]
      
      AttributeDefinitions:
        - AttributeName: productId
          AttributeType: S
      
      KeySchema:
        - AttributeName: productId
          KeyType: HASH
      
      ProvisionedThroughput: !If
        - IsProvisioned
        - ReadCapacityUnits: !FindInMap [CapacitySettings, !Ref Environment, ReadCapacity]
          WriteCapacityUnits: !FindInMap [CapacitySettings, !Ref Environment, WriteCapacity]
        - !Ref AWS::NoValue
      
      PointInTimeRecoverySpecification:
        PointInTimeRecoveryEnabled: !If
          - IsProduction
          - true
          - false
      
      SSESpecification:
        SSEEnabled: true
        SSEType: KMS
        KMSMasterKeyId: !ImportValue
          'Fn::Sub': '${Environment}-DynamoDB-KMS-Key-ID'
      
      Tags:
        - Key: Environment
          Value: !Ref Environment
        - Key: Service
          Value: order-processing

  # ========================================
  # Backup Plan
  # ========================================
  BackupVault:
    Type: AWS::Backup::BackupVault
    Condition: IsProduction
    Properties:
      BackupVaultName: !Sub '${Environment}-dynamodb-vault'

  BackupPlan:
    Type: AWS::Backup::BackupPlan
    Condition: IsProduction
    Properties:
      BackupPlan:
        BackupPlanName: !Sub '${Environment}-dynamodb-backup-plan'
        BackupPlanRule:
          - RuleName: DailyBackup
            TargetBackupVault: !Ref BackupVault
            ScheduleExpression: 'cron(0 5 ? * * *)'  # Daily at 5 AM UTC
            StartWindowMinutes: 60
            CompletionWindowMinutes: 120
            Lifecycle:
              DeleteAfterDays: 30
              MoveToColdStorageAfterDays: 7
          - RuleName: WeeklyBackup
            TargetBackupVault: !Ref BackupVault
            ScheduleExpression: 'cron(0 6 ? * 1 *)'  # Weekly on Monday
            Lifecycle:
              DeleteAfterDays: 90

  BackupSelection:
    Type: AWS::Backup::BackupSelection
    Condition: IsProduction
    Properties:
      BackupPlanId: !Ref BackupPlan
      BackupSelection:
        SelectionName: DynamoDBTables
        IamRoleArn: !Sub 'arn:aws:iam::${AWS::AccountId}:role/service-role/AWSBackupDefaultServiceRole'
        Resources:
          - !GetAtt OrdersTable.Arn
          - !GetAtt InventoryTable.Arn

# ========================================
# Conditions
# ========================================
Conditions:
  IsProvisioned: !Not
    - !Equals
      - !FindInMap [CapacitySettings, !Ref Environment, BillingMode]
      - PAY_PER_REQUEST
  
  IsProduction: !Equals
    - !Ref Environment
    - production
  
  IsProductionProvisioned: !And
    - !Condition IsProduction
    - !Condition IsProvisioned

# ========================================
# Outputs
# ========================================
Outputs:
  OrdersTableName:
    Description: Orders Table Name
    Value: !Ref OrdersTable
    Export:
      Name: !Sub '${Environment}-OrdersTable-Name'

  OrdersTableArn:
    Description: Orders Table ARN
    Value: !GetAtt OrdersTable.Arn
    Export:
      Name: !Sub '${Environment}-OrdersTable-Arn'

  OrdersTableStreamArn:
    Description: Orders Table Stream ARN
    Value: !GetAtt OrdersTable.StreamArn
    Export:
      Name: !Sub '${Environment}-OrdersTable-StreamArn'

  InventoryTableName:
    Description: Inventory Table Name
    Value: !Ref InventoryTable
    Export:
      Name: !Sub '${Environment}-InventoryTable-Name'

  InventoryTableArn:
    Description: Inventory Table ARN
    Value: !GetAtt InventoryTable.Arn
    Export:
      Name: !Sub '${Environment}-InventoryTable-Arn'
```

---

## 4️⃣ Deploy DynamoDB Stack

```powershell
# Deploy DynamoDB tables
aws cloudformation create-stack `
  --stack-name dev-dynamodb-stack `
  --template-body file://infrastructure/cloudformation/dynamodb-stack.yaml `
  --parameters ParameterKey=Environment,ParameterValue=dev `
  --region us-east-1

# Wait for completion
aws cloudformation wait stack-create-complete --stack-name dev-dynamodb-stack

# Get outputs
aws cloudformation describe-stacks `
  --stack-name dev-dynamodb-stack `
  --query 'Stacks[0].Outputs'
```

---

## 5️⃣ DynamoDB Operations - TypeScript SDK

### Initialize DynamoDB Client

```typescript
// src/utils/dynamodb.ts
import { DynamoDB } from 'aws-sdk';
import { DocumentClient } from 'aws-sdk/clients/dynamodb';

const dynamodb = new DynamoDB();
const docClient = new DocumentClient({
  region: process.env.AWS_REGION || 'us-east-1',
  maxRetries: 3,
  httpOptions: {
    timeout: 5000,
    connectTimeout: 3000
  }
});

export { dynamodb, docClient };
```

### Create Order

```typescript
// src/services/order.service.ts
import { docClient } from '../utils/dynamodb';
import { v4 as uuidv4 } from 'uuid';

export interface Order {
  orderId: string;
  customerId: string;
  orderStatus: string;
  orderDate: string;
  items: OrderItem[];
  totalAmount: number;
  currency: string;
  shippingAddress: Address;
  billingAddress: Address;
  createdAt: string;
  updatedAt: string;
  version: number;
}

export interface OrderItem {
  productId: string;
  productName: string;
  quantity: number;
  unitPrice: number;
  totalPrice: number;
}

export interface Address {
  street: string;
  city: string;
  state: string;
  zipCode: string;
  country: string;
}

export async function createOrder(orderData: Partial<Order>): Promise<Order> {
  const timestamp = new Date().toISOString();
  const orderId = `ORD-${Date.now()}-${uuidv4().substring(0, 6).toUpperCase()}`;

  const order: Order = {
    orderId,
    customerId: orderData.customerId!,
    orderStatus: 'PENDING',
    orderDate: timestamp,
    items: orderData.items!,
    totalAmount: orderData.totalAmount!,
    currency: orderData.currency || 'USD',
    shippingAddress: orderData.shippingAddress!,
    billingAddress: orderData.billingAddress!,
    createdAt: timestamp,
    updatedAt: timestamp,
    version: 1
  };

  const params: DocumentClient.PutItemInput = {
    TableName: process.env.ORDERS_TABLE!,
    Item: order,
    ConditionExpression: 'attribute_not_exists(orderId)'  // Ensure no duplicate
  };

  try {
    await docClient.put(params).promise();
    console.log(`Order created: ${orderId}`);
    return order;
  } catch (error) {
    console.error('Error creating order:', error);
    throw error;
  }
}
```

### Get Order by ID

```typescript
export async function getOrderById(orderId: string): Promise<Order | null> {
  const params: DocumentClient.GetItemInput = {
    TableName: process.env.ORDERS_TABLE!,
    Key: { orderId }
  };

  try {
    const result = await docClient.get(params).promise();
    return (result.Item as Order) || null;
  } catch (error) {
    console.error(`Error getting order ${orderId}:`, error);
    throw error;
  }
}
```

### Get Orders by Customer

```typescript
export async function getOrdersByCustomer(
  customerId: string,
  limit: number = 20,
  lastEvaluatedKey?: any
): Promise<{ orders: Order[]; lastEvaluatedKey?: any }> {
  const params: DocumentClient.QueryInput = {
    TableName: process.env.ORDERS_TABLE!,
    IndexName: 'customer-index',
    KeyConditionExpression: 'customerId = :customerId',
    ExpressionAttributeValues: {
      ':customerId': customerId
    },
    Limit: limit,
    ScanIndexForward: false,  // Descending order (newest first)
    ExclusiveStartKey: lastEvaluatedKey
  };

  try {
    const result = await docClient.query(params).promise();
    return {
      orders: (result.Items as Order[]) || [],
      lastEvaluatedKey: result.LastEvaluatedKey
    };
  } catch (error) {
    console.error(`Error getting orders for customer ${customerId}:`, error);
    throw error;
  }
}
```

### Update Order Status

```typescript
export async function updateOrderStatus(
  orderId: string,
  newStatus: string,
  currentVersion: number
): Promise<Order> {
  const timestamp = new Date().toISOString();

  const params: DocumentClient.UpdateItemInput = {
    TableName: process.env.ORDERS_TABLE!,
    Key: { orderId },
    UpdateExpression: 'SET orderStatus = :status, updatedAt = :updatedAt, version = :newVersion',
    ConditionExpression: 'version = :currentVersion',  // Optimistic locking
    ExpressionAttributeValues: {
      ':status': newStatus,
      ':updatedAt': timestamp,
      ':currentVersion': currentVersion,
      ':newVersion': currentVersion + 1
    },
    ReturnValues: 'ALL_NEW'
  };

  try {
    const result = await docClient.update(params).promise();
    return result.Attributes as Order;
  } catch (error) {
    if ((error as any).code === 'ConditionalCheckFailedException') {
      throw new Error('Order version mismatch - concurrent update detected');
    }
    console.error(`Error updating order ${orderId}:`, error);
    throw error;
  }
}
```

### Inventory Operations

```typescript
// src/services/inventory.service.ts
import { docClient } from '../utils/dynamodb';

export interface InventoryItem {
  productId: string;
  productName: string;
  sku: string;
  availableQuantity: number;
  reservedQuantity: number;
  totalQuantity: number;
  reorderPoint: number;
  reorderQuantity: number;
  updatedAt: string;
  version: number;
}

export async function checkInventory(productId: string): Promise<InventoryItem | null> {
  const params: DocumentClient.GetItemInput = {
    TableName: process.env.INVENTORY_TABLE!,
    Key: { productId }
  };

  try {
    const result = await docClient.get(params).promise();
    return (result.Item as InventoryItem) || null;
  } catch (error) {
    console.error(`Error checking inventory for ${productId}:`, error);
    throw error;
  }
}

export async function reserveInventory(
  productId: string,
  quantity: number,
  currentVersion: number
): Promise<InventoryItem> {
  const timestamp = new Date().toISOString();

  const params: DocumentClient.UpdateItemInput = {
    TableName: process.env.INVENTORY_TABLE!,
    Key: { productId },
    UpdateExpression: `
      SET availableQuantity = availableQuantity - :quantity,
          reservedQuantity = reservedQuantity + :quantity,
          updatedAt = :updatedAt,
          version = :newVersion
    `,
    ConditionExpression: `
      availableQuantity >= :quantity AND version = :currentVersion
    `,
    ExpressionAttributeValues: {
      ':quantity': quantity,
      ':updatedAt': timestamp,
      ':currentVersion': currentVersion,
      ':newVersion': currentVersion + 1
    },
    ReturnValues: 'ALL_NEW'
  };

  try {
    const result = await docClient.update(params).promise();
    return result.Attributes as InventoryItem;
  } catch (error) {
    if ((error as any).code === 'ConditionalCheckFailedException') {
      throw new Error('Insufficient inventory or version mismatch');
    }
    console.error(`Error reserving inventory for ${productId}:`, error);
    throw error;
  }
}

export async function releaseInventory(
  productId: string,
  quantity: number
): Promise<InventoryItem> {
  const timestamp = new Date().toISOString();

  const params: DocumentClient.UpdateItemInput = {
    TableName: process.env.INVENTORY_TABLE!,
    Key: { productId },
    UpdateExpression: `
      SET availableQuantity = availableQuantity + :quantity,
          reservedQuantity = reservedQuantity - :quantity,
          updatedAt = :updatedAt,
          version = version + :one
    `,
    ConditionExpression: 'reservedQuantity >= :quantity',
    ExpressionAttributeValues: {
      ':quantity': quantity,
      ':updatedAt': timestamp,
      ':one': 1
    },
    ReturnValues: 'ALL_NEW'
  };

  try {
    const result = await docClient.update(params).promise();
    return result.Attributes as InventoryItem;
  } catch (error) {
    console.error(`Error releasing inventory for ${productId}:`, error);
    throw error;
  }
}
```

---

## 6️⃣ Seed Data Script

```typescript
// scripts/seed-dynamodb.ts
import { docClient } from '../src/utils/dynamodb';

async function seedOrders() {
  const sampleOrders = [
    {
      orderId: 'ORD-1699804800000-ABC123',
      customerId: 'CUST-001',
      orderStatus: 'COMPLETED',
      orderDate: '2024-11-10T10:00:00Z',
      items: [
        {
          productId: 'PROD-ELECTRONICS-12345',
          productName: 'Laptop',
          quantity: 1,
          unitPrice: 1200.00,
          totalPrice: 1200.00
        }
      ],
      totalAmount: 1200.00,
      currency: 'USD',
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
      },
      createdAt: '2024-11-10T10:00:00Z',
      updatedAt: '2024-11-10T12:00:00Z',
      version: 1
    }
  ];

  for (const order of sampleOrders) {
    await docClient.put({
      TableName: process.env.ORDERS_TABLE!,
      Item: order
    }).promise();
    console.log(`✅ Seeded order: ${order.orderId}`);
  }
}

async function seedInventory() {
  const sampleProducts = [
    {
      productId: 'PROD-ELECTRONICS-12345',
      productName: 'Laptop',
      sku: 'LAP-2024-001',
      category: 'Electronics',
      availableQuantity: 100,
      reservedQuantity: 0,
      totalQuantity: 100,
      reorderPoint: 20,
      reorderQuantity: 50,
      unitCost: 800.00,
      unitPrice: 1200.00,
      createdAt: '2024-01-01T00:00:00Z',
      updatedAt: '2024-11-10T00:00:00Z',
      version: 1
    }
  ];

  for (const product of sampleProducts) {
    await docClient.put({
      TableName: process.env.INVENTORY_TABLE!,
      Item: product
    }).promise();
    console.log(`✅ Seeded product: ${product.productId}`);
  }
}

async function main() {
  console.log('🌱 Seeding DynamoDB tables...\n');
  
  await seedOrders();
  await seedInventory();
  
  console.log('\n✨ Seeding complete!');
}

main().catch(console.error);
```

```powershell
# Run seed script
$env:ORDERS_TABLE="dev-orders"
$env:INVENTORY_TABLE="dev-inventory"
$env:AWS_REGION="us-east-1"
npx ts-node scripts/seed-dynamodb.ts
```

---

## 🎯 Best Practices

### DO ✅

1. **Use partition keys wisely**: Distribute data evenly
2. **Implement optimistic locking**: Use version numbers
3. **Use GSIs for access patterns**: Avoid scans
4. **Enable Point-in-Time Recovery**: For production
5. **Use KMS encryption**: Secure sensitive data
6. **Implement exponential backoff**: For retries
7. **Use batch operations**: When possible
8. **Monitor capacity**: Set CloudWatch alarms
9. **Use DynamoDB Streams**: For event-driven architectures
10. **Test with realistic data**: Load test before production

### DON'T ❌

1. **Don't use Scan**: Use Query with indexes instead
2. **Don't store large items**: Keep items under 400KB
3. **Don't create hot partitions**: Distribute writes evenly
4. **Don't ignore throttling**: Implement backoff and retry
5. **Don't skip error handling**: Handle ConditionalCheckFailed
6. **Don't over-provision**: Start small, scale as needed
7. **Don't forget TTL**: Clean up old data automatically
8. **Don't ignore costs**: Monitor and optimize
9. **Don't skip backups**: Enable automated backups
10. **Don't use reserved capacity**: Unless predictable workload

---

## 📊 Cost Optimization

```javascript
const dynamoDBCosts = {
  payPerRequest: {
    writes: '$1.25 per million',
    reads: '$0.25 per million',
    storage: '$0.25 per GB/month',
    example: {
      writes: '1M writes/month = $1.25',
      reads: '10M reads/month = $2.50',
      storage: '10GB = $2.50',
      total: '$6.25/month'
    }
  },
  provisioned: {
    writes: '$0.00065 per WCU/hour',
    reads: '$0.00013 per RCU/hour',
    example: {
      writes: '10 WCU * 730 hours = $4.75',
      reads: '50 RCU * 730 hours = $4.75',
      total: '$9.50/month'
    }
  },
  recommendation: 'Use PAY_PER_REQUEST for dev/staging, PROVISIONED for production'
};
```

---

## 🎯 Next Steps

DynamoDB setup complete! You now have:

- ✅ Orders table with GSIs
- ✅ Inventory table with atomic operations
- ✅ Auto-scaling configured (production)
- ✅ Point-in-time recovery enabled
- ✅ KMS encryption configured
- ✅ Backup plan configured
- ✅ TypeScript SDK operations
- ✅ Seed data scripts

**Ready to proceed?**

👉 [SQS/SNS Setup](./06_SQS_SNS_SETUP.md)

---

**Last Updated**: November 12, 2025  
**Version**: 1.0.0
