# Q29: Database - Connection Pooling

## 📋 Summary
This question covers **database connection pooling** with PostgreSQL for handling thousands of concurrent banking transactions efficiently. You'll learn why connection pools are essential, how to configure them properly, and implement production-grade connection management for high-throughput financial systems.

**What You'll Learn**:
- Why connection pooling is critical for performance
- PostgreSQL connection pool configuration with node-postgres (pg)
- Pool sizing strategies and best practices
- Connection lifecycle management
- Handling connection errors and retries
- Monitoring pool health and metrics
- Concurrent transaction handling
- Production deployment patterns

---

## 🎯 Comprehensive Explanation

### Why Connection Pooling?

#### Problem: Creating Connections is Expensive

**Without Pooling**:
```
Client Request → Create DB Connection (100-300ms)
                → Execute Query (10ms)
                → Close Connection (50ms)
                → Total: 160-360ms per request
```

**Costs of creating new connections**:
1. **TCP handshake**: 3-way handshake with database server
2. **Authentication**: Username/password verification
3. **Session initialization**: Loading configuration, setting variables
4. **Memory allocation**: Buffers for communication
5. **Connection cleanup**: Releasing resources

#### Solution: Connection Pool

**With Pooling**:
```
Client Request → Get Connection from Pool (1ms)
                → Execute Query (10ms)
                → Return Connection to Pool (1ms)
                → Total: 12ms per request
```

**Benefits**:
- **10-30x faster**: Reuse existing connections
- **Reduced load**: Database handles fewer new connections
- **Better throughput**: Handle more concurrent requests
- **Resource efficiency**: Controlled connection limits

### Connection Pool Architecture

```
Application (Multiple Threads/Processes)
         ↓
    Connection Pool
    ┌─────────────────┐
    │ [Conn1] [Conn2] │ ← Active Connections
    │ [Conn3] [Conn4] │
    │ [Conn5] [Idle]  │ ← Idle Connections
    │ [Idle]  [Idle]  │
    └─────────────────┘
         ↓
    Database Server
```

**Pool States**:
1. **Idle**: Connection available for use
2. **Active**: Connection currently executing query
3. **Waiting**: Client waiting for available connection

### Key Configuration Parameters

#### 1. **Pool Size** (`max`)
Maximum number of connections in pool.

**Formula**: `max = ((core_count * 2) + effective_spindle_count)`

For banking application:
- **Too small**: Clients wait for connections (high latency)
- **Too large**: Database overwhelmed (context switching)
- **Recommended**: 10-20 connections for most applications

#### 2. **Minimum Connections** (`min`)
Minimum idle connections to maintain.

- **Purpose**: Ready connections for burst traffic
- **Typical**: 2-5 connections
- **Trade-off**: Uses database resources when idle

#### 3. **Connection Timeout** (`connectionTimeoutMillis`)
How long to wait for available connection.

- **Too short**: Clients timeout unnecessarily
- **Too long**: Slow response under load
- **Recommended**: 30000ms (30 seconds)

#### 4. **Idle Timeout** (`idleTimeoutMillis`)
How long idle connection stays in pool.

- **Purpose**: Close unused connections
- **Recommended**: 30000ms (30 seconds)
- **Avoid**: Setting too low (connection churn)

#### 5. **Max Lifetime** (`maxLifetimeSeconds`)
Maximum connection age before recycling.

- **Purpose**: Prevent stale connections
- **Recommended**: 1800s (30 minutes)
- **Important**: Handle database failovers

### PostgreSQL-Specific Considerations

**PostgreSQL Connection Limits**:
```sql
SHOW max_connections; -- Default: 100

-- Reserve connections for:
-- - Superuser: 3
-- - Background workers: 5
-- - Application pools: 92
```

**Calculate total pool size**:
```
Total App Instances × max_per_instance ≤ max_connections - reserved
```

Example:
- Database: `max_connections = 100`
- Reserved: 10
- App instances: 5
- Pool size per instance: `(100 - 10) / 5 = 18`

---

## 💻 Example 1: Basic PostgreSQL Connection Pool Setup

**Scenario**: Configure production-grade connection pool for banking API handling account transactions.

### Implementation

