# Q20: Process Management - Signals

## 📋 Summary
This question covers **process signals** in Node.js - how to handle SIGTERM, SIGINT, SIGUSR1, SIGUSR2, and implement graceful shutdown patterns. We'll build production-ready banking systems that shut down safely without losing in-flight transactions, properly close database connections, and drain active requests.

---

## 🎯 What You'll Learn
- Understanding Unix/POSIX signals
- Handling SIGTERM (termination signal)
- Handling SIGINT (interrupt signal - Ctrl+C)
- Custom signals (SIGUSR1, SIGUSR2)
- Graceful shutdown patterns
- Connection draining
- Cleanup strategies
- Banking Examples: Zero-downtime deployments, safe API shutdown, transaction completion

---

## 📖 Detailed Explanation

### What are Process Signals?

**Signals** are asynchronous notifications sent to a process to notify it of an event. In Node.js, you can listen for signals using `process.on(signal, handler)`.

Common signals:
- **SIGTERM**: Graceful termination request (default for `kill` command)
- **SIGINT**: Interrupt from keyboard (Ctrl+C)
- **SIGKILL**: Forced termination (cannot be caught)
- **SIGHUP**: Hangup (terminal closed)
- **SIGUSR1**: User-defined signal 1
- **SIGUSR2**: User-defined signal 2

```javascript
// Basic signal handling
process.on('SIGTERM', () => {
  console.log('Received SIGTERM');
  process.exit(0);
});

process.on('SIGINT', () => {
  console.log('Received SIGINT (Ctrl+C)');
  process.exit(0);
});
```

---

### SIGTERM - Graceful Termination

**SIGTERM** is sent by process managers (PM2, Kubernetes, Docker) to request graceful shutdown.

```javascript
process.on('SIGTERM', () => {
  console.log('SIGTERM received, shutting down gracefully...');
  
  // Close server
  server.close(() => {
    console.log('Server closed');
    
    // Close database
    db.close(() => {
      console.log('Database closed');
      process.exit(0);
    });
  });
});
```

**When SIGTERM is sent:**
- `kill <pid>` (default signal)
- `docker stop <container>` (after 10 seconds, sends SIGKILL)
- Kubernetes pod termination
- PM2 reload/restart
- Heroku dyno shutdown

---

### SIGINT - Keyboard Interrupt

**SIGINT** is sent when you press **Ctrl+C** in the terminal.

```javascript
process.on('SIGINT', () => {
  console.log('\nReceived SIGINT (Ctrl+C)');
  console.log('Shutting down gracefully...');
  
  gracefulShutdown('SIGINT');
});
```

---

### SIGUSR1 and SIGUSR2 - Custom Signals

**SIGUSR1** and **SIGUSR2** are user-defined signals for custom behavior.

```javascript
// SIGUSR1 - Trigger memory dump or debug info
process.on('SIGUSR1', () => {
  console.log('SIGUSR1 received - Generating memory report');
  const memUsage = process.memoryUsage();
  console.log('Memory Usage:', memUsage);
});

// SIGUSR2 - Reload configuration
process.on('SIGUSR2', () => {
  console.log('SIGUSR2 received - Reloading configuration');
  reloadConfig();
});
```

**Send custom signals:**
```bash
# Send SIGUSR1
kill -USR1 <pid>

# Send SIGUSR2
kill -USR2 <pid>
```

**Note**: Node.js uses SIGUSR1 internally for debugging, so using it may interfere with `--inspect`.

---

### Graceful Shutdown Pattern

A proper graceful shutdown should:

1. **Stop accepting new requests**
2. **Complete in-flight requests**
3. **Close database connections**
4. **Flush logs and caches**
5. **Exit with appropriate code**

```javascript
async function gracefulShutdown(signal) {
  console.log(`${signal} received, starting graceful shutdown`);
  
  try {
    // 1. Stop accepting new connections
    console.log('Closing HTTP server...');
    await new Promise((resolve, reject) => {
      server.close((err) => {
        if (err) reject(err);
        else resolve();
      });
    });
    console.log('✓ HTTP server closed');
    
    // 2. Close database connections
    console.log('Closing database connections...');
    await db.end();
    console.log('✓ Database closed');
    
    // 3. Close Redis connections
    console.log('Closing Redis...');
    await redis.quit();
    console.log('✓ Redis closed');
    
    // 4. Flush logs
    console.log('Flushing logs...');
    await logger.flush();
    console.log('✓ Logs flushed');
    
    console.log('Graceful shutdown completed');
    process.exit(0);
    
  } catch (error) {
    console.error('Error during shutdown:', error);
    process.exit(1);
  }
}

// Handle multiple signals
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
```

---

### Shutdown Timeout

Always implement a **timeout** to force shutdown if graceful shutdown hangs:

```javascript
const SHUTDOWN_TIMEOUT = 30000; // 30 seconds

async function gracefulShutdown(signal) {
  console.log(`${signal} received`);
  
  // Set timeout for forced shutdown
  const forceShutdown = setTimeout(() => {
    console.error('Graceful shutdown timeout - forcing exit');
    process.exit(1);
  }, SHUTDOWN_TIMEOUT);
  
  try {
    // Perform cleanup
    await cleanup();
    
    clearTimeout(forceShutdown);
    process.exit(0);
  } catch (error) {
    console.error('Shutdown error:', error);
    clearTimeout(forceShutdown);
    process.exit(1);
  }
}
```

