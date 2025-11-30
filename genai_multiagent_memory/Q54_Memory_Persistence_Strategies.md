# Q54: Memory Persistence Strategies

## Question: How do you implement persistent memory storage in production multi-agent systems?

---

## Answer:

Memory persistence ensures that agent state, conversation history, and learned patterns survive system restarts and scale across distributed deployments. A robust persistence strategy uses multiple storage backends optimized for different data types and access patterns.

---

## Table of Contents

1. [Persistence Architecture](#persistence-architecture)
2. [Redis Implementation](#redis-implementation)
3. [Pinecone Vector Store](#pinecone-vector-store)
4. [PostgreSQL Entity Storage](#postgresql-entity-storage)
5. [LangGraph Checkpointing](#langgraph-checkpointing)
6. [Cross-Session Memory](#cross-session-memory)
7. [Production Best Practices](#production-best-practices)

---

## 1. Persistence Architecture

### Three-Tier Persistence Strategy

```
┌──────────────────────────────────────────────────────────────┐
│          PRODUCTION PERSISTENCE ARCHITECTURE                 │
└──────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  TIER 1: REDIS - Short-Term Cache                           │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│                                                              │
│  Configuration:                                              │
│  • Persistence: AOF (Append-Only File) + RDB Snapshot       │
│  • Replication: Master-Replica (2 replicas)                 │
│  • Backup: Hourly snapshots to S3                           │
│  • TTL: 24 hours for QA, 1 hour for banking sessions        │
│                                                              │
│  Use Cases:                                                  │
│  ✓ Current test executions                                  │
│  ✓ Active conversation turns                                │
│  ✓ Session state                                            │
│  ✓ Temporary results cache                                  │
│                                                              │
│  Recovery: < 1 minute (from replica failover)               │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  TIER 2: PINECONE - Long-Term Vector Storage                │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│                                                              │
│  Configuration:                                              │
│  • Index Type: Serverless (auto-scaling)                    │
│  • Dimensions: 1536 (OpenAI ada-002)                        │
│  • Metric: Cosine similarity                                │
│  • Replicas: 3 (built-in redundancy)                        │
│  • Backup: Automatic (managed by Pinecone)                  │
│                                                              │
│  Use Cases:                                                  │
│  ✓ SOP document embeddings                                  │
│  ✓ Similar ticket search                                    │
│  ✓ Conversation summaries                                   │
│  ✓ Fraud pattern detection                                  │
│                                                              │
│  Recovery: N/A (fully managed, 99.9% SLA)                   │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  TIER 3: POSTGRESQL - Entity & Relational Data              │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│                                                              │
│  Configuration:                                              │
│  • Version: PostgreSQL 15                                   │
│  • Replication: Primary + 2 Read Replicas                   │
│  • Backup: Daily full + 5-minute WAL archiving              │
│  • Storage: 1TB SSD with auto-scaling                       │
│  • Connection Pool: PgBouncer (500 connections)             │
│                                                              │
│  Use Cases:                                                  │
│  ✓ Team preferences & configurations                        │
│  ✓ Test execution histories                                 │
│  ✓ Customer profiles & relationships                        │
│  ✓ Audit logs & compliance data                             │
│                                                              │
│  Recovery: < 5 minutes (from read replica promotion)        │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Redis Implementation

### Production Redis Setup

```typescript
import Redis from 'ioredis';
import { promisify } from 'util';

/**
 * Production Redis Manager with Clustering
 */
class ProductionRedisManager {
  private redis: Redis.Cluster | Redis;
  private isCluster: boolean;
  
  constructor(clusterMode: boolean = true) {
    this.isCluster = clusterMode;
    
    if (clusterMode) {
      // Cluster mode for high availability
      this.redis = new Redis.Cluster(
        [
          { host: process.env.REDIS_HOST_1, port: 6379 },
          { host: process.env.REDIS_HOST_2, port: 6379 },
          { host: process.env.REDIS_HOST_3, port: 6379 },
        ],
        {
          redisOptions: {
            password: process.env.REDIS_PASSWORD,
            tls: process.env.NODE_ENV === 'production' ? {} : undefined,
          },
          clusterRetryStrategy: (times) => {
            const delay = Math.min(100 * Math.pow(2, times), 2000);
            return delay;
          },
        }
      );
    } else {
      // Standalone with Sentinel for failover
      this.redis = new Redis({
        sentinels: [
          { host: process.env.SENTINEL_HOST_1, port: 26379 },
          { host: process.env.SENTINEL_HOST_2, port: 26379 },
          { host: process.env.SENTINEL_HOST_3, port: 26379 },
        ],
        name: 'mymaster',
        password: process.env.REDIS_PASSWORD,
      });
    }
    
    this.setupEventHandlers();
  }
  
  private setupEventHandlers() {
    this.redis.on('connect', () => {
      console.log('[Redis] Connected successfully');
    });
    
    this.redis.on('error', (err) => {
      console.error('[Redis] Error:', err);
    });
    
    this.redis.on('close', () => {
      console.warn('[Redis] Connection closed');
    });
    
    if (this.isCluster) {
      (this.redis as Redis.Cluster).on('node error', (err, node) => {
        console.error(`[Redis Cluster] Node error ${node}:`, err);
      });
    }
  }
  
  /**
   * Save with automatic serialization and TTL
   */
  async save(key: string, value: any, ttlSeconds?: number): Promise<void> {
    const serialized = JSON.stringify(value);
    
    if (ttlSeconds) {
      await this.redis.setex(key, ttlSeconds, serialized);
    } else {
      await this.redis.set(key, serialized);
    }
  }
  
  /**
   * Get with automatic deserialization
   */
  async get<T>(key: string): Promise<T | null> {
    const data = await this.redis.get(key);
    return data ? JSON.parse(data) : null;
  }
  
  /**
   * Batch save for performance
   */
  async saveBatch(items: Array<{ key: string; value: any; ttl?: number }>) {
    const pipeline = this.redis.pipeline();
    
    for (const item of items) {
      const serialized = JSON.stringify(item.value);
      if (item.ttl) {
        pipeline.setex(item.key, item.ttl, serialized);
      } else {
        pipeline.set(item.key, serialized);
      }
    }
    
    await pipeline.exec();
  }
  
  /**
   * Backup to S3
   */
  async backupToS3() {
    // Trigger Redis BGSAVE
    await this.redis.bgsave();
    
    // In production, use AWS CLI or SDK to copy RDB file to S3
    console.log('[Redis] Background save initiated for S3 backup');
  }
  
  /**
   * Health check
   */
  async healthCheck(): Promise<boolean> {
    try {
      const result = await this.redis.ping();
      return result === 'PONG';
    } catch (error) {
      console.error('[Redis] Health check failed:', error);
      return false;
    }
  }
  
  /**
   * Get memory usage stats
   */
  async getMemoryStats(): Promise<any> {
    const info = await this.redis.info('memory');
    const lines = info.split('\r\n');
    const stats: any = {};
    
    for (const line of lines) {
      const [key, value] = line.split(':');
      if (key && value) {
        stats[key] = value;
      }
    }
    
    return {
      usedMemory: stats.used_memory_human,
      peakMemory: stats.used_memory_peak_human,
      fragmentationRatio: parseFloat(stats.mem_fragmentation_ratio),
    };
  }
}

/**
 * QA Example: Test Execution State Persistence
 */
class QARedisStorage {
  private redis: ProductionRedisManager;
  
  constructor() {
    this.redis = new ProductionRedisManager();
  }
  
  /**
   * Save test execution state (24-hour TTL)
   */
  async saveTestState(ticketId: string, state: any) {
    await this.redis.save(`qa:test:${ticketId}`, state, 86400);
  }
  
  /**
   * Save test results with structured data
   */
  async saveTestResults(ticketId: string, results: any) {
    const key = `qa:results:${ticketId}`;
    
    // Save full results
    await this.redis.save(key, results, 86400);
    
    // Also save to sorted set for quick retrieval by score
    await this.redis['redis'].zadd(
      'qa:results:index',
      results.timestamp,
      ticketId
    );
  }
  
  /**
   * Get recent test results (last 24 hours)
   */
  async getRecentTestResults(limit: number = 50): Promise<any[]> {
    const now = Date.now();
    const oneDayAgo = now - 86400000;
    
    // Get ticket IDs from sorted set
    const ticketIds = await this.redis['redis'].zrevrangebyscore(
      'qa:results:index',
      now,
      oneDayAgo,
      'LIMIT',
      0,
      limit
    );
    
    // Batch get results
    const pipeline = this.redis['redis'].pipeline();
    for (const ticketId of ticketIds) {
      pipeline.get(`qa:results:${ticketId}`);
    }
    
    const results = await pipeline.exec();
    return results
      ?.map((r: any) => (r[1] ? JSON.parse(r[1]) : null))
      .filter((r) => r !== null) || [];
  }
  
  /**
   * Backup strategy
   */
  async scheduledBackup() {
    // Check memory usage first
    const stats = await this.redis.getMemoryStats();
    console.log('[QA Redis] Memory usage:', stats.usedMemory);
    
    // Backup to S3
    await this.redis.backupToS3();
    
    // Cleanup old entries
    await this.cleanupOldEntries();
  }
  
  private async cleanupOldEntries() {
    const threeDaysAgo = Date.now() - 259200000; // 3 days
    
    // Remove old entries from sorted set
    await this.redis['redis'].zremrangebyscore(
      'qa:results:index',
      0,
      threeDaysAgo
    );
  }
}

/**
 * Banking Example: Session Persistence
 */
class BankingSessionStorage {
  private redis: ProductionRedisManager;
  
  constructor() {
    this.redis = new ProductionRedisManager();
  }
  
  /**
   * Save customer session (1-hour TTL)
   */
  async saveSession(customerId: string, session: any) {
    await this.redis.save(`banking:session:${customerId}`, session, 3600);
    
    // Also track active sessions
    await this.redis['redis'].sadd('banking:active_sessions', customerId);
  }
  
  /**
   * Get session
   */
  async getSession(customerId: string): Promise<any | null> {
    return await this.redis.get(`banking:session:${customerId}`);
  }
  
  /**
   * Extend session TTL
   */
  async extendSession(customerId: string) {
    const key = `banking:session:${customerId}`;
    await this.redis['redis'].expire(key, 3600);
  }
  
  /**
   * End session
   */
  async endSession(customerId: string) {
    await this.redis['redis'].del(`banking:session:${customerId}`);
    await this.redis['redis'].srem('banking:active_sessions', customerId);
  }
  
  /**
   * Get active sessions count
   */
  async getActiveSessionsCount(): Promise<number> {
    return await this.redis['redis'].scard('banking:active_sessions');
  }
}
```

---

## 3. Pinecone Vector Store

### Production Pinecone Setup

```typescript
import { Pinecone } from '@pinecone-database/pinecone';
import { OpenAIEmbeddings } from '@langchain/openai';

/**
 * Production Pinecone Manager
 */
class ProductionPineconeManager {
  private pinecone: Pinecone;
  private embeddings: OpenAIEmbeddings;
  
  constructor() {
    this.pinecone = new Pinecone({
      apiKey: process.env.PINECONE_API_KEY!,
    });
    
    this.embeddings = new OpenAIEmbeddings({
      openAIApiKey: process.env.OPENAI_API_KEY,
      modelName: 'text-embedding-ada-002', // 1536 dimensions
    });
  }
  
  /**
   * Initialize index with proper configuration
   */
  async initializeIndex(indexName: string) {
    const existingIndexes = await this.pinecone.listIndexes();
    
    if (!existingIndexes.indexes?.find((idx) => idx.name === indexName)) {
      await this.pinecone.createIndex({
        name: indexName,
        dimension: 1536,
        metric: 'cosine',
        spec: {
          serverless: {
            cloud: 'aws',
            region: 'us-east-1',
          },
        },
      });
      
      console.log(`[Pinecone] Created index: ${indexName}`);
      
      // Wait for index to be ready
      await this.waitForIndexReady(indexName);
    }
  }
  
  private async waitForIndexReady(indexName: string, maxWaitMs = 60000) {
    const startTime = Date.now();
    
    while (Date.now() - startTime < maxWaitMs) {
      const description = await this.pinecone.describeIndex(indexName);
      if (description.status?.ready) {
        console.log(`[Pinecone] Index ${indexName} is ready`);
        return;
      }
      await new Promise((resolve) => setTimeout(resolve, 2000));
    }
    
    throw new Error(`Index ${indexName} not ready after ${maxWaitMs}ms`);
  }
  
  /**
   * Upsert with batching for performance
   */
  async upsertBatch(
    indexName: string,
    documents: Array<{ id: string; content: string; metadata: any }>
  ) {
    const index = this.pinecone.index(indexName);
    const batchSize = 100; // Pinecone recommendation
    
    for (let i = 0; i < documents.length; i += batchSize) {
      const batch = documents.slice(i, i + batchSize);
      
      // Generate embeddings in parallel
      const embeddings = await Promise.all(
        batch.map((doc) => this.embeddings.embedQuery(doc.content))
      );
      
      // Upsert batch
      await index.upsert(
        batch.map((doc, idx) => ({
          id: doc.id,
          values: embeddings[idx],
          metadata: {
            content: doc.content,
            ...doc.metadata,
            upsertedAt: new Date().toISOString(),
          },
        }))
      );
      
      console.log(`[Pinecone] Upserted batch ${i / batchSize + 1}`);
    }
  }
  
  /**
   * Query with retry logic
   */
  async queryWithRetry(
    indexName: string,
    query: string,
    topK: number = 10,
    maxRetries: number = 3
  ): Promise<any[]> {
    const index = this.pinecone.index(indexName);
    
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        const queryEmbedding = await this.embeddings.embedQuery(query);
        
        const results = await index.query({
          vector: queryEmbedding,
          topK,
          includeMetadata: true,
        });
        
        return (
          results.matches?.map((match) => ({
            id: match.id,
            score: match.score,
            content: match.metadata?.content,
            metadata: match.metadata,
          })) || []
        );
      } catch (error) {
        console.error(`[Pinecone] Query attempt ${attempt} failed:`, error);
        
        if (attempt === maxRetries) throw error;
        
        // Exponential backoff
        await new Promise((resolve) =>
          setTimeout(resolve, 1000 * Math.pow(2, attempt))
        );
      }
    }
    
    return [];
  }
  
  /**
   * Get index statistics
   */
  async getIndexStats(indexName: string) {
    const index = this.pinecone.index(indexName);
    const stats = await index.describeIndexStats();
    
    return {
      totalVectorCount: stats.totalRecordCount,
      dimension: stats.dimension,
      indexFullness: stats.indexFullness,
    };
  }
  
  /**
   * Delete vectors by metadata filter
   */
  async deleteByMetadata(indexName: string, filter: any) {
    const index = this.pinecone.index(indexName);
    await index.deleteMany({ filter });
    console.log(`[Pinecone] Deleted vectors matching filter`);
  }
}

/**
 * QA Example: SOP Document Storage
 */
class QASopVectorStorage {
  private pinecone: ProductionPineconeManager;
  private indexName = 'sop-documents';
  
  constructor() {
    this.pinecone = new ProductionPineconeManager();
  }
  
  async initialize() {
    await this.pinecone.initializeIndex(this.indexName);
  }
  
  /**
   * Store SOP documents
   */
  async storeSops(sops: Array<{ id: string; content: string; category: string }>) {
    await this.pinecone.upsertBatch(
      this.indexName,
      sops.map((sop) => ({
        id: sop.id,
        content: sop.content,
        metadata: {
          category: sop.category,
          version: '1.0',
          department: 'QA',
        },
      }))
    );
  }
  
  /**
   * Find relevant SOPs
   */
  async findRelevantSops(
    ticketDescription: string,
    category?: string
  ): Promise<any[]> {
    const results = await this.pinecone.queryWithRetry(
      this.indexName,
      ticketDescription,
      5
    );
    
    // Filter by category if provided
    if (category) {
      return results.filter((r) => r.metadata?.category === category);
    }
    
    return results;
  }
  
  /**
   * Update SOP version
   */
  async updateSopVersion(sopId: string, newContent: string, version: string) {
    await this.pinecone.upsertBatch(this.indexName, [
      {
        id: sopId,
        content: newContent,
        metadata: {
          version,
          updatedAt: new Date().toISOString(),
        },
      },
    ]);
  }
  
  /**
   * Get storage statistics
   */
  async getStorageStats() {
    return await this.pinecone.getIndexStats(this.indexName);
  }
}

/**
 * Banking Example: Transaction Pattern Storage
 */
class BankingTransactionVectorStorage {
  private pinecone: ProductionPineconeManager;
  private indexName = 'transaction-patterns';
  
  constructor() {
    this.pinecone = new ProductionPineconeManager();
  }
  
  async initialize() {
    await this.pinecone.initializeIndex(this.indexName);
  }
  
  /**
   * Store transaction patterns
   */
  async storeTransactionPattern(
    patternId: string,
    pattern: string,
    isFraud: boolean
  ) {
    await this.pinecone.upsertBatch(this.indexName, [
      {
        id: patternId,
        content: pattern,
        metadata: {
          isFraud,
          detectedAt: new Date().toISOString(),
        },
      },
    ]);
  }
  
  /**
   * Detect similar fraud patterns
   */
  async detectSimilarFraudPatterns(currentPattern: string): Promise<any[]> {
    const results = await this.pinecone.queryWithRetry(
      this.indexName,
      currentPattern,
      10
    );
    
    // Filter only fraud patterns
    return results.filter((r) => r.metadata?.isFraud === true);
  }
  
  /**
   * Cleanup old patterns (older than 1 year)
   */
  async cleanupOldPatterns() {
    const oneYearAgo = new Date();
    oneYearAgo.setFullYear(oneYearAgo.getFullYear() - 1);
    
    await this.pinecone.deleteByMetadata(this.indexName, {
      detectedAt: { $lt: oneYearAgo.toISOString() },
    });
  }
}
```

---

## 4. PostgreSQL Entity Storage

### Production PostgreSQL Setup

```typescript
import { Pool, PoolClient } from 'pg';

/**
 * Production PostgreSQL Manager
 */
class ProductionPostgresManager {
  private pool: Pool;
  
  constructor() {
    this.pool = new Pool({
      host: process.env.POSTGRES_HOST,
      port: parseInt(process.env.POSTGRES_PORT || '5432'),
      database: process.env.POSTGRES_DB,
      user: process.env.POSTGRES_USER,
      password: process.env.POSTGRES_PASSWORD,
      max: 100, // Connection pool size
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 10000,
      ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false,
    });
    
    this.setupEventHandlers();
  }
  
  private setupEventHandlers() {
    this.pool.on('connect', () => {
      console.log('[PostgreSQL] New client connected');
    });
    
    this.pool.on('error', (err) => {
      console.error('[PostgreSQL] Pool error:', err);
    });
  }
  
  /**
   * Initialize schema with proper indexes
   */
  async initializeSchema() {
    const client = await this.pool.connect();
    
    try {
      await client.query('BEGIN');
      
      // Test histories table
      await client.query(`
        CREATE TABLE IF NOT EXISTS test_histories (
          id SERIAL PRIMARY KEY,
          ticket_id VARCHAR(50) NOT NULL,
          test_plan JSONB NOT NULL,
          test_results JSONB NOT NULL,
          executed_at TIMESTAMP DEFAULT NOW(),
          execution_time_ms INTEGER,
          created_at TIMESTAMP DEFAULT NOW()
        );
        
        CREATE INDEX IF NOT EXISTS idx_test_histories_ticket_id ON test_histories(ticket_id);
        CREATE INDEX IF NOT EXISTS idx_test_histories_executed_at ON test_histories(executed_at DESC);
      `);
      
      // Team preferences table
      await client.query(`
        CREATE TABLE IF NOT EXISTS team_preferences (
          team_id VARCHAR(50) PRIMARY KEY,
          preferences JSONB NOT NULL,
          created_at TIMESTAMP DEFAULT NOW(),
          updated_at TIMESTAMP DEFAULT NOW()
        );
      `);
      
      // Project SOPs table
      await client.query(`
        CREATE TABLE IF NOT EXISTS project_sops (
          project_id VARCHAR(50) PRIMARY KEY,
          sop_ids TEXT[] NOT NULL,
          sop_metadata JSONB,
          created_at TIMESTAMP DEFAULT NOW(),
          updated_at TIMESTAMP DEFAULT NOW()
        );
      `);
      
      // Audit log table
      await client.query(`
        CREATE TABLE IF NOT EXISTS audit_logs (
          id SERIAL PRIMARY KEY,
          entity_type VARCHAR(50) NOT NULL,
          entity_id VARCHAR(100) NOT NULL,
          action VARCHAR(20) NOT NULL,
          changes JSONB,
          user_id VARCHAR(50),
          timestamp TIMESTAMP DEFAULT NOW()
        );
        
        CREATE INDEX IF NOT EXISTS idx_audit_logs_entity ON audit_logs(entity_type, entity_id);
        CREATE INDEX IF NOT EXISTS idx_audit_logs_timestamp ON audit_logs(timestamp DESC);
      `);
      
      await client.query('COMMIT');
      console.log('[PostgreSQL] Schema initialized successfully');
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
  
  /**
   * Execute query with connection pooling
   */
  async query(sql: string, params?: any[]): Promise<any> {
    const startTime = Date.now();
    const result = await this.pool.query(sql, params);
    const duration = Date.now() - startTime;
    
    if (duration > 1000) {
      console.warn(`[PostgreSQL] Slow query (${duration}ms): ${sql.substring(0, 100)}...`);
    }
    
    return result;
  }
  
  /**
   * Execute transaction
   */
  async transaction(callback: (client: PoolClient) => Promise<any>): Promise<any> {
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
  
  /**
   * Health check
   */
  async healthCheck(): Promise<boolean> {
    try {
      await this.query('SELECT 1');
      return true;
    } catch (error) {
      console.error('[PostgreSQL] Health check failed:', error);
      return false;
    }
  }
  
  /**
   * Get connection pool stats
   */
  getPoolStats() {
    return {
      totalCount: this.pool.totalCount,
      idleCount: this.pool.idleCount,
      waitingCount: this.pool.waitingCount,
    };
  }
  
  /**
   * Backup database
   */
  async backup(outputPath: string) {
    // In production, use pg_dump via child_process
    console.log(`[PostgreSQL] Would backup to: ${outputPath}`);
    // exec(`pg_dump -h ${host} -U ${user} -d ${db} -F c -f ${outputPath}`);
  }
}

/**
 * QA Example: Test History Storage
 */
class QATestHistoryStorage {
  private postgres: ProductionPostgresManager;
  
  constructor() {
    this.postgres = new ProductionPostgresManager();
  }
  
  async initialize() {
    await this.postgres.initializeSchema();
  }
  
  /**
   * Save test execution
   */
  async saveTestExecution(
    ticketId: string,
    testPlan: any,
    testResults: any,
    executionTimeMs: number
  ) {
    await this.postgres.query(
      `INSERT INTO test_histories (ticket_id, test_plan, test_results, execution_time_ms)
       VALUES ($1, $2, $3, $4)`,
      [ticketId, JSON.stringify(testPlan), JSON.stringify(testResults), executionTimeMs]
    );
    
    // Log audit trail
    await this.logAudit('test_execution', ticketId, 'CREATE', {
      testCases: testPlan.testCases,
      passed: testResults.passed,
      failed: testResults.failed,
    });
  }
  
  /**
   * Get test history
   */
  async getTestHistory(ticketId: string, limit: number = 10): Promise<any[]> {
    const result = await this.postgres.query(
      `SELECT test_plan, test_results, executed_at, execution_time_ms
       FROM test_histories
       WHERE ticket_id = $1
       ORDER BY executed_at DESC
       LIMIT $2`,
      [ticketId, limit]
    );
    
    return result.rows;
  }
  
  /**
   * Get test statistics
   */
  async getTestStatistics(days: number = 30): Promise<any> {
    const result = await this.postgres.query(
      `SELECT 
         COUNT(*) as total_executions,
         AVG(execution_time_ms) as avg_execution_time,
         SUM((test_results->>'passed')::int) as total_passed,
         SUM((test_results->>'failed')::int) as total_failed
       FROM test_histories
       WHERE executed_at > NOW() - INTERVAL '${days} days'`
    );
    
    return result.rows[0];
  }
  
  /**
   * Save team preferences
   */
  async saveTeamPreferences(teamId: string, preferences: any) {
    await this.postgres.query(
      `INSERT INTO team_preferences (team_id, preferences)
       VALUES ($1, $2)
       ON CONFLICT (team_id)
       DO UPDATE SET preferences = $2, updated_at = NOW()`,
      [teamId, JSON.stringify(preferences)]
    );
    
    await this.logAudit('team_preferences', teamId, 'UPDATE', preferences);
  }
  
  /**
   * Get team preferences
   */
  async getTeamPreferences(teamId: string): Promise<any | null> {
    const result = await this.postgres.query(
      'SELECT preferences FROM team_preferences WHERE team_id = $1',
      [teamId]
    );
    
    return result.rows.length > 0 ? result.rows[0].preferences : null;
  }
  
  /**
   * Audit logging
   */
  private async logAudit(
    entityType: string,
    entityId: string,
    action: string,
    changes: any
  ) {
    await this.postgres.query(
      `INSERT INTO audit_logs (entity_type, entity_id, action, changes)
       VALUES ($1, $2, $3, $4)`,
      [entityType, entityId, action, JSON.stringify(changes)]
    );
  }
  
  /**
   * Get audit trail
   */
  async getAuditTrail(entityType: string, entityId: string): Promise<any[]> {
    const result = await this.postgres.query(
      `SELECT action, changes, timestamp
       FROM audit_logs
       WHERE entity_type = $1 AND entity_id = $2
       ORDER BY timestamp DESC
       LIMIT 100`,
      [entityType, entityId]
    );
    
    return result.rows;
  }
}

/**
 * Banking Example: Customer Profile Storage
 */
class BankingCustomerStorage {
  private postgres: ProductionPostgresManager;
  
  constructor() {
    this.postgres = new ProductionPostgresManager();
  }
  
  async initialize() {
    await this.postgres.initializeSchema();
    
    // Banking-specific tables
    const client = await this.postgres['pool'].connect();
    try {
      await client.query(`
        CREATE TABLE IF NOT EXISTS customer_profiles (
          customer_id VARCHAR(50) PRIMARY KEY,
          profile_data JSONB NOT NULL,
          risk_score DECIMAL(5,2),
          created_at TIMESTAMP DEFAULT NOW(),
          updated_at TIMESTAMP DEFAULT NOW()
        );
        
        CREATE INDEX IF NOT EXISTS idx_customer_risk ON customer_profiles(risk_score);
        
        CREATE TABLE IF NOT EXISTS loan_applications (
          application_id VARCHAR(50) PRIMARY KEY,
          customer_id VARCHAR(50) NOT NULL,
          loan_amount DECIMAL(15,2) NOT NULL,
          loan_type VARCHAR(50) NOT NULL,
          status VARCHAR(20) NOT NULL,
          application_data JSONB,
          created_at TIMESTAMP DEFAULT NOW(),
          updated_at TIMESTAMP DEFAULT NOW(),
          FOREIGN KEY (customer_id) REFERENCES customer_profiles(customer_id)
        );
        
        CREATE INDEX IF NOT EXISTS idx_loan_customer ON loan_applications(customer_id);
        CREATE INDEX IF NOT EXISTS idx_loan_status ON loan_applications(status);
      `);
    } finally {
      client.release();
    }
  }
  
  /**
   * Save customer profile
   */
  async saveCustomerProfile(customerId: string, profileData: any, riskScore: number) {
    await this.postgres.query(
      `INSERT INTO customer_profiles (customer_id, profile_data, risk_score)
       VALUES ($1, $2, $3)
       ON CONFLICT (customer_id)
       DO UPDATE SET profile_data = $2, risk_score = $3, updated_at = NOW()`,
      [customerId, JSON.stringify(profileData), riskScore]
    );
  }
  
  /**
   * Get customer profile
   */
  async getCustomerProfile(customerId: string): Promise<any | null> {
    const result = await this.postgres.query(
      'SELECT profile_data, risk_score FROM customer_profiles WHERE customer_id = $1',
      [customerId]
    );
    
    return result.rows.length > 0 ? result.rows[0] : null;
  }
  
  /**
   * Save loan application with transaction
   */
  async saveLoanApplication(applicationId: string, customerId: string, loanData: any) {
    await this.postgres.transaction(async (client) => {
      // Insert loan application
      await client.query(
        `INSERT INTO loan_applications (application_id, customer_id, loan_amount, loan_type, status, application_data)
         VALUES ($1, $2, $3, $4, $5, $6)`,
        [
          applicationId,
          customerId,
          loanData.amount,
          loanData.type,
          'PENDING',
          JSON.stringify(loanData),
        ]
      );
      
      // Update customer profile
      await client.query(
        `UPDATE customer_profiles
         SET profile_data = jsonb_set(profile_data, '{lastLoanApplicationDate}', $1::jsonb)
         WHERE customer_id = $2`,
        [JSON.stringify(new Date().toISOString()), customerId]
      );
    });
  }
  
  /**
   * Get customer loan history
   */
  async getCustomerLoanHistory(customerId: string): Promise<any[]> {
    const result = await this.postgres.query(
      `SELECT application_id, loan_amount, loan_type, status, created_at
       FROM loan_applications
       WHERE customer_id = $1
       ORDER BY created_at DESC`,
      [customerId]
    );
    
    return result.rows;
  }
}
```

---

## 5. LangGraph Checkpointing

### Workflow State Persistence

```typescript
import { MemorySaver, SqliteSaver } from '@langchain/langgraph';
import { StateGraph } from '@langchain/langgraph';
import Database from 'better-sqlite3';

/**
 * LangGraph Checkpointing for Workflow Resume
 */
class LangGraphCheckpointManager {
  private checkpointer: SqliteSaver;
  
  constructor(dbPath: string = './checkpoints.db') {
    const db = new Database(dbPath);
    this.checkpointer = new SqliteSaver(db);
  }
  
  /**
   * Create workflow with checkpointing
   */
  createWorkflowWithCheckpointing<T>(
    graphBuilder: StateGraph<T>
  ): StateGraph<T> {
    // Add checkpointing to graph
    return graphBuilder;
  }
  
  /**
   * Resume workflow from checkpoint
   */
  async resumeWorkflow(threadId: string, graphConfig: any) {
    // Get checkpoint
    const checkpoint = await this.checkpointer.get({
      configurable: { thread_id: threadId },
    });
    
    if (!checkpoint) {
      console.log(`[Checkpoint] No checkpoint found for thread ${threadId}`);
      return null;
    }
    
    console.log(`[Checkpoint] Resuming from thread ${threadId}`);
    return checkpoint;
  }
  
  /**
   * List all checkpoints
   */
  async listCheckpoints(): Promise<any[]> {
    const checkpoints = await this.checkpointer.list({ limit: 100 });
    return checkpoints;
  }
  
  /**
   * Delete old checkpoints
   */
  async cleanupOldCheckpoints(olderThanDays: number) {
    // Implementation would query and delete old checkpoints
    console.log(`[Checkpoint] Would cleanup checkpoints older than ${olderThanDays} days`);
  }
}

/**
 * QA Example: Resume Test Execution
 */
async function qaCheckpointExample() {
  const checkpointManager = new LangGraphCheckpointManager();
  
  // Define QA workflow state
  interface QAState {
    ticketId: string;
    testPlan?: any;
    testResults?: any;
    currentStep: string;
  }
  
  const workflow = new StateGraph<QAState>({
    channels: {
      ticketId: null,
      testPlan: null,
      testResults: null,
      currentStep: null,
    },
  });
  
  // Add nodes
  workflow.addNode('generate_test_plan', async (state) => {
    console.log('[QA] Generating test plan...');
    return { ...state, testPlan: { testCases: 10 }, currentStep: 'test_plan_generated' };
  });
  
  workflow.addNode('execute_tests', async (state) => {
    console.log('[QA] Executing tests...');
    return { ...state, testResults: { passed: 9, failed: 1 }, currentStep: 'tests_executed' };
  });
  
  // Add edges
  workflow.addEdge('generate_test_plan', 'execute_tests');
  
  workflow.setEntryPoint('generate_test_plan');
  workflow.setFinishPoint('execute_tests');
  
  // Compile with checkpointing
  const app = workflow.compile({ checkpointer: checkpointManager['checkpointer'] });
  
  // Run workflow with thread ID
  const threadId = 'qa-thread-1234';
  const result = await app.invoke(
    { ticketId: 'QA-1234', currentStep: 'start' },
    { configurable: { thread_id: threadId } }
  );
  
  console.log('[QA] Workflow result:', result);
  
  // Later, resume from checkpoint
  const checkpoint = await checkpointManager.resumeWorkflow(threadId, {});
  console.log('[QA] Resumed checkpoint:', checkpoint);
}
```

---

## 6. Cross-Session Memory

### Session Management with Persistent Memory

```typescript
/**
 * Cross-Session Memory Manager
 * Handles memory across multiple user sessions
 */
class CrossSessionMemoryManager {
  private redis: ProductionRedisManager;
  private postgres: ProductionPostgresManager;
  private pinecone: ProductionPineconeManager;
  
  constructor() {
    this.redis = new ProductionRedisManager();
    this.postgres = new ProductionPostgresManager();
    this.pinecone = new ProductionPineconeManager();
  }
  
  /**
   * Start new session with history
   */
  async startSession(userId: string, sessionId: string) {
    // Load user's recent sessions from PostgreSQL
    const recentSessions = await this.postgres.query(
      `SELECT session_id, session_data, created_at
       FROM user_sessions
       WHERE user_id = $1
       ORDER BY created_at DESC
       LIMIT 5`,
      [userId]
    );
    
    // Load relevant conversation history from Pinecone
    const relevantHistory = await this.pinecone.queryWithRetry(
      'conversation-history',
      `User ${userId} sessions`,
      10
    );
    
    // Save current session to Redis (short-term)
    await this.redis.save(
      `session:${sessionId}`,
      {
        userId,
        startedAt: new Date().toISOString(),
        recentSessions: recentSessions.rows,
        relevantHistory,
      },
      3600 // 1 hour
    );
    
    return { recentSessions: recentSessions.rows, relevantHistory };
  }
  
  /**
   * End session and persist to long-term storage
   */
  async endSession(sessionId: string, summary: string) {
    // Get session from Redis
    const session = await this.redis.get<any>(`session:${sessionId}`);
    
    if (!session) {
      console.warn(`[Session] Session ${sessionId} not found`);
      return;
    }
    
    // Save to PostgreSQL
    await this.postgres.query(
      `INSERT INTO user_sessions (session_id, user_id, session_data, summary)
       VALUES ($1, $2, $3, $4)`,
      [sessionId, session.userId, JSON.stringify(session), summary]
    );
    
    // Save summary to Pinecone for future retrieval
    await this.pinecone.upsertBatch('conversation-history', [
      {
        id: sessionId,
        content: summary,
        metadata: {
          userId: session.userId,
          sessionId,
          endedAt: new Date().toISOString(),
        },
      },
    ]);
    
    console.log(`[Session] Session ${sessionId} persisted to long-term storage`);
  }
  
  /**
   * Get user context across all sessions
   */
  async getUserContext(userId: string): Promise<any> {
    // Get from PostgreSQL
    const sessionsResult = await this.postgres.query(
      `SELECT COUNT(*) as total_sessions,
              MAX(created_at) as last_session
       FROM user_sessions
       WHERE user_id = $1`,
      [userId]
    );
    
    // Get from Pinecone
    const pastConversations = await this.pinecone.queryWithRetry(
      'conversation-history',
      `User ${userId} history`,
      20
    );
    
    return {
      totalSessions: sessionsResult.rows[0].total_sessions,
      lastSession: sessionsResult.rows[0].last_session,
      relevantPastConversations: pastConversations,
    };
  }
}

/**
 * QA Example: Resume Test Across Sessions
 */
class QACrossSessionManager {
  private sessionManager: CrossSessionMemoryManager;
  
  constructor() {
    this.sessionManager = new CrossSessionMemoryManager();
  }
  
  /**
   * Start QA session with historical context
   */
  async startQASession(teamId: string, sessionId: string) {
    const context = await this.sessionManager.startSession(teamId, sessionId);
    
    console.log(`[QA Session] Started with ${context.recentSessions.length} recent sessions`);
    console.log(`[QA Session] Found ${context.relevantHistory.length} relevant test histories`);
    
    return context;
  }
  
  /**
   * End QA session
   */
  async endQASession(sessionId: string, testsExecuted: number, summary: string) {
    await this.sessionManager.endSession(
      sessionId,
      `QA session: ${testsExecuted} tests executed. ${summary}`
    );
  }
}

/**
 * Banking Example: Customer Journey Across Sessions
 */
class BankingCrossSessionManager {
  private sessionManager: CrossSessionMemoryManager;
  
  constructor() {
    this.sessionManager = new CrossSessionMemoryManager();
  }
  
  /**
   * Start customer session with history
   */
  async startCustomerSession(customerId: string, sessionId: string) {
    const context = await this.sessionManager.startSession(customerId, sessionId);
    
    // Extract customer-specific insights
    const loanInquiries = context.relevantHistory.filter(
      (item: any) => item.metadata?.topic === 'loan'
    );
    
    console.log(`[Banking] Customer has ${loanInquiries.length} past loan inquiries`);
    
    return { ...context, loanInquiries };
  }
  
  /**
   * End customer session
   */
  async endCustomerSession(sessionId: string, summary: string) {
    await this.sessionManager.endSession(sessionId, `Banking: ${summary}`);
  }
}
```

---

## 7. Production Best Practices

### Monitoring and Maintenance

```typescript
/**
 * Production Monitoring and Maintenance
 */
class MemorySystemMonitor {
  private redis: ProductionRedisManager;
  private postgres: ProductionPostgresManager;
  private pinecone: ProductionPineconeManager;
  
  constructor() {
    this.redis = new ProductionRedisManager();
    this.postgres = new ProductionPostgresManager();
    this.pinecone = new ProductionPineconeManager();
  }
  
  /**
   * Health check all systems
   */
  async healthCheck(): Promise<any> {
    const [redisHealthy, postgresHealthy] = await Promise.all([
      this.redis.healthCheck(),
      this.postgres.healthCheck(),
    ]);
    
    return {
      redis: { healthy: redisHealthy, status: redisHealthy ? 'UP' : 'DOWN' },
      postgres: { healthy: postgresHealthy, status: postgresHealthy ? 'UP' : 'DOWN' },
      pinecone: { healthy: true, status: 'UP' }, // Pinecone is managed service
      overall: redisHealthy && postgresHealthy ? 'HEALTHY' : 'DEGRADED',
    };
  }
  
  /**
   * Get storage statistics
   */
  async getStorageStats(): Promise<any> {
    const [redisStats, postgresStats] = await Promise.all([
      this.redis.getMemoryStats(),
      this.postgres.getPoolStats(),
    ]);
    
    const pineconeStats = await this.pinecone.getIndexStats('sop-documents');
    
    return {
      redis: redisStats,
      postgres: postgresStats,
      pinecone: pineconeStats,
    };
  }
  
  /**
   * Scheduled maintenance
   */
  async scheduledMaintenance() {
    console.log('[Maintenance] Starting scheduled maintenance...');
    
    // 1. Backup Redis
    await this.redis.backupToS3();
    
    // 2. Backup PostgreSQL
    await this.postgres.backup('/backups/postgres');
    
    // 3. Cleanup old data
    await this.cleanupOldData();
    
    console.log('[Maintenance] Scheduled maintenance completed');
  }
  
  /**
   * Cleanup old data
   */
  private async cleanupOldData() {
    // Redis: Keys are auto-expired with TTL, but check for orphans
    const memoryStats = await this.redis.getMemoryStats();
    console.log(`[Cleanup] Redis memory: ${memoryStats.usedMemory}`);
    
    // PostgreSQL: Delete audit logs older than 1 year
    await this.postgres.query(
      `DELETE FROM audit_logs WHERE timestamp < NOW() - INTERVAL '365 days'`
    );
    
    // Pinecone: Cleanup handled separately by retention policies
  }
}

/**
 * Best Practices Summary
 */
const bestPractices = `
┌────────────────────────────────────────────────────────────┐
│         MEMORY PERSISTENCE BEST PRACTICES                  │
└────────────────────────────────────────────────────────────┘

✅ DO:
• Set appropriate TTLs for all temporary data
• Use connection pooling (Redis Cluster, PostgreSQL Pool)
• Implement retry logic with exponential backoff
• Batch operations for performance (Pinecone upserts)
• Monitor memory usage and set alerts
• Regular backups (hourly snapshots, daily full backups)
• Use read replicas for scaling reads
• Implement audit logging for compliance
• Use transactions for multi-step operations
• Index frequently queried fields

❌ DON'T:
• Store large objects (>1MB) in Redis
• Skip TTL for temporary data
• Use single-node Redis in production
• Store sensitive data without encryption
• Forget to close database connections
• Run queries without indexes
• Skip error handling and retries
• Store application state in memory only
• Ignore backup strategies
• Mix storage tiers incorrectly

TIER SELECTION:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Redis:      Current execution state, session cache, rate limiting
Pinecone:   Semantic search, similar patterns, embeddings
PostgreSQL: Structured entities, relationships, audit logs
S3:         Large files (screenshots, PDFs, videos)

BACKUP STRATEGY:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Redis:      Hourly snapshots to S3, AOF for point-in-time recovery
PostgreSQL: Daily full backup + 5-minute WAL archiving
Pinecone:   Managed by Pinecone (automatic backups)
S3:         Versioning enabled, lifecycle policies for old data

MONITORING:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Redis:      Memory usage, eviction rate, hit rate
PostgreSQL: Connection pool size, slow queries, replication lag
Pinecone:   Query latency, index size, API usage
Overall:    Health checks every 60 seconds, alerting on failures
`;

console.log(bestPractices);
```

---

## Summary

### Persistence Strategy Overview

| Tier | Technology | Use Case | TTL | Backup | Recovery |
|------|-----------|----------|-----|--------|----------|
| Short-Term | Redis Cluster | Current execution, sessions | 1h-24h | Hourly to S3 | < 1 min |
| Long-Term | Pinecone | Semantic search, embeddings | Permanent | Automatic | N/A (managed) |
| Entity | PostgreSQL | Structured data, relationships | Permanent | Daily + WAL | < 5 min |
| Files | AWS S3 | Screenshots, PDFs, logs | 7-90 days | Versioning | Instant |

### Key Takeaways

1. **Three-Tier Architecture**: Match storage backend to data type and access pattern
2. **High Availability**: Use clustering, replication, and automatic failover
3. **Backup Strategy**: Multiple backup layers (snapshots, WAL, versioning)
4. **Monitoring**: Health checks, performance metrics, alerting
5. **Maintenance**: Regular cleanup, optimization, and capacity planning

---

**Related Files:**
- `Q53_Conversational_Memory_Management.md` - Memory types and usage
- `Q55_Context_Window_Optimization.md` - Token management
- `QA_Automation_Scenario.md` - Complete implementation
- `Architecture_Diagrams.md` - Visual architecture

---

**Document Version**: 1.0  
**Last Updated**: November 2025  
**For**: GenAI Interview Preparation

---

*End of Memory Persistence Strategies Document*
