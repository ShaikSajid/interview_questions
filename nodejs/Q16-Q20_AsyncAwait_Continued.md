# Node.js Questions 16-20: Async/Await Error Handling & Advanced Patterns

---

### Q16. What is an unhandled promise rejection and how do you prevent them in production?

**Answer:**

**Unhandled Promise Rejection** occurs when a promise is rejected but has no `.catch()` handler or is not wrapped in try-catch with async/await.

**Why Critical for Banking:**
- Can crash Node.js applications (since Node.js 15+)
- Data inconsistency if transactions fail silently
- Compliance issues with untracked errors

**Detection and Prevention:**

```javascript
// BAD: Unhandled rejection
async function badExample() {
  const result = await riskyOperation(); // If this throws, app crashes!
  return result;
}

// GOOD: Proper handling
async function goodExample() {
  try {
    const result = await riskyOperation();
    return result;
  } catch (error) {
    console.error('Operation failed:', error);
    // Handle or rethrow
    throw error;
  }
}

// Global handler for unhandled rejections
process.on('unhandledRejection', (reason, promise) => {
  console.error('🚨 Unhandled Rejection at:', promise);
  console.error('Reason:', reason);
  
  // Log to monitoring system
  logToMonitoring({
    type: 'UNHANDLED_REJECTION',
    reason: reason,
    stack: reason.stack,
    timestamp: new Date()
  });
  
  // In production, might want to gracefully shutdown
  // process.exit(1);
});

// Global handler for uncaught exceptions
process.on('uncaughtException', (error) => {
  console.error('🚨 Uncaught Exception:', error);
  
  logToMonitoring({
    type: 'UNCAUGHT_EXCEPTION',
    error: error.message,
    stack: error.stack,
    timestamp: new Date()
  });
  
  // Graceful shutdown
  process.exit(1);
});

function riskyOperation() {
  return Promise.resolve('success');
}

function logToMonitoring(data) {
  console.log('Logged to monitoring:', data);
}
```

**Banking Application Example:**

```javascript
const express = require('express');
const app = express();

// Middleware to catch async errors
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

// Transaction endpoint
app.post('/api/transfer', asyncHandler(async (req, res) => {
  const { fromAccount, toAccount, amount } = req.body;
  
  // This will be caught by asyncHandler
  const result = await processTransfer(fromAccount, toAccount, amount);
  
  res.json({ success: true, result });
}));

// Centralized error handler
app.use((err, req, res, next) => {
  console.error('Error:', err);
  
  // Log to monitoring
  logError(err, req);
  
  res.status(err.statusCode || 500).json({
    success: false,
    error: err.message
  });
});

// Track all promises in application
class PromiseTracker {
  constructor() {
    this.activePromises = new Set();
  }
  
  track(promise, context) {
    const tracked = {
      promise,
      context,
      createdAt: Date.now(),
      stack: new Error().stack
    };
    
    this.activePromises.add(tracked);
    
    promise
      .then(() => this.activePromises.delete(tracked))
      .catch((error) => {
        this.activePromises.delete(tracked);
        console.error('Tracked promise rejected:', context, error);
      });
    
    return promise;
  }
  
  getActivePromises() {
    return Array.from(this.activePromises).map(p => ({
      context: p.context,
      age: Date.now() - p.createdAt
    }));
  }
}

const tracker = new PromiseTracker();

// Usage
async function criticalOperation() {
  return tracker.track(
    performCriticalTask(),
    'Critical banking operation'
  );
}

function processTransfer(from, to, amount) {
  return Promise.resolve({ id: 'TXN-123' });
}

function logError(err, req) {
  console.error('Logged error:', err.message);
}

function performCriticalTask() {
  return Promise.resolve('done');
}
```

**Best Practices:**
1. Always use try-catch with async/await
2. Add `.catch()` to all promises
3. Implement global handlers
4. Use wrapper functions for Express routes
5. Monitor and alert on unhandled rejections
6. Never ignore errors silently

---

### Q17. Explain the microtask queue vs macrotask queue. How does this affect banking transaction processing?

**Answer:**

**JavaScript Event Loop Queues:**

1. **Microtask Queue** (Higher Priority)
   - Promise callbacks (.then, .catch, .finally)
   - process.nextTick()
   - queueMicrotask()
   - MutationObserver

2. **Macrotask Queue** (Lower Priority)
   - setTimeout, setInterval
   - setImmediate (Node.js)
   - I/O operations
   - UI rendering (browsers)

**Execution Order:**
```
1. Execute all synchronous code
2. Execute ALL microtasks (until queue is empty)
3. Execute ONE macrotask
4. Execute ALL microtasks again
5. Repeat steps 3-4
```

