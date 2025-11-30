# Q31: Caching - In-Memory Caching with Redis

## 📋 Summary
In-memory caching dramatically improves application performance by storing frequently accessed data in RAM. This guide covers **Redis implementation**, caching strategies (read-through, write-through, cache-aside), cache eviction policies, and distributed caching patterns with comprehensive banking examples.

---

## 🎯 What You'll Learn
- **Redis Basics**: Setup, data structures, commands, TTL
- **Caching Patterns**: Cache-aside, Read-through, Write-through, Write-behind
- **Cache Invalidation**: TTL, manual invalidation, event-driven updates
- **Data Structures**: Strings, Hashes, Lists, Sets, Sorted Sets
- **Performance**: Reduce database load by 70-90%, sub-millisecond response times
- **Banking Examples**: Account balance caching, exchange rate caching, session management

---

## 📖 Comprehensive Explanation

### What is Caching?

**Caching** stores frequently accessed data in fast memory (RAM) to avoid repeated expensive operations (database queries, API calls, computations).

**Performance Impact**:
- **Without cache**: Every request hits database (~50-200ms)
- **With cache**: Most requests hit cache (~1-5ms)
- **Result**: 10-100x faster response times

```javascript
// Without cache (slow)
async function getAccountBalance(accountId) {
  const result = await db.query('SELECT balance FROM accounts WHERE id = ?', [accountId]);
  return result.balance; // 50-200ms per request
}

// With cache (fast)
async function getAccountBalanceCached(accountId) {
  // Try cache first (1-5ms)
  const cached = await redis.get(`balance:${accountId}`);
  if (cached) return JSON.parse(cached);
  
  // Cache miss - query database (50-200ms)
  const result = await db.query('SELECT balance FROM accounts WHERE id = ?', [accountId]);
  
  // Store in cache for next time
  await redis.setex(`balance:${accountId}`, 300, JSON.stringify(result.balance));
  
  return result.balance;
}
```

---

## 🗃️ Why Redis?

**Redis** (Remote Dictionary Server) is an in-memory data structure store used as:
- Database
- Cache
- Message broker
- Session store

### Redis Advantages

1. **Extremely Fast**: All data in RAM (sub-millisecond latency)
2. **Rich Data Structures**: Strings, Hashes, Lists, Sets, Sorted Sets, Bitmaps
3. **Atomic Operations**: Thread-safe operations (INCR, DECR, etc.)
4. **Persistence**: Optional disk backup (RDB snapshots, AOF logs)
5. **Replication**: Master-slave replication for high availability
6. **Pub/Sub**: Real-time messaging
7. **Expiration**: Automatic TTL (Time To Live) for keys

### Redis vs Other Caches

| Feature | Redis | Memcached | In-Process (Node) |
|---------|-------|-----------|-------------------|
| **Speed** | Very Fast | Very Fast | Fastest |
| **Data Structures** | Rich | Simple | Rich |
| **Persistence** | Yes | No | No |
| **Clustering** | Yes | Yes | No |
| **Memory Limit** | GBs-TBs | GBs | MBs |
| **Use Case** | Shared cache | Simple cache | Single instance |

---

## 🎨 Caching Patterns

### 1. **Cache-Aside (Lazy Loading)**

Application manages cache explicitly.

```
┌─────────┐
│   App   │
└────┬────┘
     │ 1. Read
     ├────────┐
     │        │
┌────▼───┐ ┌─▼──────┐
│ Cache  │ │   DB   │
│  (👍)  │ │  (👎)  │
└────────┘ └────────┘

Flow:
1. Check cache
2. If hit → return data
3. If miss → query DB → store in cache → return data
```

**Pros**: Simple, cache only what's needed  
**Cons**: Cache miss penalty (first request slow)

### 2. **Read-Through**

Cache layer sits between app and database.

```
┌─────────┐
│   App   │
└────┬────┘
     │ 1. Read
┌────▼───────┐
│   Cache    │
│  (Proxy)   │
└────┬───────┘
     │ 2. If miss
┌────▼───┐
│   DB   │
└────────┘
```

**Pros**: Transparent to application  
**Cons**: All reads go through cache layer

