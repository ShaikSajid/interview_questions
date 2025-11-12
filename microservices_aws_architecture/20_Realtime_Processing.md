# Real-time Processing

## Question 39: Real-time Data Processing with Kinesis

### 📋 Question Statement

Implement real-time transaction processing for Emirates NBD using Kinesis Data Streams, Kinesis Data Analytics, and Lambda for fraud detection and analytics.

---

### 🌊 Kinesis Real-time Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│               KINESIS REAL-TIME TRANSACTION PROCESSING                      │
└────────────────────────────────────────────────────────────────────────────┘

    ┌─────────────┐         ┌──────────────────┐
    │   ATM/POS   │────────>│  API Gateway     │
    │  Terminals  │         │  /transactions   │
    └─────────────┘         └────────┬─────────┘
                                     │
                                     v
                            ┌────────────────┐
                            │ Kinesis Stream │
                            │   (Shard 1-4)  │
                            └───┬────────────┘
                                │
                  ┌─────────────┼─────────────┬─────────────┐
                  │             │             │             │
                  v             v             v             v
          ┌───────────┐  ┌───────────┐  ┌──────────┐  ┌─────────┐
          │  Lambda   │  │  Lambda   │  │ Kinesis  │  │  Lambda │
          │  Fraud    │  │ Analytics │  │  Firehose│  │ Notifier│
          │ Detection │  │  Metrics  │  │  (S3)    │  └─────────┘
          └───────────┘  └───────────┘  └──────────┘
                │              │
                v              v
          ┌──────────┐   ┌─────────┐
          │  DynamoDB│   │CloudWatch│
          │  Alerts  │   │ Metrics  │
          └──────────┘   └─────────┘
```

### 📦 Kinesis Stream CDK Infrastructure

```typescript
// infrastructure/cdk/kinesis-processing-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as kinesis from 'aws-cdk-lib/aws-kinesis';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as lambdaEventSources from 'aws-cdk-lib/aws-lambda-event-sources';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as subs from 'aws-cdk-lib/aws-sns-subscriptions';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as logs from 'aws-cdk-lib/aws-logs';

