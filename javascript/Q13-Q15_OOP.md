# JavaScript Interview Questions (Q13-Q15): Object-Oriented Programming

## Q13: Explain classes, inheritance, and polymorphism with banking system

**Answer:**

ES6 classes provide syntactic sugar over JavaScript's prototypal inheritance, making OOP patterns clearer.

**Banking System with OOP:**

```javascript
// Base Account Class
class BankAccount {
    // Private fields (ES2022)
    #balance;
    #pin;
    #transactionHistory = [];
    
    // Static properties
    static bankName = 'Emirates NBD';
    static swiftCode = 'EBILAEAD';
    static accountCount = 0;
    
    constructor(accountNumber, accountHolder, initialBalance, pin) {
        this.accountNumber = accountNumber;
        this.accountHolder = accountHolder;
        this.#balance = initialBalance;
        this.#pin = pin;
        this.createdAt = new Date();
        this.status = 'ACTIVE';
        
        BankAccount.accountCount++;
    }
    
    // Getter
    get balance() {
        return this.#balance;
    }
    
    // Setter with validation
    set balance(amount) {
        if (amount < 0) {
            throw new Error('Balance cannot be negative');
        }
        this.#balance = amount;
    }
    
    // Public methods
    deposit(amount) {
        if (amount <= 0) {
            throw new Error('Deposit amount must be positive');
        }
        
        this.#balance += amount;
        this.#recordTransaction('DEPOSIT', amount);
        
        console.log(`Deposited AED ${amount}. New balance: AED ${this.#balance}`);
        return this.#balance;
    }
    
    withdraw(amount) {
        if (amount <= 0) {
            throw new Error('Withdrawal amount must be positive');
        }
        
        if (amount > this.#balance) {
            throw new Error('Insufficient funds');
        }
        
        this.#balance -= amount;
        this.#recordTransaction('WITHDRAWAL', amount);
        
        console.log(`Withdrew AED ${amount}. New balance: AED ${this.#balance}`);
        return this.#balance;
    }
    
    verifyPin(inputPin) {
        return this.#pin === inputPin;
    }
    
    changePin(oldPin, newPin) {
        if (!this.verifyPin(oldPin)) {
            throw new Error('Invalid current PIN');
        }
        
        if (newPin.length !== 4) {
            throw new Error('PIN must be 4 digits');
        }
        
        this.#pin = newPin;
        console.log('PIN changed successfully');
    }
    
    getStatement() {
        return {
            accountNumber: this.accountNumber,
            accountHolder: this.accountHolder,
            accountType: this.constructor.name,
            balance: this.#balance,
            status: this.status,
            transactionCount: this.#transactionHistory.length,
            createdAt: this.createdAt
        };
    }
    
    getTransactionHistory(limit = 10) {
        return this.#transactionHistory
            .slice(-limit)
            .map(txn => ({ ...txn })); // Return copies
    }
    
    // Private method
    #recordTransaction(type, amount) {
        this.#transactionHistory.push({
            id: this.#transactionHistory.length + 1,
            type,
            amount,
            balance: this.#balance,
            timestamp: new Date().toISOString()
        });
    }
    
    // Static method
    static getBankInfo() {
        return {
            name: BankAccount.bankName,
            swiftCode: BankAccount.swiftCode,
            totalAccounts: BankAccount.accountCount
        };
    }
    
    // Method to be overridden
    calculateInterest() {
        return 0; // Base implementation
    }
}

// Savings Account - Inheritance
class SavingsAccount extends BankAccount {
    static interestRate = 0.025; // 2.5%
    static minBalance = 1000;
    
    constructor(accountNumber, accountHolder, initialBalance, pin) {
        super(accountNumber, accountHolder, initialBalance, pin);
        this.accountType = 'SAVINGS';
    }
    
    // Override withdraw method
    withdraw(amount) {
        const minBalance = SavingsAccount.minBalance;
        
        if (this.balance - amount < minBalance) {
            throw new Error(`Savings account must maintain minimum balance of AED ${minBalance}`);
        }
        
        return super.withdraw(amount);
    }
    
