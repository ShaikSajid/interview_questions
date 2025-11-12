# Questions 40-42: Advanced Async Patterns, Event Loop & Performance

## Question 40: Explain the Event Loop in detail - Call Stack, Task Queue, Microtask Queue

### Answer:

The **Event Loop** is JavaScript's concurrency model that enables non-blocking asynchronous execution despite JavaScript being single-threaded. Understanding the event loop is crucial for writing efficient async code.

### Event Loop Components:
1. **Call Stack** - Function execution stack
2. **Web APIs** - Browser/Node APIs (setTimeout, fetch, etc.)
3. **Task Queue (Macrotasks)** - setTimeout, setInterval, I/O
4. **Microtask Queue** - Promises, queueMicrotask, MutationObserver
5. **Event Loop** - Coordinates execution

### Banking Scenario: Async Operations at Emirates NBD

```javascript
console.log('=== Event Loop Deep Dive - Emirates NBD ===\n');

// =============================================================================
// 1. EVENT LOOP BASICS
// =============================================================================

console.log('1. EVENT LOOP EXECUTION ORDER:\n');

console.log('Start');

// Macrotask (Task Queue)
setTimeout(() => {
    console.log('Timeout 1 (Macrotask)');
}, 0);

// Microtask (Microtask Queue)
Promise.resolve().then(() => {
    console.log('Promise 1 (Microtask)');
});

// Another microtask
queueMicrotask(() => {
    console.log('Queued Microtask');
});

// Synchronous code
console.log('End');

console.log('\nExecution Order:');
console.log('  1. Start (sync)');
console.log('  2. End (sync)');
console.log('  3. Promise 1 (microtask)');
console.log('  4. Queued Microtask (microtask)');
console.log('  5. Timeout 1 (macrotask)');

setTimeout(() => {
    console.log('\n' + '='.repeat(70) + '\n');
}, 50);

// =============================================================================
// 2. MICROTASKS VS MACROTASKS
// =============================================================================

setTimeout(() => {
    console.log('2. MICROTASKS VS MACROTASKS:\n');
    
    console.log('Scenario: Multiple tasks queued\n');
    
    setTimeout(() => console.log('  Macrotask 1'), 0);
    setTimeout(() => console.log('  Macrotask 2'), 0);
    
    Promise.resolve().then(() => console.log('  Microtask 1'));
    Promise.resolve().then(() => console.log('  Microtask 2'));
    
    console.log('  Synchronous code');
    
    setTimeout(() => {
        console.log('\nKey Point: ALL microtasks execute before next macrotask\n');
    }, 50);
}, 100);

setTimeout(() => {
    console.log('='.repeat(70) + '\n');
}, 200);

// =============================================================================
// 3. BANKING TRANSACTION PROCESSING
// =============================================================================

setTimeout(() => {
    console.log('3. BANKING TRANSACTION PROCESSING:\n');
    
    class TransactionProcessor {
        constructor() {
            this.queue = [];
            this.processing = false;
        }
        
        addTransaction(transaction) {
            this.queue.push(transaction);
            console.log(`  Queued: ${transaction.id}`);
            this.processQueue();
        }
        
        async processQueue() {
            if (this.processing) return;
            this.processing = true;
            
            while (this.queue.length > 0) {
                const transaction = this.queue.shift();
                await this.processTransaction(transaction);
            }
            
            this.processing = false;
        }
        
        async processTransaction(transaction) {
            console.log(`  Processing: ${transaction.id}`);
            
            // Simulate async processing
            return new Promise(resolve => {
                setTimeout(() => {
                    console.log(`  Completed: ${transaction.id}`);
                    resolve();
                }, 10);
            });
        }
    }
    
    const processor = new TransactionProcessor();
    
    // Add transactions
    processor.addTransaction({ id: 'TXN-001', amount: 5000 });
    processor.addTransaction({ id: 'TXN-002', amount: 3000 });
    processor.addTransaction({ id: 'TXN-003', amount: 7000 });
}, 250);

setTimeout(() => {
    console.log('\n' + '='.repeat(70) + '\n');
}, 350);

// =============================================================================
// 4. PRIORITY QUEUE WITH MICROTASKS
// =============================================================================

setTimeout(() => {
    console.log('4. PRIORITY QUEUE IMPLEMENTATION:\n');
    
    class PriorityTransactionQueue {
        constructor() {
            this.highPriority = [];
            this.normalPriority = [];
        }
        
        addTransaction(transaction, priority = 'normal') {
            const txn = { ...transaction, priority };
            
            if (priority === 'high') {
                this.highPriority.push(txn);
                // Use microtask for high priority
                queueMicrotask(() => this.processHighPriority(txn));
            } else {
                this.normalPriority.push(txn);
                // Use macrotask for normal priority
                setTimeout(() => this.processNormalPriority(txn), 0);
            }
        }
        
        processHighPriority(transaction) {
            console.log(`  [HIGH PRIORITY] Processing: ${transaction.id}`);
        }
        
        processNormalPriority(transaction) {
            console.log(`  [NORMAL] Processing: ${transaction.id}`);
        }
    }
    
    const priorityQueue = new PriorityTransactionQueue();
    
    // Queue transactions
    priorityQueue.addTransaction({ id: 'TXN-001' }, 'normal');
    priorityQueue.addTransaction({ id: 'TXN-002' }, 'high');
    priorityQueue.addTransaction({ id: 'TXN-003' }, 'normal');
    priorityQueue.addTransaction({ id: 'TXN-004' }, 'high');
    
    console.log('\nNote: High priority (microtasks) execute before normal (macrotasks)\n');
}, 400);

setTimeout(() => {
    console.log('='.repeat(70) + '\n');
}, 500);

// =============================================================================
// 5. NESTED TIMERS AND PROMISES
// =============================================================================

setTimeout(() => {
    console.log('5. NESTED TIMERS AND PROMISES:\n');
    
    setTimeout(() => {
        console.log('  Outer setTimeout');
        
        Promise.resolve().then(() => {
            console.log('  Promise inside setTimeout (microtask)');
        });
        
        setTimeout(() => {
            console.log('  Inner setTimeout (macrotask)');
        }, 0);
        
        console.log('  Synchronous inside setTimeout');
    }, 0);
    
    Promise.resolve()
        .then(() => {
            console.log('  Outer Promise');
            
            setTimeout(() => {
                console.log('  setTimeout inside Promise (macrotask)');
            }, 0);
            
            return Promise.resolve();
        })
        .then(() => {
            console.log('  Chained Promise (microtask)');
        });
    
    console.log('  Main thread');
}, 550);

setTimeout(() => {
    console.log('\n' + '='.repeat(70) + '\n');
}, 650);

// =============================================================================
// 6. PRACTICAL BANKING EXAMPLE
// =============================================================================

setTimeout(() => {
    console.log('6. PRACTICAL BANKING EXAMPLE:\n');
    
    class BankingService {
        async transferMoney(fromAccount, toAccount, amount) {
            console.log(`  Starting transfer: ${amount} AED`);
            
            // Step 1: Validate (sync)
            console.log('  1. Validating...');
            
            // Step 2: Check balance (async - microtask)
            await Promise.resolve().then(() => {
                console.log('  2. Checking balance (microtask)');
            });
            
            // Step 3: Debit account (async - simulated delay)
            await new Promise(resolve => {
                setTimeout(() => {
                    console.log('  3. Debiting account (macrotask)');
                    resolve();
                }, 10);
            });
            
            // Step 4: Credit account (async - microtask)
            await Promise.resolve().then(() => {
                console.log('  4. Crediting account (microtask)');
            });
            
            // Step 5: Send notification (async - macrotask)
            setTimeout(() => {
                console.log('  5. Sending notification (macrotask)');
            }, 0);
            
            console.log('  Transfer initiated\n');
        }
    }
    
    const bankingService = new BankingService();
    bankingService.transferMoney('ACC-111', 'ACC-222', 5000);
}, 700);

setTimeout(() => {
    console.log('='.repeat(70) + '\n');
}, 850);

// =============================================================================
// EVENT LOOP SUMMARY
// =============================================================================

setTimeout(() => {
    console.log('EVENT LOOP SUMMARY:\n');
    
    console.log('Execution Order:');
    console.log('  1. Synchronous code (call stack)');
    console.log('  2. All microtasks (promise callbacks, queueMicrotask)');
    console.log('  3. One macrotask (setTimeout, setInterval, I/O)');
    console.log('  4. Repeat from step 2\n');
    
    console.log('Task Types:');
    console.log('  Microtasks:');
    console.log('    • Promise.then/catch/finally');
    console.log('    • queueMicrotask()');
    console.log('    • MutationObserver');
    console.log('    • process.nextTick (Node.js)');
    
    console.log('\n  Macrotasks:');
    console.log('    • setTimeout/setInterval');
    console.log('    • setImmediate (Node.js)');
    console.log('    • I/O operations');
    console.log('    • UI rendering\n');
    
    console.log('Key Rules:');
    console.log('  ✓ All microtasks execute before next macrotask');
    console.log('  ✓ Each macrotask runs to completion');
    console.log('  ✓ Microtasks can queue more microtasks');
    console.log('  ✓ Be careful with infinite microtask loops');
    
    console.log();
}, 900);
```

