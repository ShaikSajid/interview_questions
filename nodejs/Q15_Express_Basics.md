# Node.js Interview Question: Express.js Basics
## Question 15: Why Use Express.js Over Raw HTTP? (Routing, Middleware, Banking API)

---

## 📋 Summary of What Will Be Covered

In this comprehensive guide, you'll learn:

1. **Why Express.js?** - Benefits over raw Node.js HTTP module
2. **Express Fundamentals** - Application structure and setup
3. **Routing** - Defining routes, route parameters, route handlers
4. **Middleware** - Built-in, third-party, and custom middleware
5. **Request/Response Objects** - Enhanced req/res with Express
6. **Error Handling** - Centralized error handling middleware
7. **Banking Example**: Complete banking API with authentication, validation, and error handling

**Why This Matters in Banking**:
- Express simplifies API development with clean routing
- Middleware enables authentication, logging, and validation
- Production-ready error handling for financial operations
- Request validation prevents invalid transactions
- Structured routing improves code maintainability
- Built-in security features for banking compliance

---

## 🎯 Express.js vs Raw HTTP Module

### The Problem with Raw HTTP

```javascript
// Raw HTTP - Complex and verbose
import http from 'http';
import url from 'url';

const server = http.createServer((req, res) => {
  const parsedUrl = url.parse(req.url, true);
  const pathname = parsedUrl.pathname;
  const method = req.method;
  
  // Manual routing
  if (method === 'GET' && pathname === '/api/accounts') {
    // Handle GET accounts
  } else if (method === 'POST' && pathname === '/api/accounts') {
    // Parse body manually
    let body = '';
    req.on('data', chunk => body += chunk);
    req.on('end', () => {
      try {
        const data = JSON.parse(body);
        // Handle POST account
      } catch (error) {
        // Handle error
      }
    });
  } else if (method === 'GET' && pathname.match(/^\/api\/accounts\/[^/]+$/)) {
    // Extract ID manually
    const id = pathname.split('/')[3];
    // Handle GET account by ID
  } else {
    res.writeHead(404);
    res.end('Not Found');
  }
});
```

### The Express Solution

```javascript
// Express - Clean and concise
import express from 'express';

const app = express();

// Automatic body parsing
app.use(express.json());

// Clean routing
app.get('/api/accounts', (req, res) => {
  // Handle GET accounts
});

app.post('/api/accounts', (req, res) => {
  const data = req.body;  // Already parsed!
  // Handle POST account
});

app.get('/api/accounts/:id', (req, res) => {
  const id = req.params.id;  // Automatically extracted!
  // Handle GET account by ID
});

// Automatic 404 handling
```

---

## 📖 Visual: Express Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     EXPRESS APPLICATION                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │   HTTP REQUEST    │
                    └─────────┬─────────┘
                              ↓
                    ┌─────────────────────┐
                    │  MIDDLEWARE STACK   │
                    ├─────────────────────┤
                    │  1. Body Parser     │ → app.use(express.json())
                    │  2. CORS Handler    │ → app.use(cors())
                    │  3. Authentication  │ → app.use(authenticate)
                    │  4. Logging         │ → app.use(logger)
                    │  5. Rate Limiting   │ → app.use(rateLimit)
                    └─────────┬───────────┘
                              ↓
                    ┌─────────────────────┐
                    │   ROUTER MATCHING   │
                    ├─────────────────────┤
                    │  GET  /accounts     │
                    │  POST /accounts     │
                    │  GET  /accounts/:id │
                    └─────────┬───────────┘
                              ↓
                    ┌─────────────────────┐
                    │   ROUTE HANDLER     │
                    ├─────────────────────┤
                    │  Business Logic     │
                    │  Database Query     │
                    │  Response           │
                    └─────────┬───────────┘
                              ↓
                    ┌─────────────────────┐
                    │  ERROR MIDDLEWARE   │ (if error occurs)
                    ├─────────────────────┤
                    │  Log Error          │
                    │  Format Response    │
                    │  Send to Client     │
                    └─────────┬───────────┘
                              ↓
                    ┌─────────────────────┐
                    │   HTTP RESPONSE     │
                    └─────────────────────┘
```

---

## 🚀 Why Express Over Raw HTTP?

### 1. **Simplified Routing**

```javascript
// ❌ Raw HTTP - Manual route matching
if (method === 'GET' && pathname === '/users') { }
else if (method === 'POST' && pathname === '/users') { }
else if (method === 'GET' && pathname.match(/^\/users\/\d+$/)) { }

// ✅ Express - Declarative routing
app.get('/users', handler);
app.post('/users', handler);
app.get('/users/:id', handler);
```

### 2. **Automatic Body Parsing**

```javascript
// ❌ Raw HTTP - Manual body parsing
let body = '';
req.on('data', chunk => body += chunk);
req.on('end', () => {
  const data = JSON.parse(body);
});

// ✅ Express - Automatic
app.use(express.json());
app.post('/users', (req, res) => {
  const data = req.body;  // Already parsed!
});
```

### 3. **Middleware System**

```javascript
// ❌ Raw HTTP - No middleware support
// Must manually call each function

// ✅ Express - Middleware chain
app.use(authenticate);
app.use(logRequest);
app.use(validateInput);
```

### 4. **Enhanced Request/Response**

```javascript
// ❌ Raw HTTP - Manual response formatting
res.writeHead(200, { 'Content-Type': 'application/json' });
res.end(JSON.stringify({ data: users }));

// ✅ Express - Convenient methods
res.json({ data: users });
res.status(404).json({ error: 'Not found' });
```

### 5. **Route Parameters**

```javascript
// ❌ Raw HTTP - Manual extraction
const id = pathname.split('/')[2];

// ✅ Express - Automatic extraction
app.get('/users/:id', (req, res) => {
  const id = req.params.id;
});
```

### 6. **Query String Parsing**

```javascript
// ❌ Raw HTTP - Manual parsing
const query = url.parse(req.url, true).query;

// ✅ Express - Built-in parsing
app.get('/users', (req, res) => {
  const { page, limit } = req.query;
});
```

### 7. **Error Handling**

```javascript
// ❌ Raw HTTP - Scattered error handling
try {
  // Handle request
} catch (error) {
  res.writeHead(500);
  res.end('Error');
}

// ✅ Express - Centralized error handling
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message });
});
```

---

## 🏗️ Express Fundamentals

### Installing Express

```bash
# Initialize project
npm init -y

# Install Express
npm install express

# Install additional packages
npm install cors dotenv helmet
```

### Basic Express Application

```javascript
import express from 'express';

const app = express();
const PORT = 3000;

// Middleware
app.use(express.json());

// Routes
app.get('/', (req, res) => {
  res.json({ message: 'Hello Express!' });
});

// Start server
app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
```

---

## 🛣️ Routing in Express

### Basic Routes

```javascript
// HTTP methods
app.get('/accounts', (req, res) => {
  res.json({ message: 'GET all accounts' });
});

app.post('/accounts', (req, res) => {
  res.json({ message: 'POST create account' });
});

app.put('/accounts/:id', (req, res) => {
  res.json({ message: `PUT update account ${req.params.id}` });
});

app.patch('/accounts/:id', (req, res) => {
  res.json({ message: `PATCH update account ${req.params.id}` });
});

app.delete('/accounts/:id', (req, res) => {
  res.json({ message: `DELETE account ${req.params.id}` });
});
```

### Route Parameters

```javascript
// Single parameter
app.get('/accounts/:id', (req, res) => {
  const { id } = req.params;
  res.json({ accountId: id });
});

// Multiple parameters
app.get('/accounts/:accountId/transactions/:transactionId', (req, res) => {
  const { accountId, transactionId } = req.params;
  res.json({ accountId, transactionId });
});

// Optional parameters (using regex)
app.get('/accounts/:id/:details?', (req, res) => {
  const { id, details } = req.params;
  res.json({ id, details: details || 'basic' });
});
```

### Query Parameters

```javascript
// GET /accounts?page=2&limit=10&status=active
app.get('/accounts', (req, res) => {
  const { page = 1, limit = 10, status } = req.query;
  
  res.json({
    page: parseInt(page),
    limit: parseInt(limit),
    status
  });
});
```

### Route Chaining

```javascript
app.route('/accounts/:id')
  .get((req, res) => {
    res.json({ message: 'GET account' });
  })
  .put((req, res) => {
    res.json({ message: 'PUT account' });
  })
  .delete((req, res) => {
    res.json({ message: 'DELETE account' });
  });
