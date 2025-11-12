# GenAI Questions 21-25: Multi-modal AI & Document Processing

---

### Q21. How do you implement image generation and analysis with AI models?

**Answer:**

Multi-modal AI enables working with images, text, audio, and video. For banking, this includes document processing, signature verification, and visual analytics.

```javascript
const OpenAI = require('openai');
const Anthropic = require('@anthropic-ai/sdk');
const fs = require('fs').promises;
const axios = require('axios');

/**
 * Image Generation and Analysis Service
 */
class MultiModalImageService {
  constructor() {
    this.openai = new OpenAI({
      apiKey: process.env.OPENAI_API_KEY
    });
    
    this.anthropic = new Anthropic({
      apiKey: process.env.ANTHROPIC_API_KEY
    });
  }
  
  /**
   * Generate image with DALL-E
   */
  async generateImage(prompt, options = {}) {
    const response = await this.openai.images.generate({
      model: options.model || 'dall-e-3',
      prompt: prompt,
      n: options.n || 1,
      size: options.size || '1024x1024',
      quality: options.quality || 'standard',
      style: options.style || 'natural'
    });
    
    return {
      images: response.data.map(img => ({
        url: img.url,
        revisedPrompt: img.revised_prompt
      }))
    };
  }
  
  /**
   * Analyze image with GPT-4 Vision
   */
  async analyzeImage(imageUrl, prompt) {
    const response = await this.openai.chat.completions.create({
      model: 'gpt-4-vision-preview',
      messages: [
        {
          role: 'user',
          content: [
            { type: 'text', text: prompt },
            {
              type: 'image_url',
              image_url: {
                url: imageUrl,
                detail: 'high'
              }
            }
          ]
        }
      ],
      max_tokens: 1000
    });
    
    return {
      analysis: response.choices[0].message.content,
      usage: response.usage
    };
  }
  
  /**
   * Analyze image with Claude (alternative)
   */
  async analyzeImageWithClaude(imageBase64, mimeType, prompt) {
    const response = await this.anthropic.messages.create({
      model: 'claude-3-opus-20240229',
      max_tokens: 1024,
      messages: [
        {
          role: 'user',
          content: [
            {
              type: 'image',
              source: {
                type: 'base64',
                media_type: mimeType,
                data: imageBase64
              }
            },
            {
              type: 'text',
              text: prompt
            }
          ]
        }
      ]
    });
    
    return {
      analysis: response.content[0].text,
      usage: response.usage
    };
  }
  
  /**
   * Extract text from image (OCR)
   */
  async extractTextFromImage(imageUrl) {
    const analysis = await this.analyzeImage(
      imageUrl,
      'Extract all text from this image. Preserve the layout and formatting.'
    );
    
    return {
      text: analysis.analysis,
      usage: analysis.usage
    };
  }
  
  /**
   * Compare two images
   */
  async compareImages(imageUrl1, imageUrl2, comparisonCriteria) {
    const response = await this.openai.chat.completions.create({
      model: 'gpt-4-vision-preview',
      messages: [
        {
          role: 'user',
          content: [
            { type: 'text', text: `Compare these two images based on: ${comparisonCriteria}` },
            { type: 'image_url', image_url: { url: imageUrl1 } },
            { type: 'image_url', image_url: { url: imageUrl2 } }
          ]
        }
      ],
      max_tokens: 1000
    });
    
    return {
      comparison: response.choices[0].message.content,
      usage: response.usage
    };
  }
  
  /**
   * Convert image to base64
   */
  async imageToBase64(imagePath) {
    const imageBuffer = await fs.readFile(imagePath);
    return imageBuffer.toString('base64');
  }
  
  /**
   * Download image from URL
   */
  async downloadImage(url, outputPath) {
    const response = await axios.get(url, { responseType: 'arraybuffer' });
    await fs.writeFile(outputPath, response.data);
    return outputPath;
  }
}

/**
 * ENBD Document Processing Service
 */
class ENBDDocumentProcessingService {
  constructor() {
    this.imageService = new MultiModalImageService();
  }
  
  /**
   * Process Emirates ID
   */
  async processEmiratesID(frontImageUrl, backImageUrl) {
    console.log('Processing Emirates ID...');
    
    // Analyze front side
    const frontAnalysis = await this.imageService.analyzeImage(
      frontImageUrl,
      `Analyze this Emirates ID (front side) and extract:
      - ID Number
      - Full Name (English)
      - Nationality
      - Date of Birth
      - Gender
      - Expiry Date
      
      Return as JSON.`
    );
    
    // Analyze back side
    const backAnalysis = await this.imageService.analyzeImage(
      backImageUrl,
      `Analyze this Emirates ID (back side) and extract:
      - Address
      - MRZ (Machine Readable Zone) data
      
      Return as JSON.`
    );
    
    try {
      const frontData = JSON.parse(frontAnalysis.analysis);
      const backData = JSON.parse(backAnalysis.analysis);
      
      return {
        success: true,
        data: {
          ...frontData,
          ...backData
        },
        verification: await this.verifyIDAuthenticity(frontImageUrl)
      };
    } catch (error) {
      return {
        success: false,
        error: 'Failed to parse Emirates ID data',
        rawData: {
          front: frontAnalysis.analysis,
          back: backAnalysis.analysis
        }
      };
    }
  }
  
  /**
   * Verify ID authenticity
   */
  async verifyIDAuthenticity(imageUrl) {
    const analysis = await this.imageService.analyzeImage(
      imageUrl,
      `Analyze this Emirates ID for authenticity. Check for:
      - Hologram presence
      - Print quality
      - Security features
      - Signs of tampering
      - Overall document quality
      
      Rate authenticity from 0-100 and provide reasoning.`
    );
    
    return {
      analysis: analysis.analysis,
      confidence: this.extractConfidenceScore(analysis.analysis)
    };
  }
  
  /**
   * Process bank check
   */
  async processCheck(checkImageUrl) {
    console.log('Processing bank check...');
    
    const analysis = await this.imageService.analyzeImage(
      checkImageUrl,
      `Analyze this bank check and extract:
      - Check number
      - Date
      - Payee name
      - Amount (in numbers)
      - Amount (in words)
      - Bank name
      - Account number
      - Routing number
      - Signature present (yes/no)
      
      Return as JSON.`
    );
    
    try {
      const checkData = JSON.parse(analysis.analysis);
      
      return {
        success: true,
        data: checkData,
        validation: await this.validateCheck(checkData)
      };
    } catch (error) {
      return {
        success: false,
        error: 'Failed to parse check data',
        rawData: analysis.analysis
      };
    }
  }
  
  /**
   * Validate check data
   */
  async validateCheck(checkData) {
    const issues = [];
    
    // Validate amount consistency
    if (checkData.amountNumbers && checkData.amountWords) {
      // Would compare amounts here
      issues.push({
        type: 'warning',
        message: 'Verify amount in words matches amount in numbers'
      });
    }
    
    // Check signature presence
    if (checkData.signaturePresent === 'no') {
      issues.push({
        type: 'error',
        message: 'Check is not signed'
      });
    }
    
    return {
      valid: issues.filter(i => i.type === 'error').length === 0,
      issues: issues
    };
  }
  
  /**
   * Verify signature
   */
  async verifySignature(documentSignatureUrl, referenceSignatureUrl) {
    console.log('Verifying signature...');
    
    const comparison = await this.imageService.compareImages(
      documentSignatureUrl,
      referenceSignatureUrl,
      `Compare these two signatures for:
      - Overall shape and style
      - Pen pressure patterns
      - Letter formations
      - Stroke order
      - Overall similarity
      
      Provide a similarity score (0-100) and detailed analysis.`
    );
    
    const score = this.extractConfidenceScore(comparison.comparison);
    
    return {
      match: score >= 80,
      similarity: score,
      analysis: comparison.comparison
    };
  }
  
  /**
   * Process salary certificate
   */
  async processSalaryCertificate(imageUrl) {
    console.log('Processing salary certificate...');
    
    const analysis = await this.imageService.analyzeImage(
      imageUrl,
      `Analyze this salary certificate and extract:
      - Employee name
      - Employee ID
      - Company name
      - Designation/Position
      - Monthly salary
      - Issue date
      - Valid until date
      - Company stamp present (yes/no)
      - Signature present (yes/no)
      
      Return as JSON.`
    );
    
    try {
      const data = JSON.parse(analysis.analysis);
      
      return {
        success: true,
        data: data,
        verification: {
          hasCompanyStamp: data.companyStampPresent === 'yes',
          hasSignature: data.signaturePresent === 'yes',
          isValid: data.companyStampPresent === 'yes' && data.signaturePresent === 'yes'
        }
      };
    } catch (error) {
      return {
        success: false,
        error: 'Failed to parse salary certificate',
        rawData: analysis.analysis
      };
    }
  }
  
  /**
   * Generate marketing image
   */
  async generateMarketingImage(productType, style = 'professional') {
    console.log(`Generating marketing image for ${productType}...`);
    
    const prompts = {
      creditCard: 'A premium credit card with Emirates NBD branding, elegant design, luxury aesthetic, professional product photography',
      savingsAccount: 'Modern banking concept, piggy bank with coins and growth chart, professional illustration, Emirates NBD colors',
      personalLoan: 'Financial growth and opportunity concept, upward trending arrows, professional design, Emirates NBD branding'
    };
    
    const prompt = prompts[productType] || prompts.creditCard;
    
    const result = await this.imageService.generateImage(prompt, {
      quality: 'hd',
      style: style,
      size: '1024x1024'
    });
    
    return result;
  }
  
  /**
   * Extract confidence score from text
   */
  extractConfidenceScore(text) {
    const match = text.match(/(\d+)(?:%|\s*out of 100)/i);
    return match ? parseInt(match[1]) : 50;
  }
}

/**
 * Usage Examples
 */
async function exampleDocumentProcessing() {
  const processor = new ENBDDocumentProcessingService();
  
  // Process Emirates ID
  const emiratesIDResult = await processor.processEmiratesID(
    'https://example.com/eid-front.jpg',
    'https://example.com/eid-back.jpg'
  );
  console.log('Emirates ID:', emiratesIDResult);
  
  // Process check
  const checkResult = await processor.processCheck(
    'https://example.com/check.jpg'
  );
  console.log('Check:', checkResult);
  
  // Verify signature
  const signatureResult = await processor.verifySignature(
    'https://example.com/doc-signature.jpg',
    'https://example.com/reference-signature.jpg'
  );
  console.log('Signature verification:', signatureResult);
  
  // Process salary certificate
  const salaryResult = await processor.processSalaryCertificate(
    'https://example.com/salary-cert.jpg'
  );
  console.log('Salary certificate:', salaryResult);
  
  // Generate marketing image
  const marketingImage = await processor.generateMarketingImage('creditCard', 'professional');
  console.log('Marketing image:', marketingImage);
}

module.exports = {
  MultiModalImageService,
  ENBDDocumentProcessingService
};
```

