# Q30: Database - Transactions & ACID Principles

## 📋 Summary
Database transactions are fundamental for maintaining data integrity in financial systems. This guide covers **ACID principles** (Atomicity, Consistency, Isolation, Durability), transaction isolation levels, deadlock handling, and distributed transactions with comprehensive banking examples.

---

## 🎯 What You'll Learn
- **ACID Properties**: Atomicity, Consistency, Isolation, Durability
- **Transaction Isolation Levels**: Read Uncommitted, Read Committed, Repeatable Read, Serializable
- **Concurrency Control**: Optimistic vs Pessimistic locking
- **Deadlock Detection**: Prevention and resolution strategies
- **Distributed Transactions**: Two-Phase Commit (2PC) pattern
- **Banking Examples**: Money transfers with rollback, concurrent account updates, multi-step transactions

---

## 📖 Comprehensive Explanation

### What is a Database Transaction?

A **transaction** is a sequence of one or more database operations that are executed as a single logical unit of work. Either all operations succeed (commit) or all fail (rollback).

```javascript
// Simple transaction example
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT; // or ROLLBACK if error
```

### Why Transactions Matter in Banking

**Without transactions**:
```javascript
// ❌ Dangerous: Money can disappear!
await debitAccount(fromAccount, 500);  // Succeeds
// Server crashes here!
await creditAccount(toAccount, 500);   // Never executes - $500 lost!
```

**With transactions**:
```javascript
// ✅ Safe: All-or-nothing guarantee
await db.transaction(async (trx) => {
  await debitAccount(fromAccount, 500, trx);  // Step 1
  await creditAccount(toAccount, 500, trx);   // Step 2
  // Both succeed or both rollback automatically
});
```

---

## 🏛️ ACID Properties

### 1. **Atomicity** (All-or-Nothing)

**Definition**: A transaction is treated as a single, indivisible unit. Either all operations succeed, or none do.

```javascript
// Example: Transfer $500 between accounts
BEGIN TRANSACTION;
  // Step 1: Debit sender
  UPDATE accounts SET balance = balance - 500 WHERE account_id = 'ACC-001';
  
  // Step 2: Credit receiver
  UPDATE accounts SET balance = balance + 500 WHERE account_id = 'ACC-002';
  
  // If EITHER fails, BOTH rollback
COMMIT;
```

**Why it matters**: Without atomicity, money could disappear (debit succeeds, credit fails) or be duplicated (credit succeeds, debit fails).

### 2. **Consistency** (Valid State Transitions)

**Definition**: A transaction brings the database from one valid state to another. All constraints, triggers, and cascades must be satisfied.

```javascript
// Example: Maintaining total money invariant
// CONSTRAINT: SUM(all_balances) = CONSTANT

BEGIN TRANSACTION;
  // Before: Account A = $1000, Account B = $500 (Total = $1500)
  
  UPDATE accounts SET balance = balance - 300 WHERE id = 'A';
  UPDATE accounts SET balance = balance + 300 WHERE id = 'B';
  
  // After: Account A = $700, Account B = $800 (Total = $1500) ✅
  // Constraint maintained!
COMMIT;
```

**Database constraints**:
- Foreign keys must be valid
- Check constraints must pass (`balance >= 0`)
- Unique constraints must hold
- Triggers execute successfully

### 3. **Isolation** (Concurrent Transactions)

**Definition**: Concurrent transactions execute as if they were serialized (one after another), preventing interference.

**The Problem** (Lost Update):
```javascript
// Without isolation:
// Transaction 1                    Transaction 2
READ balance = $1000                READ balance = $1000
balance = balance - $100            balance = balance - $200
WRITE balance = $900                WRITE balance = $800  // ❌ Lost $100!
```

