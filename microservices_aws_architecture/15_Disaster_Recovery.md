# Disaster Recovery

## Question 29: Comprehensive DR Strategy

### 📋 Question Statement

Implement a disaster recovery strategy for Emirates NBD with RTO/RPO requirements, automated failover, backup strategies, cross-region replication, and DR testing procedures.

---

### 🏗️ Disaster Recovery Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│           EMIRATES NBD DISASTER RECOVERY ARCHITECTURE                       │
└────────────────────────────────────────────────────────────────────────────┘

    PRIMARY REGION (us-east-1)         DR REGION (us-west-2)
    ──────────────────────────        ────────────────────────
    
    ┌──────────────────┐              ┌──────────────────┐
    │   ACTIVE SITE    │              │   STANDBY SITE   │
    │                  │              │                  │
    │  ┌────────────┐  │              │  ┌────────────┐  │
    │  │    ECS     │  │              │  │    ECS     │  │
    │  │  Services  │  │              │  │  (Scaled   │  │
    │  │  (Full)    │  │              │  │   Down)    │  │
    │  └────────────┘  │              │  └────────────┘  │
    │                  │              │                  │
    │  ┌────────────┐  │   Continuous │  ┌────────────┐  │
    │  │    RDS     │  │   Replication│  │    RDS     │  │
    │  │  Primary   │──┼──────────────┼─►│  Read      │  │
    │  │            │  │              │  │  Replica   │  │
    │  └────────────┘  │              │  └────────────┘  │
    │                  │              │                  │
    │  ┌────────────┐  │   Real-time  │  ┌────────────┐  │
    │  │ DynamoDB   │  │      Sync    │  │ DynamoDB   │  │
    │  │  Global    │◄─┼──────────────┼─►│  Global    │  │
    │  │  Table     │  │              │  │  Table     │  │
    │  └────────────┘  │              │  └────────────┘  │
    │                  │              │                  │
    │  ┌────────────┐  │   Backup     │  ┌────────────┐  │
    │  │     S3     │  │  Replication │  │     S3     │  │
    │  │  Primary   │──┼──────────────┼─►│   Backup   │  │
    │  └────────────┘  │              │  └────────────┘  │
    └──────────────────┘              └──────────────────┘
    
    RTO: 15 minutes                    RPO: < 5 minutes
```

---

### 🔧 DR Strategy Implementation

```typescript
// infrastructure/cdk/dr-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as backup from 'aws-cdk-lib/aws-backup';
import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';
import * as lambda from 'aws-cdk-lib/aws-lambda';

