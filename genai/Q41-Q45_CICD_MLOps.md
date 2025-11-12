# GenAI Questions 41-45: CI/CD & MLOps

---

### Q41. How do you implement MLOps for LLM applications?

**Answer:**

MLOps brings DevOps practices to machine learning workflows - version control, experiment tracking, and automated deployments.

```javascript
const { ChatOpenAI } = require('@langchain/openai');
const mlflow = require('mlflow');
const fs = require('fs').promises;

/**
 * MLFlow Integration for LLM Tracking
 */
class MLFlowTracker {
  constructor() {
    this.mlflow = mlflow;
    this.experimentName = 'enbd-banking-chat';
  }
  
  /**
   * Initialize experiment
   */
  async initializeExperiment() {
    try {
      const experiment = await this.mlflow.getExperiment({ name: this.experimentName });
      this.experimentId = experiment.experiment_id;
    } catch (error) {
      const experiment = await this.mlflow.createExperiment({ name: this.experimentName });
      this.experimentId = experiment.experiment_id;
    }
    
    return this.experimentId;
  }
  
  /**
   * Track LLM run
   */
  async trackRun(runName, params, metrics, artifacts = {}) {
    await this.mlflow.startRun({
      experiment_id: this.experimentId,
      run_name: runName
    });
    
    // Log parameters
    for (const [key, value] of Object.entries(params)) {
      await this.mlflow.logParam({ key, value: String(value) });
    }
    
    // Log metrics
    for (const [key, value] of Object.entries(metrics)) {
      await this.mlflow.logMetric({ key, value });
    }
    
    // Log artifacts
    for (const [name, content] of Object.entries(artifacts)) {
      const artifactPath = `/tmp/${name}`;
      await fs.writeFile(artifactPath, JSON.stringify(content, null, 2));
      await this.mlflow.logArtifact({ local_path: artifactPath });
    }
    
    await this.mlflow.endRun();
    
    return { experiment_id: this.experimentId, run_name: runName };
  }
  
  /**
   * Compare runs
   */
  async compareRuns(runIds) {
    const runs = await Promise.all(
      runIds.map(id => this.mlflow.getRun({ run_id: id }))
    );
    
    return runs.map(run => ({
      run_id: run.info.run_id,
      params: run.data.params,
      metrics: run.data.metrics,
      start_time: run.info.start_time
    }));
  }
  
  /**
   * Get best run
   */
  async getBestRun(metricName = 'accuracy') {
    const runs = await this.mlflow.searchRuns({
      experiment_ids: [this.experimentId],
      order_by: [`metrics.${metricName} DESC`],
      max_results: 1
    });
    
    return runs[0];
  }
}

/**
 * Model Versioning with DVC
 */
class ModelVersionControl {
  constructor() {
    this.modelsDir = './models';
    this.versionsFile = './models/versions.json';
  }
  
  /**
   * Save model version
   */
  async saveVersion(modelName, version, config) {
    const versionData = {
      name: modelName,
      version,
      timestamp: new Date().toISOString(),
      config,
      hash: this.generateHash(config)
    };
    
    // Save version metadata
    const versions = await this.loadVersions();
    versions.push(versionData);
    await fs.writeFile(this.versionsFile, JSON.stringify(versions, null, 2));
    
    // Save model config
    const modelPath = `${this.modelsDir}/${modelName}_v${version}.json`;
    await fs.writeFile(modelPath, JSON.stringify(config, null, 2));
    
    return versionData;
  }
  
  /**
   * Load specific version
   */
  async loadVersion(modelName, version) {
    const modelPath = `${this.modelsDir}/${modelName}_v${version}.json`;
    const config = JSON.parse(await fs.readFile(modelPath, 'utf-8'));
    
    return config;
  }
  
  /**
   * List all versions
   */
  async loadVersions() {
    try {
      const data = await fs.readFile(this.versionsFile, 'utf-8');
      return JSON.parse(data);
    } catch (error) {
      return [];
    }
  }
  
  /**
   * Get latest version
   */
  async getLatestVersion(modelName) {
    const versions = await this.loadVersions();
    const modelVersions = versions.filter(v => v.name === modelName);
    
    return modelVersions[modelVersions.length - 1];
  }
  
  generateHash(data) {
    const crypto = require('crypto');
    return crypto.createHash('sha256').update(JSON.stringify(data)).digest('hex').substring(0, 8);
  }
}

/**
 * ENBD MLOps Service
 */
class ENBDMLOpsService {
  constructor() {
    this.tracker = new MLFlowTracker();
    this.versionControl = new ModelVersionControl();
  }
  
  /**
   * Train and track new model
   */
  async trainAndTrack(modelConfig) {
    await this.tracker.initializeExperiment();
    
    const startTime = Date.now();
    
    // Initialize model
    const model = new ChatOpenAI({
      modelName: modelConfig.modelName,
      temperature: modelConfig.temperature,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    // Run evaluation
    const metrics = await this.evaluateModel(model, modelConfig.testSet);
    
    const duration = Date.now() - startTime;
    
    // Track in MLFlow
    await this.tracker.trackRun(
      `${modelConfig.modelName}_${Date.now()}`,
      {
        model: modelConfig.modelName,
        temperature: modelConfig.temperature,
        max_tokens: modelConfig.maxTokens || 'default'
      },
      {
        accuracy: metrics.accuracy,
        relevance: metrics.relevance,
        latency_ms: duration,
        cost_per_request: metrics.avgCost
      },
      {
        'test_results.json': metrics.details
      }
    );
    
    // Save version
    const version = await this.versionControl.saveVersion(
      modelConfig.modelName,
      modelConfig.version,
      modelConfig
    );
    
    return {
      version,
      metrics,
      duration
    };
  }
  
  /**
   * Evaluate model
   */
  async evaluateModel(model, testSet) {
    const results = [];
    let totalCost = 0;
    
    for (const testCase of testSet) {
      const startTime = Date.now();
      
      const response = await model.call(testCase.query);
      
      const latency = Date.now() - startTime;
      const cost = this.estimateCost(response.content.length);
      totalCost += cost;
      
      results.push({
        query: testCase.query,
        response: response.content,
        expected: testCase.expected,
        correct: this.checkCorrectness(response.content, testCase.expected),
        latency
      });
    }
    
    const correctCount = results.filter(r => r.correct).length;
    
    return {
      accuracy: correctCount / results.length,
      relevance: 0.85, // Would use LLM judge
      avgCost: totalCost / results.length,
      details: results
    };
  }
  
  checkCorrectness(response, expected) {
    return response.toLowerCase().includes(expected.toLowerCase());
  }
  
  estimateCost(outputTokens) {
    return (outputTokens / 1000) * 0.03; // $0.03 per 1K tokens
  }
  
  /**
   * Promote model to production
   */
  async promoteToProduction(modelName, version) {
    const config = await this.versionControl.loadVersion(modelName, version);
    
    // Save as production version
    await fs.writeFile(
      './models/production.json',
      JSON.stringify({
        model: modelName,
        version,
        promoted_at: new Date().toISOString(),
        config
      }, null, 2)
    );
    
    return {
      model: modelName,
      version,
      status: 'production'
    };
  }
}

module.exports = { MLFlowTracker, ModelVersionControl, ENBDMLOpsService };
```

