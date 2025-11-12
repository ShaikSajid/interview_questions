# JavaScript Interview Questions (Q10-Q12): Memory Management & Performance

## Q10: Explain memory management, garbage collection, and memory leaks in JavaScript

**Answer:**

JavaScript uses automatic garbage collection, but understanding memory management is crucial for building performant banking applications.

**Memory Lifecycle:**
1. Allocation - Memory is allocated when variables are created
2. Use - Reading and writing to allocated memory
3. Release - Memory is freed when no longer needed (garbage collection)

**Banking Transaction Memory Management:**

```javascript
// Memory Leak Examples and Solutions
class BankingMemoryManagement {
    
    // BAD: Memory Leak - Event listeners not cleaned up
    badEventListenerExample() {
        const button = document.getElementById('transferButton');
        
        // This creates a new listener every time, never removed
        button.addEventListener('click', function() {
            console.log('Transfer initiated');
        });
        
        // Memory leak: listeners accumulate
    }
    
    // GOOD: Properly cleanup event listeners
    goodEventListenerExample() {
        const button = document.getElementById('transferButton');
        
        const handleTransfer = function() {
            console.log('Transfer initiated');
        };
        
        button.addEventListener('click', handleTransfer);
        
        // Cleanup function
        return () => {
            button.removeEventListener('click', handleTransfer);
        };
    }
    
    // BAD: Memory Leak - Detached DOM nodes
    badDOMManipulation() {
        const transactions = [];
        
        for (let i = 0; i < 1000; i++) {
            const row = document.createElement('div');
            row.innerHTML = `Transaction ${i}`;
            transactions.push(row); // Keeps reference even if removed from DOM
        }
        
        // Memory leak: DOM nodes retained in array
        return transactions;
    }
    
    // GOOD: Clear references when done
    goodDOMManipulation() {
        const container = document.getElementById('transactions');
        
        for (let i = 0; i < 1000; i++) {
            const row = document.createElement('div');
            row.innerHTML = `Transaction ${i}`;
            container.appendChild(row);
        }
        
        // Clear all when done
        return () => {
            while (container.firstChild) {
                container.removeChild(container.firstChild);
            }
        };
    }
}

// Transaction Processing with Memory Optimization
class TransactionProcessor {
    constructor() {
        this.transactions = [];
        this.maxCacheSize = 1000;
        this.processedCount = 0;
    }
    
    // Process large transaction batches efficiently
    async processBatch(transactions) {
        console.log(`Processing ${transactions.length} transactions...`);
        
        // Process in chunks to avoid memory spikes
        const chunkSize = 100;
        
        for (let i = 0; i < transactions.length; i += chunkSize) {
            const chunk = transactions.slice(i, i + chunkSize);
            
            await this.processChunk(chunk);
            
            // Allow garbage collection between chunks
            if (i % 500 === 0) {
                await this.forceGarbageCollection();
            }
        }
        
        console.log('Batch processing complete');
    }
    
    async processChunk(chunk) {
        for (const txn of chunk) {
            await this.processTransaction(txn);
            this.processedCount++;
        }
        
        // Clear processed chunk reference
        chunk.length = 0;
    }
    
    async processTransaction(transaction) {
        // Simulate processing
        return new Promise(resolve => {
            setTimeout(() => {
                // Process transaction
                resolve();
            }, 1);
        });
    }
    
    async forceGarbageCollection() {
        // Give time for GC to run
        await new Promise(resolve => setTimeout(resolve, 0));
    }
    
    // Implement LRU cache to limit memory usage
    addToCache(key, value) {
        if (this.transactions.length >= this.maxCacheSize) {
            // Remove oldest entry (LRU)
            this.transactions.shift();
        }
        
        this.transactions.push({ key, value, timestamp: Date.now() });
    }
    
    clearOldCache(maxAge = 300000) {
        const now = Date.now();
        this.transactions = this.transactions.filter(
            item => now - item.timestamp < maxAge
        );
        
        console.log(`Cleared old cache entries. Current size: ${this.transactions.length}`);
    }
}

// Weak References for Cache Management
class AccountCache {
    constructor() {
        // Use WeakMap to allow garbage collection
        this.cache = new WeakMap();
        this.metadata = new Map();
    }
    
    // Cache account object (automatically cleaned up when no other references)
    set(accountObject, data) {
        this.cache.set(accountObject, data);
        this.metadata.set(accountObject.accountNumber, {
            cachedAt: Date.now(),
            hits: 0
        });
    }
    
    get(accountObject) {
        if (this.cache.has(accountObject)) {
            const meta = this.metadata.get(accountObject.accountNumber);
            if (meta) {
                meta.hits++;
            }
            return this.cache.get(accountObject);
        }
        return null;
    }
    
    // Metadata cleanup (WeakMap handles actual cache cleanup)
    cleanupMetadata() {
        const now = Date.now();
        const maxAge = 3600000; // 1 hour
        
        for (const [accountNumber, meta] of this.metadata.entries()) {
            if (now - meta.cachedAt > maxAge) {
                this.metadata.delete(accountNumber);
            }
        }
    }
}

// Memory-efficient streaming for large datasets
class TransactionStreamProcessor {
    constructor() {
        this.batchSize = 1000;
    }
    
    // Process transactions in streaming fashion
    async *streamTransactions(dataSource) {
        let offset = 0;
        
        while (true) {
            // Fetch batch
            const batch = await dataSource.fetchBatch(offset, this.batchSize);
            
            if (batch.length === 0) break;
            
            // Yield transactions one by one
            for (const transaction of batch) {
                yield transaction;
            }
            
            offset += batch.length;
            
            // Clear batch to free memory
            batch.length = 0;
        }
    }
    
    async processStream(dataSource) {
        let processed = 0;
        
        for await (const transaction of this.streamTransactions(dataSource)) {
            await this.processTransaction(transaction);
            processed++;
            
            if (processed % 1000 === 0) {
                console.log(`Processed ${processed} transactions`);
            }
        }
        
        return processed;
    }
    
    async processTransaction(transaction) {
        // Process transaction
        return new Promise(resolve => setTimeout(resolve, 1));
    }
}

// Memory profiling helper
class MemoryProfiler {
    constructor() {
        this.snapshots = [];
    }
    
    takeSnapshot(label) {
        if (typeof process !== 'undefined' && process.memoryUsage) {
            const usage = process.memoryUsage();
            
            this.snapshots.push({
                label,
                timestamp: Date.now(),
                heapUsed: usage.heapUsed,
                heapTotal: usage.heapTotal,
                external: usage.external,
                rss: usage.rss
            });
            
            console.log(`\n[Memory Snapshot: ${label}]`);
            console.log(`Heap Used: ${(usage.heapUsed / 1024 / 1024).toFixed(2)} MB`);
            console.log(`Heap Total: ${(usage.heapTotal / 1024 / 1024).toFixed(2)} MB`);
            console.log(`RSS: ${(usage.rss / 1024 / 1024).toFixed(2)} MB`);
        }
    }
    
    getReport() {
        if (this.snapshots.length < 2) {
            return 'Not enough snapshots for comparison';
        }
        
        const first = this.snapshots[0];
        const last = this.snapshots[this.snapshots.length - 1];
        
        const heapDiff = last.heapUsed - first.heapUsed;
        const rssDiff = last.rss - first.rss;
        
        return {
            duration: last.timestamp - first.timestamp,
            heapGrowth: (heapDiff / 1024 / 1024).toFixed(2) + ' MB',
            rssGrowth: (rssDiff / 1024 / 1024).toFixed(2) + ' MB',
            snapshots: this.snapshots.length
        };
    }
}

// Usage Example
async function demonstrateMemoryManagement() {
    console.log('=== Memory Management Demo ===\n');
    
    const profiler = new MemoryProfiler();
    profiler.takeSnapshot('Start');
    
    // Create large transaction dataset
    const transactions = [];
    for (let i = 0; i < 10000; i++) {
        transactions.push({
            id: `TXN${i}`,
            amount: Math.random() * 10000,
            timestamp: new Date(),
            accountId: `ACC${Math.floor(Math.random() * 100)}`
        });
    }
    
    profiler.takeSnapshot('After creating 10k transactions');
    
    // Process transactions
    const processor = new TransactionProcessor();
    await processor.processBatch(transactions);
    
    profiler.takeSnapshot('After processing');
    
    // Clear references
    transactions.length = 0;
    
    profiler.takeSnapshot('After cleanup');
    
    console.log('\n=== Memory Report ===');
    console.log(profiler.getReport());
}

// Common Memory Leak Patterns in Banking Apps
class MemoryLeakPatterns {
    
    // Pattern 1: Timers not cleared
    leakyTimer() {
        const accountData = { /* large object */ };
        
        setInterval(() => {
            console.log(accountData.balance);
        }, 1000);
        
        // Memory leak: timer keeps accountData alive forever
    }
    
    // Fixed: Clear timer
    fixedTimer() {
        const accountData = { /* large object */ };
        
        const timerId = setInterval(() => {
            console.log(accountData.balance);
        }, 1000);
        
        // Cleanup
        return () => clearInterval(timerId);
    }
    
    // Pattern 2: Closure capturing large objects
    leakyClosure() {
        const largeArray = new Array(1000000).fill('transaction data');
        
        return function() {
            // Closure keeps entire largeArray in memory
            console.log(largeArray.length);
        };
    }
    
    // Fixed: Only capture what's needed
    fixedClosure() {
        const largeArray = new Array(1000000).fill('transaction data');
        const arrayLength = largeArray.length;
        
        return function() {
            // Only captures arrayLength, not entire array
            console.log(arrayLength);
        };
    }
    
    // Pattern 3: Global variables
    leakyGlobal() {
        // Accidentally creates global variable
        accountCache = {}; // Missing 'const', 'let', or 'var'
        
        // Memory leak: never garbage collected
    }
    
    // Fixed: Use proper scope
    fixedGlobal() {
        const accountCache = {};
        
        // Properly scoped, can be garbage collected
        return accountCache;
    }
}

// demonstrateMemoryManagement();
```

