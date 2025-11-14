# Node.js Interview Question: Error Handling
## Question 8: How to Handle Errors in Node.js? What are Error-First Callbacks?

---

## 📋 Summary of What Will Be Covered

In this comprehensive guide, you'll learn:

1. **Error-first callbacks** - The Node.js error handling convention
2. **Synchronous vs Asynchronous errors** - Different handling strategies
3. **Try-catch blocks** - When and where to use them
4. **Promise error handling** - .catch() and try-catch with async/await
5. **Custom error classes** - Creating meaningful error types
6. **Error propagation** - How errors bubble up the call stack
7. **Banking Examples**: Transaction validation errors, payment failures, account operations

**Why This Matters in Banking**:
- Transaction failures must be caught and logged
- User input validation prevents fraud
- Database errors need graceful handling
- Payment processing errors require specific handling
- Audit trails depend on proper error logging
- System stability requires defensive programming

---

## 🎯 What is Error-First Callback Pattern?

The **error-first callback** pattern is a Node.js convention where:
1. **First parameter** is always an error object (or `null` if no error)
2. **Subsequent parameters** contain the result data

```javascript
// Error-first callback signature
function callback(error, result) {
  if (error) {
    // Handle error
    console.error('Error:', error);
    return;
  }
  
  // Process result
  console.log('Success:', result);
}
```

### Why This Pattern Exists

Node.js is **asynchronous** - operations complete at different times. The error-first pattern provides a consistent way to handle both success and failure.

```javascript
import fs from 'fs';

// ❌ BAD: No error handling
fs.readFile('data.txt', 'utf8', (err, data) => {
  console.log(data);  // Crashes if file doesn't exist
});

// ✅ GOOD: Error-first pattern
fs.readFile('data.txt', 'utf8', (err, data) => {
  if (err) {
    console.error('Failed to read file:', err.message);
    return;
  }
  console.log('File content:', data);
});
```

---

## 🔍 Deep Dive: Types of Errors in Node.js

### 1. Synchronous Errors (Use try-catch)

```javascript
// Synchronous operation - error thrown immediately
try {
  const data = JSON.parse('invalid json');  // Throws SyntaxError
} catch (err) {
  console.error('Parse error:', err.message);
}
```

### 2. Asynchronous Errors with Callbacks (Use error parameter)

```javascript
import fs from 'fs';

// Asynchronous operation - error passed to callback
fs.readFile('missing.txt', 'utf8', (err, data) => {
  if (err) {
    console.error('File error:', err.message);
    return;
  }
  console.log(data);
});
```

### 3. Promise Errors (Use .catch() or try-catch)

```javascript
import fs from 'fs/promises';

// Promise-based - use .catch()
fs.readFile('data.txt', 'utf8')
  .then(data => console.log(data))
  .catch(err => console.error('Error:', err.message));

// Or with async/await
async function readData() {
  try {
    const data = await fs.readFile('data.txt', 'utf8');
    console.log(data);
  } catch (err) {
    console.error('Error:', err.message);
  }
}
```

### 4. Event Emitter Errors (Use 'error' event)

```javascript
import { EventEmitter } from 'events';

const emitter = new EventEmitter();

// Listen for errors
emitter.on('error', (err) => {
  console.error('Emitter error:', err.message);
});

// Emit error
emitter.emit('error', new Error('Something went wrong'));
```

---

## 📊 Visual: Error Handling Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    Synchronous Code                         │
│  ┌──────────────┐                                           │
│  │  try-catch   │ ──────► Catches synchronous errors        │
│  └──────────────┘                                           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                 Asynchronous Callbacks                      │
│  ┌──────────────────────┐                                   │
│  │  Error-first pattern │ ──► if (err) { handle }          │
│  └──────────────────────┘                                   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      Promises                               │
│  ┌────────────┐    ┌──────────────────┐                    │
│  │  .catch()  │ OR │ try-catch (await)│                    │
│  └────────────┘    └──────────────────┘                    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   Event Emitters                            │
│  ┌─────────────────────────┐                                │
│  │  .on('error', handler)  │                                │
│  └─────────────────────────┘                                │
└─────────────────────────────────────────────────────────────┘
```

---

## 🏦 Example 1: Banking Transaction Error Handling

This example demonstrates comprehensive error handling for banking transactions.

**File: `transaction-service.js`**

```javascript
import EventEmitter from 'events';

/**
 * Banking Transaction Service with Comprehensive Error Handling
 * Demonstrates: Error-first callbacks, custom errors, error propagation
 */

console.log('💳 Banking Transaction Service\n');
console.log('='.repeat(70));

// ============================================
// Custom Error Classes
// ============================================

class BankingError extends Error {
  constructor(message, code, details = {}) {
    super(message);
    this.name = 'BankingError';
    this.code = code;
    this.details = details;
    this.timestamp = new Date().toISOString();
    
    // Maintains proper stack trace
    Error.captureStackTrace(this, this.constructor);
  }
  
  toJSON() {
    return {
      name: this.name,
      message: this.message,
      code: this.code,
      details: this.details,
      timestamp: this.timestamp
    };
  }
}

class InsufficientFundsError extends BankingError {
  constructor(accountId, requested, available) {
    super(
      `Insufficient funds in account ${accountId}`,
      'INSUFFICIENT_FUNDS',
      { accountId, requested, available, shortfall: requested - available }
    );
    this.name = 'InsufficientFundsError';
  }
}

class AccountNotFoundError extends BankingError {
  constructor(accountId) {
    super(
      `Account ${accountId} not found`,
      'ACCOUNT_NOT_FOUND',
      { accountId }
    );
    this.name = 'AccountNotFoundError';
  }
}

class TransactionLimitError extends BankingError {
  constructor(accountId, amount, limit) {
    super(
      `Transaction amount $${amount} exceeds daily limit $${limit}`,
      'TRANSACTION_LIMIT_EXCEEDED',
      { accountId, amount, limit }
    );
    this.name = 'TransactionLimitError';
  }
}

class InvalidTransactionError extends BankingError {
  constructor(reason, details = {}) {
    super(
      `Invalid transaction: ${reason}`,
      'INVALID_TRANSACTION',
      details
    );
    this.name = 'InvalidTransactionError';
  }
}

class NetworkError extends BankingError {
  constructor(operation, originalError) {
    super(
      `Network error during ${operation}`,
      'NETWORK_ERROR',
      { operation, originalMessage: originalError.message }
    );
    this.name = 'NetworkError';
  }
}

// ============================================
// Mock Database
// ============================================

const accounts = {
  'ACC001': { id: 'ACC001', name: 'John Doe', balance: 5000, dailyLimit: 10000 },
  'ACC002': { id: 'ACC002', name: 'Jane Smith', balance: 1000, dailyLimit: 5000 },
  'ACC003': { id: 'ACC003', name: 'Bob Johnson', balance: 500, dailyLimit: 2000 }
};

const transactionHistory = [];

// ============================================
// Transaction Service
// ============================================

class TransactionService extends EventEmitter {
  constructor() {
    super();
    this.dailyTransactions = {};  // Track daily totals
  }
  
  /**
   * Validate transaction (synchronous)
   * Uses: try-catch for sync validation
   */
  validateTransaction(fromAccount, toAccount, amount) {
    console.log(`\n🔍 Validating transaction...`);
    console.log(`   From: ${fromAccount} → To: ${toAccount}`);
    console.log(`   Amount: $${amount}`);
    
    try {
      // Validation 1: Amount must be positive
      if (amount <= 0) {
        throw new InvalidTransactionError(
          'Amount must be positive',
          { amount }
        );
      }
      
      // Validation 2: Amount must be reasonable
      if (amount > 1000000) {
        throw new InvalidTransactionError(
          'Amount exceeds maximum transaction limit',
          { amount, maxLimit: 1000000 }
        );
      }
      
      // Validation 3: Cannot transfer to same account
      if (fromAccount === toAccount) {
        throw new InvalidTransactionError(
          'Cannot transfer to same account',
          { fromAccount, toAccount }
        );
      }
      
      // Validation 4: Check if amount is a valid number
      if (isNaN(amount) || !isFinite(amount)) {
        throw new InvalidTransactionError(
          'Invalid amount format',
          { amount }
        );
      }
      
      console.log(`   ✅ Validation passed`);
      return true;
      
    } catch (err) {
      console.log(`   ❌ Validation failed: ${err.message}`);
      throw err;  // Re-throw to caller
    }
  }
  
  /**
   * Get account (callback-based)
   * Uses: Error-first callback pattern
   */
  getAccount(accountId, callback) {
    // Simulate async database lookup
    setImmediate(() => {
      console.log(`\n📋 Looking up account: ${accountId}`);
      
      const account = accounts[accountId];
      
      if (!account) {
        const error = new AccountNotFoundError(accountId);
        console.log(`   ❌ ${error.message}`);
        return callback(error);  // Error-first: pass error as first param
      }
      
      console.log(`   ✅ Found: ${account.name} - Balance: $${account.balance}`);
      callback(null, account);  // Error-first: null error, account as result
    });
  }
  
  /**
   * Check sufficient balance (callback-based)
   * Uses: Error-first callback pattern
   */
  checkBalance(accountId, amount, callback) {
    this.getAccount(accountId, (err, account) => {
      // Error-first: check error first
      if (err) {
        return callback(err);  // Propagate error
      }
      
      console.log(`\n💰 Checking balance for ${accountId}`);
      console.log(`   Required: $${amount}`);
      console.log(`   Available: $${account.balance}`);
      
      if (account.balance < amount) {
        const error = new InsufficientFundsError(
          accountId,
          amount,
          account.balance
        );
        console.log(`   ❌ ${error.message}`);
        return callback(error);
      }
      
      console.log(`   ✅ Sufficient funds`);
      callback(null, true);
    });
  }
  
