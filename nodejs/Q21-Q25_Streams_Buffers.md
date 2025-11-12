# Node.js Questions 21-25: Streams and Buffers

---

### Q21. What are the four types of streams in Node.js? Explain with banking examples.

**Answer:**

**Four Types of Streams:**

1. **Readable** - Data source (read from)
2. **Writable** - Data destination (write to)
3. **Duplex** - Both readable and writable
4. **Transform** - Duplex stream that modifies data

**Banking Examples:**

```javascript
const { Readable, Writable, Duplex, Transform } = require('stream');
const fs = require('fs');

// 1. READABLE STREAM - Reading transaction log
class TransactionLogReader extends Readable {
  constructor(transactions) {
    super({ objectMode: true });
    this.transactions = transactions;
    this.index = 0;
  }
  
  _read() {
    if (this.index < this.transactions.length) {
      this.push(this.transactions[this.index]);
      this.index++;
    } else {
      this.push(null); // Signal end
    }
  }
}

// Usage
const transactions = [
  { id: 'TXN-1', amount: 1000 },
  { id: 'TXN-2', amount: 2000 },
  { id: 'TXN-3', amount: 3000 }
];

const reader = new TransactionLogReader(transactions);
reader.on('data', (txn) => {
  console.log('Transaction:', txn);
});

// 2. WRITABLE STREAM - Writing to audit log
class AuditLogWriter extends Writable {
  constructor(filePath) {
    super({ objectMode: true });
    this.filePath = filePath;
    this.fileStream = fs.createWriteStream(filePath, { flags: 'a' });
  }
  
  _write(chunk, encoding, callback) {
    const logEntry = `[${new Date().toISOString()}] ${JSON.stringify(chunk)}\n`;
    this.fileStream.write(logEntry, callback);
  }
  
  _final(callback) {
    this.fileStream.end(callback);
  }
}

// Usage
const auditLogger = new AuditLogWriter('./audit.log');
auditLogger.write({ action: 'TRANSFER', amount: 5000 });
auditLogger.write({ action: 'WITHDRAWAL', amount: 1000 });
auditLogger.end();

// 3. DUPLEX STREAM - Socket connection for real-time banking
class BankingSocket extends Duplex {
  constructor() {
    super({ objectMode: true });
    this.buffer = [];
  }
  
  _read() {
    if (this.buffer.length > 0) {
      this.push(this.buffer.shift());
    }
  }
  
  _write(chunk, encoding, callback) {
    console.log('Received:', chunk);
    // Echo back with processing
    const response = {
      ...chunk,
      processed: true,
      timestamp: new Date()
    };
    this.buffer.push(response);
    callback();
  }
}

// Usage
const socket = new BankingSocket();
socket.on('data', (data) => {
  console.log('Response:', data);
});
socket.write({ command: 'GET_BALANCE', accountId: 'ACC-001' });

// 4. TRANSFORM STREAM - Data transformation
class TransactionEnricher extends Transform {
  constructor() {
    super({ objectMode: true });
  }
  
  _transform(transaction, encoding, callback) {
    // Add computed fields
    const enriched = {
      ...transaction,
      fee: transaction.amount * 0.01,
      tax: transaction.amount * 0.05,
      timestamp: new Date().toISOString()
    };
    
    this.push(enriched);
    callback();
  }
}

// Usage with pipeline
const { pipeline } = require('stream');

const enricher = new TransactionEnricher();

pipeline(
  new TransactionLogReader(transactions),
  enricher,
  new AuditLogWriter('./enriched.log'),
  (err) => {
    if (err) {
      console.error('Pipeline failed:', err);
    } else {
      console.log('Pipeline successful');
    }
  }
);
```

### Q22. Explain Buffer in Node.js. When would you use it in banking applications?

**Answer:**

**Buffer** is a fixed-size chunk of memory for binary data, used when dealing with raw binary data or encodings.

**Banking Use Cases:**
- File encryption/decryption
- Binary protocol communication
- Image processing (checks, signatures)
- Cryptographic operations
- Network data transmission

**Complete Example:**

