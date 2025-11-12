# JavaScript Interview Questions (Q4-Q6): Advanced Concepts

## Q4: Explain the Event Loop and asynchronous JavaScript with banking transaction processing

**Answer:**

The Event Loop is JavaScript's mechanism for handling asynchronous operations. It continuously checks the call stack and task queues to determine what code to execute next.

**Event Loop Components:**

1. **Call Stack**: Executes synchronous code
2. **Web APIs**: Handle async operations (setTimeout, fetch, etc.)
3. **Task Queue (Macro tasks)**: setTimeout, setInterval callbacks
4. **Microtask Queue**: Promises, queueMicrotask
5. **Event Loop**: Coordinates execution between stack and queues

**Priority**: Call Stack → Microtasks → Macrotasks

**Banking Transaction Processing Example:**

```javascript
// Transaction Queue Processor demonstrating Event Loop
class TransactionQueueProcessor {
    constructor() {
        this.pendingTransactions = [];
        this.processingQueue = [];
        this.completedTransactions = [];
    }
    
    // Add transaction to queue
    addTransaction(transaction) {
        console.log(`[SYNC] Adding transaction ${transaction.id} to queue`);
        this.pendingTransactions.push(transaction);
        return transaction.id;
    }
    
    // Process transaction (async)
    async processTransaction(transaction) {
        console.log(`[START] Processing transaction ${transaction.id}`);
        
        // Simulate database validation (macrotask - setTimeout)
        return new Promise((resolve) => {
            setTimeout(() => {
                console.log(`[TIMEOUT] Database validation for ${transaction.id}`);
                resolve(transaction);
            }, 1000);
        }).then((txn) => {
            // Promise then (microtask)
            console.log(`[MICROTASK] Validating amount for ${txn.id}`);
            return txn;
        }).then((txn) => {
            // Another microtask
            console.log(`[MICROTASK] Checking fraud for ${txn.id}`);
            return txn;
        });
    }
    
    // Demonstrate Event Loop execution order
    demonstrateEventLoop() {
        console.log('\n=== Event Loop Demonstration ===\n');
        
        console.log('1. SYNC: Start');
        
        // Macrotask (Task Queue)
        setTimeout(() => {
            console.log('4. MACROTASK: setTimeout callback');
        }, 0);
        
        // Microtask (Microtask Queue)
        Promise.resolve().then(() => {
            console.log('3. MICROTASK: Promise.then callback');
        });
        
        // Another microtask
        queueMicrotask(() => {
            console.log('3. MICROTASK: queueMicrotask callback');
        });
        
        console.log('2. SYNC: End');
        
        // Output order:
        // 1. SYNC: Start
        // 2. SYNC: End
        // 3. MICROTASK: Promise.then callback
        // 3. MICROTASK: queueMicrotask callback
        // 4. MACROTASK: setTimeout callback
    }
}

// Real Banking Scenario: Processing Multiple Transactions
class BankingTransactionProcessor {
    constructor() {
        this.transactionId = 0;
    }
    
    createTransaction(fromAccount, toAccount, amount) {
        return {
            id: ++this.transactionId,
            fromAccount,
            toAccount,
            amount,
            status: 'pending',
            timestamp: new Date().toISOString()
        };
    }
    
    // Simulate async operations with different priorities
    async processTransactionPipeline(transaction) {
        console.log(`\n[${transaction.id}] Starting transaction pipeline`);
        
        // Step 1: Immediate sync validation
        console.log(`[${transaction.id}] SYNC: Validating input data`);
        if (transaction.amount <= 0) {
            throw new Error('Invalid amount');
        }
        
        // Step 2: Check balance (Promise - microtask)
        await Promise.resolve().then(() => {
            console.log(`[${transaction.id}] MICROTASK: Checking account balance`);
        });
        
        // Step 3: Database query (setTimeout - macrotask)
        await new Promise(resolve => {
            setTimeout(() => {
                console.log(`[${transaction.id}] MACROTASK: Querying database`);
                resolve();
            }, 100);
        });
        
        // Step 4: Fraud detection (microtask)
        await Promise.resolve().then(() => {
            console.log(`[${transaction.id}] MICROTASK: Running fraud detection`);
        });
        
        // Step 5: Update accounts (macrotask)
        await new Promise(resolve => {
            setTimeout(() => {
                console.log(`[${transaction.id}] MACROTASK: Updating account balances`);
                transaction.status = 'completed';
                resolve();
            }, 100);
        });
        
        // Step 6: Send notification (microtask)
        await Promise.resolve().then(() => {
            console.log(`[${transaction.id}] MICROTASK: Sending notification`);
        });
        
        console.log(`[${transaction.id}] Transaction completed\n`);
        return transaction;
    }
    
    // Process multiple transactions showing event loop behavior
    async processBatch(transactions) {
        console.log('\n=== Processing Transaction Batch ===\n');
        
        const startTime = Date.now();
        
        // Sequential processing (one after another)
        console.log('--- Sequential Processing ---');
        for (const txn of transactions) {
            await this.processTransactionPipeline(txn);
        }
        
        const sequentialTime = Date.now() - startTime;
        console.log(`Sequential time: ${sequentialTime}ms\n`);
        
        // Parallel processing (all at once)
        console.log('--- Parallel Processing ---');
        const parallelStart = Date.now();
        
        await Promise.all(
            transactions.map(txn => this.processTransactionPipeline(txn))
        );
        
        const parallelTime = Date.now() - parallelStart;
        console.log(`Parallel time: ${parallelTime}ms`);
        console.log(`Speedup: ${(sequentialTime / parallelTime).toFixed(2)}x faster\n`);
    }
}

// Event Loop Visualization
function visualizeEventLoop() {
    console.log('\n=== Event Loop Visualization ===\n');
    
    console.log('Call Stack: [main()]');
    
    // Synchronous code
    console.log('Call Stack: [main() → console.log]');
    console.log('1. Executing sync code');
    
    // setTimeout (Macrotask)
    setTimeout(() => {
        console.log('5. Macrotask Queue → Call Stack → Executing setTimeout');
    }, 0);
    console.log('Call Stack: Registered setTimeout in Web API');
    
    // Promise (Microtask)
    Promise.resolve()
        .then(() => {
            console.log('3. Microtask Queue → Call Stack → First Promise');
        })
        .then(() => {
            console.log('4. Microtask Queue → Call Stack → Chained Promise');
        });
    console.log('Call Stack: Registered Promise in Microtask Queue');
    
    // More sync code
    console.log('2. More sync code');
    console.log('Call Stack: [main()] completed, checking Microtask Queue...');
}

// Banking Real-time Balance Update System
class RealtimeBalanceUpdater {
    constructor() {
        this.balance = 10000;
        this.updateQueue = [];
        this.isProcessing = false;
    }
    
    // Add balance update to queue
    queueBalanceUpdate(amount, type) {
        const update = {
            id: Date.now(),
            amount,
            type, // 'credit' or 'debit'
            timestamp: new Date().toISOString()
        };
        
        console.log(`Queuing ${type}: AED ${amount}`);
        this.updateQueue.push(update);
        
        // Process queue if not already processing
        if (!this.isProcessing) {
            this.processQueue();
        }
    }
    
    // Process updates from queue (demonstrates event loop)
    async processQueue() {
        if (this.updateQueue.length === 0) {
            this.isProcessing = false;
            return;
        }
        
        this.isProcessing = true;
        const update = this.updateQueue.shift();
        
        console.log(`\nProcessing update ${update.id}...`);
        
        // Simulate async database update
        await new Promise(resolve => setTimeout(resolve, 500));
        
        // Update balance
        if (update.type === 'credit') {
            this.balance += update.amount;
        } else {
            this.balance -= update.amount;
        }
        
        console.log(`Balance updated: AED ${this.balance}`);
        
        // Continue processing queue
        setImmediate(() => this.processQueue());
    }
    
    getBalance() {
        return this.balance;
    }
}

// Usage Examples
async function runEventLoopExamples() {
    // Example 1: Basic Event Loop
    const processor = new TransactionQueueProcessor();
    processor.demonstrateEventLoop();
    
    // Example 2: Transaction Pipeline
    await new Promise(resolve => setTimeout(resolve, 2000));
    
    const bankProcessor = new BankingTransactionProcessor();
    const txn1 = bankProcessor.createTransaction('ACC001', 'ACC002', 1000);
    const txn2 = bankProcessor.createTransaction('ACC003', 'ACC004', 2000);
    
    await bankProcessor.processBatch([txn1, txn2]);
    
    // Example 3: Event Loop Visualization
    await new Promise(resolve => setTimeout(resolve, 1000));
    visualizeEventLoop();
    
    // Example 4: Real-time Balance Updates
    await new Promise(resolve => setTimeout(resolve, 2000));
    
    console.log('\n=== Real-time Balance Updates ===\n');
    const balanceUpdater = new RealtimeBalanceUpdater();
    
    balanceUpdater.queueBalanceUpdate(500, 'credit');
    balanceUpdater.queueBalanceUpdate(200, 'debit');
    balanceUpdater.queueBalanceUpdate(1000, 'credit');
    
    await new Promise(resolve => setTimeout(resolve, 3000));
    console.log(`\nFinal balance: AED ${balanceUpdater.getBalance()}`);
}

// Run examples
runEventLoopExamples();
```

