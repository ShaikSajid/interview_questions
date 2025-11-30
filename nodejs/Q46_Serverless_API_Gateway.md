# Q46: Serverless - API Gateway Integration

## 📋 Summary
This question covers **AWS API Gateway** - the front door for serverless APIs. You'll learn REST vs HTTP APIs, Lambda integration patterns, request/response transformations, throttling, caching, and building production-ready banking APIs.

**Key Topics**:
- API Gateway fundamentals (REST API vs HTTP API)
- Lambda proxy vs custom integration
- Request validation and transformation
- Response mapping and error handling
- Throttling and rate limiting
- API caching strategies
- CORS configuration
- Authentication (IAM, Cognito, Custom authorizers)
- API stages and deployment
- Monitoring and logging

**Banking Use Cases**:
- Complete serverless banking API
- Account management endpoints
- Transaction processing API
- Secure payment gateway
- Rate-limited public API

---

## 🎯 Understanding API Gateway

### What is API Gateway?

**API Gateway** is a fully managed service that acts as the "front door" for serverless applications:

```
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway Architecture                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Client (Web/Mobile)                                        │
│         │                                                    │
│         │ HTTPS Request                                     │
│         ▼                                                    │
│  ┌──────────────────────┐                                  │
│  │    API Gateway        │                                  │
│  │  ┌────────────────┐  │                                  │
│  │  │ Authentication │  │ (IAM, Cognito, Custom)          │
│  │  └────────┬───────┘  │                                  │
│  │  ┌────────▼───────┐  │                                  │
│  │  │   Validation   │  │ (Request validation)            │
│  │  └────────┬───────┘  │                                  │
│  │  ┌────────▼───────┐  │                                  │
│  │  │  Throttling    │  │ (Rate limiting)                 │
│  │  └────────┬───────┘  │                                  │
│  │  ┌────────▼───────┐  │                                  │
│  │  │     Cache      │  │ (Optional)                       │
│  │  └────────┬───────┘  │                                  │
│  │  ┌────────▼───────┐  │                                  │
│  │  │ Transformation │  │ (Request mapping)               │
│  │  └────────┬───────┘  │                                  │
│  └───────────┼──────────┘                                  │
│              │                                              │
│              │ Invoke                                       │
│              ▼                                              │
│  ┌──────────────────────┐                                  │
│  │   Lambda Function     │                                  │
│  │   (Business Logic)    │                                  │
│  └──────────┬────────────┘                                  │
│              │                                              │
│              │ Response                                     │
│              ▼                                              │
│  ┌──────────────────────┐                                  │
│  │   API Gateway         │                                  │
│  │   (Response mapping)  │                                  │
│  └──────────┬────────────┘                                  │
│              │                                              │
│              ▼                                              │
│  Client (JSON Response)                                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### REST API vs HTTP API

| Feature | REST API | HTTP API |
|---------|----------|----------|
| **Price** | $3.50/million | $1.00/million |
| **Latency** | Higher | Lower (up to 60% faster) |
| **Caching** | ✅ Built-in | ❌ Not supported |
| **Request Validation** | ✅ Built-in | ⚠️ Manual |
| **WAF Integration** | ✅ Yes | ❌ No |
| **Resource Policies** | ✅ Yes | ❌ No |
| **Usage Plans** | ✅ Yes | ❌ No |
| **Custom Authorizers** | ✅ Lambda, Cognito | ✅ Lambda, Cognito, JWT |
| **Use Case** | Enterprise, complex | Simple, cost-sensitive |

### Lambda Integration Types

**1. Lambda Proxy Integration** (Recommended)
```javascript
// Lambda receives full request details
{
  "resource": "/accounts/{id}",
  "path": "/accounts/123",
  "httpMethod": "GET",
  "headers": { "Authorization": "Bearer ..." },
  "queryStringParameters": { "details": "true" },
  "pathParameters": { "id": "123" },
  "body": null
}

// Lambda returns full HTTP response
{
  "statusCode": 200,
  "headers": { "Content-Type": "application/json" },
  "body": "{\"accountId\":\"123\",\"balance\":5000}"
}
```

**2. Custom Integration**
```javascript
// API Gateway transforms request before Lambda
// Lambda receives transformed data
{
  "accountId": "123",
  "includeDetails": true
}

// Lambda returns business object
{
  "accountId": "123",
  "balance": 5000
}

