# Advanced Architecture Patterns

## Question 59: CQRS, Event Sourcing, and Saga Pattern Integration

### 📋 Question Statement

Implement advanced architecture patterns for Emirates NBD combining CQRS, Event Sourcing, and Saga orchestration for complex banking workflows.

---

### 🏗️ CQRS + Event Sourcing Architecture

```
┌───────────────────────────────────────────────────────────────────────┐
│                    CQRS + EVENT SOURCING + SAGA                        │
└───────────────────────────────────────────────────────────────────────┘

        ┌──────────────┐                    ┌──────────────┐
        │   Client     │                    │   Client     │
        │ (Mobile/Web) │                    │ (Mobile/Web) │
        └──────┬───────┘                    └──────┬───────┘
               │                                   │
        [COMMANDS]                           [QUERIES]
               │                                   │
               v                                   v
     ┌─────────────────┐              ┌──────────────────┐
     │ Command Handler │              │  Query Handler   │
     │   (Write Side)  │              │   (Read Side)    │
     └────────┬────────┘              └────────┬─────────┘
              │                                 │
              │ Validate + Execute              │ Read from
              │                                 │ Materialized View
              v                                 │
     ┌─────────────────┐                       │
     │  Event Store    │───────────────────────┘
     │   (DynamoDB)    │      Projections
     └────────┬────────┘      (Async)
              │
              │ Publish
              v
     ┌─────────────────┐
     │   EventBridge   │
     └────────┬────────┘
              │
              ├─────────────────┐
              │                 │
              v                 v
     ┌────────────┐    ┌──────────────┐
     │ Saga       │    │ Projections  │
     │Orchestrator│    │  (DynamoDB)  │
     └────────────┘    └──────────────┘
```

### 📦 Event Sourcing Stack CDK

```typescript
// infrastructure/cdk/event-sourcing-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';
import * as sqs from 'aws-cdk-lib/aws-sqs';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';

export class EventSourcingStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Event Store (DynamoDB)
    const eventStore = new dynamodb.Table(this, 'EventStore', {
      tableName: 'banking-event-store',
      partitionKey: { name: 'aggregateId', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'version', type: dynamodb.AttributeType.NUMBER },
      stream: dynamodb.StreamViewType.NEW_IMAGE,
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      pointInTimeRecovery: true
    });

    // Read Model (Materialized Views)
    const accountView = new dynamodb.Table(this, 'AccountView', {
      tableName: 'account-view',
      partitionKey: { name: 'accountId', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST
    });

    const transactionView = new dynamodb.Table(this, 'TransactionView', {
      tableName: 'transaction-view',
      partitionKey: { name: 'accountId', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'timestamp', type: dynamodb.AttributeType.NUMBER },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST
    });

    transactionView.addGlobalSecondaryIndex({
      indexName: 'StatusIndex',
      partitionKey: { name: 'status', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'timestamp', type: dynamodb.AttributeType.NUMBER }
    });

    // Saga State Store
    const sagaStore = new dynamodb.Table(this, 'SagaStore', {
      tableName: 'saga-state-store',
      partitionKey: { name: 'sagaId', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST
    });

    // Dead Letter Queue
    const dlq = new sqs.Queue(this, 'EventDLQ', {
      queueName: 'event-processing-dlq',
      retentionPeriod: cdk.Duration.days(14)
    });

    // EventBridge Bus
    const eventBus = new events.EventBus(this, 'BankingEventBus', {
      eventBusName: 'banking-events'
    });

    // Command Handler Lambda
    const commandHandler = new lambda.Function(this, 'CommandHandler', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/command-handler'),
      timeout: cdk.Duration.seconds(30),
      environment: {
        EVENT_STORE_TABLE: eventStore.tableName,
        EVENT_BUS_NAME: eventBus.eventBusName
      }
    });

    eventStore.grantReadWriteData(commandHandler);
    eventBus.grantPutEventsTo(commandHandler);

    // Query Handler Lambda
    const queryHandler = new lambda.Function(this, 'QueryHandler', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/query-handler'),
      timeout: cdk.Duration.seconds(10),
      environment: {
        ACCOUNT_VIEW_TABLE: accountView.tableName,
        TRANSACTION_VIEW_TABLE: transactionView.tableName
      }
    });

    accountView.grantReadData(queryHandler);
    transactionView.grantReadData(queryHandler);

    // Projection Builder Lambda
    const projectionBuilder = new lambda.Function(this, 'ProjectionBuilder', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/projection-builder'),
      timeout: cdk.Duration.minutes(5),
      environment: {
        ACCOUNT_VIEW_TABLE: accountView.tableName,
        TRANSACTION_VIEW_TABLE: transactionView.tableName
      }
    });

    accountView.grantWriteData(projectionBuilder);
    transactionView.grantWriteData(projectionBuilder);

    // Saga Orchestrator Lambda
    const sagaOrchestrator = new lambda.Function(this, 'SagaOrchestrator', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/saga-orchestrator'),
      timeout: cdk.Duration.minutes(5),
      environment: {
        SAGA_STORE_TABLE: sagaStore.tableName,
        EVENT_BUS_NAME: eventBus.eventBusName
      }
    });

    sagaStore.grantReadWriteData(sagaOrchestrator);
    eventBus.grantPutEventsTo(sagaOrchestrator);

    // EventBridge Rules
    new events.Rule(this, 'ProjectionRule', {
      eventBus: eventBus,
      eventPattern: {
        source: ['banking.accounts', 'banking.transactions']
      },
      targets: [new targets.LambdaFunction(projectionBuilder, {
        deadLetterQueue: dlq,
        retryAttempts: 3
      })]
    });

    new events.Rule(this, 'SagaRule', {
      eventBus: eventBus,
      eventPattern: {
        source: ['banking.transfers'],
        detailType: ['TransferInitiated', 'TransferCompleted', 'TransferFailed']
      },
      targets: [new targets.LambdaFunction(sagaOrchestrator, {
        deadLetterQueue: dlq,
        retryAttempts: 3
      })]
    });

    // API Gateway
    const api = new apigateway.RestApi(this, 'CQRS-API', {
      restApiName: 'Banking CQRS API',
      defaultCorsPreflightOptions: {
        allowOrigins: apigateway.Cors.ALL_ORIGINS,
        allowMethods: apigateway.Cors.ALL_METHODS
      }
    });

    // Commands endpoint
    const commands = api.root.addResource('commands');
    commands.addMethod('POST', new apigateway.LambdaIntegration(commandHandler));

    // Queries endpoint
    const queries = api.root.addResource('queries');
    queries.addMethod('GET', new apigateway.LambdaIntegration(queryHandler));

    // Outputs
    new cdk.CfnOutput(this, 'ApiUrl', {
      value: api.url,
      description: 'CQRS API URL'
    });

    new cdk.CfnOutput(this, 'EventBusArn', {
      value: eventBus.eventBusArn,
      description: 'EventBridge Bus ARN'
    });
  }
}
```