---

## Q5: Explain prototypal inheritance and the `this` keyword with banking class examples

**Answer:**

JavaScript uses prototypal inheritance where objects inherit properties from other objects. The `this` keyword refers to the object that is executing the current function.

**Prototypal Inheritance in Banking System:**

```javascript
// Base Account Constructor (ES5 style)
function BankAccount(accountNumber, accountHolder, balance) {
    this.accountNumber = accountNumber;
    this.accountHolder = accountHolder;
    this.balance = balance;
    this.transactions = [];
}

// Methods on prototype (shared across all instances)
BankAccount.prototype.deposit = function(amount) {
    if (amount <= 0) {
        throw new Error('Deposit amount must be positive');
    }
    
    this.balance += amount;
    this.transactions.push({
        type: 'CREDIT',
        amount,
        balance: this.balance,
        timestamp: new Date()
    });
    
    console.log(`Deposited AED ${amount}. New balance: AED ${this.balance}`);
    return this.balance;
};

BankAccount.prototype.withdraw = function(amount) {
    if (amount <= 0) {
        throw new Error('Withdrawal amount must be positive');
    }
    
    if (amount > this.balance) {
        throw new Error('Insufficient funds');
    }
    
    this.balance -= amount;
    this.transactions.push({
        type: 'DEBIT',
        amount,
        balance: this.balance,
        timestamp: new Date()
    });
    
    console.log(`Withdrew AED ${amount}. New balance: AED ${this.balance}`);
    return this.balance;
};

BankAccount.prototype.getBalance = function() {
    return this.balance;
};

BankAccount.prototype.getAccountInfo = function() {
    return `Account: ${this.accountNumber}, Holder: ${this.accountHolder}, Balance: AED ${this.balance}`;
};

// Savings Account inherits from BankAccount
function SavingsAccount(accountNumber, accountHolder, balance, interestRate) {
    // Call parent constructor
    BankAccount.call(this, accountNumber, accountHolder, balance);
    this.interestRate = interestRate;
    this.accountType = 'SAVINGS';
}

// Set up prototype chain
SavingsAccount.prototype = Object.create(BankAccount.prototype);
SavingsAccount.prototype.constructor = SavingsAccount;

// Add savings-specific method
SavingsAccount.prototype.calculateInterest = function() {
    const interest = this.balance * this.interestRate;
    console.log(`Interest calculated: AED ${interest.toFixed(2)} at ${this.interestRate * 100}%`);
    return interest;
};

SavingsAccount.prototype.applyInterest = function() {
    const interest = this.calculateInterest();
    this.deposit(interest); // Uses inherited method
    console.log(`Interest applied. New balance: AED ${this.balance}`);
};

// Override parent method
SavingsAccount.prototype.withdraw = function(amount) {
    const MIN_BALANCE = 1000;
    
    if (this.balance - amount < MIN_BALANCE) {
        throw new Error(`Savings account must maintain minimum balance of AED ${MIN_BALANCE}`);
    }
    
    // Call parent method
    BankAccount.prototype.withdraw.call(this, amount);
};

// Current Account inherits from BankAccount
function CurrentAccount(accountNumber, accountHolder, balance, overdraftLimit) {
    BankAccount.call(this, accountNumber, accountHolder, balance);
    this.overdraftLimit = overdraftLimit;
    this.accountType = 'CURRENT';
}

CurrentAccount.prototype = Object.create(BankAccount.prototype);
CurrentAccount.prototype.constructor = CurrentAccount;

// Override withdraw to allow overdraft
CurrentAccount.prototype.withdraw = function(amount) {
    const availableBalance = this.balance + this.overdraftLimit;
    
    if (amount > availableBalance) {
        throw new Error(`Exceeds overdraft limit. Available: AED ${availableBalance}`);
    }
    
    this.balance -= amount;
    this.transactions.push({
        type: 'DEBIT',
        amount,
        balance: this.balance,
        timestamp: new Date()
    });
    
    if (this.balance < 0) {
        console.log(`Withdrew AED ${amount}. Balance: AED ${this.balance} (Overdraft used)`);
    } else {
        console.log(`Withdrew AED ${amount}. New balance: AED ${this.balance}`);
    }
    
    return this.balance;
};

CurrentAccount.prototype.getOverdraftInfo = function() {
    const used = Math.max(0, -this.balance);
    const available = this.overdraftLimit - used;
    
    return {
        overdraftLimit: this.overdraftLimit,
        overdraftUsed: used,
        overdraftAvailable: available
    };
};

// ES6 Class Syntax (syntactic sugar over prototypes)
class ModernBankAccount {
    constructor(accountNumber, accountHolder, balance) {
        this.accountNumber = accountNumber;
        this.accountHolder = accountHolder;
        this.balance = balance;
        this.transactions = [];
    }
    
    deposit(amount) {
        if (amount <= 0) throw new Error('Invalid amount');
        
        this.balance += amount;
        this._recordTransaction('CREDIT', amount);
        return this.balance;
    }
    
    withdraw(amount) {
        if (amount <= 0) throw new Error('Invalid amount');
        if (amount > this.balance) throw new Error('Insufficient funds');
        
        this.balance -= amount;
        this._recordTransaction('DEBIT', amount);
        return this.balance;
    }
    
    _recordTransaction(type, amount) {
        this.transactions.push({
            type,
            amount,
            balance: this.balance,
            timestamp: new Date()
        });
    }
    
    getBalance() {
        return this.balance;
    }
}

// Modern inheritance with ES6 classes
class ModernSavingsAccount extends ModernBankAccount {
    constructor(accountNumber, accountHolder, balance, interestRate) {
        super(accountNumber, accountHolder, balance);
        this.interestRate = interestRate;
        this.accountType = 'SAVINGS';
    }
    
    calculateInterest() {
        return this.balance * this.interestRate;
    }
    
    applyInterest() {
        const interest = this.calculateInterest();
        this.deposit(interest);
        console.log(`Interest applied: AED ${interest.toFixed(2)}`);
    }
    
    // Override parent method
    withdraw(amount) {
        const MIN_BALANCE = 1000;
        
        if (this.balance - amount < MIN_BALANCE) {
            throw new Error(`Minimum balance AED ${MIN_BALANCE} required`);
        }
        
        return super.withdraw(amount);
    }
}

// Demonstrating 'this' keyword binding
class AccountManager {
    constructor(bankName) {
        this.bankName = bankName;
        this.accounts = [];
    }
    
    // Regular method - 'this' bound to instance
    addAccount(account) {
        this.accounts.push(account);
        console.log(`Account added to ${this.bankName}`);
    }
    
    // Arrow function - 'this' lexically bound
    getAllBalances = () => {
        return this.accounts.map(acc => ({
            accountNumber: acc.accountNumber,
            balance: acc.getBalance()
        }));
    }
    
    // Regular method used as callback
    processAccounts() {
        // Problem: 'this' is lost in callback
        this.accounts.forEach(function(account) {
            // console.log(this.bankName); // undefined!
            console.log(account.accountNumber);
        });
        
        // Solution 1: Arrow function
        this.accounts.forEach((account) => {
            console.log(`${this.bankName}: ${account.accountNumber}`);
        });
        
        // Solution 2: bind
        this.accounts.forEach(function(account) {
            console.log(`${this.bankName}: ${account.accountNumber}`);
        }.bind(this));
    }
    
    getTotalBalance() {
        // 'this' in reduce callback
        return this.accounts.reduce((total, account) => {
            return total + account.getBalance();
        }, 0);
    }
}

// 'this' keyword examples
function demonstrateThisKeyword() {
    console.log('\n=== Demonstrating "this" keyword ===\n');
    
    const account = new BankAccount('ACC001', 'Ahmed', 5000);
    
    // 1. Normal method call - 'this' is the object
    account.deposit(1000); // this === account
    
    // 2. Method reference loses context
    const depositFn = account.deposit;
    try {
        depositFn(500); // Error: 'this' is undefined
    } catch (e) {
        console.log('Error:', e.message);
    }
    
    // 3. bind() fixes the context
    const boundDeposit = account.deposit.bind(account);
    boundDeposit(500); // Works! 'this' is bound to account
    
    // 4. call() and apply()
    const anotherAccount = new BankAccount('ACC002', 'Fatima', 3000);
    account.deposit.call(anotherAccount, 1000); // 'this' is anotherAccount
    account.deposit.apply(anotherAccount, [500]); // Same with array
    
    // 5. Arrow functions inherit 'this'
    const accountMethods = {
        accountNumber: 'ACC003',
        balance: 10000,
        
        // Regular function
        getBalanceRegular: function() {
            setTimeout(function() {
                // console.log(this.balance); // undefined - 'this' is lost
            }, 100);
        },
        
        // Arrow function
        getBalanceArrow: function() {
            setTimeout(() => {
                console.log(`Balance: AED ${this.balance}`); // Works! 'this' preserved
            }, 100);
        }
    };
    
    accountMethods.getBalanceArrow();
}

// Usage Examples
console.log('=== Prototypal Inheritance Examples ===\n');

// ES5 Constructor Functions
const savingsAcc = new SavingsAccount('SAV001', 'Ahmed Al Mansouri', 10000, 0.025);
console.log(savingsAcc.getAccountInfo());
savingsAcc.deposit(5000);
savingsAcc.applyInterest();
console.log('');

const currentAcc = new CurrentAccount('CUR001', 'Fatima Hassan', 5000, 10000);
console.log(currentAcc.getAccountInfo());
currentAcc.withdraw(8000); // Uses overdraft
console.log('Overdraft Info:', currentAcc.getOverdraftInfo());
console.log('');

// ES6 Classes
const modernSavings = new ModernSavingsAccount('SAV002', 'Omar Abdullah', 20000, 0.03);
modernSavings.deposit(5000);
modernSavings.applyInterest();
console.log('');

// Prototype chain verification
console.log('=== Prototype Chain ===');
console.log('savingsAcc instanceof SavingsAccount:', savingsAcc instanceof SavingsAccount);
console.log('savingsAcc instanceof BankAccount:', savingsAcc instanceof BankAccount);
console.log('savingsAcc instanceof Object:', savingsAcc instanceof Object);

// 'this' keyword examples
demonstrateThisKeyword();
```

