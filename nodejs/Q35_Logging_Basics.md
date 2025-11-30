# Q35: Logging - Basics with Winston

## 📋 Summary
Logging is essential for debugging, monitoring, and auditing applications. This guide covers **Winston logger fundamentals**, log levels, transports, formatting, log rotation, and comprehensive banking examples for transaction audit trails.

---

## 🎯 What You'll Learn
- **Why Logging Matters**: Debugging, monitoring, compliance, security
- **Winston Logger**: Setup, configuration, best practices
- **Log Levels**: ERROR, WARN, INFO, DEBUG, custom levels
- **Transports**: Console, file, database, external services
- **Log Rotation**: Daily rotation, size-based rotation
- **Banking Examples**: Transaction audit logs, compliance logging

---

## 📖 Comprehensive Explanation

### Why Logging Matters

Proper logging is critical for:

1. **Debugging** 🐞
   - Trace error origins
   - Reproduce issues
   - Understand application flow

2. **Monitoring** 📊
   - Track application health
   - Identify performance bottlenecks
   - Alert on anomalies

3. **Auditing** 📄
   - Compliance requirements (PCI-DSS, SOX)
   - Security forensics
   - Track user actions

4. **Analytics** 📊
   - User behavior patterns
   - Business metrics
   - System usage statistics

### What NOT to Log

❌ **Never log sensitive data**:
- Passwords or API keys
- Credit card numbers (full PAN)
- Social Security Numbers
- Personal identification
- Encryption keys
- Session tokens

✅ **What to log safely**:
- User IDs (not names)
- Transaction IDs
- Timestamps
- Actions performed
- Error messages (sanitized)
- Request metadata

---

## 🧪 Winston Logger

### Why Winston?

**Winston** is the most popular Node.js logging library:
- Multiple transport support (console, file, HTTP)
- Custom log levels
- Formatting options (JSON, pretty-print)
- Log rotation built-in
- Production-ready
- Excellent performance

### Basic Winston Setup

```javascript
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'app.log' })
  ]
});

logger.info('Application started');
logger.error('An error occurred', { error: err.message });
```

---

## 📊 Log Levels

Winston uses npm-style log levels:

| Level | Priority | Use Case | Example |
|-------|----------|----------|----------|
| **error** | 0 (highest) | Critical failures | Database connection failed |
| **warn** | 1 | Warnings, degraded state | High memory usage (85%) |
| **info** | 2 | Important events | User logged in, payment completed |
| **http** | 3 | HTTP requests | GET /api/accounts 200 |
| **verbose** | 4 | Detailed information | Query executed in 45ms |
| **debug** | 5 | Debug information | Entering function processPayment |
| **silly** | 6 (lowest) | Everything | Variable x = 42 |

### Custom Log Levels

```javascript
const logger = winston.createLogger({
  levels: {
    fatal: 0,
    error: 1,
    warn: 2,
    info: 3,
    debug: 4,
    trace: 5
  },
  level: 'info'
});
```

### Environment-Based Levels

```javascript
const logger = winston.createLogger({
  level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
  // ...
});
```

---

## 📦 Transports

### Console Transport

```javascript
new winston.transports.Console({
  format: winston.format.combine(
    winston.format.colorize(),
    winston.format.simple()
  )
});
```

### File Transport

```javascript
new winston.transports.File({
  filename: 'logs/error.log',
  level: 'error',
  maxsize: 5242880, // 5MB
  maxFiles: 5
});
```

### Daily Rotate File

```javascript
const DailyRotateFile = require('winston-daily-rotate-file');

new DailyRotateFile({
  filename: 'logs/application-%DATE%.log',
  datePattern: 'YYYY-MM-DD',
  maxSize: '20m',
  maxFiles: '14d' // Keep 14 days
});
```

### HTTP Transport (External Service)

```javascript
new winston.transports.Http({
  host: 'logs.example.com',
  port: 8080,
  path: '/logs'
});
```

---

## 🎨 Formatting

### JSON Format

```javascript
winston.format.json()
// Output: {"level":"info","message":"User login","timestamp":"2024-01-15T10:30:00.000Z"}
```

### Pretty Print

```javascript
winston.format.prettyPrint()
```

### Custom Format

```javascript
const customFormat = winston.format.printf(({ level, message, timestamp, ...metadata }) => {
  let msg = `${timestamp} [${level}]: ${message}`;
  if (Object.keys(metadata).length > 0) {
    msg += ` ${JSON.stringify(metadata)}`;
  }
  return msg;
});

const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    customFormat
  )
});
```

### Combining Formats

```javascript
winston.format.combine(
  winston.format.timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
  winston.format.errors({ stack: true }),
  winston.format.splat(),
  winston.format.json()
)
```

---

