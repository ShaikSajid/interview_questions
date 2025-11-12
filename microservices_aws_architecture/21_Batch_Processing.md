# Batch Processing

## Question 41: AWS Batch & Step Functions for Batch Jobs

### 📋 Question Statement

Implement batch processing for Emirates NBD end-of-day reconciliation, statement generation, and interest calculation using AWS Batch, Step Functions, and S3.

---

### 🔄 Batch Processing Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                 END-OF-DAY BATCH PROCESSING PIPELINE                        │
└────────────────────────────────────────────────────────────────────────────┘

    ┌──────────────┐
    │  EventBridge │
    │  (11:00 PM)  │
    └──────┬───────┘
           │
           v
    ┌─────────────────┐
    │ Step Functions  │
    │  Orchestrator   │
    └────────┬────────┘
             │
    ┌────────┼────────┬────────────┬─────────────┐
    │        │        │            │             │
    v        v        v            v             v
┌───────┐ ┌──────┐ ┌─────────┐ ┌─────────┐ ┌──────────┐
│ Batch │ │ Batch│ │  Batch  │ │  Batch  │ │  Batch   │
│Reconcile│ │Interest│ │Statement│ │ Reports │ │ Archival │
│  Job  │ │  Job │ │   Job   │ │   Job   │ │   Job    │
└───┬───┘ └───┬──┘ └────┬────┘ └────┬────┘ └────┬─────┘
    │         │         │           │           │
    └─────────┴─────────┴───────────┴───────────┘
                        │
                        v
                  ┌──────────┐
                  │    S3    │
                  │ Results  │
                  └──────────┘
```

### 📦 AWS Batch CDK Infrastructure

```typescript
// infrastructure/cdk/batch-processing-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as batch from 'aws-cdk-lib/aws-batch';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as sfn from 'aws-cdk-lib/aws-stepfunctions';
import * as tasks from 'aws-cdk-lib/aws-stepfunctions-tasks';
import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';
import * as logs from 'aws-cdk-lib/aws-logs';

