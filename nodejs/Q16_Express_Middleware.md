# Q16: Express.js - Middleware

## 📋 Summary
This question covers **Express.js middleware** in depth, including built-in middleware, third-party middleware, custom middleware creation, and middleware execution order. We'll build a production-ready banking API with authentication, logging, validation, error handling, and request processing middleware.

---

## 🎯 What You'll Learn
- Understanding Express.js middleware architecture
- Built-in middleware (express.json(), express.static(), etc.)
- Popular third-party middleware (helmet, cors, morgan, etc.)
- Creating custom middleware for specific business logic
- Middleware execution order and the `next()` function
- Error handling middleware
- Banking Example: Complete middleware stack for secure banking operations

---

## 📖 Detailed Explanation

### What is Middleware?

**Middleware** functions are functions that have access to:
1. **Request object** (`req`)
2. **Response object** (`res`)
3. **Next middleware function** (`next`)

Middleware can:
- Execute any code
- Make changes to request and response objects
- End the request-response cycle
- Call the next middleware in the stack

### Middleware Signature

```javascript
function middleware(req, res, next) {
  // Do something
  next(); // Pass control to next middleware
}
```

### Types of Middleware

#### 1. **Application-Level Middleware**
Bound to the app instance using `app.use()` or `app.METHOD()`.

```javascript
app.use((req, res, next) => {
  console.log('Request received');
  next();
});
```

#### 2. **Router-Level Middleware**
Bound to an instance of `express.Router()`.

```javascript
const router = express.Router();
router.use((req, res, next) => {
  console.log('Router middleware');
  next();
});
```

#### 3. **Built-in Middleware**
Provided by Express:
- `express.json()` - Parse JSON request bodies
- `express.urlencoded()` - Parse URL-encoded bodies
- `express.static()` - Serve static files
- `express.raw()` - Parse raw bodies
- `express.text()` - Parse text bodies

#### 4. **Third-Party Middleware**
Popular packages:
- **helmet** - Security headers
- **cors** - Cross-Origin Resource Sharing
- **morgan** - HTTP request logger
- **compression** - Gzip compression
- **express-rate-limit** - Rate limiting
- **express-validator** - Input validation

#### 5. **Error-Handling Middleware**
Has 4 parameters: `(err, req, res, next)`.

```javascript
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: err.message });
});
```

### Middleware Execution Order

**Critical Rule**: Middleware executes in the order it's defined.

```javascript
app.use(middleware1); // Executes first
app.use(middleware2); // Executes second
app.use(middleware3); // Executes third
```

**Important**: 
- If you don't call `next()`, the request hangs
- If you send a response, don't call `next()` after
- Error middleware must be defined AFTER route handlers

---

## 🏦 Banking Scenario

**GlobalBank** needs to implement a comprehensive middleware stack for their Express.js banking API:

### Requirements:
1. **Security**: Add security headers, CORS, rate limiting
2. **Logging**: Log all API requests with user info and timestamps
3. **Authentication**: Verify JWT tokens for protected routes
4. **Authorization**: Check user roles and permissions
5. **Validation**: Validate request bodies, params, and queries
6. **Error Handling**: Centralized error handling with proper responses
7. **Performance**: Monitor request duration and add response compression

### Challenges:
- Proper middleware order is critical
- Authentication must run before authorization
- Logging should capture both successful and failed requests
- Error handling must not expose sensitive information
- Rate limiting must prevent brute force attacks

---

## 💻 Example 1: Complete Banking API with Middleware Stack

### File Structure:
```
banking-api/
├── server.js
├── middleware/
│   ├── auth.js
│   ├── logger.js
│   ├── validator.js
│   └── errorHandler.js
├── routes/
│   ├── accounts.js
│   └── transactions.js
└── package.json
```

### Implementation:

#### **File: package.json**
```json
{
  "name": "banking-api-middleware",
  "version": "1.0.0",
  "description": "Banking API with comprehensive middleware",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "helmet": "^7.0.0",
    "cors": "^2.8.5",
    "morgan": "^1.10.0",
    "compression": "^1.7.4",
    "express-rate-limit": "^6.10.0",
    "express-validator": "^7.0.1",
    "jsonwebtoken": "^9.0.2",
    "bcrypt": "^5.1.1",
    "dotenv": "^16.3.1"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

#### **File: middleware/logger.js**
```javascript
const fs = require('fs');
const path = require('path');

// Custom logger middleware for banking operations
class BankingLogger {
  constructor() {
    this.logDir = path.join(__dirname, '../logs');
    this.ensureLogDirectory();
  }

  ensureLogDirectory() {
    if (!fs.existsSync(this.logDir)) {
      fs.mkdirSync(this.logDir, { recursive: true });
    }
  }

  formatLog(req, res, duration) {
    const log = {
      timestamp: new Date().toISOString(),
      method: req.method,
      url: req.originalUrl,
      ip: req.ip || req.connection.remoteAddress,
      userAgent: req.get('user-agent'),
      userId: req.user?.id || 'anonymous',
      username: req.user?.username || 'anonymous',
      statusCode: res.statusCode,
      duration: `${duration}ms`,
      requestBody: this.sanitizeBody(req.body),
      responseSize: res.get('content-length') || 0
    };
    return log;
  }

  sanitizeBody(body) {
    // Remove sensitive fields from logs
    if (!body) return {};
    
    const sanitized = { ...body };
    const sensitiveFields = ['password', 'pin', 'cvv', 'ssn', 'accountNumber'];
    
    sensitiveFields.forEach(field => {
      if (sanitized[field]) {
        sanitized[field] = '***REDACTED***';
      }
    });
    
    return sanitized;
  }

  writeToFile(log) {
    const date = new Date().toISOString().split('T')[0];
    const logFile = path.join(this.logDir, `banking-${date}.log`);
    
    const logLine = JSON.stringify(log) + '\n';
    fs.appendFileSync(logFile, logLine);
  }

  middleware() {
    return (req, res, next) => {
      const startTime = Date.now();

      // Capture response finish event
      res.on('finish', () => {
        const duration = Date.now() - startTime;
        const log = this.formatLog(req, res, duration);
        
        // Write to file
        this.writeToFile(log);
        
        // Console output with color
        const color = res.statusCode >= 500 ? '\x1b[31m' : // Red for 5xx
                      res.statusCode >= 400 ? '\x1b[33m' : // Yellow for 4xx
                      '\x1b[32m'; // Green for 2xx/3xx
        
        console.log(
          `${color}[${log.timestamp}] ${log.method} ${log.url} - ` +
          `Status: ${log.statusCode} - Duration: ${log.duration} - ` +
          `User: ${log.username}\x1b[0m`
        );
      });

      next();
    };
  }
}

module.exports = new BankingLogger();
```

#### **File: middleware/auth.js**
```javascript
const jwt = require('jsonwebtoken');

// JWT secret (in production, use environment variables)
const JWT_SECRET = process.env.JWT_SECRET || 'your-secret-key-change-in-production';

// Mock user database
const users = [
  { 
    id: 1, 
    username: 'john_doe', 
    email: 'john@bank.com', 
    role: 'customer',
    accountId: 'ACC001'
  },
  { 
    id: 2, 
    username: 'jane_admin', 
    email: 'jane@bank.com', 
    role: 'admin',
    accountId: 'ACC002'
  },
  { 
    id: 3, 
    username: 'mike_manager', 
    email: 'mike@bank.com', 
    role: 'manager',
    accountId: 'ACC003'
  }
];

// Generate JWT token (helper function for testing)
function generateToken(user) {
  return jwt.sign(
    { 
      id: user.id, 
      username: user.username, 
      email: user.email,
      role: user.role,
      accountId: user.accountId
    },
    JWT_SECRET,
    { expiresIn: '1h' }
  );
}

// Authentication middleware
function authenticate(req, res, next) {
  try {
    // Get token from header
    const authHeader = req.headers.authorization;
    
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      return res.status(401).json({
        success: false,
        error: 'Authentication required',
        message: 'No token provided'
      });
    }

    // Extract token
    const token = authHeader.substring(7); // Remove 'Bearer ' prefix

    // Verify token
    const decoded = jwt.verify(token, JWT_SECRET);
    
    // Attach user info to request
    req.user = decoded;
    
    // Log authentication success
    console.log(`✓ User authenticated: ${decoded.username} (${decoded.role})`);
    
    next();
  } catch (error) {
    if (error.name === 'TokenExpiredError') {
      return res.status(401).json({
        success: false,
        error: 'Token expired',
        message: 'Please login again'
      });
    }
    
    if (error.name === 'JsonWebTokenError') {
      return res.status(401).json({
        success: false,
        error: 'Invalid token',
        message: 'Authentication failed'
      });
    }

    return res.status(500).json({
      success: false,
      error: 'Authentication error',
      message: error.message
    });
  }
}

// Authorization middleware - check user roles
function authorize(...allowedRoles) {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({
        success: false,
        error: 'Authentication required',
        message: 'User not authenticated'
      });
    }

    if (!allowedRoles.includes(req.user.role)) {
      return res.status(403).json({
        success: false,
        error: 'Access denied',
        message: `This action requires one of these roles: ${allowedRoles.join(', ')}`,
        userRole: req.user.role
      });
    }

    console.log(`✓ User authorized: ${req.user.username} has role '${req.user.role}'`);
    next();
  };
}