---

### Connection Draining

**Connection draining** ensures in-flight HTTP requests complete before shutdown:

```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  // Track active connections
  activeConnections++;
  
  res.on('finish', () => {
    activeConnections--;
  });
  
  // Your handler
  handleRequest(req, res);
});

let activeConnections = 0;
let isShuttingDown = false;

server.on('connection', (socket) => {
  // Track sockets
  activeSockets.push(socket);
  
  socket.on('close', () => {
    const index = activeSockets.indexOf(socket);
    if (index !== -1) activeSockets.splice(index, 1);
  });
});

async function gracefulShutdown() {
  isShuttingDown = true;
  
  // Stop accepting new connections
  server.close();
  
  // Wait for active connections to finish
  const waitForConnections = setInterval(() => {
    if (activeConnections === 0) {
      clearInterval(waitForConnections);
      process.exit(0);
    } else {
      console.log(`Waiting for ${activeConnections} connections to close...`);
    }
  }, 1000);
  
  // Force close after timeout
  setTimeout(() => {
    console.log('Forcing shutdown');
    activeSockets.forEach(socket => socket.destroy());
    process.exit(0);
  }, 30000);
}
```

---

### Signal Handling Best Practices

```javascript
class SignalHandler {
  constructor() {
    this.signals = ['SIGTERM', 'SIGINT', 'SIGHUP'];
    this.shutdownInProgress = false;
    this.cleanupHandlers = [];
  }

  // Register cleanup function
  onShutdown(handler) {
    this.cleanupHandlers.push(handler);
  }

  // Setup signal listeners
  listen() {
    this.signals.forEach(signal => {
      process.on(signal, () => this.handleSignal(signal));
    });
  }

  async handleSignal(signal) {
    if (this.shutdownInProgress) {
      console.log('Shutdown already in progress...');
      return;
    }
    
    this.shutdownInProgress = true;
    console.log(`\nReceived ${signal}, shutting down gracefully...`);
    
    // Execute all cleanup handlers
    for (const handler of this.cleanupHandlers) {
      try {
        await handler();
      } catch (error) {
        console.error('Cleanup handler error:', error);
      }
    }
    
    process.exit(0);
  }
}

// Usage
const signalHandler = new SignalHandler();

signalHandler.onShutdown(async () => {
  console.log('Closing server...');
  await server.close();
});

signalHandler.onShutdown(async () => {
  console.log('Closing database...');
  await db.end();
});

signalHandler.listen();
```

---

## 🏦 Banking Application Examples

### Example 1: Banking API with Graceful Shutdown and Transaction Completion

**Scenario**: Build a banking API that ensures all in-flight transactions complete before shutdown. No transaction should be lost during deployment or restart.

**File: `banking-api-graceful-shutdown.js`**