### 3. **Write-Through**

Writes go to cache and database synchronously.

```
┌─────────┐
│   App   │
└────┬────┘
     │ 1. Write
┌────▼───────┐
│   Cache    │◄─── Update
└────┬───────┘
     │ 2. Write
┌────▼───┐
│   DB   │◄─── Update
└────────┘
```

**Pros**: Cache always consistent with DB  
**Cons**: Write latency (wait for both)

### 4. **Write-Behind (Write-Back)**

Writes go to cache immediately, database updated asynchronously.

```
┌─────────┐
│   App   │
└────┬────┘
     │ 1. Write (fast)
┌────▼───────┐
│   Cache    │◄─── Update immediately
└────┬───────┘
     │ 2. Async batch write
┌────▼───┐
│   DB   │◄─── Update later
└────────┘
```

**Pros**: Fast writes, reduced DB load  
**Cons**: Risk of data loss (if cache fails before sync)

---

## ⏰ Cache Eviction Policies

When cache is full, which keys to remove?

| Policy | Description | Use Case |
|--------|-------------|----------|
| **LRU** | Least Recently Used | General purpose (Redis default) |
| **LFU** | Least Frequently Used | Detect popular items |
| **FIFO** | First In First Out | Simple, predictable |
| **TTL** | Time To Live expiration | Time-sensitive data |
| **Random** | Random eviction | Low overhead |

```javascript
// Redis configuration
maxmemory 2gb
maxmemory-policy allkeys-lru // Evict least recently used keys
```

---

## 🔑 Redis Data Structures

### 1. **Strings** (Simple Key-Value)

```javascript
// Set value
await redis.set('balance:ACC-001', '10000.50');

// Get value
const balance = await redis.get('balance:ACC-001');

// Set with expiration (300 seconds)
await redis.setex('balance:ACC-001', 300, '10000.50');

// Atomic increment
await redis.incr('counter');           // 1
await redis.incrby('counter', 10);     // 11
await redis.incrbyfloat('balance', 100.50); // Add to balance
```

### 2. **Hashes** (Object Storage)

```javascript
// Store account as hash
await redis.hset('account:ACC-001', {
  name: 'Alice',
  balance: '10000',
  currency: 'USD',
  status: 'active'
});

// Get specific field
const balance = await redis.hget('account:ACC-001', 'balance');

// Get all fields
const account = await redis.hgetall('account:ACC-001');

// Update multiple fields
await redis.hmset('account:ACC-001', {
  balance: '9500',
  lastUpdated: Date.now()
});
```

### 3. **Lists** (Ordered Collections)

```javascript
// Push to list (queue)
await redis.lpush('transactions:ACC-001', JSON.stringify({ amount: 100 }));
await redis.rpush('transactions:ACC-001', JSON.stringify({ amount: 200 }));

// Pop from list
const latest = await redis.lpop('transactions:ACC-001');

// Get range
const recent = await redis.lrange('transactions:ACC-001', 0, 9); // First 10
```

### 4. **Sets** (Unique Collections)

```javascript
// Add to set
await redis.sadd('active_accounts', 'ACC-001', 'ACC-002', 'ACC-003');

// Check membership
const isActive = await redis.sismember('active_accounts', 'ACC-001'); // true

// Get all members
const accounts = await redis.smembers('active_accounts');

// Set operations
await redis.sinter('us_accounts', 'premium_accounts'); // Intersection
await redis.sunion('us_accounts', 'eu_accounts');      // Union
```

### 5. **Sorted Sets** (Ranked Collections)

```javascript
// Add with score (timestamp, rank, etc.)
await redis.zadd('high_value_accounts', 10000, 'ACC-001');
await redis.zadd('high_value_accounts', 25000, 'ACC-002');
await redis.zadd('high_value_accounts', 15000, 'ACC-003');

// Get top accounts by balance
const top5 = await redis.zrevrange('high_value_accounts', 0, 4, 'WITHSCORES');

// Get rank
const rank = await redis.zrank('high_value_accounts', 'ACC-001');
```

---

## 📝 Example 1: Account Balance Caching (Cache-Aside Pattern)

### Complete Redis Caching Service

