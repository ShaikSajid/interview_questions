# JavaScript Interview Questions (Q10-Q12): Object-Oriented Programming

## Q10: Explain classes, inheritance, and access modifiers with banking system

**Answer:**

ES6 classes provide syntactic sugar for prototypal inheritance with cleaner syntax for object-oriented programming.

**Banking System with Classes:**

```javascript
// Base BankAccount Class
class BankAccount {
    // Private fields (ES2022)
    #accountNumber;
    #balance;
    #pin;
    
    // Static property
    static bankName = 'Emirates NBD';
    static bankCode = 'EBILAEAD';
    static accountCount = 0;
    
    constructor(accountNumber, accountHolder, initialBalance, pin) {
        this.#accountNumber = accountNumber;
        this.accountHolder = accountHolder;
        this.#balance = initialBalance;
        this.#pin = pin;
        this.transactions = [];
        this.createdAt = new Date();
        this.status = 'ACTIVE';
        
        BankAccount.accountCount++;
    }
    
    // Getter for balance (controlled access)
    get balance() {
        return this.#balance;
    }
    
    // Setter for balance (validation)
    set balance(value) {
        if (value < 0) {
            throw new Error('Balance cannot be negative');
        }
        this.#balance = value;
    }
    
    // Getter for account number
    get accountNumber() {
        // Return masked number
        return this.#accountNumber.replace(/(\d{4})(\d+)(\d{4})/, '$1****$3');
    }
    
    // Public method
    deposit(amount) {
        if (amount <= 0) {
            throw new Error('Deposit amount must be positive');
        }
        
        this.#balance += amount;
        this.#recordTransaction('CREDIT', amount, 'Cash Deposit');
        
        console.log(`Deposited AED ${amount}. New balance: AED ${this.#balance}`);
        return this.#balance;
    }
    
    // Public method
    withdraw(amount, inputPin) {
        if (!this.#verifyPin(inputPin)) {
            throw new Error('Invalid PIN');
        }
        
        if (amount <= 0) {
            throw new Error('Withdrawal amount must be positive');
        }
        
        if (amount > this.#balance) {
            throw new Error('Insufficient funds');
        }
        
        this.#balance -= amount;
        this.#recordTransaction('DEBIT', amount, 'Cash Withdrawal');
        
        console.log(`Withdrew AED ${amount}. New balance: AED ${this.#balance}`);
        return this.#balance;
    }
    
    // Private method
    #verifyPin(inputPin) {
        return this.#pin === inputPin;
    }
    
    // Private method
    #recordTransaction(type, amount, description) {
        this.transactions.push({
            id: this.transactions.length + 1,
            type,
            amount,
            description,
            balance: this.#balance,
            timestamp: new Date().toISOString()
        });
    }
    
    // Protected-like method (convention: starts with _)
    _getFullAccountNumber() {
        return this.#accountNumber;
    }
    
    // Public method
    getStatement() {
        console.log(`\n=== Account Statement ===`);
        console.log(`Bank: ${BankAccount.bankName}`);
        console.log(`Account: ${this.accountNumber}`);
        console.log(`Holder: ${this.accountHolder}`);
        console.log(`Balance: AED ${this.#balance}`);
        console.log(`\nRecent Transactions:`);
        
        this.transactions.slice(-5).forEach(txn => {
            const sign = txn.type === 'CREDIT' ? '+' : '-';
            console.log(`${txn.timestamp} | ${txn.type} | ${sign}${txn.amount} | ${txn.description}`);
        });
    }
    
    // Static method
    static getTotalAccounts() {
        return BankAccount.accountCount;
    }
    
    // Static method
    static getBankInfo() {
        return {
            name: BankAccount.bankName,
            code: BankAccount.bankCode,
            totalAccounts: BankAccount.accountCount
        };
    }
}

// SavingsAccount extends BankAccount
class SavingsAccount extends BankAccount {
    #interestRate;
    #minimumBalance;
    
    static defaultInterestRate = 0.025;
    static defaultMinBalance = 1000;
    
    constructor(accountNumber, accountHolder, initialBalance, pin, interestRate) {
        super(accountNumber, accountHolder, initialBalance, pin);
        this.#interestRate = interestRate || SavingsAccount.defaultInterestRate;
        this.#minimumBalance = SavingsAccount.defaultMinBalance;
        this.accountType = 'SAVINGS';
    }
    
    // Getter
    get interestRate() {
        return this.#interestRate;
    }
    
    // Setter with validation
    set interestRate(rate) {
        if (rate < 0 || rate > 0.1) {
            throw new Error('Interest rate must be between 0 and 10%');
        }
        this.#interestRate = rate;
    }
    
    // Override withdraw method
    withdraw(amount, inputPin) {
        const newBalance = this.balance - amount;
        
        if (newBalance < this.#minimumBalance) {
            throw new Error(`Savings account must maintain minimum balance of AED ${this.#minimumBalance}`);
        }
        
        // Call parent method
        return super.withdraw(amount, inputPin);
    }
    