    // Override calculateInterest
    calculateInterest() {
        const interest = this.balance * SavingsAccount.interestRate;
        console.log(`Interest calculated: AED ${interest.toFixed(2)} at ${SavingsAccount.interestRate * 100}%`);
        return interest;
    }
    
    // New method specific to savings
    applyMonthlyInterest() {
        const interest = this.calculateInterest() / 12;
        this.deposit(interest);
        console.log(`Monthly interest applied: AED ${interest.toFixed(2)}`);
        return this.balance;
    }
}

// Current Account - Inheritance
class CurrentAccount extends BankAccount {
    #overdraftLimit;
    
    constructor(accountNumber, accountHolder, initialBalance, pin, overdraftLimit = 5000) {
        super(accountNumber, accountHolder, initialBalance, pin);
        this.accountType = 'CURRENT';
        this.#overdraftLimit = overdraftLimit;
    }
    
    get overdraftLimit() {
        return this.#overdraftLimit;
    }
    
    get availableBalance() {
        return this.balance + this.#overdraftLimit;
    }
    
    // Override withdraw to allow overdraft
    withdraw(amount) {
        if (amount > this.availableBalance) {
            throw new Error(`Exceeds overdraft limit. Available: AED ${this.availableBalance}`);
        }
        
        if (amount <= this.balance) {
            return super.withdraw(amount);
        }
        
        // Using overdraft
        this.balance = this.balance - amount;
        console.log(`Withdrew AED ${amount} using overdraft. Balance: AED ${this.balance}`);
        return this.balance;
    }
    
    getOverdraftInfo() {
        const used = Math.max(0, -this.balance);
        return {
            limit: this.#overdraftLimit,
            used,
            available: this.#overdraftLimit - used
        };
    }
}

// Fixed Deposit Account
class FixedDepositAccount extends BankAccount {
    static interestRates = {
        '1year': 0.04,
        '3years': 0.055,
        '5years': 0.065
    };
    
    constructor(accountNumber, accountHolder, amount, pin, tenure) {
        super(accountNumber, accountHolder, amount, pin);
        this.accountType = 'FIXED_DEPOSIT';
        this.tenure = tenure;
        this.maturityDate = new Date();
        this.maturityDate.setFullYear(this.maturityDate.getFullYear() + parseInt(tenure));
        this.isMatured = false;
    }
    
    // Override withdraw - not allowed before maturity
    withdraw(amount) {
        if (!this.isMatured) {
            throw new Error(`Cannot withdraw before maturity date: ${this.maturityDate.toDateString()}`);
        }
        return super.withdraw(amount);
    }
    
    // Override deposit - not allowed after creation
    deposit(amount) {
        throw new Error('Cannot deposit to fixed deposit account');
    }
    
    calculateInterest() {
        const rate = FixedDepositAccount.interestRates[this.tenure] || 0.04;
        const years = parseInt(this.tenure);
        return this.balance * rate * years;
    }
    
    checkMaturity() {
        if (new Date() >= this.maturityDate) {
            this.isMatured = true;
            const interest = this.calculateInterest();
            this.balance = this.balance + interest;
            console.log(`Fixed deposit matured! Interest: AED ${interest}, Total: AED ${this.balance}`);
        }
        return this.isMatured;
    }
}

console.log('=== OOP Banking System ===\n');

// Create different account types
const savings = new SavingsAccount('SAV001', 'Ahmed Al Mansouri', 10000, '1234');
const current = new CurrentAccount('CUR001', 'Fatima Hassan', 5000, '5678', 10000);
const fixedDeposit = new FixedDepositAccount('FD001', 'Omar Abdullah', 50000, '9012', '3years');

console.log('Savings Account:');
console.log(savings.getStatement());
savings.deposit(2000);
savings.applyMonthlyInterest();

console.log('\nCurrent Account:');
console.log(current.getStatement());
current.withdraw(8000); // Uses overdraft
console.log('Overdraft info:', current.getOverdraftInfo());

