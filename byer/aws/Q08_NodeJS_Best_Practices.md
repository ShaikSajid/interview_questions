# Question 8: Node.js Best Practices for Enterprise Integration

## 🎯 Question
**What are the best practices for building production-grade Node.js applications for Bayer's integration platform? Discuss async patterns, error handling, performance optimization, memory management, logging, security, and architectural patterns.**

---

## 📋 Answer Overview

Node.js best practices for enterprise systems:
- **Async Patterns**: Promises, async/await, proper error handling
- **Error Handling**: Graceful degradation, no unhandled rejections
- **Performance**: Event loop optimization, clustering, caching
- **Memory Management**: Prevent leaks, monitor heap usage
- **Security**: Input validation, dependency scanning, secrets management
- **Logging**: Structured logging, correlation IDs, log levels
- **Architecture**: Layered architecture, dependency injection, testability

---

## 💻 Implementation Examples

### 1. Async/Await Best Practices

```javascript
// async-patterns.js
// WHY: Proper async handling prevents memory leaks and race conditions

/**
 * ✅ GOOD: Proper error handling with try/catch
 * WHY: Unhandled rejections crash Node.js (process.exit)
 */
async function processPatientData(patientId) {
  try {
    const patient = await getPatient(patientId);
    const treatments = await getTreatments(patientId);
    const records = await getMedicalRecords(patientId);
    
    return {
      patient,
      treatments,
      records
    };
    
  } catch (error) {
    // Log error with context
    logger.error('Failed to process patient data', {
      patientId,
      error: error.message,
      stack: error.stack
    });
    
    // Rethrow or return graceful fallback
    throw new ProcessingError('Patient data unavailable', { patientId });
  }
}

/**
 * ✅ GOOD: Parallel async operations
 * WHY: Reduce latency by running independent operations concurrently
 * EXAMPLE: 3 sequential calls (300ms) vs parallel (100ms)
 */
async function getPatientOverview(patientId) {
  try {
    // Run all queries in parallel
    // WHY: Patient, treatments, records don't depend on each other
    const [patient, treatments, records] = await Promise.all([
      getPatient(patientId),
      getTreatments(patientId),
      getMedicalRecords(patientId)
    ]);
    
    return { patient, treatments, records };
    
  } catch (error) {
    // One failure fails entire operation
    // WHY: Promise.all rejects if any promise rejects
    logger.error('Failed to get patient overview', {
      patientId,
      error: error.message
    });
    throw error;
  }
}

/**
 * ✅ GOOD: Promise.allSettled for partial success
 * WHY: Continue even if some operations fail
 */
async function getPatientDataWithFallback(patientId) {
  const results = await Promise.allSettled([
    getPatient(patientId),
    getTreatments(patientId),
    getMedicalRecords(patientId)
  ]);
  
  return {
    patient: results[0].status === 'fulfilled' ? results[0].value : null,
    treatments: results[1].status === 'fulfilled' ? results[1].value : [],
    records: results[2].status === 'fulfilled' ? results[2].value : []
  };
  
  // WHY: Return partial data if some queries fail
  // EXAMPLE: Patient data available, but treatments service down
}

/**
 * ❌ BAD: Missing error handling
 * WHY: Unhandled rejection crashes process
 */
async function badExample(patientId) {
  const patient = await getPatient(patientId);  // Throws error, no catch
  return patient;
  // PROBLEM: If getPatient throws, error propagates uncaught
}

/**
 * ❌ BAD: Sequential async operations
 * WHY: Unnecessary latency (300ms instead of 100ms)
 */
async function badSequential(patientId) {
  const patient = await getPatient(patientId);        // 100ms
  const treatments = await getTreatments(patientId);  // 100ms
  const records = await getMedicalRecords(patientId); // 100ms
  return { patient, treatments, records };
  // TOTAL: 300ms (should be 100ms with Promise.all)
}

/**
 * ✅ GOOD: Async iteration with for-await-of
 * WHY: Process large datasets without loading all into memory
 */
async function* patientRecordStream(patientIds) {
  for (const patientId of patientIds) {
    try {
      const patient = await getPatient(patientId);
      yield patient;
    } catch (error) {
      logger.error('Failed to get patient', { patientId, error: error.message });
      // Continue processing other patients
    }
  }
}

// Usage: Process 10,000 patients without OOM
async function processAllPatients() {
  const patientIds = await getAllPatientIds();  // 10,000 IDs
  
  for await (const patient of patientRecordStream(patientIds)) {
    await sendToDataWarehouse(patient);
  }
  
  // WHY: Memory-efficient streaming (processes one at a time)
}
```

