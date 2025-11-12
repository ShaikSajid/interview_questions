# APIs Interview Questions (Q45-Q47): Testing & Documentation

## Q45: How do you implement comprehensive API testing strategies?

**Answer:**

```javascript
// api-testing.test.js
const request = require('supertest');
const { app } = require('../app');
const { setupTestDB, teardownTestDB } = require('./helpers/db');

describe('API Testing Suite', () => {
    beforeAll(async () => {
        await setupTestDB();
    });

    afterAll(async () => {
        await teardownTestDB();
    });

    describe('Unit Tests - Service Layer', () => {
        test('should create user with valid data', async () => {
            const userService = new UserService();
            const userData = {
                name: 'John Doe',
                email: 'john@enbd.com',
                phoneNumber: '+971501234567'
            };

            const user = await userService.createUser(userData);

            expect(user).toHaveProperty('id');
            expect(user.email).toBe(userData.email);
            expect(user.status).toBe('active');
        });

        test('should throw validation error for invalid email', async () => {
            const userService = new UserService();
            const userData = {
                name: 'John Doe',
                email: 'invalid-email',
                phoneNumber: '+971501234567'
            };

            await expect(userService.createUser(userData))
                .rejects
                .toThrow(ValidationError);
        });
    });

    describe('Integration Tests - API Endpoints', () => {
        test('POST /api/users - should create user', async () => {
            const response = await request(app)
                .post('/api/users')
                .send({
                    name: 'Jane Doe',
                    email: 'jane@enbd.com',
                    phoneNumber: '+971507654321'
                })
                .set('Authorization', 'Bearer test-token')
                .expect('Content-Type', /json/)
                .expect(201);

            expect(response.body).toHaveProperty('id');
            expect(response.body.email).toBe('jane@enbd.com');
        });

        test('GET /api/users/:id - should return user', async () => {
            const response = await request(app)
                .get('/api/users/1')
                .set('Authorization', 'Bearer test-token')
                .expect(200);

            expect(response.body).toHaveProperty('id', '1');
            expect(response.body).toHaveProperty('email');
        });

        test('PUT /api/users/:id - should update user', async () => {
            const response = await request(app)
                .put('/api/users/1')
                .send({ name: 'Jane Updated' })
                .set('Authorization', 'Bearer test-token')
                .expect(200);

            expect(response.body.name).toBe('Jane Updated');
        });

        test('DELETE /api/users/:id - should delete user', async () => {
            await request(app)
                .delete('/api/users/1')
                .set('Authorization', 'Bearer test-token')
                .expect(204);
        });
    });

    describe('Contract Tests - API Schema Validation', () => {
        test('should match OpenAPI schema for user response', async () => {
            const response = await request(app)
                .get('/api/users/1')
                .set('Authorization', 'Bearer test-token');

            const schema = {
                type: 'object',
                required: ['id', 'name', 'email', 'createdAt'],
                properties: {
                    id: { type: 'string' },
                    name: { type: 'string' },
                    email: { type: 'string', format: 'email' },
                    phoneNumber: { type: 'string' },
                    createdAt: { type: 'string', format: 'date-time' }
                }
            };

            const validator = new Ajv();
            const valid = validator.validate(schema, response.body);
            expect(valid).toBe(true);
        });
    });

    describe('Performance Tests', () => {
        test('should handle concurrent requests', async () => {
            const requests = Array(100).fill().map(() =>
                request(app)
                    .get('/api/users')
                    .set('Authorization', 'Bearer test-token')
            );

            const responses = await Promise.all(requests);
            const successCount = responses.filter(r => r.status === 200).length;

            expect(successCount).toBe(100);
        });

        test('should respond within acceptable time', async () => {
            const start = Date.now();
            
            await request(app)
                .get('/api/users')
                .set('Authorization', 'Bearer test-token')
                .expect(200);

            const duration = Date.now() - start;
            expect(duration).toBeLessThan(500); // 500ms threshold
        });
    });

    describe('Security Tests', () => {
        test('should reject requests without authentication', async () => {
            await request(app)
                .get('/api/users')
                .expect(401);
        });

        test('should prevent SQL injection', async () => {
            await request(app)
                .get('/api/users/1\' OR \'1\'=\'1')
                .set('Authorization', 'Bearer test-token')
                .expect(400);
        });

        test('should sanitize XSS in input', async () => {
            const response = await request(app)
                .post('/api/users')
                .send({
                    name: '<script>alert("xss")</script>',
                    email: 'test@enbd.com',
                    phoneNumber: '+971501234567'
                })
                .set('Authorization', 'Bearer test-token')
                .expect(201);

            expect(response.body.name).not.toContain('<script>');
        });

        test('should rate limit excessive requests', async () => {
            const requests = Array(150).fill().map(() =>
                request(app)
                    .get('/api/users')
                    .set('Authorization', 'Bearer test-token')
            );

            const responses = await Promise.all(requests);
            const rateLimited = responses.filter(r => r.status === 429).length;

            expect(rateLimited).toBeGreaterThan(0);
        });
    });

    describe('Error Handling Tests', () => {
        test('should return 404 for non-existent resource', async () => {
            await request(app)
                .get('/api/users/99999')
                .set('Authorization', 'Bearer test-token')
                .expect(404);
        });

        test('should return validation errors', async () => {
            const response = await request(app)
                .post('/api/users')
                .send({ name: 'Test' }) // Missing required fields
                .set('Authorization', 'Bearer test-token')
                .expect(400);

            expect(response.body.error.code).toBe('VALIDATION_ERROR');
            expect(response.body.error.details).toHaveProperty('email');
        });
    });
});

// Load testing with Artillery
/*
config:
  target: 'http://localhost:3000'
  phases:
    - duration: 60
      arrivalRate: 10
      name: Warm up
    - duration: 300
      arrivalRate: 100
      name: Sustained load
    - duration: 60
      arrivalRate: 200
      name: Spike test

scenarios:
  - name: User journey
    flow:
      - post:
          url: /api/auth/login
          json:
            email: test@enbd.com
            password: password123
          capture:
            - json: $.token
              as: authToken
      - get:
          url: /api/users
          headers:
            Authorization: 'Bearer {{ authToken }}'
      - post:
          url: /api/accounts
          headers:
            Authorization: 'Bearer {{ authToken }}'
          json:
            accountType: savings
            currency: AED
*/

module.exports = { describe, test, expect };
```

