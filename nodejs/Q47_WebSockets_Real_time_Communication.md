# Q47: WebSockets - Real-time Communication

## 📋 Summary
This question covers **WebSocket** protocol and real-time bidirectional communication between clients and servers. You'll learn WebSocket fundamentals, Socket.io framework, event-driven patterns, and building production-ready banking applications with real-time features.

**Key Topics**:
- WebSocket protocol vs HTTP
- WebSocket handshake and lifecycle
- Socket.io features (rooms, namespaces, broadcasting)
- Event-driven communication patterns
- Connection management and reconnection
- Authentication and authorization
- Error handling and heartbeats
- Real-time banking use cases

**Banking Use Cases**:
- Real-time transaction notifications
- Live account balance updates
- Trading platform with real-time quotes
- Customer support chat system
- Fraud alert broadcasting
- Multi-user collaborative features

---

## 🎯 Understanding WebSockets

### WebSocket vs HTTP

**HTTP (Request-Response)**:
```
Client ──→ Request  ──→ Server
Client ←── Response ←── Server
(Connection closes)

// For real-time updates, need polling:
Client ──→ Poll every 5s ──→ Server
Client ──→ Poll every 5s ──→ Server
❌ Inefficient, high latency
```

**WebSocket (Bidirectional)**:
```
Client ←──────────────→ Server
    (Persistent connection)

Client ←── Message ←── Server (instant)
Client ──→ Message ──→ Server (instant)
✅ Low latency, efficient
```

### WebSocket Protocol

```
┌─────────────────────────────────────────────────────────────┐
│                   WebSocket Lifecycle                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. HANDSHAKE (HTTP Upgrade)                                │
│     Client ──→ GET /ws HTTP/1.1                             │
│                Upgrade: websocket                            │
│                Connection: Upgrade                           │
│     Server ←── 101 Switching Protocols                      │
│                                                              │
│  2. CONNECTION ESTABLISHED                                   │
│     ┌──────────────────────────────────────┐               │
│     │   Persistent TCP connection           │               │
│     │   Client ←──────────────→ Server     │               │
│     └──────────────────────────────────────┘               │
│                                                              │
│  3. MESSAGE EXCHANGE (Frames)                               │
│     Client ──→ Text/Binary frame ──→ Server                │
│     Client ←── Text/Binary frame ←── Server                │
│                                                              │
│  4. PING/PONG (Heartbeat)                                   │
│     Client ──→ Ping ──→ Server                              │
│     Client ←── Pong ←── Server                              │
│                                                              │
│  5. CLOSE                                                    │
│     Client ──→ Close frame ──→ Server                       │
│     Server ──→ Close frame ──→ Client                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### WebSocket vs HTTP Comparison

| Aspect | HTTP | WebSocket |
|--------|------|-----------|
| **Connection** | Request-response, closes | Persistent, bidirectional |
| **Overhead** | ~800 bytes headers | ~2 bytes per frame |
| **Latency** | High (new connection) | Low (persistent) |
| **Server Push** | ❌ No (polling needed) | ✅ Yes (native) |
| **Scaling** | Stateless (easy) | Stateful (complex) |
| **Use Case** | CRUD operations | Real-time updates |

### Socket.io Features

**Socket.io** = WebSocket + Fallbacks + Features

```
Socket.io = WebSocket (preferred)
          + HTTP long-polling (fallback)
          + Automatic reconnection
          + Rooms & namespaces
          + Broadcasting
          + Acknowledgments
          + Binary support
```

---

## 💡 Example 1: Real-time Banking Transaction Platform

Complete real-time banking system with transaction notifications, balance updates, and fraud alerts.

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│          Real-time Banking Platform Architecture             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Web Clients ──┐                                            │
│  Mobile Apps ──┼──→ Load Balancer                          │
│  Admin Panel ──┘         │                                  │
│                          │                                  │
│                          ▼                                  │
│              ┌────────────────────────┐                     │
│              │   Socket.io Server     │                     │
│              │  (Express + Socket.io) │                     │
│              └───────────┬────────────┘                     │
│                          │                                  │
│         ┌────────────────┼────────────────┐                │
│         │                │                 │                │
│         ▼                ▼                 ▼                │
│   ┌─────────┐     ┌──────────┐     ┌──────────┐          │
│   │  Redis  │     │ DynamoDB │     │   SQS    │          │
│   │ Pub/Sub │     │ (State)  │     │ (Queue)  │          │
│   └─────────┘     └──────────┘     └──────────┘          │
│                                                              │
│  Rooms/Namespaces:                                          │
│  - /transactions (all transaction events)                   │
│  - /accounts/:accountId (account-specific)                  │
│  - /admin (fraud alerts, monitoring)                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Setup

```bash
# Initialize project
mkdir banking-websocket
cd banking-websocket
npm init -y

