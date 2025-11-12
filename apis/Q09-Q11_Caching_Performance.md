# APIs Questions 9-11: Caching & Performance

---

### Q9. How do you implement caching strategies for APIs?

**Answer:**

```javascript
/**
 * Multi-Level Caching Strategy for Banking API
 */
const Redis = require('ioredis');
const NodeCache = require('node-cache');

class CachingService {
  constructor() {
    // L1: In-memory cache (fastest, smallest)
    this.memoryCache = new NodeCache({
      stdTTL: 60, // 1 minute default
      checkperiod: 120,
      useClones: false // Better performance
    });
    
    // L2: Redis cache (fast, distributed)
    this.redis = new Redis({
      host: 'localhost',
      port: 6379,
      retryStrategy: (times) => Math.min(times * 50, 2000)
    });
    
    // Cache statistics
    this.stats = {
      hits: 0,
      misses: 0,
      errors: 0
    };
  }
  
  /**
   * Get from multi-level cache
   */
  async get(key, options = {}) {
    const { skipMemory = false, skipRedis = false } = options;
    
    try {
      // L1: Check memory cache
      if (!skipMemory) {
        const memValue = this.memoryCache.get(key);
        if (memValue !== undefined) {
          this.stats.hits++;
          console.log(`[CACHE HIT] Memory: ${key}`);
          return memValue;
        }
      }
      
      // L2: Check Redis
      if (!skipRedis) {
        const redisValue = await this.redis.get(key);
        if (redisValue) {
          this.stats.hits++;
          console.log(`[CACHE HIT] Redis: ${key}`);
          
          const parsed = JSON.parse(redisValue);
          
          // Populate L1 cache
          this.memoryCache.set(key, parsed);
          
          return parsed;
        }
      }
      
      this.stats.misses++;
      console.log(`[CACHE MISS]: ${key}`);
      return null;
      
    } catch (error) {
      this.stats.errors++;
      console.error('Cache get error:', error);
      return null;
    }
  }
  
  /**
   * Set in multi-level cache
   */
  async set(key, value, ttl = 300) {
    try {
      // L1: Memory cache
      this.memoryCache.set(key, value, ttl);
      
      // L2: Redis cache
      await this.redis.setex(key, ttl, JSON.stringify(value));
      
      console.log(`[CACHE SET]: ${key} (TTL: ${ttl}s)`);
      
    } catch (error) {
      console.error('Cache set error:', error);
    }
  }
  
  /**
   * Invalidate cache
   */
  async invalidate(pattern) {
    try {
      // Clear memory cache
      if (pattern.includes('*')) {
        const keys = this.memoryCache.keys();
        const regex = new RegExp(pattern.replace('*', '.*'));
        keys.filter(k => regex.test(k)).forEach(k => this.memoryCache.del(k));
      } else {
        this.memoryCache.del(pattern);
      }
      
      // Clear Redis cache
      if (pattern.includes('*')) {
        const keys = await this.redis.keys(pattern);
        if (keys.length > 0) {
          await this.redis.del(...keys);
        }
      } else {
        await this.redis.del(pattern);
      }
      
      console.log(`[CACHE INVALIDATE]: ${pattern}`);
      
    } catch (error) {
      console.error('Cache invalidate error:', error);
    }
  }
  
  /**
   * Cache-aside pattern (lazy loading)
   */
  async cacheAside(key, fetchFunction, ttl = 300) {
    // Try to get from cache
    const cached = await this.get(key);
    if (cached !== null) {
      return cached;
    }
    
    // Cache miss - fetch from source
    const data = await fetchFunction();
    
    // Store in cache
    if (data !== null && data !== undefined) {
      await this.set(key, data, ttl);
    }
    
    return data;
  }
  
  /**
   * Write-through cache pattern
   */
  async writeThrough(key, value, writeFunction, ttl = 300) {
    // Write to database first
    const result = await writeFunction(value);
    
    // Then update cache
    await this.set(key, result, ttl);
    
    return result;
  }
  
  /**
   * Write-behind (write-back) cache pattern
   */
  async writeBehind(key, value, ttl = 300) {
    // Write to cache immediately
    await this.set(key, value, ttl);
    
    // Queue write to database (async)
    await this.redis.lpush('write_queue', JSON.stringify({ key, value }));
    
    return value;
  }
  
  /**
   * Get cache statistics
   */
  getStats() {
    const total = this.stats.hits + this.stats.misses;
    const hitRate = total > 0 ? (this.stats.hits / total * 100).toFixed(2) : 0;
    
    return {
      hits: this.stats.hits,
      misses: this.stats.misses,
      errors: this.stats.errors,
      total,
      hitRate: `${hitRate}%`,
      memoryKeys: this.memoryCache.keys().length
    };
  }
}

/**
 * Cache Middleware for Express
 */
class CacheMiddleware {
  constructor(cachingService) {
    this.cache = cachingService;
  }
  
  /**
   * HTTP cache middleware
   */
  httpCache(options = {}) {
    const { 
      ttl = 300, 
      keyGenerator = (req) => `${req.method}:${req.originalUrl}`,
      varyBy = []
    } = options;
    
    return async (req, res, next) => {
      // Only cache GET requests
      if (req.method !== 'GET') {
        return next();
      }
      
      // Generate cache key
      let cacheKey = keyGenerator(req);
      
      // Add vary-by headers
      if (varyBy.length > 0) {
        const varyValues = varyBy.map(header => req.get(header) || '').join(':');
        cacheKey = `${cacheKey}:${varyValues}`;
      }
      
      // Check cache
      const cached = await this.cache.get(cacheKey);
      
      if (cached) {
        res.set('X-Cache', 'HIT');
        res.set('X-Cache-Key', cacheKey);
        return res.json(cached);
      }
      
      // Cache miss - intercept response
      res.set('X-Cache', 'MISS');
      res.set('X-Cache-Key', cacheKey);
      
      const originalJson = res.json.bind(res);
      
      res.json = (data) => {
        // Store in cache
        this.cache.set(cacheKey, data, ttl);
        return originalJson(data);
      };
      
      next();
    };
  }
  
  /**
   * ETag support
   */
  etag() {
    const crypto = require('crypto');
    
    return (req, res, next) => {
      const originalJson = res.json.bind(res);
      
      res.json = (data) => {
        const content = JSON.stringify(data);
        const etag = crypto.createHash('md5').update(content).digest('hex');
        
        res.set('ETag', `"${etag}"`);
        res.set('Cache-Control', 'private, max-age=300');
        
        // Check If-None-Match header
        const clientEtag = req.get('If-None-Match');
        if (clientEtag === `"${etag}"`) {
          return res.status(304).end();
        }
        
        return originalJson(data);
      };
      
      next();
    };
  }
  
  /**
   * Cache control headers
   */
  cacheControl(options = {}) {
    const {
      maxAge = 300,
      sMaxAge = null,
      privacy = 'private',
      noCache = false,
      noStore = false,
      mustRevalidate = false
    } = options;
    
    return (req, res, next) => {
      const directives = [];
      
      directives.push(privacy);
      
      if (noCache) directives.push('no-cache');
      if (noStore) directives.push('no-store');
      if (mustRevalidate) directives.push('must-revalidate');
      
      directives.push(`max-age=${maxAge}`);
      if (sMaxAge) directives.push(`s-maxage=${sMaxAge}`);
      
      res.set('Cache-Control', directives.join(', '));
      next();
    };
  }
}

/**
 * Banking-specific caching
 */
class BankingCacheService {
  constructor(cachingService) {
    this.cache = cachingService;
  }
  
  /**
   * Cache customer data (rarely changes)
   */
  async getCustomer(customerId, db) {
    return this.cache.cacheAside(
      `customer:${customerId}`,
      async () => {
        const result = await db.query(
          'SELECT * FROM customers WHERE id = $1',
          [customerId]
        );
        return result.rows[0];
      },
      3600 // 1 hour TTL
    );
  }
  
  /**
   * Cache account balance (changes frequently)
   */
  async getAccountBalance(accountId, db) {
    return this.cache.cacheAside(
      `account:${accountId}:balance`,
      async () => {
        const result = await db.query(
          'SELECT balance FROM accounts WHERE id = $1',
          [accountId]
        );
        return result.rows[0]?.balance;
      },
      60 // 1 minute TTL
    );
  }
  
  /**
   * Cache transaction history (immutable after creation)
   */
  async getTransactions(accountId, page, limit, db) {
    const cacheKey = `transactions:${accountId}:page:${page}:limit:${limit}`;
    
    return this.cache.cacheAside(
      cacheKey,
      async () => {
        const offset = (page - 1) * limit;
        const result = await db.query(
          'SELECT * FROM transactions WHERE account_id = $1 ORDER BY created_at DESC LIMIT $2 OFFSET $3',
          [accountId, limit, offset]
        );
        return result.rows;
      },
      600 // 10 minutes TTL
    );
  }
  
  /**
   * Invalidate cache on update
   */
  async updateAccount(accountId, updates, db) {
    // Update database
    const result = await db.query(
      'UPDATE accounts SET balance = $1 WHERE id = $2 RETURNING *',
      [updates.balance, accountId]
    );
    
    // Invalidate cache
    await this.cache.invalidate(`account:${accountId}:*`);
    
    return result.rows[0];
  }
  
  /**
   * Cache exchange rates (changes daily)
   */
  async getExchangeRates(currency) {
    return this.cache.cacheAside(
      `exchange_rates:${currency}`,
      async () => {
        // Fetch from external API
        const response = await fetch(`https://api.exchangerate.com/rates/${currency}`);
        return response.json();
      },
      86400 // 24 hours TTL
    );
  }
  
  /**
   * Cache loan products (rarely changes)
   */
  async getLoanProducts() {
    return this.cache.cacheAside(
      'loan_products',
      async () => {
        // Fetch from database
        return [
          { type: 'personal', rate: 5.5, maxAmount: 500000 },
          { type: 'home', rate: 3.5, maxAmount: 5000000 },
          { type: 'auto', rate: 4.0, maxAmount: 300000 }
        ];
      },
      3600 // 1 hour TTL
    );
  }
}

