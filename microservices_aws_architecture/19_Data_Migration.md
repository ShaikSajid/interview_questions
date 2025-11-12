# Data Migration

## Question 37: Database Migration Strategies

### 📋 Question Statement

Implement zero-downtime database migrations for Emirates NBD including schema evolution, data backfilling, dual-write patterns, and migration rollback strategies.

---

### 🔄 Database Migration Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│              ZERO-DOWNTIME DATABASE MIGRATION                               │
└────────────────────────────────────────────────────────────────────────────┘

    PHASE 1: DUAL WRITE           PHASE 2: VALIDATE           PHASE 3: CUTOVER
    ──────────────────            ─────────────────           ────────────────
    
    ┌──────────┐                  ┌──────────┐               ┌──────────┐
    │   App    │                  │   App    │               │   App    │
    └────┬─────┘                  └────┬─────┘               └────┬─────┘
         │                             │                           │
         ├──► Old DB (Write)           ├──► Old DB (Write)         └──► New DB
         │                             │                           
         └──► New DB (Write)           ├──► New DB (Write)         
                                       │
                                       └──► Validator
                                            (Compare)
```

### 📝 Migration Scripts with Flyway/Liquibase

```typescript
// migrations/V1_1__add_customer_email.ts
import { Knex } from 'knex';

export async function up(knex: Knex): Promise<void> {
  // Add column with nullable first
  await knex.schema.alterTable('customers', (table) => {
    table.string('email').nullable();
    table.index('email');
  });
  
  console.log('Added email column');
}

export async function down(knex: Knex): Promise<void> {
  await knex.schema.alterTable('customers', (table) => {
    table.dropColumn('email');
  });
}
```

```typescript
// migrations/V1_2__backfill_customer_email.ts
import { Knex } from 'knex';

export async function up(knex: Knex): Promise<void> {
  // Backfill data in batches
  const batchSize = 1000;
  let offset = 0;
  let hasMore = true;

  while (hasMore) {
    const customers = await knex('customers')
      .whereNull('email')
      .limit(batchSize)
      .offset(offset);

    if (customers.length === 0) {
      hasMore = false;
      break;
    }

    // Backfill email from legacy system
    for (const customer of customers) {
      const email = await fetchEmailFromLegacy(customer.customer_id);
      await knex('customers')
        .where('customer_id', customer.customer_id)
        .update({ email });
    }

    console.log(`Backfilled ${customers.length} customer emails`);
    offset += batchSize;
  }
}

export async function down(knex: Knex): Promise<void> {
  // Set emails back to null
  await knex('customers').update({ email: null });
}

async function fetchEmailFromLegacy(customerId: string): Promise<string> {
  // Fetch from legacy system
  return `customer${customerId}@example.com`;
}
```

```typescript
// migrations/V1_3__make_email_not_null.ts
import { Knex } from 'knex';

export async function up(knex: Knex): Promise<void> {
  // Now make it NOT NULL
  await knex.raw('ALTER TABLE customers ALTER COLUMN email SET NOT NULL');
  console.log('Made email column NOT NULL');
}

export async function down(knex: Knex): Promise<void> {
  await knex.raw('ALTER TABLE customers ALTER COLUMN email DROP NOT NULL');
}
```

### 🔀 Dual-Write Pattern Implementation

```typescript
// services/account-service/src/repository/dual-write-repository.ts
import { Pool as PgPool } from 'pg';
import { DynamoDBClient, PutItemCommand } from '@aws-sdk/client-dynamodb';
import { SNSClient, PublishCommand } from '@aws-sdk/client-sns';

export class DualWriteAccountRepository {
  private oldDb: PgPool; // PostgreSQL (old)
  private newDb: DynamoDBClient; // DynamoDB (new)
  private sns: SNSClient;
  private inconsistencyTopic: string;

  constructor(
    oldDb: PgPool,
    newDb: DynamoDBClient,
    inconsistencyTopic: string
  ) {
    this.oldDb = oldDb;
    this.newDb = newDb;
    this.sns = new SNSClient({});
    this.inconsistencyTopic = inconsistencyTopic;
  }

