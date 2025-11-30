# Q42: Microservices - Service Discovery & Load Balancing

## 📋 Summary
Service discovery enables microservices to find and communicate with each other dynamically. This guide covers **service registry** (Consul, Eureka), health checks, client/server-side discovery, load balancing strategies, service mesh, and comprehensive banking examples.

---

## 🎯 What You'll Learn
- **Service Discovery**: Why needed, how it works
- **Service Registry**: Consul, etcd, Eureka comparison
- **Health Checks**: TCP, HTTP, script-based monitoring
- **Load Balancing**: Round-robin, least connections, weighted
- **Service Mesh**: Istio, Linkerd concepts
- **Banking Examples**: Dynamic payment service discovery

---

## 📖 Comprehensive Explanation

### Why Service Discovery?

**Problem**: In microservices, services need to find each other dynamically:
- IPs change (containers, auto-scaling)
- Multiple instances per service
- Services come and go (deployments, failures)

**Without Service Discovery** ❌:
```javascript
// Hardcoded addresses
const PAYMENT_SERVICE = 'http://192.168.1.10:3000';
```

Problems:
- Can't scale (need to update config for each instance)
- Manual updates when IPs change
- No load balancing
- No health checking

**With Service Discovery** ✅:
```javascript
// Dynamic discovery
const paymentService = await serviceRegistry.discover('payment-service');
// Returns: http://payment-service-1:3000 (healthy instance)
```

---

## 🔍 Service Discovery Patterns

### 1. Client-Side Discovery

**How it works**:
1. Client queries service registry
2. Registry returns list of instances
3. Client chooses instance (load balancing)
4. Client calls service directly

```
Client → Registry ("Where is payment-service?") →
Registry → Client ("Instance A, B, C") →
Client → Service Instance B
```

**Pros**: 
- Client controls load balancing
- No extra network hop

**Cons**:
- Client complexity
- Load balancing logic in every client

### 2. Server-Side Discovery

**How it works**:
1. Client calls load balancer
2. Load balancer queries registry
3. Load balancer forwards to instance

```
Client → Load Balancer → Registry ("Where is payment-service?") →
Load Balancer → Service Instance B
```

**Pros**:
- Simple clients
- Centralized load balancing

**Cons**:
- Extra network hop
- Load balancer is single point of failure

---

## 🗄️ Service Registry Options

### Consul (HashiCorp)

**Features**:
- Service discovery
- Health checking
- Key-value store
- Multi-datacenter support

**Health Checks**:
- HTTP endpoint
- TCP connection
- Script execution
- TTL (time-to-live)

### etcd (CoreOS)

**Features**:
- Distributed key-value store
- Used by Kubernetes
- Strong consistency

### Eureka (Netflix)

**Features**:
- REST-based service registry
- Java/Spring ecosystem
- Client-side discovery

---

## 💻 Complete Consul Implementation

### Install Consul

```bash
# Docker Compose
docker run -d -p 8500:8500 -p 8600:8600/udp --name=consul consul agent -server -ui -bootstrap-expect=1 -client=0.0.0.0
```

### Service Registration Helper

