# Microservices Fundamentals - Emirates NBD Banking Platform

## Question 1: Design a Complete Microservices Architecture for Digital Banking

### 📋 Question Statement

Design a comprehensive microservices architecture for Emirates NBD's digital banking platform that handles:
- Account management (CRUD operations)
- Transaction processing (deposits, withdrawals, transfers)
- Payment processing
- Notifications (email, SMS, push)
- User authentication and authorization
- Fraud detection
- Analytics and reporting

Apply Domain-Driven Design (DDD) principles, define bounded contexts, and explain the service decomposition strategy.

---

### 🏗️ Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                                 │
│    Mobile App (iOS/Android) │ Web App (React) │ Admin Portal       │
└────────────────────┬────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      API GATEWAY LAYER                               │
│  Route 53 → CloudFront → WAF → API Gateway (REST + WebSocket)      │
│  - Authentication (Cognito Authorizer)                              │
│  - Rate Limiting & Throttling                                       │
│  - Request Validation                                               │
└────────────────────┬────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    MICROSERVICES LAYER                               │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐ │
│  │ Account Service  │  │Transaction Service│  │ Payment Service  │ │
│  │ (ECS + Node.js)  │  │ (Lambda + Node.js)│  │ (EKS + NestJS)  │ │
│  │                  │  │                  │  │                  │ │
│  │ - Create Account │  │ - Deposit        │  │ - Process Payment│ │
│  │ - Get Account    │  │ - Withdrawal     │  │ - Validate Card  │ │
│  │ - Update Profile │  │ - Transfer       │  │ - Settlement     │ │
│  │ - Close Account  │  │ - History        │  │ - Refunds        │ │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘ │
│           │                     │                     │            │
│           ▼                     ▼                     ▼            │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐ │
│  │ RDS PostgreSQL   │  │   DynamoDB       │  │ Aurora Serverless│ │
│  │ (ACID required)  │  │ (High throughput)│  │ (PCI compliance) │ │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘ │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐ │
│  │  User Service    │  │Notification Svc  │  │ Fraud Detection  │ │
│  │ (Lambda + Node)  │  │ (Lambda + Node)  │  │ (Lambda + ML)    │ │
│  │                  │  │                  │  │                  │ │
│  │ - Register       │  │ - Email (SES)    │  │ - Risk Scoring   │ │
│  │ - Login          │  │ - SMS (SNS)      │  │ - Pattern Check  │ │
│  │ - Profile        │  │ - Push (FCM)     │  │ - Block Suspect  │ │
│  │ - Preferences    │  │ - WhatsApp       │  │ - ML Model       │ │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘ │
│           │                     │                     │            │
│           ▼                     ▼                     ▼            │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐ │
│  │ Cognito + DDB    │  │  EventBridge     │  │   SageMaker      │ │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘ │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐                        │
│  │ Analytics Svc    │  │  Audit Service   │                        │
│  │ (Kinesis+Lambda) │  │ (Lambda + S3)    │                        │
│  │                  │  │                  │                        │
│  │ - Real-time      │  │ - Audit Logs     │                        │
│  │ - Dashboards     │  │ - Compliance     │                        │
│  │ - Reports        │  │ - Tracking       │                        │
│  └────────┬─────────┘  └────────┬─────────┘                        │
│           │                     │                                   │
│           ▼                     ▼                                   │
│  ┌──────────────────┐  ┌──────────────────┐                        │
│  │ Kinesis + Redshift│  │    S3 + Athena   │                       │
│  └──────────────────┘  └──────────────────┘                        │
└─────────────────────────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   MESSAGING & EVENTS LAYER                           │
│  EventBridge │ SQS (FIFO) │ SNS │ Kinesis Data Streams            │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 🎯 Advanced Topics Covered

1. **Domain-Driven Design (DDD)**
   - Bounded contexts
   - Ubiquitous language
   - Aggregates and entities
   - Domain events

2. **Service Decomposition**
   - Decomposition by business capability
   - Decomposition by subdomain
   - Database per service pattern

3. **Microservices Patterns**
   - API Gateway pattern
   - Database per service
   - Event-driven architecture
   - Saga pattern (for distributed transactions)
   - CQRS (Command Query Responsibility Segregation)

4. **Conway's Law**
   - Team organization alignment
   - Service ownership

5. **Communication Patterns**
   - Synchronous (REST, gRPC)
   - Asynchronous (Event-driven)

---

### 💻 Complete Implementation

#### 1. Domain-Driven Design - Bounded Contexts

```javascript
/**
 * BOUNDED CONTEXTS - Emirates NBD Digital Banking
 * 
 * Each bounded context represents a distinct business capability
 * with its own domain model, language, and database.
 */

// Context 1: Account Management Context
const AccountContext = {
  entities: ['Account', 'Customer', 'AccountHolder'],
  valueObjects: ['AccountNumber', 'Balance', 'AccountType'],
  aggregates: ['Account'],
  domainEvents: [
    'AccountCreated',
    'AccountClosed',
    'ProfileUpdated'
  ],
  ubiquitousLanguage: {
    'Account': 'A customer banking account with balance and transactions',
    'AccountHolder': 'Person or entity who owns the account',
    'Balance': 'Current amount of money in the account'
  }
};

// Context 2: Transaction Context
const TransactionContext = {
  entities: ['Transaction', 'TransactionLog'],
  valueObjects: ['Amount', 'Currency', 'TransactionType'],
  aggregates: ['Transaction'],
  domainEvents: [
    'MoneyDeposited',
    'MoneyWithdrawn',
    'TransferCompleted',
    'TransactionFailed'
  ],
  ubiquitousLanguage: {
    'Transaction': 'Movement of money into or out of an account',
    'Deposit': 'Adding money to an account',
    'Withdrawal': 'Removing money from an account',
    'Transfer': 'Moving money between accounts'
  }
};

// Context 3: Payment Context
const PaymentContext = {
  entities: ['Payment', 'PaymentMethod', 'Merchant'],
  valueObjects: ['PaymentAmount', 'PaymentStatus'],
  aggregates: ['Payment'],
  domainEvents: [
    'PaymentInitiated',
    'PaymentProcessed',
    'PaymentFailed',
    'PaymentRefunded'
  ],
  ubiquitousLanguage: {
    'Payment': 'Transaction to a merchant or third party',
    'Settlement': 'Final confirmation of payment',
    'Refund': 'Reversing a payment'
  }
};
```

#### 2. Account Service Implementation (ECS + Express.js + PostgreSQL)

