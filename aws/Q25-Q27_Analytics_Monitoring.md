# AWS Interview Questions: Analytics & Monitoring (Q25-Q27)

## Question 25: Explain Amazon Athena for serverless SQL queries

### 📋 Answer

**Amazon Athena** is an interactive query service that makes it easy to analyze data in Amazon S3 using standard SQL. It's serverless, so there's no infrastructure to manage.

### Athena Architecture:

```
Amazon Athena
├── Data Sources
│   ├── S3 (primary)
│   ├── DynamoDB
│   ├── CloudWatch Logs
│   └── External (via connectors)
├── Query Engine
│   ├── Presto-based
│   ├── ANSI SQL support
│   └── Federated queries
├── Data Formats
│   ├── CSV, JSON, Parquet
│   ├── ORC, Avro
│   └── Compressed files
└── Features
    ├── CTAS (Create Table As Select)
    ├── Partitions
    ├── Views
    └── Workgroups
```

### Complete Athena Implementation:

```javascript
// athena-client.js
import {
  AthenaClient,
  StartQueryExecutionCommand,
  GetQueryExecutionCommand,
  GetQueryResultsCommand,
  CreateNamedQueryCommand,
  ListQueryExecutionsCommand
} from '@aws-sdk/client-athena';

const athenaClient = new AthenaClient({ region: 'us-east-1' });

class AthenaQueryService {
  constructor(database, outputLocation, workgroup = 'primary') {
    this.database = database;
    this.outputLocation = outputLocation;
    this.workgroup = workgroup;
  }
  
  // Execute query
  async executeQuery(queryString) {
    const command = new StartQueryExecutionCommand({
      QueryString: queryString,
      QueryExecutionContext: {
        Database: this.database
      },
      ResultConfiguration: {
        OutputLocation: this.outputLocation,
        EncryptionConfiguration: {
          EncryptionOption: 'SSE_S3'
        }
      },
      WorkGroup: this.workgroup
    });
    
    try {
      const response = await athenaClient.send(command);
      console.log('Query execution started:', response.QueryExecutionId);
      return response.QueryExecutionId;
    } catch (error) {
      console.error('Query execution failed:', error);
      throw error;
    }
  }
  
  // Get query status
  async getQueryStatus(queryExecutionId) {
    const command = new GetQueryExecutionCommand({
      QueryExecutionId: queryExecutionId
    });
    
    const response = await athenaClient.send(command);
    
    return {
      state: response.QueryExecution.Status.State,
      stateChangeReason: response.QueryExecution.Status.StateChangeReason,
      submissionDateTime: response.QueryExecution.Status.SubmissionDateTime,
      completionDateTime: response.QueryExecution.Status.CompletionDateTime,
      dataScannedInBytes: response.QueryExecution.Statistics.DataScannedInBytes,
      executionTimeInMillis: response.QueryExecution.Statistics.EngineExecutionTimeInMillis
    };
  }
  
  // Wait for query completion
  async waitForQueryCompletion(queryExecutionId, pollInterval = 2000) {
    while (true) {
      const status = await this.getQueryStatus(queryExecutionId);
      
      console.log('Query status:', status.state);
      
      if (status.state === 'SUCCEEDED') {
        return status;
      } else if (['FAILED', 'CANCELLED'].includes(status.state)) {
        throw new Error(`Query ${status.state}: ${status.stateChangeReason}`);
      }
      
      await new Promise(resolve => setTimeout(resolve, pollInterval));
    }
  }
  
  // Get query results
  async getQueryResults(queryExecutionId, maxResults = 1000) {
    const command = new GetQueryResultsCommand({
      QueryExecutionId: queryExecutionId,
      MaxResults: maxResults
    });
    
    const response = await athenaClient.send(command);
    
    // Parse results
    const columnInfo = response.ResultSet.ResultSetMetadata.ColumnInfo;
    const rows = response.ResultSet.Rows;
    
    // Skip header row
    const dataRows = rows.slice(1);
    
    return dataRows.map(row => {
      const obj = {};
      row.Data.forEach((field, index) => {
        const columnName = columnInfo[index].Name;
        obj[columnName] = field.VarCharValue;
      });
      return obj;
    });
  }
  
  // Execute query and get results
  async query(queryString) {
    const queryExecutionId = await this.executeQuery(queryString);
    await this.waitForQueryCompletion(queryExecutionId);
    return await this.getQueryResults(queryExecutionId);
  }
  
  // Create table
  async createTable(tableName, s3Location, columns, partitionColumns = []) {
    const columnDefs = columns.map(col => `${col.name} ${col.type}`).join(',\n  ');
    const partitionDefs = partitionColumns.length > 0
      ? `PARTITIONED BY (${partitionColumns.map(col => `${col.name} ${col.type}`).join(', ')})`
      : '';
    
    const query = `
      CREATE EXTERNAL TABLE IF NOT EXISTS ${tableName} (
        ${columnDefs}
      )
      ${partitionDefs}
      ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
      WITH SERDEPROPERTIES (
        'serialization.format' = ',',
        'field.delim' = ','
      )
      LOCATION '${s3Location}'
      TBLPROPERTIES ('has_encrypted_data'='false')
    `;
    
    return await this.executeQuery(query);
  }
  
  // Create partitioned table (Parquet)
  async createParquetTable(tableName, s3Location, columns, partitionColumns) {
    const columnDefs = columns.map(col => `${col.name} ${col.type}`).join(',\n  ');
    const partitionDefs = partitionColumns.map(col => `${col.name} ${col.type}`).join(', ');
    
    const query = `
      CREATE EXTERNAL TABLE IF NOT EXISTS ${tableName} (
        ${columnDefs}
      )
      PARTITIONED BY (${partitionDefs})
      STORED AS PARQUET
      LOCATION '${s3Location}'
      TBLPROPERTIES (
        'parquet.compression'='SNAPPY',
        'projection.enabled' = 'true',
        'projection.year.type' = 'integer',
        'projection.year.range' = '2020,2030',
        'projection.month.type' = 'integer',
        'projection.month.range' = '1,12',
        'projection.day.type' = 'integer',
        'projection.day.range' = '1,31'
      )
    `;
    
    return await this.executeQuery(query);
  }
  
  // Add partition
  async addPartition(tableName, partitionValues, s3Location) {
    const partitionSpec = Object.entries(partitionValues)
      .map(([key, value]) => `${key}='${value}'`)
      .join(', ');
    
    const query = `
      ALTER TABLE ${tableName}
      ADD IF NOT EXISTS PARTITION (${partitionSpec})
      LOCATION '${s3Location}'
    `;
    
    return await this.executeQuery(query);
  }
  
  // CTAS (Create Table As Select)
  async createTableAsSelect(newTableName, selectQuery, s3Location) {
    const query = `
      CREATE TABLE ${newTableName}
      WITH (
        external_location = '${s3Location}',
        format = 'PARQUET',
        parquet_compression = 'SNAPPY'
      )
      AS ${selectQuery}
    `;
    
    return await this.executeQuery(query);
  }
  
  // Save named query
  async createNamedQuery(name, description, queryString) {
    const command = new CreateNamedQueryCommand({
      Name: name,
      Description: description,
      Database: this.database,
      QueryString: queryString
    });
    
    const response = await athenaClient.send(command);
    console.log('Named query created:', response.NamedQueryId);
    return response;
  }
}

// Usage
const athena = new AthenaQueryService(
  'analytics_db',
  's3://my-bucket/athena-results/',
  'primary'
);

// Create table
await athena.createParquetTable(
  'sales_transactions',
  's3://my-bucket/data/sales/',
  [
    { name: 'transaction_id', type: 'string' },
    { name: 'user_id', type: 'string' },
    { name: 'amount', type: 'decimal(10,2)' },
    { name: 'currency', type: 'string' },
    { name: 'timestamp', type: 'timestamp' }
  ],
  [
    { name: 'year', type: 'int' },
    { name: 'month', type: 'int' },
    { name: 'day', type: 'int' }
  ]
);

// Query data
const results = await athena.query(`
  SELECT 
    DATE(timestamp) as date,
    COUNT(*) as transaction_count,
    SUM(amount) as total_amount
  FROM sales_transactions
  WHERE year = 2024 AND month = 1
  GROUP BY DATE(timestamp)
  ORDER BY date DESC
  LIMIT 100
