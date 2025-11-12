# Questions 25-27: Advanced Async Patterns & Web APIs

## Question 25: What are Web Workers? How do they enable parallel processing?

### Answer:

**Web Workers** are JavaScript scripts that run in background threads, separate from the main execution thread. They enable true parallel processing in the browser, preventing heavy computations from blocking the UI.

### Key Concepts:

1. **Dedicated Workers**: Single script, single page
2. **Shared Workers**: Shared across multiple tabs/windows
3. **Service Workers**: Network proxy for PWAs
4. **Message Passing**: Communication via `postMessage()`
5. **Thread Safety**: Workers have separate memory space

### Limitations:
- No DOM access
- No `window` object
- Limited APIs available
- Must communicate via messages

### Banking Scenario: Background Processing at Emirates NBD

```javascript
console.log('=== Web Workers Demo - Emirates NBD ===\n');

// =============================================================================
// SIMULATED WORKER (for Node.js/console demonstration)
// =============================================================================

// In a real browser, you'd create a separate worker.js file
// Here we'll simulate worker behavior

class WorkerSimulator {
    constructor(workerFunction) {
        this.workerFunction = workerFunction;
        this.onmessage = null;
    }

    postMessage(data) {
        // Simulate async worker processing
        setTimeout(() => {
            const result = this.workerFunction(data);
            if (this.onmessage) {
                this.onmessage({ data: result });
            }
        }, 10);
    }

    terminate() {
        console.log('[Worker] Terminated');
    }
}

// =============================================================================
// EXAMPLE 1: Transaction Processing Worker
// =============================================================================

console.log('1. Transaction Processing Worker:\n');

// Worker code (would be in separate file: transaction-worker.js)
const transactionWorkerCode = (message) => {
    const { transactions } = message;
    
    console.log(`[Worker] Processing ${transactions.length} transactions...`);
    
    // Heavy computation
    const results = transactions.map(txn => {
        // Simulate complex validation and processing
        let fraudScore = 0;
        
        // Check amount
        if (txn.amount > 50000) fraudScore += 30;
        if (txn.amount > 100000) fraudScore += 50;
        
        // Check frequency (simulated)
        if (txn.frequency > 10) fraudScore += 20;
        
        // Simulate heavy computation
        for (let i = 0; i < 100000; i++) {
            fraudScore += Math.sin(i) * 0.0001;
        }
        
        return {
            ...txn,
            fraudScore: Math.round(fraudScore),
            risk: fraudScore > 70 ? 'HIGH' : fraudScore > 40 ? 'MEDIUM' : 'LOW',
            processedAt: new Date().toISOString()
        };
    });
    
    console.log(`[Worker] Completed processing ${results.length} transactions`);
    
    return {
        type: 'PROCESSING_COMPLETE',
        results,
        summary: {
            total: results.length,
            highRisk: results.filter(r => r.risk === 'HIGH').length,
            mediumRisk: results.filter(r => r.risk === 'MEDIUM').length,
            lowRisk: results.filter(r => r.risk === 'LOW').length
        }
    };
};

// Main thread code
function processTransactionsInBackground() {
    console.log('[Main] Starting background transaction processing...');
    
    const transactions = Array.from({ length: 100 }, (_, i) => ({
        id: `TXN-${String(i + 1).padStart(4, '0')}`,
        amount: Math.random() * 150000,
        frequency: Math.floor(Math.random() * 20),
        accountNumber: `ACC-${Math.floor(Math.random() * 1000000)}`
    }));
    
    // Create worker
    const worker = new WorkerSimulator(transactionWorkerCode);
    
    // Listen for messages from worker
    worker.onmessage = (event) => {
        const { results, summary } = event.data;
        
        console.log('\n[Main] Received results from worker:');
        console.log(`  Total transactions: ${summary.total}`);
        console.log(`  High risk: ${summary.highRisk}`);
        console.log(`  Medium risk: ${summary.mediumRisk}`);
        console.log(`  Low risk: ${summary.lowRisk}`);
        
        console.log('\nSample results:');
        results.slice(0, 3).forEach(r => {
            console.log(`  ${r.id}: Risk=${r.risk}, Score=${r.fraudScore}`);
        });
    };
    
    // Send data to worker
    worker.postMessage({ transactions });
    
    console.log('[Main] Message sent to worker, continuing main thread execution...');
    console.log('[Main] UI remains responsive!\n');
}

processTransactionsInBackground();

setTimeout(() => {
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // EXAMPLE 2: Interest Calculation Worker
    // =============================================================================
    
    console.log('2. Interest Calculation Worker:\n');
    
    const interestWorkerCode = (message) => {
        const { accounts, rate, years } = message;
        
        console.log(`[Worker] Calculating interest for ${accounts.length} accounts...`);
        
        const calculations = accounts.map(account => {
            // Complex compound interest calculation
            const monthlyRate = rate / 12 / 100;
            const periods = years * 12;
            
            let futureValue = account.balance;
            const breakdown = [];
            
            // Calculate month by month
            for (let month = 1; month <= Math.min(periods, 120); month++) {
                const interest = futureValue * monthlyRate;
                futureValue += interest;
                
                if (month % 12 === 0) {
                    breakdown.push({
                        year: month / 12,
                        balance: Math.round(futureValue * 100) / 100
                    });
                }
            }
            
            return {
                accountNumber: account.accountNumber,
                initialBalance: account.balance,
                finalBalance: Math.round(futureValue * 100) / 100,
                totalInterest: Math.round((futureValue - account.balance) * 100) / 100,
                breakdown
            };
        });
        
        console.log(`[Worker] Interest calculation complete`);
        
        return {
            type: 'INTEREST_CALCULATED',
            calculations
        };
    };
    
    function calculateInterestInBackground() {
        console.log('[Main] Starting background interest calculation...');
        
        const accounts = Array.from({ length: 1000 }, (_, i) => ({
            accountNumber: `ACC-${String(i + 1).padStart(6, '0')}`,
            balance: Math.random() * 100000 + 10000
        }));
        
        const worker = new WorkerSimulator(interestWorkerCode);
        
        worker.onmessage = (event) => {
            const { calculations } = event.data;
            
            console.log('\n[Main] Interest calculation results:');
            console.log(`  Processed ${calculations.length} accounts`);
            
            const totalInterest = calculations.reduce((sum, calc) => sum + calc.totalInterest, 0);
            console.log(`  Total interest: AED ${totalInterest.toLocaleString()}`);
            
            console.log('\nSample calculations:');
            calculations.slice(0, 2).forEach(calc => {
                console.log(`  ${calc.accountNumber}:`);
                console.log(`    Initial: AED ${calc.initialBalance.toLocaleString()}`);
                console.log(`    Final: AED ${calc.finalBalance.toLocaleString()}`);
                console.log(`    Interest: AED ${calc.totalInterest.toLocaleString()}`);
            });
        };
        
        worker.postMessage({ accounts, rate: 5, years: 10 });
        
        console.log('[Main] Message sent to worker\n');
    }
    
    calculateInterestInBackground();
    
}, 100);

setTimeout(() => {
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // EXAMPLE 3: Report Generation Worker
    // =============================================================================
    
    console.log('3. Report Generation Worker:\n');
    
    class ReportWorkerPool {
        constructor(size = 4) {
            this.workers = [];
            this.taskQueue = [];
            this.activeWorkers = 0;
            this.size = size;
            
            console.log(`[Pool] Created worker pool with ${size} workers`);
        }
        
        runTask(data, workerFunction) {
            return new Promise((resolve) => {
                const task = { data, workerFunction, resolve };
                
                if (this.activeWorkers < this.size) {
                    this.executeTask(task);
                } else {
                    this.taskQueue.push(task);
                    console.log(`[Pool] Task queued (${this.taskQueue.length} in queue)`);
                }
            });
        }
        
        executeTask(task) {
            this.activeWorkers++;
            
            const worker = new WorkerSimulator(task.workerFunction);
            
            worker.onmessage = (event) => {
                task.resolve(event.data);
                this.activeWorkers--;
                worker.terminate();
                
                // Process next task in queue
                if (this.taskQueue.length > 0) {
                    const nextTask = this.taskQueue.shift();
                    this.executeTask(nextTask);
                }
            };
            
            worker.postMessage(task.data);
        }
        
        terminate() {
            this.workers.forEach(w => w.terminate());
            console.log('[Pool] All workers terminated');
        }
    }
    
    const reportWorkerCode = (message) => {
        const { accountNumber, startDate, endDate } = message;
        
        console.log(`[Worker] Generating report for ${accountNumber}...`);
        
        // Simulate heavy report generation
        const transactions = [];
        for (let i = 0; i < 1000; i++) {
            transactions.push({
                id: `TXN-${i}`,
                date: new Date(Date.now() - Math.random() * 30 * 24 * 60 * 60 * 1000),
                amount: Math.random() * 10000,
                type: ['CREDIT', 'DEBIT'][Math.floor(Math.random() * 2)]
            });
        }
        
        const report = {
            accountNumber,
            period: { startDate, endDate },
            totalTransactions: transactions.length,
            totalCredits: transactions.filter(t => t.type === 'CREDIT')
                .reduce((sum, t) => sum + t.amount, 0),
            totalDebits: transactions.filter(t => t.type === 'DEBIT')
                .reduce((sum, t) => sum + t.amount, 0),
            generatedAt: new Date().toISOString()
        };
        
        console.log(`[Worker] Report generated for ${accountNumber}`);
        
        return report;
    };
    
    async function generateReportsInParallel() {
        console.log('[Main] Generating multiple reports in parallel...\n');
        
        const pool = new ReportWorkerPool(4);
        
        const accounts = [
            'ACC-001', 'ACC-002', 'ACC-003', 'ACC-004',
            'ACC-005', 'ACC-006', 'ACC-007', 'ACC-008'
        ];
        
        const reportPromises = accounts.map(accountNumber => 
            pool.runTask({
                accountNumber,
                startDate: '2025-01-01',
                endDate: '2025-11-11'
            }, reportWorkerCode)
        );
        
        const reports = await Promise.all(reportPromises);
        
        console.log('\n[Main] All reports generated:');
        reports.forEach(report => {
            console.log(`  ${report.accountNumber}:`);
            console.log(`    Transactions: ${report.totalTransactions}`);
            console.log(`    Credits: AED ${Math.round(report.totalCredits).toLocaleString()}`);
            console.log(`    Debits: AED ${Math.round(report.totalDebits).toLocaleString()}`);
        });
        
        pool.terminate();
    }
    
    generateReportsInParallel();
    
}, 300);

setTimeout(() => {
    console.log('\n' + '='.repeat(70) + '\n');
    
    // Summary
    console.log('4. WEB WORKERS SUMMARY:\n');
    
    console.log('Benefits:');
    console.log('  ✓ True parallel processing');
    console.log('  ✓ Non-blocking UI');
    console.log('  ✓ CPU-intensive tasks offloaded');
    console.log('  ✓ Better performance on multi-core systems');
    
    console.log('\nUse Cases:');
    console.log('  • Image/video processing');
    console.log('  • Data analysis and calculations');
    console.log('  • Encryption/decryption');
    console.log('  • Large dataset processing');
    console.log('  • Report generation');
    console.log('  • Fraud detection algorithms');
    
    console.log('\nLimitations:');
    console.log('  ✗ No DOM access');
    console.log('  ✗ No access to parent page objects');
    console.log('  ✗ Limited APIs (no localStorage, etc.)');
    console.log('  ✗ Message passing overhead');
    
    console.log('\nBest Practices:');
    console.log('  • Use for CPU-intensive tasks (>50ms)');
    console.log('  • Minimize data transfer (use Transferable objects)');
    console.log('  • Implement worker pools for multiple tasks');
    console.log('  • Handle worker errors gracefully');
    console.log('  • Terminate workers when done');
    
    console.log('\nReal Browser Usage:');
    console.log('  const worker = new Worker("worker.js");');
    console.log('  worker.postMessage({ data: "..." });');
    console.log('  worker.onmessage = (e) => console.log(e.data);');
    console.log('  worker.terminate();');
    
}, 500);
```