```

---

## 🔧 Middleware in Express

### What is Middleware?

Middleware functions have access to:
- `req` (request object)
- `res` (response object)
- `next` (next middleware function)

```javascript
function middleware(req, res, next) {
  // Do something
  console.log('Middleware executed');
  
  // Pass control to next middleware
  next();
}

app.use(middleware);
```

### Built-in Middleware

```javascript
// Parse JSON bodies
app.use(express.json());

// Parse URL-encoded bodies
app.use(express.urlencoded({ extended: true }));

// Serve static files
app.use(express.static('public'));
```

### Third-party Middleware

```javascript
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';

// CORS
app.use(cors());

// Security headers
app.use(helmet());

// HTTP request logger
app.use(morgan('combined'));
```

### Custom Middleware

```javascript
// Logging middleware
function logger(req, res, next) {
  console.log(`${req.method} ${req.url}`);
  next();
}

// Authentication middleware
function authenticate(req, res, next) {
  const token = req.headers.authorization;
  
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }
  
  // Verify token
  req.user = { id: '12345' };  // Decoded user
  next();
}

// Validation middleware
function validateAccount(req, res, next) {
  const { accountNumber, customerName } = req.body;
  
  if (!accountNumber || !customerName) {
    return res.status(400).json({ error: 'Missing required fields' });
  }
  
  next();
}

// Usage
app.use(logger);
app.post('/accounts', authenticate, validateAccount, (req, res) => {
  res.json({ message: 'Account created' });
});
```

### Middleware Order

```javascript
// Global middleware (runs for all routes)
app.use(logger);
app.use(express.json());

// Route-specific middleware
app.get('/accounts', authenticate, (req, res) => {
  // Only authenticated users reach here
});

// Error handling middleware (must be last!)
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message });
});
```

---

## 📝 Request and Response Objects

### Request Object (req)

```javascript
app.post('/accounts/:id', (req, res) => {
  // Route parameters
  const { id } = req.params;
  
  // Query parameters
  const { details } = req.query;
  
  // Request body
  const { accountNumber, balance } = req.body;
  
  // Headers
  const token = req.headers.authorization;
  const contentType = req.headers['content-type'];
  
  // URL information
  const url = req.url;           // '/accounts/123?details=true'
  const path = req.path;         // '/accounts/123'
  const method = req.method;     // 'POST'
  const protocol = req.protocol; // 'http' or 'https'
  
  // Other properties
  const ip = req.ip;
  const hostname = req.hostname;
  
  res.json({ received: 'ok' });
});
```

### Response Object (res)

```javascript
app.get('/accounts', (req, res) => {
  // Send JSON
  res.json({ data: [] });
  
  // Set status code
  res.status(201).json({ created: true });
  
  // Send plain text
  res.send('Hello');
  
  // Set headers
  res.set('X-Custom-Header', 'value');
  
  // Redirect
  res.redirect('/login');
  
  // Send file
  res.sendFile('/path/to/file.pdf');
  
  // Download file
  res.download('/path/to/file.pdf', 'statement.pdf');
  
  // Set cookie
  res.cookie('sessionId', '12345', { httpOnly: true });
  
  // Clear cookie
  res.clearCookie('sessionId');
  
  // End response
  res.end();
});
```

---

## 🏦 Example 1: Complete Banking API with Express

This example demonstrates a production-ready banking API with routing, middleware, validation, and error handling.

**File: `banking-api-express.js`**

```javascript
/**
 * Banking API with Express.js
 * Demonstrates: Routing, middleware, validation, error handling
 */

import express from 'express';
import crypto from 'crypto';

console.log('🏦 Banking API with Express.js\n');
console.log('='.repeat(70));

// ============================================
// Initialize Express App
// ============================================

const app = express();
const PORT = 3000;

// ============================================
// In-Memory Database
// ============================================

const accounts = new Map();
const transactions = new Map();

// Seed data
accounts.set('ACC-001', {
  id: 'ACC-001',
  accountNumber: '1234567890',
  customerName: 'John Doe',
  accountType: 'SAVINGS',
  balance: 5000,
  currency: 'USD',
  status: 'ACTIVE',
  createdAt: new Date('2025-01-01').toISOString()
});

accounts.set('ACC-002', {
  id: 'ACC-002',
  accountNumber: '0987654321',
  customerName: 'Jane Smith',
  accountType: 'CHECKING',
  balance: 10000,
  currency: 'USD',
  status: 'ACTIVE',
  createdAt: new Date('2025-01-02').toISOString()
});

// ============================================
// Built-in Middleware
// ============================================

// Parse JSON bodies
app.use(express.json({ limit: '1mb' }));

// Parse URL-encoded bodies
app.use(express.urlencoded({ extended: true, limit: '1mb' }));

// ============================================
// Custom Middleware
// ============================================

/**
 * Request Logger Middleware
 */
function requestLogger(req, res, next) {
  const requestId = crypto.randomUUID();
  req.requestId = requestId;
  
  const start = Date.now();
  
  console.log(`\n[${requestId}] ${req.method} ${req.url}`);
  console.log(`   IP: ${req.ip}`);
  console.log(`   User-Agent: ${req.headers['user-agent']}`);
  
  // Log response
  res.on('finish', () => {
    const duration = Date.now() - start;
    console.log(`[${requestId}] Response: ${res.statusCode} (${duration}ms)`);
  });
  
  next();
}

/**
 * CORS Middleware
 */
function corsMiddleware(req, res, next) {
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, PATCH, DELETE, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
  
  if (req.method === 'OPTIONS') {
    return res.status(204).end();
  }
  
  next();
}

/**
 * Authentication Middleware
 */
function authenticate(req, res, next) {
  const token = req.headers.authorization;
  
  if (!token) {
    return res.status(401).json({
      error: {
        message: 'Authentication required',
        code: 'UNAUTHORIZED'
      }
    });
  }
  
  // Validate token (simplified)
  if (token !== 'Bearer valid-token-12345') {
    return res.status(401).json({
      error: {
        message: 'Invalid token',
        code: 'INVALID_TOKEN'
      }
    });
  }
  
  // Set user info
  req.user = {
    id: 'USER-001',
    email: 'user@example.com'
  };
  
  next();
}

/**
 * Rate Limiting Middleware
 */
const requestCounts = new Map();

function rateLimit(maxRequests = 10, windowMs = 60000) {
  return (req, res, next) => {
    const ip = req.ip;
    const now = Date.now();
    
    if (!requestCounts.has(ip)) {
      requestCounts.set(ip, []);
    }
    
    const requests = requestCounts.get(ip);
    
    // Remove old requests outside window
    const recentRequests = requests.filter(timestamp => now - timestamp < windowMs);
    
    if (recentRequests.length >= maxRequests) {
      return res.status(429).json({
        error: {
          message: 'Too many requests',
          code: 'RATE_LIMIT_EXCEEDED',
          retryAfter: Math.ceil(windowMs / 1000)
        }
      });
    }
    
    recentRequests.push(now);
    requestCounts.set(ip, recentRequests);
    
    next();
  };
}

/**
 * Validation Middleware Factory
 */
function validateBody(schema) {
  return (req, res, next) => {
    const errors = [];
    
    for (const [field, rules] of Object.entries(schema)) {
      const value = req.body[field];
      
      if (rules.required && (value === undefined || value === null || value === '')) {
        errors.push(`${field} is required`);
        continue;
      }
      
      if (value !== undefined && value !== null) {
        if (rules.type && typeof value !== rules.type) {
          errors.push(`${field} must be ${rules.type}`);
        }
        
        if (rules.min !== undefined && value < rules.min) {
          errors.push(`${field} must be at least ${rules.min}`);
        }
        
        if (rules.max !== undefined && value > rules.max) {
          errors.push(`${field} must be at most ${rules.max}`);
        }
        
        if (rules.pattern && !rules.pattern.test(value)) {
          errors.push(`${field} has invalid format`);
        }
      }
    }
    
    if (errors.length > 0) {
      return res.status(400).json({
        error: {
          message: 'Validation failed',
          code: 'VALIDATION_ERROR',
          details: errors
        }
      });
    }
    
    next();
  };
}