/**
 * Usage Examples
 */

// Initialize services
const cachingService = new CachingService();
const cacheMiddleware = new CacheMiddleware(cachingService);
const bankingCache = new BankingCacheService(cachingService);

// Express routes with caching
const express = require('express');
const router = express.Router();

// Cache public data for 5 minutes
router.get('/api/exchange-rates/:currency',
  cacheMiddleware.httpCache({ ttl: 300 }),
  async (req, res) => {
    const rates = await bankingCache.getExchangeRates(req.params.currency);
    res.json(rates);
  }
);

// Cache customer data with ETag
router.get('/api/customers/:id',
  cacheMiddleware.etag(),
  async (req, res) => {
    const customer = await bankingCache.getCustomer(req.params.id, db);
    res.json(customer);
  }
);

// Short cache for account balance
router.get('/api/accounts/:id/balance',
  cacheMiddleware.cacheControl({ maxAge: 60 }),
  async (req, res) => {
    const balance = await bankingCache.getAccountBalance(req.params.id, db);
    res.json({ balance });
  }
);

// Cache transaction history
router.get('/api/accounts/:id/transactions',
  cacheMiddleware.httpCache({ 
    ttl: 600,
    keyGenerator: (req) => `transactions:${req.params.id}:${req.query.page}:${req.query.limit}`
  }),
  async (req, res) => {
    const page = parseInt(req.query.page) || 1;
    const limit = parseInt(req.query.limit) || 20;
    const transactions = await bankingCache.getTransactions(req.params.id, page, limit, db);
    res.json(transactions);
  }
);