export class DisasterRecoveryStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // ============================================
    // BACKUP PLAN
    // ============================================
    const backupVault = new backup.BackupVault(this, 'BackupVault', {
      backupVaultName: 'banking-backup-vault',
      encryptionKey: props.kmsKey
    });

    const backupPlan = new backup.BackupPlan(this, 'BackupPlan', {
      backupPlanName: 'banking-backup-plan',
      backupVault,
      backupPlanRules: [
        // Hourly backups (retained for 24 hours)
        new backup.BackupPlanRule({
          ruleName: 'HourlyBackups',
          scheduleExpression: events.Schedule.cron({ minute: '0' }),
          deleteAfter: cdk.Duration.hours(24),
          copyActions: [{
            destinationBackupVault: drBackupVault,
            deleteAfter: cdk.Duration.hours(24)
          }]
        }),
        // Daily backups (retained for 30 days)
        new backup.BackupPlanRule({
          ruleName: 'DailyBackups',
          scheduleExpression: events.Schedule.cron({ hour: '2', minute: '0' }),
          deleteAfter: cdk.Duration.days(30),
          copyActions: [{
            destinationBackupVault: drBackupVault,
            deleteAfter: cdk.Duration.days(30)
          }]
        }),
        // Weekly backups (retained for 1 year)
        new backup.BackupPlanRule({
          ruleName: 'WeeklyBackups',
          scheduleExpression: events.Schedule.cron({ weekDay: 'SUN', hour: '3', minute: '0' }),
          deleteAfter: cdk.Duration.days(365),
          copyActions: [{
            destinationBackupVault: drBackupVault,
            deleteAfter: cdk.Duration.days(365)
          }]
        })
      ]
    });

    // Backup selection
    backupPlan.addSelection('Resources', {
      resources: [
        backup.BackupResource.fromRdsDatabaseInstance(props.database),
        backup.BackupResource.fromDynamoDbTable(props.table),
        backup.BackupResource.fromEfsFileSystem(props.fileSystem)
      ]
    });

    // ============================================
    // AUTOMATED FAILOVER LAMBDA
    // ============================================
    const failoverLambda = new lambda.Function(this, 'FailoverLambda', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/failover'),
      timeout: cdk.Duration.minutes(15),
      environment: {
        PRIMARY_REGION: 'us-east-1',
        DR_REGION: 'us-west-2',
        ECS_CLUSTER: props.ecsCluster.clusterName,
        RDS_INSTANCE: props.database.instanceIdentifier
      }
    });

    // Grant permissions
    failoverLambda.addToRolePolicy(new iam.PolicyStatement({
      actions: [
        'rds:PromoteReadReplica',
        'rds:ModifyDBInstance',
        'ecs:UpdateService',
        'route53:ChangeResourceRecordSets',
        'sns:Publish'
      ],
      resources: ['*']
    }));

    // CloudWatch alarm for automated failover
    const primaryDownAlarm = new cloudwatch.Alarm(this, 'PrimaryDownAlarm', {
      metric: new cloudwatch.Metric({
        namespace: 'AWS/ApplicationELB',
        metricName: 'UnHealthyHostCount',
        statistic: 'Average',
        period: cdk.Duration.minutes(1)
      }),
      threshold: 2,
      evaluationPeriods: 5,
      datapointsToAlarm: 5
    });

    primaryDownAlarm.addAlarmAction(
      new cw_actions.LambdaAction(failoverLambda)
    );
  }
}
```

### 🚨 Failover Lambda Function

```javascript
// lambda/failover/index.js
const { RDSClient, PromoteReadReplicaCommand } = require('@aws-sdk/client-rds');
const { ECSClient, UpdateServiceCommand } = require('@aws-sdk/client-ecs');
const { Route53Client, ChangeResourceRecordSetsCommand } = require('@aws-sdk/client-route-53');
const { SNSClient, PublishCommand } = require('@aws-sdk/client-sns');

const rds = new RDSClient({ region: process.env.DR_REGION });
const ecs = new ECSClient({ region: process.env.DR_REGION });
const route53 = new Route53Client({});
const sns = new SNSClient({});

exports.handler = async (event) => {
  console.log('DR Failover initiated:', JSON.stringify(event));

  try {
    // Step 1: Promote RDS Read Replica
    console.log('Promoting RDS read replica...');
    await rds.send(new PromoteReadReplicaCommand({
      DBInstanceIdentifier: `${process.env.RDS_INSTANCE}-dr`
    }));

    // Wait for promotion
    await waitForDBAvailable(`${process.env.RDS_INSTANCE}-dr`);

    // Step 2: Scale up DR ECS services
    console.log('Scaling up DR ECS services...');
    await ecs.send(new UpdateServiceCommand({
      cluster: `${process.env.ECS_CLUSTER}-dr`,
      service: 'account-service',
      desiredCount: 3
    }));

    await ecs.send(new UpdateServiceCommand({
      cluster: `${process.env.ECS_CLUSTER}-dr`,
      service: 'transaction-service',
      desiredCount: 3
    }));

    // Step 3: Update Route 53 to point to DR region
    console.log('Updating Route 53...');
    await route53.send(new ChangeResourceRecordSetsCommand({
      HostedZoneId: process.env.HOSTED_ZONE_ID,
      ChangeBatch: {
        Changes: [{
          Action: 'UPSERT',
          ResourceRecordSet: {
            Name: 'api.emiratesnbd.com',
            Type: 'A',
            AliasTarget: {
              HostedZoneId: process.env.DR_ALB_HOSTED_ZONE,
              DNSName: process.env.DR_ALB_DNS,
              EvaluateTargetHealth: true
            }
          }
        }]
      }
    }));

    // Step 4: Send notifications
    await sns.send(new PublishCommand({
      TopicArn: process.env.ALERT_TOPIC,
      Subject: 'DR Failover Completed',
      Message: JSON.stringify({
        event: 'DR_FAILOVER_SUCCESS',
        timestamp: new Date().toISOString(),
        region: process.env.DR_REGION,
        rto: calculateRTO(event)
      })
    }));

    return {
      statusCode: 200,
      body: 'Failover completed successfully'
    };

  } catch (error) {
    console.error('Failover failed:', error);
    
    await sns.send(new PublishCommand({
      TopicArn: process.env.ALERT_TOPIC,
      Subject: 'DR Failover Failed',
      Message: `Failover failed: ${error.message}`
    }));

    throw error;
  }
};

