# JavaScript Interview Questions (Q7-Q9): ES6+ Features

## Q7: Explain destructuring, spread/rest operators with banking examples

**Answer:**

Destructuring and spread/rest operators are ES6+ features that simplify working with arrays and objects.

**Banking Account Management with Destructuring:**

```javascript
// Object Destructuring - Account Information
const account = {
    accountNumber: 'ENBD123456789',
    accountHolder: 'Ahmed Al Mansouri',
    accountType: 'SAVINGS',
    balance: 25000,
    currency: 'AED',
    branch: {
        code: 'DXB001',
        name: 'Dubai Mall Branch',
        location: {
            city: 'Dubai',
            country: 'UAE'
        }
    },
    contact: {
        email: 'ahmed@example.com',
        phone: '+971501234567'
    }
};

// Basic destructuring
const { accountNumber, accountHolder, balance } = account;
console.log(`Account: ${accountNumber}, Holder: ${accountHolder}, Balance: AED ${balance}`);

// Nested destructuring
const { 
    branch: { name: branchName, location: { city } },
    contact: { email }
} = account;
console.log(`Branch: ${branchName}, City: ${city}, Email: ${email}`);

// Destructuring with default values
const { 
    overdraftLimit = 0, 
    creditLimit = 0,
    accountStatus = 'ACTIVE'
} = account;
console.log(`Overdraft: ${overdraftLimit}, Credit: ${creditLimit}, Status: ${accountStatus}`);

// Destructuring with renaming
const { 
    accountNumber: accNum,
    accountHolder: holder,
    balance: currentBalance
} = account;
console.log(`${accNum}: ${holder} has AED ${currentBalance}`);

// Array Destructuring - Transaction History
const transactions = [
    { id: 1, type: 'CREDIT', amount: 5000, date: '2024-01-15' },
    { id: 2, type: 'DEBIT', amount: 1200, date: '2024-01-16' },
    { id: 3, type: 'CREDIT', amount: 3000, date: '2024-01-17' }
];

// Get first, second, and rest
const [firstTxn, secondTxn, ...remainingTxns] = transactions;
console.log('First:', firstTxn);
console.log('Second:', secondTxn);
console.log('Remaining:', remainingTxns.length);

// Skip elements
const [, , thirdTxn] = transactions;
console.log('Third transaction:', thirdTxn);

// Function parameter destructuring
class BankingService {
    // Destructure object parameters
    createAccount({ accountHolder, accountType, initialBalance = 0, currency = 'AED' }) {
        console.log(`\nCreating account for ${accountHolder}`);
        console.log(`Type: ${accountType}, Balance: ${currency} ${initialBalance}`);
        
        return {
            accountNumber: `ENBD${Date.now()}`,
            accountHolder,
            accountType,
            balance: initialBalance,
            currency,
            createdAt: new Date().toISOString()
        };
    }
    
    // Destructure array parameters
    processTransactions([first, ...rest]) {
        console.log(`\nProcessing ${rest.length + 1} transactions`);
        console.log('First transaction:', first);
        
        rest.forEach(txn => {
            console.log(`- ${txn.type}: AED ${txn.amount}`);
        });
    }
}

const service = new BankingService();
const newAccount = service.createAccount({
    accountHolder: 'Fatima Hassan',
    accountType: 'CURRENT',
    initialBalance: 10000
});

// Spread Operator - Combining Objects
console.log('\n=== SPREAD OPERATOR ===\n');

// Merge account details
const basicInfo = {
    accountNumber: 'ENBD001',
    accountHolder: 'Omar Abdullah'
};

const financialInfo = {
    balance: 50000,
    currency: 'AED',
    accountType: 'SAVINGS'
};

const additionalInfo = {
    phoneNumber: '+971509876543',
    email: 'omar@example.com'
};

// Combine all information
const completeAccount = {
    ...basicInfo,
    ...financialInfo,
    ...additionalInfo,
    status: 'ACTIVE',
    createdAt: new Date().toISOString()
};

console.log('Complete Account:', completeAccount);

// Override properties with spread
const updatedAccount = {
    ...completeAccount,
    balance: 75000,  // Override balance
    lastUpdated: new Date().toISOString()
};

console.log('Updated Balance:', updatedAccount.balance);

// Spread with Arrays - Transaction Processing
const morningTransactions = [
    { id: 1, amount: 1000, time: '09:00' },
    { id: 2, amount: 2000, time: '10:30' }
];

const afternoonTransactions = [
    { id: 3, amount: 1500, time: '14:00' },
    { id: 4, amount: 3000, time: '16:45' }
];

// Combine transactions
const allTransactions = [
    ...morningTransactions,
    ...afternoonTransactions
];

console.log('\nAll transactions:', allTransactions.length);

// Add new transaction
const updatedTransactions = [
    ...allTransactions,
    { id: 5, amount: 500, time: '18:00' }
];

console.log('Updated transactions:', updatedTransactions.length);

// Rest Operator - Function Parameters
console.log('\n=== REST OPERATOR ===\n');

class TransactionProcessor {
    // Rest parameters for multiple accounts
    calculateTotalBalance(...accounts) {
        console.log(`Calculating total for ${accounts.length} accounts`);
        
        const total = accounts.reduce((sum, account) => {
            return sum + (account.balance || 0);
        }, 0);
        
        return total;
    }
    
    // Rest with other parameters
    transferFunds(fromAccount, toAccount, ...amounts) {
        console.log(`\nTransferring from ${fromAccount.accountNumber} to ${toAccount.accountNumber}`);
        
        const totalAmount = amounts.reduce((sum, amount) => sum + amount, 0);
        
        if (fromAccount.balance < totalAmount) {
            throw new Error('Insufficient funds');
        }
        
        fromAccount.balance -= totalAmount;
        toAccount.balance += totalAmount;
        
        console.log(`Transferred AED ${totalAmount} in ${amounts.length} transactions`);
        amounts.forEach((amount, i) => {
            console.log(`  ${i + 1}. AED ${amount}`);
        });
        
        return { fromAccount, toAccount, totalAmount };
    }
    
    // Collect transaction data
    recordTransactions(accountId, ...transactionData) {
        console.log(`\nRecording ${transactionData.length} transactions for ${accountId}`);
        
        return transactionData.map((data, index) => ({
            id: index + 1,
            accountId,
            ...data,
            timestamp: new Date().toISOString()
        }));
    }
}

const processor = new TransactionProcessor();

// Calculate total balance
const acc1 = { accountNumber: 'ACC001', balance: 10000 };
const acc2 = { accountNumber: 'ACC002', balance: 25000 };
const acc3 = { accountNumber: 'ACC003', balance: 15000 };

const totalBalance = processor.calculateTotalBalance(acc1, acc2, acc3);
console.log(`Total balance: AED ${totalBalance}`);

// Multiple transfers
const sourceAcc = { accountNumber: 'SRC001', balance: 20000 };
const destAcc = { accountNumber: 'DST001', balance: 5000 };

processor.transferFunds(sourceAcc, destAcc, 3000, 2000, 1500);

// Record multiple transactions
const recorded = processor.recordTransactions(
    'ACC001',
    { type: 'CREDIT', amount: 1000, description: 'Salary' },
    { type: 'DEBIT', amount: 500, description: 'Utilities' },
    { type: 'CREDIT', amount: 2000, description: 'Transfer' }
);

console.log('\nRecorded transactions:', recorded);

// Advanced: Banking API Response Handler
class BankingAPIHandler {
    // Extract and process response data
    processAccountResponse({
        data: {
            account: {
                accountNumber,
                balance,
                ...accountDetails
            },
            transactions: [latestTransaction, ...olderTransactions] = []
        },
        metadata: { timestamp, requestId } = {}
    }) {
        console.log('\n=== API Response Processing ===\n');
        console.log(`Account: ${accountNumber}, Balance: AED ${balance}`);
        console.log('Other details:', accountDetails);
        console.log('Latest transaction:', latestTransaction);
        console.log(`${olderTransactions.length} older transactions`);
        console.log('Metadata:', { timestamp, requestId });
        
        return {
            accountNumber,
            balance,
            latestTransaction,
            transactionCount: olderTransactions.length + 1
        };
    }
    
    // Merge multiple account responses
    mergeAccountData(...responses) {
        console.log(`\nMerging ${responses.length} account responses`);
        
        return responses.map(response => {
            const { data, metadata, ...rest } = response;
            return {
                ...data,
                ...metadata,
                ...rest
            };
        });
    }
}

// Real-world Banking Scenario: Account Summary Generator
class AccountSummaryGenerator {
    generateSummary(accountData) {
        // Destructure with defaults and renaming
        const {
            accountNumber: accNo,
            accountHolder: { firstName, lastName, ...contactInfo },
            balance: { available = 0, pending = 0, total = 0 } = {},
            transactions: {
                recent = [],
                pending: pendingTxns = [],
                ...otherTxns
            } = {},
            settings: { notifications = true, overdraftProtection = false } = {}
        } = accountData;
        
        // Create summary with spread
        const summary = {
            account: {
                number: accNo,
                holder: `${firstName} ${lastName}`,
                ...contactInfo
            },
            balances: {
                available,
                pending,
                total,
                currency: 'AED'
            },
            recentActivity: [...recent],
            pendingTransactions: [...pendingTxns],
            settings: {
                notifications,
                overdraftProtection,
                lastUpdated: new Date().toISOString()
            }
        };
        
        return summary;
    }
    
    // Combine multiple summaries
    combineSummaries(...summaries) {
        return summaries.reduce((combined, summary) => ({
            ...combined,
            accounts: [
                ...(combined.accounts || []),
                summary.account
            ],
            totalBalance: (combined.totalBalance || 0) + summary.balances.total
        }), {});
    }
}

// Usage
const summaryGen = new AccountSummaryGenerator();

const sampleData = {
    accountNumber: 'ENBD987654321',
    accountHolder: {
        firstName: 'Ahmed',
        lastName: 'Al Mansouri',
        email: 'ahmed@example.com',
        phone: '+971501234567'
    },
    balance: {
        available: 45000,
        pending: 2000,
        total: 47000
    },
    transactions: {
        recent: [
            { id: 1, amount: 5000, type: 'CREDIT' },
            { id: 2, amount: 1200, type: 'DEBIT' }
        ],
        pending: [
            { id: 3, amount: 2000, type: 'CREDIT' }
        ]
    },
    settings: {
        notifications: true,
        overdraftProtection: true
    }
};

const summary = summaryGen.generateSummary(sampleData);
console.log('\n=== Account Summary ===\n');
console.log(JSON.stringify(summary, null, 2));
```

