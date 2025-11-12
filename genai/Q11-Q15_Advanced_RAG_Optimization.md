# GenAI Questions 11-15: Advanced RAG & Prompt Optimization

---

### Q11. How do you implement advanced RAG with re-ranking and filtering?

**Answer:**

Advanced RAG improves retrieval quality through re-ranking, filtering, and query optimization.

```javascript
const { ChatOpenAI, OpenAIEmbeddings } = require('@langchain/openai');
const { Cohere } = require('cohere-ai');

/**
 * Advanced RAG Service with Re-ranking
 */
class AdvancedRAGService {
  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    this.embeddings = new OpenAIEmbeddings({
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    // Cohere for re-ranking
    this.cohere = new Cohere({
      apiKey: process.env.COHERE_API_KEY
    });
  }
  
  /**
   * Query Expansion - Generate multiple search queries
   */
  async expandQuery(originalQuery) {
    const prompt = `Generate 3 different variations of this search query to improve retrieval:

Original Query: "${originalQuery}"

Generate semantically similar queries that might retrieve relevant documents:

1.`;

    const response = await this.model.call(prompt);
    
    const queries = response.content
      .split('\n')
      .filter(line => line.match(/^\d+\./))
      .map(line => line.replace(/^\d+\.\s*/, '').trim());
    
    return [originalQuery, ...queries];
  }
  
  /**
   * Hypothetical Document Embeddings (HyDE)
   * Generate hypothetical answer, then search for similar docs
   */
  async generateHypotheticalDocument(query) {
    const prompt = `Given this question about Emirates NBD banking, generate a hypothetical detailed answer:

Question: ${query}

Hypothetical Answer:`;

    const response = await this.model.call(prompt);
    
    return response.content;
  }
  
  /**
   * Multi-query retrieval with fusion
   */
  async multiQueryRetrieval(originalQuery, vectorStore, k = 20) {
    // Expand query
    const queries = await this.expandQuery(originalQuery);
    
    // Retrieve documents for each query
    const allResults = [];
    
    for (const query of queries) {
      const results = await vectorStore.similaritySearchWithScore(query, k);
      allResults.push(...results);
    }
    
    // Remove duplicates and fuse rankings
    const fusedResults = this.reciprocalRankFusion(allResults);
    
    return fusedResults.slice(0, k);
  }
  
  /**
   * Reciprocal Rank Fusion (RRF)
   * Combines multiple ranking lists
   */
  reciprocalRankFusion(results, k = 60) {
    const scoreMap = new Map();
    
    results.forEach((result, index) => {
      const docId = result.metadata?.id || result.pageContent.substring(0, 50);
      const rffScore = 1 / (k + index + 1);
      
      if (scoreMap.has(docId)) {
        scoreMap.set(docId, scoreMap.get(docId) + rffScore);
      } else {
        scoreMap.set(docId, rffScore);
      }
    });
    
    // Sort by fused score
    const fusedResults = Array.from(scoreMap.entries())
      .sort((a, b) => b[1] - a[1])
      .map(([docId, score]) => {
        const doc = results.find(r => 
          (r.metadata?.id || r.pageContent.substring(0, 50)) === docId
        );
        return { ...doc, fusedScore: score };
      });
    
    return fusedResults;
  }
  
  /**
   * Re-rank with Cohere
   */
  async rerankWithCohere(query, documents, topK = 5) {
    const docs = documents.map(doc => doc.pageContent || doc.content);
    
    const response = await this.cohere.rerank({
      query: query,
      documents: docs,
      topN: topK,
      model: 'rerank-english-v2.0'
    });
    
    // Map back to original documents with new scores
    return response.results.map(result => ({
      ...documents[result.index],
      rerankScore: result.relevance_score
    }));
  }
  
  /**
   * Metadata filtering
   */
  filterByMetadata(documents, filters) {
    return documents.filter(doc => {
      for (const [key, value] of Object.entries(filters)) {
        if (doc.metadata[key] !== value) {
          return false;
        }
      }
      return true;
    });
  }
  
  /**
   * Contextual compression - Remove irrelevant parts
   */
  async compressContext(query, documents) {
    const compressed = [];
    
    for (const doc of documents) {
      const prompt = `Given this query and document, extract only the most relevant sentences:

Query: ${query}

Document:
${doc.pageContent}