export class KinesisProcessingStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Kinesis Data Stream for transactions
    const transactionStream = new kinesis.Stream(this, 'TransactionStream', {
      streamName: 'banking-transactions',
      shardCount: 4, // 4 MB/s write, 8 MB/s read capacity
      retentionPeriod: cdk.Duration.hours(24),
      streamMode: kinesis.StreamMode.PROVISIONED,
      encryption: kinesis.StreamEncryption.MANAGED
    });

    // DynamoDB for fraud alerts
    const fraudAlertsTable = new dynamodb.Table(this, 'FraudAlerts', {
      tableName: 'fraud-alerts',
      partitionKey: { name: 'transactionId', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'timestamp', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      stream: dynamodb.StreamViewType.NEW_AND_OLD_IMAGES,
      timeToLiveAttribute: 'ttl'
    });

    // Add GSI for customer queries
    fraudAlertsTable.addGlobalSecondaryIndex({
      indexName: 'customerIdIndex',
      partitionKey: { name: 'customerId', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'timestamp', type: dynamodb.AttributeType.STRING }
    });

    // SNS Topic for fraud alerts
    const fraudAlertTopic = new sns.Topic(this, 'FraudAlertTopic', {
      topicName: 'fraud-alert-notifications',
      displayName: 'Emirates NBD Fraud Alerts'
    });

    // Lambda for fraud detection
    const fraudDetectionFunction = new lambda.Function(this, 'FraudDetection', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/fraud-detection'),
      timeout: cdk.Duration.seconds(30),
      memorySize: 1024,
      environment: {
        FRAUD_ALERTS_TABLE: fraudAlertsTable.tableName,
        FRAUD_ALERT_TOPIC: fraudAlertTopic.topicArn,
        FRAUD_THRESHOLD_AMOUNT: '10000',
        FRAUD_THRESHOLD_VELOCITY: '5' // 5 transactions in 5 minutes
      },
      logRetention: logs.RetentionDays.ONE_WEEK
    });

    // Grant permissions
    fraudAlertsTable.grantWriteData(fraudDetectionFunction);
    fraudAlertTopic.grantPublish(fraudDetectionFunction);

    // Attach Kinesis as event source
    fraudDetectionFunction.addEventSource(
      new lambdaEventSources.KinesisEventSource(transactionStream, {
        startingPosition: lambda.StartingPosition.TRIM_HORIZON,
        batchSize: 100,
        maxBatchingWindow: cdk.Duration.seconds(5),
        bisectBatchOnError: true,
        retryAttempts: 3,
        reportBatchItemFailures: true
      })
    );

    // Lambda for analytics and metrics
    const analyticsFunction = new lambda.Function(this, 'TransactionAnalytics', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/analytics'),
      timeout: cdk.Duration.seconds(30),
      memorySize: 512,
      environment: {
        METRICS_NAMESPACE: 'EmiratesNBD/Transactions'
      }
    });

    analyticsFunction.addEventSource(
      new lambdaEventSources.KinesisEventSource(transactionStream, {
        startingPosition: lambda.StartingPosition.TRIM_HORIZON,
        batchSize: 500,
        maxBatchingWindow: cdk.Duration.seconds(10),
        parallelizationFactor: 10
      })
    );

    // Lambda for notifications
    const notificationFunction = new lambda.Function(this, 'NotificationService', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/notifications'),
      timeout: cdk.Duration.seconds(10),
      memorySize: 256
    });

    notificationFunction.addEventSource(
      new lambdaEventSources.KinesisEventSource(transactionStream, {
        startingPosition: lambda.StartingPosition.TRIM_HORIZON,
        batchSize: 50,
        filters: [
          lambda.FilterCriteria.filter({
            data: {
              amount: lambda.FilterRule.greaterThan(1000)
            }
          })
        ]
      })
    );

    // Output stream ARN
    new cdk.CfnOutput(this, 'StreamArn', {
      value: transactionStream.streamArn,
      description: 'Transaction Stream ARN'
    });

    new cdk.CfnOutput(this, 'StreamName', {
      value: transactionStream.streamName,
      description: 'Transaction Stream Name'
    });
  }
}
```

### 🚨 Fraud Detection Lambda

```typescript
// lambda/fraud-detection/index.ts
import { KinesisStreamHandler, KinesisStreamRecord } from 'aws-lambda';
import { DynamoDBClient, PutItemCommand, QueryCommand } from '@aws-sdk/client-dynamodb';
import { SNSClient, PublishCommand } from '@aws-sdk/client-sns';

const dynamodb = new DynamoDBClient({});
const sns = new SNSClient({});

const FRAUD_ALERTS_TABLE = process.env.FRAUD_ALERTS_TABLE!;
const FRAUD_ALERT_TOPIC = process.env.FRAUD_ALERT_TOPIC!;
const FRAUD_THRESHOLD_AMOUNT = parseFloat(process.env.FRAUD_THRESHOLD_AMOUNT || '10000');
const FRAUD_THRESHOLD_VELOCITY = parseInt(process.env.FRAUD_THRESHOLD_VELOCITY || '5');

export const handler: KinesisStreamHandler = async (event) => {
  const results = [];

  for (const record of event.Records) {
    try {
      const transaction = parseTransaction(record);
      
      // Run fraud detection rules
      const fraudChecks = await Promise.all([
        checkHighValueTransaction(transaction),
        checkVelocityAnomaly(transaction),
        checkUnusualLocation(transaction),
        checkUnusualTime(transaction)
      ]);

      const isFraudulent = fraudChecks.some(check => check.isFraud);
      const fraudScore = fraudChecks.reduce((sum, check) => sum + check.score, 0);

      if (isFraudulent || fraudScore > 70) {
        await handleFraudAlert(transaction, fraudChecks, fraudScore);
      }

      results.push({ itemIdentifier: record.kinesis.sequenceNumber });

    } catch (error) {
      console.error('Error processing record:', error);
      results.push({ 
        itemIdentifier: record.kinesis.sequenceNumber,
        batchItemFailures: [{ itemIdentifier: record.kinesis.sequenceNumber }]
      });
    }
  }

  return { batchItemFailures: results.filter(r => r.batchItemFailures).map(r => ({ itemIdentifier: r.itemIdentifier })) };
};

