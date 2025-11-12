# JavaScript Interview Questions (Q13-Q15): Modules & Design Patterns

## Q13: Explain JavaScript module systems (CommonJS, ES Modules) in banking applications

**Answer:**

Module systems organize code into reusable, maintainable units. Understanding both CommonJS (Node.js) and ES Modules (modern JavaScript) is essential.

**Module Systems Comparison:**

```javascript
// ============================================
// CommonJS (Node.js) - Synchronous Loading
// ============================================

// account-service.js (CommonJS)
class AccountService {
    constructor(database) {
        this.db = database;
    }
    
    async getAccount(accountId) {
        return await this.db.query('SELECT * FROM accounts WHERE id = ?', [accountId]);
    }
    
    async createAccount(accountData) {
        const { accountHolder, accountType, initialBalance } = accountData;
        
        return await this.db.query(
            'INSERT INTO accounts (holder, type, balance) VALUES (?, ?, ?)',
            [accountHolder, accountType, initialBalance]
        );
    }
    
    async updateBalance(accountId, newBalance) {
        return await this.db.query(
            'UPDATE accounts SET balance = ? WHERE id = ?',
            [newBalance, accountId]
        );
    }
}

// Export (CommonJS)
module.exports = AccountService;

// Alternative: Named exports
module.exports = {
    AccountService,
    version: '1.0.0'
};

// transaction-service.js (CommonJS Import)
const AccountService = require('./account-service');
const { validateTransaction } = require('./validators');
const config = require('./config');

class TransactionService {
    constructor() {
        this.accountService = new AccountService(config.database);
    }
    
    async processTransaction(transactionData) {
        if (!validateTransaction(transactionData)) {
            throw new Error('Invalid transaction');
        }
        
        // Process transaction
        console.log('Processing transaction:', transactionData);
    }
}

module.exports = TransactionService;

// ============================================
// ES Modules (ESM) - Asynchronous, Tree-Shakeable
// ============================================

// account-service.mjs (ES Module)
export class AccountService {
    constructor(database) {
        this.db = database;
    }
    
    async getAccount(accountId) {
        return await this.db.query('SELECT * FROM accounts WHERE id = ?', [accountId]);
    }
    
    async createAccount(accountData) {
        const { accountHolder, accountType, initialBalance } = accountData;
        
        return await this.db.query(
            'INSERT INTO accounts (holder, type, balance) VALUES (?, ?, ?)',
            [accountHolder, accountType, initialBalance]
        );
    }
}

// Named exports
export const ACCOUNT_TYPES = {
    SAVINGS: 'SAVINGS',
    CURRENT: 'CURRENT',
    FIXED_DEPOSIT: 'FIXED_DEPOSIT'
};

export const DEFAULT_CURRENCY = 'AED';

// Default export
export default AccountService;

// transaction-service.mjs (ES Module Import)
import AccountService, { ACCOUNT_TYPES, DEFAULT_CURRENCY } from './account-service.mjs';
import { validateTransaction } from './validators.mjs';
import config from './config.mjs';

export class TransactionService {
    constructor() {
        this.accountService = new AccountService(config.database);
    }
    
    async processTransaction(transactionData) {
        if (!validateTransaction(transactionData)) {
            throw new Error('Invalid transaction');
        }
        
        console.log('Processing transaction:', transactionData);
    }
}

// Dynamic imports (lazy loading)
export async function loadReportingModule() {
    const reportingModule = await import('./reporting.mjs');
    return reportingModule;
}

// ============================================
// Banking Application Module Structure
// ============================================

// services/database.mjs
export class DatabaseService {
    constructor(connectionString) {
        this.connectionString = connectionString;
        this.pool = null;
    }
    
    async connect() {
        console.log('Connecting to database...');
        // Connection logic
    }
    
    async query(sql, params) {
        console.log('Executing query:', sql);
        // Query logic
        return [];
    }
    
    async disconnect() {
        console.log('Disconnecting from database...');
    }
}

// services/authentication.mjs
import jwt from 'jsonwebtoken';
import bcrypt from 'bcrypt';

export class AuthenticationService {
    constructor(secretKey) {
        this.secretKey = secretKey;
    }
    
    async hashPassword(password) {
        return await bcrypt.hash(password, 10);
    }
    
    async verifyPassword(password, hash) {
        return await bcrypt.compare(password, hash);
    }
    
    generateToken(userId, accountId) {
        return jwt.sign(
            { userId, accountId },
            this.secretKey,
            { expiresIn: '1h' }
        );
    }
    
    verifyToken(token) {
        try {
            return jwt.verify(token, this.secretKey);
        } catch (error) {
            throw new Error('Invalid token');
        }
    }
}

// models/account.mjs
export class Account {
    constructor(data) {
        this.accountNumber = data.accountNumber;
        this.accountHolder = data.accountHolder;
        this.accountType = data.accountType;
        this.balance = data.balance;
        this.currency = data.currency || 'AED';
        this.status = data.status || 'ACTIVE';
        this.createdAt = data.createdAt || new Date();
    }
    
    deposit(amount) {
        if (amount <= 0) {
            throw new Error('Deposit amount must be positive');
        }
        this.balance += amount;
    }
    
    withdraw(amount) {
        if (amount <= 0) {
            throw new Error('Withdrawal amount must be positive');
        }
        if (amount > this.balance) {
            throw new Error('Insufficient funds');
        }
        this.balance -= amount;
    }
    
    toJSON() {
        return {
            accountNumber: this.accountNumber,
            accountHolder: this.accountHolder,
            accountType: this.accountType,
            balance: this.balance,
            currency: this.currency,
            status: this.status
        };
    }
}

// models/transaction.mjs
export class Transaction {
    constructor(data) {
        this.transactionId = data.transactionId;
        this.accountId = data.accountId;
        this.type = data.type; // CREDIT, DEBIT, TRANSFER
        this.amount = data.amount;
        this.description = data.description;
        this.status = data.status || 'PENDING';
        this.timestamp = data.timestamp || new Date();
    }
    
    complete() {
        this.status = 'COMPLETED';
    }
    
    fail(reason) {
        this.status = 'FAILED';
        this.failureReason = reason;
    }
}

// utils/validators.mjs
export function validateAccountNumber(accountNumber) {
    const pattern = /^ENBD\d{10}$/;
    return pattern.test(accountNumber);
}

export function validateAmount(amount) {
    return typeof amount === 'number' && amount > 0;
}

export function validateTransaction(transaction) {
    if (!transaction.accountId) return false;
    if (!validateAmount(transaction.amount)) return false;
    if (!['CREDIT', 'DEBIT', 'TRANSFER'].includes(transaction.type)) return false;
    return true;
}

// config/app-config.mjs
export default {
    database: {
        host: process.env.DB_HOST || 'localhost',
        port: process.env.DB_PORT || 5432,
        database: 'enbd_banking',
        user: process.env.DB_USER,
        password: process.env.DB_PASSWORD
    },
    jwt: {
        secret: process.env.JWT_SECRET || 'default-secret',
        expiresIn: '1h'
    },
    api: {
        port: process.env.PORT || 3000,
        rateLimit: {
            maxRequests: 100,
            windowMs: 60000
        }
    }
};

// Main application file
// app.mjs
import { DatabaseService } from './services/database.mjs';
import { AuthenticationService } from './services/authentication.mjs';
import { AccountService } from './services/account-service.mjs';
import { TransactionService } from './services/transaction-service.mjs';
import appConfig from './config/app-config.mjs';

class BankingApplication {
    constructor() {
        this.db = new DatabaseService(appConfig.database);
        this.auth = new AuthenticationService(appConfig.jwt.secret);
        this.accountService = new AccountService(this.db);
        this.transactionService = new TransactionService();
    }
    
    async start() {
        console.log('Starting Emirates NBD Banking Application...');
        
        await this.db.connect();
        
        console.log('Application started successfully');
    }
    
    async stop() {
        console.log('Stopping application...');
        await this.db.disconnect();
    }
}

// Create and start application
const app = new BankingApplication();
await app.start();

// ============================================
// Module Patterns and Best Practices
// ============================================

// Singleton Pattern with Modules
// logger.mjs
class Logger {
    constructor() {
        if (Logger.instance) {
            return Logger.instance;
        }
        
        this.logs = [];
        Logger.instance = this;
    }
    
    log(level, message, metadata = {}) {
        const logEntry = {
            timestamp: new Date().toISOString(),
            level,
            message,
            metadata
        };
        
        this.logs.push(logEntry);
        console.log(`[${level}] ${message}`, metadata);
    }
    
    info(message, metadata) {
        this.log('INFO', message, metadata);
    }
    
    error(message, metadata) {
        this.log('ERROR', message, metadata);
    }
    
    warn(message, metadata) {
        this.log('WARN', message, metadata);
    }
}

export default new Logger(); // Export singleton instance

// Usage
import logger from './logger.mjs';

logger.info('Transaction processed', { transactionId: 'TXN001' });
logger.error('Transaction failed', { transactionId: 'TXN002', reason: 'Insufficient funds' });

// Factory Pattern with Modules
// account-factory.mjs
import { Account } from './models/account.mjs';

export class AccountFactory {
    static createAccount(type, data) {
        const accountNumber = `ENBD${Date.now()}${Math.floor(Math.random() * 1000)}`;
        
        const baseData = {
            accountNumber,
            ...data
        };
        
        switch (type) {
            case 'SAVINGS':
                return new SavingsAccount(baseData);
            case 'CURRENT':
                return new CurrentAccount(baseData);
            case 'FIXED_DEPOSIT':
                return new FixedDepositAccount(baseData);
            default:
                throw new Error(`Unknown account type: ${type}`);
        }
    }
}

class SavingsAccount extends Account {
    constructor(data) {
        super({ ...data, accountType: 'SAVINGS' });
        this.interestRate = 0.025;
        this.minimumBalance = 1000;
    }
    
    calculateInterest() {
        return this.balance * this.interestRate;
    }
}

class CurrentAccount extends Account {
    constructor(data) {
        super({ ...data, accountType: 'CURRENT' });
        this.overdraftLimit = 10000;
    }
    
    withdraw(amount) {
        const availableBalance = this.balance + this.overdraftLimit;
        
        if (amount > availableBalance) {
            throw new Error('Exceeds overdraft limit');
        }
        
        this.balance -= amount;
    }
}

class FixedDepositAccount extends Account {
    constructor(data) {
        super({ ...data, accountType: 'FIXED_DEPOSIT' });
        this.interestRate = 0.055;
        this.maturityDate = new Date(Date.now() + 365 * 24 * 60 * 60 * 1000);
        this.locked = true;
    }
    
    withdraw(amount) {
        if (this.locked) {
            throw new Error('Account is locked until maturity');
        }
        super.withdraw(amount);
    }
}

// Re-exporting for convenience
// index.mjs
export { AccountService } from './services/account-service.mjs';
export { TransactionService } from './services/transaction-service.mjs';
export { AuthenticationService } from './services/authentication.mjs';
export { Account } from './models/account.mjs';
export { Transaction } from './models/transaction.mjs';
export { AccountFactory } from './account-factory.mjs';
export { validateAccountNumber, validateTransaction } from './utils/validators.mjs';

// Barrel exports
export * from './constants.mjs';
```