// API Gateway transforms to HTTP response
```

---

## 💡 Example 1: Complete Serverless Banking API

Production-ready banking API with account management, transactions, and payments.

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│             Serverless Banking API Architecture              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Client ──> CloudFront ──> API Gateway (REST API)          │
│                               │                              │
│                               ├─> Cognito User Pool         │
│                               │   (Authentication)           │
│                               │                              │
│                               ├─> /accounts                  │
│                               │   ├─> GET    (List)         │
│                               │   ├─> POST   (Create)       │
│                               │   └─> Lambda Function       │
│                               │                              │
│                               ├─> /accounts/{id}            │
│                               │   ├─> GET    (Details)      │
│                               │   ├─> PUT    (Update)       │
│                               │   └─> Lambda Function       │
│                               │                              │
│                               ├─> /transactions             │
│                               │   ├─> GET    (List)         │
│                               │   ├─> POST   (Create)       │
│                               │   └─> Lambda Function       │
│                               │                              │
│                               └─> /payments                  │
│                                   ├─> POST   (Process)      │
│                                   └─> Lambda Function       │
│                                                              │
│  Backend:                                                    │
│  - DynamoDB (Accounts, Transactions)                        │
│  - SQS (Async processing)                                   │
│  - SNS (Notifications)                                      │
│  - CloudWatch (Logging, Metrics)                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Setup

```bash
# Install Serverless Framework
npm install -g serverless
serverless --version

# Create project
mkdir banking-api
cd banking-api
npm init -y

# Install dependencies
npm install aws-sdk uuid joi
npm install --save-dev serverless-offline
```

### Implementation

#### serverless.yml

```yaml
service: banking-api

provider:
  name: aws
  runtime: nodejs18.x
  region: us-east-1
  stage: ${opt:stage, 'dev'}
  
  # IAM Permissions
  iamRoleStatements:
    - Effect: Allow
      Action:
        - dynamodb:Query
        - dynamodb:Scan
        - dynamodb:GetItem
        - dynamodb:PutItem
        - dynamodb:UpdateItem
        - dynamodb:DeleteItem
      Resource:
        - !GetAtt AccountsTable.Arn
        - !GetAtt TransactionsTable.Arn
        - !Sub "${TransactionsTable.Arn}/index/*"
    - Effect: Allow
      Action:
        - sns:Publish
      Resource: !Ref NotificationTopic
    - Effect: Allow
      Action:
        - sqs:SendMessage
      Resource: !GetAtt ProcessingQueue.Arn
  
  # Environment variables
  environment:
    ACCOUNTS_TABLE: !Ref AccountsTable
    TRANSACTIONS_TABLE: !Ref TransactionsTable
    NOTIFICATION_TOPIC_ARN: !Ref NotificationTopic
    PROCESSING_QUEUE_URL: !Ref ProcessingQueue
    STAGE: ${self:provider.stage}

# API Gateway Configuration
apiGateway:
  # Enable API caching (5 minutes)
  cacheClusterEnabled: true
  cacheClusterSize: '0.5'
  
  # Throttling
  throttle:
    burstLimit: 200
    rateLimit: 100
  
  # Usage plan
  usagePlan:
    quota:
      limit: 10000
      period: DAY
    throttle:
      burstLimit: 200
      rateLimit: 100

functions:
  # Accounts API
  listAccounts:
    handler: handlers/accounts.list
    events:
      - http:
          path: accounts
          method: get
          cors: true
          authorizer:
            type: COGNITO_USER_POOLS
            authorizerId: !Ref CognitoAuthorizer
          caching:
            enabled: true
            ttlInSeconds: 300
  
  getAccount:
    handler: handlers/accounts.get
    events:
      - http:
          path: accounts/{id}
          method: get
          cors: true
          authorizer:
            type: COGNITO_USER_POOLS
            authorizerId: !Ref CognitoAuthorizer
          request:
            parameters:
              paths:
                id: true
          caching:
            enabled: true
            ttlInSeconds: 300
            cacheKeyParameters:
              - name: request.path.id
  
  createAccount:
    handler: handlers/accounts.create
    events:
      - http:
          path: accounts
          method: post
          cors: true
          authorizer:
            type: COGNITO_USER_POOLS
            authorizerId: !Ref CognitoAuthorizer
          request:
            schemas:
              application/json: ${file(schemas/account-create.json)}
  
  updateAccount:
    handler: handlers/accounts.update
    events:
      - http:
          path: accounts/{id}
          method: put
          cors: true
          authorizer:
            type: COGNITO_USER_POOLS
            authorizerId: !Ref CognitoAuthorizer
  
  # Transactions API
  listTransactions:
    handler: handlers/transactions.list
    events:
      - http:
          path: transactions
          method: get
          cors: true
          authorizer:
            type: COGNITO_USER_POOLS
            authorizerId: !Ref CognitoAuthorizer
          request:
            parameters:
              querystrings:
                accountId: true
                startDate: false
                endDate: false
  
  createTransaction:
    handler: handlers/transactions.create
    events:
      - http:
          path: transactions
          method: post
          cors: true
          authorizer:
            type: COGNITO_USER_POOLS
            authorizerId: !Ref CognitoAuthorizer
  
  # Payments API
  processPayment:
    handler: handlers/payments.process
    timeout: 30
    events:
      - http:
          path: payments
          method: post
          cors: true
          authorizer:
            type: COGNITO_USER_POOLS
            authorizerId: !Ref CognitoAuthorizer