module.exports = { 
  CachingService, 
  CacheMiddleware, 
  BankingCacheService 
};
```

---

### Q10. How do you implement API pagination strategies?

**Answer:**

```javascript
/**
 * Pagination Strategies for Banking APIs
 */

/**
 * 1. Offset-based pagination (traditional)
 */
class OffsetPagination {
  /**
   * Calculate pagination metadata
   */
  static getMetadata(page, limit, totalCount) {
    const totalPages = Math.ceil(totalCount / limit);
    const hasNextPage = page < totalPages;
    const hasPreviousPage = page > 1;
    
    return {
      page,
      limit,
      totalCount,
      totalPages,
      hasNextPage,
      hasPreviousPage,
      nextPage: hasNextPage ? page + 1 : null,
      previousPage: hasPreviousPage ? page - 1 : null
    };
  }
  
  /**
   * Generate pagination links (HATEOAS)
   */
  static getLinks(baseUrl, page, limit, totalPages) {
    const links = {
      self: `${baseUrl}?page=${page}&limit=${limit}`
    };
    
    if (page > 1) {
      links.first = `${baseUrl}?page=1&limit=${limit}`;
      links.previous = `${baseUrl}?page=${page - 1}&limit=${limit}`;
    }
    
    if (page < totalPages) {
      links.next = `${baseUrl}?page=${page + 1}&limit=${limit}`;
      links.last = `${baseUrl}?page=${totalPages}&limit=${limit}`;
    }
    
    return links;
  }
  