  /**
   * Check daily limit (callback-based)
   */
  checkDailyLimit(accountId, amount, callback) {
    this.getAccount(accountId, (err, account) => {
      if (err) return callback(err);
      
      console.log(`\n📊 Checking daily limit for ${accountId}`);
      
      const today = new Date().toDateString();
      const dailyKey = `${accountId}-${today}`;
      const dailyTotal = (this.dailyTransactions[dailyKey] || 0) + amount;
      
      console.log(`   Daily total: $${dailyTotal}`);
      console.log(`   Daily limit: $${account.dailyLimit}`);
      
      if (dailyTotal > account.dailyLimit) {
        const error = new TransactionLimitError(
          accountId,
          amount,
          account.dailyLimit
        );
        console.log(`   ❌ ${error.message}`);
        return callback(error);
      }
      
      console.log(`   ✅ Within daily limit`);
      callback(null, true);
    });
  }
  
  /**
   * Execute transfer (promise-based)
   * Uses: Promises with async/await
   */
  async executeTransfer(fromAccount, toAccount, amount) {
    console.log(`\n🔄 Executing transfer...`);
    
    return new Promise((resolve, reject) => {
      // Simulate network delay
      setTimeout(() => {
        try {
          // Simulate random network error (10% chance)
          if (Math.random() < 0.1) {
            throw new NetworkError('transfer execution', new Error('Connection timeout'));
          }
          
          // Update balances
          accounts[fromAccount].balance -= amount;
          accounts[toAccount].balance += amount;
          
          // Record transaction
          const transaction = {
            id: `TXN${Date.now()}`,
            from: fromAccount,
            to: toAccount,
            amount,
            timestamp: new Date().toISOString(),
            status: 'completed'
          };
          
          transactionHistory.push(transaction);
          
          // Update daily total
          const today = new Date().toDateString();
          const dailyKey = `${fromAccount}-${today}`;
          this.dailyTransactions[dailyKey] = 
            (this.dailyTransactions[dailyKey] || 0) + amount;
          
          console.log(`   ✅ Transfer completed: ${transaction.id}`);
          
          // Emit success event
          this.emit('transfer:success', transaction);
          
          resolve(transaction);
          
        } catch (err) {
          console.log(`   ❌ Transfer failed: ${err.message}`);
          
          // Emit error event
          this.emit('transfer:error', err);
          
          reject(err);
        }
      }, 100);
    });
  }
  
  /**
   * Process transaction (orchestrates all steps)
   * Demonstrates: Mixed error handling (callbacks + promises + try-catch)
   */
  async processTransaction(fromAccount, toAccount, amount) {
    console.log(`\n${'='.repeat(70)}`);
    console.log(`\n💳 Processing Transaction`);
    console.log(`   From: ${fromAccount}`);
    console.log(`   To: ${toAccount}`);
    console.log(`   Amount: $${amount}`);
    
    try {
      // Step 1: Synchronous validation (try-catch)
      this.validateTransaction(fromAccount, toAccount, amount);
      
      // Step 2: Check source account exists (callback → promisify)
      await new Promise((resolve, reject) => {
        this.getAccount(fromAccount, (err, account) => {
          if (err) return reject(err);
          resolve(account);
        });
      });
      
      // Step 3: Check destination account exists (callback → promisify)
      await new Promise((resolve, reject) => {
        this.getAccount(toAccount, (err, account) => {
          if (err) return reject(err);
          resolve(account);
        });
      });
      
      // Step 4: Check sufficient balance (callback → promisify)
      await new Promise((resolve, reject) => {
        this.checkBalance(fromAccount, amount, (err, sufficient) => {
          if (err) return reject(err);
          resolve(sufficient);
        });
      });
      
      // Step 5: Check daily limit (callback → promisify)
      await new Promise((resolve, reject) => {
        this.checkDailyLimit(fromAccount, amount, (err, withinLimit) => {
          if (err) return reject(err);
          resolve(withinLimit);
        });
      });
      
      // Step 6: Execute transfer (promise-based)
      const transaction = await this.executeTransfer(fromAccount, toAccount, amount);
      
      console.log(`\n✅ Transaction Successful!`);
      console.log(`   Transaction ID: ${transaction.id}`);
      console.log(`   New balance (${fromAccount}): $${accounts[fromAccount].balance}`);
      console.log(`   New balance (${toAccount}): $${accounts[toAccount].balance}`);
      
      return transaction;
      
    } catch (err) {
      // Centralized error handling
      console.log(`\n❌ Transaction Failed!`);
      console.log(`   Error: ${err.message}`);
      console.log(`   Code: ${err.code || 'UNKNOWN'}`);
      
      // Log error details
      if (err instanceof BankingError) {
        console.log(`   Details:`, JSON.stringify(err.details, null, 2));
      }
      
      // Emit error event for monitoring
      this.emit('transaction:failed', err);
      
      // Re-throw for caller to handle
      throw err;
    }
  }
  
  /**
   * Get transaction history
   */
  getTransactionHistory() {
    return transactionHistory;
  }
}

// ============================================
// Test Transaction Service
// ============================================

async function runTests() {
  const service = new TransactionService();
  
  // Listen to events
  service.on('transfer:success', (txn) => {
    console.log(`\n📢 Event: Transfer succeeded - ${txn.id}`);
  });
  
  service.on('transfer:error', (err) => {
    console.log(`\n📢 Event: Transfer error - ${err.message}`);
  });
  
  service.on('transaction:failed', (err) => {
    console.log(`\n📢 Event: Transaction failed - ${err.code}`);
  });
  
  // Test 1: Successful transaction
  console.log('\n' + '─'.repeat(70));
  console.log('\n📊 Test 1: Successful Transaction\n');
  
  try {
    const txn1 = await service.processTransaction('ACC001', 'ACC002', 500);
    console.log(`\n✅ Test 1 Passed`);
  } catch (err) {
    console.log(`\n❌ Test 1 Failed: ${err.message}`);
  }
  
  // Test 2: Insufficient funds
  console.log('\n' + '─'.repeat(70));
  console.log('\n📊 Test 2: Insufficient Funds Error\n');
  
  try {
    const txn2 = await service.processTransaction('ACC003', 'ACC001', 1000);
    console.log(`\n❌ Test 2 Failed: Should have thrown error`);
  } catch (err) {
    if (err instanceof InsufficientFundsError) {
      console.log(`\n✅ Test 2 Passed: Caught ${err.name}`);
      console.log(`   Shortfall: $${err.details.shortfall}`);
    } else {
      console.log(`\n❌ Test 2 Failed: Wrong error type`);
    }
  }
  
  // Test 3: Account not found
  console.log('\n' + '─'.repeat(70));
  console.log('\n📊 Test 3: Account Not Found Error\n');
  
  try {
    const txn3 = await service.processTransaction('ACC999', 'ACC001', 100);
    console.log(`\n❌ Test 3 Failed: Should have thrown error`);
  } catch (err) {
    if (err instanceof AccountNotFoundError) {
      console.log(`\n✅ Test 3 Passed: Caught ${err.name}`);
    } else {
      console.log(`\n❌ Test 3 Failed: Wrong error type`);
    }
  }
  
  // Test 4: Invalid amount
  console.log('\n' + '─'.repeat(70));
  console.log('\n📊 Test 4: Invalid Transaction Error\n');
  
  try {
    const txn4 = await service.processTransaction('ACC001', 'ACC002', -100);
    console.log(`\n❌ Test 4 Failed: Should have thrown error`);
  } catch (err) {
    if (err instanceof InvalidTransactionError) {
      console.log(`\n✅ Test 4 Passed: Caught ${err.name}`);
    } else {
      console.log(`\n❌ Test 4 Failed: Wrong error type`);
    }
  }
  
  // Test 5: Same account transfer
  console.log('\n' + '─'.repeat(70));
  console.log('\n📊 Test 5: Same Account Error\n');
  
  try {
    const txn5 = await service.processTransaction('ACC001', 'ACC001', 100);
    console.log(`\n❌ Test 5 Failed: Should have thrown error`);
  } catch (err) {
    if (err instanceof InvalidTransactionError) {
      console.log(`\n✅ Test 5 Passed: Caught ${err.name}`);
    } else {
      console.log(`\n❌ Test 5 Failed: Wrong error type`);
    }
  }
  
  // Test 6: Transaction limit exceeded
  console.log('\n' + '─'.repeat(70));
  console.log('\n📊 Test 6: Daily Limit Exceeded\n');
  
  try {
    const txn6 = await service.processTransaction('ACC002', 'ACC001', 6000);
    console.log(`\n❌ Test 6 Failed: Should have thrown error`);
  } catch (err) {
    if (err instanceof TransactionLimitError) {
      console.log(`\n✅ Test 6 Passed: Caught ${err.name}`);
    } else {
      console.log(`\n❌ Test 6 Failed: Wrong error type`);
    }
  }
  
  // Display final state
  console.log('\n' + '='.repeat(70));
  console.log('\n📊 Final Account Balances:\n');
  
  Object.values(accounts).forEach(account => {
    console.log(`   ${account.id} (${account.name}): $${account.balance}`);
  });
  
  console.log('\n📝 Transaction History:\n');
  service.getTransactionHistory().forEach(txn => {
    console.log(`   ${txn.id}: ${txn.from} → ${txn.to} ($${txn.amount})`);
  });
  
  console.log('\n' + '='.repeat(70));
  console.log('\n✅ All tests completed!\n');
}

