# Q43: Event-Driven Architecture - Event Emitters

## 📋 Summary
This question covers **EventEmitter** in Node.js - the foundation of event-driven architecture. You'll learn how to create custom events, handle errors, manage listeners, and build real-time banking systems using events.

**Key Topics**:
- EventEmitter class fundamentals
- Custom event creation and emission
- Event listener management
- Error handling in events
- Memory leak detection
- Event patterns (pub/sub, observer)
- Real-time banking applications

**Banking Use Cases**:
- Real-time transaction notifications
- Account balance updates
- Fraud detection alerts
- Multi-service event coordination

---

## 🎯 Understanding EventEmitter

### What is EventEmitter?

**EventEmitter** is a core Node.js class that enables **event-driven programming**. It allows objects to emit named events that cause function objects (listeners) to be called.

```
┌─────────────┐         emit('event')        ┌─────────────┐
│   Emitter   │ ─────────────────────────▶   │  Listener 1 │
│             │                               │  Listener 2 │
│  emit(...)  │ ◀─ .on('event', callback) ─  │  Listener 3 │
└─────────────┘                               └─────────────┘
```

### Key Concepts

1. **Events**: Named notifications (e.g., 'data', 'error', 'end')
2. **Emitters**: Objects that emit events
3. **Listeners**: Callback functions that respond to events
4. **Asynchronous**: Events are handled asynchronously

### Basic EventEmitter API

```javascript
const EventEmitter = require('events');

class MyEmitter extends EventEmitter {}
const emitter = new MyEmitter();

// Register listener
emitter.on('event', (data) => {
  console.log('Event occurred:', data);
});

// Emit event
emitter.emit('event', { message: 'Hello' });
```

---

## 📖 Comprehensive Explanation

### 1. EventEmitter Methods

#### Adding Listeners

```javascript
// on() - Add listener (alias: addListener)
emitter.on('event', callback);

// once() - Add one-time listener (auto-removes after first call)
emitter.once('event', callback);

// prependListener() - Add listener to beginning of array
emitter.prependListener('event', callback);

// prependOnceListener() - Add one-time listener to beginning
emitter.prependOnceListener('event', callback);
```

#### Removing Listeners

```javascript
// removeListener() - Remove specific listener (alias: off)
emitter.removeListener('event', callback);

// removeAllListeners() - Remove all listeners for event
emitter.removeAllListeners('event');

// removeAllListeners() - Remove ALL listeners
emitter.removeAllListeners();
```

#### Emitting Events

```javascript
// emit() - Trigger event with arguments
emitter.emit('event', arg1, arg2, arg3);

// Returns true if event had listeners, false otherwise
const hadListeners = emitter.emit('event');
```

#### Inspecting Listeners

```javascript
// listenerCount() - Count listeners for event
const count = emitter.listenerCount('event');

// listeners() - Get array of listeners
const listeners = emitter.listeners('event');

// eventNames() - Get array of registered event names
const events = emitter.eventNames();

// rawListeners() - Get array including wrapped listeners
const raw = emitter.rawListeners('event');
```

### 2. Error Handling

**CRITICAL**: Always handle 'error' events! Unhandled errors will crash the process.

```javascript
emitter.on('error', (error) => {
  console.error('Error occurred:', error);
});

// If no error listener is registered:
emitter.emit('error', new Error('Crash!')); // ❌ Throws and crashes
```

### 3. Memory Leak Detection

By default, EventEmitter warns if more than **10 listeners** are added to a single event (potential memory leak).

```javascript
// Get max listeners
emitter.getMaxListeners(); // 10 (default)

// Set max listeners
emitter.setMaxListeners(20);

// Set to Infinity to disable warning
emitter.setMaxListeners(Infinity);

// Set default for all emitters
EventEmitter.defaultMaxListeners = 15;
```

### 4. Event Execution Order

- Listeners execute in **registration order**
- Synchronous by default (unless you make them async)
- `prependListener()` adds to beginning of queue

```javascript
emitter.on('event', () => console.log('Second'));
emitter.prependListener('event', () => console.log('First'));
emitter.on('event', () => console.log('Third'));

emitter.emit('event');
// Output: First, Second, Third
```

### 5. Common Patterns

#### Observer Pattern
```javascript
class Subject extends EventEmitter {
  setState(state) {
    this.state = state;
    this.emit('stateChange', state);
  }
}
```