  /**
   * Query database with offset
   */
  static async query(db, sql, params, page, limit) {
    const offset = (page - 1) * limit;
    
    // Get total count
    const countResult = await db.query(
      sql.replace(/SELECT .+ FROM/, 'SELECT COUNT(*) FROM').split('ORDER BY')[0],
      params
    );
    const totalCount = parseInt(countResult.rows[0].count);
    
    // Get paginated results
    const dataResult = await db.query(
      `${sql} LIMIT $${params.length + 1} OFFSET $${params.length + 2}`,
      [...params, limit, offset]
    );
    
    return {
      data: dataResult.rows,
      pagination: this.getMetadata(page, limit, totalCount)
    };
  }
}

/**
 * 2. Cursor-based pagination (efficient for large datasets)
 */
class CursorPagination {
  /**
   * Encode cursor
   */
  static encodeCursor(id, createdAt) {
    const cursor = Buffer.from(JSON.stringify({ id, createdAt })).toString('base64');
    return cursor;
  }
  
  /**
   * Decode cursor
   */
  static decodeCursor(cursor) {
    try {
      const decoded = Buffer.from(cursor, 'base64').toString('utf-8');
      return JSON.parse(decoded);
    } catch (error) {
      return null;
    }
  }
  
  /**
   * Query with cursor (forward pagination)
   */
  static async queryForward(db, tableName, limit, cursor = null, filters = {}) {
    let whereClause = [];
    let params = [];
    let paramIndex = 1;
    
    // Apply filters
    Object.entries(filters).forEach(([key, value]) => {
      whereClause.push(`${key} = $${paramIndex}`);
      params.push(value);
      paramIndex++;
    });
    
    // Apply cursor
    if (cursor) {
      const decoded = this.decodeCursor(cursor);
      if (decoded) {
        whereClause.push(`created_at < $${paramIndex}`);
        params.push(decoded.createdAt);
        paramIndex++;
      }
    }
    
    const whereStr = whereClause.length > 0 ? `WHERE ${whereClause.join(' AND ')}` : '';
    
    // Fetch limit + 1 to check if there's more data
    const query = `
      SELECT * FROM ${tableName}
      ${whereStr}
      ORDER BY created_at DESC
      LIMIT $${paramIndex}
    `;
    params.push(limit + 1);
    
    const result = await db.query(query, params);
    
    const hasMore = result.rows.length > limit;
    const data = hasMore ? result.rows.slice(0, limit) : result.rows;
    
    const edges = data.map(row => ({
      node: row,
      cursor: this.encodeCursor(row.id, row.created_at)
    }));
    
    return {
      edges,
      pageInfo: {
        hasNextPage: hasMore,
        endCursor: edges.length > 0 ? edges[edges.length - 1].cursor : null,
        startCursor: edges.length > 0 ? edges[0].cursor : null
      }
    };
  }
}

/**
 * 3. Keyset pagination (seek method - most efficient)
 */
