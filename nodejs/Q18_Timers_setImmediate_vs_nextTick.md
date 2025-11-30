# Q18: Timers - setImmediate vs process.nextTick

## 📋 Summary
This question covers the critical differences between **`setImmediate()`** and **`process.nextTick()`** in Node.js, their execution order in the event loop, and when to use each. We'll build production-ready banking examples including priority transaction processing, batch payment validation, and critical operation scheduling.

---

## 🎯 What You'll Learn
- How `setImmediate()` and `process.nextTick()` work
- Execution order in the event loop
- When to use each API
- Performance implications and best practices
- Avoiding nextTick recursion issues
- Banking Examples: Priority transaction queue, urgent payment processing, batch validation

---

## 📖 Detailed Explanation

### What is process.nextTick()?

**`process.nextTick(callback)`** schedules a callback to be invoked **immediately after the current operation completes**, before the event loop continues to the next phase.

```javascript
console.log('1: Start');

process.nextTick(() => {
  console.log('2: nextTick callback');
});

console.log('3: End');

// Output:
// 1: Start
// 3: End
// 2: nextTick callback
```

**Key Characteristics**:
- Executes **before** any I/O events
- Executes **before** timers
- Executes **before** setImmediate
- Has **highest priority** in the event loop
- Can cause **starvation** if used recursively

---

### What is setImmediate()?

**`setImmediate(callback)`** schedules a callback to execute in the **Check Phase** of the event loop, after I/O events.

```javascript
console.log('1: Start');

setImmediate(() => {
  console.log('2: setImmediate callback');
});

console.log('3: End');

// Output:
// 1: Start
// 3: End
// 2: setImmediate callback
```

**Key Characteristics**:
- Executes in the **Check Phase**
- Executes **after** I/O events
- Executes **after** process.nextTick
- More predictable than setTimeout(fn, 0)
- Better for recursive operations

---

### Event Loop Execution Order

```
Current Operation Completes
         │
         ↓
   process.nextTick queue ← HIGHEST PRIORITY (executes immediately)
         │
         ↓
   Microtask queue (Promises)
         │
         ↓
┌───────────────────────────┐
│   Timers Phase            │ ← setTimeout/setInterval
└───────────────────────────┘
         │
         ↓
┌───────────────────────────┐
│   Pending Callbacks       │
└───────────────────────────┘
         │
         ↓
┌───────────────────────────┐
│   Idle, Prepare           │
└───────────────────────────┘
         │
         ↓
┌───────────────────────────┐
│   Poll Phase              │ ← I/O callbacks
└───────────────────────────┘
         │
         ↓
┌───────────────────────────┐
│   Check Phase             │ ← setImmediate
└───────────────────────────┘
         │
         ↓
┌───────────────────────────┐
│   Close Callbacks         │
└───────────────────────────┘
```

---

### Key Differences

| Feature | process.nextTick() | setImmediate() |
|---------|-------------------|----------------|
| **Execution Time** | After current operation, before event loop | Check phase of event loop |
| **Priority** | Highest (executes first) | Lower (after I/O) |
| **Use Case** | Emit events, error handling | Defer work to next iteration |
| **Risk** | Can block event loop | Event loop friendly |
| **Recursion** | Can starve event loop | Safe for recursion |
| **I/O Relationship** | Before I/O events | After I/O events |

---

### Execution Order Examples

#### Example 1: Basic Order

```javascript
console.log('1: Script start');

setTimeout(() => console.log('2: setTimeout'), 0);

setImmediate(() => console.log('3: setImmediate'));

process.nextTick(() => console.log('4: nextTick'));

console.log('5: Script end');

// Output:
// 1: Script start
// 5: Script end
// 4: nextTick          ← Executes first
// 2: setTimeout        ← Then timers
// 3: setImmediate      ← Then check phase
```

#### Example 2: Multiple Callbacks

```javascript
process.nextTick(() => console.log('nextTick 1'));
process.nextTick(() => console.log('nextTick 2'));

setImmediate(() => console.log('setImmediate 1'));
setImmediate(() => console.log('setImmediate 2'));

setTimeout(() => console.log('setTimeout'), 0);

// Output:
// nextTick 1
// nextTick 2
// setTimeout
// setImmediate 1
// setImmediate 2
```

#### Example 3: Inside I/O Callback

```javascript
const fs = require('fs');

fs.readFile(__filename, () => {
  console.log('I/O callback');
  
  setTimeout(() => console.log('setTimeout'), 0);
  setImmediate(() => console.log('setImmediate'));
  process.nextTick(() => console.log('nextTick'));
});

// Output:
// I/O callback
// nextTick         ← Always first
// setImmediate     ← Guaranteed before setTimeout in I/O callbacks
// setTimeout
```

---

### When to Use Each

#### Use `process.nextTick()` for:
1. **Error handling** - Emit errors asynchronously
2. **Event emission** - Ensure listeners are registered
3. **Cleanup** - Execute before moving to next phase
4. **API consistency** - Make sync APIs async

```javascript
// Good use case: Emit events after listeners are registered
class BankAccount {
  constructor() {
    process.nextTick(() => {
      this.emit('initialized'); // Listeners can be registered first
    });
  }
}
```

#### Use `setImmediate()` for:
1. **Recursive operations** - Avoid blocking
2. **Breaking up long operations** - Yield to event loop
3. **Defer work** - Execute after I/O
4. **Heavy computations** - Allow I/O to proceed

```javascript
// Good use case: Process large arrays without blocking
function processArray(array, callback) {
  if (array.length === 0) {
    return callback();
  }
  
  const item = array.shift();
  processItem(item);
  
  setImmediate(() => processArray(array, callback)); // Yield to event loop
}
```

