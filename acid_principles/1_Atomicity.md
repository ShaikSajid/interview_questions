# Atomicity Principle

## Definition
**Atomicity** ensures that a transaction is treated as a single, indivisible unit of work. Either all operations within the transaction complete successfully, or none of them do. There is no partial completion.

**Key Concept**: "All or Nothing" - If any part of the transaction fails, the entire transaction is rolled back to its initial state.

---

## Real-World Analogy
Think of atomicity like ordering a combo meal at a restaurant. You order a burger, fries, and a drink as one order. If the kitchen runs out of fries, they don't serve you just the burger and drink - they either give you the complete combo or cancel the entire order.

---

## Banking Transfer Example

### ❌ Without Atomicity (Bad Practice)

```typescript
// Non-atomic transfer - DANGEROUS!
async function transferMoney(fromAccount: string, toAccount: string, amount: number) {
  // Step 1: Deduct from sender
  await db.query(
    'UPDATE accounts SET balance = balance - $1 WHERE account_id = $2',
    [amount, fromAccount]
  );
  
  // ⚠️ PROBLEM: If system crashes here, money is lost!
  // The sender's account is debited but receiver never gets credited
  
  // Step 2: Credit to receiver
  await db.query(
    'UPDATE accounts SET balance = balance + $1 WHERE account_id = $2',
    [amount, toAccount]
  );
}
```

**Problem**: If the system crashes between the two operations, money disappears from the sender's account but never appears in the receiver's account!

---

### ✅ With Atomicity (Good Practice)

```typescript
// Atomic transfer using database transaction
async function transferMoney(fromAccount: string, toAccount: string, amount: number) {
  const client = await pool.connect();
  
  try {
    // Start transaction
    await client.query('BEGIN');
    
    // Step 1: Deduct from sender
    const debitResult = await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE account_id = $2 AND balance >= $1 RETURNING balance',
      [amount, fromAccount]
    );
    
    if (debitResult.rowCount === 0) {
      throw new Error('Insufficient funds or account not found');
    }
    
    // Step 2: Credit to receiver
    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE account_id = $2',
      [amount, toAccount]
    );
    
    // Step 3: Log transaction
    await client.query(
      'INSERT INTO transaction_log (from_account, to_account, amount, timestamp) VALUES ($1, $2, $3, NOW())',
      [fromAccount, toAccount, amount]
    );
    
    // Commit transaction - All operations succeed together
    await client.query('COMMIT');
    console.log('Transfer completed successfully');
    
  } catch (error) {
    // Rollback - Undo all operations
    await client.query('ROLLBACK');
    console.error('Transfer failed, all changes rolled back:', error);
    throw error;
  } finally {
    client.release();
  }
}
```

**Benefits**:
- ✅ If any operation fails, ALL changes are rolled back
- ✅ Database guarantees consistency
- ✅ No partial state possible

---

## E-Commerce Order Example

### Scenario: Processing an Online Order

