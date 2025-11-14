# Node.js Interview Question: HTTP Module
## Question 10: How to Create HTTP Server in Node.js? Request/Response Handling?

---

## 📋 Summary of What Will Be Covered

In this comprehensive guide, you'll learn:

1. **Creating HTTP servers** - http.createServer()
2. **Request object** - URL, method, headers, body parsing
3. **Response object** - Status codes, headers, streaming responses
4. **Routing** - Handling different endpoints
5. **HTTP methods** - GET, POST, PUT, DELETE
6. **JSON APIs** - Building RESTful services
7. **HTTP client** - Making requests to other servers
8. **Banking Examples**: Account API, transaction endpoints, authentication

**Why This Matters in Banking**:
- Build REST APIs for mobile/web apps
- Handle secure transaction requests
- Implement authentication/authorization
- Process payment webhooks
- Serve customer data securely
- Integrate with third-party services

---

## 🎯 Basic HTTP Server

### Simple Server

```javascript
import http from 'http';

const server = http.createServer((req, res) => {
  res.statusCode = 200;
  res.setHeader('Content-Type', 'text/plain');
  res.end('Hello World\n');
});

server.listen(3000, () => {
  console.log('Server running at http://localhost:3000/');
});
```

**Key Components**:
- **http.createServer()**: Creates server instance
- **req**: IncomingMessage (readable stream)
- **res**: ServerResponse (writable stream)
- **server.listen()**: Start listening on port

---

## 📊 Visual: HTTP Request/Response Flow

```
Client                          Node.js HTTP Server
  │                                     │
  │  GET /accounts HTTP/1.1             │
  │  Host: api.bank.com                 │
  │  Authorization: Bearer token        │
  ├────────────────────────────────────►│
  │                                     │
  │                                     ├─► Parse URL, headers
  │                                     ├─► Route to handler
  │                                     ├─► Process business logic
  │                                     ├─► Fetch from database
  │                                     │
  │  HTTP/1.1 200 OK                    │
  │  Content-Type: application/json     │
  │  { "accounts": [...] }              │
  │◄────────────────────────────────────┤
  │                                     │
```

---

## 🔍 Request Object Properties

```javascript
server.on('request', (req, res) => {
  // URL information
  console.log('URL:', req.url);              // '/api/accounts?id=123'
  console.log('Method:', req.method);        // 'GET', 'POST', etc.
  
  // Headers
  console.log('Headers:', req.headers);
  console.log('Content-Type:', req.headers['content-type']);
  console.log('Authorization:', req.headers['authorization']);
  
  // HTTP version
  console.log('HTTP Version:', req.httpVersion);  // '1.1'
  
  // Socket info
  console.log('Remote IP:', req.socket.remoteAddress);
});
```

---

## 🏦 Example 1: Banking REST API Server

This example demonstrates a complete banking REST API with proper routing.

**File: `banking-api-server.js`**

