# Node.js Interview Question: File System (fs) Module
## Question 9: How to Work with File System in Node.js? Sync vs Async?

---

## 📋 Summary of What Will Be Covered

In this comprehensive guide, you'll learn:

1. **Synchronous vs Asynchronous fs operations** - When to use each
2. **Callback-based fs API** - Traditional error-first callbacks
3. **Promise-based fs API** - Modern fs.promises
4. **Reading files** - readFile, readFileSync, createReadStream
5. **Writing files** - writeFile, appendFile, createWriteStream
6. **Directory operations** - mkdir, readdir, rmdir
7. **File watching** - fs.watch, fs.watchFile
8. **Banking Examples**: Transaction logs, audit files, statement generation, backup systems

**Why This Matters in Banking**:
- Store transaction logs persistently
- Generate PDF statements and reports
- Maintain audit trails for compliance
- Backup customer data securely
- Process large CSV files (transaction imports)
- Monitor configuration changes

---

## 🎯 Synchronous vs Asynchronous Operations

### Synchronous Operations (Blocking)

```javascript
import fs from 'fs';

// Blocks the entire Node.js process
console.log('Before read');
const data = fs.readFileSync('data.txt', 'utf8');
console.log('File content:', data);
console.log('After read');

// Output:
// Before read
// File content: Hello World
// After read
```

**Characteristics**:
- ✅ Simple, sequential code
- ❌ Blocks event loop
- ❌ No other operations can run
- ⚠️ Use only at startup or in CLI tools

---

### Asynchronous Operations (Non-Blocking)

```javascript
import fs from 'fs';

// Non-blocking - doesn't wait
console.log('Before read');
fs.readFile('data.txt', 'utf8', (err, data) => {
  if (err) throw err;
  console.log('File content:', data);
});
console.log('After read');

// Output:
// Before read
// After read  ← Executes immediately
// File content: Hello World  ← Executes later
```

**Characteristics**:
- ✅ Non-blocking event loop
- ✅ Other operations continue
- ✅ Better performance for I/O
- ⚠️ Callback-based (can lead to callback hell)

---

### Promise-Based Operations (Modern)

```javascript
import fs from 'fs/promises';

async function readData() {
  console.log('Before read');
  const data = await fs.readFile('data.txt', 'utf8');
  console.log('File content:', data);
  console.log('After read');
}

readData();

// Output:
// Before read
// File content: Hello World
// After read
```

**Characteristics**:
- ✅ Non-blocking
- ✅ Clean async/await syntax
- ✅ Better error handling
- ✅ **Recommended for modern code**

---

## 📊 Visual: File System API Comparison

```
┌──────────────────────────────────────────────────────────────────┐
│                    Synchronous (Blocking)                        │
│  fs.readFileSync()  ───► Blocks event loop                      │
│  fs.writeFileSync() ───► Use only at startup                    │
│  fs.mkdirSync()                                                  │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│               Asynchronous Callback (Non-Blocking)               │
│  fs.readFile()      ───► Error-first callback                   │
│  fs.writeFile()     ───► Can lead to callback hell              │
│  fs.mkdir()         ───► Legacy approach                        │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│            Asynchronous Promise (Modern, Recommended)            │
│  fs.promises.readFile()  ───► Clean async/await                 │
│  fs.promises.writeFile() ───► Better error handling             │
│  fs.promises.mkdir()     ───► Modern best practice              │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                      Streams (Large Files)                       │
│  fs.createReadStream()   ───► Memory efficient                  │
│  fs.createWriteStream()  ───► Process chunks                    │
│  pipe()                  ───► Best for large files              │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🏦 Example 1: Banking Transaction Log System

This example demonstrates file operations for transaction logging.

**File: `transaction-logger.js`**

```javascript
import fs from 'fs/promises';
import { createWriteStream, createReadStream } from 'fs';
import path from 'path';
import { pipeline } from 'stream/promises';
import { Transform } from 'stream';

/**
 * Banking Transaction Logger
 * Demonstrates: Reading, writing, appending files, directory operations
 */

console.log('📝 Banking Transaction Logger\n');
console.log('='.repeat(70));

// ============================================
// Transaction Logger Class
// ============================================

class TransactionLogger {
  constructor(logDir = './logs') {
    this.logDir = logDir;
    this.currentLogFile = null;
  }
  
  /**
   * Initialize logging system
   */
  async initialize() {
    console.log('\n🔧 Initializing transaction logger...');
    
    try {
      // Check if logs directory exists
      await fs.access(this.logDir);
      console.log(`   ✅ Log directory exists: ${this.logDir}`);
    } catch (err) {
      // Directory doesn't exist, create it
      console.log(`   📁 Creating log directory: ${this.logDir}`);
      await fs.mkdir(this.logDir, { recursive: true });
      console.log(`   ✅ Directory created`);
    }
    
    // Set current log file (daily rotation)
    const today = new Date().toISOString().split('T')[0];
    this.currentLogFile = path.join(this.logDir, `transactions_${today}.log`);
    
    console.log(`   📄 Current log file: ${path.basename(this.currentLogFile)}`);
  }
  
  /**
   * Log a transaction (append to file)
   */
  async logTransaction(transaction) {
    const logEntry = JSON.stringify({
      ...transaction,
      timestamp: new Date().toISOString()
    }) + '\n';
    
    console.log(`\n✍️  Logging transaction: ${transaction.id}`);
    
    try {
      // Append to file (creates if doesn't exist)
      await fs.appendFile(this.currentLogFile, logEntry, 'utf8');
      console.log(`   ✅ Transaction logged successfully`);
    } catch (err) {
      console.error(`   ❌ Failed to log transaction: ${err.message}`);
      throw err;
    }
  }
  
  /**
   * Log multiple transactions efficiently (using stream)
   */
  async logTransactionsBatch(transactions) {
    console.log(`\n📦 Batch logging ${transactions.length} transactions...`);
    
    const writeStream = createWriteStream(this.currentLogFile, {
      flags: 'a',  // Append mode
      encoding: 'utf8'
    });
    
    return new Promise((resolve, reject) => {
      writeStream.on('error', reject);
      writeStream.on('finish', () => {
        console.log(`   ✅ Batch logged successfully`);
        resolve();
      });
      
      // Write all transactions
      transactions.forEach(txn => {
        const logEntry = JSON.stringify({
          ...txn,
          timestamp: new Date().toISOString()
        }) + '\n';
        writeStream.write(logEntry);
      });
      
      writeStream.end();
    });
  }
  