// Account ownership middleware - verify user owns the account
function verifyAccountOwnership(req, res, next) {
  const accountId = req.params.accountId;
  
  if (!req.user) {
    return res.status(401).json({
      success: false,
      error: 'Authentication required'
    });
  }

  // Admins and managers can access any account
  if (req.user.role === 'admin' || req.user.role === 'manager') {
    return next();
  }

  // Regular users can only access their own account
  if (req.user.accountId !== accountId) {
    return res.status(403).json({
      success: false,
      error: 'Access denied',
      message: 'You can only access your own account',
      yourAccount: req.user.accountId,
      requestedAccount: accountId
    });
  }

  next();
}

module.exports = {
  authenticate,
  authorize,
  verifyAccountOwnership,
  generateToken,
  users,
  JWT_SECRET
};
```

#### **File: middleware/validator.js**
```javascript
const { body, param, query, validationResult } = require('express-validator');

// Validation error handler
function handleValidationErrors(req, res, next) {
  const errors = validationResult(req);
  
  if (!errors.isEmpty()) {
    return res.status(400).json({
      success: false,
      error: 'Validation failed',
      errors: errors.array().map(err => ({
        field: err.path,
        message: err.msg,
        value: err.value
      }))
    });
  }
  
  next();
}

// Account validation rules
const validateAccount = [
  body('accountType')
    .isIn(['savings', 'checking', 'credit'])
    .withMessage('Account type must be savings, checking, or credit'),
  
  body('currency')
    .optional()
    .isIn(['USD', 'EUR', 'GBP'])
    .withMessage('Currency must be USD, EUR, or GBP'),
  
  body('initialDeposit')
    .optional()
    .isFloat({ min: 0 })
    .withMessage('Initial deposit must be a positive number'),
  
  handleValidationErrors
];

// Transaction validation rules
const validateTransaction = [
  body('fromAccount')
    .matches(/^ACC\d{3,6}$/)
    .withMessage('From account must be in format ACC001 to ACC999999'),
  
  body('toAccount')
    .matches(/^ACC\d{3,6}$/)
    .withMessage('To account must be in format ACC001 to ACC999999'),
  
  body('amount')
    .isFloat({ min: 0.01, max: 1000000 })
    .withMessage('Amount must be between 0.01 and 1,000,000'),
  
  body('currency')
    .optional()
    .isIn(['USD', 'EUR', 'GBP'])
    .withMessage('Currency must be USD, EUR, or GBP'),
  
  body('description')
    .optional()
    .isLength({ min: 3, max: 200 })
    .withMessage('Description must be 3-200 characters')
    .trim()
    .escape(),
  
  // Custom validation: from and to accounts must be different
  body('toAccount').custom((value, { req }) => {
    if (value === req.body.fromAccount) {
      throw new Error('Cannot transfer to the same account');
    }
    return true;
  }),
  
  handleValidationErrors
];

// Login validation rules
const validateLogin = [
  body('username')
    .isLength({ min: 3, max: 50 })
    .withMessage('Username must be 3-50 characters')
    .trim(),
  
  body('password')
    .isLength({ min: 6 })
    .withMessage('Password must be at least 6 characters'),
  
  handleValidationErrors
];

// Account ID param validation
const validateAccountId = [
  param('accountId')
    .matches(/^ACC\d{3,6}$/)
    .withMessage('Account ID must be in format ACC001 to ACC999999'),
  
  handleValidationErrors
];

// Query parameter validation (for filtering/pagination)
const validateQueryParams = [
  query('page')
    .optional()
    .isInt({ min: 1 })
    .withMessage('Page must be a positive integer'),
  
  query('limit')
    .optional()
    .isInt({ min: 1, max: 100 })
    .withMessage('Limit must be between 1 and 100'),
  
  query('status')
    .optional()
    .isIn(['pending', 'completed', 'failed', 'cancelled'])
    .withMessage('Status must be pending, completed, failed, or cancelled'),
  
  query('startDate')
    .optional()
    .isISO8601()
    .withMessage('Start date must be in ISO 8601 format'),
  
  query('endDate')
    .optional()
    .isISO8601()
    .withMessage('End date must be in ISO 8601 format'),
  
  handleValidationErrors
];

module.exports = {
  validateAccount,
  validateTransaction,
  validateLogin,
  validateAccountId,
  validateQueryParams,
  handleValidationErrors
};
```

#### **File: middleware/errorHandler.js**
```javascript
// Custom error classes
class AppError extends Error {
  constructor(message, statusCode, errorCode) {
    super(message);
    this.statusCode = statusCode;
    this.errorCode = errorCode;
    this.isOperational = true;
    Error.captureStackTrace(this, this.constructor);
  }
}

class ValidationError extends AppError {
  constructor(message) {
    super(message, 400, 'VALIDATION_ERROR');
  }
}

class AuthenticationError extends AppError {
  constructor(message = 'Authentication failed') {
    super(message, 401, 'AUTHENTICATION_ERROR');
  }
}

class AuthorizationError extends AppError {
  constructor(message = 'Access denied') {
    super(message, 403, 'AUTHORIZATION_ERROR');
  }
}

class NotFoundError extends AppError {
  constructor(resource = 'Resource') {
    super(`${resource} not found`, 404, 'NOT_FOUND');
  }
}

class InsufficientFundsError extends AppError {
  constructor(balance, amount) {
    super(
      `Insufficient funds. Balance: ${balance}, Required: ${amount}`,
      400,
      'INSUFFICIENT_FUNDS'
    );
    this.balance = balance;
    this.requiredAmount = amount;
  }
}

class TransactionError extends AppError {
  constructor(message) {
    super(message, 400, 'TRANSACTION_ERROR');
  }
}

// Error handler middleware
function errorHandler(err, req, res, next) {
  // Log error (in production, use proper logging service)
  console.error('❌ Error occurred:');
  console.error({
    message: err.message,
    statusCode: err.statusCode,
    errorCode: err.errorCode,
    stack: process.env.NODE_ENV === 'development' ? err.stack : undefined,
    url: req.originalUrl,
    method: req.method,
    user: req.user?.username || 'anonymous'
  });

  // Default error values
  let statusCode = err.statusCode || 500;
  let errorCode = err.errorCode || 'INTERNAL_SERVER_ERROR';
  let message = err.message || 'Something went wrong';

  // Handle specific error types
  if (err.name === 'ValidationError') {
    statusCode = 400;
    errorCode = 'VALIDATION_ERROR';
  }

  if (err.name === 'JsonWebTokenError') {
    statusCode = 401;
    errorCode = 'INVALID_TOKEN';
    message = 'Invalid authentication token';
  }

  if (err.name === 'TokenExpiredError') {
    statusCode = 401;
    errorCode = 'TOKEN_EXPIRED';
    message = 'Authentication token expired';
  }

  if (err.code === 'ECONNREFUSED') {
    statusCode = 503;
    errorCode = 'SERVICE_UNAVAILABLE';
    message = 'Database connection failed';
  }

  // Don't expose error details in production
  const errorResponse = {
    success: false,
    error: {
      code: errorCode,
      message: message
    },
    timestamp: new Date().toISOString(),
    path: req.originalUrl
  };

  // Add additional error details in development
  if (process.env.NODE_ENV === 'development') {
    errorResponse.error.stack = err.stack;
    errorResponse.error.details = err;
  }

  // Add specific error properties
  if (err instanceof InsufficientFundsError) {
    errorResponse.error.balance = err.balance;
    errorResponse.error.requiredAmount = err.requiredAmount;
  }

  res.status(statusCode).json(errorResponse);
}

// 404 handler (must be defined after all routes)
function notFoundHandler(req, res, next) {
  const error = new NotFoundError('Endpoint');
  error.message = `Cannot ${req.method} ${req.originalUrl}`;
  next(error);
}