### ⚙️ Command Handler Implementation

```typescript
// lambda/command-handler/index.ts
import { DynamoDBClient, PutItemCommand, QueryCommand } from '@aws-sdk/client-dynamodb';
import { EventBridgeClient, PutEventsCommand } from '@aws-sdk/client-eventbridge';
import { v4 as uuidv4 } from 'uuid';

const dynamodb = new DynamoDBClient({});
const eventBridge = new EventBridgeClient({});

const EVENT_STORE_TABLE = process.env.EVENT_STORE_TABLE!;
const EVENT_BUS_NAME = process.env.EVENT_BUS_NAME!;

interface Command {
  type: string;
  aggregateId: string;
  data: any;
}

export const handler = async (event: any) => {
  const command: Command = JSON.parse(event.body);
  console.log('Processing command:', command);

  try {
    // Validate command
    validateCommand(command);

    // Load aggregate (rebuild from events)
    const aggregate = await loadAggregate(command.aggregateId);

    // Execute command (business logic)
    const events = executeCommand(command, aggregate);

    // Save events to event store
    await saveEvents(command.aggregateId, events);

    // Publish events to EventBridge
    await publishEvents(events);

    return {
      statusCode: 200,
      body: JSON.stringify({
        success: true,
        aggregateId: command.aggregateId,
        eventsPublished: events.length
      })
    };
  } catch (error) {
    console.error('Command failed:', error);
    return {
      statusCode: 400,
      body: JSON.stringify({ error: (error as Error).message })
    };
  }
};

function validateCommand(command: Command): void {
  if (!command.type || !command.aggregateId) {
    throw new Error('Invalid command: type and aggregateId required');
  }
}

async function loadAggregate(aggregateId: string): Promise<any> {
  const response = await dynamodb.send(
    new QueryCommand({
      TableName: EVENT_STORE_TABLE,
      KeyConditionExpression: 'aggregateId = :id',
      ExpressionAttributeValues: {
        ':id': { S: aggregateId }
      },
      ScanIndexForward: true
    })
  );

  // Rebuild state from events
  const aggregate = { id: aggregateId, version: 0, balance: 0, status: 'ACTIVE' };
  
  for (const item of response.Items || []) {
    const eventType = item.eventType.S!;
    const eventData = JSON.parse(item.eventData.S!);
    
    applyEvent(aggregate, eventType, eventData);
    aggregate.version = parseInt(item.version.N!);
  }

  return aggregate;
}

function applyEvent(aggregate: any, eventType: string, eventData: any): void {
  switch (eventType) {
    case 'AccountCreated':
      aggregate.balance = eventData.initialBalance;
      aggregate.currency = eventData.currency;
      break;
    case 'MoneyDeposited':
      aggregate.balance += eventData.amount;
      break;
    case 'MoneyWithdrawn':
      aggregate.balance -= eventData.amount;
      break;
    case 'AccountClosed':
      aggregate.status = 'CLOSED';
      break;
  }
}

function executeCommand(command: Command, aggregate: any): any[] {
  const events: any[] = [];

  switch (command.type) {
    case 'CreateAccount':
      if (aggregate.version > 0) {
        throw new Error('Account already exists');
      }
      events.push({
        eventType: 'AccountCreated',
        aggregateId: command.aggregateId,
        eventData: command.data,
        timestamp: Date.now()
      });
      break;

    case 'DepositMoney':
      if (aggregate.status !== 'ACTIVE') {
        throw new Error('Account is not active');
      }
      if (command.data.amount <= 0) {
        throw new Error('Amount must be positive');
      }
      events.push({
        eventType: 'MoneyDeposited',
        aggregateId: command.aggregateId,
        eventData: command.data,
        timestamp: Date.now()
      });
      break;

    case 'WithdrawMoney':
      if (aggregate.status !== 'ACTIVE') {
        throw new Error('Account is not active');
      }
      if (command.data.amount <= 0) {
        throw new Error('Amount must be positive');
      }
      if (aggregate.balance < command.data.amount) {
        throw new Error('Insufficient funds');
      }
      events.push({
        eventType: 'MoneyWithdrawn',
        aggregateId: command.aggregateId,
        eventData: command.data,
        timestamp: Date.now()
      });
      break;

    case 'CloseAccount':
      if (aggregate.status !== 'ACTIVE') {
        throw new Error('Account already closed');
      }
      if (aggregate.balance > 0) {
        throw new Error('Cannot close account with balance');
      }
      events.push({
        eventType: 'AccountClosed',
        aggregateId: command.aggregateId,
        eventData: {},
        timestamp: Date.now()
      });
      break;

    default:
      throw new Error(`Unknown command type: ${command.type}`);
  }

  return events;
}

async function saveEvents(aggregateId: string, events: any[]): Promise<void> {
  for (let i = 0; i < events.length; i++) {
    const event = events[i];
    
    await dynamodb.send(
      new PutItemCommand({
        TableName: EVENT_STORE_TABLE,
        Item: {
          aggregateId: { S: aggregateId },
          version: { N: String(event.timestamp) },
          eventType: { S: event.eventType },
          eventData: { S: JSON.stringify(event.eventData) },
          timestamp: { N: String(event.timestamp) },
          eventId: { S: uuidv4() }
        }
      })
    );
  }
}

async function publishEvents(events: any[]): Promise<void> {
  const entries = events.map(event => ({
    Source: 'banking.accounts',
    DetailType: event.eventType,
    Detail: JSON.stringify(event),
    EventBusName: EVENT_BUS_NAME
  }));

  await eventBridge.send(
    new PutEventsCommand({ Entries: entries })
  );
}
```