**Example:**

```javascript
console.log('1: Sync start');

setTimeout(() => {
  console.log('2: setTimeout (macrotask)');
}, 0);

Promise.resolve().then(() => {
  console.log('3: Promise 1 (microtask)');
}).then(() => {
  console.log('4: Promise 2 (microtask)');
});

process.nextTick(() => {
  console.log('5: nextTick (microtask - highest priority)');
});

queueMicrotask(() => {
  console.log('6: queueMicrotask (microtask)');
});

console.log('7: Sync end');

// Output:
// 1: Sync start
// 7: Sync end
// 5: nextTick (microtask - highest priority)
// 3: Promise 1 (microtask)
// 6: queueMicrotask (microtask)
// 4: Promise 2 (microtask)
// 2: setTimeout (macrotask)
```

**Banking Transaction Scenario:**

```javascript
class TransactionProcessor {
  constructor() {
    this.transactionQueue = [];
    this.validationQueue = [];
  }
  
  async processTransaction(txn) {
    console.log('📝 Transaction received:', txn.id);
    
    // Immediate validation (microtask - high priority)
    process.nextTick(() => {
      console.log('✓ Immediate security check:', txn.id);
      this.validationQueue.push(txn);
    });
    
    // Promise-based validation (microtask)
    Promise.resolve().then(() => {
      console.log('✓ Promise validation:', txn.id);
      return this.validateAmount(txn);
    }).then(() => {
      console.log('✓ Fraud check:', txn.id);
      return this.checkFraud(txn);
    });
    
    // Deferred processing (macrotask - lower priority)
    setTimeout(() => {
      console.log('📊 Analytics update (macrotask):', txn.id);
      this.updateAnalytics(txn);
    }, 0);
    
    // Deferred notification (macrotask)
    setImmediate(() => {
      console.log('📧 Send notification (setImmediate):', txn.id);
      this.sendNotification(txn);
    });
    
    console.log('📝 Transaction queued:', txn.id);
  }
  
  validateAmount(txn) {
    return Promise.resolve(true);
  }
  
  checkFraud(txn) {
    return Promise.resolve(true);
  }
  
  updateAnalytics(txn) {
    console.log('Analytics updated');
  }
  
  sendNotification(txn) {
    console.log('Notification sent');
  }
}

// Test
const processor = new TransactionProcessor();
processor.processTransaction({ id: 'TXN-001', amount: 1000 });

// Output shows execution order based on queue priorities
```

**Real-World Banking Use Case:**

```javascript
class BankingEventLoop {
  async processPayment(payment) {
    console.log('=== Processing Payment ===');
    
    // 1. Synchronous validation (executes first)
    if (!payment.amount || payment.amount <= 0) {
      throw new Error('Invalid amount');
    }
    console.log('1️⃣ Sync: Amount validated');
    
    // 2. Critical validations (microtasks - execute before any macrotasks)
    process.nextTick(() => {
      console.log('2️⃣ NextTick: AML check');
      this.checkAML(payment);
    });
    
    Promise.resolve().then(() => {
      console.log('3️⃣ Microtask: Account status check');
      return this.checkAccountStatus(payment.accountId);
    });
    
    queueMicrotask(() => {
      console.log('4️⃣ Microtask: Balance verification');
      this.verifyBalance(payment);
    });
    
    // 3. Non-critical operations (macrotasks - execute after microtasks)
    setTimeout(() => {
      console.log('5️⃣ Macrotask: Update customer activity');
      this.updateActivity(payment);
    }, 0);
    
    setTimeout(() => {
      console.log('6️⃣ Macrotask: Generate receipt');
      this.generateReceipt(payment);
    }, 0);
    
    setImmediate(() => {
      console.log('7️⃣ SetImmediate: Send push notification');
      this.sendPushNotification(payment);
    });
    
    console.log('8️⃣ Sync: Payment queued for processing');
  }
  
  checkAML(payment) {}
  checkAccountStatus(accountId) { return Promise.resolve(true); }
  verifyBalance(payment) {}
  updateActivity(payment) {}
  generateReceipt(payment) {}
  sendPushNotification(payment) {}
}

const banking = new BankingEventLoop();
banking.processPayment({ accountId: 'ACC-001', amount: 5000 });
```

**Performance Implications:**