```javascript
const express = require('express');
const { Pool } = require('pg');
const Redis = require('ioredis');

class BankingAPI {
  constructor() {
    this.app = express();
    this.server = null;
    this.isShuttingDown = false;
    
    // Track in-flight transactions
    this.activeTransactions = new Set();
    this.activeSockets = new Set();
    
    // Initialize connections
    this.db = new Pool({
      host: process.env.DB_HOST || 'localhost',
      port: process.env.DB_PORT || 5432,
      database: process.env.DB_NAME || 'banking',
      user: process.env.DB_USER || 'postgres',
      password: process.env.DB_PASSWORD,
      max: 20,
      idleTimeoutMillis: 30000
    });
    
    this.redis = new Redis({
      host: process.env.REDIS_HOST || 'localhost',
      port: process.env.REDIS_PORT || 6379,
      retryStrategy: (times) => Math.min(times * 50, 2000)
    });
    
    this.setupMiddleware();
    this.setupRoutes();
    this.setupSignalHandlers();
    this.setupHealthChecks();
  }

  setupMiddleware() {
    this.app.use(express.json());
    
    // Shutdown middleware - reject new requests during shutdown
    this.app.use((req, res, next) => {
      if (this.isShuttingDown) {
        res.setHeader('Connection', 'close');
        return res.status(503).json({
          error: 'Service shutting down',
          message: 'Server is shutting down, please retry'
        });
      }
      next();
    });
    
    // Request logging
    this.app.use((req, res, next) => {
      const requestId = `REQ-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
      req.requestId = requestId;
      
      console.log(`[${requestId}] ${req.method} ${req.url}`);
      
      const cleanup = () => {
        console.log(`[${requestId}] Completed`);
      };
      
      res.on('finish', cleanup);
      res.on('close', cleanup);
      
      next();
    });
  }

  setupRoutes() {
    // Health check
    this.app.get('/health', (req, res) => {
      const status = {
        status: this.isShuttingDown ? 'shutting_down' : 'healthy',
        timestamp: new Date().toISOString(),
        uptime: Math.floor(process.uptime()),
        activeTransactions: this.activeTransactions.size,
        activeSockets: this.activeSockets.size
      };
      
      const httpStatus = this.isShuttingDown ? 503 : 200;
      res.status(httpStatus).json(status);
    });

    // Transfer money (critical transaction)
    this.app.post('/api/v1/transfer', async (req, res) => {
      const transactionId = `TXN-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
      
      console.log(`[${req.requestId}] Starting transaction ${transactionId}`);
      
      // Track this transaction
      this.activeTransactions.add(transactionId);
      
      try {
        const { fromAccount, toAccount, amount, memo } = req.body;
        
        // Validate input
        if (!fromAccount || !toAccount || !amount) {
          return res.status(400).json({
            error: 'Missing required fields',
            required: ['fromAccount', 'toAccount', 'amount']
          });
        }

        if (amount <= 0) {
          return res.status(400).json({ error: 'Amount must be positive' });
        }

        // Simulate processing delay (1-3 seconds)
        const processingTime = 1000 + Math.random() * 2000;
        await new Promise(resolve => setTimeout(resolve, processingTime));
        
        // Start database transaction
        const client = await this.db.connect();
        
        try {
          await client.query('BEGIN');
          
          // Check source account balance
          const sourceResult = await client.query(
            'SELECT balance FROM accounts WHERE account_id = $1 FOR UPDATE',
            [fromAccount]
          );
          
          if (sourceResult.rows.length === 0) {
            throw new Error('Source account not found');
          }
          
          const sourceBalance = parseFloat(sourceResult.rows[0].balance);
          
          if (sourceBalance < amount) {
            throw new Error('Insufficient funds');
          }
          
          // Debit source account
          await client.query(
            'UPDATE accounts SET balance = balance - $1 WHERE account_id = $2',
            [amount, fromAccount]
          );
          
          // Credit destination account
          await client.query(
            'UPDATE accounts SET balance = balance + $1 WHERE account_id = $2',
            [amount, toAccount]
          );
          
          // Record transaction
          await client.query(`
            INSERT INTO transactions (transaction_id, from_account, to_account, amount, memo, status, created_at)
            VALUES ($1, $2, $3, $4, $5, $6, NOW())
          `, [transactionId, fromAccount, toAccount, amount, memo, 'completed']);
          
          await client.query('COMMIT');
          
          console.log(`[${req.requestId}] Transaction ${transactionId} completed successfully`);
          
          // Invalidate cache
          await this.redis.del(`balance:${fromAccount}`, `balance:${toAccount}`);
          
          res.json({
            success: true,
            transactionId,
            fromAccount,
            toAccount,
            amount,
            memo,
            status: 'completed',
            timestamp: new Date().toISOString()
          });
          
        } catch (error) {
          await client.query('ROLLBACK');
          console.error(`[${req.requestId}] Transaction ${transactionId} failed:`, error.message);
          
          // Record failed transaction
          await client.query(`
            INSERT INTO transactions (transaction_id, from_account, to_account, amount, memo, status, error, created_at)
            VALUES ($1, $2, $3, $4, $5, $6, $7, NOW())
          `, [transactionId, fromAccount, toAccount, amount, memo, 'failed', error.message]);
          
          throw error;
        } finally {
          client.release();
        }
        
      } catch (error) {
        res.status(500).json({
          error: 'Transaction failed',
          message: error.message,
          transactionId
        });
      } finally {
        // Remove from active transactions
        this.activeTransactions.delete(transactionId);
        console.log(`[${req.requestId}] Transaction ${transactionId} finalized. Active: ${this.activeTransactions.size}`);
      }
    });

    // Get account balance
    this.app.get('/api/v1/balance/:accountId', async (req, res) => {
      try {
        const { accountId } = req.params;
        
        // Check cache first
        const cached = await this.redis.get(`balance:${accountId}`);
        if (cached) {
          return res.json({
            accountId,
            balance: parseFloat(cached),
            source: 'cache'
          });
        }
        
        // Query database
        const result = await this.db.query(
          'SELECT balance FROM accounts WHERE account_id = $1',
          [accountId]
        );
        
        if (result.rows.length === 0) {
          return res.status(404).json({ error: 'Account not found' });
        }
        
        const balance = parseFloat(result.rows[0].balance);
        
        // Cache for 60 seconds
        await this.redis.setex(`balance:${accountId}`, 60, balance);
        
        res.json({
          accountId,
          balance,
          source: 'database'
        });
        
      } catch (error) {
        console.error('Balance check error:', error);
        res.status(500).json({ error: 'Failed to fetch balance' });
      }
    });

    // Transaction history
    this.app.get('/api/v1/transactions/:accountId', async (req, res) => {
      try {
        const { accountId } = req.params;
        const limit = parseInt(req.query.limit) || 10;
        
        const result = await this.db.query(`
          SELECT transaction_id, from_account, to_account, amount, memo, status, created_at
          FROM transactions
          WHERE from_account = $1 OR to_account = $1
          ORDER BY created_at DESC
          LIMIT $2
        `, [accountId, limit]);
        
        res.json({
          accountId,
          transactions: result.rows,
          count: result.rows.length
        });
        
      } catch (error) {
        console.error('Transaction history error:', error);
        res.status(500).json({ error: 'Failed to fetch transactions' });
      }
    });
  }

  setupHealthChecks() {
    // Database health check
    setInterval(async () => {
      try {
        await this.db.query('SELECT 1');
      } catch (error) {
        console.error('Database health check failed:', error);
      }
    }, 30000);

    // Redis health check
    setInterval(async () => {
      try {
        await this.redis.ping();
      } catch (error) {
        console.error('Redis health check failed:', error);
      }
    }, 30000);
  }

  setupSignalHandlers() {
    const signals = ['SIGTERM', 'SIGINT', 'SIGHUP'];
    
    signals.forEach(signal => {
      process.on(signal, () => this.gracefulShutdown(signal));
    });

    // Handle uncaught errors
    process.on('uncaughtException', (error) => {
      console.error('Uncaught Exception:', error);
      this.gracefulShutdown('uncaughtException');
    });

    process.on('unhandledRejection', (reason, promise) => {
      console.error('Unhandled Rejection at:', promise, 'reason:', reason);
      this.gracefulShutdown('unhandledRejection');
    });
  }

  async gracefulShutdown(signal) {
    if (this.isShuttingDown) {
      console.log('Shutdown already in progress...');
      return;
    }
    
    this.isShuttingDown = true;
    console.log('\n==============================================');
    console.log(`🛑 Received ${signal} - Starting graceful shutdown`);
    console.log('==============================================');
    
    // Force shutdown after timeout
    const forceShutdownTimeout = setTimeout(() => {
      console.error('\n❌ Graceful shutdown timeout - forcing exit');
      console.error(`Active transactions: ${this.activeTransactions.size}`);
      console.error(`Active sockets: ${this.activeSockets.size}`);
      process.exit(1);
    }, 30000); // 30 seconds

    try {
      // Step 1: Stop accepting new connections
      console.log('\n1️⃣ Stopping HTTP server (no new connections)...');
      await new Promise((resolve, reject) => {
        this.server.close((err) => {
          if (err) reject(err);
          else resolve();
        });
      });
      console.log('   ✓ HTTP server stopped accepting new connections');

      // Step 2: Wait for active transactions to complete
      console.log(`\n2️⃣ Waiting for ${this.activeTransactions.size} active transactions to complete...`);
      
      let waitCount = 0;
      while (this.activeTransactions.size > 0 && waitCount < 25) {
        console.log(`   ⏳ ${this.activeTransactions.size} transactions still active...`);
        console.log(`      Active IDs: ${Array.from(this.activeTransactions).slice(0, 5).join(', ')}${this.activeTransactions.size > 5 ? '...' : ''}`);
        await new Promise(resolve => setTimeout(resolve, 1000));
        waitCount++;
      }
      
      if (this.activeTransactions.size > 0) {
        console.warn(`   ⚠️ ${this.activeTransactions.size} transactions still active after waiting`);
      } else {
        console.log('   ✓ All transactions completed');
      }

      // Step 3: Close database connection pool
      console.log('\n3️⃣ Closing database connections...');
      await this.db.end();
      console.log('   ✓ Database connections closed');

      // Step 4: Close Redis connection
      console.log('\n4️⃣ Closing Redis connection...');
      await this.redis.quit();
      console.log('   ✓ Redis connection closed');

      // Step 5: Destroy remaining sockets
      if (this.activeSockets.size > 0) {
        console.log(`\n5️⃣ Destroying ${this.activeSockets.size} remaining sockets...`);
        this.activeSockets.forEach(socket => socket.destroy());
        console.log('   ✓ All sockets destroyed');
      }

      console.log('\n==============================================');
      console.log('✅ Graceful shutdown completed successfully');
      console.log('==============================================\n');
      
      clearTimeout(forceShutdownTimeout);
      process.exit(0);
      
    } catch (error) {
      console.error('\n==============================================');
      console.error('❌ Error during graceful shutdown:', error);
      console.error('==============================================\n');
      
      clearTimeout(forceShutdownTimeout);
      process.exit(1);
    }
  }

  async start() {
    const PORT = process.env.PORT || 3000;
    const HOST = process.env.HOST || '0.0.0.0';
    
    // Initialize database tables (if needed)
    await this.initializeDatabase();
    
    this.server = this.app.listen(PORT, HOST, () => {
      console.log('==============================================');
      console.log('🏦 Banking API Server Started');
      console.log('==============================================');
      console.log(`Server: http://${HOST}:${PORT}`);
      console.log(`PID: ${process.pid}`);
      console.log(`Node.js: ${process.version}`);
      console.log(`Environment: ${process.env.NODE_ENV || 'development'}`);
      console.log('==============================================\n');
      console.log('Send SIGTERM or press Ctrl+C for graceful shutdown\n');
    });

    // Track sockets
    this.server.on('connection', (socket) => {
      this.activeSockets.add(socket);
      
      socket.on('close', () => {
        this.activeSockets.delete(socket);
      });
    });
  }

  async initializeDatabase() {
    try {
      // Create tables if they don't exist
      await this.db.query(`
        CREATE TABLE IF NOT EXISTS accounts (
          account_id VARCHAR(50) PRIMARY KEY,
          account_name VARCHAR(255),
          balance DECIMAL(15, 2) DEFAULT 0,
          created_at TIMESTAMP DEFAULT NOW()
        )
      `);
      
      await this.db.query(`
        CREATE TABLE IF NOT EXISTS transactions (
          transaction_id VARCHAR(100) PRIMARY KEY,
          from_account VARCHAR(50),
          to_account VARCHAR(50),
          amount DECIMAL(15, 2),
          memo TEXT,
          status VARCHAR(20),
          error TEXT,
          created_at TIMESTAMP DEFAULT NOW()
        )
      `);
      
      // Insert sample accounts
      await this.db.query(`
        INSERT INTO accounts (account_id, account_name, balance)
        VALUES 
          ('ACC001', 'John Doe', 10000),
          ('ACC002', 'Jane Smith', 25000),
          ('ACC003', 'Acme Corp', 500000)
        ON CONFLICT (account_id) DO NOTHING
      `);
      
      console.log('✓ Database initialized');
    } catch (error) {
      console.error('Database initialization error:', error);
      throw error;
    }
  }
}

