# Questions 22-24: Advanced JavaScript Patterns & Design

## Question 22: What are Design Patterns? Explain Singleton, Factory, Observer, and Module patterns.

### Answer:

**Design Patterns** are reusable solutions to commonly occurring problems in software design. They represent best practices and provide a template for how to solve problems in various contexts.

### Banking Scenario: Architectural Patterns at Emirates NBD

```javascript
// =============================================================================
// SINGLETON PATTERN - Ensure only one instance exists
// =============================================================================

class DatabaseConnection {
    constructor() {
        if (DatabaseConnection.instance) {
            return DatabaseConnection.instance;
        }

        this.connectionId = `DB-${Date.now()}`;
        this.isConnected = false;
        this.queries = [];
        
        DatabaseConnection.instance = this;
        return this;
    }

    connect() {
        if (!this.isConnected) {
            console.log(`[DB] Connecting to database (${this.connectionId})...`);
            this.isConnected = true;
        }
        return this;
    }

    query(sql) {
        if (!this.isConnected) {
            throw new Error('Not connected to database');
        }
        this.queries.push({ sql, timestamp: new Date() });
        console.log(`[DB] Executing: ${sql}`);
        return { success: true, rows: [] };
    }

    getStats() {
        return {
            connectionId: this.connectionId,
            isConnected: this.isConnected,
            totalQueries: this.queries.length
        };
    }
}

// Modern Singleton using module pattern
const ConfigManager = (() => {
    let instance;
    let config = {
        apiUrl: 'https://api.emiratesnbd.com',
        apiKey: 'secret-key-123',
        timeout: 30000,
        retryAttempts: 3
    };

    class Config {
        get(key) {
            return config[key];
        }

        set(key, value) {
            console.log(`[Config] Setting ${key} = ${value}`);
            config[key] = value;
        }

        getAll() {
            return { ...config };
        }
    }

    return {
        getInstance: () => {
            if (!instance) {
                instance = new Config();
                console.log('[Config] Singleton instance created');
            }
            return instance;
        }
    };
})();

// =============================================================================
// FACTORY PATTERN - Create objects without specifying exact class
// =============================================================================

// Account types
class SavingsAccount {
    constructor(accountNumber, initialBalance) {
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
        this.type = 'SAVINGS';
        this.interestRate = 0.025; // 2.5%
        this.minBalance = 1000;
    }

    calculateInterest() {
        return this.balance * this.interestRate;
    }

    getAccountInfo() {
        return {
            type: this.type,
            accountNumber: this.accountNumber,
            balance: this.balance,
            interestRate: `${this.interestRate * 100}%`,
            minBalance: this.minBalance
        };
    }
}

class CurrentAccount {
    constructor(accountNumber, initialBalance) {
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
        this.type = 'CURRENT';
        this.interestRate = 0;
        this.minBalance = 5000;
        this.overdraftLimit = 10000;
    }

    calculateInterest() {
        return 0; // No interest on current accounts
    }

    getAccountInfo() {
        return {
            type: this.type,
            accountNumber: this.accountNumber,
            balance: this.balance,
            overdraftLimit: this.overdraftLimit,
            minBalance: this.minBalance
        };
    }
}

class FixedDepositAccount {
    constructor(accountNumber, initialBalance) {
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
        this.type = 'FIXED_DEPOSIT';
        this.interestRate = 0.045; // 4.5%
        this.term = 12; // months
        this.minBalance = 10000;
    }

    calculateInterest() {
        return this.balance * this.interestRate * (this.term / 12);
    }

    getAccountInfo() {
        return {
            type: this.type,
            accountNumber: this.accountNumber,
            balance: this.balance,
            interestRate: `${this.interestRate * 100}%`,
            term: `${this.term} months`,
            minBalance: this.minBalance
        };
    }
}

// Factory to create accounts
class AccountFactory {
    static createAccount(type, accountNumber, initialBalance) {
        console.log(`[Factory] Creating ${type} account...`);
        
        switch (type.toUpperCase()) {
            case 'SAVINGS':
                return new SavingsAccount(accountNumber, initialBalance);
            
            case 'CURRENT':
                return new CurrentAccount(accountNumber, initialBalance);
            
            case 'FIXED_DEPOSIT':
            case 'FD':
                return new FixedDepositAccount(accountNumber, initialBalance);
            
            default:
                throw new Error(`Unknown account type: ${type}`);
        }
    }

    static getAvailableTypes() {
        return ['SAVINGS', 'CURRENT', 'FIXED_DEPOSIT'];
    }
}

// =============================================================================
// OBSERVER PATTERN - Subscribe to events and get notified
// =============================================================================

class EventEmitter {
    constructor() {
        this.events = {};
    }

    on(event, listener) {
        if (!this.events[event]) {
            this.events[event] = [];
        }
        this.events[event].push(listener);
        console.log(`[Observer] Subscribed to '${event}' event`);
        
        // Return unsubscribe function
        return () => this.off(event, listener);
    }

    off(event, listenerToRemove) {
        if (!this.events[event]) return;
        
        this.events[event] = this.events[event].filter(
            listener => listener !== listenerToRemove
        );
        console.log(`[Observer] Unsubscribed from '${event}' event`);
    }

    emit(event, data) {
        if (!this.events[event]) return;
        
        console.log(`[Observer] Emitting '${event}' event to ${this.events[event].length} listeners`);
        this.events[event].forEach(listener => listener(data));
    }

    once(event, listener) {
        const onceWrapper = (data) => {
            listener(data);
            this.off(event, onceWrapper);
        };
        this.on(event, onceWrapper);
    }
}

class TransactionMonitor extends EventEmitter {
    constructor() {
        super();
        this.transactions = [];
    }

    processTransaction(transaction) {
        console.log(`\n[Monitor] Processing transaction: ${transaction.id}`);
        
        // Add transaction
        this.transactions.push({
            ...transaction,
            timestamp: new Date(),
            status: 'PROCESSING'
        });

        // Emit events based on transaction
        this.emit('transaction:created', transaction);

        // Check for high value
        if (transaction.amount >= 50000) {
            this.emit('transaction:high-value', transaction);
        }

        // Check for fraud indicators
        if (transaction.amount >= 100000) {
            this.emit('transaction:fraud-alert', {
                ...transaction,
                reason: 'High value transaction'
            });
        }

        // Emit completion
        setTimeout(() => {
            this.emit('transaction:completed', {
                ...transaction,
                status: 'COMPLETED'
            });
        }, 100);
    }
}

// =============================================================================
// MODULE PATTERN - Encapsulation and private members
// =============================================================================

const BankingModule = (() => {
    // Private variables
    let accounts = new Map();
    let transactionCounter = 0;
    const API_KEY = 'private-api-key-xyz';

    // Private functions
    function generateTransactionId() {
        return `TXN-${Date.now()}-${++transactionCounter}`;
    }

    function validateAccount(accountNumber) {
        if (!accounts.has(accountNumber)) {
            throw new Error(`Account not found: ${accountNumber}`);
        }
    }

    function logTransaction(transaction) {
        console.log(`[Module] Transaction logged: ${transaction.id}`);
    }

    // Public API
    return {
        // Create account
        createAccount(accountNumber, initialBalance = 0) {
            if (accounts.has(accountNumber)) {
                throw new Error(`Account already exists: ${accountNumber}`);
            }

            accounts.set(accountNumber, {
                accountNumber,
                balance: initialBalance,
                transactions: [],
                createdAt: new Date()
            });

            console.log(`[Module] Account created: ${accountNumber}`);
            return true;
        },

        // Get account
        getAccount(accountNumber) {
            validateAccount(accountNumber);
            return { ...accounts.get(accountNumber) };
        },

        // Deposit
        deposit(accountNumber, amount) {
            validateAccount(accountNumber);
            
            const account = accounts.get(accountNumber);
            account.balance += amount;

            const transaction = {
                id: generateTransactionId(),
                type: 'DEPOSIT',
                amount,
                timestamp: new Date()
            };

            account.transactions.push(transaction);
            logTransaction(transaction);

            return account.balance;
        },

        // Withdraw
        withdraw(accountNumber, amount) {
            validateAccount(accountNumber);
            
            const account = accounts.get(accountNumber);
            
            if (account.balance < amount) {
                throw new Error('Insufficient funds');
            }

            account.balance -= amount;

            const transaction = {
                id: generateTransactionId(),
                type: 'WITHDRAWAL',
                amount,
                timestamp: new Date()
            };

            account.transactions.push(transaction);
            logTransaction(transaction);

            return account.balance;
        },

        // Transfer
        transfer(fromAccount, toAccount, amount) {
            validateAccount(fromAccount);
            validateAccount(toAccount);

            this.withdraw(fromAccount, amount);
            this.deposit(toAccount, amount);

            console.log(`[Module] Transfer completed: ${fromAccount} → ${toAccount}`);
            return true;
        },

        // Get statistics
        getStats() {
            return {
                totalAccounts: accounts.size,
                totalTransactions: transactionCounter
            };
        }
    };
})();

// Revealing Module Pattern (cleaner)
const PaymentProcessor = (() => {
    // Private
    const processingFee = 0.02;
    let processedPayments = [];

    function calculateFee(amount) {
        return amount * processingFee;
    }

    function validatePayment(payment) {
        if (!payment.amount || payment.amount <= 0) {
            throw new Error('Invalid payment amount');
        }
        if (!payment.fromAccount || !payment.toAccount) {
            throw new Error('Account numbers required');
        }
        return true;
    }

    function processPayment(payment) {
        validatePayment(payment);
        
        const fee = calculateFee(payment.amount);
        const total = payment.amount + fee;

        const processedPayment = {
            ...payment,
            fee,
            total,
            id: `PAY-${Date.now()}`,
            timestamp: new Date(),
            status: 'COMPLETED'
        };

        processedPayments.push(processedPayment);
        return processedPayment;
    }

    function getPaymentHistory() {
        return [...processedPayments];
    }

    function getProcessingFee() {
        return processingFee;
    }

    // Reveal public API
    return {
        process: processPayment,
        getHistory: getPaymentHistory,
        getFeeRate: getProcessingFee
    };
})();

// =============================================================================
// DEMONSTRATION
// =============================================================================

console.log('=== Design Patterns Demo - Emirates NBD ===\n');

// Demo 1: Singleton Pattern
console.log('1. SINGLETON PATTERN - Single Instance\n');

const db1 = new DatabaseConnection();
const db2 = new DatabaseConnection();
const db3 = new DatabaseConnection();

console.log(`db1 === db2: ${db1 === db2}`);
console.log(`db2 === db3: ${db2 === db3}`);
console.log('✓ All references point to same instance\n');

db1.connect();
db1.query('SELECT * FROM accounts WHERE balance > 10000');

const stats = db2.getStats(); // Using different reference
console.log('Database stats:', stats);

console.log('\nConfig Manager Singleton:');
const config1 = ConfigManager.getInstance();
const config2 = ConfigManager.getInstance();
console.log(`config1 === config2: ${config1 === config2}`);
console.log('API URL:', config1.get('apiUrl'));

console.log('\n' + '='.repeat(70) + '\n');

// Demo 2: Factory Pattern
console.log('2. FACTORY PATTERN - Object Creation\n');

console.log('Creating different account types:');
const savings = AccountFactory.createAccount('SAVINGS', 'ACC-SAV-001', 10000);
const current = AccountFactory.createAccount('CURRENT', 'ACC-CUR-001', 50000);
const fixedDeposit = AccountFactory.createAccount('FD', 'ACC-FD-001', 100000);

console.log('\nSavings Account:');
console.log(savings.getAccountInfo());
console.log(`Interest: AED ${savings.calculateInterest().toFixed(2)}`);

console.log('\nCurrent Account:');
console.log(current.getAccountInfo());

console.log('\nFixed Deposit:');
console.log(fixedDeposit.getAccountInfo());
console.log(`Maturity Interest: AED ${fixedDeposit.calculateInterest().toFixed(2)}`);

console.log('\n' + '='.repeat(70) + '\n');

// Demo 3: Observer Pattern
console.log('3. OBSERVER PATTERN - Event Subscription\n');

const monitor = new TransactionMonitor();

// Subscribe to events
monitor.on('transaction:created', (txn) => {
    console.log(`  📝 Audit: Transaction ${txn.id} created for AED ${txn.amount.toLocaleString()}`);
});

monitor.on('transaction:high-value', (txn) => {
    console.log(`  ⚠️  Alert: High-value transaction detected - AED ${txn.amount.toLocaleString()}`);
});

monitor.on('transaction:fraud-alert', (txn) => {
    console.log(`  🚨 FRAUD ALERT: Suspicious transaction ${txn.id} - ${txn.reason}`);
});

monitor.on('transaction:completed', (txn) => {
    console.log(`  ✅ Complete: Transaction ${txn.id} is ${txn.status}`);
});

// Process transactions
console.log('Processing normal transaction:');
monitor.processTransaction({
    id: 'TXN-001',
    amount: 5000,
    from: 'ACC-123',
    to: 'ACC-456'
});

setTimeout(() => {
    console.log('\nProcessing high-value transaction:');
    monitor.processTransaction({
        id: 'TXN-002',
        amount: 75000,
        from: 'ACC-789',
        to: 'ACC-012'
    });
}, 200);

setTimeout(() => {
    console.log('\nProcessing suspicious transaction:');
    monitor.processTransaction({
        id: 'TXN-003',
        amount: 150000,
        from: 'ACC-345',
        to: 'ACC-678'
    });
}, 400);

setTimeout(() => {
    console.log('\n' + '='.repeat(70) + '\n');

    // Demo 4: Module Pattern
    console.log('4. MODULE PATTERN - Encapsulation\n');

    console.log('Creating accounts:');
    BankingModule.createAccount('ACC-MOD-001', 10000);
    BankingModule.createAccount('ACC-MOD-002', 20000);

    console.log('\nPerforming operations:');
    BankingModule.deposit('ACC-MOD-001', 5000);
    console.log(`Balance after deposit: AED ${BankingModule.getAccount('ACC-MOD-001').balance.toLocaleString()}`);

    BankingModule.withdraw('ACC-MOD-002', 3000);
    console.log(`Balance after withdrawal: AED ${BankingModule.getAccount('ACC-MOD-002').balance.toLocaleString()}`);

    BankingModule.transfer('ACC-MOD-001', 'ACC-MOD-002', 2000);

    console.log('\nModule statistics:');
    console.log(BankingModule.getStats());

    // Try to access private members (will fail)
    console.log('\nPrivate members are inaccessible:');
    console.log('transactionCounter:', typeof BankingModule.transactionCounter); // undefined
    console.log('API_KEY:', typeof BankingModule.API_KEY); // undefined

    console.log('\n' + '='.repeat(70) + '\n');

    // Demo 5: Revealing Module Pattern
    console.log('5. REVEALING MODULE PATTERN - Clean API\n');

    const payment1 = PaymentProcessor.process({
        fromAccount: 'ACC-001',
        toAccount: 'ACC-002',
        amount: 10000
    });

    console.log('Payment processed:');
    console.log(`  ID: ${payment1.id}`);
    console.log(`  Amount: AED ${payment1.amount.toLocaleString()}`);
    console.log(`  Fee: AED ${payment1.fee.toFixed(2)}`);
    console.log(`  Total: AED ${payment1.total.toFixed(2)}`);
    console.log(`  Status: ${payment1.status}`);

    const payment2 = PaymentProcessor.process({
        fromAccount: 'ACC-003',
        toAccount: 'ACC-004',
        amount: 25000
    });

    console.log('\nPayment history:');
    const history = PaymentProcessor.getHistory();
    console.log(`Total payments: ${history.length}`);
    
    console.log('\nProcessing fee rate:');
    console.log(`${(PaymentProcessor.getFeeRate() * 100).toFixed(2)}%`);

    console.log('\n' + '='.repeat(70) + '\n');

    // Summary
    console.log('6. PATTERN COMPARISON:\n');

    console.log('SINGLETON:');
    console.log('  Use Case: Database connections, configuration, logging');
    console.log('  Benefit: Controlled access to single instance');
    console.log('  Drawback: Can make testing difficult\n');

    console.log('FACTORY:');
    console.log('  Use Case: Creating objects without specifying exact class');
    console.log('  Benefit: Decouples object creation from usage');
    console.log('  Drawback: Can add unnecessary complexity\n');

    console.log('OBSERVER:');
    console.log('  Use Case: Event systems, real-time updates, notifications');
    console.log('  Benefit: Loose coupling between components');
    console.log('  Drawback: Memory leaks if not unsubscribed\n');

    console.log('MODULE:');
    console.log('  Use Case: Encapsulation, private members, namespacing');
    console.log('  Benefit: Clean API, information hiding');
    console.log('  Drawback: Less flexible than classes\n');

    console.log('When to use each:');
    console.log('  • Singleton: Shared resources (DB, cache, config)');
    console.log('  • Factory: Multiple related types of objects');
    console.log('  • Observer: Event-driven architecture');
    console.log('  • Module: Utility libraries, service layers');

}, 600);
```

