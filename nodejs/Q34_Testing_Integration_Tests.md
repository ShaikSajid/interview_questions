# Q34: Testing - Integration Tests with Supertest

## 📋 Summary
Integration testing validates that multiple components work together correctly. This guide covers **Supertest for API testing**, database integration tests, test containers, transaction rollback strategies, and comprehensive banking examples testing complete payment flows end-to-end.

---

## 🎯 What You'll Learn
- **Supertest**: HTTP assertion library for Express/API testing
- **Database Testing**: Test database setup, transactions, rollback
- **Test Containers**: Isolated database instances for testing
- **End-to-End Flows**: Complete payment workflows
- **Test Data Management**: Fixtures, factories, cleanup
- **Banking Examples**: Transfer endpoints, account management, payment processing

---

## 📖 Comprehensive Explanation

### What is Integration Testing?

**Integration testing** tests how multiple components work together:
- API endpoints + database
- Service layer + database
- Multiple services together
- External API integrations

**Differences from Unit Tests**:

| Aspect | Unit Tests | Integration Tests |
|--------|-----------|------------------|
| **Scope** | Single function/class | Multiple components |
| **Dependencies** | Mocked | Real (database, APIs) |
| **Speed** | Very fast (~1-10ms) | Slower (~50-500ms) |
| **Isolation** | Complete | Partial |
| **Purpose** | Verify logic | Verify integration |

---

## 🧪 Supertest for API Testing

### Why Supertest?

**Supertest** is the standard for testing Express/Node.js APIs:
- Make HTTP requests in tests
- Assert response status, headers, body
- Works with Jest, Mocha, any test framework
- No need to start server manually

### Basic Supertest Usage

```javascript
const request = require('supertest');
const app = require('../app');

describe('GET /api/health', () => {
  test('should return 200 OK', async () => {
    const response = await request(app)
      .get('/api/health')
      .expect(200)
      .expect('Content-Type', /json/);

    expect(response.body).toEqual({
      status: 'ok',
      timestamp: expect.any(String)
    });
  });
});
```

### Common Supertest Assertions

```javascript
await request(app)
  .get('/api/accounts/123')
  .expect(200)                           // Status code
  .expect('Content-Type', /json/)        // Header
  .expect({ balance: 1000 });            // Exact body match

// POST with body
await request(app)
  .post('/api/transfer')
  .send({ from: 'ACC-1', to: 'ACC-2', amount: 500 })
  .set('Authorization', 'Bearer token')
  .expect(201);

// Custom assertions
const response = await request(app).get('/api/accounts');
expect(response.body.accounts).toHaveLength(5);
expect(response.body.accounts[0]).toHaveProperty('balance');
```

---

## 🗄️ Database Testing Strategies

### Strategy 1: In-Memory Database (SQLite)

**Pros**: Fast, no setup  
**Cons**: Different from production DB

```javascript
// jest.config.js
module.exports = {
  testEnvironment: 'node',
  setupFilesAfterEnv: ['./tests/setup.js']
};

// tests/setup.js
const { Sequelize } = require('sequelize');

let sequelize;

beforeAll(async () => {
  sequelize = new Sequelize('sqlite::memory:');
  await sequelize.sync({ force: true });
});

afterAll(async () => {
  await sequelize.close();
});
```

### Strategy 2: Test Database with Transactions

**Pros**: Real database, isolated tests  
**Cons**: Slower, needs cleanup

```javascript
let db;

beforeAll(async () => {
  db = await createConnection({
    host: 'localhost',
    database: 'banking_test',
    // ...
  });
});

beforeEach(async () => {
  await db.query('BEGIN'); // Start transaction
});

afterEach(async () => {
  await db.query('ROLLBACK'); // Rollback changes
});

afterAll(async () => {
  await db.end();
});
```

### Strategy 3: Test Containers (Docker)

**Pros**: Isolated, matches production  
**Cons**: Requires Docker, slower startup

