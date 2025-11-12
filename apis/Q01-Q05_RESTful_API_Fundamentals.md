# APIs Questions 1-5: RESTful API Fundamentals

---

### Q1. What are RESTful APIs and what are the key principles for banking applications?

**Answer:**

REST (Representational State Transfer) is an architectural style for designing networked applications. For banking, RESTful APIs must prioritize security, reliability, and compliance.

**Key REST Principles:**

```javascript
/**
 * RESTful Banking API Design
 */
class RESTfulBankingAPI {
  getRESTPrinciples() {
    return {
      // 1. Client-Server Architecture
      clientServer: {
        description: 'Separation of concerns between client and server',
        example: 'Mobile app (client) calls banking API (server)',
        benefits: ['Independent evolution', 'Scalability', 'Platform independence']
      },
      
      // 2. Stateless
      stateless: {
        description: 'Each request contains all information needed',
        implementation: 'JWT token in Authorization header',
        example: `
GET /api/accounts/123/balance
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
        `.trim(),
        benefits: ['Scalability', 'Reliability', 'Easy caching']
      },
      
      // 3. Cacheable
      cacheable: {
        description: 'Responses explicitly indicate if they can be cached',
        headers: {
          'Cache-Control': 'private, max-age=300',
          'ETag': '"33a64df551425fcc55e4d42a148795d9f25f89d4"',
          'Last-Modified': 'Wed, 21 Oct 2024 07:28:00 GMT'
        },
        example: 'Cache account balance for 5 minutes'
      },
      
      // 4. Uniform Interface
      uniformInterface: {
        description: 'Consistent resource identification and manipulation',
        components: [
          'Resource identification via URIs',
          'Resource manipulation through representations',
          'Self-descriptive messages',
          'HATEOAS (Hypermedia as the Engine of Application State)'
        ]
      },
      
      // 5. Layered System
      layeredSystem: {
        description: 'Client cannot tell if connected directly to server',
        layers: ['Load Balancer', 'API Gateway', 'WAF', 'Caching Layer', 'Application Server'],
        benefits: ['Security', 'Scalability', 'Legacy integration']
      },
      
      // 6. Code on Demand (Optional)
      codeOnDemand: {
        description: 'Server can extend client functionality',
        example: 'Sending JavaScript widgets for custom calculators',
        usage: 'Rarely used in banking for security reasons'
      }
    };
  }
  
  /**
   * RESTful Resource Design for Banking
   */
  getResourceDesign() {
    return {
      // Resources are nouns, not verbs
      goodExamples: [
        '/api/customers',
        '/api/customers/123',
        '/api/customers/123/accounts',
        '/api/accounts/456/transactions',
        '/api/loans/789/payments'
      ],
      
      badExamples: [
        '/api/getCustomer',          // ❌ Verb in URL
        '/api/customer/get/123',     // ❌ Verb in URL
        '/api/deleteAccount',        // ❌ Use DELETE method instead
        '/api/accounts/withdraw'     // ❌ Use POST /transactions
      ],
      
      // HTTP Methods
      httpMethods: {
        GET: {
          purpose: 'Retrieve resource(s)',
          idempotent: true,
          safe: true,
          examples: [
            'GET /api/customers/123',
            'GET /api/accounts?customerId=123',
            'GET /api/transactions?accountId=456&limit=10'
          ]
        },
        
        POST: {
          purpose: 'Create new resource',
          idempotent: false,
          safe: false,
          examples: [
            'POST /api/customers (create customer)',
            'POST /api/accounts (open account)',
            'POST /api/transactions (make transaction)'
          ],
          response: '201 Created with Location header'
        },
        
        PUT: {
          purpose: 'Update entire resource (replace)',
          idempotent: true,
          safe: false,
          examples: [
            'PUT /api/customers/123 (update all customer fields)',
            'PUT /api/accounts/456/preferences'
          ],
          response: '200 OK or 204 No Content'
        },
        
        PATCH: {
          purpose: 'Partial update',
          idempotent: true,
          safe: false,
          examples: [
            'PATCH /api/customers/123 (update email only)',
            'PATCH /api/accounts/456 (update status)'
          ],
          response: '200 OK'
        },
        
        DELETE: {
          purpose: 'Remove resource',
          idempotent: true,
          safe: false,
          examples: [
            'DELETE /api/customers/123/addresses/789',
            'DELETE /api/beneficiaries/456'
          ],
          response: '204 No Content or 200 OK'
        }
      },
      
      // HTTP Status Codes
      statusCodes: {
        success: {
          200: 'OK - Request succeeded',
          201: 'Created - Resource created successfully',
          202: 'Accepted - Request accepted for async processing',
          204: 'No Content - Success but no body to return'
        },
        
        clientErrors: {
          400: 'Bad Request - Invalid request data',
          401: 'Unauthorized - Missing or invalid authentication',
          403: 'Forbidden - Authenticated but not authorized',
          404: 'Not Found - Resource does not exist',
          409: 'Conflict - Resource already exists or version conflict',
          422: 'Unprocessable Entity - Validation failed',
          429: 'Too Many Requests - Rate limit exceeded'
        },
        
        serverErrors: {
          500: 'Internal Server Error - Unexpected server error',
          502: 'Bad Gateway - Invalid response from upstream',
          503: 'Service Unavailable - Temporary overload or maintenance',
          504: 'Gateway Timeout - Upstream server timeout'
        }
      }
    };
  }
}

/**
 * ENBD Banking API Implementation
 */
const express = require('express');
const router = express.Router();

// GET /api/customers/:id
router.get('/customers/:id', async (req, res) => {
  try {
    const { id } = req.params;
    
    const customer = await db.query(
      'SELECT id, name, email, phone FROM customers WHERE id = $1',
      [id]
    );
    
    if (!customer.rows[0]) {
      return res.status(404).json({
        error: 'CustomerNotFound',
        message: 'Customer with specified ID does not exist',
        requestId: req.id
      });
    }
    
    res.status(200).json({
      data: customer.rows[0],
      links: {
        self: `/api/customers/${id}`,
        accounts: `/api/customers/${id}/accounts`,
        transactions: `/api/customers/${id}/transactions`
      }
    });
  } catch (error) {
    console.error('Error fetching customer:', error);
    res.status(500).json({
      error: 'InternalServerError',
      message: 'An unexpected error occurred',
      requestId: req.id
    });
  }
});

// POST /api/accounts
router.post('/accounts', async (req, res) => {
  try {
    const { customerId, accountType, currency } = req.body;
    
    // Validation
    if (!customerId || !accountType) {
      return res.status(400).json({
        error: 'ValidationError',
        message: 'customerId and accountType are required',
        fields: {
          customerId: !customerId ? 'Required field' : null,
          accountType: !accountType ? 'Required field' : null
        }
      });
    }
    
    // Check if customer exists
    const customerExists = await db.query(
      'SELECT id FROM customers WHERE id = $1',
      [customerId]
    );
    
    if (!customerExists.rows[0]) {
      return res.status(404).json({
        error: 'CustomerNotFound',
        message: 'Customer does not exist'
      });
    }
    
    // Create account
    const accountNumber = generateAccountNumber();
    
    const result = await db.query(`
      INSERT INTO accounts (customer_id, account_number, account_type, currency, balance)
      VALUES ($1, $2, $3, $4, 0)
      RETURNING id, account_number, account_type, currency, balance, created_at
    `, [customerId, accountNumber, accountType, currency || 'AED']);
    
    const account = result.rows[0];
    
    res.status(201)
      .location(`/api/accounts/${account.id}`)
      .json({
        data: account,
        links: {
          self: `/api/accounts/${account.id}`,
          customer: `/api/customers/${customerId}`,
          transactions: `/api/accounts/${account.id}/transactions`
        }
      });
  } catch (error) {
    console.error('Error creating account:', error);
    res.status(500).json({
      error: 'InternalServerError',
      message: 'Failed to create account',
      requestId: req.id
    });
  }
});

// PATCH /api/customers/:id
router.patch('/customers/:id', async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;
    
    // Build dynamic UPDATE query
    const allowedFields = ['email', 'phone', 'address'];
    const updateFields = [];
    const values = [];
    let paramIndex = 1;
    
    for (const [key, value] of Object.entries(updates)) {
      if (allowedFields.includes(key)) {
        updateFields.push(`${key} = $${paramIndex}`);
        values.push(value);
        paramIndex++;
      }
    }
    
    if (updateFields.length === 0) {
      return res.status(400).json({
        error: 'ValidationError',
        message: 'No valid fields to update',
        allowedFields
      });
    }
    
    values.push(id);
    
    const result = await db.query(`
      UPDATE customers
      SET ${updateFields.join(', ')}, updated_at = NOW()
      WHERE id = $${paramIndex}
      RETURNING id, name, email, phone, address, updated_at
    `, values);
    
    if (!result.rows[0]) {
      return res.status(404).json({
        error: 'CustomerNotFound',
        message: 'Customer does not exist'
      });
    }
    
    res.status(200).json({
      data: result.rows[0],
      links: {
        self: `/api/customers/${id}`
      }
    });
  } catch (error) {
    console.error('Error updating customer:', error);
    res.status(500).json({
      error: 'InternalServerError',
      message: 'Failed to update customer'
    });
  }
});

// DELETE /api/beneficiaries/:id
router.delete('/beneficiaries/:id', async (req, res) => {
  try {
    const { id } = req.params;
    const customerId = req.user.customerId; // From JWT
    
    // Check ownership
    const beneficiary = await db.query(
      'SELECT id FROM beneficiaries WHERE id = $1 AND customer_id = $2',
      [id, customerId]
    );
    
    if (!beneficiary.rows[0]) {
      return res.status(404).json({
        error: 'BeneficiaryNotFound',
        message: 'Beneficiary not found or access denied'
      });
    }
    
    await db.query('DELETE FROM beneficiaries WHERE id = $1', [id]);
    
    res.status(204).send();
  } catch (error) {
    console.error('Error deleting beneficiary:', error);
    res.status(500).json({
      error: 'InternalServerError',
      message: 'Failed to delete beneficiary'
    });
  }
});

function generateAccountNumber() {
  return '1234' + Math.random().toString().substr(2, 12);
}

module.exports = router;
```