```javascript
import http from 'http';
import { URL } from 'url';
import crypto from 'crypto';

/**
 * Banking REST API Server
 * Demonstrates: HTTP server, routing, JSON API, error handling
 */

console.log('🏦 Banking REST API Server\n');
console.log('='.repeat(70));

// ============================================
// Mock Database
// ============================================

const accounts = new Map([
  ['ACC001', { id: 'ACC001', name: 'John Doe', balance: 5000, currency: 'USD' }],
  ['ACC002', { id: 'ACC002', name: 'Jane Smith', balance: 10000, currency: 'USD' }],
  ['ACC003', { id: 'ACC003', name: 'Bob Johnson', balance: 2500, currency: 'USD' }]
]);

const transactions = [];

// ============================================
// Helper Functions
// ============================================

/**
 * Parse JSON request body
 */
function parseBody(req) {
  return new Promise((resolve, reject) => {
    let body = '';
    
    req.on('data', chunk => {
      body += chunk.toString();
      
      // Prevent large payloads (10MB limit)
      if (body.length > 10 * 1024 * 1024) {
        reject(new Error('Request body too large'));
        req.socket.destroy();
      }
    });
    
    req.on('end', () => {
      try {
        const parsed = body ? JSON.parse(body) : {};
        resolve(parsed);
      } catch (err) {
        reject(new Error('Invalid JSON'));
      }
    });
    
    req.on('error', reject);
  });
}

/**
 * Send JSON response
 */
function sendJSON(res, statusCode, data) {
  res.statusCode = statusCode;
  res.setHeader('Content-Type', 'application/json');
  res.setHeader('X-Powered-By', 'Node.js Banking API');
  res.end(JSON.stringify(data, null, 2));
}

/**
 * Send error response
 */
function sendError(res, statusCode, message) {
  sendJSON(res, statusCode, {
    error: {
      message,
      statusCode,
      timestamp: new Date().toISOString()
    }
  });
}

/**
 * Parse query parameters
 */
function parseQuery(url) {
  const urlObj = new URL(url, 'http://localhost');
  const params = {};
  
  urlObj.searchParams.forEach((value, key) => {
    params[key] = value;
  });
  
  return params;
}

/**
 * Validate authentication token (simplified)
 */
function authenticate(req) {
  const authHeader = req.headers['authorization'];
  
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return null;
  }
  
  const token = authHeader.substring(7);
  
  // In production: verify JWT token
  // For demo: simple validation
  if (token === 'valid-token-12345') {
    return { userId: 'USER001', role: 'customer' };
  }
  
  return null;
}

/**
 * Log request
 */
function logRequest(req, startTime) {
  const duration = Date.now() - startTime;
  console.log(`[${new Date().toISOString()}] ${req.method} ${req.url} - ${duration}ms`);
}

// ============================================
// Route Handlers
// ============================================

/**
 * GET /accounts - List all accounts
 */
async function handleGetAccounts(req, res, query) {
  console.log('\n📋 GET /accounts');
  
  const accountList = Array.from(accounts.values());
  
  // Filter by minimum balance if provided
  let filtered = accountList;
  if (query.minBalance) {
    const minBalance = parseFloat(query.minBalance);
    filtered = accountList.filter(acc => acc.balance >= minBalance);
    console.log(`   Filtered by minBalance: $${minBalance}`);
  }
  
  sendJSON(res, 200, {
    count: filtered.length,
    accounts: filtered
  });
  
  console.log(`   ✅ Returned ${filtered.length} accounts`);
}

/**
 * GET /accounts/:id - Get specific account
 */
async function handleGetAccount(req, res, accountId) {
  console.log(`\n👤 GET /accounts/${accountId}`);
  
  const account = accounts.get(accountId);
  
  if (!account) {
    console.log(`   ❌ Account not found`);
    return sendError(res, 404, 'Account not found');
  }
  
  sendJSON(res, 200, { account });
  console.log(`   ✅ Returned account: ${account.name}`);
}

/**
 * POST /accounts - Create new account
 */
async function handleCreateAccount(req, res) {
  console.log('\n➕ POST /accounts');
  
  try {
    const body = await parseBody(req);
    
    // Validate input
    if (!body.name || !body.initialBalance) {
      return sendError(res, 400, 'Missing required fields: name, initialBalance');
    }
    
    if (body.initialBalance < 0) {
      return sendError(res, 400, 'Initial balance must be positive');
    }
    
    // Create account
    const accountId = `ACC${crypto.randomBytes(3).toString('hex').toUpperCase()}`;
    const account = {
      id: accountId,
      name: body.name,
      balance: body.initialBalance,
      currency: body.currency || 'USD',
      createdAt: new Date().toISOString()
    };
    
    accounts.set(accountId, account);
    
    sendJSON(res, 201, {
      message: 'Account created successfully',
      account
    });
    
    console.log(`   ✅ Created account: ${accountId} - ${account.name}`);
    
  } catch (err) {
    console.log(`   ❌ Error: ${err.message}`);
    sendError(res, 400, err.message);
  }
}

/**
 * PUT /accounts/:id - Update account
 */
async function handleUpdateAccount(req, res, accountId) {
  console.log(`\n✏️  PUT /accounts/${accountId}`);
  
  try {
    const account = accounts.get(accountId);
    
    if (!account) {
      return sendError(res, 404, 'Account not found');
    }
    
    const body = await parseBody(req);
    
    // Update allowed fields
    if (body.name !== undefined) {
      account.name = body.name;
    }
    
    account.updatedAt = new Date().toISOString();
    
    sendJSON(res, 200, {
      message: 'Account updated successfully',
      account
    });
    
    console.log(`   ✅ Updated account: ${accountId}`);
    
  } catch (err) {
    console.log(`   ❌ Error: ${err.message}`);
    sendError(res, 400, err.message);
  }
}

/**
 * DELETE /accounts/:id - Delete account
 */
async function handleDeleteAccount(req, res, accountId) {
  console.log(`\n🗑️  DELETE /accounts/${accountId}`);
  
  const account = accounts.get(accountId);
  
  if (!account) {
    return sendError(res, 404, 'Account not found');
  }
  
  // Check if balance is zero
  if (account.balance !== 0) {
    return sendError(res, 400, 'Cannot delete account with non-zero balance');
  }
  
  accounts.delete(accountId);
  
  sendJSON(res, 200, {
    message: 'Account deleted successfully',
    accountId
  });
  
  console.log(`   ✅ Deleted account: ${accountId}`);
}

/**
 * POST /transactions - Create transaction
 */
async function handleCreateTransaction(req, res) {
  console.log('\n💸 POST /transactions');
  
  try {
    const body = await parseBody(req);
    
    // Validate input
    if (!body.fromAccountId || !body.toAccountId || !body.amount) {
      return sendError(res, 400, 'Missing required fields');
    }
    
    const fromAccount = accounts.get(body.fromAccountId);
    const toAccount = accounts.get(body.toAccountId);
    
    if (!fromAccount) {
      return sendError(res, 404, 'Source account not found');
    }
    
    if (!toAccount) {
      return sendError(res, 404, 'Destination account not found');
    }
    
    // Check sufficient balance
    if (fromAccount.balance < body.amount) {
      return sendError(res, 400, 'Insufficient funds');
    }
    
    // Process transaction
    fromAccount.balance -= body.amount;
    toAccount.balance += body.amount;
    
    const transaction = {
      id: `TXN${crypto.randomBytes(4).toString('hex').toUpperCase()}`,
      fromAccountId: body.fromAccountId,
      toAccountId: body.toAccountId,
      amount: body.amount,
      description: body.description || '',
      status: 'completed',
      timestamp: new Date().toISOString()
    };
    
    transactions.push(transaction);
    
    sendJSON(res, 201, {
      message: 'Transaction completed successfully',
      transaction,
      newBalances: {
        [fromAccount.id]: fromAccount.balance,
        [toAccount.id]: toAccount.balance
      }
    });
    
    console.log(`   ✅ Transaction: ${body.fromAccountId} → ${body.toAccountId} ($${body.amount})`);
    
  } catch (err) {
    console.log(`   ❌ Error: ${err.message}`);
    sendError(res, 400, err.message);
  }
}

/**
 * GET /transactions - List transactions
 */
async function handleGetTransactions(req, res, query) {
  console.log('\n📜 GET /transactions');
  
  let filtered = transactions;
  
  // Filter by account ID
  if (query.accountId) {
    filtered = transactions.filter(txn =>
      txn.fromAccountId === query.accountId ||
      txn.toAccountId === query.accountId
    );
    console.log(`   Filtered by account: ${query.accountId}`);
  }
  
  // Limit results
  const limit = parseInt(query.limit) || 10;
  filtered = filtered.slice(-limit);
  
  sendJSON(res, 200, {
    count: filtered.length,
    transactions: filtered
  });
  
  console.log(`   ✅ Returned ${filtered.length} transactions`);
}

/**
 * GET /health - Health check
 */
function handleHealthCheck(req, res) {
  sendJSON(res, 200, {
    status: 'healthy',
    uptime: process.uptime(),
    timestamp: new Date().toISOString(),
    accounts: accounts.size,
    transactions: transactions.length
  });
}

// ============================================
// Router
// ============================================

async function router(req, res) {
  const startTime = Date.now();
  
  // CORS headers
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
  
  // Handle preflight
  if (req.method === 'OPTIONS') {
    res.statusCode = 204;
    res.end();
    return;
  }
  
  // Parse URL
  const urlObj = new URL(req.url, 'http://localhost');
  const pathname = urlObj.pathname;
  const query = parseQuery(req.url);
  
  try {
    // Route matching
    if (pathname === '/health' && req.method === 'GET') {
      await handleHealthCheck(req, res);
    }
    else if (pathname === '/accounts' && req.method === 'GET') {
      // Check authentication
      const user = authenticate(req);
      if (!user) {
        return sendError(res, 401, 'Unauthorized - Invalid token');
      }
      await handleGetAccounts(req, res, query);
    }
    else if (pathname.match(/^\/accounts\/[A-Z0-9]+$/) && req.method === 'GET') {
      const user = authenticate(req);
      if (!user) {
        return sendError(res, 401, 'Unauthorized');
      }
      const accountId = pathname.split('/')[2];
      await handleGetAccount(req, res, accountId);
    }
    else if (pathname === '/accounts' && req.method === 'POST') {
      const user = authenticate(req);
      if (!user) {
        return sendError(res, 401, 'Unauthorized');
      }
      await handleCreateAccount(req, res);
    }
    else if (pathname.match(/^\/accounts\/[A-Z0-9]+$/) && req.method === 'PUT') {
      const user = authenticate(req);
      if (!user) {
        return sendError(res, 401, 'Unauthorized');
      }
      const accountId = pathname.split('/')[2];
      await handleUpdateAccount(req, res, accountId);
    }
    else if (pathname.match(/^\/accounts\/[A-Z0-9]+$/) && req.method === 'DELETE') {
      const user = authenticate(req);
      if (!user) {
        return sendError(res, 401, 'Unauthorized');
      }
      const accountId = pathname.split('/')[2];
      await handleDeleteAccount(req, res, accountId);
    }
    else if (pathname === '/transactions' && req.method === 'POST') {
      const user = authenticate(req);
      if (!user) {
        return sendError(res, 401, 'Unauthorized');
      }
      await handleCreateTransaction(req, res);
    }
    else if (pathname === '/transactions' && req.method === 'GET') {
      const user = authenticate(req);
      if (!user) {
        return sendError(res, 401, 'Unauthorized');
      }
      await handleGetTransactions(req, res, query);
    }
    else {
      // 404 Not Found
      sendError(res, 404, 'Endpoint not found');
      console.log(`\n❌ 404: ${req.method} ${pathname}`);
    }
    
  } catch (err) {
    console.error(`\n💥 Server error:`, err);
    sendError(res, 500, 'Internal server error');
  }
  
  logRequest(req, startTime);
}

// ============================================
// Create Server
// ============================================

const server = http.createServer(router);

// Error handling
server.on('error', (err) => {
  if (err.code === 'EADDRINUSE') {
    console.error(`❌ Port 3000 is already in use`);
    process.exit(1);
  } else {
    console.error('❌ Server error:', err);
  }
});

// Start server
const PORT = process.env.PORT || 3000;
server.listen(PORT, () => {
  console.log(`\n✅ Server started successfully`);
  console.log(`   URL: http://localhost:${PORT}`);
  console.log(`   Press Ctrl+C to stop`);
  console.log('\n' + '='.repeat(70));
  console.log('\n📖 Available Endpoints:\n');
  console.log('   GET    /health                    - Health check');
  console.log('   GET    /accounts                  - List accounts');
  console.log('   GET    /accounts/:id              - Get account');
  console.log('   POST   /accounts                  - Create account');
  console.log('   PUT    /accounts/:id              - Update account');
  console.log('   DELETE /accounts/:id              - Delete account');
  console.log('   POST   /transactions              - Create transaction');
  console.log('   GET    /transactions              - List transactions');
  console.log('\n   Authentication: Bearer valid-token-12345');
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