// Async error wrapper - catches errors in async route handlers
function asyncHandler(fn) {
  return (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
}

module.exports = {
  AppError,
  ValidationError,
  AuthenticationError,
  AuthorizationError,
  NotFoundError,
  InsufficientFundsError,
  TransactionError,
  errorHandler,
  notFoundHandler,
  asyncHandler
};
```

#### **File: routes/accounts.js**
```javascript
const express = require('express');
const router = express.Router();
const { authenticate, authorize, verifyAccountOwnership } = require('../middleware/auth');
const { validateAccount, validateAccountId } = require('../middleware/validator');
const { NotFoundError, asyncHandler } = require('../middleware/errorHandler');

// Mock database
const accounts = [
  { id: 'ACC001', userId: 1, type: 'checking', balance: 5000, currency: 'USD' },
  { id: 'ACC002', userId: 2, type: 'savings', balance: 15000, currency: 'USD' },
  { id: 'ACC003', userId: 3, type: 'checking', balance: 8000, currency: 'USD' }
];

// GET /api/accounts - Get all accounts (admin/manager only)
router.get('/',
  authenticate,
  authorize('admin', 'manager'),
  asyncHandler(async (req, res) => {
    res.json({
      success: true,
      count: accounts.length,
      data: accounts
    });
  })
);

// GET /api/accounts/:accountId - Get specific account
router.get('/:accountId',
  authenticate,
  validateAccountId,
  verifyAccountOwnership,
  asyncHandler(async (req, res) => {
    const account = accounts.find(acc => acc.id === req.params.accountId);
    
    if (!account) {
      throw new NotFoundError('Account');
    }

    res.json({
      success: true,
      data: account
    });
  })
);

// POST /api/accounts - Create new account
router.post('/',
  authenticate,
  validateAccount,
  asyncHandler(async (req, res) => {
    const newAccount = {
      id: `ACC${String(accounts.length + 1).padStart(3, '0')}`,
      userId: req.user.id,
      type: req.body.accountType,
      balance: req.body.initialDeposit || 0,
      currency: req.body.currency || 'USD',
      createdAt: new Date().toISOString()
    };

    accounts.push(newAccount);

    res.status(201).json({
      success: true,
      message: 'Account created successfully',
      data: newAccount
    });
  })
);

// DELETE /api/accounts/:accountId - Close account (admin only)
router.delete('/:accountId',
  authenticate,
  authorize('admin'),
  validateAccountId,
  asyncHandler(async (req, res) => {
    const index = accounts.findIndex(acc => acc.id === req.params.accountId);
    
    if (index === -1) {
      throw new NotFoundError('Account');
    }

    const account = accounts[index];
    
    if (account.balance > 0) {
      return res.status(400).json({
        success: false,
        error: 'Cannot close account with positive balance',
        balance: account.balance
      });
    }

    accounts.splice(index, 1);

    res.json({
      success: true,
      message: 'Account closed successfully',
      data: { accountId: req.params.accountId }
    });
  })
);

module.exports = router;
```

#### **File: routes/transactions.js**
```javascript
const express = require('express');
const router = express.Router();
const { authenticate, verifyAccountOwnership } = require('../middleware/auth');
const { validateTransaction, validateQueryParams } = require('../middleware/validator');
const { 
  InsufficientFundsError, 
  TransactionError,
  NotFoundError,
  asyncHandler 
} = require('../middleware/errorHandler');

// Mock database
const accounts = [
  { id: 'ACC001', userId: 1, balance: 5000 },
  { id: 'ACC002', userId: 2, balance: 15000 },
  { id: 'ACC003', userId: 3, balance: 8000 }
];

const transactions = [];

// GET /api/transactions - Get user's transactions
router.get('/',
  authenticate,
  validateQueryParams,
  asyncHandler(async (req, res) => {
    const { page = 1, limit = 10, status } = req.query;
    
    // Filter transactions for current user (unless admin)
    let userTransactions = transactions;
    if (req.user.role !== 'admin' && req.user.role !== 'manager') {
      userTransactions = transactions.filter(
        t => t.fromAccount === req.user.accountId || t.toAccount === req.user.accountId
      );
    }

    // Filter by status if provided
    if (status) {
      userTransactions = userTransactions.filter(t => t.status === status);
    }

    // Pagination
    const startIndex = (page - 1) * limit;
    const endIndex = startIndex + parseInt(limit);
    const paginatedTransactions = userTransactions.slice(startIndex, endIndex);

    res.json({
      success: true,
      count: userTransactions.length,
      page: parseInt(page),
      limit: parseInt(limit),
      data: paginatedTransactions
    });
  })
);

// POST /api/transactions - Create new transaction
router.post('/',
  authenticate,
  validateTransaction,
  asyncHandler(async (req, res) => {
    const { fromAccount, toAccount, amount, description } = req.body;

    // Verify user owns the from account (unless admin/manager)
    if (req.user.role !== 'admin' && req.user.role !== 'manager') {
      if (req.user.accountId !== fromAccount) {
        throw new TransactionError('You can only transfer from your own account');
      }
    }

    // Find accounts
    const sourceAccount = accounts.find(acc => acc.id === fromAccount);
    const targetAccount = accounts.find(acc => acc.id === toAccount);

    if (!sourceAccount) {
      throw new NotFoundError('Source account');
    }

    if (!targetAccount) {
      throw new NotFoundError('Target account');
    }

    // Check sufficient balance
    if (sourceAccount.balance < amount) {
      throw new InsufficientFundsError(sourceAccount.balance, amount);
    }

    // Perform transaction
    sourceAccount.balance -= amount;
    targetAccount.balance += amount;

    // Create transaction record
    const transaction = {
      id: `TXN${String(transactions.length + 1).padStart(6, '0')}`,
      fromAccount,
      toAccount,
      amount,
      description: description || 'Transfer',
      status: 'completed',
      timestamp: new Date().toISOString(),
      initiatedBy: req.user.username
    };

    transactions.push(transaction);

    res.status(201).json({
      success: true,
      message: 'Transaction completed successfully',
      data: transaction,
      newBalances: {
        fromAccount: sourceAccount.balance,
        toAccount: targetAccount.balance
      }
    });
  })
);

// GET /api/transactions/:transactionId - Get specific transaction
router.get('/:transactionId',
  authenticate,
  asyncHandler(async (req, res) => {
    const transaction = transactions.find(t => t.id === req.params.transactionId);

    if (!transaction) {
      throw new NotFoundError('Transaction');
    }

    // Verify user can access this transaction
    if (req.user.role !== 'admin' && req.user.role !== 'manager') {
      if (transaction.fromAccount !== req.user.accountId && 
          transaction.toAccount !== req.user.accountId) {
        throw new TransactionError('You do not have access to this transaction');
      }
    }

    res.json({
      success: true,
      data: transaction
    });
  })
);

module.exports = router;
```

#### **File: server.js**
```javascript
const express = require('express');
const helmet = require('helmet');
const cors = require('cors');
const morgan = require('morgan');
const compression = require('compression');
const rateLimit = require('express-rate-limit');
const { generateToken, users } = require('./middleware/auth');
const bankingLogger = require('./middleware/logger');
const { validateLogin } = require('./middleware/validator');
const { errorHandler, notFoundHandler } = require('./middleware/errorHandler');

const app = express();
const PORT = process.env.PORT || 3000;

// ============================================
// MIDDLEWARE STACK (ORDER MATTERS!)
// ============================================

// 1. SECURITY - Must be first
app.use(helmet()); // Security headers

// 2. CORS - Cross-origin requests
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS || '*',
  credentials: true
}));

// 3. COMPRESSION - Compress responses
app.use(compression());

// 4. BODY PARSING - Built-in Express middleware
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true, limit: '10mb' }));

// 5. LOGGING - HTTP request logger
app.use(morgan('dev')); // Console logging
app.use(bankingLogger.middleware()); // Custom banking logger

// 6. RATE LIMITING - Prevent abuse
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Limit each IP to 100 requests per windowMs
  message: {
    success: false,
    error: 'Too many requests',
    message: 'You have exceeded the 100 requests in 15 minutes limit'
  },
  standardHeaders: true,
  legacyHeaders: false
});

app.use('/api/', limiter);

// Special rate limit for login endpoint (stricter)
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5, // 5 login attempts per 15 minutes
  message: {
    success: false,
    error: 'Too many login attempts',
    message: 'Please try again after 15 minutes'
  }
});

// 7. REQUEST TIMING - Custom middleware to track request duration
app.use((req, res, next) => {
  req.startTime = Date.now();
  res.on('finish', () => {
    const duration = Date.now() - req.startTime;
    res.setHeader('X-Response-Time', `${duration}ms`);
  });
  next();
});

// ============================================
// PUBLIC ROUTES (No authentication required)
// ============================================

// Health check endpoint
app.get('/health', (req, res) => {
  res.json({
    success: true,
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: process.uptime()
  });
});

// Login endpoint
app.post('/api/login', loginLimiter, validateLogin, (req, res) => {
  const { username, password } = req.body;

  // Find user (in production, verify password hash)
  const user = users.find(u => u.username === username);

  if (!user || password !== 'password123') {
    return res.status(401).json({
      success: false,
      error: 'Invalid credentials',
      message: 'Username or password is incorrect'
    });
  }

  // Generate JWT token
  const token = generateToken(user);

  res.json({
    success: true,
    message: 'Login successful',
    token,
    user: {
      id: user.id,
      username: user.username,
      email: user.email,
      role: user.role,
      accountId: user.accountId
    }
  });
});

// Get sample token for testing (REMOVE IN PRODUCTION)
app.get('/api/get-token/:username', (req, res) => {
  const user = users.find(u => u.username === req.params.username);
  
  if (!user) {
    return res.status(404).json({
      success: false,
      error: 'User not found'
    });
  }

  const token = generateToken(user);

  res.json({
    success: true,
    message: 'Test token generated',
    token,
    user: {
      username: user.username,
      role: user.role,
      accountId: user.accountId
    },
    note: 'Use this token in Authorization header as: Bearer <token>'
  });
});

// ============================================
// PROTECTED ROUTES (Authentication required)
// ============================================

// Import routers
const accountsRouter = require('./routes/accounts');
const transactionsRouter = require('./routes/transactions');

// Mount routers
app.use('/api/accounts', accountsRouter);
app.use('/api/transactions', transactionsRouter);

// ============================================
// ERROR HANDLING MIDDLEWARE (Must be last!)
// ============================================

// 404 handler - catches all undefined routes
app.use(notFoundHandler);

// Global error handler - catches all errors
app.use(errorHandler);

// ============================================
// START SERVER
// ============================================

