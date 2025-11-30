# Node.js Interview Question: HTTP Request Methods
## Question 14: How to Handle Different HTTP Methods and Parse Request Bodies?

---

## 📋 Summary of What Will Be Covered

In this comprehensive guide, you'll learn:

1. **HTTP Request Methods** - GET, POST, PUT, DELETE, PATCH and their use cases
2. **Request Body Parsing** - Reading and parsing JSON, form data, multipart
3. **URL Parameters** - Query strings and route parameters
4. **Request Headers** - Reading and validating headers
5. **Content Negotiation** - Handling different content types
6. **RESTful API Design** - Best practices for banking operations
7. **Banking Example**: Complete RESTful account management API with CRUD operations

**Why This Matters in Banking**:
- Different operations require different HTTP methods (GET for read, POST for create)
- Request body validation is critical for transaction security
- RESTful APIs provide standardized interfaces for client applications
- Proper HTTP status codes communicate operation results
- Content-Type handling ensures data integrity
- Idempotency is crucial for financial operations

---

## 🎯 HTTP Request Methods Overview

### HTTP Method Semantics

```
┌──────────────────────────────────────────────────────────────┐
│                    HTTP METHODS                               │
└──────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
    SAFE METHODS        IDEMPOTENT             NON-IDEMPOTENT
        │                     │                     │
    GET, HEAD           PUT, DELETE               POST
        │                     │                     │
   No side effects    Same result every time   Creates new resource
```

### Method Characteristics

| Method | Safe | Idempotent | Has Body | Use Case |
|--------|------|------------|----------|----------|
| **GET** | ✅ Yes | ✅ Yes | ❌ No | Retrieve resource |
| **POST** | ❌ No | ❌ No | ✅ Yes | Create resource |
| **PUT** | ❌ No | ✅ Yes | ✅ Yes | Replace resource |
| **PATCH** | ❌ No | ❌ No | ✅ Yes | Update resource |
| **DELETE** | ❌ No | ✅ Yes | ❌ No | Delete resource |
| **HEAD** | ✅ Yes | ✅ Yes | ❌ No | Get headers only |
| **OPTIONS** | ✅ Yes | ✅ Yes | ❌ No | Get allowed methods |

**Safe**: Method doesn't modify server state  
**Idempotent**: Multiple identical requests have same effect as single request

---

## 📖 Visual: RESTful API Structure

```
Banking RESTful API Structure
══════════════════════════════════════════════════════════════

/api/v1/accounts
│
├── GET     /                  → List all accounts
├── POST    /                  → Create new account
├── GET     /:id               → Get specific account
├── PUT     /:id               → Replace account
├── PATCH   /:id               → Update account
└── DELETE  /:id               → Delete account

/api/v1/accounts/:id/transactions
│
├── GET     /                  → List account transactions
├── POST    /                  → Create new transaction
└── GET     /:transactionId    → Get transaction details

/api/v1/accounts/:id/balance
│
└── GET     /                  → Get current balance
```

---

## 🔍 GET Request - Reading Data

GET requests retrieve data without modifying server state.

### Basic GET Request Handler

```javascript
import http from 'http';
import url from 'url';

const server = http.createServer((req, res) => {
  if (req.method === 'GET') {
    // Parse URL and query parameters
    const parsedUrl = url.parse(req.url, true);
    const pathname = parsedUrl.pathname;
    const query = parsedUrl.query;
    
    if (pathname === '/api/accounts') {
      // Example: /api/accounts?limit=10&status=active
      const limit = query.limit || 10;
      const status = query.status || 'all';
      
      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({
        accounts: [],
        limit,
        status
      }));
    }
  }
});
```

### URL Parameters

```javascript
// Query Parameters: /api/accounts?page=1&limit=10
const query = parsedUrl.query;
const page = parseInt(query.page) || 1;
const limit = parseInt(query.limit) || 10;

// Route Parameters: /api/accounts/:id
// Manual parsing (or use a router like Express)
const accountId = pathname.split('/')[3];  // /api/accounts/ACC-123
```

---

## ✍️ POST Request - Creating Data

POST requests create new resources and typically include a request body.

### Reading Request Body

```javascript
function parseRequestBody(req) {
  return new Promise((resolve, reject) => {
    let body = '';
    
    req.on('data', (chunk) => {
      body += chunk.toString();
      
      // Prevent large payloads (DoS protection)
      if (body.length > 1e6) {  // 1MB limit
        req.connection.destroy();
        reject(new Error('Request body too large'));
      }
    });
    
    req.on('end', () => {
      try {
        const data = JSON.parse(body);
        resolve(data);
      } catch (error) {
        reject(new Error('Invalid JSON'));
      }
    });
    
    req.on('error', (error) => {
      reject(error);
    });
  });
}

// Usage
if (req.method === 'POST') {
  try {
    const data = await parseRequestBody(req);
    
    // Validate data
    if (!data.accountNumber || !data.customerName) {
      res.writeHead(400, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({ error: 'Missing required fields' }));
      return;
    }
    
    // Create account
    const account = createAccount(data);
    
    res.writeHead(201, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify(account));
  } catch (error) {
    res.writeHead(400, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ error: error.message }));
  }
}
```

---

## 🔄 PUT Request - Replacing Data

PUT requests replace entire resources (idempotent).

```javascript
if (req.method === 'PUT') {
  const accountId = pathname.split('/')[3];
  
  try {
    const data = await parseRequestBody(req);
    
    // PUT replaces entire resource
    const account = replaceAccount(accountId, data);
    
    if (!account) {
      res.writeHead(404, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({ error: 'Account not found' }));
      return;
    }
    
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify(account));
  } catch (error) {
    res.writeHead(400, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ error: error.message }));
  }
}
```

---

## 📝 PATCH Request - Updating Data

PATCH requests partially update resources.

```javascript
if (req.method === 'PATCH') {
  const accountId = pathname.split('/')[3];
  
  try {
    const data = await parseRequestBody(req);
    
    // PATCH updates only specified fields
    const account = updateAccount(accountId, data);
    
    if (!account) {
      res.writeHead(404, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({ error: 'Account not found' }));
      return;
    }
    
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify(account));
  } catch (error) {
    res.writeHead(400, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ error: error.message }));
  }
}
```

---

## 🗑️ DELETE Request - Removing Data

DELETE requests remove resources (idempotent).

```javascript
if (req.method === 'DELETE') {
  const accountId = pathname.split('/')[3];
  
  try {
    const success = deleteAccount(accountId);
    
    if (!success) {
      res.writeHead(404, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({ error: 'Account not found' }));
      return;
    }
    
    // 204 No Content (successful deletion, no body)
    res.writeHead(204);
    res.end();
  } catch (error) {
    res.writeHead(500, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ error: error.message }));
  }
}
```

---

## 🏦 Example 1: Complete RESTful Banking Account API

This example demonstrates a production-ready account management API with all CRUD operations.

**File: `banking-account-api.js`**