---

### Q42. How do you implement experiment tracking and A/B testing?

**Answer:**

```javascript
const { ChatOpenAI } = require('@langchain/openai');
const Redis = require('ioredis');

/**
 * Experiment Tracking Service
 */
class ExperimentTracker {
  constructor() {
    this.redis = new Redis();
    this.experiments = new Map();
  }
  
  /**
   * Create experiment
   */
  async createExperiment(experimentConfig) {
    const experiment = {
      id: `exp_${Date.now()}`,
      name: experimentConfig.name,
      variants: experimentConfig.variants,
      trafficSplit: experimentConfig.trafficSplit || {},
      metrics: {},
      startedAt: new Date().toISOString(),
      status: 'active'
    };
    
    // Store in Redis
    await this.redis.set(
      `experiment:${experiment.id}`,
      JSON.stringify(experiment)
    );
    
    this.experiments.set(experiment.id, experiment);
    
    return experiment;
  }
  
  /**
   * Assign user to variant
   */
  async assignVariant(experimentId, userId) {
    const experiment = await this.getExperiment(experimentId);
    
    // Check if user already assigned
    const existingAssignment = await this.redis.get(
      `assignment:${experimentId}:${userId}`
    );
    
    if (existingAssignment) {
      return existingAssignment;
    }
    
    // Assign based on hash
    const variantIndex = this.hashUserId(userId) % experiment.variants.length;
    const variant = experiment.variants[variantIndex];
    
    // Store assignment
    await this.redis.set(
      `assignment:${experimentId}:${userId}`,
      variant.name,
      'EX',
      86400 * 30 // 30 days
    );
    
    return variant.name;
  }
  
  /**
   * Track metric
   */
  async trackMetric(experimentId, variantName, metricName, value) {
    const key = `metrics:${experimentId}:${variantName}:${metricName}`;
    
    // Append to list
    await this.redis.rpush(key, value);
    
    // Keep last 10000 values
    await this.redis.ltrim(key, -10000, -1);
  }
  
  /**
   * Get experiment results
   */
  async getResults(experimentId) {
    const experiment = await this.getExperiment(experimentId);
    const results = {};
    
    for (const variant of experiment.variants) {
      const metrics = {};
      
      for (const metricName of ['latency', 'accuracy', 'satisfaction']) {
        const key = `metrics:${experimentId}:${variant.name}:${metricName}`;
        const values = await this.redis.lrange(key, 0, -1);
        
        if (values.length > 0) {
          const numericValues = values.map(Number);
          metrics[metricName] = {
            count: values.length,
            mean: this.mean(numericValues),
            stddev: this.stddev(numericValues),
            min: Math.min(...numericValues),
            max: Math.max(...numericValues)
          };
        }
      }
      
      results[variant.name] = metrics;
    }
    
    return {
      experiment_id: experimentId,
      experiment_name: experiment.name,
      results,
      winner: this.determineWinner(results)
    };
  }
  
  /**
   * Determine winner
   */
  determineWinner(results) {
    const variants = Object.keys(results);
    let bestVariant = null;
    let bestScore = -Infinity;
    
    for (const variant of variants) {
      const metrics = results[variant];
      
      // Composite score: higher accuracy, lower latency
      const score = 
        (metrics.accuracy?.mean || 0) * 100 - 
        (metrics.latency?.mean || 1000) / 10;
      
      if (score > bestScore) {
        bestScore = score;
        bestVariant = variant;
      }
    }
    
    return {
      variant: bestVariant,
      score: bestScore
    };
  }
  
  async getExperiment(experimentId) {
    const data = await this.redis.get(`experiment:${experimentId}`);
    return JSON.parse(data);
  }
  
  hashUserId(userId) {
    const crypto = require('crypto');
    return parseInt(crypto.createHash('md5').update(userId).digest('hex').substring(0, 8), 16);
  }
  
  mean(arr) {
    return arr.reduce((a, b) => a + b, 0) / arr.length;
  }
  
  stddev(arr) {
    const avg = this.mean(arr);
    const squareDiffs = arr.map(value => Math.pow(value - avg, 2));
    return Math.sqrt(this.mean(squareDiffs));
  }
}

/**
 * A/B Test Manager
 */
class ABTestManager {
  constructor() {
    this.tracker = new ExperimentTracker();
  }
  
  /**
   * Setup prompt A/B test
   */
  async setupPromptTest(testName, prompts) {
    const experiment = await this.tracker.createExperiment({
      name: testName,
      variants: prompts.map((prompt, idx) => ({
        name: `variant_${String.fromCharCode(65 + idx)}`, // A, B, C...
        prompt: prompt,
        config: { temperature: 0.7 }
      })),
      trafficSplit: this.evenSplit(prompts.length)
    });
    
    return experiment;
  }
  
  evenSplit(count) {
    const split = {};
    const percentage = 100 / count;
    
    for (let i = 0; i < count; i++) {
      split[`variant_${String.fromCharCode(65 + i)}`] = percentage;
    }
    
    return split;
  }
  
  /**
   * Run experiment
   */
  async runExperiment(experimentId, userId, query) {
    // Assign variant
    const variantName = await this.tracker.assignVariant(experimentId, userId);
    
    // Get experiment config
    const experiment = await this.tracker.getExperiment(experimentId);
    const variant = experiment.variants.find(v => v.name === variantName);
    
    // Execute with variant
    const model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: variant.config.temperature,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    const startTime = Date.now();
    const fullPrompt = variant.prompt.replace('{query}', query);
    const response = await model.call(fullPrompt);
    const latency = Date.now() - startTime;
    
    // Track metrics
    await this.tracker.trackMetric(experimentId, variantName, 'latency', latency);
    
    return {
      variant: variantName,
      response: response.content,
      latency
    };
  }
}

/**
 * ENBD Experiment Service
 */
class ENBDExperimentService {
  constructor() {
    this.abTest = new ABTestManager();
  }
  
  /**
   * Test different system prompts
   */
  async testSystemPrompts() {
    const prompts = [
      {
        name: 'variant_A',
        prompt: 'You are a professional Emirates NBD banking assistant. {query}'
      },
      {
        name: 'variant_B',
        prompt: 'You are an expert financial advisor at Emirates NBD. Use simple language. {query}'
      },
      {
        name: 'variant_C',
        prompt: 'You are a friendly Emirates NBD helper. Be concise and helpful. {query}'
      }
    ];
    
    const experiment = await this.abTest.setupPromptTest('system_prompt_test', prompts);
    
    console.log('Experiment created:', experiment.id);
    
    return experiment;
  }
  
  /**
   * Analyze experiment results
   */
  async analyzeResults(experimentId) {
    const results = await this.abTest.tracker.getResults(experimentId);
    
    console.log('\n=== Experiment Results ===');
    console.log(`Experiment: ${results.experiment_name}`);
    console.log(`\nWinner: ${results.winner.variant} (score: ${results.winner.score.toFixed(2)})`);
    
    console.log('\nDetailed Metrics:');
    for (const [variant, metrics] of Object.entries(results.results)) {
      console.log(`\n${variant}:`);
      console.log(`  Latency: ${metrics.latency?.mean.toFixed(0)}ms (±${metrics.latency?.stddev.toFixed(0)})`);
      console.log(`  Accuracy: ${((metrics.accuracy?.mean || 0) * 100).toFixed(1)}%`);
      console.log(`  Samples: ${metrics.latency?.count || 0}`);
    }
    
    return results;
  }
}

module.exports = { ExperimentTracker, ABTestManager, ENBDExperimentService };
```