// ============================================
// Apply Global Middleware
// ============================================

app.use(requestLogger);
app.use(corsMiddleware);
app.use(rateLimit(100, 60000));  // 100 requests per minute

// ============================================
// Routes
// ============================================

/**
 * Health Check
 */
app.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: process.uptime()
  });
});

/**
 * GET /api/accounts - List all accounts
 */
app.get('/api/accounts', authenticate, (req, res) => {
  const { page = 1, limit = 10, status, type } = req.query;
  
  // Filter accounts
  let filteredAccounts = Array.from(accounts.values());
  
  if (status) {
    filteredAccounts = filteredAccounts.filter(acc => acc.status === status);
  }
  
  if (type) {
    filteredAccounts = filteredAccounts.filter(acc => acc.accountType === type);
  }
  
  // Pagination
  const startIndex = (parseInt(page) - 1) * parseInt(limit);
  const endIndex = startIndex + parseInt(limit);
  const paginatedAccounts = filteredAccounts.slice(startIndex, endIndex);
  
  res.json({
    data: paginatedAccounts,
    pagination: {
      page: parseInt(page),
      limit: parseInt(limit),
      total: filteredAccounts.length,
      totalPages: Math.ceil(filteredAccounts.length / parseInt(limit))
    },
    metadata: {
      timestamp: new Date().toISOString(),
      requestId: req.requestId
    }
  });
});

/**
 * GET /api/accounts/:id - Get specific account
 */
app.get('/api/accounts/:id', authenticate, (req, res) => {
  const { id } = req.params;
  
  const account = accounts.get(id);
  
  if (!account) {
    return res.status(404).json({
      error: {
        message: 'Account not found',
        code: 'ACCOUNT_NOT_FOUND',
        accountId: id
      }
    });
  }
  
  res.json({
    data: account,
    metadata: {
      timestamp: new Date().toISOString(),
      requestId: req.requestId
    }
  });
});

/**
 * POST /api/accounts - Create new account
 */
app.post(
  '/api/accounts',
  authenticate,
  validateBody({
    accountNumber: { required: true, type: 'string', pattern: /^\d{10}$/ },
    customerName: { required: true, type: 'string' },
    accountType: { required: true, type: 'string' },
    balance: { type: 'number', min: 0 },
    currency: { type: 'string' }
  }),
  (req, res) => {
    const { accountNumber, customerName, accountType, balance = 0, currency = 'USD' } = req.body;
    
    // Check for duplicate account number
    const existingAccount = Array.from(accounts.values())
      .find(acc => acc.accountNumber === accountNumber);
    
    if (existingAccount) {
      return res.status(409).json({
        error: {
          message: 'Account number already exists',
          code: 'DUPLICATE_ACCOUNT',
          accountNumber
        }
      });
    }
    
    // Create account
    const account = {
      id: `ACC-${crypto.randomBytes(4).toString('hex').toUpperCase()}`,
      accountNumber,
      customerName,
      accountType,
      balance,
      currency,
      status: 'ACTIVE',
      createdAt: new Date().toISOString()
    };
    
    accounts.set(account.id, account);
    
    res.status(201).json({
      message: 'Account created successfully',
      data: account,
      metadata: {
        timestamp: new Date().toISOString(),
        requestId: req.requestId
      }
    });
  }
);

/**
 * PUT /api/accounts/:id - Replace account
 */
app.put(
  '/api/accounts/:id',
  authenticate,
  validateBody({
    accountNumber: { required: true, type: 'string' },
    customerName: { required: true, type: 'string' },
    accountType: { required: true, type: 'string' },
    balance: { required: true, type: 'number', min: 0 }
  }),
  (req, res) => {
    const { id } = req.params;
    const { accountNumber, customerName, accountType, balance, currency = 'USD' } = req.body;
    
    const existingAccount = accounts.get(id);
    
    if (!existingAccount) {
      return res.status(404).json({
        error: {
          message: 'Account not found',
          code: 'ACCOUNT_NOT_FOUND'
        }
      });
    }
    
    // Replace account
    const account = {
      id,
      accountNumber,
      customerName,
      accountType,
      balance,
      currency,
      status: existingAccount.status,
      createdAt: existingAccount.createdAt,
      updatedAt: new Date().toISOString()
    };
    
    accounts.set(id, account);
    
    res.json({
      message: 'Account updated successfully',
      data: account,
      metadata: {
        timestamp: new Date().toISOString(),
        requestId: req.requestId
      }
    });
  }
);

/**
 * PATCH /api/accounts/:id - Update account
 */
app.patch('/api/accounts/:id', authenticate, (req, res) => {
  const { id } = req.params;
  
  const existingAccount = accounts.get(id);
  
  if (!existingAccount) {
    return res.status(404).json({
      error: {
        message: 'Account not found',
        code: 'ACCOUNT_NOT_FOUND'
      }
    });
  }
  
  // Update only provided fields
  const account = {
    ...existingAccount,
    ...req.body,
    id,  // Don't allow changing ID
    createdAt: existingAccount.createdAt,  // Don't allow changing creation date
    updatedAt: new Date().toISOString()
  };
  
  accounts.set(id, account);
  
  res.json({
    message: 'Account updated successfully',
    data: account,
    metadata: {
      timestamp: new Date().toISOString(),
      requestId: req.requestId
    }
  });
});

/**
 * DELETE /api/accounts/:id - Delete account
 */
app.delete('/api/accounts/:id', authenticate, (req, res) => {
  const { id } = req.params;
  
  const account = accounts.get(id);
  
  if (!account) {
    return res.status(404).json({
      error: {
        message: 'Account not found',
        code: 'ACCOUNT_NOT_FOUND'
      }
    });
  }
  
  // Check if account has balance
  if (account.balance > 0) {
    return res.status(400).json({
      error: {
        message: 'Cannot delete account with non-zero balance',
        code: 'ACCOUNT_HAS_BALANCE',
        balance: account.balance
      }
    });
  }
  
  accounts.delete(id);
  
  res.status(204).end();
});

/**
 * POST /api/accounts/:id/transactions - Create transaction
 */
app.post(
  '/api/accounts/:id/transactions',
  authenticate,
  validateBody({
    type: { required: true, type: 'string' },
    amount: { required: true, type: 'number', min: 0.01 },
    description: { type: 'string' }
  }),
  (req, res) => {
    const { id } = req.params;
    const { type, amount, description = '' } = req.body;
    
    const account = accounts.get(id);
    
    if (!account) {
      return res.status(404).json({
        error: {
          message: 'Account not found',
          code: 'ACCOUNT_NOT_FOUND'
        }
      });
    }
    
    // Validate transaction type
    const validTypes = ['DEPOSIT', 'WITHDRAWAL', 'TRANSFER'];
    if (!validTypes.includes(type)) {
      return res.status(400).json({
        error: {
          message: `Invalid transaction type. Must be one of: ${validTypes.join(', ')}`,
          code: 'INVALID_TRANSACTION_TYPE'
        }
      });
    }
    
    // Check sufficient balance for withdrawals
    if (type === 'WITHDRAWAL' && account.balance < amount) {
      return res.status(400).json({
        error: {
          message: 'Insufficient balance',
          code: 'INSUFFICIENT_BALANCE',
          currentBalance: account.balance,
          requestedAmount: amount
        }
      });
    }
    
    // Create transaction
    const transaction = {
      id: `TXN-${crypto.randomBytes(4).toString('hex').toUpperCase()}`,
      accountId: id,
      type,
      amount,
      description,
      status: 'COMPLETED',
      createdAt: new Date().toISOString()
    };
    
    // Update account balance
    if (type === 'DEPOSIT') {
      account.balance += amount;
    } else if (type === 'WITHDRAWAL') {
      account.balance -= amount;
    }
    
    accounts.set(id, account);
    transactions.set(transaction.id, transaction);
    
    res.status(201).json({
      message: 'Transaction created successfully',
      data: transaction,
      updatedBalance: account.balance,
      metadata: {
        timestamp: new Date().toISOString(),
        requestId: req.requestId
      }
    });
  }
);