```javascript
/**
 * RESTful Banking Account API
 * Demonstrates: HTTP methods, request parsing, routing, validation
 */

import http from 'http';
import url from 'url';
import crypto from 'crypto';

console.log('🏦 RESTful Banking Account API\n');
console.log('='.repeat(70));

// ============================================
// In-memory Database
// ============================================

const accounts = new Map();
const transactions = new Map();

// Seed some data
accounts.set('ACC-001', {
  id: 'ACC-001',
  accountNumber: '1234567890',
  customerName: 'John Doe',
  accountType: 'SAVINGS',
  balance: 5000,
  currency: 'USD',
  status: 'ACTIVE',
  createdAt: new Date('2025-01-01').toISOString(),
  updatedAt: new Date('2025-01-01').toISOString()
});

accounts.set('ACC-002', {
  id: 'ACC-002',
  accountNumber: '0987654321',
  customerName: 'Jane Smith',
  accountType: 'CHECKING',
  balance: 10000,
  currency: 'USD',
  status: 'ACTIVE',
  createdAt: new Date('2025-01-02').toISOString(),
  updatedAt: new Date('2025-01-02').toISOString()
});

// ============================================
// Utility Functions
// ============================================

function parseRequestBody(req, maxSize = 1e6) {
  return new Promise((resolve, reject) => {
    let body = '';
    
    req.on('data', (chunk) => {
      body += chunk.toString();
      
      if (body.length > maxSize) {
        req.connection.destroy();
        reject(new Error('Request body too large'));
      }
    });
    
    req.on('end', () => {
      if (body.length === 0) {
        resolve({});
        return;
      }
      
      try {
        const data = JSON.parse(body);
        resolve(data);
      } catch (error) {
        reject(new Error('Invalid JSON'));
      }
    });
    
    req.on('error', reject);
  });
}

function sendJSON(res, statusCode, data) {
  res.writeHead(statusCode, {
    'Content-Type': 'application/json',
    'X-Powered-By': 'Node.js Banking API'
  });
  res.end(JSON.stringify(data, null, 2));
}

function sendError(res, statusCode, message, details = {}) {
  sendJSON(res, statusCode, {
    error: {
      message,
      statusCode,
      timestamp: new Date().toISOString(),
      ...details
    }
  });
}

function generateId(prefix = 'ACC') {
  return `${prefix}-${crypto.randomBytes(4).toString('hex').toUpperCase()}`;
}

function validateAccount(data, isUpdate = false) {
  const errors = [];
  
  if (!isUpdate) {
    if (!data.accountNumber) errors.push('accountNumber is required');
    if (!data.customerName) errors.push('customerName is required');
    if (!data.accountType) errors.push('accountType is required');
  }
  
  if (data.accountType && !['SAVINGS', 'CHECKING', 'BUSINESS'].includes(data.accountType)) {
    errors.push('accountType must be SAVINGS, CHECKING, or BUSINESS');
  }
  
  if (data.balance !== undefined && (isNaN(data.balance) || data.balance < 0)) {
    errors.push('balance must be a non-negative number');
  }
  
  if (data.status && !['ACTIVE', 'INACTIVE', 'FROZEN'].includes(data.status)) {
    errors.push('status must be ACTIVE, INACTIVE, or FROZEN');
  }
  
  return errors;
}

// ============================================
// Route Handlers
// ============================================

class AccountController {
  // GET /api/accounts - List all accounts
  static listAccounts(req, res, query) {
    console.log('📋 LIST ACCOUNTS');
    
    // Parse query parameters
    const page = parseInt(query.page) || 1;
    const limit = parseInt(query.limit) || 10;
    const status = query.status;
    const type = query.type;
    
    // Filter accounts
    let filteredAccounts = Array.from(accounts.values());
    
    if (status) {
      filteredAccounts = filteredAccounts.filter(acc => acc.status === status);
    }
    
    if (type) {
      filteredAccounts = filteredAccounts.filter(acc => acc.accountType === type);
    }
    
    // Pagination
    const startIndex = (page - 1) * limit;
    const endIndex = startIndex + limit;
    const paginatedAccounts = filteredAccounts.slice(startIndex, endIndex);
    
    console.log(`   Found ${filteredAccounts.length} accounts (showing ${paginatedAccounts.length})`);
    
    sendJSON(res, 200, {
      data: paginatedAccounts,
      pagination: {
        page,
        limit,
        total: filteredAccounts.length,
        totalPages: Math.ceil(filteredAccounts.length / limit)
      }
    });
  }
  
  // GET /api/accounts/:id - Get specific account
  static getAccount(req, res, accountId) {
    console.log(`🔍 GET ACCOUNT: ${accountId}`);
    
    const account = accounts.get(accountId);
    
    if (!account) {
      console.log('   ❌ Account not found');
      sendError(res, 404, 'Account not found', { accountId });
      return;
    }
    
    console.log(`   ✅ Found account for ${account.customerName}`);
    sendJSON(res, 200, { data: account });
  }
  
  // POST /api/accounts - Create new account
  static async createAccount(req, res) {
    console.log('➕ CREATE ACCOUNT');
    
    try {
      const data = await parseRequestBody(req);
      console.log('   Data:', JSON.stringify(data, null, 2));
      
      // Validate
      const errors = validateAccount(data);
      if (errors.length > 0) {
        console.log('   ❌ Validation errors:', errors);
        sendError(res, 400, 'Validation failed', { errors });
        return;
      }
      
      // Check for duplicate account number
      const existingAccount = Array.from(accounts.values())
        .find(acc => acc.accountNumber === data.accountNumber);
      
      if (existingAccount) {
        console.log('   ❌ Account number already exists');
        sendError(res, 409, 'Account number already exists', {
          accountNumber: data.accountNumber
        });
        return;
      }
      
      // Create account
      const account = {
        id: generateId('ACC'),
        accountNumber: data.accountNumber,
        customerName: data.customerName,
        accountType: data.accountType,
        balance: data.balance || 0,
        currency: data.currency || 'USD',
        status: 'ACTIVE',
        createdAt: new Date().toISOString(),
        updatedAt: new Date().toISOString()
      };
      
      accounts.set(account.id, account);
      
      console.log(`   ✅ Created account ${account.id} for ${account.customerName}`);
      
      sendJSON(res, 201, {
        message: 'Account created successfully',
        data: account
      });
      
    } catch (error) {
      console.error('   ❌ Error:', error.message);
      sendError(res, 400, error.message);
    }
  }
  
  // PUT /api/accounts/:id - Replace account
  static async replaceAccount(req, res, accountId) {
    console.log(`🔄 REPLACE ACCOUNT: ${accountId}`);
    
    try {
      const data = await parseRequestBody(req);
      console.log('   Data:', JSON.stringify(data, null, 2));
      
      // Check if account exists
      const existingAccount = accounts.get(accountId);
      if (!existingAccount) {
        console.log('   ❌ Account not found');
        sendError(res, 404, 'Account not found', { accountId });
        return;
      }
      
      // Validate
      const errors = validateAccount(data, false);
      if (errors.length > 0) {
        console.log('   ❌ Validation errors:', errors);
        sendError(res, 400, 'Validation failed', { errors });
        return;
      }
      
      // Replace account (keep id, createdAt)
      const account = {
        id: accountId,
        accountNumber: data.accountNumber,
        customerName: data.customerName,
        accountType: data.accountType,
        balance: data.balance || 0,
        currency: data.currency || 'USD',
        status: data.status || 'ACTIVE',
        createdAt: existingAccount.createdAt,
        updatedAt: new Date().toISOString()
      };
      
      accounts.set(accountId, account);
      
      console.log(`   ✅ Replaced account ${accountId}`);
      
      sendJSON(res, 200, {
        message: 'Account replaced successfully',
        data: account
      });
      
    } catch (error) {
      console.error('   ❌ Error:', error.message);
      sendError(res, 400, error.message);
    }
  }
  
  // PATCH /api/accounts/:id - Update account
  static async updateAccount(req, res, accountId) {
    console.log(`✏️  UPDATE ACCOUNT: ${accountId}`);
    
    try {
      const data = await parseRequestBody(req);
      console.log('   Data:', JSON.stringify(data, null, 2));
      
      // Check if account exists
      const existingAccount = accounts.get(accountId);
      if (!existingAccount) {
        console.log('   ❌ Account not found');
        sendError(res, 404, 'Account not found', { accountId });
        return;
      }
      
      // Validate
      const errors = validateAccount(data, true);
      if (errors.length > 0) {
        console.log('   ❌ Validation errors:', errors);
        sendError(res, 400, 'Validation failed', { errors });
        return;
      }
      
      // Update only provided fields
      const account = {
        ...existingAccount,
        ...data,
        id: accountId,  // Don't allow changing ID
        createdAt: existingAccount.createdAt,  // Don't allow changing creation date
        updatedAt: new Date().toISOString()
      };
      
      accounts.set(accountId, account);
      
      console.log(`   ✅ Updated account ${accountId}`);
      
      sendJSON(res, 200, {
        message: 'Account updated successfully',
        data: account
      });
      
    } catch (error) {
      console.error('   ❌ Error:', error.message);
      sendError(res, 400, error.message);
    }
  }
  
  // DELETE /api/accounts/:id - Delete account
  static deleteAccount(req, res, accountId) {
    console.log(`🗑️  DELETE ACCOUNT: ${accountId}`);
    
    const account = accounts.get(accountId);
    
    if (!account) {
      console.log('   ❌ Account not found');
      sendError(res, 404, 'Account not found', { accountId });
      return;
    }
    
    // Check if account has balance
    if (account.balance > 0) {
      console.log('   ❌ Cannot delete account with balance');
      sendError(res, 400, 'Cannot delete account with non-zero balance', {
        accountId,
        balance: account.balance
      });
      return;
    }
    
    accounts.delete(accountId);
    
    console.log(`   ✅ Deleted account ${accountId}`);
    
    // 204 No Content - successful deletion, no response body
    res.writeHead(204);
    res.end();
  }
  
  // GET /api/accounts/:id/balance - Get account balance
  static getBalance(req, res, accountId) {
    console.log(`💰 GET BALANCE: ${accountId}`);
    
    const account = accounts.get(accountId);
    
    if (!account) {
      console.log('   ❌ Account not found');
      sendError(res, 404, 'Account not found', { accountId });
      return;
    }
    
    console.log(`   ✅ Balance: ${account.currency} ${account.balance}`);
    
    sendJSON(res, 200, {
      data: {
        accountId: account.id,
        balance: account.balance,
        currency: account.currency,
        status: account.status,
        asOfDate: new Date().toISOString()
      }
    });
  }
}

// ============================================
// Router
// ============================================

function router(req, res) {
  const parsedUrl = url.parse(req.url, true);
  const pathname = parsedUrl.pathname;
  const query = parsedUrl.query;
  const method = req.method;
  
  console.log(`\n${method} ${pathname}`);
  
  // CORS headers
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, PATCH, DELETE, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type');
  
  // Handle OPTIONS (preflight)
  if (method === 'OPTIONS') {
    res.writeHead(204);
    res.end();
    return;
  }
  
  // Route matching
  const routes = {
    'GET /api/accounts': () => AccountController.listAccounts(req, res, query),
    'POST /api/accounts': () => AccountController.createAccount(req, res),
  };
  
  // Match parameterized routes
  const accountIdMatch = pathname.match(/^\/api\/accounts\/([A-Z0-9-]+)$/);
  const balanceMatch = pathname.match(/^\/api\/accounts\/([A-Z0-9-]+)\/balance$/);
  
  if (accountIdMatch) {
    const accountId = accountIdMatch[1];
    
    if (method === 'GET') {
      AccountController.getAccount(req, res, accountId);
      return;
    } else if (method === 'PUT') {
      AccountController.replaceAccount(req, res, accountId);
      return;
    } else if (method === 'PATCH') {
      AccountController.updateAccount(req, res, accountId);
      return;
    } else if (method === 'DELETE') {
      AccountController.deleteAccount(req, res, accountId);
      return;
    }
  }
  
  if (balanceMatch && method === 'GET') {
    const accountId = balanceMatch[1];
    AccountController.getBalance(req, res, accountId);
    return;
  }
  
  // Check static routes
  const routeKey = `${method} ${pathname}`;
  if (routes[routeKey]) {
    routes[routeKey]();
    return;
  }
  
  // 404 Not Found
  console.log('   ❌ Route not found');
  sendError(res, 404, 'Route not found', { method, pathname });
}

// ============================================
// Create Server
// ============================================

const PORT = process.env.PORT || 3000;

const server = http.createServer(router);

server.listen(PORT, () => {
  console.log(`\n✅ Banking Account API running on http://localhost:${PORT}`);
  console.log('\nAvailable endpoints:');
  console.log('  GET    /api/accounts              - List all accounts');
  console.log('  POST   /api/accounts              - Create new account');
  console.log('  GET    /api/accounts/:id          - Get specific account');
  console.log('  PUT    /api/accounts/:id          - Replace account');
  console.log('  PATCH  /api/accounts/:id          - Update account');
  console.log('  DELETE /api/accounts/:id          - Delete account');
  console.log('  GET    /api/accounts/:id/balance  - Get account balance');
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
node banking-account-api.js
```

**Testing with curl:**

```bash
# 1. List all accounts
curl http://localhost:3000/api/accounts

# 2. List accounts with filters
curl "http://localhost:3000/api/accounts?status=ACTIVE&limit=5"

# 3. Get specific account
curl http://localhost:3000/api/accounts/ACC-001

# 4. Create new account
curl -X POST http://localhost:3000/api/accounts \
  -H "Content-Type: application/json" \
  -d '{
    "accountNumber": "5555555555",
    "customerName": "Alice Johnson",
    "accountType": "SAVINGS",
    "balance": 1000
  }'

# 5. Update account (PATCH)
curl -X PATCH http://localhost:3000/api/accounts/ACC-001 \
  -H "Content-Type: application/json" \
  -d '{"balance": 6000}'

# 6. Replace account (PUT)
curl -X PUT http://localhost:3000/api/accounts/ACC-001 \
  -H "Content-Type: application/json" \
  -d '{
    "accountNumber": "1234567890",
    "customerName": "John Doe Updated",
    "accountType": "CHECKING",
    "balance": 7000
  }'

# 7. Get account balance
curl http://localhost:3000/api/accounts/ACC-001/balance

# 8. Delete account
curl -X DELETE http://localhost:3000/api/accounts/ACC-002

# 9. Try to delete account with balance (should fail)
curl -X DELETE http://localhost:3000/api/accounts/ACC-001
```

**Expected Output (Server):**

```
🏦 RESTful Banking Account API

======================================================================

✅ Banking Account API running on http://localhost:3000

Available endpoints:
  GET    /api/accounts              - List all accounts
  POST   /api/accounts              - Create new account
  GET    /api/accounts/:id          - Get specific account
  PUT    /api/accounts/:id          - Replace account
  PATCH  /api/accounts/:id          - Update account
  DELETE /api/accounts/:id          - Delete account
  GET    /api/accounts/:id/balance  - Get account balance

======================================================================

