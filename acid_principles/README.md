# ACID Principles - Complete Reference

## Overview

**ACID** is a set of properties that guarantee reliable processing of database transactions. These principles ensure data integrity, consistency, and reliability in database systems.

---

## The Four ACID Properties

### 1. **Atomicity** ⚛️
All operations in a transaction succeed or all fail together (all-or-nothing).

**Example**: Bank transfer - either both debit and credit happen, or neither happens.

[→ Read Detailed Atomicity Guide](./1_Atomicity.md)

---

### 2. **Consistency** ✅
Transactions bring the database from one valid state to another, maintaining all defined rules.

**Example**: Total money in the system remains constant after a transfer.

[→ Read Detailed Consistency Guide](./2_Consistency.md)

---

### 3. **Isolation** 🔒
Concurrent transactions execute independently without interfering with each other.

**Example**: Two people withdrawing from same account don't see each other's uncommitted changes.

[→ Read Detailed Isolation Guide](./3_Isolation.md)

---

### 4. **Durability** 💾
Once committed, transactions remain permanent even after system failures.

**Example**: After seeing "Transfer Successful", the data survives even if server crashes.

[→ Read Detailed Durability Guide](./4_Durability.md)

---

## ACID at a Glance

| Property | Question | Guarantee |
|----------|----------|-----------|
| **Atomicity** | Do all operations complete together? | All or nothing |
| **Consistency** | Is data always valid? | Rules never violated |
| **Isolation** | Do concurrent transactions interfere? | Independent execution |
| **Durability** | Is committed data permanent? | Survives failures |

---

## Real-World Banking Example

```typescript
// Complete ACID-compliant bank transfer
async function transferMoney(
  fromAccount: string,
  toAccount: string,
  amount: number
): Promise<string> {
  const client = await pool.connect();
  
  try {
    // ISOLATION: Set appropriate isolation level
    await client.query('BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ');
    
    // CONSISTENCY: Validate business rules
    if (amount <= 0) {
      throw new Error('Amount must be positive');
    }
    
    // Lock accounts for update (ISOLATION)
    const fromBalance = await client.query(
      'SELECT balance, status FROM accounts WHERE account_id = $1 FOR UPDATE',
      [fromAccount]
    );
    
    if (fromBalance.rowCount === 0) {
      throw new Error('Source account not found');
    }
    
    // CONSISTENCY: Check account status
    if (fromBalance.rows[0].status !== 'ACTIVE') {
      throw new Error('Source account is not active');
    }
    
    // CONSISTENCY: Check sufficient funds
    if (fromBalance.rows[0].balance < amount) {
      throw new Error('Insufficient funds');
    }
    
    // ATOMICITY: Both operations are in one transaction
    
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
    
    // Log transaction (for audit trail)
    const txnId = generateTxnId();
    await client.query(
      'INSERT INTO transactions (txn_id, from_account, to_account, amount, timestamp) VALUES ($1, $2, $3, $4, NOW())',
      [txnId, fromAccount, toAccount, amount]
    );
    
    // DURABILITY: COMMIT writes to transaction log and syncs to disk
    await client.query('COMMIT');
    
    // ✅ All ACID properties satisfied:
    // - Atomicity: All 3 operations succeeded together
    // - Consistency: All rules validated, total money unchanged
    // - Isolation: Other transactions didn't interfere
    // - Durability: Changes are permanent on disk
    
    return txnId;
    
  } catch (error) {
    // ATOMICITY: Rollback all changes on any error
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

---

## When ACID Properties Matter Most

### Critical for ACID:
- ✅ Financial transactions (banking, payments)
- ✅ E-commerce orders and inventory
- ✅ Healthcare records
- ✅ Booking systems (flights, hotels, events)
- ✅ Accounting systems
- ✅ User authentication and authorization

### Can Relax ACID:
- ⚠️ Social media likes/comments (eventual consistency OK)
- ⚠️ Analytics and reporting (read-only)
- ⚠️ Caching layers
- ⚠️ Logging and metrics (loss of few records acceptable)
- ⚠️ Real-time dashboards (approximate data OK)

---

## ACID vs BASE

| Aspect | ACID | BASE |
|--------|------|------|
| **Focus** | Consistency | Availability |
| **Model** | Strong consistency | Eventual consistency |
| **Typical Use** | Relational databases | NoSQL databases |
| **CAP Priority** | Consistency + Partition tolerance | Availability + Partition tolerance |
| **Example** | PostgreSQL, MySQL | Cassandra, DynamoDB |

**BASE** stands for:
- **B**asically **A**vailable
- **S**oft state
- **E**ventual consistency

---

## ACID in Different Databases

### Relational Databases (Full ACID)

**PostgreSQL**
```typescript
// Full ACID support with tunable isolation levels
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- All operations
COMMIT;
```

**MySQL/InnoDB**
```typescript
// ACID with row-level locking
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
-- All operations
COMMIT;
```

---

### NoSQL Databases (Limited ACID)

**MongoDB**
```typescript
// ACID transactions (v4.0+)
const session = client.startSession();
session.startTransaction();
try {
  await collection1.updateOne({}, {}, { session });
  await collection2.insertOne({}, { session });
  await session.commitTransaction();
} catch (error) {
  await session.abortTransaction();
}
```

**DynamoDB**
```typescript
// ACID transactions (up to 25 items)
await dynamodb.transactWriteItems({
  TransactItems: [
    { Update: { /* ... */ } },
    { Put: { /* ... */ } }
  ]
});
```

**Redis**
```typescript
// Atomic commands, not full ACID
const pipeline = redis.pipeline();
pipeline.decrby('account:1', 100);
pipeline.incrby('account:2', 100);
await pipeline.exec(); // Atomic execution
```

---

## SQL vs NoSQL: Choosing the Right Database

### Understanding the Trade-offs

```
┌─────────────────────────────────────────────────────────────┐
│                    SQL vs NoSQL DECISION                     │
└─────────────────────────────────────────────────────────────┘

           ACID (Strong Consistency)
                      ↑
                      |
        SQL ←─────────┼─────────→ NoSQL
    (Relational)      |        (Distributed)
                      |
                      ↓
         BASE (Eventual Consistency)
