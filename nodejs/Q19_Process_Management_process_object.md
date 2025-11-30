# Q19: Process Management - process object

## 📋 Summary
This question covers the **Node.js `process` object** - a global object that provides information about and control over the current Node.js process. We'll explore `process.env`, `process.argv`, `process.cwd()`, exit codes, and build production-ready banking applications with environment-based configuration, secure credential management, and command-line interfaces.

---

## 🎯 What You'll Learn
- Understanding the `process` global object
- Working with environment variables (`process.env`)
- Parsing command-line arguments (`process.argv`)
- Managing working directories (`process.cwd()`, `process.chdir()`)
- Exit codes and process termination
- Platform information (`process.platform`, `process.arch`)
- Environment-based configuration patterns
- Banking Examples: Multi-environment API, secure config management, CLI tools

---

## 📖 Detailed Explanation

### What is the Process Object?

The **`process`** object is a **global object** available in all Node.js applications without requiring imports. It provides:
- Information about the current Node.js process
- Methods to control the process
- Event emitters for process lifecycle events

```javascript
// process is globally available
console.log(process.pid);        // Process ID
console.log(process.platform);   // 'win32', 'linux', 'darwin'
console.log(process.version);    // Node.js version
console.log(process.versions);   // V8, OpenSSL versions
```

---

### 1. process.env - Environment Variables

**`process.env`** is an object containing the user environment variables. It's commonly used for configuration management.

```javascript
// Reading environment variables
const dbHost = process.env.DB_HOST || 'localhost';
const dbPort = process.env.DB_PORT || 5432;
const nodeEnv = process.env.NODE_ENV || 'development';

console.log(`Environment: ${nodeEnv}`);
console.log(`Database: ${dbHost}:${dbPort}`);
```

**Common Environment Variables**:
- `NODE_ENV`: 'development', 'staging', 'production'
- `PORT`: Server port number
- `DB_HOST`, `DB_PORT`, `DB_NAME`: Database configuration
- `API_KEY`, `SECRET_KEY`: Credentials (should be kept secret)

**Setting Environment Variables**:

```bash
# Linux/Mac
export NODE_ENV=production
export DB_HOST=prod-db.example.com
node app.js

# Windows (PowerShell)
$env:NODE_ENV="production"
$env:DB_HOST="prod-db.example.com"
node app.js

# Inline (works on all platforms)
NODE_ENV=production DB_HOST=prod-db.example.com node app.js
```

**Using .env Files** (with dotenv package):

```javascript
// Install: npm install dotenv
require('dotenv').config();

// Now read from .env file
const apiKey = process.env.API_KEY;
const dbPassword = process.env.DB_PASSWORD;
```

---

### 2. process.argv - Command Line Arguments

**`process.argv`** is an array containing command-line arguments passed to the Node.js process.

```javascript
// node app.js --port 3000 --env production
console.log(process.argv);

// Output:
// [
//   '/usr/local/bin/node',           // [0] Node executable path
//   '/path/to/app.js',                // [1] Script path
//   '--port',                         // [2] First argument
//   '3000',                           // [3] Second argument
//   '--env',                          // [4] Third argument
//   'production'                      // [5] Fourth argument
// ]
```

**Parsing Arguments**:

```javascript
// Simple manual parsing
const args = process.argv.slice(2); // Remove first 2 elements

const getArg = (flag) => {
  const index = args.indexOf(flag);
  return index !== -1 && args[index + 1] ? args[index + 1] : null;
};

const port = getArg('--port') || 3000;
const env = getArg('--env') || 'development';

console.log(`Starting server on port ${port} in ${env} mode`);
```

**Using Commander.js** (recommended for complex CLIs):

```javascript
const { program } = require('commander');

program
  .option('-p, --port <number>', 'port number', 3000)
  .option('-e, --env <type>', 'environment', 'development')
  .parse(process.argv);

const options = program.opts();
console.log(`Port: ${options.port}, Environment: ${options.env}`);
```

---

### 3. process.cwd() - Current Working Directory

**`process.cwd()`** returns the current working directory of the Node.js process.

```javascript
console.log('Current directory:', process.cwd());
// Output: C:\Projects\banking-api

// Change directory
process.chdir('/tmp');
console.log('New directory:', process.cwd());
// Output: /tmp
```

**Common Use Cases**:
```javascript
const path = require('path');
const fs = require('fs');

// Read file relative to current directory
const configPath = path.join(process.cwd(), 'config', 'database.json');
const config = JSON.parse(fs.readFileSync(configPath, 'utf8'));

// vs __dirname (directory of current script file)
const scriptDir = __dirname;
console.log('Script location:', scriptDir);
console.log('Working directory:', process.cwd());
```

---

### 4. Exit Codes

**`process.exit([code])`** terminates the process with an optional exit code.

**Standard Exit Codes**:
- **0**: Success (default)
- **1**: General error
- **2**: Misuse of shell command
- **3-125**: Custom error codes
- **126**: Command cannot execute
- **127**: Command not found
- **128+n**: Fatal signal (e.g., 130 = SIGINT)

```javascript
// Successful exit
process.exit(0);

// Error exit
process.exit(1);

// With cleanup before exit
process.on('beforeExit', (code) => {
  console.log('Process about to exit with code:', code);
  // Do async cleanup here
});

process.on('exit', (code) => {
  // Only synchronous operations here
  console.log('Process exiting with code:', code);
});
```

**Graceful Exit**:
```javascript
async function gracefulShutdown() {
  console.log('Shutting down gracefully...');
  
  // Close database connections
  await db.close();
  
  // Close server
  server.close(() => {
    console.log('Server closed');
    process.exit(0);
  });
  
  // Force exit after timeout
  setTimeout(() => {
    console.error('Forcing shutdown after timeout');
    process.exit(1);
  }, 10000);
}

process.on('SIGTERM', gracefulShutdown);
process.on('SIGINT', gracefulShutdown);
```

---

### 5. Platform and Architecture Information

```javascript
// Platform information
console.log('Platform:', process.platform);
// 'darwin' (macOS), 'win32' (Windows), 'linux', 'freebsd', etc.

console.log('Architecture:', process.arch);
// 'x64', 'arm64', 'arm', 'ia32', etc.

console.log('Node version:', process.version);
// 'v18.16.0'

console.log('Versions:', process.versions);
// { node: '18.16.0', v8: '10.2.154.26', uv: '1.44.2', ... }

// Platform-specific code
if (process.platform === 'win32') {
  console.log('Running on Windows');
} else if (process.platform === 'darwin') {
  console.log('Running on macOS');
} else {
  console.log('Running on Linux');
}
```

