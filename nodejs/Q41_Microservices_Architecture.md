# Q41: Microservices - Architecture Patterns

## 📋 Summary
Microservices architecture structures applications as collections of loosely coupled, independently deployable services. This guide covers **microservices vs monolith**, service boundaries, communication patterns, API gateway, database per service, and comprehensive banking examples decomposing a monolithic banking system.

---

## 🎯 What You'll Learn
- **Microservices vs Monolith**: Trade-offs, when to use each
- **Service Boundaries**: Domain-driven design, bounded contexts
- **Communication Patterns**: Sync (HTTP/gRPC), async (messaging)
- **API Gateway**: Routing, composition, authentication
- **Database Per Service**: Data isolation, distributed transactions
- **Banking Examples**: Account, Payment, Transaction services

---

## 📖 Comprehensive Explanation

### What are Microservices?

**Microservices** is an architectural style where an application is composed of small, independent services that:
- Run in their own process
- Communicate via lightweight protocols (HTTP/gRPC/messaging)
- Are independently deployable
- Are organized around business capabilities
- Can use different technologies

---

## 🏛️ Monolith vs Microservices

### Monolithic Architecture

**Structure**: Single deployable unit containing all functionality

```
┌─────────────────────────────────────┐
│         Monolithic Application      │
│  ┌──────────┐  ┌──────────┐        │
│  │ Accounts │  │ Payments │        │
│  └──────────┘  └──────────┘        │
│  ┌──────────┐  ┌──────────┐        │
│  │  Loans   │  │ Transfers│        │
│  └──────────┘  └──────────┘        │
│  ┌────────────────────────┐        │
│  │   Shared Database      │        │
│  └────────────────────────┘        │
└─────────────────────────────────────┘
```

**Pros**:
- ✅ Simple to develop initially
- ✅ Easy to test (everything in one place)
- ✅ Simple deployment
- ✅ No network latency between components

**Cons**:
- ❌ Tight coupling (change affects everything)
- ❌ Hard to scale (must scale entire app)
- ❌ Slow deployment (redeploy everything)
- ❌ Technology lock-in (one stack)
- ❌ Large codebase (hard to understand)

### Microservices Architecture

**Structure**: Multiple independent services

```
                  ┌──────────────┐
                  │  API Gateway │
                  └──────┬───────┘
         ┌────────────┬──┴──┬────────────┐
         │            │     │            │
    ┌────▼───┐  ┌────▼───┐ ┌─▼──────┐  ┌▼────────┐
    │Account │  │Payment │ │Transfer│  │  Loan   │
    │Service │  │Service │ │Service │  │ Service │
    └───┬────┘  └───┬────┘ └───┬────┘  └───┬─────┘
        │           │          │            │
    ┌───▼────┐  ┌──▼─────┐ ┌──▼──────┐ ┌──▼──────┐
    │Account │  │Payment │ │Transfer │ │  Loan   │
    │   DB   │  │   DB   │ │   DB    │ │   DB    │
    └────────┘  └────────┘ └─────────┘ └─────────┘
```

**Pros**:
- ✅ Independent deployment
- ✅ Technology diversity
- ✅ Fault isolation (one service fails, others work)
- ✅ Scalability (scale services independently)
- ✅ Team autonomy

**Cons**:
- ❌ Increased complexity
- ❌ Distributed system challenges
- ❌ Network latency
- ❌ Data consistency issues
- ❌ More infrastructure overhead

---

## 🎯 When to Use Each

### Use Monolith When:
- Small team (< 10 developers)
- Simple domain
- Startup/MVP (speed to market)
- Limited scalability needs
- Predictable traffic patterns

### Use Microservices When:
- Large team (> 10 developers)
- Complex domain
- Need independent scaling
- Different services have different tech requirements
- High availability requirements
- Frequent deployments

---

## 🔗 Communication Patterns

### 1. Synchronous (Request-Response)

**HTTP/REST**:
```javascript
// Account service calls Payment service
const axios = require('axios');

const paymentResponse = await axios.post('http://payment-service:3001/payments', {
  fromAccount: 'ACC-123',
  amount: 500
});
```

**Pros**: Simple, immediate response  
**Cons**: Tight coupling, cascading failures

**gRPC** (faster, binary):
```javascript
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');

const packageDefinition = protoLoader.loadSync('payment.proto');
const paymentProto = grpc.loadPackageDefinition(packageDefinition);

const client = new paymentProto.PaymentService(
  'payment-service:50051',
  grpc.credentials.createInsecure()
);

client.createPayment({ fromAccount: 'ACC-123', amount: 500 }, (err, response) => {
  // Handle response
});
```

### 2. Asynchronous (Event-Driven)