// Start server
if (require.main === module) {
  const api = new BankingAPI();
  api.start().catch(error => {
    console.error('Failed to start server:', error);
    process.exit(1);
  });
}

module.exports = BankingAPI;
```

**Testing Commands**:

```bash
# Install dependencies
npm install express pg ioredis

# Start PostgreSQL and Redis (using Docker)
docker run -d --name postgres -e POSTGRES_PASSWORD=password -p 5432:5432 postgres
docker run -d --name redis -p 6379:6379 redis

# Set environment variables
export DB_HOST=localhost
export DB_PORT=5432
export DB_NAME=postgres
export DB_USER=postgres
export DB_PASSWORD=password
export REDIS_HOST=localhost
export REDIS_PORT=6379

# Start server
node banking-api-graceful-shutdown.js

# In another terminal, test transactions
# Transfer money
curl -X POST http://localhost:3000/api/v1/transfer \
  -H "Content-Type: application/json" \
  -d '{
    "fromAccount": "ACC001",
    "toAccount": "ACC002",
    "amount": 500,
    "memo": "Test transfer"
  }'

# Check balance
curl http://localhost:3000/api/v1/balance/ACC001

# Transaction history
curl http://localhost:3000/api/v1/transactions/ACC001

# Health check
curl http://localhost:3000/health