```javascript
// PROBLEM: Starving the event loop with microtasks
function badMicrotaskLoop() {
  // This will block macrotasks forever!
  function recursiveMicrotask() {
    Promise.resolve().then(() => {
      console.log('Microtask running...');
      recursiveMicrotask(); // Never lets macrotasks run
    });
  }
  
  recursiveMicrotask();
  
  // This timeout will NEVER execute!
  setTimeout(() => {
    console.log('This will never print!');
  }, 0);
}

// SOLUTION: Use setImmediate or setTimeout for recursive tasks
function goodTaskLoop() {
  function recursiveMacrotask(count = 0) {
    if (count > 5) return;
    
    console.log('Macrotask:', count);
    
    // Allows other tasks to run
    setImmediate(() => {
      recursiveMacrotask(count + 1);
    });
  }
  
  recursiveMacrotask();
  
  // This will execute between recursive calls
  setTimeout(() => {
    console.log('Other tasks can run!');
  }, 0);
}

goodTaskLoop();
```

**ENBD Best Practices:**
1. Use microtasks (Promises) for critical operations
2. Use macrotasks (setTimeout) for non-critical operations
3. Avoid recursive microtasks that starve the event loop
4. Use process.nextTick() for highest priority validations
5. Profile task queue performance in production

---

### Q18. How do you implement retry logic with exponential backoff for banking APIs?

**Answer:**

**Exponential Backoff** increases delay between retries exponentially to avoid overwhelming failing services.

**Formula:** `delay = initialDelay * (multiplier ^ attemptNumber) + randomJitter`

**Complete Implementation:**

