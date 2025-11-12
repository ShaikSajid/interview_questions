# Node.js Questions 26-30: Performance Optimization

---

### Q26. How do you profile and optimize Node.js application performance?

**Answer:**

**Profiling Tools:**
1. Node.js built-in profiler
2. Chrome DevTools
3. Clinic.js
4. 0x (flamegraph)
5. New Relic / DataDog

**Implementation:**

```javascript
// 1. Built-in Performance Hooks
const { PerformanceObserver, performance } = require('perf_hooks');

class PerformanceMonitor {
  constructor() {
    this.metrics = [];
    this.setupObserver();
  }
  
  setupObserver() {
    const obs = new PerformanceObserver((items) => {
      items.getEntries().forEach((entry) => {
        console.log(`${entry.name}: ${entry.duration.toFixed(2)}ms`);
        this.metrics.push(entry);
      });
    });
    
    obs.observe({ entryTypes: ['measure', 'function'] });
  }
  
  measureOperation(name, fn) {
    return async (...args) => {
      performance.mark(`${name}-start`);
      
      try {
        const result = await fn(...args);
        performance.mark(`${name}-end`);
        performance.measure(name, `${name}-start`, `${name}-end`);
        return result;
      } catch (error) {
        performance.mark(`${name}-end`);
        performance.measure(name, `${name}-start`, `${name}-end`);
        throw error;
      }
    };
  }
  
  getMetrics() {
    return {
      total: this.metrics.length,
      averageDuration: this.metrics.reduce((sum, m) => sum + m.duration, 0) / this.metrics.length,
      slowest: this.metrics.sort((a, b) => b.duration - a.duration).slice(0, 5)
    };
  }
}

// Usage
const monitor = new PerformanceMonitor();

const processTransaction = monitor.measureOperation(
  'processTransaction',
  async (txn) => {
    await new Promise(resolve => setTimeout(resolve, 100));
    return { ...txn, processed: true };
  }
);

// 2. Memory Profiling
class MemoryProfiler {
  constructor() {
    this.snapshots = [];
    this.startTime = Date.now();
  }
  
  takeSnapshot(label) {
    const usage = process.memoryUsage();
    const snapshot = {
      label,
      timestamp: Date.now() - this.startTime,
      heapUsed: usage.heapUsed / 1024 / 1024, // MB
      heapTotal: usage.heapTotal / 1024 / 1024,
      external: usage.external / 1024 / 1024,
      rss: usage.rss / 1024 / 1024,
      arrayBuffers: usage.arrayBuffers / 1024 / 1024
    };
    
    this.snapshots.push(snapshot);
    return snapshot;
  }
  
  getReport() {
    return {
      snapshots: this.snapshots,
      growth: {
        heapUsed: this.snapshots[this.snapshots.length - 1].heapUsed - this.snapshots[0].heapUsed,
        rss: this.snapshots[this.snapshots.length - 1].rss - this.snapshots[0].rss
      }
    };
  }
}

// 3. CPU Profiling
function profileCPU(duration = 10000) {
  const profiler = require('v8-profiler-next');
  profiler.startProfiling('CPU Profile');
  
  setTimeout(() => {
    const profile = profiler.stopProfiling();
    profile.export((error, result) => {
      if (error) {
        console.error('Profiling error:', error);
        return;
      }
      
      require('fs').writeFileSync('./cpu-profile.cpuprofile', result);
      console.log('CPU profile saved');
      profile.delete();
    });
  }, duration);
}

// Banking Application Optimization Example
class OptimizedBankingService {
  constructor() {
    this.cache = new Map();
    this.batchQueue = [];
    this.batchTimer = null;
  }
  
  // Optimization 1: Caching
  async getAccountBalance(accountId) {
    // Check cache first
    if (this.cache.has(accountId)) {
      const cached = this.cache.get(accountId);
      if (Date.now() - cached.timestamp < 60000) { // 1 min TTL
        return cached.balance;
      }
    }
    
    // Fetch from database
    const balance = await this.fetchBalanceFromDB(accountId);
    
    // Cache result
    this.cache.set(accountId, {
      balance,
      timestamp: Date.now()
    });
    
    return balance;
  }
  
  // Optimization 2: Batching
  async processTransaction(transaction) {
    return new Promise((resolve, reject) => {
      this.batchQueue.push({ transaction, resolve, reject });
      
      if (!this.batchTimer) {
        this.batchTimer = setTimeout(() => this.flushBatch(), 100);
      }
    });
  }
  
  async flushBatch() {
    const batch = this.batchQueue.splice(0);
    this.batchTimer = null;
    
    try {
      // Process all transactions in one database call
      const results = await this.processBatchInDB(
        batch.map(b => b.transaction)
      );
      
      batch.forEach((item, index) => {
        item.resolve(results[index]);
      });
    } catch (error) {
      batch.forEach(item => item.reject(error));
    }
  }
  
  // Optimization 3: Connection pooling
  async fetchBalanceFromDB(accountId) {
    // Simulate DB query
    return new Promise(resolve => {
      setTimeout(() => resolve(Math.random() * 100000), 50);
    });
  }
  
  async processBatchInDB(transactions) {
    // Simulate batch processing
    return new Promise(resolve => {
      setTimeout(() => {
        resolve(transactions.map(t => ({ ...t, status: 'completed' })));
      }, 100);
    });
  }
}
```