`);

console.log('Query results:', results);

// Create aggregated table
await athena.createTableAsSelect(
  'daily_sales_summary',
  `
    SELECT 
      DATE(timestamp) as date,
      currency,
      COUNT(*) as transaction_count,
      SUM(amount) as total_amount,
      AVG(amount) as avg_amount
    FROM sales_transactions
    WHERE year = 2024 AND month = 1
    GROUP BY DATE(timestamp), currency
  `,
  's3://my-bucket/aggregated/daily_sales/'
);
```

### Query Optimization:

```javascript
// athena-optimization.js
class AthenaOptimization {
  // Partition pruning
  static getOptimizedQuery(startDate, endDate) {
    return `
      SELECT *
      FROM sales_transactions
      WHERE year = 2024 
        AND month = 1 
        AND day BETWEEN 1 AND 31
        AND timestamp BETWEEN '${startDate}' AND '${endDate}'
    `;
  }
  
  // Use columnar format (Parquet)
  static async convertToParquet(athena, sourceTable, targetTable, targetLocation) {
    const query = `
      CREATE TABLE ${targetTable}
      WITH (
        external_location = '${targetLocation}',
        format = 'PARQUET',
        parquet_compression = 'SNAPPY',
        partitioned_by = ARRAY['year', 'month', 'day']
      )
      AS SELECT * FROM ${sourceTable}
    `;
    
    return await athena.query(query);
  }
  
  // Use approximation functions
  static getApproximateQuery() {
    return `
      SELECT 
        approx_distinct(user_id) as unique_users,
        approx_percentile(amount, 0.5) as median_amount,
        approx_percentile(amount, 0.95) as p95_amount
      FROM sales_transactions
      WHERE year = 2024 AND month = 1
    `;
  }
  
  // Use EXPLAIN to analyze query plan
  static async explainQuery(athena, query) {
    const explainQuery = `EXPLAIN ${query}`;
    return await athena.query(explainQuery);
  }
}
```

