# APIs Questions 21-23: Performance Optimization & Scaling

---

### Q21. How do you implement load balancing and horizontal scaling for APIs?

**Answer:**

```javascript
/**
 * Load Balancing Strategies for Banking APIs
 */

/**
 * Round Robin Load Balancer
 */
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

/**
 * Weighted Round Robin
 */
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

/**
 * Least Connections Load Balancer
 */
class LeastConnectionsLoadBalancer {
  constructor(servers) {
    this.servers = servers.map(url => ({
      url,
      activeConnections: 0,
      isHealthy: true
    }));
  }
  
  getNextServer() {
    const healthyServers = this.servers.filter(s => s.isHealthy);
    
    if (healthyServers.length === 0) {
      throw new Error('No healthy servers available');
    }
    
    // Find server with least connections
    const server = healthyServers.reduce((min, curr) =>
      curr.activeConnections < min.activeConnections ? curr : min
    );
    
    server.activeConnections++;
    return server;
  }
  
  releaseConnection(serverUrl) {
    const server = this.servers.find(s => s.url === serverUrl);
    if (server && server.activeConnections > 0) {
      server.activeConnections--;
    }
  }
  
  markUnhealthy(serverUrl) {
    const server = this.servers.find(s => s.url === serverUrl);
    if (server) {
      server.isHealthy = false;
      server.activeConnections = 0;
    }
  }
  
  markHealthy(serverUrl) {
    const server = this.servers.find(s => s.url === serverUrl);
    if (server) {
      server.isHealthy = true;
    }
  }
}

/**
 * IP Hash Load Balancer (Session Affinity)
 */
class IPHashLoadBalancer {
  constructor(servers) {
    this.servers = servers;
  }
  
  getServerForIP(ip) {
    const hash = this.hashCode(ip);
    const index = Math.abs(hash) % this.servers.length;
    return this.servers[index];
  }
  
  hashCode(str) {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash = hash & hash; // Convert to 32-bit integer
    }
    return hash;
  }
}

/**
 * Load Balancer with Health Checks
 */
class LoadBalancerWithHealthCheck {
  constructor(servers, strategy = 'round-robin') {
    this.servers = servers.map(url => ({
      url,
      isHealthy: true,
      lastCheck: null,
      failureCount: 0,
      responseTime: 0
    }));
    
    this.strategy = strategy;
    this.healthCheckInterval = 10000; // 10 seconds
    this.maxFailures = 3;
    
    this.startHealthChecks();
  }
  
  async getNextServer(clientIP = null) {
    const healthyServers = this.servers.filter(s => s.isHealthy);
    
    if (healthyServers.length === 0) {
      throw new Error('No healthy servers available');
    }
    
    switch (this.strategy) {
      case 'round-robin':
        return this.roundRobin(healthyServers);
        
      case 'least-response-time':
        return this.leastResponseTime(healthyServers);
        
      case 'ip-hash':
        return this.ipHash(healthyServers, clientIP);
        
      default:
        return healthyServers[0];
    }
  }
  
  roundRobin(servers) {
    if (!this.currentIndex) this.currentIndex = 0;
    const server = servers[this.currentIndex % servers.length];
    this.currentIndex++;
    return server.url;
  }
  
  leastResponseTime(servers) {
    return servers.reduce((min, curr) =>
      curr.responseTime < min.responseTime ? curr : min
    ).url;
  }
  
  ipHash(servers, clientIP) {
    const hash = this.hashCode(clientIP);
    const index = Math.abs(hash) % servers.length;
    return servers[index].url;
  }
  
  hashCode(str) {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i);
    }
    return hash;
  }
  
  async healthCheck(server) {
    const start = Date.now();
    
    try {
      const response = await fetch(`${server.url}/health`, {
        method: 'GET',
        timeout: 5000
      });
      
      if (response.ok) {
        server.responseTime = Date.now() - start;
        server.failureCount = 0;
        server.isHealthy = true;
        server.lastCheck = new Date();
      } else {
        this.handleFailure(server);
      }
    } catch (error) {
      this.handleFailure(server);
    }
  }
  
  handleFailure(server) {
    server.failureCount++;
    server.lastCheck = new Date();
    
    if (server.failureCount >= this.maxFailures) {
      server.isHealthy = false;
      console.error(`Server marked unhealthy: ${server.url}`);
    }
  }
  
  startHealthChecks() {
    setInterval(() => {
      this.servers.forEach(server => {
        this.healthCheck(server);
      });
    }, this.healthCheckInterval);
    
    // Initial health check
    this.servers.forEach(server => this.healthCheck(server));
  }
  
  getServerStats() {
    return this.servers.map(server => ({
      url: server.url,
      isHealthy: server.isHealthy,
      failureCount: server.failureCount,
      responseTime: `${server.responseTime}ms`,
      lastCheck: server.lastCheck
    }));
  }
}

/**
 * Horizontal Scaling Configuration
 */
class HorizontalScalingManager {
  constructor(kubernetes, minReplicas = 2, maxReplicas = 10) {
    this.kubernetes = kubernetes;
    this.minReplicas = minReplicas;
    this.maxReplicas = maxReplicas;
    this.currentReplicas = minReplicas;
  }
  
  /**
   * Auto-scaling based on metrics
   */
  async autoScale(metrics) {
    const { cpuUsage, memoryUsage, requestRate, responseTime } = metrics;
    
    let targetReplicas = this.currentReplicas;
    
    // Scale up conditions
    if (cpuUsage > 70 || memoryUsage > 75) {
      targetReplicas = Math.min(this.currentReplicas + 2, this.maxReplicas);
      console.log(`Scaling up due to high resource usage (CPU: ${cpuUsage}%, Memory: ${memoryUsage}%)`);
    } else if (responseTime > 1000) { // > 1 second
      targetReplicas = Math.min(this.currentReplicas + 1, this.maxReplicas);
      console.log(`Scaling up due to slow response time (${responseTime}ms)`);
    } else if (requestRate > 1000) { // > 1000 req/s
      targetReplicas = Math.min(this.currentReplicas + 1, this.maxReplicas);
      console.log(`Scaling up due to high request rate (${requestRate} req/s)`);
    }
    
    // Scale down conditions
    else if (cpuUsage < 30 && memoryUsage < 40 && this.currentReplicas > this.minReplicas) {
      targetReplicas = Math.max(this.currentReplicas - 1, this.minReplicas);
      console.log(`Scaling down due to low resource usage`);
    }
    
    if (targetReplicas !== this.currentReplicas) {
      await this.scale(targetReplicas);
    }
  }
  
  /**
   * Scale deployment
   */
  async scale(replicas) {
    console.log(`Scaling from ${this.currentReplicas} to ${replicas} replicas`);
    
    // Kubernetes API call
    await this.kubernetes.scaleDeployment('banking-api', replicas);
    
    this.currentReplicas = replicas;
  }
  
  /**
   * Get Kubernetes HPA (Horizontal Pod Autoscaler) config
   */
  getHPAConfig() {
    return {
      apiVersion: 'autoscaling/v2',
      kind: 'HorizontalPodAutoscaler',
      metadata: {
        name: 'banking-api-hpa',
        namespace: 'production'
      },
      spec: {
        scaleTargetRef: {
          apiVersion: 'apps/v1',
          kind: 'Deployment',
          name: 'banking-api'
        },
        minReplicas: this.minReplicas,
        maxReplicas: this.maxReplicas,
        metrics: [
          {
            type: 'Resource',
            resource: {
              name: 'cpu',
              target: {
                type: 'Utilization',
                averageUtilization: 70
              }
            }
          },
          {
            type: 'Resource',
            resource: {
              name: 'memory',
              target: {
                type: 'Utilization',
                averageUtilization: 75
              }
            }
          },
          {
            type: 'Pods',
            pods: {
              metric: {
                name: 'http_requests_per_second'
              },
              target: {
                type: 'AverageValue',
                averageValue: '1000'
              }
            }
          }
        ],
        behavior: {
          scaleDown: {
            stabilizationWindowSeconds: 300,
            policies: [
              {
                type: 'Percent',
                value: 10,
                periodSeconds: 60
              }
            ]
          },
          scaleUp: {
            stabilizationWindowSeconds: 0,
            policies: [
              {
                type: 'Percent',
                value: 50,
                periodSeconds: 15
              },
              {
                type: 'Pods',
                value: 2,
                periodSeconds: 15
              }
            ],
            selectPolicy: 'Max'
          }
        }
      }
    };
  }
}

/**
 * Nginx Load Balancer Configuration
 */
const nginxLoadBalancerConfig = `
upstream banking_api {
    least_conn;  # Least connections algorithm
    
    # Server pool
    server api-1.enbd.com:3000 weight=3 max_fails=3 fail_timeout=30s;
    server api-2.enbd.com:3000 weight=3 max_fails=3 fail_timeout=30s;
    server api-3.enbd.com:3000 weight=2 max_fails=3 fail_timeout=30s;
    server api-4.enbd.com:3000 weight=1 backup;  # Backup server
    
    # Session persistence (IP hash)
    # ip_hash;
    
    # Keep-alive connections
    keepalive 32;
    keepalive_requests 100;
    keepalive_timeout 60s;
}