```typescript
interface OrderItem {
  productId: string;
  quantity: number;
  price: number;
}

interface Order {
  orderId: string;
  customerId: string;
  items: OrderItem[];
  totalAmount: number;
}

// ✅ Atomic order processing
async function processOrder(order: Order): Promise<string> {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // 1. Create order record
    const orderResult = await client.query(
      'INSERT INTO orders (order_id, customer_id, total_amount, status, created_at) VALUES ($1, $2, $3, $4, NOW()) RETURNING order_id',
      [order.orderId, order.customerId, order.totalAmount, 'PENDING']
    );
    
    // 2. Reserve inventory for each item
    for (const item of order.items) {
      const inventoryResult = await client.query(
        'UPDATE inventory SET quantity = quantity - $1 WHERE product_id = $2 AND quantity >= $1 RETURNING quantity',
        [item.quantity, item.productId]
      );
      
      if (inventoryResult.rowCount === 0) {
        throw new Error(`Insufficient inventory for product ${item.productId}`);
      }
      
      // 3. Create order items
      await client.query(
        'INSERT INTO order_items (order_id, product_id, quantity, price) VALUES ($1, $2, $3, $4)',
        [order.orderId, item.productId, item.quantity, item.price]
      );
    }
    
    // 4. Process payment
    const paymentResult = await client.query(
      'INSERT INTO payments (order_id, customer_id, amount, status) VALUES ($1, $2, $3, $4) RETURNING payment_id',
      [order.orderId, order.customerId, order.totalAmount, 'COMPLETED']
    );
    
    // 5. Update order status
    await client.query(
      'UPDATE orders SET status = $1, payment_id = $2 WHERE order_id = $3',
      ['CONFIRMED', paymentResult.rows[0].payment_id, order.orderId]
    );
    
    // All operations succeeded - commit transaction
    await client.query('COMMIT');
    
    return `Order ${order.orderId} processed successfully`;
    
  } catch (error) {
    // Any failure rolls back ALL operations:
    // - Order is not created
    // - Inventory is not reduced
    // - Payment is not processed
    await client.query('ROLLBACK');
    throw new Error(`Order processing failed: ${error.message}`);
  } finally {
    client.release();
  }
}
```

---

## Flight Booking System Example

```typescript
interface FlightBooking {
  bookingId: string;
  passengerId: string;
  flightId: string;
  seatNumber: string;
  price: number;
}

async function bookFlight(booking: FlightBooking): Promise<void> {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // 1. Check seat availability and lock it
    const seatCheck = await client.query(
      'SELECT * FROM seats WHERE flight_id = $1 AND seat_number = $2 AND status = $3 FOR UPDATE',
      [booking.flightId, booking.seatNumber, 'AVAILABLE']
    );
    
    if (seatCheck.rowCount === 0) {
      throw new Error('Seat not available');
    }
    
    // 2. Mark seat as booked
    await client.query(
      'UPDATE seats SET status = $1, booked_by = $2 WHERE flight_id = $3 AND seat_number = $4',
      ['BOOKED', booking.passengerId, booking.flightId, booking.seatNumber]
    );
    
    // 3. Create booking record
    await client.query(
      'INSERT INTO bookings (booking_id, passenger_id, flight_id, seat_number, price, booking_date) VALUES ($1, $2, $3, $4, $5, NOW())',
      [booking.bookingId, booking.passengerId, booking.flightId, booking.seatNumber, booking.price]
    );
    
    // 4. Charge customer
    await client.query(
      'INSERT INTO charges (booking_id, passenger_id, amount, status) VALUES ($1, $2, $3, $4)',
      [booking.bookingId, booking.passengerId, booking.price, 'SUCCESS']
    );
    
    // 5. Send confirmation email (log the intent)
    await client.query(
      'INSERT INTO email_queue (recipient_id, email_type, booking_id) VALUES ($1, $2, $3)',
      [booking.passengerId, 'BOOKING_CONFIRMATION', booking.bookingId]
    );
    
    await client.query('COMMIT');
    console.log('Flight booked successfully');
    
  } catch (error) {
    await client.query('ROLLBACK');
    console.error('Booking failed, all changes reverted:', error);
    throw error;
  } finally {
    client.release();
  }
}
```

---

## MongoDB Transaction Example

```typescript
import { MongoClient } from 'mongodb';

async function transferBetweenAccounts() {
  const client = new MongoClient('mongodb://localhost:27017');
  
  try {
    await client.connect();
    const session = client.startSession();
    
    try {
      // Start transaction
      session.startTransaction();
      
      const db = client.db('banking');
      const accounts = db.collection('accounts');
      
      // Debit from account A
      await accounts.updateOne(
        { accountId: 'A123', balance: { $gte: 1000 } },
        { $inc: { balance: -1000 } },
        { session }
      );
      
      // Credit to account B
      await accounts.updateOne(
        { accountId: 'B456' },
        { $inc: { balance: 1000 } },
        { session }
      );
      
      // Commit transaction
      await session.commitTransaction();
      console.log('Transfer completed');
      
    } catch (error) {
      // Abort transaction
      await session.abortTransaction();
      console.error('Transfer failed:', error);
      throw error;
    } finally {
      await session.endSession();
    }
  } finally {
    await client.close();
  }
}
```

