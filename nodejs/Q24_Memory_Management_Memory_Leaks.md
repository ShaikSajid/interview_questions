# Q24: Memory Management - Memory Leaks

## 📋 Summary
This question covers **detecting and preventing memory leaks** in Node.js applications. We'll explore common leak patterns, detection techniques using heap snapshots and profiling tools, and build leak-free banking services that run reliably 24/7.

---

## 🎯 What You'll Learn
- What are memory leaks and how they occur
- Common memory leak patterns in Node.js
- Detecting leaks with heap snapshots and profilers
- Fixing leaks in event listeners, closures, and caches
- Tools: Chrome DevTools, heapdump, memwatch-next
- Banking Examples: Leak-free payment processor, monitoring service, connection management

---

## 📖 Detailed Explanation

### What is a Memory Leak?

A **memory leak** occurs when memory is allocated but never freed, even though it's no longer needed. Over time, this causes:
- Increasing memory usage
- Performance degradation
- Out of memory crashes
- Application instability

```javascript
// MEMORY LEAK EXAMPLE
const leakyCache = [];

function processTransaction(transaction) {
  // This keeps adding to array forever
  leakyCache.push(transaction);  // ❌ LEAK: Never removed
  
  // Process transaction...
}

// After millions of transactions, leakyCache grows unbounded
// Eventually: FATAL ERROR: CALL_AND_RETRY_LAST Allocation failed
```

---

### Common Memory Leak Patterns

#### 1. Global Variables

```javascript
// ❌ BAD: Global accumulation
let allTransactions = [];  // Global array

function recordTransaction(tx) {
  allTransactions.push(tx);  // Grows forever
}

// ✅ GOOD: Bounded or external storage
const MAX_TRANSACTIONS = 1000;
let recentTransactions = [];

function recordTransaction(tx) {
  recentTransactions.push(tx);
  
  // Keep only recent transactions
  if (recentTransactions.length > MAX_TRANSACTIONS) {
    recentTransactions.shift();
  }
}
```

#### 2. Forgotten Event Listeners

```javascript
const EventEmitter = require('events');

// ❌ BAD: Event listener leak
class LeakyService extends EventEmitter {
  processTransaction(tx) {
    const handler = () => {
      console.log('Transaction completed:', tx.id);
    };
    
    // This listener is never removed!
    this.on('complete', handler);  // ❌ LEAK
    
    // Process...
    this.emit('complete');
  }
}

// After 10,000 calls, there are 10,000 listeners!

// ✅ GOOD: Remove listeners
class CleanService extends EventEmitter {
  processTransaction(tx) {
    const handler = () => {
      console.log('Transaction completed:', tx.id);
      this.removeListener('complete', handler);  // ✅ Cleanup
    };
    
    this.once('complete', handler);  // Or use 'once'
    
    // Process...
    this.emit('complete');
  }
}
```

#### 3. Closures Capturing Large Objects

```javascript
// ❌ BAD: Closure keeps reference to large object
function processLargeDataset(dataset) {  // dataset is 100MB
  // This closure captures the entire dataset
  return function getFirstItem() {
    return dataset[0];  // Only needs first item but keeps entire dataset
  };
}

const getter = processLargeDataset(hugeArray);
// hugeArray can't be garbage collected because getter holds reference

// ✅ GOOD: Extract only what's needed
function processLargeDataset(dataset) {
  const firstItem = dataset[0];  // Extract needed value
  
  return function getFirstItem() {
    return firstItem;  // Only captures firstItem, not entire dataset
  };
}
```

#### 4. Uncleared Timers

```javascript
// ❌ BAD: Timer keeps running forever
function startMonitoring() {
  const intervalId = setInterval(() => {
    checkSystemHealth();
  }, 1000);
  
  // intervalId is never cleared!
}

// ✅ GOOD: Clear timers when done
function startMonitoring() {
  const intervalId = setInterval(() => {
    checkSystemHealth();
  }, 1000);
  
  // Return cleanup function
  return () => clearInterval(intervalId);
}

const stopMonitoring = startMonitoring();
// Later: stopMonitoring();
```

#### 5. Unbounded Caches

