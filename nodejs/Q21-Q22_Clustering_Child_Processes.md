# Node.js Interview Questions: Clustering & Child Processes
## Questions 21-22: Scaling Banking Applications

---

## 📋 Summary of What Was Covered

### ✅ Question 21: Clustering
- Basic cluster implementation
- Zero-downtime restart strategy
- Shared state management with Redis
- Master/worker architecture
- Health monitoring and metrics

### ✅ Question 22: Child Processes
- Credit score calculation with fork()
- Statement generation with process pools
- Fraud detection with spawn() (Python integration)
- When to use each method (fork, spawn, exec, execFile)

---

## ❓ Question 21: Cluster Module for Multi-Core Processing

### 📘 Comprehensive Explanation

**The Problem**: Node.js runs on a single thread by default. On a server with 8 CPU cores, only 1 core is utilized, wasting 87.5% of available processing power.

**The Solution**: The **cluster module** allows you to create multiple Node.js processes (workers) that share the same server port, utilizing all CPU cores.

**How Clustering Works**:

1. **Master Process**: 
   - Spawns worker processes (one per CPU core)
   - Distributes incoming connections to workers
   - Monitors worker health
   - Restarts crashed workers
   - Handles graceful shutdown

2. **Worker Processes**:
   - Handle actual requests
   - Share the same server port
   - Run independently
   - Can crash without affecting others

**Load Balancing Strategies**:
- **Round-robin** (default on most platforms): Distributes connections evenly
- **OS scheduling**: Operating system handles distribution

**Banking Use Cases**:
- **High-traffic APIs**: Handle 100,000+ requests/second
- **Payment Processing**: Distribute transaction load
- **Account Services**: Balance customer requests
- **Report Generation**: Parallel processing of statements
- **Batch Processing**: Distribute workload across cores

**Architecture Pattern**:
```
                    [Master Process]
                          |
        +-----------------+-----------------+
        |                 |                 |
   [Worker 1]        [Worker 2]        [Worker 3]
   Port 3000         Port 3000         Port 3000
   (CPU Core 1)      (CPU Core 2)      (CPU Core 3)
```

**Key Benefits**:
- ✅ Utilize all CPU cores
- ✅ Automatic load balancing
- ✅ Zero-downtime restarts
- ✅ Fault tolerance (worker crashes don't affect others)
- ✅ Increased throughput

**Important Considerations**:
- Workers don't share memory (use Redis for shared state)
- Each worker has its own event loop
- Master process should be lightweight
- Stateless workers are easier to manage
- Use PM2 or similar for production

---

### 🏦 Real-World Banking Scenario

**Context**: Your banking API receives **50,000 requests per minute** during peak hours. A single Node.js process can only handle ~5,000 requests/minute, causing:
- High response times (>2 seconds)
- Request timeouts
- Poor customer experience
- Lost revenue

**Requirements**:
1. Scale to handle 50,000+ requests/minute
2. Utilize all 8 CPU cores on production servers
3. Zero-downtime deployments
4. Automatic worker restart on crashes
5. Health monitoring for all workers
6. Graceful shutdown of workers

---

### 💻 Production-Ready Code Examples

#### Example 1: Basic Cluster Implementation for Banking API

```javascript
const cluster = require('cluster');
const http = require('http');
const os = require('os');

/**
 * Banking API Cluster Manager
 * Distributes load across all CPU cores
 */

if (cluster.isMaster) {
  console.log('🏦 Banking API Cluster - Master Process Starting');
  console.log(`   Master PID: ${process.pid}`);
  
  const numCPUs = os.cpus().length;
  console.log(`   CPU Cores: ${numCPUs}`);
  console.log(`   Spawning ${numCPUs} worker processes...\n`);
  
  // Track worker metrics
  const workerMetrics = new Map();
  
  // Fork workers
  for (let i = 0; i < numCPUs; i++) {
    const worker = cluster.fork();
    
    workerMetrics.set(worker.id, {
      pid: worker.process.pid,
      startTime: Date.now(),
      requests: 0,
      errors: 0
    });
    
    console.log(`   ✓ Worker ${worker.id} started (PID: ${worker.process.pid})`);
  }
  
  console.log('\n📊 All workers ready!\n');
  
  // Handle worker messages (for metrics tracking)
  cluster.on('message', (worker, message) => {
    if (message.type === 'request') {
      const metrics = workerMetrics.get(worker.id);
      if (metrics) {
        metrics.requests++;
      }
    } else if (message.type === 'error') {
      const metrics = workerMetrics.get(worker.id);
      if (metrics) {
        metrics.errors++;
      }
    }
  });
  
  // Handle worker exit
  cluster.on('exit', (worker, code, signal) => {
    console.log(`\n❌ Worker ${worker.id} (PID: ${worker.process.pid}) died`);
    
    const metrics = workerMetrics.get(worker.id);
    if (metrics) {
      const uptime = Math.round((Date.now() - metrics.startTime) / 1000);
      console.log(`   Uptime: ${uptime}s`);
      console.log(`   Requests handled: ${metrics.requests}`);
      console.log(`   Errors: ${metrics.errors}`);
      workerMetrics.delete(worker.id);
    }
    
    if (signal) {
      console.log(`   Killed by signal: ${signal}`);
    } else if (code !== 0) {
      console.log(`   Exited with code: ${code}`);
    }
    
    // Restart worker
    console.log('   🔄 Spawning replacement worker...');
    const newWorker = cluster.fork();
    
    workerMetrics.set(newWorker.id, {
      pid: newWorker.process.pid,
      startTime: Date.now(),
      requests: 0,
      errors: 0
    });
    
    console.log(`   ✓ New worker ${newWorker.id} started (PID: ${newWorker.process.pid})\n`);
  });
  
  // Graceful shutdown
  process.on('SIGTERM', () => {
    console.log('\n🛑 SIGTERM received - Gracefully shutting down cluster');
    
    for (const id in cluster.workers) {
      cluster.workers[id].kill();
    }
  });
  
  // Display metrics every 10 seconds
  setInterval(() => {
    console.log('\n📊 Cluster Metrics:');
    console.log('─────────────────────────────────────────');
    
    let totalRequests = 0;
    let totalErrors = 0;
    
    for (const [workerId, metrics] of workerMetrics.entries()) {
      const uptime = Math.round((Date.now() - metrics.startTime) / 1000);
      console.log(`Worker ${workerId} (PID ${metrics.pid}): ${metrics.requests} requests, ${metrics.errors} errors, ${uptime}s uptime`);
      totalRequests += metrics.requests;
      totalErrors += metrics.errors;
    }
    
    console.log('─────────────────────────────────────────');
    console.log(`Total: ${totalRequests} requests, ${totalErrors} errors`);
    console.log(`Workers: ${workerMetrics.size}/${numCPUs}\n`);
  }, 10000);
  
} else {
  // Worker process - Create HTTP server
  console.log(`   [Worker ${cluster.worker.id}] Starting HTTP server...`);
  
  // Simulate database connection
  const db = {
    query: async (sql) => {
      await new Promise(resolve => setTimeout(resolve, 10)); // Simulate DB latency
      return { rows: [{ balance: 5000 }] };
    }
  };
  
  const server = http.createServer(async (req, res) => {
    // Notify master of request (for metrics)
    process.send({ type: 'request' });
    
    const url = new URL(req.url, `http://${req.headers.host}`);
    
    try {
      // Route: Get account balance
      if (url.pathname === '/api/balance' && req.method === 'GET') {
        const accountId = url.searchParams.get('accountId') || 'ACC001';
        
        // Simulate database query
        const result = await db.query(`SELECT balance FROM accounts WHERE id = '${accountId}'`);
        
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({
          success: true,
          accountId,
          balance: result.rows[0].balance,
          workerId: cluster.worker.id,
          workerPid: process.pid
        }));
        
      // Route: Health check
      } else if (url.pathname === '/health' && req.method === 'GET') {
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({
          status: 'healthy',
          workerId: cluster.worker.id,
          workerPid: process.pid,
          uptime: process.uptime()
        }));
        
      // Route: Simulate error
      } else if (url.pathname === '/error' && req.method === 'GET') {
        throw new Error('Simulated error for testing');
        
      // 404
      } else {
        res.writeHead(404, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({
          error: 'Not found',
          workerId: cluster.worker.id
        }));
      }
      
    } catch (error) {
      process.send({ type: 'error' });
      console.error(`   [Worker ${cluster.worker.id}] Error:`, error.message);
      
      res.writeHead(500, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({
        error: 'Internal server error',
        workerId: cluster.worker.id
      }));
    }
  });
  
  const PORT = 3000;
  server.listen(PORT, () => {
    console.log(`   [Worker ${cluster.worker.id}] Listening on port ${PORT}`);
  });
  
  // Graceful shutdown for worker
  process.on('SIGTERM', () => {
    console.log(`\n   [Worker ${cluster.worker.id}] SIGTERM received - Closing server`);
    
    server.close(() => {
      console.log(`   [Worker ${cluster.worker.id}] Server closed`);
      process.exit(0);
    });
    
    // Force close after 10 seconds
    setTimeout(() => {
      console.log(`   [Worker ${cluster.worker.id}] Forcing shutdown`);
      process.exit(1);
    }, 10000);
  });
}