---

## Interview Questions & Answers

### Q1: What is atomicity in ACID properties?
**Answer**: Atomicity ensures that a transaction is treated as a single unit of work - either all operations succeed and are committed, or all operations fail and are rolled back. There's no partial completion. It's the "all or nothing" principle that maintains database consistency.

**Example**: In a bank transfer, if deducting money from one account succeeds but adding to another fails, atomicity ensures the deduction is rolled back.

---

### Q2: How do you implement atomicity in a relational database?
**Answer**: Use database transactions with BEGIN, COMMIT, and ROLLBACK statements:
- `BEGIN` - Start transaction
- Execute multiple SQL operations
- `COMMIT` - If all succeed, make changes permanent
- `ROLLBACK` - If any fails, undo all changes

Modern ORMs like TypeORM, Prisma, and Sequelize provide transaction APIs that handle this automatically.

---

### Q3: What happens if a transaction is interrupted (e.g., power failure)?
**Answer**: The database automatically rolls back the transaction upon restart. Database systems maintain a transaction log (write-ahead log) that records all changes. During recovery:
1. Uncommitted transactions are rolled back
2. Committed transactions are re-applied if needed
3. Database returns to a consistent state

---

### Q4: Can you have atomicity with microservices?
**Answer**: Traditional atomicity across multiple databases is challenging with microservices. Solutions include:

1. **Saga Pattern**: Break transaction into steps with compensating actions
2. **Two-Phase Commit (2PC)**: Distributed transaction protocol (heavy overhead)
3. **Event Sourcing**: Store events and rebuild state
4. **Eventual Consistency**: Accept temporary inconsistencies

Example: E-commerce order → Use saga with compensation if payment fails:
- Reserve inventory → Place order → Process payment
- If payment fails: Compensate by releasing inventory

---

### Q5: What's the difference between atomicity and isolation?
**Answer**:
- **Atomicity**: Ensures all operations in a transaction complete or none do (all-or-nothing)
- **Isolation**: Ensures concurrent transactions don't interfere with each other

Example:
- **Atomicity**: Transfer either completes fully or not at all
- **Isolation**: Two transfers happening simultaneously don't see each other's intermediate states

---

### Q6: How do you test atomicity in your application?
**Answer**: Testing strategies:

1. **Unit Tests**: Mock database and verify rollback is called on errors
2. **Integration Tests**: 
   - Force failures at different steps
   - Verify no partial changes persist
3. **Chaos Testing**: Kill database connections mid-transaction
4. **Race Condition Tests**: Run concurrent transactions

```typescript
describe('Transfer Atomicity', () => {
  it('should rollback on insufficient funds', async () => {
    const initialBalance = await getBalance('A123');
    
    try {
      await transferMoney('A123', 'B456', 1000000); // Too much
    } catch (error) {
      // Verify balance unchanged
      const finalBalance = await getBalance('A123');
      expect(finalBalance).toBe(initialBalance);
    }
  });
});
```

---

### Q7: What are savepoints and how do they relate to atomicity?
**Answer**: Savepoints allow partial rollbacks within a transaction:

```typescript
await client.query('BEGIN');
await client.query('UPDATE accounts SET balance = balance - 100 WHERE id = 1');

await client.query('SAVEPOINT sp1');
await client.query('UPDATE accounts SET balance = balance + 100 WHERE id = 2');

// Oops, wrong account - rollback to savepoint
await client.query('ROLLBACK TO SAVEPOINT sp1');

// Try correct account
await client.query('UPDATE accounts SET balance = balance + 100 WHERE id = 3');
await client.query('COMMIT');
```

