# APIs Questions 15-17: Advanced Patterns & WebSockets

---

### Q15. How do you implement Circuit Breaker pattern for resilient APIs?

**Answer:**

```javascript
/**
 * Circuit Breaker Pattern Implementation
 */

class CircuitBreaker {
  constructor(options = {}) {
    this.name = options.name || 'CircuitBreaker';
    this.failureThreshold = options.failureThreshold || 5;
    this.successThreshold = options.successThreshold || 2;
    this.timeout = options.timeout || 10000; // 10 seconds
    this.resetTimeout = options.resetTimeout || 60000; // 1 minute
    
    // States: CLOSED, OPEN, HALF_OPEN
    this.state = 'CLOSED';
    this.failureCount = 0;
    this.successCount = 0;
    this.nextAttempt = Date.now();
    
    // Metrics
    this.metrics = {
      totalRequests: 0,
      successfulRequests: 0,
      failedRequests: 0,
      rejectedRequests: 0,
      timeouts: 0
    };
  }
  
  /**
   * Execute function with circuit breaker
   */
  async execute(fn, fallback = null) {
    this.metrics.totalRequests++;
    
    // Check if circuit is open
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        this.metrics.rejectedRequests++;
        
        if (fallback) {
          return fallback();
        }
        
        throw new Error(`Circuit breaker ${this.name} is OPEN`);
      }
      
      // Try half-open
      this.state = 'HALF_OPEN';
      this.successCount = 0;
      console.log(`[${this.name}] Circuit breaker entering HALF_OPEN state`);
    }
    
    try {
      // Execute with timeout
      const result = await this.executeWithTimeout(fn);
      
      // Success
      this.onSuccess();
      return result;
      
    } catch (error) {
      // Failure
      this.onFailure();
      
      if (fallback) {
        return fallback();
      }
      
      throw error;
    }
  }
  
  /**
   * Execute with timeout
   */
  async executeWithTimeout(fn) {
    return Promise.race([
      fn(),
      new Promise((_, reject) => {
        setTimeout(() => {
          this.metrics.timeouts++;
          reject(new Error('Request timeout'));
        }, this.timeout);
      })
    ]);
  }
  
  /**
   * Handle success
   */
  onSuccess() {
    this.metrics.successfulRequests++;
    this.failureCount = 0;
    
    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      
      if (this.successCount >= this.successThreshold) {
        this.state = 'CLOSED';
        console.log(`[${this.name}] Circuit breaker CLOSED`);
      }
    }
  }
  
  /**
   * Handle failure
   */
  onFailure() {
    this.metrics.failedRequests++;
    this.failureCount++;
    this.successCount = 0;
    
    if (this.failureCount >= this.failureThreshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.resetTimeout;
      
      console.error(`[${this.name}] Circuit breaker OPENED (failures: ${this.failureCount})`);
    }
  }
  
  /**
   * Get current state
   */
  getState() {
    return {
      state: this.state,
      failureCount: this.failureCount,
      successCount: this.successCount,
      nextAttempt: this.state === 'OPEN' ? new Date(this.nextAttempt).toISOString() : null,
      metrics: this.metrics
    };
  }
  
  /**
   * Reset circuit breaker
   */
  reset() {
    this.state = 'CLOSED';
    this.failureCount = 0;
    this.successCount = 0;
    this.nextAttempt = Date.now();
  }
}

/**
 * Banking Service with Circuit Breaker
 */
class BankingExternalService {
  constructor() {
    // Circuit breakers for external services
    this.exchangeRateBreaker = new CircuitBreaker({
      name: 'ExchangeRateAPI',
      failureThreshold: 3,
      timeout: 5000,
      resetTimeout: 30000
    });
    
    this.paymentGatewayBreaker = new CircuitBreaker({
      name: 'PaymentGateway',
      failureThreshold: 5,
      timeout: 10000,
      resetTimeout: 60000
    });
    
    this.fraudDetectionBreaker = new CircuitBreaker({
      name: 'FraudDetection',
      failureThreshold: 3,
      timeout: 3000,
      resetTimeout: 30000
    });
  }
  
  /**
   * Get exchange rates with circuit breaker
   */
  async getExchangeRates(currency) {
    return this.exchangeRateBreaker.execute(
      async () => {
        const response = await fetch(`https://api.exchangerate.com/rates/${currency}`, {
          timeout: 5000
        });
        
        if (!response.ok) {
          throw new Error('Exchange rate API failed');
        }
        
        return response.json();
      },
      // Fallback: return cached rates
      async () => {
        console.warn('[ExchangeRateAPI] Using cached rates (circuit breaker open)');
        return this.getCachedExchangeRates(currency);
      }
    );
  }
  
  /**
   * Process payment with circuit breaker
   */
  async processPayment(paymentData) {
    return this.paymentGatewayBreaker.execute(
      async () => {
        const response = await fetch('https://payment-gateway.enbd.com/api/payments', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(paymentData),
          timeout: 10000
        });
        
        if (!response.ok) {
          throw new Error('Payment gateway failed');
        }
        
        return response.json();
      },
      // Fallback: queue payment for later processing
      async () => {
        console.warn('[PaymentGateway] Queueing payment (circuit breaker open)');
        await this.queuePayment(paymentData);
        return {
          status: 'queued',
          message: 'Payment queued for processing'
        };
      }
    );
  }
  
  /**
   * Check fraud with circuit breaker
   */
  async checkFraud(transactionData) {
    return this.fraudDetectionBreaker.execute(
      async () => {
        const response = await fetch('https://fraud-detection.enbd.com/api/check', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(transactionData),
          timeout: 3000
        });
        
        if (!response.ok) {
          throw new Error('Fraud detection API failed');
        }
        
        return response.json();
      },
      // Fallback: use basic fraud checks
      async () => {
        console.warn('[FraudDetection] Using basic checks (circuit breaker open)');
        return this.basicFraudCheck(transactionData);
      }
    );
  }
  
  /**
   * Fallback: Get cached exchange rates
   */
  async getCachedExchangeRates(currency) {
    // Return last known rates from cache/database
    return {
      base: currency,
      rates: {
        USD: 3.67,
        EUR: 4.01,
        GBP: 4.62,
        AED: 1.00
      },
      cached: true
    };
  }
  
  /**
   * Fallback: Queue payment
   */
  async queuePayment(paymentData) {
    // Add to queue for retry
    await redis.lpush('payment_queue', JSON.stringify({
      ...paymentData,
      queuedAt: new Date()
    }));
  }
  
  /**
   * Fallback: Basic fraud check
   */
  basicFraudCheck(transactionData) {
    const { amount, location, time } = transactionData;
    
    let fraudScore = 0;
    
    // High amount
    if (amount > 50000) fraudScore += 30;
    
    // Unusual time (midnight to 5am)
    const hour = new Date(time).getHours();
    if (hour >= 0 && hour < 5) fraudScore += 20;
    
    // Foreign location
    if (location && !location.startsWith('AE')) fraudScore += 25;
    
    return {
      fraudScore,
      riskLevel: fraudScore > 50 ? 'HIGH' : fraudScore > 25 ? 'MEDIUM' : 'LOW',
      fallback: true
    };
  }
  
  /**
   * Get all circuit breaker states
   */
  getCircuitBreakerStates() {
    return {
      exchangeRate: this.exchangeRateBreaker.getState(),
      paymentGateway: this.paymentGatewayBreaker.getState(),
      fraudDetection: this.fraudDetectionBreaker.getState()
    };
  }
}

