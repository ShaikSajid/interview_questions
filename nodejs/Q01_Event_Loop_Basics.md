# Node.js Interview Question: Event Loop Basics
## Question 1: Understanding the Event Loop Fundamentals

---

## 📋 Summary of What Will Be Covered

### ✅ Question 1: Event Loop Basics
- What is the Node.js event loop and why it exists
- How the event loop works with the call stack
- Blocking vs non-blocking operations
- Event queue and callback queue
- Banking Example: Processing transaction queue without blocking

---

## ❓ Question 1: What is the Node.js Event Loop?

### 📘 Comprehensive Explanation

**The Problem Node.js Solves**:

Traditional servers (like Apache with PHP) create a **new thread for each request**:
```
Request 1 → Thread 1 (8MB RAM)
Request 2 → Thread 2 (8MB RAM)
Request 3 → Thread 3 (8MB RAM)
...
1000 requests → 1000 threads (8GB RAM!)
```

**Problems**:
- Each thread consumes 8-10MB of memory
- Context switching between threads is expensive
- Limited by CPU cores and RAM
- Threads block while waiting for I/O (database, file system, network)

**The Node.js Solution: Event Loop**

Node.js uses a **single-threaded event loop** with **non-blocking I/O**:
```
Request 1 ──┐
Request 2 ──┤
Request 3 ──┼──→ [Single Thread + Event Loop] ──→ Handle all requests
Request 4 ──┤
Request 5 ──┘
```

**How It Works**:

1. **Single Thread**: One thread handles all requests
2. **Event Loop**: Continuously checks for tasks to execute
3. **Non-blocking I/O**: I/O operations don't block the thread
4. **Callback Queue**: Callbacks are queued and executed when I/O completes

**Architecture**:

```
┌────────────────────────────────────────────────────────────────┐
│                        Node.js Process                         │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────────┐                                         │
│  │   Call Stack     │  ← Executes synchronous code           │
│  │                  │                                          │
│  │  function()      │                                          │
│  │  function()      │                                          │
│  └──────────────────┘                                         │
│           ↓                                                    │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │              Event Loop (libuv)                          │ │
│  │  Continuously checks:                                    │ │
│  │  - Is call stack empty?                                  │ │
│  │  - Are there callbacks ready?                            │ │
│  │  - If yes, push callback to call stack                   │ │
│  └──────────────────────────────────────────────────────────┘ │
│           ↑                                                    │
│  ┌──────────────────┐         ┌──────────────────┐           │
│  │  Callback Queue  │         │   I/O Operations │           │
│  │                  │←─────── │  (Non-blocking)  │           │
│  │  callback1()     │         │                  │           │
│  │  callback2()     │         │  Database        │           │
│  │  callback3()     │         │  File System     │           │
│  │                  │         │  Network         │           │
│  └──────────────────┘         └──────────────────┘           │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

**Key Concepts**:

1. **Call Stack**: 
   - LIFO (Last In, First Out) structure
   - Executes functions synchronously
   - Only one function executes at a time

2. **Event Loop**:
   - Checks if call stack is empty
   - If empty, takes callback from queue
   - Pushes callback to call stack

3. **Callback Queue**:
   - Stores callbacks waiting to execute
   - FIFO (First In, First Out)
   - Callbacks execute only when call stack is empty

4. **Non-blocking I/O**:
   - I/O operations run in background (libuv thread pool)
   - Main thread doesn't wait
   - Callback added to queue when I/O completes

**Blocking vs Non-blocking**:

**Blocking Example** (Bad):
```javascript
// This blocks the event loop!
const data = fs.readFileSync('large-file.txt'); // Wait here (blocking)
console.log(data);
// Nothing else can execute during file read
```

**Non-blocking Example** (Good):
```javascript
// This doesn't block!
fs.readFile('large-file.txt', (err, data) => {
  console.log(data); // Executes later
});
// Code continues immediately, file read happens in background
console.log('I execute immediately!');
```

**Why This Matters for Banking**:

A banking API needs to handle:
- **Thousands of concurrent users** checking balances
- **Database queries** (I/O operation)
- **External API calls** (payment gateways, credit bureaus)
- **File operations** (generating statements)

With blocking I/O:
```
User 1: Check balance → Query DB (wait 50ms) → Block thread
User 2: Wait for User 1...
User 3: Wait for User 1 and 2...
Result: Can handle ~20 requests/second
```

With non-blocking I/O (Event Loop):
```
User 1: Check balance → Query DB (background)
User 2: Check balance → Query DB (background)
User 3: Check balance → Query DB (background)
...all execute simultaneously...
Result: Can handle 10,000+ requests/second
```

**Real-World Performance**:
- **Blocking**: 1 thread handles ~20-50 req/sec
- **Non-blocking (Event Loop)**: 1 thread handles 5,000-10,000 req/sec
- **200x improvement** with single thread!

---

### 🏦 Real-World Banking Scenario

**Context**: You're building a **banking API** that needs to handle:
- **10,000 users** checking balances simultaneously
- Each balance check requires a **database query** (30ms average)
- Traditional blocking approach would need **300 seconds** (5 minutes!)
- Node.js event loop handles it in **~1 second**

**Requirements**:
1. Handle thousands of concurrent balance checks
2. Don't block the server while waiting for database
3. Process requests in the order they arrive
4. Handle errors gracefully
5. Demonstrate blocking vs non-blocking difference

---

### 💻 Production-Ready Code Examples

#### Example 1: Blocking vs Non-blocking Comparison

This example demonstrates the **massive performance difference** between blocking and non-blocking code.

```javascript
const http = require('http');
const crypto = require('crypto');

