# Serverless Architecture with AWS Lambda

## Question 5: Lambda-Based Banking Transaction System

### 📋 Question Statement

Design a complete serverless banking transaction system using AWS Lambda, DynamoDB, and Step Functions. The system should handle deposits, withdrawals, and transfers with:
- Event-driven architecture
- ACID transaction guarantees using DynamoDB transactions
- Saga pattern for distributed transactions
- Auto-scaling and cost optimization
- Security and compliance (PCI-DSS)

---

### 🏗️ Serverless Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│         SERVERLESS BANKING TRANSACTION SYSTEM                │
└─────────────────────────────────────────────────────────────┘

┌─────────────┐
│ Mobile App  │
│  / Web UI   │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway (REST)                        │
│  • JWT Authentication                                        │
│  • Rate Limiting (1000 req/sec)                             │
│  • Request Validation                                        │
└─────────────────────────────────────────────────────────────┘
       │
       ├─────────────┬─────────────┬─────────────┐
       │             │             │             │
       ▼             ▼             ▼             ▼
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Deposit  │  │Withdrawal│  │ Transfer │  │ History  │
│ Lambda   │  │  Lambda  │  │  Lambda  │  │  Lambda  │
│ (512MB)  │  │ (512MB)  │  │ (1024MB) │  │ (256MB)  │
└────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │             │
     └─────────────┴─────────────┴─────────────┘
                   │
                   ▼
     ┌─────────────────────────────────────┐
     │     DynamoDB Tables                  │
     │  • Transactions (main table)         │
     │  • Accounts (GSI: accountId)         │
     │  • Ledger (time-series data)         │
     │  • Idempotency (prevent duplicates)  │
     └─────────────────────────────────────┘
                   │
                   ▼
     ┌─────────────────────────────────────┐
     │       DynamoDB Streams               │
     │  (Real-time event capture)           │
     └─────────────────────────────────────┘
                   │
                   ▼
     ┌─────────────────────────────────────┐
     │   EventBridge / Lambda Processors    │
     │  • Notification Service              │
     │  • Fraud Detection                   │
     │  • Analytics                         │
     │  • Audit Logging                     │
     └─────────────────────────────────────┘
```

---

### 💳 Lambda Function: Deposit Handler

```javascript
// lambdas/deposit/index.js
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, TransactWriteCommand, GetCommand } = require('@aws-sdk/lib-dynamodb');
const { EventBridgeClient, PutEventsCommand } = require('@aws-sdk/client-eventbridge');
const { v4: uuidv4 } = require('uuid');

const dynamoClient = new DynamoDBClient({ region: process.env.AWS_REGION });
const docClient = DynamoDBDocumentClient.from(dynamoClient);
const eventBridge = new EventBridgeClient({ region: process.env.AWS_REGION });

const TRANSACTIONS_TABLE = process.env.TRANSACTIONS_TABLE;
const ACCOUNTS_TABLE = process.env.ACCOUNTS_TABLE;
const LEDGER_TABLE = process.env.LEDGER_TABLE;
const IDEMPOTENCY_TABLE = process.env.IDEMPOTENCY_TABLE;

/**
 * Lambda Handler: Process Deposit
 * Features:
 * - Idempotent operations
 * - DynamoDB transactions (ACID)
 * - Double-entry bookkeeping
 * - Event publishing
 */
exports.handler = async (event) => {
  console.log('Deposit request:', JSON.stringify(event, null, 2));

  try {
    // Parse request body
    const body = JSON.parse(event.body);
    const { accountId, amount, currency = 'AED', idempotencyKey } = body;

    // Validation
    if (!accountId || !amount || !idempotencyKey) {
      return errorResponse(400, 'Missing required fields: accountId, amount, idempotencyKey');
    }

    if (amount <= 0) {
      return errorResponse(400, 'Amount must be positive');
    }

    // Check idempotency - prevent duplicate requests
    const existingTransaction = await checkIdempotency(idempotencyKey);
    if (existingTransaction) {
      console.log('Duplicate request detected, returning cached result');
      return successResponse(200, existingTransaction);
    }

    // Get current account details
    const account = await getAccount(accountId);
    if (!account) {
      return errorResponse(404, 'Account not found');
    }

    if (account.status !== 'ACTIVE') {
      return errorResponse(400, 'Account is not active');
    }

    // Generate transaction ID
    const transactionId = uuidv4();
    const timestamp = new Date().toISOString();

    // Calculate new balance
    const oldBalance = parseFloat(account.balance);
    const newBalance = oldBalance + amount;

    // Prepare DynamoDB transaction (ACID guarantees)
    const transactionItems = [
      // 1. Insert transaction record
      {
        Put: {
          TableName: TRANSACTIONS_TABLE,
          Item: {
            transactionId,
            accountId,
            transactionType: 'DEPOSIT',
            amount,
            currency,
            oldBalance,
            newBalance,
            status: 'COMPLETED',
            timestamp,
            createdAt: timestamp,
            ttl: Math.floor(Date.now() / 1000) + (365 * 24 * 60 * 60) // 1 year retention
          }
        }
      },
      // 2. Update account balance
      {
        Update: {
          TableName: ACCOUNTS_TABLE,
          Key: { accountId },
          UpdateExpression: 'SET balance = :newBalance, updatedAt = :timestamp, version = version + :inc',
          ExpressionAttributeValues: {
            ':newBalance': newBalance,
            ':timestamp': timestamp,
            ':inc': 1,
            ':expectedVersion': account.version
          },
          ConditionExpression: 'version = :expectedVersion' // Optimistic locking
        }
      },
      // 3. Create ledger entries (double-entry bookkeeping)
      {
        Put: {
          TableName: LEDGER_TABLE,
          Item: {
            ledgerId: `${transactionId}-DEBIT`,
            transactionId,
            accountId: 'CASH', // Source: External cash
            entryType: 'DEBIT',
            amount,
            currency,
            timestamp
          }
        }
      },
      {
        Put: {
          TableName: LEDGER_TABLE,
          Item: {
            ledgerId: `${transactionId}-CREDIT`,
            transactionId,
            accountId, // Destination: Customer account
            entryType: 'CREDIT',
            amount,
            currency,
            timestamp
          }
        }
      },
      // 4. Store idempotency key
      {
        Put: {
          TableName: IDEMPOTENCY_TABLE,
          Item: {
            idempotencyKey,
            transactionId,
            response: {
              transactionId,
              accountId,
              amount,
              newBalance,
              status: 'COMPLETED'
            },
            createdAt: timestamp,
            ttl: Math.floor(Date.now() / 1000) + (24 * 60 * 60) // 24 hours
          }
        }
      }
    ];

    // Execute DynamoDB transaction
    const command = new TransactWriteCommand({
      TransactItems: transactionItems
    });

    await docClient.send(command);

    console.log('Transaction completed successfully');

    // Publish event to EventBridge
    await publishEvent({
      source: 'transaction.service',
      detailType: 'DepositCompleted',
      detail: {
        transactionId,
        accountId,
        amount,
        currency,
        oldBalance,
        newBalance,
        timestamp
      }
    });

    // Return success response
    return successResponse(200, {
      transactionId,
      accountId,
      amount,
      currency,
      oldBalance,
      newBalance,
      status: 'COMPLETED',
      timestamp
    });

  } catch (error) {
    console.error('Deposit failed:', error);

    // Handle specific DynamoDB errors
    if (error.name === 'ConditionalCheckFailedException') {
      return errorResponse(409, 'Concurrent modification detected. Please retry.');
    }

    if (error.name === 'TransactionCanceledException') {
      return errorResponse(400, 'Transaction failed. Please try again.');
    }

    return errorResponse(500, 'Internal server error');
  }
};

