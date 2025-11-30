# Q45: Serverless Computing - AWS Lambda

## 📋 Summary
This question covers **AWS Lambda** and serverless architecture - building event-driven, auto-scaling applications without managing servers. You'll learn Lambda fundamentals, cold start optimization, event sources, and production-ready banking applications.

**Key Topics**:
- Serverless architecture fundamentals
- AWS Lambda function structure and lifecycle
- Cold starts and optimization techniques
- Event sources (API Gateway, S3, DynamoDB, SQS)
- Environment variables and secrets management
- Lambda layers and dependencies
- Error handling and retries
- Monitoring with CloudWatch

**Banking Use Cases**:
- Transaction validation and fraud detection
- Payment processing webhooks
- Account statement generation
- Real-time notification system
- Automated compliance checks

---

## 🎯 Understanding Serverless & AWS Lambda

### What is Serverless?

**Serverless** doesn't mean "no servers" - it means you don't manage servers. The cloud provider handles:
- Server provisioning and scaling
- Operating system updates
- High availability
- Monitoring and logging

You only:
- Write code
- Deploy functions
- Pay for execution time

### AWS Lambda Overview

**Lambda** is AWS's Function-as-a-Service (FaaS) offering:

```
┌─────────────────────────────────────────────────────────────┐
│                    Lambda Function Lifecycle                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. INIT (Cold Start)                                       │
│     ├─ Load runtime (Node.js, Python, etc.)                │
│     ├─ Load function code                                   │
│     ├─ Run initialization code                              │
│     └─ Initialize connections (DB, external APIs)           │
│                                                              │
│  2. INVOKE                                                   │
│     ├─ Receive event                                        │
│     ├─ Execute handler function                             │
│     └─ Return response                                       │
│                                                              │
│  3. REUSE (Warm)                                            │
│     └─ Same container handles next invocation               │
│                                                              │
│  4. SHUTDOWN                                                 │
│     └─ Container terminated after inactivity                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Lambda Characteristics

| Feature | Details |
|---------|---------|
| **Max Execution** | 15 minutes |
| **Memory** | 128 MB - 10,240 MB |
| **Deployment Package** | 50 MB (zipped), 250 MB (unzipped) |
| **Concurrency** | 1,000 (default), can request increase |
| **Cold Start** | 100ms - 1s (depends on runtime, size) |
| **Pricing** | $0.20 per 1M requests + compute time |

### Cold Start vs Warm Start

```javascript
// COLD START (first invocation or after inactivity)
// Total time: 500ms-2000ms
// ├─ Container initialization: 200-800ms
// ├─ Runtime initialization: 100-400ms
// ├─ Code initialization: 100-300ms
// └─ Handler execution: 100-500ms

// WARM START (reuse container)
// Total time: 10ms-100ms
// └─ Handler execution: 10-100ms
```

---

## 📖 Lambda Function Structure

### Basic Lambda Handler

```javascript
// Traditional callback style (Node.js 12.x and earlier)
exports.handler = function(event, context, callback) {
  // event: input data
  // context: runtime information
  // callback: return response
  
  const response = {
    statusCode: 200,
    body: JSON.stringify({ message: 'Hello from Lambda!' })
  };
  
  callback(null, response);
};

// Async/await style (Node.js 14.x+, recommended)
exports.handler = async (event, context) => {
  // Return response directly
  return {
    statusCode: 200,
    body: JSON.stringify({ message: 'Hello from Lambda!' })
  };
};
```

### Event Object

Different event sources provide different event structures:

```javascript
// API Gateway event
{
  "httpMethod": "POST",
  "path": "/transactions",
  "headers": { "Authorization": "Bearer token..." },
  "body": "{\"amount\": 1000}",
  "queryStringParameters": { "id": "123" },
  "pathParameters": { "accountId": "ACC-001" }
}

// SQS event
{
  "Records": [
    {
      "messageId": "...",
      "body": "{\"transaction\": {...}}",
      "attributes": {...}
    }
  ]
}

