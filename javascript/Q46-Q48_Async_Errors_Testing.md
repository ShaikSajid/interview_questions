# Questions 46-48: Advanced Async Patterns, Error Handling & Testing

## Question 46: Explain Generators, Iterators, and async iteration in JavaScript

### Answer:

**Generators** are special functions that can pause execution and resume later, making them perfect for iterating over sequences, handling async flows, and managing state.

### Key Concepts:
1. **Iterators** - Objects with next() method
2. **Generators** - Functions with function* syntax
3. **yield** - Pause and return values
4. **Async Generators** - async function* for async iteration
5. **for...of** - Iterate over iterables

### Banking Scenario: Generators & Iterators at Emirates NBD

```javascript
console.log('=== Generators & Iterators - Emirates NBD ===\n');

// =============================================================================
// 1. BASIC ITERATOR PROTOCOL
// =============================================================================

console.log('1. ITERATOR PROTOCOL:\n');

// Manual iterator
const accountIterator = {
    accounts: ['ACC-001', 'ACC-002', 'ACC-003'],
    index: 0,
    
    next() {
        if (this.index < this.accounts.length) {
            return {
                value: this.accounts[this.index++],
                done: false
            };
        } else {
            return { done: true };
        }
    }
};

console.log('Manual iterator:');
console.log(`  ${accountIterator.next().value}`);
console.log(`  ${accountIterator.next().value}`);
console.log(`  ${accountIterator.next().value}`);
console.log(`  ${accountIterator.next().done}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. GENERATOR FUNCTIONS
// =============================================================================

console.log('2. GENERATOR FUNCTIONS:\n');

function* accountGenerator() {
    yield 'ACC-001';
    yield 'ACC-002';
    yield 'ACC-003';
}

console.log('Basic generator:');
const gen = accountGenerator();
console.log(`  ${gen.next().value}`);
console.log(`  ${gen.next().value}`);
console.log(`  ${gen.next().value}`);
console.log(`  ${gen.next().done}\n`);

// Generator with for...of
console.log('Using for...of:');
for (const account of accountGenerator()) {
    console.log(`  ${account}`);
}
console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. PRACTICAL TRANSACTION GENERATOR
// =============================================================================

console.log('3. TRANSACTION GENERATOR:\n');

function* transactionGenerator(accountNumber, balance) {
    console.log(`  Starting transactions for ${accountNumber}\n`);
    
    // Transaction 1: Deposit
    const deposit = yield { type: 'DEPOSIT', amount: 5000, balance: balance + 5000 };
    balance += 5000;
    console.log(`  Processed: Deposit ${deposit}`);
    
    // Transaction 2: Withdrawal
    const withdrawal = yield { type: 'WITHDRAWAL', amount: 2000, balance: balance - 2000 };
    balance -= 2000;
    console.log(`  Processed: Withdrawal ${withdrawal}`);
    
    // Transaction 3: Interest
    const interest = balance * 0.035;
    yield { type: 'INTEREST', amount: interest, balance: balance + interest };
    console.log(`  Processed: Interest ${interest.toFixed(2)}`);
    
    return { message: 'Transactions complete', finalBalance: balance };
}

console.log('Transaction sequence:\n');
const txnGen = transactionGenerator('ACC-123456789', 50000);

let result = txnGen.next();
console.log(`  ${result.value.type}: ${result.value.amount}\n`);

result = txnGen.next(5000);
console.log(`  Balance: ${result.value.balance}\n`);

result = txnGen.next(2000);
console.log(`  Balance: ${result.value.balance}\n`);

result = txnGen.next();
console.log(`  ${result.value.type}: ${result.value.amount.toFixed(2)}\n`);

