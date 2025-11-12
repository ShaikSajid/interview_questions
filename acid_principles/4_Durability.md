# Durability Principle

## Definition
**Durability** ensures that once a transaction is committed, it remains permanent even in the face of system failures, crashes, or power outages. The changes are persisted to non-volatile storage and can be recovered.

**Key Concept**: "Committed = Permanent" - Once you see a success confirmation, the data is safe and will survive any failure.

---

## Real-World Analogy
Think of durability like signing a legal contract. Once both parties sign and the contract is filed, it's permanent. Even if the building burns down, there are backup copies in multiple locations. Similarly, once a database transaction commits, the data is safely stored on disk and backed up.

---

## How Databases Ensure Durability

### 1. Write-Ahead Logging (WAL)
- Changes are written to log before updating actual data
- If crash occurs, replay log to recover

### 2. Transaction Logs
- Sequential log of all committed transactions
- Used for crash recovery

### 3. Checkpoints
- Periodic saves of database state to disk
- Reduces recovery time

### 4. Backups
- Regular snapshots of entire database
- Point-in-time recovery

---

## Understanding Write-Ahead Logging

```
┌────────────────────────────────────────────┐
│         TRANSACTION LIFECYCLE               │
└────────────────────────────────────────────┘

BEGIN
  ↓
[Apply Changes in Memory]
  ↓
[Write to WAL Log] ← CRITICAL STEP
  ↓
COMMIT (Write commit record to WAL)
  ↓
[Acknowledge Success to Client] ← Point of No Return
  ↓
[Lazy Write to Disk] (happens later)
  ↓
[Checkpoint] (periodic)
```

---

## Durability in PostgreSQL

```typescript
// PostgreSQL automatically ensures durability
async function transferMoneyDurable(
  fromAccount: string,
  toAccount: string,
  amount: number
): Promise<void> {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // Debit source account
    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE account_id = $2',
      [amount, fromAccount]
    );
    
    // Credit destination account
    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE account_id = $2',
      [amount, toAccount]
    );
    
    // COMMIT writes to WAL and syncs to disk
    // Before returning success, PostgreSQL ensures:
    // 1. Transaction log is written to disk
    // 2. Changes are durable
    // 3. Can survive crash
    await client.query('COMMIT');
    
    // ✅ If we reach here, transaction is DURABLE
    // Even if server crashes now, data is safe
    
    console.log('Transfer completed and persisted');
    
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

---

## Understanding fsync

```typescript
// PostgreSQL configuration for durability
const postgresqlConfig = `
  # Durability settings in postgresql.conf
  
  # fsync = on (default, recommended)
  # Forces data to disk before COMMIT returns
  # Guarantees durability but slower performance
  fsync = on
  
  # synchronous_commit = on (default)
  # Wait for WAL to be written to disk
  # Options: on, remote_apply, remote_write, local, off
  synchronous_commit = on
  
  # wal_sync_method = fsync (Unix) or open_sync (Windows)
  # How to force data to disk
  wal_sync_method = fsync
  
  # wal_level = replica (default)
  # Amount of information written to WAL
  # Options: minimal, replica, logical
  wal_level = replica
  
  # checkpoint_timeout = 5min (default)
  # Maximum time between automatic checkpoints
  checkpoint_timeout = 5min
  
  # max_wal_size = 1GB (default)
  # Maximum size of WAL before checkpoint
  max_wal_size = 1GB
`;
```

---

## Crash Recovery Example

```typescript
// Simulating crash and recovery
describe('Durability and Crash Recovery', () => {
  it('should recover committed transactions after crash', async () => {
    const client = await pool.connect();
    
    // Initial balance
    await client.query('UPDATE accounts SET balance = 1000 WHERE account_id = $1', ['ACC123']);
    
    // Start transaction
    await client.query('BEGIN');
    await client.query('UPDATE accounts SET balance = balance - 100 WHERE account_id = $1', ['ACC123']);
    await client.query('COMMIT'); // ✅ COMMITTED - Now durable
    
    client.release();
    
    // Simulate crash (restart PostgreSQL)
    await simulateDatabaseCrash();
    await restartDatabase();
    
    // After recovery, committed data is present
    const result = await pool.query('SELECT balance FROM accounts WHERE account_id = $1', ['ACC123']);
    expect(result.rows[0].balance).toBe(900); // ✅ Data survived crash
  });
  
  it('should rollback uncommitted transactions after crash', async () => {
    const client = await pool.connect();
    
    await client.query('UPDATE accounts SET balance = 1000 WHERE account_id = $1', ['ACC123']);
    
    // Start transaction but don't commit
    await client.query('BEGIN');
    await client.query('UPDATE accounts SET balance = balance - 100 WHERE account_id = $1', ['ACC123']);
    // ❌ NO COMMIT - Not durable
    
    // Simulate crash before commit
    await simulateDatabaseCrash();
    await restartDatabase();
    
    // After recovery, uncommitted changes are lost
    const result = await pool.query('SELECT balance FROM accounts WHERE account_id = $1', ['ACC123']);
    expect(result.rows[0].balance).toBe(1000); // ✅ Rolled back to last committed state
  });
});
```

---

## Durability with Replication

```typescript
// Master-Slave replication for durability
const replicationSetup = `
  -- On Master (Primary)
  wal_level = replica
  max_wal_senders = 5
  wal_keep_segments = 64
  
  -- On Slave (Standby)
  hot_standby = on
  
  -- Synchronous replication (wait for slave confirmation)
  synchronous_commit = on
  synchronous_standby_names = 'standby1'
`;