---

### Q43. How do you implement automated deployment pipelines?

**Answer:**

```javascript
const { exec } = require('child_process');
const util = require('util');
const execPromise = util.promisify(exec);

/**
 * Deployment Pipeline Manager
 */
class DeploymentPipeline {
  constructor() {
    this.stages = [
      'validate',
      'build',
      'test',
      'security_scan',
      'staging',
      'production'
    ];
    
    this.currentStage = 0;
  }
  
  /**
   * Run full deployment pipeline
   */
  async deploy(version) {
    const results = [];
    
    for (const stage of this.stages) {
      console.log(`\n=== Stage: ${stage} ===`);
      
      const result = await this.runStage(stage, version);
      results.push(result);
      
      if (!result.success) {
        console.error(`Stage ${stage} failed. Aborting deployment.`);
        await this.rollback(version);
        return { success: false, failedAt: stage, results };
      }
    }
    
    return { success: true, results };
  }
  
  /**
   * Run individual stage
   */
  async runStage(stage, version) {
    const startTime = Date.now();
    
    try {
      switch (stage) {
        case 'validate':
          await this.validateCode();
          break;
        case 'build':
          await this.buildApplication(version);
          break;
        case 'test':
          await this.runTests();
          break;
        case 'security_scan':
          await this.runSecurityScan();
          break;
        case 'staging':
          await this.deployToStaging(version);
          break;
        case 'production':
          await this.deployToProduction(version);
          break;
      }
      
      return {
        stage,
        success: true,
        duration: Date.now() - startTime
      };
    } catch (error) {
      return {
        stage,
        success: false,
        error: error.message,
        duration: Date.now() - startTime
      };
    }
  }
  
  /**
   * Validate code
   */
  async validateCode() {
    console.log('Running ESLint...');
    await execPromise('npm run lint');
    
    console.log('Checking types...');
    // await execPromise('npm run type-check');
  }
  
  /**
   * Build application
   */
  async buildApplication(version) {
    console.log(`Building version ${version}...`);
    
    // Build Docker image
    await execPromise(`docker build -t enbd-ai-service:${version} .`);
    
    // Tag as latest
    await execPromise(`docker tag enbd-ai-service:${version} enbd-ai-service:latest`);
  }
  
  /**
   * Run tests
   */
  async runTests() {
    console.log('Running test suite...');
    
    const { stdout } = await execPromise('npm test');
    console.log(stdout);
  }
  
  /**
   * Security scan
   */
  async runSecurityScan() {
    console.log('Running security scan...');
    
    // Scan for vulnerabilities
    await execPromise('npm audit --audit-level=high');
    
    // Container scan
    // await execPromise('docker scan enbd-ai-service:latest');
  }
  
  /**
   * Deploy to staging
   */
  async deployToStaging(version) {
    console.log('Deploying to staging...');
    
    // Update Kubernetes deployment
    await execPromise(`kubectl set image deployment/ai-service-staging ai-service=enbd-ai-service:${version} -n staging`);
    
    // Wait for rollout
    await execPromise('kubectl rollout status deployment/ai-service-staging -n staging');
    
    // Run smoke tests
    await this.runSmokeTests('https://staging-api.enbd.com');
  }
  
  /**
   * Deploy to production
   */
  async deployToProduction(version) {
    console.log('Deploying to production...');
    
    // Canary deployment: 10% traffic
    await execPromise(`kubectl set image deployment/ai-service-canary ai-service=enbd-ai-service:${version} -n production`);
    
    console.log('Canary deployed. Monitoring...');
    await this.sleep(300000); // 5 minutes
    
    // Check canary metrics
    const canaryHealthy = await this.checkCanaryHealth();
    
    if (!canaryHealthy) {
      throw new Error('Canary metrics show degradation');
    }
    
    // Full rollout
    await execPromise(`kubectl set image deployment/ai-service-prod ai-service=enbd-ai-service:${version} -n production`);
    await execPromise('kubectl rollout status deployment/ai-service-prod -n production');
    
    console.log('Production deployment complete!');
  }
  
  /**
   * Smoke tests
   */
  async runSmokeTests(baseUrl) {
    const tests = [
      { endpoint: '/health', expectedStatus: 200 },
      { endpoint: '/api/chat', method: 'POST', expectedStatus: 200 }
    ];
    
    for (const test of tests) {
      const response = await fetch(`${baseUrl}${test.endpoint}`);
      
      if (response.status !== test.expectedStatus) {
        throw new Error(`Smoke test failed: ${test.endpoint}`);
      }
    }
    
    console.log('Smoke tests passed ✓');
  }
  
  /**
   * Check canary health
   */
  async checkCanaryHealth() {
    // Check error rate, latency from monitoring
    // Implementation would query Prometheus/Datadog
    
    return true; // Placeholder
  }
  
  /**
   * Rollback deployment
   */
  async rollback(version) {
    console.log('Rolling back deployment...');
    
    await execPromise('kubectl rollout undo deployment/ai-service-prod -n production');
    
    console.log('Rollback complete');
  }
  
  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

/**
 * GitHub Actions Deployment
 */
class GitHubActionsDeployment {
  /**
   * Generate deployment workflow
   */
  generateWorkflow() {
    return `