## 📝 Example 1: Complete Banking Transaction Logger

### Production-Ready Logger Configuration

```javascript
// src/logger.js

const winston = require('winston');
const DailyRotateFile = require('winston-daily-rotate-file');
const path = require('path');

// Define log directory
const logDir = path.join(__dirname, '../logs');

// Custom format for production
const productionFormat = winston.format.combine(
  winston.format.timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
  winston.format.errors({ stack: true }),
  winston.format.json()
);

// Custom format for development
const developmentFormat = winston.format.combine(
  winston.format.colorize(),
  winston.format.timestamp({ format: 'HH:mm:ss' }),
  winston.format.printf(({ timestamp, level, message, ...metadata }) => {
    let msg = `${timestamp} ${level}: ${message}`;
    if (Object.keys(metadata).length > 0) {
      msg += ` ${JSON.stringify(metadata, null, 2)}`;
    }
    return msg;
  })
);

// Environment check
const isProduction = process.env.NODE_ENV === 'production';

// Create logger
const logger = winston.createLogger({
  level: isProduction ? 'info' : 'debug',
  format: isProduction ? productionFormat : developmentFormat,
  defaultMeta: { service: 'banking-api' },
  transports: [
    // Console transport
    new winston.transports.Console({
      format: isProduction ? productionFormat : developmentFormat
    }),

    // Error log file (all errors)
    new DailyRotateFile({
      filename: path.join(logDir, 'error-%DATE%.log'),
      datePattern: 'YYYY-MM-DD',
      level: 'error',
      maxSize: '20m',
      maxFiles: '30d',
      format: productionFormat
    }),

    // Combined log file (all logs)
    new DailyRotateFile({
      filename: path.join(logDir, 'combined-%DATE%.log'),
      datePattern: 'YYYY-MM-DD',
      maxSize: '20m',
      maxFiles: '14d',
      format: productionFormat
    }),

    // Audit log file (important events only)
    new DailyRotateFile({
      filename: path.join(logDir, 'audit-%DATE%.log'),
      datePattern: 'YYYY-MM-DD',
      level: 'info',
      maxSize: '20m',
      maxFiles: '90d', // Keep audit logs for 90 days (compliance)
      format: productionFormat
    })
  ],

  // Handle exceptions
  exceptionHandlers: [
    new winston.transports.File({ filename: path.join(logDir, 'exceptions.log') })
  ],

  // Handle promise rejections
  rejectionHandlers: [
    new winston.transports.File({ filename: path.join(logDir, 'rejections.log') })
  ]
});

// Create child logger for specific modules
logger.transaction = logger.child({ module: 'transaction' });
logger.auth = logger.child({ module: 'authentication' });
logger.api = logger.child({ module: 'api' });

module.exports = logger;
```

### Banking Transaction Audit Logger

```javascript
// src/services/transactionLogger.js

const logger = require('../logger');
const crypto = require('crypto');

class TransactionLogger {
  /**
   * Log transfer initiation
   */
  logTransferInitiated(transactionData) {
    const { transactionId, fromAccount, toAccount, amount, userId } = transactionData;

    logger.transaction.info('Transfer initiated', {
      event: 'TRANSFER_INITIATED',
      transactionId,
      fromAccount: this.maskAccount(fromAccount),
      toAccount: this.maskAccount(toAccount),
      amount: parseFloat(amount),
      currency: 'USD',
      userId,
      timestamp: new Date().toISOString()
    });
  }

  /**
   * Log transfer completion
   */
  logTransferCompleted(transactionData, duration) {
    const { transactionId, fromAccount, toAccount, amount } = transactionData;

    logger.transaction.info('Transfer completed', {
      event: 'TRANSFER_COMPLETED',
      transactionId,
      fromAccount: this.maskAccount(fromAccount),
      toAccount: this.maskAccount(toAccount),
      amount: parseFloat(amount),
      duration: `${duration}ms`,
      status: 'SUCCESS',
      timestamp: new Date().toISOString()
    });
  }

  /**
   * Log transfer failure
   */
  logTransferFailed(transactionData, error) {
    const { transactionId, fromAccount, toAccount, amount } = transactionData;

    logger.transaction.error('Transfer failed', {
      event: 'TRANSFER_FAILED',
      transactionId,
      fromAccount: this.maskAccount(fromAccount),
      toAccount: this.maskAccount(toAccount),
      amount: parseFloat(amount),
      errorCode: error.code || 'UNKNOWN',
      errorMessage: error.message,
      status: 'FAILED',
      timestamp: new Date().toISOString()
    });
  }

  /**
   * Log insufficient funds
   */
  logInsufficientFunds(accountId, requestedAmount, availableBalance) {
    logger.transaction.warn('Insufficient funds', {
      event: 'INSUFFICIENT_FUNDS',
      accountId: this.maskAccount(accountId),
      requestedAmount: parseFloat(requestedAmount),
      availableBalance: parseFloat(availableBalance),
      shortfall: parseFloat(requestedAmount - availableBalance),
      timestamp: new Date().toISOString()
    });
  }

  /**
   * Log suspicious activity
   */
  logSuspiciousActivity(accountId, reason, metadata) {
    logger.transaction.warn('Suspicious activity detected', {
      event: 'SUSPICIOUS_ACTIVITY',
      severity: 'HIGH',
      accountId: this.maskAccount(accountId),
      reason,
      metadata,
      alertId: this.generateAlertId(),
      timestamp: new Date().toISOString()
    });
  }

  /**
   * Log authentication events
   */
  logLogin(userId, ipAddress, userAgent, success) {
    const level = success ? 'info' : 'warn';
    logger.auth[level]('User login attempt', {
      event: success ? 'LOGIN_SUCCESS' : 'LOGIN_FAILED',
      userId,
      ipAddress,
      userAgent,
      timestamp: new Date().toISOString()
    });
  }

  /**
   * Log API requests
   */
  logApiRequest(req, res, duration) {
    logger.api.http('API request', {
      method: req.method,
      path: req.path,
      statusCode: res.statusCode,
      duration: `${duration}ms`,
      ipAddress: req.ip,
      userAgent: req.get('user-agent'),
      userId: req.user?.id,
      timestamp: new Date().toISOString()
    });
  }

  /**
   * Mask account number (show only last 4 digits)
   */
  maskAccount(accountNumber) {
    if (!accountNumber || typeof accountNumber !== 'string') {
      return '****';
    }
    const last4 = accountNumber.slice(-4);
    return `****${last4}`;
  }

  /**
   * Generate unique alert ID
   */
  generateAlertId() {
    return `ALERT-${Date.now()}-${crypto.randomBytes(4).toString('hex').toUpperCase()}`;
  }
}

module.exports = new TransactionLogger();
```