**To test this server:**

1. Start the server:
```bash
node banking-api-server.js
```

2. Test with curl:
```bash
# Health check
curl http://localhost:3000/health

# Get accounts (with auth)
curl -H "Authorization: Bearer valid-token-12345" \
     http://localhost:3000/accounts

# Get specific account
curl -H "Authorization: Bearer valid-token-12345" \
     http://localhost:3000/accounts/ACC001

# Create account
curl -X POST \
     -H "Authorization: Bearer valid-token-12345" \
     -H "Content-Type: application/json" \
     -d '{"name":"Alice Brown","initialBalance":3000}' \
     http://localhost:3000/accounts

# Create transaction
curl -X POST \
     -H "Authorization: Bearer valid-token-12345" \
     -H "Content-Type: application/json" \
     -d '{"fromAccountId":"ACC001","toAccountId":"ACC002","amount":500}' \
     http://localhost:3000/transactions

# Get transactions
curl -H "Authorization: Bearer valid-token-12345" \
     http://localhost:3000/transactions?accountId=ACC001
```

**Expected Output:**

```
🏦 Banking REST API Server

======================================================================

✅ Server started successfully
   URL: http://localhost:3000
   Press Ctrl+C to stop

======================================================================

📖 Available Endpoints:

   GET    /health                    - Health check
   GET    /accounts                  - List accounts
   GET    /accounts/:id              - Get account
   POST   /accounts                  - Create account
   PUT    /accounts/:id              - Update account
   DELETE /accounts/:id              - Delete account
   POST   /transactions              - Create transaction
   GET    /transactions              - List transactions

   Authentication: Bearer valid-token-12345

======================================================================

📋 GET /accounts
   ✅ Returned 3 accounts
[2025-11-13T10:30:45.123Z] GET /accounts - 5ms

👤 GET /accounts/ACC001
   ✅ Returned account: John Doe
[2025-11-13T10:30:46.456Z] GET /accounts/ACC001 - 2ms

➕ POST /accounts
   ✅ Created account: ACCA3F7E9 - Alice Brown
[2025-11-13T10:30:47.789Z] POST /accounts - 8ms

💸 POST /transactions
   ✅ Transaction: ACC001 → ACC002 ($500)
[2025-11-13T10:30:48.012Z] POST /transactions - 12ms
```

**Key Learnings from Example 1**:

1. **http.createServer()**: Create HTTP server with request handler
2. **Routing**: Pattern matching on req.url and req.method
3. **JSON API**: Parse JSON body, send JSON responses
4. **Authentication**: Bearer token validation
5. **Error Handling**: Proper HTTP status codes
6. **CORS**: Allow cross-origin requests
7. **Logging**: Track requests and performance
8. **Graceful Shutdown**: Handle SIGTERM signal

---

## 🏦 Example 2: HTTP Client - Making API Requests

This example demonstrates making HTTP requests to external APIs.

**File: `banking-api-client.js`**

