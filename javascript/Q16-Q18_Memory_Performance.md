# Questions 16-18: Memory Management & Performance Optimization

## Question 16: What are Memory Leaks? How do you prevent them in JavaScript?

### Answer:

**Memory leaks** occur when your application holds references to objects that are no longer needed, preventing the garbage collector from freeing that memory. Over time, this can degrade performance and crash the application.

### Common Causes of Memory Leaks:

1. **Global variables**: Unintentional globals stay in memory forever
2. **Event listeners**: Not removed when elements are destroyed
3. **Closures**: Holding references to unnecessary outer scope variables
4. **Timers**: setInterval/setTimeout not cleared
5. **Detached DOM nodes**: Removed from DOM but still referenced
6. **Circular references**: Objects referencing each other

### Banking Scenario: Memory-Safe Transaction Processing at Emirates NBD

```javascript
// Example 1: Memory Leak from Event Listeners
class BadTransactionMonitor {
    constructor() {
        this.transactions = [];
        this.listeners = [];
    }

    // ❌ MEMORY LEAK: Event listener not cleaned up
    startMonitoring(accountNumber) {
        const updateHandler = (event) => {
            console.log('Transaction update:', event.detail);
            this.transactions.push(event.detail);
        };

        // This creates a memory leak if not removed
        document.addEventListener('transaction-update', updateHandler);
        this.listeners.push(updateHandler);
    }

    // Missing cleanup method!
}

// ✅ FIXED: Proper cleanup
class GoodTransactionMonitor {
    constructor() {
        this.transactions = [];
        this.listeners = new Map(); // Track listeners by account
        this.abortControllers = new Map(); // For cleanup
    }

    startMonitoring(accountNumber) {
        // Use AbortController for automatic cleanup
        const abortController = new AbortController();
        
        const updateHandler = (event) => {
            console.log(`[${accountNumber}] Transaction update:`, event.detail);
            this.transactions.push(event.detail);
        };

        // Add listener with abort signal
        document.addEventListener('transaction-update', updateHandler, {
            signal: abortController.signal
        });

        this.listeners.set(accountNumber, updateHandler);
        this.abortControllers.set(accountNumber, abortController);
    }

    stopMonitoring(accountNumber) {
        // Abort controller automatically removes all associated listeners
        const abortController = this.abortControllers.get(accountNumber);
        if (abortController) {
            abortController.abort();
            this.abortControllers.delete(accountNumber);
            this.listeners.delete(accountNumber);
            console.log(`✅ Stopped monitoring ${accountNumber} - listeners cleaned up`);
        }
    }

    cleanup() {
        // Clean up all listeners
        for (const [accountNumber, abortController] of this.abortControllers) {
            abortController.abort();
        }
        this.abortControllers.clear();
        this.listeners.clear();
        this.transactions = [];
        console.log('✅ All monitors cleaned up');
    }
}

// Example 2: Memory Leak from Closures
class TransactionCache {
    constructor() {
        this.cache = new Map();
    }

    // ❌ MEMORY LEAK: Closure holds reference to large data
    createBadCacheGetter(accountNumber) {
        const largeTransactionHistory = this.fetchLargeHistory(accountNumber); // 10MB of data
        
        // This closure keeps largeTransactionHistory in memory forever
        return () => {
            return largeTransactionHistory[0]; // Only need first item
        };
    }

    // ✅ FIXED: Don't capture unnecessary data in closure
    createGoodCacheGetter(accountNumber) {
        const largeTransactionHistory = this.fetchLargeHistory(accountNumber);
        const firstTransaction = largeTransactionHistory[0]; // Extract only what's needed
        
        // Now closure only holds reference to small object
        return () => {
            return firstTransaction;
        };
    }

    fetchLargeHistory(accountNumber) {
        // Simulate large transaction history
        return Array.from({ length: 10000 }, (_, i) => ({
            id: `TXN-${i}`,
            amount: Math.random() * 10000,
            timestamp: Date.now(),
            accountNumber
        }));
    }
}

// Example 3: Memory Leak from Timers
class BadAccountBalancePoller {
    constructor(accountNumber) {
        this.accountNumber = accountNumber;
        this.balance = 0;
    }

    // ❌ MEMORY LEAK: Timer never cleared
    startPolling() {
        setInterval(() => {
            this.fetchBalance().then(balance => {
                this.balance = balance;
                console.log(`Balance for ${this.accountNumber}: AED ${balance}`);
            });
        }, 5000);
    }

    async fetchBalance() {
        // Simulate API call
        return Math.random() * 100000;
    }
}

// ✅ FIXED: Proper timer management
class GoodAccountBalancePoller {
    constructor(accountNumber) {
        this.accountNumber = accountNumber;
        this.balance = 0;
        this.pollingInterval = null;
        this.isActive = false;
    }

    startPolling(intervalMs = 5000) {
        if (this.isActive) {
            console.log('Polling already active');
            return;
        }

        this.isActive = true;
        
        this.pollingInterval = setInterval(async () => {
            try {
                this.balance = await this.fetchBalance();
                console.log(`Balance for ${this.accountNumber}: AED ${this.balance.toFixed(2)}`);
            } catch (error) {
                console.error('Failed to fetch balance:', error);
            }
        }, intervalMs);

        console.log(`✅ Started polling for ${this.accountNumber}`);
    }

    stopPolling() {
        if (this.pollingInterval) {
            clearInterval(this.pollingInterval);
            this.pollingInterval = null;
            this.isActive = false;
            console.log(`✅ Stopped polling for ${this.accountNumber}`);
        }
    }

    async fetchBalance() {
        // Simulate API call
        return Math.random() * 100000;
    }

    // Cleanup method
    destroy() {
        this.stopPolling();
        this.balance = null;
        this.accountNumber = null;
        console.log('✅ Poller destroyed');
    }
}

// Example 4: Memory Leak from DOM References
class TransactionDashboard {
    constructor() {
        this.transactionElements = []; // ❌ Can cause memory leak
        this.transactionData = new Map(); // ✅ Better approach
    }

    // ❌ BAD: Storing DOM elements
    addTransactionBad(transaction) {
        const element = document.createElement('div');
        element.textContent = `${transaction.id}: AED ${transaction.amount}`;
        document.body.appendChild(element);
        
        // Storing DOM reference prevents garbage collection
        this.transactionElements.push(element);
    }

    // ✅ GOOD: Store only data, recreate DOM when needed
    addTransactionGood(transaction) {
        // Store only data
        this.transactionData.set(transaction.id, transaction);
    }

    renderTransactions() {
        // Clear existing DOM
        const container = document.getElementById('transaction-list');
        if (container) {
            container.innerHTML = '';
            
            // Recreate DOM from data
            for (const [id, transaction] of this.transactionData) {
                const element = document.createElement('div');
                element.textContent = `${id}: AED ${transaction.amount}`;
                container.appendChild(element);
            }
        }
    }

    // Proper cleanup
    cleanup() {
        this.transactionData.clear();
        const container = document.getElementById('transaction-list');
        if (container) {
            container.innerHTML = '';
        }
        console.log('✅ Dashboard cleaned up');
    }
}

// Example 5: WeakMap and WeakSet for Automatic Memory Management
class AccountMetadataManager {
    constructor() {
        // ❌ Regular Map keeps strong references
        this.regularMap = new Map();
        
        // ✅ WeakMap allows garbage collection
        this.weakMap = new WeakMap();
        
        // ✅ WeakSet for tracking
        this.activeAccounts = new WeakSet();
    }

    // Using regular Map (prevents garbage collection)
    attachMetadataWithMap(accountObject, metadata) {
        this.regularMap.set(accountObject, metadata);
        // accountObject can never be garbage collected while in this map
    }

    // Using WeakMap (allows garbage collection)
    attachMetadataWithWeakMap(accountObject, metadata) {
        this.weakMap.set(accountObject, metadata);
        // When accountObject is no longer used elsewhere, it can be GC'd
        // The WeakMap entry will be automatically removed
    }

    markAccountActive(accountObject) {
        this.activeAccounts.add(accountObject);
        // WeakSet doesn't prevent garbage collection
    }

    isAccountActive(accountObject) {
        return this.activeAccounts.has(accountObject);
    }
}

// Example 6: Proper Cache with Size Limits
class LRUTransactionCache {
    constructor(maxSize = 1000) {
        this.maxSize = maxSize;
        this.cache = new Map();
    }

    set(key, value) {
        // Remove oldest entry if at capacity
        if (this.cache.size >= this.maxSize) {
            const firstKey = this.cache.keys().next().value;
            this.cache.delete(firstKey);
            console.log(`Cache full, removed oldest entry: ${firstKey}`);
        }

        // Delete and re-add to make it most recent
        this.cache.delete(key);
        this.cache.set(key, value);
    }

    get(key) {
        if (!this.cache.has(key)) {
            return null;
        }

        // Move to end (most recently used)
        const value = this.cache.get(key);
        this.cache.delete(key);
        this.cache.set(key, value);
        
        return value;
    }

    clear() {
        this.cache.clear();
        console.log('✅ Cache cleared');
    }

    getSize() {
        return this.cache.size;
    }
}

// Example 7: Resource Management with Cleanup
class BankingWebSocketConnection {
    constructor(url) {
        this.url = url;
        this.ws = null;
        this.messageHandlers = new Set();
        this.reconnectTimer = null;
    }

    connect() {
        this.ws = new WebSocket(this.url);

        this.ws.onmessage = (event) => {
            const data = JSON.parse(event.data);
            // Notify all handlers
            this.messageHandlers.forEach(handler => handler(data));
        };

        this.ws.onerror = (error) => {
            console.error('WebSocket error:', error);
        };

        this.ws.onclose = () => {
            console.log('WebSocket closed, attempting reconnect...');
            this.scheduleReconnect();
        };

        console.log('✅ WebSocket connected');
    }

    addMessageHandler(handler) {
        this.messageHandlers.add(handler);
    }

    removeMessageHandler(handler) {
        this.messageHandlers.delete(handler);
    }

    scheduleReconnect() {
        this.reconnectTimer = setTimeout(() => {
            this.connect();
        }, 5000);
    }

    // ✅ Proper cleanup
    disconnect() {
        // Clear reconnect timer
        if (this.reconnectTimer) {
            clearTimeout(this.reconnectTimer);
            this.reconnectTimer = null;
        }

        // Remove all handlers
        this.messageHandlers.clear();

        // Close WebSocket
        if (this.ws) {
            this.ws.onclose = null; // Prevent reconnect
            this.ws.close();
            this.ws = null;
        }

        console.log('✅ WebSocket disconnected and cleaned up');
    }
}

// Demonstration
console.log('=== Memory Management Demo - Emirates NBD ===\n');

// Demo 1: Event Listener Cleanup
console.log('1. Event Listener Management:\n');
const monitor = new GoodTransactionMonitor();
monitor.startMonitoring('ACC-123456789');
monitor.startMonitoring('ACC-987654321');

// Simulate some time passing
setTimeout(() => {
    monitor.stopMonitoring('ACC-123456789');
}, 1000);

setTimeout(() => {
    monitor.cleanup();
}, 2000);

// Demo 2: Timer Management
console.log('\n2. Timer Management:\n');
const poller = new GoodAccountBalancePoller('ACC-111222333');
poller.startPolling(1000);

setTimeout(() => {
    poller.stopPolling();
    poller.destroy();
}, 3000);

// Demo 3: WeakMap vs Map
console.log('\n3. WeakMap for Automatic Memory Management:\n');
const metadataManager = new AccountMetadataManager();

let tempAccount = { accountNumber: 'ACC-TEMP-001', balance: 5000 };

metadataManager.attachMetadataWithWeakMap(tempAccount, {
    createdAt: new Date(),
    lastAccessed: new Date(),
    preferences: { notifications: true }
});

console.log('Metadata attached with WeakMap');
console.log('When tempAccount is no longer referenced, it can be garbage collected\n');

// Demo 4: LRU Cache
console.log('4. LRU Cache with Size Limit:\n');
const cache = new LRUTransactionCache(5);

for (let i = 1; i <= 7; i++) {
    cache.set(`TXN-${i}`, {
        amount: i * 1000,
        timestamp: Date.now()
    });
}

console.log(`Cache size: ${cache.getSize()} (max: 5)`);
console.log('Oldest entries were automatically removed\n');

// Demo 5: Memory Profiling Tips
console.log('5. Memory Profiling Best Practices:\n');
console.log('✓ Use Chrome DevTools → Memory tab → Take Heap Snapshot');
console.log('✓ Look for Detached DOM nodes');
console.log('✓ Check for unexpectedly large arrays/objects');
console.log('✓ Monitor memory over time with Performance recording');
console.log('✓ Use console.memory (Chrome) to track memory usage');

if (typeof console.memory !== 'undefined') {
    console.log('\nCurrent Memory Usage:');
    console.log(`- Used: ${(console.memory.usedJSHeapSize / 1048576).toFixed(2)} MB`);
    console.log(`- Total: ${(console.memory.totalJSHeapSize / 1048576).toFixed(2)} MB`);
    console.log(`- Limit: ${(console.memory.jsHeapSizeLimit / 1048576).toFixed(2)} MB`);
}

console.log('\n' + '='.repeat(70));
```