// S3 event
{
  "Records": [
    {
      "eventName": "ObjectCreated:Put",
      "s3": {
        "bucket": { "name": "my-bucket" },
        "object": { "key": "file.pdf" }
      }
    }
  ]
}
```

### Context Object

```javascript
exports.handler = async (event, context) => {
  console.log('Request ID:', context.requestId);
  console.log('Function ARN:', context.invokedFunctionArn);
  console.log('Remaining time:', context.getRemainingTimeInMillis());
  console.log('Memory limit:', context.memoryLimitInMB);
  
  // ...
};
```

---

## 💡 Example 1: Transaction Validation Lambda

Complete serverless transaction validation system with fraud detection.

### Scenario
Build a Lambda function that:
- Validates incoming transactions
- Checks for fraud patterns
- Integrates with DynamoDB for account lookup
- Sends alerts via SNS
- Handles errors gracefully

### Setup

```bash
# Install dependencies
npm init -y
npm install aws-sdk

# Create Lambda deployment package
zip -r function.zip index.js node_modules/

# Deploy (using AWS CLI)
aws lambda create-function \
  --function-name transaction-validator \
  --runtime nodejs18.x \
  --handler index.handler \
  --role arn:aws:iam::ACCOUNT_ID:role/lambda-execution-role \
  --zip-file fileb://function.zip \
  --timeout 30 \
  --memory-size 512
```

### Implementation

```javascript
/**
 * Transaction Validation Lambda Function
 * Validates banking transactions with fraud detection
 */

const AWS = require('aws-sdk');

// Initialize AWS services outside handler (reused across invocations)
const dynamodb = new AWS.DynamoDB.DocumentClient();
const sns = new AWS.SNS();
const cloudwatch = new AWS.CloudWatch();

// Configuration from environment variables
const ACCOUNTS_TABLE = process.env.ACCOUNTS_TABLE || 'BankAccounts';
const ALERT_TOPIC_ARN = process.env.ALERT_TOPIC_ARN;
const MAX_TRANSACTION_AMOUNT = parseInt(process.env.MAX_TRANSACTION_AMOUNT || '10000');
const FRAUD_THRESHOLD = parseFloat(process.env.FRAUD_THRESHOLD || '0.8');

/**
 * Lambda Handler - Entry point
 */
exports.handler = async (event, context) => {
  console.log('Event:', JSON.stringify(event, null, 2));
  console.log('Context:', {
    requestId: context.requestId,
    functionName: context.functionName,
    memoryLimit: context.memoryLimitInMB,
    remainingTime: context.getRemainingTimeInMillis()
  });

  try {
    // Parse input (from API Gateway or direct invocation)
    const transaction = parseEvent(event);
    
    // Validate transaction structure
    validateTransactionStructure(transaction);
    
    // Lookup account details
    const account = await getAccount(transaction.accountId);
    
    // Validate business rules
    await validateBusinessRules(transaction, account);
    
    // Fraud detection
    const fraudScore = await checkFraud(transaction, account);
    
    if (fraudScore > FRAUD_THRESHOLD) {
      // High fraud risk - send alert and reject
      await sendFraudAlert(transaction, fraudScore);
      
      return buildResponse(403, {
        approved: false,
        reason: 'Transaction flagged as high-risk',
        fraudScore: fraudScore.toFixed(2)
      });
    }
    
    // Transaction approved
    await recordMetrics('TransactionApproved', 1);
    
    return buildResponse(200, {
      approved: true,
      transactionId: transaction.id,
      fraudScore: fraudScore.toFixed(2),
      timestamp: new Date().toISOString()
    });
    
  } catch (error) {
    console.error('Error processing transaction:', error);
    
    // Record error metric
    await recordMetrics('TransactionError', 1);
    
    // Determine if error is retryable
    if (isRetryable(error)) {
      throw error; // Lambda will retry
    }
    
    return buildResponse(400, {
      approved: false,
      error: error.message
    });
  }
};

/**
 * Parse event from different sources
 */
function parseEvent(event) {
  // API Gateway event
  if (event.body) {
    const body = typeof event.body === 'string' ? JSON.parse(event.body) : event.body;
    return {
      id: `TXN-${Date.now()}`,
      ...body,
      source: 'api-gateway'
    };
  }
  
  // SQS event
  if (event.Records && event.Records[0].eventSource === 'aws:sqs') {
    const message = JSON.parse(event.Records[0].body);
    return {
      id: event.Records[0].messageId,
      ...message,
      source: 'sqs'
    };
  }
  
  // Direct invocation
  return {
    id: `TXN-${Date.now()}`,
    ...event,
    source: 'direct'
  };
}

/**
 * Validate transaction structure
 */
