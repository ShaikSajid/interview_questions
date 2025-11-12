# Node.js Questions 41-50: Final Topics & Production Best Practices

---

### Q41. How do you implement comprehensive logging and monitoring in production?

**Answer:**

```javascript
const winston = require('winston');
const { Prometheus } = require('prom-client');

// Structured Logging Service
class LoggingService {
  constructor() {
    this.logger = winston.createLogger({
      level: process.env.LOG_LEVEL || 'info',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.errors({ stack: true }),
        winston.format.json()
      ),
      defaultMeta: {
        service: process.env.SERVICE_NAME || 'banking-service',
        environment: process.env.NODE_ENV || 'development'
      },
      transports: [
        new winston.transports.File({ filename: 'error.log', level: 'error' }),
        new winston.transports.File({ filename: 'combined.log' }),
        new winston.transports.Console({
          format: winston.format.combine(
            winston.format.colorize(),
            winston.format.simple()
          )
        })
      ]
    });
  }
  
  logTransaction(level, message, context) {
    this.logger.log(level, message, {
      ...context,
      type: 'TRANSACTION',
      timestamp: new Date().toISOString()
    });
  }
  
  logError(error, context) {
    this.logger.error('Error occurred', {
      error: {
        message: error.message,
        stack: error.stack,
        name: error.name
      },
      ...context
    });
  }
  
  logPerformance(operation, duration, context) {
    this.logger.info('Performance metric', {
      operation,
      duration,
      ...context,
      type: 'PERFORMANCE'
    });
  }
}

// Prometheus Metrics Service
class MetricsService {
  constructor() {
    const client = require('prom-client');
    
    // Create registry
    this.register = new client.Registry();
    
    // Default metrics
    client.collectDefaultMetrics({ register: this.register });
    
    // Custom metrics
    this.httpRequestDuration = new client.Histogram({
      name: 'http_request_duration_seconds',
      help: 'Duration of HTTP requests in seconds',
      labelNames: ['method', 'route', 'status_code'],
      buckets: [0.1, 0.3, 0.5, 0.7, 1, 3, 5, 7, 10]
    });
    
    this.transactionCounter = new client.Counter({
      name: 'transactions_total',
      help: 'Total number of transactions',
      labelNames: ['type', 'status']
    });
    
    this.activeTransactions = new client.Gauge({
      name: 'active_transactions',
      help: 'Number of active transactions'
    });
    
    this.transactionAmount = new client.Summary({
      name: 'transaction_amount',
      help: 'Transaction amounts',
      labelNames: ['type'],
      percentiles: [0.5, 0.9, 0.95, 0.99]
    });
    
    // Register all metrics
    this.register.registerMetric(this.httpRequestDuration);
    this.register.registerMetric(this.transactionCounter);
    this.register.registerMetric(this.activeTransactions);
    this.register.registerMetric(this.transactionAmount);
  }
  
  recordRequest(method, route, statusCode, duration) {
    this.httpRequestDuration
      .labels(method, route, statusCode)
      .observe(duration);
  }
  
  recordTransaction(type, status, amount) {
    this.transactionCounter.labels(type, status).inc();
    if (amount) {
      this.transactionAmount.labels(type).observe(amount);
    }
  }
  
  incrementActiveTransactions() {
    this.activeTransactions.inc();
  }
  
  decrementActiveTransactions() {
    this.activeTransactions.dec();
  }
  
  async getMetrics() {
    return this.register.metrics();
  }
}

// Complete Monitoring Setup
class MonitoredBankingService {
  constructor() {
    this.logger = new LoggingService();
    this.metrics = new MetricsService();
  }
  
  setupExpress() {
    const express = require('express');
    const app = express();
    
    app.use(express.json());
    
    // Metrics endpoint
    app.get('/metrics', async (req, res) => {
      res.set('Content-Type', this.metrics.register.contentType);
      res.end(await this.metrics.getMetrics());
    });
    
    // Request logging middleware
    app.use((req, res, next) => {
      const start = Date.now();
      
      res.on('finish', () => {
        const duration = (Date.now() - start) / 1000;
        
        this.logger.logTransaction('info', 'HTTP Request', {
          method: req.method,
          path: req.path,
          statusCode: res.statusCode,
          duration,
          correlationId: req.correlationId
        });
        
        this.metrics.recordRequest(
          req.method,
          req.route?.path || req.path,
          res.statusCode.toString(),
          duration
        );
      });
      
      next();
    });
    
    // Transaction endpoint
    app.post('/api/transfer', async (req, res) => {
      this.metrics.incrementActiveTransactions();
      
      try {
        const result = await this.processTransfer(req.body);
        
        this.metrics.recordTransaction(
          'TRANSFER',
          'SUCCESS',
          req.body.amount
        );
        
        res.json(result);
      } catch (error) {
        this.logger.logError(error, {
          operation: 'transfer',
          data: req.body
        });
        
        this.metrics.recordTransaction('TRANSFER', 'FAILED', req.body.amount);
        
        res.status(500).json({ error: error.message });
      } finally {
        this.metrics.decrementActiveTransactions();
      }
    });
    
    return app;
  }
  
  async processTransfer(data) {
    const start = Date.now();
    
    try {
      // Process transfer
      await new Promise(resolve => setTimeout(resolve, 100));
      
      const duration = Date.now() - start;
      this.logger.logPerformance('processTransfer', duration, { data });
      
      return { success: true, transactionId: 'TXN-' + Date.now() };
    } catch (error) {
      throw error;
    }
  }
}
```

