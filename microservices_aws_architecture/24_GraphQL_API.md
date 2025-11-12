# GraphQL API

## Question 47: GraphQL Implementation with AppSync

### 📋 Question Statement

Implement a GraphQL API for Emirates NBD using AWS AppSync, enabling flexible data queries and real-time subscriptions for banking operations.

---

### 🔷 GraphQL AppSync Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                   AWS APPSYNC GRAPHQL API                                   │
└────────────────────────────────────────────────────────────────────────────┘

    ┌──────────────┐
    │ Mobile/Web   │
    │   Client     │
    └──────┬───────┘
           │ GraphQL
           v
    ┌─────────────────┐
    │   AWS AppSync   │
    │  GraphQL API    │
    └────────┬────────┘
             │
    ┌────────┼────────┬─────────────┬─────────────┐
    │        │        │             │             │
    v        v        v             v             v
┌─────────┐┌─────────┐┌──────────┐┌──────────┐┌──────────┐
│DynamoDB ││ Lambda  ││   RDS    ││ElastiCache││   HTTP   │
│Resolver ││Resolver ││ Resolver ││ Resolver  ││ Resolver │
└─────────┘└─────────┘└──────────┘└──────────┘└──────────┘
```

### 📦 AppSync CDK Infrastructure

```typescript
// infrastructure/cdk/appsync-graphql-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as appsync from 'aws-cdk-lib/aws-appsync';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as rds from 'aws-cdk-lib/aws-rds';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as logs from 'aws-cdk-lib/aws-logs';
import * as cognito from 'aws-cdk-lib/aws-cognito';
import * as iam from 'aws-cdk-lib/aws-iam';

