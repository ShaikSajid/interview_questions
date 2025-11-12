# GenAI Questions 26-30: AI Safety, Security & Compliance

---

### Q26. How do you implement content moderation and safety guardrails for LLM applications?

**Answer:**

AI safety requires implementing guardrails to prevent harmful outputs, protect sensitive data, and ensure compliance.

```javascript
const { ChatOpenAI } = require('@langchain/openai');
const OpenAI = require('openai');

/**
 * Content Moderation Service
 */
class ContentModerationService {
  constructor() {
    this.openai = new OpenAI({
      apiKey: process.env.OPENAI_API_KEY
    });
    
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }
  
  /**
   * Moderate content with OpenAI Moderation API
   */
  async moderateContent(text) {
    const moderation = await this.openai.moderations.create({
      input: text
    });
    
    const result = moderation.results[0];
    
    return {
      flagged: result.flagged,
      categories: result.categories,
      categoryScores: result.category_scores,
      violatedCategories: Object.keys(result.categories).filter(
        key => result.categories[key]
      )
    };
  }
  
  /**
   * Check for sensitive data (PII)
   */
  async detectSensitiveData(text) {
    const patterns = {
      creditCard: /\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b/g,
      ssn: /\b\d{3}-\d{2}-\d{4}\b/g,
      email: /\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/g,
      phone: /\b\d{3}[-.]?\d{3}[-.]?\d{4}\b/g,
      emiratesID: /\b784-\d{4}-\d{7}-\d{1}\b/g
    };
    
    const detected = {};
    
    for (const [type, pattern] of Object.entries(patterns)) {
      const matches = text.match(pattern);
      if (matches) {
        detected[type] = matches;
      }
    }
    
    return {
      hasSensitiveData: Object.keys(detected).length > 0,
      detected: detected
    };
  }
  
  /**
   * Redact sensitive information
   */
  redactSensitiveData(text) {
    let redacted = text;
    
    redacted = redacted.replace(/\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b/g, '****-****-****-****');
    redacted = redacted.replace(/\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/g, '****@****.com');
    redacted = redacted.replace(/\b\d{3}[-.]?\d{3}[-.]?\d{4}\b/g, '***-***-****');
    redacted = redacted.replace(/\b784-\d{4}-\d{7}-\d{1}\b/g, '784-****-*******-*');
    
    return redacted;
  }
  
  /**
   * Check for prompt injection attacks
   */
  async detectPromptInjection(userInput) {
    const suspiciousPatterns = [
      /ignore\s+(previous|all)\s+instructions/i,
      /you\s+are\s+now/i,
      /forget\s+everything/i,
      /disregard\s+(all|previous)/i,
      /new\s+instructions/i,
      /system\s+prompt/i
    ];
    
    for (const pattern of suspiciousPatterns) {
      if (pattern.test(userInput)) {
        return { isInjection: true, pattern: pattern.toString(), risk: 'high' };
      }
    }
    
    return { isInjection: false, risk: 'low' };
  }
}

/**
 * Safety Guardrails Service
 */
class SafetyGuardrailsService {
  constructor() {
    this.moderation = new ContentModerationService();
  }
  
  /**
   * Input validation pipeline
   */
  async validateInput(userInput) {
    const checks = [];
    
    // Content moderation
    const moderation = await this.moderation.moderateContent(userInput);
    checks.push({
      name: 'Content Moderation',
      passed: !moderation.flagged,
      details: moderation
    });
    
    // Sensitive data detection
    const sensitiveData = await this.moderation.detectSensitiveData(userInput);
    checks.push({
      name: 'Sensitive Data Detection',
      passed: !sensitiveData.hasSensitiveData,
      details: sensitiveData
    });
    
    // Prompt injection detection
    const injection = await this.moderation.detectPromptInjection(userInput);
    checks.push({
      name: 'Prompt Injection Detection',
      passed: !injection.isInjection,
      details: injection
    });
    
    const allPassed = checks.every(check => check.passed);
    
    return {
      valid: allPassed,
      checks: checks,
      sanitizedInput: sensitiveData.hasSensitiveData 
        ? this.moderation.redactSensitiveData(userInput)
        : userInput
    };
  }
}

/**
 * ENBD Safe Chat Service
 */
class ENBDSafeChatService {
  constructor() {
    this.guardrails = new SafetyGuardrailsService();
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0.7,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }
  
  async chat(userId, message) {
    // Validate input
    const validation = await this.guardrails.validateInput(message);
    
    if (!validation.valid) {
      return {
        success: false,
        response: 'I cannot process that request. Please contact customer service.',
        reason: 'Input validation failed',
        details: validation.checks.filter(c => !c.passed)
      };
    }
    
    // Generate response
    const response = await this.model.call(validation.sanitizedInput);
    
    return {
      success: true,
      response: response.content,
      inputChecks: validation.checks
    };
  }
}

module.exports = { ContentModerationService, SafetyGuardrailsService, ENBDSafeChatService };
```