app.listen(PORT, () => {
  console.log(`\n🏦 Banking API Server Started`);
  console.log(`📍 Server running on: http://localhost:${PORT}`);
  console.log(`🔒 Environment: ${process.env.NODE_ENV || 'development'}`);
  console.log(`\n📚 Available endpoints:`);
  console.log(`   GET  /health - Health check`);
  console.log(`   POST /api/login - User login`);
  console.log(`   GET  /api/get-token/:username - Get test token`);
  console.log(`   GET  /api/accounts - Get all accounts (admin/manager)`);
  console.log(`   GET  /api/accounts/:id - Get account details`);
  console.log(`   POST /api/accounts - Create new account`);
  console.log(`   GET  /api/transactions - Get transactions`);
  console.log(`   POST /api/transactions - Create transaction`);
  console.log(`\n👥 Test users: john_doe, jane_admin, mike_manager`);
  console.log(`🔑 Test password: password123\n`);
});

module.exports = app;
```

### Testing Commands:

```bash
# Install dependencies
npm install

# Start server
npm start

# In another terminal, run these tests:

# 1. Health check (no auth required)
curl http://localhost:3000/health

# 2. Get test token for john_doe (customer)
curl http://localhost:3000/api/get-token/john_doe

# 3. Get test token for jane_admin (admin)
curl http://localhost:3000/api/get-token/jane_admin

# 4. Login
curl -X POST http://localhost:3000/api/login \
  -H "Content-Type: application/json" \
  -d '{"username":"john_doe","password":"password123"}'

# 5. Get all accounts (requires admin/manager role - will fail for john_doe)
curl http://localhost:3000/api/accounts \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"

# 6. Get specific account (john_doe can access ACC001)
curl http://localhost:3000/api/accounts/ACC001 \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"

# 7. Create new account
curl -X POST http://localhost:3000/api/accounts \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{"accountType":"savings","initialDeposit":1000,"currency":"USD"}'

# 8. Create transaction (john_doe can transfer from ACC001)
curl -X POST http://localhost:3000/api/transactions \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{
    "fromAccount":"ACC001",
    "toAccount":"ACC002",
    "amount":100,
    "description":"Test transfer"
  }'

# 9. Get transactions
curl http://localhost:3000/api/transactions \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"

# 10. Test validation error (invalid amount)
curl -X POST http://localhost:3000/api/transactions \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{
    "fromAccount":"ACC001",
    "toAccount":"ACC002",
    "amount":-50
  }'

# 11. Test insufficient funds
curl -X POST http://localhost:3000/api/transactions \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{
    "fromAccount":"ACC001",
    "toAccount":"ACC002",
    "amount":999999
  }'

# 12. Test 404 error
curl http://localhost:3000/api/nonexistent

# 13. Test rate limiting (run this command 101 times quickly)
for i in {1..101}; do curl http://localhost:3000/health; done
```

### Expected Output:

```json
// Health Check
{
  "success": true,
  "status": "healthy",
  "timestamp": "2024-01-15T10:30:00.000Z",
  "uptime": 123.456
}

// Get Token
{
  "success": true,
  "message": "Test token generated",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "username": "john_doe",
    "role": "customer",
    "accountId": "ACC001"
  },
  "note": "Use this token in Authorization header as: Bearer <token>"
}

// Successful Transaction
{
  "success": true,
  "message": "Transaction completed successfully",
  "data": {
    "id": "TXN000001",
    "fromAccount": "ACC001",
    "toAccount": "ACC002",
    "amount": 100,
    "description": "Test transfer",
    "status": "completed",
    "timestamp": "2024-01-15T10:35:00.000Z",
    "initiatedBy": "john_doe"
  },
  "newBalances": {
    "fromAccount": 4900,
    "toAccount": 15100
  }
}

// Validation Error
{
  "success": false,
  "error": "Validation failed",
  "errors": [
    {
      "field": "amount",
      "message": "Amount must be between 0.01 and 1,000,000",
      "value": -50
    }
  ]
}

// Insufficient Funds Error
{
  "success": false,
  "error": {
    "code": "INSUFFICIENT_FUNDS",
    "message": "Insufficient funds. Balance: 5000, Required: 999999",
    "balance": 5000,
    "requiredAmount": 999999
  },
  "timestamp": "2024-01-15T10:40:00.000Z",
  "path": "/api/transactions"
}
```

---

## 💻 Example 2: Middleware Chaining and Conditional Middleware

This example shows advanced middleware patterns including middleware chaining, conditional middleware, and dynamic middleware loading.

### Implementation:

```javascript
const express = require('express');
const app = express();

// ============================================
// ADVANCED MIDDLEWARE PATTERNS
// ============================================

// 1. MIDDLEWARE FACTORY - Creates middleware with configuration
function createRateLimiter(options = {}) {
  const { 
    windowMs = 60000, 
    max = 10, 
    message = 'Rate limit exceeded' 
  } = options;
  
  const requests = new Map();

  return (req, res, next) => {
    const key = req.ip;
    const now = Date.now();
    
    if (!requests.has(key)) {
      requests.set(key, []);
    }

    const userRequests = requests.get(key);
    
    // Remove old requests outside the window
    const validRequests = userRequests.filter(time => now - time < windowMs);
    
    if (validRequests.length >= max) {
      return res.status(429).json({
        success: false,
        error: message,
        retryAfter: Math.ceil((validRequests[0] + windowMs - now) / 1000)
      });
    }

    validRequests.push(now);
    requests.set(key, validRequests);
    
    next();
  };
}

// 2. CONDITIONAL MIDDLEWARE - Only runs if condition is met
function conditionalMiddleware(condition, middleware) {
  return (req, res, next) => {
    if (condition(req)) {
      return middleware(req, res, next);
    }
    next();
  };
}

// 3. MIDDLEWARE COMPOSITION - Combine multiple middleware
function composeMiddleware(...middlewares) {
  return (req, res, next) => {
    let index = 0;

    function dispatch(i) {
      if (i >= middlewares.length) {
        return next();
      }

      const middleware = middlewares[i];
      
      try {
        middleware(req, res, (err) => {
          if (err) {
            return next(err);
          }
          dispatch(i + 1);
        });
      } catch (err) {
        next(err);
      }
    }

    dispatch(0);
  };
}

// 4. MIDDLEWARE WITH STATE - Maintains state across requests
function createRequestCounter() {
  let totalRequests = 0;
  const requestsByEndpoint = new Map();

  return {
    middleware: (req, res, next) => {
      totalRequests++;
      
      const endpoint = `${req.method} ${req.path}`;
      requestsByEndpoint.set(
        endpoint,
        (requestsByEndpoint.get(endpoint) || 0) + 1
      );

      req.requestStats = {
        totalRequests,
        endpointRequests: requestsByEndpoint.get(endpoint)
      };

      next();
    },
    
    getStats: () => ({
      totalRequests,
      requestsByEndpoint: Object.fromEntries(requestsByEndpoint)
    })
  };
}