### Pattern Summary:

| Pattern | Purpose | Example |
|---------|---------|---------|
| **Singleton** | One instance only | Database connection, Config |
| **Factory** | Create objects flexibly | Account types creation |
| **Observer** | Event subscription | Transaction notifications |
| **Module** | Encapsulation | Private methods/data |

---

## Question 23: What is the Prototype Chain? How does inheritance work in JavaScript?

### Answer:

JavaScript uses **prototypal inheritance** where objects inherit properties and methods from other objects through a prototype chain. Every object has an internal `[[Prototype]]` property that points to another object.

### Prototype Chain Lookup:
1. Check if property exists on object itself
2. If not, check object's prototype
3. Continue up the chain until found or reach `null`

### Banking Scenario: Account Hierarchy at Emirates NBD

```javascript
console.log('=== Prototype Chain & Inheritance - Emirates NBD ===\n');

// =============================================================================
// CONSTRUCTOR FUNCTION INHERITANCE (Old Way)
// =============================================================================

console.log('1. Constructor Function Inheritance:\n');

// Base constructor
function BankAccount(accountNumber, balance) {
    this.accountNumber = accountNumber;
    this.balance = balance;
    this.transactions = [];
}

// Methods on prototype (shared by all instances)
BankAccount.prototype.deposit = function(amount) {
    this.balance += amount;
    this.transactions.push({
        type: 'DEPOSIT',
        amount,
        timestamp: new Date()
    });
    return this.balance;
};

BankAccount.prototype.withdraw = function(amount) {
    if (this.balance >= amount) {
        this.balance -= amount;
        this.transactions.push({
            type: 'WITHDRAWAL',
            amount,
            timestamp: new Date()
        });
        return this.balance;
    }
    throw new Error('Insufficient funds');
};

BankAccount.prototype.getBalance = function() {
    return this.balance;
};

// Child constructor
function SavingsAccount(accountNumber, balance, interestRate) {
    // Call parent constructor
    BankAccount.call(this, accountNumber, balance);
    this.interestRate = interestRate;
    this.accountType = 'SAVINGS';
}

// Set up prototype chain
SavingsAccount.prototype = Object.create(BankAccount.prototype);
SavingsAccount.prototype.constructor = SavingsAccount;

// Add child-specific methods
SavingsAccount.prototype.calculateInterest = function() {
    const interest = this.balance * this.interestRate;
    console.log(`Interest: AED ${interest.toFixed(2)}`);
    return interest;
};

SavingsAccount.prototype.applyInterest = function() {
    const interest = this.calculateInterest();
    this.deposit(interest);
    console.log(`Interest applied: AED ${interest.toFixed(2)}`);
    return this.balance;
};

// Create instances
const account1 = new BankAccount('ACC-001', 10000);
const savings1 = new SavingsAccount('ACC-SAV-001', 50000, 0.025);

console.log('BankAccount instance:');
account1.deposit(5000);
console.log(`Balance: AED ${account1.getBalance().toLocaleString()}\n`);

console.log('SavingsAccount instance (inherits from BankAccount):');
savings1.deposit(10000);
savings1.calculateInterest();
console.log(`Balance: AED ${savings1.getBalance().toLocaleString()}`);

console.log('\nPrototype chain check:');
console.log('savings1 instanceof SavingsAccount:', savings1 instanceof SavingsAccount);
console.log('savings1 instanceof BankAccount:', savings1 instanceof BankAccount);
console.log('savings1 instanceof Object:', savings1 instanceof Object);

console.log('\n' + '='.repeat(70) + '\n');

// =============================================================================
// ES6 CLASS INHERITANCE (Modern Way)
// =============================================================================

console.log('2. ES6 Class Inheritance:\n');

class ModernBankAccount {
    constructor(accountNumber, balance = 0) {
        this.accountNumber = accountNumber;
        this.balance = balance;
        this.transactions = [];
        this.createdAt = new Date();
    }

    deposit(amount) {
        this.balance += amount;
        this.transactions.push({
            type: 'DEPOSIT',
            amount,
            timestamp: new Date()
        });
        console.log(`Deposited AED ${amount.toLocaleString()}`);
        return this.balance;
    }

    withdraw(amount) {
        if (this.balance < amount) {
            throw new Error('Insufficient funds');
        }
        this.balance -= amount;
        this.transactions.push({
            type: 'WITHDRAWAL',
            amount,
            timestamp: new Date()
        });
        console.log(`Withdrew AED ${amount.toLocaleString()}`);
        return this.balance;
    }

    getBalance() {
        return this.balance;
    }

    getAccountInfo() {
        return {
            accountNumber: this.accountNumber,
            balance: this.balance,
            transactionCount: this.transactions.length,
            accountType: this.constructor.name
        };
    }

    // Static method
    static validateAccountNumber(accountNumber) {
        return /^ACC-\d{9}$/.test(accountNumber);
    }
}

class ModernSavingsAccount extends ModernBankAccount {
    constructor(accountNumber, balance, interestRate = 0.025) {
        super(accountNumber, balance); // Call parent constructor
        this.interestRate = interestRate;
        this.minBalance = 1000;
    }

    // Override parent method
    withdraw(amount) {
        if (this.balance - amount < this.minBalance) {
            throw new Error(`Balance cannot go below minimum: AED ${this.minBalance}`);
        }
        return super.withdraw(amount); // Call parent method
    }

    // New method specific to SavingsAccount
    calculateInterest() {
        return this.balance * this.interestRate;
    }

    applyInterest() {
        const interest = this.calculateInterest();
        this.deposit(interest);
        console.log(`✓ Interest applied: AED ${interest.toFixed(2)}`);
        return this.balance;
    }

    // Override getAccountInfo
    getAccountInfo() {
        return {
            ...super.getAccountInfo(),
            interestRate: `${(this.interestRate * 100).toFixed(2)}%`,
            minBalance: this.minBalance,
            nextInterestAmount: this.calculateInterest().toFixed(2)
        };
    }
}

class PremiumSavingsAccount extends ModernSavingsAccount {
    constructor(accountNumber, balance) {
        super(accountNumber, balance, 0.045); // Higher interest rate
        this.relationshipManager = null;
        this.benefits = ['Free ATM withdrawals', 'Travel insurance', 'Airport lounge access'];
    }

    assignRelationshipManager(managerName) {
        this.relationshipManager = managerName;
        console.log(`Relationship Manager assigned: ${managerName}`);
    }

    getBenefits() {
        return this.benefits;
    }

    getAccountInfo() {
        return {
            ...super.getAccountInfo(),
            accountTier: 'PREMIUM',
            relationshipManager: this.relationshipManager,
            benefits: this.benefits
        };
    }
}

// Create instances
console.log('Creating accounts with class inheritance:\n');

const modern1 = new ModernBankAccount('ACC-100000001', 5000);
const modern2 = new ModernSavingsAccount('ACC-200000002', 20000);
const modern3 = new PremiumSavingsAccount('ACC-300000003', 100000);

modern1.deposit(2000);
console.log(`Modern account balance: AED ${modern1.getBalance().toLocaleString()}\n`);

modern2.deposit(5000);
modern2.applyInterest();
console.log(`Savings account balance: AED ${modern2.getBalance().toLocaleString()}\n`);

modern3.deposit(50000);
modern3.assignRelationshipManager('Ahmed Ali');
modern3.applyInterest();
console.log(`Premium account balance: AED ${modern3.getBalance().toLocaleString()}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// PROTOTYPE CHAIN EXPLORATION
// =============================================================================