# CloudFormation Resources
resources:
  Resources:
    # DynamoDB Tables
    AccountsTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${self:service}-accounts-${self:provider.stage}
        BillingMode: PAY_PER_REQUEST
        AttributeDefinitions:
          - AttributeName: accountId
            AttributeType: S
          - AttributeName: userId
            AttributeType: S
        KeySchema:
          - AttributeName: accountId
            KeyType: HASH
        GlobalSecondaryIndexes:
          - IndexName: UserIdIndex
            KeySchema:
              - AttributeName: userId
                KeyType: HASH
            Projection:
              ProjectionType: ALL
    
    TransactionsTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${self:service}-transactions-${self:provider.stage}
        BillingMode: PAY_PER_REQUEST
        AttributeDefinitions:
          - AttributeName: transactionId
            AttributeType: S
          - AttributeName: accountId
            AttributeType: S
          - AttributeName: timestamp
            AttributeType: N
        KeySchema:
          - AttributeName: transactionId
            KeyType: HASH
        GlobalSecondaryIndexes:
          - IndexName: AccountTransactionsIndex
            KeySchema:
              - AttributeName: accountId
                KeyType: HASH
              - AttributeName: timestamp
                KeyType: RANGE
            Projection:
              ProjectionType: ALL
    
    # SNS Topic
    NotificationTopic:
      Type: AWS::SNS::Topic
      Properties:
        TopicName: ${self:service}-notifications-${self:provider.stage}
    
    # SQS Queue
    ProcessingQueue:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: ${self:service}-processing-${self:provider.stage}
        VisibilityTimeout: 300
    
    # Cognito User Pool
    CognitoUserPool:
      Type: AWS::Cognito::UserPool
      Properties:
        UserPoolName: ${self:service}-users-${self:provider.stage}
        AutoVerifiedAttributes:
          - email
        Schema:
          - Name: email
            Required: true
            Mutable: false
    
    CognitoUserPoolClient:
      Type: AWS::Cognito::UserPoolClient
      Properties:
        ClientName: ${self:service}-client-${self:provider.stage}
        UserPoolId: !Ref CognitoUserPool
        GenerateSecret: false
        ExplicitAuthFlows:
          - ALLOW_USER_PASSWORD_AUTH
          - ALLOW_REFRESH_TOKEN_AUTH
    
    # API Gateway Authorizer
    CognitoAuthorizer:
      Type: AWS::ApiGateway::Authorizer
      Properties:
        Name: CognitoAuthorizer
        Type: COGNITO_USER_POOLS
        IdentitySource: method.request.header.Authorization
        RestApiId: !Ref ApiGatewayRestApi
        ProviderARNs:
          - !GetAtt CognitoUserPool.Arn

plugins:
  - serverless-offline

# Outputs
outputs:
  ApiUrl:
    Description: API Gateway endpoint URL
    Value: !Sub https://${ApiGatewayRestApi}.execute-api.${AWS::Region}.amazonaws.com/${self:provider.stage}
  
  UserPoolId:
    Description: Cognito User Pool ID
    Value: !Ref CognitoUserPool
  
  UserPoolClientId:
    Description: Cognito User Pool Client ID
    Value: !Ref CognitoUserPoolClient
```

#### handlers/accounts.js

```javascript
/**
 * Account Management Handlers
 */

const AWS = require('aws-sdk');
const { v4: uuidv4 } = require('uuid');
const Joi = require('joi');

const dynamodb = new AWS.DynamoDB.DocumentClient();
const ACCOUNTS_TABLE = process.env.ACCOUNTS_TABLE;

/**
 * List all accounts for authenticated user
 */
module.exports.list = async (event) => {
  try {
    // Extract user from Cognito authorizer
    const userId = event.requestContext.authorizer.claims.sub;
    
    console.log('Listing accounts for user:', userId);
    
    const params = {
      TableName: ACCOUNTS_TABLE,
      IndexName: 'UserIdIndex',
      KeyConditionExpression: 'userId = :userId',
      ExpressionAttributeValues: {
        ':userId': userId
      }
    };
    
    const result = await dynamodb.query(params).promise();
    
    return {
      statusCode: 200,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*',
        'Cache-Control': 'max-age=300' // 5 minutes
      },
      body: JSON.stringify({
        accounts: result.Items,
        count: result.Count
      })
    };
    
  } catch (error) {
    console.error('Error listing accounts:', error);
    return errorResponse(500, 'Failed to list accounts');
  }
};

/**
 * Get account details
 */
module.exports.get = async (event) => {
  try {
    const accountId = event.pathParameters.id;
    const userId = event.requestContext.authorizer.claims.sub;
    
    console.log('Getting account:', accountId);
    
    const params = {
      TableName: ACCOUNTS_TABLE,
      Key: { accountId }
    };
    
    const result = await dynamodb.get(params).promise();
    
    if (!result.Item) {
      return errorResponse(404, 'Account not found');
    }
    
    // Verify ownership
    if (result.Item.userId !== userId) {
      return errorResponse(403, 'Access denied');
    }
    
    return {
      statusCode: 200,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*',
        'Cache-Control': 'max-age=300'
      },
      body: JSON.stringify(result.Item)
    };
    
  } catch (error) {
    console.error('Error getting account:', error);
    return errorResponse(500, 'Failed to get account');
  }
};