```javascript
const { Pool } = require('pg');

/**
 * PostgreSQL Connection Pool Manager for Banking Application
 * Implements best practices for high-availability transaction processing
 */
class DatabasePool {
  constructor(config) {
    // Pool configuration
    this.pool = new Pool({
      // Connection details
      host: process.env.DB_HOST || 'localhost',
      port: parseInt(process.env.DB_PORT) || 5432,
      database: process.env.DB_NAME || 'banking_db',
      user: process.env.DB_USER || 'banking_user',
      password: process.env.DB_PASSWORD || 'secure_password',
      
      // SSL configuration (required for production)
      ssl: process.env.NODE_ENV === 'production' ? {
        rejectUnauthorized: true,
        ca: process.env.DB_SSL_CA,
        key: process.env.DB_SSL_KEY,
        cert: process.env.DB_SSL_CERT
      } : false,
      
      // Pool sizing
      max: parseInt(process.env.DB_POOL_MAX) || 20, // Maximum connections
      min: parseInt(process.env.DB_POOL_MIN) || 5,  // Minimum idle connections
      
      // Timeouts
      connectionTimeoutMillis: 30000, // 30 seconds to get connection
      idleTimeoutMillis: 30000,       // 30 seconds before idle connection closes
      
      // Connection lifecycle
      maxUses: 7500, // Max queries per connection before recycling
      
      // Application name (visible in pg_stat_activity)
      application_name: 'banking_api',
      
      // Statement timeout (prevent runaway queries)
      statement_timeout: 60000, // 60 seconds
      
      // Query timeout
      query_timeout: 60000,
      
      // Keep alive (detect dead connections)
      keepAlive: true,
      keepAliveInitialDelayMillis: 10000,
      
      ...config
    });
    
    // Setup event listeners
    this.setupEventListeners();
    
    // Metrics
    this.metrics = {
      totalQueries: 0,
      successfulQueries: 0,
      failedQueries: 0,
      totalConnectionTime: 0,
      errors: []
    };
  }
  
  /**
   * Setup pool event listeners
   */
  setupEventListeners() {
    // When client connects
    this.pool.on('connect', (client) => {
      console.log('✅ New client connected to pool');
      // Set session parameters
      client.query(`SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL READ COMMITTED`);
    });
    
    // When client acquired from pool
    this.pool.on('acquire', (client) => {
      console.log('📤 Client acquired from pool');
    });
    
    // When client returned to pool
    this.pool.on('release', (err, client) => {
      if (err) {
        console.error('❌ Error releasing client:', err);
      } else {
        console.log('📥 Client returned to pool');
      }
    });
    
    // When client removed from pool
    this.pool.on('remove', (client) => {
      console.log('🗑️  Client removed from pool');
    });
    
    // Pool errors
    this.pool.on('error', (err, client) => {
      console.error('❌ Unexpected pool error:', err);
      this.metrics.errors.push({
        error: err.message,
        timestamp: new Date()
      });
    });
  }
  
  /**
   * Execute query with automatic connection management
   */
  async query(text, params) {
    const start = Date.now();
    this.metrics.totalQueries++;
    
    try {
      const result = await this.pool.query(text, params);
      
      this.metrics.successfulQueries++;
      this.metrics.totalConnectionTime += Date.now() - start;
      
      return result;
    } catch (error) {
      this.metrics.failedQueries++;
      console.error('Query error:', error);
      throw error;
    }
  }
  
  /**
   * Get connection for transaction
   */
  async getClient() {
    return await this.pool.connect();
  }
  
  /**
   * Get pool statistics
   */
  getStats() {
    return {
      // Pool metrics
      totalCount: this.pool.totalCount,     // Total connections
      idleCount: this.pool.idleCount,       // Idle connections
      waitingCount: this.pool.waitingCount, // Waiting clients
      
      // Query metrics
      totalQueries: this.metrics.totalQueries,
      successfulQueries: this.metrics.successfulQueries,
      failedQueries: this.metrics.failedQueries,
      averageQueryTime: this.metrics.totalQueries > 0
        ? this.metrics.totalConnectionTime / this.metrics.totalQueries
        : 0,
      
      // Error count
      errorCount: this.metrics.errors.length,
      recentErrors: this.metrics.errors.slice(-5)
    };
  }
  
  /**
   * Health check
   */
  async healthCheck() {
    try {
      const result = await this.query('SELECT NOW() as current_time');
      return {
        healthy: true,
        timestamp: result.rows[0].current_time,
        stats: this.getStats()
      };
    } catch (error) {
      return {
        healthy: false,
        error: error.message,
        stats: this.getStats()
      };
    }
  }
  
  /**
   * Graceful shutdown
   */
  async close() {
    console.log('\n🛑 Closing database pool...');
    await this.pool.end();
    console.log('✅ Pool closed');
  }
}

/**
 * Banking Account Repository using Connection Pool
 */
class AccountRepository {
  constructor(db) {
    this.db = db;
  }
  
  /**
   * Get account balance
   */
  async getBalance(accountId) {
    const query = `
      SELECT account_id, balance, currency, status
      FROM accounts
      WHERE account_id = $1
    `;
    
    const result = await this.db.query(query, [accountId]);
    
    if (result.rows.length === 0) {
      throw new Error('Account not found');
    }
    
    return result.rows[0];
  }
  
  /**
   * Update account balance
   */
  async updateBalance(accountId, newBalance) {
    const query = `
      UPDATE accounts
      SET balance = $2, updated_at = NOW()
      WHERE account_id = $1
      RETURNING account_id, balance
    `;
    
    const result = await this.db.query(query, [accountId, newBalance]);
    return result.rows[0];
  }
  
  /**
   * Get account transaction history
   */
  async getTransactions(accountId, limit = 10) {
    const query = `
      SELECT 
        transaction_id,
        transaction_type,
        amount,
        balance_after,
        description,
        created_at
      FROM transactions
      WHERE account_id = $1
      ORDER BY created_at DESC
      LIMIT $2
    `;
    
    const result = await this.db.query(query, [accountId, limit]);
    return result.rows;
  }
  
  /**
   * Record transaction
   */
  async recordTransaction(accountId, type, amount, balanceAfter, description) {
    const query = `
      INSERT INTO transactions (
        account_id, transaction_type, amount, balance_after, description
      ) VALUES ($1, $2, $3, $4, $5)
      RETURNING transaction_id, created_at
    `;
    
    const result = await this.db.query(query, [
      accountId,
      type,
      amount,
      balanceAfter,
      description
    ]);
    
    return result.rows[0];
  }
}

// Usage Example
async function main() {
  console.log('=== PostgreSQL Connection Pool Example ===\n');
  
  // Initialize pool
  const db = new DatabasePool({
    max: 10, // Smaller for demo
    min: 2
  });
  
  const accountRepo = new AccountRepository(db);
  
  try {
    // Test 1: Health check
    console.log('1. Database Health Check:');
    const health = await db.healthCheck();
    console.log('Healthy:', health.healthy);
    console.log('Database time:', health.timestamp);
    console.log('Pool stats:', health.stats);
    
    // Test 2: Setup test tables
    console.log('\n2. Setting up test tables...');
    await db.query(`
      CREATE TABLE IF NOT EXISTS accounts (
        account_id VARCHAR(20) PRIMARY KEY,
        customer_name VARCHAR(100),
        balance DECIMAL(15, 2),
        currency VARCHAR(3),
        status VARCHAR(20),
        created_at TIMESTAMP DEFAULT NOW(),
        updated_at TIMESTAMP DEFAULT NOW()
      )
    `);
    
    await db.query(`
      CREATE TABLE IF NOT EXISTS transactions (
        transaction_id SERIAL PRIMARY KEY,
        account_id VARCHAR(20) REFERENCES accounts(account_id),
        transaction_type VARCHAR(20),
        amount DECIMAL(15, 2),
        balance_after DECIMAL(15, 2),
        description TEXT,
        created_at TIMESTAMP DEFAULT NOW()
      )
    `);
    
    // Insert test account
    await db.query(`
      INSERT INTO accounts (account_id, customer_name, balance, currency, status)
      VALUES ($1, $2, $3, $4, $5)
      ON CONFLICT (account_id) DO UPDATE SET balance = $3
    `, ['ACC-001', 'John Doe', 10000.00, 'USD', 'active']);
    
    console.log('✅ Tables created');
    
    // Test 3: Get account balance
    console.log('\n3. Get Account Balance:');
    const account = await accountRepo.getBalance('ACC-001');
    console.log('Account:', account.account_id);
    console.log('Balance:', `$${parseFloat(account.balance).toFixed(2)}`);
    console.log('Status:', account.status);
    
    // Test 4: Record transaction
    console.log('\n4. Record Transaction:');
    const newBalance = parseFloat(account.balance) + 500;
    await accountRepo.updateBalance('ACC-001', newBalance);
    await accountRepo.recordTransaction(
      'ACC-001',
      'deposit',
      500,
      newBalance,
      'ATM deposit'
    );
    console.log('✅ Deposited $500');
    
    // Test 5: Get transaction history
    console.log('\n5. Transaction History:');
    const transactions = await accountRepo.getTransactions('ACC-001', 5);
    transactions.forEach(txn => {
      console.log(`  ${txn.transaction_type}: $${parseFloat(txn.amount).toFixed(2)}`);
    });
    
    // Test 6: Concurrent queries
    console.log('\n6. Testing Concurrent Queries:');
    const promises = [];
    for (let i = 0; i < 20; i++) {
      promises.push(accountRepo.getBalance('ACC-001'));
    }
    
    const start = Date.now();
    await Promise.all(promises);
    const duration = Date.now() - start;
    
    console.log(`✅ Executed 20 concurrent queries in ${duration}ms`);
    console.log('Average:', `${(duration / 20).toFixed(2)}ms per query`);
    
    // Test 7: Pool statistics
    console.log('\n7. Pool Statistics:');
    const stats = db.getStats();
    console.log('Total connections:', stats.totalCount);
    console.log('Idle connections:', stats.idleCount);
    console.log('Waiting clients:', stats.waitingCount);
    console.log('Total queries:', stats.totalQueries);
    console.log('Success rate:', `${((stats.successfulQueries / stats.totalQueries) * 100).toFixed(2)}%`);
    console.log('Average query time:', `${stats.averageQueryTime.toFixed(2)}ms`);
    
  } catch (error) {
    console.error('Error:', error);
  } finally {
    // Cleanup
    await db.close();
  }
}

// Run if executed directly
if (require.main === module) {
  main().catch(console.error);
}

module.exports = { DatabasePool, AccountRepository };
```

