# Q32: Caching - Cache Invalidation Strategies

## 📋 Summary
Cache invalidation is one of the hardest problems in computer science. This guide covers **cache invalidation patterns**, TTL strategies, eviction policies, consistency models, and event-driven invalidation with comprehensive banking examples demonstrating how to keep cached data fresh and accurate.

---

## 🎯 What You'll Learn
- **Cache Invalidation Patterns**: Time-based (TTL), Event-driven, Manual, Write-through
- **Eviction Policies**: LRU, LFU, FIFO, TTL-based
- **Consistency Models**: Strong consistency, Eventual consistency, Cache-aside
- **Advanced Strategies**: Tag-based invalidation, Dependency tracking, Probabilistic early expiration
- **Monitoring**: Stale data detection, invalidation metrics
- **Banking Examples**: Real-time balance updates, transaction cache invalidation, session management

---

## 📖 Comprehensive Explanation

### The Cache Invalidation Problem

> "There are only two hard things in Computer Science: cache invalidation and naming things."  
> — Phil Karlton

**The Challenge**: How do you ensure cached data stays synchronized with the source of truth (database)?

```javascript
// The Problem:
// 1. Cache stores balance = $1000
await redis.set('balance:ACC-001', 1000);

// 2. Someone updates database directly
await db.query('UPDATE accounts SET balance = 1500 WHERE id = ?', ['ACC-001']);

// 3. Cache is now STALE (still shows $1000, but DB has $1500)
const cached = await redis.get('balance:ACC-001'); // Returns 1000 ❌ Wrong!

// 4. How do we fix this?
```

---

## 🔄 Cache Invalidation Patterns

### 1. **Time-Based Invalidation (TTL)**

**Strategy**: Automatically expire cache entries after a fixed time period.

```javascript
// Set with 5-minute TTL
await redis.setEx('balance:ACC-001', 300, '1000');

// After 5 minutes, key automatically deleted
// Next read will fetch fresh data from database
```

**Pros**:
- Simple to implement
- Prevents stale data from lasting forever
- Automatic cleanup

**Cons**:
- Data can be stale for entire TTL period
- May cause cache stampede when many keys expire simultaneously

**Best for**: Data that changes infrequently, acceptable staleness window

**TTL Selection Guide**:
```javascript
// Ultra-fast changing data (stock prices)
redis.setEx('stock:AAPL', 1, price); // 1 second

// Moderate changes (account balances)
redis.setEx('balance:ACC-001', 300, balance); // 5 minutes

// Slow changes (user profiles)
redis.setEx('profile:USER-001', 3600, profile); // 1 hour

// Rarely changes (configuration)
redis.setEx('config:fees', 86400, config); // 24 hours
```

### 2. **Write-Through Invalidation**

**Strategy**: Update cache every time database is updated.

```javascript
async function updateBalance(accountId, newBalance) {
  // 1. Update database
  await db.query('UPDATE accounts SET balance = ? WHERE id = ?', [newBalance, accountId]);
  
  // 2. Update cache immediately
  await redis.setEx(`balance:${accountId}`, 300, newBalance);
  
  // Cache and DB always in sync!
}
```

**Pros**:
- Cache always consistent with DB
- No stale data

**Cons**:
- Write latency (must update both DB and cache)
- Complexity in handling failures

**Best for**: Critical data that must be consistent (account balances, inventory)

### 3. **Write-Behind (Lazy Invalidation)**

**Strategy**: Delete cache entry, let next read refresh it.

```javascript
async function updateBalance(accountId, newBalance) {
  // 1. Update database
  await db.query('UPDATE accounts SET balance = ? WHERE id = ?', [newBalance, accountId]);
  
  // 2. Delete cache (don't update it)
  await redis.del(`balance:${accountId}`);
  
  // Next read will fetch fresh data and re-cache
}
```

**Pros**:
- Simpler than write-through
- Faster writes (no need to update cache)