### Best Practices:

1. ✅ Partition data by date for faster queries
2. ✅ Use columnar formats (Parquet, ORC)
3. ✅ Compress data (GZIP, Snappy)
4. ✅ Use partition projection for automatic partitions
5. ✅ Limit result size with WHERE clauses
6. ✅ Use CTAS for intermediate results
7. ✅ Enable query result caching
8. ✅ Use workgroups for cost control
9. ✅ Convert JSON to Parquet for better performance
10. ✅ Monitor query costs with CloudWatch

---

## Question 26: Explain Amazon CloudWatch for monitoring and observability

### 📋 Answer

**Amazon CloudWatch** provides monitoring and observability for AWS resources and applications through metrics, logs, and alarms.

### CloudWatch Components:

```
Amazon CloudWatch
├── Metrics
│   ├── AWS Service Metrics
│   ├── Custom Metrics
│   └── Metric Math
├── Logs
│   ├── Log Groups
│   ├── Log Streams
│   ├── Insights Queries
│   └── Subscriptions
├── Alarms
│   ├── Metric Alarms
│   ├── Composite Alarms
│   └── Actions (SNS, Auto Scaling, Lambda)
├── Dashboards
│   └── Custom Visualizations
└── Events/EventBridge
    └── Event-driven automation
```

### Complete CloudWatch Implementation:

```javascript
// cloudwatch-service.js
import {
  CloudWatchClient,
  PutMetricDataCommand,
  GetMetricStatisticsCommand,
  PutMetricAlarmCommand,
  DescribeAlarmsCommand
} from '@aws-sdk/client-cloudwatch';

import {
  CloudWatchLogsClient,
  CreateLogGroupCommand,
  CreateLogStreamCommand,
  PutLogEventsCommand,
  FilterLogEventsCommand,
  StartQueryCommand,
  GetQueryResultsCommand
} from '@aws-sdk/client-cloudwatch-logs';

const cwClient = new CloudWatchClient({ region: 'us-east-1' });
const cwLogsClient = new CloudWatchLogsClient({ region: 'us-east-1' });

class CloudWatchService {
  constructor(namespace) {
    this.namespace = namespace;
  }
  
  // 1. Put Custom Metrics
  async putMetric(metricName, value, unit, dimensions = {}) {
    const command = new PutMetricDataCommand({
      Namespace: this.namespace,
      MetricData: [
        {
          MetricName: metricName,
          Value: value,
          Unit: unit,  // Seconds, Microseconds, Milliseconds, Bytes, Kilobytes, etc.
          Timestamp: new Date(),
          Dimensions: Object.entries(dimensions).map(([name, value]) => ({
            Name: name,
            Value: value
          }))
        }
      ]
    });
    
    try {
      await cwClient.send(command);
      console.log(`Metric published: ${metricName} = ${value}`);
    } catch (error) {
      console.error('Failed to put metric:', error);
      throw error;
    }
  }
  
  // 2. Batch Put Metrics
  async putMetrics(metrics) {
    const metricData = metrics.map(metric => ({
      MetricName: metric.name,
      Value: metric.value,
      Unit: metric.unit,
      Timestamp: new Date(),
      Dimensions: Object.entries(metric.dimensions || {}).map(([name, value]) => ({
        Name: name,
        Value: value
      }))
    }));
    
    const command = new PutMetricDataCommand({
      Namespace: this.namespace,
      MetricData: metricData
    });
    
    await cwClient.send(command);
    console.log(`${metrics.length} metrics published`);
  }
  
  // 3. Get Metric Statistics
  async getMetricStatistics(metricName, startTime, endTime, period = 60, statistic = 'Average') {
    const command = new GetMetricStatisticsCommand({
      Namespace: this.namespace,
      MetricName: metricName,
      StartTime: startTime,
      EndTime: endTime,
      Period: period,  // in seconds
      Statistics: [statistic],  // Average, Sum, Minimum, Maximum, SampleCount
      Dimensions: []
    });
    
    const response = await cwClient.send(command);
    return response.Datapoints.sort((a, b) => a.Timestamp - b.Timestamp);
  }
  
  // 4. Create Metric Alarm
  async createAlarm(alarmName, metricName, threshold, comparisonOperator, evaluationPeriods, snsTopicArn) {
    const command = new PutMetricAlarmCommand({
      AlarmName: alarmName,
      AlarmDescription: `Alarm for ${metricName}`,
      ActionsEnabled: true,
      AlarmActions: [snsTopicArn],
      MetricName: metricName,
      Namespace: this.namespace,
      Statistic: 'Average',
      Period: 300,  // 5 minutes
      EvaluationPeriods: evaluationPeriods,
      Threshold: threshold,
      ComparisonOperator: comparisonOperator,  // GreaterThanThreshold, LessThanThreshold, etc.
      TreatMissingData: 'notBreaching'
    });
    
    await cwClient.send(command);
    console.log('Alarm created:', alarmName);
  }
  
  // 5. Create Composite Alarm
  async createCompositeAlarm(alarmName, alarmRule, snsTopicArn) {
    const command = new PutMetricAlarmCommand({
      AlarmName: alarmName,
      AlarmDescription: 'Composite alarm for multiple conditions',
      ActionsEnabled: true,
      AlarmActions: [snsTopicArn],
      AlarmRule: alarmRule  // e.g., "ALARM(alarm1) OR ALARM(alarm2)"
    });
    
    await cwClient.send(command);
    console.log('Composite alarm created:', alarmName);
  }
}

// Application Monitoring Class
class ApplicationMonitoring {
  constructor(appName) {
    this.cloudwatch = new CloudWatchService('CustomApp');
    this.appName = appName;
  }
  
  // Track API request
  async trackRequest(endpoint, method, statusCode, duration) {
    await this.cloudwatch.putMetrics([
      {
        name: 'RequestCount',
        value: 1,
        unit: 'Count',
        dimensions: {
          Application: this.appName,
          Endpoint: endpoint,
          Method: method,
          StatusCode: statusCode
        }
      },
      {
        name: 'ResponseTime',
        value: duration,
        unit: 'Milliseconds',
        dimensions: {
          Application: this.appName,
          Endpoint: endpoint
        }
      }
    ]);
  }
  
  // Track errors
  async trackError(errorType, endpoint) {
    await this.cloudwatch.putMetric(
      'ErrorCount',
      1,
      'Count',
      {
        Application: this.appName,
        ErrorType: errorType,
        Endpoint: endpoint
      }
    );
  }
  
  // Track business metrics
  async trackBusinessMetric(metricName, value) {
    await this.cloudwatch.putMetric(
      metricName,
      value,
      'Count',
      { Application: this.appName }
    );
  }
}

// Usage
const monitoring = new ApplicationMonitoring('MyWebApp');

// Track API request
await monitoring.trackRequest('/api/users', 'GET', '200', 45);

// Track error
await monitoring.trackError('DatabaseConnectionError', '/api/users');

// Track business metrics
await monitoring.trackBusinessMetric('UserSignups', 1);
await monitoring.trackBusinessMetric('OrdersPlaced', 1);
await monitoring.trackBusinessMetric('Revenue', 99.99);
```

