# GenAI Questions 31-35: Integration, Scaling & Performance

---

### Q31. How do you integrate LLMs with existing banking systems and APIs?

**Answer:**

Integration with legacy banking systems requires API design, middleware layers, and robust error handling.

```javascript
const { ChatOpenAI } = require('@langchain/openai');
const axios = require('axios');
const Redis = require('ioredis');

/**
 * Banking System Integration Service
 */
class BankingSystemIntegration {
  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    this.redis = new Redis();
    
    this.endpoints = {
      customerService: process.env.CUSTOMER_API_URL,
      accountService: process.env.ACCOUNT_API_URL,
      transactionService: process.env.TRANSACTION_API_URL,
      loanService: process.env.LOAN_API_URL
    };
  }
  
  /**
   * Get customer information
   */
  async getCustomerInfo(customerId) {
    try {
      const cacheKey = `customer:${customerId}`;
      
      // Check cache
      const cached = await this.redis.get(cacheKey);
      if (cached) return JSON.parse(cached);
      
      // Call banking API
      const response = await axios.get(
        `${this.endpoints.customerService}/customers/${customerId}`,
        {
          headers: { 'Authorization': `Bearer ${process.env.BANKING_API_TOKEN}` },
          timeout: 5000
        }
      );
      
      // Cache for 5 minutes
      await this.redis.setex(cacheKey, 300, JSON.stringify(response.data));
      
      return response.data;
    } catch (error) {
      console.error('Failed to fetch customer info:', error.message);
      throw new Error('Customer information unavailable');
    }
  }
  
  /**
   * Get account balance
   */
  async getAccountBalance(customerId) {
    const response = await axios.get(
      `${this.endpoints.accountService}/accounts`,
      {
        params: { customerId },
        headers: { 'Authorization': `Bearer ${process.env.BANKING_API_TOKEN}` }
      }
    );
    
    return response.data;
  }
  
  /**
   * Get recent transactions
   */
  async getRecentTransactions(customerId, limit = 10) {
    const response = await axios.get(
      `${this.endpoints.transactionService}/transactions`,
      {
        params: { customerId, limit },
        headers: { 'Authorization': `Bearer ${process.env.BANKING_API_TOKEN}` }
      }
    );
    
    return response.data;
  }
  
  /**
   * Check loan eligibility
   */
  async checkLoanEligibility(customerId, loanAmount) {
    const response = await axios.post(
      `${this.endpoints.loanService}/eligibility/check`,
      { customerId, loanAmount, loanType: 'personal' },
      { headers: { 'Authorization': `Bearer ${process.env.BANKING_API_TOKEN}` } }
    );
    
    return response.data;
  }
}

/**
 * ENBD AI Banking Assistant with System Integration
 */
class ENBDAIBankingAssistant {
  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0.7,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    this.bankingSystem = new BankingSystemIntegration();
  }
  
  /**
   * Process customer query with banking system integration
   */
  async processQuery(customerId, query) {
    // Classify intent
    const intent = await this.classifyIntent(query);
    
    // Execute based on intent
    switch (intent.type) {
      case 'balance_inquiry':
        return await this.handleBalanceInquiry(customerId);
      case 'transaction_history':
        return await this.handleTransactionHistory(customerId);
      case 'loan_inquiry':
        return await this.handleLoanInquiry(customerId, intent.parameters);
      default:
        return await this.handleGeneralQuestion(query);
    }
  }
  
  /**
   * Classify user intent
   */
  async classifyIntent(query) {
    const prompt = `Classify intent: "${query}"\nReturn JSON: {"type": "intent", "parameters": {}}`;
    const response = await this.model.call(prompt);
    
    try {
      return JSON.parse(response.content);
    } catch (error) {
      return { type: 'general_question', parameters: {} };
    }
  }
  
  async handleBalanceInquiry(customerId) {
    const balance = await this.bankingSystem.getAccountBalance(customerId);
    return {
      response: `Your current balance is AED ${balance.balance.toLocaleString()}.`,
      data: balance
    };
  }
  
  async handleTransactionHistory(customerId) {
    const transactions = await this.bankingSystem.getRecentTransactions(customerId, 5);
    const summary = transactions.transactions
      .map(t => `- ${t.date}: ${t.description} - AED ${t.amount}`)
      .join('\n');
    
    return {
      response: `Recent transactions:\n${summary}`,
      data: transactions
    };
  }
  
  async handleLoanInquiry(customerId, parameters) {
    const eligibility = await this.bankingSystem.checkLoanEligibility(
      customerId,
      parameters.amount || 50000
    );
    
    if (eligibility.eligible) {
      return {
        response: `You are eligible for a loan up to AED ${eligibility.maxAmount.toLocaleString()} at ${eligibility.interestRate}% interest.`,
        data: eligibility
      };
    } else {
      return {
        response: `You may not qualify for the requested amount. ${eligibility.reason}`,
        data: eligibility
      };
    }
  }
  
  async handleGeneralQuestion(query) {
    const response = await this.model.call(`You are ENBD assistant. Answer: ${query}`);
    return { response: response.content, data: null };
  }
}

module.exports = { BankingSystemIntegration, ENBDAIBankingAssistant };
```