// Run tests
runTests().catch(err => {
  console.error('\n💥 Unhandled error:', err);
  process.exit(1);
});
```

**To test this example:**

```bash
node transaction-service.js
```

**Expected Output:**

```
💳 Banking Transaction Service

======================================================================

──────────────────────────────────────────────────────────────────────

📊 Test 1: Successful Transaction

======================================================================

💳 Processing Transaction
   From: ACC001
   To: ACC002
   Amount: $500

🔍 Validating transaction...
   From: ACC001 → To: ACC002
   Amount: $500
   ✅ Validation passed

📋 Looking up account: ACC001
   ✅ Found: John Doe - Balance: $5000

📋 Looking up account: ACC002
   ✅ Found: Jane Smith - Balance: $1000

💰 Checking balance for ACC001
   Required: $500
   Available: $5000
   ✅ Sufficient funds

📊 Checking daily limit for ACC001
   Daily total: $500
   Daily limit: $10000
   ✅ Within daily limit

🔄 Executing transfer...
   ✅ Transfer completed: TXN1731456789012

📢 Event: Transfer succeeded - TXN1731456789012

✅ Transaction Successful!
   Transaction ID: TXN1731456789012
   New balance (ACC001): $4500
   New balance (ACC002): $1500

✅ Test 1 Passed

──────────────────────────────────────────────────────────────────────

📊 Test 2: Insufficient Funds Error

======================================================================

💳 Processing Transaction
   From: ACC003
   To: ACC001
   Amount: $1000

🔍 Validating transaction...
   From: ACC003 → To: ACC001
   Amount: $1000
   ✅ Validation passed

📋 Looking up account: ACC003
   ✅ Found: Bob Johnson - Balance: $500

📋 Looking up account: ACC001
   ✅ Found: John Doe - Balance: $4500

💰 Checking balance for ACC003
   Required: $1000
   Available: $500
   ❌ Insufficient funds in account ACC003

❌ Transaction Failed!
   Error: Insufficient funds in account ACC003
   Code: INSUFFICIENT_FUNDS
   Details: {
  "accountId": "ACC003",
  "requested": 1000,
  "available": 500,
  "shortfall": 500
}

📢 Event: Transaction failed - INSUFFICIENT_FUNDS

✅ Test 2 Passed: Caught InsufficientFundsError
   Shortfall: $500

──────────────────────────────────────────────────────────────────────

📊 Test 3: Account Not Found Error

======================================================================

💳 Processing Transaction
   From: ACC999
   To: ACC001
   Amount: $100

🔍 Validating transaction...
   From: ACC999 → To: ACC001
   Amount: $100
   ✅ Validation passed

📋 Looking up account: ACC999
   ❌ Account ACC999 not found

❌ Transaction Failed!
   Error: Account ACC999 not found
   Code: ACCOUNT_NOT_FOUND
   Details: {
  "accountId": "ACC999"
}

📢 Event: Transaction failed - ACCOUNT_NOT_FOUND

✅ Test 3 Passed: Caught AccountNotFoundError

──────────────────────────────────────────────────────────────────────

📊 Test 4: Invalid Transaction Error

======================================================================

💳 Processing Transaction
   From: ACC001
   To: ACC002
   Amount: $-100

🔍 Validating transaction...
   From: ACC001 → To: ACC002
   Amount: $-100
   ❌ Validation failed: Invalid transaction: Amount must be positive

❌ Transaction Failed!
   Error: Invalid transaction: Amount must be positive
   Code: INVALID_TRANSACTION
   Details: {
  "amount": -100
}

📢 Event: Transaction failed - INVALID_TRANSACTION

✅ Test 4 Passed: Caught InvalidTransactionError

──────────────────────────────────────────────────────────────────────

📊 Test 5: Same Account Error

======================================================================

💳 Processing Transaction
   From: ACC001
   To: ACC001
   Amount: $100

🔍 Validating transaction...
   From: ACC001 → To: ACC001
   Amount: $100
   ❌ Validation failed: Invalid transaction: Cannot transfer to same account

❌ Transaction Failed!
   Error: Invalid transaction: Cannot transfer to same account
   Code: INVALID_TRANSACTION
   Details: {
  "fromAccount": "ACC001",
  "toAccount": "ACC001"
}

📢 Event: Transaction failed - INVALID_TRANSACTION

✅ Test 5 Passed: Caught InvalidTransactionError

──────────────────────────────────────────────────────────────────────

📊 Test 6: Daily Limit Exceeded

======================================================================

💳 Processing Transaction
   From: ACC002
   To: ACC001
   Amount: $6000

🔍 Validating transaction...
   From: ACC002 → To: ACC001
   Amount: $6000
   ✅ Validation passed

📋 Looking up account: ACC002
   ✅ Found: Jane Smith - Balance: $1500

📋 Looking up account: ACC001
   ✅ Found: John Doe - Balance: $4500

💰 Checking balance for ACC002
   Required: $6000
   Available: $1500
   ❌ Insufficient funds in account ACC002

❌ Transaction Failed!
   Error: Insufficient funds in account ACC002
   Code: INSUFFICIENT_FUNDS
   Details: {
  "accountId": "ACC002",
  "requested": 6000,
  "available": 1500,
  "shortfall": 4500
}

📢 Event: Transaction failed - INSUFFICIENT_FUNDS

❌ Test 6 Failed: Wrong error type

======================================================================

📊 Final Account Balances:

   ACC001 (John Doe): $4500
   ACC002 (Jane Smith): $1500
   ACC003 (Bob Johnson): $500

📝 Transaction History:

   TXN1731456789012: ACC001 → ACC002 ($500)

======================================================================

✅ All tests completed!
```

**Key Learnings from Example 1**:

1. **Custom Error Classes**: Extend `Error` for meaningful error types
2. **Error-First Callbacks**: Always check error parameter first
3. **Mixed Error Handling**: Combine callbacks, promises, and try-catch
4. **Error Propagation**: Errors bubble up automatically with promises
5. **Event Emitters**: Use 'error' events for monitoring
6. **Centralized Handling**: Single try-catch can handle all async operations

---

## 🏦 Example 2: Async Error Handling with Promises and Async/Await

This example demonstrates modern error handling with Promises and async/await.

**File: `payment-processor.js`**

```javascript
import crypto from 'crypto';

/**
 * Payment Processing with Promise-based Error Handling
 * Demonstrates: .catch(), async/await, Promise.all/allSettled, error recovery
 */

console.log('💰 Payment Processing Service\n');
console.log('='.repeat(70));

// ============================================
// Payment Gateway API (Simulated)
// ============================================

class PaymentGateway {
  /**
   * Process credit card payment (Promise-based)
   */
  static processCreditCard(cardNumber, amount) {
    return new Promise((resolve, reject) => {
      console.log(`\n💳 Processing credit card payment: $${amount}`);
      
      setTimeout(() => {
        // Simulate various error conditions
        if (cardNumber.startsWith('4000')) {
          return reject(new Error('Card declined - insufficient funds'));
        }
        
        if (cardNumber.startsWith('5000')) {
          return reject(new Error('Card declined - invalid card'));
        }
        
        if (cardNumber.startsWith('6000')) {
          return reject(new Error('Payment gateway timeout'));
        }
        
        // Success
        const transactionId = `CC_${crypto.randomBytes(8).toString('hex')}`;
        console.log(`   ✅ Payment approved: ${transactionId}`);
        resolve({
          transactionId,
          amount,
          method: 'credit_card',
          timestamp: new Date().toISOString()
        });
      }, 500);
    });
  }
  
  /**
   * Process bank transfer (Promise-based)
   */
  static processBankTransfer(accountNumber, amount) {
    return new Promise((resolve, reject) => {
      console.log(`\n🏦 Processing bank transfer: $${amount}`);
      
      setTimeout(() => {
        // Simulate errors
        if (accountNumber.startsWith('9999')) {
          return reject(new Error('Account not found'));
        }
        
        if (amount > 10000) {
          return reject(new Error('Transfer limit exceeded'));
        }
        
        // Success
        const transactionId = `BT_${crypto.randomBytes(8).toString('hex')}`;
        console.log(`   ✅ Transfer completed: ${transactionId}`);
        resolve({
          transactionId,
          amount,
          method: 'bank_transfer',
          timestamp: new Date().toISOString()
        });
      }, 700);
    });
  }
  
  /**
   * Verify payment (Promise-based)
   */
  static verifyPayment(transactionId) {
    return new Promise((resolve, reject) => {
      console.log(`\n🔍 Verifying payment: ${transactionId}`);
      
      setTimeout(() => {
        // Simulate random verification failure (10% chance)
        if (Math.random() < 0.1) {
          return reject(new Error('Verification service unavailable'));
        }
        
        console.log(`   ✅ Payment verified`);
        resolve({ verified: true, transactionId });
      }, 300);
    });
  }
  
  /**
   * Send receipt (Promise-based)
   */
  static sendReceipt(email, transactionId) {
    return new Promise((resolve, reject) => {
      console.log(`\n📧 Sending receipt to: ${email}`);
      
      setTimeout(() => {
        // Simulate email service error (5% chance)
        if (Math.random() < 0.05) {
          return reject(new Error('Email service unavailable'));
        }
        
        console.log(`   ✅ Receipt sent`);
        resolve({ sent: true, email, transactionId });
      }, 200);
    });
  }
}

// ============================================
// Payment Service with Error Handling
// ============================================

