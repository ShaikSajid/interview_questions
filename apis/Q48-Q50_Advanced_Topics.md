# APIs Interview Questions (Q48-Q50): Advanced Topics & Best Practices

## Q48: How do you implement GraphQL API alongside REST for flexible data querying?

**Answer:**

```javascript
// graphql-server.js
const { ApolloServer } = require('apollo-server-express');
const { gql } = require('apollo-server-express');
const DataLoader = require('dataloader');

// Type definitions
const typeDefs = gql`
    type User {
        id: ID!
        name: String!
        email: String!
        phoneNumber: String!
        accounts: [Account!]!
        transactions(limit: Int, offset: Int): [Transaction!]!
        status: UserStatus!
        createdAt: DateTime!
    }

    type Account {
        id: ID!
        accountNumber: String!
        accountType: AccountType!
        balance: Float!
        currency: String!
        status: AccountStatus!
        user: User!
        transactions(limit: Int): [Transaction!]!
    }

    type Transaction {
        id: ID!
        account: Account!
        type: TransactionType!
        amount: Float!
        currency: String!
        description: String
        status: TransactionStatus!
        createdAt: DateTime!
    }

    enum UserStatus {
        ACTIVE
        INACTIVE
        SUSPENDED
    }

    enum AccountType {
        SAVINGS
        CURRENT
        FIXED_DEPOSIT
    }

    enum AccountStatus {
        ACTIVE
        FROZEN
        CLOSED
    }

    enum TransactionType {
        CREDIT
        DEBIT
        TRANSFER
    }

    enum TransactionStatus {
        PENDING
        COMPLETED
        FAILED
    }

    scalar DateTime

    type Query {
        user(id: ID!): User
        users(limit: Int, offset: Int, status: UserStatus): [User!]!
        account(id: ID!): Account
        accounts(userId: ID!): [Account!]!
        transaction(id: ID!): Transaction
        transactions(accountId: ID!, limit: Int, offset: Int): [Transaction!]!
        searchUsers(query: String!): [User!]!
    }

    type Mutation {
        createUser(input: CreateUserInput!): User!
        updateUser(id: ID!, input: UpdateUserInput!): User!
        deleteUser(id: ID!): Boolean!
        
        createAccount(input: CreateAccountInput!): Account!
        freezeAccount(id: ID!, reason: String!): Account!
        unfreezeAccount(id: ID!): Account!
        
        createTransaction(input: CreateTransactionInput!): Transaction!
        transferFunds(input: TransferInput!): TransferResult!
    }

    input CreateUserInput {
        name: String!
        email: String!
        phoneNumber: String!
    }

    input UpdateUserInput {
        name: String
        email: String
        phoneNumber: String
    }

    input CreateAccountInput {
        userId: ID!
        accountType: AccountType!
        currency: String
    }

    input CreateTransactionInput {
        accountId: ID!
        type: TransactionType!
        amount: Float!
        description: String
    }

    input TransferInput {
        fromAccountId: ID!
        toAccountId: ID!
        amount: Float!
        description: String
    }

    type TransferResult {
        success: Boolean!
        transactionId: ID
        message: String!
    }
