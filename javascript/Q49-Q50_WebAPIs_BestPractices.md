# Questions 49-50: Advanced Topics & Best Practices

## Question 49: Explain Web APIs, Browser APIs, and their usage in modern applications

### Answer:

**Web APIs** provide interfaces for interacting with browsers, devices, and external services. They enable rich, interactive web applications with features like storage, notifications, geolocation, and more.

### Key Web APIs:
1. **Storage APIs** - LocalStorage, SessionStorage, IndexedDB
2. **Fetch API** - HTTP requests
3. **WebSocket API** - Real-time communication
4. **Geolocation API** - Location data
5. **Notification API** - Push notifications
6. **Web Workers** - Background threads

### Banking Scenario: Web APIs at Emirates NBD

```javascript
console.log('=== Web APIs - Emirates NBD ===\n');

// =============================================================================
// 1. LOCAL STORAGE API
// =============================================================================

console.log('1. LOCAL STORAGE:\n');

class SessionManager {
    static saveSession(key, data) {
        try {
            const serialized = JSON.stringify({
                data,
                timestamp: new Date().toISOString(),
                expiresIn: 3600000 // 1 hour
            });
            localStorage.setItem(key, serialized);
            console.log(`  ✓ Session saved: ${key}`);
        } catch (error) {
            console.log(`  ✗ Failed to save session: ${error.message}`);
        }
    }
    
    static getSession(key) {
        try {
            const item = localStorage.getItem(key);
            if (!item) return null;
            
            const { data, timestamp, expiresIn } = JSON.parse(item);
            const age = Date.now() - new Date(timestamp).getTime();
            
            if (age > expiresIn) {
                console.log(`  Session expired: ${key}`);
                localStorage.removeItem(key);
                return null;
            }
            
            console.log(`  ✓ Session retrieved: ${key}`);
            return data;
        } catch (error) {
            console.log(`  ✗ Failed to get session: ${error.message}`);
            return null;
        }
    }
    
    static clearSession(key) {
        localStorage.removeItem(key);
        console.log(`  ✓ Session cleared: ${key}`);
    }
    
    static clearAllSessions() {
        localStorage.clear();
        console.log(`  ✓ All sessions cleared`);
    }
}

console.log('Saving user session:');
SessionManager.saveSession('user', {
    accountNumber: 'ACC-123456789',
    name: 'John Doe',
    lastLogin: new Date()
});
console.log();

console.log('Retrieving user session:');
const userData = SessionManager.getSession('user');
if (userData) {
    console.log(`  Account: ${userData.accountNumber}`);
    console.log(`  Name: ${userData.name}\n`);
}

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. INDEXED DB (FOR LARGE DATA)
// =============================================================================

console.log('2. INDEXED DB:\n');

class TransactionDatabase {
    constructor() {
        this.dbName = 'BankingDB';
        this.version = 1;
        this.db = null;
    }
    
    async init() {
        // Note: IndexedDB is available in browsers, simulated here
        console.log('  Initializing IndexedDB...');
        console.log('  ✓ Database opened');
        console.log('  ✓ Object stores created: transactions, accounts\n');
        
        // Simulated structure
        this.data = {
            transactions: [],
            accounts: []
        };
    }
    
    async saveTransaction(transaction) {
        console.log(`  Saving transaction: ${transaction.id}`);
        this.data.transactions.push(transaction);
        console.log(`  ✓ Transaction saved\n`);
    }
    
    async getTransactionsByAccount(accountNumber) {
        console.log(`  Querying transactions for: ${accountNumber}`);
        const transactions = this.data.transactions.filter(
            t => t.accountNumber === accountNumber
        );
        console.log(`  ✓ Found ${transactions.length} transactions\n`);
        return transactions;
    }
    
    async getAllTransactions() {
        console.log(`  Fetching all transactions`);
        console.log(`  ✓ Found ${this.data.transactions.length} transactions\n`);
        return this.data.transactions;
    }
}

(async () => {
    console.log('Using IndexedDB for transaction history:\n');
    
    const db = new TransactionDatabase();
    await db.init();
    
    await db.saveTransaction({
        id: 'TXN-001',
        accountNumber: 'ACC-123456789',
        type: 'DEPOSIT',
        amount: 5000,
        timestamp: new Date()
    });
    
    const transactions = await db.getTransactionsByAccount('ACC-123456789');
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 3. FETCH API
    // =============================================================================
    
    console.log('3. FETCH API:\n');
    
    class BankingAPI {
        constructor(baseURL) {
            this.baseURL = baseURL;
        }
        
        async request(endpoint, options = {}) {
            const config = {
                headers: {
                    'Content-Type': 'application/json',
                    'Authorization': 'Bearer token123',
                    ...options.headers
                },
                ...options
            };
            
            console.log(`  ${config.method || 'GET'} ${this.baseURL}${endpoint}`);
            
            // Simulated response
            await new Promise(resolve => setTimeout(resolve, 100));
            
            // Mock response based on endpoint
            if (endpoint.includes('/accounts/')) {
                return {
                    accountNumber: 'ACC-123456789',
                    balance: 50000,
                    type: 'SAVINGS'
                };
            } else if (endpoint.includes('/transactions')) {
                return [
                    { id: 'TXN-001', amount: 5000, type: 'DEPOSIT' },
                    { id: 'TXN-002', amount: 2000, type: 'WITHDRAWAL' }
                ];
            }
            
            return { success: true };
        }
        
        async getAccount(accountNumber) {
            console.log('  Fetching account data...');
            const data = await this.request(`/accounts/${accountNumber}`);
            console.log(`  ✓ Account retrieved\n`);
            return data;
        }
        
        async getTransactions(accountNumber, params = {}) {
            console.log('  Fetching transactions...');
            const queryString = new URLSearchParams(params).toString();
            const endpoint = `/accounts/${accountNumber}/transactions${queryString ? '?' + queryString : ''}`;
            const data = await this.request(endpoint);
            console.log(`  ✓ ${data.length} transactions retrieved\n`);
            return data;
        }
        
        async transfer(from, to, amount) {
            console.log('  Processing transfer...');
            const data = await this.request('/transactions/transfer', {
                method: 'POST',
                body: JSON.stringify({ from, to, amount })
            });
            console.log(`  ✓ Transfer completed\n`);
            return data;
        }
    }
    
    console.log('Making API requests:\n');
    
    const api = new BankingAPI('https://api.emiratesnbd.com');
    
    const account = await api.getAccount('ACC-123456789');
    console.log(`Account balance: ${account.balance}\n`);
    
    const transactions = await api.getTransactions('ACC-123456789', {
        startDate: '2024-01-01',
        endDate: '2024-12-31'
    });
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 4. WEBSOCKET API (REAL-TIME UPDATES)
    // =============================================================================
    
    console.log('4. WEBSOCKET API:\n');
    
    class TransactionMonitor {
        constructor(url) {
            this.url = url;
            this.listeners = {
                connected: [],
                transaction: [],
                error: [],
                disconnected: []
            };
        }
        
        connect() {
            console.log(`  Connecting to ${this.url}...`);
            
            // Simulated WebSocket connection
            setTimeout(() => {
                console.log('  ✓ Connected to transaction monitor\n');
                this.trigger('connected');
                
                // Simulate receiving transactions
                this.simulateTransactions();
            }, 100);
        }
        
        simulateTransactions() {
            const transactions = [
                { id: 'TXN-101', amount: 1000, type: 'DEPOSIT', account: 'ACC-123456789' },
                { id: 'TXN-102', amount: 500, type: 'WITHDRAWAL', account: 'ACC-987654321' }
            ];
            
            transactions.forEach((txn, index) => {
                setTimeout(() => {
                    this.trigger('transaction', txn);
                }, (index + 1) * 1000);
            });
        }
        
        on(event, callback) {
            if (this.listeners[event]) {
                this.listeners[event].push(callback);
            }
        }
        
        trigger(event, data) {
            if (this.listeners[event]) {
                this.listeners[event].forEach(callback => callback(data));
            }
        }
        
        disconnect() {
            console.log('  Disconnecting...');
            console.log('  ✓ Disconnected\n');
            this.trigger('disconnected');
        }
    }
    
    console.log('Real-time transaction monitoring:\n');
    
    const monitor = new TransactionMonitor('wss://api.emiratesnbd.com/transactions');
    
    monitor.on('connected', () => {
        console.log('  Monitoring started...\n');
    });
    
    monitor.on('transaction', (txn) => {
        console.log(`  📊 New transaction: ${txn.id}`);
        console.log(`     Type: ${txn.type}, Amount: ${txn.amount}, Account: ${txn.account}\n`);
    });
    
    monitor.connect();
    
    await new Promise(resolve => setTimeout(resolve, 2500));
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 5. GEOLOCATION API
    // =============================================================================
    
    console.log('5. GEOLOCATION API:\n');
    
    class ATMLocator {
        async getCurrentLocation() {
            console.log('  Requesting location permission...');
            
            // Simulated geolocation
            await new Promise(resolve => setTimeout(resolve, 100));
            
            const position = {
                coords: {
                    latitude: 25.2048,
                    longitude: 55.2708,
                    accuracy: 10
                },
                timestamp: Date.now()
            };
            
            console.log(`  ✓ Location obtained`);
            console.log(`    Lat: ${position.coords.latitude}`);
            console.log(`    Long: ${position.coords.longitude}\n`);
            
            return position;
        }
        
        calculateDistance(lat1, lon1, lat2, lon2) {
            // Haversine formula
            const R = 6371; // Earth's radius in km
            const dLat = (lat2 - lat1) * Math.PI / 180;
            const dLon = (lon2 - lon1) * Math.PI / 180;
            
            const a = Math.sin(dLat/2) * Math.sin(dLat/2) +
                     Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) *
                     Math.sin(dLon/2) * Math.sin(dLon/2);
            
            const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
            return R * c;
        }
        
        async findNearestATMs(count = 3) {
            console.log('  Finding nearest ATMs...\n');
            
            const position = await this.getCurrentLocation();
            
            // Mock ATM locations
            const atms = [
                { id: 'ATM-001', name: 'Dubai Mall', lat: 25.1972, lon: 55.2744 },
                { id: 'ATM-002', name: 'Mall of Emirates', lat: 25.1186, lon: 55.2005 },
                { id: 'ATM-003', name: 'Burj Khalifa', lat: 25.1972, lon: 55.2744 },
                { id: 'ATM-004', name: 'Dubai Marina', lat: 25.0805, lon: 55.1403 }
            ];
            
            const atmDistances = atms.map(atm => ({
                ...atm,
                distance: this.calculateDistance(
                    position.coords.latitude,
                    position.coords.longitude,
                    atm.lat,
                    atm.lon
                )
            }));
            
            const nearest = atmDistances
                .sort((a, b) => a.distance - b.distance)
                .slice(0, count);
            
            console.log('  Nearest ATMs:\n');
            nearest.forEach((atm, index) => {
                console.log(`  ${index + 1}. ${atm.name}`);
                console.log(`     Distance: ${atm.distance.toFixed(2)} km\n`);
            });
            
            return nearest;
        }
    }
    
    const locator = new ATMLocator();
    await locator.findNearestATMs(3);
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 6. NOTIFICATION API
    // =============================================================================
    
    console.log('6. NOTIFICATION API:\n');
    
    class NotificationService {
        async requestPermission() {
            console.log('  Requesting notification permission...');
            
            // Simulated permission request
            await new Promise(resolve => setTimeout(resolve, 100));
            
            console.log('  ✓ Permission granted\n');
            return 'granted';
        }
        
        async showNotification(title, options = {}) {
            console.log('  Showing notification:');
            console.log(`    Title: ${title}`);
            
            if (options.body) {
                console.log(`    Body: ${options.body}`);
            }
            
            if (options.icon) {
                console.log(`    Icon: ${options.icon}`);
            }
            
            if (options.badge) {
                console.log(`    Badge: ${options.badge}`);
            }
            
            console.log();
        }
        
        async notifyTransaction(transaction) {
            await this.showNotification('Transaction Alert', {
                body: `${transaction.type}: ${transaction.amount} AED`,
                icon: '/icons/transaction.png',
                badge: '/icons/badge.png',
                data: transaction
            });
        }
        
        async notifyLowBalance(account, threshold) {
            await this.showNotification('Low Balance Alert', {
                body: `Your balance (${account.balance} AED) is below ${threshold} AED`,
                icon: '/icons/warning.png',
                badge: '/icons/badge.png'
            });
        }
    }
    
    console.log('Push notifications:\n');
    
    const notifier = new NotificationService();
    await notifier.requestPermission();
    
    await notifier.notifyTransaction({
        type: 'WITHDRAWAL',
        amount: 5000,
        account: 'ACC-123456789'
    });
    
    await notifier.notifyLowBalance(
        { accountNumber: 'ACC-123456789', balance: 800 },
        1000
    );
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 7. WEB WORKERS (BACKGROUND PROCESSING)
    // =============================================================================
    
    console.log('7. WEB WORKERS:\n');
    
    class TransactionProcessor {
        processTransactions(transactions) {
            console.log('  Processing transactions in background...\n');
            
            // Simulate heavy computation
            const result = transactions.map(txn => {
                // Complex calculations
                const fee = txn.amount * 0.001;
                const tax = fee * 0.05;
                const total = txn.amount + fee + tax;
                
                return {
                    ...txn,
                    fee: fee.toFixed(2),
                    tax: tax.toFixed(2),
                    total: total.toFixed(2),
                    processed: true
                };
            });
            
            return result;
        }
        
        async processAsync(transactions) {
            // Simulated Web Worker processing
            return new Promise(resolve => {
                setTimeout(() => {
                    const result = this.processTransactions(transactions);
                    console.log(`  ✓ Processed ${result.length} transactions\n`);
                    resolve(result);
                }, 200);
            });
        }
    }
    
    console.log('Background transaction processing:\n');
    
    const processor = new TransactionProcessor();
    
    const transactionsToProcess = [
        { id: 'TXN-001', amount: 5000, type: 'DEPOSIT' },
        { id: 'TXN-002', amount: 3000, type: 'WITHDRAWAL' },
        { id: 'TXN-003', amount: 10000, type: 'TRANSFER' }
    ];
    
    const processed = await processor.processAsync(transactionsToProcess);
    
    console.log('Processed transactions:');
    processed.forEach(txn => {
        console.log(`  ${txn.id}: Amount ${txn.amount}, Fee ${txn.fee}, Total ${txn.total}`);
    });
    console.log();
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 8. INTERSECTION OBSERVER (LAZY LOADING)
    // =============================================================================
    
    console.log('8. INTERSECTION OBSERVER:\n');
    
    class LazyLoader {
        constructor() {
            this.loaded = new Set();
        }
        
        observe(element) {
            console.log(`  Observing element: ${element.id}`);
            
            // Simulated intersection
            setTimeout(() => {
                this.loadElement(element);
            }, 100);
        }
        
        loadElement(element) {
            if (this.loaded.has(element.id)) return;
            
            console.log(`  ✓ Loading content for: ${element.id}`);
            this.loaded.add(element.id);
            
            // Simulate loading transaction data
            if (element.type === 'transaction-list') {
                console.log(`    Fetching transactions...`);
                console.log(`    ✓ Transactions loaded\n`);
            }
        }
    }
    
    console.log('Lazy loading transaction history:\n');
    
    const loader = new LazyLoader();
    
    loader.observe({ id: 'transactions-2024-01', type: 'transaction-list' });
    loader.observe({ id: 'transactions-2024-02', type: 'transaction-list' });
    
    await new Promise(resolve => setTimeout(resolve, 300));
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // WEB APIs SUMMARY
    // =============================================================================
    
    console.log('WEB APIs SUMMARY:\n');
    
    console.log('Storage APIs:');
    console.log('  • localStorage - Persistent key-value storage (5-10 MB)');
    console.log('  • sessionStorage - Session-scoped storage');
    console.log('  • IndexedDB - Large structured data (>50 MB)');
    console.log('  • Cache API - Offline caching\n');
    
    console.log('Communication APIs:');
    console.log('  • Fetch API - HTTP requests (modern alternative to XHR)');
    console.log('  • WebSocket - Real-time bidirectional communication');
    console.log('  • Server-Sent Events - Server push updates');
    console.log('  • WebRTC - Peer-to-peer communication\n');
    
    console.log('Device APIs:');
    console.log('  • Geolocation - Location data');
    console.log('  • Camera API - Photo/video capture');
    console.log('  • Vibration API - Device vibration');
    console.log('  • Battery API - Battery status\n');
    
    console.log('Notification APIs:');
    console.log('  • Notification API - System notifications');
    console.log('  • Push API - Push notifications');
    console.log('  • Badging API - App badge updates\n');
    
    console.log('Performance APIs:');
    console.log('  • Web Workers - Background threads');
    console.log('  • Service Workers - Offline functionality');
    console.log('  • Intersection Observer - Element visibility');
    console.log('  • Performance API - Timing metrics\n');
    
    console.log('Best Practices:');
    console.log('  ✓ Request permissions responsibly');
    console.log('  ✓ Handle API availability gracefully');
    console.log('  ✓ Use appropriate storage for data size');
    console.log('  ✓ Implement error handling');
    console.log('  ✓ Consider offline scenarios');
    console.log('  ✓ Optimize for performance');
    console.log('  ✓ Respect user privacy');
    console.log('  ✓ Clean up resources (listeners, connections)');
    
    console.log();
})();
```