**Image Processing Capabilities:**

1. ✅ **Document OCR** - Extract text from images
2. ✅ **Emirates ID Processing** - Extract personal information
3. ✅ **Check Processing** - Extract check details
4. ✅ **Signature Verification** - Compare signatures
5. ✅ **Document Verification** - Detect tampering
6. ✅ **Image Generation** - Create marketing visuals

---

### Q22. How do you implement PDF document processing and understanding?

**Answer:**

```javascript
const { PDFDocument } = require('pdf-lib');
const pdf = require('pdf-parse');
const { ChatOpenAI, OpenAIEmbeddings } = require('@langchain/openai');
const { RecursiveCharacterTextSplitter } = require('langchain/text_splitter');

/**
 * PDF Processing Service
 */
class PDFProcessingService {
  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    this.embeddings = new OpenAIEmbeddings({
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }
  
  /**
   * Extract text from PDF
   */
  async extractText(pdfBuffer) {
    const data = await pdf(pdfBuffer);
    
    return {
      text: data.text,
      numPages: data.numpages,
      info: data.info,
      metadata: data.metadata
    };
  }
  
  /**
   * Extract text with page numbers
   */
  async extractTextByPages(pdfBuffer) {
    // Custom extraction with page info
    const data = await pdf(pdfBuffer, {
      pagerender: (pageData) => {
        return pageData.getTextContent().then((textContent) => {
          return {
            pageNumber: pageData.pageNumber,
            text: textContent.items.map(item => item.str).join(' ')
          };
        });
      }
    });
    
    return data;
  }
  
  /**
   * Chunk PDF content intelligently
   */
  async chunkPDF(text, chunkSize = 1000, overlap = 200) {
    const splitter = new RecursiveCharacterTextSplitter({
      chunkSize: chunkSize,
      chunkOverlap: overlap,
      separators: ['\n\n', '\n', '. ', ' ', '']
    });
    
    const chunks = await splitter.splitText(text);
    
    return chunks;
  }
  
  /**
   * Summarize PDF
   */
  async summarizePDF(text, options = {}) {
    const prompt = `Summarize this document${options.focus ? ` focusing on ${options.focus}` : ''}:

${text.substring(0, 10000)}

Provide a ${options.length || 'concise'} summary with key points.`;

    const response = await this.model.call(prompt);
    
    return {
      summary: response.content,
      length: text.length,
      summarizedFrom: Math.min(10000, text.length)
    };
  }
  
  /**
   * Extract structured data from PDF
   */
  async extractStructuredData(text, schema) {
    const prompt = `Extract the following information from this document:

Schema: ${JSON.stringify(schema, null, 2)}

Document:
${text.substring(0, 5000)}

Return the extracted data as JSON matching the schema.`;

    const response = await this.model.call(prompt);
    
    try {
      return {
        success: true,
        data: JSON.parse(response.content)
      };
    } catch (error) {
      return {
        success: false,
        error: 'Failed to parse structured data',
        rawResponse: response.content
      };
    }
  }
  
  /**
   * Answer questions about PDF
   */
  async answerQuestion(text, question) {
    // Chunk text if too long
    const chunks = await this.chunkPDF(text, 2000, 200);
    
    // Find most relevant chunks
    const relevantChunks = await this.findRelevantChunks(chunks, question, 3);
    
    const context = relevantChunks.join('\n\n');
    
    const prompt = `Based on the following document excerpt, answer the question.

Document:
${context}

Question: ${question}

Answer:`;

    const response = await this.model.call(prompt);
    
    return {
      answer: response.content,
      sources: relevantChunks
    };
  }
  
  /**
   * Find relevant chunks using embeddings
   */
  async findRelevantChunks(chunks, query, topK = 3) {
    // Generate embeddings for query
    const queryEmbedding = await this.embeddings.embedQuery(query);
    
    // Generate embeddings for chunks
    const chunkEmbeddings = await Promise.all(
      chunks.map(chunk => this.embeddings.embedQuery(chunk))
    );
    
    // Calculate similarity scores
    const scores = chunkEmbeddings.map((embedding, idx) => ({
      chunk: chunks[idx],
      score: this.cosineSimilarity(queryEmbedding, embedding)
    }));
    
    // Sort by score and return top K
    scores.sort((a, b) => b.score - a.score);
    
    return scores.slice(0, topK).map(s => s.chunk);
  }
  
  /**
   * Cosine similarity
   */
  cosineSimilarity(vec1, vec2) {
    const dotProduct = vec1.reduce((sum, val, idx) => sum + val * vec2[idx], 0);
    const mag1 = Math.sqrt(vec1.reduce((sum, val) => sum + val * val, 0));
    const mag2 = Math.sqrt(vec2.reduce((sum, val) => sum + val * val, 0));
    return dotProduct / (mag1 * mag2);
  }
  
  /**
   * Classify document type
   */
  async classifyDocument(text) {
    const prompt = `Classify this document into one of the following categories:
    - Bank Statement
    - Loan Agreement
    - Salary Certificate
    - Emirates ID
    - Passport
    - Utility Bill
    - Other

Document excerpt:
${text.substring(0, 1000)}

Return just the category name.`;

    const response = await this.model.call(prompt);
    
    return {
      category: response.content.trim(),
      confidence: 0.8 // Would calculate based on response
    };
  }
  
  /**
   * Validate document completeness
   */
  async validateDocument(text, documentType) {
    const requiredFields = this.getRequiredFields(documentType);
    
    const prompt = `Check if this ${documentType} contains all required fields:

Required fields: ${requiredFields.join(', ')}

Document:
${text.substring(0, 3000)}