Relevant Excerpt:`;

      const response = await this.model.call(prompt);
      
      compressed.push({
        ...doc,
        compressedContent: response.content
      });
    }
    
    return compressed;
  }
}

/**
 * Practical Example: ENBD Smart Document Retrieval
 */
class ENBDSmartRetrieval {
  constructor() {
    this.ragService = new AdvancedRAGService();
  }
  
  /**
   * Complete RAG pipeline with all enhancements
   */
  async retrieveAndGenerate(query, vectorStore, options = {}) {
    const {
      useQueryExpansion = true,
      useHyDE = false,
      useReranking = true,
      useCompression = false,
      metadataFilters = {},
      topK = 5
    } = options;
    
    console.log('Step 1: Query Processing');
    let retrievalQuery = query;
    
    // HyDE: Generate hypothetical document
    if (useHyDE) {
      console.log('  - Using HyDE');
      retrievalQuery = await this.ragService.generateHypotheticalDocument(query);
    }
    
    console.log('Step 2: Document Retrieval');
    let documents;
    
    if (useQueryExpansion) {
      console.log('  - Using multi-query retrieval');
      documents = await this.ragService.multiQueryRetrieval(
        retrievalQuery, 
        vectorStore, 
        topK * 4
      );
    } else {
      documents = await vectorStore.similaritySearchWithScore(retrievalQuery, topK * 4);
    }
    
    console.log(`  - Retrieved ${documents.length} documents`);
    
    // Filter by metadata
    if (Object.keys(metadataFilters).length > 0) {
      console.log('Step 3: Metadata Filtering');
      documents = this.ragService.filterByMetadata(documents, metadataFilters);
      console.log(`  - Filtered to ${documents.length} documents`);
    }
    
    // Re-ranking
    if (useReranking && documents.length > 0) {
      console.log('Step 4: Re-ranking with Cohere');
      documents = await this.ragService.rerankWithCohere(query, documents, topK);
      console.log(`  - Re-ranked to top ${documents.length}`);
    } else {
      documents = documents.slice(0, topK);
    }
    
    // Contextual compression
    if (useCompression) {
      console.log('Step 5: Context Compression');
      documents = await this.ragService.compressContext(query, documents);
    }
    
    // Generate final answer
    console.log('Step 6: Answer Generation');
    const answer = await this.generateAnswer(query, documents);
    
    return {
      answer: answer,
      sources: documents.map(doc => ({
        content: doc.compressedContent || doc.pageContent,
        metadata: doc.metadata,
        score: doc.rerankScore || doc.fusedScore || doc.score
      }))
    };
  }
  
  /**
   * Generate answer with retrieved context
   */
  async generateAnswer(query, documents) {
    const context = documents
      .map((doc, idx) => `[${idx + 1}] ${doc.compressedContent || doc.pageContent}`)
      .join('\n\n');
    
    const prompt = `You are a helpful banking assistant for Emirates NBD.

Use the following context to answer the customer's question. If the answer is not in the context, say "I don't have enough information to answer that."

Context:
${context}

Customer Question: ${query}

Answer:`;

    const response = await this.ragService.model.call(prompt);
    
    return response.content;
  }
}

/**
 * Usage Example
 */
async function exampleAdvancedRAG() {
  const smartRetrieval = new ENBDSmartRetrieval();
  
  const query = 'What are the benefits of opening a savings account with high balance?';
  
  // With all optimizations
  const result = await smartRetrieval.retrieveAndGenerate(query, vectorStore, {
    useQueryExpansion: true,
    useReranking: true,
    useCompression: true,
    metadataFilters: { category: 'products' },
    topK: 3
  });
  
  console.log('Answer:', result.answer);
  console.log('Sources:', result.sources);
}

module.exports = {
  AdvancedRAGService,
  ENBDSmartRetrieval
};
```

**Advanced RAG Techniques:**

1. ✅ **Query Expansion** - Generate multiple search queries
2. ✅ **HyDE** - Hypothetical Document Embeddings
3. ✅ **Multi-Query Fusion** - Combine results from multiple queries
4. ✅ **Re-ranking** - Cohere re-ranker for better relevance
5. ✅ **Contextual Compression** - Remove irrelevant content
6. ✅ **Metadata Filtering** - Filter by category, date, etc.

---

### Q12. How do you implement hallucination detection and response validation?

**Answer:**

```javascript
const { ChatOpenAI } = require('@langchain/openai');

/**
 * Hallucination Detection Service
 */
class HallucinationDetectionService {
  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }
  
  /**
   * Check if answer is grounded in context
   */
  async checkGrounding(question, context, answer) {
    const prompt = `You are a fact-checker. Determine if the answer is fully supported by the given context.

Context:
${context}

Question: ${question}

Answer: ${answer}

Is this answer fully supported by the context? Respond with JSON:
{
  "grounded": true/false,
  "confidence": 0-1,
  "unsupported_claims": ["claim1", "claim2"],
  "reasoning": "explanation"
}`;

    const response = await this.model.call(prompt);
    
    try {
      return JSON.parse(response.content);
    } catch (error) {
      return {
        grounded: false,
        confidence: 0,
        error: 'Failed to parse response'
      };
    }
  }
  
  /**
   * Self-consistency check - Generate multiple answers
   */
  async selfConsistencyCheck(question, context, numSamples = 3) {
    const answers = [];
    
    // Generate multiple answers
    for (let i = 0; i < numSamples; i++) {
      const answer = await this.generateAnswer(question, context);
      answers.push(answer);
    }
    
    // Check consistency
    const consistency = await this.checkConsistency(answers);
    
    return {
      answers: answers,
      consistency: consistency,
      finalAnswer: consistency.score > 0.8 ? answers[0] : null
    };
  }
  
  /**
   * Check consistency between multiple answers
   */
  async checkConsistency(answers) {
    const prompt = `Compare these answers and determine if they are consistent:

${answers.map((a, i) => `Answer ${i + 1}: ${a}`).join('\n\n')}

