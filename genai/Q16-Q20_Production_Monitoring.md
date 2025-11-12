# GenAI Questions 16-20: Production Deployment & Monitoring

---

### Q16. How do you implement production-ready error handling and fallbacks for LLM applications?

**Answer:**

Production LLM applications require robust error handling with circuit breakers, retry logic, fallback chains, and graceful degradation.

```javascript
const { ChatOpenAI } = require('@langchain/openai');
const { EventEmitter } = require('events');

/**
 * Error Handling Service for LLM Applications
 */
class LLMErrorHandler extends EventEmitter {
  constructor() {
    super();
    
    this.models = {
      primary: new ChatOpenAI({
        modelName: 'gpt-4',
        temperature: 0.7,
        openAIApiKey: process.env.OPENAI_API_KEY,
        timeout: 30000,
        maxRetries: 2
      }),
      fallback: new ChatOpenAI({
        modelName: 'gpt-3.5-turbo',
        temperature: 0.7,
        openAIApiKey: process.env.OPENAI_API_KEY,
        timeout: 15000,
        maxRetries: 3
      })
    };
    
    this.circuitBreaker = {
      failures: 0,
      threshold: 5,
      timeout: 60000, // 1 minute
      isOpen: false,
      lastFailure: null
    };
  }
  
  /**
   * Execute with retry logic and exponential backoff
   */
  async executeWithRetry(fn, maxRetries = 3, delay = 1000) {
    let lastError;
    
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        return await fn();
      } catch (error) {
        lastError = error;
        console.log(`Attempt ${attempt} failed:`, error.message);
        
        if (this.isNonRetryableError(error)) throw error;
        
        if (attempt < maxRetries) {
          const waitTime = delay * Math.pow(2, attempt - 1);
          console.log(`Waiting ${waitTime}ms before retry...`);
          await this.sleep(waitTime);
        }
      }
    }
    
    throw lastError;
  }
  
  /**
   * Circuit breaker pattern - Prevents cascade failures
   */
  async executeWithCircuitBreaker(fn) {
    if (this.circuitBreaker.isOpen) {
      const timeSince = Date.now() - this.circuitBreaker.lastFailure;
      
      if (timeSince < this.circuitBreaker.timeout) {
        throw new Error('Circuit breaker OPEN. Service unavailable.');
      } else {
        console.log('Circuit breaker: Attempting to close...');
        this.circuitBreaker.isOpen = false;
        this.circuitBreaker.failures = 0;
      }
    }
    
    try {
      const result = await fn();
      this.circuitBreaker.failures = 0;
      return result;
    } catch (error) {
      this.circuitBreaker.failures++;
      this.circuitBreaker.lastFailure = Date.now();
      
      if (this.circuitBreaker.failures >= this.circuitBreaker.threshold) {
        console.log('⚠️  Circuit breaker OPENED');
        this.circuitBreaker.isOpen = true;
        this.emit('circuit_open', { failures: this.circuitBreaker.failures });
      }
      
      throw error;
    }
  }
  
  /**
   * Fallback chain - Multiple recovery strategies
   */
  async executeWithFallbacks(prompt, options = {}) {
    const strategies = [
      async () => {
        console.log('Strategy 1: Primary model (GPT-4)');
        return await this.models.primary.call(prompt);
      },
      async () => {
        console.log('Strategy 2: Fallback model (GPT-3.5)');
        return await this.models.fallback.call(prompt);
      },
      async () => {
        console.log('Strategy 3: Cached response');
        const cached = await this.getCachedResponse(prompt);
        if (cached) return { content: cached };
        throw new Error('No cache available');
      },
      async () => {
        console.log('Strategy 4: Template response');
        return { content: this.getTemplateResponse(options.category || 'general') };
      }
    ];
    
    let lastError;
    for (const strategy of strategies) {
      try {
        const result = await this.executeWithRetry(strategy, 2, 500);
        return { content: result.content, fallbackLevel: strategies.indexOf(strategy) };
      } catch (error) {
        lastError = error;
        console.log(`Strategy failed: ${error.message}`);
      }
    }
    
    throw new Error(`All fallback strategies failed: ${lastError.message}`);
  }
  
  getTemplateResponse(category) {
    const templates = {
      general: 'I apologize for technical difficulties. Please try again or contact support.',
      account: 'For account queries, contact ENBD at 600 54 0000.',
      loan: 'For loan inquiries, visit our branch or call 600 54 0000.',
      card: 'For card services, call our 24/7 helpline at 600 54 0000.'
    };
    return templates[category] || templates.general;
  }
  
  isNonRetryableError(error) {
    return error.message.includes('context_length');
  }
  
  async getCachedResponse(prompt) {
    return null; // Implement Redis cache
  }
  
  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

/**
 * ENBD Robust Chat Service
 */
class ENBDRobustChatService {
  constructor() {
    this.errorHandler = new LLMErrorHandler();
    
    this.errorHandler.on('circuit_open', (data) => {
      console.log('🚨 ALERT: Circuit breaker opened!', data);
      this.sendAlert('Circuit breaker opened', data);
    });
  }
  
  async chat(message, context = '', options = {}) {
    try {
      const result = await this.errorHandler.executeWithCircuitBreaker(
        async () => {
          return await this.errorHandler.executeWithFallbacks(
            this.buildPrompt(message, context),
            { category: options.category }
          );
        }
      );
      
      return {
        success: true,
        response: result.content,
        fallbackLevel: result.fallbackLevel,
        error: null
      };
    } catch (error) {
      console.error('Chat failed:', error.message);
      
      return {
        success: false,
        response: this.errorHandler.getTemplateResponse(options.category),
        fallbackLevel: 'template',
        error: { message: error.message }
      };
    }
  }
  
  buildPrompt(message, context) {
    return context 
      ? `Context: ${context}\n\nCustomer: ${message}\n\nAssistant:`
      : `Customer: ${message}\n\nAssistant:`;
  }
  
  sendAlert(message, data) {
    console.log('📢 ALERT:', message, data);
  }
}

module.exports = { LLMErrorHandler, ENBDRobustChatService };
```