```javascript
// lib/serviceRegistry.js

const axios = require('axios');
const os = require('os');

class ServiceRegistry {
  constructor(consulHost = 'localhost', consulPort = 8500) {
    this.consulUrl = `http://${consulHost}:${consulPort}/v1`;
    this.serviceName = null;
    this.serviceId = null;
    this.checkInterval = null;
  }

  /**
   * Register service with Consul
   */
  async register(config) {
    const {
      name,
      port,
      tags = [],
      healthCheckPath = '/health',
      healthCheckInterval = '10s'
    } = config;

    this.serviceName = name;
    this.serviceId = `${name}-${os.hostname()}-${port}`;

    const serviceDefinition = {
      ID: this.serviceId,
      Name: name,
      Tags: tags,
      Address: this.getLocalIP(),
      Port: port,
      Check: {
        HTTP: `http://${this.getLocalIP()}:${port}${healthCheckPath}`,
        Interval: healthCheckInterval,
        Timeout: '5s',
        DeregisterCriticalServiceAfter: '30s'
      }
    };

    try {
      await axios.put(
        `${this.consulUrl}/agent/service/register`,
        serviceDefinition
      );
      
      console.log(`✅ Service registered: ${this.serviceId}`);
      
      // Graceful shutdown
      this.setupShutdownHooks();
      
      return this.serviceId;
    } catch (error) {
      console.error('Failed to register service:', error.message);
      throw error;
    }
  }

  /**
   * Deregister service
   */
  async deregister() {
    if (!this.serviceId) return;

    try {
      await axios.put(
        `${this.consulUrl}/agent/service/deregister/${this.serviceId}`
      );
      console.log(`✅ Service deregistered: ${this.serviceId}`);
    } catch (error) {
      console.error('Failed to deregister service:', error.message);
    }
  }

  /**
   * Discover healthy service instances
   */
  async discover(serviceName) {
    try {
      const response = await axios.get(
        `${this.consulUrl}/health/service/${serviceName}?passing=true`
      );

      const instances = response.data.map(entry => ({
        id: entry.Service.ID,
        address: entry.Service.Address,
        port: entry.Service.Port,
        tags: entry.Service.Tags,
        url: `http://${entry.Service.Address}:${entry.Service.Port}`
      }));

      if (instances.length === 0) {
        throw new Error(`No healthy instances of ${serviceName} found`);
      }

      return instances;
    } catch (error) {
      console.error(`Failed to discover ${serviceName}:`, error.message);
      throw error;
    }
  }

  /**
   * Get one instance (with load balancing)
   */
  async getServiceInstance(serviceName, strategy = 'round-robin') {
    const instances = await this.discover(serviceName);
    
    switch (strategy) {
      case 'random':
        return this.randomSelect(instances);
      case 'round-robin':
        return this.roundRobinSelect(serviceName, instances);
      default:
        return instances[0];
    }
  }

  /**
   * Random selection
   */
  randomSelect(instances) {
    const index = Math.floor(Math.random() * instances.length);
    return instances[index];
  }

  /**
   * Round-robin selection
   */
  roundRobinSelect(serviceName, instances) {
    if (!this.roundRobinCounters) {
      this.roundRobinCounters = {};
    }
    
    if (!this.roundRobinCounters[serviceName]) {
      this.roundRobinCounters[serviceName] = 0;
    }
    
    const index = this.roundRobinCounters[serviceName] % instances.length;
    this.roundRobinCounters[serviceName]++;
    
    return instances[index];
  }

  /**
   * Get local IP address
   */
  getLocalIP() {
    const interfaces = os.networkInterfaces();
    
    for (const iface of Object.values(interfaces)) {
      for (const alias of iface) {
        if (alias.family === 'IPv4' && !alias.internal) {
          return alias.address;
        }
      }
    }
    
    return 'localhost';
  }

  /**
   * Setup graceful shutdown
   */
  setupShutdownHooks() {
    const shutdown = async () => {
      console.log('\nShutting down...');
      await this.deregister();
      process.exit(0);
    };

    process.on('SIGINT', shutdown);
    process.on('SIGTERM', shutdown);
  }
}

module.exports = ServiceRegistry;
```

### Payment Service with Service Discovery

```javascript
// services/payment-service/index.js

const express = require('express');
const axios = require('axios');
const ServiceRegistry = require('../../lib/serviceRegistry');

const app = express();
app.use(express.json());

const PORT = process.env.PORT || 3000;
const registry = new ServiceRegistry();

// Health check endpoint
app.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    service: 'payment-service',
    uptime: process.uptime(),
    timestamp: new Date().toISOString()
  });
});

// Process payment (discovers account service dynamically)
app.post('/payments', async (req, res) => {
  try {
    const { fromAccount, toAccount, amount } = req.body;
    const paymentId = `PAY-${Date.now()}`;

    // Discover account service
    const accountService = await registry.getServiceInstance('account-service');
    console.log(`Using account service: ${accountService.url}`);

    // Check balance
    const balanceResponse = await axios.get(
      `${accountService.url}/accounts/${fromAccount}/balance`
    );

    if (balanceResponse.data.balance < amount) {
      return res.status(400).json({ error: 'Insufficient funds' });
    }

    // Discover fraud service
    const fraudService = await registry.getServiceInstance('fraud-service');
    console.log(`Using fraud service: ${fraudService.url}`);

    // Fraud check
    const fraudResponse = await axios.post(
      `${fraudService.url}/check`,
      { accountId: fromAccount, amount, paymentId }
    );

    if (fraudResponse.data.riskLevel === 'HIGH') {
      return res.status(400).json({ error: 'Payment blocked - fraud risk' });
    }

    // Execute payment
    await axios.post(`${accountService.url}/accounts/${fromAccount}/debit`, { amount });
    await axios.post(`${accountService.url}/accounts/${toAccount}/credit`, { amount });

    res.status(201).json({
      paymentId,
      status: 'COMPLETED',
      amount,
      fromAccount,
      toAccount
    });

  } catch (error) {
    console.error('Payment error:', error.message);
    res.status(500).json({ error: error.message });
  }
});

// Start server and register with Consul
async function start() {
  app.listen(PORT, async () => {
    console.log(`Payment service running on port ${PORT}`);

    // Register with Consul
    await registry.register({
      name: 'payment-service',
      port: PORT,
      tags: ['payments', 'v1'],
      healthCheckPath: '/health',
      healthCheckInterval: '10s'
    });
  });
}

start();
```

### Account Service with Service Discovery

```javascript
// services/account-service/index.js

const express = require('express');
const { Pool } = require('pg');
const ServiceRegistry = require('../../lib/serviceRegistry');

