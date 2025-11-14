# Node.js Interview Question: Event Loop Phases
## Question 2: Understanding the 6 Phases of the Event Loop

---

## 📋 Summary of What Will Be Covered

### ✅ Question 2: Event Loop Phases
- The 6 phases of the Node.js event loop
- What happens in each phase and in what order
- Timers, pending callbacks, poll, check, close callbacks
- setImmediate() vs setTimeout() execution order
- Banking Example: Payment processing order and priority

---

## ❓ Question 2: Explain the 6 Phases of the Event Loop

### 📘 Comprehensive Explanation

While Q01 covered **what** the event loop is, Q02 dives into **how** it works internally - the specific phases it goes through in each iteration.

**The Big Picture**:

The event loop doesn't just check for callbacks randomly. It follows a **specific order**, going through **6 distinct phases** in each iteration.

**Event Loop Phases Diagram**:

```
   ┌───────────────────────────┐
┌─>│           timers          │  Phase 1: setTimeout/setInterval callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  Phase 2: I/O callbacks deferred to next loop
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  Phase 3: Internal use only
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:   │
│  │           poll            │<─────┤  connections, │  Phase 4: Retrieve new I/O events
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check           │  Phase 5: setImmediate() callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │  Phase 6: cleanup (e.g., socket.on('close'))
   └───────────────────────────┘
```

**After each phase, the event loop checks for microtasks**:
- `process.nextTick()` callbacks
- Promise callbacks (`.then()`, `.catch()`, `.finally()`)

**Microtasks always execute BEFORE moving to the next phase!**

---

### 🔍 Detailed Phase Breakdown

#### **Phase 1: Timers** ⏰

**Purpose**: Execute callbacks scheduled by `setTimeout()` and `setInterval()`

**How it works**:
1. Check if any timer thresholds have been reached
2. Execute callbacks whose time has elapsed
3. Timers are checked against the **poll phase** duration

**Important Notes**:
- Timers are **NOT guaranteed** to execute at exact time
- They execute "as soon as possible" after the threshold
- Can be delayed by blocking operations in other phases

**Example**:
```javascript
setTimeout(() => {
  console.log('Timer callback');
}, 100); // Executes ~100ms later (not exactly 100ms)
```

**Banking Use Case**: 
- Session timeout (15 minutes of inactivity)
- OTP expiration (5 minutes)
- Transaction timeout (30 seconds)

---

#### **Phase 2: Pending Callbacks** 🔄

**Purpose**: Execute I/O callbacks deferred from the previous cycle

**How it works**:
- Some I/O callbacks are deferred to the next event loop iteration
- TCP errors, for example, are handled here
- Most I/O callbacks execute in the **poll** phase, but some are deferred

**Example scenarios**:
- TCP socket errors
- Some system operations
- Deferred callbacks from previous iteration

**Banking Use Case**:
- Retry failed payment gateway connections
- Handle database connection errors
- Process deferred transaction validations

---

#### **Phase 3: Idle, Prepare** 🔧

**Purpose**: Internal use only by Node.js/libuv

**How it works**:
- Used internally for housekeeping
- Not directly accessible to developers
- Prepares for the poll phase

**Why it matters**: Understanding that Node.js does internal work between phases helps explain small timing differences.

---

#### **Phase 4: Poll** 📥 (Most Important Phase!)

**Purpose**: Retrieve new I/O events and execute their callbacks

**This is where most of your code runs!**

**How it works**:
1. **Calculate how long to block and poll for I/O**
2. **Process events in the poll queue**:
   - File system operations (`fs.readFile`)
   - Network requests (HTTP, database queries)
   - Most async operations

**Two main scenarios**:

**Scenario A: Poll queue is NOT empty**
- Execute callbacks synchronously until queue is empty
- Or until system-dependent hard limit is reached

**Scenario B: Poll queue IS empty**
- **If `setImmediate()` is scheduled**: End poll phase, go to check phase
- **If no `setImmediate()`**: Wait for callbacks to be added to queue, then execute them immediately

**This phase can "block" waiting for I/O** (but in a non-blocking way - the thread is free to handle other events).

**Banking Use Case**:
- Database queries (account balance, transaction history)
- External API calls (credit score, fraud detection)
- File reads (KYC documents, statements)

---

#### **Phase 5: Check** ✅

**Purpose**: Execute `setImmediate()` callbacks

**How it works**:
- After the poll phase completes, check phase executes
- All `setImmediate()` callbacks execute here
- **Always executes AFTER poll phase**

**Key Difference**: `setImmediate()` vs `setTimeout()`
```javascript
setImmediate(() => {
  console.log('setImmediate'); // Executes in check phase
});

setTimeout(() => {
  console.log('setTimeout'); // Executes in timers phase
}, 0);

// Output order depends on context!
```

**Banking Use Case**:
- Processing next batch of transactions after current batch completes
- Scheduling follow-up actions after I/O operations
- Deferring non-critical work until after I/O

---

#### **Phase 6: Close Callbacks** 🚪

**Purpose**: Execute close event callbacks

**How it works**:
- Cleanup operations
- Socket close events
- Stream close events

**Example**:
```javascript
socket.on('close', () => {
  console.log('Socket closed'); // Executes in close callbacks phase
});
```