// Application code with replication awareness
async function transferWithReplication(
  fromAccount: string,
  toAccount: string,
  amount: number
): Promise<void> {
  const client = await primaryPool.connect();
  
  try {
    await client.query('BEGIN');
    
    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE account_id = $2',
      [amount, fromAccount]
    );
    
    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE account_id = $2',
      [amount, toAccount]
    );
    
    // With synchronous_commit=on, COMMIT waits for:
    // 1. WAL written to primary disk
    // 2. WAL sent to standby
    // 3. Standby writes WAL to its disk
    await client.query('COMMIT');
    
    // ✅ Data is now on MULTIPLE servers
    // Survives failure of primary server
    
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

---

## Point-in-Time Recovery (PITR)

```typescript
// Backup and recovery strategy
class BackupManager {
  // Full backup (base backup)
  async createBaseBackup(): Promise<string> {
    const timestamp = new Date().toISOString();
    const backupPath = `/backups/base-${timestamp}`;
    
    // Use pg_basebackup to create full backup
    await execCommand(`
      pg_basebackup -D ${backupPath} 
      -F tar -z -X fetch 
      -U replication_user 
      -h localhost
    `);
    
    console.log(`Base backup created: ${backupPath}`);
    return backupPath;
  }
  
  // Continuous archiving (WAL archiving)
  async archiveWALFile(walFileName: string): Promise<void> {
    const archivePath = `/backups/wal_archive/${walFileName}`;
    
    await execCommand(`cp /var/lib/postgresql/pg_wal/${walFileName} ${archivePath}`);
    
    console.log(`WAL file archived: ${walFileName}`);
  }
  
  // Restore to specific point in time
  async restoreToPointInTime(recoveryTarget: Date): Promise<void> {
    console.log(`Restoring to ${recoveryTarget.toISOString()}`);
    
    // 1. Stop database
    await execCommand('pg_ctl stop');
    
    // 2. Restore base backup
    await execCommand('rm -rf /var/lib/postgresql/data/*');
    await execCommand('tar -xzf /backups/base-latest.tar.gz -C /var/lib/postgresql/data/');
    
    // 3. Create recovery.conf
    const recoveryConfig = `
      restore_command = 'cp /backups/wal_archive/%f %p'
      recovery_target_time = '${recoveryTarget.toISOString()}'
      recovery_target_action = 'promote'
    `;
    
    await writeFile('/var/lib/postgresql/data/recovery.conf', recoveryConfig);
    
    // 4. Start database (will replay WAL to recovery target)
    await execCommand('pg_ctl start');
    
    console.log('Recovery complete');
  }
}

// Usage
const backupManager = new BackupManager();

// Daily full backup
setInterval(async () => {
  await backupManager.createBaseBackup();
}, 24 * 60 * 60 * 1000);

// Continuous WAL archiving (configured in postgresql.conf)
// archive_mode = on
// archive_command = 'node backup-script.js %p'
```

