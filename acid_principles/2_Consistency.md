# Consistency Principle

## Definition
**Consistency** ensures that a transaction brings the database from one valid state to another valid state, maintaining all defined rules, constraints, triggers, and cascades. The database must satisfy all integrity constraints before and after the transaction.

**Key Concept**: "Valid State to Valid State" - No transaction should leave the database in an inconsistent or invalid state.

---

## Real-World Analogy
Think of consistency like rules in chess. You can't make an illegal move (e.g., moving a pawn backwards). The game board must always be in a valid state according to chess rules. Similarly, your database must always satisfy its rules (constraints, foreign keys, check constraints, etc.).

---

## Understanding Database Constraints

### Types of Constraints:
1. **Primary Key**: Unique, non-null identifier
2. **Foreign Key**: References another table
3. **Unique**: No duplicate values
4. **Not Null**: Must have a value
5. **Check**: Custom validation rules
6. **Domain**: Data type constraints

---

## Bank Account Example

### ❌ Without Consistency (Bad Practice)

```typescript
// No constraint checking - DANGEROUS!
async function withdrawMoney(accountId: string, amount: number) {
  // Directly update without validation
  await db.query(
    'UPDATE accounts SET balance = balance - $1 WHERE account_id = $2',
    [amount, accountId]
  );
  
  // ⚠️ PROBLEMS:
  // - Balance can go negative
  // - Amount can be negative (depositing via withdrawal)
  // - No status check (can withdraw from closed accounts)
}
```

---

### ✅ With Consistency (Good Practice)

```typescript
// Database schema with constraints
const createAccountsTable = `
  CREATE TABLE accounts (
    account_id VARCHAR(50) PRIMARY KEY,
    customer_id VARCHAR(50) NOT NULL,
    balance DECIMAL(15, 2) NOT NULL CHECK (balance >= 0),
    account_type VARCHAR(20) NOT NULL CHECK (account_type IN ('SAVINGS', 'CHECKING', 'BUSINESS')),
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE', 'FROZEN', 'CLOSED')),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE RESTRICT
  );
  
  CREATE TABLE transactions (
    transaction_id VARCHAR(50) PRIMARY KEY,
    account_id VARCHAR(50) NOT NULL,
    transaction_type VARCHAR(20) NOT NULL CHECK (transaction_type IN ('DEPOSIT', 'WITHDRAWAL', 'TRANSFER')),
    amount DECIMAL(15, 2) NOT NULL CHECK (amount > 0),
    balance_after DECIMAL(15, 2) NOT NULL CHECK (balance_after >= 0),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(account_id) ON DELETE CASCADE
  );