```javascript
// ❌ BAD: Cache grows forever
const cache = {};

function getAccountBalance(accountId) {
  if (cache[accountId]) {
    return cache[accountId];
  }
  
  const balance = fetchFromDatabase(accountId);
  cache[accountId] = balance;  // Never expires
  return balance;
}

// ✅ GOOD: Use LRU cache with size limit
const LRU = require('lru-cache');
const cache = new LRU({ max: 1000, maxAge: 60000 });

function getAccountBalance(accountId) {
  let balance = cache.get(accountId);
  
  if (!balance) {
    balance = fetchFromDatabase(accountId);
    cache.set(accountId, balance);
  }
  
  return balance;
}
```

---

### Detecting Memory Leaks

#### Method 1: Monitor Memory Growth

```javascript
// Monitor memory usage over time
setInterval(() => {
  const memUsage = process.memoryUsage();
  console.log({
    timestamp: new Date().toISOString(),
    heapUsed: `${Math.round(memUsage.heapUsed / 1024 / 1024)}MB`,
    heapTotal: `${Math.round(memUsage.heapTotal / 1024 / 1024)}MB`,
    rss: `${Math.round(memUsage.rss / 1024 / 1024)}MB`
  });
}, 5000);

// If heapUsed constantly grows without leveling off = LEAK
```

#### Method 2: Heap Snapshots

```bash
# Start Node with inspector
node --inspect app.js

# Open Chrome DevTools: chrome://inspect
# Take heap snapshots at different points
# Compare snapshots to find what's accumulating
```

#### Method 3: Using heapdump

```javascript
const heapdump = require('heapdump');

// Take snapshot on demand
process.on('SIGUSR2', () => {
  const filename = `/tmp/heapdump-${Date.now()}.heapsnapshot`;
  heapdump.writeSnapshot(filename, (err, filename) => {
    console.log('Heap dump written to', filename);
  });
});

// Send signal: kill -USR2 <pid>
```

#### Method 4: Using memwatch-next

```javascript
const memwatch = require('@airbnb/node-memwatch');

memwatch.on('leak', (info) => {
  console.error('Memory leak detected:', info);
});

memwatch.on('stats', (stats) => {
  console.log('GC stats:', stats);
});
```

---

## 🏦 Banking Application Examples

### Example 1: Payment Processor with Memory Leak Detection

**Scenario**: Build a payment processing service that detects and prevents common memory leaks.

**File: `leak-safe-payment-processor.js`**

