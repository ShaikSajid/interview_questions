# Step Functions & EventBridge Architecture

## Question 11: Step Functions Saga Orchestration

### 📋 Question Statement

Implement a distributed transaction using AWS Step Functions with the Saga pattern for Emirates NBD payment processing. Include compensation logic, error handling, and integration with microservices.

---

### 🏗️ Architecture Overview

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    EMIRATES NBD PAYMENT SAGA FLOW                           │
└────────────────────────────────────────────────────────────────────────────┘

                              ┌──────────────┐
                              │   API GW     │
                              │ (Initiate)   │
                              └──────┬───────┘
                                     │
                                     ▼
                     ┌───────────────────────────────┐
                     │   Step Functions State Machine │
                     │      (Saga Orchestrator)       │
                     └───────────────┬───────────────┘
                                     │
        ┌────────────────────────────┼────────────────────────────┐
        │                            │                            │
        ▼                            ▼                            ▼
┌───────────────┐          ┌────────────────┐          ┌──────────────────┐
│  Reserve      │          │   Validate     │          │   Process        │
│  Funds        │──────────│   Payment      │──────────│   Payment        │
│  (Lambda)     │          │   (Lambda)     │          │   (Lambda)       │
└───────┬───────┘          └────────┬───────┘          └────────┬─────────┘
        │                           │                           │
        │ Success                   │ Success                   │ Success
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────┐          ┌────────────────┐          ┌──────────────────┐
│  Update       │          │   Notify       │          │   Complete       │
│  Balance      │──────────│   Customer     │──────────│   Transaction    │
│  (DynamoDB)   │          │   (SNS)        │          │   (DynamoDB)     │
└───────────────┘          └────────────────┘          └──────────────────┘

                            COMPENSATION FLOW
                        (Triggered on any failure)

┌───────────────┐          ┌────────────────┐          ┌──────────────────┐
│  Rollback     │          │   Rollback     │          │   Rollback       │
│  Funds        │◄─────────│   Validation   │◄─────────│   Payment        │
│  (Lambda)     │          │   (Lambda)     │          │   (Lambda)       │
└───────┬───────┘          └────────┬───────┘          └────────┬─────────┘
        │                           │                           │
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────┐          ┌────────────────┐          ┌──────────────────┐
│  Restore      │          │   Log Error    │          │   Notify         │
│  Balance      │──────────│   (CloudWatch) │──────────│   Failure        │
│  (DynamoDB)   │          │                │          │   (SNS)          │
└───────────────┘          └────────────────┘          └──────────────────┘
```

---

### 🔧 Step Functions State Machine Definition

```json
{
  "Comment": "Emirates NBD Payment Processing Saga",
  "StartAt": "ReserveFunds",
  "States": {
    "ReserveFunds": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:reserve-funds",
      "TimeoutSeconds": 30,
      "Retry": [
        {
          "ErrorEquals": ["States.Timeout", "States.TaskFailed"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "ResultPath": "$.error",
          "Next": "RollbackReserveFunds"
        }
      ],
      "ResultPath": "$.reservationResult",
      "Next": "ValidatePayment"
    },
    
    "ValidatePayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:validate-payment",
      "TimeoutSeconds": 30,
      "Retry": [
        {
          "ErrorEquals": ["States.Timeout"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["ValidationError"],
          "ResultPath": "$.error",
          "Next": "RollbackReserveFunds"
        },
        {
          "ErrorEquals": ["States.ALL"],
          "ResultPath": "$.error",
          "Next": "RollbackReserveFunds"
        }
      ],
      "ResultPath": "$.validationResult",
      "Next": "CheckFraudScore"
    },
    
    "CheckFraudScore": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:fraud-detection",
      "TimeoutSeconds": 10,
      "ResultPath": "$.fraudResult",
      "Next": "EvaluateFraudScore"
    },
    
    "EvaluateFraudScore": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.fraudResult.score",
          "NumericGreaterThan": 80,
          "Next": "BlockTransaction"
        },
        {
          "Variable": "$.fraudResult.score",
          "NumericGreaterThan": 50,
          "Next": "RequireManualApproval"
        }
      ],
      "Default": "ProcessPayment"
    },
    
    "BlockTransaction": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:block-transaction",
      "ResultPath": "$.blockResult",
      "Next": "RollbackReserveFunds"
    },
    
    "RequireManualApproval": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
      "Parameters": {
        "QueueUrl": "https://sqs.us-east-1.amazonaws.com/123456789012/manual-approval-queue",
        "MessageBody": {
          "Token.$": "$$.Task.Token",
          "TransactionId.$": "$.transactionId",
          "Amount.$": "$.amount",
          "FraudScore.$": "$.fraudResult.score"
        }
      },
      "TimeoutSeconds": 3600,
      "Catch": [
        {
          "ErrorEquals": ["States.Timeout"],
          "ResultPath": "$.error",
          "Next": "RollbackReserveFunds"
        }
      ],
      "ResultPath": "$.approvalResult",
      "Next": "ProcessPayment"
    },
    
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:process-payment",
      "TimeoutSeconds": 60,
      "Retry": [
        {
          "ErrorEquals": ["States.Timeout", "States.TaskFailed"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["InsufficientFunds"],
          "ResultPath": "$.error",
          "Next": "RollbackValidation"
        },
        {
          "ErrorEquals": ["States.ALL"],
          "ResultPath": "$.error",
          "Next": "RollbackValidation"
        }
      ],
      "ResultPath": "$.paymentResult",
      "Next": "UpdateBalance"
    },
    
    "UpdateBalance": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "banking-accounts",
        "Key": {
          "accountId": {
            "S.$": "$.fromAccount"
          }
        },
        "UpdateExpression": "SET balance = balance - :amount, lastModified = :timestamp",
        "ExpressionAttributeValues": {
          ":amount": {
            "N.$": "States.Format('{}', $.amount)"
          },
          ":timestamp": {
            "S.$": "$$.State.EnteredTime"
          }
        }
      },
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "ResultPath": "$.error",
          "Next": "RollbackPayment"
        }
      ],
      "ResultPath": "$.balanceResult",
      "Next": "NotifyCustomer"
    },
    
    "NotifyCustomer": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:123456789012:payment-notifications",
        "Message": {
          "TransactionId.$": "$.transactionId",
          "Amount.$": "$.amount",
          "Status": "SUCCESS"
        }
      },
      "ResultPath": "$.notificationResult",
      "Next": "CompleteTransaction"
    },
    
    "CompleteTransaction": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:putItem",
      "Parameters": {
        "TableName": "banking-transactions",
        "Item": {
          "transactionId": {
            "S.$": "$.transactionId"
          },
          "fromAccount": {
            "S.$": "$.fromAccount"
          },
          "toAccount": {
            "S.$": "$.toAccount"
          },
          "amount": {
            "N.$": "States.Format('{}', $.amount)"
          },
          "status": {
            "S": "COMPLETED"
          },
          "timestamp": {
            "S.$": "$$.State.EnteredTime"
          }
        }
      },
      "ResultPath": "$.completionResult",
      "Next": "Success"
    },
    
    "Success": {
      "Type": "Succeed"
    },
    
    "RollbackPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:rollback-payment",
      "ResultPath": "$.rollbackPaymentResult",
      "Next": "RollbackValidation"
    },
    
    "RollbackValidation": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:rollback-validation",
      "ResultPath": "$.rollbackValidationResult",
      "Next": "RollbackReserveFunds"
    },
    
    "RollbackReserveFunds": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:rollback-reserve-funds",
      "ResultPath": "$.rollbackReserveFundsResult",
      "Next": "LogFailure"
    },
    
    "LogFailure": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:putItem",
      "Parameters": {
        "TableName": "banking-transactions",
        "Item": {
          "transactionId": {
            "S.$": "$.transactionId"
          },
          "status": {
            "S": "FAILED"
          },
          "error": {
            "S.$": "$.error.Error"
          },
          "errorCause": {
            "S.$": "$.error.Cause"
          },
          "timestamp": {
            "S.$": "$$.State.EnteredTime"
          }
        }
      },
      "Next": "NotifyFailure"
    },
    
    "NotifyFailure": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:123456789012:payment-notifications",
        "Message": {
          "TransactionId.$": "$.transactionId",
          "Status": "FAILED",
          "Error.$": "$.error.Error"
        }
      },
      "Next": "Fail"
    },
    
    "Fail": {
      "Type": "Fail",
      "Error": "PaymentSagaFailed",
      "Cause": "Payment processing failed and was rolled back"
    }
  }
}
```

---

### 💻 Lambda Functions Implementation

#### Reserve Funds Function

```javascript
// lambdas/reserve-funds/index.js
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, UpdateCommand, GetCommand } = require('@aws-sdk/lib-dynamodb');

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