---

## Question 26: What is IndexedDB? How do you use it for client-side storage?

### Answer:

**IndexedDB** is a low-level API for client-side storage of significant amounts of structured data. It's a NoSQL database built into the browser, supporting transactions, indexes, and queries.

### Key Features:

1. **Large Storage**: Much larger than localStorage (typically 50MB+)
2. **Indexed**: Fast lookups via indexes
3. **Transactional**: ACID properties
4. **Asynchronous**: Non-blocking operations
5. **Same-Origin**: Isolated per domain

### Banking Scenario: Offline Banking at Emirates NBD

```javascript
console.log('=== IndexedDB Demo - Emirates NBD ===\n');

// =============================================================================
// INDEXEDDB WRAPPER CLASS
// =============================================================================

class BankingDatabase {
    constructor(dbName = 'EmiratesNBD', version = 1) {
        this.dbName = dbName;
        this.version = version;
        this.db = null;
    }

    // Initialize database
    async init() {
        return new Promise((resolve, reject) => {
            console.log(`[IndexedDB] Opening database: ${this.dbName}`);
            
            // Simulated IndexedDB (in real browser, use: indexedDB.open())
            // For demonstration, we'll use a mock object
            
            this.db = {
                name: this.dbName,
                version: this.version,
                objectStores: {
                    accounts: new Map(),
                    transactions: new Map(),
                    cache: new Map()
                }
            };
            
            console.log('[IndexedDB] Database opened successfully\n');
            resolve(this.db);
        });
    }

    // Add account
    async addAccount(account) {
        console.log(`[IndexedDB] Adding account: ${account.accountNumber}`);
        
        this.db.objectStores.accounts.set(account.accountNumber, {
            ...account,
            createdAt: new Date(),
            syncStatus: 'SYNCED'
        });
        
        console.log('✓ Account added\n');
        return account.accountNumber;
    }

    // Get account
    async getAccount(accountNumber) {
        console.log(`[IndexedDB] Retrieving account: ${accountNumber}`);
        
        const account = this.db.objectStores.accounts.get(accountNumber);
        
        if (account) {
            console.log('✓ Account found\n');
            return account;
        } else {
            console.log('✗ Account not found\n');
            return null;
        }
    }

    // Update account
    async updateAccount(accountNumber, updates) {
        console.log(`[IndexedDB] Updating account: ${accountNumber}`);
        
        const account = this.db.objectStores.accounts.get(accountNumber);
        
        if (account) {
            const updated = {
                ...account,
                ...updates,
                updatedAt: new Date(),
                syncStatus: 'PENDING_SYNC'
            };
            
            this.db.objectStores.accounts.set(accountNumber, updated);
            console.log('✓ Account updated\n');
            return updated;
        }
        
        throw new Error('Account not found');
    }

    // Add transaction
    async addTransaction(transaction) {
        console.log(`[IndexedDB] Adding transaction: ${transaction.id}`);
        
        this.db.objectStores.transactions.set(transaction.id, {
            ...transaction,
            timestamp: new Date(),
            syncStatus: 'PENDING_SYNC'
        });
        
        console.log('✓ Transaction added\n');
        return transaction.id;
    }

    // Get transactions by account
    async getTransactionsByAccount(accountNumber, limit = 10) {
        console.log(`[IndexedDB] Getting transactions for: ${accountNumber}`);
        
        const allTransactions = Array.from(this.db.objectStores.transactions.values());
        const accountTransactions = allTransactions
            .filter(txn => txn.accountNumber === accountNumber)
            .sort((a, b) => b.timestamp - a.timestamp)
            .slice(0, limit);
        
        console.log(`✓ Found ${accountTransactions.length} transactions\n`);
        return accountTransactions;
    }

    // Get pending sync items
    async getPendingSync() {
        const accounts = Array.from(this.db.objectStores.accounts.values())
            .filter(acc => acc.syncStatus === 'PENDING_SYNC');
        
        const transactions = Array.from(this.db.objectStores.transactions.values())
            .filter(txn => txn.syncStatus === 'PENDING_SYNC');
        
        console.log(`[IndexedDB] Pending sync: ${accounts.length} accounts, ${transactions.length} transactions\n`);
        
        return { accounts, transactions };
    }

    // Mark as synced
    async markAsSynced(type, id) {
        const store = type === 'account' ? 'accounts' : 'transactions';
        const item = this.db.objectStores[store].get(id);
        
        if (item) {
            item.syncStatus = 'SYNCED';
            item.lastSyncedAt = new Date();
            this.db.objectStores[store].set(id, item);
            console.log(`[IndexedDB] Marked as synced: ${type} ${id}`);
        }
    }

    // Clear all data
    async clearAll() {
        console.log('[IndexedDB] Clearing all data...');
        
        this.db.objectStores.accounts.clear();
        this.db.objectStores.transactions.clear();
        this.db.objectStores.cache.clear();
        
        console.log('✓ All data cleared\n');
    }

    // Get storage statistics
    async getStats() {
        const stats = {
            accounts: this.db.objectStores.accounts.size,
            transactions: this.db.objectStores.transactions.size,
            cache: this.db.objectStores.cache.size,
            totalSize: this.db.objectStores.accounts.size + 
                       this.db.objectStores.transactions.size + 
                       this.db.objectStores.cache.size
        };
        
        console.log('[IndexedDB] Storage statistics:');
        console.log(`  Accounts: ${stats.accounts}`);
        console.log(`  Transactions: ${stats.transactions}`);
        console.log(`  Cache entries: ${stats.cache}`);
        console.log(`  Total items: ${stats.totalSize}\n`);
        
        return stats;
    }
}

// =============================================================================
// OFFLINE-FIRST BANKING APPLICATION
// =============================================================================

class OfflineBankingApp {
    constructor() {
        this.db = new BankingDatabase();
        this.isOnline = true;
    }

    async initialize() {
        await this.db.init();
        console.log('[App] Banking app initialized\n');
    }

    // Create account (works offline)
    async createAccount(accountData) {
        console.log('[App] Creating account...');
        
        const account = {
            accountNumber: accountData.accountNumber,
            name: accountData.name,
            balance: accountData.balance || 0,
            type: accountData.type || 'SAVINGS'
        };
        
        await this.db.addAccount(account);
        
        if (this.isOnline) {
            await this.syncToServer(account);
        } else {
            console.log('[App] Offline: Account will be synced when online\n');
        }
        
        return account;
    }

    // Deposit (works offline)
    async deposit(accountNumber, amount) {
        console.log(`[App] Processing deposit: AED ${amount.toLocaleString()}`);
        
        const account = await this.db.getAccount(accountNumber);
        
        if (!account) {
            throw new Error('Account not found');
        }
        
        // Update balance
        await this.db.updateAccount(accountNumber, {
            balance: account.balance + amount
        });
        
        // Record transaction
        const transaction = {
            id: `TXN-${Date.now()}`,
            accountNumber,
            type: 'DEPOSIT',
            amount,
            status: 'COMPLETED'
        };
        
        await this.db.addTransaction(transaction);
        
        if (this.isOnline) {
            await this.syncToServer(transaction);
        } else {
            console.log('[App] Offline: Transaction will be synced when online\n');
        }
        
        return transaction;
    }

    // Withdraw (works offline)
    async withdraw(accountNumber, amount) {
        console.log(`[App] Processing withdrawal: AED ${amount.toLocaleString()}`);
        
        const account = await this.db.getAccount(accountNumber);
        
        if (!account) {
            throw new Error('Account not found');
        }
        
        if (account.balance < amount) {
            throw new Error('Insufficient funds');
        }
        
        // Update balance
        await this.db.updateAccount(accountNumber, {
            balance: account.balance - amount
        });
        
        // Record transaction
        const transaction = {
            id: `TXN-${Date.now()}`,
            accountNumber,
            type: 'WITHDRAWAL',
            amount,
            status: 'COMPLETED'
        };
        
        await this.db.addTransaction(transaction);
        
        if (this.isOnline) {
            await this.syncToServer(transaction);
        } else {
            console.log('[App] Offline: Transaction will be synced when online\n');
        }
        
        return transaction;
    }

    // Get account summary
    async getAccountSummary(accountNumber) {
        console.log(`[App] Getting account summary for ${accountNumber}`);
        
        const account = await this.db.getAccount(accountNumber);
        const transactions = await this.db.getTransactionsByAccount(accountNumber, 5);
        
        const summary = {
            ...account,
            recentTransactions: transactions,
            lastActivity: transactions[0]?.timestamp || account.createdAt
        };
        
        console.log('✓ Summary retrieved\n');
        return summary;
    }

    // Sync to server
    async syncToServer(data) {
        console.log('[App] Syncing to server...');
        
        // Simulate API call
        await new Promise(resolve => setTimeout(resolve, 100));
        
        console.log('✓ Synced to server\n');
    }

    // Sync all pending data
    async syncAllPending() {
        console.log('[App] Syncing all pending data...\n');
        
        const pending = await this.db.getPendingSync();
        
        // Sync accounts
        for (const account of pending.accounts) {
            await this.syncToServer(account);
            await this.db.markAsSynced('account', account.accountNumber);
        }
        
        // Sync transactions
        for (const transaction of pending.transactions) {
            await this.syncToServer(transaction);
            await this.db.markAsSynced('transaction', transaction.id);
        }
        
        console.log('✓ All data synced\n');
    }

    // Go offline
    goOffline() {
        this.isOnline = false;
        console.log('[App] 📡 OFFLINE MODE\n');
    }

    // Go online
    async goOnline() {
        this.isOnline = true;
        console.log('[App] 📡 ONLINE MODE');
        await this.syncAllPending();
    }
}

// =============================================================================
// DEMONSTRATION
// =============================================================================

(async () => {
    console.log('=== Offline Banking Demo ===\n');
    
    const app = new OfflineBankingApp();
    await app.initialize();
    
    // Create account (online)
    const account = await app.createAccount({
        accountNumber: 'ACC-123456789',
        name: 'Ahmed Mohammed',
        balance: 50000,
        type: 'SAVINGS'
    });
    
    // Perform transactions (online)
    await app.deposit('ACC-123456789', 10000);
    await app.withdraw('ACC-123456789', 5000);
    
    // Go offline
    app.goOffline();
    
    // Perform transactions offline
    await app.deposit('ACC-123456789', 3000);
    await app.withdraw('ACC-123456789', 1000);
    await app.deposit('ACC-123456789', 2000);
    
    // Get summary (works offline)
    const summary = await app.getAccountSummary('ACC-123456789');
    console.log('Account Summary:');
    console.log(`  Name: ${summary.name}`);
    console.log(`  Balance: AED ${summary.balance.toLocaleString()}`);
    console.log(`  Recent Transactions: ${summary.recentTransactions.length}`);
    console.log(`  Sync Status: ${summary.syncStatus}\n`);
    
    // Go back online
    await app.goOnline();
    
    // Get final stats
    await app.db.getStats();
    
    console.log('='.repeat(70) + '\n');
    
    // Summary
    console.log('INDEXEDDB SUMMARY:\n');
    
    console.log('Features:');
    console.log('  • Large storage capacity (50MB - GBs)');
    console.log('  • Indexed for fast queries');
    console.log('  • Transactional (ACID)');
    console.log('  • Asynchronous API');
    console.log('  • Supports complex data types');
    
    console.log('\nUse Cases:');
    console.log('  • Offline-first applications');
    console.log('  • Caching API responses');
    console.log('  • Storing user-generated content');
    console.log('  • Progressive Web Apps (PWAs)');
    console.log('  • Large datasets (forms, drafts)');
    
    console.log('\nBest Practices:');
    console.log('  ✓ Use for structured data');
    console.log('  ✓ Create indexes for frequent queries');
    console.log('  ✓ Handle version upgrades carefully');
    console.log('  ✓ Implement sync mechanisms');
    console.log('  ✓ Handle quota exceeded errors');
    
    console.log('\nComparison with Other Storage:');
    console.log('  localStorage: 5-10MB, synchronous, string only');
    console.log('  sessionStorage: 5-10MB, cleared on tab close');
    console.log('  IndexedDB: 50MB+, asynchronous, any data type');
    console.log('  Cache API: Unlimited, for HTTP responses');
    
})();
```

