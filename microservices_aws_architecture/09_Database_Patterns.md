# Database Patterns for Microservices

## Question 17: Database per Service Pattern

### 📋 Question Statement

Implement the database per service pattern for Emirates NBD with proper data isolation, distributed transactions using Saga pattern, and data synchronization strategies. Include RDS, DynamoDB, and Aurora implementations.

---

### 🏗️ Database per Service Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│            EMIRATES NBD DATABASE PER SERVICE PATTERN                        │
└────────────────────────────────────────────────────────────────────────────┘

    SERVICE 1: Account Service          SERVICE 2: Transaction Service
    ─────────────────────────          ──────────────────────────────
    
    ┌──────────────────┐                ┌──────────────────┐
    │  Account Service │                │ Transaction Svc  │
    │  (ECS)           │                │  (Lambda)        │
    └────────┬─────────┘                └────────┬─────────┘
             │                                   │
             │ Private DB                        │ Private DB
             │ (No shared access)                │ (No shared access)
             ▼                                   ▼
    ┌──────────────────┐                ┌──────────────────┐
    │  RDS PostgreSQL  │                │   DynamoDB       │
    │  (Accounts)      │                │  (Transactions)  │
    │                  │                │                  │
    │  • ACID          │                │  • High TPS      │
    │  • Relational    │                │  • NoSQL         │
    │  • Complex       │                │  • Global Tables │
    │    Queries       │                │                  │
    └──────────────────┘                └──────────────────┘

    SERVICE 3: Payment Service          SERVICE 4: Loan Service
    ──────────────────────────          ───────────────────────
    
    ┌──────────────────┐                ┌──────────────────┐
    │  Payment Service │                │   Loan Service   │
    │  (EKS)           │                │   (ECS)          │
    └────────┬─────────┘                └────────┬─────────┘
             │                                   │
             │                                   │
             ▼                                   ▼
    ┌──────────────────┐                ┌──────────────────┐
    │ Aurora Serverless│                │  RDS MySQL       │
    │  (Payments)      │                │  (Loans)         │
    │                  │                │                  │
    │  • Auto-scaling  │                │  • Multi-AZ      │
    │  • Serverless    │                │  • Read Replicas │
    │  • PCI Compliant │                │                  │
    └──────────────────┘                └──────────────────┘

    DATA SYNCHRONIZATION
    ────────────────────
    
    ┌──────────────────────────────────────────────────┐
    │         EventBridge (Cross-Service Events)       │
    └────────────────┬─────────────────────────────────┘
                     │
         ┌───────────┼───────────┐
         │           │           │
         ▼           ▼           ▼
    [Account     [Transaction  [Payment
     Created]     Completed]    Processed]
         │           │           │
         └───────────┼───────────┘
                     │
                     ▼
    ┌──────────────────────────────────────────────────┐
    │      Read Models / Materialized Views            │
    │      (DynamoDB, ElastiCache)                     │
    └──────────────────────────────────────────────────┘
```

---

### 🔧 Complete Database Infrastructure (CDK)

```typescript
// infrastructure/cdk/database-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as rds from 'aws-cdk-lib/aws-rds';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as secretsmanager from 'aws-cdk-lib/aws-secretsmanager';