  async createAccount(account: Account): Promise<Account> {
    const startTime = Date.now();

    try {
      // Write to old database (authoritative source during migration)
      const oldResult = await this.writeToOldDb(account);

      try {
        // Write to new database
        await this.writeToNewDb(account);
      } catch (newDbError) {
        // Log but don't fail - old DB is still authoritative
        console.error('Failed to write to new DB:', newDbError);
        await this.reportInconsistency({
          operation: 'CREATE',
          accountId: account.accountId,
          error: (newDbError as Error).message,
          timestamp: new Date().toISOString()
        });
      }

      // Track metrics
      const duration = Date.now() - startTime;
      await this.publishMetrics('dual_write_latency', duration);

      return oldResult;

    } catch (error) {
      console.error('Failed to create account:', error);
      throw error;
    }
  }

  async updateAccount(accountId: string, updates: Partial<Account>): Promise<Account> {
    // Similar dual-write pattern for updates
    const oldResult = await this.updateInOldDb(accountId, updates);

    try {
      await this.updateInNewDb(accountId, updates);
    } catch (error) {
      console.error('Failed to update new DB:', error);
      await this.reportInconsistency({
        operation: 'UPDATE',
        accountId,
        error: (error as Error).message,
        timestamp: new Date().toISOString()
      });
    }

    return oldResult;
  }

  async getAccount(accountId: string): Promise<Account | null> {
    // During migration, read from old DB (authoritative)
    // After migration, switch to new DB
    const useNewDb = await this.isMigrationComplete();

    if (useNewDb) {
      return this.readFromNewDb(accountId);
    } else {
      return this.readFromOldDb(accountId);
    }
  }

  private async writeToOldDb(account: Account): Promise<Account> {
    const query = `
      INSERT INTO accounts (account_id, customer_id, type, balance, status, created_at)
      VALUES ($1, $2, $3, $4, $5, $6)
      RETURNING *
    `;
    
    const result = await this.oldDb.query(query, [
      account.accountId,
      account.customerId,
      account.type,
      account.balance,
      account.status,
      account.createdAt
    ]);

    return result.rows[0];
  }

  private async writeToNewDb(account: Account): Promise<void> {
    await this.newDb.send(
      new PutItemCommand({
        TableName: 'Accounts',
        Item: {
          accountId: { S: account.accountId },
          customerId: { S: account.customerId },
          type: { S: account.type },
          balance: { N: account.balance.toString() },
          status: { S: account.status },
          createdAt: { S: account.createdAt.toISOString() }
        }
      })
    );
  }

  private async updateInOldDb(accountId: string, updates: Partial<Account>): Promise<Account> {
    const fields = Object.keys(updates).map((key, i) => `${key} = $${i + 2}`).join(', ');
    const values = Object.values(updates);
    
    const query = `
      UPDATE accounts 
      SET ${fields}, updated_at = NOW()
      WHERE account_id = $1
      RETURNING *
    `;

    const result = await this.oldDb.query(query, [accountId, ...values]);
    return result.rows[0];
  }

  private async updateInNewDb(accountId: string, updates: Partial<Account>): Promise<void> {
    const updateExpression = Object.keys(updates)
      .map((key, i) => `#${key} = :val${i}`)
      .join(', ');

    const expressionAttributeNames = Object.keys(updates).reduce((acc, key) => {
      acc[`#${key}`] = key;
      return acc;
    }, {} as Record<string, string>);

    const expressionAttributeValues = Object.entries(updates).reduce((acc, [key, value], i) => {
      acc[`:val${i}`] = { S: value.toString() };
      return acc;
    }, {} as Record<string, any>);

    // DynamoDB update logic here
  }

  private async readFromOldDb(accountId: string): Promise<Account | null> {
    const result = await this.oldDb.query(
      'SELECT * FROM accounts WHERE account_id = $1',
      [accountId]
    );

    return result.rows[0] || null;
  }

  private async readFromNewDb(accountId: string): Promise<Account | null> {
    // DynamoDB read logic here
    return null;
  }

  private async reportInconsistency(inconsistency: any): Promise<void> {
    await this.sns.send(
      new PublishCommand({
        TopicArn: this.inconsistencyTopic,
        Subject: 'Data Inconsistency Detected',
        Message: JSON.stringify(inconsistency, null, 2)
      })
    );
  }

  private async isMigrationComplete(): Promise<boolean> {
    // Check feature flag or config
    return process.env.MIGRATION_COMPLETE === 'true';
  }