```javascript
class RetryHandler {
  constructor(options = {}) {
    this.maxRetries = options.maxRetries || 3;
    this.initialDelay = options.initialDelay || 1000; // 1 second
    this.maxDelay = options.maxDelay || 30000; // 30 seconds
    this.multiplier = options.multiplier || 2;
    this.jitter = options.jitter !== false; // true by default
    this.retryableErrors = options.retryableErrors || [
      'ECONNRESET',
      'ETIMEDOUT',
      'ENOTFOUND',
      'NETWORK_ERROR',
      'SERVICE_UNAVAILABLE'
    ];
  }
  
  async execute(operation, context = {}) {
    let lastError;
    
    for (let attempt = 0; attempt <= this.maxRetries; attempt++) {
      try {
        console.log(`Attempt ${attempt + 1}/${this.maxRetries + 1}: ${context.operation || 'Operation'}`);
        
        const result = await operation();
        
        if (attempt > 0) {
          console.log(`✓ Success after ${attempt} retries`);
        }
        
        return result;
        
      } catch (error) {
        lastError = error;
        
        // Check if error is retryable
        if (!this.isRetryable(error)) {
          console.error('✗ Non-retryable error:', error.message);
          throw error;
        }
        
        // Check if we've exhausted retries
        if (attempt === this.maxRetries) {
          console.error(`✗ Max retries (${this.maxRetries}) exceeded`);
          throw new Error(`Operation failed after ${this.maxRetries + 1} attempts: ${error.message}`);
        }
        
        // Calculate delay with exponential backoff
        const delay = this.calculateDelay(attempt);
        console.warn(`⚠️ Retry ${attempt + 1} failed: ${error.message}. Retrying in ${delay}ms...`);
        
        // Wait before retry
        await this.sleep(delay);
      }
    }
    
    throw lastError;
  }
  
  calculateDelay(attempt) {
    // Exponential backoff: delay = initialDelay * (multiplier ^ attempt)
    let delay = this.initialDelay * Math.pow(this.multiplier, attempt);
    
    // Cap at maxDelay
    delay = Math.min(delay, this.maxDelay);
    
    // Add jitter to prevent thundering herd
    if (this.jitter) {
      const jitterAmount = delay * 0.1; // 10% jitter
      delay += Math.random() * jitterAmount - jitterAmount / 2;
    }
    
    return Math.floor(delay);
  }
  
  isRetryable(error) {
    // Check error code
    if (error.code && this.retryableErrors.includes(error.code)) {
      return true;
    }
    
    // Check HTTP status codes (5xx are retryable)
    if (error.statusCode >= 500 && error.statusCode < 600) {
      return true;
    }
    
    // Check for specific error messages
    if (error.message) {
      const retryableMessages = [
        'timeout',
        'ECONNREFUSED',
        'socket hang up',
        'network error'
      ];
      
      return retryableMessages.some(msg => 
        error.message.toLowerCase().includes(msg.toLowerCase())
      );
    }
    
    return false;
  }
  
  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Banking API Client with Retry
class BankingAPIClient {
  constructor() {
    this.retryHandler = new RetryHandler({
      maxRetries: 3,
      initialDelay: 1000,
      maxDelay: 10000,
      multiplier: 2,
      jitter: true
    });
  }
  
  async transferMoney(fromAccount, toAccount, amount) {
    return this.retryHandler.execute(
      async () => {
        // Simulate API call
        return await this.makeTransferRequest(fromAccount, toAccount, amount);
      },
      { operation: `Transfer ${amount} from ${fromAccount} to ${toAccount}` }
    );
  }
  
  async makeTransferRequest(fromAccount, toAccount, amount) {
    // Simulate random failures
    const random = Math.random();
    
    if (random < 0.3) {
      const error = new Error('Network timeout');
      error.code = 'ETIMEDOUT';
      throw error;
    }
    
    if (random < 0.5) {
      const error = new Error('Service temporarily unavailable');
      error.statusCode = 503;
      throw error;
    }
    
    // Success
    return {
      transactionId: 'TXN-' + Date.now(),
      status: 'completed',
      amount
    };
  }
  
  async getAccountBalance(accountId) {
    return this.retryHandler.execute(
      async () => {
        return await this.makeBalanceRequest(accountId);
      },
      { operation: `Get balance for ${accountId}` }
    );
  }
  
  async makeBalanceRequest(accountId) {
    // Simulate failure
    if (Math.random() < 0.4) {
      throw new Error('Database connection timeout');
    }
    
    return { accountId, balance: 10000 };
  }
}

// Advanced: Circuit Breaker Pattern with Retry
class CircuitBreaker {
  constructor(options = {}) {
    this.failureThreshold = options.failureThreshold || 5;
    this.successThreshold = options.successThreshold || 2;
    this.timeout = options.timeout || 60000; // 1 minute
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.failureCount = 0;
    this.successCount = 0;
    this.nextAttempt = Date.now();
  }
  
  async execute(operation) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN. Service unavailable.');
      }
      
      // Transition to HALF_OPEN
      this.state = 'HALF_OPEN';
      console.log('🔶 Circuit breaker: HALF_OPEN - Testing service...');
    }
    
    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  onSuccess() {
    this.failureCount = 0;
    
    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      
      if (this.successCount >= this.successThreshold) {
        this.state = 'CLOSED';
        this.successCount = 0;
        console.log('✅ Circuit breaker: CLOSED - Service restored');
      }
    }
  }
  
  onFailure() {
    this.failureCount++;
    this.successCount = 0;
    
    if (this.failureCount >= this.failureThreshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.timeout;
      console.error(`🔴 Circuit breaker: OPEN - Too many failures (${this.failureCount})`);
    }
  }
  
  getState() {
    return {
      state: this.state,
      failureCount: this.failureCount,
      successCount: this.successCount,
      nextAttempt: new Date(this.nextAttempt)
    };
  }
}

// Combined: Retry + Circuit Breaker
class ResilientAPIClient {
  constructor() {
    this.retryHandler = new RetryHandler({
      maxRetries: 3,
      initialDelay: 1000,
      multiplier: 2
    });
    
    this.circuitBreaker = new CircuitBreaker({
      failureThreshold: 5,
      timeout: 30000
    });
  }
  
  async callAPI(endpoint, data) {
    return this.circuitBreaker.execute(async () => {
      return this.retryHandler.execute(async () => {
        return await this.makeRequest(endpoint, data);
      });
    });
  }
  
  async makeRequest(endpoint, data) {
    console.log(`API Call: ${endpoint}`);
    
    // Simulate failures
    if (Math.random() < 0.3) {
      throw new Error('Network error');
    }
    
    return { success: true, data };
  }
}

// Usage Example
async function demonstrateRetry() {
  const client = new BankingAPIClient();
  
  try {
    console.log('\n=== Transfer Money with Retry ===');
    const result = await client.transferMoney('ACC-001', 'ACC-002', 5000);
    console.log('✓ Transfer successful:', result);
  } catch (error) {
    console.error('✗ Transfer failed:', error.message);
  }
  
  try {
    console.log('\n=== Get Balance with Retry ===');
    const balance = await client.getAccountBalance('ACC-001');
    console.log('✓ Balance retrieved:', balance);
  } catch (error) {
    console.error('✗ Balance retrieval failed:', error.message);
  }
}

// demonstrateRetry();

// Test Circuit Breaker
async function testCircuitBreaker() {
  const resilient = new ResilientAPIClient();
  
  console.log('\n=== Testing Circuit Breaker ===');
  
  for (let i = 0; i < 10; i++) {
    try {
      await resilient.callAPI('/api/transfer', { amount: 1000 });
      console.log(`Request ${i + 1}: Success`);
    } catch (error) {
      console.error(`Request ${i + 1}: Failed -`, error.message);
    }
    
    console.log('Circuit State:', resilient.circuitBreaker.getState());
    await new Promise(resolve => setTimeout(resolve, 500));
  }
}

// testCircuitBreaker();
```

