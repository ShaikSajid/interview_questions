# Q37: API Design - RESTful Principles

## 📋 Summary
REST (Representational State Transfer) is an architectural style for designing networked applications. This guide covers **REST constraints**, resource-based design, HTTP methods, status codes, HATEOAS, pagination, filtering, and comprehensive banking examples for account and transaction APIs.

---

## 🎯 What You'll Learn
- **REST Constraints**: Stateless, client-server, cacheable, uniform interface
- **Resource Naming**: Nouns not verbs, plural names, hierarchical structure
- **HTTP Methods**: GET, POST, PUT, PATCH, DELETE semantics
- **Status Codes**: 2xx, 3xx, 4xx, 5xx appropriate usage
- **HATEOAS**: Hypermedia links for API discoverability
- **Pagination**: Offset, cursor, link header pagination
- **Banking Examples**: Complete account management and transaction APIs

---

## 📖 Comprehensive Explanation

### What is REST?

**REST** (Representational State Transfer) is an architectural style that defines constraints for creating web services:

1. **Client-Server**: Separation of concerns
2. **Stateless**: Each request contains all necessary information
3. **Cacheable**: Responses explicitly indicate cacheability
4. **Uniform Interface**: Consistent resource identification
5. **Layered System**: Client doesn't know if connected directly
6. **Code on Demand** (optional): Server can extend client functionality

---

## 🏗️ REST Constraints Explained

### 1. Client-Server Architecture

**Principle**: Separate user interface concerns from data storage concerns.

**Benefits**:
- UI and backend can evolve independently
- Multiple clients (web, mobile, API) use same backend
- Simplifies server components

### 2. Stateless

**Principle**: Server doesn't store client context between requests.

**✅ Stateless (Good)**:
```javascript
// Client sends authentication token with every request
GET /api/accounts/123
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**❌ Stateful (Bad)**:
```javascript
// Server stores session state
GET /api/accounts/123
Cookie: sessionId=abc123
// Server looks up session in memory/database
```

### 3. Cacheable

**Principle**: Responses should indicate if they can be cached.

```javascript
GET /api/accounts/123

HTTP/1.1 200 OK
Cache-Control: max-age=300, private
ETag: "686897696a7c876b7e"
```

### 4. Uniform Interface

**Principle**: Consistent resource identification and manipulation.

- **Resource identification**: URIs
- **Manipulation through representations**: JSON, XML
- **Self-descriptive messages**: HTTP methods, status codes
- **HATEOAS**: Links to related resources

---

## 📝 Resource Naming Conventions

### Best Practices

1. **Use nouns, not verbs**
   ```
   ✅ GET /api/accounts
   ❌ GET /api/getAccounts
   ```

2. **Use plural names**
   ```
   ✅ GET /api/accounts/123
   ❌ GET /api/account/123
   ```

3. **Use hierarchies for relationships**
   ```
   GET /api/accounts/123/transactions
   GET /api/accounts/123/transactions/456
   ```

4. **Use hyphens for readability**
   ```
   ✅ /api/account-statements
   ❌ /api/account_statements
   ❌ /api/accountStatements
   ```

5. **Lowercase URIs**
   ```
   ✅ /api/accounts
   ❌ /api/Accounts
   ```

6. **Don't use file extensions**
   ```
   ✅ GET /api/accounts/123
   Accept: application/json
   
   ❌ GET /api/accounts/123.json
   ```

### Resource Examples

```
# Accounts
GET /api/accounts                    # List all accounts
GET /api/accounts/123                # Get specific account
POST /api/accounts                   # Create account
PUT /api/accounts/123                # Update account (full)
PATCH /api/accounts/123              # Update account (partial)
DELETE /api/accounts/123             # Delete account

# Nested resources
GET /api/accounts/123/transactions   # Account's transactions
GET /api/accounts/123/balance        # Account's balance
POST /api/accounts/123/transfers     # Create transfer from account