### Memory Leak Prevention Checklist:

1. **Event Listeners**: Always remove when done
   - Use `AbortController` for automatic cleanup
   - Remove listeners in cleanup methods

2. **Timers**: Clear all setTimeout/setInterval
   - Store timer IDs
   - Clear in cleanup methods

3. **Closures**: Don't capture unnecessary data
   - Extract only needed data before creating closure
   - Be mindful of what variables are in scope

4. **DOM References**: Avoid storing DOM elements
   - Store data, recreate DOM when needed
   - Clear references when elements are removed

5. **Use Weak Collections**:
   - `WeakMap` for object metadata
   - `WeakSet` for object tracking

6. **Implement Cleanup Methods**:
   - Always provide `destroy()` or `cleanup()` methods
   - Call cleanup in lifecycle hooks (unmount, destroy, etc.)

7. **Limit Cache Sizes**: Implement LRU or TTL expiration

---

## Question 17: What is Debouncing and Throttling? When should you use each?

### Answer:

**Debouncing** and **Throttling** are techniques to control how often a function executes, especially for performance-critical operations like API calls, search, scroll, and resize events.

### Debouncing:
- Delays function execution until after a specified time has passed since the last invocation
- **Use Case**: When you want to wait for the user to "finish" an action
- **Example**: Search input, resize events, form validation