/**
 * Banking API: Blocking vs Non-blocking Demonstration
 * Shows why event loop matters for performance
 */

// ============================================
// BLOCKING VERSION (BAD ❌)
// ============================================

function createBlockingServer() {
  const server = http.createServer((req, res) => {
    const url = new URL(req.url, `http://${req.headers.host}`);
    
    if (url.pathname === '/balance') {
      const accountId = url.searchParams.get('accountId') || 'ACC001';
      
      console.log(`[BLOCKING] Processing request for ${accountId}`);
      const startTime = Date.now();
      
      // Simulate database query with BLOCKING operation
      // This blocks the entire event loop!
      const hash = crypto.pbkdf2Sync('password', 'salt', 100000, 64, 'sha512');
      
      const balance = 5000 + Math.random() * 1000;
      const processingTime = Date.now() - startTime;
      
      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({
        accountId,
        balance: balance.toFixed(2),
        processingTime: processingTime + 'ms',
        version: 'blocking'
      }));
      
      console.log(`[BLOCKING] Request completed in ${processingTime}ms`);
    } else {
      res.writeHead(404);
      res.end('Not found');
    }
  });
  
  return server;
}

// ============================================
// NON-BLOCKING VERSION (GOOD ✅)
// ============================================

function createNonBlockingServer() {
  const server = http.createServer((req, res) => {
    const url = new URL(req.url, `http://${req.headers.host}`);
    
    if (url.pathname === '/balance') {
      const accountId = url.searchParams.get('accountId') || 'ACC001';
      
      console.log(`[NON-BLOCKING] Processing request for ${accountId}`);
      const startTime = Date.now();
      
      // Simulate database query with NON-BLOCKING operation
      // This doesn't block the event loop!
      crypto.pbkdf2('password', 'salt', 100000, 64, 'sha512', (err, hash) => {
        if (err) {
          res.writeHead(500, { 'Content-Type': 'application/json' });
          res.end(JSON.stringify({ error: 'Internal error' }));
          return;
        }
        
        const balance = 5000 + Math.random() * 1000;
        const processingTime = Date.now() - startTime;
        
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({
          accountId,
          balance: balance.toFixed(2),
          processingTime: processingTime + 'ms',
          version: 'non-blocking'
        }));
        
        console.log(`[NON-BLOCKING] Request completed in ${processingTime}ms`);
      });
    } else {
      res.writeHead(404);
      res.end('Not found');
    }
  });
  
  return server;
}

// ============================================
// PERFORMANCE COMPARISON
// ============================================

async function runComparison() {
  console.log('🏦 Banking API: Event Loop Performance Comparison\n');
  console.log('=' .repeat(60));
  
  // Test 1: Blocking Server
  console.log('\n📊 Test 1: BLOCKING Server (BAD ❌)');
  console.log('─'.repeat(60));
  
  const blockingServer = createBlockingServer();
  blockingServer.listen(3000, () => {
    console.log('   Server running on http://localhost:3000');
    console.log('   Send 5 concurrent requests...\n');
  });
  
  // Wait for server to start
  await new Promise(resolve => setTimeout(resolve, 500));
  
  // Send 5 concurrent requests
  const blockingStart = Date.now();
  
  const blockingPromises = [];
  for (let i = 1; i <= 5; i++) {
    blockingPromises.push(
      fetch(`http://localhost:3000/balance?accountId=ACC${i}`)
        .then(r => r.json())
    );
  }
  
  const blockingResults = await Promise.all(blockingPromises);
  const blockingTotal = Date.now() - blockingStart;
  
  console.log('\n   Results:');
  blockingResults.forEach((result, i) => {
    console.log(`   Request ${i + 1}: ${result.processingTime}`);
  });
  console.log(`\n   ⏱️  Total time for 5 requests: ${blockingTotal}ms`);
  console.log(`   ⚠️  Problem: Requests processed SEQUENTIALLY (one at a time)`);
  
  blockingServer.close();
  
  // Wait before next test
  await new Promise(resolve => setTimeout(resolve, 1000));
  
  // Test 2: Non-blocking Server
  console.log('\n\n📊 Test 2: NON-BLOCKING Server (GOOD ✅)');
  console.log('─'.repeat(60));
  
  const nonBlockingServer = createNonBlockingServer();
  nonBlockingServer.listen(3001, () => {
    console.log('   Server running on http://localhost:3001');
    console.log('   Send 5 concurrent requests...\n');
  });
  
  // Wait for server to start
  await new Promise(resolve => setTimeout(resolve, 500));
  
  // Send 5 concurrent requests
  const nonBlockingStart = Date.now();
  
  const nonBlockingPromises = [];
  for (let i = 1; i <= 5; i++) {
    nonBlockingPromises.push(
      fetch(`http://localhost:3001/balance?accountId=ACC${i}`)
        .then(r => r.json())
    );
  }
  
  const nonBlockingResults = await Promise.all(nonBlockingPromises);
  const nonBlockingTotal = Date.now() - nonBlockingStart;
  
  console.log('\n   Results:');
  nonBlockingResults.forEach((result, i) => {
    console.log(`   Request ${i + 1}: ${result.processingTime}`);
  });
  console.log(`\n   ⏱️  Total time for 5 requests: ${nonBlockingTotal}ms`);
  console.log(`   ✅ Benefit: Requests processed CONCURRENTLY (simultaneously)`);
  
  // Performance comparison
  console.log('\n\n📈 Performance Comparison:');
  console.log('=' .repeat(60));
  console.log(`   Blocking:     ${blockingTotal}ms`);
  console.log(`   Non-blocking: ${nonBlockingTotal}ms`);
  console.log(`   Improvement:  ${(blockingTotal / nonBlockingTotal).toFixed(1)}x faster`);
  console.log(`\n   💡 Non-blocking is ~${Math.round((blockingTotal / nonBlockingTotal))}x faster!`);
  
  nonBlockingServer.close();
  
  console.log('\n' + '='.repeat(60));
  console.log('\n✅ Comparison complete!\n');
}