**Message Queue** (RabbitMQ, Kafka):
```javascript
const amqp = require('amqplib');

// Publish event
const connection = await amqp.connect('amqp://localhost');
const channel = await connection.createChannel();

const event = {
  type: 'PAYMENT_COMPLETED',
  paymentId: 'PAY-123',
  amount: 500,
  timestamp: new Date()
};

await channel.publish(
  'payments',
  'payment.completed',
  Buffer.from(JSON.stringify(event))
);
```

**Pros**: Loose coupling, fault tolerance, scalability  
**Cons**: Eventual consistency, harder to debug

---

## 📝 Example: Banking System Decomposition

### Original Monolith

```javascript
// Monolithic banking application
// Everything in one service

const express = require('express');
const app = express();

app.post('/transfer', async (req, res) => {
  const { fromAccount, toAccount, amount } = req.body;
  
  // All logic in one place
  const accountValid = await validateAccount(fromAccount);
  const balance = await checkBalance(fromAccount);
  const fraudCheck = await checkFraud(fromAccount, amount);
  const transfer = await executeTransfer(fromAccount, toAccount, amount);
  const notification = await sendNotification(fromAccount, transfer);
  
  res.json({ success: true });
});
```

### Microservices Decomposition

**Services**:
1. **Account Service**: Manage accounts, balances
2. **Payment Service**: Process payments, transfers
3. **Fraud Service**: Detect fraud
4. **Notification Service**: Send alerts
5. **Transaction Service**: Record transactions

---

## 💻 Complete Microservices Implementation

### 1. Account Service

```javascript
// services/account-service/index.js

const express = require('express');
const { Pool } = require('pg');

const app = express();
app.use(express.json());

const pool = new Pool({
  host: process.env.DB_HOST || 'account-db',
  database: 'accounts'
});

// Get account balance
app.get('/accounts/:id/balance', async (req, res) => {
  try {
    const { id } = req.params;
    
    const result = await pool.query(
      'SELECT balance FROM accounts WHERE id = $1',
      [id]
    );
    
    if (result.rows.length === 0) {
      return res.status(404).json({ error: 'Account not found' });
    }
    
    res.json({
      accountId: id,
      balance: parseFloat(result.rows[0].balance)
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Reserve funds (hold balance)
app.post('/accounts/:id/reserve', async (req, res) => {
  const client = await pool.connect();
  
  try {
    const { id } = req.params;
    const { amount, reservationId } = req.body;
    
    await client.query('BEGIN');
    
    // Check balance
    const result = await client.query(
      'SELECT balance FROM accounts WHERE id = $1 FOR UPDATE',
      [id]
    );
    
    if (result.rows.length === 0) {
      throw new Error('Account not found');
    }
    
    const balance = parseFloat(result.rows[0].balance);
    if (balance < amount) {
      throw new Error('Insufficient funds');
    }
    
    // Create reservation
    await client.query(
      'INSERT INTO reservations (id, account_id, amount, status) VALUES ($1, $2, $3, $4)',
      [reservationId, id, amount, 'PENDING']
    );
    
    // Deduct from available balance
    await client.query(
      'UPDATE accounts SET available_balance = available_balance - $1 WHERE id = $2',
      [amount, id]
    );
    
    await client.query('COMMIT');
    
    res.json({
      reservationId,
      status: 'RESERVED'
    });
    
  } catch (error) {
    await client.query('ROLLBACK');
    res.status(400).json({ error: error.message });
  } finally {
    client.release();
  }
});

// Confirm reservation (debit account)
app.post('/accounts/reservations/:id/confirm', async (req, res) => {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    const { id } = req.params;
    
    // Get reservation
    const result = await client.query(
      'SELECT * FROM reservations WHERE id = $1 FOR UPDATE',
      [id]
    );
    
    if (result.rows.length === 0) {
      throw new Error('Reservation not found');
    }
    
    const reservation = result.rows[0];
    
    // Deduct from actual balance
    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
      [reservation.amount, reservation.account_id]
    );
    
    // Mark reservation as confirmed
    await client.query(
      'UPDATE reservations SET status = $1 WHERE id = $2',
      ['CONFIRMED', id]
    );
    
    await client.query('COMMIT');
    
    res.json({ status: 'CONFIRMED' });
    
  } catch (error) {
    await client.query('ROLLBACK');
    res.status(400).json({ error: error.message });
  } finally {
    client.release();
  }
});

const PORT = process.env.PORT || 3001;
app.listen(PORT, () => {
  console.log(`Account service running on port ${PORT}`);
});
```

### 2. Payment Service (Orchestrator)

