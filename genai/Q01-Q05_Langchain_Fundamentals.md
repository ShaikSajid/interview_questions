# GenAI Questions 1-5: Langchain Fundamentals

---

### Q1. What is Langchain and how do you implement a basic chain in Node.js?

**Answer:**

Langchain is a framework for developing applications powered by large language models (LLMs). It provides abstractions for chains, prompts, memory, agents, and more.

```javascript
const { ChatOpenAI } = require('@langchain/openai');
const { PromptTemplate } = require('@langchain/core/prompts');
const { LLMChain } = require('langchain/chains');
const { BufferMemory } = require('langchain/memory');

/**
 * Basic Langchain Service for ENBD Banking
 */
class ENBDLangchainService {
  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0.7,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }
  
  /**
   * Simple LLM Chain
   */
  async createSimpleChain() {
    const template = `You are a helpful banking assistant for Emirates NBD.
    
Customer Question: {question}

Provide a clear, professional response:`;

    const prompt = PromptTemplate.fromTemplate(template);
    
    const chain = new LLMChain({
      llm: this.model,
      prompt: prompt
    });
    
    return chain;
  }
  
  /**
   * Example: Customer Service Query
   */
  async answerCustomerQuery(question) {
    const chain = await this.createSimpleChain();
    
    const response = await chain.call({
      question: question
    });
    
    return response.text;
  }
  
  /**
   * Chain with Memory
   */
  async createChainWithMemory() {
    const template = `You are a banking assistant for ENBD. Use conversation history to provide context-aware responses.

Current conversation:
{chat_history}

Customer: {input}
Assistant:`;

    const prompt = PromptTemplate.fromTemplate(template);
    
    const memory = new BufferMemory({
      memoryKey: 'chat_history',
      returnMessages: true
    });
    
    const chain = new LLMChain({
      llm: this.model,
      prompt: prompt,
      memory: memory
    });
    
    return chain;
  }
  
  /**
   * Sequential Chain (Multiple Steps)
   */
  async createSequentialChain() {
    const { SequentialChain } = require('langchain/chains');
    
    // Step 1: Analyze transaction
    const analysisPrompt = PromptTemplate.fromTemplate(
      `Analyze this transaction for risk:
Transaction: {transaction}

Risk Analysis:`
    );
    
    const analysisChain = new LLMChain({
      llm: this.model,
      prompt: analysisPrompt,
      outputKey: 'risk_analysis'
    });
    
    // Step 2: Generate recommendation
    const recommendationPrompt = PromptTemplate.fromTemplate(
      `Based on this risk analysis:
{risk_analysis}

Recommendation:`
    );
    
    const recommendationChain = new LLMChain({
      llm: this.model,
      prompt: recommendationPrompt,
      outputKey: 'recommendation'
    });
    
    // Combine chains
    const overallChain = new SequentialChain({
      chains: [analysisChain, recommendationChain],
      inputVariables: ['transaction'],
      outputVariables: ['risk_analysis', 'recommendation']
    });
    
    return overallChain;
  }
  
  /**
   * Router Chain (Conditional routing)
   */
  async createRouterChain() {
    const { MultiPromptChain } = require('langchain/chains');
    const { LLMRouterChain } = require('langchain/chains');
    
    const accountInfoPrompt = new PromptTemplate({
      template: `You are an expert in account information for ENBD.

Question: {input}

Answer:`,
      inputVariables: ['input']
    });
    
    const transactionPrompt = new PromptTemplate({
      template: `You are an expert in transaction processing for ENBD.

Question: {input}

Answer:`,
      inputVariables: ['input']
    });
    
    const loanPrompt = new PromptTemplate({
      template: `You are an expert in loan products for ENBD.

Question: {input}