// Run the comparison
runComparison().catch(console.error);

```

**To test this example:**

```bash
# Run the comparison
node Q01_Event_Loop_Basics.js

# You'll see output like:
# Blocking: 2500ms (requests wait for each other)
# Non-blocking: 500ms (requests execute simultaneously)
# Improvement: 5x faster
```

**Expected Output**:
```
🏦 Banking API: Event Loop Performance Comparison

============================================================

📊 Test 1: BLOCKING Server (BAD ❌)
────────────────────────────────────────────────────────────
   Server running on http://localhost:3000
   Send 5 concurrent requests...

[BLOCKING] Processing request for ACC1
[BLOCKING] Request completed in 497ms
[BLOCKING] Processing request for ACC2
[BLOCKING] Request completed in 501ms
[BLOCKING] Processing request for ACC3
[BLOCKING] Request completed in 495ms
[BLOCKING] Processing request for ACC4
[BLOCKING] Request completed in 498ms
[BLOCKING] Processing request for ACC5
[BLOCKING] Request completed in 502ms

   Results:
   Request 1: 497ms
   Request 2: 501ms
   Request 3: 495ms
   Request 4: 498ms
   Request 5: 502ms

   ⏱️  Total time for 5 requests: 2493ms
   ⚠️  Problem: Requests processed SEQUENTIALLY (one at a time)


📊 Test 2: NON-BLOCKING Server (GOOD ✅)
────────────────────────────────────────────────────────────
   Server running on http://localhost:3001
   Send 5 concurrent requests...

[NON-BLOCKING] Processing request for ACC1
[NON-BLOCKING] Processing request for ACC2
[NON-BLOCKING] Processing request for ACC3
[NON-BLOCKING] Processing request for ACC4
[NON-BLOCKING] Processing request for ACC5
[NON-BLOCKING] Request completed in 503ms
[NON-BLOCKING] Request completed in 505ms
[NON-BLOCKING] Request completed in 501ms
[NON-BLOCKING] Request completed in 499ms
[NON-BLOCKING] Request completed in 502ms

   Results:
   Request 1: 503ms
   Request 2: 505ms
   Request 3: 501ms
   Request 4: 499ms
   Request 5: 502ms

   ⏱️  Total time for 5 requests: 505ms
   ✅ Benefit: Requests processed CONCURRENTLY (simultaneously)


📈 Performance Comparison:
============================================================
   Blocking:     2493ms
   Non-blocking: 505ms
   Improvement:  4.9x faster

   💡 Non-blocking is ~5x faster!

============================================================

✅ Comparison complete!
```

**Key Learning**:
- **Blocking**: Requests execute one after another (2493ms total)
- **Non-blocking**: Requests execute simultaneously (505ms total)
- **Result**: 5x performance improvement with event loop!

---

#### Example 2: Visualizing Call Stack and Event Loop

This example shows **exactly how** the call stack and event loop interact, with detailed logging.

```javascript
const http = require('http');

/**
 * Banking Transaction Queue Processor
 * Visualizes call stack and event loop interaction
 */

// Global tracking
let callStackDepth = 0;
const executionLog = [];

// Helper to log execution with call stack depth
function logExecution(message, type = 'SYNC') {
  const indent = '  '.repeat(callStackDepth);
  const timestamp = Date.now();
  const log = `${indent}[${type}] ${message}`;
  
  console.log(log);
  executionLog.push({
    message,
    type,
    depth: callStackDepth,
    timestamp
  });
}

// Helper to track function entry/exit
function trackFunction(name, fn) {
  return function(...args) {
    callStackDepth++;
    logExecution(`→ Entering ${name}()`, 'STACK');
    
    const result = fn.apply(this, args);
    
    logExecution(`← Exiting ${name}()`, 'STACK');
    callStackDepth--;
    
    return result;
  };
}

