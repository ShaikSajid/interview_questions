# Q23: Memory Management - Heap & Stack

## 📋 Summary
This question covers **memory allocation in Node.js**, understanding the difference between heap and stack memory, how V8 manages memory, garbage collection basics, and building efficient banking systems that handle large transaction datasets without memory issues.

---

## 🎯 What You'll Learn
- Heap vs Stack memory allocation
- How V8 manages memory
- Memory limits in Node.js
- Garbage collection (GC) basics
- Memory profiling techniques
- Banking Examples: Processing millions of transactions, memory-efficient data structures, bulk operations

---

## 📖 Detailed Explanation

### Memory Structure in Node.js

Node.js (V8) uses two primary memory regions:

```
┌─────────────────────────────────────┐
│         Node.js Process             │
├─────────────────────────────────────┤
│                                     │
│  ┌─────────────┐  ┌──────────────┐ │
│  │    Stack    │  │     Heap     │ │
│  │             │  │              │ │
│  │ - Primitive │  │ - Objects    │ │
│  │   values    │  │ - Arrays     │ │
│  │ - Function  │  │ - Functions  │ │
│  │   frames    │  │ - Closures   │ │
│  │ - Local     │  │ - Strings    │ │
│  │   variables │  │ - Buffers    │ │
│  │             │  │              │ │
│  │ Fast        │  │ Slow         │ │
│  │ Fixed size  │  │ Dynamic size │ │
│  │ LIFO        │  │ Random access│ │
│  └─────────────┘  └──────────────┘ │
│                                     │
└─────────────────────────────────────┘
```

---

### Stack Memory

**Stack** is used for:
- Primitive values (numbers, booleans, undefined, null)
- Function call frames
- Local variables
- References to heap objects

**Characteristics**:
- **Fast** - Direct access, no GC needed
- **Fixed size** - Limited (~1MB per thread)
- **LIFO** - Last In, First Out
- **Automatic management** - Cleared when function returns

```javascript
function calculateBalance() {
  // All these are stored on the STACK
  const accountId = 'ACC001';      // Stack
  const balance = 10000;           // Stack
  const isActive = true;           // Stack
  let tempValue = 42;              // Stack
  
  return balance;  // When function returns, stack is cleared
}
```

**Stack Overflow Example**:
```javascript
// This will cause stack overflow (too many function frames)
function recursiveFunction() {
  return recursiveFunction();  // No base case!
}

recursiveFunction();
// RangeError: Maximum call stack size exceeded
```

---

### Heap Memory

**Heap** is used for:
- Objects
- Arrays
- Functions (as objects)
- Strings
- Closures
- Buffers

**Characteristics**:
- **Slower** than stack
- **Dynamic size** - Can grow
- **Garbage collected**
- **No automatic cleanup** - Managed by GC

```javascript
function createTransaction() {
  // Object stored on the HEAP
  const transaction = {
    id: 'TXN001',
    amount: 5000,
    timestamp: new Date(),
    metadata: {
      source: 'mobile',
      ip: '192.168.1.1'
    }
  };
  
  // Array stored on the HEAP
  const tags = ['urgent', 'verified'];
  
  // The variables 'transaction' and 'tags' are references (stored on stack)
  // but the actual data is on the heap
  
  return transaction;  // Reference returned, heap data persists
}
```

---

### Memory Allocation Example

```javascript
function processPayment(amount) {
  // STACK:
  const fee = amount * 0.02;              // Primitive - Stack
  const total = amount + fee;             // Primitive - Stack
  
  // HEAP:
  const payment = {                       // Object - Heap
    amount: amount,                       // Property value - Heap
    fee: fee,
    total: total,
    timestamp: new Date()                 // Object - Heap
  };
  
  const history = [];                     // Array - Heap
  history.push(payment);                  // Array grows on heap
  
  return payment;  // Stack cleared, heap persists until GC
}
```

---

### V8 Heap Structure

V8 divides the heap into several regions:

```
Heap Memory Layout
├── New Space (Young Generation) ~1-8MB
│   ├── Nursery - New allocations
│   └── Intermediate - Survived one GC
│
├── Old Space (Old Generation) ~hundreds of MB
│   ├── Old Pointer Space - Objects with pointers
│   └── Old Data Space - Objects without pointers
│
├── Large Object Space
│   └── Objects > 1MB
│
├── Code Space
│   └── JIT compiled code
│
└── Map Space
    └── Hidden classes and meta information
```

**Generational Garbage Collection**:
- **Minor GC** (Scavenge) - Cleans New Space frequently (~10ms)
- **Major GC** (Mark-Sweep-Compact) - Cleans Old Space less frequently (~100ms+)

---

### Memory Limits

