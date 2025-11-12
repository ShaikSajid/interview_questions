# Performance Optimization

## Question 25: Performance Tuning & Optimization

### 📋 Question Statement

Implement comprehensive performance optimization for Emirates NBD microservices including caching strategies (Redis, CloudFront), database optimization (indexing, connection pooling), Lambda tuning, and load testing.

---

### 🏗️ Performance Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│           EMIRATES NBD PERFORMANCE OPTIMIZATION ARCHITECTURE                │
└────────────────────────────────────────────────────────────────────────────┘

    CACHING LAYERS              DATABASE OPTIMIZATION
    ──────────────             ─────────────────────
    
    ┌──────────────┐           ┌──────────────┐
    │  CloudFront  │           │ Read Replicas│
    │     CDN      │           │  • PostgreSQL│
    │  • Static    │           │  • MySQL     │
    │    Assets    │           └──────────────┘
    └──────────────┘           
                               ┌──────────────┐
    ┌──────────────┐           │  Connection  │
    │ ElastiCache  │           │   Pooling    │
    │    Redis     │           │  • pg-pool   │
    │  • Session   │           │  • Sequelize │
    │  • API Cache │           └──────────────┘
    └──────────────┘           
                               ┌──────────────┐
    ┌──────────────┐           │   Indexes    │
    │ DAX (DynamoDB│           │  • B-tree    │
    │  Accelerator)│           │  • Hash      │
    │  • Sub-ms    │           │  • Composite │
    └──────────────┘           └──────────────┘
```

---

### 🚀 Redis Caching Implementation

```javascript
// services/shared/src/cache/redis-cache.js
const Redis = require('ioredis');

class RedisCache {
  constructor() {
    this.client = new Redis({
      host: process.env.REDIS_HOST,
      port: 6379,
      password: process.env.REDIS_PASSWORD,
      retryStrategy: (times) => Math.min(times * 50, 2000),
      maxRetriesPerRequest: 3
    });

    this.client.on('error', (err) => console.error('Redis Error:', err));
    this.client.on('connect', () => console.log('Redis Connected'));
  }

  async get(key) {
    try {
      const value = await this.client.get(key);
      return value ? JSON.parse(value) : null;
    } catch (error) {
      console.error(`Cache get error for key ${key}:`, error);
      return null;
    }
  }

  async set(key, value, ttlSeconds = 300) {
    try {
      await this.client.setex(key, ttlSeconds, JSON.stringify(value));
    } catch (error) {
      console.error(`Cache set error for key ${key}:`, error);
    }
  }

  async del(key) {
    try {
      await this.client.del(key);
    } catch (error) {
      console.error(`Cache delete error for key ${key}:`, error);
    }
  }

  async invalidatePattern(pattern) {
    try {
      const keys = await this.client.keys(pattern);
      if (keys.length > 0) {
        await this.client.del(...keys);
      }
    } catch (error) {
      console.error(`Cache invalidation error for pattern ${pattern}:`, error);
    }
  }

  // Cache-aside pattern
  async getOrSet(key, fetchFn, ttl = 300) {
    let value = await this.get(key);
    
    if (!value) {
      value = await fetchFn();
      await this.set(key, value, ttl);
    }
    
    return value;
  }

  async close() {
    await this.client.quit();
  }
}

module.exports = new RedisCache();
```

```javascript
// Usage in service
const cache = require('./cache/redis-cache');

async function getAccount(accountId) {
  const cacheKey = `account:${accountId}`;
  
  return cache.getOrSet(cacheKey, async () => {
    return await db.query('SELECT * FROM accounts WHERE account_id = $1', [accountId]);
  }, 600); // 10 minutes TTL
}

// Invalidate on update
async function updateAccount(accountId, data) {
  await db.query('UPDATE accounts SET ...', [data]);
  await cache.del(`account:${accountId}`);
}
```

### 💾 Database Connection Pooling

```javascript
// services/account-service/src/database/connection-pool.js
const { Pool } = require('pg');

class DatabasePool {
  constructor() {
    this.pool = new Pool({
      host: process.env.DB_HOST,
      port: 5432,
      database: process.env.DB_NAME,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      
      // Connection pool settings
      min: 2,
      max: 20,
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 2000,
      
      // Performance settings
      statement_timeout: 10000,
      query_timeout: 10000,
      
      // SSL for production
      ssl: process.env.NODE_ENV === 'production' ? {
        rejectUnauthorized: false
      } : false
    });

    // Monitor pool
    this.pool.on('connect', () => {
      console.log('New client connected to pool');
    });

    this.pool.on('error', (err) => {
      console.error('Unexpected pool error:', err);
    });
  }