**Banking Use Case**:
- Cleanup when user disconnects
- Close database connections
- Release resources after transaction completes

---

### 🎯 Microtasks: process.nextTick() and Promises

**Critical Understanding**: Microtasks execute **AFTER each phase**, not between phases.

**Microtask Queue Priority**:
1. **`process.nextTick()` queue** - Highest priority, executes first
2. **Promise microtask queue** - Executes after nextTick queue

**Execution Order**:
```
Phase 1 (timers)
  → process.nextTick() callbacks
  → Promise callbacks
Phase 2 (pending callbacks)
  → process.nextTick() callbacks
  → Promise callbacks
Phase 3 (idle, prepare)
  → process.nextTick() callbacks
  → Promise callbacks
... and so on
```

**Example**:
```javascript
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
process.nextTick(() => console.log('nextTick'));
Promise.resolve().then(() => console.log('promise'));

// Output:
// nextTick      (microtask - highest priority)
// promise       (microtask - after nextTick)
// timeout       (timers phase)
// immediate     (check phase)
```

**Warning**: `process.nextTick()` can **starve** the event loop if used recursively!

```javascript
// BAD ❌ - Infinite loop!
function recursiveNextTick() {
  process.nextTick(recursiveNextTick);
  // Event loop never moves to next phase!
}
```

---

### 🏦 Real-World Banking Scenario

**Context**: You're building a **payment processing system** that needs to:
1. **Validate** payment requests (immediate)
2. **Query** account balance from database (I/O)
3. **Call** external payment gateway API (I/O)
4. **Update** transaction status in database (I/O)
5. **Schedule** notification to customer (deferred)
6. **Handle** connection cleanup when user disconnects

**Requirements**:
- Understand which phase handles each operation
- Control execution order for payment priority
- Handle high-priority fraud alerts immediately
- Process batch transactions efficiently

**Challenge**: Payment requests must execute in the correct order:
1. Fraud detection (highest priority - `process.nextTick()`)
2. Payment validation (immediate - synchronous)
3. Database operations (poll phase - async)
4. Post-processing (check phase - `setImmediate()`)

---

### 💻 Production-Ready Code Examples

#### Example 1: Phase Execution Order Demonstration

This example shows **exactly** which phase executes when, with detailed logging.

```javascript
const fs = require('fs');

/**
 * Event Loop Phases Demonstration
 * Shows execution order of different async operations
 */

console.log('🏦 Event Loop Phases - Banking Transaction Processing\n');
console.log('='.repeat(70));
console.log('\n📝 Starting execution...\n');

// Track execution order
let executionOrder = 1;

function log(message, phase) {
  console.log(`${executionOrder++}. [${phase.padEnd(20)}] ${message}`);
}

// ============================================
// Synchronous code - Executes first
// ============================================
log('Synchronous: Start', 'MAIN THREAD');

// ============================================
// Phase 1: Timers
// ============================================
setTimeout(() => {
  log('setTimeout 0ms callback', 'TIMERS PHASE');
}, 0);

setTimeout(() => {
  log('setTimeout 10ms callback', 'TIMERS PHASE');
}, 10);

// ============================================
// Phase 4: Poll (I/O operations)
// ============================================
fs.readFile(__filename, () => {
  log('fs.readFile callback', 'POLL PHASE');
  
  // Nested operations to show priority
  setTimeout(() => {
    log('setTimeout inside readFile', 'TIMERS PHASE');
  }, 0);
  
  setImmediate(() => {
    log('setImmediate inside readFile', 'CHECK PHASE');
  });
  
  process.nextTick(() => {
    log('nextTick inside readFile', 'MICROTASK');
  });
  
  Promise.resolve().then(() => {
    log('Promise inside readFile', 'MICROTASK');
  });
});

// ============================================
// Phase 5: Check
// ============================================
setImmediate(() => {
  log('setImmediate callback', 'CHECK PHASE');
});

// ============================================
// Microtasks
// ============================================
process.nextTick(() => {
  log('process.nextTick callback', 'MICROTASK');
});

Promise.resolve().then(() => {
  log('Promise.resolve callback', 'MICROTASK');
});

// ============================================
// More synchronous code
// ============================================
log('Synchronous: End', 'MAIN THREAD');

console.log('\n' + '='.repeat(70));
console.log('\n⏱️  Waiting for async operations to complete...\n');

// Summary after all operations
setTimeout(() => {
  console.log('='.repeat(70));
  console.log('\n📊 Execution Summary:\n');
  console.log('Order of execution:');
  console.log('  1. Synchronous code (MAIN THREAD)');
  console.log('  2. Microtasks (nextTick, then Promises)');
  console.log('  3. Timers phase (setTimeout)');
  console.log('  4. Poll phase (I/O like fs.readFile)');
  console.log('  5. Check phase (setImmediate)');
  console.log('\n💡 Key Insight:');
  console.log('  - Microtasks execute BETWEEN every phase');
  console.log('  - process.nextTick() has highest priority');
  console.log('  - setImmediate() executes after poll phase\n');
  console.log('='.repeat(70) + '\n');
}, 100);

```

**To test this example:**

```bash
# Run the demonstration
node event-loop-phases.js
```

**Expected Output**:

```
🏦 Event Loop Phases - Banking Transaction Processing

======================================================================

📝 Starting execution...

1. [MAIN THREAD         ] Synchronous: Start
2. [MAIN THREAD         ] Synchronous: End

======================================================================

⏱️  Waiting for async operations to complete...

3. [MICROTASK           ] process.nextTick callback
4. [MICROTASK           ] Promise.resolve callback
5. [TIMERS PHASE        ] setTimeout 0ms callback
6. [POLL PHASE          ] fs.readFile callback
7. [MICROTASK           ] nextTick inside readFile
8. [MICROTASK           ] Promise inside readFile
9. [CHECK PHASE         ] setImmediate inside readFile
10. [CHECK PHASE         ] setImmediate callback
11. [TIMERS PHASE        ] setTimeout 10ms callback
12. [TIMERS PHASE        ] setTimeout inside readFile
======================================================================

📊 Execution Summary:

Order of execution:
  1. Synchronous code (MAIN THREAD)
  2. Microtasks (nextTick, then Promises)
  3. Timers phase (setTimeout)
  4. Poll phase (I/O like fs.readFile)
  5. Check phase (setImmediate)

💡 Key Insight:
  - Microtasks execute BETWEEN every phase
  - process.nextTick() has highest priority
  - setImmediate() executes after poll phase

======================================================================
```

**Key Learning**:
- **Synchronous code** always executes first
- **Microtasks** execute before any phase
- **`process.nextTick()`** has highest priority
- **Timers** execute before poll phase
- **setImmediate()** executes after poll phase

---

#### Example 2: setImmediate() vs setTimeout() - The Tricky Part

This example demonstrates the **most confusing aspect** of event loop phases: when `setImmediate()` executes before or after `setTimeout()`.

```javascript
const fs = require('fs');

/**
 * setImmediate vs setTimeout Comparison
 * Understanding execution order in different contexts
 */

console.log('\n🏦 setImmediate vs setTimeout - Banking Payment Priority\n');
console.log('='.repeat(70));

// ============================================
// Test 1: Main Module (Non-I/O Context)
// ============================================

console.log('\n📊 Test 1: Main Module Context\n');
console.log('Executing setImmediate and setTimeout(0) in main module...\n');

setTimeout(() => {
  console.log('  ⏰ setTimeout(0)');
}, 0);

setImmediate(() => {
  console.log('  ✅ setImmediate()');
});

console.log('Result: Order is NOT guaranteed!');
console.log('Why: Depends on system performance and other operations');
console.log('     - Could be setTimeout first OR setImmediate first');
console.log('     - Both are scheduled, but entry to event loop varies\n');

// ============================================
// Test 2: I/O Callback Context (Guaranteed Order!)
// ============================================

setTimeout(() => {
  console.log('\n' + '─'.repeat(70));
  console.log('\n📊 Test 2: Inside I/O Callback (fs.readFile)\n');
  console.log('Executing setImmediate and setTimeout(0) inside I/O callback...\n');
  
  fs.readFile(__filename, () => {
    setTimeout(() => {
      console.log('  ⏰ setTimeout(0) - inside readFile');
    }, 0);
    
    setImmediate(() => {
      console.log('  ✅ setImmediate() - inside readFile');
    });
    
    // Wait a bit then show explanation
    setTimeout(() => {
      console.log('\nResult: setImmediate ALWAYS executes first!');
      console.log('Why: We\'re already in POLL phase');
      console.log('     - setImmediate executes in CHECK phase (immediately next)');
      console.log('     - setTimeout executes in TIMERS phase (next iteration)\n');
      
      runBankingExample();
    }, 100);
  });
}, 50);

// ============================================
// Banking Example: Payment Priority System
// ============================================

function runBankingExample() {
  console.log('─'.repeat(70));
  console.log('\n🏦 Banking Example: Payment Processing Priority\n');
  console.log('Scenario: Process multiple payment types with different priorities\n');
  
  // Simulate database query (I/O operation - Poll phase)
  fs.readFile(__filename, () => {
    console.log('📥 [POLL PHASE] Database query completed\n');
    
    // High-priority fraud alert (process.nextTick - Highest priority)
    process.nextTick(() => {
      console.log('🚨 [MICROTASK - nextTick] FRAUD ALERT! Suspicious activity detected');
      console.log('   → Block account immediately');
      console.log('   → Send notification to security team\n');
    });
    
    // Immediate validation (Promise - Microtask, but after nextTick)
    Promise.resolve().then(() => {
      console.log('✓ [MICROTASK - Promise] Payment validation completed');
      console.log('   → Amount: $5,000');
      console.log('   → Payee: Valid\n');
    });
    
    // Post-processing: Update analytics (setImmediate - Check phase)
    setImmediate(() => {
      console.log('📊 [CHECK PHASE - setImmediate] Post-processing');
      console.log('   → Update transaction analytics');
      console.log('   → Log to audit system');
      console.log('   → Cache updated balance\n');
    });
    
    // Scheduled notification (setTimeout - Timers phase, next iteration)
    setTimeout(() => {
      console.log('📧 [TIMERS PHASE - setTimeout] Send customer notification');
      console.log('   → Email: Transaction completed');
      console.log('   → SMS: Balance updated\n');
    }, 0);
    
    // Batch processing for next cycle (setImmediate)
    setImmediate(() => {
      console.log('📦 [CHECK PHASE - setImmediate] Batch processing');
      console.log('   → Queue next 100 transactions');
      console.log('   → Prepare for next iteration\n');
    });
    
    console.log('💡 [POLL PHASE] All operations scheduled\n');
    
    // Show summary after all operations complete
    setTimeout(() => {
      console.log('='.repeat(70));
      console.log('\n📋 Execution Order Summary:\n');
      console.log('1. 🚨 Fraud Alert (process.nextTick)      - HIGHEST PRIORITY');
      console.log('2. ✓  Validation (Promise)                 - High priority');
      console.log('3. 📊 Post-processing (setImmediate)       - After current phase');
      console.log('4. 📦 Batch processing (setImmediate)      - After current phase');
      console.log('5. 📧 Notification (setTimeout)            - Next event loop iteration');
      console.log('\n💡 Key Insights:');
      console.log('   • process.nextTick() = Critical operations (fraud, security)');
      console.log('   • Promises = Important validations');
      console.log('   • setImmediate() = Post-processing, next batch');
      console.log('   • setTimeout() = Deferred, non-critical tasks\n');
      console.log('='.repeat(70) + '\n');
    }, 200);
  });
}

```

