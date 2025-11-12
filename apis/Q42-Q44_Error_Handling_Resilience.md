# APIs Interview Questions (Q42-Q44): Error Handling & Resilience

## Q42: How do you implement comprehensive error handling and recovery strategies?

**Answer:**

```javascript
// error-handling-service.js
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
}

class ValidationError extends APIError {
    constructor(message, details) {
        super(message, 400, 'VALIDATION_ERROR', details);
    }
}

class AuthenticationError extends APIError {
    constructor(message = 'Authentication failed') {
        super(message, 401, 'AUTHENTICATION_ERROR');
    }
}

class AuthorizationError extends APIError {
    constructor(message = 'Insufficient permissions') {
        super(message, 403, 'AUTHORIZATION_ERROR');
    }
}

class NotFoundError extends APIError {
    constructor(resource, id) {
        super(`${resource} with id ${id} not found`, 404, 'NOT_FOUND', { resource, id });
    }
}

class ConflictError extends APIError {
    constructor(message, details) {
        super(message, 409, 'CONFLICT', details);
    }
}

class RateLimitError extends APIError {
    constructor(retryAfter) {
        super('Rate limit exceeded', 429, 'RATE_LIMIT_EXCEEDED', { retryAfter });
        this.retryAfter = retryAfter;
    }
}

class ServiceUnavailableError extends APIError {
    constructor(service, message) {
        super(`Service ${service} is unavailable: ${message}`, 503, 'SERVICE_UNAVAILABLE', { service });
    }
}

class ErrorHandlingService {
    constructor(logger, alertingService) {
        this.logger = logger;
        this.alertingService = alertingService;
        this.errorMetrics = new Map();
    }

    handleError(error, req, res) {
        this.logError(error, req);
        this.trackErrorMetrics(error);
        
        if (this.shouldAlert(error)) {
            this.alertingService.sendAlert(error, req);
        }

        if (error instanceof APIError) {
            return this.sendErrorResponse(res, error);
        }

        if (error.name === 'ValidationError') {
            const validationError = new ValidationError(
                'Validation failed',
                error.details
            );
            return this.sendErrorResponse(res, validationError);
        }

        // Unhandled errors
        const internalError = new APIError(
            'Internal server error',
            500,
            'INTERNAL_ERROR',
            process.env.NODE_ENV === 'development' ? { stack: error.stack } : {}
        );
        
        return this.sendErrorResponse(res, internalError);
    }

    sendErrorResponse(res, error) {
        const response = {
            error: {
                code: error.errorCode,
                message: error.message,
                details: error.details,
                timestamp: error.timestamp,
                requestId: res.locals.requestId
            }
        };

        if (error instanceof RateLimitError) {
            res.setHeader('Retry-After', error.retryAfter);
        }

        res.status(error.statusCode).json(response);
    }

    logError(error, req) {
        this.logger.error({
            message: error.message,
            errorCode: error.errorCode,
            statusCode: error.statusCode,
            stack: error.stack,
            requestId: req.id,
            method: req.method,
            path: req.path,
            userId: req.user?.id,
            clientId: req.client?.id
        });
    }

    trackErrorMetrics(error) {
        const key = `${error.errorCode}_${error.statusCode}`;
        const count = this.errorMetrics.get(key) || 0;
        this.errorMetrics.set(key, count + 1);
    }

    shouldAlert(error) {
        if (error.statusCode >= 500) return true;
        
        const key = `${error.errorCode}_${error.statusCode}`;
        const count = this.errorMetrics.get(key) || 0;
        
        return count > 100; // Alert if error occurs more than 100 times
    }

    async retryWithBackoff(fn, maxRetries = 3, baseDelay = 1000) {
        for (let attempt = 0; attempt < maxRetries; attempt++) {
            try {
                return await fn();
            } catch (error) {
                if (attempt === maxRetries - 1) throw error;
                
                if (!this.isRetryable(error)) throw error;
                
                const delay = baseDelay * Math.pow(2, attempt);
                await this.sleep(delay);
            }
        }
    }

    isRetryable(error) {
        const retryableStatusCodes = [408, 429, 500, 502, 503, 504];
        return retryableStatusCodes.includes(error.statusCode);
    }

    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}

// Global error handling middleware
function errorHandlingMiddleware(errorService) {
    return (err, req, res, next) => {
        errorService.handleError(err, req, res);
    };
}

// Async handler wrapper
function asyncHandler(fn) {
    return (req, res, next) => {
        Promise.resolve(fn(req, res, next)).catch(next);
    };
}

module.exports = {
    APIError,
    ValidationError,
    AuthenticationError,
    AuthorizationError,
    NotFoundError,
    ConflictError,
    RateLimitError,
    ServiceUnavailableError,
    ErrorHandlingService,
    errorHandlingMiddleware,
    asyncHandler
};
```