#### Pub/Sub Pattern
```javascript
class EventBus extends EventEmitter {
  publish(event, data) {
    this.emit(event, data);
  }
  
  subscribe(event, callback) {
    this.on(event, callback);
  }
}
```

---

## 💡 Example 1: Banking Transaction Event System

Complete implementation of a real-time transaction processing system with events.

### Scenario
Build a banking system where:
- Transactions emit events at each stage
- Multiple services listen to transaction events
- Fraud detection runs asynchronously
- Notifications are sent to customers
- Audit logs record all activities

### Implementation

```javascript
const EventEmitter = require('events');

/**
 * TransactionProcessor - Core banking transaction system
 * Emits events at each transaction stage
 */
class TransactionProcessor extends EventEmitter {
  constructor() {
    super();
    this.transactions = new Map();
    this.dailyLimit = 10000;
    this.dailyTotal = 0;
  }

  /**
   * Process a payment transaction
   * Emits: transaction:initiated, transaction:validated, 
   *        transaction:completed, transaction:failed
   */
  async processTransaction(transactionData) {
    const transaction = {
      id: `TXN-${Date.now()}`,
      ...transactionData,
      status: 'initiated',
      timestamp: new Date(),
      steps: []
    };

    try {
      // Step 1: Initiate
      this.emit('transaction:initiated', transaction);
      transaction.steps.push({ stage: 'initiated', timestamp: new Date() });

      // Step 2: Validate
      await this.validateTransaction(transaction);
      this.emit('transaction:validated', transaction);
      transaction.steps.push({ stage: 'validated', timestamp: new Date() });

      // Step 3: Check fraud
      await this.checkFraud(transaction);
      this.emit('transaction:fraud-checked', transaction);
      transaction.steps.push({ stage: 'fraud-checked', timestamp: new Date() });

      // Step 4: Process payment
      await this.executePayment(transaction);
      transaction.status = 'completed';
      this.emit('transaction:completed', transaction);
      transaction.steps.push({ stage: 'completed', timestamp: new Date() });

      this.transactions.set(transaction.id, transaction);
      return transaction;

    } catch (error) {
      transaction.status = 'failed';
      transaction.error = error.message;
      
      // Emit error event (specific + general)
      this.emit('transaction:failed', transaction, error);
      this.emit('error', error, transaction);
      
      throw error;
    }
  }

  async validateTransaction(transaction) {
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        // Validation rules
        if (!transaction.amount || transaction.amount <= 0) {
          return reject(new Error('Invalid amount'));
        }

        if (transaction.amount > 10000) {
          return reject(new Error('Amount exceeds single transaction limit'));
        }

        if (this.dailyTotal + transaction.amount > this.dailyLimit) {
          return reject(new Error('Daily limit exceeded'));
        }

        if (!transaction.fromAccount || !transaction.toAccount) {
          return reject(new Error('Missing account information'));
        }

        resolve();
      }, 100); // Simulate async validation
    });
  }

  async checkFraud(transaction) {
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        // Fraud detection logic
        const riskScore = Math.random() * 100;
        transaction.riskScore = riskScore;

        if (riskScore > 90) {
          // Emit fraud alert
          this.emit('fraud:detected', transaction, riskScore);
          return reject(new Error(`Fraud detected (risk: ${riskScore.toFixed(2)})`));
        }

        if (riskScore > 70) {
          // Emit high-risk warning
          this.emit('fraud:high-risk', transaction, riskScore);
        }

        resolve();
      }, 150);
    });
  }

  async executePayment(transaction) {
    return new Promise((resolve) => {
      setTimeout(() => {
        // Simulate payment execution
        this.dailyTotal += transaction.amount;
        transaction.confirmationNumber = `CONF-${Date.now()}`;
        resolve();
      }, 200);
    });
  }

  // Get transaction statistics
  getStats() {
    const stats = {
      total: this.transactions.size,
      completed: 0,
      failed: 0,
      totalAmount: 0
    };

    for (const txn of this.transactions.values()) {
      if (txn.status === 'completed') {
        stats.completed++;
        stats.totalAmount += txn.amount;
      } else if (txn.status === 'failed') {
        stats.failed++;
      }
    }

    return stats;
  }
}

/**
 * FraudDetectionService - Monitors transactions for fraud
 * Subscribes to transaction events
 */
class FraudDetectionService extends EventEmitter {
  constructor(processor) {
    super();
    this.processor = processor;
    this.alerts = [];
    this.blockedAccounts = new Set();

    // Subscribe to transaction events
    this.setupListeners();
  }

  setupListeners() {
    // Monitor all initiated transactions
    this.processor.on('transaction:initiated', (txn) => {
      this.logActivity('Transaction initiated', txn);
    });

    // Check for high-risk patterns
    this.processor.on('fraud:high-risk', (txn, riskScore) => {
      const alert = {
        id: `ALERT-${Date.now()}`,
        type: 'high-risk',
        transaction: txn,
        riskScore,
        timestamp: new Date()
      };

      this.alerts.push(alert);
      this.emit('alert:high-risk', alert);
      
      console.log(`⚠️  HIGH RISK ALERT: Transaction ${txn.id} (risk: ${riskScore.toFixed(2)})`);
    });

    // Handle detected fraud
    this.processor.on('fraud:detected', (txn, riskScore) => {
      const alert = {
        id: `ALERT-${Date.now()}`,
        type: 'fraud-detected',
        transaction: txn,
        riskScore,
        timestamp: new Date()
      };

      this.alerts.push(alert);
      this.emit('alert:fraud', alert);
      
      // Block account
      this.blockedAccounts.add(txn.fromAccount);
      this.emit('account:blocked', txn.fromAccount);
      
      console.log(`🚨 FRAUD DETECTED: Transaction ${txn.id} (risk: ${riskScore.toFixed(2)})`);
      console.log(`🔒 Account ${txn.fromAccount} has been blocked`);
    });
  }

  logActivity(message, txn) {
    console.log(`[Fraud Service] ${message}: ${txn.id} ($${txn.amount})`);
  }

  getAlerts() {
    return this.alerts;
  }

  isAccountBlocked(accountNumber) {
    return this.blockedAccounts.has(accountNumber);
  }
}

/**
 * NotificationService - Sends customer notifications
 * Subscribes to transaction events
 */
class NotificationService {
  constructor(processor) {
    this.processor = processor;
    this.notifications = [];
    this.setupListeners();
  }

  setupListeners() {
    // Notify on transaction completion
    this.processor.on('transaction:completed', (txn) => {
      this.sendNotification({
        type: 'transaction-completed',
        recipient: txn.fromAccount,
        message: `Transaction ${txn.id} completed successfully`,
        amount: txn.amount,
        timestamp: new Date()
      });
    });

    // Notify on transaction failure
    this.processor.on('transaction:failed', (txn, error) => {
      this.sendNotification({
        type: 'transaction-failed',
        recipient: txn.fromAccount,
        message: `Transaction ${txn.id} failed: ${error.message}`,
        amount: txn.amount,
        timestamp: new Date()
      });
    });

    // Notify on fraud detection
    this.processor.on('fraud:detected', (txn) => {
      this.sendNotification({
        type: 'fraud-alert',
        recipient: txn.fromAccount,
        message: `🚨 Suspicious activity detected on your account`,
        urgent: true,
        timestamp: new Date()
      });
    });
  }

  sendNotification(notification) {
    this.notifications.push(notification);
    
    const emoji = notification.type === 'fraud-alert' ? '🚨' : 
                  notification.type === 'transaction-failed' ? '❌' : '✅';
    
    console.log(`${emoji} [Notification] ${notification.recipient}: ${notification.message}`);
  }

  getNotifications(recipient) {
    return this.notifications.filter(n => n.recipient === recipient);
  }
}

/**
 * AuditLogger - Records all transaction activities
 * Subscribes to all events for compliance
 */
class AuditLogger {
  constructor(processor, fraudService) {
    this.processor = processor;
    this.fraudService = fraudService;
    this.logs = [];
    this.setupListeners();
  }

  setupListeners() {
    // Log all transaction events
    const txnEvents = [
      'transaction:initiated',
      'transaction:validated',
      'transaction:fraud-checked',
      'transaction:completed',
      'transaction:failed'
    ];

    txnEvents.forEach(event => {
      this.processor.on(event, (txn, ...args) => {
        this.log(event, txn, args);
      });
    });

    // Log fraud alerts
    this.fraudService.on('alert:high-risk', (alert) => {
      this.log('fraud:high-risk-alert', alert);
    });

    this.fraudService.on('alert:fraud', (alert) => {
      this.log('fraud:fraud-detected', alert);
    });

    this.fraudService.on('account:blocked', (accountNumber) => {
      this.log('account:blocked', { accountNumber });
    });

    // Catch all errors
    this.processor.on('error', (error, txn) => {
      this.log('error', { error: error.message, transaction: txn });
    });
  }

  log(event, data, extraArgs = []) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      event,
      data,
      extraArgs
    };

    this.logs.push(logEntry);
    
    // In production, this would write to a file or database
    console.log(`[Audit] ${event}`, JSON.stringify(data, null, 2));
  }

  getLogs(eventFilter) {
    if (!eventFilter) return this.logs;
    return this.logs.filter(log => log.event.includes(eventFilter));
  }

  getLogCount() {
    return this.logs.length;
  }
}

// ============================================
// Usage Example
// ============================================

console.log('=== Banking Transaction Event System ===\n');

// Initialize services
const processor = new TransactionProcessor();
const fraudService = new FraudDetectionService(processor);
const notificationService = new NotificationService(processor);
const auditLogger = new AuditLogger(processor, fraudService);

// Process multiple transactions
async function runTransactions() {
  const transactions = [
    {
      fromAccount: 'ACC-123456',
      toAccount: 'ACC-789012',
      amount: 500,
      description: 'Payment for services'
    },
    {
      fromAccount: 'ACC-123456',
      toAccount: 'ACC-345678',
      amount: 1500,
      description: 'Rent payment'
    },
    {
      fromAccount: 'ACC-654321',
      toAccount: 'ACC-111111',
      amount: 8000,
      description: 'Large transfer (may trigger fraud check)'
    },
    {
      fromAccount: 'ACC-999999',
      toAccount: 'ACC-888888',
      amount: -100,
      description: 'Invalid amount'
    }
  ];

  console.log('Processing transactions...\n');

  for (const txnData of transactions) {
    try {
      console.log(`\n--- Transaction: ${txnData.description} ---`);
      const result = await processor.processTransaction(txnData);
      console.log(`✅ Success: ${result.id} (Confirmation: ${result.confirmationNumber})`);
    } catch (error) {
      console.log(`❌ Failed: ${error.message}`);
    }
    
    // Small delay between transactions
    await new Promise(resolve => setTimeout(resolve, 100));
  }

  // Print final statistics
  console.log('\n=== Final Statistics ===');
  const stats = processor.getStats();
  console.log(`Total transactions: ${stats.total}`);
  console.log(`Completed: ${stats.completed}`);
  console.log(`Failed: ${stats.failed}`);
  console.log(`Total amount processed: $${stats.totalAmount.toFixed(2)}`);

  console.log(`\n=== Fraud Alerts ===`);
  console.log(`Total alerts: ${fraudService.getAlerts().length}`);
  console.log(`Blocked accounts: ${fraudService.blockedAccounts.size}`);

  console.log(`\n=== Audit Logs ===`);
  console.log(`Total log entries: ${auditLogger.getLogCount()}`);
  
  console.log(`\n=== Notifications Sent ===`);
  console.log(`Total notifications: ${notificationService.notifications.length}`);
}

// Run the demo
runTransactions().catch(console.error);
```

