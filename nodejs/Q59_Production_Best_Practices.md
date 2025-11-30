# Q59: Production Best Practices

## 📋 Summary
This question covers **production-ready Node.js deployment**, including process management with PM2, health checks, graceful shutdowns, zero-downtime deployments, and comprehensive monitoring strategies for banking applications.

**Key Topics**:
- Process managers (PM2, systemd)
- Cluster mode and load balancing
- Health checks and readiness probes
- Graceful shutdown strategies
- Zero-downtime deployments
- Environment configuration
- Security hardening
- Performance monitoring
- Error tracking and alerting
- Logging best practices
- Database connection pooling
- Rate limiting and DDoS protection

**Banking Use Cases**:
- High-availability payment processing
- 24/7 transaction services
- Multi-instance deployment
- Automated failover
- Real-time health monitoring
- Audit logging
- Compliance and security
- Performance optimization

---

## 🎯 Production Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│              Production Banking API Architecture                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Load Balancer (NGINX)                                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  • SSL termination                                        │  │
│  │  • Rate limiting                                          │  │
│  │  • Request routing                                        │  │
│  └────────────┬─────────────────────────────────────────────┘  │
│               │                                                  │
│               ▼                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  PM2 Cluster Mode (4 instances)                          │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │  │
│  │  │ Node.js │ │ Node.js │ │ Node.js │ │ Node.js │       │  │
│  │  │ Worker  │ │ Worker  │ │ Worker  │ │ Worker  │       │  │
│  │  │  :3000  │ │  :3001  │ │  :3002  │ │  :3003  │       │  │
│  │  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘       │  │
│  │       │           │           │           │             │  │
│  └───────┼───────────┼───────────┼───────────┼─────────────┘  │
│          │           │           │           │                 │
│          └───────────┴───────────┴───────────┘                 │
│                      │                                          │
│                      ▼                                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Shared Services                                         │  │
│  │  ┌────────────┐ ┌──────────┐ ┌─────────────┐          │  │
│  │  │ PostgreSQL │ │  Redis   │ │   Kafka     │          │  │
│  │  │  (Primary) │ │  Cache   │ │  Messages   │          │  │
│  │  └────────────┘ └──────────┘ └─────────────┘          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Monitoring & Logging                                    │  │
│  │  • New Relic / DataDog APM                              │  │
│  │  • ELK Stack (Logs)                                     │  │
│  │  • Prometheus + Grafana (Metrics)                       │  │
│  │  • PagerDuty (Alerts)                                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 💡 Example 1: PM2 Production Setup

### 1. PM2 Configuration

```javascript
// ecosystem.config.js

module.exports = {
  apps: [
    {
      name: 'banking-api',
      script: './src/server.js',
      instances: 'max', // Use all CPU cores
      exec_mode: 'cluster',
      
      // Environment variables
      env: {
        NODE_ENV: 'development',
        PORT: 3000,
      },
      env_production: {
        NODE_ENV: 'production',
        PORT: 3000,
        DB_HOST: 'prod-db.example.com',
        DB_PORT: 5432,
        DB_NAME: 'banking',
        REDIS_HOST: 'prod-redis.example.com',
        REDIS_PORT: 6379,
        LOG_LEVEL: 'info',
        MAX_WORKERS: 4,
      },
      
      // Restart behavior
      max_restarts: 10,
      min_uptime: '10s',
      max_memory_restart: '500M',
      restart_delay: 4000,
      
      // Logging
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
      error_file: './logs/error.log',
      out_file: './logs/out.log',
      merge_logs: true,
      
      // Advanced options
      watch: false, // Don't use in production
      ignore_watch: ['node_modules', 'logs'],
      kill_timeout: 5000,
      listen_timeout: 10000,
      shutdown_with_message: true,
      
      // Health monitoring
      max_memory_restart: '500M',
      exp_backoff_restart_delay: 100,
      
      // Automation
      autorestart: true,
      cron_restart: '0 0 * * *', // Restart daily at midnight
    },
    
    // Background worker
    {
      name: 'transaction-processor',
      script: './src/workers/transactionProcessor.js',
      instances: 2,
      exec_mode: 'cluster',
      env_production: {
        NODE_ENV: 'production',
        KAFKA_BROKERS: 'kafka1:9092,kafka2:9092',
        PROCESSING_BATCH_SIZE: 100,
      },
      max_memory_restart: '1G',
    },
    
    // Scheduled jobs
    {
      name: 'daily-report',
      script: './src/jobs/dailyReport.js',
      cron_restart: '0 1 * * *', // Run at 1 AM daily
      autorestart: false,
      watch: false,
    },
  ],
  
  // Deployment configuration
  deploy: {
    production: {
      user: 'deploy',
      host: ['prod-server-1', 'prod-server-2'],
      ref: 'origin/main',
      repo: 'git@github.com:company/banking-api.git',
      path: '/var/www/banking-api',
      'post-deploy': 'npm install && pm2 reload ecosystem.config.js --env production',
      env: {
        NODE_ENV: 'production',
      },
    },
    staging: {
      user: 'deploy',
      host: 'staging-server',
      ref: 'origin/develop',
      repo: 'git@github.com:company/banking-api.git',
      path: '/var/www/banking-api-staging',
      'post-deploy': 'npm install && pm2 reload ecosystem.config.js --env staging',
    },
  },
};
```