async function waitForDBAvailable(dbInstanceId) {
  const maxAttempts = 30;
  let attempts = 0;
  
  while (attempts < maxAttempts) {
    const response = await rds.send(new DescribeDBInstancesCommand({
      DBInstanceIdentifier: dbInstanceId
    }));
    
    const status = response.DBInstances[0].DBInstanceStatus;
    if (status === 'available') {
      return;
    }
    
    await new Promise(resolve => setTimeout(resolve, 30000)); // Wait 30s
    attempts++;
  }
  
  throw new Error('Database did not become available in time');
}

function calculateRTO(event) {
  const startTime = new Date(event.time);
  const endTime = new Date();
  return Math.round((endTime - startTime) / 1000 / 60); // Minutes
}
```

### 📋 DR Testing Procedure

```markdown
# DR Testing Runbook

## Pre-Test Checklist
- [ ] Notify all stakeholders
- [ ] Verify backups are current
- [ ] Document current system state
- [ ] Prepare rollback plan

## Test Procedure

### 1. Simulate Primary Region Failure
```bash
# Stop primary ECS services
aws ecs update-service \
  --cluster banking-cluster \
  --service account-service \
  --desired-count 0 \
  --region us-east-1

# Update Route 53 health check to fail
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch file://failover-dns.json
```

### 2. Verify Automated Failover
- Monitor CloudWatch alarms
- Confirm failover Lambda execution
- Verify RDS read replica promotion
- Check ECS service scaling in DR region
- Test application endpoints

### 3. Validate DR Environment
```bash
# Test API endpoints
curl -f https://api.emiratesnbd.com/health

# Verify data consistency
aws dynamodb scan \
  --table-name banking-transactions-global \
  --region us-west-2 \
  --limit 10

# Check database connectivity
psql -h dr-db.amazonaws.com -U admin -d banking
```

### 4. Failback to Primary
```bash
# Re-sync data from DR to Primary
# Scale down DR services
# Update Route 53 to primary
# Restore normal operations
```

## Post-Test
- [ ] Document lessons learned
- [ ] Update RTO/RPO measurements
- [ ] Review and improve automation
- [ ] Schedule next DR test
```

### 🎓 Interview Discussion Points - Q29

**Q1: What is RTO vs RPO?**

**A**:
- **RTO**: Recovery Time Objective (how long to recover)
- **RPO**: Recovery Point Objective (how much data loss)
- Example: RTO 15 min, RPO 5 min

**Q2: What are DR strategies?**

**A**:
- **Backup & Restore**: Cheapest, slowest (RTO hours)
- **Pilot Light**: Minimal DR, scale up (RTO minutes)
- **Warm Standby**: Scaled-down DR (RTO minutes)
- **Multi-Site**: Active-Active (RTO seconds)

---

## Question 30: Backup & Recovery Automation

### 📋 Question Statement

Implement automated backup and recovery procedures for Emirates NBD including point-in-time recovery, automated testing of backups, and compliance with retention policies.

---

### 🔄 AWS Backup Automation Stack

```typescript
// infrastructure/cdk/backup-automation-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as backup from 'aws-cdk-lib/aws-backup';
import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as sns from 'aws-cdk-lib/aws-sns';