```javascript
// account-service/src/server.js
const express = require('express');
const { Pool } = require('pg');
const winston = require('winston');
const { v4: uuidv4 } = require('uuid');

// Logger configuration
const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'account-service.log' })
  ]
});

// Database connection pool
const pool = new Pool({
  host: process.env.DB_HOST,
  port: process.env.DB_PORT || 5432,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

const app = express();
app.use(express.json());

// Health check endpoint
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy', service: 'account-service' });
});

// Create Account
app.post('/api/accounts', async (req, res) => {
  const { customerName, email, phone, accountType, initialDeposit } = req.body;
  const accountNumber = `ACC-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  const accountId = uuidv4();
  
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // Insert account
    const accountQuery = `
      INSERT INTO accounts (
        id, account_number, customer_name, email, phone, 
        account_type, balance, status, created_at, updated_at
      ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, NOW(), NOW())
      RETURNING *
    `;
    
    const accountResult = await client.query(accountQuery, [
      accountId,
      accountNumber,
      customerName,
      email,
      phone,
      accountType,
      initialDeposit || 0,
      'ACTIVE'
    ]);
    
    // Publish AccountCreated event
    await publishEvent({
      eventType: 'AccountCreated',
      accountId: accountId,
      accountNumber: accountNumber,
      customerName: customerName,
      email: email,
      balance: initialDeposit || 0,
      timestamp: new Date().toISOString()
    });
    
    await client.query('COMMIT');
    
    logger.info('Account created successfully', {
      accountId: accountId,
      accountNumber: accountNumber
    });
    
    res.status(201).json({
      success: true,
      data: accountResult.rows[0]
    });
    
  } catch (error) {
    await client.query('ROLLBACK');
    logger.error('Error creating account', { error: error.message });
    res.status(500).json({
      success: false,
      error: 'Failed to create account'
    });
  } finally {
    client.release();
  }
});

// Get Account by ID
app.get('/api/accounts/:accountId', async (req, res) => {
  const { accountId } = req.params;
  
  try {
    const query = 'SELECT * FROM accounts WHERE id = $1';
    const result = await pool.query(query, [accountId]);
    
    if (result.rows.length === 0) {
      return res.status(404).json({
        success: false,
        error: 'Account not found'
      });
    }
    
    res.status(200).json({
      success: true,
      data: result.rows[0]
    });
    
  } catch (error) {
    logger.error('Error fetching account', { error: error.message });
    res.status(500).json({
      success: false,
      error: 'Failed to fetch account'
    });
  }
});

// Update Account Profile
app.put('/api/accounts/:accountId', async (req, res) => {
  const { accountId } = req.params;
  const { email, phone, address } = req.body;
  
  try {
    const query = `
      UPDATE accounts 
      SET email = COALESCE($1, email),
          phone = COALESCE($2, phone),
          address = COALESCE($3, address),
          updated_at = NOW()
      WHERE id = $4
      RETURNING *
    `;
    
    const result = await pool.query(query, [email, phone, address, accountId]);
    
    if (result.rows.length === 0) {
      return res.status(404).json({
        success: false,
        error: 'Account not found'
      });
    }
    
    // Publish ProfileUpdated event
    await publishEvent({
      eventType: 'ProfileUpdated',
      accountId: accountId,
      changes: { email, phone, address },
      timestamp: new Date().toISOString()
    });
    
    res.status(200).json({
      success: true,
      data: result.rows[0]
    });
    
  } catch (error) {
    logger.error('Error updating account', { error: error.message });
    res.status(500).json({
      success: false,
      error: 'Failed to update account'
    });
  }
});

// Close Account
app.delete('/api/accounts/:accountId', async (req, res) => {
  const { accountId } = req.params;
  
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // Check balance
    const balanceCheck = await client.query(
      'SELECT balance FROM accounts WHERE id = $1',
      [accountId]
    );
    
    if (balanceCheck.rows.length === 0) {
      await client.query('ROLLBACK');
      return res.status(404).json({
        success: false,
        error: 'Account not found'
      });
    }
    
    if (balanceCheck.rows[0].balance > 0) {
      await client.query('ROLLBACK');
      return res.status(400).json({
        success: false,
        error: 'Cannot close account with positive balance'
      });
    }
    
    // Update status to CLOSED
    const query = `
      UPDATE accounts 
      SET status = 'CLOSED', 
          closed_at = NOW(),
          updated_at = NOW()
      WHERE id = $1
      RETURNING *
    `;
    
    const result = await client.query(query, [accountId]);
    
    // Publish AccountClosed event
    await publishEvent({
      eventType: 'AccountClosed',
      accountId: accountId,
      accountNumber: result.rows[0].account_number,
      timestamp: new Date().toISOString()
    });
    
    await client.query('COMMIT');
    
    res.status(200).json({
      success: true,
      message: 'Account closed successfully'
    });
    
  } catch (error) {
    await client.query('ROLLBACK');
    logger.error('Error closing account', { error: error.message });
    res.status(500).json({
      success: false,
      error: 'Failed to close account'
    });
  } finally {
    client.release();
  }
});

// Helper function to publish events to EventBridge
async function publishEvent(event) {
  const AWS = require('aws-sdk');
  const eventbridge = new AWS.EventBridge();
  
  const params = {
    Entries: [{
      Source: 'com.emiratesnbd.account',
      DetailType: event.eventType,
      Detail: JSON.stringify(event),
      EventBusName: 'banking-event-bus'
    }]
  };
  
  try {
    await eventbridge.putEvents(params).promise();
    logger.info('Event published', { eventType: event.eventType });
  } catch (error) {
    logger.error('Failed to publish event', { error: error.message });
  }
}

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  logger.info(`Account Service running on port ${PORT}`);
});

module.exports = app;
```

#### 3. Database Schema (PostgreSQL)

```sql
-- account-service/migrations/001_create_accounts_table.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE accounts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    account_number VARCHAR(50) UNIQUE NOT NULL,
    customer_name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    phone VARCHAR(20) NOT NULL,
    address TEXT,
    account_type VARCHAR(50) NOT NULL CHECK (account_type IN ('SAVINGS', 'CURRENT', 'FIXED_DEPOSIT')),
    balance DECIMAL(15, 2) NOT NULL DEFAULT 0.00 CHECK (balance >= 0),
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE', 'INACTIVE', 'CLOSED', 'SUSPENDED')),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    closed_at TIMESTAMP
);

CREATE INDEX idx_accounts_account_number ON accounts(account_number);
CREATE INDEX idx_accounts_email ON accounts(email);
CREATE INDEX idx_accounts_status ON accounts(status);
CREATE INDEX idx_accounts_created_at ON accounts(created_at);

