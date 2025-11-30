# Distributed Systems - Interview Preparation

## Top 7 Interview Questions with Examples

### 1. Explain the CAP Theorem and how it applies to real-world systems

**Answer:**
CAP Theorem states that a distributed system can only guarantee two of three properties: Consistency, Availability, and Partition Tolerance.

**Example:**

```javascript
// CP System (Consistency + Partition Tolerance)
// Example: Banking system - must have consistency
class BankingService {
  async transferMoney(fromAccount, toAccount, amount) {
    const transaction = await this.startTransaction();
    
    try {
      await transaction.lock([fromAccount, toAccount]);
      await transaction.debit(fromAccount, amount);
      await transaction.credit(toAccount, amount);
      await transaction.commit();
      return { success: true };
    } catch (error) {
      await transaction.rollback();
      throw new Error('Transaction failed - consistency maintained');
    }
  }
}

// AP System (Availability + Partition Tolerance)
// Example: Social media feed - prefers availability
class SocialFeedService {
  async getUserFeed(userId) {
    try {
      return await this.primaryDC.getFeed(userId);
    } catch (error) {
      // Primary down? Serve from replica (may be stale)
      return await this.replicaDC.getFeed(userId);
    }
  }
  
  async postUpdate(userId, content) {
    const availableNode = await this.findAvailableNode();
    await availableNode.write(content);
    this.asyncReplicateToOtherNodes(content);
    return { success: true };
  }
}
```

---

### 2. How do you handle eventual consistency and conflict resolution?

**Answer:**
Eventual consistency allows temporary inconsistencies that resolve over time.

**Example:**

```javascript
// Vector Clocks for Conflict Detection
class VectorClock {
  constructor(nodeId) {
    this.nodeId = nodeId;
    this.clock = {};
  }
  
  increment() {
    this.clock[this.nodeId] = (this.clock[this.nodeId] || 0) + 1;
  }
  
  merge(other) {
    for (const [node, value] of Object.entries(other.clock)) {
      this.clock[node] = Math.max(this.clock[node] || 0, value);
    }
  }
  
  happensBefore(other) {
    let allLessOrEqual = true;
    let atLeastOneLess = false;
    
    for (const node in { ...this.clock, ...other.clock }) {
      const thisValue = this.clock[node] || 0;
      const otherValue = other.clock[node] || 0;
      
      if (thisValue > otherValue) {
        allLessOrEqual = false;
      }
      if (thisValue < otherValue) {
        atLeastOneLess = true;
      }
    }
    
    return allLessOrEqual && atLeastOneLess;
  }
  
  isConcurrent(other) {
    return !this.happensBefore(other) && !other.happensBefore(this);
  }
}

// CRDT (Conflict-free Replicated Data Type) - G-Counter
class GCounter {
  constructor(nodeId) {
    this.nodeId = nodeId;
    this.counts = {};
  }
  
  increment() {
    this.counts[this.nodeId] = (this.counts[this.nodeId] || 0) + 1;
  }
  
  value() {
    return Object.values(this.counts).reduce((a, b) => a + b, 0);
  }
  
  merge(other) {
    for (const [node, count] of Object.entries(other.counts)) {
      this.counts[node] = Math.max(this.counts[node] || 0, count);
    }
  }
}

// Last-Write-Wins with Timestamps
class LWWRegister {
  constructor(value = null) {
    this.value = value;
    this.timestamp = Date.now();
    this.nodeId = Math.random().toString(36);
  }
  
  set(value) {
    this.value = value;
    this.timestamp = Date.now();
  }
  
  merge(other) {
    if (other.timestamp > this.timestamp) {
      this.value = other.value;
      this.timestamp = other.timestamp;
    } else if (other.timestamp === this.timestamp) {
      // Tie-breaker using nodeId
      if (other.nodeId > this.nodeId) {
        this.value = other.value;
        this.nodeId = other.nodeId;
      }
    }
  }
}
```

---

### 3. Explain different load balancing strategies

**Answer:**
Load balancing distributes requests across multiple servers.

**Example:**