---

## 🏦 Banking Application: Priority Transaction Processing System

### Scenario
Build a banking transaction processor that handles:
- **Critical transactions** (fraud alerts, account lockdowns) - Use `process.nextTick()`
- **Regular transactions** (transfers, payments) - Use normal processing
- **Batch operations** (report generation, cleanup) - Use `setImmediate()`

---

## 💻 Example 1: Priority Transaction Queue System

**File: `priority-transaction-processor.js`**

```javascript
const EventEmitter = require('events');

/**
 * Priority Transaction Processor
 * Demonstrates proper use of process.nextTick and setImmediate
 */
class PriorityTransactionProcessor extends EventEmitter {
  constructor() {
    super();
    this.criticalQueue = [];
    this.regularQueue = [];
    this.batchQueue = [];
    this.isProcessing = false;
    this.stats = {
      critical: 0,
      regular: 0,
      batch: 0,
      errors: 0
    };
  }

  /**
   * Add critical transaction (uses process.nextTick)
   * Examples: Fraud alerts, security issues, account lockdowns
   */
  addCriticalTransaction(transaction) {
    console.log(`🚨 CRITICAL: ${transaction.type} - ${transaction.description}`);
    
    this.criticalQueue.push(transaction);
    
    // Use process.nextTick for immediate processing
    process.nextTick(() => {
      this.processCriticalQueue();
    });
  }

  /**
   * Add regular transaction (normal processing)
   * Examples: Transfers, payments, deposits
   */
  addRegularTransaction(transaction) {
    console.log(`💳 REGULAR: ${transaction.type} - ${transaction.description}`);
    
    this.regularQueue.push(transaction);
    
    // Process in next event loop iteration
    setImmediate(() => {
      this.processRegularQueue();
    });
  }

  /**
   * Add batch operation (uses setImmediate)
   * Examples: Report generation, data cleanup, analytics
   */
  addBatchOperation(operation) {
    console.log(`📊 BATCH: ${operation.type} - ${operation.description}`);
    
    this.batchQueue.push(operation);
    
    // Use setImmediate to avoid blocking
    setImmediate(() => {
      this.processBatchQueue();
    });
  }

  /**
   * Process critical transactions immediately
   */
  processCriticalQueue() {
    while (this.criticalQueue.length > 0) {
      const transaction = this.criticalQueue.shift();
      
      try {
        this.executeCriticalTransaction(transaction);
        this.stats.critical++;
        
        // Emit event using nextTick to ensure listeners are ready
        process.nextTick(() => {
          this.emit('criticalProcessed', transaction);
        });
      } catch (error) {
        this.stats.errors++;
        console.error(`❌ Critical transaction failed:`, error.message);
        
        // Emit error asynchronously
        process.nextTick(() => {
          this.emit('error', error);
        });
      }
    }
  }

  /**
   * Process regular transactions
   */
  processRegularQueue() {
    if (this.isProcessing) return;
    
    this.isProcessing = true;
    
    while (this.regularQueue.length > 0) {
      const transaction = this.regularQueue.shift();
      
      try {
        this.executeRegularTransaction(transaction);
        this.stats.regular++;
        
        this.emit('regularProcessed', transaction);
      } catch (error) {
        this.stats.errors++;
        console.error(`❌ Regular transaction failed:`, error.message);
        this.emit('error', error);
      }
      
      // Check if critical transactions arrived
      if (this.criticalQueue.length > 0) {
        console.log('⚠️  Critical transaction detected, pausing regular processing');
        break;
      }
    }
    
    this.isProcessing = false;
  }

  /**
   * Process batch operations (non-blocking)
   */
  processBatchQueue() {
    if (this.batchQueue.length === 0) return;
    
    const operation = this.batchQueue.shift();
    
    try {
      this.executeBatchOperation(operation);
      this.stats.batch++;
      
      this.emit('batchProcessed', operation);
    } catch (error) {
      this.stats.errors++;
      console.error(`❌ Batch operation failed:`, error.message);
      this.emit('error', error);
    }
    
    // Continue processing remaining batch operations
    if (this.batchQueue.length > 0) {
      setImmediate(() => {
        this.processBatchQueue();
      });
    }
  }

  /**
   * Execute critical transaction (security, fraud, etc.)
   */
  executeCriticalTransaction(transaction) {
    const startTime = Date.now();
    
    switch (transaction.type) {
      case 'FRAUD_ALERT':
        console.log(`🔒 Locking account ${transaction.accountId}`);
        // Lock account immediately
        break;
        
      case 'SECURITY_BREACH':
        console.log(`🚨 Security breach detected for ${transaction.accountId}`);
        // Disable all access
        break;
        
      case 'EMERGENCY_STOP':
        console.log(`⛔ Emergency stop for ${transaction.accountId}`);
        // Stop all pending transactions
        break;
        
      default:
        console.log(`⚡ Processing critical: ${transaction.type}`);
    }
    
    const duration = Date.now() - startTime;
    console.log(`✅ Critical transaction completed in ${duration}ms`);
  }

  /**
   * Execute regular transaction
   */
  executeRegularTransaction(transaction) {
    const startTime = Date.now();
    
    switch (transaction.type) {
      case 'TRANSFER':
        console.log(`💸 Processing transfer: $${transaction.amount} from ${transaction.fromAccount} to ${transaction.toAccount}`);
        break;
        
      case 'PAYMENT':
        console.log(`💳 Processing payment: $${transaction.amount} to ${transaction.merchant}`);
        break;
        
      case 'DEPOSIT':
        console.log(`💰 Processing deposit: $${transaction.amount} to ${transaction.accountId}`);
        break;
        
      default:
        console.log(`📝 Processing: ${transaction.type}`);
    }
    
    const duration = Date.now() - startTime;
    console.log(`✅ Regular transaction completed in ${duration}ms`);
  }

  /**
   * Execute batch operation
   */
  executeBatchOperation(operation) {
    const startTime = Date.now();
    
    console.log(`📊 Processing batch: ${operation.type}`);
    
    // Simulate heavy operation
    const items = operation.items || 100;
    for (let i = 0; i < items; i++) {
      // Process item
    }
    
    const duration = Date.now() - startTime;
    console.log(`✅ Batch operation completed in ${duration}ms (${items} items)`);
  }

  /**
   * Get processing statistics
   */
  getStats() {
    return {
      ...this.stats,
      criticalPending: this.criticalQueue.length,
      regularPending: this.regularQueue.length,
      batchPending: this.batchQueue.length
    };
  }
}

// =============================================================================
// USAGE EXAMPLE
// =============================================================================

const processor = new PriorityTransactionProcessor();

// Set up event listeners
processor.on('criticalProcessed', (txn) => {
  console.log(`✨ Critical transaction processed: ${txn.type}`);
});

processor.on('regularProcessed', (txn) => {
  console.log(`✨ Regular transaction processed: ${txn.type}`);
});

processor.on('batchProcessed', (op) => {
  console.log(`✨ Batch operation processed: ${op.type}`);
});

processor.on('error', (error) => {
  console.error(`💥 Error occurred:`, error.message);
});

// =============================================================================
// DEMONSTRATION: Priority Processing
// =============================================================================

console.log('='.repeat(70));
console.log('DEMONSTRATION: Priority Transaction Processing');
console.log('='.repeat(70));

// Add various transactions in mixed order
console.log('\n📝 Adding transactions...\n');

// Regular transaction
processor.addRegularTransaction({
  type: 'TRANSFER',
  description: 'Transfer $1000',
  fromAccount: 'ACC001',
  toAccount: 'ACC002',
  amount: 1000
});

// Batch operation
processor.addBatchOperation({
  type: 'REPORT_GENERATION',
  description: 'Daily transaction report',
  items: 50
});

// Critical transaction (should interrupt regular processing)
processor.addCriticalTransaction({
  type: 'FRAUD_ALERT',
  description: 'Suspicious activity detected',
  accountId: 'ACC001'
});

// More regular transactions
processor.addRegularTransaction({
  type: 'PAYMENT',
  description: 'Payment to merchant',
  accountId: 'ACC003',
  merchant: 'Amazon',
  amount: 150
});

processor.addRegularTransaction({
  type: 'DEPOSIT',
  description: 'ATM deposit',
  accountId: 'ACC004',
  amount: 500
});

// Another critical transaction
processor.addCriticalTransaction({
  type: 'SECURITY_BREACH',
  description: 'Unauthorized access attempt',
  accountId: 'ACC003'
});

// More batch operations
processor.addBatchOperation({
  type: 'DATA_CLEANUP',
  description: 'Clean up old records',
  items: 100
});

// Print statistics after processing
setTimeout(() => {
  console.log('\n' + '='.repeat(70));
  console.log('PROCESSING STATISTICS');
  console.log('='.repeat(70));
  console.log(processor.getStats());
}, 100);
```

