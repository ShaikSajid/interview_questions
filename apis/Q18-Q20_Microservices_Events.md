# APIs Questions 18-20: Microservices & Event-Driven Architecture

---

### Q18. How do you implement API composition for microservices?

**Answer:**

```javascript
/**
 * API Composition Pattern for Microservices
 */
const axios = require('axios');
const { promisify } = require('util');

/**
 * API Composer Service
 */
class APIComposer {
  constructor() {
    this.services = {
      customer: 'http://customer-service:3001',
      account: 'http://account-service:3002',
      transaction: 'http://transaction-service:3003',
      loan: 'http://loan-service:3004',
      notification: 'http://notification-service:3005'
    };
    
    this.timeout = 5000;
  }
  
  /**
   * Sequential Composition: Get customer dashboard
   */
  async getCustomerDashboard(customerId, userId) {
    try {
      // 1. Get customer details
      const customer = await this.callService(
        'customer',
        `/customers/${customerId}`,
        { headers: { 'X-User-ID': userId } }
      );
      
      // 2. Get accounts (depends on customer)
      const accounts = await this.callService(
        'account',
        `/accounts?customerId=${customerId}`,
        { headers: { 'X-User-ID': userId } }
      );
      
      // 3. Get recent transactions for each account (parallel)
      const transactionPromises = accounts.map(account =>
        this.callService(
          'transaction',
          `/transactions?accountId=${account.id}&limit=5`,
          { headers: { 'X-User-ID': userId } }
        ).catch(err => {
          console.error(`Failed to get transactions for account ${account.id}:`, err.message);
          return []; // Fallback to empty array
        })
      );
      
      const allTransactions = await Promise.all(transactionPromises);
      
      // 4. Get active loans (parallel with transactions)
      const loans = await this.callService(
        'loan',
        `/loans?customerId=${customerId}&status=active`,
        { headers: { 'X-User-ID': userId } }
      ).catch(err => {
        console.error('Failed to get loans:', err.message);
        return []; // Fallback
      });
      
      // 5. Compose response
      return {
        customer: {
          id: customer.id,
          name: customer.name,
          email: customer.email,
          phone: customer.phone
        },
        accounts: accounts.map((account, index) => ({
          ...account,
          recentTransactions: allTransactions[index] || []
        })),
        loans: loans,
        summary: {
          totalBalance: accounts.reduce((sum, acc) => sum + acc.balance, 0),
          accountCount: accounts.length,
          loanCount: loans.length,
          totalLoanAmount: loans.reduce((sum, loan) => sum + loan.outstandingBalance, 0)
        }
      };
      
    } catch (error) {
      console.error('Dashboard composition failed:', error);
      throw new Error('Failed to load customer dashboard');
    }
  }
  
  /**
   * Parallel Composition: Get account summary
   */
  async getAccountSummary(accountId, userId) {
    try {
      // Fetch all data in parallel
      const [account, transactions, beneficiaries, statements] = await Promise.allSettled([
        this.callService('account', `/accounts/${accountId}`, { headers: { 'X-User-ID': userId } }),
        this.callService('transaction', `/transactions?accountId=${accountId}&limit=10`, { headers: { 'X-User-ID': userId } }),
        this.callService('account', `/accounts/${accountId}/beneficiaries`, { headers: { 'X-User-ID': userId } }),
        this.callService('account', `/accounts/${accountId}/statements?months=3`, { headers: { 'X-User-ID': userId } })
      ]);
      
      // Handle results
      return {
        account: account.status === 'fulfilled' ? account.value : null,
        transactions: transactions.status === 'fulfilled' ? transactions.value : [],
        beneficiaries: beneficiaries.status === 'fulfilled' ? beneficiaries.value : [],
        statements: statements.status === 'fulfilled' ? statements.value : [],
        errors: {
          account: account.status === 'rejected' ? account.reason.message : null,
          transactions: transactions.status === 'rejected' ? transactions.reason.message : null,
          beneficiaries: beneficiaries.status === 'rejected' ? beneficiaries.reason.message : null,
          statements: statements.status === 'rejected' ? statements.reason.message : null
        }
      };
      
    } catch (error) {
      throw new Error('Failed to load account summary');
    }
  }
  
  /**
   * Orchestrated Composition: Create transfer (saga pattern)
   */
  async createTransfer(transferData, userId) {
    const saga = [];
    
    try {
      // Step 1: Validate source account
      const sourceAccount = await this.callService(
        'account',
        `/accounts/${transferData.fromAccountId}`,
        { headers: { 'X-User-ID': userId } }
      );
      
      saga.push({
        service: 'account',
        operation: 'validateAccount',
        accountId: sourceAccount.id
      });
      
      if (sourceAccount.balance < transferData.amount) {
        throw new Error('Insufficient balance');
      }
      
      // Step 2: Reserve funds (debit source)
      const debitTransaction = await this.callService(
        'transaction',
        '/transactions',
        {
          method: 'POST',
          headers: { 'X-User-ID': userId },
          data: {
            accountId: transferData.fromAccountId,
            amount: transferData.amount,
            type: 'debit',
            status: 'pending',
            description: `Transfer to ${transferData.toIban}`
          }
        }
      );
      
      saga.push({
        service: 'transaction',
        operation: 'createDebit',
        transactionId: debitTransaction.id,
        compensate: () => this.callService('transaction', `/transactions/${debitTransaction.id}/cancel`, { method: 'POST' })
      });
      
      // Step 3: Send to external bank (via payment gateway)
      const paymentResult = await this.callService(
        'account',
        '/transfers/external',
        {
          method: 'POST',
          headers: { 'X-User-ID': userId },
          data: {
            toIban: transferData.toIban,
            amount: transferData.amount,
            beneficiaryName: transferData.beneficiaryName,
            reference: debitTransaction.referenceNumber
          }
        }
      );
      
      saga.push({
        service: 'account',
        operation: 'externalTransfer',
        paymentId: paymentResult.paymentId
      });
      
      // Step 4: Complete debit transaction
      await this.callService(
        'transaction',
        `/transactions/${debitTransaction.id}`,
        {
          method: 'PATCH',
          headers: { 'X-User-ID': userId },
          data: { status: 'completed' }
        }
      );
      
      // Step 5: Send notification
      await this.callService(
        'notification',
        '/notifications',
        {
          method: 'POST',
          headers: { 'X-User-ID': userId },
          data: {
            customerId: transferData.customerId,
            type: 'transfer_completed',
            message: `Transfer of AED ${transferData.amount} completed successfully`,
            transactionId: debitTransaction.id
          }
        }
      ).catch(err => {
        // Notification failure should not fail the transfer
        console.error('Failed to send notification:', err.message);
      });
      
      return {
        success: true,
        transactionId: debitTransaction.id,
        paymentId: paymentResult.paymentId,
        message: 'Transfer completed successfully'
      };
      
    } catch (error) {
      // Compensate saga
      console.error('Transfer failed, compensating...', error.message);
      
      for (let i = saga.length - 1; i >= 0; i--) {
        const step = saga[i];
        if (step.compensate) {
          try {
            await step.compensate();
            console.log(`Compensated: ${step.service}.${step.operation}`);
          } catch (compensateError) {
            console.error(`Failed to compensate ${step.service}.${step.operation}:`, compensateError.message);
          }
        }
      }
      
      throw new Error(`Transfer failed: ${error.message}`);
    }
  }
  
  /**
   * Call microservice
   */
  async callService(serviceName, path, options = {}) {
    const baseUrl = this.services[serviceName];
    
    if (!baseUrl) {
      throw new Error(`Unknown service: ${serviceName}`);
    }
    
    const config = {
      url: `${baseUrl}${path}`,
      method: options.method || 'GET',
      headers: options.headers || {},
      data: options.data,
      timeout: this.timeout
    };
    
    try {
      const response = await axios(config);
      return response.data;
    } catch (error) {
      if (error.response) {
        throw new Error(`${serviceName} returned ${error.response.status}: ${error.response.data?.message || 'Unknown error'}`);
      } else if (error.code === 'ECONNABORTED') {
        throw new Error(`${serviceName} timeout`);
      } else {
        throw new Error(`${serviceName} unavailable: ${error.message}`);
      }
    }
  }
}

/**
 * GraphQL API Composition (Alternative)
 */
const { ApolloServer, gql } = require('apollo-server-express');
const { RESTDataSource } = require('apollo-datasource-rest');

class CustomerServiceAPI extends RESTDataSource {
  constructor() {
    super();
    this.baseURL = 'http://customer-service:3001';
  }
  
  async getCustomer(customerId) {
    return this.get(`/customers/${customerId}`);
  }
}

class AccountServiceAPI extends RESTDataSource {
  constructor() {
    super();
    this.baseURL = 'http://account-service:3002';
  }
  
  async getAccounts(customerId) {
    return this.get(`/accounts?customerId=${customerId}`);
  }
  
  async getAccount(accountId) {
    return this.get(`/accounts/${accountId}`);
  }
}

class TransactionServiceAPI extends RESTDataSource {
  constructor() {
    super();
    this.baseURL = 'http://transaction-service:3003';
  }
  
  async getTransactions(accountId, limit = 10) {
    return this.get(`/transactions?accountId=${accountId}&limit=${limit}`);
  }
}

const typeDefs = gql`
  type Customer {
    id: ID!
    name: String!
    email: String!
    accounts: [Account!]!
    dashboard: Dashboard!
  }
  
  type Account {
    id: ID!
    accountNumber: String!
    balance: Float!
    transactions(limit: Int): [Transaction!]!
  }
  
  type Transaction {
    id: ID!
    amount: Float!
    type: String!
    description: String
    createdAt: String!
  }
  
  type Dashboard {
    totalBalance: Float!
    accountCount: Int!
    recentActivity: [Transaction!]!
  }
  
  type Query {
    customer(id: ID!): Customer
    account(id: ID!): Account
  }