```javascript
import http from 'http';
import https from 'https';
import { URL } from 'url';

/**
 * Banking API Client
 * Demonstrates: Making HTTP requests, handling responses, error handling
 */

console.log('🌐 Banking API Client\n');
console.log('='.repeat(70));

// ============================================
// HTTP Client Helper
// ============================================

/**
 * Make HTTP/HTTPS request
 */
function makeRequest(url, options = {}) {
  return new Promise((resolve, reject) => {
    const urlObj = new URL(url);
    const isHttps = urlObj.protocol === 'https:';
    const client = isHttps ? https : http;
    
    const requestOptions = {
      method: options.method || 'GET',
      headers: options.headers || {},
      timeout: options.timeout || 30000
    };
    
    // Add body if present
    if (options.body) {
      const bodyStr = typeof options.body === 'string' 
        ? options.body 
        : JSON.stringify(options.body);
      
      requestOptions.headers['Content-Type'] = 'application/json';
      requestOptions.headers['Content-Length'] = Buffer.byteLength(bodyStr);
    }
    
    console.log(`\n📤 ${requestOptions.method} ${url}`);
    if (options.body) {
      console.log(`   Body:`, JSON.stringify(options.body, null, 2));
    }
    
    const req = client.request(url, requestOptions, (res) => {
      let data = '';
      
      console.log(`   Status: ${res.statusCode} ${res.statusMessage}`);
      console.log(`   Headers:`, res.headers);
      
      // Collect data chunks
      res.on('data', (chunk) => {
        data += chunk;
      });
      
      // Response complete
      res.on('end', () => {
        try {
          // Parse JSON if Content-Type is application/json
          const contentType = res.headers['content-type'] || '';
          
          if (contentType.includes('application/json')) {
            const parsed = JSON.parse(data);
            console.log(`   Response:`, JSON.stringify(parsed, null, 2));
            
            resolve({
              statusCode: res.statusCode,
              headers: res.headers,
              body: parsed
            });
          } else {
            console.log(`   Response: ${data.substring(0, 200)}...`);
            
            resolve({
              statusCode: res.statusCode,
              headers: res.headers,
              body: data
            });
          }
        } catch (err) {
          reject(new Error(`Failed to parse response: ${err.message}`));
        }
      });
    });
    
    // Handle errors
    req.on('error', (err) => {
      console.error(`   ❌ Request error: ${err.message}`);
      reject(err);
    });
    
    req.on('timeout', () => {
      console.error(`   ⏱️  Request timeout`);
      req.destroy();
      reject(new Error('Request timeout'));
    });
    
    // Write body if present
    if (options.body) {
      const bodyStr = typeof options.body === 'string' 
        ? options.body 
        : JSON.stringify(options.body);
      req.write(bodyStr);
    }
    
    req.end();
  });
}

// ============================================
// Banking API Client Class
// ============================================

class BankingAPIClient {
  constructor(baseURL, authToken) {
    this.baseURL = baseURL;
    this.authToken = authToken;
  }
  
  /**
   * Get default headers
   */
  getHeaders() {
    return {
      'Authorization': `Bearer ${this.authToken}`,
      'Accept': 'application/json',
      'User-Agent': 'Banking-Client/1.0'
    };
  }
  
  /**
   * GET request
   */
  async get(endpoint, query = {}) {
    const queryString = new URLSearchParams(query).toString();
    const url = `${this.baseURL}${endpoint}${queryString ? '?' + queryString : ''}`;
    
    return makeRequest(url, {
      method: 'GET',
      headers: this.getHeaders()
    });
  }
  
  /**
   * POST request
   */
  async post(endpoint, body) {
    const url = `${this.baseURL}${endpoint}`;
    
    return makeRequest(url, {
      method: 'POST',
      headers: this.getHeaders(),
      body
    });
  }
  
  /**
   * PUT request
   */
  async put(endpoint, body) {
    const url = `${this.baseURL}${endpoint}`;
    
    return makeRequest(url, {
      method: 'PUT',
      headers: this.getHeaders(),
      body
    });
  }
  
  /**
   * DELETE request
   */
  async delete(endpoint) {
    const url = `${this.baseURL}${endpoint}`;
    
    return makeRequest(url, {
      method: 'DELETE',
      headers: this.getHeaders()
    });
  }
  
  /**
   * Get all accounts
   */
  async getAccounts(filters = {}) {
    console.log('\n📊 Fetching accounts...');
    const response = await this.get('/accounts', filters);
    console.log(`✅ Retrieved ${response.body.count} accounts`);
    return response.body.accounts;
  }
  
  /**
   * Get account by ID
   */
  async getAccount(accountId) {
    console.log(`\n👤 Fetching account ${accountId}...`);
    const response = await this.get(`/accounts/${accountId}`);
    console.log(`✅ Account: ${response.body.account.name}`);
    return response.body.account;
  }
  
  /**
   * Create new account
   */
  async createAccount(name, initialBalance, currency = 'USD') {
    console.log(`\n➕ Creating account for ${name}...`);
    const response = await this.post('/accounts', {
      name,
      initialBalance,
      currency
    });
    console.log(`✅ Created: ${response.body.account.id}`);
    return response.body.account;
  }
  
  /**
   * Transfer money
   */
  async transfer(fromAccountId, toAccountId, amount, description) {
    console.log(`\n💸 Transferring $${amount}...`);
    const response = await this.post('/transactions', {
      fromAccountId,
      toAccountId,
      amount,
      description
    });
    console.log(`✅ Transaction completed: ${response.body.transaction.id}`);
    return response.body.transaction;
  }
  
  /**
   * Get transactions
   */
  async getTransactions(accountId = null, limit = 10) {
    console.log(`\n📜 Fetching transactions...`);
    const query = { limit };
    if (accountId) query.accountId = accountId;
    
    const response = await this.get('/transactions', query);
    console.log(`✅ Retrieved ${response.body.count} transactions`);
    return response.body.transactions;
  }
  
  /**
   * Health check
   */
  async healthCheck() {
    console.log(`\n🏥 Health check...`);
    const response = await this.get('/health');
    console.log(`✅ Status: ${response.body.status}`);
    return response.body;
  }
}

// ============================================
// Demo Usage
// ============================================

async function demo() {
  try {
    // Initialize client
    const client = new BankingAPIClient(
      'http://localhost:3000',
      'valid-token-12345'
    );
    
    console.log('\n🚀 Starting API client demo...\n');
    
    // 1. Health check
    await client.healthCheck();
    
    // 2. Get all accounts
    const accounts = await client.getAccounts();
    console.log(`\n   Found ${accounts.length} accounts:`);
    accounts.forEach(acc => {
      console.log(`   - ${acc.id}: ${acc.name} ($${acc.balance})`);
    });
    
    // 3. Get specific account
    const account = await client.getAccount('ACC001');
    console.log(`\n   Account details:`);
    console.log(`   Name: ${account.name}`);
    console.log(`   Balance: $${account.balance}`);
    
    // 4. Create new account
    const newAccount = await client.createAccount('Charlie Davis', 5000);
    console.log(`\n   New account created:`);
    console.log(`   ID: ${newAccount.id}`);
    console.log(`   Name: ${newAccount.name}`);
    console.log(`   Balance: $${newAccount.balance}`);
    
    // 5. Make transfer
    const transaction = await client.transfer(
      'ACC001',
      'ACC002',
      250,
      'Monthly payment'
    );
    console.log(`\n   Transfer details:`);
    console.log(`   ID: ${transaction.id}`);
    console.log(`   Amount: $${transaction.amount}`);
    console.log(`   Status: ${transaction.status}`);
    
    // 6. Get transactions for account
    const transactions = await client.getTransactions('ACC001');
    console.log(`\n   Recent transactions for ACC001:`);
    transactions.forEach(txn => {
      console.log(`   - ${txn.id}: $${txn.amount} (${txn.status})`);
    });
    
    // 7. Filter accounts by minimum balance
    const richAccounts = await client.getAccounts({ minBalance: 5000 });
    console.log(`\n   Accounts with balance >= $5000: ${richAccounts.length}`);
    
    console.log('\n✅ Demo completed successfully!');
    console.log('\n' + '='.repeat(70) + '\n');
    
  } catch (err) {
    console.error('\n❌ Demo failed:', err.message);
    console.error(err.stack);
  }
}

// Run demo if server is available
console.log('\n⚠️  Make sure the banking-api-server.js is running on port 3000');
console.log('   Start it with: node banking-api-server.js\n');

setTimeout(() => {
  demo();
}, 1000);
```

**To test this client:**

1. Start the server (in one terminal):
```bash
node banking-api-server.js
```

2. Run the client (in another terminal):
```bash
node banking-api-client.js
```

**Expected Output:**