**The Solution** (Isolation Levels):
```javascript
// With proper isolation:
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
  SELECT balance FROM accounts WHERE id = 1 FOR UPDATE; // Locks row
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

### 4. **Durability** (Permanent After Commit)

**Definition**: Once a transaction commits, changes are permanent even if the system crashes.

**Implementation**:
- Write-Ahead Logging (WAL): Changes logged before applied
- Disk persistence: Data flushed to disk
- Replication: Copies on multiple servers

```javascript
// Once committed, guaranteed to survive:
await db.transaction(async (trx) => {
  await trx('accounts').where({ id: 1 }).update({ balance: 1000 });
  await trx.commit(); // ✅ Durable - survives server crash!
});
```

---

## 🔒 Transaction Isolation Levels

### Isolation Level Hierarchy (Strictest to Loosest)

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|-------|-----------|-------------------|-------------|-------------|
| **Serializable** | ❌ No | ❌ No | ❌ No | Slowest |
| **Repeatable Read** | ❌ No | ❌ No | ⚠️ Possible | Medium |
| **Read Committed** | ❌ No | ⚠️ Possible | ⚠️ Possible | Fast |
| **Read Uncommitted** | ⚠️ Possible | ⚠️ Possible | ⚠️ Possible | Fastest |

### Anomalies Explained

**1. Dirty Read**: Reading uncommitted data from another transaction
```sql
-- Transaction 1
BEGIN;
UPDATE accounts SET balance = 1000 WHERE id = 1;
-- NOT COMMITTED YET

-- Transaction 2 (with READ UNCOMMITTED)
SELECT balance FROM accounts WHERE id = 1; -- Reads 1000 (uncommitted!)

-- Transaction 1
ROLLBACK; -- Balance reverts to original, but Transaction 2 saw bad data!
```

**2. Non-Repeatable Read**: Same query returns different results within transaction
```sql
-- Transaction 1
BEGIN;
SELECT balance FROM accounts WHERE id = 1; -- Returns 1000

-- Transaction 2
UPDATE accounts SET balance = 1500 WHERE id = 1;
COMMIT;

-- Transaction 1 (still open)
SELECT balance FROM accounts WHERE id = 1; -- Returns 1500 (different!)
```

**3. Phantom Read**: New rows appear in query results
```sql
-- Transaction 1
BEGIN;
SELECT COUNT(*) FROM accounts WHERE balance > 5000; -- Returns 10

-- Transaction 2
INSERT INTO accounts (id, balance) VALUES (999, 6000);
COMMIT;

-- Transaction 1 (still open)
SELECT COUNT(*) FROM accounts WHERE balance > 5000; -- Returns 11 (phantom!)
```

### Choosing Isolation Level for Banking

```javascript
// ✅ Use SERIALIZABLE for critical money operations
await db.transaction({ isolationLevel: 'serializable' }, async (trx) => {
  // Guaranteed no anomalies - safest for transfers
});

// ✅ Use REPEATABLE READ for account balance checks
await db.transaction({ isolationLevel: 'repeatableRead' }, async (trx) => {
  // Balance won't change during transaction
});

// ⚠️ Use READ COMMITTED for read-only analytics
await db.transaction({ isolationLevel: 'readCommitted' }, async (trx) => {
  // Faster, but accept some inconsistency
});

// ❌ NEVER use READ UNCOMMITTED for banking!
```

---

## 🔐 Locking Strategies

### 1. **Pessimistic Locking** (Lock Before Read)

**Concept**: Lock rows before reading to prevent concurrent modifications.

```sql
-- PostgreSQL
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE; -- Exclusive lock
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT; -- Lock released
```

**Pros**: Prevents conflicts, guaranteed consistency  
**Cons**: Slower, potential deadlocks

### 2. **Optimistic Locking** (Version Check)

**Concept**: Don't lock, but check if data changed before committing.

```sql
-- Add version column
SELECT id, balance, version FROM accounts WHERE id = 1;
-- Returns: { id: 1, balance: 1000, version: 5 }

-- Update only if version unchanged
UPDATE accounts 
SET balance = 900, version = version + 1 
WHERE id = 1 AND version = 5;
-- If 0 rows updated, data changed - retry!
```

**Pros**: Faster, no locks  
**Cons**: May need to retry on conflict

### 3. **Row-Level vs Table-Level Locking**

```sql
-- Row-level (better for concurrency)
SELECT * FROM accounts WHERE id = 1 FOR UPDATE; -- Locks only 1 row