**Best Practices for ENBD:**
1. Always implement retry for network calls
2. Use exponential backoff to prevent service overload
3. Add jitter to prevent thundering herd
4. Combine with circuit breaker for failing services
5. Log all retry attempts for monitoring
6. Set appropriate timeout limits
7. Don't retry non-idempotent operations without safeguards

---

### Q19. Explain async iterators and generators. How would you use them for processing large datasets in banking?

**Answer:**

**Async Iterators** allow iterating over asynchronous data sources.
**Async Generators** make it easy to create async iterators.

**Syntax:**
- `async function*` - async generator
- `for await...of` - consume async iterators
- `yield` - produce values asynchronously

**Banking Use Cases:**
- Processing large transaction files
- Streaming account data
- Paginated API responses
- Real-time data feeds

**Basic Example:**

```javascript
// Async Generator
async function* numberGenerator() {
  for (let i = 1; i <= 5; i++) {
    await new Promise(resolve => setTimeout(resolve, 1000));
    yield i;
  }
}

// Consume with for await...of
async function consumeNumbers() {
  for await (const num of numberGenerator()) {
    console.log('Number:', num);
  }
}

// consumeNumbers();
```

**Banking Example 1: Process Large Transaction File**

```javascript
const fs = require('fs');
const readline = require('readline');

class TransactionFileProcessor {
  // Async generator to read transactions line by line
  async* readTransactions(filePath) {
    const fileStream = fs.createReadStream(filePath);
    const rl = readline.createInterface({
      input: fileStream,
      crlfDelay: Infinity
    });
    
    let lineNumber = 0;
    
    for await (const line of rl) {
      lineNumber++;
      
      // Skip header
      if (lineNumber === 1) continue;
      
      try {
        const transaction = this.parseLine(line);
        yield transaction;
      } catch (error) {
        console.error(`Error parsing line ${lineNumber}:`, error.message);
      }
    }
  }
  
  parseLine(line) {
    const [id, date, from, to, amount, currency] = line.split(',');
    
    return {
      id: id.trim(),
      date: new Date(date.trim()),
      fromAccount: from.trim(),
      toAccount: to.trim(),
      amount: parseFloat(amount),
      currency: currency.trim()
    };
  }
  
  // Process transactions in batches
  async* batchTransactions(filePath, batchSize = 100) {
    let batch = [];
    
    for await (const transaction of this.readTransactions(filePath)) {
      batch.push(transaction);
      
      if (batch.length >= batchSize) {
        yield batch;
        batch = [];
      }
    }
    
    // Yield remaining transactions
    if (batch.length > 0) {
      yield batch;
    }
  }
  
  // Transform transactions
  async* transformTransactions(filePath) {
    for await (const txn of this.readTransactions(filePath)) {
      // Add computed fields
      const transformed = {
        ...txn,
        fee: this.calculateFee(txn),
        tax: this.calculateTax(txn),
        totalAmount: txn.amount + this.calculateFee(txn) + this.calculateTax(txn),
        processedAt: new Date()
      };
      
      yield transformed;
    }
  }
  
  // Filter transactions
  async* filterLargeTransactions(filePath, threshold = 10000) {
    for await (const txn of this.readTransactions(filePath)) {
      if (txn.amount >= threshold) {
        yield txn;
      }
    }
  }
  
  calculateFee(txn) {
    return txn.amount * 0.01; // 1% fee
  }
  
  calculateTax(txn) {
    return txn.amount * 0.05; // 5% tax
  }
}

// Usage
async function processTransactionFile() {
  const processor = new TransactionFileProcessor();
  
  console.log('=== Processing Transactions ===\n');
  
  // Example 1: Process all transactions
  let count = 0;
  for await (const txn of processor.readTransactions('./transactions.csv')) {
    count++;
    if (count <= 5) {
      console.log(`Transaction ${count}:`, txn);
    }
  }
  console.log(`Total transactions: ${count}\n`);
  
  // Example 2: Process in batches
  console.log('=== Batch Processing ===\n');
  let batchNum = 0;
  for await (const batch of processor.batchTransactions('./transactions.csv', 100)) {
    batchNum++;
    console.log(`Processing batch ${batchNum} with ${batch.length} transactions`);
    await processBatch(batch);
  }
  
  // Example 3: Filter large transactions
  console.log('\n=== Large Transactions (>= 10000) ===\n');
  for await (const txn of processor.filterLargeTransactions('./transactions.csv', 10000)) {
    console.log(`Large transaction: ${txn.id} - Amount: ${txn.amount}`);
  }
}

async function processBatch(batch) {
  // Simulate batch processing
  return new Promise(resolve => setTimeout(resolve, 100));
}

// processTransactionFile().catch(console.error);
```