    // New method specific to SavingsAccount
    calculateInterest() {
        const interest = this.balance * this.#interestRate;
        console.log(`Interest calculated: AED ${interest.toFixed(2)} at ${(this.#interestRate * 100).toFixed(2)}%`);
        return interest;
    }
    
    // New method
    applyMonthlyInterest() {
        const interest = this.calculateInterest() / 12;
        this.deposit(interest);
        console.log(`Monthly interest applied: AED ${interest.toFixed(2)}`);
    }
    
    // Override toString
    toString() {
        return `SavingsAccount(${this.accountNumber}, Balance: AED ${this.balance}, Rate: ${(this.#interestRate * 100).toFixed(2)}%)`;
    }
}

// CurrentAccount extends BankAccount
class CurrentAccount extends BankAccount {
    #overdraftLimit;
    #transactionFee;
    
    static defaultOverdraftLimit = 10000;
    static defaultTransactionFee = 2;
    
    constructor(accountNumber, accountHolder, initialBalance, pin, overdraftLimit) {
        super(accountNumber, accountHolder, initialBalance, pin);
        this.#overdraftLimit = overdraftLimit || CurrentAccount.defaultOverdraftLimit;
        this.#transactionFee = CurrentAccount.defaultTransactionFee;
        this.accountType = 'CURRENT';
    }
    
    // Getter
    get overdraftLimit() {
        return this.#overdraftLimit;
    }
    
    // Override withdraw to allow overdraft
    withdraw(amount, inputPin) {
        const availableBalance = this.balance + this.#overdraftLimit;
        
        if (amount > availableBalance) {
            throw new Error(`Exceeds overdraft limit. Available: AED ${availableBalance}`);
        }
        
        // Use reflection to access private balance
        // In real implementation, would need different approach
        const newBalance = this.balance - amount - this.#transactionFee;
        
        // Custom withdrawal logic
        this.balance = newBalance;
        this._recordTransaction('DEBIT', amount, 'Current Account Withdrawal');
        this._recordTransaction('DEBIT', this.#transactionFee, 'Transaction Fee');
        
        if (this.balance < 0) {
            console.log(`Withdrew AED ${amount} (Fee: AED ${this.#transactionFee}). Using overdraft. Balance: AED ${this.balance}`);
        } else {
            console.log(`Withdrew AED ${amount} (Fee: AED ${this.#transactionFee}). Balance: AED ${this.balance}`);
        }
        
        return this.balance;
    }
    
    // New method
    getOverdraftInfo() {
        const overdraftUsed = Math.max(0, -this.balance);
        const overdraftAvailable = this.#overdraftLimit - overdraftUsed;
        
        return {
            limit: this.#overdraftLimit,
            used: overdraftUsed,
            available: overdraftAvailable,
            inOverdraft: this.balance < 0
        };
    }
    
    // Override toString
    toString() {
        const overdraftInfo = this.getOverdraftInfo();
        return `CurrentAccount(${this.accountNumber}, Balance: AED ${this.balance}, Overdraft: AED ${overdraftInfo.available})`;
    }
}

// FixedDepositAccount extends BankAccount
class FixedDepositAccount extends BankAccount {
    #maturityDate;
    #fixedRate;
    #locked;
    
    constructor(accountNumber, accountHolder, depositAmount, pin, tenureMonths, rate) {
        super(accountNumber, accountHolder, depositAmount, pin);
        this.#fixedRate = rate;
        this.#maturityDate = new Date();
        this.#maturityDate.setMonth(this.#maturityDate.getMonth() + tenureMonths);
        this.#locked = true;
        this.accountType = 'FIXED_DEPOSIT';
        this.tenureMonths = tenureMonths;
    }
    