**To test this example:**

```bash
# Run the comparison
node setimmediate-vs-settimeout.js
```

**Expected Output**:

```
🏦 setImmediate vs setTimeout - Banking Payment Priority

======================================================================

📊 Test 1: Main Module Context

Executing setImmediate and setTimeout(0) in main module...

  ⏰ setTimeout(0)
  ✅ setImmediate()

Result: Order is NOT guaranteed!
Why: Depends on system performance and other operations
     - Could be setTimeout first OR setImmediate first
     - Both are scheduled, but entry to event loop varies

──────────────────────────────────────────────────────────────────────

📊 Test 2: Inside I/O Callback (fs.readFile)

Executing setImmediate and setTimeout(0) inside I/O callback...

  ✅ setImmediate() - inside readFile
  ⏰ setTimeout(0) - inside readFile

Result: setImmediate ALWAYS executes first!
Why: We're already in POLL phase
     - setImmediate executes in CHECK phase (immediately next)
     - setTimeout executes in TIMERS phase (next iteration)

──────────────────────────────────────────────────────────────────────

🏦 Banking Example: Payment Processing Priority

Scenario: Process multiple payment types with different priorities

📥 [POLL PHASE] Database query completed

💡 [POLL PHASE] All operations scheduled

🚨 [MICROTASK - nextTick] FRAUD ALERT! Suspicious activity detected
   → Block account immediately
   → Send notification to security team

✓ [MICROTASK - Promise] Payment validation completed
   → Amount: $5,000
   → Payee: Valid

📊 [CHECK PHASE - setImmediate] Post-processing
   → Update transaction analytics
   → Log to audit system
   → Cache updated balance

📦 [CHECK PHASE - setImmediate] Batch processing
   → Queue next 100 transactions
   → Prepare for next iteration

📧 [TIMERS PHASE - setTimeout] Send customer notification
   → Email: Transaction completed
   → SMS: Balance updated

======================================================================

📋 Execution Order Summary:

1. 🚨 Fraud Alert (process.nextTick)      - HIGHEST PRIORITY
2. ✓  Validation (Promise)                 - High priority
3. 📊 Post-processing (setImmediate)       - After current phase
4. 📦 Batch processing (setImmediate)      - After current phase
5. 📧 Notification (setTimeout)            - Next event loop iteration

💡 Key Insights:
   • process.nextTick() = Critical operations (fraud, security)
   • Promises = Important validations
   • setImmediate() = Post-processing, next batch
   • setTimeout() = Deferred, non-critical tasks

======================================================================
```

**Critical Rules**:

1. **In main module**: `setTimeout(0)` vs `setImmediate()` order is **non-deterministic**
2. **Inside I/O callback**: `setImmediate()` **ALWAYS** executes before `setTimeout(0)`
3. **`process.nextTick()`**: **ALWAYS** executes first, regardless of context
4. **Promises**: Execute after `nextTick()` but before any phase

---

#### Example 3: Real Banking Payment Processing System

This example shows a **production-ready payment processor** that uses event loop phases correctly.