### 2. Production Server with Graceful Shutdown

```javascript
// src/server.js

const express = require('express');
const helmet = require('helmet');
const compression = require('compression');
const { createServer } = require('http');
const { logger } = require('./utils/logger');
const { connectDatabase, disconnectDatabase } = require('./database');
const { connectRedis, disconnectRedis } = require('./cache');
const healthRoutes = require('./routes/health');
const transactionRoutes = require('./routes/transactions');

class ProductionServer {
  constructor() {
    this.app = express();
    this.server = null;
    this.isShuttingDown = false;
    this.activeConnections = new Set();
  }

  async initialize() {
    try {
      // Security middleware
      this.app.use(helmet({
        contentSecurityPolicy: {
          directives: {
            defaultSrc: ["'self'"],
            styleSrc: ["'self'", "'unsafe-inline'"],
          },
        },
        hsts: {
          maxAge: 31536000,
          includeSubDomains: true,
          preload: true,
        },
      }));

      // Compression
      this.app.use(compression());

      // Body parsing
      this.app.use(express.json({ limit: '10mb' }));
      this.app.use(express.urlencoded({ extended: true, limit: '10mb' }));

      // Request tracking
      this.app.use((req, res, next) => {
        req.id = `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
        req.startTime = Date.now();
        
        logger.info('Incoming request', {
          requestId: req.id,
          method: req.method,
          path: req.path,
          ip: req.ip,
        });

        res.on('finish', () => {
          const duration = Date.now() - req.startTime;
          logger.info('Request completed', {
            requestId: req.id,
            method: req.method,
            path: req.path,
            status: res.statusCode,
            duration,
          });
        });

        next();
      });

      // Shutdown middleware (reject new requests during shutdown)
      this.app.use((req, res, next) => {
        if (this.isShuttingDown) {
          res.set('Connection', 'close');
          return res.status(503).json({
            error: 'Server is shutting down',
            code: 'SERVICE_UNAVAILABLE',
          });
        }
        next();
      });

      // Health checks (must be first route)
      this.app.use('/health', healthRoutes);

      // API routes
      this.app.use('/api/v1/transactions', transactionRoutes);

      // 404 handler
      this.app.use((req, res) => {
        res.status(404).json({
          error: 'Not found',
          code: 'ROUTE_NOT_FOUND',
          path: req.path,
        });
      });

      // Global error handler
      this.app.use((err, req, res, next) => {
        logger.error('Unhandled error', {
          requestId: req.id,
          error: err.message,
          stack: err.stack,
        });

        // Don't expose internal errors in production
        const isDev = process.env.NODE_ENV === 'development';
        res.status(err.statusCode || 500).json({
          error: isDev ? err.message : 'Internal server error',
          code: err.code || 'INTERNAL_ERROR',
          ...(isDev && { stack: err.stack }),
        });
      });

      // Connect to database
      await connectDatabase();
      logger.info('Database connected');

      // Connect to Redis
      await connectRedis();
      logger.info('Redis connected');

      // Start server
      const port = process.env.PORT || 3000;
      this.server = createServer(this.app);

      // Track connections for graceful shutdown
      this.server.on('connection', (connection) => {
        this.activeConnections.add(connection);
        connection.on('close', () => {
          this.activeConnections.delete(connection);
        });
      });

      await new Promise((resolve) => {
        this.server.listen(port, () => {
          logger.info(`Server started`, {
            port,
            nodeEnv: process.env.NODE_ENV,
            pid: process.pid,
            nodeVersion: process.version,
          });
          resolve();
        });
      });

      // Setup graceful shutdown
      this.setupGracefulShutdown();

      // PM2 ready signal
      if (process.send) {
        process.send('ready');
      }

    } catch (error) {
      logger.error('Failed to initialize server', { error: error.message });
      process.exit(1);
    }
  }

  setupGracefulShutdown() {
    const gracefulShutdown = async (signal) => {
      if (this.isShuttingDown) {
        logger.warn('Shutdown already in progress');
        return;
      }

      logger.info(`${signal} received, starting graceful shutdown`);
      this.isShuttingDown = true;

      // Stop accepting new connections
      this.server.close(async () => {
        logger.info('Server closed to new connections');

        try {
          // Give active requests time to complete (30 seconds)
          const shutdownTimeout = setTimeout(() => {
            logger.warn('Forcefully closing remaining connections');
            this.activeConnections.forEach((conn) => conn.destroy());
          }, 30000);

          // Wait for active connections to close
          await new Promise((resolve) => {
            const checkInterval = setInterval(() => {
              if (this.activeConnections.size === 0) {
                clearInterval(checkInterval);
                clearTimeout(shutdownTimeout);
                resolve();
              }
            }, 100);
          });

          logger.info('All connections closed');

          // Disconnect from database
          await disconnectDatabase();
          logger.info('Database disconnected');

          // Disconnect from Redis
          await disconnectRedis();
          logger.info('Redis disconnected');

          logger.info('Graceful shutdown complete');
          process.exit(0);

        } catch (error) {
          logger.error('Error during shutdown', { error: error.message });
          process.exit(1);
        }
      });

      // Force shutdown after 35 seconds
      setTimeout(() => {
        logger.error('Forced shutdown due to timeout');
        process.exit(1);
      }, 35000);
    };

    // Listen to shutdown signals
    process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
    process.on('SIGINT', () => gracefulShutdown('SIGINT'));

    // Handle PM2 shutdown
    process.on('message', (msg) => {
      if (msg === 'shutdown') {
        gracefulShutdown('PM2_SHUTDOWN');
      }
    });

    // Handle uncaught errors
    process.on('uncaughtException', (error) => {
      logger.error('Uncaught exception', {
        error: error.message,
        stack: error.stack,
      });
      gracefulShutdown('UNCAUGHT_EXCEPTION');
    });

    process.on('unhandledRejection', (reason, promise) => {
      logger.error('Unhandled promise rejection', {
        reason,
        promise,
      });
      gracefulShutdown('UNHANDLED_REJECTION');
    });
  }

  getApp() {
    return this.app;
  }

  getServer() {
    return this.server;
  }
}