/**
 * Advanced Circuit Breaker with Bulkhead Pattern
 */
class BulkheadCircuitBreaker extends CircuitBreaker {
  constructor(options = {}) {
    super(options);
    this.maxConcurrent = options.maxConcurrent || 10;
    this.queue = [];
    this.running = 0;
  }
  
  async execute(fn, fallback = null) {
    // Check concurrent limit
    if (this.running >= this.maxConcurrent) {
      throw new Error(`Bulkhead limit reached for ${this.name}`);
    }
    
    this.running++;
    
    try {
      return await super.execute(fn, fallback);
    } finally {
      this.running--;
      
      // Process queue
      if (this.queue.length > 0) {
        const next = this.queue.shift();
        next();
      }
    }
  }
}

/**
 * Express Middleware
 */
function circuitBreakerMiddleware(breaker) {
  return async (req, res, next) => {
    const state = breaker.getState();
    
    res.set('X-Circuit-Breaker', state.state);
    res.set('X-CB-Failure-Count', state.failureCount.toString());
    
    if (state.state === 'OPEN') {
      return res.status(503).json({
        error: 'ServiceUnavailable',
        message: 'Service temporarily unavailable',
        retryAfter: Math.ceil((breaker.nextAttempt - Date.now()) / 1000)
      });
    }
    
    next();
  };
}

