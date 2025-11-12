# Questions 13-15: Advanced Async Patterns & Generators

## Question 13: What are Generators and Iterators? How do they work in JavaScript?

### Answer:

**Generators** are special functions that can pause execution and resume later, yielding multiple values over time. They're defined using `function*` syntax and use the `yield` keyword.

**Iterators** are objects that implement the iterator protocol with a `next()` method that returns `{value, done}`.

### Key Concepts:
1. **Generator Functions**: Use `function*` syntax
2. **yield Keyword**: Pauses execution and returns a value
3. **Iterator Protocol**: Objects with `next()` method
4. **Symbol.iterator**: Makes objects iterable
5. **for...of Loop**: Automatically works with iterables

### Banking Scenario: Transaction Batch Processing at Emirates NBD

```javascript
// Generator for processing large transaction batches
class TransactionBatchProcessor {
    constructor(transactions) {
        this.transactions = transactions;
    }

    // Generator function to process transactions in batches
    *processBatch(batchSize = 100) {
        let processedCount = 0;
        let batch = [];

        for (let i = 0; i < this.transactions.length; i++) {
            batch.push(this.transactions[i]);

            // When batch is full or it's the last transaction
            if (batch.length === batchSize || i === this.transactions.length - 1) {
                const batchResult = {
                    batchNumber: Math.floor(processedCount / batchSize) + 1,
                    transactions: batch,
                    totalAmount: batch.reduce((sum, t) => sum + t.amount, 0),
                    processedCount: processedCount + batch.length,
                    timestamp: new Date().toISOString()
                };

                processedCount += batch.length;
                batch = [];
                
                yield batchResult; // Pause and return batch result
            }
        }

        return { completed: true, totalProcessed: processedCount };
    }

    // Generator for filtering transactions by criteria
    *filterTransactions(criteria) {
        for (const transaction of this.transactions) {
            if (this.matchesCriteria(transaction, criteria)) {
                yield transaction;
            }
        }
    }

    matchesCriteria(transaction, criteria) {
        return Object.keys(criteria).every(key => 
            transaction[key] === criteria[key]
        );
    }

    // Generator with infinite sequence for transaction IDs
    *generateTransactionIds(prefix = 'TXN') {
        let counter = 1;
        while (true) {
            yield `${prefix}-${Date.now()}-${counter.toString().padStart(6, '0')}`;
            counter++;
        }
    }
}

// Custom Iterator for Account Transaction History
class TransactionHistory {
    constructor(accountNumber) {
        this.accountNumber = accountNumber;
        this.transactions = [];
    }

    addTransaction(transaction) {
        this.transactions.push({
            ...transaction,
            timestamp: new Date(),
            accountNumber: this.accountNumber
        });
    }

    // Make the class iterable
    [Symbol.iterator]() {
        let index = 0;
        const transactions = this.transactions;

        return {
            next() {
                if (index < transactions.length) {
                    return { value: transactions[index++], done: false };
                } else {
                    return { done: true };
                }
            }
        };
    }

    // Reverse iterator using generator
    *reverseIterator() {
        for (let i = this.transactions.length - 1; i >= 0; i--) {
            yield this.transactions[i];
        }
    }

    // Date range iterator
    *dateRangeIterator(startDate, endDate) {
        for (const transaction of this.transactions) {
            if (transaction.timestamp >= startDate && transaction.timestamp <= endDate) {
                yield transaction;
            }
        }
    }
}

// Example Usage
console.log('=== Generator and Iterator Demo - Emirates NBD ===\n');

// Sample transactions
const transactions = Array.from({ length: 250 }, (_, i) => ({
    id: `TXN-${i + 1}`,
    amount: Math.floor(Math.random() * 10000) + 100,
    type: ['DEPOSIT', 'WITHDRAWAL', 'TRANSFER'][Math.floor(Math.random() * 3)],
    status: 'PENDING'
}));

// 1. Batch Processing with Generator
console.log('1. Batch Processing:');
const processor = new TransactionBatchProcessor(transactions);
const batchGenerator = processor.processBatch(100);

// Process first 2 batches
const batch1 = batchGenerator.next();
console.log(`Batch ${batch1.value.batchNumber}: Processed ${batch1.value.transactions.length} transactions`);
console.log(`Total Amount: AED ${batch1.value.totalAmount.toLocaleString()}\n`);

const batch2 = batchGenerator.next();
console.log(`Batch ${batch2.value.batchNumber}: Processed ${batch2.value.transactions.length} transactions`);
console.log(`Total Amount: AED ${batch2.value.totalAmount.toLocaleString()}\n`);

// 2. Filter transactions using generator
console.log('2. Filtering Transactions:');
const filterGen = processor.filterTransactions({ type: 'DEPOSIT' });
let depositCount = 0;
for (const deposit of filterGen) {
    depositCount++;
    if (depositCount <= 3) {
        console.log(`- ${deposit.id}: AED ${deposit.amount} (${deposit.type})`);
    }
}
console.log(`Total Deposits: ${depositCount}\n`);

// 3. Transaction ID Generator (infinite sequence)
console.log('3. Transaction ID Generator:');
const idGenerator = processor.generateTransactionIds('EMR-NBD');
console.log('Generated IDs:');
for (let i = 0; i < 5; i++) {
    console.log(`- ${idGenerator.next().value}`);
}
console.log();

// 4. Custom Iterator
console.log('4. Transaction History Iterator:');
const history = new TransactionHistory('ACC-123456789');

// Add transactions
history.addTransaction({ type: 'DEPOSIT', amount: 5000, description: 'Salary Credit' });
history.addTransaction({ type: 'WITHDRAWAL', amount: 1000, description: 'ATM Withdrawal' });
history.addTransaction({ type: 'TRANSFER', amount: 2000, description: 'Transfer to Savings' });

console.log('Forward Iteration:');
for (const txn of history) {
    console.log(`- ${txn.type}: AED ${txn.amount} - ${txn.description}`);
}

console.log('\nReverse Iteration:');
for (const txn of history.reverseIterator()) {
    console.log(`- ${txn.type}: AED ${txn.amount} - ${txn.description}`);
}

// Advanced: Async Generator for API pagination
async function* fetchTransactionPages(accountNumber, pageSize = 50) {
    let page = 1;
    let hasMore = true;

    while (hasMore) {
        // Simulate API call
        const response = await new Promise(resolve => {
            setTimeout(() => {
                const transactions = Array.from({ length: pageSize }, (_, i) => ({
                    id: `TXN-${page}-${i + 1}`,
                    amount: Math.floor(Math.random() * 5000),
                    type: 'TRANSFER'
                }));

                resolve({
                    data: transactions,
                    page: page,
                    hasMore: page < 3 // Only 3 pages for demo
                });
            }, 100);
        });

        yield response;
        hasMore = response.hasMore;
        page++;
    }
}

// Using async generator
(async () => {
    console.log('\n5. Async Generator for API Pagination:');
    let totalTransactions = 0;

    for await (const pageData of fetchTransactionPages('ACC-123456789')) {
        totalTransactions += pageData.data.length;
        console.log(`Page ${pageData.page}: Loaded ${pageData.data.length} transactions`);
    }

    console.log(`Total transactions loaded: ${totalTransactions}`);
})();
```