  private async publishMetrics(metric: string, value: number): Promise<void> {
    // Publish to CloudWatch
  }
}

interface Account {
  accountId: string;
  customerId: string;
  type: string;
  balance: number;
  status: string;
  createdAt: Date;
}
```

### 🔍 Data Validation & Reconciliation

```typescript
// services/migration/data-validator.ts
import { Pool as PgPool } from 'pg';
import { DynamoDBClient, ScanCommand, GetItemCommand } from '@aws-sdk/client-dynamodb';

export class DataValidator {
  private oldDb: PgPool;
  private newDb: DynamoDBClient;

  constructor(oldDb: PgPool, newDb: DynamoDBClient) {
    this.oldDb = oldDb;
    this.newDb = newDb;
  }

  async validateAllData(): Promise<ValidationReport> {
    console.log('Starting data validation...');

    const results: ValidationResult[] = [];
    let offset = 0;
    const batchSize = 1000;

    while (true) {
      const accounts = await this.oldDb.query(
        'SELECT * FROM accounts ORDER BY account_id LIMIT $1 OFFSET $2',
        [batchSize, offset]
      );

      if (accounts.rows.length === 0) break;

      for (const oldAccount of accounts.rows) {
        const result = await this.validateAccount(oldAccount);
        results.push(result);
      }

      offset += batchSize;
      console.log(`Validated ${offset} accounts`);
    }

    return this.generateReport(results);
  }

  private async validateAccount(oldAccount: any): Promise<ValidationResult> {
    try {
      // Fetch from new database
      const response = await this.newDb.send(
        new GetItemCommand({
          TableName: 'Accounts',
          Key: {
            accountId: { S: oldAccount.account_id }
          }
        })
      );

      if (!response.Item) {
        return {
          accountId: oldAccount.account_id,
          status: 'MISSING',
          differences: ['Record missing in new database']
        };
      }

      // Compare fields
      const differences: string[] = [];

      if (response.Item.customerId.S !== oldAccount.customer_id) {
        differences.push(`customerId: ${oldAccount.customer_id} != ${response.Item.customerId.S}`);
      }

      if (response.Item.balance.N !== oldAccount.balance.toString()) {
        differences.push(`balance: ${oldAccount.balance} != ${response.Item.balance.N}`);
      }

      if (response.Item.status.S !== oldAccount.status) {
        differences.push(`status: ${oldAccount.status} != ${response.Item.status.S}`);
      }

      if (differences.length > 0) {
        return {
          accountId: oldAccount.account_id,
          status: 'MISMATCH',
          differences
        };
      }

      return {
        accountId: oldAccount.account_id,
        status: 'VALID',
        differences: []
      };

    } catch (error) {
      return {
        accountId: oldAccount.account_id,
        status: 'ERROR',
        differences: [(error as Error).message]
      };
    }
  }

  private generateReport(results: ValidationResult[]): ValidationReport {
    const total = results.length;
    const valid = results.filter(r => r.status === 'VALID').length;
    const missing = results.filter(r => r.status === 'MISSING').length;
    const mismatched = results.filter(r => r.status === 'MISMATCH').length;
    const errors = results.filter(r => r.status === 'ERROR').length;

    return {
      total,
      valid,
      missing,
      mismatched,
      errors,
      successRate: (valid / total * 100).toFixed(2) + '%',
      issues: results.filter(r => r.status !== 'VALID')
    };
  }
}

interface ValidationResult {
  accountId: string;
  status: 'VALID' | 'MISSING' | 'MISMATCH' | 'ERROR';
  differences: string[];
}

interface ValidationReport {
  total: number;
  valid: number;
  missing: number;
  mismatched: number;
  errors: number;
  successRate: string;
  issues: ValidationResult[];
}
```

### ↩️ Rollback Strategy

```typescript
// services/migration/rollback-manager.ts
export class MigrationRollbackManager {
  