```javascript
const redis = require('redis');
const { Pool } = require('pg');

class BankingCacheService {
  constructor() {
    // Redis client
    this.redisClient = redis.createClient({
      host: process.env.REDIS_HOST || 'localhost',
      port: process.env.REDIS_PORT || 6379,
      password: process.env.REDIS_PASSWORD,
      db: 0,
    });

    this.redisClient.on('error', (err) => {
      console.error('❌ Redis error:', err);
    });

    this.redisClient.on('connect', () => {
      console.log('✅ Redis connected');
    });

    // PostgreSQL connection pool
    this.dbPool = new Pool({
      host: process.env.DB_HOST || 'localhost',
      port: process.env.DB_PORT || 5432,
      database: process.env.DB_NAME || 'banking_db',
      user: process.env.DB_USER || 'postgres',
      password: process.env.DB_PASSWORD || 'password',
      max: 10,
    });

    // Cache statistics
    this.stats = {
      hits: 0,
      misses: 0,
      writes: 0,
    };
  }

  /**
   * Get account balance with caching (Cache-Aside pattern)
   * 1. Check cache first
   * 2. If miss, query database
   * 3. Store in cache for 5 minutes
   */
  async getAccountBalance(accountId) {
    const cacheKey = `balance:${accountId}`;

    try {
      // Step 1: Try cache first
      const cached = await this.redisClient.get(cacheKey);
      
      if (cached) {
        this.stats.hits++;
        console.log(`✅ Cache HIT for ${accountId}`);
        return JSON.parse(cached);
      }

      // Step 2: Cache miss - query database
      this.stats.misses++;
      console.log(`⚠️ Cache MISS for ${accountId} - querying database...`);

      const result = await this.dbPool.query(
        'SELECT account_id, balance, currency, last_updated FROM accounts WHERE account_id = $1',
        [accountId]
      );

      if (result.rows.length === 0) {
        throw new Error(`Account ${accountId} not found`);
      }

      const accountData = {
        accountId: result.rows[0].account_id,
        balance: parseFloat(result.rows[0].balance),
        currency: result.rows[0].currency || 'USD',
        lastUpdated: result.rows[0].last_updated,
        cachedAt: new Date(),
      };

      // Step 3: Store in cache with 5-minute TTL
      await this.redisClient.setEx(
        cacheKey,
        300, // 5 minutes
        JSON.stringify(accountData)
      );

      console.log(`📝 Cached ${accountId} for 5 minutes`);

      return accountData;

    } catch (error) {
      console.error(`❌ Error getting balance for ${accountId}:`, error.message);
      throw error;
    }
  }

  /**
   * Update account balance (Write-Through pattern)
   * Updates both cache and database
   */
  async updateAccountBalance(accountId, newBalance) {
    const cacheKey = `balance:${accountId}`;

    try {
      // Step 1: Update database first
      const result = await this.dbPool.query(
        `UPDATE accounts 
         SET balance = $1, last_updated = NOW() 
         WHERE account_id = $2 
         RETURNING account_id, balance, currency, last_updated`,
        [newBalance, accountId]
      );

      if (result.rows.length === 0) {
        throw new Error(`Account ${accountId} not found`);
      }

      const accountData = {
        accountId: result.rows[0].account_id,
        balance: parseFloat(result.rows[0].balance),
        currency: result.rows[0].currency || 'USD',
        lastUpdated: result.rows[0].last_updated,
        cachedAt: new Date(),
      };

      // Step 2: Update cache
      await this.redisClient.setEx(
        cacheKey,
        300, // 5 minutes
        JSON.stringify(accountData)
      );

      this.stats.writes++;
      console.log(`✅ Updated balance for ${accountId} in DB and cache`);

      return accountData;

    } catch (error) {
      console.error(`❌ Error updating balance for ${accountId}:`, error.message);
      throw error;
    }
  }

  /**
   * Invalidate cache (force refresh on next read)
   */
  async invalidateAccountCache(accountId) {
    const cacheKey = `balance:${accountId}`;
    
    try {
      const deleted = await this.redisClient.del(cacheKey);
      
      if (deleted) {
        console.log(`🗑️ Invalidated cache for ${accountId}`);
      } else {
        console.log(`⚠️ No cache entry for ${accountId}`);
      }

      return deleted > 0;

    } catch (error) {
      console.error(`❌ Error invalidating cache for ${accountId}:`, error.message);
      throw error;
    }
  }

  /**
   * Get multiple account balances efficiently (pipeline)
   */
  async getMultipleBalances(accountIds) {
    const pipeline = this.redisClient.multi();
    
    // Queue all GET commands
    accountIds.forEach(accountId => {
      pipeline.get(`balance:${accountId}`);
    });

    // Execute all at once
    const results = await pipeline.exec();
    
    const balances = [];
    const missingAccounts = [];

    // Process results
    results.forEach((result, index) => {
      if (result) {
        this.stats.hits++;
        balances.push(JSON.parse(result));
      } else {
        this.stats.misses++;
        missingAccounts.push(accountIds[index]);
      }
    });

    // Fetch missing accounts from database
    if (missingAccounts.length > 0) {
      console.log(`⚠️ Cache miss for ${missingAccounts.length} accounts, querying database...`);

      const dbResult = await this.dbPool.query(
        'SELECT account_id, balance, currency, last_updated FROM accounts WHERE account_id = ANY($1)',
        [missingAccounts]
      );

      // Cache the fetched accounts
      const cachePipeline = this.redisClient.multi();
      
      dbResult.rows.forEach(row => {
        const accountData = {
          accountId: row.account_id,
          balance: parseFloat(row.balance),
          currency: row.currency || 'USD',
          lastUpdated: row.last_updated,
          cachedAt: new Date(),
        };

        balances.push(accountData);
        
        cachePipeline.setEx(
          `balance:${row.account_id}`,
          300,
          JSON.stringify(accountData)
        );
      });

      await cachePipeline.exec();
    }

    return balances;
  }

  /**
   * Get cache statistics
   */
  getStats() {
    const total = this.stats.hits + this.stats.misses;
    const hitRate = total > 0 ? ((this.stats.hits / total) * 100).toFixed(2) : 0;

    return {
      hits: this.stats.hits,
      misses: this.stats.misses,
      writes: this.stats.writes,
      total,
      hitRate: `${hitRate}%`,
    };
  }

  /**
   * Warm up cache (preload frequently accessed data)
   */
  async warmUpCache(accountIds) {
    console.log(`🔥 Warming up cache for ${accountIds.length} accounts...`);

    const result = await this.dbPool.query(
      'SELECT account_id, balance, currency, last_updated FROM accounts WHERE account_id = ANY($1)',
      [accountIds]
    );

    const pipeline = this.redisClient.multi();

    result.rows.forEach(row => {
      const accountData = {
        accountId: row.account_id,
        balance: parseFloat(row.balance),
        currency: row.currency || 'USD',
        lastUpdated: row.last_updated,
        cachedAt: new Date(),
      };

      pipeline.setEx(
        `balance:${row.account_id}`,
        300,
        JSON.stringify(accountData)
      );
    });

    await pipeline.exec();
    
    console.log(`✅ Warmed up cache for ${result.rows.length} accounts`);
  }

  /**
   * Close connections
   */
  async close() {
    await this.redisClient.quit();
    await this.dbPool.end();
    console.log('✅ Connections closed');
  }
}

// Usage Example
async function runExample1() {
  console.log('=== Example 1: Account Balance Caching ===\n');

  const cacheService = new BankingCacheService();

  try {
    // Test 1: First access (cache miss)
    console.log('Test 1: First access (should be cache MISS)');
    const balance1 = await cacheService.getAccountBalance('ACC-001');
    console.log('Balance:', balance1);
    console.log('Stats:', cacheService.getStats(), '\n');

    // Test 2: Second access (cache hit)
    console.log('Test 2: Second access (should be cache HIT)');
    const balance2 = await cacheService.getAccountBalance('ACC-001');
    console.log('Balance:', balance2);
    console.log('Stats:', cacheService.getStats(), '\n');

    // Test 3: Update balance (invalidates and updates cache)
    console.log('Test 3: Update balance');
    const updated = await cacheService.updateAccountBalance('ACC-001', 11000);
    console.log('Updated:', updated);
    console.log('Stats:', cacheService.getStats(), '\n');

    // Test 4: Access after update (should use fresh cache)
    console.log('Test 4: Access after update (cache HIT with new value)');
    const balance3 = await cacheService.getAccountBalance('ACC-001');
    console.log('Balance:', balance3);
    console.log('Stats:', cacheService.getStats(), '\n');

    // Test 5: Manual cache invalidation
    console.log('Test 5: Manual cache invalidation');
    await cacheService.invalidateAccountCache('ACC-001');
    const balance4 = await cacheService.getAccountBalance('ACC-001');
    console.log('Balance after invalidation:', balance4);
    console.log('Stats:', cacheService.getStats(), '\n');

    // Test 6: Batch retrieval
    console.log('Test 6: Get multiple balances');
    const balances = await cacheService.getMultipleBalances([
      'ACC-001', 'ACC-002', 'ACC-003', 'ACC-004'
    ]);
    console.log(`Retrieved ${balances.length} balances`);
    balances.forEach(b => console.log(`  ${b.accountId}: $${b.balance}`));
    console.log('Stats:', cacheService.getStats(), '\n');

    // Test 7: Cache warm-up
    console.log('Test 7: Cache warm-up');
    await cacheService.warmUpCache(['ACC-001', 'ACC-002', 'ACC-003']);
    console.log('Stats:', cacheService.getStats(), '\n');

    // Final statistics
    console.log('=== Final Cache Statistics ===');
    const finalStats = cacheService.getStats();
    console.log(`Total requests: ${finalStats.total}`);
    console.log(`Cache hits: ${finalStats.hits}`);
    console.log(`Cache misses: ${finalStats.misses}`);
    console.log(`Hit rate: ${finalStats.hitRate}`);
    console.log(`Writes: ${finalStats.writes}`);

  } finally {
    await cacheService.close();
  }
}

// Run if executed directly
if (require.main === module) {
  runExample1().catch(console.error);
}

module.exports = { BankingCacheService };
```