    // Override withdraw - not allowed before maturity
    withdraw(amount, inputPin) {
        if (this.#locked) {
            const daysRemaining = Math.ceil((this.#maturityDate - new Date()) / (1000 * 60 * 60 * 24));
            throw new Error(`Fixed deposit is locked. Matures in ${daysRemaining} days on ${this.#maturityDate.toDateString()}`);
        }
        
        return super.withdraw(amount, inputPin);
    }
    
    // Override deposit - not allowed after creation
    deposit(amount) {
        throw new Error('Cannot add funds to fixed deposit account');
    }
    
    // Calculate maturity amount
    calculateMaturityAmount() {
        const years = this.tenureMonths / 12;
        const maturityAmount = this.balance * Math.pow(1 + this.#fixedRate, years);
        return maturityAmount;
    }
    
    // Check if matured
    isMatured() {
        return new Date() >= this.#maturityDate;
    }
    
    // Mature the deposit
    mature() {
        if (!this.isMatured()) {
            throw new Error('Deposit has not matured yet');
        }
        
        const maturityAmount = this.calculateMaturityAmount();
        const interest = maturityAmount - this.balance;
        
        this.balance = maturityAmount;
        this.#locked = false;
        
        this._recordTransaction('CREDIT', interest, 'Maturity Interest');
        
        console.log(`Fixed deposit matured. Interest earned: AED ${interest.toFixed(2)}`);
        console.log(`Total amount: AED ${maturityAmount.toFixed(2)}`);
        
        return maturityAmount;
    }
}

// Abstract-like class using new Error
class LoanAccount extends BankAccount {
    #loanAmount;
    #interestRate;
    #monthlyPayment;
    #remainingBalance;
    
    constructor(accountNumber, accountHolder, loanAmount, interestRate, tenureMonths, pin) {
        super(accountNumber, accountHolder, 0, pin);
        this.#loanAmount = loanAmount;
        this.#interestRate = interestRate;
        this.#remainingBalance = loanAmount;
        this.tenureMonths = tenureMonths;
        this.accountType = 'LOAN';
        
        // Calculate EMI
        const monthlyRate = interestRate / 12;
        this.#monthlyPayment = (loanAmount * monthlyRate * Math.pow(1 + monthlyRate, tenureMonths)) / 
                               (Math.pow(1 + monthlyRate, tenureMonths) - 1);
    }
    
    // Override deposit - this is payment towards loan
    deposit(amount) {
        if (amount > this.#remainingBalance) {
            amount = this.#remainingBalance;
        }
        
        this.#remainingBalance -= amount;
        this._recordTransaction('CREDIT', amount, 'Loan Payment');
        
        console.log(`Loan payment: AED ${amount}. Remaining: AED ${this.#remainingBalance.toFixed(2)}`);
        
        if (this.#remainingBalance === 0) {
            console.log('🎉 Loan fully paid!');
            this.status = 'CLOSED';
        }
        
        return this.#remainingBalance;
    }
    
    // Override withdraw - not allowed
    withdraw(amount, inputPin) {
        throw new Error('Cannot withdraw from loan account');
    }
    
    // Loan-specific methods
    getLoanDetails() {
        return {
            loanAmount: this.#loanAmount,
            interestRate: (this.#interestRate * 100).toFixed(2) + '%',
            monthlyPayment: this.#monthlyPayment.toFixed(2),
            remainingBalance: this.#remainingBalance.toFixed(2),
            tenureMonths: this.tenureMonths,
            status: this.status
        };
    }
    
    // Override getStatement
    getStatement() {
        console.log(`\n=== Loan Account Statement ===`);
        console.log(`Account: ${this.accountNumber}`);
        console.log(`Holder: ${this.accountHolder}`);
        console.log(`Loan Amount: AED ${this.#loanAmount.toFixed(2)}`);
        console.log(`Monthly Payment: AED ${this.#monthlyPayment.toFixed(2)}`);
        console.log(`Remaining Balance: AED ${this.#remainingBalance.toFixed(2)}`);
        console.log(`\nPayment History:`);
        
        this.transactions.slice(-5).forEach(txn => {
            console.log(`${txn.timestamp} | Payment: AED ${txn.amount}`);
        });
    }
}

// Usage Examples
console.log('=== Banking System with Classes ===\n');

// Create accounts
const savings = new SavingsAccount('ENBD001', 'Ahmed Al Mansouri', 10000, '1234', 0.03);
console.log('Created:', savings.toString());

savings.deposit(5000);
savings.withdraw(2000, '1234');
savings.applyMonthlyInterest();
console.log('');

const current = new CurrentAccount('ENBD002', 'Fatima Hassan', 5000, '5678', 10000);
console.log('Created:', current.toString());

current.deposit(3000);
current.withdraw(10000, '5678'); // Using overdraft
console.log('Overdraft Info:', current.getOverdraftInfo());
console.log('');

const fixedDeposit = new FixedDepositAccount('ENBD003', 'Omar Abdullah', 50000, '9012', 12, 0.05);
console.log('Fixed Deposit created');
console.log('Maturity Amount:', fixedDeposit.calculateMaturityAmount().toFixed(2));

try {
    fixedDeposit.withdraw(1000, '9012'); // Should fail
} catch (error) {
    console.log('Error:', error.message);
}
console.log('');

const loan = new LoanAccount('ENBD004', 'Khalid Mohammed', 100000, 0.08, 24, '3456');
console.log('Loan Details:', loan.getLoanDetails());

loan.deposit(5000); // Make payment
console.log('');

// Static methods
console.log('Bank Info:', BankAccount.getBankInfo());
console.log('Total Accounts:', BankAccount.getTotalAccounts());

// Inheritance check
console.log('\n=== Inheritance Check ===');
console.log('savings instanceof SavingsAccount:', savings instanceof SavingsAccount);
console.log('savings instanceof BankAccount:', savings instanceof BankAccount);
console.log('current instanceof CurrentAccount:', current instanceof CurrentAccount);
console.log('current instanceof BankAccount:', current instanceof BankAccount);

// Try accessing private fields
console.log('\n=== Private Fields ===');
console.log('Public accountNumber:', savings.accountNumber); // Masked
console.log('Public balance:', savings.balance);
// console.log(savings.#accountNumber); // SyntaxError
// console.log(savings.#balance); // SyntaxError
```

---

## Q11: Explain modules (import/export) with banking service architecture

**Answer:**

ES6 modules allow code organization and reusability using import/export statements.

**Banking Service Modular Architecture:**

```javascript
// ============================================
// File: utils/constants.js
// ============================================
export const BANK_NAME = 'Emirates NBD';
export const BANK_CODE = 'EBILAEAD';
export const SWIFT_CODE = 'EBILAEAD';

export const ACCOUNT_TYPES = {
    SAVINGS: 'SAVINGS',
    CURRENT: 'CURRENT',
    FIXED_DEPOSIT: 'FIXED_DEPOSIT',
    LOAN: 'LOAN'
};

export const TRANSACTION_TYPES = {
    CREDIT: 'CREDIT',
    DEBIT: 'DEBIT',
    TRANSFER: 'TRANSFER'
};

export const ACCOUNT_STATUS = {
    ACTIVE: 'ACTIVE',
    FROZEN: 'FROZEN',
    CLOSED: 'CLOSED',
    DORMANT: 'DORMANT'
};

// Default export
export default {
    BANK_NAME,
    BANK_CODE,
    SWIFT_CODE,
    ACCOUNT_TYPES,
    TRANSACTION_TYPES,
    ACCOUNT_STATUS
};

// ============================================
// File: utils/validators.js
// ============================================
export function validateAccountNumber(accountNumber) {
    const pattern = /^ENBD\d{9}$/;
    return pattern.test(accountNumber);
}

export function validateAmount(amount) {
    return typeof amount === 'number' && amount > 0 && !isNaN(amount);
}

export function validatePin(pin) {
    return /^\d{4}$/.test(pin);
}

export function validateEmail(email) {
    const pattern = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return pattern.test(email);
}

export function validatePhoneNumber(phone) {
    const pattern = /^\+971[0-9]{9}$/;
    return pattern.test(phone);
}

// Named export with alias
export { validateAccountNumber as isValidAccount };

// ============================================
// File: utils/formatters.js
// ============================================
export const formatCurrency = (amount, currency = 'AED') => {
    return `${currency} ${amount.toLocaleString('en-AE', {
        minimumFractionDigits: 2,
        maximumFractionDigits: 2
    })}`;
};

export const formatDate = (date) => {
    return new Date(date).toLocaleDateString('en-AE', {
        year: 'numeric',
        month: 'long',
        day: 'numeric'
    });
};

export const formatAccountNumber = (accountNumber) => {
    return accountNumber.replace(/(\d{4})(\d+)(\d{4})/, '$1****$3');
};

export const formatTransactionId = () => {
    return `TXN${Date.now()}${Math.random().toString(36).substr(2, 5).toUpperCase()}`;
};

// ============================================
// File: models/Account.js
// ============================================
import { ACCOUNT_TYPES, ACCOUNT_STATUS, TRANSACTION_TYPES } from '../utils/constants.js';
import { validateAmount } from '../utils/validators.js';
import { formatCurrency } from '../utils/formatters.js';

export class Account {
    #balance;
    #pin;
    
    constructor(accountNumber, accountHolder, initialBalance, pin) {
        this.accountNumber = accountNumber;
        this.accountHolder = accountHolder;
        this.#balance = initialBalance;
        this.#pin = pin;
        this.status = ACCOUNT_STATUS.ACTIVE;
        this.transactions = [];
        this.createdAt = new Date();
    }
    
    get balance() {
        return this.#balance;
    }
    
    deposit(amount) {
        if (!validateAmount(amount)) {
            throw new Error('Invalid amount');
        }
        
        this.#balance += amount;
        this._recordTransaction(TRANSACTION_TYPES.CREDIT, amount, 'Deposit');
        
        console.log(`Deposited ${formatCurrency(amount)}`);
        return this.#balance;
    }
    
    withdraw(amount, pin) {
        if (!this.#verifyPin(pin)) {
            throw new Error('Invalid PIN');
        }
        
        if (!validateAmount(amount)) {
            throw new Error('Invalid amount');
        }
        
        if (amount > this.#balance) {
            throw new Error('Insufficient funds');
        }
        
        this.#balance -= amount;
        this._recordTransaction(TRANSACTION_TYPES.DEBIT, amount, 'Withdrawal');
        
        console.log(`Withdrew ${formatCurrency(amount)}`);
        return this.#balance;
    }
    
    #verifyPin(pin) {
        return this.#pin === pin;
    }
    
    _recordTransaction(type, amount, description) {
        this.transactions.push({
            id: `TXN${Date.now()}`,
            type,
            amount,
            description,
            balance: this.#balance,
            timestamp: new Date()
        });
    }
}

// ============================================
// File: services/TransactionService.js
// ============================================
import { TRANSACTION_TYPES } from '../utils/constants.js';
import { validateAmount } from '../utils/validators.js';
import { formatCurrency, formatTransactionId } from '../utils/formatters.js';

export class TransactionService {
    constructor() {
        this.transactions = new Map();
    }
    
    async transfer(fromAccount, toAccount, amount, pin) {
        if (!validateAmount(amount)) {
            throw new Error('Invalid amount');
        }
        
        const transactionId = formatTransactionId();
        
        console.log(`\nInitiating transfer ${transactionId}`);
        console.log(`From: ${fromAccount.accountNumber}`);
        console.log(`To: ${toAccount.accountNumber}`);
        console.log(`Amount: ${formatCurrency(amount)}`);
        
        try {
            // Debit from source
            fromAccount.withdraw(amount, pin);
            
            // Credit to destination
            toAccount.deposit(amount);
            
            // Record transaction
            this.transactions.set(transactionId, {
                id: transactionId,
                type: TRANSACTION_TYPES.TRANSFER,
                fromAccount: fromAccount.accountNumber,
                toAccount: toAccount.accountNumber,
                amount,
                status: 'COMPLETED',
                timestamp: new Date()
            });
            
            console.log(`Transfer ${transactionId} completed successfully`);
            
            return {
                success: true,
                transactionId,
                amount: formatCurrency(amount)
            };
            
        } catch (error) {
            console.error(`Transfer ${transactionId} failed:`, error.message);
            
            this.transactions.set(transactionId, {
                id: transactionId,
                type: TRANSACTION_TYPES.TRANSFER,
                fromAccount: fromAccount.accountNumber,
                toAccount: toAccount.accountNumber,
                amount,
                status: 'FAILED',
                error: error.message,
                timestamp: new Date()
            });
            
            throw error;
        }
    }
    
    getTransaction(transactionId) {
        return this.transactions.get(transactionId);
    }
    
    getAllTransactions() {
        return Array.from(this.transactions.values());
    }
}

// Default export
export default TransactionService;

// ============================================
// File: services/AccountService.js
// ============================================
import { Account } from '../models/Account.js';
import { validateAccountNumber, validatePin } from '../utils/validators.js';
import { ACCOUNT_STATUS } from '../utils/constants.js';

export class AccountService {
    constructor() {
        this.accounts = new Map();
    }
    
    createAccount(accountNumber, accountHolder, initialBalance, pin) {
        if (!validateAccountNumber(accountNumber)) {
            throw new Error('Invalid account number format');
        }
        
        if (!validatePin(pin)) {
            throw new Error('PIN must be 4 digits');
        }
        
        if (this.accounts.has(accountNumber)) {
            throw new Error('Account already exists');
        }
        
        const account = new Account(accountNumber, accountHolder, initialBalance, pin);
        this.accounts.set(accountNumber, account);
        
        console.log(`Account created: ${accountNumber} for ${accountHolder}`);
        
        return account;
    }
    
    getAccount(accountNumber) {
        const account = this.accounts.get(accountNumber);
        
        if (!account) {
            throw new Error('Account not found');
        }
        
        return account;
    }
    
    freezeAccount(accountNumber) {
        const account = this.getAccount(accountNumber);
        account.status = ACCOUNT_STATUS.FROZEN;
        console.log(`Account ${accountNumber} frozen`);
    }
    
    unfreezeAccount(accountNumber) {
        const account = this.getAccount(accountNumber);
        account.status = ACCOUNT_STATUS.ACTIVE;
        console.log(`Account ${accountNumber} unfrozen`);
    }
    
    closeAccount(accountNumber) {
        const account = this.getAccount(accountNumber);
        
        if (account.balance > 0) {
            throw new Error('Cannot close account with positive balance');
        }
        
        account.status = ACCOUNT_STATUS.CLOSED;
        console.log(`Account ${accountNumber} closed`);
    }
}

export default AccountService;

// ============================================
// File: services/ReportService.js
// ============================================
import { formatCurrency, formatDate } from '../utils/formatters.js';
import { TRANSACTION_TYPES } from '../utils/constants.js';

export function generateAccountStatement(account) {
    console.log(`\n${'='.repeat(60)}`);
    console.log(`ACCOUNT STATEMENT`);
    console.log(`${'='.repeat(60)}`);
    console.log(`Account Number: ${account.accountNumber}`);
    console.log(`Account Holder: ${account.accountHolder}`);
    console.log(`Current Balance: ${formatCurrency(account.balance)}`);
    console.log(`Account Status: ${account.status}`);
    console.log(`${'='.repeat(60)}`);
    console.log(`\nRECENT TRANSACTIONS:\n`);
    
    account.transactions.slice(-10).forEach(txn => {
        const sign = txn.type === TRANSACTION_TYPES.CREDIT ? '+' : '-';
        console.log(`${formatDate(txn.timestamp)} | ${txn.type.padEnd(10)} | ${sign}${formatCurrency(txn.amount).padEnd(15)} | ${txn.description}`);
    });
    
    console.log(`\n${'='.repeat(60)}\n`);
}

export function generateTransactionReport(transactions) {
    const totalCredits = transactions
        .filter(t => t.type === TRANSACTION_TYPES.CREDIT)
        .reduce((sum, t) => sum + t.amount, 0);
    
    const totalDebits = transactions
        .filter(t => t.type === TRANSACTION_TYPES.DEBIT)
        .reduce((sum, t) => sum + t.amount, 0);
    
    console.log(`\n${'='.repeat(60)}`);
    console.log(`TRANSACTION REPORT`);
    console.log(`${'='.repeat(60)}`);
    console.log(`Total Transactions: ${transactions.length}`);
    console.log(`Total Credits: ${formatCurrency(totalCredits)}`);
    console.log(`Total Debits: ${formatCurrency(totalDebits)}`);
    console.log(`Net Flow: ${formatCurrency(totalCredits - totalDebits)}`);
    console.log(`${'='.repeat(60)}\n`);
}

// ============================================
// File: index.js (Main Application)
// ============================================
// Named imports
import { AccountService } from './services/AccountService.js';
import { TransactionService } from './services/TransactionService.js';
import { generateAccountStatement, generateTransactionReport } from './services/ReportService.js';

// Default import
import constants from './utils/constants.js';

// Import with alias
import { formatCurrency as currency } from './utils/formatters.js';

// Import everything as namespace
import * as validators from './utils/validators.js';

console.log('=== Banking Application Started ===');
console.log(`Bank: ${constants.BANK_NAME} (${constants.BANK_CODE})\n`);

// Initialize services
const accountService = new AccountService();
const transactionService = new TransactionService();

// Create accounts
const account1 = accountService.createAccount(
    'ENBD000000001',
    'Ahmed Al Mansouri',
    10000,
    '1234'
);

const account2 = accountService.createAccount(
    'ENBD000000002',
    'Fatima Hassan',
    5000,
    '5678'
);

// Perform operations
account1.deposit(3000);
account1.withdraw(1000, '1234');

account2.deposit(2000);

// Transfer funds
transactionService.transfer(account1, account2, 2500, '1234');

// Generate reports
generateAccountStatement(account1);
generateAccountStatement(account2);
generateTransactionReport([...account1.transactions, ...account2.transactions]);

// Validate data
console.log('=== Validations ===');
console.log('Valid account:', validators.validateAccountNumber('ENBD000000001'));
console.log('Valid email:', validators.validateEmail('test@enbd.com'));
console.log('Valid phone:', validators.validatePhoneNumber('+971501234567'));

console.log('\n=== Banking Application Ended ===');
```

---

## Q12: Explain error handling patterns and custom errors in banking transactions

**Answer:**

Proper error handling is critical in banking systems to ensure data integrity and user experience.

**Banking Error Handling System:**

```javascript
// Custom Error Classes
class BankingError extends Error {
    constructor(message, code, statusCode = 500) {
        super(message);
        this.name = this.constructor.name;
        this.code = code;
        this.statusCode = statusCode;
        this.timestamp = new Date().toISOString();
        Error.captureStackTrace(this, this.constructor);
    }
    
    toJSON() {
        return {
            name: this.name,
            message: this.message,
            code: this.code,
            statusCode: this.statusCode,
            timestamp: this.timestamp
        };
    }
}

class InsufficientFundsError extends BankingError {
    constructor(accountNumber, requested, available) {
        super(
            `Insufficient funds in account ${accountNumber}. Requested: AED ${requested}, Available: AED ${available}`,
            'INSUFFICIENT_FUNDS',
            400
        );
        this.accountNumber = accountNumber;
        this.requested = requested;
        this.available = available;
    }
}

class InvalidPinError extends BankingError {
    constructor(accountNumber, attempts = 1) {
        super(
            `Invalid PIN for account ${accountNumber}`,
            'INVALID_PIN',
            401
        );
        this.accountNumber = accountNumber;
        this.attempts = attempts;
        this.remainingAttempts = 3 - attempts;
    }
}

class AccountFrozenError extends BankingError {
    constructor(accountNumber, reason) {
        super(
            `Account ${accountNumber} is frozen: ${reason}`,
            'ACCOUNT_FROZEN',
            403
        );
        this.accountNumber = accountNumber;
        this.reason = reason;
    }
}

class DailyLimitExceededError extends BankingError {
    constructor(accountNumber, limit, attempted) {
        super(
            `Daily transaction limit exceeded for account ${accountNumber}. Limit: AED ${limit}, Attempted: AED ${attempted}`,
            'DAILY_LIMIT_EXCEEDED',
            400
        );
        this.accountNumber = accountNumber;
        this.limit = limit;
        this.attempted = attempted;
    }
}

class TransactionTimeoutError extends BankingError {
    constructor(transactionId, duration) {
        super(
            `Transaction ${transactionId} timed out after ${duration}ms`,
            'TRANSACTION_TIMEOUT',
            408
        );
        this.transactionId = transactionId;
        this.duration = duration;
    }
}

class ValidationError extends BankingError {
    constructor(field, value, constraint) {
        super(
            `Validation failed for ${field}: ${constraint}`,
            'VALIDATION_ERROR',
            400
        );
        this.field = field;
        this.value = value;
        this.constraint = constraint;
    }
}

// Banking Account with Error Handling
class SecureBankAccount {
    #accountNumber;
    #pin;
    #balance;
    #dailyLimit;
    #dailyTransactions;
    #pinAttempts;
    #status;
    
    static MAX_PIN_ATTEMPTS = 3;
    static DEFAULT_DAILY_LIMIT = 50000;
    
    constructor(accountNumber, accountHolder, initialBalance, pin) {
        this.#accountNumber = accountNumber;
        this.accountHolder = accountHolder;
        this.#balance = initialBalance;
        this.#pin = pin;
        this.#dailyLimit = SecureBankAccount.DEFAULT_DAILY_LIMIT;
        this.#dailyTransactions = 0;
        this.#pinAttempts = 0;
        this.#status = 'ACTIVE';
        this.transactions = [];
        
        this._resetDailyLimitAt = new Date();
        this._resetDailyLimitAt.setHours(24, 0, 0, 0);
    }
    
    get accountNumber() {
        return this.#accountNumber;
    }
    
    get balance() {
        return this.#balance;
    }
    
    get status() {
        return this.#status;
    }
    
    deposit(amount) {
        try {
            // Validate input
            this._validateAmount(amount);
            this._checkAccountStatus();
            
            // Process deposit
            this.#balance += amount;
            this._recordTransaction('CREDIT', amount, 'Deposit');
            
            console.log(`✓ Deposited AED ${amount}. New balance: AED ${this.#balance}`);
            
            return {
                success: true,
                balance: this.#balance,
                transactionId: this.transactions[this.transactions.length - 1].id
            };
            
        } catch (error) {
            this._handleError(error, 'DEPOSIT');
            throw error;
        }
    }
    
    withdraw(amount, pin) {
        try {
            // Validate PIN
            this._verifyPin(pin);
            
            // Validate input
            this._validateAmount(amount);
            this._checkAccountStatus();
            
            // Check balance
            if (amount > this.#balance) {
                throw new InsufficientFundsError(
                    this.#accountNumber,
                    amount,
                    this.#balance
                );
            }
            
            // Check daily limit
            this._checkDailyLimit(amount);
            
            // Process withdrawal
            this.#balance -= amount;
            this.#dailyTransactions += amount;
            this._recordTransaction('DEBIT', amount, 'Withdrawal');
            
            // Reset PIN attempts on successful transaction
            this.#pinAttempts = 0;
            
            console.log(`✓ Withdrew AED ${amount}. New balance: AED ${this.#balance}`);
            
            return {
                success: true,
                balance: this.#balance,
                dailyRemaining: this.#dailyLimit - this.#dailyTransactions,
                transactionId: this.transactions[this.transactions.length - 1].id
            };
            
        } catch (error) {
            this._handleError(error, 'WITHDRAWAL');
            throw error;
        }
    }
    
    transfer(targetAccount, amount, pin) {
        try {
            // Verify PIN
            this._verifyPin(pin);
            
            // Validate
            this._validateAmount(amount);
            this._checkAccountStatus();
            
            if (amount > this.#balance) {
                throw new InsufficientFundsError(
                    this.#accountNumber,
                    amount,
                    this.#balance
                );
            }
            
            this._checkDailyLimit(amount);
            
            // Execute transfer
            this.#balance -= amount;
            this.#dailyTransactions += amount;
            targetAccount.deposit(amount);
            
            this._recordTransaction('TRANSFER_OUT', amount, `Transfer to ${targetAccount.accountNumber}`);
            
            // Reset PIN attempts
            this.#pinAttempts = 0;
            
            console.log(`✓ Transferred AED ${amount} to ${targetAccount.accountNumber}`);
            
            return {
                success: true,
                balance: this.#balance,
                transactionId: this.transactions[this.transactions.length - 1].id
            };
            
        } catch (error) {
            this._handleError(error, 'TRANSFER');
            throw error;
        }
    }
    
    // Private validation methods
    _validateAmount(amount) {
        if (typeof amount !== 'number' || amount <= 0) {
            throw new ValidationError('amount', amount, 'Must be a positive number');
        }
        
        if (!Number.isFinite(amount)) {
            throw new ValidationError('amount', amount, 'Must be a finite number');
        }
        
        if (amount > 1000000) {
            throw new ValidationError('amount', amount, 'Exceeds maximum transaction amount');
        }
    }
    
    _checkAccountStatus() {
        if (this.#status === 'FROZEN') {
            throw new AccountFrozenError(this.#accountNumber, 'Account frozen by bank');
        }
        
        if (this.#status === 'CLOSED') {
            throw new BankingError(
                `Account ${this.#accountNumber} is closed`,
                'ACCOUNT_CLOSED',
                403
            );
        }
    }
    
    _verifyPin(pin) {
        if (this.#pinAttempts >= SecureBankAccount.MAX_PIN_ATTEMPTS) {
            this.#status = 'FROZEN';
            throw new AccountFrozenError(
                this.#accountNumber,
                'Maximum PIN attempts exceeded'
            );
        }
        
        if (pin !== this.#pin) {
            this.#pinAttempts++;
            throw new InvalidPinError(this.#accountNumber, this.#pinAttempts);
        }
    }
    
    _checkDailyLimit(amount) {
        // Reset if new day
        if (new Date() >= this._resetDailyLimitAt) {
            this.#dailyTransactions = 0;
            this._resetDailyLimitAt = new Date();
            this._resetDailyLimitAt.setHours(24, 0, 0, 0);
        }
        
        if (this.#dailyTransactions + amount > this.#dailyLimit) {
            throw new DailyLimitExceededError(
                this.#accountNumber,
                this.#dailyLimit,
                this.#dailyTransactions + amount
            );
        }
    }
    
    _recordTransaction(type, amount, description) {
        this.transactions.push({
            id: `TXN${Date.now()}${Math.random().toString(36).substr(2, 5)}`,
            type,
            amount,
            description,
            balance: this.#balance,
            timestamp: new Date().toISOString()
        });
    }
    
    _handleError(error, operation) {
        console.error(`✗ ${operation} failed:`, error.message);
        
        // Log error for monitoring
        this._logError({
            operation,
            error: error.toJSON ? error.toJSON() : {
                name: error.name,
                message: error.message
            },
            accountNumber: this.#accountNumber,
            timestamp: new Date().toISOString()
        });
    }
    
    _logError(errorDetails) {
        // In production, send to logging service
        console.error('[ERROR LOG]', JSON.stringify(errorDetails, null, 2));
    }
}

// Transaction Service with Error Handling
class TransactionService {
    static TIMEOUT_MS = 5000;
    
    async processTransaction(operation, timeout = TransactionService.TIMEOUT_MS) {
        const transactionId = `TXN${Date.now()}`;
        const startTime = Date.now();
        
        try {
            // Create timeout promise
            const timeoutPromise = new Promise((_, reject) => {
                setTimeout(() => {
                    reject(new TransactionTimeoutError(
                        transactionId,
                        Date.now() - startTime
                    ));
                }, timeout);
            });
            
            // Race between operation and timeout
            const result = await Promise.race([
                operation(),
                timeoutPromise
            ]);
            
            console.log(`✓ Transaction ${transactionId} completed in ${Date.now() - startTime}ms`);
            
            return result;
            
        } catch (error) {
            if (error instanceof TransactionTimeoutError) {
                console.error(`✗ Transaction ${transactionId} timed out`);
            } else {
                console.error(`✗ Transaction ${transactionId} failed:`, error.message);
            }
            
            throw error;
        }
    }
    
    async transferWithRetry(fromAccount, toAccount, amount, pin, maxRetries = 3) {
        let lastError;
        
        for (let attempt = 1; attempt <= maxRetries; attempt++) {
            try {
                console.log(`\nTransfer attempt ${attempt}/${maxRetries}`);
                
                const result = await this.processTransaction(async () => {
                    return fromAccount.transfer(toAccount, amount, pin);
                });
                
                return result;
                
            } catch (error) {
                lastError = error;
                
                // Don't retry on certain errors
                if (error instanceof InvalidPinError ||
                    error instanceof InsufficientFundsError ||
                    error instanceof AccountFrozenError) {
                    console.error('Non-retryable error, aborting');
                    throw error;
                }
                
                if (attempt < maxRetries) {
                    const delay = Math.pow(2, attempt) * 1000; // Exponential backoff
                    console.log(`Retrying in ${delay}ms...`);
                    await new Promise(resolve => setTimeout(resolve, delay));
                }
            }
        }
        
        throw new BankingError(
            `Transfer failed after ${maxRetries} attempts: ${lastError.message}`,
            'TRANSFER_FAILED',
            500
        );
    }
}

// Usage Examples
console.log('=== Banking Error Handling Examples ===\n');

const account1 = new SecureBankAccount('ACC001', 'Ahmed', 10000, '1234');
const account2 = new SecureBankAccount('ACC002', 'Fatima', 5000, '5678');
const transactionService = new TransactionService();

// Example 1: Successful transaction
try {
    account1.deposit(2000);
    account1.withdraw(1000, '1234');
} catch (error) {
    console.error('Error:', error.message);
}

// Example 2: Invalid PIN
try {
    console.log('\n--- Testing Invalid PIN ---');
    account1.withdraw(500, '9999');
} catch (error) {
    console.error(`Caught ${error.name}:`, error.message);
    console.error('Remaining attempts:', error.remainingAttempts);
}

// Example 3: Insufficient Funds
try {
    console.log('\n--- Testing Insufficient Funds ---');
    account1.withdraw(50000, '1234');
} catch (error) {
    console.error(`Caught ${error.name}:`, error.message);
    console.error('Available:', error.available);
}

// Example 4: Daily Limit
try {
    console.log('\n--- Testing Daily Limit ---');
    account1.withdraw(45000, '1234');
    account1.withdraw(10000, '1234'); // Exceeds limit
} catch (error) {
    console.error(`Caught ${error.name}:`, error.message);
}

// Example 5: Transfer with retry
(async () => {
    try {
        console.log('\n--- Testing Transfer with Retry ---');
        const result = await transactionService.transferWithRetry(
            account1,
            account2,
            2000,
            '1234'
        );
        console.log('Transfer successful:', result);
    } catch (error) {
        console.error('Transfer finally failed:', error.message);
    }
})();
```

---