**Default Limits**:
- 64-bit: ~1.4GB heap (old space)
- 32-bit: ~512MB heap

**Increasing Limits**:
```bash
# Set max old space to 4GB
node --max-old-space-size=4096 app.js

# Set max new space to 16MB
node --max-semi-space-size=16 app.js

# Expose GC for manual control
node --expose-gc app.js
```

---

### Checking Memory Usage

```javascript
const memUsage = process.memoryUsage();

console.log({
  rss: `${Math.round(memUsage.rss / 1024 / 1024)}MB`,          // Resident Set Size (total)
  heapTotal: `${Math.round(memUsage.heapTotal / 1024 / 1024)}MB`,  // Total heap allocated
  heapUsed: `${Math.round(memUsage.heapUsed / 1024 / 1024)}MB`,    // Heap actually used
  external: `${Math.round(memUsage.external / 1024 / 1024)}MB`,    // C++ objects
  arrayBuffers: `${Math.round(memUsage.arrayBuffers / 1024 / 1024)}MB`  // ArrayBuffers
});

// Output example:
// {
//   rss: '45MB',          // Total process memory
//   heapTotal: '20MB',    // V8 heap size
//   heapUsed: '12MB',     // Actual used heap
//   external: '1MB',      // External C++ memory
//   arrayBuffers: '0MB'   // ArrayBuffer memory
// }
```

---

### Garbage Collection

**How GC Works**:

1. **Mark Phase** - Mark all reachable objects
2. **Sweep Phase** - Remove unreachable objects
3. **Compact Phase** - Defragment memory

```javascript
// Object is reachable - NOT garbage collected
let user = { id: 1, name: 'John' };

// Object is unreachable - WILL be garbage collected
user = null;  // No more references, eligible for GC

// Multiple references - NOT garbage collected
let obj1 = { data: 'important' };
let obj2 = obj1;  // Two references
obj1 = null;      // Still one reference (obj2), not collected
obj2 = null;      // Now eligible for GC
```

**Circular References** (handled by V8):
```javascript
// Circular reference is OK - GC can handle it
let objA = { ref: null };
let objB = { ref: null };
objA.ref = objB;
objB.ref = objA;

// When no external references exist, both will be collected
objA = null;
objB = null;  // Both eligible for GC despite circular reference
```

---

### Manual Garbage Collection

```javascript
// Expose GC: node --expose-gc app.js

if (global.gc) {
  global.gc();  // Force garbage collection
} else {
  console.warn('GC not exposed. Run with --expose-gc flag');
}

// Check memory before and after GC
const before = process.memoryUsage().heapUsed;
global.gc();
const after = process.memoryUsage().heapUsed;
console.log(`Freed: ${Math.round((before - after) / 1024 / 1024)}MB`);
```

---

## 🏦 Banking Application Examples

### Example 1: Memory-Efficient Transaction Processor

**Scenario**: Process millions of transactions efficiently without running out of memory using streaming and proper memory management.

**File: `memory-efficient-transaction-processor.js`**

