 # APIs Questions 6-8: API Gateway & Documentation

---

### Q6. How do you implement an API Gateway for microservices?

**Answer:**

API Gateway acts as a single entry point for all client requests, routing them to appropriate microservices.

```javascript
/**
 * Custom API Gateway Implementation
 */
const express = require('express');
const httpProxy = require('http-proxy-middleware');
const jwt = require('jsonwebtoken');
const Redis = require('ioredis');

class APIGateway {
  constructor() {
    this.app = express();
    this.redis = new Redis();
    
    // Service registry
    this.services = {
      'customer-service': 'http://localhost:3001',
      'account-service': 'http://localhost:3002',
      'transaction-service': 'http://localhost:3003',
      'loan-service': 'http://localhost:3004',
      'notification-service': 'http://localhost:3005'
    };
    
    this.setupMiddleware();
    this.setupRoutes();
  }
  
  setupMiddleware() {
    this.app.use(express.json());
    
    // Request ID
    this.app.use((req, res, next) => {
      req.id = require('crypto').randomUUID();
      res.set('X-Request-ID', req.id);
      next();
    });
    
    // CORS
    this.app.use((req, res, next) => {
      res.header('Access-Control-Allow-Origin', '*');
      res.header('Access-Control-Allow-Methods', 'GET, POST, PUT, PATCH, DELETE, OPTIONS');
      res.header('Access-Control-Allow-Headers', 'Origin, X-Requested-With, Content-Type, Accept, Authorization');
      
      if (req.method === 'OPTIONS') {
        return res.sendStatus(200);
      }
      
      next();
    });
    
    // Logging
    this.app.use((req, res, next) => {
      console.log(`[${new Date().toISOString()}] ${req.method} ${req.url}`);
      next();
    });
  }
  
  setupRoutes() {
    // Health check
    this.app.get('/health', (req, res) => {
      res.json({ status: 'healthy', gateway: 'running' });
    });
    
    // Customer Service routes
    this.app.use('/api/customers', 
      this.authenticate.bind(this),
      this.rateLimit.bind(this),
      this.createProxy('customer-service')
    );
    
    // Account Service routes
    this.app.use('/api/accounts',
      this.authenticate.bind(this),
      this.rateLimit.bind(this),
      this.createProxy('account-service')
    );
    
    // Transaction Service routes
    this.app.use('/api/transactions',
      this.authenticate.bind(this),
      this.rateLimit.bind(this),
      this.createProxy('transaction-service')
    );
    
    // Loan Service routes
    this.app.use('/api/loans',
      this.authenticate.bind(this),
      this.rateLimit.bind(this),
      this.createProxy('loan-service')
    );
    
    // Public routes (no auth)
    this.app.use('/api/public',
      this.rateLimit.bind(this),
      this.createProxy('customer-service')
    );
  }
  
  /**
   * Authentication middleware
   */
  authenticate(req, res, next) {
    const authHeader = req.headers['authorization'];
    
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      return res.status(401).json({
        error: 'Unauthorized',
        message: 'Missing or invalid authorization header'
      });
    }
    
    const token = authHeader.substring(7);
    
    try {
      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      req.user = decoded;
      
      // Add user info to forwarded request
      req.headers['X-User-ID'] = decoded.userId;
      req.headers['X-Customer-ID'] = decoded.customerId;
      
      next();
    } catch (error) {
      return res.status(401).json({
        error: 'InvalidToken',
        message: 'Invalid or expired token'
      });
    }
  }
  
  /**
   * Rate limiting middleware
   */
  async rateLimit(req, res, next) {
    const identifier = req.user?.userId || req.ip;
    const key = `rate_limit:${identifier}`;
    
    try {
      const current = await this.redis.incr(key);
      
      if (current === 1) {
        await this.redis.expire(key, 60); // 1 minute window
      }
      
      const limit = 100;
      const remaining = Math.max(0, limit - current);
      
      res.set('X-RateLimit-Limit', limit);
      res.set('X-RateLimit-Remaining', remaining);
      
      if (current > limit) {
        return res.status(429).json({
          error: 'TooManyRequests',
          message: 'Rate limit exceeded'
        });
      }
      
      next();
    } catch (error) {
      console.error('Rate limit error:', error);
      next(); // Fail open
    }
  }
  
  /**
   * Create proxy middleware
   */
  createProxy(serviceName) {
    const target = this.services[serviceName];
    
    if (!target) {
      return (req, res) => {
        res.status(503).json({
          error: 'ServiceUnavailable',
          message: `Service ${serviceName} is not available`
        });
      };
    }
    
    return httpProxy.createProxyMiddleware({
      target,
      changeOrigin: true,
      pathRewrite: (path, req) => {
        // Remove /api prefix
        return path.replace(/^\/api\/[^/]+/, '');
      },
      onProxyReq: (proxyReq, req, res) => {
        // Add gateway headers
        proxyReq.setHeader('X-Gateway', 'ENBD-API-Gateway');
        proxyReq.setHeader('X-Request-ID', req.id);
        
        // Forward body for POST/PUT/PATCH
        if (req.body && Object.keys(req.body).length > 0) {
          const bodyData = JSON.stringify(req.body);
          proxyReq.setHeader('Content-Type', 'application/json');
          proxyReq.setHeader('Content-Length', Buffer.byteLength(bodyData));
          proxyReq.write(bodyData);
        }
      },
      onProxyRes: (proxyRes, req, res) => {
        // Add response headers
        proxyRes.headers['X-Service'] = serviceName;
      },
      onError: (err, req, res) => {
        console.error(`Proxy error for ${serviceName}:`, err.message);
        res.status(502).json({
          error: 'BadGateway',
          message: `Failed to reach ${serviceName}`,
          requestId: req.id
        });
      }
    });
  }
  
  /**
   * Service discovery (dynamic registration)
   */
  async registerService(serviceName, url) {
    this.services[serviceName] = url;
    await this.redis.hset('services', serviceName, url);
    console.log(`Service registered: ${serviceName} -> ${url}`);
  }
  
  async deregisterService(serviceName) {
    delete this.services[serviceName];
    await this.redis.hdel('services', serviceName);
    console.log(`Service deregistered: ${serviceName}`);
  }
  
  /**
   * Circuit breaker pattern
   */
  async checkServiceHealth(serviceName) {
    const target = this.services[serviceName];
    
    if (!target) return false;
    
    try {
      const response = await fetch(`${target}/health`, { timeout: 5000 });
      return response.ok;
    } catch (error) {
      return false;
    }
  }
  
  start(port = 8000) {
    this.app.listen(port, () => {
      console.log(`API Gateway running on port ${port}`);
      console.log('Registered services:', Object.keys(this.services));
    });
  }
}

/**
 * Kong API Gateway Configuration (Production-ready)
 */
class KongConfiguration {
  getKongConfig() {
    return {
      // Kong configuration via declarative config
      _format_version: "3.0",
      
      // Define services
      services: [
        {
          name: "customer-service",
          url: "http://customer-service:3001",
          retries: 5,
          connect_timeout: 60000,
          write_timeout: 60000,
          read_timeout: 60000
        },
        {
          name: "account-service",
          url: "http://account-service:3002"
        },
        {
          name: "transaction-service",
          url: "http://transaction-service:3003"
        }
      ],
      
      // Define routes
      routes: [
        {
          name: "customer-routes",
          service: { name: "customer-service" },
          paths: ["/api/customers"],
          methods: ["GET", "POST", "PUT", "PATCH", "DELETE"],
          strip_path: true
        },
        {
          name: "account-routes",
          service: { name: "account-service" },
          paths: ["/api/accounts"],
          methods: ["GET", "POST", "PUT", "PATCH", "DELETE"],
          strip_path: true
        }
      ],
      
      // Plugins
      plugins: [
        {
          name: "jwt",
          config: {
            secret_is_base64: false,
            key_claim_name: "kid",
            claims_to_verify: ["exp"]
          }
        },
        {
          name: "rate-limiting",
          config: {
            minute: 100,
            hour: 1000,
            policy: "redis",
            redis_host: "redis",
            redis_port: 6379
          }
        },
        {
          name: "cors",
          config: {
            origins: ["*"],
            methods: ["GET", "POST", "PUT", "PATCH", "DELETE"],
            headers: ["Accept", "Authorization", "Content-Type"],
            exposed_headers: ["X-Request-ID"],
            credentials: true,
            max_age: 3600
          }
        },
        {
          name: "request-transformer",
          config: {
            add: {
              headers: ["X-Gateway:Kong"]
            }
          }
        },
        {
          name: "response-transformer",
          config: {
            add: {
              headers: ["X-Gateway-Version:1.0"]
            }
          }
        }
      ]
    };
  }
  
  /**
   * Kong docker-compose.yml
   */
  getDockerCompose() {
    return `