Are these answers consistent? Respond with JSON:
{
  "consistent": true/false,
  "score": 0-1,
  "differences": ["difference1", "difference2"]
}`;

    const response = await this.model.call(prompt);
    
    try {
      return JSON.parse(response.content);
    } catch (error) {
      return { consistent: false, score: 0 };
    }
  }
  
  /**
   * Chain-of-Verification (CoVe)
   */
  async chainOfVerification(question, initialAnswer, context) {
    // Step 1: Generate verification questions
    const verificationQuestions = await this.generateVerificationQuestions(
      question,
      initialAnswer
    );
    
    // Step 2: Answer verification questions using context
    const verifications = [];
    for (const vq of verificationQuestions) {
      const verification = await this.verifyFact(vq, context);
      verifications.push(verification);
    }
    
    // Step 3: Generate final answer considering verifications
    const finalAnswer = await this.generateVerifiedAnswer(
      question,
      initialAnswer,
      verifications
    );
    
    return {
      initialAnswer: initialAnswer,
      verifications: verifications,
      finalAnswer: finalAnswer
    };
  }
  
  /**
   * Generate verification questions
   */
  async generateVerificationQuestions(question, answer) {
    const prompt = `Given this question and answer, generate 3 verification questions to check for factual accuracy:

Question: ${question}
Answer: ${answer}

Verification Questions:
1.`;

    const response = await this.model.call(prompt);
    
    const questions = response.content
      .split('\n')
      .filter(line => line.match(/^\d+\./))
      .map(line => line.replace(/^\d+\.\s*/, '').trim());
    
    return questions;
  }
  
  /**
   * Verify specific fact
   */
  async verifyFact(question, context) {
    const prompt = `Based on the context, answer this verification question:

Context: ${context}

Question: ${question}

Answer:`;

    const response = await this.model.call(prompt);
    
    return {
      question: question,
      answer: response.content
    };
  }
  
  /**
   * Generate verified answer
   */
  async generateVerifiedAnswer(originalQuestion, initialAnswer, verifications) {
    const verificationsText = verifications
      .map(v => `Q: ${v.question}\nA: ${v.answer}`)
      .join('\n\n');
    
    const prompt = `Based on these verifications, provide a corrected answer if needed:

Original Question: ${originalQuestion}
Initial Answer: ${initialAnswer}

Verifications:
${verificationsText}

If the initial answer is correct, return it unchanged. If corrections are needed, provide the corrected answer.

Final Answer:`;

    const response = await this.model.call(prompt);
    
    return response.content;
  }
  
  /**
   * Attribution - Add source citations
   */
  async addAttributions(answer, sources) {
    const sourcesText = sources
      .map((s, i) => `[${i + 1}] ${s.content} (Source: ${s.metadata.source})`)
      .join('\n\n');
    
    const prompt = `Add inline citations to this answer using the provided sources. Use [N] format.

Answer: ${answer}

Sources:
${sourcesText}

Answer with Citations:`;

    const response = await this.model.call(prompt);
    
    return response.content;
  }
  
  async generateAnswer(question, context) {
    const prompt = `Answer this question based on the context:

Context: ${context}

Question: ${question}

Answer:`;

    const response = await this.model.call(prompt);
    return response.content;
  }
}

/**
 * Practical Example: ENBD Safe Response System
 */
class ENBDSafeResponseSystem {
  constructor() {
    this.hallucinationDetector = new HallucinationDetectionService();
  }
  
  /**
   * Generate safe, verified response
   */
  async generateSafeResponse(question, context, sources) {
    console.log('Step 1: Generate initial answer');
    const initialAnswer = await this.hallucinationDetector.generateAnswer(
      question,
      context
    );
    
    console.log('Step 2: Check grounding');
    const grounding = await this.hallucinationDetector.checkGrounding(
      question,
      context,
      initialAnswer
    );
    
    if (!grounding.grounded || grounding.confidence < 0.7) {
      console.log('⚠️  Answer not well-grounded. Using Chain-of-Verification...');
      
      const verified = await this.hallucinationDetector.chainOfVerification(
        question,
        initialAnswer,
        context
      );
      
      // Add attributions
      const withCitations = await this.hallucinationDetector.addAttributions(
        verified.finalAnswer,
        sources
      );
      
      return {
        answer: withCitations,
        confidence: 'medium',
        verified: true,
        warnings: grounding.unsupported_claims
      };
    }
    
    console.log('✅ Answer is well-grounded');
    
    // Add attributions to grounded answer
    const withCitations = await this.hallucinationDetector.addAttributions(
      initialAnswer,
      sources
    );
    
    return {
      answer: withCitations,
      confidence: 'high',
      verified: true,
      warnings: []
    };
  }
  
  /**
   * Confidence scoring
   */
  calculateConfidence(grounding, consistency) {
    let score = 0;
    
    if (grounding.grounded) score += 50;
    score += grounding.confidence * 30;
    
    if (consistency) {
      score += consistency.score * 20;
    }
    
    return {
      score: score,
      level: score >= 80 ? 'high' : score >= 60 ? 'medium' : 'low'
    };
  }
}

/**
 * Usage Example
 */
async function exampleHallucinationDetection() {
  const safeSystem = new ENBDSafeResponseSystem();
  
  const question = 'What is the minimum salary for a personal loan?';
  const context = 'ENBD personal loan requires minimum monthly salary of AED 5,000.';
  const sources = [
    {
      content: 'Personal loan eligibility: Minimum salary AED 5,000/month',
      metadata: { source: 'personal-loan-policy.pdf' }
    }
  ];
  
  const result = await safeSystem.generateSafeResponse(question, context, sources);
  
  console.log('Safe Answer:', result.answer);
  console.log('Confidence:', result.confidence);
  console.log('Verified:', result.verified);
}

module.exports = {
  HallucinationDetectionService,
  ENBDSafeResponseSystem
};
```

**Hallucination Prevention:**