class KeysetPagination {
  /**
   * Query using keyset
   */
  static async query(db, tableName, limit, lastId = null, filters = {}) {
    let whereClause = [];
    let params = [];
    let paramIndex = 1;
    
    // Apply filters
    Object.entries(filters).forEach(([key, value]) => {
      whereClause.push(`${key} = $${paramIndex}`);
      params.push(value);
      paramIndex++;
    });
    
    // Apply keyset
    if (lastId) {
      whereClause.push(`id > $${paramIndex}`);
      params.push(lastId);
      paramIndex++;
    }
    
    const whereStr = whereClause.length > 0 ? `WHERE ${whereClause.join(' AND ')}` : '';
    
    const query = `
      SELECT * FROM ${tableName}
      ${whereStr}
      ORDER BY id ASC
      LIMIT $${paramIndex}
    `;
    params.push(limit + 1);
    
    const result = await db.query(query, params);
    
    const hasMore = result.rows.length > limit;
    const data = hasMore ? result.rows.slice(0, limit) : result.rows;
    
    return {
      data,
      pagination: {
        limit,
        hasMore,
        nextId: hasMore ? data[data.length - 1].id : null
      }
    };
  }
}

/**
 * Banking API Pagination Examples
 */
class BankingPaginationService {
  constructor(db) {
    this.db = db;
  }
  
  /**
   * Get paginated customer list (offset-based)
   */
  async getCustomers(page = 1, limit = 20, filters = {}) {
    let whereClause = [];
    let params = [];
    let paramIndex = 1;
    
    if (filters.name) {
      whereClause.push(`name ILIKE $${paramIndex}`);
      params.push(`%${filters.name}%`);
      paramIndex++;
    }
    
    if (filters.nationality) {
      whereClause.push(`nationality = $${paramIndex}`);
      params.push(filters.nationality);
      paramIndex++;
    }
    
    const whereStr = whereClause.length > 0 ? `WHERE ${whereClause.join(' AND ')}` : '';
    
    const sql = `
      SELECT id, emirates_id, name, email, phone, nationality, created_at
      FROM customers
      ${whereStr}
      ORDER BY created_at DESC
    `;
    
    return OffsetPagination.query(this.db, sql, params, page, limit);
  }
  
  /**
   * Get transaction history (cursor-based)
   */
  async getTransactions(accountId, limit = 20, cursor = null) {
    return CursorPagination.queryForward(
      this.db,
      'transactions',
      limit,
      cursor,
      { account_id: accountId }
    );
  }
  
  /**
   * Get accounts (keyset-based)
   */
  async getAccounts(customerId, limit = 20, lastId = null) {
    return KeysetPagination.query(
      this.db,
      'accounts',
      limit,
      lastId,
      { customer_id: customerId }
    );
  }
}

/**
 * Express Pagination Middleware
 */
function paginationMiddleware(options = {}) {
  const {
    defaultLimit = 20,
    maxLimit = 100,
    type = 'offset' // 'offset', 'cursor', or 'keyset'
  } = options;
  
  return (req, res, next) => {
    if (type === 'offset') {
      // Parse page and limit
      req.pagination = {
        page: Math.max(1, parseInt(req.query.page) || 1),
        limit: Math.min(maxLimit, parseInt(req.query.limit) || defaultLimit)
      };
    } else if (type === 'cursor') {
      // Parse cursor and limit
      req.pagination = {
        cursor: req.query.cursor || null,
        limit: Math.min(maxLimit, parseInt(req.query.limit) || defaultLimit)
      };
    } else if (type === 'keyset') {
      // Parse lastId and limit
      req.pagination = {
        lastId: req.query.lastId || null,
        limit: Math.min(maxLimit, parseInt(req.query.limit) || defaultLimit)
      };
    }
    
    next();
  };
}

/**
 * Express Routes with Pagination
 */
const express = require('express');
const router = express.Router();

// Offset-based pagination
router.get('/customers',
  paginationMiddleware({ type: 'offset', maxLimit: 50 }),
  async (req, res) => {
    const { page, limit } = req.pagination;
    const filters = {
      name: req.query.name,
      nationality: req.query.nationality
    };
    
    const result = await paginationService.getCustomers(page, limit, filters);
    
    res.json({
      data: result.data,
      pagination: result.pagination,
      _links: OffsetPagination.getLinks(
        `${req.protocol}://${req.get('host')}${req.path}`,
        page,
        limit,
        result.pagination.totalPages
      )
    });
  }
);