### Using the Transaction Logger

```javascript
// src/services/transferService.js

const transactionLogger = require('./transactionLogger');
const logger = require('../logger');

class TransferService {
  async executeTransfer(fromAccount, toAccount, amount, userId) {
    const transactionId = `TXN-${Date.now()}`;
    const startTime = Date.now();

    const transactionData = {
      transactionId,
      fromAccount,
      toAccount,
      amount,
      userId
    };

    try {
      // Log initiation
      transactionLogger.logTransferInitiated(transactionData);

      // Check balance
      const balance = await this.getBalance(fromAccount);
      if (balance < amount) {
        transactionLogger.logInsufficientFunds(fromAccount, amount, balance);
        throw new Error('Insufficient funds');
      }

      // Detect suspicious patterns
      if (amount > 100000) {
        transactionLogger.logSuspiciousActivity(
          fromAccount,
          'Large transfer amount',
          { amount, threshold: 100000 }
        );
      }

      // Execute transfer (database operations)
      await this.debitAccount(fromAccount, amount);
      await this.creditAccount(toAccount, amount);
      await this.recordTransaction(transactionData);

      // Log completion
      const duration = Date.now() - startTime;
      transactionLogger.logTransferCompleted(transactionData, duration);

      return { success: true, transactionId };

    } catch (error) {
      // Log failure
      transactionLogger.logTransferFailed(transactionData, error);
      throw error;
    }
  }

  async getBalance(accountId) {
    // Simulate database query
    logger.debug('Fetching account balance', { accountId });
    return 50000; // Example balance
  }

  async debitAccount(accountId, amount) {
    logger.debug('Debiting account', { accountId, amount });
    // Database operation
  }

  async creditAccount(accountId, amount) {
    logger.debug('Crediting account', { accountId, amount });
    // Database operation
  }

  async recordTransaction(data) {
    logger.debug('Recording transaction', { transactionId: data.transactionId });
    // Database operation
  }
}

module.exports = new TransferService();
```

### Express Middleware for Request Logging

```javascript
// src/middleware/requestLogger.js

const transactionLogger = require('../services/transactionLogger');

function requestLogger(req, res, next) {
  const startTime = Date.now();

  // Capture response
  res.on('finish', () => {
    const duration = Date.now() - startTime;
    transactionLogger.logApiRequest(req, res, duration);
  });

  next();
}

module.exports = requestLogger;
```

### Application Setup