  async query(text, params) {
    const start = Date.now();
    try {
      const result = await this.pool.query(text, params);
      const duration = Date.now() - start;
      
      // Log slow queries
      if (duration > 1000) {
        console.warn('Slow query detected:', { text, duration });
      }
      
      return result;
    } catch (error) {
      console.error('Query error:', error);
      throw error;
    }
  }

  async transaction(callback) {
    const client = await this.pool.connect();
    
    try {
      await client.query('BEGIN');
      const result = await callback(client);
      await client.query('COMMIT');
      return result;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  async getPoolStats() {
    return {
      total: this.pool.totalCount,
      idle: this.pool.idleCount,
      waiting: this.pool.waitingCount
    };
  }

  async close() {
    await this.pool.end();
  }
}

module.exports = new DatabasePool();
```

### 📊 Database Indexing Strategy

```sql
-- PostgreSQL Indexing for Performance

-- Account lookups by customer
CREATE INDEX idx_accounts_customer_id ON accounts(customer_id);
CREATE INDEX idx_accounts_status ON accounts(status) WHERE status = 'ACTIVE';

-- Composite index for common queries
CREATE INDEX idx_accounts_customer_type ON accounts(customer_id, account_type);

-- Transaction queries
CREATE INDEX idx_transactions_from_account_timestamp 
ON transactions(from_account, timestamp DESC);

CREATE INDEX idx_transactions_status_created 
ON transactions(status, created_at) 
WHERE status IN ('PENDING', 'PROCESSING');

-- Partial index for active loans
CREATE INDEX idx_loans_active ON loans(customer_id) 
WHERE status = 'ACTIVE';

-- GIN index for JSONB columns
CREATE INDEX idx_transactions_metadata ON transactions USING GIN(metadata);

-- Full-text search
CREATE INDEX idx_customers_search 
ON customers USING GIN(to_tsvector('english', full_name || ' ' || email));

-- Query to find missing indexes
SELECT schemaname, tablename, attname, n_distinct, correlation
FROM pg_stats
WHERE schemaname = 'public'
  AND n_distinct > 100
  AND correlation < 0.1
ORDER BY n_distinct DESC;

-- Query to find unused indexes
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE 'pg_%'
ORDER BY pg_relation_size(indexrelid) DESC;
```

### ⚡ Lambda Performance Optimization

```javascript
// Optimized Lambda with provisioned concurrency
const AWS = require('aws-sdk');

// Initialize outside handler (reuse connections)
const dynamodb = new AWS.DynamoDB.DocumentClient();
const secretsManager = new AWS.SecretsManager();

let cachedSecret = null;

// Warm-up connections
async function warmup() {
  if (!cachedSecret) {
    const data = await secretsManager.getSecretValue({
      SecretId: process.env.SECRET_NAME
    }).promise();
    cachedSecret = JSON.parse(data.SecretString);
  }
}

exports.handler = async (event) => {
  // Ensure connections are warmed
  await warmup();
  
  // Process records in parallel
  const promises = event.Records.map(async (record) => {
    return dynamodb.put({
      TableName: process.env.TABLE_NAME,
      Item: JSON.parse(record.body)
    }).promise();
  });
  
  await Promise.all(promises);
  
  return { statusCode: 200 };
};
```

```typescript
// CDK - Provisioned Concurrency
const lambdaFunction = new lambda.Function(this, 'OptimizedFunction', {
  runtime: lambda.Runtime.NODEJS_20_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda'),
  memorySize: 1024, // More memory = more CPU
  timeout: cdk.Duration.seconds(30),
  environment: {
    NODE_OPTIONS: '--enable-source-maps',
    AWS_NODEJS_CONNECTION_REUSE_ENABLED: '1'
  }
});

// Provisioned concurrency for consistent performance
const version = lambdaFunction.currentVersion;
const alias = new lambda.Alias(this, 'ProdAlias', {
  aliasName: 'prod',
  version,
  provisionedConcurrentExecutions: 5
});
```

### 🎓 Interview Discussion Points - Q25

**Q1: What caching strategies exist?**

**A**:
- **Cache-aside**: App checks cache first
- **Write-through**: Update cache on write
- **Write-behind**: Async cache update
- **Refresh-ahead**: Preload before expiry

**Q2: When should you use caching?**

**A**:
- **Read-heavy**: More reads than writes
- **Expensive**: Slow to compute/fetch
- **Consistent**: Data doesn't change often
- **Don't cache**: User-specific, sensitive data

**Q3: How do you optimize database performance?**

**A**:
- **Indexes**: On WHERE, JOIN, ORDER BY columns
- **Connection pooling**: Reuse connections
- **Read replicas**: Scale reads
- **Partitioning**: Split large tables

---

## Question 26: Load Testing & Benchmarking

### 📋 Question Statement

Implement load testing for Emirates NBD using Artillery, k6, and AWS distributed load testing. Include performance benchmarks, stress testing, and capacity planning.

---

### 🧪 Artillery Load Test

```yaml
# tests/load/artillery-config.yml
config:
  target: 'https://api.emiratesnbd.com'
  phases:
    - duration: 60
      arrivalRate: 10
      name: "Warm up"
    - duration: 300
      arrivalRate: 50
      name: "Sustained load"
    - duration: 120
      arrivalRate: 100
      name: "Spike"
  defaults:
    headers:
      Content-Type: 'application/json'
      Authorization: 'Bearer {{ $processEnvironment.API_TOKEN }}'

scenarios:
  - name: "Account Operations"
    weight: 40
    flow:
      - post:
          url: "/accounts"
          json:
            customerId: "{{ $randomString() }}"
            accountType: "SAVINGS"
            initialBalance: 1000
          capture:
            - json: "$.accountId"
              as: "accountId"
      - think: 2
      - get:
          url: "/accounts/{{ accountId }}"
      - think: 1
      - post:
          url: "/transactions"
          json:
            fromAccount: "{{ accountId }}"
            toAccount: "ACC-{{ $randomNumber(1000, 9999) }}"
            amount: "{{ $randomNumber(10, 1000) }}"

  - name: "Transaction Query"
    weight: 60
    flow:
      - get:
          url: "/transactions?limit=50"
      - think: 1
      - get:
          url: "/accounts/ACC-{{ $randomNumber(1000, 9999) }}/balance"
```

### 📊 k6 Load Testing Script

```javascript
// tests/load/k6-script.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '2m', target: 50 },   // Ramp up to 50 users
    { duration: '5m', target: 50 },   // Stay at 50 users
    { duration: '2m', target: 100 },  // Ramp up to 100 users
    { duration: '5m', target: 100 },  // Stay at 100 users
    { duration: '2m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% of requests < 500ms
    http_req_failed: ['rate<0.01'],    // Error rate < 1%
    errors: ['rate<0.1'],
  },
};

export default function () {
  const BASE_URL = 'https://api.emiratesnbd.com';
  
  // Create account
  let createRes = http.post(`${BASE_URL}/accounts`, JSON.stringify({
    customerId: `CUST-${Date.now()}`,
    accountType: 'SAVINGS',
    initialBalance: 1000
  }), {
    headers: { 'Content-Type': 'application/json' }
  });

  check(createRes, {
    'account created': (r) => r.status === 201,
    'has accountId': (r) => r.json('accountId') !== undefined
  }) || errorRate.add(1);

  const accountId = createRes.json('accountId');
  sleep(1);

  // Get account
  let getRes = http.get(`${BASE_URL}/accounts/${accountId}`);
  check(getRes, {
    'account retrieved': (r) => r.status === 200,
    'balance correct': (r) => r.json('balance') === 1000
  }) || errorRate.add(1);

  sleep(1);

  // Create transaction
  let txnRes = http.post(`${BASE_URL}/transactions`, JSON.stringify({
    fromAccount: accountId,
    toAccount: 'ACC-9999',
    amount: 100,
    type: 'TRANSFER'
  }), {
    headers: { 'Content-Type': 'application/json' }
  });

  check(txnRes, {
    'transaction created': (r) => r.status === 201
  }) || errorRate.add(1);

  sleep(2);
}
```

### 🔧 Performance Testing CDK Stack

```typescript
// infrastructure/cdk/load-testing-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as iam from 'aws-cdk-lib/aws-iam';

export class LoadTestingStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // ECS Cluster for distributed load testing
    const cluster = new ecs.Cluster(this, 'LoadTestCluster', {
      vpc: props.vpc,
      containerInsights: true
    });