export class BatchProcessingStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // VPC for Batch compute environment
    const vpc = new ec2.Vpc(this, 'BatchVPC', {
      maxAzs: 2,
      natGateways: 1
    });

    // S3 Bucket for batch outputs
    const batchOutputBucket = new s3.Bucket(this, 'BatchOutputs', {
      bucketName: 'emirates-nbd-batch-outputs',
      encryption: s3.BucketEncryption.S3_MANAGED,
      versioned: true,
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

    // Batch execution role
    const batchExecutionRole = new iam.Role(this, 'BatchExecutionRole', {
      assumedBy: new iam.ServicePrincipal('ecs-tasks.amazonaws.com'),
      managedPolicies: [
        iam.ManagedPolicy.fromAwsManagedPolicyName('service-role/AmazonECSTaskExecutionRolePolicy')
      ]
    });

    // Batch job role
    const batchJobRole = new iam.Role(this, 'BatchJobRole', {
      assumedBy: new iam.ServicePrincipal('ecs-tasks.amazonaws.com')
    });

    batchOutputBucket.grantReadWrite(batchJobRole);

    // Grant RDS access for reconciliation
    batchJobRole.addToPolicy(new iam.PolicyStatement({
      actions: ['rds:*', 'rds-db:connect'],
      resources: ['*']
    }));

    // Batch Compute Environment
    const computeEnvironment = new batch.CfnComputeEnvironment(this, 'BatchComputeEnv', {
      type: 'MANAGED',
      computeEnvironmentName: 'eod-batch-compute',
      computeResources: {
        type: 'FARGATE',
        maxvCpus: 256,
        subnets: vpc.privateSubnets.map(subnet => subnet.subnetId),
        securityGroupIds: [vpc.vpcDefaultSecurityGroup]
      },
      state: 'ENABLED'
    });

    // Batch Job Queue
    const jobQueue = new batch.CfnJobQueue(this, 'BatchJobQueue', {
      jobQueueName: 'eod-job-queue',
      priority: 1,
      computeEnvironmentOrder: [
        {
          order: 1,
          computeEnvironment: computeEnvironment.attrComputeEnvironmentArn
        }
      ]
    });

    // Job Definition: Reconciliation
    const reconciliationJobDef = new batch.CfnJobDefinition(this, 'ReconciliationJob', {
      jobDefinitionName: 'reconciliation-job',
      type: 'container',
      platformCapabilities: ['FARGATE'],
      containerProperties: {
        image: `${this.account}.dkr.ecr.${this.region}.amazonaws.com/batch-reconciliation:latest`,
        resourceRequirements: [
          { type: 'VCPU', value: '4' },
          { type: 'MEMORY', value: '8192' }
        ],
        executionRoleArn: batchExecutionRole.roleArn,
        jobRoleArn: batchJobRole.roleArn,
        environment: [
          { name: 'OUTPUT_BUCKET', value: batchOutputBucket.bucketName },
          { name: 'DB_HOST', value: 'banking-db.cluster-xxx.rds.amazonaws.com' },
          { name: 'DB_NAME', value: 'banking' }
        ],
        logConfiguration: {
          logDriver: 'awslogs',
          options: {
            'awslogs-group': '/aws/batch/reconciliation',
            'awslogs-region': this.region,
            'awslogs-stream-prefix': 'reconciliation'
          }
        },
        networkConfiguration: {
          assignPublicIp: 'DISABLED'
        }
      },
      retryStrategy: {
        attempts: 3,
        evaluateOnExit: [
          {
            action: 'RETRY',
            onStatusReason: 'Task failed to start'
          },
          {
            action: 'EXIT',
            onExitCode: '0'
          }
        ]
      },
      timeout: {
        attemptDurationSeconds: 3600 // 1 hour
      }
    });

    // Job Definition: Interest Calculation
    const interestJobDef = new batch.CfnJobDefinition(this, 'InterestCalculationJob', {
      jobDefinitionName: 'interest-calculation-job',
      type: 'container',
      platformCapabilities: ['FARGATE'],
      containerProperties: {
        image: `${this.account}.dkr.ecr.${this.region}.amazonaws.com/batch-interest:latest`,
        resourceRequirements: [
          { type: 'VCPU', value: '2' },
          { type: 'MEMORY', value: '4096' }
        ],
        executionRoleArn: batchExecutionRole.roleArn,
        jobRoleArn: batchJobRole.roleArn,
        environment: [
          { name: 'OUTPUT_BUCKET', value: batchOutputBucket.bucketName }
        ],
        logConfiguration: {
          logDriver: 'awslogs',
          options: {
            'awslogs-group': '/aws/batch/interest',
            'awslogs-region': this.region,
            'awslogs-stream-prefix': 'interest'
          }
        }
      },
      retryStrategy: {
        attempts: 2
      }
    });

    // Job Definition: Statement Generation
    const statementJobDef = new batch.CfnJobDefinition(this, 'StatementGenerationJob', {
      jobDefinitionName: 'statement-generation-job',
      type: 'container',
      platformCapabilities: ['FARGATE'],
      containerProperties: {
        image: `${this.account}.dkr.ecr.${this.region}.amazonaws.com/batch-statements:latest`,
        resourceRequirements: [
          { type: 'VCPU', value: '4' },
          { type: 'MEMORY', value: '8192' }
        ],
        executionRoleArn: batchExecutionRole.roleArn,
        jobRoleArn: batchJobRole.roleArn,
        environment: [
          { name: 'OUTPUT_BUCKET', value: batchOutputBucket.bucketName }
        ],
        logConfiguration: {
          logDriver: 'awslogs',
          options: {
            'awslogs-group': '/aws/batch/statements',
            'awslogs-region': this.region,
            'awslogs-stream-prefix': 'statements'
          }
        }
      }
    });

    // Step Functions Orchestration
    const reconciliationTask = new tasks.BatchSubmitJob(this, 'ReconciliationTask', {
      jobDefinitionArn: reconciliationJobDef.ref,
      jobQueueArn: jobQueue.attrJobQueueArn,
      jobName: 'eod-reconciliation',
      resultPath: '$.reconciliation'
    });

    const interestTask = new tasks.BatchSubmitJob(this, 'InterestTask', {
      jobDefinitionArn: interestJobDef.ref,
      jobQueueArn: jobQueue.attrJobQueueArn,
      jobName: 'eod-interest',
      resultPath: '$.interest'
    });

    const statementTask = new tasks.BatchSubmitJob(this, 'StatementTask', {
      jobDefinitionArn: statementJobDef.ref,
      jobQueueArn: jobQueue.attrJobQueueArn,
      jobName: 'eod-statements',
      resultPath: '$.statements'
    });

    // Parallel execution for interest and statements
    const parallelProcessing = new sfn.Parallel(this, 'ParallelProcessing')
      .branch(interestTask)
      .branch(statementTask);

    // Step Functions definition
    const definition = reconciliationTask
      .next(parallelProcessing);

    const stateMachine = new sfn.StateMachine(this, 'EODStateMachine', {
      stateMachineName: 'eod-batch-processing',
      definition,
      timeout: cdk.Duration.hours(4),
      logs: {
        destination: new logs.LogGroup(this, 'EODStateMachineLogs', {
          retention: logs.RetentionDays.ONE_MONTH
        }),
        level: sfn.LogLevel.ALL
      }
    });

    // EventBridge rule to trigger at 11:00 PM daily
    new events.Rule(this, 'EODSchedule', {
      schedule: events.Schedule.cron({
        minute: '0',
        hour: '23',
        weekDay: '*'
      }),
      targets: [new targets.SfnStateMachine(stateMachine)]
    });

    // Outputs
    new cdk.CfnOutput(this, 'JobQueueArn', {
      value: jobQueue.attrJobQueueArn
    });

    new cdk.CfnOutput(this, 'StateMachineArn', {
      value: stateMachine.stateMachineArn
    });

    new cdk.CfnOutput(this, 'OutputBucket', {
      value: batchOutputBucket.bucketName
    });
  }
}
```

### 💼 Reconciliation Batch Job

```typescript
// batch-jobs/reconciliation/src/index.ts
import { RDSDataClient, ExecuteStatementCommand } from '@aws-sdk/client-rds-data';
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';

