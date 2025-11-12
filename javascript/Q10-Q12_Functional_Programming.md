# JavaScript Interview Questions (Q10-Q12): Functional Programming

## Q10: Explain higher-order functions and pure functions with banking examples

**Answer:**

Higher-order functions take functions as arguments or return functions. Pure functions always return the same output for the same input without side effects.

**Banking Transaction Processing with Pure Functions:**

```javascript
// Pure Functions - No side effects, same input = same output
class PureFunctionExamples {
    
    // Pure function: Calculate interest
    static calculateInterest(principal, rate, years) {
        return principal * rate * years;
    }
    
    // Pure function: Calculate compound interest
    static calculateCompoundInterest(principal, rate, years, frequency = 12) {
        return principal * Math.pow((1 + rate / frequency), frequency * years);
    }
    
    // Pure function: Apply transaction fee
    static applyTransactionFee(amount, feeRate = 0.01) {
        return {
            amount,
            fee: amount * feeRate,
            total: amount + (amount * feeRate)
        };
    }
    
    // Pure function: Validate account balance
    static hasMinimumBalance(balance, minimum) {
        return balance >= minimum;
    }
    
    // Pure function: Calculate monthly payment
    static calculateMonthlyPayment(principal, annualRate, months) {
        const monthlyRate = annualRate / 12;
        return (principal * monthlyRate * Math.pow(1 + monthlyRate, months)) / 
               (Math.pow(1 + monthlyRate, months) - 1);
    }
}

console.log('=== Pure Functions ===\n');

// Pure functions always return same result
console.log('Interest:', PureFunctionExamples.calculateInterest(10000, 0.05, 2));
console.log('Interest:', PureFunctionExamples.calculateInterest(10000, 0.05, 2)); // Same result

console.log('\nCompound Interest:', PureFunctionExamples.calculateCompoundInterest(10000, 0.05, 2));

console.log('\nTransaction Fee:', PureFunctionExamples.applyTransactionFee(5000));

// Higher-Order Functions
console.log('\n=== Higher-Order Functions ===\n');

class BankingHigherOrderFunctions {
    
    // HOF: Takes a function as parameter
    static processTransactions(transactions, processFn) {
        return transactions.map(processFn);
    }
    
    // HOF: Returns a function
    static createTransactionProcessor(feeRate) {
        return (transaction) => ({
            ...transaction,
            fee: transaction.amount * feeRate,
            netAmount: transaction.amount - (transaction.amount * feeRate)
        });
    }
    
    // HOF: Filtering with custom predicate
    static filterTransactions(transactions, predicateFn) {
        return transactions.filter(predicateFn);
    }
    
    // HOF: Aggregating with custom reducer
    static aggregateTransactions(transactions, reducerFn, initialValue) {
        return transactions.reduce(reducerFn, initialValue);
    }
    
    // HOF: Creating validators
    static createValidator(rules) {
        return (transaction) => {
            const errors = [];
            
            if (rules.minAmount && transaction.amount < rules.minAmount) {
                errors.push(`Amount must be at least ${rules.minAmount}`);
            }
            
            if (rules.maxAmount && transaction.amount > rules.maxAmount) {
                errors.push(`Amount cannot exceed ${rules.maxAmount}`);
            }
            
            if (rules.requiredFields) {
                rules.requiredFields.forEach(field => {
                    if (!transaction[field]) {
                        errors.push(`${field} is required`);
                    }
                });
            }
            
            return {
                valid: errors.length === 0,
                errors
            };
        };
    }
}

// Sample transactions
const transactions = [
    { id: 1, accountId: 'ACC001', amount: 5000, type: 'CREDIT', date: '2024-01-15' },
    { id: 2, accountId: 'ACC001', amount: 1200, type: 'DEBIT', date: '2024-01-16' },
    { id: 3, accountId: 'ACC002', amount: 3000, type: 'CREDIT', date: '2024-01-17' },
    { id: 4, accountId: 'ACC001', amount: 500, type: 'DEBIT', date: '2024-01-18' }
];

// Using HOF to create a transaction processor
const addFee = BankingHigherOrderFunctions.createTransactionProcessor(0.02);
const processedTransactions = BankingHigherOrderFunctions.processTransactions(transactions, addFee);
console.log('Processed with fee:', processedTransactions[0]);

// Using HOF with custom filter
const largeTransactions = BankingHigherOrderFunctions.filterTransactions(
    transactions,
    txn => txn.amount > 2000
);
console.log('\nLarge transactions:', largeTransactions.length);

// Using HOF with custom reducer
const totalAmount = BankingHigherOrderFunctions.aggregateTransactions(
    transactions,
    (sum, txn) => sum + txn.amount,
    0
);
console.log('Total amount:', totalAmount);

// Create and use validator
const validator = BankingHigherOrderFunctions.createValidator({
    minAmount: 100,
    maxAmount: 100000,
    requiredFields: ['accountId', 'amount', 'type']
});

const validationResult = validator({ accountId: 'ACC001', amount: 50, type: 'DEBIT' });
console.log('\nValidation:', validationResult);

// Composition and Currying
console.log('\n=== Function Composition ===\n');

class FunctionalBanking {
    
    // Currying: Breaking function into smaller functions
    static curry(fn) {
        return function curried(...args) {
            if (args.length >= fn.length) {
                return fn.apply(this, args);
            }
            return function(...nextArgs) {
                return curried.apply(this, args.concat(nextArgs));
            };
        };
    }
    
    // Curried transaction fee calculator
    static calculateFee = this.curry((baseRate, urgencyMultiplier, amount) => {
        return amount * baseRate * urgencyMultiplier;
    });
    
    // Function composition
    static compose(...fns) {
        return (initialValue) => 
            fns.reduceRight((value, fn) => fn(value), initialValue);
    }
    
    // Pipe (left to right composition)
    static pipe(...fns) {
        return (initialValue) =>
            fns.reduce((value, fn) => fn(value), initialValue);
    }
}

// Currying example
const standardFeeCalculator = FunctionalBanking.calculateFee(0.02);
const urgentFeeCalculator = standardFeeCalculator(2);
console.log('Standard fee for 5000:', urgentFeeCalculator(5000));

const normalFeeCalculator = standardFeeCalculator(1);
console.log('Normal fee for 5000:', normalFeeCalculator(5000));

// Composition example
const addTax = (amount) => amount * 1.05;
const addServiceFee = (amount) => amount + 10;
const roundAmount = (amount) => Math.round(amount * 100) / 100;

const calculateFinalAmount = FunctionalBanking.pipe(
    addTax,
    addServiceFee,
    roundAmount
);

console.log('\nFinal amount for 1000:', calculateFinalAmount(1000));

// Real-world Banking Application
console.log('\n=== Real-world Application ===\n');

class TransactionPipeline {
    
    // Individual pure transformation functions
    static validateAmount(transaction) {
        if (transaction.amount <= 0) {
            throw new Error('Amount must be positive');
        }
        return transaction;
    }
    
    static applyFee(transaction) {
        const fee = transaction.amount * 0.01;
        return {
            ...transaction,
            fee,
            totalAmount: transaction.amount + fee
        };
    }
    
    static addTimestamp(transaction) {
        return {
            ...transaction,
            timestamp: new Date().toISOString()
        };
    }
    
    static convertCurrency(fromRate, toRate) {
        return (transaction) => ({
            ...transaction,
            originalAmount: transaction.amount,
            amount: (transaction.amount / fromRate) * toRate,
            convertedFrom: 'USD',
            convertedTo: 'AED'
        });
    }
    
    static categorize(transaction) {
        const categories = {
            SALARY: transaction.amount > 5000,
            RETAIL: transaction.amount < 1000,
            UTILITY: transaction.amount >= 1000 && transaction.amount <= 2000
        };
        
        const category = Object.keys(categories).find(key => categories[key]) || 'OTHER';
        
        return {
            ...transaction,
            category
        };
    }
    
    static logTransaction(transaction) {
        console.log(`Transaction ${transaction.id}: ${transaction.type} AED ${transaction.amount}`);
        return transaction;
    }
    
    // Build processing pipeline
    static buildPipeline(...processors) {
        return (transaction) => {
            return processors.reduce((txn, processor) => processor(txn), transaction);
        };
    }
}

// Create transaction processing pipeline
const processTransaction = TransactionPipeline.buildPipeline(
    TransactionPipeline.validateAmount,
    TransactionPipeline.addTimestamp,
    TransactionPipeline.applyFee,
    TransactionPipeline.convertCurrency(1, 3.67), // USD to AED
    TransactionPipeline.categorize,
    TransactionPipeline.logTransaction
);

// Process a transaction through pipeline
const inputTransaction = {
    id: 'TXN001',
    accountId: 'ACC001',
    amount: 1500,
    type: 'CREDIT'
};

const processedTransaction = processTransaction(inputTransaction);
console.log('\nProcessed transaction:', processedTransaction);

// Memoization for expensive calculations
console.log('\n=== Memoization ===\n');

class MemoizedFunctions {
    
    static memoize(fn) {
        const cache = new Map();
        
        return (...args) => {
            const key = JSON.stringify(args);
            
            if (cache.has(key)) {
                console.log('Cache hit!');
                return cache.get(key);
            }
            
            console.log('Computing...');
            const result = fn(...args);
            cache.set(key, result);
            return result;
        };
    }
    
    // Expensive calculation
    static calculateLoanAmortization(principal, annualRate, years) {
        const monthlyRate = annualRate / 12;
        const months = years * 12;
        const monthlyPayment = 
            (principal * monthlyRate * Math.pow(1 + monthlyRate, months)) /
            (Math.pow(1 + monthlyRate, months) - 1);
        
        const schedule = [];
        let balance = principal;
        
        for (let month = 1; month <= months; month++) {
            const interestPayment = balance * monthlyRate;
            const principalPayment = monthlyPayment - interestPayment;
            balance -= principalPayment;
            
            schedule.push({
                month,
                payment: monthlyPayment,
                principal: principalPayment,
                interest: interestPayment,
                balance: Math.max(0, balance)
            });
        }
        
        return schedule;
    }
}

// Create memoized version
const memoizedAmortization = MemoizedFunctions.memoize(
    MemoizedFunctions.calculateLoanAmortization
);

// First call - computes
console.log('First call:');
const result1 = memoizedAmortization(100000, 0.05, 5);
console.log('Schedule length:', result1.length);

// Second call - cached
console.log('\nSecond call (same params):');
const result2 = memoizedAmortization(100000, 0.05, 5);
console.log('Schedule length:', result2.length);

// Third call - different params, computes again
console.log('\nThird call (different params):');
const result3 = memoizedAmortization(200000, 0.04, 10);
console.log('Schedule length:', result3.length);
```