```

**To test this cluster:**
```bash
# Start the cluster
node Q21-Q22_Clustering_Child_Processes.js

# In another terminal, send requests:
curl http://localhost:3000/api/balance?accountId=ACC001
curl http://localhost:3000/health

# You'll see different worker IDs handling requests
```

---

#### Example 2: Advanced Cluster with Zero-Downtime Restart

This example shows how to implement **graceful restart** without dropping any requests - critical for banking applications where every transaction matters.

```javascript
const cluster = require('cluster');
const http = require('http');
const os = require('os');

/**
 * Zero-Downtime Banking API Cluster
 * Implements rolling restart strategy
 */
class BankingCluster {
  constructor(options = {}) {
    this.numWorkers = options.numWorkers || os.cpus().length;
    this.port = options.port || 3000;
    this.workers = new Map();
    this.restarting = false;
  }
  
  /**
   * Start cluster (Master process)
   */
  startMaster() {
    console.log('🏦 Banking API Cluster - Advanced Manager');
    console.log(`   Master PID: ${process.pid}`);
    console.log(`   Workers: ${this.numWorkers}`);
    console.log(`   Port: ${this.port}\n`);
    
    // Spawn workers
    for (let i = 0; i < this.numWorkers; i++) {
      this.spawnWorker();
    }
    
    // Setup master event handlers
    this.setupMasterHandlers();
    
    console.log('\n✅ Cluster ready! All workers online.\n');
  }
  
  /**
   * Spawn a new worker
   */
  spawnWorker() {
    const worker = cluster.fork({
      WORKER_PORT: this.port
    });
    
    this.workers.set(worker.id, {
      worker,
      pid: worker.process.pid,
      startTime: Date.now(),
      requests: 0,
      status: 'starting'
    });
    
    console.log(`   ✓ Worker ${worker.id} spawned (PID: ${worker.process.pid})`);
    
    // Worker ready notification
    worker.on('message', (msg) => {
      if (msg.type === 'ready') {
        const workerData = this.workers.get(worker.id);
        if (workerData) {
          workerData.status = 'online';
          console.log(`   ✓ Worker ${worker.id} is online and ready`);
        }
      } else if (msg.type === 'request') {
        const workerData = this.workers.get(worker.id);
        if (workerData) {
          workerData.requests++;
        }
      }
    });
    
    return worker;
  }
  
  /**
   * Setup master event handlers
   */
  setupMasterHandlers() {
    // Worker died - restart it
    cluster.on('exit', (worker, code, signal) => {
      const workerData = this.workers.get(worker.id);
      
      console.log(`\n❌ Worker ${worker.id} (PID: ${worker.process.pid}) died`);
      
      if (workerData) {
        const uptime = Math.round((Date.now() - workerData.startTime) / 1000);
        console.log(`   Uptime: ${uptime}s`);
        console.log(`   Requests: ${workerData.requests}`);
        this.workers.delete(worker.id);
      }
      
      if (signal) {
        console.log(`   Signal: ${signal}`);
      } else if (code !== 0) {
        console.log(`   Exit code: ${code}`);
      }
      
      // Don't restart during cluster shutdown
      if (!this.restarting) {
        console.log('   🔄 Spawning replacement worker...');
        this.spawnWorker();
      }
    });
    
    // Graceful shutdown
    process.on('SIGTERM', () => {
      console.log('\n🛑 SIGTERM - Initiating graceful cluster shutdown');
      this.gracefulShutdown();
    });
    
    process.on('SIGINT', () => {
      console.log('\n🛑 SIGINT - Initiating graceful cluster shutdown');
      this.gracefulShutdown();
    });
    
    // Graceful restart (no downtime)
    process.on('SIGUSR2', () => {
      console.log('\n🔄 SIGUSR2 - Initiating zero-downtime restart');
      this.rollingRestart();
    });
  }
  
  /**
   * Zero-downtime rolling restart
   * Restarts workers one at a time
   */
  async rollingRestart() {
    if (this.restarting) {
      console.log('⚠️  Restart already in progress');
      return;
    }
    
    this.restarting = true;
    console.log('🔄 Starting rolling restart...\n');
    
    const workerIds = Array.from(this.workers.keys());
    
    for (const workerId of workerIds) {
      const workerData = this.workers.get(workerId);
      if (!workerData) continue;
      
      console.log(`   Restarting worker ${workerId}...`);
      
      // Spawn new worker first
      const newWorker = this.spawnWorker();
      
      // Wait for new worker to be ready
      await this.waitForWorkerReady(newWorker.id, 10000);
      
      console.log(`   ✓ New worker ${newWorker.id} ready`);
      
      // Now gracefully shutdown old worker
      console.log(`   Shutting down old worker ${workerId}...`);
      workerData.worker.disconnect();
      
      // Wait for disconnect
      await this.waitForWorkerDisconnect(workerId, 5000);
      
      console.log(`   ✓ Worker ${workerId} shutdown complete\n`);
      
      // Small delay between restarts
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
    
    console.log('✅ Rolling restart complete! All workers updated.\n');
    this.restarting = false;
  }
  
  /**
   * Wait for worker to be ready
   */
  waitForWorkerReady(workerId, timeout = 10000) {
    return new Promise((resolve, reject) => {
      const startTime = Date.now();
      
      const checkInterval = setInterval(() => {
        const workerData = this.workers.get(workerId);
        
        if (workerData && workerData.status === 'online') {
          clearInterval(checkInterval);
          resolve();
        } else if (Date.now() - startTime > timeout) {
          clearInterval(checkInterval);
          reject(new Error(`Worker ${workerId} ready timeout`));
        }
      }, 100);
    });
  }
  
  /**
   * Wait for worker to disconnect
   */
  waitForWorkerDisconnect(workerId, timeout = 5000) {
    return new Promise((resolve) => {
      const workerData = this.workers.get(workerId);
      if (!workerData) {
        resolve();
        return;
      }
      
      const worker = workerData.worker;
      
      worker.on('disconnect', () => {
        resolve();
      });
      
      // Timeout fallback
      setTimeout(() => {
        if (worker.isConnected()) {
          worker.kill();
        }
        resolve();
      }, timeout);
    });
  }
  
  /**
   * Graceful cluster shutdown
   */
  async gracefulShutdown() {
    this.restarting = true;
    console.log('🛑 Shutting down all workers gracefully...\n');
    
    const shutdownPromises = [];
    
    for (const [workerId, workerData] of this.workers.entries()) {
      console.log(`   Shutting down worker ${workerId}...`);
      workerData.worker.disconnect();
      
      shutdownPromises.push(
        this.waitForWorkerDisconnect(workerId, 10000)
      );
    }
    
    await Promise.all(shutdownPromises);
    
    console.log('\n✅ All workers shut down gracefully');
    process.exit(0);
  }
  
  /**
   * Get cluster status
   */
  getStatus() {
    const workers = [];
    
    for (const [id, data] of this.workers.entries()) {
      workers.push({
        id,
        pid: data.pid,
        status: data.status,
        uptime: Math.round((Date.now() - data.startTime) / 1000),
        requests: data.requests
      });
    }
    
    return {
      master: process.pid,
      workers,
      totalWorkers: workers.length
    };
  }
}

// ============================================
// Worker Process Code
// ============================================

/**
 * Banking API Worker
 */
class BankingWorker {
  constructor() {
    this.port = parseInt(process.env.WORKER_PORT || '3000');
    this.workerId = cluster.worker.id;
    this.requestCount = 0;
  }
  