// ============================================
// Banking Transaction Processing Functions
// ============================================

/**
 * Process a transaction (synchronous)
 */
const processTransaction = trackFunction('processTransaction', function(transactionId) {
  logExecution(`Processing transaction ${transactionId}`, 'SYNC');
  
  // Simulate some synchronous work
  let sum = 0;
  for (let i = 0; i < 1000000; i++) {
    sum += i;
  }
  
  logExecution(`Transaction ${transactionId} validated`, 'SYNC');
  return { transactionId, status: 'validated', sum };
});

/**
 * Query database (asynchronous - goes to callback queue)
 */
function queryDatabase(accountId, callback) {
  logExecution(`Initiating DB query for ${accountId}`, 'ASYNC-START');
  logExecution(`DB query sent to background (libuv)`, 'ASYNC-START');
  
  // Simulate database query (non-blocking)
  setTimeout(() => {
    logExecution(`DB query completed for ${accountId}`, 'CALLBACK');
    logExecution(`Callback added to queue`, 'CALLBACK');
    
    callback(null, {
      accountId,
      balance: 5000 + Math.random() * 1000
    });
  }, 100);
  
  logExecution(`Main thread continues (DB query in background)`, 'ASYNC-START');
}

/**
 * Main handler (demonstrates event loop flow)
 */
const handleBalanceCheck = trackFunction('handleBalanceCheck', function(accountId) {
  logExecution(`Starting balance check for ${accountId}`, 'SYNC');
  
  // Step 1: Synchronous validation
  const validation = processTransaction(`TXN-${accountId}`);
  logExecution(`Validation result: ${validation.status}`, 'SYNC');
  
  // Step 2: Asynchronous database query
  queryDatabase(accountId, (err, result) => {
    callStackDepth++;
    logExecution(`Callback executing for ${accountId}`, 'CALLBACK');
    logExecution(`Balance retrieved: $${result.balance.toFixed(2)}`, 'CALLBACK');
    callStackDepth--;
  });
  
  // Step 3: This executes BEFORE the callback!
  logExecution(`Balance check initiated (waiting for callback)`, 'SYNC');
});

// ============================================
// Demonstration
// ============================================

console.log('\n🏦 Banking API: Call Stack & Event Loop Visualization\n');
console.log('='.repeat(70));
console.log('\nLegend:');
console.log('  [SYNC]         = Synchronous code (executes immediately)');
console.log('  [ASYNC-START]  = Async operation started (background)');
console.log('  [CALLBACK]     = Callback from event loop');
console.log('  [STACK]        = Call stack entry/exit');
console.log('  Indentation    = Call stack depth\n');
console.log('='.repeat(70));
console.log('\n📝 Execution Flow:\n');

// Process multiple balance checks
handleBalanceCheck('ACC001');
handleBalanceCheck('ACC002');

logExecution('All synchronous code executed', 'SYNC');
logExecution('Event loop will handle callbacks when ready', 'SYNC');

// Wait for callbacks to complete
setTimeout(() => {
  console.log('\n' + '='.repeat(70));
  console.log('\n📊 Execution Summary:\n');
  
  const syncOps = executionLog.filter(log => log.type === 'SYNC').length;
  const asyncStarts = executionLog.filter(log => log.type === 'ASYNC-START').length;
  const callbacks = executionLog.filter(log => log.type === 'CALLBACK').length;
  
  console.log(`   Synchronous operations:  ${syncOps}`);
  console.log(`   Async operations started: ${asyncStarts}`);
  console.log(`   Callbacks executed:       ${callbacks}`);
  console.log(`   Total operations:         ${executionLog.length}`);
  
  console.log('\n💡 Key Observations:');
  console.log('   1. All synchronous code executes first (call stack)');
  console.log('   2. Async operations start but don\'t wait (background)');
  console.log('   3. Main thread continues immediately');
  console.log('   4. Callbacks execute later (event loop)');
  console.log('   5. Multiple async operations can run simultaneously\n');
  
  console.log('='.repeat(70) + '\n');
}, 500);

```

**To test this example:**

```bash
# Run the visualization
node event-loop-visualization.js
```

**Expected Output**:

```
🏦 Banking API: Call Stack & Event Loop Visualization

======================================================================

Legend:
  [SYNC]         = Synchronous code (executes immediately)
  [ASYNC-START]  = Async operation started (background)
  [CALLBACK]     = Callback from event loop
  [STACK]        = Call stack entry/exit
  Indentation    = Call stack depth

======================================================================

📝 Execution Flow:

[STACK] → Entering handleBalanceCheck()
  [SYNC] Starting balance check for ACC001
  [STACK] → Entering processTransaction()
    [SYNC] Processing transaction TXN-ACC001
    [SYNC] Transaction TXN-ACC001 validated
  [STACK] ← Exiting processTransaction()
  [SYNC] Validation result: validated
  [ASYNC-START] Initiating DB query for ACC001
  [ASYNC-START] DB query sent to background (libuv)
  [ASYNC-START] Main thread continues (DB query in background)
  [SYNC] Balance check initiated (waiting for callback)