export class BackupAutomationStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props: any) {
    super(scope, id, props);

    // ============================================
    // BACKUP VAULTS (Primary and DR)
    // ============================================
    const primaryVault = new backup.BackupVault(this, 'PrimaryVault', {
      backupVaultName: 'banking-primary-vault',
      encryptionKey: props.kmsKey,
      notificationTopic: new sns.Topic(this, 'BackupNotifications', {
        displayName: 'Backup Notifications'
      })
    });

    const drVault = new backup.BackupVault(this, 'DRVault', {
      backupVaultName: 'banking-dr-vault',
      encryptionKey: props.drKmsKey,
      removalPolicy: cdk.RemovalPolicy.RETAIN
    });

    // ============================================
    // COMPREHENSIVE BACKUP PLAN
    // ============================================
    const backupPlan = new backup.BackupPlan(this, 'ComprehensiveBackup', {
      backupPlanName: 'banking-comprehensive-backup',
      backupVault: primaryVault,
      backupPlanRules: [
        // Continuous backup (for DynamoDB & RDS)
        new backup.BackupPlanRule({
          ruleName: 'ContinuousBackup',
          enableContinuousBackup: true,
          deleteAfter: cdk.Duration.days(35)
        }),
        
        // Hourly snapshots (critical databases)
        new backup.BackupPlanRule({
          ruleName: 'HourlySnapshots',
          scheduleExpression: events.Schedule.cron({ minute: '0' }),
          startWindow: cdk.Duration.minutes(60),
          completionWindow: cdk.Duration.hours(2),
          deleteAfter: cdk.Duration.days(1),
          copyActions: [{
            destinationBackupVault: drVault,
            deleteAfter: cdk.Duration.days(1)
          }]
        }),
        
        // Daily backups (all resources)
        new backup.BackupPlanRule({
          ruleName: 'DailyBackup',
          scheduleExpression: events.Schedule.cron({ hour: '2', minute: '0' }),
          deleteAfter: cdk.Duration.days(30),
          moveToColdStorageAfter: cdk.Duration.days(7),
          copyActions: [{
            destinationBackupVault: drVault,
            deleteAfter: cdk.Duration.days(30),
            moveToColdStorageAfter: cdk.Duration.days(7)
          }]
        }),
        
        // Monthly backups (long-term retention)
        new backup.BackupPlanRule({
          ruleName: 'MonthlyBackup',
          scheduleExpression: events.Schedule.cron({ 
            day: '1', 
            hour: '3', 
            minute: '0' 
          }),
          deleteAfter: cdk.Duration.days(2555), // 7 years
          moveToColdStorageAfter: cdk.Duration.days(90),
          copyActions: [{
            destinationBackupVault: drVault,
            deleteAfter: cdk.Duration.days(2555),
            moveToColdStorageAfter: cdk.Duration.days(90)
          }]
        })
      ]
    });

    // ============================================
    // BACKUP SELECTIONS (Resources to Backup)
    // ============================================
    
    // Critical databases
    backupPlan.addSelection('CriticalDatabases', {
      resources: [
        backup.BackupResource.fromRdsDatabaseInstance(props.accountDb),
        backup.BackupResource.fromRdsDatabaseInstance(props.transactionDb),
        backup.BackupResource.fromDynamoDbTable(props.transactionTable),
        backup.BackupResource.fromDynamoDbTable(props.customerTable)
      ],
      allowRestores: true
    });

    // File systems and storage
    backupPlan.addSelection('Storage', {
      resources: [
        backup.BackupResource.fromEfsFileSystem(props.efs),
        backup.BackupResource.fromConstruct(props.documentBucket)
      ],
      allowRestores: true
    });