---

### Q17. How do you implement comprehensive monitoring and observability for LLM applications?

**Answer:**

```javascript
const { ChatOpenAI } = require('@langchain/openai');
const winston = require('winston');
const prometheus = require('prom-client');

/**
 * LLM Monitoring and Observability Service
 */
class LLMMonitoringService {
  constructor() {
    // Prometheus metrics
    this.metrics = {
      requestCount: new prometheus.Counter({
        name: 'llm_requests_total',
        help: 'Total LLM requests',
        labelNames: ['model', 'status', 'category']
      }),
      requestDuration: new prometheus.Histogram({
        name: 'llm_request_duration_seconds',
        help: 'LLM request duration',
        labelNames: ['model', 'category'],
        buckets: [0.1, 0.5, 1, 2, 5, 10, 30]
      }),
      tokenCount: new prometheus.Counter({
        name: 'llm_tokens_total',
        help: 'Total tokens consumed',
        labelNames: ['model', 'type']
      }),
      cost: new prometheus.Counter({
        name: 'llm_cost_dollars',
        help: 'Total cost in dollars',
        labelNames: ['model']
      }),
      errorCount: new prometheus.Counter({
        name: 'llm_errors_total',
        help: 'Total errors',
        labelNames: ['model', 'error_type']
      })
    };
    
    // Winston structured logging
    this.logger = winston.createLogger({
      level: 'info',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
      ),
      transports: [
        new winston.transports.Console(),
        new winston.transports.File({ filename: 'llm-error.log', level: 'error' }),
        new winston.transports.File({ filename: 'llm-combined.log' })
      ]
    });
  }
  
  /**
   * Track LLM request with metrics and logging
   */
  async trackRequest(fn, metadata = {}) {
    const startTime = Date.now();
    const model = metadata.model || 'unknown';
    const category = metadata.category || 'general';
    
    try {
      const result = await fn();
      const duration = (Date.now() - startTime) / 1000;
      
      // Record success metrics
      this.metrics.requestCount.inc({ model, status: 'success', category });
      this.metrics.requestDuration.observe({ model, category }, duration);
      
      if (result.tokens) {
        this.metrics.tokenCount.inc({ model, type: 'input' }, result.tokens.input || 0);
        this.metrics.tokenCount.inc({ model, type: 'output' }, result.tokens.output || 0);
      }
      
      if (result.cost) {
        this.metrics.cost.inc({ model }, result.cost);
      }
      
      // Log successful request
      this.logger.info('LLM request successful', {
        model, category, duration,
        tokens: result.tokens,
        cost: result.cost,
        ...metadata
      });
      
      return result;
    } catch (error) {
      const duration = (Date.now() - startTime) / 1000;
      
      // Record error metrics
      this.metrics.requestCount.inc({ model, status: 'error', category });
      this.metrics.errorCount.inc({ model, error_type: this.classifyError(error) });
      
      // Log error
      this.logger.error('LLM request failed', {
        model, category, duration,
        error: error.message,
        stack: error.stack,
        ...metadata
      });
      
      throw error;
    }
  }
  
  /**
   * Track response quality with user feedback
   */
  trackResponseQuality(request, response, feedback) {
    this.logger.info('Response quality feedback', {
      request,
      response: response.substring(0, 100),
      score: feedback.score,
      comment: feedback.comment,
      timestamp: new Date()
    });
  }
  
  /**
   * Health check for all dependencies
   */
  async healthCheck() {
    const checks = {
      openai: await this.checkOpenAI(),
      redis: await this.checkRedis(),
      database: await this.checkDatabase()
    };
    
    const isHealthy = Object.values(checks).every(c => c.healthy);
    
    return { healthy: isHealthy, checks, timestamp: new Date() };
  }
  
  async checkOpenAI() {
    try {
      const model = new ChatOpenAI({
        modelName: 'gpt-3.5-turbo',
        timeout: 5000,
        openAIApiKey: process.env.OPENAI_API_KEY
      });
      await model.call('test');
      return { healthy: true, message: 'OpenAI API accessible' };
    } catch (error) {
      return { healthy: false, message: error.message };
    }
  }
  
  async checkRedis() {
    return { healthy: true, message: 'Redis connected' };
  }
  
  async checkDatabase() {
    return { healthy: true, message: 'Database connected' };
  }
  
  classifyError(error) {
    const msg = error.message.toLowerCase();
    if (msg.includes('rate limit')) return 'rate_limit';
    if (msg.includes('timeout')) return 'timeout';
    if (msg.includes('context')) return 'context_length';
    if (msg.includes('network')) return 'network';
    return 'unknown';
  }
  
  /**
   * Get Prometheus metrics
   */
  async getMetrics() {
    return await prometheus.register.metrics();
  }
}

/**
 * ENBD Monitored Chat Service
 */
class ENBDMonitoredChatService {
  constructor() {
    this.monitoring = new LLMMonitoringService();
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0.7,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }
  
  async chat(message, metadata = {}) {
    return await this.monitoring.trackRequest(
      async () => {
        const prompt = `You are ENBD banking assistant.\n\nCustomer: ${message}\n\nAssistant:`;
        const response = await this.model.call(prompt);
        
        return {
          response: response.content,
          tokens: { input: this.countTokens(prompt), output: this.countTokens(response.content) },
          cost: this.calculateCost('gpt-4', this.countTokens(prompt), this.countTokens(response.content))
        };
      },
      { model: 'gpt-4', category: metadata.category || 'general' }
    );
  }
  
  async submitFeedback(request, response, score, comment) {
    this.monitoring.trackResponseQuality(request, response, { score, comment });
  }
  
  async getHealth() {
    return await this.monitoring.healthCheck();
  }
  
  async getMetrics() {
    return await this.monitoring.getMetrics();
  }
  
  countTokens(text) {
    return Math.ceil(text.length / 4); // Rough approximation
  }
  
  calculateCost(model, inputTokens, outputTokens) {
    const costs = {
      'gpt-4': { input: 0.03 / 1000, output: 0.06 / 1000 }
    };
    const modelCost = costs[model];
    return (inputTokens * modelCost.input) + (outputTokens * modelCost.output);
  }
}

module.exports = { LLMMonitoringService, ENBDMonitoredChatService };
```