function validateTransactionStructure(transaction) {
  const required = ['accountId', 'amount', 'type'];
  
  for (const field of required) {
    if (!transaction[field]) {
      throw new Error(`Missing required field: ${field}`);
    }
  }
  
  if (typeof transaction.amount !== 'number' || transaction.amount <= 0) {
    throw new Error('Invalid amount');
  }
  
  const validTypes = ['deposit', 'withdrawal', 'transfer', 'payment'];
  if (!validTypes.includes(transaction.type)) {
    throw new Error(`Invalid transaction type: ${transaction.type}`);
  }
}

/**
 * Get account from DynamoDB
 */
async function getAccount(accountId) {
  const params = {
    TableName: ACCOUNTS_TABLE,
    Key: { accountId }
  };
  
  const result = await dynamodb.get(params).promise();
  
  if (!result.Item) {
    throw new Error(`Account not found: ${accountId}`);
  }
  
  return result.Item;
}

/**
 * Validate business rules
 */
async function validateBusinessRules(transaction, account) {
  // Check account status
  if (account.status !== 'active') {
    throw new Error(`Account is ${account.status}`);
  }
  
  // Check account balance (for withdrawals/transfers)
  if (['withdrawal', 'transfer'].includes(transaction.type)) {
    if (account.balance < transaction.amount) {
      throw new Error('Insufficient funds');
    }
  }
  
  // Check daily limit
  const todayTotal = await getTodayTransactionTotal(transaction.accountId);
  if (todayTotal + transaction.amount > account.dailyLimit) {
    throw new Error('Daily transaction limit exceeded');
  }
  
  // Check maximum transaction amount
  if (transaction.amount > MAX_TRANSACTION_AMOUNT) {
    throw new Error(`Amount exceeds maximum: $${MAX_TRANSACTION_AMOUNT}`);
  }
}

/**
 * Get today's transaction total (cached in Lambda container)
 */
let transactionCache = {};

async function getTodayTransactionTotal(accountId) {
  const today = new Date().toISOString().split('T')[0];
  const cacheKey = `${accountId}-${today}`;
  
  // Check cache
  if (transactionCache[cacheKey]) {
    return transactionCache[cacheKey];
  }
  
  // Query DynamoDB
  const params = {
    TableName: 'Transactions',
    IndexName: 'AccountDateIndex',
    KeyConditionExpression: 'accountId = :accountId AND begins_with(transactionDate, :date)',
    ExpressionAttributeValues: {
      ':accountId': accountId,
      ':date': today
    }
  };
  
  const result = await dynamodb.query(params).promise();
  const total = result.Items.reduce((sum, item) => sum + item.amount, 0);
  
  // Cache result
  transactionCache[cacheKey] = total;
  
  return total;
}

/**
 * Fraud detection algorithm
 */
async function checkFraud(transaction, account) {
  let fraudScore = 0;
  
  // Factor 1: Unusual amount (30% weight)
  const avgTransaction = account.avgTransactionAmount || 500;
  const amountRatio = transaction.amount / avgTransaction;
  
  if (amountRatio > 10) {
    fraudScore += 0.3; // 10x normal amount
  } else if (amountRatio > 5) {
    fraudScore += 0.15; // 5x normal amount
  }
  
  // Factor 2: Velocity check (30% weight)
  const recentCount = await getRecentTransactionCount(transaction.accountId, 5); // Last 5 minutes
  
  if (recentCount > 5) {
    fraudScore += 0.3; // More than 5 transactions in 5 minutes
  } else if (recentCount > 3) {
    fraudScore += 0.15;
  }
  
  // Factor 3: Round amount (20% weight)
  if (transaction.amount % 1000 === 0 && transaction.amount >= 5000) {
    fraudScore += 0.2; // Suspicious round amounts
  }
  
  // Factor 4: Location change (if provided) (20% weight)
  if (transaction.location && account.lastLocation) {
    const distance = calculateDistance(transaction.location, account.lastLocation);
    const timeDiff = (Date.now() - new Date(account.lastTransactionTime)) / 1000 / 60; // minutes
    
    // Impossible travel detection
    if (distance > 100 && timeDiff < 30) {
      fraudScore += 0.2; // 100+ miles in 30 minutes
    }
  }
  
  return Math.min(fraudScore, 1.0); // Cap at 1.0
}

/**
 * Get recent transaction count
 */