`;

// Resolvers
const resolvers = {
    Query: {
        user: async (_, { id }, { dataSources }) => {
            return dataSources.userAPI.getUserById(id);
        },
        
        users: async (_, { limit = 20, offset = 0, status }, { dataSources }) => {
            return dataSources.userAPI.getUsers({ limit, offset, status });
        },
        
        account: async (_, { id }, { dataSources }) => {
            return dataSources.accountAPI.getAccountById(id);
        },
        
        accounts: async (_, { userId }, { dataSources }) => {
            return dataSources.accountAPI.getAccountsByUserId(userId);
        },
        
        transaction: async (_, { id }, { dataSources }) => {
            return dataSources.transactionAPI.getTransactionById(id);
        },
        
        transactions: async (_, { accountId, limit = 50, offset = 0 }, { dataSources }) => {
            return dataSources.transactionAPI.getTransactionsByAccountId(
                accountId,
                { limit, offset }
            );
        },
        
        searchUsers: async (_, { query }, { dataSources }) => {
            return dataSources.userAPI.searchUsers(query);
        }
    },

    Mutation: {
        createUser: async (_, { input }, { dataSources }) => {
            return dataSources.userAPI.createUser(input);
        },
        
        updateUser: async (_, { id, input }, { dataSources }) => {
            return dataSources.userAPI.updateUser(id, input);
        },
        
        deleteUser: async (_, { id }, { dataSources }) => {
            await dataSources.userAPI.deleteUser(id);
            return true;
        },
        
        createAccount: async (_, { input }, { dataSources }) => {
            return dataSources.accountAPI.createAccount(input);
        },
        
        freezeAccount: async (_, { id, reason }, { dataSources }) => {
            return dataSources.accountAPI.freezeAccount(id, reason);
        },
        
        createTransaction: async (_, { input }, { dataSources }) => {
            return dataSources.transactionAPI.createTransaction(input);
        },
        
        transferFunds: async (_, { input }, { dataSources }) => {
            return dataSources.transactionAPI.transferFunds(input);
        }
    },

    User: {
        accounts: async (user, _, { dataSources }) => {
            return dataSources.accountAPI.getAccountsByUserId(user.id);
        },
        
        transactions: async (user, { limit, offset }, { dataSources, loaders }) => {
            return loaders.userTransactionsLoader.load({ 
                userId: user.id, 
                limit, 
                offset 
            });
        }
    },

    Account: {
        user: async (account, _, { loaders }) => {
            return loaders.userLoader.load(account.userId);
        },
        
        transactions: async (account, { limit = 50 }, { dataSources }) => {
            return dataSources.transactionAPI.getTransactionsByAccountId(
                account.id,
                { limit }
            );
        }
    },

    Transaction: {
        account: async (transaction, _, { loaders }) => {
            return loaders.accountLoader.load(transaction.accountId);
        }
    }
};

// DataLoader for batching and caching
function createLoaders(dataSources) {
    return {
        userLoader: new DataLoader(async (userIds) => {
            const users = await dataSources.userAPI.getUsersByIds(userIds);
            return userIds.map(id => users.find(u => u.id === id));
        }),
        
        accountLoader: new DataLoader(async (accountIds) => {
            const accounts = await dataSources.accountAPI.getAccountsByIds(accountIds);
            return accountIds.map(id => accounts.find(a => a.id === id));
        }),
        
        userTransactionsLoader: new DataLoader(async (keys) => {
            return Promise.all(
                keys.map(({ userId, limit, offset }) =>
                    dataSources.transactionAPI.getTransactionsByUserId(
                        userId,
                        { limit, offset }
                    )
                )
            );
        })
    };
}

// Apollo Server setup
const server = new ApolloServer({
    typeDefs,
    resolvers,
    context: ({ req }) => ({
        user: req.user,
        dataSources: {
            userAPI: new UserAPI(),
            accountAPI: new AccountAPI(),
            transactionAPI: new TransactionAPI()
        },
        loaders: createLoaders({
            userAPI: new UserAPI(),
            accountAPI: new AccountAPI(),
            transactionAPI: new TransactionAPI()
        })
    }),
    formatError: (error) => {
        console.error(error);
        return {
            message: error.message,
            code: error.extensions?.code,
            path: error.path
        };
    }
});

module.exports = { server, typeDefs, resolvers };
```

---

## Q49: How do you implement API governance and lifecycle management?

**Answer:**