---

## Question 41: What are Async/Await best practices and error handling patterns?

### Answer:

**Async/Await** is syntactic sugar over Promises that makes asynchronous code look and behave more like synchronous code. Understanding best practices is essential for maintainable code.

### Best Practices:
1. **Always handle errors** with try/catch
2. **Use Promise.all()** for parallel operations
3. **Avoid blocking** with sequential awaits
4. **Handle rejections** in promises
5. **Set timeouts** for network operations

### Banking Scenario: Async Patterns at Emirates NBD

```javascript
console.log('=== Async/Await Best Practices - Emirates NBD ===\n');

// =============================================================================
// 1. PROPER ERROR HANDLING
// =============================================================================

console.log('1. PROPER ERROR HANDLING:\n');

// ❌ BAD: No error handling
async function transferMoneyBad(amount) {
    const balance = await getBalance();
    const result = await processTransfer(amount);
    return result;
}

// ✅ GOOD: Proper error handling
async function transferMoneyGood(amount) {
    try {
        const balance = await getBalance();
        
        if (balance < amount) {
            throw new Error('Insufficient funds');
        }
        
        const result = await processTransfer(amount);
        return { success: true, result };
        
    } catch (error) {
        console.error(`Transfer failed: ${error.message}`);
        return { success: false, error: error.message };
    }
}

// Mock functions
async function getBalance() {
    return 50000;
}

async function processTransfer(amount) {
    return { transactionId: 'TXN-001', amount };
}

// Test
transferMoneyGood(5000).then(result => {
    console.log('  Transfer result:', result);
    console.log();
});

setTimeout(() => {
    console.log('='.repeat(70) + '\n');
}, 50);

// =============================================================================
// 2. PARALLEL VS SEQUENTIAL EXECUTION
// =============================================================================

setTimeout(() => {
    console.log('2. PARALLEL VS SEQUENTIAL:\n');
    
    // ❌ BAD: Sequential (slower)
    async function getAccountDataSequential(accountId) {
        console.log('  Sequential execution:');
        const start = Date.now();
        
        const account = await fetchAccount(accountId);
        const transactions = await fetchTransactions(accountId);
        const balance = await fetchBalance(accountId);
        
        const duration = Date.now() - start;
        console.log(`    Time: ${duration}ms\n`);
        
        return { account, transactions, balance };
    }
    
    // ✅ GOOD: Parallel (faster)
    async function getAccountDataParallel(accountId) {
        console.log('  Parallel execution:');
        const start = Date.now();
        
        const [account, transactions, balance] = await Promise.all([
            fetchAccount(accountId),
            fetchTransactions(accountId),
            fetchBalance(accountId)
        ]);
        
        const duration = Date.now() - start;
        console.log(`    Time: ${duration}ms\n`);
        
        return { account, transactions, balance };
    }
    
    // Mock async functions (10ms delay each)
    async function fetchAccount(id) {
        await new Promise(resolve => setTimeout(resolve, 10));
        return { id, holder: 'Ahmed' };
    }
    
    async function fetchTransactions(id) {
        await new Promise(resolve => setTimeout(resolve, 10));
        return [{ id: 1, amount: 5000 }];
    }
    
    async function fetchBalance(id) {
        await new Promise(resolve => setTimeout(resolve, 10));
        return 50000;
    }
    
    // Test both approaches
    (async () => {
        await getAccountDataSequential('ACC-123');
        await getAccountDataParallel('ACC-123');
    })();
}, 100);

setTimeout(() => {
    console.log('='.repeat(70) + '\n');
}, 200);

// =============================================================================
// 3. ERROR HANDLING WITH PROMISE.ALL
// =============================================================================

setTimeout(() => {
    console.log('3. ERROR HANDLING WITH PROMISE.ALL:\n');
    
    // Problem: Promise.all fails fast
    async function processTransactionsAllOrNothing(transactions) {
        try {
            const results = await Promise.all(
                transactions.map(txn => processTransaction(txn))
            );
            console.log('  All transactions succeeded');
            return results;
        } catch (error) {
            console.log('  ❌ One failure stops all:', error.message);
            return null;
        }
    }
    
    // Solution: Promise.allSettled for partial success
    async function processTransactionsPartial(transactions) {
        const results = await Promise.allSettled(
            transactions.map(txn => processTransaction(txn))
        );
        
        const succeeded = results.filter(r => r.status === 'fulfilled');
        const failed = results.filter(r => r.status === 'rejected');
        
        console.log(`  ✓ Succeeded: ${succeeded.length}`);
        console.log(`  ✗ Failed: ${failed.length}\n`);
        
        return { succeeded, failed };
    }
    
    async function processTransaction(transaction) {
        await new Promise(resolve => setTimeout(resolve, 10));
        
        if (transaction.amount < 0) {
            throw new Error(`Invalid amount for ${transaction.id}`);
        }
        
        return { ...transaction, status: 'completed' };
    }
    
    const transactions = [
        { id: 'TXN-001', amount: 5000 },
        { id: 'TXN-002', amount: -100 }, // Will fail
        { id: 'TXN-003', amount: 3000 }
    ];
    
    (async () => {
        await processTransactionsAllOrNothing(transactions);
        await processTransactionsPartial(transactions);
    })();
}, 250);

setTimeout(() => {
    console.log('='.repeat(70) + '\n');
}, 350);

// =============================================================================
// 4. TIMEOUT PATTERN
// =============================================================================

setTimeout(() => {
    console.log('4. TIMEOUT PATTERN:\n');
    
    // Create timeout promise
    function timeout(ms) {
        return new Promise((_, reject) => {
            setTimeout(() => reject(new Error('Operation timed out')), ms);
        });
    }
    
    // Wrap operation with timeout
    async function withTimeout(promise, ms) {
        return Promise.race([promise, timeout(ms)]);
    }
    
    // Slow API call
    async function slowAPICall() {
        await new Promise(resolve => setTimeout(resolve, 200));
        return { data: 'response' };
    }
    
    // Fast API call
    async function fastAPICall() {
        await new Promise(resolve => setTimeout(resolve, 10));
        return { data: 'response' };
    }
    
    (async () => {
        try {
            console.log('  Fast call with timeout:');
            const result1 = await withTimeout(fastAPICall(), 100);
            console.log('    ✓ Success\n');
        } catch (error) {
            console.log(`    ✗ ${error.message}\n`);
        }
        
        try {
            console.log('  Slow call with timeout:');
            const result2 = await withTimeout(slowAPICall(), 100);
            console.log('    ✓ Success\n');
        } catch (error) {
            console.log(`    ✗ ${error.message}\n`);
        }
    })();
}, 400);

setTimeout(() => {
    console.log('='.repeat(70) + '\n');
}, 650);

// =============================================================================
// 5. RETRY PATTERN
// =============================================================================

setTimeout(() => {
    console.log('5. RETRY PATTERN:\n');
    
    async function retryOperation(operation, maxRetries = 3, delay = 1000) {
        let lastError;
        
        for (let attempt = 1; attempt <= maxRetries; attempt++) {
            try {
                console.log(`  Attempt ${attempt}/${maxRetries}`);
                const result = await operation();
                console.log('  ✓ Success\n');
                return result;
            } catch (error) {
                lastError = error;
                console.log(`  ✗ Failed: ${error.message}`);
                
                if (attempt < maxRetries) {
                    console.log(`  Waiting ${delay}ms before retry...\n`);
                    await new Promise(resolve => setTimeout(resolve, delay));
                }
            }
        }
        
        throw new Error(`Failed after ${maxRetries} attempts: ${lastError.message}`);
    }
    
    // Simulated flaky operation
    let callCount = 0;
    async function flakyAPICall() {
        callCount++;
        await new Promise(resolve => setTimeout(resolve, 10));
        
        if (callCount < 3) {
            throw new Error('Network error');
        }
        
        return { success: true };
    }
    
    (async () => {
        try {
            await retryOperation(flakyAPICall, 3, 50);
        } catch (error) {
            console.log(`  Final error: ${error.message}\n`);
        }
    })();
}, 700);

setTimeout(() => {
    console.log('='.repeat(70) + '\n');
}, 900);

// =============================================================================
// 6. PRACTICAL BANKING SERVICE
// =============================================================================

setTimeout(() => {
    console.log('6. PRACTICAL BANKING SERVICE:\n');
    
    class BankingService {
        constructor() {
            this.timeout = 5000;
            this.maxRetries = 3;
        }
        
        // Execute with timeout
        async withTimeout(promise) {
            return Promise.race([
                promise,
                new Promise((_, reject) => 
                    setTimeout(() => reject(new Error('Timeout')), this.timeout)
                )
            ]);
        }
        
        // Execute with retry
        async withRetry(operation) {
            for (let i = 0; i < this.maxRetries; i++) {
                try {
                    return await operation();
                } catch (error) {
                    if (i === this.maxRetries - 1) throw error;
                    await new Promise(r => setTimeout(r, 1000 * (i + 1)));
                }
            }
        }
        
        // Transfer money with all patterns
        async transferMoney(fromAccount, toAccount, amount) {
            try {
                console.log('  Starting transfer...');
                
                // Parallel validation
                const [senderBalance, receiverExists] = await Promise.all([
                    this.withTimeout(this.getBalance(fromAccount)),
                    this.withTimeout(this.validateAccount(toAccount))
                ]);
                
                if (senderBalance < amount) {
                    throw new Error('Insufficient funds');
                }
                
                if (!receiverExists) {
                    throw new Error('Invalid receiver account');
                }
                
                // Process transfer with retry
                const result = await this.withRetry(() => 
                    this.processTransfer(fromAccount, toAccount, amount)
                );
                
                console.log(`  ✓ Transfer successful: ${result.transactionId}\n`);
                return result;
                
            } catch (error) {
                console.error(`  ✗ Transfer failed: ${error.message}\n`);
                throw error;
            }
        }
        
        async getBalance(account) {
            await new Promise(resolve => setTimeout(resolve, 10));
            return 50000;
        }
        
        async validateAccount(account) {
            await new Promise(resolve => setTimeout(resolve, 10));
            return true;
        }
        
        async processTransfer(from, to, amount) {
            await new Promise(resolve => setTimeout(resolve, 10));
            return { transactionId: 'TXN-' + Date.now(), amount };
        }
    }
    
    const service = new BankingService();
    service.transferMoney('ACC-111', 'ACC-222', 5000);
}, 950);

setTimeout(() => {
    console.log('='.repeat(70) + '\n');
}, 1100);

// =============================================================================
// ASYNC/AWAIT BEST PRACTICES SUMMARY
// =============================================================================

setTimeout(() => {
    console.log('ASYNC/AWAIT BEST PRACTICES:\n');
    
    console.log('Error Handling:');
    console.log('  ✓ Always use try/catch');
    console.log('  ✓ Handle errors at appropriate level');
    console.log('  ✓ Provide meaningful error messages');
    console.log('  ✓ Log errors for debugging\n');
    
    console.log('Performance:');
    console.log('  ✓ Use Promise.all() for parallel operations');
    console.log('  ✓ Avoid sequential awaits when not needed');
    console.log('  ✓ Use Promise.allSettled() for partial success');
    console.log('  ✓ Implement caching for frequent calls\n');
    
    console.log('Reliability:');
    console.log('  ✓ Set timeouts for network operations');
    console.log('  ✓ Implement retry logic for transient failures');
    console.log('  ✓ Use circuit breakers for failing services');
    console.log('  ✓ Validate inputs before async calls\n');
    
    console.log('Common Patterns:');
    console.log('  • Promise.all() - All or nothing');
    console.log('  • Promise.allSettled() - Partial success');
    console.log('  • Promise.race() - First to complete');
    console.log('  • Promise.any() - First to succeed');
    
    console.log();
}, 1150);
```