const rds = new RDSDataClient({});
const s3 = new S3Client({});

const DB_CLUSTER_ARN = process.env.DB_CLUSTER_ARN!;
const DB_NAME = process.env.DB_NAME!;
const OUTPUT_BUCKET = process.env.OUTPUT_BUCKET!;

interface ReconciliationResult {
  totalTransactions: number;
  totalAmount: number;
  matchedTransactions: number;
  unmatchedTransactions: number;
  discrepancies: Discrepancy[];
}

interface Discrepancy {
  transactionId: string;
  expectedAmount: number;
  actualAmount: number;
  difference: number;
}

async function reconcile(): Promise<ReconciliationResult> {
  console.log('Starting end-of-day reconciliation...');

  // Fetch all transactions for the day
  const transactions = await executeQuery(`
    SELECT 
      t.transaction_id,
      t.amount,
      t.status,
      t.created_at,
      a.balance_before,
      a.balance_after
    FROM transactions t
    JOIN account_balances a ON t.account_id = a.account_id
    WHERE DATE(t.created_at) = CURRENT_DATE
    ORDER BY t.created_at
  `);

  const discrepancies: Discrepancy[] = [];
  let matchedCount = 0;

  // Reconcile each transaction
  for (const txn of transactions) {
    const expectedBalance = txn.balance_before + txn.amount;
    
    if (Math.abs(expectedBalance - txn.balance_after) > 0.01) {
      discrepancies.push({
        transactionId: txn.transaction_id,
        expectedAmount: expectedBalance,
        actualAmount: txn.balance_after,
        difference: txn.balance_after - expectedBalance
      });
    } else {
      matchedCount++;
    }
  }

  const result: ReconciliationResult = {
    totalTransactions: transactions.length,
    totalAmount: transactions.reduce((sum, t) => sum + t.amount, 0),
    matchedTransactions: matchedCount,
    unmatchedTransactions: discrepancies.length,
    discrepancies
  };

  // Upload results to S3
  await uploadResults('reconciliation', result);

  console.log(`Reconciliation complete: ${matchedCount}/${transactions.length} matched`);
  
  if (discrepancies.length > 0) {
    console.error(`Found ${discrepancies.length} discrepancies`);
    throw new Error('Reconciliation failed with discrepancies');
  }

  return result;
}

async function executeQuery(sql: string): Promise<any[]> {
  const response = await rds.send(
    new ExecuteStatementCommand({
      resourceArn: DB_CLUSTER_ARN,
      secretArn: process.env.DB_SECRET_ARN,
      database: DB_NAME,
      sql
    })
  );

  return response.records || [];
}

async function uploadResults(jobType: string, results: any): Promise<void> {
  const date = new Date().toISOString().split('T')[0];
  const key = `${jobType}/${date}/${Date.now()}.json`;

  await s3.send(
    new PutObjectCommand({
      Bucket: OUTPUT_BUCKET,
      Key: key,
      Body: JSON.stringify(results, null, 2),
      ContentType: 'application/json'
    })
  );

  console.log(`Results uploaded to s3://${OUTPUT_BUCKET}/${key}`);
}