### Key Takeaways from Example 1

1. **Event Naming**: Use descriptive, hierarchical names (`transaction:completed`, `fraud:detected`)
2. **Multiple Listeners**: Multiple services can listen to same events
3. **Error Handling**: Always have error listeners on EventEmitters
4. **Async Events**: Event handlers can be async functions
5. **Event Data**: Pass relevant data as event arguments
6. **Loose Coupling**: Services communicate through events, not direct calls

---

## 💡 Example 2: Real-Time Account Balance Updates

Building a pub/sub system for real-time account updates across multiple clients.

### Scenario
Create a system where:
- Account balance changes emit events
- Multiple clients subscribe to balance updates
- Events propagate to all interested parties
- Memory leaks are prevented with proper cleanup

### Implementation

```javascript
const EventEmitter = require('events');

/**
 * AccountManager - Manages bank accounts with real-time events
 */
class AccountManager extends EventEmitter {
  constructor() {
    super();
    this.accounts = new Map();
    
    // Increase max listeners (many clients may subscribe)
    this.setMaxListeners(100);
  }

  createAccount(accountNumber, initialBalance = 0) {
    if (this.accounts.has(accountNumber)) {
      throw new Error('Account already exists');
    }

    const account = {
      accountNumber,
      balance: initialBalance,
      transactions: [],
      createdAt: new Date(),
      status: 'active'
    };

    this.accounts.set(accountNumber, account);
    
    // Emit account creation event
    this.emit('account:created', account);
    this.emit(`account:${accountNumber}:created`, account);

    return account;
  }

  deposit(accountNumber, amount) {
    const account = this.accounts.get(accountNumber);
    if (!account) {
      throw new Error('Account not found');
    }

    if (amount <= 0) {
      throw new Error('Amount must be positive');
    }

    const previousBalance = account.balance;
    account.balance += amount;

    const transaction = {
      id: `TXN-${Date.now()}`,
      type: 'deposit',
      amount,
      previousBalance,
      newBalance: account.balance,
      timestamp: new Date()
    };

    account.transactions.push(transaction);

    // Emit balance update events
    this.emit('balance:updated', accountNumber, account.balance, previousBalance);
    this.emit(`account:${accountNumber}:balance`, account.balance, previousBalance);
    this.emit(`account:${accountNumber}:deposit`, transaction);

    return account;
  }

  withdraw(accountNumber, amount) {
    const account = this.accounts.get(accountNumber);
    if (!account) {
      throw new Error('Account not found');
    }

    if (amount <= 0) {
      throw new Error('Amount must be positive');
    }

    if (amount > account.balance) {
      // Emit insufficient funds event
      this.emit('account:insufficient-funds', accountNumber, amount, account.balance);
      throw new Error('Insufficient funds');
    }

    const previousBalance = account.balance;
    account.balance -= amount;

    const transaction = {
      id: `TXN-${Date.now()}`,
      type: 'withdrawal',
      amount,
      previousBalance,
      newBalance: account.balance,
      timestamp: new Date()
    };

    account.transactions.push(transaction);

    // Emit balance update events
    this.emit('balance:updated', accountNumber, account.balance, previousBalance);
    this.emit(`account:${accountNumber}:balance`, account.balance, previousBalance);
    this.emit(`account:${accountNumber}:withdrawal`, transaction);

    // Check for low balance
    if (account.balance < 100) {
      this.emit('account:low-balance', accountNumber, account.balance);
      this.emit(`account:${accountNumber}:low-balance`, account.balance);
    }

    return account;
  }

  transfer(fromAccount, toAccount, amount) {
    // Withdraw from source
    this.withdraw(fromAccount, amount);
    
    // Deposit to destination
    this.deposit(toAccount, amount);

    // Emit transfer event
    this.emit('transfer:completed', {
      from: fromAccount,
      to: toAccount,
      amount,
      timestamp: new Date()
    });

    return { from: fromAccount, to: toAccount, amount };
  }

  getAccount(accountNumber) {
    return this.accounts.get(accountNumber);
  }

  getBalance(accountNumber) {
    const account = this.accounts.get(accountNumber);
    return account ? account.balance : null;
  }
}

/**
 * AccountSubscriber - Client that subscribes to account updates
 */
class AccountSubscriber {
  constructor(name, accountManager) {
    this.name = name;
    this.accountManager = accountManager;
    this.subscriptions = new Map(); // Track subscriptions for cleanup
  }

  // Subscribe to specific account
  subscribeToAccount(accountNumber) {
    const balanceHandler = (newBalance, previousBalance) => {
      console.log(`[${this.name}] Account ${accountNumber} balance changed: $${previousBalance} → $${newBalance}`);
    };

    const lowBalanceHandler = (balance) => {
      console.log(`[${this.name}] ⚠️  LOW BALANCE ALERT for ${accountNumber}: $${balance}`);
    };

    // Store handlers for later cleanup
    this.subscriptions.set(accountNumber, {
      balanceHandler,
      lowBalanceHandler
    });

    // Subscribe to events
    this.accountManager.on(`account:${accountNumber}:balance`, balanceHandler);
    this.accountManager.on(`account:${accountNumber}:low-balance`, lowBalanceHandler);

    console.log(`[${this.name}] Subscribed to account ${accountNumber}`);
  }

  // Unsubscribe from specific account (prevent memory leaks!)
  unsubscribeFromAccount(accountNumber) {
    const handlers = this.subscriptions.get(accountNumber);
    if (!handlers) {
      console.log(`[${this.name}] Not subscribed to ${accountNumber}`);
      return;
    }

    // Remove listeners
    this.accountManager.removeListener(`account:${accountNumber}:balance`, handlers.balanceHandler);
    this.accountManager.removeListener(`account:${accountNumber}:low-balance`, handlers.lowBalanceHandler);

    this.subscriptions.delete(accountNumber);
    console.log(`[${this.name}] Unsubscribed from account ${accountNumber}`);
  }

  // Subscribe to all balance updates
  subscribeToAllBalanceUpdates() {
    const globalHandler = (accountNumber, newBalance, previousBalance) => {
      console.log(`[${this.name}] Global update - ${accountNumber}: $${previousBalance} → $${newBalance}`);
    };

    this.accountManager.on('balance:updated', globalHandler);
    this.subscriptions.set('__global__', { globalHandler });

    console.log(`[${this.name}] Subscribed to all balance updates`);
  }

  // Cleanup all subscriptions
  cleanup() {
    for (const [accountNumber, handlers] of this.subscriptions) {
      if (accountNumber === '__global__') {
        this.accountManager.removeListener('balance:updated', handlers.globalHandler);
      } else {
        this.accountManager.removeListener(`account:${accountNumber}:balance`, handlers.balanceHandler);
        this.accountManager.removeListener(`account:${accountNumber}:low-balance`, handlers.lowBalanceHandler);
      }
    }

    this.subscriptions.clear();
    console.log(`[${this.name}] All subscriptions cleaned up`);
  }
}

/**
 * DashboardMonitor - Monitors all account activity
 */
class DashboardMonitor {
  constructor(accountManager) {
    this.accountManager = accountManager;
    this.stats = {
      deposits: 0,
      withdrawals: 0,
      transfers: 0,
      lowBalanceAlerts: 0
    };

    this.setupMonitoring();
  }

  setupMonitoring() {
    // Monitor all balance updates
    this.accountManager.on('balance:updated', (accountNumber, newBalance, previousBalance) => {
      const change = newBalance - previousBalance;
      if (change > 0) {
        this.stats.deposits++;
      } else {
        this.stats.withdrawals++;
      }
    });

    // Monitor transfers
    this.accountManager.on('transfer:completed', (transfer) => {
      this.stats.transfers++;
      console.log(`[Dashboard] Transfer: ${transfer.from} → ${transfer.to} ($${transfer.amount})`);
    });

    // Monitor low balance alerts
    this.accountManager.on('account:low-balance', (accountNumber, balance) => {
      this.stats.lowBalanceAlerts++;
      console.log(`[Dashboard] 🔴 Low balance alert: ${accountNumber} ($${balance})`);
    });

    // Monitor insufficient funds
    this.accountManager.on('account:insufficient-funds', (accountNumber, attempted, available) => {
      console.log(`[Dashboard] ❌ Insufficient funds: ${accountNumber} tried $${attempted}, has $${available}`);
    });
  }

  getStats() {
    return { ...this.stats };
  }
}

// ============================================
// Usage Example
// ============================================

console.log('\n=== Real-Time Account Balance Updates ===\n');

// Create account manager
const accountManager = new AccountManager();

// Create accounts
console.log('Creating accounts...');
accountManager.createAccount('ACC-001', 1000);
accountManager.createAccount('ACC-002', 500);
accountManager.createAccount('ACC-003', 50);

console.log('\n--- Setting up subscribers ---\n');

// Create subscribers (e.g., mobile app, web dashboard, email service)
const mobileApp = new AccountSubscriber('Mobile App', accountManager);
const webDashboard = new AccountSubscriber('Web Dashboard', accountManager);
const emailService = new AccountSubscriber('Email Service', accountManager);

// Subscribe to specific accounts
mobileApp.subscribeToAccount('ACC-001');
webDashboard.subscribeToAccount('ACC-001');
webDashboard.subscribeToAccount('ACC-002');
emailService.subscribeToAccount('ACC-003');

// Dashboard monitors everything
const dashboard = new DashboardMonitor(accountManager);

console.log('\n--- Performing transactions ---\n');

// Perform transactions
console.log('Transaction 1: Deposit $200 to ACC-001');
accountManager.deposit('ACC-001', 200);

console.log('\nTransaction 2: Withdraw $300 from ACC-002');
accountManager.withdraw('ACC-002', 300);

console.log('\nTransaction 3: Withdraw $30 from ACC-003 (will trigger low balance alert)');
accountManager.withdraw('ACC-003', 30);

console.log('\nTransaction 4: Transfer $100 from ACC-001 to ACC-002');
accountManager.transfer('ACC-001', 'ACC-002', 100);

console.log('\nTransaction 5: Try to withdraw $5000 from ACC-003 (insufficient funds)');
try {
  accountManager.withdraw('ACC-003', 5000);
} catch (error) {
  console.log(`Error: ${error.message}`);
}

console.log('\n--- Unsubscribing mobile app ---\n');
mobileApp.unsubscribeFromAccount('ACC-001');

console.log('\nTransaction 6: Deposit $50 to ACC-001 (mobile app won\'t see this)');
accountManager.deposit('ACC-001', 50);

console.log('\n--- Final Statistics ---\n');
const stats = dashboard.getStats();
console.log('Dashboard Stats:', stats);

console.log('\n--- Account Balances ---\n');
console.log('ACC-001:', `$${accountManager.getBalance('ACC-001')}`);
console.log('ACC-002:', `$${accountManager.getBalance('ACC-002')}`);
console.log('ACC-003:', `$${accountManager.getBalance('ACC-003')}`);

console.log('\n--- Cleanup ---\n');
webDashboard.cleanup();
emailService.cleanup();

console.log('\n--- Listener Count (Memory Leak Check) ---\n');
console.log('balance:updated listeners:', accountManager.listenerCount('balance:updated'));
console.log('ACC-001:balance listeners:', accountManager.listenerCount('account:ACC-001:balance'));
console.log('Total events registered:', accountManager.eventNames().length);
```