---

## Q14: Explain design patterns (Singleton, Factory, Observer) in banking context

**Answer:**

Design patterns provide reusable solutions to common software design problems. Here are key patterns for banking applications.

**Banking Design Patterns:**

```javascript
// ============================================
// 1. SINGLETON PATTERN
// ============================================

// Database Connection Pool (Singleton)
class DatabaseConnectionPool {
    constructor() {
        if (DatabaseConnectionPool.instance) {
            return DatabaseConnectionPool.instance;
        }
        
        this.connections = [];
        this.maxConnections = 10;
        this.activeConnections = 0;
        
        DatabaseConnectionPool.instance = this;
    }
    
    async getConnection() {
        if (this.activeConnections < this.maxConnections) {
            const connection = await this.createConnection();
            this.activeConnections++;
            return connection;
        }
        
        // Wait for available connection
        return await this.waitForConnection();
    }
    
    async createConnection() {
        console.log('Creating new database connection');
        return {
            id: Date.now(),
            query: async (sql) => {
                console.log('Executing:', sql);
                return [];
            }
        };
    }
    
    async waitForConnection() {
        // Wait and retry logic
        await new Promise(resolve => setTimeout(resolve, 100));
        return await this.getConnection();
    }
    
    releaseConnection(connection) {
        this.activeConnections--;
        console.log(`Connection ${connection.id} released`);
    }
    
    static getInstance() {
        if (!DatabaseConnectionPool.instance) {
            DatabaseConnectionPool.instance = new DatabaseConnectionPool();
        }
        return DatabaseConnectionPool.instance;
    }
}

// Usage
const pool1 = DatabaseConnectionPool.getInstance();
const pool2 = DatabaseConnectionPool.getInstance();
console.log(pool1 === pool2); // true - same instance

// Configuration Manager (Singleton)
class ConfigurationManager {
    constructor() {
        if (ConfigurationManager.instance) {
            return ConfigurationManager.instance;
        }
        
        this.config = {
            apiKey: process.env.API_KEY,
            database: {
                host: process.env.DB_HOST || 'localhost',
                port: process.env.DB_PORT || 5432
            },
            features: {
                enableTransfers: true,
                enableInternationalTransfers: false,
                maxDailyLimit: 50000
            }
        };
        
        ConfigurationManager.instance = this;
    }
    
    get(key) {
        return this.config[key];
    }
    
    set(key, value) {
        this.config[key] = value;
    }
    
    static getInstance() {
        if (!ConfigurationManager.instance) {
            ConfigurationManager.instance = new ConfigurationManager();
        }
        return ConfigurationManager.instance;
    }
}

// ============================================
// 2. FACTORY PATTERN
// ============================================

// Account Factory
class AccountFactory {
    static createAccount(type, data) {
        switch (type) {
            case 'SAVINGS':
                return new SavingsAccount(data);
            case 'CURRENT':
                return new CurrentAccount(data);
            case 'FIXED_DEPOSIT':
                return new FixedDepositAccount(data);
            case 'SALARY':
                return new SalaryAccount(data);
            default:
                throw new Error(`Unknown account type: ${type}`);
        }
    }
    
    static createAccountFromData(accountData) {
        const type = accountData.accountType;
        return this.createAccount(type, accountData);
    }
}

class BankAccount {
    constructor(data) {
        this.accountNumber = data.accountNumber;
        this.accountHolder = data.accountHolder;
        this.balance = data.balance || 0;
        this.currency = data.currency || 'AED';
    }
    
    deposit(amount) {
        this.balance += amount;
    }
    
    withdraw(amount) {
        if (amount > this.balance) {
            throw new Error('Insufficient funds');
        }
        this.balance -= amount;
    }
}

class SavingsAccount extends BankAccount {
    constructor(data) {
        super(data);
        this.accountType = 'SAVINGS';
        this.interestRate = 0.025;
        this.minimumBalance = 1000;
    }
    
    calculateMonthlyInterest() {
        return this.balance * (this.interestRate / 12);
    }
    
    withdraw(amount) {
        if (this.balance - amount < this.minimumBalance) {
            throw new Error(`Minimum balance of ${this.minimumBalance} required`);
        }
        super.withdraw(amount);
    }
}

class CurrentAccount extends BankAccount {
    constructor(data) {
        super(data);
        this.accountType = 'CURRENT';
        this.overdraftLimit = data.overdraftLimit || 10000;
        this.monthlyFee = 50;
    }
    
    withdraw(amount) {
        const availableBalance = this.balance + this.overdraftLimit;
        
        if (amount > availableBalance) {
            throw new Error('Exceeds overdraft limit');
        }
        
        this.balance -= amount;
    }
    
    getAvailableBalance() {
        return this.balance + this.overdraftLimit;
    }
}

class FixedDepositAccount extends BankAccount {
    constructor(data) {
        super(data);
        this.accountType = 'FIXED_DEPOSIT';
        this.interestRate = 0.055;
        this.term = data.term || 12; // months
        this.maturityDate = new Date();
        this.maturityDate.setMonth(this.maturityDate.getMonth() + this.term);
        this.earlyWithdrawalPenalty = 0.01;
    }
    
    withdraw(amount) {
        const now = new Date();
        
        if (now < this.maturityDate) {
            const penalty = amount * this.earlyWithdrawalPenalty;
            console.log(`Early withdrawal penalty: ${penalty}`);
            amount += penalty;
        }
        
        super.withdraw(amount);
    }
    
    calculateMaturityAmount() {
        const years = this.term / 12;
        return this.balance * Math.pow(1 + this.interestRate, years);
    }
}

class SalaryAccount extends CurrentAccount {
    constructor(data) {
        super(data);
        this.accountType = 'SALARY';
        this.monthlyFee = 0; // No fees for salary accounts
        this.minimumSalary = 5000;
    }
    
    validateSalaryCredit(amount) {
        if (amount < this.minimumSalary) {
            console.warn(`Salary amount below minimum: ${amount}`);
        }
    }
}

// Usage
const savingsAccount = AccountFactory.createAccount('SAVINGS', {
    accountNumber: 'ENBD001',
    accountHolder: 'Ahmed Al Mansouri',
    balance: 10000
});

console.log(savingsAccount.calculateMonthlyInterest()); // Calculate interest

// Transaction Factory
class TransactionFactory {
    static createTransaction(type, data) {
        switch (type) {
            case 'DEPOSIT':
                return new DepositTransaction(data);
            case 'WITHDRAWAL':
                return new WithdrawalTransaction(data);
            case 'TRANSFER':
                return new TransferTransaction(data);
            case 'BILL_PAYMENT':
                return new BillPaymentTransaction(data);
            default:
                throw new Error(`Unknown transaction type: ${type}`);
        }
    }
}

class Transaction {
    constructor(data) {
        this.id = data.id || `TXN${Date.now()}`;
        this.accountId = data.accountId;
        this.amount = data.amount;
        this.timestamp = new Date();
        this.status = 'PENDING';
    }
    
    async execute() {
        throw new Error('execute() must be implemented');
    }
    
    complete() {
        this.status = 'COMPLETED';
    }
    
    fail(reason) {
        this.status = 'FAILED';
        this.failureReason = reason;
    }
}

class DepositTransaction extends Transaction {
    constructor(data) {
        super(data);
        this.type = 'DEPOSIT';
        this.source = data.source;
    }
    
    async execute() {
        console.log(`Depositing ${this.amount} to account ${this.accountId}`);
        // Deposit logic
        this.complete();
    }
}

class WithdrawalTransaction extends Transaction {
    constructor(data) {
        super(data);
        this.type = 'WITHDRAWAL';
        this.atmId = data.atmId;
    }
    
    async execute() {
        console.log(`Withdrawing ${this.amount} from account ${this.accountId}`);
        // Withdrawal logic
        this.complete();
    }
}

class TransferTransaction extends Transaction {
    constructor(data) {
        super(data);
        this.type = 'TRANSFER';
        this.toAccountId = data.toAccountId;
    }
    
    async execute() {
        console.log(`Transferring ${this.amount} from ${this.accountId} to ${this.toAccountId}`);
        // Transfer logic
        this.complete();
    }
}

class BillPaymentTransaction extends Transaction {
    constructor(data) {
        super(data);
        this.type = 'BILL_PAYMENT';
        this.billerName = data.billerName;
        this.billNumber = data.billNumber;
    }
    
    async execute() {
        console.log(`Paying bill ${this.billNumber} to ${this.billerName}`);
        // Bill payment logic
        this.complete();
    }
}

// ============================================
// 3. OBSERVER PATTERN
// ============================================

// Event Emitter for Banking Events
class EventEmitter {
    constructor() {
        this.events = {};
    }
    
    on(event, listener) {
        if (!this.events[event]) {
            this.events[event] = [];
        }
        this.events[event].push(listener);
    }
    
    off(event, listenerToRemove) {
        if (!this.events[event]) return;
        
        this.events[event] = this.events[event].filter(
            listener => listener !== listenerToRemove
        );
    }
    
    emit(event, data) {
        if (!this.events[event]) return;
        
        this.events[event].forEach(listener => {
            listener(data);
        });
    }
    
    once(event, listener) {
        const onceWrapper = (data) => {
            listener(data);
            this.off(event, onceWrapper);
        };
        
        this.on(event, onceWrapper);
    }
}

// Account with Observable Events
class ObservableAccount extends EventEmitter {
    constructor(data) {
        super();
        this.accountNumber = data.accountNumber;
        this.accountHolder = data.accountHolder;
        this._balance = data.balance || 0;
    }
    
    get balance() {
        return this._balance;
    }
    
    set balance(value) {
        const oldBalance = this._balance;
        this._balance = value;
        
        this.emit('balanceChanged', {
            accountNumber: this.accountNumber,
            oldBalance,
            newBalance: value,
            timestamp: new Date()
        });
    }
    
    deposit(amount) {
        this._balance += amount;
        
        this.emit('deposit', {
            accountNumber: this.accountNumber,
            amount,
            newBalance: this._balance,
            timestamp: new Date()
        });
        
        this.emit('balanceChanged', {
            accountNumber: this.accountNumber,
            oldBalance: this._balance - amount,
            newBalance: this._balance,
            timestamp: new Date()
        });
    }
    
    withdraw(amount) {
        if (amount > this._balance) {
            this.emit('withdrawalFailed', {
                accountNumber: this.accountNumber,
                amount,
                reason: 'Insufficient funds',
                timestamp: new Date()
            });
            throw new Error('Insufficient funds');
        }
        
        this._balance -= amount;
        
        this.emit('withdrawal', {
            accountNumber: this.accountNumber,
            amount,
            newBalance: this._balance,
            timestamp: new Date()
        });
        
        this.emit('balanceChanged', {
            accountNumber: this.accountNumber,
            oldBalance: this._balance + amount,
            newBalance: this._balance,
            timestamp: new Date()
        });
    }
}

// Observers (Services listening to events)
class NotificationService {
    handleBalanceChange(data) {
        console.log(`\n[Notification] Balance changed for ${data.accountNumber}`);
        console.log(`Old: ${data.oldBalance}, New: ${data.newBalance}`);
        // Send SMS/Email notification
    }
    
    handleLowBalance(data) {
        console.log(`\n[Alert] Low balance warning for ${data.accountNumber}: ${data.balance}`);
        // Send alert
    }
    
    handleDeposit(data) {
        console.log(`\n[Notification] Deposit of ${data.amount} to ${data.accountNumber}`);
        // Send confirmation
    }
}

class AuditService {
    logTransaction(data) {
        console.log(`\n[Audit] Transaction logged:`, data);
        // Write to audit log
    }
}

class FraudDetectionService {
    checkTransaction(data) {
        console.log(`\n[Fraud Detection] Checking transaction:`, data);
        
        // Fraud detection logic
        if (data.amount > 50000) {
            console.log('⚠️  Large transaction flagged for review');
        }
    }
}

// Usage
const account = new ObservableAccount({
    accountNumber: 'ENBD001',
    accountHolder: 'Ahmed Al Mansouri',
    balance: 10000
});

const notificationService = new NotificationService();
const auditService = new AuditService();
const fraudDetection = new FraudDetectionService();

// Subscribe to events
account.on('balanceChanged', (data) => notificationService.handleBalanceChange(data));
account.on('deposit', (data) => {
    notificationService.handleDeposit(data);
    auditService.logTransaction(data);
    fraudDetection.checkTransaction(data);
});
account.on('withdrawal', (data) => auditService.logTransaction(data));

// Trigger events
account.deposit(5000);
account.withdraw(2000);

console.log(`\nFinal balance: ${account.balance}`);
```