1. ✅ **Grounding Check** - Verify answer is supported by context
2. ✅ **Self-Consistency** - Generate multiple answers and compare
3. ✅ **Chain-of-Verification** - Generate and answer verification questions
4. ✅ **Attribution** - Add source citations
5. ✅ **Confidence Scoring** - Measure answer reliability

---

### Q13. How do you implement conversation context management and memory optimization?

**Answer:**

```javascript
const { ChatOpenAI } = require('@langchain/openai');
const { BufferMemory, ConversationSummaryMemory, ConversationTokenBufferMemory } = require('langchain/memory');
const Redis = require('ioredis');

/**
 * Advanced Memory Management Service
 */
class AdvancedMemoryService {
  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0.7,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    this.redis = new Redis({
      host: process.env.REDIS_HOST || 'localhost',
      port: process.env.REDIS_PORT || 6379
    });
  }
  
  /**
   * Token-aware memory (prevents context overflow)
   */
  createTokenBufferMemory(maxTokens = 2000) {
    return new ConversationTokenBufferMemory({
      llm: this.model,
      maxTokenLimit: maxTokens,
      returnMessages: true,
      memoryKey: 'chat_history'
    });
  }
  
  /**
   * Summary memory (summarizes old conversations)
   */
  createSummaryMemory() {
    return new ConversationSummaryMemory({
      llm: this.model,
      returnMessages: true,
      memoryKey: 'chat_history'
    });
  }
  
  /**
   * Entity memory (tracks important entities)
   */
  async extractEntities(conversation) {
    const prompt = `Extract key entities from this banking conversation:

Conversation:
${conversation}

Extract these entities as JSON:
{
  "customer_name": "",
  "account_numbers": [],
  "mentioned_products": [],
  "amounts": [],
  "dates": []
}`;

    const response = await this.model.call(prompt);
    
    try {
      return JSON.parse(response.content);
    } catch (error) {
      return {};
    }
  }
  
  /**
   * Persistent memory with Redis
   */
  async saveConversation(userId, conversation) {
    const key = `conversation:${userId}`;
    
    await this.redis.setex(
      key,
      86400, // 24 hours TTL
      JSON.stringify(conversation)
    );
  }
  
  async loadConversation(userId) {
    const key = `conversation:${userId}`;
    const data = await this.redis.get(key);
    
    return data ? JSON.parse(data) : null;
  }
  
  /**
   * Conversation summarization
   */
  async summarizeConversation(messages) {
    const conversation = messages
      .map(m => `${m.role}: ${m.content}`)
      .join('\n');
    
    const prompt = `Summarize this banking conversation, keeping important details:

${conversation}

Summary (focus on: customer requests, account details, unresolved issues):`;

    const response = await this.model.call(prompt);
    
    return response.content;
  }
  
  /**
   * Sliding window with summary
   */
  async slidingWindowWithSummary(messages, windowSize = 10) {
    if (messages.length <= windowSize) {
      return messages;
    }
    
    // Old messages to summarize
    const oldMessages = messages.slice(0, -windowSize);
    
    // Recent messages to keep
    const recentMessages = messages.slice(-windowSize);
    
    // Summarize old messages
    const summary = await this.summarizeConversation(oldMessages);
    
    return [
      { role: 'system', content: `Previous conversation summary: ${summary}` },
      ...recentMessages
    ];
  }
}

/**
 * Practical Example: ENBD Conversation Manager
 */
class ENBDConversationManager {
  constructor() {
    this.memoryService = new AdvancedMemoryService();
    this.sessions = new Map();
  }
  
  /**
   * Initialize session for user
   */
  async initializeSession(userId, accountInfo = {}) {
    const memory = this.memoryService.createTokenBufferMemory(2000);
    
    // Load previous conversation if exists
    const previousConversation = await this.memoryService.loadConversation(userId);
    
    let systemMessage = `You are a banking assistant for Emirates NBD.

Customer Information:
- Customer ID: ${userId}
- Account Type: ${accountInfo.accountType || 'Unknown'}
- Relationship: ${accountInfo.relationship || 'Standard'}