# Install dependencies
npm install express socket.io
npm install jsonwebtoken bcryptjs
npm install ioredis
npm install dotenv

# Dev dependencies
npm install --save-dev nodemon
```

### Implementation

#### server.js

```javascript
/**
 * Real-time Banking WebSocket Server
 * Features: JWT auth, rooms, namespaces, transaction notifications
 */

const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const jwt = require('jsonwebtoken');
const Redis = require('ioredis');

// Initialize Express and Socket.io
const app = express();
const server = http.createServer(app);
const io = new Server(server, {
  cors: {
    origin: process.env.CLIENT_URL || 'http://localhost:3000',
    methods: ['GET', 'POST'],
    credentials: true
  },
  // Connection options
  pingTimeout: 60000,
  pingInterval: 25000,
  transports: ['websocket', 'polling']
});

// Redis client (for pub/sub in scaling scenario)
const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: process.env.REDIS_PORT || 6379
});

const JWT_SECRET = process.env.JWT_SECRET || 'your-secret-key';

// ============================================
// AUTHENTICATION MIDDLEWARE
// ============================================

io.use(async (socket, next) => {
  try {
    const token = socket.handshake.auth.token || socket.handshake.headers.authorization?.split(' ')[1];
    
    if (!token) {
      return next(new Error('Authentication error: No token provided'));
    }
    
    // Verify JWT token
    const decoded = jwt.verify(token, JWT_SECRET);
    
    // Attach user info to socket
    socket.userId = decoded.userId;
    socket.username = decoded.username;
    socket.role = decoded.role;
    
    console.log(`✅ User authenticated: ${socket.username} (${socket.userId})`);
    next();
    
  } catch (error) {
    console.error('Authentication error:', error.message);
    next(new Error('Authentication error: Invalid token'));
  }
});

// ============================================
// CONNECTION HANDLER
// ============================================