```javascript
const { GenericContainer } = require('testcontainers');

let postgresContainer;
let db;

beforeAll(async () => {
  // Start PostgreSQL container
  postgresContainer = await new GenericContainer('postgres:14')
    .withEnvironment({ POSTGRES_PASSWORD: 'test' })
    .withExposedPorts(5432)
    .start();

  const port = postgresContainer.getMappedPort(5432);
  db = await createConnection({
    host: 'localhost',
    port,
    database: 'postgres',
    user: 'postgres',
    password: 'test'
  });
}, 60000); // 60s timeout for container startup

afterAll(async () => {
  await db.end();
  await postgresContainer.stop();
});
```

---

## 📝 Example 1: Banking API Integration Tests

### Express Banking API

```javascript
// src/app.js
const express = require('express');
const { Pool } = require('pg');

const app = express();
app.use(express.json());

const pool = new Pool({
  host: process.env.DB_HOST || 'localhost',
  port: process.env.DB_PORT || 5432,
  database: process.env.DB_NAME || 'banking_db',
  user: process.env.DB_USER || 'postgres',
  password: process.env.DB_PASSWORD || 'password'
});

// Health check
app.get('/api/health', (req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// Get account balance
app.get('/api/accounts/:accountId/balance', async (req, res) => {
  try {
    const { accountId } = req.params;

    const result = await pool.query(
      'SELECT account_id, balance, currency FROM accounts WHERE account_id = $1',
      [accountId]
    );

    if (result.rows.length === 0) {
      return res.status(404).json({ error: 'Account not found' });
    }

    res.json({
      accountId: result.rows[0].account_id,
      balance: parseFloat(result.rows[0].balance),
      currency: result.rows[0].currency
    });
  } catch (error) {
    console.error('Error fetching balance:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Transfer money
app.post('/api/transfer', async (req, res) => {
  const client = await pool.connect();

  try {
    const { fromAccount, toAccount, amount, description } = req.body;

    // Validation
    if (!fromAccount || !toAccount || !amount) {
      return res.status(400).json({ error: 'Missing required fields' });
    }

    if (amount <= 0) {
      return res.status(400).json({ error: 'Amount must be positive' });
    }

    if (fromAccount === toAccount) {
      return res.status(400).json({ error: 'Cannot transfer to same account' });
    }

    await client.query('BEGIN');

    // Check sender balance
    const senderResult = await client.query(
      'SELECT balance FROM accounts WHERE account_id = $1 FOR UPDATE',
      [fromAccount]
    );

    if (senderResult.rows.length === 0) {
      await client.query('ROLLBACK');
      return res.status(404).json({ error: 'Sender account not found' });
    }

    const senderBalance = parseFloat(senderResult.rows[0].balance);

    if (senderBalance < amount) {
      await client.query('ROLLBACK');
      return res.status(400).json({ error: 'Insufficient funds' });
    }

    // Check receiver exists
    const receiverResult = await client.query(
      'SELECT account_id FROM accounts WHERE account_id = $1 FOR UPDATE',
      [toAccount]
    );

    if (receiverResult.rows.length === 0) {
      await client.query('ROLLBACK');
      return res.status(404).json({ error: 'Receiver account not found' });
    }

    // Debit sender
    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE account_id = $2',
      [amount, fromAccount]
    );

    // Credit receiver
    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE account_id = $2',
      [amount, toAccount]
    );

    // Record transaction
    const transactionId = `TXN-${Date.now()}`;
    await client.query(
      `INSERT INTO transactions (transaction_id, from_account, to_account, amount, description, status)
       VALUES ($1, $2, $3, $4, $5, 'completed')`,
      [transactionId, fromAccount, toAccount, amount, description || '']
    );

    await client.query('COMMIT');

    res.status(201).json({
      transactionId,
      fromAccount,
      toAccount,
      amount,
      status: 'completed'
    });

  } catch (error) {
    await client.query('ROLLBACK');
    console.error('Transfer error:', error);
    res.status(500).json({ error: 'Transfer failed' });
  } finally {
    client.release();
  }
});

// Get transaction history
app.get('/api/accounts/:accountId/transactions', async (req, res) => {
  try {
    const { accountId } = req.params;
    const limit = parseInt(req.query.limit) || 10;

    const result = await pool.query(
      `SELECT transaction_id, from_account, to_account, amount, description, status, created_at
       FROM transactions
       WHERE from_account = $1 OR to_account = $1
       ORDER BY created_at DESC
       LIMIT $2`,
      [accountId, limit]
    );

    res.json({
      accountId,
      transactions: result.rows.map(row => ({
        transactionId: row.transaction_id,
        fromAccount: row.from_account,
        toAccount: row.to_account,
        amount: parseFloat(row.amount),
        description: row.description,
        status: row.status,
        createdAt: row.created_at
      }))
    });
  } catch (error) {
    console.error('Error fetching transactions:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

module.exports = { app, pool };
```