version: '3.8'

services:
  kong-database:
    image: postgres:15
    environment:
      POSTGRES_USER: kong
      POSTGRES_DB: kong
      POSTGRES_PASSWORD: kong
    volumes:
      - kong_data:/var/lib/postgresql/data
    networks:
      - kong-net

  kong-migrations:
    image: kong:3.4
    command: kong migrations bootstrap
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-database
      KONG_PG_USER: kong
      KONG_PG_PASSWORD: kong
    depends_on:
      - kong-database
    networks:
      - kong-net

  kong:
    image: kong:3.4
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-database
      KONG_PG_USER: kong
      KONG_PG_PASSWORD: kong
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_ADMIN_LISTEN: 0.0.0.0:8001
    ports:
      - "8000:8000"  # Proxy
      - "8443:8443"  # Proxy SSL
      - "8001:8001"  # Admin API
    depends_on:
      - kong-database
      - kong-migrations
    networks:
      - kong-net

  redis:
    image: redis:7-alpine
    networks:
      - kong-net

volumes:
  kong_data:

networks:
  kong-net:
    driver: bridge
`;
  }
}

// Usage
const gateway = new APIGateway();
gateway.start(8000);

module.exports = { APIGateway, KongConfiguration };
```

---

### Q7. How do you create comprehensive API documentation with OpenAPI/Swagger?