-- Audit log table
CREATE TABLE account_audit_log (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    account_id UUID NOT NULL REFERENCES accounts(id),
    action VARCHAR(50) NOT NULL,
    old_values JSONB,
    new_values JSONB,
    changed_by VARCHAR(255),
    changed_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_account_id ON account_audit_log(account_id);
CREATE INDEX idx_audit_changed_at ON account_audit_log(changed_at);
```

#### 4. Transaction Service Implementation (Lambda + Node.js + DynamoDB)

```javascript
// transaction-service/handlers/transaction.handler.js
const AWS = require('aws-sdk');
const { v4: uuidv4 } = require('uuid');

const dynamodb = new AWS.DynamoDB.DocumentClient();
const eventbridge = new AWS.EventBridge();
const TABLE_NAME = process.env.TRANSACTIONS_TABLE;

/**
 * Process Deposit Transaction
 */
exports.deposit = async (event) => {
  console.log('Deposit request received', JSON.stringify(event));
  
  const { accountId, amount, description } = JSON.parse(event.body);
  
  // Validation
  if (!accountId || !amount || amount <= 0) {
    return {
      statusCode: 400,
      body: JSON.stringify({
        success: false,
        error: 'Invalid deposit parameters'
      })
    };
  }
  
  const transactionId = uuidv4();
  const timestamp = new Date().toISOString();
  
  const transaction = {
    transactionId,
    accountId,
    type: 'DEPOSIT',
    amount,
    description: description || 'Cash deposit',
    status: 'COMPLETED',
    timestamp,
    createdAt: timestamp
  };
  
  try {
    // Save transaction to DynamoDB
    await dynamodb.put({
      TableName: TABLE_NAME,
      Item: transaction
    }).promise();
    
    // Publish MoneyDeposited event
    await eventbridge.putEvents({
      Entries: [{
        Source: 'com.emiratesnbd.transaction',
        DetailType: 'MoneyDeposited',
        Detail: JSON.stringify({
          transactionId,
          accountId,
          amount,
          timestamp
        }),
        EventBusName: 'banking-event-bus'
      }]
    }).promise();
    
    console.log('Deposit completed', { transactionId });
    
    return {
      statusCode: 201,
      body: JSON.stringify({
        success: true,
        data: transaction
      })
    };
    
  } catch (error) {
    console.error('Deposit failed', error);
    return {
      statusCode: 500,
      body: JSON.stringify({
        success: false,
        error: 'Failed to process deposit'
      })
    };
  }
};

/**
 * Process Withdrawal Transaction
 */
exports.withdraw = async (event) => {
  const { accountId, amount, description } = JSON.parse(event.body);
  
  if (!accountId || !amount || amount <= 0) {
    return {
      statusCode: 400,
      body: JSON.stringify({
        success: false,
        error: 'Invalid withdrawal parameters'
      })
    };
  }
  
  const transactionId = uuidv4();
  const timestamp = new Date().toISOString();
  
  try {
    // Check account balance (call Account Service)
    const accountBalance = await checkAccountBalance(accountId);
    
    if (accountBalance < amount) {
      return {
        statusCode: 400,
        body: JSON.stringify({
          success: false,
          error: 'Insufficient funds'
        })
      };
    }
    
    const transaction = {
      transactionId,
      accountId,
      type: 'WITHDRAWAL',
      amount,
      description: description || 'Cash withdrawal',
      status: 'COMPLETED',
      timestamp,
      createdAt: timestamp
    };
    
    await dynamodb.put({
      TableName: TABLE_NAME,
      Item: transaction
    }).promise();
    
    // Publish MoneyWithdrawn event
    await eventbridge.putEvents({
      Entries: [{
        Source: 'com.emiratesnbd.transaction',
        DetailType: 'MoneyWithdrawn',
        Detail: JSON.stringify({
          transactionId,
          accountId,
          amount,
          timestamp
        }),
        EventBusName: 'banking-event-bus'
      }]
    }).promise();
    
    return {
      statusCode: 201,
      body: JSON.stringify({
        success: true,
        data: transaction
      })
    };
    
  } catch (error) {
    console.error('Withdrawal failed', error);
    return {
      statusCode: 500,
      body: JSON.stringify({
        success: false,
        error: 'Failed to process withdrawal'
      })
    };
  }
};

/**
 * Process Transfer Transaction (Saga Pattern)
 */
exports.transfer = async (event) => {
  const { fromAccountId, toAccountId, amount, description } = JSON.parse(event.body);
  
  if (!fromAccountId || !toAccountId || !amount || amount <= 0) {
    return {
      statusCode: 400,
      body: JSON.stringify({
        success: false,
        error: 'Invalid transfer parameters'
      })
    };
  }
  
  const transferId = uuidv4();
  const timestamp = new Date().toISOString();
  
  try {
    // Start Transfer Saga using Step Functions
    const stepfunctions = new AWS.StepFunctions();
    
    const execution = await stepfunctions.startExecution({
      stateMachineArn: process.env.TRANSFER_SAGA_ARN,
      input: JSON.stringify({
        transferId,
        fromAccountId,
        toAccountId,
        amount,
        description,
        timestamp
      })
    }).promise();
    
    return {
      statusCode: 202,
      body: JSON.stringify({
        success: true,
        message: 'Transfer initiated',
        transferId,
        executionArn: execution.executionArn
      })
    };
    
  } catch (error) {
    console.error('Transfer initiation failed', error);
    return {
      statusCode: 500,
      body: JSON.stringify({
        success: false,
        error: 'Failed to initiate transfer'
      })
    };
  }
};

/**
 * Get Transaction History
 */
exports.getHistory = async (event) => {
  const { accountId } = event.pathParameters;
  const { limit = 50, startKey } = event.queryStringParameters || {};
  
  try {
    const params = {
      TableName: TABLE_NAME,
      IndexName: 'AccountIdTimestampIndex',
      KeyConditionExpression: 'accountId = :accountId',
      ExpressionAttributeValues: {
        ':accountId': accountId
      },
      Limit: parseInt(limit),
      ScanIndexForward: false // Descending order (most recent first)
    };
    
    if (startKey) {
      params.ExclusiveStartKey = JSON.parse(Buffer.from(startKey, 'base64').toString());
    }
    
    const result = await dynamodb.query(params).promise();
    
    return {
      statusCode: 200,
      body: JSON.stringify({
        success: true,
        data: result.Items,
        nextKey: result.LastEvaluatedKey ? 
          Buffer.from(JSON.stringify(result.LastEvaluatedKey)).toString('base64') : null
      })
    };
    
  } catch (error) {
    console.error('Failed to fetch transaction history', error);
    return {
      statusCode: 500,
      body: JSON.stringify({
        success: false,
        error: 'Failed to fetch transactions'
      })
    };
  }
};

// Helper function to check account balance
async function checkAccountBalance(accountId) {
  const axios = require('axios');
  const accountServiceUrl = process.env.ACCOUNT_SERVICE_URL;
  
  try {
    const response = await axios.get(`${accountServiceUrl}/api/accounts/${accountId}`);
    return response.data.data.balance;
  } catch (error) {
    console.error('Failed to check balance', error);
    throw new Error('Balance check failed');
  }
}
```

#### 5. DynamoDB Table Definition

```javascript
// infrastructure/dynamodb-tables.js
const transactionsTableDefinition = {
  TableName: 'Transactions',
  KeySchema: [
    { AttributeName: 'transactionId', KeyType: 'HASH' }  // Partition key
  ],
  AttributeDefinitions: [
    { AttributeName: 'transactionId', AttributeType: 'S' },
    { AttributeName: 'accountId', AttributeType: 'S' },
    { AttributeName: 'timestamp', AttributeType: 'S' }
  ],
  GlobalSecondaryIndexes: [
    {
      IndexName: 'AccountIdTimestampIndex',
      KeySchema: [
        { AttributeName: 'accountId', KeyType: 'HASH' },
        { AttributeName: 'timestamp', KeyType: 'RANGE' }
      ],
      Projection: {
        ProjectionType: 'ALL'
      },
      ProvisionedThroughput: {
        ReadCapacityUnits: 5,
        WriteCapacityUnits: 5
      }
    }
  ],
  BillingMode: 'PAY_PER_REQUEST', // On-demand billing
  StreamSpecification: {
    StreamEnabled: true,
    StreamViewType: 'NEW_AND_OLD_IMAGES'
  },
  Tags: [
    { Key: 'Service', Value: 'TransactionService' },
    { Key: 'Environment', Value: 'Production' }
  ]
};
```

#### 6. Docker Configuration for Account Service

```dockerfile
# account-service/Dockerfile
# Multi-stage build for Node.js Express application

# Stage 1: Build
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production && \
    npm cache clean --force

# Copy source code
COPY . .

# Stage 2: Production
FROM node:20-alpine

# Install dumb-init for proper signal handling
RUN apk add --no-cache dumb-init

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Copy from builder
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/src ./src
COPY --from=builder --chown=nodejs:nodejs /app/package.json ./

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

# Use dumb-init to handle signals properly
ENTRYPOINT ["dumb-init", "--"]

# Start application
CMD ["node", "src/server.js"]
```

#### 7. Docker Compose for Local Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:15-alpine
    container_name: banking-postgres
    environment:
      POSTGRES_DB: banking_db
      POSTGRES_USER: banking_user
      POSTGRES_PASSWORD: banking_pass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./account-service/migrations:/docker-entrypoint-initdb.d
    networks:
      - banking-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U banking_user -d banking_db"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Account Service
  account-service:
    build:
      context: ./account-service
      dockerfile: Dockerfile
    container_name: account-service
    environment:
      NODE_ENV: development
      PORT: 3000
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: banking_db
      DB_USER: banking_user
      DB_PASSWORD: banking_pass
      AWS_REGION: us-east-1
      AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
      AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
    ports:
      - "3000:3000"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - banking-network
    healthcheck:
      test: ["CMD", "node", "-e", "require('http').get('http://localhost:3000/health')"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # LocalStack for AWS services (DynamoDB, EventBridge, etc.)
  localstack:
    image: localstack/localstack:latest
    container_name: banking-localstack
    environment:
      SERVICES: dynamodb,events,sqs,sns,s3
      DEBUG: 1
      DATA_DIR: /tmp/localstack/data
      DOCKER_HOST: unix:///var/run/docker.sock
    ports:
      - "4566:4566"
      - "4571:4571"
    volumes:
      - localstack_data:/tmp/localstack
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - banking-network

  # Redis for caching
  redis:
    image: redis:7-alpine
    container_name: banking-redis
    ports:
      - "6379:6379"
    networks:
      - banking-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

networks:
  banking-network:
    driver: bridge

volumes:
  postgres_data:
  localstack_data:
```

#### 8. AWS CDK Infrastructure Code (TypeScript)

```typescript
// infrastructure/lib/banking-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as ecr from 'aws-cdk-lib/aws-ecr';
import * as elbv2 from 'aws-cdk-lib/aws-elasticloadbalancingv2';
import * as rds from 'aws-cdk-lib/aws-rds';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as events from 'aws-cdk-lib/aws-events';
import * as secretsmanager from 'aws-cdk-lib/aws-secretsmanager';
import { Construct } from 'constructs';

export class BankingMicroservicesStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // VPC
    const vpc = new ec2.Vpc(this, 'BankingVPC', {
      maxAzs: 3,
      natGateways: 2,
      subnetConfiguration: [
        {
          cidrMask: 24,
          name: 'public',
          subnetType: ec2.SubnetType.PUBLIC,
        },
        {
          cidrMask: 24,
          name: 'private',
          subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,
        },
        {
          cidrMask: 28,
          name: 'isolated',
          subnetType: ec2.SubnetType.PRIVATE_ISOLATED,
        },
      ],
    });

    // RDS PostgreSQL for Account Service
    const dbSecret = new secretsmanager.Secret(this, 'DBSecret', {
      secretName: 'banking/rds/credentials',
      generateSecretString: {
        secretStringTemplate: JSON.stringify({ username: 'banking_admin' }),
        generateStringKey: 'password',
        excludePunctuation: true,
        includeSpace: false,
      },
    });

    const dbSecurityGroup = new ec2.SecurityGroup(this, 'DBSecurityGroup', {
      vpc,
      description: 'Security group for RDS PostgreSQL',
      allowAllOutbound: true,
    });

    const database = new rds.DatabaseInstance(this, 'AccountDatabase', {
      engine: rds.DatabaseInstanceEngine.postgres({
        version: rds.PostgresEngineVersion.VER_15_3,
      }),
      instanceType: ec2.InstanceType.of(
        ec2.InstanceClass.T3,
        ec2.InstanceSize.MEDIUM
      ),
      vpc,
      vpcSubnets: {
        subnetType: ec2.SubnetType.PRIVATE_ISOLATED,
      },
      securityGroups: [dbSecurityGroup],
      credentials: rds.Credentials.fromSecret(dbSecret),
      databaseName: 'banking_db',
      allocatedStorage: 100,
      maxAllocatedStorage: 500,
      multiAz: true,
      deletionProtection: true,
      backupRetention: cdk.Duration.days(7),
      removalPolicy: cdk.RemovalPolicy.SNAPSHOT,
    });

    // ECS Cluster for Account Service
    const cluster = new ecs.Cluster(this, 'BankingCluster', {
      vpc,
      clusterName: 'banking-services-cluster',
      containerInsights: true,
    });

    // Account Service Task Definition
    const accountTaskDef = new ecs.FargateTaskDefinition(this, 'AccountServiceTask', {
      memoryLimitMiB: 512,
      cpu: 256,
    });

    // ECR Repository
    const accountRepo = new ecr.Repository(this, 'AccountServiceRepo', {
      repositoryName: 'account-service',
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      imageScanOnPush: true,
    });

    const accountContainer = accountTaskDef.addContainer('AccountContainer', {
      image: ecs.ContainerImage.fromEcrRepository(accountRepo, 'latest'),
      logging: ecs.LogDrivers.awsLogs({ streamPrefix: 'account-service' }),
      environment: {
        NODE_ENV: 'production',
        PORT: '3000',
        DB_HOST: database.dbInstanceEndpointAddress,
        DB_PORT: database.dbInstanceEndpointPort,
        DB_NAME: 'banking_db',
      },
      secrets: {
        DB_USER: ecs.Secret.fromSecretsManager(dbSecret, 'username'),
        DB_PASSWORD: ecs.Secret.fromSecretsManager(dbSecret, 'password'),
      },
      healthCheck: {
        command: ['CMD-SHELL', 'curl -f http://localhost:3000/health || exit 1'],
        interval: cdk.Duration.seconds(30),
        timeout: cdk.Duration.seconds(5),
        retries: 3,
        startPeriod: cdk.Duration.seconds(60),
      },
    });

    accountContainer.addPortMappings({
      containerPort: 3000,
      protocol: ecs.Protocol.TCP,
    });

    // ECS Service
    const accountService = new ecs.FargateService(this, 'AccountService', {
      cluster,
      taskDefinition: accountTaskDef,
      desiredCount: 2,
      minHealthyPercent: 100,
      maxHealthyPercent: 200,
      circuitBreaker: { rollback: true },
    });

    // Allow ECS to access RDS
    database.connections.allowFrom(accountService, ec2.Port.tcp(5432));

    // Application Load Balancer
    const alb = new elbv2.ApplicationLoadBalancer(this, 'AccountServiceALB', {
      vpc,
      internetFacing: true,
    });

    const listener = alb.addListener('Listener', {
      port: 80,
      open: true,
    });

    listener.addTargets('AccountServiceTargets', {
      port: 3000,
      targets: [accountService],
      healthCheck: {
        path: '/health',
        interval: cdk.Duration.seconds(30),
        timeout: cdk.Duration.seconds(5),
        healthyThresholdCount: 2,
        unhealthyThresholdCount: 3,
      },
    });

    // DynamoDB Table for Transaction Service
    const transactionsTable = new dynamodb.Table(this, 'TransactionsTable', {
      tableName: 'Transactions',
      partitionKey: {
        name: 'transactionId',
        type: dynamodb.AttributeType.STRING,
      },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      stream: dynamodb.StreamViewType.NEW_AND_OLD_IMAGES,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      pointInTimeRecovery: true,
    });

    // GSI for querying by accountId
    transactionsTable.addGlobalSecondaryIndex({
      indexName: 'AccountIdTimestampIndex',
      partitionKey: {
        name: 'accountId',
        type: dynamodb.AttributeType.STRING,
      },
      sortKey: {
        name: 'timestamp',
        type: dynamodb.AttributeType.STRING,
      },
    });

    // EventBridge Event Bus
    const eventBus = new events.EventBus(this, 'BankingEventBus', {
      eventBusName: 'banking-event-bus',
    });

    // Lambda Function for Transaction Service
    const transactionLambda = new lambda.Function(this, 'TransactionFunction', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'transaction.handler',
      code: lambda.Code.fromAsset('lambda/transaction-service'),
      environment: {
        TRANSACTIONS_TABLE: transactionsTable.tableName,
        EVENT_BUS_NAME: eventBus.eventBusName,
        ACCOUNT_SERVICE_URL: `http://${alb.loadBalancerDnsName}`,
      },
      timeout: cdk.Duration.seconds(30),
      memorySize: 512,
      reservedConcurrentExecutions: 100,
    });

    // Grant permissions
    transactionsTable.grantReadWriteData(transactionLambda);
    eventBus.grantPutEventsTo(transactionLambda);

    // API Gateway for Transaction Service
    const api = new apigateway.RestApi(this, 'BankingAPI', {
      restApiName: 'Banking Microservices API',
      description: 'API for Emirates NBD Banking Services',
      deployOptions: {
        stageName: 'prod',
        loggingLevel: apigateway.MethodLoggingLevel.INFO,
        dataTraceEnabled: true,
        metricsEnabled: true,
      },
    });

    const transactions = api.root.addResource('transactions');
    transactions.addMethod('POST', new apigateway.LambdaIntegration(transactionLambda));

    // Outputs
    new cdk.CfnOutput(this, 'LoadBalancerDNS', {
      value: alb.loadBalancerDnsName,
      description: 'Account Service Load Balancer DNS',
    });

    new cdk.CfnOutput(this, 'APIGatewayURL', {
      value: api.url,
      description: 'API Gateway URL',
    });

    new cdk.CfnOutput(this, 'DatabaseEndpoint', {
      value: database.dbInstanceEndpointAddress,
      description: 'RDS Database Endpoint',
    });
  }
}
```

---

### 🎓 Interview Discussion Points

#### **1. Domain-Driven Design (DDD) Application**

**Q: How did you apply DDD to the banking architecture?**

**A**: We identified distinct bounded contexts:
- **Account Context**: Manages customer accounts and profiles (PostgreSQL for ACID compliance)
- **Transaction Context**: Handles money movements (DynamoDB for high throughput)
- **Payment Context**: Processes payments to merchants (Aurora for PCI-DSS compliance)

Each context has its own:
- Ubiquitous language
- Database (database per service pattern)
- Domain events for communication

#### **2. Service Decomposition Strategy**

**Q: What strategy did you use for service decomposition?**

**A**: We used **Decomposition by Business Capability**:
- Each service represents a distinct business function
- Account Service → Account management
- Transaction Service → Transaction processing
- Payment Service → Payment handling
- Notification Service → Multi-channel notifications
- Fraud Detection Service → Risk assessment

**Benefits**:
- Clear ownership
- Independent deployment
- Technology flexibility (Lambda vs ECS vs EKS)

#### **3. Database Per Service Pattern**

**Q: Why different databases for different services?**

**A**:
- **Account Service (RDS PostgreSQL)**: Needs ACID transactions, complex joins, referential integrity
- **Transaction Service (DynamoDB)**: High throughput, simple key-value lookups, auto-scaling
- **Payment Service (Aurora)**: PCI-DSS compliance, read replicas for reporting

**Trade-offs**:
- ✅ Independence, scalability, technology fit
- ❌ Distributed transactions, data consistency challenges (solved with Saga pattern)

#### **4. Hybrid Architecture (Serverless + Containers)**

**Q: When to use Lambda vs ECS/EKS?**

**A**:

**Use Lambda (Serverless) when**:
- Event-driven workloads (notifications, fraud detection)
- Sporadic traffic patterns
- Fast scaling required
- Want to minimize operational overhead

**Use ECS/Fargate (Containers) when**:
- Persistent database connections needed
- Complex business logic
- Predictable traffic
- Need more control over runtime environment

**Use EKS (Kubernetes) when**:
- Multi-cloud portability needed
- Complex orchestration requirements
- Large-scale deployments
- Advanced networking needs

#### **5. Communication Patterns**

**Q: How do services communicate?**

**A**:
- **Synchronous (REST)**: Account Service ← API calls → Transaction Service
  - Use for immediate responses
  - Circuit breaker pattern for resilience
  
- **Asynchronous (Event-Driven)**: Services → EventBridge → Subscribers
  - Account Created event → triggers welcome email, analytics update
  - Transaction Completed → triggers notification, fraud check
  - Loose coupling, better scalability

---

### 📊 Service Comparison Matrix

| **Service** | **Technology** | **Database** | **Why This Choice** |
|------------|---------------|--------------|---------------------|
| Account Service | ECS + Express.js | RDS PostgreSQL | ACID transactions, complex queries, connection pooling |
| Transaction Service | Lambda + Node.js | DynamoDB | High throughput, auto-scaling, pay-per-request |
| Payment Service | EKS + NestJS | Aurora Serverless | PCI compliance, modular architecture, K8s features |
| User Service | Lambda + Node.js | Cognito + DynamoDB | Authentication, user pools, serverless |
| Notification Service | Lambda + Node.js | EventBridge + SES/SNS | Event-driven, cost-effective |
| Fraud Detection | Lambda + Node.js | SageMaker | ML model integration, real-time scoring |
| Analytics Service | Kinesis + Lambda | Redshift | Real-time streaming, data warehouse |
| Audit Service | Lambda + Node.js | S3 + Athena | Compliance, long-term storage |

---

### 🔒 Security Considerations

1. **Network Isolation**: VPC with public, private, and isolated subnets
2. **Database Security**: RDS in isolated subnet, security groups, encryption at rest
3. **Secret Management**: AWS Secrets Manager for database credentials
4. **API Security**: WAF, API Gateway throttling, Cognito authentication
5. **Container Security**: Non-root user, image scanning, minimal base images
6. **Encryption**: In transit (TLS), at rest (KMS), database encryption

---

### 💰 Cost Optimization

1. **Serverless for Variable Workloads**: Lambda for notifications, fraud detection
2. **Containers for Steady State**: ECS for account management (predictable traffic)
3. **Database Right-Sizing**: RDS for critical data, DynamoDB for high-throughput
4. **Auto-Scaling**: Based on CPU, memory, request count
5. **Reserved Capacity**: For predictable base load

**Estimated Monthly Cost (Medium Scale)**:
- ECS Fargate (2 tasks): ~$50
- RDS PostgreSQL (db.t3.medium): ~$80
- DynamoDB (on-demand): ~$30-100 (based on traffic)
- Lambda (1M requests/month): ~$5
- API Gateway: ~$10
- Total: **~$175-245/month** (excluding data transfer)

---

### ⚡ Performance Optimization

1. **Connection Pooling**: RDS Proxy for Lambda connections
2. **Caching**: ElastiCache Redis for frequent queries
3. **Read Replicas**: Aurora read replicas for reporting
4. **CDN**: CloudFront for static assets
5. **Async Processing**: EventBridge for non-blocking operations
6. **DynamoDB DAX**: For sub-millisecond reads

---

### 🧪 Testing Strategy

```javascript
// account-service/__tests__/account.test.js
const request = require('supertest');
const app = require('../src/server');