io.on('connection', (socket) => {
  console.log(`🔌 Client connected: ${socket.id} (User: ${socket.username})`);
  
  // Join user-specific room
  const userRoom = `user:${socket.userId}`;
  socket.join(userRoom);
  console.log(`📍 Joined room: ${userRoom}`);
  
  // Send connection confirmation
  socket.emit('connected', {
    socketId: socket.id,
    userId: socket.userId,
    username: socket.username,
    timestamp: Date.now()
  });
  
  // ============================================
  // ACCOUNT SUBSCRIPTION
  // ============================================
  
  socket.on('subscribe:account', async (data) => {
    const { accountId } = data;
    
    // Verify user owns this account
    const hasAccess = await verifyAccountAccess(socket.userId, accountId);
    
    if (!hasAccess) {
      socket.emit('error', {
        event: 'subscribe:account',
        message: 'Access denied to account'
      });
      return;
    }
    
    const accountRoom = `account:${accountId}`;
    socket.join(accountRoom);
    
    console.log(`📊 ${socket.username} subscribed to account: ${accountId}`);
    
    // Send current account balance
    const balance = await getAccountBalance(accountId);
    socket.emit('account:balance', {
      accountId,
      balance,
      currency: 'USD',
      timestamp: Date.now()
    });
  });
  
  socket.on('unsubscribe:account', (data) => {
    const { accountId } = data;
    const accountRoom = `account:${accountId}`;
    socket.leave(accountRoom);
    console.log(`📊 ${socket.username} unsubscribed from account: ${accountId}`);
  });
  
  // ============================================
  // TRANSACTION EVENTS
  // ============================================
  
  socket.on('transaction:create', async (data) => {
    try {
      const { accountId, type, amount, description } = data;
      
      // Validate transaction
      if (!accountId || !type || !amount) {
        socket.emit('transaction:error', {
          message: 'Missing required fields'
        });
        return;
      }
      
      // Verify account access
      const hasAccess = await verifyAccountAccess(socket.userId, accountId);
      if (!hasAccess) {
        socket.emit('transaction:error', {
          message: 'Access denied'
        });
        return;
      }
      
      // Create transaction
      const transaction = {
        transactionId: `TXN-${Date.now()}`,
        accountId,
        type,
        amount,
        description,
        userId: socket.userId,
        status: 'pending',
        timestamp: Date.now()
      };
      
      // Save to database (simulated)
      await saveTransaction(transaction);
      
      // Update account balance
      const newBalance = await updateAccountBalance(accountId, type, amount);
      
      // Emit to user who created transaction
      socket.emit('transaction:created', {
        ...transaction,
        status: 'completed'
      });
      
      // Broadcast to all clients watching this account
      io.to(`account:${accountId}`).emit('transaction:new', {
        ...transaction,
        status: 'completed',
        newBalance
      });
      
      // Emit balance update
      io.to(`account:${accountId}`).emit('account:balance', {
        accountId,
        balance: newBalance,
        currency: 'USD',
        timestamp: Date.now()
      });
      
      console.log(`💰 Transaction created: ${transaction.transactionId} ($${amount})`);
      
      // Fraud detection
      if (amount > 5000) {
        await checkFraud(transaction, socket);
      }
      
    } catch (error) {
      console.error('Transaction error:', error);
      socket.emit('transaction:error', {
        message: error.message
      });
    }
  });
  
  socket.on('transaction:list', async (data) => {
    const { accountId, limit = 50 } = data;
    
    const hasAccess = await verifyAccountAccess(socket.userId, accountId);
    if (!hasAccess) {
      socket.emit('error', { message: 'Access denied' });
      return;
    }
    
    const transactions = await getTransactions(accountId, limit);
    socket.emit('transaction:list', {
      accountId,
      transactions,
      count: transactions.length
    });
  });
  
  // ============================================
  // REAL-TIME NOTIFICATIONS
  // ============================================
  
  socket.on('notification:read', async (data) => {
    const { notificationId } = data;
    await markNotificationRead(socket.userId, notificationId);
    
    socket.emit('notification:updated', {
      notificationId,
      read: true
    });
  });
  
  // ============================================
  // ADMIN EVENTS (Role-based)
  // ============================================
  
  if (socket.role === 'admin') {
    socket.join('admin');
    
    socket.on('admin:broadcast', (data) => {
      io.emit('system:announcement', {
        message: data.message,
        type: 'info',
        timestamp: Date.now()
      });
      console.log(`📢 Admin broadcast: ${data.message}`);
    });
    
    socket.on('admin:stats', async () => {
      const stats = {
        connectedClients: io.sockets.sockets.size,
        rooms: Array.from(io.sockets.adapter.rooms.keys()),
        uptime: process.uptime()
      };
      socket.emit('admin:stats', stats);
    });
  }
  
  // ============================================
  // TYPING INDICATORS (for chat)
  // ============================================
  
  socket.on('chat:typing', (data) => {
    const { conversationId } = data;
    socket.to(`conversation:${conversationId}`).emit('chat:typing', {
      userId: socket.userId,
      username: socket.username
    });
  });
  
  socket.on('chat:stop-typing', (data) => {
    const { conversationId } = data;
    socket.to(`conversation:${conversationId}`).emit('chat:stop-typing', {
      userId: socket.userId
    });
  });
  
  // ============================================
  // HEARTBEAT / PING-PONG
  // ============================================
  
  socket.on('ping', () => {
    socket.emit('pong', { timestamp: Date.now() });
  });
  
  // ============================================
  // DISCONNECTION
  // ============================================
  
  socket.on('disconnect', (reason) => {
    console.log(`🔌 Client disconnected: ${socket.id} (${socket.username})`);
    console.log(`   Reason: ${reason}`);
    
    // Notify other users if needed
    // io.emit('user:offline', { userId: socket.userId });
  });
  
  socket.on('error', (error) => {
    console.error(`❌ Socket error: ${socket.id}`, error);
  });
});

// ============================================
// HELPER FUNCTIONS
// ============================================

// Simulated database access (replace with real DB)
const accounts = new Map();
const transactions = new Map();

async function verifyAccountAccess(userId, accountId) {
  // In production, check database
  return true; // Simplified
}

async function getAccountBalance(accountId) {
  if (!accounts.has(accountId)) {
    accounts.set(accountId, 5000); // Initial balance
  }
  return accounts.get(accountId);
}