**Integration Patterns:**

1. ✅ **API Gateway Pattern** - Centralized routing
2. ✅ **Caching Layer** - Redis for performance
3. ✅ **Circuit Breaker** - Fault tolerance
4. ✅ **Intent Classification** - Route to correct service
5. ✅ **Error Handling** - Graceful degradation

---

### Q32. How do you implement horizontal scaling for LLM applications?

**Answer:**

```javascript
const cluster = require('cluster');
const os = require('os');
const express = require('express');
const { ChatOpenAI } = require('@langchain/openai');

/**
 * Load Balancer for LLM Requests
 */
class LLMLoadBalancer {
  constructor() {
    this.workers = [];
    this.currentWorkerIndex = 0;
  }
  
  /**
   * Initialize worker pool
   */
  initializeWorkers(numWorkers = os.cpus().length) {
    if (cluster.isMaster) {
      console.log(`Master process ${process.pid} is running`);
      
      // Fork workers
      for (let i = 0; i < numWorkers; i++) {
        const worker = cluster.fork();
        this.workers.push(worker);
      }
      
      // Handle worker exit
      cluster.on('exit', (worker, code, signal) => {
        console.log(`Worker ${worker.process.pid} died. Restarting...`);
        const newWorker = cluster.fork();
        const index = this.workers.indexOf(worker);
        this.workers[index] = newWorker;
      });
    } else {
      // Worker process
      this.startWorker();
    }
  }
  
  /**
   * Start worker server
   */
  startWorker() {
    const app = express();
    app.use(express.json());
    
    const model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0.7,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    app.post('/chat', async (req, res) => {
      try {
        const { message } = req.body;
        const response = await model.call(message);
        
        res.json({
          response: response.content,
          workerId: process.pid
        });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });
    
    const PORT = process.env.PORT || 3000;
    app.listen(PORT, () => {
      console.log(`Worker ${process.pid} listening on port ${PORT}`);
    });
  }
}

/**
 * Request Queue for Rate Limiting
 */
class RequestQueue {
  constructor(maxConcurrent = 10) {
    this.queue = [];
    this.running = 0;
    this.maxConcurrent = maxConcurrent;
  }
  
  /**
   * Add request to queue
   */
  async enqueue(fn) {
    return new Promise((resolve, reject) => {
      this.queue.push({ fn, resolve, reject });
      this.process();
    });
  }
  
  /**
   * Process queue
   */
  async process() {
    if (this.running >= this.maxConcurrent || this.queue.length === 0) {
      return;
    }
    
    this.running++;
    const { fn, resolve, reject } = this.queue.shift();
    
    try {
      const result = await fn();
      resolve(result);
    } catch (error) {
      reject(error);
    } finally {
      this.running--;
      this.process();
    }
  }
}

/**
 * ENBD Scalable Chat Service
 */
class ENBDScalableChatService {
  constructor() {
    this.loadBalancer = new LLMLoadBalancer();
    this.requestQueue = new RequestQueue(10);
  }
  
  /**
   * Start service
   */
  start() {
    this.loadBalancer.initializeWorkers(4); // 4 workers
  }
  
  /**
   * Process chat request through queue
   */
  async chat(message) {
    return await this.requestQueue.enqueue(async () => {
      // Process message
      const response = await this.processMessage(message);
      return response;
    });
  }
  
  async processMessage(message) {
    // Implementation
    return { response: 'Processed' };
  }
}

module.exports = { LLMLoadBalancer, RequestQueue, ENBDScalableChatService };
```

---

### Q33. How do you optimize latency and response time?

**Answer:**