---

## Q6: Explain array and object manipulation methods with banking data processing

**Answer:**

JavaScript provides powerful methods for manipulating arrays and objects, essential for processing banking data.

**Banking Data Processing Examples:**

```javascript
// Sample banking data
const transactions = [
    { id: 1, accountId: 'ACC001', type: 'CREDIT', amount: 5000, date: '2024-01-15', category: 'SALARY' },
    { id: 2, accountId: 'ACC001', type: 'DEBIT', amount: 1200, date: '2024-01-16', category: 'RENT' },
    { id: 3, accountId: 'ACC002', type: 'CREDIT', amount: 3000, date: '2024-01-17', category: 'TRANSFER' },
    { id: 4, accountId: 'ACC001', type: 'DEBIT', amount: 500, date: '2024-01-18', category: 'GROCERIES' },
    { id: 5, accountId: 'ACC002', type: 'DEBIT', amount: 2000, date: '2024-01-19', category: 'SHOPPING' },
    { id: 6, accountId: 'ACC001', type: 'CREDIT', amount: 1500, date: '2024-01-20', category: 'FREELANCE' },
    { id: 7, accountId: 'ACC003', type: 'CREDIT', amount: 10000, date: '2024-01-21', category: 'SALARY' },
    { id: 8, accountId: 'ACC003', type: 'DEBIT', amount: 3500, date: '2024-01-22', category: 'LOAN_PAYMENT' }
];

const accounts = [
    { id: 'ACC001', holder: 'Ahmed Al Mansouri', type: 'SAVINGS', balance: 15000 },
    { id: 'ACC002', holder: 'Fatima Hassan', type: 'CURRENT', balance: 8000 },
    { id: 'ACC003', holder: 'Omar Abdullah', type: 'SAVINGS', balance: 25000 }
];

// Banking Data Processing Class
class BankingDataProcessor {
    
    // 1. MAP - Transform data
    static calculateTaxOnTransactions(transactions) {
        console.log('=== MAP: Calculate Tax ===\n');
        
        const TAX_RATE = 0.05;
        
        const transactionsWithTax = transactions.map(txn => ({
            ...txn,
            tax: txn.type === 'CREDIT' ? txn.amount * TAX_RATE : 0,
            netAmount: txn.type === 'CREDIT' ? txn.amount * (1 - TAX_RATE) : txn.amount
        }));
        
        console.log('Sample:', transactionsWithTax[0]);
        return transactionsWithTax;
    }
    
    // 2. FILTER - Extract specific data
    static getHighValueTransactions(transactions, threshold = 3000) {
        console.log('\n=== FILTER: High Value Transactions ===\n');
        
        const highValue = transactions.filter(txn => txn.amount >= threshold);
        
        console.log(`Found ${highValue.length} transactions >= AED ${threshold}`);
        highValue.forEach(txn => 
            console.log(`${txn.date}: ${txn.type} AED ${txn.amount} (${txn.category})`)
        );
        
        return highValue;
    }
    
    // 3. REDUCE - Aggregate data
    static calculateAccountBalance(transactions, accountId) {
        console.log('\n=== REDUCE: Calculate Balance ===\n');
        
        const balance = transactions
            .filter(txn => txn.accountId === accountId)
            .reduce((total, txn) => {
                return txn.type === 'CREDIT' 
                    ? total + txn.amount 
                    : total - txn.amount;
            }, 0);
        
        console.log(`Account ${accountId} balance: AED ${balance}`);
        return balance;
    }
    
    // 4. FIND - Search for specific transaction
    static findTransactionById(transactions, id) {
        console.log('\n=== FIND: Search Transaction ===\n');
        
        const transaction = transactions.find(txn => txn.id === id);
        
        if (transaction) {
            console.log('Found:', transaction);
        } else {
            console.log(`Transaction ${id} not found`);
        }
        
        return transaction;
    }
    
    // 5. SOME & EVERY - Check conditions
    static analyzeTransactionPatterns(transactions, accountId) {
        console.log('\n=== SOME & EVERY: Pattern Analysis ===\n');
        
        const accountTxns = transactions.filter(txn => txn.accountId === accountId);
        
        const hasLargeDeposit = accountTxns.some(
            txn => txn.type === 'CREDIT' && txn.amount > 5000
        );
        
        const allPositiveAmounts = accountTxns.every(txn => txn.amount > 0);
        
        const hasRentPayment = accountTxns.some(txn => txn.category === 'RENT');
        
        console.log(`Account ${accountId}:`);
        console.log(`- Has large deposit (>5000): ${hasLargeDeposit}`);
        console.log(`- All amounts positive: ${allPositiveAmounts}`);
        console.log(`- Has rent payment: ${hasRentPayment}`);
        
        return { hasLargeDeposit, allPositiveAmounts, hasRentPayment };
    }
    
    // 6. SORT - Order transactions
    static sortTransactions(transactions, sortBy = 'date', order = 'desc') {
        console.log('\n=== SORT: Order Transactions ===\n');
        
        const sorted = [...transactions].sort((a, b) => {
            let comparison = 0;
            
            if (sortBy === 'date') {
                comparison = new Date(a.date) - new Date(b.date);
            } else if (sortBy === 'amount') {
                comparison = a.amount - b.amount;
            }
            
            return order === 'desc' ? -comparison : comparison;
        });
        
        console.log(`Sorted by ${sortBy} (${order}):`);
        sorted.slice(0, 3).forEach(txn => 
            console.log(`${txn.date}: AED ${txn.amount}`)
        );
        
        return sorted;
    }
    
    // 7. GROUP BY (using reduce)
    static groupTransactionsByCategory(transactions) {
        console.log('\n=== REDUCE: Group By Category ===\n');
        
        const grouped = transactions.reduce((acc, txn) => {
            if (!acc[txn.category]) {
                acc[txn.category] = [];
            }
            acc[txn.category].push(txn);
            return acc;
        }, {});
        
        console.log('Transactions by category:');
        Object.entries(grouped).forEach(([category, txns]) => {
            const total = txns.reduce((sum, txn) => sum + txn.amount, 0);
            console.log(`${category}: ${txns.length} transactions, Total: AED ${total}`);
        });
        
        return grouped;
    }
    
    // 8. FLAT & FLATMAP - Process nested data
    static getAllTransactionsFromAccounts(accountsWithTxns) {
        console.log('\n=== FLAT & FLATMAP: Nested Data ===\n');
        
        // Using flatMap to get all transactions
        const allTxns = accountsWithTxns.flatMap(acc => acc.transactions);
        
        console.log(`Total transactions across all accounts: ${allTxns.length}`);
        
        return allTxns;
    }
    
    // 9. Object methods - Account summary
    static generateAccountSummary(account, transactions) {
        console.log('\n=== Object Methods: Account Summary ===\n');
        
        const accountTxns = transactions.filter(txn => txn.accountId === account.id);
        
        const totalCredits = accountTxns
            .filter(txn => txn.type === 'CREDIT')
            .reduce((sum, txn) => sum + txn.amount, 0);
        
        const totalDebits = accountTxns
            .filter(txn => txn.type === 'DEBIT')
            .reduce((sum, txn) => sum + txn.amount, 0);
        
        const summary = {
            ...account,
            totalTransactions: accountTxns.length,
            totalCredits,
            totalDebits,
            netFlow: totalCredits - totalDebits,
            lastTransaction: accountTxns[accountTxns.length - 1],
            averageTransaction: accountTxns.reduce((sum, txn) => sum + txn.amount, 0) / accountTxns.length
        };
        
        console.log(JSON.stringify(summary, null, 2));
        
        return summary;
    }
    
    // 10. Advanced: Monthly spending analysis
    static analyzeMonthlySpending(transactions) {
        console.log('\n=== Advanced: Monthly Spending Analysis ===\n');
        
        const analysis = transactions
            .filter(txn => txn.type === 'DEBIT')
            .reduce((acc, txn) => {
                const month = txn.date.substring(0, 7); // YYYY-MM
                
                if (!acc[month]) {
                    acc[month] = {
                        total: 0,
                        count: 0,
                        categories: {}
                    };
                }
                
                acc[month].total += txn.amount;
                acc[month].count++;
                
                if (!acc[month].categories[txn.category]) {
                    acc[month].categories[txn.category] = 0;
                }
                acc[month].categories[txn.category] += txn.amount;
                
                return acc;
            }, {});
        
        Object.entries(analysis).forEach(([month, data]) => {
            console.log(`\n${month}:`);
            console.log(`  Total Spending: AED ${data.total}`);
            console.log(`  Transactions: ${data.count}`);
            console.log(`  Average: AED ${(data.total / data.count).toFixed(2)}`);
            console.log(`  By Category:`, data.categories);
        });
        
        return analysis;
    }
}

// Execute examples
console.log('=== Banking Data Processing Examples ===\n');

BankingDataProcessor.calculateTaxOnTransactions(transactions);
BankingDataProcessor.getHighValueTransactions(transactions, 3000);
BankingDataProcessor.calculateAccountBalance(transactions, 'ACC001');
BankingDataProcessor.findTransactionById(transactions, 3);
BankingDataProcessor.analyzeTransactionPatterns(transactions, 'ACC001');
BankingDataProcessor.sortTransactions(transactions, 'amount', 'desc');
BankingDataProcessor.groupTransactionsByCategory(transactions);
BankingDataProcessor.generateAccountSummary(accounts[0], transactions);
BankingDataProcessor.analyzeMonthlySpending(transactions);
```

---
