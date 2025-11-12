# Multi-Region Deployment

## Question 27: Global Multi-Region Architecture

### 📋 Question Statement

Implement a multi-region deployment for Emirates NBD with active-active architecture, Route 53 latency-based routing, DynamoDB global tables, Aurora global database, and cross-region replication.

---

### 🏗️ Multi-Region Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│           EMIRATES NBD MULTI-REGION ARCHITECTURE                            │
└────────────────────────────────────────────────────────────────────────────┘

    GLOBAL ROUTING
    ──────────────
    
    ┌──────────────────────────────────┐
    │      Route 53 Global DNS         │
    │  • Latency-based routing         │
    │  • Health checks                 │
    └──────────┬───────────────┬───────┘
               │               │
    ┌──────────▼─────┐  ┌─────▼──────────┐
    │   US-EAST-1    │  │   EU-WEST-1    │
    │   (Primary)    │  │   (Secondary)  │
    │                │  │                │
    │  ┌──────────┐  │  │  ┌──────────┐  │
    │  │    ECS   │  │  │  │    ECS   │  │
    │  │ Services │  │  │  │ Services │  │
    │  └──────────┘  │  │  └──────────┘  │
    │                │  │                │
    │  ┌──────────┐  │  │  ┌──────────┐  │
    │  │ DynamoDB │◄─┼──┼─►│ DynamoDB │  │
    │  │  Global  │  │  │  │  Global  │  │
    │  │  Table   │  │  │  │  Table   │  │
    │  └──────────┘  │  │  └──────────┘  │
    │                │  │                │
    │  ┌──────────┐  │  │  ┌──────────┐  │
    │  │  Aurora  │◄─┼──┼─►│  Aurora  │  │
    │  │  Global  │  │  │  │  Global  │  │
    │  │    DB    │  │  │  │    DB    │  │
    │  └──────────┘  │  │  └──────────┘  │
    └────────────────┘  └────────────────┘
```

---

### 🔧 Multi-Region CDK Stack

```typescript
// infrastructure/cdk/multi-region-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as route53 from 'aws-cdk-lib/aws-route53';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as rds from 'aws-cdk-lib/aws-rds';
import * as s3 from 'aws-cdk-lib/aws-s3';