  start() {
    console.log(`   [Worker ${this.workerId}] Initializing...`);
    
    // Simulate connecting to database
    this.connectDatabase()
      .then(() => {
        // Start HTTP server
        this.startServer();
      })
      .catch(error => {
        console.error(`   [Worker ${this.workerId}] Initialization failed:`, error.message);
        process.exit(1);
      });
  }
  
  async connectDatabase() {
    console.log(`   [Worker ${this.workerId}] Connecting to database...`);
    // Simulate connection delay
    await new Promise(resolve => setTimeout(resolve, 500));
    console.log(`   [Worker ${this.workerId}] Database connected`);
  }
  
  startServer() {
    const server = http.createServer((req, res) => {
      this.handleRequest(req, res);
    });
    
    server.listen(this.port, () => {
      console.log(`   [Worker ${this.workerId}] Server listening on port ${this.port}`);
      
      // Notify master that worker is ready
      process.send({ type: 'ready' });
    });
    
    // Graceful shutdown
    process.on('SIGTERM', () => {
      console.log(`\n   [Worker ${this.workerId}] SIGTERM received`);
      this.gracefulShutdown(server);
    });
    
    cluster.worker.on('disconnect', () => {
      console.log(`   [Worker ${this.workerId}] Disconnecting...`);
      this.gracefulShutdown(server);
    });
  }
  
  async handleRequest(req, res) {
    this.requestCount++;
    process.send({ type: 'request' });
    
    const url = new URL(req.url, `http://${req.headers.host}`);
    
    try {
      if (url.pathname === '/api/transfer' && req.method === 'POST') {
        // Simulate transfer processing
        await new Promise(resolve => setTimeout(resolve, 50));
        
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({
          success: true,
          transactionId: `TXN${Date.now()}`,
          workerId: this.workerId,
          workerPid: process.pid
        }));
        
      } else if (url.pathname === '/api/balance' && req.method === 'GET') {
        // Simulate balance query
        await new Promise(resolve => setTimeout(resolve, 20));
        
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({
          success: true,
          balance: 5000.00,
          workerId: this.workerId,
          workerPid: process.pid
        }));
        
      } else if (url.pathname === '/health') {
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({
          status: 'healthy',
          workerId: this.workerId,
          workerPid: process.pid,
          uptime: process.uptime(),
          requests: this.requestCount
        }));
        
      } else {
        res.writeHead(404, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ error: 'Not found' }));
      }
      
    } catch (error) {
      console.error(`   [Worker ${this.workerId}] Error:`, error.message);
      res.writeHead(500, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({ error: 'Internal server error' }));
    }
  }
  
  gracefulShutdown(server) {
    console.log(`   [Worker ${this.workerId}] Starting graceful shutdown`);
    
    // Stop accepting new connections
    server.close(() => {
      console.log(`   [Worker ${this.workerId}] Server closed`);
      process.exit(0);
    });
    
    // Force exit after 10 seconds
    setTimeout(() => {
      console.log(`   [Worker ${this.workerId}] Forcing shutdown`);
      process.exit(1);
    }, 10000);
  }
}

// ============================================
// Main Entry Point
// ============================================

if (cluster.isMaster) {
  const bankingCluster = new BankingCluster({
    numWorkers: os.cpus().length,
    port: 3000
  });
  
  bankingCluster.startMaster();
  
  // Display status every 30 seconds
  setInterval(() => {
    const status = bankingCluster.getStatus();
    console.log('\n📊 Cluster Status:');
    console.log(`   Master PID: ${status.master}`);
    console.log(`   Active Workers: ${status.totalWorkers}`);
    status.workers.forEach(w => {
      console.log(`   - Worker ${w.id} (PID ${w.pid}): ${w.status}, ${w.uptime}s uptime, ${w.requests} requests`);
    });
    console.log('');
  }, 30000);
  
} else {
  const worker = new BankingWorker();
  worker.start();
}

```

**Key Features of this Advanced Implementation:**

1. **Zero-Downtime Restart**: 
   - Spawns new worker before killing old one
   - No dropped requests during deployment

2. **Graceful Shutdown**:
   - Waits for in-flight requests to complete
   - Closes connections cleanly

3. **Health Monitoring**:
   - Tracks worker status and metrics
   - Automatic restart on crashes

4. **Production Signals**:
   - `SIGTERM`: Graceful shutdown
   - `SIGUSR2`: Rolling restart (deploy new code)

**To test zero-downtime restart:**
```bash
# Start cluster
node advanced-cluster.js

# Send continuous traffic
while true; do curl http://localhost:3000/api/balance; sleep 0.1; done

# In another terminal, trigger restart (no requests will fail)
kill -USR2 <master_pid>
```

---

#### Example 3: Shared State Management with Redis

Since cluster workers don't share memory, we need external storage for shared state like session data, rate limiting, etc.

```javascript
const cluster = require('cluster');
const http = require('http');
const os = require('os');

/**
 * Banking Cluster with Shared State via Redis
 * Demonstrates handling shared data across workers
 */

// Mock Redis client (in production, use actual 'redis' or 'ioredis' package)
class MockRedis {
  constructor() {
    this.store = new Map();
  }
  
  async get(key) {
    return this.store.get(key) || null;
  }
  
  async set(key, value, expirySeconds = null) {
    this.store.set(key, value);
    
    if (expirySeconds) {
      setTimeout(() => {
        this.store.delete(key);
      }, expirySeconds * 1000);
    }
    
    return 'OK';
  }
  
  async incr(key) {
    const current = parseInt(this.store.get(key) || '0');
    const newValue = current + 1;
    this.store.set(key, newValue.toString());
    return newValue;
  }
  
  async expire(key, seconds) {
    setTimeout(() => {
      this.store.delete(key);
    }, seconds * 1000);
    return 1;
  }
  
  async ttl(key) {
    return this.store.has(key) ? 60 : -1;
  }
}

// Simulated shared Redis instance
const redis = new MockRedis();

// ============================================
// Master Process
// ============================================