/**
 * Create new account
 */
module.exports.create = async (event) => {
  try {
    const userId = event.requestContext.authorizer.claims.sub;
    const data = JSON.parse(event.body);
    
    // Validate input
    const schema = Joi.object({
      accountType: Joi.string().valid('checking', 'savings', 'credit').required(),
      accountName: Joi.string().min(3).max(50).required(),
      currency: Joi.string().length(3).default('USD'),
      initialDeposit: Joi.number().min(0).default(0)
    });
    
    const { error, value } = schema.validate(data);
    if (error) {
      return errorResponse(400, error.details[0].message);
    }
    
    // Create account
    const account = {
      accountId: uuidv4(),
      userId,
      accountType: value.accountType,
      accountName: value.accountName,
      accountNumber: generateAccountNumber(),
      currency: value.currency,
      balance: value.initialDeposit,
      status: 'active',
      createdAt: Date.now(),
      updatedAt: Date.now()
    };
    
    await dynamodb.put({
      TableName: ACCOUNTS_TABLE,
      Item: account
    }).promise();
    
    console.log('Account created:', account.accountId);
    
    return {
      statusCode: 201,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
      },
      body: JSON.stringify(account)
    };
    
  } catch (error) {
    console.error('Error creating account:', error);
    return errorResponse(500, 'Failed to create account');
  }
};

/**
 * Update account
 */
module.exports.update = async (event) => {
  try {
    const accountId = event.pathParameters.id;
    const userId = event.requestContext.authorizer.claims.sub;
    const data = JSON.parse(event.body);
    
    // Verify ownership
    const existing = await dynamodb.get({
      TableName: ACCOUNTS_TABLE,
      Key: { accountId }
    }).promise();
    
    if (!existing.Item) {
      return errorResponse(404, 'Account not found');
    }
    
    if (existing.Item.userId !== userId) {
      return errorResponse(403, 'Access denied');
    }
    
    // Validate update
    const schema = Joi.object({
      accountName: Joi.string().min(3).max(50),
      status: Joi.string().valid('active', 'frozen', 'closed')
    });
    
    const { error, value } = schema.validate(data);
    if (error) {
      return errorResponse(400, error.details[0].message);
    }
    
    // Update account
    const updateExpr = [];
    const exprAttrNames = {};
    const exprAttrValues = {};
    
    if (value.accountName) {
      updateExpr.push('#accountName = :accountName');
      exprAttrNames['#accountName'] = 'accountName';
      exprAttrValues[':accountName'] = value.accountName;
    }
    
    if (value.status) {
      updateExpr.push('#status = :status');
      exprAttrNames['#status'] = 'status';
      exprAttrValues[':status'] = value.status;
    }
    
    updateExpr.push('#updatedAt = :updatedAt');
    exprAttrNames['#updatedAt'] = 'updatedAt';
    exprAttrValues[':updatedAt'] = Date.now();
    
    const result = await dynamodb.update({
      TableName: ACCOUNTS_TABLE,
      Key: { accountId },
      UpdateExpression: `SET ${updateExpr.join(', ')}`,
      ExpressionAttributeNames: exprAttrNames,
      ExpressionAttributeValues: exprAttrValues,
      ReturnValues: 'ALL_NEW'
    }).promise();
    
    return {
      statusCode: 200,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
      },
      body: JSON.stringify(result.Attributes)
    };
    
  } catch (error) {
    console.error('Error updating account:', error);
    return errorResponse(500, 'Failed to update account');
  }
};

/**
 * Helper functions
 */
function generateAccountNumber() {
  return Math.floor(1000000000 + Math.random() * 9000000000).toString();
}

function errorResponse(statusCode, message) {
  return {
    statusCode,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify({ error: message })
  };
}
```

#### handlers/transactions.js

```javascript
/**
 * Transaction Handlers
 */

const AWS = require('aws-sdk');
const { v4: uuidv4 } = require('uuid');
const Joi = require('joi');

const dynamodb = new AWS.DynamoDB.DocumentClient();
const ACCOUNTS_TABLE = process.env.ACCOUNTS_TABLE;
const TRANSACTIONS_TABLE = process.env.TRANSACTIONS_TABLE;

/**
 * List transactions for account
 */