### 📊 Query Handler Implementation

```typescript
// lambda/query-handler/index.ts
import { DynamoDBClient, GetItemCommand, QueryCommand } from '@aws-sdk/client-dynamodb';

const dynamodb = new DynamoDBClient({});

const ACCOUNT_VIEW_TABLE = process.env.ACCOUNT_VIEW_TABLE!;
const TRANSACTION_VIEW_TABLE = process.env.TRANSACTION_VIEW_TABLE!;

export const handler = async (event: any) => {
  const { queryStringParameters } = event;
  const queryType = queryStringParameters?.type;

  console.log('Processing query:', queryType);

  try {
    let result;

    switch (queryType) {
      case 'GetAccount':
        result = await getAccount(queryStringParameters.accountId);
        break;
      case 'GetTransactions':
        result = await getTransactions(queryStringParameters.accountId);
        break;
      case 'GetPendingTransactions':
        result = await getPendingTransactions();
        break;
      default:
        throw new Error(`Unknown query type: ${queryType}`);
    }

    return {
      statusCode: 200,
      body: JSON.stringify(result)
    };
  } catch (error) {
    console.error('Query failed:', error);
    return {
      statusCode: 400,
      body: JSON.stringify({ error: (error as Error).message })
    };
  }
};

async function getAccount(accountId: string): Promise<any> {
  const response = await dynamodb.send(
    new GetItemCommand({
      TableName: ACCOUNT_VIEW_TABLE,
      Key: {
        accountId: { S: accountId }
      }
    })
  );

  if (!response.Item) {
    throw new Error('Account not found');
  }

  return {
    accountId: response.Item.accountId.S,
    balance: parseFloat(response.Item.balance.N!),
    currency: response.Item.currency.S,
    status: response.Item.status.S,
    lastUpdated: parseInt(response.Item.lastUpdated.N!)
  };
}

async function getTransactions(accountId: string): Promise<any[]> {
  const response = await dynamodb.send(
    new QueryCommand({
      TableName: TRANSACTION_VIEW_TABLE,
      KeyConditionExpression: 'accountId = :id',
      ExpressionAttributeValues: {
        ':id': { S: accountId }
      },
      ScanIndexForward: false,
      Limit: 50
    })
  );

  return (response.Items || []).map(item => ({
    transactionId: item.transactionId.S,
    accountId: item.accountId.S,
    type: item.type.S,
    amount: parseFloat(item.amount.N!),
    timestamp: parseInt(item.timestamp.N!),
    status: item.status.S
  }));
}

async function getPendingTransactions(): Promise<any[]> {
  const response = await dynamodb.send(
    new QueryCommand({
      TableName: TRANSACTION_VIEW_TABLE,
      IndexName: 'StatusIndex',
      KeyConditionExpression: '#status = :pending',
      ExpressionAttributeNames: {
        '#status': 'status'
      },
      ExpressionAttributeValues: {
        ':pending': { S: 'PENDING' }
      },
      Limit: 100
    })
  );

  return (response.Items || []).map(item => ({
    transactionId: item.transactionId.S,
    accountId: item.accountId.S,
    type: item.type.S,
    amount: parseFloat(item.amount.N!),
    timestamp: parseInt(item.timestamp.N!)
  }));
}
```