### Q42. How do you handle database connection pooling effectively?

**Answer:**

```javascript
const mysql = require('mysql2/promise');

class DatabaseConnectionPool {
  constructor(config) {
    this.pool = mysql.createPool({
      host: config.host,
      user: config.user,
      password: config.password,
      database: config.database,
      connectionLimit: config.connectionLimit || 10,
      queueLimit: config.queueLimit || 0,
      waitForConnections: true,
      enableKeepAlive: true,
      keepAliveInitialDelay: 0
    });
    
    this.setupEventHandlers();
    this.startMonitoring();
  }
  
  setupEventHandlers() {
    this.pool.on('acquire', (connection) => {
      console.log('Connection %d acquired', connection.threadId);
    });
    
    this.pool.on('release', (connection) => {
      console.log('Connection %d released', connection.threadId);
    });
    
    this.pool.on('enqueue', () => {
      console.log('Waiting for available connection slot');
    });
  }
  
  startMonitoring() {
    setInterval(() => {
      const stats = this.getPoolStats();
      console.log('Pool stats:', stats);
      
      if (stats.activeConnections / stats.limit > 0.8) {
        console.warn('⚠️ Pool utilization high:', stats.utilization);
      }
    }, 30000);
  }
  
  async query(sql, params) {
    const connection = await this.pool.getConnection();
    
    try {
      const [rows] = await connection.execute(sql, params);
      return rows;
    } finally {
      connection.release();
    }
  }
  
  async transaction(callback) {
    const connection = await this.pool.getConnection();
    
    try {
      await connection.beginTransaction();
      
      const result = await callback(connection);
      
      await connection.commit();
      return result;
    } catch (error) {
      await connection.rollback();
      throw error;
    } finally {
      connection.release();
    }
  }
  
  getPoolStats() {
    return {
      activeConnections: this.pool._allConnections.length,
      freeConnections: this.pool._freeConnections.length,
      queuedRequests: this.pool._connectionQueue.length,
      limit: this.pool.config.connectionLimit,
      utilization: `${((this.pool._allConnections.length / this.pool.config.connectionLimit) * 100).toFixed(2)}%`
    };
  }
  
  async close() {
    await this.pool.end();
  }
}

// Banking Service with Connection Pool
class BankingDatabaseService {
  constructor() {
    this.pool = new DatabaseConnectionPool({
      host: process.env.DB_HOST,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      database: process.env.DB_NAME,
      connectionLimit: 20
    });
  }
  
  async getAccount(accountId) {
    return this.pool.query(
      'SELECT * FROM accounts WHERE id = ?',
      [accountId]
    );
  }
  
  async transferMoney(from, to, amount) {
    return this.pool.transaction(async (connection) => {
      // Debit
      await connection.execute(
        'UPDATE accounts SET balance = balance - ? WHERE id = ?',
        [amount, from]
      );
      
      // Credit
      await connection.execute(
        'UPDATE accounts SET balance = balance + ? WHERE id = ?',
        [amount, to]
      );
      
      // Log transaction
      const [result] = await connection.execute(
        'INSERT INTO transactions (from_account, to_account, amount) VALUES (?, ?, ?)',
        [from, to, amount]
      );
      
      return result.insertId;
    });
  }
}
```

### Q43. How do you implement security best practices in Node.js?

**Answer:**