# Filtering and searching
GET /api/accounts?status=active
GET /api/transactions?from=2024-01-01&to=2024-01-31
GET /api/accounts?search=john
```

---

## 🛠️ HTTP Methods

| Method | Purpose | Idempotent | Safe | Example |
|--------|---------|------------|------|----------|
| **GET** | Retrieve resource | ✅ | ✅ | Get account details |
| **POST** | Create resource | ❌ | ❌ | Create new account |
| **PUT** | Update/replace resource | ✅ | ❌ | Update full account |
| **PATCH** | Partial update | ❌ | ❌ | Update account email |
| **DELETE** | Delete resource | ✅ | ❌ | Delete account |
| **HEAD** | Get headers only | ✅ | ✅ | Check if resource exists |
| **OPTIONS** | Get allowed methods | ✅ | ✅ | CORS preflight |

**Idempotent**: Multiple identical requests have same effect as single request  
**Safe**: No side effects (read-only)

### Method Examples

```javascript
// GET - Retrieve
GET /api/accounts/123
Response: { "id": 123, "balance": 1000 }

// POST - Create
POST /api/accounts
Body: { "name": "John Doe", "currency": "USD" }
Response: 201 Created
Location: /api/accounts/456

// PUT - Full update (all fields required)
PUT /api/accounts/123
Body: { "name": "Jane Doe", "email": "jane@example.com", "currency": "USD" }
Response: 200 OK

// PATCH - Partial update (only changed fields)
PATCH /api/accounts/123
Body: { "email": "newemail@example.com" }
Response: 200 OK

// DELETE - Remove
DELETE /api/accounts/123
Response: 204 No Content
```

---

## 🚦 HTTP Status Codes

### 2xx Success

| Code | Meaning | Use Case |
|------|---------|----------|
| **200** | OK | Successful GET, PUT, PATCH |
| **201** | Created | Successful POST |
| **202** | Accepted | Request accepted, processing async |
| **204** | No Content | Successful DELETE |

### 3xx Redirection

| Code | Meaning | Use Case |
|------|---------|----------|
| **301** | Moved Permanently | Resource moved to new URI |
| **304** | Not Modified | Cached version still valid |

### 4xx Client Errors

| Code | Meaning | Use Case |
|------|---------|----------|
| **400** | Bad Request | Invalid input data |
| **401** | Unauthorized | Missing/invalid authentication |
| **403** | Forbidden | Authenticated but not authorized |
| **404** | Not Found | Resource doesn't exist |
| **405** | Method Not Allowed | Wrong HTTP method |
| **409** | Conflict | Resource conflict (duplicate) |
| **422** | Unprocessable Entity | Validation failed |
| **429** | Too Many Requests | Rate limit exceeded |

### 5xx Server Errors

| Code | Meaning | Use Case |
|------|---------|----------|
| **500** | Internal Server Error | Unexpected error |
| **502** | Bad Gateway | Upstream server error |
| **503** | Service Unavailable | Temporary unavailability |
| **504** | Gateway Timeout | Upstream timeout |

---

## 📝 Example 1: Complete Banking Account API

### Account Model

```javascript
// src/models/account.js

class Account {
  constructor(data) {
    this.id = data.id;
    this.accountNumber = data.accountNumber;
    this.customerId = data.customerId;
    this.accountType = data.accountType; // CHECKING, SAVINGS
    this.currency = data.currency;
    this.balance = data.balance;
    this.status = data.status; // ACTIVE, CLOSED, FROZEN
    this.createdAt = data.createdAt;
    this.updatedAt = data.updatedAt;
  }

  toJSON() {
    return {
      id: this.id,
      accountNumber: this.maskAccountNumber(),
      customerId: this.customerId,
      accountType: this.accountType,
      currency: this.currency,
      balance: parseFloat(this.balance),
      status: this.status,
      createdAt: this.createdAt,
      updatedAt: this.updatedAt
    };
  }

  maskAccountNumber() {
    const visible = this.accountNumber.slice(-4);
    return `****${visible}`;
  }
}

module.exports = Account;
```

### RESTful Account Controller

```javascript
// src/controllers/accountController.js

const Account = require('../models/account');
const logger = require('../logger');