// Helper: Check idempotency
async function checkIdempotency(idempotencyKey) {
  const command = new GetCommand({
    TableName: IDEMPOTENCY_TABLE,
    Key: { idempotencyKey }
  });

  const result = await docClient.send(command);
  return result.Item ? result.Item.response : null;
}

// Helper: Get account details
async function getAccount(accountId) {
  const command = new GetCommand({
    TableName: ACCOUNTS_TABLE,
    Key: { accountId }
  });

  const result = await docClient.send(command);
  return result.Item || null;
}

// Helper: Publish event to EventBridge
async function publishEvent({ source, detailType, detail }) {
  const command = new PutEventsCommand({
    Entries: [{
      Time: new Date(),
      Source: source,
      DetailType: detailType,
      Detail: JSON.stringify(detail),
      EventBusName: process.env.EVENT_BUS_NAME
    }]
  });

  try {
    await eventBridge.send(command);
    console.log('Event published:', detailType);
  } catch (error) {
    console.error('Failed to publish event:', error);
    // Don't fail the transaction if event publishing fails
  }
}

// Helper: Success response
function successResponse(statusCode, data) {
  return {
    statusCode,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify({
      success: true,
      data
    })
  };
}

// Helper: Error response
function errorResponse(statusCode, message) {
  return {
    statusCode,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify({
      success: false,
      error: message
    })
  };
}
```

---

### 🏦 Lambda Function: Withdrawal Handler

```javascript
// lambdas/withdrawal/index.js
const { DynamoDBDocumentClient, TransactWriteCommand, GetCommand } = require('@aws-sdk/lib-dynamodb');
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { EventBridgeClient, PutEventsCommand } = require('@aws-sdk/client-eventbridge');
const { v4: uuidv4 } = require('uuid');

const dynamoClient = new DynamoDBClient({ region: process.env.AWS_REGION });
const docClient = DynamoDBDocumentClient.from(dynamoClient);
const eventBridge = new EventBridgeClient({ region: process.env.AWS_REGION });

const TRANSACTIONS_TABLE = process.env.TRANSACTIONS_TABLE;
const ACCOUNTS_TABLE = process.env.ACCOUNTS_TABLE;
const LEDGER_TABLE = process.env.LEDGER_TABLE;
const IDEMPOTENCY_TABLE = process.env.IDEMPOTENCY_TABLE;

/**
 * Lambda Handler: Process Withdrawal
 * Business Rules:
 * - Check sufficient balance
 * - Maintain minimum balance (AED 1,000 for savings)
 * - Daily withdrawal limit (AED 50,000)
 */