---

## Q11: Explain closures for data privacy and module patterns in banking systems

**Answer:**

Closures enable data privacy by creating private variables that can only be accessed through specific functions.

**Banking Module Pattern:**

```javascript
// IIFE Module Pattern for Banking System
const BankingModule = (function() {
    // Private variables
    const accounts = new Map();
    const transactionLog = [];
    let nextAccountNumber = 1000;
    
    // Private functions
    function generateAccountNumber() {
        return `ENBD${String(nextAccountNumber++).padStart(8, '0')}`;
    }
    
    function validateAmount(amount) {
        if (typeof amount !== 'number' || amount <= 0) {
            throw new Error('Invalid amount');
        }
        return true;
    }
    
    function logTransaction(accountNumber, type, amount, balance) {
        transactionLog.push({
            timestamp: new Date().toISOString(),
            accountNumber,
            type,
            amount,
            balance
        });
    }
    
    function getAccount(accountNumber) {
        const account = accounts.get(accountNumber);
        if (!account) {
            throw new Error('Account not found');
        }
        return account;
    }
    
    // Public API
    return {
        // Create new account
        createAccount: function(accountHolder, initialBalance = 0) {
            validateAmount(initialBalance + 1);
            
            const accountNumber = generateAccountNumber();
            const account = {
                accountNumber,
                accountHolder,
                balance: initialBalance,
                createdAt: new Date().toISOString(),
                status: 'ACTIVE'
            };
            
            accounts.set(accountNumber, account);
            logTransaction(accountNumber, 'ACCOUNT_CREATED', initialBalance, initialBalance);
            
            console.log(`Account created: ${accountNumber} for ${accountHolder}`);
            return accountNumber;
        },
        
        // Get balance (public access to private data)
        getBalance: function(accountNumber) {
            const account = getAccount(accountNumber);
            return {
                accountNumber: account.accountNumber,
                balance: account.balance,
                currency: 'AED'
            };
        },
        
        // Deposit money
        deposit: function(accountNumber, amount) {
            validateAmount(amount);
            const account = getAccount(accountNumber);
            
            account.balance += amount;
            logTransaction(accountNumber, 'DEPOSIT', amount, account.balance);
            
            console.log(`Deposited AED ${amount} to ${accountNumber}`);
            return account.balance;
        },
        
        // Withdraw money
        withdraw: function(accountNumber, amount) {
            validateAmount(amount);
            const account = getAccount(accountNumber);
            
            if (account.balance < amount) {
                throw new Error('Insufficient funds');
            }
            
            account.balance -= amount;
            logTransaction(accountNumber, 'WITHDRAWAL', amount, account.balance);
            
            console.log(`Withdrew AED ${amount} from ${accountNumber}`);
            return account.balance;
        },
        
        // Transfer between accounts
        transfer: function(fromAccount, toAccount, amount) {
            validateAmount(amount);
            
            this.withdraw(fromAccount, amount);
            this.deposit(toAccount, amount);
            
            logTransaction(fromAccount, 'TRANSFER_OUT', amount, this.getBalance(fromAccount).balance);
            logTransaction(toAccount, 'TRANSFER_IN', amount, this.getBalance(toAccount).balance);
            
            console.log(`Transferred AED ${amount} from ${fromAccount} to ${toAccount}`);
        },
        
        // Get transaction history
        getTransactionHistory: function(accountNumber) {
            return transactionLog
                .filter(txn => txn.accountNumber === accountNumber)
                .map(txn => ({ ...txn })); // Return copy
        },
        
        // Get statistics (can't access private data directly)
        getStatistics: function() {
            return {
                totalAccounts: accounts.size,
                totalTransactions: transactionLog.length,
                nextAccountNumber: `ENBD${String(nextAccountNumber).padStart(8, '0')}`
            };
        }
    };
})();

console.log('=== Banking Module Pattern ===\n');

// Use the banking module
const acc1 = BankingModule.createAccount('Ahmed Al Mansouri', 10000);
const acc2 = BankingModule.createAccount('Fatima Hassan', 5000);

console.log('\nBalance acc1:', BankingModule.getBalance(acc1));

BankingModule.deposit(acc1, 2000);
BankingModule.withdraw(acc1, 1000);
BankingModule.transfer(acc1, acc2, 3000);

console.log('\nTransaction history acc1:', BankingModule.getTransactionHistory(acc1));
console.log('\nStatistics:', BankingModule.getStatistics());

// Try to access private data
console.log('\nDirect access to accounts:', typeof accounts); // undefined
console.log('Direct access to transactionLog:', typeof transactionLog); // undefined

// Revealing Module Pattern
console.log('\n=== Revealing Module Pattern ===\n');

const AccountManager = (function() {
    // Private state
    const _accounts = {};
    let _accountCounter = 0;
    
    // Private methods
    function _generateId() {
        return `ACC_${++_accountCounter}`;
    }
    
    function _validateAccount(accountId) {
        if (!_accounts[accountId]) {
            throw new Error(`Account ${accountId} not found`);
        }
        return _accounts[accountId];
    }
    
    function _recordActivity(accountId, activity) {
        const account = _accounts[accountId];
        if (!account.activityLog) {
            account.activityLog = [];
        }
        account.activityLog.push({
            timestamp: new Date().toISOString(),
            activity
        });
    }
    
    // Public methods
    function createAccount(holder, type, initialBalance) {
        const accountId = _generateId();
        _accounts[accountId] = {
            id: accountId,
            holder,
            type,
            balance: initialBalance,
            status: 'ACTIVE',
            createdAt: new Date()
        };
        
        _recordActivity(accountId, 'ACCOUNT_CREATED');
        return accountId;
    }
    
    function getAccountInfo(accountId) {
        const account = _validateAccount(accountId);
        return {
            id: account.id,
            holder: account.holder,
            type: account.type,
            balance: account.balance,
            status: account.status
        };
    }
    
    function updateBalance(accountId, amount) {
        const account = _validateAccount(accountId);
        account.balance += amount;
        _recordActivity(accountId, `BALANCE_UPDATED: ${amount > 0 ? '+' : ''}${amount}`);
        return account.balance;
    }
    
    function closeAccount(accountId) {
        const account = _validateAccount(accountId);
        account.status = 'CLOSED';
        _recordActivity(accountId, 'ACCOUNT_CLOSED');
    }
    
    function getActivityLog(accountId) {
        const account = _validateAccount(accountId);
        return account.activityLog || [];
    }
    
    // Reveal public methods
    return {
        createAccount,
        getAccountInfo,
        updateBalance,
        closeAccount,
        getActivityLog
    };
})();

const account1 = AccountManager.createAccount('Omar Abdullah', 'SAVINGS', 20000);
console.log('Account created:', account1);
console.log('Account info:', AccountManager.getAccountInfo(account1));

AccountManager.updateBalance(account1, 5000);
AccountManager.updateBalance(account1, -2000);

console.log('Activity log:', AccountManager.getActivityLog(account1));

// Factory Pattern with Closures
console.log('\n=== Factory Pattern with Closures ===\n');

function createBankAccount(accountNumber, accountHolder, initialBalance) {
    // Private variables
    let balance = initialBalance;
    let isLocked = false;
    const transactionHistory = [];
    
    // Private methods
    function addTransaction(type, amount) {
        transactionHistory.push({
            id: transactionHistory.length + 1,
            type,
            amount,
            balance,
            timestamp: new Date().toISOString()
        });
    }
    
    function checkLocked() {
        if (isLocked) {
            throw new Error('Account is locked');
        }
    }
    
    // Public interface
    return {
        getAccountNumber: () => accountNumber,
        getAccountHolder: () => accountHolder,
        
        getBalance: function() {
            return balance;
        },
        
        deposit: function(amount) {
            checkLocked();
            
            if (amount <= 0) {
                throw new Error('Amount must be positive');
            }
            
            balance += amount;
            addTransaction('DEPOSIT', amount);
            
            console.log(`Deposited AED ${amount}. New balance: AED ${balance}`);
            return balance;
        },
        
        withdraw: function(amount) {
            checkLocked();
            
            if (amount <= 0) {
                throw new Error('Amount must be positive');
            }
            
            if (amount > balance) {
                throw new Error('Insufficient funds');
            }
            
            balance -= amount;
            addTransaction('WITHDRAWAL', amount);
            
            console.log(`Withdrew AED ${amount}. New balance: AED ${balance}`);
            return balance;
        },
        
        lock: function() {
            isLocked = true;
            console.log(`Account ${accountNumber} locked`);
        },
        
        unlock: function() {
            isLocked = false;
            console.log(`Account ${accountNumber} unlocked`);
        },
        
        getTransactionHistory: function() {
            // Return copy to prevent external modification
            return transactionHistory.map(txn => ({ ...txn }));
        },
        
        getStatement: function() {
            return {
                accountNumber,
                accountHolder,
                currentBalance: balance,
                isLocked,
                totalTransactions: transactionHistory.length,
                transactions: this.getTransactionHistory()
            };
        }
    };
}

// Create accounts using factory
const ahmadAccount = createBankAccount('ENBD12345678', 'Ahmad Ali', 15000);
const sarahAccount = createBankAccount('ENBD87654321', 'Sarah Hassan', 25000);

console.log('Ahmad balance:', ahmadAccount.getBalance());
ahmadAccount.deposit(3000);
ahmadAccount.withdraw(2000);

console.log('\nAhmad statement:', ahmadAccount.getStatement());

// Private data is truly private
console.log('\nDirect balance access:', ahmadAccount.balance); // undefined
console.log('Direct transactionHistory access:', ahmadAccount.transactionHistory); // undefined

// Singleton Pattern with Closure
console.log('\n=== Singleton Pattern ===\n');

const BankingConfig = (function() {
    let instance;
    
    function createInstance() {
        // Private configuration
        const config = {
            bankName: 'Emirates NBD',
            swiftCode: 'EBILAEAD',
            minBalance: 1000,
            maxDailyTransfer: 50000,
            interestRates: {
                savings: 0.025,
                current: 0.001,
                fixedDeposit: 0.04
            }
        };
        
        return {
            getBankName: () => config.bankName,
            getSwiftCode: () => config.swiftCode,
            getMinBalance: () => config.minBalance,
            getMaxDailyTransfer: () => config.maxDailyTransfer,
            getInterestRate: (accountType) => config.interestRates[accountType] || 0,
            
            setMaxDailyTransfer: function(amount) {
                config.maxDailyTransfer = amount;
                console.log(`Max daily transfer updated to AED ${amount}`);
            },
            
            getAllConfig: function() {
                return {
                    bankName: config.bankName,
                    swiftCode: config.swiftCode,
                    minBalance: config.minBalance,
                    maxDailyTransfer: config.maxDailyTransfer,
                    interestRates: { ...config.interestRates }
                };
            }
        };
    }
    
    return {
        getInstance: function() {
            if (!instance) {
                instance = createInstance();
            }
            return instance;
        }
    };
})();

// Get singleton instance
const config1 = BankingConfig.getInstance();
const config2 = BankingConfig.getInstance();

console.log('Same instance?', config1 === config2); // true
console.log('Bank name:', config1.getBankName());
console.log('Savings rate:', config1.getInterestRate('savings'));

config1.setMaxDailyTransfer(100000);
console.log('Updated max transfer:', config2.getMaxDailyTransfer()); // Also updated
```