---

## Durability in Different Databases

### MongoDB (WiredTiger)

```typescript
// MongoDB durability with write concerns
import { MongoClient } from 'mongodb';

async function insertWithDurability() {
  const client = new MongoClient('mongodb://localhost:27017');
  await client.connect();
  
  const db = client.db('banking');
  const collection = db.collection('transactions');
  
  // Write concern: { w: 'majority', j: true }
  // w: 'majority' - Wait for majority of replica set to acknowledge
  // j: true - Wait for write to journal (durable)
  const result = await collection.insertOne(
    {
      fromAccount: 'ACC123',
      toAccount: 'ACC456',
      amount: 100,
      timestamp: new Date()
    },
    {
      writeConcern: {
        w: 'majority', // Wait for replication
        j: true,       // Wait for journal write
        wtimeout: 5000 // Timeout after 5 seconds
      }
    }
  );
  
  // ✅ If we reach here, write is durable
  console.log('Transaction persisted:', result.insertedId);
  
  await client.close();
}
```

---

### Redis (AOF and RDB)

```typescript
// Redis durability configuration
const redisConfig = `
  # Append-Only File (AOF) for durability
  appendonly yes
  
  # AOF sync policy
  # always - fsync after every write (slowest, most durable)
  # everysec - fsync every second (good balance)
  # no - let OS decide when to fsync (fastest, least durable)
  appendfsync everysec
  
  # RDB snapshots (backup)
  save 900 1      # Save after 900 seconds if 1 key changed
  save 300 10     # Save after 300 seconds if 10 keys changed
  save 60 10000   # Save after 60 seconds if 10000 keys changed
`;

// Using Redis with durability
import Redis from 'ioredis';

async function updateAccountBalance(accountId: string, newBalance: number): Promise<void> {
  const redis = new Redis();
  
  // SET command is durable based on appendfsync setting
  await redis.set(`account:${accountId}:balance`, newBalance);
  
  // Force immediate persistence (BGSAVE or BGREWRITEAOF)
  // Note: Blocking, use sparingly
  // await redis.save(); // Synchronous save
  await redis.bgsave(); // Background save
  
  await redis.quit();
}
```

---

### Cassandra (Commitlog and Memtable)

```typescript
// Cassandra durability
const cassandraConfig = `
  # Commit log settings
  commitlog_sync: batch
  commitlog_sync_batch_window_in_ms: 2
  
  # OR for maximum durability (slower)
  commitlog_sync: periodic
  commitlog_sync_period_in_ms: 10000
`;

// Using Cassandra
import { Client } from 'cassandra-driver';

async function insertWithDurability() {
  const client = new Client({
    contactPoints: ['localhost'],
    localDataCenter: 'datacenter1'
  });
  
  await client.connect();
  
  // Write to Cassandra (written to commit log first)
  await client.execute(
    'INSERT INTO accounts (account_id, balance) VALUES (?, ?)',
    ['ACC123', 1000],
    { prepare: true }
  );
  
  // ✅ Written to commit log (durable)
  // Later flushed to SSTable
  
  await client.shutdown();
}
```

---

## Testing Durability

```typescript
describe('Durability Tests', () => {
  it('should survive process termination', async () => {
    const client = await pool.connect();
    
    // Perform transaction
    await client.query('BEGIN');
    await client.query('INSERT INTO audit_log (event, timestamp) VALUES ($1, NOW())', ['TEST_EVENT']);
    await client.query('COMMIT');
    
    const beforeCrash = await client.query('SELECT COUNT(*) FROM audit_log WHERE event = $1', ['TEST_EVENT']);
    expect(beforeCrash.rows[0].count).toBe('1');
    
    client.release();
    
    // Forcefully kill database process
    await killDatabaseProcess();
    
    // Restart database
    await startDatabaseProcess();
    await waitForDatabaseReady();
    
    // Verify data persisted
    const afterRestart = await pool.query('SELECT COUNT(*) FROM audit_log WHERE event = $1', ['TEST_EVENT']);
    expect(afterRestart.rows[0].count).toBe('1'); // ✅ Data survived
  });
  
  it('should recover from disk failure using replica', async () => {
    // Write to primary
    await primaryClient.query('INSERT INTO transactions (txn_id, amount) VALUES ($1, $2)', ['TXN001', 100]);
    
    // Wait for replication
    await sleep(1000);
    
    // Simulate primary disk failure
    await simulatePrimaryDiskFailure();
    
    // Promote standby to primary
    await promoteStandbyToPrimary();
    
    // Verify data on new primary
    const result = await newPrimaryClient.query('SELECT amount FROM transactions WHERE txn_id = $1', ['TXN001']);
    expect(result.rows[0].amount).toBe(100); // ✅ Data available on replica
  });
});
```