name: Deploy AI Service

on:
  push:
    tags:
      - 'v*'

env:
  DOCKER_REGISTRY: registry.enbd.com
  IMAGE_NAME: enbd-ai-service

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Extract version
        id: version
        run: echo "VERSION=\${GITHUB_REF#refs/tags/}" >> $GITHUB_OUTPUT
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
      
      - name: Build Docker image
        run: |
          docker build -t \${{ env.DOCKER_REGISTRY }}/\${{ env.IMAGE_NAME }}:\${{ steps.version.outputs.VERSION }} .
          docker tag \${{ env.DOCKER_REGISTRY }}/\${{ env.IMAGE_NAME }}:\${{ steps.version.outputs.VERSION }} \${{ env.DOCKER_REGISTRY }}/\${{ env.IMAGE_NAME }}:latest
      
      - name: Push to registry
        run: |
          echo "\${{ secrets.DOCKER_PASSWORD }}" | docker login \${{ env.DOCKER_REGISTRY }} -u "\${{ secrets.DOCKER_USERNAME }}" --password-stdin
          docker push \${{ env.DOCKER_REGISTRY }}/\${{ env.IMAGE_NAME }}:\${{ steps.version.outputs.VERSION }}
          docker push \${{ env.DOCKER_REGISTRY }}/\${{ env.IMAGE_NAME }}:latest
      
      - name: Deploy to staging
        run: |
          kubectl config use-context staging
          kubectl set image deployment/ai-service ai-service=\${{ env.DOCKER_REGISTRY }}/\${{ env.IMAGE_NAME }}:\${{ steps.version.outputs.VERSION }} -n staging
          kubectl rollout status deployment/ai-service -n staging
      
      - name: Run smoke tests
        run: npm run test:smoke -- --url https://staging-api.enbd.com
      
      - name: Deploy to production
        if: success()
        run: |
          kubectl config use-context production
          kubectl set image deployment/ai-service ai-service=\${{ env.DOCKER_REGISTRY }}/\${{ env.IMAGE_NAME }}:\${{ steps.version.outputs.VERSION }} -n production
          kubectl rollout status deployment/ai-service -n production
      
      - name: Notify team
        if: always()
        uses: 8398a7/action-slack@v3
        with:
          status: \${{ job.status }}
          text: 'Deployment \${{ steps.version.outputs.VERSION }} \${{ job.status }}'
          webhook_url: \${{ secrets.SLACK_WEBHOOK }}