module.exports.list = async (event) => {
  try {
    const userId = event.requestContext.authorizer.claims.sub;
    const { accountId, startDate, endDate, limit = 50 } = event.queryStringParameters || {};
    
    if (!accountId) {
      return errorResponse(400, 'accountId query parameter required');
    }
    
    // Verify account ownership
    const account = await getAccount(accountId);
    if (!account || account.userId !== userId) {
      return errorResponse(403, 'Access denied');
    }
    
    console.log('Listing transactions for account:', accountId);
    
    const params = {
      TableName: TRANSACTIONS_TABLE,
      IndexName: 'AccountTransactionsIndex',
      KeyConditionExpression: 'accountId = :accountId',
      ExpressionAttributeValues: {
        ':accountId': accountId
      },
      Limit: parseInt(limit),
      ScanIndexForward: false // Newest first
    };
    
    // Add date range filter
    if (startDate && endDate) {
      params.KeyConditionExpression += ' AND #timestamp BETWEEN :startDate AND :endDate';
      params.ExpressionAttributeNames = { '#timestamp': 'timestamp' };
      params.ExpressionAttributeValues[':startDate'] = new Date(startDate).getTime();
      params.ExpressionAttributeValues[':endDate'] = new Date(endDate).getTime();
    }
    
    const result = await dynamodb.query(params).promise();
    
    return {
      statusCode: 200,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
      },
      body: JSON.stringify({
        transactions: result.Items,
        count: result.Count,
        lastEvaluatedKey: result.LastEvaluatedKey
      })
    };
    
  } catch (error) {
    console.error('Error listing transactions:', error);
    return errorResponse(500, 'Failed to list transactions');
  }
};

/**
 * Create transaction
 */
module.exports.create = async (event) => {
  try {
    const userId = event.requestContext.authorizer.claims.sub;
    const data = JSON.parse(event.body);
    
    // Validate input
    const schema = Joi.object({
      accountId: Joi.string().uuid().required(),
      type: Joi.string().valid('deposit', 'withdrawal', 'transfer', 'payment').required(),
      amount: Joi.number().positive().required(),
      description: Joi.string().max(200),
      metadata: Joi.object()
    });
    
    const { error, value } = schema.validate(data);
    if (error) {
      return errorResponse(400, error.details[0].message);
    }
    
    // Verify account ownership
    const account = await getAccount(value.accountId);
    if (!account || account.userId !== userId) {
      return errorResponse(403, 'Access denied');
    }
    
    // Check balance for withdrawals
    if (['withdrawal', 'transfer', 'payment'].includes(value.type)) {
      if (account.balance < value.amount) {
        return errorResponse(400, 'Insufficient funds');
      }
    }
    
    // Create transaction
    const transaction = {
      transactionId: uuidv4(),
      accountId: value.accountId,
      type: value.type,
      amount: value.amount,
      description: value.description || '',
      status: 'completed',
      timestamp: Date.now(),
      metadata: value.metadata || {}
    };
    
    // Update account balance
    const balanceChange = ['deposit'].includes(value.type) ? value.amount : -value.amount;
    
    await dynamodb.transactWrite({
      TransactItems: [
        {
          Put: {
            TableName: TRANSACTIONS_TABLE,
            Item: transaction
          }
        },
        {
          Update: {
            TableName: ACCOUNTS_TABLE,
            Key: { accountId: value.accountId },
            UpdateExpression: 'SET balance = balance + :change, updatedAt = :now',
            ExpressionAttributeValues: {
              ':change': balanceChange,
              ':now': Date.now()
            }
          }
        }
      ]
    }).promise();
    
    console.log('Transaction created:', transaction.transactionId);
    
    return {
      statusCode: 201,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
      },
      body: JSON.stringify(transaction)
    };
    
  } catch (error) {
    console.error('Error creating transaction:', error);
    return errorResponse(500, 'Failed to create transaction');
  }
};

async function getAccount(accountId) {
  const result = await dynamodb.get({
    TableName: ACCOUNTS_TABLE,
    Key: { accountId }
  }).promise();
  return result.Item;
}

function errorResponse(statusCode, message) {
  return {
    statusCode,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify({ error: message })
  };
}
```

#### handlers/payments.js

```javascript
/**
 * Payment Processing Handler
 */

const AWS = require('aws-sdk');
const { v4: uuidv4 } = require('uuid');
const Joi = require('joi');

const dynamodb = new AWS.DynamoDB.DocumentClient();
const sqs = new AWS.SQS();
const sns = new AWS.SNS();

const ACCOUNTS_TABLE = process.env.ACCOUNTS_TABLE;
const TRANSACTIONS_TABLE = process.env.TRANSACTIONS_TABLE;
const PROCESSING_QUEUE_URL = process.env.PROCESSING_QUEUE_URL;
const NOTIFICATION_TOPIC_ARN = process.env.NOTIFICATION_TOPIC_ARN;

/**
 * Process payment
 */