// Cursor-based pagination
router.get('/accounts/:accountId/transactions',
  paginationMiddleware({ type: 'cursor', maxLimit: 100 }),
  async (req, res) => {
    const { cursor, limit } = req.pagination;
    const { accountId } = req.params;
    
    const result = await paginationService.getTransactions(accountId, limit, cursor);
    
    res.json({
      edges: result.edges,
      pageInfo: result.pageInfo
    });
  }
);

// Keyset pagination
router.get('/customers/:customerId/accounts',
  paginationMiddleware({ type: 'keyset', maxLimit: 50 }),
  async (req, res) => {
    const { lastId, limit } = req.pagination;
    const { customerId } = req.params;
    
    const result = await paginationService.getAccounts(customerId, limit, lastId);
    
    res.json({
      data: result.data,
      pagination: result.pagination
    });
  }
);

module.exports = {
  OffsetPagination,
  CursorPagination,
  KeysetPagination,
  BankingPaginationService,
  paginationMiddleware
};
```

---

### Q11. How do you implement request/response validation with schemas?

**Answer:**

```javascript
/**
 * Request/Response Validation with Joi and AJV
 */
const Joi = require('joi');
const Ajv = require('ajv');
const addFormats = require('ajv-formats');

/**
 * Joi Validation Schemas for Banking
 */
const BankingSchemas = {
  // Emirates ID validation
  emiratesId: Joi.string()
    .pattern(/^784-\d{4}-\d{7}-\d{1}$/)
    .required()
    .messages({
      'string.pattern.base': 'Emirates ID must be in format 784-XXXX-XXXXXXX-X'
    }),
  
  // Phone number validation
  phone: Joi.string()
    .pattern(/^\+971-\d{2}-\d{7}$/)
    .messages({
      'string.pattern.base': 'Phone must be in format +971-XX-XXXXXXX'
    }),
  
  // IBAN validation
  iban: Joi.string()
    .pattern(/^AE\d{21}$/)
    .messages({
      'string.pattern.base': 'IBAN must be in format AEXXXXXXXXXXXXXXXXX (23 characters)'
    }),
  
  // Customer schema
  customer: Joi.object({
    emiratesId: Joi.string()
      .pattern(/^784-\d{4}-\d{7}-\d{1}$/)
      .required(),
    name: Joi.string()
      .min(2)
      .max(200)
      .required(),
    email: Joi.string()
      .email()
      .required(),
    phone: Joi.string()
      .pattern(/^\+971-\d{2}-\d{7}$/)
      .required(),
    dateOfBirth: Joi.date()
      .max('now')
      .min('1900-01-01')
      .required(),
    nationality: Joi.string()
      .length(3)
      .uppercase()
      .required(),
    address: Joi.object({
      street: Joi.string().required(),
      city: Joi.string().required(),
      emirate: Joi.string()
        .valid('Abu Dhabi', 'Dubai', 'Sharjah', 'Ajman', 'Umm Al Quwain', 'Ras Al Khaimah', 'Fujairah')
        .required(),
      poBox: Joi.string().optional()
    }).required()
  }),
  
  // Account schema
  account: Joi.object({
    customerId: Joi.string().uuid().required(),
    accountType: Joi.string()
      .valid('savings', 'current', 'fixed_deposit')
      .required(),
    currency: Joi.string()
      .valid('AED', 'USD', 'EUR', 'GBP')
      .default('AED'),
    initialDeposit: Joi.number()
      .min(1000)
      .max(10000000)
      .when('accountType', {
        is: 'savings',
        then: Joi.number().min(1000),
        otherwise: Joi.number().min(5000)
      })
      .required()
      .messages({
        'number.min': 'Minimum deposit for savings is AED 1,000, current is AED 5,000'
      })
  }),
  
  // Transaction schema
  transaction: Joi.object({
    accountId: Joi.string().uuid().required(),
    amount: Joi.number()
      .positive()
      .precision(2)
      .max(100000)
      .required()
      .messages({
        'number.max': 'Transaction amount cannot exceed AED 100,000'
      }),
    type: Joi.string()
      .valid('debit', 'credit')
      .required(),
    description: Joi.string()
      .max(500)
      .optional(),
    beneficiaryIban: Joi.when('type', {
      is: 'debit',
      then: Joi.string()
        .pattern(/^AE\d{21}$/)
        .required(),
      otherwise: Joi.forbidden()
    })
  }),
  
  // Loan application schema
  loanApplication: Joi.object({
    customerId: Joi.string().uuid().required(),
    loanType: Joi.string()
      .valid('personal', 'home', 'auto', 'business')
      .required(),
    amount: Joi.number()
      .positive()
      .when('loanType', {
        is: 'personal',
        then: Joi.number().min(10000).max(500000),
        otherwise: Joi.when('loanType', {
          is: 'home',
          then: Joi.number().min(100000).max(5000000),
          otherwise: Joi.when('loanType', {
            is: 'auto',
            then: Joi.number().min(20000).max(300000),
            otherwise: Joi.number().min(50000).max(2000000) // business
          })
        })
      })
      .required(),
    tenureMonths: Joi.number()
      .integer()
      .min(12)
      .max(360)
      .required(),
    monthlyIncome: Joi.number()
      .positive()
      .min(5000)
      .required(),
    employmentType: Joi.string()
      .valid('employed', 'self_employed', 'business_owner')
      .required()
  })
};