[STACK] ← Exiting handleBalanceCheck()
[STACK] → Entering handleBalanceCheck()
  [SYNC] Starting balance check for ACC002
  [STACK] → Entering processTransaction()
    [SYNC] Processing transaction TXN-ACC002
    [SYNC] Transaction TXN-ACC002 validated
  [STACK] ← Exiting processTransaction()
  [SYNC] Validation result: validated
  [ASYNC-START] Initiating DB query for ACC002
  [ASYNC-START] DB query sent to background (libuv)
  [ASYNC-START] Main thread continues (DB query in background)
  [SYNC] Balance check initiated (waiting for callback)
[STACK] ← Exiting handleBalanceCheck()
[SYNC] All synchronous code executed
[SYNC] Event loop will handle callbacks when ready

[CALLBACK] DB query completed for ACC001
[CALLBACK] Callback added to queue
  [CALLBACK] Callback executing for ACC001
  [CALLBACK] Balance retrieved: $5482.35

[CALLBACK] DB query completed for ACC002
[CALLBACK] Callback added to queue
  [CALLBACK] Callback executing for ACC002
  [CALLBACK] Balance retrieved: $5723.18

======================================================================

📊 Execution Summary:

   Synchronous operations:  12
   Async operations started: 6
   Callbacks executed:       6
   Total operations:         24

💡 Key Observations:
   1. All synchronous code executes first (call stack)
   2. Async operations start but don't wait (background)
   3. Main thread continues immediately
   4. Callbacks execute later (event loop)
   5. Multiple async operations can run simultaneously

======================================================================
```

**What This Demonstrates**:

1. **Call Stack Depth**: See how functions nest and return
2. **Execution Order**: Sync code → Async operations → Callbacks
3. **Non-blocking Nature**: Main thread doesn't wait for DB queries
4. **Concurrent Processing**: Both accounts processed simultaneously

---

#### Example 3: Real Banking Transaction Queue

This example shows a **production-ready transaction processor** using the event loop effectively.

```javascript
const http = require('http');
const { EventEmitter } = require('events');

/**
 * Banking Transaction Queue Processor
 * Production-ready example using event loop efficiently
 */

class TransactionProcessor extends EventEmitter {
  constructor(options = {}) {
    super();
    
    this.maxConcurrent = options.maxConcurrent || 10;
    this.processingDelay = options.processingDelay || 50;
    
    this.queue = [];
    this.processing = new Map();
    this.completed = [];
    this.failed = [];
    
    this.stats = {
      received: 0,
      processed: 0,
      failed: 0,
      avgProcessingTime: 0
    };
  }
  
  /**
   * Add transaction to queue
   */
  enqueue(transaction) {
    this.stats.received++;
    
    transaction.id = transaction.id || `TXN${Date.now()}-${this.stats.received}`;
    transaction.receivedAt = Date.now();
    transaction.status = 'queued';
    
    this.queue.push(transaction);
    
    console.log(`📥 Transaction ${transaction.id} queued (Queue size: ${this.queue.length})`);
    
    // Trigger processing (non-blocking)
    setImmediate(() => this.processQueue());
    
    return transaction.id;
  }
  
  /**
   * Process queue (event loop friendly)
   */
  async processQueue() {
    // Check if we can process more
    while (
      this.queue.length > 0 && 
      this.processing.size < this.maxConcurrent
    ) {
      const transaction = this.queue.shift();
      
      // Process asynchronously (non-blocking)
      this.processTransaction(transaction);
    }
  }
  
  /**
   * Process single transaction (async)
   */
  async processTransaction(transaction) {
    transaction.status = 'processing';
    transaction.startedAt = Date.now();
    
    this.processing.set(transaction.id, transaction);
    
    console.log(`⚙️  Processing ${transaction.id} (${this.processing.size} concurrent)`);
    
    try {
      // Simulate async operations (database, external APIs, etc.)
      await this.validateAccount(transaction);
      await this.checkFraud(transaction);
      await this.updateBalance(transaction);
      await this.notifyCustomer(transaction);
      
      // Success
      transaction.status = 'completed';
      transaction.completedAt = Date.now();
      transaction.processingTime = transaction.completedAt - transaction.startedAt;
      
      this.processing.delete(transaction.id);
      this.completed.push(transaction);
      this.stats.processed++;
      
      // Update average processing time
      const totalTime = this.completed.reduce((sum, txn) => sum + txn.processingTime, 0);
      this.stats.avgProcessingTime = Math.round(totalTime / this.completed.length);
      
      console.log(`✅ Transaction ${transaction.id} completed (${transaction.processingTime}ms)`);
      
      this.emit('transaction:completed', transaction);
      
      // Continue processing queue
      setImmediate(() => this.processQueue());
      
    } catch (error) {
      // Failure
      transaction.status = 'failed';
      transaction.error = error.message;
      transaction.failedAt = Date.now();
      
      this.processing.delete(transaction.id);
      this.failed.push(transaction);
      this.stats.failed++;
      
      console.log(`❌ Transaction ${transaction.id} failed: ${error.message}`);
      
      this.emit('transaction:failed', transaction);
      
      // Continue processing queue
      setImmediate(() => this.processQueue());
    }
  }
  