```javascript
const crypto = require('crypto');

class SecureDataHandler {
  // 1. Creating Buffers
  demonstrateBufferCreation() {
    // From string
    const buf1 = Buffer.from('Account: ACC-12345', 'utf8');
    console.log('Buffer from string:', buf1);
    console.log('As string:', buf1.toString());
    
    // Allocate empty buffer
    const buf2 = Buffer.alloc(10); // 10 bytes, filled with 0
    console.log('Empty buffer:', buf2);
    
    // Unsafe allocation (faster, but uninitialized)
    const buf3 = Buffer.allocUnsafe(10);
    buf3.fill(0); // Must fill manually
    
    // From array
    const buf4 = Buffer.from([0x62, 0x75, 0x66, 0x66, 0x65, 0x72]);
    console.log('From array:', buf4.toString()); // 'buffer'
    
    return { buf1, buf2, buf3, buf4 };
  }
  
  // 2. Encryption using Buffer
  encryptAccountNumber(accountNumber) {
    const algorithm = 'aes-256-cbc';
    const key = crypto.randomBytes(32);
    const iv = crypto.randomBytes(16);
    
    const cipher = crypto.createCipheriv(algorithm, key, iv);
    
    let encrypted = cipher.update(accountNumber, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    return {
      encrypted,
      key: key.toString('hex'),
      iv: iv.toString('hex')
    };
  }
  
  decryptAccountNumber(encrypted, keyHex, ivHex) {
    const algorithm = 'aes-256-cbc';
    const key = Buffer.from(keyHex, 'hex');
    const iv = Buffer.from(ivHex, 'hex');
    
    const decipher = crypto.createDecipheriv(algorithm, key, iv);
    
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
  
  // 3. Binary data manipulation
  encodeTransactionData(transaction) {
    // Create binary format for network transmission
    const buffer = Buffer.alloc(32);
    
    // Write transaction ID (4 bytes)
    buffer.writeUInt32BE(parseInt(transaction.id.replace('TXN-', '')), 0);
    
    // Write amount (8 bytes, double)
    buffer.writeDoubleBE(transaction.amount, 4);
    
    // Write timestamp (8 bytes)
    buffer.writeBigInt64BE(BigInt(transaction.timestamp), 12);
    
    // Write type (1 byte)
    const types = { TRANSFER: 1, WITHDRAWAL: 2, DEPOSIT: 3 };
    buffer.writeUInt8(types[transaction.type] || 0, 20);
    
    return buffer;
  }
  
  decodeTransactionData(buffer) {
    return {
      id: `TXN-${buffer.readUInt32BE(0)}`,
      amount: buffer.readDoubleBE(4),
      timestamp: Number(buffer.readBigInt64BE(12)),
      type: ['', 'TRANSFER', 'WITHDRAWAL', 'DEPOSIT'][buffer.readUInt8(20)]
    };
  }
  
  // 4. Hash generation for data integrity
  generateTransactionHash(transaction) {
    const data = JSON.stringify(transaction);
    const hash = crypto.createHash('sha256');
    hash.update(data);
    return hash.digest('hex');
  }
  
  verifyTransactionIntegrity(transaction, expectedHash) {
    const actualHash = this.generateTransactionHash(transaction);
    return actualHash === expectedHash;
  }
}

// Usage
const handler = new SecureDataHandler();

// Test encryption
const accountNumber = 'ACC-123456789';
const encrypted = handler.encryptAccountNumber(accountNumber);
console.log('Encrypted:', encrypted);

const decrypted = handler.decryptAccountNumber(
  encrypted.encrypted,
  encrypted.key,
  encrypted.iv
);
console.log('Decrypted:', decrypted);

// Test binary encoding
const txn = {
  id: 'TXN-12345',
  amount: 5000.50,
  timestamp: Date.now(),
  type: 'TRANSFER'
};

const encoded = handler.encodeTransactionData(txn);
console.log('Encoded (binary):', encoded);

const decoded = handler.decodeTransactionData(encoded);
console.log('Decoded:', decoded);
```

### Q23. How do you handle large file processing efficiently using streams?