exports.handler = async (event) => {
  console.log('Reserve Funds Event:', JSON.stringify(event, null, 2));
  
  const { transactionId, fromAccount, amount } = event;
  
  try {
    // Check current balance
    const getResponse = await docClient.send(new GetCommand({
      TableName: 'banking-accounts',
      Key: { accountId: fromAccount }
    }));
    
    if (!getResponse.Item) {
      throw new Error('Account not found');
    }
    
    const currentBalance = getResponse.Item.balance;
    const reservedFunds = getResponse.Item.reservedFunds || 0;
    const availableBalance = currentBalance - reservedFunds;
    
    if (availableBalance < amount) {
      throw new Error('Insufficient funds');
    }
    
    // Reserve funds (optimistic locking with condition)
    const updateResponse = await docClient.send(new UpdateCommand({
      TableName: 'banking-accounts',
      Key: { accountId: fromAccount },
      UpdateExpression: 'SET reservedFunds = :newReserved, version = :newVersion',
      ConditionExpression: 'version = :currentVersion',
      ExpressionAttributeValues: {
        ':newReserved': reservedFunds + amount,
        ':currentVersion': getResponse.Item.version,
        ':newVersion': getResponse.Item.version + 1
      },
      ReturnValues: 'ALL_NEW'
    }));
    
    // Store reservation details for rollback
    await docClient.send(new UpdateCommand({
      TableName: 'banking-reservations',
      Key: { transactionId },
      UpdateExpression: 'SET accountId = :accountId, amount = :amount, #status = :status, createdAt = :timestamp',
      ExpressionAttributeNames: {
        '#status': 'status'
      },
      ExpressionAttributeValues: {
        ':accountId': fromAccount,
        ':amount': amount,
        ':status': 'RESERVED',
        ':timestamp': new Date().toISOString()
      }
    }));
    
    return {
      statusCode: 200,
      reservationId: transactionId,
      reservedAmount: amount,
      newReservedTotal: updateResponse.Attributes.reservedFunds
    };
  } catch (error) {
    console.error('Reserve funds error:', error);
    
    if (error.name === 'ConditionalCheckFailedException') {
      throw new Error('ConcurrentModificationError: Account was modified by another transaction');
    }
    
    throw error;
  }
};
```

#### Validate Payment Function

```javascript
// lambdas/validate-payment/index.js
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, GetCommand } = require('@aws-sdk/lib-dynamodb');

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

exports.handler = async (event) => {
  console.log('Validate Payment Event:', JSON.stringify(event, null, 2));
  
  const { transactionId, fromAccount, toAccount, amount, currency } = event;
  
  try {
    // Validate accounts exist
    const [fromAccountData, toAccountData] = await Promise.all([
      docClient.send(new GetCommand({
        TableName: 'banking-accounts',
        Key: { accountId: fromAccount }
      })),
      docClient.send(new GetCommand({
        TableName: 'banking-accounts',
        Key: { accountId: toAccount }
      }))
    ]);
    
    if (!fromAccountData.Item) {
      const error = new Error('Source account not found');
      error.name = 'ValidationError';
      throw error;
    }
    
    if (!toAccountData.Item) {
      const error = new Error('Destination account not found');
      error.name = 'ValidationError';
      throw error;
    }
    
    // Validate account status
    if (fromAccountData.Item.status !== 'ACTIVE') {
      const error = new Error('Source account is not active');
      error.name = 'ValidationError';
      throw error;
    }
    
    if (toAccountData.Item.status !== 'ACTIVE') {
      const error = new Error('Destination account is not active');
      error.name = 'ValidationError';
      throw error;
    }
    
    // Validate amount
    if (amount <= 0) {
      const error = new Error('Amount must be greater than zero');
      error.name = 'ValidationError';
      throw error;
    }
    
    // Validate daily limit
    const dailyLimit = fromAccountData.Item.dailyTransferLimit || 50000;
    const todayTransfers = await getTodayTransferTotal(fromAccount);
    
    if (todayTransfers + amount > dailyLimit) {
      const error = new Error(`Daily transfer limit exceeded. Limit: ${dailyLimit}, Today: ${todayTransfers}`);
      error.name = 'ValidationError';
      throw error;
    }
    
    // Validate currency
    if (currency !== fromAccountData.Item.currency) {
      const error = new Error('Currency mismatch');
      error.name = 'ValidationError';
      throw error;
    }
    
    return {
      statusCode: 200,
      valid: true,
      fromAccountCurrency: fromAccountData.Item.currency,
      toAccountCurrency: toAccountData.Item.currency
    };
  } catch (error) {
    console.error('Validation error:', error);
    throw error;
  }
};

async function getTodayTransferTotal(accountId) {
  const today = new Date().toISOString().split('T')[0];
  
  // Query transactions table for today's transfers
  // (Implementation depends on your GSI structure)
  return 0; // Placeholder
}
```

#### Process Payment Function

```javascript
// lambdas/process-payment/index.js
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, TransactWriteCommand } = require('@aws-sdk/lib-dynamodb');

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