```javascript
// services/payment-service/index.js

const express = require('express');
const axios = require('axios');
const { v4: uuidv4 } = require('uuid');
const amqp = require('amqplib');

const app = express();
app.use(express.json());

const ACCOUNT_SERVICE = process.env.ACCOUNT_SERVICE_URL || 'http://account-service:3001';
const FRAUD_SERVICE = process.env.FRAUD_SERVICE_URL || 'http://fraud-service:3002';

let rabbitConnection;
let rabbitChannel;

// Initialize RabbitMQ
async function initRabbitMQ() {
  rabbitConnection = await amqp.connect(process.env.RABBITMQ_URL || 'amqp://localhost');
  rabbitChannel = await rabbitConnection.createChannel();
  await rabbitChannel.assertExchange('banking', 'topic', { durable: true });
}

initRabbitMQ();

// Process payment with saga pattern
app.post('/payments', async (req, res) => {
  const { fromAccount, toAccount, amount, description } = req.body;
  const paymentId = `PAY-${uuidv4()}`;
  const reservationId = `RES-${uuidv4()}`;
  
  try {
    // Step 1: Check fraud
    console.log(`[${paymentId}] Checking fraud...`);
    const fraudCheck = await axios.post(`${FRAUD_SERVICE}/check`, {
      accountId: fromAccount,
      amount,
      paymentId
    });
    
    if (fraudCheck.data.riskLevel === 'HIGH') {
      throw new Error('Payment blocked due to fraud risk');
    }
    
    // Step 2: Reserve funds from source account
    console.log(`[${paymentId}] Reserving funds...`);
    await axios.post(`${ACCOUNT_SERVICE}/accounts/${fromAccount}/reserve`, {
      amount,
      reservationId
    });
    
    // Step 3: Credit destination account
    console.log(`[${paymentId}] Crediting destination...`);
    await axios.post(`${ACCOUNT_SERVICE}/accounts/${toAccount}/credit`, {
      amount,
      reference: paymentId
    });
    
    // Step 4: Confirm reservation (debit source)
    console.log(`[${paymentId}] Confirming reservation...`);
    await axios.post(`${ACCOUNT_SERVICE}/accounts/reservations/${reservationId}/confirm`);
    
    // Step 5: Publish payment completed event
    const event = {
      type: 'PAYMENT_COMPLETED',
      paymentId,
      fromAccount,
      toAccount,
      amount,
      timestamp: new Date().toISOString()
    };
    
    rabbitChannel.publish(
      'banking',
      'payment.completed',
      Buffer.from(JSON.stringify(event))
    );
    
    console.log(`[${paymentId}] Payment completed successfully`);
    
    res.status(201).json({
      paymentId,
      status: 'COMPLETED',
      amount,
      fromAccount,
      toAccount
    });
    
  } catch (error) {
    console.error(`[${paymentId}] Payment failed:`, error.message);
    
    // Compensating transactions (rollback)
    try {
      // Cancel reservation if it was created
      await axios.post(`${ACCOUNT_SERVICE}/accounts/reservations/${reservationId}/cancel`);
    } catch (rollbackError) {
      console.error(`[${paymentId}] Rollback failed:`, rollbackError.message);
    }
    
    // Publish payment failed event
    const event = {
      type: 'PAYMENT_FAILED',
      paymentId,
      fromAccount,
      toAccount,
      amount,
      error: error.message,
      timestamp: new Date().toISOString()
    };
    
    rabbitChannel.publish(
      'banking',
      'payment.failed',
      Buffer.from(JSON.stringify(event))
    );
    
    res.status(400).json({
      paymentId,
      status: 'FAILED',
      error: error.message
    });
  }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Payment service running on port ${PORT}`);
});
```

### 3. Fraud Service

```javascript
// services/fraud-service/index.js

const express = require('express');
const app = express();
app.use(express.json());

// Simple fraud detection
app.post('/check', async (req, res) => {
  const { accountId, amount, paymentId } = req.body;
  
  console.log(`[FRAUD] Checking payment ${paymentId}`);
  
  // Simple rules
  let score = 0;
  let riskLevel = 'LOW';
  
  // Rule 1: High amount
  if (amount > 10000) {
    score += 50;
  }
  
  // Rule 2: Very high amount
  if (amount > 50000) {
    score += 30;
    riskLevel = 'HIGH';
  }
  
  // Rule 3: Check velocity (simplified)
  const recentPayments = await getRecentPayments(accountId);
  if (recentPayments.length > 5) {
    score += 20;
  }
  
  if (score > 50) {
    riskLevel = 'MEDIUM';
  }
  if (score > 70) {
    riskLevel = 'HIGH';
  }
  
  res.json({
    paymentId,
    score,
    riskLevel,
    checks: {
      highAmount: amount > 10000,
      velocity: recentPayments.length > 5
    }
  });
});

async function getRecentPayments(accountId) {
  // Simulate database query
  return [];
}