Answer:`,
      inputVariables: ['input']
    });
    
    const destinationChains = {
      'account_info': new LLMChain({
        llm: this.model,
        prompt: accountInfoPrompt
      }),
      'transactions': new LLMChain({
        llm: this.model,
        prompt: transactionPrompt
      }),
      'loans': new LLMChain({
        llm: this.model,
        prompt: loanPrompt
      })
    };
    
    return destinationChains;
  }
}

/**
 * Practical Example: Customer Support Chain
 */
class CustomerSupportChain {
  constructor() {
    this.langchain = new ENBDLangchainService();
  }
  
  async handleCustomerQuery(userId, query) {
    // Create chain with memory for this session
    const chain = await this.langchain.createChainWithMemory();
    
    // Get response
    const response = await chain.call({
      input: query
    });
    
    // Log for analytics
    await this.logInteraction(userId, query, response.text);
    
    return {
      answer: response.text,
      sessionId: chain.memory.sessionId
    };
  }
  
  async logInteraction(userId, query, response) {
    // Store in database for analytics
    console.log(`User ${userId}: ${query} -> ${response}`);
  }
}

module.exports = {
  ENBDLangchainService,
  CustomerSupportChain
};
```

**Key Concepts:**

1. ✅ **LLM Chain** - Basic chain with prompt template
2. ✅ **Memory** - Conversation history
3. ✅ **Sequential Chain** - Multi-step workflows
4. ✅ **Router Chain** - Conditional routing based on input

---

### Q2. How do you implement RAG (Retrieval Augmented Generation) with Langchain?

**Answer:**

RAG combines document retrieval with LLM generation to provide accurate, context-aware responses using your own data.

```javascript
const { ChatOpenAI } = require('@langchain/openai');
const { OpenAIEmbeddings } = require('@langchain/openai');
const { MemoryVectorStore } = require('langchain/vectorstores/memory');
const { RecursiveCharacterTextSplitter } = require('langchain/text_splitter');
const { RetrievalQAChain } = require('langchain/chains');
const { PDFLoader } = require('langchain/document_loaders/fs/pdf');

/**
 * RAG Service for ENBD Banking Documents
 */
class ENBDRAGService {
  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    this.embeddings = new OpenAIEmbeddings({
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    this.vectorStore = null;
  }
  
  /**
   * Load and process documents
   */
  async loadDocuments(filePaths) {
    const documents = [];
    
    // Load PDFs (policy documents, terms & conditions)
    for (const filePath of filePaths) {
      const loader = new PDFLoader(filePath);
      const docs = await loader.load();
      documents.push(...docs);
    }
    
    return documents;
  }
  
  /**
   * Split documents into chunks
   */
  async splitDocuments(documents) {
    const splitter = new RecursiveCharacterTextSplitter({
      chunkSize: 1000,
      chunkOverlap: 200
    });
    
    const splitDocs = await splitter.splitDocuments(documents);
    
    return splitDocs;
  }
  
  /**
   * Create vector store
   */
  async createVectorStore(documents) {
    const splitDocs = await this.splitDocuments(documents);
    
    this.vectorStore = await MemoryVectorStore.fromDocuments(
      splitDocs,
      this.embeddings
    );
    
    return this.vectorStore;
  }
  
  /**
   * Similarity search
   */
  async searchSimilarDocuments(query, k = 4) {
    if (!this.vectorStore) {
      throw new Error('Vector store not initialized');
    }
    
    const results = await this.vectorStore.similaritySearch(query, k);
    
    return results;
  }
  
  /**
   * Create RAG Chain
   */
  async createRAGChain() {
    if (!this.vectorStore) {
      throw new Error('Vector store not initialized');
    }
    
    const retriever = this.vectorStore.asRetriever({
      k: 4 // Top 4 most relevant documents
    });
    
    const chain = RetrievalQAChain.fromLLM(this.model, retriever, {
      returnSourceDocuments: true
    });
    
    return chain;
  }
  
  /**
   * Answer question using RAG
   */
  async answerQuestion(question) {
    const chain = await this.createRAGChain();
    
    const response = await chain.call({
      query: question
    });
    
    return {
      answer: response.text,
      sources: response.sourceDocuments.map(doc => ({
        content: doc.pageContent,
        metadata: doc.metadata
      }))
    };
  }
}

/**
 * Practical Example: ENBD Policy Q&A System
 */
class ENBDPolicyQASystem {
  constructor() {
    this.ragService = new ENBDRAGService();
    this.initialized = false;
  }
  
  /**
   * Initialize system with policy documents
   */
  async initialize() {
    console.log('Loading policy documents...');
    
    // Sample policy documents
    const policyDocuments = [
      {
        pageContent: `
ENBD Personal Loan Policy

Eligibility Criteria:
- UAE nationals and residents
- Minimum age: 21 years
- Minimum salary: AED 5,000/month
- Employment: Minimum 6 months with current employer

