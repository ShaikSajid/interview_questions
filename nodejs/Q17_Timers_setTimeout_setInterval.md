# Q17: Timers - setTimeout/setInterval

## 📋 Summary
This question covers **Node.js timers** (`setTimeout`, `setInterval`, `setImmediate`, `clearTimeout`, `clearInterval`) and their integration with the event loop. We'll build production-ready banking examples including interest calculation schedulers, payment reminders, session timeout handlers, and scheduled transaction processors.

---

## 🎯 What You'll Learn
- How `setTimeout` and `setInterval` work in Node.js
- Timer integration with the event loop phases
- Differences between timers and other async operations
- Timer precision and limitations
- Memory management with timers
- Banking Examples: Interest calculation scheduler, payment reminders, recurring transactions

---

## 📖 Detailed Explanation

### What are Timers in Node.js?

**Timers** schedule code execution after a specified delay or at regular intervals. Node.js provides global timer functions similar to browsers, but with different internal implementations.

### Timer Functions

#### 1. **setTimeout(callback, delay, ...args)**
Executes `callback` **once** after `delay` milliseconds.

```javascript
setTimeout(() => {
  console.log('Executed after 2 seconds');
}, 2000);
```

**Returns**: Timer object (can be used to cancel)

#### 2. **setInterval(callback, delay, ...args)**
Executes `callback` **repeatedly** every `delay` milliseconds.

```javascript
const intervalId = setInterval(() => {
  console.log('Executed every 1 second');
}, 1000);
```

**Returns**: Timer object (can be used to cancel)

#### 3. **clearTimeout(timeoutId)**
Cancels a timer created by `setTimeout`.

```javascript
const timeoutId = setTimeout(() => {
  console.log('This will not run');
}, 5000);

clearTimeout(timeoutId); // Cancels the timer
```

#### 4. **clearInterval(intervalId)**
Cancels a timer created by `setInterval`.

```javascript
const intervalId = setInterval(() => {
  console.log('Repeating...');
}, 1000);

setTimeout(() => {
  clearInterval(intervalId); // Stop after 5 seconds
}, 5000);
```

---

### How Timers Work with the Event Loop

Timers are processed in the **Timers Phase** of the event loop.

#### Event Loop Timer Flow:

```
┌───────────────────────────┐
│     Timers Phase          │ ← setTimeout/setInterval callbacks execute here
│  (poll expired timers)    │
└───────────────────────────┘
           │
           ↓
┌───────────────────────────┐
│   Pending Callbacks       │
└───────────────────────────┘
           │
           ↓
┌───────────────────────────┐
│      Idle, Prepare        │
└───────────────────────────┘
           │
           ↓
┌───────────────────────────┐
│         Poll              │ ← Waits for I/O events
└───────────────────────────┘
           │
           ↓
┌───────────────────────────┐
│         Check             │ ← setImmediate executes here
└───────────────────────────┘
           │
           ↓
┌───────────────────────────┐
│    Close Callbacks        │
└───────────────────────────┘
```

#### Key Points:

1. **Timers are NOT guaranteed to execute exactly at the specified time**
   - They execute **as soon as possible** after the delay
   - Other operations can block timer execution

2. **Timer Resolution**
   - Node.js checks timers in the Timers Phase
   - Minimum effective delay is ~1ms (though you can specify 0)

3. **Execution Order**
   - Timers with the same delay execute in the order they were created
   - Timers are processed in batches during each event loop iteration

---

### Timer Precision and Limitations

#### Problem: Timers are NOT Precise

```javascript
const start = Date.now();

setTimeout(() => {
  const elapsed = Date.now() - start;
  console.log(`Elapsed: ${elapsed}ms`); // May be 1005ms, not exactly 1000ms
}, 1000);

// If the event loop is blocked, timer executes late
let sum = 0;
for (let i = 0; i < 1000000000; i++) {
  sum += i; // Blocks event loop
}
```

**Output:**
```
Elapsed: 1247ms  // Not 1000ms because event loop was blocked
```

#### Why Timers Drift:

1. **Event Loop Blocking**: Synchronous code blocks timer execution
2. **System Load**: Heavy CPU usage delays timers
3. **JavaScript Single-Thread**: Only one thing runs at a time

---

### setTimeout vs setInterval

#### setTimeout - Guaranteed Delay Between Executions

```javascript
function scheduleNext() {
  console.log('Executed');
  
  setTimeout(scheduleNext, 1000); // Schedule next execution
}

setTimeout(scheduleNext, 1000);
```

**Delay**: 1000ms **BETWEEN** executions (execution time not included)

```
Execute -> Wait 1000ms -> Execute -> Wait 1000ms -> Execute
```

#### setInterval - Fixed Interval (May Overlap)

```javascript
setInterval(() => {
  console.log('Executed');
}, 1000);
```

**Problem**: If execution takes longer than interval, callbacks queue up

```
Start Interval (0ms)
Execute (0-500ms) - takes 500ms
Execute (1000ms) - on time
Execute (1500ms) - queued because previous still running
```

**Best Practice**: Use recursive `setTimeout` instead of `setInterval` for most use cases.

---

### Memory Management with Timers

#### Problem: Timers Can Cause Memory Leaks

```javascript
// ❌ BAD: Timer never cleared
function startLeakyTimer() {
  const data = new Array(1000000).fill('leak');
  
  setInterval(() => {
    console.log('Data:', data[0]); // Keeps 'data' in memory forever
  }, 1000);
}

startLeakyTimer();
```

#### Solution: Always Clear Timers

```javascript
// ✅ GOOD: Clear timers when done
function startSafeTimer() {
  const data = new Array(1000000).fill('safe');
  
  const intervalId = setInterval(() => {
    console.log('Data:', data[0]);
  }, 1000);
  
  // Clear after 10 seconds
  setTimeout(() => {
    clearInterval(intervalId);
  }, 10000);
}
```

---

## 🏦 Banking Scenario

**GlobalBank** needs to implement several scheduled operations:

### Requirements:
1. **Interest Calculation**: Calculate and apply interest to savings accounts daily at midnight
2. **Payment Reminders**: Send payment reminders to customers with upcoming loan due dates
3. **Recurring Transactions**: Process recurring payments (subscriptions, loan EMIs)
4. **Session Timeout**: Expire user sessions after 15 minutes of inactivity
5. **Statement Generation**: Generate monthly account statements on the 1st of each month
6. **Overdraft Alerts**: Check for overdrafts every 30 seconds and send alerts

### Challenges:
- Ensure timers don't drift over long periods
- Handle server restarts (timers are not persistent)
- Prevent memory leaks from long-running timers
- Coordinate timer execution across multiple server instances
- Handle timezone differences for global operations

---

## 💻 Example 1: Interest Calculator with Daily Scheduler

This example demonstrates a production-ready interest calculation system that runs daily at midnight.

### Implementation:

```javascript
const EventEmitter = require('events');

/**
 * Banking Interest Calculator
 * Calculates and applies interest to savings accounts daily
 */
class InterestCalculator extends EventEmitter {
  constructor() {
    super();
    this.accounts = [
      { id: 'ACC001', type: 'savings', balance: 10000, interestRate: 0.04 },
      { id: 'ACC002', type: 'savings', balance: 50000, interestRate: 0.05 },
      { id: 'ACC003', type: 'checking', balance: 2000, interestRate: 0.01 },
      { id: 'ACC004', type: 'savings', balance: 100000, interestRate: 0.06 }
    ];
    
    this.dailyTimerId = null;
    this.lastCalculation = null;
    this.running = false;
  }

  /**
   * Calculate daily interest for an account
   * Formula: (balance * annual_rate) / 365
   */
  calculateDailyInterest(account) {
    if (account.type !== 'savings') {
      return 0;
    }

    const dailyRate = account.interestRate / 365;
    const interest = account.balance * dailyRate;
    
    return Math.round(interest * 100) / 100; // Round to 2 decimals
  }

  /**
   * Apply interest to all eligible accounts
   */
  applyInterestToAllAccounts() {
    console.log('\n==============================================');
    console.log(`🏦 Interest Calculation Started at ${new Date().toISOString()}`);
    console.log('==============================================\n');

    const results = [];

    for (const account of this.accounts) {
      const interest = this.calculateDailyInterest(account);
      
      if (interest > 0) {
        const oldBalance = account.balance;
        account.balance += interest;
        
        const result = {
          accountId: account.id,
          accountType: account.type,
          oldBalance,
          interest,
          newBalance: account.balance,
          interestRate: account.interestRate,
          timestamp: new Date().toISOString()
        };

        results.push(result);

        console.log(`Account: ${account.id}`);
        console.log(`  Type: ${account.type}`);
        console.log(`  Old Balance: $${oldBalance.toFixed(2)}`);
        console.log(`  Interest Earned: $${interest.toFixed(2)}`);
        console.log(`  New Balance: $${account.balance.toFixed(2)}`);
        console.log(`  Annual Rate: ${(account.interestRate * 100).toFixed(2)}%`);
        console.log('');
      }
    }

    this.lastCalculation = new Date();

    // Emit event for logging/monitoring
    this.emit('interestApplied', {
      timestamp: this.lastCalculation,
      accountsProcessed: results.length,
      totalInterest: results.reduce((sum, r) => sum + r.interest, 0),
      results
    });

    console.log('==============================================');
    console.log(`✅ Interest Calculation Completed`);
    console.log(`   Accounts Processed: ${results.length}`);
    console.log(`   Total Interest: $${results.reduce((sum, r) => sum + r.interest, 0).toFixed(2)}`);
    console.log('==============================================\n');

    return results;
  }

  /**
   * Calculate milliseconds until next midnight
   */
  getMillisecondsUntilMidnight() {
    const now = new Date();
    const tomorrow = new Date(now);
    tomorrow.setDate(tomorrow.getDate() + 1);
    tomorrow.setHours(0, 0, 0, 0);
    
    return tomorrow - now;
  }

  /**
   * Schedule next interest calculation at midnight
   */
  scheduleNextCalculation() {
    const msUntilMidnight = this.getMillisecondsUntilMidnight();
    const nextRun = new Date(Date.now() + msUntilMidnight);

    console.log(`⏰ Next interest calculation scheduled for: ${nextRun.toISOString()}`);
    console.log(`   (in ${Math.round(msUntilMidnight / 1000 / 60)} minutes)\n`);

    // Clear existing timer if any
    if (this.dailyTimerId) {
      clearTimeout(this.dailyTimerId);
    }

    // Schedule next calculation
    this.dailyTimerId = setTimeout(() => {
      this.applyInterestToAllAccounts();
      
      // Schedule the next one (recursive)
      this.scheduleNextCalculation();
    }, msUntilMidnight);
  }

  /**
   * Start the interest calculator service
   */
  start() {
    if (this.running) {
      console.log('⚠️  Interest calculator already running');
      return;
    }

    console.log('\n🚀 Starting Interest Calculator Service...\n');
    
    this.running = true;
    this.scheduleNextCalculation();
    
    this.emit('started', { timestamp: new Date() });
  }

  /**
   * Stop the interest calculator service
   */
  stop() {
    if (!this.running) {
      console.log('⚠️  Interest calculator not running');
      return;
    }

    console.log('\n🛑 Stopping Interest Calculator Service...\n');

    if (this.dailyTimerId) {
      clearTimeout(this.dailyTimerId);
      this.dailyTimerId = null;
    }

    this.running = false;
    this.emit('stopped', { timestamp: new Date() });
  }

  /**
   * Get service status
   */
  getStatus() {
    return {
      running: this.running,
      lastCalculation: this.lastCalculation,
      nextCalculation: this.dailyTimerId ? 
        new Date(Date.now() + this.getMillisecondsUntilMidnight()) : null,
      accountCount: this.accounts.length,
      totalBalance: this.accounts.reduce((sum, acc) => sum + acc.balance, 0)
    };
  }

  /**
   * Manual trigger (for testing or end-of-day processing)
   */
  runNow() {
    console.log('🔧 Manual interest calculation triggered\n');
    return this.applyInterestToAllAccounts();
  }
}

// ============================================
// USAGE EXAMPLE
// ============================================

const calculator = new InterestCalculator();

// Event listeners for monitoring
calculator.on('started', (data) => {
  console.log('✅ Calculator started at:', data.timestamp);
});

calculator.on('stopped', (data) => {
  console.log('✅ Calculator stopped at:', data.timestamp);
});

calculator.on('interestApplied', (data) => {
  console.log('📊 Interest applied event:', {
    accountsProcessed: data.accountsProcessed,
    totalInterest: data.totalInterest
  });
});

// Start the service
calculator.start();

// Display status
console.log('📊 Current Status:', calculator.getStatus());

// For testing: Run immediately (comment out for production)
// setTimeout(() => {
//   console.log('\n--- TESTING: Running calculation now ---');
//   calculator.runNow();
// }, 2000);

// Graceful shutdown on SIGINT (Ctrl+C)
process.on('SIGINT', () => {
  console.log('\n\n📥 Received SIGINT, shutting down gracefully...\n');
  calculator.stop();
  process.exit(0);
});

// Keep process running
console.log('\n💡 Press Ctrl+C to stop the service\n');

module.exports = InterestCalculator;
```