---

### 6. Process Resource Usage

```javascript
// Memory usage
const memUsage = process.memoryUsage();
console.log('Memory Usage:', {
  rss: `${Math.round(memUsage.rss / 1024 / 1024)}MB`,          // Resident Set Size
  heapTotal: `${Math.round(memUsage.heapTotal / 1024 / 1024)}MB`,
  heapUsed: `${Math.round(memUsage.heapUsed / 1024 / 1024)}MB`,
  external: `${Math.round(memUsage.external / 1024 / 1024)}MB`
});

// CPU usage
const cpuUsage = process.cpuUsage();
console.log('CPU Usage:', {
  user: `${cpuUsage.user / 1000}ms`,      // User CPU time
  system: `${cpuUsage.system / 1000}ms`   // System CPU time
});

// Uptime
console.log('Process uptime:', Math.floor(process.uptime()), 'seconds');
```

---

### 7. Process Standard Streams

```javascript
// STDIN (Standard Input) - readable stream
process.stdin.on('data', (data) => {
  console.log('Received:', data.toString());
});

// STDOUT (Standard Output) - writable stream
process.stdout.write('This is stdout\n');

// STDERR (Standard Error) - writable stream
process.stderr.write('This is an error\n');

// Check if running in terminal
console.log('Is TTY (Terminal)?', process.stdout.isTTY);
```

---

## 🏦 Banking Application Examples

### Example 1: Multi-Environment Banking API Configuration

**Scenario**: Build a banking API that adapts configuration based on environment (development, staging, production) with secure credential management.

**File: `config/environment-manager.js`**

```javascript
const path = require('path');
const fs = require('fs');

class EnvironmentManager {
  constructor() {
    this.env = process.env.NODE_ENV || 'development';
    this.config = this.loadConfig();
    this.validateConfig();
  }

  loadConfig() {
    const configFile = path.join(
      process.cwd(),
      'config',
      `${this.env}.json`
    );

    if (!fs.existsSync(configFile)) {
      console.error(`Configuration file not found: ${configFile}`);
      process.exit(1);
    }

    const config = JSON.parse(fs.readFileSync(configFile, 'utf8'));

    // Override with environment variables (more secure for secrets)
    return {
      environment: this.env,
      server: {
        port: parseInt(process.env.PORT) || config.server.port,
        host: process.env.HOST || config.server.host,
        apiVersion: config.server.apiVersion
      },
      database: {
        host: process.env.DB_HOST || config.database.host,
        port: parseInt(process.env.DB_PORT) || config.database.port,
        name: process.env.DB_NAME || config.database.name,
        user: process.env.DB_USER || config.database.user,
        password: process.env.DB_PASSWORD || config.database.password,
        ssl: process.env.DB_SSL === 'true' || config.database.ssl,
        poolMin: parseInt(process.env.DB_POOL_MIN) || config.database.poolMin,
        poolMax: parseInt(process.env.DB_POOL_MAX) || config.database.poolMax
      },
      redis: {
        host: process.env.REDIS_HOST || config.redis.host,
        port: parseInt(process.env.REDIS_PORT) || config.redis.port,
        password: process.env.REDIS_PASSWORD || config.redis.password,
        ttl: parseInt(process.env.REDIS_TTL) || config.redis.ttl
      },
      security: {
        jwtSecret: process.env.JWT_SECRET || config.security.jwtSecret,
        jwtExpiry: process.env.JWT_EXPIRY || config.security.jwtExpiry,
        encryptionKey: process.env.ENCRYPTION_KEY || config.security.encryptionKey,
        bcryptRounds: parseInt(process.env.BCRYPT_ROUNDS) || config.security.bcryptRounds
      },
      external: {
        paymentGateway: {
          url: process.env.PAYMENT_GATEWAY_URL || config.external.paymentGateway.url,
          apiKey: process.env.PAYMENT_API_KEY || config.external.paymentGateway.apiKey,
          timeout: parseInt(process.env.PAYMENT_TIMEOUT) || config.external.paymentGateway.timeout
        },
        fraudDetection: {
          url: process.env.FRAUD_API_URL || config.external.fraudDetection.url,
          apiKey: process.env.FRAUD_API_KEY || config.external.fraudDetection.apiKey
        }
      },
      logging: {
        level: process.env.LOG_LEVEL || config.logging.level,
        format: process.env.LOG_FORMAT || config.logging.format,
        destination: process.env.LOG_DESTINATION || config.logging.destination
      },
      features: {
        enableTransactionLimits: process.env.ENABLE_TX_LIMITS === 'true' || config.features.enableTransactionLimits,
        enableFraudDetection: process.env.ENABLE_FRAUD === 'true' || config.features.enableFraudDetection,
        enableNotifications: process.env.ENABLE_NOTIFICATIONS === 'true' || config.features.enableNotifications,
        maxDailyTransferLimit: parseFloat(process.env.MAX_DAILY_LIMIT) || config.features.maxDailyTransferLimit
      },
      monitoring: {
        enableMetrics: process.env.ENABLE_METRICS === 'true' || config.monitoring.enableMetrics,
        metricsPort: parseInt(process.env.METRICS_PORT) || config.monitoring.metricsPort,
        healthCheckInterval: parseInt(process.env.HEALTH_CHECK_INTERVAL) || config.monitoring.healthCheckInterval
      }
    };
  }

  validateConfig() {
    const required = [
      'server.port',
      'database.host',
      'database.name',
      'security.jwtSecret'
    ];

    for (const key of required) {
      if (!this.getNestedValue(this.config, key)) {
        console.error(`Missing required configuration: ${key}`);
        process.exit(1);
      }
    }

    // Production-specific validations
    if (this.env === 'production') {
      if (!process.env.DB_PASSWORD) {
        console.error('DB_PASSWORD must be set via environment variable in production');
        process.exit(1);
      }
      if (!process.env.JWT_SECRET) {
        console.error('JWT_SECRET must be set via environment variable in production');
        process.exit(1);
      }
      if (!this.config.database.ssl) {
        console.warn('WARNING: Database SSL is not enabled in production!');
      }
    }

    console.log(`✓ Configuration validated for ${this.env} environment`);
  }

  getNestedValue(obj, path) {
    return path.split('.').reduce((current, key) => current?.[key], obj);
  }

  get(key) {
    return this.getNestedValue(this.config, key);
  }

  isDevelopment() {
    return this.env === 'development';
  }

  isProduction() {
    return this.env === 'production';
  }

  isStaging() {
    return this.env === 'staging';
  }

  printSafeConfig() {
    // Print config without sensitive data
    const safeConfig = JSON.parse(JSON.stringify(this.config));
    
    // Mask sensitive fields
    if (safeConfig.database.password) safeConfig.database.password = '***';
    if (safeConfig.redis.password) safeConfig.redis.password = '***';
    if (safeConfig.security.jwtSecret) safeConfig.security.jwtSecret = '***';
    if (safeConfig.security.encryptionKey) safeConfig.security.encryptionKey = '***';
    if (safeConfig.external.paymentGateway.apiKey) safeConfig.external.paymentGateway.apiKey = '***';
    if (safeConfig.external.fraudDetection.apiKey) safeConfig.external.fraudDetection.apiKey = '***';

    console.log('Current Configuration:');
    console.log(JSON.stringify(safeConfig, null, 2));
  }

  getResourceUsage() {
    const memUsage = process.memoryUsage();
    const cpuUsage = process.cpuUsage();
    
    return {
      pid: process.pid,
      platform: process.platform,
      nodeVersion: process.version,
      uptime: Math.floor(process.uptime()),
      memory: {
        rss: Math.round(memUsage.rss / 1024 / 1024),
        heapTotal: Math.round(memUsage.heapTotal / 1024 / 1024),
        heapUsed: Math.round(memUsage.heapUsed / 1024 / 1024),
        external: Math.round(memUsage.external / 1024 / 1024)
      },
      cpu: {
        user: Math.round(cpuUsage.user / 1000),
        system: Math.round(cpuUsage.system / 1000)
      }
    };
  }
}

module.exports = EnvironmentManager;
```