    // Tag-based selection (backup all tagged resources)
    backupPlan.addSelection('TaggedResources', {
      resources: [
        backup.BackupResource.fromTag('Backup', 'true'),
        backup.BackupResource.fromTag('Environment', 'production')
      ],
      allowRestores: true
    });

    // ============================================
    // AUTOMATED BACKUP TESTING LAMBDA
    // ============================================
    const testBackupLambda = new lambda.Function(this, 'TestBackupLambda', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/test-backup'),
      timeout: cdk.Duration.minutes(15),
      memorySize: 512,
      environment: {
        PRIMARY_VAULT: primaryVault.backupVaultName,
        DR_VAULT: drVault.backupVaultName,
        NOTIFICATION_TOPIC: props.alertTopic.topicArn
      }
    });

    // Grant permissions
    testBackupLambda.addToRolePolicy(new iam.PolicyStatement({
      actions: [
        'backup:ListRecoveryPointsByBackupVault',
        'backup:DescribeRecoveryPoint',
        'backup:StartRestoreJob',
        'backup:DescribeRestoreJob',
        'rds:DescribeDBInstances',
        'rds:DeleteDBInstance',
        'dynamodb:DescribeTable',
        'dynamodb:DeleteTable',
        'cloudwatch:PutMetricData'
      ],
      resources: ['*']
    }));

    props.alertTopic.grantPublish(testBackupLambda);

    // Schedule weekly backup tests
    const testRule = new events.Rule(this, 'WeeklyBackupTest', {
      schedule: events.Schedule.cron({ 
        weekDay: 'SAT', 
        hour: '4', 
        minute: '0' 
      })
    });

    testRule.addTarget(new targets.LambdaFunction(testBackupLambda));

    // ============================================
    // BACKUP LIFECYCLE MANAGER
    // ============================================
    const lifecycleLambda = new lambda.Function(this, 'LifecycleLambda', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/backup-lifecycle'),
      timeout: cdk.Duration.minutes(5),
      environment: {
        VAULT_NAME: primaryVault.backupVaultName,
        RETENTION_DAYS: '30',
        COLD_STORAGE_DAYS: '7'
      }
    });

    lifecycleLambda.addToRolePolicy(new iam.PolicyStatement({
      actions: [
        'backup:ListRecoveryPointsByBackupVault',
        'backup:DeleteRecoveryPoint',
        'backup:UpdateRecoveryPointLifecycle'
      ],
      resources: ['*']
    }));

    // Daily lifecycle check
    new events.Rule(this, 'DailyLifecycle', {
      schedule: events.Schedule.rate(cdk.Duration.days(1)),
      targets: [new targets.LambdaFunction(lifecycleLambda)]
    });
  }
}
```

### 🧪 Automated Backup Testing Lambda

```typescript
// lambda/test-backup/index.ts
import { 
  BackupClient, 
  ListRecoveryPointsByBackupVaultCommand,
  StartRestoreJobCommand,
  DescribeRestoreJobCommand 
} from '@aws-sdk/client-backup';
import { 
  RDSClient, 
  DescribeDBInstancesCommand,
  DeleteDBInstanceCommand 
} from '@aws-sdk/client-rds';
import { SNSClient, PublishCommand } from '@aws-sdk/client-sns';
import { CloudWatchClient, PutMetricDataCommand } from '@aws-sdk/client-cloudwatch';

const backup = new BackupClient({});
const rds = new RDSClient({});
const sns = new SNSClient({});
const cloudwatch = new CloudWatchClient({});

interface BackupTestResult {
  recoveryPointArn: string;
  resourceType: string;
  success: boolean;
  duration: number;
  error?: string;
}

