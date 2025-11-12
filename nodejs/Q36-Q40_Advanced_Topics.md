# Node.js Questions 36-40: Advanced Topics & Best Practices

---

### Q36. How do you implement request tracing and correlation IDs across microservices?

**Answer:**

**Request Tracing Implementation:**

```javascript
const { v4: uuidv4 } = require('uuid');

// Correlation ID Middleware
function correlationIdMiddleware(req, res, next) {
  // Get correlation ID from header or generate new one
  req.correlationId = req.headers['x-correlation-id'] || uuidv4();
  
  // Add to response headers
  res.setHeader('x-correlation-id', req.correlationId);
  
  // Add to request context
  req.context = {
    correlationId: req.correlationId,
    timestamp: new Date(),
    service: process.env.SERVICE_NAME || 'unknown'
  };
  
  next();
}

// Logger with correlation ID
class CorrelatedLogger {
  log(level, message, context) {
    console.log(JSON.stringify({
      level,
      message,
      correlationId: context.correlationId,
      service: context.service,
      timestamp: new Date().toISOString(),
      ...context
    }));
  }
  
  info(message, context) {
    this.log('INFO', message, context);
  }
  
  error(message, context) {
    this.log('ERROR', message, context);
  }
  
  warn(message, context) {
    this.log('WARN', message, context);
  }
}

// HTTP Client with correlation propagation
class TracedHTTPClient {
  async request(url, options, context) {
    const axios = require('axios');
    
    const headers = {
      ...options.headers,
      'x-correlation-id': context.correlationId,
      'x-request-id': uuidv4(),
      'x-calling-service': context.service
    };
    
    const startTime = Date.now();
    
    try {
      const response = await axios({
        ...options,
        url,
        headers
      });
      
      const duration = Date.now() - startTime;
      
      this.logRequest('success', {
        url,
        method: options.method || 'GET',
        duration,
        statusCode: response.status,
        correlationId: context.correlationId
      });
      
      return response;
    } catch (error) {
      const duration = Date.now() - startTime;
      
      this.logRequest('error', {
        url,
        method: options.method || 'GET',
        duration,
        error: error.message,
        correlationId: context.correlationId
      });
      
      throw error;
    }
  }
  
  logRequest(status, details) {
    console.log(JSON.stringify({
      type: 'HTTP_REQUEST',
      status,
      ...details,
      timestamp: new Date().toISOString()
    }));
  }
}

// Complete Banking Service with Tracing
class BankingService {
  constructor() {
    this.logger = new CorrelatedLogger();
    this.httpClient = new TracedHTTPClient();
  }
  
  setupExpress() {
    const express = require('express');
    const app = express();
    
    app.use(express.json());
    app.use(correlationIdMiddleware);
    
    // Request logging middleware
    app.use((req, res, next) => {
      this.logger.info('Incoming request', {
        ...req.context,
        method: req.method,
        path: req.path,
        ip: req.ip
      });
      
      // Log response
      const originalSend = res.send;
      res.send = function(data) {
        this.logger.info('Outgoing response', {
          ...req.context,
          statusCode: res.statusCode,
          path: req.path
        });
        originalSend.call(res, data);
      }.bind(this);
      
      next();
    });
    
    app.post('/api/transfer', async (req, res) => {
      try {
        const result = await this.processTransfer(req.body, req.context);
        res.json(result);
      } catch (error) {
        this.logger.error('Transfer failed', {
          ...req.context,
          error: error.message,
          stack: error.stack
        });
        res.status(500).json({ error: error.message });
      }
    });
    
    return app;
  }
  
  async processTransfer(data, context) {
    this.logger.info('Processing transfer', {
      ...context,
      from: data.from,
      to: data.to,
      amount: data.amount
    });
    
    // Call Account Service with correlation
    const accountResponse = await this.httpClient.request(
      'http://account-service/api/accounts/' + data.from,
      { method: 'GET' },
      context
    );
    
    // Call Transaction Service with correlation
    const txnResponse = await this.httpClient.request(
      'http://transaction-service/api/transactions',
      {
        method: 'POST',
        data: {
          from: data.from,
          to: data.to,
          amount: data.amount
        }
      },
      context
    );
    
    this.logger.info('Transfer completed', {
      ...context,
      transactionId: txnResponse.data.id
    });
    
    return {
      success: true,
      transactionId: txnResponse.data.id,
      correlationId: context.correlationId
    };
  }
}
```

### Q37. How do you implement health checks and readiness probes?

**Answer:**

