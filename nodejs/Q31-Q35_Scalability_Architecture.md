# Node.js Questions 31-35: Scalability & Architecture Patterns

---

### Q31. How do you design a microservices architecture with Node.js?

**Answer:**

**Microservices Pattern for Banking:**

```javascript
// Service 1: Account Service
class AccountService {
  constructor(port = 3001) {
    this.port = port;
    this.setupRoutes();
  }
  
  setupRoutes() {
    const express = require('express');
    const app = express();
    
    app.use(express.json());
    
    app.get('/accounts/:id', async (req, res) => {
      const account = await this.getAccount(req.params.id);
      res.json(account);
    });
    
    app.post('/accounts', async (req, res) => {
      const account = await this.createAccount(req.body);
      res.json(account);
    });
    
    app.listen(this.port, () => {
      console.log(`Account Service running on ${this.port}`);
    });
  }
  
  async getAccount(id) {
    return { id, balance: 10000, type: 'savings' };
  }
  
  async createAccount(data) {
    return { id: 'ACC-' + Date.now(), ...data };
  }
}

// Service 2: Transaction Service
class TransactionService {
  constructor(port = 3002) {
    this.port = port;
    this.setupRoutes();
  }
  
  setupRoutes() {
    const express = require('express');
    const app = express();
    
    app.use(express.json());
    
    app.post('/transactions', async (req, res) => {
      const result = await this.processTransaction(req.body);
      res.json(result);
    });
    
    app.get('/transactions/:accountId', async (req, res) => {
      const transactions = await this.getTransactions(req.params.accountId);
      res.json(transactions);
    });
    
    app.listen(this.port, () => {
      console.log(`Transaction Service running on ${this.port}`);
    });
  }
  
  async processTransaction(data) {
    // Call Account Service to verify balance
    const axios = require('axios');
    const account = await axios.get(`http://localhost:3001/accounts/${data.fromAccount}`);
    
    if (account.data.balance >= data.amount) {
      return { success: true, transactionId: 'TXN-' + Date.now() };
    } else {
      return { success: false, error: 'Insufficient funds' };
    }
  }
  
  async getTransactions(accountId) {
    return [
      { id: 'TXN-1', amount: 100, date: new Date() }
    ];
  }
}

// API Gateway
class APIGateway {
  constructor(port = 3000) {
    this.port = port;
    this.services = {
      accounts: 'http://localhost:3001',
      transactions: 'http://localhost:3002'
    };
    this.setupRoutes();
  }
  
  setupRoutes() {
    const express = require('express');
    const { createProxyMiddleware } = require('http-proxy-middleware');
    const app = express();
    
    // Rate limiting
    const rateLimit = require('express-rate-limit');
    const limiter = rateLimit({
      windowMs: 15 * 60 * 1000,
      max: 100
    });
    
    app.use(limiter);
    
    // Service discovery and load balancing
    app.use('/api/accounts', createProxyMiddleware({
      target: this.services.accounts,
      pathRewrite: { '^/api/accounts': '/accounts' }
    }));
    
    app.use('/api/transactions', createProxyMiddleware({
      target: this.services.transactions,
      pathRewrite: { '^/api/transactions': '/transactions' }
    }));
    
    app.listen(this.port, () => {
      console.log(`API Gateway running on ${this.port}`);
    });
  }
}
```

### Q32. How do you implement event-driven architecture in Node.js?

**Answer:**

```javascript
const EventEmitter = require('events');

// Event Bus for Banking
class BankingEventBus extends EventEmitter {
  constructor() {
    super();
    this.setMaxListeners(50);
  }
  
  // Published events
  publishTransactionCreated(transaction) {
    this.emit('transaction.created', transaction);
  }
  
  publishTransactionCompleted(transaction) {
    this.emit('transaction.completed', transaction);
  }
  
  publishAccountUpdated(account) {
    this.emit('account.updated', account);
  }
  
  // Subscriber helpers
  onTransactionCreated(handler) {
    this.on('transaction.created', handler);
  }
  
  onTransactionCompleted(handler) {
    this.on('transaction.completed', handler);
  }
  
  onAccountUpdated(handler) {
    this.on('account.updated', handler);
  }
}

// Services subscribe to events
class NotificationService {
  constructor(eventBus) {
    this.eventBus = eventBus;
    this.subscribe();
  }
  
  subscribe() {
    this.eventBus.onTransactionCompleted(async (txn) => {
      await this.sendNotification(txn);
    });
  }
  
  async sendNotification(txn) {
    console.log(`Sending notification for transaction: ${txn.id}`);
    // Send email, SMS, push notification
  }
}

class AuditService {
  constructor(eventBus) {
    this.eventBus = eventBus;
    this.subscribe();
  }
  