**Answer:**

**Problem:** Loading large files (GB+) into memory causes OOM errors.
**Solution:** Use streams to process data in chunks.

**Banking Example: Process 10GB transaction CSV**

```javascript
const fs = require('fs');
const { Transform, pipeline } = require('stream');
const csv = require('csv-parser');
const zlib = require('zlib');

class LargeFileProcessor {
  // Process large CSV file with streaming
  async processTransactionFile(inputPath, outputPath) {
    return new Promise((resolve, reject) => {
      const stats = {
        totalRows: 0,
        validTransactions: 0,
        invalidTransactions: 0,
        totalAmount: 0,
        largeTransactions: [],
        startTime: Date.now()
      };
      
      pipeline(
        // 1. Read file as stream
        fs.createReadStream(inputPath),
        
        // 2. Parse CSV
        csv(),
        
        // 3. Validate and transform
        new Transform({
          objectMode: true,
          transform(row, encoding, callback) {
            stats.totalRows++;
            
            try {
              const transaction = {
                id: row.id,
                date: new Date(row.date),
                amount: parseFloat(row.amount),
                from: row.from_account,
                to: row.to_account,
                status: row.status
              };
              
              // Validation
              if (transaction.amount > 0 && transaction.from && transaction.to) {
                stats.validTransactions++;
                stats.totalAmount += transaction.amount;
                
                // Track large transactions
                if (transaction.amount >= 100000) {
                  stats.largeTransactions.push(transaction);
                }
                
                // Pass through
                this.push(JSON.stringify(transaction) + '\n');
              } else {
                stats.invalidTransactions++;
              }
              
            } catch (error) {
              stats.invalidTransactions++;
            }
            
            // Log progress every 10000 rows
            if (stats.totalRows % 10000 === 0) {
              console.log(`Processed ${stats.totalRows} rows...`);
            }
            
            callback();
          }
        }),
        
        // 4. Compress output
        zlib.createGzip(),
        
        // 5. Write to file
        fs.createWriteStream(outputPath),
        
        (err) => {
          if (err) {
            reject(err);
          } else {
            stats.duration = Date.now() - stats.startTime;
            resolve(stats);
          }
        }
      );
    });
  }
  
  // Split large file into chunks
  async* readInChunks(filePath, chunkSize = 1024 * 1024) {
    const stream = fs.createReadStream(filePath, {
      highWaterMark: chunkSize
    });
    
    for await (const chunk of stream) {
      yield chunk;
    }
  }
  
  // Process file in parallel batches
  async processBatches(filePath, batchSize = 1000) {
    const readStream = fs.createReadStream(filePath);
    const csvStream = csv();
    
    let batch = [];
    let batchNumber = 0;
    
    return new Promise((resolve, reject) => {
      readStream
        .pipe(csvStream)
        .on('data', async (row) => {
          batch.push(row);
          
          if (batch.length >= batchSize) {
            const currentBatch = [...batch];
            batch = [];
            batchNumber++;
            
            // Pause stream while processing batch
            csvStream.pause();
            
            try {
              await this.processBatch(currentBatch, batchNumber);
              csvStream.resume();
            } catch (error) {
              reject(error);
            }
          }
        })
        .on('end', async () => {
          // Process remaining batch
          if (batch.length > 0) {
            await this.processBatch(batch, batchNumber + 1);
          }
          resolve();
        })
        .on('error', reject);
    });
  }
  
  async processBatch(batch, batchNumber) {
    console.log(`Processing batch ${batchNumber} (${batch.length} items)`);
    
    // Simulate processing
    await new Promise(resolve => setTimeout(resolve, 100));
    
    // Insert to database, etc.
    return batch;
  }
}

// Usage
async function processLargeFile() {
  const processor = new LargeFileProcessor();
  
  console.log('Processing large transaction file...\n');
  
  try {
    const stats = await processor.processTransactionFile(
      './transactions_large.csv',
      './transactions_processed.json.gz'
    );
    
    console.log('\n=== Processing Complete ===');
    console.log(`Total rows: ${stats.totalRows}`);
    console.log(`Valid: ${stats.validTransactions}`);
    console.log(`Invalid: ${stats.invalidTransactions}`);
    console.log(`Total amount: $${stats.totalAmount.toFixed(2)}`);
    console.log(`Large transactions: ${stats.largeTransactions.length}`);
    console.log(`Duration: ${stats.duration}ms`);
    
  } catch (error) {
    console.error('Processing failed:', error);
  }
}
```