    // Task definition for k6
    const taskDefinition = new ecs.FargateTaskDefinition(this, 'K6Task', {
      memoryLimitMiB: 2048,
      cpu: 1024
    });

    taskDefinition.addContainer('k6', {
      image: ecs.ContainerImage.fromRegistry('grafana/k6:latest'),
      command: ['run', '/scripts/load-test.js'],
      logging: ecs.LogDrivers.awsLogs({
        streamPrefix: 'k6-load-test'
      }),
      environment: {
        TARGET_URL: 'https://api.emiratesnbd.com'
      }
    });
  }
}
```

### 📈 Performance Benchmarking Results

```markdown
# Emirates NBD Performance Benchmarks

## Account Service Performance
- **Endpoint**: GET /accounts/:id
- **p50 latency**: 45ms
- **p95 latency**: 120ms
- **p99 latency**: 280ms
- **Throughput**: 1,200 req/sec
- **Error rate**: 0.05%

## Transaction Service Performance
- **Endpoint**: POST /transactions
- **p50 latency**: 180ms
- **p95 latency**: 450ms
- **p99 latency**: 850ms
- **Throughput**: 800 req/sec
- **Error rate**: 0.1%

## Caching Impact
- **Without cache**: 450ms average
- **With Redis**: 45ms average
- **Cache hit rate**: 85%
- **Performance improvement**: 10x faster