class PaymentService {
  /**
   * Method 1: Using .then() and .catch()
   */
  static processPaymentWithThenCatch(cardNumber, amount) {
    console.log(`\n${'─'.repeat(70)}`);
    console.log(`\n📊 Method 1: Using .then() and .catch()\n`);
    
    return PaymentGateway.processCreditCard(cardNumber, amount)
      .then(payment => {
        console.log(`   Payment successful: ${payment.transactionId}`);
        return PaymentGateway.verifyPayment(payment.transactionId);
      })
      .then(verification => {
        console.log(`   Verification successful`);
        return verification;
      })
      .catch(err => {
        console.error(`   ❌ Error in payment chain: ${err.message}`);
        throw err;  // Re-throw or handle
      });
  }
  
  /**
   * Method 2: Using async/await with try-catch
   */
  static async processPaymentWithAsyncAwait(cardNumber, amount, email) {
    console.log(`\n${'─'.repeat(70)}`);
    console.log(`\n📊 Method 2: Using async/await with try-catch\n`);
    
    try {
      // Step 1: Process payment
      const payment = await PaymentGateway.processCreditCard(cardNumber, amount);
      console.log(`   Payment successful: ${payment.transactionId}`);
      
      // Step 2: Verify payment
      const verification = await PaymentGateway.verifyPayment(payment.transactionId);
      console.log(`   Verification successful`);
      
      // Step 3: Send receipt (non-critical - don't fail if this fails)
      try {
        await PaymentGateway.sendReceipt(email, payment.transactionId);
      } catch (receiptErr) {
        // Log but don't fail the transaction
        console.warn(`   ⚠️  Receipt failed (non-critical): ${receiptErr.message}`);
      }
      
      return {
        success: true,
        payment,
        verification
      };
      
    } catch (err) {
      console.error(`   ❌ Payment failed: ${err.message}`);
      
      // Return structured error response
      return {
        success: false,
        error: err.message,
        timestamp: new Date().toISOString()
      };
    }
  }
  
  /**
   * Method 3: Multiple payments with Promise.all() - fails fast
   */
  static async processBatchPaymentsFailFast(payments) {
    console.log(`\n${'─'.repeat(70)}`);
    console.log(`\n📊 Method 3: Batch Processing with Promise.all() (Fail Fast)\n`);
    console.log(`   Processing ${payments.length} payments...\n`);
    
    try {
      // Promise.all fails immediately if any promise rejects
      const results = await Promise.all(
        payments.map(({ cardNumber, amount }) =>
          PaymentGateway.processCreditCard(cardNumber, amount)
        )
      );
      
      console.log(`\n   ✅ All ${results.length} payments successful`);
      return { success: true, results };
      
    } catch (err) {
      // If any payment fails, all fail
      console.error(`\n   ❌ Batch failed: ${err.message}`);
      return { success: false, error: err.message };
    }
  }
  
  /**
   * Method 4: Multiple payments with Promise.allSettled() - process all
   */
  static async processBatchPaymentsAllSettled(payments) {
    console.log(`\n${'─'.repeat(70)}`);
    console.log(`\n📊 Method 4: Batch Processing with Promise.allSettled() (Continue on Error)\n`);
    console.log(`   Processing ${payments.length} payments...\n`);
    
    // Promise.allSettled waits for all promises (never rejects)
    const results = await Promise.allSettled(
      payments.map(({ cardNumber, amount }) =>
        PaymentGateway.processCreditCard(cardNumber, amount)
      )
    );
    
    // Analyze results
    const successful = results.filter(r => r.status === 'fulfilled');
    const failed = results.filter(r => r.status === 'rejected');
    
    console.log(`\n   Summary:`);
    console.log(`   ✅ Successful: ${successful.length}`);
    console.log(`   ❌ Failed: ${failed.length}`);
    
    if (failed.length > 0) {
      console.log(`\n   Failed payments:`);
      failed.forEach((result, index) => {
        console.log(`      ${index + 1}. ${result.reason.message}`);
      });
    }
    
    return {
      total: payments.length,
      successful: successful.map(r => r.value),
      failed: failed.map(r => ({
        error: r.reason.message
      }))
    };
  }
  
  /**
   * Method 5: Retry logic with exponential backoff
   */
  static async processPaymentWithRetry(cardNumber, amount, maxRetries = 3) {
    console.log(`\n${'─'.repeat(70)}`);
    console.log(`\n📊 Method 5: Payment with Retry Logic (Max ${maxRetries} retries)\n`);
    
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        console.log(`   Attempt ${attempt}/${maxRetries}...`);
        
        const payment = await PaymentGateway.processCreditCard(cardNumber, amount);
        
        console.log(`   ✅ Success on attempt ${attempt}`);
        return { success: true, payment, attempts: attempt };
        
      } catch (err) {
        console.error(`   ❌ Attempt ${attempt} failed: ${err.message}`);
        
        if (attempt === maxRetries) {
          console.error(`   ❌ All ${maxRetries} attempts failed`);
          return {
            success: false,
            error: err.message,
            attempts: attempt
          };
        }
        
        // Exponential backoff: 1s, 2s, 4s
        const delay = Math.pow(2, attempt - 1) * 1000;
        console.log(`   ⏳ Waiting ${delay}ms before retry...`);
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }
  
  /**
   * Method 6: Race condition with timeout
   */
  static async processPaymentWithTimeout(cardNumber, amount, timeoutMs = 2000) {
    console.log(`\n${'─'.repeat(70)}`);
    console.log(`\n📊 Method 6: Payment with Timeout (${timeoutMs}ms)\n`);
    
    const timeoutPromise = new Promise((_, reject) => {
      setTimeout(() => reject(new Error('Payment timeout')), timeoutMs);
    });
    
    try {
      // Race between payment and timeout
      const payment = await Promise.race([
        PaymentGateway.processCreditCard(cardNumber, amount),
        timeoutPromise
      ]);
      
      console.log(`   ✅ Payment completed within timeout`);
      return { success: true, payment };
      
    } catch (err) {
      console.error(`   ❌ ${err.message}`);
      return { success: false, error: err.message };
    }
  }
}

// ============================================
// Test Payment Service
// ============================================

async function runTests() {
  // Test 1: Successful payment with .then().catch()
  console.log('\n' + '='.repeat(70));
  console.log('\n🧪 Test 1: Successful Payment (.then/.catch)\n');
  
  try {
    await PaymentService.processPaymentWithThenCatch('4111111111111111', 100);
    console.log('\n✅ Test 1 Passed');
  } catch (err) {
    console.log(`\n❌ Test 1 Failed: ${err.message}`);
  }
  
  // Test 2: Failed payment with .then().catch()
  console.log('\n' + '='.repeat(70));
  console.log('\n🧪 Test 2: Failed Payment (.then/.catch)\n');
  
  try {
    await PaymentService.processPaymentWithThenCatch('4000000000000000', 100);
    console.log('\n❌ Test 2 Failed: Should have thrown');
  } catch (err) {
    console.log(`\n✅ Test 2 Passed: Caught error`);
  }
  
  // Test 3: Successful payment with async/await
  console.log('\n' + '='.repeat(70));
  console.log('\n🧪 Test 3: Successful Payment (async/await)\n');
  
  const result3 = await PaymentService.processPaymentWithAsyncAwait(
    '4111111111111111',
    200,
    'customer@example.com'
  );
  console.log(`\n${result3.success ? '✅' : '❌'} Test 3: ${result3.success ? 'Passed' : 'Failed'}`);
  
  // Test 4: Failed payment with async/await
  console.log('\n' + '='.repeat(70));
  console.log('\n🧪 Test 4: Failed Payment (async/await)\n');
  
  const result4 = await PaymentService.processPaymentWithAsyncAwait(
    '5000000000000000',
    200,
    'customer@example.com'
  );
  console.log(`\n${!result4.success ? '✅' : '❌'} Test 4: ${!result4.success ? 'Passed' : 'Failed'}`);
  
  // Test 5: Batch payments with Promise.all (fail fast)
  console.log('\n' + '='.repeat(70));
  console.log('\n🧪 Test 5: Batch Payments (Promise.all)\n');
  
  const batch1 = [
    { cardNumber: '4111111111111111', amount: 100 },
    { cardNumber: '4222222222222222', amount: 200 },
    { cardNumber: '4000000000000000', amount: 300 }  // This will fail
  ];
  
  const result5 = await PaymentService.processBatchPaymentsFailFast(batch1);
  console.log(`\n${!result5.success ? '✅' : '❌'} Test 5: ${!result5.success ? 'Passed (expected failure)' : 'Failed'}`);
  
  // Test 6: Batch payments with Promise.allSettled
  console.log('\n' + '='.repeat(70));
  console.log('\n🧪 Test 6: Batch Payments (Promise.allSettled)\n');
  
  const result6 = await PaymentService.processBatchPaymentsAllSettled(batch1);
  console.log(`\n✅ Test 6: Passed (${result6.successful.length}/${result6.total} succeeded)`);
  
  // Test 7: Payment with retry (timeout card - should retry)
  console.log('\n' + '='.repeat(70));
  console.log('\n🧪 Test 7: Payment with Retry\n');
  
  const result7 = await PaymentService.processPaymentWithRetry('6000000000000000', 100);
  console.log(`\n${!result7.success ? '✅' : '❌'} Test 7: ${!result7.success ? 'Passed (failed after retries)' : 'Failed'}`);
  
  // Test 8: Payment with timeout
  console.log('\n' + '='.repeat(70));
  console.log('\n🧪 Test 8: Payment with Timeout\n');
  
  const result8 = await PaymentService.processPaymentWithTimeout('4111111111111111', 100, 1000);
  console.log(`\n${result8.success ? '✅' : '❌'} Test 8: ${result8.success ? 'Passed' : 'Failed'}`);
  
  console.log('\n' + '='.repeat(70));
  console.log('\n✅ All payment tests completed!\n');
}