# Test graceful shutdown while transactions are running
# Start multiple transactions in background
for i in {1..10}; do
  curl -X POST http://localhost:3000/api/v1/transfer \
    -H "Content-Type: application/json" \
    -d "{\"fromAccount\": \"ACC001\", \"toAccount\": \"ACC002\", \"amount\": 100, \"memo\": \"Test $i\"}" &
done

# Immediately send SIGTERM (watch it complete all transactions)
kill -TERM $(pgrep -f "node banking-api")

# Or press Ctrl+C in the server terminal
# Watch the graceful shutdown process

# Test forced shutdown (SIGKILL cannot be caught)
kill -9 $(pgrep -f "node banking-api")
```

---

### Example 2: Signal-Based Configuration Reload (Zero-Downtime)

**Scenario**: Reload application configuration without restarting the server using SIGUSR2 signal.

**File: `config-reload-signal.js`**

```javascript
const express = require('express');
const fs = require('fs').promises;
const path = require('path');

class ConfigurableApp {
  constructor() {
    this.app = express();
    this.config = null;
    this.configPath = path.join(__dirname, 'config.json');
    this.lastReload = null;
    
    this.setupSignals();
    this.setupRoutes();
  }

  async loadConfig() {
    try {
      console.log(`Loading configuration from ${this.configPath}...`);
      const data = await fs.readFile(this.configPath, 'utf8');
      this.config = JSON.parse(data);
      this.lastReload = new Date();
      console.log('✓ Configuration loaded successfully');
      console.log('Config:', JSON.stringify(this.config, null, 2));
      return true;
    } catch (error) {
      console.error('Failed to load configuration:', error.message);
      return false;
    }
  }

  setupSignals() {
    // SIGUSR2 - Reload configuration
    process.on('SIGUSR2', async () => {
      console.log('\n📡 Received SIGUSR2 - Reloading configuration...');
      
      // Backup current config
      const oldConfig = { ...this.config };
      
      const success = await this.loadConfig();
      
      if (success) {
        console.log('✓ Configuration reloaded successfully');
        this.logConfigChanges(oldConfig, this.config);
      } else {
        console.error('✗ Configuration reload failed - keeping old config');
      }
    });

    // SIGUSR1 - Print current configuration
    process.on('SIGUSR1', () => {
      console.log('\n📊 Received SIGUSR1 - Current configuration:');
      console.log('='.repeat(50));
      console.log(JSON.stringify(this.config, null, 2));
      console.log('='.repeat(50));
      console.log(`Last reload: ${this.lastReload ? this.lastReload.toISOString() : 'Never'}`);
    });

    // Graceful shutdown
    process.on('SIGTERM', () => this.shutdown('SIGTERM'));
    process.on('SIGINT', () => this.shutdown('SIGINT'));
  }