**Cons**:
- Next read will be slow (cache miss)

**Best for**: Infrequently read data, write-heavy workloads

### 4. **Event-Driven Invalidation**

**Strategy**: Use pub/sub or message queues to notify cache invalidation.

```javascript
// Publisher (when balance changes)
await redis.publish('invalidate', JSON.stringify({
  type: 'balance',
  accountId: 'ACC-001'
}));

// Subscriber (cache service listening)
redis.subscribe('invalidate', async (message) => {
  const { type, accountId } = JSON.parse(message);
  await redis.del(`${type}:${accountId}`);
});
```

**Pros**:
- Decoupled architecture
- Real-time invalidation across multiple cache servers
- Scales well

**Cons**:
- More complex
- Requires message broker

**Best for**: Distributed systems, microservices

### 5. **Tag-Based Invalidation**

**Strategy**: Group related cache keys with tags, invalidate entire groups.

```javascript
// Cache with tags
await redis.set('balance:ACC-001', '1000');
await redis.sAdd('tag:account:ACC-001', 'balance:ACC-001');
await redis.sAdd('tag:account:ACC-001', 'transactions:ACC-001');
await redis.sAdd('tag:account:ACC-001', 'summary:ACC-001');

// Invalidate all data related to account
async function invalidateAccount(accountId) {
  const keys = await redis.sMembers(`tag:account:${accountId}`);
  await redis.del(...keys); // Delete all related keys
  await redis.del(`tag:account:${accountId}`); // Delete tag itself
}
```

**Pros**:
- Invalidate related data together
- Prevent partial stale state

**Cons**:
- Extra storage for tags
- More complex to maintain

**Best for**: Complex relationships, multi-level caching

---

## ⏰ TTL Strategies

### Fixed TTL

```javascript
// Same TTL for all entries
await redis.setEx('balance:ACC-001', 300, balance1);
await redis.setEx('balance:ACC-002', 300, balance2);
```

### Variable TTL (Based on Data Characteristics)

```javascript
function calculateTTL(accountType) {
  switch (accountType) {
    case 'premium': return 60;    // 1 min (more accurate)
    case 'standard': return 300;  // 5 min
    case 'basic': return 600;     // 10 min (less accurate OK)
    default: return 300;
  }
}

await redis.setEx('balance:ACC-001', calculateTTL('premium'), balance);
```

### Probabilistic Early Expiration

**Problem**: Cache stampede when many clients try to refresh expired cache simultaneously.

**Solution**: Randomly expire some entries early to spread load.

```javascript
async function getWithProbabilisticExpiry(key, ttl) {
  const cached = await redis.get(key);
  
  if (!cached) {
    // Cache miss - fetch and cache
    const fresh = await fetchFromDB(key);
    await redis.setEx(key, ttl, fresh);
    return fresh;
  }
  
  // Cache hit - but maybe expire early?
  const remainingTTL = await redis.ttl(key);
  const delta = ttl - remainingTTL;
  
  // Probability increases as expiry approaches
  // When TTL = 0 (expired), probability = 1.0
  // When TTL = full, probability = 0.0
  const probability = delta / ttl;
  
  if (Math.random() < probability) {
    console.log('Probabilistic early expiration triggered');
    
    // Refresh in background (don't wait)
    fetchFromDB(key).then(fresh => {
      redis.setEx(key, ttl, fresh);
    });
  }
  
  return cached; // Return cached value (may be slightly stale)
}
```

### Sliding Window TTL

**Strategy**: Reset TTL on every access (keep popular data longer).

```javascript
async function getWithSlidingTTL(key, ttl) {
  const value = await redis.get(key);
  
  if (value) {
    // Reset TTL on access
    await redis.expire(key, ttl);
    return value;
  }
  
  // Fetch and cache
  const fresh = await fetchFromDB(key);
  await redis.setEx(key, ttl, fresh);
  return fresh;
}
```

---

## 🗑️ Eviction Policies

When Redis memory is full, which keys to remove?