if (cluster.isMaster) {
  console.log('🏦 Banking Cluster with Shared State');
  console.log(`   Master PID: ${process.pid}\n`);
  
  const numWorkers = os.cpus().length;
  
  // Spawn workers
  for (let i = 0; i < numWorkers; i++) {
    cluster.fork();
  }
  
  cluster.on('exit', (worker) => {
    console.log(`   Worker ${worker.id} died - Respawning`);
    cluster.fork();
  });
  
  console.log(`   Spawned ${numWorkers} workers\n`);

// ============================================
// Worker Process
// ============================================

} else {
  /**
   * Banking Session Manager
   * Manages user sessions across all workers using Redis
   */
  class SessionManager {
    constructor(redis) {
      this.redis = redis;
      this.sessionTimeout = 900; // 15 minutes
    }
    
    /**
     * Create new session
     */
    async createSession(userId) {
      const sessionId = `session:${userId}:${Date.now()}`;
      const sessionData = JSON.stringify({
        userId,
        createdAt: new Date().toISOString(),
        lastAccess: new Date().toISOString(),
        workerId: cluster.worker.id
      });
      
      await this.redis.set(sessionId, sessionData, this.sessionTimeout);
      
      console.log(`   [Worker ${cluster.worker.id}] Created session: ${sessionId}`);
      return sessionId;
    }
    
    /**
     * Validate session
     */
    async validateSession(sessionId) {
      const sessionData = await this.redis.get(sessionId);
      
      if (!sessionData) {
        return null;
      }
      
      // Refresh session timeout
      await this.redis.expire(sessionId, this.sessionTimeout);
      
      const session = JSON.parse(sessionData);
      session.currentWorkerId = cluster.worker.id;
      
      return session;
    }
    
    /**
     * Destroy session
     */
    async destroySession(sessionId) {
      await this.redis.set(sessionId, '', 1); // Expire in 1 second
      console.log(`   [Worker ${cluster.worker.id}] Destroyed session: ${sessionId}`);
    }
  }
  
  /**
   * Rate Limiter
   * Limits API calls per user across all workers
   */
  class RateLimiter {
    constructor(redis) {
      this.redis = redis;
      this.windowSeconds = 60;
      this.maxRequests = 100;
    }
    
    /**
     * Check if request is allowed
     */
    async checkLimit(userId) {
      const key = `ratelimit:${userId}`;
      const count = await this.redis.incr(key);
      
      if (count === 1) {
        // First request in window - set expiry
        await this.redis.expire(key, this.windowSeconds);
      }
      
      const remaining = Math.max(0, this.maxRequests - count);
      const allowed = count <= this.maxRequests;
      
      if (!allowed) {
        console.log(`   [Worker ${cluster.worker.id}] Rate limit exceeded for user ${userId}`);
      }
      
      return {
        allowed,
        remaining,
        limit: this.maxRequests,
        reset: this.windowSeconds
      };
    }
  }
  
  /**
   * Transaction Counter
   * Tracks global transaction count across all workers
   */
  class TransactionCounter {
    constructor(redis) {
      this.redis = redis;
    }
    
    async incrementTotal() {
      const total = await this.redis.incr('transactions:total');
      const today = await this.redis.incr(`transactions:${this.getToday()}`);
      
      return { total, today };
    }
    
    async getStats() {
      const total = await this.redis.get('transactions:total') || '0';
      const today = await this.redis.get(`transactions:${this.getToday()}`) || '0';
      
      return {
        total: parseInt(total),
        today: parseInt(today)
      };
    }
    
    getToday() {
      return new Date().toISOString().split('T')[0];
    }
  }
  
  // Initialize managers
  const sessionManager = new SessionManager(redis);
  const rateLimiter = new RateLimiter(redis);
  const transactionCounter = new TransactionCounter(redis);
  
  // Create HTTP server
  const server = http.createServer(async (req, res) => {
    const url = new URL(req.url, `http://${req.headers.host}`);
    
    try {
      // Route: Login (create session)
      if (url.pathname === '/api/login' && req.method === 'POST') {
        const userId = url.searchParams.get('userId') || 'USR001';
        
        const sessionId = await sessionManager.createSession(userId);
        
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({
          success: true,
          sessionId,
          workerId: cluster.worker.id
        }));
        
      // Route: Get balance (requires session)
      } else if (url.pathname === '/api/balance' && req.method === 'GET') {
        const sessionId = req.headers['x-session-id'];
        const userId = url.searchParams.get('userId') || 'USR001';
        
        // Validate session
        const session = await sessionManager.validateSession(sessionId);
        if (!session) {
          res.writeHead(401, { 'Content-Type': 'application/json' });
          res.end(JSON.stringify({
            error: 'Invalid or expired session',
            workerId: cluster.worker.id
          }));
          return;
        }
        
        // Check rate limit
        const rateLimit = await rateLimiter.checkLimit(userId);
        
        res.setHeader('X-RateLimit-Limit', rateLimit.limit);
        res.setHeader('X-RateLimit-Remaining', rateLimit.remaining);
        res.setHeader('X-RateLimit-Reset', rateLimit.reset);
        
        if (!rateLimit.allowed) {
          res.writeHead(429, { 'Content-Type': 'application/json' });
          res.end(JSON.stringify({
            error: 'Rate limit exceeded',
            retryAfter: rateLimit.reset,
            workerId: cluster.worker.id
          }));
          return;
        }
        
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({
          success: true,
          balance: 5000.00,
          workerId: cluster.worker.id,
          sessionCreatedBy: session.workerId,
          currentWorker: session.currentWorkerId
        }));
        
      // Route: Transfer (requires session, increments counter)
      } else if (url.pathname === '/api/transfer' && req.method === 'POST') {
        const sessionId = req.headers['x-session-id'];
        const userId = url.searchParams.get('userId') || 'USR001';
        
        // Validate session
        const session = await sessionManager.validateSession(sessionId);
        if (!session) {
          res.writeHead(401, { 'Content-Type': 'application/json' });
          res.end(JSON.stringify({ error: 'Invalid session' }));
          return;
        }
        
        // Check rate limit
        const rateLimit = await rateLimiter.checkLimit(userId);
        if (!rateLimit.allowed) {
          res.writeHead(429, { 'Content-Type': 'application/json' });
          res.end(JSON.stringify({ error: 'Rate limit exceeded' }));
          return;
        }
        
        // Increment transaction counter
        const stats = await transactionCounter.incrementTotal();
        
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({
          success: true,
          transactionId: `TXN${Date.now()}`,
          workerId: cluster.worker.id,
          globalStats: stats
        }));
        
      // Route: Stats (global statistics)
      } else if (url.pathname === '/api/stats' && req.method === 'GET') {
        const stats = await transactionCounter.getStats();
        
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({
          success: true,
          stats,
          workerId: cluster.worker.id
        }));
        
      // Route: Logout (destroy session)
      } else if (url.pathname === '/api/logout' && req.method === 'POST') {
        const sessionId = req.headers['x-session-id'];
        
        if (sessionId) {
          await sessionManager.destroySession(sessionId);
        }
        
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({
          success: true,
          workerId: cluster.worker.id
        }));
        
      } else {
        res.writeHead(404, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ error: 'Not found' }));
      }
      
    } catch (error) {
      console.error(`   [Worker ${cluster.worker.id}] Error:`, error.message);
      res.writeHead(500, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({ error: 'Internal server error' }));
    }
  });
  
  server.listen(3000, () => {
    console.log(`   [Worker ${cluster.worker.id}] Server ready on port 3000`);
  });
}

```

**Test the shared state:**

```bash
# 1. Start the cluster
node shared-state-cluster.js