async function getRecentTransactionCount(accountId, minutes) {
  const cutoffTime = new Date(Date.now() - minutes * 60 * 1000).toISOString();
  
  const params = {
    TableName: 'Transactions',
    IndexName: 'AccountDateIndex',
    KeyConditionExpression: 'accountId = :accountId AND transactionDate > :cutoff',
    ExpressionAttributeValues: {
      ':accountId': accountId,
      ':cutoff': cutoffTime
    },
    Select: 'COUNT'
  };
  
  const result = await dynamodb.query(params).promise();
  return result.Count;
}

/**
 * Calculate distance between coordinates (Haversine formula)
 */
function calculateDistance(loc1, loc2) {
  const R = 3959; // Earth radius in miles
  const dLat = toRad(loc2.lat - loc1.lat);
  const dLon = toRad(loc2.lon - loc1.lon);
  
  const a = Math.sin(dLat / 2) * Math.sin(dLat / 2) +
            Math.cos(toRad(loc1.lat)) * Math.cos(toRad(loc2.lat)) *
            Math.sin(dLon / 2) * Math.sin(dLon / 2);
  
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  return R * c;
}

function toRad(deg) {
  return deg * (Math.PI / 180);
}

/**
 * Send fraud alert via SNS
 */
async function sendFraudAlert(transaction, fraudScore) {
  if (!ALERT_TOPIC_ARN) {
    console.warn('Alert topic ARN not configured');
    return;
  }
  
  const message = {
    alertType: 'FRAUD_DETECTED',
    severity: 'HIGH',
    transaction: {
      id: transaction.id,
      accountId: transaction.accountId,
      amount: transaction.amount,
      type: transaction.type
    },
    fraudScore,
    timestamp: new Date().toISOString()
  };
  
  await sns.publish({
    TopicArn: ALERT_TOPIC_ARN,
    Subject: `🚨 Fraud Alert - Account ${transaction.accountId}`,
    Message: JSON.stringify(message, null, 2)
  }).promise();
  
  console.log('Fraud alert sent:', message);
}

/**
 * Record CloudWatch metrics
 */
async function recordMetrics(metricName, value) {
  try {
    await cloudwatch.putMetricData({
      Namespace: 'Banking/Transactions',
      MetricData: [{
        MetricName: metricName,
        Value: value,
        Unit: 'Count',
        Timestamp: new Date()
      }]
    }).promise();
  } catch (error) {
    console.error('Failed to record metric:', error);
    // Don't throw - metrics failure shouldn't break function
  }
}

/**
 * Check if error is retryable
 */
function isRetryable(error) {
  const retryableErrors = [
    'ProvisionedThroughputExceededException',
    'ThrottlingException',
    'ServiceUnavailable',
    'InternalServerError'
  ];
  
  return retryableErrors.some(e => error.code === e || error.message.includes(e));
}

/**
 * Build API Gateway response
 */
function buildResponse(statusCode, body) {
  return {
    statusCode,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify(body)
  };
}

// For local testing
if (require.main === module) {
  const testEvent = {
    accountId: 'ACC-123',
    amount: 500,
    type: 'payment',
    description: 'Test transaction'
  };
  
  exports.handler(testEvent, {
    requestId: 'test-request-id',
    functionName: 'transaction-validator',
    memoryLimitInMB: 512,
    getRemainingTimeInMillis: () => 30000
  }).then(result => {
    console.log('Result:', result);
  }).catch(error => {
    console.error('Error:', error);
  });
}
```

### CloudFormation Template (Infrastructure as Code)

```yaml
# template.yaml - SAM template for serverless deployment
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, prod]

Resources:
  # Lambda Function
  TransactionValidatorFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: !Sub transaction-validator-${Environment}
      Runtime: nodejs18.x
      Handler: index.handler
      CodeUri: ./
      MemorySize: 512
      Timeout: 30
      Environment:
        Variables:
          ACCOUNTS_TABLE: !Ref AccountsTable
          ALERT_TOPIC_ARN: !Ref FraudAlertTopic
          MAX_TRANSACTION_AMOUNT: 10000
          FRAUD_THRESHOLD: 0.8
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref AccountsTable
        - SNSPublishMessagePolicy:
            TopicName: !GetAtt FraudAlertTopic.TopicName
        - CloudWatchPutMetricPolicy: {}
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /validate
            Method: POST
  
  # DynamoDB Table
  AccountsTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: !Sub BankAccounts-${Environment}
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: accountId
          AttributeType: S
      KeySchema:
        - AttributeName: accountId
          KeyType: HASH
  
  # SNS Topic for Alerts
  FraudAlertTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: !Sub fraud-alerts-${Environment}
      DisplayName: Fraud Detection Alerts