---

### Q2. How do you implement API versioning strategies?

**Answer:**

```javascript
/**
 * API Versioning Strategies for Banking
 */
class APIVersioningStrategies {
  /**
   * Strategy 1: URI Versioning (Most Common)
   */
  getURIVersioning() {
    return {
      description: 'Version in URL path',
      
      examples: {
        v1: 'https://api.enbd.com/v1/accounts',
        v2: 'https://api.enbd.com/v2/accounts',
        v3: 'https://api.enbd.com/v3/accounts'
      },
      
      implementation: `
const express = require('express');
const app = express();

// Version 1 routes
const v1Router = express.Router();
v1Router.get('/accounts/:id', (req, res) => {
  res.json({
    version: 'v1',
    account: {
      id: req.params.id,
      balance: 10000
    }
  });
});

// Version 2 routes (with enhanced response)
const v2Router = express.Router();
v2Router.get('/accounts/:id', (req, res) => {
  res.json({
    version: 'v2',
    account: {
      id: req.params.id,
      accountNumber: '1234567890',
      balance: {
        available: 10000,
        pending: 500,
        total: 10500
      },
      currency: 'AED'
    }
  });
});

app.use('/api/v1', v1Router);
app.use('/api/v2', v2Router);
      `.trim(),
      
      pros: [
        'Clear and explicit',
        'Easy to route',
        'Good for public APIs',
        'Easy to cache'
      ],
      
      cons: [
        'URLs change with versions',
        'Multiple endpoints to maintain'
      ]
    };
  }
  
  /**
   * Strategy 2: Header Versioning
   */
  getHeaderVersioning() {
    return {
      description: 'Version in custom header',
      
      example: {
        request: `