  subscribe() {
    this.eventBus.onTransactionCreated(async (txn) => {
      await this.logTransaction(txn);
    });
    
    this.eventBus.onAccountUpdated(async (account) => {
      await this.logAccountChange(account);
    });
  }
  
  async logTransaction(txn) {
    console.log(`Audit: Transaction ${txn.id} created`);
  }
  
  async logAccountChange(account) {
    console.log(`Audit: Account ${account.id} updated`);
  }
}

// Usage
const eventBus = new BankingEventBus();
const notificationService = new NotificationService(eventBus);
const auditService = new AuditService(eventBus);

// Publish events
eventBus.publishTransactionCreated({
  id: 'TXN-001',
  amount: 5000
});

eventBus.publishTransactionCompleted({
  id: 'TXN-001',
  amount: 5000,
  status: 'completed'
});
```

### Q33. How do you implement the Circuit Breaker pattern?

**Answer:**

```javascript
class CircuitBreaker {
  constructor(options = {}) {
    this.failureThreshold = options.failureThreshold || 5;
    this.successThreshold = options.successThreshold || 2;
    this.timeout = options.timeout || 60000;
    this.monitorInterval = options.monitorInterval || 10000;
    
    this.state = 'CLOSED';
    this.failureCount = 0;
    this.successCount = 0;
    this.nextAttempt = Date.now();
    this.metrics = {
      total: 0,
      successful: 0,
      failed: 0,
      rejected: 0
    };
    
    this.startMonitoring();
  }
  
  async execute(operation) {
    this.metrics.total++;
    
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        this.metrics.rejected++;
        throw new Error('Circuit breaker is OPEN');
      }
      this.state = 'HALF_OPEN';
      console.log('Circuit breaker: HALF_OPEN');
    }
    
    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  onSuccess() {
    this.metrics.successful++;
    this.failureCount = 0;
    
    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      if (this.successCount >= this.successThreshold) {
        this.state = 'CLOSED';
        this.successCount = 0;
        console.log('Circuit breaker: CLOSED - Service restored');
      }
    }
  }
  
  onFailure() {
    this.metrics.failed++;
    this.failureCount++;
    this.successCount = 0;
    
    if (this.failureCount >= this.failureThreshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.timeout;
      console.error(`Circuit breaker: OPEN - ${this.failureCount} failures`);
    }
  }
  
  startMonitoring() {
    setInterval(() => {
      console.log('Circuit Breaker Status:', {
        state: this.state,
        metrics: this.metrics,
        failureCount: this.failureCount,
        successRate: ((this.metrics.successful / this.metrics.total) * 100).toFixed(2) + '%'
      });
    }, this.monitorInterval);
  }
  
  getStatus() {
    return {
      state: this.state,
      failureCount: this.failureCount,
      successCount: this.successCount,
      metrics: this.metrics
    };
  }
}

// Usage with banking service
class ExternalBankingAPI {
  constructor() {
    this.circuitBreaker = new CircuitBreaker({
      failureThreshold: 3,
      timeout: 30000
    });
  }
  
  async callExternalAPI(endpoint, data) {
    return this.circuitBreaker.execute(async () => {
      // Actual API call
      return await this.makeRequest(endpoint, data);
    });
  }
  
  async makeRequest(endpoint, data) {
    // Simulate API call with random failures
    if (Math.random() < 0.3) {
      throw new Error('API call failed');
    }
    return { success: true, data };
  }
}
```

### Q34. How do you implement CQRS (Command Query Responsibility Segregation)?

**Answer:**

```javascript
// Command side - Write operations
class BankingCommandService {
  constructor(eventStore) {
    this.eventStore = eventStore;
  }
  
  async createAccount(command) {
    const event = {
      type: 'AccountCreated',
      aggregateId: command.accountId,
      data: {
        accountId: command.accountId,
        customerId: command.customerId,
        initialBalance: command.initialBalance
      },
      timestamp: new Date()
    };
    
    await this.eventStore.append(event);
    return event;
  }
  
  async depositMoney(command) {
    const event = {
      type: 'MoneyDeposited',
      aggregateId: command.accountId,
      data: {
        accountId: command.accountId,
        amount: command.amount
      },
      timestamp: new Date()
    };
    
    await this.eventStore.append(event);
    return event;
  }
  
  async withdrawMoney(command) {
    // Check balance from query side
    const account = await this.queryService.getAccount(command.accountId);
    
    if (account.balance < command.amount) {
      throw new Error('Insufficient funds');
    }
    
    const event = {
      type: 'MoneyWithdrawn',
      aggregateId: command.accountId,
      data: {
        accountId: command.accountId,
        amount: command.amount
      },
      timestamp: new Date()
    };
    
    await this.eventStore.append(event);
    return event;
  }
}

// Query side - Read operations
class BankingQueryService {
  constructor(readModel) {
    this.readModel = readModel;
  }
  