---

### Q18. How do you implement rate limiting and quota management for LLM APIs?

**Answer:**

```javascript
const Redis = require('ioredis');
const { RateLimiterRedis } = require('rate-limiter-flexible');

/**
 * Rate Limiting Service
 */
class LLMRateLimitService {
  constructor() {
    this.redis = new Redis({
      host: process.env.REDIS_HOST || 'localhost',
      port: process.env.REDIS_PORT || 6379
    });
    
    // Different rate limits for different tiers
    this.tiers = {
      free: {
        requestsPerMinute: 10,
        requestsPerDay: 100,
        tokensPerDay: 10000,
        priority: 1
      },
      premium: {
        requestsPerMinute: 60,
        requestsPerDay: 1000,
        tokensPerDay: 100000,
        priority: 2
      },
      enterprise: {
        requestsPerMinute: 300,
        requestsPerDay: 10000,
        tokensPerDay: 1000000,
        priority: 3
      }
    };
    
    // Create rate limiters for each tier
    this.limiters = {};
    for (const [tier, limits] of Object.entries(this.tiers)) {
      this.limiters[tier] = {
        perMinute: new RateLimiterRedis({
          storeClient: this.redis,
          keyPrefix: `ratelimit_${tier}_minute`,
          points: limits.requestsPerMinute,
          duration: 60
        }),
        perDay: new RateLimiterRedis({
          storeClient: this.redis,
          keyPrefix: `ratelimit_${tier}_day`,
          points: limits.requestsPerDay,
          duration: 86400
        })
      };
    }
  }
  
  /**
   * Check if request is allowed
   */
  async checkRateLimit(userId, tier = 'free') {
    const limiter = this.limiters[tier];
    
    if (!limiter) {
      throw new Error(`Invalid tier: ${tier}`);
    }
    
    try {
      // Check minute limit
      await limiter.perMinute.consume(userId, 1);
      
      // Check daily limit
      await limiter.perDay.consume(userId, 1);
      
      return {
        allowed: true,
        remaining: await this.getRemainingQuota(userId, tier)
      };
    } catch (error) {
      if (error instanceof Error) {
        return {
          allowed: false,
          retryAfter: Math.ceil(error.msBeforeNext / 1000),
          message: `Rate limit exceeded. Retry after ${Math.ceil(error.msBeforeNext / 1000)}s`
        };
      }
      throw error;
    }
  }
  
  /**
   * Check token quota
   */
  async checkTokenQuota(userId, tier, tokensNeeded) {
    const key = `tokens:${tier}:${userId}`;
    const limit = this.tiers[tier].tokensPerDay;
    
    const used = await this.redis.get(key) || 0;
    const remaining = limit - parseInt(used);
    
    if (remaining < tokensNeeded) {
      return {
        allowed: false,
        remaining: remaining,
        limit: limit,
        message: `Token quota exceeded. ${remaining}/${limit} tokens remaining today.`
      };
    }
    
    return {
      allowed: true,
      remaining: remaining - tokensNeeded,
      limit: limit
    };
  }
  
  /**
   * Consume token quota
   */
  async consumeTokens(userId, tier, tokensUsed) {
    const key = `tokens:${tier}:${userId}`;
    
    await this.redis.incrby(key, tokensUsed);
    await this.redis.expire(key, 86400); // 24 hours
  }
  
  /**
   * Get remaining quota
   */
  async getRemainingQuota(userId, tier) {
    const limiter = this.limiters[tier];
    
    const minuteData = await limiter.perMinute.get(userId);
    const dayData = await limiter.perDay.get(userId);
    
    const tokenKey = `tokens:${tier}:${userId}`;
    const tokensUsed = await this.redis.get(tokenKey) || 0;
    
    return {
      requestsPerMinute: {
        remaining: minuteData ? this.tiers[tier].requestsPerMinute - minuteData.consumedPoints : this.tiers[tier].requestsPerMinute,
        limit: this.tiers[tier].requestsPerMinute
      },
      requestsPerDay: {
        remaining: dayData ? this.tiers[tier].requestsPerDay - dayData.consumedPoints : this.tiers[tier].requestsPerDay,
        limit: this.tiers[tier].requestsPerDay
      },
      tokensPerDay: {
        remaining: this.tiers[tier].tokensPerDay - parseInt(tokensUsed),
        limit: this.tiers[tier].tokensPerDay
      }
    };
  }
  
  /**
   * Priority queue for rate-limited requests
   */
  async addToQueue(userId, tier, request) {
    const priority = this.tiers[tier].priority;
    const queueKey = `queue:priority:${priority}`;
    
    await this.redis.zadd(queueKey, Date.now(), JSON.stringify({ userId, request }));
  }
  
  async getNextFromQueue() {
    // Check queues from highest to lowest priority
    for (let priority = 3; priority >= 1; priority--) {
      const queueKey = `queue:priority:${priority}`;
      const items = await this.redis.zrange(queueKey, 0, 0);
      
      if (items.length > 0) {
        await this.redis.zrem(queueKey, items[0]);
        return JSON.parse(items[0]);
      }
    }
    
    return null;
  }
}

/**
 * ENBD Rate-Limited Chat Service
 */
class ENBDRateLimitedChatService {
  constructor() {
    this.rateLimiter = new LLMRateLimitService();
    this.model = require('./model'); // Your LLM model
  }
  
  async chat(userId, message, tier = 'free') {
    // Check rate limit
    const rateLimitCheck = await this.rateLimiter.checkRateLimit(userId, tier);
    
    if (!rateLimitCheck.allowed) {
      return {
        error: 'Rate limit exceeded',
        retryAfter: rateLimitCheck.retryAfter,
        message: rateLimitCheck.message
      };
    }
    
    // Estimate tokens needed
    const estimatedTokens = this.estimateTokens(message);
    
    // Check token quota
    const tokenCheck = await this.rateLimiter.checkTokenQuota(userId, tier, estimatedTokens);
    
    if (!tokenCheck.allowed) {
      return {
        error: 'Token quota exceeded',
        remaining: tokenCheck.remaining,
        limit: tokenCheck.limit,
        message: tokenCheck.message
      };
    }
    
    // Process request
    const response = await this.model.chat(message);
    const tokensUsed = response.tokens.input + response.tokens.output;
    
    // Consume tokens
    await this.rateLimiter.consumeTokens(userId, tier, tokensUsed);
    
    // Get remaining quota
    const remaining = await this.rateLimiter.getRemainingQuota(userId, tier);
    
    return {
      response: response.content,
      tokensUsed: tokensUsed,
      quotaRemaining: remaining
    };
  }
  
  async getQuota(userId, tier) {
    return await this.rateLimiter.getRemainingQuota(userId, tier);
  }
  
  estimateTokens(text) {
    return Math.ceil(text.length / 4) * 2; // Input + output estimate
  }
}

module.exports = { LLMRateLimitService, ENBDRateLimitedChatService };
```