### Throttling:
- Ensures function executes at most once in a specified time period
- **Use Case**: When you want regular updates but not too frequently
- **Example**: Scroll events, mouse movements, API rate limiting

### Banking Scenario: Search and Real-time Updates at Emirates NBD

```javascript
// Utility Functions
class PerformanceOptimizer {
    // Debounce: Wait for user to stop typing
    static debounce(func, delay = 300) {
        let timeoutId;
        
        return function debounced(...args) {
            // Clear existing timer
            clearTimeout(timeoutId);
            
            // Set new timer
            timeoutId = setTimeout(() => {
                func.apply(this, args);
            }, delay);
        };
    }

    // Throttle: Execute at most once per interval
    static throttle(func, limit = 300) {
        let inThrottle;
        let lastResult;
        
        return function throttled(...args) {
            if (!inThrottle) {
                lastResult = func.apply(this, args);
                inThrottle = true;
                
                setTimeout(() => {
                    inThrottle = false;
                }, limit);
            }
            
            return lastResult;
        };
    }

    // Advanced: Debounce with leading edge (execute immediately, then wait)
    static debounceLeading(func, delay = 300) {
        let timeoutId;
        
        return function debouncedLeading(...args) {
            const isFirstCall = !timeoutId;
            
            clearTimeout(timeoutId);
            
            timeoutId = setTimeout(() => {
                timeoutId = null;
            }, delay);
            
            // Execute immediately on first call
            if (isFirstCall) {
                func.apply(this, args);
            }
        };
    }

    // Advanced: Throttle with trailing edge (execute on last call too)
    static throttleWithTrailing(func, limit = 300) {
        let inThrottle;
        let lastArgs;
        let lastContext;
        
        return function throttledTrailing(...args) {
            lastArgs = args;
            lastContext = this;
            
            if (!inThrottle) {
                func.apply(lastContext, lastArgs);
                inThrottle = true;
                
                setTimeout(() => {
                    inThrottle = false;
                    
                    // Execute with last arguments if any calls were made during throttle
                    if (lastArgs) {
                        func.apply(lastContext, lastArgs);
                        lastArgs = null;
                    }
                }, limit);
            }
        };
    }
}

// Example 1: Account Search with Debouncing
class AccountSearchService {
    constructor() {
        this.searchCallCount = 0;
        
        // Debounce search - wait 500ms after user stops typing
        this.debouncedSearch = PerformanceOptimizer.debounce(
            this.performSearch.bind(this),
            500
        );
    }

    // This gets called for every keystroke (not debounced)
    handleSearchInput(query) {
        console.log(`[INPUT] User typed: "${query}"`);
        
        // This will only execute after user stops typing for 500ms
        this.debouncedSearch(query);
    }

    // Actual search function - only called after debounce delay
    performSearch(query) {
        this.searchCallCount++;
        console.log(`[SEARCH #${this.searchCallCount}] Searching for: "${query}"`);
        console.log(`  → Making API call to: /api/accounts/search?q=${query}`);
        
        // Simulate API call
        return new Promise(resolve => {
            setTimeout(() => {
                const results = this.mockSearchResults(query);
                console.log(`  → Found ${results.length} results\n`);
                resolve(results);
            }, 100);
        });
    }

    mockSearchResults(query) {
        const accounts = [
            { number: 'ACC-123456789', name: 'Mohammed Ahmed', balance: 50000 },
            { number: 'ACC-987654321', name: 'Fatima Hassan', balance: 75000 },
            { number: 'ACC-555555555', name: 'Ahmed Mohammed', balance: 100000 }
        ];
        
        return accounts.filter(acc => 
            acc.name.toLowerCase().includes(query.toLowerCase()) ||
            acc.number.includes(query)
        );
    }
}