```

---

### SQL Databases (Relational)

**Characteristics**:
- ✅ Full ACID compliance
- ✅ Strong consistency
- ✅ Structured schema (tables, rows, columns)
- ✅ SQL query language
- ✅ Complex joins and relationships
- ✅ Referential integrity with foreign keys
- ⚠️ Vertical scaling (limited)
- ⚠️ Schema changes can be difficult

**Examples**: PostgreSQL, MySQL, Oracle, SQL Server

**Best for**:
- Financial systems (banking, payments)
- E-commerce transactions
- Enterprise applications (ERP, CRM)
- Data with complex relationships
- When ACID guarantees are critical
- Structured data with clear schema

---

### NoSQL Databases

**Characteristics**:
- ✅ Horizontal scaling (distributed)
- ✅ High availability
- ✅ Flexible schema (schema-less)
- ✅ High performance for specific use cases
- ⚠️ Limited ACID (varies by database)
- ⚠️ Eventual consistency (most cases)
- ⚠️ No complex joins
- ⚠️ Limited cross-document transactions

**Types**:
1. **Document**: MongoDB, CouchDB
2. **Key-Value**: Redis, DynamoDB
3. **Column-Family**: Cassandra, HBase
4. **Graph**: Neo4j, Amazon Neptune

**Best for**:
- High-volume data (millions of records)
- Real-time applications
- Content management systems
- Social networks
- IoT and sensor data
- Caching layers
- When availability > consistency

---

### Detailed Comparison Table

| Feature | SQL (PostgreSQL) | NoSQL (MongoDB) | NoSQL (Cassandra) | NoSQL (Redis) |
|---------|------------------|-----------------|-------------------|---------------|
| **ACID Support** | ✅ Full | ✅ Limited (single doc or multi-doc txn) | ⚠️ Row-level only | ⚠️ Command-level |
| **Consistency** | Strong | Tunable (strong or eventual) | Eventual | Eventual |
| **Schema** | Fixed (DDL) | Flexible (JSON) | Column-family | Key-Value |
| **Scaling** | Vertical | Horizontal | Horizontal | Horizontal |
| **Joins** | ✅ Complex joins | ⚠️ Limited ($lookup) | ❌ No joins | ❌ No joins |
| **Transactions** | Multi-table | Multi-document (v4.0+) | Lightweight | MULTI/EXEC |
| **Query Language** | SQL | MQL (MongoDB Query) | CQL (Cassandra Query) | Commands |
| **Best Use Case** | Complex relationships | Flexible documents | Time-series, wide data | Caching, sessions |

---

### Decision Matrix: When to Use What?

#### ✅ Choose SQL When:

1. **ACID is Critical**
```typescript
// Financial transfer - needs full ACID
async function bankTransfer(from: string, to: string, amount: number) {
  await db.query('BEGIN');
  await db.query('UPDATE accounts SET balance = balance - $1 WHERE id = $2', [amount, from]);
  await db.query('UPDATE accounts SET balance = balance + $1 WHERE id = $2', [amount, to]);
  await db.query('COMMIT'); // ✅ Guaranteed atomic, consistent, isolated, durable
}
```

2. **Complex Relationships**
```sql
-- Complex joins across multiple tables
SELECT 
  o.order_id,
  c.customer_name,
  p.product_name,
  SUM(oi.quantity * oi.price) as total
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.created_at > '2024-01-01'
GROUP BY o.order_id, c.customer_name, p.product_name
HAVING SUM(oi.quantity * oi.price) > 1000;
```

3. **Data Integrity is Paramount**
```sql
-- Referential integrity enforced by database
CREATE TABLE orders (
  order_id SERIAL PRIMARY KEY,
  customer_id INT NOT NULL,
  FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE RESTRICT
);
-- Cannot delete customer with existing orders ✅
```

4. **Structured, Well-Defined Schema**
```typescript
// Banking system - schema won't change often
interface Account {
  accountId: string;
  customerId: string;
  balance: number;
  accountType: 'SAVINGS' | 'CHECKING';
  status: 'ACTIVE' | 'CLOSED';
}
```

---

#### ✅ Choose NoSQL When:

1. **Massive Scale Required**
```typescript
// MongoDB - Store billions of documents
await collection.insertOne({
  userId: '123',
  timestamp: new Date(),
  event: 'page_view',
  metadata: { /* flexible structure */ }
});
// ✅ Scales horizontally across clusters
```

2. **Flexible/Evolving Schema**
```typescript
// NoSQL - Schema can evolve
const user1 = {
  name: 'John',
  email: 'john@example.com'
};