/**
 * Usage Example
 */
const express = require('express');
const router = express.Router();

const bankingService = new BankingExternalService();

// Get exchange rates with circuit breaker
router.get('/api/exchange-rates/:currency', async (req, res) => {
  try {
    const rates = await bankingService.getExchangeRates(req.params.currency);
    
    res.json({
      ...rates,
      cached: rates.cached || false
    });
  } catch (error) {
    res.status(503).json({
      error: 'ServiceUnavailable',
      message: 'Unable to fetch exchange rates'
    });
  }
});

// Health endpoint showing circuit breaker states
router.get('/health/circuit-breakers', (req, res) => {
  res.json(bankingService.getCircuitBreakerStates());
});

module.exports = { 
  CircuitBreaker, 
  BulkheadCircuitBreaker,
  BankingExternalService,
  circuitBreakerMiddleware 
};
```

---

### Q16. How do you implement WebSocket APIs for real-time banking?

**Answer:**

```javascript
/**
 * WebSocket Server for Real-time Banking
 */
const WebSocket = require('ws');
const jwt = require('jsonwebtoken');
const Redis = require('ioredis');

/**
 * WebSocket Server with Authentication
 */
class BankingWebSocketServer {
  constructor(server, db) {
    this.wss = new WebSocket.Server({ 
      server,
      verifyClient: this.verifyClient.bind(this)
    });
    
    this.db = db;
    this.redis = new Redis();
    this.redisSub = new Redis();
    
    // Client connections map
    this.clients = new Map(); // userId -> Set of WebSocket connections
    
    // Subscribe to Redis for pub/sub
    this.setupRedisPubSub();
    
    this.wss.on('connection', this.handleConnection.bind(this));
    
    console.log('WebSocket server initialized');
  }
  
  /**
   * Verify client authentication
   */
  async verifyClient(info, callback) {
    const token = this.extractToken(info.req);
    
    if (!token) {
      return callback(false, 401, 'Unauthorized');
    }
    
    try {
      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      info.req.user = decoded;
      callback(true);
    } catch (error) {
      callback(false, 401, 'Invalid token');
    }
  }
  
  /**
   * Extract token from request
   */
  extractToken(req) {
    const authHeader = req.headers['authorization'];
    
    if (authHeader && authHeader.startsWith('Bearer ')) {
      return authHeader.substring(7);
    }
    
    // Check query parameter
    const url = new URL(req.url, 'http://localhost');
    return url.searchParams.get('token');
  }
  
  /**
   * Handle new WebSocket connection
   */
  handleConnection(ws, req) {
    const user = req.user;
    
    // Add to clients map
    if (!this.clients.has(user.userId)) {
      this.clients.set(user.userId, new Set());
    }
    this.clients.get(user.userId).add(ws);
    
    // Set connection metadata
    ws.userId = user.userId;
    ws.customerId = user.customerId;
    ws.isAlive = true;
    ws.subscribedChannels = new Set();
    
    console.log(`[WS] Client connected: ${user.userId}`);
    
    // Send welcome message
    this.sendToClient(ws, {
      type: 'connected',
      message: 'Connected to ENBD Banking WebSocket',
      userId: user.userId
    });
    
    // Handle messages
    ws.on('message', (data) => this.handleMessage(ws, data));
    
    // Handle pong (heartbeat)
    ws.on('pong', () => {
      ws.isAlive = true;
    });
    
    // Handle close
    ws.on('close', () => this.handleClose(ws));
    
    // Handle errors
    ws.on('error', (error) => {
      console.error('[WS] Error:', error);
    });
  }
  
  /**
   * Handle incoming messages
   */
  async handleMessage(ws, data) {
    try {
      const message = JSON.parse(data);
      
      console.log(`[WS] Message from ${ws.userId}:`, message.type);
      
      switch (message.type) {
        case 'subscribe':
          await this.handleSubscribe(ws, message);
          break;
          
        case 'unsubscribe':
          await this.handleUnsubscribe(ws, message);
          break;
          
        case 'ping':
          this.sendToClient(ws, { type: 'pong', timestamp: Date.now() });
          break;
          
        case 'getAccountBalance':
          await this.handleGetAccountBalance(ws, message);
          break;
          
        default:
          this.sendToClient(ws, {
            type: 'error',
            message: `Unknown message type: ${message.type}`
          });
      }
    } catch (error) {
      console.error('[WS] Message handling error:', error);
      this.sendToClient(ws, {
        type: 'error',
        message: 'Invalid message format'
      });
    }
  }
  