  /**
   * Read all transactions for a specific date
   */
  async getTransactionsByDate(date) {
    const logFile = path.join(this.logDir, `transactions_${date}.log`);
    
    console.log(`\n📖 Reading transactions for date: ${date}`);
    console.log(`   File: ${path.basename(logFile)}`);
    
    try {
      // Check if file exists
      await fs.access(logFile);
      
      // Read entire file
      const content = await fs.readFile(logFile, 'utf8');
      
      // Parse transactions (one JSON per line)
      const transactions = content
        .trim()
        .split('\n')
        .filter(line => line)
        .map(line => JSON.parse(line));
      
      console.log(`   ✅ Found ${transactions.length} transactions`);
      
      return transactions;
      
    } catch (err) {
      if (err.code === 'ENOENT') {
        console.log(`   ℹ️  No transactions found for this date`);
        return [];
      }
      throw err;
    }
  }
  
  /**
   * Stream large log file (memory efficient)
   */
  async streamTransactions(date, filterFn = null) {
    const logFile = path.join(this.logDir, `transactions_${date}.log`);
    
    console.log(`\n🌊 Streaming transactions from: ${path.basename(logFile)}`);
    
    const readStream = createReadStream(logFile, { encoding: 'utf8' });
    const transactions = [];
    
    // Transform stream to parse JSON lines
    const parseStream = new Transform({
      transform(chunk, encoding, callback) {
        const lines = chunk.toString().split('\n');
        
        lines.forEach(line => {
          if (line.trim()) {
            try {
              const txn = JSON.parse(line);
              
              // Apply filter if provided
              if (!filterFn || filterFn(txn)) {
                this.push(JSON.stringify(txn) + '\n');
              }
            } catch (err) {
              console.error(`   ⚠️  Invalid JSON line: ${line.substring(0, 50)}...`);
            }
          }
        });
        
        callback();
      }
    });
    
    // Process stream
    await pipeline(
      readStream,
      parseStream,
      async function* (source) {
        for await (const chunk of source) {
          const txn = JSON.parse(chunk);
          transactions.push(txn);
          yield chunk;
        }
      }
    );
    
    console.log(`   ✅ Streamed ${transactions.length} transactions`);
    
    return transactions;
  }
  
  /**
   * Get transaction statistics for a date
   */
  async getStatistics(date) {
    console.log(`\n📊 Computing statistics for: ${date}`);
    
    const transactions = await this.getTransactionsByDate(date);
    
    if (transactions.length === 0) {
      return { count: 0, totalAmount: 0, avgAmount: 0 };
    }
    
    const stats = {
      count: transactions.length,
      totalAmount: transactions.reduce((sum, txn) => sum + txn.amount, 0),
      avgAmount: 0,
      maxAmount: Math.max(...transactions.map(txn => txn.amount)),
      minAmount: Math.min(...transactions.map(txn => txn.amount)),
      byType: {}
    };
    
    stats.avgAmount = stats.totalAmount / stats.count;
    
    // Group by type
    transactions.forEach(txn => {
      stats.byType[txn.type] = (stats.byType[txn.type] || 0) + 1;
    });
    
    console.log(`   Transactions: ${stats.count}`);
    console.log(`   Total Amount: $${stats.totalAmount.toFixed(2)}`);
    console.log(`   Average: $${stats.avgAmount.toFixed(2)}`);
    console.log(`   Max: $${stats.maxAmount.toFixed(2)}`);
    console.log(`   Min: $${stats.minAmount.toFixed(2)}`);
    
    return stats;
  }
  
  /**
   * List all log files
   */
  async listLogFiles() {
    console.log(`\n📋 Listing log files in: ${this.logDir}`);
    
    try {
      const files = await fs.readdir(this.logDir);
      
      // Filter for transaction logs
      const logFiles = files.filter(file => file.startsWith('transactions_'));
      
      console.log(`   Found ${logFiles.length} log files:`);
      
      // Get file stats for each
      const fileDetails = await Promise.all(
        logFiles.map(async (file) => {
          const filePath = path.join(this.logDir, file);
          const stats = await fs.stat(filePath);
          
          return {
            name: file,
            size: stats.size,
            created: stats.birthtime,
            modified: stats.mtime
          };
        })
      );
      
      // Sort by date (newest first)
      fileDetails.sort((a, b) => b.modified - a.modified);
      
      fileDetails.forEach((file, index) => {
        console.log(`   ${index + 1}. ${file.name}`);
        console.log(`      Size: ${(file.size / 1024).toFixed(2)} KB`);
        console.log(`      Modified: ${file.modified.toISOString()}`);
      });
      
      return fileDetails;
      
    } catch (err) {
      console.error(`   ❌ Error listing files: ${err.message}`);
      throw err;
    }
  }
  
  /**
   * Archive old log files (compress and move)
   */
  async archiveOldLogs(daysOld = 30) {
    console.log(`\n🗄️  Archiving logs older than ${daysOld} days...`);
    
    const archiveDir = path.join(this.logDir, 'archive');
    
    // Create archive directory if doesn't exist
    try {
      await fs.access(archiveDir);
    } catch (err) {
      await fs.mkdir(archiveDir, { recursive: true });
      console.log(`   ✅ Created archive directory`);
    }
    
    const files = await fs.readdir(this.logDir);
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - daysOld);
    
    let archivedCount = 0;
    
    for (const file of files) {
      if (!file.startsWith('transactions_')) continue;
      
      const filePath = path.join(this.logDir, file);
      const stats = await fs.stat(filePath);
      
      if (stats.mtime < cutoffDate) {
        const archivePath = path.join(archiveDir, file);
        
        // Move file to archive
        await fs.rename(filePath, archivePath);
        
        console.log(`   📦 Archived: ${file}`);
        archivedCount++;
      }
    }
    
    console.log(`   ✅ Archived ${archivedCount} files`);
    