---

## Q46: How do you implement API documentation with OpenAPI/Swagger?

**Answer:**

```javascript
// swagger-config.js
const swaggerJsDoc = require('swagger-jsdoc');
const swaggerUi = require('swagger-ui-express');

const swaggerOptions = {
    definition: {
        openapi: '3.0.0',
        info: {
            title: 'Emirates NBD API',
            version: '1.0.0',
            description: 'Banking API for Emirates NBD',
            contact: {
                name: 'API Support',
                email: 'api-support@emiratesnbd.com'
            },
            license: {
                name: 'Proprietary',
                url: 'https://emiratesnbd.com/license'
            }
        },
        servers: [
            {
                url: 'https://api.emiratesnbd.com/v1',
                description: 'Production server'
            },
            {
                url: 'https://api-staging.emiratesnbd.com/v1',
                description: 'Staging server'
            },
            {
                url: 'http://localhost:3000/v1',
                description: 'Development server'
            }
        ],
        components: {
            securitySchemes: {
                bearerAuth: {
                    type: 'http',
                    scheme: 'bearer',
                    bearerFormat: 'JWT'
                },
                apiKey: {
                    type: 'apiKey',
                    in: 'header',
                    name: 'X-API-Key'
                }
            },
            schemas: {
                User: {
                    type: 'object',
                    required: ['name', 'email', 'phoneNumber'],
                    properties: {
                        id: {
                            type: 'string',
                            format: 'uuid',
                            description: 'User unique identifier'
                        },
                        name: {
                            type: 'string',
                            minLength: 2,
                            maxLength: 100,
                            description: 'Full name'
                        },
                        email: {
                            type: 'string',
                            format: 'email',
                            description: 'Email address'
                        },
                        phoneNumber: {
                            type: 'string',
                            pattern: '^\\+971[0-9]{9}$',
                            description: 'UAE phone number'
                        },
                        status: {
                            type: 'string',
                            enum: ['active', 'inactive', 'suspended'],
                            description: 'User status'
                        },
                        createdAt: {
                            type: 'string',
                            format: 'date-time'
                        }
                    }
                },
                Account: {
                    type: 'object',
                    properties: {
                        id: { type: 'string', format: 'uuid' },
                        userId: { type: 'string', format: 'uuid' },
                        accountNumber: { type: 'string' },
                        accountType: {
                            type: 'string',
                            enum: ['savings', 'current', 'fixed_deposit']
                        },
                        balance: { type: 'number', format: 'double' },
                        currency: { type: 'string', default: 'AED' },
                        status: {
                            type: 'string',
                            enum: ['active', 'frozen', 'closed']
                        }
                    }
                },
                Error: {
                    type: 'object',
                    properties: {
                        error: {
                            type: 'object',
                            properties: {
                                code: { type: 'string' },
                                message: { type: 'string' },
                                details: { type: 'object' },
                                timestamp: { type: 'string', format: 'date-time' },
                                requestId: { type: 'string' }
                            }
                        }
                    }
                }
            },
            responses: {
                UnauthorizedError: {
                    description: 'Access token is missing or invalid',
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
            }
        },
        security: [
            { bearerAuth: [] }
        ]
    },
    apis: ['./routes/*.js', './controllers/*.js']
};

const swaggerDocs = swaggerJsDoc(swaggerOptions);

module.exports = { swaggerDocs, swaggerUi };

// Example route documentation
/**
 * @swagger
 * /users:
 *   get:
 *     summary: Get all users
 *     tags: [Users]
 *     security:
 *       - bearerAuth: []
 *     parameters:
 *       - in: query
 *         name: page
 *         schema:
 *           type: integer
 *           minimum: 1
 *           default: 1
 *         description: Page number
 *       - in: query
 *         name: limit
 *         schema:
 *           type: integer
 *           minimum: 1
 *           maximum: 100
 *           default: 20
 *         description: Number of items per page
 *       - in: query
 *         name: status
 *         schema:
 *           type: string
 *           enum: [active, inactive, suspended]
 *         description: Filter by status
 *     responses:
 *       200:
 *         description: List of users
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 data:
 *                   type: array
 *                   items:
 *                     $ref: '#/components/schemas/User'
 *                 pagination:
 *                   type: object
 *                   properties:
 *                     page: { type: integer }
 *                     limit: { type: integer }
 *                     total: { type: integer }
 *                     totalPages: { type: integer }
 *       401:
 *         $ref: '#/components/responses/UnauthorizedError'
 *
 *   post:
 *     summary: Create a new user
 *     tags: [Users]
 *     security:
 *       - bearerAuth: []
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             $ref: '#/components/schemas/User'
 *           examples:
 *             example1:
 *               value:
 *                 name: John Doe
 *                 email: john.doe@enbd.com
 *                 phoneNumber: '+971501234567'
 *     responses:
 *       201:
 *         description: User created successfully
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/User'
 *       400:
 *         $ref: '#/components/responses/ValidationError'
 *       401:
 *         $ref: '#/components/responses/UnauthorizedError'
 */
```