`;

// Application code respecting constraints
async function withdrawMoney(accountId: string, amount: number): Promise<void> {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // Validate amount
    if (amount <= 0) {
      throw new Error('Amount must be positive');
    }
    
    // Get current account state
    const accountResult = await client.query(
      'SELECT balance, status FROM accounts WHERE account_id = $1 FOR UPDATE',
      [accountId]
    );
    
    if (accountResult.rowCount === 0) {
      throw new Error('Account not found');
    }
    
    const account = accountResult.rows[0];
    
    // Check account status
    if (account.status !== 'ACTIVE') {
      throw new Error(`Cannot withdraw from ${account.status} account`);
    }
    
    // Check sufficient balance
    if (account.balance < amount) {
      throw new Error('Insufficient funds');
    }
    
    // Perform withdrawal
    const newBalance = account.balance - amount;
    
    await client.query(
      'UPDATE accounts SET balance = $1, updated_at = CURRENT_TIMESTAMP WHERE account_id = $2',
      [newBalance, accountId]
    );
    
    // Record transaction
    await client.query(
      'INSERT INTO transactions (transaction_id, account_id, transaction_type, amount, balance_after) VALUES ($1, $2, $3, $4, $5)',
      [generateTxnId(), accountId, 'WITHDRAWAL', amount, newBalance]
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

## E-Commerce Inventory Example

```typescript
// Schema with consistency rules
const createInventorySchema = `
  CREATE TABLE products (
    product_id VARCHAR(50) PRIMARY KEY,
    product_name VARCHAR(200) NOT NULL,
    price DECIMAL(10, 2) NOT NULL CHECK (price >= 0),
    category VARCHAR(50) NOT NULL,
    is_active BOOLEAN DEFAULT true
  );
  
  CREATE TABLE inventory (
    product_id VARCHAR(50) PRIMARY KEY,
    quantity INT NOT NULL CHECK (quantity >= 0),
    reserved_quantity INT NOT NULL DEFAULT 0 CHECK (reserved_quantity >= 0),
    warehouse_location VARCHAR(50) NOT NULL,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE,
    CHECK (reserved_quantity <= quantity)
  );
  
  CREATE TABLE orders (
    order_id VARCHAR(50) PRIMARY KEY,
    customer_id VARCHAR(50) NOT NULL,
    order_status VARCHAR(20) NOT NULL CHECK (order_status IN ('PENDING', 'CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED')),
    total_amount DECIMAL(15, 2) NOT NULL CHECK (total_amount >= 0),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
  );
  
  CREATE TABLE order_items (
    order_item_id VARCHAR(50) PRIMARY KEY,
    order_id VARCHAR(50) NOT NULL,
    product_id VARCHAR(50) NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10, 2) NOT NULL CHECK (unit_price >= 0),
    subtotal DECIMAL(15, 2) NOT NULL CHECK (subtotal >= 0),
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE RESTRICT
  );
`;