  logConfigChanges(oldConfig, newConfig) {
    console.log('\n📝 Configuration Changes:');
    
    const changes = [];
    
    // Check for changed values
    for (const key in newConfig) {
      if (JSON.stringify(oldConfig[key]) !== JSON.stringify(newConfig[key])) {
        changes.push({
          key,
          old: oldConfig[key],
          new: newConfig[key]
        });
      }
    }
    
    if (changes.length === 0) {
      console.log('   No changes detected');
    } else {
      changes.forEach(change => {
        console.log(`   ${change.key}:`);
        console.log(`      Old: ${JSON.stringify(change.old)}`);
        console.log(`      New: ${JSON.stringify(change.new)}`);
      });
    }
  }

  setupRoutes() {
    this.app.use(express.json());

    // Health check
    this.app.get('/health', (req, res) => {
      res.json({
        status: 'healthy',
        configLoaded: this.config !== null,
        lastReload: this.lastReload
      });
    });

    // Get current config
    this.app.get('/config', (req, res) => {
      res.json({
        config: this.config,
        lastReload: this.lastReload
      });
    });

    // Test endpoint using config
    this.app.post('/process-transaction', (req, res) => {
      const { amount } = req.body;
      
      if (!this.config) {
        return res.status(503).json({ error: 'Configuration not loaded' });
      }

      // Use config values
      const maxAmount = this.config.limits.maxTransactionAmount;
      const feePercent = this.config.fees.transactionFeePercent;
      
      if (amount > maxAmount) {
        return res.status(400).json({
          error: 'Amount exceeds limit',
          maxAmount,
          requested: amount
        });
      }

      const fee = amount * (feePercent / 100);
      const total = amount + fee;

      res.json({
        amount,
        fee,
        total,
        limits: this.config.limits,
        processingTime: this.config.performance.processingTimeMs
      });
    });
  }

  async shutdown(signal) {
    console.log(`\n🛑 Received ${signal} - Shutting down...`);
    
    this.server.close(() => {
      console.log('✓ Server closed');
      process.exit(0);
    });

    setTimeout(() => {
      console.error('Forced shutdown');
      process.exit(1);
    }, 10000);
  }

  async start() {
    // Load initial configuration
    const success = await this.loadConfig();
    
    if (!success) {
      console.error('Failed to load initial configuration');
      process.exit(1);
    }

    const PORT = this.config.server.port || 3000;
    
    this.server = this.app.listen(PORT, () => {
      console.log('='.repeat(50));
      console.log('🔧 Configurable Banking App');
      console.log('='.repeat(50));
      console.log(`Server: http://localhost:${PORT}`);
      console.log(`PID: ${process.pid}`);
      console.log('='.repeat(50));
      console.log('\nSignal Commands:');
      console.log(`  kill -USR2 ${process.pid}  # Reload configuration`);
      console.log(`  kill -USR1 ${process.pid}  # Print current config`);
      console.log(`  kill -TERM ${process.pid}  # Graceful shutdown`);
      console.log('='.repeat(50));
    });
  }
}

// Start app
if (require.main === module) {
  const app = new ConfigurableApp();
  app.start();
}

module.exports = ConfigurableApp;
```

**Configuration File (`config.json`)**:

```json
{
  "server": {
    "port": 3000,
    "environment": "development"
  },
  "limits": {
    "maxTransactionAmount": 50000,
    "dailyTransferLimit": 200000,
    "maxTransactionsPerHour": 100
  },
  "fees": {
    "transactionFeePercent": 0.5,
    "internationalFeePercent": 2.0
  },
  "performance": {
    "processingTimeMs": 2000,
    "timeoutMs": 30000
  },
  "features": {
    "enableFraudDetection": true,
    "enableNotifications": false
  }
}
```

**Testing Commands**:

```bash
# Start server
node config-reload-signal.js

# Get PID
PID=$(pgrep -f "node config-reload-signal")
echo "Server PID: $PID"

# Test transaction with current config
curl -X POST http://localhost:3000/process-transaction \
  -H "Content-Type: application/json" \
  -d '{"amount": 10000}'

# Get current config
curl http://localhost:3000/config

# Print current config (SIGUSR1)
kill -USR1 $PID

# Edit config.json (change maxTransactionAmount to 100000)
# Then reload without restarting server
kill -USR2 $PID

# Test with new limit
curl -X POST http://localhost:3000/process-transaction \
  -H "Content-Type: application/json" \
  -d '{"amount": 75000}'

# Graceful shutdown
kill -TERM $PID
```

---

### Example 3: Process Manager with Signal Handling (Like PM2)

**Scenario**: Build a simple process manager that handles worker processes, restarts on failure, and manages graceful shutdowns.

**File: `process-manager.js`**

```javascript
const { fork } = require('child_process');
const path = require('path');