const app = express();
app.use(express.json());

const PORT = process.env.PORT || 3001;
const registry = new ServiceRegistry();

const pool = new Pool({
  host: process.env.DB_HOST || 'localhost',
  database: 'accounts'
});

// Health check
app.get('/health', async (req, res) => {
  try {
    // Check database connectivity
    await pool.query('SELECT 1');
    
    res.json({
      status: 'healthy',
      service: 'account-service',
      database: 'connected',
      uptime: process.uptime()
    });
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      service: 'account-service',
      database: 'disconnected',
      error: error.message
    });
  }
});

// Get balance
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

// Debit account
app.post('/accounts/:id/debit', async (req, res) => {
  const client = await pool.connect();
  
  try {
    const { id } = req.params;
    const { amount } = req.body;
    
    await client.query('BEGIN');
    
    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
      [amount, id]
    );
    
    await client.query('COMMIT');
    
    res.json({ status: 'success' });
  } catch (error) {
    await client.query('ROLLBACK');
    res.status(500).json({ error: error.message });
  } finally {
    client.release();
  }
});

// Credit account
app.post('/accounts/:id/credit', async (req, res) => {
  const client = await pool.connect();
  
  try {
    const { id } = req.params;
    const { amount } = req.body;
    
    await client.query('BEGIN');
    
    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
      [amount, id]
    );
    
    await client.query('COMMIT');
    
    res.json({ status: 'success' });
  } catch (error) {
    await client.query('ROLLBACK');
    res.status(500).json({ error: error.message });
  } finally {
    client.release();
  }
});

// Start server
async function start() {
  app.listen(PORT, async () => {
    console.log(`Account service running on port ${PORT}`);
    
    await registry.register({
      name: 'account-service',
      port: PORT,
      tags: ['accounts', 'v1'],
      healthCheckPath: '/health',
      healthCheckInterval: '10s'
    });
  });
}

start();
```

### Docker Compose with Consul

```yaml
# docker-compose.yml

version: '3.8'

services:
  consul:
    image: consul:latest
    ports:
      - "8500:8500"
      - "8600:8600/udp"
    command: agent -server -ui -bootstrap-expect=1 -client=0.0.0.0

  account-service:
    build: ./services/account-service
    environment:
      - PORT=3001
      - CONSUL_HOST=consul
    depends_on:
      - consul
    deploy:
      replicas: 3  # Multiple instances

  payment-service:
    build: ./services/payment-service
    ports:
      - "3000-3002:3000"  # Multiple ports
    environment:
      - PORT=3000
      - CONSUL_HOST=consul
    depends_on:
      - consul
      - account-service
    deploy:
      replicas: 2

  fraud-service:
    build: ./services/fraud-service
    environment:
      - PORT=3003
      - CONSUL_HOST=consul
    depends_on:
      - consul
```

---

## ⚖️ Load Balancing Strategies

### 1. Round-Robin

**How**: Distribute requests evenly across instances

```javascript
let currentIndex = 0;

function roundRobin(instances) {
  const instance = instances[currentIndex % instances.length];
  currentIndex++;
  return instance;
}
```

**Use case**: Instances have similar capacity

### 2. Random

```javascript
function random(instances) {
  return instances[Math.floor(Math.random() * instances.length)];
}
```

### 3. Least Connections

**How**: Send to instance with fewest active connections

```javascript
function leastConnections(instances) {
  return instances.reduce((least, current) => 
    current.connections < least.connections ? current : least
  );
}
```

### 4. Weighted Round-Robin

**How**: Instances with higher capacity get more requests

```javascript
function weightedRoundRobin(instances) {
  // Instance with weight 2 gets 2x requests
  const expanded = instances.flatMap(i => 
    Array(i.weight).fill(i)
  );
  return roundRobin(expanded);
}
```

---

## 🎯 Key Takeaways

### Service Discovery Best Practices

1. **Always use health checks**
   - HTTP endpoint checking
   - Database connectivity
   - Dependent service checks

2. **Implement graceful shutdown**
   - Deregister before stopping
   - Drain connections
   - Wait for in-flight requests

3. **Cache service locations**
   - Reduce registry queries
   - Fallback to cache if registry unavailable
   - TTL-based cache invalidation

4. **Retry with exponential backoff**
   - Service temporarily unavailable
   - Network hiccups
   - Prevent cascading failures

5. **Monitor service health**
   - Track healthy/unhealthy instances
   - Alert on service degradation
   - Auto-scaling triggers

### Common Patterns

- **Client-side discovery**: Netflix Eureka
- **Server-side discovery**: Kubernetes Services
- **Service mesh**: Istio, Linkerd (advanced)

---

**File**: `Q42_Microservices_Service_Discovery.md`  
**Status**: ✅ Complete with Consul integration, health checks, load balancing strategies, and complete banking service discovery system