  /**
   * Simulate async operations
   */
  async validateAccount(transaction) {
    await this.delay(10);
    
    if (!transaction.fromAccount || !transaction.toAccount) {
      throw new Error('Invalid account');
    }
    
    console.log(`   ✓ Validated accounts for ${transaction.id}`);
  }
  
  async checkFraud(transaction) {
    await this.delay(20);
    
    // Simulate fraud detection
    if (transaction.amount > 50000) {
      // Flag for manual review (don't fail, just slow down)
      console.log(`   ⚠️  Fraud check triggered for ${transaction.id} (high amount)`);
      await this.delay(50);
    }
    
    console.log(`   ✓ Fraud check passed for ${transaction.id}`);
  }
  
  async updateBalance(transaction) {
    await this.delay(30);
    console.log(`   ✓ Balance updated for ${transaction.id}`);
  }
  
  async notifyCustomer(transaction) {
    await this.delay(15);
    console.log(`   ✓ Customer notified for ${transaction.id}`);
  }
  
  /**
   * Delay helper (non-blocking)
   */
  delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
  
  /**
   * Get current stats
   */
  getStats() {
    return {
      ...this.stats,
      queueSize: this.queue.length,
      processing: this.processing.size,
      completed: this.completed.length,
      failed: this.failed.length
    };
  }
}

// ============================================
// HTTP Server
// ============================================

const processor = new TransactionProcessor({
  maxConcurrent: 5,  // Process max 5 transactions simultaneously
  processingDelay: 50
});

// Listen to events
processor.on('transaction:completed', (txn) => {
  // Could trigger notifications, logging, etc.
});

processor.on('transaction:failed', (txn) => {
  // Could trigger alerts, retry logic, etc.
});

// Create HTTP server
const server = http.createServer(async (req, res) => {
  const url = new URL(req.url, `http://${req.headers.host}`);
  
  try {
    // Route: Submit transaction
    if (url.pathname === '/api/transfer' && req.method === 'POST') {
      let body = '';
      req.on('data', chunk => body += chunk);
      
      await new Promise(resolve => req.on('end', resolve));
      
      const transaction = JSON.parse(body);
      
      // Enqueue transaction (non-blocking)
      const transactionId = processor.enqueue(transaction);
      
      res.writeHead(202, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({
        success: true,
        transactionId,
        status: 'queued',
        message: 'Transaction queued for processing'
      }));
      
    // Route: Get stats
    } else if (url.pathname === '/api/stats' && req.method === 'GET') {
      const stats = processor.getStats();
      
      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({
        success: true,
        stats
      }));
      
    // Route: Health check
    } else if (url.pathname === '/health') {
      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({
        status: 'healthy',
        uptime: process.uptime(),
        memoryUsage: process.memoryUsage().heapUsed / 1024 / 1024 + ' MB'
      }));
      
    } else {
      res.writeHead(404);
      res.end('Not found');
    }
    
  } catch (error) {
    console.error('API Error:', error.message);
    res.writeHead(500, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ error: error.message }));
  }
});

server.listen(3000, () => {
  console.log('\n🏦 Banking Transaction Processor\n');
  console.log('   Server: http://localhost:3000');
  console.log('   Max Concurrent: 5 transactions');
  console.log('\nEndpoints:');
  console.log('   POST /api/transfer - Submit transaction');
  console.log('   GET  /api/stats - View statistics');
  console.log('   GET  /health - Health check\n');
  console.log('=' .repeat(70) + '\n');
});

```

**To test this example:**

```bash
# Start the server
node transaction-queue.js

# In another terminal, send multiple transactions:

# Transaction 1
curl -X POST http://localhost:3000/api/transfer \
  -H "Content-Type: application/json" \
  -d '{"fromAccount":"ACC001","toAccount":"ACC002","amount":1000}'

# Transaction 2
curl -X POST http://localhost:3000/api/transfer \
  -H "Content-Type: application/json" \
  -d '{"fromAccount":"ACC003","toAccount":"ACC004","amount":2000}'

# Transaction 3 (high amount - triggers fraud check)
curl -X POST http://localhost:3000/api/transfer \
  -H "Content-Type: application/json" \
  -d '{"fromAccount":"ACC005","toAccount":"ACC006","amount":60000}'

# Send 10 transactions at once (demonstrates concurrent processing)
for i in {1..10}; do
  curl -X POST http://localhost:3000/api/transfer \
    -H "Content-Type: application/json" \
    -d "{\"fromAccount\":\"ACC$i\",\"toAccount\":\"ACC$((i+1))\",\"amount\":$((i*100))}" &
done

# Check statistics
curl http://localhost:3000/api/stats
```

**Expected Output**:

```
🏦 Banking Transaction Processor

   Server: http://localhost:3000
   Max Concurrent: 5 transactions

Endpoints:
   POST /api/transfer - Submit transaction
   GET  /api/stats - View statistics
   GET  /health - Health check

======================================================================