// Example 2: Balance Updates with Throttling
class BalanceDashboard {
    constructor() {
        this.updateCount = 0;
        
        // Throttle updates - maximum once per 1000ms
        this.throttledUpdate = PerformanceOptimizer.throttle(
            this.updateBalance.bind(this),
            1000
        );
    }

    // This might be called many times (e.g., from WebSocket)
    onBalanceChange(accountNumber, newBalance) {
        console.log(`[CHANGE] Balance changed for ${accountNumber}: AED ${newBalance}`);
        
        // This will only update UI once per second max
        this.throttledUpdate(accountNumber, newBalance);
    }

    // Actual UI update - throttled to once per second
    updateBalance(accountNumber, newBalance) {
        this.updateCount++;
        console.log(`[UPDATE #${this.updateCount}] Updating dashboard: ${accountNumber} → AED ${newBalance.toFixed(2)}\n`);
        
        // Expensive DOM operation or animation
        this.animateBalanceChange(newBalance);
    }

    animateBalanceChange(newBalance) {
        // Simulate expensive DOM update
        // In real app: update charts, animations, etc.
    }
}

// Example 3: Form Validation with Debouncing
class TransferFormValidator {
    constructor() {
        this.validationCount = 0;
        
        // Debounce validation - wait 300ms after user stops typing
        this.debouncedValidateAmount = PerformanceOptimizer.debounce(
            this.validateAmount.bind(this),
            300
        );
        
        this.debouncedValidateAccount = PerformanceOptimizer.debounce(
            this.validateAccountNumber.bind(this),
            500
        );
    }

    onAmountInput(amount) {
        console.log(`[INPUT] Amount entered: ${amount}`);
        this.debouncedValidateAmount(amount);
    }

    onAccountInput(accountNumber) {
        console.log(`[INPUT] Account entered: ${accountNumber}`);
        this.debouncedValidateAccount(accountNumber);
    }

    validateAmount(amount) {
        this.validationCount++;
        console.log(`[VALIDATION #${this.validationCount}] Validating amount: ${amount}`);
        
        const numAmount = parseFloat(amount);
        
        if (isNaN(numAmount)) {
            console.log('  ❌ Invalid amount format\n');
            return false;
        }
        
        if (numAmount <= 0) {
            console.log('  ❌ Amount must be positive\n');
            return false;
        }
        
        if (numAmount > 100000) {
            console.log('  ❌ Amount exceeds transfer limit\n');
            return false;
        }
        
        console.log('  ✅ Amount is valid\n');
        return true;
    }

    async validateAccountNumber(accountNumber) {
        this.validationCount++;
        console.log(`[VALIDATION #${this.validationCount}] Validating account: ${accountNumber}`);
        
        // Simulate API call to check account existence
        await new Promise(resolve => setTimeout(resolve, 200));
        
        const isValid = /^ACC-\d{9}$/.test(accountNumber);
        console.log(isValid ? '  ✅ Account exists\n' : '  ❌ Account not found\n');
        
        return isValid;
    }
}

// Example 4: Scroll Position Tracking with Throttling
class TransactionListScroller {
    constructor() {
        this.scrollCount = 0;
        
        // Throttle scroll handler - maximum 10 times per second
        this.throttledScroll = PerformanceOptimizer.throttle(
            this.handleScroll.bind(this),
            100
        );
    }

    // Called on every scroll event (potentially hundreds per second)
    onScroll(scrollPosition) {
        // This fires very frequently
        // console.log(`[SCROLL] Position: ${scrollPosition}`);
        
        // This executes at most 10 times per second
        this.throttledScroll(scrollPosition);
    }