class AccountController {
  /**
   * GET /api/accounts
   * List all accounts with filtering, pagination
   */
  async listAccounts(req, res) {
    try {
      const {
        status,
        accountType,
        customerId,
        page = 1,
        limit = 10,
        sortBy = 'createdAt',
        order = 'desc'
      } = req.query;

      // Build query
      const filters = [];
      const params = [];
      let paramIndex = 1;

      if (status) {
        filters.push(`status = $${paramIndex++}`);
        params.push(status);
      }

      if (accountType) {
        filters.push(`account_type = $${paramIndex++}`);
        params.push(accountType);
      }

      if (customerId) {
        filters.push(`customer_id = $${paramIndex++}`);
        params.push(customerId);
      }

      const whereClause = filters.length > 0 ? `WHERE ${filters.join(' AND ')}` : '';
      const offset = (page - 1) * limit;

      // Query accounts
      const query = `
        SELECT * FROM accounts
        ${whereClause}
        ORDER BY ${sortBy} ${order}
        LIMIT $${paramIndex++} OFFSET $${paramIndex}
      `;
      params.push(limit, offset);

      const result = await req.db.query(query, params);

      // Count total
      const countQuery = `SELECT COUNT(*) FROM accounts ${whereClause}`;
      const countResult = await req.db.query(countQuery, params.slice(0, -2));
      const total = parseInt(countResult.rows[0].count);

      const accounts = result.rows.map(row => new Account(row).toJSON());

      // Response with pagination metadata
      res.status(200).json({
        data: accounts,
        pagination: {
          page: parseInt(page),
          limit: parseInt(limit),
          total,
          totalPages: Math.ceil(total / limit)
        },
        _links: {
          self: `/api/accounts?page=${page}&limit=${limit}`,
          first: `/api/accounts?page=1&limit=${limit}`,
          last: `/api/accounts?page=${Math.ceil(total / limit)}&limit=${limit}`,
          next: page < Math.ceil(total / limit) ? `/api/accounts?page=${parseInt(page) + 1}&limit=${limit}` : null,
          prev: page > 1 ? `/api/accounts?page=${parseInt(page) - 1}&limit=${limit}` : null
        }
      });

    } catch (error) {
      logger.error('Error listing accounts', { error: error.message });
      res.status(500).json({ error: 'Internal server error' });
    }
  }

  /**
   * GET /api/accounts/:id
   * Get specific account
   */
  async getAccount(req, res) {
    try {
      const { id } = req.params;

      const result = await req.db.query(
        'SELECT * FROM accounts WHERE id = $1',
        [id]
      );

      if (result.rows.length === 0) {
        return res.status(404).json({
          error: 'Account not found',
          message: `Account with ID ${id} does not exist`
        });
      }

      const account = new Account(result.rows[0]);

      // HATEOAS links
      res.status(200).json({
        data: account.toJSON(),
        _links: {
          self: `/api/accounts/${id}`,
          transactions: `/api/accounts/${id}/transactions`,
          transfers: `/api/accounts/${id}/transfers`,
          statements: `/api/accounts/${id}/statements`,
          update: { href: `/api/accounts/${id}`, method: 'PATCH' },
          delete: { href: `/api/accounts/${id}`, method: 'DELETE' }
        }
      });

    } catch (error) {
      logger.error('Error getting account', { error: error.message });
      res.status(500).json({ error: 'Internal server error' });
    }
  }

  /**
   * POST /api/accounts
   * Create new account
   */
  async createAccount(req, res) {
    try {
      const { customerId, accountType, currency = 'USD' } = req.body;

      // Validation
      if (!customerId || !accountType) {
        return res.status(400).json({
          error: 'Bad Request',
          message: 'customerId and accountType are required'
        });
      }

      if (!['CHECKING', 'SAVINGS'].includes(accountType)) {
        return res.status(400).json({
          error: 'Bad Request',
          message: 'accountType must be CHECKING or SAVINGS'
        });
      }

      // Generate account number
      const accountNumber = this.generateAccountNumber();

      // Insert account
      const result = await req.db.query(
        `INSERT INTO accounts (account_number, customer_id, account_type, currency, balance, status)
         VALUES ($1, $2, $3, $4, 0, 'ACTIVE')
         RETURNING *`,
        [accountNumber, customerId, accountType, currency]
      );

      const account = new Account(result.rows[0]);

      logger.info('Account created', { accountId: account.id });

      // 201 Created with Location header
      res.status(201)
        .location(`/api/accounts/${account.id}`)
        .json({
          data: account.toJSON(),
          _links: {
            self: `/api/accounts/${account.id}`
          }
        });

    } catch (error) {
      logger.error('Error creating account', { error: error.message });
      
      if (error.code === '23505') { // Unique violation
        return res.status(409).json({
          error: 'Conflict',
          message: 'Account already exists'
        });
      }
      
      res.status(500).json({ error: 'Internal server error' });
    }
  }