GET /api/accounts/123
Accept: application/vnd.enbd.v2+json
API-Version: 2
        `.trim(),
        
        implementation: `
const express = require('express');
const app = express();

// Version middleware
function versionMiddleware(req, res, next) {
  const version = req.headers['api-version'] || 
                  req.headers['accept'].match(/v(\\d+)/)?.[1] || 
                  '1';
  
  req.apiVersion = version;
  next();
}

app.use(versionMiddleware);

app.get('/api/accounts/:id', (req, res) => {
  if (req.apiVersion === '2') {
    // V2 response
    res.json({
      account: {
        id: req.params.id,
        balance: { available: 10000, pending: 500 }
      }
    });
  } else {
    // V1 response
    res.json({
      account: {
        id: req.params.id,
        balance: 10000
      }
    });
  }
});
        `.trim()
      },
      
      pros: [
        'Clean URLs',
        'RESTful',
        'Single endpoint'
      ],
      
      cons: [
        'Less visible',
        'Harder to test',
        'Client must set headers'
      ]
    };
  }
  
  /**
   * Strategy 3: Query Parameter Versioning
   */
  getQueryParameterVersioning() {
    return {
      description: 'Version as query parameter',
      
      examples: {
        v1: 'https://api.enbd.com/accounts?version=1',
        v2: 'https://api.enbd.com/accounts?version=2&id=123'
      },
      
      implementation: `
app.get('/api/accounts', (req, res) => {
  const version = req.query.version || '1';
  const accountId = req.query.id;
  
  if (version === '2') {
    // V2 logic
    res.json({ version: 'v2', data: {} });
  } else {
    // V1 logic
    res.json({ version: 'v1', data: {} });
  }
});
      `.trim(),
      
      pros: [
        'Easy to implement',
        'Easy to test'
      ],
      
      cons: [
        'Pollutes query parameters',
        'Caching issues',
        'Not RESTful'
      ]
    };
  }
  
  /**
   * Strategy 4: Content Negotiation (Accept Header)
   */
  getContentNegotiation() {
    return {
      description: 'Version via Accept header',
      
      example: {
        request: `
GET /api/accounts/123
Accept: application/vnd.enbd.v2+json
        `.trim(),
        
        implementation: `
app.get('/api/accounts/:id', (req, res) => {
  const acceptHeader = req.headers['accept'] || '';
  
  if (acceptHeader.includes('vnd.enbd.v2')) {
    res.type('application/vnd.enbd.v2+json');
    res.json({ version: 'v2', account: {} });
  } else {
    res.type('application/json');
    res.json({ version: 'v1', account: {} });
  }
});
        `.trim()
      },
      
      pros: [
        'True REST',
        'Clean URLs',
        'Standards-compliant'
      ],
      
      cons: [
        'Complex to implement',
        'Not obvious to developers'
      ]
    };
  }
}

/**
 * ENBD API Versioning Implementation (Recommended: URI Versioning)
 */
const express = require('express');
const app = express();

// Shared middleware
const authMiddleware = require('./middleware/auth');
const rateLimitMiddleware = require('./middleware/rateLimit');

app.use(express.json());
app.use(authMiddleware);

// Version 1 API
const v1Routes = require('./routes/v1');
app.use('/api/v1', rateLimitMiddleware({ max: 100 }), v1Routes);

// Version 2 API (current)
const v2Routes = require('./routes/v2');
app.use('/api/v2', rateLimitMiddleware({ max: 200 }), v2Routes);

// Version 3 API (beta)
const v3Routes = require('./routes/v3');
app.use('/api/v3/beta', rateLimitMiddleware({ max: 50 }), v3Routes);

// Default version (redirect to latest)
app.use('/api', (req, res) => {
  res.redirect(308, `/api/v2${req.url}`);
});

// Version deprecation warnings
app.use('/api/v1', (req, res, next) => {
  res.set('X-API-Warn', 'API v1 is deprecated. Please migrate to v2 by Dec 2025');
  res.set('X-API-Deprecation-Date', '2025-12-31');
  res.set('X-API-Sunset-Date', '2026-06-30');
  next();
});

/**
 * Versioning Best Practices
 */
class VersioningBestPractices {
  static getGuidelines() {
    return {
      // 1. Version early
      versionEarly: 'Start with /v1 even for first release',
      
      // 2. Semantic versioning for breaking changes
      semanticVersioning: {
        major: 'Breaking changes (v1 → v2)',
        minor: 'New features, backward compatible (handled in same version)',
        patch: 'Bug fixes (handled in same version)'
      },
      
      // 3. Deprecation policy
      deprecation: {
        announce: 'Warn 6 months before deprecation',
        headers: ['X-API-Warn', 'X-API-Deprecation-Date', 'X-API-Sunset-Date'],
        support: 'Support old version for 12 months minimum',
        documentation: 'Clear migration guide'
      },
      
      // 4. Version only on breaking changes
      breakingChanges: [
        'Removing endpoints',
        'Removing fields',
        'Changing field types',
        'Changing authentication',
        'Changing error responses'
      ],
      
      nonBreakingChanges: [
        'Adding new endpoints',
        'Adding optional fields',
        'Adding new error codes',
        'Performance improvements'
      ],
      
      // 5. Documentation
      documentation: {
        changelog: 'Maintain detailed changelog',
        migration: 'Step-by-step migration guides',
        examples: 'Code examples for each version',
        playground: 'Interactive API explorer per version'
      },
      
      // 6. Testing
      testing: {
        contract: 'Contract tests for each version',
        regression: 'Test suite for all supported versions',
        monitoring: 'Track version usage metrics'
      }
    };
  }
}

module.exports = {
  APIVersioningStrategies,
  VersioningBestPractices
};
```