**Configuration Files**:

**`config/development.json`**:
```json
{
  "server": {
    "port": 3000,
    "host": "localhost",
    "apiVersion": "v1"
  },
  "database": {
    "host": "localhost",
    "port": 5432,
    "name": "banking_dev",
    "user": "dev_user",
    "password": "dev_password",
    "ssl": false,
    "poolMin": 2,
    "poolMax": 10
  },
  "redis": {
    "host": "localhost",
    "port": 6379,
    "password": null,
    "ttl": 300
  },
  "security": {
    "jwtSecret": "dev-secret-key-change-in-production",
    "jwtExpiry": "24h",
    "encryptionKey": "dev-encryption-key-32-chars!!",
    "bcryptRounds": 10
  },
  "external": {
    "paymentGateway": {
      "url": "https://sandbox.payment-gateway.com",
      "apiKey": "sandbox_key_12345",
      "timeout": 5000
    },
    "fraudDetection": {
      "url": "https://sandbox.fraud-api.com",
      "apiKey": "sandbox_fraud_key"
    }
  },
  "logging": {
    "level": "debug",
    "format": "json",
    "destination": "console"
  },
  "features": {
    "enableTransactionLimits": false,
    "enableFraudDetection": false,
    "enableNotifications": false,
    "maxDailyTransferLimit": 100000
  },
  "monitoring": {
    "enableMetrics": false,
    "metricsPort": 9090,
    "healthCheckInterval": 30000
  }
}
```

**`config/production.json`**:
```json
{
  "server": {
    "port": 8080,
    "host": "0.0.0.0",
    "apiVersion": "v1"
  },
  "database": {
    "host": "prod-banking-db.cluster-xyz.us-east-1.rds.amazonaws.com",
    "port": 5432,
    "name": "banking_prod",
    "user": "prod_app_user",
    "password": "WILL_BE_OVERRIDDEN_BY_ENV",
    "ssl": true,
    "poolMin": 10,
    "poolMax": 50
  },
  "redis": {
    "host": "prod-redis.cache.amazonaws.com",
    "port": 6379,
    "password": "WILL_BE_OVERRIDDEN_BY_ENV",
    "ttl": 600
  },
  "security": {
    "jwtSecret": "MUST_BE_SET_VIA_ENV",
    "jwtExpiry": "1h",
    "encryptionKey": "MUST_BE_SET_VIA_ENV",
    "bcryptRounds": 12
  },
  "external": {
    "paymentGateway": {
      "url": "https://api.payment-gateway.com",
      "apiKey": "MUST_BE_SET_VIA_ENV",
      "timeout": 10000
    },
    "fraudDetection": {
      "url": "https://api.fraud-detection.com",
      "apiKey": "MUST_BE_SET_VIA_ENV"
    }
  },
  "logging": {
    "level": "info",
    "format": "json",
    "destination": "cloudwatch"
  },
  "features": {
    "enableTransactionLimits": true,
    "enableFraudDetection": true,
    "enableNotifications": true,
    "maxDailyTransferLimit": 50000
  },
  "monitoring": {
    "enableMetrics": true,
    "metricsPort": 9090,
    "healthCheckInterval": 15000
  }
}
```

**`app.js` - Main Application**:

```javascript
const express = require('express');
const EnvironmentManager = require('./config/environment-manager');

// Initialize environment configuration
const envManager = new EnvironmentManager();
const config = envManager.config;

const app = express();
app.use(express.json());

// Health check endpoint
app.get('/health', (req, res) => {
  const resourceUsage = envManager.getResourceUsage();
  
  res.json({
    status: 'healthy',
    environment: config.environment,
    timestamp: new Date().toISOString(),
    ...resourceUsage
  });
});

// Configuration endpoint (safe version)
app.get('/api/v1/config', (req, res) => {
  res.json({
    environment: config.environment,
    apiVersion: config.server.apiVersion,
    features: config.features,
    platform: process.platform,
    nodeVersion: process.version
  });
});

// Transaction endpoint (uses environment config)
app.post('/api/v1/transactions', async (req, res) => {
  try {
    const { amount, fromAccount, toAccount } = req.body;

    // Check transaction limits (feature flag from config)
    if (config.features.enableTransactionLimits) {
      if (amount > config.features.maxDailyTransferLimit) {
        return res.status(400).json({
          error: 'Transaction exceeds daily limit',
          limit: config.features.maxDailyTransferLimit
        });
      }
    }

    // Fraud detection (if enabled)
    if (config.features.enableFraudDetection) {
      console.log('Fraud detection enabled, checking transaction...');
      // Call fraud detection API using config.external.fraudDetection
    }

    res.json({
      success: true,
      transactionId: `TXN-${Date.now()}`,
      amount,
      message: 'Transaction processed successfully'
    });

  } catch (error) {
    console.error('Transaction error:', error);
    res.status(500).json({ error: 'Transaction failed' });
  }
});

// Graceful shutdown
const gracefulShutdown = async () => {
  console.log('\n🛑 Received shutdown signal');
  console.log('Closing server gracefully...');
  
  server.close(() => {
    console.log('✓ Server closed');
    console.log('✓ Shutdown complete');
    process.exit(0);
  });

  // Force exit after 10 seconds
  setTimeout(() => {
    console.error('❌ Forced shutdown after timeout');
    process.exit(1);
  }, 10000);
};

process.on('SIGTERM', gracefulShutdown);
process.on('SIGINT', gracefulShutdown);

// Start server
const server = app.listen(config.server.port, config.server.host, () => {
  console.log('=================================================');
  console.log('🏦 Banking API Server');
  console.log('=================================================');
  console.log(`Environment: ${config.environment}`);
  console.log(`Server: http://${config.server.host}:${config.server.port}`);
  console.log(`Process ID: ${process.pid}`);
  console.log(`Platform: ${process.platform} (${process.arch})`);
  console.log(`Node.js: ${process.version}`);
  console.log(`Working Directory: ${process.cwd()}`);
  console.log('=================================================');
  
  if (!envManager.isProduction()) {
    envManager.printSafeConfig();
  }
});
```

**Testing Commands**:

```bash
# Development mode (default)
node app.js

# Production mode
NODE_ENV=production \
DB_PASSWORD=secure_prod_password \
JWT_SECRET=super_secret_jwt_key_production \
ENCRYPTION_KEY=32_character_encryption_key! \
PAYMENT_API_KEY=prod_payment_key_xyz \
FRAUD_API_KEY=prod_fraud_key_abc \
node app.js

# Using .env file (recommended)
# Create .env.production file:
# NODE_ENV=production
# DB_PASSWORD=secure_prod_password
# JWT_SECRET=super_secret_jwt_key_production
# ...

# Then run:
node -r dotenv/config app.js dotenv_config_path=.env.production

# Test health endpoint
curl http://localhost:3000/health

# Test configuration endpoint
curl http://localhost:3000/api/v1/config

# Test transaction (within limit)
curl -X POST http://localhost:3000/api/v1/transactions \
  -H "Content-Type: application/json" \
  -d '{"amount": 5000, "fromAccount": "ACC001", "toAccount": "ACC002"}'

# Test transaction (exceeding limit in production)
curl -X POST http://localhost:3000/api/v1/transactions \
  -H "Content-Type: application/json" \
  -d '{"amount": 100000, "fromAccount": "ACC001", "toAccount": "ACC002"}'
```

---

### Example 2: Banking CLI Tool with Command-Line Arguments

**Scenario**: Build a command-line tool for bank administrators to manage accounts, transactions, and generate reports.

**File: `cli/banking-cli.js`**

```javascript
const { program } = require('commander');
const chalk = require('chalk');
const Table = require('cli-table3');

class BankingCLI {
  constructor() {
    this.version = '1.0.0';
    this.setupCommands();
  }

  setupCommands() {
    program
      .name('banking-cli')
      .description('Banking Administration CLI Tool')
      .version(this.version);

    // Account commands
    program
      .command('account:create')
      .description('Create a new bank account')
      .requiredOption('-n, --name <name>', 'Account holder name')
      .requiredOption('-t, --type <type>', 'Account type (savings, checking, business)')
      .option('-b, --balance <amount>', 'Initial balance', '0')
      .action((options) => this.createAccount(options));

    program
      .command('account:list')
      .description('List all accounts')
      .option('-t, --type <type>', 'Filter by account type')
      .option('-l, --limit <number>', 'Limit results', '10')
      .action((options) => this.listAccounts(options));

    program
      .command('account:balance')
      .description('Check account balance')
      .requiredOption('-a, --account <id>', 'Account ID')
      .action((options) => this.checkBalance(options));

    // Transaction commands
    program
      .command('transaction:transfer')
      .description('Transfer money between accounts')
      .requiredOption('-f, --from <account>', 'Source account ID')
      .requiredOption('-t, --to <account>', 'Destination account ID')
      .requiredOption('-a, --amount <number>', 'Transfer amount')
      .option('-m, --memo <text>', 'Transaction memo')
      .action((options) => this.transferMoney(options));

    program
      .command('transaction:history')
      .description('View transaction history')
      .requiredOption('-a, --account <id>', 'Account ID')
      .option('-d, --days <number>', 'Number of days', '30')
      .option('--format <type>', 'Output format (table, json, csv)', 'table')
      .action((options) => this.transactionHistory(options));

    // Report commands
    program
      .command('report:daily')
      .description('Generate daily transaction report')
      .option('-d, --date <date>', 'Date (YYYY-MM-DD)', new Date().toISOString().split('T')[0])
      .option('--export <path>', 'Export to file')
      .action((options) => this.dailyReport(options));

    program
      .command('report:summary')
      .description('Generate account summary')
      .option('--csv', 'Export as CSV')
      .action((options) => this.accountSummary(options));

    // Admin commands
    program
      .command('admin:stats')
      .description('Show system statistics')
      .action(() => this.systemStats());

    program
      .command('admin:cleanup')
      .description('Clean up old transaction logs')
      .option('-d, --days <number>', 'Keep logs for N days', '90')
      .option('--dry-run', 'Preview without deleting')
      .action((options) => this.cleanup(options));
  }

  // Mock database (in real app, this would be a database)
  getAccounts() {
    return [
      { id: 'ACC001', name: 'John Doe', type: 'checking', balance: 15000 },
      { id: 'ACC002', name: 'Jane Smith', type: 'savings', balance: 50000 },
      { id: 'ACC003', name: 'Acme Corp', type: 'business', balance: 250000 },
      { id: 'ACC004', name: 'Bob Johnson', type: 'checking', balance: 8500 }
    ];
  }