export class MultiRegionStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // ============================================
    // ROUTE 53 GLOBAL DNS
    // ============================================
    const hostedZone = route53.HostedZone.fromLookup(this, 'Zone', {
      domainName: 'emiratesnbd.com'
    });

    // Health check for primary region
    const healthCheck = new route53.CfnHealthCheck(this, 'HealthCheck', {
      healthCheckConfig: {
        type: 'HTTPS',
        resourcePath: '/health',
        fullyQualifiedDomainName: 'api-us-east-1.emiratesnbd.com',
        port: 443,
        requestInterval: 30,
        failureThreshold: 3
      }
    });

    // Primary region record (us-east-1)
    new route53.ARecord(this, 'PrimaryRecord', {
      zone: hostedZone,
      recordName: 'api',
      target: route53.RecordTarget.fromAlias(
        new targets.LoadBalancerTarget(primaryALB)
      ),
      geoLocation: route53.GeoLocation.continent(route53.Continent.NORTH_AMERICA)
    });

    // Secondary region record (eu-west-1)
    new route53.ARecord(this, 'SecondaryRecord', {
      zone: hostedZone,
      recordName: 'api',
      target: route53.RecordTarget.fromAlias(
        new targets.LoadBalancerTarget(secondaryALB)
      ),
      geoLocation: route53.GeoLocation.continent(route53.Continent.EUROPE)
    });

    // Latency-based routing
    new route53.ARecord(this, 'LatencyUSRecord', {
      zone: hostedZone,
      recordName: 'api',
      target: route53.RecordTarget.fromAlias(
        new targets.LoadBalancerTarget(primaryALB)
      ),
      region: 'us-east-1'
    });

    new route53.ARecord(this, 'LatencyEURecord', {
      zone: hostedZone,
      recordName: 'api',
      target: route53.RecordTarget.fromAlias(
        new targets.LoadBalancerTarget(secondaryALB)
      ),
      region: 'eu-west-1'
    });

    // ============================================
    // DYNAMODB GLOBAL TABLE
    // ============================================
    const globalTable = new dynamodb.Table(this, 'GlobalTable', {
      tableName: 'banking-transactions-global',
      partitionKey: { name: 'transactionId', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'timestamp', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      replicationRegions: ['us-east-1', 'eu-west-1', 'ap-southeast-1'],
      stream: dynamodb.StreamViewType.NEW_AND_OLD_IMAGES,
      pointInTimeRecovery: true
    });

    // ============================================
    // AURORA GLOBAL DATABASE
    // ============================================
    const globalCluster = new rds.DatabaseCluster(this, 'GlobalCluster', {
      engine: rds.DatabaseClusterEngine.auroraPostgres({
        version: rds.AuroraPostgresEngineVersion.VER_13_9
      }),
      instanceProps: {
        vpc: props.vpc,
        instanceType: ec2.InstanceType.of(ec2.InstanceClass.R5, ec2.InstanceSize.LARGE)
      },
      instances: 2,
      storageEncrypted: true
    });

    // Add global cluster identifier
    const cfnGlobalCluster = new rds.CfnGlobalCluster(this, 'CfnGlobalCluster', {
      globalClusterIdentifier: 'banking-global-cluster',
      sourceDbClusterIdentifier: globalCluster.clusterArn,
      engine: 'aurora-postgresql',
      engineVersion: '13.9'
    });

    // ============================================
    // S3 CROSS-REGION REPLICATION
    // ============================================
    const primaryBucket = new s3.Bucket(this, 'PrimaryBucket', {
      bucketName: 'banking-data-us-east-1',
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED
    });

    const replicaBucket = new s3.Bucket(this, 'ReplicaBucket', {
      bucketName: 'banking-data-eu-west-1',
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED
    });

    // Replication configuration
    const replicationRole = new iam.Role(this, 'ReplicationRole', {
      assumedBy: new iam.ServicePrincipal('s3.amazonaws.com')
    });

    primaryBucket.addToResourcePolicy(new iam.PolicyStatement({
      effect: iam.Effect.ALLOW,
      principals: [replicationRole],
      actions: ['s3:ReplicateObject', 's3:ReplicateDelete'],
      resources: [`${primaryBucket.bucketArn}/*`]
    }));

    new s3.CfnBucket(this, 'ReplicationConfig', {
      bucket: primaryBucket.bucketName,
      replicationConfiguration: {
        role: replicationRole.roleArn,
        rules: [{
          id: 'ReplicateAll',
          status: 'Enabled',
          priority: 1,
          filter: {},
          destination: {
            bucket: replicaBucket.bucketArn,
            replicationTime: {
              status: 'Enabled',
              time: { minutes: 15 }
            },
            metrics: {
              status: 'Enabled',
              eventThreshold: { minutes: 15 }
            }
          }
        }]
      }
    });
  }
}
```

### 🎓 Interview Discussion Points - Q27

**Q1: Active-Active vs Active-Passive?**

**A**:
- **Active-Active**: Both regions serve traffic
- **Active-Passive**: One region on standby
- **Trade-offs**: Cost vs availability

**Q2: How to handle data conflicts in multi-region?**

**A**:
- **Last-write-wins**: DynamoDB default
- **Version vectors**: Track causality
- **CRDT**: Conflict-free data types
- **Application logic**: Business rules

---

## Question 28: Data Synchronization

### 📋 Question Statement

Implement real-time data synchronization across regions using DynamoDB Streams, EventBridge, and custom replication logic. Handle conflict resolution and eventual consistency.

---

### 🔄 DynamoDB Streams Cross-Region Sync

```typescript
// services/data-sync/stream-processor.ts
import { DynamoDBStreamEvent, DynamoDBRecord } from 'aws-lambda';
import { DynamoDBClient, PutItemCommand, UpdateItemCommand } from '@aws-sdk/client-dynamodb';
import { EventBridgeClient, PutEventsCommand } from '@aws-sdk/client-eventbridge';

interface ConflictResolution {
  strategy: 'last-write-wins' | 'version-vector' | 'custom';
  resolve: (local: any, remote: any) => any;
}

export class CrossRegionSyncHandler {
  private dynamoClients: Map<string, DynamoDBClient>;
  private eventBridge: EventBridgeClient;