---

## Interview Questions & Answers

### Q1: What is durability in ACID properties?
**Answer**: Durability guarantees that once a transaction is committed, it remains permanent even if the system crashes, loses power, or encounters failures. The changes are written to non-volatile storage (disk) and can be recovered.

**Example**: After receiving confirmation that your bank transfer succeeded, the transaction is durable - even if the bank's servers crash immediately after, your transfer is safe and will be there when systems recover.

---

### Q2: How do databases ensure durability?
**Answer**: Databases use multiple mechanisms:

1. **Write-Ahead Logging (WAL)**: Write changes to log before updating data
2. **Transaction Logs**: Sequential record of all committed transactions
3. **fsync**: Force data from memory to disk
4. **Checkpoints**: Periodic saves of database state
5. **Replication**: Copy data to multiple servers
6. **Backups**: Regular snapshots for disaster recovery

**Flow**:
```
Transaction → Write to WAL → fsync (to disk) → COMMIT returns → Data is durable
```

---

### Q3: What is Write-Ahead Logging (WAL)?
**Answer**: WAL is a technique where changes are first written to a sequential log file before being applied to the actual data files.

**Benefits**:
- **Fast**: Sequential writes are faster than random writes
- **Recoverable**: Replay log after crash to restore state
- **Atomic**: Either entire log entry exists or it doesn't

**Example**:
```
1. BEGIN transaction
2. Write "UPDATE accounts SET balance=900 WHERE id=1" to WAL
3. Write "UPDATE accounts SET balance=1100 WHERE id=2" to WAL
4. Write "COMMIT" to WAL
5. fsync WAL to disk ← Point of durability
6. Return success to client
7. Later: Apply changes to actual data files (lazy)
```

---

### Q4: What is the trade-off between durability and performance?
**Answer**: 

**High Durability (Slow)**:
- `fsync` after every transaction
- Synchronous replication
- Wait for disk write before COMMIT

**High Performance (Less Durable)**:
- `fsync` every second (small window of data loss)
- Asynchronous replication
- Rely on OS to flush to disk

**Example settings**:
```typescript
// Maximum durability (slow)
synchronous_commit = on
fsync = on

// Better performance (small risk)
synchronous_commit = off  // Risk: Lose last few transactions if crash
fsync = off               // Risk: Data loss on power failure
```

**Banking**: Choose durability
**Social media**: Choose performance (acceptable to lose a few likes)

---

### Q5: What happens during crash recovery?
**Answer**: Database recovery process:

1. **Read checkpoint**: Find last known good state
2. **Replay WAL**: Apply all committed transactions since checkpoint
3. **Rollback uncommitted**: Undo transactions that didn't complete
4. **Restore consistency**: Ensure database is in valid state

**Timeline**:
```
[Checkpoint] → [Transaction A COMMIT] → [Transaction B START] → [CRASH]
                      ↓                           ↓
                 Replayed ✅                 Rolled back ❌
```

---

### Q6: How does replication improve durability?
**Answer**: Replication creates multiple copies of data on different servers:

**Synchronous Replication** (High Durability):
- Primary waits for standby to acknowledge write
- Guaranteed to have 2+ copies before COMMIT returns
- Survives primary server failure

**Asynchronous Replication** (High Performance):
- Primary doesn't wait for standby
- Small window where standby might be behind
- Risk of losing recent transactions if primary fails