---

## Q8: Explain Template Literals and Symbol with banking notification system

**Answer:**

Template literals provide string interpolation and multi-line strings. Symbols create unique identifiers for object properties.

**Banking Notification System with Template Literals:**

```javascript
// Template Literals for Banking Messages
class BankingNotificationService {
    
    // Basic template literal
    generateWelcomeMessage(customerName, accountNumber) {
        return `Dear ${customerName},

Welcome to Emirates NBD!

Your account ${accountNumber} has been successfully created.

We're delighted to have you as our valued customer.

Best regards,
Emirates NBD Team`;
    }
    
    // Multi-line transaction receipt
    generateTransactionReceipt(transaction) {
        const {
            transactionId,
            type,
            amount,
            accountNumber,
            timestamp,
            balance
        } = transaction;
        
        return `
╔════════════════════════════════════════════════╗
║         EMIRATES NBD - TRANSACTION RECEIPT      ║
╠════════════════════════════════════════════════╣
║ Transaction ID: ${transactionId.padEnd(28)} ║
║ Type:           ${type.padEnd(28)} ║
║ Amount:         AED ${amount.toFixed(2).padStart(23)} ║
║ Account:        ${accountNumber.padEnd(28)} ║
║ Date/Time:      ${new Date(timestamp).toLocaleString().padEnd(28)} ║
║ New Balance:    AED ${balance.toFixed(2).padStart(23)} ║
╚════════════════════════════════════════════════╝
        `.trim();
    }
    
    // Expression interpolation
    generateBalanceAlert(accountHolder, balance, threshold) {
        const percentage = ((balance / threshold) * 100).toFixed(1);
        const status = balance < threshold ? 'LOW BALANCE ALERT' : 'BALANCE OK';
        
        return `