### LRU (Least Recently Used) - Default

```javascript
// Redis config
maxmemory 2gb
maxmemory-policy allkeys-lru

// Evicts keys that haven't been accessed in longest time
// Perfect for general caching
```

### LFU (Least Frequently Used)

```javascript
maxmemory-policy allkeys-lfu

// Evicts keys accessed least often
// Better for identifying "hot" data
```

### Volatile TTL

```javascript
maxmemory-policy volatile-ttl

// Only evict keys with TTL set
// Evict keys closest to expiring first
```

### No Eviction

```javascript
maxmemory-policy noeviction

// Never evict - return errors when full
// Use for critical data that must stay cached
```

### Comparison Table

| Policy | Evicts | Best For |
|--------|--------|----------|
| `allkeys-lru` | Least recently used (any key) | General purpose |
| `allkeys-lfu` | Least frequently used (any key) | Popular content |
| `volatile-lru` | Least recently used (with TTL) | Mixed persistent + cache |
| `volatile-lfu` | Least frequently used (with TTL) | Hot data detection |
| `volatile-ttl` | Keys expiring soonest | Time-sensitive data |
| `noeviction` | None (return error) | Critical data |

---

## 📝 Example 1: Multi-Strategy Cache Invalidation System

### Complete Banking Cache Manager