### Key Takeaways from Example 2

1. **Specific Events**: Use account-specific events (`account:${accountNumber}:balance`)
2. **Memory Management**: Always clean up listeners when done
3. **Listener Limits**: Increase `maxListeners` for pub/sub systems
4. **Event Naming**: Hierarchical naming helps organization
5. **Multiple Subscribers**: Many clients can listen to same events
6. **Cleanup Pattern**: Store handlers for proper removal

---

## 🎯 Key Concepts & Best Practices

### 1. Event Naming Conventions

```javascript
// ✅ Good naming (descriptive, hierarchical)
'transaction:completed'
'account:balance:updated'
'fraud:high-risk:detected'
'user:login:success'

// ❌ Avoid generic names
'done'
'update'
'change'
```

### 2. Error Handling Pattern

```javascript
class SafeEmitter extends EventEmitter {
  constructor() {
    super();
    
    // Always set up error handler
    this.on('error', (error) => {
      console.error('Emitter error:', error);
    });
  }
  
  safeEmit(event, ...args) {
    try {
      return this.emit(event, ...args);
    } catch (error) {
      this.emit('error', error);
      return false;
    }
  }
}
```

### 3. Memory Leak Prevention

```javascript
// ❌ Memory leak - listeners never removed
function badPattern() {
  setInterval(() => {
    emitter.on('event', () => {
      // New listener every second!
    });
  }, 1000);
}

// ✅ Good - one-time listener
function goodPattern() {
  emitter.once('event', () => {
    // Auto-removed after first call
  });
}

// ✅ Good - proper cleanup
function goodPattern2() {
  const handler = () => { /* ... */ };
  emitter.on('event', handler);
  
  // Later...
  emitter.removeListener('event', handler);
}
```