---

## Q11: Explain JavaScript performance optimization techniques for banking applications

**Answer:**

Performance optimization is critical for banking applications handling high transaction volumes.

**Performance Optimization Strategies:**

```javascript
// 1. Debouncing and Throttling for Account Search
class AccountSearchOptimizer {
    
    // Debounce: Execute after user stops typing
    debounce(func, delay = 300) {
        let timeoutId;
        
        return function(...args) {
            clearTimeout(timeoutId);
            
            timeoutId = setTimeout(() => {
                func.apply(this, args);
            }, delay);
        };
    }
    
    // Throttle: Execute at most once per time period
    throttle(func, limit = 1000) {
        let inThrottle;
        
        return function(...args) {
            if (!inThrottle) {
                func.apply(this, args);
                inThrottle = true;
                
                setTimeout(() => {
                    inThrottle = false;
                }, limit);
            }
        };
    }
    
    setupAccountSearch() {
        const searchInput = document.getElementById('accountSearch');
        
        // Debounced API call
        const debouncedSearch = this.debounce(async (query) => {
            console.log(`Searching for: ${query}`);
            const results = await this.searchAccounts(query);
            this.displayResults(results);
        }, 300);
        
        searchInput.addEventListener('input', (e) => {
            debouncedSearch(e.target.value);
        });
    }
    
    async searchAccounts(query) {
        // Simulated API call
        return [
            { accountNumber: 'ACC001', holder: 'Ahmed' },
            { accountNumber: 'ACC002', holder: 'Fatima' }
        ];
    }
    
    displayResults(results) {
        console.log('Results:', results);
    }
}

// 2. Virtual Scrolling for Large Transaction Lists
class VirtualScrollTransactionList {
    constructor(container, data, rowHeight = 50) {
        this.container = container;
        this.data = data;
        this.rowHeight = rowHeight;
        this.visibleRows = Math.ceil(container.clientHeight / rowHeight);
        this.bufferRows = 5;
        
        this.setupVirtualScroll();
    }
    
    setupVirtualScroll() {
        // Create viewport
        const viewport = document.createElement('div');
        viewport.style.height = `${this.data.length * this.rowHeight}px`;
        viewport.style.position = 'relative';
        
        // Visible container
        const visibleContainer = document.createElement('div');
        visibleContainer.style.position = 'absolute';
        visibleContainer.style.top = '0';
        visibleContainer.style.width = '100%';
        
        viewport.appendChild(visibleContainer);
        this.container.appendChild(viewport);
        
        // Handle scroll
        this.container.addEventListener('scroll', () => {
            this.renderVisibleRows(visibleContainer);
        });
        
        // Initial render
        this.renderVisibleRows(visibleContainer);
    }
    
    renderVisibleRows(container) {
        const scrollTop = this.container.scrollTop;
        const startIndex = Math.floor(scrollTop / this.rowHeight);
        const endIndex = Math.min(
            startIndex + this.visibleRows + this.bufferRows,
            this.data.length
        );
        
        // Clear existing rows
        container.innerHTML = '';
        
        // Render only visible rows
        for (let i = startIndex; i < endIndex; i++) {
            const row = this.createRow(this.data[i], i);
            container.appendChild(row);
        }
        
        // Adjust position
        container.style.transform = `translateY(${startIndex * this.rowHeight}px)`;
    }
    
    createRow(transaction, index) {
        const row = document.createElement('div');
        row.style.height = `${this.rowHeight}px`;
        row.innerHTML = `
            <div>${transaction.id}: ${transaction.type} AED ${transaction.amount}</div>
        `;
        return row;
    }
}

// 3. Memoization for Expensive Calculations
class InterestCalculator {
    constructor() {
        this.cache = new Map();
    }
    
    // Memoized calculation
    calculateCompoundInterest(principal, rate, years, frequency = 12) {
        const key = `${principal}-${rate}-${years}-${frequency}`;
        
        if (this.cache.has(key)) {
            console.log('Cache hit');
            return this.cache.get(key);
        }
        
        console.log('Computing...');
        
        // Expensive calculation
        const result = principal * Math.pow(
            1 + rate / frequency,
            frequency * years
        );
        
        this.cache.set(key, result);
        return result;
    }
    
    // Generic memoization helper
    memoize(fn) {
        const cache = new Map();
        
        return function(...args) {
            const key = JSON.stringify(args);
            
            if (cache.has(key)) {
                return cache.get(key);
            }
            
            const result = fn.apply(this, args);
            cache.set(key, result);
            return result;
        };
    }
}

// 4. Web Workers for Heavy Computations
class TransactionAnalyzer {
    
    analyzeWithWorker(transactions) {
        return new Promise((resolve, reject) => {
            // Create worker
            const workerCode = `
                self.addEventListener('message', (e) => {
                    const transactions = e.data;
                    
                    // Heavy computation
                    const analysis = {
                        total: transactions.length,
                        totalAmount: 0,
                        byCategory: {},
                        averages: {}
                    };
                    
                    transactions.forEach(txn => {
                        analysis.totalAmount += txn.amount;
                        
                        if (!analysis.byCategory[txn.category]) {
                            analysis.byCategory[txn.category] = {
                                count: 0,
                                total: 0
                            };
                        }
                        
                        analysis.byCategory[txn.category].count++;
                        analysis.byCategory[txn.category].total += txn.amount;
                    });
                    
                    // Calculate averages
                    for (const [category, data] of Object.entries(analysis.byCategory)) {
                        analysis.averages[category] = data.total / data.count;
                    }
                    
                    self.postMessage(analysis);
                });
            `;
            
            const blob = new Blob([workerCode], { type: 'application/javascript' });
            const worker = new Worker(URL.createObjectURL(blob));
            
            worker.addEventListener('message', (e) => {
                resolve(e.data);
                worker.terminate();
            });
            
            worker.addEventListener('error', reject);
            
            worker.postMessage(transactions);
        });
    }
}

// 5. Request Batching and Caching
class AccountDataService {
    constructor() {
        this.cache = new Map();
        this.pendingRequests = new Map();
        this.batchQueue = [];
        this.batchTimeout = null;
    }
    
    // Batch multiple requests
    async getAccount(accountId) {
        // Check cache first
        if (this.cache.has(accountId)) {
            console.log(`Cache hit: ${accountId}`);
            return this.cache.get(accountId);
        }
        
        // Check if request already pending
        if (this.pendingRequests.has(accountId)) {
            console.log(`Joining pending request: ${accountId}`);
            return this.pendingRequests.get(accountId);
        }
        
        // Create promise for this request
        const promise = new Promise((resolve, reject) => {
            this.batchQueue.push({ accountId, resolve, reject });
            
            // Schedule batch execution
            if (this.batchTimeout) {
                clearTimeout(this.batchTimeout);
            }
            
            this.batchTimeout = setTimeout(() => {
                this.executeBatch();
            }, 50); // Batch window: 50ms
        });
        
        this.pendingRequests.set(accountId, promise);
        return promise;
    }
    
    async executeBatch() {
        if (this.batchQueue.length === 0) return;
        
        const batch = [...this.batchQueue];
        this.batchQueue = [];
        
        const accountIds = batch.map(req => req.accountId);
        console.log(`Executing batch request for ${accountIds.length} accounts`);
        
        try {
            // Single API call for all accounts
            const results = await this.fetchMultipleAccounts(accountIds);
            
            // Resolve individual promises
            batch.forEach(req => {
                const data = results[req.accountId];
                this.cache.set(req.accountId, data);
                this.pendingRequests.delete(req.accountId);
                req.resolve(data);
            });
            
        } catch (error) {
            batch.forEach(req => {
                this.pendingRequests.delete(req.accountId);
                req.reject(error);
            });
        }
    }
    
    async fetchMultipleAccounts(accountIds) {
        // Simulated batch API call
        const results = {};
        accountIds.forEach(id => {
            results[id] = {
                accountNumber: id,
                balance: Math.random() * 100000,
                type: 'SAVINGS'
            };
        });
        return results;
    }
}

// 6. Lazy Loading and Code Splitting
class ModuleLoader {
    
    async loadTransactionModule() {
        console.log('Loading transaction module...');
        
        // Dynamic import (code splitting)
        const module = await import('./transactions.js').catch(() => {
            console.log('Module not found, using fallback');
            return {
                processTransaction: (txn) => console.log('Processing:', txn)
            };
        });
        
        return module;
    }
    
    async loadOnDemand(feature) {
        const modules = {
            'reports': () => import('./reports.js'),
            'analytics': () => import('./analytics.js'),
            'settings': () => import('./settings.js')
        };
        
        if (modules[feature]) {
            return await modules[feature]();
        }
        
        throw new Error(`Unknown feature: ${feature}`);
    }
}

// 7. Performance Monitoring
class PerformanceMonitor {
    
    measureExecutionTime(label, fn) {
        const start = performance.now();
        
        const result = fn();
        
        const end = performance.now();
        const duration = end - start;
        
        console.log(`${label}: ${duration.toFixed(2)}ms`);
        
        // Log slow operations
        if (duration > 100) {
            console.warn(`Slow operation detected: ${label}`);
        }
        
        return result;
    }
    
    async measureAsyncTime(label, asyncFn) {
        const start = performance.now();
        
        const result = await asyncFn();
        
        const end = performance.now();
        console.log(`${label}: ${(end - start).toFixed(2)}ms`);
        
        return result;
    }
    
    // Mark performance checkpoints
    markCheckpoint(name) {
        performance.mark(name);
    }
    
    measureBetweenCheckpoints(startMark, endMark, measureName) {
        performance.measure(measureName, startMark, endMark);
        
        const measures = performance.getEntriesByName(measureName);
        if (measures.length > 0) {
            console.log(`${measureName}: ${measures[0].duration.toFixed(2)}ms`);
        }
    }
}

// Usage Examples
async function demonstratePerformanceOptimizations() {
    console.log('=== Performance Optimization Demo ===\n');
    
    const monitor = new PerformanceMonitor();
    
    // 1. Memoization
    monitor.markCheckpoint('memoization-start');
    const calculator = new InterestCalculator();
    
    calculator.calculateCompoundInterest(10000, 0.05, 5, 12); // First call
    calculator.calculateCompoundInterest(10000, 0.05, 5, 12); // Cached
    
    monitor.markCheckpoint('memoization-end');
    monitor.measureBetweenCheckpoints('memoization-start', 'memoization-end', 'Memoization');
    
    // 2. Request Batching
    console.log('\n--- Request Batching ---');
    const dataService = new AccountDataService();
    
    // Multiple requests executed as single batch
    const promises = [
        dataService.getAccount('ACC001'),
        dataService.getAccount('ACC002'),
        dataService.getAccount('ACC003')
    ];
    
    await Promise.all(promises);
    
    // 3. Debouncing
    console.log('\n--- Debouncing ---');
    const searchOptimizer = new AccountSearchOptimizer();
    const debouncedFunc = searchOptimizer.debounce(() => {
        console.log('Search executed');
    }, 300);
    
    // Only last call executes
    debouncedFunc();
    debouncedFunc();
    debouncedFunc();
}

// demonstratePerformanceOptimizations();
```