// Start server
const server = new ProductionServer();
server.initialize();

module.exports = server;
```

### 3. Health Check Implementation

```javascript
// src/routes/health.js

const express = require('express');
const { checkDatabaseHealth } = require('../database');
const { checkRedisHealth } = require('../cache');
const { logger } = require('../utils/logger');

const router = express.Router();

/**
 * Liveness probe - is the application running?
 * Used by Kubernetes/Docker to restart unhealthy containers
 */
router.get('/live', (req, res) => {
  res.status(200).json({
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    pid: process.pid,
  });
});

/**
 * Readiness probe - is the application ready to serve traffic?
 * Checks all dependencies
 */
router.get('/ready', async (req, res) => {
  const checks = {
    database: false,
    redis: false,
    memory: false,
  };

  let isReady = true;

  try {
    // Check database
    checks.database = await checkDatabaseHealth();
    if (!checks.database) isReady = false;

    // Check Redis
    checks.redis = await checkRedisHealth();
    if (!checks.redis) isReady = false;

    // Check memory usage
    const memUsage = process.memoryUsage();
    const memLimit = 500 * 1024 * 1024; // 500MB
    checks.memory = memUsage.heapUsed < memLimit;
    if (!checks.memory) {
      logger.warn('Memory usage high', {
        heapUsed: memUsage.heapUsed,
        limit: memLimit,
      });
      isReady = false;
    }

    const status = isReady ? 200 : 503;
    res.status(status).json({
      status: isReady ? 'ready' : 'not_ready',
      checks,
      timestamp: new Date().toISOString(),
      uptime: process.uptime(),
      memory: {
        heapUsed: memUsage.heapUsed,
        heapTotal: memUsage.heapTotal,
        external: memUsage.external,
        rss: memUsage.rss,
      },
    });

  } catch (error) {
    logger.error('Health check failed', { error: error.message });
    res.status(503).json({
      status: 'error',
      checks,
      error: error.message,
      timestamp: new Date().toISOString(),
    });
  }
});