---

### 2. Error Handling Strategy

```javascript
// error-handling.js
// WHY: Proper error handling prevents crashes and improves debugging

/**
 * Custom error classes
 * WHY: Distinguish error types for different handling strategies
 */
class ApplicationError extends Error {
  constructor(message, metadata = {}) {
    super(message);
    this.name = this.constructor.name;
    this.metadata = metadata;
    this.timestamp = new Date().toISOString();
    Error.captureStackTrace(this, this.constructor);
  }
}

class ValidationError extends ApplicationError {
  constructor(message, field) {
    super(message, { field });
    this.statusCode = 400;
  }
}

class NotFoundError extends ApplicationError {
  constructor(resource, id) {
    super(`${resource} not found`, { id });
    this.statusCode = 404;
  }
}

class ExternalServiceError extends ApplicationError {
  constructor(service, message) {
    super(`${service} error: ${message}`, { service });
    this.statusCode = 503;
    this.retryable = true;  // Can retry
  }
}

/**
 * Global error handler
 * WHY: Catch all unhandled errors, prevent process crash
 */
process.on('unhandledRejection', (reason, promise) => {
  logger.error('Unhandled Promise Rejection', {
    reason: reason?.message || reason,
    stack: reason?.stack,
    promise
  });
  
  // In Lambda, let function fail (AWS will retry)
  // In long-running process, gracefully shutdown
  if (process.env.AWS_LAMBDA_FUNCTION_NAME) {
    process.exit(1);
  } else {
    gracefulShutdown();
  }
});

process.on('uncaughtException', (error) => {
  logger.error('Uncaught Exception', {
    error: error.message,
    stack: error.stack
  });
  
  // Always exit on uncaught exception
  // WHY: Process in undefined state, cannot continue
  process.exit(1);
});

/**
 * Express error middleware
 * WHY: Centralized error handling for HTTP endpoints
 */
function errorMiddleware(error, req, res, next) {
  // Log error with request context
  logger.error('Request error', {
    method: req.method,
    path: req.path,
    error: error.message,
    stack: error.stack,
    userId: req.user?.id,
    correlationId: req.correlationId
  });
  
  // Send appropriate response
  if (error instanceof ValidationError) {
    return res.status(400).json({
      error: 'Validation Error',
      message: error.message,
      field: error.metadata.field
    });
  }
  
  if (error instanceof NotFoundError) {
    return res.status(404).json({
      error: 'Not Found',
      message: error.message
    });
  }
  
  if (error instanceof ExternalServiceError) {
    return res.status(503).json({
      error: 'Service Unavailable',
      message: 'External service temporarily unavailable',
      retryAfter: 60  // Retry after 60 seconds
    });
  }
  
  // Generic error (don't expose internal details)
  // WHY: Security - don't leak implementation details
  res.status(500).json({
    error: 'Internal Server Error',
    message: 'An unexpected error occurred',
    requestId: req.correlationId
  });
}

/**
 * ✅ GOOD: Error handling in async middleware
 * WHY: Express doesn't catch async errors by default
 */
function asyncHandler(fn) {
  return (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
}

// Usage
app.get('/patients/:id', asyncHandler(async (req, res) => {
  const patient = await getPatient(req.params.id);
  
  if (!patient) {
    throw new NotFoundError('Patient', req.params.id);
  }
  
  res.json(patient);
}));
```

---

### 3. Performance Optimization