interface Transaction {
  transactionId: string;
  customerId: string;
  accountId: string;
  amount: number;
  currency: string;
  merchantId: string;
  merchantCategory: string;
  location: {
    country: string;
    city: string;
    latitude: number;
    longitude: number;
  };
  timestamp: string;
  type: 'CARD_PURCHASE' | 'ATM_WITHDRAWAL' | 'TRANSFER';
}

interface FraudCheck {
  ruleName: string;
  isFraud: boolean;
  score: number;
  reason: string;
}

function parseTransaction(record: KinesisStreamRecord): Transaction {
  const data = Buffer.from(record.kinesis.data, 'base64').toString('utf-8');
  return JSON.parse(data);
}

async function checkHighValueTransaction(transaction: Transaction): Promise<FraudCheck> {
  const isHighValue = transaction.amount > FRAUD_THRESHOLD_AMOUNT;

  return {
    ruleName: 'HIGH_VALUE_TRANSACTION',
    isFraud: isHighValue,
    score: isHighValue ? 50 : 0,
    reason: isHighValue 
      ? `Transaction amount ${transaction.amount} exceeds threshold ${FRAUD_THRESHOLD_AMOUNT}`
      : 'Normal transaction amount'
  };
}

async function checkVelocityAnomaly(transaction: Transaction): Promise<FraudCheck> {
  // Check recent transactions for this customer
  const fiveMinutesAgo = new Date(Date.now() - 5 * 60 * 1000).toISOString();

  const response = await dynamodb.send(
    new QueryCommand({
      TableName: FRAUD_ALERTS_TABLE,
      IndexName: 'customerIdIndex',
      KeyConditionExpression: 'customerId = :customerId AND #ts > :timestamp',
      ExpressionAttributeNames: {
        '#ts': 'timestamp'
      },
      ExpressionAttributeValues: {
        ':customerId': { S: transaction.customerId },
        ':timestamp': { S: fiveMinutesAgo }
      }
    })
  );

  const recentCount = response.Items?.length || 0;
  const isVelocityAnomaly = recentCount >= FRAUD_THRESHOLD_VELOCITY;

  return {
    ruleName: 'VELOCITY_ANOMALY',
    isFraud: isVelocityAnomaly,
    score: isVelocityAnomaly ? 40 : 0,
    reason: isVelocityAnomaly
      ? `${recentCount} transactions in last 5 minutes (threshold: ${FRAUD_THRESHOLD_VELOCITY})`
      : 'Normal transaction velocity'
  };
}

async function checkUnusualLocation(transaction: Transaction): Promise<FraudCheck> {
  // Check if location is unusual for this customer
  // (In production, this would query historical patterns)
  
  const isUnusualLocation = transaction.location.country !== 'UAE';

  return {
    ruleName: 'UNUSUAL_LOCATION',
    isFraud: isUnusualLocation,
    score: isUnusualLocation ? 30 : 0,
    reason: isUnusualLocation
      ? `Transaction from ${transaction.location.country} (expected UAE)`
      : 'Normal location'
  };
}

async function checkUnusualTime(transaction: Transaction): Promise<FraudCheck> {
  const hour = new Date(transaction.timestamp).getHours();
  const isUnusualTime = hour >= 2 && hour <= 5; // 2 AM - 5 AM

  return {
    ruleName: 'UNUSUAL_TIME',
    isFraud: isUnusualTime,
    score: isUnusualTime ? 20 : 0,
    reason: isUnusualTime
      ? `Transaction at ${hour}:00 (unusual hours)`
      : 'Normal transaction time'
  };
}