### Complete Integration Test Suite

```javascript
// __tests__/integration/api.test.js

const request = require('supertest');
const { app, pool } = require('../../src/app');

// Test database setup
beforeAll(async () => {
  // Create tables
  await pool.query(`
    CREATE TABLE IF NOT EXISTS accounts (
      account_id VARCHAR(50) PRIMARY KEY,
      customer_name VARCHAR(100),
      balance NUMERIC(15, 2) DEFAULT 0,
      currency VARCHAR(3) DEFAULT 'USD'
    )
  `);

  await pool.query(`
    CREATE TABLE IF NOT EXISTS transactions (
      transaction_id VARCHAR(100) PRIMARY KEY,
      from_account VARCHAR(50),
      to_account VARCHAR(50),
      amount NUMERIC(15, 2),
      description TEXT,
      status VARCHAR(20),
      created_at TIMESTAMP DEFAULT NOW()
    )
  `);
});

beforeEach(async () => {
  // Insert test data
  await pool.query(`
    INSERT INTO accounts (account_id, customer_name, balance, currency) VALUES
    ('ACC-001', 'Alice Johnson', 10000, 'USD'),
    ('ACC-002', 'Bob Smith', 5000, 'USD'),
    ('ACC-003', 'Charlie Brown', 2500, 'USD')
    ON CONFLICT (account_id) DO UPDATE SET
      balance = EXCLUDED.balance,
      customer_name = EXCLUDED.customer_name
  `);

  // Clear transactions
  await pool.query('DELETE FROM transactions');
});

afterAll(async () => {
  // Cleanup
  await pool.query('DROP TABLE IF EXISTS transactions');
  await pool.query('DROP TABLE IF EXISTS accounts');
  await pool.end();
});

describe('Banking API Integration Tests', () => {
  describe('GET /api/health', () => {
    test('should return health status', async () => {
      const response = await request(app)
        .get('/api/health')
        .expect(200)
        .expect('Content-Type', /json/);

      expect(response.body).toEqual({
        status: 'ok',
        timestamp: expect.any(String)
      });
    });
  });

  describe('GET /api/accounts/:accountId/balance', () => {
    test('should return account balance', async () => {
      const response = await request(app)
        .get('/api/accounts/ACC-001/balance')
        .expect(200);

      expect(response.body).toEqual({
        accountId: 'ACC-001',
        balance: 10000,
        currency: 'USD'
      });
    });

    test('should return 404 for non-existent account', async () => {
      const response = await request(app)
        .get('/api/accounts/ACC-999/balance')
        .expect(404);

      expect(response.body).toEqual({
        error: 'Account not found'
      });
    });
  });

  describe('POST /api/transfer', () => {
    test('should transfer money successfully', async () => {
      const transferData = {
        fromAccount: 'ACC-001',
        toAccount: 'ACC-002',
        amount: 500,
        description: 'Test transfer'
      };

      const response = await request(app)
        .post('/api/transfer')
        .send(transferData)
        .expect(201);

      expect(response.body).toMatchObject({
        transactionId: expect.stringMatching(/^TXN-/),
        fromAccount: 'ACC-001',
        toAccount: 'ACC-002',
        amount: 500,
        status: 'completed'
      });

      // Verify balances updated
      const senderBalance = await request(app)
        .get('/api/accounts/ACC-001/balance');
      expect(senderBalance.body.balance).toBe(9500);

      const receiverBalance = await request(app)
        .get('/api/accounts/ACC-002/balance');
      expect(receiverBalance.body.balance).toBe(5500);
    });

    test('should reject transfer with insufficient funds', async () => {
      const transferData = {
        fromAccount: 'ACC-003', // Balance: 2500
        toAccount: 'ACC-001',
        amount: 5000
      };

      const response = await request(app)
        .post('/api/transfer')
        .send(transferData)
        .expect(400);

      expect(response.body).toEqual({
        error: 'Insufficient funds'
      });

      // Verify balances unchanged
      const senderBalance = await request(app)
        .get('/api/accounts/ACC-003/balance');
      expect(senderBalance.body.balance).toBe(2500);
    });

    test('should reject transfer to same account', async () => {
      const transferData = {
        fromAccount: 'ACC-001',
        toAccount: 'ACC-001',
        amount: 100
      };

      await request(app)
        .post('/api/transfer')
        .send(transferData)
        .expect(400)
        .expect({ error: 'Cannot transfer to same account' });
    });

    test('should reject transfer with negative amount', async () => {
      const transferData = {
        fromAccount: 'ACC-001',
        toAccount: 'ACC-002',
        amount: -100
      };

      await request(app)
        .post('/api/transfer')
        .send(transferData)
        .expect(400)
        .expect({ error: 'Amount must be positive' });
    });

    test('should reject transfer with missing fields', async () => {
      const transferData = {
        fromAccount: 'ACC-001'
        // Missing toAccount and amount
      };

      await request(app)
        .post('/api/transfer')
        .send(transferData)
        .expect(400)
        .expect({ error: 'Missing required fields' });
    });

    test('should return 404 for non-existent sender', async () => {
      const transferData = {
        fromAccount: 'ACC-999',
        toAccount: 'ACC-001',
        amount: 100
      };

      await request(app)
        .post('/api/transfer')
        .send(transferData)
        .expect(404)
        .expect({ error: 'Sender account not found' });
    });

    test('should return 404 for non-existent receiver', async () => {
      const transferData = {
        fromAccount: 'ACC-001',
        toAccount: 'ACC-999',
        amount: 100
      };

      await request(app)
        .post('/api/transfer')
        .send(transferData)
        .expect(404)
        .expect({ error: 'Receiver account not found' });
    });
  });

  describe('GET /api/accounts/:accountId/transactions', () => {
    beforeEach(async () => {
      // Create test transactions
      await pool.query(`
        INSERT INTO transactions (transaction_id, from_account, to_account, amount, description, status)
        VALUES
        ('TXN-001', 'ACC-001', 'ACC-002', 100, 'Test 1', 'completed'),
        ('TXN-002', 'ACC-002', 'ACC-001', 50, 'Test 2', 'completed'),
        ('TXN-003', 'ACC-001', 'ACC-003', 200, 'Test 3', 'completed')
      `);
    });

    test('should return transaction history', async () => {
      const response = await request(app)
        .get('/api/accounts/ACC-001/transactions')
        .expect(200);

      expect(response.body.accountId).toBe('ACC-001');
      expect(response.body.transactions).toHaveLength(3);
      expect(response.body.transactions[0]).toMatchObject({
        transactionId: expect.any(String),
        amount: expect.any(Number),
        status: 'completed'
      });
    });

    test('should respect limit parameter', async () => {
      const response = await request(app)
        .get('/api/accounts/ACC-001/transactions?limit=2')
        .expect(200);

      expect(response.body.transactions).toHaveLength(2);
    });

    test('should return empty array for account with no transactions', async () => {
      // Create new account with no transactions
      await pool.query(
        "INSERT INTO accounts (account_id, customer_name, balance) VALUES ('ACC-999', 'Test', 1000)"
      );

      const response = await request(app)
        .get('/api/accounts/ACC-999/transactions')
        .expect(200);

      expect(response.body.transactions).toHaveLength(0);
    });
  });

  describe('End-to-End Transfer Flow', () => {
    test('should handle multiple sequential transfers', async () => {
      // Initial balances: ACC-001: 10000, ACC-002: 5000, ACC-003: 2500

      // Transfer 1: ACC-001 → ACC-002 (500)
      await request(app)
        .post('/api/transfer')
        .send({ fromAccount: 'ACC-001', toAccount: 'ACC-002', amount: 500 })
        .expect(201);

      // Transfer 2: ACC-002 → ACC-003 (1000)
      await request(app)
        .post('/api/transfer')
        .send({ fromAccount: 'ACC-002', toAccount: 'ACC-003', amount: 1000 })
        .expect(201);

      // Transfer 3: ACC-003 → ACC-001 (500)
      await request(app)
        .post('/api/transfer')
        .send({ fromAccount: 'ACC-003', toAccount: 'ACC-001', amount: 500 })
        .expect(201);

      // Verify final balances
      const balance1 = await request(app).get('/api/accounts/ACC-001/balance');
      expect(balance1.body.balance).toBe(10000); // 10000 - 500 + 500

      const balance2 = await request(app).get('/api/accounts/ACC-002/balance');
      expect(balance2.body.balance).toBe(4500); // 5000 + 500 - 1000

      const balance3 = await request(app).get('/api/accounts/ACC-003/balance');
      expect(balance3.body.balance).toBe(3000); // 2500 + 1000 - 500

      // Verify transaction count
      const history = await request(app).get('/api/accounts/ACC-001/transactions');
      expect(history.body.transactions).toHaveLength(2); // 1 sent, 1 received
    });

    test('should rollback on transfer failure', async () => {
      // Attempt transfer that will fail (insufficient funds)
      await request(app)
        .post('/api/transfer')
        .send({ fromAccount: 'ACC-003', toAccount: 'ACC-001', amount: 10000 })
        .expect(400);

      // Verify balances unchanged
      const balance = await request(app).get('/api/accounts/ACC-003/balance');
      expect(balance.body.balance).toBe(2500);

      // Verify no transaction recorded
      const history = await request(app).get('/api/accounts/ACC-003/transactions');
      expect(history.body.transactions).toHaveLength(0);
    });
  });
});
```