// Main execution
reconcile()
  .then(() => {
    console.log('Batch job completed successfully');
    process.exit(0);
  })
  .catch((error) => {
    console.error('Batch job failed:', error);
    process.exit(1);
  });
```

### 💰 Interest Calculation Job

```typescript
// batch-jobs/interest/src/index.ts
import { RDSDataClient, ExecuteStatementCommand, BatchExecuteStatementCommand } from '@aws-sdk/client-rds-data';
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';

const rds = new RDSDataClient({});
const s3 = new S3Client({});

interface InterestCalculation {
  accountId: string;
  accountType: string;
  balance: number;
  interestRate: number;
  daysInPeriod: number;
  interestEarned: number;
}

async function calculateInterest(): Promise<void> {
  console.log('Starting interest calculation...');

  // Fetch all eligible accounts
  const accounts = await executeQuery(`
    SELECT 
      account_id,
      account_type,
      balance,
      interest_rate
    FROM accounts
    WHERE status = 'ACTIVE'
      AND account_type IN ('SAVINGS', 'FIXED_DEPOSIT')
      AND balance > 0
  `);

  const calculations: InterestCalculation[] = [];
  const updates: any[] = [];

  for (const account of accounts) {
    const daysInPeriod = 1; // Daily calculation
    const dailyRate = account.interest_rate / 365;
    const interestEarned = account.balance * dailyRate / 100;

    calculations.push({
      accountId: account.account_id,
      accountType: account.account_type,
      balance: account.balance,
      interestRate: account.interest_rate,
      daysInPeriod,
      interestEarned
    });

    // Prepare batch update
    updates.push({
      sql: `
        UPDATE accounts 
        SET interest_accrued = interest_accrued + :interest,
            updated_at = NOW()
        WHERE account_id = :accountId
      `,
      parameters: [
        { name: 'interest', value: { doubleValue: interestEarned } },
        { name: 'accountId', value: { stringValue: account.account_id } }
      ]
    });
  }

  // Batch execute updates
  if (updates.length > 0) {
    await batchExecute(updates);
  }

  // Upload results
  await uploadResults('interest', {
    date: new Date().toISOString(),
    totalAccounts: accounts.length,
    totalInterest: calculations.reduce((sum, c) => sum + c.interestEarned, 0),
    calculations
  });

  console.log(`Interest calculated for ${accounts.length} accounts`);
}

async function executeQuery(sql: string): Promise<any[]> {
  const response = await rds.send(
    new ExecuteStatementCommand({
      resourceArn: process.env.DB_CLUSTER_ARN!,
      secretArn: process.env.DB_SECRET_ARN!,
      database: process.env.DB_NAME!,
      sql
    })
  );

  return response.records || [];
}

async function batchExecute(statements: any[]): Promise<void> {
  // Process in chunks of 100
  for (let i = 0; i < statements.length; i += 100) {
    const chunk = statements.slice(i, i + 100);
    
    await rds.send(
      new BatchExecuteStatementCommand({
        resourceArn: process.env.DB_CLUSTER_ARN!,
        secretArn: process.env.DB_SECRET_ARN!,
        database: process.env.DB_NAME!,
        sqls: chunk.map(s => s.sql),
        parameterSets: chunk.map(s => s.parameters)
      })
    );
  }
}

async function uploadResults(jobType: string, results: any): Promise<void> {
  const date = new Date().toISOString().split('T')[0];
  const key = `${jobType}/${date}/${Date.now()}.json`;

  await s3.send(
    new PutObjectCommand({
      Bucket: process.env.OUTPUT_BUCKET!,
      Key: key,
      Body: JSON.stringify(results, null, 2),
      ContentType: 'application/json'
    })
  );
}

calculateInterest()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error('Interest calculation failed:', error);
    process.exit(1);
  });