const user2 = {
  name: 'Jane',
  email: 'jane@example.com',
  social: { twitter: '@jane' }, // New field added
  preferences: { theme: 'dark' } // Another new field
};
// ✅ No schema migration needed
```

3. **High Availability Over Consistency**
```typescript
// Cassandra - Eventually consistent reads
await client.execute(
  'INSERT INTO user_activity (user_id, timestamp, action) VALUES (?, ?, ?)',
  [userId, Date.now(), 'login'],
  { consistency: cassandra.types.consistencies.one }
);
// ✅ Writes succeed even if some nodes are down
```

4. **Simple Access Patterns**
```typescript
// Redis - Key-value lookups
await redis.set(`user:${userId}:session`, sessionData);
const session = await redis.get(`user:${userId}:session`);
// ✅ No complex queries, just fast key-value access
```

---

#### 🔄 Use Both (Polyglot Persistence):

Many modern applications use **both SQL and NoSQL** for different purposes:

```typescript
// Hybrid Architecture Example: E-Commerce System

class ECommerceSystem {
  // SQL for transactional data (ACID critical)
  private postgresDB: PostgresClient;
  
  // NoSQL for different purposes
  private mongoDBClient: MongoClient;    // Product catalog (flexible schema)
  private redisClient: RedisClient;      // Caching & sessions
  private cassandraClient: CassandraClient; // User activity logs
  
  // 1. Use PostgreSQL for orders (needs ACID)
  async createOrder(order: Order): Promise<string> {
    const client = await this.postgresDB.connect();
    
    try {
      await client.query('BEGIN');
      
      // Critical transaction - needs ACID
      const orderId = await client.query(
        'INSERT INTO orders (customer_id, total) VALUES ($1, $2) RETURNING order_id',
        [order.customerId, order.total]
      );
      
      await client.query(
        'UPDATE inventory SET quantity = quantity - $1 WHERE product_id = $2',
        [order.quantity, order.productId]
      );
      
      await client.query('COMMIT');
      return orderId;
      
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    }
  }
  