Outputs:
  ApiUrl:
    Description: API Gateway endpoint
    Value: !Sub https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/validate
  
  FunctionArn:
    Description: Lambda Function ARN
    Value: !GetAtt TransactionValidatorFunction.Arn
```

### Deployment

```bash
# Using SAM CLI
sam build
sam deploy --guided

# Using Serverless Framework
# serverless.yml
service: transaction-validator

provider:
  name: aws
  runtime: nodejs18.x
  region: us-east-1
  environment:
    ACCOUNTS_TABLE: ${self:service}-accounts-${self:provider.stage}

functions:
  validate:
    handler: index.handler
    memorySize: 512
    timeout: 30
    events:
      - http:
          path: validate
          method: post
          cors: true

# Deploy
serverless deploy --stage dev
```

### Testing

```bash
# Test locally
sam local invoke TransactionValidatorFunction -e test-event.json

# Test-event.json
{
  "accountId": "ACC-123",
  "amount": 500,
  "type": "payment",
  "description": "Test transaction"
}

# Test deployed function
aws lambda invoke \
  --function-name transaction-validator-dev \
  --payload '{"accountId":"ACC-123","amount":500,"type":"payment"}' \
  response.json

# View logs
aws logs tail /aws/lambda/transaction-validator-dev --follow
```

### Key Takeaways from Example 1

1. **Initialization Outside Handler**: AWS SDK clients initialized once (warm start optimization)
2. **Environment Variables**: Configuration via env vars (12-factor app)
3. **Error Handling**: Distinguish retryable vs non-retryable errors
4. **Metrics**: CloudWatch custom metrics for monitoring
5. **Idempotency**: Transaction IDs prevent duplicate processing
6. **Resource Cleanup**: Use `finally` blocks for cleanup

---

## 💡 Example 2: Cold Start Optimization Techniques

Comprehensive guide to minimizing cold starts in production Lambda functions.

### Implementation

```javascript
/**
 * Optimized Lambda Function - Cold Start Best Practices
 */

// ============================================
// 1. MINIMIZE DEPENDENCIES
// ============================================
// ❌ Bad: Import entire SDK
// const AWS = require('aws-sdk');

// ✅ Good: Import only what you need
const DynamoDB = require('aws-sdk/clients/dynamodb');
const SNS = require('aws-sdk/clients/sns');

// ============================================
// 2. INITIALIZE OUTSIDE HANDLER
// ============================================
// Initialized once per container (warm starts reuse)
const dynamodb = new DynamoDB.DocumentClient({
  maxRetries: 3,
  httpOptions: {
    timeout: 5000,
    connectTimeout: 1000
  }
});

// Connection pool (reused across invocations)
let dbConnection = null;

// Cache (persists across warm invocations)
const cache = new Map();

// ============================================
// 3. LAZY INITIALIZATION
// ============================================
async function getDbConnection() {
  if (dbConnection && dbConnection.isConnected()) {
    return dbConnection;
  }
  
  console.log('Initializing database connection...');
  dbConnection = await createConnection({
    host: process.env.DB_HOST,
    database: process.env.DB_NAME,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    // Keep connection alive
    keepAlive: true,
    keepAliveInitialDelay: 10000
  });
  
  return dbConnection;
}

// ============================================
// 4. PROVISIONED CONCURRENCY
// ============================================
/**
 * Main handler with warm-up support
 */
exports.handler = async (event, context) => {
  // Handle warmup events (from CloudWatch scheduled rule)
  if (event.source === 'aws.events' && event['detail-type'] === 'Scheduled Event') {
    console.log('Warmup event received');
    return { statusCode: 200, body: 'Warmed up!' };
  }
  
  const startTime = Date.now();
  
  try {
    // Your business logic
    const result = await processTransaction(event);
    
    const duration = Date.now() - startTime;
    console.log(`Execution time: ${duration}ms`);
    
    return {
      statusCode: 200,
      body: JSON.stringify(result)
    };
    
  } catch (error) {
    console.error('Error:', error);
    return {
      statusCode: 500,
      body: JSON.stringify({ error: error.message })
    };
  }
};