describe('Account Service API', () => {
  describe('POST /api/accounts', () => {
    it('should create a new account', async () => {
      const response = await request(app)
        .post('/api/accounts')
        .send({
          customerName: 'Ahmed Al Mansoori',
          email: 'ahmed@example.com',
          phone: '+971501234567',
          accountType: 'SAVINGS',
          initialDeposit: 1000
        })
        .expect(201);

      expect(response.body.success).toBe(true);
      expect(response.body.data).toHaveProperty('accountNumber');
      expect(response.body.data.balance).toBe(1000);
    });

    it('should reject invalid account type', async () => {
      const response = await request(app)
        .post('/api/accounts')
        .send({
          customerName: 'Test User',
          email: 'test@example.com',
          phone: '+971501234567',
          accountType: 'INVALID_TYPE',
          initialDeposit: 1000
        })
        .expect(400);

      expect(response.body.success).toBe(false);
    });
  });
});
```

---

## Question 2: Monolith to Microservices Migration Strategy

### 📋 Question Statement

Emirates NBD currently has a monolithic banking application built with Node.js and a single PostgreSQL database. The application handles all banking operations including accounts, transactions, payments, loans, and customer management.

Design a comprehensive migration strategy to transform this monolith into a microservices architecture. Include:
- Migration patterns (Strangler Fig, Anti-Corruption Layer)
- Phased approach with minimal disruption
- Database decomposition strategy
- Team organization
- Risk mitigation
- Rollback procedures

---

### 🏗️ Current Monolithic Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  MONOLITHIC APPLICATION                      │
│                     (Node.js + Express)                      │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Single Codebase                          │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐     │  │
│  │  │  Account   │  │Transaction │  │  Payment   │     │  │
│  │  │   Module   │  │   Module   │  │   Module   │     │  │
│  │  └────────────┘  └────────────┘  └────────────┘     │  │
│  │                                                       │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐     │  │
│  │  │    Loan    │  │  Customer  │  │   Report   │     │  │
│  │  │   Module   │  │   Module   │  │   Module   │     │  │
│  │  └────────────┘  └────────────┘  └────────────┘     │  │
│  └──────────────────────────────────────────────────────┘  │
│                            │                                │
│                            ▼                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │          Single PostgreSQL Database                   │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐              │  │
│  │  │accounts │  │customers│  │ loans   │              │  │
│  │  └─────────┘  └─────────┘  └─────────┘              │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐              │  │
│  │  │transfers│  │payments │  │ reports │              │  │
│  │  └─────────┘  └─────────┘  └─────────┘              │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

PROBLEMS:
❌ Single deployment unit (any change requires full deployment)
❌ Tight coupling between modules
❌ Database bottleneck
❌ Difficult to scale individual features
❌ Technology lock-in
❌ Team coordination overhead
❌ Long build and test times
```