exports.handler = async (event) => {
  console.log('Process Payment Event:', JSON.stringify(event, null, 2));
  
  const { transactionId, fromAccount, toAccount, amount } = event;
  
  try {
    // Use DynamoDB transaction for atomic debit/credit
    await docClient.send(new TransactWriteCommand({
      TransactItems: [
        {
          // Debit from source account
          Update: {
            TableName: 'banking-accounts',
            Key: { accountId: fromAccount },
            UpdateExpression: 'SET balance = balance - :amount, reservedFunds = reservedFunds - :amount',
            ConditionExpression: 'balance >= reservedFunds AND reservedFunds >= :amount',
            ExpressionAttributeValues: {
              ':amount': amount
            }
          }
        },
        {
          // Credit to destination account
          Update: {
            TableName: 'banking-accounts',
            Key: { accountId: toAccount },
            UpdateExpression: 'SET balance = balance + :amount',
            ExpressionAttributeValues: {
              ':amount': amount
            }
          }
        },
        {
          // Update reservation status
          Update: {
            TableName: 'banking-reservations',
            Key: { transactionId },
            UpdateExpression: 'SET #status = :status, processedAt = :timestamp',
            ExpressionAttributeNames: {
              '#status': 'status'
            },
            ExpressionAttributeValues: {
              ':status': 'PROCESSED',
              ':timestamp': new Date().toISOString()
            }
          }
        }
      ]
    }));
    
    return {
      statusCode: 200,
      transactionId,
      status: 'COMPLETED',
      processedAt: new Date().toISOString()
    };
  } catch (error) {
    console.error('Process payment error:', error);
    
    if (error.name === 'TransactionCanceledException') {
      const reasons = error.CancellationReasons;
      console.error('Transaction cancelled:', reasons);
      
      // Check if it's due to insufficient funds
      if (reasons.some(r => r.Code === 'ConditionalCheckFailed')) {
        const insufficientError = new Error('Insufficient funds or reservation mismatch');
        insufficientError.name = 'InsufficientFunds';
        throw insufficientError;
      }
    }
    
    throw error;
  }
};
```

#### Rollback Functions

```javascript
// lambdas/rollback-reserve-funds/index.js
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, UpdateCommand, GetCommand } = require('@aws-sdk/lib-dynamodb');

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

exports.handler = async (event) => {
  console.log('Rollback Reserve Funds Event:', JSON.stringify(event, null, 2));
  
  const { transactionId, fromAccount, amount } = event;
  
  try {
    // Get reservation details
    const reservation = await docClient.send(new GetCommand({
      TableName: 'banking-reservations',
      Key: { transactionId }
    }));
    
    if (!reservation.Item || reservation.Item.status !== 'RESERVED') {
      console.log('No active reservation to rollback');
      return { statusCode: 200, message: 'No rollback needed' };
    }
    
    // Release reserved funds
    await docClient.send(new UpdateCommand({
      TableName: 'banking-accounts',
      Key: { accountId: fromAccount },
      UpdateExpression: 'SET reservedFunds = reservedFunds - :amount',
      ExpressionAttributeValues: {
        ':amount': amount
      }
    }));
    
    // Mark reservation as rolled back
    await docClient.send(new UpdateCommand({
      TableName: 'banking-reservations',
      Key: { transactionId },
      UpdateExpression: 'SET #status = :status, rolledBackAt = :timestamp',
      ExpressionAttributeNames: {
        '#status': 'status'
      },
      ExpressionAttributeValues: {
        ':status': 'ROLLED_BACK',
        ':timestamp': new Date().toISOString()
      }
    }));
    
    return {
      statusCode: 200,
      message: 'Funds reservation rolled back successfully'
    };
  } catch (error) {
    console.error('Rollback error:', error);
    // Log error but don't fail - compensation should be idempotent
    return {
      statusCode: 500,
      message: 'Rollback failed',
      error: error.message
    };
  }
};

// lambdas/rollback-validation/index.js
exports.handler = async (event) => {
  console.log('Rollback Validation Event:', JSON.stringify(event, null, 2));
  
  // Validation is stateless, nothing to rollback
  // Just log the failure for audit purposes
  
  return {
    statusCode: 200,
    message: 'Validation rollback completed (no-op)'
  };
};

// lambdas/rollback-payment/index.js
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, TransactWriteCommand, GetCommand } = require('@aws-sdk/lib-dynamodb');

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

exports.handler = async (event) => {
  console.log('Rollback Payment Event:', JSON.stringify(event, null, 2));
  
  const { transactionId, fromAccount, toAccount, amount } = event;
  
  try {
    // Check if payment was actually processed
    const reservation = await docClient.send(new GetCommand({
      TableName: 'banking-reservations',
      Key: { transactionId }
    }));
    
    if (!reservation.Item || reservation.Item.status !== 'PROCESSED') {
      console.log('Payment was not processed, no rollback needed');
      return { statusCode: 200, message: 'No payment rollback needed' };
    }
    
    // Reverse the payment
    await docClient.send(new TransactWriteCommand({
      TransactItems: [
        {
          // Credit back to source account
          Update: {
            TableName: 'banking-accounts',
            Key: { accountId: fromAccount },
            UpdateExpression: 'SET balance = balance + :amount, reservedFunds = reservedFunds + :amount',
            ExpressionAttributeValues: {
              ':amount': amount
            }
          }
        },
        {
          // Debit from destination account
          Update: {
            TableName: 'banking-accounts',
            Key: { accountId: toAccount },
            UpdateExpression: 'SET balance = balance - :amount',
            ExpressionAttributeValues: {
              ':amount': amount
            }
          }
        },
        {
          // Update reservation status
          Update: {
            TableName: 'banking-reservations',
            Key: { transactionId },
            UpdateExpression: 'SET #status = :status, paymentRolledBackAt = :timestamp',
            ExpressionAttributeNames: {
              '#status': 'status'
            },
            ExpressionAttributeValues: {
              ':status': 'PAYMENT_ROLLED_BACK',
              ':timestamp': new Date().toISOString()
            }
          }
        }
      ]
    }));
    
    return {
      statusCode: 200,
      message: 'Payment rolled back successfully'
    };
  } catch (error) {
    console.error('Payment rollback error:', error);
    return {
      statusCode: 500,
      message: 'Payment rollback failed',
      error: error.message
    };
  }
};
```

---

### 🚀 CDK Infrastructure for Step Functions

```typescript
// infrastructure/cdk/saga-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as sfn from 'aws-cdk-lib/aws-stepfunctions';
import * as tasks from 'aws-cdk-lib/aws-stepfunctions-tasks';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as sqs from 'aws-cdk-lib/aws-sqs';
import * as iam from 'aws-cdk-lib/aws-iam';