**Example**:
```typescript
// Synchronous replication
await primary.query('COMMIT'); 
// ✅ Returns only after standby has data
// Slower but more durable

// Asynchronous replication
await primary.query('COMMIT'); 
// ✅ Returns immediately
// Faster but small risk of data loss
```

---

### Q7: What is Point-in-Time Recovery (PITR)?
**Answer**: PITR allows restoring database to any specific moment in time using:

1. **Base Backup**: Full database snapshot
2. **WAL Archive**: Continuous log of all changes
3. **Recovery Target**: Desired point in time

**Use cases**:
- Accidental data deletion (restore to before deletion)
- Corruption detection (restore to before corruption)
- Testing (create copy from specific point)

**Example**:
```
Monday 2 AM: Full backup
Monday-Friday: Archive all WAL files
Friday 3 PM: Accidental DELETE all customers
Friday 3:30 PM: Restore to Friday 2:59 PM (before deletion)
```

---

### Q8: How do NoSQL databases handle durability?
**Answer**: Varies by database:

**MongoDB**:
- Write Concern `{ w: 'majority', j: true }` ensures durability
- Journal writes before data files

**Cassandra**:
- Commit log (similar to WAL)
- Memtable → SSTable flush

**Redis**:
- AOF (Append-Only File) with `appendfsync everysec`
- RDB snapshots for point-in-time backups

**DynamoDB**:
- Automatically replicates across 3 AZs
- Durability built-in

**Trade-off**: Many NoSQL databases prioritize availability over durability (eventual consistency)

---

### Q9: What are common mistakes that break durability?
**Answer**:

1. **Disabling fsync**:
```typescript
// ❌ DANGEROUS in production
fsync = off  // Risk of data loss on power failure
```

2. **Not using transactions**:
```typescript
// ❌ No durability guarantee
await db.query('UPDATE accounts SET balance = 900');
// If crash happens here, change might be lost
```

3. **Trusting caches**:
```typescript
// ❌ Cache write-back without persistence
cache.set('balance:123', 900);
// If server crashes, change is lost
```

4. **No backups**:
- Disk failure = permanent data loss without backups

5. **Async replication only**:
- Primary failure = lose recent transactions

---

### Q10: How do you test durability?
**Answer**: Testing strategies:

1. **Kill test**: Forcefully terminate database process mid-transaction
```typescript
await client.query('BEGIN');
await client.query('INSERT INTO test VALUES (1)');
await client.query('COMMIT');
await killProcess('postgres');
await restartDatabase();
// Verify data exists
```

2. **Power failure simulation**: Pull power plug (use VMs)

3. **Disk failure simulation**: Simulate disk errors

4. **Backup/restore tests**: Verify backups actually work

5. **Replication lag tests**: Verify standby catches up

6. **PITR tests**: Restore to specific point in time

**Critical**: Test backups regularly! Untested backups = no backups

---

## Scenario-Based Problems

### Scenario 1: E-Commerce Order Durability
**Problem**: Ensure customer orders are never lost, even if payment processing takes time.

**Solution**:
```typescript
async function processOrderDurable(order: Order): Promise<string> {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // 1. Persist order immediately (durable)
    const orderId = generateId();
    await client.query(
      `INSERT INTO orders (order_id, customer_id, items, status, created_at) 
       VALUES ($1, $2, $3, $4, NOW())`,
      [orderId, order.customerId, JSON.stringify(order.items), 'PENDING']
    );
    
    await client.query('COMMIT'); // ✅ Order is now DURABLE
    
    // 2. Process payment asynchronously
    // Even if this fails, order is saved
    await queuePaymentProcessing(orderId);
    
    return orderId;
    
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}

// Background worker processes payments
async function paymentWorker() {
  while (true) {
    const order = await getNextPendingOrder();
    
    try {
      await processPayment(order);
      await updateOrderStatus(order.orderId, 'PAID');
    } catch (error) {
      // Payment failed, but order is still durable
      await updateOrderStatus(order.orderId, 'PAYMENT_FAILED');
      await notifyCustomer(order.customerId);
    }
  }
}
```

---

### Scenario 2: High-Availability Banking System
**Problem**: Bank transfers must survive any single server failure.