`;

const resolvers = {
  Query: {
    customer: async (_, { id }, { dataSources }) => {
      return dataSources.customerAPI.getCustomer(id);
    },
    account: async (_, { id }, { dataSources }) => {
      return dataSources.accountAPI.getAccount(id);
    }
  },
  
  Customer: {
    accounts: async (parent, _, { dataSources }) => {
      return dataSources.accountAPI.getAccounts(parent.id);
    },
    dashboard: async (parent, _, { dataSources }) => {
      const accounts = await dataSources.accountAPI.getAccounts(parent.id);
      
      const transactionsPromises = accounts.map(account =>
        dataSources.transactionAPI.getTransactions(account.id, 5)
      );
      
      const allTransactions = await Promise.all(transactionsPromises);
      const recentActivity = allTransactions.flat().sort((a, b) =>
        new Date(b.createdAt) - new Date(a.createdAt)
      ).slice(0, 10);
      
      return {
        totalBalance: accounts.reduce((sum, acc) => sum + acc.balance, 0),
        accountCount: accounts.length,
        recentActivity
      };
    }
  },
  
  Account: {
    transactions: async (parent, { limit = 10 }, { dataSources }) => {
      return dataSources.transactionAPI.getTransactions(parent.id, limit);
    }
  }
};

/**
 * Express Routes
 */
const express = require('express');
const router = express.Router();

const composer = new APIComposer();

// Composed endpoint: Customer dashboard
router.get('/api/dashboard/:customerId', async (req, res) => {
  try {
    const dashboard = await composer.getCustomerDashboard(
      req.params.customerId,
      req.user.userId
    );
    
    res.json(dashboard);
  } catch (error) {
    res.status(500).json({
      error: 'InternalServerError',
      message: error.message
    });
  }
});

// Composed endpoint: Account summary
router.get('/api/accounts/:accountId/summary', async (req, res) => {
  try {
    const summary = await composer.getAccountSummary(
      req.params.accountId,
      req.user.userId
    );
    
    res.json(summary);
  } catch (error) {
    res.status(500).json({
      error: 'InternalServerError',
      message: error.message
    });
  }
});

// Orchestrated endpoint: Create transfer
router.post('/api/transfers', async (req, res) => {
  try {
    const result = await composer.createTransfer(req.body, req.user.userId);
    
    res.status(201).json(result);
  } catch (error) {
    res.status(400).json({
      error: 'TransferFailed',
      message: error.message
    });
  }
});

module.exports = { APIComposer, typeDefs, resolvers };
```