export class PaymentSagaStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // DynamoDB Tables
    const accountsTable = new dynamodb.Table(this, 'AccountsTable', {
      tableName: 'banking-accounts',
      partitionKey: { name: 'accountId', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST
    });

    const reservationsTable = new dynamodb.Table(this, 'ReservationsTable', {
      tableName: 'banking-reservations',
      partitionKey: { name: 'transactionId', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST
    });

    const transactionsTable = new dynamodb.Table(this, 'TransactionsTable', {
      tableName: 'banking-transactions',
      partitionKey: { name: 'transactionId', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST
    });

    // SNS Topic for notifications
    const notificationTopic = new sns.Topic(this, 'PaymentNotificationsTopic', {
      topicName: 'payment-notifications'
    });

    // SQS Queue for manual approval
    const approvalQueue = new sqs.Queue(this, 'ManualApprovalQueue', {
      queueName: 'manual-approval-queue',
      visibilityTimeout: cdk.Duration.hours(1)
    });

    // Lambda Functions
    const reserveFundsLambda = new lambda.Function(this, 'ReserveFundsFunction', {
      functionName: 'reserve-funds',
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambdas/reserve-funds'),
      timeout: cdk.Duration.seconds(30),
      environment: {
        ACCOUNTS_TABLE: accountsTable.tableName,
        RESERVATIONS_TABLE: reservationsTable.tableName
      }
    });

    const validatePaymentLambda = new lambda.Function(this, 'ValidatePaymentFunction', {
      functionName: 'validate-payment',
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambdas/validate-payment'),
      timeout: cdk.Duration.seconds(30),
      environment: {
        ACCOUNTS_TABLE: accountsTable.tableName
      }
    });

    const processPaymentLambda = new lambda.Function(this, 'ProcessPaymentFunction', {
      functionName: 'process-payment',
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambdas/process-payment'),
      timeout: cdk.Duration.seconds(60),
      environment: {
        ACCOUNTS_TABLE: accountsTable.tableName,
        RESERVATIONS_TABLE: reservationsTable.tableName
      }
    });

    const fraudDetectionLambda = new lambda.Function(this, 'FraudDetectionFunction', {
      functionName: 'fraud-detection',
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambdas/fraud-detection'),
      timeout: cdk.Duration.seconds(10)
    });

    // Rollback functions
    const rollbackReserveFundsLambda = new lambda.Function(this, 'RollbackReserveFundsFunction', {
      functionName: 'rollback-reserve-funds',
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambdas/rollback-reserve-funds'),
      timeout: cdk.Duration.seconds(30),
      environment: {
        ACCOUNTS_TABLE: accountsTable.tableName,
        RESERVATIONS_TABLE: reservationsTable.tableName
      }
    });

    // Grant permissions
    accountsTable.grantReadWriteData(reserveFundsLambda);
    accountsTable.grantReadWriteData(validatePaymentLambda);
    accountsTable.grantReadWriteData(processPaymentLambda);
    accountsTable.grantReadWriteData(rollbackReserveFundsLambda);
    
    reservationsTable.grantReadWriteData(reserveFundsLambda);
    reservationsTable.grantReadWriteData(processPaymentLambda);
    reservationsTable.grantReadWriteData(rollbackReserveFundsLambda);

    // Load state machine definition
    const stateMachineDefinition = sfn.DefinitionBody.fromFile(
      'state-machines/payment-saga.json'
    );

    // Create state machine
    const stateMachine = new sfn.StateMachine(this, 'PaymentSagaStateMachine', {
      stateMachineName: 'payment-saga',
      definitionBody: stateMachineDefinition,
      timeout: cdk.Duration.minutes(5),
      tracingEnabled: true
    });

    // Grant state machine permissions
    reserveFundsLambda.grantInvoke(stateMachine);
    validatePaymentLambda.grantInvoke(stateMachine);
    processPaymentLambda.grantInvoke(stateMachine);
    fraudDetectionLambda.grantInvoke(stateMachine);
    rollbackReserveFundsLambda.grantInvoke(stateMachine);
    
    transactionsTable.grantReadWriteData(stateMachine);
    notificationTopic.grantPublish(stateMachine);
    approvalQueue.grantSendMessages(stateMachine);

    // Output
    new cdk.CfnOutput(this, 'StateMachineArn', {
      value: stateMachine.stateMachineArn,
      description: 'Payment Saga State Machine ARN'
    });
  }
}
```

---

### 📊 Testing the Saga

```javascript
// test/saga-test.js
const { SFNClient, StartExecutionCommand, DescribeExecutionCommand } = require('@aws-sdk/client-sfn');

const sfnClient = new SFNClient({ region: 'us-east-1' });

async function testSuccessfulPayment() {
  console.log('Testing successful payment...');
  
  const input = {
    transactionId: `txn-${Date.now()}`,
    fromAccount: 'ACC-12345',
    toAccount: 'ACC-67890',
    amount: 1000,
    currency: 'AED'
  };
  
  const response = await sfnClient.send(new StartExecutionCommand({
    stateMachineArn: 'arn:aws:states:us-east-1:123456789012:stateMachine:payment-saga',
    input: JSON.stringify(input)
  }));
  
  console.log('Execution started:', response.executionArn);
  
  // Wait and check execution status
  await sleep(5000);
  
  const execution = await sfnClient.send(new DescribeExecutionCommand({
    executionArn: response.executionArn
  }));
  
  console.log('Execution status:', execution.status);
  console.log('Output:', execution.output);
}

async function testFailedPayment() {
  console.log('Testing failed payment (insufficient funds)...');
  
  const input = {
    transactionId: `txn-${Date.now()}`,
    fromAccount: 'ACC-12345',
    toAccount: 'ACC-67890',
    amount: 1000000, // Very large amount to trigger failure
    currency: 'AED'
  };
  
  const response = await sfnClient.send(new StartExecutionCommand({
    stateMachineArn: 'arn:aws:states:us-east-1:123456789012:stateMachine:payment-saga',
    input: JSON.stringify(input)
  }));
  
  console.log('Execution started:', response.executionArn);
  
  // Wait and check execution status
  await sleep(5000);
  
  const execution = await sfnClient.send(new DescribeExecutionCommand({
    executionArn: response.executionArn
  }));
  
  console.log('Execution status:', execution.status);
  console.log('Error:', execution.cause);
}

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// Run tests
(async () => {
  await testSuccessfulPayment();
  await testFailedPayment();
})();
```

---

### 🎓 Interview Discussion Points - Q11

**Q1: What is the Saga pattern and when should you use it?**

**A**: The Saga pattern manages distributed transactions across microservices by breaking them into local transactions with compensating actions. Use it when:
- You need ACID-like guarantees across services
- Two-phase commit is not feasible (microservices with different databases)
- You can tolerate eventual consistency
- Failures need graceful handling with rollback

**Q2: What's the difference between orchestration and choreography in Saga?**

**A**:
- **Orchestration** (Step Functions): Central coordinator directs all services. Easier to understand and debug, but creates a single point of failure.
- **Choreography** (EventBridge): Services react to events independently. More decoupled, but harder to track overall flow.

**Q3: How do you ensure idempotency in Saga compensation?**

**A**:
- Use unique transaction IDs
- Check current state before compensating
- Store compensation status in database
- Handle "already rolled back" gracefully
- Use conditional updates in DynamoDB

**Q4: What are the limitations of Step Functions for Saga?**

**A**:
- **Execution history size**: 25,000 events max
- **Execution time**: 1 year max (but typically much shorter)
- **State size**: 256 KB per state
- **Cost**: $0.025 per 1,000 state transitions

**Q5: How do you handle partial failures in Saga?**

**A**:
- Define compensation logic for each step
- Use try-catch blocks in state machine
- Log all failures for auditing
- Send notifications on rollback
- Retry transient failures before compensating

---

## Question 12: EventBridge Event-Driven Architecture

### 📋 Question Statement

Design a comprehensive event-driven architecture for Emirates NBD using Amazon EventBridge. Include:
- Event schema registry and versioning
- Cross-service communication patterns
- Event replay and archiving
- Dead letter queues and error handling
- Multi-account event routing

---

### 🏗️ Event-Driven Architecture Overview

```
┌────────────────────────────────────────────────────────────────────────────┐
│               EMIRATES NBD EVENT-DRIVEN ARCHITECTURE                        │
└────────────────────────────────────────────────────────────────────────────┘

                          ┌──────────────────────┐
                          │   Event Producers    │
                          └──────────┬───────────┘
                                     │
                 ┌───────────────────┼───────────────────┐
                 │                   │                   │
                 ▼                   ▼                   ▼
         ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
         │   Account    │    │ Transaction  │    │   Payment    │
         │   Service    │    │   Service    │    │   Service    │
         │   (ECS)      │    │   (Lambda)   │    │   (EKS)      │
         └──────┬───────┘    └──────┬───────┘    └──────┬───────┘
                │                   │                   │
                └───────────────────┼───────────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │    Amazon EventBridge Bus     │
                    │    (banking-event-bus)        │
                    │                               │
                    │  • Event Schema Registry      │
                    │  • Event Archive (365 days)   │
                    │  • Content Filtering          │
                    └───────────────┬───────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────┐           ┌───────────────┐         ┌───────────────┐
│  Event Rules  │           │  Event Rules  │         │  Event Rules  │
│  (Routing)    │           │  (Transform)  │         │  (Filter)     │
└───────┬───────┘           └───────┬───────┘         └───────┬───────┘
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────────────────────────────────────────────────────────────┐
│                         EVENT CONSUMERS                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │ Lambda   │  │   SQS    │  │ Kinesis  │  │ SNS      │            │
│  │ (Notify) │  │ (Process)│  │ (Stream) │  │ (Alert)  │            │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘            │
│                                                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │   ECS    │  │ Step Fns │  │  API GW  │  │   S3     │            │
│  │(Webhook) │  │  (Saga)  │  │ (Webhook)│  │ (Archive)│            │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘            │
└───────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │    Cross-Account Routing      │
                    │                               │
                    │  ┌─────────────────────────┐  │
                    │  │  Analytics Account      │  │
                    │  │  (Data Lake, Athena)    │  │
                    │  └─────────────────────────┘  │
                    │  ┌─────────────────────────┐  │
                    │  │  Compliance Account     │  │
                    │  │  (Audit Logs, S3)       │  │
                    │  └─────────────────────────┘  │
                    └───────────────────────────────┘
```

---

### 📋 Event Schema Registry

```json
{
  "schemas": [
    {
      "schemaName": "AccountCreated",
      "schemaVersion": "1",
      "description": "Published when a new bank account is created",
      "schema": {
        "$schema": "http://json-schema.org/draft-07/schema#",
        "type": "object",
        "required": ["accountId", "customerId", "accountType", "currency", "timestamp"],
        "properties": {
          "accountId": {
            "type": "string",
            "pattern": "^ACC-[0-9]{8}$",
            "description": "Unique account identifier"
          },
          "customerId": {
            "type": "string",
            "pattern": "^CUST-[0-9]{8}$",
            "description": "Customer identifier"
          },
          "accountType": {
            "type": "string",
            "enum": ["SAVINGS", "CURRENT", "FIXED_DEPOSIT", "CREDIT_CARD"],
            "description": "Type of bank account"
          },
          "currency": {
            "type": "string",
            "enum": ["AED", "USD", "EUR", "GBP"],
            "description": "Account currency"
          },
          "initialBalance": {
            "type": "number",
            "minimum": 0,
            "description": "Initial account balance"
          },
          "timestamp": {
            "type": "string",
            "format": "date-time",
            "description": "Account creation timestamp"
          },
          "branchCode": {
            "type": "string",
            "description": "Branch where account was created"
          },
          "metadata": {
            "type": "object",
            "properties": {
              "createdBy": {"type": "string"},
              "source": {"type": "string"}
            }
          }
        }
      }
    },
    {
      "schemaName": "TransactionCompleted",
      "schemaVersion": "2",
      "description": "Published when a transaction is successfully completed",
      "schema": {
        "$schema": "http://json-schema.org/draft-07/schema#",
        "type": "object",
        "required": ["transactionId", "fromAccount", "toAccount", "amount", "currency", "status", "timestamp"],
        "properties": {
          "transactionId": {
            "type": "string",
            "pattern": "^TXN-[0-9]{10}$"
          },
          "fromAccount": {
            "type": "string",
            "pattern": "^ACC-[0-9]{8}$"
          },
          "toAccount": {
            "type": "string",
            "pattern": "^ACC-[0-9]{8}$"
          },
          "amount": {
            "type": "number",
            "minimum": 0.01,
            "description": "Transaction amount"
          },
          "currency": {
            "type": "string",
            "enum": ["AED", "USD", "EUR", "GBP"]
          },
          "transactionType": {
            "type": "string",
            "enum": ["TRANSFER", "PAYMENT", "WITHDRAWAL", "DEPOSIT"]
          },
          "status": {
            "type": "string",
            "enum": ["COMPLETED", "PENDING", "FAILED"]
          },
          "timestamp": {
            "type": "string",
            "format": "date-time"
          },
          "description": {
            "type": "string",
            "maxLength": 500
          },
          "fees": {
            "type": "number",
            "minimum": 0
          },
          "metadata": {
            "type": "object",
            "properties": {
              "channel": {"type": "string"},
              "ipAddress": {"type": "string"},
              "deviceId": {"type": "string"}
            }
          }
        }
      }
    },
    {
      "schemaName": "FraudDetected",
      "schemaVersion": "1",
      "description": "Published when potential fraud is detected",
      "schema": {
        "$schema": "http://json-schema.org/draft-07/schema#",
        "type": "object",
        "required": ["alertId", "transactionId", "fraudScore", "riskLevel", "timestamp"],
        "properties": {
          "alertId": {
            "type": "string",
            "pattern": "^ALERT-[0-9]{10}$"
          },
          "transactionId": {
            "type": "string"
          },
          "accountId": {
            "type": "string"
          },
          "fraudScore": {
            "type": "number",
            "minimum": 0,
            "maximum": 100
          },
          "riskLevel": {
            "type": "string",
            "enum": ["LOW", "MEDIUM", "HIGH", "CRITICAL"]
          },
          "fraudIndicators": {
            "type": "array",
            "items": {
              "type": "string",
              "enum": [
                "UNUSUAL_LOCATION",
                "UNUSUAL_AMOUNT",
                "VELOCITY_CHECK_FAILED",
                "BLACKLISTED_IP",
                "DEVICE_FINGERPRINT_MISMATCH"
              ]
            }
          },
          "timestamp": {
            "type": "string",
            "format": "date-time"
          },
          "recommendation": {
            "type": "string",
            "enum": ["ALLOW", "REVIEW", "BLOCK"]
          }
        }
      }
    }
  ]
}
```

---

### 🔧 EventBridge Infrastructure (CDK)

```typescript
// infrastructure/cdk/eventbridge-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as sqs from 'aws-cdk-lib/aws-sqs';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as kinesis from 'aws-cdk-lib/aws-kinesis';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as iam from 'aws-cdk-lib/aws-iam';

export class EventBridgeStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // ============================================
    // 1. EVENT BUS
    // ============================================
    const eventBus = new events.EventBus(this, 'BankingEventBus', {
      eventBusName: 'banking-event-bus'
    });

    // Archive events for replay (365 days)
    new events.Archive(this, 'EventArchive', {
      sourceEventBus: eventBus,
      archiveName: 'banking-event-archive',
      eventPattern: {
        source: events.Match.prefix('account.'),
        detailType: events.Match.anyOf([
          'AccountCreated',
          'AccountUpdated',
          'AccountClosed'
        ])
      },
      retention: cdk.Duration.days(365)
    });

    // ============================================
    // 2. DEAD LETTER QUEUE
    // ============================================
    const dlq = new sqs.Queue(this, 'EventDLQ', {
      queueName: 'event-processing-dlq',
      retentionPeriod: cdk.Duration.days(14)
    });

    // ============================================
    // 3. LAMBDA CONSUMERS
    // ============================================

    // Notification Lambda
    const notificationLambda = new lambda.Function(this, 'NotificationHandler', {
      functionName: 'event-notification-handler',
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambdas/notification-handler'),
      timeout: cdk.Duration.seconds(30),
      deadLetterQueue: dlq,
      retryAttempts: 2
    });

    // Fraud Detection Lambda
    const fraudLambda = new lambda.Function(this, 'FraudHandler', {
      functionName: 'event-fraud-handler',
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambdas/fraud-handler'),
      timeout: cdk.Duration.seconds(10),
      memorySize: 1024
    });

    // Analytics Lambda
    const analyticsLambda = new lambda.Function(this, 'AnalyticsHandler', {
      functionName: 'event-analytics-handler',
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambdas/analytics-handler'),
      timeout: cdk.Duration.seconds(60)
    });

    // ============================================
    // 4. SQS QUEUES FOR PROCESSING
    // ============================================

    // Transaction processing queue
    const transactionQueue = new sqs.Queue(this, 'TransactionQueue', {
      queueName: 'transaction-processing-queue',
      visibilityTimeout: cdk.Duration.seconds(300),
      deadLetterQueue: {
        queue: dlq,
        maxReceiveCount: 3
      }
    });

    // Account update queue
    const accountQueue = new sqs.Queue(this, 'AccountQueue', {
      queueName: 'account-update-queue',
      visibilityTimeout: cdk.Duration.seconds(300)
    });

    // ============================================
    // 5. SNS TOPICS FOR NOTIFICATIONS
    // ============================================

    const alertTopic = new sns.Topic(this, 'AlertTopic', {
      topicName: 'fraud-alerts',
      displayName: 'Fraud Detection Alerts'
    });

    // ============================================
    // 6. KINESIS STREAM FOR ANALYTICS
    // ============================================

    const analyticsStream = new kinesis.Stream(this, 'AnalyticsStream', {
      streamName: 'banking-analytics-stream',
      shardCount: 2,
      retentionPeriod: cdk.Duration.days(7)
    });

    // ============================================
    // 7. S3 BUCKET FOR ARCHIVING
    // ============================================

    const archiveBucket = new s3.Bucket(this, 'EventArchiveBucket', {
      bucketName: `banking-event-archive-${this.account}`,
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      lifecycleRules: [
        {
          transitions: [
            {
              storageClass: s3.StorageClass.GLACIER,
              transitionAfter: cdk.Duration.days(90)
            }
          ]
        }
      ]
    });

    // ============================================
    // 8. EVENT RULES
    // ============================================

    // Rule 1: Account Created -> Multiple Targets
    new events.Rule(this, 'AccountCreatedRule', {
      eventBus,
      ruleName: 'account-created-rule',
      description: 'Route account creation events to multiple consumers',
      eventPattern: {
        source: ['account.service'],
        detailType: ['AccountCreated']
      },
      targets: [
        new targets.LambdaFunction(notificationLambda),
        new targets.SqsQueue(accountQueue),
        new targets.KinesisStream(analyticsStream)
      ]
    });

    // Rule 2: Transaction Completed -> Fraud Detection + Notification
    new events.Rule(this, 'TransactionCompletedRule', {
      eventBus,
      ruleName: 'transaction-completed-rule',
      eventPattern: {
        source: ['transaction.service'],
        detailType: ['TransactionCompleted'],
        detail: {
          amount: [{ numeric: ['>', 10000] }] // Only large transactions
        }
      },
      targets: [
        new targets.LambdaFunction(fraudLambda),
        new targets.LambdaFunction(notificationLambda),
        new targets.SqsQueue(transactionQueue)
      ]
    });

    // Rule 3: Fraud Detected -> Alert + Manual Review
    new events.Rule(this, 'FraudDetectedRule', {
      eventBus,
      ruleName: 'fraud-detected-rule',
      eventPattern: {
        source: ['fraud.service'],
        detailType: ['FraudDetected'],
        detail: {
          riskLevel: ['HIGH', 'CRITICAL']
        }
      },
      targets: [
        new targets.SnsTopic(alertTopic),
        new targets.LambdaFunction(notificationLambda, {
          deadLetterQueue: dlq,
          retryAttempts: 3
        })
      ]
    });

    // Rule 4: All Events -> Analytics Stream
    new events.Rule(this, 'AllEventsToAnalyticsRule', {
      eventBus,
      ruleName: 'all-events-analytics-rule',
      eventPattern: {
        source: events.Match.prefix('') // All sources
      },
      targets: [
        new targets.KinesisStream(analyticsStream),
        new targets.CloudWatchLogGroup(
          new cdk.aws_logs.LogGroup(this, 'EventLogGroup', {
            logGroupName: '/aws/events/banking'
          })
        )
      ]
    });

    // Rule 5: Archive to S3
    new events.Rule(this, 'ArchiveToS3Rule', {
      eventBus,
      ruleName: 'archive-to-s3-rule',
      eventPattern: {
        source: events.Match.prefix('')
      },
      targets: [
        new targets.LambdaFunction(
          new lambda.Function(this, 'S3ArchiveHandler', {
            functionName: 'event-s3-archiver',
            runtime: lambda.Runtime.NODEJS_20_X,
            handler: 'index.handler',
            code: lambda.Code.fromInline(`
              const { S3Client, PutObjectCommand } = require('@aws-sdk/client-s3');
              const s3 = new S3Client({});
              
              exports.handler = async (event) => {
                const timestamp = new Date().toISOString();
                const key = \`events/\${timestamp.split('T')[0]}/\${event.id}.json\`;
                
                await s3.send(new PutObjectCommand({
                  Bucket: process.env.BUCKET_NAME,
                  Key: key,
                  Body: JSON.stringify(event, null, 2),
                  ContentType: 'application/json'
                }));
                
                return { statusCode: 200 };
              };
            `),
            environment: {
              BUCKET_NAME: archiveBucket.bucketName
            }
          })
        )
      ]
    });

    // Grant permissions
    archiveBucket.grantWrite(
      iam.ServicePrincipal.fromStaticServicePrincipleName('lambda.amazonaws.com')
    );

    // ============================================
    // 9. CROSS-ACCOUNT EVENT ROUTING
    // ============================================

    // Rule to route to analytics account
    const analyticsAccountEventBus = events.EventBus.fromEventBusArn(
      this,
      'AnalyticsAccountBus',
      `arn:aws:events:us-east-1:987654321098:event-bus/analytics-event-bus`
    );

    new events.Rule(this, 'CrossAccountAnalyticsRule', {
      eventBus,
      ruleName: 'cross-account-analytics-rule',
      eventPattern: {
        source: events.Match.prefix(''),
        detailType: ['TransactionCompleted', 'AccountCreated']
      },
      targets: [new targets.EventBus(analyticsAccountEventBus)]
    });

    // ============================================
    // 10. OUTPUTS
    // ============================================

    new cdk.CfnOutput(this, 'EventBusArn', {
      value: eventBus.eventBusArn,
      description: 'Banking Event Bus ARN'
    });

    new cdk.CfnOutput(this, 'EventBusName', {
      value: eventBus.eventBusName,
      description: 'Banking Event Bus Name'
    });
  }
}
```

---

### 💻 Event Producer Implementation

```javascript
// services/account-service/src/events/publisher.js
const { EventBridgeClient, PutEventsCommand } = require('@aws-sdk/client-eventbridge');