    // Actual scroll handler - throttled
    handleScroll(scrollPosition) {
        this.scrollCount++;
        console.log(`[THROTTLED SCROLL #${this.scrollCount}] Processing scroll at: ${scrollPosition}`);
        
        // Expensive operations:
        // - Load more transactions if near bottom
        // - Update visible transaction range
        // - Lazy load images
        
        if (this.isNearBottom(scrollPosition)) {
            console.log('  → Loading more transactions...');
            this.loadMoreTransactions();
        }
    }

    isNearBottom(scrollPosition) {
        // Simplified check
        return scrollPosition > 800;
    }

    loadMoreTransactions() {
        // Simulate loading more data
        console.log('  → Fetched 20 more transactions\n');
    }
}

// Example 5: API Rate Limiting
class RateLimitedBankingAPI {
    constructor(requestsPerMinute = 60) {
        this.requestsPerMinute = requestsPerMinute;
        this.requestCount = 0;
        
        // Throttle to enforce rate limit
        this.throttledRequest = PerformanceOptimizer.throttle(
            this.makeRequest.bind(this),
            60000 / requestsPerMinute // Convert to milliseconds per request
        );
    }

    // Public method - automatically throttled
    async getAccountDetails(accountNumber) {
        return this.throttledRequest('GET', '/accounts/' + accountNumber);
    }

    async getTransactions(accountNumber) {
        return this.throttledRequest('GET', `/accounts/${accountNumber}/transactions`);
    }

    // Internal request method
    makeRequest(method, endpoint) {
        this.requestCount++;
        console.log(`[API REQUEST #${this.requestCount}] ${method} ${endpoint}`);
        console.log(`  Rate: ${this.requestsPerMinute} requests/minute`);
        
        // Simulate API call
        return new Promise(resolve => {
            setTimeout(() => {
                resolve({ success: true, data: {} });
            }, 50);
        });
    }
}

// Demonstration
console.log('=== Debouncing & Throttling Demo - Emirates NBD ===\n');

// Demo 1: Account Search (Debouncing)
console.log('1. DEBOUNCING - Account Search:');
console.log('User types: "M" → "Mo" → "Moh" → "Moha" → "Moham" → "Mohame" → "Mohamed"\n');

const searchService = new AccountSearchService();

// Simulate rapid typing (search only executes after user stops)
searchService.handleSearchInput('M');
setTimeout(() => searchService.handleSearchInput('Mo'), 50);
setTimeout(() => searchService.handleSearchInput('Moh'), 100);
setTimeout(() => searchService.handleSearchInput('Moha'), 150);
setTimeout(() => searchService.handleSearchInput('Moham'), 200);
setTimeout(() => searchService.handleSearchInput('Mohame'), 250);
setTimeout(() => searchService.handleSearchInput('Mohamed'), 300);
// Search executes ~500ms after last input (at ~800ms total)

setTimeout(() => {
    console.log('📊 Result: Only 1 API call made (after user stopped typing)\n');
    console.log('='.repeat(70) + '\n');
}, 1200);

// Demo 2: Balance Updates (Throttling)
setTimeout(() => {
    console.log('2. THROTTLING - Balance Dashboard Updates:');
    console.log('WebSocket sends rapid balance updates...\n');
    
    const dashboard = new BalanceDashboard();
    
    // Simulate rapid balance changes from WebSocket
    dashboard.onBalanceChange('ACC-123456789', 50000.00);
    setTimeout(() => dashboard.onBalanceChange('ACC-123456789', 50001.50), 100);
    setTimeout(() => dashboard.onBalanceChange('ACC-123456789', 50003.25), 200);
    setTimeout(() => dashboard.onBalanceChange('ACC-123456789', 50005.00), 300);
    setTimeout(() => dashboard.onBalanceChange('ACC-123456789', 50007.75), 400);
    setTimeout(() => dashboard.onBalanceChange('ACC-123456789', 50010.00), 500);
    setTimeout(() => dashboard.onBalanceChange('ACC-123456789', 50015.00), 1100);
    setTimeout(() => dashboard.onBalanceChange('ACC-123456789', 50020.00), 1200);
    
    setTimeout(() => {
        console.log('📊 Result: UI updated only 3 times (throttled to once per second)\n');
        console.log('='.repeat(70) + '\n');
    }, 2000);
}, 1500);

// Demo 3: Form Validation (Debouncing)
setTimeout(() => {
    console.log('3. DEBOUNCING - Transfer Form Validation:');
    console.log('User types amount rapidly...\n');
    
    const validator = new TransferFormValidator();
    
    // Simulate typing amount
    validator.onAmountInput('1');
    setTimeout(() => validator.onAmountInput('15'), 50);
    setTimeout(() => validator.onAmountInput('150'), 100);
    setTimeout(() => validator.onAmountInput('1500'), 150);
    setTimeout(() => validator.onAmountInput('15000'), 200);
    // Validation executes ~300ms after last input
    
    setTimeout(() => {
        console.log('📊 Result: Validation ran only once (after user stopped typing)\n');
        console.log('='.repeat(70) + '\n');
    }, 800);
}, 3700);