### 🔄 Saga Orchestrator Implementation

```typescript
// lambda/saga-orchestrator/index.ts
import { DynamoDBClient, GetItemCommand, PutItemCommand, UpdateItemCommand } from '@aws-sdk/client-dynamodb';
import { EventBridgeClient, PutEventsCommand } from '@aws-sdk/client-eventbridge';
import { v4 as uuidv4 } from 'uuid';

const dynamodb = new DynamoDBClient({});
const eventBridge = new EventBridgeClient({});

const SAGA_STORE_TABLE = process.env.SAGA_STORE_TABLE!;
const EVENT_BUS_NAME = process.env.EVENT_BUS_NAME!;

interface SagaState {
  sagaId: string;
  type: string;
  status: 'STARTED' | 'COMPLETED' | 'FAILED' | 'COMPENSATING';
  currentStep: number;
  data: any;
  createdAt: number;
  updatedAt: number;
}

export const handler = async (event: any) => {
  console.log('Saga event received:', JSON.stringify(event));

  for (const record of event.Records) {
    const detail = JSON.parse(record.body);
    await processSagaEvent(detail);
  }
};

async function processSagaEvent(event: any): Promise<void> {
  const eventType = event['detail-type'];
  const eventData = event.detail;

  switch (eventType) {
    case 'TransferInitiated':
      await startTransferSaga(eventData);
      break;
    case 'MoneyWithdrawn':
      await handleWithdrawalComplete(eventData);
      break;
    case 'MoneyDeposited':
      await handleDepositComplete(eventData);
      break;
    case 'WithdrawalFailed':
      await handleWithdrawalFailed(eventData);
      break;
    case 'DepositFailed':
      await handleDepositFailed(eventData);
      break;
  }
}

async function startTransferSaga(data: any): Promise<void> {
  const sagaId = uuidv4();

  const saga: SagaState = {
    sagaId,
    type: 'MONEY_TRANSFER',
    status: 'STARTED',
    currentStep: 1,
    data: {
      fromAccount: data.fromAccount,
      toAccount: data.toAccount,
      amount: data.amount,
      currency: data.currency
    },
    createdAt: Date.now(),
    updatedAt: Date.now()
  };

  await saveSaga(saga);

  // Step 1: Withdraw from source account
  await publishCommand({
    type: 'WithdrawMoney',
    aggregateId: data.fromAccount,
    data: {
      amount: data.amount,
      reference: sagaId
    }
  });
}

async function handleWithdrawalComplete(data: any): Promise<void> {
  const saga = await loadSaga(data.reference);

  if (!saga || saga.status !== 'STARTED') return;

  saga.currentStep = 2;
  saga.updatedAt = Date.now();
  await saveSaga(saga);

  // Step 2: Deposit to destination account
  await publishCommand({
    type: 'DepositMoney',
    aggregateId: saga.data.toAccount,
    data: {
      amount: saga.data.amount,
      reference: saga.sagaId
    }
  });
}

async function handleDepositComplete(data: any): Promise<void> {
  const saga = await loadSaga(data.reference);

  if (!saga) return;

  saga.status = 'COMPLETED';
  saga.currentStep = 3;
  saga.updatedAt = Date.now();
  await saveSaga(saga);

  // Publish success event
  await publishEvent({
    Source: 'banking.transfers',
    DetailType: 'TransferCompleted',
    Detail: JSON.stringify({
      sagaId: saga.sagaId,
      ...saga.data
    })
  });
}

async function handleWithdrawalFailed(data: any): Promise<void> {
  const saga = await loadSaga(data.reference);

  if (!saga) return;

  saga.status = 'FAILED';
  saga.updatedAt = Date.now();
  await saveSaga(saga);

  await publishEvent({
    Source: 'banking.transfers',
    DetailType: 'TransferFailed',
    Detail: JSON.stringify({
      sagaId: saga.sagaId,
      reason: 'Withdrawal failed',
      ...saga.data
    })
  });
}

async function handleDepositFailed(data: any): Promise<void> {
  const saga = await loadSaga(data.reference);

  if (!saga) return;

  saga.status = 'COMPENSATING';
  saga.updatedAt = Date.now();
  await saveSaga(saga);

  // Compensate: Return money to source account
  await publishCommand({
    type: 'DepositMoney',
    aggregateId: saga.data.fromAccount,
    data: {
      amount: saga.data.amount,
      reference: `COMPENSATION-${saga.sagaId}`
    }
  });
}

async function loadSaga(sagaId: string): Promise<SagaState | null> {
  const response = await dynamodb.send(
    new GetItemCommand({
      TableName: SAGA_STORE_TABLE,
      Key: {
        sagaId: { S: sagaId }
      }
    })
  );

  if (!response.Item) return null;

  return {
    sagaId: response.Item.sagaId.S!,
    type: response.Item.type.S!,
    status: response.Item.status.S as any,
    currentStep: parseInt(response.Item.currentStep.N!),
    data: JSON.parse(response.Item.data.S!),
    createdAt: parseInt(response.Item.createdAt.N!),
    updatedAt: parseInt(response.Item.updatedAt.N!)
  };
}

async function saveSaga(saga: SagaState): Promise<void> {
  await dynamodb.send(
    new PutItemCommand({
      TableName: SAGA_STORE_TABLE,
      Item: {
        sagaId: { S: saga.sagaId },
        type: { S: saga.type },
        status: { S: saga.status },
        currentStep: { N: String(saga.currentStep) },
        data: { S: JSON.stringify(saga.data) },
        createdAt: { N: String(saga.createdAt) },
        updatedAt: { N: String(saga.updatedAt) }
      }
    })
  );
}

async function publishCommand(command: any): Promise<void> {
  await eventBridge.send(
    new PutEventsCommand({
      Entries: [{
        Source: 'banking.commands',
        DetailType: command.type,
        Detail: JSON.stringify(command),
        EventBusName: EVENT_BUS_NAME
      }]
    })
  );
}

async function publishEvent(event: any): Promise<void> {
  await eventBridge.send(
    new PutEventsCommand({
      Entries: [{ ...event, EventBusName: EVENT_BUS_NAME }]
    })
  );
}
```