class EventPublisher {
  constructor() {
    this.client = new EventBridgeClient({ region: 'us-east-1' });
    this.eventBusName = 'banking-event-bus';
  }

  async publishAccountCreated(account) {
    const event = {
      Source: 'account.service',
      DetailType: 'AccountCreated',
      Detail: JSON.stringify({
        accountId: account.accountId,
        customerId: account.customerId,
        accountType: account.accountType,
        currency: account.currency,
        initialBalance: account.initialBalance,
        timestamp: new Date().toISOString(),
        branchCode: account.branchCode,
        metadata: {
          createdBy: account.createdBy,
          source: 'mobile-app'
        }
      }),
      EventBusName: this.eventBusName
    };

    try {
      const response = await this.client.send(
        new PutEventsCommand({ Entries: [event] })
      );

      if (response.FailedEntryCount > 0) {
        console.error('Failed to publish event:', response.Entries[0].ErrorMessage);
        throw new Error('Event publication failed');
      }

      console.log('Event published successfully:', response.Entries[0].EventId);
      return response.Entries[0].EventId;
    } catch (error) {
      console.error('Error publishing event:', error);
      throw error;
    }
  }

  async publishTransactionCompleted(transaction) {
    const event = {
      Source: 'transaction.service',
      DetailType: 'TransactionCompleted',
      Detail: JSON.stringify({
        transactionId: transaction.transactionId,
        fromAccount: transaction.fromAccount,
        toAccount: transaction.toAccount,
        amount: transaction.amount,
        currency: transaction.currency,
        transactionType: transaction.transactionType,
        status: 'COMPLETED',
        timestamp: new Date().toISOString(),
        description: transaction.description,
        fees: transaction.fees,
        metadata: {
          channel: transaction.channel,
          ipAddress: transaction.ipAddress,
          deviceId: transaction.deviceId
        }
      }),
      EventBusName: this.eventBusName
    };

    try {
      const response = await this.client.send(
        new PutEventsCommand({ Entries: [event] })
      );

      if (response.FailedEntryCount > 0) {
        console.error('Failed to publish event:', response.Entries[0].ErrorMessage);
        throw new Error('Event publication failed');
      }

      return response.Entries[0].EventId;
    } catch (error) {
      console.error('Error publishing event:', error);
      throw error;
    }
  }