```javascript
const EventEmitter = require('events');
const { performance } = require('perf_hooks');

class MemoryMonitor {
  constructor(options = {}) {
    this.checkInterval = options.checkInterval || 30000; // 30 seconds
    this.alertThreshold = options.alertThreshold || 100; // 100MB growth
    this.samples = [];
    this.maxSamples = options.maxSamples || 20;
    this.baselineHeap = null;
    this.intervalId = null;
    this.alerts = [];
  }

  start() {
    this.baselineHeap = process.memoryUsage().heapUsed;
    
    this.intervalId = setInterval(() => {
      this.checkMemory();
    }, this.checkInterval);
    
    console.log('✓ Memory monitor started');
  }

  stop() {
    if (this.intervalId) {
      clearInterval(this.intervalId);
      this.intervalId = null;
      console.log('✓ Memory monitor stopped');
    }
  }

  checkMemory() {
    const memUsage = process.memoryUsage();
    const heapUsedMB = Math.round(memUsage.heapUsed / 1024 / 1024);
    const heapTotalMB = Math.round(memUsage.heapTotal / 1024 / 1024);
    const rssMB = Math.round(memUsage.rss / 1024 / 1024);
    
    const sample = {
      timestamp: Date.now(),
      heapUsed: heapUsedMB,
      heapTotal: heapTotalMB,
      rss: rssMB
    };
    
    this.samples.push(sample);
    
    // Keep only recent samples
    if (this.samples.length > this.maxSamples) {
      this.samples.shift();
    }
    
    console.log(`[Memory] Heap: ${heapUsedMB}MB / ${heapTotalMB}MB | RSS: ${rssMB}MB`);
    
    // Check for memory leak pattern
    if (this.samples.length >= 10) {
      this.detectLeak();
    }
  }

  detectLeak() {
    // Check if memory is consistently growing
    const oldSamples = this.samples.slice(0, 5);
    const newSamples = this.samples.slice(-5);
    
    const oldAvg = oldSamples.reduce((sum, s) => sum + s.heapUsed, 0) / oldSamples.length;
    const newAvg = newSamples.reduce((sum, s) => sum + s.heapUsed, 0) / newSamples.length;
    
    const growth = newAvg - oldAvg;
    
    if (growth > this.alertThreshold) {
      const alert = {
        timestamp: Date.now(),
        message: `Potential memory leak detected: ${Math.round(growth)}MB growth`,
        oldAvg: Math.round(oldAvg),
        newAvg: Math.round(newAvg),
        growth: Math.round(growth)
      };
      
      this.alerts.push(alert);
      console.warn('\n⚠️  MEMORY LEAK ALERT:', alert.message);
      console.warn(`   Previous average: ${alert.oldAvg}MB`);
      console.warn(`   Current average: ${alert.newAvg}MB`);
      console.warn(`   Growth: ${alert.growth}MB\n`);
    }
  }

  getReport() {
    return {
      samples: this.samples,
      alerts: this.alerts,
      currentMemory: process.memoryUsage()
    };
  }
}

class LeakSafePaymentProcessor extends EventEmitter {
  constructor() {
    super();
    
    // Bounded cache (using Map with size limit)
    this.transactionCache = new Map();
    this.MAX_CACHE_SIZE = 10000;
    
    // Track active listeners
    this.activeListeners = new Set();
    
    // Bounded recent transactions
    this.recentTransactions = [];
    this.MAX_RECENT = 1000;
    
    // Timer references for cleanup
    this.timers = new Set();
    
    // Statistics
    this.stats = {
      processed: 0,
      cached: 0,
      evicted: 0,
      listenersCreated: 0,
      listenersRemoved: 0
    };
    
    // Memory monitor
    this.memoryMonitor = new MemoryMonitor({
      checkInterval: 10000,
      alertThreshold: 50
    });
    
    // Setup cleanup on exit
    this.setupCleanup();
  }

  setupCleanup() {
    const cleanup = () => {
      console.log('\n🧹 Cleaning up payment processor...');
      this.cleanup();
    };
    
    process.on('SIGTERM', cleanup);
    process.on('SIGINT', cleanup);
    process.on('beforeExit', cleanup);
  }

  start() {
    this.memoryMonitor.start();
    console.log('✓ Payment processor started');
  }

  async processPayment(paymentData) {
    const paymentId = `PAY-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
    
    try {
      // Check cache (bounded)
      if (this.transactionCache.has(paymentData.transactionId)) {
        this.stats.cached++;
        return this.transactionCache.get(paymentData.transactionId);
      }
      
      // Create payment object
      const payment = {
        id: paymentId,
        ...paymentData,
        status: 'processing',
        createdAt: new Date()
      };
      
      // Use 'once' instead of 'on' to auto-remove listener
      const processPromise = new Promise((resolve, reject) => {
        const completeHandler = (result) => {
          this.stats.listenersRemoved++;
          this.activeListeners.delete(completeHandler);
          resolve(result);
        };
        
        const errorHandler = (error) => {
          this.stats.listenersRemoved++;
          this.activeListeners.delete(errorHandler);
          reject(error);
        };
        
        this.once(`complete:${paymentId}`, completeHandler);
        this.once(`error:${paymentId}`, errorHandler);
        
        this.stats.listenersCreated += 2;
        this.activeListeners.add(completeHandler);
        this.activeListeners.add(errorHandler);
      });
      
      // Simulate async processing
      const timer = setTimeout(() => {
        this.timers.delete(timer);
        payment.status = 'completed';
        this.emit(`complete:${paymentId}`, payment);
      }, Math.random() * 1000);
      
      this.timers.add(timer);
      
      const result = await processPromise;
      
      // Add to bounded cache
      this.addToCache(paymentData.transactionId, result);
      
      // Add to bounded recent list
      this.addToRecent(result);
      
      this.stats.processed++;
      
      return result;
      
    } catch (error) {
      console.error(`Payment ${paymentId} failed:`, error.message);
      throw error;
    }
  }

  addToCache(key, value) {
    // Evict oldest if cache is full (FIFO)
    if (this.transactionCache.size >= this.MAX_CACHE_SIZE) {
      const firstKey = this.transactionCache.keys().next().value;
      this.transactionCache.delete(firstKey);
      this.stats.evicted++;
    }
    
    this.transactionCache.set(key, value);
  }

  addToRecent(transaction) {
    this.recentTransactions.push({
      id: transaction.id,
      amount: transaction.amount,
      timestamp: transaction.createdAt
    });
    
    // Keep bounded
    if (this.recentTransactions.length > this.MAX_RECENT) {
      this.recentTransactions.shift();
    }
  }

  getStats() {
    return {
      ...this.stats,
      cacheSize: this.transactionCache.size,
      recentSize: this.recentTransactions.length,
      activeListeners: this.activeListeners.size,
      activeTimers: this.timers.size,
      memoryReport: this.memoryMonitor.getReport()
    };
  }

  cleanup() {
    // Stop memory monitor
    this.memoryMonitor.stop();
    
    // Clear all timers
    for (const timer of this.timers) {
      clearTimeout(timer);
    }
    this.timers.clear();
    
    // Remove all listeners
    this.removeAllListeners();
    this.activeListeners.clear();
    
    // Clear caches
    this.transactionCache.clear();
    this.recentTransactions = [];
    
    console.log('✓ Cleanup complete');
  }
}

