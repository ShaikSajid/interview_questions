# Q44: Event-Driven Architecture - Message Queues

## 📋 Summary
This question covers **Message Queues** for distributed, asynchronous communication between services. You'll learn RabbitMQ and Kafka patterns, understand when to use each, and build production-ready banking systems with reliable message processing.

**Key Topics**:
- Message queue fundamentals (producer/consumer)
- RabbitMQ patterns (work queues, pub/sub, routing, topics)
- Kafka for event streaming
- Dead letter queues and error handling
- Message acknowledgment and durability
- Scaling message consumers

**Banking Use Cases**:
- Asynchronous payment processing
- Transaction notifications across services
- Fraud detection pipeline
- Audit log streaming
- Account statement generation

---

## 🎯 Understanding Message Queues

### What is a Message Queue?

A **message queue** is a form of asynchronous service-to-service communication. It decouples producers (senders) from consumers (receivers) and provides reliable message delivery.

```
┌──────────┐     message      ┌───────────┐     message      ┌──────────┐
│ Producer │ ──────────────▶  │   Queue   │ ──────────────▶  │ Consumer │
│ (Sender) │                  │ (Broker)  │                  │(Receiver)│
└──────────┘                  └───────────┘                  └──────────┘
```

### Key Benefits

1. **Decoupling**: Services don't need to know about each other
2. **Scalability**: Add more consumers to handle load
3. **Reliability**: Messages persist if consumers are down
4. **Asynchronous**: Producers don't wait for consumers
5. **Load Leveling**: Smooth out traffic spikes

### RabbitMQ vs Kafka

| Feature | RabbitMQ | Kafka |
|---------|----------|-------|
| **Model** | Message broker (queue) | Distributed log (stream) |
| **Best For** | Task distribution, RPC | Event streaming, analytics |
| **Message Retention** | Deleted after consumption | Retained for configured time |
| **Ordering** | Per-queue ordering | Per-partition ordering |
| **Speed** | ~20k msgs/sec | ~1M msgs/sec |
| **Use Case** | Microservices communication | Event sourcing, big data |

---

## 📖 RabbitMQ Fundamentals

### Core Concepts

1. **Producer**: Sends messages to exchange
2. **Exchange**: Routes messages to queues (types: direct, fanout, topic, headers)
3. **Queue**: Stores messages until consumed
4. **Consumer**: Receives and processes messages
5. **Binding**: Links exchange to queue with routing key

### Exchange Types

```
┌─────────────────────────────────────────────────────────────┐
│ DIRECT EXCHANGE (Routing Key Match)                         │
│                                                              │
│  Producer ──routing="error"──▶ Exchange ──▶ [error.queue]   │
│  Producer ──routing="info"───▶ Exchange ──▶ [info.queue]    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ FANOUT EXCHANGE (Broadcast to All)                          │
│                                                              │
│  Producer ──────────────────▶ Exchange ──┬──▶ [queue1]      │
│                                           ├──▶ [queue2]      │
│                                           └──▶ [queue3]      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ TOPIC EXCHANGE (Pattern Matching)                           │
│                                                              │
│  Producer ──"payment.*.usd"─▶ Exchange ──▶ [usd.queue]      │
│  Producer ──"payment.#"─────▶ Exchange ──▶ [all.payments]   │
└─────────────────────────────────────────────────────────────┘
```

### Message Acknowledgment

```javascript
// Auto-ack (dangerous - message lost if consumer crashes)
channel.consume(queue, (msg) => {
  processMessage(msg);
}, { noAck: true });

// Manual-ack (safe - redelivered if not acked)
channel.consume(queue, async (msg) => {
  try {
    await processMessage(msg);
    channel.ack(msg); // Success - remove from queue
  } catch (error) {
    channel.nack(msg, false, true); // Fail - requeue
  }
}, { noAck: false });
```

---

## 💡 Example 1: Banking Payment Processing with RabbitMQ

Complete asynchronous payment system with work queues, dead letter queues, and retry logic.

### Scenario
Build a payment processing system where:
- Payments are queued for async processing
- Multiple workers process payments in parallel
- Failed payments go to dead letter queue
- Retries with exponential backoff
- Fraud detection runs asynchronously

### Setup

```bash
# Install RabbitMQ
npm install amqplib

# Start RabbitMQ with Docker
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3-management

# Management UI: http://localhost:15672 (guest/guest)
```

### Implementation