```javascript
// api-governance-service.js
class APIGovernanceService {
    constructor(db) {
        this.db = db;
        this.policies = new Map();
        this.loadPolicies();
    }

    async loadPolicies() {
        const policies = await this.db.query('SELECT * FROM api_policies WHERE is_active = true');
        
        policies.rows.forEach(policy => {
            this.policies.set(policy.name, {
                ...policy,
                rules: JSON.parse(policy.rules)
            });
        });
    }

    async enforcePolicy(policyName, context) {
        const policy = this.policies.get(policyName);
        
        if (!policy) {
            throw new Error(`Policy ${policyName} not found`);
        }

        for (const rule of policy.rules) {
            const result = await this.evaluateRule(rule, context);
            
            if (!result.passed) {
                return {
                    allowed: false,
                    reason: result.reason,
                    policy: policyName
                };
            }
        }

        return { allowed: true };
    }

    async evaluateRule(rule, context) {
        switch (rule.type) {
            case 'rate_limit':
                return this.checkRateLimit(rule, context);
            
            case 'quota':
                return this.checkQuota(rule, context);
            
            case 'authentication':
                return this.checkAuthentication(rule, context);
            
            case 'authorization':
                return this.checkAuthorization(rule, context);
            
            case 'data_classification':
                return this.checkDataClassification(rule, context);
            
            default:
                return { passed: true };
        }
    }

    async checkRateLimit(rule, context) {
        const key = `rate_limit:${context.clientId}:${rule.scope}`;
        const current = await redis.incr(key);
        
        if (current === 1) {
            await redis.expire(key, rule.window);
        }

        if (current > rule.limit) {
            return {
                passed: false,
                reason: `Rate limit exceeded: ${current}/${rule.limit} in ${rule.window}s`
            };
        }

        return { passed: true };
    }

    async checkQuota(rule, context) {
        const usage = await this.getMonthlyUsage(context.clientId);
        
        if (usage >= rule.monthlyLimit) {
            return {
                passed: false,
                reason: `Monthly quota exceeded: ${usage}/${rule.monthlyLimit}`
            };
        }

        return { passed: true };
    }

    async checkAuthentication(rule, context) {
        if (!context.user) {
            return {
                passed: false,
                reason: 'Authentication required'
            };
        }

        return { passed: true };
    }

    async checkAuthorization(rule, context) {
        const hasPermission = await this.checkPermission(
            context.user.id,
            rule.requiredPermissions
        );

        if (!hasPermission) {
            return {
                passed: false,
                reason: 'Insufficient permissions'
            };
        }

        return { passed: true };
    }

    async checkDataClassification(rule, context) {
        const dataClass = await this.getDataClassification(context.resource);
        
        if (rule.allowedClassifications.includes(dataClass)) {
            return { passed: true };
        }

        return {
            passed: false,
            reason: `Data classification ${dataClass} not allowed`
        };
    }
}

// API Lifecycle Management
class APILifecycleManager {
    constructor(db) {
        this.db = db;
        this.stages = ['draft', 'published', 'deprecated', 'retired'];
    }

    async createAPI(apiData) {
        const api = await this.db.query(`
            INSERT INTO apis (name, version, description, stage, created_by)
            VALUES ($1, $2, $3, 'draft', $4)
            RETURNING *
        `, [apiData.name, apiData.version, apiData.description, apiData.createdBy]);

        await this.auditLog('create', api.rows[0]);
        return api.rows[0];
    }

    async publishAPI(apiId) {
        const api = await this.getAPI(apiId);
        
        if (api.stage !== 'draft') {
            throw new Error('Only draft APIs can be published');
        }

        await this.validateAPI(api);

        await this.db.query(`
            UPDATE apis 
            SET stage = 'published', published_at = NOW()
            WHERE id = $1
        `, [apiId]);

        await this.notifyStakeholders(apiId, 'published');
        await this.auditLog('publish', api);
    }

    async deprecateAPI(apiId, sunsetDate, migrationGuide) {
        await this.db.query(`
            UPDATE apis 
            SET stage = 'deprecated',
                deprecated_at = NOW(),
                sunset_date = $2,
                migration_guide = $3
            WHERE id = $1
        `, [apiId, sunsetDate, migrationGuide]);

        await this.notifyConsumers(apiId, 'deprecated');
        await this.auditLog('deprecate', { apiId, sunsetDate });
    }

    async retireAPI(apiId) {
        const api = await this.getAPI(apiId);
        
        if (api.stage !== 'deprecated') {
            throw new Error('Only deprecated APIs can be retired');
        }

        const activeConsumers = await this.getActiveConsumers(apiId);
        
        if (activeConsumers.length > 0) {
            throw new Error(
                `Cannot retire API with active consumers: ${activeConsumers.join(', ')}`
            );
        }

        await this.db.query(`
            UPDATE apis 
            SET stage = 'retired', retired_at = NOW()
            WHERE id = $1
        `, [apiId]);

        await this.auditLog('retire', api);
    }

    async validateAPI(api) {
        // Validate OpenAPI spec
        // Check security requirements
        // Verify documentation
        // Test endpoints
        return true;
    }

    async getActiveConsumers(apiId) {
        const result = await this.db.query(`
            SELECT DISTINCT client_id 
            FROM api_usage 
            WHERE api_id = $1 
            AND timestamp > NOW() - INTERVAL '7 days'
        `, [apiId]);

        return result.rows.map(r => r.client_id);
    }

    async notifyConsumers(apiId, event) {
        const consumers = await this.getActiveConsumers(apiId);
        
        for (const consumerId of consumers) {
            await this.sendNotification(consumerId, {
                type: 'api_lifecycle',
                apiId,
                event,
                timestamp: new Date()
            });
        }
    }

    async auditLog(action, data) {
        await this.db.query(`
            INSERT INTO api_audit_log (action, data, timestamp)
            VALUES ($1, $2, NOW())
        `, [action, JSON.stringify(data)]);
    }
}

module.exports = { APIGovernanceService, APILifecycleManager };
```