---

## Question 50: Explain JavaScript best practices, code quality, and modern development workflows

### Answer:

**Best Practices** ensure code is maintainable, scalable, and performant. Following established patterns and conventions leads to better collaboration and fewer bugs.

### Key Areas:
1. **Code Quality** - Clean, readable, maintainable code
2. **Performance** - Optimization techniques
3. **Security** - Secure coding practices
4. **Tooling** - Linters, formatters, bundlers
5. **Workflows** - Git, CI/CD, testing

### Banking Scenario: Best Practices at Emirates NBD

```javascript
console.log('=== JavaScript Best Practices - Emirates NBD ===\n');

// =============================================================================
// 1. CODE ORGANIZATION AND STRUCTURE
// =============================================================================

console.log('1. CODE ORGANIZATION:\n');

// ❌ BAD: Global variables, unclear structure
var balance = 50000;
var accountNo = 'ACC-123';

function deposit(amt) {
    balance = balance + amt;
}

console.log('Bad practices demonstrated (see code comments)\n');

// ✅ GOOD: Encapsulated, clear structure
class BankAccount {
    #balance; // Private field
    
    constructor(accountNumber, initialBalance = 0) {
        this.accountNumber = accountNumber;
        this.#balance = initialBalance;
        this.transactions = [];
    }
    
    deposit(amount) {
        if (!this.#isValidAmount(amount)) {
            throw new Error('Invalid amount');
        }
        
        this.#balance += amount;
        this.#recordTransaction('DEPOSIT', amount);
        return this.#balance;
    }
    
    withdraw(amount) {
        if (!this.#isValidAmount(amount)) {
            throw new Error('Invalid amount');
        }
        
        if (amount > this.#balance) {
            throw new Error('Insufficient funds');
        }
        
        this.#balance -= amount;
        this.#recordTransaction('WITHDRAWAL', amount);
        return this.#balance;
    }
    
    getBalance() {
        return this.#balance;
    }
    
    #isValidAmount(amount) {
        return typeof amount === 'number' && amount > 0;
    }
    
    #recordTransaction(type, amount) {
        this.transactions.push({
            type,
            amount,
            balance: this.#balance,
            timestamp: new Date()
        });
    }
}

const account = new BankAccount('ACC-123456789', 50000);
console.log('Good practices:');
console.log('  ✓ Encapsulation with classes');
console.log('  ✓ Private fields with #');
console.log('  ✓ Validation methods');
console.log('  ✓ Clear naming conventions\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. NAMING CONVENTIONS
// =============================================================================

console.log('2. NAMING CONVENTIONS:\n');

// ✅ GOOD: Descriptive, consistent naming
const MAX_TRANSACTION_AMOUNT = 100000; // Constants: UPPER_SNAKE_CASE
const DEFAULT_INTEREST_RATE = 0.035;

class TransactionService { // Classes: PascalCase
    constructor() {
        this.transactionQueue = []; // Properties: camelCase
        this.isProcessing = false;
    }
    
    processTransaction(transaction) { // Methods: camelCase
        // Clear, descriptive names
        const isValid = this.validateTransaction(transaction);
        const hasMinimumBalance = this.checkMinimumBalance(transaction);
        
        return isValid && hasMinimumBalance;
    }
    
    validateTransaction(transaction) {
        return transaction.amount <= MAX_TRANSACTION_AMOUNT;
    }
    
    checkMinimumBalance(transaction) {
        return true; // Simplified
    }
}

console.log('Naming best practices:');
console.log('  • Classes: PascalCase (TransactionService)');
console.log('  • Methods/functions: camelCase (processTransaction)');
console.log('  • Constants: UPPER_SNAKE_CASE (MAX_AMOUNT)');
console.log('  • Private fields: #privateField');
console.log('  • Boolean names: is/has/can prefix (isValid, hasBalance)\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. FUNCTION DESIGN
// =============================================================================

console.log('3. FUNCTION DESIGN:\n');

// ❌ BAD: Long function, multiple responsibilities
function processBadTransaction(acc, amt, type, fee, tax) {
    // Too many parameters, doing too much
    const total = amt + fee + tax;
    acc.balance -= total;
    console.log('Transaction processed');
    // Send email
    // Update database
    // Log transaction
    return acc.balance;
}

// ✅ GOOD: Single responsibility, clear purpose
class Transaction {
    constructor(accountNumber, amount, type) {
        this.accountNumber = accountNumber;
        this.amount = amount;
        this.type = type;
        this.timestamp = new Date();
    }
    
    calculateFees() {
        const fee = this.amount * 0.001;
        const tax = fee * 0.05;
        return { fee, tax, total: this.amount + fee + tax };
    }
}

class TransactionProcessor {
    process(transaction) {
        this.validate(transaction);
        this.execute(transaction);
        this.notify(transaction);
        this.log(transaction);
    }
    
    validate(transaction) {
        // Validation logic
        console.log('  ✓ Validated');
    }
    
    execute(transaction) {
        // Execution logic
        console.log('  ✓ Executed');
    }
    
    notify(transaction) {
        // Notification logic
        console.log('  ✓ Notified');
    }
    
    log(transaction) {
        // Logging logic
        console.log('  ✓ Logged');
    }
}

console.log('Function best practices:');
console.log('  • Single Responsibility Principle');
console.log('  • Keep functions small (<20 lines)');
console.log('  • Limit parameters (max 3-4)');
console.log('  • Use descriptive names');
console.log('  • Return early for validation\n');

const txn = new Transaction('ACC-123', 5000, 'DEPOSIT');
const processor = new TransactionProcessor();
console.log('\nProcessing transaction:');
processor.process(txn);
console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. ERROR HANDLING
// =============================================================================

console.log('4. ERROR HANDLING:\n');

// ❌ BAD: Silent failures, generic errors
function badWithdraw(account, amount) {
    if (account.balance >= amount) {
        account.balance -= amount;
        return account.balance;
    }
    return null; // Silent failure
}

// ✅ GOOD: Explicit errors, specific error types
class BankingError extends Error {
    constructor(message, code, details = {}) {
        super(message);
        this.name = 'BankingError';
        this.code = code;
        this.details = details;
    }
}

class AccountService {
    withdraw(account, amount) {
        // Input validation
        if (!account) {
            throw new BankingError(
                'Account is required',
                'INVALID_INPUT',
                { parameter: 'account' }
            );
        }
        
        if (typeof amount !== 'number' || amount <= 0) {
            throw new BankingError(
                'Invalid amount',
                'INVALID_AMOUNT',
                { amount, type: typeof amount }
            );
        }
        
        // Business validation
        if (account.balance < amount) {
            throw new BankingError(
                'Insufficient funds',
                'INSUFFICIENT_FUNDS',
                { required: amount, available: account.balance }
            );
        }
        
        // Execute
        account.balance -= amount;
        return account.balance;
    }
}

console.log('Error handling best practices:');
console.log('  • Use custom error classes');
console.log('  • Include error codes');
console.log('  • Provide context/details');
console.log('  • Validate early');
console.log('  • Never swallow errors\n');

const accountService = new AccountService();
try {
    accountService.withdraw({ balance: 1000 }, 2000);
} catch (error) {
    console.log(`Error caught: ${error.code} - ${error.message}`);
    console.log(`Details: ${JSON.stringify(error.details)}\n`);
}

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. PERFORMANCE OPTIMIZATION
// =============================================================================

console.log('5. PERFORMANCE:\n');

// ❌ BAD: Inefficient operations
function badSearchTransactions(transactions, accountNumber) {
    const results = [];
    for (let i = 0; i < transactions.length; i++) {
        if (transactions[i].accountNumber === accountNumber) {
            results.push(transactions[i]);
        }
    }
    return results;
}

// ✅ GOOD: Optimized with proper data structures
class TransactionIndex {
    constructor() {
        this.byAccount = new Map(); // O(1) lookups
        this.byType = new Map();
        this.byDate = new Map();
    }
    
    addTransaction(transaction) {
        // Index by account
        if (!this.byAccount.has(transaction.accountNumber)) {
            this.byAccount.set(transaction.accountNumber, []);
        }
        this.byAccount.get(transaction.accountNumber).push(transaction);
        
        // Index by type
        if (!this.byType.has(transaction.type)) {
            this.byType.set(transaction.type, []);
        }
        this.byType.get(transaction.type).push(transaction);
    }
    
    findByAccount(accountNumber) {
        return this.byAccount.get(accountNumber) || [];
    }
    
    findByType(type) {
        return this.byType.get(type) || [];
    }
}

console.log('Performance best practices:\n');

// Memoization
function createMemoizedCalculator() {
    const cache = new Map();
    
    return function calculateInterest(balance, rate, years) {
        const key = `${balance}-${rate}-${years}`;
        
        if (cache.has(key)) {
            console.log('  Cache hit');
            return cache.get(key);
        }
        
        console.log('  Computing...');
        const result = balance * Math.pow(1 + rate, years) - balance;
        cache.set(key, result);
        return result;
    };
}

const memoizedCalc = createMemoizedCalculator();
console.log('Memoization:');
console.log(`  Interest: ${memoizedCalc(100000, 0.035, 5).toFixed(2)}`);
console.log(`  Interest: ${memoizedCalc(100000, 0.035, 5).toFixed(2)}`); // From cache
console.log();

// Debouncing
function debounce(fn, delay) {
    let timeoutId;
    return function(...args) {
        clearTimeout(timeoutId);
        timeoutId = setTimeout(() => fn.apply(this, args), delay);
    };
}

const searchAccounts = debounce((query) => {
    console.log(`  Searching for: ${query}`);
}, 300);

console.log('Debouncing search:');
searchAccounts('ACC');
searchAccounts('ACC-1');
searchAccounts('ACC-12');
setTimeout(() => {
    console.log('  Only last search executed\n');
}, 400);

setTimeout(() => {
    console.log('Performance tips:');
    console.log('  • Use appropriate data structures (Map, Set)');
    console.log('  • Implement memoization for expensive calculations');
    console.log('  • Debounce frequent operations');
    console.log('  • Avoid nested loops where possible');
    console.log('  • Use const/let instead of var');
    console.log('  • Minimize DOM manipulation\n');
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 6. CODE DOCUMENTATION
    // =============================================================================
    
    console.log('6. DOCUMENTATION:\n');
    
    /**
     * Processes a transfer between two bank accounts.
     * 
     * @param {string} fromAccount - Source account number (format: ACC-XXXXXXXXX)
     * @param {string} toAccount - Destination account number (format: ACC-XXXXXXXXX)
     * @param {number} amount - Transfer amount in AED (must be positive)
     * @param {Object} options - Additional options
     * @param {boolean} options.urgent - Whether this is an urgent transfer
     * @param {string} options.reference - Transfer reference/note
     * 
     * @returns {Promise<Object>} Transaction result with id and status
     * 
     * @throws {BankingError} If validation fails or insufficient funds
     * 
     * @example
     * const result = await processTransfer(
     *   'ACC-123456789',
     *   'ACC-987654321',
     *   5000,
     *   { urgent: true, reference: 'Invoice payment' }
     * );
     */
    async function processTransfer(fromAccount, toAccount, amount, options = {}) {
        // Implementation
        return {
            id: 'TXN-001',
            status: 'COMPLETED',
            amount,
            from: fromAccount,
            to: toAccount
        };
    }
    
    console.log('Documentation best practices:');
    console.log('  • Use JSDoc comments for public APIs');
    console.log('  • Describe parameters and return values');
    console.log('  • Include examples');
    console.log('  • Document edge cases and errors');
    console.log('  • Keep comments up to date');
    console.log('  • Explain "why", not "what"\n');
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 7. MODERN JAVASCRIPT FEATURES
    // =============================================================================
    
    console.log('7. MODERN FEATURES:\n');
    
    // Destructuring
    const transaction1 = { id: 'TXN-001', amount: 5000, type: 'DEPOSIT' };
    const { id, amount, type } = transaction1;
    console.log('Destructuring:');
    console.log(`  ${id}: ${type} ${amount}\n`);
    
    // Spread operator
    const baseTransaction = { timestamp: new Date(), status: 'PENDING' };
    const depositTransaction = { ...baseTransaction, type: 'DEPOSIT', amount: 5000 };
    console.log('Spread operator:');
    console.log(`  Created transaction with base properties\n`);
    
    // Optional chaining
    const user = { account: { balance: 50000 } };
    const balance = user?.account?.balance ?? 0;
    console.log('Optional chaining:');
    console.log(`  Balance: ${balance}\n`);
    
    // Nullish coalescing
    const defaultAmount = null ?? 1000;
    const zeroAmount = 0 ?? 1000; // 0 is falsy but not nullish
    console.log('Nullish coalescing:');
    console.log(`  Default: ${defaultAmount}, Zero: ${zeroAmount}\n`);
    
    // Template literals
    const accountNumber = 'ACC-123456789';
    const balance2 = 50000;
    console.log('Template literals:');
    console.log(`  Account ${accountNumber} has ${balance2} AED\n`);
    
    // Array methods
    const transactions = [
        { amount: 5000, type: 'DEPOSIT' },
        { amount: 2000, type: 'WITHDRAWAL' },
        { amount: 10000, type: 'DEPOSIT' }
    ];
    
    const totalDeposits = transactions
        .filter(t => t.type === 'DEPOSIT')
        .reduce((sum, t) => sum + t.amount, 0);
    
    console.log('Array methods:');
    console.log(`  Total deposits: ${totalDeposits}\n`);
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 8. SECURITY BEST PRACTICES
    // =============================================================================
    
    console.log('8. SECURITY:\n');
    
    class SecurityService {
        // Input sanitization
        sanitizeInput(input) {
            if (typeof input !== 'string') return input;
            
            // Remove potentially dangerous characters
            return input
                .replace(/[<>]/g, '') // Remove < >
                .replace(/javascript:/gi, '') // Remove javascript: protocol
                .trim();
        }
        
        // Validate account number format
        validateAccountNumber(accountNumber) {
            const sanitized = this.sanitizeInput(accountNumber);
            
            if (!/^ACC-\d{9}$/.test(sanitized)) {
                throw new BankingError('Invalid account number format', 'INVALID_FORMAT');
            }
            
            return sanitized;
        }
        
        // Hash sensitive data (simulated)
        hashPassword(password) {
            // In production, use bcrypt or similar
            console.log('  Hashing password...');
            return `hashed_${password}`;
        }
        
        // Validate and sanitize transaction amount
        validateAmount(amount) {
            const parsed = parseFloat(amount);
            
            if (isNaN(parsed) || parsed <= 0) {
                throw new BankingError('Invalid amount', 'INVALID_AMOUNT');
            }
            
            if (parsed > 1000000) {
                throw new BankingError('Amount exceeds limit', 'LIMIT_EXCEEDED');
            }
            
            return parsed;
        }
    }
    
    const security = new SecurityService();
    
    console.log('Security practices:');
    console.log('  • Sanitize all user input');
    console.log('  • Validate data formats');
    console.log('  • Hash sensitive data');
    console.log('  • Use HTTPS for API calls');
    console.log('  • Implement rate limiting');
    console.log('  • Use Content Security Policy');
    console.log('  • Avoid eval() and Function()');
    console.log('  • Use secure random numbers\n');
    
    console.log('Input sanitization example:');
    const malicious = '<script>alert("xss")</script>';
    console.log(`  Input: ${malicious}`);
    console.log(`  Sanitized: ${security.sanitizeInput(malicious)}\n`);
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // BEST PRACTICES SUMMARY
    // =============================================================================
    
    console.log('BEST PRACTICES SUMMARY:\n');
    
    console.log('Code Quality:');
    console.log('  ✓ Use ESLint for code linting');
    console.log('  ✓ Use Prettier for formatting');
    console.log('  ✓ Follow consistent style guide');
    console.log('  ✓ Write self-documenting code');
    console.log('  ✓ Keep functions small and focused');
    console.log('  ✓ Use meaningful variable names\n');
    
    console.log('Architecture:');
    console.log('  ✓ SOLID principles');
    console.log('  ✓ Separation of concerns');
    console.log('  ✓ DRY (Don\'t Repeat Yourself)');
    console.log('  ✓ KISS (Keep It Simple, Stupid)');
    console.log('  ✓ YAGNI (You Aren\'t Gonna Need It)\n');
    
    console.log('Performance:');
    console.log('  ✓ Use appropriate data structures');
    console.log('  ✓ Implement caching/memoization');
    console.log('  ✓ Optimize loops and iterations');
    console.log('  ✓ Debounce/throttle frequent operations');
    console.log('  ✓ Lazy load resources');
    console.log('  ✓ Use Web Workers for heavy tasks\n');
    
    console.log('Security:');
    console.log('  ✓ Validate and sanitize input');
    console.log('  ✓ Use HTTPS for communications');
    console.log('  ✓ Implement authentication/authorization');
    console.log('  ✓ Avoid XSS and injection attacks');
    console.log('  ✓ Use CSP headers');
    console.log('  ✓ Keep dependencies updated\n');
    
    console.log('Testing:');
    console.log('  ✓ Write unit tests (70% coverage)');
    console.log('  ✓ Integration tests for workflows');
    console.log('  ✓ E2E tests for critical paths');
    console.log('  ✓ Test edge cases and errors');
    console.log('  ✓ Use continuous integration\n');
    
    console.log('Development Workflow:');
    console.log('  ✓ Use version control (Git)');
    console.log('  ✓ Write meaningful commit messages');
    console.log('  ✓ Code reviews before merging');
    console.log('  ✓ Use feature branches');
    console.log('  ✓ Automate testing and deployment');
    console.log('  ✓ Document APIs and changes\n');
    
    console.log('Modern Tooling:');
    console.log('  • Bundlers: Webpack, Vite, Rollup');
    console.log('  • Transpilers: Babel, TypeScript');
    console.log('  • Linters: ESLint');
    console.log('  • Formatters: Prettier');
    console.log('  • Testing: Jest, Vitest, Cypress');
    console.log('  • Build: npm scripts, Task runners');
    console.log('  • CI/CD: GitHub Actions, GitLab CI\n');
    
    console.log('Remember:');
    console.log('  "Any fool can write code that a computer can understand.');
    console.log('   Good programmers write code that humans can understand."');
    console.log('   - Martin Fowler\n');
    
    console.log('='.repeat(70) + '\n');
    
    console.log('🎉 CONGRATULATIONS! 🎉\n');
    console.log('You have completed all 50 JavaScript interview questions!\n');
    
    console.log('Topics Covered:');
    console.log('  ✓ Core Fundamentals (Q1-Q6)');
    console.log('  ✓ ES6+ Features (Q7-Q9)');
    console.log('  ✓ OOP & Prototypes (Q10-Q12)');
    console.log('  ✓ Async Programming (Q13-Q15, Q40-Q42, Q46)');
    console.log('  ✓ Functional Programming (Q19-Q21)');
    console.log('  ✓ Design Patterns (Q22-Q24, Q45)');
    console.log('  ✓ Testing & Security (Q28-Q30, Q47-Q48)');
    console.log('  ✓ Advanced Topics (Q31-Q39, Q43-Q44)');
    console.log('  ✓ Web APIs & Best Practices (Q49-Q50)\n');
    
    console.log('You are now well-prepared for JavaScript interviews!');
    console.log('Good luck! 🚀');
}, 500);
```

### Key Takeaways:

**Web APIs**:
- localStorage/IndexedDB for data persistence
- Fetch API for HTTP requests
- WebSocket for real-time communication
- Geolocation for location services
- Notification API for push notifications
- Web Workers for background processing

**Best Practices**:
- Write clean, maintainable code
- Follow SOLID principles
- Implement proper error handling
- Optimize for performance
- Ensure security (input validation, sanitization)
- Use modern JavaScript features
- Document code appropriately
- Test thoroughly
- Use proper tooling

**Development Workflow**:
- Version control with Git
- Code reviews
- Automated testing
- Continuous integration
- Linting and formatting
- Build optimization
- Deployment automation

---

**🎉 END OF 50 JAVASCRIPT INTERVIEW QUESTIONS 🎉**

All questions completed with comprehensive examples using Emirates NBD banking scenarios!