```javascript
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');
const mongoSanitize = require('express-mongo-sanitize');
const xss = require('xss-clean');
const hpp = require('hpp');

class SecurityService {
  setupSecurityMiddleware(app) {
    // 1. Helmet - Security headers
    app.use(helmet());
    
    // 2. Rate limiting
    const limiter = rateLimit({
      windowMs: 15 * 60 * 1000, // 15 minutes
      max: 100, // limit each IP to 100 requests per windowMs
      message: 'Too many requests from this IP'
    });
    app.use('/api/', limiter);
    
    // 3. NoSQL injection protection
    app.use(mongoSanitize());
    
    // 4. XSS protection
    app.use(xss());
    
    // 5. HTTP Parameter Pollution protection
    app.use(hpp());
    
    // 6. CORS configuration
    app.use((req, res, next) => {
      res.header('Access-Control-Allow-Origin', process.env.ALLOWED_ORIGINS);
      res.header('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
      res.header('Access-Control-Allow-Headers', 'Content-Type, Authorization');
      next();
    });
    
    // 7. Content Security Policy
    app.use(helmet.contentSecurityPolicy({
      directives: {
        defaultSrc: ["'self'"],
        styleSrc: ["'self'", "'unsafe-inline'"],
        scriptSrc: ["'self'"],
        imgSrc: ["'self'", 'data:', 'https:']
      }
    }));
  }
  
  // Input validation
  validateTransferRequest(data) {
    const errors = [];
    
    if (!data.fromAccount || !data.toAccount) {
      errors.push('Account IDs are required');
    }
    
    if (typeof data.amount !== 'number' || data.amount <= 0) {
      errors.push('Invalid amount');
    }
    
    if (data.amount > 1000000) {
      errors.push('Amount exceeds maximum limit');
    }
    
    if (errors.length > 0) {
      throw new Error(errors.join(', '));
    }
  }
  
  // SQL injection prevention
  sanitizeSQL(input) {
    // Use parameterized queries instead
    // This is just an example of what NOT to do
    return input.replace(/['";\\]/g, '');
  }
  
  // Sensitive data encryption
  encryptSensitiveData(data) {
    const crypto = require('crypto');
    const algorithm = 'aes-256-cbc';
    const key = Buffer.from(process.env.ENCRYPTION_KEY, 'hex');
    const iv = crypto.randomBytes(16);
    
    const cipher = crypto.createCipheriv(algorithm, key, iv);
    let encrypted = cipher.update(JSON.stringify(data), 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    return {
      encrypted,
      iv: iv.toString('hex')
    };
  }
  
  decryptSensitiveData(encrypted, ivHex) {
    const crypto = require('crypto');
    const algorithm = 'aes-256-cbc';
    const key = Buffer.from(process.env.ENCRYPTION_KEY, 'hex');
    const iv = Buffer.from(ivHex, 'hex');
    
    const decipher = crypto.createDecipheriv(algorithm, key, iv);
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return JSON.parse(decrypted);
  }
}
```

### Q44. How do you implement API versioning in Node.js?

**Answer:**

```javascript
// API Versioning Strategies
class VersionedBankingAPI {
  setupRoutes(app) {
    // Strategy 1: URL Path versioning
    app.use('/api/v1', this.getV1Routes());
    app.use('/api/v2', this.getV2Routes());
    
    // Strategy 2: Header versioning
    app.use('/api', this.headerVersionMiddleware.bind(this));
  }
  
  getV1Routes() {
    const router = require('express').Router();
    
    router.get('/accounts/:id', async (req, res) => {
      // V1 implementation
      const account = await this.getAccountV1(req.params.id);
      res.json(account);
    });
    
    return router;
  }
  
  getV2Routes() {
    const router = require('express').Router();
    
    router.get('/accounts/:id', async (req, res) => {
      // V2 implementation with additional fields
      const account = await this.getAccountV2(req.params.id);
      res.json(account);
    });
    
    return router;
  }
  
  headerVersionMiddleware(req, res, next) {
    const version = req.headers['api-version'] || '1';
    
    req.apiVersion = version;
    
    if (version === '1') {
      this.handleV1Request(req, res, next);
    } else if (version === '2') {
      this.handleV2Request(req, res, next);
    } else {
      res.status(400).json({
        error: 'Unsupported API version'
      });
    }
  }
  
  async getAccountV1(accountId) {
    return {
      id: accountId,
      balance: 10000
    };
  }
  
  async getAccountV2(accountId) {
    return {
      id: accountId,
      balance: 10000,
      currency: 'USD',
      type: 'savings',
      createdAt: new Date()
    };
  }
  
  handleV1Request(req, res, next) {
    // V1 specific logic
    next();
  }
  
  handleV2Request(req, res, next) {
    // V2 specific logic
    next();
  }
}
```

### Q45. How do you implement data validation and sanitization?

**Answer:**