### Setup Instructions

```bash
# Install dependencies
npm install redis pg

# Start Redis server (Docker)
docker run -d --name redis -p 6379:6379 redis:latest

# Or install locally
# macOS: brew install redis && redis-server
# Ubuntu: sudo apt install redis-server && redis-server
# Windows: Use WSL or Docker

# Verify Redis is running
redis-cli ping  # Should return "PONG"

# Set environment variables
export REDIS_HOST=localhost
export REDIS_PORT=6379
export DB_HOST=localhost
export DB_PORT=5432
export DB_NAME=banking_db
export DB_USER=postgres
export DB_PASSWORD=password

# Run example
node Q31_example1_balance_caching.js
```

### Expected Output

```
=== Example 1: Account Balance Caching ===

✅ Redis connected

Test 1: First access (should be cache MISS)
⚠️ Cache MISS for ACC-001 - querying database...
📝 Cached ACC-001 for 5 minutes
Balance: {
  accountId: 'ACC-001',
  balance: 10000,
  currency: 'USD',
  lastUpdated: 2025-11-15T10:30:00.000Z,
  cachedAt: 2025-11-15T10:35:00.000Z
}
Stats: { hits: 0, misses: 1, writes: 0, total: 1, hitRate: '0.00%' }

Test 2: Second access (should be cache HIT)
✅ Cache HIT for ACC-001
Balance: {
  accountId: 'ACC-001',
  balance: 10000,
  currency: 'USD',
  lastUpdated: 2025-11-15T10:30:00.000Z,
  cachedAt: 2025-11-15T10:35:00.000Z
}
Stats: { hits: 1, misses: 1, writes: 0, total: 2, hitRate: '50.00%' }

Test 3: Update balance
✅ Updated balance for ACC-001 in DB and cache
Updated: {
  accountId: 'ACC-001',
  balance: 11000,
  currency: 'USD',
  lastUpdated: 2025-11-15T10:35:05.000Z,
  cachedAt: 2025-11-15T10:35:05.000Z
}
Stats: { hits: 1, misses: 1, writes: 1, total: 2, hitRate: '50.00%' }

=== Final Cache Statistics ===
Total requests: 8
Cache hits: 5
Cache misses: 3
Hit rate: 62.50%
Writes: 1

✅ Connections closed
```