---

### Q19. How do you implement Event-Driven APIs with message queues?

**Answer:**

```javascript
/**
 * Event-Driven Architecture with RabbitMQ and Kafka
 */
const amqp = require('amqplib');
const { Kafka } = require('kafkajs');
const { EventEmitter } = require('events');

/**
 * Event Bus using RabbitMQ
 */
class RabbitMQEventBus {
  constructor(url = 'amqp://localhost') {
    this.url = url;
    this.connection = null;
    this.channel = null;
    this.exchanges = {
      banking: 'banking.events',
      notifications: 'notifications.events',
      analytics: 'analytics.events'
    };
  }
  
  /**
   * Connect to RabbitMQ
   */
  async connect() {
    this.connection = await amqp.connect(this.url);
    this.channel = await this.connection.createChannel();
    
    // Create exchanges
    for (const [name, exchange] of Object.entries(this.exchanges)) {
      await this.channel.assertExchange(exchange, 'topic', { durable: true });
      console.log(`Exchange created: ${exchange}`);
    }
    
    // Handle connection errors
    this.connection.on('error', (err) => {
      console.error('RabbitMQ connection error:', err);
    });
    
    this.connection.on('close', () => {
      console.log('RabbitMQ connection closed');
    });
  }
  
  /**
   * Publish event
   */
  async publish(exchange, routingKey, event) {
    const message = JSON.stringify({
      ...event,
      timestamp: new Date().toISOString(),
      eventId: require('crypto').randomUUID()
    });
    
    const published = this.channel.publish(
      this.exchanges[exchange],
      routingKey,
      Buffer.from(message),
      {
        persistent: true,
        contentType: 'application/json'
      }
    );
    
    if (!published) {
      console.error('Failed to publish event');
    }
    
    console.log(`Published event: ${routingKey} to ${exchange}`);
  }
  
  /**
   * Subscribe to events
   */
  async subscribe(exchange, routingKeys, queueName, handler) {
    // Create queue
    const queue = await this.channel.assertQueue(queueName, {
      durable: true,
      arguments: {
        'x-message-ttl': 86400000, // 24 hours
        'x-max-length': 10000
      }
    });
    
    // Bind routing keys
    for (const routingKey of routingKeys) {
      await this.channel.bindQueue(
        queue.queue,
        this.exchanges[exchange],
        routingKey
      );
      console.log(`Queue ${queueName} bound to ${routingKey}`);
    }
    
    // Consume messages
    this.channel.consume(queue.queue, async (msg) => {
      if (!msg) return;
      
      try {
        const event = JSON.parse(msg.content.toString());
        
        console.log(`Received event: ${msg.fields.routingKey}`);
        
        await handler(event, msg.fields.routingKey);
        
        // Acknowledge
        this.channel.ack(msg);
        
      } catch (error) {
        console.error('Error processing event:', error);
        
        // Reject and requeue
        this.channel.nack(msg, false, true);
      }
    }, {
      noAck: false
    });
  }
  
  async close() {
    await this.channel.close();
    await this.connection.close();
  }
}

/**
 * Banking Events Publisher
 */
class BankingEventPublisher {
  constructor(eventBus) {
    this.eventBus = eventBus;
  }
  
  /**
   * Publish transaction created event
   */
  async transactionCreated(transaction) {
    await this.eventBus.publish('banking', 'transaction.created', {
      type: 'TransactionCreated',
      data: {
        transactionId: transaction.id,
        accountId: transaction.accountId,
        customerId: transaction.customerId,
        amount: transaction.amount,
        type: transaction.type,
        status: transaction.status,
        description: transaction.description
      }
    });
  }
  
  /**
   * Publish account balance updated
   */
  async accountBalanceUpdated(account) {
    await this.eventBus.publish('banking', 'account.balance.updated', {
      type: 'AccountBalanceUpdated',
      data: {
        accountId: account.id,
        customerId: account.customerId,
        oldBalance: account.oldBalance,
        newBalance: account.balance,
        difference: account.balance - account.oldBalance
      }
    });
  }
  
  /**
   * Publish fraud detected
   */
  async fraudDetected(transaction, fraudScore) {
    await this.eventBus.publish('banking', 'fraud.detected', {
      type: 'FraudDetected',
      priority: 'high',
      data: {
        transactionId: transaction.id,
        accountId: transaction.accountId,
        customerId: transaction.customerId,
        amount: transaction.amount,
        fraudScore,
        riskLevel: fraudScore > 75 ? 'HIGH' : 'MEDIUM'
      }
    });
  }
  
  /**
   * Publish loan application submitted
   */
  async loanApplicationSubmitted(loan) {
    await this.eventBus.publish('banking', 'loan.application.submitted', {
      type: 'LoanApplicationSubmitted',
      data: {
        loanId: loan.id,
        customerId: loan.customerId,
        loanType: loan.loanType,
        amount: loan.amount,
        tenureMonths: loan.tenureMonths
      }
    });
  }
}

/**
 * Event Handlers (Consumers)
 */
class NotificationEventHandler {
  constructor(notificationService) {
    this.notificationService = notificationService;
  }
  
  async handleTransactionCreated(event) {
    const { data } = event;
    
    await this.notificationService.sendNotification({
      customerId: data.customerId,
      type: 'transaction',
      title: 'Transaction Alert',
      message: `${data.type === 'debit' ? 'Debit' : 'Credit'} of AED ${data.amount}`,
      transactionId: data.transactionId
    });
  }
  
  async handleFraudDetected(event) {
    const { data } = event;
    
    await this.notificationService.sendUrgentAlert({
      customerId: data.customerId,
      type: 'fraud_alert',
      title: 'Suspicious Activity Detected',
      message: `Transaction of AED ${data.amount} flagged for review`,
      transactionId: data.transactionId,
      riskLevel: data.riskLevel
    });
  }
}

/**
 * Kafka Event Streaming (for high-throughput)
 */
class KafkaEventStream {
  constructor() {
    this.kafka = new Kafka({
      clientId: 'enbd-banking',
      brokers: ['localhost:9092', 'localhost:9093', 'localhost:9094'],
      retry: {
        initialRetryTime: 100,
        retries: 8
      }
    });
    
    this.producer = this.kafka.producer({
      allowAutoTopicCreation: true,
      transactionTimeout: 30000
    });
    
    this.consumers = new Map();
  }
  
  /**
   * Connect producer
   */
  async connectProducer() {
    await this.producer.connect();
    console.log('Kafka producer connected');
  }
  
  /**
   * Produce event
   */
  async produce(topic, event) {
    const message = {
      key: event.eventId || require('crypto').randomUUID(),
      value: JSON.stringify(event),
      timestamp: Date.now().toString(),
      headers: {
        'event-type': event.type,
        'correlation-id': event.correlationId || ''
      }
    };
    
    await this.producer.send({
      topic,
      messages: [message],
      compression: 1 // GZIP
    });
    
    console.log(`Produced to ${topic}:`, event.type);
  }
  
  /**
   * Produce batch
   */
  async produceBatch(topic, events) {
    const messages = events.map(event => ({
      key: event.eventId,
      value: JSON.stringify(event),
      timestamp: Date.now().toString()
    }));
    
    await this.producer.sendBatch({
      topicMessages: [{
        topic,
        messages
      }]
    });
  }
  
  /**
   * Consume events
   */
  async consume(topic, groupId, handler) {
    const consumer = this.kafka.consumer({
      groupId,
      sessionTimeout: 30000,
      heartbeatInterval: 3000
    });
    
    await consumer.connect();
    await consumer.subscribe({ topic, fromBeginning: false });
    
    await consumer.run({
      autoCommit: false,
      eachMessage: async ({ topic, partition, message }) => {
        try {
          const event = JSON.parse(message.value.toString());
          
          console.log(`Consumed from ${topic} [${partition}]:`, event.type);
          
          await handler(event);
          
          // Commit offset
          await consumer.commitOffsets([{
            topic,
            partition,
            offset: (parseInt(message.offset) + 1).toString()
          }]);
          
        } catch (error) {
          console.error('Error processing message:', error);
          // Don't commit offset, will retry
        }
      }
    });
    
    this.consumers.set(groupId, consumer);
  }
  
  /**
   * Disconnect
   */
  async disconnect() {
    await this.producer.disconnect();
    
    for (const [groupId, consumer] of this.consumers) {
      await consumer.disconnect();
    }
  }
}

/**
 * Event Sourcing Pattern
 */
class EventStore {
  constructor(db) {
    this.db = db;
  }
  
  /**
   * Append event to stream
   */
  async appendEvent(streamId, event) {
    await this.db.query(`
      INSERT INTO events (stream_id, event_type, event_data, version, created_at)
      VALUES ($1, $2, $3, 
        COALESCE((SELECT MAX(version) FROM events WHERE stream_id = $1), 0) + 1,
        NOW())
      RETURNING *
    `, [streamId, event.type, JSON.stringify(event.data)]);
  }
  
  /**
   * Get event stream
   */
  async getEventStream(streamId, fromVersion = 0) {
    const result = await this.db.query(`
      SELECT * FROM events
      WHERE stream_id = $1 AND version > $2
      ORDER BY version ASC
    `, [streamId, fromVersion]);
    
    return result.rows.map(row => ({
      ...JSON.parse(row.event_data),
      type: row.event_type,
      version: row.version,
      createdAt: row.created_at
    }));
  }
  
  /**
   * Rebuild aggregate from events
   */
  async rebuildAggregate(streamId) {
    const events = await this.getEventStream(streamId);
    
    let state = { balance: 0, transactions: [] };
    
    for (const event of events) {
      switch (event.type) {
        case 'AccountCreated':
          state.accountId = event.accountId;
          state.balance = 0;
          break;
          
        case 'TransactionCreated':
          if (event.type === 'credit') {
            state.balance += event.amount;
          } else {
            state.balance -= event.amount;
          }
          state.transactions.push(event);
          break;
      }
    }
    
    return state;
  }
}

/**
 * Usage Example
 */
async function main() {
  // RabbitMQ Event Bus
  const eventBus = new RabbitMQEventBus();
  await eventBus.connect();
  
  const publisher = new BankingEventPublisher(eventBus);
  
  // Publish events
  await publisher.transactionCreated({
    id: '123',
    accountId: 'acc-456',
    customerId: 'cust-789',
    amount: 500,
    type: 'debit',
    status: 'completed'
  });
  
  // Subscribe to events
  const notificationHandler = new NotificationEventHandler(notificationService);
  
  await eventBus.subscribe(
    'banking',
    ['transaction.*', 'fraud.*'],
    'notification-service',
    async (event, routingKey) => {
      if (routingKey === 'transaction.created') {
        await notificationHandler.handleTransactionCreated(event);
      } else if (routingKey === 'fraud.detected') {
        await notificationHandler.handleFraudDetected(event);
      }
    }
  );
}

module.exports = {
  RabbitMQEventBus,
  BankingEventPublisher,
  KafkaEventStream,
  EventStore
};
```