export class AppSyncGraphQLStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Cognito User Pool for authentication
    const userPool = new cognito.UserPool(this, 'BankingUserPool', {
      userPoolName: 'emirates-nbd-users',
      selfSignUpEnabled: false,
      signInAliases: { email: true, username: true },
      passwordPolicy: {
        minLength: 12,
        requireLowercase: true,
        requireUppercase: true,
        requireDigits: true,
        requireSymbols: true
      },
      mfa: cognito.Mfa.REQUIRED,
      mfaSecondFactor: {
        sms: true,
        otp: true
      }
    });

    const userPoolClient = userPool.addClient('AppClient', {
      authFlows: {
        userPassword: true,
        userSrp: true
      }
    });

    // DynamoDB Tables
    const accountsTable = new dynamodb.Table(this, 'Accounts', {
      tableName: 'banking-accounts',
      partitionKey: { name: 'accountId', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      stream: dynamodb.StreamViewType.NEW_AND_OLD_IMAGES
    });

    accountsTable.addGlobalSecondaryIndex({
      indexName: 'customerIdIndex',
      partitionKey: { name: 'customerId', type: dynamodb.AttributeType.STRING }
    });

    const transactionsTable = new dynamodb.Table(this, 'Transactions', {
      tableName: 'banking-transactions',
      partitionKey: { name: 'transactionId', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'timestamp', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST
    });

    transactionsTable.addGlobalSecondaryIndex({
      indexName: 'accountIdIndex',
      partitionKey: { name: 'accountId', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'timestamp', type: dynamodb.AttributeType.STRING }
    });

    // AppSync GraphQL API
    const api = new appsync.GraphqlApi(this, 'BankingGraphQLAPI', {
      name: 'emirates-nbd-graphql-api',
      schema: appsync.SchemaFile.fromAsset('graphql/schema.graphql'),
      authorizationConfig: {
        defaultAuthorization: {
          authorizationType: appsync.AuthorizationType.USER_POOL,
          userPoolConfig: {
            userPool
          }
        },
        additionalAuthorizationModes: [
          {
            authorizationType: appsync.AuthorizationType.API_KEY,
            apiKeyConfig: {
              expires: cdk.Expiration.after(cdk.Duration.days(365))
            }
          }
        ]
      },
      logConfig: {
        fieldLogLevel: appsync.FieldLogLevel.ALL,
        excludeVerboseContent: false,
        retention: logs.RetentionDays.ONE_WEEK
      },
      xrayEnabled: true
    });

    // DynamoDB Data Sources
    const accountsDataSource = api.addDynamoDbDataSource(
      'AccountsDataSource',
      accountsTable
    );

    const transactionsDataSource = api.addDynamoDbDataSource(
      'TransactionsDataSource',
      transactionsTable
    );

    // Lambda Data Source for complex logic
    const businessLogicFunction = new lambda.Function(this, 'BusinessLogic', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/graphql-resolvers'),
      timeout: cdk.Duration.seconds(30),
      environment: {
        ACCOUNTS_TABLE: accountsTable.tableName,
        TRANSACTIONS_TABLE: transactionsTable.tableName
      }
    });

    accountsTable.grantReadWriteData(businessLogicFunction);
    transactionsTable.grantReadWriteData(businessLogicFunction);

    const lambdaDataSource = api.addLambdaDataSource(
      'BusinessLogicDataSource',
      businessLogicFunction
    );

    // Query Resolvers
    accountsDataSource.createResolver('GetAccountResolver', {
      typeName: 'Query',
      fieldName: 'getAccount',
      requestMappingTemplate: appsync.MappingTemplate.dynamoDbGetItem('accountId', 'accountId'),
      responseMappingTemplate: appsync.MappingTemplate.dynamoDbResultItem()
    });

    accountsDataSource.createResolver('ListAccountsResolver', {
      typeName: 'Query',
      fieldName: 'listAccountsByCustomer',
      requestMappingTemplate: appsync.MappingTemplate.dynamoDbQuery(
        appsync.KeyCondition.eq('customerId', 'customerId'),
        'customerIdIndex'
      ),
      responseMappingTemplate: appsync.MappingTemplate.dynamoDbResultList()
    });

    transactionsDataSource.createResolver('ListTransactionsResolver', {
      typeName: 'Query',
      fieldName: 'listTransactionsByAccount',
      requestMappingTemplate: appsync.MappingTemplate.dynamoDbQuery(
        appsync.KeyCondition.eq('accountId', 'accountId'),
        'accountIdIndex'
      ),
      responseMappingTemplate: appsync.MappingTemplate.dynamoDbResultList()
    });

    // Mutation Resolvers (using Lambda for business logic)
    lambdaDataSource.createResolver('CreateTransferResolver', {
      typeName: 'Mutation',
      fieldName: 'createTransfer'
    });

    lambdaDataSource.createResolver('CreatePaymentResolver', {
      typeName: 'Mutation',
      fieldName: 'createPayment'
    });

    accountsDataSource.createResolver('UpdateAccountResolver', {
      typeName: 'Mutation',
      fieldName: 'updateAccount',
      requestMappingTemplate: appsync.MappingTemplate.dynamoDbPutItem(
        appsync.PrimaryKey.partition('accountId').is('accountId'),
        appsync.Values.projecting('input')
      ),
      responseMappingTemplate: appsync.MappingTemplate.dynamoDbResultItem()
    });

    // Subscription Resolvers (automatic with @aws_subscribe directive)

    // Outputs
    new cdk.CfnOutput(this, 'GraphQLEndpoint', {
      value: api.graphqlUrl,
      description: 'GraphQL API Endpoint'
    });

    new cdk.CfnOutput(this, 'ApiKey', {
      value: api.apiKey || 'N/A',
      description: 'API Key for testing'
    });

    new cdk.CfnOutput(this, 'UserPoolId', {
      value: userPool.userPoolId,
      description: 'Cognito User Pool ID'
    });

    new cdk.CfnOutput(this, 'UserPoolClientId', {
      value: userPoolClient.userPoolClientId,
      description: 'User Pool Client ID'
    });
  }
}
```

### 📝 GraphQL Schema

```graphql
# graphql/schema.graphql

type Query {
  getAccount(accountId: ID!): Account
  listAccountsByCustomer(customerId: ID!, limit: Int, nextToken: String): AccountConnection
  getTransaction(transactionId: ID!): Transaction
  listTransactionsByAccount(accountId: ID!, limit: Int, nextToken: String): TransactionConnection
  getCustomer(customerId: ID!): Customer
  searchTransactions(filter: TransactionFilter!): TransactionConnection
}