---

## Q43: How do you implement circuit breakers and fallback mechanisms?

**Answer:**

```javascript
// circuit-breaker-service.js
const EventEmitter = require('events');

class CircuitBreaker extends EventEmitter {
    constructor(options = {}) {
        super();
        
        this.state = 'CLOSED';
        this.failureCount = 0;
        this.successCount = 0;
        this.nextAttempt = Date.now();
        
        this.failureThreshold = options.failureThreshold || 5;
        this.successThreshold = options.successThreshold || 2;
        this.timeout = options.timeout || 3000;
        this.resetTimeout = options.resetTimeout || 60000;
        
        this.fallback = options.fallback;
        this.onStateChange = options.onStateChange;
    }

    async execute(fn) {
        if (this.state === 'OPEN') {
            if (Date.now() < this.nextAttempt) {
                return this.executeFallback();
            }
            this.setState('HALF_OPEN');
        }

        try {
            const result = await this.executeWithTimeout(fn);
            this.onSuccess();
            return result;
        } catch (error) {
            this.onFailure();
            throw error;
        }
    }

    async executeWithTimeout(fn) {
        return Promise.race([
            fn(),
            new Promise((_, reject) => 
                setTimeout(() => reject(new Error('Timeout')), this.timeout)
            )
        ]);
    }

    onSuccess() {
        this.failureCount = 0;

        if (this.state === 'HALF_OPEN') {
            this.successCount++;
            
            if (this.successCount >= this.successThreshold) {
                this.setState('CLOSED');
                this.successCount = 0;
            }
        }
    }

    onFailure() {
        this.failureCount++;
        this.successCount = 0;

        if (this.state === 'HALF_OPEN' || this.failureCount >= this.failureThreshold) {
            this.setState('OPEN');
            this.nextAttempt = Date.now() + this.resetTimeout;
        }
    }

    setState(newState) {
        if (this.state !== newState) {
            const oldState = this.state;
            this.state = newState;
            
            this.emit('stateChange', { from: oldState, to: newState });
            
            if (this.onStateChange) {
                this.onStateChange(oldState, newState);
            }
        }
    }

    executeFallback() {
        if (this.fallback) {
            return this.fallback();
        }
        throw new Error('Circuit breaker is OPEN');
    }

    getState() {
        return {
            state: this.state,
            failureCount: this.failureCount,
            successCount: this.successCount,
            nextAttempt: this.nextAttempt
        };
    }
}

// Service-specific circuit breakers
class CircuitBreakerManager {
    constructor() {
        this.breakers = new Map();
    }

    getBreaker(serviceName, options = {}) {
        if (!this.breakers.has(serviceName)) {
            const breaker = new CircuitBreaker({
                ...options,
                onStateChange: (from, to) => {
                    console.log(`Circuit breaker for ${serviceName}: ${from} -> ${to}`);
                }
            });
            
            this.breakers.set(serviceName, breaker);
        }
        
        return this.breakers.get(serviceName);
    }

    async executeWithBreaker(serviceName, fn, fallback) {
        const breaker = this.getBreaker(serviceName, { fallback });
        return breaker.execute(fn);
    }

    getHealthStatus() {
        const status = {};
        
        for (const [service, breaker] of this.breakers) {
            status[service] = breaker.getState();
        }
        
        return status;
    }
}

// Usage example
const breakerManager = new CircuitBreakerManager();

async function callPaymentService(data) {
    return breakerManager.executeWithBreaker(
        'payment-service',
        async () => {
            const response = await axios.post('http://payment-service/api/process', data, {
                timeout: 3000
            });
            return response.data;
        },
        () => {
            // Fallback: Queue for later processing
            return {
                status: 'queued',
                message: 'Payment will be processed later',
                queueId: queuePayment(data)
            };
        }
    );
}

module.exports = { CircuitBreaker, CircuitBreakerManager };
```

---

## Q44: How do you implement request retry logic with idempotency?

**Answer:**