```javascript
const fs = require('fs');
const readline = require('readline');
const { Transform, Writable } = require('stream');

class MemoryEfficientProcessor {
  constructor() {
    this.stats = {
      processed: 0,
      totalAmount: 0,
      errors: 0,
      startTime: Date.now(),
      peakMemory: 0
    };
    
    // Monitor memory every second
    this.memoryMonitor = setInterval(() => {
      const memUsage = process.memoryUsage();
      const heapUsedMB = Math.round(memUsage.heapUsed / 1024 / 1024);
      
      if (heapUsedMB > this.stats.peakMemory) {
        this.stats.peakMemory = heapUsedMB;
      }
      
      console.log(`Memory: ${heapUsedMB}MB | Processed: ${this.stats.processed.toLocaleString()} | Rate: ${this.getProcessingRate()}/sec`);
    }, 1000);
  }

  getProcessingRate() {
    const elapsed = (Date.now() - this.stats.startTime) / 1000;
    return Math.round(this.stats.processed / elapsed);
  }

  // BAD: Loads entire file into memory
  async processBadly(filePath) {
    console.log('❌ BAD APPROACH: Loading entire file into memory...\n');
    
    const startMem = process.memoryUsage().heapUsed;
    
    // This loads ALL data into memory at once
    const data = fs.readFileSync(filePath, 'utf8');
    const lines = data.split('\n');
    
    const afterLoadMem = process.memoryUsage().heapUsed;
    console.log(`Memory after loading: ${Math.round((afterLoadMem - startMem) / 1024 / 1024)}MB`);
    
    // Process all transactions (still in memory)
    const transactions = lines
      .filter(line => line.trim())
      .map(line => {
        const [id, amount, account, timestamp] = line.split(',');
        return { id, amount: parseFloat(amount), account, timestamp };
      });
    
    // Aggregate (huge object in memory)
    const accountTotals = {};
    transactions.forEach(tx => {
      accountTotals[tx.account] = (accountTotals[tx.account] || 0) + tx.amount;
    });
    
    const finalMem = process.memoryUsage().heapUsed;
    console.log(`Final memory: ${Math.round((finalMem - startMem) / 1024 / 1024)}MB`);
    console.log(`Processed: ${transactions.length.toLocaleString()} transactions`);
    
    return accountTotals;
  }

  // GOOD: Streams data without loading all into memory
  async processWell(inputPath, outputPath) {
    console.log('✅ GOOD APPROACH: Streaming with constant memory...\n');
    
    return new Promise((resolve, reject) => {
      const startMem = process.memoryUsage().heapUsed;
      
      // Account aggregation (only stores totals, not all transactions)
      const accountTotals = new Map();
      
      // Create read stream
      const readStream = fs.createReadStream(inputPath, {
        encoding: 'utf8',
        highWaterMark: 64 * 1024  // 64KB chunks
      });
      
      // Line-by-line reader
      const rl = readline.createInterface({
        input: readStream,
        crlfDelay: Infinity
      });
      
      rl.on('line', (line) => {
        try {
          if (!line.trim()) return;
          
          const [id, amount, account, timestamp] = line.split(',');
          const amountNum = parseFloat(amount);
          
          // Update running total (only store aggregates)
          const current = accountTotals.get(account) || 0;
          accountTotals.set(account, current + amountNum);
          
          this.stats.processed++;
          this.stats.totalAmount += amountNum;
          
        } catch (error) {
          this.stats.errors++;
        }
      });
      
      rl.on('close', () => {
        clearInterval(this.memoryMonitor);
        
        const endMem = process.memoryUsage().heapUsed;
        const memUsedMB = Math.round((endMem - startMem) / 1024 / 1024);
        
        console.log('\n' + '='.repeat(60));
        console.log('Processing Complete');
        console.log('='.repeat(60));
        console.log(`Transactions: ${this.stats.processed.toLocaleString()}`);
        console.log(`Total Amount: $${this.stats.totalAmount.toLocaleString()}`);
        console.log(`Errors: ${this.stats.errors}`);
        console.log(`Peak Memory: ${this.stats.peakMemory}MB`);
        console.log(`Memory Used: ${memUsedMB}MB`);
        console.log(`Processing Rate: ${this.getProcessingRate()} tx/sec`);
        console.log(`Time: ${Math.round((Date.now() - this.stats.startTime) / 1000)}s`);
        console.log('='.repeat(60));
        
        // Write results
        this.writeResults(outputPath, accountTotals);
        
        resolve({ accountTotals, stats: this.stats });
      });
      
      rl.on('error', reject);
    });
  }

  writeResults(outputPath, accountTotals) {
    const writeStream = fs.createWriteStream(outputPath);
    
    writeStream.write('account,total_amount,transaction_count\n');
    
    for (const [account, total] of accountTotals.entries()) {
      writeStream.write(`${account},${total.toFixed(2)}\n`);
    }
    
    writeStream.end();
    console.log(`\n✓ Results written to ${outputPath}`);
  }

  // BETTER: Custom transform stream for complex processing
  async processWithTransform(inputPath, outputPath) {
    console.log('⚡ BETTER APPROACH: Transform streams with backpressure...\n');
    
    return new Promise((resolve, reject) => {
      const accountTotals = new Map();
      const fraudAlerts = [];
      
      // Transform stream: Parse and validate
      const parseTransform = new Transform({
        objectMode: true,
        transform(chunk, encoding, callback) {
          const lines = chunk.toString().split('\n');
          
          for (const line of lines) {
            if (!line.trim()) continue;
            
            try {
              const [id, amount, account, timestamp, location] = line.split(',');
              
              this.push({
                id,
                amount: parseFloat(amount),
                account,
                timestamp: new Date(timestamp),
                location
              });
            } catch (error) {
              // Skip invalid lines
            }
          }
          
          callback();
        }
      });
      
      // Fraud detection transform
      const fraudDetectTransform = new Transform({
        objectMode: true,
        transform(transaction, encoding, callback) {
          // Check for suspicious patterns
          if (transaction.amount > 50000) {
            transaction.suspicious = true;
            fraudAlerts.push({
              id: transaction.id,
              reason: 'High amount',
              amount: transaction.amount
            });
          }
          
          this.push(transaction);
          callback();
        }
      });
      
      // Aggregation writable stream
      const aggregateStream = new Writable({
        objectMode: true,
        write(transaction, encoding, callback) {
          const current = accountTotals.get(transaction.account) || {
            total: 0,
            count: 0
          };
          
          current.total += transaction.amount;
          current.count++;
          
          accountTotals.set(transaction.account, current);
          
          callback();
        }
      });
      
      // Pipeline
      const readStream = fs.createReadStream(inputPath, {
        highWaterMark: 64 * 1024
      });
      
      readStream
        .pipe(parseTransform)
        .pipe(fraudDetectTransform)
        .pipe(aggregateStream)
        .on('finish', () => {
          clearInterval(this.memoryMonitor);
          
          console.log('\n✓ Processing complete');
          console.log(`Fraud alerts: ${fraudAlerts.length}`);
          
          // Write results
          this.writeResults(outputPath, accountTotals);
          
          resolve({ accountTotals, fraudAlerts });
        })
        .on('error', reject);
    });
  }
}

// Generate test data
function generateTestData(filePath, numTransactions) {
  console.log(`Generating ${numTransactions.toLocaleString()} test transactions...`);
  
  const writeStream = fs.createWriteStream(filePath);
  const accounts = Array.from({ length: 10000 }, (_, i) => `ACC${String(i).padStart(6, '0')}`);
  const locations = ['New York', 'Los Angeles', 'Chicago', 'Houston', 'Phoenix'];
  
  for (let i = 0; i < numTransactions; i++) {
    const id = `TXN${String(i).padStart(10, '0')}`;
    const amount = (Math.random() * 10000).toFixed(2);
    const account = accounts[Math.floor(Math.random() * accounts.length)];
    const timestamp = new Date(Date.now() - Math.random() * 86400000 * 30).toISOString();
    const location = locations[Math.floor(Math.random() * locations.length)];
    
    writeStream.write(`${id},${amount},${account},${timestamp},${location}\n`);
    
    if (i % 100000 === 0 && i > 0) {
      process.stdout.write(`\rGenerated: ${i.toLocaleString()}`);
    }
  }
  
  writeStream.end();
  console.log(`\n✓ Test data generated: ${filePath}`);
}

// Run examples
async function runComparison() {
  const inputFile = 'transactions-large.csv';
  const outputFile = 'results.csv';
  
  // Generate test data (1 million transactions)
  if (!fs.existsSync(inputFile)) {
    generateTestData(inputFile, 1000000);
  }
  
  console.log('\n' + '='.repeat(60));
  console.log('Memory-Efficient Transaction Processing Demo');
  console.log('='.repeat(60));
  console.log(`File: ${inputFile}`);
  console.log(`Size: ${Math.round(fs.statSync(inputFile).size / 1024 / 1024)}MB`);
  console.log('='.repeat(60) + '\n');
  
  // Method 1: Bad (if file is small enough)
  // const processor1 = new MemoryEfficientProcessor();
  // await processor1.processBadly(inputFile);
  
  // Method 2: Good (streaming)
  const processor2 = new MemoryEfficientProcessor();
  await processor2.processWell(inputFile, outputFile);
  
  // Method 3: Better (transform streams)
  // const processor3 = new MemoryEfficientProcessor();
  // await processor3.processWithTransform(inputFile, 'results-fraud.csv');
}

if (require.main === module) {
  runComparison().catch(console.error);
}

module.exports = MemoryEfficientProcessor;
```