```

### 🎓 Interview Discussion Points - Q41

**Q1: When to use AWS Batch vs Lambda?**

**A**:
**AWS Batch:**
- Long-running jobs (>15 minutes)
- High compute/memory requirements
- Large data processing (GB/TB)
- Docker containers
- Use case: EOD reconciliation, data processing

**Lambda:**
- Short-lived tasks (<15 minutes)
- Event-driven processing
- Lightweight compute
- No container management
- Use case: API, stream processing

**Q2: How to handle batch job failures?**

**A**:
- **Retry strategy**: Configure automatic retries (1-10 attempts)
- **DLQ**: Send failed jobs to Dead Letter Queue
- **Idempotency**: Design jobs to be safely retried
- **Checkpointing**: Save progress to resume from failure
- **Monitoring**: CloudWatch alarms for job failures
- **Manual intervention**: SNS alerts for critical failures

**Q3: How to optimize batch job costs?**

**A**:
- **Fargate Spot**: Up to 70% savings for fault-tolerant jobs
- **Right-sizing**: Choose appropriate vCPU/memory
- **Parallelization**: Process data in parallel
- **Compression**: Compress data before processing
- **Scheduled execution**: Run during off-peak hours
- **Auto-scaling**: Scale compute based on queue depth

**Q4: What is Step Functions orchestration?**

**A**:
- **Sequential**: Execute tasks one after another
- **Parallel**: Execute multiple tasks simultaneously
- **Choice**: Conditional branching based on state
- **Wait**: Add delays between tasks
- **Retry/Catch**: Handle errors and retries
- **Map**: Process arrays in parallel

**Q5: How to monitor batch jobs?**

**A**:
- **CloudWatch Logs**: Application logs from containers
- **CloudWatch Metrics**: Job duration, success/failure rate
- **EventBridge**: Trigger on job state changes
- **X-Ray**: Distributed tracing
- **Custom metrics**: Business KPIs (records processed, errors)

---

## Question 42: Batch Job Monitoring & Error Handling

### 📋 Question Statement

Implement comprehensive monitoring and error handling for batch jobs including CloudWatch alarms, SNS notifications, automatic retries, dead letter queues, and job recovery strategies.

---

### 📊 Batch Job Monitoring CDK Stack

```typescript
// infrastructure/cdk/batch-monitoring-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as cloudwatch from 'aws-cdk-lib/aws-cloudwatch';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as subscriptions from 'aws-cdk-lib/aws-sns-subscriptions';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as sqs from 'aws-cdk-lib/aws-sqs';
import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';

export class BatchMonitoringStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // SNS Topics for notifications
    const criticalAlertTopic = new sns.Topic(this, 'CriticalAlerts', {
      displayName: 'Batch Job Critical Alerts'
    });

    const warningAlertTopic = new sns.Topic(this, 'WarningAlerts', {
      displayName: 'Batch Job Warnings'
    });

    // Add email subscriptions
    criticalAlertTopic.addSubscription(
      new subscriptions.EmailSubscription('platform-team@emiratesnbd.com')
    );

    criticalAlertTopic.addSubscription(
      new subscriptions.SmsSubscription('+971501234567')
    );

    // Dead Letter Queue for failed jobs
    const dlq = new sqs.Queue(this, 'BatchJobDLQ', {
      queueName: 'batch-jobs-dlq',
      retentionPeriod: cdk.Duration.days(14),
      encryption: sqs.QueueEncryption.KMS_MANAGED
    });

    // Batch job monitoring Lambda
    const monitoringFunction = new lambda.Function(this, 'BatchMonitor', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/batch-monitoring'),
      environment: {
        CRITICAL_TOPIC_ARN: criticalAlertTopic.topicArn,
        WARNING_TOPIC_ARN: warningAlertTopic.topicArn,
        DLQ_URL: dlq.queueUrl
      },
      timeout: cdk.Duration.minutes(5)
    });

    criticalAlertTopic.grantPublish(monitoringFunction);
    warningAlertTopic.grantPublish(monitoringFunction);
    dlq.grantSendMessages(monitoringFunction);

    // CloudWatch Alarms

    // 1. Batch Job Failure Rate
    const failureRateAlarm = new cloudwatch.Alarm(this, 'BatchJobFailureRate', {
      metric: new cloudwatch.Metric({
        namespace: 'Banking/BatchJobs',
        metricName: 'FailedJobs',
        statistic: 'Sum',
        period: cdk.Duration.minutes(5)
      }),
      threshold: 3,
      evaluationPeriods: 1,
      alarmDescription: 'Alert when batch jobs fail',
      actionsEnabled: true
    });

    failureRateAlarm.addAlarmAction(
      new cdk.aws_cloudwatch_actions.SnsAction(criticalAlertTopic)
    );

    // 2. Batch Job Duration
    const durationAlarm = new cloudwatch.Alarm(this, 'BatchJobDuration', {
      metric: new cloudwatch.Metric({
        namespace: 'Banking/BatchJobs',
        metricName: 'JobDuration',
        statistic: 'Maximum',
        period: cdk.Duration.minutes(15)
      }),
      threshold: 3600, // 1 hour
      evaluationPeriods: 1,
      alarmDescription: 'Alert when batch job takes too long'
    });

    durationAlarm.addAlarmAction(
      new cdk.aws_cloudwatch_actions.SnsAction(warningAlertTopic)
    );

    // 3. DLQ Depth
    const dlqDepthAlarm = new cloudwatch.Alarm(this, 'DLQDepth', {
      metric: dlq.metricApproximateNumberOfMessagesVisible(),
      threshold: 10,
      evaluationPeriods: 1,
      alarmDescription: 'Alert when DLQ has messages'
    });

    dlqDepthAlarm.addAlarmAction(
      new cdk.aws_cloudwatch_actions.SnsAction(criticalAlertTopic)
    );

    // EventBridge rule for failed Step Functions executions
    const failedExecutionRule = new events.Rule(this, 'StepFunctionsFailed', {
      eventPattern: {
        source: ['aws.states'],
        detailType: ['Step Functions Execution Status Change'],
        detail: {
          status: ['FAILED', 'TIMED_OUT', 'ABORTED']
        }
      }
    });

    failedExecutionRule.addTarget(new targets.LambdaFunction(monitoringFunction));

    // Dashboard
    const dashboard = new cloudwatch.Dashboard(this, 'BatchJobDashboard', {
      dashboardName: 'Banking-Batch-Jobs'
    });

    dashboard.addWidgets(
      new cloudwatch.GraphWidget({
        title: 'Job Success/Failure Rate',
        left: [
          new cloudwatch.Metric({
            namespace: 'Banking/BatchJobs',
            metricName: 'SuccessfulJobs',
            statistic: 'Sum',
            period: cdk.Duration.minutes(5),
            label: 'Successful',
            color: '#2ca02c'
          }),
          new cloudwatch.Metric({
            namespace: 'Banking/BatchJobs',
            metricName: 'FailedJobs',
            statistic: 'Sum',
            period: cdk.Duration.minutes(5),
            label: 'Failed',
            color: '#d62728'
          })
        ]
      }),
      new cloudwatch.GraphWidget({
        title: 'Job Duration (seconds)',
        left: [
          new cloudwatch.Metric({
            namespace: 'Banking/BatchJobs',
            metricName: 'JobDuration',
            statistic: 'Average',
            period: cdk.Duration.minutes(5)
          })
        ]
      })
    );
  }
}
```

### 🔔 Batch Monitoring Lambda

```typescript
// lambda/batch-monitoring/index.ts
import { EventBridgeEvent } from 'aws-lambda';
import { SNS, SQS, StepFunctions, CloudWatch } from 'aws-sdk';