### Q24. What is stream backpressure and how do you handle it?

**Answer:**

**Backpressure** occurs when a writable stream can't handle data as fast as a readable stream produces it.

**Handling Backpressure:**

```javascript
const fs = require('fs');
const { Transform } = require('stream');

// BAD: Ignoring backpressure
function ignoringBackpressure() {
  const readable = fs.createReadStream('large-file.txt');
  const writable = fs.createWriteStream('output.txt');
  
  readable.on('data', (chunk) => {
    // WRONG: Ignoring return value
    writable.write(chunk); // Can cause memory issues!
  });
}

// GOOD: Handling backpressure manually
function handlingBackpressure() {
  const readable = fs.createReadStream('large-file.txt');
  const writable = fs.createWriteStream('output.txt');
  
  readable.on('data', (chunk) => {
    const canContinue = writable.write(chunk);
    
    if (!canContinue) {
      // Backpressure detected - pause reading
      readable.pause();
      console.log('⚠️ Backpressure - pausing read stream');
    }
  });
  
  writable.on('drain', () => {
    // Buffer drained - resume reading
    readable.resume();
    console.log('✓ Buffer drained - resuming read stream');
  });
  
  readable.on('end', () => {
    writable.end();
  });
}

// BEST: Using pipe (automatic backpressure)
function usingPipe() {
  const readable = fs.createReadStream('large-file.txt');
  const writable = fs.createWriteStream('output.txt');
  
  readable.pipe(writable);
}

// Banking: Transaction processing with backpressure
class BackpressureHandler {
  async processTransactions(transactions) {
    const { Readable, Writable, pipeline } = require('stream');
    
    // Source stream
    const source = Readable.from(transactions, { objectMode: true });
    
    // Processing stream with artificial delay
    const processor = new Transform({
      objectMode: true,
      async transform(transaction, encoding, callback) {
        // Simulate slow processing
        await new Promise(resolve => setTimeout(resolve, 100));
        
        const processed = {
          ...transaction,
          processed: true,
          timestamp: new Date()
        };
        
        callback(null, processed);
      }
    });
    
    // Destination stream
    const destination = new Writable({
      objectMode: true,
      write(transaction, encoding, callback) {
        console.log('Saved:', transaction.id);
        callback();
      }
    });
    
    return new Promise((resolve, reject) => {
      pipeline(source, processor, destination, (err) => {
        if (err) reject(err);
        else resolve();
      });
    });
  }
}
```

### Q25. How do you implement custom streams for banking operations?

**Answer:**

**Complete Custom Stream Implementation:**