Loan Amount: AED 10,000 to AED 500,000
Tenure: 12 to 60 months
Interest Rate: Starting from 2.99% per annum
Processing Fee: 1% of loan amount (min AED 500)

Required Documents:
- Emirates ID
- Passport copy
- Salary certificate
- Bank statements (last 3 months)
        `,
        metadata: { source: 'personal-loan-policy.pdf', page: 1 }
      },
      {
        pageContent: `
ENBD Credit Card Policy

Eligibility:
- Minimum age: 21 years
- Minimum salary: AED 5,000/month (Platinum), AED 10,000/month (Signature)
- Valid UAE residence visa

Annual Fees:
- Classic: AED 200
- Platinum: AED 500
- Signature: AED 1,000

Credit Limit: Based on salary (up to 2x monthly salary)

Reward Points:
- 1 point per AED 10 spent on local purchases
- 2 points per AED 10 spent on international purchases
- Points valid for 2 years

Foreign Transaction Fee: 3% of transaction amount
        `,
        metadata: { source: 'credit-card-policy.pdf', page: 1 }
      },
      {
        pageContent: `
ENBD Account Opening Policy

Savings Account:
- Minimum opening balance: AED 3,000
- Minimum balance to avoid fees: AED 3,000
- Monthly fee (if below minimum): AED 25

Current Account:
- Minimum opening balance: AED 10,000
- Minimum balance to avoid fees: AED 10,000
- Monthly fee (if below minimum): AED 50

Required Documents:
- Original Emirates ID
- Original passport with valid residence visa
- Salary certificate (if employed)
- Trade license (if self-employed)

Account activation: Within 24 hours
Debit card delivery: 3-5 business days
        `,
        metadata: { source: 'account-opening-policy.pdf', page: 1 }
      }
    ];
    
    // Create documents
    const { Document } = require('langchain/document');
    const docs = policyDocuments.map(doc => 
      new Document({
        pageContent: doc.pageContent,
        metadata: doc.metadata
      })
    );
    
    // Create vector store
    await this.ragService.createVectorStore(docs);
    
    this.initialized = true;
    console.log('System initialized successfully');
  }
  
  /**
   * Handle customer question
   */
  async askQuestion(question) {
    if (!this.initialized) {
      await this.initialize();
    }
    
    const result = await this.ragService.answerQuestion(question);
    
    return {
      question,
      answer: result.answer,
      sources: result.sources,
      timestamp: new Date()
    };
  }
}

// Usage Example
async function example() {
  const qaSystem = new ENBDPolicyQASystem();
  
  // Question 1: Loan eligibility
  const q1 = await qaSystem.askQuestion(
    'What is the minimum salary required for a personal loan?'
  );
  console.log('Q:', q1.question);
  console.log('A:', q1.answer);
  console.log('Sources:', q1.sources.length);
  
  // Question 2: Credit card rewards
  const q2 = await qaSystem.askQuestion(
    'How many reward points do I earn on international purchases?'
  );
  console.log('Q:', q2.question);
  console.log('A:', q2.answer);
}

module.exports = {
  ENBDRAGService,
  ENBDPolicyQASystem
};
```

**RAG Pipeline:**

1. ✅ **Document Loading** - PDFs, text files
2. ✅ **Text Splitting** - Chunk documents
3. ✅ **Embedding** - Convert text to vectors
4. ✅ **Vector Store** - Store embeddings
5. ✅ **Retrieval** - Find relevant documents
6. ✅ **Generation** - LLM generates answer with context

---

### Q3. How do you implement conversational memory in Langchain?

**Answer:**