export class DatabasePerServiceStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, 'BankingVPC', { maxAzs: 3 });

    // ============================================
    // 1. ACCOUNT SERVICE - RDS PostgreSQL
    // ============================================
    const accountDBSecret = new secretsmanager.Secret(this, 'AccountDBSecret', {
      generateSecretString: {
        secretStringTemplate: JSON.stringify({ username: 'accountadmin' }),
        generateStringKey: 'password',
        excludePunctuation: true
      }
    });

    const accountDB = new rds.DatabaseInstance(this, 'AccountDB', {
      engine: rds.DatabaseInstanceEngine.postgres({ version: rds.PostgresEngineVersion.VER_15_3 }),
      instanceType: ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.MEDIUM),
      vpc,
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_ISOLATED },
      credentials: rds.Credentials.fromSecret(accountDBSecret),
      multiAz: true,
      allocatedStorage: 100,
      maxAllocatedStorage: 500,
      storageEncrypted: true,
      backupRetention: cdk.Duration.days(7),
      deletionProtection: true,
      databaseName: 'accounts',
      cloudwatchLogsExports: ['postgresql']
    });

    // ============================================
    // 2. TRANSACTION SERVICE - DynamoDB
    // ============================================
    const transactionTable = new dynamodb.Table(this, 'TransactionTable', {
      tableName: 'banking-transactions',
      partitionKey: { name: 'transactionId', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'timestamp', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      stream: dynamodb.StreamViewType.NEW_AND_OLD_IMAGES,
      pointInTimeRecovery: true,
      encryption: dynamodb.TableEncryption.AWS_MANAGED
    });

    transactionTable.addGlobalSecondaryIndex({
      indexName: 'AccountIdIndex',
      partitionKey: { name: 'fromAccount', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'timestamp', type: dynamodb.AttributeType.STRING }
    });

    // ============================================
    // 3. PAYMENT SERVICE - Aurora Serverless
    // ============================================
    const paymentDBSecret = new secretsmanager.Secret(this, 'PaymentDBSecret', {
      generateSecretString: {
        secretStringTemplate: JSON.stringify({ username: 'paymentadmin' }),
        generateStringKey: 'password',
        excludePunctuation: true
      }
    });

    const paymentDB = new rds.ServerlessCluster(this, 'PaymentDB', {
      engine: rds.DatabaseClusterEngine.auroraPostgres({
        version: rds.AuroraPostgresEngineVersion.VER_13_9
      }),
      vpc,
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_ISOLATED },
      credentials: rds.Credentials.fromSecret(paymentDBSecret),
      scaling: {
        minCapacity: rds.AuroraCapacityUnit.ACU_2,
        maxCapacity: rds.AuroraCapacityUnit.ACU_16,
        autoPause: cdk.Duration.minutes(10)
      },
      defaultDatabaseName: 'payments',
      enableDataApi: true
    });

    // ============================================
    // 4. LOAN SERVICE - RDS MySQL with Read Replica
    // ============================================
    const loanDBSecret = new secretsmanager.Secret(this, 'LoanDBSecret', {
      generateSecretString: {
        secretStringTemplate: JSON.stringify({ username: 'loanadmin' }),
        generateStringKey: 'password',
        excludePunctuation: true
      }
    });

    const loanDB = new rds.DatabaseInstance(this, 'LoanDB', {
      engine: rds.DatabaseInstanceEngine.mysql({ version: rds.MysqlEngineVersion.VER_8_0_35 }),
      instanceType: ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.MEDIUM),
      vpc,
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_ISOLATED },
      credentials: rds.Credentials.fromSecret(loanDBSecret),
      multiAz: true,
      allocatedStorage: 100,
      storageEncrypted: true,
      databaseName: 'loans'
    });

    // Read replica for reporting
    new rds.DatabaseInstanceReadReplica(this, 'LoanDBReadReplica', {
      sourceDatabaseInstance: loanDB,
      instanceType: ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.SMALL),
      vpc,
      publiclyAccessible: false
    });

    // ============================================
    // 5. MATERIALIZED VIEW - DynamoDB
    // ============================================
    const customerViewTable = new dynamodb.Table(this, 'CustomerViewTable', {
      tableName: 'customer-aggregated-view',
      partitionKey: { name: 'customerId', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST
    });

    new cdk.CfnOutput(this, 'AccountDBEndpoint', {
      value: accountDB.dbInstanceEndpointAddress
    });

    new cdk.CfnOutput(this, 'TransactionTableName', {
      value: transactionTable.tableName
    });
  }
}
```

### 📊 Database Schemas

```sql
-- Account Service Database (PostgreSQL)
-- accounts_schema.sql