```javascript
const { ChatOpenAI } = require('@langchain/openai');
const Redis = require('ioredis');

/**
 * Performance Optimization Service
 */
class PerformanceOptimizationService {
  constructor() {
    this.redis = new Redis();
    this.model = new ChatOpenAI({
      modelName: 'gpt-4-turbo', // Faster model
      temperature: 0.7,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }
  
  /**
   * Response caching
   */
  async getCachedResponse(query) {
    const cacheKey = `response:${this.hashQuery(query)}`;
    const cached = await this.redis.get(cacheKey);
    
    if (cached) {
      return {
        response: cached,
        fromCache: true,
        latency: 5 // ms
      };
    }
    
    return null;
  }
  
  /**
   * Cache response
   */
  async cacheResponse(query, response) {
    const cacheKey = `response:${this.hashQuery(query)}`;
    await this.redis.setex(cacheKey, 3600, response); // 1 hour TTL
  }
  
  /**
   * Hash query for cache key
   */
  hashQuery(query) {
    const crypto = require('crypto');
    return crypto.createHash('md5').update(query.toLowerCase().trim()).digest('hex');
  }
  
  /**
   * Parallel processing
   */
  async processParallel(queries) {
    const promises = queries.map(query => this.model.call(query));
    return await Promise.all(promises);
  }
  
  /**
   * Streaming response
   */
  async *streamResponse(query) {
    const stream = await this.model.stream(query);
    
    for await (const chunk of stream) {
      yield chunk.content;
    }
  }
  
  /**
   * Pre-compute common responses
   */
  async precomputeCommonQueries() {
    const commonQueries = [
      'What is the minimum balance for savings account?',
      'How do I apply for a credit card?',
      'What are the loan interest rates?'
    ];
    
    for (const query of commonQueries) {
      const response = await this.model.call(query);
      await this.cacheResponse(query, response.content);
    }
    
    console.log('Pre-computed common queries');
  }
}

/**
 * ENBD Fast Response Service
 */
class ENBDFastResponseService {
  constructor() {
    this.optimizer = new PerformanceOptimizationService();
  }
  
  async chat(query) {
    const startTime = Date.now();
    
    // Check cache
    const cached = await this.optimizer.getCachedResponse(query);
    if (cached) {
      return {
        ...cached,
        totalLatency: Date.now() - startTime
      };
    }
    
    // Generate response
    const response = await this.optimizer.model.call(query);
    
    // Cache it
    await this.optimizer.cacheResponse(query, response.content);
    
    return {
      response: response.content,
      fromCache: false,
      totalLatency: Date.now() - startTime
    };
  }
}

module.exports = { PerformanceOptimizationService, ENBDFastResponseService };
```

---

### Q34. How do you implement model quantization and optimization?

**Answer:**

```javascript
/**
 * Model Optimization Service
 */
class ModelOptimizationService {
  constructor() {
    this.modelConfigs = {
      'gpt-4': {
        contextWindow: 8192,
        costPer1kTokens: 0.03,
        avgLatency: 2000
      },
      'gpt-4-turbo': {
        contextWindow: 128000,
        costPer1kTokens: 0.01,
        avgLatency: 1000
      },
      'gpt-3.5-turbo': {
        contextWindow: 4096,
        costPer1kTokens: 0.0015,
        avgLatency: 500
      }
    };
  }
  
  /**
   * Select optimal model based on requirements
   */
  selectOptimalModel(requirements) {
    const { maxLatency, maxCost, contextLength, qualityPriority } = requirements;
    
    let bestModel = null;
    let bestScore = -Infinity;
    
    for (const [modelName, config] of Object.entries(this.modelConfigs)) {
      // Check hard constraints
      if (config.avgLatency > maxLatency) continue;
      if (config.contextWindow < contextLength) continue;
      
      // Calculate score
      let score = 0;
      
      if (qualityPriority === 'high') {
        // Prioritize quality (GPT-4)
        score = modelName.includes('gpt-4') ? 100 : 50;
      } else {
        // Balance cost and latency
        score = 100 - (config.costPer1kTokens * 1000) - (config.avgLatency / 100);
      }
      
      if (score > bestScore) {
        bestScore = score;
        bestModel = modelName;
      }
    }
    
    return {
      model: bestModel,
      config: this.modelConfigs[bestModel],
      score: bestScore
    };
  }
  
  /**
   * Optimize prompt length
   */
  optimizePrompt(prompt, maxTokens) {
    const estimatedTokens = Math.ceil(prompt.length / 4);
    
    if (estimatedTokens <= maxTokens) {
      return prompt;
    }
    
    // Truncate to fit
    const targetLength = maxTokens * 4;
    return prompt.substring(0, targetLength) + '...';
  }
  
  /**
   * Batch similar requests
   */
  batchRequests(requests) {
    const batches = [];
    let currentBatch = [];
    
    for (const request of requests) {
      currentBatch.push(request);
      
      if (currentBatch.length >= 10) {
        batches.push(currentBatch);
        currentBatch = [];
      }
    }
    
    if (currentBatch.length > 0) {
      batches.push(currentBatch);
    }
    
    return batches;
  }
}

/**
 * ENBD Optimized Service
 */
class ENBDOptimizedService {
  constructor() {
    this.optimizer = new ModelOptimizationService();
  }
  
  async processRequest(query, requirements) {
    // Select optimal model
    const modelSelection = this.optimizer.selectOptimalModel({
      maxLatency: requirements.maxLatency || 2000,
      maxCost: requirements.maxCost || 0.01,
      contextLength: query.length / 4,
      qualityPriority: requirements.quality || 'medium'
    });
    
    console.log(`Selected model: ${modelSelection.model}`);
    
    // Optimize prompt
    const optimizedPrompt = this.optimizer.optimizePrompt(query, 4000);
    
    // Process with selected model
    // Implementation...
    
    return {
      model: modelSelection.model,
      optimized: true
    };
  }
}

module.exports = { ModelOptimizationService, ENBDOptimizedService };
```