console.log('3. Prototype Chain Exploration:\n');

const testAccount = new ModernSavingsAccount('ACC-TEST-001', 10000);

console.log('Object properties:');
console.log('Own properties:', Object.keys(testAccount));
console.log('Has accountNumber:', testAccount.hasOwnProperty('accountNumber'));
console.log('Has deposit:', testAccount.hasOwnProperty('deposit')); // false - it's on prototype

console.log('\nPrototype chain:');
console.log('testAccount.__proto__ === ModernSavingsAccount.prototype:', 
    Object.getPrototypeOf(testAccount) === ModernSavingsAccount.prototype);
console.log('ModernSavingsAccount.prototype.__proto__ === ModernBankAccount.prototype:',
    Object.getPrototypeOf(ModernSavingsAccount.prototype) === ModernBankAccount.prototype);
console.log('ModernBankAccount.prototype.__proto__ === Object.prototype:',
    Object.getPrototypeOf(ModernBankAccount.prototype) === Object.prototype);

console.log('\nProperty lookup chain:');
console.log('testAccount.balance → found on instance');
console.log('testAccount.deposit() → found on ModernBankAccount.prototype');
console.log('testAccount.calculateInterest() → found on ModernSavingsAccount.prototype');
console.log('testAccount.toString() → found on Object.prototype');