---

## Q47: How do you implement API mocking and contract testing?

**Answer:**

```javascript
// mock-server.js
const express = require('express');
const { faker } = require('@faker-js/faker');

class APIMockServer {
    constructor(port = 4000) {
        this.app = express();
        this.port = port;
        this.setupMiddleware();
        this.setupRoutes();
    }

    setupMiddleware() {
        this.app.use(express.json());
        
        this.app.use((req, res, next) => {
            console.log(`[MOCK] ${req.method} ${req.path}`);
            next();
        });
    }

    setupRoutes() {
        // Mock user endpoints
        this.app.get('/api/users', (req, res) => {
            const users = Array(10).fill().map(() => this.generateUser());
            res.json({
                data: users,
                pagination: {
                    page: 1,
                    limit: 10,
                    total: 100,
                    totalPages: 10
                }
            });
        });

        this.app.get('/api/users/:id', (req, res) => {
            res.json(this.generateUser(req.params.id));
        });

        this.app.post('/api/users', (req, res) => {
            res.status(201).json({
                ...req.body,
                id: faker.string.uuid(),
                createdAt: new Date().toISOString()
            });
        });

        // Mock account endpoints
        this.app.get('/api/accounts/:id', (req, res) => {
            res.json(this.generateAccount(req.params.id));
        });

        // Mock error scenarios
        this.app.get('/api/error/500', (req, res) => {
            res.status(500).json({
                error: {
                    code: 'INTERNAL_ERROR',
                    message: 'Internal server error'
                }
            });
        });

        this.app.get('/api/error/timeout', (req, res) => {
            // Don't respond - simulate timeout
        });

        // Mock latency
        this.app.get('/api/slow', async (req, res) => {
            await this.sleep(3000);
            res.json({ message: 'Slow response' });
        });
    }

    generateUser(id = null) {
        return {
            id: id || faker.string.uuid(),
            name: faker.person.fullName(),
            email: faker.internet.email(),
            phoneNumber: '+971' + faker.string.numeric(9),
            status: faker.helpers.arrayElement(['active', 'inactive']),
            createdAt: faker.date.past().toISOString()
        };
    }

    generateAccount(id = null) {
        return {
            id: id || faker.string.uuid(),
            accountNumber: 'ENBD' + faker.string.numeric(12),
            accountType: faker.helpers.arrayElement(['savings', 'current']),
            balance: faker.number.float({ min: 0, max: 1000000, precision: 0.01 }),
            currency: 'AED',
            status: 'active'
        };
    }

    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }

    start() {
        this.app.listen(this.port, () => {
            console.log(`Mock server running on port ${this.port}`);
        });
    }
}

// Contract testing with Pact
const { Pact } = require('@pact-foundation/pact');
const { like, eachLike } = require('@pact-foundation/pact/dsl/matchers');

describe('User Service Contract Tests', () => {
    const provider = new Pact({
        consumer: 'api-gateway',
        provider: 'user-service',
        port: 4000
    });

    beforeAll(() => provider.setup());
    afterAll(() => provider.finalize());
    afterEach(() => provider.verify());

    test('should get user by id', async () => {
        await provider.addInteraction({
            state: 'user exists',
            uponReceiving: 'a request for user',
            withRequest: {
                method: 'GET',
                path: '/api/users/123',
                headers: {
                    'Authorization': like('Bearer token')
                }
            },
            willRespondWith: {
                status: 200,
                headers: {
                    'Content-Type': 'application/json'
                },
                body: {
                    id: '123',
                    name: like('John Doe'),
                    email: like('john@enbd.com'),
                    phoneNumber: like('+971501234567'),
                    status: like('active'),
                    createdAt: like('2024-01-01T00:00:00Z')
                }
            }
        });

        const response = await axios.get('http://localhost:4000/api/users/123', {
            headers: { 'Authorization': 'Bearer token' }
        });

        expect(response.status).toBe(200);
        expect(response.data).toHaveProperty('id', '123');
    });

    test('should get list of users', async () => {
        await provider.addInteraction({
            state: 'users exist',
            uponReceiving: 'a request for users list',
            withRequest: {
                method: 'GET',
                path: '/api/users',
                query: { page: '1', limit: '10' }
            },
            willRespondWith: {
                status: 200,
                body: {
                    data: eachLike({
                        id: like('123'),
                        name: like('John Doe'),
                        email: like('john@enbd.com')
                    }),
                    pagination: {
                        page: 1,
                        limit: 10,
                        total: like(100),
                        totalPages: like(10)
                    }
                }
            }
        });

        const response = await axios.get('http://localhost:4000/api/users', {
            params: { page: 1, limit: 10 }
        });

        expect(response.status).toBe(200);
        expect(response.data.data).toBeInstanceOf(Array);
    });
});

module.exports = { APIMockServer };
```

---