### 🎓 Interview Discussion Points - Q59

**Q1: What is CQRS and why use it?**

**A**:
**CQRS = Command Query Responsibility Segregation**
- **Separate models**: Commands (write) vs Queries (read)
- **Benefits**:
  - Independent scaling (read-heavy vs write-heavy)
  - Optimized data models for each use case
  - Simpler business logic (commands)
  - Better performance (materialized views)
  - Easier to cache queries

**When to use**: Complex domains, high read/write ratio, need for different consistency models

**Q2: What is Event Sourcing?**

**A**:
**Store events, not current state**
- **Traditional**: UPDATE balance = 100 (lose history)
- **Event Sourcing**: 
  - AccountCreated (balance: 0)
  - MoneyDeposited (amount: 50)
  - MoneyWithdrawn (amount: 20)
  - **Current state**: Replay events → balance = 30

**Benefits**:
- Complete audit trail
- Time travel (rebuild state at any point)
- Event replay for projections
- Debug production issues

**Trade-offs**: More complex, eventual consistency

**Q3: How does Saga pattern work?**

**A**:
**Saga = Distributed transaction pattern**

**Choreography** (Event-driven):
- Service A → Event → Service B → Event → Service C
- Decentralized, loose coupling

**Orchestration** (Coordinator):
- Coordinator calls Service A, B, C sequentially
- Centralized control, easier to monitor

**Compensation**: If step fails, undo previous steps
- Transfer saga: Withdraw → Deposit
- If deposit fails → Compensate (re-deposit to source)

**Q4: How to handle eventual consistency?**

**A**:
**Strategies**:
1. **User feedback**: "Your request is being processed"
2. **Optimistic UI**: Show change immediately, rollback if fails
3. **Versioning**: Use version numbers to detect conflicts
4. **Idempotency**: Use request IDs to prevent duplicates
5. **Timeouts**: Set max wait time, then compensate

**For banking**:
- Show "pending" status during transfer
- Send notification when complete
- Allow user to track progress

**Q5: How to test event-sourced systems?**

**A**:
**Testing approaches**:
1. **Given-When-Then**: 
   - Given: AccountCreated, MoneyDeposited
   - When: WithdrawMoney(50)
   - Then: MoneyWithdrawn event published
2. **Replay events**: Load production events in test env
3. **Projections**: Test materialized views separately
4. **Sagas**: Test happy path + compensation paths
5. **Chaos**: Kill services mid-transaction

**Tools**: Jest for unit tests, LocalStack for integration

---

## Question 60: AWS Well-Architected Framework Review

### 📋 Question Statement

Comprehensive review and implementation of AWS Well-Architected Framework pillars for Emirates NBD banking platform including operational excellence, security, reliability, performance efficiency, cost optimization, and sustainability.

---

### 🏛️ Well-Architected Framework Implementation

#### 1. Operational Excellence