```
🌐 Banking API Client

======================================================================

⚠️  Make sure the banking-api-server.js is running on port 3000
   Start it with: node banking-api-server.js

🚀 Starting API client demo...

🏥 Health check...

📤 GET http://localhost:3000/health
   Status: 200 OK
   Headers: {
     'content-type': 'application/json',
     'x-powered-by': 'Node.js Banking API'
   }
   Response: {
     "status": "healthy",
     "uptime": 45.123,
     "timestamp": "2025-11-13T10:30:00.000Z",
     "accounts": 3,
     "transactions": 0
   }
✅ Status: healthy

📊 Fetching accounts...

📤 GET http://localhost:3000/accounts
   Status: 200 OK
   Response: {
     "count": 3,
     "accounts": [...]
   }
✅ Retrieved 3 accounts

   Found 3 accounts:
   - ACC001: John Doe ($5000)
   - ACC002: Jane Smith ($10000)
   - ACC003: Bob Johnson ($2500)

👤 Fetching account ACC001...

📤 GET http://localhost:3000/accounts/ACC001
   Status: 200 OK
   Response: {
     "account": {
       "id": "ACC001",
       "name": "John Doe",
       "balance": 5000,
       "currency": "USD"
     }
   }
✅ Account: John Doe

   Account details:
   Name: John Doe
   Balance: $5000

➕ Creating account for Charlie Davis...

📤 POST http://localhost:3000/accounts
   Body: {
     "name": "Charlie Davis",
     "initialBalance": 5000,
     "currency": "USD"
   }
   Status: 201 Created
   Response: {
     "message": "Account created successfully",
     "account": {
       "id": "ACCF3A9B2",
       "name": "Charlie Davis",
       "balance": 5000,
       "currency": "USD"
     }
   }
✅ Created: ACCF3A9B2

💸 Transferring $250...

📤 POST http://localhost:3000/transactions
   Body: {
     "fromAccountId": "ACC001",
     "toAccountId": "ACC002",
     "amount": 250,
     "description": "Monthly payment"
   }
   Status: 201 Created
   Response: {
     "message": "Transaction completed successfully",
     "transaction": {
       "id": "TXN7F2E4A9C",
       "amount": 250,
       "status": "completed"
     }
   }
✅ Transaction completed: TXN7F2E4A9C

✅ Demo completed successfully!

======================================================================
```

**Key Learnings from Example 2**:

1. **http.request()**: Make HTTP requests programmatically
2. **https module**: Same API for HTTPS requests
3. **Promise wrapper**: Convert callback-based API to Promises
4. **Request body**: Write JSON data to request stream
5. **Response handling**: Collect chunks, parse JSON
6. **Error handling**: Network errors, timeouts, parsing errors
7. **Client class**: Organize API calls in reusable class
8. **Headers**: Authorization, Content-Type, User-Agent

---

## ✅ 10 DO's: Best Practices for HTTP Module

### 1. ✅ DO: Always Set Appropriate Status Codes

**Why**: Status codes communicate the result of the request clearly.

```javascript
// Good: Proper status codes
function sendResponse(res, success, data) {
  if (success) {
    res.statusCode = 200;  // OK
  } else if (data.notFound) {
    res.statusCode = 404;  // Not Found
  } else if (data.unauthorized) {
    res.statusCode = 401;  // Unauthorized
  } else {
    res.statusCode = 500;  // Internal Server Error
  }
  
  res.end(JSON.stringify(data));
}

// Common status codes:
// 200 OK - Success
// 201 Created - Resource created
// 204 No Content - Success, no body
// 400 Bad Request - Invalid input
// 401 Unauthorized - No auth
// 403 Forbidden - No permission
// 404 Not Found - Resource not found
// 500 Internal Server Error - Server error
```

**Banking Use Case**: Return 400 for invalid transaction amounts, 404 for non-existent accounts.

---

### 2. ✅ DO: Set Content-Type Header

**Why**: Tells the client what type of data is being sent.

```javascript
// Good: Always set Content-Type
function sendJSON(res, data) {
  res.setHeader('Content-Type', 'application/json');
  res.end(JSON.stringify(data));
}

function sendHTML(res, html) {
  res.setHeader('Content-Type', 'text/html; charset=utf-8');
  res.end(html);
}

function sendText(res, text) {
  res.setHeader('Content-Type', 'text/plain; charset=utf-8');
  res.end(text);
}

function sendCSV(res, csv) {
  res.setHeader('Content-Type', 'text/csv');
  res.setHeader('Content-Disposition', 'attachment; filename=report.csv');
  res.end(csv);
}
```

**Banking Use Case**: Send JSON for API responses, CSV for transaction reports.

---

### 3. ✅ DO: Parse Request Body Safely

**Why**: Prevent memory exhaustion from large payloads.

```javascript
// Good: Limit body size and handle errors
function parseBody(req, maxSize = 1024 * 1024) {  // 1MB default
  return new Promise((resolve, reject) => {
    let body = '';
    let size = 0;
    
    req.on('data', chunk => {
      size += chunk.length;
      
      if (size > maxSize) {
        req.socket.destroy();
        reject(new Error('Request body too large'));
        return;
      }
      
      body += chunk.toString();
    });
    
    req.on('end', () => {
      try {
        const parsed = body ? JSON.parse(body) : {};
        resolve(parsed);
      } catch (err) {
        reject(new Error('Invalid JSON'));
      }
    });
    
    req.on('error', reject);
  });
}

// Usage
async function handler(req, res) {
  try {
    const body = await parseBody(req);
    // Process body...
  } catch (err) {
    res.statusCode = 400;
    res.end(JSON.stringify({ error: err.message }));
  }
}
```

**Banking Use Case**: Protect against malicious large payloads in transaction requests.

---

### 4. ✅ DO: Handle Errors Properly

**Why**: Prevent server crashes and provide useful error messages.

```javascript
// Good: Comprehensive error handling
const server = http.createServer(async (req, res) => {
  try {
    await handleRequest(req, res);
  } catch (err) {
    console.error('Request error:', err);
    
    if (!res.headersSent) {
      res.statusCode = 500;
      res.setHeader('Content-Type', 'application/json');
      res.end(JSON.stringify({
        error: {
          message: 'Internal server error',
          code: 'SERVER_ERROR',
          timestamp: new Date().toISOString()
        }
      }));
    }
  }
});

server.on('error', (err) => {
  if (err.code === 'EADDRINUSE') {
    console.error('Port already in use');
    process.exit(1);
  } else {
    console.error('Server error:', err);
  }
});

server.on('clientError', (err, socket) => {
  socket.end('HTTP/1.1 400 Bad Request\r\n\r\n');
});
```

**Banking Use Case**: Log errors for audit trail, return safe error messages to clients.

---

### 5. ✅ DO: Implement Request Timeouts

**Why**: Prevent slow requests from hanging indefinitely.

```javascript
// Good: Set timeouts
const server = http.createServer((req, res) => {
  // Set timeout for the entire request
  req.setTimeout(30000, () => {
    console.log('Request timeout');
    res.statusCode = 408;  // Request Timeout
    res.end('Request timeout');
  });
  
  // Handle request...
});

// Set server timeout
server.timeout = 60000;  // 60 seconds

// For HTTP client requests
function makeRequest(url) {
  return new Promise((resolve, reject) => {
    const req = http.request(url, { timeout: 30000 }, resolve);
    
    req.on('timeout', () => {
      req.destroy();
      reject(new Error('Request timeout'));
    });
    
    req.end();
  });
}
```

**Banking Use Case**: Timeout long-running payment processing requests.

---

### 6. ✅ DO: Use URL Module for Parsing

**Why**: Properly parse URLs and query parameters.