GET /api/accounts
📋 LIST ACCOUNTS
   Found 2 accounts (showing 2)

GET /api/accounts/ACC-001
🔍 GET ACCOUNT: ACC-001
   ✅ Found account for John Doe

POST /api/accounts
➕ CREATE ACCOUNT
   Data: {
  "accountNumber": "5555555555",
  "customerName": "Alice Johnson",
  "accountType": "SAVINGS",
  "balance": 1000
}
   ✅ Created account ACC-AB12CD34 for Alice Johnson

PATCH /api/accounts/ACC-001
✏️  UPDATE ACCOUNT: ACC-001
   Data: {
  "balance": 6000
}
   ✅ Updated account ACC-001

GET /api/accounts/ACC-001/balance
💰 GET BALANCE: ACC-001
   ✅ Balance: USD 6000

DELETE /api/accounts/ACC-001
🗑️  DELETE ACCOUNT: ACC-001
   ❌ Cannot delete account with balance
```

**Key Learnings from Example 1**:

1. **HTTP Method Handling**: Different methods for different operations
2. **Request Body Parsing**: Asynchronous body reading with size limits
3. **URL Routing**: Pattern matching for parameterized routes
4. **Query Parameters**: Pagination, filtering, sorting
5. **Validation**: Input validation with error messages
6. **HTTP Status Codes**: Proper status codes (200, 201, 400, 404, 409)
7. **Error Handling**: Structured error responses
8. **CORS Support**: Cross-origin resource sharing headers
9. **Idempotency**: PUT and DELETE are idempotent
10. **RESTful Design**: Resource-oriented URLs

---

## 🏦 Example 2: Request Body Parsers for Different Content Types

This example demonstrates handling multiple content types: JSON, URL-encoded forms, and multipart file uploads.

**File: `content-type-handler.js`**

```javascript
/**
 * Content Type Handler for Banking API
 * Demonstrates: JSON, URL-encoded, multipart parsing
 */

import http from 'http';
import url from 'url';
import querystring from 'querystring';
import { Buffer } from 'buffer';

console.log('📦 Content Type Handler System\n');
console.log('='.repeat(70));

// ============================================
// Request Body Parsers
// ============================================

/**
 * Parse JSON request body
 */
function parseJSON(req, maxSize = 1e6) {
  return new Promise((resolve, reject) => {
    const contentType = req.headers['content-type'];
    
    if (!contentType || !contentType.includes('application/json')) {
      reject(new Error('Content-Type must be application/json'));
      return;
    }
    
    let body = '';
    
    req.on('data', (chunk) => {
      body += chunk.toString();
      
      if (body.length > maxSize) {
        req.connection.destroy();
        reject(new Error('Request body too large'));
      }
    });
    
    req.on('end', () => {
      try {
        const data = body.length === 0 ? {} : JSON.parse(body);
        resolve(data);
      } catch (error) {
        reject(new Error('Invalid JSON: ' + error.message));
      }
    });
    
    req.on('error', reject);
  });
}

/**
 * Parse URL-encoded form data
 */
function parseURLEncoded(req, maxSize = 1e6) {
  return new Promise((resolve, reject) => {
    const contentType = req.headers['content-type'];
    
    if (!contentType || !contentType.includes('application/x-www-form-urlencoded')) {
      reject(new Error('Content-Type must be application/x-www-form-urlencoded'));
      return;
    }
    
    let body = '';
    
    req.on('data', (chunk) => {
      body += chunk.toString();
      
      if (body.length > maxSize) {
        req.connection.destroy();
        reject(new Error('Request body too large'));
      }
    });
    
    req.on('end', () => {
      try {
        const data = querystring.parse(body);
        resolve(data);
      } catch (error) {
        reject(new Error('Invalid URL-encoded data: ' + error.message));
      }
    });
    
    req.on('error', reject);
  });
}

/**
 * Parse multipart/form-data (simple implementation)
 */
function parseMultipart(req, maxSize = 10e6) {
  return new Promise((resolve, reject) => {
    const contentType = req.headers['content-type'];
    
    if (!contentType || !contentType.includes('multipart/form-data')) {
      reject(new Error('Content-Type must be multipart/form-data'));
      return;
    }
    
    // Extract boundary from Content-Type header
    const boundaryMatch = contentType.match(/boundary=([^;]+)/);
    if (!boundaryMatch) {
      reject(new Error('Missing boundary in multipart data'));
      return;
    }
    
    const boundary = '--' + boundaryMatch[1];
    const chunks = [];
    let totalSize = 0;
    
    req.on('data', (chunk) => {
      chunks.push(chunk);
      totalSize += chunk.length;
      
      if (totalSize > maxSize) {
        req.connection.destroy();
        reject(new Error('Request body too large'));
      }
    });
    
    req.on('end', () => {
      try {
        const buffer = Buffer.concat(chunks);
        const parts = parseMultipartBuffer(buffer, boundary);
        resolve(parts);
      } catch (error) {
        reject(new Error('Invalid multipart data: ' + error.message));
      }
    });
    
    req.on('error', reject);
  });
}

/**
 * Parse multipart buffer into fields and files
 */
function parseMultipartBuffer(buffer, boundary) {
  const fields = {};
  const files = [];
  
  // Split by boundary
  const parts = buffer.toString('binary').split(boundary);
  
  for (let part of parts) {
    if (!part || part === '--\r\n' || part === '--') continue;
    
    // Remove leading/trailing CRLF
    part = part.replace(/^\r\n/, '').replace(/\r\n$/, '');
    
    // Split headers and body
    const headerEndIndex = part.indexOf('\r\n\r\n');
    if (headerEndIndex === -1) continue;
    
    const headerSection = part.substring(0, headerEndIndex);
    const bodySection = part.substring(headerEndIndex + 4);
    
    // Parse Content-Disposition header
    const dispositionMatch = headerSection.match(/Content-Disposition: form-data; name="([^"]+)"(?:; filename="([^"]+)")?/);
    if (!dispositionMatch) continue;
    
    const fieldName = dispositionMatch[1];
    const fileName = dispositionMatch[2];
    
    if (fileName) {
      // This is a file upload
      const contentTypeMatch = headerSection.match(/Content-Type: (.+)/);
      const contentType = contentTypeMatch ? contentTypeMatch[1].trim() : 'application/octet-stream';
      
      files.push({
        fieldName,
        fileName,
        contentType,
        data: Buffer.from(bodySection, 'binary'),
        size: bodySection.length
      });
    } else {
      // This is a regular field
      fields[fieldName] = bodySection.trim();
    }
  }
  
  return { fields, files };
}

/**
 * Universal body parser (detects content type)
 */
async function parseBody(req) {
  const contentType = req.headers['content-type'] || '';
  
  if (contentType.includes('application/json')) {
    return await parseJSON(req);
  } else if (contentType.includes('application/x-www-form-urlencoded')) {
    return await parseURLEncoded(req);
  } else if (contentType.includes('multipart/form-data')) {
    return await parseMultipart(req);
  } else {
    throw new Error(`Unsupported Content-Type: ${contentType}`);
  }
}

// ============================================
// Document Upload Handler
// ============================================

class DocumentUploadHandler {
  constructor() {
    this.uploads = new Map();
    this.allowedTypes = ['application/pdf', 'image/jpeg', 'image/png'];
    this.maxFileSize = 5 * 1024 * 1024;  // 5MB
  }
  