result = txnGen.next();
console.log(`  ${result.value.message}: ${result.value.finalBalance}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. INFINITE SEQUENCES
// =============================================================================

console.log('4. INFINITE SEQUENCES:\n');

function* accountNumberGenerator() {
    let id = 1;
    while (true) {
        yield `ACC-${String(id).padStart(9, '0')}`;
        id++;
    }
}

console.log('Generating account numbers:');
const accountGen = accountNumberGenerator();
for (let i = 0; i < 5; i++) {
    console.log(`  ${accountGen.next().value}`);
}
console.log();

// Transaction ID generator
function* transactionIdGenerator(prefix = 'TXN') {
    let id = 1;
    while (true) {
        yield `${prefix}-${Date.now()}-${String(id).padStart(6, '0')}`;
        id++;
    }
}

console.log('Generating transaction IDs:');
const txnIdGen = transactionIdGenerator();
for (let i = 0; i < 3; i++) {
    console.log(`  ${txnIdGen.next().value}`);
}
console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. GENERATOR DELEGATION
// =============================================================================

console.log('5. GENERATOR DELEGATION:\n');

function* savingsAccountTransactions() {
    yield { account: 'SAVINGS-001', type: 'DEPOSIT', amount: 5000 };
    yield { account: 'SAVINGS-001', type: 'INTEREST', amount: 175 };
}

function* currentAccountTransactions() {
    yield { account: 'CURRENT-001', type: 'DEPOSIT', amount: 3000 };
    yield { account: 'CURRENT-001', type: 'WITHDRAWAL', amount: 1000 };
}

function* allTransactions() {
    console.log('  Processing savings transactions:');
    yield* savingsAccountTransactions();
    console.log('  Processing current transactions:');
    yield* currentAccountTransactions();
}

console.log('Delegating to sub-generators:\n');
for (const txn of allTransactions()) {
    console.log(`    ${txn.account}: ${txn.type} ${txn.amount}`);
}
console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 6. ASYNC GENERATORS
// =============================================================================

console.log('6. ASYNC GENERATORS:\n');

// Simulate async data source
async function fetchAccountData(accountNumber) {
    return new Promise(resolve => {
        setTimeout(() => {
            resolve({
                accountNumber,
                balance: Math.floor(Math.random() * 100000),
                timestamp: new Date()
            });
        }, 100);
    });
}

async function* asyncAccountGenerator(accountNumbers) {
    for (const accountNumber of accountNumbers) {
        const data = await fetchAccountData(accountNumber);
        yield data;
    }
}

(async () => {
    console.log('Fetching account data asynchronously:\n');
    
    const accounts = ['ACC-001', 'ACC-002', 'ACC-003'];
    const asyncGen = asyncAccountGenerator(accounts);
    
    for await (const account of asyncGen) {
        console.log(`  ${account.accountNumber}: ${account.balance} AED`);
    }
    console.log();
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 7. ASYNC ITERATION FOR PAGINATED DATA
    // =============================================================================
    
    console.log('7. PAGINATED DATA:\n');
    
    // Simulate API with pagination
    async function fetchTransactionPage(page, pageSize) {
        return new Promise(resolve => {
            setTimeout(() => {
                const transactions = [];
                for (let i = 0; i < pageSize; i++) {
                    const txnId = page * pageSize + i + 1;
                    if (txnId <= 25) { // Total 25 transactions
                        transactions.push({
                            id: `TXN-${String(txnId).padStart(3, '0')}`,
                            amount: Math.floor(Math.random() * 10000)
                        });
                    }
                }
                resolve(transactions);
            }, 50);
        });
    }
    
    async function* paginatedTransactions(pageSize = 10) {
        let page = 0;
        let hasMore = true;
        
        while (hasMore) {
            const transactions = await fetchTransactionPage(page, pageSize);
            
            if (transactions.length === 0) {
                hasMore = false;
            } else {
                yield transactions;
                page++;
            }
        }
    }
    
    console.log('Fetching paginated transactions:\n');
    
    let totalTransactions = 0;
    for await (const page of paginatedTransactions(10)) {
        console.log(`  Page ${Math.floor(totalTransactions / 10) + 1}: ${page.length} transactions`);
        totalTransactions += page.length;
    }
    
    console.log(`\n  Total transactions: ${totalTransactions}\n`);
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 8. PRACTICAL TRANSACTION STREAM
    // =============================================================================
    
    console.log('8. TRANSACTION STREAM:\n');
    
    class TransactionStream {
        constructor() {
            this.transactions = [];
        }
        
        addTransaction(transaction) {
            this.transactions.push(transaction);
        }
        
        *[Symbol.iterator]() {
            for (const txn of this.transactions) {
                yield txn;
            }
        }
        
        async *asyncProcess() {
            for (const txn of this.transactions) {
                // Simulate async processing
                await new Promise(resolve => setTimeout(resolve, 50));
                
                yield {
                    ...txn,
                    processed: true,
                    processedAt: new Date()
                };
            }
        }
        
        *filter(predicate) {
            for (const txn of this.transactions) {
                if (predicate(txn)) {
                    yield txn;
                }
            }
        }
        
        *map(transformer) {
            for (const txn of this.transactions) {
                yield transformer(txn);
            }
        }
    }
    
    const stream = new TransactionStream();
    stream.addTransaction({ id: 'TXN-001', amount: 5000, type: 'DEPOSIT' });
    stream.addTransaction({ id: 'TXN-002', amount: 2000, type: 'WITHDRAWAL' });
    stream.addTransaction({ id: 'TXN-003', amount: 10000, type: 'DEPOSIT' });
    stream.addTransaction({ id: 'TXN-004', amount: 500, type: 'WITHDRAWAL' });
    
    console.log('All transactions:');
    for (const txn of stream) {
        console.log(`  ${txn.id}: ${txn.type} ${txn.amount}`);
    }
    console.log();
    
    console.log('Filtered (Deposits only):');
    for (const txn of stream.filter(t => t.type === 'DEPOSIT')) {
        console.log(`  ${txn.id}: ${txn.amount}`);
    }
    console.log();
    
    console.log('Mapped (with fees):');
    for (const txn of stream.map(t => ({ ...t, fee: t.amount * 0.001 }))) {
        console.log(`  ${txn.id}: ${txn.amount} (fee: ${txn.fee.toFixed(2)})`);
    }
    console.log();
    
    console.log('Async processing:');
    for await (const txn of stream.asyncProcess()) {
        console.log(`  ${txn.id}: Processed at ${txn.processedAt.toTimeString().split(' ')[0]}`);
    }
    console.log();
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 9. GENERATOR-BASED STATE MACHINE
    // =============================================================================
    
    console.log('9. STATE MACHINE:\n');
    
    function* accountStateMachine() {
        let state = 'PENDING';
        console.log(`  Initial state: ${state}\n`);
        
        // Pending -> Active
        const activationCode = yield state;
        if (activationCode === 'ACTIVATE') {
            state = 'ACTIVE';
            console.log(`  State changed: ${state}\n`);
        }
        
        // Active -> Frozen or Closed
        const action = yield state;
        if (action === 'FREEZE') {
            state = 'FROZEN';
            console.log(`  State changed: ${state}\n`);
        } else if (action === 'CLOSE') {
            state = 'CLOSED';
            console.log(`  State changed: ${state}\n`);
        }
        
        // Frozen -> Active
        if (state === 'FROZEN') {
            const unfreezeCode = yield state;
            if (unfreezeCode === 'UNFREEZE') {
                state = 'ACTIVE';
                console.log(`  State changed: ${state}\n`);
            }
        }
        
        return state;
    }
    
    console.log('Account state transitions:\n');
    
    const stateMachine = accountStateMachine();
    
    let currentState = stateMachine.next();
    console.log(`  Current: ${currentState.value}`);
    
    currentState = stateMachine.next('ACTIVATE');
    console.log(`  Current: ${currentState.value}`);
    
    currentState = stateMachine.next('FREEZE');
    console.log(`  Current: ${currentState.value}`);
    
    currentState = stateMachine.next('UNFREEZE');
    console.log(`  Current: ${currentState.value}`);
    
    currentState = stateMachine.next();
    console.log(`  Final: ${currentState.value}\n`);
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // GENERATORS SUMMARY
    // =============================================================================
    
    console.log('GENERATORS SUMMARY:\n');
    
    console.log('Iterator Protocol:');
    console.log('  • next() method returns { value, done }');
    console.log('  • Used by for...of, spread operator');
    console.log('  • Custom iteration behavior\n');
    
    console.log('Generator Functions:');
    console.log('  • function* syntax');
    console.log('  • yield to pause and return values');
    console.log('  • return to complete');
    console.log('  • Can receive values via next(value)\n');
    
    console.log('Async Generators:');
    console.log('  • async function* syntax');
    console.log('  • yield await for async values');
    console.log('  • for await...of for iteration');
    console.log('  • Perfect for async data streams\n');
    
    console.log('Use Cases:');
    console.log('  ✓ Lazy evaluation of sequences');
    console.log('  ✓ Infinite sequences (IDs, pagination)');
    console.log('  ✓ State machines');
    console.log('  ✓ Async data streams');
    console.log('  ✓ Custom iteration logic');
    console.log('  ✓ Memory-efficient data processing\n');
    
    console.log('Best Practices:');
    console.log('  ✓ Use for lazy evaluation');
    console.log('  ✓ Async generators for async sequences');
    console.log('  ✓ Generator delegation with yield*');
    console.log('  ✓ Combine with for...of');
    console.log('  ✓ Implement Symbol.iterator for custom iterables');
    
    console.log();
})();
```

---

## Question 47: Explain comprehensive error handling strategies in JavaScript applications

### Answer:

**Error Handling** is crucial for building robust applications. JavaScript provides multiple mechanisms for handling errors gracefully and maintaining application stability.

### Error Handling Mechanisms:
1. **try...catch...finally** - Synchronous error handling
2. **Promise.catch()** - Promise error handling
3. **async/await with try...catch** - Async error handling
4. **Error objects** - Custom error types
5. **Global handlers** - window.onerror, unhandledrejection

### Banking Scenario: Error Handling at Emirates NBD

```javascript
console.log('=== Error Handling - Emirates NBD ===\n');

// =============================================================================
// 1. BASIC ERROR HANDLING
// =============================================================================

console.log('1. BASIC TRY...CATCH:\n');

function validateAccount(accountNumber) {
    if (!accountNumber) {
        throw new Error('Account number is required');
    }
    
    if (!/^ACC-\d{9}$/.test(accountNumber)) {
        throw new Error('Invalid account number format');
    }
    
    return true;
}

try {
    console.log('  Validating valid account:');
    validateAccount('ACC-123456789');
    console.log('  ✓ Valid\n');
    
    console.log('  Validating invalid account:');
    validateAccount('INVALID');
} catch (error) {
    console.log(`  ✗ Error: ${error.message}\n`);
} finally {
    console.log('  Validation complete\n');
}

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. CUSTOM ERROR CLASSES
// =============================================================================

console.log('2. CUSTOM ERROR CLASSES:\n');

class BankingError extends Error {
    constructor(message, code) {
        super(message);
        this.name = 'BankingError';
        this.code = code;
        this.timestamp = new Date();
    }
}

class InsufficientFundsError extends BankingError {
    constructor(required, available) {
        super(`Insufficient funds. Required: ${required}, Available: ${available}`, 'INSUFFICIENT_FUNDS');
        this.name = 'InsufficientFundsError';
        this.required = required;
        this.available = available;
    }
}

class AccountNotFoundError extends BankingError {
    constructor(accountNumber) {
        super(`Account not found: ${accountNumber}`, 'ACCOUNT_NOT_FOUND');
        this.name = 'AccountNotFoundError';
        this.accountNumber = accountNumber;
    }
}

class TransactionLimitError extends BankingError {
    constructor(amount, limit) {
        super(`Transaction exceeds limit. Amount: ${amount}, Limit: ${limit}`, 'LIMIT_EXCEEDED');
        this.name = 'TransactionLimitError';
        this.amount = amount;
        this.limit = limit;
    }
}

function withdraw(balance, amount) {
    if (amount > balance) {
        throw new InsufficientFundsError(amount, balance);
    }
    return balance - amount;
}

try {
    console.log('  Attempting withdrawal:');
    withdraw(5000, 10000);
} catch (error) {
    if (error instanceof InsufficientFundsError) {
        console.log(`  ✗ ${error.name}: ${error.message}`);
        console.log(`    Code: ${error.code}`);
        console.log(`    Required: ${error.required}`);
        console.log(`    Available: ${error.available}\n`);
    }
}

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. ASYNC ERROR HANDLING
// =============================================================================

console.log('3. ASYNC ERROR HANDLING:\n');

// Promise-based
function fetchAccountData(accountNumber) {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            if (accountNumber === 'ACC-999999999') {
                reject(new AccountNotFoundError(accountNumber));
            } else {
                resolve({ accountNumber, balance: 50000 });
            }
        }, 100);
    });
}

console.log('Promise with .catch():');
fetchAccountData('ACC-999999999')
    .then(account => {
        console.log(`  Account found: ${account.accountNumber}`);
    })
    .catch(error => {
        console.log(`  ✗ ${error.name}: ${error.message}\n`);
    });

// Async/await
async function getAccountBalance(accountNumber) {
    try {
        console.log('  Fetching account data...');
        const account = await fetchAccountData(accountNumber);
        console.log(`  Balance: ${account.balance}`);
        return account.balance;
    } catch (error) {
        console.log(`  ✗ ${error.name}: ${error.message}`);
        throw error; // Re-throw for caller to handle
    }
}

setTimeout(async () => {
    console.log('\nAsync/await with try...catch:');
    try {
        await getAccountBalance('ACC-123456789');
    } catch (error) {
        console.log('  Handled by caller\n');
    }
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 4. ERROR RECOVERY STRATEGIES
    // =============================================================================
    
    console.log('4. ERROR RECOVERY:\n');
    
    class TransactionService {
        constructor() {
            this.retryCount = 3;
            this.retryDelay = 100;
        }
        
        async processWithRetry(operation, ...args) {
            let lastError;
            
            for (let attempt = 1; attempt <= this.retryCount; attempt++) {
                try {
                    console.log(`  Attempt ${attempt}/${this.retryCount}`);
                    const result = await operation(...args);
                    console.log(`  ✓ Success\n`);
                    return result;
                } catch (error) {
                    lastError = error;
                    console.log(`  ✗ Failed: ${error.message}`);
                    
                    if (attempt < this.retryCount) {
                        console.log(`  Retrying in ${this.retryDelay}ms...\n`);
                        await new Promise(resolve => setTimeout(resolve, this.retryDelay));
                    }
                }
            }
            
            console.log(`  All attempts failed\n`);
            throw lastError;
        }
        
        async processWithFallback(primary, fallback) {
            try {
                console.log('  Trying primary operation...');
                return await primary();
            } catch (error) {
                console.log(`  ✗ Primary failed: ${error.message}`);
                console.log('  Trying fallback operation...');
                return await fallback();
            }
        }
    }
    
    // Simulate unreliable operation
    let attemptNumber = 0;
    async function unreliableOperation() {
        attemptNumber++;
        if (attemptNumber < 3) {
            throw new Error('Network timeout');
        }
        return { success: true };
    }
    
    console.log('Retry strategy:\n');
    const service = new TransactionService();
    
    try {
        await service.processWithRetry(unreliableOperation);
    } catch (error) {
        console.log(`  Final error: ${error.message}\n`);
    }
    
    console.log('Fallback strategy:\n');
    await service.processWithFallback(
        async () => {
            throw new Error('Primary service down');
        },
        async () => {
            console.log('  ✓ Fallback successful\n');
            return { source: 'fallback', data: 'cached data' };
        }
    );
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 5. COMPREHENSIVE ERROR HANDLER
    // =============================================================================
    
    console.log('5. COMPREHENSIVE ERROR HANDLER:\n');
    
    class ErrorHandler {
        constructor() {
            this.errorLog = [];
        }
        
        handle(error, context = {}) {
            const errorInfo = {
                name: error.name,
                message: error.message,
                code: error.code || 'UNKNOWN',
                stack: error.stack,
                context,
                timestamp: new Date()
            };
            
            this.errorLog.push(errorInfo);
            
            // Log error
            this.logError(errorInfo);
            
            // Determine severity
            const severity = this.getSeverity(error);
            
            // Handle based on severity
            if (severity === 'CRITICAL') {
                this.handleCriticalError(errorInfo);
            } else if (severity === 'WARNING') {
                this.handleWarning(errorInfo);
            }
            
            return errorInfo;
        }
        
        getSeverity(error) {
            if (error instanceof AccountNotFoundError) {
                return 'WARNING';
            }
            if (error instanceof InsufficientFundsError) {
                return 'WARNING';
            }
            if (error instanceof TransactionLimitError) {
                return 'WARNING';
            }
            return 'CRITICAL';
        }
        
        logError(errorInfo) {
            console.log(`  [${errorInfo.timestamp.toISOString()}] ${errorInfo.name}`);
            console.log(`  Message: ${errorInfo.message}`);
            console.log(`  Code: ${errorInfo.code}`);
            if (Object.keys(errorInfo.context).length > 0) {
                console.log(`  Context: ${JSON.stringify(errorInfo.context)}`);
            }
        }
        
        handleCriticalError(errorInfo) {
            console.log('  ⚠️  CRITICAL ERROR - Alerting operations team\n');
        }
        
        handleWarning(errorInfo) {
            console.log('  ℹ️  WARNING - Logged for review\n');
        }
        
        getErrorStats() {
            const stats = {};
            this.errorLog.forEach(error => {
                stats[error.code] = (stats[error.code] || 0) + 1;
            });
            return stats;
        }
    }
    
    const errorHandler = new ErrorHandler();
    
    console.log('Handling various errors:\n');
    
    try {
        throw new InsufficientFundsError(10000, 5000);
    } catch (error) {
        errorHandler.handle(error, { accountNumber: 'ACC-123456789', operation: 'withdraw' });
    }
    
    try {
        throw new AccountNotFoundError('ACC-999999999');
    } catch (error) {
        errorHandler.handle(error, { operation: 'getAccount' });
    }
    
    try {
        throw new Error('Database connection failed');
    } catch (error) {
        errorHandler.handle(error, { service: 'database' });
    }
    
    console.log('Error statistics:');
    console.log(`  ${JSON.stringify(errorHandler.getErrorStats(), null, 2)}\n`);
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 6. VALIDATION AND ERROR PREVENTION
    // =============================================================================
    
    console.log('6. VALIDATION:\n');
    
    class TransactionValidator {
        static validate(transaction) {
            const errors = [];
            
            if (!transaction.accountNumber) {
                errors.push('Account number is required');
            } else if (!/^ACC-\d{9}$/.test(transaction.accountNumber)) {
                errors.push('Invalid account number format');
            }
            
            if (!transaction.amount) {
                errors.push('Amount is required');
            } else if (transaction.amount <= 0) {
                errors.push('Amount must be positive');
            } else if (transaction.amount > 100000) {
                errors.push('Amount exceeds maximum limit');
            }
            
            if (!transaction.type) {
                errors.push('Transaction type is required');
            } else if (!['DEPOSIT', 'WITHDRAWAL', 'TRANSFER'].includes(transaction.type)) {
                errors.push('Invalid transaction type');
            }
            
            if (errors.length > 0) {
                throw new BankingError(
                    `Validation failed: ${errors.join(', ')}`,
                    'VALIDATION_ERROR'
                );
            }
            
            return true;
        }
    }
    
    function processTransaction(transaction) {
        try {
            console.log('  Validating transaction...');
            TransactionValidator.validate(transaction);
            console.log('  ✓ Validation passed');
            console.log('  Processing transaction...');
            console.log('  ✓ Transaction complete\n');
            return { success: true };
        } catch (error) {
            console.log(`  ✗ ${error.message}\n`);
            return { success: false, error: error.message };
        }
    }
    
    console.log('Valid transaction:');
    processTransaction({
        accountNumber: 'ACC-123456789',
        amount: 5000,
        type: 'DEPOSIT'
    });
    
    console.log('Invalid transaction:');
    processTransaction({
        accountNumber: 'INVALID',
        amount: -1000,
        type: 'UNKNOWN'
    });
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // ERROR HANDLING SUMMARY
    // =============================================================================
    
    console.log('ERROR HANDLING SUMMARY:\n');
    
    console.log('Error Types:');
    console.log('  • Error - Base error class');
    console.log('  • TypeError - Type-related errors');
    console.log('  • ReferenceError - Reference errors');
    console.log('  • RangeError - Range errors');
    console.log('  • SyntaxError - Syntax errors');
    console.log('  • Custom errors - Application-specific\n');
    
    console.log('Handling Mechanisms:');
    console.log('  • try...catch...finally - Synchronous');
    console.log('  • Promise.catch() - Promise errors');
    console.log('  • async/await with try...catch - Async errors');
    console.log('  • Error boundaries - React components\n');
    
    console.log('Recovery Strategies:');
    console.log('  • Retry with exponential backoff');
    console.log('  • Fallback to alternative methods');
    console.log('  • Circuit breaker pattern');
    console.log('  • Graceful degradation\n');
    
    console.log('Best Practices:');
    console.log('  ✓ Use custom error classes');
    console.log('  ✓ Include error codes');
    console.log('  ✓ Log errors with context');
    console.log('  ✓ Validate input early');
    console.log('  ✓ Handle errors at appropriate level');
    console.log('  ✓ Provide meaningful error messages');
    console.log('  ✓ Don\'t swallow errors');
    console.log('  ✓ Clean up resources in finally');
    
    console.log();
}, 500);
```

---

## Question 48: Explain testing strategies and methodologies in JavaScript

### Answer:

**Testing** ensures code quality, reliability, and maintainability. JavaScript applications require various testing approaches to verify functionality at different levels.

### Testing Types:
1. **Unit Tests** - Test individual functions/components
2. **Integration Tests** - Test component interactions
3. **End-to-End Tests** - Test complete user flows
4. **Performance Tests** - Test speed and efficiency
5. **Security Tests** - Test vulnerabilities

### Banking Scenario: Testing Strategies at Emirates NBD

```javascript
console.log('=== Testing Strategies - Emirates NBD ===\n');

// =============================================================================
// 1. UNIT TESTING FUNDAMENTALS
// =============================================================================

console.log('1. UNIT TESTING:\n');

// Simple assertion library
class Assert {
    static assertEqual(actual, expected, message = '') {
        if (actual === expected) {
            console.log(`  ✓ ${message || 'Assertion passed'}`);
            return true;
        } else {
            console.log(`  ✗ ${message || 'Assertion failed'}`);
            console.log(`    Expected: ${expected}`);
            console.log(`    Actual: ${actual}`);
            return false;
        }
    }
    
    static assertTrue(condition, message = '') {
        return this.assertEqual(condition, true, message);
    }
    
    static assertFalse(condition, message = '') {
        return this.assertEqual(condition, false, message);
    }
    
    static assertThrows(fn, expectedError, message = '') {
        try {
            fn();
            console.log(`  ✗ ${message || 'Expected error not thrown'}`);
            return false;
        } catch (error) {
            if (expectedError && !(error instanceof expectedError)) {
                console.log(`  ✗ ${message || 'Wrong error type'}`);
                console.log(`    Expected: ${expectedError.name}`);
                console.log(`    Actual: ${error.name}`);
                return false;
            }
            console.log(`  ✓ ${message || 'Correctly threw error'}`);
            return true;
        }
    }
}

// Code under test
class BankAccount {
    constructor(accountNumber, initialBalance = 0) {
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
    }
    
    deposit(amount) {
        if (amount <= 0) {
            throw new Error('Amount must be positive');
        }
        this.balance += amount;
        return this.balance;
    }
    
    withdraw(amount) {
        if (amount <= 0) {
            throw new Error('Amount must be positive');
        }
        if (amount > this.balance) {
            throw new Error('Insufficient funds');
        }
        this.balance -= amount;
        return this.balance;
    }
    
    getBalance() {
        return this.balance;
    }
}

// Unit tests
console.log('Testing BankAccount class:\n');

console.log('Test: Account creation');
const account = new BankAccount('ACC-123456789', 50000);
Assert.assertEqual(account.accountNumber, 'ACC-123456789', 'Account number set');
Assert.assertEqual(account.balance, 50000, 'Initial balance set');
console.log();

console.log('Test: Deposit');
account.deposit(5000);
Assert.assertEqual(account.balance, 55000, 'Balance after deposit');
console.log();

console.log('Test: Withdrawal');
account.withdraw(10000);
Assert.assertEqual(account.balance, 45000, 'Balance after withdrawal');
console.log();

console.log('Test: Error handling');
Assert.assertThrows(
    () => account.deposit(-1000),
    Error,
    'Negative deposit throws error'
);
Assert.assertThrows(
    () => account.withdraw(100000),
    Error,
    'Insufficient funds throws error'
);
console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. TEST SUITE ORGANIZATION
// =============================================================================

console.log('2. TEST SUITE:\n');

class TestRunner {
    constructor() {
        this.tests = [];
        this.results = { passed: 0, failed: 0 };
    }
    
    describe(suiteName, fn) {
        console.log(`\n${suiteName}:`);
        fn();
    }
    
    it(testName, fn) {
        try {
            fn();
            this.results.passed++;
            console.log(`  ✓ ${testName}`);
        } catch (error) {
            this.results.failed++;
            console.log(`  ✗ ${testName}`);
            console.log(`    ${error.message}`);
        }
    }
    
    expect(actual) {
        return {
            toBe: (expected) => {
                if (actual !== expected) {
                    throw new Error(`Expected ${expected}, got ${actual}`);
                }
            },
            toEqual: (expected) => {
                if (JSON.stringify(actual) !== JSON.stringify(expected)) {
                    throw new Error(`Expected ${JSON.stringify(expected)}, got ${JSON.stringify(actual)}`);
                }
            },
            toBeGreaterThan: (expected) => {
                if (actual <= expected) {
                    throw new Error(`Expected ${actual} to be greater than ${expected}`);
                }
            },
            toBeLessThan: (expected) => {
                if (actual >= expected) {
                    throw new Error(`Expected ${actual} to be less than ${expected}`);
                }
            },
            toThrow: () => {
                try {
                    actual();
                    throw new Error('Expected function to throw');
                } catch (error) {
                    // Expected
                }
            }
        };
    }
    
    printResults() {
        console.log('\n' + '='.repeat(70));
        console.log('\nTest Results:');
        console.log(`  Passed: ${this.results.passed}`);
        console.log(`  Failed: ${this.results.failed}`);
        console.log(`  Total: ${this.results.passed + this.results.failed}`);
        
        const percentage = (this.results.passed / (this.results.passed + this.results.failed) * 100).toFixed(2);
        console.log(`  Success Rate: ${percentage}%\n`);
    }
}

const test = new TestRunner();

test.describe('BankAccount Tests', () => {
    test.it('should create account with initial balance', () => {
        const acc = new BankAccount('ACC-001', 1000);
        test.expect(acc.balance).toBe(1000);
    });
    
    test.it('should deposit money correctly', () => {
        const acc = new BankAccount('ACC-002', 1000);
        acc.deposit(500);
        test.expect(acc.balance).toBe(1500);
    });
    
    test.it('should withdraw money correctly', () => {
        const acc = new BankAccount('ACC-003', 1000);
        acc.withdraw(300);
        test.expect(acc.balance).toBe(700);
    });
    
    test.it('should throw error on insufficient funds', () => {
        const acc = new BankAccount('ACC-004', 100);
        test.expect(() => acc.withdraw(200)).toThrow();
    });
});

test.describe('Interest Calculation Tests', () => {
    function calculateInterest(balance, rate, years) {
        return balance * Math.pow(1 + rate, years) - balance;
    }
    
    test.it('should calculate simple interest correctly', () => {
        const interest = calculateInterest(100000, 0.035, 1);
        test.expect(interest).toBeGreaterThan(3400);
        test.expect(interest).toBeLessThan(3600);
    });
    
    test.it('should handle zero balance', () => {
        const interest = calculateInterest(0, 0.035, 1);
        test.expect(interest).toBe(0);
    });
});

test.printResults();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. MOCKING AND STUBBING
// =============================================================================

console.log('3. MOCKING:\n');

// Database service (to be mocked)
class DatabaseService {
    async getAccount(accountNumber) {
        // Real implementation would query database
        throw new Error('Database not available in test');
    }
    
    async saveTransaction(transaction) {
        throw new Error('Database not available in test');
    }
}

// Mock implementation
class MockDatabaseService {
    constructor() {
        this.accounts = new Map([
            ['ACC-001', { accountNumber: 'ACC-001', balance: 50000 }],
            ['ACC-002', { accountNumber: 'ACC-002', balance: 30000 }]
        ]);
        this.transactions = [];
        this.calls = {
            getAccount: [],
            saveTransaction: []
        };
    }
    
    async getAccount(accountNumber) {
        this.calls.getAccount.push(accountNumber);
        const account = this.accounts.get(accountNumber);
        if (!account) {
            throw new Error('Account not found');
        }
        return account;
    }
    
    async saveTransaction(transaction) {
        this.calls.saveTransaction.push(transaction);
        this.transactions.push(transaction);
        return { id: `TXN-${this.transactions.length}`, ...transaction };
    }
    
    // Test helper methods
    getCalls(method) {
        return this.calls[method] || [];
    }
    
    reset() {
        this.calls = { getAccount: [], saveTransaction: [] };
        this.transactions = [];
    }
}

// Service using database
class TransactionService {
    constructor(database) {
        this.database = database;
    }
    
    async transfer(fromAccount, toAccount, amount) {
        const from = await this.database.getAccount(fromAccount);
        const to = await this.database.getAccount(toAccount);
        
        if (from.balance < amount) {
            throw new Error('Insufficient funds');
        }
        
        from.balance -= amount;
        to.balance += amount;
        
        await this.database.saveTransaction({
            type: 'TRANSFER',
            from: fromAccount,
            to: toAccount,
            amount
        });
        
        return { success: true };
    }
}

// Test with mock
(async () => {
    console.log('Testing with mock database:\n');
    
    const mockDb = new MockDatabaseService();
    const service = new TransactionService(mockDb);
    
    await service.transfer('ACC-001', 'ACC-002', 5000);
    
    console.log('  ✓ Transfer completed');
    console.log(`  Database calls: ${mockDb.getCalls('getAccount').length} reads, ${mockDb.getCalls('saveTransaction').length} writes`);
    console.log(`  Transactions saved: ${mockDb.transactions.length}\n`);
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 4. INTEGRATION TESTING
    // =============================================================================
    
    console.log('4. INTEGRATION TESTING:\n');
    
    // Multiple components working together
    class AccountRepository {
        constructor() {
            this.accounts = new Map();
        }
        
        save(account) {
            this.accounts.set(account.accountNumber, account);
        }
        
        find(accountNumber) {
            return this.accounts.get(accountNumber);
        }
    }
    
    class TransactionRepository {
        constructor() {
            this.transactions = [];
        }
        
        save(transaction) {
            this.transactions.push(transaction);
        }
        
        findByAccount(accountNumber) {
            return this.transactions.filter(
                t => t.from === accountNumber || t.to === accountNumber
            );
        }
    }
    
    class BankingSystem {
        constructor(accountRepo, transactionRepo) {
            this.accountRepo = accountRepo;
            this.transactionRepo = transactionRepo;
        }
        
        createAccount(accountNumber, initialBalance) {
            const account = { accountNumber, balance: initialBalance };
            this.accountRepo.save(account);
            return account;
        }
        
        transfer(from, to, amount) {
            const fromAccount = this.accountRepo.find(from);
            const toAccount = this.accountRepo.find(to);
            
            if (!fromAccount || !toAccount) {
                throw new Error('Account not found');
            }
            
            if (fromAccount.balance < amount) {
                throw new Error('Insufficient funds');
            }
            
            fromAccount.balance -= amount;
            toAccount.balance += amount;
            
            const transaction = { from, to, amount, timestamp: new Date() };
            this.transactionRepo.save(transaction);
            
            return transaction;
        }
        
        getAccountStatement(accountNumber) {
            const account = this.accountRepo.find(accountNumber);
            const transactions = this.transactionRepo.findByAccount(accountNumber);
            
            return { account, transactions };
        }
    }
    
    console.log('Integration test - Complete banking flow:\n');
    
    const accountRepo = new AccountRepository();
    const transactionRepo = new TransactionRepository();
    const banking = new BankingSystem(accountRepo, transactionRepo);
    
    // Create accounts
    console.log('  Creating accounts...');
    banking.createAccount('ACC-111', 50000);
    banking.createAccount('ACC-222', 30000);
    console.log('  ✓ Accounts created\n');
    
    // Transfer money
    console.log('  Transferring 5000 AED...');
    banking.transfer('ACC-111', 'ACC-222', 5000);
    console.log('  ✓ Transfer completed\n');
    
    // Verify balances
    console.log('  Verifying balances...');
    const acc1 = accountRepo.find('ACC-111');
    const acc2 = accountRepo.find('ACC-222');
    console.log(`  ACC-111: ${acc1.balance} AED`);
    console.log(`  ACC-222: ${acc2.balance} AED`);
    Assert.assertEqual(acc1.balance, 45000, '  Source account balance');
    Assert.assertEqual(acc2.balance, 35000, '  Destination account balance');
    console.log();
    
    // Verify transaction history
    console.log('  Verifying transaction history...');
    const statement = banking.getAccountStatement('ACC-111');
    console.log(`  Transactions: ${statement.transactions.length}`);
    Assert.assertEqual(statement.transactions.length, 1, '  Transaction count');
    console.log();
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 5. PERFORMANCE TESTING
    // =============================================================================
    
    console.log('5. PERFORMANCE TESTING:\n');
    
    class PerformanceTest {
        static measure(name, fn) {
            const start = performance.now();
            const result = fn();
            const end = performance.now();
            const duration = (end - start).toFixed(2);
            
            console.log(`  ${name}: ${duration}ms`);
            return { result, duration };
        }
        
        static async measureAsync(name, fn) {
            const start = performance.now();
            const result = await fn();
            const end = performance.now();
            const duration = (end - start).toFixed(2);
            
            console.log(`  ${name}: ${duration}ms`);
            return { result, duration };
        }
        
        static benchmark(name, fn, iterations = 1000) {
            const start = performance.now();
            
            for (let i = 0; i < iterations; i++) {
                fn();
            }
            
            const end = performance.now();
            const total = (end - start).toFixed(2);
            const average = ((end - start) / iterations).toFixed(4);
            
            console.log(`  ${name}:`);
            console.log(`    Total: ${total}ms`);
            console.log(`    Average: ${average}ms`);
            console.log(`    Ops/sec: ${(1000 / parseFloat(average)).toFixed(0)}`);
        }
    }
    
    console.log('Performance benchmarks:\n');
    
    // Test array operations
    console.log('Array operations:');
    PerformanceTest.benchmark('Array push', () => {
        const arr = [];
        arr.push(1);
    });
    
    PerformanceTest.benchmark('Array unshift', () => {
        const arr = [];
        arr.unshift(1);
    });
    console.log();
    
    // Test Map vs Object
    console.log('Map vs Object lookup:');
    const map = new Map();
    const obj = {};
    for (let i = 0; i < 1000; i++) {
        map.set(`key${i}`, i);
        obj[`key${i}`] = i;
    }
    
    PerformanceTest.benchmark('Map.get()', () => {
        map.get('key500');
    }, 10000);
    
    PerformanceTest.benchmark('Object property access', () => {
        obj['key500'];
    }, 10000);
    console.log();
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // TESTING SUMMARY
    // =============================================================================
    
    console.log('TESTING SUMMARY:\n');
    
    console.log('Testing Pyramid:');
    console.log('  1. Unit Tests (70%) - Fast, isolated');
    console.log('  2. Integration Tests (20%) - Component interactions');
    console.log('  3. E2E Tests (10%) - Complete user flows\n');
    
    console.log('Test Characteristics (F.I.R.S.T):');
    console.log('  • Fast - Run quickly');
    console.log('  • Independent - No dependencies between tests');
    console.log('  • Repeatable - Same results every time');
    console.log('  • Self-validating - Pass or fail, no manual check');
    console.log('  • Timely - Written with or before code\n');
    
    console.log('Testing Tools:');
    console.log('  • Jest - Unit & integration testing');
    console.log('  • Mocha - Test framework');
    console.log('  • Chai - Assertion library');
    console.log('  • Sinon - Mocking & stubbing');
    console.log('  • Cypress - E2E testing');
    console.log('  • Testing Library - UI component testing\n');
    
    console.log('Best Practices:');
    console.log('  ✓ Write tests first (TDD)');
    console.log('  ✓ Test one thing per test');
    console.log('  ✓ Use descriptive test names');
    console.log('  ✓ Keep tests simple and readable');
    console.log('  ✓ Mock external dependencies');
    console.log('  ✓ Test edge cases and errors');
    console.log('  ✓ Maintain test coverage >80%');
    console.log('  ✓ Run tests in CI/CD pipeline');
    
    console.log();
})();
```

### Key Takeaways:

**Generators & Iterators**:
- function* for generators, yield to pause
- Iterator protocol: next() returns {value, done}
- async function* for async iteration
- Perfect for lazy evaluation and streams
- Use for infinite sequences, pagination

**Error Handling**:
- Use custom error classes with codes
- Try...catch for synchronous errors
- Promise.catch() for async errors
- Implement retry and fallback strategies
- Log errors with context
- Validate input early

**Testing**:
- Unit tests for individual functions
- Integration tests for component interactions
- Mock dependencies for isolation
- Measure performance for optimization
- Follow F.I.R.S.T principles
- Maintain high test coverage

---

**End of Questions 46-48**
