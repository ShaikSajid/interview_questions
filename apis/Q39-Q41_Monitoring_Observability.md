# APIs Interview Questions (Q39-Q41): Monitoring & Observability

## Q39: How do you implement distributed tracing for microservices APIs?

**Answer:**

```javascript
// tracing-service.js
const opentelemetry = require('@opentelemetry/api');
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');
const { registerInstrumentations } = require('@opentelemetry/instrumentation');
const { HttpInstrumentation } = require('@opentelemetry/instrumentation-http');
const { ExpressInstrumentation } = require('@opentelemetry/instrumentation-express');

class DistributedTracingService {
    constructor() {
        this.provider = new NodeTracerProvider();
        
        const jaegerExporter = new JaegerExporter({
            serviceName: 'enbd-api-gateway',
            endpoint: 'http://localhost:14268/api/traces'
        });
        
        this.provider.addSpanProcessor(
            new opentelemetry.BatchSpanProcessor(jaegerExporter)
        );
        
        this.provider.register();
        
        registerInstrumentations({
            instrumentations: [
                new HttpInstrumentation(),
                new ExpressInstrumentation()
            ]
        });
        
        this.tracer = opentelemetry.trace.getTracer('enbd-api');
    }

    createSpan(name, attributes = {}) {
        return this.tracer.startSpan(name, { attributes });
    }

    async traceAPICall(operationName, fn) {
        const span = this.createSpan(operationName);
        
        try {
            const result = await fn(span);
            span.setStatus({ code: opentelemetry.SpanStatusCode.OK });
            return result;
        } catch (error) {
            span.setStatus({
                code: opentelemetry.SpanStatusCode.ERROR,
                message: error.message
            });
            span.recordException(error);
            throw error;
        } finally {
            span.end();
        }
    }

    injectTraceContext(headers = {}) {
        opentelemetry.propagation.inject(
            opentelemetry.context.active(),
            headers
        );
        return headers;
    }

    extractTraceContext(headers) {
        return opentelemetry.propagation.extract(
            opentelemetry.context.active(),
            headers
        );
    }
}

// Express middleware
function tracingMiddleware(req, res, next) {
    const tracingService = new DistributedTracingService();
    const span = tracingService.createSpan(`${req.method} ${req.path}`, {
        'http.method': req.method,
        'http.url': req.url,
        'http.client_ip': req.ip
    });
    
    req.span = span;
    
    res.on('finish', () => {
        span.setAttribute('http.status_code', res.statusCode);
        span.end();
    });
    
    next();
}

module.exports = { DistributedTracingService, tracingMiddleware };
```

---

## Q40: How do you implement API health checks and service discovery?

**Answer:**

```javascript
// health-check-service.js
class HealthCheckService {
    constructor(db, redis, serviceRegistry) {
        this.db = db;
        this.redis = redis;
        this.serviceRegistry = serviceRegistry;
        this.healthStatus = {
            status: 'healthy',
            checks: {}
        };
    }

    async performHealthCheck() {
        const checks = await Promise.allSettled([
            this.checkDatabase(),
            this.checkRedis(),
            this.checkExternalServices(),
            this.checkDiskSpace(),
            this.checkMemory()
        ]);

        this.healthStatus = {
            status: checks.every(c => c.status === 'fulfilled' && c.value.status === 'healthy') 
                ? 'healthy' : 'unhealthy',
            timestamp: new Date().toISOString(),
            checks: {
                database: checks[0].value,
                redis: checks[1].value,
                externalServices: checks[2].value,
                diskSpace: checks[3].value,
                memory: checks[4].value
            }
        };

        return this.healthStatus;
    }

    async checkDatabase() {
        try {
            const start = Date.now();
            await this.db.query('SELECT 1');
            const duration = Date.now() - start;

            return {
                status: duration < 1000 ? 'healthy' : 'degraded',
                responseTime: duration,
                message: `Database responding in ${duration}ms`
            };
        } catch (error) {
            return {
                status: 'unhealthy',
                message: error.message
            };
        }
    }

    async checkRedis() {
        try {
            const start = Date.now();
            await this.redis.ping();
            const duration = Date.now() - start;

            return {
                status: duration < 100 ? 'healthy' : 'degraded',
                responseTime: duration
            };
        } catch (error) {
            return {
                status: 'unhealthy',
                message: error.message
            };
        }
    }

    async checkExternalServices() {
        const services = ['payment-service', 'notification-service'];
        const checks = await Promise.all(
            services.map(service => this.checkService(service))
        );

        return {
            status: checks.every(c => c.status === 'healthy') ? 'healthy' : 'degraded',
            services: checks
        };
    }

    async checkService(serviceName) {
        try {
            const serviceUrl = await this.serviceRegistry.getServiceUrl(serviceName);
            const response = await axios.get(`${serviceUrl}/health`, { timeout: 3000 });
            
            return {
                name: serviceName,
                status: 'healthy',
                url: serviceUrl
            };
        } catch (error) {
            return {
                name: serviceName,
                status: 'unhealthy',
                error: error.message
            };
        }
    }

    checkDiskSpace() {
        const disk = require('diskusage');
        const path = process.platform === 'win32' ? 'c:' : '/';
        
        return new Promise((resolve) => {
            disk.check(path, (err, info) => {
                if (err) {
                    return resolve({ status: 'unhealthy', message: err.message });
                }
                
                const usedPercent = ((info.total - info.available) / info.total) * 100;
                
                resolve({
                    status: usedPercent < 90 ? 'healthy' : 'degraded',
                    used: usedPercent.toFixed(2) + '%',
                    available: (info.available / 1024 / 1024 / 1024).toFixed(2) + ' GB'
                });
            });
        });
    }

    checkMemory() {
        const used = process.memoryUsage();
        const total = require('os').totalmem();
        const usedPercent = (used.heapUsed / total) * 100;

        return {
            status: usedPercent < 80 ? 'healthy' : 'degraded',
            heapUsed: (used.heapUsed / 1024 / 1024).toFixed(2) + ' MB',
            heapTotal: (used.heapTotal / 1024 / 1024).toFixed(2) + ' MB',
            rss: (used.rss / 1024 / 1024).toFixed(2) + ' MB'
        };
    }
}

// Express routes
app.get('/health', async (req, res) => {
    const healthCheck = new HealthCheckService(db, redis, serviceRegistry);
    const status = await healthCheck.performHealthCheck();
    
    const statusCode = status.status === 'healthy' ? 200 : 503;
    res.status(statusCode).json(status);
});

app.get('/health/live', (req, res) => {
    res.status(200).json({ status: 'alive' });
});

app.get('/health/ready', async (req, res) => {
    const healthCheck = new HealthCheckService(db, redis, serviceRegistry);
    const status = await healthCheck.performHealthCheck();
    
    if (status.status === 'healthy') {
        res.status(200).json({ status: 'ready' });
    } else {
        res.status(503).json({ status: 'not ready', details: status });
    }
});

module.exports = { HealthCheckService };
```