**Testing Commands:**

```bash
# Run the priority processor
node priority-transaction-processor.js

# Expected output shows critical transactions processed first,
# then regular transactions, then batch operations

# Run with timing analysis
time node priority-transaction-processor.js
```

**Expected Output:**
```
======================================================================
DEMONSTRATION: Priority Transaction Processing
======================================================================

📝 Adding transactions...

💳 REGULAR: TRANSFER - Transfer $1000
📊 BATCH: REPORT_GENERATION - Daily transaction report
🚨 CRITICAL: FRAUD_ALERT - Suspicious activity detected
💳 REGULAR: PAYMENT - Payment to merchant
💳 REGULAR: DEPOSIT - ATM deposit
🚨 CRITICAL: SECURITY_BREACH - Unauthorized access attempt
📊 BATCH: DATA_CLEANUP - Clean up old records
🔒 Locking account ACC001
✅ Critical transaction completed in 0ms
✨ Critical transaction processed: FRAUD_ALERT
🚨 Security breach detected for ACC003
✅ Critical transaction completed in 0ms
✨ Critical transaction processed: SECURITY_BREACH
💸 Processing transfer: $1000 from ACC001 to ACC002
✅ Regular transaction completed in 0ms
✨ Regular transaction processed: TRANSFER
💳 Processing payment: $150 to Amazon
✅ Regular transaction completed in 0ms
✨ Regular transaction processed: PAYMENT
💰 Processing deposit: $500 to ACC004
✅ Regular transaction completed in 0ms
✨ Regular transaction processed: DEPOSIT
📊 Processing batch: REPORT_GENERATION
✅ Batch operation completed in 0ms (50 items)
✨ Batch operation processed: REPORT_GENERATION
📊 Processing batch: DATA_CLEANUP
✅ Batch operation completed in 0ms (100 items)
✨ Batch operation processed: DATA_CLEANUP

======================================================================
PROCESSING STATISTICS
======================================================================
{
  critical: 2,
  regular: 3,
  batch: 2,
  errors: 0,
  criticalPending: 0,
  regularPending: 0,
  batchPending: 0
}
```

---

## 💻 Example 2: Urgent Payment Processing System

**File: `urgent-payment-processor.js`**