### Testing Commands:

```bash
# Run the interest calculator
node interest-calculator.js

# Expected output:
# 🚀 Starting Interest Calculator Service...
# 
# ⏰ Next interest calculation scheduled for: 2024-01-16T00:00:00.000Z
#    (in 847 minutes)
# 
# 📊 Current Status: {
#   running: true,
#   lastCalculation: null,
#   nextCalculation: 2024-01-16T00:00:00.000Z,
#   accountCount: 4,
#   totalBalance: 162000
# }
```

### Testing with Immediate Execution:

```javascript
// test-interest-calculator.js
const InterestCalculator = require('./interest-calculator');

const calculator = new InterestCalculator();

// Override getMillisecondsUntilMidnight for testing
calculator.getMillisecondsUntilMidnight = () => 3000; // Run every 3 seconds

calculator.start();

// Stop after 15 seconds
setTimeout(() => {
  calculator.stop();
  console.log('\n✅ Test completed');
  process.exit(0);
}, 15000);
```

### Expected Output:

```
🚀 Starting Interest Calculator Service...

⏰ Next interest calculation scheduled for: 2024-01-15T14:32:03.000Z
   (in 0 minutes)

==============================================
🏦 Interest Calculation Started at 2024-01-15T14:32:03.000Z
==============================================

Account: ACC001
  Type: savings
  Old Balance: $10000.00
  Interest Earned: $1.10
  New Balance: $10001.10
  Annual Rate: 4.00%

Account: ACC002
  Type: savings
  Old Balance: $50000.00
  Interest Earned: $6.85
  New Balance: $50006.85
  Annual Rate: 5.00%

Account: ACC004
  Type: savings
  Old Balance: $100000.00
  Interest Earned: $16.44
  New Balance: $100016.44
  Annual Rate: 6.00%

==============================================
✅ Interest Calculation Completed
   Accounts Processed: 3
   Total Interest: $24.38
==============================================
```

---

## 💻 Example 2: Recurring Transaction Processor with setInterval

This example shows how to process recurring transactions (subscriptions, loan EMIs) using timers with proper error handling and retry logic.

### Implementation:

```javascript
const EventEmitter = require('events');

/**
 * Recurring Transaction Processor
 * Handles scheduled payments like subscriptions, loan EMIs, etc.
 */
class RecurringTransactionProcessor extends EventEmitter {
  constructor() {
    super();
    
    // Mock database of recurring transactions
    this.recurringTransactions = [
      {
        id: 'REC001',
        accountId: 'ACC001',
        type: 'subscription',
        amount: 9.99,
        frequency: 'monthly',
        nextDue: new Date('2024-01-20'),
        payee: 'Netflix',
        status: 'active'
      },
      {
        id: 'REC002',
        accountId: 'ACC002',
        type: 'loan_emi',
        amount: 500,
        frequency: 'monthly',
        nextDue: new Date('2024-01-15'),
        payee: 'Home Loan',
        status: 'active'
      },
      {
        id: 'REC003',
        accountId: 'ACC001',
        type: 'subscription',
        amount: 14.99,
        frequency: 'monthly',
        nextDue: new Date('2024-01-18'),
        payee: 'Spotify',
        status: 'active'
      }
    ];

    // Mock account balances
    this.accounts = {
      'ACC001': { balance: 1000 },
      'ACC002': { balance: 5000 },
      'ACC003': { balance: 500 }
    };

    this.checkIntervalId = null;
    this.checkFrequency = 60 * 1000; // Check every 60 seconds
    this.running = false;
    this.processedToday = [];
  }

  /**
   * Check if transaction is due for processing
   */
  isDue(transaction) {
    const now = new Date();
    return transaction.nextDue <= now && transaction.status === 'active';
  }

  /**
   * Calculate next due date based on frequency
   */
  calculateNextDueDate(currentDue, frequency) {
    const nextDue = new Date(currentDue);
    
    switch (frequency) {
      case 'daily':
        nextDue.setDate(nextDue.getDate() + 1);
        break;
      case 'weekly':
        nextDue.setDate(nextDue.getDate() + 7);
        break;
      case 'monthly':
        nextDue.setMonth(nextDue.getMonth() + 1);
        break;
      case 'yearly':
        nextDue.setFullYear(nextDue.getFullYear() + 1);
        break;
      default:
        throw new Error(`Unknown frequency: ${frequency}`);
    }
    
    return nextDue;
  }

  /**
   * Process a single recurring transaction
   */
  async processTransaction(transaction) {
    console.log(`\n💳 Processing recurring transaction: ${transaction.id}`);
    console.log(`   Account: ${transaction.accountId}`);
    console.log(`   Type: ${transaction.type}`);
    console.log(`   Amount: $${transaction.amount}`);
    console.log(`   Payee: ${transaction.payee}`);

    const account = this.accounts[transaction.accountId];

    if (!account) {
      console.log(`   ❌ Error: Account not found`);
      return {
        success: false,
        transactionId: transaction.id,
        error: 'Account not found'
      };
    }

    // Check sufficient balance
    if (account.balance < transaction.amount) {
      console.log(`   ❌ Error: Insufficient funds`);
      console.log(`      Balance: $${account.balance}`);
      console.log(`      Required: $${transaction.amount}`);
      
      this.emit('transactionFailed', {
        transactionId: transaction.id,
        reason: 'insufficient_funds',
        balance: account.balance,
        required: transaction.amount
      });

      return {
        success: false,
        transactionId: transaction.id,
        error: 'Insufficient funds',
        willRetry: true
      };
    }

    // Simulate processing delay
    await new Promise(resolve => setTimeout(resolve, 500));

    // Deduct amount
    account.balance -= transaction.amount;

    // Update next due date
    const oldDueDate = transaction.nextDue;
    transaction.nextDue = this.calculateNextDueDate(transaction.nextDue, transaction.frequency);

    console.log(`   ✅ Success: Transaction processed`);
    console.log(`      New Balance: $${account.balance}`);
    console.log(`      Next Due: ${transaction.nextDue.toISOString()}`);

    const result = {
      success: true,
      transactionId: transaction.id,
      accountId: transaction.accountId,
      amount: transaction.amount,
      payee: transaction.payee,
      oldBalance: account.balance + transaction.amount,
      newBalance: account.balance,
      oldDueDate,
      nextDueDate: transaction.nextDue,
      processedAt: new Date()
    };

    this.processedToday.push(result);

    this.emit('transactionProcessed', result);

    return result;
  }

  /**
   * Check and process all due transactions
   */
  async checkAndProcessDueTransactions() {
    console.log(`\n⏰ Checking for due transactions at ${new Date().toISOString()}`);

    const dueTransactions = this.recurringTransactions.filter(t => this.isDue(t));

    if (dueTransactions.length === 0) {
      console.log('   No transactions due at this time');
      return [];
    }

    console.log(`   Found ${dueTransactions.length} due transaction(s)`);

    const results = [];

    for (const transaction of dueTransactions) {
      try {
        const result = await this.processTransaction(transaction);
        results.push(result);
      } catch (error) {
        console.error(`   ❌ Error processing ${transaction.id}:`, error.message);
        
        this.emit('processingError', {
          transactionId: transaction.id,
          error: error.message
        });

        results.push({
          success: false,
          transactionId: transaction.id,
          error: error.message
        });
      }
    }

    return results;
  }

  /**
   * Start the recurring transaction processor
   */
  start() {
    if (this.running) {
      console.log('⚠️  Processor already running');
      return;
    }

    console.log('\n🚀 Starting Recurring Transaction Processor...');
    console.log(`   Check Frequency: Every ${this.checkFrequency / 1000} seconds\n`);

    this.running = true;

    // Run immediately on start
    this.checkAndProcessDueTransactions();

    // Then run periodically
    this.checkIntervalId = setInterval(() => {
      this.checkAndProcessDueTransactions();
    }, this.checkFrequency);

    this.emit('started', { timestamp: new Date() });
  }

  /**
   * Stop the processor
   */
  stop() {
    if (!this.running) {
      console.log('⚠️  Processor not running');
      return;
    }

    console.log('\n🛑 Stopping Recurring Transaction Processor...\n');

    if (this.checkIntervalId) {
      clearInterval(this.checkIntervalId);
      this.checkIntervalId = null;
    }

    this.running = false;
    this.emit('stopped', { timestamp: new Date() });
  }

  /**
   * Get processor status
   */
  getStatus() {
    return {
      running: this.running,
      totalRecurringTransactions: this.recurringTransactions.length,
      activeTransactions: this.recurringTransactions.filter(t => t.status === 'active').length,
      processedToday: this.processedToday.length,
      checkFrequency: `${this.checkFrequency / 1000}s`,
      accounts: Object.entries(this.accounts).map(([id, acc]) => ({
        accountId: id,
        balance: acc.balance
      }))
    };
  }

  /**
   * Get upcoming transactions
   */
  getUpcomingTransactions(days = 7) {
    const now = new Date();
    const futureDate = new Date(now.getTime() + days * 24 * 60 * 60 * 1000);

    return this.recurringTransactions
      .filter(t => t.status === 'active' && t.nextDue <= futureDate)
      .sort((a, b) => a.nextDue - b.nextDue)
      .map(t => ({
        id: t.id,
        accountId: t.accountId,
        type: t.type,
        amount: t.amount,
        payee: t.payee,
        nextDue: t.nextDue,
        daysUntilDue: Math.ceil((t.nextDue - now) / (24 * 60 * 60 * 1000))
      }));
  }
}

// ============================================
// USAGE EXAMPLE
// ============================================

const processor = new RecurringTransactionProcessor();

// Event listeners
processor.on('transactionProcessed', (data) => {
  console.log('\n📊 Event: Transaction processed successfully');
  // Here you could send email notification, update database, etc.
});

processor.on('transactionFailed', (data) => {
  console.log('\n⚠️  Event: Transaction failed');
  console.log('   Reason:', data.reason);
  // Here you could send alert, schedule retry, etc.
});

// For testing: Set due dates to now
processor.recurringTransactions.forEach(t => {
  t.nextDue = new Date(Date.now() + 2000); // 2 seconds from now
});

// Set check frequency to 5 seconds for testing
processor.checkFrequency = 5000;

// Start processor
processor.start();

// Display status
setTimeout(() => {
  console.log('\n📊 Processor Status:', JSON.stringify(processor.getStatus(), null, 2));
  
  console.log('\n📅 Upcoming Transactions:');
  console.table(processor.getUpcomingTransactions());
}, 3000);

// Graceful shutdown
process.on('SIGINT', () => {
  console.log('\n\n📥 Received SIGINT, shutting down...\n');
  processor.stop();
  
  console.log('\n📊 Final Status:');
  console.log('   Transactions processed today:', processor.processedToday.length);
  console.log('   Total amount processed:', 
    `$${processor.processedToday.reduce((sum, t) => sum + t.amount, 0).toFixed(2)}`);
  
  process.exit(0);
});

console.log('\n💡 Press Ctrl+C to stop\n');

module.exports = RecurringTransactionProcessor;
```