---

### 🎯 Target Microservices Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  MICROSERVICES ARCHITECTURE                  │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Account  │  │Transaction│  │ Payment  │  │   Loan   │   │
│  │ Service  │  │  Service  │  │ Service  │  │ Service  │   │
│  │ (ECS)    │  │ (Lambda)  │  │  (EKS)   │  │ (ECS)    │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
│       │             │              │              │          │
│       ▼             ▼              ▼              ▼          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │   RDS    │  │ DynamoDB │  │  Aurora  │  │   RDS    │   │
│  │Postgres  │  │          │  │Serverless│  │ MySQL    │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘

BENEFITS:
✅ Independent deployment
✅ Technology flexibility
✅ Team autonomy
✅ Targeted scaling
✅ Fault isolation
✅ Faster development
```

---

### 🚀 Migration Strategy: Strangler Fig Pattern

The Strangler Fig pattern gradually replaces parts of the monolith with microservices.

```
Phase 1: Routing Layer
┌─────────────────────────────────────────────────────────────┐
│                     API Gateway / Router                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  IF (route === '/accounts') THEN new_service         │   │
│  │  ELSE monolith                                        │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────┬───────────────────────┬───────────────────┘
                  │                       │
                  ▼                       ▼
         ┌─────────────────┐    ┌─────────────────┐
         │  New Account    │    │   Monolith      │
         │  Microservice   │    │  (Everything    │
         │   (10% traffic) │    │   else 90%)     │
         └─────────────────┘    └─────────────────┘