  /**
   * PATCH /api/accounts/:id
   * Partial update account
   */
  async updateAccount(req, res) {
    try {
      const { id } = req.params;
      const updates = req.body;

      // Validate allowed fields
      const allowedFields = ['status', 'accountType'];
      const updateFields = Object.keys(updates);
      const invalidFields = updateFields.filter(f => !allowedFields.includes(f));

      if (invalidFields.length > 0) {
        return res.status(400).json({
          error: 'Bad Request',
          message: `Invalid fields: ${invalidFields.join(', ')}`,
          allowedFields
        });
      }

      if (updates.status && !['ACTIVE', 'CLOSED', 'FROZEN'].includes(updates.status)) {
        return res.status(400).json({
          error: 'Bad Request',
          message: 'status must be ACTIVE, CLOSED, or FROZEN'
        });
      }

      // Build update query
      const setClauses = [];
      const params = [];
      let paramIndex = 1;

      updateFields.forEach(field => {
        setClauses.push(`${field.replace(/([A-Z])/g, '_$1').toLowerCase()} = $${paramIndex++}`);
        params.push(updates[field]);
      });

      params.push(id);

      const result = await req.db.query(
        `UPDATE accounts
         SET ${setClauses.join(', ')}, updated_at = NOW()
         WHERE id = $${paramIndex}
         RETURNING *`,
        params
      );

      if (result.rows.length === 0) {
        return res.status(404).json({ error: 'Account not found' });
      }

      const account = new Account(result.rows[0]);

      logger.info('Account updated', { accountId: id, updates });

      res.status(200).json({
        data: account.toJSON(),
        _links: {
          self: `/api/accounts/${id}`
        }
      });

    } catch (error) {
      logger.error('Error updating account', { error: error.message });
      res.status(500).json({ error: 'Internal server error' });
    }
  }

  /**
   * DELETE /api/accounts/:id
   * Delete account (soft delete)
   */
  async deleteAccount(req, res) {
    try {
      const { id } = req.params;

      // Check if account exists and has zero balance
      const result = await req.db.query(
        'SELECT balance FROM accounts WHERE id = $1',
        [id]
      );

      if (result.rows.length === 0) {
        return res.status(404).json({ error: 'Account not found' });
      }

      const balance = parseFloat(result.rows[0].balance);
      if (balance !== 0) {
        return res.status(400).json({
          error: 'Bad Request',
          message: 'Cannot delete account with non-zero balance',
          balance
        });
      }

      // Soft delete (update status)
      await req.db.query(
        "UPDATE accounts SET status = 'CLOSED', updated_at = NOW() WHERE id = $1",
        [id]
      );

      logger.info('Account deleted', { accountId: id });

      // 204 No Content (no body)
      res.status(204).send();

    } catch (error) {
      logger.error('Error deleting account', { error: error.message });
      res.status(500).json({ error: 'Internal server error' });
    }
  }

  /**
   * GET /api/accounts/:id/balance
   * Get account balance
   */
  async getBalance(req, res) {
    try {
      const { id } = req.params;

      const result = await req.db.query(
        'SELECT balance, currency FROM accounts WHERE id = $1',
        [id]
      );

      if (result.rows.length === 0) {
        return res.status(404).json({ error: 'Account not found' });
      }

      res.status(200).json({
        accountId: id,
        balance: parseFloat(result.rows[0].balance),
        currency: result.rows[0].currency,
        timestamp: new Date().toISOString()
      });

    } catch (error) {
      logger.error('Error getting balance', { error: error.message });
      res.status(500).json({ error: 'Internal server error' });
    }
  }

  generateAccountNumber() {
    return `ACC${Date.now()}${Math.floor(Math.random() * 10000)}`;
  }
}

module.exports = new AccountController();
```

### Express Routes

```javascript
// src/routes/accounts.js

const express = require('express');
const accountController = require('../controllers/accountController');

const router = express.Router();

// Account CRUD
router.get('/accounts', accountController.listAccounts.bind(accountController));
router.get('/accounts/:id', accountController.getAccount.bind(accountController));
router.post('/accounts', accountController.createAccount.bind(accountController));
router.patch('/accounts/:id', accountController.updateAccount.bind(accountController));
router.delete('/accounts/:id', accountController.deleteAccount.bind(accountController));