### CloudWatch Logs:

```javascript
// cloudwatch-logs.js
class CloudWatchLogs {
  constructor(logGroupName) {
    this.logGroupName = logGroupName;
    this.logStreamName = null;
    this.sequenceToken = null;
  }
  
  // Initialize log group and stream
  async initialize() {
    // Create log group
    try {
      await cwLogsClient.send(new CreateLogGroupCommand({
        logGroupName: this.logGroupName
      }));
      console.log('Log group created:', this.logGroupName);
    } catch (error) {
      if (error.name !== 'ResourceAlreadyExistsException') {
        throw error;
      }
    }
    
    // Create log stream
    this.logStreamName = `stream-${Date.now()}`;
    try {
      await cwLogsClient.send(new CreateLogStreamCommand({
        logGroupName: this.logGroupName,
        logStreamName: this.logStreamName
      }));
      console.log('Log stream created:', this.logStreamName);
    } catch (error) {
      if (error.name !== 'ResourceAlreadyExistsException') {
        throw error;
      }
    }
  }
  
  // Put log events
  async putLogs(messages) {
    if (!this.logStreamName) {
      await this.initialize();
    }
    
    const logEvents = messages.map(msg => ({
      message: typeof msg === 'string' ? msg : JSON.stringify(msg),
      timestamp: Date.now()
    }));
    
    const command = new PutLogEventsCommand({
      logGroupName: this.logGroupName,
      logStreamName: this.logStreamName,
      logEvents: logEvents,
      sequenceToken: this.sequenceToken
    });
    
    try {
      const response = await cwLogsClient.send(command);
      this.sequenceToken = response.nextSequenceToken;
    } catch (error) {
      console.error('Failed to put logs:', error);
      throw error;
    }
  }
  
  // Filter log events
  async filterLogs(filterPattern, startTime, endTime) {
    const command = new FilterLogEventsCommand({
      logGroupName: this.logGroupName,
      filterPattern: filterPattern,
      startTime: startTime,
      endTime: endTime,
      limit: 100
    });
    
    const response = await cwLogsClient.send(command);
    return response.events.map(event => ({
      timestamp: event.timestamp,
      message: event.message,
      logStreamName: event.logStreamName
    }));
  }
  
  // CloudWatch Insights Query
  async insightsQuery(queryString, startTime, endTime) {
    // Start query
    const startCommand = new StartQueryCommand({
      logGroupName: this.logGroupName,
      startTime: Math.floor(startTime.getTime() / 1000),
      endTime: Math.floor(endTime.getTime() / 1000),
      queryString: queryString
    });
    
    const { queryId } = await cwLogsClient.send(startCommand);
    
    // Wait for results
    while (true) {
      const getCommand = new GetQueryResultsCommand({ queryId });
      const response = await cwLogsClient.send(getCommand);
      
      if (response.status === 'Complete') {
        return response.results;
      } else if (response.status === 'Failed') {
        throw new Error('Query failed');
      }
      
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }
}

// Logger Class
class Logger {
  constructor(appName) {
    this.appName = appName;
    this.cwLogs = new CloudWatchLogs(`/aws/application/${appName}`);
  }
  
  async log(level, message, context = {}) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      level: level,
      application: this.appName,
      message: message,
      ...context
    };
    
    await this.cwLogs.putLogs([logEntry]);
    console.log(JSON.stringify(logEntry));
  }
  
  async info(message, context) {
    await this.log('INFO', message, context);
  }
  
  async error(message, context) {
    await this.log('ERROR', message, context);
  }
  
  async warn(message, context) {
    await this.log('WARN', message, context);
  }
}

// Usage
const logger = new Logger('MyWebApp');

await logger.info('User logged in', {
  userId: 'user123',
  ipAddress: '192.168.1.1'
});

await logger.error('Database connection failed', {
  database: 'users-db',
  error: 'Connection timeout',
  retryCount: 3
});

// CloudWatch Insights query
const results = await logger.cwLogs.insightsQuery(
  `fields @timestamp, level, message, userId
   | filter level = "ERROR"
   | sort @timestamp desc
   | limit 100`,
  new Date(Date.now() - 24 * 60 * 60 * 1000),
  new Date()
);
```