/**
 * Joi Validation Middleware
 */
function validateRequest(schema, property = 'body') {
  return (req, res, next) => {
    const { error, value } = schema.validate(req[property], {
      abortEarly: false,
      stripUnknown: true,
      convert: true
    });
    
    if (error) {
      const details = error.details.reduce((acc, detail) => {
        acc[detail.path.join('.')] = detail.message;
        return acc;
      }, {});
      
      return res.status(400).json({
        error: 'ValidationError',
        message: 'Request validation failed',
        details
      });
    }
    
    // Replace with validated and sanitized data
    req[property] = value;
    next();
  };
}

/**
 * AJV JSON Schema Validation (faster than Joi)
 */
const ajv = new Ajv({ allErrors: true, removeAdditional: true });
addFormats(ajv);

const customerJsonSchema = {
  type: 'object',
  required: ['emiratesId', 'name', 'email', 'phone'],
  properties: {
    emiratesId: {
      type: 'string',
      pattern: '^784-\\d{4}-\\d{7}-\\d{1}$'
    },
    name: {
      type: 'string',
      minLength: 2,
      maxLength: 200
    },
    email: {
      type: 'string',
      format: 'email'
    },
    phone: {
      type: 'string',
      pattern: '^\\+971-\\d{2}-\\d{7}$'
    },
    dateOfBirth: {
      type: 'string',
      format: 'date'
    },
    nationality: {
      type: 'string',
      minLength: 3,
      maxLength: 3
    }
  },
  additionalProperties: false
};

const validateCustomerAjv = ajv.compile(customerJsonSchema);

function validateWithAjv(schema) {
  return (req, res, next) => {
    const valid = schema(req.body);
    
    if (!valid) {
      const details = schema.errors.reduce((acc, error) => {
        const field = error.instancePath.substring(1) || error.params.missingProperty;
        acc[field] = error.message;
        return acc;
      }, {});
      
      return res.status(400).json({
        error: 'ValidationError',
        message: 'Request validation failed',
        details
      });
    }
    
    next();
  };
}

/**
 * Custom Validators
 */
class CustomValidators {
  /**
   * Validate Emirates ID checksum
   */
  static validateEmiratesId(emiratesId) {
    // Remove dashes
    const clean = emiratesId.replace(/-/g, '');
    
    // Check length
    if (clean.length !== 18) return false;
    
    // Validate checksum (simplified)
    const digits = clean.split('').map(Number);
    const checksum = digits[17];
    
    // Luhn algorithm
    let sum = 0;
    for (let i = 0; i < 17; i++) {
      let digit = digits[i];
      if (i % 2 === 0) {
        digit *= 2;
        if (digit > 9) digit -= 9;
      }
      sum += digit;
    }
    
    return (10 - (sum % 10)) % 10 === checksum;
  }
  