  async executeRollback(migrationId: string): Promise<void> {
    console.log(`Starting rollback for migration: ${migrationId}`);

    // Step 1: Switch traffic back to old database
    await this.updateFeatureFlag('USE_NEW_DB', false);
    console.log('Switched traffic to old database');

    // Step 2: Stop dual writes to new database
    await this.updateFeatureFlag('DUAL_WRITE_ENABLED', false);
    console.log('Stopped dual writes');

    // Step 3: Clear inconsistent data from new database (optional)
    // await this.cleanupNewDatabase();

    // Step 4: Notify stakeholders
    await this.notifyRollback(migrationId);

    console.log('Rollback completed successfully');
  }

  async verifyRollback(): Promise<boolean> {
    // Verify application is reading from old database
    const isUsingOldDb = await this.checkDatabaseInUse();
    
    // Verify no errors in application logs
    const hasErrors = await this.checkApplicationErrors();

    return isUsingOldDb && !hasErrors;
  }

  private async updateFeatureFlag(flag: string, value: boolean): Promise<void> {
    // Update in AWS AppConfig or similar
    console.log(`Updated feature flag ${flag} to ${value}`);
  }

  private async checkDatabaseInUse(): Promise<boolean> {
    // Check metrics to see which database is being used
    return true;
  }

  private async checkApplicationErrors(): Promise<boolean> {
    // Check CloudWatch logs for errors
    return false;
  }

  private async notifyRollback(migrationId: string): Promise<void> {
    // Send SNS notification
    console.log(`Rollback notification sent for ${migrationId}`);
  }
}
```

### 🎓 Interview Discussion Points - Q37

**Q1: What is a zero-downtime migration strategy?**

**A**:
- **Dual-write pattern**: Write to both old and new databases
- **Read from old**: Keep old database authoritative during migration
- **Validate data**: Compare old and new databases for consistency
- **Gradual cutover**: Switch read traffic incrementally (10% → 50% → 100%)
- **Rollback plan**: Revert if issues detected

**Q2: How to handle schema changes without downtime?**

**A**:
**Backward-compatible approach:**
1. Add new column as NULLABLE
2. Deploy application code to populate new column
3. Backfill existing data
4. Make column NOT NULL
5. Remove old column (separate migration)

**Never do:**
- Rename columns (add new, migrate, drop old)
- Change data types directly
- Add NOT NULL columns without default

**Q3: What is the dual-write pattern?**

**A**:
- **Write to both** old and new databases simultaneously
- **Old DB authoritative**: Read from old database initially
- **Eventual consistency**: Async writes to new DB can fail without breaking app
- **Validation**: Compare data between databases
- **Cutover**: Switch reads to new database when validated

**Q4: How to validate data migration?**

**A**:
- **Row count**: Verify same number of records
- **Checksum**: Compare data checksums
- **Sampling**: Random sampling and comparison
- **Consistency checks**: Business rule validation
- **Automated tests**: Run test queries on both databases
- **Reconciliation reports**: Daily reports on differences

**Q5: What are common migration pitfalls?**

**A**:
- **No rollback plan**: Always have a way to revert
- **Ignoring performance**: Migration queries can lock tables
- **Not testing at scale**: Test with production data volume
- **Skipping validation**: Always validate migrated data
- **Big bang migration**: Prefer incremental approach
- **No monitoring**: Track migration progress and errors

---

## Question 38: Blue/Green Database Migration with AWS DMS

### 📋 Question Statement

Implement blue/green database migration for Emirates NBD using AWS Database Migration Service (DMS), change data capture (CDC), validation, and automated rollback strategies.

---

### 🔄 DMS Replication Instance Setup

```typescript
// infrastructure/cdk/dms-migration-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as dms from 'aws-cdk-lib/aws-dms';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as rds from 'aws-cdk-lib/aws-rds';
import * as secretsmanager from 'aws-cdk-lib/aws-secretsmanager';