// Run tests
runTests().catch(err => {
  console.error('\n💥 Unhandled error:', err);
  process.exit(1);
});
```

**To test this example:**

```bash
node payment-processor.js
```

**Expected Output:**

```
💰 Payment Processing Service

======================================================================

🧪 Test 1: Successful Payment (.then/.catch)

──────────────────────────────────────────────────────────────────────

📊 Method 1: Using .then() and .catch()

💳 Processing credit card payment: $100
   ✅ Payment approved: CC_a3f7e9d2c8b4f6a1
   Payment successful: CC_a3f7e9d2c8b4f6a1

🔍 Verifying payment: CC_a3f7e9d2c8b4f6a1
   ✅ Payment verified
   Verification successful

✅ Test 1 Passed

======================================================================

🧪 Test 2: Failed Payment (.then/.catch)

──────────────────────────────────────────────────────────────────────

📊 Method 1: Using .then() and .catch()

💳 Processing credit card payment: $100
   ❌ Error in payment chain: Card declined - insufficient funds

✅ Test 2 Passed: Caught error

======================================================================

🧪 Test 3: Successful Payment (async/await)

──────────────────────────────────────────────────────────────────────

📊 Method 2: Using async/await with try-catch

💳 Processing credit card payment: $200
   ✅ Payment approved: CC_5d7c9b3f8a2e6d4c
   Payment successful: CC_5d7c9b3f8a2e6d4c

🔍 Verifying payment: CC_5d7c9b3f8a2e6d4c
   ✅ Payment verified
   Verification successful

📧 Sending receipt to: customer@example.com
   ✅ Receipt sent

✅ Test 3: Passed

======================================================================

🧪 Test 4: Failed Payment (async/await)

──────────────────────────────────────────────────────────────────────

📊 Method 2: Using async/await with try-catch

💳 Processing credit card payment: $200
   ❌ Payment failed: Card declined - invalid card

✅ Test 4: Passed

======================================================================

🧪 Test 5: Batch Payments (Promise.all)

──────────────────────────────────────────────────────────────────────

📊 Method 3: Batch Processing with Promise.all() (Fail Fast)

   Processing 3 payments...

💳 Processing credit card payment: $100
💳 Processing credit card payment: $200
💳 Processing credit card payment: $300
   ✅ Payment approved: CC_1a7f3e9d2c8b4f6a
   ✅ Payment approved: CC_2e5d7c9b3f8a2e6d

   ❌ Batch failed: Card declined - insufficient funds

✅ Test 5: Passed (expected failure)

======================================================================

🧪 Test 6: Batch Payments (Promise.allSettled)

──────────────────────────────────────────────────────────────────────

📊 Method 4: Batch Processing with Promise.allSettled() (Continue on Error)

   Processing 3 payments...

💳 Processing credit card payment: $100
💳 Processing credit card payment: $200
💳 Processing credit card payment: $300
   ✅ Payment approved: CC_3c1a7f3e9d2c8b4f
   ✅ Payment approved: CC_4f6a1e5d7c9b3f8a

   Summary:
   ✅ Successful: 2
   ❌ Failed: 1

   Failed payments:
      1. Card declined - insufficient funds

✅ Test 6: Passed (2/3 succeeded)

======================================================================

🧪 Test 7: Payment with Retry

──────────────────────────────────────────────────────────────────────

📊 Method 5: Payment with Retry Logic (Max 3 retries)

   Attempt 1/3...

💳 Processing credit card payment: $100
   ❌ Attempt 1 failed: Payment gateway timeout
   ⏳ Waiting 1000ms before retry...
   Attempt 2/3...

💳 Processing credit card payment: $100
   ❌ Attempt 2 failed: Payment gateway timeout
   ⏳ Waiting 2000ms before retry...
   Attempt 3/3...

💳 Processing credit card payment: $100
   ❌ Attempt 3 failed: Payment gateway timeout
   ❌ All 3 attempts failed

✅ Test 7: Passed (failed after retries)

======================================================================

🧪 Test 8: Payment with Timeout

──────────────────────────────────────────────────────────────────────

📊 Method 6: Payment with Timeout (1000ms)

💳 Processing credit card payment: $100
   ✅ Payment approved: CC_6d4c1a7f3e9d2c8b
   ✅ Payment completed within timeout

✅ Test 8: Passed

======================================================================

✅ All payment tests completed!
```

**Key Learnings from Example 2**:

1. **.then().catch()**: Chain promises, catch handles all errors in chain
2. **async/await**: Cleaner syntax, use try-catch for error handling
3. **Nested try-catch**: Handle non-critical errors separately
4. **Promise.all()**: Fails fast if any promise rejects
5. **Promise.allSettled()**: Waits for all, never rejects
6. **Retry Logic**: Exponential backoff for transient errors
7. **Promise.race()**: Implement timeouts

---

## 🏦 Example 3: Global Error Handling and Process-Level Errors

This example demonstrates handling uncaught errors and process-level error events.

**File: `error-monitoring.js`**

```javascript
import { EventEmitter } from 'events';

/**
 * Error Monitoring and Global Error Handlers
 * Demonstrates: unhandledRejection, uncaughtException, error logging
 */

console.log('🔍 Error Monitoring Service\n');
console.log('='.repeat(70));

// ============================================
// Error Logger
// ============================================

class ErrorLogger {
  constructor() {
    this.errors = [];
  }
  
  log(error, context = {}) {
    const errorLog = {
      timestamp: new Date().toISOString(),
      name: error.name,
      message: error.message,
      stack: error.stack,
      context,
      severity: this.determineSeverity(error)
    };
    
    this.errors.push(errorLog);
    
    // Log to console with color
    const emoji = this.getEmojiForSeverity(errorLog.severity);
    console.error(`\n${emoji} [${errorLog.severity}] ${error.name}: ${error.message}`);
    console.error(`   Context:`, JSON.stringify(context, null, 2));
    
    // In production, send to monitoring service (e.g., Sentry, DataDog)
    // this.sendToMonitoring(errorLog);
  }
  
  determineSeverity(error) {
    if (error.name === 'InsufficientFundsError') return 'WARNING';
    if (error.name === 'NetworkError') return 'ERROR';
    if (error.name === 'DatabaseError') return 'CRITICAL';
    return 'ERROR';
  }
  
  getEmojiForSeverity(severity) {
    const emojis = {
      'WARNING': '⚠️ ',
      'ERROR': '❌',
      'CRITICAL': '💥'
    };
    return emojis[severity] || '❌';
  }
  
  getErrors() {
    return this.errors;
  }
  
  getErrorsByType() {
    const byType = {};
    this.errors.forEach(err => {
      byType[err.name] = (byType[err.name] || 0) + 1;
    });
    return byType;
  }
}

const errorLogger = new ErrorLogger();

// ============================================
// Global Error Handlers
// ============================================

/**
 * Unhandled Promise Rejection
 * Catches promises that reject without .catch()
 */
process.on('unhandledRejection', (reason, promise) => {
  console.error('\n💥 UNHANDLED PROMISE REJECTION');
  console.error('   Promise:', promise);
  console.error('   Reason:', reason);
  
  errorLogger.log(
    reason instanceof Error ? reason : new Error(String(reason)),
    { type: 'unhandledRejection', promise: String(promise) }
  );
  
  // In production: log, alert, but don't crash
  // For now, we'll let it continue
});

/**
 * Uncaught Exception
 * Catches synchronous errors not in try-catch
 */
process.on('uncaughtException', (error, origin) => {
  console.error('\n💥 UNCAUGHT EXCEPTION');
  console.error('   Origin:', origin);
  console.error('   Error:', error);
  
  errorLogger.log(error, { type: 'uncaughtException', origin });
  
  // IMPORTANT: Process should exit after uncaughtException
  // The application is in an undefined state
  console.error('\n⚠️  Application is in undefined state, exiting...');
  process.exit(1);
});

/**
 * Warning events (e.g., deprecations)
 */
process.on('warning', (warning) => {
  console.warn('\n⚠️  PROCESS WARNING');
  console.warn('   Name:', warning.name);
  console.warn('   Message:', warning.message);
  console.warn('   Stack:', warning.stack);
});

// ============================================
// Error Scenarios
// ============================================

class BankingOperations {
  /**
   * Scenario 1: Promise rejection without .catch()
   */
  static async unhandledPromiseRejection() {
    console.log(`\n${'─'.repeat(70)}`);
    console.log('\n📊 Scenario 1: Unhandled Promise Rejection\n');
    
    // This promise rejects but has no .catch()
    Promise.reject(new Error('Payment gateway unavailable'));
    
    // Give time for handler to trigger
    await new Promise(resolve => setTimeout(resolve, 100));
    
    console.log('\n   ✅ Scenario 1 completed (error caught by global handler)');
  }
  
  /**
   * Scenario 2: Async function without try-catch
   */
  static async asyncFunctionWithoutTryCatch() {
    console.log(`\n${'─'.repeat(70)}`);
    console.log('\n📊 Scenario 2: Async Function Without try-catch\n');
    
    const operation = async () => {
      throw new Error('Database connection failed');
    };
    
    // Call async function but don't await or .catch()
    operation();  // This triggers unhandledRejection
    
    await new Promise(resolve => setTimeout(resolve, 100));
    
    console.log('\n   ✅ Scenario 2 completed (error caught by global handler)');
  }
  