/**
 * GET /api/accounts/:id/balance - Get account balance
 */
app.get('/api/accounts/:id/balance', authenticate, (req, res) => {
  const { id } = req.params;
  
  const account = accounts.get(id);
  
  if (!account) {
    return res.status(404).json({
      error: {
        message: 'Account not found',
        code: 'ACCOUNT_NOT_FOUND'
      }
    });
  }
  
  res.json({
    data: {
      accountId: account.id,
      balance: account.balance,
      currency: account.currency,
      status: account.status,
      asOfDate: new Date().toISOString()
    },
    metadata: {
      timestamp: new Date().toISOString(),
      requestId: req.requestId
    }
  });
});

// ============================================
// Error Handling Middleware
// ============================================

/**
 * 404 Not Found Handler
 */
app.use((req, res) => {
  res.status(404).json({
    error: {
      message: 'Route not found',
      code: 'NOT_FOUND',
      method: req.method,
      path: req.path
    }
  });
});

/**
 * Global Error Handler
 */
app.use((err, req, res, next) => {
  console.error(`[${req.requestId}] Error:`, err);
  
  // Handle specific error types
  if (err.name === 'SyntaxError' && err.status === 400 && 'body' in err) {
    return res.status(400).json({
      error: {
        message: 'Invalid JSON',
        code: 'INVALID_JSON'
      }
    });
  }
  
  // Generic error response
  res.status(err.status || 500).json({
    error: {
      message: err.message || 'Internal server error',
      code: err.code || 'INTERNAL_ERROR',
      ...(process.env.NODE_ENV === 'development' && { stack: err.stack })
    }
  });
});

// ============================================
// Start Server
// ============================================

app.listen(PORT, () => {
  console.log(`\n✅ Banking API running on http://localhost:${PORT}`);
  console.log('\nAvailable endpoints:');
  console.log('  GET    /health                              - Health check');
  console.log('  GET    /api/accounts                        - List accounts');
  console.log('  POST   /api/accounts                        - Create account');
  console.log('  GET    /api/accounts/:id                    - Get account');
  console.log('  PUT    /api/accounts/:id                    - Replace account');
  console.log('  PATCH  /api/accounts/:id                    - Update account');
  console.log('  DELETE /api/accounts/:id                    - Delete account');
  console.log('  POST   /api/accounts/:id/transactions       - Create transaction');
  console.log('  GET    /api/accounts/:id/balance            - Get balance');
  console.log('\nAuthentication: Use header: Authorization: Bearer valid-token-12345');
  console.log('\n' + '='.repeat(70) + '\n');
});

// Graceful shutdown
process.on('SIGTERM', () => {
  console.log('\n🛑 SIGTERM received, shutting down gracefully...');
  server.close(() => {
    console.log('✅ Server closed');
    process.exit(0);
  });
});
```

**To run this example:**

```bash
# Install dependencies
npm install express

# Run the server
node banking-api-express.js
```

**Testing with curl:**

```bash
# 1. Health check (no auth required)
curl http://localhost:3000/health

# 2. List accounts (requires auth)
curl http://localhost:3000/api/accounts \
  -H "Authorization: Bearer valid-token-12345"

# 3. List accounts with filters
curl "http://localhost:3000/api/accounts?status=ACTIVE&page=1&limit=5" \
  -H "Authorization: Bearer valid-token-12345"

# 4. Get specific account
curl http://localhost:3000/api/accounts/ACC-001 \
  -H "Authorization: Bearer valid-token-12345"

# 5. Create new account
curl -X POST http://localhost:3000/api/accounts \
  -H "Authorization: Bearer valid-token-12345" \
  -H "Content-Type: application/json" \
  -d '{
    "accountNumber": "5555555555",
    "customerName": "Alice Johnson",
    "accountType": "SAVINGS",
    "balance": 1000
  }'

# 6. Update account (PATCH)
curl -X PATCH http://localhost:3000/api/accounts/ACC-001 \
  -H "Authorization: Bearer valid-token-12345" \
  -H "Content-Type: application/json" \
  -d '{"balance": 6000}'

# 7. Create transaction
curl -X POST http://localhost:3000/api/accounts/ACC-001/transactions \
  -H "Authorization: Bearer valid-token-12345" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "DEPOSIT",
    "amount": 500,
    "description": "Salary deposit"
  }'

# 8. Get account balance
curl http://localhost:3000/api/accounts/ACC-001/balance \
  -H "Authorization: Bearer valid-token-12345"

# 9. Try to delete account with balance (should fail)
curl -X DELETE http://localhost:3000/api/accounts/ACC-001 \
  -H "Authorization: Bearer valid-token-12345"

# 10. Test authentication failure
curl http://localhost:3000/api/accounts

# 11. Test rate limiting (send 101 requests quickly)
for i in {1..101}; do
  curl http://localhost:3000/health &
done
wait

# 12. Test validation error
curl -X POST http://localhost:3000/api/accounts \
  -H "Authorization: Bearer valid-token-12345" \
  -H "Content-Type: application/json" \
  -d '{"accountNumber": "123"}'

# 13. Test 404 error
curl http://localhost:3000/api/invalid-route \
  -H "Authorization: Bearer valid-token-12345"
```

**Expected Output (Server):**

```
🏦 Banking API with Express.js

======================================================================

✅ Banking API running on http://localhost:3000

Available endpoints:
  GET    /health                              - Health check
  GET    /api/accounts                        - List accounts
  POST   /api/accounts                        - Create account
  GET    /api/accounts/:id                    - Get account
  PUT    /api/accounts/:id                    - Replace account
  PATCH  /api/accounts/:id                    - Update account
  DELETE /api/accounts/:id                    - Delete account
  POST   /api/accounts/:id/transactions       - Create transaction
  GET    /api/accounts/:id/balance            - Get balance

Authentication: Use header: Authorization: Bearer valid-token-12345

======================================================================

[abc-123-def] GET /api/accounts
   IP: ::1
   User-Agent: curl/7.64.1
[abc-123-def] Response: 200 (5ms)

[xyz-456-ghi] POST /api/accounts
   IP: ::1
   User-Agent: curl/7.64.1
[xyz-456-ghi] Response: 201 (3ms)

[pqr-789-stu] POST /api/accounts/ACC-001/transactions
   IP: ::1
   User-Agent: curl/7.64.1
[pqr-789-stu] Response: 201 (4ms)
```

**Key Learnings from Example 1**:

1. **Express Setup**: Simple initialization with express()
2. **Built-in Middleware**: JSON parsing, URL-encoded parsing
3. **Custom Middleware**: Logger, CORS, authentication, rate limiting
4. **Validation Middleware**: Reusable schema-based validation
5. **Route Handlers**: Clean, declarative routing
6. **Request/Response**: Enhanced req/res objects
7. **Error Handling**: Centralized error middleware
8. **Status Codes**: Proper HTTP status codes (200, 201, 400, 401, 404, 409, 429)
9. **Response Format**: Consistent JSON response structure
10. **Security**: Authentication, rate limiting, input validation

---

## 🏦 Example 2: Express Router for Modular Banking API

This example demonstrates using Express Router to organize routes into separate modules for better code organization.

**File Structure:**
```
banking-api/
├── app.js                 # Main application
├── routes/
│   ├── accounts.js        # Account routes
│   ├── transactions.js    # Transaction routes
│   └── auth.js            # Authentication routes
└── middleware/
    ├── auth.js            # Authentication middleware
    └── validation.js      # Validation middleware
```

**File: `middleware/auth.js`**

```javascript
/**
 * Authentication Middleware
 */

export function authenticate(req, res, next) {
  const token = req.headers.authorization;
  
  if (!token) {
    return res.status(401).json({
      error: {
        message: 'Authentication required',
        code: 'UNAUTHORIZED'
      }
    });
  }
  
  // Verify token (simplified - use JWT in production)
  if (!token.startsWith('Bearer ')) {
    return res.status(401).json({
      error: {
        message: 'Invalid token format',
        code: 'INVALID_TOKEN_FORMAT'
      }
    });
  }
  
  const tokenValue = token.substring(7);
  
  if (tokenValue !== 'valid-token-12345') {
    return res.status(401).json({
      error: {
        message: 'Invalid token',
        code: 'INVALID_TOKEN'
      }
    });
  }
  
  // Attach user to request
  req.user = {
    id: 'USER-001',
    email: 'user@example.com',
    role: 'CUSTOMER'
  };
  
  next();
}