```javascript
const { Readable, Writable, Transform, Duplex } = require('stream');

// 1. Custom Readable: Real-time transaction feed
class TransactionFeed extends Readable {
  constructor(options = {}) {
    super({ objectMode: true, ...options });
    this.transactionCount = 0;
    this.maxTransactions = options.maxTransactions || 100;
    this.interval = options.interval || 1000;
    this.timer = null;
  }
  
  _read() {
    if (this.transactionCount >= this.maxTransactions) {
      this.push(null); // End stream
      if (this.timer) clearInterval(this.timer);
      return;
    }
    
    // Generate transactions periodically
    if (!this.timer) {
      this.timer = setInterval(() => {
        if (this.transactionCount < this.maxTransactions) {
          const transaction = this.generateTransaction();
          this.push(transaction);
          this.transactionCount++;
        } else {
          this.push(null);
          clearInterval(this.timer);
        }
      }, this.interval);
    }
  }
  
  generateTransaction() {
    return {
      id: `TXN-${Date.now()}-${this.transactionCount}`,
      amount: Math.random() * 10000,
      type: ['TRANSFER', 'WITHDRAWAL', 'DEPOSIT'][Math.floor(Math.random() * 3)],
      timestamp: new Date()
    };
  }
  
  _destroy(error, callback) {
    if (this.timer) clearInterval(this.timer);
    callback(error);
  }
}

// 2. Custom Writable: Database writer
class TransactionWriter extends Writable {
  constructor(database) {
    super({ objectMode: true });
    this.database = database;
    this.writeCount = 0;
  }
  
  async _write(transaction, encoding, callback) {
    try {
      await this.database.insert(transaction);
      this.writeCount++;
      
      if (this.writeCount % 10 === 0) {
        console.log(`Written ${this.writeCount} transactions`);
      }
      
      callback();
    } catch (error) {
      callback(error);
    }
  }
  
  _final(callback) {
    console.log(`Total transactions written: ${this.writeCount}`);
    callback();
  }
}

// 3. Custom Transform: Fraud detection
class FraudDetector extends Transform {
  constructor() {
    super({ objectMode: true });
    this.suspiciousCount = 0;
  }
  
  _transform(transaction, encoding, callback) {
    const riskScore = this.calculateRiskScore(transaction);
    
    const enriched = {
      ...transaction,
      riskScore,
      flagged: riskScore > 70
    };
    
    if (enriched.flagged) {
      this.suspiciousCount++;
      console.warn(`⚠️ Suspicious transaction: ${transaction.id}`);
    }
    
    this.push(enriched);
    callback();
  }
  
  calculateRiskScore(txn) {
    let score = 0;
    
    if (txn.amount > 50000) score += 30;
    if (txn.type === 'TRANSFER') score += 20;
    if (new Date().getHours() < 6 || new Date().getHours() > 22) score += 25;
    
    return score + Math.random() * 25;
  }
  
  _flush(callback) {
    console.log(`Total suspicious transactions: ${this.suspiciousCount}`);
    callback();
  }
}

// 4. Custom Duplex: Transaction processor service
class TransactionProcessor extends Duplex {
  constructor() {
    super({ objectMode: true });
    this.queue = [];
    this.processing = false;
  }
  
  _read() {
    if (this.queue.length > 0 && !this.processing) {
      this.push(this.queue.shift());
    }
  }
  
  async _write(transaction, encoding, callback) {
    try {
      const processed = await this.processTransaction(transaction);
      this.queue.push(processed);
      callback();
    } catch (error) {
      callback(error);
    }
  }
  
  async processTransaction(txn) {
    // Simulate processing
    await new Promise(resolve => setTimeout(resolve, 50));
    
    return {
      ...txn,
      processed: true,
      processedAt: new Date(),
      confirmationNumber: `CONF-${Date.now()}`
    };
  }
}

// Complete pipeline example
async function runBankingPipeline() {
  const { pipeline } = require('stream');
  
  // Mock database
  const database = {
    transactions: [],
    async insert(txn) {
      this.transactions.push(txn);
      return Promise.resolve();
    }
  };
  
  const feed = new TransactionFeed({ maxTransactions: 50, interval: 100 });
  const fraudDetector = new FraudDetector();
  const writer = new TransactionWriter(database);
  
  console.log('Starting banking pipeline...\n');
  
  return new Promise((resolve, reject) => {
    pipeline(
      feed,
      fraudDetector,
      writer,
      (err) => {
        if (err) {
          console.error('Pipeline failed:', err);
          reject(err);
        } else {
          console.log('\n✓ Pipeline completed successfully');
          console.log(`Total transactions in DB: ${database.transactions.length}`);
          resolve(database.transactions);
        }
      }
    );
  });
}

// runBankingPipeline().catch(console.error);
```

**Key Takeaways:**
- Always handle backpressure
- Use `objectMode: true` for object streams
- Implement `_read()`, `_write()`, `_transform()` appropriately
- Clean up resources in `_destroy()` and `_final()`
- Use `pipeline()` for automatic error handling

---

**🎯 Section 3 Complete! Questions 21-25 Finished**

Topics: Streams, Buffers, Backpressure, File Processing