export class DMSMigrationStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpc = props.vpc;

    // DMS Subnet Group
    const subnetGroup = new dms.CfnReplicationSubnetGroup(this, 'DMSSubnetGroup', {
      replicationSubnetGroupDescription: 'DMS Replication Subnet Group',
      subnetIds: vpc.privateSubnets.map(subnet => subnet.subnetId),
      replicationSubnetGroupIdentifier: 'banking-dms-subnet-group'
    });

    // DMS Replication Instance
    const replicationInstance = new dms.CfnReplicationInstance(this, 'ReplicationInstance', {
      replicationInstanceClass: 'dms.r5.xlarge',
      replicationInstanceIdentifier: 'banking-migration-instance',
      allocatedStorage: 100,
      engineVersion: '3.5.1',
      multiAz: true,
      publiclyAccessible: false,
      replicationSubnetGroupIdentifier: subnetGroup.ref,
      vpcSecurityGroupIds: [props.securityGroup.securityGroupId]
    });

    replicationInstance.addDependency(subnetGroup);

    // Source Endpoint (Legacy MySQL)
    const sourceEndpoint = new dms.CfnEndpoint(this, 'SourceEndpoint', {
      endpointType: 'source',
      engineName: 'mysql',
      endpointIdentifier: 'source-mysql-legacy',
      serverName: props.sourceDatabaseHost,
      port: 3306,
      databaseName: 'banking_legacy',
      username: props.sourceSecret.secretValueFromJson('username').unsafeUnwrap(),
      password: props.sourceSecret.secretValueFromJson('password').unsafeUnwrap(),
      mySqlSettings: {
        serverTimezone: 'UTC',
        maxFileSize: 512000,
        parallelLoadThreads: 4
      }
    });

    // Target Endpoint (Aurora PostgreSQL)
    const targetEndpoint = new dms.CfnEndpoint(this, 'TargetEndpoint', {
      endpointType: 'target',
      engineName: 'aurora-postgresql',
      endpointIdentifier: 'target-aurora-postgres',
      serverName: props.targetDatabase.clusterEndpoint.hostname,
      port: 5432,
      databaseName: 'banking_modern',
      username: props.targetSecret.secretValueFromJson('username').unsafeUnwrap(),
      password: props.targetSecret.secretValueFromJson('password').unsafeUnwrap(),
      postgreSqlSettings: {
        maxFileSize: 512000,
        executeTimeout: 300
      }
    });

    // DMS Replication Task (Full Load + CDC)
    const replicationTask = new dms.CfnReplicationTask(this, 'ReplicationTask', {
      replicationTaskIdentifier: 'banking-migration-task',
      sourceEndpointArn: sourceEndpoint.ref,
      targetEndpointArn: targetEndpoint.ref,
      replicationInstanceArn: replicationInstance.ref,
      migrationType: 'full-load-and-cdc',
      tableMappings: JSON.stringify({
        rules: [
          {
            'rule-type': 'selection',
            'rule-id': '1',
            'rule-name': 'accounts-table',
            'object-locator': {
              'schema-name': 'banking_legacy',
              'table-name': 'accounts'
            },
            'rule-action': 'include'
          },
          {
            'rule-type': 'selection',
            'rule-id': '2',
            'rule-name': 'transactions-table',
            'object-locator': {
              'schema-name': 'banking_legacy',
              'table-name': 'transactions'
            },
            'rule-action': 'include'
          },
          {
            'rule-type': 'transformation',
            'rule-id': '3',
            'rule-name': 'rename-accounts',
            'rule-target': 'table',
            'object-locator': {
              'schema-name': 'banking_legacy',
              'table-name': 'accounts'
            },
            'rule-action': 'rename',
            'value': 'accounts_v2'
          }
        ]
      }),
      replicationTaskSettings: JSON.stringify({
        TargetMetadata: {
          SupportLobs: true,
          FullLobMode: false,
          LobChunkSize: 64,
          LimitedSizeLobMode: true,
          LobMaxSize: 32
        },
        FullLoadSettings: {
          TargetTablePrepMode: 'DROP_AND_CREATE',
          CreatePkAfterFullLoad: false,
          StopTaskCachedChangesApplied: false,
          StopTaskCachedChangesNotApplied: false,
          MaxFullLoadSubTasks: 8,
          TransactionConsistencyTimeout: 600,
          CommitRate: 10000
        },
        Logging: {
          EnableLogging: true,
          LogComponents: [
            {
              Id: 'SOURCE_CAPTURE',
              Severity: 'LOGGER_SEVERITY_DEFAULT'
            },
            {
              Id: 'TARGET_APPLY',
              Severity: 'LOGGER_SEVERITY_INFO'
            }
          ]
        },
        ChangeProcessingTuning: {
          BatchApplyPreserveTransaction: true,
          BatchApplyTimeoutMin: 1,
          BatchApplyTimeoutMax: 30,
          BatchApplyMemoryLimit: 500,
          BatchSplitSize: 0,
          MinTransactionSize: 1000,
          CommitTimeout: 1,
          MemoryLimitTotal: 1024,
          MemoryKeepTime: 60,
          StatementCacheSize: 50
        }
      })
    });

    replicationTask.addDependency(replicationInstance);
  }
}
```

### 🔍 Migration Validation Service

```typescript
// services/migration/validation-service.ts
import { RDSDataService } from 'aws-sdk';
import { createPool, Pool } from 'mysql2/promise';