---

### Q19. How do you implement A/B testing and experimentation for LLM prompts?

**Answer:**

```javascript
const { ChatOpenAI } = require('@langchain/openai');

/**
 * A/B Testing Service for LLM Prompts
 */
class LLMABTestingService {
  constructor() {
    this.experiments = new Map();
    this.results = new Map();
  }
  
  /**
   * Create new experiment
   */
  createExperiment(experimentId, config) {
    this.experiments.set(experimentId, {
      id: experimentId,
      name: config.name,
      variants: config.variants, // { A: {...}, B: {...} }
      allocation: config.allocation || { A: 50, B: 50 }, // Traffic split %
      metrics: config.metrics || ['response_time', 'token_count', 'quality_score'],
      startDate: new Date(),
      status: 'active'
    });
    
    // Initialize results tracking
    this.results.set(experimentId, {
      variants: Object.keys(config.variants).reduce((acc, variant) => {
        acc[variant] = {
          requests: 0,
          totalResponseTime: 0,
          totalTokens: 0,
          qualityScores: [],
          errors: 0
        };
        return acc;
      }, {})
    });
  }
  
  /**
   * Assign user to variant
   */
  assignVariant(userId, experimentId) {
    const experiment = this.experiments.get(experimentId);
    
    if (!experiment || experiment.status !== 'active') {
      return null;
    }
    
    // Consistent hashing for stable assignment
    const hash = this.hashUserId(userId, experimentId);
    const allocation = experiment.allocation;
    
    let cumulativeAllocation = 0;
    for (const [variant, percentage] of Object.entries(allocation)) {
      cumulativeAllocation += percentage;
      if (hash < cumulativeAllocation) {
        return variant;
      }
    }
    
    return Object.keys(allocation)[0]; // Fallback
  }
  
  /**
   * Hash user ID for consistent variant assignment
   */
  hashUserId(userId, experimentId) {
    const str = `${userId}:${experimentId}`;
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i);
      hash |= 0;
    }
    return Math.abs(hash) % 100;
  }
  
  /**
   * Execute variant and track results
   */
  async executeVariant(userId, experimentId, input) {
    const experiment = this.experiments.get(experimentId);
    const variant = this.assignVariant(userId, experimentId);
    
    if (!variant) {
      throw new Error('No active experiment');
    }
    
    const variantConfig = experiment.variants[variant];
    const startTime = Date.now();
    
    try {
      // Execute variant
      const result = await this.runVariant(variantConfig, input);
      const responseTime = Date.now() - startTime;
      
      // Track metrics
      this.trackMetrics(experimentId, variant, {
        responseTime,
        tokens: result.tokens,
        error: false
      });
      
      return {
        variant: variant,
        result: result,
        responseTime: responseTime
      };
    } catch (error) {
      this.trackMetrics(experimentId, variant, { error: true });
      throw error;
    }
  }
  
  /**
   * Run specific variant
   */
  async runVariant(variantConfig, input) {
    const model = new ChatOpenAI({
      modelName: variantConfig.model || 'gpt-4',
      temperature: variantConfig.temperature || 0.7,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    // Build prompt from variant template
    const prompt = variantConfig.promptTemplate.replace('{input}', input);
    
    const response = await model.call(prompt);
    
    return {
      content: response.content,
      tokens: this.countTokens(prompt + response.content),
      config: variantConfig
    };
  }
  
  /**
   * Track metrics for variant
   */
  trackMetrics(experimentId, variant, metrics) {
    const results = this.results.get(experimentId);
    const variantResults = results.variants[variant];
    
    variantResults.requests++;
    
    if (metrics.error) {
      variantResults.errors++;
    } else {
      variantResults.totalResponseTime += metrics.responseTime;
      variantResults.totalTokens += metrics.tokens;
    }
  }
  
  /**
   * Add quality score (from user feedback)
   */
  addQualityScore(experimentId, variant, score) {
    const results = this.results.get(experimentId);
    results.variants[variant].qualityScores.push(score);
  }
  
  /**
   * Get experiment results
   */
  getResults(experimentId) {
    const experiment = this.experiments.get(experimentId);
    const results = this.results.get(experimentId);
    
    const analysis = {};
    
    for (const [variant, data] of Object.entries(results.variants)) {
      const avgResponseTime = data.totalResponseTime / data.requests;
      const avgTokens = data.totalTokens / data.requests;
      const avgQuality = data.qualityScores.length > 0
        ? data.qualityScores.reduce((a, b) => a + b, 0) / data.qualityScores.length
        : null;
      const errorRate = (data.errors / data.requests) * 100;
      
      analysis[variant] = {
        requests: data.requests,
        avgResponseTime: avgResponseTime.toFixed(2),
        avgTokens: avgTokens.toFixed(0),
        avgQualityScore: avgQuality ? avgQuality.toFixed(2) : 'N/A',
        errorRate: errorRate.toFixed(2) + '%'
      };
    }
    
    return {
      experiment: experiment,
      results: analysis,
      winner: this.determineWinner(analysis)
    };
  }
  
  /**
   * Determine winning variant
   */
  determineWinner(analysis) {
    let bestVariant = null;
    let bestScore = -Infinity;
    
    for (const [variant, metrics] of Object.entries(analysis)) {
      // Composite score: quality (most important) - latency penalty
      const qualityScore = parseFloat(metrics.avgQualityScore) || 0;
      const latencyPenalty = parseFloat(metrics.avgResponseTime) / 1000;
      const score = qualityScore - (latencyPenalty * 0.1);
      
      if (score > bestScore) {
        bestScore = score;
        bestVariant = variant;
      }
    }
    
    return bestVariant;
  }
  
  /**
   * Stop experiment
   */
  stopExperiment(experimentId) {
    const experiment = this.experiments.get(experimentId);
    if (experiment) {
      experiment.status = 'stopped';
      experiment.endDate = new Date();
    }
  }
  
  countTokens(text) {
    return Math.ceil(text.length / 4);
  }
}

/**
 * ENBD A/B Testing Chat Service
 */
class ENBDAB TestChatService {
  constructor() {
    this.abTesting = new LLMABTestingService();
    this.setupExperiments();
  }
  
  setupExperiments() {
    // Experiment 1: Prompt style
    this.abTesting.createExperiment('prompt_style_test', {
      name: 'Banking Assistant Prompt Style',
      variants: {
        A: {
          model: 'gpt-4',
          temperature: 0.7,
          promptTemplate: `You are a helpful banking assistant for Emirates NBD.