type Mutation {
  createAccount(input: CreateAccountInput!): Account
  updateAccount(accountId: ID!, input: UpdateAccountInput!): Account
  closeAccount(accountId: ID!): Account
  
  createTransfer(input: CreateTransferInput!): Transfer
  createPayment(input: CreatePaymentInput!): Payment
  
  createTransaction(input: CreateTransactionInput!): Transaction
    @aws_cognito_user_pools
}

type Subscription {
  onAccountUpdated(accountId: ID!): Account
    @aws_subscribe(mutations: ["updateAccount"])
  
  onTransactionCreated(accountId: ID!): Transaction
    @aws_subscribe(mutations: ["createTransaction"])
  
  onPaymentStatusChanged(paymentId: ID!): Payment
    @aws_subscribe(mutations: ["updatePaymentStatus"])
}

type Account {
  accountId: ID!
  customerId: ID!
  accountNumber: String!
  accountType: AccountType!
  balance: Float!
  currency: String!
  status: AccountStatus!
  createdAt: AWSDateTime!
  updatedAt: AWSDateTime!
  
  # Nested fields (resolved separately)
  transactions(limit: Int, nextToken: String): TransactionConnection
  customer: Customer
}

type Customer {
  customerId: ID!
  firstName: String!
  lastName: String!
  email: AWSEmail!
  phoneNumber: AWSPhone
  dateOfBirth: AWSDate
  accounts: [Account!]!
}

type Transaction {
  transactionId: ID!
  accountId: ID!
  amount: Float!
  type: TransactionType!
  status: TransactionStatus!
  description: String
  merchantName: String
  merchantCategory: String
  location: Location
  timestamp: AWSDateTime!
  
  account: Account
}

type Transfer {
  transferId: ID!
  fromAccountId: ID!
  toAccountId: ID!
  amount: Float!
  currency: String!
  status: TransferStatus!
  reference: String
  createdAt: AWSDateTime!
}

type Payment {
  paymentId: ID!
  accountId: ID!
  beneficiaryId: ID!
  amount: Float!
  currency: String!
  status: PaymentStatus!
  scheduledDate: AWSDate
  completedAt: AWSDateTime
}

type Location {
  country: String!
  city: String
  latitude: Float
  longitude: Float
}

type AccountConnection {
  items: [Account!]!
  nextToken: String
}

type TransactionConnection {
  items: [Transaction!]!
  nextToken: String
}

enum AccountType {
  SAVINGS
  CURRENT
  FIXED_DEPOSIT
  CREDIT_CARD
}

enum AccountStatus {
  ACTIVE
  INACTIVE
  BLOCKED
  CLOSED
}

enum TransactionType {
  DEBIT
  CREDIT
  TRANSFER
  PAYMENT
  WITHDRAWAL
  DEPOSIT
}

enum TransactionStatus {
  PENDING
  COMPLETED
  FAILED
  CANCELLED
}

enum TransferStatus {
  PENDING
  PROCESSING
  COMPLETED
  FAILED
}

enum PaymentStatus {
  SCHEDULED
  PROCESSING
  COMPLETED
  FAILED
  CANCELLED
}

input CreateAccountInput {
  customerId: ID!
  accountType: AccountType!
  currency: String!
  initialDeposit: Float
}

input UpdateAccountInput {
  status: AccountStatus
  balance: Float
}

input CreateTransferInput {
  fromAccountId: ID!
  toAccountId: ID!
  amount: Float!
  currency: String!
  reference: String
}

input CreatePaymentInput {
  accountId: ID!
  beneficiaryId: ID!
  amount: Float!
  currency: String!
  scheduledDate: AWSDate
}

input CreateTransactionInput {
  accountId: ID!
  amount: Float!
  type: TransactionType!
  description: String
  merchantName: String
}