Guidelines:
- Be professional and helpful
- Reference previous conversation context when relevant
- Never ask for passwords or PINs
- Provide accurate banking information`;

    if (previousConversation) {
      const summary = await this.memoryService.summarizeConversation(
        previousConversation.messages
      );
      systemMessage += `\n\nPrevious Session Summary:\n${summary}`;
    }
    
    this.sessions.set(userId, {
      memory: memory,
      entities: {},
      messageCount: 0,
      startTime: new Date(),
      systemMessage: systemMessage
    });
  }
  
  /**
   * Process message with context
   */
  async processMessage(userId, message) {
    let session = this.sessions.get(userId);
    
    if (!session) {
      await this.initializeSession(userId);
      session = this.sessions.get(userId);
    }
    
    // Add message to memory
    await session.memory.chatHistory.addUserMessage(message);
    
    // Extract entities from message
    const entities = await this.memoryService.extractEntities(message);
    session.entities = { ...session.entities, ...entities };
    
    // Get conversation history
    const history = await session.memory.chatHistory.getMessages();
    
    // Apply sliding window if too long
    let contextMessages = history;
    if (history.length > 20) {
      contextMessages = await this.memoryService.slidingWindowWithSummary(
        history,
        10
      );
    }
    
    // Generate response
    const response = await this.generateResponse(
      session.systemMessage,
      contextMessages,
      message,
      session.entities
    );
    
    // Add response to memory
    await session.memory.chatHistory.addAIMessage(response);
    
    session.messageCount++;
    
    // Save conversation periodically
    if (session.messageCount % 5 === 0) {
      await this.saveSession(userId);
    }
    
    return {
      response: response,
      entities: session.entities,
      messageCount: session.messageCount
    };
  }
  
  /**
   * Generate response with context
   */
  async generateResponse(systemMessage, history, currentMessage, entities) {
    const messages = [
      { role: 'system', content: systemMessage },
      ...history.map(m => ({
        role: m._getType() === 'human' ? 'user' : 'assistant',
        content: m.content
      })),
      { role: 'user', content: currentMessage }
    ];
    
    // Add entity context if relevant
    if (Object.keys(entities).length > 0) {
      const entityContext = `\n\nExtracted entities: ${JSON.stringify(entities)}`;
      messages[0].content += entityContext;
    }
    
    const response = await this.memoryService.model.call(
      messages.map(m => ({ role: m.role, content: m.content }))
    );
    
    return response.content;
  }
  
  /**
   * Save session to Redis
   */
  async saveSession(userId) {
    const session = this.sessions.get(userId);
    
    if (!session) return;
    
    const history = await session.memory.chatHistory.getMessages();
    
    const conversationData = {
      messages: history.map(m => ({
        role: m._getType(),
        content: m.content
      })),
      entities: session.entities,
      messageCount: session.messageCount,
      lastActivity: new Date()
    };
    
    await this.memoryService.saveConversation(userId, conversationData);
  }
  
  /**
   * End session
   */
  async endSession(userId) {
    await this.saveSession(userId);
    this.sessions.delete(userId);
  }
  
  /**
   * Get conversation statistics
   */
  getSessionStats(userId) {
    const session = this.sessions.get(userId);
    
    if (!session) return null;
    
    return {
      messageCount: session.messageCount,
      duration: Date.now() - session.startTime.getTime(),
      entities: session.entities
    };
  }
}

/**
 * Usage Example
 */
async function exampleConversationManagement() {
  const manager = new ENBDConversationManager();
  const userId = 'customer123';
  
  // Initialize session
  await manager.initializeSession(userId, {
    accountType: 'Savings',
    relationship: 'Premium'
  });
  
  // Message 1
  const res1 = await manager.processMessage(userId, 
    'I want to apply for a personal loan of AED 100,000'
  );
  console.log('Assistant:', res1.response);
  console.log('Entities:', res1.entities);
  
  // Message 2 (uses context)
  const res2 = await manager.processMessage(userId,
    'What documents do I need?'
  );
  console.log('Assistant:', res2.response);
  
  // Message 3 (uses context)
  const res3 = await manager.processMessage(userId,
    'How long will approval take?'
  );
  console.log('Assistant:', res3.response);
  
  // Get stats
  const stats = manager.getSessionStats(userId);
  console.log('Session Stats:', stats);
  
  // End session
  await manager.endSession(userId);
}

module.exports = {
  AdvancedMemoryService,
  ENBDConversationManager
};
```

**Memory Optimization:**

1. ✅ **Token-aware Memory** - Prevent context overflow
2. ✅ **Conversation Summarization** - Compress old messages
3. ✅ **Entity Extraction** - Track important information
4. ✅ **Sliding Window** - Keep recent + summary of old
5. ✅ **Persistent Storage** - Redis for session management
6. ✅ **Memory Strategies** - Buffer, Window, Summary

---

### Q14. How do you implement prompt optimization and testing?

**Answer:**

```javascript
const { ChatOpenAI } = require('@langchain/openai');

/**
 * Prompt Optimization Service
 */
class PromptOptimizationService {
  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }
  
  /**
   * Prompt templates library
   */
  getPromptTemplates() {
    return {
      basic: {
        template: `Answer this question: {question}`,
        description: 'Basic prompt without context'
      },
      
      withRole: {
        template: `You are a banking expert at Emirates NBD.

Question: {question}

Answer:`,
        description: 'Assigns role to AI'
      },
      
      withExamples: {
        template: `You are a banking expert at Emirates NBD. Here are some examples:

Q: What is minimum balance for savings account?
A: The minimum balance for ENBD savings account is AED 3,000.

Q: {question}
A:`,
        description: 'Few-shot learning with examples'
      },
      
      withConstraints: {
        template: `You are a banking expert at Emirates NBD.

Guidelines:
- Provide accurate information only
- If unsure, say "I don't have that information"
- Keep answers concise (max 3 sentences)
- Use professional language

Question: {question}

Answer:`,
        description: 'Adds constraints and guidelines'
      },
      
      chainOfThought: {
        template: `You are a banking expert at Emirates NBD.

Question: {question}

Let's think step by step:
1. First, identify what information is needed
2. Then, recall relevant banking policies
3. Finally, provide a clear answer

Answer:`,
        description: 'Encourages reasoning process'
      },
      
      structured: {
        template: `You are a banking expert at Emirates NBD.

Question: {question}