  async publishFraudDetected(fraudAlert) {
    const event = {
      Source: 'fraud.service',
      DetailType: 'FraudDetected',
      Detail: JSON.stringify({
        alertId: fraudAlert.alertId,
        transactionId: fraudAlert.transactionId,
        accountId: fraudAlert.accountId,
        fraudScore: fraudAlert.fraudScore,
        riskLevel: fraudAlert.riskLevel,
        fraudIndicators: fraudAlert.fraudIndicators,
        timestamp: new Date().toISOString(),
        recommendation: fraudAlert.recommendation
      }),
      EventBusName: this.eventBusName
    };

    try {
      const response = await this.client.send(
        new PutEventsCommand({ Entries: [event] })
      );

      if (response.FailedEntryCount > 0) {
        throw new Error('Event publication failed');
      }

      return response.Entries[0].EventId;
    } catch (error) {
      console.error('Error publishing fraud alert:', error);
      throw error;
    }
  }

  // Batch event publishing for high throughput
  async publishBatch(events) {
    const eventEntries = events.map(event => ({
      Source: event.source,
      DetailType: event.detailType,
      Detail: JSON.stringify(event.detail),
      EventBusName: this.eventBusName
    }));

    // EventBridge supports max 10 events per batch
    const batches = [];
    for (let i = 0; i < eventEntries.length; i += 10) {
      batches.push(eventEntries.slice(i, i + 10));
    }

    const results = [];
    for (const batch of batches) {
      const response = await this.client.send(
        new PutEventsCommand({ Entries: batch })
      );
      results.push(response);
    }

    return results;
  }
}