export class MigrationValidationService {
  private sourcePool: Pool;
  private targetPool: Pool;

  constructor(sourceConfig: any, targetConfig: any) {
    this.sourcePool = createPool(sourceConfig);
    this.targetPool = createPool(targetConfig);
  }

  async validateRowCounts(): Promise<ValidationReport> {
    const tables = ['accounts', 'transactions', 'customers'];
    const results: TableValidation[] = [];

    for (const table of tables) {
      const [sourceRows] = await this.sourcePool.query(
        `SELECT COUNT(*) as count FROM ${table}`
      );
      
      const [targetRows] = await this.targetPool.query(
        `SELECT COUNT(*) as count FROM ${table}_v2`
      );

      const sourceCount = (sourceRows as any)[0].count;
      const targetCount = (targetRows as any)[0].count;

      results.push({
        table,
        sourceCount,
        targetCount,
        valid: sourceCount === targetCount,
        difference: targetCount - sourceCount
      });
    }

    return {
      timestamp: new Date().toISOString(),
      tables: results,
      allValid: results.every(r => r.valid)
    };
  }

  async validateDataIntegrity(): Promise<IntegrityReport> {
    // Check account balances match
    const [sourceBalances] = await this.sourcePool.query(`
      SELECT 
        SUM(balance) as total_balance,
        COUNT(*) as account_count
      FROM accounts
      WHERE status = 'ACTIVE'
    `);

    const [targetBalances] = await this.targetPool.query(`
      SELECT 
        SUM(balance) as total_balance,
        COUNT(*) as account_count
      FROM accounts_v2
      WHERE status = 'ACTIVE'
    `);

    const source = (sourceBalances as any)[0];
    const target = (targetBalances as any)[0];

    return {
      totalBalanceMatch: Math.abs(source.total_balance - target.total_balance) < 0.01,
      accountCountMatch: source.account_count === target.account_count,
      sourceTotal: source.total_balance,
      targetTotal: target.total_balance,
      difference: target.total_balance - source.total_balance
    };
  }

  async validateRecentTransactions(minutes: number = 5): Promise<boolean> {
    const cutoffTime = new Date(Date.now() - minutes * 60 * 1000);

    const [sourceRecent] = await this.sourcePool.query(`
      SELECT transaction_id, amount, created_at
      FROM transactions
      WHERE created_at > ?
      ORDER BY transaction_id
    `, [cutoffTime]);

    const [targetRecent] = await this.targetPool.query(`
      SELECT transaction_id, amount, created_at
      FROM transactions_v2
      WHERE created_at > ?
      ORDER BY transaction_id
    `, [cutoffTime]);

    const sourceIds = (sourceRecent as any[]).map(t => t.transaction_id);
    const targetIds = (targetRecent as any[]).map(t => t.transaction_id);

    const missingInTarget = sourceIds.filter(id => !targetIds.includes(id));

    if (missingInTarget.length > 0) {
      console.error('Missing transactions in target:', missingInTarget);
      return false;
    }

    return true;
  }

  async compareChecksums(table: string, batchSize: number = 1000): Promise<boolean> {
    const [sourceMax] = await this.sourcePool.query(
      `SELECT MAX(id) as max_id FROM ${table}`
    );
    const maxId = (sourceMax as any)[0].max_id;

    for (let offset = 0; offset < maxId; offset += batchSize) {
      const [sourceChecksum] = await this.sourcePool.query(`
        SELECT MD5(GROUP_CONCAT(
          CONCAT_WS('|', id, account_number, balance, status)
          ORDER BY id
        )) as checksum
        FROM ${table}
        WHERE id BETWEEN ? AND ?
      `, [offset, offset + batchSize]);

      const [targetChecksum] = await this.targetPool.query(`
        SELECT MD5(STRING_AGG(
          CONCAT_WS('|', id, account_number, balance, status),
          '' ORDER BY id
        )) as checksum
        FROM ${table}_v2
        WHERE id BETWEEN $1 AND $2
      `, [offset, offset + batchSize]);

      if ((sourceChecksum as any)[0].checksum !== (targetChecksum as any)[0].checksum) {
        console.error(`Checksum mismatch for ${table} batch ${offset}-${offset + batchSize}`);
        return false;
      }
    }

    return true;
  }
}