  /**
   * Scenario 3: Properly handled errors
   */
  static async properlyHandledError() {
    console.log(`\n${'─'.repeat(70)}`);
    console.log('\n📊 Scenario 3: Properly Handled Error\n');
    
    try {
      await Promise.reject(new Error('Transaction validation failed'));
    } catch (err) {
      console.log(`   ✅ Error caught locally: ${err.message}`);
      errorLogger.log(err, { type: 'handledError', operation: 'validation' });
    }
    
    console.log('\n   ✅ Scenario 3 completed (error handled properly)');
  }
  
  /**
   * Scenario 4: Error in callback (not caught by try-catch)
   */
  static callbackError() {
    console.log(`\n${'─'.repeat(70)}`);
    console.log('\n📊 Scenario 4: Error in Callback\n');
    
    setTimeout(() => {
      try {
        throw new Error('Error in setTimeout callback');
      } catch (err) {
        // This catches it
        console.log(`   ✅ Error caught in callback: ${err.message}`);
        errorLogger.log(err, { type: 'callbackError' });
      }
    }, 100);
  }
  
  /**
   * Scenario 5: EventEmitter error without listener
   */
  static eventEmitterError() {
    console.log(`\n${'─'.repeat(70)}`);
    console.log('\n📊 Scenario 5: EventEmitter Error\n');
    
    const emitter = new EventEmitter();
    
    // Add error listener
    emitter.on('error', (err) => {
      console.log(`   ✅ Error event caught: ${err.message}`);
      errorLogger.log(err, { type: 'eventEmitterError' });
    });
    
    // Emit error
    emitter.emit('error', new Error('Transaction processing error'));
    
    console.log('\n   ✅ Scenario 5 completed');
  }
  
  /**
   * Scenario 6: Multiple async operations with one failure
   */
  static async multipleOperationsWithFailure() {
    console.log(`\n${'─'.repeat(70)}`);
    console.log('\n📊 Scenario 6: Multiple Operations (One Fails)\n');
    
    const operation1 = async () => {
      console.log('   Operation 1: Starting...');
      await new Promise(resolve => setTimeout(resolve, 100));
      console.log('   Operation 1: ✅ Success');
      return 'result1';
    };
    
    const operation2 = async () => {
      console.log('   Operation 2: Starting...');
      await new Promise(resolve => setTimeout(resolve, 150));
      throw new Error('Operation 2 failed');
    };
    
    const operation3 = async () => {
      console.log('   Operation 3: Starting...');
      await new Promise(resolve => setTimeout(resolve, 200));
      console.log('   Operation 3: ✅ Success');
      return 'result3';
    };
    
    try {
      const results = await Promise.all([
        operation1(),
        operation2(),
        operation3()
      ]);
      console.log('   All operations completed:', results);
    } catch (err) {
      console.log(`   ❌ One operation failed: ${err.message}`);
      errorLogger.log(err, { type: 'multiOpFailure' });
      
      // Retry with allSettled to get partial results
      console.log('\n   Retrying with Promise.allSettled()...');
      const results = await Promise.allSettled([
        operation1(),
        operation2(),
        operation3()
      ]);
      
      const successful = results.filter(r => r.status === 'fulfilled');
      console.log(`   Completed ${successful.length}/3 operations successfully`);
    }
    
    console.log('\n   ✅ Scenario 6 completed');
  }
}

// ============================================
// Test Error Scenarios
// ============================================

async function runTests() {
  console.log('\n🧪 Testing Error Scenarios\n');
  console.log('='.repeat(70));
  
  try {
    // Scenario 1: Unhandled promise rejection
    await BankingOperations.unhandledPromiseRejection();
    await new Promise(resolve => setTimeout(resolve, 200));
    
    // Scenario 2: Async without try-catch
    await BankingOperations.asyncFunctionWithoutTryCatch();
    await new Promise(resolve => setTimeout(resolve, 200));
    
    // Scenario 3: Properly handled
    await BankingOperations.properlyHandledError();
    await new Promise(resolve => setTimeout(resolve, 200));
    
    // Scenario 4: Callback error
    BankingOperations.callbackError();
    await new Promise(resolve => setTimeout(resolve, 200));
    
    // Scenario 5: EventEmitter error
    BankingOperations.eventEmitterError();
    await new Promise(resolve => setTimeout(resolve, 200));
    
    // Scenario 6: Multiple operations
    await BankingOperations.multipleOperationsWithFailure();
    await new Promise(resolve => setTimeout(resolve, 200));
    
    // Display error summary
    console.log('\n' + '='.repeat(70));
    console.log('\n📊 Error Summary\n');
    
    const errors = errorLogger.getErrors();
    console.log(`   Total errors logged: ${errors.length}`);
    
    const byType = errorLogger.getErrorsByType();
    console.log('\n   Errors by type:');
    Object.entries(byType).forEach(([type, count]) => {
      console.log(`      ${type}: ${count}`);
    });
    
    console.log('\n' + '='.repeat(70));
    console.log('\n✅ All error scenarios completed!\n');
    
  } catch (err) {
    console.error('\n💥 Test suite error:', err);
  }
}

// Run tests
runTests();
```

**To test this example:**

```bash
node error-monitoring.js
```

**Expected Output:**

```
🔍 Error Monitoring Service

======================================================================

🧪 Testing Error Scenarios

======================================================================

──────────────────────────────────────────────────────────────────────

📊 Scenario 1: Unhandled Promise Rejection

💥 UNHANDLED PROMISE REJECTION
   Promise: Promise { <rejected> [Error: Payment gateway unavailable] }
   Reason: Error: Payment gateway unavailable

❌ [ERROR] Error: Payment gateway unavailable
   Context: {
  "type": "unhandledRejection",
  "promise": "Promise { <rejected> [Error: Payment gateway unavailable] }"
}

   ✅ Scenario 1 completed (error caught by global handler)

──────────────────────────────────────────────────────────────────────

📊 Scenario 2: Async Function Without try-catch

💥 UNHANDLED PROMISE REJECTION
   Promise: Promise { <rejected> [Error: Database connection failed] }
   Reason: Error: Database connection failed

❌ [ERROR] Error: Database connection failed
   Context: {
  "type": "unhandledRejection",
  "promise": "Promise { <rejected> [Error: Database connection failed] }"
}

   ✅ Scenario 2 completed (error caught by global handler)

──────────────────────────────────────────────────────────────────────

📊 Scenario 3: Properly Handled Error

   ✅ Error caught locally: Transaction validation failed

❌ [ERROR] Error: Transaction validation failed
   Context: {
  "type": "handledError",
  "operation": "validation"
}

   ✅ Scenario 3 completed (error handled properly)

──────────────────────────────────────────────────────────────────────

📊 Scenario 4: Error in Callback

   ✅ Error caught in callback: Error in setTimeout callback

❌ [ERROR] Error: Error in setTimeout callback
   Context: {
  "type": "callbackError"
}

──────────────────────────────────────────────────────────────────────

📊 Scenario 5: EventEmitter Error

   ✅ Error event caught: Transaction processing error

❌ [ERROR] Error: Transaction processing error
   Context: {
  "type": "eventEmitterError"
}

   ✅ Scenario 5 completed

──────────────────────────────────────────────────────────────────────

📊 Scenario 6: Multiple Operations (One Fails)

   Operation 1: Starting...
   Operation 2: Starting...
   Operation 3: Starting...
   Operation 1: ✅ Success
   ❌ One operation failed: Operation 2 failed

❌ [ERROR] Error: Operation 2 failed
   Context: {
  "type": "multiOpFailure"
}

   Retrying with Promise.allSettled()...
   Operation 1: Starting...
   Operation 2: Starting...
   Operation 3: Starting...
   Operation 1: ✅ Success
   Operation 3: ✅ Success
   Completed 2/3 operations successfully

   ✅ Scenario 6 completed

======================================================================

📊 Error Summary

   Total errors logged: 6

   Errors by type:
      Error: 6

======================================================================

✅ All error scenarios completed!
```

**Key Learnings from Example 3**:

1. **unhandledRejection**: Catches promises without .catch()
2. **uncaughtException**: Catches sync errors not in try-catch (process should exit)
3. **Error Logger**: Centralized logging with severity levels
4. **EventEmitter Errors**: Always add 'error' listeners
5. **Process Warnings**: Monitor deprecations and issues
6. **Production Practice**: Log errors but keep service running (except uncaughtException)

---

## ✅ 10 DO's: Error Handling Best Practices

### 1. ✅ DO always check errors in error-first callbacks

```javascript
// ✅ GOOD: Check error first
fs.readFile('data.txt', 'utf8', (err, data) => {
  if (err) {
    console.error('Error:', err.message);
    return;  // Stop execution
  }
  console.log(data);
});

// ❌ BAD: No error check
fs.readFile('data.txt', 'utf8', (err, data) => {
  console.log(data);  // Crashes if err is not null
});
```

**Why**: Prevents crashes from unexpected errors.

---

### 2. ✅ DO create custom error classes

```javascript
// ✅ GOOD: Specific error types
class ValidationError extends Error {
  constructor(field, message) {
    super(`Validation failed for ${field}: ${message}`);
    this.name = 'ValidationError';
    this.field = field;
    Error.captureStackTrace(this, this.constructor);
  }
}

// ❌ BAD: Generic errors
throw new Error('something went wrong');
```

**Why**: Enables specific error handling and better debugging.

---

### 3. ✅ DO use try-catch with async/await

```javascript
// ✅ GOOD: Wrap await in try-catch
async function processPayment() {
  try {
    const result = await paymentGateway.charge();
    return result;
  } catch (err) {
    console.error('Payment failed:', err);
    throw err;
  }
}