```typescript
// services/operations/observability-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as cloudwatch from 'aws-cdk-lib/aws-cloudwatch';
import * as logs from 'aws-cdk-lib/aws-logs';
import * as xray from 'aws-cdk-lib/aws-xray';

export class OperationalExcellenceStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string) {
    super(scope, id);

    // Centralized logging
    const logGroup = new logs.LogGroup(this, 'ApplicationLogs', {
      logGroupName: '/aws/banking/application',
      retention: logs.RetentionDays.ONE_MONTH
    });

    // Metrics Dashboard
    const dashboard = new cloudwatch.Dashboard(this, 'OperationsDashboard', {
      dashboardName: 'Banking-Operations'
    });

    // Key Performance Indicators
    dashboard.addWidgets(
      new cloudwatch.GraphWidget({
        title: 'Transaction Success Rate',
        left: [
          new cloudwatch.Metric({
            namespace: 'Banking',
            metricName: 'TransactionSuccess',
            statistic: 'Average',
            label: 'Success Rate',
            color: '#2ca02c'
          })
        ]
      }),
      new cloudwatch.GraphWidget({
        title: 'API Latency (P50, P99)',
        left: [
          new cloudwatch.Metric({
            namespace: 'AWS/ApiGateway',
            metricName: 'Latency',
            statistic: 'p50'
          }),
          new cloudwatch.Metric({
            namespace: 'AWS/ApiGateway',
            metricName: 'Latency',
            statistic: 'p99'
          })
        ]
      })
    );

    // Best Practices:
    // ✅ Runbooks for common operations
    // ✅ Automated deployment pipelines
    // ✅ Infrastructure as Code (CDK)
    // ✅ Monitoring and alerting
    // ✅ Game days for incident response
  }
}
```

#### 2. Security

```typescript
// infrastructure/security/security-baseline.ts
import * as cdk from 'aws-cdk-lib';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as kms from 'aws-cdk-lib/aws-kms';
import * as secretsmanager from 'aws-cdk-lib/aws-secretsmanager';

export class SecurityBaselineStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string) {
    super(scope, id);

    // KMS key for encryption
    const encryptionKey = new kms.Key(this, 'DataEncryptionKey', {
      enableKeyRotation: true,
      description: 'Key for encrypting banking data'
    });

    // Secrets Manager for credentials
    const dbSecret = new secretsmanager.Secret(this, 'DBCredentials', {
      generateSecretString: {
        secretStringTemplate: JSON.stringify({ username: 'admin' }),
        generateStringKey: 'password',
        excludePunctuation: true
      }
    });

    // IAM principle of least privilege
    const lambdaRole = new iam.Role(this, 'LambdaExecutionRole', {
      assumedBy: new iam.ServicePrincipal('lambda.amazonaws.com'),
      managedPolicies: [
        iam.ManagedPolicy.fromAwsManagedPolicyName('service-role/AWSLambdaBasicExecutionRole')
      ]
    });

    // Grant specific permissions only
    encryptionKey.grantEncryptDecrypt(lambdaRole);
    dbSecret.grantRead(lambdaRole);

    // Security Best Practices:
    // ✅ Encryption at rest (KMS)
    // ✅ Encryption in transit (TLS 1.2+)
    // ✅ IAM least privilege
    // ✅ Secrets rotation
    // ✅ VPC isolation
    // ✅ Security groups (defense in depth)
    // ✅ AWS WAF for APIs
    // ✅ GuardDuty for threat detection
  }
}
```

#### 3. Reliability

```typescript
// infrastructure/reliability/multi-az-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as rds from 'aws-cdk-lib/aws-rds';
import * as elasticache from 'aws-cdk-lib/aws-elasticache';
import * as ecs from 'aws-cdk-lib/aws-ecs';

export class ReliabilityStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props: any) {
    super(scope, id);

    // Multi-AZ RDS
    const database = new rds.DatabaseInstance(this, 'Database', {
      engine: rds.DatabaseInstanceEngine.postgres({ version: rds.PostgresEngineVersion.VER_15 }),
      instanceType: props.instanceType,
      vpc: props.vpc,
      multiAz: true, // High availability
      backupRetention: cdk.Duration.days(7),
      deleteAutomatedBackups: false,
      autoMinorVersionUpgrade: true
    });

    // Multi-AZ ElastiCache
    const cacheSubnetGroup = new elasticache.CfnSubnetGroup(this, 'CacheSubnetGroup', {
      description: 'Cache subnet group',
      subnetIds: props.vpc.privateSubnets.map((subnet: any) => subnet.subnetId)
    });

    const replicationGroup = new elasticache.CfnReplicationGroup(this, 'RedisCluster', {
      replicationGroupDescription: 'Redis cluster with automatic failover',
      engine: 'redis',
      cacheNodeType: 'cache.r6g.large',
      numNodeGroups: 3, // 3 shards
      replicasPerNodeGroup: 2, // 2 replicas per shard
      automaticFailoverEnabled: true,
      multiAzEnabled: true,
      cacheSubnetGroupName: cacheSubnetGroup.ref
    });

    // ECS auto-scaling
    const ecsService = new ecs.FargateService(this, 'Service', {
      cluster: props.cluster,
      taskDefinition: props.taskDefinition,
      desiredCount: 3,
      minHealthyPercent: 100,
      maxHealthyPercent: 200
    });

    const scaling = ecsService.autoScaleTaskCount({
      minCapacity: 3,
      maxCapacity: 20
    });

    scaling.scaleOnCpuUtilization('CpuScaling', {
      targetUtilizationPercent: 70
    });

    // Reliability Best Practices:
    // ✅ Multi-AZ deployment
    // ✅ Auto-scaling
    // ✅ Health checks
    // ✅ Circuit breakers
    // ✅ Retry with exponential backoff
    // ✅ Automated backups
    // ✅ Disaster recovery plan (RTO/RPO)
  }
}
```

