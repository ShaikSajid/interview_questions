# Isolation Principle

## Definition
**Isolation** ensures that concurrent transactions execute independently without interfering with each other. Each transaction should be isolated from others until it's complete, preventing dirty reads, non-repeatable reads, and phantom reads.

**Key Concept**: "Concurrent transactions don't interfere" - Multiple transactions can run simultaneously as if they were running sequentially.

---

## Real-World Analogy
Think of isolation like ATM booths. When you're using an ATM, other users can't see or modify your transaction in progress. Each person works in isolation, even though multiple people might be using ATMs at the same time. You only see the final result after they're done.

---

## Understanding Isolation Levels

### 1. Read Uncommitted (Lowest Isolation)
- Can read uncommitted changes from other transactions
- **Problem**: Dirty reads

### 2. Read Committed (Default in most databases)
- Only reads committed data
- **Problem**: Non-repeatable reads

### 3. Repeatable Read
- Same query returns same results within transaction
- **Problem**: Phantom reads

### 4. Serializable (Highest Isolation)
- Transactions appear to run serially
- **Problem**: Performance overhead

---

## Isolation Problems Explained

### Problem 1: Dirty Read

```typescript
// ❌ Read Uncommitted - DANGEROUS!

// Transaction A (not committed yet)
await db.query('BEGIN');
await db.query('UPDATE accounts SET balance = 1000 WHERE account_id = $1', ['ACC123']);
// Not committed yet!

// Transaction B reads uncommitted data
const result = await db.query('SELECT balance FROM accounts WHERE account_id = $1', ['ACC123']);
console.log(result.rows[0].balance); // 1000 (DIRTY READ!)

// Transaction A rolls back
await db.query('ROLLBACK'); // Balance is actually still 500!

// Transaction B made decisions based on wrong data!
```

---

### Problem 2: Non-Repeatable Read

```typescript
// ❌ Read Committed - Data changes between reads

// Transaction A
await client.query('BEGIN');

// First read
const read1 = await client.query('SELECT balance FROM accounts WHERE account_id = $1', ['ACC123']);
console.log('First read:', read1.rows[0].balance); // 500

// Transaction B commits a change
await otherClient.query('UPDATE accounts SET balance = 1000 WHERE account_id = $1', ['ACC123']);

// Second read in same transaction
const read2 = await client.query('SELECT balance FROM accounts WHERE account_id = $1', ['ACC123']);
console.log('Second read:', read2.rows[0].balance); // 1000 (DIFFERENT!)

await client.query('COMMIT');
```

---

### Problem 3: Phantom Read

```typescript
// ❌ Repeatable Read - New rows appear

// Transaction A
await client.query('BEGIN');

// First query
const read1 = await client.query('SELECT * FROM accounts WHERE balance > 1000');
console.log('Count:', read1.rowCount); // 5 accounts

// Transaction B inserts new row
await otherClient.query('INSERT INTO accounts (account_id, balance) VALUES ($1, $2)', ['ACC999', 2000]);

// Second query in same transaction
const read2 = await client.query('SELECT * FROM accounts WHERE balance > 1000');
console.log('Count:', read2.rowCount); // 6 accounts (PHANTOM!)

await client.query('COMMIT');
```

---

## Isolation Levels in Practice

### ✅ Read Committed (Most Common)