### Key Takeaways:
- **Generators** enable lazy evaluation and memory-efficient iteration
- **yield** pauses function execution and can be resumed
- **Iterators** provide custom iteration behavior
- **Async Generators** combine async/await with generators for streaming data
- Perfect for processing large datasets, pagination, and infinite sequences

---

## Question 14: What is the Proxy object? How can you use it for validation and logging?

### Answer:

**Proxy** is a meta-programming feature that allows you to intercept and customize operations on objects. It creates a wrapper around an object and lets you define custom behavior for fundamental operations.

### Key Concepts:
1. **Handler**: Object with traps (methods) for operations
2. **Target**: The original object being proxied
3. **Traps**: Methods that intercept operations (get, set, deleteProperty, etc.)
4. **Reflect API**: Companion API for default behavior
5. **Revocable Proxies**: Proxies that can be disabled

### Common Traps:
- `get`: Property access
- `set`: Property assignment
- `has`: `in` operator
- `deleteProperty`: `delete` operator
- `apply`: Function calls
- `construct`: `new` operator

### Banking Scenario: Secure Account Operations at Emirates NBD

```javascript
// Validation and logging with Proxy
class BankAccount {
    constructor(accountNumber, initialBalance = 0) {
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
        this.transactions = [];
        this.metadata = {
            createdAt: new Date(),
            lastModified: new Date()
        };
    }

    deposit(amount) {
        this.balance += amount;
        this.transactions.push({ type: 'DEPOSIT', amount, timestamp: new Date() });
    }

    withdraw(amount) {
        if (this.balance >= amount) {
            this.balance -= amount;
            this.transactions.push({ type: 'WITHDRAWAL', amount, timestamp: new Date() });
            return true;
        }
        return false;
    }
}

// Audit Logger
class AuditLogger {
    constructor() {
        this.logs = [];
    }

    log(action, details) {
        const logEntry = {
            timestamp: new Date().toISOString(),
            action,
            details,
            userId: 'current-user-id' // Would come from auth context
        };
        this.logs.push(logEntry);
        console.log(`[AUDIT] ${action}:`, details);
    }

    getAuditTrail() {
        return this.logs;
    }
}

// Create Secure Account Proxy with validation and logging
function createSecureAccountProxy(account, auditLogger) {
    const validators = {
        balance: (value) => {
            if (typeof value !== 'number') {
                throw new TypeError('Balance must be a number');
            }
            if (value < 0) {
                throw new RangeError('Balance cannot be negative');
            }
            return true;
        },
        accountNumber: (value) => {
            if (typeof value !== 'string' || !/^ACC-\d{9}$/.test(value)) {
                throw new Error('Invalid account number format. Must be ACC-XXXXXXXXX');
            }
            return true;
        }
    };

    const handler = {
        // Intercept property access (get)
        get(target, property, receiver) {
            // Log sensitive property access
            if (['balance', 'transactions'].includes(property)) {
                auditLogger.log('PROPERTY_ACCESS', {
                    property,
                    accountNumber: target.accountNumber
                });
            }

            const value = Reflect.get(target, property, receiver);

            // If it's a method, wrap it to log calls
            if (typeof value === 'function') {
                return function (...args) {
                    auditLogger.log('METHOD_CALL', {
                        method: property,
                        arguments: args,
                        accountNumber: target.accountNumber
                    });

                    const result = value.apply(target, args);

                    auditLogger.log('METHOD_RESULT', {
                        method: property,
                        result,
                        accountNumber: target.accountNumber
                    });

                    return result;
                };
            }

            return value;
        },

        // Intercept property assignment (set)
        set(target, property, value, receiver) {
            // Validate before setting
            if (validators[property]) {
                validators[property](value);
            }

            // Prevent direct balance modification
            if (property === 'balance') {
                auditLogger.log('SECURITY_VIOLATION', {
                    action: 'DIRECT_BALANCE_MODIFICATION_BLOCKED',
                    attemptedValue: value,
                    accountNumber: target.accountNumber
                });
                throw new Error('Direct balance modification not allowed. Use deposit() or withdraw()');
            }

            // Log the change
            const oldValue = target[property];
            auditLogger.log('PROPERTY_SET', {
                property,
                oldValue,
                newValue: value,
                accountNumber: target.accountNumber
            });

            // Update metadata
            target.metadata.lastModified = new Date();

            return Reflect.set(target, property, value, receiver);
        },

        // Intercept property deletion
        deleteProperty(target, property) {
            if (['accountNumber', 'balance', 'transactions'].includes(property)) {
                auditLogger.log('SECURITY_VIOLATION', {
                    action: 'CRITICAL_PROPERTY_DELETE_BLOCKED',
                    property,
                    accountNumber: target.accountNumber
                });
                throw new Error(`Cannot delete critical property: ${property}`);
            }

            auditLogger.log('PROPERTY_DELETE', {
                property,
                accountNumber: target.accountNumber
            });

            return Reflect.deleteProperty(target, property);
        },

        // Intercept 'in' operator
        has(target, property) {
            // Don't expose internal properties
            if (property.startsWith('_')) {
                return false;
            }
            return Reflect.has(target, property);
        }
    };

    return new Proxy(account, handler);
}

// Rate Limiting Proxy for API calls
function createRateLimitedAPI(apiService, maxCalls = 5, timeWindow = 60000) {
    const callTimestamps = [];

    return new Proxy(apiService, {
        get(target, property) {
            const value = target[property];

            if (typeof value === 'function') {
                return function (...args) {
                    const now = Date.now();
                    
                    // Remove old timestamps outside time window
                    while (callTimestamps.length > 0 && callTimestamps[0] < now - timeWindow) {
                        callTimestamps.shift();
                    }

                    // Check rate limit
                    if (callTimestamps.length >= maxCalls) {
                        const oldestCall = callTimestamps[0];
                        const waitTime = Math.ceil((oldestCall + timeWindow - now) / 1000);
                        throw new Error(
                            `Rate limit exceeded. Please wait ${waitTime} seconds before trying again.`
                        );
                    }

                    // Record this call
                    callTimestamps.push(now);

                    console.log(`[RATE LIMIT] ${property}: ${callTimestamps.length}/${maxCalls} calls used`);

                    return value.apply(target, args);
                };
            }

            return value;
        }
    });
}

// Validation Proxy for user input
function createValidatedObject(schema) {
    return new Proxy({}, {
        set(target, property, value) {
            const validator = schema[property];

            if (!validator) {
                throw new Error(`Property '${property}' is not defined in schema`);
            }

            // Validate value
            if (!validator.validate(value)) {
                throw new Error(`Validation failed for '${property}': ${validator.message}`);
            }

            return Reflect.set(target, property, value);
        }
    });
}

// Example Usage
console.log('=== Proxy Demo - Emirates NBD ===\n');

// 1. Secure Account Proxy
console.log('1. Secure Account with Proxy:\n');
const auditLogger = new AuditLogger();
const rawAccount = new BankAccount('ACC-123456789', 5000);
const secureAccount = createSecureAccountProxy(rawAccount, auditLogger);

// Access balance (logged)
console.log(`\nCurrent Balance: AED ${secureAccount.balance}\n`);

// Call methods (logged)
secureAccount.deposit(2000);
console.log(`Balance after deposit: AED ${secureAccount.balance}\n`);

secureAccount.withdraw(1000);
console.log(`Balance after withdrawal: AED ${secureAccount.balance}\n`);

// Try to directly modify balance (blocked)
try {
    secureAccount.balance = 999999;
} catch (error) {
    console.log(`Error caught: ${error.message}\n`);
}

// Try to delete critical property (blocked)
try {
    delete secureAccount.accountNumber;
} catch (error) {
    console.log(`Error caught: ${error.message}\n`);
}

// 2. Rate Limited API Proxy
console.log('\n2. Rate Limited API:');
const bankingAPI = {
    getAccountBalance(accountNumber) {
        return { accountNumber, balance: 5000 };
    },
    getTransactionHistory(accountNumber) {
        return { accountNumber, transactions: [] };
    }
};

const rateLimitedAPI = createRateLimitedAPI(bankingAPI, 3, 5000); // 3 calls per 5 seconds

try {
    for (let i = 1; i <= 5; i++) {
        console.log(`\nAPI Call ${i}:`);
        rateLimitedAPI.getAccountBalance('ACC-123456789');
    }
} catch (error) {
    console.log(`\nRate Limit Error: ${error.message}`);
}

// 3. Validated Object Proxy
console.log('\n\n3. Validated Transfer Request:');
const transferSchema = {
    fromAccount: {
        validate: (v) => typeof v === 'string' && /^ACC-\d{9}$/.test(v),
        message: 'Must be valid account number format'
    },
    toAccount: {
        validate: (v) => typeof v === 'string' && /^ACC-\d{9}$/.test(v),
        message: 'Must be valid account number format'
    },
    amount: {
        validate: (v) => typeof v === 'number' && v > 0 && v <= 100000,
        message: 'Amount must be between 0 and 100,000'
    },
    currency: {
        validate: (v) => ['AED', 'USD', 'EUR', 'GBP'].includes(v),
        message: 'Currency must be AED, USD, EUR, or GBP'
    }
};

const transferRequest = createValidatedObject(transferSchema);

try {
    transferRequest.fromAccount = 'ACC-123456789';
    transferRequest.toAccount = 'ACC-987654321';
    transferRequest.amount = 5000;
    transferRequest.currency = 'AED';
    
    console.log('Valid transfer request created:', {
        fromAccount: transferRequest.fromAccount,
        toAccount: transferRequest.toAccount,
        amount: transferRequest.amount,
        currency: transferRequest.currency
    });
} catch (error) {
    console.log(`Validation Error: ${error.message}`);
}

// Try invalid amount
try {
    console.log('\nTrying invalid amount:');
    transferRequest.amount = -100;
} catch (error) {
    console.log(`Validation Error: ${error.message}`);
}

// Try invalid property
try {
    console.log('\nTrying undefined property:');
    transferRequest.invalidField = 'test';
} catch (error) {
    console.log(`Validation Error: ${error.message}`);
}

// Revocable Proxy Example
console.log('\n\n4. Revocable Proxy for Session Management:');
const sessionData = { userId: 'user123', accountNumber: 'ACC-123456789' };
const { proxy: sessionProxy, revoke } = Proxy.revocable(sessionData, {
    get(target, property) {
        console.log(`Accessing session property: ${property}`);
        return target[property];
    }
});

console.log('Session active:', sessionProxy.userId);
console.log('Revoking session...');
revoke();

try {
    console.log('Trying to access after revoke:', sessionProxy.userId);
} catch (error) {
    console.log('Error: Session has been revoked (proxy is no longer accessible)');
}

// Show audit trail
console.log('\n\n5. Audit Trail Summary:');
const auditTrail = auditLogger.getAuditTrail();
console.log(`Total audit entries: ${auditTrail.length}`);
console.log('\nRecent entries:');
auditTrail.slice(-5).forEach(entry => {
    console.log(`- ${entry.action} at ${entry.timestamp}`);
});
```