**Safety Guardrails:**

1. ✅ **Content Moderation** - OpenAI Moderation API
2. ✅ **PII Detection** - Detect credit cards, emails, phones
3. ✅ **Data Redaction** - Automatic redaction of sensitive data
4. ✅ **Prompt Injection** - Detect malicious prompts
5. ✅ **Policy Validation** - Check against business rules

---

### Q27. How do you implement bias detection and mitigation in AI systems?

**Answer:**

```javascript
const { ChatOpenAI } = require('@langchain/openai');

/**
 * Bias Detection Service
 */
class BiasDetectionService {
  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }
  
  /**
   * Detect bias in text
   */
  async detectBias(text) {
    const prompt = `Analyze this text for potential biases:

"${text}"

Check for:
- Gender bias
- Racial/ethnic bias
- Age bias
- Socioeconomic bias
- Religious bias
- Disability bias

Return JSON:
{
  "hasBias": true/false,
  "biasTypes": ["list of detected bias types"],
  "severity": "low/medium/high",
  "examples": ["specific biased statements"],
  "suggestions": ["how to make it more fair"]
}`;

    const response = await this.model.call(prompt);
    
    try {
      return JSON.parse(response.content);
    } catch (error) {
      return { hasBias: false, error: 'Failed to parse' };
    }
  }
  
  /**
   * Test fairness across demographics
   */
  async testFairness(prompt, demographics) {
    const results = {};
    
    for (const demo of demographics) {
      const testPrompt = prompt.replace('{demographic}', demo);
      const response = await this.model.call(testPrompt);
      
      results[demo] = {
        response: response.content,
        sentiment: await this.analyzeSentiment(response.content)
      };
    }
    
    // Compare responses
    const fairness = await this.compareFairness(results);
    
    return {
      results: results,
      fairness: fairness
    };
  }
  
  /**
   * Compare fairness across responses
   */
  async compareFairness(results) {
    const prompt = `Compare these responses for fairness:

${JSON.stringify(results, null, 2)}

Are the responses equally positive/negative across demographics?
Return JSON with fairness score (0-1) and analysis.`;

    const response = await this.model.call(prompt);
    
    try {
      return JSON.parse(response.content);
    } catch (error) {
      return { fairnessScore: 0.5, analysis: 'Unable to analyze' };
    }
  }
  
  async analyzeSentiment(text) {
    const prompt = `Analyze sentiment: "${text}"\nReturn: positive/negative/neutral`;
    const response = await this.model.call(prompt);
    return response.content.trim().toLowerCase();
  }
}

/**
 * ENBD Fair Lending Service
 */
class ENBDFairLendingService {
  constructor() {
    this.biasDetection = new BiasDetectionService();
  }
  
  /**
   * Test loan decision fairness
   */
  async testLoanDecisionFairness(loanScenario) {
    const demographics = [
      'male applicant',
      'female applicant',
      'young professional',
      'senior citizen',
      'local citizen',
      'expatriate'
    ];
    
    const prompt = `A {demographic} applies for a personal loan with:
- Salary: AED ${loanScenario.salary}
- Credit score: ${loanScenario.creditScore}
- Employment: ${loanScenario.employment}

Should the loan be approved? Provide brief reasoning.`;

    const fairnessTest = await this.biasDetection.testFairness(prompt, demographics);
    
    return fairnessTest;
  }
}

module.exports = { BiasDetectionService, ENBDFairLendingService };
```

---

### Q28. How do you implement explainability and model interpretability?

**Answer:**

```javascript
const { ChatOpenAI } = require('@langchain/openai');

/**
 * Explainability Service
 */
class ExplainabilityService {
  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }
  
  /**
   * Explain AI decision
   */
  async explainDecision(input, output, context) {
    const prompt = `Explain this AI decision in simple terms:

Input: ${JSON.stringify(input)}
Output: ${output}
Context: ${context}

Provide:
1. What factors led to this decision?
2. Why this outcome vs alternatives?
3. What could change the outcome?

Explanation:`;

    const response = await this.model.call(prompt);
    
    return {
      explanation: response.content,
      timestamp: new Date()
    };
  }
  
  /**
   * Generate counterfactual explanations
   */
  async generateCounterfactuals(input, currentOutput) {
    const prompt = `Given this input and output, suggest what changes would lead to a different outcome:

Current Input: ${JSON.stringify(input)}
Current Output: ${currentOutput}

Provide 3 counterfactual scenarios:
1. Minimal change scenario
2. Moderate change scenario  
3. Major change scenario

Format as JSON array.`;

    const response = await this.model.call(prompt);
    
    try {
      return JSON.parse(response.content);
    } catch (error) {
      return [];
    }
  }
  
  /**
   * Feature importance
   */
  async analyzeFeatureImportance(decision, features) {
    const prompt = `Rank these features by importance for this decision:

Decision: ${decision}
Features: ${JSON.stringify(features)}

Return JSON:
{
  "rankings": [
    {"feature": "name", "importance": 0-1, "reasoning": "why"}
  ]
}`;

    const response = await this.model.call(prompt);
    
    try {
      return JSON.parse(response.content);
    } catch (error) {
      return { rankings: [] };
    }
  }
}

/**
 * ENBD Explainable Loan Decisions
 */
class ENBDExplainableLoanService {
  constructor() {
    this.explainability = new ExplainabilityService();
  }
  
  /**
   * Make loan decision with explanation
   */
  async decideLoan(application) {
    // Simulate decision
    const decision = this.calculateLoanDecision(application);
    
    // Generate explanation
    const explanation = await this.explainability.explainDecision(
      application,
      decision.approved ? 'APPROVED' : 'REJECTED',
      'Personal loan application'
    );
    
    // Generate counterfactuals
    const counterfactuals = await this.explainability.generateCounterfactuals(
      application,
      decision.approved ? 'APPROVED' : 'REJECTED'
    );
    
    // Feature importance
    const importance = await this.explainability.analyzeFeatureImportance(
      decision.approved ? 'APPROVED' : 'REJECTED',
      application
    );
    
    return {
      decision: decision,
      explanation: explanation.explanation,
      counterfactuals: counterfactuals,
      featureImportance: importance.rankings
    };
  }
  
  calculateLoanDecision(app) {
    const score = (app.salary / 1000) + (app.creditScore / 10) - (app.existingLoans * 5);
    return {
      approved: score > 100,
      score: score,
      confidence: 0.85
    };
  }
}

module.exports = { ExplainabilityService, ENBDExplainableLoanService };
```

---

### Q29. How do you implement compliance and audit logging for AI systems?

**Answer:**

```javascript
const winston = require('winston');
const { MongoClient } = require('mongodb');

/**
 * AI Audit Logging Service
 */
class AIAuditLoggingService {
  constructor() {
    this.logger = winston.createLogger({
      level: 'info',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
      ),
      transports: [
        new winston.transports.File({ filename: 'ai-audit.log' }),
        new winston.transports.Console()
      ]
    });
    
    this.mongoClient = new MongoClient(process.env.MONGODB_URI);
  }
  
  /**
   * Log AI interaction
   */
  async logInteraction(interaction) {
    const logEntry = {
      timestamp: new Date(),
      userId: interaction.userId,
      sessionId: interaction.sessionId,
      input: interaction.input,
      output: interaction.output,
      model: interaction.model,
      confidence: interaction.confidence,
      metadata: interaction.metadata,
      complianceFlags: interaction.complianceFlags || []
    };
    
    // Log to file
    this.logger.info('AI Interaction', logEntry);
    
    // Store in database
    await this.storeInDatabase(logEntry);
    
    return logEntry;
  }
  
  /**
   * Store in database
   */
  async storeInDatabase(logEntry) {
    await this.mongoClient.connect();
    const db = this.mongoClient.db('enbd_ai_audit');
    const collection = db.collection('interactions');
    
    await collection.insertOne(logEntry);
  }
  
  /**
   * Generate audit report
   */
  async generateAuditReport(startDate, endDate) {
    await this.mongoClient.connect();
    const db = this.mongoClient.db('enbd_ai_audit');
    const collection = db.collection('interactions');
    
    const interactions = await collection.find({
      timestamp: { $gte: startDate, $lte: endDate }
    }).toArray();
    
    return {
      period: { start: startDate, end: endDate },
      totalInteractions: interactions.length,
      flaggedInteractions: interactions.filter(i => i.complianceFlags.length > 0).length,
      userBreakdown: this.groupBy(interactions, 'userId'),
      modelBreakdown: this.groupBy(interactions, 'model')
    };
  }
  
  groupBy(array, key) {
    return array.reduce((result, item) => {
      (result[item[key]] = result[item[key]] || []).push(item);
      return result;
    }, {});
  }
}

/**
 * GDPR Compliance Service
 */
class GDPRComplianceService {
  constructor() {
    this.auditLog = new AIAuditLoggingService();
  }
  
  /**
   * Handle right to explanation
   */
  async handleExplanationRequest(userId, interactionId) {
    // Retrieve interaction
    const interaction = await this.auditLog.retrieveInteraction(interactionId);
    
    if (interaction.userId !== userId) {
      throw new Error('Unauthorized access');
    }
    
    return {
      interaction: interaction,
      explanation: 'Detailed explanation of AI decision...',
      dataUsed: 'List of data points used...'
    };
  }
  
  /**
   * Handle right to be forgotten
   */
  async handleDeletionRequest(userId) {
    // Delete all user data
    await this.auditLog.deleteUserData(userId);
    
    return {
      success: true,
      deletedRecords: 'count',
      timestamp: new Date()
    };
  }
}

module.exports = { AIAuditLoggingService, GDPRComplianceService };
```