### 4. Async Event Handlers

```javascript
// Event handlers can be async
emitter.on('data', async (data) => {
  await processData(data);
  await saveToDatabase(data);
});

// But emit() doesn't wait for async handlers!
emitter.emit('data', someData);
console.log('This runs immediately!');

// To wait for async handlers, use Promise.all
async function emitAndWait(emitter, event, data) {
  const listeners = emitter.listeners(event);
  await Promise.all(listeners.map(listener => listener(data)));
}
```

### 5. Once vs On

```javascript
// on() - Fires every time
emitter.on('transaction', (txn) => {
  console.log('Transaction:', txn.id);
});

// once() - Fires only once, then auto-removes
emitter.once('connection', (conn) => {
  console.log('Connected once:', conn.id);
});

// Manual removal
const handler = (data) => console.log(data);
emitter.on('event', handler);
emitter.removeListener('event', handler);
```

### 6. Event Propagation (Inheritance)

```javascript
class ChildEmitter extends EventEmitter {
  constructor() {
    super();
  }
  
  doSomething() {
    this.emit('action', 'child');
    
    // Events don't bubble to parent automatically
    // You must explicitly emit to parent if needed
  }
}

const child = new ChildEmitter();
child.on('action', (source) => {
  console.log('Action from:', source);
});
```