```javascript
// Good: Use URL module
import { URL } from 'url';

function handleRequest(req, res) {
  const urlObj = new URL(req.url, `http://${req.headers.host}`);
  
  const pathname = urlObj.pathname;       // '/api/accounts'
  const searchParams = urlObj.searchParams;
  
  const accountId = searchParams.get('id');      // Single value
  const filters = searchParams.getAll('filter'); // Multiple values
  
  console.log('Path:', pathname);
  console.log('Query:', Object.fromEntries(searchParams));
  
  // Use pathname for routing
  if (pathname === '/api/accounts') {
    // Handle accounts endpoint
  }
}
```

**Banking Use Case**: Parse account filters, date ranges from query parameters.

---

### 7. ✅ DO: Implement CORS for Cross-Origin Requests

**Why**: Allow frontend applications to call your API.

```javascript
// Good: Proper CORS handling
function setCORSHeaders(res) {
  res.setHeader('Access-Control-Allow-Origin', '*');
  // Or specific origin:
  // res.setHeader('Access-Control-Allow-Origin', 'https://bank.com');
  
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
  res.setHeader('Access-Control-Max-Age', '86400');  // 24 hours
}

const server = http.createServer((req, res) => {
  setCORSHeaders(res);
  
  // Handle preflight requests
  if (req.method === 'OPTIONS') {
    res.statusCode = 204;
    res.end();
    return;
  }
  
  // Handle actual request...
});
```

**Banking Use Case**: Allow mobile app and web app to access banking API.

---

### 8. ✅ DO: Log Requests for Monitoring

**Why**: Track API usage, debug issues, security auditing.

```javascript
// Good: Comprehensive logging
function logRequest(req, res, startTime) {
  const duration = Date.now() - startTime;
  const timestamp = new Date().toISOString();
  const ip = req.socket.remoteAddress;
  const userAgent = req.headers['user-agent'];
  
  console.log(JSON.stringify({
    timestamp,
    method: req.method,
    url: req.url,
    statusCode: res.statusCode,
    duration: `${duration}ms`,
    ip,
    userAgent
  }));
}

const server = http.createServer(async (req, res) => {
  const startTime = Date.now();
  
  // Track when response finishes
  res.on('finish', () => {
    logRequest(req, res, startTime);
  });
  
  // Handle request...
});
```

**Banking Use Case**: Audit trail for all transactions and account access.

---

### 9. ✅ DO: Validate Input Data

**Why**: Prevent invalid data from entering the system.

```javascript
// Good: Validate all inputs
function validateAccount(data) {
  const errors = [];
  
  if (!data.name || typeof data.name !== 'string') {
    errors.push('Name is required and must be a string');
  }
  
  if (data.name && data.name.length > 100) {
    errors.push('Name must be less than 100 characters');
  }
  
  if (typeof data.initialBalance !== 'number' || data.initialBalance < 0) {
    errors.push('Initial balance must be a positive number');
  }
  
  if (data.currency && !['USD', 'EUR', 'GBP'].includes(data.currency)) {
    errors.push('Invalid currency');
  }
  
  return errors;
}

async function handleCreateAccount(req, res) {
  const body = await parseBody(req);
  const errors = validateAccount(body);
  
  if (errors.length > 0) {
    res.statusCode = 400;
    res.end(JSON.stringify({ errors }));
    return;
  }
  
  // Create account...
}
```

**Banking Use Case**: Validate transaction amounts, account numbers, currency codes.

---

### 10. ✅ DO: Implement Graceful Shutdown

**Why**: Clean up resources when server stops.

```javascript
// Good: Graceful shutdown
const server = http.createServer(handler);

// Track active connections
const connections = new Set();

server.on('connection', (conn) => {
  connections.add(conn);
  
  conn.on('close', () => {
    connections.delete(conn);
  });
});

function shutdown() {
  console.log('Shutting down gracefully...');
  
  // Stop accepting new connections
  server.close(() => {
    console.log('Server closed');
    process.exit(0);
  });
  
  // Close existing connections after timeout
  setTimeout(() => {
    console.log('Forcing shutdown...');
    connections.forEach(conn => conn.destroy());
    process.exit(1);
  }, 10000);  // 10 second grace period
}

process.on('SIGTERM', shutdown);
process.on('SIGINT', shutdown);
```

**Banking Use Case**: Complete in-flight transactions before shutdown.

---

## ❌ 10 DON'Ts: Common Mistakes to Avoid

### 1. ❌ DON'T: Forget to End the Response

**Why**: Request will hang and timeout.

```javascript
// Bad: Forgot res.end()
const server = http.createServer((req, res) => {
  res.statusCode = 200;
  res.setHeader('Content-Type', 'application/json');
  // Oops! Forgot res.end()
});
// Client hangs forever!

// Good: Always end the response
const server = http.createServer((req, res) => {
  res.statusCode = 200;
  res.setHeader('Content-Type', 'application/json');
  res.end(JSON.stringify({ status: 'ok' }));  // ✅
});
```

---

### 2. ❌ DON'T: Send Multiple Responses

**Why**: Causes "Cannot set headers after they are sent" error.

```javascript
// Bad: Multiple res.end() calls
async function handler(req, res) {
  res.end('First response');
  
  // Later...
  res.end('Second response');  // ❌ Error!
}

// Good: Guard against multiple responses
async function handler(req, res) {
  if (res.headersSent) {
    return;  // Already sent response
  }
  
  res.end('Response');
}
```

---

### 3. ❌ DON'T: Block the Event Loop

**Why**: Blocks all requests, makes server unresponsive.

```javascript
// Bad: Synchronous operations
const server = http.createServer((req, res) => {
  // This blocks ALL requests!
  const data = fs.readFileSync('/large-file.txt');
  
  // CPU-intensive calculation blocks event loop
  let sum = 0;
  for (let i = 0; i < 1000000000; i++) {
    sum += i;
  }
  
  res.end('Done');
});

// Good: Use async operations
const server = http.createServer(async (req, res) => {
  // Non-blocking
  const data = await fs.promises.readFile('/large-file.txt');
  
  // Move CPU-intensive work to worker thread
  const worker = new Worker('./worker.js');
  const sum = await new Promise(resolve => {
    worker.on('message', resolve);
    worker.postMessage({ task: 'calculate' });
  });
  
  res.end('Done');
});
```

---

### 4. ❌ DON'T: Ignore Request Body Size

**Why**: Memory exhaustion from large payloads.

```javascript
// Bad: No size limit
function parseBody(req) {
  return new Promise((resolve) => {
    let body = '';
    req.on('data', chunk => {
      body += chunk;  // ❌ Can grow infinitely!
    });
    req.on('end', () => resolve(body));
  });
}