```javascript
// retry-service.js
const crypto = require('crypto');

class RetryService {
    constructor(redis) {
        this.redis = redis;
    }

    async executeWithRetry(fn, options = {}) {
        const {
            maxRetries = 3,
            baseDelay = 1000,
            maxDelay = 10000,
            backoffMultiplier = 2,
            retryableErrors = [408, 429, 500, 502, 503, 504],
            idempotencyKey = null
        } = options;

        let lastError;

        for (let attempt = 0; attempt <= maxRetries; attempt++) {
            try {
                if (idempotencyKey) {
                    const cachedResult = await this.getCachedResult(idempotencyKey);
                    if (cachedResult) {
                        return cachedResult;
                    }
                }

                const result = await fn(attempt);

                if (idempotencyKey) {
                    await this.cacheResult(idempotencyKey, result);
                }

                return result;
            } catch (error) {
                lastError = error;

                if (attempt === maxRetries) break;

                if (!this.shouldRetry(error, retryableErrors)) {
                    throw error;
                }

                const delay = Math.min(
                    baseDelay * Math.pow(backoffMultiplier, attempt),
                    maxDelay
                );

                await this.sleep(delay);
            }
        }

        throw lastError;
    }

    shouldRetry(error, retryableErrors) {
        if (error.response && retryableErrors.includes(error.response.status)) {
            return true;
        }

        if (error.code === 'ECONNRESET' || error.code === 'ETIMEDOUT') {
            return true;
        }

        return false;
    }

    async getCachedResult(idempotencyKey) {
        const cached = await this.redis.get(`idempotency:${idempotencyKey}`);
        return cached ? JSON.parse(cached) : null;
    }

    async cacheResult(idempotencyKey, result, ttl = 86400) {
        await this.redis.setex(
            `idempotency:${idempotencyKey}`,
            ttl,
            JSON.stringify(result)
        );
    }

    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}

class IdempotencyService {
    constructor(db) {
        this.db = db;
    }

    generateKey(method, url, body) {
        const data = `${method}:${url}:${JSON.stringify(body)}`;
        return crypto.createHash('sha256').update(data).digest('hex');
    }

    async checkIdempotency(key) {
        const result = await this.db.query(
            'SELECT * FROM idempotency_keys WHERE key = $1',
            [key]
        );

        if (result.rows.length > 0) {
            const record = result.rows[0];
            
            if (record.status === 'processing') {
                throw new ConflictError('Request is already being processed', {
                    key,
                    createdAt: record.created_at
                });
            }

            if (record.status === 'completed') {
                return {
                    cached: true,
                    response: record.response
                };
            }
        }

        await this.db.query(
            `INSERT INTO idempotency_keys (key, status, created_at) 
             VALUES ($1, 'processing', NOW())
             ON CONFLICT (key) DO NOTHING`,
            [key]
        );

        return { cached: false };
    }

    async saveResult(key, response) {
        await this.db.query(
            `UPDATE idempotency_keys 
             SET status = 'completed', 
                 response = $2, 
                 completed_at = NOW()
             WHERE key = $1`,
            [key, JSON.stringify(response)]
        );
    }

    async markFailed(key, error) {
        await this.db.query(
            `UPDATE idempotency_keys 
             SET status = 'failed', 
                 error = $2, 
                 completed_at = NOW()
             WHERE key = $1`,
            [key, JSON.stringify(error)]
        );
    }
}

// Middleware for idempotent requests
function idempotencyMiddleware(idempotencyService) {
    return async (req, res, next) => {
        if (req.method !== 'POST' && req.method !== 'PUT') {
            return next();
        }

        let idempotencyKey = req.headers['idempotency-key'];
        
        if (!idempotencyKey) {
            idempotencyKey = idempotencyService.generateKey(
                req.method,
                req.path,
                req.body
            );
        }

        try {
            const check = await idempotencyService.checkIdempotency(idempotencyKey);
            
            if (check.cached) {
                return res.json(check.response);
            }

            req.idempotencyKey = idempotencyKey;
            
            const originalJson = res.json.bind(res);
            res.json = async function(data) {
                await idempotencyService.saveResult(idempotencyKey, data);
                return originalJson(data);
            };

            next();
        } catch (error) {
            next(error);
        }
    };
}

// Database schema
/*
CREATE TABLE idempotency_keys (
    key VARCHAR(64) PRIMARY KEY,
    status VARCHAR(20) NOT NULL,
    response JSONB,
    error JSONB,
    created_at TIMESTAMP NOT NULL,
    completed_at TIMESTAMP,
    expires_at TIMESTAMP DEFAULT NOW() + INTERVAL '24 hours'
);

CREATE INDEX idx_idempotency_expires ON idempotency_keys(expires_at);
*/

module.exports = { RetryService, IdempotencyService, idempotencyMiddleware };
```

---