🏦 ${status}

Hello ${accountHolder},

Your account balance is AED ${balance.toLocaleString()}
${balance < threshold 
    ? `⚠️  This is below your threshold of AED ${threshold.toLocaleString()}`
    : `✓ This is above your threshold of AED ${threshold.toLocaleString()}`
}

Current: ${percentage}% of threshold
${balance < threshold 
    ? `Please add AED ${(threshold - balance).toLocaleString()} to reach threshold.`
    : 'Your account is in good standing.'
}
        `.trim();
    }
    
    // Tagged template literals
    formatCurrency(strings, ...values) {
        return strings.reduce((result, string, i) => {
            const value = values[i];
            const formatted = typeof value === 'number' 
                ? `AED ${value.toLocaleString('en-AE', { minimumFractionDigits: 2, maximumFractionDigits: 2 })}`
                : value;
            
            return result + string + (formatted || '');
        }, '');
    }
    
    // SQL query with template literal (safe with parameterization)
    generateTransactionQuery(accountId, startDate, endDate) {
        return `
            SELECT 
                t.transaction_id,
                t.transaction_type,
                t.amount,
                t.currency,
                t.description,
                t.timestamp,
                a.account_number,
                a.account_holder
            FROM transactions t
            JOIN accounts a ON t.account_id = a.account_id
            WHERE t.account_id = $1
            AND t.timestamp BETWEEN $2 AND $3
            ORDER BY t.timestamp DESC
            LIMIT 50;
        `.trim();
    }
    
    // HTML email template
    generateEmailTemplate(customerName, transactions) {
        const transactionRows = transactions.map(txn => `
            <tr>
                <td>${new Date(txn.timestamp).toLocaleDateString()}</td>
                <td>${txn.description}</td>
                <td class="${txn.type.toLowerCase()}">${txn.type}</td>
                <td style="text-align: right;">AED ${txn.amount.toFixed(2)}</td>
            </tr>
        `).join('');
        
        return `