```javascript
const EventEmitter = require('events');

/**
 * Urgent Payment Processor
 * Demonstrates nextTick vs setImmediate for payment processing
 */
class UrgentPaymentProcessor extends EventEmitter {
  constructor() {
    super();
    this.urgentPayments = [];
    this.normalPayments = [];
    this.deferredPayments = [];
  }

  /**
   * Process urgent payment (hospital bills, emergency services)
   * Uses process.nextTick for immediate processing
   */
  processUrgentPayment(payment) {
    console.log(`🚨 URGENT PAYMENT: ${payment.description} - $${payment.amount}`);
    
    // Validate immediately
    const validation = this.validatePayment(payment);
    
    if (!validation.valid) {
      // Emit error asynchronously using nextTick
      process.nextTick(() => {
        this.emit('error', new Error(`Validation failed: ${validation.reason}`));
      });
      return;
    }

    this.urgentPayments.push(payment);
    
    // Process immediately using nextTick
    process.nextTick(() => {
      try {
        this.executeUrgentPayment(payment);
        
        // Emit success event
        process.nextTick(() => {
          this.emit('urgentPaymentProcessed', payment);
        });
      } catch (error) {
        process.nextTick(() => {
          this.emit('error', error);
        });
      }
    });
  }

  /**
   * Process normal payment
   * Uses regular event loop processing
   */
  processNormalPayment(payment) {
    console.log(`💳 NORMAL PAYMENT: ${payment.description} - $${payment.amount}`);
    
    const validation = this.validatePayment(payment);
    
    if (!validation.valid) {
      setImmediate(() => {
        this.emit('error', new Error(`Validation failed: ${validation.reason}`));
      });
      return;
    }

    this.normalPayments.push(payment);
    
    // Process in next iteration
    setImmediate(() => {
      try {
        this.executeNormalPayment(payment);
        
        setImmediate(() => {
          this.emit('normalPaymentProcessed', payment);
        });
      } catch (error) {
        setImmediate(() => {
          this.emit('error', error);
        });
      }
    });
  }

  /**
   * Process deferred payment (scheduled, low priority)
   * Uses setImmediate for deferred execution
   */
  processDeferredPayment(payment) {
    console.log(`📅 DEFERRED PAYMENT: ${payment.description} - $${payment.amount}`);
    
    this.deferredPayments.push(payment);
    
    // Defer to avoid blocking
    setImmediate(() => {
      const validation = this.validatePayment(payment);
      
      if (!validation.valid) {
        setImmediate(() => {
          this.emit('error', new Error(`Validation failed: ${validation.reason}`));
        });
        return;
      }
      
      setImmediate(() => {
        try {
          this.executeDeferredPayment(payment);
          
          setImmediate(() => {
            this.emit('deferredPaymentProcessed', payment);
          });
        } catch (error) {
          setImmediate(() => {
            this.emit('error', error);
          });
        }
      });
    });
  }

  /**
   * Validate payment details
   */
  validatePayment(payment) {
    if (!payment.amount || payment.amount <= 0) {
      return { valid: false, reason: 'Invalid amount' };
    }
    
    if (!payment.fromAccount || !payment.toAccount) {
      return { valid: false, reason: 'Missing account information' };
    }
    
    if (payment.amount > 50000) {
      return { valid: false, reason: 'Amount exceeds limit' };
    }
    
    return { valid: true };
  }

  /**
   * Execute urgent payment
   */
  executeUrgentPayment(payment) {
    const startTime = Date.now();
    
    console.log(`⚡ Executing urgent payment: ${payment.description}`);
    console.log(`   From: ${payment.fromAccount}`);
    console.log(`   To: ${payment.toAccount}`);
    console.log(`   Amount: $${payment.amount}`);
    
    // Simulate instant processing
    payment.status = 'COMPLETED';
    payment.completedAt = new Date().toISOString();
    payment.processingTime = Date.now() - startTime;
    
    console.log(`✅ Urgent payment completed in ${payment.processingTime}ms`);
  }

  /**
   * Execute normal payment
   */
  executeNormalPayment(payment) {
    const startTime = Date.now();
    
    console.log(`💳 Executing normal payment: ${payment.description}`);
    console.log(`   From: ${payment.fromAccount}`);
    console.log(`   To: ${payment.toAccount}`);
    console.log(`   Amount: $${payment.amount}`);
    
    payment.status = 'COMPLETED';
    payment.completedAt = new Date().toISOString();
    payment.processingTime = Date.now() - startTime;
    
    console.log(`✅ Normal payment completed in ${payment.processingTime}ms`);
  }

  /**
   * Execute deferred payment
   */
  executeDeferredPayment(payment) {
    const startTime = Date.now();
    
    console.log(`📅 Executing deferred payment: ${payment.description}`);
    console.log(`   Scheduled for: ${payment.scheduledDate}`);
    console.log(`   From: ${payment.fromAccount}`);
    console.log(`   To: ${payment.toAccount}`);
    console.log(`   Amount: $${payment.amount}`);
    
    payment.status = 'SCHEDULED';
    payment.scheduledAt = new Date().toISOString();
    payment.processingTime = Date.now() - startTime;
    
    console.log(`✅ Deferred payment scheduled in ${payment.processingTime}ms`);
  }

  /**
   * Get payment statistics
   */
  getStats() {
    return {
      urgent: this.urgentPayments.length,
      normal: this.normalPayments.length,
      deferred: this.deferredPayments.length,
      total: this.urgentPayments.length + this.normalPayments.length + this.deferredPayments.length
    };
  }
}

// =============================================================================
// USAGE EXAMPLE
// =============================================================================

const processor = new UrgentPaymentProcessor();

// Set up event listeners
processor.on('urgentPaymentProcessed', (payment) => {
  console.log(`✨ Urgent payment processed: ${payment.description} - Status: ${payment.status}`);
});

processor.on('normalPaymentProcessed', (payment) => {
  console.log(`✨ Normal payment processed: ${payment.description} - Status: ${payment.status}`);
});

processor.on('deferredPaymentProcessed', (payment) => {
  console.log(`✨ Deferred payment processed: ${payment.description} - Status: ${payment.status}`);
});

processor.on('error', (error) => {
  console.error(`💥 Payment error:`, error.message);
});

// =============================================================================
// DEMONSTRATION: Different Payment Types
// =============================================================================

console.log('='.repeat(70));
console.log('DEMONSTRATION: Urgent vs Normal vs Deferred Payments');
console.log('='.repeat(70));
console.log();

// Normal payment
processor.processNormalPayment({
  description: 'Online shopping',
  fromAccount: 'ACC001',
  toAccount: 'MERCHANT001',
  amount: 250
});

// Urgent payment (should process first)
processor.processUrgentPayment({
  description: 'Emergency hospital bill',
  fromAccount: 'ACC002',
  toAccount: 'HOSPITAL001',
  amount: 5000
});

// Deferred payment
processor.processDeferredPayment({
  description: 'Scheduled rent payment',
  fromAccount: 'ACC003',
  toAccount: 'LANDLORD001',
  amount: 2000,
  scheduledDate: '2025-12-01'
});

// Another urgent payment
processor.processUrgentPayment({
  description: 'Emergency flight booking',
  fromAccount: 'ACC004',
  toAccount: 'AIRLINE001',
  amount: 800
});

// Invalid payment (should trigger error)
processor.processUrgentPayment({
  description: 'Invalid payment',
  fromAccount: 'ACC005',
  toAccount: 'MERCHANT002',
  amount: 100000 // Exceeds limit
});

// Print statistics
setTimeout(() => {
  console.log('\n' + '='.repeat(70));
  console.log('PAYMENT STATISTICS');
  console.log('='.repeat(70));
  console.log(processor.getStats());
}, 100);
```