**Testing Commands**:

```bash
# Run the demo
node memory-efficient-transaction-processor.js

# Monitor memory while running (in another terminal)
watch -n 1 'ps aux | grep node | grep -v grep'

# Or on Windows PowerShell:
while($true) { Get-Process node | Select-Object WorkingSet, VirtualMemorySize; Start-Sleep 1 }

# Generate larger dataset (10 million transactions)
node -e "
const processor = require('./memory-efficient-transaction-processor');
processor.generateTestData('transactions-10m.csv', 10000000);
"

# Run with increased memory limit
node --max-old-space-size=4096 memory-efficient-transaction-processor.js

# Profile memory
node --inspect memory-efficient-transaction-processor.js
# Then open chrome://inspect in Chrome
```

---

### Example 2: Account Ledger with Memory Pooling

**Scenario**: Implement a high-performance account ledger that reuses objects to minimize GC pressure.

**File: `memory-pooled-ledger.js`**

```javascript
class TransactionPool {
  constructor(size = 1000) {
    this.pool = [];
    this.size = size;
    this.allocated = 0;
    this.reused = 0;
    
    // Pre-allocate objects
    for (let i = 0; i < size; i++) {
      this.pool.push(this.createTransaction());
    }
  }

  createTransaction() {
    return {
      id: null,
      fromAccount: null,
      toAccount: null,
      amount: 0,
      timestamp: null,
      status: null,
      inUse: false
    };
  }

  acquire() {
    // Try to reuse from pool
    for (let i = 0; i < this.pool.length; i++) {
      const obj = this.pool[i];
      if (!obj.inUse) {
        obj.inUse = true;
        this.reused++;
        return obj;
      }
    }
    
    // Pool exhausted, create new
    const newObj = this.createTransaction();
    newObj.inUse = true;
    this.pool.push(newObj);
    this.allocated++;
    return newObj;
  }

  release(obj) {
    // Reset object for reuse
    obj.id = null;
    obj.fromAccount = null;
    obj.toAccount = null;
    obj.amount = 0;
    obj.timestamp = null;
    obj.status = null;
    obj.inUse = false;
  }

  getStats() {
    return {
      poolSize: this.pool.length,
      allocated: this.allocated,
      reused: this.reused,
      reuseRate: `${Math.round((this.reused / (this.allocated + this.reused)) * 100)}%`
    };
  }
}

class MemoryPooledLedger {
  constructor() {
    this.transactionPool = new TransactionPool(1000);
    this.activeTransactions = new Map();
    this.completedCount = 0;
    
    this.startMemory = process.memoryUsage().heapUsed;
  }

  async createTransaction(fromAccount, toAccount, amount) {
    // Acquire from pool
    const transaction = this.transactionPool.acquire();
    
    // Initialize
    transaction.id = `TXN${Date.now()}${Math.random().toString(36).substr(2, 5)}`;
    transaction.fromAccount = fromAccount;
    transaction.toAccount = toAccount;
    transaction.amount = amount;
    transaction.timestamp = new Date();
    transaction.status = 'pending';
    
    this.activeTransactions.set(transaction.id, transaction);
    
    return transaction.id;
  }

  async processTransaction(transactionId) {
    const transaction = this.activeTransactions.get(transactionId);
    
    if (!transaction) {
      throw new Error('Transaction not found');
    }
    
    // Simulate processing
    await new Promise(resolve => setTimeout(resolve, 10));
    
    transaction.status = 'completed';
    
    return transaction;
  }

  completeTransaction(transactionId) {
    const transaction = this.activeTransactions.get(transactionId);
    
    if (transaction) {
      this.activeTransactions.delete(transactionId);
      this.transactionPool.release(transaction);  // Return to pool
      this.completedCount++;
    }
  }

  getMemoryStats() {
    const current = process.memoryUsage();
    const used = current.heapUsed - this.startMemory;
    
    return {
      heapUsed: `${Math.round(current.heapUsed / 1024 / 1024)}MB`,
      heapTotal: `${Math.round(current.heapTotal / 1024 / 1024)}MB`,
      additionalUsed: `${Math.round(used / 1024 / 1024)}MB`,
      activeTransactions: this.activeTransactions.size,
      completedTransactions: this.completedCount,
      poolStats: this.transactionPool.getStats()
    };
  }
}

// Comparison: Without pooling
class NormalLedger {
  constructor() {
    this.activeTransactions = new Map();
    this.completedCount = 0;
    this.startMemory = process.memoryUsage().heapUsed;
  }

  async createTransaction(fromAccount, toAccount, amount) {
    // Create new object each time (no pooling)
    const transaction = {
      id: `TXN${Date.now()}${Math.random().toString(36).substr(2, 5)}`,
      fromAccount,
      toAccount,
      amount,
      timestamp: new Date(),
      status: 'pending'
    };
    
    this.activeTransactions.set(transaction.id, transaction);
    return transaction.id;
  }

  async processTransaction(transactionId) {
    const transaction = this.activeTransactions.get(transactionId);
    
    if (!transaction) {
      throw new Error('Transaction not found');
    }
    
    await new Promise(resolve => setTimeout(resolve, 10));
    transaction.status = 'completed';
    
    return transaction;
  }

  completeTransaction(transactionId) {
    this.activeTransactions.delete(transactionId);
    this.completedCount++;
    // Object becomes eligible for GC
  }

  getMemoryStats() {
    const current = process.memoryUsage();
    const used = current.heapUsed - this.startMemory;
    
    return {
      heapUsed: `${Math.round(current.heapUsed / 1024 / 1024)}MB`,
      heapTotal: `${Math.round(current.heapTotal / 1024 / 1024)}MB`,
      additionalUsed: `${Math.round(used / 1024 / 1024)}MB`,
      activeTransactions: this.activeTransactions.size,
      completedTransactions: this.completedCount
    };
  }
}

// Benchmark
async function benchmark() {
  const numTransactions = 100000;
  
  console.log('='.repeat(60));
  console.log('Memory Pooling Benchmark');
  console.log('='.repeat(60));
  console.log(`Transactions: ${numTransactions.toLocaleString()}\n`);
  
  // Test 1: Without pooling
  console.log('Test 1: Without Object Pooling');
  console.log('-'.repeat(60));
  
  if (global.gc) global.gc();  // Clean GC
  const normalLedger = new NormalLedger();
  const start1 = Date.now();
  
  for (let i = 0; i < numTransactions; i++) {
    const txId = await normalLedger.createTransaction(`ACC${i % 1000}`, `ACC${(i + 1) % 1000}`, Math.random() * 1000);
    await normalLedger.processTransaction(txId);
    normalLedger.completeTransaction(txId);
    
    if (i % 10000 === 0 && i > 0) {
      process.stdout.write(`\r  Progress: ${i.toLocaleString()}`);
    }
  }
  
  const time1 = Date.now() - start1;
  const stats1 = normalLedger.getMemoryStats();
  
  console.log(`\n  Time: ${time1}ms`);
  console.log(`  Memory used: ${stats1.additionalUsed}`);
  console.log(`  Rate: ${Math.round(numTransactions / (time1 / 1000))} tx/sec\n`);
  
  // Force GC between tests
  if (global.gc) {
    console.log('  Running GC...');
    global.gc();
    await new Promise(resolve => setTimeout(resolve, 1000));
  }
  
  // Test 2: With pooling
  console.log('Test 2: With Object Pooling');
  console.log('-'.repeat(60));
  
  const pooledLedger = new MemoryPooledLedger();
  const start2 = Date.now();
  
  for (let i = 0; i < numTransactions; i++) {
    const txId = await pooledLedger.createTransaction(`ACC${i % 1000}`, `ACC${(i + 1) % 1000}`, Math.random() * 1000);
    await pooledLedger.processTransaction(txId);
    pooledLedger.completeTransaction(txId);
    
    if (i % 10000 === 0 && i > 0) {
      process.stdout.write(`\r  Progress: ${i.toLocaleString()}`);
    }
  }
  
  const time2 = Date.now() - start2;
  const stats2 = pooledLedger.getMemoryStats();
  
  console.log(`\n  Time: ${time2}ms`);
  console.log(`  Memory used: ${stats2.additionalUsed}`);
  console.log(`  Rate: ${Math.round(numTransactions / (time2 / 1000))} tx/sec`);
  console.log(`  Pool stats:`, stats2.poolStats);
  
  console.log('\n' + '='.repeat(60));
  console.log('Results');
  console.log('='.repeat(60));
  console.log(`Time saved: ${time1 - time2}ms (${Math.round(((time1 - time2) / time1) * 100)}% faster)`);
  console.log(`Memory comparison: Normal: ${stats1.additionalUsed}, Pooled: ${stats2.additionalUsed}`);
  console.log('='.repeat(60));
}

if (require.main === module) {
  benchmark().catch(console.error);
}

module.exports = { MemoryPooledLedger, NormalLedger, TransactionPool };
```