// 5. ASYNC MIDDLEWARE - Handles promises
function asyncMiddleware(fn) {
  return (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
}

// 6. MIDDLEWARE WITH CLEANUP - Runs cleanup on response finish
function withCleanup(setup, cleanup) {
  return (req, res, next) => {
    setup(req, res);
    
    res.on('finish', () => {
      cleanup(req, res);
    });
    
    next();
  };
}

// 7. ROLE-BASED MIDDLEWARE LOADER - Loads different middleware for different roles
function loadMiddlewareByRole(roleMiddlewareMap) {
  return (req, res, next) => {
    const userRole = req.user?.role || 'guest';
    const roleMiddleware = roleMiddlewareMap[userRole] || [];

    if (roleMiddleware.length === 0) {
      return next();
    }

    // Chain role-specific middleware
    let index = 0;
    function runNext() {
      if (index >= roleMiddleware.length) {
        return next();
      }
      const middleware = roleMiddleware[index++];
      middleware(req, res, runNext);
    }

    runNext();
  };
}

// ============================================
// PRACTICAL EXAMPLES
// ============================================

app.use(express.json());

// Example: Request counter with state
const requestCounter = createRequestCounter();
app.use(requestCounter.middleware);

// Example: Different rate limits for different endpoints
app.use('/api/public', createRateLimiter({ max: 100, windowMs: 60000 }));
app.use('/api/sensitive', createRateLimiter({ max: 10, windowMs: 60000 }));

// Example: Conditional middleware - only log POST requests
app.use(conditionalMiddleware(
  (req) => req.method === 'POST',
  (req, res, next) => {
    console.log(`POST request to ${req.path}`, req.body);
    next();
  }
));

// Example: Composed middleware for transaction endpoints
const transactionMiddleware = composeMiddleware(
  (req, res, next) => {
    console.log('Step 1: Validate transaction');
    next();
  },
  (req, res, next) => {
    console.log('Step 2: Check fraud rules');
    next();
  },
  (req, res, next) => {
    console.log('Step 3: Log transaction attempt');
    next();
  }
);

// Example: Async middleware with database simulation
const checkAccountExists = asyncMiddleware(async (req, res, next) => {
  const accountId = req.params.accountId;
  
  // Simulate async database check
  await new Promise(resolve => setTimeout(resolve, 100));
  
  // Mock check
  if (!accountId.startsWith('ACC')) {
    throw new Error('Invalid account ID format');
  }
  
  req.accountExists = true;
  next();
});

// Example: Middleware with resource cleanup
const withDatabaseConnection = withCleanup(
  (req, res) => {
    req.dbConnection = { id: Math.random(), connected: true };
    console.log(`Database connection opened: ${req.dbConnection.id}`);
  },
  (req, res) => {
    if (req.dbConnection) {
      console.log(`Database connection closed: ${req.dbConnection.id}`);
      req.dbConnection = null;
    }
  }
);

// Example: Role-based middleware
const roleBasedMiddleware = loadMiddlewareByRole({
  admin: [
    (req, res, next) => {
      console.log('Admin middleware: Full access granted');
      req.permissions = ['read', 'write', 'delete'];
      next();
    }
  ],
  customer: [
    (req, res, next) => {
      console.log('Customer middleware: Limited access');
      req.permissions = ['read'];
      next();
    },
    (req, res, next) => {
      console.log('Customer middleware: Rate limit check');
      next();
    }
  ],
  guest: [
    (req, res, next) => {
      console.log('Guest middleware: Public access only');
      req.permissions = [];
      next();
    }
  ]
});

// Mock authentication middleware for testing
app.use((req, res, next) => {
  const role = req.query.role || 'guest';
  req.user = { role };
  next();
});

// Apply role-based middleware
app.use(roleBasedMiddleware);

// ============================================
// ROUTES
// ============================================

// Public endpoint
app.get('/api/public/info', (req, res) => {
  res.json({
    success: true,
    message: 'Public information',
    stats: requestCounter.getStats()
  });
});

// Sensitive endpoint
app.post('/api/sensitive/data', (req, res) => {
  res.json({
    success: true,
    message: 'Sensitive operation completed',
    permissions: req.permissions
  });
});

// Transaction endpoint with composed middleware
app.post('/api/transactions', 
  transactionMiddleware,
  (req, res) => {
    res.json({
      success: true,
      message: 'Transaction processed'
    });
  }
);

// Account endpoint with async middleware
app.get('/api/accounts/:accountId',
  checkAccountExists,
  withDatabaseConnection,
  (req, res) => {
    res.json({
      success: true,
      accountId: req.params.accountId,
      accountExists: req.accountExists,
      dbConnection: req.dbConnection.id,
      stats: req.requestStats
    });
  }
);

// Stats endpoint
app.get('/api/stats', (req, res) => {
  res.json({
    success: true,
    data: requestCounter.getStats()
  });
});

// Error handler
app.use((err, req, res, next) => {
  console.error('Error:', err.message);
  res.status(500).json({
    success: false,
    error: err.message
  });
});

// Start server
const PORT = 3001;
app.listen(PORT, () => {
  console.log(`\n🚀 Advanced Middleware Server Started`);
  console.log(`📍 http://localhost:${PORT}\n`);
  console.log('Test endpoints:');
  console.log('  GET  /api/public/info');
  console.log('  POST /api/sensitive/data');
  console.log('  POST /api/transactions');
  console.log('  GET  /api/accounts/:accountId');
  console.log('  GET  /api/stats');
  console.log('\nTest with different roles: ?role=admin, ?role=customer, ?role=guest\n');
});
```

### Testing Commands:

```bash
# Save as advanced-middleware.js and run:
node advanced-middleware.js

# Test public endpoint (high rate limit)
curl "http://localhost:3001/api/public/info"

# Test with different roles
curl "http://localhost:3001/api/public/info?role=admin"
curl "http://localhost:3001/api/public/info?role=customer"
curl "http://localhost:3001/api/public/info?role=guest"

# Test sensitive endpoint (low rate limit)
curl -X POST "http://localhost:3001/api/sensitive/data?role=admin" \
  -H "Content-Type: application/json" \
  -d '{"data":"test"}'

# Test transaction with composed middleware
curl -X POST "http://localhost:3001/api/transactions" \
  -H "Content-Type: application/json" \
  -d '{"amount":100}'

# Test async middleware with account check
curl "http://localhost:3001/api/accounts/ACC001"
curl "http://localhost:3001/api/accounts/INVALID" # Should error

# Test rate limiting (run 15 times quickly)
for i in {1..15}; do 
  curl "http://localhost:3001/api/sensitive/data" 
  echo ""
done

# View stats
curl "http://localhost:3001/api/stats"
```

### Expected Output:

```json
// Public endpoint with stats
{
  "success": true,
  "message": "Public information",
  "stats": {
    "totalRequests": 5,
    "requestsByEndpoint": {
      "GET /api/public/info": 3,
      "POST /api/sensitive/data": 2
    }
  }
}

// Admin role response
{
  "success": true,
  "message": "Sensitive operation completed",
  "permissions": ["read", "write", "delete"]
}

// Customer role response
{
  "success": true,
  "message": "Sensitive operation completed",
  "permissions": ["read"]
}

// Account check with async middleware
{
  "success": true,
  "accountId": "ACC001",
  "accountExists": true,
  "dbConnection": 0.12345678,
  "stats": {
    "totalRequests": 10,
    "endpointRequests": 2
  }
}

// Rate limit exceeded
{
  "success": false,
  "error": "Rate limit exceeded",
  "retryAfter": 45
}

// Console output shows middleware execution:
// Admin middleware: Full access granted
// Step 1: Validate transaction
// Step 2: Check fraud rules
// Step 3: Log transaction attempt
// Database connection opened: 0.12345678
// Database connection closed: 0.12345678
```

---

## 📊 Interview Questions & Answers

### Q1: What is the difference between app.use() and app.METHOD()?

**Answer:**

**`app.use()`**:
- Matches ALL HTTP methods (GET, POST, PUT, DELETE, etc.)
- Typically used for middleware that should run on every request
- Can match a path prefix (e.g., `/api` matches `/api/users`, `/api/posts`)

```javascript
app.use('/api', middleware); // Runs on all /api/* routes
```

**`app.METHOD()`** (e.g., `app.get()`, `app.post()`):
- Matches specific HTTP method only
- Used for defining route handlers
- Exact path matching (unless wildcards used)

```javascript
app.get('/api/users', handler); // Only matches GET /api/users
```

**Banking Example:**
```javascript
// Authentication runs on ALL API routes
app.use('/api', authenticate);

// Specific transaction endpoint
app.post('/api/transactions', validateTransaction, createTransaction);
```

---

### Q2: What happens if you don't call next() in middleware?

**Answer:**

If you don't call `next()`, the **request hangs** and never reaches the next middleware or route handler. The client will wait until timeout.

**Correct usage:**
```javascript
// Always call next() unless you send a response
app.use((req, res, next) => {
  console.log('Request logged');
  next(); // ✅ Continues to next middleware
});

// Or send a response and don't call next()
app.use((req, res, next) => {
  if (!req.headers.authorization) {
    return res.status(401).json({ error: 'Unauthorized' }); // ✅ Ends here
  }
  next();
});
```

**Wrong usage:**
```javascript
app.use((req, res, next) => {
  console.log('Logging...');
  // ❌ Forgot next() - request hangs forever
});
```

**Banking Impact:**
- Transaction requests hang and timeout
- Users get frustrated waiting
- System appears unresponsive

---

### Q3: What is the order of middleware execution?

**Answer:**

Middleware executes in the **exact order you define it** using `app.use()` or `app.METHOD()`.

**Critical Order:**
```javascript
// 1. Security headers (first!)
app.use(helmet());

// 2. Body parsing (before validation)
app.use(express.json());

// 3. Logging (after parsing)
app.use(logger);

// 4. Authentication (before authorization)
app.use(authenticate);

// 5. Authorization (after authentication)
app.use(authorize);

// 6. Routes (after all middleware)
app.use('/api', routes);

// 7. 404 handler (after routes)
app.use(notFoundHandler);

// 8. Error handler (LAST!)
app.use(errorHandler);
```

**Wrong Order Example:**
```javascript
// ❌ WRONG: Routes before authentication
app.use('/api', routes);
app.use(authenticate); // Never runs for /api routes!

// ✅ CORRECT: Authentication before routes
app.use(authenticate);
app.use('/api', routes);
```

---

### Q4: What is error-handling middleware and how does it differ from regular middleware?

**Answer:**

**Error-handling middleware** has **4 parameters**: `(err, req, res, next)`.

**Regular Middleware (3 params):**
```javascript
function regularMiddleware(req, res, next) {
  // Normal processing
  next();
}
```

**Error Middleware (4 params):**
```javascript
function errorHandler(err, req, res, next) {
  console.error(err.stack);
  res.status(500).json({ error: err.message });
}
```

**Key Differences:**

1. **Signature**: Error middleware has `err` as first parameter
2. **Position**: Must be defined AFTER all routes and regular middleware
3. **Invocation**: Called when `next(err)` is invoked or error is thrown
4. **Purpose**: Centralized error handling

**Banking Example:**
```javascript
// Route with error
app.post('/api/transactions', async (req, res, next) => {
  try {
    const result = await processTransaction(req.body);
    res.json(result);
  } catch (err) {
    next(err); // Passes error to error handler
  }
});

// Error handler catches all errors
app.use((err, req, res, next) => {
  if (err.code === 'INSUFFICIENT_FUNDS') {
    return res.status(400).json({
      error: 'Insufficient funds',
      balance: err.balance,
      required: err.amount
    });
  }

  res.status(500).json({ error: 'Internal server error' });
});
```

---

### Q5: How do you pass data between middleware functions?

**Answer:**

You can attach data to the **`req` object** to share it across middleware.

**Methods:**

1. **Direct property assignment:**
```javascript
app.use((req, res, next) => {
  req.user = { id: 123, username: 'john' };
  next();
});

app.get('/api/profile', (req, res) => {
  res.json({ user: req.user }); // Access shared data
});
```

2. **Using `res.locals` (mainly for views):**
```javascript
app.use((req, res, next) => {
  res.locals.timestamp = new Date();
  next();
});
```

3. **Request context pattern:**
```javascript
app.use((req, res, next) => {
  req.context = {
    requestId: generateId(),
    startTime: Date.now(),
    user: null
  };
  next();
});

app.use(authenticate);

app.use((req, res, next) => {
  req.context.user = req.user; // Add to context
  next();
});
```

**Banking Example:**
```javascript
// Authentication middleware sets user
app.use(authenticate);

app.use((req, res, next) => {
  req.user = { id: 1, accountId: 'ACC001' };
  next();
});

// Transaction middleware uses user data
app.post('/api/transactions', (req, res) => {
  const { accountId } = req.user; // Access from previous middleware
  
  if (req.body.fromAccount !== accountId) {
    return res.status(403).json({ error: 'Not your account' });
  }
  
  // Process transaction...
});
```

---

### Q6: What are the different types of Express middleware?

**Answer:**

**5 Types of Middleware:**

#### 1. **Application-Level Middleware**
Bound to `app` instance:
```javascript
app.use((req, res, next) => {
  console.log('Application middleware');
  next();
});

app.get('/api/users', (req, res) => {
  res.json({ users: [] });
});
```

#### 2. **Router-Level Middleware**
Bound to `express.Router()`:
```javascript
const router = express.Router();

router.use((req, res, next) => {
  console.log('Router middleware');
  next();
});

router.get('/accounts', handler);

app.use('/api', router);
```

#### 3. **Built-in Middleware**
Provided by Express:
```javascript
app.use(express.json()); // Parse JSON
app.use(express.urlencoded({ extended: true })); // Parse form data
app.use(express.static('public')); // Serve static files
```

#### 4. **Third-Party Middleware**
From npm packages:
```javascript
const helmet = require('helmet');
const cors = require('cors');
const morgan = require('morgan');

app.use(helmet()); // Security
app.use(cors()); // CORS
app.use(morgan('combined')); // Logging
```

#### 5. **Error-Handling Middleware**
Has 4 parameters:
```javascript
app.use((err, req, res, next) => {
  console.error(err);
  res.status(500).json({ error: err.message });
});
```

**Banking API Stack:**
```javascript
// 1. Built-in
app.use(express.json());

// 2. Third-party
app.use(helmet());
app.use(cors());

// 3. Application-level (custom)
app.use(logger);
app.use(authenticate);

// 4. Router-level
const accountRouter = express.Router();
accountRouter.use(validateAccount);
app.use('/api/accounts', accountRouter);

// 5. Error-handling
app.use(errorHandler);
```

---

### Q7: How does rate limiting middleware work?

**Answer:**

Rate limiting middleware **tracks request counts per IP/user** and blocks requests that exceed limits.

**Basic Implementation:**
```javascript
function createRateLimiter(windowMs, maxRequests) {
  const requests = new Map(); // IP -> [timestamps]

  return (req, res, next) => {
    const ip = req.ip;
    const now = Date.now();
    
    // Get user's request history
    if (!requests.has(ip)) {
      requests.set(ip, []);
    }

    const userRequests = requests.get(ip);
    
    // Remove expired requests (outside window)
    const validRequests = userRequests.filter(
      time => now - time < windowMs
    );

    // Check if limit exceeded
    if (validRequests.length >= maxRequests) {
      return res.status(429).json({
        error: 'Too many requests',
        retryAfter: Math.ceil((validRequests[0] + windowMs - now) / 1000)
      });
    }

    // Add current request
    validRequests.push(now);
    requests.set(ip, validRequests);
    
    next();
  };
}
```

**Usage:**
```javascript
// 100 requests per 15 minutes
const limiter = createRateLimiter(15 * 60 * 1000, 100);
app.use('/api', limiter);

// Stricter limit for login
const loginLimiter = createRateLimiter(15 * 60 * 1000, 5);
app.post('/api/login', loginLimiter, loginHandler);
```

**Banking Benefits:**
- Prevents brute force attacks on login
- Protects against DDoS
- Prevents transaction spamming
- Ensures fair resource usage

**Production Package:**
```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  message: 'Too many requests',
  standardHeaders: true,
  legacyHeaders: false
});