// Demo 4: Scroll Events (Throttling)
setTimeout(() => {
    console.log('4. THROTTLING - Transaction List Scrolling:');
    console.log('Simulating rapid scroll events...\n');
    
    const scroller = new TransactionListScroller();
    
    // Simulate rapid scrolling (many events)
    for (let i = 0; i <= 1000; i += 50) {
        setTimeout(() => scroller.onScroll(i), i / 10);
    }
    
    setTimeout(() => {
        console.log('📊 Result: Scroll handler executed ~10 times (throttled to 100ms)\n');
        console.log('='.repeat(70) + '\n');
    }, 2000);
}, 4700);

// Demo 5: Comparison Summary
setTimeout(() => {
    console.log('5. DEBOUNCING vs THROTTLING - When to Use:\n');
    
    console.log('DEBOUNCING - Wait for user to finish:');
    console.log('  ✓ Search input (wait for user to stop typing)');
    console.log('  ✓ Form validation (validate after user completes field)');
    console.log('  ✓ Window resize (wait for resize to complete)');
    console.log('  ✓ Autosave (save after user stops editing)');
    console.log('  ✓ API calls based on user input');
    
    console.log('\nTHROTTLING - Execute at regular intervals:');
    console.log('  ✓ Scroll events (update UI periodically while scrolling)');
    console.log('  ✓ Mouse movement tracking');
    console.log('  ✓ Real-time dashboard updates');
    console.log('  ✓ API rate limiting');
    console.log('  ✓ Game loop updates');
    console.log('  ✓ Progress indicators');
    
    console.log('\nPerformance Impact:');
    console.log('  Without optimization: 100+ function calls');
    console.log('  With debouncing: 1 function call');
    console.log('  With throttling: 10-20 function calls (controlled rate)');
    
}, 7000);
```

### Key Takeaways:

| Feature | Debouncing | Throttling |
|---------|------------|------------|
| **Behavior** | Delays execution until calm period | Executes at fixed intervals |
| **Triggers** | After delay since last call | At most once per time period |
| **Best For** | User input completion | Continuous events |
| **Example** | Search after typing stops | Update UI during scroll |
| **Result** | 1 call after activity stops | N calls during activity |

**Debouncing**: "Wait for it... wait for it... NOW!"  
**Throttling**: "Do it now, then wait before doing again"

---

## Question 18: What is Memoization? How does it improve performance?

### Answer:

**Memoization** is an optimization technique that caches the results of expensive function calls and returns the cached result when the same inputs occur again. It trades memory for speed.

### Key Concepts:
1. **Cache**: Store previous results
2. **Key**: Function arguments (serialized)
3. **Hit**: Return cached result
4. **Miss**: Calculate and cache
5. **Invalidation**: Clear stale cache

### Banking Scenario: Optimizing Calculations at Emirates NBD

```javascript
// Basic Memoization Implementation
function memoize(fn) {
    const cache = new Map();
    
    return function memoized(...args) {
        // Create cache key from arguments
        const key = JSON.stringify(args);
        
        // Check if result is cached
        if (cache.has(key)) {
            console.log(`  💾 Cache HIT for ${key}`);
            return cache.get(key);
        }
        
        // Calculate and cache result
        console.log(`  🔄 Cache MISS for ${key} - calculating...`);
        const result = fn.apply(this, args);
        cache.set(key, result);
        
        return result;
    };
}

// Example 1: Memoized Interest Calculation
class InterestCalculator {
    constructor() {
        this.calculateCallCount = 0;
        
        // Memoize the expensive calculation
        this.calculateCompoundInterestMemoized = memoize(
            this.calculateCompoundInterest.bind(this)
        );
    }

    // Expensive calculation (not memoized)
    calculateCompoundInterest(principal, rate, time, frequency = 12) {
        this.calculateCallCount++;
        console.log(`[CALCULATION #${this.calculateCallCount}] Computing compound interest...`);
        
        // A = P(1 + r/n)^(nt)
        // Simulate expensive calculation with delay
        for (let i = 0; i < 1000000; i++) { /* Simulate work */ }
        
        const amount = principal * Math.pow(
            (1 + rate / frequency),
            frequency * time
        );
        
        const interest = amount - principal;
        
        return {
            principal,
            rate,
            time,
            frequency,
            totalAmount: amount,
            totalInterest: interest
        };
    }
}

// Example 2: Advanced Memoization with TTL (Time To Live)
class MemoizeWithTTL {
    constructor(ttlMs = 60000) { // Default 1 minute
        this.cache = new Map();
        this.ttlMs = ttlMs;
    }

    memoize(fn) {
        return (...args) => {
            const key = JSON.stringify(args);
            const now = Date.now();
            
            // Check if cached and not expired
            if (this.cache.has(key)) {
                const cached = this.cache.get(key);
                
                if (now - cached.timestamp < this.ttlMs) {
                    const age = Math.round((now - cached.timestamp) / 1000);
                    console.log(`  💾 Cache HIT (${age}s old) for ${key}`);
                    return cached.value;
                } else {
                    console.log(`  ⏰ Cache EXPIRED for ${key}`);
                    this.cache.delete(key);
                }
            }
            
            // Calculate and cache with timestamp
            console.log(`  🔄 Cache MISS for ${key} - calculating...`);
            const result = fn.apply(this, args);
            
            this.cache.set(key, {
                value: result,
                timestamp: now
            });
            
            return result;
        };
    }

    clear() {
        this.cache.clear();
        console.log('  🗑️  Cache cleared');
    }

    getSize() {
        return this.cache.size;
    }
}

// Example 3: Memoized Exchange Rate Calculations
class ExchangeRateService {
    constructor() {
        this.apiCallCount = 0;
        const memoizer = new MemoizeWithTTL(30000); // 30 second cache
        
        this.convertCurrencyMemoized = memoizer.memoize(
            this.convertCurrency.bind(this)
        );
    }