// ❌ BAD: No try-catch with await
async function processPaymentBad() {
  const result = await paymentGateway.charge();  // Unhandled rejection
  return result;
}
```

**Why**: Unhandled rejections can crash your app in older Node versions.

---

### 4. ✅ DO propagate errors to caller

```javascript
// ✅ GOOD: Re-throw after logging
async function validateTransaction(txn) {
  try {
    // validation logic
  } catch (err) {
    logger.error('Validation failed:', err);
    throw err;  // Let caller handle
  }
}

// ❌ BAD: Swallow errors
async function validateTransactionBad(txn) {
  try {
    // validation logic
  } catch (err) {
    console.log('Error occurred');
    // Error lost, caller thinks everything is fine
  }
}
```

**Why**: Caller needs to know about failures.

---

### 5. ✅ DO add context to errors

```javascript
// ✅ GOOD: Add context
try {
  await db.updateBalance(accountId, amount);
} catch (err) {
  throw new Error(
    `Failed to update balance for account ${accountId}: ${err.message}`
  );
}

// ❌ BAD: No context
try {
  await db.updateBalance(accountId, amount);
} catch (err) {
  throw err;  // No idea which account failed
}
```

**Why**: Makes debugging much easier.

---

### 6. ✅ DO handle errors at appropriate levels

```javascript
// ✅ GOOD: Handle at right level
async function processOrder(orderId) {
  try {
    const order = await getOrder(orderId);
    const payment = await processPayment(order);
    await sendConfirmation(order.email);
    return payment;
  } catch (err) {
    // Handle all order processing errors here
    logger.error('Order processing failed:', err);
    await sendErrorNotification(orderId, err);
    throw err;
  }
}

// ❌ BAD: Handle too low
async function processOrderBad(orderId) {
  const order = await getOrder(orderId);  // Unhandled
  
  try {
    const payment = await processPayment(order);
  } catch (err) {
    // Only handles payment errors, not getOrder
  }
}
```

**Why**: Ensures all errors in a workflow are handled.

---

### 7. ✅ DO use Error.captureStackTrace()

```javascript
// ✅ GOOD: Maintain clean stack trace
class DatabaseError extends Error {
  constructor(message, query) {
    super(message);
    this.name = 'DatabaseError';
    this.query = query;
    Error.captureStackTrace(this, this.constructor);  // Clean stack
  }
}

// ❌ BAD: Polluted stack trace
class DatabaseErrorBad extends Error {
  constructor(message) {
    super(message);
    this.name = 'DatabaseError';
    // Stack includes constructor internals
  }
}
```

**Why**: Cleaner stack traces for debugging.

---

### 8. ✅ DO log errors with structured data

```javascript
// ✅ GOOD: Structured logging
logger.error('Transaction failed', {
  transactionId: txn.id,
  accountId: txn.accountId,
  amount: txn.amount,
  error: err.message,
  stack: err.stack,
  timestamp: new Date().toISOString()
});

// ❌ BAD: Unstructured
console.error('Error:', err);
```

**Why**: Enables better monitoring and debugging.

---

### 9. ✅ DO add error listeners to EventEmitters

```javascript
// ✅ GOOD: Always handle error events
const emitter = new EventEmitter();

emitter.on('error', (err) => {
  console.error('Emitter error:', err);
});

emitter.emit('error', new Error('Something failed'));

// ❌ BAD: No error listener
const emitterBad = new EventEmitter();
emitterBad.emit('error', new Error('Crash!'));  // Crashes app
```

**Why**: Unhandled 'error' events crash the process.

---

### 10. ✅ DO use Promise.allSettled() for independent operations

```javascript
// ✅ GOOD: Get results of all operations
const results = await Promise.allSettled([
  sendEmail(customer),
  sendSMS(customer),
  logActivity(customer)
]);

// Check which succeeded
const successful = results.filter(r => r.status === 'fulfilled');
console.log(`${successful.length}/3 operations succeeded`);

// ❌ BAD: One failure stops all
try {
  await Promise.all([
    sendEmail(customer),
    sendSMS(customer),  // If this fails...
    logActivity(customer)  // ...this never runs
  ]);
} catch (err) {
  // Don't know which operations completed
}
```

**Why**: Independent operations shouldn't block each other.

---

## ❌ 10 DON'Ts: Common Error Handling Mistakes

### 1. ❌ DON'T ignore errors

```javascript
// ❌ BAD: Ignoring callback error
fs.readFile('data.txt', 'utf8', (err, data) => {
  // No error check - app crashes on file not found
  console.log(data);
});

// ❌ BAD: Ignoring promise rejection
fetch('https://api.example.com/data')
  .then(res => res.json())
  .then(data => console.log(data));
  // No .catch() - unhandled rejection

// ✅ GOOD: Always handle
fs.readFile('data.txt', 'utf8', (err, data) => {
  if (err) return console.error('Error:', err.message);
  console.log(data);
});

fetch('https://api.example.com/data')
  .then(res => res.json())
  .then(data => console.log(data))
  .catch(err => console.error('Fetch failed:', err));