### Q27. What are clustering and load balancing in Node.js? How do you implement them?

**Answer:**

**Clustering** creates multiple Node.js processes to utilize all CPU cores.

**Implementation:**

```javascript
const cluster = require('cluster');
const os = require('os');
const http = require('http');

if (cluster.isMaster) {
  const numCPUs = os.cpus().length;
  console.log(`Master ${process.pid} is running`);
  console.log(`Starting ${numCPUs} workers...`);
  
  // Fork workers
  for (let i = 0; i < numCPUs; i++) {
    const worker = cluster.fork();
    
    worker.on('message', (msg) => {
      if (msg.cmd === 'notifyRequest') {
        console.log(`Worker ${worker.process.pid} processed request`);
      }
    });
  }
  
  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died. Starting new worker...`);
    cluster.fork();
  });
  
  // Graceful shutdown
  process.on('SIGTERM', () => {
    console.log('Master received SIGTERM, shutting down workers...');
    
    for (const id in cluster.workers) {
      cluster.workers[id].send({ cmd: 'shutdown' });
    }
    
    setTimeout(() => {
      console.log('Force killing remaining workers');
      for (const id in cluster.workers) {
        cluster.workers[id].kill();
      }
      process.exit(0);
    }, 10000);
  });
  
} else {
  // Worker process
  const server = http.createServer((req, res) => {
    // Simulate work
    const start = Date.now();
    while (Date.now() - start < 100) {} // CPU work
    
    res.writeHead(200);
    res.end(`Handled by worker ${process.pid}\n`);
    
    process.send({ cmd: 'notifyRequest' });
  });
  
  server.listen(3000, () => {
    console.log(`Worker ${process.pid} started`);
  });
  
  process.on('message', (msg) => {
    if (msg.cmd === 'shutdown') {
      console.log(`Worker ${process.pid} shutting down gracefully`);
      server.close(() => {
        process.exit(0);
      });
    }
  });
}

// Banking Application with Clustering
class ClusteredBankingApp {
  start() {
    if (cluster.isMaster) {
      this.startMaster();
    } else {
      this.startWorker();
    }
  }
  
  startMaster() {
    const numWorkers = process.env.NODE_ENV === 'production'
      ? os.cpus().length
      : 2;
    
    console.log(`Starting banking app with ${numWorkers} workers`);
    
    // Create workers
    for (let i = 0; i < numWorkers; i++) {
      this.forkWorker(i);
    }
    
    // Monitor workers
    setInterval(() => {
      const workers = Object.values(cluster.workers);
      console.log(`Active workers: ${workers.length}`);
    }, 30000);
  }
  
  forkWorker(id) {
    const worker = cluster.fork({ WORKER_ID: id });
    
    worker.on('exit', (code, signal) => {
      if (signal) {
        console.log(`Worker killed by signal: ${signal}`);
      } else if (code !== 0) {
        console.log(`Worker exited with error code: ${code}`);
      }
      
      console.log('Starting new worker...');
      this.forkWorker(id);
    });
  }
  
  startWorker() {
    const express = require('express');
    const app = express();
    
    app.get('/health', (req, res) => {
      res.json({
        worker: process.pid,
        uptime: process.uptime(),
        memory: process.memoryUsage()
      });
    });
    
    app.post('/api/transfer', async (req, res) => {
      try {
        const result = await this.processTransfer(req.body);
        res.json(result);
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });
    
    app.listen(3000, () => {
      console.log(`Worker ${process.pid} listening on 3000`);
    });
  }
  
  async processTransfer(data) {
    // Process transaction
    return { success: true, workerId: process.pid };
  }
}
```

### Q28. How do you implement caching strategies in Node.js?

**Answer:**

**Caching Strategies:**
1. In-memory cache (Map, LRU)
2. Redis
3. CDN caching
4. HTTP caching headers

```javascript
// 1. LRU Cache Implementation
class LRUCache {
  constructor(maxSize = 100) {
    this.maxSize = maxSize;
    this.cache = new Map();
  }
  