```javascript
const { ChatOpenAI } = require('@langchain/openai');
const { ConversationChain } = require('langchain/chains');
const { BufferMemory, BufferWindowMemory, ConversationSummaryMemory } = require('langchain/memory');
const { ChatPromptTemplate, MessagesPlaceholder } = require('@langchain/core/prompts');

/**
 * Conversational Memory Service
 */
class ConversationalMemoryService {
  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0.7,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }
  
  /**
   * Buffer Memory - Stores all messages
   */
  createBufferMemoryChain() {
    const memory = new BufferMemory({
      returnMessages: true,
      memoryKey: 'chat_history'
    });
    
    const chain = new ConversationChain({
      llm: this.model,
      memory: memory
    });
    
    return chain;
  }
  
  /**
   * Window Memory - Stores last K messages
   */
  createWindowMemoryChain(k = 5) {
    const memory = new BufferWindowMemory({
      k: k, // Keep last 5 interactions
      returnMessages: true,
      memoryKey: 'chat_history'
    });
    
    const chain = new ConversationChain({
      llm: this.model,
      memory: memory
    });
    
    return chain;
  }
  
  /**
   * Summary Memory - Summarizes old conversations
   */
  async createSummaryMemoryChain() {
    const memory = new ConversationSummaryMemory({
      llm: this.model,
      returnMessages: true,
      memoryKey: 'chat_history'
    });
    
    const chain = new ConversationChain({
      llm: this.model,
      memory: memory
    });
    
    return chain;
  }
}

/**
 * Practical Example: ENBD Banking Chatbot with Memory
 */
class ENBDBankingChatbot {
  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0.7,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    this.sessions = new Map(); // Store sessions by userId
  }
  
  /**
   * Create session for user
   */
  async createSession(userId) {
    const memory = new BufferWindowMemory({
      k: 10, // Keep last 10 messages
      returnMessages: true,
      memoryKey: 'chat_history'
    });
    
    const prompt = ChatPromptTemplate.fromMessages([
      ['system', `You are a helpful banking assistant for Emirates NBD. 
      
Your capabilities:
- Account balance inquiries
- Transaction history
- Credit card information
- Loan applications
- General banking questions

Always be professional, clear, and secure. Never ask for passwords or PINs.`],
      new MessagesPlaceholder('chat_history'),
      ['human', '{input}']
    ]);
    
    const chain = new ConversationChain({
      llm: this.model,
      memory: memory,
      prompt: prompt
    });
    
    this.sessions.set(userId, {
      chain,
      memory,
      createdAt: new Date(),
      lastActivity: new Date()
    });
    
    return chain;
  }
  
  /**
   * Get or create session
   */
  async getSession(userId) {
    if (!this.sessions.has(userId)) {
      await this.createSession(userId);
    }
    
    const session = this.sessions.get(userId);
    session.lastActivity = new Date();
    
    return session.chain;
  }
  
  /**
   * Send message
   */
  async sendMessage(userId, message) {
    const chain = await this.getSession(userId);
    
    const response = await chain.call({
      input: message
    });
    
    return {
      message: response.response,
      timestamp: new Date()
    };
  }
  
  /**
   * Get conversation history
   */
  async getHistory(userId) {
    const session = this.sessions.get(userId);
    
    if (!session) {
      return [];
    }
    
    const history = await session.memory.chatHistory.getMessages();
    
    return history.map(msg => ({
      type: msg._getType(),
      content: msg.content,
      timestamp: msg.additional_kwargs.timestamp
    }));
  }
  
  /**
   * Clear session
   */
  async clearSession(userId) {
    if (this.sessions.has(userId)) {
      const session = this.sessions.get(userId);
      await session.memory.clear();
      this.sessions.delete(userId);
    }
  }
  
  /**
   * Save conversation to database
   */
  async saveConversation(userId) {
    const history = await this.getHistory(userId);
    
    // Store in database
    await this.storeInDatabase(userId, history);
    
    return history;
  }
  
  async storeInDatabase(userId, history) {
    // Implementation: Save to MongoDB/PostgreSQL
    console.log(`Saving conversation for user ${userId}`);
  }
}

/**
 * Usage Example
 */
async function exampleConversation() {
  const chatbot = new ENBDBankingChatbot();
  const userId = 'customer123';
  
  // Message 1
  const res1 = await chatbot.sendMessage(userId, 
    'What are the requirements for opening a savings account?'
  );
  console.log('Bot:', res1.message);
  
  // Message 2 (with context)
  const res2 = await chatbot.sendMessage(userId,
    'How long does it take?'
  );
  console.log('Bot:', res2.message);
  
  // Message 3 (with context)
  const res3 = await chatbot.sendMessage(userId,
    'What documents do I need?'
  );
  console.log('Bot:', res3.message);
  
  // Get full history
  const history = await chatbot.getHistory(userId);
  console.log('Conversation:', history);
}

module.exports = {
  ConversationalMemoryService,
  ENBDBankingChatbot
};
```