```javascript
const EventEmitter = require('events');
const crypto = require('crypto');

/**
 * Banking Payment Processor
 * Uses event loop phases for optimal performance and priority
 */

class PaymentProcessor extends EventEmitter {
  constructor() {
    super();
    
    this.queue = {
      critical: [],    // Fraud alerts, security issues
      high: [],        // Standard payments
      normal: [],      // Batch processing
      low: []          // Analytics, notifications
    };
    
    this.stats = {
      processed: 0,
      fraudBlocked: 0,
      avgProcessingTime: 0
    };
    
    this.processing = false;
  }
  
  /**
   * Submit payment for processing
   */
  async submitPayment(payment) {
    payment.id = payment.id || this.generateTransactionId();
    payment.submittedAt = Date.now();
    payment.status = 'queued';
    
    console.log(`\n💳 Payment submitted: ${payment.id}`);
    console.log(`   From: ${payment.fromAccount}`);
    console.log(`   To: ${payment.toAccount}`);
    console.log(`   Amount: $${payment.amount}`);
    
    // Immediate fraud check (process.nextTick - Critical!)
    process.nextTick(() => {
      this.checkFraud(payment);
    });
    
    // Queue for processing
    this.queue.high.push(payment);
    
    // Trigger processing (setImmediate - next after current I/O)
    setImmediate(() => {
      if (!this.processing) {
        this.processQueue();
      }
    });
    
    return payment.id;
  }
  
  /**
   * Critical: Fraud detection (process.nextTick)
   */
  checkFraud(payment) {
    console.log(`   🔍 [MICROTASK] Fraud check for ${payment.id}...`);
    
    // Simple fraud rules
    const isFraudulent = 
      payment.amount > 50000 || 
      payment.toAccount === 'SUSPICIOUS_ACCOUNT';
    
    if (isFraudulent) {
      console.log(`   🚨 [MICROTASK] FRAUD DETECTED! Blocking ${payment.id}`);
      
      payment.status = 'blocked';
      payment.blockedReason = 'Fraud detection triggered';
      
      this.stats.fraudBlocked++;
      
      // Remove from queue
      this.queue.high = this.queue.high.filter(p => p.id !== payment.id);
      
      // Emit fraud event (for immediate handling)
      this.emit('fraud-detected', payment);
      
    } else {
      console.log(`   ✅ [MICROTASK] Fraud check passed for ${payment.id}`);
    }
  }
  
  /**
   * Process payment queue (setImmediate)
   */
  async processQueue() {
    if (this.processing) return;
    
    this.processing = true;
    
    console.log(`\n📊 [CHECK PHASE] Processing payment queue...`);
    
    // Process critical queue first
    await this.processQueueByPriority('critical');
    
    // Then high priority
    await this.processQueueByPriority('high');
    
    // Schedule normal and low priority for later (setTimeout)
    if (this.queue.normal.length > 0 || this.queue.low.length > 0) {
      setTimeout(() => {
        this.processBatchOperations();
      }, 0);
    }
    
    this.processing = false;
  }
  
  /**
   * Process queue by priority
   */
  async processQueueByPriority(priority) {
    const queue = this.queue[priority];
    
    while (queue.length > 0) {
      const payment = queue.shift();
      
      if (payment.status === 'blocked') {
        continue; // Skip blocked payments
      }
      
      await this.processPayment(payment);
    }
  }
  
  /**
   * Process single payment (async I/O - Poll phase)
   */
  async processPayment(payment) {
    const startTime = Date.now();
    
    console.log(`\n⚙️  [POLL PHASE] Processing ${payment.id}...`);
    
    payment.status = 'processing';
    
    try {
      // Step 1: Validate accounts (async I/O)
      await this.validateAccounts(payment);
      console.log(`   ✓ Accounts validated`);
      
      // Step 2: Check balance (async I/O)
      await this.checkBalance(payment);
      console.log(`   ✓ Balance sufficient`);
      
      // Step 3: Debit source account (async I/O)
      await this.debitAccount(payment.fromAccount, payment.amount);
      console.log(`   ✓ Debited $${payment.amount} from ${payment.fromAccount}`);
      
      // Step 4: Credit destination account (async I/O)
      await this.creditAccount(payment.toAccount, payment.amount);
      console.log(`   ✓ Credited $${payment.amount} to ${payment.toAccount}`);
      
      // Step 5: Record transaction (async I/O)
      await this.recordTransaction(payment);
      console.log(`   ✓ Transaction recorded`);
      
      payment.status = 'completed';
      payment.completedAt = Date.now();
      payment.processingTime = payment.completedAt - startTime;
      
      this.stats.processed++;
      
      console.log(`✅ Payment ${payment.id} completed (${payment.processingTime}ms)`);
      
      // Post-processing operations (setImmediate - Check phase)
      setImmediate(() => {
        this.postProcessPayment(payment);
      });
      
      // Emit completion event
      this.emit('payment-completed', payment);
      
    } catch (error) {
      console.log(`❌ Payment ${payment.id} failed: ${error.message}`);
      
      payment.status = 'failed';
      payment.error = error.message;
      
      // Rollback if needed
      setImmediate(() => {
        this.rollbackPayment(payment);
      });
      
      this.emit('payment-failed', payment);
    }
  }
  
  /**
   * Post-processing (setImmediate - Check phase)
   */
  postProcessPayment(payment) {
    console.log(`\n📊 [CHECK PHASE] Post-processing ${payment.id}...`);
    
    // Update analytics
    console.log(`   ✓ Analytics updated`);
    
    // Cache balance
    console.log(`   ✓ Balance cached`);
    
    // Queue notification for later (setTimeout)
    setTimeout(() => {
      this.sendNotification(payment);
    }, 0);
  }
  
  /**
   * Send notifications (setTimeout - Timers phase, next iteration)
   */
  sendNotification(payment) {
    console.log(`\n📧 [TIMERS PHASE] Sending notifications for ${payment.id}...`);
    console.log(`   ✓ Email sent to customer`);
    console.log(`   ✓ SMS notification sent`);
    console.log(`   ✓ Push notification sent`);
  }
  
  /**
   * Batch operations (setTimeout - Low priority)
   */
  processBatchOperations() {
    console.log(`\n📦 [TIMERS PHASE] Processing batch operations...`);
    console.log(`   • Updating daily statistics`);
    console.log(`   • Generating reports`);
    console.log(`   • Archiving old transactions`);
  }
  
  /**
   * Simulate async operations
   */
  async validateAccounts(payment) {
    await this.delay(10);
    if (!payment.fromAccount || !payment.toAccount) {
      throw new Error('Invalid accounts');
    }
  }
  
  async checkBalance(payment) {
    await this.delay(15);
    // Simulate balance check
    const balance = 10000;
    if (balance < payment.amount) {
      throw new Error('Insufficient balance');
    }
  }
  
  async debitAccount(account, amount) {
    await this.delay(20);
    // Simulate database update
  }
  
  async creditAccount(account, amount) {
    await this.delay(20);
    // Simulate database update
  }
  
  async recordTransaction(payment) {
    await this.delay(15);
    // Simulate database insert
  }
  
  async rollbackPayment(payment) {
    console.log(`\n🔄 [CHECK PHASE] Rolling back ${payment.id}...`);
    console.log(`   • Reversing debits`);
    console.log(`   • Refunding amounts`);
    console.log(`   • Updating status`);
  }
  
  delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
  
  generateTransactionId() {
    return 'TXN' + crypto.randomBytes(8).toString('hex').toUpperCase();
  }
  
  /**
   * Get statistics
   */
  getStats() {
    return {
      ...this.stats,
      queueSizes: {
        critical: this.queue.critical.length,
        high: this.queue.high.length,
        normal: this.queue.normal.length,
        low: this.queue.low.length
      }
    };
  }
}

// ============================================
// Demo Usage
// ============================================

console.log('🏦 Banking Payment Processing System\n');
console.log('='.repeat(70));

const processor = new PaymentProcessor();

// Listen to events
processor.on('fraud-detected', (payment) => {
  console.log(`\n⚠️  [EVENT] Fraud detected for ${payment.id} - Security team notified`);
});

processor.on('payment-completed', (payment) => {
  console.log(`\n✅ [EVENT] Payment ${payment.id} completed successfully`);
});

processor.on('payment-failed', (payment) => {
  console.log(`\n❌ [EVENT] Payment ${payment.id} failed - Customer notified`);
});

// Submit test payments
(async () => {
  // Normal payment
  await processor.submitPayment({
    fromAccount: 'ACC001',
    toAccount: 'ACC002',
    amount: 1000
  });
  
  // Wait a bit
  await new Promise(resolve => setTimeout(resolve, 200));
  
  // High amount payment (triggers fraud check)
  await processor.submitPayment({
    fromAccount: 'ACC003',
    toAccount: 'ACC004',
    amount: 60000
  });
  
  // Wait for all processing
  setTimeout(() => {
    console.log('\n' + '='.repeat(70));
    console.log('\n📊 Final Statistics:\n');
    const stats = processor.getStats();
    console.log(`   Processed: ${stats.processed}`);
    console.log(`   Fraud Blocked: ${stats.fraudBlocked}`);
    console.log('\n' + '='.repeat(70) + '\n');
  }, 500);
})();

```