### Database Setup

```sql
-- Create database
CREATE DATABASE banking_db;

-- Create user
CREATE USER banking_user WITH PASSWORD 'secure_password';

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE banking_db TO banking_user;
```

### Running the Example

```bash
# Install dependencies
npm install pg

# Set environment variables
export DB_HOST=localhost
export DB_PORT=5432
export DB_NAME=banking_db
export DB_USER=banking_user
export DB_PASSWORD=secure_password
export DB_POOL_MAX=20
export DB_POOL_MIN=5

# Run example
node connection_pool_example.js
```

### Expected Output

```
=== PostgreSQL Connection Pool Example ===

1. Database Health Check:
✅ New client connected to pool
📤 Client acquired from pool
📥 Client returned to pool
Healthy: true
Database time: 2024-11-15T10:30:45.123Z
Pool stats: {
  totalCount: 2,
  idleCount: 2,
  waitingCount: 0,
  totalQueries: 1,
  successfulQueries: 1
}

2. Setting up test tables...
✅ Tables created

3. Get Account Balance:
Account: ACC-001
Balance: $10000.00
Status: active

4. Record Transaction:
✅ Deposited $500

5. Transaction History:
  deposit: $500.00

6. Testing Concurrent Queries:
✅ Executed 20 concurrent queries in 45ms
Average: 2.25ms per query

7. Pool Statistics:
Total connections: 10
Idle connections: 8
Waiting clients: 0
Total queries: 26
Success rate: 100.00%
Average query time: 3.85ms

🛑 Closing database pool...
✅ Pool closed
```