**Memory Types:**

1. ✅ **Buffer Memory** - Stores all messages (good for short conversations)
2. ✅ **Window Memory** - Stores last K messages (good for medium conversations)
3. ✅ **Summary Memory** - Summarizes old messages (good for long conversations)
4. ✅ **Entity Memory** - Extracts and remembers entities

---

### Q4. How do you implement Langchain agents for autonomous task execution?

**Answer:**

```javascript
const { ChatOpenAI } = require('@langchain/openai');
const { initializeAgentExecutorWithOptions } = require('langchain/agents');
const { DynamicTool } = require('@langchain/core/tools');
const { Calculator } = require('langchain/tools/calculator');

/**
 * ENBD Banking Agent with Tools
 */
class ENBDBankingAgent {
  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    this.tools = this.createTools();
  }
  
  /**
   * Create custom tools
   */
  createTools() {
    // Tool 1: Check Account Balance
    const checkBalanceTool = new DynamicTool({
      name: 'check_account_balance',
      description: 'Check account balance for a customer. Input should be account number.',
      func: async (accountNumber) => {
        // Simulate database query
        const balance = await this.getAccountBalance(accountNumber);
        return `Account ${accountNumber} balance: AED ${balance.toLocaleString()}`;
      }
    });
    
    // Tool 2: Get Transaction History
    const transactionHistoryTool = new DynamicTool({
      name: 'get_transaction_history',
      description: 'Get recent transaction history. Input should be account number.',
      func: async (accountNumber) => {
        const transactions = await this.getTransactions(accountNumber);
        return JSON.stringify(transactions);
      }
    });
    
    // Tool 3: Calculate Loan EMI
    const loanCalculatorTool = new DynamicTool({
      name: 'calculate_loan_emi',
      description: 'Calculate monthly EMI for a loan. Input should be: principal,rate,tenure (e.g., "100000,3.5,60")',
      func: async (input) => {
        const [principal, rate, tenure] = input.split(',').map(Number);
        const monthlyRate = rate / 12 / 100;
        const emi = principal * monthlyRate * Math.pow(1 + monthlyRate, tenure) / 
                     (Math.pow(1 + monthlyRate, tenure) - 1);
        return `Monthly EMI: AED ${emi.toFixed(2)}`;
      }
    });
    
    // Tool 4: Check Credit Card Limit
    const creditLimitTool = new DynamicTool({
      name: 'check_credit_limit',
      description: 'Check available credit limit. Input should be card number.',
      func: async (cardNumber) => {
        const limit = await this.getCreditLimit(cardNumber);
        return `Available credit: AED ${limit.available.toLocaleString()} / Total: AED ${limit.total.toLocaleString()}`;
      }
    });
    
    // Tool 5: Search Bank Policies
    const policySearchTool = new DynamicTool({
      name: 'search_bank_policies',
      description: 'Search bank policies and procedures. Input should be search query.',
      func: async (query) => {
        const results = await this.searchPolicies(query);
        return results;
      }
    });
    
    return [
      checkBalanceTool,
      transactionHistoryTool,
      loanCalculatorTool,
      creditLimitTool,
      policySearchTool,
      new Calculator() // Built-in calculator
    ];
  }
  
  /**
   * Initialize agent
   */
  async createAgent() {
    const executor = await initializeAgentExecutorWithOptions(
      this.tools,
      this.model,
      {
        agentType: 'openai-functions',
        verbose: true,
        maxIterations: 5
      }
    );
    
    return executor;
  }
  
  /**
   * Execute task
   */
  async executeTask(input) {
    const agent = await this.createAgent();
    
    const result = await agent.call({
      input: input
    });
    
    return result.output;
  }
  
  // Mock database functions
  async getAccountBalance(accountNumber) {
    return 125000; // AED
  }
  
  async getTransactions(accountNumber) {
    return [
      { date: '2024-01-15', description: 'Salary Credit', amount: 15000, type: 'credit' },
      { date: '2024-01-16', description: 'Rent Payment', amount: -5000, type: 'debit' },
      { date: '2024-01-17', description: 'Grocery Shopping', amount: -350, type: 'debit' }
    ];
  }
  
  async getCreditLimit(cardNumber) {
    return {
      total: 50000,
      used: 12500,
      available: 37500
    };
  }
  
  async searchPolicies(query) {
    return 'Policy information retrieved from knowledge base...';
  }
}

/**
 * Practical Example: Multi-step Agent Task
 */
async function exampleAgentExecution() {
  const agent = new ENBDBankingAgent();
  
  // Complex query requiring multiple tool calls
  const query = `