console.log('\n' + '='.repeat(70) + '\n');

// =============================================================================
// PROTOTYPE MANIPULATION
// =============================================================================

console.log('4. Prototype Manipulation:\n');

// Adding methods to prototype after creation
ModernBankAccount.prototype.getTransactionCount = function() {
    return this.transactions.length;
};

// Now all instances have this method (even existing ones!)
console.log('Transaction count:', testAccount.getTransactionCount());

// Checking prototype methods
console.log('\nAvailable methods on ModernSavingsAccount:');
console.log('- deposit (inherited)');
console.log('- withdraw (overridden)');
console.log('- getBalance (inherited)');
console.log('- calculateInterest (own)');
console.log('- applyInterest (own)');
console.log('- getTransactionCount (added dynamically)');

console.log('\n' + '='.repeat(70) + '\n');

// =============================================================================
// OBJECT.CREATE() - Manual Prototype Chain
// =============================================================================

console.log('5. Object.create() - Manual Prototype Setup:\n');

const accountPrototype = {
    deposit(amount) {
        this.balance += amount;
        console.log(`Deposited AED ${amount.toLocaleString()}`);
    },
    
    withdraw(amount) {
        if (this.balance >= amount) {
            this.balance -= amount;
            console.log(`Withdrew AED ${amount.toLocaleString()}`);
        } else {
            console.log('Insufficient funds');
        }
    },
    
    getInfo() {
        return `Account: ${this.accountNumber}, Balance: AED ${this.balance.toLocaleString()}`;
    }
};