// Good: Enforce size limits
function parseBody(req, maxSize = 1024 * 1024) {
  return new Promise((resolve, reject) => {
    let body = '';
    let size = 0;
    
    req.on('data', chunk => {
      size += chunk.length;
      if (size > maxSize) {
        req.socket.destroy();
        reject(new Error('Payload too large'));
        return;
      }
      body += chunk;
    });
    
    req.on('end', () => resolve(body));
  });
}
```

---

### 5. ❌ DON'T: Return Sensitive Error Details

**Why**: Security risk, information leakage.

```javascript
// Bad: Expose internal errors
try {
  await database.query('SELECT * FROM accounts WHERE id = ?', [accountId]);
} catch (err) {
  res.statusCode = 500;
  res.end(JSON.stringify({
    error: err.message,      // ❌ "Connection to DB failed at 192.168.1.100:3306"
    stack: err.stack         // ❌ Full stack trace!
  }));
}

// Good: Generic error messages
try {
  await database.query('SELECT * FROM accounts WHERE id = ?', [accountId]);
} catch (err) {
  console.error('Database error:', err);  // Log internally
  
  res.statusCode = 500;
  res.end(JSON.stringify({
    error: 'An error occurred processing your request',
    code: 'INTERNAL_ERROR',
    requestId: generateRequestId()
  }));
}
```

---

### 6. ❌ DON'T: Hardcode URLs and Ports

**Why**: Makes deployment difficult, not configurable.

```javascript
// Bad: Hardcoded values
const server = http.createServer(handler);
server.listen(3000, 'localhost');

const apiURL = 'http://api.bank.com:8080';

// Good: Use environment variables
const PORT = process.env.PORT || 3000;
const HOST = process.env.HOST || '0.0.0.0';
const API_URL = process.env.API_URL || 'http://localhost:8080';

const server = http.createServer(handler);
server.listen(PORT, HOST, () => {
  console.log(`Server running on ${HOST}:${PORT}`);
});
```

---

### 7. ❌ DON'T: Skip Input Validation

**Why**: Security vulnerabilities, data integrity issues.

```javascript
// Bad: No validation
async function handleTransfer(req, res) {
  const body = await parseBody(req);
  
  // Direct use without validation!
  await db.transfer(body.from, body.to, body.amount);  // ❌
  
  res.end('OK');
}

// Good: Validate all inputs
async function handleTransfer(req, res) {
  const body = await parseBody(req);
  
  // Validate
  if (!body.from || !body.to || !body.amount) {
    return sendError(res, 400, 'Missing required fields');
  }
  
  if (typeof body.amount !== 'number' || body.amount <= 0) {
    return sendError(res, 400, 'Invalid amount');
  }
  
  if (body.amount > 100000) {
    return sendError(res, 400, 'Amount exceeds limit');
  }
  
  await db.transfer(body.from, body.to, body.amount);
  res.end('OK');
}
```

---

### 8. ❌ DON'T: Ignore HTTP Methods

**Why**: REST conventions, security (GET should be idempotent).

```javascript
// Bad: All methods do the same thing
const server = http.createServer(async (req, res) => {
  // Handles all methods the same!
  await deleteAccount(req, res);  // ❌ Even for GET!
});

// Good: Check HTTP method
const server = http.createServer(async (req, res) => {
  if (req.method === 'GET') {
    await getAccount(req, res);
  } else if (req.method === 'POST') {
    await createAccount(req, res);
  } else if (req.method === 'DELETE') {
    await deleteAccount(req, res);
  } else {
    res.statusCode = 405;  // Method Not Allowed
    res.setHeader('Allow', 'GET, POST, DELETE');
    res.end('Method not allowed');
  }
});
```

---

### 9. ❌ DON'T: Store State in Server Memory

**Why**: Doesn't scale, lost on restart.

```javascript
// Bad: In-memory state
const sessions = new Map();  // ❌ Lost on restart!

const server = http.createServer((req, res) => {
  const sessionId = req.headers.cookie;
  const session = sessions.get(sessionId);
  
  // Session lost if server restarts or scales to multiple instances
});

// Good: Use external store
import Redis from 'redis';
const redis = Redis.createClient();

const server = http.createServer(async (req, res) => {
  const sessionId = req.headers.cookie;
  const session = await redis.get(`session:${sessionId}`);
  
  // Works across restarts and multiple instances
});
```

---

### 10. ❌ DON'T: Forget HTTPS in Production

**Why**: Data transmitted in plain text, security risk.

```javascript
// Bad: HTTP in production
const server = http.createServer(handler);
server.listen(3000);
// ❌ All data (passwords, tokens) sent unencrypted!

// Good: Use HTTPS in production
import https from 'https';
import fs from 'fs';

const options = {
  key: fs.readFileSync('privatekey.pem'),
  cert: fs.readFileSync('certificate.pem')
};

const server = https.createServer(options, handler);
server.listen(443);