-- Table-level (blocks all operations)
LOCK TABLE accounts IN EXCLUSIVE MODE; -- Locks entire table
```

---

## ⚠️ Deadlock Detection & Prevention

### What is a Deadlock?

Two transactions waiting for each other to release locks.

```javascript
// Transaction 1                    Transaction 2
BEGIN;                              BEGIN;
LOCK Account A;                     LOCK Account B;
// Waiting for Account B...         // Waiting for Account A...
// ⚠️ DEADLOCK! Both waiting forever
```

### Prevention Strategies

**1. Lock Ordering**: Always lock resources in same order
```javascript
// ✅ Correct: Always lock accounts in ID order
const [account1, account2] = [fromAccount, toAccount].sort();
await lockAccount(account1);
await lockAccount(account2);
```

**2. Lock Timeout**: Automatically rollback after timeout
```javascript
SET lock_timeout = '5s';
BEGIN;
-- If lock not acquired in 5 seconds, rollback
```

**3. Deadlock Detection**: Database automatically detects and kills one transaction
```javascript
// PostgreSQL detects deadlock and aborts one transaction
// Application should catch error and retry
try {
  await db.transaction(async (trx) => {
    // Operations
  });
} catch (error) {
  if (error.code === '40P01') { // Deadlock detected
    console.log('Deadlock detected, retrying...');
    // Retry logic
  }
}
```

---

## 🌍 Distributed Transactions (Two-Phase Commit)

For transactions spanning multiple databases or services.

**Phase 1 - Prepare**:
1. Coordinator asks all participants: "Can you commit?"
2. Each participant prepares and votes YES or NO

**Phase 2 - Commit/Abort**:
1. If all vote YES → Coordinator tells all to COMMIT
2. If any vote NO → Coordinator tells all to ABORT

```javascript
// Simplified 2PC example
async function distributedTransfer(from, to, amount) {
  const participants = [database1, database2];
  
  // Phase 1: Prepare
  const votes = await Promise.all(
    participants.map(db => db.prepare(from, to, amount))
  );
  
  // Phase 2: Commit or Abort
  if (votes.every(vote => vote === 'YES')) {
    await Promise.all(participants.map(db => db.commit()));
    return 'SUCCESS';
  } else {
    await Promise.all(participants.map(db => db.abort()));
    return 'FAILED';
  }
}
```

---

## 📝 Example 1: Basic Money Transfer with ACID

### Complete Banking Transaction System

```javascript
const { Pool } = require('pg');

class BankingTransactionService {
  constructor() {
    this.pool = new Pool({
      host: process.env.DB_HOST || 'localhost',
      port: process.env.DB_PORT || 5432,
      database: process.env.DB_NAME || 'banking_db',
      user: process.env.DB_USER || 'postgres',
      password: process.env.DB_PASSWORD || 'password',
      max: 20,
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 10000,
    });
  }