**To test this example:**

```bash
# Run the payment processor
node payment-processor.js
```

**Key Features**:

1. **Priority-based Processing**:
   - `process.nextTick()`: Fraud detection (critical)
   - `Promise`: Validation (high)
   - `setImmediate()`: Post-processing (medium)
   - `setTimeout()`: Notifications (low)

2. **Event Loop Phases Used Correctly**:
   - **Microtasks**: Critical security checks
   - **Poll Phase**: Database operations (I/O)
   - **Check Phase**: Post-processing, analytics
   - **Timers Phase**: Deferred notifications, batch jobs

3. **Production-Ready**:
   - Error handling and rollback
   - Event emission for monitoring
   - Statistics tracking
   - Queue management

---

### ✅ DO's and ❌ DON'Ts

#### ✅ DO's:

1. **DO** use `process.nextTick()` for critical, high-priority operations
   ```javascript
   // Good ✅ - Fraud detection before anything else
   process.nextTick(() => {
     checkForFraud(transaction);
   });
   ```

2. **DO** use `setImmediate()` for operations after I/O completes
   ```javascript
   // Good ✅ - Post-processing after database operation
   db.query('SELECT * FROM accounts', (err, results) => {
     processResults(results);
     setImmediate(() => {
       updateCache(results); // Defer to check phase
     });
   });
   ```

3. **DO** use `setTimeout()` for deferred, non-critical operations
   ```javascript
   // Good ✅ - Send notifications later
   setTimeout(() => {
     sendEmailNotification(user);
   }, 0);
   ```

4. **DO** understand that microtasks execute between every phase
   ```javascript
   // Good ✅ - Knowing execution order
   setTimeout(() => console.log('timer'), 0);
   Promise.resolve().then(() => console.log('promise')); // Executes first
   ```

5. **DO** use `setImmediate()` over `setTimeout(0)` inside I/O callbacks
   ```javascript
   // Good ✅ - Guaranteed to execute immediately after I/O
   fs.readFile('file.txt', (err, data) => {
     setImmediate(() => {
       processData(data); // Executes in check phase (next)
     });
   });
   ```