  /**
   * Handle subscribe to channels
   */
  async handleSubscribe(ws, message) {
    const { channels } = message;
    
    for (const channel of channels) {
      // Validate access to channel
      if (await this.canAccessChannel(ws, channel)) {
        ws.subscribedChannels.add(channel);
        
        this.sendToClient(ws, {
          type: 'subscribed',
          channel
        });
      } else {
        this.sendToClient(ws, {
          type: 'error',
          message: `Access denied to channel: ${channel}`
        });
      }
    }
  }
  
  /**
   * Handle unsubscribe from channels
   */
  async handleUnsubscribe(ws, message) {
    const { channels } = message;
    
    for (const channel of channels) {
      ws.subscribedChannels.delete(channel);
      
      this.sendToClient(ws, {
        type: 'unsubscribed',
        channel
      });
    }
  }
  
  /**
   * Handle get account balance
   */
  async handleGetAccountBalance(ws, message) {
    const { accountId } = message;
    
    try {
      const result = await this.db.query(
        'SELECT balance FROM accounts WHERE id = $1 AND customer_id = $2',
        [accountId, ws.customerId]
      );
      
      if (result.rows.length === 0) {
        return this.sendToClient(ws, {
          type: 'error',
          message: 'Account not found'
        });
      }
      
      this.sendToClient(ws, {
        type: 'accountBalance',
        accountId,
        balance: result.rows[0].balance,
        timestamp: Date.now()
      });
    } catch (error) {
      this.sendToClient(ws, {
        type: 'error',
        message: 'Failed to fetch account balance'
      });
    }
  }
  
  /**
   * Check if client can access channel
   */
  async canAccessChannel(ws, channel) {
    // Channel format: account:123 or customer:456
    const [type, id] = channel.split(':');
    
    if (type === 'account') {
      // Check if account belongs to customer
      const result = await this.db.query(
        'SELECT 1 FROM accounts WHERE id = $1 AND customer_id = $2',
        [id, ws.customerId]
      );
      return result.rows.length > 0;
    }
    
    if (type === 'customer') {
      return id === ws.customerId;
    }
    
    return false;
  }
  
  /**
   * Setup Redis Pub/Sub for broadcasting
   */
  setupRedisPubSub() {
    this.redisSub.psubscribe('banking:*', (err, count) => {
      if (err) {
        console.error('Redis subscribe error:', err);
      } else {
        console.log(`Subscribed to ${count} channels`);
      }
    });
    
    this.redisSub.on('pmessage', (pattern, channel, message) => {
      this.handleRedisMessage(channel, message);
    });
  }
  
  /**
   * Handle Redis pub/sub messages
   */
  handleRedisMessage(channel, message) {
    try {
      const data = JSON.parse(message);
      
      // Broadcast to subscribed clients
      this.broadcast(channel, data);
    } catch (error) {
      console.error('[Redis] Message parsing error:', error);
    }
  }
  
  /**
   * Broadcast message to subscribed clients
   */
  broadcast(channel, data) {
    let sentCount = 0;
    
    for (const [userId, connections] of this.clients) {
      for (const ws of connections) {
        if (ws.subscribedChannels.has(channel) && ws.readyState === WebSocket.OPEN) {
          this.sendToClient(ws, {
            ...data,
            channel
          });
          sentCount++;
        }
      }
    }
    
    console.log(`[WS] Broadcast to ${sentCount} clients on channel: ${channel}`);
  }
  
  /**
   * Send message to specific client
   */
  sendToClient(ws, data) {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify(data));
    }
  }
  
  /**
   * Send to all connections of a user
   */
  sendToUser(userId, data) {
    const connections = this.clients.get(userId);
    
    if (connections) {
      for (const ws of connections) {
        this.sendToClient(ws, data);
      }
    }
  }
  
  /**
   * Publish event to Redis (for broadcasting)
   */
  async publishEvent(channel, data) {
    await this.redis.publish(`banking:${channel}`, JSON.stringify(data));
  }
  
  /**
   * Handle client disconnect
   */
  handleClose(ws) {
    console.log(`[WS] Client disconnected: ${ws.userId}`);
    
    // Remove from clients map
    const connections = this.clients.get(ws.userId);
    if (connections) {
      connections.delete(ws);
      
      if (connections.size === 0) {
        this.clients.delete(ws.userId);
      }
    }
  }
  
  /**
   * Heartbeat to detect dead connections
   */
  startHeartbeat() {
    setInterval(() => {
      for (const [userId, connections] of this.clients) {
        for (const ws of connections) {
          if (!ws.isAlive) {
            console.log(`[WS] Terminating dead connection: ${userId}`);
            ws.terminate();
            continue;
          }
          
          ws.isAlive = false;
          ws.ping();
        }
      }
    }, 30000); // Every 30 seconds
  }
}