// ============================================
// 5. CACHING STRATEGIES
// ============================================
async function getAccount(accountId) {
  // Check in-memory cache first
  if (cache.has(accountId)) {
    const cached = cache.get(accountId);
    
    // Check if cache is still valid (5 minutes TTL)
    if (Date.now() - cached.timestamp < 5 * 60 * 1000) {
      console.log('Cache hit:', accountId);
      return cached.data;
    }
  }
  
  // Cache miss - fetch from DynamoDB
  console.log('Cache miss:', accountId);
  const result = await dynamodb.get({
    TableName: process.env.ACCOUNTS_TABLE,
    Key: { accountId }
  }).promise();
  
  if (result.Item) {
    // Store in cache
    cache.set(accountId, {
      data: result.Item,
      timestamp: Date.now()
    });
  }
  
  return result.Item;
}

// ============================================
// 6. PARALLEL OPERATIONS
// ============================================
async function processTransaction(event) {
  const transaction = JSON.parse(event.body);
  
  // ❌ Bad: Sequential (slow)
  // const account = await getAccount(transaction.accountId);
  // const settings = await getSettings();
  // const riskScore = await calculateRisk(transaction);
  
  // ✅ Good: Parallel (fast)
  const [account, settings, riskScore] = await Promise.all([
    getAccount(transaction.accountId),
    getSettings(),
    calculateRisk(transaction)
  ]);
  
  return validateTransaction(transaction, account, settings, riskScore);
}

// ============================================
// 7. LAMBDA LAYERS (Shared Dependencies)
// ============================================
/**
 * Lambda Layer structure:
 * 
 * layer/
 * └── nodejs/
 *     ├── node_modules/
 *     │   ├── aws-sdk/
 *     │   ├── lodash/
 *     │   └── moment/
 *     └── package.json
 * 
 * Deploy layer:
 * zip -r layer.zip nodejs/
 * aws lambda publish-layer-version \
 *   --layer-name common-dependencies \
 *   --zip-file fileb://layer.zip \
 *   --compatible-runtimes nodejs18.x
 */

// Use layer in function (no need to bundle these deps)
const _ = require('lodash');
const moment = require('moment');

// ============================================
// 8. WEBPACK BUNDLING
// ============================================
/**
 * webpack.config.js - Reduce bundle size
 */
module.exports = {
  entry: './index.js',
  target: 'node',
  mode: 'production',
  optimization: {
    minimize: true
  },
  externals: {
    'aws-sdk': 'aws-sdk' // Exclude AWS SDK (included in Lambda runtime)
  },
  output: {
    filename: 'index.js',
    path: __dirname + '/dist',
    libraryTarget: 'commonjs2'
  }
};

// ============================================
// 9. ENVIRONMENT-SPECIFIC OPTIMIZATIONS
// ============================================
const IS_COLD_START = !global.isWarm;
global.isWarm = true;

exports.optimizedHandler = async (event, context) => {
  if (IS_COLD_START) {
    console.log('🥶 Cold start detected');
    
    // Perform expensive initialization
    await initializeConnections();
  } else {
    console.log('🔥 Warm invocation');
  }
  
  // Business logic
  return processRequest(event);
};

// ============================================
// 10. MONITORING COLD STARTS
// ============================================
const AWS = require('aws-sdk');
const cloudwatch = new AWS.CloudWatch();

async function recordColdStart(duration) {
  await cloudwatch.putMetricData({
    Namespace: 'Lambda/ColdStarts',
    MetricData: [{
      MetricName: 'ColdStartDuration',
      Value: duration,
      Unit: 'Milliseconds',
      Dimensions: [{
        Name: 'FunctionName',
        Value: process.env.AWS_LAMBDA_FUNCTION_NAME
      }]
    }]
  }).promise();
}

// ============================================
// HELPER FUNCTIONS
// ============================================
async function getSettings() {
  // Stub implementation
  return {
    maxTransactionAmount: 10000,
    fraudThreshold: 0.8
  };
}

async function calculateRisk(transaction) {
  // Stub implementation
  return Math.random();
}

async function validateTransaction(transaction, account, settings, riskScore) {
  return {
    valid: riskScore < settings.fraudThreshold,
    transactionId: `TXN-${Date.now()}`,
    riskScore
  };
}

async function initializeConnections() {
  // Stub implementation
  console.log('Initializing connections...');
}

async function processRequest(event) {
  return { message: 'Processed successfully' };
}