This maintains atomicity for the overall transaction while allowing fine-grained control.

---

### Q8: How does atomicity work with distributed systems?
**Answer**: Distributed atomicity is complex:

**Traditional Approach (2PC)**:
1. Prepare phase: All nodes prepare to commit
2. Commit phase: Coordinator tells all to commit
3. Problem: Blocking, coordinator is single point of failure

**Modern Approach (Saga Pattern)**:
1. Break into local transactions
2. Each service commits independently
3. Use compensating transactions for rollback

```typescript
// Saga for order processing
async function orderSaga(order: Order) {
  const sagaLog: string[] = [];
  
  try {
    // Step 1
    await reserveInventory(order);
    sagaLog.push('INVENTORY_RESERVED');
    
    // Step 2
    await chargeCustomer(order);
    sagaLog.push('CUSTOMER_CHARGED');
    
    // Step 3
    await shipOrder(order);
    sagaLog.push('ORDER_SHIPPED');
    
  } catch (error) {
    // Compensate in reverse order
    if (sagaLog.includes('CUSTOMER_CHARGED')) {
      await refundCustomer(order);
    }
    if (sagaLog.includes('INVENTORY_RESERVED')) {
      await releaseInventory(order);
    }
    throw error;
  }
}
```

---

### Q9: What are common mistakes that break atomicity?
**Answer**:

1. **Not using transactions**:
```typescript
// ❌ BAD
await updateBalance(accountA, -100);
await updateBalance(accountB, +100); // If this fails, money is lost!
```

2. **Committing too early**:
```typescript
// ❌ BAD
await client.query('BEGIN');
await debitAccount(accountA, 100);
await client.query('COMMIT'); // Too early!
await creditAccount(accountB, 100); // Not in transaction!
```

3. **Mixing transactional and non-transactional operations**:
```typescript
// ❌ BAD
await client.query('BEGIN');
await updateDatabase();
await sendEmail(); // This succeeds even if transaction rolls back!
await client.query('COMMIT');
```

4. **Not handling errors properly**:
```typescript
// ❌ BAD - No rollback on error
await client.query('BEGIN');
await operation1();
await operation2(); // If this throws, no rollback happens
```

---

### Q10: How do NoSQL databases handle atomicity?
**Answer**:

**Document Databases (MongoDB)**:
- Single document operations are atomic
- Multi-document transactions supported (v4.0+)

**Key-Value Stores (Redis)**:
- Commands are atomic
- Use MULTI/EXEC for transaction-like behavior

**Column Stores (Cassandra)**:
- Lightweight transactions (LWT) using Paxos
- Limited compared to relational databases

```typescript
// MongoDB multi-document transaction
session.startTransaction();
await users.updateOne({ _id: userId }, { $inc: { balance: -100 } }, { session });
await accounts.insertOne({ type: 'debit', amount: 100 }, { session });
await session.commitTransaction();
```

**Trade-off**: NoSQL databases often prioritize availability and partition tolerance over strong consistency (CAP theorem).

---

## Scenario-Based Problems

### Scenario 1: Hotel Booking System
**Problem**: Multiple users are trying to book the last available room simultaneously. How do you ensure atomicity?

**Solution**:
```typescript
async function bookRoom(roomId: string, guestId: string): Promise<boolean> {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // Lock the room row
    const room = await client.query(
      'SELECT * FROM rooms WHERE room_id = $1 AND status = $2 FOR UPDATE',
      [roomId, 'AVAILABLE']
    );
    
    if (room.rowCount === 0) {
      throw new Error('Room not available');
    }
    
    // Book the room
    await client.query(
      'UPDATE rooms SET status = $1, booked_by = $2 WHERE room_id = $3',
      ['BOOKED', guestId, roomId]
    );
    
    await client.query(
      'INSERT INTO bookings (room_id, guest_id, booking_date) VALUES ($1, $2, NOW())',
      [roomId, guestId]
    );
    
    await client.query('COMMIT');
    return true;
    
  } catch (error) {
    await client.query('ROLLBACK');
    return false;
  } finally {
    client.release();
  }
}
```

