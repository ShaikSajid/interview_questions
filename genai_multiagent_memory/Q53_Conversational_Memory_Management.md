# Q53: Conversational Memory Management

## Question: How do you manage conversational memory in multi-agent systems?

---

## Answer:

Conversational memory enables agents to maintain context across interactions, remember past conversations, and provide personalized responses. In multi-agent systems, memory management becomes crucial for coordination and continuity.

---

## Table of Contents

1. [Memory Types Overview](#memory-types-overview)
2. [Short-Term Memory](#short-term-memory)
3. [Long-Term Memory](#long-term-memory)
4. [Entity Memory](#entity-memory)
5. [QA Automation Memory](#qa-automation-memory)
6. [Banking Memory Examples](#banking-memory-examples)
7. [Memory Best Practices](#memory-best-practices)

---

## 1. Memory Types Overview

### Three-Tier Memory Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              THREE-TIER MEMORY ARCHITECTURE                 │
└─────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│  TIER 1: SHORT-TERM MEMORY (Redis)                         │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  Purpose: Recent interactions, current session context     │
│  Storage: In-memory (Redis)                                │
│  TTL: 24 hours                                             │
│  Speed: <5ms read/write                                    │
│                                                            │
│  Examples:                                                 │
│  • Recent test execution results                          │
│  • Current JIRA ticket context                            │
│  • Active conversation turns                              │
└────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────┐
│  TIER 2: LONG-TERM MEMORY (Vector DB - Pinecone)          │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  Purpose: Semantic search, similar conversations          │
│  Storage: Vector embeddings                               │
│  TTL: Permanent (until explicitly deleted)                │
│  Speed: 50-200ms semantic search                          │
│                                                            │
│  Examples:                                                 │
│  • SOP document embeddings                                │
│  • Similar JIRA ticket history                            │
│  • Past conversation summaries                            │
└────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────┐
│  TIER 3: ENTITY MEMORY (PostgreSQL)                        │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  Purpose: Structured entity data, relationships           │
│  Storage: Relational database                             │
│  TTL: Permanent with audit trail                          │
│  Speed: 10-50ms indexed queries                           │
│                                                            │
│  Examples:                                                 │
│  • Team preferences                                        │
│  • Project SOPs                                           │
│  • Customer profiles                                       │
│  • Test execution histories                               │
└────────────────────────────────────────────────────────────┘
```

---

## 2. Short-Term Memory

### ConversationBufferMemory (Redis)

```typescript
import { ConversationBufferMemory } from 'langchain/memory';
import { ChatOpenAI } from '@langchain/openai';
import Redis from 'ioredis';

/**
 * Short-Term Memory with Redis Backing
 */
class ShortTermMemoryManager {
  private redis: Redis;
  
  constructor() {
    this.redis = new Redis({
      host: process.env.REDIS_HOST || 'localhost',
      port: parseInt(process.env.REDIS_PORT || '6379'),
    });
  }
  
  /**
   * Save conversation turn with 24-hour TTL
   */
  async saveConversationTurn(
    sessionId: string,
    userMessage: string,
    assistantMessage: string
  ) {
    const key = `conversation:${sessionId}`;
    const turn = {
      user: userMessage,
      assistant: assistantMessage,
      timestamp: new Date().toISOString(),
    };
    
    // Append to conversation history
    await this.redis.rpush(key, JSON.stringify(turn));
    
    // Set 24-hour expiration
    await this.redis.expire(key, 86400);
    
    console.log(`[Short-Term Memory] Saved turn to ${sessionId} (TTL: 24h)`);
  }
  
  /**
   * Get conversation history
   */
  async getConversationHistory(sessionId: string): Promise<any[]> {
    const key = `conversation:${sessionId}`;
    const turns = await this.redis.lrange(key, 0, -1);
    
    return turns.map((turn) => JSON.parse(turn));
  }
  
  /**
   * Get recent N turns
   */
  async getRecentTurns(sessionId: string, n: number = 5): Promise<any[]> {
    const key = `conversation:${sessionId}`;
    const turns = await this.redis.lrange(key, -n, -1);
    
    return turns.map((turn) => JSON.parse(turn));
  }
  
  /**
   * Clear conversation
   */
  async clearConversation(sessionId: string) {
    const key = `conversation:${sessionId}`;
    await this.redis.del(key);
    console.log(`[Short-Term Memory] Cleared ${sessionId}`);
  }
}

/**
 * QA Example: Test Execution Context
 */
class QATestMemory {
  private redis: Redis;
  
  constructor() {
    this.redis = new Redis();
  }
  
  /**
   * Save test execution context
   */
  async saveTestContext(ticketId: string, context: any) {
    const key = `qa:test:${ticketId}`;
    await this.redis.setex(
      key,
      86400, // 24 hours
      JSON.stringify(context)
    );
    
    console.log(`[QA Memory] Saved test context for ${ticketId}`);
  }
  
  /**
   * Get test context
   */
  async getTestContext(ticketId: string): Promise<any | null> {
    const key = `qa:test:${ticketId}`;
    const data = await this.redis.get(key);
    
    if (!data) {
      console.log(`[QA Memory] No context found for ${ticketId}`);
      return null;
    }
    
    return JSON.parse(data);
  }
  
  /**
   * Save test results temporarily
   */
  async saveTestResults(ticketId: string, results: any) {
    const key = `qa:results:${ticketId}`;
    await this.redis.setex(key, 86400, JSON.stringify(results));
  }
  
  /**
   * Get all test results for today
   */
  async getTodayTestResults(): Promise<any[]> {
    const pattern = 'qa:results:*';
    const keys = await this.redis.keys(pattern);
    
    const results = await Promise.all(
      keys.map(async (key) => {
        const data = await this.redis.get(key);
        return data ? JSON.parse(data) : null;
      })
    );
    
    return results.filter((r) => r !== null);
  }
}

/**
 * Banking Example: Customer Session Memory
 */
class BankingSessionMemory {
  private redis: Redis;
  
  constructor() {
    this.redis = new Redis();
  }
  
  /**
   * Save customer session
   */
  async saveCustomerSession(customerId: string, session: any) {
    const key = `banking:session:${customerId}`;
    await this.redis.setex(
      key,
      3600, // 1 hour for banking session
      JSON.stringify(session)
    );
  }
  
  /**
   * Get active customer session
   */
  async getCustomerSession(customerId: string): Promise<any | null> {
    const key = `banking:session:${customerId}`;
    const data = await this.redis.get(key);
    return data ? JSON.parse(data) : null;
  }
  
  /**
   * Update loan application progress
   */
  async updateLoanProgress(applicationId: string, progress: any) {
    const key = `banking:loan:${applicationId}`;
    await this.redis.setex(key, 86400, JSON.stringify(progress));
  }
}

/**
 * Usage Example: QA Test Memory
 */
async function qaMemoryExample() {
  const qaMemory = new QATestMemory();
  
  // Save test context when test starts
  await qaMemory.saveTestContext('QA-1234', {
    ticketSummary: 'Implement login functionality',
    testPlan: { testCases: 8, coverage: 85 },
    startedAt: new Date().toISOString(),
  });
  
  // Later, retrieve context for resume or retry
  const context = await qaMemory.getTestContext('QA-1234');
  console.log('Retrieved context:', context);
  
  // Save test results
  await qaMemory.saveTestResults('QA-1234', {
    passed: 7,
    failed: 1,
    completedAt: new Date().toISOString(),
  });
  
  // Get all test results from today
  const todayResults = await qaMemory.getTodayTestResults();
  console.log(`Today's test executions: ${todayResults.length}`);
}
```

---

## 3. Long-Term Memory

### VectorStoreRetrieverMemory (Pinecone)

```typescript
import { Pinecone } from '@pinecone-database/pinecone';
import { OpenAIEmbeddings } from '@langchain/openai';

/**
 * Long-Term Memory with Vector Search
 */
class LongTermMemoryManager {
  private pinecone: Pinecone;
  private embeddings: OpenAIEmbeddings;
  
  constructor() {
    this.pinecone = new Pinecone({
      apiKey: process.env.PINECONE_API_KEY!,
    });
    this.embeddings = new OpenAIEmbeddings({
      openAIApiKey: process.env.OPENAI_API_KEY,
    });
  }
  
  /**
   * Store conversation summary with embedding
   */
  async storeConversationSummary(
    conversationId: string,
    summary: string,
    metadata: any
  ) {
    const index = this.pinecone.index('conversation-memory');
    
    // Generate embedding
    const embedding = await this.embeddings.embedQuery(summary);
    
    // Store in Pinecone
    await index.upsert([
      {
        id: conversationId,
        values: embedding,
        metadata: {
          summary,
          ...metadata,
          storedAt: new Date().toISOString(),
        },
      },
    ]);
    
    console.log(`[Long-Term Memory] Stored conversation ${conversationId}`);
  }
  
  /**
   * Find similar past conversations
   */
  async findSimilarConversations(
    query: string,
    topK: number = 5
  ): Promise<any[]> {
    const index = this.pinecone.index('conversation-memory');
    
    // Generate query embedding
    const queryEmbedding = await this.embeddings.embedQuery(query);
    
    // Search similar conversations
    const results = await index.query({
      vector: queryEmbedding,
      topK,
      includeMetadata: true,
    });
    
    return (
      results.matches?.map((match) => ({
        conversationId: match.id,
        similarity: match.score,
        summary: match.metadata?.summary,
        metadata: match.metadata,
      })) || []
    );
  }
  
  /**
   * Delete old conversations (cleanup)
   */
  async deleteOldConversations(olderThanDays: number) {
    const index = this.pinecone.index('conversation-memory');
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - olderThanDays);
    
    // In production, you'd query by metadata and delete matching IDs
    console.log(`[Long-Term Memory] Would delete conversations older than ${olderThanDays} days`);
  }
}

/**
 * QA Example: SOP Memory
 */
class QASopMemory {
  private pinecone: Pinecone;
  private embeddings: OpenAIEmbeddings;
  
  constructor() {
    this.pinecone = new Pinecone({ apiKey: process.env.PINECONE_API_KEY! });
    this.embeddings = new OpenAIEmbeddings();
  }
  
  /**
   * Store SOP document
   */
  async storeSop(sopId: string, content: string, metadata: any) {
    const index = this.pinecone.index('sop-embeddings');
    
    const embedding = await this.embeddings.embedQuery(content);
    
    await index.upsert([
      {
        id: sopId,
        values: embedding,
        metadata: {
          content,
          ...metadata,
        },
      },
    ]);
    
    console.log(`[SOP Memory] Stored SOP ${sopId}`);
  }
  
  /**
   * Find relevant SOPs for ticket
   */
  async findRelevantSops(ticketDescription: string, topK: number = 3): Promise<any[]> {
    const index = this.pinecone.index('sop-embeddings');
    
    const queryEmbedding = await this.embeddings.embedQuery(ticketDescription);
    
    const results = await index.query({
      vector: queryEmbedding,
      topK,
      includeMetadata: true,
    });
    
    return (
      results.matches?.map((match) => ({
        sopId: match.id,
        relevanceScore: match.score,
        content: match.metadata?.content,
      })) || []
    );
  }
  
  /**
   * Store similar ticket history
   */
  async storeSimilarTicket(ticketId: string, description: string, resolution: string) {
    const index = this.pinecone.index('ticket-history');
    
    const embedding = await this.embeddings.embedQuery(description);
    
    await index.upsert([
      {
        id: ticketId,
        values: embedding,
        metadata: {
          description,
          resolution,
          resolvedAt: new Date().toISOString(),
        },
      },
    ]);
  }
  
  /**
   * Find similar past tickets
   */
  async findSimilarTickets(currentDescription: string, topK: number = 5): Promise<any[]> {
    const index = this.pinecone.index('ticket-history');
    
    const queryEmbedding = await this.embeddings.embedQuery(currentDescription);
    
    const results = await index.query({
      vector: queryEmbedding,
      topK,
      includeMetadata: true,
    });
    
    return (
      results.matches?.map((match) => ({
        ticketId: match.id,
        similarity: match.score,
        description: match.metadata?.description,
        resolution: match.metadata?.resolution,
      })) || []
    );
  }
}

/**
 * Banking Example: Transaction Pattern Memory
 */
class BankingTransactionMemory {
  private pinecone: Pinecone;
  private embeddings: OpenAIEmbeddings;
  
  constructor() {
    this.pinecone = new Pinecone({ apiKey: process.env.PINECONE_API_KEY! });
    this.embeddings = new OpenAIEmbeddings();
  }
  
  /**
   * Store customer transaction patterns
   */
  async storeTransactionPattern(
    customerId: string,
    pattern: string,
    metadata: any
  ) {
    const index = this.pinecone.index('transaction-patterns');
    
    const embedding = await this.embeddings.embedQuery(pattern);
    
    await index.upsert([
      {
        id: `${customerId}_${Date.now()}`,
        values: embedding,
        metadata: {
          customerId,
          pattern,
          ...metadata,
        },
      },
    ]);
  }
  
  /**
   * Find similar fraud patterns
   */
  async findSimilarFraudPatterns(
    currentPattern: string,
    topK: number = 10
  ): Promise<any[]> {
    const index = this.pinecone.index('fraud-patterns');
    
    const queryEmbedding = await this.embeddings.embedQuery(currentPattern);
    
    const results = await index.query({
      vector: queryEmbedding,
      topK,
      includeMetadata: true,
    });
    
    return (
      results.matches?.map((match) => ({
        patternId: match.id,
        similarity: match.score,
        fraudType: match.metadata?.fraudType,
        detectedAt: match.metadata?.detectedAt,
      })) || []
    );
  }
}

/**
 * Usage Example
 */
async function longTermMemoryExample() {
  const sopMemory = new QASopMemory();
  
  // Store SOP
  await sopMemory.storeSop(
    'SOP-AUTH-001',
    'Login testing standards: Validate email format, password strength, error messages...',
    { category: 'Authentication', version: '2.0' }
  );
  
  // Find relevant SOPs for new ticket
  const relevantSops = await sopMemory.findRelevantSops(
    'Implement user login with email and password'
  );
  
  console.log('Relevant SOPs:', relevantSops);
  
  // Find similar past tickets
  const similarTickets = await sopMemory.findSimilarTickets(
    'Add login form with validation'
  );
  
  console.log('Similar past tickets:', similarTickets);
}
```

---

## 4. Entity Memory

### ConversationEntityMemory (PostgreSQL)

```typescript
import { Pool } from 'pg';

/**
 * Entity Memory for tracking structured entities
 */
class EntityMemoryManager {
  private pool: Pool;
  
  constructor() {
    this.pool = new Pool({
      host: process.env.POSTGRES_HOST,
      database: process.env.POSTGRES_DB,
      user: process.env.POSTGRES_USER,
      password: process.env.POSTGRES_PASSWORD,
      port: parseInt(process.env.POSTGRES_PORT || '5432'),
    });
  }
  
  /**
   * Initialize database schema
   */
  async initialize() {
    await this.pool.query(`
      CREATE TABLE IF NOT EXISTS entities (
        id SERIAL PRIMARY KEY,
        entity_type VARCHAR(50) NOT NULL,
        entity_id VARCHAR(100) NOT NULL,
        entity_data JSONB NOT NULL,
        created_at TIMESTAMP DEFAULT NOW(),
        updated_at TIMESTAMP DEFAULT NOW(),
        UNIQUE(entity_type, entity_id)
      );
      
      CREATE INDEX IF NOT EXISTS idx_entity_type ON entities(entity_type);
      CREATE INDEX IF NOT EXISTS idx_entity_id ON entities(entity_id);
    `);
  }
  
  /**
   * Save or update entity
   */
  async saveEntity(entityType: string, entityId: string, data: any) {
    await this.pool.query(
      `INSERT INTO entities (entity_type, entity_id, entity_data)
       VALUES ($1, $2, $3)
       ON CONFLICT (entity_type, entity_id)
       DO UPDATE SET entity_data = $3, updated_at = NOW()`,
      [entityType, entityId, JSON.stringify(data)]
    );
    
    console.log(`[Entity Memory] Saved ${entityType}:${entityId}`);
  }
  
  /**
   * Get entity
   */
  async getEntity(entityType: string, entityId: string): Promise<any | null> {
    const result = await this.pool.query(
      'SELECT entity_data FROM entities WHERE entity_type = $1 AND entity_id = $2',
      [entityType, entityId]
    );
    
    if (result.rows.length === 0) return null;
    
    return result.rows[0].entity_data;
  }
  
  /**
   * Get all entities of a type
   */
  async getEntitiesByType(entityType: string): Promise<any[]> {
    const result = await this.pool.query(
      'SELECT entity_id, entity_data FROM entities WHERE entity_type = $1',
      [entityType]
    );
    
    return result.rows.map((row) => ({
      entityId: row.entity_id,
      data: row.entity_data,
    }));
  }
  
  /**
   * Delete entity
   */
  async deleteEntity(entityType: string, entityId: string) {
    await this.pool.query(
      'DELETE FROM entities WHERE entity_type = $1 AND entity_id = $2',
      [entityType, entityId]
    );
  }
}

/**
 * QA Example: Team and Project Entities
 */
class QAEntityMemory {
  private entityMemory: EntityMemoryManager;
  
  constructor() {
    this.entityMemory = new EntityMemoryManager();
  }
  
  async initialize() {
    await this.entityMemory.initialize();
    
    // Create additional QA-specific tables
    await this.entityMemory['pool'].query(`
      CREATE TABLE IF NOT EXISTS test_histories (
        id SERIAL PRIMARY KEY,
        ticket_id VARCHAR(50) NOT NULL,
        test_plan JSONB,
        test_results JSONB,
        executed_at TIMESTAMP DEFAULT NOW()
      );
      
      CREATE TABLE IF NOT EXISTS team_preferences (
        team_id VARCHAR(50) PRIMARY KEY,
        preferences JSONB NOT NULL,
        updated_at TIMESTAMP DEFAULT NOW()
      );
    `);
  }
  
  /**
   * Save team preferences
   */
  async saveTeamPreferences(teamId: string, preferences: any) {
    await this.entityMemory['pool'].query(
      `INSERT INTO team_preferences (team_id, preferences)
       VALUES ($1, $2)
       ON CONFLICT (team_id)
       DO UPDATE SET preferences = $2, updated_at = NOW()`,
      [teamId, JSON.stringify(preferences)]
    );
  }
  
  /**
   * Get team preferences
   */
  async getTeamPreferences(teamId: string): Promise<any | null> {
    const result = await this.entityMemory['pool'].query(
      'SELECT preferences FROM team_preferences WHERE team_id = $1',
      [teamId]
    );
    
    return result.rows.length > 0 ? result.rows[0].preferences : null;
  }
  
  /**
   * Save test execution history
   */
  async saveTestHistory(ticketId: string, testPlan: any, testResults: any) {
    await this.entityMemory['pool'].query(
      'INSERT INTO test_histories (ticket_id, test_plan, test_results) VALUES ($1, $2, $3)',
      [ticketId, JSON.stringify(testPlan), JSON.stringify(testResults)]
    );
  }
  
  /**
   * Get test history for ticket
   */
  async getTestHistory(ticketId: string): Promise<any[]> {
    const result = await this.entityMemory['pool'].query(
      'SELECT test_plan, test_results, executed_at FROM test_histories WHERE ticket_id = $1 ORDER BY executed_at DESC',
      [ticketId]
    );
    
    return result.rows;
  }
  
  /**
   * Save project SOPs
   */
  async saveProjectSop(projectId: string, sops: string[]) {
    await this.entityMemory.saveEntity('project_sops', projectId, { sops });
  }
  
  /**
   * Get project SOPs
   */
  async getProjectSops(projectId: string): Promise<string[]> {
    const entity = await this.entityMemory.getEntity('project_sops', projectId);
    return entity?.sops || [];
  }
}

/**
 * Banking Example: Customer Entity Memory
 */
class BankingEntityMemory {
  private entityMemory: EntityMemoryManager;
  
  constructor() {
    this.entityMemory = new EntityMemoryManager();
  }
  
  /**
   * Save customer profile
   */
  async saveCustomerProfile(customerId: string, profile: any) {
    await this.entityMemory.saveEntity('customer_profile', customerId, profile);
  }
  
  /**
   * Get customer profile
   */
  async getCustomerProfile(customerId: string): Promise<any | null> {
    return await this.entityMemory.getEntity('customer_profile', customerId);
  }
  
  /**
   * Save loan application
   */
  async saveLoanApplication(applicationId: string, application: any) {
    await this.entityMemory.saveEntity('loan_application', applicationId, application);
  }
  
  /**
   * Get customer loan history
   */
  async getCustomerLoanHistory(customerId: string): Promise<any[]> {
    const allLoans = await this.entityMemory.getEntitiesByType('loan_application');
    return allLoans.filter((loan) => loan.data.customerId === customerId);
  }
  
  /**
   * Save risk assessment
   */
  async saveRiskAssessment(customerId: string, assessment: any) {
    await this.entityMemory.saveEntity('risk_assessment', customerId, {
      ...assessment,
      assessedAt: new Date().toISOString(),
    });
  }
}

/**
 * Usage Example
 */
async function entityMemoryExample() {
  const qaMemory = new QAEntityMemory();
  await qaMemory.initialize();
  
  // Save team preferences
  await qaMemory.saveTeamPreferences('team-qa-backend', {
    defaultBrowser: 'chromium',
    screenshotOnFailure: true,
    maxRetries: 3,
    notificationChannel: 'slack-qa-alerts',
  });
  
  // Get team preferences
  const preferences = await qaMemory.getTeamPreferences('team-qa-backend');
  console.log('Team preferences:', preferences);
  
  // Save test history
  await qaMemory.saveTestHistory(
    'QA-1234',
    { testCases: 8, coverage: 85 },
    { passed: 7, failed: 1 }
  );
  
  // Get test history
  const history = await qaMemory.getTestHistory('QA-1234');
  console.log('Test history:', history);
  
  // Save project SOPs
  await qaMemory.saveProjectSop('project-auth', [
    'SOP-AUTH-001',
    'SOP-AUTH-002',
    'SOP-SEC-005',
  ]);
}
```

---

## 5. QA Automation Memory

### Complete 3-Tier Memory for QA

```typescript
/**
 * Complete QA Memory Manager
 * Combines all three tiers
 */
class CompleteQAMemoryManager {
  private shortTerm: ShortTermMemoryManager;
  private longTerm: QASopMemory;
  private entity: QAEntityMemory;
  
  constructor() {
    this.shortTerm = new ShortTermMemoryManager();
    this.longTerm = new QASopMemory();
    this.entity = new QAEntityMemory();
  }
  
  async initialize() {
    await this.entity.initialize();
  }
  
  /**
   * Start test execution - save to short-term
   */
  async startTestExecution(ticketId: string, ticketDetails: any) {
    // Short-term: Save current execution context
    await this.shortTerm.redis.setex(
      `qa:current:${ticketId}`,
      86400,
      JSON.stringify({
        ticketDetails,
        status: 'in_progress',
        startedAt: new Date().toISOString(),
      })
    );
    
    console.log(`[QA Memory] Started test execution for ${ticketId}`);
  }
  
  /**
   * Find relevant context from long-term memory
   */
  async findRelevantContext(ticketDescription: string) {
    // Long-term: Find similar SOPs and past tickets
    const [sops, similarTickets] = await Promise.all([
      this.longTerm.findRelevantSops(ticketDescription, 3),
      this.longTerm.findSimilarTickets(ticketDescription, 5),
    ]);
    
    return { sops, similarTickets };
  }
  
  /**
   * Get team-specific configuration
   */
  async getTeamConfiguration(teamId: string) {
    // Entity: Get team preferences
    const preferences = await this.entity.getTeamPreferences(teamId);
    return preferences || this.getDefaultConfiguration();
  }
  
  /**
   * Complete test execution - save to all tiers
   */
  async completeTestExecution(
    ticketId: string,
    testPlan: any,
    testResults: any
  ) {
    // 1. Update short-term status
    await this.shortTerm.redis.setex(
      `qa:current:${ticketId}`,
      86400,
      JSON.stringify({
        status: 'completed',
        testResults,
        completedAt: new Date().toISOString(),
      })
    );
    
    // 2. Save to long-term for future similarity search
    const ticketDescription = `Test execution for ${ticketId}: ${testResults.passed} passed, ${testResults.failed} failed`;
    await this.longTerm.storeSimilarTicket(
      ticketId,
      ticketDescription,
      `Automated testing completed with ${testPlan.coverage}% coverage`
    );
    
    // 3. Save to entity memory for historical tracking
    await this.entity.saveTestHistory(ticketId, testPlan, testResults);
    
    console.log(`[QA Memory] Saved test execution to all memory tiers`);
  }
  
  /**
   * Get comprehensive test context
   */
  async getTestContext(ticketId: string) {
    // Gather from all tiers
    const [currentExecution, testHistory] = await Promise.all([
      this.shortTerm.redis.get(`qa:current:${ticketId}`),
      this.entity.getTestHistory(ticketId),
    ]);
    
    return {
      current: currentExecution ? JSON.parse(currentExecution) : null,
      history: testHistory,
    };
  }
  
  private getDefaultConfiguration() {
    return {
      defaultBrowser: 'chromium',
      screenshotOnFailure: true,
      maxRetries: 3,
    };
  }
}

/**
 * Usage in QA Workflow
 */
async function qaWorkflowWithMemory() {
  const memory = new CompleteQAMemoryManager();
  await memory.initialize();
  
  const ticketId = 'QA-1234';
  const ticketDescription = 'Implement user login with email and password validation';
  
  // 1. Start execution - save to short-term
  await memory.startTestExecution(ticketId, {
    summary: 'Login feature',
    description: ticketDescription,
  });
  
  // 2. Find relevant context from long-term memory
  const context = await memory.findRelevantContext(ticketDescription);
  console.log('Relevant SOPs:', context.sops.length);
  console.log('Similar tickets:', context.similarTickets.length);
  
  // 3. Get team configuration from entity memory
  const teamConfig = await memory.getTeamConfiguration('team-qa-backend');
  console.log('Team config:', teamConfig);
  
  // 4. Execute tests (simulated)
  const testResults = {
    passed: 7,
    failed: 1,
    screenshots: ['/tmp/screenshot_1.png'],
  };
  
  // 5. Complete execution - save to all tiers
  await memory.completeTestExecution(
    ticketId,
    { testCases: 8, coverage: 85 },
    testResults
  );
  
  // 6. Later, retrieve test context
  const fullContext = await memory.getTestContext(ticketId);
  console.log('Full test context:', fullContext);
}
```

---

## 6. Banking Memory Examples

### Customer Conversation Memory

```typescript
/**
 * Banking: Customer Service Memory
 */
class BankingConversationMemory {
  private shortTerm: Redis;
  private longTerm: LongTermMemoryManager;
  private entity: BankingEntityMemory;
  
  constructor() {
    this.shortTerm = new Redis();
    this.longTerm = new LongTermMemoryManager();
    this.entity = new BankingEntityMemory();
  }
  
  /**
   * Save customer conversation turn
   */
  async saveConversationTurn(
    customerId: string,
    userMessage: string,
    agentResponse: string
  ) {
    const conversationKey = `banking:conversation:${customerId}`;
    
    const turn = {
      user: userMessage,
      agent: agentResponse,
      timestamp: new Date().toISOString(),
    };
    
    // Short-term: Append to conversation (1 hour TTL)
    await this.shortTerm.rpush(conversationKey, JSON.stringify(turn));
    await this.shortTerm.expire(conversationKey, 3600);
  }
  
  /**
   * Get conversation context
   */
  async getConversationContext(customerId: string, lastN: number = 5): Promise<string> {
    const conversationKey = `banking:conversation:${customerId}`;
    const turns = await this.shortTerm.lrange(conversationKey, -lastN, -1);
    
    const conversation = turns
      .map((turn) => JSON.parse(turn))
      .map((t) => `User: ${t.user}\nAgent: ${t.agent}`)
      .join('\n\n');
    
    return conversation;
  }
  
  /**
   * Save conversation summary to long-term memory
   */
  async saveConversationSummary(customerId: string, summary: string) {
    await this.longTerm.storeConversationSummary(
      `${customerId}_${Date.now()}`,
      summary,
      {
        customerId,
        type: 'customer_service',
        savedAt: new Date().toISOString(),
      }
    );
  }
  
  /**
   * Find similar past conversations
   */
  async findSimilarConversations(query: string): Promise<any[]> {
    return await this.longTerm.findSimilarConversations(query, 5);
  }
  
  /**
   * Update customer profile
   */
  async updateCustomerProfile(customerId: string, updates: any) {
    const existingProfile = await this.entity.getCustomerProfile(customerId);
    
    const updatedProfile = {
      ...existingProfile,
      ...updates,
      lastUpdated: new Date().toISOString(),
    };
    
    await this.entity.saveCustomerProfile(customerId, updatedProfile);
  }
}

/**
 * Usage: Customer Service with Memory
 */
async function customerServiceWithMemory() {
  const memory = new BankingConversationMemory();
  const customerId = 'CUST-12345';
  
  // Customer asks about loan
  await memory.saveConversationTurn(
    customerId,
    'I want to apply for a personal loan',
    'I can help you with that. What amount are you looking for?'
  );
  
  // Get conversation context for next turn
  const context = await memory.getConversationContext(customerId);
  console.log('Conversation context:', context);
  
  // Customer continues
  await memory.saveConversationTurn(
    customerId,
    'I need AED 500,000 for home renovation',
    'Great! Let me check your eligibility. Your credit score is excellent...'
  );
  
  // After conversation ends, save summary to long-term
  await memory.saveConversationSummary(
    customerId,
    'Customer inquired about personal loan of AED 500,000 for home renovation. Eligibility confirmed.'
  );
  
  // Update customer profile
  await memory.updateCustomerProfile(customerId, {
    loanInterest: 'personal_loan',
    loanAmount: 500000,
    loanPurpose: 'home_renovation',
  });
}
```

---

## 7. Memory Best Practices

### Memory Management Patterns

```typescript
/**
 * Best Practices for Memory Management
 */

// ✅ DO: Use appropriate memory tier
class MemoryBestPractices {
  // Short-term for current session/execution
  async saveCurrentExecution(data: any) {
    await redis.setex('current:execution', 86400, JSON.stringify(data));
  }
  
  // Long-term for semantic search
  async savePastPattern(pattern: string) {
    const embedding = await generateEmbedding(pattern);
    await pinecone.upsert({ embedding, metadata: { pattern } });
  }
  
  // Entity for structured relationships
  async saveEntityRelationship(entity: any) {
    await postgres.query('INSERT INTO entities ...', [entity]);
  }
  
  // ✅ DO: Set appropriate TTLs
  async saveWithTTL(key: string, data: any, ttl: number) {
    await redis.setex(key, ttl, JSON.stringify(data));
    // 3600 = 1 hour for banking sessions
    // 86400 = 24 hours for test executions
    // -1 = permanent for critical data
  }
  
  // ✅ DO: Implement memory cleanup
  async cleanupOldMemory() {
    // Delete old short-term keys
    const oldKeys = await redis.keys('qa:test:*');
    for (const key of oldKeys) {
      const ttl = await redis.ttl(key);
      if (ttl === -1) {
        // No TTL set, clean up manually
        await redis.del(key);
      }
    }
    
    // Delete old vectors from Pinecone
    // In production, implement date-based cleanup
  }
  
  // ✅ DO: Handle memory failures gracefully
  async getMemoryWithFallback(key: string): Promise<any> {
    try {
      const data = await redis.get(key);
      return data ? JSON.parse(data) : null;
    } catch (error) {
      console.error('Memory retrieval failed:', error);
      return null; // Graceful degradation
    }
  }
  
  // ❌ DON'T: Store large objects in Redis
  async badLargeStorage() {
    // Bad: Storing 10MB screenshot in Redis
    const largeScreenshot = Buffer.alloc(10 * 1024 * 1024);
    await redis.set('screenshot', largeScreenshot.toString('base64')); // ❌
    
    // Good: Store in S3, save URL in Redis
    const s3Url = await uploadToS3(largeScreenshot);
    await redis.setex('screenshot:url', 86400, s3Url); // ✅
  }
  
  // ❌ DON'T: Skip TTL for temporary data
  async badNoTTL() {
    // Bad: No TTL, will stay forever
    await redis.set('temp:data', 'value'); // ❌
    
    // Good: Always set TTL for temporary data
    await redis.setex('temp:data', 3600, 'value'); // ✅
  }
}
```

### Memory Architecture Summary

```
┌─────────────────────────────────────────────────────────────┐
│         MEMORY TIER SELECTION GUIDE                         │
└─────────────────────────────────────────────────────────────┘

Use Case                    Tier        Storage      TTL
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Current test execution      Short       Redis        24h
Current banking session     Short       Redis        1h
Recent chat turns           Short       Redis        24h

SOP documents               Long        Pinecone     Permanent
Similar ticket search       Long        Pinecone     Permanent
Conversation summaries      Long        Pinecone     90 days
Fraud patterns              Long        Pinecone     Permanent

Team preferences            Entity      PostgreSQL   Permanent
Test histories              Entity      PostgreSQL   Permanent
Customer profiles           Entity      PostgreSQL   Permanent
Project configurations      Entity      PostgreSQL   Permanent

Large files (screenshots)   N/A         S3           7-30 days
PDF reports                 N/A         S3           90 days
```

---

## Summary

### Key Takeaways

1. **Three-Tier Architecture**: Short-term (Redis), Long-term (Pinecone), Entity (PostgreSQL)
2. **Appropriate Storage**: Match data type to memory tier
3. **TTL Management**: Set expiration for temporary data
4. **Graceful Degradation**: Handle memory failures without breaking workflows
5. **Regular Cleanup**: Implement maintenance routines

### QA Automation Memory

- **Short-term**: Current test executions, recent results (24h)
- **Long-term**: SOP embeddings, similar tickets (permanent)
- **Entity**: Team preferences, test histories (permanent)

### Banking Memory

- **Short-term**: Customer sessions, active conversations (1h)
- **Long-term**: Transaction patterns, fraud detection (permanent)
- **Entity**: Customer profiles, loan applications (permanent)

---

**Related Files:**
- `Q54_Memory_Persistence_Strategies.md` - Implementation details
- `Q52_Agent_Communication_Patterns.md` - Agent coordination
- `Q55_Context_Window_Optimization.md` - Token management
- `QA_Automation_Scenario.md` - Complete QA implementation

---

**Document Version**: 1.0  
**Last Updated**: November 2025  
**For**: GenAI Interview Preparation

---

*End of Conversational Memory Management Document*