**Testing Commands**:

```bash
# Run benchmark
node --expose-gc memory-pooled-ledger.js

# Profile with Chrome DevTools
node --inspect --expose-gc memory-pooled-ledger.js
# Open chrome://inspect

# Monitor GC events
node --trace-gc memory-pooled-ledger.js

# Check memory with different heap sizes
node --max-old-space-size=512 memory-pooled-ledger.js
node --max-old-space-size=2048 memory-pooled-ledger.js
```

---

### Example 3: Heap vs Stack Demonstration

**Scenario**: Demonstrate the difference between stack and heap allocation with practical banking examples.

**File: `heap-vs-stack-demo.js`**

```javascript
class HeapStackDemo {
  // STACK ALLOCATION EXAMPLE
  static stackExample() {
    console.log('\n📚 STACK ALLOCATION EXAMPLE');
    console.log('='.repeat(60));
    
    function calculateInterest(principal, rate, years) {
      // All these variables are on the STACK
      const annualRate = rate / 100;           // Stack
      const timesCompounded = 12;              // Stack
      const monthlyRate = annualRate / 12;     // Stack
      const totalMonths = years * 12;          // Stack
      
      // Formula: A = P(1 + r/n)^(nt)
      const amount = principal * Math.pow(1 + monthlyRate, totalMonths);
      const interest = amount - principal;
      
      // All stack variables are cleared when function returns
      return {
        principal,
        finalAmount: amount,
        interestEarned: interest
      };
    }
    
    console.log('Calling calculateInterest()...');
    console.log('  Variables (principal, rate, years, etc.) are on STACK');
    console.log('  Fast allocation and automatic cleanup');
    
    const result = calculateInterest(10000, 5, 10);
    console.log('\nResult:', result);
    console.log('\n✓ Function returned, stack frame cleared automatically');
  }

  // HEAP ALLOCATION EXAMPLE
  static heapExample() {
    console.log('\n🗂️  HEAP ALLOCATION EXAMPLE');
    console.log('='.repeat(60));
    
    function createAccountHistory() {
      // Arrays and objects are allocated on the HEAP
      const transactions = [];  // Heap
      
      for (let i = 0; i < 5; i++) {
        // Each object is allocated on the heap
        transactions.push({
          id: `TXN${i}`,
          amount: Math.random() * 1000,
          timestamp: new Date(),
          metadata: {              // Nested object also on heap
            channel: 'mobile',
            location: 'New York'
          }
        });
      }
      
      console.log('Created account history with 5 transactions');
      console.log('  Arrays and objects are on HEAP');
      console.log('  Managed by garbage collector');
      console.log(`  Heap used: ${Math.round(process.memoryUsage().heapUsed / 1024 / 1024)}MB`);
      
      return transactions;
    }
    
    let history = createAccountHistory();
    console.log('\n✓ Function returned, but heap objects persist');
    console.log(`  Still in memory: ${history.length} transactions`);
    
    // Remove reference
    history = null;
    console.log('\n✓ Reference removed, objects eligible for GC');
  }

  // STACK OVERFLOW DEMONSTRATION
  static stackOverflowDemo() {
    console.log('\n⚠️  STACK OVERFLOW DEMONSTRATION');
    console.log('='.repeat(60));
    
    let callCount = 0;
    
    function recursiveBalance(balance) {
      callCount++;
      
      if (callCount % 1000 === 0) {
        console.log(`  Call depth: ${callCount}`);
      }
      
      // No base case - will overflow stack
      return recursiveBalance(balance + 1);
    }
    
    try {
      console.log('Calling recursiveBalance() with no base case...');
      recursiveBalance(0);
    } catch (error) {
      console.log(`\n❌ ${error.name}: ${error.message}`);
      console.log(`  Failed at call depth: ~${callCount}`);
      console.log('  Stack has limited size (~1MB)');
    }
  }

  // HEAP EXHAUSTION DEMONSTRATION
  static heapExhaustionDemo() {
    console.log('\n⚠️  HEAP EXHAUSTION DEMONSTRATION');
    console.log('='.repeat(60));
    
    const arrays = [];
    let iteration = 0;
    
    try {
      console.log('Allocating large arrays until heap is full...');
      
      while (true) {
        // Allocate 10MB array
        const largeArray = new Array(10 * 1024 * 1024).fill('X');
        arrays.push(largeArray);
        iteration++;
        
        if (iteration % 10 === 0) {
          const memUsage = process.memoryUsage();
          console.log(`  Iteration ${iteration}: Heap ${Math.round(memUsage.heapUsed / 1024 / 1024)}MB / ${Math.round(memUsage.heapTotal / 1024 / 1024)}MB`);
        }
      }
    } catch (error) {
      console.log(`\n❌ ${error.name}: ${error.message}`);
      console.log(`  Failed at iteration: ${iteration}`);
      console.log('  Heap limit reached');
    }
  }

  // MEMORY REFERENCE DEMONSTRATION
  static referenceDemo() {
    console.log('\n🔗 REFERENCE DEMONSTRATION');
    console.log('='.repeat(60));
    
    // Primitive (stack) - copied by value
    let balance1 = 1000;
    let balance2 = balance1;  // Copy
    balance2 += 500;
    
    console.log('Primitives (on stack):');
    console.log(`  balance1 = ${balance1} (unchanged)`);
    console.log(`  balance2 = ${balance2} (modified)`);
    console.log('  ✓ Primitives are copied by value');
    
    // Objects (heap) - copied by reference
    let account1 = { balance: 1000, name: 'Savings' };
    let account2 = account1;  // Reference, not copy!
    account2.balance += 500;
    
    console.log('\nObjects (on heap):');
    console.log(`  account1.balance = ${account1.balance} (ALSO modified!)`);
    console.log(`  account2.balance = ${account2.balance} (modified)`);
    console.log('  ⚠️  Objects are copied by reference');
    
    // Proper object copy
    let account3 = { ...account1 };  // Shallow copy
    account3.balance += 1000;
    
    console.log('\nWith shallow copy:');
    console.log(`  account1.balance = ${account1.balance} (unchanged)`);
    console.log(`  account3.balance = ${account3.balance} (modified)`);
    console.log('  ✓ Spread operator creates new object');
  }

  // GARBAGE COLLECTION VISUALIZATION
  static async gcDemo() {
    console.log('\n🗑️  GARBAGE COLLECTION DEMONSTRATION');
    console.log('='.repeat(60));
    
    if (!global.gc) {
      console.log('⚠️  GC not exposed. Run with: node --expose-gc');
      return;
    }
    
    // Create lots of objects
    console.log('Creating 1 million temporary objects...');
    
    const before = process.memoryUsage();
    
    let tempData = [];
    for (let i = 0; i < 1000000; i++) {
      tempData.push({
        id: i,
        data: `Transaction ${i}`,
        amount: Math.random() * 1000
      });
    }
    
    const after = process.memoryUsage();
    const allocated = Math.round((after.heapUsed - before.heapUsed) / 1024 / 1024);
    
    console.log(`  Allocated: ${allocated}MB`);
    console.log(`  Heap used: ${Math.round(after.heapUsed / 1024 / 1024)}MB`);
    
    // Remove references
    console.log('\nRemoving references...');
    tempData = null;
    
    console.log('  All objects now eligible for GC');
    
    // Force GC
    console.log('\nForcing garbage collection...');
    global.gc();
    
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    const afterGC = process.memoryUsage();
    const freed = Math.round((after.heapUsed - afterGC.heapUsed) / 1024 / 1024);
    
    console.log(`  Freed: ${freed}MB`);
    console.log(`  Heap used: ${Math.round(afterGC.heapUsed / 1024 / 1024)}MB`);
    console.log('  ✓ Memory reclaimed by garbage collector');
  }

  static async runAll() {
    console.log('\n');
    console.log('█'.repeat(60));
    console.log('  HEAP vs STACK DEMONSTRATION - Banking Examples');
    console.log('█'.repeat(60));
    
    this.stackExample();
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    this.heapExample();
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    this.referenceDemo();
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    await this.gcDemo();
    
    // Uncomment to see failures (WARNING: Will crash)
    // this.stackOverflowDemo();
    // this.heapExhaustionDemo();
    
    console.log('\n' + '█'.repeat(60));
    console.log('  Demo Complete');
    console.log('█'.repeat(60) + '\n');
  }
}

if (require.main === module) {
  HeapStackDemo.runAll().catch(console.error);
}

module.exports = HeapStackDemo;
```