/**
 * Real-time Banking Events
 */
class BankingEventsService {
  constructor(wsServer, db) {
    this.wsServer = wsServer;
    this.db = db;
  }
  
  /**
   * Notify transaction created
   */
  async notifyTransactionCreated(transaction) {
    const channel = `account:${transaction.accountId}`;
    
    await this.wsServer.publishEvent(channel, {
      type: 'transactionCreated',
      transaction: {
        id: transaction.id,
        amount: transaction.amount,
        type: transaction.type,
        description: transaction.description,
        balanceAfter: transaction.balanceAfter,
        createdAt: transaction.createdAt
      }
    });
  }
  
  /**
   * Notify account balance changed
   */
  async notifyBalanceChanged(accountId, newBalance) {
    const channel = `account:${accountId}`;
    
    await this.wsServer.publishEvent(channel, {
      type: 'balanceChanged',
      accountId,
      balance: newBalance,
      timestamp: Date.now()
    });
  }
  
  /**
   * Notify fraud alert
   */
  async notifyFraudAlert(customerId, alert) {
    const channel = `customer:${customerId}`;
    
    await this.wsServer.publishEvent(channel, {
      type: 'fraudAlert',
      severity: alert.severity,
      message: alert.message,
      transactionId: alert.transactionId,
      timestamp: Date.now()
    });
  }
  
  /**
   * Notify loan status updated
   */
  async notifyLoanStatusUpdated(customerId, loan) {
    const channel = `customer:${customerId}`;
    
    await this.wsServer.publishEvent(channel, {
      type: 'loanStatusUpdated',
      loanId: loan.id,
      status: loan.status,
      message: `Your loan application has been ${loan.status}`,
      timestamp: Date.now()
    });
  }
}

/**
 * Client-side WebSocket usage example
 */
const clientExample = `
// Browser/React client
class BankingWebSocketClient {
  constructor(token) {
    this.token = token;
    this.ws = null;
    this.listeners = new Map();
  }
  
  connect() {
    this.ws = new WebSocket(\`wss://api.enbd.com?token=\${this.token}\`);
    
    this.ws.onopen = () => {
      console.log('Connected to ENBD Banking');
      
      // Subscribe to account updates
      this.send({
        type: 'subscribe',
        channels: ['account:123', 'customer:456']
      });
    };
    
    this.ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      this.handleMessage(data);
    };
    
    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };
    
    this.ws.onclose = () => {
      console.log('Disconnected');
      setTimeout(() => this.connect(), 5000); // Reconnect
    };
  }
  
  handleMessage(data) {
    const listeners = this.listeners.get(data.type) || [];
    listeners.forEach(callback => callback(data));
  }
  
  on(eventType, callback) {
    if (!this.listeners.has(eventType)) {
      this.listeners.set(eventType, []);
    }
    this.listeners.get(eventType).push(callback);
  }
  
  send(data) {
    if (this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(data));
    }
  }
}

// Usage
const client = new BankingWebSocketClient('your-jwt-token');
client.connect();

client.on('transactionCreated', (data) => {
  console.log('New transaction:', data.transaction);
  updateUI(data.transaction);
});

client.on('balanceChanged', (data) => {
  console.log('Balance updated:', data.balance);
  updateBalance(data.balance);
});

client.on('fraudAlert', (data) => {
  showAlert(data.message, 'danger');
});
`;

module.exports = { 
  BankingWebSocketServer, 
  BankingEventsService 
};
```

---

### Q17. How do you implement API Gateway patterns with Kong/Nginx?

**Answer:**

```javascript
/**
 * Kong API Gateway Configuration
 */

/**
 * Kong Declarative Configuration (kong.yml)
 */