console.log('\nFixed Deposit:');
console.log(fixedDeposit.getStatement());
console.log('Interest:', fixedDeposit.calculateInterest());

try {
    fixedDeposit.withdraw(1000); // Should fail
} catch (error) {
    console.log('Error:', error.message);
}

console.log('\nBank Info:', BankAccount.getBankInfo());

// Polymorphism Example
console.log('\n=== Polymorphism ===\n');

class AccountManager {
    static processMonthEnd(accounts) {
        console.log('Processing month-end for all accounts...\n');
        
        accounts.forEach(account => {
            console.log(`\nAccount: ${account.accountNumber} (${account.accountType})`);
            
            // Polymorphic call - different behavior for each account type
            const interest = account.calculateInterest();
            
            if (account instanceof SavingsAccount) {
                account.applyMonthlyInterest();
            } else if (account instanceof FixedDepositAccount) {
                account.checkMaturity();
            } else if (account instanceof CurrentAccount) {
                console.log('Current account - no interest');
            }
            
            console.log('Current balance:', account.balance);
        });
    }
    
    static generateReport(accounts) {
        console.log('\n=== Account Report ===\n');
        
        const report = accounts.map(account => ({
            number: account.accountNumber,
            holder: account.accountHolder,
            type: account.accountType,
            balance: account.balance,
            interest: account.calculateInterest()
        }));
        
        console.table(report);
        
        const totalBalance = accounts.reduce((sum, acc) => sum + acc.balance, 0);
        console.log(`\nTotal balance across all accounts: AED ${totalBalance}`);
    }
}

const allAccounts = [savings, current, fixedDeposit];
AccountManager.processMonthEnd(allAccounts);
AccountManager.generateReport(allAccounts);

// Abstract pattern with base class
console.log('\n=== Abstract Pattern ===\n');

class Transaction {
    constructor(fromAccount, toAccount, amount) {
        if (this.constructor === Transaction) {
            throw new Error('Cannot instantiate abstract Transaction class');
        }
        
        this.fromAccount = fromAccount;
        this.toAccount = toAccount;
        this.amount = amount;
        this.timestamp = new Date();
        this.status = 'PENDING';
    }
    
    // Abstract method
    execute() {
        throw new Error('execute() must be implemented by subclass');
    }
    
    validate() {
        if (this.amount <= 0) {
            throw new Error('Amount must be positive');
        }
        return true;
    }
}

class InstantTransfer extends Transaction {
    execute() {
        this.validate();
        
        console.log(`Executing instant transfer of AED ${this.amount}`);
        this.fromAccount.withdraw(this.amount);
        this.toAccount.deposit(this.amount);
        
        this.status = 'COMPLETED';
        console.log('Transfer completed instantly');
    }
}

class ScheduledTransfer extends Transaction {
    constructor(fromAccount, toAccount, amount, scheduledDate) {
        super(fromAccount, toAccount, amount);
        this.scheduledDate = scheduledDate;
    }
    
    execute() {
        this.validate();
        
        if (new Date() < this.scheduledDate) {
            console.log(`Transfer scheduled for ${this.scheduledDate.toDateString()}`);
            this.status = 'SCHEDULED';
            return;
        }
        
        console.log(`Executing scheduled transfer of AED ${this.amount}`);
        this.fromAccount.withdraw(this.amount);
        this.toAccount.deposit(this.amount);
        
        this.status = 'COMPLETED';
        console.log('Scheduled transfer completed');
    }
}

const acc1 = new SavingsAccount('SAV002', 'Ali Ahmed', 15000, '1111');
const acc2 = new CurrentAccount('CUR002', 'Sara Mohamed', 8000, '2222', 5000);

const instantTxn = new InstantTransfer(acc1, acc2, 2000);
instantTxn.execute();

const futureDate = new Date();
futureDate.setDate(futureDate.getDate() + 7);
const scheduledTxn = new ScheduledTransfer(acc1, acc2, 1000, futureDate);
scheduledTxn.execute();
```

---

## Q14: Explain mixins, composition, and advanced OOP patterns in banking

**Answer:**

Mixins allow adding functionality to classes without traditional inheritance. Composition favors combining objects over inheritance.

**Banking System with Mixins and Composition:**

```javascript
// Mixin Pattern
console.log('=== Mixin Pattern ===\n');