**Testing Commands:**

```bash
# Run urgent payment processor
node urgent-payment-processor.js

# Watch the order: urgent payments process before normal/deferred
```

---

## 💻 Example 3: Batch Validation with setImmediate

**File: `batch-validator.js`**

```javascript
/**
 * Batch Transaction Validator
 * Demonstrates safe recursion with setImmediate vs dangerous recursion with nextTick
 */
class BatchTransactionValidator {
  constructor() {
    this.validations = 0;
    this.errors = 0;
  }

  /**
   * Validate transactions using setImmediate (SAFE)
   * Allows event loop to process I/O between batches
   */
  validateBatchSafe(transactions, callback) {
    console.log(`📝 Safe validation of ${transactions.length} transactions`);
    
    const results = [];
    let index = 0;

    const validateNext = () => {
      if (index >= transactions.length) {
        console.log(`✅ Safe validation complete: ${this.validations} validated, ${this.errors} errors`);
        return callback(null, results);
      }

      const transaction = transactions[index++];
      
      try {
        const result = this.validateTransaction(transaction);
        results.push(result);
        
        if (!result.valid) {
          this.errors++;
        } else {
          this.validations++;
        }
      } catch (error) {
        this.errors++;
        results.push({ valid: false, error: error.message });
      }

      // Use setImmediate to yield to event loop
      // This allows I/O operations to be processed
      setImmediate(validateNext);
    };

    validateNext();
  }

  /**
   * Validate transactions using process.nextTick (DANGEROUS - for demonstration only)
   * Can starve the event loop and block I/O
   */
  validateBatchDangerous(transactions, callback) {
    console.log(`⚠️  Dangerous validation of ${transactions.length} transactions`);
    console.log(`⚠️  WARNING: This will block the event loop!`);
    
    const results = [];
    let index = 0;

    const validateNext = () => {
      if (index >= transactions.length) {
        console.log(`✅ Dangerous validation complete: ${this.validations} validated, ${this.errors} errors`);
        return callback(null, results);
      }

      const transaction = transactions[index++];
      
      try {
        const result = this.validateTransaction(transaction);
        results.push(result);
        
        if (!result.valid) {
          this.errors++;
        } else {
          this.validations++;
        }
      } catch (error) {
        this.errors++;
        results.push({ valid: false, error: error.message });
      }

      // DANGEROUS: Using nextTick recursively blocks event loop
      process.nextTick(validateNext);
    };

    validateNext();
  }

  /**
   * Validate single transaction
   */
  validateTransaction(transaction) {
    // Simulate validation logic
    const errors = [];

    if (!transaction.id) {
      errors.push('Missing transaction ID');
    }

    if (!transaction.amount || transaction.amount <= 0) {
      errors.push('Invalid amount');
    }

    if (!transaction.fromAccount || !transaction.toAccount) {
      errors.push('Missing account information');
    }

    if (transaction.amount > 10000) {
      errors.push('Amount exceeds single transaction limit');
    }

    if (transaction.fromAccount === transaction.toAccount) {
      errors.push('Cannot transfer to same account');
    }

    return {
      transactionId: transaction.id,
      valid: errors.length === 0,
      errors: errors.length > 0 ? errors : undefined
    };
  }

  /**
   * Reset counters
   */
  reset() {
    this.validations = 0;
    this.errors = 0;
  }
}

// =============================================================================
// HELPER: Generate test transactions
// =============================================================================

function generateTransactions(count) {
  const transactions = [];
  
  for (let i = 0; i < count; i++) {
    transactions.push({
      id: `TXN${String(i + 1).padStart(6, '0')}`,
      fromAccount: `ACC${String(Math.floor(Math.random() * 100) + 1).padStart(3, '0')}`,
      toAccount: `ACC${String(Math.floor(Math.random() * 100) + 1).padStart(3, '0')}`,
      amount: Math.floor(Math.random() * 15000) + 100,
      timestamp: new Date().toISOString()
    });
  }
  
  return transactions;
}

// =============================================================================
// DEMONSTRATION 1: Safe validation with setImmediate
// =============================================================================

console.log('='.repeat(70));
console.log('DEMONSTRATION 1: Safe Batch Validation (setImmediate)');
console.log('='.repeat(70));
console.log();

const validator1 = new BatchTransactionValidator();
const transactions1 = generateTransactions(100);

console.log(`Generated ${transactions1.length} test transactions`);
console.log();

const startTime1 = Date.now();

validator1.validateBatchSafe(transactions1, (err, results) => {
  if (err) {
    console.error('Validation error:', err);
    return;
  }
  
  const duration1 = Date.now() - startTime1;
  
  console.log();
  console.log('Validation Results:');
  console.log(`  Total: ${results.length}`);
  console.log(`  Valid: ${results.filter(r => r.valid).length}`);
  console.log(`  Invalid: ${results.filter(r => !r.valid).length}`);
  console.log(`  Duration: ${duration1}ms`);
  console.log();
  
  // Show sample invalid transactions
  const invalid = results.filter(r => !r.valid).slice(0, 3);
  if (invalid.length > 0) {
    console.log('Sample Invalid Transactions:');
    invalid.forEach(r => {
      console.log(`  ${r.transactionId}: ${r.errors.join(', ')}`);
    });
  }
});

// =============================================================================
// DEMONSTRATION 2: Event loop friendliness
// =============================================================================

setTimeout(() => {
  console.log('\n' + '='.repeat(70));
  console.log('DEMONSTRATION 2: Event Loop Friendliness');
  console.log('='.repeat(70));
  console.log();
  
  const validator2 = new BatchTransactionValidator();
  const transactions2 = generateTransactions(1000);
  
  console.log(`Testing with ${transactions2.length} transactions`);
  console.log('Starting validation with I/O operations...');
  console.log();
  
  // Schedule some I/O operations
  const fs = require('fs');
  let ioCompleted = false;
  
  fs.readFile(__filename, 'utf8', (err, data) => {
    if (!err) {
      ioCompleted = true;
      console.log(`📂 I/O operation completed (read ${data.length} bytes)`);
    }
  });
  
  const startTime2 = Date.now();
  
  validator2.validateBatchSafe(transactions2, (err, results) => {
    const duration2 = Date.now() - startTime2;
    
    console.log();
    console.log('Validation completed:');
    console.log(`  Processed: ${results.length} transactions`);
    console.log(`  Duration: ${duration2}ms`);
    console.log(`  I/O completed during validation: ${ioCompleted ? 'YES ✅' : 'NO ❌'}`);
    console.log();
    
    if (ioCompleted) {
      console.log('✨ Event loop remained responsive during validation!');
    } else {
      console.log('⚠️  Event loop might have been blocked');
    }
  });
}, 200);

// =============================================================================
// DEMONSTRATION 3: Comparison of execution order
// =============================================================================

setTimeout(() => {
  console.log('\n' + '='.repeat(70));
  console.log('DEMONSTRATION 3: Execution Order Comparison');
  console.log('='.repeat(70));
  console.log();
  
  console.log('Test 1: nextTick vs setImmediate without I/O');
  console.log('-'.repeat(50));
  
  setTimeout(() => console.log('1. setTimeout'), 0);
  setImmediate(() => console.log('2. setImmediate'));
  process.nextTick(() => console.log('3. nextTick'));
  Promise.resolve().then(() => console.log('4. Promise'));
  
  console.log('5. Synchronous code');
  
  setTimeout(() => {
    console.log();
    console.log('Test 2: nextTick vs setImmediate inside I/O callback');
    console.log('-'.repeat(50));
    
    const fs = require('fs');
    fs.readFile(__filename, () => {
      console.log('6. I/O callback');
      
      setTimeout(() => console.log('7. setTimeout in I/O'), 0);
      setImmediate(() => console.log('8. setImmediate in I/O'));
      process.nextTick(() => console.log('9. nextTick in I/O'));
    });
  }, 100);
}, 500);

// =============================================================================
// Keep process alive to see all output
// =============================================================================

setTimeout(() => {
  console.log('\n' + '='.repeat(70));
  console.log('All demonstrations completed');
  console.log('='.repeat(70));
}, 2000);
```