### Key Takeaways:
- **Proxy** enables meta-programming and custom object behavior
- Perfect for **validation**, **logging**, and **security**
- **Traps** intercept fundamental operations
- **Reflect API** provides default behavior for traps
- Use for rate limiting, audit trails, and access control
- **Revocable proxies** useful for session management

---

## Question 15: What is the Event Loop? Explain microtasks vs macrotasks.

### Answer:

The **Event Loop** is JavaScript's concurrency model that handles asynchronous operations. It continuously checks the call stack and task queues, executing code in a specific order.

### Key Concepts:

1. **Call Stack**: Where function execution contexts are stored (LIFO)
2. **Heap**: Memory allocation for objects
3. **Task Queues**: Where callbacks wait to be executed
   - **Macrotask Queue** (Task Queue): setTimeout, setInterval, I/O operations
   - **Microtask Queue** (Job Queue): Promises, queueMicrotask, MutationObserver
4. **Event Loop**: Coordinates execution between stack and queues

### Execution Order:
1. Execute all synchronous code (call stack)
2. Execute all microtasks (promise callbacks, queueMicrotask)
3. Render (in browsers)
4. Execute one macrotask (setTimeout, setInterval)
5. Repeat from step 2

### Banking Scenario: Transaction Processing Priority at Emirates NBD