// Auditable mixin
const AuditableMixin = {
    initAudit() {
        this._auditLog = [];
    },
    
    logAudit(action, details) {
        this._auditLog.push({
            timestamp: new Date().toISOString(),
            action,
            details,
            user: this.currentUser || 'SYSTEM'
        });
    },
    
    getAuditLog() {
        return [...this._auditLog];
    },
    
    getAuditSummary() {
        const summary = {};
        this._auditLog.forEach(entry => {
            summary[entry.action] = (summary[entry.action] || 0) + 1;
        });
        return summary;
    }
};

// Lockable mixin
const LockableMixin = {
    initLockable() {
        this._isLocked = false;
        this._lockReason = null;
        this._lockedAt = null;
    },
    
    lock(reason) {
        this._isLocked = true;
        this._lockReason = reason;
        this._lockedAt = new Date();
        console.log(`Locked: ${reason}`);
    },
    
    unlock() {
        this._isLocked = false;
        this._lockReason = null;
        console.log('Unlocked');
    },
    
    isLocked() {
        return this._isLocked;
    },
    
    checkLocked() {
        if (this._isLocked) {
            throw new Error(`Account locked: ${this._lockReason}`);
        }
    }
};

// Notifiable mixin
const NotifiableMixin = {
    initNotifiable() {
        this._notifications = [];
        this._notificationPreferences = {
            email: true,
            sms: true,
            push: false
        };
    },
    
    sendNotification(type, message) {
        const notification = {
            type,
            message,
            timestamp: new Date().toISOString(),
            read: false
        };
        
        this._notifications.push(notification);
        
        if (this._notificationPreferences[type]) {
            console.log(`📧 Notification [${type}]: ${message}`);
        }
    },
    
    getNotifications(unreadOnly = false) {
        if (unreadOnly) {
            return this._notifications.filter(n => !n.read);
        }
        return [...this._notifications];
    },
    
    markAllAsRead() {
        this._notifications.forEach(n => n.read = true);
    }
};

// Apply mixins to class
class EnhancedBankAccount {
    constructor(accountNumber, accountHolder, balance) {
        this.accountNumber = accountNumber;
        this.accountHolder = accountHolder;
        this.balance = balance;
        
        // Initialize mixins
        this.initAudit();
        this.initLockable();
        this.initNotifiable();
        
        this.logAudit('ACCOUNT_CREATED', { accountNumber, initialBalance: balance });
    }
    
    deposit(amount) {
        this.checkLocked();
        
        this.balance += amount;
        this.logAudit('DEPOSIT', { amount, newBalance: this.balance });
        this.sendNotification('email', `Deposited AED ${amount}`);
        
        console.log(`Deposited AED ${amount}. Balance: AED ${this.balance}`);
    }
    
    withdraw(amount) {
        this.checkLocked();
        
        if (amount > this.balance) {
            throw new Error('Insufficient funds');
        }
        
        this.balance -= amount;
        this.logAudit('WITHDRAWAL', { amount, newBalance: this.balance });
        this.sendNotification('email', `Withdrew AED ${amount}`);
        
        console.log(`Withdrew AED ${amount}. Balance: AED ${this.balance}`);
    }
}

// Mix in functionality
Object.assign(EnhancedBankAccount.prototype, AuditableMixin);
Object.assign(EnhancedBankAccount.prototype, LockableMixin);
Object.assign(EnhancedBankAccount.prototype, NotifiableMixin);

const enhancedAccount = new EnhancedBankAccount('ENH001', 'Khalid Ahmed', 10000);
enhancedAccount.deposit(5000);
enhancedAccount.withdraw(2000);

console.log('\nAudit Log:', enhancedAccount.getAuditLog());
console.log('Audit Summary:', enhancedAccount.getAuditSummary());