  // 2. Use MongoDB for product catalog (flexible schema)
  async getProduct(productId: string): Promise<Product> {
    // Products have varying attributes (color, size, weight, etc.)
    // Flexible schema is beneficial
    return await this.mongoDBClient
      .db('catalog')
      .collection('products')
      .findOne({ _id: productId });
  }
  
  // 3. Use Redis for caching and sessions
  async getUserSession(sessionId: string): Promise<Session | null> {
    const cached = await this.redisClient.get(`session:${sessionId}`);
    
    if (cached) {
      return JSON.parse(cached); // Fast cache hit
    }
    
    // Cache miss - fetch from PostgreSQL
    const session = await this.postgresDB.query(
      'SELECT * FROM sessions WHERE session_id = $1',
      [sessionId]
    );
    
    // Cache for 1 hour
    await this.redisClient.setex(
      `session:${sessionId}`,
      3600,
      JSON.stringify(session)
    );
    
    return session;
  }
  
  // 4. Use Cassandra for user activity logs (high volume)
  async logUserActivity(userId: string, action: string): Promise<void> {
    // Millions of events per day - need horizontal scaling
    // Eventual consistency is acceptable for analytics
    await this.cassandraClient.execute(
      'INSERT INTO user_activity (user_id, timestamp, action) VALUES (?, ?, ?)',
      [userId, Date.now(), action],
      { prepare: true }
    );
  }
}
```

---

### Real-World Use Case Examples

#### Example 1: Banking Application

```typescript
// ✅ Use SQL (PostgreSQL)
// Reason: Full ACID required for financial transactions

class BankingSystem {
  async transfer(from: string, to: string, amount: number): Promise<void> {
    const client = await pool.connect();
    
    try {
      await client.query('BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE');
      
      // Atomicity: Both operations or neither
      await client.query('UPDATE accounts SET balance = balance - $1 WHERE id = $2', [amount, from]);
      await client.query('UPDATE accounts SET balance = balance + $1 WHERE id = $2', [amount, to]);
      
      // Consistency: Constraints enforced
      // Isolation: No other transaction can interfere
      // Durability: Committed to disk
      
      await client.query('COMMIT');
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    }
  }
}
```

---

#### Example 2: Social Media Platform

```typescript
// ✅ Use NoSQL (MongoDB + Redis)
// Reason: Flexible schema, high volume, eventual consistency OK

class SocialMediaPlatform {
  // MongoDB for posts (flexible content)
  async createPost(post: any): Promise<string> {
    // Posts can have text, images, videos, polls, etc.
    // Schema varies by post type
    return await mongodb.collection('posts').insertOne({
      userId: post.userId,
      content: post.content, // Flexible structure
      likes: 0,
      createdAt: new Date()
    });
  }
  
  // Redis for like counter (high performance)
  async likePost(postId: string): Promise<void> {
    // Eventual consistency is OK for likes
    await redis.incr(`post:${postId}:likes`);
    
    // Sync to MongoDB periodically (background job)
  }
  
  // Cassandra for activity feed (high volume)
  async logActivity(userId: string, activity: any): Promise<void> {
    await cassandra.execute(
      'INSERT INTO user_feed (user_id, timestamp, activity) VALUES (?, ?, ?)',
      [userId, Date.now(), activity]
    );
  }
}
```

---

#### Example 3: E-Commerce Platform (Hybrid)

```typescript
// ✅ Use Both SQL + NoSQL
// Reason: Different data types have different requirements

class ECommerce {
  // PostgreSQL for orders and payments (ACID critical)
  async checkout(cart: Cart): Promise<string> {
    const client = await postgres.connect();
    try {
      await client.query('BEGIN');
      
      const orderId = await createOrder(cart);
      await processPayment(cart.total);
      await updateInventory(cart.items);
      
      await client.query('COMMIT');
      return orderId;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    }
  }
  