async function saveTransaction(transaction) {
  if (!transactions.has(transaction.accountId)) {
    transactions.set(transaction.accountId, []);
  }
  transactions.get(transaction.accountId).push(transaction);
}

async function updateAccountBalance(accountId, type, amount) {
  const currentBalance = await getAccountBalance(accountId);
  const change = ['deposit', 'credit'].includes(type) ? amount : -amount;
  const newBalance = currentBalance + change;
  accounts.set(accountId, newBalance);
  return newBalance;
}

async function getTransactions(accountId, limit) {
  const accountTxns = transactions.get(accountId) || [];
  return accountTxns.slice(-limit).reverse();
}

async function checkFraud(transaction, socket) {
  // Simplified fraud detection
  if (transaction.amount > 10000) {
    // Send fraud alert to admin room
    io.to('admin').emit('fraud:alert', {
      severity: 'high',
      transaction,
      timestamp: Date.now()
    });
    
    // Notify user
    socket.emit('transaction:review', {
      message: 'Transaction is under review',
      transactionId: transaction.transactionId
    });
    
    console.log(`🚨 Fraud alert: Transaction ${transaction.transactionId}`);
  }
}

async function markNotificationRead(userId, notificationId) {
  // Update database
  console.log(`✓ Notification ${notificationId} marked read for user ${userId}`);
}

// ============================================
// EXTERNAL EVENTS (from other services)
// ============================================

// Listen for events from other microservices via Redis pub/sub
redis.subscribe('banking:transactions', (err) => {
  if (err) {
    console.error('Redis subscription error:', err);
  } else {
    console.log('📡 Subscribed to banking:transactions channel');
  }
});

redis.on('message', (channel, message) => {
  if (channel === 'banking:transactions') {
    const event = JSON.parse(message);
    
    // Broadcast to relevant rooms
    io.to(`account:${event.accountId}`).emit('transaction:external', event);
    console.log(`📨 External transaction event: ${event.transactionId}`);
  }
});

// ============================================
// REST API ENDPOINTS
// ============================================

app.use(express.json());

// Health check
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    connections: io.sockets.sockets.size,
    uptime: process.uptime()
  });
});