```javascript
// performance.js
// WHY: Optimize for low latency and high throughput

const { performance } = require('perf_hooks');

/**
 * ✅ GOOD: Database connection pooling
 * WHY: Reuse connections, avoid setup overhead
 */
const { Pool } = require('pg');

const pool = new Pool({
  host: process.env.DB_HOST,
  port: 5432,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  max: 20,          // Max connections in pool
  idleTimeoutMillis: 30000,  // Close idle connections after 30s
  connectionTimeoutMillis: 2000  // Fail fast if no connection available
  // WHY: Limit connections to avoid overwhelming database
});

/**
 * ✅ GOOD: Caching with TTL
 * WHY: Reduce database queries, improve response time
 */
const NodeCache = require('node-cache');
const cache = new NodeCache({
  stdTTL: 300,  // 5 minutes default
  checkperiod: 60,  // Check for expired keys every 60s
  useClones: false  // Performance optimization (don't clone objects)
});

async function getPatientCached(patientId) {
  const cacheKey = `patient:${patientId}`;
  
  // Check cache first
  const cached = cache.get(cacheKey);
  if (cached) {
    logger.debug('Cache hit', { patientId });
    return cached;
  }
  
  // Cache miss, query database
  logger.debug('Cache miss', { patientId });
  const patient = await getPatientFromDB(patientId);
  
  // Store in cache
  cache.set(cacheKey, patient);
  
  return patient;
}

/**
 * ✅ GOOD: Batch operations
 * WHY: Reduce round trips to database
 */
async function getMultiplePatients(patientIds) {
  // Single query instead of N queries
  // WHY: 1 round trip instead of 100 (99% latency reduction)
  const result = await pool.query(
    'SELECT * FROM patients WHERE patient_id = ANY($1)',
    [patientIds]
  );
  
  return result.rows;
}

/**
 * ✅ GOOD: Streaming large responses
 * WHY: Reduce memory usage, faster time-to-first-byte
 */
async function streamPatientRecords(req, res) {
  res.setHeader('Content-Type', 'application/json');
  res.write('[');
  
  let first = true;
  
  for await (const patient of patientRecordStream()) {
    if (!first) res.write(',');
    res.write(JSON.stringify(patient));
    first = false;
  }
  
  res.write(']');
  res.end();
  
  // WHY: Stream JSON array without loading all records into memory
}

/**
 * ✅ GOOD: Performance monitoring
 * WHY: Track slow operations, identify bottlenecks
 */
async function withPerformanceTracking(name, fn) {
  const start = performance.now();
  
  try {
    const result = await fn();
    const duration = performance.now() - start;
    
    logger.info('Operation completed', {
      operation: name,
      durationMs: duration.toFixed(2)
    });
    
    // Send metric to CloudWatch
    if (duration > 1000) {  // Slow operation (>1s)
      logger.warn('Slow operation detected', {
        operation: name,
        durationMs: duration.toFixed(2)
      });
    }
    
    return result;
    
  } catch (error) {
    const duration = performance.now() - start;
    
    logger.error('Operation failed', {
      operation: name,
      durationMs: duration.toFixed(2),
      error: error.message
    });
    
    throw error;
  }
}

// Usage
const patient = await withPerformanceTracking(
  'getPatient',
  () => getPatient(patientId)
);
```

---

### 4. Memory Management

```javascript
// memory-management.js
// WHY: Prevent memory leaks, optimize heap usage

/**
 * ✅ GOOD: Monitor heap usage
 * WHY: Detect memory leaks early
 */
function logMemoryUsage() {
  const usage = process.memoryUsage();
  
  logger.info('Memory usage', {
    heapUsedMB: (usage.heapUsed / 1024 / 1024).toFixed(2),
    heapTotalMB: (usage.heapTotal / 1024 / 1024).toFixed(2),
    externalMB: (usage.external / 1024 / 1024).toFixed(2),
    rssMB: (usage.rss / 1024 / 1024).toFixed(2)
  });
  
  // Alert if heap usage > 80% of max
  const heapPercent = (usage.heapUsed / usage.heapTotal) * 100;
  if (heapPercent > 80) {
    logger.warn('High heap usage', {
      heapPercent: heapPercent.toFixed(2)
    });
  }
}

// Log memory every 30 seconds
setInterval(logMemoryUsage, 30000);

/**
 * ❌ BAD: Memory leak with event listeners
 * WHY: Listeners not removed, accumulate over time
 */
function badMemoryLeak() {
  const emitter = new EventEmitter();
  
  setInterval(() => {
    emitter.on('data', (data) => {
      // Process data
    });
  }, 1000);
  
  // PROBLEM: New listener added every second, never removed
  // RESULT: Memory leak, eventual crash
}

/**
 * ✅ GOOD: Remove event listeners
 * WHY: Prevent memory leaks
 */
function goodMemoryManagement() {
  const emitter = new EventEmitter();
  
  const handler = (data) => {
    // Process data
  };
  
  emitter.on('data', handler);
  
  // Clean up when done
  return () => {
    emitter.removeListener('data', handler);
  };
}

/**
 * ✅ GOOD: Limit array size
 * WHY: Prevent unbounded growth
 */
class BoundedQueue {
  constructor(maxSize = 1000) {
    this.queue = [];
    this.maxSize = maxSize;
  }
  
  push(item) {
    if (this.queue.length >= this.maxSize) {
      // Remove oldest item
      this.queue.shift();
    }
    
    this.queue.push(item);
  }
  
  // WHY: Queue never exceeds maxSize
}
```