server {
    listen 443 ssl http2;
    server_name api.enbd.com;
    
    location / {
        proxy_pass http://banking_api;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        
        # Headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # Timeouts
        proxy_connect_timeout 5s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
        
        # Health checks (Nginx Plus)
        health_check interval=10 fails=3 passes=2 uri=/health;
    }
}
`;

module.exports = {
  RoundRobinLoadBalancer,
  WeightedRoundRobinLoadBalancer,
  LeastConnectionsLoadBalancer,
  IPHashLoadBalancer,
  LoadBalancerWithHealthCheck,
  HorizontalScalingManager
};
```

---

### Q22. How do you optimize database queries for API performance?

**Answer:**

```javascript
/**
 * Database Query Optimization for Banking APIs
 */
const { Pool } = require('pg');

/**
 * Optimized Database Service
 */
class OptimizedDatabaseService {
  constructor() {
    // Connection pooling
    this.pool = new Pool({
      host: 'localhost',
      port: 5432,
      database: 'enbd_banking',
      user: 'postgres',
      password: 'password',
      max: 20, // Maximum pool size
      min: 5, // Minimum pool size
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 2000,
      maxUses: 7500, // Close connection after 7500 uses
      application_name: 'banking-api'
    });
    
    this.pool.on('error', (err) => {
      console.error('Unexpected pool error:', err);
    });
  }
  
  /**
   * Query with prepared statements (prevents SQL injection)
   */
  async query(text, params) {
    const start = Date.now();
    const result = await this.pool.query(text, params);
    const duration = Date.now() - start;
    
    // Log slow queries
    if (duration > 500) {
      console.warn(`Slow query (${duration}ms):`, text.substring(0, 100));
    }
    
    return result;
  }
  
  /**
   * Batch queries using transactions
   */
  async batchInsert(tableName, records) {
    const client = await this.pool.connect();
    
    try {
      await client.query('BEGIN');
      
      const promises = records.map(record => {
        const columns = Object.keys(record);
        const values = Object.values(record);
        const placeholders = columns.map((_, i) => `$${i + 1}`).join(', ');
        
        return client.query(
          `INSERT INTO ${tableName} (${columns.join(', ')}) VALUES (${placeholders})`,
          values
        );
      });
      
      await Promise.all(promises);
      await client.query('COMMIT');
      
      return { inserted: records.length };
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
  
  /**
   * Optimized pagination with cursor
   */
  async getTransactionsPaginated(accountId, cursor = null, limit = 20) {
    let query;
    let params;
    
    if (cursor) {
      // Cursor-based pagination (more efficient for large datasets)
      query = `
        SELECT id, amount, type, description, created_at
        FROM transactions
        WHERE account_id = $1 AND created_at < $2
        ORDER BY created_at DESC
        LIMIT $3
      `;
      params = [accountId, cursor, limit];
    } else {
      query = `
        SELECT id, amount, type, description, created_at
        FROM transactions
        WHERE account_id = $1
        ORDER BY created_at DESC
        LIMIT $2
      `;
      params = [accountId, limit];
    }
    
    const result = await this.query(query, params);
    
    return {
      data: result.rows,
      nextCursor: result.rows.length === limit
        ? result.rows[result.rows.length - 1].created_at
        : null
    };
  }
  
  /**
   * Optimized JOIN query
   */
  async getAccountWithCustomer(accountId) {
    // Single JOIN instead of multiple queries
    const query = `
      SELECT 
        a.id as account_id,
        a.account_number,
        a.balance,
        a.account_type,
        c.id as customer_id,
        c.name as customer_name,
        c.email as customer_email
      FROM accounts a
      INNER JOIN customers c ON a.customer_id = c.id
      WHERE a.id = $1
    `;
    
    const result = await this.query(query, [accountId]);
    
    if (result.rows.length === 0) return null;
    
    const row = result.rows[0];
    
    return {
      id: row.account_id,
      accountNumber: row.account_number,
      balance: row.balance,
      accountType: row.account_type,
      customer: {
        id: row.customer_id,
        name: row.customer_name,
        email: row.customer_email
      }
    };
  }
  
  /**
   * Bulk fetch using IN clause
   */
  async getAccountsByIds(accountIds) {
    // Use ANY instead of IN for better performance with arrays
    const query = `
      SELECT * FROM accounts
      WHERE id = ANY($1)
    `;
    
    const result = await this.query(query, [accountIds]);
    return result.rows;
  }
  
  /**
   * Aggregation query with indexes
   */
  async getAccountSummary(customerId) {
    const query = `
      SELECT 
        COUNT(*) as account_count,
        SUM(balance) as total_balance,
        AVG(balance) as average_balance,
        MAX(balance) as max_balance
      FROM accounts
      WHERE customer_id = $1 AND status = 'active'
    `;
    
    const result = await this.query(query, [customerId]);
    return result.rows[0];
  }
  
  /**
   * Query with materialized view (pre-computed)
   */
  async getCustomerDashboard(customerId) {
    // Use materialized view for complex aggregations
    const query = `
      SELECT * FROM customer_dashboard_view
      WHERE customer_id = $1
    `;
    
    const result = await this.query(query, [customerId]);
    return result.rows[0];
  }
  
  /**
   * Create indexes
   */
  async createIndexes() {
    const indexes = [
      // Single column indexes
      'CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_accounts_customer_id ON accounts(customer_id)',
      'CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_transactions_account_id ON transactions(account_id)',
      'CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_transactions_created_at ON transactions(created_at DESC)',
      
      // Composite indexes
      'CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_transactions_account_created ON transactions(account_id, created_at DESC)',
      'CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_accounts_customer_status ON accounts(customer_id, status)',
      
      // Partial indexes (for specific conditions)
      'CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_transactions_pending ON transactions(account_id) WHERE status = \'pending\'',
      
      // Expression indexes
      'CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_customers_email_lower ON customers(LOWER(email))',
      
      // BRIN index (for time-series data)
      'CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_transactions_created_brin ON transactions USING BRIN(created_at)'
    ];
    
    for (const indexQuery of indexes) {
      try {
        await this.query(indexQuery);
        console.log('Index created:', indexQuery.match(/idx_\w+/)?.[0]);
      } catch (error) {
        console.error('Failed to create index:', error.message);
      }
    }
  }
  
  /**
   * Create materialized view
   */
  async createMaterializedView() {
    await this.query(`
      CREATE MATERIALIZED VIEW IF NOT EXISTS customer_dashboard_view AS
      SELECT 
        c.id as customer_id,
        c.name,
        c.email,
        COUNT(DISTINCT a.id) as account_count,
        COALESCE(SUM(a.balance), 0) as total_balance,
        COUNT(DISTINCT t.id) FILTER (WHERE t.created_at > NOW() - INTERVAL '30 days') as recent_transaction_count,
        MAX(t.created_at) as last_transaction_at
      FROM customers c
      LEFT JOIN accounts a ON c.id = a.customer_id
      LEFT JOIN transactions t ON a.id = t.account_id
      GROUP BY c.id, c.name, c.email
      WITH DATA;
      
      CREATE UNIQUE INDEX ON customer_dashboard_view(customer_id);
    `);
    
    console.log('Materialized view created');
  }
  
  /**
   * Refresh materialized view
   */
  async refreshMaterializedView() {
    await this.query('REFRESH MATERIALIZED VIEW CONCURRENTLY customer_dashboard_view');
    console.log('Materialized view refreshed');
  }
  
  /**
   * Query execution plan analysis
   */
  async analyzeQuery(query, params) {
    const explainQuery = `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${query}`;
    const result = await this.query(explainQuery, params);
    
    const plan = result.rows[0]['QUERY PLAN'][0];
    
    return {
      executionTime: plan['Execution Time'],
      planningTime: plan['Planning Time'],
      totalCost: plan.Plan['Total Cost'],
      plan: plan.Plan
    };
  }
}

/**
 * Query Builder with Optimization
 */
class OptimizedQueryBuilder {
  constructor() {
    this.select = [];
    this.from = '';
    this.joins = [];
    this.where = [];
    this.orderBy = [];
    this.limit = null;
    this.offset = null;
    this.params = [];
  }
  
  selectFields(...fields) {
    this.select.push(...fields);
    return this;
  }
  
  fromTable(table) {
    this.from = table;
    return this;
  }
  
  innerJoin(table, condition) {
    this.joins.push(`INNER JOIN ${table} ON ${condition}`);
    return this;
  }
  
  leftJoin(table, condition) {
    this.joins.push(`LEFT JOIN ${table} ON ${condition}`);
    return this;
  }
  
  whereCondition(condition, value) {
    this.params.push(value);
    this.where.push(`${condition} $${this.params.length}`);
    return this;
  }
  
  orderByField(field, direction = 'ASC') {
    this.orderBy.push(`${field} ${direction}`);
    return this;
  }
  
  limitRows(limit) {
    this.limit = limit;
    return this;
  }
  
  offsetRows(offset) {
    this.offset = offset;
    return this;
  }
  
  build() {
    let query = `SELECT ${this.select.join(', ')} FROM ${this.from}`;
    
    if (this.joins.length > 0) {
      query += ' ' + this.joins.join(' ');
    }
    
    if (this.where.length > 0) {
      query += ' WHERE ' + this.where.join(' AND ');
    }
    
    if (this.orderBy.length > 0) {
      query += ' ORDER BY ' + this.orderBy.join(', ');
    }
    
    if (this.limit !== null) {
      query += ` LIMIT ${this.limit}`;
    }
    
    if (this.offset !== null) {
      query += ` OFFSET ${this.offset}`;
    }
    
    return { query, params: this.params };
  }
}

module.exports = {
  OptimizedDatabaseService,
  OptimizedQueryBuilder
};
```