**Banking Example 2: Paginated API Data**

```javascript
class BankingAPIClient {
  constructor(baseURL) {
    this.baseURL = baseURL;
  }
  
  // Async generator for paginated customer accounts
  async* fetchAllAccounts(customerId) {
    let page = 1;
    let hasMore = true;
    
    while (hasMore) {
      console.log(`Fetching page ${page}...`);
      
      const response = await this.fetchAccountsPage(customerId, page);
      
      // Yield each account
      for (const account of response.data) {
        yield account;
      }
      
      hasMore = response.hasMore;
      page++;
      
      // Rate limiting - pause between requests
      await this.sleep(100);
    }
  }
  
  async fetchAccountsPage(customerId, page) {
    // Simulate API call
    return new Promise(resolve => {
      setTimeout(() => {
        const accounts = Array.from({ length: 10 }, (_, i) => ({
          id: `ACC-${page}-${i}`,
          balance: Math.random() * 100000,
          type: ['savings', 'current', 'investment'][Math.floor(Math.random() * 3)]
        }));
        
        resolve({
          data: accounts,
          hasMore: page < 5, // Only 5 pages
          page
        });
      }, 500);
    });
  }
  
  // Aggregate data from paginated results
  async* fetchTransactionsByDate(accountId, startDate, endDate) {
    let currentDate = new Date(startDate);
    const end = new Date(endDate);
    
    while (currentDate <= end) {
      const transactions = await this.fetchTransactionsForDate(accountId, currentDate);
      
      for (const txn of transactions) {
        yield txn;
      }
      
      // Move to next day
      currentDate.setDate(currentDate.getDate() + 1);
    }
  }
  
  async fetchTransactionsForDate(accountId, date) {
    // Simulate fetching transactions for a specific date
    return new Promise(resolve => {
      setTimeout(() => {
        const count = Math.floor(Math.random() * 10);
        const transactions = Array.from({ length: count }, (_, i) => ({
          id: `TXN-${date.toISOString().split('T')[0]}-${i}`,
          accountId,
          amount: Math.random() * 1000,
          date: date
        }));
        resolve(transactions);
      }, 100);
    });
  }
  
  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Usage
async function fetchAllCustomerAccounts() {
  const api = new BankingAPIClient('https://api.bank.com');
  
  console.log('=== Fetching All Accounts ===\n');
  
  const accounts = [];
  for await (const account of api.fetchAllAccounts('CUST-001')) {
    accounts.push(account);
    console.log(`Account: ${account.id}, Balance: ${account.balance.toFixed(2)}`);
  }
  
  console.log(`\nTotal accounts fetched: ${accounts.length}`);
}

// fetchAllCustomerAccounts().catch(console.error);
```

**Banking Example 3: Real-time Transaction Stream**

```javascript
const { EventEmitter } = require('events');

class TransactionStream extends EventEmitter {
  constructor() {
    super();
    this.isActive = false;
  }
  
  start() {
    this.isActive = true;
    this.simulateTransactions();
  }
  
  stop() {
    this.isActive = false;
  }
  
  simulateTransactions() {
    if (!this.isActive) return;
    
    const transaction = {
      id: `TXN-${Date.now()}`,
      amount: Math.random() * 10000,
      timestamp: new Date()
    };
    
    this.emit('transaction', transaction);
    
    // Next transaction in 500ms - 2s
    setTimeout(() => this.simulateTransactions(), 500 + Math.random() * 1500);
  }
  
  // Convert EventEmitter to async iterator
  async* [Symbol.asyncIterator]() {
    const queue = [];
    let resolve;
    let promise = new Promise(r => resolve = r);
    
    const onTransaction = (txn) => {
      queue.push(txn);
      resolve();
      promise = new Promise(r => resolve = r);
    };
    
    this.on('transaction', onTransaction);
    
    try {
      while (this.isActive) {
        while (queue.length > 0) {
          yield queue.shift();
        }
        await promise;
      }
    } finally {
      this.off('transaction', onTransaction);
    }
  }
}

// Usage
async function monitorTransactions() {
  const stream = new TransactionStream();
  stream.start();
  
  console.log('=== Monitoring Live Transactions ===\n');
  
  let count = 0;
  for await (const txn of stream) {
    count++;
    console.log(`[${count}] Transaction: ${txn.id}, Amount: ${txn.amount.toFixed(2)}`);
    
    // Stop after 10 transactions
    if (count >= 10) {
      stream.stop();
      break;
    }
  }
  
  console.log('\n✓ Monitoring stopped');
}

// monitorTransactions().catch(console.error);
```

**Advanced: Composing Async Generators**