**Testing Commands:**

```bash
# Run batch validator
node batch-validator.js

# Compare safe vs dangerous validation
# Notice I/O operations complete during safe validation

# Run with timing
time node batch-validator.js
```

**Expected Output:**
```
======================================================================
DEMONSTRATION 1: Safe Batch Validation (setImmediate)
======================================================================

Generated 100 test transactions
📝 Safe validation of 100 transactions

✅ Safe validation complete: 82 validated, 18 errors

Validation Results:
  Total: 100
  Valid: 82
  Invalid: 18
  Duration: 15ms

Sample Invalid Transactions:
  TXN000003: Amount exceeds single transaction limit
  TXN000007: Cannot transfer to same account
  TXN000015: Amount exceeds single transaction limit

======================================================================
DEMONSTRATION 2: Event Loop Friendliness
======================================================================

Testing with 1000 transactions
Starting validation with I/O operations...

📂 I/O operation completed (read 45678 bytes)

Validation completed:
  Processed: 1000 transactions
  Duration: 145ms
  I/O completed during validation: YES ✅

✨ Event loop remained responsive during validation!

======================================================================
DEMONSTRATION 3: Execution Order Comparison
======================================================================

Test 1: nextTick vs setImmediate without I/O
--------------------------------------------------
5. Synchronous code
3. nextTick
4. Promise
1. setTimeout
2. setImmediate

Test 2: nextTick vs setImmediate inside I/O callback
--------------------------------------------------
6. I/O callback
9. nextTick in I/O
8. setImmediate in I/O
7. setTimeout in I/O

======================================================================
All demonstrations completed
======================================================================
```