### Express.js Middleware:

```javascript
// cloudwatch-middleware.js
import { ApplicationMonitoring } from './cloudwatch-service.js';

const monitoring = new ApplicationMonitoring('MyAPI');

export function cloudwatchMiddleware(req, res, next) {
  const startTime = Date.now();
  
  res.on('finish', async () => {
    const duration = Date.now() - startTime;
    
    await monitoring.trackRequest(
      req.path,
      req.method,
      res.statusCode.toString(),
      duration
    );
    
    if (res.statusCode >= 400) {
      await monitoring.trackError(
        `HTTP${res.statusCode}`,
        req.path
      );
    }
  });
  
  next();
}

// Usage in Express
import express from 'express';
const app = express();

app.use(cloudwatchMiddleware);
```

### Best Practices:

1. ✅ Use custom metrics for business KPIs
2. ✅ Set up alarms for critical metrics
3. ✅ Use log groups to organize logs
4. ✅ Enable detailed monitoring for EC2
5. ✅ Use metric filters for log analysis
6. ✅ Create dashboards for visualization
7. ✅ Use CloudWatch Insights for log queries
8. ✅ Implement structured logging (JSON)
9. ✅ Set log retention policies
10. ✅ Use anomaly detection for smart alarms

---

## Question 27: Explain AWS CloudFormation for Infrastructure as Code

### 📋 Answer

**AWS CloudFormation** provides a common language to describe and provision AWS infrastructure resources in a safe and repeatable manner.

### CloudFormation Concepts:

```
AWS CloudFormation
├── Templates
│   ├── JSON or YAML
│   ├── Resources (required)
│   ├── Parameters
│   ├── Mappings
│   ├── Conditions
│   ├── Outputs
│   └── Metadata
├── Stacks
│   ├── Create
│   ├── Update
│   ├── Delete
│   └── Change Sets
└── StackSets
    └── Multi-account/region deployment
```

### Complete VPC Stack Template:

```yaml
# vpc-stack.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Complete VPC with public and private subnets'

Parameters:
  EnvironmentName:
    Description: Environment name prefix
    Type: String
    Default: Production
    
  VpcCIDR:
    Description: CIDR block for VPC
    Type: String
    Default: 10.0.0.0/16
    
  PublicSubnet1CIDR:
    Description: CIDR for public subnet 1
    Type: String
    Default: 10.0.1.0/24
    
  PublicSubnet2CIDR:
    Description: CIDR for public subnet 2
    Type: String
    Default: 10.0.2.0/24
    
  PrivateSubnet1CIDR:
    Description: CIDR for private subnet 1
    Type: String
    Default: 10.0.11.0/24
    
  PrivateSubnet2CIDR:
    Description: CIDR for private subnet 2
    Type: String
    Default: 10.0.12.0/24

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCIDR
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-VPC
          
  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-IGW
          
  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway
      
  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      CidrBlock: !Ref PublicSubnet1CIDR
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-PublicSubnet1
          
  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      CidrBlock: !Ref PublicSubnet2CIDR
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-PublicSubnet2
          
  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      CidrBlock: !Ref PrivateSubnet1CIDR
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-PrivateSubnet1
          
  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      CidrBlock: !Ref PrivateSubnet2CIDR
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-PrivateSubnet2
          
  NatGateway1EIP:
    Type: AWS::EC2::EIP
    DependsOn: AttachGateway
    Properties:
      Domain: vpc
      
  NatGateway2EIP:
    Type: AWS::EC2::EIP
    DependsOn: AttachGateway
    Properties:
      Domain: vpc
      
  NatGateway1:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGateway1EIP.AllocationId
      SubnetId: !Ref PublicSubnet1
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-NAT1
          
  NatGateway2:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGateway2EIP.AllocationId
      SubnetId: !Ref PublicSubnet2
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-NAT2
          
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-PublicRoutes
          
  DefaultPublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway
      
  PublicSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet1
      RouteTableId: !Ref PublicRouteTable
      
  PublicSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet2
      RouteTableId: !Ref PublicRouteTable
      
  PrivateRouteTable1:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-PrivateRoutes1
          
  DefaultPrivateRoute1:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable1
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway1
      
  PrivateSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet1
      RouteTableId: !Ref PrivateRouteTable1
      
  PrivateRouteTable2:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-PrivateRoutes2
          
  DefaultPrivateRoute2:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable2
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway2
      
  PrivateSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet2
      RouteTableId: !Ref PrivateRouteTable2

Outputs:
  VPC:
    Description: VPC ID
    Value: !Ref VPC
    Export:
      Name: !Sub ${EnvironmentName}-VPC
      
  PublicSubnets:
    Description: Public subnet IDs
    Value: !Join [',', [!Ref PublicSubnet1, !Ref PublicSubnet2]]
    Export:
      Name: !Sub ${EnvironmentName}-PublicSubnets
      
  PrivateSubnets:
    Description: Private subnet IDs
    Value: !Join [',', [!Ref PrivateSubnet1, !Ref PrivateSubnet2]]
    Export:
      Name: !Sub ${EnvironmentName}-PrivateSubnets
```

### CloudFormation SDK Operations:

```javascript
// cloudformation-manager.js
import {
  CloudFormationClient,
  CreateStackCommand,
  UpdateStackCommand,
  DeleteStackCommand,
  DescribeStacksCommand,
  DescribeStackEventsCommand,
  ValidateTemplateCommand,
  CreateChangeSetCommand,
  ExecuteChangeSetCommand,
  ListStacksCommand
} from '@aws-sdk/client-cloudformation';
import { readFileSync } from 'fs';

const cfnClient = new CloudFormationClient({ region: 'us-east-1' });

class CloudFormationManager {
  // Create stack
  async createStack(stackName, templatePath, parameters) {
    const templateBody = readFileSync(templatePath, 'utf8');
    
    const command = new CreateStackCommand({
      StackName: stackName,
      TemplateBody: templateBody,
      Parameters: Object.entries(parameters).map(([key, value]) => ({
        ParameterKey: key,
        ParameterValue: value
      })),
      Capabilities: ['CAPABILITY_IAM', 'CAPABILITY_NAMED_IAM'],
      Tags: [
        { Key: 'ManagedBy', Value: 'CloudFormation' },
        { Key: 'Environment', Value: 'Production' }
      ]
    });
    
    try {
      const response = await cfnClient.send(command);
      console.log('Stack creation started:', response.StackId);
      return response.StackId;
    } catch (error) {
      console.error('Failed to create stack:', error);
      throw error;
    }
  }
  
  // Wait for stack completion
  async waitForStack(stackName, pollInterval = 10000) {
    while (true) {
      const status = await this.getStackStatus(stackName);
      
      console.log('Stack status:', status);
      
      if (status.endsWith('_COMPLETE')) {
        return status;
      } else if (status.endsWith('_FAILED') || status === 'ROLLBACK_COMPLETE') {
        const events = await this.getStackEvents(stackName);
        const failedEvents = events.filter(e => 
          e.ResourceStatus.endsWith('_FAILED')
        );
        throw new Error(`Stack operation failed: ${JSON.stringify(failedEvents)}`);
      }
      
      await new Promise(resolve => setTimeout(resolve, pollInterval));
    }
  }
  
  // Get stack status
  async getStackStatus(stackName) {
    const command = new DescribeStacksCommand({
      StackName: stackName
    });
    
    const response = await cfnClient.send(command);
    return response.Stacks[0].StackStatus;
  }
  
  // Get stack outputs
  async getStackOutputs(stackName) {
    const command = new DescribeStacksCommand({
      StackName: stackName
    });
    
    const response = await cfnClient.send(command);
    const outputs = response.Stacks[0].Outputs || [];
    
    return outputs.reduce((acc, output) => {
      acc[output.OutputKey] = output.OutputValue;
      return acc;
    }, {});
  }
  
  // Get stack events
  async getStackEvents(stackName) {
    const command = new DescribeStackEventsCommand({
      StackName: stackName
    });
    
    const response = await cfnClient.send(command);
    return response.StackEvents;
  }
  
  // Update stack
  async updateStack(stackName, templatePath, parameters) {
    const templateBody = readFileSync(templatePath, 'utf8');
    
    const command = new UpdateStackCommand({
      StackName: stackName,
      TemplateBody: templateBody,
      Parameters: Object.entries(parameters).map(([key, value]) => ({
        ParameterKey: key,
        ParameterValue: value
      })),
      Capabilities: ['CAPABILITY_IAM', 'CAPABILITY_NAMED_IAM']
    });
    
    try {
      await cfnClient.send(command);
      console.log('Stack update started');
    } catch (error) {
      if (error.message.includes('No updates')) {
        console.log('No updates to perform');
      } else {
        throw error;
      }
    }
  }
  
  // Delete stack
  async deleteStack(stackName) {
    const command = new DeleteStackCommand({
      StackName: stackName
    });
    
    await cfnClient.send(command);
    console.log('Stack deletion started');
  }
}

// Usage
const cfnManager = new CloudFormationManager();

// Create stack
const stackId = await cfnManager.createStack(
  'my-vpc-stack',
  './vpc-stack.yaml',
  {
    EnvironmentName: 'Production',
    VpcCIDR: '10.0.0.0/16'
  }
);

// Wait for completion
await cfnManager.waitForStack('my-vpc-stack');

// Get outputs
const outputs = await cfnManager.getStackOutputs('my-vpc-stack');
console.log('Stack outputs:', outputs);
```

### Best Practices:

1. ✅ Use parameters for reusability
2. ✅ Validate templates before deployment
3. ✅ Use change sets for updates
4. ✅ Implement rollback on failure
5. ✅ Use nested stacks for modularity
6. ✅ Export values for cross-stack references
7. ✅ Use stack policies to prevent updates
8. ✅ Enable termination protection
9. ✅ Tag resources consistently
10. ✅ Use StackSets for multi-account deployment

---

## Key Takeaways

### Question 25 (Athena):
- Serverless SQL queries on S3 data
- Pay per query (data scanned)
- Supports Parquet, ORC, JSON, CSV
- Partition data for performance
- Use CTAS for ETL operations
- Integrates with Glue Data Catalog

### Question 26 (CloudWatch):
- Unified monitoring platform
- Custom metrics for application monitoring
- Log aggregation and analysis
- Metric-based and composite alarms
- CloudWatch Insights for log queries
- Dashboards for visualization

### Question 27 (CloudFormation):
- Infrastructure as Code (IaC)
- YAML or JSON templates
- Automated resource provisioning
- Stack updates with change sets
- Cross-stack references with exports
- StackSets for multi-account deployment
- Rollback on failures