const kongConfig = `
_format_version: "3.0"

services:
  # Customer Service
  - name: customer-service
    url: http://customer-service:3001
    tags:
      - banking
      - core
    retries: 3
    connect_timeout: 60000
    write_timeout: 60000
    read_timeout: 60000
    
    routes:
      - name: customer-routes
        paths:
          - /api/v1/customers
          - /api/v2/customers
        methods:
          - GET
          - POST
          - PUT
          - PATCH
          - DELETE
        strip_path: false
        preserve_host: false
        
    plugins:
      # JWT Authentication
      - name: jwt
        config:
          header_names:
            - Authorization
          key_claim_name: iss
          secret_is_base64: false
          
      # Rate Limiting
      - name: rate-limiting
        config:
          minute: 100
          hour: 1000
          day: 10000
          policy: redis
          redis_host: redis
          redis_port: 6379
          redis_password: null
          redis_database: 0
          hide_client_headers: false
          
      # Request Transformer
      - name: request-transformer
        config:
          add:
            headers:
              - X-Gateway: Kong
              - X-Gateway-Version: 3.0
          remove:
            headers:
              - X-Internal-Secret
          replace:
            headers:
              - Host: customer-service
              
      # Response Transformer
      - name: response-transformer
        config:
          add:
            headers:
              - X-Response-Time: {{latency}}
              - X-Request-ID: {{request_id}}
          remove:
            headers:
              - X-Internal-Header
              
      # CORS
      - name: cors
        config:
          origins:
            - https://www.enbd.com
            - https://app.enbd.com
          methods:
            - GET
            - POST
            - PUT
            - PATCH
            - DELETE
            - OPTIONS
          headers:
            - Authorization
            - Content-Type
            - X-Request-ID
          exposed_headers:
            - X-Auth-Token
            - X-Request-ID
          credentials: true
          max_age: 3600
          
      # IP Restriction
      - name: ip-restriction
        config:
          allow:
            - 10.0.0.0/8
            - 172.16.0.0/12
            - 192.168.0.0/16
          deny: []
          
      # Request Size Limiting
      - name: request-size-limiting
        config:
          allowed_payload_size: 10
          size_unit: megabytes
          
      # Prometheus
      - name: prometheus
        config:
          status_code_metrics: true
          latency_metrics: true
          bandwidth_metrics: true
          upstream_health_metrics: true

  # Account Service
  - name: account-service
    url: http://account-service:3002
    
    routes:
      - name: account-routes
        paths:
          - /api/v1/accounts
        methods:
          - GET
          - POST
          - PUT
          - DELETE
        strip_path: false
        
    plugins:
      - name: jwt
      
      - name: rate-limiting
        config:
          minute: 200
          hour: 2000
          
      # Circuit Breaker (Kong Enterprise)
      - name: circuit-breaker
        config:
          window_size: 60
          threshold_percentage: 50
          minimum_number_of_calls: 10
          failure_rate_threshold: 30
          slow_call_duration_threshold: 3000
          slow_call_rate_threshold: 50
          wait_duration_in_open_state: 60
          
      # Canary Release
      - name: canary
        config:
          percentage: 10
          upstream_host: account-service-v2:3002
          upstream_port: 3002

  # Transaction Service  
  - name: transaction-service
    url: http://transaction-service:3003
    
    routes:
      - name: transaction-routes
        paths:
          - /api/v1/transactions
        
    plugins:
      - name: jwt
      
      - name: rate-limiting
        config:
          minute: 300
          hour: 5000
          
      # Request Validation
      - name: request-validator
        config:
          version: draft4
          body_schema: |
            {
              "type": "object",
              "required": ["accountId", "amount", "type"],
              "properties": {
                "accountId": {"type": "string", "format": "uuid"},
                "amount": {"type": "number", "minimum": 0.01},
                "type": {"type": "string", "enum": ["debit", "credit"]},
                "description": {"type": "string", "maxLength": 500}
              }
            }

# Global Plugins
plugins:
  # Request ID
  - name: correlation-id
    config:
      header_name: X-Request-ID
      generator: uuid
      echo_downstream: true
      
  # Logging
  - name: file-log
    config:
      path: /var/log/kong/requests.log
      reopen: true
      
  # HTTP Log (to centralized logging)
  - name: http-log
    config:
      http_endpoint: http://logstash:5000
      method: POST
      timeout: 10000
      keepalive: 60000
      
  # Bot Detection
  - name: bot-detection
    config:
      allow:
        - googlebot
        - bingbot
      deny:
        - scrapy
        - curl
        
  # Datadog APM
  - name: datadog
    config:
      host: datadog-agent
      port: 8126
      service_name_tag: kong-gateway

# Upstreams (Load Balancing)
upstreams:
  - name: customer-service-upstream
    algorithm: round-robin
    hash_on: none
    healthchecks:
      active:
        type: http
        http_path: /health
        timeout: 5
        concurrency: 10
        healthy:
          interval: 10
          http_statuses: [200, 302]
          successes: 2
        unhealthy:
          interval: 5
          http_statuses: [429, 500, 503]
          timeouts: 3
          http_failures: 5
      passive:
        healthy:
          http_statuses: [200, 201, 202, 203, 204, 205, 206, 207, 208, 226, 300, 301, 302, 303, 304, 305, 306, 307, 308]
          successes: 5
        unhealthy:
          http_statuses: [429, 500, 503]
          timeouts: 2
          http_failures: 5
    targets:
      - target: customer-service-1:3001
        weight: 100
      - target: customer-service-2:3001
        weight: 100
      - target: customer-service-3:3001
        weight: 100

# Consumers (API Clients)
consumers:
  - username: mobile-app
    custom_id: mobile-app-v2
    tags:
      - client
      - mobile
    jwt_secrets:
      - key: mobile-app-key
        secret: your-jwt-secret
        algorithm: HS256
    acls:
      - group: mobile-apps
      
  - username: web-app
    custom_id: web-app-v3
    jwt_secrets:
      - key: web-app-key
        secret: your-jwt-secret

# Certificates (SSL/TLS)
certificates:
  - cert: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
    key: |
      -----BEGIN PRIVATE KEY-----
      ...
      -----END PRIVATE KEY-----
    tags:
      - production
    snis:
      - name: api.enbd.com
      - name: www.enbd.com
`;