# 2. Login and get session ID
SESSION=$(curl -s http://localhost:3000/api/login?userId=USR001 | jq -r .sessionId)
echo "Session: $SESSION"

# 3. Make requests with session (watch different workers handle requests)
curl -H "X-Session-Id: $SESSION" http://localhost:3000/api/balance?userId=USR001

# 4. Check global stats (consistent across all workers)
curl http://localhost:3000/api/stats

# 5. Test rate limiting (try 110+ requests quickly)
for i in {1..110}; do 
  curl -H "X-Session-Id: $SESSION" http://localhost:3000/api/balance?userId=USR001
done
```

**Key Learnings:**
- Workers don't share memory
- Use Redis/database for shared state
- Sessions work across all workers
- Rate limiting is global, not per-worker
- Transaction counters are consistent

---

### ✅ DO's and ❌ DON'Ts

#### ✅ DO's:

1. **DO** use clustering to utilize all CPU cores
2. **DO** implement graceful shutdown for workers
3. **DO** restart crashed workers automatically
4. **DO** use Redis for shared state across workers
5. **DO** implement zero-downtime deployments
6. **DO** monitor worker health and metrics
7. **DO** use PM2 or similar process managers in production
8. **DO** test cluster behavior under load
9. **DO** implement proper load balancing
10. **DO** log worker-specific information for debugging

#### ❌ DON'Ts:

1. **DON'T** share state via worker memory (use Redis/DB)
2. **DON'T** spawn too many workers (1 per CPU core is optimal)
3. **DON'T** forget to handle worker crashes
4. **DON'T** skip graceful shutdown logic
5. **DON'T** restart all workers simultaneously (rolling restart)
6. **DON'T** ignore memory leaks in workers
7. **DON'T** use clustering for CPU-light applications
8. **DON'T** forget to pass environment variables to workers
9. **DON'T** leave zombie processes after crashes
10. **DON'T** cluster in development (adds complexity)

---

## ❓ Question 22: Child Processes for Heavy Computations

### 📘 Comprehensive Explanation

While **clustering** distributes HTTP requests across workers, **child processes** handle CPU-intensive tasks without blocking the main event loop.

**The Problem**: 
Banking applications often need to perform heavy computations:
- **Credit score calculation**: Complex algorithms analyzing credit history
- **Fraud detection**: Machine learning models processing transactions
- **Report generation**: Large PDF/Excel files with thousands of rows
- **Image processing**: Check deposits, ID verification
- **Data encryption**: Large file encryption/decryption

These tasks can block the event loop for seconds, preventing the server from handling other requests.

**The Solution**: 
Spawn **child processes** to handle CPU-intensive work in parallel.

**Four Ways to Create Child Processes**:

1. **`spawn()`** - Stream-based, for long-running commands
   ```javascript
   const { spawn } = require('child_process');
   const ls = spawn('ls', ['-lh', '/usr']);
   ```

2. **`exec()`** - Buffer-based, for short commands
   ```javascript
   const { exec } = require('child_process');
   exec('ls -lh /usr', (error, stdout, stderr) => {});
   ```

3. **`execFile()`** - Execute file directly (no shell)
   ```javascript
   const { execFile } = require('child_process');
   execFile('/usr/bin/node', ['--version'], (error, stdout) => {});
   ```

4. **`fork()`** - Special case for Node.js processes
   ```javascript
   const { fork } = require('child_process');
   const child = fork('child-script.js');
   ```

**Key Differences**:

| Method | Use Case | Communication | Shell |
|--------|----------|---------------|-------|
| `spawn()` | Long-running, streams | stdout/stderr/stdin | No |
| `exec()` | Short commands | Buffered | Yes |
| `execFile()` | Direct execution | Buffered | No |
| `fork()` | Node.js scripts | IPC messages | No |

**Banking Use Cases**:
- **Credit Score Calculation**: Complex algorithms (fork)
- **Report Generation**: Large PDFs/CSVs (fork)
- **Fraud Detection**: ML models (fork/spawn)
- **Batch Processing**: Thousands of transactions (fork)
- **Data Migration**: Large datasets (spawn)
- **File Encryption**: Encrypt customer documents (fork)

---

### 🏦 Real-World Banking Scenario

**Context**: Your banking app needs to generate **monthly account statements** (PDFs) for 100,000 customers. Each PDF takes ~500ms to generate.

**Problem**: 
- Sequential processing: 100,000 × 500ms = **13.9 hours**
- Main process blocked → API unresponsive

**Solution**: 
Use child processes to parallelize PDF generation across CPU cores:
- 8 cores processing simultaneously
- **13.9 hours → 1.7 hours** (8x faster)

---

### 💻 Production-Ready Code Examples

#### Example 1: Credit Score Calculator with Fork

```javascript
// ============================================
// main.js - Main API Server
// ============================================

const { fork } = require('child_process');
const http = require('http');
const path = require('path');

/**
 * Banking API with CPU-Intensive Credit Score Calculation
 * Uses child process to avoid blocking main thread
 */

class CreditScoreService {
  constructor() {
    this.activeCalculations = new Map();
  }
  
  /**
   * Calculate credit score in child process
   */
  async calculateScore(customerId, creditData) {
    return new Promise((resolve, reject) => {
      console.log(`\n💳 Starting credit score calculation for ${customerId}`);
      console.log(`   Data points: ${creditData.transactions.length} transactions`);
      
      // Fork child process
      const childPath = path.join(__dirname, 'credit-score-worker.js');
      const child = fork(childPath);
      
      // Track calculation
      this.activeCalculations.set(customerId, child);
      
      // Set timeout (max 30 seconds)
      const timeout = setTimeout(() => {
        child.kill();
        this.activeCalculations.delete(customerId);
        reject(new Error('Credit score calculation timeout'));
      }, 30000);
      
      // Send data to child
      child.send({
        type: 'calculate',
        customerId,
        creditData
      });
      
      // Handle response
      child.on('message', (message) => {
        clearTimeout(timeout);
        
        if (message.type === 'result') {
          console.log(`   ✓ Credit score calculated: ${message.score}`);
          console.log(`   Calculation time: ${message.calculationTime}ms`);
          
          this.activeCalculations.delete(customerId);
          child.kill();
          resolve(message);
          
        } else if (message.type === 'error') {
          console.error(`   ✗ Calculation failed: ${message.error}`);
          
          this.activeCalculations.delete(customerId);
          child.kill();
          reject(new Error(message.error));
        }
      });
      
      // Handle errors
      child.on('error', (error) => {
        clearTimeout(timeout);
        this.activeCalculations.delete(customerId);
        reject(error);
      });
      
      child.on('exit', (code) => {
        clearTimeout(timeout);
        this.activeCalculations.delete(customerId);
        
        if (code !== 0) {
          reject(new Error(`Worker exited with code ${code}`));
        }
      });
    });
  }
  
  /**
   * Get active calculations
   */
  getActiveCalculations() {
    return Array.from(this.activeCalculations.keys());
  }
}

// Create service
const creditScoreService = new CreditScoreService();

// Create HTTP server
const server = http.createServer(async (req, res) => {
  const url = new URL(req.url, `http://${req.headers.host}`);
  
  try {
    // Route: Calculate credit score
    if (url.pathname === '/api/credit-score' && req.method === 'POST') {
      // Parse request body
      let body = '';
      req.on('data', chunk => body += chunk);
      
      await new Promise(resolve => req.on('end', resolve));
      
      const { customerId, creditData } = JSON.parse(body);
      
      // Calculate in child process (non-blocking)
      const result = await creditScoreService.calculateScore(customerId, creditData);
      
      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({
        success: true,
        customerId,
        creditScore: result.score,
        rating: result.rating,
        calculationTime: result.calculationTime
      }));
      
    // Route: Get active calculations
    } else if (url.pathname === '/api/active-calculations' && req.method === 'GET') {
      const active = creditScoreService.getActiveCalculations();
      
      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({
        success: true,
        activeCalculations: active.length,
        customers: active
      }));
      
    // Route: Health check (instant response even during heavy calculations)
    } else if (url.pathname === '/health') {
      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({
        status: 'healthy',
        uptime: process.uptime(),
        memory: process.memoryUsage().heapUsed / 1024 / 1024 + ' MB'
      }));
      
    } else {
      res.writeHead(404, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({ error: 'Not found' }));
    }
    
  } catch (error) {
    console.error('API Error:', error.message);
    res.writeHead(500, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
      error: error.message
    }));
  }
});

