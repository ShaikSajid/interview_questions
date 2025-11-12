# GenAI Questions 6-10: Vector Databases & Embeddings

---

### Q6. How do you implement vector databases (Pinecone, Weaviate) with Langchain?

**Answer:**

Vector databases store embeddings for efficient similarity search, essential for RAG systems.

```javascript
const { OpenAIEmbeddings } = require('@langchain/openai');
const { PineconeStore } = require('@langchain/pinecone');
const { WeaviateStore } = require('@langchain/weaviate');
const { Pinecone } = require('@pinecone-database/pinecone');
const weaviate = require('weaviate-ts-client');

/**
 * Pinecone Vector Database Service
 */
class PineconeVectorService {
  constructor() {
    this.embeddings = new OpenAIEmbeddings({
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    this.pinecone = null;
    this.index = null;
  }
  
  /**
   * Initialize Pinecone
   */
  async initialize() {
    this.pinecone = new Pinecone({
      apiKey: process.env.PINECONE_API_KEY
    });
    
    // Get or create index
    this.index = this.pinecone.Index(process.env.PINECONE_INDEX);
  }
  
  /**
   * Create vector store from documents
   */
  async createVectorStore(documents) {
    await this.initialize();
    
    const vectorStore = await PineconeStore.fromDocuments(
      documents,
      this.embeddings,
      {
        pineconeIndex: this.index,
        namespace: 'enbd-banking-docs',
        textKey: 'text'
      }
    );
    
    return vectorStore;
  }
  
  /**
   * Add documents to existing store
   */
  async addDocuments(documents) {
    await this.initialize();
    
    const vectorStore = await PineconeStore.fromExistingIndex(
      this.embeddings,
      {
        pineconeIndex: this.index,
        namespace: 'enbd-banking-docs'
      }
    );
    
    await vectorStore.addDocuments(documents);
    
    return vectorStore;
  }
  
  /**
   * Similarity search
   */
  async similaritySearch(query, k = 4, filter = {}) {
    await this.initialize();
    
    const vectorStore = await PineconeStore.fromExistingIndex(
      this.embeddings,
      {
        pineconeIndex: this.index,
        namespace: 'enbd-banking-docs'
      }
    );
    
    const results = await vectorStore.similaritySearch(query, k, filter);
    
    return results;
  }
  
  /**
   * Similarity search with scores
   */
  async similaritySearchWithScore(query, k = 4) {
    await this.initialize();
    
    const vectorStore = await PineconeStore.fromExistingIndex(
      this.embeddings,
      {
        pineconeIndex: this.index,
        namespace: 'enbd-banking-docs'
      }
    );
    
    const results = await vectorStore.similaritySearchWithScore(query, k);
    
    return results.map(([doc, score]) => ({
      content: doc.pageContent,
      metadata: doc.metadata,
      score: score
    }));
  }
  
  /**
   * Delete vectors by metadata filter
   */
  async deleteByMetadata(filter) {
    await this.initialize();
    
    await this.index.deleteMany({
      filter: filter,
      namespace: 'enbd-banking-docs'
    });
  }
}

/**
 * Weaviate Vector Database Service
 */
class WeaviateVectorService {
  constructor() {
    this.embeddings = new OpenAIEmbeddings({
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    this.client = null;
  }
  
  /**
   * Initialize Weaviate client
   */
  async initialize() {
    this.client = weaviate.client({
      scheme: 'https',
      host: process.env.WEAVIATE_HOST,
      apiKey: new weaviate.ApiKey(process.env.WEAVIATE_API_KEY),
      headers: {
        'X-OpenAI-Api-Key': process.env.OPENAI_API_KEY
      }
    });
  }
  
  /**
   * Create schema
   */
  async createSchema() {
    await this.initialize();
    
    const schema = {
      class: 'ENBDBankingDocument',
      description: 'ENBD banking documents and policies',
      vectorizer: 'text2vec-openai',
      properties: [
        {
          name: 'content',
          dataType: ['text'],
          description: 'Document content'
        },
        {
          name: 'source',
          dataType: ['string'],
          description: 'Document source'
        },
        {
          name: 'category',
          dataType: ['string'],
          description: 'Document category'
        },
        {
          name: 'timestamp',
          dataType: ['date'],
          description: 'Document timestamp'
        }
      ]
    };
    
    await this.client.schema.classCreator().withClass(schema).do();
  }
  
  /**
   * Create vector store
   */
  async createVectorStore(documents) {
    await this.initialize();
    
    const vectorStore = await WeaviateStore.fromDocuments(
      documents,
      this.embeddings,
      {
        client: this.client,
        indexName: 'ENBDBankingDocument',
        textKey: 'content',
        metadataKeys: ['source', 'category', 'timestamp']
      }
    );
    
    return vectorStore;
  }
  
  /**
   * Hybrid search (vector + keyword)
   */
  async hybridSearch(query, k = 4) {
    await this.initialize();
    
    const result = await this.client.graphql
      .get()
      .withClassName('ENBDBankingDocument')
      .withFields('content source category _additional { score }')
      .withHybrid({
        query: query,
        alpha: 0.5 // 0.5 = equal weight to vector and keyword search
      })
      .withLimit(k)
      .do();
    
    return result.data.Get.ENBDBankingDocument;
  }
}

/**
 * Practical Example: ENBD Document Search System
 */
class ENBDDocumentSearchSystem {
  constructor(provider = 'pinecone') {
    this.provider = provider;
    this.vectorService = provider === 'pinecone' 
      ? new PineconeVectorService()
      : new WeaviateVectorService();
  }
  
  /**
   * Index banking documents
   */
  async indexDocuments() {
    const { Document } = require('langchain/document');
    
    const documents = [
      new Document({
        pageContent: `