    // Simulated API call (expensive, should be memoized)
    convertCurrency(amount, fromCurrency, toCurrency) {
        this.apiCallCount++;
        console.log(`[API CALL #${this.apiCallCount}] Fetching exchange rate: ${fromCurrency} → ${toCurrency}`);
        
        // Simulate network delay
        const exchangeRates = {
            'AED-USD': 0.27,
            'AED-EUR': 0.25,
            'AED-GBP': 0.22,
            'USD-AED': 3.67,
            'EUR-AED': 4.00,
            'GBP-AED': 4.55
        };
        
        const rateKey = `${fromCurrency}-${toCurrency}`;
        const rate = exchangeRates[rateKey] || 1;
        
        const convertedAmount = amount * rate;
        
        return {
            originalAmount: amount,
            fromCurrency,
            toCurrency,
            exchangeRate: rate,
            convertedAmount,
            timestamp: new Date().toISOString()
        };
    }
}

// Example 4: Fibonacci with and without Memoization
class FibonacciCalculator {
    constructor() {
        this.recursiveCallCount = 0;
        this.memoizedCallCount = 0;
        this.cache = new Map();
    }

    // Without memoization - VERY slow for large n
    fibonacciRecursive(n) {
        this.recursiveCallCount++;
        
        if (n <= 1) return n;
        return this.fibonacciRecursive(n - 1) + this.fibonacciRecursive(n - 2);
    }

    // With memoization - MUCH faster
    fibonacciMemoized(n) {
        this.memoizedCallCount++;
        
        if (n <= 1) return n;
        
        if (this.cache.has(n)) {
            return this.cache.get(n);
        }
        
        const result = this.fibonacciMemoized(n - 1) + this.fibonacciMemoized(n - 2);
        this.cache.set(n, result);
        
        return result;
    }

    resetCounters() {
        this.recursiveCallCount = 0;
        this.memoizedCallCount = 0;
        this.cache.clear();
    }
}

// Example 5: LRU Cache for Account Data
class LRUCache {
    constructor(capacity) {
        this.capacity = capacity;
        this.cache = new Map();
    }

    get(key) {
        if (!this.cache.has(key)) {
            return null;
        }
        
        // Move to end (most recently used)
        const value = this.cache.get(key);
        this.cache.delete(key);
        this.cache.set(key, value);
        
        return value;
    }

    set(key, value) {
        // If key exists, remove it first
        if (this.cache.has(key)) {
            this.cache.delete(key);
        }
        
        // If at capacity, remove least recently used (first item)
        if (this.cache.size >= this.capacity) {
            const firstKey = this.cache.keys().next().value;
            this.cache.delete(firstKey);
            console.log(`  ♻️  Evicted LRU entry: ${firstKey}`);
        }
        
        // Add new entry (most recently used)
        this.cache.set(key, value);
    }

    has(key) {
        return this.cache.has(key);
    }

    size() {
        return this.cache.size;
    }
}

// Memoized Account Balance Aggregator
class AccountAggregator {
    constructor() {
        this.cache = new LRUCache(100);
        this.calculationCount = 0;
    }

    getTotalBalance(accountNumbers) {
        const cacheKey = accountNumbers.sort().join(',');
        
        // Check cache
        const cached = this.cache.get(cacheKey);
        if (cached) {
            console.log(`  💾 Retrieved from cache: ${accountNumbers.length} accounts`);
            return cached;
        }
        
        // Calculate
        this.calculationCount++;
        console.log(`  🔄 Calculating total for ${accountNumbers.length} accounts...`);
        
        const total = accountNumbers.reduce((sum, accNum) => {
            // Simulate fetching balance for each account
            const balance = Math.random() * 100000;
            return sum + balance;
        }, 0);
        
        const result = {
            accountCount: accountNumbers.length,
            totalBalance: total,
            calculatedAt: new Date().toISOString()
        };
        
        // Store in cache
        this.cache.set(cacheKey, result);
        
        return result;
    }
}

// Demonstration
console.log('=== Memoization Demo - Emirates NBD ===\n');

// Demo 1: Interest Calculation
console.log('1. Memoized Interest Calculation:\n');
const interestCalc = new InterestCalculator();

console.log('First call (cache miss):');
const result1 = interestCalc.calculateCompoundInterestMemoized(100000, 0.05, 10, 12);
console.log(`Interest: AED ${result1.totalInterest.toFixed(2)}\n`);

console.log('Second call with SAME parameters (cache hit):');
const result2 = interestCalc.calculateCompoundInterestMemoized(100000, 0.05, 10, 12);
console.log(`Interest: AED ${result2.totalInterest.toFixed(2)}\n`);

console.log('Third call with DIFFERENT parameters (cache miss):');
const result3 = interestCalc.calculateCompoundInterestMemoized(50000, 0.05, 10, 12);
console.log(`Interest: AED ${result3.totalInterest.toFixed(2)}\n`);

console.log('='.repeat(70) + '\n');

// Demo 2: Exchange Rate Conversion
console.log('2. Exchange Rate Conversion with TTL:\n');
const exchangeService = new ExchangeRateService();

console.log('Converting 10,000 AED to USD (cache miss):');
const conv1 = exchangeService.convertCurrencyMemoized(10000, 'AED', 'USD');
console.log(`Result: ${conv1.convertedAmount.toFixed(2)} ${conv1.toCurrency}\n`);