    return archivedCount;
  }
  
  /**
   * Delete logs older than specified days
   */
  async deleteOldLogs(daysOld = 90) {
    console.log(`\n🗑️  Deleting logs older than ${daysOld} days...`);
    
    const files = await fs.readdir(this.logDir);
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - daysOld);
    
    let deletedCount = 0;
    
    for (const file of files) {
      if (!file.startsWith('transactions_')) continue;
      
      const filePath = path.join(this.logDir, file);
      const stats = await fs.stat(filePath);
      
      if (stats.mtime < cutoffDate) {
        await fs.unlink(filePath);
        console.log(`   🗑️  Deleted: ${file}`);
        deletedCount++;
      }
    }
    
    console.log(`   ✅ Deleted ${deletedCount} files`);
    
    return deletedCount;
  }
  
  /**
   * Copy log file (backup)
   */
  async backupLog(date, backupDir = './backups') {
    console.log(`\n💾 Backing up log for: ${date}`);
    
    const sourceFile = path.join(this.logDir, `transactions_${date}.log`);
    
    // Create backup directory
    try {
      await fs.access(backupDir);
    } catch (err) {
      await fs.mkdir(backupDir, { recursive: true });
    }
    
    const backupFile = path.join(
      backupDir,
      `transactions_${date}_backup_${Date.now()}.log`
    );
    
    // Copy file
    await fs.copyFile(sourceFile, backupFile);
    
    console.log(`   ✅ Backup created: ${path.basename(backupFile)}`);
    
    return backupFile;
  }
  
  /**
   * Search transactions by criteria
   */
  async searchTransactions(date, criteria) {
    console.log(`\n🔍 Searching transactions for: ${date}`);
    console.log(`   Criteria:`, JSON.stringify(criteria));
    
    const transactions = await this.getTransactionsByDate(date);
    
    const results = transactions.filter(txn => {
      if (criteria.minAmount && txn.amount < criteria.minAmount) return false;
      if (criteria.maxAmount && txn.amount > criteria.maxAmount) return false;
      if (criteria.type && txn.type !== criteria.type) return false;
      if (criteria.accountId && txn.accountId !== criteria.accountId) return false;
      
      return true;
    });
    
    console.log(`   ✅ Found ${results.length} matching transactions`);
    
    return results;
  }
}

// ============================================
// Test Transaction Logger
// ============================================

async function runTests() {
  const logger = new TransactionLogger('./test_logs');
  
  // Test 1: Initialize
  console.log('\n' + '='.repeat(70));
  console.log('\n📊 Test 1: Initialize Logger\n');
  
  await logger.initialize();
  
  // Test 2: Log single transaction
  console.log('\n' + '='.repeat(70));
  console.log('\n📊 Test 2: Log Single Transaction\n');
  
  await logger.logTransaction({
    id: 'TXN001',
    accountId: 'ACC123',
    type: 'deposit',
    amount: 1000.00,
    description: 'Salary deposit'
  });
  
  // Test 3: Batch log transactions
  console.log('\n' + '='.repeat(70));
  console.log('\n📊 Test 3: Batch Log Transactions\n');
  
  const batchTransactions = [
    { id: 'TXN002', accountId: 'ACC123', type: 'withdrawal', amount: 50.00, description: 'ATM withdrawal' },
    { id: 'TXN003', accountId: 'ACC456', type: 'deposit', amount: 2000.00, description: 'Check deposit' },
    { id: 'TXN004', accountId: 'ACC123', type: 'transfer', amount: 300.00, description: 'Bill payment' },
    { id: 'TXN005', accountId: 'ACC789', type: 'withdrawal', amount: 100.00, description: 'Online purchase' }
  ];
  
  await logger.logTransactionsBatch(batchTransactions);
  
  // Test 4: Read transactions
  console.log('\n' + '='.repeat(70));
  console.log('\n📊 Test 4: Read Transactions\n');
  
  const today = new Date().toISOString().split('T')[0];
  const transactions = await logger.getTransactionsByDate(today);
  
  console.log(`\n   Transactions retrieved:`);
  transactions.forEach((txn, index) => {
    console.log(`   ${index + 1}. ${txn.id}: ${txn.type} - $${txn.amount}`);
  });
  
  // Test 5: Get statistics
  console.log('\n' + '='.repeat(70));
  console.log('\n📊 Test 5: Get Statistics\n');
  
  const stats = await logger.getStatistics(today);
  console.log(`\n   Statistics:`, JSON.stringify(stats, null, 2));
  
  // Test 6: List log files
  console.log('\n' + '='.repeat(70));
  console.log('\n📊 Test 6: List Log Files\n');
  
  await logger.listLogFiles();
  
  // Test 7: Search transactions
  console.log('\n' + '='.repeat(70));
  console.log('\n📊 Test 7: Search Transactions\n');
  
  const searchResults = await logger.searchTransactions(today, {
    minAmount: 100,
    maxAmount: 1000
  });
  
  console.log(`\n   Search results:`);
  searchResults.forEach((txn, index) => {
    console.log(`   ${index + 1}. ${txn.id}: $${txn.amount}`);
  });
  
  // Test 8: Backup log
  console.log('\n' + '='.repeat(70));
  console.log('\n📊 Test 8: Backup Log File\n');
  
  await logger.backupLog(today, './test_backups');
  
  console.log('\n' + '='.repeat(70));
  console.log('\n✅ All file system tests completed!\n');
}

// Run tests
runTests().catch(err => {
  console.error('\n💥 Test error:', err);
  process.exit(1);
});
```

**To test this example:**

```bash
node transaction-logger.js
```

**Expected Output:**

```
📝 Banking Transaction Logger

======================================================================

======================================================================

📊 Test 1: Initialize Logger

🔧 Initializing transaction logger...
   📁 Creating log directory: ./test_logs
   ✅ Directory created
   📄 Current log file: transactions_2025-11-13.log

======================================================================

📊 Test 2: Log Single Transaction

✍️  Logging transaction: TXN001
   ✅ Transaction logged successfully

======================================================================

📊 Test 3: Batch Log Transactions

📦 Batch logging 4 transactions...
   ✅ Batch logged successfully

======================================================================

📊 Test 4: Read Transactions

📖 Reading transactions for date: 2025-11-13
   File: transactions_2025-11-13.log
   ✅ Found 5 transactions

   Transactions retrieved:
   1. TXN001: deposit - $1000
   2. TXN002: withdrawal - $50
   3. TXN003: deposit - $2000
   4. TXN004: transfer - $300
   5. TXN005: withdrawal - $100

======================================================================

📊 Test 5: Get Statistics

📊 Computing statistics for: 2025-11-13
   Transactions: 5
   Total Amount: $3450.00
   Average: $690.00
   Max: $2000.00
   Min: $50.00

   Statistics: {
  "count": 5,
  "totalAmount": 3450,
  "avgAmount": 690,
  "maxAmount": 2000,
  "minAmount": 50,
  "byType": {
    "deposit": 2,
    "withdrawal": 2,
    "transfer": 1
  }
}

======================================================================

📊 Test 6: List Log Files

📋 Listing log files in: ./test_logs
   Found 1 log files:
   1. transactions_2025-11-13.log
      Size: 0.67 KB
      Modified: 2025-11-13T10:30:45.123Z

======================================================================

📊 Test 7: Search Transactions

🔍 Searching transactions for: 2025-11-13
   Criteria: {"minAmount":100,"maxAmount":1000}
   ✅ Found 2 matching transactions

   Search results:
   1. TXN001: $1000
   2. TXN004: $300

======================================================================

📊 Test 8: Backup Log File

💾 Backing up log for: 2025-11-13
   ✅ Backup created: transactions_2025-11-13_backup_1731497445789.log

======================================================================