---

### 5. Structured Logging

```javascript
// logging.js
// WHY: Structured logs enable powerful queries in CloudWatch Logs Insights

const winston = require('winston');

/**
 * ✅ GOOD: Structured JSON logging
 * WHY: Easy to parse and query
 */
const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  defaultMeta: {
    service: 'integration-api',
    environment: process.env.ENVIRONMENT,
    version: process.env.APP_VERSION
  },
  transports: [
    new winston.transports.Console()
  ]
});

/**
 * ✅ GOOD: Correlation ID for tracing
 * WHY: Track request across multiple services
 */
function correlationIdMiddleware(req, res, next) {
  // Get correlation ID from header or generate new one
  req.correlationId = req.headers['x-correlation-id'] || generateId();
  
  // Add to response headers
  res.setHeader('X-Correlation-ID', req.correlationId);
  
  // Add to all logs for this request
  req.logger = logger.child({
    correlationId: req.correlationId,
    method: req.method,
    path: req.path
  });
  
  next();
}

// Usage
app.get('/patients/:id', (req, res) => {
  req.logger.info('Getting patient', {
    patientId: req.params.id
  });
  
  // All logs include correlationId automatically
});

/**
 * Log levels
 * WHY: Filter logs by severity
 */
logger.error('Critical error', { error: error.message });  // Always logged
logger.warn('Warning', { issue: 'High memory usage' });     // Production
logger.info('Request completed', { duration: 123 });        // Production
logger.debug('Detailed info', { query: sql });              // Dev only
logger.trace('Very detailed', { variables: {} });           // Dev only
```

---

## 📊 Node.js Best Practices Summary

### ✅ Do's
1. **Use async/await** (not callbacks)
2. **Handle all promise rejections** (try/catch)
3. **Use connection pooling** (database, HTTP)
4. **Cache frequently accessed data** (5 min TTL)
5. **Batch database operations** (reduce round trips)
6. **Monitor memory usage** (detect leaks early)
7. **Use structured logging** (JSON format)
8. **Add correlation IDs** (trace requests)
9. **Validate all inputs** (prevent injection)
10. **Use environment variables** (no hardcoded values)

### ❌ Don'ts
1. **Don't use synchronous APIs** (blocking event loop)
2. **Don't ignore errors** (crashes production)
3. **Don't use global variables** (memory leaks)
4. **Don't load large files into memory** (use streams)
5. **Don't expose sensitive errors** (security risk)
6. **Don't use long timeouts** (tie up resources)
7. **Don't skip input validation** (security vulnerabilities)

---

## 💰 Performance Metrics

| Optimization | Impact | Example |
|--------------|--------|---------|
| **Connection pooling** | 90% faster | 10ms vs 100ms per query |
| **Caching** | 95% latency reduction | 5ms vs 100ms |
| **Parallel queries** | 3x faster | 100ms vs 300ms |
| **Streaming** | 10x less memory | 10MB vs 100MB |

---

## 🎯 Key Takeaways

**Production-grade Node.js requires**:
- ✅ **Robust error handling**: No unhandled rejections
- ✅ **Performance optimization**: Caching, pooling, parallelization
- ✅ **Memory management**: Monitor usage, prevent leaks
- ✅ **Structured logging**: JSON format with correlation IDs
- ✅ **Security**: Input validation, secrets management

---

**Estimated Reading Time**: 16-19 minutes  
**Difficulty Level**: ⭐⭐⭐⭐ Advanced  
**Prerequisites**: Node.js, async programming  
**Bayer Job Alignment**: 100% - Core Node.js skills