---

### Q3. How do you implement comprehensive API authentication and authorization?

**Answer:**

```javascript
/**
 * API Authentication & Authorization for Banking
 */
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');
const crypto = require('crypto');

/**
 * JWT Authentication Middleware
 */
class JWTAuthMiddleware {
  constructor() {
    this.secretKey = process.env.JWT_SECRET;
    this.refreshSecretKey = process.env.JWT_REFRESH_SECRET;
  }
  
  /**
   * Generate access token (short-lived)
   */
  generateAccessToken(user) {
    return jwt.sign(
      {
        userId: user.id,
        customerId: user.customerId,
        email: user.email,
        roles: user.roles,
        permissions: user.permissions
      },
      this.secretKey,
      {
        expiresIn: '15m', // 15 minutes
        issuer: 'enbd-api',
        audience: 'enbd-banking-app'
      }
    );
  }
  
  /**
   * Generate refresh token (long-lived)
   */
  generateRefreshToken(user) {
    return jwt.sign(
      {
        userId: user.id,
        tokenVersion: user.tokenVersion // For revocation
      },
      this.refreshSecretKey,
      {
        expiresIn: '7d', // 7 days
        issuer: 'enbd-api'
      }
    );
  }
  
  /**
   * Verify access token middleware
   */
  verifyToken() {
    return async (req, res, next) => {
      try {
        const authHeader = req.headers['authorization'];
        
        if (!authHeader || !authHeader.startsWith('Bearer ')) {
          return res.status(401).json({
            error: 'Unauthorized',
            message: 'Missing or invalid authorization header'
          });
        }
        
        const token = authHeader.substring(7);
        
        const decoded = jwt.verify(token, this.secretKey, {
          issuer: 'enbd-api',
          audience: 'enbd-banking-app'
        });
        
        // Attach user info to request
        req.user = {
          userId: decoded.userId,
          customerId: decoded.customerId,
          email: decoded.email,
          roles: decoded.roles,
          permissions: decoded.permissions
        };
        
        next();
      } catch (error) {
        if (error.name === 'TokenExpiredError') {
          return res.status(401).json({
            error: 'TokenExpired',
            message: 'Access token has expired',
            expiredAt: error.expiredAt
          });
        }
        
        if (error.name === 'JsonWebTokenError') {
          return res.status(401).json({
            error: 'InvalidToken',
            message: 'Invalid access token'
          });
        }
        
        return res.status(500).json({
          error: 'InternalServerError',
          message: 'Authentication error'
        });
      }
    };
  }
  
  /**
   * Refresh token endpoint
   */
  async refreshAccessToken(req, res) {
    try {
      const { refreshToken } = req.body;
      
      if (!refreshToken) {
        return res.status(400).json({
          error: 'BadRequest',
          message: 'Refresh token is required'
        });
      }
      
      const decoded = jwt.verify(refreshToken, this.refreshSecretKey);
      
      // Check if token version matches (for revocation)
      const user = await db.query(
        'SELECT id, customer_id, email, roles, permissions, token_version FROM users WHERE id = $1',
        [decoded.userId]
      );
      
      if (!user.rows[0]) {
        return res.status(401).json({
          error: 'InvalidToken',
          message: 'User not found'
        });
      }
      
      if (user.rows[0].token_version !== decoded.tokenVersion) {
        return res.status(401).json({
          error: 'TokenRevoked',
          message: 'Refresh token has been revoked'
        });
      }
      
      // Generate new access token
      const newAccessToken = this.generateAccessToken(user.rows[0]);
      
      res.json({
        accessToken: newAccessToken,
        expiresIn: 900 // 15 minutes in seconds
      });
    } catch (error) {
      return res.status(401).json({
        error: 'InvalidToken',
        message: 'Invalid refresh token'
      });
    }
  }
}

/**
 * Role-Based Access Control (RBAC)
 */
class RBACMiddleware {
  /**
   * Check if user has required role
   */
  requireRole(...allowedRoles) {
    return (req, res, next) => {
      if (!req.user) {
        return res.status(401).json({
          error: 'Unauthorized',
          message: 'Authentication required'
        });
      }
      
      const userRoles = req.user.roles || [];
      const hasRole = allowedRoles.some(role => userRoles.includes(role));
      
      if (!hasRole) {
        return res.status(403).json({
          error: 'Forbidden',
          message: 'Insufficient permissions',
          required: allowedRoles,
          current: userRoles
        });
      }
      
      next();
    };
  }
  
  /**
   * Check if user has required permission
   */
  requirePermission(...requiredPermissions) {
    return (req, res, next) => {
      if (!req.user) {
        return res.status(401).json({
          error: 'Unauthorized',
          message: 'Authentication required'
        });
      }
      
      const userPermissions = req.user.permissions || [];
      const hasPermission = requiredPermissions.every(perm => 
        userPermissions.includes(perm)
      );
      
      if (!hasPermission) {
        return res.status(403).json({
          error: 'Forbidden',
          message: 'Insufficient permissions',
          required: requiredPermissions,
          current: userPermissions
        });
      }
      
      next();
    };
  }
  
  /**
   * Resource ownership check
   */
  requireOwnership(resourceParam = 'customerId') {
    return (req, res, next) => {
      const requestedResourceId = req.params[resourceParam] || req.body[resourceParam];
      const userCustomerId = req.user.customerId;
      
      // Admin can access any resource
      if (req.user.roles.includes('admin')) {
        return next();
      }
      
      if (requestedResourceId !== userCustomerId) {
        return res.status(403).json({
          error: 'Forbidden',
          message: 'You can only access your own resources'
        });
      }
      
      next();
    };
  }
}

/**
 * API Key Authentication (for service-to-service)
 */
class APIKeyAuth {
  constructor() {
    this.apiKeys = new Map(); // In production, use Redis or database
  }
  
  /**
   * Generate API key
   */
  generateAPIKey(clientId, permissions) {
    const apiKey = 'enbd_' + crypto.randomBytes(32).toString('hex');
    const hashedKey = crypto.createHash('sha256').update(apiKey).digest('hex');
    
    this.apiKeys.set(hashedKey, {
      clientId,
      permissions,
      createdAt: new Date(),
      lastUsed: null
    });
    
    return apiKey; // Return only once, never again
  }
  
  /**
   * Verify API key middleware
   */
  verifyAPIKey() {
    return async (req, res, next) => {
      const apiKey = req.headers['x-api-key'];
      
      if (!apiKey) {
        return res.status(401).json({
          error: 'Unauthorized',
          message: 'API key is required'
        });
      }
      
      const hashedKey = crypto.createHash('sha256').update(apiKey).digest('hex');
      const keyData = this.apiKeys.get(hashedKey);
      
      if (!keyData) {
        return res.status(401).json({
          error: 'InvalidAPIKey',
          message: 'API key is invalid'
        });
      }
      
      // Update last used
      keyData.lastUsed = new Date();
      
      req.client = {
        clientId: keyData.clientId,
        permissions: keyData.permissions
      };
      
      next();
    };
  }
}

/**
 * OAuth 2.0 Implementation
 */
class OAuth2Server {
  /**
   * Authorization Code Flow (for third-party apps)
   */
  async authorize(req, res) {
    const { client_id, redirect_uri, response_type, scope, state } = req.query;
    
    // Validate client
    const client = await this.validateClient(client_id, redirect_uri);
    
    if (!client) {
      return res.status(400).json({
        error: 'invalid_client',
        error_description: 'Invalid client credentials'
      });
    }
    
    // Show authorization page to user
    res.render('authorize', {
      clientName: client.name,
      scope: scope.split(' '),
      state
    });
  }
  
  /**
   * Exchange authorization code for access token
   */
  async token(req, res) {
    const { grant_type, code, client_id, client_secret, redirect_uri } = req.body;
    
    if (grant_type === 'authorization_code') {
      // Verify authorization code
      const authCode = await this.validateAuthCode(code, client_id);
      
      if (!authCode) {
        return res.status(400).json({
          error: 'invalid_grant',
          error_description: 'Invalid authorization code'
        });
      }
      
      // Generate tokens
      const accessToken = this.generateAccessToken(authCode.userId, authCode.scope);
      const refreshToken = this.generateRefreshToken(authCode.userId);
      
      res.json({
        access_token: accessToken,
        token_type: 'Bearer',
        expires_in: 3600,
        refresh_token: refreshToken,
        scope: authCode.scope
      });
    }
  }
  
  async validateClient(clientId, redirectUri) {
    // Validate against registered clients
    return { id: clientId, name: 'Partner App' };
  }
  
  async validateAuthCode(code, clientId) {
    // Validate code
    return { userId: '123', scope: 'read:accounts' };
  }
  
  generateAccessToken(userId, scope) {
    return jwt.sign({ userId, scope }, process.env.JWT_SECRET, { expiresIn: '1h' });
  }
  
  generateRefreshToken(userId) {
    return jwt.sign({ userId }, process.env.JWT_REFRESH_SECRET, { expiresIn: '30d' });
  }
}

/**
 * Usage Examples
 */
const express = require('express');
const router = express.Router();

const jwtAuth = new JWTAuthMiddleware();
const rbac = new RBACMiddleware();
const apiKeyAuth = new APIKeyAuth();

// Public endpoint (no auth)
router.get('/health', (req, res) => {
  res.json({ status: 'healthy' });
});

// JWT protected endpoint
router.get('/accounts/:customerId', 
  jwtAuth.verifyToken(),
  rbac.requireOwnership('customerId'),
  async (req, res) => {
    // User can only access their own accounts
    res.json({ accounts: [] });
  }
);

// Admin only endpoint
router.get('/admin/customers',
  jwtAuth.verifyToken(),
  rbac.requireRole('admin', 'super_admin'),
  async (req, res) => {
    res.json({ customers: [] });
  }
);

// Permission-based endpoint
router.post('/transactions',
  jwtAuth.verifyToken(),
  rbac.requirePermission('transactions:create'),
  async (req, res) => {
    res.json({ transaction: {} });
  }
);

// API key authentication (service-to-service)
router.post('/webhooks/process',
  apiKeyAuth.verifyAPIKey(),
  async (req, res) => {
    res.json({ processed: true });
  }
);

module.exports = router;
```