/**
 * Detailed health check - full system diagnostics
 */
router.get('/status', async (req, res) => {
  const startTime = Date.now();

  const status = {
    application: {
      name: 'banking-api',
      version: process.env.npm_package_version || '1.0.0',
      environment: process.env.NODE_ENV,
      nodeVersion: process.version,
      pid: process.pid,
      uptime: process.uptime(),
      startTime: new Date(Date.now() - process.uptime() * 1000).toISOString(),
    },
    system: {
      platform: process.platform,
      arch: process.arch,
      cpus: require('os').cpus().length,
      memory: {
        total: require('os').totalmem(),
        free: require('os').freemem(),
        process: process.memoryUsage(),
      },
      loadAverage: require('os').loadavg(),
    },
    dependencies: {},
  };

  try {
    // Check database
    const dbStart = Date.now();
    status.dependencies.database = {
      status: await checkDatabaseHealth() ? 'healthy' : 'unhealthy',
      responseTime: Date.now() - dbStart,
    };

    // Check Redis
    const redisStart = Date.now();
    status.dependencies.redis = {
      status: await checkRedisHealth() ? 'healthy' : 'unhealthy',
      responseTime: Date.now() - redisStart,
    };

    status.responseTime = Date.now() - startTime;
    status.timestamp = new Date().toISOString();

    res.status(200).json(status);

  } catch (error) {
    logger.error('Status check failed', { error: error.message });
    res.status(500).json({
      ...status,
      error: error.message,
      timestamp: new Date().toISOString(),
    });
  }
});

/**
 * Metrics endpoint for Prometheus
 */
router.get('/metrics', async (req, res) => {
  const memUsage = process.memoryUsage();
  const metrics = `
# HELP nodejs_heap_used_bytes Node.js heap used in bytes
# TYPE nodejs_heap_used_bytes gauge
nodejs_heap_used_bytes ${memUsage.heapUsed}

# HELP nodejs_heap_total_bytes Node.js heap total in bytes
# TYPE nodejs_heap_total_bytes gauge
nodejs_heap_total_bytes ${memUsage.heapTotal}

# HELP nodejs_external_memory_bytes Node.js external memory in bytes
# TYPE nodejs_external_memory_bytes gauge
nodejs_external_memory_bytes ${memUsage.external}

# HELP nodejs_process_uptime_seconds Node.js process uptime in seconds
# TYPE nodejs_process_uptime_seconds gauge
nodejs_process_uptime_seconds ${process.uptime()}

# HELP nodejs_active_handles Number of active handles
# TYPE nodejs_active_handles gauge
nodejs_active_handles ${process._getActiveHandles().length}

# HELP nodejs_active_requests Number of active requests
# TYPE nodejs_active_requests gauge
nodejs_active_requests ${process._getActiveRequests().length}
`.trim();

  res.set('Content-Type', 'text/plain');
  res.send(metrics);
});

module.exports = router;
```

### 4. Database Connection Pool

```javascript
// src/database/index.js

const { Pool } = require('pg');
const { logger } = require('../utils/logger');

let pool = null;

/**
 * Initialize database connection pool
 */