## Database Query Optimization
- **Before indexing**: 2.3s
- **After indexing**: 45ms
- **Improvement**: 51x faster

## Lambda Cold Start Analysis
- **Cold start**: 1,200ms
- **Warm start**: 15ms
- **Provisioned concurrency**: 18ms (consistent)
- **Memory impact**: 512MB: 25ms, 1024MB: 15ms, 2048MB: 12ms
```

### 🎯 Capacity Planning

```javascript
// tools/capacity-planner.js
class CapacityPlanner {
  constructor(config) {
    this.config = config;
  }

  calculateECSCapacity(requestsPerSecond, avgResponseTime) {
    // Calculate concurrent requests
    const concurrentRequests = (requestsPerSecond * avgResponseTime) / 1000;
    
    // Each ECS task can handle N concurrent requests
    const requestsPerTask = 100;
    
    // Calculate required tasks
    const requiredTasks = Math.ceil(concurrentRequests / requestsPerTask);
    
    // Add 30% buffer for spikes
    const bufferedTasks = Math.ceil(requiredTasks * 1.3);
    
    return {
      minTasks: Math.max(2, Math.floor(bufferedTasks * 0.5)),
      maxTasks: bufferedTasks * 2,
      desiredTasks: bufferedTasks
    };
  }

  calculateDatabaseConnections(ecsTaskCount, connectionsPerTask) {
    return {
      required: ecsTaskCount * connectionsPerTask,
      recommended: Math.ceil(ecsTaskCount * connectionsPerTask * 1.2)
    };
  }

  calculateCacheSize(dailyActiveUsers, avgCachePerUser) {
    // Cache size in MB
    const totalCacheMB = (dailyActiveUsers * avgCachePerUser) / (1024 * 1024);
    
    return {
      minimumMB: totalCacheMB,
      recommendedMB: Math.ceil(totalCacheMB * 1.5),
      redisInstanceType: this.selectRedisInstance(totalCacheMB * 1.5)
    };
  }

  selectRedisInstance(sizeMB) {
    if (sizeMB < 1000) return 'cache.t3.small (1.5GB)';
    if (sizeMB < 3000) return 'cache.t3.medium (3.1GB)';
    if (sizeMB < 6000) return 'cache.m5.large (6.4GB)';
    if (sizeMB < 12000) return 'cache.m5.xlarge (12.9GB)';
    return 'cache.r5.large (13.1GB)';
  }

  estimateCosts(infrastructure) {
    const costs = {
      ecs: infrastructure.ecsTaskCount * 0.04 * 730, // $0.04/hour
      rds: 0.35 * 730, // r5.large
      elasticache: 0.068 * 730, // cache.t3.medium
      alb: 0.0225 * 730 + (infrastructure.requestsPerMonth / 1000000) * 0.008,
      lambda: (infrastructure.lambdaInvocations / 1000000) * 0.20,
      total: 0
    };

    costs.total = Object.values(costs).reduce((a, b) => a + b, 0);
    return costs;
  }
}