input TransactionFilter {
  accountId: ID!
  minAmount: Float
  maxAmount: Float
  types: [TransactionType!]
  startDate: AWSDateTime
  endDate: AWSDateTime
}
```

### 🔧 Lambda Resolver for Complex Logic

```typescript
// lambda/graphql-resolvers/index.ts
import { AppSyncResolverHandler } from 'aws-lambda';
import { DynamoDBClient, TransactWriteItemsCommand, GetItemCommand, PutItemCommand } from '@aws-sdk/client-dynamodb';
import { v4 as uuidv4 } from 'uuid';

const dynamodb = new DynamoDBClient({});
const ACCOUNTS_TABLE = process.env.ACCOUNTS_TABLE!;
const TRANSACTIONS_TABLE = process.env.TRANSACTIONS_TABLE!;

export const handler: AppSyncResolverHandler<any, any> = async (event) => {
  console.log('GraphQL Event:', JSON.stringify(event, null, 2));

  const { fieldName } = event.info;

  switch (fieldName) {
    case 'createTransfer':
      return createTransfer(event.arguments.input);
    
    case 'createPayment':
      return createPayment(event.arguments.input);
    
    default:
      throw new Error(`Unknown field: ${fieldName}`);
  }
};

interface CreateTransferInput {
  fromAccountId: string;
  toAccountId: string;
  amount: number;
  currency: string;
  reference?: string;
}

async function createTransfer(input: CreateTransferInput) {
  const transferId = uuidv4();
  const timestamp = new Date().toISOString();

  // Validate accounts exist
  const fromAccount = await getAccount(input.fromAccountId);
  const toAccount = await getAccount(input.toAccountId);

  if (!fromAccount || !toAccount) {
    throw new Error('One or both accounts not found');
  }

  // Validate sufficient balance
  if (fromAccount.balance < input.amount) {
    throw new Error('Insufficient balance');
  }

  // Atomic transaction using DynamoDB TransactWriteItems
  try {
    await dynamodb.send(
      new TransactWriteItemsCommand({
        TransactItems: [
          // Deduct from source account
          {
            Update: {
              TableName: ACCOUNTS_TABLE,
              Key: { accountId: { S: input.fromAccountId } },
              UpdateExpression: 'SET balance = balance - :amount, updatedAt = :timestamp',
              ConditionExpression: 'balance >= :amount',
              ExpressionAttributeValues: {
                ':amount': { N: input.amount.toString() },
                ':timestamp': { S: timestamp }
              }
            }
          },
          // Add to destination account
          {
            Update: {
              TableName: ACCOUNTS_TABLE,
              Key: { accountId: { S: input.toAccountId } },
              UpdateExpression: 'SET balance = balance + :amount, updatedAt = :timestamp',
              ExpressionAttributeValues: {
                ':amount': { N: input.amount.toString() },
                ':timestamp': { S: timestamp }
              }
            }
          },
          // Create debit transaction
          {
            Put: {
              TableName: TRANSACTIONS_TABLE,
              Item: {
                transactionId: { S: `${transferId}-debit` },
                accountId: { S: input.fromAccountId },
                amount: { N: (-input.amount).toString() },
                type: { S: 'TRANSFER' },
                status: { S: 'COMPLETED' },
                description: { S: `Transfer to ${input.toAccountId}` },
                timestamp: { S: timestamp }
              }
            }
          },
          // Create credit transaction
          {
            Put: {
              TableName: TRANSACTIONS_TABLE,
              Item: {
                transactionId: { S: `${transferId}-credit` },
                accountId: { S: input.toAccountId },
                amount: { N: input.amount.toString() },
                type: { S: 'TRANSFER' },
                status: { S: 'COMPLETED' },
                description: { S: `Transfer from ${input.fromAccountId}` },
                timestamp: { S: timestamp }
              }
            }
          }
        ]
      })
    );

    return {
      transferId,
      fromAccountId: input.fromAccountId,
      toAccountId: input.toAccountId,
      amount: input.amount,
      currency: input.currency,
      status: 'COMPLETED',
      reference: input.reference,
      createdAt: timestamp
    };

  } catch (error: any) {
    console.error('Transfer failed:', error);
    
    if (error.name === 'TransactionCanceledException') {
      throw new Error('Transfer failed: Insufficient balance or account locked');
    }
    
    throw new Error(`Transfer failed: ${error.message}`);
  }
}