  constructor() {
    // Initialize clients for each region
    this.dynamoClients = new Map([
      ['us-east-1', new DynamoDBClient({ region: 'us-east-1' })],
      ['eu-west-1', new DynamoDBClient({ region: 'eu-west-1' })],
      ['ap-southeast-1', new DynamoDBClient({ region: 'ap-southeast-1' })]
    ]);
    
    this.eventBridge = new EventBridgeClient({ region: process.env.AWS_REGION });
  }

  async handleStreamEvent(event: DynamoDBStreamEvent) {
    for (const record of event.Records) {
      await this.processRecord(record);
    }
  }

  private async processRecord(record: DynamoDBRecord) {
    const eventType = record.eventName; // INSERT, MODIFY, REMOVE
    const tableName = record.eventSourceARN?.split('/')[1];
    
    // Extract source region from ARN
    const sourceRegion = record.eventSourceARN?.split(':')[3];
    
    // Get new and old images
    const newImage = record.dynamodb?.NewImage;
    const oldImage = record.dynamodb?.OldImage;

    console.log(`Processing ${eventType} from ${sourceRegion} for table ${tableName}`);

    // Publish to EventBridge for fan-out to other regions
    await this.publishCrossRegionEvent({
      eventType,
      tableName,
      sourceRegion,
      newImage,
      oldImage,
      timestamp: Date.now()
    });

    // Apply conflict resolution if needed
    if (eventType === 'MODIFY') {
      await this.handleConflict(tableName!, newImage!, oldImage);
    }
  }

  private async publishCrossRegionEvent(data: any) {
    const event = {
      Source: 'banking.data-sync',
      DetailType: 'Cross-Region Data Change',
      Detail: JSON.stringify(data),
      EventBusName: 'banking-global-bus'
    };

    await this.eventBridge.send(new PutEventsCommand({
      Entries: [event]
    }));
  }

  private async handleConflict(
    tableName: string,
    newImage: any,
    oldImage: any
  ) {
    const strategy = this.getConflictStrategy(tableName);
    
    switch (strategy.strategy) {
      case 'last-write-wins':
        return this.lastWriteWins(newImage);
      
      case 'version-vector':
        return this.versionVectorResolution(newImage, oldImage);
      
      case 'custom':
        return strategy.resolve(newImage, oldImage);
    }
  }

  private lastWriteWins(newImage: any): any {
    // Simple: newest timestamp wins
    return newImage;
  }

  private versionVectorResolution(newImage: any, oldImage: any): any {
    // Extract version vectors
    const newVector = this.parseVersionVector(newImage.versionVector?.S || '{}');
    const oldVector = this.parseVersionVector(oldImage?.versionVector?.S || '{}');

    // Compare vectors
    if (this.isNewer(newVector, oldVector)) {
      return newImage;
    }

    // Conflict detected - merge
    return this.mergeVersions(newImage, oldImage, newVector, oldVector);
  }

  private parseVersionVector(vectorString: string): Record<string, number> {
    try {
      return JSON.parse(vectorString);
    } catch {
      return {};
    }
  }

  private isNewer(v1: Record<string, number>, v2: Record<string, number>): boolean {
    // v1 is newer if all its counters >= v2's counters
    const allRegions = new Set([...Object.keys(v1), ...Object.keys(v2)]);
    
    for (const region of allRegions) {
      if ((v1[region] || 0) < (v2[region] || 0)) {
        return false;
      }
    }
    
    return true;
  }

  private mergeVersions(
    newImage: any,
    oldImage: any,
    newVector: Record<string, number>,
    oldVector: Record<string, number>
  ): any {
    // Merge strategy: combine both versions with metadata
    const merged = {
      ...newImage,
      _conflict: true,
      _versions: [
        { data: newImage, vector: newVector },
        { data: oldImage, vector: oldVector }
      ],
      _resolvedAt: Date.now()
    };

    return merged;
  }

  private getConflictStrategy(tableName: string): ConflictResolution {
    // Define strategies per table
    const strategies: Record<string, ConflictResolution> = {
      'banking-transactions': {
        strategy: 'last-write-wins',
        resolve: (local, remote) => local
      },
      'customer-profiles': {
        strategy: 'version-vector',
        resolve: (local, remote) => this.mergeCustomerProfiles(local, remote)
      },
      'account-balances': {
        strategy: 'custom',
        resolve: (local, remote) => this.resolveAccountBalance(local, remote)
      }
    };

    return strategies[tableName] || strategies['banking-transactions'];
  }