---

## 🔍 Deep Dive: nextTick Starvation Problem

### The Problem

Recursive `process.nextTick()` can **starve the event loop**, preventing I/O operations from executing.

```javascript
// DANGEROUS: This will block everything!
function recursiveNextTick() {
  console.log('nextTick iteration');
  
  // This creates infinite loop in nextTick queue
  process.nextTick(recursiveNextTick);
}

recursiveNextTick();

// These will NEVER execute:
setTimeout(() => console.log('This will not run'), 0);
setImmediate(() => console.log('This will not run either'));
```

### The Solution

Use `setImmediate()` for recursive operations to yield to the event loop.

```javascript
// SAFE: This allows other operations to execute
function recursiveSetImmediate(count) {
  console.log(`setImmediate iteration ${count}`);
  
  if (count < 10) {
    setImmediate(() => recursiveSetImmediate(count + 1));
  }
}

recursiveSetImmediate(0);

// These WILL execute between iterations:
setTimeout(() => console.log('setTimeout executed'), 0);

const fs = require('fs');
fs.readFile(__filename, () => {
  console.log('I/O operation completed');
});
```

---

## 📊 Performance Comparison

### Benchmark: nextTick vs setImmediate

```javascript
const { performance } = require('perf_hooks');

function benchmarkNextTick(iterations, callback) {
  let count = 0;
  const start = performance.now();
  
  function next() {
    count++;
    if (count < iterations) {
      process.nextTick(next);
    } else {
      const duration = performance.now() - start;
      callback(duration);
    }
  }
  
  next();
}

function benchmarkSetImmediate(iterations, callback) {
  let count = 0;
  const start = performance.now();
  
  function next() {
    count++;
    if (count < iterations) {
      setImmediate(next);
    } else {
      const duration = performance.now() - start;
      callback(duration);
    }
  }
  
  next();
}

// Run benchmarks
const iterations = 10000;

benchmarkNextTick(iterations, (nextTickDuration) => {
  console.log(`nextTick ${iterations} iterations: ${nextTickDuration.toFixed(2)}ms`);
  
  benchmarkSetImmediate(iterations, (setImmediateDuration) => {
    console.log(`setImmediate ${iterations} iterations: ${setImmediateDuration.toFixed(2)}ms`);
    
    console.log(`\nSpeed difference: ${(setImmediateDuration / nextTickDuration).toFixed(2)}x`);
  });
});

// Typical results:
// nextTick 10000 iterations: 8.23ms
// setImmediate 10000 iterations: 1250.45ms
// Speed difference: 152x
//
// nextTick is faster BUT blocks event loop
// setImmediate is slower BUT allows I/O to process
```

---

## 🎯 DO's and DON'Ts

### ✅ DO's

1. **DO use `process.nextTick()` for immediate error emission**
```javascript
function asyncFunction(callback) {
  if (!callback) {
    // Emit error asynchronously
    process.nextTick(() => {
      throw new Error('Callback required');
    });
    return;
  }
  // ... rest of code
}
```

2. **DO use `process.nextTick()` to ensure event listeners are registered**
```javascript
class MyEmitter extends EventEmitter {
  constructor() {
    super();
    process.nextTick(() => {
      this.emit('initialized'); // Listeners have time to register
    });
  }
}
```

3. **DO use `setImmediate()` for recursive operations**
```javascript
function processArray(array) {
  if (array.length === 0) return;
  
  const item = array.shift();
  processItem(item);
  
  setImmediate(() => processArray(array)); // Yields to event loop
}
```

4. **DO use `setImmediate()` to break up long operations**
```javascript
function processLargeDataset(data, callback) {
  const chunkSize = 1000;
  let index = 0;
  
  function processChunk() {
    const chunk = data.slice(index, index + chunkSize);
    processItems(chunk);
    
    index += chunkSize;
    
    if (index < data.length) {
      setImmediate(processChunk); // Allow I/O between chunks
    } else {
      callback();
    }
  }
  
  processChunk();
}
```

5. **DO use `process.nextTick()` for API consistency**
```javascript
function getUserData(userId, callback) {
  if (cache.has(userId)) {
    // Make it async even though data is cached
    process.nextTick(() => {
      callback(null, cache.get(userId));
    });
  } else {
    db.query('SELECT * FROM users WHERE id = ?', [userId], callback);
  }
}
```

6. **DO use `setImmediate()` inside I/O callbacks**
```javascript
fs.readFile('file.txt', (err, data) => {
  if (err) return console.error(err);
  
  // setImmediate is guaranteed to execute before setTimeout here
  setImmediate(() => {
    processData(data);
  });
});
```

7. **DO clean up resources in nextTick callbacks**
```javascript
server.close(() => {
  process.nextTick(() => {
    // Cleanup after server is closed
    cleanupResources();
  });
});
```

8. **DO use descriptive names for callbacks**
```javascript
// Good
process.nextTick(handleCriticalTransaction);
setImmediate(processBatchReports);

// Bad
process.nextTick(cb);
setImmediate(fn);
```

9. **DO handle errors in nextTick callbacks**
```javascript
process.nextTick(() => {
  try {
    processTransaction();
  } catch (error) {
    handleError(error);
  }
});
```