  /**
   * Handle document upload
   */
  async handleUpload(req, res) {
    console.log('\n📤 UPLOAD: Document upload request');
    
    try {
      const { fields, files } = await parseMultipart(req);
      
      console.log('   Fields:', fields);
      console.log('   Files:', files.length);
      
      // Validate required fields
      if (!fields.accountId || !fields.documentType) {
        return this.sendError(res, 400, 'Missing required fields: accountId, documentType');
      }
      
      // Validate file upload
      if (!files || files.length === 0) {
        return this.sendError(res, 400, 'No file uploaded');
      }
      
      const file = files[0];
      console.log(`   File: ${file.fileName} (${file.size} bytes, ${file.contentType})`);
      
      // Validate file type
      if (!this.allowedTypes.includes(file.contentType)) {
        return this.sendError(res, 400, `Invalid file type. Allowed: ${this.allowedTypes.join(', ')}`);
      }
      
      // Validate file size
      if (file.size > this.maxFileSize) {
        return this.sendError(res, 400, `File too large. Max size: ${this.maxFileSize / 1024 / 1024}MB`);
      }
      
      // Generate upload ID
      const uploadId = `DOC-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
      
      // Store upload metadata (in production, save file to disk/S3)
      const upload = {
        uploadId,
        accountId: fields.accountId,
        documentType: fields.documentType,
        fileName: file.fileName,
        fileSize: file.size,
        contentType: file.contentType,
        uploadedAt: new Date().toISOString(),
        status: 'PENDING_REVIEW'
      };
      
      this.uploads.set(uploadId, upload);
      
      console.log(`   ✅ Upload successful: ${uploadId}`);
      
      this.sendJSON(res, 201, {
        message: 'Document uploaded successfully',
        data: upload
      });
      
    } catch (error) {
      console.error('   ❌ Upload error:', error.message);
      this.sendError(res, 400, error.message);
    }
  }
  
  /**
   * Get upload status
   */
  getUploadStatus(req, res, uploadId) {
    console.log(`\n📄 GET UPLOAD: ${uploadId}`);
    
    const upload = this.uploads.get(uploadId);
    
    if (!upload) {
      return this.sendError(res, 404, 'Upload not found');
    }
    
    console.log(`   ✅ Found upload for account ${upload.accountId}`);
    
    this.sendJSON(res, 200, { data: upload });
  }
  
  sendJSON(res, statusCode, data) {
    res.writeHead(statusCode, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify(data, null, 2));
  }
  
  sendError(res, statusCode, message) {
    this.sendJSON(res, statusCode, {
      error: {
        message,
        statusCode,
        timestamp: new Date().toISOString()
      }
    });
  }
}

// ============================================
// Form Submission Handler
// ============================================

class FormHandler {
  /**
   * Handle URL-encoded form submission
   */
  async handleFormSubmit(req, res) {
    console.log('\n📝 FORM: URL-encoded form submission');
    
    try {
      const data = await parseURLEncoded(req);
      
      console.log('   Form data:', data);
      
      // Validate form data
      if (!data.email || !data.phone) {
        return this.sendError(res, 400, 'Missing required fields: email, phone');
      }
      
      // Process form (e.g., update customer contact info)
      const result = {
        customerId: data.customerId || 'NEW',
        email: data.email,
        phone: data.phone,
        preferredContact: data.preferredContact || 'email',
        updatedAt: new Date().toISOString()
      };
      
      console.log('   ✅ Form processed successfully');
      
      this.sendJSON(res, 200, {
        message: 'Contact information updated',
        data: result
      });
      
    } catch (error) {
      console.error('   ❌ Form error:', error.message);
      this.sendError(res, 400, error.message);
    }
  }
  
  sendJSON(res, statusCode, data) {
    res.writeHead(statusCode, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify(data, null, 2));
  }
  
  sendError(res, statusCode, message) {
    this.sendJSON(res, statusCode, {
      error: {
        message,
        statusCode,
        timestamp: new Date().toISOString()
      }
    });
  }
}

// ============================================
// Create Server
// ============================================

const PORT = 3001;

const documentHandler = new DocumentUploadHandler();
const formHandler = new FormHandler();

const server = http.createServer(async (req, res) => {
  const parsedUrl = url.parse(req.url, true);
  const pathname = parsedUrl.pathname;
  const method = req.method;
  
  console.log(`\n${method} ${pathname}`);
  
  // CORS headers
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type');
  
  // Handle OPTIONS
  if (method === 'OPTIONS') {
    res.writeHead(204);
    res.end();
    return;
  }
  
  try {
    // Route: POST /api/documents/upload
    if (method === 'POST' && pathname === '/api/documents/upload') {
      await documentHandler.handleUpload(req, res);
      return;
    }
    
    // Route: GET /api/documents/:uploadId
    const uploadMatch = pathname.match(/^\/api\/documents\/([A-Z0-9-]+)$/);
    if (method === 'GET' && uploadMatch) {
      documentHandler.getUploadStatus(req, res, uploadMatch[1]);
      return;
    }
    
    // Route: POST /api/forms/contact
    if (method === 'POST' && pathname === '/api/forms/contact') {
      await formHandler.handleFormSubmit(req, res);
      return;
    }
    
    // 404 Not Found
    res.writeHead(404, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
      error: {
        message: 'Route not found',
        statusCode: 404
      }
    }));
    
  } catch (error) {
    console.error('Server error:', error);
    res.writeHead(500, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
      error: {
        message: 'Internal server error',
        statusCode: 500
      }
    }));
  }
});

server.listen(PORT, () => {
  console.log(`\n✅ Content Type Handler running on http://localhost:${PORT}`);
  console.log('\nAvailable endpoints:');
  console.log('  POST /api/documents/upload  - Upload document (multipart/form-data)');
  console.log('  GET  /api/documents/:id     - Get upload status');
  console.log('  POST /api/forms/contact     - Submit form (application/x-www-form-urlencoded)');
  console.log('\n' + '='.repeat(70) + '\n');
});
```

**To run this example:**

```bash
node content-type-handler.js
```

**Testing with curl:**

```bash
# 1. Upload document (multipart/form-data)
curl -X POST http://localhost:3001/api/documents/upload \
  -F "accountId=ACC-12345" \
  -F "documentType=ID_PROOF" \
  -F "file=@/path/to/document.pdf"

# 2. Get upload status
curl http://localhost:3001/api/documents/DOC-1731579600000-abc123def

# 3. Submit contact form (URL-encoded)
curl -X POST http://localhost:3001/api/forms/contact \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "customerId=CUST-123&email=john@example.com&phone=555-1234&preferredContact=email"
```

**Expected Output:**

```
📦 Content Type Handler System

======================================================================

✅ Content Type Handler running on http://localhost:3001

Available endpoints:
  POST /api/documents/upload  - Upload document (multipart/form-data)
  GET  /api/documents/:id     - Get upload status
  POST /api/forms/contact     - Submit form (application/x-www-form-urlencoded)

======================================================================

POST /api/documents/upload
📤 UPLOAD: Document upload request
   Fields: { accountId: 'ACC-12345', documentType: 'ID_PROOF' }
   Files: 1
   File: document.pdf (51234 bytes, application/pdf)
   ✅ Upload successful: DOC-1731579600000-abc123def

GET /api/documents/DOC-1731579600000-abc123def
📄 GET UPLOAD: DOC-1731579600000-abc123def
   ✅ Found upload for account ACC-12345

POST /api/forms/contact
📝 FORM: URL-encoded form submission
   Form data: {
  customerId: 'CUST-123',
  email: 'john@example.com',
  phone: '555-1234',
  preferredContact: 'email'
}
   ✅ Form processed successfully
```

**Key Learnings from Example 2**:

1. **Content Type Detection**: Parse based on Content-Type header
2. **JSON Parsing**: Handle application/json requests
3. **URL-Encoded Forms**: Parse application/x-www-form-urlencoded
4. **Multipart Upload**: Handle file uploads with multipart/form-data
5. **Boundary Parsing**: Extract boundary from multipart headers
6. **File Validation**: Check file type, size, format
7. **Binary Data Handling**: Work with Buffer for file data
8. **Security**: Enforce size limits to prevent DoS
9. **Metadata Storage**: Store upload information
10. **Error Messages**: Clear validation error messages

---

## 📊 Content Negotiation

Handle different response formats based on client preferences:

```javascript
function sendResponse(req, res, data) {
  const acceptHeader = req.headers.accept || 'application/json';
  
  if (acceptHeader.includes('application/json')) {
    // Send JSON
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify(data));
    
  } else if (acceptHeader.includes('application/xml')) {
    // Send XML
    res.writeHead(200, { 'Content-Type': 'application/xml' });
    res.end(`<?xml version="1.0"?>
      <response>
        <status>success</status>
        <data>${JSON.stringify(data)}</data>
      </response>
    `);
    
  } else if (acceptHeader.includes('text/html')) {
    // Send HTML
    res.writeHead(200, { 'Content-Type': 'text/html' });
    res.end(`
      <html>
        <body>
          <h1>Response</h1>
          <pre>${JSON.stringify(data, null, 2)}</pre>
        </body>
      </html>
    `);
    
  } else {
    // Default to JSON
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify(data));
  }
}
```

---

## 🎯 HTTP Headers Deep Dive

### Request Headers

```javascript
function logRequestHeaders(req) {
  console.log('Request Headers:');
  console.log('  Content-Type:', req.headers['content-type']);
  console.log('  Content-Length:', req.headers['content-length']);
  console.log('  Accept:', req.headers.accept);
  console.log('  Accept-Encoding:', req.headers['accept-encoding']);
  console.log('  User-Agent:', req.headers['user-agent']);
  console.log('  Authorization:', req.headers.authorization);
  console.log('  Cookie:', req.headers.cookie);
  console.log('  Origin:', req.headers.origin);
  console.log('  Referer:', req.headers.referer);
  console.log('  X-Forwarded-For:', req.headers['x-forwarded-for']);
}
```

### Response Headers

```javascript
function setSecurityHeaders(res) {
  // Security headers
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('X-XSS-Protection', '1; mode=block');
  res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
  
  // CORS headers
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
  
  // Cache control
  res.setHeader('Cache-Control', 'no-store, no-cache, must-revalidate');
  res.setHeader('Pragma', 'no-cache');
  
  // Custom headers
  res.setHeader('X-Powered-By', 'Banking API v1.0');
  res.setHeader('X-Request-ID', generateRequestId());
}
```

---

## 💡 DO's - Best Practices

### 1. ✅ DO: Always Validate Request Body

```javascript
// ✅ GOOD: Comprehensive validation
async function createTransaction(req, res) {
  try {
    const data = await parseJSON(req);
    
    const errors = [];
    
    if (!data.fromAccount) errors.push('fromAccount is required');
    if (!data.toAccount) errors.push('toAccount is required');
    if (!data.amount || data.amount <= 0) errors.push('amount must be positive');
    if (data.amount > 1000000) errors.push('amount exceeds limit');
    
    if (errors.length > 0) {
      res.writeHead(400, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({ errors }));
      return;
    }
    
    // Process transaction
  } catch (error) {
    res.writeHead(400, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ error: error.message }));
  }
}

// ❌ BAD: No validation
async function createTransaction(req, res) {
  const data = await parseJSON(req);
  // Directly use data without validation - DANGEROUS!
  processTransaction(data);
}
```

**Why**: Input validation prevents invalid data, SQL injection, and business logic errors.

---

### 2. ✅ DO: Use Appropriate HTTP Status Codes

```javascript
// ✅ GOOD: Precise status codes
if (account) {
  res.writeHead(200, { 'Content-Type': 'application/json' });  // OK
} else {
  res.writeHead(404, { 'Content-Type': 'application/json' });  // Not Found
}

if (validationErrors) {
  res.writeHead(400, { 'Content-Type': 'application/json' });  // Bad Request
}

if (duplicate) {
  res.writeHead(409, { 'Content-Type': 'application/json' });  // Conflict
}

if (created) {
  res.writeHead(201, { 'Content-Type': 'application/json' });  // Created
}

// ❌ BAD: Always 200
res.writeHead(200, { 'Content-Type': 'application/json' });
res.end(JSON.stringify({ error: 'Not found' }));  // Wrong! Should be 404
```

**Why**: Proper status codes help clients understand response without parsing body.

---

### 3. ✅ DO: Implement Request Size Limits

```javascript
// ✅ GOOD: Size limit protection
function parseBody(req, maxSize = 1e6) {  // 1MB default
  return new Promise((resolve, reject) => {
    let body = '';
    let size = 0;
    
    req.on('data', (chunk) => {
      size += chunk.length;
      
      if (size > maxSize) {
        req.connection.destroy();
        reject(new Error('Payload too large'));
        return;
      }
      
      body += chunk.toString();
    });
    
    req.on('end', () => resolve(JSON.parse(body)));
  });
}

// ❌ BAD: No size limit
function parseBody(req) {
  return new Promise((resolve, reject) => {
    let body = '';
    req.on('data', (chunk) => {
      body += chunk.toString();  // Unbounded growth!
    });
    req.on('end', () => resolve(JSON.parse(body)));
  });
}
```

**Why**: Prevents DoS attacks via large payloads exhausting server memory.

---

### 4. ✅ DO: Handle Content-Type Properly

```javascript
// ✅ GOOD: Check and validate Content-Type
async function handleRequest(req, res) {
  const contentType = req.headers['content-type'];
  
  if (!contentType) {
    res.writeHead(400, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ error: 'Content-Type header required' }));
    return;
  }
  
  if (contentType.includes('application/json')) {
    const data = await parseJSON(req);
    // Process JSON data
  } else {
    res.writeHead(415, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ error: 'Unsupported Media Type' }));
  }
}

// ❌ BAD: Assume JSON
async function handleRequest(req, res) {
  const data = await parseJSON(req);  // What if not JSON?
}
```

**Why**: Prevents parsing errors and ensures correct data interpretation.

---

### 5. ✅ DO: Use Idempotent Methods Correctly

```javascript
// ✅ GOOD: PUT is idempotent (same result every time)
app.put('/api/accounts/:id', async (req, res) => {
  const data = await parseJSON(req);
  
  // Replace entire resource
  const account = {
    id: req.params.id,
    ...data,
    updatedAt: new Date().toISOString()
  };
  
  accounts.set(req.params.id, account);
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify(account));
});

// Calling PUT multiple times with same data = same result