exports.handler = async (event) => {
  console.log('Withdrawal request:', JSON.stringify(event, null, 2));

  try {
    const body = JSON.parse(event.body);
    const { accountId, amount, currency = 'AED', idempotencyKey } = body;

    // Validation
    if (!accountId || !amount || !idempotencyKey) {
      return errorResponse(400, 'Missing required fields');
    }

    if (amount <= 0) {
      return errorResponse(400, 'Amount must be positive');
    }

    // Check idempotency
    const existingTransaction = await checkIdempotency(idempotencyKey);
    if (existingTransaction) {
      return successResponse(200, existingTransaction);
    }

    // Get account details
    const account = await getAccount(accountId);
    if (!account) {
      return errorResponse(404, 'Account not found');
    }

    if (account.status !== 'ACTIVE') {
      return errorResponse(400, 'Account is not active');
    }

    // Business rule: Check minimum balance
    const minimumBalance = account.accountType === 'SAVINGS' ? 1000 : 0;
    const oldBalance = parseFloat(account.balance);
    const newBalance = oldBalance - amount;

    if (newBalance < minimumBalance) {
      return errorResponse(400, `Insufficient balance. Minimum balance required: AED ${minimumBalance}`);
    }

    // Business rule: Check daily withdrawal limit
    const dailyLimit = 50000;
    const todayWithdrawals = await getTodayWithdrawals(accountId);
    
    if (todayWithdrawals + amount > dailyLimit) {
      return errorResponse(400, `Daily withdrawal limit exceeded. Limit: AED ${dailyLimit}`);
    }

    // Generate transaction ID
    const transactionId = uuidv4();
    const timestamp = new Date().toISOString();

    // DynamoDB transaction
    const transactionItems = [
      {
        Put: {
          TableName: TRANSACTIONS_TABLE,
          Item: {
            transactionId,
            accountId,
            transactionType: 'WITHDRAWAL',
            amount,
            currency,
            oldBalance,
            newBalance,
            status: 'COMPLETED',
            timestamp,
            createdAt: timestamp,
            ttl: Math.floor(Date.now() / 1000) + (365 * 24 * 60 * 60)
          }
        }
      },
      {
        Update: {
          TableName: ACCOUNTS_TABLE,
          Key: { accountId },
          UpdateExpression: 'SET balance = :newBalance, updatedAt = :timestamp, version = version + :inc',
          ExpressionAttributeValues: {
            ':newBalance': newBalance,
            ':timestamp': timestamp,
            ':inc': 1,
            ':expectedVersion': account.version
          },
          ConditionExpression: 'version = :expectedVersion'
        }
      },
      {
        Put: {
          TableName: LEDGER_TABLE,
          Item: {
            ledgerId: `${transactionId}-DEBIT`,
            transactionId,
            accountId,
            entryType: 'DEBIT',
            amount,
            currency,
            timestamp
          }
        }
      },
      {
        Put: {
          TableName: LEDGER_TABLE,
          Item: {
            ledgerId: `${transactionId}-CREDIT`,
            transactionId,
            accountId: 'CASH',
            entryType: 'CREDIT',
            amount,
            currency,
            timestamp
          }
        }
      },
      {
        Put: {
          TableName: IDEMPOTENCY_TABLE,
          Item: {
            idempotencyKey,
            transactionId,
            response: {
              transactionId,
              accountId,
              amount,
              newBalance,
              status: 'COMPLETED'
            },
            createdAt: timestamp,
            ttl: Math.floor(Date.now() / 1000) + (24 * 60 * 60)
          }
        }
      }
    ];

    const command = new TransactWriteCommand({
      TransactItems: transactionItems
    });

    await docClient.send(command);

    // Publish event
    await publishEvent({
      source: 'transaction.service',
      detailType: 'WithdrawalCompleted',
      detail: {
        transactionId,
        accountId,
        amount,
        currency,
        oldBalance,
        newBalance,
        timestamp
      }
    });

    return successResponse(200, {
      transactionId,
      accountId,
      amount,
      currency,
      oldBalance,
      newBalance,
      status: 'COMPLETED',
      timestamp
    });

  } catch (error) {
    console.error('Withdrawal failed:', error);

    if (error.name === 'ConditionalCheckFailedException') {
      return errorResponse(409, 'Concurrent modification detected. Please retry.');
    }

    return errorResponse(500, 'Internal server error');
  }
};

// Helper: Get today's withdrawals
async function getTodayWithdrawals(accountId) {
  const today = new Date().toISOString().split('T')[0];
  
  // Query transactions table (requires GSI on accountId + date)
  // Simplified version - in production, use DynamoDB Query with GSI
  return 0; // Placeholder
}

// Helper functions (same as deposit handler)
async function checkIdempotency(idempotencyKey) {
  const command = new GetCommand({
    TableName: IDEMPOTENCY_TABLE,
    Key: { idempotencyKey }
  });
  const result = await docClient.send(command);
  return result.Item ? result.Item.response : null;
}

async function getAccount(accountId) {
  const command = new GetCommand({
    TableName: ACCOUNTS_TABLE,
    Key: { accountId }
  });
  const result = await docClient.send(command);
  return result.Item || null;
}

async function publishEvent({ source, detailType, detail }) {
  const command = new PutEventsCommand({
    Entries: [{
      Time: new Date(),
      Source: source,
      DetailType: detailType,
      Detail: JSON.stringify(detail),
      EventBusName: process.env.EVENT_BUS_NAME
    }]
  });
  try {
    await eventBridge.send(command);
  } catch (error) {
    console.error('Failed to publish event:', error);
  }
}

function successResponse(statusCode, data) {
  return {
    statusCode,
    headers: { 'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*' },
    body: JSON.stringify({ success: true, data })
  };
}

function errorResponse(statusCode, message) {
  return {
    statusCode,
    headers: { 'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*' },
    body: JSON.stringify({ success: false, error: message })
  };
}
```

---

### 🔄 Lambda Function: Transfer Handler (Step Functions Saga)

```javascript
// lambdas/transfer/index.js
const { SFNClient, StartExecutionCommand } = require('@aws-sdk/client-sfn');
const { v4: uuidv4 } = require('uuid');

const sfnClient = new SFNClient({ region: process.env.AWS_REGION });
const STATE_MACHINE_ARN = process.env.TRANSFER_STATE_MACHINE_ARN;

/**
 * Lambda Handler: Initiate Transfer
 * Starts Step Functions state machine for saga orchestration
 */
exports.handler = async (event) => {
  console.log('Transfer request:', JSON.stringify(event, null, 2));

  try {
    const body = JSON.parse(event.body);
    const { fromAccountId, toAccountId, amount, currency = 'AED', idempotencyKey } = body;

    // Validation
    if (!fromAccountId || !toAccountId || !amount || !idempotencyKey) {
      return errorResponse(400, 'Missing required fields');
    }

    if (amount <= 0) {
      return errorResponse(400, 'Amount must be positive');
    }

    if (fromAccountId === toAccountId) {
      return errorResponse(400, 'Cannot transfer to same account');
    }

    // Generate transaction ID
    const transactionId = uuidv4();

    // Start Step Functions execution
    const input = {
      transactionId,
      fromAccountId,
      toAccountId,
      amount,
      currency,
      idempotencyKey,
      timestamp: new Date().toISOString()
    };

    const command = new StartExecutionCommand({
      stateMachineArn: STATE_MACHINE_ARN,
      name: `transfer-${transactionId}`,
      input: JSON.stringify(input)
    });

    const result = await sfnClient.send(command);

    console.log('Step Functions execution started:', result.executionArn);

    return successResponse(202, {
      transactionId,
      status: 'PROCESSING',
      message: 'Transfer initiated successfully',
      executionArn: result.executionArn
    });

  } catch (error) {
    console.error('Transfer initiation failed:', error);
    return errorResponse(500, 'Failed to initiate transfer');
  }
};