// Create object with specific prototype
const manualAccount = Object.create(accountPrototype);
manualAccount.accountNumber = 'ACC-MANUAL-001';
manualAccount.balance = 15000;

console.log('Manual prototype account:');
console.log(manualAccount.getInfo());
manualAccount.deposit(5000);
console.log(manualAccount.getInfo());

console.log('\nPrototype check:');
console.log('manualAccount.__proto__ === accountPrototype:', 
    Object.getPrototypeOf(manualAccount) === accountPrototype);

console.log('\n' + '='.repeat(70) + '\n');

// =============================================================================
// MIXINS - Multiple Inheritance Pattern
// =============================================================================

console.log('6. Mixins - Composition over Inheritance:\n');

// Mixin for auditing functionality
const AuditableMixin = {
    enableAudit() {
        this._auditLog = [];
        this._auditEnabled = true;
        console.log('[Audit] Enabled for', this.accountNumber);
    },

    logAction(action, details) {
        if (this._auditEnabled) {
            this._auditLog.push({
                action,
                details,
                timestamp: new Date(),
                accountNumber: this.accountNumber
            });
        }
    },

    getAuditLog() {
        return this._auditLog || [];
    }
};

// Mixin for notifications
const NotifiableMixin = {
    enableNotifications(email, phone) {
        this._notifications = { email, phone, enabled: true };
        console.log('[Notifications] Enabled for', email);
    },

    sendNotification(message) {
        if (this._notifications?.enabled) {
            console.log(`[Notification] ${message} → ${this._notifications.email}`);
        }
    }
};

