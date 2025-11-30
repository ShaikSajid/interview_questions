# Q55: Context Window Optimization

## Question: How do you optimize context window usage in production GenAI applications with multi-agent systems?

---

## Answer:

Context window optimization is critical for cost management, performance, and maintaining relevant context in GenAI applications. With GPT-4's 8K-128K token limits and costs of $0.03/1K input tokens, efficient context management directly impacts both latency and operational costs.

---

## Table of Contents

1. [Context Window Fundamentals](#context-window-fundamentals)
2. [Token Management](#token-management)
3. [Summarization Techniques](#summarization-techniques)
4. [Selective Memory Retrieval](#selective-memory-retrieval)
5. [QA Automation Optimization](#qa-automation-optimization)
6. [Banking Optimization](#banking-optimization)
7. [Production Strategies](#production-strategies)

---

## 1. Context Window Fundamentals

### Understanding Token Limits

```
┌──────────────────────────────────────────────────────────────┐
│          MODEL CONTEXT WINDOW COMPARISON                     │
└──────────────────────────────────────────────────────────────┘

Model                Context Window    Input Cost      Output Cost
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

GPT-4 Turbo         128K tokens       $0.01/1K        $0.03/1K
GPT-4               8K tokens         $0.03/1K        $0.06/1K
GPT-3.5 Turbo       16K tokens        $0.0015/1K      $0.002/1K
Claude 2            100K tokens       $0.008/1K       $0.024/1K
Claude 3 Opus       200K tokens       $0.015/1K       $0.075/1K

COST CALCULATION EXAMPLE (GPT-4 Turbo):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Scenario: QA Test Generation
• Input: 5,000 tokens (JIRA ticket + SOP + history)
• Output: 2,000 tokens (test plan)
• Cost per execution: (5K × $0.01) + (2K × $0.03) = $0.11

❌ WITHOUT OPTIMIZATION:
• 1,000 executions/day × $0.11 = $110/day = $3,300/month

✅ WITH OPTIMIZATION (50% token reduction):
• 1,000 executions/day × $0.055 = $55/day = $1,650/month
• SAVINGS: $1,650/month (50%)

┌────────────────────────────────────────────────────────────┐
│  TOKEN BREAKDOWN BY AGENT (QA Automation)                  │
└────────────────────────────────────────────────────────────┘

Agent                     Input Tokens    Output Tokens    Cost
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. JIRA Reader            500             200              $0.011
2. SOP Analyzer           3,000           800              $0.054
3. Test Generator         2,000           2,500            $0.095
4. Test Executor          1,000           500              $0.025
5. Artifact Manager       500             300              $0.014
6. Report Generator       1,500           1,200            $0.051

TOTAL PER WORKFLOW:       8,500           5,500            $0.250

OPTIMIZATION TARGET: Reduce to 4,000 input + 3,000 output = $0.130 (48% savings)
```

---

## 2. Token Management

### Token Counting and Budgeting

```typescript
import { encoding_for_model } from '@dqbd/tiktoken';

/**
 * Token Manager for Context Window Optimization
 */
class TokenManager {
  private encoder: any;
  private modelName: string;
  private maxContextWindow: number;
  
  constructor(modelName: string = 'gpt-4-turbo') {
    this.modelName = modelName;
    this.encoder = encoding_for_model(modelName as any);
    
    // Set max context window based on model
    this.maxContextWindow = this.getMaxContextWindow(modelName);
  }
  
  private getMaxContextWindow(model: string): number {
    const limits: { [key: string]: number } = {
      'gpt-4-turbo': 128000,
      'gpt-4': 8192,
      'gpt-3.5-turbo': 16384,
      'claude-2': 100000,
    };
    return limits[model] || 8192;
  }
  
  /**
   * Count tokens in text
   */
  countTokens(text: string): number {
    const tokens = this.encoder.encode(text);
    return tokens.length;
  }
  
  /**
   * Count tokens in messages array
   */
  countMessagesTokens(messages: Array<{ role: string; content: string }>): number {
    let totalTokens = 0;
    
    for (const message of messages) {
      // 4 tokens per message (role, content wrappers)
      totalTokens += 4;
      totalTokens += this.countTokens(message.role);
      totalTokens += this.countTokens(message.content);
    }
    
    // 2 tokens for reply priming
    totalTokens += 2;
    
    return totalTokens;
  }
  
  /**
   * Estimate cost
   */
  estimateCost(inputTokens: number, outputTokens: number): number {
    // GPT-4 Turbo pricing
    const inputCost = (inputTokens / 1000) * 0.01;
    const outputCost = (outputTokens / 1000) * 0.03;
    return inputCost + outputCost;
  }
  
  /**
   * Check if within budget
   */
  isWithinBudget(
    messages: Array<{ role: string; content: string }>,
    maxTokens: number = 8000
  ): boolean {
    const tokenCount = this.countMessagesTokens(messages);
    return tokenCount <= maxTokens;
  }
  
  /**
   * Truncate text to fit token budget
   */
  truncateToTokenBudget(text: string, maxTokens: number): string {
    const tokens = this.encoder.encode(text);
    
    if (tokens.length <= maxTokens) {
      return text;
    }
    
    // Truncate tokens
    const truncatedTokens = tokens.slice(0, maxTokens);
    const truncatedText = this.encoder.decode(truncatedTokens);
    
    return truncatedText + '... [truncated]';
  }
  
  /**
   * Get token budget allocation
   */
  allocateTokenBudget(totalBudget: number): {
    systemPrompt: number;
    context: number;
    userQuery: number;
    outputReserve: number;
  } {
    // Allocate tokens strategically
    return {
      systemPrompt: Math.floor(totalBudget * 0.05), // 5%
      context: Math.floor(totalBudget * 0.60), // 60%
      userQuery: Math.floor(totalBudget * 0.10), // 10%
      outputReserve: Math.floor(totalBudget * 0.25), // 25%
    };
  }
}

/**
 * Usage Example
 */
async function tokenManagementExample() {
  const tokenManager = new TokenManager('gpt-4-turbo');
  
  // Count tokens
  const text = 'This is a sample JIRA ticket description for testing...';
  const tokenCount = tokenManager.countTokens(text);
  console.log(`Token count: ${tokenCount}`);
  
  // Check budget
  const messages = [
    { role: 'system', content: 'You are a QA test generator.' },
    { role: 'user', content: text },
  ];
  
  const isWithinBudget = tokenManager.isWithinBudget(messages, 8000);
  console.log(`Within budget: ${isWithinBudget}`);
  
  // Estimate cost
  const cost = tokenManager.estimateCost(5000, 2000);
  console.log(`Estimated cost: $${cost.toFixed(4)}`);
  
  // Allocate budget
  const allocation = tokenManager.allocateTokenBudget(8000);
  console.log('Token allocation:', allocation);
}
```

---

## 3. Summarization Techniques

### Hierarchical Summarization

```typescript
import { ChatOpenAI } from '@langchain/openai';
import { PromptTemplate } from '@langchain/core/prompts';

/**
 * Summarization Manager for Context Compression
 */
class SummarizationManager {
  private llm: ChatOpenAI;
  private tokenManager: TokenManager;
  
  constructor() {
    this.llm = new ChatOpenAI({
      modelName: 'gpt-3.5-turbo', // Use cheaper model for summarization
      temperature: 0,
    });
    this.tokenManager = new TokenManager();
  }
  
  /**
   * Extractive summarization (fast, cheap)
   */
  extractiveSummarize(text: string, sentenceCount: number = 5): string {
    const sentences = text.split(/[.!?]+/).filter((s) => s.trim().length > 0);
    
    // Simple scoring: sentence length and position
    const scoredSentences = sentences.map((sentence, index) => ({
      sentence,
      score: sentence.split(' ').length * (1 - index / sentences.length),
    }));
    
    // Sort by score and take top N
    const topSentences = scoredSentences
      .sort((a, b) => b.score - a.score)
      .slice(0, sentenceCount)
      .map((s) => s.sentence.trim());
    
    return topSentences.join('. ') + '.';
  }
  
  /**
   * Abstractive summarization (high quality, more expensive)
   */
  async abstractiveSummarize(
    text: string,
    maxTokens: number = 150
  ): Promise<string> {
    const prompt = PromptTemplate.fromTemplate(`
Summarize the following text in {maxTokens} tokens or less. 
Focus on key information and action items.

Text:
{text}

Summary:`);
    
    const formattedPrompt = await prompt.format({
      text,
      maxTokens: maxTokens.toString(),
    });
    
    const response = await this.llm.invoke(formattedPrompt);
    return response.content as string;
  }
  
  /**
   * Hierarchical summarization for large documents
   */
  async hierarchicalSummarize(
    text: string,
    chunkSize: number = 2000,
    maxTokens: number = 500
  ): Promise<string> {
    // Split into chunks
    const chunks = this.splitIntoChunks(text, chunkSize);
    
    console.log(`[Summarization] Processing ${chunks.length} chunks`);
    
    // Summarize each chunk
    const chunkSummaries = await Promise.all(
      chunks.map((chunk) => this.abstractiveSummarize(chunk, 150))
    );
    
    // If summaries are still too long, summarize summaries
    const combinedSummaries = chunkSummaries.join('\n\n');
    const summaryTokens = this.tokenManager.countTokens(combinedSummaries);
    
    if (summaryTokens > maxTokens) {
      console.log(`[Summarization] Second-level summarization needed`);
      return await this.abstractiveSummarize(combinedSummaries, maxTokens);
    }
    
    return combinedSummaries;
  }
  
  /**
   * Progressive summarization (iterate to compress)
   */
  async progressiveSummarize(
    text: string,
    targetTokens: number
  ): Promise<string> {
    let currentText = text;
    let currentTokens = this.tokenManager.countTokens(currentText);
    let iteration = 1;
    
    while (currentTokens > targetTokens) {
      console.log(
        `[Progressive] Iteration ${iteration}: ${currentTokens} tokens → target ${targetTokens}`
      );
      
      const compressionRatio = targetTokens / currentTokens;
      const chunkTargetTokens = Math.floor(150 * compressionRatio);
      
      currentText = await this.abstractiveSummarize(
        currentText,
        chunkTargetTokens
      );
      currentTokens = this.tokenManager.countTokens(currentText);
      iteration++;
      
      // Safety: max 5 iterations
      if (iteration > 5) break;
    }
    
    return currentText;
  }
  
  /**
   * Split text into chunks
   */
  private splitIntoChunks(text: string, chunkSize: number): string[] {
    const words = text.split(' ');
    const chunks: string[] = [];
    
    for (let i = 0; i < words.length; i += chunkSize) {
      chunks.push(words.slice(i, i + chunkSize).join(' '));
    }
    
    return chunks;
  }
}

/**
 * QA Example: Summarize JIRA Ticket
 */
async function qaTicketSummarization() {
  const summarizer = new SummarizationManager();
  
  const longTicketDescription = `
    As a user, I want to be able to log in to the system using my email and password.
    
    Acceptance Criteria:
    1. Login form should have email and password fields
    2. Email validation should check for valid format
    3. Password should be masked
    4. Show error message on invalid credentials
    5. Redirect to dashboard on successful login
    6. Remember me option should persist session
    7. Forgot password link should navigate to password reset
    8. Login attempts should be rate limited to prevent brute force
    
    Technical Details:
    - Use JWT tokens for authentication
    - Password should be hashed with bcrypt
    - Session should expire after 24 hours
    - Implement CSRF protection
    - Add logging for security audit
    
    Dependencies:
    - Backend API endpoint /api/auth/login must be ready
    - Database schema for users table must be finalized
  `;
  
  // Extractive (fast, free)
  const extractive = summarizer.extractiveSummarize(longTicketDescription, 3);
  console.log('\n[Extractive Summary]:\n', extractive);
  
  // Abstractive (higher quality)
  const abstractive = await summarizer.abstractiveSummarize(
    longTicketDescription,
    100
  );
  console.log('\n[Abstractive Summary]:\n', abstractive);
  
  // Cost comparison
  const originalTokens = new TokenManager().countTokens(longTicketDescription);
  const extractiveTokens = new TokenManager().countTokens(extractive);
  const abstractiveTokens = new TokenManager().countTokens(abstractive);
  
  console.log('\n[Token Comparison]:');
  console.log(`Original: ${originalTokens} tokens`);
  console.log(`Extractive: ${extractiveTokens} tokens (${Math.round((1 - extractiveTokens / originalTokens) * 100)}% reduction)`);
  console.log(`Abstractive: ${abstractiveTokens} tokens (${Math.round((1 - abstractiveTokens / originalTokens) * 100)}% reduction)`);
}
```

---

## 4. Selective Memory Retrieval

### Relevance-Based Filtering

```typescript
/**
 * Selective Memory Retrieval Manager
 */
class SelectiveMemoryManager {
  private tokenManager: TokenManager;
  private summarizer: SummarizationManager;
  
  constructor() {
    this.tokenManager = new TokenManager();
    this.summarizer = new SummarizationManager();
  }
  
  /**
   * Retrieve memory with token budget
   */
  async retrieveWithBudget(
    query: string,
    allMemories: Array<{ id: string; content: string; timestamp: Date; relevance?: number }>,
    tokenBudget: number
  ): Promise<string> {
    // 1. Score memories by relevance (simplified scoring)
    const scoredMemories = allMemories.map((memory) => ({
      ...memory,
      relevance: this.calculateRelevance(query, memory.content, memory.timestamp),
    }));
    
    // 2. Sort by relevance
    scoredMemories.sort((a, b) => b.relevance! - a.relevance!);
    
    // 3. Select memories within token budget
    let selectedMemories: typeof scoredMemories = [];
    let currentTokens = 0;
    
    for (const memory of scoredMemories) {
      const memoryTokens = this.tokenManager.countTokens(memory.content);
      
      if (currentTokens + memoryTokens <= tokenBudget) {
        selectedMemories.push(memory);
        currentTokens += memoryTokens;
      } else {
        break;
      }
    }
    
    console.log(`[Memory] Selected ${selectedMemories.length}/${allMemories.length} memories (${currentTokens}/${tokenBudget} tokens)`);
    
    // 4. Format selected memories
    return selectedMemories.map((m) => m.content).join('\n\n');
  }
  
  /**
   * Calculate relevance score
   */
  private calculateRelevance(
    query: string,
    content: string,
    timestamp: Date
  ): number {
    // Simple relevance scoring (in production, use embeddings)
    const queryWords = new Set(query.toLowerCase().split(/\s+/));
    const contentWords = content.toLowerCase().split(/\s+/);
    
    // Keyword overlap
    const overlapCount = contentWords.filter((word) =>
      queryWords.has(word)
    ).length;
    const keywordScore = overlapCount / queryWords.size;
    
    // Recency score (newer = higher)
    const ageInDays = (Date.now() - timestamp.getTime()) / (1000 * 60 * 60 * 24);
    const recencyScore = 1 / (1 + ageInDays / 30); // Decay over 30 days
    
    // Combined score
    return keywordScore * 0.7 + recencyScore * 0.3;
  }
  
  /**
   * Time-based windowing
   */
  async timeBasedWindow(
    memories: Array<{ content: string; timestamp: Date }>,
    windowDays: number = 7,
    tokenBudget: number = 2000
  ): Promise<string> {
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - windowDays);
    
    // Filter by time window
    const recentMemories = memories.filter((m) => m.timestamp >= cutoffDate);
    
    console.log(`[Window] ${recentMemories.length}/${memories.length} memories in ${windowDays}-day window`);
    
    // Concatenate and truncate if needed
    let combinedContent = recentMemories.map((m) => m.content).join('\n\n');
    
    const currentTokens = this.tokenManager.countTokens(combinedContent);
    if (currentTokens > tokenBudget) {
      // Summarize to fit budget
      combinedContent = await this.summarizer.progressiveSummarize(
        combinedContent,
        tokenBudget
      );
    }
    
    return combinedContent;
  }
  
  /**
   * Priority-based selection
   */
  priorityBasedSelection(
    memories: Array<{
      content: string;
      priority: 'critical' | 'high' | 'medium' | 'low';
    }>,
    tokenBudget: number
  ): string {
    const priorityOrder = { critical: 4, high: 3, medium: 2, low: 1 };
    
    // Sort by priority
    const sortedMemories = [...memories].sort(
      (a, b) => priorityOrder[b.priority] - priorityOrder[a.priority]
    );
    
    // Select within budget
    let selectedMemories: typeof memories = [];
    let currentTokens = 0;
    
    for (const memory of sortedMemories) {
      const memoryTokens = this.tokenManager.countTokens(memory.content);
      
      if (currentTokens + memoryTokens <= tokenBudget) {
        selectedMemories.push(memory);
        currentTokens += memoryTokens;
      }
    }
    
    console.log(`[Priority] Selected ${selectedMemories.length} memories by priority`);
    
    return selectedMemories.map((m) => m.content).join('\n\n');
  }
}

/**
 * QA Example: Selective SOP Retrieval
 */
async function qaSelectiveRetrieval() {
  const memoryManager = new SelectiveMemoryManager();
  
  const sopMemories = [
    {
      id: 'SOP-001',
      content: 'Login testing standards: Check email validation, password strength...',
      timestamp: new Date('2024-11-01'),
    },
    {
      id: 'SOP-002',
      content: 'API testing standards: Validate response codes, error messages...',
      timestamp: new Date('2024-10-15'),
    },
    {
      id: 'SOP-003',
      content: 'UI testing standards: Check responsive design, accessibility...',
      timestamp: new Date('2024-09-20'),
    },
  ];
  
  const query = 'Implement login functionality with email validation';
  const tokenBudget = 500;
  
  const selectedMemories = await memoryManager.retrieveWithBudget(
    query,
    sopMemories,
    tokenBudget
  );
  
  console.log('\n[Selected SOPs]:\n', selectedMemories);
}
```

---

## 5. QA Automation Optimization

### QA-Specific Context Management

```typescript
/**
 * QA Context Optimizer
 */
class QAContextOptimizer {
  private tokenManager: TokenManager;
  private summarizer: SummarizationManager;
  private memoryManager: SelectiveMemoryManager;
  
  constructor() {
    this.tokenManager = new TokenManager();
    this.summarizer = new SummarizationManager();
    this.memoryManager = new SelectiveMemoryManager();
  }
  
  /**
   * Optimize JIRA ticket context
   */
  async optimizeTicketContext(ticket: {
    summary: string;
    description: string;
    comments: string[];
    attachments: string[];
  }): Promise<{ optimized: string; tokensSaved: number }> {
    const originalContent = `
Summary: ${ticket.summary}
Description: ${ticket.description}
Comments: ${ticket.comments.join('\n')}
Attachments: ${ticket.attachments.join(', ')}
    `;
    
    const originalTokens = this.tokenManager.countTokens(originalContent);
    
    // Strategy 1: Summarize long description
    let optimizedDescription = ticket.description;
    if (this.tokenManager.countTokens(ticket.description) > 500) {
      optimizedDescription = await this.summarizer.abstractiveSummarize(
        ticket.description,
        200
      );
    }
    
    // Strategy 2: Keep only last 3 comments
    const recentComments = ticket.comments.slice(-3);
    
    // Strategy 3: Metadata only for attachments (not full content)
    const attachmentMeta = ticket.attachments.map((att) => ({
      name: att,
      type: att.split('.').pop(),
    }));
    
    const optimizedContent = `
Summary: ${ticket.summary}
Description: ${optimizedDescription}
Recent Comments (${recentComments.length}): ${recentComments.join('\n')}
Attachments (${attachmentMeta.length}): ${attachmentMeta.map((a) => a.name).join(', ')}
    `;
    
    const optimizedTokens = this.tokenManager.countTokens(optimizedContent);
    const tokensSaved = originalTokens - optimizedTokens;
    
    console.log(`[Ticket Optimization] Saved ${tokensSaved} tokens (${Math.round((tokensSaved / originalTokens) * 100)}%)`);
    
    return {
      optimized: optimizedContent,
      tokensSaved,
    };
  }
  
  /**
   * Optimize test logs
   */
  optimizeTestLogs(logs: string[]): string {
    // Keep only: errors, warnings, and last 5 info logs
    const errors = logs.filter((log) => log.includes('[ERROR]'));
    const warnings = logs.filter((log) => log.includes('[WARN]'));
    const infos = logs.filter((log) => log.includes('[INFO]')).slice(-5);
    
    const optimizedLogs = [
      `Errors (${errors.length}):`,
      ...errors,
      `Warnings (${warnings.length}):`,
      ...warnings,
      `Recent Info (${infos.length}):`,
      ...infos,
    ];
    
    return optimizedLogs.join('\n');
  }
  
  /**
   * Optimize screenshot metadata
   */
  optimizeScreenshots(screenshots: Array<{
    path: string;
    timestamp: Date;
    size: number;
  }>): string {
    // Don't include screenshot bytes, just metadata
    return screenshots
      .map(
        (ss) =>
          `Screenshot: ${ss.path} (${new Date(ss.timestamp).toISOString()}, ${(ss.size / 1024).toFixed(2)}KB)`
      )
      .join('\n');
  }
  
  /**
   * Complete QA workflow optimization
   */
  async optimizeQAWorkflow(workflowState: {
    ticket: any;
    sops: string[];
    similarTickets: any[];
    testLogs: string[];
    screenshots: any[];
  }): Promise<{ context: string; totalTokens: number; costEstimate: number }> {
    // 1. Optimize ticket
    const { optimized: optimizedTicket } = await this.optimizeTicketContext(
      workflowState.ticket
    );
    
    // 2. Select top 2 most relevant SOPs
    const topSops = workflowState.sops.slice(0, 2).join('\n\n');
    
    // 3. Summarize similar tickets
    const similarTicketsSummary = `Found ${workflowState.similarTickets.length} similar tickets with avg resolution time of 2 hours`;
    
    // 4. Optimize logs
    const optimizedLogs = this.optimizeTestLogs(workflowState.testLogs);
    
    // 5. Screenshot metadata only
    const screenshotMeta = this.optimizeScreenshots(workflowState.screenshots);
    
    // Combine all context
    const context = `
=== TICKET ===
${optimizedTicket}

=== RELEVANT SOPs (2) ===
${topSops}

=== SIMILAR TICKETS ===
${similarTicketsSummary}

=== TEST LOGS (FILTERED) ===
${optimizedLogs}

=== SCREENSHOTS ===
${screenshotMeta}
    `;
    
    const totalTokens = this.tokenManager.countTokens(context);
    const costEstimate = this.tokenManager.estimateCost(totalTokens, 2000);
    
    console.log(`[QA Workflow] Total context: ${totalTokens} tokens, Estimated cost: $${costEstimate.toFixed(4)}`);
    
    return { context, totalTokens, costEstimate };
  }
}

/**
 * Usage Example
 */
async function qaOptimizationExample() {
  const optimizer = new QAContextOptimizer();
  
  const workflowState = {
    ticket: {
      summary: 'Implement user login',
      description:
        'As a user, I want to log in with email and password. The login form should validate email format, mask password, show error messages on invalid credentials, and redirect to dashboard on success. Additional requirements: remember me option, forgot password link, rate limiting for brute force prevention. Technical details: use JWT tokens, bcrypt for password hashing, 24-hour session expiry, CSRF protection, security audit logging.',
      comments: [
        'Comment 1: Please prioritize email validation',
        'Comment 2: Use OAuth as alternative',
        'Comment 3: Add 2FA support',
        'Comment 4: Update: 2FA is out of scope',
        'Comment 5: Ready for testing',
      ],
      attachments: ['login_mockup.png', 'flow_diagram.pdf'],
    },
    sops: [
      'SOP-AUTH-001: Login testing standards...',
      'SOP-SEC-005: Security testing checklist...',
      'SOP-UI-010: UI testing guidelines...',
    ],
    similarTickets: [
      { id: 'QA-100', resolution: '2h' },
      { id: 'QA-150', resolution: '1.5h' },
    ],
    testLogs: [
      '[INFO] Test started',
      '[INFO] Navigating to login page',
      '[ERROR] Email validation failed',
      '[WARN] Password field not masked',
      '[INFO] Retrying...',
      '[INFO] Test case 1 passed',
      '[INFO] Test case 2 passed',
    ],
    screenshots: [
      { path: '/tmp/login_page.png', timestamp: new Date(), size: 102400 },
      { path: '/tmp/error_state.png', timestamp: new Date(), size: 98304 },
    ],
  };
  
  const result = await optimizer.optimizeQAWorkflow(workflowState);
  
  console.log('\n=== OPTIMIZED CONTEXT ===');
  console.log(result.context);
  console.log(`\nTotal tokens: ${result.totalTokens}`);
  console.log(`Cost estimate: $${result.costEstimate.toFixed(4)}`);
}
```

---

## 6. Banking Optimization

### Banking-Specific Context Management

```typescript
/**
 * Banking Context Optimizer
 */
class BankingContextOptimizer {
  private tokenManager: TokenManager;
  private summarizer: SummarizationManager;
  
  constructor() {
    this.tokenManager = new TokenManager();
    this.summarizer = new SummarizationManager();
  }
  
  /**
   * Optimize customer profile
   */
  async optimizeCustomerProfile(customer: {
    id: string;
    personalInfo: any;
    accountHistory: any[];
    transactionHistory: any[];
    loanHistory: any[];
  }): Promise<string> {
    // Keep: ID, key personal info, summary of histories
    const recentTransactions = customer.transactionHistory.slice(-10);
    const activeLoan = customer.loanHistory.find((loan) => loan.status === 'ACTIVE');
    
    const optimizedProfile = {
      customerId: customer.id,
      name: customer.personalInfo.name,
      riskScore: customer.personalInfo.riskScore,
      accountSummary: {
        totalAccounts: customer.accountHistory.length,
        activeAccounts: customer.accountHistory.filter((acc) => acc.status === 'ACTIVE')
          .length,
      },
      transactionSummary: {
        recentCount: recentTransactions.length,
        avgAmount: this.calculateAverage(recentTransactions, 'amount'),
      },
      loanStatus: activeLoan ? 'HAS_ACTIVE_LOAN' : 'NO_ACTIVE_LOAN',
    };
    
    return JSON.stringify(optimizedProfile, null, 2);
  }
  
  /**
   * Optimize transaction history
   */
  optimizeTransactionHistory(transactions: any[]): string {
    // Group by type and aggregate
    const grouped = transactions.reduce((acc, txn) => {
      const type = txn.type;
      if (!acc[type]) {
        acc[type] = { count: 0, totalAmount: 0 };
      }
      acc[type].count++;
      acc[type].totalAmount += txn.amount;
      return acc;
    }, {} as any);
    
    const summary = Object.entries(grouped)
      .map(([type, data]: [string, any]) => {
        return `${type}: ${data.count} transactions, Total: AED ${data.totalAmount.toLocaleString()}`;
      })
      .join('\n');
    
    return `Transaction History (Last ${transactions.length}):\n${summary}`;
  }
  
  /**
   * Optimize conversation history
   */
  async optimizeConversationHistory(
    conversationTurns: Array<{ user: string; agent: string; timestamp: Date }>
  ): Promise<string> {
    // Keep last 5 turns
    const recentTurns = conversationTurns.slice(-5);
    
    // If still too long, summarize
    const conversation = recentTurns
      .map((turn) => `User: ${turn.user}\nAgent: ${turn.agent}`)
      .join('\n\n');
    
    const tokens = this.tokenManager.countTokens(conversation);
    
    if (tokens > 1000) {
      return await this.summarizer.abstractiveSummarize(conversation, 300);
    }
    
    return conversation;
  }
  
  /**
   * Complete banking workflow optimization
   */
  async optimizeBankingWorkflow(workflowState: {
    customer: any;
    conversationHistory: any[];
    fraudPatterns: any[];
    creditBureauData: any;
  }): Promise<{ context: string; totalTokens: number }> {
    // 1. Optimize customer profile
    const optimizedProfile = await this.optimizeCustomerProfile(
      workflowState.customer
    );
    
    // 2. Optimize conversation
    const optimizedConversation = await this.optimizeConversationHistory(
      workflowState.conversationHistory
    );
    
    // 3. Fraud patterns summary
    const fraudSummary =
      workflowState.fraudPatterns.length > 0
        ? `ALERT: ${workflowState.fraudPatterns.length} similar fraud patterns detected`
        : 'No fraud patterns detected';
    
    // 4. Credit bureau summary
    const creditSummary = `Credit Score: ${workflowState.creditBureauData.score}, Status: ${workflowState.creditBureauData.status}`;
    
    const context = `
=== CUSTOMER PROFILE ===
${optimizedProfile}

=== RECENT CONVERSATION ===
${optimizedConversation}

=== FRAUD CHECK ===
${fraudSummary}

=== CREDIT BUREAU ===
${creditSummary}
    `;
    
    const totalTokens = this.tokenManager.countTokens(context);
    
    console.log(`[Banking Workflow] Total context: ${totalTokens} tokens`);
    
    return { context, totalTokens };
  }
  
  private calculateAverage(items: any[], field: string): number {
    const sum = items.reduce((acc, item) => acc + item[field], 0);
    return sum / items.length;
  }
}
```

---

## 7. Production Strategies

### Complete Optimization Framework

```typescript
/**
 * Production Context Window Manager
 */
class ProductionContextManager {
  private tokenManager: TokenManager;
  private qaOptimizer: QAContextOptimizer;
  private bankingOptimizer: BankingContextOptimizer;
  
  constructor() {
    this.tokenManager = new TokenManager('gpt-4-turbo');
    this.qaOptimizer = new QAContextOptimizer();
    this.bankingOptimizer = new BankingContextOptimizer();
  }
  
  /**
   * Dynamic context allocation based on priority
   */
  async dynamicContextAllocation(
    contextSources: Array<{
      name: string;
      content: string;
      priority: number; // 1-10
      required: boolean;
    }>,
    totalBudget: number
  ): Promise<string> {
    // Sort by priority
    const sorted = [...contextSources].sort((a, b) => b.priority - a.priority);
    
    // Always include required context
    const required = sorted.filter((c) => c.required);
    const optional = sorted.filter((c) => !c.required);
    
    let selectedContext: typeof contextSources = [...required];
    let currentTokens = required.reduce(
      (sum, c) => sum + this.tokenManager.countTokens(c.content),
      0
    );
    
    // Add optional context within budget
    for (const context of optional) {
      const tokens = this.tokenManager.countTokens(context.content);
      if (currentTokens + tokens <= totalBudget) {
        selectedContext.push(context);
        currentTokens += tokens;
      }
    }
    
    console.log(
      `[Dynamic Allocation] Selected ${selectedContext.length}/${contextSources.length} context sources (${currentTokens}/${totalBudget} tokens)`
    );
    
    return selectedContext.map((c) => `[${c.name}]\n${c.content}`).join('\n\n');
  }
  
  /**
   * Streaming context for long outputs
   */
  async streamingContext(
    generator: AsyncGenerator<string>,
    tokenBudget: number
  ): Promise<string> {
    let accumulated = '';
    let tokens = 0;
    
    for await (const chunk of generator) {
      const chunkTokens = this.tokenManager.countTokens(chunk);
      
      if (tokens + chunkTokens > tokenBudget) {
        console.log(`[Streaming] Reached token budget at ${tokens} tokens`);
        break;
      }
      
      accumulated += chunk;
      tokens += chunkTokens;
    }
    
    return accumulated;
  }
  
  /**
   * Cost-aware context selection
   */
  selectContextByCost(
    contextOptions: Array<{ name: string; content: string; value: number }>,
    costBudget: number
  ): string {
    // Calculate cost per token of value
    const withCostEfficiency = contextOptions.map((option) => {
      const tokens = this.tokenManager.countTokens(option.content);
      const cost = this.tokenManager.estimateCost(tokens, 0);
      const efficiency = option.value / cost; // value per dollar
      
      return { ...option, tokens, cost, efficiency };
    });
    
    // Sort by efficiency
    withCostEfficiency.sort((a, b) => b.efficiency - a.efficiency);
    
    // Select within budget
    let selected: typeof withCostEfficiency = [];
    let totalCost = 0;
    
    for (const option of withCostEfficiency) {
      if (totalCost + option.cost <= costBudget) {
        selected.push(option);
        totalCost += option.cost;
      }
    }
    
    console.log(
      `[Cost-Aware] Selected ${selected.length} contexts for $${totalCost.toFixed(4)}`
    );
    
    return selected.map((c) => c.content).join('\n\n');
  }
  
  /**
   * Caching strategy
   */
  async contextWithCaching(
    staticContext: string,
    dynamicContext: string
  ): Promise<{ fullContext: string; cacheKey: string }> {
    // Static context can be cached (SOPs, system prompts)
    // Dynamic context changes per request (user query, current state)
    
    const cacheKey = this.hashContent(staticContext);
    
    // In production, check cache first
    // const cachedStatic = await redis.get(`context:${cacheKey}`);
    
    const fullContext = `
[STATIC CONTEXT - CACHED]
${staticContext}

[DYNAMIC CONTEXT]
${dynamicContext}
    `;
    
    // Cache static context
    // await redis.setex(`context:${cacheKey}`, 3600, staticContext);
    
    return { fullContext, cacheKey };
  }
  
  private hashContent(content: string): string {
    // Simple hash (in production, use crypto.createHash)
    return Buffer.from(content).toString('base64').substring(0, 16);
  }
}

/**
 * Optimization Best Practices Summary
 */
const optimizationStrategies = `
┌────────────────────────────────────────────────────────────┐
│         CONTEXT WINDOW OPTIMIZATION STRATEGIES             │
└────────────────────────────────────────────────────────────┘

STRATEGY                  SAVINGS     TRADE-OFF           USE WHEN
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Extractive Summary        20-40%      Loses nuance        Large docs, quick needs
Abstractive Summary       40-60%      API cost            Critical context
Hierarchical Summary      50-70%      Processing time     Very large docs
Token Truncation          Variable    Data loss           Non-critical context
Selective Retrieval       30-50%      May miss info       Large memory stores
Time-based Windowing      40-60%      Loses history       Recent data focus
Priority-based            30-50%      Requires scoring    Mixed importance
Caching                   Free        Staleness risk      Static content

COST OPTIMIZATION CHECKLIST:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✅ Use GPT-3.5-Turbo for summarization (10x cheaper than GPT-4)
✅ Cache static context (SOPs, system prompts)
✅ Batch API calls where possible
✅ Use extractive summarization first, abstractive only when needed
✅ Implement token budgets per agent
✅ Monitor token usage per workflow
✅ Set alerts for high-cost operations
✅ Use function calling for structured output (fewer tokens)
✅ Compress JSON (remove whitespace)
✅ Store metadata only, fetch full content on demand

ROI CALCULATION:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Before Optimization:
• 1,000 workflows/day
• 8,500 input tokens + 5,500 output tokens per workflow
• Cost: $0.25/workflow × 1,000 = $250/day = $7,500/month

After Optimization (50% reduction):
• 4,000 input tokens + 3,000 output tokens per workflow
• Cost: $0.13/workflow × 1,000 = $130/day = $3,900/month

Monthly Savings: $3,600 (48%)
Annual Savings: $43,200
`;

console.log(optimizationStrategies);
```

---

## Summary

### Optimization Framework

| Technique | Token Savings | Cost Impact | Latency | Quality Impact |
|-----------|--------------|-------------|---------|----------------|
| Extractive Summary | 20-40% | None (free) | +50ms | Minor |
| Abstractive Summary | 40-60% | $0.002/1K | +500ms | Minimal |
| Hierarchical Summary | 50-70% | $0.005/1K | +1s | Minimal |
| Selective Retrieval | 30-50% | None (filter) | +100ms | Moderate |
| Time Window | 40-60% | None (filter) | +50ms | Moderate |
| Priority Selection | 30-50% | None (sort) | +80ms | Low |
| Caching | Up to 100% | None (amortized) | -200ms | None |

### Implementation Priorities

1. **Quick Wins** (Implement First):
   - Token counting and budgeting
   - Extractive summarization for long text
   - Time-based windowing for memory
   - Caching static content

2. **Medium Term**:
   - Abstractive summarization for critical context
   - Hierarchical summarization for very large docs
   - Dynamic context allocation
   - Cost monitoring and alerts

3. **Advanced Optimization**:
   - Progressive summarization
   - Streaming context management
   - Cost-aware context selection
   - Multi-tier caching strategy

### Cost Savings Example (QA Automation)

**Before Optimization**: $7,500/month  
**After Optimization**: $3,900/month  
**Savings**: $3,600/month (48%)  
**Annual ROI**: $43,200

---

**Related Files:**
- `Q53_Conversational_Memory_Management.md` - Memory types
- `Q54_Memory_Persistence_Strategies.md` - Storage implementation
- `QA_Automation_Scenario.md` - Complete QA implementation
- `Tools_in_Agent_Systems.md` - Tool integration

---

**Document Version**: 1.0  
**Last Updated**: November 2025  
**For**: GenAI Interview Preparation

---

*End of Context Window Optimization Document*