### Key Takeaways

1. **Reuse connections** - 10-30x faster than creating new ones
2. **Configure pool size** - Based on database limits and app instances
3. **Monitor metrics** - Track idle, active, waiting connections
4. **Handle errors** - Pool errors, connection failures
5. **Graceful shutdown** - Close pool properly on exit
6. **Session configuration** - Set isolation levels, timeouts
7. **Health checks** - Monitor pool and database status

---

## 💻 Example 2: Transaction Management with Connection Pool

**Scenario**: Handle banking transfers with ACID transactions using connection pool.

### Implementation

```javascript
const { Pool } = require('pg');

/**
 * Transaction Manager for Banking Operations
 * Ensures ACID compliance for financial transactions
 */
class TransactionManager {
  constructor(pool) {
    this.pool = pool;
  }
  
  /**
   * Execute function within database transaction
   * Automatically handles BEGIN, COMMIT, ROLLBACK
   */
  async executeTransaction(callback) {
    const client = await this.pool.connect();
    
    try {
      // Start transaction
      await client.query('BEGIN');
      
      // Execute callback with client
      const result = await callback(client);
      
      // Commit transaction
      await client.query('COMMIT');
      
      return result;
    } catch (error) {
      // Rollback on error
      await client.query('ROLLBACK');
      throw error;
    } finally {
      // Always release client back to pool
      client.release();
    }
  }
  
  /**
   * Transfer money between accounts (ACID transaction)
   */
  async transfer(fromAccountId, toAccountId, amount) {
    return await this.executeTransaction(async (client) => {
      // 1. Lock source account (FOR UPDATE prevents concurrent modifications)
      const sourceResult = await client.query(
        `SELECT account_id, balance, status
         FROM accounts
         WHERE account_id = $1
         FOR UPDATE`,
        [fromAccountId]
      );
      
      if (sourceResult.rows.length === 0) {
        throw new Error('Source account not found');
      }
      
      const sourceAccount = sourceResult.rows[0];
      
      if (sourceAccount.status !== 'active') {
        throw new Error('Source account is not active');
      }
      
      // 2. Check sufficient funds
      if (parseFloat(sourceAccount.balance) < amount) {
        throw new Error('Insufficient funds');
      }
      
      // 3. Lock destination account
      const destResult = await client.query(
        `SELECT account_id, balance, status
         FROM accounts
         WHERE account_id = $1
         FOR UPDATE`,
        [toAccountId]
      );
      
      if (destResult.rows.length === 0) {
        throw new Error('Destination account not found');
      }
      
      const destAccount = destResult.rows[0];
      
      if (destAccount.status !== 'active') {
        throw new Error('Destination account is not active');
      }
      
      // 4. Debit source account
      const newSourceBalance = parseFloat(sourceAccount.balance) - amount;
      await client.query(
        `UPDATE accounts
         SET balance = $1, updated_at = NOW()
         WHERE account_id = $2`,
        [newSourceBalance, fromAccountId]
      );
      
      // 5. Credit destination account
      const newDestBalance = parseFloat(destAccount.balance) + amount;
      await client.query(
        `UPDATE accounts
         SET balance = $1, updated_at = NOW()
         WHERE account_id = $2`,
        [newDestBalance, toAccountId]
      );
      
      // 6. Record transactions
      const transferId = `TXN-${Date.now()}`;
      
      await client.query(
        `INSERT INTO transactions (
           account_id, transaction_type, amount, balance_after, description
         ) VALUES ($1, $2, $3, $4, $5)`,
        [fromAccountId, 'debit', amount, newSourceBalance, `Transfer to ${toAccountId} (${transferId})`]
      );
      
      await client.query(
        `INSERT INTO transactions (
           account_id, transaction_type, amount, balance_after, description
         ) VALUES ($1, $2, $3, $4, $5)`,
        [toAccountId, 'credit', amount, newDestBalance, `Transfer from ${fromAccountId} (${transferId})`]
      );
      
      // 7. Return transfer summary
      return {
        transferId,
        from: {
          accountId: fromAccountId,
          previousBalance: parseFloat(sourceAccount.balance),
          newBalance: newSourceBalance
        },
        to: {
          accountId: toAccountId,
          previousBalance: parseFloat(destAccount.balance),
          newBalance: newDestBalance
        },
        amount,
        timestamp: new Date()
      };
    });
  }
  
  /**
   * Batch transfer (all or nothing)
   */
  async batchTransfer(transfers) {
    return await this.executeTransaction(async (client) => {
      const results = [];
      
      for (const transfer of transfers) {
        const { from, to, amount } = transfer;
        
        // Perform transfer within same transaction
        const result = await this.singleTransfer(client, from, to, amount);
        results.push(result);
      }
      
      return results;
    });
  }
  
  /**
   * Single transfer within existing transaction
   */
  async singleTransfer(client, fromAccountId, toAccountId, amount) {
    // Lock and validate source
    const sourceResult = await client.query(
      'SELECT balance FROM accounts WHERE account_id = $1 FOR UPDATE',
      [fromAccountId]
    );
    
    if (sourceResult.rows.length === 0) {
      throw new Error(`Account ${fromAccountId} not found`);
    }
    
    const sourceBalance = parseFloat(sourceResult.rows[0].balance);
    
    if (sourceBalance < amount) {
      throw new Error(`Insufficient funds in ${fromAccountId}`);
    }
    
    // Debit source
    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE account_id = $2',
      [amount, fromAccountId]
    );
    
    // Credit destination
    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE account_id = $2',
      [amount, toAccountId]
    );
    
    return { from: fromAccountId, to: toAccountId, amount };
  }
}

/**
 * Concurrent Transaction Load Tester
 */
class LoadTester {
  constructor(transactionManager) {
    this.txnManager = transactionManager;
  }
  
  /**
   * Simulate concurrent transactions
   */
  async runConcurrentTransfers(count = 100) {
    console.log(`\n🚀 Running ${count} concurrent transfers...`);
    
    const start = Date.now();
    const promises = [];
    
    for (let i = 0; i < count; i++) {
      const amount = Math.floor(Math.random() * 100) + 10;
      
      // Randomly transfer between two test accounts
      const promise = this.txnManager.transfer('ACC-001', 'ACC-002', amount)
        .catch(error => ({
          error: error.message
        }));
      
      promises.push(promise);
    }
    
    const results = await Promise.all(promises);
    const duration = Date.now() - start;
    
    const successful = results.filter(r => !r.error).length;
    const failed = results.filter(r => r.error).length;
    
    return {
      total: count,
      successful,
      failed,
      duration,
      throughput: (successful / (duration / 1000)).toFixed(2)
    };
  }
}

// Usage Example
async function main() {
  console.log('=== Transaction Management with Connection Pool ===\n');
  
  const pool = new Pool({
    host: 'localhost',
    port: 5432,
    database: 'banking_db',
    user: 'banking_user',
    password: 'secure_password',
    max: 20,
    min: 5
  });
  
  const txnManager = new TransactionManager(pool);
  
  try {
    // Setup test accounts
    console.log('1. Setting up test accounts...');
    await pool.query(`
      INSERT INTO accounts (account_id, customer_name, balance, currency, status)
      VALUES
        ('ACC-001', 'Alice', 10000.00, 'USD', 'active'),
        ('ACC-002', 'Bob', 5000.00, 'USD', 'active')
      ON CONFLICT (account_id) DO UPDATE
      SET balance = EXCLUDED.balance
    `);
    
    // Test 2: Simple transfer
    console.log('\n2. Simple Transfer:');
    const transfer1 = await txnManager.transfer('ACC-001', 'ACC-002', 500);
    console.log('Transfer ID:', transfer1.transferId);
    console.log('From:', `${transfer1.from.accountId} ($${transfer1.from.previousBalance} → $${transfer1.from.newBalance})`);
    console.log('To:', `${transfer1.to.accountId} ($${transfer1.to.previousBalance} → $${transfer1.to.newBalance})`);
    
    // Test 3: Failed transfer (insufficient funds)
    console.log('\n3. Failed Transfer (Insufficient Funds):');
    try {
      await txnManager.transfer('ACC-001', 'ACC-002', 999999);
    } catch (error) {
      console.log('❌ Transfer failed:', error.message);
    }
    
    // Test 4: Concurrent transfers
    console.log('\n4. Concurrent Transfers:');
    const loadTester = new LoadTester(txnManager);
    const loadResults = await loadTester.runConcurrentTransfers(50);
    
    console.log('\nLoad Test Results:');
    console.log('Total attempts:', loadResults.total);
    console.log('Successful:', loadResults.successful);
    console.log('Failed:', loadResults.failed);
    console.log('Duration:', `${loadResults.duration}ms`);
    console.log('Throughput:', `${loadResults.throughput} txn/sec`);
    
    // Test 5: Verify final balances
    console.log('\n5. Final Balances:');
    const acc1 = await pool.query('SELECT balance FROM accounts WHERE account_id = $1', ['ACC-001']);
    const acc2 = await pool.query('SELECT balance FROM accounts WHERE account_id = $1', ['ACC-002']);
    
    console.log('ACC-001:', `$${parseFloat(acc1.rows[0].balance).toFixed(2)}`);
    console.log('ACC-002:', `$${parseFloat(acc2.rows[0].balance).toFixed(2)}`);
    
    const total = parseFloat(acc1.rows[0].balance) + parseFloat(acc2.rows[0].balance);
    console.log('Total:', `$${total.toFixed(2)}`);
    console.log('Conservation:', total === 15000 ? '✅' : '❌');
    
  } catch (error) {
    console.error('Error:', error);
  } finally {
    await pool.end();
  }
}

if (require.main === module) {
  main().catch(console.error);
}

module.exports = { TransactionManager, LoadTester };
```