interface ValidationReport {
  timestamp: string;
  tables: TableValidation[];
  allValid: boolean;
}

interface TableValidation {
  table: string;
  sourceCount: number;
  targetCount: number;
  valid: boolean;
  difference: number;
}

interface IntegrityReport {
  totalBalanceMatch: boolean;
  accountCountMatch: boolean;
  sourceTotal: number;
  targetTotal: number;
  difference: number;
}
```

### 🎯 Blue/Green Cutover Orchestrator

```typescript
// services/migration/cutover-orchestrator.ts
import { Route53, ECS, RDS } from 'aws-sdk';
import { MigrationValidationService } from './validation-service';

export class CutoverOrchestrator {
  private route53: Route53;
  private ecs: ECS;
  private rds: RDS;
  private validator: MigrationValidationService;

  constructor() {
    this.route53 = new Route53();
    this.ecs = new ECS();
    this.rds = new RDS();
  }

  async executeBlueToCutover(): Promise<CutoverResult> {
    console.log('Starting blue-green database cutover...');

    // Step 1: Final validation
    console.log('Step 1: Running final validation...');
    const validation = await this.validator.validateRowCounts();
    
    if (!validation.allValid) {
      throw new Error('Validation failed before cutover');
    }

    // Step 2: Stop writes to source (enable read-only)
    console.log('Step 2: Setting source database to read-only...');
    await this.setDatabaseReadOnly('source-mysql-legacy', true);

    // Step 3: Wait for CDC lag to catch up
    console.log('Step 3: Waiting for CDC replication lag...');
    await this.waitForReplicationCatchup('banking-migration-task', 10);

    // Step 4: Final validation after CDC catchup
    console.log('Step 4: Final validation after CDC catchup...');
    const finalValidation = await this.validator.validateDataIntegrity();
    
    if (!finalValidation.totalBalanceMatch) {
      console.error('Data integrity check failed!');
      await this.rollback();
      throw new Error('Cutover aborted due to data mismatch');
    }

    // Step 5: Update application configuration (weighted routing)
    console.log('Step 5: Starting gradual traffic shift...');
    await this.updateDNSWeightedRouting('banking-db.internal', {
      blue: { weight: 100, endpoint: 'source-mysql-legacy.xyz.rds.amazonaws.com' },
      green: { weight: 0, endpoint: 'target-aurora-postgres.xyz.rds.amazonaws.com' }
    });

    // Gradual shift: 10% -> 50% -> 100%
    await this.sleep(60000); // 1 minute
    await this.updateDNSWeightedRouting('banking-db.internal', {
      blue: { weight: 90, endpoint: 'source-mysql-legacy.xyz.rds.amazonaws.com' },
      green: { weight: 10, endpoint: 'target-aurora-postgres.xyz.rds.amazonaws.com' }
    });

    await this.sleep(300000); // 5 minutes - monitor
    await this.updateDNSWeightedRouting('banking-db.internal', {
      blue: { weight: 50, endpoint: 'source-mysql-legacy.xyz.rds.amazonaws.com' },
      green: { weight: 50, endpoint: 'target-aurora-postgres.xyz.rds.amazonaws.com' }
    });

    await this.sleep(300000); // 5 minutes - monitor
    await this.updateDNSWeightedRouting('banking-db.internal', {
      blue: { weight: 0, endpoint: 'source-mysql-legacy.xyz.rds.amazonaws.com' },
      green: { weight: 100, endpoint: 'target-aurora-postgres.xyz.rds.amazonaws.com' }
    });

    // Step 6: Monitor for errors
    console.log('Step 6: Monitoring for application errors...');
    const healthCheck = await this.monitorApplicationHealth(300); // 5 minutes

    if (!healthCheck.healthy) {
      console.error('Application health check failed!');
      await this.rollback();
      throw new Error('Cutover aborted due to health check failure');
    }

    // Step 7: Cutover complete
    console.log('Cutover successful! Target database is now primary.');

    return {
      success: true,
      cutoverTime: new Date().toISOString(),
      validationResults: finalValidation,
      downtimeSeconds: 0 // Zero downtime!
    };
  }