```javascript
// Priority-based transaction processing system
class TransactionPriorityProcessor {
    constructor() {
        this.transactions = [];
        this.processedCount = 0;
    }

    // Immediate priority - executes synchronously
    processImmediatePriority(transaction) {
        console.log(`[IMMEDIATE] Processing ${transaction.type}: ${transaction.id}`);
        this.processedCount++;
        return {
            ...transaction,
            status: 'COMPLETED',
            processedAt: Date.now(),
            priority: 'IMMEDIATE'
        };
    }

    // High priority - uses microtask (Promise)
    processHighPriority(transaction) {
        return Promise.resolve().then(() => {
            console.log(`[HIGH-MICROTASK] Processing ${transaction.type}: ${transaction.id}`);
            this.processedCount++;
            return {
                ...transaction,
                status: 'COMPLETED',
                processedAt: Date.now(),
                priority: 'HIGH'
            };
        });
    }

    // Medium priority - uses queueMicrotask
    processMediumPriority(transaction) {
        return new Promise((resolve) => {
            queueMicrotask(() => {
                console.log(`[MEDIUM-MICROTASK] Processing ${transaction.type}: ${transaction.id}`);
                this.processedCount++;
                resolve({
                    ...transaction,
                    status: 'COMPLETED',
                    processedAt: Date.now(),
                    priority: 'MEDIUM'
                });
            });
        });
    }

    // Low priority - uses macrotask (setTimeout)
    processLowPriority(transaction) {
        return new Promise((resolve) => {
            setTimeout(() => {
                console.log(`[LOW-MACROTASK] Processing ${transaction.type}: ${transaction.id}`);
                this.processedCount++;
                resolve({
                    ...transaction,
                    status: 'COMPLETED',
                    processedAt: Date.now(),
                    priority: 'LOW'
                });
            }, 0);
        });
    }

    // Background priority - uses setImmediate-like behavior
    processBackgroundPriority(transaction) {
        return new Promise((resolve) => {
            setTimeout(() => {
                console.log(`[BACKGROUND-MACROTASK] Processing ${transaction.type}: ${transaction.id}`);
                this.processedCount++;
                resolve({
                    ...transaction,
                    status: 'COMPLETED',
                    processedAt: Date.now(),
                    priority: 'BACKGROUND'
                });
            }, 10);
        });
    }
}

// Demonstrate Event Loop behavior
async function demonstrateEventLoop() {
    console.log('=== Event Loop Demo - Emirates NBD Transaction Processing ===\n');
    
    console.log('1️⃣ START - Call Stack Execution\n');

    // Synchronous code
    console.log('2️⃣ Synchronous: Validating user session...');
    console.log('3️⃣ Synchronous: Checking account permissions...\n');

    // Macrotask (setTimeout)
    setTimeout(() => {
        console.log('🔴 MACROTASK 1: Processing low-priority batch report');
    }, 0);

    setTimeout(() => {
        console.log('🔴 MACROTASK 2: Sending daily summary email');
    }, 0);

    // Microtask (Promise)
    Promise.resolve().then(() => {
        console.log('🟢 MICROTASK 1: Validating transaction signature');
    });

    Promise.resolve().then(() => {
        console.log('🟢 MICROTASK 2: Updating account cache');
    });

    // Another Promise chain
    Promise.resolve()
        .then(() => {
            console.log('🟢 MICROTASK 3: Logging audit trail');
            return Promise.resolve();
        })
        .then(() => {
            console.log('🟢 MICROTASK 4: Checking fraud detection');
        });

    // queueMicrotask (also microtask)
    queueMicrotask(() => {
        console.log('🟢 MICROTASK 5: Notifying real-time dashboard');
    });

    // More synchronous code
    console.log('4️⃣ Synchronous: Transaction submitted to queue\n');

    // Nested setTimeout
    setTimeout(() => {
        console.log('🔴 MACROTASK 3: Processing scheduled backup');
        
        Promise.resolve().then(() => {
            console.log('🟢 MICROTASK 6: Backup validation (after macrotask 3)');
        });
    }, 0);

    console.log('5️⃣ Synchronous: Returning confirmation to client\n');
    console.log('6️⃣ END - Call Stack Empty, Event Loop Takes Over\n');
}

// Real-world banking transaction processor
class BankingTransactionQueue {
    constructor() {
        this.microQueue = [];
        this.macroQueue = [];
        this.isProcessing = false;
    }

    // Add transaction to appropriate queue
    enqueueTransaction(transaction, priority = 'LOW') {
        const txn = {
            ...transaction,
            id: `TXN-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
            enqueuedAt: Date.now(),
            priority
        };

        if (priority === 'HIGH' || priority === 'CRITICAL') {
            this.microQueue.push(txn);
            console.log(`✅ Enqueued to MICROTASK queue: ${txn.id} (${priority})`);
        } else {
            this.macroQueue.push(txn);
            console.log(`✅ Enqueued to MACROTASK queue: ${txn.id} (${priority})`);
        }

        this.processQueue();
    }

    // Process queues respecting event loop priorities
    processQueue() {
        if (this.isProcessing) return;
        this.isProcessing = true;

        // Process all microtasks first (high priority)
        while (this.microQueue.length > 0) {
            const txn = this.microQueue.shift();
            Promise.resolve().then(() => {
                console.log(`⚡ Processing CRITICAL transaction: ${txn.id} - ${txn.type}`);
            });
        }

        // Then process one macrotask
        if (this.macroQueue.length > 0) {
            const txn = this.macroQueue.shift();
            setTimeout(() => {
                console.log(`📦 Processing STANDARD transaction: ${txn.id} - ${txn.type}`);
                this.isProcessing = false;
                
                // Continue processing if more tasks remain
                if (this.macroQueue.length > 0 || this.microQueue.length > 0) {
                    this.processQueue();
                }
            }, 0);
        } else {
            this.isProcessing = false;
        }
    }
}