  // MongoDB for product catalog (flexible attributes)
  async getProducts(category: string): Promise<Product[]> {
    return await mongodb.collection('products').find({
      category: category,
      inStock: true
    }).toArray();
  }
  
  // Redis for shopping cart (temporary data)
  async getCart(userId: string): Promise<Cart> {
    const cart = await redis.get(`cart:${userId}`);
    return cart ? JSON.parse(cart) : { items: [] };
  }
  
  // Elasticsearch for product search (full-text search)
  async searchProducts(query: string): Promise<Product[]> {
    return await elasticsearch.search({
      index: 'products',
      body: {
        query: { match: { name: query } }
      }
    });
  }
}
```

---

### Migration Strategies

#### Migrating from SQL to NoSQL
```typescript
// When you've outgrown SQL due to scale

// 1. Identify bottlenecks
// - High read volume? → Add Redis cache
// - Complex documents? → Move to MongoDB
// - Time-series data? → Move to Cassandra

// 2. Gradual migration
class HybridDataLayer {
  async getUser(userId: string): Promise<User> {
    // Try cache first
    const cached = await redis.get(`user:${userId}`);
    if (cached) return JSON.parse(cached);
    
    // Fall back to SQL
    const user = await postgres.query('SELECT * FROM users WHERE id = $1', [userId]);
    
    // Cache for next time
    await redis.setex(`user:${userId}`, 3600, JSON.stringify(user));
    
    return user;
  }
}
```

#### Migrating from NoSQL to SQL
```typescript
// When you need stronger consistency guarantees

// 1. Data modeling
// Convert documents to normalized tables

// MongoDB document
const orderDoc = {
  orderId: '123',
  customer: { name: 'John', email: 'john@example.com' },
  items: [
    { productId: 'P1', quantity: 2, price: 10 },
    { productId: 'P2', quantity: 1, price: 20 }
  ]
};

// SQL normalized tables
CREATE TABLE orders (
  order_id VARCHAR PRIMARY KEY,
  customer_id VARCHAR REFERENCES customers(customer_id)
);

CREATE TABLE order_items (
  order_id VARCHAR REFERENCES orders(order_id),
  product_id VARCHAR REFERENCES products(product_id),
  quantity INT,
  price DECIMAL
);
```

---

### Performance Comparison

```typescript
// Benchmark: 1 million records

// SQL (PostgreSQL) - Complex query with joins
const sqlResult = await postgres.query(`
  SELECT o.order_id, c.name, SUM(oi.quantity * oi.price) as total
  FROM orders o
  JOIN customers c ON o.customer_id = c.customer_id
  JOIN order_items oi ON o.order_id = oi.order_id
  GROUP BY o.order_id, c.name
  HAVING SUM(oi.quantity * oi.price) > 1000
`);
// Time: 500ms (with proper indexes)

// NoSQL (MongoDB) - Same data, denormalized
const mongoResult = await mongodb.collection('orders').aggregate([
  { $match: { total: { $gt: 1000 } } },
  { $project: { orderId: 1, customerName: 1, total: 1 } }
]).toArray();
// Time: 100ms (no joins, pre-computed total)

// Trade-off: MongoDB faster but requires data duplication
```

---

### Key Decision Factors

```
┌─────────────────────────────────────────────────────────────┐
│              SQL vs NoSQL DECISION TREE                      │
└─────────────────────────────────────────────────────────────┘

Do you need ACID guarantees?
├─ YES → Consider SQL
│   │
│   ├─ Complex relationships? → PostgreSQL ✅
│   ├─ High read volume? → PostgreSQL + Redis cache ✅
│   └─ Need full-text search? → PostgreSQL + Elasticsearch ✅
│
└─ NO → Consider NoSQL
    │
    ├─ Document storage? → MongoDB ✅
    ├─ Key-value only? → Redis ✅
    ├─ Time-series data? → Cassandra ✅
    ├─ Graph data? → Neo4j ✅
    └─ Need both? → Polyglot persistence (SQL + NoSQL) ✅