```javascript
// Round Robin
class RoundRobinLoadBalancer {
  constructor(servers) {
    this.servers = servers;
    this.currentIndex = 0;
  }
  
  getNextServer() {
    const server = this.servers[this.currentIndex];
    this.currentIndex = (this.currentIndex + 1) % this.servers.length;
    return server;
  }
}

// Weighted Round Robin
class WeightedRoundRobinLoadBalancer {
  constructor(servers) {
    // servers: [{ url: 'server1', weight: 3 }, { url: 'server2', weight: 1 }]
    this.servers = [];
    servers.forEach(server => {
      for (let i = 0; i < server.weight; i++) {
        this.servers.push(server.url);
      }
    });
    this.currentIndex = 0;
  }
  
  getNextServer() {
    const server = this.servers[this.currentIndex];
    this.currentIndex = (this.currentIndex + 1) % this.servers.length;
    return server;
  }
}

// Least Connections
class LeastConnectionsLoadBalancer {
  constructor(servers) {
    this.servers = servers.map(url => ({
      url,
      activeConnections: 0
    }));
  }
  
  getNextServer() {
    const server = this.servers.reduce((min, current) =>
      current.activeConnections < min.activeConnections ? current : min
    );
    server.activeConnections++;
    return server.url;
  }
  
  releaseConnection(serverUrl) {
    const server = this.servers.find(s => s.url === serverUrl);
    if (server) {
      server.activeConnections--;
    }
  }
}

// Consistent Hashing (for cache distribution)
class ConsistentHashLoadBalancer {
  constructor(servers, virtualNodes = 150) {
    this.ring = [];
    this.virtualNodes = virtualNodes;
    
    servers.forEach(server => {
      for (let i = 0; i < virtualNodes; i++) {
        const hash = this.hash(`${server}-${i}`);
        this.ring.push({ hash, server });
      }
    });
    
    this.ring.sort((a, b) => a.hash - b.hash);
  }
  
  getServer(key) {
    if (this.ring.length === 0) return null;
    
    const hash = this.hash(key);
    
    // Find first server with hash >= key hash
    for (const node of this.ring) {
      if (node.hash >= hash) {
        return node.server;
      }
    }
    
    // Wrap around to first server
    return this.ring[0].server;
  }
  
  hash(key) {
    let hash = 0;
    for (let i = 0; i < key.length; i++) {
      hash = ((hash << 5) - hash) + key.charCodeAt(i);
      hash = hash & hash;
    }
    return Math.abs(hash);
  }
  
  addServer(server) {
    for (let i = 0; i < this.virtualNodes; i++) {
      const hash = this.hash(`${server}-${i}`);
      this.ring.push({ hash, server });
    }
    this.ring.sort((a, b) => a.hash - b.hash);
  }
  
  removeServer(server) {
    this.ring = this.ring.filter(node => node.server !== server);
  }
}

// Adaptive Load Balancer (based on response time)
class AdaptiveLoadBalancer {
  constructor(servers) {
    this.servers = servers.map(url => ({
      url,
      responseTime: 0,
      requestCount: 0
    }));
  }
  
  async makeRequest(url) {
    const server = this.servers.find(s => s.url === url);
    const start = Date.now();
    
    try {
      const response = await fetch(url);
      const duration = Date.now() - start;
      
      // Update moving average
      server.responseTime = (server.responseTime * server.requestCount + duration) 
                           / (server.requestCount + 1);
      server.requestCount++;
      
      return response;
    } catch (error) {
      server.responseTime = Infinity; // Mark as unavailable
      throw error;
    }
  }
  
  getNextServer() {
    // Choose server with lowest average response time
    const server = this.servers.reduce((min, current) =>
      current.responseTime < min.responseTime ? current : min
    );
    return server.url;
  }
}
```

---

### 4. How do you implement distributed caching?

**Answer:**
Distributed caching improves performance and reduces database load.

**Example:**