module.exports = EventPublisher;

// Example usage in Express.js route
/*
const EventPublisher = require('./events/publisher');
const publisher = new EventPublisher();

app.post('/accounts', async (req, res) => {
  try {
    const account = await createAccount(req.body);
    await publisher.publishAccountCreated(account);
    res.status(201).json(account);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
*/
```

---

### 📥 Event Consumer Implementation

```javascript
// lambdas/notification-handler/index.js
const { SNSClient, PublishCommand } = require('@aws-sdk/client-sns');
const { SESClient, SendEmailCommand } = require('@aws-sdk/client-ses');

const snsClient = new SNSClient({ region: 'us-east-1' });
const sesClient = new SESClient({ region: 'us-east-1' });

exports.handler = async (event) => {
  console.log('Received event:', JSON.stringify(event, null, 2));

  const { source, 'detail-type': detailType, detail } = event;

  try {
    switch (detailType) {
      case 'AccountCreated':
        await handleAccountCreated(detail);
        break;
      case 'TransactionCompleted':
        await handleTransactionCompleted(detail);
        break;
      case 'FraudDetected':
        await handleFraudDetected(detail);
        break;
      default:
        console.log('Unhandled event type:', detailType);
    }

    return { statusCode: 200, body: 'Event processed successfully' };
  } catch (error) {
    console.error('Error processing event:', error);
    throw error; // Will trigger DLQ
  }
};

async function handleAccountCreated(detail) {
  const { accountId, customerId, accountType } = detail;

  // Send welcome email
  await sesClient.send(new SendEmailCommand({
    Source: 'noreply@emiratesnbd.com',
    Destination: {
      ToAddresses: [await getCustomerEmail(customerId)]
    },
    Message: {
      Subject: { Data: 'Welcome to Emirates NBD' },
      Body: {
        Text: {
          Data: `Your ${accountType} account ${accountId} has been successfully created.`
        }
      }
    }
  }));

  console.log('Welcome email sent for account:', accountId);
}