app.use('/api', limiter);
```

---

### Q8: What is the purpose of helmet middleware?

**Answer:**

**Helmet** sets various **HTTP security headers** to protect your Express app from common vulnerabilities.

**Headers Set by Helmet:**

1. **Content-Security-Policy**: Prevents XSS attacks
2. **X-DNS-Prefetch-Control**: Controls DNS prefetching
3. **X-Frame-Options**: Prevents clickjacking
4. **X-Content-Type-Options**: Prevents MIME type sniffing
5. **Strict-Transport-Security**: Enforces HTTPS
6. **X-Download-Options**: Prevents file download in IE
7. **X-Permitted-Cross-Domain-Policies**: Controls cross-domain access

**Usage:**
```javascript
const helmet = require('helmet');

// Use all default protections
app.use(helmet());

// Or customize
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"]
    }
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true
  }
}));
```

**Before Helmet:**
```
HTTP/1.1 200 OK
Content-Type: application/json
```

**After Helmet:**
```
HTTP/1.1 200 OK
Content-Type: application/json
X-DNS-Prefetch-Control: off
X-Frame-Options: SAMEORIGIN
Strict-Transport-Security: max-age=15552000; includeSubDomains
X-Content-Type-Options: nosniff
X-XSS-Protection: 0
```

**Banking Importance:**
- Protects sensitive financial data
- Prevents XSS attacks that could steal credentials
- Enforces HTTPS for secure communication
- Prevents clickjacking on transaction pages
- Industry compliance requirement (PCI DSS)

---

### Q9: How do you handle async errors in middleware?

**Answer:**

**Problem:** Async errors in middleware are not caught by Express by default.

**Wrong Approach:**
```javascript
app.use(async (req, res, next) => {
  const data = await fetchData(); // ❌ If this throws, Express won't catch it
  req.data = data;
  next();
});
```

**Solution 1: Try-Catch**
```javascript
app.use(async (req, res, next) => {
  try {
    const data = await fetchData();
    req.data = data;
    next();
  } catch (err) {
    next(err); // Pass error to error handler
  }
});
```

**Solution 2: Async Handler Wrapper**
```javascript
function asyncHandler(fn) {
  return (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
}

// Use it
app.use(asyncHandler(async (req, res, next) => {
  const data = await fetchData(); // ✅ Errors automatically caught
  req.data = data;
  next();
}));
```

**Solution 3: Express 5+ (Native Support)**
```javascript
// In Express 5, async errors are caught automatically
app.use(async (req, res, next) => {
  const data = await fetchData(); // ✅ Works in Express 5
  req.data = data;
  next();
});
```

**Banking Example:**
```javascript
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

// Async authentication
app.use(asyncHandler(async (req, res, next) => {
  const token = req.headers.authorization;
  const user = await verifyToken(token); // Async DB call
  req.user = user;
  next();
}));

// Async transaction processing
app.post('/api/transactions', asyncHandler(async (req, res) => {
  const account = await db.findAccount(req.body.accountId);
  const result = await db.createTransaction(req.body);
  res.json({ success: true, data: result });
}));

// Error handler catches all async errors
app.use((err, req, res, next) => {
  if (err.code === 'ECONNREFUSED') {
    return res.status(503).json({ error: 'Database unavailable' });
  }
  res.status(500).json({ error: err.message });
});
```

---

### Q10: What is the difference between `res.send()`, `res.json()`, and `res.end()`?

**Answer:**

#### **`res.send(body)`**
- Sends response of **any type** (string, buffer, object, array)
- Automatically sets `Content-Type` header
- Can be called only once per request

```javascript
res.send('Hello'); // Content-Type: text/html
res.send({ name: 'John' }); // Content-Type: application/json
res.send(Buffer.from('data')); // Content-Type: application/octet-stream
```

#### **`res.json(obj)`**
- Sends **JSON response** specifically
- Always sets `Content-Type: application/json`
- Converts object to JSON string
- Preferred for APIs

```javascript
res.json({ success: true, balance: 5000 });
// Same as:
res.send({ success: true, balance: 5000 });
```

#### **`res.end([data])`**
- Ends response **without data** (or with simple data)
- Does NOT set `Content-Type`
- Used for responses with no body

```javascript
res.status(204).end(); // No content
res.end('Done'); // Ends with string (no content-type)
```

**When to Use Each:**

| Method | Use Case | Banking Example |
|--------|----------|-----------------|
| `res.json()` | **API responses** (recommended) | `res.json({ balance: 5000 })` |
| `res.send()` | Mixed content types | `res.send('<h1>Error</h1>')` |
| `res.end()` | Empty responses | `res.status(204).end()` |

**Best Practice for Banking API:**
```javascript
// ✅ Always use res.json() for API responses
app.get('/api/balance', (req, res) => {
  res.json({
    success: true,
    balance: 5000,
    currency: 'USD'
  });
});

// ✅ Use res.end() for no-content responses
app.delete('/api/accounts/:id', (req, res) => {
  deleteAccount(req.params.id);
  res.status(204).end();
});

// ❌ Don't use res.send() for APIs (inconsistent)
app.get('/api/balance', (req, res) => {
  res.send({ balance: 5000 }); // Works, but use res.json()
});
```

**Calling Multiple Times (ERROR):**
```javascript
// ❌ WRONG: Cannot send response twice
app.get('/api/test', (req, res) => {
  res.json({ message: 'First' });
  res.json({ message: 'Second' }); // Error: Cannot set headers after sent
});

// ✅ CORRECT: Send once
app.get('/api/test', (req, res) => {
  const data = calculateData();
  res.json({ data });
});
```

---

## ✅ DO's - Best Practices

### 1. ✅ **DO: Order Middleware Correctly**

**Why:** Middleware order determines execution flow. Wrong order can break functionality.

**Correct Order:**
```javascript
// 1. Security first
app.use(helmet());

// 2. Body parsing before validation
app.use(express.json());

// 3. Authentication before authorization
app.use(authenticate);
app.use(authorize);

// 4. Routes after middleware
app.use('/api', routes);

// 5. Error handler LAST
app.use(errorHandler);
```

**Banking Impact:**
- Authentication runs before routes (secure)
- Error handler catches all errors
- Body parsing before validation (correct data flow)

---

### 2. ✅ **DO: Always Call next() or Send Response**

**Why:** Not calling `next()` causes requests to hang indefinitely.

**Correct:**
```javascript
app.use((req, res, next) => {
  if (req.headers.authorization) {
    req.authenticated = true;
    next(); // ✅ Continue to next middleware
  } else {
    res.status(401).json({ error: 'Unauthorized' }); // ✅ End request
  }
});
```

**Wrong:**
```javascript
app.use((req, res, next) => {
  if (req.headers.authorization) {
    req.authenticated = true;
    // ❌ Forgot next() - request hangs!
  }
});
```

---

### 3. ✅ **DO: Use Built-in Middleware for Common Tasks**

**Why:** Built-in middleware is optimized, tested, and maintained by Express team.

**Use Built-in:**
```javascript
// ✅ Use express.json() for JSON parsing
app.use(express.json());

// ✅ Use express.urlencoded() for form data
app.use(express.urlencoded({ extended: true }));

// ✅ Use express.static() for serving static files
app.use(express.static('public'));
```

**Don't Reinvent:**
```javascript
// ❌ Don't manually parse JSON
app.use((req, res, next) => {
  let body = '';
  req.on('data', chunk => { body += chunk; });
  req.on('end', () => {
    req.body = JSON.parse(body); // ❌ Use express.json() instead!
    next();
  });
});
```

---

### 4. ✅ **DO: Use Error-Handling Middleware with 4 Parameters**

**Why:** Express only recognizes error handlers with exactly 4 parameters.

**Correct:**
```javascript
// ✅ 4 parameters - Express recognizes as error handler
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: err.message });
});
```

**Wrong:**
```javascript
// ❌ 3 parameters - Not an error handler!
app.use((err, req, res) => {
  console.error(err);
  res.status(500).json({ error: err.message });
});
```

**Banking Example:**
```javascript
app.use((err, req, res, next) => {
  // Log error for auditing
  console.error({
    timestamp: new Date(),
    user: req.user?.username,
    error: err.message,
    stack: err.stack
  });

  // Send appropriate response
  if (err.code === 'INSUFFICIENT_FUNDS') {
    return res.status(400).json({
      error: 'Transaction failed',
      reason: 'Insufficient funds'
    });
  }

  res.status(500).json({ error: 'Internal server error' });
});
```

---

### 5. ✅ **DO: Use Helmet for Security Headers**

**Why:** Helmet protects against common web vulnerabilities.

**Implementation:**
```javascript
const helmet = require('helmet');

