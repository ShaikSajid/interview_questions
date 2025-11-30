# Topic 1: Node.js Core & Advanced Concepts

**Target Role:** GenAI – NodeJS Engineer (ENBD)  
**Experience Required:** 7+ years  
**Focus Areas:** Async programming, Event-driven architecture, Performance, Scalability

---

## 📚 Table of Contents

1. [Event Loop & Asynchronous Programming](#section-1-event-loop--asynchronous-programming) (Q1-Q10)
2. [Promises, Async/Await, and Error Handling](#section-2-promises-asyncawait-and-error-handling) (Q11-Q20)
3. [Streams and Buffers](#section-3-streams-and-buffers) (Q21-Q30)
4. [Performance Optimization](#section-4-performance-optimization) (Q31-Q40)
5. [Scalability & Architecture Patterns](#section-5-scalability--architecture-patterns) (Q41-Q50)

---

## Section 1: Event Loop & Asynchronous Programming

### Questions 1-10

---

### Q1. Explain the Node.js Event Loop in detail. How does it handle asynchronous operations?

**Answer:**

The Node.js Event Loop is the mechanism that allows Node.js to perform non-blocking I/O operations despite JavaScript being single-threaded. It works through multiple phases:

**Event Loop Phases:**
1. **Timers** - Executes callbacks scheduled by `setTimeout()` and `setInterval()`
2. **Pending Callbacks** - Executes I/O callbacks deferred to the next loop iteration
3. **Idle, Prepare** - Internal use only
4. **Poll** - Retrieves new I/O events; executes I/O callbacks
5. **Check** - `setImmediate()` callbacks are invoked here
6. **Close Callbacks** - Close event callbacks (e.g., `socket.on('close')`)

**Example:**
```javascript
console.log('1: Start');

setTimeout(() => {
  console.log('2: setTimeout');
}, 0);

setImmediate(() => {
  console.log('3: setImmediate');
});

process.nextTick(() => {
  console.log('4: nextTick');
});

Promise.resolve().then(() => {
  console.log('5: Promise');
});

console.log('6: End');

// Output:
// 1: Start
// 6: End
// 4: nextTick
// 5: Promise
// 3: setImmediate
// 2: setTimeout
```

**Key Points:**
- `process.nextTick()` and Promise callbacks execute before any phase
- `nextTick` queue has priority over microtask queue (Promises)
- Understanding this is critical for banking applications where order of operations matters

---

### Q2. What is the difference between `process.nextTick()`, `setImmediate()`, and `setTimeout()`?

**Answer:**

These three methods schedule callbacks differently in the event loop:

| Method | Execution Timing | Use Case |
|--------|------------------|----------|
| `process.nextTick()` | Before any phase of event loop | High priority operations |
| `setImmediate()` | Check phase of event loop | I/O operations |
| `setTimeout(fn, 0)` | Timer phase of next loop iteration | Delayed execution |

**Practical Example:**
```javascript
const fs = require('fs');

// Banking transaction scenario
function processTransaction(transactionId) {
  console.log('Processing transaction:', transactionId);
  
  // Immediate validation (highest priority)
  process.nextTick(() => {
    console.log('Validating transaction:', transactionId);
  });
  
  // Database write (after I/O)
  setImmediate(() => {
    console.log('Writing to database:', transactionId);
  });
  
  // Audit log (delayed)
  setTimeout(() => {
    console.log('Audit log created:', transactionId);
  }, 0);
}

processTransaction('TXN-12345');

// Output order:
// Processing transaction: TXN-12345
// Validating transaction: TXN-12345
// Writing to database: TXN-12345
// Audit log created: TXN-12345
```

**Banking Scenario:**
In ENBD's payment processing, you might use:
- `process.nextTick()` for fraud detection checks (immediate)
- `setImmediate()` for transaction processing
- `setTimeout()` for retry mechanisms

---

### Q3. Explain callback hell and how you would refactor it. Provide a banking-related example.

**Answer:**

**Callback Hell** (Pyramid of Doom) occurs when callbacks are nested within callbacks, making code hard to read and maintain.

**Bad Example (Callback Hell):**
```javascript
// Banking: Process loan application
function processLoanApplication(customerId, callback) {
  validateCustomer(customerId, (err, customer) => {
    if (err) return callback(err);
    
    checkCreditScore(customer.id, (err, score) => {
      if (err) return callback(err);
      
      verifyIncome(customer.id, (err, income) => {
        if (err) return callback(err);
        
        calculateEligibility(score, income, (err, eligible) => {
          if (err) return callback(err);
          
          if (eligible) {
            createLoanOffer(customer.id, (err, offer) => {
              if (err) return callback(err);
              
              sendNotification(customer.email, offer, (err) => {
                if (err) return callback(err);
                callback(null, offer);
              });
            });
          } else {
            callback(new Error('Not eligible'));
          }
        });
      });
    });
  });
}
```

**Good Solution 1: Using Promises:**
```javascript
function processLoanApplication(customerId) {
  return validateCustomer(customerId)
    .then(customer => checkCreditScore(customer.id))
    .then(score => verifyIncome(customerId).then(income => ({ score, income })))
    .then(({ score, income }) => calculateEligibility(score, income))
    .then(eligible => {
      if (!eligible) throw new Error('Not eligible');
      return createLoanOffer(customerId);
    })
    .then(offer => sendNotification(customer.email, offer).then(() => offer))
    .catch(err => {
      console.error('Loan processing failed:', err);
      throw err;
    });
}
```

**Best Solution: Using Async/Await:**
```javascript
async function processLoanApplication(customerId) {
  try {
    const customer = await validateCustomer(customerId);
    const score = await checkCreditScore(customer.id);
    const income = await verifyIncome(customer.id);
    const eligible = await calculateEligibility(score, income);
    
    if (!eligible) {
      throw new Error('Customer not eligible for loan');
    }
    
    const offer = await createLoanOffer(customer.id);
    await sendNotification(customer.email, offer);
    
    return offer;
  } catch (error) {
    console.error('Loan processing failed:', error);
    throw error;
  }
}
```

**Key Improvements:**
- Readable, linear flow
- Centralized error handling
- Easy to maintain and test
- Better debugging experience

---

### Q4. How does Node.js handle concurrency despite being single-threaded? Explain with a real-world banking scenario.

**Answer:**

Node.js achieves concurrency through:
1. **Event-driven architecture**
2. **Non-blocking I/O operations**
3. **libuv thread pool** for heavy operations
4. **Asynchronous callbacks**

**Architecture:**
```
┌─────────────────────────────┐
│   JavaScript (Single Thread) │
│   Your Application Code      │
└─────────────┬───────────────┘
              │
┌─────────────▼───────────────┐
│   Event Loop (libuv)         │
│   - Handles async operations │
└─────────────┬───────────────┘
              │
┌─────────────▼───────────────┐
│   Thread Pool (4-128 threads)│
│   - File I/O                 │
│   - DNS lookup               │
│   - Crypto operations        │
└──────────────────────────────┘
```

**Banking Scenario: Concurrent Transaction Processing**
```javascript
const crypto = require('crypto');
const { promisify } = require('util');
const pbkdf2 = promisify(crypto.pbkdf2);

// Process 10,000 transactions concurrently
async function processTransactions() {
  const startTime = Date.now();
  const transactions = Array.from({ length: 10000 }, (_, i) => ({
    id: `TXN-${i}`,
    amount: Math.random() * 1000,
    accountFrom: `ACC-${i}`,
    accountTo: `ACC-${i + 1}`
  }));
  
  // All transactions processed concurrently
  const results = await Promise.all(
    transactions.map(async (txn) => {
      // Validate transaction (CPU-bound - uses thread pool)
      const hash = await pbkdf2(txn.id, 'salt', 100000, 64, 'sha512');
      
      // Database operations (I/O - non-blocking)
      await updateAccount(txn.accountFrom, -txn.amount);
      await updateAccount(txn.accountTo, txn.amount);
      
      return {
        id: txn.id,
        status: 'completed',
        hash: hash.toString('hex')
      };
    })
  );
  
  const duration = Date.now() - startTime;
  console.log(`Processed ${results.length} transactions in ${duration}ms`);
  return results;
}

async function updateAccount(accountId, amount) {
  // Simulated database call (non-blocking I/O)
  return new Promise(resolve => {
    setImmediate(() => resolve({ accountId, newBalance: amount }));
  });
}

// Usage
processTransactions()
  .then(results => console.log('All transactions completed'))
  .catch(err => console.error('Transaction processing failed:', err));
```

**Key Points for ENBD:**
- Can handle 10,000+ concurrent requests with single thread
- I/O operations don't block other transactions
- Critical for high-volume banking operations
- Need to monitor thread pool size for optimal performance

---

### Q5. What are Worker Threads in Node.js and when would you use them in a banking application?

**Answer:**

**Worker Threads** allow running JavaScript in parallel threads, useful for CPU-intensive tasks that would otherwise block the event loop.

**When to Use in Banking:**
- Complex financial calculations
- Large data processing (report generation)
- Encryption/decryption operations
- Risk analysis algorithms

**Example: Parallel Portfolio Risk Calculation**
```javascript
// main.js
const { Worker } = require('worker_threads');
const path = require('path');

async function calculatePortfolioRisk(portfolios) {
  const workerPromises = portfolios.map(portfolio => {
    return new Promise((resolve, reject) => {
      const worker = new Worker(path.join(__dirname, 'risk-calculator.js'), {
        workerData: portfolio
      });
      
      worker.on('message', resolve);
      worker.on('error', reject);
      worker.on('exit', (code) => {
        if (code !== 0) {
          reject(new Error(`Worker stopped with exit code ${code}`));
        }
      });
    });
  });
  
  return Promise.all(workerPromises);
}

// Process 1000 customer portfolios in parallel
const portfolios = Array.from({ length: 1000 }, (_, i) => ({
  customerId: `CUST-${i}`,
  holdings: generateRandomHoldings(),
  marketData: getMarketData()
}));

calculatePortfolioRisk(portfolios)
  .then(results => {
    console.log('Risk analysis completed for all portfolios');
    results.forEach(result => {
      console.log(`Customer: ${result.customerId}, Risk Score: ${result.riskScore}`);
    });
  })
  .catch(err => console.error('Risk calculation failed:', err));
```

```javascript
// risk-calculator.js (Worker Thread)
const { parentPort, workerData } = require('worker_threads');

function calculateRisk(portfolio) {
  // CPU-intensive calculation
  let riskScore = 0;
  
  for (const holding of portfolio.holdings) {
    // Complex risk calculation
    const volatility = calculateVolatility(holding);
    const beta = calculateBeta(holding);
    const var95 = calculateVaR(holding, 0.95);
    
    riskScore += (volatility * 0.4) + (beta * 0.3) + (var95 * 0.3);
  }
  
  return {
    customerId: portfolio.customerId,
    riskScore: riskScore,
    riskLevel: getRiskLevel(riskScore),
    timestamp: new Date().toISOString()
  };
}

function calculateVolatility(holding) {
  // Intensive calculation
  let sum = 0;
  for (let i = 0; i < 1000000; i++) {
    sum += Math.sqrt(holding.value * Math.random());
  }
  return sum / 1000000;
}

function calculateBeta(holding) {
  // Market correlation calculation
  return Math.random() * 2;
}

function calculateVaR(holding, confidence) {
  // Value at Risk calculation
  return holding.value * 0.05 * (1 - confidence);
}

function getRiskLevel(score) {
  if (score < 30) return 'LOW';
  if (score < 60) return 'MEDIUM';
  return 'HIGH';
}

// Send result back to main thread
parentPort.postMessage(calculateRisk(workerData));
```

**Key Benefits for ENBD:**
- Doesn't block main event loop
- Utilizes multi-core processors
- Better performance for CPU-intensive tasks
- Maintains responsiveness for other requests

**When NOT to Use:**
- Simple I/O operations (use async/await)
- Small calculations (overhead not worth it)
- Already have external microservices for heavy processing

---

### Q6. Explain the differences between `setImmediate()` vs `process.nextTick()` in an I/O context. Provide a file system example.

**Answer:**

**Key Differences:**

| Aspect | `process.nextTick()` | `setImmediate()` |
|--------|---------------------|------------------|
| Execution | Before any I/O events | After I/O events in Check phase |
| Priority | Higher (can cause starvation) | Lower (allows I/O) |
| Use Case | Critical operations | Non-blocking I/O callbacks |

**File System Example: Banking Document Processing**
```javascript
const fs = require('fs');

function processCustomerDocuments(customerId) {
  console.log('1. Starting document processing for:', customerId);
  
  // Read customer data file
  fs.readFile(`./customers/${customerId}.json`, 'utf8', (err, data) => {
    if (err) {
      console.error('Error reading file:', err);
      return;
    }
    
    console.log('2. File read completed');
    
    // Using process.nextTick - executes before I/O polling
    process.nextTick(() => {
      console.log('3. Immediate validation (nextTick) - High Priority');
      validateDocument(data);
    });
    
    // Using setImmediate - executes in Check phase after I/O
    setImmediate(() => {
      console.log('4. Processing document (setImmediate) - After I/O');
      parseAndStore(data);
    });
    
    // Another nextTick
    process.nextTick(() => {
      console.log('5. Audit log (nextTick) - High Priority');
      logAuditTrail(customerId);
    });
    
    console.log('6. Scheduling background tasks');
  });
  
  console.log('7. Main thread continues...');
}

// Output order:
// 1. Starting document processing for: CUST-12345
// 7. Main thread continues...
// 2. File read completed
// 6. Scheduling background tasks
// 3. Immediate validation (nextTick) - High Priority
// 5. Audit log (nextTick) - High Priority
// 4. Processing document (setImmediate) - After I/O

processCustomerDocuments('CUST-12345');
```

**Real-World Banking Scenario:**
```javascript
const fs = require('fs').promises;

async function processDailyTransactionReport() {
  try {
    // Read large transaction file
    const fileHandle = await fs.open('./reports/transactions.csv', 'r');
    const stream = fileHandle.createReadStream();
    
    let lineCount = 0;
    let buffer = '';
    
    stream.on('data', (chunk) => {
      buffer += chunk.toString();
      
      // Critical: Validate data integrity immediately
      process.nextTick(() => {
        // This runs before processing more chunks
        if (buffer.includes('ERROR')) {
          console.error('Data corruption detected!');
          stream.pause();
          // Immediate alert to operations team
        }
      });
      
      // Non-critical: Process the data after I/O
      setImmediate(() => {
        // This allows reading more chunks first
        const lines = buffer.split('\n');
        buffer = lines.pop(); // Keep incomplete line
        lineCount += lines.length;
        
        lines.forEach(line => processTransaction(line));
      });
    });
    
    stream.on('end', () => {
      console.log(`Processed ${lineCount} transactions`);
    });
    
  } catch (error) {
    console.error('Report processing failed:', error);
  }
}

function processTransaction(line) {
  // Parse and validate transaction
  const txn = parseTransactionLine(line);
  // Store in database
}

function parseTransactionLine(line) {
  const parts = line.split(',');
  return {
    id: parts[0],
    amount: parseFloat(parts[1]),
    timestamp: parts[2]
  };
}
```

**Best Practices for ENBD:**
- Use `process.nextTick()` for critical validations
- Use `setImmediate()` for non-blocking processing
- Avoid recursive `nextTick()` (can starve event loop)
- Consider Worker Threads for heavy processing

---

### Q7. How would you prevent blocking the event loop in Node.js? Provide examples with CPU-intensive operations.

**Answer:**

**Strategies to Prevent Blocking:**

1. **Break up long-running tasks**
2. **Use Worker Threads for CPU-intensive operations**
3. **Leverage async operations**
4. **Use streams for large data**
5. **Implement timeouts and limits**

**Example 1: Breaking Up CPU-Intensive Tasks**
```javascript
// BAD: Blocks event loop
function calculateInterestBad(accounts) {
  const results = [];
  
  // Processes 1 million accounts synchronously - BLOCKS!
  for (let i = 0; i < accounts.length; i++) {
    const interest = calculateCompoundInterest(
      accounts[i].balance,
      accounts[i].rate,
      365
    );
    results.push({ accountId: accounts[i].id, interest });
  }
  
  return results;
}

// GOOD: Non-blocking with batching
async function calculateInterestGood(accounts, batchSize = 1000) {
  const results = [];
  
  for (let i = 0; i < accounts.length; i += batchSize) {
    const batch = accounts.slice(i, i + batchSize);
    
    // Process batch
    const batchResults = batch.map(account => ({
      accountId: account.id,
      interest: calculateCompoundInterest(
        account.balance,
        account.rate,
        365
      )
    }));
    
    results.push(...batchResults);
    
    // Yield control back to event loop
    await new Promise(resolve => setImmediate(resolve));
  }
  
  return results;
}

function calculateCompoundInterest(principal, rate, compounds) {
  return principal * Math.pow((1 + rate / compounds), compounds) - principal;
}

// Usage
const accounts = Array.from({ length: 1000000 }, (_, i) => ({
  id: `ACC-${i}`,
  balance: Math.random() * 100000,
  rate: 0.05
}));

calculateInterestGood(accounts)
  .then(results => console.log(`Calculated interest for ${results.length} accounts`))
  .catch(err => console.error(err));
```

**Example 2: Using Worker Threads for Heavy Operations**
```javascript
// main-thread.js
const { Worker } = require('worker_threads');

class CryptoService {
  constructor(numWorkers = 4) {
    this.workers = [];
    this.queue = [];
    this.currentWorker = 0;
    
    // Create worker pool
    for (let i = 0; i < numWorkers; i++) {
      const worker = new Worker('./crypto-worker.js');
      worker.on('message', this.handleWorkerMessage.bind(this));
      this.workers.push({ worker, busy: false });
    }
  }
  
  async encryptCustomerData(customerId, data) {
    return new Promise((resolve, reject) => {
      const task = { customerId, data, resolve, reject };
      this.queue.push(task);
      this.processQueue();
    });
  }
  
  processQueue() {
    if (this.queue.length === 0) return;
    
    // Find available worker
    const availableWorker = this.workers.find(w => !w.busy);
    if (!availableWorker) return;
    
    const task = this.queue.shift();
    availableWorker.busy = true;
    availableWorker.currentTask = task;
    
    availableWorker.worker.postMessage({
      operation: 'encrypt',
      customerId: task.customerId,
      data: task.data
    });
  }
  
  handleWorkerMessage(message) {
    const worker = this.workers.find(w => 
      w.worker.threadId === message.threadId
    );
    
    if (worker && worker.currentTask) {
      worker.currentTask.resolve(message.result);
      worker.busy = false;
      worker.currentTask = null;
      
      // Process next task
      this.processQueue();
    }
  }
}

// Usage in API endpoint
const cryptoService = new CryptoService(4);

app.post('/api/customer/encrypt', async (req, res) => {
  try {
    const { customerId, sensitiveData } = req.body;
    
    // This doesn't block the event loop!
    const encrypted = await cryptoService.encryptCustomerData(
      customerId,
      sensitiveData
    );
    
    res.json({ success: true, encrypted });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

```javascript
// crypto-worker.js
const { parentPort, threadId } = require('worker_threads');
const crypto = require('crypto');

parentPort.on('message', async (message) => {
  try {
    const { operation, customerId, data } = message;
    
    if (operation === 'encrypt') {
      // CPU-intensive encryption
      const cipher = crypto.createCipher('aes-256-cbc', process.env.SECRET_KEY);
      let encrypted = cipher.update(JSON.stringify(data), 'utf8', 'hex');
      encrypted += cipher.final('hex');
      
      parentPort.postMessage({
        threadId,
        result: {
          customerId,
          encrypted,
          timestamp: new Date().toISOString()
        }
      });
    }
  } catch (error) {
    parentPort.postMessage({
      threadId,
      error: error.message
    });
  }
});
```

**Example 3: Monitoring Event Loop Lag**
```javascript
const { performance } = require('perf_hooks');

class EventLoopMonitor {
  constructor(threshold = 100) {
    this.threshold = threshold; // milliseconds
    this.lastCheck = performance.now();
    this.startMonitoring();
  }
  
  startMonitoring() {
    setInterval(() => {
      const now = performance.now();
      const lag = now - this.lastCheck - 1000; // Expected 1000ms
      
      if (lag > this.threshold) {
        console.warn(`⚠️ Event loop lag detected: ${lag.toFixed(2)}ms`);
        // Alert operations team
        this.alertOps({ lag, timestamp: new Date() });
      }
      
      this.lastCheck = now;
    }, 1000);
  }
  
  alertOps(data) {
    // Send alert to monitoring system
    console.error('ALERT: Event loop blocking detected', data);
  }
}

// Initialize monitor
const monitor = new EventLoopMonitor(50); // Alert if lag > 50ms
```

**Key Takeaways for ENBD:**
- Never run synchronous loops over large datasets
- Use batching with `setImmediate()` for long tasks
- Offload CPU-intensive work to Worker Threads
- Monitor event loop lag in production
- Set timeouts for all operations

---

### Q8. Explain how you would implement graceful shutdown in a Node.js banking application. Why is it critical?

**Answer:**

**Graceful shutdown** ensures that:
1. In-flight requests complete before shutdown
2. Database connections close properly
3. No data loss or corruption
4. Resources are released cleanly

**Critical for Banking Because:**
- Prevents incomplete transactions
- Ensures data consistency
- Maintains audit trail integrity
- Avoids customer impact

**Complete Implementation:**
```javascript
// app.js
const express = require('express');
const mongoose = require('mongoose');
const redis = require('redis');

class BankingApplication {
  constructor() {
    this.app = express();
    this.server = null;
    this.isShuttingDown = false;
    this.activeConnections = new Set();
    
    this.setupMiddleware();
    this.setupRoutes();
    this.setupShutdownHandlers();
  }
  
  setupMiddleware() {
    // Track active connections
    this.app.use((req, res, next) => {
      if (this.isShuttingDown) {
        res.status(503).json({
          error: 'Service shutting down',
          message: 'Please retry your request'
        });
        return;
      }
      
      this.activeConnections.add(req);
      
      res.on('finish', () => {
        this.activeConnections.delete(req);
      });
      
      next();
    });
    
    this.app.use(express.json());
  }
  
  setupRoutes() {
    // Sample banking endpoint
    this.app.post('/api/transfer', async (req, res) => {
      try {
        const { fromAccount, toAccount, amount } = req.body;
        
        // Simulate transaction processing
        const result = await this.processTransfer(fromAccount, toAccount, amount);
        
        res.json({ success: true, transactionId: result.id });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });
    
    // Health check endpoint
    this.app.get('/health', (req, res) => {
      const status = this.isShuttingDown ? 'shutting_down' : 'healthy';
      res.status(this.isShuttingDown ? 503 : 200).json({
        status,
        activeConnections: this.activeConnections.size,
        uptime: process.uptime()
      });
    });
  }
  
  async processTransfer(fromAccount, toAccount, amount) {
    // Simulate database transaction
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve({
          id: `TXN-${Date.now()}`,
          status: 'completed'
        });
      }, 1000);
    });
  }
  
  setupShutdownHandlers() {
    // Handle SIGTERM (Kubernetes, Docker)
    process.on('SIGTERM', () => {
      console.log('📥 SIGTERM received, starting graceful shutdown...');
      this.gracefulShutdown('SIGTERM');
    });
    
    // Handle SIGINT (Ctrl+C)
    process.on('SIGINT', () => {
      console.log('📥 SIGINT received, starting graceful shutdown...');
      this.gracefulShutdown('SIGINT');
    });
    
    // Handle uncaught exceptions
    process.on('uncaughtException', (error) => {
      console.error('❌ Uncaught Exception:', error);
      this.gracefulShutdown('uncaughtException');
    });
    
    // Handle unhandled promise rejections
    process.on('unhandledRejection', (reason, promise) => {
      console.error('❌ Unhandled Rejection at:', promise, 'reason:', reason);
      this.gracefulShutdown('unhandledRejection');
    });
  }
  
  async gracefulShutdown(signal) {
    if (this.isShuttingDown) {
      console.log('⚠️ Shutdown already in progress...');
      return;
    }
    
    this.isShuttingDown = true;
    console.log(`\n🛑 Starting graceful shutdown (signal: ${signal})...`);
    
    const shutdownTimeout = 30000; // 30 seconds
    const shutdownTimer = setTimeout(() => {
      console.error('❌ Shutdown timeout - forcing exit');
      process.exit(1);
    }, shutdownTimeout);
    
    try {
      // Step 1: Stop accepting new connections
      console.log('1️⃣ Stopping server from accepting new connections...');
      await this.stopServer();
      
      // Step 2: Wait for active requests to complete
      console.log(`2️⃣ Waiting for ${this.activeConnections.size} active connections to complete...`);
      await this.waitForActiveConnections();
      
      // Step 3: Close database connections
      console.log('3️⃣ Closing database connections...');
      await this.closeDatabaseConnections();
      
      // Step 4: Close Redis connections
      console.log('4️⃣ Closing Redis connections...');
      await this.closeRedisConnections();
      
      // Step 5: Flush logs and cleanup
      console.log('5️⃣ Flushing logs and cleaning up...');
      await this.cleanup();
      
      clearTimeout(shutdownTimer);
      console.log('✅ Graceful shutdown completed successfully');
      process.exit(0);
      
    } catch (error) {
      console.error('❌ Error during graceful shutdown:', error);
      clearTimeout(shutdownTimer);
      process.exit(1);
    }
  }
  
  stopServer() {
    return new Promise((resolve, reject) => {
      if (!this.server) return resolve();
      
      this.server.close((err) => {
        if (err) {
          console.error('Error stopping server:', err);
          return reject(err);
        }
        console.log('✓ Server stopped accepting new connections');
        resolve();
      });
    });
  }
  
  async waitForActiveConnections(maxWait = 25000) {
    const startTime = Date.now();
    
    while (this.activeConnections.size > 0) {
      if (Date.now() - startTime > maxWait) {
        console.warn(`⚠️ Timeout waiting for connections. ${this.activeConnections.size} still active.`);
        break;
      }
      
      console.log(`⏳ Waiting for ${this.activeConnections.size} connections...`);
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
    
    console.log('✓ All active connections completed');
  }
  
  async closeDatabaseConnections() {
    try {
      await mongoose.connection.close();
      console.log('✓ MongoDB connections closed');
    } catch (error) {
      console.error('Error closing MongoDB:', error);
      throw error;
    }
  }
  
  async closeRedisConnections() {
    // Close Redis client if exists
    if (global.redisClient) {
      await global.redisClient.quit();
      console.log('✓ Redis connections closed');
    }
  }
  
  async cleanup() {
    // Flush any pending logs
    // Clear caches
    // Release resources
    console.log('✓ Cleanup completed');
  }
  
  async start(port = 3000) {
    // Connect to database
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('✓ Connected to MongoDB');
    
    // Start server
    this.server = this.app.listen(port, () => {
      console.log(`🚀 Banking API running on port ${port}`);
      console.log(`📊 Process ID: ${process.pid}`);
    });
  }
}

// Start application
const app = new BankingApplication();
app.start(3000).catch(err => {
  console.error('Failed to start application:', err);
  process.exit(1);
});
```

**Kubernetes Integration (deployment.yaml):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: banking-api
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: banking-api
        image: enbd/banking-api:latest
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
        terminationGracePeriodSeconds: 30
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
```

**Key Points for ENBD:**
- Never kill processes abruptly in production
- Monitor active connections during shutdown
- Set appropriate timeout values (30s recommended)
- Integrate with K8s health checks
- Log all shutdown events for audit

---

### Q9. What is backpressure in Node.js streams and how do you handle it? Provide a banking data processing example.

**Answer:**

**Backpressure** occurs when data is written to a stream faster than it can be consumed, causing memory issues.

**Why Critical for Banking:**
- Processing large transaction files
- Generating customer reports
- Exporting account statements
- Real-time data streaming

**Example: Processing Large Transaction File**
```javascript
const fs = require('fs');
const { Transform } = require('stream');
const { pipeline } = require('stream/promises');

// BAD: No backpressure handling - Memory leak!
function processBad(inputFile, outputFile) {
  const readStream = fs.createReadStream(inputFile);
  const writeStream = fs.createWriteStream(outputFile);
  
  readStream.on('data', (chunk) => {
    // Process chunk
    const processed = processTransactions(chunk);
    // This ignores backpressure!
    writeStream.write(processed);
  });
  
  readStream.on('end', () => {
    writeStream.end();
  });
}

// GOOD: Proper backpressure handling
async function processGood(inputFile, outputFile) {
  const readStream = fs.createReadStream(inputFile, {
    highWaterMark: 64 * 1024 // 64KB chunks
  });
  
  const writeStream = fs.createWriteStream(outputFile, {
    highWaterMark: 64 * 1024
  });
  
  readStream.on('data', (chunk) => {
    const processed = processTransactions(chunk);
    const canContinue = writeStream.write(processed);
    
    if (!canContinue) {
      // Backpressure detected - pause reading
      console.log('⚠️ Backpressure detected, pausing read stream');
      readStream.pause();
    }
  });
  
  writeStream.on('drain', () => {
    // Buffer drained - resume reading
    console.log('✓ Buffer drained, resuming read stream');
    readStream.resume();
  });
  
  readStream.on('end', () => {
    writeStream.end();
  });
}

// BEST: Using pipeline (automatic backpressure handling)
async function processBest(inputFile, outputFile) {
  try {
    await pipeline(
      fs.createReadStream(inputFile, { encoding: 'utf8' }),
      new TransactionParser(),
      new TransactionValidator(),
      new TransactionTransformer(),
      new TransactionAggregator(),
      fs.createWriteStream(outputFile)
    );
    
    console.log('✓ Transaction processing completed successfully');
  } catch (error) {
    console.error('❌ Transaction processing failed:', error);
    throw error;
  }
}

// Custom Transform Stream: Parse CSV transactions
class TransactionParser extends Transform {
  constructor() {
    super({ objectMode: true });
    this.buffer = '';
    this.lineCount = 0;
  }
  
  _transform(chunk, encoding, callback) {
    this.buffer += chunk.toString();
    const lines = this.buffer.split('\n');
    this.buffer = lines.pop(); // Keep incomplete line
    
    for (const line of lines) {
      if (line.trim()) {
        try {
          const transaction = this.parseLine(line);
          this.push(transaction);
          this.lineCount++;
          
          if (this.lineCount % 10000 === 0) {
            console.log(`Parsed ${this.lineCount} transactions`);
          }
        } catch (error) {
          console.error('Error parsing line:', line, error);
        }
      }
    }
    
    callback();
  }
  
  _flush(callback) {
    if (this.buffer.trim()) {
      try {
        const transaction = this.parseLine(this.buffer);
        this.push(transaction);
      } catch (error) {
        console.error('Error parsing final line:', error);
      }
    }
    console.log(`✓ Parsing completed. Total: ${this.lineCount} transactions`);
    callback();
  }
  
  parseLine(line) {
    const [id, date, fromAccount, toAccount, amount, currency, type] = line.split(',');
    return {
      id: id.trim(),
      date: new Date(date.trim()),
      fromAccount: fromAccount.trim(),
      toAccount: toAccount.trim(),
      amount: parseFloat(amount),
      currency: currency.trim(),
      type: type.trim()
    };
  }
}

// Custom Transform Stream: Validate transactions
class TransactionValidator extends Transform {
  constructor() {
    super({ objectMode: true });
    this.validCount = 0;
    this.invalidCount = 0;
  }
  
  _transform(transaction, encoding, callback) {
    if (this.isValid(transaction)) {
      this.validCount++;
      this.push(transaction);
    } else {
      this.invalidCount++;
      console.warn('Invalid transaction:', transaction.id);
    }
    callback();
  }
  
  _flush(callback) {
    console.log(`✓ Validation completed. Valid: ${this.validCount}, Invalid: ${this.invalidCount}`);
    callback();
  }
  
  isValid(txn) {
    return (
      txn.amount > 0 &&
      txn.fromAccount !== txn.toAccount &&
      txn.currency &&
      !isNaN(txn.date.getTime())
    );
  }
}

// Custom Transform Stream: Transform and enrich data
class TransactionTransformer extends Transform {
  constructor() {
    super({ objectMode: true });
  }
  
  _transform(transaction, encoding, callback) {
    // Enrich transaction with additional data
    const enriched = {
      ...transaction,
      fee: this.calculateFee(transaction),
      riskScore: this.calculateRiskScore(transaction),
      processedAt: new Date().toISOString()
    };
    
    this.push(enriched);
    callback();
  }
  
  calculateFee(txn) {
    if (txn.type === 'DOMESTIC') {
      return txn.amount * 0.001; // 0.1%
    } else if (txn.type === 'INTERNATIONAL') {
      return txn.amount * 0.025; // 2.5%
    }
    return 0;
  }
  
  calculateRiskScore(txn) {
    let score = 0;
    if (txn.amount > 100000) score += 30;
    if (txn.type === 'INTERNATIONAL') score += 20;
    if (txn.amount > 500000) score += 50;
    return score;
  }
}

// Custom Transform Stream: Aggregate and format output
class TransactionAggregator extends Transform {
  constructor() {
    super({ objectMode: false }); // Output as string
    this.totalAmount = 0;
    this.totalFees = 0;
    this.count = 0;
    this.isFirst = true;
  }
  
  _transform(transaction, encoding, callback) {
    this.totalAmount += transaction.amount;
    this.totalFees += transaction.fee;
    this.count++;
    
    // Convert to CSV format
    const line = this.toCsvLine(transaction);
    
    if (this.isFirst) {
      this.push(this.getCsvHeader() + '\n');
      this.isFirst = false;
    }
    
    this.push(line + '\n');
    callback();
  }
  
  _flush(callback) {
    // Add summary at the end
    const summary = `\n\nSUMMARY:\n` +
      `Total Transactions: ${this.count}\n` +
      `Total Amount: ${this.totalAmount.toFixed(2)}\n` +
      `Total Fees: ${this.totalFees.toFixed(2)}\n` +
      `Average Transaction: ${(this.totalAmount / this.count).toFixed(2)}`;
    
    this.push(summary);
    callback();
  }
  
  getCsvHeader() {
    return 'ID,Date,From,To,Amount,Currency,Type,Fee,RiskScore,ProcessedAt';
  }
  
  toCsvLine(txn) {
    return `${txn.id},${txn.date.toISOString()},${txn.fromAccount},${txn.toAccount},` +
      `${txn.amount},${txn.currency},${txn.type},${txn.fee},${txn.riskScore},${txn.processedAt}`;
  }
}

function processTransactions(chunk) {
  // Placeholder
  return chunk;
}

// Usage
processBest(
  './data/transactions_2024.csv',
  './output/processed_transactions.csv'
).catch(console.error);
```

**Monitoring Backpressure:**
```javascript
const { Readable, Writable } = require('stream');

class MonitoredWriteStream extends Writable {
  constructor(options) {
    super(options);
    this.backpressureCount = 0;
    this.totalWrites = 0;
  }
  
  _write(chunk, encoding, callback) {
    this.totalWrites++;
    
    // Simulate slow write
    setTimeout(() => {
      callback();
    }, 10);
  }
  
  write(chunk, encoding, callback) {
    const canContinue = super.write(chunk, encoding, callback);
    
    if (!canContinue) {
      this.backpressureCount++;
      console.log(`⚠️ Backpressure event #${this.backpressureCount} ` +
        `(${this.totalWrites} writes)`);
    }
    
    return canContinue;
  }
}

// Test
const readable = new Readable({
  read() {
    for (let i = 0; i < 1000; i++) {
      const canPush = this.push(`data-${i}\n`);
      if (!canPush) break;
    }
  }
});

const writable = new MonitoredWriteStream({ highWaterMark: 16 });

readable.pipe(writable);
```

**Key Takeaways:**
- Always handle backpressure in production
- Use `pipeline()` for automatic handling
- Monitor stream performance
- Set appropriate `highWaterMark` values
- Transform streams for data processing

---

### Q10. Explain memory management in Node.js. How would you detect and fix memory leaks in a banking application?

**Answer:**

**Memory Management in Node.js:**
- Uses V8 JavaScript engine
- Automatic garbage collection
- Heap memory limits (default: ~1.4GB on 64-bit)
- Common causes of leaks: global variables, closures, event listeners, caches

**Detection Tools:**
1. `process.memoryUsage()`
2. Chrome DevTools
3. Heapdump
4. Clinic.js
5. Node.js `--inspect` flag

**Example: Banking Application with Memory Leak**
```javascript
// BAD: Memory leak example
class TransactionProcessor {
  constructor() {
    this.transactions = []; // Growing array - LEAK!
    this.eventEmitter = new EventEmitter();
    
    // Leaked event listeners
    this.eventEmitter.on('transaction', (txn) => {
      this.processTransaction(txn);
    });
  }
  
  processTransaction(txn) {
    // Stores every transaction in memory - LEAK!
    this.transactions.push(txn);
    
    // Create closure that references large data - LEAK!
    setTimeout(() => {
      console.log(`Processing ${txn.id}`, this.transactions.length);
    }, 10000);
  }
  
  // No cleanup method - LEAK!
}

// GOOD: Proper memory management
class TransactionProcessorFixed {
  constructor(maxCacheSize = 1000) {
    this.transactionCache = new Map();
    this.maxCacheSize = maxCacheSize;
    this.eventEmitter = new EventEmitter();
    this.activeListeners = new Set();
    
    this.setupEventListeners();
    this.startMemoryMonitoring();
  }
  
  setupEventListeners() {
    const handler = this.processTransaction.bind(this);
    this.eventEmitter.on('transaction', handler);
    this.activeListeners.add({ event: 'transaction', handler });
  }
  
  async processTransaction(txn) {
    // Use cache with size limit
    if (this.transactionCache.size >= this.maxCacheSize) {
      // Remove oldest entry (FIFO)
      const firstKey = this.transactionCache.keys().next().value;
      this.transactionCache.delete(firstKey);
    }
    
    this.transactionCache.set(txn.id, {
      ...txn,
      processedAt: Date.now()
    });
    
    // Process and clean up
    try {
      await this.saveToDatabase(txn);
      // Remove from cache after saving
      this.transactionCache.delete(txn.id);
    } catch (error) {
      console.error('Transaction processing failed:', error);
    }
  }
  
  async saveToDatabase(txn) {
    // Simulate DB operation
    return new Promise(resolve => setTimeout(resolve, 100));
  }
  
  startMemoryMonitoring() {
    this.monitorInterval = setInterval(() => {
      const usage = process.memoryUsage();
      const heapUsedMB = (usage.heapUsed / 1024 / 1024).toFixed(2);
      const heapTotalMB = (usage.heapTotal / 1024 / 1024).toFixed(2);
      
      console.log(`Memory: ${heapUsedMB} MB / ${heapTotalMB} MB`);
      console.log(`Cache size: ${this.transactionCache.size}`);
      
      // Alert if memory usage is high
      if (usage.heapUsed / usage.heapTotal > 0.9) {
        console.warn('⚠️ High memory usage detected!');
        this.triggerGarbageCollection();
      }
    }, 30000); // Every 30 seconds
  }
  
  triggerGarbageCollection() {
    if (global.gc) {
      console.log('🗑️ Triggering garbage collection...');
      global.gc();
    } else {
      console.warn('Garbage collection not exposed. Run with --expose-gc');
    }
  }
  
  cleanup() {
    // Clear cache
    this.transactionCache.clear();
    
    // Remove event listeners
    for (const { event, handler } of this.activeListeners) {
      this.eventEmitter.removeListener(event, handler);
    }
    this.activeListeners.clear();
    
    // Stop monitoring
    if (this.monitorInterval) {
      clearInterval(this.monitorInterval);
    }
    
    console.log('✓ Cleanup completed');
  }
}

// Memory leak detection utility
class MemoryLeakDetector {
  constructor(app) {
    this.app = app;
    this.snapshots = [];
    this.maxSnapshots = 10;
  }
  
  takeSnapshot() {
    const usage = process.memoryUsage();
    const snapshot = {
      timestamp: new Date().toISOString(),
      heapUsed: usage.heapUsed,
      heapTotal: usage.heapTotal,
      external: usage.external,
      rss: usage.rss
    };
    
    this.snapshots.push(snapshot);
    
    if (this.snapshots.length > this.maxSnapshots) {
      this.snapshots.shift();
    }
    
    return snapshot;
  }
  
  analyzeLeaks() {
    if (this.snapshots.length < 2) {
      return { status: 'insufficient_data' };
    }
    
    const first = this.snapshots[0];
    const last = this.snapshots[this.snapshots.length - 1];
    
    const heapGrowth = last.heapUsed - first.heapUsed;
    const heapGrowthPercent = (heapGrowth / first.heapUsed) * 100;
    
    const analysis = {
      status: 'analyzed',
      timespan: new Date(last.timestamp) - new Date(first.timestamp),
      heapGrowth: heapGrowth,
      heapGrowthPercent: heapGrowthPercent.toFixed(2),
      snapshots: this.snapshots,
      leakDetected: heapGrowthPercent > 50 // 50% growth indicates potential leak
    };
    
    if (analysis.leakDetected) {
      console.error('🚨 MEMORY LEAK DETECTED!', analysis);
      this.generateHeapDump();
    }
    
    return analysis;
  }
  
  generateHeapDump() {
    const heapdump = require('heapdump');
    const filename = `./heapdump-${Date.now()}.heapsnapshot`;
    
    heapdump.writeSnapshot(filename, (err, filename) => {
      if (err) {
        console.error('Failed to write heap dump:', err);
      } else {
        console.log(`✓ Heap dump written to: ${filename}`);
        console.log('Analyze with Chrome DevTools');
      }
    });
  }
  
  startMonitoring(intervalMs = 60000) {
    console.log('🔍 Starting memory leak detection...');
    
    this.monitoringInterval = setInterval(() => {
      this.takeSnapshot();
      this.analyzeLeaks();
    }, intervalMs);
  }
  
  stopMonitoring() {
    if (this.monitoringInterval) {
      clearInterval(this.monitoringInterval);
      console.log('✓ Memory monitoring stopped');
    }
  }
}

// Usage in Express app
const express = require('express');
const app = express();

const processor = new TransactionProcessorFixed();
const leakDetector = new MemoryLeakDetector(app);

// Start monitoring
leakDetector.startMonitoring(60000); // Every minute

// Endpoint to check memory
app.get('/api/memory', (req, res) => {
  const usage = process.memoryUsage();
  const analysis = leakDetector.analyzeLeaks();
  
  res.json({
    current: {
      heapUsed: `${(usage.heapUsed / 1024 / 1024).toFixed(2)} MB`,
      heapTotal: `${(usage.heapTotal / 1024 / 1024).toFixed(2)} MB`,
      rss: `${(usage.rss / 1024 / 1024).toFixed(2)} MB`,
      external: `${(usage.external / 1024 / 1024).toFixed(2)} MB`
    },
    analysis
  });
});

// Endpoint to force GC (for testing)
app.post('/api/gc', (req, res) => {
  if (global.gc) {
    const before = process.memoryUsage().heapUsed;
    global.gc();
    const after = process.memoryUsage().heapUsed;
    const freed = before - after;
    
    res.json({
      success: true,
      freedMemory: `${(freed / 1024 / 1024).toFixed(2)} MB`
    });
  } else {
    res.status(400).json({
      error: 'GC not exposed. Run with --expose-gc flag'
    });
  }
});

// Graceful shutdown with cleanup
process.on('SIGTERM', () => {
  console.log('Shutting down...');
  processor.cleanup();
  leakDetector.stopMonitoring();
  process.exit(0);
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
  console.log('Run with: node --expose-gc --max-old-space-size=4096 app.js');
});
```

**Best Practices for ENBD:**
```javascript
// 1. Use WeakMap for caching
const cache = new WeakMap();

// 2. Set max listeners
eventEmitter.setMaxListeners(20);

// 3. Clean up timers
const timerId = setTimeout(() => {}, 1000);
clearTimeout(timerId);

// 4. Remove event listeners
const handler = () => {};
emitter.on('event', handler);
emitter.off('event', handler);

// 5. Use connection pooling
const pool = new Pool({
  max: 20,
  min: 5,
  idleTimeoutMillis: 30000
});
```

**Key Commands:**
```bash
# Run with exposed GC
node --expose-gc app.js

# Increase heap size
node --max-old-space-size=4096 app.js

# Generate heap snapshot
node --inspect app.js
# Then use Chrome DevTools

# Profile with clinic
clinic doctor -- node app.js
```

---

**🎯 Summary - Section 1 Complete (Questions 1-10)**

You've covered:
- Event Loop architecture and phases
- Async execution timing (nextTick, setImmediate, setTimeout)
- Callback hell and modern solutions
- Concurrency in single-threaded environment
- Worker Threads for CPU-intensive tasks
- Event loop blocking prevention
- Graceful shutdown implementation
- Backpressure in streams
- Memory management and leak detection

**Next:** Section 2 (Q11-Q20) - Promises, Async/Await, and Error Handling

---

## Section 2: Promises, Async/Await, and Error Handling

### Questions 11-20

---

### Q11. Explain Promise internals. What are the three states of a Promise and how do they transition?

**Answer:**

**Promise States:**
1. **Pending** - Initial state, neither fulfilled nor rejected
2. **Fulfilled** - Operation completed successfully
3. **Rejected** - Operation failed

**State Transitions:**
```
Pending → Fulfilled (resolve called)
Pending → Rejected (reject called)
```
Once settled (fulfilled or rejected), a Promise **cannot change state**.

**Promise Internals:**
```javascript
// Simplified Promise implementation
class MyPromise {
  constructor(executor) {
    this.state = 'pending';
    this.value = undefined;
    this.reason = undefined;
    this.onFulfilledCallbacks = [];
    this.onRejectedCallbacks = [];
    
    const resolve = (value) => {
      if (this.state === 'pending') {
        this.state = 'fulfilled';
        this.value = value;
        this.onFulfilledCallbacks.forEach(fn => fn(value));
      }
    };
    
    const reject = (reason) => {
      if (this.state === 'pending') {
        this.state = 'rejected';
        this.reason = reason;
        this.onRejectedCallbacks.forEach(fn => fn(reason));
      }
    };
    
    try {
      executor(resolve, reject);
    } catch (error) {
      reject(error);
    }
  }
  
  then(onFulfilled, onRejected) {
    return new MyPromise((resolve, reject) => {
      if (this.state === 'fulfilled') {
        try {
          const result = onFulfilled ? onFulfilled(this.value) : this.value;
          resolve(result);
        } catch (error) {
          reject(error);
        }
      } else if (this.state === 'rejected') {
        try {
          const result = onRejected ? onRejected(this.reason) : this.reason;
          reject(result);
        } catch (error) {
          reject(error);
        }
      } else {
        this.onFulfilledCallbacks.push((value) => {
          try {
            const result = onFulfilled ? onFulfilled(value) : value;
            resolve(result);
          } catch (error) {
            reject(error);
          }
        });
        
        this.onRejectedCallbacks.push((reason) => {
          try {
            const result = onRejected ? onRejected(reason) : reason;
            reject(result);
          } catch (error) {
            reject(error);
          }
        });
      }
    });
  }
  
  catch(onRejected) {
    return this.then(null, onRejected);
  }
  
  finally(onFinally) {
    return this.then(
      value => {
        onFinally();
        return value;
      },
      reason => {
        onFinally();
        throw reason;
      }
    );
  }
}

// Banking Example: Transaction Processing with State Tracking
class TransactionPromise {
  constructor(transactionId, amount) {
    this.transactionId = transactionId;
    this.amount = amount;
    
    return new Promise((resolve, reject) => {
      console.log(`[${transactionId}] State: PENDING`);
      
      // Simulate transaction processing
      setTimeout(() => {
        if (amount > 0 && amount <= 1000000) {
          console.log(`[${transactionId}] State: FULFILLED`);
          resolve({
            transactionId,
            amount,
            status: 'SUCCESS',
            timestamp: new Date().toISOString()
          });
        } else {
          console.log(`[${transactionId}] State: REJECTED`);
          reject({
            transactionId,
            error: 'Invalid amount',
            code: 'INVALID_AMOUNT'
          });
        }
      }, 1000);
    });
  }
}

// Usage
const txn1 = new TransactionPromise('TXN-001', 5000);

txn1
  .then(result => {
    console.log('✓ Transaction successful:', result);
    return result;
  })
  .catch(error => {
    console.error('✗ Transaction failed:', error);
  })
  .finally(() => {
    console.log('Transaction processing completed');
  });
```

**Real-World Banking Scenario:**
```javascript
class BankingTransactionService {
  async processTransaction(txnData) {
    // Promise chain with state transitions
    return this.validateTransaction(txnData)       // State: Pending
      .then(validated => this.checkBalance(validated))  // Fulfilled → Pending
      .then(checked => this.debitAccount(checked))      // Fulfilled → Pending
      .then(debited => this.creditAccount(debited))     // Fulfilled → Pending
      .then(credited => this.updateLedger(credited))    // Fulfilled → Pending
      .then(updated => ({                               // Fulfilled
        status: 'COMPLETED',
        transactionId: updated.id,
        timestamp: new Date()
      }))
      .catch(error => ({                                // Rejected
        status: 'FAILED',
        error: error.message,
        rollback: this.rollbackTransaction(txnData)
      }));
  }
  
  validateTransaction(txnData) {
    return new Promise((resolve, reject) => {
      if (!txnData.fromAccount || !txnData.toAccount) {
        reject(new Error('Missing account information'));
      } else if (txnData.amount <= 0) {
        reject(new Error('Invalid amount'));
      } else {
        resolve(txnData);
      }
    });
  }
  
  checkBalance(txnData) {
    return new Promise((resolve, reject) => {
      // Simulate DB call
      const balance = 10000;
      if (balance >= txnData.amount) {
        resolve({ ...txnData, balance });
      } else {
        reject(new Error('Insufficient balance'));
      }
    });
  }
  
  debitAccount(txnData) {
    return new Promise(resolve => {
      setTimeout(() => resolve(txnData), 100);
    });
  }
  
  creditAccount(txnData) {
    return new Promise(resolve => {
      setTimeout(() => resolve(txnData), 100);
    });
  }
  
  updateLedger(txnData) {
    return new Promise(resolve => {
      setTimeout(() => resolve({ id: 'TXN-' + Date.now() }), 100);
    });
  }
  
  rollbackTransaction(txnData) {
    console.log('Rolling back transaction:', txnData);
  }
}
```

**Key Points for ENBD:**
- Promises are immutable once settled
- Always handle rejections to prevent unhandled promise rejections
- Use Promise state transitions for transaction tracking
- Implement proper error handling in promise chains

---

### Q12. What is Promise chaining vs Promise.all() vs Promise.race() vs Promise.allSettled()? When would you use each in banking operations?

**Answer:**

| Method | Behavior | Use Case | Failure Handling |
|--------|----------|----------|------------------|
| **Promise Chaining** | Sequential execution | Dependent operations | Stops at first error |
| **Promise.all()** | Parallel, waits for all | All must succeed | Fails if any fails |
| **Promise.race()** | Returns first settled | Fastest response | Returns first result |
| **Promise.allSettled()** | Waits for all, no fail-fast | Need all results | Never rejects |
| **Promise.any()** | First fulfilled | First success wins | Fails if all fail |

**Complete Banking Examples:**

```javascript
// 1. PROMISE CHAINING - Sequential Dependent Operations
async function processLoanApplication(customerId) {
  console.log('=== Promise Chaining: Loan Application ===');
  
  return validateCustomer(customerId)
    .then(customer => {
      console.log('✓ Customer validated:', customer.name);
      return checkCreditScore(customer.id);
    })
    .then(creditScore => {
      console.log('✓ Credit score retrieved:', creditScore);
      if (creditScore < 650) {
        throw new Error('Credit score too low');
      }
      return verifyEmployment(customerId);
    })
    .then(employment => {
      console.log('✓ Employment verified');
      return calculateLoanAmount(employment.income);
    })
    .then(loanAmount => {
      console.log('✓ Loan approved:', loanAmount);
      return { approved: true, amount: loanAmount };
    })
    .catch(error => {
      console.error('✗ Loan application failed:', error.message);
      return { approved: false, reason: error.message };
    });
}

// 2. PROMISE.ALL() - Multiple Independent Operations (All Must Succeed)
async function fetchCustomerDashboard(customerId) {
  console.log('\n=== Promise.all(): Customer Dashboard ===');
  
  try {
    const [
      accountDetails,
      recentTransactions,
      cards,
      loans,
      investments
    ] = await Promise.all([
      fetchAccountDetails(customerId),
      fetchRecentTransactions(customerId),
      fetchCards(customerId),
      fetchLoans(customerId),
      fetchInvestments(customerId)
    ]);
    
    console.log('✓ All dashboard data loaded');
    return {
      accountDetails,
      recentTransactions,
      cards,
      loans,
      investments,
      loadedAt: new Date()
    };
  } catch (error) {
    // If ANY promise fails, entire operation fails
    console.error('✗ Dashboard loading failed:', error.message);
    throw error;
  }
}

// 3. PROMISE.RACE() - Fastest Response Wins (Redundant Services)
async function fetchExchangeRate(fromCurrency, toCurrency) {
  console.log('\n=== Promise.race(): Exchange Rate ===');
  
  // Query multiple providers, use fastest response
  const providers = [
    fetchFromProvider1(fromCurrency, toCurrency),
    fetchFromProvider2(fromCurrency, toCurrency),
    fetchFromProvider3(fromCurrency, toCurrency),
    timeoutPromise(5000, 'Exchange rate timeout')
  ];
  
  try {
    const result = await Promise.race(providers);
    console.log('✓ Exchange rate received:', result);
    return result;
  } catch (error) {
    console.error('✗ All providers failed or timeout');
    throw error;
  }
}

// 4. PROMISE.ALLSETTLED() - Need All Results Regardless of Success
async function processEndOfDayReconciliation(accountIds) {
  console.log('\n=== Promise.allSettled(): End of Day Reconciliation ===');
  
  // Process all accounts, even if some fail
  const reconciliationPromises = accountIds.map(accountId =>
    reconcileAccount(accountId)
      .then(result => ({ accountId, status: 'success', result }))
      .catch(error => ({ accountId, status: 'failed', error: error.message }))
  );
  
  const results = await Promise.allSettled(reconciliationPromises);
  
  const summary = {
    total: results.length,
    successful: 0,
    failed: 0,
    details: []
  };
  
  results.forEach(result => {
    if (result.status === 'fulfilled') {
      const data = result.value;
      if (data.status === 'success') {
        summary.successful++;
        console.log(`✓ Account ${data.accountId}: Reconciled`);
      } else {
        summary.failed++;
        console.error(`✗ Account ${data.accountId}: ${data.error}`);
      }
      summary.details.push(data);
    } else {
      summary.failed++;
      console.error(`✗ Promise rejected:`, result.reason);
    }
  });
  
  console.log('\nReconciliation Summary:', summary);
  return summary;
}

// 5. PROMISE.ANY() - First Success Wins (Fallback Services)
async function sendTransactionAlert(customerId, message) {
  console.log('\n=== Promise.any(): Transaction Alert ===');
  
  // Try multiple channels, succeed if ANY succeeds
  const channels = [
    sendViaSMS(customerId, message),
    sendViaEmail(customerId, message),
    sendViaPushNotification(customerId, message),
    sendViaWhatsApp(customerId, message)
  ];
  
  try {
    const result = await Promise.any(channels);
    console.log('✓ Alert sent via:', result.channel);
    return result;
  } catch (error) {
    // AggregateError - all promises rejected
    console.error('✗ All alert channels failed');
    console.error('Errors:', error.errors);
    throw new Error('Failed to send alert via any channel');
  }
}

// Helper Functions
function validateCustomer(customerId) {
  return new Promise(resolve => {
    setTimeout(() => resolve({ id: customerId, name: 'John Doe' }), 100);
  });
}

function checkCreditScore(customerId) {
  return new Promise(resolve => {
    setTimeout(() => resolve(720), 100);
  });
}

function verifyEmployment(customerId) {
  return new Promise(resolve => {
    setTimeout(() => resolve({ income: 75000, verified: true }), 100);
  });
}

function calculateLoanAmount(income) {
  return new Promise(resolve => {
    setTimeout(() => resolve(income * 3), 100);
  });
}

function fetchAccountDetails(customerId) {
  return new Promise(resolve => {
    setTimeout(() => resolve({ balance: 50000, type: 'savings' }), 200);
  });
}

function fetchRecentTransactions(customerId) {
  return new Promise(resolve => {
    setTimeout(() => resolve([{ id: 1, amount: 100 }]), 300);
  });
}

function fetchCards(customerId) {
  return new Promise(resolve => {
    setTimeout(() => resolve([{ type: 'credit', limit: 10000 }]), 150);
  });
}

function fetchLoans(customerId) {
  return new Promise(resolve => {
    setTimeout(() => resolve([{ type: 'personal', amount: 20000 }]), 250);
  });
}

function fetchInvestments(customerId) {
  return new Promise(resolve => {
    setTimeout(() => resolve([{ type: 'mutual fund', value: 30000 }]), 180);
  });
}

function fetchFromProvider1(from, to) {
  return new Promise((resolve) => {
    setTimeout(() => resolve({ rate: 3.67, provider: 'Provider1' }), 800);
  });
}

function fetchFromProvider2(from, to) {
  return new Promise((resolve) => {
    setTimeout(() => resolve({ rate: 3.68, provider: 'Provider2' }), 500);
  });
}

function fetchFromProvider3(from, to) {
  return new Promise((resolve, reject) => {
    setTimeout(() => reject(new Error('Provider3 unavailable')), 1000);
  });
}

function timeoutPromise(ms, message) {
  return new Promise((_, reject) => {
    setTimeout(() => reject(new Error(message)), ms);
  });
}

function reconcileAccount(accountId) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      // Simulate random failures
      if (Math.random() > 0.8) {
        reject(new Error('Reconciliation mismatch'));
      } else {
        resolve({ reconciled: true, balance: 10000 });
      }
    }, 100);
  });
}

function sendViaSMS(customerId, message) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      Math.random() > 0.5 
        ? resolve({ channel: 'SMS', sent: true })
        : reject(new Error('SMS service unavailable'));
    }, 200);
  });
}

function sendViaEmail(customerId, message) {
  return new Promise((resolve) => {
    setTimeout(() => resolve({ channel: 'Email', sent: true }), 300);
  });
}

function sendViaPushNotification(customerId, message) {
  return new Promise((resolve, reject) => {
    setTimeout(() => reject(new Error('Push notification failed')), 100);
  });
}

function sendViaWhatsApp(customerId, message) {
  return new Promise((resolve, reject) => {
    setTimeout(() => reject(new Error('WhatsApp not configured')), 250);
  });
}

// Run Examples
async function runExamples() {
  await processLoanApplication('CUST-001');
  await fetchCustomerDashboard('CUST-001');
  await fetchExchangeRate('USD', 'AED');
  await processEndOfDayReconciliation(['ACC-001', 'ACC-002', 'ACC-003', 'ACC-004', 'ACC-005']);
  await sendTransactionAlert('CUST-001', 'Transaction completed');
}

runExamples().catch(console.error);
```

**Decision Matrix for ENBD:**

```javascript
// Use CHAINING when:
// - Operations depend on previous results
// - Sequential execution required
await step1()
  .then(result1 => step2(result1))
  .then(result2 => step3(result2));

// Use PROMISE.ALL when:
// - All operations must succeed
// - Operations are independent
// - Need all results to proceed
const [data1, data2, data3] = await Promise.all([fetch1(), fetch2(), fetch3()]);

// Use PROMISE.RACE when:
// - Need fastest response
// - Multiple redundant services
// - Implement timeouts
const result = await Promise.race([fetchData(), timeout(5000)]);

// Use PROMISE.ALLSETTLED when:
// - Need all results regardless of failures
// - Reporting/reconciliation
// - Batch operations
const results = await Promise.allSettled(operations);

// Use PROMISE.ANY when:
// - Need at least one success
// - Fallback mechanisms
// - Multi-channel operations
const result = await Promise.any([primary(), fallback1(), fallback2()]);
```

---

### Q13. Explain async/await internals. How does it differ from Promises and what are the performance implications?

**Answer:**

**Async/Await is Syntactic Sugar over Promises:**
- `async` function always returns a Promise
- `await` pauses execution until Promise settles
- Under the hood, converted to Promise chains by JavaScript engine

**How It Works:**
```javascript
// This async/await code:
async function fetchUserData(userId) {
  const user = await getUser(userId);
  const posts = await getPosts(user.id);
  return { user, posts };
}

// Is transformed to this by the engine:
function fetchUserData(userId) {
  return getUser(userId).then(user => {
    return getPosts(user.id).then(posts => {
      return { user, posts };
    });
  });
}
```

**Performance Comparison:**

```javascript
// Banking Example: Transaction Processing Performance Test
const { performance } = require('perf_hooks');

class PerformanceComparator {
  // Method 1: Callback Hell (Fastest but unmaintainable)
  processWithCallbacks(txnData, callback) {
    this.validateTransaction(txnData, (err, validated) => {
      if (err) return callback(err);
      
      this.checkFraud(validated, (err, checked) => {
        if (err) return callback(err);
        
        this.debitAccount(checked, (err, debited) => {
          if (err) return callback(err);
          
          this.creditAccount(debited, (err, credited) => {
            if (err) return callback(err);
            callback(null, credited);
          });
        });
      });
    });
  }
  
  // Method 2: Promise Chain (Fast, better than callbacks)
  processWithPromises(txnData) {
    return this.validateTransaction(txnData)
      .then(validated => this.checkFraud(validated))
      .then(checked => this.debitAccount(checked))
      .then(debited => this.creditAccount(debited));
  }
  
  // Method 3: Async/Await (Cleanest, minimal overhead)
  async processWithAsyncAwait(txnData) {
    const validated = await this.validateTransaction(txnData);
    const checked = await this.checkFraud(validated);
    const debited = await this.debitAccount(checked);
    const credited = await this.creditAccount(debited);
    return credited;
  }
  
  // Method 4: Parallel Async/Await (Fastest when operations independent)
  async processParallel(txnData) {
    const [validated, fraudCheck] = await Promise.all([
      this.validateTransaction(txnData),
      this.checkFraud(txnData)
    ]);
    
    const [debited, credited] = await Promise.all([
      this.debitAccount(validated),
      this.creditAccount(validated)
    ]);
    
    return { debited, credited };
  }
  
  // Helper methods (async operations)
  validateTransaction(txnData) {
    return new Promise(resolve => {
      setTimeout(() => resolve(txnData), 10);
    });
  }
  
  checkFraud(txnData) {
    return new Promise(resolve => {
      setTimeout(() => resolve(txnData), 10);
    });
  }
  
  debitAccount(txnData) {
    return new Promise(resolve => {
      setTimeout(() => resolve(txnData), 10);
    });
  }
  
  creditAccount(txnData) {
    return new Promise(resolve => {
      setTimeout(() => resolve(txnData), 10);
    });
  }
}

// Performance Benchmark
async function runBenchmark() {
  const comparator = new PerformanceComparator();
  const iterations = 10000;
  const txnData = { amount: 1000, from: 'ACC1', to: 'ACC2' };
  
  console.log('=== Performance Benchmark ===\n');
  
  // Test 1: Callbacks
  const callbackStart = performance.now();
  let callbacksCompleted = 0;
  
  for (let i = 0; i < iterations; i++) {
    comparator.processWithCallbacks(txnData, (err, result) => {
      callbacksCompleted++;
      if (callbacksCompleted === iterations) {
        const callbackEnd = performance.now();
        console.log(`Callbacks: ${(callbackEnd - callbackStart).toFixed(2)}ms`);
        testPromises();
      }
    });
  }
  
  // Test 2: Promises
  async function testPromises() {
    const promiseStart = performance.now();
    const promiseTests = [];
    
    for (let i = 0; i < iterations; i++) {
      promiseTests.push(comparator.processWithPromises(txnData));
    }
    
    await Promise.all(promiseTests);
    const promiseEnd = performance.now();
    console.log(`Promises: ${(promiseEnd - promiseStart).toFixed(2)}ms`);
    
    // Test 3: Async/Await
    const asyncStart = performance.now();
    const asyncTests = [];
    
    for (let i = 0; i < iterations; i++) {
      asyncTests.push(comparator.processWithAsyncAwait(txnData));
    }
    
    await Promise.all(asyncTests);
    const asyncEnd = performance.now();
    console.log(`Async/Await: ${(asyncEnd - asyncStart).toFixed(2)}ms`);
    
    // Test 4: Parallel
    const parallelStart = performance.now();
    const parallelTests = [];
    
    for (let i = 0; i < iterations; i++) {
      parallelTests.push(comparator.processParallel(txnData));
    }
    
    await Promise.all(parallelTests);
    const parallelEnd = performance.now();
    console.log(`Parallel Async/Await: ${(parallelEnd - parallelStart).toFixed(2)}ms`);
  }
}

// runBenchmark();
```

**Common Pitfalls and Solutions:**

```javascript
// PITFALL 1: Sequential when should be parallel
// ❌ BAD - Takes 6 seconds
async function fetchCustomerDataBad(customerId) {
  const profile = await fetchProfile(customerId);      // 2 seconds
  const transactions = await fetchTransactions(customerId); // 2 seconds
  const cards = await fetchCards(customerId);          // 2 seconds
  return { profile, transactions, cards };
}

// ✅ GOOD - Takes 2 seconds
async function fetchCustomerDataGood(customerId) {
  const [profile, transactions, cards] = await Promise.all([
    fetchProfile(customerId),
    fetchTransactions(customerId),
    fetchCards(customerId)
  ]);
  return { profile, transactions, cards };
}

// PITFALL 2: Forgetting error handling
// ❌ BAD - Unhandled promise rejection
async function transferMoneyBad(from, to, amount) {
  await debitAccount(from, amount);  // If this fails, no rollback!
  await creditAccount(to, amount);
}

// ✅ GOOD - Proper error handling with rollback
async function transferMoneyGood(from, to, amount) {
  let debited = false;
  
  try {
    await debitAccount(from, amount);
    debited = true;
    
    await creditAccount(to, amount);
    
    return { success: true, transactionId: generateId() };
  } catch (error) {
    // Rollback if debit succeeded but credit failed
    if (debited) {
      await creditAccount(from, amount); // Refund
    }
    
    throw new Error(`Transfer failed: ${error.message}`);
  }
}

// PITFALL 3: Not handling Promise.all failures
// ❌ BAD - One failure fails everything
async function processTransactionsBad(transactions) {
  return await Promise.all(
    transactions.map(txn => processTransaction(txn))
  );
}

// ✅ GOOD - Handle individual failures
async function processTransactionsGood(transactions) {
  const results = await Promise.allSettled(
    transactions.map(async (txn) => {
      try {
        return await processTransaction(txn);
      } catch (error) {
        return { error: error.message, txn };
      }
    })
  );
  
  const successful = results.filter(r => r.status === 'fulfilled');
  const failed = results.filter(r => r.status === 'rejected');
  
  return { successful, failed };
}

// PITFALL 4: Async in loops
// ❌ BAD - Sequential processing (slow)
async function processAccountsBad(accountIds) {
  const results = [];
  for (const id of accountIds) {
    const result = await updateAccount(id); // Waits for each!
    results.push(result);
  }
  return results;
}

// ✅ GOOD - Batch processing
async function processAccountsGood(accountIds, batchSize = 100) {
  const results = [];
  
  for (let i = 0; i < accountIds.length; i += batchSize) {
    const batch = accountIds.slice(i, i + batchSize);
    const batchResults = await Promise.all(
      batch.map(id => updateAccount(id))
    );
    results.push(...batchResults);
  }
  
  return results;
}

// Helper functions
function fetchProfile(id) {
  return new Promise(resolve => setTimeout(() => resolve({ id }), 2000));
}

function fetchTransactions(id) {
  return new Promise(resolve => setTimeout(() => resolve([]), 2000));
}

function fetchCards(id) {
  return new Promise(resolve => setTimeout(() => resolve([]), 2000));
}

function debitAccount(account, amount) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (Math.random() > 0.1) resolve();
      else reject(new Error('Debit failed'));
    }, 100);
  });
}

function creditAccount(account, amount) {
  return new Promise(resolve => setTimeout(resolve, 100));
}

function generateId() {
  return 'TXN-' + Date.now();
}

function processTransaction(txn) {
  return new Promise(resolve => setTimeout(() => resolve(txn), 100));
}

function updateAccount(id) {
  return new Promise(resolve => setTimeout(() => resolve({ id }), 100));
}
```

**Best Practices for ENBD:**

```javascript
class BestPracticesExample {
  // 1. Always use try-catch with async/await
  async safeOperation() {
    try {
      const result = await riskyOperation();
      return result;
    } catch (error) {
      console.error('Operation failed:', error);
      // Handle or rethrow
      throw error;
    }
  }
  
  // 2. Use Promise.all for parallel operations
  async parallelOperations() {
    const [result1, result2, result3] = await Promise.all([
      operation1(),
      operation2(),
      operation3()
    ]);
    return { result1, result2, result3 };
  }
  
  // 3. Add timeouts to prevent hanging
  async operationWithTimeout(timeoutMs = 5000) {
    const timeout = new Promise((_, reject) =>
      setTimeout(() => reject(new Error('Timeout')), timeoutMs)
    );
    
    return Promise.race([
      actualOperation(),
      timeout
    ]);
  }
  
  // 4. Proper cleanup with finally
  async operationWithCleanup() {
    const connection = await getConnection();
    
    try {
      return await performOperation(connection);
    } catch (error) {
      console.error('Operation failed:', error);
      throw error;
    } finally {
      await connection.close(); // Always cleanup
    }
  }
  
  // 5. Avoid mixing async/await with .then()
  // ❌ BAD
  async mixedBad() {
    return await fetchData().then(data => processData(data));
  }
  
  // ✅ GOOD
  async mixedGood() {
    const data = await fetchData();
    return processData(data);
  }
}

function riskyOperation() {
  return Promise.resolve('success');
}

function operation1() {
  return Promise.resolve('result1');
}

function operation2() {
  return Promise.resolve('result2');
}

function operation3() {
  return Promise.resolve('result3');
}

function actualOperation() {
  return Promise.resolve('completed');
}

function getConnection() {
  return Promise.resolve({ close: () => Promise.resolve() });
}

function performOperation(conn) {
  return Promise.resolve('done');
}

function fetchData() {
  return Promise.resolve('data');
}

function processData(data) {
  return data;
}
```

**Performance Implications:**
- Async/await has ~5-10% overhead vs raw Promises (negligible)
- Readability and maintainability benefits far outweigh minimal overhead
- Parallel execution with `Promise.all()` is critical for performance
- Always profile your specific use case

---

### Q14. How do you handle errors in async/await? Explain different error handling patterns and when to use each.

**Answer:**

**Error Handling Patterns:**

```javascript
// Pattern 1: Try-Catch Block (Most Common)
async function pattern1_TryCatch(accountId) {
  try {
    const account = await fetchAccount(accountId);
    const balance = await getBalance(account);
    const transactions = await getTransactions(account);
    
    return { account, balance, transactions };
  } catch (error) {
    console.error('Error in pattern1:', error);
    throw error; // Rethrow or handle
  }
}

// Pattern 2: Try-Catch with Specific Error Handling
async function pattern2_SpecificErrors(accountId) {
  try {
    const account = await fetchAccount(accountId);
    return account;
  } catch (error) {
    if (error.code === 'ACCOUNT_NOT_FOUND') {
      console.log('Account not found, creating new...');
      return await createAccount(accountId);
    } else if (error.code === 'ACCOUNT_LOCKED') {
      throw new Error('Account is locked. Contact support.');
    } else {
      throw error; // Unknown error
    }
  }
}

// Pattern 3: Multiple Try-Catch Blocks
async function pattern3_MultipleTryCatch(customerId) {
  let customer;
  
  // Critical operation
  try {
    customer = await fetchCustomer(customerId);
  } catch (error) {
    console.error('Failed to fetch customer:', error);
    throw error; // Cannot proceed without customer
  }
  
  // Non-critical operation
  let preferences = null;
  try {
    preferences = await fetchPreferences(customerId);
  } catch (error) {
    console.warn('Failed to fetch preferences, using defaults:', error);
    preferences = getDefaultPreferences();
  }
  
  // Another non-critical operation
  let avatar = null;
  try {
    avatar = await fetchAvatar(customerId);
  } catch (error) {
    console.warn('Failed to fetch avatar:', error);
    // Continue without avatar
  }
  
  return { customer, preferences, avatar };
}

// Pattern 4: Promise.catch() Method
async function pattern4_PromiseCatch(accountId) {
  const account = await fetchAccount(accountId)
    .catch(error => {
      console.error('Fetch failed, using cached:', error);
      return getCachedAccount(accountId);
    });
  
  return account;
}

// Pattern 5: Wrapper Function (Go-style Error Handling)
function asyncWrapper(promise) {
  return promise
    .then(data => [null, data])
    .catch(error => [error, null]);
}

async function pattern5_Wrapper(accountId) {
  const [error, account] = await asyncWrapper(fetchAccount(accountId));
  
  if (error) {
    console.error('Error:', error);
    return null;
  }
  
  return account;
}

// Pattern 6: Custom Error Classes
class BankingError extends Error {
  constructor(message, code, statusCode = 500) {
    super(message);
    this.name = this.constructor.name;
    this.code = code;
    this.statusCode = statusCode;
    Error.captureStackTrace(this, this.constructor);
  }
}

class InsufficientFundsError extends BankingError {
  constructor(balance, required) {
    super(
      `Insufficient funds. Balance: ${balance}, Required: ${required}`,
      'INSUFFICIENT_FUNDS',
      400
    );
    this.balance = balance;
    this.required = required;
  }
}

class AccountNotFoundError extends BankingError {
  constructor(accountId) {
    super(
      `Account ${accountId} not found`,
      'ACCOUNT_NOT_FOUND',
      404
    );
    this.accountId = accountId;
  }
}

async function pattern6_CustomErrors(accountId, amount) {
  try {
    const account = await fetchAccount(accountId);
    
    if (!account) {
      throw new AccountNotFoundError(accountId);
    }
    
    if (account.balance < amount) {
      throw new InsufficientFundsError(account.balance, amount);
    }
    
    return await processWithdrawal(account, amount);
  } catch (error) {
    if (error instanceof InsufficientFundsError) {
      console.error('Insufficient funds:', error.balance, '<', error.required);
      return { success: false, reason: 'insufficient_funds' };
    } else if (error instanceof AccountNotFoundError) {
      console.error('Account not found:', error.accountId);
      return { success: false, reason: 'account_not_found' };
    } else {
      console.error('Unexpected error:', error);
      throw error;
    }
  }
}

// Pattern 7: Centralized Error Handler (Express.js)
class ErrorHandler {
  static handle(error, req, res, next) {
    console.error('Error:', error);
    
    if (error instanceof BankingError) {
      return res.status(error.statusCode).json({
        success: false,
        error: {
          code: error.code,
          message: error.message
        }
      });
    }
    
    // Unknown error
    return res.status(500).json({
      success: false,
      error: {
        code: 'INTERNAL_ERROR',
        message: 'An unexpected error occurred'
      }
    });
  }
}

// Usage in Express
const express = require('express');
const app = express();

app.post('/api/withdraw', async (req, res, next) => {
  try {
    const { accountId, amount } = req.body;
    const result = await pattern6_CustomErrors(accountId, amount);
    res.json(result);
  } catch (error) {
    next(error); // Pass to centralized handler
  }
});

app.use(ErrorHandler.handle);
```

**Real-World Banking Scenario: Complete Error Handling**

```javascript
class TransactionService {
  constructor() {
    this.maxRetries = 3;
    this.retryDelay = 1000;
  }
  
  // Comprehensive error handling for money transfer
  async transferMoney(fromAccountId, toAccountId, amount) {
    const transactionId = this.generateTransactionId();
    console.log(`Starting transaction: ${transactionId}`);
    
    // Step 1: Validate inputs
    try {
      this.validateTransferInputs(fromAccountId, toAccountId, amount);
    } catch (error) {
      console.error('Validation error:', error.message);
      return {
        success: false,
        transactionId,
        error: 'VALIDATION_ERROR',
        message: error.message
      };
    }
    
    let fromAccount, toAccount, debitComplete = false;
    
    try {
      // Step 2: Fetch accounts (with retry)
      fromAccount = await this.fetchAccountWithRetry(fromAccountId);
      toAccount = await this.fetchAccountWithRetry(toAccountId);
      
      // Step 3: Check balance
      if (fromAccount.balance < amount) {
        throw new InsufficientFundsError(fromAccount.balance, amount);
      }
      
      // Step 4: Check daily limit
      const todayTotal = await this.getTodayTransactionTotal(fromAccountId);
      if (todayTotal + amount > fromAccount.dailyLimit) {
        throw new BankingError(
          'Daily transaction limit exceeded',
          'DAILY_LIMIT_EXCEEDED',
          400
        );
      }
      
      // Step 5: Fraud check (non-critical)
      let fraudScore = 0;
      try {
        fraudScore = await this.checkFraud(fromAccountId, toAccountId, amount);
        if (fraudScore > 80) {
          throw new BankingError(
            'Transaction flagged for fraud review',
            'FRAUD_DETECTED',
            403
          );
        }
      } catch (error) {
        if (error.code === 'FRAUD_DETECTED') {
          throw error; // Rethrow fraud detection
        }
        // Fraud service down - log but continue
        console.warn('Fraud check failed, proceeding:', error.message);
      }
      
      // Step 6: Debit account (critical)
      try {
        await this.debitAccount(fromAccountId, amount);
        debitComplete = true;
        console.log(`✓ Debited ${amount} from ${fromAccountId}`);
      } catch (error) {
        console.error('Debit failed:', error);
        throw new BankingError(
          'Failed to debit account',
          'DEBIT_FAILED',
          500
        );
      }
      
      // Step 7: Credit account (critical with rollback)
      try {
        await this.creditAccount(toAccountId, amount);
        console.log(`✓ Credited ${amount} to ${toAccountId}`);
      } catch (error) {
        console.error('Credit failed, rolling back:', error);
        
        // Rollback: refund the debit
        try {
          await this.creditAccount(fromAccountId, amount);
          console.log('✓ Rollback successful');
        } catch (rollbackError) {
          console.error('❌ CRITICAL: Rollback failed!', rollbackError);
          await this.alertOps({
            severity: 'CRITICAL',
            transaction: transactionId,
            issue: 'Rollback failed',
            fromAccount: fromAccountId,
            amount
          });
        }
        
        throw new BankingError(
          'Failed to credit account',
          'CREDIT_FAILED',
          500
        );
      }
      
      // Step 8: Record transaction (best effort)
      try {
        await this.recordTransaction({
          transactionId,
          from: fromAccountId,
          to: toAccountId,
          amount,
          timestamp: new Date()
        });
      } catch (error) {
        // Log but don't fail transaction
        console.error('Failed to record transaction:', error);
        await this.queueForRetry(transactionId);
      }
      
      // Step 9: Send notifications (fire and forget)
      this.sendNotifications(fromAccountId, toAccountId, amount)
        .catch(error => console.warn('Notification failed:', error));
      
      return {
        success: true,
        transactionId,
        amount,
        timestamp: new Date()
      };
      
    } catch (error) {
      console.error('Transaction failed:', error);
      
      // Log to monitoring system
      await this.logError({
        transactionId,
        error: error.message,
        code: error.code,
        fromAccount: fromAccountId,
        toAccount: toAccountId,
        amount
      });
      
      return {
        success: false,
        transactionId,
        error: error.code || 'UNKNOWN_ERROR',
        message: error.message
      };
    }
  }
  
  // Helper: Retry mechanism
  async fetchAccountWithRetry(accountId, attempt = 1) {
    try {
      return await fetchAccount(accountId);
    } catch (error) {
      if (attempt >= this.maxRetries) {
        throw new BankingError(
          `Failed to fetch account after ${this.maxRetries} attempts`,
          'FETCH_FAILED',
          503
        );
      }
      
      console.log(`Retry ${attempt}/${this.maxRetries} for account ${accountId}`);
      await this.sleep(this.retryDelay * attempt);
      return this.fetchAccountWithRetry(accountId, attempt + 1);
    }
  }
  
  validateTransferInputs(fromAccountId, toAccountId, amount) {
    if (!fromAccountId || !toAccountId) {
      throw new Error('Account IDs are required');
    }
    if (fromAccountId === toAccountId) {
      throw new Error('Cannot transfer to same account');
    }
    if (amount <= 0) {
      throw new Error('Amount must be positive');
    }
    if (amount > 1000000) {
      throw new Error('Amount exceeds maximum transfer limit');
    }
  }
  
  generateTransactionId() {
    return `TXN-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
  
  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
  
  async getTodayTransactionTotal(accountId) {
    return 5000; // Placeholder
  }
  
  async checkFraud(from, to, amount) {
    return Math.random() * 100; // Placeholder
  }
  
  async debitAccount(accountId, amount) {
    return new Promise(resolve => setTimeout(resolve, 100));
  }
  
  async creditAccount(accountId, amount) {
    return new Promise(resolve => setTimeout(resolve, 100));
  }
  
  async recordTransaction(data) {
    return new Promise(resolve => setTimeout(resolve, 50));
  }
  
  async sendNotifications(from, to, amount) {
    return new Promise(resolve => setTimeout(resolve, 100));
  }
  
  async alertOps(data) {
    console.error('🚨 OPERATIONS ALERT:', data);
  }
  
  async queueForRetry(txnId) {
    console.log('Queued for retry:', txnId);
  }
  
  async logError(data) {
    console.error('Error logged:', data);
  }
}

// Helper functions
function fetchAccount(accountId) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (Math.random() > 0.1) {
        resolve({
          id: accountId,
          balance: 10000,
          dailyLimit: 50000
        });
      } else {
        reject(new Error('Database connection failed'));
      }
    }, 100);
  });
}

function processWithdrawal(account, amount) {
  return Promise.resolve({ success: true });
}

function getCachedAccount(accountId) {
  return { id: accountId, cached: true };
}

function getDefaultPreferences() {
  return { theme: 'light' };
}

function fetchCustomer(customerId) {
  return Promise.resolve({ id: customerId, name: 'John' });
}

function fetchPreferences(customerId) {
  return Promise.resolve({ theme: 'dark' });
}

function fetchAvatar(customerId) {
  return Promise.resolve('avatar.jpg');
}

function getBalance(account) {
  return Promise.resolve(10000);
}

function getTransactions(account) {
  return Promise.resolve([]);
}

function createAccount(accountId) {
  return Promise.resolve({ id: accountId, new: true });
}

// Usage
const service = new TransactionService();
service.transferMoney('ACC-001', 'ACC-002', 5000)
  .then(result => console.log('Transfer result:', result))
  .catch(error => console.error('Transfer error:', error));
```

**Best Practices for ENBD:**
1. Use try-catch for critical operations
2. Implement retry mechanisms for transient failures
3. Always rollback on partial failures
4. Use custom error classes for better error handling
5. Log all errors with context
6. Separate critical from non-critical operations
7. Implement timeout mechanisms
8. Alert operations team for critical failures

---

### Q15. Explain error-first callbacks vs Promises. How would you convert a callback-based API to Promise-based?

**Answer:**

**Error-First Callbacks** (Node.js convention):
- First parameter is always error (null if success)
- Second parameter is the result
- Used in older Node.js APIs

**Conversion Methods:**

```javascript
const fs = require('fs');
const { promisify } = require('util');

// === Method 1: Manual Promise Wrapper ===
function readFilePromise(filePath) {
  return new Promise((resolve, reject) => {
    fs.readFile(filePath, 'utf8', (err, data) => {
      if (err) {
        reject(err);
      } else {
        resolve(data);
      }
    });
  });
}

// === Method 2: Using util.promisify ===
const readFileAsync = promisify(fs.readFile);

// === Method 3: fs.promises (Built-in) ===
const fsPromises = require('fs').promises;

// Banking Example: Legacy Callback-based Database API
class LegacyBankingDB {
  // Old callback-based methods
  getAccount(accountId, callback) {
    setTimeout(() => {
      if (!accountId) {
        return callback(new Error('Account ID required'));
      }
      callback(null, {
        id: accountId,
        balance: 10000,
        type: 'savings'
      });
    }, 100);
  }
  
  updateBalance(accountId, newBalance, callback) {
    setTimeout(() => {
      if (newBalance < 0) {
        return callback(new Error('Balance cannot be negative'));
      }
      callback(null, { id: accountId, balance: newBalance });
    }, 100);
  }
  
  getTransactionHistory(accountId, limit, callback) {
    setTimeout(() => {
      callback(null, [
        { id: 'TXN-1', amount: 100 },
        { id: 'TXN-2', amount: -50 }
      ]);
    }, 100);
  }
}

// Modern Promise-based Wrapper
class ModernBankingDB {
  constructor() {
    this.legacy = new LegacyBankingDB();
  }
  
  // Method 1: Manual wrapping
  getAccount(accountId) {
    return new Promise((resolve, reject) => {
      this.legacy.getAccount(accountId, (err, account) => {
        if (err) reject(err);
        else resolve(account);
      });
    });
  }
  
  // Method 2: Generic promisify helper
  promisify(fn) {
    return (...args) => {
      return new Promise((resolve, reject) => {
        fn(...args, (err, result) => {
          if (err) reject(err);
          else resolve(result);
        });
      });
    };
  }
  
  // Use the helper
  updateBalance(accountId, newBalance) {
    const promisified = this.promisify(
      this.legacy.updateBalance.bind(this.legacy)
    );
    return promisified(accountId, newBalance);
  }
  
  // Method 3: Using util.promisify
  getTransactionHistory(accountId, limit) {
    const promisified = promisify(
      this.legacy.getTransactionHistory.bind(this.legacy)
    );
    return promisified(accountId, limit);
  }
}

// Usage Comparison
async function demonstrateUsage() {
  const modern = new ModernBankingDB();
  
  console.log('=== Old Callback Style ===');
  modern.legacy.getAccount('ACC-001', (err, account) => {
    if (err) {
      console.error('Error:', err);
      return;
    }
    console.log('Account:', account);
    
    modern.legacy.updateBalance('ACC-001', 15000, (err, updated) => {
      if (err) {
        console.error('Error:', err);
        return;
      }
      console.log('Updated:', updated);
      
      modern.legacy.getTransactionHistory('ACC-001', 10, (err, history) => {
        if (err) {
          console.error('Error:', err);
          return;
        }
        console.log('History:', history);
      });
    });
  });
  
  console.log('\n=== New Promise/Async Style ===');
  try {
    const account = await modern.getAccount('ACC-001');
    console.log('Account:', account);
    
    const updated = await modern.updateBalance('ACC-001', 15000);
    console.log('Updated:', updated);
    
    const history = await modern.getTransactionHistory('ACC-001', 10);
    console.log('History:', history);
  } catch (error) {
    console.error('Error:', error);
  }
}

// demonstrateUsage();
```

**Complete Banking Service Migration:**

```javascript
// Legacy callback-based service
class LegacyTransactionService {
  validateTransaction(txnData, callback) {
    setTimeout(() => {
      if (!txnData.amount || txnData.amount <= 0) {
        return callback(new Error('Invalid amount'));
      }
      callback(null, { ...txnData, validated: true });
    }, 50);
  }
  
  checkFraud(txnData, callback) {
    setTimeout(() => {
      const riskScore = Math.random() * 100;
      if (riskScore > 90) {
        return callback(new Error('High fraud risk detected'));
      }
      callback(null, { ...txnData, riskScore });
    }, 50);
  }
  
  processPayment(txnData, callback) {
    setTimeout(() => {
      callback(null, {
        ...txnData,
        transactionId: 'TXN-' + Date.now(),
        status: 'completed'
      });
    }, 50);
  }
  
  sendNotification(txnData, callback) {
    setTimeout(() => {
      callback(null, { notificationSent: true });
    }, 50);
  }
}

// Modern async/await service
class ModernTransactionService {
  constructor() {
    this.legacy = new LegacyTransactionService();
    
    // Promisify all methods
    this.validateTransaction = promisify(
      this.legacy.validateTransaction.bind(this.legacy)
    );
    this.checkFraud = promisify(
      this.legacy.checkFraud.bind(this.legacy)
    );
    this.processPayment = promisify(
      this.legacy.processPayment.bind(this.legacy)
    );
    this.sendNotification = promisify(
      this.legacy.sendNotification.bind(this.legacy)
    );
  }
  
  // New clean async method
  async executeTransaction(txnData) {
    try {
      // Sequential with clean error handling
      const validated = await this.validateTransaction(txnData);
      const fraudChecked = await this.checkFraud(validated);
      const processed = await this.processPayment(fraudChecked);
      
      // Fire and forget notification
      this.sendNotification(processed).catch(err => {
        console.warn('Notification failed:', err);
      });
      
      return processed;
    } catch (error) {
      console.error('Transaction failed:', error);
      throw error;
    }
  }
  
  // Parallel operations
  async executeTransactionParallel(txnData) {
    try {
      // Run validation and fraud check in parallel
      const [validated, fraudCheck] = await Promise.all([
        this.validateTransaction(txnData),
        this.checkFraud(txnData)
      ]);
      
      // Combine results
      const combined = { ...validated, ...fraudCheck };
      
      // Process payment
      const processed = await this.processPayment(combined);
      
      return processed;
    } catch (error) {
      console.error('Transaction failed:', error);
      throw error;
    }
  }
}

// Generic conversion utility
class CallbackToPromiseConverter {
  /**
   * Converts a callback-based function to promise-based
   * @param {Function} fn - Callback function
   * @param {Object} context - 'this' context
   */
  static convert(fn, context = null) {
    return (...args) => {
      return new Promise((resolve, reject) => {
        const callback = (err, ...results) => {
          if (err) reject(err);
          else resolve(results.length <= 1 ? results[0] : results);
        };
        
        fn.call(context, ...args, callback);
      });
    };
  }
  
  /**
   * Converts entire object/class methods
   * @param {Object} obj - Object with callback methods
   * @param {Array} methods - Method names to convert
   */
  static convertObject(obj, methods) {
    const converted = {};
    
    methods.forEach(method => {
      if (typeof obj[method] === 'function') {
        converted[method] = this.convert(obj[method], obj);
      }
    });
    
    return converted;
  }
}

// Usage
const legacy = new LegacyTransactionService();
const promisified = CallbackToPromiseConverter.convertObject(
  legacy,
  ['validateTransaction', 'checkFraud', 'processPayment', 'sendNotification']
);

async function testConversion() {
  try {
    const txn = { amount: 1000, from: 'ACC1', to: 'ACC2' };
    
    const validated = await promisified.validateTransaction(txn);
    console.log('Validated:', validated);
    
    const checked = await promisified.checkFraud(validated);
    console.log('Fraud Check:', checked);
    
    const processed = await promisified.processPayment(checked);
    console.log('Processed:', processed);
    
  } catch (error) {
    console.error('Error:', error);
  }
}

// testConversion();
```

**Advanced: Handling Multiple Callback Patterns:**

```javascript
// Pattern 1: Standard error-first callback
function standardCallback(arg, callback) {
  callback(null, result);
  // or
  callback(error);
}

// Pattern 2: Multiple results
function multipleResults(arg, callback) {
  callback(null, result1, result2, result3);
}

// Pattern 3: Success/error callbacks (jQuery style)
function jQueryStyle(arg, successCallback, errorCallback) {
  successCallback(result);
  // or
  errorCallback(error);
}

// Universal converter
class UniversalConverter {
  // For standard error-first
  static errorFirst(fn, context) {
    return (...args) => {
      return new Promise((resolve, reject) => {
        fn.call(context, ...args, (err, result) => {
          if (err) reject(err);
          else resolve(result);
        });
      });
    };
  }
  
  // For multiple results
  static multipleResults(fn, context) {
    return (...args) => {
      return new Promise((resolve, reject) => {
        fn.call(context, ...args, (err, ...results) => {
          if (err) reject(err);
          else resolve(results);
        });
      });
    };
  }
  
  // For success/error callbacks
  static successError(fn, context) {
    return (...args) => {
      return new Promise((resolve, reject) => {
        fn.call(
          context,
          ...args,
          (result) => resolve(result),
          (error) => reject(error)
        );
      });
    };
  }
}

// Example: Converting XMLHttpRequest to Promise
function httpRequest(url) {
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();
    xhr.open('GET', url);
    
    xhr.onload = () => {
      if (xhr.status >= 200 && xhr.status < 300) {
        resolve(JSON.parse(xhr.responseText));
      } else {
        reject(new Error(`HTTP ${xhr.status}: ${xhr.statusText}`));
      }
    };
    
    xhr.onerror = () => reject(new Error('Network error'));
    xhr.ontimeout = () => reject(new Error('Request timeout'));
    
    xhr.send();
  });
}

// Usage
async function fetchData() {
  try {
    const data = await httpRequest('https://api.example.com/data');
    console.log(data);
  } catch (error) {
    console.error('Request failed:', error);
  }
}
```

**Key Takeaways for ENBD:**
1. Use `util.promisify` for Node.js callback functions
2. Create wrapper classes for legacy systems
3. Build generic converters for consistent APIs
4. Always handle errors properly in conversions
5. Test thoroughly when migrating legacy code
6. Document which methods are promisified
7. Consider using `fs.promises` for file operations

---

**🎯 Progress Update:**
✅ **Completed Q11-Q15** (5 questions)

**Remaining for Section 2:** Q16-Q20 (5 more questions)

---

### Q16. What is Promise.allSettled() and how does it differ from Promise.all()? When would you use it in banking?

**Answer:**

**Key Differences:**

| Feature | Promise.all() | Promise.allSettled() |
|---------|--------------|---------------------|
| **Behavior** | Fails fast on first rejection | Waits for all to settle |
| **Return** | Array of values | Array of {status, value/reason} |
| **Use Case** | All must succeed | Need all results |
| **Error Handling** | Rejects immediately | Never rejects |

**Banking Example: Multi-Account Operations**

```javascript
// Scenario: Process end-of-day operations for all customer accounts

class EndOfDayProcessor {
  async processAllAccounts(accountIds) {
    console.log(`Processing ${accountIds.length} accounts...`);
    
    // Using Promise.allSettled - continues even if some fail
    const results = await Promise.allSettled(
      accountIds.map(id => this.processAccount(id))
    );
    
    // Categorize results
    const summary = {
      successful: [],
      failed: [],
      total: results.length
    };
    
    results.forEach((result, index) => {
      const accountId = accountIds[index];
      
      if (result.status === 'fulfilled') {
        summary.successful.push({
          accountId,
          data: result.value
        });
        console.log(`✓ Account ${accountId}: Success`);
      } else {
        summary.failed.push({
          accountId,
          error: result.reason.message
        });
        console.error(`✗ Account ${accountId}: ${result.reason.message}`);
      }
    });
    
    return summary;
  }
  
  async processAccount(accountId) {
    // Simulate account processing with random failures
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        if (Math.random() > 0.2) {
          resolve({
            accountId,
            balance: Math.random() * 10000,
            interestCalculated: true,
            timestamp: new Date()
          });
        } else {
          reject(new Error(`Account ${accountId} processing failed`));
        }
      }, Math.random() * 1000);
    });
  }
}

// Usage
const processor = new EndOfDayProcessor();
const accounts = ['ACC-001', 'ACC-002', 'ACC-003', 'ACC-004', 'ACC-005'];

processor.processAllAccounts(accounts)
  .then(summary => {
    console.log('\n=== Summary ===');
    console.log(`Total: ${summary.total}`);
    console.log(`Successful: ${summary.successful.length}`);
    console.log(`Failed: ${summary.failed.length}`);
    
    // Generate report even if some failed
    console.log('\nFailed accounts need manual review:');
    summary.failed.forEach(fail => {
      console.log(`- ${fail.accountId}: ${fail.error}`);
    });
  });
```

**Real-World Banking Scenarios:**

```javascript
// 1. Multi-Channel Notification System
async function sendTransactionNotifications(customerId, transaction) {
  const channels = [
    sendEmail(customerId, transaction),
    sendSMS(customerId, transaction),
    sendPushNotification(customerId, transaction),
    sendWhatsApp(customerId, transaction)
  ];
  
  // Send via all channels, track which succeeded
  const results = await Promise.allSettled(channels);
  
  const sentChannels = results
    .filter(r => r.status === 'fulfilled')
    .map((r, i) => ['Email', 'SMS', 'Push', 'WhatsApp'][i]);
  
  console.log(`Notification sent via: ${sentChannels.join(', ')}`);
  
  // At least one channel must succeed
  if (sentChannels.length === 0) {
    throw new Error('All notification channels failed');
  }
  
  return { sentChannels, results };
}

// 2. Reconciliation Across Multiple Banks
async function reconcileTransactions(date) {
  const banks = ['VISA', 'Mastercard', 'UnionPay', 'AmEx'];
  
  const results = await Promise.allSettled(
    banks.map(bank => fetchBankTransactions(bank, date))
  );
  
  const reconciliation = {
    date,
    banks: {}
  };
  
  results.forEach((result, index) => {
    const bank = banks[index];
    
    if (result.status === 'fulfilled') {
      reconciliation.banks[bank] = {
        status: 'reconciled',
        transactions: result.value.length,
        data: result.value
      };
    } else {
      reconciliation.banks[bank] = {
        status: 'failed',
        error: result.reason.message,
        requiresManualReview: true
      };
    }
  });
  
  return reconciliation;
}

// 3. Credit Check from Multiple Bureaus
async function performCreditCheck(customerId) {
  const bureaus = [
    checkEquifax(customerId),
    checkExperian(customerId),
    checkTransUnion(customerId)
  ];
  
  const results = await Promise.allSettled(bureaus);
  
  // Use available scores even if one bureau fails
  const scores = results
    .filter(r => r.status === 'fulfilled')
    .map(r => r.value.score);
  
  if (scores.length === 0) {
    throw new Error('All credit bureaus unavailable');
  }
  
  // Calculate average from available scores
  const averageScore = scores.reduce((a, b) => a + b, 0) / scores.length;
  
  return {
    scores,
    averageScore,
    bureausChecked: scores.length,
    bureausFailed: results.length - scores.length
  };
}

// Helper functions
function sendEmail(customerId, transaction) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      Math.random() > 0.2 
        ? resolve({ channel: 'email', sent: true })
        : reject(new Error('Email service down'));
    }, 100);
  });
}

function sendSMS(customerId, transaction) {
  return new Promise((resolve) => {
    setTimeout(() => resolve({ channel: 'sms', sent: true }), 150);
  });
}

function sendPushNotification(customerId, transaction) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      Math.random() > 0.3
        ? resolve({ channel: 'push', sent: true })
        : reject(new Error('Push notification failed'));
    }, 80);
  });
}

function sendWhatsApp(customerId, transaction) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      Math.random() > 0.5
        ? resolve({ channel: 'whatsapp', sent: true })
        : reject(new Error('WhatsApp not configured'));
    }, 120);
  });
}

function fetchBankTransactions(bank, date) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (Math.random() > 0.25) {
        resolve([
          { id: `${bank}-TXN-001`, amount: 100 },
          { id: `${bank}-TXN-002`, amount: 200 }
        ]);
      } else {
        reject(new Error(`${bank} API unavailable`));
      }
    }, Math.random() * 500);
  });
}

function checkEquifax(customerId) {
  return new Promise((resolve) => {
    setTimeout(() => resolve({ bureau: 'Equifax', score: 720 }), 300);
  });
}

function checkExperian(customerId) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      Math.random() > 0.3
        ? resolve({ bureau: 'Experian', score: 710 })
        : reject(new Error('Experian timeout'));
    }, 400);
  });
}

function checkTransUnion(customerId) {
  return new Promise((resolve) => {
    setTimeout(() => resolve({ bureau: 'TransUnion', score: 730 }), 250);
  });
}
```

**Comparison Example:**

```javascript
// Using Promise.all() - fails if ANY operation fails
async function usingPromiseAll() {
  try {
    const results = await Promise.all([
      operation1(), // succeeds
      operation2(), // fails
      operation3()  // never executed
    ]);
    console.log(results); // Never reaches here
  } catch (error) {
    console.error('Failed:', error);
    // Lost results from successful operations!
  }
}

// Using Promise.allSettled() - gets all results
async function usingPromiseAllSettled() {
  const results = await Promise.allSettled([
    operation1(), // succeeds
    operation2(), // fails
    operation3()  // still executes
  ]);
  
  results.forEach((result, index) => {
    if (result.status === 'fulfilled') {
      console.log(`Operation ${index + 1}: Success -`, result.value);
    } else {
      console.error(`Operation ${index + 1}: Failed -`, result.reason);
    }
  });
  
  // Can still use successful results!
}

function operation1() {
  return Promise.resolve('Success 1');
}

function operation2() {
  return Promise.reject(new Error('Failed 2'));
}

function operation3() {
  return Promise.resolve('Success 3');
}
```

**When to Use Each:**

```javascript
// Use Promise.all() when:
// - All operations MUST succeed
// - Need atomic behavior (all or nothing)
// - Fast-fail is acceptable
async function criticalTransfer(fromAccount, toAccount, amount) {
  // Both must succeed or entire transaction fails
  const [debit, credit] = await Promise.all([
    debitAccount(fromAccount, amount),
    creditAccount(toAccount, amount)
  ]);
  return { debit, credit };
}

// Use Promise.allSettled() when:
// - Need all results regardless of success/failure
// - Partial success is acceptable
// - Need comprehensive reporting
async function batchProcessing(items) {
  const results = await Promise.allSettled(
    items.map(item => processItem(item))
  );
  
  // Generate detailed report
  return {
    total: results.length,
    successful: results.filter(r => r.status === 'fulfilled').length,
    failed: results.filter(r => r.status === 'rejected').length,
    details: results
  };
}

function debitAccount(account, amount) {
  return Promise.resolve({ account, debited: amount });
}

function creditAccount(account, amount) {
  return Promise.resolve({ account, credited: amount });
}

function processItem(item) {
  return Promise.resolve(item);
}
```

**Key Takeaways for ENBD:**
- Use `Promise.allSettled()` for batch operations
- Critical for reporting and reconciliation
- Ensures no results are lost due to partial failures
- Better for multi-channel operations
- Always analyze the results array for proper handling

---

### Q17. Explain async iteration and async generators. Provide a banking data streaming example.

**Answer:**

**Async Iteration** allows iterating over asynchronous data sources using `for await...of`.

**Async Generators** use `async function*` to yield values asynchronously.

**Banking Example: Transaction Stream Processing**

```javascript
// Async Generator: Stream transactions from database
async function* streamTransactions(accountId, startDate, endDate) {
  let offset = 0;
  const batchSize = 100;
  
  while (true) {
    // Fetch batch from database
    const batch = await fetchTransactionBatch(accountId, offset, batchSize);
    
    if (batch.length === 0) {
      break; // No more transactions
    }
    
    // Yield each transaction
    for (const transaction of batch) {
      yield transaction;
    }
    
    offset += batchSize;
    
    // Simulate rate limiting
    await sleep(100);
  }
}

// Async iteration to process stream
async function processTransactionStream(accountId) {
  console.log('Processing transaction stream...\n');
  
  let totalAmount = 0;
  let count = 0;
  
  // For await...of consumes async generator
  for await (const txn of streamTransactions(accountId, new Date('2024-01-01'), new Date())) {
    console.log(`Transaction ${txn.id}: ${txn.amount} ${txn.currency}`);
    totalAmount += txn.amount;
    count++;
  }
  
  console.log(`\nProcessed ${count} transactions`);
  console.log(`Total amount: ${totalAmount}`);
}

// Helper: Simulate database fetch
async function fetchTransactionBatch(accountId, offset, limit) {
  return new Promise(resolve => {
    setTimeout(() => {
      // Simulate fetching data
      if (offset >= 250) {
        resolve([]); // No more data
      } else {
        const batch = Array.from({ length: Math.min(limit, 250 - offset) }, (_, i) => ({
          id: `TXN-${offset + i + 1}`,
          accountId,
          amount: Math.random() * 1000,
          currency: 'AED',
          date: new Date()
        }));
        resolve(batch);
      }
    }, 50);
  });
}

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// processTransactionStream('ACC-12345');
```

**Real-World Banking Applications:**

```javascript
// 1. Large Report Generation with Progress Updates
async function* generateAnnualReport(customerId, year) {
  yield { type: 'status', message: 'Fetching account information...' };
  const accounts = await fetchCustomerAccounts(customerId);
  
  yield { type: 'status', message: 'Loading transactions...' };
  const transactions = await fetchYearTransactions(customerId, year);
  
  yield { type: 'status', message: 'Calculating statistics...' };
  const stats = calculateStatistics(transactions);
  
  yield { type: 'status', message: 'Generating charts...' };
  const charts = await generateCharts(stats);
  
  yield { type: 'status', message: 'Compiling report...' };
  const report = compileReport(accounts, transactions, stats, charts);
  
  yield { type: 'complete', data: report };
}

async function processReportGeneration(customerId, year) {
  console.log('Generating annual report...\n');
  
  for await (const update of generateAnnualReport(customerId, year)) {
    if (update.type === 'status') {
      console.log(`⏳ ${update.message}`);
    } else if (update.type === 'complete') {
      console.log(`✓ Report generated successfully`);
      return update.data;
    }
  }
}

// 2. Real-time Transaction Monitoring
async function* monitorTransactions(accountId) {
  while (true) {
    // Fetch latest transactions
    const transactions = await fetchLatestTransactions(accountId);
    
    for (const txn of transactions) {
      // Check for suspicious activity
      if (txn.amount > 10000) {
        yield {
          type: 'alert',
          severity: 'high',
          transaction: txn,
          reason: 'Large transaction detected'
        };
      }
      
      if (await checkFraudRules(txn)) {
        yield {
          type: 'alert',
          severity: 'critical',
          transaction: txn,
          reason: 'Fraud pattern detected'
        };
      }
      
      yield {
        type: 'transaction',
        data: txn
      };
    }
    
    // Poll every 5 seconds
    await sleep(5000);
  }
}

async function startMonitoring(accountId) {
  console.log('Starting transaction monitoring...\n');
  
  const monitor = monitorTransactions(accountId);
  
  // Monitor for 30 seconds
  const timeout = setTimeout(() => {
    console.log('\nMonitoring stopped');
  }, 30000);
  
  for await (const event of monitor) {
    if (event.type === 'alert') {
      console.log(`🚨 ${event.severity.toUpperCase()}: ${event.reason}`);
      console.log(`   Transaction: ${event.transaction.id} - ${event.transaction.amount}`);
    } else {
      console.log(`✓ Transaction: ${event.data.id}`);
    }
  }
  
  clearTimeout(timeout);
}

// 3. Async Pipeline Processing
async function* readTransactionFile(filePath) {
  // Simulate reading large file in chunks
  const chunks = [
    'TXN-001,1000,ACC-123\n',
    'TXN-002,2000,ACC-456\n',
    'TXN-003,3000,ACC-789\n'
  ];
  
  for (const chunk of chunks) {
    await sleep(100);
    yield chunk;
  }
}

async function* parseTransactions(source) {
  for await (const chunk of source) {
    const lines = chunk.split('\n').filter(line => line.trim());
    for (const line of lines) {
      const [id, amount, accountId] = line.split(',');
      yield {
        id,
        amount: parseFloat(amount),
        accountId
      };
    }
  }
}

async function* validateTransactions(source) {
  for await (const txn of source) {
    if (txn.amount > 0 && txn.accountId) {
      yield { ...txn, valid: true };
    } else {
      console.warn(`Invalid transaction: ${txn.id}`);
    }
  }
}

async function* enrichTransactions(source) {
  for await (const txn of source) {
    const accountDetails = await fetchAccountDetails(txn.accountId);
    yield {
      ...txn,
      accountName: accountDetails.name,
      accountType: accountDetails.type
    };
  }
}

// Compose pipeline
async function processTransactionPipeline(filePath) {
  const pipeline = enrichTransactions(
    validateTransactions(
      parseTransactions(
        readTransactionFile(filePath)
      )
    )
  );
  
  const results = [];
  for await (const enrichedTxn of pipeline) {
    console.log('Processed:', enrichedTxn);
    results.push(enrichedTxn);
  }
  
  return results;
}

// Helper functions
async function fetchCustomerAccounts(customerId) {
  return Promise.resolve([{ id: 'ACC-1', type: 'savings' }]);
}

async function fetchYearTransactions(customerId, year) {
  return Promise.resolve([
    { id: 'TXN-1', amount: 100 },
    { id: 'TXN-2', amount: 200 }
  ]);
}

function calculateStatistics(transactions) {
  return { total: transactions.length };
}

async function generateCharts(stats) {
  return Promise.resolve({ chart: 'data' });
}

function compileReport(accounts, transactions, stats, charts) {
  return { accounts, transactions, stats, charts };
}

async function fetchLatestTransactions(accountId) {
  return Promise.resolve([
    { id: 'TXN-' + Date.now(), amount: Math.random() * 20000, accountId }
  ]);
}

async function checkFraudRules(txn) {
  return Promise.resolve(txn.amount > 15000);
}

async function fetchAccountDetails(accountId) {
  return Promise.resolve({ name: 'John Doe', type: 'checking' });
}
```

**Advanced Pattern: Async Iterator with Cleanup**

```javascript
class TransactionStream {
  constructor(accountId) {
    this.accountId = accountId;
    this.connection = null;
  }
  
  async *[Symbol.asyncIterator]() {
    // Setup
    this.connection = await this.connect();
    console.log('Connected to database');
    
    try {
      let hasMore = true;
      let cursor = 0;
      
      while (hasMore) {
        const batch = await this.fetchBatch(cursor);
        
        if (batch.length === 0) {
          hasMore = false;
        } else {
          for (const txn of batch) {
            yield txn;
          }
          cursor += batch.length;
        }
      }
    } finally {
      // Cleanup
      await this.disconnect();
      console.log('Disconnected from database');
    }
  }
  
  async connect() {
    return new Promise(resolve => {
      setTimeout(() => resolve({ connected: true }), 100);
    });
  }
  
  async disconnect() {
    return new Promise(resolve => {
      setTimeout(() => resolve(), 100);
    });
  }
  
  async fetchBatch(cursor) {
    return new Promise(resolve => {
      setTimeout(() => {
        if (cursor >= 50) {
          resolve([]);
        } else {
          resolve(Array.from({ length: 10 }, (_, i) => ({
            id: `TXN-${cursor + i + 1}`,
            amount: Math.random() * 1000
          })));
        }
      }, 50);
    });
  }
}

// Usage with automatic cleanup
async function useTransactionStream() {
  const stream = new TransactionStream('ACC-12345');
  
  try {
    for await (const txn of stream) {
      console.log(`Processing: ${txn.id}`);
      
      // Can break early - cleanup still happens
      if (txn.id === 'TXN-25') {
        console.log('Stopping early');
        break;
      }
    }
  } catch (error) {
    console.error('Stream error:', error);
  }
  // Cleanup is automatic even on break or error
}

// useTransactionStream();
```

**Key Takeaways for ENBD:**
- Use async generators for large dataset streaming
- Memory efficient - processes data in chunks
- Perfect for report generation with progress updates
- Supports early termination with cleanup
- Ideal for real-time data monitoring
- Can build processing pipelines

---

### Q18. What are microtasks and macrotasks? How do they affect async execution order?

**Answer:**

**Microtasks vs Macrotasks:**

**Microtasks** (executed first, higher priority):
- `Promise.then/catch/finally`
- `process.nextTick()` (Node.js specific, highest priority)
- `queueMicrotask()`
- `MutationObserver` (browser)

**Macrotasks** (executed after microtasks):
- `setTimeout/setInterval`
- `setImmediate` (Node.js)
- I/O operations
- `requestAnimationFrame` (browser)

**Execution Order:**

```
1. Execute synchronous code
2. Execute microtask queue (until empty)
3. Execute one macrotask
4. Execute microtask queue again (until empty)
5. Repeat steps 3-4
```

**Visual Example:**

```javascript
console.log('1: Start');

setTimeout(() => {
  console.log('2: setTimeout (macrotask)');
}, 0);

Promise.resolve().then(() => {
  console.log('3: Promise (microtask)');
});

process.nextTick(() => {
  console.log('4: nextTick (microtask, highest priority)');
});

queueMicrotask(() => {
  console.log('5: queueMicrotask (microtask)');
});

setImmediate(() => {
  console.log('6: setImmediate (macrotask)');
});

console.log('7: End');

// Output Order:
// 1: Start
// 7: End
// 4: nextTick (microtask, highest priority)
// 3: Promise (microtask)
// 5: queueMicrotask (microtask)
// 6: setImmediate (macrotask)
// 2: setTimeout (macrotask)
```

**Banking Example: Transaction Processing Order**

```javascript
class TransactionProcessor {
  processTransaction(txnData) {
    console.log('🔵 1. Processing transaction:', txnData.id);
    
    // Synchronous validation
    if (!this.validateSync(txnData)) {
      throw new Error('Invalid transaction');
    }
    
    // Microtask - High priority fraud check
    process.nextTick(() => {
      console.log('🟢 2. Fraud check (nextTick):', txnData.id);
      this.performFraudCheck(txnData);
    });
    
    // Microtask - Promise-based database operation
    Promise.resolve().then(() => {
      console.log('🟢 3. Database write (Promise):', txnData.id);
      return this.writeToDatabase(txnData);
    }).then(() => {
      console.log('🟢 4. Database write complete:', txnData.id);
    });
    
    // Microtask - Another promise operation
    queueMicrotask(() => {
      console.log('🟢 5. Update cache (queueMicrotask):', txnData.id);
      this.updateCache(txnData);
    });
    
    // Macrotask - Non-critical notification (can be delayed)
    setTimeout(() => {
      console.log('🔴 6. Send notification (setTimeout):', txnData.id);
      this.sendNotification(txnData);
    }, 0);
    
    // Macrotask - Audit log (lower priority)
    setImmediate(() => {
      console.log('🔴 7. Write audit log (setImmediate):', txnData.id);
      this.writeAuditLog(txnData);
    });
    
    console.log('🔵 8. Transaction queued:', txnData.id);
  }
  
  validateSync(txnData) {
    return txnData.amount > 0;
  }
  
  performFraudCheck(txnData) {
    // High priority check
  }
  
  writeToDatabase(txnData) {
    return Promise.resolve();
  }
  
  updateCache(txnData) {
    // Update cache
  }
  
  sendNotification(txnData) {
    // Send notification
  }
  
  writeAuditLog(txnData) {
    // Write audit log
  }
}

const processor = new TransactionProcessor();
// processor.processTransaction({ id: 'TXN-001', amount: 1000 });

/* Output:
🔵 1. Processing transaction: TXN-001
🔵 8. Transaction queued: TXN-001
🟢 2. Fraud check (nextTick): TXN-001
🟢 3. Database write (Promise): TXN-001
🟢 5. Update cache (queueMicrotask): TXN-001
🟢 4. Database write complete: TXN-001
🔴 7. Write audit log (setImmediate): TXN-001
🔴 6. Send notification (setTimeout): TXN-001
*/
```

**Complex Nesting Example:**

```javascript
console.log('Start');

setTimeout(() => {
  console.log('setTimeout 1');
  
  Promise.resolve().then(() => {
    console.log('Promise inside setTimeout 1');
  });
  
  process.nextTick(() => {
    console.log('nextTick inside setTimeout 1');
  });
}, 0);

Promise.resolve().then(() => {
  console.log('Promise 1');
  
  setTimeout(() => {
    console.log('setTimeout inside Promise 1');
  }, 0);
  
  return Promise.resolve();
}).then(() => {
  console.log('Promise 2');
});

process.nextTick(() => {
  console.log('nextTick 1');
  
  process.nextTick(() => {
    console.log('nextTick nested');
  });
});

console.log('End');

/* Output:
Start
End
nextTick 1
nextTick nested
Promise 1
Promise 2
setTimeout 1
nextTick inside setTimeout 1
Promise inside setTimeout 1
setTimeout inside Promise 1
*/
```

**Real-World Banking Application:**

```javascript
class PaymentGateway {
  async processPayment(paymentData) {
    console.log('💳 Starting payment processing...\n');
    
    // 1. Immediate synchronous validation
    console.log('Step 1: Validate payment data (synchronous)');
    this.validatePaymentData(paymentData);
    
    // 2. High-priority microtask - Security check
    process.nextTick(() => {
      console.log('Step 2: Security validation (nextTick - highest priority)');
      this.validateSecurity(paymentData);
    });
    
    // 3. Promise microtask - Check account balance
    const balancePromise = Promise.resolve().then(() => {
      console.log('Step 3: Check balance (Promise microtask)');
      return this.checkBalance(paymentData.accountId, paymentData.amount);
    });
    
    // 4. Another microtask - Fraud detection
    queueMicrotask(() => {
      console.log('Step 4: Run fraud detection (queueMicrotask)');
      this.detectFraud(paymentData);
    });
    
    // Wait for critical microtasks
    await balancePromise;
    
    // 5. Process payment (next tick to ensure all checks done)
    await new Promise(resolve => {
      process.nextTick(async () => {
        console.log('Step 5: Process payment (nextTick)');
        await this.executePayment(paymentData);
        resolve();
      });
    });
    
    // 6. Macrotask - Update analytics (non-critical)
    setImmediate(() => {
      console.log('Step 6: Update analytics (setImmediate)');
      this.updateAnalytics(paymentData);
    });
    
    // 7. Macrotask - Send confirmation (can be slightly delayed)
    setTimeout(() => {
      console.log('Step 7: Send confirmation email (setTimeout)');
      this.sendConfirmation(paymentData);
    }, 0);
    
    // 8. Final microtask - Cache update
    queueMicrotask(() => {
      console.log('Step 8: Update cache (queueMicrotask)');
      this.updatePaymentCache(paymentData);
    });
    
    console.log('💳 Payment processing initiated\n');
  }
  
  validatePaymentData(data) {
    if (!data.amount || !data.accountId) {
      throw new Error('Invalid payment data');
    }
  }
  
  validateSecurity(data) {
    // Validate security tokens
  }
  
  checkBalance(accountId, amount) {
    return Promise.resolve(true);
  }
  
  detectFraud(data) {
    // Run fraud detection
  }
  
  executePayment(data) {
    return Promise.resolve();
  }
  
  updateAnalytics(data) {
    // Update analytics
  }
  
  sendConfirmation(data) {
    // Send confirmation
  }
  
  updatePaymentCache(data) {
    // Update cache
  }
}

const gateway = new PaymentGateway();
// gateway.processPayment({ accountId: 'ACC-001', amount: 1000 });
```

**Performance Implications:**

```javascript
// ❌ BAD: Creating too many microtasks can starve macrotasks
function badMicrotaskLoop() {
  let count = 0;
  
  function scheduleMicrotask() {
    if (count < 1000000) {
      count++;
      Promise.resolve().then(scheduleMicrotask);
    }
  }
  
  scheduleMicrotask();
  
  // This macrotask may never execute!
  setTimeout(() => {
    console.log('This might never run!');
  }, 0);
}

// ✅ GOOD: Balance microtasks and macrotasks
function goodTaskScheduling() {
  let count = 0;
  const batchSize = 100;
  
  function processBatch() {
    for (let i = 0; i < batchSize && count < 1000000; i++) {
      count++;
      // Process item
    }
    
    if (count < 1000000) {
      // Use setImmediate to allow other tasks to run
      setImmediate(processBatch);
    }
  }
  
  processBatch();
  
  setTimeout(() => {
    console.log('This will execute between batches');
  }, 0);
}
```

**Debugging Tool:**

```javascript
class TaskMonitor {
  constructor() {
    this.microtaskCount = 0;
    this.macrotaskCount = 0;
  }
  
  trackMicrotask(name, fn) {
    this.microtaskCount++;
    const id = this.microtaskCount;
    
    queueMicrotask(() => {
      console.log(`[Microtask ${id}] ${name}`);
      fn();
    });
  }
  
  trackMacrotask(name, fn, delay = 0) {
    this.macrotaskCount++;
    const id = this.macrotaskCount;
    
    setTimeout(() => {
      console.log(`[Macrotask ${id}] ${name}`);
      fn();
    }, delay);
  }
  
  reset() {
    this.microtaskCount = 0;
    this.macrotaskCount = 0;
  }
}

const monitor = new TaskMonitor();

console.log('Execution started');

monitor.trackMicrotask('Critical validation', () => {
  console.log('  Validating transaction');
});

monitor.trackMacrotask('Send notification', () => {
  console.log('  Notification sent');
});

monitor.trackMicrotask('Update database', () => {
  console.log('  Database updated');
});

console.log('Execution queued');
```

**Key Takeaways for ENBD:**
- Use microtasks for critical, high-priority operations (fraud checks, validation)
- Use macrotasks for non-critical operations (notifications, analytics)
- Beware of microtask starvation - can block I/O
- `process.nextTick()` has highest priority - use sparingly
- Promises are microtasks - execute before timeouts
- Balance task types for optimal performance

---

### Q19. How do you handle promise rejection tracking? Explain unhandledRejection events.

**Answer:**

**Unhandled Promise Rejections** occur when a Promise is rejected but no `.catch()` handler is attached.

**Why Critical for Banking:**
- Prevents silent failures
- Ensures all errors are logged
- Maintains audit trail
- Prevents data inconsistency
- Required for compliance

**Handling Unhandled Rejections:**

```javascript
// Global handler for unhandled rejections
process.on('unhandledRejection', (reason, promise) => {
  console.error('🚨 Unhandled Rejection at:', promise);
  console.error('Reason:', reason);
  
  // Log to monitoring system
  logToMonitoring({
    type: 'UNHANDLED_REJECTION',
    reason: reason,
    stack: reason?.stack,
    timestamp: new Date()
  });
  
  // For critical banking systems, you might want to exit
  // process.exit(1);
});

// Handle rejectionHandled (when catch is added later)
process.on('rejectionHandled', (promise) => {
  console.log('✓ Rejection was handled (late):', promise);
});

function logToMonitoring(data) {
  console.log('Logged to monitoring:', data);
}
```

**Banking Example: Transaction Error Tracking**

```javascript
class TransactionErrorTracker {
  constructor() {
    this.unhandledErrors = new Map();
    this.setupErrorHandlers();
  }
  
  setupErrorHandlers() {
    process.on('unhandledRejection', (reason, promise) => {
      const errorId = this.generateErrorId();
      
      const errorInfo = {
        id: errorId,
        reason: reason,
        message: reason?.message,
        stack: reason?.stack,
        timestamp: new Date(),
        handled: false,
        promise: promise
      };
      
      this.unhandledErrors.set(promise, errorInfo);
      
      console.error(`\n🚨 CRITICAL: Unhandled rejection detected`);
      console.error(`Error ID: ${errorId}`);
      console.error(`Message: ${errorInfo.message}`);
      console.error(`Time: ${errorInfo.timestamp.toISOString()}\n`);
      
      // Alert operations team
      this.alertOperations(errorInfo);
      
      // Log to database
      this.logToDatabase(errorInfo);
      
      // In production, might want to gracefully shutdown
      // this.gracefulShutdown(errorInfo);
    });
    
    process.on('rejectionHandled', (promise) => {
      const errorInfo = this.unhandledErrors.get(promise);
      
      if (errorInfo) {
        errorInfo.handled = true;
        errorInfo.handledAt = new Date();
        
        console.log(`\n✓ Late rejection handling for Error ID: ${errorInfo.id}`);
        console.log(`Handled after: ${errorInfo.handledAt - errorInfo.timestamp}ms\n`);
        
        this.unhandledErrors.delete(promise);
      }
    });
  }
  
  generateErrorId() {
    return `ERR-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
  
  alertOperations(errorInfo) {
    // Send to PagerDuty, Slack, etc.
    console.log(`📧 Alert sent to operations team`);
  }
  
  async logToDatabase(errorInfo) {
    // Log to database for audit trail
    console.log(`💾 Error logged to database`);
  }
  
  gracefulShutdown(errorInfo) {
    console.log(`\n🛑 Initiating graceful shutdown...`);
    
    // Give time for in-flight requests to complete
    setTimeout(() => {
      console.log('Exiting process');
      process.exit(1);
    }, 5000);
  }
  
  getUnhandledErrors() {
    return Array.from(this.unhandledErrors.values());
  }
}

// Initialize tracker
const errorTracker = new TransactionErrorTracker();

// Example: Unhandled rejection
async function processPaymentWithUnhandledError() {
  // This promise is never caught!
  const payment = processPayment({ amount: -1000 });
  
  console.log('Payment processing started (but no error handling!)');
}

function processPayment(data) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (data.amount < 0) {
        reject(new Error('Invalid amount: cannot be negative'));
      } else {
        resolve({ status: 'success' });
      }
    }, 100);
  });
}

// processPaymentWithUnhandledError();

// Example: Late error handling
async function processPaymentWithLateHandling() {
  const paymentPromise = processPayment({ amount: -500 });
  
  console.log('Payment processing started...');
  
  // Error handler added after 2 seconds (late!)
  setTimeout(() => {
    console.log('Adding error handler (late)...');
    paymentPromise.catch(error => {
      console.log('Caught error (late):', error.message);
    });
  }, 2000);
}

// processPaymentWithLateHandling();
```

**Proper Error Handling Patterns:**

```javascript
class BankingTransactionService {
  // ✅ GOOD: Always attach catch handlers
  async processTransactionGood(txnData) {
    try {
      await this.validateTransaction(txnData);
      await this.executeTransaction(txnData);
      await this.sendConfirmation(txnData);
      
      return { success: true };
    } catch (error) {
      console.error('Transaction failed:', error);
      await this.rollbackTransaction(txnData);
      throw error; // Rethrow for caller to handle
    }
  }
  
  // ❌ BAD: Fire-and-forget without error handling
  async processTransactionBad(txnData) {
    // These promises have no error handling!
    this.validateTransaction(txnData);
    this.executeTransaction(txnData);
    this.sendConfirmation(txnData);
  }
  
  // ✅ GOOD: Fire-and-forget with error handling
  async processTransactionFireAndForget(txnData) {
    // Non-critical operations that can fail silently
    this.sendConfirmation(txnData)
      .catch(error => {
        console.warn('Confirmation failed (non-critical):', error);
        // Log but don't fail transaction
      });
    
    this.updateAnalytics(txnData)
      .catch(error => {
        console.warn('Analytics update failed (non-critical):', error);
      });
  }
  
  // ✅ GOOD: Promise.all with proper error handling
  async processMultipleTransactions(transactions) {
    try {
      const results = await Promise.all(
        transactions.map(txn => 
          this.processTransactionGood(txn)
            .catch(error => ({
              transaction: txn,
              error: error.message,
              success: false
            }))
        )
      );
      
      return results;
    } catch (error) {
      console.error('Batch processing failed:', error);
      throw error;
    }
  }
  
  validateTransaction(txnData) {
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        if (!txnData || !txnData.amount) {
          reject(new Error('Invalid transaction data'));
        } else {
          resolve();
        }
      }, 50);
    });
  }
  
  executeTransaction(txnData) {
    return new Promise((resolve) => {
      setTimeout(() => resolve(), 50);
    });
  }
  
  sendConfirmation(txnData) {
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        Math.random() > 0.7
          ? resolve()
          : reject(new Error('Confirmation service unavailable'));
      }, 50);
    });
  }
  
  updateAnalytics(txnData) {
    return new Promise((resolve) => {
      setTimeout(() => resolve(), 50);
    });
  }
  
  rollbackTransaction(txnData) {
    return Promise.resolve();
  }
}
```

**Comprehensive Error Tracking System:**

```javascript
class PromiseErrorMonitor {
  constructor() {
    this.errors = [];
    this.maxErrors = 100;
    this.setupMonitoring();
  }
  
  setupMonitoring() {
    process.on('unhandledRejection', (reason, promise) => {
      const error = {
        id: this.generateId(),
        type: 'UNHANDLED_REJECTION',
        reason: this.serializeError(reason),
        timestamp: new Date(),
        stackTrace: this.getStackTrace(reason),
        context: this.getContext()
      };
      
      this.recordError(error);
      this.handleCriticalError(error);
    });
    
    process.on('rejectionHandled', (promise) => {
      // Track that rejection was eventually handled
      const error = this.errors.find(e => e.handled === false);
      if (error) {
        error.handled = true;
        error.handledAt = new Date();
        error.handledAfter = error.handledAt - error.timestamp;
      }
    });
    
    process.on('uncaughtException', (error) => {
      const errorInfo = {
        id: this.generateId(),
        type: 'UNCAUGHT_EXCEPTION',
        reason: this.serializeError(error),
        timestamp: new Date(),
        stackTrace: this.getStackTrace(error),
        context: this.getContext()
      };
      
      this.recordError(errorInfo);
      this.handleCriticalError(errorInfo);
      
      // Must exit on uncaught exception
      process.exit(1);
    });
  }
  
  generateId() {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
  
  serializeError(error) {
    if (error instanceof Error) {
      return {
        name: error.name,
        message: error.message,
        code: error.code,
        stack: error.stack
      };
    }
    return { message: String(error) };
  }
  
  getStackTrace(error) {
    return error?.stack || new Error().stack;
  }
  
  getContext() {
    return {
      pid: process.pid,
      memory: process.memoryUsage(),
      uptime: process.uptime(),
      nodeVersion: process.version
    };
  }
  
  recordError(error) {
    this.errors.push(error);
    
    // Keep only recent errors
    if (this.errors.length > this.maxErrors) {
      this.errors.shift();
    }
    
    // Log to console
    console.error('\n🚨 ERROR DETECTED:');
    console.error(`ID: ${error.id}`);
    console.error(`Type: ${error.type}`);
    console.error(`Message: ${error.reason.message}`);
    console.error(`Time: ${error.timestamp.toISOString()}\n`);
  }
  
  handleCriticalError(error) {
    // Send to monitoring services
    this.sendToSentry(error);
    this.sendToDatadog(error);
    this.alertOps(error);
  }
  
  sendToSentry(error) {
    // Sentry.captureException(error);
    console.log('📊 Sent to Sentry');
  }
  
  sendToDatadog(error) {
    // datadog.log(error);
    console.log('📊 Sent to Datadog');
  }
  
  alertOps(error) {
    // PagerDuty alert for critical errors
    console.log('📧 Ops team alerted');
  }
  
  getErrorReport() {
    return {
      total: this.errors.length,
      unhandled: this.errors.filter(e => !e.handled).length,
      byType: this.groupByType(),
      recent: this.errors.slice(-10)
    };
  }
  
  groupByType() {
    return this.errors.reduce((acc, error) => {
      acc[error.type] = (acc[error.type] || 0) + 1;
      return acc;
    }, {});
  }
}

// Initialize monitor
const monitor = new PromiseErrorMonitor();

// API endpoint to get error report
function getErrorReportAPI(req, res) {
  res.json(monitor.getErrorReport());
}
```

**Testing Unhandled Rejections:**

```javascript
// Test 1: Simple unhandled rejection
async function test1() {
  Promise.reject(new Error('Test error 1'));
  console.log('Test 1: Promise rejected without handler');
}

// Test 2: Async function without try-catch
async function test2() {
  async function failingOperation() {
    throw new Error('Test error 2');
  }
  
  failingOperation(); // No await, no catch!
  console.log('Test 2: Async function called without await/catch');
}

// Test 3: Promise chain with missing catch
async function test3() {
  Promise.resolve()
    .then(() => {
      throw new Error('Test error 3');
    })
    .then(() => {
      console.log('This will never execute');
    });
  // Missing .catch()!
}

// Run tests
// test1();
// setTimeout(() => test2(), 1000);
// setTimeout(() => test3(), 2000);
```

**Best Practices for ENBD:**

```javascript
// 1. Always use try-catch with async/await
async function bestPractice1() {
  try {
    await riskyOperation();
  } catch (error) {
    console.error('Error handled:', error);
  }
}

// 2. Always add .catch() to promises
function bestPractice2() {
  riskyPromise()
    .then(result => processResult(result))
    .catch(error => handleError(error));
}

// 3. Use Promise.allSettled for batch operations
async function bestPractice3(operations) {
  const results = await Promise.allSettled(operations);
  // All errors are contained in results
}

// 4. Wrap third-party promises
function bestPractice4() {
  return thirdPartyPromise()
    .catch(error => {
      console.error('Third-party error:', error);
      throw new Error('Wrapped error for internal handling');
    });
}

// Helper functions
function riskyOperation() {
  return Promise.resolve();
}

function riskyPromise() {
  return Promise.resolve('success');
}

function processResult(result) {
  return result;
}

function handleError(error) {
  console.error(error);
}

function thirdPartyPromise() {
  return Promise.resolve();
}
```

**Key Takeaways for ENBD:**
- Always handle promise rejections
- Set up global `unhandledRejection` handler
- Log all errors for audit trail
- Alert operations team for critical errors
- Use try-catch with async/await
- Add .catch() to all promise chains
- Test error handling thoroughly
- Consider graceful shutdown on critical errors

---

### Q20. Explain Promise combinators and their use cases. When would you use each in banking operations?

**Answer:**

**Promise Combinators** are methods that handle multiple promises:

| Combinator | Resolves When | Rejects When | Return Value |
|------------|--------------|--------------|--------------|
| `Promise.all()` | All fulfill | Any rejects | Array of values |
| `Promise.allSettled()` | All settle | Never | Array of {status, value/reason} |
| `Promise.race()` | First settles | First rejects | First settled value |
| `Promise.any()` | First fulfills | All reject | First fulfilled value |

**Complete Banking Examples:**

```javascript
class BankingPromiseCombinators {
  // 1. PROMISE.ALL - All must succeed (Atomic Operations)
  async atomicTransfer(fromAccount, toAccount, amount) {
    console.log('\n=== Promise.all(): Atomic Transfer ===');
    
    try {
      // Both operations must succeed or entire transaction fails
      const [debitResult, creditResult] = await Promise.all([
        this.debitAccount(fromAccount, amount),
        this.creditAccount(toAccount, amount)
      ]);
      
      console.log('✓ Transfer successful');
      console.log(`  Debited: ${debitResult.accountId} - ${amount}`);
      console.log(`  Credited: ${creditResult.accountId} + ${amount}`);
      
      return {
        success: true,
        transactionId: this.generateTxnId(),
        debit: debitResult,
        credit: creditResult
      };
    } catch (error) {
      console.error('✗ Transfer failed:', error.message);
      // Both operations rolled back
      throw error;
    }
  }
  
  // 2. PROMISE.ALLSETTLED - Need all results (Batch Processing)
  async processBatchPayments(payments) {
    console.log('\n=== Promise.allSettled(): Batch Processing ===');
    console.log(`Processing ${payments.length} payments...`);
    
    const results = await Promise.allSettled(
      payments.map(payment => this.processPayment(payment))
    );
    
    const summary = {
      total: results.length,
      successful: 0,
      failed: 0,
      details: []
    };
    
    results.forEach((result, index) => {
      const payment = payments[index];
      
      if (result.status === 'fulfilled') {
        summary.successful++;
        summary.details.push({
          paymentId: payment.id,
          status: 'success',
          result: result.value
        });
        console.log(`✓ Payment ${payment.id}: Success`);
      } else {
        summary.failed++;
        summary.details.push({
          paymentId: payment.id,
          status: 'failed',
          error: result.reason.message
        });
        console.error(`✗ Payment ${payment.id}: ${result.reason.message}`);
      }
    });
    
    console.log(`\nSummary: ${summary.successful} success, ${summary.failed} failed`);
    return summary;
  }
  
  // 3. PROMISE.RACE - Fastest response wins (Timeout/Redundancy)
  async fetchExchangeRateWithTimeout(fromCurrency, toCurrency) {
    console.log('\n=== Promise.race(): Exchange Rate with Timeout ===');
    
    const timeout = new Promise((_, reject) => {
      setTimeout(() => reject(new Error('Request timeout')), 5000);
    });
    
    const providers = [
      this.fetchFromProvider('Provider A', 300),
      this.fetchFromProvider('Provider B', 500),
      this.fetchFromProvider('Provider C', 200),
      timeout
    ];
    
    try {
      const result = await Promise.race(providers);
      console.log(`✓ Rate received from: ${result.provider}`);
      console.log(`  Rate: ${result.rate} ${fromCurrency}/${toCurrency}`);
      return result;
    } catch (error) {
      console.error('✗ All providers failed or timeout');
      throw error;
    }
  }
  
  // 4. PROMISE.ANY - First success wins (Fallback Services)
  async sendAlertViaAnyChannel(customerId, message) {
    console.log('\n=== Promise.any(): Multi-Channel Alert ===');
    
    const channels = [
      this.sendViaEmail(customerId, message),
      this.sendViaSMS(customerId, message),
      this.sendViaPush(customerId, message),
      this.sendViaWhatsApp(customerId, message)
    ];
    
    try {
      const result = await Promise.any(channels);
      console.log(`✓ Alert sent via: ${result.channel}`);
      return result;
    } catch (error) {
      // AggregateError - all channels failed
      console.error('✗ All channels failed');
      console.error('Errors:', error.errors.map(e => e.message));
      throw new Error('Failed to send alert via any channel');
    }
  }
  
  // Helper methods
  debitAccount(accountId, amount) {
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        if (Math.random() > 0.9) {
          reject(new Error('Insufficient funds'));
        } else {
          resolve({ accountId, balance: 10000 - amount });
        }
      }, 100);
    });
  }
  
  creditAccount(accountId, amount) {
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve({ accountId, balance: 5000 + amount });
      }, 100);
    });
  }
  
  processPayment(payment) {
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        if (Math.random() > 0.3) {
          resolve({ paymentId: payment.id, status: 'completed' });
        } else {
          reject(new Error('Payment processing failed'));
        }
      }, Math.random() * 500);
    });
  }
  
  fetchFromProvider(provider, delay) {
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        if (Math.random() > 0.4) {
          resolve({
            provider,
            rate: 3.67 + Math.random() * 0.1,
            timestamp: new Date()
          });
        } else {
          reject(new Error(`${provider} unavailable`));
        }
      }, delay);
    });
  }
  
  sendViaEmail(customerId, message) {
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        Math.random() > 0.6
          ? resolve({ channel: 'Email', sent: true })
          : reject(new Error('Email service down'));
      }, 200);
    });
  }
  
  sendViaSMS(customerId, message) {
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        Math.random() > 0.5
          ? resolve({ channel: 'SMS', sent: true })
          : reject(new Error('SMS gateway unavailable'));
      }, 150);
    });
  }
  
  sendViaPush(customerId, message) {
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve({ channel: 'Push Notification', sent: true });
      }, 100);
    });
  }
  
  sendViaWhatsApp(customerId, message) {
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        Math.random() > 0.7
          ? resolve({ channel: 'WhatsApp', sent: true })
          : reject(new Error('WhatsApp not configured'));
      }, 250);
    });
  }
  
  generateTxnId() {
    return `TXN-${Date.now()}`;
  }
}

// Usage examples
const banking = new BankingPromiseCombinators();

async function runExamples() {
  // 1. Atomic transfer
  try {
    await banking.atomicTransfer('ACC-001', 'ACC-002', 1000);
  } catch (error) {
    console.error('Transfer failed:', error.message);
  }
  
  // 2. Batch processing
  const payments = [
    { id: 'PAY-001', amount: 100 },
    { id: 'PAY-002', amount: 200 },
    { id: 'PAY-003', amount: 300 },
    { id: 'PAY-004', amount: 400 },
    { id: 'PAY-005', amount: 500 }
  ];
  await banking.processBatchPayments(payments);
  
  // 3. Exchange rate with timeout
  try {
    await banking.fetchExchangeRateWithTimeout('USD', 'AED');
  } catch (error) {
    console.error('Exchange rate failed:', error.message);
  }
  
  // 4. Multi-channel alert
  try {
    await banking.sendAlertViaAnyChannel('CUST-001', 'Transaction completed');
  } catch (error) {
    console.error('Alert failed:', error.message);
  }
}

// runExamples();
```

**Advanced Use Cases:**

```javascript
class AdvancedBankingOperations {
  // Combining multiple combinators
  async complexLoanApproval(customerId) {
    console.log('\n=== Complex Loan Approval ===');
    
    // Step 1: Race to get fastest credit score
    const creditScore = await Promise.race([
      this.checkCreditBureau1(customerId),
      this.checkCreditBureau2(customerId),
      this.timeout(3000, 'Credit check timeout')
    ]);
    
    console.log(`Credit score: ${creditScore.score}`);
    
    // Step 2: All required checks must pass
    const [employment, income, identity] = await Promise.all([
      this.verifyEmployment(customerId),
      this.verifyIncome(customerId),
      this.verifyIdentity(customerId)
    ]);
    
    console.log('All verifications passed');
    
    // Step 3: Try to send approval via any channel
    const notification = await Promise.any([
      this.sendEmail(customerId, 'Loan approved'),
      this.sendSMS(customerId, 'Loan approved'),
      this.sendPush(customerId, 'Loan approved')
    ]);
    
    console.log(`Notification sent via: ${notification.channel}`);
    
    // Step 4: Non-critical updates (all settled)
    await Promise.allSettled([
      this.updateCRM(customerId),
      this.updateAnalytics(customerId),
      this.updateReporting(customerId)
    ]);
    
    return { approved: true, creditScore, employment, income, identity };
  }
  
  // Retry with Promise.race
  async fetchWithRetry(operation, maxRetries = 3) {
    for (let i = 0; i < maxRetries; i++) {
      try {
        return await Promise.race([
          operation(),
          this.timeout(2000, 'Operation timeout')
        ]);
      } catch (error) {
        console.log(`Attempt ${i + 1} failed: ${error.message}`);
        if (i === maxRetries - 1) throw error;
        await this.sleep(1000 * (i + 1)); // Exponential backoff
      }
    }
  }
  
  // Parallel with limit (avoid overwhelming services)
  async processWithConcurrencyLimit(items, processor, limit = 5) {
    const results = [];
    
    for (let i = 0; i < items.length; i += limit) {
      const batch = items.slice(i, i + limit);
      const batchResults = await Promise.all(
        batch.map(item => processor(item))
      );
      results.push(...batchResults);
      console.log(`Processed batch ${Math.floor(i / limit) + 1}`);
    }
    
    return results;
  }
  
  // Helper methods
  checkCreditBureau1(customerId) {
    return new Promise((resolve) => {
      setTimeout(() => resolve({ bureau: 'Bureau1', score: 720 }), 400);
    });
  }
  
  checkCreditBureau2(customerId) {
    return new Promise((resolve) => {
      setTimeout(() => resolve({ bureau: 'Bureau2', score: 710 }), 300);
    });
  }
  
  verifyEmployment(customerId) {
    return Promise.resolve({ verified: true });
  }
  
  verifyIncome(customerId) {
    return Promise.resolve({ verified: true, amount: 75000 });
  }
  
  verifyIdentity(customerId) {
    return Promise.resolve({ verified: true });
  }
  
  sendEmail(customerId, message) {
    return Promise.resolve({ channel: 'Email' });
  }
  
  sendSMS(customerId, message) {
    return Promise.resolve({ channel: 'SMS' });
  }
  
  sendPush(customerId, message) {
    return Promise.resolve({ channel: 'Push' });
  }
  
  updateCRM(customerId) {
    return Promise.resolve();
  }
  
  updateAnalytics(customerId) {
    return Promise.resolve();
  }
  
  updateReporting(customerId) {
    return Promise.resolve();
  }
  
  timeout(ms, message) {
    return new Promise((_, reject) => {
      setTimeout(() => reject(new Error(message)), ms);
    });
  }
  
  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

**Decision Matrix:**

```javascript
// Choose the right combinator based on requirements

// ALL operations MUST succeed → Promise.all()
const criticalData = await Promise.all([
  fetchAccountBalance(),
  fetchAccountDetails(),
  fetchAccountHistory()
]);

// Need results from ALL operations → Promise.allSettled()
const batchResults = await Promise.allSettled([
  processPayment1(),
  processPayment2(),
  processPayment3()
]);

// Need FASTEST response → Promise.race()
const exchangeRate = await Promise.race([
  fetchFromProvider1(),
  fetchFromProvider2(),
  timeoutAfter(5000)
]);

// Need AT LEAST ONE success → Promise.any()
const notification = await Promise.any([
  sendViaEmail(),
  sendViaSMS(),
  sendViaPush()
]);

function fetchAccountBalance() {
  return Promise.resolve(10000);
}

function fetchAccountDetails() {
  return Promise.resolve({ type: 'savings' });
}

function fetchAccountHistory() {
  return Promise.resolve([]);
}

function processPayment1() {
  return Promise.resolve('success');
}

function processPayment2() {
  return Promise.resolve('success');
}

function processPayment3() {
  return Promise.resolve('success');
}

function fetchFromProvider1() {
  return Promise.resolve({ rate: 3.67 });
}

function fetchFromProvider2() {
  return Promise.resolve({ rate: 3.68 });
}

function timeoutAfter(ms) {
  return new Promise((_, reject) => setTimeout(() => reject(new Error('Timeout')), ms));
}

function sendViaEmail() {
  return Promise.resolve({ channel: 'email' });
}

function sendViaSMS() {
  return Promise.resolve({ channel: 'sms' });
}

function sendViaPush() {
  return Promise.resolve({ channel: 'push' });
}
```

**Key Takeaways for ENBD:**
- `Promise.all()` for atomic operations (transfers)
- `Promise.allSettled()` for batch processing
- `Promise.race()` for timeouts and redundancy
- `Promise.any()` for fallback mechanisms
- Combine multiple combinators for complex workflows
- Always handle errors appropriately
- Consider performance implications of parallel operations

---

**🎯 Section 2 Complete!**
✅ **Completed Q11-Q20** (10 questions on Promises, Async/Await, and Error Handling)

**Next Section:** Section 3 (Q21-Q30) - Streams and Buffers

---



---

## Section 3: Streams and Buffers

### Questions 21-30

### Q21. What are Streams in Node.js? Explain the four types of streams with banking examples.

**Answer:**

**Streams** are objects that let you read or write data in chunks (piece by piece) rather than loading everything into memory at once. They are especially useful for handling large files or continuous data flows.

**Why Streams Matter for Banking:**
- Process large transaction files without memory overflow
- Real-time data processing (live feeds)
- Efficient file uploads/downloads
- Memory-efficient report generation
- Handle high-volume data pipelines

**Four Types of Streams:**

1. **Readable Streams** - Source of data (reading)
2. **Writable Streams** - Destination for data (writing)
3. **Duplex Streams** - Both readable and writable
4. **Transform Streams** - Modify data as it passes through

**Stream Events:**
- `data` - When chunk is available
- `end` - When no more data
- `error` - When error occurs
- `finish` - When all data written (writable)
- `pipe` - When piped to writable stream
- `unpipe` - When unpiped

**1. Readable Stream - Reading Transaction Files:**

```typescript
import { Readable } from 'stream';
import * as fs from 'fs';
import * as readline from 'readline';

// Example 1: Reading large transaction file
class TransactionFileReader {
  async processLargeFile(filePath: string): Promise<void> {
    const readStream = fs.createReadStream(filePath, {
      encoding: 'utf8',
      highWaterMark: 64 * 1024 // 64KB chunks
    });
    
    let processedCount = 0;
    let totalAmount = 0;
    
    // Listen to data chunks
    readStream.on('data', (chunk: string) => {
      const lines = chunk.split('\n');
      
      for (const line of lines) {
        if (line.trim()) {
          const transaction = JSON.parse(line);
          totalAmount += transaction.amount;
          processedCount++;
          
          // Process without loading entire file
          if (transaction.amount > 10000) {
            console.log(`Large transaction: ${transaction.id} - $${transaction.amount}`);
          }
        }
      }
    });
    
    // Handle completion
    readStream.on('end', () => {
      console.log(`\nProcessed ${processedCount} transactions`);
      console.log(`Total amount: $${totalAmount.toFixed(2)}`);
    });
    
    // Handle errors
    readStream.on('error', (error) => {
      console.error('Error reading file:', error.message);
    });
  }
  
  // Example 2: Custom readable stream
  createTransactionStream(transactions: any[]): Readable {
    let index = 0;
    
    return new Readable({
      objectMode: true,
      read() {
        if (index < transactions.length) {
          // Push one transaction at a time
          this.push(transactions[index]);
          index++;
        } else {
          // Signal end of stream
          this.push(null);
        }
      }
    });
  }
}

// Usage
const reader = new TransactionFileReader();

// Create sample transactions
const transactions = Array.from({ length: 1000 }, (_, i) => ({
  id: `TXN-${String(i + 1).padStart(4, '0')}`,
  accountId: `ACC-${Math.floor(Math.random() * 100)}`,
  amount: Math.random() * 10000,
  type: Math.random() > 0.5 ? 'deposit' : 'withdrawal',
  timestamp: new Date()
}));

const txnStream = reader.createTransactionStream(transactions);

txnStream.on('data', (transaction) => {
  console.log(`Processing: ${transaction.id} - $${transaction.amount.toFixed(2)}`);
});

txnStream.on('end', () => {
  console.log('All transactions processed');
});
```

**2. Writable Stream - Writing Audit Logs:**

```typescript
import { Writable } from 'stream';

// Example 1: Writing to audit log file
class AuditLogger {
  private writeStream: fs.WriteStream;
  
  constructor(logFilePath: string) {
    this.writeStream = fs.createWriteStream(logFilePath, {
      flags: 'a', // Append mode
      encoding: 'utf8'
    });
    
    this.writeStream.on('error', (error) => {
      console.error('Audit log error:', error);
    });
    
    this.writeStream.on('finish', () => {
      console.log('Audit log closed');
    });
  }
  
  logTransaction(transaction: any): void {
    const logEntry = JSON.stringify({
      ...transaction,
      timestamp: new Date().toISOString()
    }) + '\n';
    
    // Write to stream
    const canContinue = this.writeStream.write(logEntry);
    
    if (!canContinue) {
      // Buffer is full, wait for drain event
      this.writeStream.once('drain', () => {
        console.log('Buffer drained, ready to write more');
      });
    }
  }
  
  close(): Promise<void> {
    return new Promise((resolve) => {
      this.writeStream.end(() => {
        resolve();
      });
    });
  }
}

// Example 2: Custom writable stream
class TransactionValidator extends Writable {
  private validCount = 0;
  private invalidCount = 0;
  private validTransactions: any[] = [];
  
  constructor() {
    super({ objectMode: true });
  }
  
  _write(transaction: any, encoding: string, callback: (error?: Error | null) => void): void {
    try {
      // Validate transaction
      if (this.isValid(transaction)) {
        this.validCount++;
        this.validTransactions.push(transaction);
        console.log(`✓ Valid: ${transaction.id}`);
      } else {
        this.invalidCount++;
        console.log(`✗ Invalid: ${transaction.id}`);
      }
      
      callback(); // Signal completion
    } catch (error) {
      callback(error as Error);
    }
  }
  
  _final(callback: (error?: Error | null) => void): void {
    // Called when stream is ending
    console.log(`\nValidation Summary:`);
    console.log(`Valid: ${this.validCount}`);
    console.log(`Invalid: ${this.invalidCount}`);
    callback();
  }
  
  private isValid(transaction: any): boolean {
    return transaction.amount > 0 && 
           transaction.accountId && 
           transaction.type;
  }
  
  getValidTransactions(): any[] {
    return this.validTransactions;
  }
}

// Usage
const logger = new AuditLogger('./audit.log');
logger.logTransaction({
  id: 'TXN-001',
  accountId: 'ACC-001',
  amount: 1000,
  type: 'deposit'
});
```

**3. Duplex Stream - TCP Socket Communication:**

```typescript
import { Duplex } from 'stream';
import * as net from 'net';

// Real-time payment processing over TCP
class PaymentProcessor extends Duplex {
  private buffer: any[] = [];
  
  constructor() {
    super({ objectMode: true });
  }
  
  // Implement read
  _read(size: number): void {
    if (this.buffer.length > 0) {
      const chunk = this.buffer.shift();
      this.push(chunk);
    }
  }
  
  // Implement write
  _write(chunk: any, encoding: string, callback: (error?: Error | null) => void): void {
    try {
      // Process payment request
      console.log(`Processing payment: ${chunk.id}`);
      
      const result = {
        id: chunk.id,
        status: 'completed',
        processedAt: new Date(),
        amount: chunk.amount
      };
      
      // Add to buffer for reading
      this.buffer.push(result);
      
      // Emit readable event
      this.emit('readable');
      
      callback();
    } catch (error) {
      callback(error as Error);
    }
  }
}

// TCP Server for payment processing
class PaymentGateway {
  private server: net.Server;
  
  constructor(port: number) {
    this.server = net.createServer((socket) => {
      console.log('Client connected');
      
      // Socket is a duplex stream
      socket.setEncoding('utf8');
      
      socket.on('data', (data) => {
        try {
          const payment = JSON.parse(data.toString());
          console.log(`Received payment: ${payment.id}`);
          
          // Process and respond
          const response = {
            id: payment.id,
            status: 'success',
            transactionId: `TXN-${Date.now()}`,
            timestamp: new Date()
          };
          
          socket.write(JSON.stringify(response) + '\n');
        } catch (error) {
          socket.write(JSON.stringify({ error: 'Invalid payment data' }) + '\n');
        }
      });
      
      socket.on('end', () => {
        console.log('Client disconnected');
      });
      
      socket.on('error', (error) => {
        console.error('Socket error:', error.message);
      });
    });
    
    this.server.listen(port, () => {
      console.log(`Payment gateway listening on port ${port}`);
    });
  }
  
  close(): void {
    this.server.close();
  }
}

// Usage
const processor = new PaymentProcessor();

// Write payment requests
processor.write({ id: 'PAY-001', amount: 1000 });
processor.write({ id: 'PAY-002', amount: 2000 });

// Read processed results
processor.on('readable', () => {
  let result;
  while (null !== (result = processor.read())) {
    console.log(`Payment processed:`, result);
  }
});
```

**4. Transform Stream - Data Transformation Pipeline:**

```typescript
import { Transform } from 'stream';
import { createHash } from 'crypto';

// Example 1: Encrypt transactions
class TransactionEncryptor extends Transform {
  constructor() {
    super({ objectMode: true });
  }
  
  _transform(transaction: any, encoding: string, callback: (error?: Error | null, data?: any) => void): void {
    try {
      // Encrypt sensitive data
      const encrypted = {
        ...transaction,
        accountId: this.encrypt(transaction.accountId),
        customerName: this.encrypt(transaction.customerName),
        amount: transaction.amount, // Keep amount readable for processing
        encrypted: true
      };
      
      callback(null, encrypted);
    } catch (error) {
      callback(error as Error);
    }
  }
  
  private encrypt(data: string): string {
    // Simple encryption for demo (use proper encryption in production)
    const hash = createHash('sha256');
    hash.update(data);
    return hash.digest('hex');
  }
}

// Example 2: Format transaction data
class TransactionFormatter extends Transform {
  constructor() {
    super({ objectMode: true });
  }
  
  _transform(transaction: any, encoding: string, callback: (error?: Error | null, data?: any) => void): void {
    try {
      const formatted = {
        transactionId: transaction.id,
        date: new Date(transaction.timestamp).toLocaleDateString(),
        time: new Date(transaction.timestamp).toLocaleTimeString(),
        type: transaction.type.toUpperCase(),
        amount: `$${transaction.amount.toFixed(2)}`,
        status: transaction.status || 'PENDING'
      };
      
      callback(null, formatted);
    } catch (error) {
      callback(error as Error);
    }
  }
}

// Example 3: Filter large transactions
class LargeTransactionFilter extends Transform {
  private threshold: number;
  
  constructor(threshold: number = 10000) {
    super({ objectMode: true });
    this.threshold = threshold;
  }
  
  _transform(transaction: any, encoding: string, callback: (error?: Error | null, data?: any) => void): void {
    // Only pass through large transactions
    if (transaction.amount >= this.threshold) {
      callback(null, transaction);
    } else {
      callback(); // Skip this transaction
    }
  }
}

// Example 4: Add fraud score
class FraudDetectionTransform extends Transform {
  constructor() {
    super({ objectMode: true });
  }
  
  _transform(transaction: any, encoding: string, callback: (error?: Error | null, data?: any) => void): void {
    try {
      const fraudScore = this.calculateFraudScore(transaction);
      
      const enriched = {
        ...transaction,
        fraudScore,
        riskLevel: this.getRiskLevel(fraudScore),
        requiresReview: fraudScore > 70
      };
      
      callback(null, enriched);
    } catch (error) {
      callback(error as Error);
    }
  }
  
  private calculateFraudScore(transaction: any): number {
    let score = 0;
    
    // Large amount increases risk
    if (transaction.amount > 50000) score += 30;
    else if (transaction.amount > 10000) score += 15;
    
    // International transactions are riskier
    if (transaction.isInternational) score += 25;
    
    // Off-hours transactions
    const hour = new Date(transaction.timestamp).getHours();
    if (hour < 6 || hour > 22) score += 20;
    
    return Math.min(score, 100);
  }
  
  private getRiskLevel(score: number): string {
    if (score >= 70) return 'HIGH';
    if (score >= 40) return 'MEDIUM';
    return 'LOW';
  }
}

// Usage: Build processing pipeline
const transactionSource = reader.createTransactionStream(transactions);
const encryptor = new TransactionEncryptor();
const formatter = new TransactionFormatter();
const fraudDetector = new FraudDetectionTransform();
const largeFilter = new LargeTransactionFilter(5000);
const validator = new TransactionValidator();

// Pipe streams together
transactionSource
  .pipe(fraudDetector)      // Add fraud scores
  .pipe(largeFilter)        // Filter large transactions
  .pipe(encryptor)          // Encrypt sensitive data
  .pipe(formatter)          // Format for display
  .pipe(validator);         // Validate and store

validator.on('finish', () => {
  console.log('\nPipeline complete!');
  const valid = validator.getValidTransactions();
  console.log(`Valid transactions: ${valid.length}`);
});
```

**Stream Piping:**

```typescript
// Piping connects streams together
class StreamPipelineExample {
  processTransactionFile(inputPath: string, outputPath: string): void {
    const readStream = fs.createReadStream(inputPath);
    const writeStream = fs.createWriteStream(outputPath);
    const transformer = new TransactionFormatter();
    
    // Pipe: read → transform → write
    readStream
      .pipe(transformer)
      .pipe(writeStream)
      .on('finish', () => {
        console.log('File processing complete');
      })
      .on('error', (error) => {
        console.error('Pipeline error:', error);
      });
    
    // Handle backpressure automatically
  }
  
  // Multiple pipes
  complexPipeline(inputPath: string, outputPath: string): void {
    fs.createReadStream(inputPath)
      .pipe(new FraudDetectionTransform())
      .pipe(new LargeTransactionFilter(10000))
      .pipe(new TransactionEncryptor())
      .pipe(new TransactionFormatter())
      .pipe(fs.createWriteStream(outputPath));
  }
}
```

**Key Takeaways for ENBD:**
- Use readable streams for processing large files
- Use writable streams for logging and persistence
- Use duplex streams for network communication
- Use transform streams for data processing pipelines
- Piping automatically handles backpressure
- Streams are memory-efficient (process in chunks)
- Always handle error events
- Use `objectMode: true` for non-buffer data

---

### Q22. What is backpressure in streams? How do you handle it in banking applications?

**Answer:**

**Backpressure** occurs when data is being written to a stream faster than it can be consumed. The writable stream's internal buffer fills up, and the source needs to slow down.

**Why Critical for Banking:**
- Prevents memory overflow in high-volume processing
- Ensures data integrity (no data loss)
- Maintains system stability under load
- Prevents application crashes
- Required for production-grade systems

**How Backpressure Works:**

```
Producer (Fast) → Buffer → Consumer (Slow)
                   ↓
              Buffer Full!
                   ↓
            Backpressure Signal
                   ↓
          Producer Pauses/Slows
```

**Understanding write() Return Value:**

```typescript
import * as fs from 'fs';

// Write returns boolean indicating if you should continue
function demonstrateBackpressure(): void {
  const writeStream = fs.createWriteStream('./large-file.txt');
  
  let canWrite = true;
  let written = 0;
  
  function writeData() {
    while (written < 1000000 && canWrite) {
      written++;
      const data = `Transaction ${written}\n`;
      
      // write() returns false when buffer is full
      canWrite = writeStream.write(data);
      
      if (!canWrite) {
        console.log(`Buffer full at transaction ${written}`);
        console.log('Pausing writes until buffer drains...');
        
        // Wait for drain event
        writeStream.once('drain', () => {
          console.log('Buffer drained, resuming writes...');
          canWrite = true;
          writeData(); // Resume writing
        });
      }
    }
    
    if (written >= 1000000) {
      writeStream.end(() => {
        console.log('Writing complete');
      });
    }
  }
  
  writeData();
}
```

**Banking Example 1: Transaction Processing with Backpressure:**

```typescript
import { Readable, Writable, pipeline } from 'stream';
import { promisify } from 'util';

const pipelineAsync = promisify(pipeline);

// Readable stream that respects backpressure
class TransactionStream extends Readable {
  private currentIndex = 0;
  private totalTransactions: number;
  private paused = false;
  
  constructor(totalTransactions: number) {
    super({ objectMode: true, highWaterMark: 16 }); // Small buffer
    this.totalTransactions = totalTransactions;
  }
  
  _read(): void {
    if (this.paused) {
      console.log('Stream is paused, waiting...');
      return;
    }
    
    if (this.currentIndex >= this.totalTransactions) {
      this.push(null); // End of stream
      return;
    }
    
    // Generate transaction
    const transaction = {
      id: `TXN-${String(this.currentIndex + 1).padStart(6, '0')}`,
      accountId: `ACC-${Math.floor(Math.random() * 1000)}`,
      amount: Math.random() * 10000,
      type: Math.random() > 0.5 ? 'deposit' : 'withdrawal',
      timestamp: new Date()
    };
    
    this.currentIndex++;
    
    // Push returns false if buffer is full (backpressure)
    const canContinue = this.push(transaction);
    
    if (!canContinue) {
      console.log(`Backpressure detected at transaction ${this.currentIndex}`);
      this.paused = true;
      
      // Stream will automatically call _read() again when ready
      setTimeout(() => {
        this.paused = false;
      }, 100);
    }
    
    // Show progress
    if (this.currentIndex % 1000 === 0) {
      console.log(`Generated ${this.currentIndex} transactions`);
    }
  }
}

// Writable stream with slow processing (simulates database writes)
class DatabaseWriter extends Writable {
  private processedCount = 0;
  private totalAmount = 0;
  
  constructor() {
    super({ objectMode: true, highWaterMark: 8 }); // Small buffer for demo
  }
  
  async _write(
    transaction: any,
    encoding: string,
    callback: (error?: Error | null) => void
  ): Promise<void> {
    try {
      // Simulate slow database write
      await this.writeToDatabase(transaction);
      
      this.processedCount++;
      this.totalAmount += transaction.amount;
      
      if (this.processedCount % 100 === 0) {
        console.log(`Processed ${this.processedCount} transactions`);
      }
      
      callback();
    } catch (error) {
      callback(error as Error);
    }
  }
  
  _final(callback: (error?: Error | null) => void): void {
    console.log(`\nFinal Statistics:`);
    console.log(`Total processed: ${this.processedCount}`);
    console.log(`Total amount: $${this.totalAmount.toFixed(2)}`);
    callback();
  }
  
  private async writeToDatabase(transaction: any): Promise<void> {
    // Simulate slow database write (10ms)
    return new Promise(resolve => setTimeout(resolve, 10));
  }
}

// Usage with automatic backpressure handling
async function processTransactionsWithBackpressure(): Promise<void> {
  console.log('Starting transaction processing with backpressure handling...\n');
  
  const source = new TransactionStream(10000);
  const destination = new DatabaseWriter();
  
  try {
    await pipelineAsync(source, destination);
    console.log('\nAll transactions processed successfully');
  } catch (error) {
    console.error('Pipeline error:', error);
  }
}

// processTransactionsWithBackpressure();
```

**Banking Example 2: Manual Backpressure Handling:**

```typescript
class TransactionProcessor {
  private writeStream: fs.WriteStream;
  private processing = false;
  private queue: any[] = [];
  
  constructor(outputPath: string) {
    this.writeStream = fs.createWriteStream(outputPath);
    
    // Listen for drain event
    this.writeStream.on('drain', () => {
      console.log('Buffer drained, processing queued transactions');
      this.processQueue();
    });
  }
  
  async processTransaction(transaction: any): Promise<void> {
    const data = JSON.stringify(transaction) + '\n';
    
    // Try to write
    const canWrite = this.writeStream.write(data);
    
    if (!canWrite) {
      // Buffer is full, queue the transaction
      console.log(`Backpressure! Queueing transaction ${transaction.id}`);
      return new Promise((resolve) => {
        this.queue.push({ transaction, resolve });
      });
    }
  }
  
  private processQueue(): void {
    if (this.processing || this.queue.length === 0) {
      return;
    }
    
    this.processing = true;
    
    while (this.queue.length > 0) {
      const item = this.queue[0];
      const data = JSON.stringify(item.transaction) + '\n';
      
      const canWrite = this.writeStream.write(data);
      
      if (canWrite) {
        // Written successfully, remove from queue
        this.queue.shift();
        item.resolve();
      } else {
        // Buffer full again, wait for next drain
        break;
      }
    }
    
    this.processing = false;
  }
  
  async close(): Promise<void> {
    return new Promise((resolve) => {
      this.writeStream.end(() => {
        resolve();
      });
    });
  }
}
```

**Banking Example 3: Rate-Limited Payment Processing:**

```typescript
import { Transform } from 'stream';

class RateLimitedPaymentProcessor extends Transform {
  private processedCount = 0;
  private maxPerSecond = 100;
  private windowStart = Date.now();
  private pendingTransactions: any[] = [];
  
  constructor(maxPerSecond: number = 100) {
    super({ objectMode: true });
    this.maxPerSecond = maxPerSecond;
  }
  
  _transform(
    payment: any,
    encoding: string,
    callback: (error?: Error | null, data?: any) => void
  ): void {
    const now = Date.now();
    const elapsed = now - this.windowStart;
    
    // Reset counter every second
    if (elapsed >= 1000) {
      console.log(`Processed ${this.processedCount} payments in last second`);
      this.processedCount = 0;
      this.windowStart = now;
    }
    
    // Check rate limit
    if (this.processedCount >= this.maxPerSecond) {
      // Rate limit exceeded, add delay
      const delay = 1000 - elapsed;
      console.log(`Rate limit reached, delaying by ${delay}ms`);
      
      setTimeout(() => {
        this.processedCount++;
        callback(null, payment);
      }, delay);
    } else {
      // Under limit, process immediately
      this.processedCount++;
      callback(null, payment);
    }
  }
}

// Usage
async function processPaymentsWithRateLimit(): Promise<void> {
  const payments = new TransactionStream(1000);
  const rateLimiter = new RateLimitedPaymentProcessor(50); // 50/second
  const writer = new DatabaseWriter();
  
  await pipelineAsync(payments, rateLimiter, writer);
}
```

**Banking Example 4: Buffering Strategy:**

```typescript
class BufferedTransactionWriter {
  private writeStream: fs.WriteStream;
  private buffer: string[] = [];
  private bufferSize = 100;
  private flushInterval = 1000; // 1 second
  private flushTimer: NodeJS.Timeout | null = null;
  
  constructor(outputPath: string) {
    this.writeStream = fs.createWriteStream(outputPath);
    this.startFlushTimer();
  }
  
  addTransaction(transaction: any): void {
    this.buffer.push(JSON.stringify(transaction) + '\n');
    
    // Flush if buffer is full
    if (this.buffer.length >= this.bufferSize) {
      this.flush();
    }
  }
  
  private flush(): void {
    if (this.buffer.length === 0) {
      return;
    }
    
    const data = this.buffer.join('');
    this.buffer = [];
    
    // Write entire buffer at once
    const canWrite = this.writeStream.write(data);
    
    if (!canWrite) {
      console.log('Backpressure detected, waiting for drain...');
      
      this.writeStream.once('drain', () => {
        console.log('Drain complete, ready for more data');
      });
    }
  }
  
  private startFlushTimer(): void {
    this.flushTimer = setInterval(() => {
      this.flush();
    }, this.flushInterval);
  }
  
  async close(): Promise<void> {
    if (this.flushTimer) {
      clearInterval(this.flushTimer);
    }
    
    // Flush remaining buffer
    this.flush();
    
    return new Promise((resolve) => {
      this.writeStream.end(() => {
        resolve();
      });
    });
  }
}

// Usage
const writer = new BufferedTransactionWriter('./transactions.log');

// Add transactions rapidly
for (let i = 0; i < 10000; i++) {
  writer.addTransaction({
    id: `TXN-${i}`,
    amount: Math.random() * 1000,
    timestamp: new Date()
  });
}

// Close when done
await writer.close();
```

**Monitoring Backpressure:**

```typescript
class StreamMonitor {
  monitorStream(stream: any, name: string): void {
    // Monitor buffer levels
    const checkBuffer = () => {
      const bufferLength = stream.writableLength || stream.readableLength || 0;
      const highWaterMark = stream.writableHighWaterMark || stream.readableHighWaterMark || 0;
      const utilization = (bufferLength / highWaterMark) * 100;
      
      console.log(`${name} buffer utilization: ${utilization.toFixed(2)}%`);
      
      if (utilization > 80) {
        console.warn(`⚠️ ${name} buffer is ${utilization.toFixed(2)}% full!`);
      }
    };
    
    setInterval(checkBuffer, 1000);
    
    // Monitor events
    stream.on('drain', () => {
      console.log(`✓ ${name}: Buffer drained`);
    });
    
    stream.on('pause', () => {
      console.log(`⏸️ ${name}: Stream paused`);
    });
    
    stream.on('resume', () => {
      console.log(`▶️ ${name}: Stream resumed`);
    });
  }
}
```

**Best Practices for Backpressure:**

```typescript
// ✅ GOOD: Use pipeline (handles backpressure automatically)
import { pipeline } from 'stream';

async function goodPipeline(): Promise<void> {
  await pipelineAsync(
    createReadStream('input.txt'),
    new TransformStream(),
    createWriteStream('output.txt')
  );
}

// ✅ GOOD: Respect write() return value
function goodWrite(stream: fs.WriteStream, data: string): Promise<void> {
  return new Promise((resolve, reject) => {
    if (!stream.write(data)) {
      stream.once('drain', resolve);
      stream.once('error', reject);
    } else {
      process.nextTick(resolve);
    }
  });
}

// ❌ BAD: Ignore backpressure (memory leak!)
function badWrite(stream: fs.WriteStream, data: string): void {
  stream.write(data); // Ignoring return value!
}

// ❌ BAD: Manual piping without backpressure handling
function badPipeline(): void {
  const read = createReadStream('input.txt');
  const write = createWriteStream('output.txt');
  
  read.on('data', (chunk) => {
    write.write(chunk); // Ignoring return value!
  });
}
```

**Key Takeaways for ENBD:**
- Always use `pipeline()` or handle `write()` return value
- Listen to `drain` event when backpressure occurs
- Use appropriate `highWaterMark` for your use case
- Monitor buffer utilization in production
- Implement buffering strategies for high-volume data
- Rate limiting prevents overwhelming downstream systems
- Test with realistic data volumes
- Backpressure prevents memory leaks and crashes

---

### Q23. Explain Buffers in Node.js. When would you use them in banking applications?

**Answer:**

**Buffer** is a class in Node.js for handling binary data directly. It represents a fixed-size chunk of memory allocated outside the V8 heap.

**Why Buffers Matter for Banking:**
- Reading/writing binary files (PDFs, images, encrypted data)
- Network communication (TCP/UDP protocols)
- Cryptographic operations (encryption, hashing)
- File uploads/downloads
- Performance (faster than strings for binary data)
- Protocol implementation (custom banking protocols)

**Creating Buffers:**

```typescript
// 1. Allocate buffer (uninitialized)
const buf1 = Buffer.alloc(10); // 10 bytes, filled with 0
console.log(buf1); // <Buffer 00 00 00 00 00 00 00 00 00 00>

// 2. Allocate buffer (uninitialized - faster but unsafe)
const buf2 = Buffer.allocUnsafe(10); // Contains whatever was in memory
console.log(buf2); // <Buffer ?? ?? ?? ?? ?? ?? ?? ?? ?? ??>

// 3. From string
const buf3 = Buffer.from('Hello', 'utf8');
console.log(buf3); // <Buffer 48 65 6c 6c 6f>
console.log(buf3.toString()); // "Hello"

// 4. From array
const buf4 = Buffer.from([72, 101, 108, 108, 111]);
console.log(buf4.toString()); // "Hello"

// 5. From another buffer
const buf5 = Buffer.from(buf3);
console.log(buf5.toString()); // "Hello"
```

**Buffer Operations:**

```typescript
// Writing to buffer
const buffer = Buffer.alloc(256);

// Write string
buffer.write('Emirates NBD', 0, 'utf8');
console.log(buffer.toString('utf8', 0, 12)); // "Emirates NBD"

// Write numbers
buffer.writeInt32BE(1000, 20);  // Big-endian
buffer.writeInt32LE(2000, 24);  // Little-endian
buffer.writeFloatBE(99.99, 28);
buffer.writeDoubleBE(123.456, 32);

// Reading from buffer
const amount = buffer.readInt32BE(20);
console.log(`Amount: ${amount}`); // 1000

const price = buffer.readFloatBE(28);
console.log(`Price: ${price}`); // 99.99

// Buffer methods
console.log(`Length: ${buffer.length}`);
console.log(`Slice: ${buffer.slice(0, 12).toString()}`);

// Compare buffers
const buf6 = Buffer.from('ABC');
const buf7 = Buffer.from('ABC');
const buf8 = Buffer.from('ABD');

console.log(buf6.compare(buf7)); // 0 (equal)
console.log(buf6.compare(buf8)); // -1 (buf6 < buf8)

// Concatenate buffers
const combined = Buffer.concat([buf6, buf7]);
console.log(combined.toString()); // "ABCABC"

// Copy buffer
const source = Buffer.from('Source');
const target = Buffer.alloc(6);
source.copy(target);
console.log(target.toString()); // "Source"

// Fill buffer
const buf9 = Buffer.alloc(10);
buf9.fill('A');
console.log(buf9.toString()); // "AAAAAAAAAA"
```

**Banking Example 1: Encryption/Decryption with Buffers:**

```typescript
import { createCipheriv, createDecipheriv, randomBytes, createHash } from 'crypto';

class BankingEncryption {
  private algorithm = 'aes-256-cbc';
  
  // Encrypt sensitive data
  encrypt(data: string, key: Buffer): { encrypted: Buffer; iv: Buffer } {
    // Generate random IV
    const iv = randomBytes(16);
    
    // Ensure key is 32 bytes for AES-256
    const keyHash = createHash('sha256').update(key).digest();
    
    // Create cipher
    const cipher = createCipheriv(this.algorithm, keyHash, iv);
    
    // Encrypt
    const encrypted = Buffer.concat([
      cipher.update(data, 'utf8'),
      cipher.final()
    ]);
    
    return { encrypted, iv };
  }
  
  // Decrypt sensitive data
  decrypt(encrypted: Buffer, key: Buffer, iv: Buffer): string {
    // Hash key
    const keyHash = createHash('sha256').update(key).digest();
    
    // Create decipher
    const decipher = createDecipheriv(this.algorithm, keyHash, iv);
    
    // Decrypt
    const decrypted = Buffer.concat([
      decipher.update(encrypted),
      decipher.final()
    ]);
    
    return decrypted.toString('utf8');
  }
  
  // Hash account number
  hashAccountNumber(accountNumber: string): string {
    const hash = createHash('sha256');
    hash.update(accountNumber);
    return hash.digest('hex');
  }
  
  // Generate secure token
  generateToken(length: number = 32): string {
    return randomBytes(length).toString('hex');
  }
}

// Usage
const encryption = new BankingEncryption();
const key = Buffer.from('my-secret-key-123456789012345678901234');

// Encrypt sensitive data
const sensitiveData = JSON.stringify({
  accountNumber: 'ACC-123456789',
  cardNumber: '4111111111111111',
  cvv: '123',
  pin: '1234'
});

const { encrypted, iv } = encryption.encrypt(sensitiveData, key);
console.log('Encrypted:', encrypted.toString('hex'));
console.log('IV:', iv.toString('hex'));

// Decrypt
const decrypted = encryption.decrypt(encrypted, key, iv);
console.log('Decrypted:', decrypted);

// Hash account number
const hashed = encryption.hashAccountNumber('ACC-123456789');
console.log('Hashed account:', hashed);

// Generate token
const token = encryption.generateToken();
console.log('Token:', token);
```

**Banking Example 2: Binary Protocol for Inter-Bank Communication:**

```typescript
class BankingProtocol {
  // Protocol structure:
  // [4 bytes: message length] [1 byte: message type] [payload]
  
  private MESSAGE_TYPES = {
    PAYMENT: 0x01,
    INQUIRY: 0x02,
    RESPONSE: 0x03,
    ERROR: 0xFF
  };
  
  // Encode payment message
  encodePaymentMessage(payment: {
    fromAccount: string;
    toAccount: string;
    amount: number;
    currency: string;
    reference: string;
  }): Buffer {
    // Create payload
    const payload = JSON.stringify(payment);
    const payloadBuffer = Buffer.from(payload, 'utf8');
    
    // Calculate total length
    const totalLength = 5 + payloadBuffer.length; // header + payload
    
    // Create buffer
    const buffer = Buffer.alloc(totalLength);
    
    // Write header
    buffer.writeUInt32BE(totalLength, 0);     // Message length
    buffer.writeUInt8(this.MESSAGE_TYPES.PAYMENT, 4); // Message type
    
    // Write payload
    payloadBuffer.copy(buffer, 5);
    
    return buffer;
  }
  
  // Decode message
  decodeMessage(buffer: Buffer): { type: string; payload: any } {
    // Read header
    const length = buffer.readUInt32BE(0);
    const type = buffer.readUInt8(4);
    
    // Validate length
    if (length !== buffer.length) {
      throw new Error('Invalid message length');
    }
    
    // Read payload
    const payloadBuffer = buffer.slice(5);
    const payload = JSON.parse(payloadBuffer.toString('utf8'));
    
    // Get type name
    const typeName = Object.keys(this.MESSAGE_TYPES).find(
      key => this.MESSAGE_TYPES[key as keyof typeof this.MESSAGE_TYPES] === type
    ) || 'UNKNOWN';
    
    return { type: typeName, payload };
  }
  
  // Encode transaction in compact binary format
  encodeTransaction(transaction: {
    id: number;
    accountId: string;
    amount: number;
    timestamp: Date;
    type: number;
  }): Buffer {
    const buffer = Buffer.alloc(100);
    let offset = 0;
    
    // Transaction ID (4 bytes)
    buffer.writeUInt32BE(transaction.id, offset);
    offset += 4;
    
    // Account ID (20 bytes, padded)
    buffer.write(transaction.accountId, offset, 20, 'utf8');
    offset += 20;
    
    // Amount (8 bytes, double)
    buffer.writeDoubleBE(transaction.amount, offset);
    offset += 8;
    
    // Timestamp (8 bytes, milliseconds since epoch)
    buffer.writeBigUInt64BE(BigInt(transaction.timestamp.getTime()), offset);
    offset += 8;
    
    // Type (1 byte)
    buffer.writeUInt8(transaction.type, offset);
    offset += 1;
    
    return buffer.slice(0, offset);
  }
  
  // Decode transaction
  decodeTransaction(buffer: Buffer): any {
    let offset = 0;
    
    // Transaction ID
    const id = buffer.readUInt32BE(offset);
    offset += 4;
    
    // Account ID
    const accountId = buffer.toString('utf8', offset, offset + 20).trim();
    offset += 20;
    
    // Amount
    const amount = buffer.readDoubleBE(offset);
    offset += 8;
    
    // Timestamp
    const timestamp = new Date(Number(buffer.readBigUInt64BE(offset)));
    offset += 8;
    
    // Type
    const type = buffer.readUInt8(offset);
    
    return { id, accountId, amount, timestamp, type };
  }
}

// Usage
const protocol = new BankingProtocol();

// Encode payment
const payment = {
  fromAccount: 'ACC-001',
  toAccount: 'ACC-002',
  amount: 1000.50,
  currency: 'AED',
  reference: 'REF-12345'
};

const encoded = protocol.encodePaymentMessage(payment);
console.log('Encoded message:', encoded.toString('hex'));
console.log('Message size:', encoded.length, 'bytes');

// Decode message
const decoded = protocol.decodeMessage(encoded);
console.log('Decoded:', decoded);

// Encode transaction (compact binary)
const transaction = {
  id: 12345,
  accountId: 'ACC-123456789',
  amount: 5000.75,
  timestamp: new Date(),
  type: 1
};

const encodedTxn = protocol.encodeTransaction(transaction);
console.log('\nEncoded transaction:', encodedTxn.toString('hex'));
console.log('Transaction size:', encodedTxn.length, 'bytes');

// Decode transaction
const decodedTxn = protocol.decodeTransaction(encodedTxn);
console.log('Decoded transaction:', decodedTxn);
```

**Banking Example 3: PDF Generation with Buffers:**

```typescript
import * as fs from 'fs';
import { Readable } from 'stream';

class StatementGenerator {
  // Generate account statement as PDF buffer
  async generateStatement(accountId: string, transactions: any[]): Promise<Buffer> {
    // In real application, use library like pdfkit
    // This is a simplified example
    
    const content: string[] = [];
    
    content.push('EMIRATES NBD');
    content.push('ACCOUNT STATEMENT');
    content.push(`Account: ${accountId}`);
    content.push(`Date: ${new Date().toLocaleDateString()}`);
    content.push('\n');
    content.push('TRANSACTIONS:');
    content.push('-'.repeat(50));
    
    transactions.forEach(txn => {
      content.push(
        `${txn.date} | ${txn.type.padEnd(15)} | ${txn.amount.toFixed(2).padStart(12)}`
      );
    });
    
    content.push('-'.repeat(50));
    
    const text = content.join('\n');
    return Buffer.from(text, 'utf8');
  }
  
  // Save statement
  async saveStatement(accountId: string, buffer: Buffer): Promise<void> {
    const filename = `statement_${accountId}_${Date.now()}.txt`;
    await fs.promises.writeFile(filename, buffer);
    console.log(`Statement saved: ${filename}`);
  }
  
  // Stream statement to client
  streamStatement(buffer: Buffer): Readable {
    const stream = new Readable();
    stream.push(buffer);
    stream.push(null); // End of stream
    return stream;
  }
}

// Usage
const generator = new StatementGenerator();

const transactions = [
  { date: '2025-01-01', type: 'Deposit', amount: 1000 },
  { date: '2025-01-02', type: 'Withdrawal', amount: 200 },
  { date: '2025-01-03', type: 'Transfer', amount: 500 }
];

const statementBuffer = await generator.generateStatement('ACC-001', transactions);
console.log('Statement size:', statementBuffer.length, 'bytes');
console.log('\nStatement content:');
console.log(statementBuffer.toString('utf8'));

// Save to file
await generator.saveStatement('ACC-001', statementBuffer);
```

**Banking Example 4: Checksums and Data Integrity:**

```typescript
class DataIntegrity {
  // Calculate checksum for transaction
  calculateChecksum(data: Buffer): Buffer {
    const hash = createHash('sha256');
    hash.update(data);
    return hash.digest();
  }
  
  // Verify data integrity
  verifyChecksum(data: Buffer, checksum: Buffer): boolean {
    const calculated = this.calculateChecksum(data);
    return calculated.equals(checksum);
  }
  
  // Create signed transaction
  createSignedTransaction(transaction: any, privateKey: string): {
    data: Buffer;
    signature: Buffer;
  } {
    // Serialize transaction
    const data = Buffer.from(JSON.stringify(transaction), 'utf8');
    
    // Sign with private key (simplified - use crypto.sign in production)
    const signature = createHash('sha256')
      .update(data)
      .update(privateKey)
      .digest();
    
    return { data, signature };
  }
  
  // Verify signed transaction
  verifySignedTransaction(
    data: Buffer,
    signature: Buffer,
    publicKey: string
  ): boolean {
    // Recreate signature
    const expected = createHash('sha256')
      .update(data)
      .update(publicKey)
      .digest();
    
    return signature.equals(expected);
  }
}

// Usage
const integrity = new DataIntegrity();

const transactionData = Buffer.from(JSON.stringify({
  id: 'TXN-001',
  amount: 1000,
  from: 'ACC-001',
  to: 'ACC-002'
}));

// Calculate checksum
const checksum = integrity.calculateChecksum(transactionData);
console.log('Checksum:', checksum.toString('hex'));

// Verify
const isValid = integrity.verifyChecksum(transactionData, checksum);
console.log('Valid:', isValid);

// Signed transaction
const signed = integrity.createSignedTransaction(
  { id: 'TXN-001', amount: 1000 },
  'private-key-123'
);

console.log('\nSigned transaction:');
console.log('Data:', signed.data.toString('hex'));
console.log('Signature:', signed.signature.toString('hex'));

const verified = integrity.verifySignedTransaction(
  signed.data,
  signed.signature,
  'private-key-123'
);
console.log('Signature valid:', verified);
```

**Performance Comparison:**

```typescript
// String concatenation vs Buffer
function performanceTest() {
  const iterations = 100000;
  
  // Test 1: String concatenation
  console.time('String concatenation');
  let str = '';
  for (let i = 0; i < iterations; i++) {
    str += 'X';
  }
  console.timeEnd('String concatenation');
  
  // Test 2: Buffer
  console.time('Buffer');
  const buffers: Buffer[] = [];
  for (let i = 0; i < iterations; i++) {
    buffers.push(Buffer.from('X'));
  }
  const finalBuffer = Buffer.concat(buffers);
  console.timeEnd('Buffer');
  
  console.log('String length:', str.length);
  console.log('Buffer length:', finalBuffer.length);
}

performanceTest();
```

**Key Takeaways for ENBD:**
- Use Buffers for binary data (files, network, crypto)
- Buffers are faster than strings for binary operations
- Always validate buffer length before reading/writing
- Use `Buffer.alloc()` for security (initialized)
- Buffers are fixed-size (cannot be resized)
- Convert between Buffer and string with encoding
- Use Buffers for encryption/hashing operations
- Checksums ensure data integrity
- Binary protocols are more efficient than JSON/XML

---

### Q24. How do you implement streaming file uploads and downloads in a banking application?

**Answer:**

**Streaming file uploads/downloads** is essential for handling large files efficiently without loading everything into memory at once.

**Banking Use Cases:**
- Document uploads (ID verification, statements)
- Large report downloads (annual statements, transaction history)
- Batch file processing (payment files, reconciliation)
- Image/PDF handling (checks, receipts)
- Data export/import operations

**File Upload with Streams:**

```typescript
import { createWriteStream, createReadStream } from 'fs';
import { pipeline } from 'stream/promises';
import { createHash } from 'crypto';
import * as path from 'path';

// File upload handler
class FileUploadService {
  private uploadDir = './uploads';
  private maxFileSize = 100 * 1024 * 1024; // 100MB
  
  async uploadFile(
    fileStream: NodeJS.ReadableStream,
    filename: string,
    metadata: {
      customerId: string;
      documentType: string;
      fileSize: number;
    }
  ): Promise<{
    success: boolean;
    fileId: string;
    checksum: string;
    uploadedSize: number;
  }> {
    // Validate file size
    if (metadata.fileSize > this.maxFileSize) {
      throw new Error(`File size exceeds maximum allowed (${this.maxFileSize} bytes)`);
    }
    
    // Generate unique file ID
    const fileId = `${metadata.customerId}_${Date.now()}_${filename}`;
    const filePath = path.join(this.uploadDir, fileId);
    
    // Create write stream
    const writeStream = createWriteStream(filePath);
    
    // Track progress
    let uploadedSize = 0;
    const hash = createHash('sha256');
    
    // Transform stream to track progress and calculate checksum
    fileStream.on('data', (chunk: Buffer) => {
      uploadedSize += chunk.length;
      hash.update(chunk);
      
      // Emit progress event
      const progress = (uploadedSize / metadata.fileSize) * 100;
      console.log(`Upload progress: ${progress.toFixed(2)}%`);
      
      // Check if exceeding declared size
      if (uploadedSize > metadata.fileSize) {
        fileStream.destroy(new Error('File size exceeds declared size'));
      }
    });
    
    try {
      // Use pipeline for automatic error handling and cleanup
      await pipeline(fileStream, writeStream);
      
      const checksum = hash.digest('hex');
      
      console.log(`✓ File uploaded successfully: ${fileId}`);
      console.log(`  Size: ${uploadedSize} bytes`);
      console.log(`  Checksum: ${checksum}`);
      
      // Save metadata to database
      await this.saveMetadata({
        fileId,
        filename,
        ...metadata,
        uploadedSize,
        checksum,
        uploadedAt: new Date()
      });
      
      return {
        success: true,
        fileId,
        checksum,
        uploadedSize
      };
    } catch (error) {
      console.error('Upload failed:', error);
      
      // Cleanup partial file
      try {
        await fs.promises.unlink(filePath);
      } catch {}
      
      throw error;
    }
  }
  
  private async saveMetadata(metadata: any): Promise<void> {
    // Save to database
    console.log('Metadata saved:', metadata);
  }
}

// Usage with HTTP server (Express example)
import express from 'express';
import multer from 'multer';

const app = express();
const upload = multer();
const uploadService = new FileUploadService();

app.post('/upload', upload.single('document'), async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({ error: 'No file provided' });
    }
    
    // Create readable stream from buffer
    const { Readable } = require('stream');
    const fileStream = Readable.from(req.file.buffer);
    
    const result = await uploadService.uploadFile(
      fileStream,
      req.file.originalname,
      {
        customerId: req.body.customerId,
        documentType: req.body.documentType,
        fileSize: req.file.size
      }
    );
    
    res.json(result);
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
});
```

**File Download with Streams:**

```typescript
import { stat } from 'fs/promises';

class FileDownloadService {
  private uploadDir = './uploads';
  
  async downloadFile(
    fileId: string,
    response: express.Response
  ): Promise<void> {
    const filePath = path.join(this.uploadDir, fileId);
    
    try {
      // Check if file exists and get stats
      const stats = await stat(filePath);
      
      // Get metadata from database
      const metadata = await this.getMetadata(fileId);
      
      if (!metadata) {
        throw new Error('File not found');
      }
      
      // Set response headers
      response.setHeader('Content-Type', this.getContentType(metadata.filename));
      response.setHeader('Content-Length', stats.size);
      response.setHeader(
        'Content-Disposition',
        `attachment; filename="${metadata.filename}"`
      );
      
      // Create read stream
      const readStream = createReadStream(filePath, {
        highWaterMark: 64 * 1024 // 64KB chunks
      });
      
      // Track download progress
      let downloadedSize = 0;
      readStream.on('data', (chunk: Buffer) => {
        downloadedSize += chunk.length;
        const progress = (downloadedSize / stats.size) * 100;
        console.log(`Download progress: ${progress.toFixed(2)}%`);
      });
      
      // Handle errors
      readStream.on('error', (error) => {
        console.error('Download error:', error);
        response.status(500).end('Download failed');
      });
      
      // Pipe to response
      await pipeline(readStream, response);
      
      console.log(`✓ File downloaded successfully: ${fileId}`);
      
      // Log download activity
      await this.logDownload({
        fileId,
        downloadedAt: new Date(),
        downloadedSize
      });
    } catch (error: any) {
      console.error('Download failed:', error);
      throw error;
    }
  }
  
  private async getMetadata(fileId: string): Promise<any> {
    // Retrieve from database
    return {
      fileId,
      filename: 'statement.pdf',
      documentType: 'statement'
    };
  }
  
  private getContentType(filename: string): string {
    const ext = path.extname(filename).toLowerCase();
    const types: Record<string, string> = {
      '.pdf': 'application/pdf',
      '.png': 'image/png',
      '.jpg': 'image/jpeg',
      '.jpeg': 'image/jpeg',
      '.csv': 'text/csv',
      '.xlsx': 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
    };
    return types[ext] || 'application/octet-stream';
  }
  
  private async logDownload(log: any): Promise<void> {
    console.log('Download logged:', log);
  }
}

// Express route
const downloadService = new FileDownloadService();

app.get('/download/:fileId', async (req, res) => {
  try {
    await downloadService.downloadFile(req.params.fileId, res);
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
});
```

**Chunked Upload for Very Large Files:**

```typescript
class ChunkedUploadService {
  private uploadDir = './uploads/chunks';
  private uploads = new Map<string, {
    totalChunks: number;
    receivedChunks: Set<number>;
    filename: string;
    metadata: any;
  }>();
  
  async initializeUpload(
    filename: string,
    totalChunks: number,
    metadata: any
  ): Promise<string> {
    const uploadId = `upload_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
    
    this.uploads.set(uploadId, {
      totalChunks,
      receivedChunks: new Set(),
      filename,
      metadata
    });
    
    console.log(`Upload initialized: ${uploadId}`);
    console.log(`Total chunks: ${totalChunks}`);
    
    return uploadId;
  }
  
  async uploadChunk(
    uploadId: string,
    chunkNumber: number,
    chunkStream: NodeJS.ReadableStream
  ): Promise<{ success: boolean; receivedChunks: number; totalChunks: number }> {
    const upload = this.uploads.get(uploadId);
    
    if (!upload) {
      throw new Error('Upload not found');
    }
    
    // Save chunk to disk
    const chunkPath = path.join(this.uploadDir, `${uploadId}_chunk_${chunkNumber}`);
    const writeStream = createWriteStream(chunkPath);
    
    await pipeline(chunkStream, writeStream);
    
    // Mark chunk as received
    upload.receivedChunks.add(chunkNumber);
    
    console.log(`Chunk ${chunkNumber}/${upload.totalChunks} received`);
    
    return {
      success: true,
      receivedChunks: upload.receivedChunks.size,
      totalChunks: upload.totalChunks
    };
  }
  
  async finalizeUpload(uploadId: string): Promise<{
    success: boolean;
    fileId: string;
    finalPath: string;
  }> {
    const upload = this.uploads.get(uploadId);
    
    if (!upload) {
      throw new Error('Upload not found');
    }
    
    // Check all chunks received
    if (upload.receivedChunks.size !== upload.totalChunks) {
      throw new Error(
        `Missing chunks: received ${upload.receivedChunks.size}/${upload.totalChunks}`
      );
    }
    
    console.log('All chunks received, assembling file...');
    
    // Assemble chunks
    const finalPath = path.join('./uploads', `${uploadId}_${upload.filename}`);
    const writeStream = createWriteStream(finalPath);
    
    // Write chunks in order
    for (let i = 0; i < upload.totalChunks; i++) {
      const chunkPath = path.join(this.uploadDir, `${uploadId}_chunk_${i}`);
      const chunkStream = createReadStream(chunkPath);
      
      await new Promise((resolve, reject) => {
        chunkStream.on('data', (chunk) => writeStream.write(chunk));
        chunkStream.on('end', resolve);
        chunkStream.on('error', reject);
      });
      
      // Delete chunk file
      await fs.promises.unlink(chunkPath);
    }
    
    writeStream.end();
    
    // Cleanup
    this.uploads.delete(uploadId);
    
    console.log(`✓ File assembled successfully: ${finalPath}`);
    
    return {
      success: true,
      fileId: uploadId,
      finalPath
    };
  }
}

// Express routes for chunked upload
const chunkedUpload = new ChunkedUploadService();

app.post('/upload/init', async (req, res) => {
  try {
    const { filename, totalChunks, metadata } = req.body;
    const uploadId = await chunkedUpload.initializeUpload(
      filename,
      totalChunks,
      metadata
    );
    res.json({ uploadId });
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
});

app.post('/upload/chunk/:uploadId/:chunkNumber', upload.single('chunk'), async (req, res) => {
  try {
    const { uploadId, chunkNumber } = req.params;
    const { Readable } = require('stream');
    const chunkStream = Readable.from(req.file.buffer);
    
    const result = await chunkedUpload.uploadChunk(
      uploadId,
      parseInt(chunkNumber),
      chunkStream
    );
    
    res.json(result);
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
});

app.post('/upload/finalize/:uploadId', async (req, res) => {
  try {
    const result = await chunkedUpload.finalizeUpload(req.params.uploadId);
    res.json(result);
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
});
```

**Resumable Upload:**

```typescript
class ResumableUploadService {
  private uploadDir = './uploads/resumable';
  
  async getUploadStatus(uploadId: string): Promise<{
    bytesReceived: number;
    complete: boolean;
  }> {
    const filePath = path.join(this.uploadDir, uploadId);
    
    try {
      const stats = await stat(filePath);
      return {
        bytesReceived: stats.size,
        complete: false
      };
    } catch {
      return {
        bytesReceived: 0,
        complete: false
      };
    }
  }
  
  async resumeUpload(
    uploadId: string,
    startByte: number,
    fileStream: NodeJS.ReadableStream,
    totalSize: number
  ): Promise<{ success: boolean; bytesReceived: number }> {
    const filePath = path.join(this.uploadDir, uploadId);
    
    // Open file in append mode
    const writeStream = createWriteStream(filePath, {
      flags: 'a', // Append mode
      start: startByte
    });
    
    let bytesReceived = startByte;
    
    fileStream.on('data', (chunk: Buffer) => {
      bytesReceived += chunk.length;
      const progress = (bytesReceived / totalSize) * 100;
      console.log(`Upload progress: ${progress.toFixed(2)}%`);
    });
    
    await pipeline(fileStream, writeStream);
    
    const complete = bytesReceived >= totalSize;
    
    console.log(`Upload ${complete ? 'completed' : 'paused'}: ${bytesReceived}/${totalSize} bytes`);
    
    return {
      success: true,
      bytesReceived
    };
  }
}

// Express route
const resumableUpload = new ResumableUploadService();

app.get('/upload/status/:uploadId', async (req, res) => {
  try {
    const status = await resumableUpload.getUploadStatus(req.params.uploadId);
    res.json(status);
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
});

app.post('/upload/resume/:uploadId', upload.single('file'), async (req, res) => {
  try {
    const { uploadId } = req.params;
    const { startByte, totalSize } = req.body;
    const { Readable } = require('stream');
    const fileStream = Readable.from(req.file.buffer);
    
    const result = await resumableUpload.resumeUpload(
      uploadId,
      parseInt(startByte),
      fileStream,
      parseInt(totalSize)
    );
    
    res.json(result);
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
});
```

**Stream Transformation During Upload:**

```typescript
import { Transform } from 'stream';
import zlib from 'zlib';

class SecureUploadService {
  async uploadWithCompression(
    fileStream: NodeJS.ReadableStream,
    filename: string
  ): Promise<void> {
    const outputPath = `./uploads/compressed_${filename}.gz`;
    
    // Compress while uploading
    const gzip = zlib.createGzip({
      level: 6 // Compression level
    });
    
    const writeStream = createWriteStream(outputPath);
    
    console.log('Uploading with compression...');
    
    await pipeline(
      fileStream,
      gzip,        // Compress
      writeStream  // Write
    );
    
    console.log('Upload with compression complete');
  }
  
  async uploadWithEncryption(
    fileStream: NodeJS.ReadableStream,
    filename: string,
    key: Buffer
  ): Promise<void> {
    const { createCipheriv, randomBytes } = require('crypto');
    
    const iv = randomBytes(16);
    const cipher = createCipheriv('aes-256-cbc', key, iv);
    
    const outputPath = `./uploads/encrypted_${filename}`;
    const writeStream = createWriteStream(outputPath);
    
    // Write IV first
    writeStream.write(iv);
    
    console.log('Uploading with encryption...');
    
    await pipeline(
      fileStream,
      cipher,      // Encrypt
      writeStream  // Write
    );
    
    console.log('Upload with encryption complete');
  }
  
  async uploadWithVirusScan(
    fileStream: NodeJS.ReadableStream,
    filename: string
  ): Promise<void> {
    // Create virus scanner transform stream
    class VirusScannerStream extends Transform {
      private scannedBytes = 0;
      
      _transform(chunk: Buffer, encoding: string, callback: Function) {
        this.scannedBytes += chunk.length;
        
        // Simulate virus scanning
        const hasVirus = this.scanChunk(chunk);
        
        if (hasVirus) {
          callback(new Error('Virus detected!'));
        } else {
          callback(null, chunk);
        }
      }
      
      scanChunk(chunk: Buffer): boolean {
        // In production, integrate with actual antivirus
        // For demo, just check for suspicious patterns
        return false;
      }
    }
    
    const scanner = new VirusScannerStream();
    const outputPath = `./uploads/${filename}`;
    const writeStream = createWriteStream(outputPath);
    
    console.log('Uploading with virus scan...');
    
    try {
      await pipeline(
        fileStream,
        scanner,     // Scan for viruses
        writeStream  // Write if clean
      );
      
      console.log('Upload complete - file is clean');
    } catch (error: any) {
      console.error('Upload failed:', error.message);
      
      // Delete infected file
      await fs.promises.unlink(outputPath).catch(() => {});
      
      throw error;
    }
  }
}
```

**Key Takeaways for ENBD:**
- Use streams for memory-efficient file handling
- Implement chunked uploads for large files
- Support resumable uploads for reliability
- Calculate checksums to verify integrity
- Add virus scanning during upload
- Compress files to save storage
- Encrypt sensitive documents
- Track upload/download progress
- Clean up on errors
- Set appropriate file size limits

---

### Q25. How do you handle stream errors and implement proper error handling in production?

**Answer:**

**Stream error handling** is critical for production systems. Unhandled stream errors can crash your application.

**Common Stream Errors:**
- File not found
- Permission denied
- Disk full
- Network interruption
- Data corruption
- Timeout
- Backpressure issues

**Basic Error Handling:**

```typescript
import { createReadStream, createWriteStream } from 'fs';
import { pipeline } from 'stream/promises';

// ❌ BAD: Unhandled stream errors
function badErrorHandling() {
  const read = createReadStream('input.txt');
  const write = createWriteStream('output.txt');
  
  // These errors will crash the application!
  read.pipe(write);
}

// ✅ GOOD: Proper error handling
async function goodErrorHandling() {
  try {
    const read = createReadStream('input.txt');
    const write = createWriteStream('output.txt');
    
    await pipeline(read, write);
    console.log('File copied successfully');
  } catch (error) {
    console.error('File copy failed:', error);
    // Handle error appropriately
  }
}
```

**Comprehensive Error Handling:**

```typescript
class RobustFileProcessor {
  async processFile(inputPath: string, outputPath: string): Promise<void> {
    let readStream: fs.ReadStream | null = null;
    let writeStream: fs.WriteStream | null = null;
    
    try {
      // Check if input file exists
      await fs.promises.access(inputPath, fs.constants.R_OK);
      
      // Create streams
      readStream = createReadStream(inputPath);
      writeStream = createWriteStream(outputPath);
      
      // Set up error handlers before piping
      readStream.on('error', (error) => {
        console.error('Read stream error:', error.message);
        this.handleReadError(error, inputPath);
      });
      
      writeStream.on('error', (error) => {
        console.error('Write stream error:', error.message);
        this.handleWriteError(error, outputPath);
      });
      
      // Use pipeline for automatic cleanup
      await pipeline(readStream, writeStream);
      
      console.log('✓ File processed successfully');
      
    } catch (error: any) {
      console.error('❌ Processing failed:', error.message);
      
      // Cleanup partial output
      if (writeStream) {
        writeStream.destroy();
        
        try {
          await fs.promises.unlink(outputPath);
          console.log('Cleaned up partial output file');
        } catch {}
      }
      
      throw error;
      
    } finally {
      // Ensure streams are closed
      if (readStream && !readStream.destroyed) {
        readStream.destroy();
      }
      if (writeStream && !writeStream.destroyed) {
        writeStream.destroy();
      }
    }
  }
  
  private handleReadError(error: Error, path: string): void {
    if ((error as any).code === 'ENOENT') {
      console.error(`File not found: ${path}`);
    } else if ((error as any).code === 'EACCES') {
      console.error(`Permission denied: ${path}`);
    } else {
      console.error(`Read error: ${error.message}`);
    }
  }
  
  private handleWriteError(error: Error, path: string): void {
    if ((error as any).code === 'ENOSPC') {
      console.error('Disk full!');
    } else if ((error as any).code === 'EACCES') {
      console.error(`Permission denied: ${path}`);
    } else {
      console.error(`Write error: ${error.message}`);
    }
  }
}
```

**Banking Example: Transaction File Processing with Error Recovery:**

```typescript
import { Transform } from 'stream';
import * as fs from 'fs';

class TransactionFileProcessor {
  private errorLog: any[] = [];
  private processedCount = 0;
  private errorCount = 0;
  
  async processTransactionFile(
    inputPath: string,
    outputPath: string,
    errorPath: string
  ): Promise<{
    processed: number;
    errors: number;
    errorLogPath: string;
  }> {
    console.log('Starting transaction file processing...');
    
    const readStream = createReadStream(inputPath, { encoding: 'utf8' });
    const writeStream = createWriteStream(outputPath);
    const errorStream = createWriteStream(errorPath);
    
    // Transform stream with error handling
    const processTransform = new Transform({
      objectMode: true,
      transform: (line: string, encoding, callback) => {
        try {
          // Skip empty lines
          if (!line.trim()) {
            return callback();
          }
          
          // Parse transaction
          const transaction = JSON.parse(line);
          
          // Validate transaction
          const validation = this.validateTransaction(transaction);
          
          if (validation.valid) {
            // Process valid transaction
            const processed = this.processTransaction(transaction);
            this.processedCount++;
            
            // Output processed transaction
            callback(null, JSON.stringify(processed) + '\n');
          } else {
            // Log invalid transaction
            this.errorCount++;
            this.logError(transaction, validation.errors, errorStream);
            
            // Continue processing
            callback();
          }
        } catch (error: any) {
          // Handle parsing errors
          this.errorCount++;
          this.logError(
            { raw: line },
            [`Parse error: ${error.message}`],
            errorStream
          );
          
          // Don't stop processing
          callback();
        }
      }
    });
    
    // Split lines
    let buffer = '';
    const lineSplitter = new Transform({
      transform(chunk: Buffer, encoding, callback) {
        buffer += chunk.toString();
        const lines = buffer.split('\n');
        buffer = lines.pop() || '';
        
        for (const line of lines) {
          this.push(line);
        }
        callback();
      },
      flush(callback) {
        if (buffer) {
          this.push(buffer);
        }
        callback();
      }
    });
    
    try {
      // Set up error handlers
      readStream.on('error', (error) => {
        console.error('❌ Read error:', error.message);
      });
      
      writeStream.on('error', (error) => {
        console.error('❌ Write error:', error.message);
      });
      
      processTransform.on('error', (error) => {
        console.error('❌ Process error:', error.message);
      });
      
      // Process pipeline
      await pipeline(
        readStream,
        lineSplitter,
        processTransform,
        writeStream
      );
      
      // Close error log
      errorStream.end();
      
      console.log('\n✓ Processing complete:');
      console.log(`  Processed: ${this.processedCount}`);
      console.log(`  Errors: ${this.errorCount}`);
      
      return {
        processed: this.processedCount,
        errors: this.errorCount,
        errorLogPath: errorPath
      };
      
    } catch (error: any) {
      console.error('❌ Fatal error:', error.message);
      
      // Cleanup
      errorStream.end();
      
      throw error;
    }
  }
  
  private validateTransaction(transaction: any): {
    valid: boolean;
    errors: string[];
  } {
    const errors: string[] = [];
    
    if (!transaction.id) {
      errors.push('Missing transaction ID');
    }
    
    if (!transaction.accountId) {
      errors.push('Missing account ID');
    }
    
    if (typeof transaction.amount !== 'number' || transaction.amount <= 0) {
      errors.push('Invalid amount');
    }
    
    if (!['deposit', 'withdrawal', 'transfer'].includes(transaction.type)) {
      errors.push('Invalid transaction type');
    }
    
    return {
      valid: errors.length === 0,
      errors
    };
  }
  
  private processTransaction(transaction: any): any {
    // Add processing timestamp
    return {
      ...transaction,
      processedAt: new Date().toISOString(),
      status: 'processed'
    };
  }
  
  private logError(
    transaction: any,
    errors: string[],
    errorStream: fs.WriteStream
  ): void {
    const errorEntry = {
      transaction,
      errors,
      timestamp: new Date().toISOString()
    };
    
    this.errorLog.push(errorEntry);
    errorStream.write(JSON.stringify(errorEntry) + '\n');
  }
}

// Usage
const processor = new TransactionFileProcessor();
// await processor.processTransactionFile(
//   './transactions.jsonl',
//   './processed.jsonl',
//   './errors.jsonl'
// );
```

**Retry Logic for Stream Operations:**

```typescript
class RetryableStreamOperation {
  async processWithRetry(
    operation: () => Promise<void>,
    maxRetries: number = 3,
    delayMs: number = 1000
  ): Promise<void> {
    let lastError: Error | null = null;
    
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        console.log(`Attempt ${attempt}/${maxRetries}...`);
        await operation();
        console.log('✓ Operation succeeded');
        return;
      } catch (error: any) {
        lastError = error;
        console.error(`✗ Attempt ${attempt} failed:`, error.message);
        
        if (attempt < maxRetries) {
          // Exponential backoff
          const delay = delayMs * Math.pow(2, attempt - 1);
          console.log(`Retrying in ${delay}ms...`);
          await this.sleep(delay);
        }
      }
    }
    
    throw new Error(
      `Operation failed after ${maxRetries} attempts: ${lastError?.message}`
    );
  }
  
  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Usage
const retryable = new RetryableStreamOperation();

await retryable.processWithRetry(async () => {
  const read = createReadStream('input.txt');
  const write = createWriteStream('output.txt');
  await pipeline(read, write);
});
```

**Circuit Breaker Pattern for Stream Operations:**

```typescript
class StreamCircuitBreaker {
  private failureCount = 0;
  private successCount = 0;
  private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED';
  private lastFailureTime: number = 0;
  
  constructor(
    private failureThreshold: number = 5,
    private resetTimeout: number = 60000 // 1 minute
  ) {}
  
  async execute(operation: () => Promise<void>): Promise<void> {
    // Check if circuit is open
    if (this.state === 'OPEN') {
      const timeSinceFailure = Date.now() - this.lastFailureTime;
      
      if (timeSinceFailure >= this.resetTimeout) {
        console.log('Circuit breaker: transitioning to HALF_OPEN');
        this.state = 'HALF_OPEN';
      } else {
        throw new Error('Circuit breaker is OPEN - operation rejected');
      }
    }
    
    try {
      await operation();
      
      // Operation succeeded
      this.onSuccess();
      
    } catch (error) {
      // Operation failed
      this.onFailure();
      throw error;
    }
  }
  
  private onSuccess(): void {
    this.failureCount = 0;
    this.successCount++;
    
    if (this.state === 'HALF_OPEN') {
      console.log('Circuit breaker: transitioning to CLOSED');
      this.state = 'CLOSED';
    }
  }
  
  private onFailure(): void {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    
    if (this.failureCount >= this.failureThreshold) {
      console.log('Circuit breaker: transitioning to OPEN');
      this.state = 'OPEN';
    }
  }
  
  getState(): string {
    return this.state;
  }
}

// Usage
const circuitBreaker = new StreamCircuitBreaker(3, 30000);

try {
  await circuitBreaker.execute(async () => {
    const read = createReadStream('input.txt');
    const write = createWriteStream('output.txt');
    await pipeline(read, write);
  });
} catch (error) {
  console.error('Operation failed or circuit breaker is open');
}
```

**Graceful Degradation:**

```typescript
class GracefulStreamProcessor {
  async processWithFallback(
    primaryPath: string,
    fallbackPath: string,
    outputPath: string
  ): Promise<void> {
    try {
      console.log('Attempting primary source...');
      await this.processFile(primaryPath, outputPath);
      console.log('✓ Processed from primary source');
    } catch (primaryError) {
      console.warn('⚠️ Primary source failed, trying fallback...');
      
      try {
        await this.processFile(fallbackPath, outputPath);
        console.log('✓ Processed from fallback source');
      } catch (fallbackError) {
        console.error('❌ Both sources failed');
        throw new Error('All sources exhausted');
      }
    }
  }
  
  private async processFile(inputPath: string, outputPath: string): Promise<void> {
    const read = createReadStream(inputPath);
    const write = createWriteStream(outputPath);
    await pipeline(read, write);
  }
}
```

**Key Takeaways for ENBD:**
- Always handle stream errors
- Use `pipeline()` for automatic cleanup
- Implement retry logic with exponential backoff
- Use circuit breakers to prevent cascading failures
- Log errors for debugging
- Clean up partial files on error
- Implement fallback mechanisms
- Use try-finally for resource cleanup
- Monitor error rates in production
- Set appropriate timeouts

---

### Q26. How do you optimize stream performance? Explain highWaterMark and buffering strategies.

**Answer:**

**Stream performance optimization** is crucial for high-throughput banking applications.

**highWaterMark** - Controls internal buffer size:
- Readable: How much data to buffer
- Writable: When to signal backpressure
- Default: 16KB (16384 bytes) for buffers, 16 for object mode

**Impact of highWaterMark:**

```typescript
import { performance } from 'perf_hooks';

class StreamPerformanceTest {
  async testDifferentBufferSizes(filePath: string): Promise<void> {
    const sizes = [
      { size: 16 * 1024, name: '16KB (default)' },
      { size: 64 * 1024, name: '64KB' },
      { size: 256 * 1024, name: '256KB' },
      { size: 1024 * 1024, name: '1MB' }
    ];
    
    for (const { size, name } of sizes) {
      const start = performance.now();
      
      const readStream = createReadStream(filePath, {
        highWaterMark: size
      });
      
      const writeStream = createWriteStream(`./output_${name}.txt`, {
        highWaterMark: size
      });
      
      await pipeline(readStream, writeStream);
      
      const duration = performance.now() - start;
      console.log(`${name}: ${duration.toFixed(2)}ms`);
    }
  }
}

// Results (example with 100MB file):
// 16KB (default): 1250ms
// 64KB: 850ms (32% faster)
// 256KB: 720ms (42% faster)
// 1MB: 680ms (46% faster)
```

**Optimal Buffer Sizing:**

```typescript
class OptimizedFileProcessor {
  getOptimalBufferSize(fileSize: number): number {
    // Small files: 16KB
    if (fileSize < 1024 * 1024) {
      return 16 * 1024;
    }
    
    // Medium files (1MB - 100MB): 256KB
    if (fileSize < 100 * 1024 * 1024) {
      return 256 * 1024;
    }
    
    // Large files (100MB+): 1MB
    return 1024 * 1024;
  }
  
  async processFileOptimized(inputPath: string, outputPath: string): Promise<void> {
    // Get file size
    const stats = await fs.promises.stat(inputPath);
    const bufferSize = this.getOptimalBufferSize(stats.size);
    
    console.log(`File size: ${(stats.size / 1024 / 1024).toFixed(2)}MB`);
    console.log(`Using buffer: ${(bufferSize / 1024).toFixed(0)}KB`);
    
    const readStream = createReadStream(inputPath, {
      highWaterMark: bufferSize
    });
    
    const writeStream = createWriteStream(outputPath, {
      highWaterMark: bufferSize
    });
    
    await pipeline(readStream, writeStream);
  }
}
```

**Banking Example: High-Performance Transaction Processing:**

```typescript
class HighPerformanceTransactionProcessor {
  async processBulkTransactions(
    inputPath: string,
    outputPath: string
  ): Promise<{
    processed: number;
    duration: number;
    throughput: number;
  }> {
    const start = performance.now();
    let processedCount = 0;
    
    // Optimize for high throughput
    const readStream = createReadStream(inputPath, {
      highWaterMark: 512 * 1024, // 512KB chunks
      encoding: 'utf8'
    });
    
    const writeStream = createWriteStream(outputPath, {
      highWaterMark: 512 * 1024
    });
    
    // Batch processor transform
    class BatchProcessor extends Transform {
      private batch: any[] = [];
      private batchSize = 1000;
      
      constructor() {
        super({ objectMode: true, highWaterMark: 100 });
      }
      
      _transform(transaction: any, encoding: string, callback: Function) {
        this.batch.push(transaction);
        
        if (this.batch.length >= this.batchSize) {
          // Process batch
          const processed = this.processBatch(this.batch);
          this.batch = [];
          
          // Output batch
          callback(null, processed);
        } else {
          callback();
        }
      }
      
      _flush(callback: Function) {
        if (this.batch.length > 0) {
          const processed = this.processBatch(this.batch);
          callback(null, processed);
        } else {
          callback();
        }
      }
      
      processBatch(transactions: any[]): string {
        // Bulk process transactions
        const processed = transactions.map(txn => ({
          ...txn,
          processedAt: Date.now(),
          status: 'completed'
        }));
        
        processedCount += processed.length;
        
        return processed.map(t => JSON.stringify(t)).join('\n') + '\n';
      }
    }
    
    // Parse JSON lines
    const parser = new Transform({
      objectMode: true,
      highWaterMark: 100,
      transform(line: string, encoding, callback) {
        try {
          if (line.trim()) {
            callback(null, JSON.parse(line));
          } else {
            callback();
          }
        } catch (error) {
          callback();
        }
      }
    });
    
    // Line splitter
    let buffer = '';
    const splitter = new Transform({
      highWaterMark: 512 * 1024,
      transform(chunk: Buffer, encoding, callback) {
        buffer += chunk.toString();
        const lines = buffer.split('\n');
        buffer = lines.pop() || '';
        
        for (const line of lines) {
          this.push(line + '\n');
        }
        callback();
      },
      flush(callback) {
        if (buffer) {
          this.push(buffer);
        }
        callback();
      }
    });
    
    const batchProcessor = new BatchProcessor();
    
    await pipeline(
      readStream,
      splitter,
      parser,
      batchProcessor,
      writeStream
    );
    
    const duration = performance.now() - start;
    const throughput = processedCount / (duration / 1000);
    
    console.log(`\nPerformance Metrics:`);
    console.log(`  Processed: ${processedCount} transactions`);
    console.log(`  Duration: ${duration.toFixed(2)}ms`);
    console.log(`  Throughput: ${throughput.toFixed(0)} txn/sec`);
    
    return {
      processed: processedCount,
      duration,
      throughput
    };
  }
}

// Usage
const processor = new HighPerformanceTransactionProcessor();
// await processor.processBulkTransactions(
//   './transactions.jsonl',
//   './processed.jsonl'
// );
```

**Parallel Stream Processing:**

```typescript
import { Worker } from 'worker_threads';

class ParallelStreamProcessor {
  async processInParallel(
    inputPath: string,
    outputPath: string,
    workerCount: number = 4
  ): Promise<void> {
    console.log(`Processing with ${workerCount} workers...`);
    
    // Split file into chunks
    const stats = await fs.promises.stat(inputPath);
    const chunkSize = Math.ceil(stats.size / workerCount);
    
    const workers: Promise<void>[] = [];
    
    for (let i = 0; i < workerCount; i++) {
      const start = i * chunkSize;
      const end = Math.min((i + 1) * chunkSize, stats.size);
      
      workers.push(
        this.processChunk(inputPath, `${outputPath}.${i}`, start, end)
      );
    }
    
    // Wait for all workers
    await Promise.all(workers);
    
    // Merge results
    await this.mergeFiles(outputPath, workerCount);
    
    console.log('✓ Parallel processing complete');
  }
  
  private async processChunk(
    inputPath: string,
    outputPath: string,
    start: number,
    end: number
  ): Promise<void> {
    const readStream = createReadStream(inputPath, {
      start,
      end: end - 1,
      highWaterMark: 256 * 1024
    });
    
    const writeStream = createWriteStream(outputPath);
    
    await pipeline(readStream, writeStream);
  }
  
  private async mergeFiles(outputPath: string, chunkCount: number): Promise<void> {
    const writeStream = createWriteStream(outputPath);
    
    for (let i = 0; i < chunkCount; i++) {
      const chunkPath = `${outputPath}.${i}`;
      const readStream = createReadStream(chunkPath);
      
      await new Promise((resolve, reject) => {
        readStream.on('data', chunk => writeStream.write(chunk));
        readStream.on('end', resolve);
        readStream.on('error', reject);
      });
      
      // Delete chunk
      await fs.promises.unlink(chunkPath);
    }
    
    writeStream.end();
  }
}
```

**Memory Monitoring:**

```typescript
class MemoryEfficientProcessor {
  private readonly maxMemoryMB = 100;
  
  async processWithMemoryLimit(inputPath: string, outputPath: string): Promise<void> {
    const startMemory = process.memoryUsage();
    
    let peakMemory = 0;
    const memoryMonitor = setInterval(() => {
      const current = process.memoryUsage().heapUsed / 1024 / 1024;
      peakMemory = Math.max(peakMemory, current);
      
      if (current > this.maxMemoryMB) {
        console.warn(`⚠️ Memory usage high: ${current.toFixed(2)}MB`);
      }
    }, 100);
    
    try {
      const readStream = createReadStream(inputPath, {
        highWaterMark: 64 * 1024 // Conservative buffer
      });
      
      const writeStream = createWriteStream(outputPath, {
        highWaterMark: 64 * 1024
      });
      
      await pipeline(readStream, writeStream);
      
      const endMemory = process.memoryUsage();
      
      console.log('\nMemory Usage:');
      console.log(`  Peak: ${peakMemory.toFixed(2)}MB`);
      console.log(`  Start: ${(startMemory.heapUsed / 1024 / 1024).toFixed(2)}MB`);
      console.log(`  End: ${(endMemory.heapUsed / 1024 / 1024).toFixed(2)}MB`);
      
    } finally {
      clearInterval(memoryMonitor);
    }
  }
}
```

**Key Takeaways for ENBD:**
- Increase `highWaterMark` for large files (256KB - 1MB)
- Use smaller buffers for many concurrent streams
- Batch processing improves throughput
- Monitor memory usage in production
- Consider parallel processing for very large files
- Match read and write buffer sizes
- Profile before optimizing
- Balance memory usage vs performance
- Use object mode carefully (higher overhead)
- Test with realistic data volumes

---

### **Q27: How do you implement custom Transform streams for data processing in Node.js?**

**Answer:**

Transform streams are Duplex streams where the output is computed from the input. They're perfect for data transformation pipelines like encryption, compression, parsing, or validation.

**1. Basic Transform Stream Structure:**

```typescript
import { Transform, TransformCallback } from 'stream';

// Simple Transform stream
class UpperCaseTransform extends Transform {
  constructor(options?: any) {
    super(options);
  }

  _transform(chunk: Buffer, encoding: string, callback: TransformCallback): void {
    // Transform the data
    const transformed = chunk.toString().toUpperCase();
    
    // Push transformed data to output
    this.push(transformed);
    
    // Signal completion
    callback();
  }

  _flush(callback: TransformCallback): void {
    // Called at the end of the stream
    // Push any remaining data
    callback();
  }
}

// Usage
const upperCaseStream = new UpperCaseTransform();
process.stdin.pipe(upperCaseStream).pipe(process.stdout);
```

**2. Banking Example - Transaction Validator Transform:**

```typescript
import { Transform, TransformCallback } from 'stream';
import { createReadStream, createWriteStream } from 'fs';
import { pipeline } from 'stream/promises';

interface Transaction {
  id: string;
  accountNumber: string;
  amount: number;
  type: 'debit' | 'credit';
  timestamp: string;
  currency: string;
}

interface ValidationResult {
  transaction: Transaction;
  isValid: boolean;
  errors: string[];
  warnings: string[];
}

// Transform stream for validating transactions
class TransactionValidatorTransform extends Transform {
  private validationRules: {
    maxAmount: number;
    minAmount: number;
    allowedCurrencies: string[];
  };
  private validCount: number = 0;
  private invalidCount: number = 0;

  constructor(options?: any) {
    super({ ...options, objectMode: true });
    
    this.validationRules = {
      maxAmount: 1000000, // 1 million AED max per transaction
      minAmount: 1, // 1 AED minimum
      allowedCurrencies: ['AED', 'USD', 'EUR', 'GBP']
    };
  }

  _transform(chunk: any, encoding: string, callback: TransformCallback): void {
    try {
      const transaction: Transaction = typeof chunk === 'string' 
        ? JSON.parse(chunk) 
        : chunk;

      const result = this.validateTransaction(transaction);
      
      if (result.isValid) {
        this.validCount++;
      } else {
        this.invalidCount++;
      }

      // Push validation result
      this.push(JSON.stringify(result) + '\n');
      callback();
    } catch (error) {
      callback(error as Error);
    }
  }

  _flush(callback: TransformCallback): void {
    // Push summary at the end
    const summary = {
      type: 'SUMMARY',
      totalValid: this.validCount,
      totalInvalid: this.invalidCount,
      timestamp: new Date().toISOString()
    };
    
    this.push(JSON.stringify(summary) + '\n');
    callback();
  }

  private validateTransaction(txn: Transaction): ValidationResult {
    const errors: string[] = [];
    const warnings: string[] = [];

    // Validate required fields
    if (!txn.id) errors.push('Transaction ID is required');
    if (!txn.accountNumber) errors.push('Account number is required');
    if (typeof txn.amount !== 'number') errors.push('Amount must be a number');
    if (!txn.type || !['debit', 'credit'].includes(txn.type)) {
      errors.push('Type must be debit or credit');
    }

    // Validate amount
    if (txn.amount < this.validationRules.minAmount) {
      errors.push(`Amount must be at least ${this.validationRules.minAmount}`);
    }
    if (txn.amount > this.validationRules.maxAmount) {
      errors.push(`Amount exceeds maximum of ${this.validationRules.maxAmount}`);
    }

    // Validate currency
    if (!this.validationRules.allowedCurrencies.includes(txn.currency)) {
      errors.push(`Currency ${txn.currency} is not supported`);
    }

    // Warnings for large amounts
    if (txn.amount > 100000) {
      warnings.push('Large transaction - requires manager approval');
    }

    // Validate account number format (ENBD format: 12 digits)
    if (txn.accountNumber && !/^\d{12}$/.test(txn.accountNumber)) {
      errors.push('Account number must be 12 digits');
    }

    return {
      transaction: txn,
      isValid: errors.length === 0,
      errors,
      warnings
    };
  }
}

// Usage example
async function validateTransactionFile() {
  const validator = new TransactionValidatorTransform();
  
  await pipeline(
    createReadStream('transactions.jsonl'), // JSONL format
    validator,
    createWriteStream('validated-transactions.jsonl')
  );
  
  console.log('Validation complete!');
}
```

**3. Transform Stream for Data Enrichment:**

```typescript
import { Transform, TransformCallback } from 'stream';

interface BasicTransaction {
  id: string;
  accountNumber: string;
  amount: number;
}

interface EnrichedTransaction extends BasicTransaction {
  customerName?: string;
  accountType?: string;
  branch?: string;
  category?: string;
  riskLevel?: string;
}

// Transform that enriches transactions with customer data
class TransactionEnricherTransform extends Transform {
  private customerCache: Map<string, any>;
  private cacheHits: number = 0;
  private cacheMisses: number = 0;

  constructor(private dbClient: any, options?: any) {
    super({ ...options, objectMode: true });
    this.customerCache = new Map();
  }

  async _transform(chunk: any, encoding: string, callback: TransformCallback): Promise<void> {
    try {
      const transaction: BasicTransaction = typeof chunk === 'string'
        ? JSON.parse(chunk)
        : chunk;

      // Check cache first
      let customerData = this.customerCache.get(transaction.accountNumber);
      
      if (customerData) {
        this.cacheHits++;
      } else {
        this.cacheMisses++;
        // Fetch from database
        customerData = await this.fetchCustomerData(transaction.accountNumber);
        this.customerCache.set(transaction.accountNumber, customerData);
      }

      // Enrich transaction
      const enriched: EnrichedTransaction = {
        ...transaction,
        customerName: customerData?.name,
        accountType: customerData?.accountType,
        branch: customerData?.branch,
        category: this.categorizeTransaction(transaction),
        riskLevel: this.assessRisk(transaction, customerData)
      };

      this.push(enriched);
      callback();
    } catch (error) {
      callback(error as Error);
    }
  }

  _flush(callback: TransformCallback): void {
    console.log(`Cache hits: ${this.cacheHits}, Cache misses: ${this.cacheMisses}`);
    callback();
  }

  private async fetchCustomerData(accountNumber: string): Promise<any> {
    // Simulated database call
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve({
          name: `Customer-${accountNumber}`,
          accountType: 'Savings',
          branch: 'Dubai Main Branch'
        });
      }, 10);
    });
  }

  private categorizeTransaction(txn: BasicTransaction): string {
    if (txn.amount < 100) return 'SMALL';
    if (txn.amount < 10000) return 'MEDIUM';
    return 'LARGE';
  }

  private assessRisk(txn: BasicTransaction, customer: any): string {
    if (txn.amount > 500000) return 'HIGH';
    if (txn.amount > 100000) return 'MEDIUM';
    return 'LOW';
  }
}
```

**4. Transform Stream with Batching:**

```typescript
import { Transform, TransformCallback } from 'stream';

// Batching transform - groups items into batches
class BatchTransform extends Transform {
  private batch: any[] = [];
  private batchCount: number = 0;

  constructor(private batchSize: number, options?: any) {
    super({ ...options, objectMode: true });
  }

  _transform(chunk: any, encoding: string, callback: TransformCallback): void {
    this.batch.push(chunk);

    if (this.batch.length >= this.batchSize) {
      this.flushBatch();
    }

    callback();
  }

  _flush(callback: TransformCallback): void {
    // Flush remaining items
    if (this.batch.length > 0) {
      this.flushBatch();
    }
    callback();
  }

  private flushBatch(): void {
    this.batchCount++;
    this.push({
      batchNumber: this.batchCount,
      items: [...this.batch],
      count: this.batch.length,
      timestamp: new Date().toISOString()
    });
    this.batch = [];
  }
}

// Usage in banking: Process transactions in batches
async function processTransactionsInBatches() {
  const batchProcessor = new BatchTransform(1000); // 1000 transactions per batch
  
  const processor = new Transform({
    objectMode: true,
    async transform(batch: any, encoding: string, callback: TransformCallback) {
      try {
        console.log(`Processing batch ${batch.batchNumber} with ${batch.count} transactions`);
        
        // Simulate batch processing
        await processBatchToDatabase(batch.items);
        
        this.push({
          ...batch,
          status: 'processed',
          processedAt: new Date().toISOString()
        });
        
        callback();
      } catch (error) {
        callback(error as Error);
      }
    }
  });

  await pipeline(
    createReadStream('large-transaction-file.jsonl'),
    batchProcessor,
    processor,
    createWriteStream('processed-batches.jsonl')
  );
}

async function processBatchToDatabase(transactions: any[]): Promise<void> {
  // Simulated batch database insert
  return new Promise(resolve => setTimeout(resolve, 100));
}
```

**5. Transform Stream with Error Handling:**

```typescript
import { Transform, TransformCallback } from 'stream';
import { createWriteStream } from 'fs';

class RobustTransactionTransform extends Transform {
  private errorStream: NodeJS.WritableStream;
  private successCount: number = 0;
  private errorCount: number = 0;

  constructor(errorLogPath: string, options?: any) {
    super({ ...options, objectMode: true });
    this.errorStream = createWriteStream(errorLogPath);
  }

  _transform(chunk: any, encoding: string, callback: TransformCallback): void {
    try {
      const transaction = this.parseTransaction(chunk);
      const processed = this.processTransaction(transaction);
      
      this.successCount++;
      this.push(processed);
      callback();
    } catch (error) {
      this.errorCount++;
      
      // Log error but continue processing
      this.errorStream.write(JSON.stringify({
        error: (error as Error).message,
        data: chunk.toString(),
        timestamp: new Date().toISOString()
      }) + '\n');
      
      // Don't pass error to callback - continue processing
      callback();
    }
  }

  _flush(callback: TransformCallback): void {
    console.log(`Processed: ${this.successCount}, Errors: ${this.errorCount}`);
    this.errorStream.end();
    callback();
  }

  private parseTransaction(chunk: any): Transaction {
    if (typeof chunk === 'string') {
      return JSON.parse(chunk);
    }
    return chunk;
  }

  private processTransaction(txn: Transaction): any {
    // Simulate processing
    if (!txn.amount || txn.amount < 0) {
      throw new Error('Invalid amount');
    }
    
    return {
      ...txn,
      processed: true,
      processedAt: new Date().toISOString()
    };
  }
}
```

**6. Transform Stream for CSV to JSON Conversion:**

```typescript
import { Transform, TransformCallback } from 'stream';

class CSVtoJSONTransform extends Transform {
  private headers: string[] = [];
  private isFirstLine: boolean = true;

  constructor(options?: any) {
    super({ ...options, objectMode: true });
  }

  _transform(chunk: Buffer, encoding: string, callback: TransformCallback): void {
    const lines = chunk.toString().split('\n');

    for (const line of lines) {
      if (!line.trim()) continue;

      if (this.isFirstLine) {
        // First line is headers
        this.headers = line.split(',').map(h => h.trim());
        this.isFirstLine = false;
        continue;
      }

      const values = line.split(',').map(v => v.trim());
      const obj: any = {};

      this.headers.forEach((header, index) => {
        obj[header] = values[index] || '';
      });

      this.push(JSON.stringify(obj) + '\n');
    }

    callback();
  }
}

// Usage: Convert CSV transaction export to JSON
async function convertTransactionCSV() {
  await pipeline(
    createReadStream('transactions.csv'),
    new CSVtoJSONTransform(),
    createWriteStream('transactions.json')
  );
}
```

**7. Performance-Optimized Transform with Object Pooling:**

```typescript
import { Transform, TransformCallback } from 'stream';

class PooledTransactionTransform extends Transform {
  private objectPool: any[] = [];
  private poolSize: number = 100;
  private processed: number = 0;

  constructor(options?: any) {
    super({ ...options, objectMode: true, highWaterMark: 1000 });
    this.initializePool();
  }

  private initializePool(): void {
    for (let i = 0; i < this.poolSize; i++) {
      this.objectPool.push({});
    }
  }

  _transform(chunk: any, encoding: string, callback: TransformCallback): void {
    // Get object from pool
    const result = this.objectPool.pop() || {};
    
    // Reset object
    Object.keys(result).forEach(key => delete result[key]);
    
    // Process transaction
    Object.assign(result, chunk, {
      processed: true,
      sequenceNumber: ++this.processed,
      timestamp: Date.now()
    });

    this.push(result);
    
    // Return object to pool after use
    setImmediate(() => {
      if (this.objectPool.length < this.poolSize) {
        this.objectPool.push(result);
      }
    });

    callback();
  }
}
```

**Key Takeaways for ENBD:**
- Transform streams are ideal for data processing pipelines
- Use `objectMode: true` for processing JavaScript objects
- Implement `_transform()` for per-chunk processing
- Implement `_flush()` for end-of-stream cleanup
- Handle errors gracefully to avoid stopping the pipeline
- Use caching in transforms to reduce database calls
- Batch processing improves throughput for bulk operations
- Object pooling reduces garbage collection overhead
- Always push data before calling callback
- Use async/await carefully in transform methods

---

### **Q28: What is stream.pipeline() and how does it improve error handling in Node.js streams?**

**Answer:**

`stream.pipeline()` is a module method that pipes between streams and forwards errors, automatically cleaning up and destroying all streams when an error occurs or when the pipeline completes. It's the recommended way to pipe multiple streams together.

**1. Why pipeline() is Better than pipe():**

```typescript
import { pipeline } from 'stream/promises';
import { createReadStream, createWriteStream } from 'fs';
import { createGzip } from 'zlib';

// ❌ Old way with .pipe() - Error handling is complex
createReadStream('input.txt')
  .on('error', handleError)
  .pipe(createGzip())
  .on('error', handleError)
  .pipe(createWriteStream('output.txt.gz'))
  .on('error', handleError)
  .on('finish', () => console.log('Done'));

// ✅ Better way with pipeline() - Single error handler
pipeline(
  createReadStream('input.txt'),
  createGzip(),
  createWriteStream('output.txt.gz')
)
  .then(() => console.log('Pipeline succeeded'))
  .catch(err => console.error('Pipeline failed:', err));
```

**2. Basic Pipeline Usage:**

```typescript
import { pipeline } from 'stream/promises';
import { Transform } from 'stream';

// Simple pipeline example
async function processFile() {
  try {
    await pipeline(
      createReadStream('transactions.csv'),
      new Transform({
        transform(chunk, encoding, callback) {
          // Process chunk
          const processed = chunk.toString().toUpperCase();
          callback(null, processed);
        }
      }),
      createWriteStream('transactions-upper.csv')
    );
    
    console.log('Pipeline completed successfully');
  } catch (err) {
    console.error('Pipeline failed:', err);
    // All streams are automatically destroyed
  }
}
```

**3. Banking Example - Transaction Processing Pipeline:**

```typescript
import { pipeline } from 'stream/promises';
import { createReadStream, createWriteStream } from 'fs';
import { Transform } from 'stream';
import { createGzip, createGunzip } from 'zlib';

interface Transaction {
  id: string;
  accountNumber: string;
  amount: number;
  type: 'debit' | 'credit';
  timestamp: string;
}

// Complete transaction processing pipeline
class TransactionPipeline {
  private stats = {
    processed: 0,
    valid: 0,
    invalid: 0,
    totalAmount: 0
  };

  // Step 1: Parse JSON lines
  createParser(): Transform {
    return new Transform({
      objectMode: true,
      transform(chunk: Buffer, encoding: string, callback) {
        const lines = chunk.toString().split('\n').filter(line => line.trim());
        
        for (const line of lines) {
          try {
            const transaction = JSON.parse(line);
            this.push(transaction);
          } catch (err) {
            // Skip invalid JSON but don't stop pipeline
            console.error('Parse error:', err);
          }
        }
        
        callback();
      }
    });
  }

  // Step 2: Validate transactions
  createValidator(): Transform {
    return new Transform({
      objectMode: true,
      transform: (transaction: Transaction, encoding: string, callback) => {
        const errors: string[] = [];

        if (!transaction.id) errors.push('Missing ID');
        if (!transaction.accountNumber) errors.push('Missing account number');
        if (typeof transaction.amount !== 'number' || transaction.amount <= 0) {
          errors.push('Invalid amount');
        }
        if (!['debit', 'credit'].includes(transaction.type)) {
          errors.push('Invalid type');
        }

        if (errors.length > 0) {
          this.stats.invalid++;
          // Log invalid but continue processing
          console.error(`Invalid transaction ${transaction.id}:`, errors);
          callback(); // Don't push invalid transactions
        } else {
          this.stats.valid++;
          this.stats.totalAmount += transaction.amount;
          callback(null, transaction);
        }
      }
    });
  }

  // Step 3: Enrich with customer data
  createEnricher(): Transform {
    const customerCache = new Map();

    return new Transform({
      objectMode: true,
      async transform(transaction: Transaction, encoding: string, callback) {
        try {
          // Check cache first
          let customerData = customerCache.get(transaction.accountNumber);
          
          if (!customerData) {
            // Simulate database lookup
            customerData = await this.fetchCustomerData(transaction.accountNumber);
            customerCache.set(transaction.accountNumber, customerData);
          }

          const enriched = {
            ...transaction,
            customerName: customerData.name,
            branch: customerData.branch,
            customerType: customerData.type
          };

          callback(null, enriched);
        } catch (err) {
          callback(err as Error);
        }
      },
      
      async fetchCustomerData(accountNumber: string) {
        // Simulated async database call
        await new Promise(resolve => setTimeout(resolve, 10));
        return {
          name: `Customer-${accountNumber}`,
          branch: 'Dubai Main',
          type: 'Premium'
        };
      }
    });
  }

  // Step 4: Apply business rules
  createBusinessRuleProcessor(): Transform {
    return new Transform({
      objectMode: true,
      transform(transaction: any, encoding: string, callback) {
        // Apply ENBD business rules
        
        // Flag large transactions
        if (transaction.amount > 50000) {
          transaction.requiresApproval = true;
          transaction.approvalLevel = 'MANAGER';
        }
        
        if (transaction.amount > 200000) {
          transaction.approvalLevel = 'DIRECTOR';
        }

        // Calculate fees
        if (transaction.type === 'debit') {
          transaction.fee = transaction.amount > 10000 ? 25 : 10;
          transaction.totalAmount = transaction.amount + transaction.fee;
        } else {
          transaction.fee = 0;
          transaction.totalAmount = transaction.amount;
        }

        // Risk assessment
        transaction.riskScore = this.calculateRisk(transaction);
        
        callback(null, transaction);
      },
      
      calculateRisk(transaction: any): string {
        if (transaction.amount > 100000) return 'HIGH';
        if (transaction.amount > 50000) return 'MEDIUM';
        return 'LOW';
      }
    });
  }

  // Step 5: Format output
  createFormatter(): Transform {
    return new Transform({
      objectMode: true,
      writableObjectMode: true,
      readableObjectMode: false,
      transform(transaction: any, encoding: string, callback) {
        const formatted = JSON.stringify(transaction) + '\n';
        callback(null, formatted);
      }
    });
  }

  // Execute the complete pipeline
  async process(inputFile: string, outputFile: string, compress: boolean = false) {
    try {
      const streams: any[] = [
        createReadStream(inputFile),
        this.createParser(),
        this.createValidator(),
        this.createEnricher(),
        this.createBusinessRuleProcessor(),
        this.createFormatter()
      ];

      if (compress) {
        streams.push(createGzip());
      }

      streams.push(createWriteStream(outputFile));

      await pipeline(...streams);

      console.log('Pipeline completed successfully!');
      console.log('Statistics:', this.stats);
      
      return this.stats;
    } catch (err) {
      console.error('Pipeline failed:', err);
      throw err;
    }
  }
}

// Usage
async function processBankTransactions() {
  const processor = new TransactionPipeline();
  
  await processor.process(
    'daily-transactions.jsonl',
    'processed-transactions.jsonl.gz',
    true // Enable compression
  );
}
```

**4. Pipeline with Error Recovery:**

```typescript
import { pipeline } from 'stream/promises';
import { PassThrough, Transform } from 'stream';

// Create a resilient pipeline that continues on errors
class ResilientPipeline {
  async processWithRetry(
    inputPath: string,
    outputPath: string,
    maxRetries: number = 3
  ) {
    let attempt = 0;
    
    while (attempt < maxRetries) {
      try {
        await pipeline(
          createReadStream(inputPath),
          this.createResilientProcessor(),
          createWriteStream(outputPath)
        );
        
        console.log('Pipeline succeeded on attempt', attempt + 1);
        return;
      } catch (err) {
        attempt++;
        console.error(`Pipeline failed (attempt ${attempt}):`, err);
        
        if (attempt >= maxRetries) {
          throw new Error(`Pipeline failed after ${maxRetries} attempts`);
        }
        
        // Wait before retry (exponential backoff)
        await new Promise(resolve => setTimeout(resolve, 1000 * Math.pow(2, attempt)));
      }
    }
  }

  createResilientProcessor(): Transform {
    let errorCount = 0;
    const MAX_ERRORS = 100;

    return new Transform({
      objectMode: true,
      transform(chunk: any, encoding: string, callback) {
        try {
          // Process chunk
          const processed = this.processChunk(chunk);
          callback(null, processed);
        } catch (err) {
          errorCount++;
          
          if (errorCount > MAX_ERRORS) {
            // Too many errors, fail the pipeline
            callback(err as Error);
          } else {
            // Log error but continue
            console.error('Processing error:', err);
            callback(); // Continue without this chunk
          }
        }
      },
      
      processChunk(chunk: any) {
        // Simulated processing
        if (Math.random() < 0.01) throw new Error('Random processing error');
        return chunk;
      }
    });
  }
}
```

**5. Pipeline with Progress Tracking:**

```typescript
import { Transform } from 'stream';
import { pipeline } from 'stream/promises';

class ProgressTracker extends Transform {
  private processed: number = 0;
  private startTime: number;
  private lastReport: number;

  constructor(
    private reportInterval: number = 10000, // Report every 10k items
    options?: any
  ) {
    super({ ...options, objectMode: true });
    this.startTime = Date.now();
    this.lastReport = Date.now();
  }

  _transform(chunk: any, encoding: string, callback: Function) {
    this.processed++;

    // Report progress
    if (this.processed % this.reportInterval === 0) {
      this.reportProgress();
    }

    callback(null, chunk);
  }

  _flush(callback: Function) {
    this.reportProgress();
    console.log('\nProcessing complete!');
    callback();
  }

  private reportProgress() {
    const now = Date.now();
    const elapsed = (now - this.startTime) / 1000;
    const rate = this.processed / elapsed;
    const interval = (now - this.lastReport) / 1000;
    
    console.log(`Processed: ${this.processed.toLocaleString()} | Rate: ${rate.toFixed(0)}/s | Elapsed: ${elapsed.toFixed(1)}s`);
    
    this.lastReport = now;
  }
}

// Usage in banking pipeline
async function processLargeTransactionFile() {
  await pipeline(
    createReadStream('large-transactions.jsonl'),
    new ProgressTracker(10000),
    new Transform({
      objectMode: true,
      transform(chunk, encoding, callback) {
        // Process transaction
        callback(null, chunk);
      }
    }),
    createWriteStream('processed.jsonl')
  );
}
```

**6. Pipeline with Multiple Outputs:**

```typescript
import { PassThrough, Transform, Writable } from 'stream';
import { pipeline } from 'stream/promises';

// Split stream to multiple destinations
class StreamSplitter extends Transform {
  private outputs: Writable[] = [];

  constructor(options?: any) {
    super({ ...options, objectMode: true });
  }

  addOutput(output: Writable) {
    this.outputs.push(output);
  }

  _transform(chunk: any, encoding: string, callback: Function) {
    // Write to all outputs
    this.outputs.forEach(output => {
      output.write(chunk);
    });
    
    // Also pass through to next stream
    callback(null, chunk);
  }

  _flush(callback: Function) {
    // Close all outputs
    this.outputs.forEach(output => output.end());
    callback();
  }
}

// Banking example: Split transactions by type
async function splitTransactionsByType() {
  const splitter = new StreamSplitter();
  
  // Create separate output streams
  const debitStream = createWriteStream('debit-transactions.jsonl');
  const creditStream = createWriteStream('credit-transactions.jsonl');
  const largeStream = createWriteStream('large-transactions.jsonl');

  const router = new Transform({
    objectMode: true,
    transform(transaction: any, encoding: string, callback) {
      const txn = typeof transaction === 'string' 
        ? JSON.parse(transaction) 
        : transaction;

      // Route based on transaction type
      if (txn.type === 'debit') {
        debitStream.write(JSON.stringify(txn) + '\n');
      } else if (txn.type === 'credit') {
        creditStream.write(JSON.stringify(txn) + '\n');
      }

      // Also route large transactions
      if (txn.amount > 50000) {
        largeStream.write(JSON.stringify(txn) + '\n');
      }

      callback(null, transaction);
    },
    flush(callback) {
      debitStream.end();
      creditStream.end();
      largeStream.end();
      callback();
    }
  });

  await pipeline(
    createReadStream('all-transactions.jsonl'),
    router,
    createWriteStream('all-processed.jsonl')
  );
}
```

**7. Pipeline with Compression and Encryption:**

```typescript
import { pipeline } from 'stream/promises';
import { createGzip, createGunzip } from 'zlib';
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';

class SecureTransactionPipeline {
  private algorithm = 'aes-256-cbc';
  private key: Buffer;

  constructor(encryptionKey: string) {
    // Derive 32-byte key from password
    this.key = Buffer.from(encryptionKey.padEnd(32, '0').slice(0, 32));
  }

  // Encrypt and compress
  async secure(inputPath: string, outputPath: string) {
    const iv = randomBytes(16);
    
    // Write IV to the beginning of file
    const outputStream = createWriteStream(outputPath);
    outputStream.write(iv);

    const cipher = createCipheriv(this.algorithm, this.key, iv);

    await pipeline(
      createReadStream(inputPath),
      createGzip(), // Compress first
      cipher,       // Then encrypt
      outputStream
    );

    console.log('File secured successfully');
  }

  // Decrypt and decompress
  async unsecure(inputPath: string, outputPath: string) {
    const inputStream = createReadStream(inputPath);
    
    // Read IV from file
    const iv = await this.readIV(inputStream);
    const decipher = createDecipheriv(this.algorithm, this.key, iv);

    await pipeline(
      inputStream,
      decipher,         // Decrypt first
      createGunzip(),   // Then decompress
      createWriteStream(outputPath)
    );

    console.log('File unsecured successfully');
  }

  private async readIV(stream: any): Promise<Buffer> {
    return new Promise((resolve, reject) => {
      stream.once('readable', () => {
        const iv = stream.read(16);
        if (iv) {
          resolve(iv);
        } else {
          reject(new Error('Could not read IV'));
        }
      });
    });
  }
}

// Usage for banking data
async function secureTransactionFile() {
  const pipeline = new SecureTransactionPipeline('enbd-secret-key-2024');
  
  // Encrypt and compress sensitive transaction data
  await pipeline.secure(
    'sensitive-transactions.csv',
    'sensitive-transactions.enc'
  );
  
  // Later, decrypt and decompress
  await pipeline.unsecure(
    'sensitive-transactions.enc',
    'sensitive-transactions-decrypted.csv'
  );
}
```

**Key Takeaways for ENBD:**
- Use `pipeline()` instead of `pipe()` for automatic error handling
- Pipeline destroys all streams on error or completion
- Single try-catch block handles all stream errors
- Supports both callback and promise-based patterns
- Streams are automatically cleaned up (no memory leaks)
- Can chain any number of transform streams
- Perfect for ETL (Extract, Transform, Load) operations
- Use `stream/promises` for async/await syntax
- Pipeline preserves backpressure through all streams
- Ideal for processing large transaction files securely

---

### **Q29: What's the difference between flowing mode and paused mode in Node.js streams?**

**Answer:**

Node.js Readable streams operate in two modes: **flowing mode** (data flows automatically) and **paused mode** (data must be explicitly read). Understanding these modes is crucial for controlling data flow and managing backpressure.

**1. Flowing Mode vs Paused Mode:**

```typescript
import { Readable } from 'stream';

// Flowing Mode - Data flows automatically
const flowingStream = new Readable({
  read() {
    this.push('data chunk');
    this.push(null); // End stream
  }
});

// Method 1: Using 'data' event (switches to flowing mode)
flowingStream.on('data', (chunk) => {
  console.log('Flowing mode:', chunk.toString());
});

// Paused Mode - Must explicitly read data
const pausedStream = new Readable({
  read() {
    this.push('data chunk');
    this.push(null);
  }
});

// Method 2: Using read() method (paused mode)
pausedStream.on('readable', () => {
  let chunk;
  while ((chunk = pausedStream.read()) !== null) {
    console.log('Paused mode:', chunk.toString());
  }
});
```

**2. Banking Example - Transaction Stream with Mode Control:**

```typescript
import { Readable } from 'stream';
import { EventEmitter } from 'events';

interface Transaction {
  id: string;
  accountNumber: string;
  amount: number;
  type: 'debit' | 'credit';
  timestamp: Date;
}

class TransactionStream extends Readable {
  private transactions: Transaction[] = [];
  private currentIndex: number = 0;
  private isPaused: boolean = false;

  constructor(transactions: Transaction[], options?: any) {
    super({ ...options, objectMode: true });
    this.transactions = transactions;
  }

  _read(size: number) {
    // Called automatically in flowing mode
    // Called when read() is called in paused mode
    
    if (this.currentIndex >= this.transactions.length) {
      this.push(null); // End stream
      return;
    }

    // Push transactions in batches
    let count = 0;
    while (count < size && this.currentIndex < this.transactions.length) {
      const transaction = this.transactions[this.currentIndex++];
      
      // Stop pushing if buffer is full (returns false)
      if (!this.push(transaction)) {
        break;
      }
      
      count++;
    }
  }
}

// Demo: Flowing Mode
async function demoFlowingMode() {
  console.log('=== FLOWING MODE ===\n');
  
  const transactions: Transaction[] = Array.from({ length: 10 }, (_, i) => ({
    id: `TXN-${i + 1}`,
    accountNumber: '100000000001',
    amount: 100 * (i + 1),
    type: i % 2 === 0 ? 'debit' : 'credit',
    timestamp: new Date()
  }));

  const stream = new TransactionStream(transactions);
  let count = 0;

  // 'data' event automatically switches to flowing mode
  stream.on('data', (transaction: Transaction) => {
    count++;
    console.log(`${count}. Received (flowing): ${transaction.id} - $${transaction.amount}`);
  });

  stream.on('end', () => {
    console.log('\nFlowing mode completed');
  });
}

// Demo: Paused Mode
async function demoPausedMode() {
  console.log('\n=== PAUSED MODE ===\n');
  
  const transactions: Transaction[] = Array.from({ length: 10 }, (_, i) => ({
    id: `TXN-${i + 1}`,
    accountNumber: '100000000001',
    amount: 100 * (i + 1),
    type: i % 2 === 0 ? 'debit' : 'credit',
    timestamp: new Date()
  }));

  const stream = new TransactionStream(transactions);
  let count = 0;

  // 'readable' event keeps stream in paused mode
  stream.on('readable', () => {
    let transaction;
    
    // Manually read all available data
    while ((transaction = stream.read()) !== null) {
      count++;
      console.log(`${count}. Received (paused): ${transaction.id} - $${transaction.amount}`);
    }
  });

  stream.on('end', () => {
    console.log('\nPaused mode completed');
  });
}
```

**3. Controlling Stream Mode:**

```typescript
import { Readable } from 'stream';

class ControlledTransactionStream extends Readable {
  private data: any[] = [];
  private index: number = 0;

  constructor(data: any[]) {
    super({ objectMode: true });
    this.data = data;
  }

  _read() {
    if (this.index < this.data.length) {
      this.push(this.data[this.index++]);
    } else {
      this.push(null);
    }
  }
}

// Demo mode switching
async function demoModeSwitching() {
  const transactions = Array.from({ length: 20 }, (_, i) => ({
    id: `TXN-${i + 1}`,
    amount: 100 * (i + 1)
  }));

  const stream = new ControlledTransactionStream(transactions);
  let count = 0;

  // Start in paused mode with 'readable'
  stream.on('readable', () => {
    console.log('readable event fired');
  });

  // Switch to flowing mode with 'data' event
  setTimeout(() => {
    console.log('\nSwitching to flowing mode...\n');
    
    stream.on('data', (txn) => {
      count++;
      console.log(`${count}. Flowing: ${txn.id}`);
      
      // Manually pause after 5 items
      if (count === 5) {
        console.log('\nPausing stream...\n');
        stream.pause();
        
        // Resume after 2 seconds
        setTimeout(() => {
          console.log('Resuming stream...\n');
          stream.resume();
        }, 2000);
      }
    });
  }, 1000);

  stream.on('end', () => {
    console.log('\nStream ended');
  });
}
```

**4. Practical Banking Example - Rate-Limited Transaction Processor:**

```typescript
import { Readable, Writable } from 'stream';

class RateLimitedTransactionProcessor {
  private processedCount: number = 0;
  private maxPerSecond: number;
  private lastResetTime: number;
  private stream: Readable;

  constructor(stream: Readable, maxPerSecond: number = 100) {
    this.stream = stream;
    this.maxPerSecond = maxPerSecond;
    this.lastResetTime = Date.now();
  }

  async process() {
    // Start in paused mode for control
    this.stream.pause();

    this.stream.on('readable', () => {
      this.processAvailable();
    });

    this.stream.on('end', () => {
      console.log(`\nTotal processed: ${this.processedCount}`);
    });

    // Start processing
    this.stream.resume();
  }

  private processAvailable() {
    const now = Date.now();
    
    // Reset counter every second
    if (now - this.lastResetTime >= 1000) {
      this.processedCount = 0;
      this.lastResetTime = now;
    }

    // Process up to rate limit
    let transaction;
    while (this.processedCount < this.maxPerSecond) {
      transaction = this.stream.read();
      
      if (transaction === null) {
        // No more data available
        break;
      }

      this.processTransaction(transaction);
      this.processedCount++;
    }

    // If rate limit reached, pause and resume after delay
    if (this.processedCount >= this.maxPerSecond) {
      console.log(`Rate limit reached (${this.maxPerSecond}/s), pausing...`);
      this.stream.pause();
      
      setTimeout(() => {
        this.processedCount = 0;
        this.lastResetTime = Date.now();
        this.stream.resume();
      }, 1000);
    }
  }

  private processTransaction(transaction: any) {
    // Simulate processing
    console.log(`Processing: ${transaction.id} - $${transaction.amount}`);
  }
}

// Usage
async function processWithRateLimit() {
  const transactions = Array.from({ length: 500 }, (_, i) => ({
    id: `TXN-${i + 1}`,
    amount: Math.random() * 1000
  }));

  const stream = new ControlledTransactionStream(transactions);
  const processor = new RateLimitedTransactionProcessor(stream, 50); // 50 per second
  
  await processor.process();
}
```

**5. Object Mode vs Binary Mode:**

```typescript
import { Readable } from 'stream';

// Binary mode (default) - works with Buffers/strings
class BinaryTransactionStream extends Readable {
  private sent: boolean = false;

  _read() {
    if (!this.sent) {
      // Push buffer data
      this.push(Buffer.from('Transaction data in binary'));
      this.sent = true;
    } else {
      this.push(null);
    }
  }
}

// Object mode - works with JavaScript objects
class ObjectTransactionStream extends Readable {
  private transactions: any[];
  private index: number = 0;

  constructor(transactions: any[]) {
    super({ objectMode: true }); // Enable object mode
    this.transactions = transactions;
  }

  _read() {
    if (this.index < this.transactions.length) {
      // Push JavaScript objects directly
      this.push(this.transactions[this.index++]);
    } else {
      this.push(null);
    }
  }
}

// Demo both modes
async function demoBinaryVsObjectMode() {
  console.log('=== BINARY MODE ===');
  const binaryStream = new BinaryTransactionStream();
  
  binaryStream.on('data', (chunk) => {
    console.log('Type:', typeof chunk); // Buffer
    console.log('Data:', chunk.toString());
  });

  await new Promise(resolve => binaryStream.on('end', resolve));

  console.log('\n=== OBJECT MODE ===');
  const transactions = [
    { id: 'TXN-1', amount: 100 },
    { id: 'TXN-2', amount: 200 }
  ];
  
  const objectStream = new ObjectTransactionStream(transactions);
  
  objectStream.on('data', (transaction) => {
    console.log('Type:', typeof transaction); // object
    console.log('Data:', transaction);
  });
}
```

**6. Backpressure Handling with Mode Control:**

```typescript
import { Readable, Writable } from 'stream';
import { pipeline } from 'stream/promises';

class SlowProcessor extends Writable {
  private processed: number = 0;

  constructor(options?: any) {
    super({ ...options, objectMode: true });
  }

  async _write(chunk: any, encoding: string, callback: Function) {
    // Simulate slow processing
    await new Promise(resolve => setTimeout(resolve, 100));
    
    this.processed++;
    console.log(`Processed ${this.processed}: ${chunk.id}`);
    
    callback();
  }
}

class FastProducer extends Readable {
  private count: number = 0;
  private maxCount: number;

  constructor(maxCount: number = 100) {
    super({ objectMode: true });
    this.maxCount = maxCount;
  }

  _read() {
    if (this.count < this.maxCount) {
      const canPush = this.push({
        id: `TXN-${++this.count}`,
        amount: Math.random() * 1000
      });

      // If buffer is full, stop producing
      if (!canPush) {
        console.log('Buffer full, pausing production...');
      }
    } else {
      this.push(null);
    }
  }
}

// Demo backpressure
async function demoBackpressure() {
  console.log('=== BACKPRESSURE DEMO ===\n');
  
  const producer = new FastProducer(50);
  const processor = new SlowProcessor();

  // Pipeline automatically handles backpressure
  await pipeline(producer, processor);
  
  console.log('\nBackpressure demo completed');
}
```

**7. Manual Stream Control:**

```typescript
class ManualTransactionStream extends Readable {
  private data: any[] = [];
  private index: number = 0;

  constructor(data: any[]) {
    super({ objectMode: true });
    this.data = data;
  }

  _read() {
    if (this.index < this.data.length) {
      this.push(this.data[this.index++]);
    } else {
      this.push(null);
    }
  }
}

// Manual control example
async function manualStreamControl() {
  const transactions = Array.from({ length: 20 }, (_, i) => ({
    id: `TXN-${i + 1}`,
    amount: 100 * (i + 1)
  }));

  const stream = new ManualTransactionStream(transactions);
  
  console.log('=== MANUAL CONTROL ===\n');
  console.log('Stream is paused by default');
  
  // Manually read one chunk at a time
  for (let i = 0; i < 5; i++) {
    stream.resume(); // Switch to flowing mode briefly
    
    await new Promise(resolve => {
      stream.once('data', (transaction) => {
        console.log(`Read: ${transaction.id} - $${transaction.amount}`);
        stream.pause(); // Pause after each read
        resolve(null);
      });
    });
    
    await new Promise(resolve => setTimeout(resolve, 500)); // Delay between reads
  }
  
  console.log('\nReading remaining in bulk...');
  
  // Read rest in flowing mode
  stream.on('data', (transaction) => {
    console.log(`Bulk read: ${transaction.id}`);
  });
  
  stream.resume();
  
  await new Promise(resolve => stream.on('end', resolve));
}
```

**Key Takeaways for ENBD:**
- **Flowing mode**: Data flows automatically, triggered by `data` event
- **Paused mode**: Data must be explicitly read with `read()` method
- Use `pause()` and `resume()` to control flow
- `readable` event keeps stream in paused mode
- `data` event switches stream to flowing mode
- Paused mode gives fine-grained control for rate limiting
- Flowing mode is simpler for straightforward processing
- `objectMode: true` allows streaming JavaScript objects
- Backpressure is automatic in paused mode
- Use paused mode for complex control logic (throttling, batching)

---

### **Q30: How do you implement custom Readable, Writable, and Duplex streams in Node.js?**

**Answer:**

Custom streams allow you to create specialized data sources and sinks. You implement them by extending the base stream classes and overriding specific methods.

**1. Custom Readable Stream - Database Query Stream:**

```typescript
import { Readable } from 'stream';

interface TransactionRecord {
  id: string;
  accountNumber: string;
  amount: number;
  date: Date;
}

class DatabaseQueryStream extends Readable {
  private offset: number = 0;
  private batchSize: number;
  private tableName: string;
  private dbClient: any;
  private endReached: boolean = false;

  constructor(dbClient: any, tableName: string, batchSize: number = 100) {
    super({ objectMode: true, highWaterMark: 10 });
    this.dbClient = dbClient;
    this.tableName = tableName;
    this.batchSize = batchSize;
  }

  async _read() {
    if (this.endReached) {
      return;
    }

    try {
      // Fetch batch from database
      const query = `SELECT * FROM ${this.tableName} LIMIT ${this.batchSize} OFFSET ${this.offset}`;
      const results = await this.dbClient.query(query);

      if (results.length === 0) {
        // No more data
        this.push(null);
        this.endReached = true;
        return;
      }

      // Push each record
      for (const record of results) {
        if (!this.push(record)) {
          // Buffer is full, stop pushing
          break;
        }
      }

      this.offset += results.length;
    } catch (error) {
      this.destroy(error as Error);
    }
  }

  _destroy(error: Error | null, callback: Function) {
    // Cleanup database connection
    if (this.dbClient) {
      this.dbClient.close();
    }
    callback(error);
  }
}

// Usage
async function streamFromDatabase() {
  const dbClient = {
    query: async (sql: string) => {
      // Simulated database query
      return Array.from({ length: 100 }, (_, i) => ({
        id: `TXN-${i + 1}`,
        accountNumber: '100000000001',
        amount: Math.random() * 1000,
        date: new Date()
      }));
    },
    close: () => console.log('DB connection closed')
  };

  const stream = new DatabaseQueryStream(dbClient, 'transactions', 50);
  
  let count = 0;
  stream.on('data', (record) => {
    count++;
    console.log(`Record ${count}: ${record.id} - $${record.amount.toFixed(2)}`);
  });

  stream.on('end', () => {
    console.log(`\nStreamed ${count} records from database`);
  });

  stream.on('error', (err) => {
    console.error('Stream error:', err);
  });
}
```

**2. Custom Writable Stream - Batch Database Writer:**

```typescript
import { Writable } from 'stream';

class BatchDatabaseWriter extends Writable {
  private batch: any[] = [];
  private batchSize: number;
  private dbClient: any;
  private tableName: string;
  private writtenCount: number = 0;

  constructor(dbClient: any, tableName: string, batchSize: number = 1000) {
    super({ objectMode: true, highWaterMark: 100 });
    this.dbClient = dbClient;
    this.tableName = tableName;
    this.batchSize = batchSize;
  }

  async _write(chunk: any, encoding: string, callback: Function) {
    this.batch.push(chunk);

    if (this.batch.length >= this.batchSize) {
      try {
        await this.flushBatch();
        callback();
      } catch (error) {
        callback(error);
      }
    } else {
      callback();
    }
  }

  async _writev(chunks: Array<{ chunk: any; encoding: string }>, callback: Function) {
    // Optimized for multiple writes
    for (const { chunk } of chunks) {
      this.batch.push(chunk);
    }

    if (this.batch.length >= this.batchSize) {
      try {
        await this.flushBatch();
      } catch (error) {
        return callback(error);
      }
    }

    callback();
  }

  async _final(callback: Function) {
    // Flush remaining data when stream ends
    try {
      if (this.batch.length > 0) {
        await this.flushBatch();
      }
      console.log(`\nTotal written: ${this.writtenCount} records`);
      callback();
    } catch (error) {
      callback(error);
    }
  }

  private async flushBatch() {
    const batchToWrite = [...this.batch];
    this.batch = [];

    // Simulated batch insert
    await this.dbClient.batchInsert(this.tableName, batchToWrite);
    
    this.writtenCount += batchToWrite.length;
    console.log(`Wrote batch of ${batchToWrite.length} records (total: ${this.writtenCount})`);
  }

  _destroy(error: Error | null, callback: Function) {
    // Cleanup
    if (this.dbClient) {
      this.dbClient.close();
    }
    callback(error);
  }
}

// Usage
async function writeToDatabaseInBatches() {
  const dbClient = {
    batchInsert: async (table: string, records: any[]) => {
      // Simulated batch insert
      await new Promise(resolve => setTimeout(resolve, 100));
    },
    close: () => console.log('DB connection closed')
  };

  const writer = new BatchDatabaseWriter(dbClient, 'transactions', 500);

  // Write 5000 transactions
  for (let i = 0; i < 5000; i++) {
    const canContinue = writer.write({
      id: `TXN-${i + 1}`,
      accountNumber: '100000000001',
      amount: Math.random() * 1000,
      date: new Date()
    });

    if (!canContinue) {
      // Wait for drain event if buffer is full
      await new Promise(resolve => writer.once('drain', resolve));
    }
  }

  writer.end();
  await new Promise(resolve => writer.once('finish', resolve));
}
```

**3. Custom Duplex Stream - Transaction Validator:**

```typescript
import { Duplex } from 'stream';

interface ValidationResult {
  original: any;
  isValid: boolean;
  errors: string[];
  validatedAt: Date;
}

class TransactionValidatorStream extends Duplex {
  private validCount: number = 0;
  private invalidCount: number = 0;
  private queue: any[] = [];

  constructor(options?: any) {
    super({ ...options, objectMode: true });
  }

  // Writable side - receives transactions to validate
  async _write(chunk: any, encoding: string, callback: Function) {
    try {
      const result = await this.validateTransaction(chunk);
      
      if (result.isValid) {
        this.validCount++;
      } else {
        this.invalidCount++;
      }

      // Add to internal queue for reading
      this.queue.push(result);
      
      callback();
    } catch (error) {
      callback(error);
    }
  }

  // Readable side - provides validation results
  _read() {
    if (this.queue.length > 0) {
      const result = this.queue.shift();
      this.push(result);
    }
  }

  _final(callback: Function) {
    console.log(`\nValidation Summary:`);
    console.log(`  Valid: ${this.validCount}`);
    console.log(`  Invalid: ${this.invalidCount}`);
    
    // Push remaining items
    while (this.queue.length > 0) {
      this.push(this.queue.shift());
    }
    
    callback();
  }

  private async validateTransaction(transaction: any): Promise<ValidationResult> {
    const errors: string[] = [];

    // Validation rules for ENBD
    if (!transaction.id) {
      errors.push('Transaction ID is required');
    }

    if (!transaction.accountNumber || !/^\d{12}$/.test(transaction.accountNumber)) {
      errors.push('Valid 12-digit account number is required');
    }

    if (typeof transaction.amount !== 'number' || transaction.amount <= 0) {
      errors.push('Amount must be a positive number');
    }

    if (transaction.amount > 1000000) {
      errors.push('Amount exceeds maximum limit (1,000,000 AED)');
    }

    if (!['debit', 'credit'].includes(transaction.type)) {
      errors.push('Type must be debit or credit');
    }

    // Simulate async validation (e.g., check account exists)
    await new Promise(resolve => setTimeout(resolve, 1));

    return {
      original: transaction,
      isValid: errors.length === 0,
      errors,
      validatedAt: new Date()
    };
  }
}

// Usage
async function validateTransactions() {
  const validator = new TransactionValidatorStream();

  // Write invalid transactions
  validator.write({ id: 'TXN-1', amount: 100, type: 'debit' }); // Missing accountNumber
  validator.write({ id: 'TXN-2', accountNumber: '100000000001', amount: -50, type: 'debit' }); // Negative
  validator.write({ id: 'TXN-3', accountNumber: '100000000001', amount: 500, type: 'debit' }); // Valid
  validator.write({ id: 'TXN-4', accountNumber: '100000000001', amount: 2000000, type: 'credit' }); // Too large

  validator.end();

  // Read validation results
  validator.on('data', (result: ValidationResult) => {
    console.log(`\n${result.original.id}:`);
    console.log(`  Valid: ${result.isValid}`);
    if (!result.isValid) {
      console.log(`  Errors:`, result.errors);
    }
  });

  await new Promise(resolve => validator.on('end', resolve));
}
```

**4. Custom Transform Stream - Data Aggregator:**

```typescript
import { Transform } from 'stream';

interface AggregatedData {
  accountNumber: string;
  totalDebit: number;
  totalCredit: number;
  transactionCount: number;
  avgTransactionAmount: number;
}

class TransactionAggregatorStream extends Transform {
  private aggregates: Map<string, AggregatedData> = new Map();
  private flushInterval: number;
  private timer: NodeJS.Timeout | null = null;

  constructor(flushIntervalMs: number = 5000, options?: any) {
    super({ ...options, objectMode: true });
    this.flushInterval = flushIntervalMs;
    this.startTimer();
  }

  _transform(chunk: any, encoding: string, callback: Function) {
    const accountNumber = chunk.accountNumber;
    
    let agg = this.aggregates.get(accountNumber);
    
    if (!agg) {
      agg = {
        accountNumber,
        totalDebit: 0,
        totalCredit: 0,
        transactionCount: 0,
        avgTransactionAmount: 0
      };
      this.aggregates.set(accountNumber, agg);
    }

    // Update aggregates
    if (chunk.type === 'debit') {
      agg.totalDebit += chunk.amount;
    } else {
      agg.totalCredit += chunk.amount;
    }
    
    agg.transactionCount++;
    agg.avgTransactionAmount = (agg.totalDebit + agg.totalCredit) / agg.transactionCount;

    callback();
  }

  _flush(callback: Function) {
    this.stopTimer();
    this.flushAggregates();
    callback();
  }

  private startTimer() {
    this.timer = setInterval(() => {
      this.flushAggregates();
    }, this.flushInterval);
  }

  private stopTimer() {
    if (this.timer) {
      clearInterval(this.timer);
      this.timer = null;
    }
  }

  private flushAggregates() {
    for (const [accountNumber, agg] of this.aggregates) {
      this.push({ ...agg, flushedAt: new Date() });
    }
    
    // Clear aggregates after flushing
    this.aggregates.clear();
  }

  _destroy(error: Error | null, callback: Function) {
    this.stopTimer();
    callback(error);
  }
}

// Usage
async function aggregateTransactions() {
  const aggregator = new TransactionAggregatorStream(2000); // Flush every 2 seconds

  // Send transactions for multiple accounts
  const accounts = ['100000000001', '100000000002', '100000000003'];
  
  for (let i = 0; i < 100; i++) {
    const account = accounts[i % accounts.length];
    
    aggregator.write({
      id: `TXN-${i + 1}`,
      accountNumber: account,
      amount: Math.random() * 1000,
      type: i % 2 === 0 ? 'debit' : 'credit'
    });

    await new Promise(resolve => setTimeout(resolve, 100));
  }

  aggregator.end();

  // Read aggregated results
  aggregator.on('data', (agg: AggregatedData) => {
    console.log(`\nAccount: ${agg.accountNumber}`);
    console.log(`  Transactions: ${agg.transactionCount}`);
    console.log(`  Total Debit: $${agg.totalDebit.toFixed(2)}`);
    console.log(`  Total Credit: $${agg.totalCredit.toFixed(2)}`);
    console.log(`  Average: $${agg.avgTransactionAmount.toFixed(2)}`);
  });

  await new Promise(resolve => aggregator.on('end', resolve));
}
```

**5. Real-World Example - File Processing Pipeline:**

```typescript
import { Readable, Writable, Transform, pipeline } from 'stream';
import { createReadStream, createWriteStream } from 'fs';
import { createGzip } from 'zlib';

// Custom Readable: Generate mock transactions
class TransactionGenerator extends Readable {
  private count: number = 0;
  private maxCount: number;

  constructor(maxCount: number) {
    super({ objectMode: true });
    this.maxCount = maxCount;
  }

  _read() {
    if (this.count < this.maxCount) {
      this.push({
        id: `TXN-${++this.count}`,
        accountNumber: `10000000000${Math.floor(Math.random() * 9) + 1}`,
        amount: Math.random() * 10000,
        type: Math.random() > 0.5 ? 'debit' : 'credit',
        timestamp: new Date().toISOString()
      });
    } else {
      this.push(null);
    }
  }
}

// Custom Transform: Format to CSV
class CSVFormatter extends Transform {
  private headerWritten: boolean = false;

  constructor() {
    super({ objectMode: true });
  }

  _transform(chunk: any, encoding: string, callback: Function) {
    if (!this.headerWritten) {
      this.push('id,accountNumber,amount,type,timestamp\n');
      this.headerWritten = true;
    }

    const csv = `${chunk.id},${chunk.accountNumber},${chunk.amount.toFixed(2)},${chunk.type},${chunk.timestamp}\n`;
    callback(null, csv);
  }
}

// Custom Writable: Write with statistics
class StatisticsWriter extends Writable {
  private writeStream: NodeJS.WritableStream;
  private bytesWritten: number = 0;
  private linesWritten: number = 0;

  constructor(filePath: string) {
    super();
    this.writeStream = createWriteStream(filePath);
  }

  _write(chunk: Buffer, encoding: string, callback: Function) {
    this.bytesWritten += chunk.length;
    this.linesWritten++;

    this.writeStream.write(chunk, callback);
  }

  _final(callback: Function) {
    console.log(`\nWrite Statistics:`);
    console.log(`  Lines: ${this.linesWritten}`);
    console.log(`  Bytes: ${this.bytesWritten.toLocaleString()}`);
    console.log(`  Size: ${(this.bytesWritten / 1024).toFixed(2)} KB`);
    
    this.writeStream.end(callback);
  }
}

// Complete pipeline
async function runCompleteCustomPipeline() {
  console.log('Starting custom stream pipeline...\n');

  const generator = new TransactionGenerator(10000);
  const formatter = new CSVFormatter();
  const writer = new StatisticsWriter('transactions.csv');

  await pipeline(
    generator,
    formatter,
    writer
  );

  console.log('\nPipeline completed!');
}
```

**6. Custom Stream with Advanced Features:**

```typescript
import { Duplex } from 'stream';

class SmartCacheStream extends Duplex {
  private cache: Map<string, any> = new Map();
  private cacheHits: number = 0;
  private cacheMisses: number = 0;
  private maxCacheSize: number;

  constructor(maxCacheSize: number = 1000, options?: any) {
    super({ ...options, objectMode: true });
    this.maxCacheSize = maxCacheSize;
  }

  _write(chunk: any, encoding: string, callback: Function) {
    // Cache the data
    if (this.cache.size < this.maxCacheSize) {
      this.cache.set(chunk.id, chunk);
    }
    
    // Pass through
    this.push(chunk);
    callback();
  }

  _read() {
    // Implement custom read logic if needed
  }

  // Custom method to query cache
  async query(id: string): Promise<any | null> {
    if (this.cache.has(id)) {
      this.cacheHits++;
      return this.cache.get(id);
    } else {
      this.cacheMisses++;
      return null;
    }
  }

  getCacheStats() {
    return {
      size: this.cache.size,
      hits: this.cacheHits,
      misses: this.cacheMisses,
      hitRate: this.cacheHits / (this.cacheHits + this.cacheMisses)
    };
  }
}
```

**Key Takeaways for ENBD:**
- **Readable**: Override `_read()` to produce data (database queries, file generation)
- **Writable**: Override `_write()` to consume data (batch inserts, file writes)
- **Duplex**: Override both `_read()` and `_write()` for two-way streams
- **Transform**: Override `_transform()` for data modification (extends Duplex)
- Use `_final()` in Writable/Transform for cleanup before stream ends
- Use `_destroy()` for cleanup on errors or explicit destruction
- Set `objectMode: true` to stream JavaScript objects
- Control `highWaterMark` for buffer size tuning
- Implement `_writev()` for optimized batch writes
- Return `false` from `push()` to respect backpressure
- Custom streams enable specialized data processing pipelines

---

## Section 4: Performance Optimization

### Questions 31-40

### **Q31: How do you use the cluster module to scale Node.js applications across multiple CPU cores?**

**Answer:**

The cluster module allows you to create child processes (workers) that share the same server port, enabling you to utilize all CPU cores and handle more concurrent requests. This is essential for production Node.js applications.

**1. Basic Cluster Setup:**

```typescript
import cluster from 'cluster';
import { cpus } from 'os';
import http from 'http';

if (cluster.isPrimary) {
  // Primary process - fork workers
  const numCPUs = cpus().length;
  console.log(`Primary ${process.pid} is running`);
  console.log(`Forking ${numCPUs} workers...`);

  // Fork workers
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  // Listen for worker events
  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died (${signal || code})`);
    console.log('Starting a new worker...');
    cluster.fork(); // Replace dead worker
  });

} else {
  // Worker process - create server
  const server = http.createServer((req, res) => {
    res.writeHead(200);
    res.end(`Handled by worker ${process.pid}\n`);
  });

  server.listen(3000, () => {
    console.log(`Worker ${process.pid} started`);
  });
}
```

**2. Banking API with Cluster Module:**

```typescript
import cluster from 'cluster';
import { cpus } from 'os';
import express, { Request, Response } from 'express';
import { createServer } from 'http';

interface Transaction {
  id: string;
  accountNumber: string;
  amount: number;
  type: 'debit' | 'credit';
  timestamp: Date;
}

class ClusteredBankingAPI {
  private app: express.Application;
  private transactions: Map<string, Transaction> = new Map();
  private requestCount: number = 0;

  constructor() {
    this.app = express();
    this.app.use(express.json());
    this.setupRoutes();
    this.setupHealthCheck();
  }

  private setupRoutes() {
    // Transaction processing endpoint
    this.app.post('/api/transactions', (req: Request, res: Response) => {
      try {
        const transaction: Transaction = {
          id: `TXN-${Date.now()}-${Math.random()}`,
          ...req.body,
          timestamp: new Date()
        };

        // Simulate processing
        this.processTransaction(transaction);
        
        this.requestCount++;
        
        res.json({
          success: true,
          transaction,
          processedBy: process.pid,
          workerRequests: this.requestCount
        });
      } catch (error) {
        res.status(500).json({
          success: false,
          error: (error as Error).message
        });
      }
    });

    // Get transaction
    this.app.get('/api/transactions/:id', (req: Request, res: Response) => {
      const transaction = this.transactions.get(req.params.id);
      
      if (transaction) {
        res.json({ transaction, processedBy: process.pid });
      } else {
        res.status(404).json({ error: 'Transaction not found' });
      }
    });

    // Account balance endpoint (CPU intensive)
    this.app.get('/api/accounts/:accountNumber/balance', (req: Request, res: Response) => {
      const balance = this.calculateBalance(req.params.accountNumber);
      
      res.json({
        accountNumber: req.params.accountNumber,
        balance,
        processedBy: process.pid,
        workerRequests: this.requestCount
      });
    });
  }

  private setupHealthCheck() {
    this.app.get('/health', (req: Request, res: Response) => {
      res.json({
        status: 'healthy',
        pid: process.pid,
        memory: process.memoryUsage(),
        uptime: process.uptime(),
        requests: this.requestCount
      });
    });
  }

  private processTransaction(transaction: Transaction) {
    // Simulate processing time
    const start = Date.now();
    while (Date.now() - start < 10) {} // 10ms processing
    
    this.transactions.set(transaction.id, transaction);
  }

  private calculateBalance(accountNumber: string): number {
    // Simulate CPU-intensive calculation
    let balance = 0;
    
    for (const [_, txn] of this.transactions) {
      if (txn.accountNumber === accountNumber) {
        balance += txn.type === 'credit' ? txn.amount : -txn.amount;
      }
    }
    
    return balance;
  }

  start(port: number) {
    this.app.listen(port, () => {
      console.log(`Worker ${process.pid} listening on port ${port}`);
    });
  }
}

// Cluster setup
if (cluster.isPrimary) {
  const numWorkers = cpus().length;
  console.log(`\n=== ENBD Banking API - Cluster Mode ===`);
  console.log(`Primary process ${process.pid} is running`);
  console.log(`CPU cores available: ${numWorkers}`);
  console.log(`Starting ${numWorkers} worker processes...\n`);

  const workers: Map<number, any> = new Map();

  // Fork workers
  for (let i = 0; i < numWorkers; i++) {
    const worker = cluster.fork({ WORKER_ID: i + 1 });
    workers.set(worker.id, {
      id: worker.id,
      pid: worker.process.pid,
      startTime: Date.now()
    });
  }

  // Worker online event
  cluster.on('online', (worker) => {
    console.log(`✅ Worker ${worker.process.pid} is online`);
  });

  // Worker exit event
  cluster.on('exit', (worker, code, signal) => {
    const workerInfo = workers.get(worker.id);
    const uptime = workerInfo ? (Date.now() - workerInfo.startTime) / 1000 : 0;
    
    console.log(`\n❌ Worker ${worker.process.pid} died`);
    console.log(`   Exit code: ${code}`);
    console.log(`   Signal: ${signal}`);
    console.log(`   Uptime: ${uptime.toFixed(2)}s`);
    
    workers.delete(worker.id);
    
    // Restart worker
    console.log(`   Starting replacement worker...`);
    const newWorker = cluster.fork({ WORKER_ID: workers.size + 1 });
    workers.set(newWorker.id, {
      id: newWorker.id,
      pid: newWorker.process.pid,
      startTime: Date.now()
    });
  });

  // Graceful shutdown
  process.on('SIGTERM', () => {
    console.log('\n📡 SIGTERM received, shutting down gracefully...');
    
    for (const id in cluster.workers) {
      cluster.workers[id]?.kill();
    }
  });

  // Display cluster status every 30 seconds
  setInterval(() => {
    console.log(`\n📊 Cluster Status:`);
    console.log(`   Active workers: ${Object.keys(cluster.workers || {}).length}`);
    console.log(`   Total workers spawned: ${workers.size}`);
  }, 30000);

} else {
  // Worker process
  const api = new ClusteredBankingAPI();
  api.start(3000);
  
  // Handle graceful shutdown
  process.on('SIGTERM', () => {
    console.log(`Worker ${process.pid} received SIGTERM, shutting down...`);
    process.exit(0);
  });
}
```

**3. Advanced Cluster with Load Balancing Strategy:**

```typescript
import cluster from 'cluster';
import { cpus } from 'os';

class ClusterManager {
  private workers: Map<number, WorkerInfo> = new Map();
  private roundRobinIndex: number = 0;

  interface WorkerInfo {
    id: number;
    pid: number;
    load: number; // Current load (0-100)
    requests: number;
    startTime: number;
  }

  startCluster(numWorkers?: number) {
    const workerCount = numWorkers || cpus().length;
    
    console.log(`Starting cluster with ${workerCount} workers`);

    for (let i = 0; i < workerCount; i++) {
      this.forkWorker();
    }

    this.setupEventListeners();
    this.startMonitoring();
  }

  private forkWorker() {
    const worker = cluster.fork();
    
    this.workers.set(worker.id, {
      id: worker.id,
      pid: worker.process.pid!,
      load: 0,
      requests: 0,
      startTime: Date.now()
    });

    // Listen to messages from worker
    worker.on('message', (msg) => {
      if (msg.type === 'metrics') {
        this.updateWorkerMetrics(worker.id, msg.data);
      }
    });
  }

  private setupEventListeners() {
    cluster.on('exit', (worker, code, signal) => {
      console.log(`Worker ${worker.process.pid} died`);
      this.workers.delete(worker.id);
      
      // Restart worker if it didn't exit cleanly
      if (code !== 0 && !worker.exitedAfterDisconnect) {
        console.log('Starting replacement worker...');
        this.forkWorker();
      }
    });
  }

  private updateWorkerMetrics(workerId: number, metrics: any) {
    const worker = this.workers.get(workerId);
    if (worker) {
      worker.load = metrics.load;
      worker.requests = metrics.requests;
    }
  }

  private startMonitoring() {
    setInterval(() => {
      console.log('\n📊 Worker Status:');
      
      for (const [id, info] of this.workers) {
        const uptime = ((Date.now() - info.startTime) / 1000).toFixed(0);
        console.log(`   Worker ${info.pid}: Load=${info.load}%, Requests=${info.requests}, Uptime=${uptime}s`);
      }
    }, 10000);
  }

  // Graceful shutdown
  shutdown() {
    console.log('\nInitiating graceful shutdown...');
    
    for (const id in cluster.workers) {
      const worker = cluster.workers[id];
      if (worker) {
        worker.send({ type: 'shutdown' });
        
        // Force kill after 10 seconds
        setTimeout(() => {
          if (!worker.isDead()) {
            worker.kill();
          }
        }, 10000);
      }
    }
  }

  // Zero-downtime restart
  async restart() {
    console.log('\n♻️ Performing zero-downtime restart...');
    
    const workerIds = Array.from(this.workers.keys());
    
    for (const id of workerIds) {
      // Fork new worker
      this.forkWorker();
      
      // Wait for new worker to be ready
      await new Promise(resolve => setTimeout(resolve, 2000));
      
      // Gracefully shutdown old worker
      const worker = cluster.workers![id];
      if (worker) {
        worker.disconnect();
      }
      
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
    
    console.log('✅ Zero-downtime restart completed');
  }
}

// Usage
if (cluster.isPrimary) {
  const manager = new ClusterManager();
  manager.startCluster();

  // Handle signals
  process.on('SIGTERM', () => manager.shutdown());
  process.on('SIGUSR2', () => manager.restart()); // Zero-downtime restart

} else {
  // Worker code here
  const api = new ClusteredBankingAPI();
  api.start(3000);

  // Send metrics to primary
  setInterval(() => {
    process.send!({
      type: 'metrics',
      data: {
        load: Math.random() * 100,
        requests: Math.floor(Math.random() * 1000),
        memory: process.memoryUsage()
      }
    });
  }, 5000);

  // Handle shutdown message
  process.on('message', (msg) => {
    if (msg.type === 'shutdown') {
      console.log(`Worker ${process.pid} shutting down gracefully...`);
      // Close connections, finish requests, etc.
      setTimeout(() => process.exit(0), 5000);
    }
  });
}
```

**4. Cluster with Sticky Sessions (WebSocket/Socket.io):**

```typescript
import cluster from 'cluster';
import http from 'http';
import { Server } from 'socket.io';
import express from 'express';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

if (cluster.isPrimary) {
  const numWorkers = cpus().length;
  
  for (let i = 0; i < numWorkers; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker) => {
    console.log(`Worker ${worker.process.pid} died, restarting...`);
    cluster.fork();
  });

} else {
  const app = express();
  const server = http.createServer(app);
  const io = new Server(server);

  // Redis adapter for cross-worker communication
  const pubClient = createClient({ host: 'localhost', port: 6379 });
  const subClient = pubClient.duplicate();

  Promise.all([pubClient.connect(), subClient.connect()]).then(() => {
    io.adapter(createAdapter(pubClient, subClient));
  });

  // Socket.io connection
  io.on('connection', (socket) => {
    console.log(`Client connected to worker ${process.pid}`);

    socket.on('transaction', (data) => {
      // Process transaction
      console.log(`Transaction processed by worker ${process.pid}`);
      
      // Broadcast to all workers
      io.emit('transaction:completed', {
        ...data,
        processedBy: process.pid
      });
    });
  });

  server.listen(3000, () => {
    console.log(`Worker ${process.pid} listening on port 3000`);
  });
}
```

**Key Takeaways for ENBD:**
- Use cluster module to utilize all CPU cores
- Primary process manages worker lifecycle
- Workers share the same port (OS handles load balancing)
- Automatically restart workers on crash
- Implement graceful shutdown to finish pending requests
- Use Redis for shared state across workers
- Monitor worker health and performance
- Zero-downtime deployments with rolling restarts
- Sticky sessions required for WebSocket connections
- Scale horizontally by adding more workers
- Each worker has isolated memory space

---

### **Q32: What's the difference between Worker Threads and Child Processes? When should you use each?**

**Answer:**

Worker Threads and Child Processes are both mechanisms for parallel execution in Node.js, but they differ significantly in resource usage, communication, and use cases.

**Key Differences:**

| Feature | Worker Threads | Child Processes |
|---------|---------------|-----------------|
| Memory | Shared memory (via SharedArrayBuffer) | Separate memory space |
| Resource Usage | Lightweight | Heavy (full V8 instance) |
| Communication | Fast (shared memory) | Slower (IPC/pipes) |
| Use Case | CPU-intensive tasks | Isolated execution, different programs |
| Startup Time | Fast (~10-20ms) | Slow (~50-100ms) |
| Data Sharing | Efficient | Serialization required |

**1. Worker Threads - CPU Intensive Tasks:**

```typescript
import { Worker, isMainThread, parentPort, workerData } from 'worker_threads';

// Worker Threads for transaction validation
class TransactionValidationWorker {
  private worker: Worker | null = null;

  async validateBatch(transactions: any[]): Promise<any[]> {
    return new Promise((resolve, reject) => {
      // Create worker
      this.worker = new Worker(__filename, {
        workerData: { transactions }
      });

      // Listen for results
      this.worker.on('message', (result) => {
        resolve(result);
        this.worker?.terminate();
      });

      this.worker.on('error', reject);
      this.worker.on('exit', (code) => {
        if (code !== 0) {
          reject(new Error(`Worker stopped with exit code ${code}`));
        }
      });
    });
  }
}

if (!isMainThread) {
  // Worker thread code
  const { transactions } = workerData;
  
  // CPU-intensive validation
  const results = transactions.map((txn: any) => {
    // Complex fraud detection algorithm
    const fraudScore = calculateFraudScore(txn);
    
    return {
      ...txn,
      isValid: fraudScore < 0.7,
      fraudScore
    };
  });
  
  parentPort?.postMessage(results);
}

function calculateFraudScore(transaction: any): number {
  // Simulate CPU-intensive calculation
  let score = 0;
  
  // Complex pattern matching
  for (let i = 0; i < 100000; i++) {
    score += Math.sin(transaction.amount * i) * Math.cos(i);
  }
  
  return Math.abs(score) % 1;
}

// Usage
async function processTransactions() {
  const transactions = [
    { id: '1', accountNumber: '123456789012', amount: 5000 },
    { id: '2', accountNumber: '123456789013', amount: 15000 },
    // ... thousands more
  ];

  const validator = new TransactionValidationWorker();
  const results = await validator.validateBatch(transactions);
  
  console.log(`Validated ${results.length} transactions`);
}
```

**2. Worker Thread Pool for High Performance:**

```typescript
import { Worker } from 'worker_threads';
import path from 'path';
import { EventEmitter } from 'events';

interface Task {
  id: string;
  data: any;
  resolve: (value: any) => void;
  reject: (reason: any) => void;
}

class WorkerPool extends EventEmitter {
  private workers: Worker[] = [];
  private freeWorkers: Worker[] = [];
  private taskQueue: Task[] = [];
  private workerScript: string;

  constructor(workerScript: string, poolSize: number = 4) {
    super();
    this.workerScript = workerScript;
    
    // Initialize worker pool
    for (let i = 0; i < poolSize; i++) {
      this.createWorker();
    }
  }

  private createWorker() {
    const worker = new Worker(this.workerScript);
    
    worker.on('message', (result: any) => {
      this.handleWorkerMessage(worker, result);
    });

    worker.on('error', (error) => {
      console.error('Worker error:', error);
      this.removeWorker(worker);
      this.createWorker(); // Replace failed worker
    });

    worker.on('exit', (code) => {
      if (code !== 0) {
        console.error(`Worker exited with code ${code}`);
      }
      this.removeWorker(worker);
    });

    this.workers.push(worker);
    this.freeWorkers.push(worker);
  }

  private handleWorkerMessage(worker: Worker, result: any) {
    // Worker completed task
    this.freeWorkers.push(worker);
    
    // Process next task if any
    if (this.taskQueue.length > 0) {
      this.processNextTask();
    }
    
    this.emit('taskComplete', result);
  }

  private removeWorker(worker: Worker) {
    this.workers = this.workers.filter(w => w !== worker);
    this.freeWorkers = this.freeWorkers.filter(w => w !== worker);
  }

  async execute(data: any): Promise<any> {
    return new Promise((resolve, reject) => {
      const task: Task = {
        id: Math.random().toString(36).substr(2, 9),
        data,
        resolve,
        reject
      };

      this.taskQueue.push(task);
      this.processNextTask();
    });
  }

  private processNextTask() {
    if (this.taskQueue.length === 0 || this.freeWorkers.length === 0) {
      return;
    }

    const task = this.taskQueue.shift()!;
    const worker = this.freeWorkers.shift()!;

    worker.postMessage(task.data);
  }

  async destroy() {
    for (const worker of this.workers) {
      await worker.terminate();
    }
    this.workers = [];
    this.freeWorkers = [];
    this.taskQueue = [];
  }

  getStats() {
    return {
      totalWorkers: this.workers.length,
      freeWorkers: this.freeWorkers.length,
      queuedTasks: this.taskQueue.length,
      busyWorkers: this.workers.length - this.freeWorkers.length
    };
  }
}

// Transaction processing with worker pool
class TransactionProcessor {
  private workerPool: WorkerPool;

  constructor() {
    this.workerPool = new WorkerPool(
      path.join(__dirname, 'transaction-worker.js'),
      4 // 4 workers
    );

    this.workerPool.on('taskComplete', (result) => {
      console.log(`Task completed:`, result);
    });
  }

  async processTransaction(transaction: any): Promise<any> {
    const startTime = Date.now();
    
    try {
      const result = await this.workerPool.execute({
        type: 'validate',
        transaction
      });

      const processingTime = Date.now() - startTime;
      
      return {
        ...result,
        processingTime,
        stats: this.workerPool.getStats()
      };
    } catch (error) {
      throw new Error(`Transaction processing failed: ${error}`);
    }
  }

  async processBatch(transactions: any[]): Promise<any[]> {
    const promises = transactions.map(txn => 
      this.processTransaction(txn)
    );

    return Promise.all(promises);
  }

  async shutdown() {
    await this.workerPool.destroy();
  }
}

// Usage
async function main() {
  const processor = new TransactionProcessor();

  const transactions = Array.from({ length: 100 }, (_, i) => ({
    id: `TXN-${i}`,
    accountNumber: `12345678901${i % 10}`,
    amount: Math.random() * 10000,
    timestamp: new Date()
  }));

  console.log('Processing 100 transactions with worker pool...');
  const results = await processor.processBatch(transactions);
  
  console.log(`Processed ${results.length} transactions`);
  console.log(`Average processing time: ${
    results.reduce((sum, r) => sum + r.processingTime, 0) / results.length
  }ms`);

  await processor.shutdown();
}
```

**3. Child Processes - Isolated Execution:**

```typescript
import { spawn, fork, exec, execFile } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

// Child process for running external programs
class ReportGenerator {
  
  // Using spawn for streaming large reports
  async generatePDFReport(transactions: any[]): Promise<Buffer> {
    return new Promise((resolve, reject) => {
      // Spawn external PDF generation tool
      const child = spawn('python3', [
        'generate_report.py',
        '--format', 'pdf'
      ]);

      const chunks: Buffer[] = [];

      // Stream transaction data to Python process
      child.stdin.write(JSON.stringify(transactions));
      child.stdin.end();

      // Collect PDF output
      child.stdout.on('data', (chunk) => {
        chunks.push(chunk);
      });

      child.stderr.on('data', (data) => {
        console.error(`Report generation error: ${data}`);
      });

      child.on('close', (code) => {
        if (code !== 0) {
          reject(new Error(`Report generation failed with code ${code}`));
        } else {
          resolve(Buffer.concat(chunks));
        }
      });
    });
  }

  // Using exec for simple commands
  async compressReport(filePath: string): Promise<string> {
    try {
      const { stdout, stderr } = await execAsync(
        `gzip -c ${filePath} > ${filePath}.gz`
      );
      
      if (stderr) {
        console.error('Compression warning:', stderr);
      }
      
      return `${filePath}.gz`;
    } catch (error) {
      throw new Error(`Compression failed: ${error}`);
    }
  }

  // Using fork for Node.js child processes
  async processLargeDataset(dataPath: string): Promise<any> {
    return new Promise((resolve, reject) => {
      // Fork a separate Node.js process
      const child = fork(path.join(__dirname, 'data-processor.js'));

      child.send({ type: 'process', dataPath });

      child.on('message', (message: any) => {
        if (message.type === 'complete') {
          resolve(message.result);
          child.kill();
        } else if (message.type === 'error') {
          reject(new Error(message.error));
          child.kill();
        } else if (message.type === 'progress') {
          console.log(`Progress: ${message.percent}%`);
        }
      });

      child.on('error', reject);
      
      child.on('exit', (code) => {
        if (code !== 0) {
          reject(new Error(`Child process exited with code ${code}`));
        }
      });
    });
  }
}

// data-processor.js (child process)
/*
process.on('message', async (message) => {
  if (message.type === 'process') {
    try {
      const data = await loadLargeDataset(message.dataPath);
      
      for (let i = 0; i < data.length; i++) {
        // Process data
        processRecord(data[i]);
        
        // Send progress updates
        if (i % 1000 === 0) {
          process.send!({
            type: 'progress',
            percent: (i / data.length * 100).toFixed(2)
          });
        }
      }
      
      process.send!({
        type: 'complete',
        result: { processed: data.length }
      });
    } catch (error) {
      process.send!({
        type: 'error',
        error: error.message
      });
    }
  }
});
*/
```

**4. Comparison: Processing 10,000 Transactions:**

```typescript
import { performance } from 'perf_hooks';

class PerformanceComparison {
  
  // Worker Threads approach
  async withWorkerThreads(transactions: any[]): Promise<void> {
    const start = performance.now();
    
    const pool = new WorkerPool('./worker.js', 4);
    
    // Split into batches
    const batchSize = 2500;
    const batches: any[][] = [];
    
    for (let i = 0; i < transactions.length; i += batchSize) {
      batches.push(transactions.slice(i, i + batchSize));
    }
    
    // Process batches in parallel
    const promises = batches.map(batch => pool.execute(batch));
    await Promise.all(promises);
    
    await pool.destroy();
    
    const duration = performance.now() - start;
    console.log(`Worker Threads: ${duration.toFixed(2)}ms`);
    console.log(`Memory: ${(process.memoryUsage().heapUsed / 1024 / 1024).toFixed(2)}MB`);
  }

  // Child Process approach
  async withChildProcesses(transactions: any[]): Promise<void> {
    const start = performance.now();
    
    const batchSize = 2500;
    const processes: Promise<any>[] = [];
    
    for (let i = 0; i < transactions.length; i += batchSize) {
      const batch = transactions.slice(i, i + batchSize);
      
      processes.push(new Promise((resolve, reject) => {
        const child = fork('./processor.js');
        
        child.send({ transactions: batch });
        
        child.on('message', (result) => {
          resolve(result);
          child.kill();
        });
        
        child.on('error', reject);
      }));
    }
    
    await Promise.all(processes);
    
    const duration = performance.now() - start;
    console.log(`Child Processes: ${duration.toFixed(2)}ms`);
    console.log(`Memory: ${(process.memoryUsage().heapUsed / 1024 / 1024).toFixed(2)}MB`);
  }

  // Run comparison
  async compare() {
    const transactions = Array.from({ length: 10000 }, (_, i) => ({
      id: `TXN-${i}`,
      accountNumber: `12345678901${i % 10}`,
      amount: Math.random() * 10000,
      timestamp: new Date()
    }));

    console.log('\n=== Performance Comparison ===');
    console.log(`Processing ${transactions.length} transactions\n`);

    await this.withWorkerThreads(transactions);
    console.log('');
    await this.withChildProcesses(transactions);
    
    console.log('\n=== Results ===');
    console.log('Worker Threads: Faster startup, shared memory, lower overhead');
    console.log('Child Processes: Isolated, can run different programs, higher overhead');
  }
}

// Run comparison
new PerformanceComparison().compare();
```

**5. When to Use Each:**

```typescript
// Decision Matrix
class ParallelExecutionStrategy {
  
  static choose(task: {
    type: 'cpu' | 'io' | 'external';
    isolation: 'required' | 'optional';
    dataSize: 'small' | 'large';
    frequency: 'high' | 'low';
  }): 'worker_threads' | 'child_process' | 'cluster' {
    
    // External programs → Child Process
    if (task.type === 'external') {
      return 'child_process';
    }
    
    // Strict isolation required → Child Process
    if (task.isolation === 'required') {
      return 'child_process';
    }
    
    // CPU-intensive + frequent → Worker Threads
    if (task.type === 'cpu' && task.frequency === 'high') {
      return 'worker_threads';
    }
    
    // I/O bound + scale horizontally → Cluster
    if (task.type === 'io') {
      return 'cluster';
    }
    
    // Default: Worker Threads (lighter weight)
    return 'worker_threads';
  }
}

// Example decisions:
console.log(ParallelExecutionStrategy.choose({
  type: 'cpu',
  isolation: 'optional',
  dataSize: 'small',
  frequency: 'high'
})); // → worker_threads

console.log(ParallelExecutionStrategy.choose({
  type: 'external',
  isolation: 'required',
  dataSize: 'large',
  frequency: 'low'
})); // → child_process
```

**Key Takeaways for ENBD:**
- **Worker Threads**: CPU-intensive tasks (fraud detection, encryption, data analysis)
- **Child Processes**: Isolated execution, external programs (PDF generation, image processing)
- Worker Threads share memory → faster communication, lower overhead
- Child Processes have separate memory → better isolation, can crash independently
- Use Worker Thread pool for high-frequency tasks
- Use Child Processes for running Python/Java/external tools
- Worker Threads: ~10-20ms startup, Child Process: ~50-100ms startup
- Cluster module for scaling I/O-bound applications across cores
- Worker Threads can share ArrayBuffer/SharedArrayBuffer
- Child Processes require data serialization (JSON)
- Monitor memory usage - Child Processes use more memory

---

### **Q33: How do you detect and fix memory leaks in Node.js applications?**

**Answer:**

Memory leaks in Node.js occur when the application retains references to objects that are no longer needed, preventing garbage collection. This is critical for long-running banking applications.

**1. Common Memory Leak Causes:**

```typescript
// ❌ BAD: Memory leak examples

// Leak 1: Global variables accumulating data
const transactionCache: any[] = []; // Never cleared!

function processTransaction(txn: any) {
  transactionCache.push(txn); // Grows forever
  // ... process transaction
}

// Leak 2: Event listeners not removed
class TransactionMonitor {
  constructor() {
    setInterval(() => {
      // This timer never gets cleared
      this.checkTransactions();
    }, 1000);
  }
  
  checkTransactions() {
    // Monitoring logic
  }
}

// Leak 3: Closures holding references
function createProcessor() {
  const largeData = new Array(1000000).fill('data');
  
  return function process() {
    // This closure holds reference to largeData forever
    console.log('Processing...');
  };
}

// Leak 4: Forgotten event emitters
import { EventEmitter } from 'events';

class LeakyService extends EventEmitter {
  start() {
    this.on('data', (data) => {
      // Listener never removed
      console.log(data);
    });
  }
}
```

**2. Memory Profiling with Node.js Built-in Tools:**

```typescript
import v8 from 'v8';
import fs from 'fs';
import path from 'path';

class MemoryProfiler {
  private snapshots: Map<string, any> = new Map();

  // Take heap snapshot
  takeSnapshot(name: string) {
    const snapshotStream = v8.writeHeapSnapshot();
    const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
    const filename = `heap-${name}-${timestamp}.heapsnapshot`;
    
    console.log(`📸 Heap snapshot saved: ${filename}`);
    return filename;
  }

  // Get current memory usage
  getMemoryUsage() {
    const usage = process.memoryUsage();
    
    return {
      rss: `${(usage.rss / 1024 / 1024).toFixed(2)} MB`, // Total memory
      heapTotal: `${(usage.heapTotal / 1024 / 1024).toFixed(2)} MB`,
      heapUsed: `${(usage.heapUsed / 1024 / 1024).toFixed(2)} MB`,
      external: `${(usage.external / 1024 / 1024).toFixed(2)} MB`,
      arrayBuffers: `${(usage.arrayBuffers / 1024 / 1024).toFixed(2)} MB`
    };
  }

  // Monitor memory over time
  startMonitoring(intervalMs: number = 5000) {
    const measurements: any[] = [];
    
    const interval = setInterval(() => {
      const usage = process.memoryUsage();
      const timestamp = new Date().toISOString();
      
      measurements.push({
        timestamp,
        heapUsed: usage.heapUsed,
        heapTotal: usage.heapTotal,
        external: usage.external
      });
      
      console.log(`[${timestamp}] Heap Used: ${(usage.heapUsed / 1024 / 1024).toFixed(2)} MB`);
      
      // Alert if memory grows significantly
      if (measurements.length > 10) {
        const first = measurements[0].heapUsed;
        const current = usage.heapUsed;
        const growth = ((current - first) / first) * 100;
        
        if (growth > 50) {
          console.warn(`⚠️  Memory grew ${growth.toFixed(2)}% - possible leak!`);
        }
      }
    }, intervalMs);

    return {
      stop: () => clearInterval(interval),
      getMeasurements: () => measurements
    };
  }

  // Get heap statistics
  getHeapStatistics() {
    const stats = v8.getHeapStatistics();
    
    return {
      totalHeapSize: `${(stats.total_heap_size / 1024 / 1024).toFixed(2)} MB`,
      usedHeapSize: `${(stats.used_heap_size / 1024 / 1024).toFixed(2)} MB`,
      heapSizeLimit: `${(stats.heap_size_limit / 1024 / 1024).toFixed(2)} MB`,
      mallocedMemory: `${(stats.malloced_memory / 1024 / 1024).toFixed(2)} MB`,
      peakMallocedMemory: `${(stats.peak_malloced_memory / 1024 / 1024).toFixed(2)} MB`
    };
  }

  // Force garbage collection (requires --expose-gc flag)
  forceGC() {
    if (global.gc) {
      const before = process.memoryUsage().heapUsed;
      global.gc();
      const after = process.memoryUsage().heapUsed;
      const freed = before - after;
      
      console.log(`🗑️  GC freed ${(freed / 1024 / 1024).toFixed(2)} MB`);
      return freed;
    } else {
      console.warn('GC not exposed. Run with --expose-gc flag');
      return 0;
    }
  }
}

// Usage
const profiler = new MemoryProfiler();

console.log('Initial memory:', profiler.getMemoryUsage());
console.log('Heap stats:', profiler.getHeapStatistics());

// Start monitoring
const monitor = profiler.startMonitoring(5000);

// ... run your application

// Take snapshot before and after
profiler.takeSnapshot('before-operation');
// ... perform operation
profiler.takeSnapshot('after-operation');

// Compare snapshots in Chrome DevTools
```

**3. Memory Leak Detector:**

```typescript
class MemoryLeakDetector {
  private baseline: number = 0;
  private samples: number[] = [];
  private readonly sampleSize: number = 10;
  private readonly growthThreshold: number = 0.1; // 10% growth

  start() {
    this.baseline = process.memoryUsage().heapUsed;
    
    const interval = setInterval(() => {
      this.checkForLeaks();
    }, 10000); // Check every 10 seconds

    return () => clearInterval(interval);
  }

  private checkForLeaks() {
    const currentUsage = process.memoryUsage().heapUsed;
    this.samples.push(currentUsage);

    // Keep only recent samples
    if (this.samples.length > this.sampleSize) {
      this.samples.shift();
    }

    // Need enough samples to analyze
    if (this.samples.length < this.sampleSize) {
      return;
    }

    // Calculate trend
    const trend = this.calculateTrend();
    const growth = (currentUsage - this.baseline) / this.baseline;

    console.log(`
📊 Memory Analysis:
   Current: ${(currentUsage / 1024 / 1024).toFixed(2)} MB
   Baseline: ${(this.baseline / 1024 / 1024).toFixed(2)} MB
   Growth: ${(growth * 100).toFixed(2)}%
   Trend: ${trend > 0 ? '📈 Increasing' : '📉 Decreasing'}
    `);

    // Detect leak
    if (growth > this.growthThreshold && trend > 0) {
      this.reportLeak(currentUsage, growth);
    }
  }

  private calculateTrend(): number {
    // Simple linear regression
    const n = this.samples.length;
    let sumX = 0, sumY = 0, sumXY = 0, sumX2 = 0;

    for (let i = 0; i < n; i++) {
      sumX += i;
      sumY += this.samples[i];
      sumXY += i * this.samples[i];
      sumX2 += i * i;
    }

    const slope = (n * sumXY - sumX * sumY) / (n * sumX2 - sumX * sumX);
    return slope;
  }

  private reportLeak(currentUsage: number, growth: number) {
    console.error(`
🚨 MEMORY LEAK DETECTED!
   Current Usage: ${(currentUsage / 1024 / 1024).toFixed(2)} MB
   Growth: ${(growth * 100).toFixed(2)}%
   
   Recommendations:
   1. Take heap snapshot: node --inspect app.js
   2. Check for:
      - Uncleared intervals/timeouts
      - Event listeners not removed
      - Global variables accumulating data
      - Closures holding large objects
   3. Use Chrome DevTools Memory Profiler
    `);

    // Optionally take heap snapshot
    const profiler = new MemoryProfiler();
    profiler.takeSnapshot('leak-detected');
  }
}
```

**4. Fixed Examples (Memory Leak Prevention):**

```typescript
// ✅ GOOD: Fixed implementations

// Fix 1: LRU Cache with size limit
class TransactionCache {
  private cache: Map<string, any> = new Map();
  private readonly maxSize: number = 10000;

  set(key: string, value: any) {
    // Remove oldest if at capacity
    if (this.cache.size >= this.maxSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    
    this.cache.set(key, value);
  }

  get(key: string) {
    return this.cache.get(key);
  }

  clear() {
    this.cache.clear();
  }

  getSize() {
    return this.cache.size;
  }
}

// Fix 2: Proper cleanup in class
class TransactionMonitor {
  private interval: NodeJS.Timeout | null = null;
  private isRunning: boolean = false;

  start() {
    if (this.isRunning) return;
    
    this.isRunning = true;
    this.interval = setInterval(() => {
      this.checkTransactions();
    }, 1000);
  }

  stop() {
    if (this.interval) {
      clearInterval(this.interval);
      this.interval = null;
      this.isRunning = false;
    }
  }

  checkTransactions() {
    // Monitoring logic
  }
  
  // Cleanup method
  destroy() {
    this.stop();
  }
}

// Fix 3: Avoid closure memory retention
function createProcessor() {
  // Don't capture large data unnecessarily
  return function process() {
    // Only reference what you need
    console.log('Processing...');
  };
}

// Fix 4: Proper event emitter cleanup
class FixedService extends EventEmitter {
  private dataListener: ((data: any) => void) | null = null;

  start() {
    this.dataListener = (data) => {
      console.log(data);
    };
    
    this.on('data', this.dataListener);
  }

  stop() {
    if (this.dataListener) {
      this.removeListener('data', this.dataListener);
      this.dataListener = null;
    }
  }
}

// Fix 5: WeakMap for object references
class TransactionProcessor {
  // WeakMap allows garbage collection
  private processedTransactions = new WeakMap<object, any>();

  process(transaction: object) {
    const result = this.doProcessing(transaction);
    this.processedTransactions.set(transaction, result);
    return result;
  }

  private doProcessing(transaction: object) {
    // Processing logic
    return { processed: true };
  }
}
```

**5. Banking Application with Memory Management:**

```typescript
class ENBDBankingAPI {
  private transactionCache: TransactionCache;
  private monitor: TransactionMonitor;
  private leakDetector: MemoryLeakDetector;
  private profiler: MemoryProfiler;
  private cleanupInterval: NodeJS.Timeout | null = null;

  constructor() {
    this.transactionCache = new TransactionCache();
    this.monitor = new TransactionMonitor();
    this.leakDetector = new MemoryLeakDetector();
    this.profiler = new MemoryProfiler();
    
    this.setupMemoryManagement();
  }

  private setupMemoryManagement() {
    // Start leak detection
    this.leakDetector.start();

    // Periodic cleanup
    this.cleanupInterval = setInterval(() => {
      this.performCleanup();
    }, 60000); // Every minute

    // Log memory usage
    setInterval(() => {
      const usage = this.profiler.getMemoryUsage();
      console.log(`Memory: Heap Used = ${usage.heapUsed}`);
    }, 30000);

    // Handle process signals
    process.on('SIGTERM', () => this.shutdown());
    process.on('SIGINT', () => this.shutdown());

    // Monitor for approaching memory limit
    this.monitorMemoryLimit();
  }

  private performCleanup() {
    console.log('🧹 Performing cleanup...');
    
    // Clear old cache entries
    const cacheSize = this.transactionCache.getSize();
    console.log(`Cache size: ${cacheSize} entries`);

    // Force GC if needed
    const heapUsed = process.memoryUsage().heapUsed;
    const heapLimit = v8.getHeapStatistics().heap_size_limit;
    const usage = heapUsed / heapLimit;

    if (usage > 0.8) {
      console.warn(`⚠️  Heap usage at ${(usage * 100).toFixed(2)}%`);
      this.profiler.forceGC();
    }
  }

  private monitorMemoryLimit() {
    setInterval(() => {
      const usage = process.memoryUsage();
      const limit = v8.getHeapStatistics().heap_size_limit;
      const percent = (usage.heapUsed / limit) * 100;

      if (percent > 90) {
        console.error(`🚨 CRITICAL: Memory usage at ${percent.toFixed(2)}%`);
        this.performEmergencyCleanup();
      } else if (percent > 75) {
        console.warn(`⚠️  WARNING: Memory usage at ${percent.toFixed(2)}%`);
      }
    }, 5000);
  }

  private performEmergencyCleanup() {
    console.log('🚨 Emergency cleanup initiated...');
    
    // Clear all caches
    this.transactionCache.clear();
    
    // Force garbage collection
    this.profiler.forceGC();
    
    // Take snapshot for analysis
    this.profiler.takeSnapshot('emergency-cleanup');
  }

  async processTransaction(transaction: any) {
    // Process transaction
    const result = await this.doProcess(transaction);
    
    // Cache result
    this.transactionCache.set(transaction.id, result);
    
    return result;
  }

  private async doProcess(transaction: any) {
    // Simulate processing
    return {
      id: transaction.id,
      status: 'completed',
      timestamp: new Date()
    };
  }

  async shutdown() {
    console.log('\n🛑 Shutting down gracefully...');
    
    // Stop monitoring
    this.monitor.stop();
    
    // Clear cleanup interval
    if (this.cleanupInterval) {
      clearInterval(this.cleanupInterval);
    }
    
    // Final cleanup
    this.transactionCache.clear();
    
    // Take final snapshot
    this.profiler.takeSnapshot('shutdown');
    
    console.log('✅ Shutdown complete');
    process.exit(0);
  }
}

// Start application with monitoring
const api = new ENBDBankingAPI();

// Simulate load
setInterval(async () => {
  await api.processTransaction({
    id: Math.random().toString(36),
    amount: Math.random() * 10000
  });
}, 100);
```

**Key Takeaways for ENBD:**
- Use heap snapshots to identify memory leaks (Chrome DevTools)
- Monitor memory usage continuously in production
- Implement LRU caches with size limits
- Always clean up timers, intervals, and event listeners
- Use WeakMap/WeakSet for automatic garbage collection
- Avoid global variables that accumulate data
- Be careful with closures capturing large objects
- Set memory limits: `node --max-old-space-size=4096 app.js`
- Run with `--expose-gc` for manual GC in development
- Take heap snapshots before/after operations to compare
- Monitor heap usage percentage (alert at 75%, critical at 90%)
- Implement graceful shutdown with cleanup
- Use `process.memoryUsage()` for runtime monitoring
- Analyze trends - consistent growth indicates leak
- Profile in production with minimal overhead tools

---

### **Q34: How do you profile CPU performance and optimize hot code paths in Node.js?**

**Answer:**

CPU profiling helps identify bottlenecks and optimize performance-critical code paths. This is crucial for high-throughput banking applications processing thousands of transactions per second.

**1. Built-in CPU Profiler:**

```typescript
import { Session } from 'inspector';
import fs from 'fs';

class CPUProfiler {
  private session: Session | null = null;

  start(duration: number = 60000) {
    this.session = new Session();
    this.session.connect();

    // Start profiling
    this.session.post('Profiler.enable', () => {
      this.session!.post('Profiler.start', () => {
        console.log(`🔍 CPU profiling started for ${duration}ms`);
      });
    });

    // Stop after duration
    setTimeout(() => {
      this.stop();
    }, duration);
  }

  stop() {
    if (!this.session) return;

    this.session.post('Profiler.stop', (err, { profile }) => {
      if (err) {
        console.error('Profiling error:', err);
        return;
      }

      // Save profile
      const filename = `cpu-profile-${Date.now()}.cpuprofile`;
      fs.writeFileSync(filename, JSON.stringify(profile));
      
      console.log(`✅ CPU profile saved: ${filename}`);
      console.log('   Open in Chrome DevTools > Performance > Load Profile');
      
      this.session!.disconnect();
      this.session = null;
    });
  }
}

// Usage
const profiler = new CPUProfiler();
profiler.start(30000); // Profile for 30 seconds
```

**2. Transaction Processing with Performance Analysis:**

```typescript
import { performance, PerformanceObserver } from 'perf_hooks';

interface Transaction {
  id: string;
  accountNumber: string;
  amount: number;
  type: 'debit' | 'credit';
  timestamp: Date;
}

class PerformanceAnalyzer {
  private measurements: Map<string, number[]> = new Map();
  private observer: PerformanceObserver;

  constructor() {
    // Set up performance observer
    this.observer = new PerformanceObserver((items) => {
      items.getEntries().forEach((entry) => {
        console.log(`⏱️  ${entry.name}: ${entry.duration.toFixed(2)}ms`);
      });
    });

    this.observer.observe({ entryTypes: ['measure', 'function'] });
  }

  // Measure function execution
  measure<T>(name: string, fn: () => T): T {
    const start = performance.now();
    
    try {
      return fn();
    } finally {
      const duration = performance.now() - start;
      
      // Store measurement
      if (!this.measurements.has(name)) {
        this.measurements.set(name, []);
      }
      this.measurements.get(name)!.push(duration);
      
      // Create performance mark
      performance.mark(`${name}-end`);
      performance.measure(name, { start, end: performance.now() });
    }
  }

  // Measure async function
  async measureAsync<T>(name: string, fn: () => Promise<T>): Promise<T> {
    const start = performance.now();
    
    try {
      return await fn();
    } finally {
      const duration = performance.now() - start;
      
      if (!this.measurements.has(name)) {
        this.measurements.set(name, []);
      }
      this.measurements.get(name)!.push(duration);
    }
  }

  // Get statistics
  getStats(name: string) {
    const measurements = this.measurements.get(name);
    if (!measurements || measurements.length === 0) {
      return null;
    }

    const sorted = [...measurements].sort((a, b) => a - b);
    const sum = measurements.reduce((a, b) => a + b, 0);

    return {
      count: measurements.length,
      min: sorted[0].toFixed(2),
      max: sorted[sorted.length - 1].toFixed(2),
      avg: (sum / measurements.length).toFixed(2),
      median: sorted[Math.floor(sorted.length / 2)].toFixed(2),
      p95: sorted[Math.floor(sorted.length * 0.95)].toFixed(2),
      p99: sorted[Math.floor(sorted.length * 0.99)].toFixed(2)
    };
  }

  // Report all stats
  report() {
    console.log('\n📊 Performance Report:');
    console.log('='.repeat(70));
    
    for (const [name, _] of this.measurements) {
      const stats = this.getStats(name);
      if (stats) {
        console.log(`\n${name}:`);
        console.log(`  Count: ${stats.count}`);
        console.log(`  Min: ${stats.min}ms | Max: ${stats.max}ms | Avg: ${stats.avg}ms`);
        console.log(`  Median: ${stats.median}ms | P95: ${stats.p95}ms | P99: ${stats.p99}ms`);
      }
    }
    
    console.log('='.repeat(70));
  }

  clear() {
    this.measurements.clear();
    performance.clearMarks();
    performance.clearMeasures();
  }

  disconnect() {
    this.observer.disconnect();
  }
}

// Transaction processor with performance tracking
class OptimizedTransactionProcessor {
  private analyzer: PerformanceAnalyzer;
  private accountCache: Map<string, any> = new Map();

  constructor() {
    this.analyzer = new PerformanceAnalyzer();
  }

  // ❌ SLOW: Unoptimized version
  processTransactionSlow(transaction: Transaction): any {
    return this.analyzer.measure('processTransaction-slow', () => {
      // Inefficient account lookup
      const account = this.lookupAccountSlow(transaction.accountNumber);
      
      // Inefficient validation
      const isValid = this.validateTransactionSlow(transaction, account);
      
      if (!isValid) {
        throw new Error('Invalid transaction');
      }
      
      // Inefficient balance calculation
      const newBalance = this.calculateBalanceSlow(account, transaction);
      
      return { transaction, account, newBalance };
    });
  }

  private lookupAccountSlow(accountNumber: string): any {
    // Simulate slow database lookup
    let result = null;
    for (let i = 0; i < 10000; i++) {
      result = { accountNumber, balance: 10000 };
    }
    return result;
  }

  private validateTransactionSlow(transaction: Transaction, account: any): boolean {
    // Inefficient validation logic
    const rules = [
      () => transaction.amount > 0,
      () => transaction.amount <= 1000000,
      () => account.balance >= transaction.amount,
      () => transaction.accountNumber.length === 12,
      () => /^\d+$/.test(transaction.accountNumber)
    ];
    
    // Slow: checks all rules even if one fails
    return rules.every(rule => rule());
  }

  private calculateBalanceSlow(account: any, transaction: Transaction): number {
    // Inefficient calculation
    let balance = account.balance;
    for (let i = 0; i < 1000; i++) {
      balance = account.balance + (transaction.type === 'credit' ? transaction.amount : -transaction.amount);
    }
    return balance;
  }

  // ✅ FAST: Optimized version
  processTransactionFast(transaction: Transaction): any {
    return this.analyzer.measure('processTransaction-fast', () => {
      // Cached account lookup
      const account = this.lookupAccountFast(transaction.accountNumber);
      
      // Short-circuit validation
      const isValid = this.validateTransactionFast(transaction, account);
      
      if (!isValid) {
        throw new Error('Invalid transaction');
      }
      
      // Direct calculation
      const newBalance = this.calculateBalanceFast(account, transaction);
      
      return { transaction, account, newBalance };
    });
  }

  private lookupAccountFast(accountNumber: string): any {
    // Use cache
    if (this.accountCache.has(accountNumber)) {
      return this.accountCache.get(accountNumber);
    }
    
    const account = { accountNumber, balance: 10000 };
    this.accountCache.set(accountNumber, account);
    return account;
  }

  private validateTransactionFast(transaction: Transaction, account: any): boolean {
    // Short-circuit: return immediately on first failure
    if (transaction.amount <= 0) return false;
    if (transaction.amount > 1000000) return false;
    if (account.balance < transaction.amount) return false;
    if (transaction.accountNumber.length !== 12) return false;
    if (!/^\d+$/.test(transaction.accountNumber)) return false;
    
    return true;
  }

  private calculateBalanceFast(account: any, transaction: Transaction): number {
    // Direct calculation - no loops
    return account.balance + (transaction.type === 'credit' ? transaction.amount : -transaction.amount);
  }

  // Benchmark both versions
  benchmark(iterations: number = 1000) {
    console.log(`\n🏁 Running benchmark with ${iterations} iterations...\n`);
    
    const transactions: Transaction[] = Array.from({ length: iterations }, (_, i) => ({
      id: `TXN-${i}`,
      accountNumber: '123456789012',
      amount: Math.random() * 10000,
      type: Math.random() > 0.5 ? 'credit' : 'debit',
      timestamp: new Date()
    }));

    // Warm up
    for (let i = 0; i < 100; i++) {
      this.processTransactionFast(transactions[0]);
    }

    // Benchmark slow version
    console.log('Running slow version...');
    for (const txn of transactions) {
      try {
        this.processTransactionSlow(txn);
      } catch (e) {
        // Continue
      }
    }

    // Benchmark fast version
    console.log('Running fast version...');
    for (const txn of transactions) {
      try {
        this.processTransactionFast(txn);
      } catch (e) {
        // Continue
      }
    }

    // Report results
    this.analyzer.report();
    
    const slowStats = this.analyzer.getStats('processTransaction-slow');
    const fastStats = this.analyzer.getStats('processTransaction-fast');
    
    if (slowStats && fastStats) {
      const improvement = ((parseFloat(slowStats.avg) / parseFloat(fastStats.avg)) - 1) * 100;
      console.log(`\n🚀 Performance Improvement: ${improvement.toFixed(2)}%`);
    }
  }

  cleanup() {
    this.analyzer.disconnect();
  }
}

// Run benchmark
const processor = new OptimizedTransactionProcessor();
processor.benchmark(10000);
processor.cleanup();
```

**3. Hot Path Optimization Techniques:**

```typescript
class OptimizationTechniques {
  
  // ❌ BAD: String concatenation in loop
  badStringConcat(transactions: Transaction[]): string {
    let report = '';
    for (const txn of transactions) {
      report += `${txn.id},${txn.amount}\n`; // Creates new string each time!
    }
    return report;
  }

  // ✅ GOOD: Use array join
  goodStringConcat(transactions: Transaction[]): string {
    const lines = transactions.map(txn => `${txn.id},${txn.amount}`);
    return lines.join('\n');
  }

  // ❌ BAD: Repeated regex creation
  badRegex(transactions: Transaction[]): Transaction[] {
    return transactions.filter(txn => {
      return /^\d{12}$/.test(txn.accountNumber); // Regex compiled each time!
    });
  }

  // ✅ GOOD: Reuse regex
  private accountRegex = /^\d{12}$/;
  goodRegex(transactions: Transaction[]): Transaction[] {
    return transactions.filter(txn => 
      this.accountRegex.test(txn.accountNumber)
    );
  }

  // ❌ BAD: Unnecessary object creation
  badObjectCreation(count: number): any[] {
    const results = [];
    for (let i = 0; i < count; i++) {
      results.push({
        id: i,
        data: new Array(1000).fill(0) // Creates array every time!
      });
    }
    return results;
  }

  // ✅ GOOD: Object pooling
  private pool: any[] = [];
  goodObjectCreation(count: number): any[] {
    const results = [];
    for (let i = 0; i < count; i++) {
      let obj = this.pool.pop();
      if (!obj) {
        obj = { id: 0, data: new Array(1000).fill(0) };
      }
      obj.id = i;
      results.push(obj);
    }
    return results;
  }

  releaseToPool(objects: any[]) {
    this.pool.push(...objects);
  }

  // ❌ BAD: Synchronous blocking operations
  badSync(transactions: Transaction[]): any[] {
    return transactions.map(txn => {
      // Simulate blocking operation
      const start = Date.now();
      while (Date.now() - start < 10) {} // Blocks event loop!
      return { ...txn, processed: true };
    });
  }

  // ✅ GOOD: Async with batching
  async goodAsync(transactions: Transaction[], batchSize: number = 100): Promise<any[]> {
    const results: any[] = [];
    
    for (let i = 0; i < transactions.length; i += batchSize) {
      const batch = transactions.slice(i, i + batchSize);
      
      // Process batch
      const batchResults = await Promise.all(
        batch.map(async txn => {
          // Non-blocking operation
          await new Promise(resolve => setImmediate(resolve));
          return { ...txn, processed: true };
        })
      );
      
      results.push(...batchResults);
    }
    
    return results;
  }

  // ❌ BAD: Array operations creating copies
  badArrayOps(transactions: Transaction[]): number {
    return transactions
      .filter(t => t.type === 'credit')
      .map(t => t.amount)
      .reduce((sum, amount) => sum + amount, 0);
  }

  // ✅ GOOD: Single pass
  goodArrayOps(transactions: Transaction[]): number {
    let sum = 0;
    for (const txn of transactions) {
      if (txn.type === 'credit') {
        sum += txn.amount;
      }
    }
    return sum;
  }

  // ❌ BAD: Unnecessary JSON parsing
  badJsonParsing(data: string[]): any[] {
    return data.map(str => JSON.parse(str)); // Parsing every time!
  }

  // ✅ GOOD: Parse once, cache
  private parsedCache = new Map<string, any>();
  goodJsonParsing(data: string[]): any[] {
    return data.map(str => {
      if (!this.parsedCache.has(str)) {
        this.parsedCache.set(str, JSON.parse(str));
      }
      return this.parsedCache.get(str);
    });
  }
}

// Benchmark optimizations
function benchmarkOptimizations() {
  const techniques = new OptimizationTechniques();
  const transactions: Transaction[] = Array.from({ length: 10000 }, (_, i) => ({
    id: `TXN-${i}`,
    accountNumber: '123456789012',
    amount: Math.random() * 10000,
    type: Math.random() > 0.5 ? 'credit' : 'debit',
    timestamp: new Date()
  }));

  console.log('Benchmarking string concatenation...');
  console.time('bad-concat');
  techniques.badStringConcat(transactions);
  console.timeEnd('bad-concat');

  console.time('good-concat');
  techniques.goodStringConcat(transactions);
  console.timeEnd('good-concat');

  console.log('\nBenchmarking array operations...');
  console.time('bad-array');
  techniques.badArrayOps(transactions);
  console.timeEnd('bad-array');

  console.time('good-array');
  techniques.goodArrayOps(transactions);
  console.timeEnd('good-array');
}

benchmarkOptimizations();
```

**4. V8 Optimization Hints:**

```typescript
// Functions that stay "hot" get optimized by V8

// ✅ GOOD: Monomorphic function (always same type)
function processMonomorphic(obj: { id: string; amount: number }) {
  return obj.id + obj.amount; // V8 can optimize this
}

// ❌ BAD: Polymorphic function (different types)
function processPolymorphic(obj: any) {
  return obj.id + obj.amount; // V8 can't optimize well
}

// ✅ GOOD: Consistent object shape
class Transaction {
  constructor(
    public id: string,
    public amount: number,
    public type: string
  ) {}
}

// ❌ BAD: Inconsistent object shapes
const txn1 = { id: '1', amount: 100 }; // Shape 1
const txn2 = { amount: 200, id: '2' }; // Shape 2 (different order!)
const txn3 = { id: '3', amount: 300, type: 'credit' }; // Shape 3

// ✅ GOOD: Use arrays for numeric keys
const fastArray = [1, 2, 3, 4, 5]; // V8 optimizes as packed array

// ❌ BAD: Sparse arrays
const slowArray: any[] = [];
slowArray[0] = 1;
slowArray[1000] = 2; // Creates holes, V8 can't optimize

// ✅ GOOD: Preallocate arrays
const preallocated = new Array(10000);
for (let i = 0; i < 10000; i++) {
  preallocated[i] = i; // No resizing needed
}

// ❌ BAD: Growing arrays
const growing: number[] = [];
for (let i = 0; i < 10000; i++) {
  growing.push(i); // Multiple reallocations
}
```

**Key Takeaways for ENBD:**
- Use Node.js Inspector for CPU profiling (`--inspect` flag)
- Measure performance with `performance.now()` and Performance Observer
- Identify hot paths - functions called most frequently
- Optimize string operations - use arrays and join instead of concatenation
- Cache regex patterns - don't recreate in loops
- Use object pooling for frequently created/destroyed objects
- Avoid blocking the event loop - use async batching
- Single-pass array operations instead of chaining map/filter/reduce
- Keep object shapes consistent for V8 optimization
- Use monomorphic functions (consistent types)
- Preallocate arrays when size is known
- Avoid sparse arrays and holes
- Profile in production with minimal overhead (`--cpu-prof` flag)
- Focus on 80/20 rule - optimize the 20% causing 80% of CPU time
- Compare before/after with realistic data volumes

---

*Questions will be added here...*

---

## Section 5: Scalability & Architecture Patterns

### Questions 41-50

*Questions will be added here...*

---

**Status:** 🔄 In Progress - Adding questions in batches of 10