```

**Why**: Ignored errors cause crashes and bugs.

---

### 2. ❌ DON'T use try-catch with callbacks

```javascript
// ❌ BAD: try-catch doesn't catch async errors
try {
  fs.readFile('data.txt', 'utf8', (err, data) => {
    throw new Error('This won't be caught!');
  });
} catch (err) {
  // This never runs - callback is async
  console.error('Caught:', err);
}

// ✅ GOOD: Handle error in callback
fs.readFile('data.txt', 'utf8', (err, data) => {
  if (err) return console.error('Error:', err);
  // or use try-catch inside callback
  try {
    processData(data);
  } catch (err) {
    console.error('Processing error:', err);
  }
});
```

**Why**: try-catch only works with synchronous code.

---

### 3. ❌ DON'T swallow errors silently

```javascript
// ❌ BAD: Empty catch block
try {
  await processPayment(order);
} catch (err) {
  // Silent failure - caller doesn't know
}

// ❌ BAD: Log but don't propagate
try {
  await processPayment(order);
} catch (err) {
  console.log('Payment failed');
  // Caller thinks payment succeeded
}

// ✅ GOOD: Log and propagate
try {
  await processPayment(order);
} catch (err) {
  logger.error('Payment failed:', err);
  throw err;  // Let caller handle
}
```

**Why**: Swallowed errors make debugging impossible.

---

### 4. ❌ DON'T use strings for errors

```javascript
// ❌ BAD: String errors (no stack trace)
if (!user) {
  throw 'User not found';  // Just a string
}

// ❌ BAD: String in Promise.reject
return Promise.reject('Invalid input');

// ✅ GOOD: Always use Error objects
if (!user) {
  throw new Error('User not found');
}

return Promise.reject(new Error('Invalid input'));
```

**Why**: Strings don't have stack traces or error properties.

---

### 5. ❌ DON'T mix error handling styles

```javascript
// ❌ BAD: Mixing callbacks and promises
function processData(callback) {
  return fs.readFile('data.txt')  // Returns promise
    .then(data => {
      callback(null, data);  // Also uses callback
    })
    .catch(err => {
      callback(err);  // Confusing dual approach
    });
}

// ✅ GOOD: Consistent approach (choose one)
// Option 1: Callback-only
function processDataCallback(callback) {
  fs.readFile('data.txt', 'utf8', callback);
}

// Option 2: Promise-only
function processDataPromise() {
  return fs.promises.readFile('data.txt', 'utf8');
}
```

**Why**: Mixing styles causes confusion and bugs.

---

### 6. ❌ DON'T create errors in production without codes

```javascript
// ❌ BAD: No error code
throw new Error('Payment failed');  // How to handle?

// ✅ GOOD: Include error code
class PaymentError extends Error {
  constructor(message, code) {
    super(message);
    this.name = 'PaymentError';
    this.code = code;  // 'INSUFFICIENT_FUNDS', 'DECLINED', etc.
  }
}

throw new PaymentError('Payment failed', 'INSUFFICIENT_FUNDS');

// Now caller can switch on code
try {
  await processPayment();
} catch (err) {
  switch (err.code) {
    case 'INSUFFICIENT_FUNDS':
      return showAddFundsDialog();
    case 'DECLINED':
      return showTryAgainDialog();
    default:
      return showGenericError();
  }
}
```

**Why**: Error codes enable programmatic error handling.

---

### 7. ❌ DON'T throw errors in async callbacks without catching

```javascript
// ❌ BAD: Throwing in setTimeout
setTimeout(() => {
  throw new Error('Boom!');  // Crashes the app
}, 1000);

// ❌ BAD: Throwing in Promise constructor
new Promise((resolve, reject) => {
  throw new Error('Boom!');  // Should use reject()
});

// ✅ GOOD: Use reject or wrap in try-catch
setTimeout(() => {
  try {
    riskyOperation();
  } catch (err) {
    logger.error('Operation failed:', err);
  }
}, 1000);

new Promise((resolve, reject) => {
  try {
    riskyOperation();
    resolve();
  } catch (err) {
    reject(err);  // Proper rejection
  }
});
```

**Why**: Thrown errors in callbacks bypass error handlers.

---

### 8. ❌ DON'T forget to return after error callback

```javascript
// ❌ BAD: Continues after error
fs.readFile('data.txt', 'utf8', (err, data) => {
  if (err) {
    console.error('Error:', err);
    // Missing return - continues below!
  }
  console.log(data);  // data is undefined, causes another error
});

// ✅ GOOD: Return after error
fs.readFile('data.txt', 'utf8', (err, data) => {
  if (err) {
    console.error('Error:', err);
    return;  // Stop here
  }
  console.log(data);
});
```

**Why**: Code continues executing, causing more errors.

---

### 9. ❌ DON'T use process.exit() in libraries

```javascript
// ❌ BAD: Library exits process
export function connectDatabase(url) {
  try {
    return db.connect(url);
  } catch (err) {
    console.error('Database connection failed');
    process.exit(1);  // Kills the entire app!
  }
}

// ✅ GOOD: Let caller decide
export function connectDatabase(url) {
  try {
    return db.connect(url);
  } catch (err) {
    throw new DatabaseConnectionError('Failed to connect', err);
  }
}

// Caller can choose to exit
try {
  await connectDatabase(DB_URL);
} catch (err) {
  logger.fatal('Cannot start without database');
  process.exit(1);
}
```

**Why**: Libraries shouldn't control process lifecycle.

---

### 10. ❌ DON'T create multiple error handlers for same operation

```javascript
// ❌ BAD: Multiple handlers
processPayment()
  .catch(err => {
    logger.error('Payment error:', err);
  })
  .catch(err => {
    sendAlert(err);  // First catch handled it, this won't run
  })
  .catch(err => {
    updateUI(err);  // This won't run either
  });

// ✅ GOOD: Single comprehensive handler
processPayment()
  .catch(err => {
    logger.error('Payment error:', err);
    sendAlert(err);
    updateUI(err);
    throw err;  // Propagate if needed
  });

// OR: Handle at different levels
async function processOrder() {
  try {
    await processPayment();
  } catch (err) {
    // Handle payment-specific errors
    if (err.code === 'INSUFFICIENT_FUNDS') {
      throw new OrderError('Cannot complete order', err);
    }
    throw err;
  }
}

processOrder().catch(err => {
  // Handle order-level errors
  logger.error('Order failed:', err);
});
```

**Why**: .catch() handles the error, subsequent catches won't run.

---

## 🎓 Key Takeaways

### Error Handling Decision Tree

```
Is the operation synchronous?
├─ YES → Use try-catch
│    try {
│      const result = JSON.parse(data);
│    } catch (err) {
│      console.error(err);
│    }
│
└─ NO → Is it async?
     ├─ Callback-based? → Use error-first callback
     │    fs.readFile(path, (err, data) => {
     │      if (err) return handleError(err);
     │      processData(data);
     │    });
     │
     ├─ Promise-based? → Use .catch() or try-catch with await
     │    // Option 1
     │    promise.catch(err => handleError(err));
     │
     │    // Option 2
     │    try {
     │      await promise;
     │    } catch (err) {
     │      handleError(err);
     │    }
     │
     └─ EventEmitter? → Use 'error' event
          emitter.on('error', err => handleError(err));
```

### Error Handling Comparison

| Pattern | Use For | Syntax | Catches |
|---------|---------|--------|---------|
| **try-catch** | Synchronous code | `try { } catch (err) { }` | Sync errors only |
| **Error-first callback** | Callback APIs | `(err, result) => { if (err) ... }` | Async errors in callback |
| **.catch()** | Promises | `.catch(err => ...)` | Promise rejections |
| **try-catch + await** | Async/await | `try { await } catch (err) { }` | Promise rejections |
| **error event** | EventEmitters | `.on('error', err => ...)` | Emitted errors |

### Promise Error Handling Methods

| Method | Behavior | Use Case |
|--------|----------|----------|
| **Promise.all()** | Rejects on first error | All must succeed |
| **Promise.allSettled()** | Never rejects, returns all | Independent operations |
| **Promise.race()** | Resolves/rejects on first settled | Timeout patterns |
| **Promise.any()** | Rejects if all reject | Fallback patterns |

### Custom Error Class Template

```javascript
class CustomError extends Error {
  constructor(message, code, details = {}) {
    super(message);
    this.name = 'CustomError';
    this.code = code;
    this.details = details;
    this.timestamp = new Date().toISOString();
    Error.captureStackTrace(this, this.constructor);
  }
  
  toJSON() {
    return {
      name: this.name,
      message: this.message,
      code: this.code,
      details: this.details,
      timestamp: this.timestamp
    };
  }
}
```

### Global Error Handler Setup

```javascript
// Unhandled promise rejections
process.on('unhandledRejection', (reason, promise) => {
  logger.error('Unhandled Rejection:', reason);
  // In production: alert but don't crash
});

// Uncaught exceptions
process.on('uncaughtException', (error, origin) => {
  logger.fatal('Uncaught Exception:', error);
  // Must exit - app is in undefined state
  process.exit(1);
});

// Process warnings
process.on('warning', (warning) => {
  logger.warn('Process Warning:', warning);
});
```

### Common Interview Questions

**Q1: What is error-first callback pattern?**
- Convention where callback's first parameter is error (or null)
- Second parameter is result data
- Example: `fs.readFile(path, (err, data) => { if (err) ... })`

**Q2: Difference between throw and reject?**
- `throw`: For synchronous errors, caught by try-catch
- `reject()`: For async Promise errors, caught by .catch()
- Don't throw in Promise executor, use reject()

**Q3: Why use custom error classes?**
- Enable specific error handling with `instanceof`
- Add context (error codes, details)
- Better stack traces with captureStackTrace()
- Easier debugging and logging

**Q4: How to handle errors in async/await?**
```javascript
try {
  const result = await asyncOperation();
} catch (err) {
  // Handle error
}
```

**Q5: What happens with unhandled promise rejections?**
- Node.js emits 'unhandledRejection' event
- In older versions: Warning logged
- In newer versions (15+): Process terminates by default
- Always add .catch() or use try-catch with await

**Q6: How to handle errors in Promise.all()?**
- Promise.all() rejects immediately when any promise rejects
- Use Promise.allSettled() if you want all results
- Example:
```javascript
// Fail fast
const results = await Promise.all([p1, p2, p3]);

// Get all results
const results = await Promise.allSettled([p1, p2, p3]);
const successful = results.filter(r => r.status === 'fulfilled');
```

**Q7: Should libraries use process.exit()?**
- **NO** - Libraries should throw errors
- Let the application decide whether to exit
- Only top-level application code should call process.exit()

**Q8: How to prevent crashes from EventEmitter errors?**
- Always add 'error' event listener
- Without listener, error event crashes the process
```javascript
emitter.on('error', (err) => {
  logger.error('Emitter error:', err);
});
```

### Real-World Banking Error Hierarchy

```
BankingError (base)
├── ValidationError
│   ├── InvalidAmountError
│   ├── InvalidAccountError
│   └── InvalidTransactionTypeError
├── BusinessRuleError
│   ├── InsufficientFundsError
│   ├── TransactionLimitError
│   ├── AccountFrozenError
│   └── DailyLimitExceededError
├── TechnicalError
│   ├── DatabaseError
│   ├── NetworkError
│   ├── TimeoutError
│   └── ServiceUnavailableError
└── SecurityError
    ├── AuthenticationError
    ├── AuthorizationError
    └── FraudDetectedError
```

### Error Severity Levels

| Level | Description | Action |
|-------|-------------|--------|
| **TRACE** | Detailed debugging | Log only |
| **DEBUG** | Development info | Log only |
| **INFO** | Normal operation | Log only |
| **WARN** | Potential issue | Log + monitor |
| **ERROR** | Operation failed | Log + alert |
| **FATAL** | System failure | Log + alert + exit |

### Production Error Handling Checklist

- [ ] All promises have .catch() or try-catch
- [ ] Error-first callbacks check error parameter
- [ ] Custom error classes with codes
- [ ] Centralized error logging
- [ ] Structured error data (JSON)
- [ ] Global unhandledRejection handler
- [ ] Global uncaughtException handler
- [ ] EventEmitters have error listeners
- [ ] Errors include context (IDs, amounts, etc.)
- [ ] Error monitoring service integrated (Sentry, etc.)

---

## 🎯 Conclusion

Proper error handling is critical for banking applications:

1. **Error-first callbacks**: Always check error parameter first
2. **Custom errors**: Create meaningful error types with codes
3. **Try-catch**: For synchronous code
4. **.catch() / try-catch + await**: For async code
5. **Global handlers**: Catch unhandled rejections/exceptions
6. **Structured logging**: Include context for debugging
7. **Error propagation**: Don't swallow errors
8. **EventEmitter errors**: Always add listeners

**Banking Best Practices**:
- Validate all inputs with specific errors
- Track transaction state even during errors
- Log all errors with transaction context
- Alert on critical errors (fraud, data corruption)
- Graceful degradation (continue with partial success)
- Clear error messages for customers

Mastering error handling ensures reliable, debuggable banking systems!

---
Content includes:

Comprehensive Coverage: Error-first callbacks, synchronous vs async errors, try-catch, promises, event emitters
Example 1: Banking transaction service with custom error classes (InsufficientFundsError, AccountNotFoundError, TransactionLimitError, InvalidTransactionError, NetworkError) - demonstrates mixed error handling
Example 2: Payment processing with 6 different methods (.then/.catch, async/await, Promise.all, Promise.allSettled, retry logic with exponential backoff, timeout with Promise.race)
Example 3: Global error monitoring (unhandledRejection, uncaughtException, error logging, EventEmitter errors)
10 DO's: Check errors first, custom error classes, try-catch with await, propagate errors, add context, handle at right level, captureStackTrace, structured logging, EventEmitter listeners, Promise.allSettled
10 DON'Ts: Ignoring errors, try-catch with callbacks, swallowing errors, string errors, mixing styles, no error codes, throwing in async callbacks, missing returns, process.exit in libraries, multiple handlers
Key Takeaways: Decision tree, comparison tables, Promise methods, custom error template, global handlers, error hierarchy, severity levels, production checklist, interview Q&A
---

**Next Topics to Explore**:
- File System operations (fs module)
- HTTP Module and networking
- Express.js middleware and error handling
- Process management and signals

---