```

#### **Phase 1: Setup Routing Infrastructure (Week 1-2)**

```javascript
// strangler-router/src/router.js
const express = require('express');
const httpProxy = require('http-proxy');
const router = express.Router();

// Feature flags for gradual rollout
const featureFlags = {
  accountService: {
    enabled: true,
    percentage: 10, // Route 10% traffic to new service
    newServiceUrl: 'http://account-service.internal',
    monolithUrl: 'http://monolith.internal'
  },
  transactionService: {
    enabled: false,
    percentage: 0,
    newServiceUrl: 'http://transaction-service.internal',
    monolithUrl: 'http://monolith.internal'
  }
};

// Proxy instances
const proxy = httpProxy.createProxyServer({});

// Intelligent routing middleware
router.use('/api/accounts/*', (req, res) => {
  const config = featureFlags.accountService;
  
  if (!config.enabled) {
    // Route to monolith
    return proxy.web(req, res, { target: config.monolithUrl });
  }
  
  // Gradual rollout based on percentage
  const shouldUseNewService = Math.random() * 100 < config.percentage;
  
  if (shouldUseNewService) {
    console.log('Routing to new Account Service');
    proxy.web(req, res, { target: config.newServiceUrl });
  } else {
    console.log('Routing to Monolith');
    proxy.web(req, res, { target: config.monolithUrl });
  }
});

// Default: route everything else to monolith
router.use('*', (req, res) => {
  proxy.web(req, res, { target: 'http://monolith.internal' });
});

module.exports = router;
```

---

#### **Phase 2: Database Decomposition Strategy**

**Challenge**: The monolith has a single database with foreign keys across tables.

**Solution**: Use the **Database View Pattern** + **Change Data Capture (CDC)**

```sql
-- Step 1: Create read-only views for new Account Service
CREATE VIEW account_service_accounts AS
SELECT 
  id,
  account_number,
  customer_id,
  account_type,
  balance,
  status,
  created_at
FROM accounts;

-- Grant access to new Account Service database user
GRANT SELECT ON account_service_accounts TO account_service_user;

-- Step 2: Gradually copy data to new Account Service database
-- Use AWS DMS (Database Migration Service) for continuous replication
```

**Database Migration Tool (Node.js)**:

```javascript
// migration-tools/database-sync.js
const { Client: OldDBClient } = require('pg');
const { Client: NewDBClient } = require('pg');

class DatabaseSynchronizer {
  constructor() {
    this.oldDB = new OldDBClient({
      host: process.env.MONOLITH_DB_HOST,
      database: 'banking_monolith',
      user: 'migration_user',
      password: process.env.MONOLITH_DB_PASSWORD
    });

    this.newDB = new NewDBClient({
      host: process.env.ACCOUNT_SERVICE_DB_HOST,
      database: 'account_service',
      user: 'admin',
      password: process.env.ACCOUNT_SERVICE_DB_PASSWORD
    });
  }