async function createPayment(input: any) {
  const paymentId = uuidv4();
  const timestamp = new Date().toISOString();

  // Implement payment logic
  // For now, return mock response
  return {
    paymentId,
    accountId: input.accountId,
    beneficiaryId: input.beneficiaryId,
    amount: input.amount,
    currency: input.currency,
    status: 'SCHEDULED',
    scheduledDate: input.scheduledDate,
    completedAt: null
  };
}

async function getAccount(accountId: string): Promise<any> {
  const response = await dynamodb.send(
    new GetItemCommand({
      TableName: ACCOUNTS_TABLE,
      Key: { accountId: { S: accountId } }
    })
  );

  if (!response.Item) return null;

  return {
    accountId: response.Item.accountId.S,
    balance: parseFloat(response.Item.balance.N || '0'),
    currency: response.Item.currency?.S || 'AED',
    status: response.Item.status?.S || 'ACTIVE'
  };
}
```

### 📱 GraphQL Client Usage

```typescript
// client/graphql-client.ts
import { ApolloClient, InMemoryCache, gql, HttpLink, split } from '@apollo/client';
import { GraphQLWsLink } from '@apollo/client/link/subscriptions';
import { getMainDefinition } from '@apollo/client/utilities';
import { createClient } from 'graphql-ws';

const httpLink = new HttpLink({
  uri: 'https://xxx.appsync-api.us-east-1.amazonaws.com/graphql',
  headers: {
    'x-api-key': 'da2-xxxxxxxxxxxxxxxxxx'
  }
});

const wsLink = new GraphQLWsLink(
  createClient({
    url: 'wss://xxx.appsync-realtime-api.us-east-1.amazonaws.com/graphql',
    connectionParams: {
      apiKey: 'da2-xxxxxxxxxxxxxxxxxx'
    }
  })
);

const splitLink = split(
  ({ query }) => {
    const definition = getMainDefinition(query);
    return (
      definition.kind === 'OperationDefinition' &&
      definition.operation === 'subscription'
    );
  },
  wsLink,
  httpLink
);

const client = new ApolloClient({
  link: splitLink,
  cache: new InMemoryCache()
});

// Query: Get account
const GET_ACCOUNT = gql`
  query GetAccount($accountId: ID!) {
    getAccount(accountId: $accountId) {
      accountId
      accountNumber
      accountType
      balance
      currency
      status
      customer {
        firstName
        lastName
        email
      }
    }
  }
`;

client.query({
  query: GET_ACCOUNT,
  variables: { accountId: 'ACC-12345' }
}).then(result => console.log(result.data));

// Mutation: Create transfer
const CREATE_TRANSFER = gql`
  mutation CreateTransfer($input: CreateTransferInput!) {
    createTransfer(input: $input) {
      transferId
      fromAccountId
      toAccountId
      amount
      status
      createdAt
    }
  }
`;

client.mutate({
  mutation: CREATE_TRANSFER,
  variables: {
    input: {
      fromAccountId: 'ACC-12345',
      toAccountId: 'ACC-67890',
      amount: 1000,
      currency: 'AED',
      reference: 'Payment for services'
    }
  }
}).then(result => console.log(result.data));

// Subscription: Real-time transactions
const TRANSACTION_SUBSCRIPTION = gql`
  subscription OnTransactionCreated($accountId: ID!) {
    onTransactionCreated(accountId: $accountId) {
      transactionId
      amount
      type
      status
      description
      timestamp
    }
  }
`;