export const handler = async (event: any) => {
  console.log('Starting automated backup test');
  
  const results: BackupTestResult[] = [];

  try {
    // Get latest recovery points
    const recoveryPoints = await getLatestRecoveryPoints();
    
    // Test each recovery point
    for (const point of recoveryPoints) {
      const result = await testRecoveryPoint(point);
      results.push(result);
      
      // Publish metrics
      await publishMetrics(result);
    }

    // Generate report
    const report = generateReport(results);
    
    // Send notification
    await sendNotification(report);

    return {
      statusCode: 200,
      body: JSON.stringify(report)
    };

  } catch (error) {
    console.error('Backup test failed:', error);
    
    await sns.send(new PublishCommand({
      TopicArn: process.env.NOTIFICATION_TOPIC,
      Subject: 'Backup Test Failed',
      Message: `Backup test encountered an error: ${(error as Error).message}`
    }));

    throw error;
  }
};

async function getLatestRecoveryPoints() {
  const response = await backup.send(
    new ListRecoveryPointsByBackupVaultCommand({
      BackupVaultName: process.env.PRIMARY_VAULT,
      MaxResults: 5
    })
  );

  return response.RecoveryPoints || [];
}

async function testRecoveryPoint(point: any): Promise<BackupTestResult> {
  const startTime = Date.now();
  
  try {
    console.log(`Testing recovery point: ${point.RecoveryPointArn}`);
    
    // Determine resource type
    const resourceType = point.ResourceType;
    
    // Start restore job to test instance
    const restoreJobId = await startTestRestore(point);
    
    // Wait for restore to complete
    await waitForRestore(restoreJobId);
    
    // Validate restored resource
    await validateRestore(restoreJobId, resourceType);
    
    // Cleanup test resource
    await cleanupTestResource(restoreJobId, resourceType);
    
    const duration = Date.now() - startTime;
    
    return {
      recoveryPointArn: point.RecoveryPointArn,
      resourceType,
      success: true,
      duration
    };

  } catch (error) {
    return {
      recoveryPointArn: point.RecoveryPointArn,
      resourceType: point.ResourceType,
      success: false,
      duration: Date.now() - startTime,
      error: (error as Error).message
    };
  }
}

async function startTestRestore(point: any): Promise<string> {
  const metadata: Record<string, string> = {};
  
  // Configure test restore based on resource type
  if (point.ResourceType === 'RDS') {
    metadata.DBInstanceIdentifier = `test-restore-${Date.now()}`;
    metadata.DBInstanceClass = 'db.t3.micro'; // Use small instance for testing
    metadata.PubliclyAccessible = 'false';
  }

  const response = await backup.send(
    new StartRestoreJobCommand({
      RecoveryPointArn: point.RecoveryPointArn,
      Metadata: metadata,
      IamRoleArn: process.env.RESTORE_ROLE_ARN
    })
  );

  return response.RestoreJobId!;
}

async function waitForRestore(restoreJobId: string, maxWaitMinutes: number = 30) {
  const maxAttempts = maxWaitMinutes * 2; // Check every 30 seconds
  let attempts = 0;

  while (attempts < maxAttempts) {
    const response = await backup.send(
      new DescribeRestoreJobCommand({ RestoreJobId: restoreJobId })
    );

    const status = response.Status;
    
    if (status === 'COMPLETED') {
      console.log('Restore completed successfully');
      return;
    }
    
    if (status === 'FAILED' || status === 'ABORTED') {
      throw new Error(`Restore failed with status: ${status}`);
    }

    await new Promise(resolve => setTimeout(resolve, 30000)); // Wait 30s
    attempts++;
  }

  throw new Error('Restore timeout');
}

async function validateRestore(restoreJobId: string, resourceType: string) {
  if (resourceType === 'RDS') {
    // Get restored DB instance
    const dbInstanceId = await getRestoredDbInstance(restoreJobId);
    
    const response = await rds.send(
      new DescribeDBInstancesCommand({
        DBInstanceIdentifier: dbInstanceId
      })
    );

    const instance = response.DBInstances?.[0];
    
    if (!instance || instance.DBInstanceStatus !== 'available') {
      throw new Error('Restored database is not available');
    }

    console.log(`Validated restored DB: ${dbInstanceId}`);
  }
  
  // Add validation for other resource types (DynamoDB, EFS, etc.)
}