---

### Q20. How do you implement CQRS pattern for APIs?

**Answer:**

```javascript
/**
 * CQRS (Command Query Responsibility Segregation) Pattern
 */
const { EventEmitter } = require('events');

/**
 * Command Side (Write Model)
 */
class BankingCommandHandler {
  constructor(db, eventBus) {
    this.db = db;
    this.eventBus = eventBus;
  }
  
  /**
   * Command: Create Account
   */
  async createAccount(command) {
    const { customerId, accountType, currency } = command;
    
    // Validate
    if (!['savings', 'current', 'fixed_deposit'].includes(accountType)) {
      throw new Error('Invalid account type');
    }
    
    // Generate account number
    const accountNumber = this.generateAccountNumber();
    
    // Execute command (write to DB)
    const result = await this.db.query(`
      INSERT INTO accounts (customer_id, account_number, account_type, currency, balance, status)
      VALUES ($1, $2, $3, $4, 0, 'active')
      RETURNING *
    `, [customerId, accountNumber, accountType, currency]);
    
    const account = result.rows[0];
    
    // Publish event
    await this.eventBus.publish('banking', 'account.created', {
      type: 'AccountCreated',
      data: {
        accountId: account.id,
        customerId: account.customer_id,
        accountNumber: account.account_number,
        accountType: account.account_type,
        currency: account.currency
      }
    });
    
    return { accountId: account.id, accountNumber: account.account_number };
  }
  
  /**
   * Command: Create Transaction
   */
  async createTransaction(command) {
    const { accountId, amount, type, description } = command;
    
    // Start transaction
    const client = await this.db.connect();
    
    try {
      await client.query('BEGIN');
      
      // Get account (with row lock)
      const accountResult = await client.query(
        'SELECT * FROM accounts WHERE id = $1 FOR UPDATE',
        [accountId]
      );
      
      if (accountResult.rows.length === 0) {
        throw new Error('Account not found');
      }
      
      const account = accountResult.rows[0];
      
      // Validate balance for debit
      if (type === 'debit' && account.balance < amount) {
        throw new Error('Insufficient balance');
      }
      
      // Calculate new balance
      const newBalance = type === 'debit'
        ? account.balance - amount
        : account.balance + amount;
      
      // Create transaction
      const txnResult = await client.query(`
        INSERT INTO transactions (account_id, amount, type, description, balance_after, status)
        VALUES ($1, $2, $3, $4, $5, 'completed')
        RETURNING *
      `, [accountId, amount, type, description, newBalance]);
      
      const transaction = txnResult.rows[0];
      
      // Update account balance
      await client.query(
        'UPDATE accounts SET balance = $1, updated_at = NOW() WHERE id = $2',
        [newBalance, accountId]
      );
      
      await client.query('COMMIT');
      
      // Publish events
      await this.eventBus.publish('banking', 'transaction.created', {
        type: 'TransactionCreated',
        data: {
          transactionId: transaction.id,
          accountId: transaction.account_id,
          customerId: account.customer_id,
          amount: transaction.amount,
          type: transaction.type,
          balanceAfter: transaction.balance_after
        }
      });
      
      await this.eventBus.publish('banking', 'account.balance.updated', {
        type: 'AccountBalanceUpdated',
        data: {
          accountId,
          customerId: account.customer_id,
          oldBalance: account.balance,
          newBalance
        }
      });
      
      return { transactionId: transaction.id, balanceAfter: newBalance };
      
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
  
  generateAccountNumber() {
    return '1234' + Date.now().toString().substr(-12);
  }
}

/**
 * Query Side (Read Model)
 */
class BankingQueryHandler {
  constructor(readDb, cache) {
    this.readDb = readDb; // Can be different DB (MongoDB, Elasticsearch)
    this.cache = cache;
  }
  
  /**
   * Query: Get Account Summary
   */
  async getAccountSummary(accountId) {
    // Check cache
    const cacheKey = `account_summary:${accountId}`;
    const cached = await this.cache.get(cacheKey);
    
    if (cached) {
      return JSON.parse(cached);
    }
    
    // Query from read model
    const summary = await this.readDb.collection('account_summaries').findOne({
      accountId
    });
    
    if (!summary) {
      throw new Error('Account not found');
    }
    
    // Cache for 30 seconds
    await this.cache.setex(cacheKey, 30, JSON.stringify(summary));
    
    return summary;
  }
  
  /**
   * Query: Get Transaction History
   */
  async getTransactionHistory(accountId, filters = {}) {
    const { startDate, endDate, type, limit = 20, offset = 0 } = filters;
    
    const query = { accountId };
    
    if (startDate || endDate) {
      query.createdAt = {};
      if (startDate) query.createdAt.$gte = new Date(startDate);
      if (endDate) query.createdAt.$lte = new Date(endDate);
    }
    
    if (type) {
      query.type = type;
    }
    
    const transactions = await this.readDb.collection('transactions')
      .find(query)
      .sort({ createdAt: -1 })
      .skip(offset)
      .limit(limit)
      .toArray();
    
    const total = await this.readDb.collection('transactions').countDocuments(query);
    
    return {
      transactions,
      pagination: {
        limit,
        offset,
        total,
        hasMore: offset + limit < total
      }
    };
  }
  
  /**
   * Query: Get Customer Dashboard
   */
  async getCustomerDashboard(customerId) {
    const cacheKey = `dashboard:${customerId}`;
    const cached = await this.cache.get(cacheKey);
    
    if (cached) {
      return JSON.parse(cached);
    }
    
    const dashboard = await this.readDb.collection('customer_dashboards').findOne({
      customerId
    });
    
    await this.cache.setex(cacheKey, 60, JSON.stringify(dashboard));
    
    return dashboard;
  }
}

/**
 * Read Model Projections (Event Handlers)
 */
class ReadModelProjection {
  constructor(readDb, cache) {
    this.readDb = readDb;
    this.cache = cache;
  }
  
  /**
   * Handle AccountCreated event
   */
  async onAccountCreated(event) {
    const { data } = event;
    
    await this.readDb.collection('account_summaries').insertOne({
      accountId: data.accountId,
      customerId: data.customerId,
      accountNumber: data.accountNumber,
      accountType: data.accountType,
      currency: data.currency,
      balance: 0,
      transactionCount: 0,
      lastTransactionAt: null,
      createdAt: new Date()
    });
    
    // Invalidate customer dashboard cache
    await this.cache.del(`dashboard:${data.customerId}`);
  }
  
  /**
   * Handle TransactionCreated event
   */
  async onTransactionCreated(event) {
    const { data } = event;
    
    // Update account summary
    await this.readDb.collection('account_summaries').updateOne(
      { accountId: data.accountId },
      {
        $set: {
          balance: data.balanceAfter,
          lastTransactionAt: new Date()
        },
        $inc: { transactionCount: 1 }
      }
    );
    
    // Add to transactions collection
    await this.readDb.collection('transactions').insertOne({
      transactionId: data.transactionId,
      accountId: data.accountId,
      customerId: data.customerId,
      amount: data.amount,
      type: data.type,
      balanceAfter: data.balanceAfter,
      createdAt: new Date()
    });
    
    // Invalidate caches
    await this.cache.del(`account_summary:${data.accountId}`);
    await this.cache.del(`dashboard:${data.customerId}`);
  }
  
  /**
   * Handle AccountBalanceUpdated event
   */
  async onAccountBalanceUpdated(event) {
    const { data } = event;
    
    // Update customer dashboard
    await this.readDb.collection('customer_dashboards').updateOne(
      { customerId: data.customerId },
      {
        $set: {
          [`accounts.${data.accountId}.balance`]: data.newBalance,
          totalBalance: data.newBalance, // Simplified
          updatedAt: new Date()
        }
      },
      { upsert: true }
    );
    
    await this.cache.del(`dashboard:${data.customerId}`);
  }
}

/**
 * Express API Routes
 */
const express = require('express');
const router = express.Router();

// Commands (POST/PUT/DELETE)
router.post('/api/commands/accounts', async (req, res) => {
  try {
    const result = await commandHandler.createAccount(req.body);
    
    res.status(202).json({
      message: 'Command accepted',
      ...result
    });
  } catch (error) {
    res.status(400).json({
      error: 'CommandFailed',
      message: error.message
    });
  }
});

router.post('/api/commands/transactions', async (req, res) => {
  try {
    const result = await commandHandler.createTransaction(req.body);
    
    res.status(202).json({
      message: 'Command accepted',
      ...result
    });
  } catch (error) {
    res.status(400).json({
      error: 'CommandFailed',
      message: error.message
    });
  }
});

// Queries (GET)
router.get('/api/queries/accounts/:accountId/summary', async (req, res) => {
  try {
    const summary = await queryHandler.getAccountSummary(req.params.accountId);
    
    res.json(summary);
  } catch (error) {
    res.status(404).json({
      error: 'NotFound',
      message: error.message
    });
  }
});

router.get('/api/queries/accounts/:accountId/transactions', async (req, res) => {
  try {
    const transactions = await queryHandler.getTransactionHistory(
      req.params.accountId,
      req.query
    );
    
    res.json(transactions);
  } catch (error) {
    res.status(500).json({
      error: 'QueryFailed',
      message: error.message
    });
  }
});

router.get('/api/queries/customers/:customerId/dashboard', async (req, res) => {
  try {
    const dashboard = await queryHandler.getCustomerDashboard(req.params.customerId);
    
    res.json(dashboard);
  } catch (error) {
    res.status(404).json({
      error: 'NotFound',
      message: error.message
    });
  }
});

module.exports = {
  BankingCommandHandler,
  BankingQueryHandler,
  ReadModelProjection
};
```

---

**Summary Q18-Q20:**
- API Composition patterns (sequential, parallel, orchestrated/saga) with fallbacks ✅
- Event-Driven Architecture with RabbitMQ/Kafka, event sourcing, pub/sub patterns ✅
- CQRS pattern with separate command/query handlers, read model projections, caching ✅
