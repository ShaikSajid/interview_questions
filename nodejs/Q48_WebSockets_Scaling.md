# Q48: WebSockets - Scaling & Redis Pub/Sub

## 📋 Summary
This question covers **scaling WebSocket servers** horizontally using Redis pub/sub adapter, handling multiple server instances, sticky sessions, and building production-ready distributed real-time systems for banking applications.

**Key Topics**:
- Horizontal scaling challenges with WebSockets
- Redis pub/sub adapter for Socket.io
- Sticky sessions and load balancing
- Multi-server coordination
- Message broadcasting across servers
- Connection state management
- High availability and failover
- Performance optimization

**Banking Use Cases**:
- Multi-region trading platform
- Global transaction broadcasting
- Distributed fraud detection alerts
- Real-time exchange rates across regions
- Scalable customer support chat
- High-throughput notification system

---

## 🎯 Understanding WebSocket Scaling Challenges

### The Scaling Problem

**Single Server Scenario** (Works Fine):
```
Clients ──→ WebSocket Server ──→ Database
         (All connections on one instance)

Client A ──→ Server ──→ emit('event') ──→ Client B ✅
(Both clients connected to same server)
```

**Multi-Server Scenario** (Problem):
```
Client A ──→ Server 1 ──→ emit('event') ──→ Client B ❌
Client B ──→ Server 2 (Different server, doesn't receive event!)

Server 1 doesn't know about Server 2's connections
```

### Why WebSocket Scaling is Different from HTTP

| Aspect | HTTP | WebSocket |
|--------|------|-----------|
| **Connection** | Stateless | Stateful |
| **Load Balancing** | Round-robin works | Needs sticky sessions |
| **Broadcasting** | N/A | Requires coordination |
| **State** | None | Per-connection state |
| **Scaling** | Easy (add servers) | Complex (needs pub/sub) |

### Solution: Redis Pub/Sub Adapter

```
┌─────────────────────────────────────────────────────────────┐
│          Scaled WebSocket Architecture with Redis            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Client A ──┐                                               │
│  Client B ──┼──→ Load Balancer ──┐                         │
│  Client C ──┘                     │                         │
│                                   ├──→ Server 1             │
│  Client D ──┐                     │    (clients A, B)       │
│  Client E ──┼──→ (Sticky Session)─┤         │               │
│  Client F ──┘                     │         ▼               │
│                                   │    ┌────────────┐       │
│                                   │    │   Redis    │       │
│                                   │    │  Pub/Sub   │       │
│                                   │    └────────────┘       │
│                                   │         ▲               │
│                                   ├──→ Server 2             │
│                                   │    (clients D, E, F)    │
│                                   │                         │
│                                   └──→ Server 3             │
│                                        (clients ...)        │
│                                                              │
│  Broadcast Flow:                                            │
│  1. Client A emits event to Server 1                       │
│  2. Server 1 publishes to Redis                            │
│  3. Redis broadcasts to all servers                        │
│  4. All servers emit to their connected clients            │
│  5. Client F (on Server 2) receives event ✅               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 💡 Example: Scaled Banking Platform with Redis

Complete implementation with cluster mode, Redis adapter, and production patterns.

### Setup

```bash
npm install express socket.io @socket.io/redis-adapter ioredis cluster
```

### Implementation (server-scaled.js)

```javascript
const cluster = require('cluster');
const os = require('os');
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const { createAdapter } = require('@socket.io/redis-adapter');
const Redis = require('ioredis');

const NUM_WORKERS = os.cpus().length;
const PORT = process.env.PORT || 3000;

if (cluster.isPrimary) {
  console.log(`Primary ${process.pid} spawning ${NUM_WORKERS} workers`);
  for (let i = 0; i < NUM_WORKERS; i++) {
    cluster.fork();
  }
  cluster.on('exit', (worker) => {
    console.log(`Worker ${worker.process.pid} died, respawning...`);
    cluster.fork();
  });
} else {
  const app = express();
  const server = http.createServer(app);
  const io = new Server(server, {
    cors: { origin: '*' },
    transports: ['websocket', 'polling']
  });
  
  // Redis adapter
  const pubClient = new Redis();
  const subClient = pubClient.duplicate();
  io.adapter(createAdapter(pubClient, subClient));
  
  console.log(`Worker ${process.pid} connected to Redis`);
  
  io.on('connection', (socket) => {
    console.log(`[Worker ${process.pid}] Client connected: ${socket.id}`);
    
    socket.on('subscribe:account', (accountId) => {
      socket.join(`account:${accountId}`);
      console.log(`Subscribed to account:${accountId}`);
    });
    
    socket.on('transaction:create', async (data) => {
      const { accountId, type, amount } = data;
      
      const transaction = {
        id: `TXN-${Date.now()}`,
        accountId,
        type,
        amount,
        workerId: process.pid,
        timestamp: Date.now()
      };
      
      // Broadcast to ALL servers watching this account
      io.to(`account:${accountId}`).emit('transaction:new', transaction);
      
      console.log(`[Worker ${process.pid}] Transaction broadcast to all servers`);
    });
    
    socket.on('disconnect', () => {
      console.log(`[Worker ${process.pid}] Client disconnected`);
    });
  });
  
  app.get('/health', (req, res) => {
    res.json({
      workerId: process.pid,
      connections: io.sockets.sockets.size
    });
  });
  
  server.listen(PORT, () => {
    console.log(`Worker ${process.pid} listening on ${PORT}`);
  });
}
```

### Testing Multi-Server Broadcasting

```bash
# Start multiple instances
PORT=3001 node server-scaled.js &
PORT=3002 node server-scaled.js &
PORT=3003 node server-scaled.js &

# Client 1 connects to server 1
# Client 2 connects to server 2
# Transaction from Client 1 → Both clients receive it ✅
```

### NGINX Configuration (Sticky Sessions)

```nginx
upstream websocket_backend {
    ip_hash;  # Sticky sessions
    server localhost:3001;
    server localhost:3002;
    server localhost:3003;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://websocket_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_connect_timeout 7d;
        proxy_send_timeout 7d;
        proxy_read_timeout 7d;
    }
}
```

---

## 🎯 Best Practices

### 1. Connection Management

```javascript
// Track connections in Redis
async function trackConnection(userId, socketId) {
  await redis.hset(`connections:${userId}`, socketId, JSON.stringify({
    workerId: process.pid,
    timestamp: Date.now()
  }));
}

async function getUserConnections(userId) {
  return await redis.hgetall(`connections:${userId}`);
}
```

### 2. High Availability Redis

```bash
# Redis Sentinel for failover
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000
```

### 3. Monitoring

```javascript
// Prometheus metrics
const connectionsGauge = new promClient.Gauge({
  name: 'websocket_connections',
  labelNames: ['worker_id']
});

io.on('connection', () => {
  connectionsGauge.inc({ worker_id: process.pid });
});
```

---

## 📚 Interview Questions

**Q1: Why Redis for scaling WebSockets?**

Redis pub/sub allows servers to communicate. When Server 1 emits to a room, it publishes to Redis, which broadcasts to all servers.

**Q2: Sticky sessions vs Redis adapter?**

Both! Sticky sessions keep client on same server (efficiency). Redis syncs events across servers (functionality).

**Q3: How many connections per server?**

Typically 5,000-10,000 connections per server with proper tuning.

---

## ✅ Summary

- **Redis Adapter**: Syncs events across servers
- **Sticky Sessions**: Routes client to same server  
- **Cluster Mode**: Multiple processes per machine
- **High Availability**: Redis Sentinel/Cluster

---

**Status**: ✅ Complete with production-ready scaled WebSocket platform!