  get(key) {
    if (!this.cache.has(key)) return null;
    
    // Move to end (most recently used)
    const value = this.cache.get(key);
    this.cache.delete(key);
    this.cache.set(key, value);
    
    return value;
  }
  
  set(key, value) {
    // Delete if exists
    if (this.cache.has(key)) {
      this.cache.delete(key);
    }
    
    // Check size limit
    if (this.cache.size >= this.maxSize) {
      // Remove least recently used (first item)
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    
    this.cache.set(key, value);
  }
  
  has(key) {
    return this.cache.has(key);
  }
  
  clear() {
    this.cache.clear();
  }
}

// 2. Multi-level Cache
class MultiLevelCache {
  constructor() {
    this.l1 = new LRUCache(100); // Memory cache
    this.l2 = null; // Redis cache (initialized separately)
  }
  
  async get(key) {
    // Try L1 (memory)
    let value = this.l1.get(key);
    if (value) {
      console.log('L1 cache hit');
      return value;
    }
    
    // Try L2 (Redis)
    if (this.l2) {
      value = await this.l2.get(key);
      if (value) {
        console.log('L2 cache hit');
        // Promote to L1
        this.l1.set(key, value);
        return JSON.parse(value);
      }
    }
    
    console.log('Cache miss');
    return null;
  }
  
  async set(key, value, ttl = 3600) {
    // Set in L1
    this.l1.set(key, value);
    
    // Set in L2 with TTL
    if (this.l2) {
      await this.l2.setex(key, ttl, JSON.stringify(value));
    }
  }
}

// 3. Banking Cache Service
class BankingCacheService {
  constructor() {
    this.accountCache = new LRUCache(1000);
    this.transactionCache = new LRUCache(5000);
    this.cacheStats = {
      hits: 0,
      misses: 0
    };
  }
  
  async getAccount(accountId) {
    const cacheKey = `account:${accountId}`;
    
    // Check cache
    let account = this.accountCache.get(cacheKey);
    
    if (account) {
      this.cacheStats.hits++;
      return account;
    }
    
    this.cacheStats.misses++;
    
    // Fetch from database
    account = await this.fetchAccountFromDB(accountId);
    
    // Cache for 5 minutes
    this.accountCache.set(cacheKey, {
      data: account,
      timestamp: Date.now(),
      ttl: 300000 // 5 min
    });
    
    return account;
  }
  
  invalidateAccount(accountId) {
    const cacheKey = `account:${accountId}`;
    if (this.accountCache.has(cacheKey)) {
      this.accountCache.delete(cacheKey);
    }
  }
  
  getCacheStats() {
    const total = this.cacheStats.hits + this.cacheStats.misses;
    return {
      ...this.cacheStats,
      hitRate: total > 0 ? (this.cacheStats.hits / total * 100).toFixed(2) + '%' : '0%'
    };
  }
  
  async fetchAccountFromDB(accountId) {
    // Simulate DB query
    return new Promise(resolve => {
      setTimeout(() => {
        resolve({
          id: accountId,
          balance: Math.random() * 100000
        });
      }, 100);
    });
  }
}
```

### Q29. How do you optimize database queries in Node.js applications?

**Answer:**

```javascript
// Database Query Optimization
class OptimizedDatabaseService {
  constructor(pool) {
    this.pool = pool;
    this.queryCache = new Map();
  }
  
  // 1. Use connection pooling
  async query(sql, params) {
    const connection = await this.pool.getConnection();
    try {
      const [rows] = await connection.execute(sql, params);
      return rows;
    } finally {
      connection.release();
    }
  }
  
  // 2. Batch queries
  async getMultipleAccounts(accountIds) {
    // Instead of N queries, use single query with IN clause
    const placeholders = accountIds.map(() => '?').join(',');
    const sql = `SELECT * FROM accounts WHERE id IN (${placeholders})`;
    return this.query(sql, accountIds);
  }
  
  // 3. Use prepared statements
  async getAccountBalance(accountId) {
    const stmt = await this.pool.prepare(
      'SELECT balance FROM accounts WHERE id = ?'
    );
    const [rows] = await stmt.execute([accountId]);
    await stmt.close();
    return rows[0];
  }
  