console.log('Converting 10,000 AED to USD again (cache hit):');
const conv2 = exchangeService.convertCurrencyMemoized(10000, 'AED', 'USD');
console.log(`Result: ${conv2.convertedAmount.toFixed(2)} ${conv2.toCurrency}\n`);

console.log('Converting 10,000 AED to EUR (cache miss):');
const conv3 = exchangeService.convertCurrencyMemoized(10000, 'AED', 'EUR');
console.log(`Result: ${conv3.convertedAmount.toFixed(2)} ${conv3.toCurrency}\n`);

console.log('Converting 10,000 AED to EUR again (cache hit):');
const conv4 = exchangeService.convertCurrencyMemoized(10000, 'AED', 'EUR');
console.log(`Result: ${conv4.convertedAmount.toFixed(2)} ${conv4.toCurrency}\n`);

console.log(`Total API calls made: ${exchangeService.apiCallCount} (saved 2 calls with caching)\n`);
console.log('='.repeat(70) + '\n');

// Demo 3: Fibonacci Performance Comparison
console.log('3. Fibonacci Performance Comparison:\n');
const fibCalc = new FibonacciCalculator();

console.log('Calculating fibonacci(20) WITHOUT memoization:');
const start1 = Date.now();
const fib1 = fibCalc.fibonacciRecursive(20);
const time1 = Date.now() - start1;
console.log(`Result: ${fib1}`);
console.log(`Time: ${time1}ms`);
console.log(`Function calls: ${fibCalc.recursiveCallCount}\n`);

fibCalc.resetCounters();

console.log('Calculating fibonacci(20) WITH memoization:');
const start2 = Date.now();
const fib2 = fibCalc.fibonacciMemoized(20);
const time2 = Date.now() - start2;
console.log(`Result: ${fib2}`);
console.log(`Time: ${time2}ms`);
console.log(`Function calls: ${fibCalc.memoizedCallCount}\n`);

console.log(`Performance improvement: ${(time1 / time2).toFixed(1)}x faster with memoization\n`);
console.log('='.repeat(70) + '\n');

// Demo 4: Account Aggregator with LRU Cache
console.log('4. Account Aggregator with LRU Cache:\n');
const aggregator = new AccountAggregator();

const accounts1 = ['ACC-001', 'ACC-002', 'ACC-003'];
const accounts2 = ['ACC-004', 'ACC-005'];
const accounts3 = ['ACC-001', 'ACC-002', 'ACC-003']; // Same as accounts1

console.log('First aggregation (cache miss):');
const agg1 = aggregator.getTotalBalance(accounts1);
console.log(`Total: AED ${agg1.totalBalance.toFixed(2)}\n`);

console.log('Second aggregation, different accounts (cache miss):');
const agg2 = aggregator.getTotalBalance(accounts2);
console.log(`Total: AED ${agg2.totalBalance.toFixed(2)}\n`);

console.log('Third aggregation, same as first (cache hit):');
const agg3 = aggregator.getTotalBalance(accounts3);
console.log(`Total: AED ${agg3.totalBalance.toFixed(2)}\n`);

console.log(`Calculations performed: ${aggregator.calculationCount} (1 saved by cache)\n`);
console.log('='.repeat(70) + '\n');

// Demo 5: Memoization Best Practices
console.log('5. Memoization Best Practices:\n');

console.log('✅ GOOD Use Cases:');
console.log('  • Pure functions (same input → same output)');
console.log('  • Expensive calculations (complex math, algorithms)');
console.log('  • API calls with stable data');
console.log('  • Recursive functions (fibonacci, factorial)');
console.log('  • Data transformations');

console.log('\n❌ BAD Use Cases:');
console.log('  • Functions with side effects');
console.log('  • Functions dependent on external state');
console.log('  • Random number generation');
console.log('  • Date/time-dependent calculations');
console.log('  • Very cheap functions (overhead > benefit)');

console.log('\n🔑 Key Considerations:');
console.log('  • Memory usage (cache can grow large)');
console.log('  • Cache invalidation (when to clear?)');
console.log('  • TTL (time-to-live) for stale data');
console.log('  • LRU (least recently used) eviction');
console.log('  • Serialization of complex arguments');

console.log('\n📊 Performance Impact:');
console.log('  • Time: O(1) for cache hits vs O(n) for calculation');
console.log('  • Space: O(n) for cache storage');
console.log('  • Trade-off: Memory for Speed');
```

### Key Takeaways:

1. **When to Use Memoization**:
   - Pure functions (deterministic output)
   - Expensive computations
   - Repeated calls with same arguments
   - Recursive algorithms

2. **Types of Memoization**:
   - **Simple**: Basic Map-based cache
   - **TTL**: Time-based expiration
   - **LRU**: Size-limited with eviction
   - **WeakMap**: Automatic garbage collection

3. **Performance Benefits**:
   - Fibonacci(40): ~1,000,000x faster with memoization
   - API calls: Eliminate redundant network requests
   - Complex calculations: Cache results for instant retrieval

4. **Implementation Patterns**:
   ```javascript
   // Basic memoization
   const memoized = memoize(expensiveFunction);
   
   // With TTL
   const cached = memoizeWithTTL(apiCall, 60000);
   
   // LRU cache
   const lru = new LRUCache(100);
   ```

5. **Trade-offs**:
   - **Pro**: Dramatically faster for repeated calls
   - **Pro**: Reduces server load
   - **Con**: Increased memory usage
   - **Con**: Stale data if not invalidated properly

---

**End of Questions 16-18**