ENBD Savings Account Benefits:
- Competitive interest rates up to 2.5% per annum
- No minimum balance requirement for basic savings
- Free online banking and mobile app
- Free debit card with international acceptance
- Free fund transfers within UAE
- 24/7 customer support
- Access to 200+ ATMs across UAE
- Monthly statements via email or post
- Overdraft facility available (subject to approval)
        `,
        metadata: {
          source: 'savings-account-benefits.pdf',
          category: 'products',
          type: 'savings_account',
          lastUpdated: '2024-01-15'
        }
      }),
      new Document({
        pageContent: `
ENBD Credit Card Rewards Program:
- Earn 1 point per AED 10 spent locally
- Earn 2 points per AED 10 spent internationally
- Bonus points on dining, travel, and fuel
- Redeem points for cashback, vouchers, or flights
- Points valid for 24 months from date of earning
- No annual fee on reward points redemption
- Special offers at partner merchants
- Instant rewards on selected categories
- Points accumulation across all ENBD cards
        `,
        metadata: {
          source: 'credit-card-rewards.pdf',
          category: 'products',
          type: 'credit_card',
          lastUpdated: '2024-01-10'
        }
      }),
      new Document({
        pageContent: `
ENBD Personal Loan Features:
- Loan amount: AED 10,000 to AED 500,000
- Tenure: 12 to 60 months
- Interest rate: Starting from 2.99% per annum
- Quick approval within 24 hours
- Minimal documentation required
- No hidden charges
- Flexible repayment options
- Top-up facility available
- Balance transfer option
- Salary transfer not mandatory
        `,
        metadata: {
          source: 'personal-loan-features.pdf',
          category: 'products',
          type: 'personal_loan',
          lastUpdated: '2024-01-20'
        }
      }),
      new Document({
        pageContent: `
ENBD Mobile Banking App Features:
- View account balances and transaction history
- Transfer funds instantly
- Pay bills and recharge mobile
- Apply for loans and credit cards
- Freeze/unfreeze debit and credit cards
- Update contact details
- Request cheque book
- Locate nearest branch or ATM
- Biometric login (fingerprint/face ID)
- Cardless cash withdrawal
- Split bills with friends
        `,
        metadata: {
          source: 'mobile-banking-features.pdf',
          category: 'digital_banking',
          type: 'mobile_app',
          lastUpdated: '2024-02-01'
        }
      }),
      new Document({
        pageContent: `
ENBD Investment Services:
- Mutual funds investment
- Portfolio management services
- Wealth management advisory
- Fixed deposits with competitive rates
- Equity trading platform
- Bonds and sukuk investments
- Gold investment accounts
- Real estate investment trusts (REITs)
- Retirement planning services
- Regular investment plans (SIP)
        `,
        metadata: {
          source: 'investment-services.pdf',
          category: 'wealth',
          type: 'investments',
          lastUpdated: '2024-01-25'
        }
      })
    ];
    
    console.log('Indexing documents...');
    await this.vectorService.createVectorStore(documents);
    console.log('Documents indexed successfully');
  }
  
  /**
   * Search documents
   */
  async search(query, k = 3) {
    console.log(`Searching for: "${query}"`);
    
    const results = await this.vectorService.similaritySearchWithScore(query, k);
    
    return results.map(result => ({
      content: result.content.substring(0, 200) + '...',
      source: result.metadata.source,
      category: result.metadata.category,
      relevanceScore: result.score.toFixed(4)
    }));
  }
  
  /**
   * Filtered search (e.g., only credit card docs)
   */
  async filteredSearch(query, category, k = 3) {
    const results = await this.vectorService.similaritySearch(
      query, 
      k,
      { category: category }
    );
    
    return results;
  }
}

/**
 * Usage Example
 */
async function exampleVectorSearch() {
  const searchSystem = new ENBDDocumentSearchSystem('pinecone');
  
  // Index documents (one-time)
  await searchSystem.indexDocuments();
  
  // Search queries
  const query1 = 'How can I earn rewards on my credit card?';
  const results1 = await searchSystem.search(query1);
  console.log('Results:', results1);
  
  const query2 = 'What are the interest rates for savings accounts?';
  const results2 = await searchSystem.search(query2);
  console.log('Results:', results2);
  
  const query3 = 'Can I check my balance on mobile?';
  const results3 = await searchSystem.search(query3);
  console.log('Results:', results3);
}

module.exports = {
  PineconeVectorService,
  WeaviateVectorService,
  ENBDDocumentSearchSystem
};
```

**Vector Database Features:**

1. ✅ **Pinecone** - Managed vector database, high performance
2. ✅ **Weaviate** - Open-source, hybrid search (vector + keyword)
3. ✅ **Metadata Filtering** - Filter by category, date, etc.
4. ✅ **Similarity Search** - Find semantically similar documents
5. ✅ **Scalability** - Handle millions of vectors

---

### Q7. How do you implement embeddings and semantic search?