`;
  }
}

module.exports = { DeploymentPipeline, GitHubActionsDeployment };
```

---

### Q44. How do you implement model monitoring in production?

**Answer:**

```javascript
const prometheus = require('prom-client');
const { ChatOpenAI } = require('@langchain/openai');

/**
 * Production Model Monitor
 */
class ProductionModelMonitor {
  constructor() {
    this.register = new prometheus.Registry();
    
    // Define metrics
    this.modelInferenceCounter = new prometheus.Counter({
      name: 'model_inference_total',
      help: 'Total number of model inferences',
      labelNames: ['model', 'status'],
      registers: [this.register]
    });
    
    this.modelLatencyHistogram = new prometheus.Histogram({
      name: 'model_inference_duration_seconds',
      help: 'Model inference duration',
      labelNames: ['model'],
      buckets: [0.1, 0.5, 1, 2, 5, 10],
      registers: [this.register]
    });
    
    this.modelQualityGauge = new prometheus.Gauge({
      name: 'model_quality_score',
      help: 'Model quality score (0-1)',
      labelNames: ['model', 'metric'],
      registers: [this.register]
    });
    
    this.tokenUsageCounter = new prometheus.Counter({
      name: 'model_tokens_total',
      help: 'Total tokens used',
      labelNames: ['model', 'type'],
      registers: [this.register]
    });
    
    this.costGauge = new prometheus.Gauge({
      name: 'model_cost_dollars',
      help: 'Estimated cost in dollars',
      labelNames: ['model'],
      registers: [this.register]
    });
  }
  
  /**
   * Track inference
   */
  async trackInference(modelName, inferenceFunction) {
    const end = this.modelLatencyHistogram.startTimer({ model: modelName });
    
    try {
      const result = await inferenceFunction();
      
      this.modelInferenceCounter.inc({ model: modelName, status: 'success' });
      end();
      
      // Track tokens
      if (result.tokens) {
        this.tokenUsageCounter.inc(
          { model: modelName, type: 'prompt' },
          result.tokens.prompt
        );
        this.tokenUsageCounter.inc(
          { model: modelName, type: 'completion' },
          result.tokens.completion
        );
        
        // Estimate cost
        const cost = this.estimateCost(modelName, result.tokens);
        this.costGauge.set({ model: modelName }, cost);
      }
      
      return result;
    } catch (error) {
      this.modelInferenceCounter.inc({ model: modelName, status: 'error' });
      end();
      throw error;
    }
  }
  
  /**
   * Track quality metrics
   */
  trackQuality(modelName, metrics) {
    for (const [metricName, value] of Object.entries(metrics)) {
      this.modelQualityGauge.set(
        { model: modelName, metric: metricName },
        value
      );
    }
  }
  
  /**
   * Estimate cost
   */
  estimateCost(modelName, tokens) {
    const pricing = {
      'gpt-4': { prompt: 0.03, completion: 0.06 },
      'gpt-4-turbo': { prompt: 0.01, completion: 0.03 },
      'gpt-3.5-turbo': { prompt: 0.0015, completion: 0.002 }
    };
    
    const prices = pricing[modelName] || pricing['gpt-4'];
    
    return (
      (tokens.prompt / 1000) * prices.prompt +
      (tokens.completion / 1000) * prices.completion
    );
  }
  
  /**
   * Get metrics endpoint
   */
  getMetrics() {
    return this.register.metrics();
  }
}

/**
 * Drift Detection
 */
class DriftDetector {
  constructor() {
    this.baseline = null;
    this.window = [];
    this.windowSize = 100;
  }
  
  /**
   * Set baseline distribution
   */
  setBaseline(samples) {
    this.baseline = {
      mean: this.mean(samples),
      stddev: this.stddev(samples),
      samples: samples.length
    };
  }
  
  /**
   * Detect drift
   */
  detectDrift(newSample) {
    this.window.push(newSample);
    
    if (this.window.length > this.windowSize) {
      this.window.shift();
    }
    
    if (this.window.length < this.windowSize || !this.baseline) {
      return { drifted: false, confidence: 0 };
    }
    
    const currentMean = this.mean(this.window);
    const currentStddev = this.stddev(this.window);
    
    // Z-score test
    const zScore = Math.abs(
      (currentMean - this.baseline.mean) / this.baseline.stddev
    );
    
    const drifted = zScore > 2; // 2 standard deviations
    
    return {
      drifted,
      confidence: Math.min(zScore / 3, 1),
      currentMean,
      baselineMean: this.baseline.mean,
      zScore
    };
  }
  
  mean(arr) {
    return arr.reduce((a, b) => a + b, 0) / arr.length;
  }
  
  stddev(arr) {
    const avg = this.mean(arr);
    const squareDiffs = arr.map(value => Math.pow(value - avg, 2));
    return Math.sqrt(this.mean(squareDiffs));
  }
}

/**
 * ENBD Production Monitor
 */
class ENBDProductionMonitor {
  constructor() {
    this.monitor = new ProductionModelMonitor();
    this.driftDetector = new DriftDetector();
    
    // Set baseline from historical data
    this.driftDetector.setBaseline([0.85, 0.87, 0.86, 0.88, 0.84, 0.89]);
  }
  
  /**
   * Monitor chat request
   */
  async monitorChatRequest(query) {
    const model = new ChatOpenAI({
      modelName: 'gpt-4',
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    const result = await this.monitor.trackInference('gpt-4', async () => {
      const response = await model.call(query);
      
      return {
        response: response.content,
        tokens: {
          prompt: Math.ceil(query.length / 4),
          completion: Math.ceil(response.content.length / 4)
        }
      };
    });
    
    // Evaluate quality
    const quality = await this.evaluateQuality(query, result.response);
    
    // Track quality
    this.monitor.trackQuality('gpt-4', {
      relevance: quality.relevance,
      coherence: quality.coherence
    });
    
    // Check for drift
    const drift = this.driftDetector.detectDrift(quality.relevance);
    
    if (drift.drifted) {
      console.warn('Model drift detected!', drift);
      // Alert team
    }
    
    return result;
  }
  
  async evaluateQuality(query, response) {
    return {
      relevance: 0.85,
      coherence: 0.90
    };
  }
}

module.exports = { ProductionModelMonitor, DriftDetector, ENBDProductionMonitor };
```