// ❌ BAD: PUT increments balance (not idempotent)
app.put('/api/accounts/:id', async (req, res) => {
  const data = await parseJSON(req);
  account.balance += data.amount;  // Different result each call!
});
```

**Why**: Idempotency allows safe retries and prevents duplicate operations.

---

### 6. ✅ DO: Return Consistent Error Format

```javascript
// ✅ GOOD: Consistent error structure
function sendError(res, statusCode, message, details = {}) {
  res.writeHead(statusCode, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({
    error: {
      message,
      statusCode,
      timestamp: new Date().toISOString(),
      ...details
    }
  }, null, 2));
}

// Usage
sendError(res, 400, 'Validation failed', {
  errors: ['amount is required', 'amount must be positive']
});

// ❌ BAD: Inconsistent error format
res.writeHead(400);
res.end('Error: bad request');  // String

res.writeHead(404);
res.end(JSON.stringify({ msg: 'Not found' }));  // Different structure

res.writeHead(500);
res.end(JSON.stringify({ errorMessage: 'Server error', code: 500 }));  // Yet another structure
```

**Why**: Consistent format makes client error handling predictable and easier.

---

### 7. ✅ DO: Use Query Parameters for Filtering/Pagination

```javascript
// ✅ GOOD: Query params for GET requests
app.get('/api/transactions', (req, res) => {
  const parsedUrl = url.parse(req.url, true);
  const query = parsedUrl.query;
  
  // Parse query parameters
  const page = parseInt(query.page) || 1;
  const limit = parseInt(query.limit) || 10;
  const status = query.status;
  const startDate = query.startDate;
  
  // Filter and paginate
  let filtered = transactions;
  
  if (status) {
    filtered = filtered.filter(t => t.status === status);
  }
  
  if (startDate) {
    filtered = filtered.filter(t => t.date >= startDate);
  }
  
  const start = (page - 1) * limit;
  const paginated = filtered.slice(start, start + limit);
  
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({
    data: paginated,
    pagination: {
      page,
      limit,
      total: filtered.length
    }
  }));
});

// ❌ BAD: Body in GET request
app.get('/api/transactions', async (req, res) => {
  const body = await parseJSON(req);  // GET shouldn't have body!
  const filtered = transactions.filter(t => t.status === body.status);
});
```

**Why**: GET requests should use query parameters, not request body.

---

### 8. ✅ DO: Implement CORS Properly

```javascript
// ✅ GOOD: Proper CORS handling
function handleCORS(req, res) {
  const origin = req.headers.origin;
  const allowedOrigins = [
    'https://banking.example.com',
    'https://mobile.example.com'
  ];
  
  if (allowedOrigins.includes(origin)) {
    res.setHeader('Access-Control-Allow-Origin', origin);
  }
  
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
  res.setHeader('Access-Control-Allow-Credentials', 'true');
  res.setHeader('Access-Control-Max-Age', '86400');  // 24 hours
  
  // Handle preflight
  if (req.method === 'OPTIONS') {
    res.writeHead(204);
    res.end();
    return true;
  }
  
  return false;
}

// ❌ BAD: Allow all origins
res.setHeader('Access-Control-Allow-Origin', '*');
res.setHeader('Access-Control-Allow-Credentials', 'true');
// Security vulnerability! Can't use * with credentials
```

**Why**: Proper CORS prevents unauthorized cross-origin requests while allowing legitimate ones.

---

### 9. ✅ DO: Add Request ID for Tracing

```javascript
// ✅ GOOD: Add request ID for tracing
import crypto from 'crypto';

function handleRequest(req, res) {
  const requestId = req.headers['x-request-id'] || 
                    crypto.randomUUID();
  
  res.setHeader('X-Request-ID', requestId);
  
  console.log(`[${requestId}] ${req.method} ${req.url}`);
  
  // Use requestId in all logs for this request
  req.requestId = requestId;
  
  // Process request...
}

// ❌ BAD: No request tracking
function handleRequest(req, res) {
  console.log(`${req.method} ${req.url}`);
  // Can't correlate logs for same request
}
```

**Why**: Request IDs enable tracing requests through distributed systems and logs.

---

### 10. ✅ DO: Use Method Override for Limited Clients

```javascript
// ✅ GOOD: Support X-HTTP-Method-Override
function getActualMethod(req) {
  const methodOverride = req.headers['x-http-method-override'];
  
  if (req.method === 'POST' && methodOverride) {
    // Allow POST to tunnel PUT/PATCH/DELETE
    if (['PUT', 'PATCH', 'DELETE'].includes(methodOverride)) {
      return methodOverride;
    }
  }
  
  return req.method;
}

// Usage
const method = getActualMethod(req);

if (method === 'PUT') {
  // Handle PUT even if actual HTTP method was POST
}
```

**Why**: Some clients (old browsers, proxies) only support GET/POST. Method override provides compatibility.

---

## 🚫 DON'Ts - Common Mistakes

### 1. ❌ DON'T: Parse Body Before Checking Method

```javascript
// ❌ BAD: Parse body for GET request
app.get('/api/accounts', async (req, res) => {
  const body = await parseJSON(req);  // GET has no body!
  // This will hang or timeout
});

// ✅ GOOD: Only parse body for methods that have it
if (['POST', 'PUT', 'PATCH'].includes(req.method)) {
  const body = await parseJSON(req);
}
```

**Why**: GET, DELETE, HEAD, OPTIONS don't have request bodies. Parsing will hang.

---

### 2. ❌ DON'T: Ignore Request Method

```javascript
// ❌ BAD: Handle all methods the same
app.all('/api/accounts', (req, res) => {
  // Handles GET, POST, PUT, DELETE all the same way
  accounts.create(data);  // Wrong for GET!
});

// ✅ GOOD: Handle each method appropriately
if (req.method === 'GET') {
  // List accounts
} else if (req.method === 'POST') {
  // Create account
} else if (req.method === 'PUT') {
  // Update account
} else {
  res.writeHead(405, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({ error: 'Method not allowed' }));
}
```

**Why**: Different methods have different semantics and side effects.

---

### 3. ❌ DON'T: Use POST for Everything

```javascript
// ❌ BAD: POST for read operations
POST /api/getAccount
POST /api/listTransactions
POST /api/deleteAccount

// ✅ GOOD: Use appropriate methods
GET    /api/accounts/:id           // Read
GET    /api/transactions            // List
DELETE /api/accounts/:id           // Delete
POST   /api/accounts                // Create
PUT    /api/accounts/:id           // Replace
PATCH  /api/accounts/:id           // Update
```

**Why**: HTTP methods have semantic meaning. Using correct methods enables caching, idempotency, and proper REST.

---

### 4. ❌ DON'T: Return Different Status for Same Error

```javascript
// ❌ BAD: Inconsistent status codes
if (!account) {
  res.writeHead(404);  // Sometimes 404
}

if (!account) {
  res.writeHead(400);  // Sometimes 400 for same error!
}

// ✅ GOOD: Consistent status codes
if (!account) {
  res.writeHead(404);  // Always 404 for not found
}

if (validationError) {
  res.writeHead(400);  // Always 400 for validation
}
```

**Why**: Consistent status codes make client error handling predictable.

---

### 5. ❌ DON'T: Include Sensitive Data in URLs

```javascript
// ❌ BAD: Sensitive data in query params
GET /api/transfer?pin=1234&amount=5000&account=123-456-7890

// Query params are:
// - Logged in server logs
// - Visible in browser history
// - Sent in referer header

// ✅ GOOD: Sensitive data in request body
POST /api/transfer
Content-Type: application/json

{
  "pin": "1234",
  "amount": 5000,
  "account": "123-456-7890"
}
```

**Why**: URL parameters are logged everywhere and visible in browser history.

---

### 6. ❌ DON'T: Parse Request Body Multiple Times

```javascript
// ❌ BAD: Parse body twice
async function handleRequest(req, res) {
  const data1 = await parseJSON(req);  // Consumes stream
  const data2 = await parseJSON(req);  // Empty! Stream already consumed
}

// ✅ GOOD: Parse once, reuse
async function handleRequest(req, res) {
  const data = await parseJSON(req);
  
  validate(data);
  process(data);
  log(data);
}
```

**Why**: Request body stream can only be read once. Second read will be empty.

---

### 7. ❌ DON'T: Trust Content-Length Header

```javascript
// ❌ BAD: Trust Content-Length
const contentLength = parseInt(req.headers['content-length']);
const buffer = Buffer.alloc(contentLength);  // Can be forged!

// ✅ GOOD: Track actual size received
let totalSize = 0;
const maxSize = 1e6;

req.on('data', (chunk) => {
  totalSize += chunk.length;
  
  if (totalSize > maxSize) {
    req.connection.destroy();
    return;
  }
});
```

**Why**: Content-Length can be forged or incorrect. Always validate actual data received.

---

### 8. ❌ DON'T: Use Synchronous Operations in Handlers

```javascript
// ❌ BAD: Blocks entire server
app.post('/api/accounts', (req, res) => {
  const data = parseBodySync(req);  // BLOCKS!
  const result = processSync(data);  // BLOCKS!
  res.end(JSON.stringify(result));
});

// ✅ GOOD: Non-blocking async
app.post('/api/accounts', async (req, res) => {
  const data = await parseBody(req);
  const result = await process(data);
  res.end(JSON.stringify(result));
});
```

**Why**: Synchronous operations block the event loop, preventing all other requests.

---

### 9. ❌ DON'T: Return Stack Traces in Production

```javascript
// ❌ BAD: Leak internal details
try {
  await processTransaction(data);
} catch (error) {
  res.writeHead(500, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({
    error: error.message,
    stack: error.stack  // Exposes internal paths!
  }));
}

// ✅ GOOD: Generic error message
try {
  await processTransaction(data);
} catch (error) {
  console.error('Transaction error:', error);  // Log internally
  
  res.writeHead(500, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({
    error: 'An internal error occurred',
    requestId: req.requestId
  }));
}
```

**Why**: Stack traces expose internal paths, dependencies, and code structure.

---

### 10. ❌ DON'T: Forget to End Response

```javascript
// ❌ BAD: Response never sent
app.get('/api/accounts', (req, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  // Forgot res.end()!
  // Client hangs waiting for response
});