async function handleFraudAlert(
  transaction: Transaction, 
  checks: FraudCheck[], 
  fraudScore: number
): Promise<void> {
  // Store fraud alert in DynamoDB
  await dynamodb.send(
    new PutItemCommand({
      TableName: FRAUD_ALERTS_TABLE,
      Item: {
        transactionId: { S: transaction.transactionId },
        customerId: { S: transaction.customerId },
        timestamp: { S: transaction.timestamp },
        amount: { N: transaction.amount.toString() },
        fraudScore: { N: fraudScore.toString() },
        checks: { S: JSON.stringify(checks) },
        status: { S: 'PENDING_REVIEW' },
        ttl: { N: Math.floor(Date.now() / 1000 + 30 * 24 * 60 * 60).toString() } // 30 days TTL
      }
    })
  );

  // Send SNS notification
  await sns.send(
    new PublishCommand({
      TopicArn: FRAUD_ALERT_TOPIC,
      Subject: `⚠️ Fraud Alert - Transaction ${transaction.transactionId}`,
      Message: JSON.stringify({
        transactionId: transaction.transactionId,
        customerId: transaction.customerId,
        amount: transaction.amount,
        fraudScore,
        checks: checks.filter(c => c.isFraud),
        timestamp: transaction.timestamp
      }, null, 2)
    })
  );

  console.log(`Fraud alert created for transaction ${transaction.transactionId}, score: ${fraudScore}`);
}
```

### 📊 Analytics Lambda

```typescript
// lambda/analytics/index.ts
import { KinesisStreamHandler } from 'aws-lambda';
import { CloudWatchClient, PutMetricDataCommand, MetricDatum } from '@aws-sdk/client-cloudwatch';

const cloudwatch = new CloudWatchClient({});
const METRICS_NAMESPACE = process.env.METRICS_NAMESPACE || 'EmiratesNBD/Transactions';

export const handler: KinesisStreamHandler = async (event) => {
  const metrics = new Map<string, MetricDatum[]>();

  for (const record of event.Records) {
    const data = Buffer.from(record.kinesis.data, 'base64').toString('utf-8');
    const transaction = JSON.parse(data);

    // Aggregate metrics
    addMetric(metrics, 'TransactionCount', 1, [
      { Name: 'Type', Value: transaction.type },
      { Name: 'Currency', Value: transaction.currency }
    ]);

    addMetric(metrics, 'TransactionAmount', transaction.amount, [
      { Name: 'Type', Value: transaction.type },
      { Name: 'Currency', Value: transaction.currency }
    ]);

    addMetric(metrics, 'TransactionsByMerchantCategory', 1, [
      { Name: 'MerchantCategory', Value: transaction.merchantCategory }
    ]);
  }

  // Publish metrics to CloudWatch
  for (const [metricName, metricData] of metrics) {
    await cloudwatch.send(
      new PutMetricDataCommand({
        Namespace: METRICS_NAMESPACE,
        MetricData: metricData
      })
    );
  }

  console.log(`Published ${metrics.size} metric types to CloudWatch`);
};

function addMetric(
  metrics: Map<string, MetricDatum[]>,
  metricName: string,
  value: number,
  dimensions: { Name: string; Value: string }[]
): void {
  if (!metrics.has(metricName)) {
    metrics.set(metricName, []);
  }

  metrics.get(metricName)!.push({
    MetricName: metricName,
    Value: value,
    Unit: 'None',
    Timestamp: new Date(),
    Dimensions: dimensions
  });
}
```

### 🎓 Interview Discussion Points - Q39

**Q1: When to use Kinesis vs SQS?**

**A**:
**Kinesis:**
- Real-time streaming data (logs, metrics, transactions)
- Multiple consumers reading same data
- Ordered data within shard
- Replay capability (24 hours retention)
- Use case: Real-time analytics, fraud detection

**SQS:**
- Decoupled message queue
- Single consumer per message
- No ordering guarantee (except FIFO)
- Message deleted after processing
- Use case: Task queues, async processing

**Q2: How to handle Kinesis Lambda errors?**

**A**:
- **Batch item failures**: Return failed sequence numbers
- **Bisect on error**: Split batch in half to isolate failures
- **Retry attempts**: Configure max retries (default 3)
- **DLQ**: Send failed records to DLQ after retries
- **Monitoring**: CloudWatch metrics for iterator age, failures

**Q3: How to scale Kinesis processing?**

**A**:
- **Shard count**: More shards = more throughput (1 MB/s per shard)
- **Lambda parallelization**: Up to 10 concurrent executions per shard
- **Batch size**: Larger batches = fewer invocations (max 10,000 records)
- **Enhanced fan-out**: Dedicated read throughput per consumer (2 MB/s)
- **Auto-scaling**: Use CloudWatch metrics to scale shards

**Q4: What is Kinesis shard iterator?**

**A**:
- Pointer to position in shard
- Types:
  - **TRIM_HORIZON**: Start from oldest record
  - **LATEST**: Start from newest record
  - **AT_SEQUENCE_NUMBER**: Start from specific sequence
  - **AFTER_SEQUENCE_NUMBER**: After specific sequence
  - **AT_TIMESTAMP**: Start from timestamp

**Q5: How to handle hot shards?**

**A**:
- **Better partition key**: Use high-cardinality key (e.g., customer ID)
- **Shard splitting**: Split hot shard into two
- **Random suffix**: Add random suffix to partition key
- **Enhanced fan-out**: Avoid consumer throttling
- **Monitor**: Track `IncomingRecords` per shard

---

## Question 40: Kinesis Data Firehose & S3 Data Lake

### 📋 Question Statement

Implement streaming data ingestion to S3 data lake for Emirates NBD using Kinesis Data Firehose with Lambda transformations, Parquet conversion, and Athena queries for analytics.

---

### 🔥 Kinesis Firehose CDK Stack

```typescript
// infrastructure/cdk/firehose-data-lake-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as firehose from 'aws-cdk-lib/aws-kinesisfirehose';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as glue from 'aws-cdk-lib/aws-glue';
import * as logs from 'aws-cdk-lib/aws-logs';