enhancedAccount.lock('Suspicious activity detected');
try {
    enhancedAccount.withdraw(1000);
} catch (error) {
    console.log('Error:', error.message);
}

enhancedAccount.unlock();
console.log('\nNotifications:', enhancedAccount.getNotifications());

// Composition Pattern
console.log('\n=== Composition Pattern ===\n');

// Component objects
class TransactionValidator {
    validate(transaction) {
        const errors = [];
        
        if (!transaction.amount || transaction.amount <= 0) {
            errors.push('Invalid amount');
        }
        
        if (!transaction.fromAccount) {
            errors.push('From account required');
        }
        
        if (!transaction.toAccount) {
            errors.push('To account required');
        }
        
        return {
            valid: errors.length === 0,
            errors
        };
    }
}

class FraudDetector {
    constructor() {
        this.suspiciousPatterns = new Map();
    }
    
    checkTransaction(transaction) {
        const key = transaction.fromAccount;
        const recent = this.suspiciousPatterns.get(key) || [];
        
        // Check for rapid successive transactions
        const now = Date.now();
        const recentCount = recent.filter(t => now - t < 60000).length;
        
        if (recentCount > 5) {
            return {
                suspicious: true,
                reason: 'Too many transactions in short time'
            };
        }
        
        // Check for large amount
        if (transaction.amount > 50000) {
            return {
                suspicious: true,
                reason: 'Large transaction amount'
            };
        }
        
        recent.push(now);
        this.suspiciousPatterns.set(key, recent);
        
        return { suspicious: false };
    }
}

class TransactionLogger {
    constructor() {
        this.logs = [];
    }
    
    log(transaction, status) {
        this.logs.push({
            ...transaction,
            status,
            timestamp: new Date().toISOString()
        });
    }
    
    getLogs(filter) {
        if (filter) {
            return this.logs.filter(filter);
        }
        return [...this.logs];
    }
}

// Composed Transaction Processor
class TransactionProcessor {
    constructor() {
        this.validator = new TransactionValidator();
        this.fraudDetector = new FraudDetector();
        this.logger = new TransactionLogger();
    }
    
    processTransaction(transaction) {
        console.log(`\nProcessing transaction: ${transaction.id}`);
        
        // Validate
        const validation = this.validator.validate(transaction);
        if (!validation.valid) {
            console.log('❌ Validation failed:', validation.errors);
            this.logger.log(transaction, 'VALIDATION_FAILED');
            return { success: false, errors: validation.errors };
        }
        
        // Fraud check
        const fraudCheck = this.fraudDetector.checkTransaction(transaction);
        if (fraudCheck.suspicious) {
            console.log('⚠️  Suspicious transaction:', fraudCheck.reason);
            this.logger.log(transaction, 'FRAUD_SUSPECTED');
            return { success: false, reason: fraudCheck.reason };
        }
        
        // Process
        try {
            // Simulate processing
            console.log(`✓ Transaction processed: AED ${transaction.amount}`);
            this.logger.log(transaction, 'SUCCESS');
            return { success: true };
        } catch (error) {
            console.log('❌ Processing failed:', error.message);
            this.logger.log(transaction, 'FAILED');
            return { success: false, error: error.message };
        }
    }
    
    getTransactionHistory() {
        return this.logger.getLogs();
    }
}

const processor = new TransactionProcessor();

processor.processTransaction({
    id: 'TXN001',
    fromAccount: 'ACC001',
    toAccount: 'ACC002',
    amount: 5000
});

processor.processTransaction({
    id: 'TXN002',
    fromAccount: 'ACC001',
    toAccount: 'ACC003',
    amount: -100 // Invalid
});

processor.processTransaction({
    id: 'TXN003',
    fromAccount: 'ACC001',
    toAccount: 'ACC004',
    amount: 75000 // Suspicious
});

console.log('\nTransaction History:');
console.table(processor.getTransactionHistory());

// Builder Pattern
console.log('\n=== Builder Pattern ===\n');

class AccountBuilder {
    constructor() {
        this.account = {
            features: []
        };
    }
    