async function handleTransactionCompleted(detail) {
  const { transactionId, fromAccount, toAccount, amount, currency } = detail;

  // Send SMS notification
  await snsClient.send(new PublishCommand({
    PhoneNumber: await getAccountPhoneNumber(fromAccount),
    Message: `Transaction ${transactionId}: ${currency} ${amount} sent to ${toAccount}`
  }));

  console.log('Transaction notification sent:', transactionId);
}

async function handleFraudDetected(detail) {
  const { alertId, transactionId, riskLevel, fraudScore } = detail;

  // Send urgent alert to security team
  await snsClient.send(new PublishCommand({
    TopicArn: 'arn:aws:sns:us-east-1:123456789012:fraud-alerts',
    Subject: `Fraud Alert: ${riskLevel}`,
    Message: JSON.stringify({
      alertId,
      transactionId,
      riskLevel,
      fraudScore,
      timestamp: new Date().toISOString()
    }, null, 2)
  }));

  console.log('Fraud alert sent:', alertId);
}

async function getCustomerEmail(customerId) {
  // Query customer database
  return 'customer@example.com'; // Placeholder
}

async function getAccountPhoneNumber(accountId) {
  // Query account database
  return '+971501234567'; // Placeholder
}
```

---

### 🔄 Event Replay Mechanism

```javascript
// scripts/replay-events.js
const { EventBridgeClient, StartReplayCommand, DescribeReplayCommand } = require('@aws-sdk/client-eventbridge');

const client = new EventBridgeClient({ region: 'us-east-1' });

async function replayEvents({ archiveName, startTime, endTime, destinationBus }) {
  console.log('Starting event replay...');
  
  const replayName = `replay-${Date.now()}`;
  
  const response = await client.send(new StartReplayCommand({
    ReplayName: replayName,
    EventSourceArn: `arn:aws:events:us-east-1:123456789012:archive/${archiveName}`,
    EventStartTime: new Date(startTime),
    EventEndTime: new Date(endTime),
    Destination: {
      Arn: `arn:aws:events:us-east-1:123456789012:event-bus/${destinationBus}`
    }
  }));
  
  console.log('Replay started:', response.ReplayArn);
  
  // Monitor replay progress
  let replayComplete = false;
  while (!replayComplete) {
    await sleep(5000);
    
    const status = await client.send(new DescribeReplayCommand({
      ReplayName: replayName
    }));
    
    console.log('Replay status:', status.State);
    console.log('Events replayed:', status.EventLastReplayedTime);
    
    if (status.State === 'COMPLETED') {
      replayComplete = true;
      console.log('Replay completed successfully');
    } else if (status.State === 'FAILED') {
      throw new Error(`Replay failed: ${status.StateReason}`);
    }
  }
}

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// Example: Replay last 24 hours of AccountCreated events
(async () => {
  const now = new Date();
  const yesterday = new Date(now.getTime() - 24 * 60 * 60 * 1000);
  
  await replayEvents({
    archiveName: 'banking-event-archive',
    startTime: yesterday.toISOString(),
    endTime: now.toISOString(),
    destinationBus: 'banking-event-bus'
  });
})();
```

---

### 📊 Event Monitoring Dashboard

```javascript
// monitoring/event-metrics.js
const { CloudWatchClient, GetMetricStatisticsCommand } = require('@aws-sdk/client-cloudwatch');

const cloudwatch = new CloudWatchClient({ region: 'us-east-1' });

class EventMetrics {
  async getEventPublishRate(eventBusName, hours = 1) {
    const endTime = new Date();
    const startTime = new Date(endTime.getTime() - hours * 60 * 60 * 1000);
    
    const response = await cloudwatch.send(new GetMetricStatisticsCommand({
      Namespace: 'AWS/Events',
      MetricName: 'Invocations',
      Dimensions: [
        {
          Name: 'EventBusName',
          Value: eventBusName
        }
      ],
      StartTime: startTime,
      EndTime: endTime,
      Period: 300, // 5 minutes
      Statistics: ['Sum', 'Average']
    }));
    
    return response.Datapoints;
  }

  async getFailedInvocations(ruleName, hours = 1) {
    const endTime = new Date();
    const startTime = new Date(endTime.getTime() - hours * 60 * 60 * 1000);
    
    const response = await cloudwatch.send(new GetMetricStatisticsCommand({
      Namespace: 'AWS/Events',
      MetricName: 'FailedInvocations',
      Dimensions: [
        {
          Name: 'RuleName',
          Value: ruleName
        }
      ],
      StartTime: startTime,
      EndTime: endTime,
      Period: 300,
      Statistics: ['Sum']
    }));
    
    return response.Datapoints;
  }

  async getThrottledRules(eventBusName, hours = 1) {
    const endTime = new Date();
    const startTime = new Date(endTime.getTime() - hours * 60 * 60 * 1000);
    
    const response = await cloudwatch.send(new GetMetricStatisticsCommand({
      Namespace: 'AWS/Events',
      MetricName: 'ThrottledRules',
      Dimensions: [
        {
          Name: 'EventBusName',
          Value: eventBusName
        }
      ],
      StartTime: startTime,
      EndTime: endTime,
      Period: 300,
      Statistics: ['Sum']
    }));
    
    return response.Datapoints;
  }

  async generateReport() {
    console.log('Event Metrics Report');
    console.log('===================\n');
    
    const publishRate = await this.getEventPublishRate('banking-event-bus');
    console.log('Event Publish Rate (last hour):');
    console.log(JSON.stringify(publishRate, null, 2));
    
    const failures = await this.getFailedInvocations('transaction-completed-rule');
    console.log('\nFailed Invocations:');
    console.log(JSON.stringify(failures, null, 2));
    
    const throttled = await this.getThrottledRules('banking-event-bus');
    console.log('\nThrottled Rules:');
    console.log(JSON.stringify(throttled, null, 2));
  }
}

// Run report
(async () => {
  const metrics = new EventMetrics();
  await metrics.generateReport();
})();
```

---

### 🎓 Interview Discussion Points - Q12

**Q1: When should you use EventBridge vs SNS vs SQS?**

**A**:
- **EventBridge**: Complex routing, content-based filtering, schema registry, cross-account communication
- **SNS**: Simple pub/sub, fan-out to multiple subscribers, mobile push notifications
- **SQS**: Decoupling, buffering, guaranteed delivery, work queues

**Q2: How do you handle event versioning in EventBridge?**

**A**:
- Use schema registry with version numbers
- Include version field in event payload
- Support multiple versions simultaneously (consumers check version)
- Deprecate old versions gradually
- Use event transformers to upgrade/downgrade

**Q3: What are the limits of EventBridge?**

**A**:
- **PutEvents**: 10,000 requests/sec per account (soft limit)
- **Event size**: 256 KB
- **Targets per rule**: 5
- **Rules per event bus**: 300 (soft limit)
- **Archive retention**: Up to indefinite

**Q4: How do you ensure event delivery reliability?**

**A**:
- Use DLQ for failed invocations
- Enable retry with exponential backoff
- Monitor FailedInvocations metric
- Archive events for replay
- Use idempotent consumers

**Q5: How do you test event-driven architectures?**

**A**:
- Unit test event producers/consumers separately
- Use EventBridge schema validation
- Create test event bus for integration tests
- Replay archived events in test environment
- Monitor event flow with X-Ray

---

**End of File 6**