---

## Q41: How do you implement API versioning and backward compatibility?

**Answer:**

```javascript
// versioning-service.js
class APIVersioningService {
    constructor() {
        this.versions = new Map();
        this.deprecationWarnings = new Map();
    }

    registerVersion(version, routes) {
        this.versions.set(version, routes);
    }

    getVersionFromRequest(req) {
        // Support multiple versioning strategies
        
        // 1. URL path versioning: /v1/users
        const pathMatch = req.path.match(/^\/v(\d+)\//);
        if (pathMatch) return pathMatch[1];
        
        // 2. Header versioning: API-Version: 1
        if (req.headers['api-version']) {
            return req.headers['api-version'];
        }
        
        // 3. Accept header versioning: Accept: application/vnd.enbd.v1+json
        const acceptMatch = req.headers.accept?.match(/vnd\.enbd\.v(\d+)/);
        if (acceptMatch) return acceptMatch[1];
        
        // 4. Query parameter: ?version=1
        if (req.query.version) return req.query.version;
        
        return '1'; // Default to v1
    }

    checkDeprecation(version, endpoint) {
        const key = `${version}:${endpoint}`;
        return this.deprecationWarnings.get(key);
    }

    deprecateEndpoint(version, endpoint, sunsetDate, migrationPath) {
        const key = `${version}:${endpoint}`;
        this.deprecationWarnings.set(key, {
            sunsetDate,
            migrationPath,
            message: `This endpoint is deprecated and will be removed on ${sunsetDate}`
        });
    }
}

// Versioning middleware
function versioningMiddleware(versionService) {
    return (req, res, next) => {
        const version = versionService.getVersionFromRequest(req);
        req.apiVersion = version;
        
        const deprecation = versionService.checkDeprecation(version, req.path);
        if (deprecation) {
            res.setHeader('Deprecation', 'true');
            res.setHeader('Sunset', deprecation.sunsetDate);
            res.setHeader('Link', `<${deprecation.migrationPath}>; rel="alternate"`);
        }
        
        next();
    };
}

// Version-specific route handlers
class UserServiceV1 {
    async getUser(userId) {
        const result = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
        return result.rows[0];
    }
}

class UserServiceV2 {
    async getUser(userId) {
        const result = await db.query(`
            SELECT u.*, p.bio, p.avatar_url 
            FROM users u 
            LEFT JOIN user_profiles p ON u.id = p.user_id 
            WHERE u.id = $1
        `, [userId]);
        
        return {
            ...result.rows[0],
            profile: {
                bio: result.rows[0].bio,
                avatarUrl: result.rows[0].avatar_url
            }
        };
    }
}

// Response transformation for backward compatibility
class ResponseTransformer {
    transformToV1(data) {
        // Remove new fields not present in v1
        const { profile, newField, ...v1Data } = data;
        return v1Data;
    }

    transformToV2(data) {
        // Keep all fields for v2
        return data;
    }

    transform(version, data) {
        switch (version) {
            case '1':
                return this.transformToV1(data);
            case '2':
                return this.transformToV2(data);
            default:
                return data;
        }
    }
}

// Usage in routes
const versionService = new APIVersioningService();
const transformer = new ResponseTransformer();

app.use(versioningMiddleware(versionService));

app.get('/users/:id', async (req, res) => {
    const version = req.apiVersion;
    let user;
    
    if (version === '1') {
        const service = new UserServiceV1();
        user = await service.getUser(req.params.id);
    } else {
        const service = new UserServiceV2();
        user = await service.getUser(req.params.id);
    }
    
    const transformed = transformer.transform(version, user);
    res.json(transformed);
});

// Deprecation setup
versionService.deprecateEndpoint(
    '1',
    '/users/:id',
    '2024-12-31',
    '/v2/users/:id'
);

module.exports = { APIVersioningService, ResponseTransformer };
```

---