```javascript
class HealthCheckService {
  constructor() {
    this.dependencies = {
      database: { healthy: false, lastCheck: null },
      redis: { healthy: false, lastCheck: null },
      externalAPI: { healthy: false, lastCheck: null }
    };
    
    this.startTime = Date.now();
    this.ready = false;
    
    // Start health checks
    this.startHealthChecks();
  }
  
  setupRoutes(app) {
    // Liveness probe - is the service alive?
    app.get('/health/live', (req, res) => {
      res.json({
        status: 'UP',
        uptime: process.uptime(),
        timestamp: new Date().toISOString()
      });
    });
    
    // Readiness probe - is the service ready to handle traffic?
    app.get('/health/ready', (req, res) => {
      const checks = this.getHealthChecks();
      const allHealthy = Object.values(checks).every(c => c.healthy);
      
      if (allHealthy && this.ready) {
        res.json({
          status: 'READY',
          checks,
          timestamp: new Date().toISOString()
        });
      } else {
        res.status(503).json({
          status: 'NOT_READY',
          checks,
          timestamp: new Date().toISOString()
        });
      }
    });
    
    // Detailed health check
    app.get('/health', async (req, res) => {
      const checks = await this.performHealthChecks();
      const allHealthy = Object.values(checks).every(c => c.healthy);
      
      res.status(allHealthy ? 200 : 503).json({
        status: allHealthy ? 'HEALTHY' : 'UNHEALTHY',
        uptime: process.uptime(),
        memory: process.memoryUsage(),
        checks,
        timestamp: new Date().toISOString()
      });
    });
  }
  
  async performHealthChecks() {
    const results = {};
    
    // Check database
    try {
      await this.checkDatabase();
      results.database = { healthy: true, message: 'Connected' };
    } catch (error) {
      results.database = { healthy: false, message: error.message };
    }
    
    // Check Redis
    try {
      await this.checkRedis();
      results.redis = { healthy: true, message: 'Connected' };
    } catch (error) {
      results.redis = { healthy: false, message: error.message };
    }
    
    // Check external API
    try {
      await this.checkExternalAPI();
      results.externalAPI = { healthy: true, message: 'Reachable' };
    } catch (error) {
      results.externalAPI = { healthy: false, message: error.message };
    }
    
    return results;
  }
  
  startHealthChecks() {
    // Check every 30 seconds
    setInterval(async () => {
      const checks = await this.performHealthChecks();
      this.dependencies = checks;
      
      // Service is ready if all dependencies are healthy
      this.ready = Object.values(checks).every(c => c.healthy);
    }, 30000);
  }
  
  async checkDatabase() {
    // Simulate DB health check
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        Math.random() > 0.1 ? resolve() : reject(new Error('DB unreachable'));
      }, 100);
    });
  }
  
  async checkRedis() {
    // Simulate Redis health check
    return new Promise((resolve) => {
      setTimeout(resolve, 50);
    });
  }
  
  async checkExternalAPI() {
    // Simulate external API health check
    return new Promise((resolve) => {
      setTimeout(resolve, 100);
    });
  }
  
  getHealthChecks() {
    return this.dependencies;
  }
}
```

### Q38. How do you implement rate limiting at application level?

**Answer:**