```javascript
const redis = require('redis');
const { Pool } = require('pg');
const EventEmitter = require('events');

class BankingCacheManager extends EventEmitter {
  constructor() {
    super();
    
    // Redis clients
    this.cacheClient = redis.createClient({ host: 'localhost', port: 6379 });
    this.pubClient = redis.createClient({ host: 'localhost', port: 6379 });
    this.subClient = redis.createClient({ host: 'localhost', port: 6379 });

    // Database
    this.dbPool = new Pool({
      host: 'localhost',
      database: 'banking_db',
      user: 'postgres',
      password: 'password',
    });

    // Invalidation statistics
    this.stats = {
      ttlExpired: 0,
      manualInvalidations: 0,
      eventDrivenInvalidations: 0,
      tagInvalidations: 0,
    };

    // Setup event-driven invalidation
    this.setupEventListeners();

    console.log('✅ Banking Cache Manager initialized');
  }

  /**
   * Setup pub/sub for event-driven invalidation
   */
  setupEventListeners() {
    // Subscribe to invalidation events
    this.subClient.subscribe('cache:invalidate', async (message) => {
      const event = JSON.parse(message);
      console.log('📡 Received invalidation event:', event);

      switch (event.type) {
        case 'balance':
          await this.invalidateBalance(event.accountId);
          this.stats.eventDrivenInvalidations++;
          break;
        
        case 'account':
          await this.invalidateAccountTag(event.accountId);
          this.stats.tagInvalidations++;
          break;
        
        case 'transaction':
          await this.invalidateTransactions(event.accountId);
          this.stats.eventDrivenInvalidations++;
          break;
      }
    });

    console.log('📡 Subscribed to invalidation events');
  }

  /**
   * Strategy 1: TTL-Based Caching
   * Get balance with automatic expiration
   */
  async getBalance(accountId, ttlSeconds = 300) {
    const cacheKey = `balance:${accountId}`;

    try {
      // Check cache
      const cached = await this.cacheClient.get(cacheKey);
      
      if (cached) {
        console.log(`✅ Cache HIT: ${cacheKey}`);
        
        // Check remaining TTL
        const ttl = await this.cacheClient.ttl(cacheKey);
        console.log(`  TTL remaining: ${ttl}s`);
        
        return JSON.parse(cached);
      }

      // Cache miss
      console.log(`⚠️ Cache MISS: ${cacheKey}`);
      
      const result = await this.dbPool.query(
        'SELECT balance, currency, updated_at FROM accounts WHERE account_id = $1',
        [accountId]
      );

      if (result.rows.length === 0) {
        throw new Error(`Account ${accountId} not found`);
      }

      const balanceData = {
        accountId,
        balance: parseFloat(result.rows[0].balance),
        currency: result.rows[0].currency,
        updatedAt: result.rows[0].updated_at,
        cachedAt: new Date(),
      };

      // Cache with TTL
      await this.cacheClient.setEx(cacheKey, ttlSeconds, JSON.stringify(balanceData));
      console.log(`📝 Cached with ${ttlSeconds}s TTL`);

      // Tag for group invalidation
      await this.cacheClient.sAdd(`tag:account:${accountId}`, cacheKey);

      return balanceData;

    } catch (error) {
      console.error('❌ Error:', error.message);
      throw error;
    }
  }

  /**
   * Strategy 2: Write-Through Invalidation
   * Update balance and immediately update cache
   */
  async updateBalanceWriteThrough(accountId, newBalance) {
    const cacheKey = `balance:${accountId}`;

    try {
      // Update database
      const result = await this.dbPool.query(
        `UPDATE accounts 
         SET balance = $1, updated_at = NOW() 
         WHERE account_id = $2 
         RETURNING balance, currency, updated_at`,
        [newBalance, accountId]
      );

      if (result.rows.length === 0) {
        throw new Error(`Account ${accountId} not found`);
      }

      const balanceData = {
        accountId,
        balance: parseFloat(result.rows[0].balance),
        currency: result.rows[0].currency,
        updatedAt: result.rows[0].updated_at,
        cachedAt: new Date(),
      };

      // Update cache immediately (write-through)
      await this.cacheClient.setEx(cacheKey, 300, JSON.stringify(balanceData));
      console.log(`✅ Write-through: Updated DB and cache for ${accountId}`);

      // Publish event for other cache instances
      await this.pubClient.publish('cache:invalidate', JSON.stringify({
        type: 'balance',
        accountId,
        timestamp: new Date(),
      }));

      return balanceData;

    } catch (error) {
      console.error('❌ Error:', error.message);
      throw error;
    }
  }

  /**
   * Strategy 3: Write-Behind Invalidation
   * Update database and delete cache (lazy refresh)
   */
  async updateBalanceWriteBehind(accountId, newBalance) {
    const cacheKey = `balance:${accountId}`;

    try {
      // Update database
      await this.dbPool.query(
        'UPDATE accounts SET balance = $1, updated_at = NOW() WHERE account_id = $2',
        [newBalance, accountId]
      );

      // Delete cache (lazy invalidation)
      const deleted = await this.cacheClient.del(cacheKey);
      
      if (deleted) {
        console.log(`✅ Write-behind: Updated DB, deleted cache for ${accountId}`);
        this.stats.manualInvalidations++;
      }

      // Publish event
      await this.pubClient.publish('cache:invalidate', JSON.stringify({
        type: 'balance',
        accountId,
        timestamp: new Date(),
      }));

      return { success: true };

    } catch (error) {
      console.error('❌ Error:', error.message);
      throw error;
    }
  }

  /**
   * Strategy 4: Manual Invalidation
   * Explicitly remove cache entry
   */
  async invalidateBalance(accountId) {
    const cacheKey = `balance:${accountId}`;
    
    try {
      const deleted = await this.cacheClient.del(cacheKey);
      
      if (deleted) {
        console.log(`🗑️ Manually invalidated ${cacheKey}`);
        this.stats.manualInvalidations++;
      }

      return deleted > 0;

    } catch (error) {
      console.error('❌ Error:', error.message);
      throw error;
    }
  }

  /**
   * Strategy 5: Tag-Based Invalidation
   * Invalidate all cache entries related to an account
   */
  async invalidateAccountTag(accountId) {
    const tagKey = `tag:account:${accountId}`;

    try {
      // Get all keys with this tag
      const keys = await this.cacheClient.sMembers(tagKey);
      
      if (keys.length === 0) {
        console.log(`⚠️ No cached data for tag ${tagKey}`);
        return 0;
      }

      // Delete all tagged keys
      const deleted = await this.cacheClient.del(...keys);
      
      // Delete tag itself
      await this.cacheClient.del(tagKey);

      console.log(`🗑️ Tag invalidation: Deleted ${deleted} keys for ${accountId}`);
      this.stats.tagInvalidations++;

      return deleted;

    } catch (error) {
      console.error('❌ Error:', error.message);
      throw error;
    }
  }

  /**
   * Get balance with probabilistic early expiration
   * Prevents cache stampede
   */
  async getBalanceWithEarlyExpiration(accountId, ttl = 300) {
    const cacheKey = `balance:${accountId}`;

    try {
      const cached = await this.cacheClient.get(cacheKey);

      if (!cached) {
        // Cache miss - normal flow
        return await this.getBalance(accountId, ttl);
      }

      // Check remaining TTL
      const remainingTTL = await this.cacheClient.ttl(cacheKey);
      const elapsed = ttl - remainingTTL;

      // Probability of early expiration increases as TTL approaches 0
      const probability = (elapsed / ttl) * 0.3; // Max 30% chance

      if (Math.random() < probability) {
        console.log(`🎲 Probabilistic early expiration triggered (${(probability * 100).toFixed(1)}% chance)`);
        
        // Refresh in background
        this.getBalance(accountId, ttl).catch(err => {
          console.error('Background refresh failed:', err.message);
        });
      }

      return JSON.parse(cached);

    } catch (error) {
      console.error('❌ Error:', error.message);
      throw error;
    }
  }

  /**
   * Get transactions with sliding window TTL
   * Reset TTL on every access (keep popular data longer)
   */
  async getTransactions(accountId, ttl = 600) {
    const cacheKey = `transactions:${accountId}`;

    try {
      const cached = await this.cacheClient.get(cacheKey);

      if (cached) {
        // Reset TTL (sliding window)
        await this.cacheClient.expire(cacheKey, ttl);
        console.log(`✅ Cache HIT: ${cacheKey} (TTL reset to ${ttl}s)`);
        return JSON.parse(cached);
      }

      // Fetch from database
      const result = await this.dbPool.query(
        `SELECT transaction_id, amount, description, created_at 
         FROM transactions 
         WHERE from_account = $1 OR to_account = $1 
         ORDER BY created_at DESC 
         LIMIT 10`,
        [accountId]
      );

      const transactions = result.rows;

      // Cache with TTL
      await this.cacheClient.setEx(cacheKey, ttl, JSON.stringify(transactions));
      console.log(`📝 Cached transactions for ${accountId}`);

      // Tag for group invalidation
      await this.cacheClient.sAdd(`tag:account:${accountId}`, cacheKey);

      return transactions;

    } catch (error) {
      console.error('❌ Error:', error.message);
      throw error;
    }
  }

  /**
   * Invalidate transactions cache
   */
  async invalidateTransactions(accountId) {
    const cacheKey = `transactions:${accountId}`;
    
    const deleted = await this.cacheClient.del(cacheKey);
    
    if (deleted) {
      console.log(`🗑️ Invalidated transactions for ${accountId}`);
      this.stats.manualInvalidations++;
    }

    return deleted > 0;
  }

  /**
   * Get invalidation statistics
   */
  getStats() {
    return {
      ...this.stats,
      total: Object.values(this.stats).reduce((sum, val) => sum + val, 0),
    };
  }

  /**
   * Close connections
   */
  async close() {
    await this.cacheClient.quit();
    await this.pubClient.quit();
    await this.subClient.quit();
    await this.dbPool.end();
    console.log('✅ Connections closed');
  }
}

// Usage Example
async function runExample1() {
  console.log('=== Example 1: Multi-Strategy Cache Invalidation ===\n');

  const cacheManager = new BankingCacheManager();

  // Wait for connections
  await new Promise(resolve => setTimeout(resolve, 1000));

  try {
    // Test 1: TTL-based caching
    console.log('\n--- Test 1: TTL-Based Caching ---');
    const balance1 = await cacheManager.getBalance('ACC-001', 10); // 10-second TTL
    console.log('Balance:', balance1);

    // Wait and check TTL
    await new Promise(resolve => setTimeout(resolve, 3000));
    await cacheManager.getBalance('ACC-001'); // Should still be cached

    // Test 2: Write-through invalidation
    console.log('\n--- Test 2: Write-Through Invalidation ---');
    await cacheManager.updateBalanceWriteThrough('ACC-001', 12000);
    const balance2 = await cacheManager.getBalance('ACC-001');
    console.log('Updated balance:', balance2);

    // Test 3: Write-behind invalidation
    console.log('\n--- Test 3: Write-Behind Invalidation ---');
    await cacheManager.updateBalanceWriteBehind('ACC-002', 8000);
    const balance3 = await cacheManager.getBalance('ACC-002'); // Cache miss, will refetch
    console.log('Balance after write-behind:', balance3);

    // Test 4: Manual invalidation
    console.log('\n--- Test 4: Manual Invalidation ---');
    await cacheManager.getBalance('ACC-003');
    await cacheManager.invalidateBalance('ACC-003');
    await cacheManager.getBalance('ACC-003'); // Cache miss

    // Test 5: Tag-based invalidation
    console.log('\n--- Test 5: Tag-Based Invalidation ---');
    await cacheManager.getBalance('ACC-004');
    await cacheManager.getTransactions('ACC-004');
    await cacheManager.invalidateAccountTag('ACC-004'); // Deletes both

    // Test 6: Probabilistic early expiration
    console.log('\n--- Test 6: Probabilistic Early Expiration ---');
    for (let i = 0; i < 5; i++) {
      await cacheManager.getBalanceWithEarlyExpiration('ACC-001', 10);
      await new Promise(resolve => setTimeout(resolve, 2000));
    }

    // Test 7: Sliding window TTL
    console.log('\n--- Test 7: Sliding Window TTL ---');
    await cacheManager.getTransactions('ACC-001', 10);
    await new Promise(resolve => setTimeout(resolve, 3000));
    await cacheManager.getTransactions('ACC-001', 10); // TTL reset
    await new Promise(resolve => setTimeout(resolve, 3000));
    await cacheManager.getTransactions('ACC-001', 10); // TTL reset again

    // Statistics
    console.log('\n--- Invalidation Statistics ---');
    const stats = cacheManager.getStats();
    console.log(stats);

  } finally {
    await cacheManager.close();
  }
}

// Run if executed directly
if (require.main === module) {
  runExample1().catch(console.error);
}

module.exports = { BankingCacheManager };
```