  private mergeCustomerProfiles(local: any, remote: any): any {
    // Merge non-conflicting fields
    return {
      ...local,
      email: local.updatedAt?.N > remote.updatedAt?.N ? local.email : remote.email,
      phone: local.updatedAt?.N > remote.updatedAt?.N ? local.phone : remote.phone,
      address: remote.address || local.address
    };
  }

  private resolveAccountBalance(local: any, remote: any): any {
    // Account balance requires special handling
    // Use the highest balance to avoid data loss
    const localBalance = parseFloat(local.balance?.N || '0');
    const remoteBalance = parseFloat(remote.balance?.N || '0');

    return {
      ...local,
      balance: { N: Math.max(localBalance, remoteBalance).toString() },
      _conflictResolved: true,
      _resolutionStrategy: 'max-balance'
    };
  }
}

// Lambda handler
export const handler = async (event: DynamoDBStreamEvent) => {
  const syncHandler = new CrossRegionSyncHandler();
  await syncHandler.handleStreamEvent(event);
};
```

### 📡 EventBridge Cross-Region Bus

```typescript
// infrastructure/cdk/cross-region-eventbridge-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as iam from 'aws-cdk-lib/aws-iam';

export class CrossRegionEventBridgeStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props: any) {
    super(scope, id, props);

    // Global event bus
    const globalBus = new events.EventBus(this, 'GlobalBus', {
      eventBusName: 'banking-global-bus'
    });

    // Cross-region rule (receives events from other regions)
    const crossRegionRule = new events.Rule(this, 'CrossRegionRule', {
      eventBus: globalBus,
      eventPattern: {
        source: ['banking.data-sync'],
        detailType: ['Cross-Region Data Change']
      }
    });

    // Sync handler Lambda
    const syncHandler = new lambda.Function(this, 'SyncHandler', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('dist/sync-handler'),
      environment: {
        TARGET_REGIONS: 'us-east-1,eu-west-1,ap-southeast-1',
        CURRENT_REGION: process.env.CDK_DEFAULT_REGION!
      },
      timeout: cdk.Duration.seconds(30)
    });

    // Grant permissions to write to DynamoDB in all regions
    syncHandler.addToRolePolicy(new iam.PolicyStatement({
      actions: [
        'dynamodb:PutItem',
        'dynamodb:UpdateItem',
        'dynamodb:GetItem'
      ],
      resources: [
        `arn:aws:dynamodb:us-east-1:${this.account}:table/*`,
        `arn:aws:dynamodb:eu-west-1:${this.account}:table/*`,
        `arn:aws:dynamodb:ap-southeast-1:${this.account}:table/*`
      ]
    }));

    // Add Lambda as target
    crossRegionRule.addTarget(new targets.LambdaFunction(syncHandler));

    // Cross-region event bus target
    const euWestBus = events.EventBus.fromEventBusArn(
      this,
      'EUBus',
      `arn:aws:events:eu-west-1:${this.account}:event-bus/banking-global-bus`
    );

    const crossRegionTarget = new events.CfnRule.TargetProperty({
      arn: euWestBus.eventBusArn,
      id: 'CrossRegionBus',
      roleArn: this.createCrossRegionRole().roleArn
    });
  }

  private createCrossRegionRole(): iam.Role {
    const role = new iam.Role(this, 'CrossRegionRole', {
      assumedBy: new iam.ServicePrincipal('events.amazonaws.com')
    });

    role.addToPolicy(new iam.PolicyStatement({
      actions: ['events:PutEvents'],
      resources: [
        `arn:aws:events:eu-west-1:${this.account}:event-bus/banking-global-bus`,
        `arn:aws:events:ap-southeast-1:${this.account}:event-bus/banking-global-bus`
      ]
    }));

    return role;
  }
}
```

### 🔍 Eventual Consistency Monitor

```typescript
// services/monitoring/consistency-monitor.ts
import { DynamoDBClient, GetItemCommand } from '@aws-sdk/client-dynamodb';
import { CloudWatchClient, PutMetricDataCommand } from '@aws-sdk/client-cloudwatch';

export class ConsistencyMonitor {
  private clients: Map<string, DynamoDBClient>;
  private cloudwatch: CloudWatchClient;