  async getAccount(accountId) {
    return this.readModel.accounts.get(accountId);
  }
  
  async getAccountBalance(accountId) {
    const account = await this.getAccount(accountId);
    return account ? account.balance : 0;
  }
  
  async getTransactionHistory(accountId, limit = 50) {
    return this.readModel.transactions
      .filter(t => t.accountId === accountId)
      .slice(0, limit);
  }
}

// Event Store
class EventStore {
  constructor() {
    this.events = [];
  }
  
  async append(event) {
    event.id = this.events.length + 1;
    this.events.push(event);
    
    // Publish to event bus for projections
    this.publish(event);
  }
  
  async getEvents(aggregateId) {
    return this.events.filter(e => e.aggregateId === aggregateId);
  }
  
  publish(event) {
    // Notify projections
    console.log('Event published:', event.type);
  }
}

// Read Model Projection
class AccountProjection {
  constructor(eventStore) {
    this.accounts = new Map();
    this.eventStore = eventStore;
    this.rebuild();
  }
  
  async rebuild() {
    const events = await this.eventStore.getEvents();
    
    for (const event of events) {
      this.apply(event);
    }
  }
  
  apply(event) {
    switch (event.type) {
      case 'AccountCreated':
        this.accounts.set(event.data.accountId, {
          id: event.data.accountId,
          customerId: event.data.customerId,
          balance: event.data.initialBalance,
          createdAt: event.timestamp
        });
        break;
        
      case 'MoneyDeposited':
        const depositAccount = this.accounts.get(event.data.accountId);
        if (depositAccount) {
          depositAccount.balance += event.data.amount;
        }
        break;
        
      case 'MoneyWithdrawn':
        const withdrawAccount = this.accounts.get(event.data.accountId);
        if (withdrawAccount) {
          withdrawAccount.balance -= event.data.amount;
        }
        break;
    }
  }
}
```

### Q35. How do you handle distributed transactions across microservices?

**Answer:**

**Saga Pattern Implementation:**

```javascript
// Saga Orchestrator
class TransferSaga {
  constructor() {
    this.steps = [];
    this.compensations = [];
  }
  
  async execute(transfer) {
    const context = {
      transfer,
      completed: [],
      failed: false
    };
    
    try {
      // Step 1: Debit source account
      await this.debitAccount(context);
      context.completed.push('debit');
      
      // Step 2: Credit destination account
      await this.creditAccount(context);
      context.completed.push('credit');
      
      // Step 3: Record transaction
      await this.recordTransaction(context);
      context.completed.push('record');
      
      // Step 4: Send notification
      await this.sendNotification(context);
      
      return { success: true, transactionId: context.transactionId };
      
    } catch (error) {
      console.error('Saga failed, running compensations:', error);
      context.failed = true;
      
      await this.compensate(context);
      
      throw new Error(`Transfer failed: ${error.message}`);
    }
  }
  
  async debitAccount(context) {
    console.log('Debit account:', context.transfer.from);
    // Call Account Service
    if (Math.random() < 0.1) {
      throw new Error('Debit failed');
    }
  }
  
  async creditAccount(context) {
    console.log('Credit account:', context.transfer.to);
    // Call Account Service
    if (Math.random() < 0.1) {
      throw new Error('Credit failed');
    }
  }
  
  async recordTransaction(context) {
    console.log('Record transaction');
    context.transactionId = 'TXN-' + Date.now();
    // Call Transaction Service
  }
  
  async sendNotification(context) {
    console.log('Send notification');
    // Call Notification Service (best effort, no compensation)
  }
  
  async compensate(context) {
    console.log('Running compensations for:', context.completed);
    
    // Reverse in opposite order
    for (const step of context.completed.reverse()) {
      try {
        await this[`compensate_${step}`](context);
      } catch (error) {
        console.error(`Compensation failed for ${step}:`, error);
        // Alert operations team
      }
    }
  }
  
  async compensate_debit(context) {
    console.log('Compensating debit - refunding account');
    // Credit back the debited amount
  }
  
  async compensate_credit(context) {
    console.log('Compensating credit - reversing credit');
    // Debit the credited amount
  }
  
  async compensate_record(context) {
    console.log('Compensating record - marking as failed');
    // Update transaction status to FAILED
  }
}

// Usage
async function transferMoney() {
  const saga = new TransferSaga();
  
  try {
    const result = await saga.execute({
      from: 'ACC-001',
      to: 'ACC-002',
      amount: 5000
    });
    
    console.log('Transfer successful:', result);
  } catch (error) {
    console.error('Transfer failed:', error.message);
  }
}
```

---

**🎯 Section 5 Complete! Questions 31-35 Finished**

Topics: Microservices, Event-driven architecture, Circuit breaker, CQRS, Distributed transactions