---

### Q23. How do you implement CDN and static asset optimization?

**Answer:**

```javascript
/**
 * CDN and Static Asset Optimization
 */
const crypto = require('crypto');
const sharp = require('sharp');
const { S3Client, PutObjectCommand } = require('@aws-sdk/client-s3');
const { CloudFrontClient, CreateInvalidationCommand } = require('@aws-sdk/client-cloudfront');

/**
 * CDN Manager
 */
class CDNManager {
  constructor() {
    this.s3Client = new S3Client({ region: 'me-south-1' }); // Bahrain
    this.cloudFrontClient = new CloudFrontClient({ region: 'me-south-1' });
    
    this.cdnDomain = 'https://cdn.enbd.com';
    this.s3Bucket = 'enbd-banking-assets';
    this.distributionId = 'E1234567890ABC';
  }
  
  /**
   * Upload file to S3 with CDN cache headers
   */
  async uploadAsset(file, path, options = {}) {
    const {
      contentType = 'application/octet-stream',
      cacheControl = 'public, max-age=31536000', // 1 year
      metadata = {}
    } = options;
    
    // Generate file hash for cache busting
    const hash = this.generateHash(file);
    const extension = path.split('.').pop();
    const filename = `${hash}.${extension}`;
    
    const command = new PutObjectCommand({
      Bucket: this.s3Bucket,
      Key: `${path}/${filename}`,
      Body: file,
      ContentType: contentType,
      CacheControl: cacheControl,
      Metadata: {
        ...metadata,
        'upload-timestamp': Date.now().toString()
      },
      // Enable gzip compression
      ContentEncoding: 'gzip',
      // Set ACL
      ACL: 'public-read'
    });
    
    await this.s3Client.send(command);
    
    const cdnUrl = `${this.cdnDomain}/${path}/${filename}`;
    
    return {
      url: cdnUrl,
      hash,
      filename
    };
  }
  
  /**
   * Optimize and upload image
   */
  async uploadImage(imageBuffer, path, options = {}) {
    const {
      width = null,
      height = null,
      quality = 80,
      format = 'webp'
    } = options;
    
    let pipeline = sharp(imageBuffer);
    
    // Resize if dimensions provided
    if (width || height) {
      pipeline = pipeline.resize(width, height, {
        fit: 'inside',
        withoutEnlargement: true
      });
    }
    
    // Convert to WebP or specified format
    if (format === 'webp') {
      pipeline = pipeline.webp({ quality });
    } else if (format === 'jpeg') {
      pipeline = pipeline.jpeg({ quality });
    } else if (format === 'png') {
      pipeline = pipeline.png({ compressionLevel: 9 });
    }
    
    const optimizedBuffer = await pipeline.toBuffer();
    
    return this.uploadAsset(optimizedBuffer, path, {
      contentType: `image/${format}`,
      cacheControl: 'public, max-age=31536000, immutable'
    });
  }
  
  /**
   * Generate responsive image variants
   */
  async uploadResponsiveImages(imageBuffer, path) {
    const variants = [
      { width: 320, suffix: 'mobile' },
      { width: 768, suffix: 'tablet' },
      { width: 1024, suffix: 'desktop' },
      { width: 1920, suffix: 'large' }
    ];
    
    const uploadPromises = variants.map(async ({ width, suffix }) => {
      const result = await this.uploadImage(imageBuffer, `${path}/${suffix}`, {
        width,
        format: 'webp'
      });
      
      return {
        ...result,
        width,
        suffix
      };
    });
    
    const results = await Promise.all(uploadPromises);
    
    return {
      variants: results,
      srcset: results.map(r => `${r.url} ${r.width}w`).join(', ')
    };
  }
  
  /**
   * Invalidate CDN cache
   */
  async invalidateCache(paths) {
    const command = new CreateInvalidationCommand({
      DistributionId: this.distributionId,
      InvalidationBatch: {
        CallerReference: Date.now().toString(),
        Paths: {
          Quantity: paths.length,
          Items: paths
        }
      }
    });
    
    const response = await this.cloudFrontClient.send(command);
    
    console.log('Cache invalidation created:', response.Invalidation.Id);
    
    return response.Invalidation;
  }
  
  /**
   * Generate signed URL (for private content)
   */
  generateSignedUrl(path, expiresIn = 3600) {
    const expiration = Math.floor(Date.now() / 1000) + expiresIn;
    const policy = JSON.stringify({
      Statement: [{
        Resource: `${this.cdnDomain}/${path}`,
        Condition: {
          DateLessThan: { 'AWS:EpochTime': expiration }
        }
      }]
    });
    
    // Sign policy (simplified - use AWS SDK in production)
    const signature = crypto
      .createHmac('sha256', process.env.CLOUDFRONT_PRIVATE_KEY)
      .update(policy)
      .digest('base64');
    
    return `${this.cdnDomain}/${path}?Expires=${expiration}&Signature=${encodeURIComponent(signature)}`;
  }
  
  generateHash(data) {
    return crypto.createHash('sha256').update(data).digest('hex').substring(0, 16);
  }
}

/**
 * Asset Optimization Middleware
 */
class AssetOptimizationMiddleware {
  /**
   * Compress responses
   */
  static compression() {
    const compression = require('compression');
    
    return compression({
      filter: (req, res) => {
        if (req.headers['x-no-compression']) {
          return false;
        }
        return compression.filter(req, res);
      },
      level: 6,
      threshold: 1024 // Only compress files > 1KB
    });
  }
  
  /**
   * Static file caching
   */
  static cacheControl() {
    return (req, res, next) => {
      // Check file extension
      const extension = req.path.split('.').pop();
      
      const cacheSettings = {
        // Long cache for immutable assets
        'css': 'public, max-age=31536000, immutable',
        'js': 'public, max-age=31536000, immutable',
        'woff': 'public, max-age=31536000, immutable',
        'woff2': 'public, max-age=31536000, immutable',
        
        // Moderate cache for images
        'jpg': 'public, max-age=2592000', // 30 days
        'jpeg': 'public, max-age=2592000',
        'png': 'public, max-age=2592000',
        'webp': 'public, max-age=2592000',
        'svg': 'public, max-age=2592000',
        
        // Short cache for HTML
        'html': 'public, max-age=3600, must-revalidate', // 1 hour
        
        // No cache for dynamic content
        'json': 'no-cache, no-store, must-revalidate'
      };
      
      const cacheControl = cacheSettings[extension];
      
      if (cacheControl) {
        res.set('Cache-Control', cacheControl);
      }
      
      next();
    };
  }
  
  /**
   * ETag generation
   */
  static etag() {
    return (req, res, next) => {
      const originalSend = res.send.bind(res);
      
      res.send = (body) => {
        if (typeof body === 'string' || Buffer.isBuffer(body)) {
          const hash = crypto.createHash('md5').update(body).digest('hex');
          res.set('ETag', `"${hash}"`);
          
          // Check If-None-Match
          if (req.get('If-None-Match') === `"${hash}"`) {
            res.status(304).end();
            return res;
          }
        }
        
        return originalSend(body);
      };
      
      next();
    };
  }
}

/**
 * CloudFlare Configuration (Alternative CDN)
 */
const cloudflareConfig = {
  zones: {
    'api.enbd.com': {
      // Page Rules
      pageRules: [
        {
          targets: ['/static/*'],
          actions: {
            cacheLevel: 'cache_everything',
            edgeCacheTTL: 31536000, // 1 year
            browserCacheTTL: 31536000
          }
        },
        {
          targets: ['/api/*'],
          actions: {
            cacheLevel: 'bypass',
            disableApps: true,
            disablePerformance: false
          }
        }
      ],
      
      // Caching settings
      caching: {
        cachingLevel: 'aggressive',
        browserCacheTTL: 14400,
        developmentMode: false,
        queryStringSort: true,
        respectStrongEtags: true
      },
      
      // Performance
      performance: {
        minify: {
          js: true,
          css: true,
          html: true
        },
        brotli: true,
        http2: true,
        http3: true,
        earlyHints: true,
        rocketLoader: false
      },
      
      // Security
      security: {
        waf: true,
        ddos: true,
        botManagement: true,
        ssl: 'full_strict'
      }
    }
  }
};

/**
 * Express Setup with CDN
 */
const express = require('express');
const app = express();

// Apply middleware
app.use(AssetOptimizationMiddleware.compression());
app.use(AssetOptimizationMiddleware.cacheControl());
app.use(AssetOptimizationMiddleware.etag());

// Static files with CDN URL helper
app.locals.cdn = (path) => {
  if (process.env.NODE_ENV === 'production') {
    return `https://cdn.enbd.com${path}`;
  }
  return path;
};

// Example route
app.get('/profile', (req, res) => {
  res.send(`
    <!DOCTYPE html>
    <html>
    <head>
      <link rel="stylesheet" href="${app.locals.cdn('/css/styles.css')}">
    </head>
    <body>
      <img srcset="${app.locals.cdn('/images/profile-mobile.webp')} 320w,
                   ${app.locals.cdn('/images/profile-tablet.webp')} 768w,
                   ${app.locals.cdn('/images/profile-desktop.webp')} 1024w"
           sizes="(max-width: 320px) 320px,
                  (max-width: 768px) 768px,
                  1024px"
           src="${app.locals.cdn('/images/profile-desktop.webp')}"
           alt="Profile">
      <script src="${app.locals.cdn('/js/app.js')}"></script>
    </body>
    </html>
  `);
});

module.exports = {
  CDNManager,
  AssetOptimizationMiddleware,
  cloudflareConfig
};
```

---

**Summary Q21-Q23:**
- Load balancing strategies (Round Robin, Least Connections, IP Hash) with health checks & HPA ✅
- Database query optimization (connection pooling, indexes, materialized views, query analysis) ✅
- CDN integration (S3/CloudFront/CloudFlare) with image optimization & cache management ✅
