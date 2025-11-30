# Q38: API Design - API Versioning Strategies

## 📋 Summary
API versioning enables backward compatibility when introducing breaking changes. This guide covers **versioning strategies** (URL, header, content negotiation), breaking vs non-breaking changes, deprecation policies, migration strategies, and comprehensive banking examples for evolving payment APIs.

---

## 🎯 What You'll Learn
- **Versioning Strategies**: URL path, header, query parameter, content negotiation
- **Breaking Changes**: What requires a new version
- **Deprecation Policy**: Sunset headers, documentation, migration timeline
- **Migration Strategies**: Supporting multiple versions, graceful transitions
- **Banking Examples**: Payment API v1 → v2 migration with backward compatibility

---

## 📖 Comprehensive Explanation

### Why API Versioning?

**Problem**: As your API evolves, you need to:
- Add new features
- Fix design flaws
- Remove deprecated functionality
- Change data structures

**Without versioning**: Breaking existing clients 💔  
**With versioning**: Maintain backward compatibility ✅

### When to Version

**Breaking Changes** (require new version):
- Removing fields from response
- Renaming fields
- Changing field types
- Changing HTTP status codes
- Removing endpoints
- Changing request validation rules

**Non-Breaking Changes** (no version bump):
- Adding new fields to response
- Adding new endpoints
- Adding optional request parameters
- Making required fields optional
- Relaxing validation rules

---

## 📚 Versioning Strategies

### 1. URL Path Versioning (Most Common)

**Format**: `/api/v1/resource`, `/api/v2/resource`

**Pros**:
- ✅ Clear and visible
- ✅ Easy to route
- ✅ Simple to test
- ✅ Browser-friendly