// Or use reverse proxy (nginx) to handle HTTPS
```

---

## 🎓 Key Takeaways

### HTTP Status Codes Reference

| Code | Name | When to Use | Example |
|------|------|-------------|---------|
| **2xx Success** | | | |
| 200 | OK | Request successful | GET /accounts |
| 201 | Created | Resource created | POST /accounts |
| 204 | No Content | Success, no body | DELETE /accounts/123 |
| **3xx Redirection** | | | |
| 301 | Moved Permanently | Resource moved | Old API version |
| 302 | Found | Temporary redirect | After login |
| 304 | Not Modified | Cached version valid | Conditional GET |
| **4xx Client Error** | | | |
| 400 | Bad Request | Invalid input | Missing required field |
| 401 | Unauthorized | Authentication required | No token |
| 403 | Forbidden | No permission | Access denied |
| 404 | Not Found | Resource doesn't exist | Account not found |
| 405 | Method Not Allowed | Wrong HTTP method | POST to read-only endpoint |
| 409 | Conflict | Resource conflict | Duplicate account |
| 422 | Unprocessable Entity | Validation failed | Invalid email format |
| 429 | Too Many Requests | Rate limit exceeded | Too many API calls |
| **5xx Server Error** | | | |
| 500 | Internal Server Error | Unexpected error | Database crash |
| 502 | Bad Gateway | Upstream error | External API down |
| 503 | Service Unavailable | Server overloaded | Maintenance mode |
| 504 | Gateway Timeout | Upstream timeout | Slow external API |

### HTTP Methods (REST)

| Method | Purpose | Idempotent | Safe | Body |
|--------|---------|------------|------|------|
| **GET** | Retrieve resource | ✅ Yes | ✅ Yes | No |
| **POST** | Create resource | ❌ No | ❌ No | Yes |
| **PUT** | Update/Replace | ✅ Yes | ❌ No | Yes |
| **PATCH** | Partial update | ❌ No | ❌ No | Yes |
| **DELETE** | Delete resource | ✅ Yes | ❌ No | No |
| **HEAD** | Get headers only | ✅ Yes | ✅ Yes | No |
| **OPTIONS** | Get allowed methods | ✅ Yes | ✅ Yes | No |

- **Idempotent**: Multiple identical requests have same effect as single request
- **Safe**: Request doesn't modify server state

### Common Headers

**Request Headers**:
```
Authorization: Bearer <token>          # Authentication
Content-Type: application/json         # Request body type
Accept: application/json               # Desired response type
User-Agent: MyApp/1.0                  # Client identifier
Cookie: session=abc123                 # Session/auth
If-None-Match: "etag123"              # Conditional request
```

**Response Headers**:
```
Content-Type: application/json         # Response body type
Content-Length: 1234                   # Body size in bytes
Cache-Control: max-age=3600           # Caching directives
ETag: "abc123"                        # Version identifier
Set-Cookie: session=xyz; HttpOnly     # Set cookie
Access-Control-Allow-Origin: *        # CORS
```

### Request/Response Objects

**Request (IncomingMessage)**:
```javascript
req.url           // '/api/accounts?id=123'
req.method        // 'GET', 'POST', etc.
req.headers       // { 'content-type': 'application/json', ... }
req.httpVersion   // '1.1'
req.socket        // Underlying socket
```

**Response (ServerResponse)**:
```javascript
res.statusCode = 200
res.setHeader('Content-Type', 'application/json')
res.writeHead(200, { 'Content-Type': 'application/json' })
res.write(chunk)   // Write chunk (can call multiple times)
res.end(data)      // End response (optional data)
```

### When to Use HTTP vs Express

| Use Case | HTTP Module | Express |
|----------|-------------|---------|
| **Learning** | ✅ Good | Optional |
| **Microservice** | ✅ Good | ✅ Better |
| **REST API** | ⚠️ Verbose | ✅ Best |
| **Routing** | ❌ Manual | ✅ Built-in |
| **Middleware** | ❌ Manual | ✅ Built-in |
| **Performance** | ✅ Slightly faster | ⚠️ Overhead minimal |
| **Flexibility** | ✅ Full control | ⚠️ Framework constraints |

**Recommendation**: Use Express for most applications. Use raw HTTP module for:
- Learning Node.js internals
- Maximum performance requirements
- Building custom frameworks
- Minimal dependencies needed

### Interview Q&A

**Q1: What's the difference between http.createServer() and server.listen()?**

**A**: `http.createServer()` creates a server instance but doesn't start it. `server.listen()` binds to a port and starts accepting connections.

```javascript
const server = http.createServer(handler);  // Create
server.listen(3000);                        // Start listening
```

**Q2: How do you parse JSON request body in Node.js HTTP module?**

**A**: Collect data chunks and parse when complete:

```javascript
function parseBody(req) {
  return new Promise((resolve, reject) => {
    let body = '';
    req.on('data', chunk => body += chunk);
    req.on('end', () => {
      try {
        resolve(JSON.parse(body));
      } catch (err) {
        reject(err);
      }
    });
  });
}
```

**Q3: What's the difference between res.write() and res.end()?**

**A**:
- `res.write(chunk)`: Write data chunk, can call multiple times, response stays open
- `res.end(data)`: Write final data (optional) and close response

```javascript
res.write('Part 1');
res.write('Part 2');
res.end('Final part');  // Complete response
```

**Q4: How do you implement routing without Express?**

**A**: Parse URL and method, use conditionals or routing table:

```javascript
const routes = {
  'GET /accounts': handleGetAccounts,
  'POST /accounts': handleCreateAccount,
  'DELETE /accounts/:id': handleDeleteAccount
};

function router(req, res) {
  const key = `${req.method} ${req.url}`;
  const handler = routes[key];
  
  if (handler) {
    handler(req, res);
  } else {
    res.statusCode = 404;
    res.end('Not found');
  }
}
```

**Q5: How do you make HTTPS requests?**

**A**: Use `https` module (same API as `http`):

```javascript
import https from 'https';

https.get('https://api.example.com/data', (res) => {
  let data = '';
  res.on('data', chunk => data += chunk);
  res.on('end', () => console.log(data));
});
```

**Q6: How do you handle file uploads?**

**A**: Stream the request to a file:

```javascript
import fs from 'fs';

function handleUpload(req, res) {
  const writeStream = fs.createWriteStream('/uploads/file.pdf');
  
  req.pipe(writeStream);
  
  writeStream.on('finish', () => {
    res.end('Upload complete');
  });
  
  writeStream.on('error', (err) => {
    res.statusCode = 500;
    res.end('Upload failed');
  });
}
```

**Q7: What's the difference between http.request() and http.get()?**

**A**:
- `http.get()`: Shorthand for GET requests, automatically calls `req.end()`
- `http.request()`: Full control, supports all methods, must call `req.end()`

```javascript
// http.get - automatic end()
http.get('http://api.com/data', (res) => {
  // Handle response
});

// http.request - manual control
const req = http.request({
  method: 'POST',
  hostname: 'api.com',
  path: '/data'
}, (res) => {
  // Handle response
});
req.write(JSON.stringify(data));
req.end();
```

**Q8: How do you implement request timeout?**

**A**: Set timeout on request object:

```javascript
const server = http.createServer((req, res) => {
  req.setTimeout(30000, () => {
    res.statusCode = 408;
    res.end('Request timeout');
  });
  
  // Handle request...
});
```

**Q9: How do you stream large responses?**

**A**: Use `res.write()` in chunks or pipe a readable stream:

```javascript
// Method 1: Write chunks
function streamResponse(res) {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  
  for (let i = 0; i < 1000; i++) {
    res.write(JSON.stringify({ chunk: i }) + '\n');
  }
  
  res.end();
}

// Method 2: Pipe stream
function streamFile(res) {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  
  const stream = fs.createReadStream('/large-file.txt');
  stream.pipe(res);
}
```

**Q10: How do you implement CORS?**

**A**: Set Access-Control headers:

```javascript
function setCORS(res) {
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
}

const server = http.createServer((req, res) => {
  setCORS(res);
  
  if (req.method === 'OPTIONS') {
    res.statusCode = 204;
    res.end();
    return;
  }
  
  // Handle request...
});
```

### Banking-Specific HTTP Considerations

1. **Security**:
   - Always use HTTPS in production
   - Validate authentication tokens on every request
   - Rate limit to prevent brute force
   - Log all requests for audit trail

2. **Idempotency**:
   - Use idempotency keys for payments
   - Prevent duplicate transactions
   - Return same result for duplicate requests

3. **Error Handling**:
   - Never expose internal errors
   - Return generic error messages
   - Log detailed errors internally
   - Include request ID for tracking

4. **Performance**:
   - Cache account balances (short TTL)
   - Use streaming for large reports
   - Implement pagination for list endpoints
   - Set appropriate timeouts

5. **Compliance**:
   - Log all financial transactions
   - Encrypt sensitive data
   - Implement audit trails
   - Handle PCI DSS requirements

---

## 🏁 Summary

The HTTP module is the foundation for building web servers in Node.js. Key points:

1. **Server Creation**: `http.createServer()` creates server, `listen()` starts it
2. **Request Object**: Contains URL, method, headers, body (stream)
3. **Response Object**: Set status code, headers, send data
4. **Routing**: Parse URL and method to route requests
5. **JSON API**: Parse request body, send JSON responses
6. **HTTP Client**: Make requests with `http.request()` or `https.request()`
7. **Best Practices**: Validation, error handling, timeouts, logging, CORS
8. **Banking**: Authentication, security, audit logging, compliance

While raw HTTP module gives full control, most applications benefit from using Express.js for built-in routing, middleware, and convenience features.

---