Customer question: {input}

Provide a clear, professional answer:`
        },
        B: {
          model: 'gpt-4',
          temperature: 0.7,
          promptTemplate: `You are an expert banking advisor at Emirates NBD with 10+ years of experience.

Guidelines:
- Be professional and empathetic
- Provide accurate information only
- Offer specific product details when relevant

Customer: {input}

Expert Answer:`
        }
      },
      allocation: { A: 50, B: 50 }
    });
  }
  
  async chat(userId, message) {
    const result = await this.abTesting.executeVariant(
      userId,
      'prompt_style_test',
      message
    );
    
    return {
      response: result.result.content,
      variant: result.variant,
      responseTime: result.responseTime
    };
  }
  
  async submitFeedback(userId, experimentId, variant, score) {
    this.abTesting.addQualityScore(experimentId, variant, score);
  }
  
  async getExperimentResults(experimentId) {
    return this.abTesting.getResults(experimentId);
  }
}

module.exports = { LLMABTestingService, ENBDABTestChatService };
```

---

### Q20. How do you implement model versioning and deployment strategies?

**Answer:**

```javascript
/**
 * Model Versioning and Deployment Service
 */
class LLMVersioningService {
  constructor() {
    this.models = new Map();
    this.deployments = new Map();
  }
  
  /**
   * Register model version
   */
  registerModel(modelId, version, config) {
    const key = `${modelId}:${version}`;
    
    this.models.set(key, {
      modelId,
      version,
      config,
      registeredAt: new Date(),
      status: 'registered',
      metadata: config.metadata || {}
    });
  }
  
  /**
   * Deploy model with strategy
   */
  async deployModel(modelId, version, strategy = 'blue-green') {
    const modelKey = `${modelId}:${version}`;
    const model = this.models.get(modelKey);
    
    if (!model) {
      throw new Error(`Model ${modelKey} not found`);
    }
    
    const deployment = {
      modelId,
      version,
      strategy,
      trafficAllocation: this.getInitialTraffic(strategy),
      deployedAt: new Date(),
      status: 'deploying'
    };
    
    this.deployments.set(modelId, deployment);
    
    // Execute deployment strategy
    await this.executeDeploymentStrategy(modelId, version, strategy);
    
    deployment.status = 'active';
    
    return deployment;
  }
  
  /**
   * Get initial traffic allocation based on strategy
   */
  getInitialTraffic(strategy) {
    switch (strategy) {
      case 'blue-green':
        return { old: 100, new: 0 }; // Start with 100% on old
      case 'canary':
        return { old: 95, new: 5 }; // Start with 5% on new
      case 'rolling':
        return { old: 80, new: 20 }; // Gradual rollout
      default:
        return { old: 0, new: 100 }; // Immediate switch
    }
  }
  
  /**
   * Execute deployment strategy
   */
  async executeDeploymentStrategy(modelId, version, strategy) {
    switch (strategy) {
      case 'blue-green':
        await this.blueGreenDeployment(modelId, version);
        break;
      case 'canary':
        await this.canaryDeployment(modelId, version);
        break;
      case 'rolling':
        await this.rollingDeployment(modelId, version);
        break;
    }
  }
  
  /**
   * Blue-Green Deployment
   * 1. Deploy new version (green)
   * 2. Test green environment
   * 3. Switch traffic from blue to green
   * 4. Keep blue as backup
   */
  async blueGreenDeployment(modelId, version) {
    console.log(`Blue-Green deployment for ${modelId}:${version}`);
    
    // Deploy green environment
    console.log('1. Deploying green environment...');
    await this.sleep(1000);
    
    // Run health checks on green
    console.log('2. Testing green environment...');
    const healthCheck = await this.runHealthCheck(modelId, version);
    
    if (!healthCheck.healthy) {
      throw new Error('Health check failed on green environment');
    }
    
    // Switch traffic
    console.log('3. Switching traffic to green...');
    this.updateTrafficAllocation(modelId, { old: 0, new: 100 });
    
    console.log('✅ Blue-Green deployment complete');
  }
  
  /**
   * Canary Deployment
   * 1. Deploy new version to small % of traffic
   * 2. Monitor metrics
   * 3. Gradually increase traffic if metrics good
   * 4. Rollback if issues detected
   */
  async canaryDeployment(modelId, version) {
    console.log(`Canary deployment for ${modelId}:${version}`);
    
    const stages = [5, 25, 50, 100];
    
    for (const percentage of stages) {
      console.log(`Deploying to ${percentage}% of traffic...`);
      
      this.updateTrafficAllocation(modelId, {
        old: 100 - percentage,
        new: percentage
      });
      
      // Monitor for issues
      await this.sleep(2000);
      const metrics = await this.monitorMetrics(modelId, version);
      
      if (!metrics.healthy) {
        console.log('⚠️  Issues detected! Rolling back...');
        await this.rollback(modelId);
        throw new Error('Canary deployment failed');
      }
      
      console.log(`✅ ${percentage}% deployment successful`);
    }
    
    console.log('✅ Canary deployment complete');
  }
  
  /**
   * Rolling Deployment
   * Gradually replace instances
   */
  async rollingDeployment(modelId, version) {
    console.log(`Rolling deployment for ${modelId}:${version}`);
    
    const batches = [20, 40, 60, 80, 100];
    
    for (const percentage of batches) {
      console.log(`Rolling out to ${percentage}%...`);
      
      this.updateTrafficAllocation(modelId, {
        old: 100 - percentage,
        new: percentage
      });
      
      await this.sleep(1000);
      
      console.log(`✅ ${percentage}% rolled out`);
    }
    
    console.log('✅ Rolling deployment complete');
  }
  
  /**
   * Update traffic allocation
   */
  updateTrafficAllocation(modelId, allocation) {
    const deployment = this.deployments.get(modelId);
    if (deployment) {
      deployment.trafficAllocation = allocation;
    }
  }
  
  /**
   * Rollback to previous version
   */
  async rollback(modelId) {
    console.log(`Rolling back ${modelId}...`);
    
    const deployment = this.deployments.get(modelId);
    if (deployment) {
      deployment.trafficAllocation = { old: 100, new: 0 };
      deployment.status = 'rolled_back';
    }
    
    console.log('✅ Rollback complete');
  }
  
  /**
   * Health check
   */
  async runHealthCheck(modelId, version) {
    // Simulate health check
    await this.sleep(500);
    
    return {
      healthy: true,
      checks: {
        apiAvailable: true,
        responseTime: 120,
        errorRate: 0.1
      }
    };
  }
  
  /**
   * Monitor metrics during deployment
   */
  async monitorMetrics(modelId, version) {
    // Simulate monitoring
    await this.sleep(500);
    
    return {
      healthy: true,
      errorRate: 0.5,
      latency: 150,
      throughput: 100
    };
  }
  
  /**
   * Get current deployment status
   */
  getDeploymentStatus(modelId) {
    return this.deployments.get(modelId);
  }
  
  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

/**
 * ENBD Model Deployment Manager
 */
class ENBDModelDeploymentManager {
  constructor() {
    this.versioning = new LLMVersioningService();
  }
  
  async deployNewModel(modelId, version, strategy) {
    console.log(`\n=== Deploying ${modelId}:${version} ===`);
    console.log(`Strategy: ${strategy}\n`);
    
    // Register model
    this.versioning.registerModel(modelId, version, {
      model: 'gpt-4',
      temperature: 0.7,
      metadata: {
        description: 'ENBD Banking Assistant',
        trained: new Date()
      }
    });
    
    // Deploy with chosen strategy
    const deployment = await this.versioning.deployModel(modelId, version, strategy);
    
    console.log('\nDeployment Status:', deployment);
    
    return deployment;
  }
  
  async checkDeployment(modelId) {
    return this.versioning.getDeploymentStatus(modelId);
  }
  
  async rollback(modelId) {
    return await this.versioning.rollback(modelId);
  }
}

// Usage Example
async function exampleDeployment() {
  const manager = new ENBDModelDeploymentManager();
  
  // Blue-Green deployment
  await manager.deployNewModel('enbd-assistant', 'v2.0', 'blue-green');
  
  // Canary deployment
  await manager.deployNewModel('enbd-assistant', 'v2.1', 'canary');
  
  // Check status
  const status = await manager.checkDeployment('enbd-assistant');
  console.log('Current status:', status);
}

module.exports = { LLMVersioningService, ENBDModelDeploymentManager };
```

---

**Summary Q16-Q20:**
- Error handling with circuit breakers, retry logic, fallback chains ✅
- Monitoring with Prometheus metrics, Winston logging, health checks ✅
- Rate limiting with Redis, tier-based quotas, priority queues ✅
- A/B testing for prompt optimization and experimentation ✅
- Model versioning with blue-green, canary, rolling deployments ✅