Return JSON with:
{
  "complete": true/false,
  "missingFields": [],
  "presentFields": []
}`;

    const response = await this.model.call(prompt);
    
    try {
      return JSON.parse(response.content);
    } catch (error) {
      return {
        complete: false,
        error: 'Failed to validate document'
      };
    }
  }
  
  /**
   * Get required fields for document type
   */
  getRequiredFields(documentType) {
    const fields = {
      'Bank Statement': ['Account Number', 'Statement Period', 'Transactions', 'Balance'],
      'Loan Agreement': ['Loan Amount', 'Interest Rate', 'Term', 'Borrower Name', 'Signatures'],
      'Salary Certificate': ['Employee Name', 'Monthly Salary', 'Company Name', 'Issue Date']
    };
    
    return fields[documentType] || [];
  }
}

/**
 * ENBD PDF Document Intelligence Service
 */
class ENBDPDFIntelligenceService {
  constructor() {
    this.pdfService = new PDFProcessingService();
  }
  
  /**
   * Process loan application documents
   */
  async processLoanApplication(documents) {
    console.log('Processing loan application documents...');
    
    const results = {
      emiratesID: null,
      salaryСertificate: null,
      bankStatements: [],
      validation: { complete: false, issues: [] }
    };
    
    for (const doc of documents) {
      const { text } = await this.pdfService.extractText(doc.buffer);
      const classification = await this.pdfService.classifyDocument(text);
      
      console.log(`Processing ${classification.category}...`);
      
      switch (classification.category) {
        case 'Emirates ID':
          results.emiratesID = await this.processEmiratesIDPDF(text);
          break;
        case 'Salary Certificate':
          results.salaryCertificate = await this.processSalaryCertificate(text);
          break;
        case 'Bank Statement':
          const statement = await this.processBankStatement(text);
          results.bankStatements.push(statement);
          break;
      }
    }
    
    // Validate application completeness
    results.validation = await this.validateLoanApplication(results);
    
    return results;
  }
  
  /**
   * Process Emirates ID PDF
   */
  async processEmiratesIDPDF(text) {
    const schema = {
      idNumber: 'string',
      fullName: 'string',
      nationality: 'string',
      dateOfBirth: 'string',
      expiryDate: 'string'
    };
    
    const result = await this.pdfService.extractStructuredData(text, schema);
    
    return result.data;
  }
  
  /**
   * Process salary certificate
   */
  async processSalaryCertificate(text) {
    const schema = {
      employeeName: 'string',
      employeeID: 'string',
      companyName: 'string',
      designation: 'string',
      monthlySalary: 'number',
      issueDate: 'string'
    };
    
    const result = await this.pdfService.extractStructuredData(text, schema);
    
    return result.data;
  }
  
  /**
   * Process bank statement
   */
  async processBankStatement(text) {
    const schema = {
      accountNumber: 'string',
      statementPeriod: 'string',
      openingBalance: 'number',
      closingBalance: 'number',
      totalCredits: 'number',
      totalDebits: 'number',
      averageBalance: 'number'
    };
    
    const result = await this.pdfService.extractStructuredData(text, schema);
    
    return result.data;
  }
  
  /**
   * Validate loan application
   */
  async validateLoanApplication(applicationData) {
    const issues = [];
    
    if (!applicationData.emiratesID) {
      issues.push({ type: 'error', message: 'Emirates ID missing' });
    }
    
    if (!applicationData.salaryCertificate) {
      issues.push({ type: 'error', message: 'Salary certificate missing' });
    }
    
    if (applicationData.bankStatements.length === 0) {
      issues.push({ type: 'error', message: 'Bank statements missing' });
    } else if (applicationData.bankStatements.length < 3) {
      issues.push({ type: 'warning', message: 'Less than 3 months of bank statements' });
    }
    
    // Check Emirates ID expiry
    if (applicationData.emiratesID && applicationData.emiratesID.expiryDate) {
      const expiryDate = new Date(applicationData.emiratesID.expiryDate);
      if (expiryDate < new Date()) {
        issues.push({ type: 'error', message: 'Emirates ID expired' });
      }
    }
    
    return {
      complete: issues.filter(i => i.type === 'error').length === 0,
      issues: issues
    };
  }
  
  /**
   * Generate document summary
   */
  async generateDocumentSummary(pdfBuffer) {
    const { text } = await this.pdfService.extractText(pdfBuffer);
    
    const summary = await this.pdfService.summarizePDF(text, {
      focus: 'key financial information and important dates',
      length: 'detailed'
    });
    
    return summary;
  }
  
  /**
   * Compare documents
   */
  async compareDocuments(pdfBuffer1, pdfBuffer2) {
    const text1 = (await this.pdfService.extractText(pdfBuffer1)).text;
    const text2 = (await this.pdfService.extractText(pdfBuffer2)).text;
    
    const prompt = `Compare these two documents and identify:
    - Key differences
    - Common information
    - Discrepancies in data

Document 1:
${text1.substring(0, 2000)}

Document 2:
${text2.substring(0, 2000)}

Comparison:`;

    const model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    const response = await model.call(prompt);
    
    return {
      comparison: response.content
    };
  }
}

/**
 * Usage Example
 */
async function examplePDFProcessing() {
  const service = new ENBDPDFIntelligenceService();
  
  // Process loan application
  const documents = [
    { type: 'eid', buffer: Buffer.from('...') },
    { type: 'salary', buffer: Buffer.from('...') },
    { type: 'statement', buffer: Buffer.from('...') }
  ];
  
  const result = await service.processLoanApplication(documents);
  
  console.log('Application Data:', result);
  console.log('Validation:', result.validation);
}

module.exports = {
  PDFProcessingService,
  ENBDPDFIntelligenceService
};
```

---

### Q23. How do you implement audio processing and transcription?

**Answer:**