---

### Q45. How do you implement infrastructure as code for AI services?

**Answer:**

```javascript
/**
 * Terraform Configuration Generator
 */
class TerraformConfig {
  /**
   * Generate main.tf
   */
  generateMainTF() {
    return `
# Provider configuration
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket = "enbd-terraform-state"
    key    = "ai-service/terraform.tfstate"
    region = "me-south-1"
  }
}

provider "aws" {
  region = var.aws_region
}

# ECS Cluster
resource "aws_ecs_cluster" "ai_service" {
  name = "enbd-ai-service-\${var.environment}"
  
  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

# Task Definition
resource "aws_ecs_task_definition" "ai_service" {
  family                   = "enbd-ai-service"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = var.task_cpu
  memory                   = var.task_memory
  execution_role_arn       = aws_iam_role.ecs_execution_role.arn
  
  container_definitions = jsonencode([
    {
      name      = "ai-service"
      image     = "\${var.docker_image}:\${var.image_tag}"
      essential = true
      
      portMappings = [
        {
          containerPort = 3000
          protocol      = "tcp"
        }
      ]
      
      environment = [
        {
          name  = "NODE_ENV"
          value = var.environment
        },
        {
          name  = "REDIS_URL"
          value = aws_elasticache_cluster.redis.cache_nodes[0].address
        }
      ]
      
      secrets = [
        {
          name      = "OPENAI_API_KEY"
          valueFrom = aws_secretsmanager_secret.openai_key.arn
        }
      ]
      
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = "/ecs/enbd-ai-service"
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "ecs"
        }
      }
    }
  ])
}