```javascript
const Joi = require('joi');

class ValidationService {
  // Schema definitions
  getTransferSchema() {
    return Joi.object({
      fromAccount: Joi.string()
        .pattern(/^ACC-[0-9]{6}$/)
        .required(),
      toAccount: Joi.string()
        .pattern(/^ACC-[0-9]{6}$/)
        .required(),
      amount: Joi.number()
        .positive()
        .max(1000000)
        .required(),
      currency: Joi.string()
        .valid('USD', 'EUR', 'GBP', 'AED')
        .default('USD'),
      description: Joi.string()
        .max(200)
        .optional()
    });
  }
  
  // Validation middleware
  validateTransfer(req, res, next) {
    const schema = this.getTransferSchema();
    
    const { error, value } = schema.validate(req.body, {
      abortEarly: false,
      stripUnknown: true
    });
    
    if (error) {
      return res.status(400).json({
        error: 'Validation failed',
        details: error.details.map(d => ({
          field: d.path.join('.'),
          message: d.message
        }))
      });
    }
    
    req.validatedData = value;
    next();
  }
  
  // Custom validators
  async validateAccountExists(accountId) {
    // Check if account exists in database
    const exists = await this.checkAccountInDB(accountId);
    
    if (!exists) {
      throw new Error(`Account ${accountId} does not exist`);
    }
  }
  
  async validateSufficientBalance(accountId, amount) {
    const balance = await this.getAccountBalance(accountId);
    
    if (balance < amount) {
      throw new Error('Insufficient balance');
    }
  }
  
  sanitizeInput(input) {
    // Remove HTML tags
    const clean = input.replace(/<[^>]*>/g, '');
    
    // Trim whitespace
    return clean.trim();
  }
  
  async checkAccountInDB(accountId) {
    // Database check
    return true;
  }
  
  async getAccountBalance(accountId) {
    // Database query
    return 10000;
  }
}
```

### Q46-50. Quick Fire Essential Topics

**Q46. How do you implement file upload handling securely?**

```javascript
const multer = require('multer');
const path = require('path');

const storage = multer.diskStorage({
  destination: './uploads/',
  filename: (req, file, cb) => {
    const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1E9);
    cb(null, file.fieldname + '-' + uniqueSuffix + path.extname(file.originalname));
  }
});

const upload = multer({
  storage: storage,
  limits: { fileSize: 5 * 1024 * 1024 }, // 5MB
  fileFilter: (req, file, cb) => {
    const allowedTypes = /jpeg|jpg|png|pdf/;
    const extname = allowedTypes.test(path.extname(file.originalname).toLowerCase());
    const mimetype = allowedTypes.test(file.mimetype);
    
    if (mimetype && extname) {
      return cb(null, true);
    } else {
      cb(new Error('Invalid file type'));
    }
  }
});

app.post('/upload', upload.single('document'), (req, res) => {
  res.json({ file: req.file });
});
```

**Q47. How do you implement WebSocket for real-time updates?**

```javascript
const WebSocket = require('ws');

class BankingWebSocketServer {
  constructor(server) {
    this.wss = new WebSocket.Server({ server });
    this.clients = new Map();
    
    this.wss.on('connection', (ws, req) => {
      const userId = this.authenticateUser(req);
      this.clients.set(userId, ws);
      
      ws.on('message', (message) => {
        this.handleMessage(userId, message);
      });
      
      ws.on('close', () => {
        this.clients.delete(userId);
      });
    });
  }
  
  authenticateUser(req) {
    // Extract user from token
    return 'user123';
  }
  
  handleMessage(userId, message) {
    // Process message
  }
  
  sendTransactionUpdate(userId, transaction) {
    const ws = this.clients.get(userId);
    if (ws && ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({
        type: 'TRANSACTION_UPDATE',
        data: transaction
      }));
    }
  }
}
```

**Q48. How do you implement pagination efficiently?**

```javascript
class PaginationService {
  async getTransactions(accountId, page = 1, limit = 50) {
    const offset = (page - 1) * limit;
    
    const [transactions, total] = await Promise.all([
      db.query(
        'SELECT * FROM transactions WHERE account_id = ? ORDER BY date DESC LIMIT ? OFFSET ?',
        [accountId, limit, offset]
      ),
      db.query(
        'SELECT COUNT(*) as count FROM transactions WHERE account_id = ?',
        [accountId]
      )
    ]);
    
    return {
      data: transactions,
      pagination: {
        page,
        limit,
        total: total[0].count,
        totalPages: Math.ceil(total[0].count / limit),
        hasNext: page < Math.ceil(total[0].count / limit),
        hasPrev: page > 1
      }
    };
  }
  
  // Cursor-based pagination (more efficient for large datasets)
  async getTransactionsCursor(accountId, cursor = null, limit = 50) {
    const query = cursor
      ? 'SELECT * FROM transactions WHERE account_id = ? AND id > ? ORDER BY id LIMIT ?'
      : 'SELECT * FROM transactions WHERE account_id = ? ORDER BY id LIMIT ?';
    
    const params = cursor ? [accountId, cursor, limit] : [accountId, limit];
    
    const transactions = await db.query(query, params);
    
    return {
      data: transactions,
      nextCursor: transactions.length === limit ? transactions[transactions.length - 1].id : null
    };
  }
}
```