module.exports.process = async (event) => {
  try {
    const userId = event.requestContext.authorizer.claims.sub;
    const data = JSON.parse(event.body);
    
    // Validate input
    const schema = Joi.object({
      accountId: Joi.string().uuid().required(),
      amount: Joi.number().positive().required(),
      payee: Joi.string().required(),
      payeeAccount: Joi.string().required(),
      description: Joi.string().max(200)
    });
    
    const { error, value } = schema.validate(data);
    if (error) {
      return errorResponse(400, error.details[0].message);
    }
    
    // Verify account ownership and balance
    const account = await getAccount(value.accountId);
    if (!account || account.userId !== userId) {
      return errorResponse(403, 'Access denied');
    }
    
    if (account.balance < value.amount) {
      return errorResponse(400, 'Insufficient funds');
    }
    
    if (account.status !== 'active') {
      return errorResponse(400, `Account is ${account.status}`);
    }
    
    // Create payment transaction
    const payment = {
      transactionId: uuidv4(),
      accountId: value.accountId,
      type: 'payment',
      amount: value.amount,
      payee: value.payee,
      payeeAccount: value.payeeAccount,
      description: value.description || '',
      status: 'pending',
      timestamp: Date.now(),
      userId
    };
    
    // Deduct amount and create transaction record
    await dynamodb.transactWrite({
      TransactItems: [
        {
          Put: {
            TableName: TRANSACTIONS_TABLE,
            Item: payment
          }
        },
        {
          Update: {
            TableName: ACCOUNTS_TABLE,
            Key: { accountId: value.accountId },
            UpdateExpression: 'SET balance = balance - :amount, updatedAt = :now',
            ConditionExpression: 'balance >= :amount AND #status = :active',
            ExpressionAttributeNames: {
              '#status': 'status'
            },
            ExpressionAttributeValues: {
              ':amount': value.amount,
              ':now': Date.now(),
              ':active': 'active'
            }
          }
        }
      ]
    }).promise();
    
    // Send to processing queue for async processing
    await sqs.sendMessage({
      QueueUrl: PROCESSING_QUEUE_URL,
      MessageBody: JSON.stringify(payment),
      MessageAttributes: {
        transactionType: {
          DataType: 'String',
          StringValue: 'payment'
        }
      }
    }).promise();
    
    // Send notification
    await sns.publish({
      TopicArn: NOTIFICATION_TOPIC_ARN,
      Subject: 'Payment Initiated',
      Message: JSON.stringify({
        userId,
        accountId: value.accountId,
        transactionId: payment.transactionId,
        amount: value.amount,
        payee: value.payee
      })
    }).promise();
    
    console.log('Payment initiated:', payment.transactionId);
    
    return {
      statusCode: 202, // Accepted
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
      },
      body: JSON.stringify({
        transactionId: payment.transactionId,
        status: 'pending',
        message: 'Payment is being processed'
      })
    };
    
  } catch (error) {
    console.error('Error processing payment:', error);
    
    if (error.code === 'TransactionCanceledException') {
      return errorResponse(400, 'Payment failed: insufficient funds or account inactive');
    }
    
    return errorResponse(500, 'Failed to process payment');
  }
};

async function getAccount(accountId) {
  const result = await dynamodb.get({
    TableName: ACCOUNTS_TABLE,
    Key: { accountId }
  }).promise();
  return result.Item;
}

function errorResponse(statusCode, message) {
  return {
    statusCode,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify({ error: message })
  };
}
```

### Request Validation Schema

```json
// schemas/account-create.json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "accountType": {
      "type": "string",
      "enum": ["checking", "savings", "credit"]
    },
    "accountName": {
      "type": "string",
      "minLength": 3,
      "maxLength": 50
    },
    "currency": {
      "type": "string",
      "pattern": "^[A-Z]{3}$"
    },
    "initialDeposit": {
      "type": "number",
      "minimum": 0
    }
  },
  "required": ["accountType", "accountName"]
}
```

### Deployment

```bash
# Deploy to dev
serverless deploy --stage dev

# Deploy to production
serverless deploy --stage prod

# Deploy single function
serverless deploy function -f createAccount

# View logs
serverless logs -f createAccount --tail

# Remove stack
serverless remove --stage dev
```

### Testing

```bash
# Create Cognito user
aws cognito-idp sign-up \
  --client-id YOUR_CLIENT_ID \
  --username user@example.com \
  --password SecurePassword123!

# Confirm user
aws cognito-idp admin-confirm-sign-up \
  --user-pool-id YOUR_USER_POOL_ID \
  --username user@example.com

# Authenticate
aws cognito-idp initiate-auth \
  --client-id YOUR_CLIENT_ID \
  --auth-flow USER_PASSWORD_AUTH \
  --auth-parameters USERNAME=user@example.com,PASSWORD=SecurePassword123!

# Use ID token in API requests
curl -X POST https://API_URL/dev/accounts \
  -H "Authorization: Bearer ID_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "accountType": "checking",
    "accountName": "My Checking Account",
    "initialDeposit": 1000
  }'

# Get accounts
curl -X GET https://API_URL/dev/accounts \
  -H "Authorization: Bearer ID_TOKEN"

# Create transaction
curl -X POST https://API_URL/dev/transactions \
  -H "Authorization: Bearer ID_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "accountId": "ACCOUNT_ID",
    "type": "deposit",
    "amount": 500,
    "description": "Salary deposit"
  }'
```

---

## 💡 Example 2: Advanced API Gateway Features

### Custom Authorizer (Lambda Authorizer)

```javascript
/**
 * Custom API Key Authorizer
 */

const AWS = require('aws-sdk');
const dynamodb = new AWS.DynamoDB.DocumentClient();