📥 Transaction TXN1699900000001-1 queued (Queue size: 1)
⚙️  Processing TXN1699900000001-1 (1 concurrent)
   ✓ Validated accounts for TXN1699900000001-1
   ✓ Fraud check passed for TXN1699900000001-1
   ✓ Balance updated for TXN1699900000001-1
   ✓ Customer notified for TXN1699900000001-1
✅ Transaction TXN1699900000001-1 completed (75ms)

📥 Transaction TXN1699900000002-2 queued (Queue size: 1)
⚙️  Processing TXN1699900000002-2 (1 concurrent)
📥 Transaction TXN1699900000003-3 queued (Queue size: 1)
⚙️  Processing TXN1699900000003-3 (2 concurrent)
   ✓ Validated accounts for TXN1699900000002-2
   ✓ Validated accounts for TXN1699900000003-3
   ✓ Fraud check passed for TXN1699900000002-2
   ✓ Fraud check passed for TXN1699900000003-3
   ⚠️  Fraud check triggered for TXN1699900000003-3 (high amount)
   ✓ Balance updated for TXN1699900000002-2
   ✓ Customer notified for TXN1699900000002-2
✅ Transaction TXN1699900000002-2 completed (78ms)
   ✓ Balance updated for TXN1699900000003-3
   ✓ Customer notified for TXN1699900000003-3
✅ Transaction TXN1699900000003-3 completed (135ms)
```

**Key Features**:

1. **Non-blocking Queue**: Transactions queued instantly
2. **Concurrent Processing**: Up to 5 transactions processed simultaneously
3. **Event Loop Friendly**: Uses async/await and setTimeout
4. **Production Ready**: Error handling, stats, monitoring
5. **Scalable**: Can handle thousands of req/sec

---

### ✅ DO's and ❌ DON'Ts

#### ✅ DO's:

1. **DO** use asynchronous operations for I/O (database, files, network)
   ```javascript
   // Good ✅
   fs.readFile('file.txt', (err, data) => { /* ... */ });
   ```

2. **DO** use Promises and async/await for cleaner async code
   ```javascript
   // Good ✅
   const data = await fs.promises.readFile('file.txt');
   ```

3. **DO** understand that the event loop is single-threaded
   - Only one piece of code executes at a time
   - I/O operations run in background (libuv)

4. **DO** use `setImmediate()` for deferring execution to next event loop iteration
   ```javascript
   // Good ✅
   setImmediate(() => {
     // Executes after I/O events
   });
   ```

5. **DO** keep event loop responsive by avoiding heavy synchronous operations
   ```javascript
   // Good ✅ - Break up work
   async function processLargeArray(items) {
     for (let i = 0; i < items.length; i++) {
       await processItem(items[i]);
       if (i % 100 === 0) {
         await new Promise(resolve => setImmediate(resolve)); // Yield to event loop
       }
     }
   }
   ```

6. **DO** use Worker Threads for CPU-intensive tasks
   ```javascript
   // Good ✅
   const { Worker } = require('worker_threads');
   const worker = new Worker('./cpu-intensive-task.js');
   ```

7. **DO** monitor event loop lag in production
   ```javascript
   // Good ✅
   const start = Date.now();
   setImmediate(() => {
     const lag = Date.now() - start;
     if (lag > 100) console.warn('Event loop lag:', lag);
   });
   ```

8. **DO** use streaming for large data processing
   ```javascript
   // Good ✅
   const readStream = fs.createReadStream('large-file.txt');
   readStream.pipe(processStream).pipe(writeStream);
   ```

9. **DO** understand callback queue vs microtask queue
   - Promises/microtasks execute before callbacks
   - `process.nextTick()` executes before both

10. **DO** handle errors in async operations
    ```javascript
    // Good ✅
    try {
      await riskyOperation();
    } catch (error) {
      console.error('Operation failed:', error);
    }
    ```

#### ❌ DON'Ts:

1. **DON'T** use synchronous operations in production code
   ```javascript
   // Bad ❌
   const data = fs.readFileSync('file.txt'); // Blocks event loop!
   ```

2. **DON'T** perform CPU-intensive operations in the main thread
   ```javascript
   // Bad ❌
   function calculateFibonacci(n) {
     if (n <= 1) return n;
     return calculateFibonacci(n - 1) + calculateFibonacci(n - 2); // Blocks!
   }
   ```

3. **DON'T** forget that callbacks execute AFTER synchronous code
   ```javascript
   // Bad ❌ - Misunderstanding async
   let result;
   setTimeout(() => {
     result = 'data';
   }, 0);
   console.log(result); // undefined! Callback not executed yet
   ```

4. **DON'T** use blocking operations in HTTP request handlers
   ```javascript
   // Bad ❌
   app.get('/data', (req, res) => {
     const data = fs.readFileSync('large-file.txt'); // Blocks all other requests!
     res.send(data);
   });
   ```

5. **DON'T** use `process.nextTick()` for regular async operations
   ```javascript
   // Bad ❌ - Can starve event loop
   function recursiveNextTick() {
     process.nextTick(recursiveNextTick); // Infinite loop!
   }
   ```

6. **DON'T** mix callback styles without proper error handling
   ```javascript
   // Bad ❌
   doSomething((err, result) => {
     // Forgot to check err!
     console.log(result.data); // Could crash if err exists
   });
   ```

7. **DON'T** assume code executes in the order it's written with async
   ```javascript
   // Bad ❌
   fs.readFile('file1.txt', callback1);
   fs.readFile('file2.txt', callback2);
   // callback2 might execute before callback1!
   ```

8. **DON'T** use long-running loops without yielding to event loop
   ```javascript
   // Bad ❌
   for (let i = 0; i < 1000000; i++) {
     processItem(i); // Blocks event loop for entire duration
   }
   ```

9. **DON'T** forget that event loop is shared across all requests
   ```javascript
   // Bad ❌
   app.get('/slow', (req, res) => {
     // This blocks ALL other requests too!
     for (let i = 0; i < 10000000000; i++) {}
     res.send('done');
   });
   ```

10. **DON'T** ignore event loop monitoring in production
    - Event loop lag indicates performance problems
    - Monitor with tools like `clinic.js` or New Relic

---

### 🎯 Key Takeaways

#### **What is the Event Loop?**
The event loop is Node.js's mechanism for handling asynchronous operations in a single-threaded environment. It continuously checks if the call stack is empty and if there are callbacks waiting in the queue.

#### **How Does It Work?**

```
1. Execute synchronous code (call stack)
2. When async operation starts → send to background (libuv)
3. Continue executing remaining synchronous code
4. When async operation completes → callback added to queue
5. When call stack is empty → event loop moves callback to call stack
6. Callback executes
7. Repeat
```

#### **Architecture Summary**:

```
┌─────────────────────────────────────────────────────┐
│               Node.js Application                    │
├─────────────────────────────────────────────────────┤
│                                                      │
│  ┌────────────┐      ┌──────────────────────────┐  │
│  │ Call Stack │ ←──→ │     Event Loop (libuv)   │  │
│  └────────────┘      └──────────────────────────┘  │
│        ↕                          ↕                 │
│  ┌────────────┐      ┌──────────────────────────┐  │
│  │ Callback   │      │   Thread Pool (libuv)    │  │
│  │ Queue      │      │   • File I/O              │  │
│  └────────────┘      │   • DNS lookups           │  │
│                      │   • Crypto operations     │  │
│                      └──────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