  getTransactions(accountId, days) {
    const mockTransactions = [
      { id: 'TXN001', date: '2025-11-14', type: 'debit', amount: 500, memo: 'ATM Withdrawal' },
      { id: 'TXN002', date: '2025-11-13', type: 'credit', amount: 2000, memo: 'Salary Deposit' },
      { id: 'TXN003', date: '2025-11-12', type: 'debit', amount: 150, memo: 'Restaurant' },
      { id: 'TXN004', date: '2025-11-10', type: 'credit', amount: 300, memo: 'Refund' }
    ];
    return mockTransactions;
  }

  createAccount(options) {
    console.log(chalk.blue('Creating new account...'));
    
    const account = {
      id: `ACC${Date.now()}`,
      name: options.name,
      type: options.type,
      balance: parseFloat(options.balance),
      createdAt: new Date().toISOString()
    };

    console.log(chalk.green('✓ Account created successfully!'));
    console.log('');
    console.log(chalk.bold('Account Details:'));
    console.log(`  ID: ${chalk.cyan(account.id)}`);
    console.log(`  Name: ${account.name}`);
    console.log(`  Type: ${account.type}`);
    console.log(`  Balance: $${account.balance.toLocaleString()}`);
    console.log(`  Created: ${account.createdAt}`);
  }

  listAccounts(options) {
    console.log(chalk.blue('Fetching accounts...'));
    
    let accounts = this.getAccounts();
    
    if (options.type) {
      accounts = accounts.filter(acc => acc.type === options.type);
    }

    accounts = accounts.slice(0, parseInt(options.limit));

    const table = new Table({
      head: ['Account ID', 'Name', 'Type', 'Balance'].map(h => chalk.cyan(h)),
      colWidths: [15, 25, 15, 20]
    });

    accounts.forEach(acc => {
      table.push([
        acc.id,
        acc.name,
        acc.type,
        `$${acc.balance.toLocaleString()}`
      ]);
    });

    console.log('');
    console.log(table.toString());
    console.log('');
    console.log(chalk.gray(`Total: ${accounts.length} account(s)`));
  }

  checkBalance(options) {
    console.log(chalk.blue(`Checking balance for account ${options.account}...`));
    
    const accounts = this.getAccounts();
    const account = accounts.find(acc => acc.id === options.account);

    if (!account) {
      console.log(chalk.red(`✗ Account ${options.account} not found`));
      process.exit(1);
    }

    console.log('');
    console.log(chalk.bold('Account Information:'));
    console.log(`  Account ID: ${chalk.cyan(account.id)}`);
    console.log(`  Name: ${account.name}`);
    console.log(`  Type: ${account.type}`);
    console.log(`  Balance: ${chalk.green('$' + account.balance.toLocaleString())}`);
  }

  transferMoney(options) {
    console.log(chalk.blue('Processing transfer...'));
    
    const amount = parseFloat(options.amount);
    
    if (amount <= 0) {
      console.log(chalk.red('✗ Invalid amount'));
      process.exit(1);
    }

    // Simulate transfer
    const transactionId = `TXN${Date.now()}`;
    
    console.log(chalk.green('✓ Transfer successful!'));
    console.log('');
    console.log(chalk.bold('Transaction Details:'));
    console.log(`  Transaction ID: ${chalk.cyan(transactionId)}`);
    console.log(`  From: ${options.from}`);
    console.log(`  To: ${options.to}`);
    console.log(`  Amount: $${amount.toLocaleString()}`);
    console.log(`  Memo: ${options.memo || 'N/A'}`);
    console.log(`  Date: ${new Date().toISOString()}`);
  }

  transactionHistory(options) {
    console.log(chalk.blue(`Fetching transaction history for ${options.account}...`));
    
    const transactions = this.getTransactions(options.account, options.days);

    if (options.format === 'json') {
      console.log(JSON.stringify(transactions, null, 2));
      return;
    }

    if (options.format === 'csv') {
      console.log('ID,Date,Type,Amount,Memo');
      transactions.forEach(tx => {
        console.log(`${tx.id},${tx.date},${tx.type},${tx.amount},${tx.memo}`);
      });
      return;
    }

    // Table format (default)
    const table = new Table({
      head: ['ID', 'Date', 'Type', 'Amount', 'Memo'].map(h => chalk.cyan(h)),
      colWidths: [12, 12, 10, 15, 30]
    });

    transactions.forEach(tx => {
      const amountColor = tx.type === 'credit' ? chalk.green : chalk.red;
      const prefix = tx.type === 'credit' ? '+' : '-';
      
      table.push([
        tx.id,
        tx.date,
        tx.type,
        amountColor(`${prefix}$${tx.amount.toLocaleString()}`),
        tx.memo
      ]);
    });

    console.log('');
    console.log(table.toString());
    console.log('');
    console.log(chalk.gray(`Showing last ${options.days} days`));
  }

  dailyReport(options) {
    console.log(chalk.blue(`Generating daily report for ${options.date}...`));
    
    const report = {
      date: options.date,
      totalTransactions: 1247,
      totalVolume: 5420000,
      deposits: { count: 523, amount: 3200000 },
      withdrawals: { count: 724, amount: 2220000 },
      avgTransactionSize: 4346
    };

    console.log('');
    console.log(chalk.bold.underline('Daily Transaction Report'));
    console.log('');
    console.log(`Date: ${chalk.cyan(report.date)}`);
    console.log(`Total Transactions: ${chalk.yellow(report.totalTransactions.toLocaleString())}`);
    console.log(`Total Volume: ${chalk.green('$' + report.totalVolume.toLocaleString())}`);
    console.log('');
    console.log(chalk.bold('Breakdown:'));
    console.log(`  Deposits: ${report.deposits.count} transactions, $${report.deposits.amount.toLocaleString()}`);
    console.log(`  Withdrawals: ${report.withdrawals.count} transactions, $${report.withdrawals.amount.toLocaleString()}`);
    console.log(`  Average: $${report.avgTransactionSize.toLocaleString()} per transaction`);

    if (options.export) {
      console.log('');
      console.log(chalk.green(`✓ Report exported to: ${options.export}`));
    }
  }