### Testing Commands:

```bash
# Run the recurring transaction processor
node recurring-processor.js

# Expected output shows:
# - Initial check for due transactions
# - Processing of each transaction
# - Updates to account balances
# - Rescheduling of next due dates
# - Periodic checks every 5 seconds (in test mode)
```

### Expected Output:

```
🚀 Starting Recurring Transaction Processor...
   Check Frequency: Every 5 seconds

⏰ Checking for due transactions at 2024-01-15T14:45:00.000Z
   Found 3 due transaction(s)

💳 Processing recurring transaction: REC001
   Account: ACC001
   Type: subscription
   Amount: $9.99
   Payee: Netflix
   ✅ Success: Transaction processed
      New Balance: $990.01
      Next Due: 2024-02-20T00:00:00.000Z

💳 Processing recurring transaction: REC002
   Account: ACC002
   Type: loan_emi
   Amount: $500
   Payee: Home Loan
   ✅ Success: Transaction processed
      New Balance: $4500
      Next Due: 2024-02-15T00:00:00.000Z

💳 Processing recurring transaction: REC003
   Account: ACC001
   Type: subscription
   Amount: $14.99
   Payee: Spotify
   ✅ Success: Transaction processed
      New Balance: $975.02
      Next Due: 2024-02-18T00:00:00.000Z

📊 Processor Status: {
  "running": true,
  "totalRecurringTransactions": 3,
  "activeTransactions": 3,
  "processedToday": 3,
  "checkFrequency": "5s",
  "accounts": [
    { "accountId": "ACC001", "balance": 975.02 },
    { "accountId": "ACC002", "balance": 4500 },
    { "accountId": "ACC003", "balance": 500 }
  ]
}
```

---

## 💻 Example 3: Session Timeout Manager with Cleanup

This example demonstrates session management with automatic timeouts and periodic cleanup of expired sessions.

### Implementation:

```javascript
const EventEmitter = require('events');
const crypto = require('crypto');

/**
 * Banking Session Manager
 * Manages user sessions with automatic timeouts and cleanup
 */
class SessionManager extends EventEmitter {
  constructor(options = {}) {
    super();
    
    this.sessions = new Map(); // sessionId -> session data
    this.userSessions = new Map(); // userId -> Set of sessionIds
    
    this.sessionTimeout = options.sessionTimeout || 15 * 60 * 1000; // 15 minutes default
    this.cleanupInterval = options.cleanupInterval || 5 * 60 * 1000; // 5 minutes default
    this.maxSessionsPerUser = options.maxSessionsPerUser || 3;
    
    this.cleanupTimerId = null;
    this.running = false;
    
    this.stats = {
      created: 0,
      expired: 0,
      destroyed: 0,
      cleanupRuns: 0
    };
  }

  /**
   * Generate unique session ID
   */
  generateSessionId() {
    return crypto.randomBytes(32).toString('hex');
  }

  /**
   * Create a new session
   */
  createSession(userId, metadata = {}) {
    const sessionId = this.generateSessionId();
    
    const session = {
      sessionId,
      userId,
      createdAt: new Date(),
      lastActivity: new Date(),
      expiresAt: new Date(Date.now() + this.sessionTimeout),
      metadata,
      timeoutId: null
    };

    // Set automatic timeout
    session.timeoutId = setTimeout(() => {
      this.expireSession(sessionId, 'timeout');
    }, this.sessionTimeout);

    // Store session
    this.sessions.set(sessionId, session);

    // Track user sessions
    if (!this.userSessions.has(userId)) {
      this.userSessions.set(userId, new Set());
    }
    this.userSessions.get(userId).add(sessionId);

    // Enforce max sessions per user
    const userSessionList = Array.from(this.userSessions.get(userId));
    if (userSessionList.length > this.maxSessionsPerUser) {
      // Remove oldest session
      const oldestSessionId = userSessionList[0];
      this.destroySession(oldestSessionId, 'max_sessions_exceeded');
    }

    this.stats.created++;

    this.emit('sessionCreated', {
      sessionId,
      userId,
      expiresAt: session.expiresAt
    });

    console.log(`✅ Session created: ${sessionId}`);
    console.log(`   User: ${userId}`);
    console.log(`   Expires: ${session.expiresAt.toISOString()}`);
    console.log(`   Timeout: ${this.sessionTimeout / 1000}s\n`);

    return sessionId;
  }

  /**
   * Get session by ID
   */
  getSession(sessionId) {
    const session = this.sessions.get(sessionId);
    
    if (!session) {
      return null;
    }

    // Check if expired
    if (new Date() > session.expiresAt) {
      this.expireSession(sessionId, 'expired');
      return null;
    }

    return session;
  }

  /**
   * Refresh session (extend timeout)
   */
  refreshSession(sessionId) {
    const session = this.sessions.get(sessionId);
    
    if (!session) {
      console.log(`⚠️  Session not found: ${sessionId}`);
      return false;
    }

    // Clear old timeout
    if (session.timeoutId) {
      clearTimeout(session.timeoutId);
    }

    // Update session
    session.lastActivity = new Date();
    session.expiresAt = new Date(Date.now() + this.sessionTimeout);

    // Set new timeout
    session.timeoutId = setTimeout(() => {
      this.expireSession(sessionId, 'timeout');
    }, this.sessionTimeout);

    console.log(`🔄 Session refreshed: ${sessionId}`);
    console.log(`   New expiry: ${session.expiresAt.toISOString()}\n`);

    this.emit('sessionRefreshed', {
      sessionId,
      userId: session.userId,
      expiresAt: session.expiresAt
    });

    return true;
  }

  /**
   * Expire a session (timeout or manual)
   */
  expireSession(sessionId, reason = 'timeout') {
    const session = this.sessions.get(sessionId);
    
    if (!session) {
      return false;
    }

    console.log(`⏰ Session expired: ${sessionId}`);
    console.log(`   Reason: ${reason}`);
    console.log(`   User: ${session.userId}`);
    console.log(`   Duration: ${Math.round((Date.now() - session.createdAt) / 1000)}s\n`);

    // Clear timeout
    if (session.timeoutId) {
      clearTimeout(session.timeoutId);
    }

    // Remove from storage
    this.sessions.delete(sessionId);
    
    // Remove from user sessions
    if (this.userSessions.has(session.userId)) {
      this.userSessions.get(session.userId).delete(sessionId);
      
      if (this.userSessions.get(session.userId).size === 0) {
        this.userSessions.delete(session.userId);
      }
    }

    this.stats.expired++;

    this.emit('sessionExpired', {
      sessionId,
      userId: session.userId,
      reason,
      duration: Date.now() - session.createdAt
    });

    return true;
  }

  /**
   * Destroy a session (logout)
   */
  destroySession(sessionId, reason = 'logout') {
    const session = this.sessions.get(sessionId);
    
    if (!session) {
      return false;
    }

    console.log(`🗑️  Session destroyed: ${sessionId}`);
    console.log(`   Reason: ${reason}`);
    console.log(`   User: ${session.userId}\n`);

    // Clear timeout
    if (session.timeoutId) {
      clearTimeout(session.timeoutId);
    }

    // Remove from storage
    this.sessions.delete(sessionId);
    
    // Remove from user sessions
    if (this.userSessions.has(session.userId)) {
      this.userSessions.get(session.userId).delete(sessionId);
      
      if (this.userSessions.get(session.userId).size === 0) {
        this.userSessions.delete(session.userId);
      }
    }

    this.stats.destroyed++;

    this.emit('sessionDestroyed', {
      sessionId,
      userId: session.userId,
      reason
    });

    return true;
  }

  /**
   * Cleanup expired sessions (runs periodically)
   */
  cleanupExpiredSessions() {
    console.log(`\n🧹 Running cleanup at ${new Date().toISOString()}`);
    
    const now = new Date();
    let cleaned = 0;

    for (const [sessionId, session] of this.sessions.entries()) {
      if (now > session.expiresAt) {
        this.expireSession(sessionId, 'cleanup');
        cleaned++;
      }
    }

    this.stats.cleanupRuns++;

    console.log(`   Cleaned up ${cleaned} expired session(s)`);
    console.log(`   Active sessions: ${this.sessions.size}\n`);

    this.emit('cleanupCompleted', {
      cleanedSessions: cleaned,
      activeSessions: this.sessions.size
    });
  }

  /**
   * Start periodic cleanup
   */
  startCleanup() {
    if (this.running) {
      console.log('⚠️  Cleanup already running');
      return;
    }

    console.log('\n🚀 Starting Session Manager with Cleanup');
    console.log(`   Session Timeout: ${this.sessionTimeout / 1000}s`);
    console.log(`   Cleanup Interval: ${this.cleanupInterval / 1000}s\n`);

    this.running = true;

    // Run cleanup periodically
    this.cleanupTimerId = setInterval(() => {
      this.cleanupExpiredSessions();
    }, this.cleanupInterval);

    this.emit('started');
  }

  /**
   * Stop cleanup
   */
  stopCleanup() {
    if (!this.running) {
      console.log('⚠️  Cleanup not running');
      return;
    }

    console.log('\n🛑 Stopping Session Manager\n');

    if (this.cleanupTimerId) {
      clearInterval(this.cleanupTimerId);
      this.cleanupTimerId = null;
    }

    // Clear all session timeouts
    for (const session of this.sessions.values()) {
      if (session.timeoutId) {
        clearTimeout(session.timeoutId);
      }
    }

    this.running = false;
    this.emit('stopped');
  }

  /**
   * Get manager statistics
   */
  getStats() {
    return {
      activeSessions: this.sessions.size,
      activeUsers: this.userSessions.size,
      stats: { ...this.stats },
      config: {
        sessionTimeout: `${this.sessionTimeout / 1000}s`,
        cleanupInterval: `${this.cleanupInterval / 1000}s`,
        maxSessionsPerUser: this.maxSessionsPerUser
      }
    };
  }

  /**
   * Get all active sessions for a user
   */
  getUserSessions(userId) {
    const sessionIds = this.userSessions.get(userId);
    
    if (!sessionIds) {
      return [];
    }

    return Array.from(sessionIds).map(id => this.sessions.get(id));
  }
}

// ============================================
// USAGE EXAMPLE & TESTING
// ============================================

// Create manager with short timeouts for testing
const manager = new SessionManager({
  sessionTimeout: 10 * 1000,      // 10 seconds
  cleanupInterval: 5 * 1000,       // 5 seconds  
  maxSessionsPerUser: 2
});

// Event listeners
manager.on('sessionCreated', (data) => {
  console.log('📊 Event: Session created');
});

manager.on('sessionExpired', (data) => {
  console.log(`📊 Event: Session expired (${data.reason})`);
});

manager.on('sessionDestroyed', (data) => {
  console.log(`📊 Event: Session destroyed (${data.reason})`);
});

manager.on('cleanupCompleted', (data) => {
  console.log(`📊 Event: Cleanup completed (${data.cleanedSessions} cleaned)`);
});

// Start cleanup
manager.startCleanup();

// Create test sessions
console.log('--- Creating test sessions ---\n');

const session1 = manager.createSession('user123', { ip: '192.168.1.1', device: 'mobile' });
const session2 = manager.createSession('user456', { ip: '192.168.1.2', device: 'desktop' });
const session3 = manager.createSession('user123', { ip: '192.168.1.3', device: 'tablet' });

// Refresh session1 after 5 seconds
setTimeout(() => {
  console.log('--- Refreshing session1 ---\n');
  manager.refreshSession(session1);
}, 5000);

// Create another session for user123 (should remove oldest)
setTimeout(() => {
  console.log('--- Creating 3rd session for user123 (exceeds max) ---\n');
  manager.createSession('user123', { ip: '192.168.1.4', device: 'laptop' });
}, 7000);

// Display stats periodically
const statsInterval = setInterval(() => {
  console.log('\n📊 Current Stats:', JSON.stringify(manager.getStats(), null, 2), '\n');
}, 8000);

// Cleanup and exit after 30 seconds
setTimeout(() => {
  clearInterval(statsInterval);
  manager.stopCleanup();
  
  console.log('\n📊 Final Stats:');
  console.table(manager.getStats());
  
  process.exit(0);
}, 30000);

console.log('\n💡 Sessions will timeout after 10 seconds of inactivity\n');
console.log('💡 Cleanup runs every 5 seconds\n');

module.exports = SessionManager;
```