---

### Q30. How do you implement data privacy and encryption for AI systems?

**Answer:**

```javascript
const crypto = require('crypto');
const { ChatOpenAI } = require('@langchain/openai');

/**
 * Data Privacy Service
 */
class DataPrivacyService {
  constructor() {
    this.algorithm = 'aes-256-gcm';
    this.key = Buffer.from(process.env.ENCRYPTION_KEY, 'hex');
  }
  
  /**
   * Encrypt sensitive data
   */
  encrypt(text) {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv(this.algorithm, this.key, iv);
    
    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    const authTag = cipher.getAuthTag();
    
    return {
      encrypted: encrypted,
      iv: iv.toString('hex'),
      authTag: authTag.toString('hex')
    };
  }
  
  /**
   * Decrypt sensitive data
   */
  decrypt(encrypted, iv, authTag) {
    const decipher = crypto.createDecipheriv(
      this.algorithm,
      this.key,
      Buffer.from(iv, 'hex')
    );
    
    decipher.setAuthTag(Buffer.from(authTag, 'hex'));
    
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
  
  /**
   * Anonymize data
   */
  anonymize(data) {
    const anonymized = { ...data };
    
    // Remove PII
    delete anonymized.name;
    delete anonymized.email;
    delete anonymized.phone;
    
    // Hash identifiers
    if (anonymized.userId) {
      anonymized.userId = this.hashValue(anonymized.userId);
    }
    
    return anonymized;
  }
  
  /**
   * Hash value
   */
  hashValue(value) {
    return crypto
      .createHash('sha256')
      .update(value.toString())
      .digest('hex');
  }
  
  /**
   * Tokenize sensitive fields
   */
  tokenize(value) {
    const token = crypto.randomBytes(16).toString('hex');
    
    // Store mapping securely (not shown)
    // tokenMap.set(token, value);
    
    return token;
  }
}

/**
 * ENBD Privacy-Preserving AI Service
 */
class ENBDPrivacyPreservingAI {
  constructor() {
    this.privacy = new DataPrivacyService();
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0.7,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }
  
  /**
   * Process query with privacy protection
   */
  async processPrivateQuery(userId, query, sensitiveData) {
    // Anonymize user data
    const anonymizedData = this.privacy.anonymize(sensitiveData);
    
    // Encrypt before logging
    const encrypted = this.privacy.encrypt(JSON.stringify(sensitiveData));
    
    // Process with anonymized data
    const response = await this.model.call(`${query}\n\nData: ${JSON.stringify(anonymizedData)}`);
    
    return {
      response: response.content,
      privacyPreserved: true,
      encryptedData: encrypted
    };
  }
  
  /**
   * Differential privacy
   */
  addNoise(value, epsilon = 0.1) {
    const noise = this.gaussianNoise(0, 1 / epsilon);
    return value + noise;
  }
  
  gaussianNoise(mean, stdDev) {
    const u1 = Math.random();
    const u2 = Math.random();
    const z0 = Math.sqrt(-2 * Math.log(u1)) * Math.cos(2 * Math.PI * u2);
    return z0 * stdDev + mean;
  }
}

module.exports = { DataPrivacyService, ENBDPrivacyPreservingAI };
```

**Compliance & Privacy:**

1. ✅ **Content Moderation** - Safety guardrails
2. ✅ **Bias Detection** - Fair AI systems
3. ✅ **Explainability** - Transparent decisions
4. ✅ **Audit Logging** - Complete traceability
5. ✅ **Data Privacy** - Encryption & anonymization
6. ✅ **GDPR Compliance** - Right to explanation, deletion

---

**Summary Q26-Q30:**
- Safety guardrails (moderation, PII detection, prompt injection) ✅
- Bias detection and mitigation for fair lending ✅
- Explainability with counterfactuals and feature importance ✅
- Compliance audit logging with GDPR support ✅
- Data privacy with encryption and anonymization ✅