#### 4. Performance Efficiency

```typescript
// services/performance/optimization-config.ts
export class PerformanceOptimization {
  static getDatabaseOptimizations() {
    return {
      // Connection pooling
      connectionPool: {
        min: 10,
        max: 100,
        idleTimeoutMillis: 30000,
        connectionTimeoutMillis: 2000
      },
      
      // Query optimization
      indexes: [
        'CREATE INDEX idx_account_customer ON accounts(customer_id)',
        'CREATE INDEX idx_transaction_account_date ON transactions(account_id, created_at)',
        'CREATE INDEX idx_payment_status ON payments(status) WHERE status = \'PENDING\''
      ],
      
      // Read replicas
      readReplicas: 2,
      
      // Caching strategy
      cache: {
        accountData: { ttl: 300 },      // 5 minutes
        customerProfile: { ttl: 3600 },  // 1 hour
        transactionHistory: { ttl: 60 }  // 1 minute
      }
    };
  }

  static getLambdaOptimizations() {
    return {
      // Memory allocation
      memorySize: 1024, // MB (more memory = more CPU)
      
      // Provisioned concurrency
      provisionedConcurrency: 10,
      
      // Environment variables
      environment: {
        NODE_OPTIONS: '--enable-source-maps --max-old-space-size=900'
      },
      
      // Architecture
      architecture: 'arm64', // Graviton2 (better price/performance)
      
      // Cold start mitigation
      keepWarm: true,
      warmingSchedule: 'rate(5 minutes)'
    };
  }

  // Performance Best Practices:
  // ✅ CloudFront CDN for static assets
  // ✅ ElastiCache Redis for caching
  // ✅ DynamoDB DAX for microsecond latency
  // ✅ Lambda@Edge for edge computing
  // ✅ Aurora Serverless for variable workloads
  // ✅ S3 Transfer Acceleration
}
```

#### 5. Cost Optimization

```typescript
// services/cost/optimization-strategies.ts
export class CostOptimizationStrategies {
  static getRecommendations() {
    return {
      compute: {
        // EC2/ECS
        reservedInstances: {
          recommendation: 'Purchase 1-year RIs for baseline capacity',
          savings: '~40% vs on-demand'
        },
        savingsPlans: {
          recommendation: 'Compute Savings Plans for flexible workloads',
          savings: '~66% vs on-demand'
        },
        spotInstances: {
          recommendation: 'Use Spot for batch jobs and dev/test',
          savings: '~70% vs on-demand'
        },
        rightsizing: {
          recommendation: 'Downsize over-provisioned instances',
          currentWaste: '$2,000/month'
        }
      },
      
      storage: {
        s3IntelligentTiering: {
          recommendation: 'Enable for infrequently accessed data',
          savings: '~70% on access patterns'
        },
        ebsSnapshots: {
          recommendation: 'Delete old snapshots (>90 days)',
          savings: '$500/month'
        }
      },
      
      database: {
        auroraServerless: {
          recommendation: 'Use for dev/test environments',
          savings: '~50% vs provisioned'
        },
        readReplicaScheduling: {
          recommendation: 'Stop read replicas during off-hours',
          savings: '$1,200/month'
        }
      }
    };
  }

  // Cost Optimization Best Practices:
  // ✅ Right-size resources
  // ✅ Use Savings Plans / Reserved Instances
  // ✅ Leverage Spot Instances
  // ✅ Implement auto-scaling
  // ✅ S3 Intelligent-Tiering
  // ✅ Delete unused resources
  // ✅ Cost allocation tags
  // ✅ Budgets and alerts
}
```

#### 6. Sustainability

```typescript
// services/sustainability/green-architecture.ts
export class SustainabilityPractices {
  static getGreenArchitecturePatterns() {
    return {
      // Optimize resource utilization
      resourceOptimization: {
        cpuTarget: 70, // Target 70% utilization
        memoryTarget: 80,
        rightSizing: 'Eliminate idle resources'
      },
      
      // Use managed services (better efficiency)
      managedServices: [
        'Lambda (serverless)',
        'Fargate (no idle EC2)',
        'Aurora Serverless',
        'DynamoDB (on-demand)'
      ],
      
      // Regional optimization
      regions: {
        primary: 'us-east-1', // Lower carbon intensity
        secondary: 'eu-central-1',
        avoidRegions: ['Coal-heavy regions']
      },
      
      // Graviton2 processors
      graviton: {
        benefit: '60% better energy efficiency vs x86',
        costSaving: '20% cheaper',
        use: 'Lambda, ECS, RDS'
      },
      
      // Lifecycle policies
      dataLifecycle: {
        hotData: 'DynamoDB (7 days)',
        warmData: 'S3 Standard (30 days)',
        coldData: 'S3 Glacier (1 year)',
        archiveData: 'S3 Glacier Deep Archive (7 years)'
      }
    };
  }

  // Sustainability Best Practices:
  // ✅ Serverless-first approach
  // ✅ Graviton2 processors
  // ✅ Auto-scaling (avoid idle resources)
  // ✅ Data lifecycle management
  // ✅ Select regions with renewable energy
  // ✅ Optimize code efficiency
  // ✅ Monitor carbon footprint
}
```