client.subscribe({
  query: TRANSACTION_SUBSCRIPTION,
  variables: { accountId: 'ACC-12345' }
}).subscribe({
  next: ({ data }) => {
    console.log('New transaction:', data.onTransactionCreated);
    showNotification(`New transaction: ${data.onTransactionCreated.amount}`);
  },
  error: (error) => console.error('Subscription error:', error)
});
```

### 🎓 Interview Discussion Points - Q47

**Q1: GraphQL vs REST - when to use which?**

**A**:
**GraphQL:**
- Flexible queries (client specifies fields)
- Single endpoint for all operations
- Avoid over-fetching/under-fetching
- Real-time with subscriptions
- Use case: Mobile apps, dashboards

**REST:**
- Simple, widely adopted
- Better caching (HTTP)
- Easier to version
- Lightweight
- Use case: Public APIs, simple CRUD

**Q2: How do AppSync resolvers work?**

**A**:
- **Direct resolvers**: DynamoDB, Lambda, HTTP
- **VTL templates**: Request/response mapping
- **Pipeline resolvers**: Chain multiple data sources
- **Unit resolvers**: Single data source operation
- **Batch resolvers**: Resolve multiple items efficiently

**Q3: How to handle authentication in AppSync?**

**A**:
- **Cognito User Pools**: User authentication
- **API Key**: Public/test access
- **IAM**: Service-to-service
- **OIDC**: Third-party providers
- **Lambda Authorizers**: Custom logic
- **Multiple auth**: Different modes per field

**Q4: How to optimize GraphQL performance?**

**A**:
- **DataLoader**: Batch and cache requests
- **Caching**: AppSync caching (TTL-based)
- **Pagination**: Limit query results
- **Field-level caching**: Cache expensive fields
- **N+1 prevention**: Batch resolvers
- **Query complexity limits**: Prevent expensive queries

**Q5: How do GraphQL subscriptions work in AppSync?**

**A**:
- **MQTT over WebSocket**: Real-time protocol
- **@aws_subscribe**: Link to mutations
- **Filtering**: Client-side or server-side
- **Connection limits**: 100K concurrent connections
- **Pricing**: $0.08 per million connection-minutes + $1 per million messages
- **Auto-cleanup**: Idle timeout (10 minutes)

---

## Question 48: GraphQL Subscriptions & Real-time Updates

### 📋 Question Statement

Implement GraphQL subscriptions for Emirates NBD using AWS AppSync to provide real-time updates for account balance changes, transaction notifications, and payment status updates.

---

### 📡 GraphQL Subscription Schema

```graphql
# schema.graphql

type Subscription {
  onAccountBalanceChanged(accountId: ID!): AccountBalance
    @aws_subscribe(mutations: ["updateAccountBalance"])
  
  onTransactionCreated(accountId: ID!): Transaction
    @aws_subscribe(mutations: ["createTransaction"])
  
  onPaymentStatusChanged(paymentId: ID!): Payment
    @aws_subscribe(mutations: ["updatePaymentStatus"])
  
  onAlertTriggered(customerId: ID!): Alert
    @aws_subscribe(mutations: ["createAlert"])
}

type Mutation {
  updateAccountBalance(input: UpdateBalanceInput!): AccountBalance
  createTransaction(input: CreateTransactionInput!): Transaction
  updatePaymentStatus(input: UpdatePaymentInput!): Payment
  createAlert(input: CreateAlertInput!): Alert
}

type AccountBalance {
  accountId: ID!
  balance: Float!
  currency: String!
  updatedAt: AWSDateTime!
}

type Transaction {
  transactionId: ID!
  accountId: ID!
  amount: Float!
  type: TransactionType!
  status: TransactionStatus!
  merchant: String
  timestamp: AWSDateTime!
}

type Payment {
  paymentId: ID!
  fromAccount: ID!
  toAccount: ID!
  amount: Float!
  status: PaymentStatus!
  updatedAt: AWSDateTime!
}

type Alert {
  alertId: ID!
  customerId: ID!
  type: AlertType!
  message: String!
  severity: AlertSeverity!
  timestamp: AWSDateTime!
}

enum TransactionType {
  DEBIT
  CREDIT
  TRANSFER
}

enum TransactionStatus {
  PENDING
  COMPLETED
  FAILED
}

enum PaymentStatus {
  INITIATED
  PROCESSING
  COMPLETED
  FAILED
  CANCELLED
}

enum AlertType {
  LOW_BALANCE
  LARGE_TRANSACTION
  SUSPICIOUS_ACTIVITY
  PAYMENT_RECEIVED
}

enum AlertSeverity {
  INFO
  WARNING
  CRITICAL
}

input UpdateBalanceInput {
  accountId: ID!
  balance: Float!
  currency: String!
}