async function connectDatabase() {
  if (pool) {
    logger.warn('Database already connected');
    return pool;
  }

  pool = new Pool({
    host: process.env.DB_HOST || 'localhost',
    port: parseInt(process.env.DB_PORT || '5432'),
    database: process.env.DB_NAME || 'banking',
    user: process.env.DB_USER || 'postgres',
    password: process.env.DB_PASSWORD,
    
    // Connection pool settings
    min: parseInt(process.env.DB_POOL_MIN || '2'),
    max: parseInt(process.env.DB_POOL_MAX || '10'),
    
    // Connection timeout
    connectionTimeoutMillis: 5000,
    
    // Idle timeout
    idleTimeoutMillis: 30000,
    
    // Statement timeout
    statement_timeout: 10000,
    
    // SSL in production
    ssl: process.env.NODE_ENV === 'production' ? {
      rejectUnauthorized: false,
    } : false,
  });

  // Handle pool errors
  pool.on('error', (err, client) => {
    logger.error('Unexpected database error', {
      error: err.message,
      stack: err.stack,
    });
  });

  // Test connection
  try {
    const client = await pool.connect();
    await client.query('SELECT NOW()');
    client.release();
    logger.info('Database connection pool initialized');
  } catch (error) {
    logger.error('Database connection failed', { error: error.message });
    throw error;
  }

  return pool;
}

/**
 * Close database connection pool
 */
async function disconnectDatabase() {
  if (pool) {
    await pool.end();
    pool = null;
    logger.info('Database connection pool closed');
  }
}

/**
 * Check database health
 */
async function checkDatabaseHealth() {
  if (!pool) return false;

  try {
    const client = await pool.connect();
    const result = await client.query('SELECT 1 as health');
    client.release();
    return result.rows[0].health === 1;
  } catch (error) {
    logger.error('Database health check failed', { error: error.message });
    return false;
  }
}

/**
 * Get a client from the pool
 */
function getPool() {
  if (!pool) {
    throw new Error('Database not connected');
  }
  return pool;
}

module.exports = {
  connectDatabase,
  disconnectDatabase,
  checkDatabaseHealth,
  getPool,
};
```

### 5. PM2 Management Commands

```bash
#!/bin/bash
# scripts/pm2-commands.sh

# Start application
pm2 start ecosystem.config.js --env production

# View logs
pm2 logs banking-api

# Monitor
pm2 monit

# Reload (zero-downtime)
pm2 reload banking-api

# Restart
pm2 restart banking-api

# Stop
pm2 stop banking-api

# Delete
pm2 delete banking-api

# List all processes
pm2 list

# Show process details
pm2 show banking-api

# Save current process list
pm2 save

# Setup startup script
pm2 startup

# Update PM2
pm2 update

# Reset restart counter
pm2 reset banking-api

# Scale to 4 instances
pm2 scale banking-api 4
```

---

## 💡 Example 2: Zero-Downtime Deployment

```bash
#!/bin/bash
# scripts/deploy.sh

set -e

echo "🚀 Starting deployment..."

# Variables
APP_NAME="banking-api"
APP_DIR="/var/www/banking-api"
BACKUP_DIR="/var/backups/banking-api"
CURRENT_DATE=$(date +%Y%m%d_%H%M%S)

# Create backup
echo "📦 Creating backup..."
mkdir -p $BACKUP_DIR
tar -czf $BACKUP_DIR/backup_$CURRENT_DATE.tar.gz -C $APP_DIR .

# Pull latest code
echo "📥 Pulling latest code..."
cd $APP_DIR
git fetch origin
git reset --hard origin/main

# Install dependencies
echo "📚 Installing dependencies..."
npm ci --production

# Run database migrations
echo "🗄️ Running database migrations..."
npm run migrate

# Health check before reload
echo "🏥 Pre-deployment health check..."
curl -f http://localhost:3000/health/ready || {
  echo "❌ Health check failed before deployment"
  exit 1
}

# Reload PM2 (zero-downtime)
echo "🔄 Reloading application..."
pm2 reload ecosystem.config.js --env production --update-env

# Wait for application to be ready
echo "⏳ Waiting for application to be ready..."
sleep 5

# Health check after reload
MAX_RETRIES=10
RETRY_COUNT=0