export function requireRole(role) {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({
        error: {
          message: 'Authentication required',
          code: 'UNAUTHORIZED'
        }
      });
    }
    
    if (req.user.role !== role) {
      return res.status(403).json({
        error: {
          message: 'Insufficient permissions',
          code: 'FORBIDDEN',
          requiredRole: role,
          userRole: req.user.role
        }
      });
    }
    
    next();
  };
}
```

**File: `middleware/validation.js`**

```javascript
/**
 * Validation Middleware
 */

export function validateBody(schema) {
  return (req, res, next) => {
    const errors = [];
    
    for (const [field, rules] of Object.entries(schema)) {
      const value = req.body[field];
      
      if (rules.required && (value === undefined || value === null || value === '')) {
        errors.push(`${field} is required`);
        continue;
      }
      
      if (value !== undefined && value !== null && value !== '') {
        if (rules.type && typeof value !== rules.type) {
          errors.push(`${field} must be ${rules.type}`);
        }
        
        if (rules.min !== undefined && value < rules.min) {
          errors.push(`${field} must be at least ${rules.min}`);
        }
        
        if (rules.max !== undefined && value > rules.max) {
          errors.push(`${field} must be at most ${rules.max}`);
        }
        
        if (rules.pattern && !rules.pattern.test(value)) {
          errors.push(`${field} has invalid format`);
        }
        
        if (rules.enum && !rules.enum.includes(value)) {
          errors.push(`${field} must be one of: ${rules.enum.join(', ')}`);
        }
      }
    }
    
    if (errors.length > 0) {
      return res.status(400).json({
        error: {
          message: 'Validation failed',
          code: 'VALIDATION_ERROR',
          details: errors
        }
      });
    }
    
    next();
  };
}

export function validateParams(schema) {
  return (req, res, next) => {
    const errors = [];
    
    for (const [field, rules] of Object.entries(schema)) {
      const value = req.params[field];
      
      if (rules.required && !value) {
        errors.push(`${field} is required`);
        continue;
      }
      
      if (value && rules.pattern && !rules.pattern.test(value)) {
        errors.push(`${field} has invalid format`);
      }
    }
    
    if (errors.length > 0) {
      return res.status(400).json({
        error: {
          message: 'Invalid parameters',
          code: 'INVALID_PARAMS',
          details: errors
        }
      });
    }
    
    next();
  };
}
```

**File: `routes/accounts.js`**

```javascript
/**
 * Account Routes
 */

import express from 'express';
import crypto from 'crypto';
import { authenticate } from '../middleware/auth.js';
import { validateBody, validateParams } from '../middleware/validation.js';

const router = express.Router();

// In-memory database (shared with main app)
const accounts = new Map();

// Seed data
accounts.set('ACC-001', {
  id: 'ACC-001',
  accountNumber: '1234567890',
  customerName: 'John Doe',
  accountType: 'SAVINGS',
  balance: 5000,
  currency: 'USD',
  status: 'ACTIVE',
  createdAt: new Date('2025-01-01').toISOString()
});

/**
 * GET /accounts - List all accounts
 */
router.get('/', authenticate, (req, res) => {
  const { page = 1, limit = 10, status, type } = req.query;
  
  let filteredAccounts = Array.from(accounts.values());
  
  if (status) {
    filteredAccounts = filteredAccounts.filter(acc => acc.status === status);
  }
  
  if (type) {
    filteredAccounts = filteredAccounts.filter(acc => acc.accountType === type);
  }
  
  const startIndex = (parseInt(page) - 1) * parseInt(limit);
  const endIndex = startIndex + parseInt(limit);
  const paginatedAccounts = filteredAccounts.slice(startIndex, endIndex);
  
  res.json({
    data: paginatedAccounts,
    pagination: {
      page: parseInt(page),
      limit: parseInt(limit),
      total: filteredAccounts.length,
      totalPages: Math.ceil(filteredAccounts.length / parseInt(limit))
    }
  });
});

/**
 * GET /accounts/:id - Get specific account
 */
router.get(
  '/:id',
  authenticate,
  validateParams({
    id: { required: true, pattern: /^ACC-[A-Z0-9]+$/ }
  }),
  (req, res) => {
    const { id } = req.params;
    
    const account = accounts.get(id);
    
    if (!account) {
      return res.status(404).json({
        error: {
          message: 'Account not found',
          code: 'ACCOUNT_NOT_FOUND',
          accountId: id
        }
      });
    }
    
    res.json({ data: account });
  }
);

/**
 * POST /accounts - Create new account
 */
router.post(
  '/',
  authenticate,
  validateBody({
    accountNumber: { required: true, type: 'string', pattern: /^\d{10}$/ },
    customerName: { required: true, type: 'string' },
    accountType: { required: true, type: 'string', enum: ['SAVINGS', 'CHECKING', 'BUSINESS'] },
    balance: { type: 'number', min: 0 },
    currency: { type: 'string' }
  }),
  (req, res) => {
    const { accountNumber, customerName, accountType, balance = 0, currency = 'USD' } = req.body;
    
    // Check for duplicate
    const existingAccount = Array.from(accounts.values())
      .find(acc => acc.accountNumber === accountNumber);
    
    if (existingAccount) {
      return res.status(409).json({
        error: {
          message: 'Account number already exists',
          code: 'DUPLICATE_ACCOUNT',
          accountNumber
        }
      });
    }
    
    const account = {
      id: `ACC-${crypto.randomBytes(4).toString('hex').toUpperCase()}`,
      accountNumber,
      customerName,
      accountType,
      balance,
      currency,
      status: 'ACTIVE',
      createdAt: new Date().toISOString()
    };
    
    accounts.set(account.id, account);
    
    res.status(201).json({
      message: 'Account created successfully',
      data: account
    });
  }
);

/**
 * PATCH /accounts/:id - Update account
 */
router.patch(
  '/:id',
  authenticate,
  validateParams({
    id: { required: true, pattern: /^ACC-[A-Z0-9]+$/ }
  }),
  (req, res) => {
    const { id } = req.params;
    
    const existingAccount = accounts.get(id);
    
    if (!existingAccount) {
      return res.status(404).json({
        error: {
          message: 'Account not found',
          code: 'ACCOUNT_NOT_FOUND'
        }
      });
    }
    
    const account = {
      ...existingAccount,
      ...req.body,
      id,
      createdAt: existingAccount.createdAt,
      updatedAt: new Date().toISOString()
    };
    
    accounts.set(id, account);
    
    res.json({
      message: 'Account updated successfully',
      data: account
    });
  }
);

/**
 * DELETE /accounts/:id - Delete account
 */
router.delete(
  '/:id',
  authenticate,
  validateParams({
    id: { required: true, pattern: /^ACC-[A-Z0-9]+$/ }
  }),
  (req, res) => {
    const { id } = req.params;
    
    const account = accounts.get(id);
    
    if (!account) {
      return res.status(404).json({
        error: {
          message: 'Account not found',
          code: 'ACCOUNT_NOT_FOUND'
        }
      });
    }
    
    if (account.balance > 0) {
      return res.status(400).json({
        error: {
          message: 'Cannot delete account with non-zero balance',
          code: 'ACCOUNT_HAS_BALANCE',
          balance: account.balance
        }
      });
    }
    
    accounts.delete(id);
    
    res.status(204).end();
  }
);

export default router;
export { accounts };
```

**File: `routes/transactions.js`**

```javascript
/**
 * Transaction Routes
 */

import express from 'express';
import crypto from 'crypto';
import { authenticate } from '../middleware/auth.js';
import { validateBody, validateParams } from '../middleware/validation.js';
import { accounts } from './accounts.js';

const router = express.Router();

const transactions = new Map();

/**
 * POST /accounts/:accountId/transactions - Create transaction
 */