```

---

### Best Practices

1. **Start with SQL if unsure** - Easier to scale up than fix consistency issues
2. **Use NoSQL for specific problems** - Not as a default choice
3. **Polyglot persistence is OK** - Use the right tool for each job
4. **Plan for consistency model** - Understand eventual vs strong consistency
5. **Test at scale** - Performance characteristics change with volume
6. **Monitor both** - Different databases need different monitoring strategies

---

## Common ACID Violations and Fixes

### ❌ Violation 1: No Transaction
```typescript
// BAD: No atomicity
await debitAccount(fromAccount, 100);
await creditAccount(toAccount, 100); // If this fails, money is lost!
```

### ✅ Fix: Use Transaction
```typescript
// GOOD: Atomic
await client.query('BEGIN');
await debitAccount(fromAccount, 100);
await creditAccount(toAccount, 100);
await client.query('COMMIT');
```

---

### ❌ Violation 2: No Validation
```typescript
// BAD: No consistency check
await db.query('UPDATE accounts SET balance = -100');
```

### ✅ Fix: Add Constraints
```typescript
// GOOD: Database constraint + validation
CREATE TABLE accounts (
  balance DECIMAL(15,2) CHECK (balance >= 0)
);

// Application validation
if (newBalance < 0) {
  throw new Error('Balance cannot be negative');
}
```

---

### ❌ Violation 3: Wrong Isolation Level
```typescript
// BAD: Read uncommitted data
await client.query('BEGIN TRANSACTION ISOLATION LEVEL READ UNCOMMITTED');
```

### ✅ Fix: Use Appropriate Level
```typescript
// GOOD: Read committed or higher
await client.query('BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED');
```

---

### ❌ Violation 4: No Durability
```typescript
// BAD: Disabled fsync
fsync = off // Risk of data loss on crash
```

### ✅ Fix: Enable Durability
```typescript
// GOOD: Enable fsync and replication
fsync = on
synchronous_commit = on
// Set up replication for high availability
```

---

## Testing ACID Properties

```typescript
describe('ACID Compliance', () => {
  describe('Atomicity', () => {
    it('should rollback on error', async () => {
      const initialBalance = await getBalance('ACC123');
      
      try {
        await client.query('BEGIN');
        await debit('ACC123', 100);
        throw new Error('Simulated error');
        await credit('ACC456', 100); // Never executed
        await client.query('COMMIT');
      } catch (error) {
        await client.query('ROLLBACK');
      }
      
      const finalBalance = await getBalance('ACC123');
      expect(finalBalance).toBe(initialBalance); // ✅ Rolled back
    });
  });
  
  describe('Consistency', () => {
    it('should enforce constraints', async () => {
      await expect(
        updateBalance('ACC123', -100) // Violates balance >= 0
      ).rejects.toThrow('constraint');
    });
  });
  
  describe('Isolation', () => {
    it('should prevent dirty reads', async () => {
      // Transaction 1: Update but don't commit
      const tx1 = client.query('BEGIN');
      await client.query('UPDATE accounts SET balance = 1000 WHERE id = 1');
      
      // Transaction 2: Read (should see old value)
      const balance = await otherClient.query('SELECT balance FROM accounts WHERE id = 1');
      expect(balance.rows[0].balance).not.toBe(1000); // ✅ No dirty read
      
      await client.query('ROLLBACK');
    });
  });
  
  describe('Durability', () => {
    it('should survive crash', async () => {
      await client.query('BEGIN');
      await client.query('INSERT INTO test VALUES (1)');
      await client.query('COMMIT');
      
      await killDatabase();
      await restartDatabase();
      
      const result = await client.query('SELECT * FROM test WHERE id = 1');
      expect(result.rowCount).toBe(1); // ✅ Data survived
    });
  });
});
```

---

## Best Practices

### 1. **Always Use Transactions for Related Operations**
```typescript
// ✅ Good
await client.query('BEGIN');
await operation1();
await operation2();
await client.query('COMMIT');
```

### 2. **Validate at Multiple Layers**
```typescript
// ✅ Good: Application + Database validation
// Application
if (amount <= 0) throw new Error('Invalid amount');