### Running Example 1

```bash
# Make sure Redis and PostgreSQL are running
redis-server &
postgres -D /path/to/data &

# Run example
node Q32_example1_multi_strategy_invalidation.js
```

### Expected Output

```
=== Example 1: Multi-Strategy Cache Invalidation ===

✅ Banking Cache Manager initialized
📡 Subscribed to invalidation events

--- Test 1: TTL-Based Caching ---
⚠️ Cache MISS: balance:ACC-001
📝 Cached with 10s TTL
Balance: { accountId: 'ACC-001', balance: 10000, currency: 'USD', ... }
✅ Cache HIT: balance:ACC-001
  TTL remaining: 7s

--- Test 2: Write-Through Invalidation ---
✅ Write-through: Updated DB and cache for ACC-001
📡 Received invalidation event: { type: 'balance', accountId: 'ACC-001', ... }
✅ Cache HIT: balance:ACC-001
Updated balance: { accountId: 'ACC-001', balance: 12000, ... }

--- Test 3: Write-Behind Invalidation ---
✅ Write-behind: Updated DB, deleted cache for ACC-002
📡 Received invalidation event: { type: 'balance', accountId: 'ACC-002', ... }
⚠️ Cache MISS: balance:ACC-002
Balance after write-behind: { accountId: 'ACC-002', balance: 8000, ... }

--- Test 4: Manual Invalidation ---
⚠️ Cache MISS: balance:ACC-003
🗑️ Manually invalidated balance:ACC-003
⚠️ Cache MISS: balance:ACC-003

--- Test 5: Tag-Based Invalidation ---
⚠️ Cache MISS: balance:ACC-004
⚠️ Cache MISS: transactions:ACC-004
🗑️ Tag invalidation: Deleted 2 keys for ACC-004

--- Test 6: Probabilistic Early Expiration ---
✅ Cache HIT: balance:ACC-001
✅ Cache HIT: balance:ACC-001
🎲 Probabilistic early expiration triggered (12.0% chance)
⚠️ Cache MISS: balance:ACC-001 (background refresh)
✅ Cache HIT: balance:ACC-001

--- Test 7: Sliding Window TTL ---
⚠️ Cache MISS: transactions:ACC-001
✅ Cache HIT: transactions:ACC-001 (TTL reset to 10s)
✅ Cache HIT: transactions:ACC-001 (TTL reset to 10s)

--- Invalidation Statistics ---
{
  ttlExpired: 0,
  manualInvalidations: 3,
  eventDrivenInvalidations: 2,
  tagInvalidations: 1,
  total: 6
}

✅ Connections closed
```