**Answer:**

```javascript
/**
 * OpenAPI 3.0 Specification for ENBD Banking API
 */
const swaggerJsdoc = require('swagger-jsdoc');
const swaggerUi = require('swagger-ui-express');

const swaggerOptions = {
  definition: {
    openapi: '3.0.0',
    info: {
      title: 'Emirates NBD Banking API',
      version: '2.0.0',
      description: 'RESTful API for Emirates NBD banking operations',
      contact: {
        name: 'ENBD API Team',
        email: 'api@enbd.com',
        url: 'https://developer.enbd.com'
      },
      license: {
        name: 'Proprietary',
        url: 'https://enbd.com/api-license'
      }
    },
    servers: [
      {
        url: 'https://api.enbd.com/v2',
        description: 'Production server'
      },
      {
        url: 'https://sandbox-api.enbd.com/v2',
        description: 'Sandbox server'
      },
      {
        url: 'http://localhost:3000/api/v2',
        description: 'Development server'
      }
    ],
    components: {
      securitySchemes: {
        BearerAuth: {
          type: 'http',
          scheme: 'bearer',
          bearerFormat: 'JWT',
          description: 'JWT token obtained from /auth/login'
        },
        ApiKeyAuth: {
          type: 'apiKey',
          in: 'header',
          name: 'X-API-Key',
          description: 'API key for service-to-service authentication'
        },
        OAuth2: {
          type: 'oauth2',
          flows: {
            authorizationCode: {
              authorizationUrl: 'https://auth.enbd.com/oauth/authorize',
              tokenUrl: 'https://auth.enbd.com/oauth/token',
              scopes: {
                'read:accounts': 'Read account information',
                'write:transactions': 'Create transactions',
                'read:transactions': 'Read transaction history',
                'admin': 'Full administrative access'
              }
            }
          }
        }
      },
      schemas: {
        Customer: {
          type: 'object',
          required: ['emiratesId', 'name', 'email'],
          properties: {
            id: {
              type: 'string',
              format: 'uuid',
              description: 'Unique customer identifier',
              example: '123e4567-e89b-12d3-a456-426614174000'
            },
            emiratesId: {
              type: 'string',
              pattern: '^784-\\d{4}-\\d{7}-\\d{1}$',
              description: 'Emirates ID number',
              example: '784-1234-1234567-1'
            },
            name: {
              type: 'string',
              minLength: 2,
              maxLength: 200,
              description: 'Full name',
              example: 'Ahmed Al Maktoum'
            },
            email: {
              type: 'string',
              format: 'email',
              description: 'Email address',
              example: 'ahmed@example.com'
            },
            phone: {
              type: 'string',
              pattern: '^\\+971-\\d{2}-\\d{7}$',
              description: 'Phone number',
              example: '+971-50-1234567'
            },
            dateOfBirth: {
              type: 'string',
              format: 'date',
              description: 'Date of birth',
              example: '1990-01-15'
            },
            nationality: {
              type: 'string',
              description: 'Nationality',
              example: 'UAE'
            },
            createdAt: {
              type: 'string',
              format: 'date-time',
              description: 'Account creation timestamp',
              readOnly: true
            },
            updatedAt: {
              type: 'string',
              format: 'date-time',
              description: 'Last update timestamp',
              readOnly: true
            }
          }
        },
        
        Account: {
          type: 'object',
          required: ['customerId', 'accountType'],
          properties: {
            id: {
              type: 'string',
              format: 'uuid'
            },
            customerId: {
              type: 'string',
              format: 'uuid',
              description: 'Owner customer ID'
            },
            accountNumber: {
              type: 'string',
              pattern: '^\\d{16}$',
              description: '16-digit account number',
              example: '1234567890123456'
            },
            accountType: {
              type: 'string',
              enum: ['savings', 'current', 'fixed_deposit'],
              description: 'Type of account'
            },
            currency: {
              type: 'string',
              enum: ['AED', 'USD', 'EUR', 'GBP'],
              default: 'AED',
              description: 'Account currency'
            },
            balance: {
              type: 'number',
              format: 'double',
              minimum: 0,
              description: 'Current balance',
              example: 15000.50
            },
            status: {
              type: 'string',
              enum: ['active', 'frozen', 'closed'],
              description: 'Account status'
            }
          }
        },
        
        Transaction: {
          type: 'object',
          required: ['accountId', 'amount', 'type'],
          properties: {
            id: {
              type: 'string',
              format: 'uuid'
            },
            accountId: {
              type: 'string',
              format: 'uuid'
            },
            amount: {
              type: 'number',
              format: 'double',
              minimum: 0.01,
              description: 'Transaction amount',
              example: 500.00
            },
            type: {
              type: 'string',
              enum: ['debit', 'credit'],
              description: 'Transaction type'
            },
            description: {
              type: 'string',
              maxLength: 500,
              description: 'Transaction description',
              example: 'Payment to merchant'
            },
            referenceNumber: {
              type: 'string',
              description: 'Unique reference number'
            },
            status: {
              type: 'string',
              enum: ['pending', 'completed', 'failed', 'blocked'],
              description: 'Transaction status'
            },
            createdAt: {
              type: 'string',
              format: 'date-time'
            }
          }
        },
        
        Error: {
          type: 'object',
          required: ['error', 'message'],
          properties: {
            error: {
              type: 'string',
              description: 'Error code',
              example: 'ValidationError'
            },
            message: {
              type: 'string',
              description: 'Human-readable error message',
              example: 'Invalid request parameters'
            },
            details: {
              type: 'object',
              description: 'Additional error details',
              additionalProperties: true
            },
            requestId: {
              type: 'string',
              format: 'uuid',
              description: 'Unique request identifier for tracking'
            },
            timestamp: {
              type: 'string',
              format: 'date-time'
            }
          }
        }
      },
      
      responses: {
        UnauthorizedError: {
          description: 'Authentication required',
          content: {
            'application/json': {
              schema: { $ref: '#/components/schemas/Error' },
              example: {
                error: 'Unauthorized',
                message: 'Missing or invalid authorization header',
                requestId: '123e4567-e89b-12d3-a456-426614174000',
                timestamp: '2024-11-10T12:00:00Z'
              }
            }
          }
        },
        
        ForbiddenError: {
          description: 'Insufficient permissions',
          content: {
            'application/json': {
              schema: { $ref: '#/components/schemas/Error' }
            }
          }
        },
        
        NotFoundError: {
          description: 'Resource not found',
          content: {
            'application/json': {
              schema: { $ref: '#/components/schemas/Error' }
            }
          }
        },
        
        ValidationError: {
          description: 'Validation failed',
          content: {
            'application/json': {
              schema: { $ref: '#/components/schemas/Error' }
            }
          }
        }
      },
      
      parameters: {
        RequestId: {
          in: 'header',
          name: 'X-Request-ID',
          schema: { type: 'string', format: 'uuid' },
          description: 'Unique request identifier',
          required: false
        },
        
        PageNumber: {
          in: 'query',
          name: 'page',
          schema: { type: 'integer', minimum: 1, default: 1 },
          description: 'Page number'
        },
        
        PageSize: {
          in: 'query',
          name: 'limit',
          schema: { type: 'integer', minimum: 1, maximum: 100, default: 20 },
          description: 'Items per page'
        }
      }
    },
    
    security: [
      { BearerAuth: [] }
    ],
    
    tags: [
      {
        name: 'Authentication',
        description: 'Authentication and authorization operations'
      },
      {
        name: 'Customers',
        description: 'Customer management'
      },
      {
        name: 'Accounts',
        description: 'Account operations'
      },
      {
        name: 'Transactions',
        description: 'Transaction operations'
      },
      {
        name: 'Loans',
        description: 'Loan management'
      }
    ],
    
    paths: {
      '/auth/login': {
        post: {
          tags: ['Authentication'],
          summary: 'User login',
          description: 'Authenticate user and receive JWT token',
          security: [], // No auth required for login
          requestBody: {
            required: true,
            content: {
              'application/json': {
                schema: {
                  type: 'object',
                  required: ['email', 'password'],
                  properties: {
                    email: {
                      type: 'string',
                      format: 'email',
                      example: 'ahmed@example.com'
                    },
                    password: {
                      type: 'string',
                      format: 'password',
                      example: 'SecureP@ssw0rd'
                    }
                  }
                }
              }
            }
          },
          responses: {
            200: {
              description: 'Login successful',
              content: {
                'application/json': {
                  schema: {
                    type: 'object',
                    properties: {
                      accessToken: { type: 'string' },
                      refreshToken: { type: 'string' },
                      expiresIn: { type: 'integer' }
                    }
                  }
                }
              }
            },
            401: { $ref: '#/components/responses/UnauthorizedError' }
          }
        }
      },
      
      '/customers': {
        get: {
          tags: ['Customers'],
          summary: 'List customers',
          description: 'Retrieve paginated list of customers (admin only)',
          security: [{ BearerAuth: [] }],
          parameters: [
            { $ref: '#/components/parameters/PageNumber' },
            { $ref: '#/components/parameters/PageSize' }
          ],
          responses: {
            200: {
              description: 'Successful response',
              content: {
                'application/json': {
                  schema: {
                    type: 'object',
                    properties: {
                      data: {
                        type: 'array',
                        items: { $ref: '#/components/schemas/Customer' }
                      },
                      pagination: {
                        type: 'object',
                        properties: {
                          page: { type: 'integer' },
                          limit: { type: 'integer' },
                          total: { type: 'integer' },
                          totalPages: { type: 'integer' }
                        }
                      }
                    }
                  }
                }
              }
            },
            401: { $ref: '#/components/responses/UnauthorizedError' },
            403: { $ref: '#/components/responses/ForbiddenError' }
          }
        },
        
        post: {
          tags: ['Customers'],
          summary: 'Create customer',
          description: 'Register new customer',
          requestBody: {
            required: true,
            content: {
              'application/json': {
                schema: { $ref: '#/components/schemas/Customer' }
              }
            }
          },
          responses: {
            201: {
              description: 'Customer created',
              headers: {
                Location: {
                  schema: { type: 'string' },
                  description: 'URL of created customer'
                }
              },
              content: {
                'application/json': {
                  schema: { $ref: '#/components/schemas/Customer' }
                }
              }
            },
            400: { $ref: '#/components/responses/ValidationError' }
          }
        }
      },
      
      '/customers/{customerId}': {
        get: {
          tags: ['Customers'],
          summary: 'Get customer',
          description: 'Retrieve customer by ID',
          parameters: [
            {
              in: 'path',
              name: 'customerId',
              required: true,
              schema: { type: 'string', format: 'uuid' },
              description: 'Customer ID'
            }
          ],
          responses: {
            200: {
              description: 'Successful response',
              content: {
                'application/json': {
                  schema: { $ref: '#/components/schemas/Customer' }
                }
              }
            },
            404: { $ref: '#/components/responses/NotFoundError' }
          }
        }
      },
      
      '/accounts/{accountId}/transactions': {
        get: {
          tags: ['Transactions'],
          summary: 'Get account transactions',
          description: 'Retrieve transaction history for an account',
          parameters: [
            {
              in: 'path',
              name: 'accountId',
              required: true,
              schema: { type: 'string', format: 'uuid' }
            },
            { $ref: '#/components/parameters/PageNumber' },
            { $ref: '#/components/parameters/PageSize' },
            {
              in: 'query',
              name: 'startDate',
              schema: { type: 'string', format: 'date' },
              description: 'Filter from date'
            },
            {
              in: 'query',
              name: 'endDate',
              schema: { type: 'string', format: 'date' },
              description: 'Filter to date'
            }
          ],
          responses: {
            200: {
              description: 'Transaction list',
              content: {
                'application/json': {
                  schema: {
                    type: 'object',
                    properties: {
                      data: {
                        type: 'array',
                        items: { $ref: '#/components/schemas/Transaction' }
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  },
  
  apis: ['./routes/*.js'] // Path to API route files
};

const swaggerSpec = swaggerJsdoc(swaggerOptions);

/**
 * Setup Swagger UI in Express
 */
function setupSwagger(app) {
  // Serve Swagger UI
  app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerSpec, {
    customCss: '.swagger-ui .topbar { display: none }',
    customSiteTitle: 'ENBD API Documentation',
    swaggerOptions: {
      persistAuthorization: true,
      displayRequestDuration: true,
      filter: true,
      tryItOutEnabled: true
    }
  }));
  
  // Serve OpenAPI JSON
  app.get('/api-docs.json', (req, res) => {
    res.setHeader('Content-Type', 'application/json');
    res.send(swaggerSpec);
  });
  
  console.log('Swagger documentation available at /api-docs');
}

/**
 * JSDoc comments for automatic documentation
 */

/**
 * @swagger
 * /api/accounts:
 *   post:
 *     tags:
 *       - Accounts
 *     summary: Create new account
 *     security:
 *       - BearerAuth: []
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             $ref: '#/components/schemas/Account'
 *     responses:
 *       201:
 *         description: Account created successfully
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Account'
 */
router.post('/accounts', createAccount);

module.exports = { setupSwagger, swaggerSpec };
```