function createConnection(config) {
  // Stub implementation
  return {
    isConnected: () => true
  };
}
```

### Cold Start Optimization Summary

| Technique | Impact | Difficulty |
|-----------|--------|------------|
| **Minimize dependencies** | High | Easy |
| **Initialize outside handler** | High | Easy |
| **Lazy initialization** | Medium | Medium |
| **Provisioned concurrency** | Very High | Easy (costs more) |
| **Caching** | Medium | Medium |
| **Parallel operations** | Medium | Easy |
| **Lambda layers** | High | Medium |
| **Webpack bundling** | Medium | Medium |
| **Keep functions warm** | High | Easy |

### Provisioned Concurrency Configuration

```bash
# Enable provisioned concurrency (eliminates cold starts)
aws lambda put-provisioned-concurrency-config \
  --function-name transaction-validator \
  --provisioned-concurrent-executions 5 \
  --qualifier $LATEST

# Auto-scaling configuration
aws application-autoscaling register-scalable-target \
  --service-namespace lambda \
  --resource-id function:transaction-validator:provisioned-concurrency \
  --scalable-dimension lambda:function:ProvisionedConcurrentExecutions \
  --min-capacity 2 \
  --max-capacity 10

# Costs: ~$0.015 per hour per GB (plus invocation costs)
```

### Keep-Warm Strategy

```javascript
// CloudWatch Events Rule (EventBridge)
{
  "schedule": "rate(5 minutes)",
  "targets": [{
    "arn": "arn:aws:lambda:...:function:transaction-validator",
    "input": {
      "source": "aws.events",
      "detail-type": "Scheduled Event",
      "warmup": true
    }
  }]
}

// Handler responds to warmup
if (event.warmup) {
  return { statusCode: 200, body: 'Staying warm!' };
}
```

---

## 🎯 Lambda Best Practices

### 1. Function Design

```javascript
// ✅ Single Responsibility
exports.validateTransaction = async (event) => { /* ... */ };
exports.processPayment = async (event) => { /* ... */ };
exports.sendNotification = async (event) => { /* ... */ };

// ❌ God Function
exports.doEverything = async (event) => { /* handles 10 different things */ };
```

### 2. Error Handling

```javascript
exports.handler = async (event) => {
  try {
    return await processEvent(event);
  } catch (error) {
    console.error('Error:', error);
    
    // Structured error logging
    console.error(JSON.stringify({
      errorType: error.constructor.name,
      errorMessage: error.message,
      stackTrace: error.stack,
      event: event
    }));
    
    // Throw for Lambda to retry (if using SQS, SNS, etc.)
    if (shouldRetry(error)) {
      throw error;
    }
    
    // Return error response (if using API Gateway)
    return {
      statusCode: error.statusCode || 500,
      body: JSON.stringify({ error: error.message })
    };
  }
};
```

### 3. Secrets Management

```javascript
const AWS = require('aws-sdk');
const ssm = new AWS.SSM();

// ❌ Bad: Hardcoded secrets
const dbPassword = 'hardcoded-password';

// ✅ Good: AWS Systems Manager Parameter Store
let cachedSecrets = null;

async function getSecrets() {
  if (cachedSecrets) return cachedSecrets;
  
  const params = {
    Names: [
      '/banking/db-password',
      '/banking/api-key'
    ],
    WithDecryption: true
  };
  
  const result = await ssm.getParameters(params).promise();
  
  cachedSecrets = result.Parameters.reduce((acc, param) => {
    const key = param.Name.split('/').pop();
    acc[key] = param.Value;
    return acc;
  }, {});
  
  return cachedSecrets;
}

// Or use AWS Secrets Manager
const secretsManager = new AWS.SecretsManager();

async function getSecret(secretId) {
  const data = await secretsManager.getSecretValue({ SecretId: secretId }).promise();
  return JSON.parse(data.SecretString);
}
```

### 4. Monitoring and Logging

```javascript
// Structured logging
function log(level, message, meta = {}) {
  console.log(JSON.stringify({
    timestamp: new Date().toISOString(),
    level,
    message,
    requestId: context.requestId,
    functionName: context.functionName,
    ...meta
  }));
}

// Usage
log('INFO', 'Processing transaction', {
  transactionId: 'TXN-123',
  amount: 1000,
  accountId: 'ACC-456'
});

// Custom metrics
await cloudwatch.putMetricData({
  Namespace: 'Banking',
  MetricData: [{
    MetricName: 'TransactionProcessingTime',
    Value: duration,
    Unit: 'Milliseconds',
    Dimensions: [
      { Name: 'TransactionType', Value: transaction.type },
      { Name: 'Environment', Value: process.env.ENVIRONMENT }
    ]
  }]
}).promise();
```

### 5. Testing

```javascript
// Unit tests (Jest)
const { handler } = require('./index');