```typescript
// PostgreSQL default: READ COMMITTED
async function transferMoney(fromAccount: string, toAccount: string, amount: number) {
  const client = await pool.connect();
  
  try {
    // Isolation level is READ COMMITTED by default
    await client.query('BEGIN');
    
    // Read only committed data
    const fromBalance = await client.query(
      'SELECT balance FROM accounts WHERE account_id = $1',
      [fromAccount]
    );
    
    if (fromBalance.rows[0].balance < amount) {
      throw new Error('Insufficient funds');
    }
    
    // Perform transfer
    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE account_id = $2',
      [amount, fromAccount]
    );
    
    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE account_id = $2',
      [amount, toAccount]
    );
    
    await client.query('COMMIT');
    
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

---

### ✅ Repeatable Read (When Consistency Matters)

```typescript
// Set isolation level to REPEATABLE READ
async function generateAccountStatement(accountId: string): Promise<Statement> {
  const client = await pool.connect();
  
  try {
    // Set isolation level
    await client.query('BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ');
    
    // All reads in this transaction will be consistent
    const account = await client.query(
      'SELECT * FROM accounts WHERE account_id = $1',
      [accountId]
    );
    
    const transactions = await client.query(
      'SELECT * FROM transactions WHERE account_id = $1 ORDER BY timestamp DESC LIMIT 50',
      [accountId]
    );
    
    const summary = await client.query(
      'SELECT COUNT(*), SUM(amount) FROM transactions WHERE account_id = $1',
      [accountId]
    );
    
    // Even if other transactions are updating this account,
    // all our reads show the same consistent snapshot
    
    await client.query('COMMIT');
    
    return {
      account: account.rows[0],
      transactions: transactions.rows,
      summary: summary.rows[0]
    };
    
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

---

### ✅ Serializable (Maximum Isolation)

```typescript
// Set isolation level to SERIALIZABLE
async function bookConcertSeats(eventId: string, seats: string[], userId: string): Promise<string> {
  const client = await pool.connect();
  
  try {
    // Highest isolation level
    await client.query('BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE');
    
    const bookingId = generateId();
    
    // Check all seats are available
    for (const seatNumber of seats) {
      const seat = await client.query(
        'SELECT status FROM seats WHERE event_id = $1 AND seat_number = $2',
        [eventId, seatNumber]
      );
      
      if (seat.rowCount === 0 || seat.rows[0].status !== 'AVAILABLE') {
        throw new Error(`Seat ${seatNumber} not available`);
      }
      
      // Book seat
      await client.query(
        'UPDATE seats SET status = $1, booking_id = $2, user_id = $3 WHERE event_id = $4 AND seat_number = $5',
        ['BOOKED', bookingId, userId, eventId, seatNumber]
      );
    }
    
    // If another transaction tries to book same seats,
    // one will be rolled back due to serialization conflict
    
    await client.query('COMMIT');
    return bookingId;
    
  } catch (error) {
    if (error.code === '40001') { // Serialization failure
      console.log('Serialization conflict, retrying...');
      // Implement retry logic
    }
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

---

## Using Row-Level Locking

### SELECT FOR UPDATE (Pessimistic Locking)

```typescript
// Lock rows for update
async function reserveInventory(productId: string, quantity: number): Promise<void> {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // Lock the row - no other transaction can modify it
    const product = await client.query(
      'SELECT * FROM inventory WHERE product_id = $1 FOR UPDATE',
      [productId]
    );
    
    if (product.rowCount === 0) {
      throw new Error('Product not found');
    }
    
    const availableQty = product.rows[0].quantity;
    
    if (availableQty < quantity) {
      throw new Error('Insufficient inventory');
    }
    
    // Update inventory
    await client.query(
      'UPDATE inventory SET quantity = quantity - $1 WHERE product_id = $2',
      [quantity, productId]
    );
    
    await client.query('COMMIT');
    
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

---

### SELECT FOR UPDATE NOWAIT (Non-Blocking)

```typescript
// Don't wait for lock, fail immediately
async function quickCheckout(productId: string, quantity: number): Promise<boolean> {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // Try to lock, but don't wait
    const product = await client.query(
      'SELECT * FROM inventory WHERE product_id = $1 FOR UPDATE NOWAIT',
      [productId]
    );
    
    // If we get here, we have the lock
    const availableQty = product.rows[0].quantity;
    
    if (availableQty < quantity) {
      throw new Error('Insufficient inventory');
    }
    
    await client.query(
      'UPDATE inventory SET quantity = quantity - $1 WHERE product_id = $2',
      [quantity, productId]
    );
    
    await client.query('COMMIT');
    return true;
    
  } catch (error) {
    if (error.code === '55P03') { // Lock not available
      console.log('Product locked by another transaction');
      return false; // Fail fast
    }
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

---

### SELECT FOR UPDATE SKIP LOCKED

```typescript
// Process queue items without blocking
async function processNextTask(): Promise<Task | null> {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // Get next available task, skip locked ones
    const result = await client.query(`
      SELECT * FROM tasks 
      WHERE status = 'PENDING' 
      ORDER BY created_at 
      LIMIT 1 
      FOR UPDATE SKIP LOCKED
    `);
    
    if (result.rowCount === 0) {
      await client.query('COMMIT');
      return null; // No tasks available
    }
    
    const task = result.rows[0];
    
    // Mark as processing
    await client.query(
      'UPDATE tasks SET status = $1, started_at = NOW() WHERE task_id = $2',
      ['PROCESSING', task.task_id]
    );
    
    await client.query('COMMIT');
    return task;
    
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

---

## Optimistic Locking (Version-Based)

```typescript
interface Account {
  accountId: string;
  balance: number;
  version: number;
}

// Optimistic concurrency control using version numbers
async function updateAccountOptimistic(
  accountId: string,
  newBalance: number,
  expectedVersion: number
): Promise<void> {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // Update only if version matches
    const result = await client.query(
      `UPDATE accounts 
       SET balance = $1, version = version + 1, updated_at = NOW()
       WHERE account_id = $2 AND version = $3
       RETURNING version`,
      [newBalance, accountId, expectedVersion]
    );
    
    if (result.rowCount === 0) {
      throw new Error('Concurrent modification detected - version mismatch');
    }
    
    await client.query('COMMIT');
    
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}

// Usage with retry
async function withdrawWithOptimisticLocking(accountId: string, amount: number): Promise<void> {
  const maxRetries = 3;
  let attempt = 0;
  
  while (attempt < maxRetries) {
    try {
      // Read current state
      const account = await getAccount(accountId);
      
      if (account.balance < amount) {
        throw new Error('Insufficient funds');
      }
      
      // Try to update with version check
      await updateAccountOptimistic(
        accountId,
        account.balance - amount,
        account.version
      );
      
      return; // Success!
      
    } catch (error) {
      if (error.message.includes('version mismatch')) {
        attempt++;
        console.log(`Retry ${attempt}/${maxRetries}`);
        await sleep(100 * attempt); // Exponential backoff
      } else {
        throw error;
      }
    }
  }
  
  throw new Error('Max retries exceeded');
}
```

---

## Handling Deadlocks

```typescript
// Deadlock scenario
async function transferWithDeadlockHandling(
  fromAccount: string,
  toAccount: string,
  amount: number
): Promise<void> {
  const maxRetries = 3;
  let attempt = 0;
  
  while (attempt < maxRetries) {
    const client = await pool.connect();
    
    try {
      await client.query('BEGIN');
      
      // Lock accounts in consistent order to prevent deadlock
      const [first, second] = [fromAccount, toAccount].sort();
      
      await client.query(
        'SELECT * FROM accounts WHERE account_id = $1 FOR UPDATE',
        [first]
      );
      
      await client.query(
        'SELECT * FROM accounts WHERE account_id = $1 FOR UPDATE',
        [second]
      );
      
      // Perform transfer
      const balance = await client.query(
        'SELECT balance FROM accounts WHERE account_id = $1',
        [fromAccount]
      );
      
      if (balance.rows[0].balance < amount) {
        throw new Error('Insufficient funds');
      }
      
      await client.query(
        'UPDATE accounts SET balance = balance - $1 WHERE account_id = $2',
        [amount, fromAccount]
      );
      
      await client.query(
        'UPDATE accounts SET balance = balance + $1 WHERE account_id = $2',
        [amount, toAccount]
      );
      
      await client.query('COMMIT');
      return; // Success
      
    } catch (error) {
      await client.query('ROLLBACK');
      
      if (error.code === '40P01') { // Deadlock detected
        attempt++;
        console.log(`Deadlock detected, retry ${attempt}/${maxRetries}`);
        await sleep(Math.random() * 100 * attempt);
      } else {
        throw error;
      }
    } finally {
      client.release();
    }
  }
  
  throw new Error('Max retries exceeded due to deadlocks');
}
```

---

## Isolation in Distributed Systems

```typescript
// Distributed transaction coordination
interface DistributedTransaction {
  transactionId: string;
  services: string[];
  status: 'PENDING' | 'COMMITTED' | 'ABORTED';
}

// Saga pattern for distributed isolation
class SagaCoordinator {
  async executeSaga(steps: SagaStep[]): Promise<void> {
    const compensations: (() => Promise<void>)[] = [];
    
    try {
      for (const step of steps) {
        // Execute step
        await step.execute();
        
        // Save compensation
        compensations.unshift(step.compensate);
      }
      
      // All steps succeeded
      console.log('Saga completed successfully');
      
    } catch (error) {
      console.error('Saga failed, compensating...', error);
      
      // Execute compensations in reverse order
      for (const compensate of compensations) {
        try {
          await compensate();
        } catch (compError) {
          console.error('Compensation failed:', compError);
          // Log for manual intervention
        }
      }
      
      throw error;
    }
  }
}

interface SagaStep {
  execute: () => Promise<void>;
  compensate: () => Promise<void>;
}

// Example: Distributed order processing
async function processDistributedOrder(order: Order): Promise<void> {
  const saga = new SagaCoordinator();
  
  const steps: SagaStep[] = [
    {
      execute: async () => {
        await inventoryService.reserve(order.items);
      },
      compensate: async () => {
        await inventoryService.release(order.items);
      }
    },
    {
      execute: async () => {
        await paymentService.charge(order.customerId, order.total);
      },
      compensate: async () => {
        await paymentService.refund(order.customerId, order.total);
      }
    },
    {
      execute: async () => {
        await orderService.create(order);
      },
      compensate: async () => {
        await orderService.cancel(order.orderId);
      }
    }
  ];
  
  await saga.executeSaga(steps);
}
```

---

## Interview Questions & Answers

### Q1: What is isolation in ACID properties?
**Answer**: Isolation ensures that concurrent transactions execute independently without interfering with each other. Each transaction is isolated from others until it completes, preventing issues like dirty reads, non-repeatable reads, and phantom reads.

**Example**: Two users withdrawing from the same account simultaneously shouldn't see each other's uncommitted changes or cause race conditions.

---

### Q2: What are the different isolation levels?
**Answer**: Four standard isolation levels (increasing isolation):

1. **Read Uncommitted**: Can read uncommitted changes (dirty reads)
2. **Read Committed**: Only reads committed data (default in most DBs)
3. **Repeatable Read**: Same query returns same results in transaction
4. **Serializable**: Transactions appear to run sequentially (highest isolation)

**Trade-off**: Higher isolation = Better consistency but lower concurrency

---

### Q3: What is a dirty read and how to prevent it?
**Answer**: **Dirty read** occurs when a transaction reads uncommitted changes from another transaction.

**Problem**:
```typescript
// Transaction A updates but doesn't commit
UPDATE accounts SET balance = 1000;
// Transaction B reads 1000 (dirty read)
SELECT balance FROM accounts; // 1000
// Transaction A rolls back
ROLLBACK; // Balance is actually 500!
```

**Prevention**: Use **Read Committed** or higher isolation level (most databases default to this).

---

### Q4: What is the difference between pessimistic and optimistic locking?
**Answer**:

**Pessimistic Locking**: Lock immediately, preventing concurrent access
```typescript
// Lock row immediately
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
// Others must wait
```
- **Use when**: High contention expected
- **Pros**: Prevents conflicts
- **Cons**: Can cause blocking

**Optimistic Locking**: No lock, check version before commit
```typescript
// Read with version
SELECT balance, version FROM accounts WHERE id = 1; // balance=100, version=5

// Later, update only if version matches
UPDATE accounts SET balance = 50, version = 6 
WHERE id = 1 AND version = 5;
// Fails if version changed
```
- **Use when**: Low contention expected
- **Pros**: Better performance
- **Cons**: May need retries

---

### Q5: What causes deadlocks and how to prevent them?
**Answer**: **Deadlock** occurs when transactions wait for each other's locks in a circular manner.

**Example**:
```typescript
// Transaction A locks Account 1, wants Account 2
// Transaction B locks Account 2, wants Account 1
// Both wait forever!
```

**Prevention strategies**:
1. **Lock in consistent order**: Always lock resources in same order
```typescript
const [first, second] = [acc1, acc2].sort();
await lock(first);
await lock(second);
```

2. **Use timeouts**: 
```typescript
SET lock_timeout = '5s';
```

3. **Use NOWAIT**: Fail immediately instead of waiting
```typescript
SELECT * FROM accounts FOR UPDATE NOWAIT;
```

4. **Keep transactions short**: Release locks quickly

---

### Q6: When should you use SERIALIZABLE isolation level?
**Answer**: Use **SERIALIZABLE** when:

1. **Critical financial operations**: Prevent any anomalies
2. **Inventory management**: Prevent overselling
3. **Sequential processing**: Maintain strict ordering
4. **Compliance requirements**: Audit trail demands

**Example**:
```typescript
// Booking last seat - must be serializable
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT available_seats FROM flights WHERE flight_id = 'FL123';
// If 1 seat left, two transactions can't both book it
UPDATE flights SET available_seats = available_seats - 1;
COMMIT;
```

**When NOT to use**: High-traffic read-heavy operations (use lower isolation levels)

---

### Q7: What is SELECT FOR UPDATE and when to use it?
**Answer**: **SELECT FOR UPDATE** locks rows for updating, preventing other transactions from modifying them.

**Use cases**:
1. **Reserving resources**: Inventory, seats, etc.
2. **Queue processing**: Prevent duplicate processing
3. **Sequential operations**: Ensure consistency

**Variants**:
- `FOR UPDATE`: Wait for lock
- `FOR UPDATE NOWAIT`: Fail immediately if locked
- `FOR UPDATE SKIP LOCKED`: Skip locked rows

**Example**:
```typescript
// Process next available task
SELECT * FROM tasks 
WHERE status = 'PENDING' 
LIMIT 1 
FOR UPDATE SKIP LOCKED;
```

---

### Q8: How do you test isolation in your application?
**Answer**: Testing strategies:

1. **Concurrent transaction tests**:
```typescript
it('should handle concurrent withdrawals', async () => {
  const promises = [];
  for (let i = 0; i < 10; i++) {
    promises.push(withdraw('ACC123', 10));
  }
  await Promise.all(promises);
  
  const balance = await getBalance('ACC123');
  expect(balance).toBe(initialBalance - 100);
});
```

2. **Dirty read tests**: Verify uncommitted data isn't visible

3. **Deadlock simulation**: Force circular lock waits

4. **Race condition tests**: Rapid concurrent updates

5. **Load testing**: Stress test under high concurrency

---

### Q9: How does isolation work in NoSQL databases?
**Answer**: NoSQL databases have varying isolation guarantees:

**MongoDB**:
- Single document operations are isolated
- Multi-document transactions (v4.0+) support isolation levels

**Cassandra**:
- Row-level atomicity
- Lightweight transactions (LWT) for linearizable operations
- Eventual consistency by default

**DynamoDB**:
- ACID transactions for up to 25 items
- Conditional writes for optimistic locking

**Redis**:
- Single commands are atomic
- MULTI/EXEC for transaction-like behavior (not true isolation)

**Trade-off**: NoSQL often sacrifices isolation for availability and partition tolerance (CAP theorem)

---

### Q10: What are common isolation mistakes?
**Answer**:

1. **Not using transactions**:
```typescript
// ❌ BAD - No isolation
await update1();
await update2(); // Another transaction can interfere!
```

2. **Wrong isolation level**:
```typescript
// ❌ Using READ UNCOMMITTED in production
BEGIN TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```

3. **Not handling deadlocks**:
```typescript
// ❌ No retry logic for deadlocks
try {
  await transfer();
} catch (error) {
  throw error; // Should retry on deadlock!
}
```

4. **Long-running transactions**:
```typescript
// ❌ Holding locks too long
BEGIN;
SELECT FOR UPDATE; // Locks row
await callExternalAPI(); // 5 seconds!
UPDATE;
COMMIT; // Finally releases lock
```

5. **Inconsistent lock ordering**: Causes deadlocks

---

## Scenario-Based Problems

### Scenario 1: Concurrent Seat Booking
**Problem**: 100 people trying to book 10 remaining seats simultaneously.

**Solution**:
```typescript
async function bookSeat(eventId: string, seatNumber: string, userId: string): Promise<boolean> {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE');
    
    // Lock seat row
    const seat = await client.query(
      'SELECT * FROM seats WHERE event_id = $1 AND seat_number = $2 FOR UPDATE NOWAIT',
      [eventId, seatNumber]
    );
    
    if (seat.rowCount === 0) {
      throw new Error('Seat not found');
    }
    
    if (seat.rows[0].status !== 'AVAILABLE') {
      throw new Error('Seat already booked');
    }
    
    // Book seat
    await client.query(
      'UPDATE seats SET status = $1, booked_by = $2, booked_at = NOW() WHERE event_id = $3 AND seat_number = $4',
      ['BOOKED', userId, eventId, seatNumber]
    );
    
    await client.query('COMMIT');
    return true;
    
  } catch (error) {
    await client.query('ROLLBACK');
    if (error.code === '55P03') { // Lock not available
      return false; // Seat being booked by someone else
    }
    throw error;
  } finally {
    client.release();
  }
}
```

---

### Scenario 2: Inventory Management
**Problem**: Prevent overselling when stock is low and many orders arrive simultaneously.

**Solution**:
```typescript
async function createOrderWithInventory(
  productId: string,
  quantity: number,
  customerId: string
): Promise<string> {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // Lock inventory row
    const inventory = await client.query(
      'SELECT quantity, reserved FROM inventory WHERE product_id = $1 FOR UPDATE',
      [productId]
    );
    
    if (inventory.rowCount === 0) {
      throw new Error('Product not found');
    }
    
    const available = inventory.rows[0].quantity - inventory.rows[0].reserved;
    
    if (available < quantity) {
      throw new Error(`Only ${available} items available`);
    }
    
    // Reserve inventory
    await client.query(
      'UPDATE inventory SET reserved = reserved + $1 WHERE product_id = $2',
      [quantity, productId]
    );
    
    // Create order
    const orderId = generateId();
    await client.query(
      'INSERT INTO orders (order_id, customer_id, status) VALUES ($1, $2, $3)',
      [orderId, customerId, 'PENDING']
    );
    
    await client.query(
      'INSERT INTO order_items (order_id, product_id, quantity) VALUES ($1, $2, $3)',
      [orderId, productId, quantity]
    );
    
    await client.query('COMMIT');
    return orderId;
    
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

---

### Scenario 3: Task Queue Processing
**Problem**: Multiple workers processing same queue without duplicates.

**Solution**:
```typescript
// Worker processes tasks without conflicts
async function processNextTask(workerId: string): Promise<void> {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // Get next available task, skip locked ones
    const result = await client.query(`
      SELECT * FROM task_queue 
      WHERE status = 'PENDING' 
      ORDER BY priority DESC, created_at ASC 
      LIMIT 1 
      FOR UPDATE SKIP LOCKED
    `);
    
    if (result.rowCount === 0) {
      await client.query('COMMIT');
      return; // No tasks available
    }
    
    const task = result.rows[0];
    
    // Mark as processing
    await client.query(
      'UPDATE task_queue SET status = $1, worker_id = $2, started_at = NOW() WHERE task_id = $3',
      ['PROCESSING', workerId, task.task_id]
    );
    
    await client.query('COMMIT');
    
    // Process task outside transaction
    try {
      await executeTask(task);
      
      // Mark complete
      await markTaskComplete(task.task_id);
    } catch (error) {
      // Mark failed
      await markTaskFailed(task.task_id, error.message);
    }
    
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

---

## Key Takeaways

1. ✅ **Choose appropriate isolation level** for your use case
2. ✅ **Use pessimistic locking** (FOR UPDATE) for high contention
3. ✅ **Use optimistic locking** (versioning) for low contention
4. ✅ **Lock resources in consistent order** to prevent deadlocks
5. ✅ **Keep transactions short** to reduce lock contention
6. ✅ **Handle deadlocks gracefully** with retry logic
7. ✅ **Test concurrent scenarios** thoroughly
8. ✅ **Monitor lock waits** and deadlocks in production

**Remember**: Isolation prevents chaos in concurrent systems. Choose the right level for your requirements!