### Testing Commands:

```bash
# Run the session manager
node session-manager.js

# The test will:
# 1. Create 3 sessions
# 2. Refresh one session after 5s
# 3. Create another session that exceeds max (removes oldest)
# 4. Run cleanup every 5s
# 5. Show stats every 8s
# 6. Exit after 30s
```

### Expected Output:

```
🚀 Starting Session Manager with Cleanup
   Session Timeout: 10s
   Cleanup Interval: 5s

--- Creating test sessions ---

✅ Session created: a1b2c3...
   User: user123
   Expires: 2024-01-15T14:50:10.000Z
   Timeout: 10s

✅ Session created: d4e5f6...
   User: user456
   Expires: 2024-01-15T14:50:10.000Z
   Timeout: 10s

✅ Session created: g7h8i9...
   User: user123
   Expires: 2024-01-15T14:50:10.000Z
   Timeout: 10s

🧹 Running cleanup at 2024-01-15T14:50:05.000Z
   Cleaned up 0 expired session(s)
   Active sessions: 3

--- Refreshing session1 ---

🔄 Session refreshed: a1b2c3...
   New expiry: 2024-01-15T14:50:15.000Z

--- Creating 3rd session for user123 (exceeds max) ---

🗑️  Session destroyed: a1b2c3...
   Reason: max_sessions_exceeded
   User: user123

✅ Session created: j1k2l3...
   User: user123
   Expires: 2024-01-15T14:50:17.000Z
   Timeout: 10s

⏰ Session expired: d4e5f6...
   Reason: timeout
   User: user456
   Duration: 10s

⏰ Session expired: g7h8i9...
   Reason: timeout
   User: user123
   Duration: 10s

📊 Current Stats: {
  "activeSessions": 1,
  "activeUsers": 1,
  "stats": {
    "created": 4,
    "expired": 2,
    "destroyed": 1,
    "cleanupRuns": 3
  }
}
```

---

## 📊 Interview Questions & Answers

### Q1: What is the difference between setTimeout and setInterval?

**Answer:**

**`setTimeout`** executes a callback **once** after a delay:
```javascript
setTimeout(() => {
  console.log('Executes once after 2 seconds');
}, 2000);
```

**`setInterval`** executes a callback **repeatedly** at fixed intervals:
```javascript
setInterval(() => {
  console.log('Executes every 2 seconds');
}, 2000);
```

**Key Differences:**

| Feature | setTimeout | setInterval |
|---------|-----------|-------------|
| **Executions** | Once | Repeatedly |
| **Delay Guarantee** | After callback completes | Fixed interval (may overlap) |
| **Use Case** | Delayed execution, retry logic | Polling, periodic tasks |
| **Memory** | Auto-cleaned after execution | Must manually clear |

**Important Problem with setInterval:**

If the callback takes longer than the interval, callbacks can **queue up**:

```javascript
// ❌ Problem: Long-running callback with setInterval
setInterval(() => {
  // This takes 3 seconds
  heavyComputation(); // 3000ms
}, 1000); // Interval is 1000ms

// Result: Callbacks queue up and execute back-to-back
```

**Solution: Use Recursive setTimeout:**

```javascript
// ✅ Better: Recursive setTimeout ensures gap between executions
function scheduleNext() {
  heavyComputation(); // Takes 3000ms
  
  setTimeout(scheduleNext, 1000); // 1000ms AFTER completion
}

setTimeout(scheduleNext, 1000);
```

**Banking Example:**
```javascript
// ❌ BAD: setInterval for transaction processing
setInterval(async () => {
  await processPendingTransactions(); // May take 5 seconds
}, 1000); // Runs every 1s, can overlap!

// ✅ GOOD: Recursive setTimeout
async function processTransactions() {
  await processPendingTransactions();
  
  setTimeout(processTransactions, 1000); // 1s after completion
}

setTimeout(processTransactions, 1000);
```

---

### Q2: Are timers guaranteed to execute at the exact specified time?

**Answer:**

**No, timers are NOT guaranteed to execute exactly at the specified time.**

**Why:**

1. **Event Loop Blocking**: If JavaScript is busy executing code, timers wait
2. **Timer Phase**: Timers only execute during the Timers Phase of the event loop
3. **System Load**: Heavy CPU usage can delay timer execution

**Example:**

```javascript
console.log('Start:', Date.now());

setTimeout(() => {
  console.log('Timer executed:', Date.now());
  console.log('Expected: 1000ms');
}, 1000);

// Block event loop for 2 seconds
const start = Date.now();
while (Date.now() - start < 2000) {
  // Blocking operation
}

console.log('Blocking finished:', Date.now());
```

**Output:**
```
Start: 1642345678000
Blocking finished: 1642345680000
Timer executed: 1642345680001
Expected: 1000ms
// Timer executed at ~2000ms, not 1000ms!
```

**Timers Specify MINIMUM Delay:**
- `setTimeout(fn, 1000)` means "execute at least 1000ms from now"
- Could execute at 1000ms, 1005ms, 1500ms, etc. depending on event loop state

**Banking Impact:**
```javascript
// Interest calculation scheduled for midnight
const msUntilMidnight = calculateMsUntilMidnight();

setTimeout(() => {
  applyDailyInterest(); // May run at 00:00:00 or 00:00:03
}, msUntilMidnight);

// For critical banking operations, use:
// 1. Cron jobs (node-cron, bull queue)
// 2. Database triggers
// 3. Separate scheduler service
```