exports.handler = async (event) => {
  console.log('Authorizer event:', JSON.stringify(event));
  
  try {
    // Extract API key from header
    const apiKey = event.headers['X-API-Key'] || event.headers['x-api-key'];
    
    if (!apiKey) {
      throw new Error('Missing API key');
    }
    
    // Validate API key from DynamoDB
    const result = await dynamodb.get({
      TableName: 'ApiKeys',
      Key: { apiKey }
    }).promise();
    
    if (!result.Item || result.Item.status !== 'active') {
      throw new Error('Invalid or inactive API key');
    }
    
    // Check rate limit
    if (result.Item.requestCount >= result.Item.rateLimit) {
      throw new Error('Rate limit exceeded');
    }
    
    // Increment usage counter
    await dynamodb.update({
      TableName: 'ApiKeys',
      Key: { apiKey },
      UpdateExpression: 'SET requestCount = requestCount + :inc',
      ExpressionAttributeValues: { ':inc': 1 }
    }).promise();
    
    // Return IAM policy
    return generatePolicy(result.Item.clientId, 'Allow', event.methodArn, {
      clientId: result.Item.clientId,
      tier: result.Item.tier,
      remainingRequests: result.Item.rateLimit - result.Item.requestCount
    });
    
  } catch (error) {
    console.error('Authorization failed:', error);
    throw new Error('Unauthorized');
  }
};

function generatePolicy(principalId, effect, resource, context) {
  return {
    principalId,
    policyDocument: {
      Version: '2012-10-17',
      Statement: [{
        Action: 'execute-api:Invoke',
        Effect: effect,
        Resource: resource
      }]
    },
    context // Available in Lambda as event.requestContext.authorizer
  };
}
```

### Request/Response Transformation

```javascript
// API Gateway VTL (Velocity Template Language) for request transformation

// Input: { "account": "123", "amt": 100 }
// Transform to: { "accountId": "123", "amount": 100 }

#set($inputRoot = $input.path('$'))
{
  "accountId": "$inputRoot.account",
  "amount": $inputRoot.amt,
  "requestTime": "$context.requestTime",
  "sourceIp": "$context.identity.sourceIp"
}

// Response transformation (add headers, wrap response)
#set($inputRoot = $input.path('$'))
{
  "success": true,
  "data": $inputRoot,
  "timestamp": "$context.requestTime",
  "requestId": "$context.requestId"
}
```

### Rate Limiting & Throttling

```yaml
# serverless.yml
functions:
  publicApi:
    handler: handlers/public.get
    events:
      - http:
          path: public/data
          method: get
          # Per-method throttling
          throttle:
            burstLimit: 50   # Max concurrent requests
            rateLimit: 25    # Requests per second
```

### API Caching

```yaml
functions:
  cachedEndpoint:
    handler: handlers/data.get
    events:
      - http:
          path: data/{id}
          method: get
          caching:
            enabled: true
            ttlInSeconds: 300  # 5 minutes
            # Cache per path parameter
            cacheKeyParameters:
              - name: request.path.id
            # Cache per query string
            cacheKeyParameters:
              - name: request.querystring.format
```

### CORS Configuration

```yaml
functions:
  api:
    handler: handler.main
    events:
      - http:
          path: api/resource
          method: post
          cors:
            origin: 'https://myapp.com'
            headers:
              - Content-Type
              - X-Amz-Date
              - Authorization
              - X-Api-Key
            allowCredentials: true
            maxAge: 3600
```

---

## 🎯 API Gateway Best Practices

### 1. API Versioning

```yaml
# Method 1: Path-based versioning
functions:
  v1Api:
    handler: handlers/v1/accounts.list
    events:
      - http:
          path: v1/accounts
          method: get
  
  v2Api:
    handler: handlers/v2/accounts.list
    events:
      - http:
          path: v2/accounts
          method: get

# Method 2: Header-based versioning (custom authorizer)
# Header: X-API-Version: 2.0

# Method 3: Separate stages
# https://api.example.com/v1/accounts
# https://api.example.com/v2/accounts
```

### 2. Error Handling

```javascript
// Standardized error responses
class ApiError extends Error {
  constructor(statusCode, message, code) {
    super(message);
    this.statusCode = statusCode;
    this.code = code;
  }
}

function errorResponse(error) {
  const statusCode = error.statusCode || 500;
  const body = {
    error: {
      code: error.code || 'INTERNAL_ERROR',
      message: error.message,
      timestamp: new Date().toISOString(),
      requestId: context.requestId
    }
  };
  
  // Don't expose internal errors in production
  if (statusCode === 500 && process.env.STAGE === 'prod') {
    body.error.message = 'Internal server error';
  }
  
  return {
    statusCode,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify(body)
  };
}
```

### 3. Request Validation

```javascript
// Input validation middleware
function validateRequest(schema) {
  return async (event) => {
    const data = JSON.parse(event.body);
    const { error, value } = schema.validate(data, {
      abortEarly: false,
      stripUnknown: true
    });
    
    if (error) {
      const errors = error.details.map(d => ({
        field: d.path.join('.'),
        message: d.message
      }));
      
      return {
        statusCode: 400,
        body: JSON.stringify({
          error: 'Validation failed',
          errors
        })
      };
    }
    
    event.validatedBody = value;
  };
}
```

### 4. Monitoring & Logging

```javascript
// Enhanced logging
class Logger {
  constructor(context) {
    this.requestId = context.requestId;
    this.functionName = context.functionName;
  }
  