✅ All file system tests completed!
```

**Key Learnings from Example 1**:

1. **fs.promises API**: Modern, clean async/await syntax
2. **mkdir with recursive**: Creates parent directories automatically
3. **appendFile**: Efficient for logging (creates file if doesn't exist)
4. **readFile**: Loads entire file into memory
5. **readdir + stat**: List files with details
6. **rename**: Move/rename files
7. **copyFile**: Create backups
8. **unlink**: Delete files
9. **Streams**: Memory-efficient for large files

---

## 🏦 Example 2: File Watching and Real-Time Monitoring

This example demonstrates watching files for changes (configuration monitoring).

**File: `config-monitor.js`**

```javascript
import fs from 'fs/promises';
import { watch, watchFile } from 'fs';
import path from 'path';
import { EventEmitter } from 'events';

/**
 * Configuration File Monitor
 * Demonstrates: fs.watch, fs.watchFile, hot-reloading config
 */

console.log('👁️  Configuration Monitor\n');
console.log('='.repeat(70));

// ============================================
// Config Monitor Class
// ============================================

class ConfigMonitor extends EventEmitter {
  constructor(configPath = './config.json') {
    super();
    this.configPath = configPath;
    this.config = null;
    this.watcher = null;
  }
  
  /**
   * Initialize and load config
   */
  async initialize() {
    console.log(`\n🔧 Initializing config monitor...`);
    console.log(`   Config file: ${this.configPath}`);
    
    // Create config file if doesn't exist
    try {
      await fs.access(this.configPath);
      console.log(`   ✅ Config file exists`);
    } catch (err) {
      console.log(`   📝 Creating default config file...`);
      await this.createDefaultConfig();
    }
    
    // Load initial config
    await this.loadConfig();
    
    // Start watching
    this.startWatching();
  }
  
  /**
   * Create default configuration
   */
  async createDefaultConfig() {
    const defaultConfig = {
      database: {
        host: 'localhost',
        port: 5432,
        name: 'banking_db',
        pool: {
          min: 2,
          max: 10
        }
      },
      server: {
        port: 3000,
        host: '0.0.0.0',
        timeout: 30000
      },
      security: {
        jwtSecret: 'change-me-in-production',
        tokenExpiry: '1h',
        maxLoginAttempts: 3
      },
      features: {
        enableTransfers: true,
        enableInternationalPayments: false,
        dailyTransferLimit: 10000
      },
      logging: {
        level: 'info',
        enableFileLogging: true,
        logDirectory: './logs'
      }
    };
    
    await fs.writeFile(
      this.configPath,
      JSON.stringify(defaultConfig, null, 2),
      'utf8'
    );
    
    console.log(`   ✅ Default config created`);
  }
  
  /**
   * Load configuration from file
   */
  async loadConfig() {
    console.log(`\n📖 Loading configuration...`);
    
    try {
      const content = await fs.readFile(this.configPath, 'utf8');
      this.config = JSON.parse(content);
      
      console.log(`   ✅ Configuration loaded`);
      console.log(`   Keys: ${Object.keys(this.config).join(', ')}`);
      
      this.emit('config:loaded', this.config);
      
      return this.config;
      
    } catch (err) {
      console.error(`   ❌ Failed to load config: ${err.message}`);
      throw err;
    }
  }
  
  /**
   * Start watching for file changes
   */
  startWatching() {
    console.log(`\n👁️  Starting file watcher...`);
    
    // Method 1: fs.watch (more efficient, uses OS events)
    this.watcher = watch(this.configPath, async (eventType, filename) => {
      console.log(`\n🔔 File change detected!`);
      console.log(`   Event: ${eventType}`);
      console.log(`   File: ${filename}`);
      
      if (eventType === 'change') {
        // Debounce: wait a bit for write to complete
        await new Promise(resolve => setTimeout(resolve, 100));
        
        try {
          await this.reloadConfig();
        } catch (err) {
          console.error(`   ❌ Failed to reload: ${err.message}`);
          this.emit('config:error', err);
        }
      }
    });
    
    this.watcher.on('error', (err) => {
      console.error(`   ❌ Watcher error: ${err.message}`);
      this.emit('watcher:error', err);
    });
    
    console.log(`   ✅ Watcher started`);
  }
  
  /**
   * Reload configuration (hot-reload)
   */
  async reloadConfig() {
    console.log(`\n🔄 Reloading configuration...`);
    
    const oldConfig = { ...this.config };
    
    await this.loadConfig();
    
    // Compare configs to detect changes
    const changes = this.detectChanges(oldConfig, this.config);
    
    if (changes.length > 0) {
      console.log(`\n   📊 Configuration changes detected:`);
      changes.forEach(change => {
        console.log(`      ${change.path}: ${change.oldValue} → ${change.newValue}`);
      });
      
      this.emit('config:changed', {
        oldConfig,
        newConfig: this.config,
        changes
      });
    } else {
      console.log(`   ℹ️  No configuration changes`);
    }
  }
  
  /**
   * Detect changes between old and new config
   */
  detectChanges(oldObj, newObj, path = '') {
    const changes = [];
    
    // Check all keys in new config
    for (const key in newObj) {
      const currentPath = path ? `${path}.${key}` : key;
      
      if (typeof newObj[key] === 'object' && newObj[key] !== null) {
        // Recursive for nested objects
        if (oldObj[key]) {
          changes.push(...this.detectChanges(oldObj[key], newObj[key], currentPath));
        }
      } else if (oldObj[key] !== newObj[key]) {
        changes.push({
          path: currentPath,
          oldValue: oldObj[key],
          newValue: newObj[key]
        });
      }
    }
    
    return changes;
  }
  
  /**
   * Get configuration value
   */
  get(key) {
    const keys = key.split('.');
    let value = this.config;
    
    for (const k of keys) {
      if (value === undefined) return undefined;
      value = value[k];
    }
    
    return value;
  }
  
  /**
   * Update configuration value and save
   */
  async set(key, value) {
    console.log(`\n✏️  Updating config: ${key} = ${value}`);
    
    const keys = key.split('.');
    let obj = this.config;
    
    // Navigate to parent object
    for (let i = 0; i < keys.length - 1; i++) {
      if (obj[keys[i]] === undefined) {
        obj[keys[i]] = {};
      }
      obj = obj[keys[i]];
    }
    
    // Set value
    obj[keys[keys.length - 1]] = value;
    
    // Save to file
    await fs.writeFile(
      this.configPath,
      JSON.stringify(this.config, null, 2),
      'utf8'
    );
    
    console.log(`   ✅ Config updated and saved`);
  }
  
  /**
   * Stop watching
   */
  stopWatching() {
    if (this.watcher) {
      this.watcher.close();
      console.log(`\n🛑 Watcher stopped`);
    }
  }
  