  /**
   * Validate IBAN checksum
   */
  static validateIban(iban) {
    // Remove spaces
    const clean = iban.replace(/\s/g, '');
    
    // Check format
    if (!/^AE\d{21}$/.test(clean)) return false;
    
    // Rearrange: move first 4 chars to end
    const rearranged = clean.substring(4) + clean.substring(0, 4);
    
    // Replace letters with numbers (A=10, E=14)
    const numeric = rearranged.replace(/[A-Z]/g, char => char.charCodeAt(0) - 55);
    
    // Calculate mod 97
    let remainder = numeric;
    while (remainder.length > 2) {
      const block = remainder.substring(0, 9);
      remainder = (parseInt(block, 10) % 97) + remainder.substring(block.length);
    }
    
    return parseInt(remainder, 10) % 97 === 1;
  }
  
  /**
   * Validate credit card using Luhn algorithm
   */
  static validateCreditCard(cardNumber) {
    const clean = cardNumber.replace(/\s/g, '');
    
    if (!/^\d{16}$/.test(clean)) return false;
    
    const digits = clean.split('').map(Number);
    let sum = 0;
    
    for (let i = digits.length - 1; i >= 0; i--) {
      let digit = digits[i];
      
      if ((digits.length - 1 - i) % 2 === 1) {
        digit *= 2;
        if (digit > 9) digit -= 9;
      }
      
      sum += digit;
    }
    
    return sum % 10 === 0;
  }
}

/**
 * Response Validation (ensure API returns consistent structure)
 */
function validateResponse(schema) {
  return (req, res, next) => {
    const originalJson = res.json.bind(res);
    
    res.json = (data) => {
      const { error, value } = schema.validate(data, {
        abortEarly: false,
        stripUnknown: false
      });
      
      if (error) {
        console.error('Response validation failed:', error.details);
        
        // In production, log error but don't expose to client
        return originalJson({
          error: 'InternalServerError',
          message: 'Response validation failed'
        });
      }
      
      return originalJson(value);
    };
    
    next();
  };
}

/**
 * Express Routes with Validation
 */
const express = require('express');
const router = express.Router();

// Create customer with Joi validation
router.post('/customers',
  validateRequest(BankingSchemas.customer),
  async (req, res) => {
    // Additional custom validation
    if (!CustomValidators.validateEmiratesId(req.body.emiratesId)) {
      return res.status(400).json({
        error: 'ValidationError',
        message: 'Invalid Emirates ID checksum'
      });
    }
    
    // Create customer
    const customer = await db.query(
      'INSERT INTO customers (emirates_id, name, email, phone) VALUES ($1, $2, $3, $4) RETURNING *',
      [req.body.emiratesId, req.body.name, req.body.email, req.body.phone]
    );
    
    res.status(201).json(customer.rows[0]);
  }
);

// Create account with validation
router.post('/accounts',
  validateRequest(BankingSchemas.account),
  async (req, res) => {
    // Business logic
    const account = await createAccount(req.body);
    res.status(201).json(account);
  }
);

// Create transaction with validation
router.post('/transactions',
  validateRequest(BankingSchemas.transaction),
  async (req, res) => {
    // Validate IBAN if present
    if (req.body.beneficiaryIban && !CustomValidators.validateIban(req.body.beneficiaryIban)) {
      return res.status(400).json({
        error: 'ValidationError',
        message: 'Invalid IBAN checksum'
      });
    }
    
    const transaction = await createTransaction(req.body);
    res.status(201).json(transaction);
  }
);

module.exports = {
  BankingSchemas,
  validateRequest,
  validateWithAjv,
  CustomValidators,
  validateResponse
};
```

---

**Summary Q9-Q11:**
- Multi-level caching (Memory + Redis) with cache-aside/write-through patterns ✅
- Three pagination strategies (Offset, Cursor, Keyset) with HATEOAS ✅
- Comprehensive validation (Joi, AJV, custom validators) with Emirates ID/IBAN checksums ✅