---

## 🚀 Performance Considerations

### 1. Listener Count

```javascript
// Check listener count to detect potential issues
if (emitter.listenerCount('event') > 50) {
  console.warn('High listener count detected');
}
```

### 2. Event Payload Size

```javascript
// ❌ Don't emit large objects repeatedly
emitter.emit('data', { huge: largeArray });

// ✅ Emit references or IDs
emitter.emit('data:ready', dataId);
// Listener fetches data if needed
```

### 3. Synchronous vs Asynchronous

```javascript
// Events are synchronous by default
emitter.on('sync', () => {
  console.log('2');
});

console.log('1');
emitter.emit('sync');
console.log('3');
// Output: 1, 2, 3

// Make async if needed
emitter.on('async', async () => {
  await someAsyncOperation();
});
```

---

## 📚 Common Interview Questions

### Q1: What happens if you emit an error event with no listeners?

**Answer**: The process will crash with an uncaught exception. Always add error listeners:

```javascript
emitter.on('error', (err) => {
  console.error('Handled error:', err);
});
```

### Q2: How do you prevent memory leaks with EventEmitters?

**Answer**:
1. Use `once()` for one-time listeners
2. Always `removeListener()` when done
3. Set appropriate `maxListeners`
4. Monitor listener counts in production

### Q3: What's the difference between `on()` and `once()`?