---

## Q15: Explain functional programming concepts in JavaScript with banking examples

**Answer:**

Functional programming emphasizes pure functions, immutability, and function composition for more predictable and maintainable code.

**Functional Programming in Banking:**

```javascript
// ============================================
// 1. PURE FUNCTIONS
// ============================================

// Pure function: Same input always produces same output, no side effects
const calculateSimpleInterest = (principal, rate, time) => {
    return principal * rate * time;
};

console.log(calculateSimpleInterest(10000, 0.05, 2)); // 1000
console.log(calculateSimpleInterest(10000, 0.05, 2)); // 1000 (always same)

// Impure function (has side effects)
let totalTransactions = 0;

const processTransactionImpure = (amount) => {
    totalTransactions++; // Side effect: modifies external state
    console.log('Processing...'); // Side effect: I/O
    return amount * 1.05;
};

// Pure alternative
const processTransactionPure = (amount, transactionCount) => {
    return {
        result: amount * 1.05,
        newCount: transactionCount + 1
    };
};

// ============================================
// 2. IMMUTABILITY
// ============================================

// Immutable account operations
const createAccount = (accountHolder, initialBalance) => ({
    accountNumber: `ENBD${Date.now()}`,
    accountHolder,
    balance: initialBalance,
    transactions: [],
    createdAt: new Date().toISOString()
});

// Return new object instead of mutating
const deposit = (account, amount) => ({
    ...account,
    balance: account.balance + amount,
    transactions: [
        ...account.transactions,
        {
            type: 'DEPOSIT',
            amount,
            timestamp: new Date().toISOString()
        }
    ]
});

const withdraw = (account, amount) => {
    if (amount > account.balance) {
        throw new Error('Insufficient funds');
    }
    
    return {
        ...account,
        balance: account.balance - amount,
        transactions: [
            ...account.transactions,
            {
                type: 'WITHDRAWAL',
                amount,
                timestamp: new Date().toISOString()
            }
        ]
    };
};

// Usage
let account = createAccount('Ahmed Al Mansouri', 10000);
console.log('Initial balance:', account.balance);

account = deposit(account, 5000);
console.log('After deposit:', account.balance);

account = withdraw(account, 2000);
console.log('After withdrawal:', account.balance);

// ============================================
// 3. HIGHER-ORDER FUNCTIONS
// ============================================

// Function that returns a function
const createInterestCalculator = (rate) => {
    return (principal, years) => {
        return principal * Math.pow(1 + rate, years) - principal;
    };
};

const calculateSavingsInterest = createInterestCalculator(0.025);
const calculateFixedDepositInterest = createInterestCalculator(0.055);

console.log('Savings interest:', calculateSavingsInterest(10000, 5));
console.log('FD interest:', calculateFixedDepositInterest(10000, 5));

// Function that takes function as argument
const applyTransactionFee = (transaction, feeCalculator) => ({
    ...transaction,
    fee: feeCalculator(transaction.amount),
    netAmount: transaction.amount - feeCalculator(transaction.amount)
});

const domesticTransferFee = (amount) => amount * 0.001; // 0.1%
const internationalTransferFee = (amount) => amount * 0.025; // 2.5%

const domesticTxn = applyTransactionFee(
    { id: 'TXN001', amount: 10000 },
    domesticTransferFee
);

console.log('Domestic transfer:', domesticTxn);

// ============================================
// 4. FUNCTION COMPOSITION
// ============================================

// compose: right to left
const compose = (...fns) => (value) =>
    fns.reduceRight((acc, fn) => fn(acc), value);

// pipe: left to right
const pipe = (...fns) => (value) =>
    fns.reduce((acc, fn) => fn(acc), value);

// Banking pipeline
const validateAmount = (amount) => {
    if (amount <= 0) throw new Error('Invalid amount');
    return amount;
};

const applyServiceCharge = (amount) => amount - (amount * 0.005);

const roundToTwoDecimals = (amount) => Math.round(amount * 100) / 100;

const formatCurrency = (amount) => `AED ${amount.toFixed(2)}`;

// Compose transaction processing pipeline
const processPayment = pipe(
    validateAmount,
    applyServiceCharge,
    roundToTwoDecimals,
    formatCurrency
);

console.log(processPayment(10000)); // "AED 9950.00"

// ============================================
// 5. MAP, FILTER, REDUCE
// ============================================

const transactions = [
    { id: 1, type: 'CREDIT', amount: 5000, category: 'SALARY' },
    { id: 2, type: 'DEBIT', amount: 1200, category: 'RENT' },
    { id: 3, type: 'CREDIT', amount: 3000, category: 'TRANSFER' },
    { id: 4, type: 'DEBIT', amount: 500, category: 'GROCERIES' },
    { id: 5, type: 'DEBIT', amount: 2000, category: 'SHOPPING' }
];

// MAP: Transform transactions
const transactionsWithTax = transactions.map(txn => ({
    ...txn,
    tax: txn.type === 'CREDIT' ? txn.amount * 0.05 : 0
}));

// FILTER: Get only debits
const debits = transactions.filter(txn => txn.type === 'DEBIT');

// REDUCE: Calculate totals
const totalDebit = transactions
    .filter(txn => txn.type === 'DEBIT')
    .reduce((sum, txn) => sum + txn.amount, 0);

const totalCredit = transactions
    .filter(txn => txn.type === 'CREDIT')
    .reduce((sum, txn) => sum + txn.amount, 0);

console.log('Total Debit:', totalDebit);
console.log('Total Credit:', totalCredit);
console.log('Net:', totalCredit - totalDebit);

// Group transactions by category
const groupByCategory = (transactions) =>
    transactions.reduce((groups, txn) => ({
        ...groups,
        [txn.category]: [...(groups[txn.category] || []), txn]
    }), {});

console.log('Grouped:', groupByCategory(transactions));

// ============================================
// 6. CURRYING
// ============================================

// Curried function
const calculateLoanPayment = (principal) => (rate) => (term) => {
    const monthlyRate = rate / 12;
    const numPayments = term * 12;
    
    return (principal * monthlyRate * Math.pow(1 + monthlyRate, numPayments)) /
           (Math.pow(1 + monthlyRate, numPayments) - 1);
};

// Partial application
const calculateHomeLoan = calculateLoanPayment(500000); // Principal fixed
const calculateAt3Percent = calculateHomeLoan(0.03); // Rate fixed
const monthlyPayment = calculateAt3Percent(20); // Term fixed

console.log('Monthly payment:', monthlyPayment.toFixed(2));

// Practical currying example
const createTransactionLogger = (logLevel) => (service) => (message) => {
    const timestamp = new Date().toISOString();
    console.log(`[${timestamp}] [${logLevel}] [${service}] ${message}`);
};

const logInfo = createTransactionLogger('INFO');
const logAccountInfo = logInfo('AccountService');
const logTransactionInfo = logInfo('TransactionService');

logAccountInfo('Account created');
logTransactionInfo('Transfer completed');

// ============================================
// 7. RECURSION
// ============================================

// Calculate compound interest recursively
const calculateCompoundInterestRecursive = (principal, rate, years) => {
    if (years === 0) return principal;
    
    return calculateCompoundInterestRecursive(
        principal * (1 + rate),
        rate,
        years - 1
    );
};

console.log('Compound interest (recursive):', 
    calculateCompoundInterestRecursive(10000, 0.05, 5).toFixed(2)
);

// Process nested account structure recursively
const accountHierarchy = {
    name: 'Main',
    balance: 10000,
    subAccounts: [
        {
            name: 'Savings',
            balance: 50000,
            subAccounts: [
                { name: 'Emergency Fund', balance: 20000, subAccounts: [] },
                { name: 'Vacation Fund', balance: 10000, subAccounts: [] }
            ]
        },
        {
            name: 'Investment',
            balance: 100000,
            subAccounts: []
        }
    ]
};

const calculateTotalBalance = (account) => {
    const subTotal = account.subAccounts.reduce(
        (sum, subAccount) => sum + calculateTotalBalance(subAccount),
        0
    );
    
    return account.balance + subTotal;
};

console.log('Total balance:', calculateTotalBalance(accountHierarchy));

// ============================================
// 8. FUNCTIONAL TRANSACTION PROCESSING
// ============================================

// Functional approach to transaction processing
const TransactionProcessor = {
    validate: (transaction) => {
        if (!transaction.accountId) throw new Error('Missing account ID');
        if (transaction.amount <= 0) throw new Error('Invalid amount');
        return transaction;
    },
    
    applyFee: (transaction) => ({
        ...transaction,
        fee: transaction.amount * 0.005,
        netAmount: transaction.amount - (transaction.amount * 0.005)
    }),
    
    checkFraud: (transaction) => {
        if (transaction.amount > 50000) {
            return {
                ...transaction,
                flagged: true,
                flagReason: 'Large transaction'
            };
        }
        return { ...transaction, flagged: false };
    },
    
    addTimestamp: (transaction) => ({
        ...transaction,
        processedAt: new Date().toISOString()
    }),
    
    process: pipe(
        TransactionProcessor.validate,
        TransactionProcessor.applyFee,
        TransactionProcessor.checkFraud,
        TransactionProcessor.addTimestamp
    )
};

const txn = {
    id: 'TXN001',
    accountId: 'ACC001',
    amount: 10000,
    type: 'TRANSFER'
};

const processedTxn = TransactionProcessor.process(txn);
console.log('Processed transaction:', processedTxn);
```

---