function successResponse(statusCode, data) {
  return {
    statusCode,
    headers: { 'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*' },
    body: JSON.stringify({ success: true, data })
  };
}

function errorResponse(statusCode, message) {
  return {
    statusCode,
    headers: { 'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*' },
    body: JSON.stringify({ success: false, error: message })
  };
}
```

**Step Functions State Machine Definition**:

```json
{
  "Comment": "Transfer Saga - Orchestrated by Step Functions",
  "StartAt": "ValidateAccounts",
  "States": {
    "ValidateAccounts": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:validate-accounts",
      "ResultPath": "$.validation",
      "Catch": [{
        "ErrorEquals": ["ValidationError"],
        "ResultPath": "$.error",
        "Next": "TransferFailed"
      }],
      "Next": "DebitSourceAccount"
    },
    "DebitSourceAccount": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:debit-account",
      "ResultPath": "$.debitResult",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "ResultPath": "$.error",
        "Next": "TransferFailed"
      }],
      "Next": "CreditDestinationAccount"
    },
    "CreditDestinationAccount": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:credit-account",
      "ResultPath": "$.creditResult",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "ResultPath": "$.error",
        "Next": "CompensateDebit"
      }],
      "Next": "RecordTransaction"
    },
    "RecordTransaction": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:record-transaction",
      "ResultPath": "$.transaction",
      "Next": "PublishEvent"
    },
    "PublishEvent": {
      "Type": "Task",
      "Resource": "arn:aws:states:::events:putEvents",
      "Parameters": {
        "Entries": [{
          "Source": "transaction.service",
          "DetailType": "TransferCompleted",
          "Detail.$": "$"
        }]
      },
      "Next": "TransferSuccess"
    },
    "TransferSuccess": {
      "Type": "Succeed"
    },
    "CompensateDebit": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:compensate-debit",
      "ResultPath": "$.compensation",
      "Next": "TransferFailed"
    },
    "TransferFailed": {
      "Type": "Fail",
      "Error": "TransferFailed",
      "Cause": "Transfer could not be completed"
    }
  }
}
```

---

### 📊 DynamoDB Table Definitions

```javascript
// infrastructure/cdk/dynamodb-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