```javascript
// Redis-based Distributed Cache
class DistributedCache {
  constructor(redisClient) {
    this.redis = redisClient;
    this.localCache = new Map();
    this.localCacheTTL = 5000; // 5 seconds
  }
  
  async get(key) {
    // Check local cache first (L1)
    const localValue = this.localCache.get(key);
    if (localValue && Date.now() < localValue.expiry) {
      return localValue.data;
    }
    
    // Check Redis (L2)
    const redisValue = await this.redis.get(key);
    if (redisValue) {
      // Update local cache
      this.localCache.set(key, {
        data: JSON.parse(redisValue),
        expiry: Date.now() + this.localCacheTTL
      });
      return JSON.parse(redisValue);
    }
    
    return null;
  }
  
  async set(key, value, ttl = 3600) {
    // Set in Redis
    await this.redis.setex(key, ttl, JSON.stringify(value));
    
    // Set in local cache
    this.localCache.set(key, {
      data: value,
      expiry: Date.now() + this.localCacheTTL
    });
  }
  
  async delete(key) {
    await this.redis.del(key);
    this.localCache.delete(key);
  }
  
  async invalidatePattern(pattern) {
    const keys = await this.redis.keys(pattern);
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }
    // Clear local cache
    this.localCache.clear();
  }
}

// Cache-Aside Pattern
class CacheAsideRepository {
  constructor(cache, database) {
    this.cache = cache;
    this.database = database;
  }
  
  async getUser(userId) {
    const cacheKey = `user:${userId}`;
    
    // Try cache first
    let user = await this.cache.get(cacheKey);
    if (user) {
      return user;
    }
    
    // Cache miss - fetch from database
    user = await this.database.findById(userId);
    if (user) {
      await this.cache.set(cacheKey, user, 3600);
    }
    
    return user;
  }
  
  async updateUser(userId, updates) {
    // Update database
    const user = await this.database.update(userId, updates);
    
    // Invalidate cache
    await this.cache.delete(`user:${userId}`);
    
    return user;
  }
}

// Write-Through Cache
class WriteThroughCache {
  async set(key, value) {
    // Write to cache and database simultaneously
    await Promise.all([
      this.cache.set(key, value),
      this.database.set(key, value)
    ]);
  }
}

// Write-Behind (Write-Back) Cache
class WriteBehindCache {
  constructor(cache, database) {
    this.cache = cache;
    this.database = database;
    this.writeQueue = [];
    this.flushInterval = 5000; // Flush every 5 seconds
    
    this.startFlushing();
  }
  
  async set(key, value) {
    // Write to cache immediately
    await this.cache.set(key, value);
    
    // Queue database write
    this.writeQueue.push({ key, value, timestamp: Date.now() });
  }
  
  startFlushing() {
    setInterval(async () => {
      if (this.writeQueue.length > 0) {
        const batch = this.writeQueue.splice(0, 100);
        await this.flushBatch(batch);
      }
    }, this.flushInterval);
  }
  
  async flushBatch(batch) {
    const promises = batch.map(item =>
      this.database.set(item.key, item.value)
    );
    await Promise.all(promises);
  }
}

// Cache Stampede Prevention
class StampedeProtectedCache {
  constructor(cache, database) {
    this.cache = cache;
    this.database = database;
    this.locks = new Map();
  }
  
  async get(key) {
    let value = await this.cache.get(key);
    if (value) return value;
    
    // Check if another request is already fetching
    if (this.locks.has(key)) {
      return await this.locks.get(key);
    }
    
    // Create promise for this fetch
    const fetchPromise = this.fetchAndCache(key);
    this.locks.set(key, fetchPromise);
    
    try {
      value = await fetchPromise;
      return value;
    } finally {
      this.locks.delete(key);
    }
  }
  
  async fetchAndCache(key) {
    const value = await this.database.get(key);
    if (value) {
      await this.cache.set(key, value, 3600);
    }
    return value;
  }
}
```

---

### 5. Explain circuit breaker pattern and implementation

**Answer:**
Circuit breaker prevents cascading failures in distributed systems.

**Example:**