  async rollback(): Promise<void> {
    console.log('Executing rollback...');

    // Revert DNS to source database
    await this.updateDNSWeightedRouting('banking-db.internal', {
      blue: { weight: 100, endpoint: 'source-mysql-legacy.xyz.rds.amazonaws.com' },
      green: { weight: 0, endpoint: 'target-aurora-postgres.xyz.rds.amazonaws.com' }
    });

    // Re-enable writes to source
    await this.setDatabaseReadOnly('source-mysql-legacy', false);

    console.log('Rollback complete. Source database is primary again.');
  }

  private async setDatabaseReadOnly(instanceId: string, readOnly: boolean): Promise<void> {
    const parameterGroup = readOnly ? 'read-only-params' : 'read-write-params';
    
    await this.rds.modifyDBInstance({
      DBInstanceIdentifier: instanceId,
      DBParameterGroupName: parameterGroup,
      ApplyImmediately: true
    }).promise();

    await this.sleep(30000); // Wait for parameter changes
  }

  private async waitForReplicationCatchup(taskArn: string, maxLagSeconds: number): Promise<void> {
    // Poll DMS task status until CDC lag < maxLagSeconds
    let lag = Infinity;
    
    while (lag > maxLagSeconds) {
      await this.sleep(5000);
      // Check CDC lag from CloudWatch metrics
      lag = await this.getCDCLag(taskArn);
      console.log(`Current CDC lag: ${lag} seconds`);
    }
  }

  private async getCDCLag(taskArn: string): Promise<number> {
    // Query CloudWatch for CDCLatencySource metric
    return 5; // Simulated
  }

  private async updateDNSWeightedRouting(
    domain: string,
    records: { blue: { weight: number; endpoint: string }; green: { weight: number; endpoint: string } }
  ): Promise<void> {
    // Update Route 53 weighted routing
    console.log(`Updating DNS: Blue=${records.blue.weight}% Green=${records.green.weight}%`);
  }

  private async monitorApplicationHealth(durationSeconds: number): Promise<{ healthy: boolean }> {
    // Monitor CloudWatch alarms, error rates, latency
    return { healthy: true };
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

interface CutoverResult {
  success: boolean;
  cutoverTime: string;
  validationResults: any;
  downtimeSeconds: number;
}
```

### 🎓 Interview Discussion Points - Q38

**Q1: What is blue/green deployment for databases?**

**A**:
- **Blue**: Current production database
- **Green**: New database with migrated data
- **Process**: Migrate data, validate, switch traffic, keep blue for rollback
- **Benefits**: Zero downtime, instant rollback, validation before cutover

**Q2: How does AWS DMS CDC work?**

**A**:
- **Full Load**: Initial data copy
- **CDC (Change Data Capture)**: Continuous replication of changes
- **Transaction logs**: Reads MySQL binlog or PostgreSQL WAL
- **Latency**: Typically <1 second
- **Use case**: Keep databases in sync during migration

**Q3: How to validate data migration?**

**A**:
- **Row counts**: Compare table sizes
- **Checksums**: MD5 hash of data batches
- **Business metrics**: Total balances, transaction counts
- **Recent data**: Verify last N minutes of data replicated
- **Automated tests**: Run queries against both databases

**Q4: What happens if cutover fails?**

**A**:
- **Rollback plan**: Revert DNS to source database
- **Zero data loss**: Source database remains intact
- **Investigation**: Analyze logs, identify root cause
- **Retry**: Fix issues and attempt cutover again

**Q5: How to minimize cutover downtime?**

**A**:
- **Weighted routing**: Gradual traffic shift (10% → 50% → 100%)
- **Read-only mode**: Stop writes briefly during final sync
- **CDC replication**: Real-time data synchronization
- **Connection pooling**: Drain connections gracefully
- **Target**: <1 minute downtime (often zero!)

---