const PORT = process.env.PORT || 3002;
app.listen(PORT, () => {
  console.log(`Fraud service running on port ${PORT}`);
});
```

### 4. Notification Service (Event Consumer)

```javascript
// services/notification-service/index.js

const amqp = require('amqplib');

async function startNotificationService() {
  const connection = await amqp.connect(process.env.RABBITMQ_URL || 'amqp://localhost');
  const channel = await connection.createChannel();
  
  await channel.assertExchange('banking', 'topic', { durable: true });
  const q = await channel.assertQueue('', { exclusive: true });
  
  // Subscribe to payment events
  await channel.bindQueue(q.queue, 'banking', 'payment.*');
  
  console.log('Notification service listening for events...');
  
  channel.consume(q.queue, async (msg) => {
    const event = JSON.parse(msg.content.toString());
    
    console.log(`[NOTIFICATION] Received event: ${event.type}`);
    
    if (event.type === 'PAYMENT_COMPLETED') {
      await sendEmail(event.fromAccount, `Payment of $${event.amount} completed`);
      await sendPushNotification(event.fromAccount, 'Payment successful');
    }
    
    if (event.type === 'PAYMENT_FAILED') {
      await sendEmail(event.fromAccount, `Payment of $${event.amount} failed: ${event.error}`);
    }
    
    channel.ack(msg);
  });
}

async function sendEmail(accountId, message) {
  console.log(`[EMAIL] To: ${accountId}, Message: ${message}`);
}

async function sendPushNotification(accountId, message) {
  console.log(`[PUSH] To: ${accountId}, Message: ${message}`);
}

startNotificationService();
```

### 5. API Gateway

```javascript
// api-gateway/index.js

const express = require('express');
const { createProxyMiddleware } = require('http-proxy-middleware');
const jwt = require('jsonwebtoken');

const app = express();
app.use(express.json());

// Authentication middleware
function authMiddleware(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];
  
  if (!token) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
}

// Route to services
app.use('/api/accounts', authMiddleware, createProxyMiddleware({
  target: 'http://account-service:3001',
  changeOrigin: true,
  pathRewrite: { '^/api/accounts': '/accounts' }
}));

app.use('/api/payments', authMiddleware, createProxyMiddleware({
  target: 'http://payment-service:3000',
  changeOrigin: true,
  pathRewrite: { '^/api/payments': '/payments' }
}));

// Health check
app.get('/health', (req, res) => {
  res.json({ status: 'ok' });
});

const PORT = process.env.PORT || 8080;
app.listen(PORT, () => {
  console.log(`API Gateway running on port ${PORT}`);
});
```

### Docker Compose Setup

```yaml
# docker-compose.yml

version: '3.8'

services:
  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"
    environment:
      - JWT_SECRET=your-secret-key
    depends_on:
      - account-service
      - payment-service

  account-service:
    build: ./services/account-service
    ports:
      - "3001:3001"
    environment:
      - DB_HOST=account-db
      - DB_NAME=accounts
    depends_on:
      - account-db

  payment-service:
    build: ./services/payment-service
    ports:
      - "3000:3000"
    environment:
      - ACCOUNT_SERVICE_URL=http://account-service:3001
      - FRAUD_SERVICE_URL=http://fraud-service:3002
      - RABBITMQ_URL=amqp://rabbitmq
    depends_on:
      - rabbitmq

  fraud-service:
    build: ./services/fraud-service
    ports:
      - "3002:3002"

  notification-service:
    build: ./services/notification-service
    environment:
      - RABBITMQ_URL=amqp://rabbitmq
    depends_on:
      - rabbitmq

  account-db:
    image: postgres:14
    environment:
      - POSTGRES_DB=accounts
      - POSTGRES_PASSWORD=password

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"
```

---

## 🎯 Key Takeaways

### Microservices Best Practices

1. **Single Responsibility**: Each service does one thing well
2. **Database Per Service**: No shared databases
3. **API Gateway**: Single entry point
4. **Asynchronous Communication**: Use events for loose coupling
5. **Circuit Breakers**: Prevent cascading failures
6. **Distributed Tracing**: Track requests across services
7. **Centralized Logging**: Aggregate logs from all services

### Common Patterns

- **Saga Pattern**: Distributed transactions
- **CQRS**: Command Query Responsibility Segregation
- **Event Sourcing**: Store events, not state
- **API Composition**: Combine data from multiple services

### When NOT to Use Microservices

- Small team or simple domain
- Need for strong consistency
- Limited DevOps resources
- Startup/MVP phase

---

**File**: `Q41_Microservices_Architecture.md`  
**Status**: ✅ Complete with microservices patterns, communication, saga pattern, and complete banking system decomposition