export class FirehoseDataLakeStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // S3 Data Lake Bucket
    const dataLakeBucket = new s3.Bucket(this, 'DataLakeBucket', {
      bucketName: 'emirates-nbd-datalake',
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      lifecycleRules: [
        {
          transitions: [
            {
              storageClass: s3.StorageClass.INTELLIGENT_TIERING,
              transitionAfter: cdk.Duration.days(30)
            },
            {
              storageClass: s3.StorageClass.GLACIER,
              transitionAfter: cdk.Duration.days(90)
            }
          ]
        }
      ]
    });

    // Lambda for data transformation
    const transformationFunction = new lambda.Function(this, 'DataTransformation', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/firehose-transformation'),
      timeout: cdk.Duration.minutes(3),
      memorySize: 512
    });

    // Glue Database for Athena
    const glueDatabase = new glue.CfnDatabase(this, 'TransactionsDB', {
      catalogId: this.account,
      databaseInput: {
        name: 'banking_transactions',
        description: 'Transaction data lake'
      }
    });

    // Glue Table for Transactions
    const glueTable = new glue.CfnTable(this, 'TransactionsTable', {
      catalogId: this.account,
      databaseName: glueDatabase.ref,
      tableInput: {
        name: 'transactions',
        storageDescriptor: {
          columns: [
            { name: 'transaction_id', type: 'string' },
            { name: 'account_id', type: 'string' },
            { name: 'amount', type: 'decimal(18,2)' },
            { name: 'type', type: 'string' },
            { name: 'status', type: 'string' },
            { name: 'timestamp', type: 'timestamp' },
            { name: 'merchant', type: 'string' },
            { name: 'category', type: 'string' }
          ],
          location: `s3://${dataLakeBucket.bucketName}/transactions/`,
          inputFormat: 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat',
          outputFormat: 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat',
          serdeInfo: {
            serializationLibrary: 'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe'
          }
        },
        partitionKeys: [
          { name: 'year', type: 'string' },
          { name: 'month', type: 'string' },
          { name: 'day', type: 'string' }
        ],
        tableType: 'EXTERNAL_TABLE'
      }
    });

    // Firehose Delivery Stream IAM Role
    const firehoseRole = new iam.Role(this, 'FirehoseRole', {
      assumedBy: new iam.ServicePrincipal('firehose.amazonaws.com')
    });

    dataLakeBucket.grantReadWrite(firehoseRole);
    transformationFunction.grantInvoke(firehoseRole);

    // Kinesis Data Firehose Delivery Stream
    const deliveryStream = new firehose.CfnDeliveryStream(this, 'TransactionStream', {
      deliveryStreamName: 'banking-transactions-stream',
      deliveryStreamType: 'DirectPut',
      extendedS3DestinationConfiguration: {
        bucketArn: dataLakeBucket.bucketArn,
        roleArn: firehoseRole.roleArn,
        prefix: 'transactions/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/',
        errorOutputPrefix: 'errors/!{firehose:error-output-type}/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/',
        bufferingHints: {
          sizeInMBs: 128,
          intervalInSeconds: 300 // 5 minutes
        },
        compressionFormat: 'UNCOMPRESSED', // Parquet handles compression
        dataFormatConversionConfiguration: {
          enabled: true,
          schemaConfiguration: {
            databaseName: glueDatabase.ref,
            tableName: glueTable.ref,
            roleArn: firehoseRole.roleArn
          },
          inputFormatConfiguration: {
            deserializer: {
              openXJsonSerDe: {}
            }
          },
          outputFormatConfiguration: {
            serializer: {
              parquetSerDe: {
                compression: 'SNAPPY'
              }
            }
          }
        },
        processingConfiguration: {
          enabled: true,
          processors: [
            {
              type: 'Lambda',
              parameters: [
                {
                  parameterName: 'LambdaArn',
                  parameterValue: transformationFunction.functionArn
                },
                {
                  parameterName: 'BufferSizeInMBs',
                  parameterValue: '3'
                },
                {
                  parameterName: 'BufferIntervalInSeconds',
                  parameterValue: '60'
                }
              ]
            }
          ]
        },
        cloudWatchLoggingOptions: {
          enabled: true,
          logGroupName: '/aws/kinesisfirehose/banking-transactions',
          logStreamName: 'S3Delivery'
        }
      }
    });

    new cdk.CfnOutput(this, 'DeliveryStreamName', {
      value: deliveryStream.ref
    });

    new cdk.CfnOutput(this, 'DataLakeBucket', {
      value: dataLakeBucket.bucketName
    });
  }
}
```

### 🔄 Firehose Transformation Lambda

```typescript
// lambda/firehose-transformation/index.ts
import { FirehoseTransformationEvent, FirehoseTransformationResult } from 'aws-lambda';