interface StepFunctionFailureEvent {
  executionArn: string;
  stateMachineArn: string;
  name: string;
  status: 'FAILED' | 'TIMED_OUT' | 'ABORTED';
  startDate: string;
  stopDate: string;
  cause?: string;
  error?: string;
}

export const handler = async (event: EventBridgeEvent<string, StepFunctionFailureEvent>) => {
  const sns = new SNS();
  const sqs = new SQS();
  const sfn = new StepFunctions();
  const cloudwatch = new CloudWatch();

  const detail = event.detail;

  console.log('Batch job failure detected:', JSON.stringify(detail, null, 2));

  // Get execution history for detailed error
  const executionHistory = await sfn.getExecutionHistory({
    executionArn: detail.executionArn,
    reverseOrder: true,
    maxResults: 10
  }).promise();

  const failedEvent = executionHistory.events.find(e => 
    e.type === 'TaskFailed' || e.type === 'ExecutionFailed'
  );

  const errorDetails = {
    jobName: detail.name,
    executionArn: detail.executionArn,
    status: detail.status,
    cause: detail.cause || failedEvent?.taskFailedEventDetails?.cause || 'Unknown',
    error: detail.error || failedEvent?.taskFailedEventDetails?.error || 'Unknown',
    duration: calculateDuration(detail.startDate, detail.stopDate)
  };

  // Determine severity
  const isCritical = detail.status === 'FAILED' || errorDetails.jobName.includes('CRITICAL');

  // Send SNS notification
  const topicArn = isCritical 
    ? process.env.CRITICAL_TOPIC_ARN 
    : process.env.WARNING_TOPIC_ARN;

  await sns.publish({
    TopicArn: topicArn!,
    Subject: `🚨 Batch Job ${detail.status}: ${errorDetails.jobName}`,
    Message: `
Batch Job Failure Report
=======================

Job Name: ${errorDetails.jobName}
Status: ${detail.status}
Execution ARN: ${detail.executionArn}

Error Details:
--------------
Error: ${errorDetails.error}
Cause: ${errorDetails.cause}

Timing:
-------
Started: ${detail.startDate}
Stopped: ${detail.stopDate}
Duration: ${errorDetails.duration} seconds

Action Required:
----------------
1. Check CloudWatch Logs for detailed error trace
2. Verify input data integrity
3. Check downstream service dependencies
4. Review DLQ for failed messages

Dashboard: https://console.aws.amazon.com/cloudwatch/dashboards/Banking-Batch-Jobs
    `.trim()
  }).promise();

  // Send to DLQ for manual review
  await sqs.sendMessage({
    QueueUrl: process.env.DLQ_URL!,
    MessageBody: JSON.stringify(errorDetails),
    MessageAttributes: {
      JobName: { DataType: 'String', StringValue: errorDetails.jobName },
      Status: { DataType: 'String', StringValue: detail.status },
      Timestamp: { DataType: 'String', StringValue: new Date().toISOString() }
    }
  }).promise();

  // Publish custom metric
  await cloudwatch.putMetricData({
    Namespace: 'Banking/BatchJobs',
    MetricData: [
      {
        MetricName: 'FailedJobs',
        Value: 1,
        Unit: 'Count',
        Timestamp: new Date(),
        Dimensions: [
          { Name: 'JobName', Value: errorDetails.jobName },
          { Name: 'Status', Value: detail.status }
        ]
      }
    ]
  }).promise();

  console.log('Monitoring complete');
};