### Running the Example

```bash
node transaction_example.js
```

### Expected Output

```
=== Transaction Management with Connection Pool ===

1. Setting up test accounts...

2. Simple Transfer:
Transfer ID: TXN-1700000000000
From: ACC-001 ($10000 → $9500)
To: ACC-002 ($5000 → $5500)

3. Failed Transfer (Insufficient Funds):
❌ Transfer failed: Insufficient funds

4. Concurrent Transfers:
🚀 Running 50 concurrent transfers...

Load Test Results:
Total attempts: 50
Successful: 45
Failed: 5
Duration: 234ms
Throughput: 192.31 txn/sec

5. Final Balances:
ACC-001: $7250.00
ACC-002: $7750.00
Total: $15000.00
Conservation: ✅
```

### Key Takeaways

1. **ACID transactions** - All or nothing with BEGIN/COMMIT/ROLLBACK
2. **Row locking** - FOR UPDATE prevents race conditions
3. **Connection management** - Acquire, use, release pattern
4. **Error handling** - Automatic rollback on errors
5. **Money conservation** - Total money never changes
6. **Concurrency** - Pool handles many simultaneous transactions
7. **Performance** - 100+ txn/sec on modest hardware

---

## 🎓 Key Takeaways

### Connection Pool Best Practices