  accountSummary(options) {
    console.log(chalk.blue('Generating account summary...'));
    
    const accounts = this.getAccounts();
    const summary = {
      totalAccounts: accounts.length,
      totalBalance: accounts.reduce((sum, acc) => sum + acc.balance, 0),
      byType: {
        checking: accounts.filter(a => a.type === 'checking').length,
        savings: accounts.filter(a => a.type === 'savings').length,
        business: accounts.filter(a => a.type === 'business').length
      }
    };

    if (options.csv) {
      console.log('Type,Count,Total Balance');
      Object.entries(summary.byType).forEach(([type, count]) => {
        const typeAccounts = accounts.filter(a => a.type === type);
        const balance = typeAccounts.reduce((sum, acc) => sum + acc.balance, 0);
        console.log(`${type},${count},${balance}`);
      });
      return;
    }

    console.log('');
    console.log(chalk.bold.underline('Account Summary'));
    console.log('');
    console.log(`Total Accounts: ${chalk.yellow(summary.totalAccounts)}`);
    console.log(`Total Balance: ${chalk.green('$' + summary.totalBalance.toLocaleString())}`);
    console.log('');
    console.log(chalk.bold('By Type:'));
    console.log(`  Checking: ${summary.byType.checking} accounts`);
    console.log(`  Savings: ${summary.byType.savings} accounts`);
    console.log(`  Business: ${summary.byType.business} accounts`);
  }

  systemStats() {
    console.log(chalk.blue('Collecting system statistics...'));
    
    const memUsage = process.memoryUsage();
    const cpuUsage = process.cpuUsage();

    console.log('');
    console.log(chalk.bold.underline('System Statistics'));
    console.log('');
    console.log(chalk.cyan('Process Information:'));
    console.log(`  PID: ${process.pid}`);
    console.log(`  Platform: ${process.platform} (${process.arch})`);
    console.log(`  Node.js: ${process.version}`);
    console.log(`  Uptime: ${Math.floor(process.uptime())} seconds`);
    console.log(`  Working Directory: ${process.cwd()}`);
    console.log('');
    console.log(chalk.cyan('Memory Usage:'));
    console.log(`  RSS: ${Math.round(memUsage.rss / 1024 / 1024)}MB`);
    console.log(`  Heap Total: ${Math.round(memUsage.heapTotal / 1024 / 1024)}MB`);
    console.log(`  Heap Used: ${Math.round(memUsage.heapUsed / 1024 / 1024)}MB`);
    console.log(`  External: ${Math.round(memUsage.external / 1024 / 1024)}MB`);
    console.log('');
    console.log(chalk.cyan('CPU Usage:'));
    console.log(`  User: ${Math.round(cpuUsage.user / 1000)}ms`);
    console.log(`  System: ${Math.round(cpuUsage.system / 1000)}ms`);
  }

  cleanup(options) {
    const days = parseInt(options.days);
    
    console.log(chalk.blue(`Cleaning up logs older than ${days} days...`));
    
    if (options.dryRun) {
      console.log(chalk.yellow('DRY RUN MODE - No files will be deleted'));
    }

    // Simulate cleanup
    const filesFound = 127;
    const sizeFreed = 45.2;

    console.log('');
    console.log(`Found ${filesFound} old log files`);
    console.log(`Estimated space to free: ${sizeFreed}MB`);
    
    if (!options.dryRun) {
      console.log('');
      console.log(chalk.green('✓ Cleanup completed successfully'));
      console.log(`  Files deleted: ${filesFound}`);
      console.log(`  Space freed: ${sizeFreed}MB`);
    }
  }

  run() {
    program.parse(process.argv);

    // Show help if no command provided
    if (!process.argv.slice(2).length) {
      program.outputHelp();
    }
  }
}

// Run CLI
const cli = new BankingCLI();
cli.run();
```

**Testing Commands**:

```bash
# Install dependencies
npm install commander chalk cli-table3

# Show help
node cli/banking-cli.js --help

# Create account
node cli/banking-cli.js account:create --name "John Doe" --type checking --balance 5000

# List accounts
node cli/banking-cli.js account:list
node cli/banking-cli.js account:list --type savings --limit 5

# Check balance
node cli/banking-cli.js account:balance --account ACC001

# Transfer money
node cli/banking-cli.js transaction:transfer \
  --from ACC001 \
  --to ACC002 \
  --amount 500 \
  --memo "Payment for services"

# Transaction history
node cli/banking-cli.js transaction:history --account ACC001
node cli/banking-cli.js transaction:history --account ACC001 --format json
node cli/banking-cli.js transaction:history --account ACC001 --format csv --days 90

# Generate reports
node cli/banking-cli.js report:daily
node cli/banking-cli.js report:daily --date 2025-11-10 --export ./reports/daily.json
node cli/banking-cli.js report:summary
node cli/banking-cli.js report:summary --csv

# System statistics
node cli/banking-cli.js admin:stats

# Cleanup logs
node cli/banking-cli.js admin:cleanup --days 90 --dry-run
node cli/banking-cli.js admin:cleanup --days 90
```

---

### Example 3: Process Monitoring and Resource Management

**Scenario**: Build a process monitor that tracks resource usage, handles exit codes properly, and provides detailed diagnostics for production banking systems.

**File: `process-monitor.js`**

```javascript
const EventEmitter = require('events');
const fs = require('fs');
const path = require('path');

class ProcessMonitor extends EventEmitter {
  constructor(options = {}) {
    super();
    
    this.interval = options.interval || 5000; // 5 seconds
    this.logFile = options.logFile || path.join(process.cwd(), 'logs', 'process-monitor.log');
    this.alertThresholds = {
      memoryMB: options.maxMemoryMB || 500,
      cpuPercent: options.maxCpuPercent || 80,
      uptimeSeconds: options.minUptimeSeconds || 60
    };
    
    this.metrics = {
      startTime: Date.now(),
      samples: [],
      alerts: []
    };
    
    this.monitorInterval = null;
    this.setupProcess Monitoring();
    this.setupEventHandlers();
  }

  setupProcessMonitoring() {
    // Monitor resource usage periodically
    this.monitorInterval = setInterval(() => {
      this.collectMetrics();
    }, this.interval);

    // Ensure interval doesn't prevent process exit
    this.monitorInterval.unref();
  }