---

## 📝 Example 2: Exchange Rate Caching with Pub/Sub Updates

### Real-time Exchange Rate System

```javascript
const redis = require('redis');
const axios = require('axios');

class ExchangeRateCacheService {
  constructor() {
    // Redis client for caching
    this.cacheClient = redis.createClient({
      host: 'localhost',
      port: 6379,
    });

    // Redis client for pub/sub (separate connection required)
    this.pubSubClient = redis.createClient({
      host: 'localhost',
      port: 6379,
    });

    // Publisher client
    this.publisher = redis.createClient({
      host: 'localhost',
      port: 6379,
    });

    this.cacheClient.on('connect', () => console.log('✅ Cache client connected'));
    this.pubSubClient.on('connect', () => console.log('✅ Pub/Sub client connected'));
    this.publisher.on('connect', () => console.log('✅ Publisher connected'));

    // Subscribe to rate updates
    this.setupSubscriptions();
  }

  /**
   * Setup pub/sub subscriptions for real-time updates
   */
  setupSubscriptions() {
    this.pubSubClient.subscribe('exchange_rates:update', (message) => {
      console.log('📡 Received rate update:', message);
      
      const { from, to, rate } = JSON.parse(message);
      console.log(`  ${from}/${to} = ${rate}`);
    });

    this.pubSubClient.subscribe('exchange_rates:invalidate', async (message) => {
      console.log('🗑️ Invalidating cache:', message);
      
      const { from, to } = JSON.parse(message);
      await this.cacheClient.del(`rate:${from}:${to}`);
    });
  }

  /**
   * Get exchange rate with caching
   * Example: getExchangeRate('USD', 'EUR') → 0.85
   */
  async getExchangeRate(fromCurrency, toCurrency) {
    const cacheKey = `rate:${fromCurrency}:${toCurrency}`;

    try {
      // Check cache
      const cached = await this.cacheClient.get(cacheKey);
      
      if (cached) {
        console.log(`✅ Cache HIT for ${fromCurrency}/${toCurrency}`);
        return JSON.parse(cached);
      }

      // Cache miss - fetch from external API (simulated)
      console.log(`⚠️ Cache MISS for ${fromCurrency}/${toCurrency} - fetching...`);
      
      const rate = await this.fetchExchangeRate(fromCurrency, toCurrency);

      const rateData = {
        from: fromCurrency,
        to: toCurrency,
        rate,
        timestamp: new Date(),
      };

      // Cache for 1 hour
      await this.cacheClient.setEx(
        cacheKey,
        3600,
        JSON.stringify(rateData)
      );

      console.log(`📝 Cached ${fromCurrency}/${toCurrency} for 1 hour`);

      return rateData;

    } catch (error) {
      console.error(`❌ Error getting rate for ${fromCurrency}/${toCurrency}:`, error.message);
      throw error;
    }
  }

  /**
   * Simulate fetching rate from external API
   * In production, call actual API (e.g., exchangerate-api.com)
   */
  async fetchExchangeRate(fromCurrency, toCurrency) {
    // Simulate API delay
    await new Promise(resolve => setTimeout(resolve, 500));

    // Mock rates
    const rates = {
      'USD:EUR': 0.85,
      'USD:GBP': 0.73,
      'USD:JPY': 110.50,
      'EUR:USD': 1.18,
      'EUR:GBP': 0.86,
      'GBP:USD': 1.37,
    };

    const key = `${fromCurrency}:${toCurrency}`;
    return rates[key] || 1.0;
  }

  /**
   * Update exchange rate and notify subscribers
   * (Admin function - called when rates change)
   */
  async updateExchangeRate(fromCurrency, toCurrency, newRate) {
    const cacheKey = `rate:${fromCurrency}:${toCurrency}`;

    try {
      const rateData = {
        from: fromCurrency,
        to: toCurrency,
        rate: newRate,
        timestamp: new Date(),
      };

      // Update cache
      await this.cacheClient.setEx(
        cacheKey,
        3600,
        JSON.stringify(rateData)
      );

      console.log(`✅ Updated rate ${fromCurrency}/${toCurrency} = ${newRate}`);

      // Publish update event
      await this.publisher.publish(
        'exchange_rates:update',
        JSON.stringify(rateData)
      );

      return rateData;

    } catch (error) {
      console.error(`❌ Error updating rate:`, error.message);
      throw error;
    }
  }

  /**
   * Convert amount between currencies
   */
  async convertCurrency(amount, fromCurrency, toCurrency) {
    const { rate } = await this.getExchangeRate(fromCurrency, toCurrency);
    
    const converted = amount * rate;

    return {
      from: { amount, currency: fromCurrency },
      to: { amount: converted, currency: toCurrency },
      rate,
      timestamp: new Date(),
    };
  }

  /**
   * Get all cached exchange rates
   */
  async getAllCachedRates() {
    const keys = await this.cacheClient.keys('rate:*');
    
    const pipeline = this.cacheClient.multi();
    keys.forEach(key => pipeline.get(key));
    
    const results = await pipeline.exec();
    
    return results.map(result => JSON.parse(result));
  }

  /**
   * Close connections
   */
  async close() {
    await this.cacheClient.quit();
    await this.pubSubClient.quit();
    await this.publisher.quit();
    console.log('✅ Connections closed');
  }
}

// Usage Example
async function runExample2() {
  console.log('=== Example 2: Exchange Rate Caching with Pub/Sub ===\n');

  const rateService = new ExchangeRateCacheService();

  // Wait for connections
  await new Promise(resolve => setTimeout(resolve, 1000));

  try {
    // Test 1: Get exchange rate (cache miss)
    console.log('\nTest 1: Get USD → EUR rate (cache miss)');
    const rate1 = await rateService.getExchangeRate('USD', 'EUR');
    console.log('Rate:', rate1);

    // Test 2: Get same rate (cache hit)
    console.log('\nTest 2: Get USD → EUR rate again (cache hit)');
    const rate2 = await rateService.getExchangeRate('USD', 'EUR');
    console.log('Rate:', rate2);

    // Test 3: Convert currency
    console.log('\nTest 3: Convert $1000 USD to EUR');
    const conversion = await rateService.convertCurrency(1000, 'USD', 'EUR');
    console.log('Conversion:', conversion);
    console.log(`  $${conversion.from.amount} ${conversion.from.currency} = €${conversion.to.amount.toFixed(2)} ${conversion.to.currency}`);

    // Test 4: Update rate (triggers pub/sub notification)
    console.log('\nTest 4: Update USD → EUR rate to 0.90');
    await rateService.updateExchangeRate('USD', 'EUR', 0.90);

    // Wait for pub/sub message
    await new Promise(resolve => setTimeout(resolve, 500));

    // Test 5: Get updated rate
    console.log('\nTest 5: Get updated USD → EUR rate');
    const rate3 = await rateService.getExchangeRate('USD', 'EUR');
    console.log('Updated rate:', rate3);

    // Test 6: Multiple currencies
    console.log('\nTest 6: Get multiple exchange rates');
    await rateService.getExchangeRate('USD', 'GBP');
    await rateService.getExchangeRate('USD', 'JPY');
    await rateService.getExchangeRate('EUR', 'GBP');

    // Test 7: View all cached rates
    console.log('\nTest 7: All cached rates');
    const allRates = await rateService.getAllCachedRates();
    console.log(`Total cached rates: ${allRates.length}`);
    allRates.forEach(r => {
      console.log(`  ${r.from}/${r.to} = ${r.rate}`);
    });

  } finally {
    await rateService.close();
  }
}

// Run if executed directly
if (require.main === module) {
  runExample2().catch(console.error);
}

module.exports = { ExchangeRateCacheService };
```