// Complex async operation with event loop awareness
class AsyncTransactionProcessor {
    async processTransaction(transaction) {
        console.log(`\n📋 Starting transaction ${transaction.id}...`);

        // Step 1: Immediate validation (synchronous)
        console.log('  1. [SYNC] Validating transaction format...');
        
        // Step 2: Check fraud (microtask - Promise)
        await Promise.resolve().then(() => {
            console.log('  2. [MICROTASK] Running fraud detection...');
        });

        // Step 3: Check balance (simulated async DB call)
        await new Promise(resolve => {
            setTimeout(() => {
                console.log('  3. [MACROTASK] Checking account balance...');
                resolve();
            }, 0);
        });

        // Step 4: Update ledger (microtask)
        await Promise.resolve().then(() => {
            console.log('  4. [MICROTASK] Updating transaction ledger...');
        });

        // Step 5: Send notification (macrotask)
        await new Promise(resolve => {
            setTimeout(() => {
                console.log('  5. [MACROTASK] Sending notification...');
                resolve();
            }, 0);
        });

        console.log(`  ✅ Transaction ${transaction.id} completed!\n`);
    }
}

// Execute demonstrations
(async () => {
    // Demo 1: Basic Event Loop
    await demonstrateEventLoop();

    // Wait for event loop to settle
    await new Promise(resolve => setTimeout(resolve, 100));

    console.log('\n' + '='.repeat(70) + '\n');

    // Demo 2: Priority-based Processing
    console.log('=== Priority-Based Transaction Processing ===\n');
    
    const processor = new TransactionPriorityProcessor();
    
    const transaction = {
        id: 'TXN-001',
        type: 'TRANSFER',
        amount: 5000,
        from: 'ACC-123',
        to: 'ACC-456'
    };

    console.log('Submitting transactions with different priorities:\n');

    // These will execute in priority order (not submission order)
    processor.processLowPriority({ ...transaction, id: 'TXN-LOW' });
    processor.processHighPriority({ ...transaction, id: 'TXN-HIGH-1' });
    processor.processImmediatePriority({ ...transaction, id: 'TXN-IMMEDIATE' });
    processor.processMediumPriority({ ...transaction, id: 'TXN-MEDIUM' });
    processor.processHighPriority({ ...transaction, id: 'TXN-HIGH-2' });
    processor.processBackgroundPriority({ ...transaction, id: 'TXN-BACKGROUND' });

    // Wait for all to complete
    await new Promise(resolve => setTimeout(resolve, 50));

    console.log('\n' + '='.repeat(70) + '\n');

    // Demo 3: Transaction Queue
    console.log('=== Banking Transaction Queue System ===\n');
    
    const queue = new BankingTransactionQueue();
    
    queue.enqueueTransaction({ type: 'TRANSFER', amount: 1000 }, 'LOW');
    queue.enqueueTransaction({ type: 'SALARY_CREDIT', amount: 5000 }, 'CRITICAL');
    queue.enqueueTransaction({ type: 'BILL_PAYMENT', amount: 200 }, 'LOW');
    queue.enqueueTransaction({ type: 'FRAUD_ALERT', amount: 0 }, 'HIGH');
    queue.enqueueTransaction({ type: 'WITHDRAWAL', amount: 500 }, 'LOW');

    await new Promise(resolve => setTimeout(resolve, 100));

    console.log('\n' + '='.repeat(70) + '\n');

    // Demo 4: Complex Async Processing
    console.log('=== Complex Async Transaction Processing ===\n');
    
    const asyncProcessor = new AsyncTransactionProcessor();
    await asyncProcessor.processTransaction({ id: 'TXN-ASYNC-001', type: 'TRANSFER', amount: 3000 });

    console.log('\n' + '='.repeat(70) + '\n');

    // Demo 5: Event Loop Visualization
    console.log('=== Event Loop Execution Order ===\n');
    console.log('Order of Execution:');
    console.log('1. All synchronous code (call stack)');
    console.log('2. All microtasks (Promises, queueMicrotask)');
    console.log('3. One macrotask (setTimeout, setInterval)');
    console.log('4. Repeat from step 2\n');

    console.log('Priority: IMMEDIATE > MICROTASKS > MACROTASKS\n');

    console.log('Example:');
    console.log('console.log("A")              // 1st - synchronous');
    console.log('setTimeout(() => log("B"), 0) // 4th - macrotask');
    console.log('Promise.resolve().then(log("C")) // 3rd - microtask');
    console.log('console.log("D")              // 2nd - synchronous');
    console.log('\nOutput: A, D, C, B\n');

})();