#### **Why It Matters for Banking**:

1. **High Throughput**: Handle 10,000+ requests/sec with single thread
2. **Low Latency**: No waiting for blocking I/O operations
3. **Resource Efficient**: No thread-per-request overhead
4. **Scalable**: Can handle millions of concurrent connections
5. **Cost Effective**: Fewer servers needed for same load

#### **Performance Impact**:

| Operation Type | Blocking | Non-blocking | Improvement |
|---------------|----------|--------------|-------------|
| File read (10KB) | ~5ms | ~0.01ms | 500x faster |
| Database query | ~50ms | ~0.01ms | 5000x faster |
| HTTP request | ~100ms | ~0.01ms | 10000x faster |
| 1000 concurrent ops | 50s | 0.1s | 500x faster |

#### **Best Practices**:

✅ **Use async/await** for all I/O operations  
✅ **Monitor event loop lag** in production  
✅ **Break up CPU-intensive work** or use Worker Threads  
✅ **Stream large datasets** instead of loading into memory  
✅ **Handle errors** in all async operations  

❌ **Never use `*Sync` methods** in production request handlers  
❌ **Never block the event loop** with long computations  
❌ **Never assume execution order** with async operations  

#### **Real-World Banking Example**:

A typical banking API handling balance checks:

**Without Event Loop (Multi-threaded)**:
- 1000 threads for 1000 concurrent users
- 8GB RAM consumed (8MB per thread)
- Context switching overhead
- ~50-100 requests/sec per core

**With Event Loop (Single-threaded)**:
- 1 thread for 1000 concurrent users
- 50MB RAM consumed
- No context switching
- ~5,000-10,000 requests/sec per core

**Result**: **50-100x better performance** with event loop!

---

### 📚 Further Reading

- **Official Node.js Docs**: [The Node.js Event Loop](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
- **libuv Documentation**: [Design Overview](http://docs.libuv.org/en/v1.x/design.html)
- **Don't Block the Event Loop**: [Node.js Best Practices](https://nodejs.org/en/docs/guides/dont-block-the-event-loop/)

---

### 🏁 Summary

**Event Loop** is the heart of Node.js's asynchronous, non-blocking architecture. It enables a single-threaded application to handle thousands of concurrent operations efficiently by:

1. **Executing synchronous code** on the call stack
2. **Delegating I/O operations** to the background (libuv)
3. **Continuing execution** without waiting
4. **Processing callbacks** when I/O completes
5. **Repeating** this cycle continuously

For banking applications, this means:
- ✅ Handle 10,000+ users simultaneously
- ✅ Process transactions without blocking
- ✅ Respond to balance checks in milliseconds
- ✅ Scale efficiently with minimal resources

**Key Principle**: *Never block the event loop!*

---

**File Status**: ✅ Complete  
**Next Topic**: Q02 - Event Loop Phases (6 phases in detail)

