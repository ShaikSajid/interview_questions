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