router.post(
  '/accounts/:accountId/transactions',
  authenticate,
  validateParams({
    accountId: { required: true, pattern: /^ACC-[A-Z0-9]+$/ }
  }),
  validateBody({
    type: { required: true, type: 'string', enum: ['DEPOSIT', 'WITHDRAWAL', 'TRANSFER'] },
    amount: { required: true, type: 'number', min: 0.01 },
    description: { type: 'string' }
  }),
  (req, res) => {
    const { accountId } = req.params;
    const { type, amount, description = '' } = req.body;
    
    const account = accounts.get(accountId);
    
    if (!account) {
      return res.status(404).json({
        error: {
          message: 'Account not found',
          code: 'ACCOUNT_NOT_FOUND'
        }
      });
    }
    
    // Check sufficient balance for withdrawals
    if (type === 'WITHDRAWAL' && account.balance < amount) {
      return res.status(400).json({
        error: {
          message: 'Insufficient balance',
          code: 'INSUFFICIENT_BALANCE',
          currentBalance: account.balance,
          requestedAmount: amount
        }
      });
    }
    
    // Create transaction
    const transaction = {
      id: `TXN-${crypto.randomBytes(4).toString('hex').toUpperCase()}`,
      accountId,
      type,
      amount,
      description,
      status: 'COMPLETED',
      createdAt: new Date().toISOString()
    };
    
    // Update balance
    if (type === 'DEPOSIT') {
      account.balance += amount;
    } else if (type === 'WITHDRAWAL') {
      account.balance -= amount;
    }
    
    accounts.set(accountId, account);
    transactions.set(transaction.id, transaction);
    
    res.status(201).json({
      message: 'Transaction created successfully',
      data: transaction,
      updatedBalance: account.balance
    });
  }
);

/**
 * GET /accounts/:accountId/transactions - List account transactions
 */
router.get(
  '/accounts/:accountId/transactions',
  authenticate,
  validateParams({
    accountId: { required: true, pattern: /^ACC-[A-Z0-9]+$/ }
  }),
  (req, res) => {
    const { accountId } = req.params;
    const { page = 1, limit = 10, type } = req.query;
    
    const account = accounts.get(accountId);
    
    if (!account) {
      return res.status(404).json({
        error: {
          message: 'Account not found',
          code: 'ACCOUNT_NOT_FOUND'
        }
      });
    }
    
    let accountTransactions = Array.from(transactions.values())
      .filter(txn => txn.accountId === accountId);
    
    if (type) {
      accountTransactions = accountTransactions.filter(txn => txn.type === type);
    }
    
    const startIndex = (parseInt(page) - 1) * parseInt(limit);
    const endIndex = startIndex + parseInt(limit);
    const paginatedTransactions = accountTransactions.slice(startIndex, endIndex);
    
    res.json({
      data: paginatedTransactions,
      pagination: {
        page: parseInt(page),
        limit: parseInt(limit),
        total: accountTransactions.length,
        totalPages: Math.ceil(accountTransactions.length / parseInt(limit))
      }
    });
  }
);

/**
 * GET /transactions/:id - Get transaction details
 */
router.get(
  '/transactions/:id',
  authenticate,
  validateParams({
    id: { required: true, pattern: /^TXN-[A-Z0-9]+$/ }
  }),
  (req, res) => {
    const { id } = req.params;
    
    const transaction = transactions.get(id);
    
    if (!transaction) {
      return res.status(404).json({
        error: {
          message: 'Transaction not found',
          code: 'TRANSACTION_NOT_FOUND',
          transactionId: id
        }
      });
    }
    
    res.json({ data: transaction });
  }
);

export default router;
```

**File: `routes/auth.js`**

```javascript
/**
 * Authentication Routes
 */

import express from 'express';
import crypto from 'crypto';
import { validateBody } from '../middleware/validation.js';

const router = express.Router();

// In-memory user store (use database in production)
const users = new Map([
  ['user@example.com', {
    id: 'USER-001',
    email: 'user@example.com',
    password: 'hashed_password_123',  // Use bcrypt in production
    role: 'CUSTOMER'
  }]
]);

const sessions = new Map();

/**
 * POST /auth/login - User login
 */
router.post(
  '/login',
  validateBody({
    email: { required: true, type: 'string', pattern: /^[^\s@]+@[^\s@]+\.[^\s@]+$/ },
    password: { required: true, type: 'string' }
  }),
  (req, res) => {
    const { email, password } = req.body;
    
    const user = users.get(email);
    
    if (!user || user.password !== `hashed_${password}`) {
      return res.status(401).json({
        error: {
          message: 'Invalid credentials',
          code: 'INVALID_CREDENTIALS'
        }
      });
    }
    
    // Generate session token
    const token = `Bearer valid-token-${crypto.randomBytes(8).toString('hex')}`;
    
    sessions.set(token, {
      userId: user.id,
      email: user.email,
      role: user.role,
      createdAt: new Date().toISOString()
    });
    
    res.json({
      message: 'Login successful',
      data: {
        token,
        user: {
          id: user.id,
          email: user.email,
          role: user.role
        }
      }
    });
  }
);

/**
 * POST /auth/logout - User logout
 */
router.post('/logout', (req, res) => {
  const token = req.headers.authorization;
  
  if (token && sessions.has(token)) {
    sessions.delete(token);
  }
  
  res.json({
    message: 'Logout successful'
  });
});

/**
 * GET /auth/me - Get current user
 */
router.get('/me', (req, res) => {
  const token = req.headers.authorization;
  
  if (!token) {
    return res.status(401).json({
      error: {
        message: 'Authentication required',
        code: 'UNAUTHORIZED'
      }
    });
  }
  
  const session = sessions.get(token);
  
  if (!session) {
    return res.status(401).json({
      error: {
        message: 'Invalid or expired token',
        code: 'INVALID_TOKEN'
      }
    });
  }
  
  const user = users.get(session.email);
  
  res.json({
    data: {
      id: user.id,
      email: user.email,
      role: user.role
    }
  });
});

export default router;
```

**File: `app.js`** (Main Application)

```javascript
/**
 * Modular Banking API with Express Router
 * Demonstrates: Router, modular structure, organized routes
 */

import express from 'express';
import accountRoutes from './routes/accounts.js';
import transactionRoutes from './routes/transactions.js';
import authRoutes from './routes/auth.js';

console.log('🏦 Modular Banking API with Express Router\n');
console.log('='.repeat(70));

const app = express();
const PORT = 3001;

// ============================================
// Global Middleware
// ============================================

// Body parsing
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Request logger
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
});

// CORS
app.use((req, res, next) => {
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, PATCH, DELETE, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
  
  if (req.method === 'OPTIONS') {
    return res.status(204).end();
  }
  
  next();
});

// ============================================
// Mount Routers
// ============================================

// Health check
app.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    timestamp: new Date().toISOString()
  });
});

// API routes
app.use('/api/accounts', accountRoutes);
app.use('/api', transactionRoutes);
app.use('/api/auth', authRoutes);

// ============================================
// Error Handling
// ============================================

// 404 handler
app.use((req, res) => {
  res.status(404).json({
    error: {
      message: 'Route not found',
      code: 'NOT_FOUND',
      method: req.method,
      path: req.path
    }
  });
});

// Global error handler
app.use((err, req, res, next) => {
  console.error('Error:', err);
  
  res.status(err.status || 500).json({
    error: {
      message: err.message || 'Internal server error',
      code: err.code || 'INTERNAL_ERROR'
    }
  });
});

// ============================================
// Start Server
// ============================================

app.listen(PORT, () => {
  console.log(`\n✅ Banking API running on http://localhost:${PORT}`);
  console.log('\nAvailable endpoints:');
  console.log('  Authentication:');
  console.log('    POST /api/auth/login          - Login');
  console.log('    POST /api/auth/logout         - Logout');
  console.log('    GET  /api/auth/me             - Get current user');
  console.log('\n  Accounts:');
  console.log('    GET    /api/accounts          - List accounts');
  console.log('    POST   /api/accounts          - Create account');
  console.log('    GET    /api/accounts/:id      - Get account');
  console.log('    PATCH  /api/accounts/:id      - Update account');
  console.log('    DELETE /api/accounts/:id      - Delete account');
  console.log('\n  Transactions:');
  console.log('    POST /api/accounts/:id/transactions  - Create transaction');
  console.log('    GET  /api/accounts/:id/transactions  - List transactions');
  console.log('    GET  /api/transactions/:id           - Get transaction');
  console.log('\n' + '='.repeat(70) + '\n');
});
```

**Testing:**

```bash
# 1. Login
curl -X POST http://localhost:3001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "password_123"
  }'