### 📊 Well-Architected Review Checklist

```typescript
export const WellArchitectedChecklist = {
  operationalExcellence: [
    { question: 'Do you have runbooks for common operations?', status: '✅' },
    { question: 'Is infrastructure defined as code?', status: '✅' },
    { question: 'Do you use CI/CD pipelines?', status: '✅' },
    { question: 'Are deployments automated?', status: '✅' },
    { question: 'Do you conduct game days?', status: '✅' }
  ],
  
  security: [
    { question: 'Is data encrypted at rest?', status: '✅' },
    { question: 'Is data encrypted in transit (TLS)?', status: '✅' },
    { question: 'Do you use IAM least privilege?', status: '✅' },
    { question: 'Are secrets rotated regularly?', status: '✅' },
    { question: 'Is MFA enabled for privileged accounts?', status: '✅' }
  ],
  
  reliability: [
    { question: 'Is the application multi-AZ?', status: '✅' },
    { question: 'Do you have automated backups?', status: '✅' },
    { question: 'Are there health checks and auto-scaling?', status: '✅' },
    { question: 'Have you tested disaster recovery?', status: '✅' },
    { question: 'Do you use circuit breakers?', status: '✅' }
  ],
  
  performance: [
    { question: 'Do you use caching (Redis/CloudFront)?', status: '✅' },
    { question: 'Are databases optimized (indexes, read replicas)?', status: '✅' },
    { question: 'Do you use CDN for static assets?', status: '✅' },
    { question: 'Have you right-sized resources?', status: '✅' },
    { question: 'Do you use ARM/Graviton processors?', status: '✅' }
  ],
  
  costOptimization: [
    { question: 'Do you use Savings Plans/Reserved Instances?', status: '✅' },
    { question: 'Are unused resources deleted?', status: '✅' },
    { question: 'Do you use S3 Intelligent-Tiering?', status: '✅' },
    { question: 'Are cost allocation tags implemented?', status: '✅' },
    { question: 'Do you have budget alerts?', status: '✅' }
  ],
  
  sustainability: [
    { question: 'Do you use serverless where appropriate?', status: '✅' },
    { question: 'Are you using Graviton processors?', status: '✅' },
    { question: 'Do you have data lifecycle policies?', status: '✅' },
    { question: 'Is resource utilization optimized?', status: '✅' },
    { question: 'Are you in regions with renewable energy?', status: '✅' }
  ]
};
```

### 🎓 Interview Discussion Points - Q60

**Q1: What are the 6 pillars of Well-Architected Framework?**

**A**:
1. **Operational Excellence**: Run and monitor systems
2. **Security**: Protect information and systems
3. **Reliability**: Recover from failures
4. **Performance Efficiency**: Use resources efficiently
5. **Cost Optimization**: Avoid unnecessary costs
6. **Sustainability**: Minimize environmental impact

**Q2: What is the difference between RTO and RPO?**

**A**:
- **RTO (Recovery Time Objective)**: How long to restore service (e.g., 1 hour)
- **RPO (Recovery Point Objective)**: How much data loss is acceptable (e.g., 5 minutes)
- **Example**: RTO=1hr means system back in 1hr, RPO=5min means max 5min data loss

**Q3: What is least privilege principle?**

**A**:
- **Definition**: Grant minimum permissions needed
- **Example**: Lambda only needs `dynamodb:PutItem`, not `dynamodb:*`
- **Benefits**: Reduce blast radius of compromised credentials
- **IAM**: Use roles, not root account

**Q4: How to optimize Lambda cold starts?**

**A**:
- **Provisioned concurrency**: Keep functions warm
- **Increase memory**: More memory = faster cold start
- **Reduce package size**: Smaller deployment = faster init
- **ARM/Graviton2**: 20% faster cold starts
- **Connection reuse**: Initialize outside handler

**Q5: What is the cost of not optimizing?**

**A**:
- **Idle resources**: $10k/month wasted on unused EC2
- **No auto-scaling**: Over-provisioned 24/7
- **No caching**: Expensive database queries
- **No Savings Plans**: Paying 3x on-demand price
- **Typical**: 30-50% cost reduction possible

---

**🎉 Congratulations! You've completed all 60 comprehensive microservices interview questions covering serverless (Lambda), server-based (ECS/EKS), and hybrid architectures for Emirates NBD Digital Banking Platform. 🎉**

---

**End of File 30 - Complete 60-Question Microservices Architecture Course**