// ✅ Enable all helmet protections
app.use(helmet());

// Or customize
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'"]
    }
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  }
}));
```

**Banking Importance:**
- Prevents XSS attacks on transaction pages
- Enforces HTTPS for secure communication
- Prevents clickjacking on sensitive operations
- Required for PCI DSS compliance

---

### 6. ✅ **DO: Implement Rate Limiting**

**Why:** Prevents brute force attacks and API abuse.

**Implementation:**
```javascript
const rateLimit = require('express-rate-limit');

// General API rate limit
const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // 100 requests per window
  message: 'Too many requests, please try again later'
});

app.use('/api/', apiLimiter);

// Stricter limit for sensitive endpoints
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5, // Only 5 login attempts
  skipSuccessfulRequests: true
});

app.post('/api/login', authLimiter, loginHandler);
```

**Banking Protection:**
- Prevents brute force on login (5 attempts per 15 min)
- Limits transaction API calls (prevents spamming)
- Protects against DDoS attacks
- Ensures fair resource usage

---

### 7. ✅ **DO: Validate Input with express-validator**

**Why:** Input validation prevents injection attacks and ensures data integrity.

**Implementation:**
```javascript
const { body, validationResult } = require('express-validator');

const validateTransaction = [
  body('amount')
    .isFloat({ min: 0.01, max: 1000000 })
    .withMessage('Amount must be between 0.01 and 1,000,000'),
  
  body('toAccount')
    .matches(/^ACC\d{3,6}$/)
    .withMessage('Invalid account format'),
  
  (req, res, next) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }
    next();
  }
];

app.post('/api/transactions', validateTransaction, transactionHandler);
```

**Banking Benefits:**
- Prevents invalid transaction amounts
- Validates account number formats
- Sanitizes user input (prevents XSS)
- Ensures data consistency

---

### 8. ✅ **DO: Use Async Handler Wrapper for Async Routes**

**Why:** Prevents unhandled promise rejections.

**Implementation:**
```javascript
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

// Use it for async routes
app.post('/api/transactions', asyncHandler(async (req, res) => {
  const account = await db.findAccount(req.body.accountId);
  
  if (account.balance < req.body.amount) {
    throw new Error('Insufficient funds');
  }
  
  const result = await db.createTransaction(req.body);
  res.json({ success: true, data: result });
}));

// Error handler catches all errors
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message });
});
```

---

### 9. ✅ **DO: Log Requests with Proper Middleware**

**Why:** Logging helps debug issues, audit actions, and monitor performance.

**Implementation:**
```javascript
const morgan = require('morgan');
const fs = require('fs');
const path = require('path');

// Create log directory
const logDirectory = path.join(__dirname, 'logs');
fs.existsSync(logDirectory) || fs.mkdirSync(logDirectory);

// Console logging (development)
if (process.env.NODE_ENV === 'development') {
  app.use(morgan('dev'));
}

// File logging (production)
const accessLogStream = fs.createWriteStream(
  path.join(logDirectory, 'access.log'),
  { flags: 'a' }
);
app.use(morgan('combined', { stream: accessLogStream }));

// Custom banking logger
app.use((req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    const duration = Date.now() - start;
    console.log({
      method: req.method,
      url: req.url,
      status: res.statusCode,
      duration: `${duration}ms`,
      user: req.user?.username || 'anonymous'
    });
  });
  next();
});
```

**Banking Audit Trail:**
- Who performed the action
- What action was performed
- When it was performed
- Response status and duration
- Required for compliance (SOX, PCI DSS)

---

### 10. ✅ **DO: Use Router for Modular Code Organization**

**Why:** Routers keep code organized, maintainable, and testable.

**Implementation:**
```javascript
// routes/accounts.js
const express = require('express');
const router = express.Router();

router.use(authenticate); // Authentication for all account routes

router.get('/', getAllAccounts);
router.get('/:id', getAccount);
router.post('/', createAccount);
router.put('/:id', updateAccount);
router.delete('/:id', closeAccount);

module.exports = router;

// routes/transactions.js
const router = express.Router();

router.use(authenticate);

router.get('/', getTransactions);
router.post('/', validateTransaction, createTransaction);

module.exports = router;

// server.js
const accountRouter = require('./routes/accounts');
const transactionRouter = require('./routes/transactions');

app.use('/api/accounts', accountRouter);
app.use('/api/transactions', transactionRouter);
```

**Benefits:**
- Each router is a separate file (easy to maintain)
- Middleware can be router-specific
- Easy to test individual routers
- Clear separation of concerns

---

## ❌ DON'Ts - Common Mistakes

### 1. ❌ **DON'T: Define Error Handler Before Routes**

**Why:** Error handlers must be defined AFTER all routes to catch errors.

**Wrong:**
```javascript
// ❌ Error handler before routes - won't catch route errors!
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message });
});

app.get('/api/accounts', (req, res) => {
  throw new Error('Something went wrong'); // Not caught!
});
```

**Correct:**
```javascript
// ✅ Routes first
app.get('/api/accounts', (req, res) => {
  throw new Error('Something went wrong');
});