CREATE TABLE customers (
    customer_id VARCHAR(50) PRIMARY KEY,
    full_name VARCHAR(200) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone VARCHAR(20),
    date_of_birth DATE,
    kyc_status VARCHAR(20) DEFAULT 'PENDING',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE accounts (
    account_id VARCHAR(50) PRIMARY KEY,
    customer_id VARCHAR(50) REFERENCES customers(customer_id),
    account_type VARCHAR(20) NOT NULL CHECK (account_type IN ('SAVINGS', 'CURRENT', 'FIXED_DEPOSIT')),
    currency VARCHAR(3) DEFAULT 'AED',
    balance DECIMAL(15, 2) DEFAULT 0.00,
    reserved_funds DECIMAL(15, 2) DEFAULT 0.00,
    status VARCHAR(20) DEFAULT 'ACTIVE',
    daily_transfer_limit DECIMAL(15, 2) DEFAULT 50000.00,
    version INTEGER DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT positive_balance CHECK (balance >= 0),
    CONSTRAINT positive_reserved CHECK (reserved_funds >= 0)
);

CREATE INDEX idx_accounts_customer ON accounts(customer_id);
CREATE INDEX idx_accounts_status ON accounts(status);

-- Audit table
CREATE TABLE account_audit (
    audit_id SERIAL PRIMARY KEY,
    account_id VARCHAR(50) REFERENCES accounts(account_id),
    operation VARCHAR(20),
    old_balance DECIMAL(15, 2),
    new_balance DECIMAL(15, 2),
    changed_by VARCHAR(100),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Payment Service Database (Aurora PostgreSQL)
-- payments_schema.sql

CREATE TABLE payments (
    payment_id VARCHAR(50) PRIMARY KEY,
    from_account VARCHAR(50) NOT NULL,
    to_account VARCHAR(50) NOT NULL,
    amount DECIMAL(15, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    payment_method VARCHAR(20),
    reference_number VARCHAR(100) UNIQUE,
    status VARCHAR(20) DEFAULT 'PENDING',
    failure_reason TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processed_at TIMESTAMP,
    completed_at TIMESTAMP,
    CONSTRAINT positive_amount CHECK (amount > 0)
);

CREATE TABLE payment_history (
    history_id SERIAL PRIMARY KEY,
    payment_id VARCHAR(50) REFERENCES payments(payment_id),
    status VARCHAR(20),
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_payments_from_account ON payments(from_account);
CREATE INDEX idx_payments_status ON payments(status);

-- Loan Service Database (MySQL)
-- loans_schema.sql

CREATE TABLE loan_applications (
    loan_id VARCHAR(50) PRIMARY KEY,
    customer_id VARCHAR(50) NOT NULL,
    account_id VARCHAR(50) NOT NULL,
    loan_type ENUM('PERSONAL', 'HOME', 'AUTO', 'BUSINESS') NOT NULL,
    requested_amount DECIMAL(15, 2) NOT NULL,
    approved_amount DECIMAL(15, 2),
    interest_rate DECIMAL(5, 2),
    tenure_months INT,
    status ENUM('APPLIED', 'UNDER_REVIEW', 'APPROVED', 'REJECTED', 'DISBURSED') DEFAULT 'APPLIED',
    application_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    approval_date TIMESTAMP,
    disbursement_date TIMESTAMP,
    INDEX idx_customer (customer_id),
    INDEX idx_status (status)
);

CREATE TABLE loan_payments (
    payment_id INT AUTO_INCREMENT PRIMARY KEY,
    loan_id VARCHAR(50) REFERENCES loan_applications(loan_id),
    emi_amount DECIMAL(15, 2) NOT NULL,
    principal_amount DECIMAL(15, 2) NOT NULL,
    interest_amount DECIMAL(15, 2) NOT NULL,
    due_date DATE NOT NULL,
    paid_date DATE,
    status ENUM('PENDING', 'PAID', 'OVERDUE', 'DEFAULTED') DEFAULT 'PENDING',
    INDEX idx_loan (loan_id),
    INDEX idx_due_date (due_date)
);
```

### 💾 Data Access Patterns

```javascript
// services/account-service/src/repositories/account-repository.js
const { Pool } = require('pg');

class AccountRepository {
  constructor() {
    this.pool = new Pool({
      host: process.env.DB_HOST,
      port: 5432,
      database: 'accounts',
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      max: 20,
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 2000
    });
  }

  async getAccount(accountId) {
    const result = await this.pool.query(
      'SELECT * FROM accounts WHERE account_id = $1',
      [accountId]
    );
    return result.rows[0];
  }

  async createAccount(account) {
    const { customer_id, account_type, currency, balance } = account;
    
    const result = await this.pool.query(
      `INSERT INTO accounts (account_id, customer_id, account_type, currency, balance)
       VALUES ($1, $2, $3, $4, $5)
       RETURNING *`,
      [`ACC-${Date.now()}`, customer_id, account_type, currency, balance]
    );
    
    return result.rows[0];
  }

  async updateBalance(accountId, amount, operation) {
    const client = await this.pool.connect();
    
    try {
      await client.query('BEGIN');
      
      // Lock row for update
      const current = await client.query(
        'SELECT balance, version FROM accounts WHERE account_id = $1 FOR UPDATE',
        [accountId]
      );
      
      if (current.rows.length === 0) {
        throw new Error('Account not found');
      }
      
      const oldBalance = current.rows[0].balance;
      const version = current.rows[0].version;
      let newBalance;
      
      if (operation === 'CREDIT') {
        newBalance = parseFloat(oldBalance) + parseFloat(amount);
      } else {
        if (oldBalance < amount) {
          throw new Error('Insufficient funds');
        }
        newBalance = parseFloat(oldBalance) - parseFloat(amount);
      }
      
      // Update with optimistic locking
      const updateResult = await client.query(
        `UPDATE accounts 
         SET balance = $1, version = version + 1, updated_at = CURRENT_TIMESTAMP
         WHERE account_id = $2 AND version = $3
         RETURNING *`,
        [newBalance, accountId, version]
      );
      
      if (updateResult.rows.length === 0) {
        throw new Error('Concurrent modification detected');
      }
      
      // Audit log
      await client.query(
        `INSERT INTO account_audit (account_id, operation, old_balance, new_balance, changed_by)
         VALUES ($1, $2, $3, $4, $5)`,
        [accountId, operation, oldBalance, newBalance, 'system']
      );
      
      await client.query('COMMIT');
      return updateResult.rows[0];
      
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  async getAccountsByCustomer(customerId) {
    const result = await this.pool.query(
      'SELECT * FROM accounts WHERE customer_id = $1',
      [customerId]
    );
    return result.rows;
  }

  async close() {
    await this.pool.end();
  }
}

module.exports = AccountRepository;
```

```javascript
// services/transaction-service/src/repositories/transaction-repository.js
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, PutCommand, GetCommand, QueryCommand } = require('@aws-sdk/lib-dynamodb');

class TransactionRepository {
  constructor() {
    const client = new DynamoDBClient({});
    this.docClient = DynamoDBDocumentClient.from(client);
    this.tableName = 'banking-transactions';
  }

  async createTransaction(transaction) {
    await this.docClient.send(new PutCommand({
      TableName: this.tableName,
      Item: {
        transactionId: transaction.transactionId,
        fromAccount: transaction.fromAccount,
        toAccount: transaction.toAccount,
        amount: transaction.amount,
        currency: transaction.currency,
        type: transaction.type,
        status: transaction.status || 'PENDING',
        timestamp: new Date().toISOString(),
        ttl: Math.floor(Date.now() / 1000) + (365 * 24 * 60 * 60) // 1 year
      }
    }));
    
    return transaction;
  }

  async getTransaction(transactionId) {
    const result = await this.docClient.send(new GetCommand({
      TableName: this.tableName,
      Key: { transactionId }
    }));
    
    return result.Item;
  }

  async getTransactionsByAccount(accountId, limit = 50) {
    const result = await this.docClient.send(new QueryCommand({
      TableName: this.tableName,
      IndexName: 'AccountIdIndex',
      KeyConditionExpression: 'fromAccount = :accountId',
      ExpressionAttributeValues: {
        ':accountId': accountId
      },
      Limit: limit,
      ScanIndexForward: false // Descending order
    }));
    
    return result.Items;
  }

  async updateTransactionStatus(transactionId, status) {
    await this.docClient.send(new UpdateCommand({
      TableName: this.tableName,
      Key: { transactionId },
      UpdateExpression: 'SET #status = :status, updatedAt = :updatedAt',
      ExpressionAttributeNames: {
        '#status': 'status'
      },
      ExpressionAttributeValues: {
        ':status': status,
        ':updatedAt': new Date().toISOString()
      }
    }));
  }
}

module.exports = TransactionRepository;
```

### 🎓 Interview Discussion Points - Q17

**Q1: Why use database per service instead of shared database?**

**A**:
- **Loose coupling**: Services can evolve independently
- **Technology choice**: Use best database for each service (SQL/NoSQL)
- **Scalability**: Scale databases independently
- **Failure isolation**: Database failure doesn't affect all services

**Q2: How do you handle cross-service queries?**

**A**:
- **API composition**: Call multiple services and aggregate
- **CQRS**: Maintain read models/materialized views
- **Data streaming**: Use CDC (Change Data Capture) to sync
- **Event sourcing**: Replay events to build views

**Q3: What are the challenges of database per service?**

**A**:
- **Data consistency**: No ACID transactions across services
- **Complex queries**: Need to join data from multiple services
- **Data duplication**: Same data in multiple databases
- **Increased complexity**: More databases to manage

---

## Question 18: CQRS and Event Sourcing

### 📋 Question Statement

Implement CQRS (Command Query Responsibility Segregation) and Event Sourcing for Emirates NBD account management. Include command/query separation, event store, read model projections, and eventual consistency handling.

---

### 🏗️ CQRS + Event Sourcing Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│              EMIRATES NBD CQRS + EVENT SOURCING PATTERN                     │
└────────────────────────────────────────────────────────────────────────────┘

    WRITE SIDE (Commands)              READ SIDE (Queries)
    ─────────────────────              ───────────────────
    
    ┌──────────────────┐               ┌──────────────────┐
    │   Command API    │               │    Query API     │
    │  POST /accounts  │               │  GET /accounts   │
    └────────┬─────────┘               └────────┬─────────┘
             │                                   │
             ▼                                   ▼
    ┌──────────────────┐               ┌──────────────────┐
    │  Command Handler │               │  Query Handler   │
    │  • Validate      │               │  • Read Model    │
    │  • Business Logic│               │  • Fast Queries  │
    └────────┬─────────┘               └────────┬─────────┘
             │                                   │
             ▼                                   ▼
    ┌──────────────────┐               ┌──────────────────┐
    │   Event Store    │               │  Read Database   │
    │   (DynamoDB)     │               │  (ElastiCache)   │
    │                  │               │                  │
    │  Events:         │               │  Views:          │
    │  • Created       │───────────────>│  • Account View  │
    │  • Deposited     │   Projection  │  • Customer View │
    │  • Withdrawn     │               │  • Balance View  │
    └──────────────────┘               └──────────────────┘
             │
             └──> EventBridge ──> Notifications, Analytics, Audit
```

---

### 🔧 CQRS Infrastructure

```typescript
// infrastructure/cdk/cqrs-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as elasticache from 'aws-cdk-lib/aws-elasticache';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';

export class CQRSStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Event Store (DynamoDB)
    const eventStore = new dynamodb.Table(this, 'EventStore', {
      tableName: 'banking-event-store',
      partitionKey: { name: 'aggregateId', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'version', type: dynamodb.AttributeType.NUMBER },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      stream: dynamodb.StreamViewType.NEW_IMAGE
    });

    // Read Model (DynamoDB)
    const accountReadModel = new dynamodb.Table(this, 'AccountReadModel', {
      tableName: 'account-read-model',
      partitionKey: { name: 'accountId', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST
    });

    // Command Handler
    const commandHandler = new lambda.Function(this, 'CommandHandler', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambdas/command-handler'),
      environment: {
        EVENT_STORE_TABLE: eventStore.tableName
      }
    });

    eventStore.grantReadWriteData(commandHandler);

    // Projection Handler
    const projectionHandler = new lambda.Function(this, 'ProjectionHandler', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambdas/projection-handler'),
      environment: {
        READ_MODEL_TABLE: accountReadModel.tableName
      }
    });

    projectionHandler.addEventSource(new lambdaEventSources.DynamoEventSource(eventStore, {
      startingPosition: lambda.StartingPosition.LATEST,
      batchSize: 10
    }));

    accountReadModel.grantReadWriteData(projectionHandler);

    // Query Handler
    const queryHandler = new lambda.Function(this, 'QueryHandler', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambdas/query-handler'),
      environment: {
        READ_MODEL_TABLE: accountReadModel.tableName
      }
    });

    accountReadModel.grantReadData(queryHandler);

    // API Gateway
    const api = new apigateway.RestApi(this, 'CQRSAPI', {
      restApiName: 'Banking CQRS API'
    });

    const commands = api.root.addResource('commands');
    commands.addMethod('POST', new apigateway.LambdaIntegration(commandHandler));

    const queries = api.root.addResource('queries');
    queries.addMethod('GET', new apigateway.LambdaIntegration(queryHandler));
  }
}
```

### 📦 Event Store Implementation

```javascript
// services/cqrs-service/src/event-store.js
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, PutCommand, QueryCommand } = require('@aws-sdk/lib-dynamodb');
const { EventBridgeClient, PutEventsCommand } = require('@aws-sdk/client-eventbridge');

class EventStore {
  constructor() {
    const client = new DynamoDBClient({});
    this.docClient = DynamoDBDocumentClient.from(client);
    this.eventBridge = new EventBridgeClient({});
    this.tableName = process.env.EVENT_STORE_TABLE;
  }

  async appendEvents(aggregateId, expectedVersion, events) {
    // Optimistic concurrency control
    const existingEvents = await this.getEvents(aggregateId);
    const currentVersion = existingEvents.length;
    
    if (currentVersion !== expectedVersion) {
      throw new Error(`Concurrency conflict: expected ${expectedVersion}, got ${currentVersion}`);
    }

    // Append new events
    for (let i = 0; i < events.length; i++) {
      const event = {
        aggregateId,
        version: currentVersion + i + 1,
        eventType: events[i].type,
        eventData: events[i].data,
        metadata: {
          timestamp: new Date().toISOString(),
          userId: events[i].userId || 'system',
          correlationId: events[i].correlationId
        }
      };

      await this.docClient.send(new PutCommand({
        TableName: this.tableName,
        Item: event,
        ConditionExpression: 'attribute_not_exists(aggregateId) AND attribute_not_exists(version)'
      }));

      // Publish to EventBridge for projections
      await this.publishEvent(event);
    }

    return currentVersion + events.length;
  }

  async getEvents(aggregateId) {
    const result = await this.docClient.send(new QueryCommand({
      TableName: this.tableName,
      KeyConditionExpression: 'aggregateId = :aggregateId',
      ExpressionAttributeValues: {
        ':aggregateId': aggregateId
      },
      ScanIndexForward: true // Ascending order
    }));

    return result.Items || [];
  }

  async publishEvent(event) {
    await this.eventBridge.send(new PutEventsCommand({
      Entries: [{
        Source: 'banking.eventstore',
        DetailType: event.eventType,
        Detail: JSON.stringify({
          aggregateId: event.aggregateId,
          version: event.version,
          data: event.eventData,
          metadata: event.metadata
        }),
        EventBusName: 'banking-event-bus'
      }]
    }));
  }
}

module.exports = EventStore;
```

### 🏦 Account Aggregate (Domain Model)

```javascript
// services/cqrs-service/src/domain/account-aggregate.js

class AccountAggregate {
  constructor(accountId) {
    this.accountId = accountId;
    this.customerId = null;
    this.accountType = null;
    this.balance = 0;
    this.status = 'PENDING';
    this.version = 0;
    this.uncommittedEvents = [];
  }

  // Command: Create Account
  static create(accountId, customerId, accountType, initialBalance) {
    const account = new AccountAggregate(accountId);
    account.applyEvent({
      type: 'AccountCreated',
      data: {
        accountId,
        customerId,
        accountType,
        initialBalance,
        createdAt: new Date().toISOString()
      }
    });
    return account;
  }

  // Command: Deposit Money
  deposit(amount, reference) {
    if (this.status !== 'ACTIVE') {
      throw new Error('Account is not active');
    }
    if (amount <= 0) {
      throw new Error('Deposit amount must be positive');
    }

    this.applyEvent({
      type: 'MoneyDeposited',
      data: {
        accountId: this.accountId,
        amount,
        reference,
        previousBalance: this.balance,
        newBalance: this.balance + amount,
        depositedAt: new Date().toISOString()
      }
    });
  }

  // Command: Withdraw Money
  withdraw(amount, reference) {
    if (this.status !== 'ACTIVE') {
      throw new Error('Account is not active');
    }
    if (amount <= 0) {
      throw new Error('Withdrawal amount must be positive');
    }
    if (this.balance < amount) {
      throw new Error('Insufficient funds');
    }

    this.applyEvent({
      type: 'MoneyWithdrawn',
      data: {
        accountId: this.accountId,
        amount,
        reference,
        previousBalance: this.balance,
        newBalance: this.balance - amount,
        withdrawnAt: new Date().toISOString()
      }
    });
  }

  // Command: Close Account
  close(reason) {
    if (this.status === 'CLOSED') {
      throw new Error('Account is already closed');
    }
    if (this.balance > 0) {
      throw new Error('Cannot close account with positive balance');
    }

    this.applyEvent({
      type: 'AccountClosed',
      data: {
        accountId: this.accountId,
        reason,
        closedAt: new Date().toISOString()
      }
    });
  }

  // Apply event to aggregate state
  applyEvent(event) {
    // Apply to current state
    this.apply(event);
    
    // Track uncommitted events
    this.uncommittedEvents.push(event);
    this.version++;
  }

  // Event handlers (state mutations)
  apply(event) {
    switch (event.type) {
      case 'AccountCreated':
        this.customerId = event.data.customerId;
        this.accountType = event.data.accountType;
        this.balance = event.data.initialBalance;
        this.status = 'ACTIVE';
        break;

      case 'MoneyDeposited':
        this.balance = event.data.newBalance;
        break;

      case 'MoneyWithdrawn':
        this.balance = event.data.newBalance;
        break;

      case 'AccountClosed':
        this.status = 'CLOSED';
        break;
    }
  }

  // Reconstruct aggregate from events
  static fromEvents(accountId, events) {
    const account = new AccountAggregate(accountId);
    events.forEach(event => {
      account.apply({ type: event.eventType, data: event.eventData });
      account.version = event.version;
    });
    return account;
  }

  getUncommittedEvents() {
    return this.uncommittedEvents;
  }

  clearUncommittedEvents() {
    this.uncommittedEvents = [];
  }
}

module.exports = AccountAggregate;
```

### 📝 Command Handler (Lambda)

```javascript
// lambdas/command-handler/index.js
const EventStore = require('./event-store');
const AccountAggregate = require('./domain/account-aggregate');

const eventStore = new EventStore();

exports.handler = async (event) => {
  try {
    const body = JSON.parse(event.body);
    const { command, accountId, data } = body;

    let account;
    let result;

    switch (command) {
      case 'CreateAccount':
        account = AccountAggregate.create(
          accountId,
          data.customerId,
          data.accountType,
          data.initialBalance || 0
        );
        break;

      case 'DepositMoney':
        // Load existing aggregate from events
        const depositEvents = await eventStore.getEvents(accountId);
        account = AccountAggregate.fromEvents(accountId, depositEvents);
        account.deposit(data.amount, data.reference);
        break;

      case 'WithdrawMoney':
        const withdrawEvents = await eventStore.getEvents(accountId);
        account = AccountAggregate.fromEvents(accountId, withdrawEvents);
        account.withdraw(data.amount, data.reference);
        break;

      case 'CloseAccount':
        const closeEvents = await eventStore.getEvents(accountId);
        account = AccountAggregate.fromEvents(accountId, closeEvents);
        account.close(data.reason);
        break;

      default:
        throw new Error(`Unknown command: ${command}`);
    }

    // Save uncommitted events
    const uncommittedEvents = account.getUncommittedEvents();
    const newVersion = await eventStore.appendEvents(
      accountId,
      account.version - uncommittedEvents.length,
      uncommittedEvents.map(e => ({
        type: e.type,
        data: e.data,
        userId: data.userId,
        correlationId: data.correlationId
      }))
    );

    return {
      statusCode: 202,
      body: JSON.stringify({
        message: 'Command accepted',
        accountId,
        version: newVersion
      })
    };

  } catch (error) {
    console.error('Command handler error:', error);
    return {
      statusCode: error.message.includes('Concurrency') ? 409 : 400,
      body: JSON.stringify({ error: error.message })
    };
  }
};
```

### 📊 Projection Handler (Read Model)

```javascript
// lambdas/projection-handler/index.js
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, PutCommand, UpdateCommand } = require('@aws-sdk/lib-dynamodb');

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);
const readModelTable = process.env.READ_MODEL_TABLE;

exports.handler = async (event) => {
  for (const record of event.Records) {
    if (record.eventName === 'INSERT') {
      const newEvent = record.dynamodb.NewImage;
      await projectEvent(newEvent);
    }
  }
};

async function projectEvent(eventRecord) {
  const aggregateId = eventRecord.aggregateId.S;
  const eventType = eventRecord.eventType.S;
  const eventData = JSON.parse(eventRecord.eventData.S);

  console.log(`Projecting event: ${eventType} for ${aggregateId}`);

  switch (eventType) {
    case 'AccountCreated':
      await docClient.send(new PutCommand({
        TableName: readModelTable,
        Item: {
          accountId: eventData.accountId,
          customerId: eventData.customerId,
          accountType: eventData.accountType,
          balance: eventData.initialBalance,
          status: 'ACTIVE',
          createdAt: eventData.createdAt,
          updatedAt: eventData.createdAt,
          transactionCount: 0
        }
      }));
      break;

    case 'MoneyDeposited':
      await docClient.send(new UpdateCommand({
        TableName: readModelTable,
        Key: { accountId: eventData.accountId },
        UpdateExpression: 'SET balance = :balance, updatedAt = :updatedAt, transactionCount = transactionCount + :inc',
        ExpressionAttributeValues: {
          ':balance': eventData.newBalance,
          ':updatedAt': eventData.depositedAt,
          ':inc': 1
        }
      }));
      break;

    case 'MoneyWithdrawn':
      await docClient.send(new UpdateCommand({
        TableName: readModelTable,
        Key: { accountId: eventData.accountId },
        UpdateExpression: 'SET balance = :balance, updatedAt = :updatedAt, transactionCount = transactionCount + :inc',
        ExpressionAttributeValues: {
          ':balance': eventData.newBalance,
          ':updatedAt': eventData.withdrawnAt,
          ':inc': 1
        }
      }));
      break;

    case 'AccountClosed':
      await docClient.send(new UpdateCommand({
        TableName: readModelTable,
        Key: { accountId: eventData.accountId },
        UpdateExpression: 'SET #status = :status, updatedAt = :updatedAt',
        ExpressionAttributeNames: {
          '#status': 'status'
        },
        ExpressionAttributeValues: {
          ':status': 'CLOSED',
          ':updatedAt': eventData.closedAt
        }
      }));
      break;
  }
}
```

### 🔍 Query Handler (Lambda)

```javascript
// lambdas/query-handler/index.js
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, GetCommand, QueryCommand } = require('@aws-sdk/lib-dynamodb');

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);
const readModelTable = process.env.READ_MODEL_TABLE;

exports.handler = async (event) => {
  try {
    const { queryStringParameters } = event;
    
    if (queryStringParameters && queryStringParameters.accountId) {
      // Get single account
      const result = await docClient.send(new GetCommand({
        TableName: readModelTable,
        Key: { accountId: queryStringParameters.accountId }
      }));

      return {
        statusCode: 200,
        body: JSON.stringify(result.Item || {})
      };
    }

    // Get all accounts (in production, add pagination)
    const result = await docClient.send(new ScanCommand({
      TableName: readModelTable,
      Limit: 100
    }));

    return {
      statusCode: 200,
      body: JSON.stringify({
        accounts: result.Items,
        count: result.Count
      })
    };

  } catch (error) {
    console.error('Query handler error:', error);
    return {
      statusCode: 500,
      body: JSON.stringify({ error: error.message })
    };
  }
};
```

### 🧪 Testing CQRS System

```javascript
// test/cqrs-test.js
const axios = require('axios');

const API_URL = 'https://your-api-gateway.execute-api.us-east-1.amazonaws.com/prod';

async function testCQRS() {
  const accountId = `ACC-${Date.now()}`;
  const correlationId = `CORR-${Date.now()}`;

  // Command 1: Create Account
  console.log('Creating account...');
  const createResponse = await axios.post(`${API_URL}/commands`, {
    command: 'CreateAccount',
    accountId,
    data: {
      customerId: 'CUST-001',
      accountType: 'SAVINGS',
      initialBalance: 1000,
      userId: 'test-user',
      correlationId
    }
  });
  console.log('Create response:', createResponse.data);

  // Wait for projection
  await new Promise(resolve => setTimeout(resolve, 2000));

  // Query 1: Get Account
  console.log('\nQuerying account...');
  const queryResponse1 = await axios.get(`${API_URL}/queries?accountId=${accountId}`);
  console.log('Query response:', queryResponse1.data);

  // Command 2: Deposit Money
  console.log('\nDepositing money...');
  const depositResponse = await axios.post(`${API_URL}/commands`, {
    command: 'DepositMoney',
    accountId,
    data: {
      amount: 500,
      reference: 'DEP-001',
      userId: 'test-user',
      correlationId
    }
  });
  console.log('Deposit response:', depositResponse.data);

  // Wait for projection
  await new Promise(resolve => setTimeout(resolve, 2000));

  // Query 2: Get Updated Account
  console.log('\nQuerying updated account...');
  const queryResponse2 = await axios.get(`${API_URL}/queries?accountId=${accountId}`);
  console.log('Query response:', queryResponse2.data);
  console.log('Balance should be 1500:', queryResponse2.data.balance);

  // Command 3: Withdraw Money
  console.log('\nWithdrawing money...');
  const withdrawResponse = await axios.post(`${API_URL}/commands`, {
    command: 'WithdrawMoney',
    accountId,
    data: {
      amount: 300,
      reference: 'WITH-001',
      userId: 'test-user',
      correlationId
    }
  });
  console.log('Withdraw response:', withdrawResponse.data);

  // Wait for projection
  await new Promise(resolve => setTimeout(resolve, 2000));

  // Query 3: Get Final Account
  console.log('\nQuerying final account...');
  const queryResponse3 = await axios.get(`${API_URL}/queries?accountId=${accountId}`);
  console.log('Query response:', queryResponse3.data);
  console.log('Balance should be 1200:', queryResponse3.data.balance);

  // Test concurrency conflict
  console.log('\nTesting concurrency conflict...');
  try {
    await axios.post(`${API_URL}/commands`, {
      command: 'DepositMoney',
      accountId,
      data: {
        amount: 100,
        reference: 'DEP-002',
        userId: 'test-user',
        correlationId
      }
    });
  } catch (error) {
    if (error.response && error.response.status === 409) {
      console.log('Concurrency conflict detected as expected');
    }
  }
}

testCQRS().catch(console.error);
```

### 📈 Event Replay & Snapshotting

```javascript
// services/cqrs-service/src/snapshot-manager.js
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, PutCommand, GetCommand } = require('@aws-sdk/lib-dynamodb');

class SnapshotManager {
  constructor() {
    const client = new DynamoDBClient({});
    this.docClient = DynamoDBDocumentClient.from(client);
    this.snapshotTable = 'account-snapshots';
    this.snapshotInterval = 100; // Create snapshot every 100 events
  }

  async saveSnapshot(aggregateId, aggregate, version) {
    await this.docClient.send(new PutCommand({
      TableName: this.snapshotTable,
      Item: {
        aggregateId,
        version,
        state: {
          accountId: aggregate.accountId,
          customerId: aggregate.customerId,
          accountType: aggregate.accountType,
          balance: aggregate.balance,
          status: aggregate.status
        },
        createdAt: new Date().toISOString()
      }
    }));
  }

  async getLatestSnapshot(aggregateId) {
    const result = await this.docClient.send(new GetCommand({
      TableName: this.snapshotTable,
      Key: { aggregateId }
    }));
    return result.Item;
  }

  async shouldCreateSnapshot(version) {
    return version % this.snapshotInterval === 0;
  }

  // Rebuild aggregate from snapshot + subsequent events
  async rebuildAggregate(aggregateId, eventStore) {
    const snapshot = await this.getLatestSnapshot(aggregateId);
    
    if (!snapshot) {
      // No snapshot, load all events
      const events = await eventStore.getEvents(aggregateId);
      return AccountAggregate.fromEvents(aggregateId, events);
    }

    // Start from snapshot
    const aggregate = new AccountAggregate(aggregateId);
    aggregate.accountId = snapshot.state.accountId;
    aggregate.customerId = snapshot.state.customerId;
    aggregate.accountType = snapshot.state.accountType;
    aggregate.balance = snapshot.state.balance;
    aggregate.status = snapshot.state.status;
    aggregate.version = snapshot.version;

    // Load events after snapshot
    const events = await eventStore.getEvents(aggregateId);
    const eventsAfterSnapshot = events.filter(e => e.version > snapshot.version);
    
    eventsAfterSnapshot.forEach(event => {
      aggregate.apply({ type: event.eventType, data: event.eventData });
      aggregate.version = event.version;
    });

    return aggregate;
  }
}

module.exports = SnapshotManager;
```

### 🎓 Interview Discussion Points - Q18

**Q1: What are the benefits of CQRS?**

**A**:
- **Scalability**: Scale read and write sides independently
- **Performance**: Optimize read models for specific queries
- **Flexibility**: Different databases for reads/writes (SQL/NoSQL)
- **Simplicity**: Queries don't need complex joins

**Q2: What are the challenges of Event Sourcing?**

**A**:
- **Complexity**: More complex than CRUD
- **Event versioning**: Schema evolution is difficult
- **Rebuild time**: Replaying many events takes time
- **Storage**: Events grow unbounded (use snapshots)

**Q3: How do you handle eventual consistency?**

**A**:
- **UI feedback**: Show "processing" state
- **Polling**: Poll read model until updated
- **WebSockets**: Push updates when projection completes
- **Versioning**: Include version numbers in responses

**Q4: When should you NOT use CQRS + Event Sourcing?**

**A**:
- **Simple CRUD apps**: Overhead not justified
- **No audit requirements**: Don't need full history
- **Small team**: Requires experienced developers
- **Immediate consistency required**: Can't tolerate eventual consistency

**Q5: How do you handle event versioning?**

**A**:
- **Upcasting**: Convert old events to new format when loading
- **Multiple versions**: Support old and new event formats
- **Weak schema**: Use flexible JSON structure
- **Event migration**: Batch process to upgrade events

---

### 🚀 Production Considerations

**Monitoring**:
```javascript
// Add CloudWatch metrics
const { CloudWatchClient, PutMetricDataCommand } = require('@aws-sdk/client-cloudwatch');

async function recordMetric(metricName, value) {
  const cloudwatch = new CloudWatchClient({});
  await cloudwatch.send(new PutMetricDataCommand({
    Namespace: 'BankingCQRS',
    MetricData: [{
      MetricName: metricName,
      Value: value,
      Unit: 'Count',
      Timestamp: new Date()
    }]
  }));
}

// In event store
await recordMetric('EventsAppended', events.length);

// In projection handler
await recordMetric('ProjectionLatency', latencyMs);
```

**Error Handling**:
- Dead letter queues for failed projections
- Idempotent projection handlers
- Retry with exponential backoff
- Alert on projection lag

**Performance Optimization**:
- Cache read models in ElastiCache
- Use DynamoDB DAX for hot data
- Parallel projection handlers
- Snapshot aggregates with many events

---

**End of File 9 - Database Patterns**