```javascript
// src/app.js

const express = require('express');
const logger = require('./logger');
const requestLogger = require('./middleware/requestLogger');
const transferService = require('./services/transferService');

const app = express();

app.use(express.json());
app.use(requestLogger);

// Transfer endpoint
app.post('/api/transfer', async (req, res) => {
  try {
    const { fromAccount, toAccount, amount } = req.body;
    const userId = req.user?.id || 'anonymous';

    const result = await transferService.executeTransfer(
      fromAccount,
      toAccount,
      amount,
      userId
    );

    res.status(201).json(result);
  } catch (error) {
    logger.error('Transfer endpoint error', { error: error.message });
    res.status(500).json({ error: error.message });
  }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  logger.info('Server started', { port: PORT });
});
```

### Sample Log Output

**Development Mode (Pretty Print)**:
```
10:30:15 info: Server started {"port":3000,"service":"banking-api"}
10:30:22 info: Transfer initiated {
  "event": "TRANSFER_INITIATED",
  "transactionId": "TXN-1705315222000",
  "fromAccount": "****6789",
  "toAccount": "****1234",
  "amount": 500,
  "currency": "USD",
  "userId": "user-123",
  "timestamp": "2024-01-15T10:30:22.000Z",
  "module": "transaction"
}
10:30:22 debug: Fetching account balance {"accountId":"ACC-123456789","module":"transaction"}
10:30:22 debug: Debiting account {"accountId":"ACC-123456789","amount":500,"module":"transaction"}
10:30:22 debug: Crediting account {"accountId":"ACC-987654321","amount":500,"module":"transaction"}
10:30:22 info: Transfer completed {
  "event": "TRANSFER_COMPLETED",
  "transactionId": "TXN-1705315222000",
  "fromAccount": "****6789",
  "toAccount": "****1234",
  "amount": 500,
  "duration": "45ms",
  "status": "SUCCESS",
  "timestamp": "2024-01-15T10:30:22.045Z",
  "module": "transaction"
}
10:30:22 http: API request {
  "method": "POST",
  "path": "/api/transfer",
  "statusCode": 201,
  "duration": "50ms",
  "ipAddress": "::1",
  "userAgent": "PostmanRuntime/7.32.0",
  "userId": "user-123",
  "timestamp": "2024-01-15T10:30:22.050Z",
  "module": "api"
}
```

**Production Mode (JSON)**:
```json
{"level":"info","message":"Server started","port":3000,"service":"banking-api","timestamp":"2024-01-15 10:30:15"}
{"level":"info","message":"Transfer initiated","event":"TRANSFER_INITIATED","transactionId":"TXN-1705315222000","fromAccount":"****6789","toAccount":"****1234","amount":500,"currency":"USD","userId":"user-123","timestamp":"2024-01-15T10:30:22.000Z","module":"transaction","service":"banking-api","timestamp":"2024-01-15 10:30:22"}
{"level":"info","message":"Transfer completed","event":"TRANSFER_COMPLETED","transactionId":"TXN-1705315222000","fromAccount":"****6789","toAccount":"****1234","amount":500,"duration":"45ms","status":"SUCCESS","timestamp":"2024-01-15T10:30:22.045Z","module":"transaction","service":"banking-api","timestamp":"2024-01-15 10:30:22"}
```

### Installation and Setup

```bash
# Install dependencies
npm install winston winston-daily-rotate-file

# Create logs directory
mkdir logs

# Run in development
export NODE_ENV=development
node src/app.js

# Run in production
export NODE_ENV=production
node src/app.js
```

---

## 🎯 Key Takeaways

### Logging Best Practices

1. **Use appropriate log levels**
   ```javascript
   logger.debug('Variable state', { x: 42 });           // Development only
   logger.info('User logged in', { userId });           // Important events
   logger.warn('High memory usage', { usage: '85%' });  // Warnings
   logger.error('Database error', { error: err });      // Errors
   ```

2. **Include context**
   ```javascript
   // ❌ Bad: No context
   logger.info('Transfer completed');
   
   // ✅ Good: Rich context
   logger.info('Transfer completed', {
     transactionId: 'TXN-123',
     amount: 500,
     duration: '45ms'
   });
   ```

3. **Never log sensitive data**
   ```javascript
   // ❌ Bad
   logger.info('User created', { password: 'secret123' });
   
   // ✅ Good
   logger.info('User created', { userId: 'user-123' });
   ```

4. **Use structured logging**
   - JSON format for production
   - Easy to parse and search
   - Integrate with log aggregation tools

5. **Rotate logs**
   - Daily rotation for manageable file sizes
   - Retention policies (14 days combined, 90 days audit)
   - Automatic cleanup of old logs

### Production Checklist

- ✅ Log rotation enabled
- ✅ Error logs in separate file
- ✅ Audit logs retained for compliance
- ✅ Sensitive data masked
- ✅ Exception handlers configured
- ✅ Environment-based log levels
- ✅ Structured JSON format
- ✅ Service/module identification

---

**File**: `Q35_Logging_Basics.md`  
**Status**: ✅ Complete with Winston setup, log levels, transports, and banking audit system