while [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
  if curl -f http://localhost:3000/health/ready; then
    echo "✅ Application is healthy"
    break
  fi
  
  RETRY_COUNT=$((RETRY_COUNT + 1))
  echo "⏳ Retry $RETRY_COUNT/$MAX_RETRIES..."
  sleep 3
done

if [ $RETRY_COUNT -eq $MAX_RETRIES ]; then
  echo "❌ Health check failed after deployment"
  echo "🔙 Rolling back..."
  
  # Rollback
  tar -xzf $BACKUP_DIR/backup_$CURRENT_DATE.tar.gz -C $APP_DIR
  pm2 reload ecosystem.config.js --env production
  
  exit 1
fi

# Cleanup old backups (keep last 5)
echo "🧹 Cleaning up old backups..."
cd $BACKUP_DIR
ls -t | tail -n +6 | xargs -r rm

echo "✅ Deployment completed successfully!"

# Send notification
curl -X POST https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK \
  -H 'Content-Type: application/json' \
  -d "{\"text\": \"✅ Banking API deployed successfully to production\"}"
```

---

## 🎯 Environment Configuration

### 1. Environment Variables

```bash
# .env.production

# Application
NODE_ENV=production
PORT=3000
APP_NAME=banking-api
APP_VERSION=1.0.0

# Database
DB_HOST=prod-db.example.com
DB_PORT=5432
DB_NAME=banking
DB_USER=banking_user
DB_PASSWORD=<secure-password>
DB_POOL_MIN=5
DB_POOL_MAX=20

# Redis
REDIS_HOST=prod-redis.example.com
REDIS_PORT=6379
REDIS_PASSWORD=<secure-password>
REDIS_DB=0

# Security
JWT_SECRET=<secure-jwt-secret>
JWT_EXPIRY=1h
ENCRYPTION_KEY=<32-byte-hex-key>
RATE_LIMIT_MAX=100
RATE_LIMIT_WINDOW=15m

# External Services
KAFKA_BROKERS=kafka1:9092,kafka2:9092,kafka3:9092
KAFKA_CLIENT_ID=banking-api
KAFKA_GROUP_ID=banking-api-group

# Monitoring
NEW_RELIC_LICENSE_KEY=<license-key>
NEW_RELIC_APP_NAME=banking-api-production
SENTRY_DSN=<sentry-dsn>

# Logging
LOG_LEVEL=info
LOG_FORMAT=json
LOG_FILE=./logs/app.log

# Feature Flags
ENABLE_FRAUD_DETECTION=true
ENABLE_RATE_LIMITING=true
ENABLE_METRICS=true
```

### 2. Configuration Management

```javascript
// src/config/index.js

const Joi = require('joi');

/**
 * Configuration schema validation
 */
const configSchema = Joi.object({
  node_env: Joi.string()
    .valid('development', 'staging', 'production')
    .default('development'),
  
  port: Joi.number()
    .port()
    .default(3000),
  
  database: Joi.object({
    host: Joi.string().required(),
    port: Joi.number().port().default(5432),
    name: Joi.string().required(),
    user: Joi.string().required(),
    password: Joi.string().required(),
    pool: Joi.object({
      min: Joi.number().min(1).default(2),
      max: Joi.number().min(1).default(10),
    }),
  }).required(),
  
  redis: Joi.object({
    host: Joi.string().required(),
    port: Joi.number().port().default(6379),
    password: Joi.string().allow(''),
    db: Joi.number().min(0).default(0),
  }).required(),
  
  jwt: Joi.object({
    secret: Joi.string().min(32).required(),
    expiry: Joi.string().default('1h'),
  }).required(),
  
  rateLimit: Joi.object({
    max: Joi.number().min(1).default(100),
    windowMs: Joi.number().min(1000).default(900000), // 15 minutes
  }),
  
  logging: Joi.object({
    level: Joi.string()
      .valid('error', 'warn', 'info', 'debug')
      .default('info'),
    format: Joi.string()
      .valid('json', 'pretty')
      .default('json'),
  }),
}).unknown();

/**
 * Load and validate configuration
 */
function loadConfig() {
  const config = {
    node_env: process.env.NODE_ENV,
    port: parseInt(process.env.PORT || '3000'),
    
    database: {
      host: process.env.DB_HOST,
      port: parseInt(process.env.DB_PORT || '5432'),
      name: process.env.DB_NAME,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      pool: {
        min: parseInt(process.env.DB_POOL_MIN || '2'),
        max: parseInt(process.env.DB_POOL_MAX || '10'),
      },
    },
    
    redis: {
      host: process.env.REDIS_HOST,
      port: parseInt(process.env.REDIS_PORT || '6379'),
      password: process.env.REDIS_PASSWORD || '',
      db: parseInt(process.env.REDIS_DB || '0'),
    },
    
    jwt: {
      secret: process.env.JWT_SECRET,
      expiry: process.env.JWT_EXPIRY || '1h',
    },
    
    rateLimit: {
      max: parseInt(process.env.RATE_LIMIT_MAX || '100'),
      windowMs: parseInt(process.env.RATE_LIMIT_WINDOW || '900000'),
    },
    
    logging: {
      level: process.env.LOG_LEVEL || 'info',
      format: process.env.LOG_FORMAT || 'json',
    },
  };

  // Validate configuration
  const { error, value } = configSchema.validate(config);
  
  if (error) {
    throw new Error(`Configuration validation failed: ${error.message}`);
  }

  return value;
}

module.exports = loadConfig();
```

---

## 🔒 Security Hardening

```javascript
// src/middleware/security.js

const helmet = require('helmet');
const rateLimit = require('express-rate-limit');
const mongoSanitize = require('express-mongo-sanitize');
const hpp = require('hpp');
const cors = require('cors');

/**
 * Apply security middleware
 */
function applySecurityMiddleware(app) {
  // Helmet - Security headers
  app.use(helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'", "'unsafe-inline'"],
        styleSrc: ["'self'", "'unsafe-inline'"],
        imgSrc: ["'self'", 'data:', 'https:'],
      },
    },
    hsts: {
      maxAge: 31536000,
      includeSubDomains: true,
      preload: true,
    },
  }));

  // CORS
  app.use(cors({
    origin: process.env.ALLOWED_ORIGINS?.split(',') || ['https://example.com'],
    credentials: true,
    maxAge: 86400,
  }));

  // Rate limiting
  const limiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100, // Limit each IP to 100 requests per windowMs
    message: 'Too many requests from this IP',
    standardHeaders: true,
    legacyHeaders: false,
  });
  app.use('/api/', limiter);

  // Strict rate limiting for authentication
  const authLimiter = rateLimit({
    windowMs: 15 * 60 * 1000,
    max: 5,
    skipSuccessfulRequests: true,
  });
  app.use('/api/v1/auth/login', authLimiter);

  // Data sanitization against NoSQL injection
  app.use(mongoSanitize());

  // Prevent HTTP Parameter Pollution
  app.use(hpp());

  // Disable X-Powered-By header
  app.disable('x-powered-by');
}