---

## 🎯 Key Takeaways

### Choosing Invalidation Strategy

| Strategy | When to Use | Pros | Cons |
|----------|-------------|------|------|
| **TTL-Based** | Acceptable staleness, simple setup | Easy, automatic | Can be stale for TTL period |
| **Write-Through** | Critical consistency (balances) | Always fresh | Slower writes |
| **Write-Behind** | Write-heavy workloads | Fast writes | Next read slower |
| **Event-Driven** | Distributed systems | Real-time, scalable | Complex setup |
| **Tag-Based** | Related data groups | Batch invalidation | Extra storage |

### Best Practices

1. **Use multiple strategies** based on data characteristics
   ```javascript
   // Critical balance data: Write-through
   await updateBalanceWriteThrough(accountId, newBalance);
   
   // Non-critical profile data: TTL-based
   await redis.setEx('profile:USER-001', 3600, profile);
   ```

2. **Set appropriate TTLs**
   ```javascript
   // Frequently changing: Short TTL
   redis.setEx('stock:AAPL', 10, price);
   
   // Rarely changing: Long TTL
   redis.setEx('config', 86400, config);
   ```

3. **Monitor stale data**
   ```javascript
   const cached = await redis.get(key);
   const cachedTime = new Date(cached.cachedAt);
   const age = Date.now() - cachedTime;
   
   if (age > MAX_AGE) {
     console.warn('Stale data detected!');
   }
   ```

4. **Handle cache stampede**
   ```javascript
   // Use probabilistic early expiration
   // Use distributed locks
   // Use request coalescing
   ```

5. **Invalidate dependencies**
   ```javascript
   // When balance changes, invalidate related caches
   await redis.del(`balance:${accountId}`);
   await redis.del(`transactions:${accountId}`);
   await redis.del(`summary:${accountId}`);
   ```

### Common Pitfalls

- ❌ **No invalidation strategy**: Cache never updates
- ❌ **TTL too long**: Stale data for extended periods
- ❌ **TTL too short**: Frequent cache misses (no benefit)
- ❌ **Cache stampede**: Many requests refresh simultaneously
- ❌ **Forgetting dependencies**: Invalidate balance but not transactions
- ❌ **No monitoring**: Don't know if cache is effective

### Performance Impact

**Effective invalidation results in**:
- 80-95% cache hit rate
- Sub-millisecond response times
- Minimal stale data exposure
- Reduced database load

---

**File**: `Q32_Caching_Cache_Invalidation.md`  
**Status**: ✅ Complete with comprehensive invalidation strategies