```javascript
const OpenAI = require('openai');
const fs = require('fs').promises;
const axios = require('axios');

/**
 * Audio Processing Service
 */
class AudioProcessingService {
  constructor() {
    this.openai = new OpenAI({
      apiKey: process.env.OPENAI_API_KEY
    });
  }
  
  /**
   * Transcribe audio file
   */
  async transcribeAudio(audioFilePath, options = {}) {
    const audioFile = await fs.readFile(audioFilePath);
    
    const transcription = await this.openai.audio.transcriptions.create({
      file: audioFile,
      model: options.model || 'whisper-1',
      language: options.language || 'en',
      response_format: options.responseFormat || 'json',
      temperature: options.temperature || 0
    });
    
    return {
      text: transcription.text,
      duration: transcription.duration,
      language: transcription.language
    };
  }
  
  /**
   * Transcribe with timestamps
   */
  async transcribeWithTimestamps(audioFilePath) {
    const audioFile = await fs.readFile(audioFilePath);
    
    const transcription = await this.openai.audio.transcriptions.create({
      file: audioFile,
      model: 'whisper-1',
      response_format: 'verbose_json',
      timestamp_granularities: ['segment']
    });
    
    return {
      text: transcription.text,
      segments: transcription.segments.map(seg => ({
        start: seg.start,
        end: seg.end,
        text: seg.text
      }))
    };
  }
  
  /**
   * Translate audio to English
   */
  async translateAudio(audioFilePath) {
    const audioFile = await fs.readFile(audioFilePath);
    
    const translation = await this.openai.audio.translations.create({
      file: audioFile,
      model: 'whisper-1'
    });
    
    return {
      translation: translation.text
    };
  }
  
  /**
   * Generate speech from text
   */
  async textToSpeech(text, options = {}) {
    const mp3 = await this.openai.audio.speech.create({
      model: options.model || 'tts-1',
      voice: options.voice || 'alloy', // alloy, echo, fable, onyx, nova, shimmer
      input: text,
      speed: options.speed || 1.0
    });
    
    const buffer = Buffer.from(await mp3.arrayBuffer());
    
    return {
      audio: buffer,
      format: 'mp3'
    };
  }
  
  /**
   * Analyze sentiment from audio transcription
   */
  async analyzeSentiment(transcription) {
    const model = new require('@langchain/openai').ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    const prompt = `Analyze the sentiment and tone of this transcription:

"${transcription}"

Provide:
- Overall sentiment (positive/negative/neutral)
- Confidence score (0-1)
- Key emotions detected
- Customer satisfaction level (1-10)

Return as JSON.`;

    const response = await model.call(prompt);
    
    try {
      return JSON.parse(response.content);
    } catch (error) {
      return {
        error: 'Failed to parse sentiment analysis',
        rawResponse: response.content
      };
    }
  }
  
  /**
   * Extract key points from audio
   */
  async extractKeyPoints(transcription) {
    const model = new require('@langchain/openai').ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    const prompt = `Extract key points from this conversation:

"${transcription}"

Provide:
- Main topics discussed
- Action items
- Important dates/numbers
- Customer requests
- Issues raised

Return as structured JSON.`;

    const response = await model.call(prompt);
    
    try {
      return JSON.parse(response.content);
    } catch (error) {
      return {
        error: 'Failed to parse key points',
        rawResponse: response.content
      };
    }
  }
}

/**
 * ENBD Call Center Analytics Service
 */
class ENBDCallCenterAnalytics {
  constructor() {
    this.audioService = new AudioProcessingService();
  }
  
  /**
   * Process customer call
   */
  async processCustomerCall(audioFilePath, metadata = {}) {
    console.log('Processing customer call...');
    
    // Transcribe call
    console.log('1. Transcribing audio...');
    const transcription = await this.audioService.transcribeWithTimestamps(audioFilePath);
    
    // Analyze sentiment
    console.log('2. Analyzing sentiment...');
    const sentiment = await this.audioService.analyzeSentiment(transcription.text);
    
    // Extract key points
    console.log('3. Extracting key points...');
    const keyPoints = await this.audioService.extractKeyPoints(transcription.text);
    
    // Generate summary
    console.log('4. Generating summary...');
    const summary = await this.generateCallSummary(transcription.text, sentiment, keyPoints);
    
    return {
      metadata: {
        callId: metadata.callId,
        duration: metadata.duration,
        timestamp: new Date()
      },
      transcription: transcription,
      sentiment: sentiment,
      keyPoints: keyPoints,
      summary: summary,
      recommendations: await this.generateRecommendations(sentiment, keyPoints)
    };
  }
  
  /**
   * Generate call summary
   */
  async generateCallSummary(transcription, sentiment, keyPoints) {
    const model = new require('@langchain/openai').ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0.3,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    const prompt = `Generate a concise call summary:

Transcription: "${transcription.substring(0, 1000)}"
Sentiment: ${JSON.stringify(sentiment)}
Key Points: ${JSON.stringify(keyPoints)}

Summary (2-3 paragraphs):`;

    const response = await model.call(prompt);
    
    return response.content;
  }
  
  /**
   * Generate recommendations
   */
  async generateRecommendations(sentiment, keyPoints) {
    const recommendations = [];
    
    // Based on sentiment
    if (sentiment.sentiment === 'negative' && sentiment.confidence > 0.7) {
      recommendations.push({
        type: 'urgent',
        action: 'Follow-up call required',
        reason: 'Customer expressed dissatisfaction'
      });
    }
    
    // Based on key points
    if (keyPoints.issues && keyPoints.issues.length > 0) {
      recommendations.push({
        type: 'action',
        action: 'Resolve outstanding issues',
        details: keyPoints.issues
      });
    }
    
    return recommendations;
  }
  
  /**
   * Generate automated response audio
   */
  async generateCustomerResponse(message, voiceType = 'professional') {
    const voiceMap = {
      professional: 'alloy',
      friendly: 'nova',
      formal: 'echo'
    };
    
    const audio = await this.audioService.textToSpeech(message, {
      voice: voiceMap[voiceType] || 'alloy',
      model: 'tts-1-hd'
    });
    
    return audio;
  }
  
  /**
   * Analyze call quality
   */
  async analyzeCallQuality(transcription) {
    const model = new require('@langchain/openai').ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    const prompt = `Analyze this customer service call for quality:

"${transcription}"

Evaluate:
- Agent professionalism (1-10)
- Problem resolution (1-10)
- Communication clarity (1-10)
- Customer satisfaction (1-10)
- Compliance with scripts (yes/no)
- Areas for improvement