# Response: {"token": "Bearer valid-token-xxxxx"}

# 2. Get current user
curl http://localhost:3001/api/auth/me \
  -H "Authorization: Bearer valid-token-xxxxx"

# 3. List accounts
curl http://localhost:3001/api/accounts \
  -H "Authorization: Bearer valid-token-12345"

# 4. Create transaction
curl -X POST http://localhost:3001/api/accounts/ACC-001/transactions \
  -H "Authorization: Bearer valid-token-12345" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "DEPOSIT",
    "amount": 500,
    "description": "Salary"
  }'

# 5. Logout
curl -X POST http://localhost:3001/api/auth/logout \
  -H "Authorization: Bearer valid-token-xxxxx"
```

**Key Learnings from Example 2**:

1. **Express Router**: Modular route organization
2. **Code Organization**: Separate files for routes and middleware
3. **Middleware Sharing**: Reusable middleware across routes
4. **Route Mounting**: Mount routers at specific paths
5. **Parameter Validation**: Validate route parameters
6. **Role-Based Access**: Middleware for authorization
7. **Clean Structure**: Easy to maintain and scale
8. **Separation of Concerns**: Routes, middleware, and logic separated
9. **Testability**: Easier to test individual route modules
10. **Scalability**: Add new routes without cluttering main file

---

## 💡 DO's - Best Practices

### 1. ✅ DO: Use Middleware for Cross-Cutting Concerns

```javascript
// ✅ GOOD: Middleware for common functionality
app.use(logger);
app.use(authenticate);
app.use(rateLimit);

app.get('/accounts', (req, res) => {
  // Clean handler - authentication already done
});

// ❌ BAD: Duplicate logic in every route
app.get('/accounts', (req, res) => {
  log(req);
  if (!req.headers.authorization) return res.status(401).end();
  if (exceedsRateLimit()) return res.status(429).end();
  // Now handle request...
});
```

**Why**: Middleware keeps route handlers clean and DRY (Don't Repeat Yourself).

---

### 2. ✅ DO: Use Express Router for Organization

```javascript
// ✅ GOOD: Organized with routers
// routes/accounts.js
const router = express.Router();
router.get('/', listAccounts);
router.post('/', createAccount);
export default router;

// app.js
app.use('/api/accounts', accountRouter);

// ❌ BAD: Everything in one file
app.get('/api/accounts', listAccounts);
app.post('/api/accounts', createAccount);
app.get('/api/transactions', listTransactions);
app.post('/api/transactions', createTransaction);
// 100+ more routes...
```

**Why**: Routers enable modular, maintainable code structure.

---

### 3. ✅ DO: Use res.json() for JSON Responses

```javascript
// ✅ GOOD: Use res.json()
app.get('/accounts', (req, res) => {
  res.json({ data: accounts });
});

// ❌ BAD: Manual JSON stringification
app.get('/accounts', (req, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({ data: accounts }));
});
```

**Why**: res.json() automatically sets Content-Type and handles edge cases.

---

### 4. ✅ DO: Chain Status Code with Response

```javascript
// ✅ GOOD: Chained status and response
app.post('/accounts', (req, res) => {
  res.status(201).json({ created: true });
});

app.get('/accounts/:id', (req, res) => {
  if (!account) {
    return res.status(404).json({ error: 'Not found' });
  }
});

// ❌ BAD: Separate calls
app.post('/accounts', (req, res) => {
  res.status(201);
  res.json({ created: true });
});
```

**Why**: Chaining is cleaner and prevents accidentally sending response twice.

---

### 5. ✅ DO: Use Route Parameters for Resource IDs

```javascript
// ✅ GOOD: Route parameters
app.get('/accounts/:id', (req, res) => {
  const { id } = req.params;
});

app.get('/accounts/:accountId/transactions/:transactionId', (req, res) => {
  const { accountId, transactionId } = req.params;
});

// ❌ BAD: Query parameters for IDs
app.get('/accounts', (req, res) => {
  const { id } = req.query;  // /accounts?id=123
});
```

**Why**: Route parameters are RESTful and clearly identify resources.

---

### 6. ✅ DO: Validate Input with Middleware

```javascript
// ✅ GOOD: Validation middleware
const validateAccount = validateBody({
  accountNumber: { required: true, pattern: /^\d{10}$/ },
  balance: { type: 'number', min: 0 }
});

app.post('/accounts', validateAccount, (req, res) => {
  // Input is already validated
});

// ❌ BAD: Manual validation in every route
app.post('/accounts', (req, res) => {
  if (!req.body.accountNumber) return res.status(400).json({});
  if (typeof req.body.balance !== 'number') return res.status(400).json({});
  // ... more validation
});
```

**Why**: Validation middleware is reusable and keeps handlers clean.

---

### 7. ✅ DO: Use Centralized Error Handling

```javascript
// ✅ GOOD: Centralized error handler
app.get('/accounts/:id', async (req, res, next) => {
  try {
    const account = await getAccount(req.params.id);
    res.json({ data: account });
  } catch (error) {
    next(error);  // Pass to error handler
  }
});

// Error handling middleware
app.use((err, req, res, next) => {
  res.status(err.status || 500).json({
    error: {
      message: err.message,
      code: err.code
    }
  });
});

// ❌ BAD: Scattered error handling
app.get('/accounts/:id', async (req, res) => {
  try {
    const account = await getAccount(req.params.id);
    res.json({ data: account });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

**Why**: Centralized error handling ensures consistent error responses.

---

### 8. ✅ DO: Set Appropriate HTTP Status Codes

```javascript
// ✅ GOOD: Proper status codes
app.post('/accounts', (req, res) => {
  res.status(201).json({ data: account });  // Created
});

app.get('/accounts/:id', (req, res) => {
  if (!account) {
    return res.status(404).json({ error: 'Not found' });  // Not Found
  }
  res.status(200).json({ data: account });  // OK
});

app.delete('/accounts/:id', (req, res) => {
  res.status(204).end();  // No Content
});

// ❌ BAD: Always 200
app.post('/accounts', (req, res) => {
  res.json({ data: account });  // Default 200, should be 201
});
```

**Why**: Proper status codes communicate success/failure type to clients.

---

### 9. ✅ DO: Use app.route() for Route Chaining

```javascript
// ✅ GOOD: Route chaining
app.route('/accounts/:id')
  .get(getAccount)
  .put(updateAccount)
  .delete(deleteAccount);

// ❌ BAD: Separate route definitions
app.get('/accounts/:id', getAccount);
app.put('/accounts/:id', updateAccount);
app.delete('/accounts/:id', deleteAccount);
```

**Why**: Route chaining reduces duplication and keeps related routes together.

---

### 10. ✅ DO: Use Async/Await with try-catch

```javascript
// ✅ GOOD: Async/await with error handling
app.get('/accounts/:id', async (req, res, next) => {
  try {
    const account = await getAccount(req.params.id);
    res.json({ data: account });
  } catch (error) {
    next(error);
  }
});

// ❌ BAD: Unhandled promises
app.get('/accounts/:id', (req, res) => {
  getAccount(req.params.id)
    .then(account => res.json({ data: account }));
  // Missing .catch() - unhandled rejection!
});
```

**Why**: Proper async error handling prevents unhandled promise rejections.

---

## 🚫 DON'Ts - Common Mistakes

### 1. ❌ DON'T: Forget to Call next() in Middleware

```javascript
// ❌ BAD: Middleware doesn't call next()
app.use((req, res, next) => {
  console.log('Log');
  // Forgot to call next() - request hangs!
});

// ✅ GOOD: Always call next()
app.use((req, res, next) => {
  console.log('Log');
  next();  // Pass control to next middleware
});
```

**Why**: Without next(), the request never reaches route handlers.

---

### 2. ❌ DON'T: Send Response Multiple Times

```javascript
// ❌ BAD: Multiple responses
app.get('/accounts/:id', (req, res) => {
  if (!account) {
    res.status(404).json({ error: 'Not found' });
    // Code continues! Second response attempted
  }
  res.json({ data: account });  // Error: Can't set headers after sent
});

// ✅ GOOD: Return after response
app.get('/accounts/:id', (req, res) => {
  if (!account) {
    return res.status(404).json({ error: 'Not found' });  // Return!
  }
  res.json({ data: account });
});
```

**Why**: Sending response twice causes "Can't set headers after they are sent" error.

---

### 3. ❌ DON'T: Put Error Handler Before Routes

```javascript
// ❌ BAD: Error handler before routes
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message });
});

app.get('/accounts', (req, res) => {
  // This route never executes!
});

// ✅ GOOD: Error handler after routes
app.get('/accounts', (req, res) => {
  // Route executes
});

// Error handler LAST
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message });
});
```

**Why**: Middleware executes in order. Error handler must be last to catch errors from routes.

---

### 4. ❌ DON'T: Use app.all() Unnecessarily

```javascript
// ❌ BAD: Handle all methods the same
app.all('/accounts', (req, res) => {
  // Handles GET, POST, PUT, DELETE the same way!
  createAccount();  // Wrong for GET!
});

// ✅ GOOD: Use specific methods
app.get('/accounts', listAccounts);
app.post('/accounts', createAccount);
app.put('/accounts/:id', updateAccount);
```

**Why**: Different HTTP methods should have different behavior.

---

### 5. ❌ DON'T: Hardcode Port Numbers

```javascript
// ❌ BAD: Hardcoded port
app.listen(3000, () => {
  console.log('Server on port 3000');
});

// ✅ GOOD: Use environment variable
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Server on port ${PORT}`);
});
```

**Why**: Environment variables allow configuration without code changes.

---

### 6. ❌ DON'T: Trust User Input

```javascript
// ❌ BAD: No validation
app.post('/accounts', (req, res) => {
  const account = createAccount(req.body);  // Direct use!
  res.json({ data: account });
});