```javascript
class TransactionPipeline {
  // Source: Read transactions
  static async* source(filePath) {
    for (let i = 1; i <= 100; i++) {
      yield {
        id: `TXN-${i}`,
        amount: Math.random() * 10000,
        date: new Date()
      };
    }
  }
  
  // Transform: Add fees
  static async* addFees(source) {
    for await (const txn of source) {
      yield {
        ...txn,
        fee: txn.amount * 0.01
      };
    }
  }
  
  // Filter: Only large transactions
  static async* filterLarge(source, threshold) {
    for await (const txn of source) {
      if (txn.amount >= threshold) {
        yield txn;
      }
    }
  }
  
  // Aggregate: Calculate statistics
  static async aggregate(source) {
    let count = 0;
    let total = 0;
    let max = 0;
    let min = Infinity;
    
    for await (const txn of source) {
      count++;
      total += txn.amount;
      max = Math.max(max, txn.amount);
      min = Math.min(min, txn.amount);
    }
    
    return {
      count,
      total,
      average: total / count,
      max,
      min
    };
  }
}

// Usage: Chain operations
async function runPipeline() {
  console.log('=== Transaction Pipeline ===\n');
  
  const source = TransactionPipeline.source();
  const withFees = TransactionPipeline.addFees(source);
  const largeOnly = TransactionPipeline.filterLarge(withFees, 5000);
  
  const stats = await TransactionPipeline.aggregate(largeOnly);
  
  console.log('Statistics for large transactions (>= 5000):');
  console.log(stats);
}

// runPipeline().catch(console.error);
```

**Key Benefits for ENBD:**
1. Memory efficient - processes data incrementally
2. Handles large datasets without loading all into memory
3. Easy to compose and chain operations
4. Built-in backpressure handling
5. Clean, readable code for complex workflows

---

### Q20. How do you handle concurrent API calls efficiently? Explain Promise pooling and rate limiting.

**Answer:**

**Problem:** Making thousands of concurrent API calls can:
- Overwhelm servers
- Exhaust connection pools
- Trigger rate limits
- Cause memory issues

**Solutions:**
1. **Promise Pooling** - Limit concurrent operations
2. **Rate Limiting** - Control request rate
3. **Batching** - Group requests
4. **Queuing** - Manage backlog

**Implementation:**

```javascript
class PromisePool {
  constructor(concurrency = 5) {
    this.concurrency = concurrency;
    this.running = 0;
    this.queue = [];
  }
  
  async execute(promiseFn) {
    // Wait if at capacity
    while (this.running >= this.concurrency) {
      await new Promise(resolve => this.queue.push(resolve));
    }
    
    this.running++;
    
    try {
      return await promiseFn();
    } finally {
      this.running--;
      
      // Process next in queue
      const resolve = this.queue.shift();
      if (resolve) resolve();
    }
  }
  
  async executeAll(promiseFns) {
    return Promise.all(
      promiseFns.map(fn => this.execute(fn))
    );
  }
}

// Rate Limiter
class RateLimiter {
  constructor(maxRequests, perMilliseconds) {
    this.maxRequests = maxRequests;
    this.perMilliseconds = perMilliseconds;
    this.requests = [];
  }
  
  async acquire() {
    const now = Date.now();
    
    // Remove old requests outside the time window
    this.requests = this.requests.filter(
      time => time > now - this.perMilliseconds
    );
    
    // If at limit, wait
    if (this.requests.length >= this.maxRequests) {
      const oldestRequest = this.requests[0];
      const waitTime = this.perMilliseconds - (now - oldestRequest);
      
      console.log(`Rate limit reached. Waiting ${waitTime}ms...`);
      await new Promise(resolve => setTimeout(resolve, waitTime));
      
      return this.acquire(); // Retry
    }
    
    this.requests.push(now);
  }
  
  async execute(fn) {
    await this.acquire();
    return fn();
  }
}

// Combined: Pool + Rate Limiter
class ManagedAPIClient {
  constructor(options = {}) {
    this.pool = new PromisePool(options.concurrency || 5);
    this.rateLimiter = new RateLimiter(
      options.maxRequests || 10,
      options.perMilliseconds || 1000
    );
  }
  
  async request(url, data) {
    return this.pool.execute(async () => {
      return this.rateLimiter.execute(async () => {
        return this.makeRequest(url, data);
      });
    });
  }
  
  async makeRequest(url, data) {
    console.log(`Making request to ${url}`);
    // Simulate API call
    return new Promise(resolve => {
      setTimeout(() => {
        resolve({ success: true, data });
      }, 100);
    });
  }
}

// Banking Example: Bulk Account Updates
async function updateAccountsInBulk() {
  const client = new ManagedAPIClient({
    concurrency: 10,    // Max 10 concurrent requests
    maxRequests: 50,    // Max 50 requests
    perMilliseconds: 1000 // per 1 second
  });
  
  // 1000 accounts to update
  const accountIds = Array.from({ length: 1000 }, (_, i) => `ACC-${i}`);
  
  console.log('=== Updating 1000 Accounts ===\n');
  const startTime = Date.now();
  
  const promises = accountIds.map(accountId =>
    client.request('/api/accounts/update', { accountId, data: { verified: true } })
  );
  
  const results = await Promise.all(promises);
  
  const duration = Date.now() - startTime;
  console.log(`\n✓ All accounts updated in ${duration}ms`);
  console.log(`Success: ${results.filter(r => r.success).length}/${results.length}`);
}

// updateAccountsInBulk().catch(console.error);
```