1. **Size your pool correctly**
   ```javascript
   // Formula: (core_count * 2) + spindles
   // For most apps: 10-20 connections
   max: 20,
   min: 5
   ```

2. **Set appropriate timeouts**
   ```javascript
   connectionTimeoutMillis: 30000,  // 30s to get connection
   idleTimeoutMillis: 30000,        // 30s before idle closes
   statement_timeout: 60000         // 60s max query time
   ```

3. **Monitor pool health**
   - totalCount: Total connections
   - idleCount: Available connections
   - waitingCount: Clients waiting
   - Alert if waitingCount > 0 consistently

4. **Handle errors gracefully**
   - Connection failures: Retry with backoff
   - Pool errors: Alert operations team
   - Query timeouts: Log for analysis

5. **Use transactions correctly**
   - Always release client in finally block
   - Use FOR UPDATE for row locking
   - Keep transactions short

6. **Connection lifecycle**
   - connect event: Set session parameters
   - acquire event: Track usage
   - release event: Return to pool
   - remove event: Connection closed

7. **Production configuration**
   ```javascript
   {
     max: 20,
     min: 5,
     connectionTimeoutMillis: 30000,
     idleTimeoutMillis: 30000,
     maxUses: 7500,
     keepAlive: true,
     ssl: { rejectUnauthorized: true }
   }
   ```

8. **Scaling considerations**
   - Horizontal: `total_instances × max_per_instance ≤ db_max_connections`
   - Vertical: Increase database max_connections
   - Connection poolers: PgBouncer, PgPool for very high scale

9. **Testing**
   - Load test with concurrent connections
   - Test connection exhaustion scenarios
   - Verify transaction isolation

10. **Monitoring metrics**
    - Average query time
    - Pool exhaustion events
    - Connection errors
    - Transaction rollback rate

---

## 📚 Additional Resources

- **node-postgres**: [Official Documentation](https://node-postgres.com/)
- **PostgreSQL Connection Limits**: [Documentation](https://www.postgresql.org/docs/current/runtime-config-connection.html)
- **PgBouncer**: [Connection Pooler](https://www.pgbouncer.org/)
- **ACID Transactions**: [PostgreSQL Guide](https://www.postgresql.org/docs/current/tutorial-transactions.html)

---

**Next**: Q30 - Database Transactions & ACID Principles
