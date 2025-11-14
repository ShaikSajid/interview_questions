# AWS Interview Questions: Data & Analytics Services (Q22-Q24)

## Question 22: Explain Amazon Aurora and its key features

### 📋 Answer

**Amazon Aurora** is a MySQL and PostgreSQL-compatible relational database built for the cloud, combining the performance of high-end commercial databases with the simplicity of open-source databases.

### Aurora Architecture:

```
Amazon Aurora
├── Compute Layer
│   ├── Primary Instance (Writer)
│   └── Read Replicas (up to 15)
├── Storage Layer
│   ├── Distributed Storage (6 copies across 3 AZs)
│   ├── Auto-scaling (10GB to 128TB)
│   └── Continuous Backup to S3
└── Features
    ├── Aurora Serverless
    ├── Global Database
    ├── Parallel Query
    └── Backtrack
```

### Aurora vs Standard RDS:

| Feature | Aurora | Standard RDS |
|---------|--------|--------------|
| **Performance** | 5x MySQL, 3x PostgreSQL | Standard |
| **Storage** | Auto-scaling up to 128TB | Manual scaling up to 64TB |
| **Replication** | 6 copies across 3 AZs | Single or Multi-AZ |
| **Read Replicas** | Up to 15 | Up to 5 |
| **Failover** | < 30 seconds | 1-2 minutes |
| **Backtrack** | Yes (point-in-time) | No |
| **Serverless** | Yes | No |
| **Global Database** | Yes (cross-region) | Read Replicas only |

### Complete Aurora Implementation:

```javascript
// aurora-connection.js
import { createPool } from 'mysql2/promise';
import { Signer } from '@aws-sdk/rds-signer';

class AuroraConnectionPool {
  constructor(config) {
    this.writerEndpoint = config.writerEndpoint;
    this.readerEndpoint = config.readerEndpoint;
    this.database = config.database;
    this.region = config.region;
    this.username = config.username;
    
    // Connection pools
    this.writerPool = null;
    this.readerPool = null;
  }
  
  // Generate IAM auth token
  async generateAuthToken(hostname) {
    const signer = new Signer({
      region: this.region,
      hostname: hostname,
      port: 3306,
      username: this.username
    });
    
    return await signer.getAuthToken();
  }
  
  // Initialize writer pool
  async initWriterPool() {
    const token = await this.generateAuthToken(this.writerEndpoint);
    
    this.writerPool = createPool({
      host: this.writerEndpoint,
      user: this.username,
      password: token,
      database: this.database,
      ssl: { rejectUnauthorized: true },
      waitForConnections: true,
      connectionLimit: 10,
      queueLimit: 0,
      enableKeepAlive: true,
      keepAliveInitialDelay: 0
    });
    
    console.log('Writer pool initialized');
  }
  
  // Initialize reader pool
  async initReaderPool() {
    const token = await this.generateAuthToken(this.readerEndpoint);
    
    this.readerPool = createPool({
      host: this.readerEndpoint,
      user: this.username,
      password: token,
      database: this.database,
      ssl: { rejectUnauthorized: true },
      waitForConnections: true,
      connectionLimit: 20,
      queueLimit: 0,
      enableKeepAlive: true,
      keepAliveInitialDelay: 0
    });
    
    console.log('Reader pool initialized');
  }
  
  // Execute write query
  async executeWrite(query, params) {
    if (!this.writerPool) {
      await this.initWriterPool();
    }
    
    try {
      const [results] = await this.writerPool.execute(query, params);
      return results;
    } catch (error) {
      console.error('Write query failed:', error);
      throw error;
    }
  }
  
  // Execute read query
  async executeRead(query, params) {
    if (!this.readerPool) {
      await this.readerPool();
    }
    
    try {
      const [results] = await this.readerPool.execute(query, params);
      return results;
    } catch (error) {
      console.error('Read query failed:', error);
      throw error;
    }
  }
  
  // Transaction support
  async executeTransaction(queries) {
    if (!this.writerPool) {
      await this.initWriterPool();
    }
    
    const connection = await this.writerPool.getConnection();
    
    try {
      await connection.beginTransaction();
      
      const results = [];
      for (const { query, params } of queries) {
        const [result] = await connection.execute(query, params);
        results.push(result);
      }
      
      await connection.commit();
      return results;
      
    } catch (error) {
      await connection.rollback();
      console.error('Transaction failed:', error);
      throw error;
    } finally {
      connection.release();
    }
  }
  
  // Close pools
  async close() {
    if (this.writerPool) {
      await this.writerPool.end();
    }
    if (this.readerPool) {
      await this.readerPool.end();
    }
  }
}

// Usage
const aurora = new AuroraConnectionPool({
  writerEndpoint: 'my-cluster.cluster-xyz.us-east-1.rds.amazonaws.com',
  readerEndpoint: 'my-cluster.cluster-ro-xyz.us-east-1.rds.amazonaws.com',
  database: 'myapp',
  region: 'us-east-1',
  username: 'admin'
});

// Write operation
await aurora.executeWrite(
  'INSERT INTO users (name, email) VALUES (?, ?)',
  ['John Doe', 'john@example.com']
);

// Read operation
const users = await aurora.executeRead(
  'SELECT * FROM users WHERE email = ?',
  ['john@example.com']
);

// Transaction
await aurora.executeTransaction([
  {
    query: 'INSERT INTO orders (user_id, total) VALUES (?, ?)',
    params: [1, 99.99]
  },
  {
    query: 'UPDATE users SET order_count = order_count + 1 WHERE id = ?',
    params: [1]
  }
]);
```