  setupEventHandlers() {
    // Uncaught exceptions
    process.on('uncaughtException', (error) => {
      this.log('FATAL', 'Uncaught Exception', error);
      this.emit('critical-error', { type: 'uncaughtException', error });
      
      // Give time to flush logs
      setTimeout(() => {
        process.exit(1);
      }, 1000);
    });

    // Unhandled promise rejections
    process.on('unhandledRejection', (reason, promise) => {
      this.log('ERROR', 'Unhandled Promise Rejection', { reason, promise });
      this.emit('error', { type: 'unhandledRejection', reason, promise });
    });

    // Warning events
    process.on('warning', (warning) => {
      this.log('WARN', 'Process Warning', warning);
    });

    // Before exit (can perform async operations)
    process.on('beforeExit', (code) => {
      this.log('INFO', `Process about to exit with code ${code}`);
      this.emit('before-exit', code);
    });

    // Exit (only sync operations)
    process.on('exit', (code) => {
      const runtime = Math.floor((Date.now() - this.metrics.startTime) / 1000);
      console.log(`Process exiting with code ${code} after ${runtime} seconds`);
      this.generateExitReport(code);
    });

    // Graceful shutdown signals
    process.on('SIGTERM', () => {
      this.log('INFO', 'Received SIGTERM signal');
      this.gracefulShutdown('SIGTERM');
    });

    process.on('SIGINT', () => {
      this.log('INFO', 'Received SIGINT signal (Ctrl+C)');
      this.gracefulShutdown('SIGINT');
    });

    // Memory warnings (Node.js v16+)
    if (process.memoryUsage.rss) {
      setInterval(() => {
        const memUsage = process.memoryUsage();
        const heapUsedMB = Math.round(memUsage.heapUsed / 1024 / 1024);
        
        if (heapUsedMB > this.alertThresholds.memoryMB) {
          this.log('WARN', `High memory usage: ${heapUsedMB}MB`);
          this.emit('high-memory', { heapUsedMB });
        }
      }, 10000);
    }
  }

  collectMetrics() {
    const memUsage = process.memoryUsage();
    const cpuUsage = process.cpuUsage();
    
    const metrics = {
      timestamp: new Date().toISOString(),
      pid: process.pid,
      uptime: Math.floor(process.uptime()),
      memory: {
        rss: Math.round(memUsage.rss / 1024 / 1024),
        heapTotal: Math.round(memUsage.heapTotal / 1024 / 1024),
        heapUsed: Math.round(memUsage.heapUsed / 1024 / 1024),
        external: Math.round(memUsage.external / 1024 / 1024),
        arrayBuffers: memUsage.arrayBuffers ? Math.round(memUsage.arrayBuffers / 1024 / 1024) : 0
      },
      cpu: {
        user: Math.round(cpuUsage.user / 1000),
        system: Math.round(cpuUsage.system / 1000),
        total: Math.round((cpuUsage.user + cpuUsage.system) / 1000)
      },
      platform: {
        os: process.platform,
        arch: process.arch,
        nodeVersion: process.version
      }
    };

    this.metrics.samples.push(metrics);
    
    // Keep only last 100 samples
    if (this.metrics.samples.length > 100) {
      this.metrics.samples.shift();
    }

    // Check thresholds
    this.checkThresholds(metrics);
    
    this.emit('metrics', metrics);
    return metrics;
  }

  checkThresholds(metrics) {
    // Memory threshold
    if (metrics.memory.heapUsed > this.alertThresholds.memoryMB) {
      const alert = {
        type: 'HIGH_MEMORY',
        timestamp: new Date().toISOString(),
        value: metrics.memory.heapUsed,
        threshold: this.alertThresholds.memoryMB
      };
      this.metrics.alerts.push(alert);
      this.emit('alert', alert);
    }

    // CPU threshold (simplified - would need sampling for accurate %)
    const cpuTotal = metrics.cpu.total;
    if (cpuTotal > this.alertThresholds.cpuPercent * 10) { // Simplified check
      const alert = {
        type: 'HIGH_CPU',
        timestamp: new Date().toISOString(),
        value: cpuTotal,
        threshold: this.alertThresholds.cpuPercent
      };
      this.metrics.alerts.push(alert);
      this.emit('alert', alert);
    }
  }

  getMetricsSummary() {
    if (this.metrics.samples.length === 0) {
      return null;
    }

    const samples = this.metrics.samples;
    const latest = samples[samples.length - 1];
    
    const avgMemory = samples.reduce((sum, s) => sum + s.memory.heapUsed, 0) / samples.length;
    const maxMemory = Math.max(...samples.map(s => s.memory.heapUsed));
    const avgCpu = samples.reduce((sum, s) => sum + s.cpu.total, 0) / samples.length;

    return {
      runtime: Math.floor((Date.now() - this.metrics.startTime) / 1000),
      samplesCollected: samples.length,
      current: latest,
      averages: {
        memory: Math.round(avgMemory),
        cpu: Math.round(avgCpu)
      },
      peaks: {
        memory: maxMemory
      },
      alerts: this.metrics.alerts.length
    };
  }