  /**
   * Transfer money between accounts with ACID guarantees
   * Demonstrates: Atomicity, Consistency, Isolation, Durability
   */
  async transferMoney(fromAccountId, toAccountId, amount, description = '') {
    // Validate inputs
    if (amount <= 0) {
      throw new Error('Transfer amount must be positive');
    }

    if (fromAccountId === toAccountId) {
      throw new Error('Cannot transfer to the same account');
    }

    const client = await this.pool.connect();
    
    try {
      // BEGIN TRANSACTION - Start atomic unit
      await client.query('BEGIN');
      
      console.log(`🔄 Starting transfer: ${fromAccountId} → ${toAccountId} ($${amount})`);

      // STEP 1: Lock and verify sender account (pessimistic locking)
      const senderResult = await client.query(
        `SELECT account_id, balance, account_status 
         FROM accounts 
         WHERE account_id = $1 
         FOR UPDATE`, // Exclusive lock prevents concurrent modifications
        [fromAccountId]
      );

      if (senderResult.rows.length === 0) {
        throw new Error(`Sender account ${fromAccountId} not found`);
      }

      const sender = senderResult.rows[0];

      // Check account status
      if (sender.account_status !== 'active') {
        throw new Error(`Sender account ${fromAccountId} is not active`);
      }

      // Check sufficient funds (Consistency check)
      if (parseFloat(sender.balance) < amount) {
        throw new Error(
          `Insufficient funds. Available: $${sender.balance}, Required: $${amount}`
        );
      }

      // STEP 2: Lock and verify receiver account
      const receiverResult = await client.query(
        `SELECT account_id, balance, account_status 
         FROM accounts 
         WHERE account_id = $2 
         FOR UPDATE`,
        [toAccountId]
      );

      if (receiverResult.rows.length === 0) {
        throw new Error(`Receiver account ${toAccountId} not found`);
      }

      const receiver = receiverResult.rows[0];

      if (receiver.account_status !== 'active') {
        throw new Error(`Receiver account ${toAccountId} is not active`);
      }

      // STEP 3: Debit sender account
      const newSenderBalance = parseFloat(sender.balance) - amount;
      await client.query(
        `UPDATE accounts 
         SET balance = $1, 
             updated_at = NOW() 
         WHERE account_id = $2`,
        [newSenderBalance, fromAccountId]
      );

      console.log(`  ✅ Debited $${amount} from ${fromAccountId} (New balance: $${newSenderBalance})`);

      // STEP 4: Credit receiver account
      const newReceiverBalance = parseFloat(receiver.balance) + amount;
      await client.query(
        `UPDATE accounts 
         SET balance = $1, 
             updated_at = NOW() 
         WHERE account_id = $2`,
        [newReceiverBalance, toAccountId]
      );

      console.log(`  ✅ Credited $${amount} to ${toAccountId} (New balance: $${newReceiverBalance})`);

      // STEP 5: Record transaction in ledger (audit trail)
      const transactionId = `TXN-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
      
      await client.query(
        `INSERT INTO transactions 
         (transaction_id, from_account, to_account, amount, description, status, created_at)
         VALUES ($1, $2, $3, $4, $5, $6, NOW())`,
        [transactionId, fromAccountId, toAccountId, amount, description, 'completed']
      );

      console.log(`  📝 Recorded transaction ${transactionId}`);

      // COMMIT - Make all changes permanent (Durability)
      await client.query('COMMIT');
      
      console.log(`✅ Transfer completed successfully!\n`);

      return {
        success: true,
        transactionId,
        fromAccount: {
          accountId: fromAccountId,
          previousBalance: parseFloat(sender.balance),
          newBalance: newSenderBalance,
        },
        toAccount: {
          accountId: toAccountId,
          previousBalance: parseFloat(receiver.balance),
          newBalance: newReceiverBalance,
        },
        amount,
        timestamp: new Date(),
      };

    } catch (error) {
      // ROLLBACK - Undo all changes on any error (Atomicity)
      await client.query('ROLLBACK');
      
      console.error(`❌ Transfer failed, rolling back: ${error.message}\n`);
      
      throw {
        success: false,
        error: error.message,
        fromAccount: fromAccountId,
        toAccount: toAccountId,
        amount,
      };

    } finally {
      // Always release connection back to pool
      client.release();
    }
  }

  /**
   * Get account balance (read-only transaction)
   */
  async getAccountBalance(accountId) {
    const result = await this.pool.query(
      'SELECT account_id, balance, account_status FROM accounts WHERE account_id = $1',
      [accountId]
    );

    if (result.rows.length === 0) {
      throw new Error(`Account ${accountId} not found`);
    }

    return result.rows[0];
  }

  /**
   * Get transaction history
   */
  async getTransactionHistory(accountId, limit = 10) {
    const result = await this.pool.query(
      `SELECT transaction_id, from_account, to_account, amount, description, status, created_at
       FROM transactions
       WHERE from_account = $1 OR to_account = $1
       ORDER BY created_at DESC
       LIMIT $2`,
      [accountId, limit]
    );

    return result.rows;
  }

  /**
   * Close connection pool
   */
  async close() {
    await this.pool.end();
  }
}

// Usage Example
async function runExample1() {
  console.log('=== Example 1: Basic Money Transfer with ACID ===\n');

  const bankingService = new BankingTransactionService();

  try {
    // Check initial balances
    console.log('📊 Initial Balances:');
    const account1 = await bankingService.getAccountBalance('ACC-001');
    const account2 = await bankingService.getAccountBalance('ACC-002');
    console.log(`  ACC-001: $${account1.balance}`);
    console.log(`  ACC-002: $${account2.balance}\n`);

    // Successful transfer
    console.log('Test 1: Successful Transfer');
    const result1 = await bankingService.transferMoney(
      'ACC-001',
      'ACC-002',
      500,
      'Rent payment'
    );
    console.log('Result:', result1);

    // Failed transfer (insufficient funds)
    console.log('Test 2: Failed Transfer (Insufficient Funds)');
    try {
      await bankingService.transferMoney(
        'ACC-001',
        'ACC-002',
        100000, // More than balance
        'Large payment'
      );
    } catch (error) {
      console.log('Expected error:', error.error);
    }

    // Check final balances
    console.log('\n📊 Final Balances:');
    const finalAccount1 = await bankingService.getAccountBalance('ACC-001');
    const finalAccount2 = await bankingService.getAccountBalance('ACC-002');
    console.log(`  ACC-001: $${finalAccount1.balance}`);
    console.log(`  ACC-002: $${finalAccount2.balance}\n`);

    // Transaction history
    console.log('📜 Transaction History for ACC-001:');
    const history = await bankingService.getTransactionHistory('ACC-001', 5);
    history.forEach(txn => {
      console.log(`  ${txn.transaction_id}: ${txn.from_account} → ${txn.to_account} ($${txn.amount})`);
    });

  } finally {
    await bankingService.close();
  }
}

// Run if executed directly
if (require.main === module) {
  runExample1().catch(console.error);
}

module.exports = { BankingTransactionService };
```

### Database Setup (PostgreSQL)

```sql
-- Create banking database
CREATE DATABASE banking_db;

-- Connect to database
\c banking_db

-- Create accounts table
CREATE TABLE accounts (
  account_id VARCHAR(50) PRIMARY KEY,
  customer_name VARCHAR(100) NOT NULL,
  balance NUMERIC(15, 2) NOT NULL DEFAULT 0.00,
  account_status VARCHAR(20) NOT NULL DEFAULT 'active',
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
  CONSTRAINT balance_non_negative CHECK (balance >= 0),
  CONSTRAINT valid_status CHECK (account_status IN ('active', 'suspended', 'closed'))
);

-- Create transactions table (audit trail)
CREATE TABLE transactions (
  transaction_id VARCHAR(100) PRIMARY KEY,
  from_account VARCHAR(50) NOT NULL,
  to_account VARCHAR(50) NOT NULL,
  amount NUMERIC(15, 2) NOT NULL,
  description TEXT,
  status VARCHAR(20) NOT NULL DEFAULT 'pending',
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  CONSTRAINT valid_transaction_status CHECK (status IN ('pending', 'completed', 'failed')),
  CONSTRAINT positive_amount CHECK (amount > 0)
);

-- Create indexes for performance
CREATE INDEX idx_transactions_from ON transactions(from_account);
CREATE INDEX idx_transactions_to ON transactions(to_account);
CREATE INDEX idx_transactions_created ON transactions(created_at);

-- Insert sample accounts
INSERT INTO accounts (account_id, customer_name, balance) VALUES
('ACC-001', 'Alice Johnson', 10000.00),
('ACC-002', 'Bob Smith', 5000.00),
('ACC-003', 'Charlie Brown', 7500.00),
('ACC-004', 'Diana Prince', 15000.00);

-- Verify data
SELECT * FROM accounts;
```

### Running the Example

```bash
# Install dependencies
npm install pg

# Set environment variables
export DB_HOST=localhost
export DB_PORT=5432
export DB_NAME=banking_db
export DB_USER=postgres
export DB_PASSWORD=your_password

# Run the example
node Q30_example1_basic_transfer.js
```

### Expected Output

```
=== Example 1: Basic Money Transfer with ACID ===

📊 Initial Balances:
  ACC-001: $10000.00
  ACC-002: $5000.00

Test 1: Successful Transfer
🔄 Starting transfer: ACC-001 → ACC-002 ($500)
  ✅ Debited $500 from ACC-001 (New balance: $9500)
  ✅ Credited $500 to ACC-002 (New balance: $5500)
  📝 Recorded transaction TXN-1234567890-abc123
✅ Transfer completed successfully!

Result: {
  success: true,
  transactionId: 'TXN-1234567890-abc123',
  fromAccount: { accountId: 'ACC-001', previousBalance: 10000, newBalance: 9500 },
  toAccount: { accountId: 'ACC-002', previousBalance: 5000, newBalance: 5500 },
  amount: 500,
  timestamp: 2025-11-15T10:30:00.000Z
}

Test 2: Failed Transfer (Insufficient Funds)
🔄 Starting transfer: ACC-001 → ACC-002 ($100000)
❌ Transfer failed, rolling back: Insufficient funds. Available: $9500, Required: $100000

Expected error: Insufficient funds. Available: $9500, Required: $100000

📊 Final Balances:
  ACC-001: $9500.00
  ACC-002: $5500.00

📜 Transaction History for ACC-001:
  TXN-1234567890-abc123: ACC-001 → ACC-002 ($500)
```

---

## 📝 Example 2: Transaction Isolation Levels & Race Conditions

### Demonstrating Different Isolation Levels

```javascript
const { Pool } = require('pg');

class IsolationLevelDemo {
  constructor() {
    this.pool = new Pool({
      host: 'localhost',
      port: 5432,
      database: 'banking_db',
      user: 'postgres',
      password: 'password',
    });
  }

  /**
   * Demonstrate READ COMMITTED isolation (default)
   * Allows non-repeatable reads
   */
  async demonstrateReadCommitted() {
    console.log('\n=== READ COMMITTED Isolation ===\n');

    const client1 = await this.pool.connect();
    const client2 = await this.pool.connect();

    try {
      // Transaction 1: Read balance twice
      await client1.query('BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED');
      
      // First read
      const read1 = await client1.query(
        'SELECT balance FROM accounts WHERE account_id = $1',
        ['ACC-001']
      );
      console.log(`Transaction 1 - First read: $${read1.rows[0].balance}`);

      // Transaction 2: Update balance and commit
      await client2.query('BEGIN');
      await client2.query(
        'UPDATE accounts SET balance = balance + 1000 WHERE account_id = $1',
        ['ACC-001']
      );
      await client2.query('COMMIT');
      console.log('Transaction 2 - Updated balance by +$1000 and committed\n');

      // Transaction 1: Second read (sees committed changes)
      const read2 = await client1.query(
        'SELECT balance FROM accounts WHERE account_id = $1',
        ['ACC-001']
      );
      console.log(`Transaction 1 - Second read: $${read2.rows[0].balance}`);
      
      await client1.query('COMMIT');

      console.log('⚠️ Non-repeatable read occurred! Balance changed within same transaction.\n');

    } finally {
      client1.release();
      client2.release();
    }
  }

  /**
   * Demonstrate REPEATABLE READ isolation
   * Prevents non-repeatable reads
   */
  async demonstrateRepeatableRead() {
    console.log('=== REPEATABLE READ Isolation ===\n');

    const client1 = await this.pool.connect();
    const client2 = await this.pool.connect();

    try {
      // Transaction 1: Read balance twice
      await client1.query('BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ');
      
      // First read
      const read1 = await client1.query(
        'SELECT balance FROM accounts WHERE account_id = $1',
        ['ACC-002']
      );
      console.log(`Transaction 1 - First read: $${read1.rows[0].balance}`);

      // Transaction 2: Update balance and commit
      await client2.query('BEGIN');
      await client2.query(
        'UPDATE accounts SET balance = balance + 500 WHERE account_id = $1',
        ['ACC-002']
      );
      await client2.query('COMMIT');
      console.log('Transaction 2 - Updated balance by +$500 and committed\n');

      // Transaction 1: Second read (DOES NOT see changes)
      const read2 = await client1.query(
        'SELECT balance FROM accounts WHERE account_id = $1',
        ['ACC-002']
      );
      console.log(`Transaction 1 - Second read: $${read2.rows[0].balance}`);
      
      await client1.query('COMMIT');

      console.log('✅ Repeatable read! Balance remained consistent within transaction.\n');

    } finally {
      client1.release();
      client2.release();
    }
  }

  /**
   * Demonstrate SERIALIZABLE isolation
   * Prevents all anomalies
   */
  async demonstrateSerializable() {
    console.log('=== SERIALIZABLE Isolation ===\n');

    const client1 = await this.pool.connect();
    const client2 = await this.pool.connect();

    try {
      // Transaction 1: Read all high-value accounts
      await client1.query('BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE');
      
      const read1 = await client1.query(
        'SELECT COUNT(*) FROM accounts WHERE balance > 7000'
      );
      console.log(`Transaction 1 - High-value accounts: ${read1.rows[0].count}`);

      // Transaction 2: Try to insert new high-value account
      await client2.query('BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE');
      
      try {
        await client2.query(
          `INSERT INTO accounts (account_id, customer_name, balance) 
           VALUES ('ACC-999', 'New Customer', 10000)`
        );
        await client2.query('COMMIT');
        console.log('Transaction 2 - Inserted new account\n');
      } catch (error) {
        await client2.query('ROLLBACK');
        console.log('Transaction 2 - Failed to insert (serialization conflict)\n');
      }

      // Transaction 1: Read again
      const read2 = await client1.query(
        'SELECT COUNT(*) FROM accounts WHERE balance > 7000'
      );
      console.log(`Transaction 1 - High-value accounts (recount): ${read2.rows[0].count}`);
      
      await client1.query('COMMIT');

      console.log('✅ No phantom reads! Count remained consistent.\n');

    } finally {
      client1.release();
      client2.release();
      
      // Cleanup
      await this.pool.query('DELETE FROM accounts WHERE account_id = $1', ['ACC-999']);
    }
  }

  /**
   * Demonstrate lost update problem and solution
   */
  async demonstrateLostUpdate() {
    console.log('=== Lost Update Problem & Solution ===\n');

    // Problem: Without locking
    console.log('❌ Problem: Without row locking');
    
    const client1 = await this.pool.connect();
    const client2 = await this.pool.connect();

    try {
      await client1.query('BEGIN');
      await client2.query('BEGIN');

      // Both read same balance
      const balance1 = await client1.query(
        'SELECT balance FROM accounts WHERE account_id = $1',
        ['ACC-003']
      );
      const balance2 = await client2.query(
        'SELECT balance FROM accounts WHERE account_id = $1',
        ['ACC-003']
      );

      console.log(`  Transaction 1 read: $${balance1.rows[0].balance}`);
      console.log(`  Transaction 2 read: $${balance2.rows[0].balance}\n`);

      // Both update based on their read
      const newBalance1 = parseFloat(balance1.rows[0].balance) - 100;
      const newBalance2 = parseFloat(balance2.rows[0].balance) - 200;

      await client1.query(
        'UPDATE accounts SET balance = $1 WHERE account_id = $2',
        [newBalance1, 'ACC-003']
      );
      console.log(`  Transaction 1 wrote: $${newBalance1}`);

      await client2.query(
        'UPDATE accounts SET balance = $1 WHERE account_id = $2',
        [newBalance2, 'ACC-003']
      );
      console.log(`  Transaction 2 wrote: $${newBalance2}\n`);

      await client1.query('COMMIT');
      await client2.query('COMMIT');

      const finalBalance = await this.pool.query(
        'SELECT balance FROM accounts WHERE account_id = $1',
        ['ACC-003']
      );
      console.log(`  Final balance: $${finalBalance.rows[0].balance}`);
      console.log('  ⚠️ Lost update! One deduction was overwritten.\n');

    } finally {
      client1.release();
      client2.release();
    }

    // Solution: With row locking (FOR UPDATE)
    console.log('✅ Solution: With row locking (FOR UPDATE)');

    const client3 = await this.pool.connect();
    const client4 = await this.pool.connect();

    try {
      await client3.query('BEGIN');
      
      // Transaction 1 locks the row
      const lockedBalance1 = await client3.query(
        'SELECT balance FROM accounts WHERE account_id = $1 FOR UPDATE',
        ['ACC-003']
      );
      console.log(`  Transaction 1 locked and read: $${lockedBalance1.rows[0].balance}`);

      // Transaction 2 tries to lock (will wait)
      console.log('  Transaction 2 trying to lock... (waiting)');
      
      // Transaction 1 updates and commits
      const newBalance3 = parseFloat(lockedBalance1.rows[0].balance) - 100;
      await client3.query(
        'UPDATE accounts SET balance = $1 WHERE account_id = $2',
        [newBalance3, 'ACC-003']
      );
      await client3.query('COMMIT');
      console.log(`  Transaction 1 updated to $${newBalance3} and committed\n`);

      // Now Transaction 2 can proceed
      await client4.query('BEGIN');
      const lockedBalance2 = await client4.query(
        'SELECT balance FROM accounts WHERE account_id = $1 FOR UPDATE',
        ['ACC-003']
      );
      console.log(`  Transaction 2 acquired lock and read: $${lockedBalance2.rows[0].balance}`);

      const newBalance4 = parseFloat(lockedBalance2.rows[0].balance) - 200;
      await client4.query(
        'UPDATE accounts SET balance = $1 WHERE account_id = $2',
        [newBalance4, 'ACC-003']
      );
      await client4.query('COMMIT');
      console.log(`  Transaction 2 updated to $${newBalance4} and committed\n`);

      const finalLockedBalance = await this.pool.query(
        'SELECT balance FROM accounts WHERE account_id = $1',
        ['ACC-003']
      );
      console.log(`  Final balance: $${finalLockedBalance.rows[0].balance}`);
      console.log('  ✅ Both deductions applied correctly!\n');

    } finally {
      client3.release();
      client4.release();
    }
  }

  async close() {
    await this.pool.end();
  }
}

// Run demonstrations
async function runExample2() {
  console.log('=== Example 2: Transaction Isolation Levels ===');

  const demo = new IsolationLevelDemo();

  try {
    await demo.demonstrateReadCommitted();
    await demo.demonstrateRepeatableRead();
    await demo.demonstrateSerializable();
    await demo.demonstrateLostUpdate();
  } finally {
    await demo.close();
  }
}

if (require.main === module) {
  runExample2().catch(console.error);
}

module.exports = { IsolationLevelDemo };
```

### Running Example 2

```bash
node Q30_example2_isolation_levels.js
```

### Expected Output

```
=== Example 2: Transaction Isolation Levels ===

=== READ COMMITTED Isolation ===

Transaction 1 - First read: $9500.00
Transaction 2 - Updated balance by +$1000 and committed

Transaction 1 - Second read: $10500.00
⚠️ Non-repeatable read occurred! Balance changed within same transaction.

=== REPEATABLE READ Isolation ===

Transaction 1 - First read: $5500.00
Transaction 2 - Updated balance by +$500 and committed

Transaction 1 - Second read: $5500.00
✅ Repeatable read! Balance remained consistent within transaction.

=== SERIALIZABLE Isolation ===

Transaction 1 - High-value accounts: 3
Transaction 2 - Inserted new account

Transaction 1 - High-value accounts (recount): 3
✅ No phantom reads! Count remained consistent.

=== Lost Update Problem & Solution ===

❌ Problem: Without row locking
  Transaction 1 read: $7500.00
  Transaction 2 read: $7500.00

  Transaction 1 wrote: $7400.00
  Transaction 2 wrote: $7300.00

  Final balance: $7300.00
  ⚠️ Lost update! One deduction was overwritten.

✅ Solution: With row locking (FOR UPDATE)
  Transaction 1 locked and read: $7300.00
  Transaction 2 trying to lock... (waiting)
  Transaction 1 updated to $7200.00 and committed

  Transaction 2 acquired lock and read: $7200.00
  Transaction 2 updated to $7000.00 and committed

  Final balance: $7000.00
  ✅ Both deductions applied correctly!
```

---

## 🎯 Key Takeaways

### ACID Properties
- ✅ **Atomicity**: Use transactions for all-or-nothing operations
- ✅ **Consistency**: Enforce constraints (balance >= 0, foreign keys)
- ✅ **Isolation**: Choose appropriate isolation level for your use case
- ✅ **Durability**: Trust the database to persist committed changes

### Best Practices
1. **Always use transactions** for financial operations
2. **Use FOR UPDATE** for pessimistic locking when needed
3. **Choose isolation levels wisely**:
   - SERIALIZABLE for critical transfers
   - REPEATABLE READ for balance checks
   - READ COMMITTED for analytics
4. **Handle deadlocks** with retry logic
5. **Keep transactions short** to minimize lock contention
6. **Log all transactions** for audit trails
7. **Test concurrent scenarios** thoroughly

### Common Pitfalls
- ❌ Forgetting to commit or rollback
- ❌ Using wrong isolation level
- ❌ Not handling deadlocks
- ❌ Long-running transactions
- ❌ Ignoring lock timeouts

### Performance Tips
- Use connection pooling
- Minimize transaction scope
- Index foreign keys
- Monitor lock wait times
- Consider optimistic locking for read-heavy workloads

---

**File**: `Q30_Database_Transactions.md`  
**Status**: ✅ Complete with 2 comprehensive examples