---

## Question 42: What are memory management and performance optimization techniques in JavaScript?

### Answer:

**Memory Management** and **Performance Optimization** are critical for building scalable applications. JavaScript's garbage collector handles most memory management, but developers must avoid common pitfalls.

### Key Concepts:
1. **Garbage Collection** - Automatic memory management
2. **Memory Leaks** - Unintentional memory retention
3. **Performance Profiling** - Identifying bottlenecks
4. **Optimization Techniques** - Improving execution speed

### Banking Scenario: Performance Optimization at Emirates NBD

```javascript
console.log('=== Memory Management & Performance - Emirates NBD ===\n');

// =============================================================================
// 1. MEMORY LEAK PREVENTION
// =============================================================================

console.log('1. MEMORY LEAK PREVENTION:\n');

// ❌ BAD: Event listener not removed
class BadTransactionMonitor {
    constructor() {
        this.transactions = [];
        this.element = { addEventListener: () => {} }; // Mock DOM
        
        this.element.addEventListener('transaction', (e) => {
            this.transactions.push(e); // Leaks memory!
        });
    }
}

// ✅ GOOD: Proper cleanup
class GoodTransactionMonitor {
    constructor() {
        this.transactions = [];
        this.element = { 
            addEventListener: () => {},
            removeEventListener: () => {}
        };
        
        this.handleTransaction = this.handleTransaction.bind(this);
        this.element.addEventListener('transaction', this.handleTransaction);
    }
    
    handleTransaction(e) {
        this.transactions.push(e);
    }
    
    destroy() {
        this.element.removeEventListener('transaction', this.handleTransaction);
        this.transactions = null;
        console.log('  ✓ Cleaned up event listeners\n');
    }
}

const monitor = new GoodTransactionMonitor();
monitor.destroy();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. EFFICIENT DATA STRUCTURES
// =============================================================================

console.log('2. EFFICIENT DATA STRUCTURES:\n');

// ❌ SLOW: Array lookup O(n)
class SlowAccountLookup {
    constructor() {
        this.accounts = [];
    }
    
    addAccount(account) {
        this.accounts.push(account);
    }
    
    findAccount(accountNumber) {
        return this.accounts.find(a => a.accountNumber === accountNumber);
    }
}

// ✅ FAST: Map lookup O(1)
class FastAccountLookup {
    constructor() {
        this.accounts = new Map();
    }
    
    addAccount(account) {
        this.accounts.set(account.accountNumber, account);
    }
    
    findAccount(accountNumber) {
        return this.accounts.get(accountNumber);
    }
}

// Performance comparison
const slow = new SlowAccountLookup();
const fast = new FastAccountLookup();

console.log('Adding 10,000 accounts...');

const startSlow = performance.now();
for (let i = 0; i < 10000; i++) {
    slow.addAccount({ accountNumber: `ACC-${i}`, balance: 50000 });
}
const endSlow = performance.now();

const startFast = performance.now();
for (let i = 0; i < 10000; i++) {
    fast.addAccount({ accountNumber: `ACC-${i}`, balance: 50000 });
}
const endFast = performance.now();

console.log(`  Array: ${(endSlow - startSlow).toFixed(2)}ms`);
console.log(`  Map: ${(endFast - startFast).toFixed(2)}ms`);

console.log('\nLookup performance:');

const lookupStartSlow = performance.now();
slow.findAccount('ACC-9999');
const lookupEndSlow = performance.now();

const lookupStartFast = performance.now();
fast.findAccount('ACC-9999');
const lookupEndFast = performance.now();

console.log(`  Array lookup: ${((lookupEndSlow - lookupStartSlow) * 1000).toFixed(3)}µs`);
console.log(`  Map lookup: ${((lookupEndFast - lookupStartFast) * 1000).toFixed(3)}µs\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. MEMOIZATION FOR EXPENSIVE CALCULATIONS
// =============================================================================