# ECS Service
resource "aws_ecs_service" "ai_service" {
  name            = "enbd-ai-service"
  cluster         = aws_ecs_cluster.ai_service.id
  task_definition = aws_ecs_task_definition.ai_service.arn
  desired_count   = var.service_count
  launch_type     = "FARGATE"
  
  network_configuration {
    subnets         = var.private_subnets
    security_groups = [aws_security_group.ai_service.id]
  }
  
  load_balancer {
    target_group_arn = aws_lb_target_group.ai_service.arn
    container_name   = "ai-service"
    container_port   = 3000
  }
}

# Application Load Balancer
resource "aws_lb" "ai_service" {
  name               = "enbd-ai-service-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = var.public_subnets
}

# Redis Cache
resource "aws_elasticache_cluster" "redis" {
  cluster_id           = "enbd-ai-cache"
  engine               = "redis"
  node_type            = "cache.t3.medium"
  num_cache_nodes      = 1
  parameter_group_name = "default.redis7"
  port                 = 6379
  subnet_group_name    = aws_elasticache_subnet_group.redis.name
}

# RDS PostgreSQL
resource "aws_db_instance" "postgres" {
  identifier        = "enbd-ai-db"
  engine            = "postgres"
  engine_version    = "15.3"
  instance_class    = "db.t3.medium"
  allocated_storage = 100
  
  db_name  = "enbd_ai"
  username = var.db_username
  password = var.db_password
  
  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.postgres.name
  
  backup_retention_period = 7
  skip_final_snapshot     = var.environment != "production"
}