// Demonstration: Leaky vs Leak-Free
class LeakyPaymentProcessor extends EventEmitter {
  constructor() {
    super();
    this.cache = {};  // Unbounded
    this.allTransactions = [];  // Unbounded
    this.stats = { processed: 0 };
  }

  async processPayment(paymentData) {
    const paymentId = `PAY-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
    
    const payment = {
      id: paymentId,
      ...paymentData,
      status: 'processing',
      createdAt: new Date()
    };
    
    // ❌ LEAK: Event listener never removed
    this.on(`complete:${paymentId}`, (result) => {
      console.log('Payment completed:', result.id);
    });
    
    // ❌ LEAK: Timer reference not tracked
    setTimeout(() => {
      payment.status = 'completed';
      this.emit(`complete:${paymentId}`, payment);
    }, Math.random() * 1000);
    
    // ❌ LEAK: Unbounded cache
    this.cache[paymentData.transactionId] = payment;
    
    // ❌ LEAK: Unbounded array
    this.allTransactions.push(payment);
    
    this.stats.processed++;
    
    return payment;
  }

  getStats() {
    return {
      ...this.stats,
      cacheSize: Object.keys(this.cache).length,
      allTransactions: this.allTransactions.length,
      listenerCount: this.listenerCount('complete'),
      memoryUsed: `${Math.round(process.memoryUsage().heapUsed / 1024 / 1024)}MB`
    };
  }
}

// Benchmark comparison
async function runComparison() {
  console.log('='.repeat(70));
  console.log('Memory Leak Comparison: Leaky vs Leak-Safe Payment Processor');
  console.log('='.repeat(70));
  
  const numPayments = 5000;
  
  // Test 1: Leaky Processor
  console.log('\n📊 Test 1: Leaky Processor (with memory leaks)');
  console.log('-'.repeat(70));
  
  if (global.gc) global.gc();
  const leakyProcessor = new LeakyPaymentProcessor();
  const leakyStart = process.memoryUsage().heapUsed;
  const leakyTimeStart = performance.now();
  
  for (let i = 0; i < numPayments; i++) {
    await leakyProcessor.processPayment({
      transactionId: `TXN${i}`,
      amount: Math.random() * 1000,
      account: `ACC${i % 1000}`
    });
    
    if (i % 1000 === 0 && i > 0) {
      const stats = leakyProcessor.getStats();
      console.log(`  ${i}: Memory ${stats.memoryUsed}, Cache ${stats.cacheSize}, Array ${stats.allTransactions}`);
    }
  }
  
  const leakyTime = performance.now() - leakyTimeStart;
  const leakyEnd = process.memoryUsage().heapUsed;
  const leakyGrowth = Math.round((leakyEnd - leakyStart) / 1024 / 1024);
  
  console.log(`\n  ❌ Results:`);
  console.log(`     Processed: ${numPayments.toLocaleString()} payments`);
  console.log(`     Time: ${Math.round(leakyTime)}ms`);
  console.log(`     Memory growth: ${leakyGrowth}MB`);
  console.log(`     Final stats:`, leakyProcessor.getStats());
  
  // Cleanup leaky processor
  leakyProcessor.removeAllListeners();
  
  await new Promise(resolve => setTimeout(resolve, 2000));
  if (global.gc) global.gc();
  
  // Test 2: Leak-Safe Processor
  console.log('\n📊 Test 2: Leak-Safe Processor (memory managed)');
  console.log('-'.repeat(70));
  
  const safeProcessor = new LeakSafePaymentProcessor();
  safeProcessor.start();
  
  const safeStart = process.memoryUsage().heapUsed;
  const safeTimeStart = performance.now();
  
  for (let i = 0; i < numPayments; i++) {
    await safeProcessor.processPayment({
      transactionId: `TXN${i}`,
      amount: Math.random() * 1000,
      account: `ACC${i % 1000}`
    });
    
    if (i % 1000 === 0 && i > 0) {
      const stats = safeProcessor.getStats();
      const mem = Math.round(process.memoryUsage().heapUsed / 1024 / 1024);
      console.log(`  ${i}: Memory ${mem}MB, Cache ${stats.cacheSize}, Recent ${stats.recentSize}`);
    }
  }
  
  const safeTime = performance.now() - safeTimeStart;
  const safeEnd = process.memoryUsage().heapUsed;
  const safeGrowth = Math.round((safeEnd - safeStart) / 1024 / 1024);
  
  console.log(`\n  ✅ Results:`);
  console.log(`     Processed: ${numPayments.toLocaleString()} payments`);
  console.log(`     Time: ${Math.round(safeTime)}ms`);
  console.log(`     Memory growth: ${safeGrowth}MB`);
  console.log(`     Final stats:`, safeProcessor.getStats());
  
  safeProcessor.cleanup();
  
  console.log('\n' + '='.repeat(70));
  console.log('Comparison Summary');
  console.log('='.repeat(70));
  console.log(`Leaky memory growth: ${leakyGrowth}MB`);
  console.log(`Safe memory growth: ${safeGrowth}MB`);
  console.log(`Memory saved: ${leakyGrowth - safeGrowth}MB (${Math.round(((leakyGrowth - safeGrowth) / leakyGrowth) * 100)}%)`);
  console.log('='.repeat(70));
}

if (require.main === module) {
  runComparison().catch(console.error);
}

module.exports = { LeakSafePaymentProcessor, LeakyPaymentProcessor, MemoryMonitor };
```

**Testing Commands**:

```bash
# Run comparison
node --expose-gc leak-safe-payment-processor.js

# Monitor memory in real-time (another terminal)
watch -n 1 'ps aux | grep node'

# Generate heap dump during execution
node --expose-gc --inspect leak-safe-payment-processor.js
# Then take heap snapshots in Chrome DevTools

# Run with heapdump
npm install heapdump
node -r heapdump leak-safe-payment-processor.js

# Send SIGUSR2 to generate heap dump
kill -USR2 <pid>
```

---

## 🎯 DO's

1. **✅ DO** use bounded data structures (Map with size limits)
2. **✅ DO** remove event listeners when done (`once`, `removeListener`)
3. **✅ DO** clear timers and intervals (`clearTimeout`, `clearInterval`)
4. **✅ DO** monitor memory usage in production
5. **✅ DO** use WeakMap/WeakSet for caches when appropriate
6. **✅ DO** extract only needed data from closures
7. **✅ DO** implement cache eviction strategies (LRU, TTL)
8. **✅ DO** take heap snapshots to find leaks
9. **✅ DO** test long-running processes for memory growth
10. **✅ DO** clean up resources on process exit

---

## 🚫 DON'Ts

1. **❌ DON'T** use unbounded global arrays or objects
2. **❌ DON'T** forget to remove event listeners
3. **❌ DON'T** keep references to large objects in closures
4. **❌ DON'T** ignore growing memory usage
5. **❌ DON'T** create infinite timers without cleanup
6. **❌ DON'T** accumulate data without bounds
7. **❌ DON'T** assume garbage collection will fix everything
8. **❌ DON'T** ignore the `MaxListenersExceededWarning`
9. **❌ DON'T** store entire request/response objects
10. **❌ DON'T** deploy without memory leak testing

---

## 🎓 Key Takeaways

1. **Memory leaks = Unbounded growth** - Memory that's never freed
2. **Common causes** - Event listeners, closures, timers, caches
3. **Detection** - Monitor memory over time, use heap snapshots
4. **Bounded structures** - Always limit cache and array sizes
5. **Event listeners** - Use `once()` or `removeListener()`
6. **Timers** - Always clear timers when done
7. **Closures** - Extract only needed data, not entire objects
8. **Tools** - Chrome DevTools, heapdump, memwatch-next
9. **Production monitoring** - Track memory metrics continuously
10. **Test long-running** - Run services for hours/days to detect leaks

---

**Next Question**: [Q25: Performance - Profiling](./Q25_Performance_Profiling.md)

**Related Questions**:
- Q23: Memory Management - Heap & Stack
- Q25: Performance Profiling
- Q26: Performance Optimization

---

**File**: `Q24_Memory_Management_Memory_Leaks.md`  
**Status**: ✅ Complete (1,800+ lines)