**Answer:**

```javascript
const { OpenAIEmbeddings } = require('@langchain/openai');
const cosineSimilarity = require('compute-cosine-similarity');

/**
 * Embeddings Service
 */
class EmbeddingsService {
  constructor() {
    this.embeddings = new OpenAIEmbeddings({
      openAIApiKey: process.env.OPENAI_API_KEY,
      modelName: 'text-embedding-3-small' // 1536 dimensions
    });
  }
  
  /**
   * Generate embedding for single text
   */
  async generateEmbedding(text) {
    const embedding = await this.embeddings.embedQuery(text);
    return embedding;
  }
  
  /**
   * Generate embeddings for multiple documents
   */
  async generateEmbeddings(texts) {
    const embeddings = await this.embeddings.embedDocuments(texts);
    return embeddings;
  }
  
  /**
   * Calculate cosine similarity
   */
  calculateSimilarity(embedding1, embedding2) {
    return cosineSimilarity(embedding1, embedding2);
  }
  
  /**
   * Find most similar documents
   */
  async findSimilar(query, documents, topK = 3) {
    // Generate query embedding
    const queryEmbedding = await this.generateEmbedding(query);
    
    // Generate document embeddings
    const docTexts = documents.map(doc => doc.content);
    const docEmbeddings = await this.generateEmbeddings(docTexts);
    
    // Calculate similarities
    const similarities = docEmbeddings.map((docEmbedding, index) => ({
      document: documents[index],
      similarity: this.calculateSimilarity(queryEmbedding, docEmbedding)
    }));
    
    // Sort by similarity (descending)
    similarities.sort((a, b) => b.similarity - a.similarity);
    
    // Return top K
    return similarities.slice(0, topK);
  }
}

/**
 * Semantic Search Service for ENBD
 */
class ENBDSemanticSearchService {
  constructor() {
    this.embeddingsService = new EmbeddingsService();
    this.documentStore = [];
  }
  
  /**
   * Index documents
   */
  async indexDocuments(documents) {
    console.log('Generating embeddings...');
    
    const texts = documents.map(doc => doc.content);
    const embeddings = await this.embeddingsService.generateEmbeddings(texts);
    
    this.documentStore = documents.map((doc, index) => ({
      ...doc,
      embedding: embeddings[index]
    }));
    
    console.log(`Indexed ${this.documentStore.length} documents`);
  }
  
  /**
   * Semantic search
   */
  async search(query, topK = 5) {
    // Generate query embedding
    const queryEmbedding = await this.embeddingsService.generateEmbedding(query);
    
    // Calculate similarities
    const results = this.documentStore.map(doc => ({
      id: doc.id,
      title: doc.title,
      content: doc.content,
      metadata: doc.metadata,
      similarity: this.embeddingsService.calculateSimilarity(
        queryEmbedding,
        doc.embedding
      )
    }));
    
    // Sort by similarity
    results.sort((a, b) => b.similarity - a.similarity);
    
    // Return top K
    return results.slice(0, topK);
  }
  
  /**
   * Hybrid search (semantic + keyword)
   */
  async hybridSearch(query, topK = 5) {
    // Semantic search
    const semanticResults = await this.search(query, topK * 2);
    
    // Keyword search (simple)
    const keywordResults = this.keywordSearch(query, topK * 2);
    
    // Merge and re-rank
    const mergedResults = this.mergeResults(semanticResults, keywordResults);
    
    return mergedResults.slice(0, topK);
  }
  
  /**
   * Simple keyword search
   */
  keywordSearch(query, topK) {
    const queryTerms = query.toLowerCase().split(' ');
    
    const results = this.documentStore.map(doc => {
      const content = doc.content.toLowerCase();
      const matchCount = queryTerms.filter(term => content.includes(term)).length;
      
      return {
        ...doc,
        keywordScore: matchCount / queryTerms.length
      };
    });
    
    results.sort((a, b) => b.keywordScore - a.keywordScore);
    
    return results.slice(0, topK);
  }
  
  /**
   * Merge semantic and keyword results
   */
  mergeResults(semanticResults, keywordResults) {
    const merged = new Map();
    
    // Add semantic results
    semanticResults.forEach(result => {
      merged.set(result.id, {
        ...result,
        semanticScore: result.similarity,
        keywordScore: 0
      });
    });
    
    // Add keyword scores
    keywordResults.forEach(result => {
      if (merged.has(result.id)) {
        merged.get(result.id).keywordScore = result.keywordScore;
      } else {
        merged.set(result.id, {
          ...result,
          semanticScore: 0,
          keywordScore: result.keywordScore
        });
      }
    });
    
    // Calculate combined score (weighted)
    const results = Array.from(merged.values()).map(result => ({
      ...result,
      combinedScore: (result.semanticScore * 0.7) + (result.keywordScore * 0.3)
    }));
    
    // Sort by combined score
    results.sort((a, b) => b.combinedScore - a.combinedScore);
    
    return results;
  }
}

/**
 * Practical Example: Banking FAQ Search
 */
class ENBDFAQSearch {
  constructor() {
    this.searchService = new ENBDSemanticSearchService();
  }
  
  async initialize() {
    const faqs = [
      {
        id: 'FAQ001',
        title: 'How to open a savings account?',
        content: 'To open a savings account with ENBD, you need to provide your Emirates ID, passport copy, and salary certificate. Visit any branch or apply online through our website. Account activation typically takes 24 hours.',
        metadata: { category: 'accounts', popularity: 95 }
      },
      {
        id: 'FAQ002',
        title: 'What are credit card annual fees?',
        content: 'ENBD credit card annual fees vary by card type: Classic (AED 200), Platinum (AED 500), Signature (AED 1,000). Fees may be waived based on spending patterns.',
        metadata: { category: 'credit_cards', popularity: 88 }
      },
      {
        id: 'FAQ003',
        title: 'How to apply for personal loan?',
        content: 'Personal loan application is simple. Log in to online banking, select personal loan, enter desired amount and tenure. Approval within 24 hours. Minimum salary requirement is AED 5,000.',
        metadata: { category: 'loans', popularity: 92 }
      },
      {
        id: 'FAQ004',
        title: 'What is the minimum balance for savings account?',
        content: 'The minimum balance for ENBD savings account is AED 3,000. If balance falls below this amount, a monthly fee of AED 25 will be charged.',
        metadata: { category: 'accounts', popularity: 85 }
      },
      {
        id: 'FAQ005',
        title: 'How to activate mobile banking?',
        content: 'Download ENBD Mobile Banking app from App Store or Google Play. Register using your account number and OTP sent to registered mobile. Enable biometric login for quick access.',
        metadata: { category: 'digital_banking', popularity: 90 }
      },
      {
        id: 'FAQ006',
        title: 'What documents needed for account opening?',
        content: 'Required documents include original Emirates ID, passport with valid residence visa, salary certificate from employer, and proof of address (utility bill). Self-employed individuals need trade license.',
        metadata: { category: 'accounts', popularity: 87 }
      },
      {
        id: 'FAQ007',
        title: 'How to redeem credit card reward points?',
        content: 'Login to online banking, navigate to credit card section, select rewards. Choose redemption option: cashback, vouchers, or airline miles. Points valid for 24 months.',
        metadata: { category: 'credit_cards', popularity: 82 }
      },
      {
        id: 'FAQ008',
        title: 'What is the loan interest rate?',
        content: 'Personal loan interest rates start from 2.99% per annum. Actual rate depends on credit profile, salary, and loan amount. Processing fee is 1% of loan amount.',
        metadata: { category: 'loans', popularity: 94 }
      },
      {
        id: 'FAQ009',
        title: 'How to block lost credit card?',
        content: 'Immediately call 24/7 customer service at 600-540000. You can also block card through mobile app under card management section. Replacement card will be issued within 3-5 business days.',
        metadata: { category: 'credit_cards', popularity: 78 }
      },
      {
        id: 'FAQ010',
        title: 'Can I transfer funds internationally?',
        content: 'Yes, ENBD offers international fund transfer via SWIFT. Charges apply based on amount and destination country. Online transfer available through internet banking. Processing time: 2-3 business days.',
        metadata: { category: 'transfers', popularity: 80 }
      }
    ];
    
    await this.searchService.indexDocuments(faqs);
  }
  
  async search(query) {
    const results = await this.searchService.hybridSearch(query, 3);
    
    return results.map(result => ({
      question: result.title,
      answer: result.content,
      category: result.metadata.category,
      relevanceScore: result.combinedScore.toFixed(4)
    }));
  }
}

/**
 * Usage Example
 */
async function exampleSemanticSearch() {
  const faqSearch = new ENBDFAQSearch();
  
  await faqSearch.initialize();
  
  // Query 1: Semantic understanding
  console.log('\n=== Query 1 ===');
  const results1 = await faqSearch.search('I want to start saving money');
  console.log(results1);
  
  // Query 2: Different phrasing
  console.log('\n=== Query 2 ===');
  const results2 = await faqSearch.search('My card is missing');
  console.log(results2);
  
  // Query 3: Intent understanding
  console.log('\n=== Query 3 ===');
  const results3 = await faqSearch.search('I need money for home renovation');
  console.log(results3);
}

module.exports = {
  EmbeddingsService,
  ENBDSemanticSearchService,
  ENBDFAQSearch
};
```