---

## Q12: Explain generators and iterators with banking data streaming

**Answer:**

Generators are functions that can pause and resume execution, useful for lazy evaluation and data streaming.

**Banking Data Streaming with Generators:**

```javascript
// Generator Functions for Banking
console.log('=== Generator Functions ===\n');

// Basic generator: Account number generator
function* accountNumberGenerator(prefix = 'ENBD', startFrom = 1000) {
    let counter = startFrom;
    while (true) {
        yield `${prefix}${String(counter++).padStart(8, '0')}`;
    }
}

const accountGen = accountNumberGenerator();
console.log('Account 1:', accountGen.next().value);
console.log('Account 2:', accountGen.next().value);
console.log('Account 3:', accountGen.next().value);

// Generator: Transaction stream
function* transactionStream(transactions) {
    console.log('\nStreaming transactions...');
    
    for (const transaction of transactions) {
        console.log(`Processing: ${transaction.id}`);
        yield transaction;
    }
    
    console.log('Stream complete');
}

const transactions = [
    { id: 'TXN001', amount: 1000, type: 'CREDIT' },
    { id: 'TXN002', amount: 500, type: 'DEBIT' },
    { id: 'TXN003', amount: 2000, type: 'CREDIT' }
];

const txnStream = transactionStream(transactions);
console.log('\nFirst transaction:', txnStream.next().value);
console.log('Second transaction:', txnStream.next().value);

// Generator: Paginated data
function* paginatedTransactions(allTransactions, pageSize = 5) {
    let currentPage = 0;
    
    while (currentPage * pageSize < allTransactions.length) {
        const start = currentPage * pageSize;
        const end = start + pageSize;
        const page = allTransactions.slice(start, end);
        
        yield {
            page: currentPage + 1,
            data: page,
            hasMore: end < allTransactions.length
        };
        
        currentPage++;
    }
}

const allTxns = Array.from({ length: 23 }, (_, i) => ({
    id: `TXN${String(i + 1).padStart(3, '0')}`,
    amount: Math.random() * 5000
}));

console.log('\n=== Paginated Data ===\n');
const paginator = paginatedTransactions(allTxns, 10);

let page1 = paginator.next().value;
console.log(`Page ${page1.page}: ${page1.data.length} transactions, hasMore: ${page1.hasMore}`);

let page2 = paginator.next().value;
console.log(`Page ${page2.page}: ${page2.data.length} transactions, hasMore: ${page2.hasMore}`);

// Generator: Infinite sequence
function* interestCalculator(principal, rate) {
    let balance = principal;
    let year = 0;
    
    while (true) {
        year++;
        const interest = balance * rate;
        balance += interest;
        
        yield {
            year,
            interest,
            balance: Math.round(balance * 100) / 100
        };
    }
}

console.log('\n=== Interest Calculation Stream ===\n');
const interestGen = interestCalculator(10000, 0.05);

for (let i = 0; i < 5; i++) {
    const result = interestGen.next().value;
    console.log(`Year ${result.year}: Interest AED ${result.interest.toFixed(2)}, Balance AED ${result.balance}`);
}

// Custom Iterator for Account Collection
console.log('\n=== Custom Iterator ===\n');

class AccountCollection {
    constructor() {
        this.accounts = [];
    }
    
    addAccount(account) {
        this.accounts.push(account);
    }
    
    // Make collection iterable
    [Symbol.iterator]() {
        let index = 0;
        const accounts = this.accounts;
        
        return {
            next() {
                if (index < accounts.length) {
                    return {
                        value: accounts[index++],
                        done: false
                    };
                }
                return { done: true };
            }
        };
    }
    
    // Generator method for filtered iteration
    *filterByType(accountType) {
        for (const account of this.accounts) {
            if (account.type === accountType) {
                yield account;
            }
        }
    }
    
    // Generator method for balance above threshold
    *balanceAbove(threshold) {
        for (const account of this.accounts) {
            if (account.balance > threshold) {
                yield account;
            }
        }
    }
}

const collection = new AccountCollection();
collection.addAccount({ id: 'ACC001', type: 'SAVINGS', balance: 15000 });
collection.addAccount({ id: 'ACC002', type: 'CURRENT', balance: 5000 });
collection.addAccount({ id: 'ACC003', type: 'SAVINGS', balance: 25000 });
collection.addAccount({ id: 'ACC004', type: 'CURRENT', balance: 8000 });

console.log('All accounts:');
for (const account of collection) {
    console.log(`${account.id}: ${account.type} - AED ${account.balance}`);
}

console.log('\nSavings accounts only:');
for (const account of collection.filterByType('SAVINGS')) {
    console.log(`${account.id}: AED ${account.balance}`);
}

console.log('\nAccounts with balance > 10000:');
for (const account of collection.balanceAbove(10000)) {
    console.log(`${account.id}: AED ${account.balance}`);
}

// Async Generator for API data streaming
console.log('\n=== Async Generator ===\n');

async function* fetchTransactionBatches(accountId, batchSize = 100) {
    let offset = 0;
    let hasMore = true;
    
    while (hasMore) {
        // Simulate API call
        const response = await simulateAPICall(accountId, offset, batchSize);
        
        yield response.data;
        
        hasMore = response.hasMore;
        offset += batchSize;
        
        console.log(`Fetched batch at offset ${offset - batchSize}`);
    }
}

function simulateAPICall(accountId, offset, limit) {
    return new Promise(resolve => {
        setTimeout(() => {
            const data = Array.from({ length: Math.min(limit, 250 - offset) }, (_, i) => ({
                id: `TXN${offset + i + 1}`,
                amount: Math.random() * 1000
            }));
            
            resolve({
                data,
                hasMore: offset + limit < 250
            });
        }, 100);
    });
}

// Use async generator
async function processTransactionStream() {
    const generator = fetchTransactionBatches('ACC001', 100);
    
    let totalTransactions = 0;
    for await (const batch of generator) {
        totalTransactions += batch.length;
        console.log(`Processed batch of ${batch.length} transactions`);
    }
    
    console.log(`Total transactions processed: ${totalTransactions}`);
}

processTransactionStream();

// Generator for state machine
console.log('\n=== State Machine with Generator ===\n');

function* transactionStateMachine(transaction) {
    console.log('State: INITIATED');
    yield 'INITIATED';
    
    console.log('State: VALIDATING');
    // Simulate validation
    if (transaction.amount <= 0) {
        console.log('State: REJECTED');
        return 'REJECTED';
    }
    yield 'VALIDATING';
    
    console.log('State: PROCESSING');
    yield 'PROCESSING';
    
    console.log('State: COMPLETED');
    yield 'COMPLETED';
    
    return 'SUCCESS';
}

const stateMachine = transactionStateMachine({ amount: 1000 });

console.log(stateMachine.next()); // INITIATED
console.log(stateMachine.next()); // VALIDATING
console.log(stateMachine.next()); // PROCESSING
console.log(stateMachine.next()); // COMPLETED
console.log(stateMachine.next()); // Done

// Real-world: Statement Generator
console.log('\n=== Statement Generator ===\n');

class StatementGenerator {
    constructor(accountId, transactions) {
        this.accountId = accountId;
        this.transactions = transactions;
    }
    
    *generateByMonth() {
        const byMonth = this.transactions.reduce((acc, txn) => {
            const month = txn.date.substring(0, 7); // YYYY-MM
            if (!acc[month]) acc[month] = [];
            acc[month].push(txn);
            return acc;
        }, {});
        
        for (const [month, txns] of Object.entries(byMonth)) {
            const total = txns.reduce((sum, txn) => 
                sum + (txn.type === 'CREDIT' ? txn.amount : -txn.amount), 0
            );
            
            yield {
                month,
                transactions: txns.length,
                credits: txns.filter(t => t.type === 'CREDIT').length,
                debits: txns.filter(t => t.type === 'DEBIT').length,
                netAmount: total
            };
        }
    }
    
    *generateByCategory() {
        const byCategory = this.transactions.reduce((acc, txn) => {
            const cat = txn.category || 'UNCATEGORIZED';
            if (!acc[cat]) acc[cat] = [];
            acc[cat].push(txn);
            return acc;
        }, {});
        
        for (const [category, txns] of Object.entries(byCategory)) {
            const total = txns.reduce((sum, txn) => sum + txn.amount, 0);
            
            yield {
                category,
                count: txns.length,
                total,
                average: total / txns.length
            };
        }
    }
}

const statementData = [
    { date: '2024-01-15', amount: 5000, type: 'CREDIT', category: 'SALARY' },
    { date: '2024-01-16', amount: 1200, type: 'DEBIT', category: 'RENT' },
    { date: '2024-02-15', amount: 5000, type: 'CREDIT', category: 'SALARY' },
    { date: '2024-02-20', amount: 500, type: 'DEBIT', category: 'GROCERIES' }
];

const statementGen = new StatementGenerator('ACC001', statementData);

console.log('Monthly summary:');
for (const monthData of statementGen.generateByMonth()) {
    console.log(`${monthData.month}: ${monthData.transactions} txns, Net: AED ${monthData.netAmount}`);
}

console.log('\nCategory summary:');
for (const catData of statementGen.generateByCategory()) {
    console.log(`${catData.category}: ${catData.count} txns, Total: AED ${catData.total.toFixed(2)}`);
}
```

---