  /**
   * Validate configuration
   */
  validateConfig() {
    console.log(`\n✔️  Validating configuration...`);
    
    const errors = [];
    
    // Validate database config
    if (!this.config.database || !this.config.database.host) {
      errors.push('database.host is required');
    }
    
    if (this.config.database && this.config.database.port < 1024) {
      errors.push('database.port must be >= 1024');
    }
    
    // Validate server config
    if (!this.config.server || !this.config.server.port) {
      errors.push('server.port is required');
    }
    
    // Validate security
    if (this.config.security && this.config.security.jwtSecret === 'change-me-in-production') {
      errors.push('⚠️  Security warning: JWT secret should be changed in production');
    }
    
    if (errors.length > 0) {
      console.log(`   ❌ Validation errors:`);
      errors.forEach(err => console.log(`      - ${err}`));
      return false;
    }
    
    console.log(`   ✅ Configuration is valid`);
    return true;
  }
}

// ============================================
// Test Config Monitor
// ============================================

async function runTests() {
  const monitor = new ConfigMonitor('./test_config.json');
  
  // Listen to events
  monitor.on('config:loaded', (config) => {
    console.log(`\n📢 Event: Configuration loaded`);
  });
  
  monitor.on('config:changed', ({ changes }) => {
    console.log(`\n📢 Event: Configuration changed (${changes.length} changes)`);
  });
  
  monitor.on('config:error', (err) => {
    console.log(`\n📢 Event: Configuration error - ${err.message}`);
  });
  
  // Test 1: Initialize
  console.log('\n' + '='.repeat(70));
  console.log('\n📊 Test 1: Initialize Monitor\n');
  
  await monitor.initialize();
  
  // Test 2: Validate config
  console.log('\n' + '='.repeat(70));
  console.log('\n📊 Test 2: Validate Configuration\n');
  
  monitor.validateConfig();
  
  // Test 3: Get values
  console.log('\n' + '='.repeat(70));
  console.log('\n📊 Test 3: Get Configuration Values\n');
  
  console.log(`   database.host: ${monitor.get('database.host')}`);
  console.log(`   server.port: ${monitor.get('server.port')}`);
  console.log(`   features.dailyTransferLimit: ${monitor.get('features.dailyTransferLimit')}`);
  
  // Test 4: Update value (triggers file watch)
  console.log('\n' + '='.repeat(70));
  console.log('\n📊 Test 4: Update Configuration Value\n');
  
  await monitor.set('features.dailyTransferLimit', 15000);
  
  // Wait for watcher to detect change
  await new Promise(resolve => setTimeout(resolve, 500));
  
  // Test 5: Multiple updates
  console.log('\n' + '='.repeat(70));
  console.log('\n📊 Test 5: Multiple Updates\n');
  
  await monitor.set('server.port', 4000);
  await new Promise(resolve => setTimeout(resolve, 200));
  
  await monitor.set('features.enableInternationalPayments', true);
  await new Promise(resolve => setTimeout(resolve, 200));
  
  // Test 6: Display final config
  console.log('\n' + '='.repeat(70));
  console.log('\n📊 Test 6: Final Configuration\n');
  
  console.log(JSON.stringify(monitor.config, null, 2));
  
  // Cleanup
  monitor.stopWatching();
  
  // Delete test file
  await fs.unlink('./test_config.json');
  console.log(`\n🗑️  Test config file deleted`);
  
  console.log('\n' + '='.repeat(70));
  console.log('\n✅ All config monitoring tests completed!\n');
}

// Run tests
runTests().catch(err => {
  console.error('\n💥 Test error:', err);
  process.exit(1);
});
```

**Expected Output:**

```
👁️  Configuration Monitor

======================================================================

======================================================================

📊 Test 1: Initialize Monitor

🔧 Initializing config monitor...
   Config file: ./test_config.json
   📝 Creating default config file...
   ✅ Default config created

📖 Loading configuration...
   ✅ Configuration loaded
   Keys: database, server, security, features, logging

📢 Event: Configuration loaded

👁️  Starting file watcher...
   ✅ Watcher started

======================================================================

📊 Test 2: Validate Configuration

✔️  Validating configuration...
   ❌ Validation errors:
      - ⚠️  Security warning: JWT secret should be changed in production

======================================================================

📊 Test 3: Get Configuration Values

   database.host: localhost
   server.port: 3000
   features.dailyTransferLimit: 10000

======================================================================

📊 Test 4: Update Configuration Value

✏️  Updating config: features.dailyTransferLimit = 15000
   ✅ Config updated and saved

🔔 File change detected!
   Event: change
   File: test_config.json

🔄 Reloading configuration...

📖 Loading configuration...
   ✅ Configuration loaded
   Keys: database, server, security, features, logging

   📊 Configuration changes detected:
      features.dailyTransferLimit: 10000 → 15000

📢 Event: Configuration changed (1 changes)

======================================================================

📊 Test 5: Multiple Updates

✏️  Updating config: server.port = 4000
   ✅ Config updated and saved

🔔 File change detected!
   Event: change
   File: test_config.json

🔄 Reloading configuration...

📖 Loading configuration...
   ✅ Configuration loaded
   Keys: database, server, security, features, logging

   📊 Configuration changes detected:
      server.port: 3000 → 4000

📢 Event: Configuration changed (1 changes)

✏️  Updating config: features.enableInternationalPayments = true
   ✅ Config updated and saved

🔔 File change detected!
   Event: change
   File: test_config.json

🔄 Reloading configuration...

📖 Loading configuration...
   ✅ Configuration loaded
   Keys: database, server, security, features, logging

   📊 Configuration changes detected:
      features.enableInternationalPayments: false → true

📢 Event: Configuration changed (1 changes)

======================================================================

📊 Test 6: Final Configuration

{
  "database": {
    "host": "localhost",
    "port": 5432,
    "name": "banking_db",
    "pool": {
      "min": 2,
      "max": 10
    }
  },
  "server": {
    "port": 4000,
    "host": "0.0.0.0",
    "timeout": 30000
  },
  "security": {
    "jwtSecret": "change-me-in-production",
    "tokenExpiry": "1h",
    "maxLoginAttempts": 3
  },
  "features": {
    "enableTransfers": true,
    "enableInternationalPayments": true,
    "dailyTransferLimit": 15000
  },
  "logging": {
    "level": "info",
    "enableFileLogging": true,
    "logDirectory": "./logs"
  }
}

🛑 Watcher stopped

🗑️  Test config file deleted

======================================================================