# Secrets Manager
resource "aws_secretsmanager_secret" "openai_key" {
  name = "enbd-ai-service-openai-key-\${var.environment}"
}

# CloudWatch Log Group
resource "aws_cloudwatch_log_group" "ai_service" {
  name              = "/ecs/enbd-ai-service"
  retention_in_days = 30
}

# Auto Scaling
resource "aws_appautoscaling_target" "ecs_target" {
  max_capacity       = var.max_capacity
  min_capacity       = var.min_capacity
  resource_id        = "service/\${aws_ecs_cluster.ai_service.name}/\${aws_ecs_service.ai_service.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "ecs_policy_cpu" {
  name               = "cpu-autoscaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs_target.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs_target.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs_target.service_namespace
  
  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value = 70.0
  }
}
`;
  }
  
  /**
   * Generate variables.tf
   */
  generateVariablesTF() {
    return `
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "me-south-1"
}

variable "environment" {
  description = "Environment (dev/staging/production)"
  type        = string
}

variable "docker_image" {
  description = "Docker image name"
  type        = string
  default     = "registry.enbd.com/ai-service"
}

variable "image_tag" {
  description = "Docker image tag"
  type        = string
}

variable "task_cpu" {
  description = "ECS task CPU units"
  type        = string
  default     = "1024"
}

variable "task_memory" {
  description = "ECS task memory in MB"
  type        = string
  default     = "2048"
}

variable "service_count" {
  description = "Number of ECS tasks"
  type        = number
  default     = 2
}

variable "min_capacity" {
  description = "Minimum number of tasks"
  type        = number
  default     = 2
}

variable "max_capacity" {
  description = "Maximum number of tasks"
  type        = number
  default     = 10
}
`;
  }
  
  /**
   * Generate Kubernetes manifest
   */
  generateK8sManifest() {
    return `
apiVersion: apps/v1
kind: Deployment
metadata:
  name: enbd-ai-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: enbd-ai-service
  template:
    metadata:
      labels:
        app: enbd-ai-service
    spec:
      containers:
      - name: ai-service
        image: registry.enbd.com/ai-service:latest
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: redis-config
              key: url
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: openai-credentials
              key: api-key
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: enbd-ai-service
  namespace: production
spec:
  selector:
    app: enbd-ai-service
  ports:
  - port: 80
    targetPort: 3000
  type: LoadBalancer
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: enbd-ai-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: enbd-ai-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
`;
  }
}

module.exports = { TerraformConfig };
```

**CI/CD & MLOps:**

1. ✅ **MLOps** - Experiment tracking, model versioning ✅
2. ✅ **A/B Testing** - Variant assignment, metric tracking ✅
3. ✅ **Deployment Pipelines** - Automated staged deployments ✅
4. ✅ **Production Monitoring** - Metrics, drift detection ✅
5. ✅ **Infrastructure as Code** - Terraform, Kubernetes manifests ✅

---

**Summary Q41-Q45:**
- MLFlow integration for experiment tracking ✅
- A/B testing framework with variant assignment ✅
- Automated deployment pipelines with canary releases ✅
- Production monitoring with Prometheus metrics ✅
- Infrastructure as code with Terraform and Kubernetes ✅