async function cleanupTestResource(restoreJobId: string, resourceType: string) {
  if (resourceType === 'RDS') {
    const dbInstanceId = await getRestoredDbInstance(restoreJobId);
    
    await rds.send(
      new DeleteDBInstanceCommand({
        DBInstanceIdentifier: dbInstanceId,
        SkipFinalSnapshot: true,
        DeleteAutomatedBackups: true
      })
    );

    console.log(`Cleaned up test DB: ${dbInstanceId}`);
  }
}

async function getRestoredDbInstance(restoreJobId: string): Promise<string> {
  const response = await backup.send(
    new DescribeRestoreJobCommand({ RestoreJobId: restoreJobId })
  );

  const arn = response.CreatedResourceArn!;
  return arn.split(':')[6]; // Extract DB instance ID from ARN
}

async function publishMetrics(result: BackupTestResult) {
  await cloudwatch.send(
    new PutMetricDataCommand({
      Namespace: 'Banking/Backup',
      MetricData: [
        {
          MetricName: 'BackupTestSuccess',
          Value: result.success ? 1 : 0,
          Unit: 'Count',
          Dimensions: [
            { Name: 'ResourceType', Value: result.resourceType }
          ]
        },
        {
          MetricName: 'RestoreDuration',
          Value: result.duration / 1000 / 60, // Convert to minutes
          Unit: 'Seconds',
          Dimensions: [
            { Name: 'ResourceType', Value: result.resourceType }
          ]
        }
      ]
    })
  );
}

function generateReport(results: BackupTestResult[]) {
  const total = results.length;
  const successful = results.filter(r => r.success).length;
  const failed = results.filter(r => !r.success);

  return {
    timestamp: new Date().toISOString(),
    total,
    successful,
    failed: failed.length,
    successRate: (successful / total * 100).toFixed(2) + '%',
    averageRestoreTime: calculateAverage(results.map(r => r.duration)),
    failedTests: failed.map(r => ({
      recoveryPoint: r.recoveryPointArn,
      error: r.error
    }))
  };
}

function calculateAverage(values: number[]): string {
  const avg = values.reduce((a, b) => a + b, 0) / values.length;
  return (avg / 1000 / 60).toFixed(2) + ' minutes';
}

async function sendNotification(report: any) {
  const message = `
Backup Test Report
==================
Timestamp: ${report.timestamp}
Total Tests: ${report.total}
Successful: ${report.successful}
Failed: ${report.failed}
Success Rate: ${report.successRate}
Average Restore Time: ${report.averageRestoreTime}

${report.failedTests.length > 0 ? `
Failed Tests:
${report.failedTests.map((t: any) => `- ${t.recoveryPoint}: ${t.error}`).join('\n')}
` : ''}
`;

  await sns.send(
    new PublishCommand({
      TopicArn: process.env.NOTIFICATION_TOPIC,
      Subject: `Backup Test Report - ${report.successRate} Success`,
      Message: message
    })
  );
}
```

### 📊 Point-in-Time Recovery (PITR)

```typescript
// services/recovery/pitr-service.ts
import { BackupClient, StartRestoreJobCommand } from '@aws-sdk/client-backup';
import { RDSClient, RestoreDBInstanceToPointInTimeCommand } from '@aws-sdk/client-rds';
import { DynamoDBClient, RestoreTableToPointInTimeCommand } from '@aws-sdk/client-dynamodb';

export class PointInTimeRecoveryService {
  private backup: BackupClient;
  private rds: RDSClient;
  private dynamodb: DynamoDBClient;

  constructor() {
    this.backup = new BackupClient({});
    this.rds = new RDSClient({});
    this.dynamodb = new DynamoDBClient({});
  }