```javascript
// Token Bucket Rate Limiter
class TokenBucketRateLimiter {
  constructor(capacity, refillRate) {
    this.capacity = capacity; // Max tokens
    this.tokens = capacity;
    this.refillRate = refillRate; // Tokens per second
    this.lastRefill = Date.now();
  }
  
  async consume(tokens = 1) {
    this.refill();
    
    if (this.tokens >= tokens) {
      this.tokens -= tokens;
      return true;
    }
    
    return false;
  }
  
  refill() {
    const now = Date.now();
    const timePassed = (now - this.lastRefill) / 1000;
    const tokensToAdd = timePassed * this.refillRate;
    
    this.tokens = Math.min(this.capacity, this.tokens + tokensToAdd);
    this.lastRefill = now;
  }
}

// Sliding Window Rate Limiter
class SlidingWindowRateLimiter {
  constructor(maxRequests, windowMs) {
    this.maxRequests = maxRequests;
    this.windowMs = windowMs;
    this.requests = new Map(); // userId -> array of timestamps
  }
  
  async isAllowed(userId) {
    const now = Date.now();
    const windowStart = now - this.windowMs;
    
    // Get user's requests
    let userRequests = this.requests.get(userId) || [];
    
    // Remove old requests outside window
    userRequests = userRequests.filter(timestamp => timestamp > windowStart);
    
    // Check if under limit
    if (userRequests.length < this.maxRequests) {
      userRequests.push(now);
      this.requests.set(userId, userRequests);
      return true;
    }
    
    return false;
  }
  
  cleanup() {
    const now = Date.now();
    
    for (const [userId, requests] of this.requests.entries()) {
      const valid = requests.filter(timestamp => timestamp > now - this.windowMs);
      
      if (valid.length === 0) {
        this.requests.delete(userId);
      } else {
        this.requests.set(userId, valid);
      }
    }
  }
}

// Express Middleware
function createRateLimiter(options = {}) {
  const limiter = new SlidingWindowRateLimiter(
    options.maxRequests || 100,
    options.windowMs || 60000
  );
  
  // Cleanup every minute
  setInterval(() => limiter.cleanup(), 60000);
  
  return async (req, res, next) => {
    const userId = req.user?.id || req.ip;
    
    const allowed = await limiter.isAllowed(userId);
    
    if (allowed) {
      next();
    } else {
      res.status(429).json({
        error: 'Too many requests',
        message: 'Rate limit exceeded. Please try again later.',
        retryAfter: options.windowMs / 1000
      });
    }
  };
}

// Banking API with Rate Limiting
const express = require('express');
const app = express();

// Global rate limiter: 1000 requests per minute
app.use(createRateLimiter({
  maxRequests: 1000,
  windowMs: 60000
}));

// Specific endpoint rate limiter: 10 transfers per minute
app.post('/api/transfer',
  createRateLimiter({ maxRequests: 10, windowMs: 60000 }),
  async (req, res) => {
    // Process transfer
    res.json({ success: true });
  }
);
```

### Q39. How do you implement distributed caching with Redis?

**Answer:**

```javascript
const redis = require('redis');
const { promisify } = require('util');

class RedisCacheService {
  constructor(options = {}) {
    this.client = redis.createClient(options);
    
    // Promisify Redis methods
    this.get = promisify(this.client.get).bind(this.client);
    this.set = promisify(this.client.set).bind(this.client);
    this.del = promisify(this.client.del).bind(this.client);
    this.setex = promisify(this.client.setex).bind(this.client);
    this.exists = promisify(this.client.exists).bind(this.client);
    this.ttl = promisify(this.client.ttl).bind(this.client);
    
    this.client.on('error', (err) => {
      console.error('Redis error:', err);
    });
    
    this.client.on('connect', () => {
      console.log('Redis connected');
    });
  }
  
  // Cache with TTL
  async cacheWithTTL(key, value, ttl = 3600) {
    const serialized = JSON.stringify(value);
    await this.setex(key, ttl, serialized);
  }
  
  // Get cached value
  async getCached(key) {
    const value = await this.get(key);
    return value ? JSON.parse(value) : null;
  }
  
  // Cache-aside pattern
  async getOrFetch(key, fetchFn, ttl = 3600) {
    // Try cache first
    let value = await this.getCached(key);
    
    if (value !== null) {
      console.log('Cache hit:', key);
      return value;
    }
    
    console.log('Cache miss:', key);
    
    // Fetch from source
    value = await fetchFn();
    
    // Cache result
    await this.cacheWithTTL(key, value, ttl);
    
    return value;
  }
  
  // Invalidate cache
  async invalidate(pattern) {
    if (pattern.includes('*')) {
      // Pattern-based deletion
      const keys = await this.scanKeys(pattern);
      if (keys.length > 0) {
        await this.del(...keys);
      }
    } else {
      await this.del(pattern);
    }
  }
  
  // Scan for keys matching pattern
  async scanKeys(pattern) {
    const keys = [];
    let cursor = '0';
    
    do {
      const reply = await promisify(this.client.scan).bind(this.client)(
        cursor,
        'MATCH',
        pattern,
        'COUNT',
        100
      );
      
      cursor = reply[0];
      keys.push(...reply[1]);
    } while (cursor !== '0');
    
    return keys;
  }
  
  async close() {
    return promisify(this.client.quit).bind(this.client)();
  }
}

// Banking Service with Redis Cache
class CachedBankingService {
  constructor() {
    this.cache = new RedisCacheService({
      host: process.env.REDIS_HOST || 'localhost',
      port: process.env.REDIS_PORT || 6379
    });
  }
  
  async getAccount(accountId) {
    const cacheKey = `account:${accountId}`;
    
    return this.cache.getOrFetch(
      cacheKey,
      async () => {
        // Fetch from database
        console.log('Fetching from DB:', accountId);
        return {
          id: accountId,
          balance: Math.random() * 100000,
          type: 'savings'
        };
      },
      300 // 5 minutes TTL
    );
  }
  
  async updateAccount(accountId, data) {
    // Update database
    console.log('Updating account:', accountId);
    
    // Invalidate cache
    await this.cache.invalidate(`account:${accountId}`);
    
    return { success: true };
  }
  
  async getAccountTransactions(accountId, page = 1) {
    const cacheKey = `transactions:${accountId}:page:${page}`;
    
    return this.cache.getOrFetch(
      cacheKey,
      async () => {
        console.log('Fetching transactions from DB');
        return [
          { id: 'TXN-1', amount: 100 },
          { id: 'TXN-2', amount: 200 }
        ];
      },
      60 // 1 minute TTL
    );
  }
  
  async invalidateAccountCache(accountId) {
    // Invalidate all cache keys for this account
    await this.cache.invalidate(`account:${accountId}*`);
    await this.cache.invalidate(`transactions:${accountId}*`);
  }
}
```