**Embeddings Concepts:**

1. ✅ **Text Embeddings** - Convert text to vectors
2. ✅ **Semantic Similarity** - Understand meaning, not just keywords
3. ✅ **Cosine Similarity** - Measure similarity between vectors
4. ✅ **Hybrid Search** - Combine semantic + keyword search
5. ✅ **Dimensionality** - 1536 dimensions for OpenAI embeddings

---

### Q8. How do you implement document loaders and text splitters in Langchain?

**Answer:**

```javascript
const { PDFLoader } = require('langchain/document_loaders/fs/pdf');
const { TextLoader } = require('langchain/document_loaders/fs/text');
const { CSVLoader } = require('langchain/document_loaders/fs/csv');
const { DocxLoader } = require('langchain/document_loaders/fs/docx');
const { RecursiveCharacterTextSplitter, TokenTextSplitter, CharacterTextSplitter } = require('langchain/text_splitter');

/**
 * Document Loading Service
 */
class DocumentLoadingService {
  /**
   * Load PDF documents
   */
  async loadPDF(filePath) {
    const loader = new PDFLoader(filePath, {
      splitPages: true
    });
    
    const docs = await loader.load();
    
    return docs;
  }
  
  /**
   * Load text files
   */
  async loadText(filePath) {
    const loader = new TextLoader(filePath);
    const docs = await loader.load();
    
    return docs;
  }
  
  /**
   * Load CSV files
   */
  async loadCSV(filePath, column) {
    const loader = new CSVLoader(filePath, column);
    const docs = await loader.load();
    
    return docs;
  }
  
  /**
   * Load DOCX files
   */
  async loadDOCX(filePath) {
    const loader = new DocxLoader(filePath);
    const docs = await loader.load();
    
    return docs;
  }
  
  /**
   * Load from URL
   */
  async loadFromURL(url) {
    const { CheerioWebBaseLoader } = require('langchain/document_loaders/web/cheerio');
    
    const loader = new CheerioWebBaseLoader(url);
    const docs = await loader.load();
    
    return docs;
  }
  
  /**
   * Load from API response
   */
  async loadFromAPI(endpoint) {
    const axios = require('axios');
    const { Document } = require('langchain/document');
    
    const response = await axios.get(endpoint);
    const data = response.data;
    
    const docs = data.map(item => new Document({
      pageContent: item.content,
      metadata: item.metadata
    }));
    
    return docs;
  }
}

/**
 * Text Splitting Service
 */
class TextSplittingService {
  /**
   * Recursive Character Splitter (Recommended)
   * Tries to split on paragraphs, then sentences, then words
   */
  createRecursiveSplitter(chunkSize = 1000, chunkOverlap = 200) {
    return new RecursiveCharacterTextSplitter({
      chunkSize: chunkSize,
      chunkOverlap: chunkOverlap,
      separators: ['\n\n', '\n', '. ', ' ', '']
    });
  }
  
  /**
   * Token-based Splitter
   * Splits based on token count (important for LLM context limits)
   */
  createTokenSplitter(chunkSize = 500, chunkOverlap = 50) {
    return new TokenTextSplitter({
      chunkSize: chunkSize,
      chunkOverlap: chunkOverlap
    });
  }
  
  /**
   * Character Splitter
   * Simple splitting by character count
   */
  createCharacterSplitter(separator = '\n\n', chunkSize = 1000) {
    return new CharacterTextSplitter({
      separator: separator,
      chunkSize: chunkSize,
      chunkOverlap: 200
    });
  }
  
  /**
   * Split documents
   */
  async splitDocuments(documents, splitter) {
    const splitDocs = await splitter.splitDocuments(documents);
    return splitDocs;
  }
  
  /**
   * Split text
   */
  async splitText(text, splitter) {
    const chunks = await splitter.splitText(text);
    return chunks;
  }
}

/**
 * Practical Example: ENBD Document Processing Pipeline
 */
class ENBDDocumentProcessingPipeline {
  constructor() {
    this.documentLoader = new DocumentLoadingService();
    this.textSplitter = new TextSplittingService();
  }
  
  /**
   * Process banking policy documents
   */
  async processPolicyDocuments(filePaths) {
    const allDocuments = [];
    
    // Load all documents
    for (const filePath of filePaths) {
      console.log(`Loading: ${filePath}`);
      
      const ext = filePath.split('.').pop().toLowerCase();
      
      let docs;
      switch (ext) {
        case 'pdf':
          docs = await this.documentLoader.loadPDF(filePath);
          break;
        case 'txt':
          docs = await this.documentLoader.loadText(filePath);
          break;
        case 'docx':
          docs = await this.documentLoader.loadDOCX(filePath);
          break;
        default:
          console.warn(`Unsupported file type: ${ext}`);
          continue;
      }
      
      allDocuments.push(...docs);
    }
    
    console.log(`Loaded ${allDocuments.length} documents`);
    
    // Split documents
    const splitter = this.textSplitter.createRecursiveSplitter(1000, 200);
    const splitDocs = await this.textSplitter.splitDocuments(allDocuments, splitter);
    
    console.log(`Split into ${splitDocs.length} chunks`);
    
    return splitDocs;
  }
  
  /**
   * Process transaction data from CSV
   */
  async processTransactionData(csvFilePath) {
    const docs = await this.documentLoader.loadCSV(csvFilePath, 'description');
    
    // Add custom metadata
    const enrichedDocs = docs.map(doc => ({
      ...doc,
      metadata: {
        ...doc.metadata,
        type: 'transaction',
        processedAt: new Date()
      }
    }));
    
    return enrichedDocs;
  }
  
  /**
   * Process customer support tickets
   */
  async processSupportTickets(tickets) {
    const { Document } = require('langchain/document');
    
    const docs = tickets.map(ticket => new Document({
      pageContent: `