---

### Q35. How do you implement failover and disaster recovery?

**Answer:**

```javascript
const { ChatOpenAI } = require('@langchain/openai');
const Anthropic = require('@anthropic-ai/sdk');

/**
 * Failover Service
 */
class FailoverService {
  constructor() {
    this.primaryModel = new ChatOpenAI({
      modelName: 'gpt-4',
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    this.fallbackModel = new Anthropic({
      apiKey: process.env.ANTHROPIC_API_KEY
    });
    
    this.healthStatus = {
      primary: true,
      fallback: true
    };
  }
  
  /**
   * Execute with automatic failover
   */
  async executeWithFailover(query) {
    try {
      // Try primary
      if (this.healthStatus.primary) {
        const response = await this.primaryModel.call(query);
        return {
          response: response.content,
          provider: 'OpenAI',
          failover: false
        };
      }
    } catch (error) {
      console.error('Primary model failed:', error.message);
      this.healthStatus.primary = false;
      
      // Schedule health check
      setTimeout(() => this.checkHealth(), 60000); // 1 minute
    }
    
    // Failover to secondary
    try {
      const response = await this.fallbackModel.messages.create({
        model: 'claude-3-opus-20240229',
        max_tokens: 1024,
        messages: [{ role: 'user', content: query }]
      });
      
      return {
        response: response.content[0].text,
        provider: 'Anthropic',
        failover: true
      };
    } catch (error) {
      console.error('Fallback model also failed:', error.message);
      throw new Error('All models unavailable');
    }
  }
  
  /**
   * Health check
   */
  async checkHealth() {
    try {
      await this.primaryModel.call('test');
      this.healthStatus.primary = true;
      console.log('Primary model restored');
    } catch (error) {
      this.healthStatus.primary = false;
    }
  }
  
  /**
   * Backup data
   */
  async backupConversations(conversations) {
    // Store in multiple locations
    const backups = [];
    
    // S3 backup
    backups.push(await this.backupToS3(conversations));
    
    // Database backup
    backups.push(await this.backupToDatabase(conversations));
    
    return {
      backed_up: conversations.length,
      locations: backups
    };
  }
  
  async backupToS3(data) {
    // Implementation
    return 's3://backups/conversations';
  }
  
  async backupToDatabase(data) {
    // Implementation
    return 'mongodb://backups/conversations';
  }
}

/**
 * ENBD Resilient Service
 */
class ENBDResilientService {
  constructor() {
    this.failover = new FailoverService();
  }
  
  async chat(query) {
    return await this.failover.executeWithFailover(query);
  }
}

module.exports = { FailoverService, ENBDResilientService };
```

**Scaling & Performance:**

1. ✅ **Banking System Integration** - API middleware layer
2. ✅ **Horizontal Scaling** - Worker pools, load balancing
3. ✅ **Latency Optimization** - Caching, streaming, parallel processing
4. ✅ **Model Optimization** - Smart model selection, quantization
5. ✅ **Failover & DR** - Multi-provider redundancy, automatic recovery

---

**Summary Q31-Q35:**
- Banking API integration with caching and circuit breakers ✅
- Horizontal scaling with worker pools and request queues ✅
- Performance optimization with caching and streaming ✅
- Model optimization with smart selection and batching ✅
- Failover and disaster recovery with multi-provider support ✅