### Aurora Serverless v2:

```javascript
// aurora-serverless.js
import { RDSDataClient, ExecuteStatementCommand, BatchExecuteStatementCommand } from '@aws-sdk/client-rds-data';

class AuroraServerless {
  constructor(resourceArn, secretArn, database) {
    this.client = new RDSDataClient({ region: 'us-east-1' });
    this.resourceArn = resourceArn;
    this.secretArn = secretArn;
    this.database = database;
  }
  
  // Execute SQL statement
  async executeStatement(sql, parameters = []) {
    const command = new ExecuteStatementCommand({
      resourceArn: this.resourceArn,
      secretArn: this.secretArn,
      database: this.database,
      sql: sql,
      parameters: parameters.map(param => ({
        name: param.name,
        value: { stringValue: String(param.value) }
      }))
    });
    
    try {
      const response = await this.client.send(command);
      return this.formatRecords(response.records, response.columnMetadata);
    } catch (error) {
      console.error('Query failed:', error);
      throw error;
    }
  }
  
  // Batch execute
  async batchExecute(sql, parameterSets) {
    const command = new BatchExecuteStatementCommand({
      resourceArn: this.resourceArn,
      secretArn: this.secretArn,
      database: this.database,
      sql: sql,
      parameterSets: parameterSets.map(params =>
        params.map(param => ({
          name: param.name,
          value: { stringValue: String(param.value) }
        }))
      )
    });
    
    const response = await this.client.send(command);
    return response.updateResults;
  }
  
  // Format records
  formatRecords(records, columnMetadata) {
    if (!records) return [];
    
    return records.map(record => {
      const row = {};
      record.forEach((field, index) => {
        const columnName = columnMetadata[index].name;
        row[columnName] = Object.values(field)[0];
      });
      return row;
    });
  }
  
  // CRUD operations
  async createUser(name, email) {
    const sql = 'INSERT INTO users (name, email, created_at) VALUES (:name, :email, NOW())';
    await this.executeStatement(sql, [
      { name: 'name', value: name },
      { name: 'email', value: email }
    ]);
  }
  
  async getUserByEmail(email) {
    const sql = 'SELECT * FROM users WHERE email = :email';
    const results = await this.executeStatement(sql, [
      { name: 'email', value: email }
    ]);
    return results[0];
  }
  
  async updateUser(id, data) {
    const sql = 'UPDATE users SET name = :name, email = :email WHERE id = :id';
    await this.executeStatement(sql, [
      { name: 'id', value: id },
      { name: 'name', value: data.name },
      { name: 'email', value: data.email }
    ]);
  }
  
  async deleteUser(id) {
    const sql = 'DELETE FROM users WHERE id = :id';
    await this.executeStatement(sql, [
      { name: 'id', value: id }
    ]);
  }
  
  // Bulk insert
  async bulkInsertUsers(users) {
    const sql = 'INSERT INTO users (name, email) VALUES (:name, :email)';
    const parameterSets = users.map(user => [
      { name: 'name', value: user.name },
      { name: 'email', value: user.email }
    ]);
    
    return await this.batchExecute(sql, parameterSets);
  }
}

// Usage
const db = new AuroraServerless(
  'arn:aws:rds:us-east-1:123456789012:cluster:my-aurora-cluster',
  'arn:aws:secretsmanager:us-east-1:123456789012:secret:my-db-secret',
  'myapp'
);

await db.createUser('John Doe', 'john@example.com');
const user = await db.getUserByEmail('john@example.com');
```