**Solution**:
```typescript
// Multi-region replication setup
class DurableBankingSystem {
  primaryDB: DatabaseConnection;
  standbyDB: DatabaseConnection;
  
  async transfer(fromAccount: string, toAccount: string, amount: number): Promise<string> {
    const client = await this.primaryDB.connect();
    
    try {
      // Enable synchronous replication
      await client.query("SET synchronous_commit TO 'remote_apply'");
      
      await client.query('BEGIN');
      
      const txnId = generateTxnId();
      
      // Debit
      await client.query(
        'UPDATE accounts SET balance = balance - $1 WHERE account_id = $2',
        [amount, fromAccount]
      );
      
      // Credit
      await client.query(
        'UPDATE accounts SET balance = balance + $1 WHERE account_id = $2',
        [amount, toAccount]
      );
      
      // Log transaction
      await client.query(
        'INSERT INTO transaction_log (txn_id, from_account, to_account, amount) VALUES ($1, $2, $3, $4)',
        [txnId, fromAccount, toAccount, amount]
      );
      
      // COMMIT waits for:
      // 1. Primary writes to WAL
      // 2. Standby receives WAL
      // 3. Standby writes to WAL
      // 4. Standby applies changes
      await client.query('COMMIT');
      
      // ✅ Data is on PRIMARY + STANDBY
      // Survives primary server failure
      
      return txnId;
      
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
  
  // Automatic failover
  async handlePrimaryFailure(): Promise<void> {
    console.log('Primary failed, promoting standby...');
    
    // Promote standby to primary
    await this.standbyDB.query('SELECT pg_promote()');
    
    // Update application config
    this.primaryDB = this.standbyDB;
    
    // Bring up new standby
    await this.setupNewStandby();
    
    console.log('Failover complete, service restored');
  }
}
```

---

### Scenario 3: Audit Trail Durability
**Problem**: Regulatory requirement to never lose audit logs.

**Solution**:
```typescript
class AuditLogger {
  async logAction(userId: string, action: string, details: any): Promise<void> {
    const client = await pool.connect();
    
    try {
      // Use serializable isolation + fsync for maximum durability
      await client.query('BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE');
      
      const logId = generateId();
      
      // Insert audit log
      await client.query(
        `INSERT INTO audit_log 
         (log_id, user_id, action, details, timestamp, checksum) 
         VALUES ($1, $2, $3, $4, NOW(), $5)`,
        [
          logId,
          userId,
          action,
          JSON.stringify(details),
          calculateChecksum({ userId, action, details })
        ]
      );
      
      // Force immediate commit to disk
      await client.query('COMMIT');
      
      // Also write to append-only file (redundant copy)
      await this.writeToAppendOnlyFile(logId, userId, action, details);
      
      // Archive to S3 (for compliance)
      await this.archiveToS3(logId, userId, action, details);
      
    } catch (error) {
      await client.query('ROLLBACK');
      // Audit log failure is critical - alert ops team
      await this.alertOpsTeam('AUDIT_LOG_FAILURE', error);
      throw error;
    } finally {
      client.release();
    }
  }
  
  async writeToAppendOnlyFile(logId: string, userId: string, action: string, details: any): Promise<void> {
    const logEntry = `${new Date().toISOString()}|${logId}|${userId}|${action}|${JSON.stringify(details)}\n`;
    
    // Append to file with immediate fsync
    await fs.promises.appendFile('/var/log/audit/audit.log', logEntry, { flag: 'a' });
    
    // Force fsync
    const fd = await fs.promises.open('/var/log/audit/audit.log', 'a');
    await fd.sync();
    await fd.close();
  }
}
```

---

## Key Takeaways

1. ✅ **Enable fsync** in production (default in most databases)
2. ✅ **Use transactions** for critical operations
3. ✅ **Set up replication** for high availability
4. ✅ **Implement backups** and test them regularly
5. ✅ **Monitor WAL** growth and checkpoints
6. ✅ **Use synchronous replication** for critical data
7. ✅ **Test crash recovery** scenarios
8. ✅ **Archive transaction logs** for PITR

**Remember**: Durability is your insurance policy against data loss. Once committed, data should survive any failure!