I want to apply for a personal loan. 
My account number is ACC123456. 
First check my current balance, then calculate the EMI for a 
AED 100,000 loan at 3.5% interest for 60 months.
  `;
  
  console.log('Query:', query);
  
  const result = await agent.executeTask(query);
  
  console.log('Agent Result:', result);
  
  // Agent will:
  // 1. Use check_account_balance tool to check balance
  // 2. Use calculate_loan_emi tool to calculate EMI
  // 3. Synthesize results into coherent response
}

module.exports = {
  ENBDBankingAgent
};
```

**Agent Capabilities:**

1. ✅ **Tool Selection** - Agent chooses appropriate tools
2. ✅ **Multi-step Reasoning** - Breaks down complex tasks
3. ✅ **Custom Tools** - Create domain-specific tools
4. ✅ **Error Handling** - Retries and fallbacks

---

### Q5. How do you implement prompt engineering and templates in Langchain?

**Answer:**

```javascript
const { ChatOpenAI } = require('@langchain/openai');
const { PromptTemplate, ChatPromptTemplate, HumanMessagePromptTemplate, SystemMessagePromptTemplate } = require('@langchain/core/prompts');
const { FewShotPromptTemplate } = require('@langchain/core/prompts');

/**
 * Prompt Engineering Service for ENBD
 */
class PromptEngineeringService {
  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0.7,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }
  
  /**
   * Basic Prompt Template
   */
  createBasicPrompt() {
    const template = `You are a banking assistant for Emirates NBD.

Customer Query: {query}

Provide a helpful response:`;

    return PromptTemplate.fromTemplate(template);
  }
  
  /**
   * Chat Prompt Template (System + Human)
   */
  createChatPrompt() {
    const systemTemplate = `You are a senior banking advisor at Emirates NBD with 10 years of experience.

Your expertise includes:
- Personal banking products
- Corporate banking solutions
- Investment advisory
- Loan structuring

Guidelines:
- Be professional and courteous
- Provide accurate information
- Suggest relevant products when appropriate
- Always prioritize customer security`;

    const humanTemplate = `Customer Question: {question}`;

    const chatPrompt = ChatPromptTemplate.fromMessages([
      SystemMessagePromptTemplate.fromTemplate(systemTemplate),
      HumanMessagePromptTemplate.fromTemplate(humanTemplate)
    ]);
    
    return chatPrompt;
  }
  
  /**
   * Few-Shot Prompt (Learning from examples)
   */
  createFewShotPrompt() {
    // Examples for transaction categorization
    const examples = [
      {
        transaction: 'CARREFOUR HYPERMARKET - AED 450',
        category: 'Groceries'
      },
      {
        transaction: 'EMIRATES AIRLINE - AED 2500',
        category: 'Travel'
      },
      {
        transaction: 'DUBAI MALL - AED 800',
        category: 'Shopping'
      },
      {
        transaction: 'DEWA ELECTRICITY - AED 350',
        category: 'Utilities'
      },
      {
        transaction: 'UBER TRIP - AED 45',
        category: 'Transportation'
      }
    ];
    
    const exampleTemplate = `Transaction: {transaction}
Category: {category}`;

    const examplePrompt = new PromptTemplate({
      template: exampleTemplate,
      inputVariables: ['transaction', 'category']
    });
    
    const fewShotPrompt = new FewShotPromptTemplate({
      examples: examples,
      examplePrompt: examplePrompt,
      prefix: 'Categorize the following banking transactions based on these examples:',
      suffix: 'Transaction: {input}\nCategory:',
      inputVariables: ['input']
    });
    
    return fewShotPrompt;
  }
  
  /**
   * Dynamic Prompt with Conditional Logic
   */
  createDynamicPrompt(userProfile) {
    let template = `You are a banking assistant for Emirates NBD.

Customer Profile:
- Account Type: {accountType}
- Relationship Status: {relationshipStatus}`;

    // Add personalized recommendations
    if (userProfile.relationshipStatus === 'Premium') {
      template += `