---

## Q50: What are the best practices for building production-ready APIs at scale?

**Answer:**

```javascript
// best-practices-guide.js

/**
 * EMIRATES NBD API BEST PRACTICES GUIDE
 * ====================================
 */

// 1. Design Principles
const designPrinciples = {
    restful: {
        description: 'Follow REST principles consistently',
        guidelines: [
            'Use nouns for resources, not verbs',
            'Use HTTP methods correctly (GET, POST, PUT, PATCH, DELETE)',
            'Return appropriate status codes',
            'Use plurals for collections (/users, not /user)',
            'Design hierarchical URIs (/users/{id}/accounts)'
        ]
    },
    
    consistency: {
        description: 'Maintain consistency across all APIs',
        guidelines: [
            'Use consistent naming conventions (camelCase or snake_case)',
            'Standardize error response formats',
            'Use consistent date/time formats (ISO 8601)',
            'Maintain consistent pagination patterns',
            'Use standard authentication mechanisms'
        ]
    },
    
    versioning: {
        description: 'Plan for API evolution',
        strategies: [
            'URL versioning: /v1/users',
            'Header versioning: API-Version: 1',
            'Accept header: application/vnd.enbd.v1+json',
            'Never break backward compatibility without version bump',
            'Deprecate gracefully with sunset dates'
        ]
    }
};

// 2. Security Best Practices
class SecurityBestPractices {
    static implementSecurity() {
        return {
            authentication: [
                'Use OAuth 2.0 for authentication',
                'Implement JWT with short expiry',
                'Use refresh tokens for long sessions',
                'Store tokens securely (HttpOnly cookies)',
                'Implement API key rotation'
            ],
            
            authorization: [
                'Implement role-based access control (RBAC)',
                'Use principle of least privilege',
                'Validate permissions on every request',
                'Implement resource-level authorization',
                'Log all authorization failures'
            ],
            
            dataProtection: [
                'Encrypt sensitive data at rest (AES-256)',
                'Use TLS 1.3 for data in transit',
                'Implement field-level encryption for PII',
                'Sanitize all user inputs',
                'Never expose sensitive data in URLs or logs'
            ],
            
            apiSecurity: [
                'Implement rate limiting per client',
                'Use CORS with strict origin policies',
                'Validate Content-Type headers',
                'Implement request size limits',
                'Use CSRF tokens for state-changing operations'
            ]
        };
    }

    static validateInput(input, schema) {
        // Use Joi or similar for validation
        const { error, value } = schema.validate(input, {
            abortEarly: false,
            stripUnknown: true
        });

        if (error) {
            throw new ValidationError(
                'Validation failed',
                error.details.map(d => ({
                    field: d.path.join('.'),
                    message: d.message
                }))
            );
        }

        return value;
    }
}

// 3. Performance Optimization
class PerformanceOptimization {
    static strategies = {
        caching: {
            description: 'Implement multi-layer caching',
            implementation: [
                'Use Redis for session and frequent data',
                'Implement HTTP caching with ETag/If-None-Match',
                'Cache at CDN level for static content',
                'Use Cache-Control headers appropriately',
                'Implement cache invalidation strategies'
            ]
        },
        
        databaseOptimization: [
            'Use connection pooling',
            'Create indexes on frequently queried fields',
            'Use read replicas for read-heavy operations',
            'Implement query result caching',
            'Use database query optimization tools',
            'Implement pagination for large datasets'
        ],
        
        apiOptimization: [
            'Compress responses with gzip/brotli',
            'Use HTTP/2 for multiplexing',
            'Implement GraphQL for flexible queries',
            'Use field filtering (?fields=id,name)',
            'Batch operations where possible',
            'Implement async processing for heavy tasks'
        ]
    };

    static async optimizeQuery(query, options = {}) {
        const {
            useCache = true,
            cacheKey,
            cacheTTL = 300,
            timeout = 5000
        } = options;

        if (useCache && cacheKey) {
            const cached = await redis.get(cacheKey);
            if (cached) return JSON.parse(cached);
        }

        const result = await Promise.race([
            query(),
            new Promise((_, reject) => 
                setTimeout(() => reject(new Error('Query timeout')), timeout)
            )
        ]);

        if (useCache && cacheKey) {
            await redis.setex(cacheKey, cacheTTL, JSON.stringify(result));
        }

        return result;
    }
}

// 4. Monitoring and Observability
class ObservabilityBestPractices {
    static setupMonitoring() {
        return {
            logging: [
                'Use structured logging (JSON format)',
                'Include correlation IDs in all logs',
                'Log request/response with sanitized data',
                'Use appropriate log levels',
                'Centralize logs with ELK or similar'
            ],
            
            metrics: [
                'Track request rate, latency, error rate (RED metrics)',
                'Monitor resource usage (CPU, memory, disk)',
                'Track business metrics (transactions, users)',
                'Set up dashboards in Grafana',
                'Export metrics to Prometheus'
            ],
            
            tracing: [
                'Implement distributed tracing with OpenTelemetry',
                'Track request flow across services',
                'Identify bottlenecks and latency sources',
                'Use Jaeger or Zipkin for visualization',
                'Add spans for critical operations'
            ],
            
            alerting: [
                'Set up alerts for SLO breaches',
                'Monitor error rate thresholds',
                'Alert on service health degradation',
                'Use PagerDuty or similar for on-call',
                'Implement escalation policies'
            ]
        };
    }
}

// 5. Documentation Standards
class DocumentationStandards {
    static requirements = {
        openapi: [
            'Maintain up-to-date OpenAPI 3.0 specs',
            'Document all endpoints, parameters, responses',
            'Include examples for all operations',
            'Document error responses',
            'Generate docs from code annotations'
        ],
        
        guides: [
            'Provide getting started guide',
            'Include authentication tutorial',
            'Document common use cases',
            'Provide code examples in multiple languages',
            'Maintain changelog for all versions'
        ],
        
        interactive: [
            'Provide Swagger UI for testing',
            'Include Postman collections',
            'Offer API sandbox environment',
            'Provide SDK/client libraries',
            'Include interactive tutorials'
        ]
    };
}

// 6. Testing Requirements
class TestingRequirements {
    static testPyramid = {
        unit: '70% - Fast, isolated tests',
        integration: '20% - Test component interactions',
        e2e: '10% - Full user journey tests'
    };

    static testTypes = [
        'Unit tests for business logic',
        'Integration tests for APIs',
        'Contract tests for service boundaries',
        'Performance/load tests',
        'Security penetration tests',
        'Chaos engineering tests'
    ];
}

// 7. Deployment Best Practices
class DeploymentBestPractices {
    static cicdPipeline = {
        stages: [
            '1. Code commit → trigger build',
            '2. Run unit tests',
            '3. Run integration tests',
            '4. Security scanning (SAST, dependency check)',
            '5. Build Docker image',
            '6. Push to registry',
            '7. Deploy to staging',
            '8. Run smoke tests',
            '9. Deploy to production (blue-green)',
            '10. Monitor and validate'
        ],
        
        strategies: [
            'Use blue-green deployment for zero downtime',
            'Implement canary releases for gradual rollout',
            'Maintain rollback capability',
            'Use feature flags for controlled releases',
            'Automate database migrations'
        ]
    };
}

// Summary Checklist
const productionReadinessChecklist = {
    design: ['✓ RESTful design', '✓ Versioning', '✓ Documentation'],
    security: ['✓ Authentication', '✓ Authorization', '✓ Encryption', '✓ Rate limiting'],
    performance: ['✓ Caching', '✓ DB optimization', '✓ Compression'],
    reliability: ['✓ Error handling', '✓ Circuit breakers', '✓ Retries', '✓ Health checks'],
    observability: ['✓ Logging', '✓ Metrics', '✓ Tracing', '✓ Alerting'],
    testing: ['✓ Unit tests', '✓ Integration tests', '✓ Load tests', '✓ Security tests'],
    deployment: ['✓ CI/CD', '✓ Zero-downtime', '✓ Rollback', '✓ Monitoring']
};

module.exports = {
    designPrinciples,
    SecurityBestPractices,
    PerformanceOptimization,
    ObservabilityBestPractices,
    DocumentationStandards,
    TestingRequirements,
    DeploymentBestPractices,
    productionReadinessChecklist
};
```

---