module.exports = { applySecurityMiddleware };
```

---

## 📊 Monitoring & Alerting

```javascript
// src/monitoring/metrics.js

const client = require('prom-client');
const { logger } = require('../utils/logger');

// Create a Registry
const register = new client.Registry();

// Add default metrics
client.collectDefaultMetrics({ register });

// Custom metrics
const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5],
});

const httpRequestTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status'],
});

const transactionTotal = new client.Counter({
  name: 'banking_transactions_total',
  help: 'Total number of banking transactions',
  labelNames: ['type', 'status'],
});

const transactionAmount = new client.Histogram({
  name: 'banking_transaction_amount',
  help: 'Transaction amounts',
  labelNames: ['type'],
  buckets: [10, 50, 100, 500, 1000, 5000, 10000],
});

const fraudDetections = new client.Counter({
  name: 'banking_fraud_detections_total',
  help: 'Total number of fraud detections',
  labelNames: ['risk_level'],
});

// Register custom metrics
register.registerMetric(httpRequestDuration);
register.registerMetric(httpRequestTotal);
register.registerMetric(transactionTotal);
register.registerMetric(transactionAmount);
register.registerMetric(fraudDetections);

/**
 * Middleware to track HTTP metrics
 */
function metricsMiddleware(req, res, next) {
  const start = Date.now();

  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    const route = req.route?.path || req.path;

    httpRequestDuration
      .labels(req.method, route, res.statusCode)
      .observe(duration);

    httpRequestTotal
      .labels(req.method, route, res.statusCode)
      .inc();
  });

  next();
}