function calculateDuration(start: string, stop: string): number {
  const startTime = new Date(start).getTime();
  const stopTime = new Date(stop).getTime();
  return Math.round((stopTime - startTime) / 1000);
}
```

### 🔄 Automatic Retry Strategy

```typescript
// services/batch/retry-handler.ts
import { StepFunctions, DynamoDB } from 'aws-sdk';

export class BatchJobRetryHandler {
  private sfn: StepFunctions;
  private dynamoDB: DynamoDB.DocumentClient;

  constructor() {
    this.sfn = new StepFunctions();
    this.dynamoDB = new DynamoDB.DocumentClient();
  }

  async retryFailedJob(executionArn: string, maxRetries: number = 3): Promise<RetryResult> {
    // Get job metadata
    const jobRecord = await this.getJobRecord(executionArn);

    if (jobRecord.retryCount >= maxRetries) {
      console.log(`Max retries (${maxRetries}) exceeded for ${executionArn}`);
      return {
        success: false,
        reason: 'MAX_RETRIES_EXCEEDED',
        retryCount: jobRecord.retryCount
      };
    }

    // Exponential backoff: 2^retryCount minutes
    const backoffMinutes = Math.pow(2, jobRecord.retryCount);
    await this.sleep(backoffMinutes * 60 * 1000);

    // Start new execution with same input
    const execution = await this.sfn.describeExecution({
      executionArn
    }).promise();

    const newExecution = await this.sfn.startExecution({
      stateMachineArn: execution.stateMachineArn,
      input: execution.input,
      name: `${jobRecord.jobName}-retry-${jobRecord.retryCount + 1}-${Date.now()}`
    }).promise();

    // Update retry count
    await this.updateJobRecord(executionArn, jobRecord.retryCount + 1);

    return {
      success: true,
      newExecutionArn: newExecution.executionArn,
      retryCount: jobRecord.retryCount + 1
    };
  }

  async recoverFromCheckpoint(
    jobName: string,
    checkpointId: string
  ): Promise<void> {
    // Get checkpoint data
    const checkpoint = await this.dynamoDB.get({
      TableName: 'BatchJobCheckpoints',
      Key: { jobName, checkpointId }
    }).promise();

    if (!checkpoint.Item) {
      throw new Error(`Checkpoint ${checkpointId} not found`);
    }

    // Resume from checkpoint
    const { stateMachineArn, processedRecords, totalRecords } = checkpoint.Item;

    await this.sfn.startExecution({
      stateMachineArn,
      input: JSON.stringify({
        jobName,
        resumeFrom: processedRecords,
        totalRecords
      }),
      name: `${jobName}-recovered-${Date.now()}`
    }).promise();

    console.log(`Recovered ${jobName} from checkpoint ${checkpointId}`);
  }

  private async getJobRecord(executionArn: string): Promise<JobRecord> {
    const result = await this.dynamoDB.get({
      TableName: 'BatchJobExecutions',
      Key: { executionArn }
    }).promise();

    return result.Item as JobRecord || {
      executionArn,
      jobName: 'unknown',
      retryCount: 0
    };
  }