server.listen(3000, () => {
  console.log('🏦 Banking API with Credit Score Service');
  console.log('   Server listening on port 3000');
  console.log('   Main process PID:', process.pid);
  console.log('\nEndpoints:');
  console.log('   POST /api/credit-score - Calculate credit score');
  console.log('   GET  /api/active-calculations - View active calculations');
  console.log('   GET  /health - Health check\n');
});


// ============================================
// credit-score-worker.js - Child Process Worker
// ============================================

/**
 * Credit Score Calculation Worker
 * Runs CPU-intensive calculations without blocking main thread
 */

process.on('message', (message) => {
  if (message.type === 'calculate') {
    calculateCreditScore(message.customerId, message.creditData);
  }
});

function calculateCreditScore(customerId, creditData) {
  const startTime = Date.now();
  
  try {
    console.log(`   [Worker ${process.pid}] Calculating credit score for ${customerId}`);
    
    // Simulate complex calculation
    let score = 300; // Base score
    
    // Payment history (35% weight)
    const paymentHistory = analyzePay

mentHistory(creditData.transactions);
    score += paymentHistory * 0.35;
    
    // Credit utilization (30% weight)
    const utilization = creditData.totalBalance / creditData.totalLimit;
    const utilizationScore = (1 - utilization) * 300;
    score += utilizationScore * 0.30;
    
    // Length of credit history (15% weight)
    const historyYears = creditData.accountAge / 365;
    const historyScore = Math.min(historyYears * 30, 150);
    score += historyScore * 0.15;
    
    // New credit (10% weight)
    const newAccounts = creditData.recentInquiries;
    const newCreditScore = Math.max(0, 100 - (newAccounts * 10));
    score += newCreditScore * 0.10;
    
    // Credit mix (10% weight)
    const mixScore = creditData.accountTypes.length * 25;
    score += mixScore * 0.10;
    
    // Round to nearest integer
    score = Math.round(Math.min(Math.max(score, 300), 850));
    
    // Determine rating
    const rating = getRating(score);
    
    const calculationTime = Date.now() - startTime;
    
    // Simulate CPU-intensive work (in real app, this would be ML model)
    simulateCPUWork(500); // 500ms of CPU work
    
    // Send result back to parent
    process.send({
      type: 'result',
      score,
      rating,
      calculationTime: Date.now() - startTime
    });
    
  } catch (error) {
    process.send({
      type: 'error',
      error: error.message
    });
  }
}

function analyzePaymentHistory(transactions) {
  // Analyze on-time payments, late payments, defaults
  const onTimePayments = transactions.filter(t => t.status === 'on_time').length;
  const latePayments = transactions.filter(t => t.status === 'late').length;
  
  const onTimeRatio = onTimePayments / transactions.length;
  return onTimeRatio * 300; // Max 300 points for perfect payment history
}

function getRating(score) {
  if (score >= 800) return 'Excellent';
  if (score >= 740) return 'Very Good';
  if (score >= 670) return 'Good';
  if (score >= 580) return 'Fair';
  return 'Poor';
}

function simulateCPUWork(ms) {
  // Simulate CPU-intensive calculation
  const end = Date.now() + ms;
  let count = 0;
  
  while (Date.now() < end) {
    count++;
    Math.sqrt(count);
  }
}

console.log(`[Worker ${process.pid}] Credit score worker ready`);

```

**Test the credit score API:**

```bash
# Start server
node main.js

# In another terminal, test while checking main process responsiveness

# Calculate credit score (CPU-intensive, takes ~500ms)
curl -X POST http://localhost:3000/api/credit-score \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "CUST001",
    "creditData": {
      "transactions": [
        {"status": "on_time"}, {"status": "on_time"}, {"status": "late"}
      ],
      "totalBalance": 5000,
      "totalLimit": 10000,
      "accountAge": 1825,
      "recentInquiries": 2,
      "accountTypes": ["credit_card", "mortgage", "auto_loan"]
    }
  }'

# While calculation is running, health check is still instant!
curl http://localhost:3000/health

# Check active calculations
curl http://localhost:3000/api/active-calculations
```

**Key Observation**: Even while credit score calculation is running (CPU-intensive), the health check endpoint responds instantly because the calculation runs in a separate child process.

---

#### Example 2: Batch Statement Generation with Process Pool

```javascript
const { fork } = require('child_process');
const path = require('path');

/**
 * Statement Generation Service
 * Generates monthly statements for thousands of customers in parallel
 */
class StatementGenerator {
  constructor(options = {}) {
    this.poolSize = options.poolSize || require('os').cpus().length;
    this.workers = [];
    this.queue = [];
    this.activeJobs = new Map();
    
    this.initializePool();
  }
  
  /**
   * Initialize worker pool
   */
  initializePool() {
    console.log(`\n📊 Initializing statement generator pool`);
    console.log(`   Pool size: ${this.poolSize} workers\n`);
    
    for (let i = 0; i < this.poolSize; i++) {
      this.createWorker(i);
    }
  }
  
  /**
   * Create worker
   */
  createWorker(id) {
    const worker = {
      id,
      process: fork(path.join(__dirname, 'statement-worker.js')),
      busy: false
    };
    
    console.log(`   ✓ Worker ${id} created (PID: ${worker.process.pid})`);
    
    // Handle worker messages
    worker.process.on('message', (message) => {
      this.handleWorkerMessage(worker, message);
    });
    
    // Handle worker errors
    worker.process.on('error', (error) => {
      console.error(`   ✗ Worker ${id} error:`, error.message);
    });
    
    // Handle worker exit
    worker.process.on('exit', (code) => {
      if (code !== 0) {
        console.log(`   ✗ Worker ${id} exited with code ${code}, restarting...`);
        this.workers = this.workers.filter(w => w.id !== id);
        this.createWorker(id);
      }
    });
    
    this.workers.push(worker);
  }
  
  /**
   * Handle message from worker
   */
  handleWorkerMessage(worker, message) {
    if (message.type === 'ready') {
      worker.busy = false;
      this.processQueue();
      
    } else if (message.type === 'complete') {
      const job = this.activeJobs.get(message.jobId);
      
      if (job) {
        console.log(`   ✓ Statement generated: ${message.accountId} (${message.processingTime}ms)`);
        job.resolve(message.result);
        this.activeJobs.delete(message.jobId);
      }
      
      worker.busy = false;
      this.processQueue();
      
    } else if (message.type === 'error') {
      const job = this.activeJobs.get(message.jobId);
      
      if (job) {
        console.error(`   ✗ Statement generation failed: ${message.error}`);
        job.reject(new Error(message.error));
        this.activeJobs.delete(message.jobId);
      }
      
      worker.busy = false;
      this.processQueue();
    }
  }
  
  /**
   * Generate statement for account
   */
  async generateStatement(accountId, month, year) {
    return new Promise((resolve, reject) => {
      const jobId = `${accountId}-${month}-${year}-${Date.now()}`;
      
      const job = {
        jobId,
        accountId,
        month,
        year,
        resolve,
        reject
      };
      
      this.queue.push(job);
      this.activeJobs.set(jobId, job);
      
      this.processQueue();
    });
  }
  