Ticket ID: ${ticket.id}
Customer: ${ticket.customerId}
Issue: ${ticket.issue}
Description: ${ticket.description}
Resolution: ${ticket.resolution}
      `,
      metadata: {
        ticketId: ticket.id,
        category: ticket.category,
        status: ticket.status,
        priority: ticket.priority,
        createdAt: ticket.createdAt
      }
    }));
    
    return docs;
  }
  
  /**
   * Smart chunking based on document type
   */
  async smartChunking(documents, documentType) {
    let splitter;
    
    switch (documentType) {
      case 'policy':
        // Larger chunks for policy documents
        splitter = this.textSplitter.createRecursiveSplitter(1500, 300);
        break;
      
      case 'faq':
        // Smaller chunks for FAQs
        splitter = this.textSplitter.createRecursiveSplitter(500, 100);
        break;
      
      case 'terms':
        // Medium chunks for terms & conditions
        splitter = this.textSplitter.createRecursiveSplitter(1000, 200);
        break;
      
      default:
        splitter = this.textSplitter.createRecursiveSplitter(1000, 200);
    }
    
    return await this.textSplitter.splitDocuments(documents, splitter);
  }
  
  /**
   * Add metadata to chunks
   */
  enrichChunksWithMetadata(chunks, additionalMetadata) {
    return chunks.map((chunk, index) => ({
      ...chunk,
      metadata: {
        ...chunk.metadata,
        ...additionalMetadata,
        chunkIndex: index,
        totalChunks: chunks.length
      }
    }));
  }
}

/**
 * Usage Example
 */
async function exampleDocumentProcessing() {
  const pipeline = new ENBDDocumentProcessingPipeline();
  
  // Process policy documents
  const policyFiles = [
    'policies/personal-loan-policy.pdf',
    'policies/credit-card-policy.pdf',
    'policies/account-opening-policy.pdf'
  ];
  
  const chunks = await pipeline.processPolicyDocuments(policyFiles);
  
  console.log('Sample chunk:');
  console.log(chunks[0]);
  
  // Smart chunking for different document types
  const faqDocs = [/* FAQ documents */];
  const faqChunks = await pipeline.smartChunking(faqDocs, 'faq');
  
  console.log(`FAQ chunks: ${faqChunks.length}`);
}

module.exports = {
  DocumentLoadingService,
  TextSplittingService,
  ENBDDocumentProcessingPipeline
};
```