// Practical Example: Rate limiting with event loop awareness
class RateLimitedBankingAPI {
    constructor(requestsPerSecond = 10) {
        this.requestsPerSecond = requestsPerSecond;
        this.requestQueue = [];
        this.processingQueue = false;
    }

    async makeRequest(endpoint, data) {
        return new Promise((resolve, reject) => {
            this.requestQueue.push({ endpoint, data, resolve, reject });
            this.processQueue();
        });
    }

    processQueue() {
        if (this.processingQueue || this.requestQueue.length === 0) {
            return;
        }

        this.processingQueue = true;
        const delay = 1000 / this.requestsPerSecond;

        const processNext = () => {
            if (this.requestQueue.length === 0) {
                this.processingQueue = false;
                return;
            }

            const request = this.requestQueue.shift();
            
            // Use microtask for immediate processing
            Promise.resolve().then(() => {
                console.log(`Processing API request: ${request.endpoint}`);
                request.resolve({ success: true, data: request.data });
            });

            // Schedule next request as macrotask
            setTimeout(processNext, delay);
        };

        processNext();
    }
}

// Example usage
setTimeout(async () => {
    console.log('\n' + '='.repeat(70) + '\n');
    console.log('=== Rate Limited API Example ===\n');
    
    const api = new RateLimitedBankingAPI(5); // 5 requests per second
    
    const requests = [
        api.makeRequest('/balance', { account: 'ACC-001' }),
        api.makeRequest('/transactions', { account: 'ACC-001' }),
        api.makeRequest('/transfer', { from: 'ACC-001', to: 'ACC-002', amount: 100 }),
        api.makeRequest('/balance', { account: 'ACC-002' }),
        api.makeRequest('/statement', { account: 'ACC-001' })
    ];

    await Promise.all(requests);
    console.log('\nAll requests completed!');
}, 200);
```

### Key Takeaways:

1. **Event Loop Order**:
   - Synchronous code → Microtasks → Render → One Macrotask → Repeat

2. **Microtasks** (Job Queue):
   - Promise callbacks (.then, .catch, .finally)
   - queueMicrotask()
   - MutationObserver
   - Higher priority than macrotasks

3. **Macrotasks** (Task Queue):
   - setTimeout, setInterval
   - setImmediate (Node.js)
   - I/O operations
   - UI rendering events

4. **Best Practices**:
   - Use Promises for high-priority async operations
   - Use setTimeout for deferred/low-priority tasks
   - Be aware of execution order for critical operations
   - Don't block the event loop with heavy synchronous code

5. **Banking Applications**:
   - Critical transactions → Microtasks (immediate processing)
   - Batch operations → Macrotasks (deferred processing)
   - Real-time updates → Microtasks
   - Background jobs → Macrotasks with delays

---

**End of Questions 13-15**