### Running Example 2

```bash
# Install additional dependency
npm install axios

# Run example
node Q31_example2_exchange_rate_caching.js
```

### Expected Output

```
=== Example 2: Exchange Rate Caching with Pub/Sub ===

✅ Cache client connected
✅ Pub/Sub client connected
✅ Publisher connected

Test 1: Get USD → EUR rate (cache miss)
⚠️ Cache MISS for USD/EUR - fetching...
📝 Cached USD/EUR for 1 hour
Rate: { from: 'USD', to: 'EUR', rate: 0.85, timestamp: 2025-11-15T10:30:00.000Z }

Test 2: Get USD → EUR rate again (cache hit)
✅ Cache HIT for USD/EUR
Rate: { from: 'USD', to: 'EUR', rate: 0.85, timestamp: 2025-11-15T10:30:00.000Z }

Test 3: Convert $1000 USD to EUR
✅ Cache HIT for USD/EUR
Conversion: {
  from: { amount: 1000, currency: 'USD' },
  to: { amount: 850, currency: 'EUR' },
  rate: 0.85,
  timestamp: 2025-11-15T10:30:05.000Z
}
  $1000 USD = €850.00 EUR

Test 4: Update USD → EUR rate to 0.90
✅ Updated rate USD/EUR = 0.9
📡 Received rate update: {"from":"USD","to":"EUR","rate":0.9}
  USD/EUR = 0.9

Test 5: Get updated USD → EUR rate
✅ Cache HIT for USD/EUR
Updated rate: { from: 'USD', to: 'EUR', rate: 0.9, timestamp: 2025-11-15T10:30:10.000Z }

Test 6: Get multiple exchange rates
⚠️ Cache MISS for USD/GBP - fetching...
⚠️ Cache MISS for USD/JPY - fetching...
⚠️ Cache MISS for EUR/GBP - fetching...

Test 7: All cached rates
Total cached rates: 4
  USD/EUR = 0.9
  USD/GBP = 0.73
  USD/JPY = 110.5
  EUR/GBP = 0.86

✅ Connections closed
```