  log(level, message, meta = {}) {
    console.log(JSON.stringify({
      timestamp: new Date().toISOString(),
      level,
      requestId: this.requestId,
      functionName: this.functionName,
      message,
      ...meta
    }));
  }
  
  info(message, meta) { this.log('INFO', message, meta); }
  warn(message, meta) { this.log('WARN', message, meta); }
  error(message, meta) { this.log('ERROR', message, meta); }
}

// Usage
const logger = new Logger(context);
logger.info('Processing request', { accountId, amount });

// Custom metrics
const cloudwatch = new AWS.CloudWatch();

async function recordMetric(name, value, unit = 'Count') {
  await cloudwatch.putMetricData({
    Namespace: 'BankingAPI',
    MetricData: [{
      MetricName: name,
      Value: value,
      Unit: unit,
      Dimensions: [
        { Name: 'Environment', Value: process.env.STAGE },
        { Name: 'FunctionName', Value: context.functionName }
      ]
    }]
  }).promise();
}
```

---

## 📚 Common Interview Questions

### Q1: REST API vs HTTP API - when to use which?

**Answer**:
- **Use REST API** when you need: caching, request validation, WAF integration, usage plans, resource policies
- **Use HTTP API** when you need: lower cost (70% cheaper), lower latency (60% faster), simpler use case, JWT authorizers

### Q2: How does API Gateway handle throttling?

**Answer**:
API Gateway throttles at multiple levels:
1. **Account level**: Default 10,000 RPS across all APIs
2. **Stage level**: Configurable per stage
3. **Method level**: Per-endpoint throttling
4. **Usage plan**: Per-client rate limiting

Returns 429 (Too Many Requests) when exceeded.

### Q3: How do you secure API Gateway?

**Answer**:
1. **Authentication**: Cognito, Lambda authorizers, IAM
2. **Authorization**: Resource policies, IAM policies
3. **Network**: VPC links for private resources
4. **Data**: TLS 1.2+, client certificates
5. **WAF**: SQL injection, XSS protection
6. **Throttling**: Rate limiting per client

### Q4: Explain API Gateway caching

**Answer**:
```javascript
// Cache configuration
{
  cacheClusterEnabled: true,
  cacheClusterSize: '0.5', // 0.5GB to 237GB
  cacheTtlInSeconds: 300,
  cacheKeyParameters: [
    'method.request.path.id',
    'method.request.querystring.format'
  ]
}

// Cache invalidation
await apigateway.flushStageCache({
  restApiId: 'abc123',
  stageName: 'prod'
}).promise();

// Per-request cache override
headers: {
  'Cache-Control': 'max-age=0' // Bypass cache
}
```

### Q5: How do you version APIs in API Gateway?

**Answer**:
Three approaches:
1. **Path**: `/v1/resource`, `/v2/resource`
2. **Header**: `X-API-Version: 2.0`
3. **Stages**: Separate stage per version

Recommendation: Path-based for public APIs (explicit), header-based for internal (flexible).

---

## ✅ Summary & Key Takeaways

### Core Concepts

1. **API Gateway**: Fully managed API front door for serverless
2. **Integration Types**: Proxy (pass-through) vs Custom (transformation)
3. **REST vs HTTP**: REST for features, HTTP for cost/performance
4. **Stages**: Dev, staging, prod environments

### Security

1. **Cognito**: User pool authentication
2. **Lambda Authorizers**: Custom API key/token validation
3. **IAM**: Programmatic access control
4. **Resource Policies**: IP whitelisting, VPC access

### Performance

1. **Caching**: Reduce backend load (5-300 seconds TTL)
2. **Throttling**: Protect backend from overload
3. **Usage Plans**: Per-client rate limiting
4. **CDN**: CloudFront for global distribution

### Banking API Checklist

```yaml
✅ Production Readiness:
  - [ ] Cognito authentication
  - [ ] Request validation (schemas)
  - [ ] Error handling (standardized responses)
  - [ ] Logging (structured JSON)
  - [ ] Metrics (CloudWatch custom metrics)
  - [ ] Caching (frequently accessed data)
  - [ ] Throttling (rate limiting)
  - [ ] CORS (frontend integration)
  - [ ] API versioning
  - [ ] Documentation (OpenAPI/Swagger)
  - [ ] Monitoring (X-Ray tracing)
  - [ ] Alerting (CloudWatch alarms)
```

### Common Patterns

1. **Account Management**: CRUD operations with ownership validation
2. **Transaction Processing**: Atomic updates with DynamoDB transactions
3. **Payment Processing**: Async with SQS, SNS notifications
4. **Fraud Detection**: Real-time validation in authorizers

---

**Status**: ✅ Complete with production-ready serverless banking API, API Gateway features, and security best practices!