Provide answer in this format:
- Direct Answer: [answer]
- Explanation: [explanation]
- Related Products: [if applicable]
- Next Steps: [if applicable]`,
        description: 'Requests structured output'
      }
    };
  }
  
  /**
   * Test prompt with multiple inputs
   */
  async testPrompt(promptTemplate, testCases) {
    const results = [];
    
    for (const testCase of testCases) {
      const prompt = promptTemplate.replace('{question}', testCase.input);
      
      const startTime = Date.now();
      const response = await this.model.call(prompt);
      const latency = Date.now() - startTime;
      
      results.push({
        input: testCase.input,
        output: response.content,
        expectedOutput: testCase.expected,
        latency: latency,
        score: await this.scoreResponse(response.content, testCase.expected)
      });
    }
    
    return results;
  }
  
  /**
   * Score response quality
   */
  async scoreResponse(actual, expected) {
    const prompt = `Rate the similarity between these two answers (0-1):

Expected: ${expected}
Actual: ${actual}

Score (0-1):`;

    const response = await this.model.call(prompt);
    
    const score = parseFloat(response.content.match(/[\d.]+/)?.[0] || '0');
    return Math.min(Math.max(score, 0), 1);
  }
  
  /**
   * A/B test prompts
   */
  async abTestPrompts(promptA, promptB, testCases) {
    console.log('Testing Prompt A...');
    const resultsA = await this.testPrompt(promptA, testCases);
    
    console.log('Testing Prompt B...');
    const resultsB = await this.testPrompt(promptB, testCases);
    
    // Calculate metrics
    const metricsA = this.calculateMetrics(resultsA);
    const metricsB = this.calculateMetrics(resultsB);
    
    return {
      promptA: {
        prompt: promptA,
        results: resultsA,
        metrics: metricsA
      },
      promptB: {
        prompt: promptB,
        results: resultsB,
        metrics: metricsB
      },
      winner: metricsA.avgScore > metricsB.avgScore ? 'A' : 'B'
    };
  }
  
  /**
   * Calculate performance metrics
   */
  calculateMetrics(results) {
    const scores = results.map(r => r.score);
    const latencies = results.map(r => r.latency);
    
    return {
      avgScore: scores.reduce((a, b) => a + b, 0) / scores.length,
      minScore: Math.min(...scores),
      maxScore: Math.max(...scores),
      avgLatency: latencies.reduce((a, b) => a + b, 0) / latencies.length,
      totalTests: results.length
    };
  }
  
  /**
   * Generate optimized prompt using AI
   */
  async generateOptimizedPrompt(task, examples) {
    const examplesText = examples
      .map(e => `Input: ${e.input}\nExpected: ${e.output}`)
      .join('\n\n');
    
    const prompt = `Given this task and examples, create an optimized system prompt:

Task: ${task}

Examples:
${examplesText}

Generate a clear, effective system prompt that will produce high-quality results for this task.

Optimized Prompt:`;

    const response = await this.model.call(prompt);
    
    return response.content;
  }
}

/**
 * Practical Example: ENBD Prompt Testing System
 */
class ENBDPromptTestingSystem {
  constructor() {
    this.optimizer = new PromptOptimizationService();
  }
  
  /**
   * Run comprehensive prompt tests
   */
  async runPromptTests() {
    const testCases = [
      {
        input: 'What is the minimum salary for personal loan?',
        expected: 'The minimum monthly salary requirement for ENBD personal loan is AED 5,000.'
      },
      {
        input: 'How do I open savings account?',
        expected: 'To open a savings account, provide Emirates ID, passport, and salary certificate. Apply online or visit any branch.'
      },
      {
        input: 'What are credit card fees?',
        expected: 'Credit card annual fees: Classic (AED 200), Platinum (AED 500), Signature (AED 1,000).'
      }
    ];
    
    const templates = this.optimizer.getPromptTemplates();
    
    console.log('=== Testing All Prompt Templates ===\n');
    
    const allResults = {};
    
    for (const [name, template] of Object.entries(templates)) {
      console.log(`Testing: ${name}`);
      console.log(`Description: ${template.description}\n`);
      
      const results = await this.optimizer.testPrompt(
        template.template,
        testCases
      );
      
      const metrics = this.optimizer.calculateMetrics(results);
      
      allResults[name] = {
        template: template,
        metrics: metrics,
        results: results
      };
      
      console.log(`Avg Score: ${metrics.avgScore.toFixed(2)}`);
      console.log(`Avg Latency: ${metrics.avgLatency}ms\n`);
    }
    
    // Find best performing prompt
    const best = Object.entries(allResults)
      .sort((a, b) => b[1].metrics.avgScore - a[1].metrics.avgScore)[0];
    
    console.log(`🏆 Best Prompt: ${best[0]}`);
    console.log(`Score: ${best[1].metrics.avgScore.toFixed(2)}`);
    
    return allResults;
  }
  
  /**
   * A/B test two specific prompts
   */
  async comparePrompts() {
    const promptA = `You are a helpful banking assistant.

Question: {question}
Answer:`;

    const promptB = `You are an expert banking advisor at Emirates NBD with 10 years of experience.

Guidelines:
- Provide accurate, concise answers
- Use professional language
- Reference specific policies when relevant

Customer Question: {question}