input CreateTransactionInput {
  accountId: ID!
  amount: Float!
  type: TransactionType!
  merchant: String
}

input UpdatePaymentInput {
  paymentId: ID!
  status: PaymentStatus!
}

input CreateAlertInput {
  customerId: ID!
  type: AlertType!
  message: String!
  severity: AlertSeverity!
}
```

### 🔧 AppSync CDK Stack with Subscriptions

```typescript
// infrastructure/cdk/appsync-subscription-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as appsync from 'aws-cdk-lib/aws-appsync';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as lambda from 'aws-cdk-lib/aws-lambda';

export class AppSyncSubscriptionStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // DynamoDB tables
    const accountsTable = new dynamodb.Table(this, 'Accounts', {
      partitionKey: { name: 'accountId', type: dynamodb.AttributeType.STRING },
      stream: dynamodb.StreamViewType.NEW_AND_OLD_IMAGES
    });

    const transactionsTable = new dynamodb.Table(this, 'Transactions', {
      partitionKey: { name: 'transactionId', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'timestamp', type: dynamodb.AttributeType.STRING },
      stream: dynamodb.StreamViewType.NEW_IMAGE
    });

    transactionsTable.addGlobalSecondaryIndex({
      indexName: 'AccountIdIndex',
      partitionKey: { name: 'accountId', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'timestamp', type: dynamodb.AttributeType.STRING }
    });

    // AppSync API
    const api = new appsync.GraphqlApi(this, 'BankingAPI', {
      name: 'emirates-nbd-banking-api',
      schema: appsync.SchemaFile.fromAsset('graphql/schema.graphql'),
      authorizationConfig: {
        defaultAuthorization: {
          authorizationType: appsync.AuthorizationType.API_KEY,
          apiKeyConfig: {
            expires: cdk.Expiration.after(cdk.Duration.days(365))
          }
        },
        additionalAuthorizationModes: [
          {
            authorizationType: appsync.AuthorizationType.USER_POOL,
            userPoolConfig: {
              userPool: props.userPool
            }
          }
        ]
      },
      logConfig: {
        fieldLogLevel: appsync.FieldLogLevel.ALL
      },
      xrayEnabled: true
    });

    // Data sources
    const accountsDataSource = api.addDynamoDbDataSource(
      'AccountsDataSource',
      accountsTable
    );

    const transactionsDataSource = api.addDynamoDbDataSource(
      'TransactionsDataSource',
      transactionsTable
    );

    // Lambda for complex business logic
    const transactionProcessor = new lambda.Function(this, 'TransactionProcessor', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/transaction-processor')
    });

    const lambdaDataSource = api.addLambdaDataSource(
      'TransactionProcessorDS',
      transactionProcessor
    );

    // Resolvers for mutations
    accountsDataSource.createResolver('UpdateAccountBalanceResolver', {
      typeName: 'Mutation',
      fieldName: 'updateAccountBalance',
      requestMappingTemplate: appsync.MappingTemplate.dynamoDbPutItem(
        appsync.PrimaryKey.partition('accountId').is('accountId'),
        appsync.Values.projecting()
      ),
      responseMappingTemplate: appsync.MappingTemplate.dynamoDbResultItem()
    });

    lambdaDataSource.createResolver('CreateTransactionResolver', {
      typeName: 'Mutation',
      fieldName: 'createTransaction'
    });

    // Subscription filters (server-side filtering)
    const subscriptionFilterTemplate = `
    #if($ctx.args.accountId == $ctx.result.accountId)
      $util.toJson($ctx.result)
    #else
      $util.toJson(null)
    #end
    `;

    new cdk.CfnOutput(this, 'GraphQLAPIURL', {
      value: api.graphqlUrl
    });

    new cdk.CfnOutput(this, 'APIKey', {
      value: api.apiKey || ''
    });
  }
}
```

### 🎯 Subscription Client Implementation

```typescript
// client/subscription-client.ts
import { AWSAppSyncClient, AUTH_TYPE } from 'aws-appsync';
import gql from 'graphql-tag';

export class BankingSubscriptionClient {
  private client: AWSAppSyncClient<any>;