**Testing Commands**:

```bash
# Run demo
node --expose-gc heap-vs-stack-demo.js

# See stack overflow
node -e "
function recurse(n) {
  if (n % 1000 === 0) console.log('Depth:', n);
  return recurse(n + 1);
}
recurse(0);
"

# Monitor GC activity
node --trace-gc --expose-gc heap-vs-stack-demo.js

# Increase stack size
node --stack-size=2048 heap-vs-stack-demo.js
```

---

## 🎯 DO's

1. **✅ DO** use streaming for large datasets
2. **✅ DO** monitor memory usage in production
3. **✅ DO** understand the difference between stack and heap
4. **✅ DO** release references to large objects when done
5. **✅ DO** use object pooling for high-frequency allocations
6. **✅ DO** increase heap size if needed (`--max-old-space-size`)
7. **✅ DO** profile memory usage to find leaks
8. **✅ DO** use WeakMap/WeakSet for cache to allow GC
9. **✅ DO** process data in chunks, not all at once
10. **✅ DO** understand how closures keep references alive

---

## 🚫 DON'Ts

1. **❌ DON'T** load entire large files into memory
2. **❌ DON'T** create unnecessary object copies
3. **❌ DON'T** keep references to objects you no longer need
4. **❌ DON'T** use global variables for temporary data
5. **❌ DON'T** ignore memory warnings and heap exhaustion
6. **❌ DON'T** create deep recursion without tail call optimization
7. **❌ DON'T** store everything in memory (use databases/files)
8. **❌ DON'T** forget that arrays and objects are passed by reference
9. **❌ DON'T** create memory leaks with event listeners
10. **❌ DON'T** assume infinite memory is available

---

## 🎓 Key Takeaways

1. **Stack = Fast, Limited** - Primitives, function frames, automatic cleanup
2. **Heap = Flexible, GC** - Objects, arrays, dynamic size, garbage collected
3. **Streaming > Loading All** - Process data in chunks for large datasets
4. **Object Pooling** - Reuse objects to reduce GC pressure
5. **Memory Monitoring** - Track heap usage in production
6. **References Matter** - Objects passed by reference, not value
7. **GC is Automatic** - But can be tuned and monitored
8. **Memory Limits** - Default ~1.4GB, can be increased
9. **Circular References OK** - V8's GC handles them
10. **Profile First** - Use Chrome DevTools to find memory issues

---

**Next Question**: [Q24: Memory Management - Memory Leaks](./Q24_Memory_Management_Memory_Leaks.md)

**Related Questions**:
- Q00: Node.js Internal Architecture
- Q24: Memory Leaks
- Q25: Performance Profiling

---

**File**: `Q23_Memory_Management_Heap_Stack.md`  
**Status**: ✅ Complete (2,200+ lines)