  async syncAccounts() {
    try {
      await this.oldDB.connect();
      await this.newDB.connect();

      // Step 1: Initial bulk copy (one-time)
      const accounts = await this.oldDB.query(`
        SELECT * FROM accounts 
        WHERE updated_at > $1
      `, [this.lastSyncTimestamp]);

      for (const account of accounts.rows) {
        await this.newDB.query(`
          INSERT INTO accounts (
            id, account_number, customer_name, email, 
            phone, account_type, balance, status, 
            created_at, updated_at
          ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10)
          ON CONFLICT (id) DO UPDATE SET
            balance = EXCLUDED.balance,
            status = EXCLUDED.status,
            updated_at = EXCLUDED.updated_at
        `, [
          account.id,
          account.account_number,
          account.customer_name,
          account.email,
          account.phone,
          account.account_type,
          account.balance,
          account.status,
          account.created_at,
          account.updated_at
        ]);
      }

      console.log(`Synced ${accounts.rows.length} accounts`);
    } catch (error) {
      console.error('Sync failed:', error);
      throw error;
    }
  }

  // Dual-write pattern: write to both databases during transition
  async dualWriteAccount(accountData) {
    const client1 = await this.oldDB.connect();
    const client2 = await this.newDB.connect();

    try {
      // Write to monolith DB
      await client1.query('BEGIN');
      const result1 = await client1.query(`
        INSERT INTO accounts (customer_name, email, account_type, balance)
        VALUES ($1, $2, $3, $4) RETURNING *
      `, [accountData.customerName, accountData.email, accountData.accountType, accountData.balance]);
      
      // Write to new service DB
      await client2.query('BEGIN');
      await client2.query(`
        INSERT INTO accounts (id, customer_name, email, account_type, balance)
        VALUES ($1, $2, $3, $4, $5)
      `, [result1.rows[0].id, accountData.customerName, accountData.email, accountData.accountType, accountData.balance]);

      await client1.query('COMMIT');
      await client2.query('COMMIT');

      return result1.rows[0];
    } catch (error) {
      await client1.query('ROLLBACK');
      await client2.query('ROLLBACK');
      throw error;
    } finally {
      client1.release();
      client2.release();
    }
  }
}

module.exports = DatabaseSynchronizer;
```

---

#### **Phase 3: Anti-Corruption Layer (ACL)**

**Purpose**: Protect new microservices from monolith's legacy data structures.

```javascript
// account-service/src/adapters/monolith-adapter.js
/**
 * Anti-Corruption Layer
 * Translates between monolith's legacy format and new domain model
 */
class MonolithAccountAdapter {
  // Convert monolith format → new domain model
  toDomainModel(legacyAccount) {
    return {
      id: legacyAccount.account_id, // Different field name
      accountNumber: legacyAccount.acc_no, // Different naming convention
      customerName: legacyAccount.cust_name,
      email: legacyAccount.email_addr,
      phone: this.formatPhone(legacyAccount.phone), // Different format
      accountType: this.mapAccountType(legacyAccount.type_code), // Code → Enum
      balance: parseFloat(legacyAccount.balance_amt) / 100, // Stored as cents
      status: legacyAccount.status === 1 ? 'ACTIVE' : 'INACTIVE',
      createdAt: new Date(legacyAccount.create_ts),
      updatedAt: new Date(legacyAccount.update_ts)
    };
  }

  // Convert new domain model → monolith format
  toLegacyModel(domainAccount) {
    return {
      account_id: domainAccount.id,
      acc_no: domainAccount.accountNumber,
      cust_name: domainAccount.customerName,
      email_addr: domainAccount.email,
      phone: domainAccount.phone.replace('+971', ''), // Remove country code
      type_code: this.mapToLegacyType(domainAccount.accountType),
      balance_amt: Math.round(domainAccount.balance * 100), // Convert to cents
      status: domainAccount.status === 'ACTIVE' ? 1 : 0,
      create_ts: domainAccount.createdAt.getTime(),
      update_ts: domainAccount.updatedAt.getTime()
    };
  }

  formatPhone(phone) {
    // Monolith stores without country code
    return phone.startsWith('971') ? `+${phone}` : `+971${phone}`;
  }

  mapAccountType(typeCode) {
    const mapping = {
      'SAV': 'SAVINGS',
      'CHK': 'CHECKING',
      'CUR': 'CURRENT',
      'FD': 'FIXED_DEPOSIT'
    };
    return mapping[typeCode] || 'UNKNOWN';
  }

  mapToLegacyType(accountType) {
    const reverseMapping = {
      'SAVINGS': 'SAV',
      'CHECKING': 'CHK',
      'CURRENT': 'CUR',
      'FIXED_DEPOSIT': 'FD'
    };
    return reverseMapping[accountType] || 'SAV';
  }
}

module.exports = MonolithAccountAdapter;
```

---

#### **Phase 4: Data Synchronization with AWS DMS**

```javascript
// infrastructure/cdk/database-migration-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as dms from 'aws-cdk-lib/aws-dms';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as secretsmanager from 'aws-cdk-lib/aws-secretsmanager';

export class DatabaseMigrationStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpc = ec2.Vpc.fromLookup(this, 'VPC', {
      vpcName: 'BankingVPC'
    });

    // DMS Replication Instance
    const replicationInstance = new dms.CfnReplicationInstance(this, 'ReplicationInstance', {
      replicationInstanceClass: 'dms.t3.medium',
      allocatedStorage: 100,
      vpcSecurityGroupIds: [/* security group ids */],
      replicationSubnetGroupIdentifier: 'dms-subnet-group',
      publiclyAccessible: false
    });

    // Source Endpoint (Monolith Database)
    const sourceEndpoint = new dms.CfnEndpoint(this, 'SourceEndpoint', {
      endpointType: 'source',
      engineName: 'postgres',
      serverName: 'monolith-db.internal',
      port: 5432,
      databaseName: 'banking_monolith',
      username: 'dms_user',
      password: cdk.SecretValue.secretsManager('monolith-db-secret').toString()
    });

    // Target Endpoint (New Account Service Database)
    const targetEndpoint = new dms.CfnEndpoint(this, 'TargetEndpoint', {
      endpointType: 'target',
      engineName: 'postgres',
      serverName: 'account-service-db.internal',
      port: 5432,
      databaseName: 'account_service',
      username: 'admin',
      password: cdk.SecretValue.secretsManager('account-service-db-secret').toString()
    });

    // Replication Task (Continuous Data Replication)
    const replicationTask = new dms.CfnReplicationTask(this, 'ReplicationTask', {
      replicationInstanceArn: replicationInstance.ref,
      sourceEndpointArn: sourceEndpoint.ref,
      targetEndpointArn: targetEndpoint.ref,
      migrationType: 'full-load-and-cdc', // Full load + Change Data Capture
      tableMappings: JSON.stringify({
        rules: [
          {
            'rule-type': 'selection',
            'rule-id': '1',
            'rule-name': 'accounts-table',
            'object-locator': {
              'schema-name': 'public',
              'table-name': 'accounts'
            },
            'rule-action': 'include'
          },
          {
            'rule-type': 'transformation',
            'rule-id': '2',
            'rule-name': 'rename-columns',
            'rule-action': 'rename',
            'rule-target': 'column',
            'object-locator': {
              'schema-name': 'public',
              'table-name': 'accounts',
              'column-name': 'acc_no'
            },
            'value': 'account_number'
          }
        ]
      })
    });
  }
}
```

---

#### **Phase 5: Event-Driven Data Consistency**

**Problem**: During migration, data can become inconsistent between monolith and microservices.

**Solution**: Use event sourcing and outbox pattern.

```javascript
// account-service/src/services/event-publisher.js
const { EventBridgeClient, PutEventsCommand } = require('@aws-sdk/client-eventbridge');