✅ All config monitoring tests completed!
```

**Key Learnings from Example 2**:

1. **fs.watch**: Efficient file watching using OS events
2. **Hot-reload**: Update configuration without restarting
3. **EventEmitter**: Notify listeners of config changes
4. **Debouncing**: Wait for writes to complete before reading
5. **Change Detection**: Compare old vs new config
6. **Validation**: Ensure config integrity
7. **Nested Access**: Get/set deep object properties

---

## ✅ 10 DO's: File System Best Practices

### 1. ✅ DO use fs.promises for modern async code

```javascript
// ✅ GOOD: Promise-based API
import fs from 'fs/promises';

async function readConfig() {
  try {
    const data = await fs.readFile('config.json', 'utf8');
    return JSON.parse(data);
  } catch (err) {
    console.error('Failed to read config:', err);
    throw err;
  }
}

// ❌ BAD: Callback-based (harder to manage)
import fs from 'fs';

function readConfigCallback(callback) {
  fs.readFile('config.json', 'utf8', (err, data) => {
    if (err) return callback(err);
    callback(null, JSON.parse(data));
  });
}
```

**Why**: Promises with async/await are cleaner and easier to maintain.

---

### 2. ✅ DO use streams for large files

```javascript
// ✅ GOOD: Stream large file
import { createReadStream } from 'fs';
import { createHash } from 'crypto';

function hashLargeFile(filePath) {
  return new Promise((resolve, reject) => {
    const hash = createHash('sha256');
    const stream = createReadStream(filePath);
    
    stream.on('data', chunk => hash.update(chunk));
    stream.on('end', () => resolve(hash.digest('hex')));
    stream.on('error', reject);
  });
}

// ❌ BAD: Load entire file into memory
import fs from 'fs/promises';

async function hashLargeFileBad(filePath) {
  const data = await fs.readFile(filePath);  // May run out of memory
  return createHash('sha256').update(data).digest('hex');
}
```

**Why**: Streams process data in chunks, preventing memory exhaustion.

---

### 3. ✅ DO use mkdir with recursive option

```javascript
// ✅ GOOD: Create nested directories
import fs from 'fs/promises';

await fs.mkdir('logs/2024/november', { recursive: true });

// ❌ BAD: Manual directory creation
await fs.mkdir('logs');
await fs.mkdir('logs/2024');
await fs.mkdir('logs/2024/november');  // Fails if parents don't exist
```

**Why**: `recursive: true` creates all parent directories automatically.

---

### 4. ✅ DO check file existence with fs.access

```javascript
// ✅ GOOD: Use fs.access
import fs from 'fs/promises';

try {
  await fs.access('config.json');
  console.log('File exists');
} catch (err) {
  console.log('File does not exist');
}

// ❌ BAD: Use fs.exists (deprecated)
import fs from 'fs';

fs.exists('config.json', (exists) => {
  console.log(exists ? 'Exists' : 'Does not exist');
});
```

**Why**: `fs.exists` is deprecated and has race condition issues.

---

### 5. ✅ DO handle ENOENT errors gracefully

```javascript
// ✅ GOOD: Check error code
import fs from 'fs/promises';

async function readFileOrDefault(filePath, defaultValue) {
  try {
    return await fs.readFile(filePath, 'utf8');
  } catch (err) {
    if (err.code === 'ENOENT') {
      return defaultValue;  // File doesn't exist
    }
    throw err;  // Other errors
  }
}

// ❌ BAD: Catch all errors the same way
async function readFileBad(filePath) {
  try {
    return await fs.readFile(filePath, 'utf8');
  } catch (err) {
    return '';  // Masks permission errors, disk errors, etc.
  }
}
```

**Why**: Different errors require different handling.

---

### 6. ✅ DO use writeFile with atomic writes

```javascript
// ✅ GOOD: Atomic write (write to temp, then rename)
import fs from 'fs/promises';
import path from 'path';
import { randomBytes } from 'crypto';

async function atomicWriteFile(filePath, data) {
  const tempPath = `${filePath}.${randomBytes(8).toString('hex')}.tmp`;
  
  try {
    await fs.writeFile(tempPath, data, 'utf8');
    await fs.rename(tempPath, filePath);  // Atomic on most systems
  } catch (err) {
    // Clean up temp file if exists
    try {
      await fs.unlink(tempPath);
    } catch {}
    throw err;
  }
}

// ❌ BAD: Direct write (can corrupt on crash)
await fs.writeFile('important.json', data);  // Partial write if crash
```

**Why**: Atomic writes prevent data corruption during crashes.

---

### 7. ✅ DO close file descriptors

```javascript
// ✅ GOOD: Use try-finally to ensure close
import fs from 'fs/promises';

async function readFirstBytes(filePath, count) {
  let fileHandle;
  try {
    fileHandle = await fs.open(filePath, 'r');
    const buffer = Buffer.alloc(count);
    await fileHandle.read(buffer, 0, count, 0);
    return buffer;
  } finally {
    if (fileHandle) {
      await fileHandle.close();  // Always close
    }
  }
}

// ❌ BAD: No close (leaks file descriptors)
async function readFirstBytesBad(filePath, count) {
  const fileHandle = await fs.open(filePath, 'r');
  const buffer = Buffer.alloc(count);
  await fileHandle.read(buffer, 0, count, 0);
  return buffer;  // Never closed!
}
```

**Why**: Leaking file descriptors can exhaust system resources.

---

### 8. ✅ DO use appropriate file modes

```javascript
// ✅ GOOD: Set correct permissions
import fs from 'fs/promises';

// Read-only file
await fs.writeFile('readonly.txt', 'data', { mode: 0o444 });

// Executable script
await fs.writeFile('script.sh', '#!/bin/bash\necho hello', { mode: 0o755 });

// Sensitive file (owner only)
await fs.writeFile('secrets.json', data, { mode: 0o600 });

// ❌ BAD: Default permissions (may be too permissive)
await fs.writeFile('secrets.json', data);  // May be world-readable
```

**Why**: Proper permissions prevent unauthorized access.

---

### 9. ✅ DO use path.join for cross-platform paths

```javascript
// ✅ GOOD: Use path.join
import path from 'path';

const logFile = path.join('logs', '2024', 'transactions.log');
// Works on Windows: logs\2024\transactions.log
// Works on Linux/Mac: logs/2024/transactions.log

// ❌ BAD: Hardcoded separators
const logFileBad = 'logs/2024/transactions.log';  // Fails on Windows
```

**Why**: path.join handles platform-specific separators.

---

### 10. ✅ DO handle file watching errors

```javascript
// ✅ GOOD: Error handling
import { watch } from 'fs';

const watcher = watch('config.json');

watcher.on('change', (eventType) => {
  console.log('File changed:', eventType);
});

watcher.on('error', (err) => {
  console.error('Watcher error:', err);
  // Attempt to restart watcher
});