### Running Integration Tests

```bash
# Install dependencies
npm install --save-dev jest supertest

# Setup test database
createdb banking_test

# Environment variables
export DB_NAME=banking_test
export DB_USER=postgres
export DB_PASSWORD=password

# Run tests
npm test

# Run only integration tests
npm test -- __tests__/integration
```

---

## 🎯 Key Takeaways

### Integration Testing Best Practices

1. **Use separate test database**
   - Never test on production data
   - Use `banking_test` instead of `banking_db`

2. **Clean up between tests**
   ```javascript
   beforeEach(async () => {
     await db.query('DELETE FROM transactions');
     await db.query('DELETE FROM accounts');
     // Insert fresh test data
   });
   ```

3. **Test real scenarios**
   - Happy path (successful transfer)
   - Error cases (insufficient funds)
   - Edge cases (transfer to self)
   - Concurrent operations

4. **Verify side effects**
   ```javascript
   // After transfer, check:
   // - Both account balances
   // - Transaction recorded
   // - Audit logs created
   ```

5. **Use transactions for isolation**
   ```javascript
   beforeEach(() => db.query('BEGIN'));
   afterEach(() => db.query('ROLLBACK'));
   ```

### Common Pitfalls

- ❌ Testing on production database
- ❌ Tests depend on each other
- ❌ Not cleaning up test data
- ❌ Hardcoded IDs (use factories)
- ❌ Ignoring async operations
- ❌ Not testing error cases

### Performance Tips

- Run integration tests in parallel (Jest default)
- Use transactions for fast rollback
- Minimize database setup time
- Mock external APIs (payment gateways)
- Use test containers for CI/CD

---

**File**: `Q34_Testing_Integration_Tests.md`  
**Status**: ✅ Complete with Supertest examples, database testing, and end-to-end banking flows