// ✅ GOOD: Always end response
app.get('/api/accounts', (req, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify(accounts));
});
```

**Why**: Response must be explicitly ended, or client will timeout waiting.

---

## 🎯 Key Takeaways

### HTTP Method Summary

| Method | Safe | Idempotent | Has Body | Use Case | Status Codes |
|--------|------|------------|----------|----------|--------------|
| **GET** | ✅ | ✅ | ❌ | Retrieve resource | 200, 304, 404 |
| **POST** | ❌ | ❌ | ✅ | Create resource | 201, 400, 409 |
| **PUT** | ❌ | ✅ | ✅ | Replace entire resource | 200, 204, 404 |
| **PATCH** | ❌ | ❌ | ✅ | Partial update | 200, 400, 404 |
| **DELETE** | ❌ | ✅ | ❌ | Remove resource | 204, 404 |
| **HEAD** | ✅ | ✅ | ❌ | Get headers only | 200, 404 |
| **OPTIONS** | ✅ | ✅ | ❌ | Check allowed methods | 200, 204 |

### Common HTTP Status Codes

| Code | Name | Meaning | Use Case |
|------|------|---------|----------|
| **200** | OK | Success | Successful GET, PUT, PATCH |
| **201** | Created | Resource created | Successful POST |
| **204** | No Content | Success, no body | Successful DELETE |
| **400** | Bad Request | Invalid input | Validation failure |
| **401** | Unauthorized | Not authenticated | Missing/invalid token |
| **403** | Forbidden | Not authorized | Insufficient permissions |
| **404** | Not Found | Resource doesn't exist | Invalid resource ID |
| **409** | Conflict | Resource conflict | Duplicate email/ID |
| **415** | Unsupported Media Type | Wrong Content-Type | Expected JSON, got XML |
| **422** | Unprocessable Entity | Semantic errors | Valid format, invalid data |
| **429** | Too Many Requests | Rate limit exceeded | Too many API calls |
| **500** | Internal Server Error | Server error | Unexpected exception |
| **503** | Service Unavailable | Temporarily down | Maintenance mode |

### Content Types

| Content-Type | Usage | Parser |
|--------------|-------|--------|
| `application/json` | API requests/responses | JSON.parse() |
| `application/x-www-form-urlencoded` | HTML form submission | querystring.parse() |
| `multipart/form-data` | File uploads | Custom multipart parser |
| `text/plain` | Plain text | toString() |
| `application/xml` | XML data | XML parser |
| `application/pdf` | PDF files | Binary handling |

### Request vs Response Headers

| Header | Type | Purpose |
|--------|------|---------|
| **Content-Type** | Both | Data format |
| **Content-Length** | Both | Body size in bytes |
| **Accept** | Request | Preferred response format |
| **Authorization** | Request | Authentication credentials |
| **Cookie** | Request | Session/tracking data |
| **Set-Cookie** | Response | Set browser cookie |
| **Cache-Control** | Both | Caching behavior |
| **Location** | Response | Redirect URL |
| **ETag** | Response | Resource version |
| **Last-Modified** | Response | Last change time |

---

## 🎓 Interview Questions & Answers

### Q1: What's the difference between PUT and PATCH?

**Answer**:

**PUT** replaces the **entire resource**:
```javascript
// PUT /api/accounts/ACC-001
PUT /api/accounts/ACC-001
{
  "accountNumber": "1234567890",
  "customerName": "John Doe",
  "accountType": "SAVINGS",
  "balance": 5000,
  "status": "ACTIVE"
}

// Result: ALL fields replaced
// If you omit a field, it's removed!
```

**PATCH** updates **only specified fields**:
```javascript
// PATCH /api/accounts/ACC-001
PATCH /api/accounts/ACC-001
{
  "balance": 6000
}

// Result: Only balance updated
// All other fields unchanged
```

**Key differences**:
1. **PUT** requires sending complete resource
2. **PATCH** only sends changed fields
3. **PUT** is idempotent (same result every time)
4. **PATCH** may or may not be idempotent
5. **PUT** can create resource if doesn't exist
6. **PATCH** typically requires resource to exist

**Banking use case**:
- Use **PUT** to replace account details (all fields)
- Use **PATCH** to update account balance (one field)

---

### Q2: Why shouldn't POST requests be idempotent?

**Answer**:

**POST creates new resources**, so calling it multiple times creates multiple resources:

```javascript
// First POST
POST /api/accounts
{ "customerName": "John Doe", "accountType": "SAVINGS" }
→ Creates ACC-001

// Second POST (same data)
POST /api/accounts
{ "customerName": "John Doe", "accountType": "SAVINGS" }
→ Creates ACC-002 (different resource!)

// Not idempotent: Different result each time
```

**Compare with PUT (idempotent)**:
```javascript
// First PUT
PUT /api/accounts/ACC-001
{ "customerName": "John Doe", "accountType": "SAVINGS" }
→ Creates/updates ACC-001

// Second PUT (same data)
PUT /api/accounts/ACC-001
{ "customerName": "John Doe", "accountType": "SAVINGS" }
→ Updates ACC-001 (same resource, same result)

// Idempotent: Same result every time
```

**Why it matters in banking**:
- Retry failed POST = duplicate transaction!
- Retry failed PUT = safe, same result
- Network issues require retries
- Idempotent operations are retry-safe

**Solution for POST**:
```javascript
// Use idempotency keys
POST /api/transactions
Idempotency-Key: abc-123-xyz

// Server tracks idempotency key
// Duplicate requests with same key return original result
```

---

### Q3: How do you parse request body in vanilla Node.js?

**Answer**:

**Request body arrives in chunks** via stream:

```javascript
function parseRequestBody(req) {
  return new Promise((resolve, reject) => {
    let body = '';
    
    // Listen for data chunks
    req.on('data', (chunk) => {
      body += chunk.toString();
      
      // Prevent DoS: Limit size
      if (body.length > 1e6) {  // 1MB
        req.connection.destroy();
        reject(new Error('Payload too large'));
      }
    });
    
    // All data received
    req.on('end', () => {
      try {
        // Parse based on Content-Type
        const contentType = req.headers['content-type'];
        
        if (contentType.includes('application/json')) {
          resolve(JSON.parse(body));
        } else if (contentType.includes('application/x-www-form-urlencoded')) {
          resolve(querystring.parse(body));
        } else {
          resolve(body);  // Return as string
        }
      } catch (error) {
        reject(error);
      }
    });
    
    // Handle errors
    req.on('error', reject);
  });
}

// Usage
const data = await parseRequestBody(req);
```

**Why manual parsing is needed**:
- Node.js HTTP module doesn't parse body automatically
- Body is a stream (arrives in chunks)
- Must handle multiple Content-Types
- Must protect against large payloads

**In practice**: Use Express/Fastify with built-in body parsers.

---

### Q4: What are the different ways to pass data to an API?

**Answer**:

**1. URL Path Parameters** (resource identification):
```javascript
GET /api/accounts/ACC-12345/transactions/TXN-001
                 ↑              ↑
              accountId      transactionId

// Extract in Node.js
const accountId = url.pathname.split('/')[3];  // ACC-12345
const txnId = url.pathname.split('/')[5];      // TXN-001
```

**2. Query Parameters** (filtering, pagination, options):
```javascript
GET /api/transactions?status=completed&page=2&limit=50&sort=date
                      ↑                  ↑       ↑       ↑
                   filter           pagination  pagination  sort

// Extract in Node.js
const query = url.parse(req.url, true).query;
const status = query.status;    // 'completed'
const page = query.page;        // '2'
```

**3. Request Headers** (metadata, authentication):
```javascript
POST /api/transactions
Authorization: Bearer eyJhbGc...
Content-Type: application/json
X-Idempotency-Key: abc-123

// Extract in Node.js
const token = req.headers.authorization;
const idempotencyKey = req.headers['x-idempotency-key'];
```

**4. Request Body** (resource data):
```javascript
POST /api/transactions
Content-Type: application/json

{
  "fromAccount": "ACC-001",
  "toAccount": "ACC-002",
  "amount": 1000
}

// Extract in Node.js
const data = await parseJSON(req);
const amount = data.amount;  // 1000
```

**When to use each**:
- **Path params**: Resource IDs (required)
- **Query params**: Filters, pagination (optional)
- **Headers**: Auth, metadata, client info
- **Body**: Resource data (create/update)

**Golden rules**:
- GET requests: No body, use query params
- POST/PUT/PATCH: Use body for data
- DELETE: No body typically
- Sensitive data: Use body or headers, NEVER query params

---

### Q5: How do you implement rate limiting based on request method?

**Answer**:

Different methods have different rate limits based on their impact:

```javascript
class RateLimiter {
  constructor() {
    this.requests = new Map();  // IP -> { method -> [timestamps] }
    
    // Different limits for different methods
    this.limits = {
      'GET':    { window: 60000, max: 100 },   // 100 per minute
      'POST':   { window: 60000, max: 20 },    // 20 per minute
      'PUT':    { window: 60000, max: 20 },    // 20 per minute
      'PATCH':  { window: 60000, max: 20 },    // 20 per minute
      'DELETE': { window: 60000, max: 10 }     // 10 per minute
    };
  }
  
  checkLimit(ip, method) {
    const now = Date.now();
    const limit = this.limits[method];
    
    if (!limit) return true;  // No limit for this method
    
    // Get request history for this IP and method
    if (!this.requests.has(ip)) {
      this.requests.set(ip, {});
    }
    
    const ipRequests = this.requests.get(ip);
    
    if (!ipRequests[method]) {
      ipRequests[method] = [];
    }
    
    // Remove old timestamps outside window
    ipRequests[method] = ipRequests[method].filter(
      timestamp => now - timestamp < limit.window
    );
    
    // Check if limit exceeded
    if (ipRequests[method].length >= limit.max) {
      return false;  // Rate limit exceeded
    }
    
    // Record this request
    ipRequests[method].push(now);
    return true;
  }
  
  getRemainingRequests(ip, method) {
    const ipRequests = this.requests.get(ip)?.[method] || [];
    const limit = this.limits[method];
    return limit ? limit.max - ipRequests.length : Infinity;
  }
}

// Usage
const rateLimiter = new RateLimiter();

function handleRequest(req, res) {
  const ip = req.connection.remoteAddress;
  const method = req.method;
  
  if (!rateLimiter.checkLimit(ip, method)) {
    res.writeHead(429, {
      'Content-Type': 'application/json',
      'Retry-After': '60'
    });
    res.end(JSON.stringify({
      error: 'Rate limit exceeded',
      message: `Too many ${method} requests. Please try again later.`
    }));
    return;
  }
  
  // Add rate limit headers
  const remaining = rateLimiter.getRemainingRequests(ip, method);
  res.setHeader('X-RateLimit-Limit', rateLimiter.limits[method].max);
  res.setHeader('X-RateLimit-Remaining', remaining);
  
  // Process request...
}
```

**Why different limits**:
- **GET** (read): Least expensive, highest limit
- **POST** (create): Creates data, moderate limit
- **PUT/PATCH** (update): Modifies data, moderate limit
- **DELETE** (delete): Most dangerous, lowest limit

**Banking considerations**:
- Transaction creation (POST): Strict limits
- Balance inquiry (GET): Higher limits
- Account deletion (DELETE): Very strict limits
- Failed auth attempts: Separate, very strict limits

---

### Q6: How do you handle HTTP method overriding?

**Answer**:

Some clients (old browsers, HTML forms) only support GET and POST. Method overriding allows tunneling other methods through POST:

**Method 1: X-HTTP-Method-Override header**:
```javascript
function getActualMethod(req) {
  const override = req.headers['x-http-method-override'];
  
  // Only allow override for POST requests
  if (req.method === 'POST' && override) {
    const allowedOverrides = ['PUT', 'PATCH', 'DELETE'];
    if (allowedOverrides.includes(override.toUpperCase())) {
      return override.toUpperCase();
    }
  }
  
  return req.method;
}

// Usage
const actualMethod = getActualMethod(req);