Return as JSON.`;

    const response = await model.call(prompt);
    
    try {
      return JSON.parse(response.content);
    } catch (error) {
      return {
        error: 'Failed to analyze call quality',
        rawResponse: response.content
      };
    }
  }
}

/**
 * Usage Example
 */
async function exampleAudioProcessing() {
  const analytics = new ENBDCallCenterAnalytics();
  
  // Process customer call
  const result = await analytics.processCustomerCall('customer-call.mp3', {
    callId: 'CALL-12345',
    duration: 180,
    agent: 'Agent-007'
  });
  
  console.log('Call Analysis:', result);
  
  // Generate automated response
  const responseAudio = await analytics.generateCustomerResponse(
    'Thank you for calling Emirates NBD. Your request has been processed.',
    'professional'
  );
  
  await fs.writeFile('response.mp3', responseAudio.audio);
  console.log('Response audio generated');
}

module.exports = {
  AudioProcessingService,
  ENBDCallCenterAnalytics
};
```

---

### Q24. How do you implement real-time video analysis?

**Answer:**

```javascript
const OpenAI = require('openai');
const ffmpeg = require('fluent-ffmpeg');
const fs = require('fs').promises;
const path = require('path');

/**
 * Video Analysis Service
 */
class VideoAnalysisService {
  constructor() {
    this.openai = new OpenAI({
      apiKey: process.env.OPENAI_API_KEY
    });
  }
  
  /**
   * Extract frames from video
   */
  async extractFrames(videoPath, options = {}) {
    const outputDir = options.outputDir || './frames';
    const fps = options.fps || 1; // 1 frame per second
    
    await fs.mkdir(outputDir, { recursive: true });
    
    return new Promise((resolve, reject) => {
      ffmpeg(videoPath)
        .outputOptions(`-vf fps=${fps}`)
        .output(path.join(outputDir, 'frame-%04d.jpg'))
        .on('end', () => {
          resolve({ outputDir, fps });
        })
        .on('error', reject)
        .run();
    });
  }
  
  /**
   * Analyze video frames
   */
  async analyzeVideo(videoPath, analysisPrompt) {
    // Extract frames
    const { outputDir } = await this.extractFrames(videoPath, { fps: 0.5 }); // 1 frame per 2 seconds
    
    // Get all frame files
    const files = await fs.readdir(outputDir);
    const framePaths = files
      .filter(f => f.endsWith('.jpg'))
      .map(f => path.join(outputDir, f))
      .slice(0, 10); // Analyze first 10 frames
    
    // Analyze each frame
    const frameAnalyses = [];
    
    for (const framePath of framePaths) {
      const frameBuffer = await fs.readFile(framePath);
      const base64Frame = frameBuffer.toString('base64');
      
      const analysis = await this.analyzeFrame(base64Frame, analysisPrompt);
      frameAnalyses.push({
        frame: framePath,
        analysis: analysis
      });
    }
    
    // Aggregate results
    const summary = await this.aggregateFrameAnalyses(frameAnalyses);
    
    // Cleanup
    await this.cleanupFrames(outputDir);
    
    return {
      frames: frameAnalyses,
      summary: summary
    };
  }
  
  /**
   * Analyze single frame
   */
  async analyzeFrame(base64Image, prompt) {
    const response = await this.openai.chat.completions.create({
      model: 'gpt-4-vision-preview',
      messages: [
        {
          role: 'user',
          content: [
            { type: 'text', text: prompt },
            {
              type: 'image_url',
              image_url: {
                url: `data:image/jpeg;base64,${base64Image}`,
                detail: 'low'
              }
            }
          ]
        }
      ],
      max_tokens: 300
    });
    
    return response.choices[0].message.content;
  }
  
  /**
   * Aggregate frame analyses
   */
  async aggregateFrameAnalyses(analyses) {
    const model = new require('@langchain/openai').ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    const analysesText = analyses
      .map((a, idx) => `Frame ${idx + 1}: ${a.analysis}`)
      .join('\n');
    
    const prompt = `Aggregate these frame-by-frame analyses into a comprehensive video summary:

${analysesText}

Provide:
- Overall video content
- Key events or actions
- Timeline of important moments
- Final summary`;

    const response = await model.call(prompt);
    
    return response.content;
  }
  
  /**
   * Cleanup extracted frames
   */
  async cleanupFrames(outputDir) {
    const files = await fs.readdir(outputDir);
    
    for (const file of files) {
      await fs.unlink(path.join(outputDir, file));
    }
    
    await fs.rmdir(outputDir);
  }
  
  /**
   * Extract audio from video
   */
  async extractAudio(videoPath, outputPath) {
    return new Promise((resolve, reject) => {
      ffmpeg(videoPath)
        .output(outputPath)
        .audioCodec('libmp3lame')
        .on('end', () => resolve(outputPath))
        .on('error', reject)
        .run();
    });
  }
}

/**
 * ENBD Video KYC Service
 */
class ENBDVideoKYCService {
  constructor() {
    this.videoService = new VideoAnalysisService();
    this.audioService = new (require('./audio').AudioProcessingService)();
  }
  
  /**
   * Process Video KYC session
   */
  async processVideoKYC(videoPath, metadata = {}) {
    console.log('Processing Video KYC session...');
    
    // Extract and analyze audio
    console.log('1. Processing audio...');
    const audioPath = './temp-audio.mp3';
    await this.videoService.extractAudio(videoPath, audioPath);
    
    const transcription = await this.audioService.transcribeAudio(audioPath);
    
    // Analyze video frames
    console.log('2. Analyzing video frames...');
    const videoAnalysis = await this.videoService.analyzeVideo(
      videoPath,
      `Analyze this frame from a banking KYC video call. Check for:
      - Is a person clearly visible?
      - Is the person holding an ID document?
      - Is the ID document clearly visible?
      - Any signs of fraud or suspicious activity?`
    );
    
    // Verify identity
    console.log('3. Verifying identity...');
    const identityVerification = await this.verifyIdentity(videoAnalysis);
    
    // Liveness detection
    console.log('4. Performing liveness detection...');
    const livenessCheck = await this.checkLiveness(videoAnalysis);
    
    // Generate report
    console.log('5. Generating report...');
    const report = await this.generateKYCReport({
      metadata,
      transcription,
      videoAnalysis,
      identityVerification,
      livenessCheck
    });
    
    // Cleanup
    await fs.unlink(audioPath);
    
    return report;
  }
  
  /**
   * Verify identity from video
   */
  async verifyIdentity(videoAnalysis) {
    const model = new require('@langchain/openai').ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    const prompt = `Based on this video analysis, verify identity:

${JSON.stringify(videoAnalysis, null, 2)}

Check:
- ID document is clearly visible
- Photo on ID matches person in video
- ID appears authentic
- All required fields are visible

Return JSON:
{
  "verified": true/false,
  "confidence": 0-1,
  "issues": []
}`;

    const response = await model.call(prompt);
    
    try {
      return JSON.parse(response.content);
    } catch (error) {
      return {
        verified: false,
        confidence: 0,
        issues: ['Failed to analyze video']
      };
    }
  }
  
  /**
   * Check liveness (detect real person vs photo/video)
   */
  async checkLiveness(videoAnalysis) {
    const model = new require('@langchain/openai').ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    const prompt = `Analyze for liveness detection:

${JSON.stringify(videoAnalysis, null, 2)}

Check for:
- Natural movement and blinking
- Depth perception (not a flat photo)
- Lighting consistency
- Natural expressions
- Response to instructions

Return JSON:
{
  "isLive": true/false,
  "confidence": 0-1,
  "indicators": []
}`;

    const response = await model.call(prompt);
    
    try {
      return JSON.parse(response.content);
    } catch (error) {
      return {
        isLive: false,
        confidence: 0,
        indicators: []
      };
    }
  }
  
  /**
   * Generate KYC report
   */
  async generateKYCReport(data) {
    const passed = 
      data.identityVerification.verified &&
      data.identityVerification.confidence > 0.8 &&
      data.livenessCheck.isLive &&
      data.livenessCheck.confidence > 0.8;
    
    return {
      status: passed ? 'PASSED' : 'FAILED',
      timestamp: new Date(),
      metadata: data.metadata,
      verification: {
        identity: data.identityVerification,
        liveness: data.livenessCheck
      },
      transcription: data.transcription.text,
      videoAnalysis: data.videoAnalysis.summary,
      recommendations: passed 
        ? ['Proceed with account opening']
        : ['Manual review required', 'Request additional verification']
    };
  }
  
  /**
   * Detect fraud indicators
   */
  async detectFraud(videoAnalysis, transcription) {
    const model = new require('@langchain/openai').ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
    
    const prompt = `Detect potential fraud indicators:

Video Analysis: ${JSON.stringify(videoAnalysis.summary)}
Transcription: "${transcription}"

Check for:
- Suspicious behavior
- Inconsistent information
- Coaching or prompting
- Multiple persons
- Deepfake indicators

Return JSON with fraud score (0-1) and detected indicators.`;

    const response = await model.call(prompt);
    
    try {
      return JSON.parse(response.content);
    } catch (error) {
      return {
        fraudScore: 0,
        indicators: []
      };
    }
  }
}

/**
 * Usage Example
 */
async function exampleVideoKYC() {
  const kycService = new ENBDVideoKYCService();
  
  const report = await kycService.processVideoKYC('kyc-session.mp4', {
    customerId: 'CUST-12345',
    sessionId: 'KYC-SESSION-789',
    timestamp: new Date()
  });
  
  console.log('KYC Report:', report);
}

module.exports = {
  VideoAnalysisService,
  ENBDVideoKYCService
};
```

---

### Q25. How do you implement multi-modal embeddings and search?

**Answer:**