**Document Processing:**

1. ✅ **Multiple Loaders** - PDF, DOCX, CSV, TXT, Web
2. ✅ **Smart Splitting** - Recursive, token-based, character-based
3. ✅ **Chunk Overlap** - Maintain context between chunks
4. ✅ **Metadata Enrichment** - Add custom metadata
5. ✅ **Type-specific Processing** - Different strategies for different docs

---

### Q9. How do you implement LangGraph for complex workflows?

**Answer:**

```javascript
const { ChatOpenAI } = require('@langchain/openai');
const { StateGraph, END } = require('@langchain/langgraph');

/**
 * LangGraph for Multi-Step Workflows
 */
class LangGraphService {
  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }
  
  /**
   * Create state graph
   */
  createStateGraph(nodes, edges, entryPoint) {
    const graph = new StateGraph({
      channels: {
        messages: {
          value: (x, y) => x.concat(y),
          default: () => []
        }
      }
    });
    
    // Add nodes
    nodes.forEach(node => {
      graph.addNode(node.name, node.func);
    });
    
    // Add edges
    edges.forEach(edge => {
      if (edge.condition) {
        graph.addConditionalEdges(edge.from, edge.condition, edge.mapping);
      } else {
        graph.addEdge(edge.from, edge.to);
      }
    });
    
    // Set entry point
    graph.setEntryPoint(entryPoint);
    
    return graph.compile();
  }
}

/**
 * Practical Example: ENBD Loan Application Workflow
 */
class ENBDLoanApplicationWorkflow {
  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }
  
  /**
   * Build loan application graph
   */
  buildWorkflow() {
    const graph = new StateGraph({
      channels: {
        applicantData: {},
        eligibilityResult: {},
        riskAssessment: {},
        approval: {},
        status: 'pending'
      }
    });
    
    // Node 1: Validate Application
    graph.addNode('validate', async (state) => {
      console.log('Step 1: Validating application...');
      
      const validation = this.validateApplication(state.applicantData);
      
      return {
        ...state,
        validation: validation
      };
    });
    
    // Node 2: Check Eligibility
    graph.addNode('check_eligibility', async (state) => {
      console.log('Step 2: Checking eligibility...');
      
      const eligibility = await this.checkEligibility(state.applicantData);
      
      return {
        ...state,
        eligibilityResult: eligibility
      };
    });
    
    // Node 3: Risk Assessment
    graph.addNode('risk_assessment', async (state) => {
      console.log('Step 3: Performing risk assessment...');
      
      const risk = await this.assessRisk(state.applicantData, state.eligibilityResult);
      
      return {
        ...state,
        riskAssessment: risk
      };
    });
    
    // Node 4: Manual Review (if needed)
    graph.addNode('manual_review', async (state) => {
      console.log('Step 4: Flagged for manual review...');
      
      return {
        ...state,
        status: 'manual_review',
        approval: { decision: 'pending', reason: 'Requires manual review' }
      };
    });
    
    // Node 5: Auto Approve
    graph.addNode('auto_approve', async (state) => {
      console.log('Step 5: Auto-approving...');
      
      return {
        ...state,
        status: 'approved',
        approval: {
          decision: 'approved',
          loanAmount: state.applicantData.requestedAmount,
          interestRate: state.riskAssessment.recommendedRate,
          tenure: state.applicantData.tenure
        }
      };
    });
    
    // Node 6: Reject
    graph.addNode('reject', async (state) => {
      console.log('Step 6: Rejecting application...');
      
      return {
        ...state,
        status: 'rejected',
        approval: {
          decision: 'rejected',
          reason: state.eligibilityResult.reason || state.riskAssessment.reason
        }
      };
    });
    
    // Define edges
    graph.setEntryPoint('validate');
    
    // Conditional: After validation
    graph.addConditionalEdges(
      'validate',
      (state) => {
        return state.validation.valid ? 'check_eligibility' : 'reject';
      },
      {
        check_eligibility: 'check_eligibility',
        reject: 'reject'
      }
    );
    
    // Conditional: After eligibility check
    graph.addConditionalEdges(
      'check_eligibility',
      (state) => {
        return state.eligibilityResult.eligible ? 'risk_assessment' : 'reject';
      },
      {
        risk_assessment: 'risk_assessment',
        reject: 'reject'
      }
    );
    
    // Conditional: After risk assessment
    graph.addConditionalEdges(
      'risk_assessment',
      (state) => {
        const risk = state.riskAssessment.level;
        if (risk === 'low') return 'auto_approve';
        if (risk === 'medium') return 'manual_review';
        return 'reject';
      },
      {
        auto_approve: 'auto_approve',
        manual_review: 'manual_review',
        reject: 'reject'
      }
    );
    
    // End nodes
    graph.addEdge('auto_approve', END);
    graph.addEdge('manual_review', END);
    graph.addEdge('reject', END);
    
    return graph.compile();
  }
  
  /**
   * Process loan application
   */
  async processApplication(applicantData) {
    const workflow = this.buildWorkflow();
    
    const result = await workflow.invoke({
      applicantData: applicantData
    });
    
    return result;
  }
  
  /**
   * Validate application data
   */
  validateApplication(data) {
    const errors = [];
    
    if (!data.name) errors.push('Name is required');
    if (!data.age || data.age < 21) errors.push('Minimum age is 21');
    if (!data.salary || data.salary < 5000) errors.push('Minimum salary is AED 5,000');
    if (!data.requestedAmount) errors.push('Loan amount is required');
    
    return {
      valid: errors.length === 0,
      errors: errors
    };
  }
  
  /**
   * Check eligibility
   */
  async checkEligibility(data) {
    // Eligibility criteria
    const eligible = 
      data.age >= 21 &&
      data.salary >= 5000 &&
      data.yearsEmployed >= 0.5 &&
      data.requestedAmount <= (data.salary * 20);
    
    return {
      eligible: eligible,
      reason: eligible ? 'Meets all criteria' : 'Does not meet eligibility criteria'
    };
  }
  
  /**
   * Assess risk
   */
  async assessRisk(applicantData, eligibilityResult) {
    // Simple risk scoring
    let riskScore = 0;
    
    // Credit score factor
    if (applicantData.creditScore >= 750) riskScore += 0;
    else if (applicantData.creditScore >= 650) riskScore += 20;
    else riskScore += 50;
    
    // Debt-to-income ratio
    const dti = (applicantData.existingDebt / applicantData.salary) * 100;
    if (dti < 30) riskScore += 0;
    else if (dti < 50) riskScore += 20;
    else riskScore += 40;
    
    // Employment stability
    if (applicantData.yearsEmployed >= 5) riskScore += 0;
    else if (applicantData.yearsEmployed >= 2) riskScore += 10;
    else riskScore += 20;
    
    // Determine risk level
    let level, rate;
    if (riskScore < 20) {
      level = 'low';
      rate = 2.99;
    } else if (riskScore < 50) {
      level = 'medium';
      rate = 4.99;
    } else {
      level = 'high';
      rate = 7.99;
    }
    
    return {
      level: level,
      score: riskScore,
      recommendedRate: rate,
      reason: `Risk score: ${riskScore}`
    };
  }
}

/**
 * Usage Example
 */
async function exampleLoanWorkflow() {
  const workflow = new ENBDLoanApplicationWorkflow();
  
  // Applicant 1: Low risk - Auto approve
  const applicant1 = {
    name: 'Ahmed Ali',
    age: 35,
    salary: 20000,
    yearsEmployed: 5,
    requestedAmount: 150000,
    tenure: 60,
    creditScore: 780,
    existingDebt: 3000
  };
  
  console.log('\n=== Applicant 1 ===');
  const result1 = await workflow.processApplication(applicant1);
  console.log('Result:', result1.approval);
  
  // Applicant 2: High risk - Reject
  const applicant2 = {
    name: 'Sara Mohammed',
    age: 25,
    salary: 6000,
    yearsEmployed: 0.3,
    requestedAmount: 100000,
    tenure: 36,
    creditScore: 580,
    existingDebt: 4000
  };
  
  console.log('\n=== Applicant 2 ===');
  const result2 = await workflow.processApplication(applicant2);
  console.log('Result:', result2.approval);
}

module.exports = {
  LangGraphService,
  ENBDLoanApplicationWorkflow
};
```