---

## Question 27: What are Service Workers? How do they enable PWA features?

### Answer:

**Service Workers** are scripts that run in the background, separate from web pages. They act as a network proxy, enabling offline functionality, push notifications, and background sync.

### Key Features:

1. **Offline Support**: Cache resources for offline access
2. **Push Notifications**: Receive notifications when app is closed
3. **Background Sync**: Sync data when connection is restored
4. **Intercept Network**: Modify requests/responses
5. **HTTPS Only**: Security requirement

### Service Worker Lifecycle:
1. **Install**: Download and cache resources
2. **Activate**: Clean up old caches
3. **Fetch**: Intercept network requests
4. **Message**: Communicate with pages

### Banking Scenario: PWA Features at Emirates NBD

```javascript
console.log('=== Service Workers & PWA Demo - Emirates NBD ===\n');

// =============================================================================
// SERVICE WORKER SIMULATION
// =============================================================================

class ServiceWorkerSimulator {
    constructor() {
        this.cache = new Map();
        this.cacheVersion = 'v1';
        this.state = 'installing';
        this.syncQueue = [];
    }

    // Install event - cache static assets
    async install() {
        console.log('[SW] Installing...');
        this.state = 'installing';
        
        const resourcesToCache = [
            '/',
            '/index.html',
            '/styles.css',
            '/app.js',
            '/api/accounts',
            '/api/transactions'
        ];
        
        console.log('[SW] Caching resources:');
        for (const resource of resourcesToCache) {
            await this.cacheResource(resource);
        }
        
        this.state = 'installed';
        console.log('[SW] Installation complete\n');
    }

    // Activate event - clean up old caches
    async activate() {
        console.log('[SW] Activating...');
        this.state = 'activating';
        
        // Clean up old caches
        console.log('[SW] Cleaning up old caches...');
        
        this.state = 'activated';
        console.log('[SW] Activation complete\n');
    }

    // Cache resource
    async cacheResource(url) {
        // Simulate fetching and caching
        const mockResponse = {
            url,
            data: `Cached content for ${url}`,
            timestamp: Date.now()
        };
        
        this.cache.set(url, mockResponse);
        console.log(`  ✓ Cached: ${url}`);
    }

    // Fetch event - intercept network requests
    async fetch(request) {
        const { url, options = {} } = request;
        
        console.log(`[SW] Intercepting request: ${url}`);
        
        // Check cache first
        if (this.cache.has(url)) {
            console.log(`  💾 Returning from cache`);
            return this.cache.get(url);
        }
        
        // If offline, return error
        if (!options.online) {
            console.log(`  📡 Offline: Cannot fetch ${url}`);
            return {
                error: 'Offline',
                url,
                cachedVersion: null
            };
        }
        
        // Fetch from network and cache
        console.log(`  🌐 Fetching from network...`);
        const response = await this.fetchFromNetwork(url);
        
        // Cache the response
        this.cache.set(url, response);
        
        return response;
    }

    // Simulate network fetch
    async fetchFromNetwork(url) {
        await new Promise(resolve => setTimeout(resolve, 50));
        
        return {
            url,
            data: `Fresh content for ${url}`,
            timestamp: Date.now()
        };
    }

    // Background sync
    async backgroundSync() {
        console.log('[SW] Running background sync...');
        
        if (this.syncQueue.length === 0) {
            console.log('  No pending syncs\n');
            return;
        }
        
        console.log(`  Processing ${this.syncQueue.length} pending requests...`);
        
        for (const item of this.syncQueue) {
            try {
                await this.fetchFromNetwork(item.url);
                console.log(`  ✓ Synced: ${item.url}`);
            } catch (error) {
                console.log(`  ✗ Failed: ${item.url}`);
            }
        }
        
        this.syncQueue = [];
        console.log('  Background sync complete\n');
    }

    // Add to sync queue
    queueSync(request) {
        this.syncQueue.push(request);
        console.log(`[SW] Queued for sync: ${request.url}`);
    }

    // Push notification
    async showNotification(title, options) {
        console.log('[SW] Showing notification:');
        console.log(`  Title: ${title}`);
        console.log(`  Body: ${options.body}`);
        console.log(`  Icon: ${options.icon || 'default'}`);
        console.log(`  Tag: ${options.tag || 'none'}\n`);
    }
}

// =============================================================================
// PWA BANKING APPLICATION
// =============================================================================

class PWABankingApp {
    constructor() {
        this.serviceWorker = new ServiceWorkerSimulator();
        this.isOnline = true;
    }

    async initialize() {
        console.log('[App] Initializing PWA...\n');
        
        // Register and install service worker
        await this.serviceWorker.install();
        await this.serviceWorker.activate();
        
        console.log('[App] PWA initialized\n');
    }

    // Fetch with service worker
    async fetchData(url) {
        return await this.serviceWorker.fetch({
            url,
            options: { online: this.isOnline }
        });
    }

    // Make transaction (with offline support)
    async makeTransaction(transaction) {
        console.log('[App] Making transaction...');
        
        if (this.isOnline) {
            const response = await this.fetchData('/api/transactions');
            console.log(`  ✓ Transaction completed: ${transaction.id}\n`);
            
            // Send notification
            await this.serviceWorker.showNotification(
                'Transaction Successful',
                {
                    body: `AED ${transaction.amount.toLocaleString()} ${transaction.type}`,
                    icon: '/icons/success.png',
                    tag: 'transaction'
                }
            );
            
            return response;
        } else {
            console.log('  📡 Offline: Queueing transaction for sync\n');
            
            this.serviceWorker.queueSync({
                url: '/api/transactions',
                data: transaction
            });
            
            // Show offline notification
            await this.serviceWorker.showNotification(
                'Transaction Queued',
                {
                    body: 'Will sync when online',
                    icon: '/icons/offline.png',
                    tag: 'offline'
                }
            );
            
            return { queued: true };
        }
    }

    // Get account data (with caching)
    async getAccountData(accountNumber) {
        console.log(`[App] Getting account data: ${accountNumber}`);
        
        const response = await this.fetchData(`/api/accounts/${accountNumber}`);
        
        console.log('  Account data retrieved\n');
        return response;
    }

    // Simulate going offline
    goOffline() {
        this.isOnline = false;
        console.log('[App] 📡 OFFLINE MODE\n');
    }

    // Simulate going online
    async goOnline() {
        this.isOnline = true;
        console.log('[App] 📡 ONLINE MODE');
        
        // Trigger background sync
        await this.serviceWorker.backgroundSync();
    }

    // Install PWA
    async installPWA() {
        console.log('[App] Installing PWA...');
        console.log('  ✓ Added to home screen');
        console.log('  ✓ Standalone mode enabled');
        console.log('  ✓ App icon created\n');
    }
}

// =============================================================================
// DEMONSTRATION
// =============================================================================

(async () => {
    console.log('=== PWA Features Demo ===\n');
    
    const app = new PWABankingApp();
    await app.initialize();
    
    // Fetch account data (will be cached)
    console.log('1. Fetching Account Data (Online):\n');
    await app.getAccountData('ACC-123456789');
    
    // Make transaction (online)
    console.log('2. Making Transaction (Online):\n');
    await app.makeTransaction({
        id: 'TXN-001',
        type: 'DEPOSIT',
        amount: 5000,
        accountNumber: 'ACC-123456789'
    });
    
    // Go offline
    app.goOffline();
    
    // Fetch cached data (offline)
    console.log('3. Fetching Account Data (Offline - from cache):\n');
    await app.getAccountData('ACC-123456789');
    
    // Make transaction offline (queued)
    console.log('4. Making Transaction (Offline - queued):\n');
    await app.makeTransaction({
        id: 'TXN-002',
        type: 'WITHDRAWAL',
        amount: 2000,
        accountNumber: 'ACC-123456789'
    });
    
    await app.makeTransaction({
        id: 'TXN-003',
        type: 'TRANSFER',
        amount: 1000,
        accountNumber: 'ACC-123456789'
    });
    
    // Go back online
    console.log('5. Going Online (Syncing queued transactions):\n');
    await app.goOnline();
    
    // Install as PWA
    console.log('6. Installing PWA:\n');
    await app.installPWA();
    
    console.log('='.repeat(70) + '\n');
    
    // Summary
    console.log('SERVICE WORKERS & PWA SUMMARY:\n');
    
    console.log('Service Worker Capabilities:');
    console.log('  • Offline caching (Cache API)');
    console.log('  • Network request interception');
    console.log('  • Background sync');
    console.log('  • Push notifications');
    console.log('  • Periodic background sync');
    
    console.log('\nPWA Features:');
    console.log('  • Install to home screen');
    console.log('  • Standalone app experience');
    console.log('  • Offline functionality');
    console.log('  • Fast loading (cached assets)');
    console.log('  • Push notifications');
    
    console.log('\nCaching Strategies:');
    console.log('  1. Cache First: Try cache, fallback to network');
    console.log('  2. Network First: Try network, fallback to cache');
    console.log('  3. Cache Only: Only serve from cache');
    console.log('  4. Network Only: Always fetch from network');
    console.log('  5. Stale While Revalidate: Serve cache, update in background');
    
    console.log('\nLifecycle Events:');
    console.log('  • install: Cache resources');
    console.log('  • activate: Clean up old caches');
    console.log('  • fetch: Intercept requests');
    console.log('  • sync: Background sync');
    console.log('  • push: Handle push notifications');
    
    console.log('\nBest Practices:');
    console.log('  ✓ Version your caches');
    console.log('  ✓ Clean up old caches on activate');
    console.log('  ✓ Use appropriate caching strategy');
    console.log('  ✓ Handle failed network requests');
    console.log('  ✓ Update service worker carefully');
    console.log('  ✓ Test offline functionality thoroughly');
    
    console.log('\nRequirements:');
    console.log('  • HTTPS (required for security)');
    console.log('  • Manifest file (for installation)');
    console.log('  • Icons (various sizes)');
    console.log('  • Service worker registration');
    
    console.log('\nReal Implementation:');
    console.log("  navigator.serviceWorker.register('/sw.js')");
    console.log("    .then(reg => console.log('Registered', reg))");
    console.log("    .catch(err => console.error('Failed', err));");
    
})();
```

### Service Worker Caching Strategies:

| Strategy | When to Use | Example |
|----------|-------------|---------|
| **Cache First** | Static assets | Images, CSS, JS |
| **Network First** | Dynamic data | API calls, user data |
| **Cache Only** | Offline-only | Cached pages |
| **Network Only** | Always fresh | Real-time data |
| **Stale While Revalidate** | Good UX | News, feeds |

### Key Takeaways:

**Web Workers**:
- Run JavaScript in background threads
- Enable parallel processing
- Perfect for CPU-intensive tasks
- No DOM access, message-based communication

**IndexedDB**:
- Client-side NoSQL database
- Large storage capacity (50MB+)
- Asynchronous, transactional
- Perfect for offline-first apps

**Service Workers**:
- Network proxy for PWAs
- Enable offline functionality
- Push notifications & background sync
- HTTPS only, powerful caching

---

**End of Questions 25-27**