6. **DO** break up CPU-intensive work across multiple event loop iterations
   ```javascript
   // Good ✅ - Yield to event loop
   async function processLargeDataset(items) {
     for (let i = 0; i < items.length; i++) {
       processItem(items[i]);
       if (i % 1000 === 0) {
         await new Promise(resolve => setImmediate(resolve));
       }
     }
   }
   ```

7. **DO** use Promises over callbacks for better async flow
   ```javascript
   // Good ✅ - Cleaner async code
   async function processPayment() {
     await validateAccount();
     await checkBalance();
     await transferFunds();
   }
   ```

8. **DO** handle Promise rejections
   ```javascript
   // Good ✅ - Proper error handling
   Promise.resolve()
     .then(() => riskyOperation())
     .catch(err => console.error('Error:', err));
   ```

9. **DO** monitor event loop lag in production
   ```javascript
   // Good ✅ - Detect performance issues
   const start = Date.now();
   setImmediate(() => {
     const lag = Date.now() - start;
     if (lag > 50) console.warn('Event loop lag:', lag);
   });
   ```

10. **DO** use event loop phases to prioritize operations
    ```javascript
    // Good ✅ - Security first, notifications last
    process.nextTick(() => checkSecurity());      // Highest priority
    Promise.resolve().then(() => validateData()); // High priority
    setImmediate(() => updateAnalytics());        // Medium priority
    setTimeout(() => sendNotification(), 0);      // Low priority
    ```

#### ❌ DON'Ts:

1. **DON'T** use `process.nextTick()` recursively
   ```javascript
   // Bad ❌ - Starves the event loop!
   function recursiveNextTick() {
     process.nextTick(recursiveNextTick);
     // Event loop never moves to next phase!
   }
   ```

2. **DON'T** use `process.nextTick()` for regular async operations
   ```javascript
   // Bad ❌ - Use setImmediate instead
   process.nextTick(() => {
     updateCache(); // Not critical, shouldn't use nextTick
   });
   ```

3. **DON'T** assume `setTimeout(0)` executes before `setImmediate()`
   ```javascript
   // Bad ❌ - Order is non-deterministic in main module
   setTimeout(() => console.log('timeout'), 0);
   setImmediate(() => console.log('immediate'));
   // Could be either order!
   ```

4. **DON'T** forget that microtasks can delay phase transitions
   ```javascript
   // Bad ❌ - Too many microtasks delay other work
   for (let i = 0; i < 10000; i++) {
     Promise.resolve().then(() => {
       // Delays moving to next phase
     });
   }
   ```

5. **DON'T** perform blocking operations in any phase
   ```javascript
   // Bad ❌ - Blocks entire event loop
   setTimeout(() => {
     const result = crypto.pbkdf2Sync('password', 'salt', 100000, 64, 'sha512');
     // Blocks all other operations!
   }, 0);
   ```

6. **DON'T** rely on precise timing with setTimeout
   ```javascript
   // Bad ❌ - Not guaranteed to be exact
   setTimeout(() => {
     console.log('Exactly 100ms later?'); // Could be 100ms, 105ms, or more
   }, 100);
   ```

7. **DON'T** mix callback styles without understanding phases
   ```javascript
   // Bad ❌ - Confusing execution order
   fs.readFile('file.txt', () => {
     setTimeout(() => { /* timers */ }, 0);
     setImmediate(() => { /* check */ });
     process.nextTick(() => { /* microtask */ });
     // Hard to predict order!
   });
   ```

8. **DON'T** forget error handling in Promise chains
   ```javascript
   // Bad ❌ - Unhandled rejection
   Promise.resolve()
     .then(() => riskyOperation())
     // No .catch()! Will cause unhandled rejection
   ```

9. **DON'T** use `setImmediate()` for time-critical operations
   ```javascript
   // Bad ❌ - Use process.nextTick for critical operations
   setImmediate(() => {
     blockFraudulentTransaction(); // Too late! Use nextTick
   });
   ```

10. **DON'T** ignore the poll phase's importance
    ```javascript
    // Bad ❌ - Blocking poll phase affects all I/O
    app.get('/api/data', (req, res) => {
      // Long synchronous operation in I/O callback
      for (let i = 0; i < 1000000000; i++) {}
      // Blocks ALL other requests!
    });
    ```

---

### 🎯 Key Takeaways

#### **The 6 Phases Summary**:

```
┌─────────────────────────────────────────────────────────────┐
│                 EVENT LOOP ITERATION                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. TIMERS PHASE          → setTimeout, setInterval         │
│     ↓ (microtasks)                                          │
│                                                             │
│  2. PENDING CALLBACKS     → Deferred I/O callbacks          │
│     ↓ (microtasks)                                          │
│                                                             │
│  3. IDLE, PREPARE         → Internal Node.js operations     │
│     ↓ (microtasks)                                          │
│                                                             │
│  4. POLL PHASE ⭐         → Most I/O callbacks execute here │
│     ↓ (microtasks)        → Database, file system, network  │
│                                                             │
│  5. CHECK PHASE           → setImmediate callbacks          │
│     ↓ (microtasks)                                          │
│                                                             │
│  6. CLOSE CALLBACKS       → socket.on('close'), etc.        │
│     ↓ (microtasks)                                          │
│                                                             │
│  ← Back to TIMERS (next iteration)                          │
└─────────────────────────────────────────────────────────────┘

After EACH phase:
  1. process.nextTick() callbacks (highest priority)
  2. Promise microtasks
```