```javascript
const { OpenAIEmbeddings } = require('@langchain/openai');
const OpenAI = require('openai');
const { Pinecone } = require('@pinecone-database/pinecone');

/**
 * Multi-Modal Embeddings Service
 */
class MultiModalEmbeddingsService {
  constructor() {
    this.openai = new OpenAI({
      apiKey: process.env.OPENAI_API_KEY
    });
    
    this.textEmbeddings = new OpenAIEmbeddings({
      openAIApiKey: process.env.OPENAI_API_KEY,
      modelName: 'text-embedding-3-small'
    });
    
    this.pinecone = new Pinecone({
      apiKey: process.env.PINECONE_API_KEY
    });
  }
  
  /**
   * Generate text embedding
   */
  async embedText(text) {
    const embedding = await this.textEmbeddings.embedQuery(text);
    return embedding;
  }
  
  /**
   * Generate image embedding (using CLIP)
   */
  async embedImage(imageUrl) {
    // Get image description first
    const description = await this.openai.chat.completions.create({
      model: 'gpt-4-vision-preview',
      messages: [
        {
          role: 'user',
          content: [
            { type: 'text', text: 'Describe this image in detail for semantic search.' },
            { type: 'image_url', image_url: { url: imageUrl } }
          ]
        }
      ],
      max_tokens: 300
    });
    
    // Embed the description
    const descriptionText = description.choices[0].message.content;
    const embedding = await this.embedText(descriptionText);
    
    return {
      embedding: embedding,
      description: descriptionText
    };
  }
  
  /**
   * Index multi-modal document
   */
  async indexDocument(documentId, content, metadata = {}) {
    const index = this.pinecone.index('enbd-multimodal');
    
    const vectors = [];
    
    // Index text content
    if (content.text) {
      const textEmbedding = await this.embedText(content.text);
      
      vectors.push({
        id: `${documentId}-text`,
        values: textEmbedding,
        metadata: {
          ...metadata,
          type: 'text',
          content: content.text.substring(0, 500),
          documentId: documentId
        }
      });
    }
    
    // Index images
    if (content.images && content.images.length > 0) {
      for (let i = 0; i < content.images.length; i++) {
        const imageData = await this.embedImage(content.images[i].url);
        
        vectors.push({
          id: `${documentId}-image-${i}`,
          values: imageData.embedding,
          metadata: {
            ...metadata,
            type: 'image',
            content: imageData.description,
            imageUrl: content.images[i].url,
            documentId: documentId
          }
        });
      }
    }
    
    // Upsert vectors
    await index.upsert(vectors);
    
    return {
      documentId: documentId,
      vectorsIndexed: vectors.length
    };
  }
  
  /**
   * Search across text and images
   */
  async multiModalSearch(query, options = {}) {
    const queryEmbedding = await this.embedText(query);
    
    const index = this.pinecone.index('enbd-multimodal');
    
    const results = await index.query({
      vector: queryEmbedding,
      topK: options.topK || 10,
      includeMetadata: true,
      filter: options.filter || {}
    });
    
    // Group by document
    const grouped = this.groupByDocument(results.matches);
    
    return {
      results: grouped,
      query: query
    };
  }
  
  /**
   * Group results by document
   */
  groupByDocument(matches) {
    const grouped = {};
    
    for (const match of matches) {
      const docId = match.metadata.documentId;
      
      if (!grouped[docId]) {
        grouped[docId] = {
          documentId: docId,
          score: 0,
          matches: []
        };
      }
      
      grouped[docId].matches.push({
        type: match.metadata.type,
        content: match.metadata.content,
        score: match.score,
        imageUrl: match.metadata.imageUrl
      });
      
      // Use highest score
      grouped[docId].score = Math.max(grouped[docId].score, match.score);
    }
    
    return Object.values(grouped).sort((a, b) => b.score - a.score);
  }
  
  /**
   * Hybrid search (text + semantic)
   */
  async hybridSearch(query, keywords, options = {}) {
    // Semantic search
    const semanticResults = await this.multiModalSearch(query, options);
    
    // Keyword filter
    const filtered = semanticResults.results.filter(result => {
      const content = result.matches.map(m => m.content).join(' ').toLowerCase();
      return keywords.some(keyword => content.includes(keyword.toLowerCase()));
    });
    
    return {
      results: filtered,
      query: query,
      keywords: keywords
    };
  }
}

/**
 * ENBD Multi-Modal Knowledge Base
 */
class ENBDMultiModalKnowledgeBase {
  constructor() {
    this.embeddingService = new MultiModalEmbeddingsService();
  }
  
  /**
   * Index banking document with images
   */
  async indexBankingDocument(document) {
    console.log(`Indexing document: ${document.id}`);
    
    const result = await this.embeddingService.indexDocument(
      document.id,
      {
        text: document.text,
        images: document.images || []
      },
      {
        category: document.category,
        title: document.title,
        date: document.date
      }
    );
    
    console.log(`✅ Indexed ${result.vectorsIndexed} vectors`);
    
    return result;
  }
  
  /**
   * Search knowledge base
   */
  async search(query, options = {}) {
    console.log(`Searching for: "${query}"`);
    
    const results = await this.embeddingService.multiModalSearch(query, {
      topK: options.limit || 5,
      filter: options.category ? { category: options.category } : {}
    });
    
    return results.results.map(result => ({
      documentId: result.documentId,
      relevance: result.score,
      matches: result.matches
    }));
  }
  
  /**
   * Find similar documents
   */
  async findSimilar(documentId, limit = 5) {
    // Get document content
    const index = this.embeddingService.pinecone.index('enbd-multimodal');
    
    const vector = await index.fetch([`${documentId}-text`]);
    
    if (vector.records[`${documentId}-text`]) {
      const results = await index.query({
        vector: vector.records[`${documentId}-text`].values,
        topK: limit + 1, // +1 to exclude self
        includeMetadata: true
      });
      
      // Filter out the source document
      const similar = results.matches
        .filter(m => m.metadata.documentId !== documentId)
        .slice(0, limit);
      
      return similar.map(m => ({
        documentId: m.metadata.documentId,
        similarity: m.score,
        type: m.metadata.type,
        content: m.metadata.content
      }));
    }
    
    return [];
  }
}

/**
 * Usage Examples
 */
async function exampleMultiModalSearch() {
  const kb = new ENBDMultiModalKnowledgeBase();
  
  // Index document with text and images
  await kb.indexBankingDocument({
    id: 'DOC-001',
    title: 'Credit Card Application Guide',
    category: 'products',
    text: 'Apply for Emirates NBD credit card online. Get instant approval...',
    images: [
      { url: 'https://example.com/card-image.jpg' },
      { url: 'https://example.com/application-form.jpg' }
    ],
    date: '2024-01-15'
  });
  
  // Search across text and images
  const results = await kb.search('How to apply for credit card?', {
    limit: 3,
    category: 'products'
  });
  
  console.log('Search Results:', results);
  
  // Find similar documents
  const similar = await kb.findSimilar('DOC-001', 5);
  console.log('Similar Documents:', similar);
}

module.exports = {
  MultiModalEmbeddingsService,
  ENBDMultiModalKnowledgeBase
};
```

**Multi-Modal AI Capabilities:**

1. ✅ **Image Generation** - DALL-E for marketing visuals
2. ✅ **Image Analysis** - GPT-4 Vision for document processing
3. ✅ **PDF Processing** - Extract and analyze PDF documents
4. ✅ **Audio Transcription** - Whisper for call center analytics
5. ✅ **Video Analysis** - Frame-by-frame analysis for Video KYC
6. ✅ **Multi-Modal Search** - Search across text, images, audio, video

---

**Summary Q21-Q25:**
- Image generation and analysis (DALL-E, GPT-4 Vision) ✅
- PDF document processing and intelligence ✅
- Audio transcription and call center analytics ✅
- Video analysis for KYC and fraud detection ✅
- Multi-modal embeddings and semantic search ✅