    setAccountNumber(number) {
        this.account.accountNumber = number;
        return this;
    }
    
    setAccountHolder(holder) {
        this.account.accountHolder = holder;
        return this;
    }
    
    setAccountType(type) {
        this.account.accountType = type;
        return this;
    }
    
    setInitialBalance(balance) {
        this.account.balance = balance;
        return this;
    }
    
    withOnlineBanking() {
        this.account.features.push('ONLINE_BANKING');
        return this;
    }
    
    withMobileBanking() {
        this.account.features.push('MOBILE_BANKING');
        return this;
    }
    
    withChequeBook() {
        this.account.features.push('CHEQUE_BOOK');
        return this;
    }
    
    withDebitCard() {
        this.account.features.push('DEBIT_CARD');
        return this;
    }
    
    withCreditCard(limit) {
        this.account.features.push('CREDIT_CARD');
        this.account.creditLimit = limit;
        return this;
    }
    
    build() {
        if (!this.account.accountNumber || !this.account.accountHolder) {
            throw new Error('Account number and holder are required');
        }
        
        return { ...this.account };
    }
}

const premiumAccount = new AccountBuilder()
    .setAccountNumber('PRM001')
    .setAccountHolder('Mohammed Ali')
    .setAccountType('PREMIUM')
    .setInitialBalance(100000)
    .withOnlineBanking()
    .withMobileBanking()
    .withChequeBook()
    .withDebitCard()
    .withCreditCard(50000)
    .build();

console.log('Premium Account:', premiumAccount);

const basicAccount = new AccountBuilder()
    .setAccountNumber('BSC001')
    .setAccountHolder('Aisha Hassan')
    .setAccountType('BASIC')
    .setInitialBalance(5000)
    .withOnlineBanking()
    .withDebitCard()
    .build();

console.log('\nBasic Account:', basicAccount);
```

---

## Q15: Explain design patterns (Singleton, Factory, Observer) in banking context

**Answer:**

Design patterns provide reusable solutions to common problems in software design.

**Banking System with Design Patterns:**

```javascript
// Singleton Pattern - Bank Configuration
console.log('=== Singleton Pattern ===\n');

class BankConfig {
    static #instance = null;
    
    #config = {
        bankName: 'Emirates NBD',
        swiftCode: 'EBILAEAD',
        minBalance: 1000,
        maxDailyWithdrawal: 50000,
        transactionFee: 0.01,
        interestRates: {
            savings: 0.025,
            current: 0.001,
            fixedDeposit: 0.055
        }
    };
    
    constructor() {
        if (BankConfig.#instance) {
            throw new Error('Use BankConfig.getInstance()');
        }
    }
    
    static getInstance() {
        if (!BankConfig.#instance) {
            BankConfig.#instance = new BankConfig();
        }
        return BankConfig.#instance;
    }
    
    get(key) {
        return this.#config[key];
    }
    
    set(key, value) {
        this.#config[key] = value;
        console.log(`Config updated: ${key} = ${value}`);
    }
    
    getInterestRate(accountType) {
        return this.#config.interestRates[accountType] || 0;
    }
    
    getAllConfig() {
        return { ...this.#config };
    }
}

const config1 = BankConfig.getInstance();
const config2 = BankConfig.getInstance();

console.log('Same instance?', config1 === config2); // true
console.log('Bank Name:', config1.get('bankName'));
console.log('Savings Rate:', config1.getInterestRate('savings'));

config1.set('maxDailyWithdrawal', 75000);
console.log('Updated limit from config2:', config2.get('maxDailyWithdrawal'));

// Factory Pattern - Account Creation
console.log('\n=== Factory Pattern ===\n');

class AccountFactory {
    static createAccount(type, accountNumber, holder, balance, ...extraParams) {
        switch (type.toUpperCase()) {
            case 'SAVINGS':
                return new SavingsAccountImpl(accountNumber, holder, balance);
            
            case 'CURRENT':
                const [overdraftLimit] = extraParams;
                return new CurrentAccountImpl(accountNumber, holder, balance, overdraftLimit);
            
            case 'FIXED_DEPOSIT':
                const [tenure] = extraParams;
                return new FixedDepositAccountImpl(accountNumber, holder, balance, tenure);
            
            default:
                throw new Error(`Unknown account type: ${type}`);
        }
    }
    