// ✅ Error handler last - catches all errors
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message });
});
```

---

### 2. ❌ **DON'T: Send Response and Then Call next()**

**Why:** Causes "Cannot set headers after they are sent" error.

**Wrong:**
```javascript
app.use((req, res, next) => {
  res.json({ message: 'Hello' });
  next(); // ❌ Error: headers already sent!
});
```

**Correct:**
```javascript
app.use((req, res, next) => {
  if (someCondition) {
    return res.json({ message: 'Hello' }); // ✅ Return after response
  }
  next(); // ✅ Only call next() if no response sent
});
```

---

### 3. ❌ **DON'T: Trust User Input Without Validation**

**Why:** Unvalidated input can lead to injection attacks and data corruption.

**Wrong:**
```javascript
app.post('/api/transactions', (req, res) => {
  const { amount, toAccount } = req.body;
  
  // ❌ No validation - dangerous!
  db.createTransaction({
    amount,
    toAccount,
    fromAccount: req.user.accountId
  });
});
```

**Correct:**
```javascript
const { body, validationResult } = require('express-validator');

app.post('/api/transactions',
  body('amount').isFloat({ min: 0.01, max: 1000000 }),
  body('toAccount').matches(/^ACC\d{3,6}$/),
  (req, res) => {
    const errors = validationResult(req);
    
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }
    
    // ✅ Safe to use validated data
    db.createTransaction(req.body);
  }
);
```

---

### 4. ❌ **DON'T: Forget to Handle Async Errors**

**Why:** Unhandled promise rejections crash your application.

**Wrong:**
```javascript
app.get('/api/accounts/:id', async (req, res) => {
  const account = await db.findAccount(req.params.id); // ❌ If this throws, app crashes!
  res.json(account);
});
```

**Correct:**
```javascript
// Option 1: Try-catch
app.get('/api/accounts/:id', async (req, res, next) => {
  try {
    const account = await db.findAccount(req.params.id);
    res.json(account);
  } catch (err) {
    next(err); // ✅ Pass to error handler
  }
});

// Option 2: Async wrapper
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

app.get('/api/accounts/:id', asyncHandler(async (req, res) => {
  const account = await db.findAccount(req.params.id);
  res.json(account);
}));
```

---

### 5. ❌ **DON'T: Expose Sensitive Error Details in Production**

**Why:** Error stack traces can reveal system architecture and vulnerabilities.

**Wrong:**
```javascript
app.use((err, req, res, next) => {
  // ❌ Exposes internal details
  res.status(500).json({
    error: err.message,
    stack: err.stack,
    query: req.query,
    body: req.body
  });
});
```

**Correct:**
```javascript
app.use((err, req, res, next) => {
  // ✅ Log full error internally
  console.error({
    error: err,
    stack: err.stack,
    url: req.url,
    method: req.method,
    user: req.user?.username
  });

  // ✅ Send generic error to client (production)
  if (process.env.NODE_ENV === 'production') {
    return res.status(500).json({
      error: 'Internal server error',
      errorId: generateErrorId()
    });
  }

  // ✅ Detailed errors in development
  res.status(500).json({
    error: err.message,
    stack: err.stack
  });
});
```

---

### 6. ❌ **DON'T: Use Synchronous Operations in Middleware**

**Why:** Synchronous operations block the event loop and kill performance.

**Wrong:**
```javascript
const fs = require('fs');

app.use((req, res, next) => {
  // ❌ Blocks event loop!
  const data = fs.readFileSync('config.json');
  req.config = JSON.parse(data);
  next();
});
```

**Correct:**
```javascript
const fs = require('fs').promises;

const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

app.use(asyncHandler(async (req, res, next) => {
  // ✅ Async - doesn't block
  const data = await fs.readFile('config.json', 'utf8');
  req.config = JSON.parse(data);
  next();
}));
```

---

### 7. ❌ **DON'T: Parse JSON Body Manually**

**Why:** Express has optimized built-in middleware for this.

**Wrong:**
```javascript
app.use((req, res, next) => {
  // ❌ Reinventing the wheel
  let body = '';
  req.on('data', chunk => { body += chunk.toString(); });
  req.on('end', () => {
    try {
      req.body = JSON.parse(body);
      next();
    } catch (err) {
      next(err);
    }
  });
});
```

**Correct:**
```javascript
// ✅ Use built-in middleware
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
```

---

### 8. ❌ **DON'T: Modify req or res Prototype**

**Why:** Modifying prototypes can cause conflicts and unpredictable behavior.

**Wrong:**
```javascript
// ❌ Modifying Express prototypes
app.use((req, res, next) => {
  res.__proto__.sendSuccess = function(data) {
    this.json({ success: true, data });
  };
  next();
});
```

**Correct:**
```javascript
// ✅ Add helper function to res object
app.use((req, res, next) => {
  res.sendSuccess = function(data) {
    return this.json({ success: true, data });
  };
  next();
});

// Or create a helper module
function sendSuccess(res, data) {
  return res.json({ success: true, data });
}
```

---

### 9. ❌ **DON'T: Ignore Content-Type Headers**

**Why:** Wrong Content-Type can break request parsing.

**Wrong:**
```javascript
// ❌ Assuming all requests are JSON
app.post('/api/transactions', (req, res) => {
  const { amount } = req.body; // undefined if Content-Type is wrong!
  // ...
});
```

**Correct:**
```javascript
// ✅ Check Content-Type
app.post('/api/transactions', (req, res) => {
  if (!req.is('application/json')) {
    return res.status(400).json({
      error: 'Content-Type must be application/json'
    });
  }
  
  const { amount } = req.body;
  // ...
});

// Or use middleware
app.use((req, res, next) => {
  if (req.method === 'POST' && !req.is('application/json')) {
    return res.status(400).json({
      error: 'Content-Type must be application/json'
    });
  }
  next();
});
```

---

### 10. ❌ **DON'T: Perform Heavy Computation in Middleware**

**Why:** Middleware blocks the event loop. Move heavy work to worker threads or queues.

**Wrong:**
```javascript
app.use((req, res, next) => {
  // ❌ Heavy computation blocks all requests!
  const hash = bcrypt.hashSync(req.body.password, 10); // Synchronous!
  req.hashedPassword = hash;
  next();
});
```

**Correct:**
```javascript
// ✅ Use async version
app.use(asyncHandler(async (req, res, next) => {
  const hash = await bcrypt.hash(req.body.password, 10); // Async!
  req.hashedPassword = hash;
  next();
}));

// Or move to route handler
app.post('/api/register', async (req, res) => {
  const hash = await bcrypt.hash(req.body.password, 10);
  await db.createUser({ ...req.body, password: hash });
  res.json({ success: true });
});
```

---

## 🎯 Key Takeaways

1. **Middleware Order Matters**: Always define middleware in the correct order (security → parsing → auth → routes → error handler)

2. **Always Call next()**: Either call `next()` or send a response, never both

3. **4-Parameter Error Handlers**: Error-handling middleware MUST have 4 parameters: `(err, req, res, next)`

4. **Use Built-in Middleware**: Prefer Express built-in middleware (`express.json()`, `express.static()`) over custom implementations

5. **Security First**: Use `helmet` for security headers and rate limiting to prevent abuse

6. **Validate All Input**: Never trust user input. Use `express-validator` or similar libraries

7. **Handle Async Errors**: Always wrap async routes with try-catch or async handler wrapper

8. **Don't Block Event Loop**: Use async operations, never synchronous file I/O or heavy computation

9. **Log Everything**: Implement comprehensive logging for debugging, auditing, and compliance

10. **Modular Organization**: Use Express Router to organize routes into separate modules

---

## 📚 Summary

**Express.js middleware** is the backbone of request processing. Key concepts:

### Middleware Types:
1. **Application-level**: `app.use(middleware)`
2. **Router-level**: `router.use(middleware)`
3. **Built-in**: `express.json()`, `express.static()`
4. **Third-party**: `helmet`, `cors`, `morgan`
5. **Error-handling**: 4-parameter middleware

### Critical Patterns:
- **Order**: Security → Parsing → Auth → Routes → 404 → Error Handler
- **Execution**: Synchronous chain via `next()` calls
- **Error Flow**: `next(err)` → Error Handler
- **Async Safety**: Use try-catch or async wrappers

### Banking Application Stack:
```javascript
app.use(helmet());                    // Security headers
app.use(cors());                      // CORS
app.use(express.json());              // JSON parsing
app.use(morgan('combined'));          // HTTP logging
app.use(rateLimit());                 // Rate limiting
app.use(authenticate);                // Authentication
app.use(authorize);                   // Authorization
app.use('/api/accounts', accountRouter);    // Routes
app.use('/api/transactions', transRouter);  // Routes
app.use(notFoundHandler);             // 404
app.use(errorHandler);                // Error handling
```

### Production Checklist:
- ✅ Security headers (helmet)
- ✅ Rate limiting (express-rate-limit)
- ✅ Input validation (express-validator)
- ✅ Authentication & authorization
- ✅ Request logging (morgan)
- ✅ Error handling (centralized)
- ✅ Async error handling
- ✅ CORS configuration
- ✅ Compression
- ✅ Response time tracking

**Middleware is the heart of Express.js** - master it to build secure, scalable banking APIs! 🏦

---

## 📖 Additional Resources

- [Express.js Official Middleware Guide](https://expressjs.com/en/guide/using-middleware.html)
- [Express.js Error Handling](https://expressjs.com/en/guide/error-handling.html)
- [Helmet.js Security](https://helmetjs.github.io/)
- [Express Validator](https://express-validator.github.io/)
- [Express Rate Limit](https://www.npmjs.com/package/express-rate-limit)