  constructor(apiUrl: string, apiKey: string) {
    this.client = new AWSAppSyncClient({
      url: apiUrl,
      region: 'us-east-1',
      auth: {
        type: AUTH_TYPE.API_KEY,
        apiKey
      },
      disableOffline: true
    });
  }

  subscribeToAccountBalance(accountId: string, callback: (data: any) => void) {
    const subscription = this.client.subscribe({
      query: gql`
        subscription OnBalanceChanged($accountId: ID!) {
          onAccountBalanceChanged(accountId: $accountId) {
            accountId
            balance
            currency
            updatedAt
          }
        }
      `,
      variables: { accountId }
    });

    return subscription.subscribe({
      next: (data) => {
        console.log('Balance update received:', data);
        callback(data.data.onAccountBalanceChanged);
      },
      error: (error) => {
        console.error('Subscription error:', error);
      }
    });
  }

  subscribeToTransactions(accountId: string, callback: (transaction: any) => void) {
    const subscription = this.client.subscribe({
      query: gql`
        subscription OnTransactionCreated($accountId: ID!) {
          onTransactionCreated(accountId: $accountId) {
            transactionId
            accountId
            amount
            type
            status
            merchant
            timestamp
          }
        }
      `,
      variables: { accountId }
    });

    return subscription.subscribe({
      next: (data) => {
        console.log('New transaction:', data);
        callback(data.data.onTransactionCreated);
      },
      error: (error) => {
        console.error('Subscription error:', error);
      }
    });
  }

  subscribeToPaymentStatus(paymentId: string, callback: (payment: any) => void) {
    const subscription = this.client.subscribe({
      query: gql`
        subscription OnPaymentStatusChanged($paymentId: ID!) {
          onPaymentStatusChanged(paymentId: $paymentId) {
            paymentId
            status
            updatedAt
          }
        }
      `,
      variables: { paymentId }
    });

    return subscription.subscribe({
      next: (data) => {
        console.log('Payment status update:', data);
        callback(data.data.onPaymentStatusChanged);
      },
      error: (error) => {
        console.error('Subscription error:', error);
      }
    });
  }

  async triggerBalanceUpdate(accountId: string, newBalance: number) {
    const mutation = gql`
      mutation UpdateBalance($input: UpdateBalanceInput!) {
        updateAccountBalance(input: $input) {
          accountId
          balance
          currency
          updatedAt
        }
      }
    `;

    const result = await this.client.mutate({
      mutation,
      variables: {
        input: {
          accountId,
          balance: newBalance,
          currency: 'AED'
        }
      }
    });

    return result.data.updateAccountBalance;
  }
}
```

### 🎓 Interview Discussion Points - Q48

**Q1: How do GraphQL subscriptions work in AppSync?**

**A**:
- **MQTT over WebSocket**: Real-time protocol
- **Publish on mutation**: Mutations trigger subscriptions
- **Server-side filtering**: Filter by arguments (e.g., accountId)
- **Auto-reconnect**: Client handles connection drops
- **Cost**: $2.00 per million connection minutes

**Q2: What is subscription filtering?**

**A**:
```graphql
subscription OnTransaction($accountId: ID!) {
  onTransactionCreated(accountId: $accountId) {
    transactionId
    amount
  }
}
```
Only receive updates for specific `accountId`, not all transactions

**Q3: How to handle subscription connection limits?**

**A**:
- **Limit**: 100,000 concurrent connections
- **Connection pooling**: Group users
- **Fallback**: Polling for overflow
- **Regional endpoints**: Scale across regions

**Q4: What is the difference between query and subscription?**

**A**:
- **Query**: One-time request/response
- **Subscription**: Long-lived connection, receives push updates
- **Use query**: Initial data load
- **Use subscription**: Real-time updates (balance changes, alerts)

**Q5: How to implement subscription authorization?**

**A**:
```typescript
// Lambda authorizer checks if user owns accountId
if (event.accountId !== userContext.accountId) {
  throw new Error('Unauthorized');
}
```
**Per-field auth**: `@aws_auth`, `@aws_cognito_user_pools`

---

**End of File 24**