    static createAccountWithConfig(config) {
        return this.createAccount(
            config.type,
            config.accountNumber,
            config.holder,
            config.balance,
            ...config.extraParams || []
        );
    }
}

class SavingsAccountImpl {
    constructor(accountNumber, holder, balance) {
        this.accountNumber = accountNumber;
        this.holder = holder;
        this.balance = balance;
        this.type = 'SAVINGS';
        this.interestRate = 0.025;
    }
    
    getInfo() {
        return `Savings Account ${this.accountNumber}: ${this.holder} - AED ${this.balance}`;
    }
}

class CurrentAccountImpl {
    constructor(accountNumber, holder, balance, overdraftLimit) {
        this.accountNumber = accountNumber;
        this.holder = holder;
        this.balance = balance;
        this.type = 'CURRENT';
        this.overdraftLimit = overdraftLimit;
    }
    
    getInfo() {
        return `Current Account ${this.accountNumber}: ${this.holder} - AED ${this.balance} (OD: ${this.overdraftLimit})`;
    }
}

class FixedDepositAccountImpl {
    constructor(accountNumber, holder, balance, tenure) {
        this.accountNumber = accountNumber;
        this.holder = holder;
        this.balance = balance;
        this.type = 'FIXED_DEPOSIT';
        this.tenure = tenure;
    }
    
    getInfo() {
        return `Fixed Deposit ${this.accountNumber}: ${this.holder} - AED ${this.balance} (${this.tenure})`;
    }
}

// Create accounts using factory
const acc1 = AccountFactory.createAccount('SAVINGS', 'SAV001', 'Ahmad', 10000);
const acc2 = AccountFactory.createAccount('CURRENT', 'CUR001', 'Fatima', 5000, 10000);
const acc3 = AccountFactory.createAccount('FIXED_DEPOSIT', 'FD001', 'Omar', 50000, '3years');

console.log(acc1.getInfo());
console.log(acc2.getInfo());
console.log(acc3.getInfo());

// Observer Pattern - Event System
console.log('\n=== Observer Pattern ===\n');

class EventEmitter {
    constructor() {
        this.events = new Map();
    }
    
    on(eventName, callback) {
        if (!this.events.has(eventName)) {
            this.events.set(eventName, []);
        }
        this.events.get(eventName).push(callback);
    }
    
    off(eventName, callback) {
        if (!this.events.has(eventName)) return;
        
        const callbacks = this.events.get(eventName);
        const index = callbacks.indexOf(callback);
        if (index > -1) {
            callbacks.splice(index, 1);
        }
    }
    
    emit(eventName, data) {
        if (!this.events.has(eventName)) return;
        
        const callbacks = this.events.get(eventName);
        callbacks.forEach(callback => callback(data));
    }
}

class ObservableAccount extends EventEmitter {
    constructor(accountNumber, holder, balance) {
        super();
        this.accountNumber = accountNumber;
        this.holder = holder;
        this.balance = balance;
    }
    
    deposit(amount) {
        this.balance += amount;
        this.emit('deposit', { accountNumber: this.accountNumber, amount, balance: this.balance });
    }
    
    withdraw(amount) {
        if (amount > this.balance) {
            this.emit('error', { accountNumber: this.accountNumber, error: 'Insufficient funds' });
            throw new Error('Insufficient funds');
        }
        
        this.balance -= amount;
        this.emit('withdrawal', { accountNumber: this.accountNumber, amount, balance: this.balance });
        
        if (this.balance < 1000) {
            this.emit('lowBalance', { accountNumber: this.accountNumber, balance: this.balance });
        }
    }
}

// Observers
class EmailNotifier {
    update(data) {
        console.log(`📧 Email: Transaction on account ${data.accountNumber}`);
    }
}