  /**
   * Process job queue
   */
  processQueue() {
    // Find available worker
    const availableWorker = this.workers.find(w => !w.busy);
    
    if (!availableWorker || this.queue.length === 0) {
      return;
    }
    
    const job = this.queue.shift();
    availableWorker.busy = true;
    
    // Send job to worker
    availableWorker.process.send({
      type: 'generate',
      jobId: job.jobId,
      accountId: job.accountId,
      month: job.month,
      year: job.year
    });
  }
  
  /**
   * Generate statements for multiple accounts (batch)
   */
  async generateBatch(accounts, month, year) {
    console.log(`\n📦 Batch statement generation started`);
    console.log(`   Accounts: ${accounts.length}`);
    console.log(`   Period: ${month}/${year}\n`);
    
    const startTime = Date.now();
    
    const promises = accounts.map(accountId => 
      this.generateStatement(accountId, month, year)
    );
    
    const results = await Promise.allSettled(promises);
    
    const successful = results.filter(r => r.status === 'fulfilled').length;
    const failed = results.filter(r => r.status === 'rejected').length;
    const totalTime = Date.now() - startTime;
    
    console.log(`\n✅ Batch generation complete:`);
    console.log(`   Total: ${accounts.length} statements`);
    console.log(`   Successful: ${successful}`);
    console.log(`   Failed: ${failed}`);
    console.log(`   Total time: ${totalTime}ms`);
    console.log(`   Average: ${Math.round(totalTime / accounts.length)}ms per statement\n`);
    
    return {
      total: accounts.length,
      successful,
      failed,
      totalTime,
      results
    };
  }
  
  /**
   * Get pool status
   */
  getStatus() {
    return {
      poolSize: this.poolSize,
      busyWorkers: this.workers.filter(w => w.busy).length,
      availableWorkers: this.workers.filter(w => !w.busy).length,
      queueLength: this.queue.length,
      activeJobs: this.activeJobs.size
    };
  }
  
  /**
   * Shutdown pool
   */
  shutdown() {
    console.log('\n🛑 Shutting down worker pool...');
    
    this.workers.forEach(worker => {
      worker.process.kill();
    });
    
    this.workers = [];
    console.log('   ✓ All workers terminated\n');
  }
}


// ============================================
// statement-worker.js - Worker Script
// ============================================

/**
 * Statement Generation Worker
 * Generates PDF/CSV statements for accounts
 */

process.on('message', async (message) => {
  if (message.type === 'generate') {
    await generateStatement(message);
  }
});

async function generateStatement(job) {
  const startTime = Date.now();
  
  try {
    const { jobId, accountId, month, year } = job;
    
    // Simulate fetching transactions from database
    await simulateDelay(50);
    const transactions = await fetchTransactions(accountId, month, year);
    
    // Simulate generating PDF (CPU-intensive)
    await simulateDelay(200);
    const pdfBuffer = await generatePDF(accountId, transactions, month, year);
    
    // Simulate saving to storage
    await simulateDelay(30);
    const fileUrl = await saveToStorage(accountId, pdfBuffer, month, year);
    
    const processingTime = Date.now() - startTime;
    
    // Send success response
    process.send({
      type: 'complete',
      jobId,
      accountId,
      result: {
        fileUrl,
        fileSize: pdfBuffer.length,
        transactionCount: transactions.length
      },
      processingTime
    });
    
  } catch (error) {
    process.send({
      type: 'error',
      jobId: job.jobId,
      error: error.message
    });
  }
  
  // Notify ready for next job
  process.send({ type: 'ready' });
}

async function fetchTransactions(accountId, month, year) {
  // Simulate database query
  const count = Math.floor(Math.random() * 50) + 10;
  return Array.from({ length: count }, (_, i) => ({
    id: `TXN${i}`,
    date: `${year}-${String(month).padStart(2, '0')}-${String(i + 1).padStart(2, '0')}`,
    description: 'Transaction ' + i,
    amount: (Math.random() * 1000).toFixed(2)
  }));
}

async function generatePDF(accountId, transactions, month, year) {
  // Simulate PDF generation (CPU-intensive)
  let pdfContent = `Statement for Account ${accountId}\n`;
  pdfContent += `Period: ${month}/${year}\n\n`;
  pdfContent += `Transactions: ${transactions.length}\n`;
  
  transactions.forEach(txn => {
    pdfContent += `${txn.date} - ${txn.description}: $${txn.amount}\n`;
  });
  
  // Simulate CPU work
  const end = Date.now() + 100;
  while (Date.now() < end) {
    Math.sqrt(Math.random());
  }
  
  return Buffer.from(pdfContent);
}

async function saveToStorage(accountId, pdfBuffer, month, year) {
  // Simulate S3/storage upload
  return `s3://statements/${accountId}/${year}/${month}/statement.pdf`;
}