describe('Transaction Validator', () => {
  test('approves valid transaction', async () => {
    const event = {
      body: JSON.stringify({
        accountId: 'ACC-123',
        amount: 500,
        type: 'payment'
      })
    };
    
    const result = await handler(event, mockContext);
    
    expect(result.statusCode).toBe(200);
    const body = JSON.parse(result.body);
    expect(body.approved).toBe(true);
  });
  
  test('rejects high-risk transaction', async () => {
    const event = {
      body: JSON.stringify({
        accountId: 'ACC-123',
        amount: 50000, // Very high amount
        type: 'transfer'
      })
    };
    
    const result = await handler(event, mockContext);
    
    expect(result.statusCode).toBe(403);
  });
});

// Integration tests (with LocalStack)
// docker run -d -p 4566:4566 localstack/localstack
process.env.AWS_ENDPOINT = 'http://localhost:4566';
```

---

## 📚 Common Interview Questions

### Q1: What is a cold start and how do you minimize it?

**Answer**:
Cold start is the initialization time when Lambda creates a new container. Minimize by:
1. Reduce deployment package size
2. Initialize connections outside handler
3. Use provisioned concurrency
4. Implement keep-warm strategies
5. Use Lambda layers for shared dependencies

### Q2: What's the maximum execution time for Lambda?

**Answer**: 15 minutes. For longer tasks, use:
- Step Functions (orchestrate multiple Lambdas)
- ECS/Fargate (container-based)
- Batch processing (SQS + Lambda with shorter chunks)

### Q3: How do you handle database connections in Lambda?

**Answer**:
```javascript
// Initialize outside handler (reused)
let connection = null;

async function getConnection() {
  if (!connection) {
    connection = await createConnection();
  }
  return connection;
}

exports.handler = async (event) => {
  const conn = await getConnection();
  // Use connection...
};
```

### Q4: What are Lambda execution models?

**Answer**:
- **Synchronous**: API Gateway, direct invoke (wait for response)
- **Asynchronous**: S3, SNS (Lambda retries on failure)
- **Stream-based**: DynamoDB Streams, Kinesis (poll-based)

### Q5: How do you ensure idempotency in Lambda?

**Answer**:
```javascript
// Store processed request IDs in DynamoDB
async function isProcessed(requestId) {
  try {
    await dynamodb.get({
      TableName: 'ProcessedRequests',
      Key: { requestId }
    }).promise();
    return true; // Already processed
  } catch {
    return false;
  }
}

// Mark as processed
await dynamodb.put({
  TableName: 'ProcessedRequests',
  Item: {
    requestId,
    processedAt: Date.now(),
    ttl: Date.now() / 1000 + 86400 // 24 hours
  }
}).promise();
```

---

## ✅ Summary & Key Takeaways

### Core Concepts

1. **Serverless**: No server management, auto-scaling, pay-per-use
2. **Lambda**: AWS's FaaS, event-driven, 15-minute max execution
3. **Cold Start**: Container initialization delay (minimize it!)
4. **Event Sources**: API Gateway, S3, DynamoDB, SQS, SNS, etc.

### Optimization Techniques

1. **Minimize bundle size**: Only import what you need
2. **Initialize outside handler**: Reuse connections, SDK clients
3. **Caching**: In-memory cache for frequently accessed data
4. **Parallel operations**: Use `Promise.all()` for concurrent calls
5. **Provisioned concurrency**: Eliminate cold starts (costs more)

### Production Checklist

```javascript
// ✅ Lambda Production Readiness
- [ ] Error handling with structured logging
- [ ] Secrets in Parameter Store/Secrets Manager
- [ ] Custom CloudWatch metrics
- [ ] X-Ray tracing enabled
- [ ] Dead letter queue configured
- [ ] Resource cleanup in finally blocks
- [ ] Idempotency for retryable operations
- [ ] Environment-specific configuration
- [ ] Unit and integration tests
- [ ] CI/CD pipeline (SAM/Serverless Framework)
```

### Banking Applications

1. **Transaction Validation**: Real-time fraud detection
2. **Payment Processing**: Async payment webhooks
3. **Statement Generation**: On-demand document creation
4. **Notifications**: Event-driven customer alerts
5. **Compliance**: Automated regulatory checks

---

**Status**: ✅ Complete with production-ready Lambda examples and cold start optimization techniques!