export const handler = async (
  event: FirehoseTransformationEvent
): Promise<FirehoseTransformationResult> => {
  const output = event.records.map(record => {
    try {
      // Decode base64 payload
      const payload = Buffer.from(record.data, 'base64').toString('utf8');
      const data = JSON.parse(payload);

      // Transform and enrich data
      const transformed = {
        transaction_id: data.transactionId,
        account_id: data.accountId,
        amount: parseFloat(data.amount),
        type: data.type,
        status: data.status,
        timestamp: new Date(data.timestamp).toISOString(),
        merchant: data.merchant || 'N/A',
        category: categorizeTransaction(data),
        risk_score: calculateRiskScore(data),
        enriched_at: new Date().toISOString()
      };

      // Add newline for JSON records (required for Athena)
      const transformedData = JSON.stringify(transformed) + '\n';

      return {
        recordId: record.recordId,
        result: 'Ok',
        data: Buffer.from(transformedData).toString('base64')
      };
    } catch (error) {
      console.error('Transformation failed:', error);
      return {
        recordId: record.recordId,
        result: 'ProcessingFailed',
        data: record.data
      };
    }
  });

  return { records: output };
};

function categorizeTransaction(data: any): string {
  if (data.merchant) {
    if (data.merchant.includes('GROCERY')) return 'GROCERIES';
    if (data.merchant.includes('RESTAURANT')) return 'DINING';
    if (data.merchant.includes('GAS')) return 'FUEL';
    if (data.merchant.includes('AMAZON')) return 'SHOPPING';
  }
  return 'OTHER';
}

function calculateRiskScore(data: any): number {
  let score = 0;
  
  if (data.amount > 10000) score += 30;
  if (data.type === 'INTERNATIONAL') score += 20;
  if (isUnusualTime(data.timestamp)) score += 15;
  if (data.merchant?.includes('CRYPTO')) score += 25;
  
  return Math.min(score, 100);
}

function isUnusualTime(timestamp: string): boolean {
  const hour = new Date(timestamp).getHours();
  return hour < 6 || hour > 23; // 2 AM - 6 AM
}
```

### 📊 Athena Query Service

```typescript
// services/analytics/athena-query-service.ts
import { Athena, S3 } from 'aws-sdk';

export class AthenaQueryService {
  private athena: Athena;
  private s3: S3;
  private outputBucket: string;
  private database: string;

  constructor(outputBucket: string, database: string = 'banking_transactions') {
    this.athena = new Athena();
    this.s3 = new S3();
    this.outputBucket = outputBucket;
    this.database = database;
  }