- You have access to premium banking services
- Eligible for preferential rates`;
    }
    
    if (userProfile.accountBalance > 100000) {
      template += `
- Eligible for investment advisory services`;
    }
    
    template += `

Customer Query: {query}

Provide a personalized response:`;

    return PromptTemplate.fromTemplate(template);
  }
  
  /**
   * Output Parser Prompt
   */
  createStructuredOutputPrompt() {
    const template = `Extract customer information from the following text.

Text: {text}

Return a JSON object with these fields:
- name: Customer's full name
- account_number: Account number if mentioned
- request_type: Type of request (e.g., "balance_inquiry", "loan_application", "card_issue")
- urgency: Urgency level (low, medium, high)

JSON:`;

    return PromptTemplate.fromTemplate(template);
  }
}

/**
 * Practical Example: Loan Application Prompt System
 */
class LoanApplicationPromptSystem {
  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }
  
  /**
   * Loan eligibility assessment prompt
   */
  createEligibilityPrompt() {
    const template = `You are a loan officer at Emirates NBD. Assess loan eligibility.

Applicant Information:
- Name: {name}
- Age: {age}
- Monthly Income: AED {income}
- Employment Type: {employmentType}
- Years with Current Employer: {yearsEmployed}
- Existing Loans: {existingLoans}
- Credit Score: {creditScore}

Loan Request:
- Amount: AED {loanAmount}
- Purpose: {purpose}
- Tenure: {tenure} months

ENBD Loan Policies:
- Minimum age: 21 years
- Minimum monthly income: AED 5,000
- Minimum employment: 6 months with current employer
- Maximum loan amount: 20x monthly salary
- Credit score: Minimum 650

Provide:
1. Eligibility decision (Approved/Rejected/Further Review)
2. Reasoning
3. Recommended loan amount (if different)
4. Suggested interest rate based on credit profile
5. Next steps

Assessment:`;

    return PromptTemplate.fromTemplate(template);
  }
  
  /**
   * Risk assessment prompt
   */
  createRiskAssessmentPrompt() {
    const template = `Perform a risk assessment for this loan application.

Application Details:
{applicationDetails}

Financial Analysis:
- Debt-to-Income Ratio: {dti}%
- Loan-to-Value Ratio: {ltv}%
- Credit History: {creditHistory}

Assess the following risk factors:
1. Repayment Risk (1-10)
2. Default Probability (%)
3. Risk Category (Low/Medium/High)
4. Mitigation Measures
5. Recommended Actions

Risk Assessment:`;

    return PromptTemplate.fromTemplate(template);
  }
  
  /**
   * Generate response using prompt
   */
  async assessLoanEligibility(applicantData) {
    const prompt = this.createEligibilityPrompt();
    
    const formattedPrompt = await prompt.format(applicantData);
    
    const response = await this.model.call(formattedPrompt);
    
    return response.content;
  }
}

/**
 * Usage Example
 */
async function examplePromptEngineering() {
  const loanSystem = new LoanApplicationPromptSystem();
  
  const applicant = {
    name: 'Ahmed Ali',
    age: 35,
    income: 15000,
    employmentType: 'Salaried',
    yearsEmployed: 3,
    existingLoans: 'Personal Loan: AED 50,000',
    creditScore: 720,
    loanAmount: 200000,
    purpose: 'Home Renovation',
    tenure: 60
  };
  
  const assessment = await loanSystem.assessLoanEligibility(applicant);
  
  console.log('Loan Assessment:', assessment);
}

module.exports = {
  PromptEngineeringService,
  LoanApplicationPromptSystem
};
```

**Prompt Engineering Techniques:**

1. ✅ **Clear Instructions** - Specific role and guidelines
2. ✅ **Few-Shot Learning** - Provide examples
3. ✅ **Structured Output** - Request specific format (JSON)
4. ✅ **Context Injection** - Add relevant background
5. ✅ **Chain-of-Thought** - Ask for step-by-step reasoning

---

**Summary Q1-Q5:**
- Langchain basics (chains, prompts, memory) ✅
- RAG implementation (document retrieval + generation) ✅
- Conversational memory (buffer, window, summary) ✅
- Agents with tools (autonomous task execution) ✅
- Prompt engineering (templates, few-shot, structured output) ✅