/**
 * Nginx API Gateway Configuration
 */
const nginxConfig = `
# nginx.conf

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 4096;
    use epoll;
}

http {
    include /etc/nginx/mime.types;
    default_type application/json;
    
    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time uct="$upstream_connect_time" '
                    'uht="$upstream_header_time" urt="$upstream_response_time"';
    
    access_log /var/log/nginx/access.log main;
    
    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    
    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1000;
    gzip_types text/plain text/css application/json application/javascript;
    
    # Rate limiting zones
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/m;
    limit_req_zone $http_authorization zone=user_limit:10m rate=1000r/m;
    limit_conn_zone $binary_remote_addr zone=addr:10m;
    
    # Upstream servers
    upstream customer_service {
        least_conn;
        server customer-service-1:3001 max_fails=3 fail_timeout=30s;
        server customer-service-2:3001 max_fails=3 fail_timeout=30s;
        server customer-service-3:3001 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }
    
    upstream account_service {
        least_conn;
        server account-service-1:3002 weight=3;
        server account-service-2:3002 weight=2;
        server account-service-3:3002 weight=1 backup;
        keepalive 32;
    }
    
    # Cache
    proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=api_cache:10m max_size=1g inactive=60m;
    
    # API Gateway Server
    server {
        listen 80;
        listen [::]:80;
        server_name api.enbd.com;
        
        # Redirect to HTTPS
        return 301 https://$server_name$request_uri;
    }
    
    server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name api.enbd.com;
        
        # SSL Configuration
        ssl_certificate /etc/nginx/ssl/api.enbd.com.crt;
        ssl_certificate_key /etc/nginx/ssl/api.enbd.com.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;
        
        # Security Headers
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
        
        # Rate limiting
        limit_req zone=api_limit burst=20 nodelay;
        limit_conn addr 10;
        
        # Request ID
        set $request_id $request_id;
        if ($http_x_request_id != "") {
            set $request_id $http_x_request_id;
        }
        add_header X-Request-ID $request_id always;
        
        # Customer Service
        location /api/v1/customers {
            # Rate limiting
            limit_req zone=user_limit burst=50;
            
            # JWT validation (using lua-resty-jwt)
            access_by_lua_block {
                local jwt = require "resty.jwt"
                local validators = require "resty.jwt-validators"
                
                local auth_header = ngx.var.http_Authorization
                if not auth_header then
                    ngx.status = 401
                    ngx.say('{"error":"Unauthorized"}')
                    ngx.exit(ngx.HTTP_UNAUTHORIZED)
                end
                
                local token = auth_header:sub(8)  -- Remove "Bearer "
                local jwt_obj = jwt:verify(os.getenv("JWT_SECRET"), token, {
                    exp = validators.is_not_expired(),
                    iss = validators.equals("enbd-api")
                })
                
                if not jwt_obj.verified then
                    ngx.status = 401
                    ngx.say('{"error":"Invalid token"}')
                    ngx.exit(ngx.HTTP_UNAUTHORIZED)
                end
                
                ngx.req.set_header("X-User-ID", jwt_obj.payload.userId)
                ngx.req.set_header("X-Customer-ID", jwt_obj.payload.customerId)
            }
            
            # Proxy to upstream
            proxy_pass http://customer_service;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Request-ID $request_id;
            
            # Timeouts
            proxy_connect_timeout 5s;
            proxy_send_timeout 30s;
            proxy_read_timeout 30s;
            
            # Buffering
            proxy_buffering on;
            proxy_buffer_size 4k;
            proxy_buffers 8 4k;
        }
        
        # Account Service with caching
        location /api/v1/accounts {
            # Cache GET requests
            proxy_cache api_cache;
            proxy_cache_key "$request_uri|$http_authorization";
            proxy_cache_valid 200 60s;
            proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
            proxy_cache_background_update on;
            proxy_cache_lock on;
            add_header X-Cache-Status $upstream_cache_status;
            
            proxy_pass http://account_service;
            proxy_set_header Host $host;
            proxy_set_header X-Request-ID $request_id;
        }
        
        # Health Check
        location /health {
            access_log off;
            return 200 "healthy\\n";
            add_header Content-Type text/plain;
        }
        
        # Metrics (Prometheus)
        location /metrics {
            allow 10.0.0.0/8;
            deny all;
            
            content_by_lua_block {
                local prometheus = require("prometheus").init("prometheus_metrics")
                prometheus:collect()
            }
        }
    }
}
`;