// ✅ GOOD: Validate everything
app.post('/accounts', validateInput, (req, res) => {
  const { accountNumber, customerName } = req.body;
  // Only use validated data
});
```

**Why**: Unvalidated input can cause security vulnerabilities and crashes.

---

### 7. ❌ DON'T: Expose Stack Traces in Production

```javascript
// ❌ BAD: Expose internals
app.use((err, req, res, next) => {
  res.status(500).json({
    error: err.message,
    stack: err.stack  // Exposes internal paths!
  });
});

// ✅ GOOD: Generic error in production
app.use((err, req, res, next) => {
  console.error(err);  // Log internally
  
  res.status(500).json({
    error: 'Internal server error',
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack })
  });
});
```

**Why**: Stack traces reveal internal structure and security vulnerabilities.

---

### 8. ❌ DON'T: Ignore Query Parameter Types

```javascript
// ❌ BAD: Assume query params are numbers
app.get('/accounts', (req, res) => {
  const page = req.query.page;  // String "1", not number!
  const results = accounts.slice(page * 10);  // Wrong calculation!
});

// ✅ GOOD: Parse and validate
app.get('/accounts', (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 10;
  
  if (page < 1 || limit < 1 || limit > 100) {
    return res.status(400).json({ error: 'Invalid pagination' });
  }
});
```

**Why**: Query parameters are always strings and need parsing/validation.

---

### 9. ❌ DON'T: Use Synchronous Operations in Routes

```javascript
// ❌ BAD: Blocks event loop
app.get('/accounts', (req, res) => {
  const data = fs.readFileSync('accounts.json');  // BLOCKS!
  res.json(JSON.parse(data));
});

// ✅ GOOD: Use async operations
app.get('/accounts', async (req, res, next) => {
  try {
    const data = await fs.promises.readFile('accounts.json', 'utf8');
    res.json(JSON.parse(data));
  } catch (error) {
    next(error);
  }
});
```

**Why**: Synchronous operations block the entire server.

---

### 10. ❌ DON'T: Nest Routers Too Deeply

```javascript
// ❌ BAD: Too many nested routers
const router1 = express.Router();
const router2 = express.Router();
const router3 = express.Router();

router3.get('/detail', handler);
router2.use('/transactions', router3);
router1.use('/accounts', router2);
app.use('/api', router1);

// Final URL: /api/accounts/transactions/detail (confusing!)

// ✅ GOOD: Flat, clear structure
const accountRouter = express.Router();
accountRouter.get('/', listAccounts);
accountRouter.get('/:id/transactions', listTransactions);

app.use('/api/accounts', accountRouter);
```

**Why**: Deep nesting makes URLs and code hard to understand.

---

## 🎯 Key Takeaways

### Express vs Raw HTTP

| Feature | Raw HTTP | Express |
|---------|----------|---------|
| **Routing** | Manual pattern matching | Declarative routes |
| **Body Parsing** | Manual stream handling | Built-in middleware |
| **Middleware** | Not supported | Full middleware system |
| **Request/Response** | Basic objects | Enhanced with helpers |
| **Error Handling** | Scattered try-catch | Centralized middleware |
| **Code Organization** | Single file | Modular routers |
| **Development Speed** | Slow | Fast |
| **Maintainability** | Difficult | Easy |

### Middleware Execution Order

```javascript
1. Global middleware (app.use)
2. Route-specific middleware
3. Route handler
4. Error handling middleware (if error occurs)
```

### Express Request Object

| Property | Description | Example |
|----------|-------------|---------|
| `req.params` | Route parameters | `/users/:id` → `req.params.id` |
| `req.query` | Query string | `/users?age=25` → `req.query.age` |
| `req.body` | Request body | `req.body.email` |
| `req.headers` | HTTP headers | `req.headers.authorization` |
| `req.method` | HTTP method | `GET`, `POST`, etc. |
| `req.path` | URL path | `/api/users` |
| `req.url` | Full URL | `/api/users?page=1` |
| `req.ip` | Client IP | `192.168.1.1` |

### Express Response Methods

| Method | Purpose | Example |
|--------|---------|---------|
| `res.json()` | Send JSON | `res.json({ data })` |
| `res.send()` | Send response | `res.send('Hello')` |
| `res.status()` | Set status code | `res.status(404)` |
| `res.redirect()` | Redirect | `res.redirect('/login')` |
| `res.sendFile()` | Send file | `res.sendFile('index.html')` |
| `res.download()` | Download file | `res.download('report.pdf')` |
| `res.set()` | Set header | `res.set('X-Custom', 'value')` |
| `res.cookie()` | Set cookie | `res.cookie('sessionId', '123')` |
| `res.end()` | End response | `res.status(204).end()` |

---

## 🎓 Summary

**Express.js** simplifies Node.js web development with clean routing, powerful middleware, and enhanced request/response objects:

1. **Simplified Routing**: Declarative routes vs manual pattern matching
2. **Middleware System**: Reusable functions for cross-cutting concerns
3. **Body Parsing**: Automatic JSON and URL-encoded parsing
4. **Router Organization**: Modular route structure for scalability
5. **Enhanced Objects**: Convenient methods on req/res
6. **Error Handling**: Centralized error middleware
7. **Development Speed**: Build APIs faster with less boilerplate
8. **Maintainability**: Clean, organized code structure
9. **Community**: Large ecosystem of middleware packages
10. **Production-Ready**: Battle-tested in millions of applications

**Remember**: In banking, Express enables rapid development of secure, maintainable APIs while providing the flexibility needed for complex financial operations. Always validate inputs, handle errors properly, and use middleware for authentication and authorization.

---

## 📚 Additional Resources

- [Express.js Official Documentation](https://expressjs.com/)
- [Express.js Guide](https://expressjs.com/en/guide/routing.html)
- [Express Middleware Guide](https://expressjs.com/en/guide/using-middleware.html)
- [Express Best Practices](https://expressjs.com/en/advanced/best-practice-performance.html)
- [Awesome Express](https://github.com/rajikaimal/awesome-express) - Curated list of Express resources

---

**End of Question 15: Express.js Basics** 🎉

**Total Content**: ~2,300+ lines of comprehensive Express.js content with production-ready banking examples!