**Advanced: p-limit Library Pattern**

```javascript
class PLimitStyle {
  constructor(concurrency) {
    this.concurrency = concurrency;
    this.activeCount = 0;
    this.queue = [];
  }
  
  async run(fn) {
    while (this.activeCount >= this.concurrency) {
      await new Promise(resolve => this.queue.push(resolve));
    }
    
    this.activeCount++;
    
    try {
      const result = await fn();
      return result;
    } finally {
      this.activeCount--;
      const next = this.queue.shift();
      if (next) next();
    }
  }
}

// Usage
async function processWithLimit() {
  const limit = new PLimitStyle(3);
  
  const tasks = Array.from({ length: 10 }, (_, i) => 
    limit.run(async () => {
      console.log(`Task ${i} started`);
      await new Promise(resolve => setTimeout(resolve, 1000));
      console.log(`Task ${i} completed`);
      return i;
    })
  );
  
  const results = await Promise.all(tasks);
  console.log('All tasks completed:', results);
}

// processWithLimit().catch(console.error);
```

**Banking Use Case: Transaction Processing**

```javascript
class BulkTransactionProcessor {
  constructor() {
    this.pool = new PromisePool(20); // 20 concurrent transactions
    this.rateLimiter = new RateLimiter(100, 1000); // 100 per second
  }
  
  async processTransactions(transactions) {
    console.log(`Processing ${transactions.length} transactions...\n`);
    
    const results = {
      successful: [],
      failed: [],
      startTime: Date.now()
    };
    
    await this.pool.executeAll(
      transactions.map(txn => async () => {
        try {
          const result = await this.rateLimiter.execute(async () => {
            return await this.processTransaction(txn);
          });
          
          results.successful.push(result);
          return result;
        } catch (error) {
          results.failed.push({ txn, error: error.message });
          console.error(`Transaction ${txn.id} failed:`, error.message);
        }
      })
    );
    
    results.duration = Date.now() - results.startTime;
    
    return results;
  }
  
  async processTransaction(txn) {
    // Simulate transaction processing
    await new Promise(resolve => setTimeout(resolve, 50 + Math.random() * 100));
    
    // Random failures
    if (Math.random() < 0.05) {
      throw new Error('Transaction failed');
    }
    
    return {
      transactionId: txn.id,
      status: 'completed',
      amount: txn.amount
    };
  }
}

// Test
async function bulkProcessTest() {
  const processor = new BulkTransactionProcessor();
  
  const transactions = Array.from({ length: 500 }, (_, i) => ({
    id: `TXN-${i}`,
    amount: Math.random() * 10000,
    from: `ACC-${i}`,
    to: `ACC-${i + 1}`
  }));
  
  const results = await processor.processTransactions(transactions);
  
  console.log('\n=== Results ===');
  console.log(`Total: ${transactions.length}`);
  console.log(`Successful: ${results.successful.length}`);
  console.log(`Failed: ${results.failed.length}`);
  console.log(`Duration: ${results.duration}ms`);
  console.log(`Rate: ${(transactions.length / results.duration * 1000).toFixed(2)} txn/sec`);
}

// bulkProcessTest().catch(console.error);
```

**Best Practices for ENBD:**
1. Always limit concurrent operations
2. Implement rate limiting for external APIs
3. Monitor and adjust pool size based on performance
4. Handle failures gracefully with retries
5. Log metrics for capacity planning
6. Use queuing for high-volume scenarios
7. Implement circuit breakers for failing endpoints

---

**🎯 Section 2 Complete! Questions 16-20 Finished**

**Topics Covered:**
- Unhandled promise rejections
- Microtask vs macrotask queues
- Retry logic with exponential backoff
- Async iterators and generators
- Promise pooling and rate limiting

**Progress: 20/50 questions completed**