/**
 * Custom Kong Plugin (Lua)
 */
const customKongPlugin = `
-- Banking Rate Limiter Plugin

local BasePlugin = require "kong.plugins.base_plugin"
local redis = require "resty.redis"

local BankingRateLimiter = BasePlugin:extend()

BankingRateLimiter.PRIORITY = 1000
BankingRateLimiter.VERSION = "1.0.0"

function BankingRateLimiter:new()
  BankingRateLimiter.super.new(self, "banking-rate-limiter")
end

function BankingRateLimiter:access(conf)
  BankingRateLimiter.super.access(self)
  
  -- Get user tier from JWT
  local jwt = kong.ctx.shared.authenticated_credential
  local tier = jwt and jwt.tier or "free"
  
  -- Get limits based on tier
  local limits = {
    free = { minute = 10, hour = 100 },
    basic = { minute = 60, hour = 1000 },
    premium = { minute = 300, hour = 5000 },
    enterprise = { minute = 1000, hour = 20000 }
  }
  
  local limit = limits[tier] or limits.free
  
  -- Connect to Redis
  local red = redis:new()
  red:set_timeout(1000)
  
  local ok, err = red:connect(conf.redis_host, conf.redis_port)
  if not ok then
    kong.log.err("Failed to connect to Redis: ", err)
    return
  end
  
  -- Check rate limit
  local identifier = kong.client.get_forwarded_ip()
  local key = "rate_limit:" .. tier .. ":" .. identifier
  
  local count, err = red:incr(key)
  if not count then
    kong.log.err("Failed to increment counter: ", err)
    return
  end
  
  if count == 1 then
    red:expire(key, 60)  -- 1 minute window
  end
  
  -- Set headers
  kong.response.set_header("X-RateLimit-Limit", limit.minute)
  kong.response.set_header("X-RateLimit-Remaining", math.max(0, limit.minute - count))
  kong.response.set_header("X-RateLimit-Tier", tier)
  
  -- Deny if exceeded
  if count > limit.minute then
    return kong.response.exit(429, {
      error = "TooManyRequests",
      message = "Rate limit exceeded for tier: " .. tier
    })
  end
  
  red:close()
end

return BankingRateLimiter
`;

module.exports = { kongConfig, nginxConfig, customKongPlugin };
```

---

**Summary Q15-Q17:**
- Circuit Breaker pattern with fallback strategies for resilient microservices ✅
- WebSocket server for real-time banking (transactions, balance updates, fraud alerts) ✅
- Kong/Nginx API Gateway configurations with JWT, rate limiting, caching, load balancing ✅