---

### Q3: How do timers interact with the event loop?

**Answer:**

Timers execute in the **Timers Phase** of the event loop.

**Event Loop Phases:**

```
   ┌───────────────────────┐
┌─→│       timers          │ ← setTimeout, setInterval execute here
│  └───────────┬───────────┘
│  ┌───────────┴───────────┐
│  │  pending callbacks    │
│  └───────────┬───────────┘
│  ┌───────────┴───────────┐
│  │    idle, prepare      │
│  └───────────┬───────────┘
│  ┌───────────┴───────────┐
│  │        poll           │ ← Waits for I/O events
│  └───────────┬───────────┘
│  ┌───────────┴───────────┐
│  │        check          │ ← setImmediate executes here
│  └───────────┬───────────┘
│  ┌───────────┴───────────┐
│  │   close callbacks     │
│  └───────────┬───────────┘
└──────────────┘
```

**Timer Execution Flow:**

1. **Timer is set**: Stored in heap with expiration time
2. **Event loop enters Timers Phase**: Checks for expired timers
3. **Execute callbacks**: All expired timer callbacks run
4. **Move to next phase**: Continue event loop

**Example:**

```javascript
console.log('1. Start');

setTimeout(() => {
  console.log('3. Timer 1 (0ms)');
}, 0);

setTimeout(() => {
  console.log('4. Timer 2 (0ms)');
}, 0);

setImmediate(() => {
  console.log('5. setImmediate');
});

console.log('2. End');

// Output:
// 1. Start
// 2. End
// 3. Timer 1 (0ms)
// 4. Timer 2 (0ms)
// 5. setImmediate
```

**Why This Order:**
1. Synchronous code runs first
2. Event loop starts
3. Timers Phase: setTimeout callbacks execute
4. Check Phase: setImmediate callback executes

**Banking Example:**
```javascript
console.log('Processing transaction...');

// 1. Synchronous validation
validateTransaction(txn); // Runs immediately

// 2. Asynchronous database operation
db.saveTransaction(txn).then(() => {
  console.log('Saved to DB'); // Runs in poll phase (I/O callback)
});

// 3. Schedule followup action
setTimeout(() => {
  console.log('Send confirmation email'); // Runs in next timers phase
}, 1000);

// 4. Immediate followup
setImmediate(() => {
  console.log('Log to audit trail'); // Runs in check phase
});

// Order: validate -> log -> saved -> email
```

---

### Q4: What happens if you set timeout delay to 0?

**Answer:**

`setTimeout(fn, 0)` does **NOT execute immediately**. It schedules the callback to run **as soon as possible** in the next event loop iteration.

**Example:**

```javascript
console.log('1');

setTimeout(() => {
  console.log('3');
}, 0);

console.log('2');

// Output: 1, 2, 3
// NOT: 1, 3, 2
```

**Why:**

1. Synchronous code executes first (call stack must be empty)
2. Event loop then processes timers
3. Timer callback executes

**Use Cases:**

**1. Defer Execution:**
```javascript
function heavyTask() {
  // Do heavy computation
  for (let i = 0; i < 1000000000; i++) {}
}

// ❌ BAD: Blocks immediately
heavyTask();

// ✅ GOOD: Defers to next event loop
setTimeout(() => {
  heavyTask();
}, 0);

// Other code can run before heavy task
console.log('This runs first');
```

**2. Break Up Long Operations:**
```javascript
function processLargeArray(arr) {
  const chunkSize = 1000;
  let index = 0;

  function processChunk() {
    const end = Math.min(index + chunkSize, arr.length);
    
    for (let i = index; i < end; i++) {
      // Process item
      processItem(arr[i]);
    }

    index = end;

    if (index < arr.length) {
      setTimeout(processChunk, 0); // Yield to event loop
    }
  }

  processChunk();
}
```

**Banking Example:**
```javascript
// Process 1 million transactions without blocking
async function processBulkTransactions(transactions) {
  const BATCH_SIZE = 100;
  
  for (let i = 0; i < transactions.length; i += BATCH_SIZE) {
    const batch = transactions.slice(i, i + BATCH_SIZE);
    
    // Process batch
    await Promise.all(batch.map(t => processTransaction(t)));
    
    // Yield to event loop every batch
    if (i + BATCH_SIZE < transactions.length) {
      await new Promise(resolve => setTimeout(resolve, 0));
    }
  }
}
```

---

### Q5: How do you prevent memory leaks with timers?

**Answer:**

**Memory leaks occur when:**
1. Timers are not cleared
2. Timers hold references to large objects
3. Timers accumulate over time

**Common Leak Scenarios:**

**1. Not Clearing setInterval:**
```javascript
// ❌ MEMORY LEAK: Never cleared
function startPolling() {
  const data = new Array(1000000).fill('leak');
  
  setInterval(() => {
    console.log(data[0]); // Keeps 'data' in memory forever
  }, 1000);
}

startPolling(); // Memory leak!
```

**Solution:**
```javascript
// ✅ CORRECT: Store and clear interval
let intervalId = null;
let data = null;

function startPolling() {
  data = new Array(1000000).fill('safe');
  
  intervalId = setInterval(() => {
    console.log(data[0]);
  }, 1000);
}

function stopPolling() {
  if (intervalId) {
    clearInterval(intervalId);
    intervalId = null;
    data = null; // Release memory
  }
}

// Start polling
startPolling();

// Stop after 10 seconds
setTimeout(() => {
  stopPolling();
}, 10000);
```

**2. Creating Multiple Timers:**
```javascript
// ❌ LEAK: Creates new timer on every call
function startMonitoring() {
  setInterval(() => {
    checkAccountBalance();
  }, 5000);
}

// Called multiple times = multiple timers!
startMonitoring();
startMonitoring();
startMonitoring();
```

**Solution:**
```javascript
// ✅ CORRECT: Check if already running
let monitoringId = null;

function startMonitoring() {
  if (monitoringId) {
    console.log('Already monitoring');
    return;
  }
  
  monitoringId = setInterval(() => {
    checkAccountBalance();
  }, 5000);
}

function stopMonitoring() {
  if (monitoringId) {
    clearInterval(monitoringId);
    monitoringId = null;
  }
}
```

**3. Timers in Event Listeners:**
```javascript
// ❌ LEAK: New timer on every click
button.addEventListener('click', () => {
  setTimeout(() => {
    processTransaction();
  }, 1000);
});

// Clicking 100 times = 100 timers
```

**Solution:**
```javascript
// ✅ CORRECT: Debounce with single timer
let timerId = null;

button.addEventListener('click', () => {
  if (timerId) {
    clearTimeout(timerId);
  }
  
  timerId = setTimeout(() => {
    processTransaction();
    timerId = null;
  }, 1000);
});
```

**Banking Best Practices:**

```javascript
class TransactionMonitor {
  constructor() {
    this.intervalId = null;
    this.running = false;
  }

  start() {
    if (this.running) return;
    
    this.running = true;
    this.intervalId = setInterval(() => {
      this.checkPendingTransactions();
    }, 5000);
  }

  stop() {
    if (!this.running) return;
    
    if (this.intervalId) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
    
    this.running = false;
  }

  // Cleanup on process exit
  destroy() {
    this.stop();
  }
}

const monitor = new TransactionMonitor();
monitor.start();

// Cleanup on exit
process.on('SIGTERM', () => {
  monitor.destroy();
  process.exit(0);
});
```

---

### Q6: What is the difference between setTimeout 0ms and setImmediate?

**Answer:**

Both schedule callbacks to run **after current synchronous code**, but in **different event loop phases**.

**Key Differences:**

| Feature | setTimeout(fn, 0) | setImmediate(fn) |
|---------|------------------|------------------|
| **Event Loop Phase** | Timers Phase | Check Phase |
| **Execution** | Next timers phase | Next check phase |
| **Order** | May vary | After poll phase |
| **Use Case** | General deferral | I/O callbacks |

**Execution Order (Main Context):**

```javascript
setTimeout(() => {
  console.log('setTimeout');
}, 0);

setImmediate(() => {
  console.log('setImmediate');
});

// Order is non-deterministic in main context
// Can be: setTimeout, setImmediate
// Or: setImmediate, setTimeout
```

**Execution Order (I/O Context):**

```javascript
const fs = require('fs');

fs.readFile('file.txt', () => {
  // Inside I/O callback
  
  setTimeout(() => {
    console.log('setTimeout');
  }, 0);

  setImmediate(() => {
    console.log('setImmediate');
  });
});

// Order is ALWAYS: setImmediate, setTimeout
// Because we're in poll phase, check phase comes next
```

**When to Use Each:**

**Use `setImmediate` when:**
- Inside I/O callbacks
- Want callback to run after I/O events
- Building server/I/O-heavy applications

**Use `setTimeout(fn, 0)` when:**
- Breaking up CPU-intensive tasks
- General async deferral
- Browser compatibility needed

**Banking Example:**

```javascript
const fs = require('fs').promises;

async function processTransactionFile() {
  const data = await fs.readFile('transactions.txt');
  
  // Process immediately after I/O
  setImmediate(() => {
    console.log('Parse transactions');
    parseTransactions(data);
  });
  
  // Defer heavy computation
  setTimeout(() => {
    console.log('Calculate totals');
    calculateTotals(data);
  }, 0);
}

// Order: parseTransactions, calculateTotals
```

---

### Q7: How do you implement a retry mechanism with exponential backoff using timers?

**Answer:**

**Exponential backoff** increases delay between retries exponentially: 1s, 2s, 4s, 8s, etc.

**Implementation:**

```javascript
async function retryWithExponentialBackoff(fn, maxRetries = 5, baseDelay = 1000) {
  let attempt = 0;

  while (attempt < maxRetries) {
    try {
      const result = await fn();
      return result; // Success
    } catch (error) {
      attempt++;
      
      if (attempt >= maxRetries) {
        throw new Error(`Failed after ${maxRetries} attempts: ${error.message}`);
      }

      // Calculate delay: baseDelay * 2^attempt
      const delay = baseDelay * Math.pow(2, attempt);
      
      console.log(`Attempt ${attempt} failed. Retrying in ${delay}ms...`);
      
      // Wait before retry
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

**Usage:**

```javascript
// Banking: Retry failed transaction
async function processTransaction(transactionId) {
  // Simulate API call that may fail
  const response = await fetch(`/api/transactions/${transactionId}`, {
    method: 'POST'
  });

  if (!response.ok) {
    throw new Error(`Transaction failed: ${response.status}`);
  }

  return response.json();
}