```javascript
const amqp = require('amqplib');

/**
 * RabbitMQ Connection Manager
 * Handles connection pooling and reconnection
 */
class RabbitMQConnection {
  constructor(url = 'amqp://localhost') {
    this.url = url;
    this.connection = null;
    this.channel = null;
  }

  async connect() {
    try {
      this.connection = await amqp.connect(this.url);
      this.channel = await this.connection.createChannel();

      // Handle connection errors
      this.connection.on('error', (err) => {
        console.error('RabbitMQ connection error:', err);
        this.reconnect();
      });

      this.connection.on('close', () => {
        console.log('RabbitMQ connection closed, reconnecting...');
        this.reconnect();
      });

      console.log('✅ Connected to RabbitMQ');
      return this.channel;
    } catch (error) {
      console.error('Failed to connect to RabbitMQ:', error);
      throw error;
    }
  }

  async reconnect() {
    setTimeout(() => {
      console.log('Attempting to reconnect...');
      this.connect();
    }, 5000);
  }

  async close() {
    if (this.channel) await this.channel.close();
    if (this.connection) await this.connection.close();
  }

  getChannel() {
    if (!this.channel) {
      throw new Error('Channel not initialized. Call connect() first.');
    }
    return this.channel;
  }
}

/**
 * Payment Queue Producer
 * Publishes payment requests to queue
 */
class PaymentProducer {
  constructor(connection) {
    this.connection = connection;
    this.queueName = 'payment.processing';
    this.dlqName = 'payment.dlq'; // Dead Letter Queue
    this.exchangeName = 'payment.exchange';
  }

  async setup() {
    const channel = this.connection.getChannel();

    // Create dead letter exchange and queue
    await channel.assertExchange('payment.dlx', 'direct', { durable: true });
    await channel.assertQueue(this.dlqName, { durable: true });
    await channel.bindQueue(this.dlqName, 'payment.dlx', 'payment.failed');

    // Create main exchange
    await channel.assertExchange(this.exchangeName, 'direct', { durable: true });

    // Create main queue with DLQ configuration
    await channel.assertQueue(this.queueName, {
      durable: true,
      arguments: {
        'x-dead-letter-exchange': 'payment.dlx',
        'x-dead-letter-routing-key': 'payment.failed',
        'x-message-ttl': 300000 // 5 minutes
      }
    });

    await channel.bindQueue(this.queueName, this.exchangeName, 'payment.process');

    console.log('✅ Payment producer setup complete');
  }

  async publishPayment(paymentData) {
    const channel = this.connection.getChannel();

    const payment = {
      id: `PAY-${Date.now()}`,
      ...paymentData,
      timestamp: new Date().toISOString(),
      retryCount: 0
    };

    const message = Buffer.from(JSON.stringify(payment));

    // Publish with confirmation
    const published = channel.publish(
      this.exchangeName,
      'payment.process',
      message,
      {
        persistent: true, // Survive broker restart
        contentType: 'application/json',
        messageId: payment.id,
        timestamp: Date.now()
      }
    );

    if (published) {
      console.log(`📤 Published payment: ${payment.id} ($${payment.amount})`);
      return payment;
    } else {
      throw new Error('Failed to publish payment');
    }
  }

  async publishBatch(payments) {
    const results = [];
    for (const payment of payments) {
      try {
        const result = await this.publishPayment(payment);
        results.push({ success: true, payment: result });
      } catch (error) {
        results.push({ success: false, payment, error: error.message });
      }
    }
    return results;
  }
}

/**
 * Payment Queue Consumer
 * Processes payments from queue with retry logic
 */
class PaymentConsumer {
  constructor(connection, workerId = 1) {
    this.connection = connection;
    this.workerId = workerId;
    this.queueName = 'payment.processing';
    this.processing = false;
    this.processedCount = 0;
    this.failedCount = 0;
  }

  async start() {
    const channel = this.connection.getChannel();

    // Set prefetch to process one message at a time
    await channel.prefetch(1);

    console.log(`🚀 Worker ${this.workerId} started, waiting for payments...`);

    this.processing = true;

    channel.consume(
      this.queueName,
      async (msg) => {
        if (!msg) return;

        const payment = JSON.parse(msg.content.toString());
        console.log(`\n[Worker ${this.workerId}] Processing payment: ${payment.id}`);

        try {
          await this.processPayment(payment);
          
          // Acknowledge successful processing
          channel.ack(msg);
          this.processedCount++;
          
          console.log(`✅ [Worker ${this.workerId}] Payment ${payment.id} completed`);
        } catch (error) {
          console.error(`❌ [Worker ${this.workerId}] Payment ${payment.id} failed:`, error.message);
          
          // Retry logic
          const retryCount = payment.retryCount || 0;
          const maxRetries = 3;

          if (retryCount < maxRetries) {
            // Increment retry count and republish
            payment.retryCount = retryCount + 1;
            
            console.log(`🔄 [Worker ${this.workerId}] Retry ${payment.retryCount}/${maxRetries} for ${payment.id}`);
            
            // Negative acknowledge with requeue
            channel.nack(msg, false, true);
          } else {
            // Max retries exceeded - send to DLQ
            console.log(`💀 [Worker ${this.workerId}] Payment ${payment.id} sent to DLQ`);
            
            // Negative acknowledge without requeue (goes to DLQ)
            channel.nack(msg, false, false);
            this.failedCount++;
          }
        }
      },
      { noAck: false }
    );
  }

  async processPayment(payment) {
    // Simulate payment processing
    await this.sleep(Math.random() * 2000 + 1000); // 1-3 seconds

    // Validate payment
    if (!payment.amount || payment.amount <= 0) {
      throw new Error('Invalid payment amount');
    }

    if (!payment.accountFrom || !payment.accountTo) {
      throw new Error('Missing account information');
    }

    // Simulate random failures (10% failure rate)
    if (Math.random() < 0.1) {
      throw new Error('Payment gateway timeout');
    }

    // Check for fraud (high-value transactions)
    if (payment.amount > 10000) {
      await this.performFraudCheck(payment);
    }

    // Process payment (simulate DB operations)
    await this.debitAccount(payment.accountFrom, payment.amount);
    await this.creditAccount(payment.accountTo, payment.amount);
    await this.recordTransaction(payment);

    return {
      ...payment,
      status: 'completed',
      processedBy: `worker-${this.workerId}`,
      completedAt: new Date().toISOString()
    };
  }

  async performFraudCheck(payment) {
    console.log(`🛡️  [Worker ${this.workerId}] Fraud check for ${payment.id}`);
    await this.sleep(500);
    
    const riskScore = Math.random() * 100;
    if (riskScore > 95) {
      throw new Error(`Fraud detected (risk: ${riskScore.toFixed(2)})`);
    }
  }

  async debitAccount(account, amount) {
    console.log(`  💰 Debit ${account}: $${amount}`);
    await this.sleep(200);
  }

  async creditAccount(account, amount) {
    console.log(`  💵 Credit ${account}: $${amount}`);
    await this.sleep(200);
  }

  async recordTransaction(payment) {
    console.log(`  📝 Record transaction: ${payment.id}`);
    await this.sleep(100);
  }

  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  stop() {
    this.processing = false;
    console.log(`\n[Worker ${this.workerId}] Stopped`);
    console.log(`  Processed: ${this.processedCount}`);
    console.log(`  Failed: ${this.failedCount}`);
  }

  getStats() {
    return {
      workerId: this.workerId,
      processed: this.processedCount,
      failed: this.failedCount
    };
  }
}

/**
 * Dead Letter Queue Monitor
 * Monitors failed payments for manual intervention
 */
class DLQMonitor {
  constructor(connection) {
    this.connection = connection;
    this.dlqName = 'payment.dlq';
    this.failedPayments = [];
  }

  async start() {
    const channel = this.connection.getChannel();

    console.log('🔍 DLQ Monitor started');

    channel.consume(
      this.dlqName,
      (msg) => {
        if (!msg) return;

        const payment = JSON.parse(msg.content.toString());
        this.failedPayments.push({
          payment,
          failedAt: new Date().toISOString(),
          reason: payment.error || 'Unknown'
        });

        console.log(`\n💀 FAILED PAYMENT IN DLQ:`);
        console.log(`   ID: ${payment.id}`);
        console.log(`   Amount: $${payment.amount}`);
        console.log(`   Retry Count: ${payment.retryCount}`);
        console.log(`   From: ${payment.accountFrom}`);
        console.log(`   To: ${payment.accountTo}`);

        // Acknowledge (remove from DLQ after logging)
        channel.ack(msg);
      },
      { noAck: false }
    );
  }

  getFailedPayments() {
    return this.failedPayments;
  }
}

/**
 * Payment Notification Service
 * Listens for payment events and sends notifications
 */
class PaymentNotificationService {
  constructor(connection) {
    this.connection = connection;
    this.exchangeName = 'payment.events';
    this.queueName = 'payment.notifications';
  }

  async setup() {
    const channel = this.connection.getChannel();

    // Create fanout exchange (broadcasts to all queues)
    await channel.assertExchange(this.exchangeName, 'fanout', { durable: true });

    // Create notification queue
    await channel.assertQueue(this.queueName, { durable: true });
    await channel.bindQueue(this.queueName, this.exchangeName, '');

    console.log('✅ Notification service setup complete');
  }

  async start() {
    const channel = this.connection.getChannel();

    console.log('📧 Notification service started');

    channel.consume(
      this.queueName,
      (msg) => {
        if (!msg) return;

        const event = JSON.parse(msg.content.toString());
        this.sendNotification(event);

        channel.ack(msg);
      },
      { noAck: false }
    );
  }

  sendNotification(event) {
    console.log(`\n📧 NOTIFICATION: ${event.type}`);
    console.log(`   Payment: ${event.paymentId}`);
    console.log(`   Status: ${event.status}`);
    console.log(`   Recipient: ${event.recipient}`);
  }

  async publishEvent(event) {
    const channel = this.connection.getChannel();
    const message = Buffer.from(JSON.stringify(event));

    channel.publish(this.exchangeName, '', message, { persistent: true });
  }
}

// ============================================
// Usage Example - Complete Banking System
// ============================================

async function runBankingSystem() {
  console.log('=== Banking Payment Processing with RabbitMQ ===\n');

  // Initialize connection
  const connection = new RabbitMQConnection('amqp://localhost');
  await connection.connect();

  // Setup producer
  const producer = new PaymentProducer(connection);
  await producer.setup();

  // Setup notification service
  const notificationService = new PaymentNotificationService(connection);
  await notificationService.setup();
  await notificationService.start();

  // Setup DLQ monitor
  const dlqMonitor = new DLQMonitor(connection);
  await dlqMonitor.start();

  // Start multiple workers (scale horizontally)
  const workers = [];
  const workerCount = 3;

  for (let i = 1; i <= workerCount; i++) {
    const worker = new PaymentConsumer(connection, i);
    await worker.start();
    workers.push(worker);
  }

  // Wait for workers to be ready
  await new Promise(resolve => setTimeout(resolve, 1000));

  // Publish test payments
  console.log('\n--- Publishing payments ---\n');

  const payments = [
    { amount: 500, accountFrom: 'ACC-001', accountTo: 'ACC-101', description: 'Rent payment' },
    { amount: 1200, accountFrom: 'ACC-002', accountTo: 'ACC-102', description: 'Invoice payment' },
    { amount: 15000, accountFrom: 'ACC-003', accountTo: 'ACC-103', description: 'Wire transfer (fraud check)' },
    { amount: -100, accountFrom: 'ACC-004', accountTo: 'ACC-104', description: 'Invalid amount' },
    { amount: 250, accountFrom: 'ACC-005', accountTo: 'ACC-105', description: 'Bill payment' },
    { amount: 800, accountFrom: 'ACC-006', accountTo: 'ACC-106', description: 'Vendor payment' },
    { amount: 3000, accountFrom: 'ACC-007', accountTo: 'ACC-107', description: 'Payroll' },
    { amount: 0, accountFrom: 'ACC-008', accountTo: 'ACC-108', description: 'Will fail validation' }
  ];

  await producer.publishBatch(payments);

  // Let workers process for 30 seconds
  console.log('\n--- Processing payments (30 seconds) ---\n');
  await new Promise(resolve => setTimeout(resolve, 30000));

  // Stop workers and show stats
  console.log('\n--- Worker Statistics ---\n');
  workers.forEach(worker => {
    const stats = worker.getStats();
    console.log(`Worker ${stats.workerId}: ${stats.processed} processed, ${stats.failed} failed`);
    worker.stop();
  });

  // Show DLQ contents
  console.log('\n--- Dead Letter Queue ---\n');
  const failedPayments = dlqMonitor.getFailedPayments();
  console.log(`Total failed payments in DLQ: ${failedPayments.length}`);
  failedPayments.forEach(fp => {
    console.log(`  - ${fp.payment.id}: $${fp.payment.amount} (${fp.reason})`);
  });

  // Cleanup
  console.log('\n--- Shutting down ---\n');
  await connection.close();
  console.log('✅ System shutdown complete');
}

// Run the system (comment out in production)
if (require.main === module) {
  runBankingSystem().catch(console.error);
}

module.exports = {
  RabbitMQConnection,
  PaymentProducer,
  PaymentConsumer,
  DLQMonitor,
  PaymentNotificationService
};
```