function simulateDelay(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

console.log(`[Worker ${process.pid}] Statement worker ready`);
process.send({ type: 'ready' });


// ============================================
// Usage Example
// ============================================

// Create generator with 4 workers
const generator = new StatementGenerator({ poolSize: 4 });

// Simulate batch generation for 20 accounts
const accounts = Array.from({ length: 20 }, (_, i) => `ACC${String(i + 1).padStart(4, '0')}`);

// Generate statements
(async () => {
  const result = await generator.generateBatch(accounts, 11, 2024);
  
  console.log('📊 Final Results:');
  console.log(`   Success rate: ${(result.successful / result.total * 100).toFixed(2)}%`);
  console.log(`   Throughput: ${(result.total / (result.totalTime / 1000)).toFixed(2)} statements/sec`);
  
  // Shutdown
  generator.shutdown();
})();

```

**Key Features:**
1. **Worker Pool**: Reuses workers instead of creating/destroying for each job
2. **Job Queue**: Queues requests when all workers are busy
3. **Parallel Processing**: Multiple statements generated simultaneously
4. **Resource Management**: Limits concurrent workers to CPU count

**Performance Comparison:**
```
Sequential (1 worker):  20 statements × 300ms = 6000ms
Parallel (4 workers):   20 statements ÷ 4 × 300ms = 1500ms (4x faster)
Parallel (8 workers):   20 statements ÷ 8 × 300ms = 750ms (8x faster)
```

---

#### Example 3: Fraud Detection with Spawn (Python ML Model)

Banks often use Python for machine learning. Here's how to call Python fraud detection from Node.js:

```javascript
const { spawn } = require('child_process');
const path = require('path');

/**
 * Fraud Detection Service
 * Uses Python ML model via spawn
 */
class FraudDetectionService {
  constructor() {
    this.pythonPath = process.env.PYTHON_PATH || 'python3';
    this.modelScript = path.join(__dirname, 'fraud_model.py');
  }
  
  /**
   * Analyze transaction for fraud
   */
  async analyzeTransaction(transaction) {
    return new Promise((resolve, reject) => {
      console.log(`\n🔍 Analyzing transaction ${transaction.id} for fraud`);
      
      const startTime = Date.now();
      
      // Spawn Python process
      const python = spawn(this.pythonPath, [
        this.modelScript,
        JSON.stringify(transaction)
      ]);
      
      let output = '';
      let errorOutput = '';
      
      // Collect stdout
      python.stdout.on('data', (data) => {
        output += data.toString();
      });
      
      // Collect stderr
      python.stderr.on('data', (data) => {
        errorOutput += data.toString();
      });
      
      // Handle completion
      python.on('close', (code) => {
        const analysisTime = Date.now() - startTime;
        
        if (code !== 0) {
          console.error(`   ✗ Fraud detection failed (exit code ${code})`);
          console.error(`   Error: ${errorOutput}`);
          reject(new Error(`Fraud detection failed: ${errorOutput}`));
          return;
        }
        
        try {
          const result = JSON.parse(output);
          
          console.log(`   ${result.isFraud ? '🚨 FRAUD DETECTED' : '✓ Transaction appears legitimate'}`);
          console.log(`   Fraud probability: ${(result.fraudScore * 100).toFixed(2)}%`);
          console.log(`   Analysis time: ${analysisTime}ms`);
          
          resolve({
            ...result,
            analysisTime
          });
          
        } catch (error) {
          reject(new Error(`Failed to parse fraud detection output: ${error.message}`));
        }
      });
      
      // Handle errors
      python.on('error', (error) => {
        reject(new Error(`Failed to start fraud detection: ${error.message}`));
      });
      
      // Set timeout
      setTimeout(() => {
        python.kill();
        reject(new Error('Fraud detection timeout'));
      }, 5000);
    });
  }
  
  /**
   * Batch analyze multiple transactions
   */
  async batchAnalyze(transactions) {
    console.log(`\n📦 Batch fraud analysis: ${transactions.length} transactions\n`);
    
    const startTime = Date.now();
    
    // Analyze in parallel (limit concurrency to avoid overwhelming system)
    const concurrency = 5;
    const results = [];
    
    for (let i = 0; i < transactions.length; i += concurrency) {
      const batch = transactions.slice(i, i + concurrency);
      const batchResults = await Promise.allSettled(
        batch.map(txn => this.analyzeTransaction(txn))
      );
      results.push(...batchResults);
    }
    
    const totalTime = Date.now() - startTime;
    const successful = results.filter(r => r.status === 'fulfilled').length;
    const fraudulent = results
      .filter(r => r.status === 'fulfilled' && r.value.isFraud)
      .length;
    
    console.log(`\n✅ Batch analysis complete:`);
    console.log(`   Total: ${transactions.length}`);
    console.log(`   Successful: ${successful}`);
    console.log(`   Fraudulent: ${fraudulent}`);
    console.log(`   Total time: ${totalTime}ms`);
    console.log(`   Average: ${Math.round(totalTime / transactions.length)}ms per transaction\n`);
    
    return {
      total: transactions.length,
      successful,
      fraudulent,
      totalTime,
      results
    };
  }
}


// ============================================
// fraud_model.py - Python ML Model (Simulated)
// ============================================

/*
#!/usr/bin/env python3
import sys
import json
import time
import random

def analyze_fraud(transaction):
    """
    Simulated ML fraud detection model
    In production, this would use scikit-learn, TensorFlow, etc.
    """
    
    # Simulate model inference time
    time.sleep(0.1)
    
    # Simple rule-based fraud detection (replace with real ML model)
    fraud_score = 0.0
    
    # High amount transactions
    if transaction['amount'] > 10000:
        fraud_score += 0.3
    
    # International transactions
    if transaction.get('isInternational', False):
        fraud_score += 0.2
    
    # Unusual time (late night)
    hour = int(transaction.get('timestamp', '12:00:00').split(':')[0])
    if hour < 6 or hour > 22:
        fraud_score += 0.15
    
    # Multiple transactions in short time
    if transaction.get('recentTransactionCount', 0) > 5:
        fraud_score += 0.25
    
    # Add some randomness
    fraud_score += random.uniform(0, 0.1)
    
    # Cap at 1.0
    fraud_score = min(fraud_score, 1.0)
    
    result = {
        'transactionId': transaction['id'],
        'isFraud': fraud_score > 0.6,
        'fraudScore': fraud_score,
        'riskLevel': 'HIGH' if fraud_score > 0.7 else 'MEDIUM' if fraud_score > 0.4 else 'LOW',
        'reasons': []
    }
    
    if transaction['amount'] > 10000:
        result['reasons'].append('High transaction amount')
    if transaction.get('isInternational'):
        result['reasons'].append('International transaction')
    if hour < 6 or hour > 22:
        result['reasons'].append('Unusual transaction time')
    
    return result

if __name__ == '__main__':
    if len(sys.argv) < 2:
        print(json.dumps({'error': 'No transaction data provided'}))
        sys.exit(1)
    
    try:
        transaction = json.loads(sys.argv[1])
        result = analyze_fraud(transaction)
        print(json.dumps(result))
        sys.exit(0)
        
    except Exception as e:
        print(json.dumps({'error': str(e)}), file=sys.stderr)
        sys.exit(1)
*/


// ============================================
// Usage Example
// ============================================

const fraudService = new FraudDetectionService();

// Test single transaction
(async () => {
  const transaction = {
    id: 'TXN001',
    amount: 15000,
    isInternational: true,
    timestamp: '02:30:00',
    recentTransactionCount: 7
  };
  
  try {
    const result = await fraudService.analyzeTransaction(transaction);
    
    if (result.isFraud) {
      console.log('\n🚨 ACTION REQUIRED: Flag transaction for manual review');
      console.log(`   Risk Level: ${result.riskLevel}`);
      console.log(`   Reasons: ${result.reasons.join(', ')}`);
    }
    
  } catch (error) {
    console.error('Fraud detection error:', error.message);
  }
  
  // Test batch analysis
  const transactions = [
    { id: 'TXN001', amount: 100, isInternational: false, timestamp: '14:00:00', recentTransactionCount: 1 },
    { id: 'TXN002', amount: 25000, isInternational: true, timestamp: '03:00:00', recentTransactionCount: 8 },
    { id: 'TXN003', amount: 500, isInternational: false, timestamp: '10:00:00', recentTransactionCount: 2 }
  ];
  
  await fraudService.batchAnalyze(transactions);
})();

```

**Why use `spawn()` for Python?**
- **Streaming**: Can process large outputs without buffering
- **Real-time**: Get stdout data as it arrives
- **Flexibility**: Works with any executable, not just Python
- **Memory-efficient**: Doesn't buffer entire output

---

### ✅ DO's and ❌ DON'Ts

#### ✅ DO's:

1. **DO** use `fork()` for Node.js child processes
2. **DO** use `spawn()` for long-running or streaming processes
3. **DO** implement process pools for repeated tasks
4. **DO** set timeouts for child processes
5. **DO** handle process errors and exit codes
6. **DO** limit concurrent child processes
7. **DO** clean up (kill) child processes properly
8. **DO** use IPC for communication with forked processes
9. **DO** validate data from child processes
10. **DO** monitor memory usage of child processes

#### ❌ DON'Ts:

1. **DON'T** use child processes for I/O-bound tasks (use async instead)
2. **DON'T** create unlimited child processes (use pools)
3. **DON'T** forget to handle process exit/error events
4. **DON'T** trust data from child processes without validation
5. **DON'T** use `exec()` with user input (security risk)
6. **DON'T** forget to set timeouts
7. **DON'T** leave zombie processes
8. **DON'T** use child processes for simple tasks
9. **DON'T** ignore memory overhead of spawning processes
10. **DON'T** block the event loop while waiting for child process

---

## 🎓 Key Takeaways

### Clustering:
1. **Utilizes all CPU cores** for HTTP servers
2. **Master/Worker pattern** with automatic load balancing
3. **Zero-downtime restarts** with rolling updates
4. **Shared state requires Redis** or external storage
5. **PM2** recommended for production clustering

### Child Processes:
1. **`fork()`** for Node.js CPU-intensive tasks
2. **`spawn()`** for long-running or streaming processes
3. **Process pools** for efficient resource management
4. **Non-blocking** - keeps main thread responsive
5. **Cross-language** - call Python, Java, etc. from Node.js

### When to Use What:
- **Clustering**: Scale HTTP servers across cores
- **Child Processes**: CPU-intensive computations
- **Worker Threads**: Shared memory CPU tasks
- **Async/Await**: I/O-bound operations

---

**Next Topic**: Memory Management & Performance Optimization