  private async updateJobRecord(executionArn: string, retryCount: number): Promise<void> {
    await this.dynamoDB.update({
      TableName: 'BatchJobExecutions',
      Key: { executionArn },
      UpdateExpression: 'SET retryCount = :count, lastRetry = :timestamp',
      ExpressionAttributeValues: {
        ':count': retryCount,
        ':timestamp': new Date().toISOString()
      }
    }).promise();
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

interface JobRecord {
  executionArn: string;
  jobName: string;
  retryCount: number;
}

interface RetryResult {
  success: boolean;
  newExecutionArn?: string;
  retryCount: number;
  reason?: string;
}
```

### 📝 Checkpoint Service

```typescript
// services/batch/checkpoint-service.ts
import { DynamoDB } from 'aws-sdk';

export class CheckpointService {
  private dynamoDB: DynamoDB.DocumentClient;

  constructor() {
    this.dynamoDB = new DynamoDB.DocumentClient();
  }

  async saveCheckpoint(checkpoint: Checkpoint): Promise<void> {
    await this.dynamoDB.put({
      TableName: 'BatchJobCheckpoints',
      Item: {
        ...checkpoint,
        timestamp: new Date().toISOString(),
        ttl: Math.floor(Date.now() / 1000) + (7 * 24 * 60 * 60) // 7 days
      }
    }).promise();

    console.log(`Checkpoint saved: ${checkpoint.checkpointId}`);
  }

  async getLatestCheckpoint(jobName: string): Promise<Checkpoint | null> {
    const result = await this.dynamoDB.query({
      TableName: 'BatchJobCheckpoints',
      KeyConditionExpression: 'jobName = :jobName',
      ExpressionAttributeValues: {
        ':jobName': jobName
      },
      ScanIndexForward: false,
      Limit: 1
    }).promise();

    return result.Items?.[0] as Checkpoint || null;
  }

  async cleanupOldCheckpoints(jobName: string, daysToKeep: number = 7): Promise<number> {
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - daysToKeep);

    const oldCheckpoints = await this.dynamoDB.query({
      TableName: 'BatchJobCheckpoints',
      KeyConditionExpression: 'jobName = :jobName AND #ts < :cutoff',
      ExpressionAttributeNames: {
        '#ts': 'timestamp'
      },
      ExpressionAttributeValues: {
        ':jobName': jobName,
        ':cutoff': cutoffDate.toISOString()
      }
    }).promise();

    let deletedCount = 0;

    for (const checkpoint of oldCheckpoints.Items || []) {
      await this.dynamoDB.delete({
        TableName: 'BatchJobCheckpoints',
        Key: {
          jobName: checkpoint.jobName,
          checkpointId: checkpoint.checkpointId
        }
      }).promise();

      deletedCount++;
    }

    return deletedCount;
  }
}

export interface Checkpoint {
  jobName: string;
  checkpointId: string;
  stateMachineArn: string;
  processedRecords: number;
  totalRecords: number;
  state: Record<string, any>;
}
```

### 🎓 Interview Discussion Points - Q42

**Q1: What metrics should you monitor for batch jobs?**

**A**:
- **Success rate**: % of successful jobs
- **Duration**: Execution time trends
- **Data processed**: Records/bytes processed
- **Error rate**: Failed tasks per job
- **Resource utilization**: CPU, memory, network

**Q2: How to handle transient vs permanent failures?**

**A**:
- **Transient**: Network timeout, rate limiting → Retry with exponential backoff
- **Permanent**: Invalid data, logic error → DLQ for manual review
- **Detection**: Error type, retry count, error message patterns

**Q3: What is a checkpoint in batch processing?**

**A**:
- **Purpose**: Save progress periodically to resume after failure
- **Contains**: Processed record count, current state, batch ID
- **Recovery**: Resume from last checkpoint instead of restarting
- **Example**: Processed 5,000 of 10,000 records → Resume from 5,001

**Q4: How to implement circuit breaker for batch jobs?**

**A**:
```typescript
if (failureRate > 50% && sampleSize > 10) {
  throw new Error('Circuit breaker open - too many failures');
}
```
**Prevents**: Cascading failures, resource exhaustion

**Q5: What is the difference between DLQ and retry queue?**

**A**:
- **Retry Queue**: Auto-retry with backoff, temporary failures
- **DLQ**: Permanent failures, manual intervention required
- **Flow**: Job → Retry (3x) → DLQ → Manual review

---