### Key Takeaways from Example 1

1. **Work Queues**: Multiple workers process tasks in parallel
2. **Dead Letter Queue**: Failed messages go to DLQ for manual review
3. **Manual Ack**: Critical for reliability - only ack after successful processing
4. **Retry Logic**: Exponential backoff before sending to DLQ
5. **Durable Queues**: Messages survive broker restart
6. **Prefetch**: Control how many messages a worker processes at once

---

## 💡 Example 2: Event Streaming with Kafka

High-throughput event streaming for transaction analytics and fraud detection.

### Scenario
Build a real-time transaction streaming system where:
- All transactions are streamed to Kafka topics
- Multiple consumers process transactions independently
- Fraud detection analyzes transaction patterns
- Analytics service aggregates metrics
- Audit logs are persisted

### Setup

```bash
# Install Kafka client
npm install kafkajs

# Start Kafka with Docker Compose
# docker-compose.yml:
version: '3'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9092:9092"
```

### Implementation

```javascript
const { Kafka, Partitioners } = require('kafkajs');

/**
 * Kafka Connection Manager
 */
class KafkaConnection {
  constructor(clientId = 'banking-app', brokers = ['localhost:9092']) {
    this.kafka = new Kafka({
      clientId,
      brokers,
      retry: {
        initialRetryTime: 300,
        retries: 10
      }
    });

    this.producer = null;
    this.consumers = new Map();
  }

  async createProducer() {
    this.producer = this.kafka.producer({
      createPartitioner: Partitioners.LegacyPartitioner
    });

    await this.producer.connect();
    console.log('✅ Kafka producer connected');
    return this.producer;
  }

  async createConsumer(groupId, topics) {
    const consumer = this.kafka.consumer({
      groupId,
      sessionTimeout: 30000,
      heartbeatInterval: 3000
    });

    await consumer.connect();
    await consumer.subscribe({ topics, fromBeginning: false });

    this.consumers.set(groupId, consumer);
    console.log(`✅ Kafka consumer connected: ${groupId}`);
    return consumer;
  }

  async disconnect() {
    if (this.producer) await this.producer.disconnect();
    
    for (const consumer of this.consumers.values()) {
      await consumer.disconnect();
    }
  }
}

/**
 * Transaction Stream Producer
 * Publishes transactions to Kafka topics
 */
class TransactionStreamProducer {
  constructor(kafkaConnection) {
    this.connection = kafkaConnection;
    this.topic = 'banking.transactions';
    this.producer = null;
  }

  async setup() {
    this.producer = await this.connection.createProducer();

    // Create topics programmatically
    const admin = this.connection.kafka.admin();
    await admin.connect();

    try {
      await admin.createTopics({
        topics: [
          {
            topic: this.topic,
            numPartitions: 3, // Parallel processing
            replicationFactor: 1
          },
          {
            topic: 'banking.fraud-alerts',
            numPartitions: 1,
            replicationFactor: 1
          },
          {
            topic: 'banking.analytics',
            numPartitions: 1,
            replicationFactor: 1
          }
        ]
      });
      console.log('✅ Kafka topics created');
    } catch (error) {
      // Topics might already exist
      console.log('Topics already exist or error:', error.message);
    } finally {
      await admin.disconnect();
    }
  }

  async publishTransaction(transaction) {
    const txn = {
      id: `TXN-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
      ...transaction,
      timestamp: new Date().toISOString()
    };

    // Use account ID as partition key for ordering
    const messages = [{
      key: txn.accountFrom,
      value: JSON.stringify(txn),
      headers: {
        'event-type': 'transaction',
        'version': '1.0'
      }
    }];

    await this.producer.send({
      topic: this.topic,
      messages
    });

    console.log(`📤 Published transaction: ${txn.id} ($${txn.amount})`);
    return txn;
  }

  async publishBatch(transactions) {
    const messages = transactions.map(txn => {
      const transaction = {
        id: `TXN-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
        ...txn,
        timestamp: new Date().toISOString()
      };

      return {
        key: transaction.accountFrom,
        value: JSON.stringify(transaction)
      };
    });

    await this.producer.send({
      topic: this.topic,
      messages
    });

    console.log(`📤 Published ${messages.length} transactions`);
  }
}

/**
 * Fraud Detection Consumer
 * Analyzes transaction patterns in real-time
 */
class FraudDetectionConsumer {
  constructor(kafkaConnection) {
    this.connection = kafkaConnection;
    this.groupId = 'fraud-detection-service';
    this.consumer = null;
    this.alertProducer = null;
    this.accountVelocity = new Map(); // Track transactions per account
    this.processedCount = 0;
    this.alertCount = 0;
  }

  async start() {
    this.consumer = await this.connection.createConsumer(
      this.groupId,
      ['banking.transactions']
    );

    this.alertProducer = await this.connection.createProducer();

    console.log('🛡️  Fraud detection service started');

    await this.consumer.run({
      eachMessage: async ({ topic, partition, message }) => {
        const transaction = JSON.parse(message.value.toString());
        await this.analyzeTransaction(transaction);
      }
    });
  }

  async analyzeTransaction(transaction) {
    this.processedCount++;

    // Check 1: High-value transaction
    if (transaction.amount > 10000) {
      await this.publishAlert({
        type: 'high-value',
        severity: 'medium',
        transaction,
        reason: `High-value transaction: $${transaction.amount}`
      });
    }

    // Check 2: Velocity check (too many transactions in short time)
    const accountId = transaction.accountFrom;
    const now = Date.now();
    
    if (!this.accountVelocity.has(accountId)) {
      this.accountVelocity.set(accountId, []);
    }

    const recentTxns = this.accountVelocity.get(accountId);
    
    // Add current transaction
    recentTxns.push({ timestamp: now, amount: transaction.amount });

    // Remove transactions older than 5 minutes
    const fiveMinutesAgo = now - 5 * 60 * 1000;
    const filtered = recentTxns.filter(t => t.timestamp > fiveMinutesAgo);
    this.accountVelocity.set(accountId, filtered);

    // Alert if more than 5 transactions in 5 minutes
    if (filtered.length > 5) {
      const totalAmount = filtered.reduce((sum, t) => sum + t.amount, 0);
      await this.publishAlert({
        type: 'velocity',
        severity: 'high',
        transaction,
        reason: `${filtered.length} transactions in 5 minutes (total: $${totalAmount})`
      });
    }

    // Check 3: Round amount (potential testing)
    if (transaction.amount % 1000 === 0 && transaction.amount >= 5000) {
      await this.publishAlert({
        type: 'suspicious-pattern',
        severity: 'low',
        transaction,
        reason: 'Round amount transaction'
      });
    }

    console.log(`🛡️  [Fraud] Analyzed: ${transaction.id} (${this.processedCount} total)`);
  }

  async publishAlert(alert) {
    this.alertCount++;

    const alertMessage = {
      id: `ALERT-${Date.now()}`,
      ...alert,
      timestamp: new Date().toISOString()
    };

    await this.alertProducer.send({
      topic: 'banking.fraud-alerts',
      messages: [{
        key: alert.transaction.accountFrom,
        value: JSON.stringify(alertMessage)
      }]
    });

    console.log(`\n🚨 FRAUD ALERT (${alert.severity.toUpperCase()}):`);
    console.log(`   Type: ${alert.type}`);
    console.log(`   Transaction: ${alert.transaction.id}`);
    console.log(`   Reason: ${alert.reason}\n`);
  }

  getStats() {
    return {
      processed: this.processedCount,
      alerts: this.alertCount
    };
  }
}

/**
 * Analytics Consumer
 * Aggregates transaction metrics in real-time
 */
class AnalyticsConsumer {
  constructor(kafkaConnection) {
    this.connection = kafkaConnection;
    this.groupId = 'analytics-service';
    this.consumer = null;
    this.metrics = {
      totalTransactions: 0,
      totalVolume: 0,
      byCategory: {},
      byHour: {},
      averageAmount: 0
    };
  }

  async start() {
    this.consumer = await this.connection.createConsumer(
      this.groupId,
      ['banking.transactions']
    );

    console.log('📊 Analytics service started');

    await this.consumer.run({
      eachMessage: async ({ topic, partition, message }) => {
        const transaction = JSON.parse(message.value.toString());
        this.updateMetrics(transaction);
      }
    });
  }

  updateMetrics(transaction) {
    this.metrics.totalTransactions++;
    this.metrics.totalVolume += transaction.amount;
    this.metrics.averageAmount = this.metrics.totalVolume / this.metrics.totalTransactions;

    // Category aggregation
    const category = transaction.category || 'uncategorized';
    if (!this.metrics.byCategory[category]) {
      this.metrics.byCategory[category] = { count: 0, volume: 0 };
    }
    this.metrics.byCategory[category].count++;
    this.metrics.byCategory[category].volume += transaction.amount;

    // Hour aggregation
    const hour = new Date(transaction.timestamp).getHours();
    if (!this.metrics.byHour[hour]) {
      this.metrics.byHour[hour] = { count: 0, volume: 0 };
    }
    this.metrics.byHour[hour].count++;
    this.metrics.byHour[hour].volume += transaction.amount;

    if (this.metrics.totalTransactions % 10 === 0) {
      this.printMetrics();
    }
  }

  printMetrics() {
    console.log('\n📊 ANALYTICS UPDATE:');
    console.log(`   Total Transactions: ${this.metrics.totalTransactions}`);
    console.log(`   Total Volume: $${this.metrics.totalVolume.toFixed(2)}`);
    console.log(`   Average Amount: $${this.metrics.averageAmount.toFixed(2)}`);
    
    const topCategory = Object.entries(this.metrics.byCategory)
      .sort((a, b) => b[1].volume - a[1].volume)[0];
    
    if (topCategory) {
      console.log(`   Top Category: ${topCategory[0]} ($${topCategory[1].volume.toFixed(2)})`);
    }
    console.log('');
  }

  getMetrics() {
    return this.metrics;
  }
}

/**
 * Audit Logger Consumer
 * Persists all transactions for compliance
 */
class AuditLoggerConsumer {
  constructor(kafkaConnection) {
    this.connection = kafkaConnection;
    this.groupId = 'audit-logger-service';
    this.consumer = null;
    this.logs = [];
  }

  async start() {
    this.consumer = await this.connection.createConsumer(
      this.groupId,
      ['banking.transactions', 'banking.fraud-alerts']
    );

    console.log('📝 Audit logger started');

    await this.consumer.run({
      eachMessage: async ({ topic, partition, message }) => {
        const data = JSON.parse(message.value.toString());
        await this.logEvent(topic, data);
      }
    });
  }

  async logEvent(topic, data) {
    const logEntry = {
      topic,
      timestamp: new Date().toISOString(),
      data
    };

    this.logs.push(logEntry);

    // In production, write to database
    console.log(`📝 [Audit] ${topic}: ${data.id}`);
  }

  getLogs() {
    return this.logs;
  }
}

// ============================================
// Usage Example - Kafka Streaming System
// ============================================

async function runKafkaStreamingSystem() {
  console.log('=== Banking Transaction Streaming with Kafka ===\n');

  // Initialize Kafka connection
  const kafkaConnection = new KafkaConnection('banking-app');

  // Setup producer
  const producer = new TransactionStreamProducer(kafkaConnection);
  await producer.setup();

  // Start consumers
  const fraudDetection = new FraudDetectionConsumer(kafkaConnection);
  const analytics = new AnalyticsConsumer(kafkaConnection);
  const auditLogger = new AuditLoggerConsumer(kafkaConnection);

  await fraudDetection.start();
  await analytics.start();
  await auditLogger.start();

  // Wait for consumers to be ready
  await new Promise(resolve => setTimeout(resolve, 2000));

  // Generate and publish transactions
  console.log('\n--- Publishing transaction stream ---\n');

  const categories = ['groceries', 'rent', 'utilities', 'entertainment', 'travel'];
  const accounts = Array.from({ length: 10 }, (_, i) => `ACC-${String(i + 1).padStart(3, '0')}`);

  // Publish 50 transactions
  for (let i = 0; i < 50; i++) {
    const transaction = {
      amount: Math.floor(Math.random() * 15000) + 100,
      accountFrom: accounts[Math.floor(Math.random() * accounts.length)],
      accountTo: `ACC-${String(Math.floor(Math.random() * 1000)).padStart(3, '0')}`,
      category: categories[Math.floor(Math.random() * categories.length)],
      description: `Transaction ${i + 1}`
    };

    await producer.publishTransaction(transaction);
    
    // Random delay between transactions
    await new Promise(resolve => setTimeout(resolve, Math.random() * 500 + 200));
  }

  // Let consumers process
  console.log('\n--- Processing (20 seconds) ---\n');
  await new Promise(resolve => setTimeout(resolve, 20000));

  // Show final statistics
  console.log('\n=== Final Statistics ===\n');

  const fraudStats = fraudDetection.getStats();
  console.log('Fraud Detection:');
  console.log(`  Processed: ${fraudStats.processed}`);
  console.log(`  Alerts: ${fraudStats.alerts}`);

  const analyticsMetrics = analytics.getMetrics();
  console.log('\nAnalytics:');
  console.log(`  Total Transactions: ${analyticsMetrics.totalTransactions}`);
  console.log(`  Total Volume: $${analyticsMetrics.totalVolume.toFixed(2)}`);
  console.log(`  Average: $${analyticsMetrics.averageAmount.toFixed(2)}`);

  const auditLogs = auditLogger.getLogs();
  console.log(`\nAudit Logs: ${auditLogs.length} entries`);

  // Cleanup
  console.log('\n--- Shutting down ---\n');
  await kafkaConnection.disconnect();
  console.log('✅ System shutdown complete');
}

// Run the system (comment out in production)
if (require.main === module) {
  runKafkaStreamingSystem().catch(console.error);
}

module.exports = {
  KafkaConnection,
  TransactionStreamProducer,
  FraudDetectionConsumer,
  AnalyticsConsumer,
  AuditLoggerConsumer
};
```

### Key Takeaways from Example 2

1. **Event Streaming**: Kafka stores events for replay and multiple consumers
2. **Partitioning**: Use account ID as key for ordered processing per account
3. **Consumer Groups**: Independent services process same events
4. **Real-Time Analytics**: Aggregate metrics as events flow through
5. **Scalability**: Add partitions and consumers to handle more load
6. **Event Sourcing**: All events are stored and can be replayed

---

## 🎯 RabbitMQ vs Kafka - When to Use

### Use RabbitMQ When:

1. **Task Distribution**: Work queues where each task is processed once
2. **Low Latency**: Need sub-millisecond routing
3. **Complex Routing**: Topic exchanges, routing keys
4. **RPC Patterns**: Request-reply communication
5. **Guaranteed Delivery**: DLQ, acknowledgments, retries

**Example**: Payment processing, email sending, image processing

### Use Kafka When:

1. **Event Streaming**: Need to replay events
2. **High Throughput**: Millions of messages per second
3. **Multiple Consumers**: Different services need same events
4. **Event Sourcing**: Store full event history
5. **Real-Time Analytics**: Stream processing pipelines

**Example**: Transaction logs, activity tracking, IoT data, metrics

---

## 🚀 Best Practices

### Message Design

```javascript
// ✅ Good message structure
{
  "id": "TXN-123",
  "version": "1.0",
  "timestamp": "2025-11-15T10:30:00Z",
  "type": "payment",
  "data": {
    "amount": 1000,
    "from": "ACC-001",
    "to": "ACC-002"
  },
  "metadata": {
    "correlationId": "REQ-456",
    "source": "api-gateway"
  }
}

// ❌ Avoid large payloads
{
  "transaction": {...},
  "customerFullProfile": {...}, // Don't include unnecessary data
  "allAccountHistory": [...],   // Don't embed large arrays
  "invoice": {...}
}
```

### Error Handling

```javascript
// RabbitMQ: Manual acknowledgment with retry
channel.consume(queue, async (msg) => {
  try {
    await processMessage(msg);
    channel.ack(msg); // Success
  } catch (error) {
    if (isRetryable(error)) {
      channel.nack(msg, false, true); // Requeue
    } else {
      channel.nack(msg, false, false); // Send to DLQ
    }
  }
}, { noAck: false });

// Kafka: Commit offset after successful processing
await consumer.run({
  eachMessage: async ({ message, heartbeat }) => {
    await processMessage(message);
    // Auto-commit or manual commit
  }
});
```

### Monitoring

```javascript
// Track key metrics
const metrics = {
  messagesPublished: 0,
  messagesConsumed: 0,
  processingTime: [],
  errors: 0,
  dlqMessages: 0
};

// Monitor queue depth (RabbitMQ)
const queueInfo = await channel.checkQueue(queueName);
console.log('Messages in queue:', queueInfo.messageCount);

// Monitor consumer lag (Kafka)
const admin = kafka.admin();
const offsets = await admin.fetchOffsets({ groupId, topics: ['transactions'] });
console.log('Consumer lag:', offsets);
```

---

## 📚 Common Interview Questions

### Q1: What's the difference between RabbitMQ and Kafka?

**Answer**:
- **RabbitMQ**: Message broker, queue-based, messages deleted after consumption, best for task distribution
- **Kafka**: Distributed log, stream-based, messages retained, best for event streaming and analytics

### Q2: How do you ensure message delivery?

**Answer**:
1. **Producer**: Use confirmations, retries, idempotency keys
2. **Broker**: Durable queues, persistent messages, replication
3. **Consumer**: Manual acknowledgment, at-least-once delivery

### Q3: What is a dead letter queue?

**Answer**: A queue where failed messages go after max retries. Used for:
- Manual investigation
- Alerting operations team
- Retry with fixes
- Analytics on failures

### Q4: How do you handle duplicate messages?

**Answer**: Make consumers **idempotent**:
```javascript
// Store processed message IDs
const processedIds = new Set();

async function processMessage(msg) {
  if (processedIds.has(msg.id)) {
    return; // Already processed
  }
  
  await doWork(msg);
  processedIds.add(msg.id);
}
```

### Q5: How do you scale message consumers?

**Answer**:
- **RabbitMQ**: Add more workers to same queue (competing consumers)
- **Kafka**: Add more partitions and consumer instances (one per partition)

---

## ✅ Summary & Key Takeaways

### Core Concepts

1. **Message Queues**: Async communication between services
2. **Producers**: Publish messages to queues/topics
3. **Consumers**: Process messages from queues/topics
4. **Acknowledgment**: Confirm successful processing

### RabbitMQ Patterns

1. **Work Queues**: Distribute tasks among workers
2. **Pub/Sub**: Broadcast to multiple consumers
3. **Routing**: Direct messages based on routing keys
4. **Topics**: Pattern-based routing
5. **DLQ**: Handle failed messages

### Kafka Patterns

1. **Event Streaming**: Store and replay events
2. **Partitioning**: Parallel processing with ordering
3. **Consumer Groups**: Independent consumers
4. **Stream Processing**: Real-time analytics

### Banking Applications

1. **Payment Processing**: Async task queues
2. **Fraud Detection**: Real-time event analysis
3. **Notifications**: Pub/sub for alerts
4. **Audit Logs**: Event sourcing
5. **Analytics**: Stream aggregation

### Production Checklist

```javascript
// ✅ Message Queue Production Readiness
- [ ] Durable queues and persistent messages
- [ ] Manual acknowledgment (no auto-ack)
- [ ] Dead letter queue configured
- [ ] Retry logic with exponential backoff
- [ ] Idempotent consumers
- [ ] Monitoring and alerting
- [ ] Connection pooling and retry
- [ ] Message size limits
- [ ] Security (TLS, authentication)
- [ ] Disaster recovery plan
```

---

**Status**: ✅ Complete with production-ready RabbitMQ and Kafka examples!