<!DOCTYPE html>
<html>
<head>
    <style>
        body { font-family: Arial, sans-serif; }
        .header { background-color: #c41e3a; color: white; padding: 20px; }
        table { width: 100%; border-collapse: collapse; margin: 20px 0; }
        th, td { padding: 10px; text-align: left; border-bottom: 1px solid #ddd; }
        .credit { color: green; }
        .debit { color: red; }
    </style>
</head>
<body>
    <div class="header">
        <h1>Emirates NBD</h1>
        <h2>Monthly Statement</h2>
    </div>
    
    <p>Dear ${customerName},</p>
    
    <p>Here is your monthly transaction summary:</p>
    
    <table>
        <thead>
            <tr>
                <th>Date</th>
                <th>Description</th>
                <th>Type</th>
                <th>Amount</th>
            </tr>
        </thead>
        <tbody>
            ${transactionRows}
        </tbody>
    </table>
    
    <p>Thank you for banking with Emirates NBD.</p>
</body>
</html>
        `.trim();
    }
}

// Usage examples
const notificationService = new BankingNotificationService();

console.log('\n=== Welcome Message ===\n');
console.log(notificationService.generateWelcomeMessage('Ahmed Al Mansouri', 'ENBD123456789'));

console.log('\n=== Transaction Receipt ===\n');
console.log(notificationService.generateTransactionReceipt({
    transactionId: 'TXN20240115001',
    type: 'CREDIT',
    amount: 5000,
    accountNumber: 'ENBD123456789',
    timestamp: new Date().toISOString(),
    balance: 25000
}));

console.log('\n=== Balance Alert ===\n');
console.log(notificationService.generateBalanceAlert('Fatima Hassan', 800, 1000));

console.log('\n=== Tagged Template ===\n');
const amount1 = 15000;
const amount2 = 3500;
const message = notificationService.formatCurrency`You transferred ${amount1} and received ${amount2}`;
console.log(message);

// Symbols for Banking System
console.log('\n=== SYMBOLS ===\n');

// Private properties with Symbols
const _accountNumber = Symbol('accountNumber');
const _pin = Symbol('pin');
const _balance = Symbol('balance');

class SecureBankAccount {
    constructor(accountNumber, pin, initialBalance) {
        this[_accountNumber] = accountNumber;
        this[_pin] = pin;
        this[_balance] = initialBalance;
    }
    
    // Public methods to access private data
    getAccountNumber() {
        return this[_accountNumber];
    }
    
    verifyPin(inputPin) {
        return this[_pin] === inputPin;
    }
    
    getBalance(inputPin) {
        if (this.verifyPin(inputPin)) {
            return this[_balance];
        }
        throw new Error('Invalid PIN');
    }
    
    withdraw(amount, inputPin) {
        if (!this.verifyPin(inputPin)) {
            throw new Error('Invalid PIN');
        }
        
        if (amount > this[_balance]) {
            throw new Error('Insufficient funds');
        }
        
        this[_balance] -= amount;
        console.log(`Withdrew AED ${amount}. New balance: AED ${this[_balance]}`);
        return this[_balance];
    }
    
    deposit(amount) {
        this[_balance] += amount;
        console.log(`Deposited AED ${amount}. New balance: AED ${this[_balance]}`);
        return this[_balance];
    }
}

const secureAccount = new SecureBankAccount('ENBD001', '1234', 10000);

console.log('Account Number:', secureAccount.getAccountNumber());
console.log('Balance:', secureAccount.getBalance('1234'));

// Try to access private properties
console.log('Direct access to balance:', secureAccount[_balance]); // undefined from outside
console.log('Keys:', Object.keys(secureAccount)); // []
console.log('Symbols:', Object.getOwnPropertySymbols(secureAccount)); // Shows symbols

// Well-known Symbols
console.log('\n=== Well-known Symbols ===\n');

class TransactionCollection {
    constructor(transactions) {
        this.transactions = transactions;
    }
    
    // Symbol.iterator for for...of loop
    [Symbol.iterator]() {
        let index = 0;
        const transactions = this.transactions;
        
        return {
            next() {
                if (index < transactions.length) {
                    return { value: transactions[index++], done: false };
                }
                return { done: true };
            }
        };
    }
    
    // Symbol.toStringTag for custom toString
    get [Symbol.toStringTag]() {
        return 'TransactionCollection';
    }
}

const txnCollection = new TransactionCollection([
    { id: 1, amount: 1000, type: 'CREDIT' },
    { id: 2, amount: 500, type: 'DEBIT' },
    { id: 3, amount: 2000, type: 'CREDIT' }
]);

console.log('Collection type:', txnCollection.toString()); // [object TransactionCollection]

console.log('\nIterating transactions:');
for (const txn of txnCollection) {
    console.log(`${txn.type}: AED ${txn.amount}`);
}

// Symbol as unique event identifiers
const ACCOUNT_EVENTS = {
    CREATED: Symbol('account.created'),
    UPDATED: Symbol('account.updated'),
    CLOSED: Symbol('account.closed'),
    BALANCE_LOW: Symbol('account.balance.low'),
    OVERDRAFT: Symbol('account.overdraft')
};

class AccountEventEmitter {
    constructor() {
        this.listeners = new Map();
    }
    
    on(event, callback) {
        if (!this.listeners.has(event)) {
            this.listeners.set(event, []);
        }
        this.listeners.get(event).push(callback);
    }
    
    emit(event, data) {
        if (this.listeners.has(event)) {
            this.listeners.get(event).forEach(callback => callback(data));
        }
    }
}

const eventEmitter = new AccountEventEmitter();

eventEmitter.on(ACCOUNT_EVENTS.BALANCE_LOW, (data) => {
    console.log(`\n⚠️  Low Balance Alert: Account ${data.accountNumber} has AED ${data.balance}`);
});

eventEmitter.on(ACCOUNT_EVENTS.OVERDRAFT, (data) => {
    console.log(`\n🚨 Overdraft Alert: Account ${data.accountNumber} is AED ${Math.abs(data.balance)} overdrawn`);
});

// Trigger events
eventEmitter.emit(ACCOUNT_EVENTS.BALANCE_LOW, {
    accountNumber: 'ENBD001',
    balance: 500
});

eventEmitter.emit(ACCOUNT_EVENTS.OVERDRAFT, {
    accountNumber: 'ENBD002',
    balance: -250
});
```

---

## Q9: Explain Map, Set, WeakMap, and WeakSet with banking cache system

**Answer:**

Map and Set are collections with better performance than objects and arrays for certain operations. WeakMap and WeakSet allow garbage collection of keys/values.

**Banking Data Structures:**

```javascript
// MAP - Account Cache System
class AccountCacheService {
    constructor() {
        // Map for account data (key-value pairs with any type as key)
        this.accountCache = new Map();
        this.accessCount = new Map();
        this.maxCacheSize = 1000;
    }
    
    // Cache account data
    cacheAccount(accountNumber, accountData) {
        // Map allows any type as key
        this.accountCache.set(accountNumber, {
            ...accountData,
            cachedAt: Date.now()
        });
        
        // Track access count
        this.accessCount.set(accountNumber, 0);
        
        // Implement LRU cache
        if (this.accountCache.size > this.maxCacheSize) {
            const firstKey = this.accountCache.keys().next().value;
            this.accountCache.delete(firstKey);
            this.accessCount.delete(firstKey);
        }
        
        console.log(`Cached account: ${accountNumber}`);
    }
    
    // Get cached account
    getAccount(accountNumber) {
        if (this.accountCache.has(accountNumber)) {
            // Increment access count
            const count = this.accessCount.get(accountNumber);
            this.accessCount.set(accountNumber, count + 1);
            
            const cached = this.accountCache.get(accountNumber);
            const age = Date.now() - cached.cachedAt;
            
            console.log(`Cache hit for ${accountNumber} (age: ${age}ms, accesses: ${count + 1})`);
            return cached;
        }
        
        console.log(`Cache miss for ${accountNumber}`);
        return null;
    }
    
    // Clear expired cache entries
    clearExpired(maxAge = 300000) { // 5 minutes
        const now = Date.now();
        let cleared = 0;
        
        for (const [accountNumber, data] of this.accountCache.entries()) {
            if (now - data.cachedAt > maxAge) {
                this.accountCache.delete(accountNumber);
                this.accessCount.delete(accountNumber);
                cleared++;
            }
        }
        
        console.log(`Cleared ${cleared} expired entries`);
    }
    
    // Get cache statistics
    getStats() {
        return {
            size: this.accountCache.size,
            maxSize: this.maxCacheSize,
            accounts: Array.from(this.accountCache.keys()),
            mostAccessed: Array.from(this.accessCount.entries())
                .sort((a, b) => b[1] - a[1])
                .slice(0, 5)
                .map(([acc, count]) => ({ account: acc, accesses: count }))
        };
    }
}

// Usage
console.log('=== MAP: Account Cache ===\n');

const cacheService = new AccountCacheService();

cacheService.cacheAccount('ACC001', {
    accountHolder: 'Ahmed',
    balance: 10000,
    type: 'SAVINGS'
});

cacheService.cacheAccount('ACC002', {
    accountHolder: 'Fatima',
    balance: 25000,
    type: 'CURRENT'
});

// Access accounts
cacheService.getAccount('ACC001');
cacheService.getAccount('ACC001'); // Hit
cacheService.getAccount('ACC002');
cacheService.getAccount('ACC003'); // Miss

console.log('\nCache Stats:', cacheService.getStats());

// SET - Unique Transaction IDs
console.log('\n=== SET: Transaction Tracking ===\n');

class TransactionTracker {
    constructor() {
        // Set automatically handles uniqueness
        this.processedTransactions = new Set();
        this.failedTransactions = new Set();
        this.duplicateAttempts = 0;
    }
    
    // Process transaction (idempotent)
    processTransaction(transactionId) {
        // Check if already processed
        if (this.processedTransactions.has(transactionId)) {
            this.duplicateAttempts++;
            console.log(`⚠️  Duplicate transaction detected: ${transactionId}`);
            return { success: true, duplicate: true };
        }
        
        // Simulate processing
        const success = Math.random() > 0.2; // 80% success rate
        
        if (success) {
            this.processedTransactions.add(transactionId);
            console.log(`✓ Processed: ${transactionId}`);
            return { success: true, duplicate: false };
        } else {
            this.failedTransactions.add(transactionId);
            console.log(`✗ Failed: ${transactionId}`);
            return { success: false, duplicate: false };
        }
    }
    
    // Retry failed transactions
    retryFailed() {
        console.log(`\nRetrying ${this.failedTransactions.size} failed transactions...`);
        
        const toRetry = Array.from(this.failedTransactions);
        this.failedTransactions.clear();
        
        toRetry.forEach(txnId => {
            this.processTransaction(txnId);
        });
    }
    
    // Check if transaction was processed
    isProcessed(transactionId) {
        return this.processedTransactions.has(transactionId);
    }
    
    // Get statistics
    getStats() {
        return {
            processed: this.processedTransactions.size,
            failed: this.failedTransactions.size,
            duplicateAttempts: this.duplicateAttempts,
            uniqueTransactions: this.processedTransactions.size + this.failedTransactions.size
        };
    }
    
    // Set operations
    compareWithAnotherTracker(otherTracker) {
        const thisSet = this.processedTransactions;
        const otherSet = otherTracker.processedTransactions;
        
        // Union: all transactions from both
        const union = new Set([...thisSet, ...otherSet]);
        
        // Intersection: common transactions
        const intersection = new Set(
            Array.from(thisSet).filter(x => otherSet.has(x))
        );
        
        // Difference: in this but not in other
        const difference = new Set(
            Array.from(thisSet).filter(x => !otherSet.has(x))
        );
        
        return {
            union: union.size,
            intersection: intersection.size,
            difference: difference.size
        };
    }
}

const tracker = new TransactionTracker();

// Process transactions
const txnIds = ['TXN001', 'TXN002', 'TXN003', 'TXN004', 'TXN005'];
txnIds.forEach(id => tracker.processTransaction(id));

// Try duplicate
tracker.processTransaction('TXN001');
tracker.processTransaction('TXN002');

console.log('\nTracker Stats:', tracker.getStats());

// WEAKMAP - Session Management
console.log('\n=== WEAKMAP: Session Management ===\n');

class SessionManager {
    constructor() {
        // WeakMap: keys are objects, automatically garbage collected
        this.sessions = new WeakMap();
        this.sessionData = new WeakMap();
    }
    
    // Create session for user object
    createSession(userObject, sessionData) {
        const sessionId = `SESSION_${Date.now()}_${Math.random()}`;
        
        this.sessions.set(userObject, {
            sessionId,
            createdAt: Date.now(),
            lastAccess: Date.now()
        });
        
        this.sessionData.set(userObject, sessionData);
        
        console.log(`Session created for user: ${sessionId}`);
        return sessionId;
    }
    
    // Get session
    getSession(userObject) {
        if (this.sessions.has(userObject)) {
            const session = this.sessions.get(userObject);
            session.lastAccess = Date.now();
            return session;
        }
        return null;
    }
    
    // Get session data
    getSessionData(userObject) {
        return this.sessionData.get(userObject);
    }
    
    // Destroy session
    destroySession(userObject) {
        this.sessions.delete(userObject);
        this.sessionData.delete(userObject);
        console.log('Session destroyed');
    }
}

const sessionManager = new SessionManager();

// Create user objects
let user1 = { id: 'USER001', name: 'Ahmed' };
let user2 = { id: 'USER002', name: 'Fatima' };

sessionManager.createSession(user1, {
    accountNumber: 'ACC001',
    loginTime: new Date(),
    permissions: ['view', 'transfer']
});

sessionManager.createSession(user2, {
    accountNumber: 'ACC002',
    loginTime: new Date(),
    permissions: ['view']
});

console.log('User1 session:', sessionManager.getSession(user1));
console.log('User1 data:', sessionManager.getSessionData(user1));

// When user objects are garbage collected, sessions are automatically removed
user1 = null; // Session will be garbage collected

// WEAKSET - Active Connections Tracking
console.log('\n=== WEAKSET: Connection Tracking ===\n');

class ConnectionPool {
    constructor() {
        // WeakSet: automatically cleans up disconnected connections
        this.activeConnections = new WeakSet();
    }
    
    // Add connection
    addConnection(connectionObject) {
        this.activeConnections.add(connectionObject);
        console.log(`Connection added: ${connectionObject.id}`);
    }
    
    // Check if connection is active
    isActive(connectionObject) {
        return this.activeConnections.has(connectionObject);
    }
    
    // Remove connection
    removeConnection(connectionObject) {
        this.activeConnections.delete(connectionObject);
        console.log(`Connection removed: ${connectionObject.id}`);
    }
}

const connectionPool = new ConnectionPool();

let conn1 = { id: 'CONN001', socket: {} };
let conn2 = { id: 'CONN002', socket: {} };

connectionPool.addConnection(conn1);
connectionPool.addConnection(conn2);

console.log('conn1 active:', connectionPool.isActive(conn1)); // true
console.log('conn2 active:', connectionPool.isActive(conn2)); // true

conn1 = null; // Connection automatically removed from WeakSet

// Practical Banking Example: Rate Limiter with Map
console.log('\n=== Practical: Rate Limiter ===\n');

class RateLimiter {
    constructor(maxRequests = 10, windowMs = 60000) {
        this.requestMap = new Map();
        this.maxRequests = maxRequests;
        this.windowMs = windowMs;
    }
    
    // Check if request is allowed
    allowRequest(clientId) {
        const now = Date.now();
        
        if (!this.requestMap.has(clientId)) {
            this.requestMap.set(clientId, []);
        }
        
        const requests = this.requestMap.get(clientId);
        
        // Remove old requests outside window
        const validRequests = requests.filter(time => now - time < this.windowMs);
        
        if (validRequests.length >= this.maxRequests) {
            const oldestRequest = Math.min(...validRequests);
            const retryAfter = this.windowMs - (now - oldestRequest);
            
            console.log(`❌ Rate limit exceeded for ${clientId}. Retry after ${retryAfter}ms`);
            return { allowed: false, retryAfter };
        }
        
        validRequests.push(now);
        this.requestMap.set(clientId, validRequests);
        
        console.log(`✓ Request allowed for ${clientId} (${validRequests.length}/${this.maxRequests})`);
        return { allowed: true, remaining: this.maxRequests - validRequests.length };
    }
    
    // Clear old entries
    cleanup() {
        const now = Date.now();
        
        for (const [clientId, requests] of this.requestMap.entries()) {
            const validRequests = requests.filter(time => now - time < this.windowMs);
            
            if (validRequests.length === 0) {
                this.requestMap.delete(clientId);
            } else {
                this.requestMap.set(clientId, validRequests);
            }
        }
    }
}

const rateLimiter = new RateLimiter(5, 10000); // 5 requests per 10 seconds

// Simulate requests
for (let i = 0; i < 7; i++) {
    rateLimiter.allowRequest('CLIENT001');
}
```

---