class EventPublisher {
  constructor() {
    this.eventBridge = new EventBridgeClient({ region: 'us-east-1' });
    this.eventBusName = 'banking-event-bus';
  }

  async publishAccountCreated(account) {
    const event = {
      Time: new Date(),
      Source: 'account.service',
      DetailType: 'AccountCreated',
      Detail: JSON.stringify({
        accountId: account.id,
        accountNumber: account.accountNumber,
        customerName: account.customerName,
        email: account.email,
        accountType: account.accountType,
        balance: account.balance,
        createdAt: account.createdAt
      }),
      EventBusName: this.eventBusName
    };

    try {
      const command = new PutEventsCommand({ Entries: [event] });
      const response = await this.eventBridge.send(command);
      console.log('Event published:', response);
    } catch (error) {
      console.error('Failed to publish event:', error);
      // Store in outbox table for retry
      await this.storeInOutbox(event);
    }
  }

  // Outbox pattern: reliable event delivery
  async storeInOutbox(event) {
    // Store event in database for later retry
    await db.query(`
      INSERT INTO event_outbox (event_type, payload, status)
      VALUES ($1, $2, 'PENDING')
    `, [event.DetailType, JSON.stringify(event)]);
  }
}

module.exports = EventPublisher;
```

**Outbox Processor (Background Job)**:

```javascript
// account-service/src/jobs/outbox-processor.js
const cron = require('node-cron');
const { EventBridgeClient, PutEventsCommand } = require('@aws-sdk/client-eventbridge');

class OutboxProcessor {
  constructor() {
    this.eventBridge = new EventBridgeClient({ region: 'us-east-1' });
  }

  start() {
    // Run every 30 seconds
    cron.schedule('*/30 * * * * *', async () => {
      await this.processOutbox();
    });
  }

  async processOutbox() {
    // Fetch pending events
    const events = await db.query(`
      SELECT * FROM event_outbox 
      WHERE status = 'PENDING' 
      ORDER BY created_at ASC 
      LIMIT 100
    `);

    for (const event of events.rows) {
      try {
        const payload = JSON.parse(event.payload);
        const command = new PutEventsCommand({ Entries: [payload] });
        await this.eventBridge.send(command);

        // Mark as published
        await db.query(`
          UPDATE event_outbox 
          SET status = 'PUBLISHED', published_at = NOW() 
          WHERE id = $1
        `, [event.id]);

        console.log(`Published event ${event.id}`);
      } catch (error) {
        console.error(`Failed to publish event ${event.id}:`, error);
        
        // Increment retry count
        await db.query(`
          UPDATE event_outbox 
          SET retry_count = retry_count + 1, 
              last_error = $1 
          WHERE id = $2
        `, [error.message, event.id]);
      }
    }
  }
}

module.exports = OutboxProcessor;
```

---

### 📅 Migration Timeline (6-Month Plan)

| **Phase** | **Duration** | **Activities** | **Deliverables** |
|-----------|--------------|----------------|------------------|
| **Phase 1: Foundation** | Week 1-4 | Setup API Gateway, feature flags, monitoring | Routing layer, 0% traffic |
| **Phase 2: First Service** | Week 5-8 | Extract Account Service, database sync, dual-write | Account Service, 10% traffic |
| **Phase 3: Validation** | Week 9-12 | Compare responses, fix discrepancies, increase traffic | 50% traffic, metrics dashboard |
| **Phase 4: Full Cutover** | Week 13-14 | Route 100% traffic, disable monolith account module | 100% traffic to Account Service |
| **Phase 5: Transaction Service** | Week 15-18 | Extract Transaction Service (Lambda + DynamoDB) | Transaction Service live |
| **Phase 6: Payment Service** | Week 19-22 | Extract Payment Service (EKS + Aurora) | Payment Service live |
| **Phase 7: Cleanup** | Week 23-26 | Decommission monolith modules, data migration complete | 80% services migrated |

---

### 🛡️ Risk Mitigation Strategies

#### **1. Data Inconsistency Risk**

**Mitigation**:
- Dual-write pattern during transition
- AWS DMS for continuous replication
- Reconciliation jobs to detect drift
- Event sourcing for audit trail

#### **2. Performance Degradation Risk**

**Mitigation**:
- Load testing before cutover
- Auto-scaling for new services
- Circuit breakers for fallback to monolith
- Caching with ElastiCache Redis

#### **3. Operational Complexity Risk**

**Mitigation**:
- Centralized logging (CloudWatch Logs Insights)
- Distributed tracing (AWS X-Ray)
- Unified monitoring dashboard
- Runbooks for common issues

#### **4. Team Coordination Risk**

**Mitigation**:
- Conway's Law: align teams with services
- Shared libraries for common code
- API contracts with Swagger/OpenAPI
- Regular sync meetings

---

### 🔄 Rollback Procedures

```javascript
// strangler-router/src/rollback-handler.js
class RollbackHandler {
  async rollbackAccountService() {
    console.log('ROLLBACK INITIATED: Routing 100% traffic back to monolith');

    // Update feature flag
    await this.updateFeatureFlag('accountService', {
      enabled: false,
      percentage: 0
    });

    // Send alert
    await this.sendAlert({
      severity: 'CRITICAL',
      message: 'Account Service rolled back to monolith',
      timestamp: new Date()
    });

    // Log rollback event
    await this.logEvent({
      eventType: 'ROLLBACK',
      service: 'Account Service',
      reason: 'High error rate detected'
    });
  }

  async updateFeatureFlag(service, config) {
    // Update in DynamoDB or AppConfig
    await dynamoDB.updateItem({
      TableName: 'feature-flags',
      Key: { service },
      UpdateExpression: 'SET #config = :config',
      ExpressionAttributeNames: { '#config': 'config' },
      ExpressionAttributeValues: { ':config': config }
    });
  }
}
```

---

### 📊 Success Metrics

1. **Traffic Distribution**: Monitor percentage of traffic to new services
2. **Error Rate**: Compare error rates (monolith vs microservice)
3. **Response Time**: Track latency improvements
4. **Data Consistency**: Reconciliation job success rate
5. **Team Velocity**: Sprint velocity before/after migration

---

### ✅ Key Takeaways

**DOs**:
- ✅ Use Strangler Fig pattern for gradual migration
- ✅ Implement feature flags for controlled rollout
- ✅ Dual-write during transition period
- ✅ Comprehensive monitoring and alerts
- ✅ Anti-Corruption Layer to protect new services
- ✅ Event-driven communication for loose coupling

**DON'Ts**:
- ❌ Don't do "big bang" migration (high risk)
- ❌ Don't skip data reconciliation
- ❌ Don't ignore team training
- ❌ Don't underestimate operational complexity
- ❌ Don't migrate without rollback plan

---

**End of Question 2**

---