**Q49. How do you implement job queues and background processing?**

```javascript
const Bull = require('bull');

class JobQueueService {
  constructor() {
    this.transactionQueue = new Bull('transactions', {
      redis: { host: 'localhost', port: 6379 }
    });
    
    this.setupProcessors();
  }
  
  setupProcessors() {
    this.transactionQueue.process(async (job) => {
      console.log('Processing transaction:', job.data);
      
      await this.processTransaction(job.data);
      
      return { success: true };
    });
    
    this.transactionQueue.on('completed', (job, result) => {
      console.log('Job completed:', job.id, result);
    });
    
    this.transactionQueue.on('failed', (job, err) => {
      console.error('Job failed:', job.id, err);
    });
  }
  
  async queueTransaction(data) {
    const job = await this.transactionQueue.add(data, {
      attempts: 3,
      backoff: {
        type: 'exponential',
        delay: 2000
      }
    });
    
    return job.id;
  }
  
  async processTransaction(data) {
    // Process transaction
    await new Promise(resolve => setTimeout(resolve, 1000));
  }
}
```

**Q50. How do you implement comprehensive error tracking?**

```javascript
const Sentry = require('@sentry/node');

class ErrorTrackingService {
  constructor() {
    Sentry.init({
      dsn: process.env.SENTRY_DSN,
      environment: process.env.NODE_ENV,
      tracesSampleRate: 1.0
    });
  }
  
  setupExpressErrorHandler(app) {
    // Request handler must be first
    app.use(Sentry.Handlers.requestHandler());
    
    // Tracing handler
    app.use(Sentry.Handlers.tracingHandler());
    
    // Your routes here
    
    // Error handler must be last
    app.use(Sentry.Handlers.errorHandler());
    
    // Custom error handler
    app.use((err, req, res, next) => {
      console.error(err.stack);
      
      // Send to Sentry with context
      Sentry.captureException(err, {
        user: req.user,
        extra: {
          body: req.body,
          params: req.params,
          query: req.query
        }
      });
      
      res.status(500).json({
        error: 'Internal server error',
        message: process.env.NODE_ENV === 'development' ? err.message : undefined
      });
    });
  }
  
  captureError(error, context = {}) {
    Sentry.captureException(error, {
      extra: context
    });
  }
  
  captureMessage(message, level = 'info') {
    Sentry.captureMessage(message, level);
  }
}
```

---

**🎉 COMPLETE! All 50 Node.js Questions Finished!**

## Summary of All Topics Covered:

### **Section 1 (Q1-Q10): Event Loop & Async Programming**
- Event Loop phases and mechanics
- Async timing (nextTick, setImmediate, setTimeout)
- Callback hell solutions
- Concurrency patterns
- Worker Threads
- Graceful shutdown
- Backpressure handling
- Memory management

### **Section 2 (Q11-Q20): Promises & Async/Await**
- Promise internals
- Promise combinators
- Async/await patterns
- Error handling strategies
- Callback to Promise conversion
- Unhandled rejections
- Microtask vs Macrotask
- Retry logic
- Async iterators
- Promise pooling

### **Section 3 (Q21-Q25): Streams & Buffers**
- Stream types
- Buffer operations
- Large file processing
- Backpressure management
- Custom streams

### **Section 4 (Q26-Q30): Performance**
- Profiling techniques
- Clustering
- Caching strategies
- Database optimization
- Garbage collection

### **Section 5 (Q31-Q35): Architecture**
- Microservices design
- Event-driven architecture
- Circuit breaker pattern
- CQRS implementation
- Distributed transactions

### **Section 6 (Q36-Q40): Advanced Topics**
- Request tracing
- Health checks
- Rate limiting
- Redis caching
- Graceful degradation

### **Section 7 (Q41-Q50): Production**
- Logging & monitoring
- Connection pooling
- Security best practices
- API versioning
- Validation
- File uploads
- WebSockets
- Pagination
- Job queues
- Error tracking

---

**All files created successfully! Ready for your ENBD interview! 🚀**