// Nested resources
router.get('/accounts/:id/balance', accountController.getBalance.bind(accountController));

module.exports = router;
```

### Sample API Responses

**GET /api/accounts?page=1&limit=2**
```json
{
  "data": [
    {
      "id": "123",
      "accountNumber": "****6789",
      "customerId": "CUST-001",
      "accountType": "CHECKING",
      "currency": "USD",
      "balance": 5000.00,
      "status": "ACTIVE",
      "createdAt": "2024-01-15T10:00:00Z",
      "updatedAt": "2024-01-15T10:00:00Z"
    },
    {
      "id": "124",
      "accountNumber": "****1234",
      "customerId": "CUST-002",
      "accountType": "SAVINGS",
      "currency": "USD",
      "balance": 10000.00,
      "status": "ACTIVE",
      "createdAt": "2024-01-15T11:00:00Z",
      "updatedAt": "2024-01-15T11:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 2,
    "total": 10,
    "totalPages": 5
  },
  "_links": {
    "self": "/api/accounts?page=1&limit=2",
    "first": "/api/accounts?page=1&limit=2",
    "last": "/api/accounts?page=5&limit=2",
    "next": "/api/accounts?page=2&limit=2",
    "prev": null
  }
}
```

**GET /api/accounts/123**
```json
{
  "data": {
    "id": "123",
    "accountNumber": "****6789",
    "customerId": "CUST-001",
    "accountType": "CHECKING",
    "currency": "USD",
    "balance": 5000.00,
    "status": "ACTIVE",
    "createdAt": "2024-01-15T10:00:00Z",
    "updatedAt": "2024-01-15T10:00:00Z"
  },
  "_links": {
    "self": "/api/accounts/123",
    "transactions": "/api/accounts/123/transactions",
    "transfers": "/api/accounts/123/transfers",
    "statements": "/api/accounts/123/statements",
    "update": { "href": "/api/accounts/123", "method": "PATCH" },
    "delete": { "href": "/api/accounts/123", "method": "DELETE" }
  }
}
```

**POST /api/accounts**
```json
// Request
{
  "customerId": "CUST-003",
  "accountType": "CHECKING",
  "currency": "USD"
}

// Response: 201 Created
// Location: /api/accounts/125
{
  "data": {
    "id": "125",
    "accountNumber": "****5678",
    "customerId": "CUST-003",
    "accountType": "CHECKING",
    "currency": "USD",
    "balance": 0,
    "status": "ACTIVE",
    "createdAt": "2024-01-15T12:00:00Z",
    "updatedAt": "2024-01-15T12:00:00Z"
  },
  "_links": {
    "self": "/api/accounts/125"
  }
}
```

---

## 🎯 Key Takeaways

### RESTful API Design Checklist

- ✅ **Use nouns for resources** (`/accounts` not `/getAccounts`)
- ✅ **Plural resource names** (`/accounts` not `/account`)
- ✅ **Proper HTTP methods** (GET for read, POST for create, etc.)
- ✅ **Appropriate status codes** (201 for created, 404 for not found)
- ✅ **Hierarchical URIs** (`/accounts/123/transactions`)
- ✅ **Filtering via query params** (`?status=active&type=checking`)
- ✅ **Pagination metadata** (page, limit, total, totalPages)
- ✅ **HATEOAS links** (discoverability)
- ✅ **Versioning** (`/api/v1/accounts`)
- ✅ **Error responses** (consistent error format)

### Best Practices

1. **Consistent error format**
   ```json
   {
     "error": "Bad Request",
     "message": "customerId is required",
     "field": "customerId",
     "code": "MISSING_REQUIRED_FIELD"
   }
   ```

2. **Pagination for large datasets**
   ```
   GET /api/accounts?page=2&limit=50
   ```

3. **Filtering and sorting**
   ```
   GET /api/transactions?status=completed&sortBy=createdAt&order=desc
   ```

4. **Include metadata**
   ```json
   {
     "data": [...],
     "pagination": {...},
     "_links": {...}
   }
   ```

5. **Use HTTP caching**
   ```
   Cache-Control: max-age=3600
   ETag: "abc123"
   ```

---

**File**: `Q37_API_Design_RESTful_Principles.md`  
**Status**: ✅ Complete with REST constraints, resource design, HTTP methods, status codes, pagination, HATEOAS, and complete banking account API