### Best Practices:

1. ✅ Use reader endpoints for read queries
2. ✅ Enable Performance Insights
3. ✅ Configure automatic backups
4. ✅ Use IAM database authentication
5. ✅ Enable Enhanced Monitoring
6. ✅ Use connection pooling
7. ✅ Implement retry logic
8. ✅ Monitor with CloudWatch alarms
9. ✅ Use Aurora Global Database for DR
10. ✅ Enable encryption at rest

---

## Question 23: Explain Amazon Kinesis for real-time data streaming

### 📋 Answer

**Amazon Kinesis** is a platform for streaming data on AWS, making it easy to collect, process, and analyze real-time data.

### Kinesis Services:

```
Amazon Kinesis
├── Kinesis Data Streams
│   ├── Producers (Applications, SDK, KPL, Agent)
│   ├── Shards (1MB/s write, 2MB/s read)
│   └── Consumers (Applications, Lambda, Firehose)
├── Kinesis Data Firehose
│   ├── Data transformation (Lambda)
│   ├── Destinations (S3, Redshift, Elasticsearch, HTTP)
│   └── Automatic scaling
├── Kinesis Data Analytics
│   ├── SQL queries on streams
│   ├── Apache Flink applications
│   └── Real-time analytics
└── Kinesis Video Streams
    └── Video ingestion and processing
```

### Kinesis Data Streams Implementation:

```javascript
// kinesis-producer.js
import {
  KinesisClient,
  PutRecordCommand,
  PutRecordsCommand,
  CreateStreamCommand,
  DescribeStreamCommand
} from '@aws-sdk/client-kinesis';

const kinesisClient = new KinesisClient({ region: 'us-east-1' });

class KinesisProducer {
  constructor(streamName) {
    this.streamName = streamName;
    this.buffer = [];
    this.bufferSize = 500;  // Max 500 records per batch
    this.flushInterval = 1000;  // Flush every 1 second
    
    this.startAutoFlush();
  }
  
  // Put single record
  async putRecord(data, partitionKey) {
    const command = new PutRecordCommand({
      StreamName: this.streamName,
      Data: JSON.stringify(data),
      PartitionKey: partitionKey
    });
    
    try {
      const response = await kinesisClient.send(command);
      console.log('Record sent:', response.SequenceNumber);
      return response;
    } catch (error) {
      console.error('Failed to send record:', error);
      throw error;
    }
  }
  
  // Buffer records for batch sending
  addToBuffer(data, partitionKey) {
    this.buffer.push({
      Data: JSON.stringify(data),
      PartitionKey: partitionKey
    });
    
    if (this.buffer.length >= this.bufferSize) {
      return this.flush();
    }
  }
  
  // Flush buffer
  async flush() {
    if (this.buffer.length === 0) return;
    
    const records = this.buffer.splice(0, this.bufferSize);
    
    const command = new PutRecordsCommand({
      StreamName: this.streamName,
      Records: records
    });
    
    try {
      const response = await kinesisClient.send(command);
      
      console.log(`Sent ${records.length} records, Failed: ${response.FailedRecordCount || 0}`);
      
      // Retry failed records
      if (response.FailedRecordCount > 0) {
        const failedRecords = response.Records
          .map((record, index) => record.ErrorCode ? records[index] : null)
          .filter(Boolean);
        
        this.buffer.unshift(...failedRecords);
      }
      
      return response;
    } catch (error) {
      console.error('Failed to send batch:', error);
      this.buffer.unshift(...records);  // Re-add to buffer
      throw error;
    }
  }
  
  // Auto-flush timer
  startAutoFlush() {
    setInterval(async () => {
      if (this.buffer.length > 0) {
        await this.flush();
      }
    }, this.flushInterval);
  }
  
  // Calculate partition key for even distribution
  getPartitionKey(data) {
    // Use user ID, session ID, or other identifier
    return data.userId || Math.random().toString(36).substring(7);
  }
}

// Usage - Click stream data
const producer = new KinesisProducer('clickstream');

// Send page view event
await producer.addToBuffer({
  eventType: 'page_view',
  userId: 'user123',
  url: '/products/123',
  timestamp: Date.now(),
  userAgent: 'Mozilla/5.0...',
  referrer: 'https://google.com'
}, 'user123');

// Send purchase event
await producer.addToBuffer({
  eventType: 'purchase',
  userId: 'user123',
  orderId: 'order456',
  amount: 99.99,
  currency: 'USD',
  timestamp: Date.now()
}, 'user123');
```