  // 4. Implement query result caching
  async getCachedQuery(sql, params, ttl = 60000) {
    const cacheKey = `${sql}:${JSON.stringify(params)}`;
    
    if (this.queryCache.has(cacheKey)) {
      const cached = this.queryCache.get(cacheKey);
      if (Date.now() - cached.timestamp < ttl) {
        return cached.result;
      }
    }
    
    const result = await this.query(sql, params);
    this.queryCache.set(cacheKey, {
      result,
      timestamp: Date.now()
    });
    
    return result;
  }
  
  // 5. Use indexes effectively
  async createIndexes() {
    await this.query('CREATE INDEX idx_account_id ON transactions(account_id)');
    await this.query('CREATE INDEX idx_date ON transactions(date)');
    await this.query('CREATE INDEX idx_amount ON transactions(amount)');
  }
  
  // 6. Pagination
  async getTransactions(accountId, page = 1, limit = 50) {
    const offset = (page - 1) * limit;
    const sql = `
      SELECT * FROM transactions 
      WHERE account_id = ? 
      ORDER BY date DESC 
      LIMIT ? OFFSET ?
    `;
    return this.query(sql, [accountId, limit, offset]);
  }
}
```

### Q30. Explain garbage collection in V8 and how to optimize memory usage.

**Answer:**

**V8 Garbage Collection:**
- **Young Generation** (New Space): Short-lived objects
- **Old Generation** (Old Space): Long-lived objects
- **Scavenge**: GC for young generation (fast, frequent)
- **Mark-Sweep-Compact**: GC for old generation (slower, less frequent)

**Optimization Strategies:**

```javascript
// 1. Avoid memory leaks
class MemoryOptimized {
  constructor() {
    this.cache = new Map();
    this.timers = new Set();
    this.listeners = new Map();
  }
  
  // Always clean up timers
  scheduleTask(fn, delay) {
    const timer = setTimeout(() => {
      fn();
      this.timers.delete(timer);
    }, delay);
    
    this.timers.add(timer);
    return timer;
  }
  
  // Always remove event listeners
  addEventListener(emitter, event, handler) {
    emitter.on(event, handler);
    
    if (!this.listeners.has(emitter)) {
      this.listeners.set(emitter, []);
    }
    
    this.listeners.get(emitter).push({ event, handler });
  }
  
  cleanup() {
    // Clear timers
    this.timers.forEach(timer => clearTimeout(timer));
    this.timers.clear();
    
    // Remove listeners
    this.listeners.forEach((handlers, emitter) => {
      handlers.forEach(({ event, handler }) => {
        emitter.removeListener(event, handler);
      });
    });
    this.listeners.clear();
    
    // Clear cache
    this.cache.clear();
  }
}

// 2. Use Object pooling
class ObjectPool {
  constructor(createFn, resetFn, maxSize = 100) {
    this.createFn = createFn;
    this.resetFn = resetFn;
    this.maxSize = maxSize;
    this.pool = [];
  }
  
  acquire() {
    if (this.pool.length > 0) {
      return this.pool.pop();
    }
    return this.createFn();
  }
  
  release(obj) {
    if (this.pool.length < this.maxSize) {
      this.resetFn(obj);
      this.pool.push(obj);
    }
  }
}

// Usage
const transactionPool = new ObjectPool(
  () => ({ id: null, amount: 0, timestamp: null }),
  (obj) => {
    obj.id = null;
    obj.amount = 0;
    obj.timestamp = null;
  },
  1000
);

// 3. Monitor GC
if (global.gc) {
  setInterval(() => {
    const before = process.memoryUsage();
    global.gc();
    const after = process.memoryUsage();
    
    console.log('GC freed:', (before.heapUsed - after.heapUsed) / 1024 / 1024, 'MB');
  }, 60000);
}

// 4. Optimize data structures
class OptimizedDataStructures {
  // Use typed arrays for large numeric data
  createBalanceArray(size) {
    return new Float64Array(size); // More memory efficient
  }
  
  // Use Buffer for binary data
  serializeTransaction(txn) {
    const buffer = Buffer.allocUnsafe(32);
    buffer.writeDoubleBE(txn.amount, 0);
    buffer.writeBigInt64BE(BigInt(txn.timestamp), 8);
    return buffer;
  }
  
  // Use WeakMap for metadata
  constructor() {
    this.metadata = new WeakMap(); // Auto GC'd when object is released
  }
  
  setMetadata(obj, meta) {
    this.metadata.set(obj, meta);
  }
}
```

---

**🎯 Section 4 Complete! Questions 26-30 Finished**

Topics: Performance profiling, Clustering, Caching, Database optimization, Garbage collection