// Usage
const planner = new CapacityPlanner({});

const capacity = planner.calculateECSCapacity(1000, 150); // 1000 req/s, 150ms avg
console.log('ECS Capacity:', capacity);
// Output: { minTasks: 2, maxTasks: 4, desiredTasks: 2 }

const cache = planner.calculateCacheSize(100000, 50 * 1024); // 100k users, 50KB each
console.log('Cache Size:', cache);
// Output: { minimumMB: 4882, recommendedMB: 7323, redisInstanceType: 'cache.m5.xlarge' }
```

### 🔍 Performance Monitoring Dashboard

```typescript
// monitoring/performance-dashboard.ts
import * as cloudwatch from 'aws-cdk-lib/aws-cloudwatch';

export function createPerformanceDashboard(stack: cdk.Stack) {
  const dashboard = new cloudwatch.Dashboard(stack, 'PerformanceDashboard', {
    dashboardName: 'Banking-Performance-Metrics'
  });

  // API Latency
  const apiLatency = new cloudwatch.GraphWidget({
    title: 'API Response Time (p50, p95, p99)',
    left: [
      new cloudwatch.Metric({
        namespace: 'BankingApplication',
        metricName: 'APILatency',
        statistic: 'p50'
      }),
      new cloudwatch.Metric({
        namespace: 'BankingApplication',
        metricName: 'APILatency',
        statistic: 'p95'
      }),
      new cloudwatch.Metric({
        namespace: 'BankingApplication',
        metricName: 'APILatency',
        statistic: 'p99'
      })
    ]
  });

  // Database Performance
  const dbPerformance = new cloudwatch.GraphWidget({
    title: 'Database Performance',
    left: [
      new cloudwatch.Metric({
        namespace: 'AWS/RDS',
        metricName: 'ReadLatency',
        statistic: 'Average'
      }),
      new cloudwatch.Metric({
        namespace: 'AWS/RDS',
        metricName: 'WriteLatency',
        statistic: 'Average'
      })
    ]
  });

  // Cache Hit Rate
  const cacheMetrics = new cloudwatch.GraphWidget({
    title: 'Cache Hit Rate',
    left: [
      new cloudwatch.Metric({
        namespace: 'BankingApplication',
        metricName: 'CacheHits',
        statistic: 'Sum'
      }),
      new cloudwatch.Metric({
        namespace: 'BankingApplication',
        metricName: 'CacheMisses',
        statistic: 'Sum'
      })
    ]
  });

  dashboard.addWidgets(apiLatency, dbPerformance, cacheMetrics);
}
```

### 🎓 Interview Discussion Points - Q26

**Q1: What metrics matter in load testing?**

**A**:
- **Response time**: p50, p95, p99 percentiles
- **Throughput**: Requests per second (RPS)
- **Error rate**: Percentage of failed requests
- **Concurrency**: Simultaneous users/connections
- **Resource utilization**: CPU, memory, network

**Q2: What is the difference between load, stress, and spike testing?**

**A**:
- **Load testing**: Normal expected load
- **Stress testing**: Beyond normal capacity to find breaking point
- **Spike testing**: Sudden increase in load
- **Soak testing**: Sustained load over time (memory leaks)

**Q3: How do you determine the right ECS task count?**

**A**:
- Calculate concurrent requests: (RPS × response_time_ms) / 1000
- Determine requests per task: Based on CPU/memory
- Add buffer: 30-50% for traffic spikes
- Formula: tasks = ceil((concurrent_requests / requests_per_task) × 1.3)

**Q4: What causes performance degradation?**

**A**:
- **N+1 queries**: Multiple database calls in loop
- **No caching**: Repeated expensive operations
- **Poor indexing**: Slow database queries
- **Memory leaks**: Gradual memory consumption
- **Blocking operations**: Synchronous I/O in async code

**Q5: How to optimize Lambda cold starts?**

**A**:
- **Provisioned concurrency**: Keep functions warm
- **Reduce package size**: Minimize dependencies
- **Increase memory**: More memory = more CPU
- **Connection reuse**: Initialize outside handler
- **Use ARM**: Graviton2 processors are faster

---

**End of File 13 - Performance Optimization**