// Use retry mechanism
retryWithExponentialBackoff(
  () => processTransaction('TXN001'),
  5,    // Max 5 retries
  1000  // Start with 1 second delay
)
  .then(result => {
    console.log('Transaction successful:', result);
  })
  .catch(error => {
    console.error('Transaction failed permanently:', error.message);
  });
```

**Retry Timeline:**
```
Attempt 1: Immediate
Attempt 2: After 2s   (1000 * 2^1)
Attempt 3: After 4s   (1000 * 2^2)
Attempt 4: After 8s   (1000 * 2^3)
Attempt 5: After 16s  (1000 * 2^4)
```

**With Jitter (Random Variation):**

```javascript
async function retryWithJitter(fn, maxRetries = 5, baseDelay = 1000) {
  let attempt = 0;

  while (attempt < maxRetries) {
    try {
      return await fn();
    } catch (error) {
      attempt++;
      
      if (attempt >= maxRetries) {
        throw new Error(`Failed after ${maxRetries} attempts`);
      }

      // Exponential backoff with jitter
      const exponentialDelay = baseDelay * Math.pow(2, attempt);
      const jitter = Math.random() * 1000; // 0-1000ms random
      const delay = exponentialDelay + jitter;
      
      console.log(`Retry ${attempt}/${maxRetries} in ${Math.round(delay)}ms`);
      
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

**Banking Use Cases:**
- Retrying failed payment gateway calls
- Reconnecting to database after connection loss
- Retrying API calls to external services (credit check, KYC)
- Handling transient network errors

---

### Q8: How do you schedule tasks at specific times (like midnight)?

**Answer:**

**Calculate milliseconds until target time**, then use `setTimeout`.

**Implementation:**

```javascript
function scheduleAtMidnight(callback) {
  const now = new Date();
  const midnight = new Date(now);
  midnight.setHours(24, 0, 0, 0); // Next midnight

  const msUntilMidnight = midnight - now;

  console.log(`Scheduling task for ${midnight.toISOString()}`);
  console.log(`Time until execution: ${msUntilMidnight}ms (${Math.round(msUntilMidnight / 1000 / 60)} minutes)`);

  setTimeout(() => {
    callback();
    
    // Reschedule for next day (recursive)
    scheduleAtMidnight(callback);
  }, msUntilMidnight);
}

// Usage
scheduleAtMidnight(() => {
  console.log('Midnight task executed at:', new Date().toISOString());
  applyDailyInterest();
});
```

**Schedule at Specific Time (e.g., 3:30 AM):**

```javascript
function scheduleAtTime(hours, minutes, callback) {
  const now = new Date();
  const scheduled = new Date(now);
  scheduled.setHours(hours, minutes, 0, 0);

  // If time has passed today, schedule for tomorrow
  if (scheduled <= now) {
    scheduled.setDate(scheduled.getDate() + 1);
  }

  const msUntilScheduled = scheduled - now;

  console.log(`Task scheduled for ${scheduled.toISOString()}`);

  setTimeout(() => {
    callback();
    
    // Reschedule for next day
    scheduleAtTime(hours, minutes, callback);
  }, msUntilScheduled);
}

// Run at 3:30 AM every day
scheduleAtTime(3, 30, () => {
  console.log('3:30 AM task running');
  generateDailyReports();
});
```

**Better Solution: Use Cron Package**

For production, use `node-cron` package:

```javascript
const cron = require('node-cron');

// Run at midnight every day
cron.schedule('0 0 * * *', () => {
  console.log('Running midnight task');
  applyDailyInterest();
});

// Run at 3:30 AM every day
cron.schedule('30 3 * * *', () => {
  console.log('Running 3:30 AM task');
  generateDailyReports();
});

// Run every Monday at 9:00 AM
cron.schedule('0 9 * * 1', () => {
  console.log('Weekly report');
  generateWeeklyReport();
});
```

**Banking Schedule Examples:**

```javascript
const cron = require('node-cron');

// Daily interest calculation at midnight
cron.schedule('0 0 * * *', () => {
  applyDailyInterest();
});

// Monthly statement generation on 1st at 2 AM
cron.schedule('0 2 1 * *', () => {
  generateMonthlyStatements();
});

// Check for due loan payments every hour
cron.schedule('0 * * * *', () => {
  checkDueLoanPayments();
});

// Send payment reminders at 9 AM on weekdays
cron.schedule('0 9 * * 1-5', () => {
  sendPaymentReminders();
});
```

---

### Q9: How do you cancel a timer and why is it important?

**Answer:**

**Canceling timers is critical** to prevent memory leaks and unwanted code execution.

**How to Cancel:**

**1. setTimeout:**
```javascript
const timerId = setTimeout(() => {
  console.log('This will not run');
}, 5000);

// Cancel before it executes
clearTimeout(timerId);
```

**2. setInterval:**
```javascript
const intervalId = setInterval(() => {
  console.log('Repeating...');
}, 1000);

// Cancel after 5 seconds
setTimeout(() => {
  clearInterval(intervalId);
  console.log('Interval stopped');
}, 5000);
```

**Why It's Important:**

**1. Prevent Memory Leaks:**
```javascript
class DataFetcher {
  constructor() {
    this.intervalId = null;
  }

  start() {
    this.intervalId = setInterval(() => {
      this.fetchData(); // Keeps 'this' in memory
    }, 1000);
  }

  stop() {
    clearInterval(this.intervalId); // Release memory
    this.intervalId = null;
  }
}

const fetcher = new DataFetcher();
fetcher.start();

// Without stop(), memory leak!
// With stop(), properly cleaned up
fetcher.stop();
```

**2. Prevent Duplicate Operations:**
```javascript
let refreshTimerId = null;

function scheduleRefresh() {
  // Cancel existing timer
  if (refreshTimerId) {
    clearTimeout(refreshTimerId);
  }

  // Schedule new one
  refreshTimerId = setTimeout(() => {
    refreshAccountBalance();
  }, 5000);
}

// Multiple calls won't create multiple timers
scheduleRefresh();
scheduleRefresh();
scheduleRefresh(); // Only this one runs
```

**3. Cleanup on Component Unmount (React Example):**
```javascript
function AccountBalance() {
  const [balance, setBalance] = React.useState(0);

  React.useEffect(() => {
    const intervalId = setInterval(() => {
      fetchBalance().then(setBalance);
    }, 5000);

    // Cleanup when component unmounts
    return () => {
      clearInterval(intervalId);
    };
  }, []);

  return <div>Balance: ${balance}</div>;
}
```

**4. Graceful Shutdown:**
```javascript
class BankingServer {
  constructor() {
    this.timers = new Set();
  }

  startInterestCalculator() {
    const timerId = setInterval(() => {
      this.calculateInterest();
    }, 24 * 60 * 60 * 1000);
    
    this.timers.add(timerId);
  }

  startTransactionMonitor() {
    const timerId = setInterval(() => {
      this.monitorTransactions();
    }, 5000);
    
    this.timers.add(timerId);
  }

  shutdown() {
    console.log('Shutting down gracefully...');
    
    // Clear all timers
    for (const timerId of this.timers) {
      clearInterval(timerId);
    }
    
    this.timers.clear();
    console.log('All timers cleared');
  }
}

const server = new BankingServer();
server.startInterestCalculator();
server.startTransactionMonitor();

// Graceful shutdown on SIGTERM
process.on('SIGTERM', () => {
  server.shutdown();
  process.exit(0);
});
```

**Banking Best Practice:**

```javascript
class TimerManager {
  constructor() {
    this.timers = new Map();
  }

  setTimeout(name, callback, delay) {
    // Clear existing timer with same name
    this.clearTimeout(name);
    
    const timerId = setTimeout(() => {
      callback();
      this.timers.delete(name);
    }, delay);
    
    this.timers.set(name, timerId);
    return timerId;
  }

  setInterval(name, callback, delay) {
    this.clearInterval(name);
    
    const timerId = setInterval(callback, delay);
    this.timers.set(name, timerId);
    return timerId;
  }

  clearTimeout(name) {
    const timerId = this.timers.get(name);
    if (timerId) {
      clearTimeout(timerId);
      this.timers.delete(name);
    }
  }

  clearInterval(name) {
    const timerId = this.timers.get(name);
    if (timerId) {
      clearInterval(timerId);
      this.timers.delete(name);
    }
  }

  clearAll() {
    for (const [name, timerId] of this.timers) {
      clearTimeout(timerId); // Works for both setTimeout and setInterval
    }
    this.timers.clear();
  }
}

// Usage
const timerManager = new TimerManager();

timerManager.setInterval('interestCalculator', () => {
  calculateInterest();
}, 24 * 60 * 60 * 1000);

timerManager.setTimeout('sessionTimeout', () => {
  expireSession();
}, 15 * 60 * 1000);

// Cleanup
process.on('SIGTERM', () => {
  timerManager.clearAll();
  process.exit(0);
});
```

---

### Q10: What are the performance implications of using many timers?

**Answer:**

**Every timer consumes resources**. Too many timers can impact performance.

**Performance Concerns:**

**1. Memory Usage:**
- Each timer allocates memory
- Stores callback closure and context
- References prevent garbage collection

```javascript
// ❌ BAD: Creating 10,000 timers
for (let i = 0; i < 10000; i++) {
  setTimeout(() => {
    console.log(i);
  }, i * 1000);
}
// Memory: ~10,000 timer objects + closures
```

**2. CPU Overhead:**
- Event loop must check timers every iteration
- More timers = more checking

**3. Timer Resolution:**
- Node.js uses a binary heap for timers
- Insertion: O(log n)
- Finding expired: O(1)
- But with 10,000+ timers, overhead increases

**Best Practices:**

**1. Batch Operations:**

```javascript
// ❌ BAD: Timer per account (1,000,000 timers)
for (const account of accounts) {
  setTimeout(() => {
    calculateInterest(account);
  }, 1000);
}

// ✅ GOOD: Single timer, batch processing
setTimeout(() => {
  for (const account of accounts) {
    calculateInterest(account);
  }
}, 1000);

// ✅ BETTER: Process in chunks to avoid blocking
async function processInChunks(accounts, chunkSize = 1000) {
  for (let i = 0; i < accounts.length; i += chunkSize) {
    const chunk = accounts.slice(i, i + chunkSize);
    
    await Promise.all(chunk.map(acc => calculateInterest(acc)));
    
    // Yield to event loop
    await new Promise(resolve => setImmediate(resolve));
  }
}

setTimeout(() => {
  processInChunks(accounts);
}, 1000);
```

**2. Use Recurring Intervals Wisely:**

```javascript
// ❌ BAD: Many intervals
setInterval(() => checkAccount1(), 1000);
setInterval(() => checkAccount2(), 1000);
setInterval(() => checkAccount3(), 1000);
// ... 1000 more

// ✅ GOOD: Single interval, check all
setInterval(() => {
  checkAccount1();
  checkAccount2();
  checkAccount3();
  // ... all accounts
}, 1000);
```

**3. Clear Unused Timers:**

```javascript
class UserSession {
  constructor(userId) {
    this.userId = userId;
    this.timeoutId = null;
  }

  startTimeout() {
    this.timeoutId = setTimeout(() => {
      this.expire();
    }, 15 * 60 * 1000);
  }

  destroy() {
    // Always clear timer when session ends
    if (this.timeoutId) {
      clearTimeout(this.timeoutId);
      this.timeoutId = null;
    }
  }
}

// Create 100,000 sessions
const sessions = [];
for (let i = 0; i < 100000; i++) {
  const session = new UserSession(i);
  session.startTimeout();
  sessions.push(session);
}

// Destroy sessions when done (clears timers)
sessions.forEach(s => s.destroy());
```

**4. Use Queue Systems for High Volume:**

```javascript
// For processing millions of transactions, use a queue
const Bull = require('bull');

const transactionQueue = new Bull('transactions');

// Producer: Add transactions to queue
for (const transaction of transactions) {
  transactionQueue.add(transaction, {
    delay: transaction.scheduledTime - Date.now()
  });
}

// Consumer: Process transactions
transactionQueue.process(async (job) => {
  await processTransaction(job.data);
});

// Better than creating millions of setTimeout calls
```

**Performance Benchmarks:**

```javascript
const { performance } = require('perf_hooks');

// Test: Creating N timers
function benchmarkTimers(count) {
  const start = performance.now();
  const memStart = process.memoryUsage().heapUsed;

  const timers = [];
  for (let i = 0; i < count; i++) {
    const id = setTimeout(() => {}, 60000);
    timers.push(id);
  }

  const end = performance.now();
  const memEnd = process.memoryUsage().heapUsed;

  console.log(`Created ${count} timers:`);
  console.log(`  Time: ${(end - start).toFixed(2)}ms`);
  console.log(`  Memory: ${((memEnd - memStart) / 1024 / 1024).toFixed(2)}MB`);

  // Cleanup
  timers.forEach(id => clearTimeout(id));
}

benchmarkTimers(1000);
benchmarkTimers(10000);
benchmarkTimers(100000);

// Results (approximate):
// 1,000 timers: ~5ms, ~1MB
// 10,000 timers: ~50ms, ~10MB
// 100,000 timers: ~500ms, ~100MB
```

**Banking Recommendation:**
- **< 100 timers**: No concern
- **100-1,000 timers**: Monitor performance
- **> 1,000 timers**: Consider batching or queue system
- **> 10,000 timers**: Use dedicated scheduler service (Bull, Agenda, node-cron)

---

## ✅ DO's - Best Practices

### 1. ✅ **DO: Use Recursive setTimeout Instead of setInterval**

**Why:** Recursive `setTimeout` ensures a fixed delay **between** executions, preventing callback overlap.

```javascript
// ❌ BAD: setInterval can overlap
setInterval(async () => {
  await longRunningTask(); // Takes 5 seconds
}, 1000); // Runs every 1 second (callbacks queue up!)

// ✅ GOOD: Recursive setTimeout
async function scheduleNext() {
  await longRunningTask(); // Takes 5 seconds
  
  setTimeout(scheduleNext, 1000); // 1 second AFTER completion
}

setTimeout(scheduleNext, 1000);
```

**Banking Example:**
```javascript
async function processTransactionQueue() {
  try {
    await processPendingTransactions(); // May take variable time
  } catch (error) {
    console.error('Error processing transactions:', error);
  }
  
  // Schedule next run after current one completes
  setTimeout(processTransactionQueue, 5000);
}

processTransactionQueue();
```

---

### 2. ✅ **DO: Always Store Timer IDs for Cleanup**

**Why:** Prevents memory leaks and allows graceful shutdown.

```javascript
class AccountMonitor {
  constructor() {
    this.intervalId = null;
  }

  start() {
    // ✅ Store timer ID
    this.intervalId = setInterval(() => {
      this.checkOverdrafts();
    }, 30000);
  }

  stop() {
    // ✅ Clear timer
    if (this.intervalId) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
  }
}

const monitor = new AccountMonitor();
monitor.start();

// Cleanup on shutdown
process.on('SIGTERM', () => {
  monitor.stop();
  process.exit(0);
});
```

---

### 3. ✅ **DO: Handle Timer Errors with Try-Catch**

**Why:** Errors in timer callbacks can crash your application.

```javascript
// ✅ GOOD: Error handling
setInterval(() => {
  try {
    calculateDailyInterest();
  } catch (error) {
    console.error('Error calculating interest:', error);
    // Log to monitoring service
    logError(error);
  }
}, 24 * 60 * 60 * 1000);
```

**For Async Callbacks:**
```javascript
setInterval(async () => {
  try {
    await processRecurringPayments();
  } catch (error) {
    console.error('Payment processing failed:', error);
    await notifyAdmins(error);
  }
}, 60000);
```

---

### 4. ✅ **DO: Use Node-Cron for Production Scheduling**

**Why:** More reliable, feature-rich, and handles edge cases.

```javascript
const cron = require('node-cron');

// ✅ GOOD: Use cron for production
cron.schedule('0 0 * * *', () => {
  calculateDailyInterest();
}, {
  timezone: "America/New_York"
});

// Benefits:
// - Handles DST transitions
// - Timezone support
// - Cron expression syntax
// - Automatic rescheduling
// - More predictable than setTimeout
```

---

### 5. ✅ **DO: Implement Exponential Backoff for Retries**

**Why:** Prevents overwhelming services during outages.

```javascript
async function retryTransaction(fn, maxRetries = 5) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxRetries - 1) throw error;
      
      // ✅ Exponential backoff: 1s, 2s, 4s, 8s, 16s
      const delay = Math.pow(2, attempt) * 1000;
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}

// Usage
await retryTransaction(() => processPayment(transaction));
```

---

### 6. ✅ **DO: Clear Timers in Cleanup Functions**

**Why:** Ensures timers don't keep running after they're no longer needed.

```javascript
class SessionManager {
  constructor() {
    this.sessions = new Map();
  }

  createSession(userId) {
    const sessionId = generateId();
    
    // ✅ Store timeout ID
    const timeoutId = setTimeout(() => {
      this.expireSession(sessionId);
    }, 15 * 60 * 1000);

    this.sessions.set(sessionId, { userId, timeoutId });
    return sessionId;
  }

  destroySession(sessionId) {
    const session = this.sessions.get(sessionId);
    if (!session) return;

    // ✅ Clear timeout before deleting
    clearTimeout(session.timeoutId);
    this.sessions.delete(sessionId);
  }

  // ✅ Cleanup all on shutdown
  shutdown() {
    for (const session of this.sessions.values()) {
      clearTimeout(session.timeoutId);
    }
    this.sessions.clear();
  }
}
```

---

### 7. ✅ **DO: Add Jitter to Distributed System Timers**

**Why:** Prevents thundering herd problem when many instances restart.

```javascript
function scheduleWithJitter(callback, baseDelay) {
  // ✅ Add random jitter (0-20% of baseDelay)
  const jitter = Math.random() * baseDelay * 0.2;
  const delay = baseDelay + jitter;

  setTimeout(callback, delay);
}

// Multiple servers won't all hit database at same time
scheduleWithJitter(() => {
  refreshCache();
}, 60000);
```

---

### 8. ✅ **DO: Use setImmediate for I/O-Heavy Operations**

**Why:** Better suited for deferring after I/O operations complete.

```javascript
const fs = require('fs').promises;

async function processFile() {
  const data = await fs.readFile('transactions.csv');
  
  // ✅ Use setImmediate after I/O
  setImmediate(() => {
    parseAndProcessTransactions(data);
  });
}
```

---

### 9. ✅ **DO: Document Timer Behavior**

**Why:** Makes code maintainable and prevents confusion.

```javascript
/**
 * Calculates and applies daily interest to all savings accounts.
 * Runs daily at midnight UTC.
 * 
 * @scheduled Daily at 00:00 UTC
 * @duration ~30 seconds for 100,000 accounts
 * @dependencies Database connection required
 */
function scheduleDailyInterest() {
  const msUntilMidnight = calculateMsUntilMidnight();
  
  setTimeout(() => {
    applyDailyInterest();
    scheduleDailyInterest(); // Reschedule for next day
  }, msUntilMidnight);
}
```

---

### 10. ✅ **DO: Monitor Timer Health**

**Why:** Detect when scheduled tasks aren't running.

```javascript
class MonitoredTimer {
  constructor(name, callback, interval) {
    this.name = name;
    this.callback = callback;
    this.interval = interval;
    this.lastRun = null;
    this.runCount = 0;
    this.intervalId = null;
  }

  start() {
    this.intervalId = setInterval(async () => {
      const start = Date.now();
      
      try {
        await this.callback();
        
        this.lastRun = new Date();
        this.runCount++;
        
        const duration = Date.now() - start;
        
        // ✅ Log metrics
        console.log(`Timer "${this.name}" executed in ${duration}ms`);
        
        // Alert if taking too long
        if (duration > this.interval * 0.8) {
          console.warn(`Timer "${this.name}" is slow (${duration}ms)`);
        }
      } catch (error) {
        console.error(`Timer "${this.name}" failed:`, error);
      }
    }, this.interval);
  }

  getHealth() {
    const now = Date.now();
    const timeSinceLastRun = this.lastRun ? now - this.lastRun : null;
    
    return {
      name: this.name,
      running: !!this.intervalId,
      runCount: this.runCount,
      lastRun: this.lastRun,
      timeSinceLastRun,
      healthy: timeSinceLastRun < this.interval * 2
    };
  }
}

const timer = new MonitoredTimer('interest-calculator', 
  calculateInterest, 
  24 * 60 * 60 * 1000
);

timer.start();

// Health check endpoint
app.get('/health/timers', (req, res) => {
  res.json({
    timers: [timer.getHealth()]
  });
});
```

---

## ❌ DON'Ts - Common Mistakes

### 1. ❌ **DON'T: Use setInterval Without Clearing It**

**Why:** Causes memory leaks and keeps process alive.

```javascript
// ❌ BAD: Never cleared
function startPolling() {
  setInterval(() => {
    checkAccountStatus();
  }, 5000);
}

// ✅ GOOD: Store and clear
let pollingId = null;

function startPolling() {
  pollingId = setInterval(() => {
    checkAccountStatus();
  }, 5000);
}

function stopPolling() {
  if (pollingId) {
    clearInterval(pollingId);
    pollingId = null;
  }
}
```

---

### 2. ❌ **DON'T: Assume Timers are Precise**

**Why:** Timers can drift or execute late if event loop is blocked.

```javascript
// ❌ BAD: Assuming exactly 1000ms between logs
let counter = 0;
setInterval(() => {
  counter++;
  console.log(`Counter: ${counter}`); // May not be exactly every second
}, 1000);

// ✅ GOOD: Use actual timestamps
let counter = 0;
let lastTime = Date.now();

setInterval(() => {
  const now = Date.now();
  const elapsed = now - lastTime;
  lastTime = now;
  
  counter++;
  console.log(`Counter: ${counter}, Actual elapsed: ${elapsed}ms`);
}, 1000);
```

---

### 3. ❌ **DON'T: Create Timers in Loops Without Cleanup**

**Why:** Creates too many timers, wastes memory.

```javascript
// ❌ BAD: Creating 1,000,000 timers
for (let i = 0; i < 1000000; i++) {
  setTimeout(() => {
    processAccount(i);
  }, 1000);
}

// ✅ GOOD: Batch processing
setTimeout(() => {
  for (let i = 0; i < 1000000; i++) {
    processAccount(i);
  }
}, 1000);

// ✅ BETTER: Process in chunks
async function processInChunks() {
  const CHUNK_SIZE = 1000;
  
  for (let i = 0; i < 1000000; i += CHUNK_SIZE) {
    await Promise.all(
      Array.from({ length: CHUNK_SIZE }, (_, j) => 
        processAccount(i + j)
      )
    );
    
    await new Promise(resolve => setImmediate(resolve));
  }
}

setTimeout(processInChunks, 1000);
```

---

### 4. ❌ **DON'T: Ignore Errors in Timer Callbacks**

**Why:** Unhandled errors can crash your application.

```javascript
// ❌ BAD: No error handling
setInterval(() => {
  const result = riskyOperation(); // May throw
  processResult(result);
}, 5000);

// ✅ GOOD: Proper error handling
setInterval(() => {
  try {
    const result = riskyOperation();
    processResult(result);
  } catch (error) {
    console.error('Timer callback error:', error);
    logToMonitoring(error);
  }
}, 5000);

// ✅ BETTER: For async operations
setInterval(async () => {
  try {
    const result = await asyncRiskyOperation();
    await processResult(result);
  } catch (error) {
    console.error('Async timer error:', error);
    await logToMonitoring(error);
  }
}, 5000);
```

---

### 5. ❌ **DON'T: Use setTimeout for Long-Running Scheduled Tasks**

**Why:** Process restarts lose timer state. Use persistent schedulers.

```javascript
// ❌ BAD: Lost on restart
setTimeout(() => {
  generateMonthlyReport();
}, 30 * 24 * 60 * 60 * 1000); // 30 days

// If server restarts, timer is lost!

// ✅ GOOD: Use cron or persistent queue
const cron = require('node-cron');

cron.schedule('0 0 1 * *', () => {
  generateMonthlyReport();
});

// Or use a job queue (Bull, Agenda)
const Bull = require('bull');
const queue = new Bull('reports');

queue.add('monthly-report', {}, {
  repeat: {
    cron: '0 0 1 * *' // 1st of every month
  }
});
```

---

### 6. ❌ **DON'T: Block the Event Loop in Timer Callbacks**

**Why:** Delays ALL timers and I/O operations.

```javascript
// ❌ BAD: Blocking operation
setInterval(() => {
  // Blocks event loop for 5 seconds
  const sum = heavySyncComputation(); // 5 seconds
  
  console.log('Result:', sum);
}, 1000);

// ✅ GOOD: Break into chunks
setInterval(async () => {
  // Process in non-blocking chunks
  const sum = await heavyComputationInChunks();
  
  console.log('Result:', sum);
}, 1000);

async function heavyComputationInChunks() {
  let result = 0;
  
  for (let i = 0; i < 1000000; i += 10000) {
    // Process chunk
    for (let j = 0; j < 10000; j++) {
      result += i + j;
    }
    
    // Yield to event loop
    await new Promise(resolve => setImmediate(resolve));
  }
  
  return result;
}
```

---

### 7. ❌ **DON'T: Forget Timezones for Banking Operations**

**Why:** Interest calculations, statement generation must respect timezones.

```javascript
// ❌ BAD: Uses server timezone
function scheduleAtMidnight(callback) {
  const now = new Date();
  const midnight = new Date(now);
  midnight.setHours(24, 0, 0, 0); // Server timezone
  
  setTimeout(callback, midnight - now);
}

// ✅ GOOD: Use specific timezone
const cron = require('node-cron');

cron.schedule('0 0 * * *', () => {
  applyDailyInterest();
}, {
  timezone: 'America/New_York' // EST/EDT
});

// For multiple regions
const timezones = ['America/New_York', 'America/Chicago', 'America/Los_Angeles'];

timezones.forEach(tz => {
  cron.schedule('0 0 * * *', () => {
    applyDailyInterest(tz);
  }, {
    timezone: tz
  });
});
```

---

### 8. ❌ **DON'T: Use Timers for Critical Business Logic**

**Why:** Timers can be delayed or skipped. Use idempotent operations.

```javascript
// ❌ BAD: Critical payment depends on timer
setTimeout(() => {
  processCriticalPayment(); // What if timer is delayed?
}, 60000);

// ✅ GOOD: Idempotent check-and-process
async function processPayments() {
  const pendingPayments = await db.query(
    'SELECT * FROM payments WHERE status = ? AND due_date <= ?',
    ['pending', new Date()]
  );

  for (const payment of pendingPayments) {
    await processPayment(payment);
  }
}

// Run check periodically
setInterval(processPayments, 60000);

// Even if delayed, will process all due payments
```

---

### 9. ❌ **DON'T: Rely on setTimeout(fn, 0) for Ordering**

**Why:** Order vs setImmediate is non-deterministic in main context.

```javascript
// ❌ BAD: Assuming order
setTimeout(() => {
  console.log('1');
}, 0);

setImmediate(() => {
  console.log('2');
});

// Output is non-deterministic: could be 1,2 or 2,1

// ✅ GOOD: Use promises for guaranteed order
async function orderedExecution() {
  await step1();
  await step2();
}
```

---

### 10. ❌ **DON'T: Create Multiple Timers for Same Task**

**Why:** Wastes resources and causes duplicate operations.

```javascript
// ❌ BAD: Multiple calls create multiple timers
function startMonitoring() {
  setInterval(() => {
    checkSystem();
  }, 5000);
}

startMonitoring();
startMonitoring(); // Oops! Two timers now
startMonitoring(); // Three timers!

// ✅ GOOD: Singleton pattern
let monitoringTimer = null;

function startMonitoring() {
  if (monitoringTimer) {
    console.log('Already monitoring');
    return;
  }
  
  monitoringTimer = setInterval(() => {
    checkSystem();
  }, 5000);
}

function stopMonitoring() {
  if (monitoringTimer) {
    clearInterval(monitoringTimer);
    monitoringTimer = null;
  }
}
```

---

## 🎯 Key Takeaways

1. **setTimeout vs setInterval**: Use recursive `setTimeout` for most cases to avoid callback overlap

2. **Timers are NOT Precise**: They specify minimum delay, not exact delay. Event loop blocking delays execution.

3. **Always Clear Timers**: Use `clearTimeout`/`clearInterval` to prevent memory leaks and enable graceful shutdown

4. **Event Loop Integration**: Timers execute in the Timers Phase, after poll phase checks I/O

5. **Exponential Backoff**: Implement retry logic with exponentially increasing delays for resilience

6. **Use Cron for Production**: For scheduled tasks, use `node-cron` or job queues (Bull, Agenda) instead of raw timers

7. **Handle Errors**: Always wrap timer callbacks in try-catch to prevent crashes

8. **Batch Operations**: Don't create thousands of timers. Batch operations in a single timer.

9. **Cleanup on Shutdown**: Clear all timers in SIGTERM/SIGINT handlers for graceful shutdown

10. **Monitor Timer Health**: Track last execution time and duration to detect failures

---

## 📚 Summary

**Node.js timers** (`setTimeout`, `setInterval`) are essential for scheduling operations but come with important considerations:

### Core Concepts:
- **setTimeout**: Execute once after delay
- **setInterval**: Execute repeatedly at intervals
- **clearTimeout/clearInterval**: Cancel timers
- **Minimum Delay**: Timers specify minimum time, not exact time

### Event Loop Integration:
1. Timers execute in **Timers Phase**
2. Checked after poll phase completes
3. Can be delayed by blocking operations
4. Not guaranteed to be precise

### Banking Applications:
- **Daily Interest Calculation**: Schedule at midnight with recursive `setTimeout`
- **Recurring Payments**: Process with `setInterval` or cron
- **Session Timeouts**: Automatic expiry after inactivity
- **Payment Reminders**: Scheduled notifications

### Production Best Practices:
```javascript
// Use node-cron for production scheduling
const cron = require('node-cron');

// Daily at midnight
cron.schedule('0 0 * * *', calculateDailyInterest);

// Monthly on 1st
cron.schedule('0 0 1 * *', generateStatements);

// Every 5 minutes
cron.schedule('*/5 * * * *', checkOverdrafts);
```

### Critical Rules:
✅ **DO**: Store timer IDs and clear them
✅ **DO**: Use recursive `setTimeout` over `setInterval`
✅ **DO**: Handle errors in timer callbacks
✅ **DO**: Use cron for production scheduling
❌ **DON'T**: Create thousands of timers
❌ **DON'T**: Assume precise timing
❌ **DON'T**: Block event loop in callbacks
❌ **DON'T**: Forget cleanup on shutdown

**Master timers to build reliable banking schedulers!** ⏰🏦

---

## 📖 Additional Resources

- [Node.js Timers Documentation](https://nodejs.org/api/timers.html)
- [Event Loop Explained](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
- [node-cron Package](https://www.npmjs.com/package/node-cron)
- [Bull Queue (Job Scheduling)](https://github.com/OptimalBits/bull)
- [Understanding setImmediate](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#setimmediate-vs-settimeout)