// Consistent order placement
async function placeOrder(customerId: string, items: OrderItem[]): Promise<string> {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    const orderId = generateOrderId();
    let totalAmount = 0;
    
    // Validate all items first
    for (const item of items) {
      // Check product exists and is active
      const product = await client.query(
        'SELECT price, is_active FROM products WHERE product_id = $1',
        [item.productId]
      );
      
      if (product.rowCount === 0) {
        throw new Error(`Product ${item.productId} not found`);
      }
      
      if (!product.rows[0].is_active) {
        throw new Error(`Product ${item.productId} is not available`);
      }
      
      // Check inventory
      const inventory = await client.query(
        'SELECT quantity, reserved_quantity FROM inventory WHERE product_id = $1 FOR UPDATE',
        [item.productId]
      );
      
      if (inventory.rowCount === 0) {
        throw new Error(`No inventory for product ${item.productId}`);
      }
      
      const availableQty = inventory.rows[0].quantity - inventory.rows[0].reserved_quantity;
      
      if (availableQty < item.quantity) {
        throw new Error(`Insufficient inventory for product ${item.productId}`);
      }
      
      const price = product.rows[0].price;
      const subtotal = price * item.quantity;
      totalAmount += subtotal;
      
      // Reserve inventory
      await client.query(
        'UPDATE inventory SET reserved_quantity = reserved_quantity + $1, last_updated = CURRENT_TIMESTAMP WHERE product_id = $2',
        [item.quantity, item.productId]
      );
      
      // Create order item
      await client.query(
        'INSERT INTO order_items (order_item_id, order_id, product_id, quantity, unit_price, subtotal) VALUES ($1, $2, $3, $4, $5, $6)',
        [generateOrderItemId(), orderId, item.productId, item.quantity, price, subtotal]
      );
    }
    
    // Create order
    await client.query(
      'INSERT INTO orders (order_id, customer_id, order_status, total_amount) VALUES ($1, $2, $3, $4)',
      [orderId, customerId, 'PENDING', totalAmount]
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

## Referential Integrity Example

```typescript
// Maintaining referential integrity across tables
const createRelationalSchema = `
  -- Parent table
  CREATE TABLE departments (
    department_id VARCHAR(50) PRIMARY KEY,
    department_name VARCHAR(100) NOT NULL UNIQUE,
    budget DECIMAL(15, 2) CHECK (budget >= 0)
  );
  
  -- Child table with foreign key
  CREATE TABLE employees (
    employee_id VARCHAR(50) PRIMARY KEY,
    employee_name VARCHAR(100) NOT NULL,
    department_id VARCHAR(50) NOT NULL,
    salary DECIMAL(10, 2) NOT NULL CHECK (salary >= 0),
    hire_date DATE NOT NULL,
    FOREIGN KEY (department_id) REFERENCES departments(department_id) ON DELETE RESTRICT ON UPDATE CASCADE
  );
  
  -- Grandchild table
  CREATE TABLE projects (
    project_id VARCHAR(50) PRIMARY KEY,
    project_name VARCHAR(200) NOT NULL,
    employee_id VARCHAR(50) NOT NULL,
    budget DECIMAL(15, 2) CHECK (budget >= 0),
    start_date DATE NOT NULL,
    end_date DATE,
    FOREIGN KEY (employee_id) REFERENCES employees(employee_id) ON DELETE CASCADE,
    CHECK (end_date IS NULL OR end_date >= start_date)
  );
`;

// Consistent employee assignment
async function assignEmployeeToProject(
  employeeId: string,
  projectId: string
): Promise<void> {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // Verify employee exists
    const employee = await client.query(
      'SELECT employee_id FROM employees WHERE employee_id = $1',
      [employeeId]
    );
    
    if (employee.rowCount === 0) {
      throw new Error('Employee not found - referential integrity violation');
    }
    
    // Verify project exists
    const project = await client.query(
      'SELECT project_id FROM projects WHERE project_id = $1',
      [projectId]
    );
    
    if (project.rowCount === 0) {
      throw new Error('Project not found');
    }
    
    // Update project assignment
    await client.query(
      'UPDATE projects SET employee_id = $1 WHERE project_id = $2',
      [employeeId, projectId]
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

## Triggers for Consistency

```typescript
// Database triggers to maintain consistency
const createConsistencyTriggers = `
  -- Trigger to update total amount when order items change
  CREATE OR REPLACE FUNCTION update_order_total()
  RETURNS TRIGGER AS $$
  BEGIN
    UPDATE orders
    SET total_amount = (
      SELECT COALESCE(SUM(subtotal), 0)
      FROM order_items
      WHERE order_id = NEW.order_id
    )
    WHERE order_id = NEW.order_id;
    
    RETURN NEW;
  END;
  $$ LANGUAGE plpgsql;
  
  CREATE TRIGGER order_items_changed
  AFTER INSERT OR UPDATE OR DELETE ON order_items
  FOR EACH ROW
  EXECUTE FUNCTION update_order_total();
  
  -- Trigger to prevent overspending department budget
  CREATE OR REPLACE FUNCTION check_department_budget()
  RETURNS TRIGGER AS $$
  DECLARE
    total_salaries DECIMAL(15, 2);
    dept_budget DECIMAL(15, 2);
  BEGIN
    SELECT COALESCE(SUM(salary), 0) INTO total_salaries
    FROM employees
    WHERE department_id = NEW.department_id;
    
    SELECT budget INTO dept_budget
    FROM departments
    WHERE department_id = NEW.department_id;
    
    IF total_salaries + NEW.salary > dept_budget THEN
      RAISE EXCEPTION 'Total salaries exceed department budget';
    END IF;
    
    RETURN NEW;
  END;
  $$ LANGUAGE plpgsql;
  
  CREATE TRIGGER check_budget_before_hire
  BEFORE INSERT OR UPDATE ON employees
  FOR EACH ROW
  EXECUTE FUNCTION check_department_budget();
`;
```

---

## Application-Level Validation

```typescript
// Combining database constraints with application logic
interface Account {
  accountId: string;
  customerId: string;
  balance: number;
  accountType: 'SAVINGS' | 'CHECKING' | 'BUSINESS';
  status: 'ACTIVE' | 'FROZEN' | 'CLOSED';
  dailyWithdrawalLimit: number;
  withdrawalsToday: number;
}

class AccountService {
  async withdraw(accountId: string, amount: number): Promise<void> {
    // Application-level validation
    this.validateWithdrawalAmount(amount);
    
    const client = await pool.connect();
    
    try {
      await client.query('BEGIN');
      
      const account = await this.getAccount(client, accountId);
      
      // Business rule validations
      this.validateAccountStatus(account);
      this.validateSufficientBalance(account, amount);
      this.validateDailyLimit(account, amount);
      
      // Perform withdrawal (database constraints will also validate)
      await this.performWithdrawal(client, accountId, amount);
      
      // Update daily withdrawal counter
      await this.incrementDailyWithdrawals(client, accountId, amount);
      
      await client.query('COMMIT');
      
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
  
  private validateWithdrawalAmount(amount: number): void {
    if (amount <= 0) {
      throw new Error('Withdrawal amount must be positive');
    }
    
    if (amount > 50000) {
      throw new Error('Withdrawal amount exceeds maximum limit');
    }
    
    if (amount % 1 !== 0) {
      throw new Error('Amount must be a whole number');
    }
  }
  
  private validateAccountStatus(account: Account): void {
    if (account.status !== 'ACTIVE') {
      throw new Error(`Account is ${account.status}, cannot withdraw`);
    }
  }
  
  private validateSufficientBalance(account: Account, amount: number): void {
    const minimumBalance = account.accountType === 'SAVINGS' ? 100 : 0;
    
    if (account.balance - amount < minimumBalance) {
      throw new Error('Insufficient funds (minimum balance required)');
    }
  }
  
  private validateDailyLimit(account: Account, amount: number): void {
    if (account.withdrawalsToday + amount > account.dailyWithdrawalLimit) {
      throw new Error('Daily withdrawal limit exceeded');
    }
  }
  
  private async getAccount(client: any, accountId: string): Promise<Account> {
    const result = await client.query(
      'SELECT * FROM accounts WHERE account_id = $1 FOR UPDATE',
      [accountId]
    );
    
    if (result.rowCount === 0) {
      throw new Error('Account not found');
    }
    
    return result.rows[0];
  }
  
  private async performWithdrawal(client: any, accountId: string, amount: number): Promise<void> {
    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE account_id = $2',
      [amount, accountId]
    );
  }
  
  private async incrementDailyWithdrawals(client: any, accountId: string, amount: number): Promise<void> {
    await client.query(
      'UPDATE accounts SET withdrawals_today = withdrawals_today + $1 WHERE account_id = $2',
      [amount, accountId]
    );
  }
}
```

---

## Interview Questions & Answers

### Q1: What is consistency in ACID properties?
**Answer**: Consistency ensures that a transaction brings the database from one valid state to another valid state. All defined rules (constraints, triggers, cascades) must be satisfied before and after the transaction. The database should never be left in an inconsistent state where constraints are violated.

**Example**: If you have a foreign key constraint that orders must reference a valid customer, consistency ensures no order can be created with an invalid customer_id.

---

### Q2: How is consistency different from atomicity?
**Answer**:
- **Atomicity**: All operations in a transaction succeed or fail together (all-or-nothing)
- **Consistency**: The database remains in a valid state (rules are never violated)

**Example**: 
- Atomicity ensures a bank transfer either completes both debit and credit, or neither
- Consistency ensures the total money in the system remains constant and no account goes below minimum balance

They work together: Atomicity helps achieve consistency by preventing partial updates that could violate rules.

---

### Q3: What types of constraints maintain consistency?
**Answer**:

1. **Primary Key**: Uniqueness and non-null
```sql
account_id VARCHAR(50) PRIMARY KEY
```

2. **Foreign Key**: Referential integrity
```sql
FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
```

3. **Check Constraint**: Custom validation
```sql
CHECK (balance >= 0)
CHECK (age >= 18)
```

4. **Not Null**: Required fields
```sql
email VARCHAR(100) NOT NULL
```

5. **Unique**: No duplicates
```sql
email VARCHAR(100) UNIQUE
```

6. **Domain Constraint**: Data type limits
```sql
quantity INT CHECK (quantity BETWEEN 1 AND 1000)
```

---

### Q4: Should validation happen in application or database?
**Answer**: **Both layers** for defense in depth:

**Database Layer** (Must have):
- Fundamental data integrity (NOT NULL, PRIMARY KEY)
- Referential integrity (FOREIGN KEY)
- Basic constraints (CHECK)
- Guarantees consistency regardless of application

**Application Layer** (Should have):
- Complex business rules
- Better error messages
- Performance optimization (fail fast)
- Cross-field validation

**Example**:
```typescript
// Application validation
if (age < 18) {
  throw new Error('Must be 18 or older to open account');
}

// Database constraint (backup)
CREATE TABLE customers (
  age INT CHECK (age >= 18)
);
```

---

### Q5: What happens if a constraint is violated during a transaction?
**Answer**: The database immediately rolls back the transaction and returns an error. No partial changes are saved.

**Example**:
```typescript
try {
  await client.query('BEGIN');
  
  // This will violate balance >= 0 constraint
  await client.query(
    'UPDATE accounts SET balance = -100 WHERE account_id = $1',
    ['ACC123']
  );
  
  await client.query('COMMIT'); // Never reached
  
} catch (error) {
  // Error: new row violates check constraint "accounts_balance_check"
  await client.query('ROLLBACK');
}
```

The database automatically rolls back to maintain consistency.

---

### Q6: How do you maintain consistency across microservices?
**Answer**: Distributed consistency is challenging. Strategies include:

1. **Saga Pattern**: Coordinate using events/orchestration
2. **Event Sourcing**: Single source of truth
3. **API Contracts**: Validate at boundaries
4. **Eventual Consistency**: Accept temporary inconsistencies
5. **Distributed Validation**: Each service validates its domain

**Example - Saga with consistency checks**:
```typescript
async function orderSaga(order: Order) {
  // Step 1: Validate and reserve inventory
  const inventoryResult = await inventoryService.reserve(order.items);
  if (!inventoryResult.success) {
    throw new Error('Inventory validation failed');
  }
  
  // Step 2: Validate and charge payment
  const paymentResult = await paymentService.charge(order.total);
  if (!paymentResult.success) {
    await inventoryService.release(order.items); // Compensate
    throw new Error('Payment validation failed');
  }
  
  // Step 3: Create order
  await orderService.create(order);
}
```

---

### Q7: What are database triggers and how do they help consistency?
**Answer**: Triggers are automatic actions executed when certain events occur (INSERT, UPDATE, DELETE). They help maintain consistency by:

1. **Automatic calculations**
2. **Cross-table validation**
3. **Audit trails**
4. **Cascading updates**

**Example**:
```sql
-- Automatically update order total when items change
CREATE TRIGGER update_order_total
AFTER INSERT OR UPDATE ON order_items
FOR EACH ROW
EXECUTE FUNCTION recalculate_order_total();

-- Prevent deletion if balance > 0
CREATE TRIGGER prevent_account_deletion
BEFORE DELETE ON accounts
FOR EACH ROW
EXECUTE FUNCTION check_zero_balance();
```

**Caution**: Overuse can make debugging difficult.

---

### Q8: How do you test consistency in your application?
**Answer**: Testing strategies:

1. **Constraint violation tests**:
```typescript
it('should prevent negative balance', async () => {
  await expect(
    updateBalance('ACC123', -1000)
  ).rejects.toThrow('balance constraint');
});
```

2. **Referential integrity tests**:
```typescript
it('should prevent order with invalid customer', async () => {
  await expect(
    createOrder({ customerId: 'INVALID', items: [] })
  ).rejects.toThrow('foreign key constraint');
});
```

3. **Business rule tests**:
```typescript
it('should enforce daily withdrawal limit', async () => {
  await withdraw('ACC123', 5000);
  await withdraw('ACC123', 5000);
  
  await expect(
    withdraw('ACC123', 5000) // Exceeds 10K daily limit
  ).rejects.toThrow('Daily limit exceeded');
});
```

4. **Concurrent transaction tests**: Verify consistency under load

---

### Q9: What is eventual consistency and when to use it?
**Answer**: **Eventual consistency** means data will become consistent over time, but may be temporarily inconsistent.

**When to use**:
- Distributed systems (microservices)
- High availability requirements
- Read-heavy workloads
- Non-critical data

**When NOT to use**:
- Financial transactions
- Inventory management
- User authentication
- Any operation requiring immediate accuracy

**Example**:
```typescript
// Eventual consistency: Social media likes
async function likePost(postId: string) {
  // Update counter (eventually consistent)
  await redis.incr(`post:${postId}:likes`);
  
  // Publish event for eventual synchronization
  await eventBus.publish({
    type: 'POST_LIKED',
    postId,
    timestamp: Date.now()
  });
  
  // Database will be updated asynchronously
}

// Strong consistency: Bank transfer
async function transfer(from: string, to: string, amount: number) {
  // Must be immediately consistent
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await debit(from, amount);
    await credit(to, amount);
    await client.query('COMMIT');
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  }
}
```

---

### Q10: How do ORMs help maintain consistency?
**Answer**: ORMs (TypeORM, Prisma, Sequelize) provide:

1. **Schema definition** with constraints
2. **Validation** before database operations
3. **Type safety** preventing invalid data
4. **Relationship management**
5. **Transaction support**

**Example with Prisma**:
```typescript
// Schema definition
model Account {
  id        String   @id @default(uuid())
  customerId String
  balance   Decimal  @default(0)
  status    Status   @default(ACTIVE)
  
  customer  Customer @relation(fields: [customerId], references: [id])
  
  @@check([balance >= 0], name: "positive_balance")
}

enum Status {
  ACTIVE
  FROZEN
  CLOSED
}

// Application code
await prisma.account.update({
  where: { id: accountId },
  data: {
    balance: { decrement: 100 } // Prisma validates constraints
  }
});
```

**Benefits**: Consistency enforced at multiple layers (TypeScript types + database constraints)

---

## Scenario-Based Problems

### Scenario 1: E-Commerce Stock Management
**Problem**: Prevent overselling when multiple users order the same product simultaneously.

**Solution**:
```typescript
async function createOrder(customerId: string, productId: string, quantity: number): Promise<string> {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // Lock product row to prevent concurrent modifications
    const product = await client.query(
      `SELECT stock_quantity, price, is_active 
       FROM products 
       WHERE product_id = $1 
       FOR UPDATE`,
      [productId]
    );
    
    if (product.rowCount === 0) {
      throw new Error('Product not found');
    }
    
    const { stock_quantity, price, is_active } = product.rows[0];
    
    // Consistency checks
    if (!is_active) {
      throw new Error('Product is no longer available');
    }
    
    if (stock_quantity < quantity) {
      throw new Error(`Only ${stock_quantity} items available`);
    }
    
    if (quantity <= 0) {
      throw new Error('Quantity must be positive');
    }
    
    // Deduct stock
    await client.query(
      'UPDATE products SET stock_quantity = stock_quantity - $1 WHERE product_id = $2',
      [quantity, productId]
    );
    
    // Create order
    const orderId = generateId();
    await client.query(
      'INSERT INTO orders (order_id, customer_id, total_amount, status) VALUES ($1, $2, $3, $4)',
      [orderId, customerId, price * quantity, 'CONFIRMED']
    );
    
    await client.query(
      'INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES ($1, $2, $3, $4)',
      [orderId, productId, quantity, price]
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

### Scenario 2: Prevent Double Booking
**Problem**: Two users trying to book the same hotel room at the same time.

**Solution**:
```typescript
const createBookingSchema = `
  CREATE TABLE rooms (
    room_id VARCHAR(50) PRIMARY KEY,
    room_number VARCHAR(10) NOT NULL UNIQUE,
    room_type VARCHAR(50) NOT NULL,
    price_per_night DECIMAL(10, 2) NOT NULL CHECK (price_per_night > 0)
  );
  
  CREATE TABLE bookings (
    booking_id VARCHAR(50) PRIMARY KEY,
    room_id VARCHAR(50) NOT NULL,
    guest_id VARCHAR(50) NOT NULL,
    check_in DATE NOT NULL,
    check_out DATE NOT NULL,
    status VARCHAR(20) DEFAULT 'CONFIRMED' CHECK (status IN ('CONFIRMED', 'CANCELLED')),
    FOREIGN KEY (room_id) REFERENCES rooms(room_id),
    CHECK (check_out > check_in),
    CONSTRAINT no_overlap EXCLUDE USING gist (
      room_id WITH =,
      daterange(check_in, check_out) WITH &&
    ) WHERE (status = 'CONFIRMED')
  );
`;

async function bookRoom(
  roomId: string,
  guestId: string,
  checkIn: Date,
  checkOut: Date
): Promise<string> {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // Check for overlapping bookings
    const overlap = await client.query(
      `SELECT booking_id 
       FROM bookings 
       WHERE room_id = $1 
       AND status = 'CONFIRMED'
       AND daterange($2::date, $3::date) && daterange(check_in, check_out)`,
      [roomId, checkIn, checkOut]
    );
    
    if (overlap.rowCount > 0) {
      throw new Error('Room not available for selected dates');
    }
    
    // Create booking
    const bookingId = generateId();
    await client.query(
      'INSERT INTO bookings (booking_id, room_id, guest_id, check_in, check_out) VALUES ($1, $2, $3, $4, $5)',
      [bookingId, roomId, guestId, checkIn, checkOut]
    );
    
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

### Scenario 3: Hierarchical Data Consistency
**Problem**: Maintain organizational hierarchy (employees must belong to existing departments, managers must be employees).

**Solution**:
```typescript
const createHierarchySchema = `
  CREATE TABLE departments (
    department_id VARCHAR(50) PRIMARY KEY,
    department_name VARCHAR(100) NOT NULL UNIQUE,
    parent_department_id VARCHAR(50),
    FOREIGN KEY (parent_department_id) REFERENCES departments(department_id)
  );
  
  CREATE TABLE employees (
    employee_id VARCHAR(50) PRIMARY KEY,
    employee_name VARCHAR(100) NOT NULL,
    department_id VARCHAR(50) NOT NULL,
    manager_id VARCHAR(50),
    salary DECIMAL(10, 2) CHECK (salary > 0),
    FOREIGN KEY (department_id) REFERENCES departments(department_id),
    FOREIGN KEY (manager_id) REFERENCES employees(employee_id),
    CHECK (manager_id != employee_id) -- Can't be own manager
  );
  
  -- Trigger to prevent circular references
  CREATE OR REPLACE FUNCTION check_manager_hierarchy()
  RETURNS TRIGGER AS $$
  DECLARE
    current_manager VARCHAR(50);
    depth INT := 0;
  BEGIN
    current_manager := NEW.manager_id;
    
    WHILE current_manager IS NOT NULL AND depth < 10 LOOP
      IF current_manager = NEW.employee_id THEN
        RAISE EXCEPTION 'Circular manager hierarchy detected';
      END IF;
      
      SELECT manager_id INTO current_manager
      FROM employees
      WHERE employee_id = current_manager;
      
      depth := depth + 1;
    END LOOP;
    
    RETURN NEW;
  END;
  $$ LANGUAGE plpgsql;
  
  CREATE TRIGGER prevent_circular_hierarchy
  BEFORE INSERT OR UPDATE ON employees
  FOR EACH ROW
  EXECUTE FUNCTION check_manager_hierarchy();
`;
```

---

## Key Takeaways

1. ✅ **Define constraints** at database level (first line of defense)
2. ✅ **Validate in application** for better UX and error messages
3. ✅ **Use foreign keys** to maintain referential integrity
4. ✅ **Add check constraints** for business rules
5. ✅ **Leverage triggers** for complex consistency rules (cautiously)
6. ✅ **Test constraint violations** to ensure they work correctly
7. ✅ **Design consistent schemas** from the start
8. ✅ **Document business rules** that enforce consistency

**Remember**: Consistency is about maintaining valid states. Your database should make it impossible to create invalid data!