class ProcessManager {
  constructor(scriptPath, options = {}) {
    this.scriptPath = scriptPath;
    this.maxRestarts = options.maxRestarts || 5;
    this.restartDelay = options.restartDelay || 1000;
    this.workers = new Map();
    this.restartCount = 0;
    this.isShuttingDown = false;
    
    this.setupSignals();
  }

  setupSignals() {
    // Graceful shutdown
    process.on('SIGTERM', () => this.shutdown('SIGTERM'));
    process.on('SIGINT', () => this.shutdown('SIGINT'));
    
    // Reload all workers
    process.on('SIGHUP', () => this.reloadWorkers());
    
    // Restart all workers
    process.on('SIGUSR2', () => this.restartAll());
  }

  async startWorker(workerId) {
    console.log(`[Manager] Starting worker ${workerId}...`);
    
    const worker = fork(this.scriptPath, [], {
      env: {
        ...process.env,
        WORKER_ID: workerId,
        WORKER_PORT: 3000 + workerId
      }
    });

    worker.workerId = workerId;
    worker.startTime = Date.now();
    
    this.workers.set(workerId, worker);

    worker.on('message', (msg) => {
      console.log(`[Worker ${workerId}] Message:`, msg);
    });

    worker.on('exit', (code, signal) => {
      console.log(`[Worker ${workerId}] Exited with code ${code}, signal ${signal}`);
      this.workers.delete(workerId);
      
      if (!this.isShuttingDown && this.restartCount < this.maxRestarts) {
        console.log(`[Manager] Restarting worker ${workerId} in ${this.restartDelay}ms...`);
        this.restartCount++;
        
        setTimeout(() => {
          this.startWorker(workerId);
        }, this.restartDelay);
      } else if (this.restartCount >= this.maxRestarts) {
        console.error(`[Manager] Max restarts (${this.maxRestarts}) reached for worker ${workerId}`);
      }
    });

    worker.on('error', (error) => {
      console.error(`[Worker ${workerId}] Error:`, error);
    });

    console.log(`[Manager] Worker ${workerId} started (PID: ${worker.pid})`);
  }

  async startAll(count = 4) {
    console.log(`[Manager] Starting ${count} workers...`);
    
    for (let i = 0; i < count; i++) {
      await this.startWorker(i);
      await new Promise(resolve => setTimeout(resolve, 500));
    }
    
    console.log(`[Manager] All ${count} workers started`);
    this.printStatus();
  }

  async reloadWorkers() {
    console.log('\n[Manager] 🔄 Reloading all workers (zero-downtime)...');
    
    const workerIds = Array.from(this.workers.keys());
    
    for (const workerId of workerIds) {
      console.log(`[Manager] Reloading worker ${workerId}...`);
      
      const oldWorker = this.workers.get(workerId);
      
      // Start new worker first
      const newWorkerId = workerId + 1000; // Temporary ID
      await this.startWorker(newWorkerId);
      
      // Wait for new worker to be ready
      await new Promise(resolve => setTimeout(resolve, 2000));
      
      // Gracefully shutdown old worker
      console.log(`[Manager] Shutting down old worker ${workerId}...`);
      oldWorker.send({ type: 'shutdown' });
      
      // Wait for old worker to exit
      await new Promise(resolve => {
        oldWorker.once('exit', resolve);
        setTimeout(() => {
          console.log(`[Manager] Force killing old worker ${workerId}`);
          oldWorker.kill('SIGKILL');
          resolve();
        }, 10000);
      });
      
      // Rename new worker to old ID
      const newWorker = this.workers.get(newWorkerId);
      this.workers.delete(newWorkerId);
      newWorker.workerId = workerId;
      this.workers.set(workerId, newWorker);
      
      console.log(`[Manager] Worker ${workerId} reloaded successfully`);
    }
    
    console.log('[Manager] ✓ All workers reloaded');
    this.printStatus();
  }

  async restartAll() {
    console.log('\n[Manager] 🔄 Restarting all workers...');
    
    for (const [workerId, worker] of this.workers) {
      console.log(`[Manager] Stopping worker ${workerId}...`);
      worker.kill('SIGTERM');
    }
    
    // Wait for all to stop
    await new Promise(resolve => setTimeout(resolve, 5000));
    
    // Start new workers
    const workerCount = this.workers.size || 4;
    await this.startAll(workerCount);
    
    console.log('[Manager] ✓ All workers restarted');
  }

  async shutdown(signal) {
    if (this.isShuttingDown) {
      console.log('[Manager] Shutdown already in progress...');
      return;
    }
    
    this.isShuttingDown = true;
    
    console.log(`\n[Manager] 🛑 Received ${signal} - Shutting down all workers...`);
    
    const shutdownPromises = [];
    
    for (const [workerId, worker] of this.workers) {
      console.log(`[Manager] Shutting down worker ${workerId} (PID: ${worker.pid})...`);
      
      const promise = new Promise((resolve) => {
        worker.once('exit', () => {
          console.log(`[Manager] Worker ${workerId} shut down`);
          resolve();
        });
        
        worker.send({ type: 'shutdown' });
        
        // Force kill after timeout
        setTimeout(() => {
          if (!worker.killed) {
            console.log(`[Manager] Force killing worker ${workerId}`);
            worker.kill('SIGKILL');
            resolve();
          }
        }, 10000);
      });
      
      shutdownPromises.push(promise);
    }
    
    await Promise.all(shutdownPromises);
    
    console.log('[Manager] ✓ All workers shut down');
    console.log('[Manager] Exiting...');
    process.exit(0);
  }