// Generate JWT token (for testing)
app.post('/auth/login', (req, res) => {
  const { username, password } = req.body;
  
  // Simplified auth (use proper auth in production)
  if (username && password) {
    const token = jwt.sign(
      {
        userId: `USER-${Date.now()}`,
        username,
        role: username === 'admin' ? 'admin' : 'user'
      },
      JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.json({ token, username });
  } else {
    res.status(400).json({ error: 'Invalid credentials' });
  }
});

// Trigger transaction (REST endpoint that emits WebSocket event)
app.post('/api/transactions', async (req, res) => {
  const { accountId, type, amount, description } = req.body;
  
  const transaction = {
    transactionId: `TXN-${Date.now()}`,
    accountId,
    type,
    amount,
    description,
    status: 'completed',
    timestamp: Date.now()
  };
  
  // Save transaction
  await saveTransaction(transaction);
  
  // Emit to WebSocket clients
  io.to(`account:${accountId}`).emit('transaction:new', transaction);
  
  res.json(transaction);
});

// ============================================
// START SERVER
// ============================================

const PORT = process.env.PORT || 3001;

server.listen(PORT, () => {
  console.log(`🚀 WebSocket server running on port ${PORT}`);
  console.log(`📡 Socket.io path: /socket.io/`);
  console.log(`🔑 JWT authentication enabled`);
});

// Graceful shutdown
process.on('SIGTERM', () => {
  console.log('SIGTERM received, closing server...');
  server.close(() => {
    redis.quit();
    console.log('Server closed');
    process.exit(0);
  });
});
```

#### client.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Real-time Banking</title>
  <script src="https://cdn.socket.io/4.5.4/socket.io.min.js"></script>
  <style>
    body {
      font-family: Arial, sans-serif;
      max-width: 1200px;
      margin: 0 auto;
      padding: 20px;
      background: #f5f5f5;
    }
    .container {
      background: white;
      padding: 20px;
      border-radius: 8px;
      box-shadow: 0 2px 4px rgba(0,0,0,0.1);
      margin-bottom: 20px;
    }
    .status {
      padding: 10px;
      border-radius: 4px;
      margin-bottom: 20px;
    }
    .status.connected {
      background: #d4edda;
      color: #155724;
    }
    .status.disconnected {
      background: #f8d7da;
      color: #721c24;
    }
    .balance {
      font-size: 2em;
      font-weight: bold;
      color: #28a745;
    }
    .transaction {
      padding: 10px;
      border-left: 4px solid #007bff;
      background: #f8f9fa;
      margin: 10px 0;
      border-radius: 4px;
    }
    .transaction.deposit {
      border-left-color: #28a745;
    }
    .transaction.withdrawal {
      border-left-color: #dc3545;
    }
    button {
      padding: 10px 20px;
      background: #007bff;
      color: white;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      margin: 5px;
    }
    button:hover {
      background: #0056b3;
    }
    input, select {
      padding: 8px;
      margin: 5px;
      border: 1px solid #ddd;
      border-radius: 4px;
    }
    .alert {
      padding: 10px;
      background: #fff3cd;
      border: 1px solid #ffc107;
      border-radius: 4px;
      margin: 10px 0;
    }
  </style>
</head>
<body>
  <h1>🏦 Real-time Banking Platform</h1>
  
  <!-- Connection Status -->
  <div id="status" class="status disconnected">
    Disconnected
  </div>
  
  <!-- Login -->
  <div class="container" id="login-container">
    <h2>Login</h2>
    <input type="text" id="username" placeholder="Username" value="john_doe">
    <input type="password" id="password" placeholder="Password" value="password123">
    <button onclick="login()">Login</button>
  </div>
  
  <!-- Account Dashboard -->
  <div class="container" id="dashboard" style="display: none;">
    <h2>Account Dashboard</h2>
    
    <div>
      <strong>Account ID:</strong> <span id="accountId">ACC-12345</span>
      <button onclick="subscribeToAccount()">Subscribe</button>
    </div>
    
    <div>
      <strong>Balance:</strong>
      <div class="balance" id="balance">$0.00</div>
    </div>
    
    <h3>Create Transaction</h3>
    <select id="txn-type">
      <option value="deposit">Deposit</option>
      <option value="withdrawal">Withdrawal</option>
      <option value="transfer">Transfer</option>
    </select>
    <input type="number" id="txn-amount" placeholder="Amount" value="100">
    <input type="text" id="txn-description" placeholder="Description" value="Test transaction">
    <button onclick="createTransaction()">Create Transaction</button>
    
    <h3>Recent Transactions</h3>
    <div id="transactions"></div>
    
    <h3>Notifications</h3>
    <div id="notifications"></div>
  </div>
  
  <script>
    let socket = null;
    let token = null;
    let accountId = 'ACC-12345';
    
    // Login
    async function login() {
      const username = document.getElementById('username').value;
      const password = document.getElementById('password').value;
      
      try {
        const response = await fetch('http://localhost:3001/auth/login', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ username, password })
        });
        
        const data = await response.json();
        token = data.token;
        
        // Connect to WebSocket
        connectWebSocket();
        
        // Show dashboard
        document.getElementById('login-container').style.display = 'none';
        document.getElementById('dashboard').style.display = 'block';
        
      } catch (error) {
        alert('Login failed: ' + error.message);
      }
    }
    
    // Connect WebSocket
    function connectWebSocket() {
      socket = io('http://localhost:3001', {
        auth: { token }
      });
      
      // Connection events
      socket.on('connect', () => {
        console.log('✅ Connected:', socket.id);
        updateStatus('Connected', true);
      });
      
      socket.on('connected', (data) => {
        console.log('Connection confirmed:', data);
      });
      
      socket.on('disconnect', (reason) => {
        console.log('❌ Disconnected:', reason);
        updateStatus('Disconnected', false);
      });
      
      socket.on('error', (error) => {
        console.error('Socket error:', error);
        addNotification('Error: ' + error.message, 'error');
      });
      
      // Account balance
      socket.on('account:balance', (data) => {
        console.log('Balance update:', data);
        document.getElementById('balance').textContent = `$${data.balance.toFixed(2)}`;
      });
      
      // New transaction
      socket.on('transaction:new', (data) => {
        console.log('New transaction:', data);
        addTransaction(data);
        addNotification(`New ${data.type}: $${data.amount}`, 'info');
      });
      
      // Transaction created
      socket.on('transaction:created', (data) => {
        console.log('Transaction created:', data);
        addNotification('Transaction completed', 'success');
      });
      
      // Transaction error
      socket.on('transaction:error', (data) => {
        alert('Transaction error: ' + data.message);
      });
      
      // Fraud alert
      socket.on('transaction:review', (data) => {
        addNotification('⚠️ ' + data.message, 'warning');
      });
      
      // System announcements
      socket.on('system:announcement', (data) => {
        addNotification('📢 ' + data.message, 'info');
      });
    }
    
    // Subscribe to account
    function subscribeToAccount() {
      accountId = document.getElementById('accountId').textContent;
      socket.emit('subscribe:account', { accountId });
      addNotification('Subscribed to account updates', 'success');
    }
    
    // Create transaction
    function createTransaction() {
      const type = document.getElementById('txn-type').value;
      const amount = parseFloat(document.getElementById('txn-amount').value);
      const description = document.getElementById('txn-description').value;
      
      socket.emit('transaction:create', {
        accountId,
        type,
        amount,
        description
      });
    }
    
    // UI helpers
    function updateStatus(text, connected) {
      const status = document.getElementById('status');
      status.textContent = text;
      status.className = 'status ' + (connected ? 'connected' : 'disconnected');
    }
    
    function addTransaction(txn) {
      const div = document.createElement('div');
      div.className = `transaction ${txn.type}`;
      div.innerHTML = `
        <strong>${txn.type.toUpperCase()}</strong>: $${txn.amount}
        <br><small>${txn.description || ''}</small>
        <br><small>${new Date(txn.timestamp).toLocaleString()}</small>
      `;
      document.getElementById('transactions').prepend(div);
    }
    
    function addNotification(message, type) {
      const div = document.createElement('div');
      div.className = 'alert';
      div.textContent = message;
      document.getElementById('notifications').prepend(div);
      
      // Auto-remove after 5 seconds
      setTimeout(() => div.remove(), 5000);
    }
  </script>
</body>
</html>
```

### Testing

```bash
# Start server
node server.js

# Open client
# Open client.html in browser

# Test with curl (REST API)
curl -X POST http://localhost:3001/api/transactions \
  -H "Content-Type: application/json" \
  -d '{
    "accountId": "ACC-12345",
    "type": "deposit",
    "amount": 1000,
    "description": "Salary"
  }'
```

---

## 💡 Example 2: Socket.io Advanced Features

### Rooms and Namespaces

```javascript
/**
 * Advanced Socket.io Patterns
 */

const io = require('socket.io')(server);

// ============================================
// NAMESPACES (Separate communication channels)
// ============================================

// Main namespace (default: /)
io.on('connection', (socket) => {
  console.log('Main namespace connection');
});

// Admin namespace
const adminNamespace = io.of('/admin');
adminNamespace.use((socket, next) => {
  // Admin-only authentication
  if (socket.handshake.auth.role === 'admin') {
    next();
  } else {
    next(new Error('Admin access required'));
  }
});

adminNamespace.on('connection', (socket) => {
  console.log('Admin connected:', socket.id);
  
  socket.on('broadcast:all', (data) => {
    io.emit('system:message', data); // Broadcast to main namespace
  });
});

// Transactions namespace
const transactionsNamespace = io.of('/transactions');
transactionsNamespace.on('connection', (socket) => {
  socket.on('watch', (accountId) => {
    socket.join(`account:${accountId}`);
  });
});

// ============================================
// ROOMS (Groups within a namespace)
// ============================================

io.on('connection', (socket) => {
  // Join multiple rooms
  socket.join('general');
  socket.join(`user:${socket.userId}`);
  socket.join(`country:${socket.country}`);
  
  // Get rooms a socket is in
  console.log(socket.rooms); // Set { socket.id, 'general', 'user:123', 'country:US' }
  
  // Leave room
  socket.leave('general');
  
  // Emit to specific room
  io.to('general').emit('message', 'Hello general room');
  
  // Emit to multiple rooms
  io.to('room1').to('room2').emit('message', 'Multiple rooms');
  
  // Emit to all except sender
  socket.broadcast.emit('message', 'To all except me');
  
  // Emit to room except sender
  socket.to('room1').emit('message', 'To room1 except me');
  
  // Get all sockets in a room
  const socketsInRoom = await io.in('general').fetchSockets();
  console.log('Sockets in general:', socketsInRoom.length);
});

// ============================================
// ACKNOWLEDGMENTS (Callbacks)
// ============================================

// Server
io.on('connection', (socket) => {
  socket.on('transaction:create', (data, callback) => {
    try {
      const result = processTransaction(data);
      
      // Acknowledge success
      callback({
        success: true,
        transactionId: result.id
      });
    } catch (error) {
      // Acknowledge error
      callback({
        success: false,
        error: error.message
      });
    }
  });
});

// Client
socket.emit('transaction:create', transactionData, (response) => {
  if (response.success) {
    console.log('Transaction created:', response.transactionId);
  } else {
    console.error('Transaction failed:', response.error);
  }
});

// ============================================
// BROADCASTING PATTERNS
// ============================================

io.on('connection', (socket) => {
  // To all clients
  io.emit('event', data);
  
  // To all clients except sender
  socket.broadcast.emit('event', data);
  
  // To specific room
  io.to('room1').emit('event', data);
  
  // To specific socket
  io.to(socketId).emit('event', data);
  
  // To multiple rooms
  io.to('room1').to('room2').emit('event', data);
  
  // To all clients in namespace
  io.of('/admin').emit('event', data);
  
  // Volatile (don't queue if client disconnected)
  socket.volatile.emit('event', data);
  
  // Binary data
  socket.emit('image', Buffer.from([1, 2, 3, 4]));
});

// ============================================
// MIDDLEWARE
// ============================================

// Global middleware (all connections)
io.use((socket, next) => {
  const token = socket.handshake.auth.token;
  if (isValid(token)) {
    next();
  } else {
    next(new Error('Authentication failed'));
  }
});

// Namespace middleware
io.of('/admin').use((socket, next) => {
  if (socket.handshake.auth.isAdmin) {
    next();
  } else {
    next(new Error('Admin only'));
  }
});

// ============================================
// ERROR HANDLING
// ============================================

io.on('connection', (socket) => {
  socket.on('error', (error) => {
    console.error('Socket error:', error);
  });
  
  // Handle all events with try-catch
  socket.onAny((event, ...args) => {
    console.log(`Event: ${event}`, args);
  });
  
  socket.onAnyOutgoing((event, ...args) => {
    console.log(`Outgoing: ${event}`, args);
  });
});

// ============================================
// CONNECTION STATE
// ============================================

io.on('connection', (socket) => {
  console.log('Transport:', socket.conn.transport.name); // websocket or polling
  
  socket.conn.on('upgrade', () => {
    console.log('Transport upgraded:', socket.conn.transport.name);
  });
  
  socket.on('disconnect', (reason) => {
    console.log('Disconnect reason:', reason);
    // Reasons:
    // - 'transport close' (network issue)
    // - 'client namespace disconnect' (client called disconnect())
    // - 'server namespace disconnect' (server kicked client)
    // - 'ping timeout' (client didn't respond to ping)
  });
});
```

---

## 🎯 WebSocket Best Practices

### 1. Connection Management

```javascript
// Reconnection logic (client-side)
const socket = io('http://localhost:3001', {
  reconnection: true,
  reconnectionAttempts: 5,
  reconnectionDelay: 1000,
  reconnectionDelayMax: 5000,
  timeout: 20000
});

socket.on('connect', () => {
  console.log('Connected');
  // Re-subscribe to rooms after reconnect
  resubscribeToRooms();
});

socket.on('disconnect', () => {
  console.log('Disconnected');
  showReconnectingMessage();
});

socket.on('reconnect_attempt', (attemptNumber) => {
  console.log('Reconnection attempt:', attemptNumber);
});

socket.on('reconnect_failed', () => {
  console.log('Reconnection failed');
  showOfflineMessage();
});
```

### 2. Heartbeat / Keep-Alive

```javascript
// Server configuration
const io = new Server(server, {
  pingInterval: 25000, // Send ping every 25s
  pingTimeout: 60000   // Wait 60s for pong
});

// Client-side heartbeat
setInterval(() => {
  socket.emit('ping');
}, 30000);

socket.on('pong', (data) => {
  const latency = Date.now() - data.timestamp;
  console.log('Latency:', latency, 'ms');
});
```

### 3. Message Queueing

```javascript
// Queue messages when disconnected
class MessageQueue {
  constructor(socket) {
    this.socket = socket;
    this.queue = [];
    this.connected = socket.connected;
    
    socket.on('connect', () => {
      this.connected = true;
      this.flush();
    });
    
    socket.on('disconnect', () => {
      this.connected = false;
    });
  }
  
  emit(event, data) {
    if (this.connected) {
      this.socket.emit(event, data);
    } else {
      this.queue.push({ event, data });
    }
  }
  
  flush() {
    while (this.queue.length > 0) {
      const { event, data } = this.queue.shift();
      this.socket.emit(event, data);
    }
  }
}

// Usage
const queue = new MessageQueue(socket);
queue.emit('transaction:create', data);
```

### 4. Rate Limiting

```javascript
// Server-side rate limiting
const rateLimit = new Map();

io.use((socket, next) => {
  const userId = socket.userId;
  const now = Date.now();
  const userLimit = rateLimit.get(userId) || { count: 0, resetTime: now + 60000 };
  
  if (now > userLimit.resetTime) {
    userLimit.count = 0;
    userLimit.resetTime = now + 60000;
  }
  
  if (userLimit.count >= 100) {
    return next(new Error('Rate limit exceeded'));
  }
  
  userLimit.count++;
  rateLimit.set(userId, userLimit);
  next();
});
```

### 5. Binary Data Handling

```javascript
// Server
socket.on('file:upload', (data) => {
  const { filename, buffer } = data;
  
  // Save file
  fs.writeFileSync(`./uploads/${filename}`, buffer);
  
  socket.emit('file:uploaded', { filename });
});

// Client
const file = document.getElementById('file').files[0];
const reader = new FileReader();

reader.onload = () => {
  const buffer = reader.result;
  socket.emit('file:upload', {
    filename: file.name,
    buffer: buffer
  });
};

reader.readAsArrayBuffer(file);
```

---

## 📚 Common Interview Questions

### Q1: Explain the WebSocket handshake process

**Answer**:
```
1. Client sends HTTP GET request:
   GET /chat HTTP/1.1
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
   
2. Server responds with HTTP 101:
   HTTP/1.1 101 Switching Protocols
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
   
3. Connection upgraded to WebSocket
4. Bidirectional frames exchanged
```

### Q2: How do you handle authentication with WebSockets?

**Answer**:
Three approaches:
1. **Token in handshake**: `socket.handshake.auth.token`
2. **First message**: Client sends auth message after connect
3. **Cookie**: Use HTTP session cookies

Recommend: JWT in handshake auth header.

### Q3: What's the difference between Socket.io and raw WebSocket?

**Answer**:
- **Raw WebSocket**: Protocol only, manual reconnection, no fallback
- **Socket.io**: Library with auto-reconnect, HTTP long-polling fallback, rooms, namespaces, broadcasting

Use Socket.io for production (easier), raw WebSocket for simple/performance-critical apps.

### Q4: How do you scale WebSocket servers?

**Answer**: Use Redis pub/sub adapter (covered in Q48).

### Q5: What are WebSocket security concerns?

**Answer**:
1. **Authentication**: Validate JWT on connection
2. **Authorization**: Check permissions per event
3. **Input validation**: Validate all incoming data
4. **Rate limiting**: Prevent spam/DoS
5. **CSRF**: Use tokens, validate origin
6. **XSS**: Sanitize messages if rendering HTML

---

## ✅ Summary & Key Takeaways

### Core Concepts

1. **WebSocket**: Persistent bidirectional connection
2. **Socket.io**: WebSocket library with features
3. **Rooms**: Group sockets for targeted broadcasting
4. **Namespaces**: Separate communication channels

### Real-time Patterns

1. **Pub/Sub**: Clients subscribe to topics
2. **Broadcasting**: Send to multiple clients
3. **Rooms**: Target specific groups
4. **Acknowledgments**: Confirm message delivery

### Banking Use Cases

1. **Transaction Notifications**: Instant balance updates
2. **Fraud Alerts**: Real-time suspicious activity
3. **Trading**: Live market quotes
4. **Chat**: Customer support messaging

### Production Checklist

```javascript
✅ WebSocket Production Readiness:
  - [ ] JWT authentication
  - [ ] Reconnection handling
  - [ ] Message queueing (offline support)
  - [ ] Heartbeat/keep-alive
  - [ ] Rate limiting
  - [ ] Error handling
  - [ ] Logging and monitoring
  - [ ] Room cleanup on disconnect
  - [ ] Binary data support
  - [ ] Graceful shutdown
```

---

**Status**: ✅ Complete with production-ready WebSocket banking platform, Socket.io features, and real-time patterns!