```javascript
class CircuitBreaker {
  constructor(options = {}) {
    this.failureThreshold = options.failureThreshold || 5;
    this.successThreshold = options.successThreshold || 2;
    this.timeout = options.timeout || 60000;
    this.resetTimeout = options.resetTimeout || 30000;
    
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.failureCount = 0;
    this.successCount = 0;
    this.nextAttempt = Date.now();
  }
  
  async execute(operation) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN');
      }
      this.state = 'HALF_OPEN';
      this.successCount = 0;
    }
    
    try {
      const result = await this.executeWithTimeout(operation, this.timeout);
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  async executeWithTimeout(operation, timeout) {
    return Promise.race([
      operation(),
      new Promise((_, reject) =>
        setTimeout(() => reject(new Error('Operation timeout')), timeout)
      )
    ]);
  }
  
  onSuccess() {
    this.failureCount = 0;
    
    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      if (this.successCount >= this.successThreshold) {
        this.state = 'CLOSED';
        console.log('Circuit breaker CLOSED');
      }
    }
  }
  
  onFailure() {
    this.failureCount++;
    
    if (this.failureCount >= this.failureThreshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.resetTimeout;
      console.log(`Circuit breaker OPEN for ${this.resetTimeout}ms`);
    }
  }
  
  getState() {
    return {
      state: this.state,
      failureCount: this.failureCount,
      successCount: this.successCount
    };
  }
}

// Usage with external API
class ExternalAPIClient {
  constructor(baseUrl) {
    this.baseUrl = baseUrl;
    this.circuitBreaker = new CircuitBreaker({
      failureThreshold: 5,
      successThreshold: 2,
      timeout: 5000,
      resetTimeout: 30000
    });
  }
  
  async fetchData(endpoint) {
    try {
      return await this.circuitBreaker.execute(async () => {
        const response = await fetch(`${this.baseUrl}${endpoint}`);
        if (!response.ok) {
          throw new Error(`HTTP ${response.status}`);
        }
        return await response.json();
      });
    } catch (error) {
      console.error('API call failed:', error.message);
      
      // Return cached data or default
      return this.getFallbackData(endpoint);
    }
  }
  
  getFallbackData(endpoint) {
    // Return cached or default data
    return { cached: true, data: [] };
  }
}
```

---

### 6. How do you implement distributed rate limiting?

**Answer:**
Distributed rate limiting coordinates limits across multiple instances.

**Example:**

```javascript
// Redis-based Sliding Window Rate Limiter
class DistributedRateLimiter {
  constructor(redis) {
    this.redis = redis;
  }
  
  async isAllowed(key, maxRequests, windowSeconds) {
    const now = Date.now();
    const windowStart = now - (windowSeconds * 1000);
    const redisKey = `rate_limit:${key}`;
    
    // Remove old entries
    await this.redis.zremrangebyscore(redisKey, 0, windowStart);
    
    // Count current requests
    const requestCount = await this.redis.zcard(redisKey);
    
    if (requestCount < maxRequests) {
      // Add current request
      await this.redis.zadd(redisKey, now, `${now}-${Math.random()}`);
      await this.redis.expire(redisKey, windowSeconds);
      
      return {
        allowed: true,
        remaining: maxRequests - requestCount - 1,
        resetAt: now + (windowSeconds * 1000)
      };
    }
    
    // Rate limit exceeded
    const oldest = await this.redis.zrange(redisKey, 0, 0, 'WITHSCORES');
    const resetAt = parseInt(oldest[1]) + (windowSeconds * 1000);
    
    return {
      allowed: false,
      remaining: 0,
      resetAt,
      retryAfter: Math.ceil((resetAt - now) / 1000)
    };
  }
}

// Token Bucket with Redis
class TokenBucketRateLimiter {
  constructor(redis) {
    this.redis = redis;
  }
  
  async tryConsume(key, tokens, capacity, refillRate) {
    const script = `
      local key = KEYS[1]
      local capacity = tonumber(ARGV[1])
      local tokens_requested = tonumber(ARGV[2])
      local refill_rate = tonumber(ARGV[3])
      local now = tonumber(ARGV[4])
      
      local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
      local current_tokens = tonumber(bucket[1]) or capacity
      local last_refill = tonumber(bucket[2]) or now
      
      -- Refill tokens
      local time_passed = now - last_refill
      local tokens_to_add = time_passed * refill_rate / 1000
      current_tokens = math.min(capacity, current_tokens + tokens_to_add)
      
      if current_tokens >= tokens_requested then
        current_tokens = current_tokens - tokens_requested
        redis.call('HMSET', key, 'tokens', current_tokens, 'last_refill', now)
        redis.call('EXPIRE', key, 3600)
        return {1, current_tokens}
      else
        return {0, current_tokens}
      end
    `;
    
    const result = await this.redis.eval(
      script,
      1,
      `token_bucket:${key}`,
      capacity,
      tokens,
      refillRate,
      Date.now()
    );
    
    return {
      allowed: result[0] === 1,
      remaining: Math.floor(result[1])
    };
  }
}

// Express Middleware
function createRateLimitMiddleware(limiter) {
  return async (req, res, next) => {
    const key = req.user?.id || req.ip;
    const result = await limiter.isAllowed(key, 100, 60); // 100 req/min
    
    res.setHeader('X-RateLimit-Limit', 100);
    res.setHeader('X-RateLimit-Remaining', result.remaining);
    res.setHeader('X-RateLimit-Reset', result.resetAt);
    
    if (!result.allowed) {
      res.setHeader('Retry-After', result.retryAfter);
      return res.status(429).json({
        error: 'Too many requests',
        retryAfter: result.retryAfter
      });
    }
    
    next();
  };
}
```