**Answer**:
- `on()`: Listener persists and fires every time
- `once()`: Listener fires once then auto-removes

### Q4: Are event handlers synchronous or asynchronous?

**Answer**: Event handlers execute **synchronously** by default in the order they were registered. However, handlers themselves can be async functions - but `emit()` doesn't wait for them.

### Q5: How do you pass multiple arguments to event listeners?

**Answer**: Add them as additional parameters to `emit()`:

```javascript
emitter.on('transfer', (from, to, amount, currency) => {
  console.log(`${from} → ${to}: ${amount} ${currency}`);
});

emitter.emit('transfer', 'ACC-1', 'ACC-2', 100, 'USD');
```

---

## ✅ Summary & Key Takeaways

### Core Concepts

1. **EventEmitter**: Foundation of event-driven programming in Node.js
2. **Events**: Named notifications that trigger callback functions
3. **Listeners**: Functions that respond to events
4. **Async by Nature**: Perfect for real-time, reactive systems

### Critical Patterns

1. **Always handle errors**: Use `.on('error', callback)`
2. **Clean up listeners**: Prevent memory leaks with proper removal
3. **Descriptive naming**: Use hierarchical event names
4. **Loose coupling**: Services communicate via events, not direct calls

### Banking Applications

1. **Transaction Processing**: Multi-stage transaction workflows
2. **Real-Time Updates**: Balance changes propagate to clients
3. **Fraud Detection**: Async monitoring without blocking
4. **Audit Logging**: Centralized event logging for compliance
5. **Notifications**: Customer alerts on transaction events

### Best Practices

```javascript
// ✅ DO
- Use descriptive event names ('transaction:completed')
- Handle 'error' events
- Remove listeners when done
- Use once() for one-time events
- Set appropriate maxListeners

// ❌ DON'T
- Forget error handlers (will crash!)
- Create listeners in loops without cleanup
- Use generic event names ('done', 'update')
- Emit huge payloads repeatedly
- Assume emit() waits for async handlers
```

### Interview Readiness

You should be able to explain:
- How EventEmitter works internally
- When to use events vs callbacks vs promises
- Memory leak prevention strategies
- Error handling in event-driven systems
- Real-world use cases in banking/finance

### Next Steps

- **Q44**: Message Queues (RabbitMQ, Kafka) for distributed events
- **Q45**: Serverless event-driven architectures (AWS Lambda)
- **Q47**: WebSockets for real-time client communication

---

**Status**: ✅ Complete with production-ready banking examples!