// ❌ BAD: No error handler
const watcherBad = watch('config.json');
watcherBad.on('change', (eventType) => {
  console.log('File changed');
});
// Crashes if file is deleted or moved
```

**Why**: File watchers can fail in various ways.

---

## ❌ 10 DON'Ts: Common File System Mistakes

### 1. ❌ DON'T use sync methods in servers

```javascript
// ❌ BAD: Blocks all requests
import express from 'express';
import fs from 'fs';

const app = express();

app.get('/data', (req, res) => {
  const data = fs.readFileSync('data.json', 'utf8');  // Blocks server!
  res.json(JSON.parse(data));
});

// ✅ GOOD: Use async
import fs from 'fs/promises';

app.get('/data', async (req, res) => {
  try {
    const data = await fs.readFile('data.json', 'utf8');
    res.json(JSON.parse(data));
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});
```

**Why**: Sync operations block the entire event loop.

---

### 2. ❌ DON'T forget error handling

```javascript
// ❌ BAD: No error handling
import fs from 'fs/promises';

async function readData() {
  const data = await fs.readFile('data.txt', 'utf8');
  return data;  // Crashes if file doesn't exist
}

// ✅ GOOD: Handle errors
async function readDataSafe() {
  try {
    const data = await fs.readFile('data.txt', 'utf8');
    return data;
  } catch (err) {
    console.error('Failed to read file:', err);
    return null;  // Or throw, depending on needs
  }
}
```

**Why**: File operations fail frequently (missing files, permissions, etc.).

---

### 3. ❌ DON'T use deprecated APIs

```javascript
// ❌ BAD: Deprecated methods
import fs from 'fs';

fs.exists('file.txt', (exists) => {  // Deprecated
  if (exists) {
    fs.readFile('file.txt', callback);
  }
});

// ✅ GOOD: Use modern alternatives
import fs from 'fs/promises';

try {
  await fs.access('file.txt');  // Throws if doesn't exist
  const data = await fs.readFile('file.txt', 'utf8');
} catch (err) {
  if (err.code === 'ENOENT') {
    console.log('File not found');
  }
}
```

**Why**: Deprecated APIs will be removed in future versions.

---

### 4. ❌ DON'T read large files entirely into memory

```javascript
// ❌ BAD: Load 1GB file into memory
import fs from 'fs/promises';

async function processLargeFile() {
  const data = await fs.readFile('huge.csv', 'utf8');  // 1GB in memory!
  const lines = data.split('\n');
  return lines.length;
}

// ✅ GOOD: Use streams
import { createReadStream } from 'fs';
import readline from 'readline';

async function processLargeFileStream() {
  let count = 0;
  const stream = createReadStream('huge.csv');
  const rl = readline.createInterface({ input: stream });
  
  for await (const line of rl) {
    count++;
  }
  
  return count;
}
```

**Why**: Large files can exhaust available memory.

---

### 5. ❌ DON'T ignore race conditions

```javascript
// ❌ BAD: Check-then-act race condition
import fs from 'fs/promises';

async function createFileIfNotExists(filePath, data) {
  try {
    await fs.access(filePath);
    // File exists, don't create
  } catch (err) {
    // File doesn't exist, create it
    await fs.writeFile(filePath, data);  // Race: another process may have created it
  }
}

// ✅ GOOD: Use atomic operations
async function createFileSafe(filePath, data) {
  try {
    // 'wx' flag: create file, fail if exists
    await fs.writeFile(filePath, data, { flag: 'wx' });
  } catch (err) {
    if (err.code === 'EEXIST') {
      console.log('File already exists');
    } else {
      throw err;
    }
  }
}
```

**Why**: Files can change between check and action.

---

### 6. ❌ DON'T use relative paths without resolution

```javascript
// ❌ BAD: Relative path (depends on cwd)
import fs from 'fs/promises';

await fs.readFile('./config.json');  // Fails if cwd changes

// ✅ GOOD: Resolve relative to module
import { fileURLToPath } from 'url';
import path from 'path';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);
const configPath = path.join(__dirname, 'config.json');

await fs.readFile(configPath);
```

**Why**: Current working directory can change at runtime.

---

### 7. ❌ DON'T forget to specify encoding

```javascript
// ❌ BAD: Returns Buffer instead of string
import fs from 'fs/promises';

const data = await fs.readFile('text.txt');
console.log(data);  // <Buffer 48 65 6c 6c 6f>

// ✅ GOOD: Specify encoding
const text = await fs.readFile('text.txt', 'utf8');
console.log(text);  // "Hello"
```

**Why**: Without encoding, you get a Buffer (binary data).

---

### 8. ❌ DON'T delete files without confirmation

```javascript
// ❌ BAD: No confirmation
import fs from 'fs/promises';

async function cleanup() {
  await fs.unlink('important-data.json');  // Gone forever!
}

// ✅ GOOD: Check before delete
async function cleanupSafe(filePath) {
  try {
    const stats = await fs.stat(filePath);
    console.log(`Delete ${filePath} (${stats.size} bytes)?`);
    
    // In production: require explicit confirmation
    const confirmed = await getUserConfirmation();
    
    if (confirmed) {
      await fs.unlink(filePath);
      console.log('File deleted');
    }
  } catch (err) {
    console.error('Failed to delete:', err);
  }
}
```

**Why**: Accidental deletions can cause data loss.

---

### 9. ❌ DON'T nest callbacks (callback hell)

```javascript
// ❌ BAD: Callback hell
import fs from 'fs';

fs.readFile('config.json', 'utf8', (err, data) => {
  if (err) return console.error(err);
  
  const config = JSON.parse(data);
  
  fs.readFile(config.dataFile, 'utf8', (err, data) => {
    if (err) return console.error(err);
    
    const users = JSON.parse(data);
    
    fs.writeFile('output.json', JSON.stringify(users), (err) => {
      if (err) return console.error(err);
      console.log('Done');
    });
  });
});

// ✅ GOOD: Use async/await
import fs from 'fs/promises';

async function processData() {
  try {
    const configData = await fs.readFile('config.json', 'utf8');
    const config = JSON.parse(configData);
    
    const userData = await fs.readFile(config.dataFile, 'utf8');
    const users = JSON.parse(userData);
    
    await fs.writeFile('output.json', JSON.stringify(users));
    console.log('Done');
  } catch (err) {
    console.error('Error:', err);
  }
}
```

**Why**: Nested callbacks are hard to read and maintain.

---

### 10. ❌ DON'T mix sync and async operations

```javascript
// ❌ BAD: Mixing sync and async
import fs from 'fs';
import fsp from 'fs/promises';

async function processFiles() {
  const config = fs.readFileSync('config.json', 'utf8');  // Sync!
  const data = await fsp.readFile('data.json', 'utf8');    // Async!
  // Inconsistent and confusing
}

// ✅ GOOD: Consistent async
import fs from 'fs/promises';