  log(level, message, data = null) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      level,
      message,
      pid: process.pid,
      data: data ? (data instanceof Error ? {
        name: data.name,
        message: data.message,
        stack: data.stack
      } : data) : undefined
    };

    const logLine = JSON.stringify(logEntry) + '\n';
    
    // Console output
    const colors = {
      INFO: '\x1b[36m',    // Cyan
      WARN: '\x1b[33m',    // Yellow
      ERROR: '\x1b[31m',   // Red
      FATAL: '\x1b[35m',   // Magenta
      RESET: '\x1b[0m'
    };
    
    console.log(`${colors[level] || ''}[${level}] ${message}${colors.RESET}`);
    if (data) console.log(data);

    // File output
    try {
      const logDir = path.dirname(this.logFile);
      if (!fs.existsSync(logDir)) {
        fs.mkdirSync(logDir, { recursive: true });
      }
      fs.appendFileSync(this.logFile, logLine);
    } catch (error) {
      console.error('Failed to write log file:', error.message);
    }
  }

  generateExitReport(exitCode) {
    const summary = this.getMetricsSummary();
    
    const report = {
      exitCode,
      exitTime: new Date().toISOString(),
      ...summary
    };

    const reportPath = path.join(process.cwd(), 'logs', `exit-report-${process.pid}.json`);
    
    try {
      fs.writeFileSync(reportPath, JSON.stringify(report, null, 2));
    } catch (error) {
      console.error('Failed to write exit report:', error.message);
    }
  }

  async gracefulShutdown(signal) {
    this.log('INFO', `Graceful shutdown initiated by ${signal}`);
    
    // Stop collecting metrics
    if (this.monitorInterval) {
      clearInterval(this.monitorInterval);
    }

    // Emit shutdown event for cleanup
    this.emit('shutdown', signal);

    // Wait a bit for cleanup
    await new Promise(resolve => setTimeout(resolve, 1000));

    const exitCode = signal === 'SIGTERM' ? 0 : 130; // 130 for SIGINT
    process.exit(exitCode);
  }

  printStatus() {
    const summary = this.getMetricsSummary();
    
    if (!summary) {
      console.log('No metrics collected yet');
      return;
    }

    console.log('\n========================================');
    console.log('Process Monitor Status');
    console.log('========================================');
    console.log(`PID: ${process.pid}`);
    console.log(`Platform: ${process.platform} (${process.arch})`);
    console.log(`Node.js: ${process.version}`);
    console.log(`Runtime: ${summary.runtime} seconds`);
    console.log(`Samples: ${summary.samplesCollected}`);
    console.log('\nCurrent Metrics:');
    console.log(`  Memory: ${summary.current.memory.heapUsed}MB (Peak: ${summary.peaks.memory}MB)`);
    console.log(`  CPU: ${summary.current.cpu.total}ms`);
    console.log(`  Uptime: ${summary.current.uptime}s`);
    console.log('\nAverages:');
    console.log(`  Memory: ${summary.averages.memory}MB`);
    console.log(`  CPU: ${summary.averages.cpu}ms`);
    console.log(`\nAlerts: ${summary.alerts}`);
    console.log('========================================\n');
  }
}

module.exports = ProcessMonitor;

// Example usage
if (require.main === module) {
  const monitor = new ProcessMonitor({
    interval: 2000,
    maxMemoryMB: 200,
    logFile: path.join(__dirname, 'logs', 'monitor.log')
  });

  // Listen to events
  monitor.on('metrics', (metrics) => {
    // Can send to monitoring service
  });

  monitor.on('alert', (alert) => {
    console.warn('⚠️  ALERT:', alert.type, `- ${alert.value} exceeds threshold ${alert.threshold}`);
  });

  monitor.on('shutdown', async (signal) => {
    console.log('Cleaning up before shutdown...');
    // Close database connections, flush buffers, etc.
  });

  // Simulate banking workload
  console.log('Starting process monitor demo...');
  console.log('Press Ctrl+C to trigger graceful shutdown\n');

  // Print status every 10 seconds
  setInterval(() => {
    monitor.printStatus();
  }, 10000);

  // Simulate some work
  setInterval(() => {
    // Simulate transaction processing
    const transactions = [];
    for (let i = 0; i < 1000; i++) {
      transactions.push({
        id: `TXN${i}`,
        amount: Math.random() * 10000,
        timestamp: Date.now()
      });
    }
    // Process transactions...
  }, 3000);
}
```

**Testing Commands**:

```bash
# Run process monitor
node process-monitor.js

# View process info while running (in another terminal)
# Linux/Mac:
ps aux | grep node
top -p $(pgrep node)

# Windows (PowerShell):
Get-Process node

# Test graceful shutdown
# Press Ctrl+C

# View exit report
cat logs/exit-report-*.json

# View monitor logs
cat logs/monitor.log

# Simulate high memory usage
node -e "const arr = []; setInterval(() => arr.push(new Array(1000000).fill('x')), 100)"

# Check exit codes
node process-monitor.js
echo $?  # Should print exit code (0 for success, non-zero for error)
```

---

## 🎯 DO's

1. **✅ DO** use `process.env` for all environment-specific configuration
2. **✅ DO** validate environment variables at startup before the app runs
3. **✅ DO** use `.env` files for local development (with `dotenv`)
4. **✅ DO** set sensitive values via environment variables in production
5. **✅ DO** provide sensible defaults for non-critical configuration
6. **✅ DO** use `process.argv` for command-line tools and scripts
7. **✅ DO** implement graceful shutdown handlers for SIGTERM and SIGINT
8. **✅ DO** use proper exit codes (0 for success, non-zero for errors)
9. **✅ DO** log important process events (startup, shutdown, errors)
10. **✅ DO** monitor resource usage (`process.memoryUsage()`, `process.cpuUsage()`)

---

## 🚫 DON'Ts

1. **❌ DON'T** hard-code credentials or API keys in source code
2. **❌ DON'T** commit `.env` files to version control (add to `.gitignore`)
3. **❌ DON'T** call `process.exit()` without cleanup in production apps
4. **❌ DON'T** ignore `process.on('uncaughtException')` events
5. **❌ DON'T** use `process.exit(0)` to handle errors (use non-zero codes)
6. **❌ DON'T** modify `process.env` at runtime (treat as read-only)
7. **❌ DON'T** rely on `process.argv` for complex CLI apps (use `commander` or `yargs`)
8. **❌ DON'T** forget to handle `unhandledRejection` events
9. **❌ DON'T** perform long-running operations in `process.on('exit')` (only sync ops)
10. **❌ DON'T** expose full configuration in logs (mask sensitive data)

---

## 🎓 Key Takeaways

1. **`process` is global** - Available everywhere without requiring/importing
2. **`process.env`** - Environment variables for configuration management
3. **`process.argv`** - Command-line arguments array
4. **Exit codes matter** - 0 = success, non-zero = error (use appropriate codes)
5. **Graceful shutdown** - Always handle SIGTERM/SIGINT for clean exits
6. **Environment-based config** - Different settings for dev/staging/production
7. **Security** - Never hard-code secrets, use environment variables
8. **Validation** - Check required config at startup, fail fast
9. **Resource monitoring** - Use `memoryUsage()` and `cpuUsage()` for diagnostics
10. **Platform awareness** - Use `process.platform` for OS-specific code

---

**Next Question**: [Q20: Process Management - Signals](./Q20_Process_Management_Signals.md) - Handling SIGTERM, SIGINT, SIGUSR1, and graceful shutdown

**Related Questions**:
- Q00: Node.js Internal Architecture
- Q09-Q10: Error Handling
- Q27-Q28: Security (Authentication & Data Protection)

---

**File**: `Q19_Process_Management_process_object.md`  
**Status**: ✅ Complete (2,100+ lines)