export class DynamoDBStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Transactions Table
    const transactionsTable = new dynamodb.Table(this, 'TransactionsTable', {
      tableName: 'banking-transactions',
      partitionKey: { name: 'transactionId', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      stream: dynamodb.StreamViewType.NEW_AND_OLD_IMAGES,
      pointInTimeRecovery: true,
      encryption: dynamodb.TableEncryption.AWS_MANAGED,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
      timeToLiveAttribute: 'ttl'
    });

    // GSI for querying by account
    transactionsTable.addGlobalSecondaryIndex({
      indexName: 'AccountIdIndex',
      partitionKey: { name: 'accountId', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'timestamp', type: dynamodb.AttributeType.STRING },
      projectionType: dynamodb.ProjectionType.ALL
    });

    // GSI for querying by date
    transactionsTable.addGlobalSecondaryIndex({
      indexName: 'DateIndex',
      partitionKey: { name: 'transactionType', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'timestamp', type: dynamodb.AttributeType.STRING },
      projectionType: dynamodb.ProjectionType.ALL
    });

    // Accounts Table
    const accountsTable = new dynamodb.Table(this, 'AccountsTable', {
      tableName: 'banking-accounts',
      partitionKey: { name: 'accountId', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      pointInTimeRecovery: true,
      encryption: dynamodb.TableEncryption.AWS_MANAGED
    });

    // Ledger Table (Double-entry bookkeeping)
    const ledgerTable = new dynamodb.Table(this, 'LedgerTable', {
      tableName: 'banking-ledger',
      partitionKey: { name: 'ledgerId', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'timestamp', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      pointInTimeRecovery: true
    });

    ledgerTable.addGlobalSecondaryIndex({
      indexName: 'TransactionIndex',
      partitionKey: { name: 'transactionId', type: dynamodb.AttributeType.STRING },
      projectionType: dynamodb.ProjectionType.ALL
    });

    // Idempotency Table
    const idempotencyTable = new dynamodb.Table(this, 'IdempotencyTable', {
      tableName: 'banking-idempotency',
      partitionKey: { name: 'idempotencyKey', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      timeToLiveAttribute: 'ttl'
    });

    // Outputs
    new cdk.CfnOutput(this, 'TransactionsTableName', {
      value: transactionsTable.tableName
    });
    new cdk.CfnOutput(this, 'AccountsTableName', {
      value: accountsTable.tableName
    });
  }
}
```

---

### 🔐 Lambda Function: API Gateway Authorizer

```javascript
// lambdas/authorizer/index.js
const jwt = require('jsonwebtoken');
const jwksClient = require('jwks-rsa');

const JWKS_URI = process.env.JWKS_URI;
const TOKEN_ISSUER = process.env.TOKEN_ISSUER;
const AUDIENCE = process.env.AUDIENCE;

const client = jwksClient({
  cache: true,
  rateLimit: true,
  jwksRequestsPerMinute: 10,
  jwksUri: JWKS_URI
});

/**
 * Lambda Authorizer for API Gateway
 * Validates JWT tokens from Cognito
 */
exports.handler = async (event) => {
  console.log('Authorization request:', JSON.stringify(event, null, 2));

  const token = event.authorizationToken;

  if (!token || !token.startsWith('Bearer ')) {
    return generatePolicy('user', 'Deny', event.methodArn);
  }

  const jwtToken = token.substring(7);

  try {
    // Decode token header to get kid
    const decoded = jwt.decode(jwtToken, { complete: true });
    if (!decoded) {
      throw new Error('Invalid token');
    }

    // Get signing key
    const key = await getSigningKey(decoded.header.kid);

    // Verify token
    const verified = jwt.verify(jwtToken, key, {
      issuer: TOKEN_ISSUER,
      audience: AUDIENCE,
      algorithms: ['RS256']
    });

    console.log('Token verified:', verified);

    // Generate allow policy with context
    return generatePolicy(verified.sub, 'Allow', event.methodArn, {
      userId: verified.sub,
      email: verified.email,
      accountId: verified['custom:accountId'] || 'unknown'
    });

  } catch (error) {
    console.error('Authorization failed:', error);
    return generatePolicy('user', 'Deny', event.methodArn);
  }
};

// Get signing key from JWKS
function getSigningKey(kid) {
  return new Promise((resolve, reject) => {
    client.getSigningKey(kid, (err, key) => {
      if (err) {
        reject(err);
      } else {
        const signingKey = key.publicKey || key.rsaPublicKey;
        resolve(signingKey);
      }
    });
  });
}

// Generate IAM policy
function generatePolicy(principalId, effect, resource, context = {}) {
  const authResponse = {
    principalId
  };

  if (effect && resource) {
    authResponse.policyDocument = {
      Version: '2012-10-17',
      Statement: [{
        Action: 'execute-api:Invoke',
        Effect: effect,
        Resource: resource
      }]
    };
  }

  // Pass user context to Lambda
  authResponse.context = context;

  return authResponse;
}
```

---

### 🎓 Interview Discussion Points - Q5

**Q1: How do you ensure ACID transactions in DynamoDB?**

**A**: Use `TransactWriteItems` API:
- Atomic: All or nothing execution
- Consistent: Conditional checks with optimistic locking
- Isolated: Version numbers prevent conflicts
- Durable: Data is replicated across AZs

**Q2: How do you handle idempotency in Lambda?**

**A**:
- Client-generated idempotency key
- Store in DynamoDB with 24-hour TTL
- Check before processing
- Return cached result for duplicates

**Q3: What are Lambda cold starts and how to mitigate?**

**A**:
- **Cold start**: First invocation takes longer (100-500ms)
- **Mitigation**:
  - Provisioned concurrency for critical functions
  - Keep functions warm with CloudWatch Events
  - Optimize package size (< 10MB)
  - Use ARM64 architecture (Graviton2)

---

## Question 6: API Gateway Integration with Custom Authorizers

### 📋 Question Statement

Design a complete API Gateway solution for Emirates NBD with:
- Multiple authorizer types (Cognito, Lambda, IAM)
- Request/response validation
- Rate limiting and throttling
- Request transformation
- Caching strategies
- VPC Link for private integrations
- Monitoring and logging

---

### 🚪 API Gateway Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      API GATEWAY                             │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │             Request Pipeline                        │    │
│  │  1. Authentication (Cognito/Lambda/IAM)            │    │
│  │  2. Rate Limiting (Usage Plans)                    │    │
│  │  3. Request Validation (JSON Schema)               │    │
│  │  4. Request Transformation (VTL)                   │    │
│  │  5. Caching (300 seconds)                          │    │
│  │  6. Integration (Lambda/HTTP/VPC Link)             │    │
│  │  7. Response Transformation                        │    │
│  │  8. CORS Headers                                   │    │
│  └────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
         │             │              │             │
         ▼             ▼              ▼             ▼
   ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
   │ Lambda  │  │  Lambda  │  │   HTTP   │  │ VPC Link │
   │Deposit  │  │Withdraw  │  │ Backend  │  │   ECS    │
   └─────────┘  └──────────┘  └──────────┘  └──────────┘
```

---

### 🔐 Complete CDK Stack: API Gateway with Authorizers

```typescript
// infrastructure/cdk/api-gateway-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as cognito from 'aws-cdk-lib/aws-cognito';
import * as logs from 'aws-cdk-lib/aws-logs';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as wafv2 from 'aws-cdk-lib/aws-wafv2';

export class APIGatewayStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // ============================================
    // 1. COGNITO USER POOL
    // ============================================
    const userPool = new cognito.UserPool(this, 'BankingUserPool', {
      userPoolName: 'emirates-nbd-users',
      selfSignUpEnabled: false, // Admin creates accounts
      signInAliases: {
        email: true,
        phone: true,
        username: true
      },
      autoVerify: {
        email: true,
        phone: true
      },
      passwordPolicy: {
        minLength: 12,
        requireLowercase: true,
        requireUppercase: true,
        requireDigits: true,
        requireSymbols: true,
        tempPasswordValidity: cdk.Duration.days(3)
      },
      mfa: cognito.Mfa.REQUIRED,
      mfaSecondFactor: {
        sms: true,
        otp: true
      },
      accountRecovery: cognito.AccountRecovery.EMAIL_AND_PHONE_WITHOUT_MFA,
      removalPolicy: cdk.RemovalPolicy.RETAIN
    });

    // Add custom attributes
    userPool.addDomain('BankingDomain', {
      cognitoDomain: {
        domainPrefix: 'emirates-nbd-banking'
      }
    });

    const userPoolClient = userPool.addClient('WebClient', {
      userPoolClientName: 'web-app',
      authFlows: {
        userPassword: true,
        userSrp: true,
        custom: true
      },
      oAuth: {
        flows: {
          authorizationCodeGrant: true,
          implicitCodeGrant: true
        },
        scopes: [
          cognito.OAuthScope.OPENID,
          cognito.OAuthScope.EMAIL,
          cognito.OAuthScope.PROFILE
        ],
        callbackUrls: ['https://app.emiratesnbd.com/callback'],
        logoutUrls: ['https://app.emiratesnbd.com/logout']
      },
      accessTokenValidity: cdk.Duration.minutes(60),
      idTokenValidity: cdk.Duration.minutes(60),
      refreshTokenValidity: cdk.Duration.days(30)
    });

    // ============================================
    // 2. LAMBDA AUTHORIZER
    // ============================================
    const authorizerLambda = new lambda.Function(this, 'CustomAuthorizer', {
      functionName: 'api-gateway-authorizer',
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambdas/authorizer'),
      environment: {
        JWKS_URI: `https://cognito-idp.${this.region}.amazonaws.com/${userPool.userPoolId}/.well-known/jwks.json`,
        TOKEN_ISSUER: `https://cognito-idp.${this.region}.amazonaws.com/${userPool.userPoolId}`,
        AUDIENCE: userPoolClient.userPoolClientId
      },
      timeout: cdk.Duration.seconds(10),
      memorySize: 256
    });

    // ============================================
    // 3. API GATEWAY REST API
    // ============================================
    const logGroup = new logs.LogGroup(this, 'APIGatewayLogs', {
      logGroupName: '/aws/apigateway/banking-api',
      retention: logs.RetentionDays.ONE_MONTH,
      removalPolicy: cdk.RemovalPolicy.DESTROY
    });

    const api = new apigateway.RestApi(this, 'BankingAPI', {
      restApiName: 'Emirates NBD Banking API',
      description: 'Comprehensive banking API with multiple authorizers',
      deployOptions: {
        stageName: 'prod',
        loggingLevel: apigateway.MethodLoggingLevel.INFO,
        dataTraceEnabled: true,
        metricsEnabled: true,
        tracingEnabled: true,
        accessLogDestination: new apigateway.LogGroupLogDestination(logGroup),
        accessLogFormat: apigateway.AccessLogFormat.jsonWithStandardFields({
          caller: true,
          httpMethod: true,
          ip: true,
          protocol: true,
          requestTime: true,
          resourcePath: true,
          responseLength: true,
          status: true,
          user: true
        }),
        throttlingBurstLimit: 5000,
        throttlingRateLimit: 2000
      },
      defaultCorsPreflightOptions: {
        allowOrigins: ['https://app.emiratesnbd.com'],
        allowMethods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
        allowHeaders: [
          'Content-Type',
          'Authorization',
          'X-Amz-Date',
          'X-Api-Key',
          'X-Correlation-Id'
        ],
        allowCredentials: true,
        maxAge: cdk.Duration.hours(1)
      },
      cloudWatchRole: true,
      endpointConfiguration: {
        types: [apigateway.EndpointType.REGIONAL]
      }
    });

    // ============================================
    // 4. AUTHORIZERS
    // ============================================

    // Cognito Authorizer
    const cognitoAuthorizer = new apigateway.CognitoUserPoolsAuthorizer(this, 'CognitoAuthorizer', {
      cognitoUserPools: [userPool],
      authorizerName: 'CognitoAuth',
      identitySource: 'method.request.header.Authorization',
      resultsCacheTtl: cdk.Duration.minutes(5)
    });

    // Lambda Token Authorizer
    const lambdaAuthorizer = new apigateway.TokenAuthorizer(this, 'LambdaTokenAuthorizer', {
      handler: authorizerLambda,
      authorizerName: 'LambdaAuth',
      identitySource: 'method.request.header.Authorization',
      resultsCacheTtl: cdk.Duration.minutes(5)
    });

    // Lambda Request Authorizer (with request parameters)
    const requestAuthorizer = new apigateway.RequestAuthorizer(this, 'LambdaRequestAuthorizer', {
      handler: authorizerLambda,
      authorizerName: 'RequestAuth',
      identitySources: [
        apigateway.IdentitySource.header('Authorization'),
        apigateway.IdentitySource.queryString('token')
      ],
      resultsCacheTtl: cdk.Duration.minutes(5)
    });

    // ============================================
    // 5. REQUEST MODELS & VALIDATORS
    // ============================================

    // Request model for deposit
    const depositRequestModel = api.addModel('DepositRequest', {
      contentType: 'application/json',
      modelName: 'DepositRequest',
      schema: {
        type: apigateway.JsonSchemaType.OBJECT,
        required: ['accountId', 'amount', 'idempotencyKey'],
        properties: {
          accountId: {
            type: apigateway.JsonSchemaType.STRING,
            pattern: '^[a-zA-Z0-9-]{8,}$',
            description: 'Account identifier'
          },
          amount: {
            type: apigateway.JsonSchemaType.NUMBER,
            minimum: 1,
            maximum: 1000000,
            description: 'Deposit amount in AED'
          },
          currency: {
            type: apigateway.JsonSchemaType.STRING,
            enum: ['AED', 'USD', 'EUR', 'GBP'],
            default: 'AED'
          },
          idempotencyKey: {
            type: apigateway.JsonSchemaType.STRING,
            pattern: '^[a-zA-Z0-9-]{20,}$',
            description: 'Unique request identifier'
          },
          description: {
            type: apigateway.JsonSchemaType.STRING,
            maxLength: 200
          }
        }
      }
    });

    // Response model
    const transactionResponseModel = api.addModel('TransactionResponse', {
      contentType: 'application/json',
      modelName: 'TransactionResponse',
      schema: {
        type: apigateway.JsonSchemaType.OBJECT,
        properties: {
          success: { type: apigateway.JsonSchemaType.BOOLEAN },
          data: {
            type: apigateway.JsonSchemaType.OBJECT,
            properties: {
              transactionId: { type: apigateway.JsonSchemaType.STRING },
              accountId: { type: apigateway.JsonSchemaType.STRING },
              amount: { type: apigateway.JsonSchemaType.NUMBER },
              status: { type: apigateway.JsonSchemaType.STRING },
              timestamp: { type: apigateway.JsonSchemaType.STRING }
            }
          }
        }
      }
    });

    // Request validator
    const requestValidator = api.addRequestValidator('RequestValidator', {
      requestValidatorName: 'body-and-params-validator',
      validateRequestBody: true,
      validateRequestParameters: true
    });

    // ============================================
    // 6. LAMBDA FUNCTIONS
    // ============================================

    const depositLambda = new lambda.Function(this, 'DepositFunction', {
      functionName: 'deposit-handler',
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambdas/deposit'),
      environment: {
        TRANSACTIONS_TABLE: 'banking-transactions',
        ACCOUNTS_TABLE: 'banking-accounts',
        LEDGER_TABLE: 'banking-ledger',
        IDEMPOTENCY_TABLE: 'banking-idempotency',
        EVENT_BUS_NAME: 'banking-event-bus'
      },
      timeout: cdk.Duration.seconds(30),
      memorySize: 512,
      reservedConcurrentExecutions: 100,
      tracing: lambda.Tracing.ACTIVE
    });

    const withdrawalLambda = new lambda.Function(this, 'WithdrawalFunction', {
      functionName: 'withdrawal-handler',
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambdas/withdrawal'),
      environment: {
        TRANSACTIONS_TABLE: 'banking-transactions',
        ACCOUNTS_TABLE: 'banking-accounts',
        LEDGER_TABLE: 'banking-ledger',
        IDEMPOTENCY_TABLE: 'banking-idempotency',
        EVENT_BUS_NAME: 'banking-event-bus'
      },
      timeout: cdk.Duration.seconds(30),
      memorySize: 512,
      tracing: lambda.Tracing.ACTIVE
    });

    // ============================================
    // 7. API RESOURCES & METHODS
    // ============================================

    // /transactions resource
    const transactionsResource = api.root.addResource('transactions');

    // POST /transactions/deposit (Cognito Authorizer)
    const depositResource = transactionsResource.addResource('deposit');
    depositResource.addMethod('POST', 
      new apigateway.LambdaIntegration(depositLambda, {
        proxy: true,
        integrationResponses: [{
          statusCode: '200',
          responseTemplates: {
            'application/json': '$input.json("$")'
          }
        }]
      }),
      {
        authorizer: cognitoAuthorizer,
        authorizationType: apigateway.AuthorizationType.COGNITO,
        requestValidator,
        requestModels: {
          'application/json': depositRequestModel
        },
        methodResponses: [{
          statusCode: '200',
          responseModels: {
            'application/json': transactionResponseModel
          }
        }]
      }
    );

    // POST /transactions/withdraw (Lambda Authorizer)
    const withdrawResource = transactionsResource.addResource('withdraw');
    withdrawResource.addMethod('POST',
      new apigateway.LambdaIntegration(withdrawalLambda, {
        proxy: true
      }),
      {
        authorizer: lambdaAuthorizer,
        authorizationType: apigateway.AuthorizationType.CUSTOM,
        requestValidator,
        requestModels: {
          'application/json': depositRequestModel // Reuse model
        }
      }
    );

    // GET /transactions/{transactionId} (IAM Authorizer - for internal services)
    const transactionByIdResource = transactionsResource.addResource('{transactionId}');
    transactionByIdResource.addMethod('GET',
      new apigateway.LambdaIntegration(depositLambda), // Reuse lambda
      {
        authorizationType: apigateway.AuthorizationType.IAM,
        requestParameters: {
          'method.request.path.transactionId': true
        }
      }
    );

    // ============================================
    // 8. USAGE PLANS & API KEYS
    // ============================================

    // Free Tier
    const freeTierPlan = api.addUsagePlan('FreeTier', {
      name: 'Free-Tier',
      description: 'Free tier for basic usage',
      throttle: {
        rateLimit: 10,
        burstLimit: 20
      },
      quota: {
        limit: 1000,
        period: apigateway.Period.MONTH
      }
    });

    // Premium Tier
    const premiumPlan = api.addUsagePlan('PremiumTier', {
      name: 'Premium-Tier',
      description: 'Premium tier for high-volume customers',
      throttle: {
        rateLimit: 1000,
        burstLimit: 2000
      },
      quota: {
        limit: 1000000,
        period: apigateway.Period.MONTH
      }
    });

    // Enterprise Tier
    const enterprisePlan = api.addUsagePlan('EnterpriseTier', {
      name: 'Enterprise-Tier',
      description: 'Enterprise tier with unlimited access',
      throttle: {
        rateLimit: 10000,
        burstLimit: 20000
      }
      // No quota limit
    });

    // Associate with API stage
    freeTierPlan.addApiStage({ stage: api.deploymentStage });
    premiumPlan.addApiStage({ stage: api.deploymentStage });
    enterprisePlan.addApiStage({ stage: api.deploymentStage });

    // Create API Keys
    const partnerApiKey = api.addApiKey('PartnerKey', {
      apiKeyName: 'partner-integration-key',
      description: 'API key for partner integrations'
    });

    premiumPlan.addApiKey(partnerApiKey);

    // ============================================
    // 9. VPC LINK (Private Integration)
    // ============================================

    const vpc = ec2.Vpc.fromLookup(this, 'BankingVPC', {
      vpcName: 'banking-vpc'
    });

    const vpcLink = new apigateway.VpcLink(this, 'VPCLink', {
      vpcLinkName: 'banking-vpc-link',
      targets: [/* NLB target */],
      description: 'VPC Link to private ECS services'
    });

    // Private resource (ECS backend via VPC Link)
    const accountsResource = api.root.addResource('accounts');
    accountsResource.addMethod('GET',
      new apigateway.Integration({
        type: apigateway.IntegrationType.HTTP_PROXY,
        integrationHttpMethod: 'GET',
        uri: 'http://account-service.internal:3000/api/accounts',
        options: {
          connectionType: apigateway.ConnectionType.VPC_LINK,
          vpcLink
        }
      }),
      {
        authorizer: cognitoAuthorizer,
        authorizationType: apigateway.AuthorizationType.COGNITO
      }
    );

    // ============================================
    // 10. CACHING
    // ============================================

    const cachedResource = api.root.addResource('balance');
    cachedResource.addMethod('GET',
      new apigateway.LambdaIntegration(depositLambda),
      {
        authorizer: cognitoAuthorizer,
        authorizationType: apigateway.AuthorizationType.COGNITO,
        requestParameters: {
          'method.request.querystring.accountId': true
        },
        methodResponses: [{
          statusCode: '200',
          responseParameters: {
            'method.response.header.Cache-Control': true
          }
        }]
      }
    );

    // Enable caching on deployment stage
    const deployment = new apigateway.Deployment(this, 'Deployment', {
      api
    });

    const stage = new apigateway.Stage(this, 'ProdStage', {
      deployment,
      stageName: 'prod',
      cacheClusterEnabled: true,
      cacheClusterSize: '0.5', // 0.5GB cache
      cacheTtl: cdk.Duration.seconds(300),
      cacheDataEncrypted: true,
      cachingEnabled: true,
      methodOptions: {
        '/balance/GET': {
          cachingEnabled: true,
          cacheTtl: cdk.Duration.seconds(60),
          cacheDataEncrypted: true
        }
      }
    });

    // ============================================
    // 11. WAF (Web Application Firewall)
    // ============================================

    const webAcl = new wafv2.CfnWebACL(this, 'APIGatewayWAF', {
      scope: 'REGIONAL',
      defaultAction: { allow: {} },
      rules: [
        {
          name: 'RateLimitRule',
          priority: 1,
          statement: {
            rateBasedStatement: {
              limit: 2000,
              aggregateKeyType: 'IP'
            }
          },
          action: { block: {} },
          visibilityConfig: {
            sampledRequestsEnabled: true,
            cloudWatchMetricsEnabled: true,
            metricName: 'RateLimitRule'
          }
        },
        {
          name: 'GeoBlockRule',
          priority: 2,
          statement: {
            geoMatchStatement: {
              countryCodes: ['CN', 'RU', 'KP'] // Block specific countries
            }
          },
          action: { block: {} },
          visibilityConfig: {
            sampledRequestsEnabled: true,
            cloudWatchMetricsEnabled: true,
            metricName: 'GeoBlockRule'
          }
        }
      ],
      visibilityConfig: {
        sampledRequestsEnabled: true,
        cloudWatchMetricsEnabled: true,
        metricName: 'BankingAPIWAF'
      }
    });

    // ============================================
    // 12. OUTPUTS
    // ============================================

    new cdk.CfnOutput(this, 'APIEndpoint', {
      value: api.url,
      description: 'API Gateway Endpoint',
      exportName: 'BankingAPIEndpoint'
    });

    new cdk.CfnOutput(this, 'UserPoolId', {
      value: userPool.userPoolId,
      description: 'Cognito User Pool ID',
      exportName: 'BankingUserPoolId'
    });

    new cdk.CfnOutput(this, 'UserPoolClientId', {
      value: userPoolClient.userPoolClientId,
      description: 'Cognito User Pool Client ID',
      exportName: 'BankingUserPoolClientId'
    });

    new cdk.CfnOutput(this, 'APIKeyId', {
      value: partnerApiKey.keyId,
      description: 'Partner API Key ID'
    });
  }
}
```

---

### 📊 Request/Response Transformation with VTL

```velocity
## Request Mapping Template (VTL)
## Transform API Gateway request to backend format