  constructor() {
    this.clients = new Map([
      ['us-east-1', new DynamoDBClient({ region: 'us-east-1' })],
      ['eu-west-1', new DynamoDBClient({ region: 'eu-west-1' })],
      ['ap-southeast-1', new DynamoDBClient({ region: 'ap-southeast-1' })]
    ]);

    this.cloudwatch = new CloudWatchClient({ region: process.env.AWS_REGION });
  }

  async checkConsistency(tableName: string, key: any) {
    const startTime = Date.now();
    
    // Read same item from all regions
    const results = await Promise.all(
      Array.from(this.clients.entries()).map(([region, client]) =>
        this.getItem(client, tableName, key, region)
      )
    );

    // Compare results
    const consistent = this.areConsistent(results.map(r => r.item));
    const lagMs = Date.now() - startTime;

    // Publish metrics
    await this.publishMetrics({
      tableName,
      consistent,
      lagMs,
      regions: results.length
    });

    return {
      consistent,
      lagMs,
      results
    };
  }

  private async getItem(
    client: DynamoDBClient,
    tableName: string,
    key: any,
    region: string
  ) {
    try {
      const result = await client.send(new GetItemCommand({
        TableName: tableName,
        Key: key,
        ConsistentRead: true
      }));

      return {
        region,
        item: result.Item,
        timestamp: Date.now()
      };
    } catch (error) {
      console.error(`Error reading from ${region}:`, error);
      return {
        region,
        item: null,
        error: (error as Error).message
      };
    }
  }

  private areConsistent(items: any[]): boolean {
    if (items.length <= 1) return true;

    const first = JSON.stringify(items[0]);
    return items.every(item => JSON.stringify(item) === first);
  }

  private async publishMetrics(data: any) {
    await this.cloudwatch.send(new PutMetricDataCommand({
      Namespace: 'Banking/DataSync',
      MetricData: [
        {
          MetricName: 'ConsistencyCheck',
          Value: data.consistent ? 1 : 0,
          Unit: 'Count',
          Dimensions: [
            { Name: 'TableName', Value: data.tableName }
          ]
        },
        {
          MetricName: 'ReplicationLag',
          Value: data.lagMs,
          Unit: 'Milliseconds',
          Dimensions: [
            { Name: 'TableName', Value: data.tableName }
          ]
        }
      ]
    }));
  }
}

// Scheduled check
export const handler = async () => {
  const monitor = new ConsistencyMonitor();
  
  const checks = [
    { tableName: 'banking-transactions', key: { transactionId: { S: 'txn-123' } } },
    { tableName: 'customer-profiles', key: { customerId: { S: 'cust-456' } } }
  ];

  for (const check of checks) {
    await monitor.checkConsistency(check.tableName, check.key);
  }
};
```

### 🎓 Interview Discussion Points - Q28

**Q1: What is eventual consistency?**

**A**:
- **Definition**: Updates propagate asynchronously across replicas
- **Guarantee**: Eventually all replicas converge to same state
- **Trade-off**: Lower latency and higher availability vs immediate consistency
- **Example**: DynamoDB global tables typically consistent within 1 second

**Q2: How do you handle write conflicts in multi-region?**

**A**:
- **Last-Write-Wins (LWW)**: Use timestamps, latest write wins
- **Version Vectors**: Track causality per region
- **Custom Logic**: Business rules (e.g., max balance for accounts)
- **CRDT**: Conflict-free replicated data types

**Q3: What is replication lag and how do you measure it?**

**A**:
- **Definition**: Time delay between write in source and appearance in replicas
- **Measurement**: Write timestamp vs read timestamp in different regions
- **Monitoring**: CloudWatch metrics for DynamoDB global tables
- **Typical**: <1 second for DynamoDB, 1-5 seconds for custom solutions

**Q4: When should you use strong consistency vs eventual consistency?**

**A**:
- **Strong Consistency**: Financial transactions, inventory management, critical updates
- **Eventual Consistency**: User profiles, product catalogs, read-heavy workloads
- **Trade-offs**: Strong = higher latency, lower availability; Eventual = opposite

**Q5: How do you test multi-region consistency?**

**A**:
- Write to one region, immediately read from others
- Measure replication lag
- Inject network failures to test behavior
- Use chaos engineering (AWS FIS)
- Monitor metrics: consistency rate, lag time, conflict rate

---

**End of File 14**