### Q40. How do you implement graceful degradation and fallback mechanisms?

**Answer:**

```javascript
class FallbackService {
  constructor() {
    this.primaryService = new PrimaryBankingService();
    this.fallbackService = new FallbackBankingService();
    this.cache = new Map();
  }
  
  async getAccountBalance(accountId) {
    try {
      // Try primary service
      const balance = await this.primaryService.getBalance(accountId);
      
      // Cache successful response
      this.cache.set(accountId, {
        balance,
        timestamp: Date.now()
      });
      
      return balance;
    } catch (error) {
      console.warn('Primary service failed, trying fallback:', error.message);
      
      try {
        // Try fallback service
        return await this.fallbackService.getBalance(accountId);
      } catch (fallbackError) {
        console.error('Fallback also failed:', fallbackError.message);
        
        // Return cached value if available
        const cached = this.cache.get(accountId);
        if (cached && Date.now() - cached.timestamp < 300000) { // 5 min
          console.log('Returning stale cache');
          return cached.balance;
        }
        
        // Last resort: return default
        console.error('No fallback available, returning default');
        return 0;
      }
    }
  }
  
  async processTransferWithDegradation(transfer) {
    const degradationLevel = await this.getDegradationLevel();
    
    switch (degradationLevel) {
      case 'NORMAL':
        return this.processTransferFull(transfer);
        
      case 'DEGRADED':
        return this.processTransferBasic(transfer);
        
      case 'CRITICAL':
        return this.processTransferMinimal(transfer);
        
      case 'OFFLINE':
        throw new Error('Service temporarily unavailable');
    }
  }
  
  async processTransferFull(transfer) {
    // Full processing with all validations and notifications
    await this.validateAccount(transfer.from);
    await this.validateAccount(transfer.to);
    await this.checkFraudScore(transfer);
    await this.updateAccounts(transfer);
    await this.sendNotifications(transfer);
    return { success: true, mode: 'FULL' };
  }
  
  async processTransferBasic(transfer) {
    // Skip non-critical steps
    await this.validateAccount(transfer.from);
    await this.updateAccounts(transfer);
    // Skip notifications
    return { success: true, mode: 'DEGRADED' };
  }
  
  async processTransferMinimal(transfer) {
    // Only essential operations
    await this.updateAccounts(transfer);
    return { success: true, mode: 'MINIMAL' };
  }
  
  async getDegradationLevel() {
    // Check system health
    const cpuUsage = process.cpuUsage();
    const memUsage = process.memoryUsage();
    
    if (memUsage.heapUsed / memUsage.heapTotal > 0.9) {
      return 'CRITICAL';
    } else if (memUsage.heapUsed / memUsage.heapTotal > 0.7) {
      return 'DEGRADED';
    }
    
    return 'NORMAL';
  }
  
  async validateAccount(accountId) {
    // Validation logic
  }
  
  async checkFraudScore(transfer) {
    // Fraud detection
  }
  
  async updateAccounts(transfer) {
    // Update balances
  }
  
  async sendNotifications(transfer) {
    // Send notifications
  }
}

class PrimaryBankingService {
  async getBalance(accountId) {
    if (Math.random() < 0.2) {
      throw new Error('Primary service unavailable');
    }
    return 10000;
  }
}

class FallbackBankingService {
  async getBalance(accountId) {
    if (Math.random() < 0.1) {
      throw new Error('Fallback service unavailable');
    }
    return 9500; // Slightly stale data
  }
}
```

---

**🎯 Section 6 Complete! Questions 36-40 Finished**

Topics: Request tracing, Health checks, Rate limiting, Redis caching, Graceful degradation