---

## Q12: Explain error handling patterns and best practices for banking systems

**Answer:**

Robust error handling is critical for banking applications to ensure data integrity and user trust.

**Comprehensive Error Handling:**

```javascript
// Custom Error Classes for Banking Domain
class BankingError extends Error {
    constructor(message, code, details = {}) {
        super(message);
        this.name = this.constructor.name;
        this.code = code;
        this.details = details;
        this.timestamp = new Date().toISOString();
        Error.captureStackTrace(this, this.constructor);
    }
    
    toJSON() {
        return {
            name: this.name,
            message: this.message,
            code: this.code,
            details: this.details,
            timestamp: this.timestamp
        };
    }
}

class InsufficientFundsError extends BankingError {
    constructor(availableBalance, requestedAmount) {
        super(
            `Insufficient funds. Available: AED ${availableBalance}, Requested: AED ${requestedAmount}`,
            'INSUFFICIENT_FUNDS',
            { availableBalance, requestedAmount, shortfall: requestedAmount - availableBalance }
        );
    }
}

class InvalidAccountError extends BankingError {
    constructor(accountNumber) {
        super(
            `Account not found: ${accountNumber}`,
            'INVALID_ACCOUNT',
            { accountNumber }
        );
    }
}

class TransactionLimitExceededError extends BankingError {
    constructor(limit, attempted) {
        super(
            `Transaction limit exceeded. Limit: AED ${limit}, Attempted: AED ${attempted}`,
            'LIMIT_EXCEEDED',
            { limit, attempted }
        );
    }
}

class FraudDetectionError extends BankingError {
    constructor(reason, transactionId) {
        super(
            `Transaction flagged for fraud: ${reason}`,
            'FRAUD_DETECTED',
            { reason, transactionId }
        );
    }
}

class NetworkError extends BankingError {
    constructor(originalError) {
        super(
            'Network request failed',
            'NETWORK_ERROR',
            { originalMessage: originalError.message }
        );
    }
}

// Error Handler Service
class ErrorHandlerService {
    constructor() {
        this.errorLog = [];
        this.errorListeners = [];
    }
    
    // Handle errors with different strategies
    handleError(error, context = {}) {
        // Log error
        this.logError(error, context);
        
        // Categorize error
        const category = this.categorizeError(error);
        
        // Apply recovery strategy
        const recovery = this.getRecoveryStrategy(category);
        
        // Notify listeners
        this.notifyListeners(error, context);
        
        return {
            error,
            category,
            recovery,
            context
        };
    }
    
    logError(error, context) {
        const logEntry = {
            timestamp: new Date().toISOString(),
            error: error instanceof Error ? {
                name: error.name,
                message: error.message,
                stack: error.stack,
                code: error.code
            } : error,
            context,
            userId: context.userId,
            sessionId: context.sessionId
        };
        
        this.errorLog.push(logEntry);
        
        // In production, send to logging service
        console.error('Error logged:', logEntry);
    }
    
    categorizeError(error) {
        if (error instanceof InsufficientFundsError) return 'BUSINESS_RULE';
        if (error instanceof InvalidAccountError) return 'VALIDATION';
        if (error instanceof TransactionLimitExceededError) return 'BUSINESS_RULE';
        if (error instanceof FraudDetectionError) return 'SECURITY';
        if (error instanceof NetworkError) return 'INFRASTRUCTURE';
        if (error.name === 'TypeError') return 'PROGRAMMING';
        if (error.name === 'ReferenceError') return 'PROGRAMMING';
        return 'UNKNOWN';
    }
    
    getRecoveryStrategy(category) {
        const strategies = {
            'BUSINESS_RULE': {
                retry: false,
                userAction: 'Display error to user',
                logLevel: 'INFO'
            },
            'VALIDATION': {
                retry: false,
                userAction: 'Request correct input',
                logLevel: 'INFO'
            },
            'SECURITY': {
                retry: false,
                userAction: 'Block transaction, notify security',
                logLevel: 'CRITICAL'
            },
            'INFRASTRUCTURE': {
                retry: true,
                retryAttempts: 3,
                retryDelay: 1000,
                userAction: 'Show retry option',
                logLevel: 'ERROR'
            },
            'PROGRAMMING': {
                retry: false,
                userAction: 'Show generic error, alert developers',
                logLevel: 'CRITICAL'
            },
            'UNKNOWN': {
                retry: false,
                userAction: 'Show generic error',
                logLevel: 'ERROR'
            }
        };
        
        return strategies[category] || strategies['UNKNOWN'];
    }
    
    onError(callback) {
        this.errorListeners.push(callback);
    }
    
    notifyListeners(error, context) {
        this.errorListeners.forEach(callback => {
            try {
                callback(error, context);
            } catch (err) {
                console.error('Error in error listener:', err);
            }
        });
    }
    
    getErrorReport() {
        const totalErrors = this.errorLog.length;
        const errorsByCategory = {};
        
        this.errorLog.forEach(log => {
            const category = this.categorizeError(log.error);
            errorsByCategory[category] = (errorsByCategory[category] || 0) + 1;
        });
        
        return {
            totalErrors,
            errorsByCategory,
            recentErrors: this.errorLog.slice(-10)
        };
    }
}

// Transaction Service with Error Handling
class TransactionService {
    constructor(errorHandler) {
        this.errorHandler = errorHandler;
        this.retryAttempts = 3;
        this.retryDelay = 1000;
    }
    
    // Try-catch with specific error types
    async processTransfer(fromAccount, toAccount, amount) {
        try {
            console.log(`\nProcessing transfer: ${amount} from ${fromAccount} to ${toAccount}`);
            
            // Validate accounts
            await this.validateAccount(fromAccount);
            await this.validateAccount(toAccount);
            
            // Check balance
            const balance = await this.getBalance(fromAccount);
            if (balance < amount) {
                throw new InsufficientFundsError(balance, amount);
            }
            
            // Check limits
            const dailyLimit = 50000;
            if (amount > dailyLimit) {
                throw new TransactionLimitExceededError(dailyLimit, amount);
            }
            
            // Fraud detection
            if (await this.detectFraud(fromAccount, toAccount, amount)) {
                throw new FraudDetectionError('Unusual transaction pattern', `TXN${Date.now()}`);
            }
            
            // Execute transfer
            const result = await this.executeTransfer(fromAccount, toAccount, amount);
            
            console.log('✓ Transfer successful');
            return result;
            
        } catch (error) {
            return this.handleTransactionError(error, { fromAccount, toAccount, amount });
        }
    }
    
    async handleTransactionError(error, context) {
        const handled = this.errorHandler.handleError(error, context);
        
        // Apply recovery strategy
        if (handled.recovery.retry) {
            console.log(`Retrying transaction...`);
            return await this.retryTransaction(context);
        }
        
        // Return error response
        return {
            success: false,
            error: error instanceof BankingError ? error.toJSON() : {
                name: error.name,
                message: error.message
            },
            recovery: handled.recovery.userAction
        };
    }
    
    async retryTransaction(context, attempt = 0) {
        if (attempt >= this.retryAttempts) {
            throw new Error('Max retry attempts exceeded');
        }
        
        try {
            await new Promise(resolve => 
                setTimeout(resolve, this.retryDelay * Math.pow(2, attempt))
            );
            
            return await this.processTransfer(
                context.fromAccount,
                context.toAccount,
                context.amount
            );
            
        } catch (error) {
            if (error instanceof NetworkError && attempt < this.retryAttempts - 1) {
                return await this.retryTransaction(context, attempt + 1);
            }
            throw error;
        }
    }
    
    // Promise chaining with error propagation
    async getAccountSummary(accountId) {
        return this.validateAccount(accountId)
            .then(() => this.getBalance(accountId))
            .then(balance => this.getTransactionHistory(accountId)
                .then(transactions => ({ accountId, balance, transactions }))
            )
            .catch(error => {
                // Centralized error handling
                this.errorHandler.handleError(error, { accountId });
                throw error;
            });
    }
    
    // Async/await with proper error handling
    async getBalance(accountId) {
        try {
            // Simulated API call
            if (accountId === 'INVALID') {
                throw new InvalidAccountError(accountId);
            }
            
            return Math.random() * 100000;
            
        } catch (error) {
            // Re-throw with context
            if (!(error instanceof BankingError)) {
                throw new NetworkError(error);
            }
            throw error;
        }
    }
    
    async validateAccount(accountId) {
        if (!accountId || accountId === 'INVALID') {
            throw new InvalidAccountError(accountId);
        }
        return true;
    }
    
    async detectFraud(fromAccount, toAccount, amount) {
        // Simulated fraud detection
        return amount > 100000; // Flag large transactions
    }
    
    async executeTransfer(fromAccount, toAccount, amount) {
        // Simulated transfer execution
        return {
            transactionId: `TXN${Date.now()}`,
            fromAccount,
            toAccount,
            amount,
            status: 'COMPLETED',
            timestamp: new Date().toISOString()
        };
    }
    
    async getTransactionHistory(accountId) {
        return [
            { id: 1, amount: 1000, type: 'CREDIT' },
            { id: 2, amount: 500, type: 'DEBIT' }
        ];
    }
}

// Global Error Handling
class GlobalErrorHandler {
    static setupGlobalHandlers() {
        // Unhandled promise rejections
        if (typeof window !== 'undefined') {
            window.addEventListener('unhandledrejection', (event) => {
                console.error('Unhandled Promise Rejection:', event.reason);
                // Send to error tracking service
                event.preventDefault();
            });
            
            // Global error handler
            window.addEventListener('error', (event) => {
                console.error('Global Error:', event.error);
                // Send to error tracking service
            });
        }
        
        // Node.js process handlers
        if (typeof process !== 'undefined') {
            process.on('unhandledRejection', (reason, promise) => {
                console.error('Unhandled Rejection at:', promise, 'reason:', reason);
            });
            
            process.on('uncaughtException', (error) => {
                console.error('Uncaught Exception:', error);
                // Graceful shutdown
                process.exit(1);
            });
        }
    }
}

// Usage Examples
async function demonstrateErrorHandling() {
    console.log('=== Error Handling Demo ===');
    
    GlobalErrorHandler.setupGlobalHandlers();
    
    const errorHandler = new ErrorHandlerService();
    const transactionService = new TransactionService(errorHandler);
    
    // Listen to errors
    errorHandler.onError((error, context) => {
        console.log(`\n[Error Listener] ${error.name}: ${error.message}`);
    });
    
    // Test Case 1: Insufficient Funds
    console.log('\n--- Test: Insufficient Funds ---');
    await transactionService.processTransfer('ACC001', 'ACC002', 1000000);
    
    // Test Case 2: Invalid Account
    console.log('\n--- Test: Invalid Account ---');
    await transactionService.processTransfer('INVALID', 'ACC002', 5000);
    
    // Test Case 3: Fraud Detection
    console.log('\n--- Test: Fraud Detection ---');
    await transactionService.processTransfer('ACC001', 'ACC002', 150000);
    
    // Test Case 4: Successful Transaction
    console.log('\n--- Test: Successful Transaction ---');
    await transactionService.processTransfer('ACC001', 'ACC002', 5000);
    
    // Error Report
    console.log('\n--- Error Report ---');
    console.log(errorHandler.getErrorReport());
}

// demonstrateErrorHandling();
```

---