#### **Execution Priority**:

| Priority | Mechanism | Use Case | Example |
|----------|-----------|----------|---------|
| 🔴 Highest | `process.nextTick()` | Critical operations | Fraud detection, security |
| 🟠 Very High | `Promise` | Important validation | Payment validation |
| 🟡 High | `setImmediate()` | Post-I/O processing | Analytics, caching |
| 🟢 Normal | `setTimeout(0)` | Deferred tasks | Notifications |
| 🔵 Low | `setTimeout(delay)` | Scheduled tasks | Cleanup, reports |

#### **setImmediate vs setTimeout**:

**In Main Module**:
```javascript
// Order is NON-DETERMINISTIC
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
// Could be either order!
```

**Inside I/O Callback**:
```javascript
fs.readFile('file.txt', () => {
  // setImmediate ALWAYS executes first
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
  // Output: immediate, timeout (guaranteed)
});
```

**Reason**: 
- In I/O callback, we're already in **poll phase**
- **Check phase** (setImmediate) comes immediately after poll
- **Timers phase** (setTimeout) comes in next iteration

#### **Best Practices for Banking Applications**:

1. **Security First**:
   ```javascript
   process.nextTick(() => checkFraud());  // Execute immediately
   ```

2. **Validation Second**:
   ```javascript
   Promise.resolve().then(() => validatePayment());
   ```

3. **Processing Third**:
   ```javascript
   // Database operations execute in poll phase automatically
   await db.query('UPDATE accounts...');
   ```

4. **Post-Processing Fourth**:
   ```javascript
   setImmediate(() => updateAnalytics());
   ```

5. **Notifications Last**:
   ```javascript
   setTimeout(() => sendEmail(), 0);
   ```

#### **Common Patterns**:

**Pattern 1: Priority Queue**
```javascript
// Critical
process.nextTick(() => handleCritical());

// High priority
Promise.resolve().then(() => handleHigh());

// Normal priority
setImmediate(() => handleNormal());

// Low priority
setTimeout(() => handleLow(), 0);
```

**Pattern 2: Batch Processing**
```javascript
function processBatch(items) {
  let index = 0;
  
  function processNext() {
    if (index >= items.length) return;
    
    processItem(items[index++]);
    
    // Yield to event loop every 100 items
    if (index % 100 === 0) {
      setImmediate(processNext);
    } else {
      processNext();
    }
  }
  
  processNext();
}
```

**Pattern 3: Post-I/O Operations**
```javascript
db.query('SELECT * FROM accounts', (err, results) => {
  // Process results
  processResults(results);
  
  // Defer non-critical operations
  setImmediate(() => {
    updateCache(results);
    logAnalytics(results);
  });
});
```

#### **Performance Impact**:

| Scenario | Wrong Approach | Right Approach | Improvement |
|----------|---------------|----------------|-------------|
| Critical operation | `setTimeout()` | `process.nextTick()` | Executes immediately |
| Post-I/O work | `setTimeout()` | `setImmediate()` | 2x faster |
| Notifications | `setImmediate()` | `setTimeout()` | Better priority |
| Batch processing | Synchronous loop | `setImmediate()` per batch | Event loop stays responsive |

---

### 📚 Further Reading

- **Node.js Docs**: [Event Loop Phases](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
- **libuv Design**: [Understanding the Event Loop](http://docs.libuv.org/en/v1.x/design.html)
- **Node.js Timers**: [Timers in Node.js](https://nodejs.org/api/timers.html)

---

### 🏁 Summary

The Node.js event loop operates in **6 distinct phases**:

1. **Timers**: Execute `setTimeout()` and `setInterval()` callbacks
2. **Pending Callbacks**: Execute deferred I/O callbacks
3. **Idle, Prepare**: Internal Node.js operations
4. **Poll**: ⭐ Most important - execute I/O callbacks
5. **Check**: Execute `setImmediate()` callbacks
6. **Close Callbacks**: Execute close event callbacks

**After each phase**, the event loop processes **microtasks**:
- `process.nextTick()` callbacks (highest priority)
- Promise callbacks

**Key Rules**:
- ✅ `process.nextTick()` = Critical operations (security, fraud)
- ✅ Promises = Important validations
- ✅ `setImmediate()` = Post-I/O processing
- ✅ `setTimeout()` = Deferred, non-critical tasks

**For Banking Applications**:
This phase structure enables efficient **priority-based processing**:
1. Fraud detection (nextTick) - Highest priority
2. Payment validation (Promise) - High priority
3. Database operations (Poll phase) - Normal I/O
4. Analytics/caching (setImmediate) - Post-processing
5. Notifications (setTimeout) - Low priority

Understanding these phases is **critical** for building high-performance, secure banking applications!

---

**File Status**: ✅ Complete  
**Previous Topic**: Q01 - Event Loop Basics  
**Next Topic**: Q03 - Streams Fundamentals