---

### Scenario 2: Shopping Cart Checkout
**Problem**: Process checkout involving inventory deduction, order creation, and payment processing atomically.

**Solution**:
```typescript
async function checkout(cartId: string, customerId: string): Promise<Order> {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // 1. Get cart items
    const cartItems = await client.query(
      'SELECT * FROM cart_items WHERE cart_id = $1',
      [cartId]
    );
    
    let totalAmount = 0;
    
    // 2. Validate and reserve inventory
    for (const item of cartItems.rows) {
      const inventory = await client.query(
        'UPDATE inventory SET quantity = quantity - $1 WHERE product_id = $2 AND quantity >= $1 RETURNING *',
        [item.quantity, item.product_id]
      );
      
      if (inventory.rowCount === 0) {
        throw new Error(`Product ${item.product_id} out of stock`);
      }
      
      totalAmount += item.price * item.quantity;
    }
    
    // 3. Create order
    const orderId = generateOrderId();
    await client.query(
      'INSERT INTO orders (order_id, customer_id, total_amount, status) VALUES ($1, $2, $3, $4)',
      [orderId, customerId, totalAmount, 'PENDING']
    );
    
    // 4. Process payment
    const payment = await processPayment(customerId, totalAmount);
    if (!payment.success) {
      throw new Error('Payment failed');
    }
    
    // 5. Update order status
    await client.query(
      'UPDATE orders SET status = $1, payment_id = $2 WHERE order_id = $3',
      ['CONFIRMED', payment.paymentId, orderId]
    );
    
    // 6. Clear cart
    await client.query('DELETE FROM cart_items WHERE cart_id = $1', [cartId]);
    
    await client.query('COMMIT');
    
    return { orderId, totalAmount, status: 'CONFIRMED' };
    
  } catch (error) {
    await client.query('ROLLBACK');
    throw new Error(`Checkout failed: ${error.message}`);
  } finally {
    client.release();
  }
}
```

---

### Scenario 3: Ticket Booking with Seat Selection
**Problem**: Book multiple seats for a concert where each seat must be atomic.

**Solution**:
```typescript
async function bookTickets(
  eventId: string,
  seats: string[],
  customerId: string
): Promise<string> {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    const bookingId = generateBookingId();
    
    // Lock and book all seats atomically
    for (const seatNumber of seats) {
      const seat = await client.query(
        'SELECT * FROM seats WHERE event_id = $1 AND seat_number = $2 AND status = $3 FOR UPDATE',
        [eventId, seatNumber, 'AVAILABLE']
      );
      
      if (seat.rowCount === 0) {
        throw new Error(`Seat ${seatNumber} not available`);
      }
      
      await client.query(
        'UPDATE seats SET status = $1, booking_id = $2 WHERE event_id = $3 AND seat_number = $4',
        ['BOOKED', bookingId, eventId, seatNumber]
      );
      
      await client.query(
        'INSERT INTO tickets (booking_id, event_id, seat_number, customer_id) VALUES ($1, $2, $3, $4)',
        [bookingId, eventId, seatNumber, customerId]
      );
    }
    
    await client.query('COMMIT');
    return bookingId;
    
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

1. ✅ **Always use transactions** for operations that must complete together
2. ✅ **Handle errors properly** with try-catch and rollback
3. ✅ **Test failure scenarios** to ensure rollback works correctly
4. ✅ **Use row-level locking** (FOR UPDATE) for concurrent access
5. ✅ **Keep transactions short** to avoid blocking other operations
6. ✅ **Log transaction outcomes** for debugging and auditing
7. ✅ **Consider distributed patterns** (Saga) for microservices
8. ✅ **Understand your database's** transaction guarantees

**Remember**: Atomicity is your safety net against partial failures. When in doubt, wrap it in a transaction!