{
  "accountId": "$input.params('accountId')",
  "amount": $input.json('$.amount'),
  "currency": "$input.json('$.currency')",
  "requestId": "$context.requestId",
  "sourceIp": "$context.identity.sourceIp",
  "userAgent": "$context.identity.userAgent",
  "timestamp": "$context.requestTimeEpoch",
  "userId": "$context.authorizer.claims.sub",
  "email": "$context.authorizer.claims.email"
}
```

```velocity
## Response Mapping Template (VTL)
## Transform backend response to API Gateway format

#set($inputRoot = $input.path('$'))
{
  "statusCode": 200,
  "headers": {
    "Content-Type": "application/json",
    "X-Request-Id": "$context.requestId",
    "Cache-Control": "max-age=300"
  },
  "body": {
    "success": true,
    "data": {
      "transactionId": "$inputRoot.transactionId",
      "accountId": "$inputRoot.accountId",
      "amount": $inputRoot.amount,
      "balance": $inputRoot.newBalance,
      "timestamp": "$inputRoot.timestamp"
    },
    "metadata": {
      "requestId": "$context.requestId",
      "processingTime": $context.integrationLatency
    }
  }
}
```

---

### 🎓 Interview Discussion Points - Q6

**Q1: Cognito vs Lambda Authorizer - when to use which?**

**A**:
- **Cognito**: Standard OAuth/OIDC, user management, built-in
- **Lambda**: Custom logic, third-party auth, complex rules

**Q2: How do you implement rate limiting?**

**A**:
- **Usage Plans**: Per-API key rate limits
- **Throttling**: Burst and steady-state limits per stage
- **WAF**: IP-based rate limiting

**Q3: What is VPC Link and when to use?**

**A**: Connects API Gateway to private resources (ECS, ALB) without exposing them publicly. Use for internal services that shouldn't be internet-accessible.

---

**End of File 3**