---

### Q8. How do you implement GraphQL as an alternative to REST?

**Answer:**

```javascript
/**
 * GraphQL API for Banking (Alternative to REST)
 */
const { ApolloServer, gql } = require('apollo-server-express');
const { makeExecutableSchema } = require('@graphql-tools/schema');
const DataLoader = require('dataloader');

/**
 * GraphQL Type Definitions (Schema)
 */
const typeDefs = gql`
  # Scalars
  scalar Date
  scalar DateTime
  
  # Enums
  enum AccountType {
    SAVINGS
    CURRENT
    FIXED_DEPOSIT
  }
  
  enum TransactionType {
    DEBIT
    CREDIT
  }
  
  enum AccountStatus {
    ACTIVE
    FROZEN
    CLOSED
  }
  
  # Types
  type Customer {
    id: ID!
    emiratesId: String!
    name: String!
    email: String!
    phone: String
    dateOfBirth: Date
    nationality: String
    accounts: [Account!]!
    createdAt: DateTime!
    updatedAt: DateTime!
  }
  
  type Account {
    id: ID!
    customerId: ID!
    customer: Customer!
    accountNumber: String!
    accountType: AccountType!
    currency: String!
    balance: Float!
    status: AccountStatus!
    transactions(
      limit: Int = 20
      offset: Int = 0
      startDate: Date
      endDate: Date
    ): [Transaction!]!
    createdAt: DateTime!
    updatedAt: DateTime!
  }
  
  type Transaction {
    id: ID!
    accountId: ID!
    account: Account!
    amount: Float!
    type: TransactionType!
    description: String
    referenceNumber: String!
    balanceAfter: Float
    status: String!
    createdAt: DateTime!
  }
  
  type Loan {
    id: ID!
    customerId: ID!
    customer: Customer!
    loanType: String!
    amount: Float!
    interestRate: Float!
    tenureMonths: Int!
    monthlyPayment: Float!
    outstandingBalance: Float!
    status: String!
    applicationDate: DateTime!
    approvalDate: DateTime
  }
  
  # Pagination
  type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
    totalCount: Int!
  }
  
  type AccountConnection {
    edges: [AccountEdge!]!
    pageInfo: PageInfo!
  }
  
  type AccountEdge {
    node: Account!
    cursor: String!
  }
  
  # Input types
  input CreateCustomerInput {
    emiratesId: String!
    name: String!
    email: String!
    phone: String
    dateOfBirth: Date
    nationality: String
  }
  
  input CreateAccountInput {
    customerId: ID!
    accountType: AccountType!
    currency: String = "AED"
  }
  
  input CreateTransactionInput {
    accountId: ID!
    amount: Float!
    type: TransactionType!
    description: String
  }
  
  # Query root
  type Query {
    # Customer queries
    customer(id: ID!): Customer
    customers(limit: Int = 20, offset: Int = 0): [Customer!]!
    
    # Account queries
    account(id: ID!): Account
    accounts(customerId: ID!): [Account!]!
    accountsConnection(
      customerId: ID!
      first: Int
      after: String
    ): AccountConnection!
    
    # Transaction queries
    transaction(id: ID!): Transaction
    transactions(
      accountId: ID!
      limit: Int = 20
      offset: Int = 0
      startDate: Date
      endDate: Date
    ): [Transaction!]!
    
    # Loan queries
    loan(id: ID!): Loan
    loans(customerId: ID!): [Loan!]!
    
    # Search
    searchCustomers(query: String!): [Customer!]!
  }
  
  # Mutation root
  type Mutation {
    # Customer mutations
    createCustomer(input: CreateCustomerInput!): Customer!
    updateCustomer(id: ID!, input: CreateCustomerInput!): Customer!
    deleteCustomer(id: ID!): Boolean!
    
    # Account mutations
    createAccount(input: CreateAccountInput!): Account!
    updateAccountStatus(id: ID!, status: AccountStatus!): Account!
    closeAccount(id: ID!): Boolean!
    
    # Transaction mutations
    createTransaction(input: CreateTransactionInput!): Transaction!
    
    # Loan mutations
    applyForLoan(
      customerId: ID!
      loanType: String!
      amount: Float!
      tenureMonths: Int!
    ): Loan!
  }
  
  # Subscription root
  type Subscription {
    transactionCreated(accountId: ID!): Transaction!
    accountBalanceChanged(accountId: ID!): Account!
  }