---

### 7. How do you handle service discovery in microservices?

**Answer:**
Service discovery allows services to find and communicate with each other.

**Example:**

```javascript
// Client-side Service Discovery
class ServiceRegistry {
  constructor() {
    this.services = new Map();
  }
  
  register(serviceName, instance) {
    if (!this.services.has(serviceName)) {
      this.services.set(serviceName, []);
    }
    
    this.services.get(serviceName).push({
      ...instance,
      registeredAt: Date.now(),
      lastHealthCheck: Date.now()
    });
    
    console.log(`Registered ${serviceName}:`, instance);
  }
  
  deregister(serviceName, instanceId) {
    const instances = this.services.get(serviceName);
    if (instances) {
      this.services.set(
        serviceName,
        instances.filter(i => i.id !== instanceId)
      );
    }
  }
  
  discover(serviceName) {
    const instances = this.services.get(serviceName) || [];
    
    // Filter healthy instances
    const healthyInstances = instances.filter(i =>
      Date.now() - i.lastHealthCheck < 30000 // 30 seconds
    );
    
    if (healthyInstances.length === 0) {
      throw new Error(`No healthy instances of ${serviceName}`);
    }
    
    // Load balance - round robin
    return healthyInstances[Math.floor(Math.random() * healthyInstances.length)];
  }
  
  async healthCheck(serviceName, instanceId) {
    const instances = this.services.get(serviceName);
    const instance = instances?.find(i => i.id === instanceId);
    
    if (instance) {
      instance.lastHealthCheck = Date.now();
    }
  }
}

// Service Client with Discovery
class ServiceClient {
  constructor(serviceName, registry) {
    this.serviceName = serviceName;
    this.registry = registry;
  }
  
  async request(endpoint, options = {}) {
    const instance = this.registry.discover(this.serviceName);
    const url = `${instance.protocol}://${instance.host}:${instance.port}${endpoint}`;
    
    try {
      const response = await fetch(url, options);
      return await response.json();
    } catch (error) {
      console.error(`Request to ${this.serviceName} failed:`, error);
      throw error;
    }
  }
}

// AWS-based Service Discovery (using ECS)
class AWSServiceDiscovery {
  constructor() {
    this.ecs = new AWS.ECS();
    this.servicediscovery = new AWS.ServiceDiscovery();
  }
  
  async discoverService(serviceName) {
    const params = {
      NamespaceName: 'my-namespace',
      ServiceName: serviceName
    };
    
    const response = await this.servicediscovery.discoverInstances(params).promise();
    
    return response.Instances.map(instance => ({
      id: instance.InstanceId,
      host: instance.Attributes.AWS_INSTANCE_IPV4,
      port: instance.Attributes.AWS_INSTANCE_PORT,
      healthy: instance.HealthStatus === 'HEALTHY'
    }));
  }
}

// DNS-based Service Discovery
class DNSServiceDiscovery {
  constructor() {
    this.dns = require('dns').promises;
  }
  
  async discoverService(serviceName) {
    // Kubernetes DNS format: service-name.namespace.svc.cluster.local
    const dnsName = `${serviceName}.default.svc.cluster.local`;
    
    try {
      const addresses = await this.dns.resolve4(dnsName);
      return addresses.map(ip => ({
        host: ip,
        port: 8080
      }));
    } catch (error) {
      throw new Error(`Service ${serviceName} not found in DNS`);
    }
  }
}
```

---

## Key Points to Emphasize:
- Experience with distributed systems challenges
- Understanding of trade-offs (CAP theorem)
- Knowledge of consistency patterns
- Experience with cloud-native architectures
- Monitoring distributed systems
- Handling partial failures
- Performance optimization strategies