---

### Q4. How do you implement API rate limiting and throttling?

**Answer:**

```javascript
/**
 * API Rate Limiting for Banking APIs
 */
const Redis = require('ioredis');
const redis = new Redis();

/**
 * Token Bucket Algorithm
 */
class TokenBucketRateLimiter {
  constructor(options = {}) {
    this.capacity = options.capacity || 100; // tokens
    this.refillRate = options.refillRate || 10; // tokens per second
    this.redis = redis;
  }
  
  /**
   * Check if request is allowed
   */
  async checkLimit(identifier) {
    const key = `rate_limit:${identifier}`;
    const now = Date.now();
    
    // Get current bucket state
    const bucket = await this.redis.hgetall(key);
    
    let tokens = parseFloat(bucket.tokens) || this.capacity;
    let lastRefill = parseInt(bucket.lastRefill) || now;
    
    // Refill tokens based on time elapsed
    const timePassed = (now - lastRefill) / 1000;
    const tokensToAdd = timePassed * this.refillRate;
    tokens = Math.min(this.capacity, tokens + tokensToAdd);
    
    // Check if request can be made
    if (tokens >= 1) {
      tokens -= 1;
      
      // Update bucket
      await this.redis.hmset(key, {
        tokens: tokens.toString(),
        lastRefill: now.toString()
      });
      
      await this.redis.expire(key, 3600); // 1 hour TTL
      
      return {
        allowed: true,
        remaining: Math.floor(tokens),
        resetAt: now + ((this.capacity - tokens) / this.refillRate) * 1000
      };
    }
    
    return {
      allowed: false,
      remaining: 0,
      resetAt: now + ((1 - tokens) / this.refillRate) * 1000,
      retryAfter: Math.ceil((1 - tokens) / this.refillRate)
    };
  }
}

/**
 * Sliding Window Counter
 */
class SlidingWindowRateLimiter {
  constructor(options = {}) {
    this.maxRequests = options.maxRequests || 100;
    this.windowMs = options.windowMs || 60000; // 1 minute
    this.redis = redis;
  }
  
  async checkLimit(identifier) {
    const key = `rate_limit:${identifier}`;
    const now = Date.now();
    const windowStart = now - this.windowMs;
    
    // Remove old entries
    await this.redis.zremrangebyscore(key, 0, windowStart);
    
    // Count requests in current window
    const requestCount = await this.redis.zcard(key);
    
    if (requestCount < this.maxRequests) {
      // Add current request
      await this.redis.zadd(key, now, `${now}-${Math.random()}`);
      await this.redis.expire(key, Math.ceil(this.windowMs / 1000));
      
      return {
        allowed: true,
        remaining: this.maxRequests - requestCount - 1,
        resetAt: now + this.windowMs
      };
    }
    
    // Get oldest request time
    const oldest = await this.redis.zrange(key, 0, 0, 'WITHSCORES');
    const oldestTime = parseInt(oldest[1]);
    const resetAt = oldestTime + this.windowMs;
    
    return {
      allowed: false,
      remaining: 0,
      resetAt,
      retryAfter: Math.ceil((resetAt - now) / 1000)
    };
  }
}

/**
 * Tiered Rate Limiting
 */
class TieredRateLimiter {
  constructor() {
    this.tiers = {
      free: {
        perMinute: 10,
        perHour: 100,
        perDay: 1000
      },
      basic: {
        perMinute: 60,
        perHour: 1000,
        perDay: 10000
      },
      premium: {
        perMinute: 300,
        perHour: 5000,
        perDay: 50000
      },
      enterprise: {
        perMinute: 1000,
        perHour: 20000,
        perDay: 200000
      }
    };
    
    this.redis = redis;
  }
  
  async checkLimit(identifier, tier = 'free') {
    const limits = this.tiers[tier];
    
    if (!limits) {
      throw new Error(`Invalid tier: ${tier}`);
    }
    
    // Check all time windows
    const checks = await Promise.all([
      this.checkWindow(identifier, 'minute', limits.perMinute, 60),
      this.checkWindow(identifier, 'hour', limits.perHour, 3600),
      this.checkWindow(identifier, 'day', limits.perDay, 86400)
    ]);
    
    // If any window is exceeded, deny request
    const exceeded = checks.find(check => !check.allowed);
    
    if (exceeded) {
      return exceeded;
    }
    
    return checks[0]; // Return minute window stats
  }
  
  async checkWindow(identifier, window, limit, seconds) {
    const key = `rate_limit:${identifier}:${window}`;
    const current = await this.redis.incr(key);
    
    if (current === 1) {
      await this.redis.expire(key, seconds);
    }
    
    const ttl = await this.redis.ttl(key);
    
    return {
      allowed: current <= limit,
      remaining: Math.max(0, limit - current),
      resetAt: Date.now() + (ttl * 1000),
      retryAfter: current > limit ? ttl : 0,
      window
    };
  }
}

/**
 * Rate Limiting Middleware
 */
function rateLimitMiddleware(options = {}) {
  const limiter = options.algorithm === 'token-bucket'
    ? new TokenBucketRateLimiter(options)
    : new SlidingWindowRateLimiter(options);
  
  return async (req, res, next) => {
    try {
      // Identify user (by JWT user ID or IP)
      const identifier = req.user?.userId || req.ip;
      
      const result = await limiter.checkLimit(identifier);
      
      // Set rate limit headers
      res.set('X-RateLimit-Limit', options.maxRequests || 100);
      res.set('X-RateLimit-Remaining', result.remaining);
      res.set('X-RateLimit-Reset', new Date(result.resetAt).toISOString());
      
      if (!result.allowed) {
        res.set('Retry-After', result.retryAfter);
        
        return res.status(429).json({
          error: 'TooManyRequests',
          message: 'Rate limit exceeded',
          retryAfter: result.retryAfter,
          resetAt: new Date(result.resetAt).toISOString()
        });
      }
      
      next();
    } catch (error) {
      console.error('Rate limit error:', error);
      // Fail open (allow request) rather than fail closed
      next();
    }
  };
}

/**
 * Tiered Rate Limiting Middleware
 */
function tieredRateLimitMiddleware() {
  const limiter = new TieredRateLimiter();
  
  return async (req, res, next) => {
    try {
      const identifier = req.user?.userId || req.ip;
      const tier = req.user?.tier || 'free'; // From JWT or user profile
      
      const result = await limiter.checkLimit(identifier, tier);
      
      res.set('X-RateLimit-Tier', tier);
      res.set('X-RateLimit-Remaining', result.remaining);
      res.set('X-RateLimit-Reset', new Date(result.resetAt).toISOString());
      
      if (!result.allowed) {
        res.set('Retry-After', result.retryAfter);
        
        return res.status(429).json({
          error: 'TooManyRequests',
          message: `Rate limit exceeded for ${tier} tier`,
          tier,
          window: result.window,
          retryAfter: result.retryAfter
        });
      }
      
      next();
    } catch (error) {
      console.error('Rate limit error:', error);
      next();
    }
  };
}

/**
 * ENBD Banking API with Rate Limiting
 */
const express = require('express');
const app = express();

// Global rate limit (DDoS protection)
app.use(rateLimitMiddleware({
  maxRequests: 1000,
  windowMs: 60000, // 1 minute
  algorithm: 'sliding-window'
}));

// Authentication endpoints (stricter limits)
app.use('/api/auth', rateLimitMiddleware({
  maxRequests: 5,
  windowMs: 60000 // 5 requests per minute
}));

// Public APIs
app.use('/api/v1/public', rateLimitMiddleware({
  maxRequests: 100,
  windowMs: 60000
}));

// Authenticated APIs (tiered)
app.use('/api/v1/accounts', 
  authMiddleware,
  tieredRateLimitMiddleware()
);

// Admin APIs (no limits)
app.use('/api/v1/admin',
  authMiddleware,
  requireRole('admin'),
  // No rate limiting for admin
);

module.exports = {
  TokenBucketRateLimiter,
  SlidingWindowRateLimiter,
  TieredRateLimiter,
  rateLimitMiddleware,
  tieredRateLimitMiddleware
};
```