  async restoreRDSToPointInTime(params: {
    sourceDbInstance: string;
    targetDbInstance: string;
    restoreTime: Date;
  }) {
    console.log(`Restoring ${params.sourceDbInstance} to ${params.restoreTime}`);

    const response = await this.rds.send(
      new RestoreDBInstanceToPointInTimeCommand({
        SourceDBInstanceIdentifier: params.sourceDbInstance,
        TargetDBInstanceIdentifier: params.targetDbInstance,
        RestoreTime: params.restoreTime,
        UseLatestRestorableTime: false,
        DBInstanceClass: 'db.r5.large',
        MultiAZ: true,
        PubliclyAccessible: false
      })
    );

    return response.DBInstance;
  }

  async restoreDynamoDBToPointInTime(params: {
    sourceTable: string;
    targetTable: string;
    restoreTime: Date;
  }) {
    console.log(`Restoring DynamoDB ${params.sourceTable} to ${params.restoreTime}`);

    const response = await this.dynamodb.send(
      new RestoreTableToPointInTimeCommand({
        SourceTableName: params.sourceTable,
        TargetTableName: params.targetTable,
        RestoreDateTime: params.restoreTime,
        UseLatestRestorableTime: false
      })
    );

    return response.TableDescription;
  }

  async validateRecoveryPointObjective(resourceArn: string): Promise<{
    rpoMet: boolean;
    lastBackupAge: number;
    targetRPO: number;
  }> {
    // Get latest recovery point
    const recoveryPoints = await this.backup.send(
      new ListRecoveryPointsByResourceCommand({
        ResourceArn: resourceArn,
        MaxResults: 1
      })
    );

    const latestPoint = recoveryPoints.RecoveryPoints?.[0];
    
    if (!latestPoint) {
      throw new Error('No recovery points found');
    }

    const lastBackupAge = Date.now() - latestPoint.CreationDate!.getTime();
    const targetRPO = 5 * 60 * 1000; // 5 minutes in ms

    return {
      rpoMet: lastBackupAge <= targetRPO,
      lastBackupAge: lastBackupAge / 1000 / 60, // Convert to minutes
      targetRPO: targetRPO / 1000 / 60
    };
  }
}
```

### 🎓 Interview Discussion Points - Q30

**Q1: How often should you test DR and backups?**

**A**:
- **Quarterly**: Full DR test with complete failover
- **Monthly**: Backup restoration test (automated)
- **Weekly**: Automated backup validation
- **Daily**: Backup completion monitoring
- **Continuous**: RPO/RTO compliance checks

**Q2: What should be backed up in a banking application?**

**A**:
- **Databases**: RDS (transactions, customers, accounts) with continuous backup
- **Files**: S3 (documents, reports) with versioning + cross-region replication
- **Configuration**: Infrastructure as Code (CDK), environment configs
- **Secrets**: AWS Secrets Manager with automatic rotation
- **Logs**: CloudWatch Logs with retention policies
- **State**: DynamoDB tables with PITR enabled

**Q3: What is the difference between snapshot and continuous backup?**

**A**:
- **Snapshot**: Point-in-time copy, taken periodically (hourly/daily)
  - Use case: Daily backups, long-term retention
  - RPO: Hours to days
- **Continuous Backup**: Tracks every change in real-time
  - Use case: Restore to any second in last 35 days
  - RPO: Seconds (typically 5 seconds for RDS/DynamoDB)

**Q4: How do you ensure backup compliance?**

**A**:
- **Automated backup plans**: AWS Backup with schedules
- **Retention policies**: 30 days hot, 7 years cold storage
- **Cross-region copies**: Protect against regional failure
- **Encryption**: All backups encrypted at rest
- **Automated testing**: Weekly restore tests
- **Audit trails**: AWS CloudTrail for all backup operations
- **Compliance reports**: AWS Backup Audit Manager

**Q5: What is the cost optimization strategy for backups?**

**A**:
- **Tiered storage**: Move old backups to cold storage after 7 days
- **Lifecycle policies**: Delete backups after retention period
- **Incremental backups**: Only backup changed data
- **Compression**: Reduce storage costs
- **Cross-region**: Only for critical data
- **Cost**: Hot storage $0.05/GB/month, Cold $0.01/GB/month

---

**End of File 15**