---

## 🎯 Key Takeaways

### When to Use Caching

✅ **Good candidates**:
- Frequently accessed data (account balances, user profiles)
- Expensive computations (interest calculations, reports)
- External API results (exchange rates, stock prices)
- Session data (user tokens, shopping carts)
- Configuration data (feature flags, settings)

❌ **Bad candidates**:
- Rarely accessed data (archive records)
- Rapidly changing data (stock tickers updating every second)
- Data that must be 100% consistent (legal requirements)

### Best Practices

1. **Set appropriate TTL** (Time To Live)
   ```javascript
   // Short TTL for dynamic data
   redis.setEx('stock:AAPL', 10, price); // 10 seconds
   
   // Long TTL for static data
   redis.setEx('config:fees', 86400, fees); // 24 hours
   ```

2. **Handle cache failures gracefully**
   ```javascript
   try {
     const data = await redis.get(key);
     return data || await fetchFromDB();
   } catch (error) {
     // If Redis fails, fall back to database
     return await fetchFromDB();
   }
   ```

3. **Use cache warming** for critical data
   ```javascript
   // Preload cache on startup
   await warmUpCache(['top-100-accounts']);
   ```

4. **Monitor cache performance**
   ```javascript
   const hitRate = (hits / (hits + misses)) * 100;
   // Target: 80%+ hit rate for effective caching
   ```

5. **Invalidate strategically**
   ```javascript
   // Invalidate related keys
   await redis.del(`balance:${accountId}`);
   await redis.del(`transactions:${accountId}`);
   await redis.del(`summary:${accountId}`);
   ```

### Common Pitfalls

- ❌ **Cache stampede**: Multiple requests fetch same data simultaneously
  - Solution: Use locking or probabilistic early expiration
  
- ❌ **Stale data**: Cache not updated after database changes
  - Solution: Use write-through or event-driven invalidation
  
- ❌ **Memory overflow**: Cache grows too large
  - Solution: Set maxmemory and eviction policy

- ❌ **Cold start**: Empty cache after restart
  - Solution: Implement cache warming strategy

### Performance Impact

**Typical improvements with caching**:
- Response time: 50-200ms → 1-5ms (10-100x faster)
- Database load: 90% reduction
- Throughput: 3-10x more requests/second
- Cost: Lower database server requirements

---

**File**: `Q31_Caching_In_Memory.md`  
**Status**: ✅ Complete with 2 comprehensive examples