---

### Q5. How do you implement comprehensive API error handling and logging?

**Answer:**

```javascript
/**
 * API Error Handling for Banking
 */

/**
 * Custom Error Classes
 */
class APIError extends Error {
  constructor(message, statusCode, errorCode, details = {}) {
    super(message);
    this.name = this.constructor.name;
    this.statusCode = statusCode;
    this.errorCode = errorCode;
    this.details = details;
    this.timestamp = new Date().toISOString();
    Error.captureStackTrace(this, this.constructor);
  }
  
  toJSON() {
    return {
      error: this.errorCode,
      message: this.message,
      details: this.details,
      timestamp: this.timestamp
    };
  }
}

class ValidationError extends APIError {
  constructor(message, details) {
    super(message, 400, 'ValidationError', details);
  }
}

class AuthenticationError extends APIError {
  constructor(message = 'Authentication failed') {
    super(message, 401, 'AuthenticationError');
  }
}

class AuthorizationError extends APIError {
  constructor(message = 'Insufficient permissions') {
    super(message, 403, 'Forbidden');
  }
}

class NotFoundError extends APIError {
  constructor(resource = 'Resource') {
    super(`${resource} not found`, 404, 'NotFound');
  }
}

class ConflictError extends APIError {
  constructor(message, details) {
    super(message, 409, 'Conflict', details);
  }
}

class RateLimitError extends APIError {
  constructor(retryAfter) {
    super('Rate limit exceeded', 429, 'TooManyRequests', { retryAfter });
  }
}

class InternalServerError extends APIError {
  constructor(message = 'Internal server error') {
    super(message, 500, 'InternalServerError');
  }
}

/**
 * Global Error Handler Middleware
 */
function errorHandler(err, req, res, next) {
  // Log error
  console.error('API Error:', {
    error: err.name,
    message: err.message,
    stack: err.stack,
    url: req.url,
    method: req.method,
    userId: req.user?.userId,
    requestId: req.id
  });
  
  // Handle known errors
  if (err instanceof APIError) {
    return res.status(err.statusCode).json({
      ...err.toJSON(),
      requestId: req.id
    });
  }
  
  // Handle Mongoose validation errors
  if (err.name === 'ValidationError') {
    return res.status(400).json({
      error: 'ValidationError',
      message: 'Input validation failed',
      details: Object.keys(err.errors).reduce((acc, key) => {
        acc[key] = err.errors[key].message;
        return acc;
      }, {}),
      requestId: req.id
    });
  }
  
  // Handle JWT errors
  if (err.name === 'JsonWebTokenError') {
    return res.status(401).json({
      error: 'InvalidToken',
      message: 'Invalid authentication token',
      requestId: req.id
    });
  }
  
  if (err.name === 'TokenExpiredError') {
    return res.status(401).json({
      error: 'TokenExpired',
      message: 'Authentication token has expired',
      expiredAt: err.expiredAt,
      requestId: req.id
    });
  }
  
  // Handle database errors
  if (err.code === '23505') { // PostgreSQL unique violation
    return res.status(409).json({
      error: 'Conflict',
      message: 'Resource already exists',
      requestId: req.id
    });
  }
  
  // Default error response (don't expose internal details)
  res.status(500).json({
    error: 'InternalServerError',
    message: 'An unexpected error occurred',
    requestId: req.id
  });
}

/**
 * Async Handler Wrapper
 */
function asyncHandler(fn) {
  return (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
}

/**
 * Request Logging Middleware
 */
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
    new winston.transports.Console({
      format: winston.format.simple()
    })
  ]
});

function requestLogger(req, res, next) {
  const startTime = Date.now();
  
  // Generate unique request ID
  req.id = require('crypto').randomUUID();
  
  // Log request
  logger.info('Incoming request', {
    requestId: req.id,
    method: req.method,
    url: req.url,
    ip: req.ip,
    userId: req.user?.userId,
    userAgent: req.get('user-agent')
  });
  
  // Log response
  res.on('finish', () => {
    const duration = Date.now() - startTime;
    
    logger.info('Request completed', {
      requestId: req.id,
      method: req.method,
      url: req.url,
      statusCode: res.statusCode,
      duration: `${duration}ms`,
      userId: req.user?.userId
    });
  });
  
  next();
}

/**
 * Structured Logging Service
 */
class LoggingService {
  constructor() {
    this.logger = logger;
  }
  
  info(message, meta = {}) {
    this.logger.info(message, meta);
  }
  
  warn(message, meta = {}) {
    this.logger.warn(message, meta);
  }
  
  error(message, error, meta = {}) {
    this.logger.error(message, {
      ...meta,
      error: {
        name: error.name,
        message: error.message,
        stack: error.stack
      }
    });
  }
  
  // Business event logging
  logBusinessEvent(event, data) {
    this.logger.info('Business event', {
      event,
      data,
      timestamp: new Date().toISOString()
    });
  }
  
  // Security event logging
  logSecurityEvent(event, data) {
    this.logger.warn('Security event', {
      event,
      data,
      severity: 'high',
      timestamp: new Date().toISOString()
    });
  }
  
  // Audit logging (compliance)
  logAudit(action, userId, resource, details) {
    this.logger.info('Audit log', {
      action,
      userId,
      resource,
      details,
      timestamp: new Date().toISOString()
    });
  }
}

/**
 * Usage Example
 */
const express = require('express');
const app = express();

// Middleware
app.use(express.json());
app.use(requestLogger);

// Request ID middleware
app.use((req, res, next) => {
  req.id = require('crypto').randomUUID();
  res.set('X-Request-ID', req.id);
  next();
});

const loggingService = new LoggingService();

// Routes
app.get('/api/accounts/:id', asyncHandler(async (req, res) => {
  const { id } = req.params;
  
  // Validate input
  if (!id || id.length !== 36) {
    throw new ValidationError('Invalid account ID', {
      field: 'id',
      expected: 'UUID format'
    });
  }
  
  // Fetch account
  const account = await db.query('SELECT * FROM accounts WHERE id = $1', [id]);
  
  if (!account.rows[0]) {
    throw new NotFoundError('Account');
  }
  
  // Log audit event
  loggingService.logAudit('account:view', req.user.userId, `account:${id}`, {
    action: 'viewed account details'
  });
  
  res.json({
    data: account.rows[0],
    requestId: req.id
  });
}));

app.post('/api/transactions', asyncHandler(async (req, res) => {
  const { accountId, amount, type } = req.body;
  
  // Validation
  if (!accountId || !amount || !type) {
    throw new ValidationError('Missing required fields', {
      accountId: !accountId ? 'Required' : null,
      amount: !amount ? 'Required' : null,
      type: !type ? 'Required' : null
    });
  }
  
  if (amount <= 0) {
    throw new ValidationError('Invalid amount', {
      amount: 'Must be greater than 0'
    });
  }
  
  // Check account exists and belongs to user
  const account = await db.query(
    'SELECT * FROM accounts WHERE id = $1 AND customer_id = $2',
    [accountId, req.user.customerId]
  );
  
  if (!account.rows[0]) {
    throw new NotFoundError('Account');
  }
  
  // Check sufficient balance for debit
  if (type === 'debit' && account.rows[0].balance < amount) {
    throw new ConflictError('Insufficient funds', {
      available: account.rows[0].balance,
      requested: amount
    });
  }
  
  // Create transaction
  const result = await db.query(`
    INSERT INTO transactions (account_id, amount, type)
    VALUES ($1, $2, $3)
    RETURNING *
  `, [accountId, amount, type]);
  
  // Log business event
  loggingService.logBusinessEvent('transaction:created', {
    transactionId: result.rows[0].id,
    accountId,
    amount,
    type,
    userId: req.user.userId
  });
  
  res.status(201).json({
    data: result.rows[0],
    requestId: req.id
  });
}));

// 404 handler
app.use((req, res, next) => {
  throw new NotFoundError('Endpoint');
});

// Global error handler (must be last)
app.use(errorHandler);

module.exports = {
  APIError,
  ValidationError,
  AuthenticationError,
  AuthorizationError,
  NotFoundError,
  ConflictError,
  RateLimitError,
  InternalServerError,
  errorHandler,
  asyncHandler,
  requestLogger,
  LoggingService
};
```

---

**Summary Q1-Q5:**
- RESTful API principles and resource design ✅
- API versioning strategies (URI, header, query, content negotiation) ✅
- Authentication & authorization (JWT, RBAC, API keys, OAuth 2.0) ✅
- Rate limiting (token bucket, sliding window, tiered limits) ✅
- Error handling and structured logging ✅