Professional Answer:`;

    const testCases = [
      {
        input: 'How much can I borrow?',
        expected: 'Loan amount depends on your salary. Generally, you can borrow up to 20 times your monthly salary, ranging from AED 10,000 to AED 500,000.'
      },
      {
        input: 'Is there a fee?',
        expected: 'Yes, there is a processing fee of 1% of the loan amount (minimum AED 500).'
      }
    ];
    
    const comparison = await this.optimizer.abTestPrompts(
      promptA,
      promptB,
      testCases
    );
    
    console.log('\n=== A/B Test Results ===');
    console.log(`\nPrompt A Score: ${comparison.promptA.metrics.avgScore.toFixed(2)}`);
    console.log(`Prompt B Score: ${comparison.promptB.metrics.avgScore.toFixed(2)}`);
    console.log(`\nWinner: Prompt ${comparison.winner}`);
    
    return comparison;
  }
}

/**
 * Usage Example
 */
async function examplePromptTesting() {
  const testSystem = new ENBDPromptTestingSystem();
  
  // Test all templates
  await testSystem.runPromptTests();
  
  // A/B test specific prompts
  await testSystem.comparePrompts();
}

module.exports = {
  PromptOptimizationService,
  ENBDPromptTestingSystem
};
```

**Prompt Optimization:**

1. ✅ **Template Library** - Reusable prompt patterns
2. ✅ **A/B Testing** - Compare prompt performance
3. ✅ **Automated Scoring** - Measure response quality
4. ✅ **Performance Metrics** - Score, latency, consistency
5. ✅ **Few-Shot Examples** - Improve accuracy
6. ✅ **Chain-of-Thought** - Better reasoning

---

### Q15. How do you implement cost optimization and caching for LLM applications?

**Answer:**

```javascript
const { ChatOpenAI } = require('@langchain/openai');
const Redis = require('ioredis');
const crypto = require('crypto');

/**
 * Cost Optimization Service
 */
class CostOptimizationService {
  constructor() {
    this.redis = new Redis({
      host: process.env.REDIS_HOST || 'localhost',
      port: process.env.REDIS_PORT || 6379
    });
    
    this.models = {
      'gpt-4': {
        inputCost: 0.03 / 1000, // per token
        outputCost: 0.06 / 1000,
        contextWindow: 8192
      },
      'gpt-3.5-turbo': {
        inputCost: 0.0015 / 1000,
        outputCost: 0.002 / 1000,
        contextWindow: 4096
      },
      'gpt-4-turbo': {
        inputCost: 0.01 / 1000,
        outputCost: 0.03 / 1000,
        contextWindow: 128000
      }
    };
    
    this.usageStats = {
      totalTokens: 0,
      totalCost: 0,
      cacheHits: 0,
      cacheMisses: 0
    };
  }
  
  /**
   * Semantic cache - Cache similar queries
   */
  async semanticCache(query, generateResponse) {
    // Generate cache key using embedding
    const cacheKey = await this.generateCacheKey(query);
    
    // Check cache
    const cached = await this.redis.get(cacheKey);
    
    if (cached) {
      this.usageStats.cacheHits++;
      console.log('✅ Cache HIT');
      return {
        response: cached,
        fromCache: true,
        cost: 0
      };
    }
    
    this.usageStats.cacheMisses++;
    console.log('❌ Cache MISS');
    
    // Generate response
    const { response, cost, tokens } = await generateResponse();
    
    // Cache for 1 hour
    await this.redis.setex(cacheKey, 3600, response);
    
    // Update stats
    this.usageStats.totalTokens += tokens;
    this.usageStats.totalCost += cost;
    
    return {
      response: response,
      fromCache: false,
      cost: cost
    };
  }
  
  /**
   * Generate cache key
   */
  async generateCacheKey(query) {
    // Normalize query
    const normalized = query.toLowerCase().trim();
    
    // Hash for cache key
    const hash = crypto
      .createHash('sha256')
      .update(normalized)
      .digest('hex')
      .substring(0, 16);
    
    return `cache:${hash}`;
  }
  
  /**
   * Model selection based on complexity
   */
  async selectOptimalModel(query, context) {
    const complexity = await this.assessComplexity(query, context);
    
    if (complexity === 'simple') {
      return 'gpt-3.5-turbo'; // Cheaper model
    } else if (complexity === 'medium') {
      return 'gpt-4-turbo'; // Balanced
    } else {
      return 'gpt-4'; // Most capable
    }
  }
  
  /**
   * Assess query complexity
   */
  async assessComplexity(query, context) {
    // Simple heuristics
    const wordCount = query.split(' ').length;
    const hasContext = context && context.length > 0;
    
    if (wordCount < 10 && !hasContext) {
      return 'simple';
    } else if (wordCount < 30) {
      return 'medium';
    } else {
      return 'complex';
    }
  }
  
  /**
   * Calculate cost
   */
  calculateCost(modelName, inputTokens, outputTokens) {
    const model = this.models[modelName];
    
    if (!model) {
      throw new Error(`Unknown model: ${modelName}`);
    }
    
    const inputCost = inputTokens * model.inputCost;
    const outputCost = outputTokens * model.outputCost;
    
    return {
      inputCost: inputCost,
      outputCost: outputCost,
      totalCost: inputCost + outputCost,
      inputTokens: inputTokens,
      outputTokens: outputTokens,
      totalTokens: inputTokens + outputTokens
    };
  }
  
  /**
   * Token counting (approximate)
   */
  countTokens(text) {
    // Rough approximation: 1 token ≈ 4 characters
    return Math.ceil(text.length / 4);
  }
  
  /**
   * Batch processing (process multiple queries together)
   */
  async batchProcess(queries, model) {
    const batchedPrompt = queries
      .map((q, i) => `Query ${i + 1}: ${q}`)
      .join('\n\n');
    
    const prompt = `Answer each of these banking queries separately:

${batchedPrompt}