**Cons**:
- ❌ Violates REST principle (version isn't a resource)
- ❌ Multiple versions in codebase

**Example**:
```javascript
// Version 1
GET /api/v1/payments/123
Response: {
  "id": "123",
  "amount": 100.50,
  "status": "completed"
}

// Version 2 (changed field name)
GET /api/v2/payments/123
Response: {
  "id": "123",
  "amount": 100.50,
  "paymentStatus": "completed"  // Renamed from 'status'
}
```

### 2. Header Versioning

**Format**: Custom header like `X-API-Version: 2` or `Accept-Version: v2`

**Pros**:
- ✅ Cleaner URLs
- ✅ Follows REST principles
- ✅ Supports content negotiation

**Cons**:
- ❌ Less visible
- ❌ Harder to test in browser
- ❌ Requires header inspection

**Example**:
```javascript
GET /api/payments/123
X-API-Version: 2

Response: { ... }
```

### 3. Content Negotiation (Accept Header)

**Format**: `Accept: application/vnd.company.v2+json`

**Pros**:
- ✅ RESTful
- ✅ Standards-based
- ✅ Supports multiple representations

**Cons**:
- ❌ Complex to implement
- ❌ Not intuitive for developers
- ❌ Harder to debug

**Example**:
```javascript
GET /api/payments/123
Accept: application/vnd.banking.v2+json

Response: { ... }
```

### 4. Query Parameter Versioning

**Format**: `/api/payments?version=2`

**Pros**:
- ✅ Easy to implement
- ✅ Easy to test

**Cons**:
- ❌ Mixes versioning with filtering
- ❌ Pollutes query string
- ❌ Can be accidentally omitted

**Example**:
```javascript
GET /api/payments/123?version=2
```

### Recommendation

**🏆 URL Path Versioning** is the industry standard:
- Used by Stripe, Twilio, GitHub, AWS
- Clear and explicit
- Easy for developers
- Simple routing

---

## 📝 Example 1: Banking Payment API - Version Evolution

### Version 1 (Initial API)

```javascript
// src/routes/v1/payments.js

const express = require('express');
const router = express.Router();

class PaymentControllerV1 {
  /**
   * POST /api/v1/payments
   * Create payment (v1 format)
   */
  async createPayment(req, res) {
    try {
      const { fromAccount, toAccount, amount, description } = req.body;

      // V1 validation
      if (!fromAccount || !toAccount || !amount) {
        return res.status(400).json({
          error: 'Missing required fields'
        });
      }

      if (amount <= 0) {
        return res.status(400).json({
          error: 'Amount must be positive'
        });
      }

      // Create payment
      const payment = await this.processPayment({
        fromAccount,
        toAccount,
        amount,
        description
      });

      // V1 response format
      res.status(201).json({
        id: payment.id,
        from: fromAccount,
        to: toAccount,
        amount: amount,
        status: payment.status,  // V1: 'status' field
        created: payment.createdAt
      });

    } catch (error) {
      res.status(500).json({ error: error.message });
    }
  }

  /**
   * GET /api/v1/payments/:id
   * Get payment (v1 format)
   */
  async getPayment(req, res) {
    try {
      const { id } = req.params;
      const payment = await this.fetchPayment(id);

      if (!payment) {
        return res.status(404).json({ error: 'Payment not found' });
      }

      // V1 response format
      res.status(200).json({
        id: payment.id,
        from: payment.fromAccount,
        to: payment.toAccount,
        amount: payment.amount,
        status: payment.status,
        created: payment.createdAt
      });

    } catch (error) {
      res.status(500).json({ error: error.message });
    }
  }

  async processPayment(data) {
    // Payment processing logic
    return {
      id: `PAY-${Date.now()}`,
      fromAccount: data.fromAccount,
      toAccount: data.toAccount,
      amount: data.amount,
      status: 'pending',
      createdAt: new Date().toISOString()
    };
  }

  async fetchPayment(id) {
    // Database query
    return {
      id,
      fromAccount: 'ACC-001',
      toAccount: 'ACC-002',
      amount: 500,
      status: 'completed',
      createdAt: '2024-01-15T10:00:00Z'
    };
  }
}

const controller = new PaymentControllerV1();

router.post('/payments', controller.createPayment.bind(controller));
router.get('/payments/:id', controller.getPayment.bind(controller));

module.exports = router;
```

### Version 2 (Breaking Changes)

**Changes in V2**:
1. Renamed `status` → `paymentStatus`
2. Added `currency` field (required)
3. Changed `created` → `createdAt`
4. Added `metadata` object
5. More detailed error responses

```javascript
// src/routes/v2/payments.js

const express = require('express');
const router = express.Router();

class PaymentControllerV2 {
  /**
   * POST /api/v2/payments
   * Create payment (v2 format)
   */
  async createPayment(req, res) {
    try {
      const { fromAccount, toAccount, amount, currency, description, metadata } = req.body;

      // V2 validation - currency required
      if (!fromAccount || !toAccount || !amount || !currency) {
        return res.status(400).json({
          error: {
            code: 'MISSING_REQUIRED_FIELDS',
            message: 'Required fields: fromAccount, toAccount, amount, currency',
            fields: {
              fromAccount: !fromAccount ? 'required' : undefined,
              toAccount: !toAccount ? 'required' : undefined,
              amount: !amount ? 'required' : undefined,
              currency: !currency ? 'required' : undefined
            }
          }
        });
      }

      if (amount <= 0) {
        return res.status(422).json({
          error: {
            code: 'INVALID_AMOUNT',
            message: 'Amount must be positive',
            field: 'amount',
            value: amount
          }
        });
      }

      // Validate currency
      const validCurrencies = ['USD', 'EUR', 'GBP'];
      if (!validCurrencies.includes(currency)) {
        return res.status(422).json({
          error: {
            code: 'INVALID_CURRENCY',
            message: `Currency must be one of: ${validCurrencies.join(', ')}`,
            field: 'currency',
            value: currency
          }
        });
      }

      // Create payment
      const payment = await this.processPayment({
        fromAccount,
        toAccount,
        amount,
        currency,
        description,
        metadata: metadata || {}
      });

      // V2 response format
      res.status(201).json({
        id: payment.id,
        fromAccount,
        toAccount,
        amount,
        currency,
        paymentStatus: payment.status,  // V2: renamed from 'status'
        description: description || null,
        metadata: metadata || {},
        createdAt: payment.createdAt,   // V2: renamed from 'created'
        updatedAt: payment.updatedAt,
        _links: {
          self: `/api/v2/payments/${payment.id}`,
          cancel: `/api/v2/payments/${payment.id}/cancel`
        }
      });

    } catch (error) {
      res.status(500).json({
        error: {
          code: 'INTERNAL_ERROR',
          message: 'An unexpected error occurred'
        }
      });
    }
  }

  /**
   * GET /api/v2/payments/:id
   * Get payment (v2 format)
   */
  async getPayment(req, res) {
    try {
      const { id } = req.params;
      const payment = await this.fetchPayment(id);

      if (!payment) {
        return res.status(404).json({
          error: {
            code: 'PAYMENT_NOT_FOUND',
            message: `Payment with ID ${id} does not exist`
          }
        });
      }

      // V2 response format
      res.status(200).json({
        id: payment.id,
        fromAccount: payment.fromAccount,
        toAccount: payment.toAccount,
        amount: payment.amount,
        currency: payment.currency,
        paymentStatus: payment.status,  // V2 field name
        description: payment.description,
        metadata: payment.metadata || {},
        createdAt: payment.createdAt,
        updatedAt: payment.updatedAt,
        _links: {
          self: `/api/v2/payments/${id}`,
          cancel: payment.status === 'pending' ? `/api/v2/payments/${id}/cancel` : null
        }
      });

    } catch (error) {
      res.status(500).json({
        error: {
          code: 'INTERNAL_ERROR',
          message: 'An unexpected error occurred'
        }
      });
    }
  }

  /**
   * POST /api/v2/payments/:id/cancel
   * New endpoint in v2
   */
  async cancelPayment(req, res) {
    try {
      const { id } = req.params;
      const payment = await this.fetchPayment(id);

      if (!payment) {
        return res.status(404).json({
          error: {
            code: 'PAYMENT_NOT_FOUND',
            message: `Payment with ID ${id} does not exist`
          }
        });
      }

      if (payment.status !== 'pending') {
        return res.status(400).json({
          error: {
            code: 'CANNOT_CANCEL',
            message: `Cannot cancel payment with status: ${payment.status}`,
            currentStatus: payment.status
          }
        });
      }

      // Cancel payment
      await this.updatePaymentStatus(id, 'cancelled');

      res.status(200).json({
        id,
        paymentStatus: 'cancelled',
        cancelledAt: new Date().toISOString()
      });

    } catch (error) {
      res.status(500).json({
        error: {
          code: 'INTERNAL_ERROR',
          message: 'An unexpected error occurred'
        }
      });
    }
  }

  async processPayment(data) {
    return {
      id: `PAY-${Date.now()}`,
      fromAccount: data.fromAccount,
      toAccount: data.toAccount,
      amount: data.amount,
      currency: data.currency,
      status: 'pending',
      description: data.description,
      metadata: data.metadata,
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString()
    };
  }

  async fetchPayment(id) {
    return {
      id,
      fromAccount: 'ACC-001',
      toAccount: 'ACC-002',
      amount: 500,
      currency: 'USD',
      status: 'completed',
      description: 'Test payment',
      metadata: {},
      createdAt: '2024-01-15T10:00:00Z',
      updatedAt: '2024-01-15T10:05:00Z'
    };
  }

  async updatePaymentStatus(id, status) {
    // Database update
  }
}

const controller = new PaymentControllerV2();

router.post('/payments', controller.createPayment.bind(controller));
router.get('/payments/:id', controller.getPayment.bind(controller));
router.post('/payments/:id/cancel', controller.cancelPayment.bind(controller));

module.exports = router;
```

### Main App with Version Routing

```javascript
// src/app.js

const express = require('express');
const paymentsV1 = require('./routes/v1/payments');
const paymentsV2 = require('./routes/v2/payments');

const app = express();

app.use(express.json());

// Version routing
app.use('/api/v1', paymentsV1);
app.use('/api/v2', paymentsV2);

// Default version (latest)
app.use('/api', paymentsV2);

// Deprecation warning middleware for v1
app.use('/api/v1/*', (req, res, next) => {
  res.set('Sunset', 'Sat, 01 Jun 2024 00:00:00 GMT');
  res.set('Deprecation', 'true');
  res.set('Link', '</api/v2>; rel="successor-version"');
  next();
});

// Version info endpoint
app.get('/api/versions', (req, res) => {
  res.json({
    versions: [
      {
        version: 'v1',
        status: 'deprecated',
        sunsetDate: '2024-06-01',
        documentation: 'https://docs.example.com/v1'
      },
      {
        version: 'v2',
        status: 'stable',
        documentation: 'https://docs.example.com/v2'
      }
    ],
    latest: 'v2',
    recommended: 'v2'
  });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`API server running on port ${PORT}`);
});

module.exports = app;
```

### API Comparison

**V1 Request/Response**:
```bash
# V1 Create Payment
curl -X POST http://localhost:3000/api/v1/payments \
  -H "Content-Type: application/json" \
  -d '{
    "fromAccount": "ACC-001",
    "toAccount": "ACC-002",
    "amount": 500
  }'

# V1 Response
HTTP/1.1 201 Created
Sunset: Sat, 01 Jun 2024 00:00:00 GMT
Deprecation: true
{
  "id": "PAY-1705315822000",
  "from": "ACC-001",
  "to": "ACC-002",
  "amount": 500,
  "status": "pending",
  "created": "2024-01-15T10:30:22.000Z"
}
```

**V2 Request/Response**:
```bash
# V2 Create Payment
curl -X POST http://localhost:3000/api/v2/payments \
  -H "Content-Type: application/json" \
  -d '{
    "fromAccount": "ACC-001",
    "toAccount": "ACC-002",
    "amount": 500,
    "currency": "USD",
    "metadata": {
      "reference": "INV-123"
    }
  }'

# V2 Response
HTTP/1.1 201 Created
{
  "id": "PAY-1705315822000",
  "fromAccount": "ACC-001",
  "toAccount": "ACC-002",
  "amount": 500,
  "currency": "USD",
  "paymentStatus": "pending",
  "description": null,
  "metadata": {
    "reference": "INV-123"
  },
  "createdAt": "2024-01-15T10:30:22.000Z",
  "updatedAt": "2024-01-15T10:30:22.000Z",
  "_links": {
    "self": "/api/v2/payments/PAY-1705315822000",
    "cancel": "/api/v2/payments/PAY-1705315822000/cancel"
  }
}
```

---

## 🔄 Deprecation Strategy

### Deprecation Headers

```javascript
// Add to deprecated versions
res.set('Sunset', 'Sat, 01 Jun 2024 00:00:00 GMT');  // End-of-life date
res.set('Deprecation', 'true');                      // Mark as deprecated
res.set('Link', '</api/v2>; rel="successor-version"');  // Link to new version
```

### Deprecation Timeline

**Best Practice Timeline**:
1. **Announcement** (T-0): Announce deprecation, publish migration guide
2. **Deprecation** (T+3 months): Add deprecation headers, warning logs
3. **Sunset** (T+6 months): Stop accepting new clients on old version
4. **End-of-Life** (T+12 months): Remove old version entirely

### Migration Guide

```markdown
# Migration Guide: v1 → v2

## Breaking Changes

### 1. Field Renames
- `status` → `paymentStatus`
- `created` → `createdAt`
- `from` → `fromAccount`
- `to` → `toAccount`

### 2. New Required Fields
- `currency` is now required

### 3. Error Response Format
V1:
```json
{ "error": "Missing required fields" }
```

V2:
```json
{
  "error": {
    "code": "MISSING_REQUIRED_FIELDS",
    "message": "Required fields: fromAccount, toAccount, amount, currency"
  }
}
```

### 4. New Endpoints
- `POST /api/v2/payments/:id/cancel` - Cancel pending payment

## Migration Steps

1. Update request to include `currency` field
2. Update response parsing to use new field names
3. Update error handling to expect new error format
4. Test thoroughly in staging environment
5. Deploy with backward compatibility monitoring
```

---

## 🎯 Key Takeaways

### Versioning Best Practices

1. **Use URL path versioning**
   ```
   /api/v1/payments
   /api/v2/payments
   ```

2. **Version only when necessary**
   - Breaking changes only
   - Major redesigns
   - Don't version for every change

3. **Support multiple versions**
   - Run v1 and v2 simultaneously
   - Give clients time to migrate (6-12 months)

4. **Communicate deprecation**
   - Deprecation headers
   - Email notifications
   - Documentation updates
   - Migration guides

5. **Track version usage**
   ```javascript
   // Log which versions are being used
   logger.info('API request', {
     version: 'v1',
     endpoint: '/payments',
     clientId: 'client-123'
   });
   ```

### Version Lifecycle

```
v1 (Active) → v2 (Released) → v1 (Deprecated) → v1 (EOL) → v1 (Removed)
   0 months      3 months        6 months        12 months     Cleanup
```

### Breaking vs Non-Breaking Changes

**Breaking** (new version):
- ❌ Removing fields
- ❌ Renaming fields
- ❌ Changing types
- ❌ Adding required fields
- ❌ Removing endpoints

**Non-Breaking** (same version):
- ✅ Adding optional fields
- ✅ Adding new endpoints
- ✅ Making required fields optional
- ✅ Adding new response fields

---

**File**: `Q38_API_Design_Versioning.md`  
**Status**: ✅ Complete with versioning strategies, deprecation policies, breaking changes management, and complete v1 → v2 migration example