// Apply mixins to a class
class AuditableAccount extends ModernBankAccount {
    constructor(accountNumber, balance) {
        super(accountNumber, balance);
        // Apply mixins
        Object.assign(this, AuditableMixin, NotifiableMixin);
        this.enableAudit();
    }

    deposit(amount) {
        const result = super.deposit(amount);
        this.logAction('DEPOSIT', { amount });
        this.sendNotification(`Deposit of AED ${amount.toLocaleString()} received`);
        return result;
    }

    withdraw(amount) {
        const result = super.withdraw(amount);
        this.logAction('WITHDRAWAL', { amount });
        this.sendNotification(`Withdrawal of AED ${amount.toLocaleString()} processed`);
        return result;
    }
}

const auditableAcc = new AuditableAccount('ACC-AUDIT-001', 25000);
auditableAcc.enableNotifications('ahmed@example.com', '+971501234567');

console.log('\nPerforming operations:');
auditableAcc.deposit(5000);
auditableAcc.withdraw(2000);

console.log('\nAudit log:');
auditableAcc.getAuditLog().forEach((entry, i) => {
    console.log(`${i + 1}. ${entry.action}: AED ${entry.details.amount.toLocaleString()} at ${entry.timestamp.toLocaleTimeString()}`);
});

console.log('\n' + '='.repeat(70) + '\n');

// Summary
console.log('7. INHERITANCE SUMMARY:\n');

console.log('Prototype Chain:');
console.log('  • Every object has [[Prototype]] (accessible via __proto__)');
console.log('  • Property lookup walks up the chain');
console.log('  • Chain ends at Object.prototype (then null)');

console.log('\nConstructor Functions (Old):');
console.log('  • Use function constructors');
console.log('  • Manually set up prototype chain');
console.log('  • Use .call() to invoke parent constructor');

console.log('\nES6 Classes (Modern):');
console.log('  • Syntactic sugar over prototypes');
console.log('  • extends for inheritance');
console.log('  • super() to call parent');
console.log('  • Cleaner, more intuitive syntax');