  async executeQuery(query: string): Promise<QueryResult> {
    // Start query execution
    const execution = await this.athena.startQueryExecution({
      QueryString: query,
      QueryExecutionContext: {
        Database: this.database
      },
      ResultConfiguration: {
        OutputLocation: `s3://${this.outputBucket}/athena-results/`
      }
    }).promise();

    const queryExecutionId = execution.QueryExecutionId!;

    // Wait for query to complete
    await this.waitForQueryCompletion(queryExecutionId);

    // Get results
    const results = await this.athena.getQueryResults({
      QueryExecutionId: queryExecutionId
    }).promise();

    return this.parseResults(results);
  }

  async getDailyTransactionVolume(date: string): Promise<TransactionVolume> {
    const [year, month, day] = date.split('-');
    
    const query = `
      SELECT 
        type,
        COUNT(*) as transaction_count,
        SUM(amount) as total_amount,
        AVG(amount) as avg_amount,
        MAX(amount) as max_amount
      FROM transactions
      WHERE year = '${year}'
        AND month = '${month}'
        AND day = '${day}'
      GROUP BY type
      ORDER BY total_amount DESC
    `;

    const result = await this.executeQuery(query);

    return {
      date,
      byType: result.rows.map(row => ({
        type: row.type,
        count: parseInt(row.transaction_count),
        totalAmount: parseFloat(row.total_amount),
        avgAmount: parseFloat(row.avg_amount),
        maxAmount: parseFloat(row.max_amount)
      }))
    };
  }

  async getTopMerchants(startDate: string, endDate: string, limit: number = 10) {
    const query = `
      SELECT 
        merchant,
        COUNT(*) as transaction_count,
        SUM(amount) as total_spent,
        AVG(amount) as avg_transaction
      FROM transactions
      WHERE timestamp BETWEEN TIMESTAMP '${startDate}' AND TIMESTAMP '${endDate}'
        AND merchant != 'N/A'
      GROUP BY merchant
      ORDER BY total_spent DESC
      LIMIT ${limit}
    `;

    return await this.executeQuery(query);
  }

  async getHighRiskTransactions(threshold: number = 70): Promise<RiskTransaction[]> {
    const query = `
      SELECT 
        transaction_id,
        account_id,
        amount,
        merchant,
        risk_score,
        timestamp
      FROM transactions
      WHERE risk_score >= ${threshold}
        AND timestamp >= CURRENT_TIMESTAMP - INTERVAL '24' HOUR
      ORDER BY risk_score DESC, timestamp DESC
    `;

    const result = await this.executeQuery(query);

    return result.rows.map(row => ({
      transactionId: row.transaction_id,
      accountId: row.account_id,
      amount: parseFloat(row.amount),
      merchant: row.merchant,
      riskScore: parseInt(row.risk_score),
      timestamp: row.timestamp
    }));
  }

  async getCategoryBreakdown(accountId: string, month: string) {
    const [year, mon] = month.split('-');
    
    const query = `
      SELECT 
        category,
        COUNT(*) as count,
        SUM(amount) as total,
        ROUND(SUM(amount) * 100.0 / SUM(SUM(amount)) OVER(), 2) as percentage
      FROM transactions
      WHERE account_id = '${accountId}'
        AND year = '${year}'
        AND month = '${mon}'
      GROUP BY category
      ORDER BY total DESC
    `;

    return await this.executeQuery(query);
  }

  private async waitForQueryCompletion(queryExecutionId: string): Promise<void> {
    while (true) {
      const status = await this.athena.getQueryExecution({
        QueryExecutionId: queryExecutionId
      }).promise();

      const state = status.QueryExecution?.Status?.State;

      if (state === 'SUCCEEDED') {
        return;
      } else if (state === 'FAILED' || state === 'CANCELLED') {
        throw new Error(`Query ${state}: ${status.QueryExecution?.Status?.StateChangeReason}`);
      }

      await this.sleep(1000);
    }
  }