console.log('3. MEMOIZATION:\n');

// Without memoization
function calculateInterest(balance, rate, years) {
    // Simulate expensive calculation
    let result = balance;
    for (let i = 0; i < years; i++) {
        result *= (1 + rate);
    }
    return result;
}

// With memoization
function memoize(fn) {
    const cache = new Map();
    
    return function(...args) {
        const key = JSON.stringify(args);
        
        if (cache.has(key)) {
            console.log('  ✓ Cache hit');
            return cache.get(key);
        }
        
        console.log('  ✗ Cache miss - calculating...');
        const result = fn.apply(this, args);
        cache.set(key, result);
        return result;
    };
}

const memoizedInterest = memoize(calculateInterest);

console.log('First call:');
const result1 = memoizedInterest(100000, 0.05, 10);
console.log(`  Result: ${result1.toFixed(2)}\n`);

console.log('Second call (same args):');
const result2 = memoizedInterest(100000, 0.05, 10);
console.log(`  Result: ${result2.toFixed(2)}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. DEBOUNCING AND THROTTLING
// =============================================================================

console.log('4. DEBOUNCING AND THROTTLING:\n');

// Debounce - wait for pause in events
function debounce(func, delay) {
    let timeoutId;
    return function(...args) {
        clearTimeout(timeoutId);
        timeoutId = setTimeout(() => func.apply(this, args), delay);
    };
}

// Throttle - limit execution frequency
function throttle(func, limit) {
    let inThrottle;
    return function(...args) {
        if (!inThrottle) {
            func.apply(this, args);
            inThrottle = true;
            setTimeout(() => inThrottle = false, limit);
        }
    };
}

// Example: Search with debounce
const searchAccounts = debounce((query) => {
    console.log(`  Searching for: ${query}`);
}, 300);

console.log('Debounced search (waits for pause):');
searchAccounts('Ahmed');
searchAccounts('Ahmed Al');
searchAccounts('Ahmed Al Maktoum');

setTimeout(() => {
    console.log('  → Only last search executed\n');
}, 400);

setTimeout(() => {
    console.log('='.repeat(70) + '\n');
}, 500);

// =============================================================================
// 5. OBJECT POOLING
// =============================================================================

setTimeout(() => {
    console.log('5. OBJECT POOLING:\n');
    
    class TransactionPool {
        constructor(size = 100) {
            this.pool = [];
            this.inUse = new Set();
            
            // Pre-create objects
            for (let i = 0; i < size; i++) {
                this.pool.push(this.createTransaction());
            }
            
            console.log(`  Pool initialized with ${size} objects\n`);
        }
        
        createTransaction() {
            return {
                id: null,
                amount: 0,
                timestamp: null,
                reset() {
                    this.id = null;
                    this.amount = 0;
                    this.timestamp = null;
                }
            };
        }
        
        acquire() {
            let transaction = this.pool.pop();
            
            if (!transaction) {
                transaction = this.createTransaction();
                console.log('  ! Pool exhausted - creating new object');
            }
            
            this.inUse.add(transaction);
            return transaction;
        }
        
        release(transaction) {
            transaction.reset();
            this.inUse.delete(transaction);
            this.pool.push(transaction);
        }
        
        getStats() {
            return {
                pooled: this.pool.length,
                inUse: this.inUse.size,
                total: this.pool.length + this.inUse.size
            };
        }
    }
    
    const pool = new TransactionPool(3);
    
    const txn1 = pool.acquire();
    txn1.id = 'TXN-001';
    txn1.amount = 5000;
    
    const txn2 = pool.acquire();
    txn2.id = 'TXN-002';
    txn2.amount = 3000;
    
    console.log('After acquiring 2 transactions:');
    console.log('  ', pool.getStats());
    
    pool.release(txn1);
    
    console.log('\nAfter releasing 1 transaction:');
    console.log('  ', pool.getStats());
    console.log();
}, 550);

setTimeout(() => {
    console.log('='.repeat(70) + '\n');
}, 650);

// =============================================================================
// 6. LAZY LOADING
// =============================================================================

setTimeout(() => {
    console.log('6. LAZY LOADING:\n');
    
    class TransactionHistory {
        constructor(accountNumber) {
            this.accountNumber = accountNumber;
            this._transactions = null;
        }
        
        // Lazy load transactions
        async getTransactions() {
            if (this._transactions === null) {
                console.log('  Loading transactions...');
                this._transactions = await this.fetchTransactions();
            } else {
                console.log('  Using cached transactions');
            }
            
            return this._transactions;
        }
        
        async fetchTransactions() {
            // Simulate API call
            await new Promise(resolve => setTimeout(resolve, 100));
            return [
                { id: 1, amount: 5000 },
                { id: 2, amount: 3000 },
                { id: 3, amount: 7000 }
            ];
        }
        
        invalidateCache() {
            this._transactions = null;
            console.log('  Cache invalidated\n');
        }
    }
    
    const history = new TransactionHistory('ACC-123');
    
    (async () => {
        console.log('First access:');
        await history.getTransactions();
        
        console.log('\nSecond access:');
        await history.getTransactions();
        
        history.invalidateCache();
        
        console.log('\nAfter cache invalidation:');
        await history.getTransactions();
    })();
}, 700);

setTimeout(() => {
    console.log('\n' + '='.repeat(70) + '\n');
}, 1000);

// =============================================================================
// PERFORMANCE OPTIMIZATION SUMMARY
// =============================================================================

setTimeout(() => {
    console.log('PERFORMANCE OPTIMIZATION SUMMARY:\n');
    
    console.log('Memory Management:');
    console.log('  ✓ Remove event listeners when done');
    console.log('  ✓ Clear intervals and timeouts');
    console.log('  ✓ Avoid circular references');
    console.log('  ✓ Use WeakMap/WeakSet for caches');
    console.log('  ✓ Implement object pooling for frequent allocations\n');
    
    console.log('Data Structures:');
    console.log('  ✓ Use Map/Set for lookups (O(1) vs O(n))');
    console.log('  ✓ Use TypedArrays for numeric data');
    console.log('  ✓ Choose appropriate data structure');
    console.log('  ✓ Avoid sparse arrays\n');
    
    console.log('Computation:');
    console.log('  ✓ Memoize expensive calculations');
    console.log('  ✓ Debounce frequent events');
    console.log('  ✓ Throttle high-frequency operations');
    console.log('  ✓ Use Web Workers for CPU-intensive tasks\n');
    
    console.log('Loading:');
    console.log('  ✓ Lazy load resources');
    console.log('  ✓ Implement pagination');
    console.log('  ✓ Use virtual scrolling for large lists');
    console.log('  ✓ Cache API responses\n');
    
    console.log('Best Practices:');
    console.log('  ✓ Profile before optimizing');
    console.log('  ✓ Measure performance improvements');
    console.log('  ✓ Optimize hot paths first');
    console.log('  ✓ Balance readability and performance');
    
    console.log();
}, 1050);
```

### Key Takeaways:

**Event Loop**:
- Microtasks execute before macrotasks
- Understand execution order for debugging
- Use priority queues for task management
- Be careful with infinite microtask loops

**Async/Await**:
- Always handle errors with try/catch
- Use Promise.all() for parallel operations
- Implement timeouts and retries
- Choose right Promise method (all, allSettled, race, any)

**Performance**:
- Prevent memory leaks (cleanup listeners)
- Use efficient data structures (Map vs Array)
- Memoize expensive calculations
- Debounce/throttle frequent operations
- Implement lazy loading and caching

---

**End of Questions 40-42**