class SMSNotifier {
    update(data) {
        console.log(`📱 SMS: Balance updated to AED ${data.balance}`);
    }
}

class FraudMonitor {
    update(data) {
        if (data.amount > 10000) {
            console.log(`🚨 Fraud Alert: Large transaction of AED ${data.amount}`);
        }
    }
}

class BalanceMonitor {
    update(data) {
        console.log(`⚠️  Low Balance Alert: Account ${data.accountNumber} has only AED ${data.balance}`);
    }
}

// Setup observers
const observableAcc = new ObservableAccount('OBS001', 'Ali Hassan', 5000);

const emailNotifier = new EmailNotifier();
const smsNotifier = new SMSNotifier();
const fraudMonitor = new FraudMonitor();
const balanceMonitor = new BalanceMonitor();

observableAcc.on('deposit', (data) => {
    emailNotifier.update(data);
    smsNotifier.update(data);
    fraudMonitor.update(data);
});

observableAcc.on('withdrawal', (data) => {
    emailNotifier.update(data);
    smsNotifier.update(data);
});

observableAcc.on('lowBalance', (data) => {
    balanceMonitor.update(data);
});

observableAcc.on('error', (data) => {
    console.log(`❌ Error: ${data.error}`);
});

// Trigger events
console.log('\nDeposit AED 3000:');
observableAcc.deposit(3000);

console.log('\nWithdraw AED 7500:');
observableAcc.withdraw(7500);

console.log('\nTry to withdraw AED 2000 (insufficient):');
try {
    observableAcc.withdraw(2000);
} catch (error) {
    // Error event already emitted
}

// Strategy Pattern
console.log('\n=== Strategy Pattern ===\n');

class InterestCalculationStrategy {
    calculate(balance) {
        throw new Error('Must implement calculate method');
    }
}

class SimpleInterestStrategy extends InterestCalculationStrategy {
    constructor(rate) {
        super();
        this.rate = rate;
    }
    
    calculate(balance) {
        return balance * this.rate;
    }
}

class CompoundInterestStrategy extends InterestCalculationStrategy {
    constructor(rate, frequency = 12) {
        super();
        this.rate = rate;
        this.frequency = frequency;
    }
    
    calculate(balance) {
        return balance * Math.pow(1 + this.rate / this.frequency, this.frequency) - balance;
    }
}

class TieredInterestStrategy extends InterestCalculationStrategy {
    constructor() {
        super();
        this.tiers = [
            { min: 0, max: 10000, rate: 0.02 },
            { min: 10000, max: 50000, rate: 0.025 },
            { min: 50000, max: Infinity, rate: 0.03 }
        ];
    }
    
    calculate(balance) {
        let interest = 0;
        
        for (const tier of this.tiers) {
            if (balance > tier.min) {
                const amount = Math.min(balance, tier.max) - tier.min;
                interest += amount * tier.rate;
            }
        }
        
        return interest;
    }
}

class AccountWithStrategy {
    constructor(accountNumber, balance, interestStrategy) {
        this.accountNumber = accountNumber;
        this.balance = balance;
        this.interestStrategy = interestStrategy;
    }
    
    setInterestStrategy(strategy) {
        this.interestStrategy = strategy;
    }
    
    calculateInterest() {
        return this.interestStrategy.calculate(this.balance);
    }
    
    applyInterest() {
        const interest = this.calculateInterest();
        this.balance += interest;
        console.log(`Interest applied: AED ${interest.toFixed(2)}, New balance: AED ${this.balance.toFixed(2)}`);
    }
}

const strategyAcc1 = new AccountWithStrategy('STR001', 25000, new SimpleInterestStrategy(0.025));
console.log('Simple interest:', strategyAcc1.calculateInterest());

strategyAcc1.setInterestStrategy(new CompoundInterestStrategy(0.025));
console.log('Compound interest:', strategyAcc1.calculateInterest());

strategyAcc1.setInterestStrategy(new TieredInterestStrategy());
console.log('Tiered interest:', strategyAcc1.calculateInterest());
```

---