async function processFilesAsync() {
  const config = await fs.readFile('config.json', 'utf8');
  const data = await fs.readFile('data.json', 'utf8');
}
```

**Why**: Mixing creates confusion and potential bugs.

---

## 🎓 Key Takeaways

### File System API Comparison

| Operation | Sync (Blocking) | Callback (Async) | Promise (Modern) |
|-----------|----------------|------------------|------------------|
| **Read file** | `fs.readFileSync()` | `fs.readFile(cb)` | `fs.promises.readFile()` |
| **Write file** | `fs.writeFileSync()` | `fs.writeFile(cb)` | `fs.promises.writeFile()` |
| **Append** | `fs.appendFileSync()` | `fs.appendFile(cb)` | `fs.promises.appendFile()` |
| **Delete** | `fs.unlinkSync()` | `fs.unlink(cb)` | `fs.promises.unlink()` |
| **List dir** | `fs.readdirSync()` | `fs.readdir(cb)` | `fs.promises.readdir()` |
| **Make dir** | `fs.mkdirSync()` | `fs.mkdir(cb)` | `fs.promises.mkdir()` |
| **File stats** | `fs.statSync()` | `fs.stat(cb)` | `fs.promises.stat()` |

### When to Use Each API

| Scenario | Use | Example |
|----------|-----|---------|
| **CLI tool** | Sync | Configuration loading at startup |
| **Web server** | Async (promises) | All request handlers |
| **Large files** | Streams | CSV processing, video encoding |
| **Configuration** | File watch | Hot-reload server config |

### Common File Flags

| Flag | Mode | Behavior |
|------|------|----------|
| **r** | Read | File must exist |
| **w** | Write | Truncate/create file |
| **a** | Append | Create if missing, append to end |
| **wx** | Exclusive write | Create, fail if exists |
| **r+** | Read/write | File must exist |
| **w+** | Read/write | Truncate/create |

### Error Codes Reference

| Code | Meaning | Common Cause |
|------|---------|--------------|
| **ENOENT** | No such file | File doesn't exist |
| **EACCES** | Permission denied | No read/write permission |
| **EEXIST** | File exists | Creating file that exists (with 'wx') |
| **EISDIR** | Is a directory | Trying to read directory as file |
| **ENOTDIR** | Not a directory | Parent is not a directory |
| **EMFILE** | Too many open files | File descriptor limit reached |

### File Watching Comparison

| Method | Efficiency | Reliability | Use Case |
|--------|-----------|-------------|----------|
| **fs.watch** | ⚡⚡⚡ High (OS events) | ⚠️ Platform dependent | Real-time monitoring |
| **fs.watchFile** | 🐌 Low (polling) | ✅ Consistent | Cross-platform compatibility |

### Common Interview Questions

**Q1: Difference between sync and async fs methods?**
- **Sync**: Blocks event loop until operation completes
- **Async**: Non-blocking, uses callbacks/promises
- **When to use sync**: Only at application startup or in CLI tools
- **When to use async**: Always in servers/web apps

**Q2: When to use readFile vs createReadStream?**
- **readFile**: Small files (<100MB), need entire content at once
- **createReadStream**: Large files, process in chunks, memory efficient
- Example: Reading 1GB CSV file should use stream

**Q3: How to prevent race conditions in file operations?**
- Use atomic operations: `fs.writeFile()` with 'wx' flag
- Use file locking mechanisms
- Avoid check-then-act patterns

**Q4: What is the 'wx' file flag?**
- Opens file for writing
- Fails if file already exists
- Atomic operation (no race condition)
- Useful for exclusive file creation

**Q5: How to handle ENOENT errors?**
```javascript
try {
  await fs.readFile('file.txt');
} catch (err) {
  if (err.code === 'ENOENT') {
    // File doesn't exist - handle gracefully
    return defaultValue;
  }
  throw err;  // Other errors
}
```

**Q6: Best way to watch files for changes?**
- Use `fs.watch()` for efficiency (OS events)
- Add debouncing to avoid multiple triggers
- Always add error handler
- Consider using libraries like `chokidar` for production

**Q7: How to ensure file descriptor cleanup?**
```javascript
let fileHandle;
try {
  fileHandle = await fs.open('file.txt');
  // Use fileHandle
} finally {
  if (fileHandle) await fileHandle.close();
}
```

### Real-World Banking File Operations

```
Banking Application File Usage:
├── Transaction Logs
│   ├── Daily log files (append)
│   ├── Log rotation (rename)
│   └── Archive old logs (move/compress)
├── Configuration
│   ├── Hot-reload (fs.watch)
│   ├── Validation on change
│   └── Rollback on error
├── Reports
│   ├── PDF generation (write)
│   ├── CSV exports (stream)
│   └── Scheduled cleanup (unlink)
├── Audit Trail
│   ├── Immutable logs (append-only)
│   ├── Tamper detection (checksums)
│   └── Compliance retention (archive)
└── Backups
    ├── Daily snapshots (copy)
    ├── Compression (stream)
    └── Remote upload (read stream)
```

### Performance Tips

1. **Use streams for large files** (>10MB)
2. **Batch writes** instead of many small writes
3. **Use append mode** for logs (`'a'` flag)
4. **Cache file stats** instead of repeated `stat()` calls
5. **Use `fs.promises`** for cleaner async code
6. **Set appropriate buffer sizes** for streams
7. **Implement retry logic** for transient errors
8. **Use write-through caching** for frequently accessed files

### Security Checklist

- [ ] Validate file paths (prevent directory traversal)
- [ ] Set restrictive file permissions (0o600 for sensitive)
- [ ] Never trust user-provided file names
- [ ] Sanitize file paths before use
- [ ] Use absolute paths when possible
- [ ] Implement file size limits
- [ ] Validate file types/extensions
- [ ] Audit file access (who, when, what)
- [ ] Encrypt sensitive files at rest
- [ ] Implement proper cleanup (temp files)

---

## 🎯 Conclusion

Mastering the File System module is essential for Node.js development:

1. **Always use async** in servers (fs.promises or callbacks)
2. **Use streams** for large files (memory efficient)
3. **Handle errors** properly (check error codes)
4. **Watch files** for configuration hot-reloading
5. **Atomic operations** prevent race conditions
6. **Proper permissions** ensure security
7. **Path.join** for cross-platform compatibility

**Banking Applications**:
- Transaction logs with daily rotation
- Configuration hot-reloading
- Audit trails for compliance
- Report generation (PDF/CSV)
- Secure backup systems
- File-based queuing

Understanding file operations enables building robust, scalable banking systems!

---

**Next Topics to Explore**:
- HTTP Module and networking
- Express.js framework
- Timers and scheduling
- Process management

---