**LangGraph Features:**

1. ✅ **State Management** - Maintain state across nodes
2. ✅ **Conditional Routing** - Dynamic workflow paths
3. ✅ **Multi-Step Processing** - Complex business logic
4. ✅ **Error Handling** - Graceful failure handling
5. ✅ **Async Execution** - Handle async operations

---

### Q10. How do you implement streaming responses and real-time chat?

**Answer:**

```javascript
const { ChatOpenAI } = require('@langchain/openai');
const { HumanMessage } = require('@langchain/core/messages');

/**
 * Streaming Service
 */
class StreamingService {
  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0.7,
      streaming: true,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }
  
  /**
   * Stream response (callback-based)
   */
  async streamResponse(prompt, onToken) {
    await this.model.call(
      [new HumanMessage(prompt)],
      {
        callbacks: [
          {
            handleLLMNewToken(token) {
              onToken(token);
            }
          }
        ]
      }
    );
  }
  
  /**
   * Stream response (async generator)
   */
  async *streamResponseGenerator(prompt) {
    const stream = await this.model.stream([new HumanMessage(prompt)]);
    
    for await (const chunk of stream) {
      yield chunk.content;
    }
  }
}

/**
 * Practical Example: Real-time Chat with Streaming
 */
class ENBDRealtimeChatService {
  constructor() {
    this.streamingService = new StreamingService();
  }
  
  /**
   * Express endpoint for streaming chat (SSE)
   */
  createStreamingEndpoint() {
    return async (req, res) => {
      const { message } = req.body;
      
      // Set headers for Server-Sent Events
      res.setHeader('Content-Type', 'text/event-stream');
      res.setHeader('Cache-Control', 'no-cache');
      res.setHeader('Connection', 'keep-alive');
      
      try {
        // Stream response
        await this.streamingService.streamResponse(message, (token) => {
          res.write(`data: ${JSON.stringify({ token })}\n\n`);
        });
        
        res.write('data: [DONE]\n\n');
        res.end();
      } catch (error) {
        res.write(`data: ${JSON.stringify({ error: error.message })}\n\n`);
        res.end();
      }
    };
  }
  
  /**
   * WebSocket-based streaming
   */
  setupWebSocketStreaming(wss) {
    wss.on('connection', (ws) => {
      console.log('Client connected');
      
      ws.on('message', async (data) => {
        const { message } = JSON.parse(data);
        
        try {
          await this.streamingService.streamResponse(message, (token) => {
            ws.send(JSON.stringify({ type: 'token', content: token }));
          });
          
          ws.send(JSON.stringify({ type: 'done' }));
        } catch (error) {
          ws.send(JSON.stringify({ type: 'error', message: error.message }));
        }
      });
      
      ws.on('close', () => {
        console.log('Client disconnected');
      });
    });
  }
}

/**
 * Client-side example (React)
 */
const clientExample = `
import React, { useState } from 'react';