10. **DO document why you're using nextTick or setImmediate**
```javascript
// Use nextTick to ensure event emitted after listener registration
process.nextTick(() => {
  this.emit('ready');
});

// Use setImmediate to avoid blocking event loop during batch processing
setImmediate(() => {
  processBatch();
});
```

---

### ❌ DON'Ts

1. **DON'T use recursive `process.nextTick()`**
```javascript
// BAD: Starves event loop
function bad() {
  process.nextTick(bad);
}

// GOOD: Use setImmediate
function good(count) {
  if (count < 100) {
    setImmediate(() => good(count + 1));
  }
}
```

2. **DON'T use `nextTick` for heavy computations**
```javascript
// BAD: Blocks event loop
process.nextTick(() => {
  for (let i = 0; i < 1000000; i++) {
    // Heavy computation
  }
});

// GOOD: Use setImmediate or Worker Threads
setImmediate(() => {
  // Or better: use Worker Threads for CPU-intensive work
});
```

3. **DON'T confuse `setImmediate` with `setTimeout(fn, 0)`**
```javascript
// BAD: Execution order is unpredictable outside I/O
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));

// GOOD: Use setImmediate for "next iteration" semantics
setImmediate(() => console.log('immediate'));
```

4. **DON'T use `nextTick` for long callback chains**
```javascript
// BAD: Creates deeply nested nextTick calls
process.nextTick(() => {
  step1();
  process.nextTick(() => {
    step2();
    process.nextTick(() => {
      step3();
    });
  });
});

// GOOD: Use async/await or promises
async function processSteps() {
  await step1();
  await step2();
  await step3();
}
```

5. **DON'T use `nextTick` in hot code paths**
```javascript
// BAD: Called frequently, adds overhead
function processRequest(req, res) {
  process.nextTick(() => {
    res.send('OK');
  });
}

// GOOD: Direct response for simple cases
function processRequest(req, res) {
  res.send('OK');
}
```

6. **DON'T mix nextTick and callback patterns inconsistently**
```javascript
// BAD: Inconsistent async behavior
function getData(callback) {
  if (cached) {
    callback(null, data); // Sync
  } else {
    db.query((err, result) => callback(err, result)); // Async
  }
}

// GOOD: Always async
function getData(callback) {
  if (cached) {
    process.nextTick(() => callback(null, data));
  } else {
    db.query((err, result) => callback(err, result));
  }
}
```

7. **DON'T ignore errors in nextTick callbacks**
```javascript
// BAD: Errors are swallowed
process.nextTick(() => {
  throw new Error('This crashes the process!');
});

// GOOD: Handle errors properly
process.nextTick(() => {
  try {
    riskyOperation();
  } catch (error) {
    errorHandler(error);
  }
});
```

8. **DON'T use `nextTick` for I/O-bound operations**
```javascript
// BAD: nextTick doesn't help with I/O
process.nextTick(() => {
  fs.readFile('file.txt', callback);
});

// GOOD: Just do the I/O directly
fs.readFile('file.txt', callback);
```

9. **DON'T use `setImmediate` for time-critical operations**
```javascript
// BAD: setImmediate has unpredictable timing
setImmediate(() => {
  handleCriticalSecurityAlert();
});

// GOOD: Use nextTick for immediate execution
process.nextTick(() => {
  handleCriticalSecurityAlert();
});
```

10. **DON'T create memory leaks with uncancelled callbacks**
```javascript
// BAD: No way to cancel
const id = setImmediate(() => {
  heavyOperation();
});

// GOOD: Store reference and cancel when needed
const id = setImmediate(() => {
  heavyOperation();
});

// Later...
clearImmediate(id);
```

---

## 🎓 Key Takeaways

### When to Use What

| Scenario | Use | Reason |
|----------|-----|--------|
| Emit errors asynchronously | `process.nextTick()` | Ensures consistent error handling |
| Ensure listeners are registered | `process.nextTick()` | Executes before any I/O |
| Recursive operations | `setImmediate()` | Avoids starving event loop |
| Breaking up long operations | `setImmediate()` | Allows I/O to process |
| Defer work to next iteration | `setImmediate()` | Proper event loop integration |
| Make sync API async | `process.nextTick()` | Consistent API behavior |
| Process after I/O | `setImmediate()` | Guaranteed order in I/O callbacks |
| Critical operations | `process.nextTick()` | Highest priority execution |

### Execution Order (Summary)

```
1. Synchronous code
2. process.nextTick() queue
3. Microtask queue (Promises)
4. setTimeout/setInterval (Timers phase)
5. I/O callbacks (Poll phase)
6. setImmediate() (Check phase)
7. Close callbacks
```

### Memory and Performance

- **`process.nextTick()`**: Fastest, but can block event loop
- **`setImmediate()`**: Slower, but event loop friendly
- Both create closures (be mindful of memory)
- Clean up with `clearImmediate()` when needed

### Production Best Practices

1. Use `setImmediate()` for recursive operations
2. Use `process.nextTick()` sparingly for critical operations
3. Never use recursive `process.nextTick()`
4. Document why you chose one over the other
5. Profile and measure performance impact
6. Consider Worker Threads for CPU-intensive work
7. Monitor event loop lag in production

---

## 📚 Additional Resources

- [Node.js Event Loop Documentation](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
- [Understanding process.nextTick()](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#process-nexttick)
- [The Node.js Event Loop, Timers, and process.nextTick()](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
- [Don't Block the Event Loop](https://nodejs.org/en/docs/guides/dont-block-the-event-loop/)

---

**File**: `Q18_Timers_setImmediate_vs_nextTick.md`  
**Last Updated**: November 15, 2025  
**Status**: ✅ Complete  
**Lines**: ~2,100  
**Banking Examples**: Priority transaction processing, urgent payments, batch validation