if (actualMethod === 'DELETE') {
  // Handle DELETE even though actual HTTP method was POST
  deleteResource(req, res);
}
```

**Method 2: _method query parameter**:
```javascript
// HTML form (only supports GET/POST)
<form action="/api/accounts/ACC-001?_method=DELETE" method="POST">
  <button type="submit">Delete Account</button>
</form>

// Server
function getActualMethod(req) {
  const parsedUrl = url.parse(req.url, true);
  const methodParam = parsedUrl.query._method;
  
  if (req.method === 'POST' && methodParam) {
    const allowedOverrides = ['PUT', 'PATCH', 'DELETE'];
    if (allowedOverrides.includes(methodParam.toUpperCase())) {
      return methodParam.toUpperCase();
    }
  }
  
  return req.method;
}
```

**Security considerations**:
1. Only allow override for POST (never GET)
2. Whitelist allowed methods (PUT, PATCH, DELETE)
3. Validate that override makes sense for endpoint
4. Log method overrides for auditing

**When to use**:
- Supporting legacy browsers
- HTML forms without JavaScript
- Compatibility with proxy servers
- REST APIs accessed via limited clients

**Modern approach**: Use JavaScript fetch/XMLHttpRequest which support all methods:
```javascript
// Modern client - no override needed
fetch('/api/accounts/ACC-001', {
  method: 'DELETE'  // Native DELETE support
});
```

---

### Q7: How do you validate Content-Type for security?

**Answer**:

**Content-Type validation prevents attacks and parsing errors**:

```javascript
class ContentTypeValidator {
  constructor() {
    // Allowed Content-Types for each endpoint
    this.allowedTypes = {
      '/api/accounts': ['application/json'],
      '/api/documents/upload': ['multipart/form-data'],
      '/api/forms/contact': ['application/x-www-form-urlencoded', 'application/json']
    };
  }
  
  validate(req, endpoint) {
    const contentType = req.headers['content-type'];
    
    // Methods without body don't need Content-Type
    if (['GET', 'DELETE', 'HEAD', 'OPTIONS'].includes(req.method)) {
      return { valid: true };
    }
    
    // POST, PUT, PATCH require Content-Type
    if (!contentType) {
      return {
        valid: false,
        error: 'Content-Type header is required',
        statusCode: 400
      };
    }
    
    // Check against allowed types
    const allowed = this.allowedTypes[endpoint] || [];
    const isAllowed = allowed.some(type => contentType.includes(type));
    
    if (!isAllowed) {
      return {
        valid: false,
        error: `Invalid Content-Type. Allowed: ${allowed.join(', ')}`,
        statusCode: 415  // Unsupported Media Type
      };
    }
    
    // Additional security checks
    
    // 1. Check for charset injection
    const charsetMatch = contentType.match(/charset=([^;]+)/);
    if (charsetMatch) {
      const charset = charsetMatch[1].toLowerCase();
      const allowedCharsets = ['utf-8', 'utf8', 'iso-8859-1'];
      
      if (!allowedCharsets.includes(charset)) {
        return {
          valid: false,
          error: `Invalid charset. Allowed: ${allowedCharsets.join(', ')}`,
          statusCode: 400
        };
      }
    }
    
    // 2. Check for multiple Content-Type headers (header injection)
    const contentTypeHeaders = Object.keys(req.headers).filter(
      h => h.toLowerCase() === 'content-type'
    );
    
    if (contentTypeHeaders.length > 1) {
      return {
        valid: false,
        error: 'Multiple Content-Type headers not allowed',
        statusCode: 400
      };
    }
    
    return { valid: true };
  }
}

// Usage
const validator = new ContentTypeValidator();

function handleRequest(req, res) {
  const parsedUrl = url.parse(req.url);
  const validation = validator.validate(req, parsedUrl.pathname);
  
  if (!validation.valid) {
    res.writeHead(validation.statusCode, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
      error: validation.error
    }));
    return;
  }
  
  // Process request...
}
```

**Common attacks prevented**:
1. **Content-Type mismatch**: Send XML when expecting JSON
2. **Charset injection**: Use unusual charset to bypass filtering
3. **Header injection**: Multiple Content-Type headers
4. **MIME confusion**: Browser interprets data differently than server

**Banking security requirements**:
- Always validate Content-Type
- Never trust client's declared type
- Verify actual content matches Content-Type
- Reject unexpected types immediately

---

### Q8: What's the proper way to handle file uploads?

**Answer**:

**File uploads require multipart/form-data parsing with security checks**:

```javascript
async function handleFileUpload(req, res) {
  try {
    // 1. Validate Content-Type
    const contentType = req.headers['content-type'];
    if (!contentType || !contentType.includes('multipart/form-data')) {
      return sendError(res, 415, 'Must be multipart/form-data');
    }
    
    // 2. Parse multipart with size limit
    const { fields, files } = await parseMultipart(req, 10 * 1024 * 1024);  // 10MB max
    
    // 3. Validate file exists
    if (!files || files.length === 0) {
      return sendError(res, 400, 'No file uploaded');
    }
    
    const file = files[0];
    
    // 4. Validate file size
    const maxSize = 5 * 1024 * 1024;  // 5MB
    if (file.size > maxSize) {
      return sendError(res, 413, `File too large. Max: ${maxSize / 1024 / 1024}MB`);
    }
    
    // 5. Validate file type by Content-Type
    const allowedTypes = [
      'application/pdf',
      'image/jpeg',
      'image/png',
      'image/gif'
    ];
    
    if (!allowedTypes.includes(file.contentType)) {
      return sendError(res, 400, `Invalid file type. Allowed: ${allowedTypes.join(', ')}`);
    }
    
    // 6. Validate file extension
    const allowedExtensions = ['.pdf', '.jpg', '.jpeg', '.png', '.gif'];
    const ext = path.extname(file.fileName).toLowerCase();
    
    if (!allowedExtensions.includes(ext)) {
      return sendError(res, 400, `Invalid file extension. Allowed: ${allowedExtensions.join(', ')}`);
    }
    
    // 7. Validate file signature (magic bytes)
    const isValid = await validateFileSignature(file.data, file.contentType);
    if (!isValid) {
      return sendError(res, 400, 'File content does not match declared type');
    }
    
    // 8. Sanitize filename
    const safeFilename = sanitizeFilename(file.fileName);
    
    // 9. Generate unique filename
    const uniqueFilename = `${crypto.randomUUID()}-${safeFilename}`;
    
    // 10. Scan for malware (in production)
    // await scanForMalware(file.data);
    
    // 11. Save file securely
    const uploadDir = path.join(__dirname, 'uploads');
    await mkdir(uploadDir, { recursive: true });
    
    const filePath = path.join(uploadDir, uniqueFilename);
    await writeFile(filePath, file.data);
    
    // 12. Store metadata in database
    const upload = {
      id: crypto.randomUUID(),
      originalName: file.fileName,
      storedName: uniqueFilename,
      size: file.size,
      contentType: file.contentType,
      uploadedAt: new Date().toISOString(),
      uploadedBy: fields.userId
    };
    
    res.writeHead(201, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
      message: 'File uploaded successfully',
      data: upload
    }));
    
  } catch (error) {
    console.error('Upload error:', error);
    sendError(res, 500, 'Upload failed');
  }
}

function sanitizeFilename(filename) {
  // Remove path traversal attempts
  return filename
    .replace(/\.\./g, '')
    .replace(/[\/\\]/g, '')
    .replace(/[^a-zA-Z0-9.-]/g, '_')
    .substring(0, 255);  // Limit length
}

async function validateFileSignature(data, contentType) {
  // Check magic bytes (file signature)
  const signatures = {
    'application/pdf': [0x25, 0x50, 0x44, 0x46],  // %PDF
    'image/jpeg': [0xFF, 0xD8, 0xFF],             // JPEG
    'image/png': [0x89, 0x50, 0x4E, 0x47]         // PNG
  };
  
  const signature = signatures[contentType];
  if (!signature) return true;  // Unknown type, skip check
  
  for (let i = 0; i < signature.length; i++) {
    if (data[i] !== signature[i]) {
      return false;
    }
  }
  
  return true;
}
```

**Security checklist**:
- ✅ Validate Content-Type header
- ✅ Enforce file size limits
- ✅ Whitelist allowed file types
- ✅ Validate file extension
- ✅ Check magic bytes (file signature)
- ✅ Sanitize filename
- ✅ Use unique filenames
- ✅ Store outside web root
- ✅ Scan for malware
- ✅ Set proper file permissions

**Banking use cases**:
- ID document uploads (passport, driver's license)
- Income verification (pay stubs, tax returns)
- Bank statements
- Signature cards

---

### Q9: How do you implement proper CORS for a banking API?

**Answer**:

**CORS must be restrictive for banking applications**:

```javascript
class CORSHandler {
  constructor() {
    // Whitelist of allowed origins
    this.allowedOrigins = [
      'https://banking.example.com',
      'https://mobile.example.com',
      'https://admin.example.com'
    ];
    
    // Allow additional origins in development
    if (process.env.NODE_ENV === 'development') {
      this.allowedOrigins.push('http://localhost:3000');
      this.allowedOrigins.push('http://localhost:4200');
    }
  }
  
  handleCORS(req, res) {
    const origin = req.headers.origin;
    
    // 1. Validate origin
    if (this.allowedOrigins.includes(origin)) {
      res.setHeader('Access-Control-Allow-Origin', origin);
    } else {
      // Don't set CORS headers for non-whitelisted origins
      // This blocks the browser from reading the response
      console.warn(`Blocked CORS request from: ${origin}`);
    }
    
    // 2. Allow credentials (cookies, auth headers)
    res.setHeader('Access-Control-Allow-Credentials', 'true');
    
    // 3. Specify allowed methods
    res.setHeader(
      'Access-Control-Allow-Methods',
      'GET, POST, PUT, PATCH, DELETE, OPTIONS'
    );
    
    // 4. Specify allowed headers
    res.setHeader(
      'Access-Control-Allow-Headers',
      'Content-Type, Authorization, X-Request-ID, X-Idempotency-Key'
    );
    
    // 5. Specify exposed headers (readable by client)
    res.setHeader(
      'Access-Control-Expose-Headers',
      'X-Request-ID, X-RateLimit-Remaining'
    );
    
    // 6. Cache preflight for 1 hour
    res.setHeader('Access-Control-Max-Age', '3600');
    
    // 7. Handle preflight requests
    if (req.method === 'OPTIONS') {
      res.writeHead(204);  // No Content
      res.end();
      return true;  // Indicate preflight was handled
    }
    
    return false;  // Continue to actual handler
  }
  
  // Method-specific CORS
  handleMethodSpecificCORS(req, res, method) {
    const origin = req.headers.origin;
    
    // More restrictive CORS for sensitive operations
    if (['DELETE', 'PATCH'].includes(method)) {
      const adminOrigins = [
        'https://admin.example.com'
      ];
      
      if (adminOrigins.includes(origin)) {
        res.setHeader('Access-Control-Allow-Origin', origin);
        res.setHeader('Access-Control-Allow-Credentials', 'true');
      } else {
        // Block non-admin origins from DELETE/PATCH
        return false;
      }
    }
    
    return true;
  }
}