  private parseResults(results: Athena.GetQueryResultsOutput): QueryResult {
    const rows = results.ResultSet?.Rows || [];
    
    if (rows.length === 0) {
      return { columns: [], rows: [] };
    }

    const columns = rows[0].Data?.map(col => col.VarCharValue || '') || [];
    const dataRows = rows.slice(1).map(row => {
      const obj: Record<string, string> = {};
      row.Data?.forEach((cell, index) => {
        obj[columns[index]] = cell.VarCharValue || '';
      });
      return obj;
    });

    return {
      columns,
      rows: dataRows
    };
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

interface QueryResult {
  columns: string[];
  rows: Record<string, string>[];
}

interface TransactionVolume {
  date: string;
  byType: Array<{
    type: string;
    count: number;
    totalAmount: number;
    avgAmount: number;
    maxAmount: number;
  }>;
}

interface RiskTransaction {
  transactionId: string;
  accountId: string;
  amount: number;
  merchant: string;
  riskScore: number;
  timestamp: string;
}
```

### 📈 Data Lake Analytics Dashboard

```typescript
// services/analytics/dashboard-service.ts
import { AthenaQueryService } from './athena-query-service';
import { CloudWatch } from 'aws-sdk';

export class DataLakeAnalyticsDashboard {
  private athena: AthenaQueryService;
  private cloudwatch: CloudWatch;

  constructor(outputBucket: string) {
    this.athena = new AthenaQueryService(outputBucket);
    this.cloudwatch = new CloudWatch();
  }

  async generateDailyReport(date: string): Promise<DailyReport> {
    // Get transaction volume
    const volume = await this.athena.getDailyTransactionVolume(date);

    // Get high risk transactions
    const highRisk = await this.athena.getHighRiskTransactions(80);

    // Get top merchants
    const topMerchants = await this.athena.getTopMerchants(
      `${date} 00:00:00`,
      `${date} 23:59:59`,
      10
    );

    return {
      date,
      transactionVolume: volume,
      highRiskCount: highRisk.length,
      topMerchants: topMerchants.rows.slice(0, 5)
    };
  }

  async publishMetricsToCloudWatch(metrics: DailyReport): Promise<void> {
    await this.cloudwatch.putMetricData({
      Namespace: 'Banking/Analytics',
      MetricData: [
        {
          MetricName: 'DailyTransactionCount',
          Value: metrics.transactionVolume.byType.reduce((sum, t) => sum + t.count, 0),
          Unit: 'Count',
          Timestamp: new Date()
        },
        {
          MetricName: 'HighRiskTransactionCount',
          Value: metrics.highRiskCount,
          Unit: 'Count',
          Timestamp: new Date()
        }
      ]
    }).promise();
  }
}

interface DailyReport {
  date: string;
  transactionVolume: any;
  highRiskCount: number;
  topMerchants: any[];
}
```

### 🎓 Interview Discussion Points - Q40

**Q1: What is Kinesis Data Firehose vs Kinesis Data Streams?**

**A**:
- **Firehose**: Fully managed, auto-scaling, delivers to S3/Redshift/ElasticSearch
- **Streams**: Lower level, manual shard management, custom consumers
- **Use Firehose**: Simple S3/data lake ingestion
- **Use Streams**: Custom processing, multiple consumers, sub-second latency

**Q2: Why convert JSON to Parquet?**

**A**:
- **Compression**: 10x smaller than JSON
- **Query performance**: Columnar format, read only needed columns
- **Cost**: Less storage, less data scanned by Athena
- **Schema evolution**: Easy to add columns
- **Example**: 1GB JSON → 100MB Parquet

**Q3: How does Firehose buffering work?**

**A**:
- **Size**: Buffer until 128MB (configurable)
- **Time**: Buffer until 5 minutes (configurable)
- **Whichever first**: Size OR time triggers delivery
- **Trade-off**: Larger buffer = fewer S3 objects but higher latency

**Q4: What is partition projection in Athena?**

**A**:
```sql
-- Partition by year/month/day
s3://bucket/transactions/year=2025/month=11/day=11/

-- Athena automatically filters:
SELECT * FROM transactions
WHERE year = '2025' AND month = '11' AND day = '11'
```
**Benefit**: Faster queries, lower cost (scans less data)

**Q5: How to handle late-arriving data?**

**A**:
- **Grace period**: Accept data up to N hours late
- **Reprocessing**: Run daily job to backfill previous day
- **Partitioning**: Use event timestamp, not ingestion time
- **Firehose prefix**: `year=!{timestamp:yyyy}/month=!{timestamp:MM}/`

---

