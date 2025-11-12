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

Type "continue" for the next 5 questions!



---

## Section 3: Streams and Buffers

### Questions 21-30

*Questions will be added here...*

---

## Section 4: Performance Optimization

### Questions 31-40

*Questions will be added here...*

---

## Section 5: Scalability & Architecture Patterns

### Questions 41-50

*Questions will be added here...*

---

**Status:** 🔄 In Progress - Adding questions in batches of 10