// Usage
const corsHandler = new CORSHandler();

function handleRequest(req, res) {
  // Handle CORS
  const isPrelight = corsHandler.handleCORS(req, res);
  if (isPreflight) return;  // Preflight handled
  
  // Check method-specific CORS
  if (!corsHandler.handleMethodSpecificCORS(req, res, req.method)) {
    res.writeHead(403, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ error: 'CORS policy violation' }));
    return;
  }
  
  // Process request...
}
```

**CORS Security Rules for Banking**:

1. **Never use `Access-Control-Allow-Origin: *`**
   - Always whitelist specific origins
   - Especially when using credentials

2. **Validate origin on every request**
   - Don't trust the Origin header blindly
   - Log rejected origins for security monitoring

3. **Restrict sensitive operations**
   - DELETE/PATCH only from admin origins
   - Transaction creation only from main app

4. **Short preflight cache**
   - Max 1 hour (3600 seconds)
   - Allows policy updates to take effect quickly

5. **Minimal exposed headers**
   - Only expose necessary headers
   - Don't expose sensitive server information

**Example attack prevented**:
```javascript
// Attacker's malicious site
fetch('https://banking-api.com/api/accounts/transfer', {
  method: 'POST',
  credentials: 'include',  // Include victim's cookies
  body: JSON.stringify({
    toAccount: 'ATTACKER-ACCOUNT',
    amount: 10000
  })
});

// Blocked by CORS because:
// 1. Origin not in whitelist
// 2. Server doesn't send Access-Control-Allow-Origin header
// 3. Browser blocks response from being read
```

---

### Q10: How do you design a RESTful API following best practices?

**Answer**:

**RESTful API design principles for banking**:

```
Banking RESTful API Structure
══════════════════════════════════════════════════════════════

Base URL: https://api.banking.example.com/v1

ACCOUNTS
────────────────────────────────────────────────────────────
GET    /accounts                    List all accounts
POST   /accounts                    Create new account
GET    /accounts/:id                Get specific account
PUT    /accounts/:id                Replace account (full update)
PATCH  /accounts/:id                Update account (partial)
DELETE /accounts/:id                Close account

TRANSACTIONS
────────────────────────────────────────────────────────────
GET    /accounts/:id/transactions   List account transactions
POST   /accounts/:id/transactions   Create transaction
GET    /transactions/:id            Get transaction details
PATCH  /transactions/:id            Update transaction status

TRANSFERS
────────────────────────────────────────────────────────────
POST   /transfers                   Initiate transfer
GET    /transfers/:id               Get transfer status
DELETE /transfers/:id               Cancel pending transfer

BENEFICIARIES
────────────────────────────────────────────────────────────
GET    /accounts/:id/beneficiaries  List beneficiaries
POST   /accounts/:id/beneficiaries  Add beneficiary
DELETE /beneficiaries/:id           Remove beneficiary

CARDS
────────────────────────────────────────────────────────────
GET    /accounts/:id/cards          List account cards
POST   /accounts/:id/cards          Issue new card
PATCH  /cards/:id                   Update card (block/unblock)
DELETE /cards/:id                   Cancel card

STATEMENTS
────────────────────────────────────────────────────────────
GET    /accounts/:id/statements     List statements
GET    /statements/:id              Download statement (PDF)
```

**RESTful Design Principles**:

**1. Resource-Oriented URLs**:
```javascript
// ✅ GOOD: Noun-based, hierarchical
GET  /accounts/ACC-001
GET  /accounts/ACC-001/transactions
POST /accounts/ACC-001/transactions

// ❌ BAD: Verb-based, flat
GET  /getAccount?id=ACC-001
GET  /getTransactionsForAccount?id=ACC-001
POST /createTransaction
```

**2. HTTP Methods for Actions**:
```javascript
// ✅ GOOD: Method indicates action
POST   /accounts          // Create
GET    /accounts          // Read list
GET    /accounts/:id      // Read one
PUT    /accounts/:id      // Replace
PATCH  /accounts/:id      // Update
DELETE /accounts/:id      // Delete

// ❌ BAD: Action in URL
POST /accounts/create
POST /accounts/update
POST /accounts/delete
```

**3. Proper Status Codes**:
```javascript
// Success
200 OK                    // Successful GET, PUT, PATCH
201 Created               // Successful POST
204 No Content            // Successful DELETE

// Client Errors
400 Bad Request           // Invalid input
401 Unauthorized          // Not authenticated
403 Forbidden             // Not authorized
404 Not Found             // Resource doesn't exist
409 Conflict              // Duplicate resource
422 Unprocessable Entity  // Validation failed

// Server Errors
500 Internal Server Error // Unexpected error
503 Service Unavailable   // Maintenance/overload
```

**4. Consistent Response Format**:
```javascript
// Success response
{
  "data": {
    "id": "ACC-001",
    "accountNumber": "1234567890",
    "balance": 5000
  },
  "metadata": {
    "timestamp": "2025-11-15T10:30:00Z",
    "requestId": "req-abc-123"
  }
}

// Error response
{
  "error": {
    "message": "Account not found",
    "statusCode": 404,
    "code": "ACCOUNT_NOT_FOUND",
    "timestamp": "2025-11-15T10:30:00Z",
    "requestId": "req-abc-123"
  }
}

// List response
{
  "data": [ /* array of resources */ ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 100,
    "totalPages": 10
  },
  "metadata": {
    "timestamp": "2025-11-15T10:30:00Z"
  }
}
```

**5. Versioning**:
```javascript
// URL versioning (recommended for banking)
https://api.banking.example.com/v1/accounts
https://api.banking.example.com/v2/accounts

// Header versioning (alternative)
Accept: application/vnd.banking.v1+json
```

**6. Filtering, Sorting, Pagination**:
```javascript
// Filtering
GET /transactions?status=completed&type=TRANSFER&minAmount=1000

// Sorting
GET /transactions?sort=-date,+amount  // -date = descending, +amount = ascending

// Pagination
GET /transactions?page=2&limit=50

// Search
GET /transactions?search=john+doe
```

**7. HATEOAS (Hypermedia)**:
```javascript
{
  "data": {
    "id": "ACC-001",
    "balance": 5000
  },
  "links": {
    "self": "/accounts/ACC-001",
    "transactions": "/accounts/ACC-001/transactions",
    "statements": "/accounts/ACC-001/statements",
    "transfer": "/transfers"
  }
}
```

**8. Rate Limiting Headers**:
```javascript
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 75
X-RateLimit-Reset: 1700000000
```

**9. Idempotency Keys**:
```javascript
POST /transfers
Idempotency-Key: transfer-abc-123-xyz

// Server tracks key, returns same result for duplicate requests
```

**10. Caching Headers**:
```javascript
// Cacheable (GET requests)
Cache-Control: private, max-age=300  // Cache for 5 minutes
ETag: "abc123"

// Not cacheable (sensitive data)
Cache-Control: no-store, no-cache, must-revalidate
```

**Complete Example**:
```javascript
// Request
GET /v1/accounts/ACC-001/transactions?status=completed&page=1&limit=10
Authorization: Bearer eyJhbGc...
Accept: application/json

// Response
HTTP/1.1 200 OK
Content-Type: application/json
X-Request-ID: req-abc-123
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
Cache-Control: private, max-age=60

{
  "data": [
    {
      "id": "TXN-001",
      "type": "TRANSFER",
      "amount": 1000,
      "status": "completed",
      "date": "2025-11-15T10:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 45,
    "totalPages": 5
  },
  "links": {
    "self": "/accounts/ACC-001/transactions?page=1",
    "next": "/accounts/ACC-001/transactions?page=2",
    "last": "/accounts/ACC-001/transactions?page=5"
  },
  "metadata": {
    "timestamp": "2025-11-15T10:30:00Z",
    "requestId": "req-abc-123"
  }
}
```

---

## 🔧 Banking HTTP Operations Checklist

### Request Handling
- [ ] Parse request method correctly
- [ ] Validate Content-Type header
- [ ] Parse request body with size limits
- [ ] Extract URL parameters (path, query)
- [ ] Read and validate headers
- [ ] Implement request timeout
- [ ] Add request ID for tracing
- [ ] Log all incoming requests

### Response Handling
- [ ] Use appropriate HTTP status codes
- [ ] Set correct Content-Type header
- [ ] Add security headers (X-Frame-Options, etc.)
- [ ] Include request ID in response
- [ ] Add CORS headers (if needed)
- [ ] Set rate limit headers
- [ ] Always end response (res.end())
- [ ] Handle response errors gracefully

### Method-Specific
- [ ] GET: No request body, use query params
- [ ] POST: Create resources, return 201
- [ ] PUT: Replace entire resource, idempotent
- [ ] PATCH: Partial update, return changed resource
- [ ] DELETE: Return 204 No Content
- [ ] OPTIONS: Handle preflight requests

### Security
- [ ] Validate all inputs
- [ ] Sanitize user data
- [ ] Implement rate limiting
- [ ] Check authentication/authorization
- [ ] Validate Content-Type
- [ ] Enforce HTTPS
- [ ] Add security headers
- [ ] Implement CORS properly
- [ ] Use idempotency keys
- [ ] Log security events

### Performance
- [ ] Implement caching headers
- [ ] Add ETag support
- [ ] Use compression (gzip)
- [ ] Implement pagination
- [ ] Optimize query parameters
- [ ] Monitor response times
- [ ] Use connection pooling

---

## 🎓 Summary

**HTTP request methods** are fundamental to building RESTful banking APIs:

1. **Method Semantics**: Use GET (read), POST (create), PUT (replace), PATCH (update), DELETE (remove)
2. **Request Body Parsing**: Handle JSON, URL-encoded, and multipart data
3. **Status Codes**: Return appropriate codes (200, 201, 400, 404, 500)
4. **Content Negotiation**: Support multiple content types
5. **Validation**: Always validate inputs before processing
6. **Security**: Implement size limits, CORS, rate limiting
7. **Idempotency**: Design safe retry mechanisms
8. **RESTful Design**: Resource-oriented URLs with proper HTTP methods
9. **Error Handling**: Consistent error response format
10. **Headers**: Use headers for metadata, auth, caching

**Remember**: In banking, HTTP methods directly impact transaction integrity, security, and regulatory compliance. Always follow REST principles, validate all inputs, and implement proper error handling.

---

## 📚 Additional Resources

- [HTTP/1.1 Specification (RFC 7231)](https://tools.ietf.org/html/rfc7231)
- [REST API Tutorial](https://restfulapi.net/)
- [HTTP Status Codes Reference](https://httpstatuses.com/)
- [Express.js Body Parser](https://expressjs.com/en/api.html#req.body)
- [CORS Explained](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [RESTful API Design Best Practices](https://restfulapi.net/rest-api-design-tutorial-with-example/)

---

**End of Question 14: HTTP Request Methods** 🎉

**Total Content**: ~2,800+ lines of comprehensive HTTP request methods content with production-ready banking examples!