### Kinesis Consumer:

```javascript
// kinesis-consumer.js
import {
  KinesisClient,
  GetShardIteratorCommand,
  GetRecordsCommand,
  DescribeStreamCommand
} from '@aws-sdk/client-kinesis';

const kinesisClient = new KinesisClient({ region: 'us-east-1' });

class KinesisConsumer {
  constructor(streamName, shardId) {
    this.streamName = streamName;
    this.shardId = shardId;
    this.running = false;
  }
  
  // Start consuming
  async start(processRecordCallback) {
    this.running = true;
    
    // Get shard iterator
    let shardIterator = await this.getShardIterator('LATEST');
    
    while (this.running) {
      try {
        const command = new GetRecordsCommand({
          ShardIterator: shardIterator,
          Limit: 100
        });
        
        const response = await kinesisClient.send(command);
        
        // Process records
        for (const record of response.Records) {
          const data = JSON.parse(record.Data.toString('utf-8'));
          await processRecordCallback(data, record);
        }
        
        // Update iterator
        shardIterator = response.NextShardIterator;
        
        // Sleep if no records
        if (response.Records.length === 0) {
          await this.sleep(1000);
        }
        
      } catch (error) {
        console.error('Error consuming records:', error);
        await this.sleep(5000);
        shardIterator = await this.getShardIterator('LATEST');
      }
    }
  }
  
  // Get shard iterator
  async getShardIterator(iteratorType = 'LATEST') {
    const command = new GetShardIteratorCommand({
      StreamName: this.streamName,
      ShardId: this.shardId,
      ShardIteratorType: iteratorType
    });
    
    const response = await kinesisClient.send(command);
    return response.ShardIterator;
  }
  
  stop() {
    this.running = false;
  }
  
  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Usage
const consumer = new KinesisConsumer('clickstream', 'shardId-000000000000');

await consumer.start(async (data, record) => {
  console.log('Processing event:', data.eventType);
  
  // Process different event types
  switch (data.eventType) {
    case 'page_view':
      await trackPageView(data);
      break;
    case 'purchase':
      await processPurchase(data);
      break;
    default:
      console.log('Unknown event type:', data.eventType);
  }
});
```

### Kinesis Data Firehose:

```javascript
// kinesis-firehose.js
import {
  FirehoseClient,
  PutRecordCommand,
  PutRecordBatchCommand,
  CreateDeliveryStreamCommand
} from '@aws-sdk/client-firehose';

const firehoseClient = new FirehoseClient({ region: 'us-east-1' });

class FirehoseProducer {
  constructor(deliveryStreamName) {
    this.deliveryStreamName = deliveryStreamName;
  }
  
  // Send single record
  async putRecord(data) {
    const command = new PutRecordCommand({
      DeliveryStreamName: this.deliveryStreamName,
      Record: {
        Data: JSON.stringify(data) + '\n'  // Add newline for S3
      }
    });
    
    const response = await firehoseClient.send(command);
    console.log('Record sent to Firehose:', response.RecordId);
    return response;
  }
  
  // Send batch
  async putRecordBatch(records) {
    const command = new PutRecordBatchCommand({
      DeliveryStreamName: this.deliveryStreamName,
      Records: records.map(record => ({
        Data: JSON.stringify(record) + '\n'
      }))
    });
    
    const response = await firehoseClient.send(command);
    console.log(`Sent ${records.length} records, Failed: ${response.FailedPutCount}`);
    return response;
  }
}

// Create Firehose delivery stream to S3
async function createFirehoseStream(streamName, s3Bucket, s3Prefix) {
  const command = new CreateDeliveryStreamCommand({
    DeliveryStreamName: streamName,
    DeliveryStreamType: 'DirectPut',
    S3DestinationConfiguration: {
      RoleARN: 'arn:aws:iam::123456789012:role/FirehoseRole',
      BucketARN: `arn:aws:s3:::${s3Bucket}`,
      Prefix: s3Prefix,
      BufferingHints: {
        SizeInMBs: 5,
        IntervalInSeconds: 300
      },
      CompressionFormat: 'GZIP',
      EncryptionConfiguration: {
        NoEncryptionConfig: 'NoEncryption'
      }
    }
  });
  
  const response = await firehoseClient.send(command);
  console.log('Firehose stream created:', response.DeliveryStreamARN);
}

// Usage - Log aggregation
const firehose = new FirehoseProducer('application-logs');

await firehose.putRecord({
  timestamp: Date.now(),
  level: 'ERROR',
  service: 'api-service',
  message: 'Database connection failed',
  stackTrace: '...',
  userId: 'user123'
});

// Batch send
const logs = [
  { timestamp: Date.now(), level: 'INFO', message: 'Request received' },
  { timestamp: Date.now(), level: 'INFO', message: 'Request processed' },
  { timestamp: Date.now(), level: 'ERROR', message: 'Error occurred' }
];

await firehose.putRecordBatch(logs);
```

### Lambda Consumer (Kinesis Data Streams):

```javascript
// lambda-kinesis-consumer.js
export const handler = async (event) => {
  console.log(`Processing ${event.Records.length} records`);
  
  const results = await Promise.allSettled(
    event.Records.map(async (record) => {
      try {
        // Decode data
        const data = JSON.parse(
          Buffer.from(record.kinesis.data, 'base64').toString('utf-8')
        );
        
        console.log('Processing record:', data);
        
        // Process based on event type
        switch (data.eventType) {
          case 'user_signup':
            await sendWelcomeEmail(data);
            break;
          case 'purchase':
            await updateInventory(data);
            await createInvoice(data);
            break;
          default:
            console.log('Unknown event type:', data.eventType);
        }
        
        return { success: true, sequenceNumber: record.kinesis.sequenceNumber };
        
      } catch (error) {
        console.error('Error processing record:', error);
        return { success: false, error: error.message };
      }
    })
  );
  
  // Report failures
  const failures = results.filter(r => r.status === 'rejected' || !r.value.success);
  
  if (failures.length > 0) {
    console.error(`${failures.length} records failed processing`);
    // Return failed record IDs for retry
    throw new Error('Batch processing failed');
  }
  
  return {
    statusCode: 200,
    body: JSON.stringify({ processed: results.length })
  };
};
```

### Best Practices:

1. ✅ Use partition keys for even distribution
2. ✅ Implement batch sending for efficiency
3. ✅ Handle throttling with exponential backoff
4. ✅ Monitor shard utilization
5. ✅ Use enhanced fan-out for consumers
6. ✅ Enable server-side encryption
7. ✅ Set appropriate retention period
8. ✅ Use CloudWatch for monitoring
9. ✅ Implement error handling and retries
10. ✅ Consider Kinesis Data Firehose for S3/Redshift

---

## Question 24: Explain AWS Glue for ETL operations

### 📋 Answer

**AWS Glue** is a fully managed ETL (Extract, Transform, Load) service that makes it easy to prepare and load data for analytics.

### Glue Components:

```
AWS Glue
├── Data Catalog
│   ├── Databases
│   ├── Tables (Metadata)
│   └── Partitions
├── Crawlers
│   ├── Data Discovery
│   └── Schema Inference
├── ETL Jobs
│   ├── Spark (Python/Scala)
│   └── Python Shell
└── Triggers
    ├── Scheduled
    ├── On-demand
    └── Conditional
```

### Glue Data Catalog:

```javascript
// glue-catalog.js
import {
  GlueClient,
  CreateDatabaseCommand,
  CreateTableCommand,
  GetTableCommand,
  GetTablesCommand,
  StartCrawlerCommand,
  GetCrawlerCommand
} from '@aws-sdk/client-glue';

const glueClient = new GlueClient({ region: 'us-east-1' });

// Create database
async function createDatabase(databaseName, description) {
  const command = new CreateDatabaseCommand({
    DatabaseInput: {
      Name: databaseName,
      Description: description
    }
  });
  
  await glueClient.send(command);
  console.log('Database created:', databaseName);
}

// Create table
async function createTable(databaseName, tableName, s3Location, columns) {
  const command = new CreateTableCommand({
    DatabaseName: databaseName,
    TableInput: {
      Name: tableName,
      StorageDescriptor: {
        Columns: columns,
        Location: s3Location,
        InputFormat: 'org.apache.hadoop.mapred.TextInputFormat',
        OutputFormat: 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat',
        SerdeInfo: {
          SerializationLibrary: 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe',
          Parameters: {
            'field.delim': ','
          }
        }
      },
      PartitionKeys: [
        { Name: 'year', Type: 'int' },
        { Name: 'month', Type: 'int' },
        { Name: 'day', Type: 'int' }
      ]
    }
  });
  
  await glueClient.send(command);
  console.log('Table created:', tableName);
}

// Usage
await createDatabase('sales_data', 'Sales transaction data');

await createTable(
  'sales_data',
  'transactions',
  's3://my-bucket/data/transactions/',
  [
    { Name: 'transaction_id', Type: 'string' },
    { Name: 'user_id', Type: 'string' },
    { Name: 'amount', Type: 'double' },
    { Name: 'currency', Type: 'string' },
    { Name: 'timestamp', Type: 'timestamp' }
  ]
);
```

### Glue ETL Job (PySpark):

```python
# glue-etl-job.py
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.dynamicframe import DynamicFrame

# Get job parameters
args = getResolvedOptions(sys.argv, ['JOB_NAME', 'SOURCE_BUCKET', 'TARGET_BUCKET'])

# Initialize contexts
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# Read data from S3
source_dynamic_frame = glueContext.create_dynamic_frame.from_catalog(
    database = "sales_data",
    table_name = "transactions",
    transformation_ctx = "source_dynamic_frame"
)

# Convert to DataFrame for complex transformations
df = source_dynamic_frame.toDF()

# Data transformations
from pyspark.sql.functions import col, year, month, dayofmonth, sum as _sum

# Filter invalid records
df_filtered = df.filter(col("amount") > 0)

# Add date columns
df_with_dates = df_filtered.withColumn("year", year(col("timestamp"))) \
                            .withColumn("month", month(col("timestamp"))) \
                            .withColumn("day", dayofmonth(col("timestamp")))

# Aggregate daily sales
df_aggregated = df_with_dates.groupBy("year", "month", "day", "currency") \
                              .agg(_sum("amount").alias("total_sales"))

# Convert back to DynamicFrame
result_dynamic_frame = DynamicFrame.fromDF(df_aggregated, glueContext, "result_dynamic_frame")

# Write to S3 (partitioned by year/month/day)
glueContext.write_dynamic_frame.from_options(
    frame = result_dynamic_frame,
    connection_type = "s3",
    connection_options = {
        "path": f"s3://{args['TARGET_BUCKET']}/aggregated-sales/",
        "partitionKeys": ["year", "month", "day"]
    },
    format = "parquet",
    transformation_ctx = "write_to_s3"
)

job.commit()
```

### Manage Glue Jobs:

```javascript
// glue-job-manager.js
import {
  GlueClient,
  CreateJobCommand,
  StartJobRunCommand,
  GetJobRunCommand,
  GetJobRunsCommand
} from '@aws-sdk/client-glue';

const glueClient = new GlueClient({ region: 'us-east-1' });

// Create Glue job
async function createGlueJob(jobName, scriptLocation, roleArn) {
  const command = new CreateJobCommand({
    Name: jobName,
    Role: roleArn,
    Command: {
      Name: 'glueetl',
      ScriptLocation: scriptLocation,
      PythonVersion: '3'
    },
    DefaultArguments: {
      '--job-language': 'python',
      '--job-bookmark-option': 'job-bookmark-enable',
      '--enable-metrics': 'true',
      '--enable-spark-ui': 'true',
      '--spark-event-logs-path': 's3://my-bucket/glue-logs/'
    },
    MaxRetries: 1,
    Timeout: 2880,  // 48 hours
    GlueVersion: '4.0',
    NumberOfWorkers: 10,
    WorkerType: 'G.1X'  // G.1X, G.2X, G.025X
  });
  
  const response = await glueClient.send(command);
  console.log('Glue job created:', response.Name);
  return response;
}

// Start job run
async function startJobRun(jobName, arguments) {
  const command = new StartJobRunCommand({
    JobName: jobName,
    Arguments: arguments
  });
  
  const response = await glueClient.send(command);
  console.log('Job run started:', response.JobRunId);
  return response.JobRunId;
}

// Get job run status
async function getJobRunStatus(jobName, jobRunId) {
  const command = new GetJobRunCommand({
    JobName: jobName,
    RunId: jobRunId
  });
  
  const response = await glueClient.send(command);
  
  return {
    status: response.JobRun.JobRunState,
    startedOn: response.JobRun.StartedOn,
    completedOn: response.JobRun.CompletedOn,
    executionTime: response.JobRun.ExecutionTime,
    errorMessage: response.JobRun.ErrorMessage
  };
}

// Wait for job completion
async function waitForJobCompletion(jobName, jobRunId, pollInterval = 30000) {
  while (true) {
    const status = await getJobRunStatus(jobName, jobRunId);
    
    console.log('Job status:', status.status);
    
    if (['SUCCEEDED', 'FAILED', 'STOPPED', 'TIMEOUT'].includes(status.status)) {
      return status;
    }
    
    await new Promise(resolve => setTimeout(resolve, pollInterval));
  }
}

// Usage
const jobName = 'sales-aggregation-job';

await createGlueJob(
  jobName,
  's3://my-bucket/glue-scripts/aggregate-sales.py',
  'arn:aws:iam::123456789012:role/GlueServiceRole'
);

const jobRunId = await startJobRun(jobName, {
  '--SOURCE_BUCKET': 'my-source-bucket',
  '--TARGET_BUCKET': 'my-target-bucket'
});

const result = await waitForJobCompletion(jobName, jobRunId);
console.log('Job completed:', result);
```

### Best Practices:

1. ✅ Use Glue Data Catalog as central metadata repository
2. ✅ Partition data for better query performance
3. ✅ Enable job bookmarks to process incremental data
4. ✅ Use appropriate worker types for workload
5. ✅ Monitor job metrics with CloudWatch
6. ✅ Use Glue DataBrew for no-code transformations
7. ✅ Implement error handling and retries
8. ✅ Use development endpoints for testing
9. ✅ Enable Spark UI for debugging
10. ✅ Use Glue Triggers for workflow orchestration

---

## Key Takeaways

### Question 22 (Aurora):
- MySQL/PostgreSQL compatible with 5x/3x performance
- Auto-scaling storage (10GB to 128TB)
- Up to 15 read replicas across 3 AZs
- Aurora Serverless for variable workloads
- Global Database for disaster recovery
- < 30 second failover time

### Question 23 (Kinesis):
- Real-time data streaming platform
- Data Streams for custom processing
- Data Firehose for direct delivery to S3/Redshift
- Partition keys for load distribution
- Lambda integration for serverless processing
- Automatic scaling with Firehose

### Question 24 (Glue):
- Fully managed ETL service
- Data Catalog for metadata management
- Crawlers for automatic schema discovery
- Spark-based job execution
- Job bookmarks for incremental processing
- Integration with Athena, Redshift, EMR