// Database
CREATE TABLE accounts (
  balance DECIMAL CHECK (balance >= 0)
);
```

### 3. **Choose Appropriate Isolation Level**
```typescript
// ✅ Good: Match isolation to use case
// Read-heavy: READ COMMITTED
// Critical operations: SERIALIZABLE
await client.query('BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE');
```

### 4. **Enable Durability Features**
```typescript
// ✅ Good: Production settings
fsync = on
synchronous_commit = on
wal_level = replica
```

### 5. **Handle Errors Properly**
```typescript
// ✅ Good: Always rollback on error
try {
  await client.query('BEGIN');
  await operations();
  await client.query('COMMIT');
} catch (error) {
  await client.query('ROLLBACK');
  throw error;
}
```

### 6. **Keep Transactions Short**
```typescript
// ✅ Good: Minimize lock time
await client.query('BEGIN');
await fastDatabaseOperations();
await client.query('COMMIT');

// ❌ Bad: Long transaction
await client.query('BEGIN');
await callExternalAPI(); // 5 seconds!
await client.query('COMMIT');
```

### 7. **Monitor and Alert**
```typescript
// ✅ Good: Monitor ACID metrics
- Transaction rollback rate
- Lock wait time
- Deadlock frequency
- Replication lag
- WAL size
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│                    ACID QUICK REFERENCE                      │
├─────────────────────────────────────────────────────────────┤
│ ATOMICITY                                                    │
│  ✓ Use BEGIN/COMMIT/ROLLBACK                                │
│  ✓ Wrap related operations in transaction                   │
│  ✓ Always handle errors with rollback                       │
├─────────────────────────────────────────────────────────────┤
│ CONSISTENCY                                                  │
│  ✓ Define database constraints (CHECK, FK, UNIQUE)          │
│  ✓ Validate in application layer                            │
│  ✓ Use triggers for complex rules                           │
├─────────────────────────────────────────────────────────────┤
│ ISOLATION                                                    │
│  ✓ Choose appropriate isolation level                       │
│  ✓ Use SELECT FOR UPDATE for critical sections              │
│  ✓ Lock resources in consistent order                       │
│  ✓ Handle deadlocks with retry logic                        │
├─────────────────────────────────────────────────────────────┤
│ DURABILITY                                                   │
│  ✓ Enable fsync and synchronous_commit                      │
│  ✓ Set up replication for HA                                │
│  ✓ Implement regular backups                                │
│  ✓ Test crash recovery scenarios                            │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Preparation Checklist

- [ ] Explain each ACID property with examples
- [ ] Describe isolation levels and their trade-offs
- [ ] Explain Write-Ahead Logging (WAL)
- [ ] Discuss optimistic vs pessimistic locking
- [ ] Understand deadlock prevention strategies
- [ ] Know when to use SERIALIZABLE vs READ COMMITTED
- [ ] Explain crash recovery process
- [ ] Describe replication types (sync vs async)
- [ ] Understand ACID in distributed systems (Saga pattern)
- [ ] Know ACID support in different databases (SQL vs NoSQL)
- [ ] Explain CAP theorem and ACID trade-offs
- [ ] Describe testing strategies for ACID properties

---

## Additional Resources

### Detailed Guides
- [1. Atomicity](./1_Atomicity.md) - All-or-nothing transactions
- [2. Consistency](./2_Consistency.md) - Valid state maintenance
- [3. Isolation](./3_Isolation.md) - Concurrent transaction handling
- [4. Durability](./4_Durability.md) - Permanent data persistence

### Related Topics
- CAP Theorem
- BASE vs ACID
- Two-Phase Commit (2PC)
- Saga Pattern
- Event Sourcing
- CQRS

---

## Summary

**ACID** properties are fundamental to reliable database systems:

1. **Atomicity**: All or nothing - operations succeed together or fail together
2. **Consistency**: Database always in valid state - rules never violated
3. **Isolation**: Concurrent transactions don't interfere - independent execution
4. **Durability**: Committed data is permanent - survives all failures

**Key Takeaway**: ACID properties work together to ensure data integrity, reliability, and consistency in database systems. Understanding and properly implementing these properties is crucial for building robust applications, especially those dealing with critical data like financial transactions, healthcare records, or e-commerce systems.

**When in doubt**: Use transactions, validate data, choose appropriate isolation, enable durability features, and test thoroughly!