  printStatus() {
    console.log('\n' + '='.repeat(60));
    console.log('Process Manager Status');
    console.log('='.repeat(60));
    console.log(`Manager PID: ${process.pid}`);
    console.log(`Active Workers: ${this.workers.size}`);
    console.log(`Restart Count: ${this.restartCount}`);
    console.log('\nWorkers:');
    
    for (const [workerId, worker] of this.workers) {
      const uptime = Math.floor((Date.now() - worker.startTime) / 1000);
      console.log(`  Worker ${workerId}: PID ${worker.pid}, Uptime: ${uptime}s`);
    }
    
    console.log('='.repeat(60));
    console.log('\nSignal Commands:');
    console.log(`  kill -HUP ${process.pid}   # Reload workers (zero-downtime)`);
    console.log(`  kill -USR2 ${process.pid}  # Restart all workers`);
    console.log(`  kill -TERM ${process.pid}  # Graceful shutdown`);
    console.log('='.repeat(60) + '\n');
  }

  start(workerCount = 4) {
    console.log('🚀 Process Manager Starting...\n');
    this.startAll(workerCount);
  }
}

// Example worker script
if (require.main === module) {
  const workerScript = path.join(__dirname, 'banking-api-graceful-shutdown.js');
  
  const manager = new ProcessManager(workerScript, {
    maxRestarts: 5,
    restartDelay: 2000
  });
  
  manager.start(4);
}

module.exports = ProcessManager;
```

**Testing Commands**:

```bash
# Start process manager
node process-manager.js

# Get manager PID
PM_PID=$(pgrep -f "node process-manager")
echo "Manager PID: $PM_PID"

# Check worker processes
ps aux | grep "banking-api"

# Reload workers (zero-downtime)
kill -HUP $PM_PID

# Restart all workers
kill -USR2 $PM_PID

# Graceful shutdown
kill -TERM $PM_PID

# Test worker crash recovery
# Kill a worker process manually
WORKER_PID=$(pgrep -f "WORKER_ID=0")
kill -9 $WORKER_PID
# Watch it automatically restart
```

---

## 🎯 DO's

1. **✅ DO** handle SIGTERM and SIGINT for graceful shutdowns
2. **✅ DO** stop accepting new connections immediately when shutting down
3. **✅ DO** wait for in-flight requests to complete before exiting
4. **✅ DO** implement shutdown timeouts to prevent hanging
5. **✅ DO** close database connections properly during shutdown
6. **✅ DO** log all signal events for debugging
7. **✅ DO** return appropriate exit codes (0 = success, non-zero = error)
8. **✅ DO** test shutdown behavior in development
9. **✅ DO** use SIGUSR1/SIGUSR2 for custom behaviors (config reload, debug info)
10. **✅ DO** track active requests/transactions during shutdown

---

## 🚫 DON'Ts

1. **❌ DON'T** ignore SIGTERM in production applications
2. **❌ DON'T** call `process.exit()` without cleanup
3. **❌ DON'T** perform long-running operations in signal handlers
4. **❌ DON'T** forget to close database connections before exit
5. **❌ DON'T** leave sockets open after shutdown
6. **❌ DON'T** lose in-flight transactions during shutdown
7. **❌ DON'T** use synchronous operations in shutdown handlers
8. **❌ DON'T** rely on SIGKILL for normal shutdowns (can't be caught)
9. **❌ DON'T** have multiple shutdown handlers that conflict
10. **❌ DON'T** forget to test graceful shutdown under load

---

## 🎓 Key Takeaways

1. **SIGTERM = Graceful** - Always implement graceful shutdown for SIGTERM
2. **SIGINT = Ctrl+C** - Handle for development and manual stops
3. **SIGKILL cannot be caught** - Always exits immediately, no cleanup
4. **Connection draining** - Complete in-flight requests before exit
5. **Shutdown timeout** - Force exit after reasonable timeout (30s typical)
6. **Database cleanup** - Close connections to prevent connection leaks
7. **Zero-downtime** - Use rolling restarts or reload workers individually
8. **Custom signals** - SIGUSR1/SIGUSR2 for config reload, debug dumps
9. **Exit codes** - Use proper codes for monitoring and alerting
10. **Production-ready** - Test shutdown behavior with real load

---

**Next Question**: [Q23: Memory Management - Heap & Stack](./Q23_Memory_Management_Heap_Stack.md)

**Related Questions**:
- Q19: Process Management - process object
- Q21: Clustering
- Q22: Child Processes
- Q59: Production Best Practices

---

**File**: `Q20_Process_Management_Signals.md`  
**Status**: ✅ Complete (2,300+ lines)