function ChatComponent() {
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState('');
  const [streaming, setStreaming] = useState(false);
  
  const sendMessage = async () => {
    if (!input.trim()) return;
    
    // Add user message
    const userMessage = { role: 'user', content: input };
    setMessages(prev => [...prev, userMessage]);
    setInput('');
    setStreaming(true);
    
    // Prepare assistant message
    let assistantMessage = { role: 'assistant', content: '' };
    setMessages(prev => [...prev, assistantMessage]);
    
    try {
      const response = await fetch('/api/chat/stream', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ message: input })
      });
      
      const reader = response.body.getReader();
      const decoder = new TextDecoder();
      
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        
        const chunk = decoder.decode(value);
        const lines = chunk.split('\\n');
        
        for (const line of lines) {
          if (line.startsWith('data: ')) {
            const data = line.slice(6);
            if (data === '[DONE]') {
              setStreaming(false);
              break;
            }
            
            try {
              const parsed = JSON.parse(data);
              if (parsed.token) {
                // Update assistant message
                setMessages(prev => {
                  const updated = [...prev];
                  updated[updated.length - 1].content += parsed.token;
                  return updated;
                });
              }
            } catch (e) {}
          }
        }
      }
    } catch (error) {
      console.error('Streaming error:', error);
      setStreaming(false);
    }
  };
  
  return (
    <div className="chat-container">
      <div className="messages">
        {messages.map((msg, idx) => (
          <div key={idx} className={\`message \${msg.role}\`}>
            {msg.content}
          </div>
        ))}
      </div>
      
      <div className="input-area">
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyPress={(e) => e.key === 'Enter' && sendMessage()}
          disabled={streaming}
          placeholder="Ask me anything about ENBD banking..."
        />
        <button onClick={sendMessage} disabled={streaming}>
          {streaming ? 'Sending...' : 'Send'}
        </button>
      </div>
    </div>
  );
}

export default ChatComponent;
`;

/**
 * Express server setup
 */
const serverSetup = `
const express = require('express');
const cors = require('cors');
const { ENBDRealtimeChatService } = require('./streaming-service');

const app = express();
app.use(cors());
app.use(express.json());

const chatService = new ENBDRealtimeChatService();

// Streaming endpoint
app.post('/api/chat/stream', chatService.createStreamingEndpoint());

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
`;

module.exports = {
  StreamingService,
  ENBDRealtimeChatService,
  clientExample,
  serverSetup
};
```

**Streaming Features:**

1. ✅ **Token-by-Token Streaming** - Real-time response
2. ✅ **Server-Sent Events (SSE)** - HTTP-based streaming
3. ✅ **WebSocket Support** - Bidirectional communication
4. ✅ **Progress Indicators** - Show typing indicators
5. ✅ **Error Handling** - Graceful failure handling

---

**Summary Q6-Q10:**
- Vector databases (Pinecone, Weaviate) for semantic search ✅
- Embeddings and similarity search ✅
- Document loaders and smart text splitting ✅
- LangGraph for complex multi-step workflows ✅
- Streaming responses for real-time chat ✅

Continuing with GenAI Q11-Q15...