Provide answers in this format:
Answer 1: [answer]
Answer 2: [answer]
...`;

    const response = await model.call(prompt);
    
    // Parse responses
    const answers = response.content
      .split(/Answer \d+:/)
      .slice(1)
      .map(a => a.trim());
    
    return answers;
  }
  
  /**
   * Get usage statistics
   */
  getUsageStats() {
    const cacheHitRate = this.usageStats.cacheHits / 
      (this.usageStats.cacheHits + this.usageStats.cacheMisses);
    
    const costSaved = this.usageStats.cacheHits * 0.01; // Estimated
    
    return {
      ...this.usageStats,
      cacheHitRate: cacheHitRate.toFixed(2),
      costSaved: costSaved.toFixed(4)
    };
  }
}

/**
 * Practical Example: ENBD Cost-Efficient Chat
 */
class ENBDCostEfficientChat {
  constructor() {
    this.costService = new CostOptimizationService();
  }
  
  /**
   * Answer question with cost optimization
   */
  async answerQuestion(question, context = '') {
    // Use semantic cache
    const result = await this.costService.semanticCache(
      question,
      async () => {
        // Select optimal model
        const modelName = await this.costService.selectOptimalModel(
          question,
          context
        );
        
        console.log(`Using model: ${modelName}`);
        
        const model = new ChatOpenAI({
          modelName: modelName,
          temperature: 0.7,
          openAIApiKey: process.env.OPENAI_API_KEY
        });
        
        // Build prompt
        const prompt = context 
          ? `Context: ${context}\n\nQuestion: ${question}\n\nAnswer:`
          : `Question: ${question}\n\nAnswer:`;
        
        // Generate response
        const response = await model.call(prompt);
        
        // Calculate cost
        const inputTokens = this.costService.countTokens(prompt);
        const outputTokens = this.costService.countTokens(response.content);
        
        const cost = this.costService.calculateCost(
          modelName,
          inputTokens,
          outputTokens
        );
        
        console.log(`Cost: $${cost.totalCost.toFixed(6)}`);
        console.log(`Tokens: ${cost.totalTokens}`);
        
        return {
          response: response.content,
          cost: cost.totalCost,
          tokens: cost.totalTokens
        };
      }
    );
    
    return result;
  }
  
  /**
   * Batch process FAQ
   */
  async batchAnswerFAQs(questions) {
    console.log(`Batch processing ${questions.length} questions`);
    
    // Use cheaper model for batch
    const model = new ChatOpenAI({
      modelName: 'gpt-3.5-turbo',
      temperature: 0.7,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    const answers = await this.costService.batchProcess(questions, model);
    
    return questions.map((q, i) => ({
      question: q,
      answer: answers[i]
    }));
  }
  
  /**
   * Get cost report
   */
  getCostReport() {
    const stats = this.costService.getUsageStats();
    
    return {
      totalCost: `$${stats.totalCost.toFixed(6)}`,
      totalTokens: stats.totalTokens,
      cacheHitRate: `${(stats.cacheHitRate * 100).toFixed(1)}%`,
      costSaved: `$${stats.costSaved}`,
      cacheHits: stats.cacheHits,
      cacheMisses: stats.cacheMisses
    };
  }
}

/**
 * Usage Example
 */
async function exampleCostOptimization() {
  const chat = new ENBDCostEfficientChat();
  
  // Query 1 (cache miss)
  console.log('\n=== Query 1 ===');
  const result1 = await chat.answerQuestion(
    'What is minimum balance for savings account?'
  );
  console.log('Answer:', result1.response);
  console.log('From Cache:', result1.fromCache);
  
  // Query 2 (same as query 1 - cache hit)
  console.log('\n=== Query 2 (duplicate) ===');
  const result2 = await chat.answerQuestion(
    'What is minimum balance for savings account?'
  );
  console.log('Answer:', result2.response);
  console.log('From Cache:', result2.fromCache);
  
  // Query 3 (different question)
  console.log('\n=== Query 3 ===');
  const result3 = await chat.answerQuestion(
    'How to apply for credit card?'
  );
  console.log('Answer:', result3.response);
  
  // Batch processing
  console.log('\n=== Batch Processing ===');
  const faqs = [
    'What are the credit card fees?',
    'How long does loan approval take?',
    'Can I transfer money internationally?'
  ];
  
  const batchResults = await chat.batchAnswerFAQs(faqs);
  batchResults.forEach(r => {
    console.log(`Q: ${r.question}`);
    console.log(`A: ${r.answer}\n`);
  });
  
  // Cost report
  console.log('\n=== Cost Report ===');
  const report = chat.getCostReport();
  console.log(report);
}

module.exports = {
  CostOptimizationService,
  ENBDCostEfficientChat
};
```

**Cost Optimization Strategies:**

1. ✅ **Semantic Caching** - Cache similar queries
2. ✅ **Model Selection** - Use cheaper models when possible
3. ✅ **Batch Processing** - Process multiple queries together
4. ✅ **Token Counting** - Track and optimize token usage
5. ✅ **Response Caching** - Redis-based caching
6. ✅ **Usage Analytics** - Track costs and savings

---

**Summary Q11-Q15:**
- Advanced RAG (re-ranking, query expansion, HyDE) ✅
- Hallucination detection (grounding, CoVe, attribution) ✅
- Conversation memory optimization (token-aware, summarization, entities) ✅
- Prompt optimization and A/B testing ✅
- Cost optimization (caching, model selection, batch processing) ✅

Continuing with GenAI Q16-Q20...