/**
 * Record transaction metrics
 */
function recordTransaction(type, amount, status) {
  transactionTotal.labels(type, status).inc();
  transactionAmount.labels(type).observe(amount);
}

/**
 * Record fraud detection
 */
function recordFraudDetection(riskLevel) {
  fraudDetections.labels(riskLevel).inc();
}

/**
 * Get metrics for Prometheus
 */
async function getMetrics() {
  return register.metrics();
}

module.exports = {
  metricsMiddleware,
  recordTransaction,
  recordFraudDetection,
  getMetrics,
};
```

---

## 📚 Common Interview Questions

### Q1: What is graceful shutdown and why is it important?

**Answer**:
Graceful shutdown ensures the application closes cleanly:
1. **Stop accepting new requests**
2. **Complete pending requests** (with timeout)
3. **Close database connections**
4. **Flush logs**
5. **Release resources**

Without it, you risk data corruption, incomplete transactions, and resource leaks.

### Q2: How does PM2 cluster mode work?

**Answer**:
PM2 cluster mode uses Node.js `cluster` module:
- **Master process** manages worker processes
- **Workers** handle requests (one per CPU core)
- **Load balancing** via round-robin
- **Zero-downtime reload** by restarting workers one-by-one
- **Automatic restart** on crashes

### Q3: What are health checks and why multiple endpoints?

**Answer**:
- **Liveness** (`/health/live`): Is process running? Used by orchestrators to restart dead containers
- **Readiness** (`/health/ready`): Can handle traffic? Checks dependencies (DB, Redis)
- **Startup**: Has initialization completed?

Multiple endpoints allow fine-grained health monitoring.

### Q4: How do you achieve zero-downtime deployment?

**Answer**:
1. **Rolling updates**: Deploy to instances gradually
2. **Blue-green deployment**: Switch traffic between environments
3. **PM2 reload**: Gracefully restart workers one-by-one
4. **Health checks**: Ensure new version healthy before routing traffic
5. **Rollback strategy**: Automated rollback on failure

### Q5: What production monitoring is essential?

**Answer**:
1. **APM**: Request tracing, latency, errors (New Relic, DataDog)
2. **Logs**: Structured logging, centralized (ELK, Splunk)
3. **Metrics**: CPU, memory, custom business metrics (Prometheus)
4. **Alerts**: PagerDuty, Slack for critical issues
5. **Synthetic monitoring**: Periodic health checks from external locations

---

## ✅ Production Checklist

```yaml
✅ Before Going to Production:

Performance:
  - [ ] Load testing completed (expected traffic + 2x)
  - [ ] Database connection pooling configured
  - [ ] Redis caching strategy implemented
  - [ ] Static asset compression enabled
  - [ ] CDN configured for static files

Security:
  - [ ] Helmet security headers applied
  - [ ] Rate limiting configured
  - [ ] CORS properly configured
  - [ ] Input validation on all endpoints
  - [ ] Secrets stored in environment variables
  - [ ] SSL/TLS certificates valid
  - [ ] SQL injection prevention
  - [ ] XSS protection enabled

Reliability:
  - [ ] PM2 cluster mode enabled
  - [ ] Graceful shutdown implemented
  - [ ] Health check endpoints working
  - [ ] Database migrations automated
  - [ ] Automatic restart on crash
  - [ ] Circuit breakers for external services

Monitoring:
  - [ ] APM tool integrated (New Relic/DataDog)
  - [ ] Structured logging configured
  - [ ] Error tracking (Sentry)
  - [ ] Metrics collection (Prometheus)
  - [ ] Alerting configured (PagerDuty)
  - [ ] Dashboard created (Grafana)

Operations:
  - [ ] Deployment automation working
  - [ ] Rollback procedure tested
  - [ ] Backup strategy in place
  - [ ] Documentation complete
  - [ ] Runbooks for common issues
  - [ ] On-call rotation established

Compliance:
  - [ ] Audit logging enabled
  - [ ] PII data encrypted
  - [ ] GDPR compliance verified
  - [ ] PCI DSS if handling cards
  - [ ] Data retention policy implemented
```

---

**Status**: ✅ Complete with production-ready deployment strategies, PM2 configuration, health checks, and comprehensive monitoring for banking applications!