`;

/**
 * GraphQL Resolvers
 */
const resolvers = {
  Query: {
    customer: async (parent, { id }, context) => {
      if (!context.user) {
        throw new Error('Unauthorized');
      }
      
      return context.loaders.customer.load(id);
    },
    
    customers: async (parent, { limit, offset }, context) => {
      if (!context.user?.roles.includes('admin')) {
        throw new Error('Forbidden');
      }
      
      const result = await context.db.query(
        'SELECT * FROM customers LIMIT $1 OFFSET $2',
        [limit, offset]
      );
      
      return result.rows;
    },
    
    account: async (parent, { id }, context) => {
      return context.loaders.account.load(id);
    },
    
    accounts: async (parent, { customerId }, context) => {
      // Check ownership
      if (context.user.customerId !== customerId && !context.user.roles.includes('admin')) {
        throw new Error('Forbidden');
      }
      
      const result = await context.db.query(
        'SELECT * FROM accounts WHERE customer_id = $1',
        [customerId]
      );
      
      return result.rows;
    },
    
    transactions: async (parent, { accountId, limit, offset, startDate, endDate }, context) => {
      let query = 'SELECT * FROM transactions WHERE account_id = $1';
      const params = [accountId];
      let paramIndex = 2;
      
      if (startDate) {
        query += ` AND created_at >= $${paramIndex}`;
        params.push(startDate);
        paramIndex++;
      }
      
      if (endDate) {
        query += ` AND created_at <= $${paramIndex}`;
        params.push(endDate);
        paramIndex++;
      }
      
      query += ` ORDER BY created_at DESC LIMIT $${paramIndex} OFFSET $${paramIndex + 1}`;
      params.push(limit, offset);
      
      const result = await context.db.query(query, params);
      return result.rows;
    }
  },
  
  Mutation: {
    createCustomer: async (parent, { input }, context) => {
      const { emiratesId, name, email, phone, dateOfBirth, nationality } = input;
      
      const result = await context.db.query(`
        INSERT INTO customers (emirates_id, name, email, phone, date_of_birth, nationality)
        VALUES ($1, $2, $3, $4, $5, $6)
        RETURNING *
      `, [emiratesId, name, email, phone, dateOfBirth, nationality]);
      
      return result.rows[0];
    },
    
    createAccount: async (parent, { input }, context) => {
      const { customerId, accountType, currency } = input;
      
      const accountNumber = generateAccountNumber();
      
      const result = await context.db.query(`
        INSERT INTO accounts (customer_id, account_number, account_type, currency, balance)
        VALUES ($1, $2, $3, $4, 0)
        RETURNING *
      `, [customerId, accountNumber, accountType, currency]);
      
      return result.rows[0];
    },
    
    createTransaction: async (parent, { input }, context) => {
      const { accountId, amount, type, description } = input;
      
      const result = await context.db.query(`
        INSERT INTO transactions (account_id, amount, type, description)
        VALUES ($1, $2, $3, $4)
        RETURNING *
      `, [accountId, amount, type, description]);
      
      return result.rows[0];
    }
  },
  
  // Field resolvers
  Customer: {
    accounts: async (parent, args, context) => {
      return context.loaders.accountsByCustomer.load(parent.id);
    }
  },
  
  Account: {
    customer: async (parent, args, context) => {
      return context.loaders.customer.load(parent.customerId);
    },
    
    transactions: async (parent, { limit, offset, startDate, endDate }, context) => {
      const result = await context.db.query(
        'SELECT * FROM transactions WHERE account_id = $1 ORDER BY created_at DESC LIMIT $2 OFFSET $3',
        [parent.id, limit, offset]
      );
      
      return result.rows;
    }
  },
  
  Transaction: {
    account: async (parent, args, context) => {
      return context.loaders.account.load(parent.accountId);
    }
  }
};

/**
 * DataLoader for batching and caching
 */
function createLoaders(db) {
  return {
    customer: new DataLoader(async (ids) => {
      const result = await db.query(
        'SELECT * FROM customers WHERE id = ANY($1)',
        [ids]
      );
      
      const customerMap = new Map(result.rows.map(c => [c.id, c]));
      return ids.map(id => customerMap.get(id));
    }),
    
    account: new DataLoader(async (ids) => {
      const result = await db.query(
        'SELECT * FROM accounts WHERE id = ANY($1)',
        [ids]
      );
      
      const accountMap = new Map(result.rows.map(a => [a.id, a]));
      return ids.map(id => accountMap.get(id));
    }),
    
    accountsByCustomer: new DataLoader(async (customerIds) => {
      const result = await db.query(
        'SELECT * FROM accounts WHERE customer_id = ANY($1)',
        [customerIds]
      );
      
      const accountsByCustomer = new Map();
      customerIds.forEach(id => accountsByCustomer.set(id, []));
      
      result.rows.forEach(account => {
        accountsByCustomer.get(account.customer_id).push(account);
      });
      
      return customerIds.map(id => accountsByCustomer.get(id));
    })
  };
}

/**
 * Setup Apollo Server
 */
async function setupGraphQL(app, db) {
  const schema = makeExecutableSchema({ typeDefs, resolvers });
  
  const server = new ApolloServer({
    schema,
    context: ({ req }) => ({
      user: req.user, // From JWT middleware
      db,
      loaders: createLoaders(db)
    }),
    formatError: (error) => {
      console.error('GraphQL Error:', error);
      
      return {
        message: error.message,
        code: error.extensions.code,
        path: error.path
      };
    },
    plugins: [
      {
        requestDidStart(requestContext) {
          console.log(`GraphQL Request: ${requestContext.request.operationName}`);
        }
      }
    ]
  });
  
  await server.start();
  server.applyMiddleware({ app, path: '/graphql' });
  
  console.log(`GraphQL endpoint: http://localhost:3000${server.graphqlPath}`);
}

function generateAccountNumber() {
  return '1234' + Math.random().toString().substr(2, 12);
}

module.exports = { typeDefs, resolvers, setupGraphQL, createLoaders };
```

---

**Summary Q6-Q8:**
- API Gateway implementation (custom + Kong) ✅
- Comprehensive OpenAPI/Swagger documentation ✅
- GraphQL as REST alternative with DataLoader ✅