console.log('\nBest Practices:');
console.log('  ✓ Prefer ES6 classes for clarity');
console.log('  ✓ Use composition (mixins) for multiple behaviors');
console.log('  ✓ Don't modify built-in prototypes');
console.log('  ✓ Use Object.create() for simple delegation');
console.log('  ✓ Check with instanceof and hasOwnProperty()');
```

### Prototype Chain Diagram:

```
instance
  ↓ [[Prototype]]
Class.prototype
  ↓ [[Prototype]]
ParentClass.prototype
  ↓ [[Prototype]]
Object.prototype
  ↓ [[Prototype]]
null
```

---

## Question 24: What is Hoisting? Explain TDZ (Temporal Dead Zone).

### Answer:

**Hoisting** is JavaScript's behavior of moving declarations to the top of their scope during compilation. However, only declarations are hoisted, not initializations.

**Temporal Dead Zone (TDZ)** is the period between entering scope and variable declaration where accessing the variable throws a `ReferenceError`.

### Banking Scenario: Variable Scoping at Emirates NBD

```javascript
console.log('=== Hoisting & TDZ - Emirates NBD ===\n');

// =============================================================================
// VAR HOISTING
// =============================================================================

console.log('1. VAR Hoisting:\n');

console.log('Before declaration - accountBalance:', typeof accountBalance); // undefined
var accountBalance = 10000;
console.log('After declaration - accountBalance:', accountBalance);

// What actually happens:
/*
var accountBalance; // Declaration hoisted to top
console.log(accountBalance); // undefined
accountBalance = 10000; // Assignment stays in place
*/

console.log('\nFunction hoisting:');

// Can call function before declaration
processTransaction('TXN-001', 5000);

function processTransaction(id, amount) {
    console.log(`Processing transaction ${id}: AED ${amount.toLocaleString()}`);
}

// Function declarations are fully hoisted (declaration + definition)

console.log('\n' + '='.repeat(70) + '\n');

// =============================================================================
// LET/CONST HOISTING & TDZ
// =============================================================================

console.log('2. LET/CONST and Temporal Dead Zone:\n');

console.log('Demonstration of TDZ:\n');

function checkBalance() {
    // TDZ starts here for 'balance'
    
    console.log('Entering function...');
    
    // ❌ Accessing 'balance' here throws ReferenceError (in TDZ)
    // console.log(balance); // ReferenceError: Cannot access 'balance' before initialization
    
    let balance = 50000; // TDZ ends here
    
    // ✅ Now 'balance' is accessible
    console.log('Balance:', balance);
    
    return balance;
}

checkBalance();

console.log('\nTDZ with const:');

function createAccount() {
    // TDZ starts for 'accountNumber'
    
    // ❌ This would throw ReferenceError
    // console.log(accountNumber);
    
    const accountNumber = 'ACC-123456789'; // TDZ ends
    
    // ✅ Now accessible
    console.log('Account created:', accountNumber);
    
    // ❌ Cannot reassign const
    // accountNumber = 'ACC-999999999'; // TypeError
    
    return accountNumber;
}

createAccount();

console.log('\n' + '='.repeat(70) + '\n');

// =============================================================================
// FUNCTION HOISTING vs FUNCTION EXPRESSIONS
// =============================================================================

console.log('3. Function Hoisting vs Expressions:\n');

// ✅ Function Declaration - Fully hoisted
console.log('Calling hoisted function:');
calculateInterest(10000, 0.05); // Works!

function calculateInterest(principal, rate) {
    const interest = principal * rate;
    console.log(`Interest: AED ${interest.toFixed(2)}`);
    return interest;
}

// ❌ Function Expression - Variable hoisted, but not the function
console.log('\nFunction expression:');

// var is hoisted as undefined
console.log('Type of calculateFee:', typeof calculateFee); // undefined

// ❌ Calling here would throw TypeError (not a function)
// calculateFee(5000); // TypeError: calculateFee is not a function

var calculateFee = function(amount) {
    return amount * 0.02;
};

// ✅ Now it works
console.log('Fee:', calculateFee(5000));

// ❌ Arrow function - Same as function expression
console.log('\nArrow function:');
console.log('Type of processPayment:', typeof processPayment); // undefined

// processPayment(1000); // TypeError

var processPayment = (amount) => {
    console.log(`Processing payment: AED ${amount.toLocaleString()}`);
};

processPayment(1000); // Now works

console.log('\n' + '='.repeat(70) + '\n');

// =============================================================================
// TDZ IN DIFFERENT SCOPES
// =============================================================================

console.log('4. TDZ in Different Scopes:\n');

const globalBalance = 100000;
console.log('Global balance:', globalBalance);

function transferFunds(amount) {
    // TDZ for 'fee' starts here
    
    console.log('\nInside transferFunds:');
    console.log('Amount:', amount);
    
    // ❌ Accessing 'fee' here is in TDZ
    // console.log('Fee:', fee); // ReferenceError
    
    if (amount > 10000) {
        // TDZ for 'fee' continues here
        
        // ❌ Still in TDZ
        // console.log('High amount fee:', fee); // ReferenceError
        
        let fee = amount * 0.03; // TDZ ends here
        
        // ✅ Now accessible
        console.log('High amount fee: AED', fee.toFixed(2));
    } else {
        let fee = amount * 0.01; // Different 'fee' in different block scope
        console.log('Standard fee: AED', fee.toFixed(2));
    }
    
    // ❌ 'fee' not accessible here (block scoped)
    // console.log('Final fee:', fee); // ReferenceError
}

transferFunds(15000);
transferFunds(5000);

console.log('\n' + '='.repeat(70) + '\n');

// =============================================================================
// PRACTICAL EXAMPLES
// =============================================================================

console.log('5. Practical Examples:\n');

// Example 1: Variable shadowing and hoisting
var accountStatus = 'ACTIVE';

function checkAccountStatus() {
    console.log('Status at start:', accountStatus); // undefined (not 'ACTIVE')
    
    // 'var accountStatus' is hoisted to top of function
    var accountStatus = 'INACTIVE';
    
    console.log('Status at end:', accountStatus); // 'INACTIVE'
}

checkAccountStatus();
console.log('Global status:', accountStatus); // Still 'ACTIVE'

console.log();

// Example 2: Loop variable hoisting
console.log('Loop variable hoisting with var:');

for (var i = 0; i < 3; i++) {
    setTimeout(() => {
        console.log('  Iteration (var):', i); // All print 3!
    }, 10);
}

setTimeout(() => {
    console.log('  i after loop:', i); // 3 (var is function-scoped)
    
    console.log('\nLoop variable with let:');
    
    for (let j = 0; j < 3; j++) {
        setTimeout(() => {
            console.log('  Iteration (let):', j); // Prints 0, 1, 2
        }, 10);
    }
    
    // ❌ j is not accessible here (block-scoped)
    // console.log('j after loop:', j); // ReferenceError
}, 50);

// Example 3: Class hoisting
setTimeout(() => {
    console.log('\nClass hoisting:');
    
    // ❌ Classes are NOT hoisted like function declarations
    // const acc = new Account('ACC-001', 10000); // ReferenceError
    
    class Account {
        constructor(accountNumber, balance) {
            this.accountNumber = accountNumber;
            this.balance = balance;
        }
    }
    
    // ✅ Works after declaration
    const acc = new Account('ACC-001', 10000);
    console.log('Account created:', acc.accountNumber);
    
    console.log('\n' + '='.repeat(70) + '\n');
    
    // Summary
    console.log('6. HOISTING SUMMARY:\n');
    
    console.log('VAR:');
    console.log('  • Hoisted to top of function scope');
    console.log('  • Initialized with undefined');
    console.log('  • Can be accessed before declaration (returns undefined)');
    console.log('  • Function-scoped');
    
    console.log('\nLET/CONST:');
    console.log('  • Hoisted to top of block scope');
    console.log('  • NOT initialized');
    console.log('  • TDZ from start of scope to declaration');
    console.log('  • ReferenceError if accessed in TDZ');
    console.log('  • Block-scoped');
    
    console.log('\nFUNCTION DECLARATIONS:');
    console.log('  • Fully hoisted (can be called before declaration)');
    console.log('  • Both declaration and definition hoisted');
    
    console.log('\nFUNCTION EXPRESSIONS & ARROW FUNCTIONS:');
    console.log('  • Only variable hoisted (as var/let/const)');
    console.log('  • Function not available until assignment');
    console.log('  • TypeError if called before assignment');
    
    console.log('\nCLASSES:');
    console.log('  • NOT hoisted like function declarations');
    console.log('  • Must be declared before use');
    console.log('  • ReferenceError if used before declaration');
    
    console.log('\nBEST PRACTICES:');
    console.log('  ✓ Prefer let/const over var');
    console.log('  ✓ Declare variables at top of scope');
    console.log('  ✓ Declare functions before using them');
    console.log('  ✓ Use strict mode to catch errors');
    console.log('  ✓ Avoid relying on hoisting behavior');
    
    console.log('\nTDZ (Temporal Dead Zone):');
    console.log('  • Period between scope entry and variable declaration');
    console.log('  • Only applies to let/const');
    console.log('  • Accessing variable in TDZ throws ReferenceError');
    console.log('  • Helps catch errors and enforce better coding practices');
    
}, 100);
```

### Hoisting Behavior Summary:

| Type | Hoisted? | Initialized? | TDZ? | Scope |
|------|----------|--------------|------|-------|
| `var` | ✅ Yes | ✅ undefined | ❌ No | Function |
| `let` | ✅ Yes | ❌ No | ✅ Yes | Block |
| `const` | ✅ Yes | ❌ No | ✅ Yes | Block |
| `function` declaration | ✅ Yes (fully) | ✅ Yes | ❌ No | Function |
| `function` expression | Depends on `var`/`let`/`const` | ❌ No | Depends | Variable scope |
| `class` | ❌ No | ❌ No | ✅ Yes | Block |

### Key Takeaways:

- **var**: Hoisted and initialized to `undefined`
- **let/const**: Hoisted but not initialized (TDZ)
- **Functions**: Declarations fully hoisted, expressions are not
- **TDZ**: Prevents accessing variables before declaration
- **Best Practice**: Declare variables at the top, use `let`/`const`

---

**End of Questions 22-24**
