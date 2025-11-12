# Container Architecture with Docker, ECS, and EKS

## Question 7: Docker + Express.js + ECS Banking Service

### 📋 Question Statement

Design and implement a containerized banking service using Docker, Express.js, and Amazon ECS Fargate. Include:
- Multi-stage Dockerfile optimization
- Docker Compose for local development
- ECS task definition with best practices
- Application Load Balancer integration
- Auto-scaling policies
- Health checks and graceful shutdown
- Secrets management
- Container security hardening

---

### 🐳 Container Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│           ECS FARGATE CONTAINERIZED ARCHITECTURE             │
└─────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│                Application Load Balancer                    │
│  • SSL Termination (ACM Certificate)                       │
│  • Health Checks (/health endpoint)                        │
│  • Sticky Sessions                                         │
└────────────────┬───────────────────────────────────────────┘
                 │
       ┌─────────┴─────────┐
       │                   │
       ▼                   ▼
┌────────────────┐  ┌────────────────┐
│  ECS Service   │  │  ECS Service   │
│   (Task 1)     │  │   (Task 2)     │
│                │  │                │
│ ┌────────────┐ │  │ ┌────────────┐ │
│ │  Express   │ │  │ │  Express   │ │
│ │  Container │ │  │ │  Container │ │
│ │  (Node.js) │ │  │ │  (Node.js) │ │
│ │  512MB RAM │ │  │ │  512MB RAM │ │
│ │  256 CPU   │ │  │ │  256 CPU   │ │
│ └────────────┘ │  │ └────────────┘ │
└────────┬───────┘  └────────┬───────┘
         │                   │
         └─────────┬─────────┘
                   │
                   ▼
         ┌────────────────────┐
         │  RDS PostgreSQL    │
         │  • Multi-AZ        │
         │  • Encrypted       │
         │  • Read Replicas   │
         └────────────────────┘
```

---

### 📦 Multi-Stage Dockerfile (Optimized)

```dockerfile
# ============================================
# STAGE 1: Builder
# ============================================
FROM node:20-alpine AS builder

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies (including devDependencies for build)
RUN npm ci --only=production && \
    npm cache clean --force

# Copy source code
COPY . .

# Build TypeScript (if using TypeScript)
# RUN npm run build

# ============================================
# STAGE 2: Production
# ============================================
FROM node:20-alpine

# Install dumb-init (proper signal handling)
RUN apk add --no-cache dumb-init

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Set working directory
WORKDIR /app

# Copy built artifacts from builder
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/package*.json ./
COPY --from=builder --chown=nodejs:nodejs /app/src ./src

# Environment variables
ENV NODE_ENV=production \
    PORT=3000 \
    NODE_OPTIONS="--max-old-space-size=460"

# Expose port
EXPOSE 3000

# Switch to non-root user
USER nodejs

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

# Use dumb-init as entrypoint (PID 1)
ENTRYPOINT ["dumb-init", "--"]

# Start application
CMD ["node", "src/server.js"]
```

---

### 🚀 Express.js Banking Service

```javascript
// src/server.js
const express = require('express');
const helmet = require('helmet');
const compression = require('compression');
const morgan = require('morgan');
const { Pool } = require('pg');
const { EventBridgeClient, PutEventsCommand } = require('@aws-sdk/client-eventbridge');

const app = express();
const PORT = process.env.PORT || 3000;

// ============================================
// MIDDLEWARE
// ============================================

// Security headers
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"]
    }
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  }
}));

// Compression
app.use(compression());

// Request logging
app.use(morgan('combined'));

// Body parsing
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true, limit: '10mb' }));

// Request ID middleware
app.use((req, res, next) => {
  req.id = req.headers['x-correlation-id'] || `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  res.setHeader('X-Request-ID', req.id);
  next();
});

// ============================================
// DATABASE CONNECTION
// ============================================

const pool = new Pool({
  host: process.env.DB_HOST,
  port: process.env.DB_PORT || 5432,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
  ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false
});

// Test database connection
pool.on('connect', () => {
  console.log('Database connected');
});

pool.on('error', (err) => {
  console.error('Database connection error:', err);
});

// ============================================
// EVENT BRIDGE CLIENT
// ============================================

const eventBridge = new EventBridgeClient({ region: process.env.AWS_REGION || 'us-east-1' });

// ============================================
// ROUTES
// ============================================

// Health check endpoint
app.get('/health', async (req, res) => {
  try {
    // Check database connectivity
    await pool.query('SELECT 1');
    
    res.status(200).json({
      status: 'healthy',
      timestamp: new Date().toISOString(),
      uptime: process.uptime(),
      memory: process.memoryUsage(),
      database: 'connected'
    });
  } catch (error) {
    console.error('Health check failed:', error);
    res.status(503).json({
      status: 'unhealthy',
      error: error.message
    });
  }
});

// Readiness check
app.get('/ready', async (req, res) => {
  try {
    await pool.query('SELECT 1');
    res.status(200).json({ status: 'ready' });
  } catch (error) {
    res.status(503).json({ status: 'not ready', error: error.message });
  }
});

// Create account
app.post('/api/accounts', async (req, res) => {
  const client = await pool.connect();
  
  try {
    const { customerName, email, phone, accountType, initialDeposit } = req.body;

    // Validation
    if (!customerName || !email || !accountType) {
      return res.status(400).json({
        success: false,
        error: 'Missing required fields'
      });
    }

    await client.query('BEGIN');

    // Generate account number
    const accountNumber = `101${Date.now()}`.substr(0, 12);

    // Insert account
    const result = await client.query(`
      INSERT INTO accounts (
        account_number, customer_name, email, phone, 
        account_type, balance, status, created_at, updated_at
      ) VALUES ($1, $2, $3, $4, $5, $6, 'ACTIVE', NOW(), NOW())
      RETURNING *
    `, [accountNumber, customerName, email, phone, accountType, initialDeposit || 0]);

    const account = result.rows[0];

    // Log audit
    await client.query(`
      INSERT INTO account_audit_log (
        account_id, action, performed_by, details, created_at
      ) VALUES ($1, 'ACCOUNT_CREATED', $2, $3, NOW())
    `, [account.id, 'system', JSON.stringify({ initialDeposit })]);

    await client.query('COMMIT');

    // Publish event
    await publishEvent({
      source: 'account.service',
      detailType: 'AccountCreated',
      detail: {
        accountId: account.id,
        accountNumber: account.account_number,
        customerName: account.customer_name,
        email: account.email
      }
    });

    res.status(201).json({
      success: true,
      data: account
    });

  } catch (error) {
    await client.query('ROLLBACK');
    console.error('Create account error:', error);
    res.status(500).json({
      success: false,
      error: 'Failed to create account'
    });
  } finally {
    client.release();
  }
});

// Get account by ID
app.get('/api/accounts/:id', async (req, res) => {
  try {
    const { id } = req.params;

    const result = await pool.query(
      'SELECT * FROM accounts WHERE id = $1',
      [id]
    );

    if (result.rows.length === 0) {
      return res.status(404).json({
        success: false,
        error: 'Account not found'
      });
    }

    res.status(200).json({
      success: true,
      data: result.rows[0]
    });

  } catch (error) {
    console.error('Get account error:', error);
    res.status(500).json({
      success: false,
      error: 'Failed to retrieve account'
    });
  }
});

// Update account
app.put('/api/accounts/:id', async (req, res) => {
  const client = await pool.connect();
  
  try {
    const { id } = req.params;
    const { email, phone } = req.body;

    await client.query('BEGIN');

    const result = await client.query(`
      UPDATE accounts 
      SET email = COALESCE($1, email),
          phone = COALESCE($2, phone),
          updated_at = NOW()
      WHERE id = $3
      RETURNING *
    `, [email, phone, id]);

    if (result.rows.length === 0) {
      await client.query('ROLLBACK');
      return res.status(404).json({
        success: false,
        error: 'Account not found'
      });
    }

    // Audit log
    await client.query(`
      INSERT INTO account_audit_log (
        account_id, action, performed_by, details, created_at
      ) VALUES ($1, 'ACCOUNT_UPDATED', $2, $3, NOW())
    `, [id, 'system', JSON.stringify({ email, phone })]);

    await client.query('COMMIT');

    res.status(200).json({
      success: true,
      data: result.rows[0]
    });

  } catch (error) {
    await client.query('ROLLBACK');
    console.error('Update account error:', error);
    res.status(500).json({
      success: false,
      error: 'Failed to update account'
    });
  } finally {
    client.release();
  }
});

// ============================================
// HELPER FUNCTIONS
// ============================================

async function publishEvent({ source, detailType, detail }) {
  try {
    const command = new PutEventsCommand({
      Entries: [{
        Time: new Date(),
        Source: source,
        DetailType: detailType,
        Detail: JSON.stringify(detail),
        EventBusName: process.env.EVENT_BUS_NAME || 'banking-event-bus'
      }]
    });

    await eventBridge.send(command);
    console.log(`Event published: ${detailType}`);
  } catch (error) {
    console.error('Failed to publish event:', error);
  }
}

// ============================================
// ERROR HANDLING
// ============================================

// 404 handler
app.use((req, res) => {
  res.status(404).json({
    success: false,
    error: 'Route not found'
  });
});

// Global error handler
app.use((err, req, res, next) => {
  console.error('Unhandled error:', err);
  res.status(500).json({
    success: false,
    error: 'Internal server error',
    requestId: req.id
  });
});

// ============================================
// GRACEFUL SHUTDOWN
// ============================================

let server;

async function gracefulShutdown(signal) {
  console.log(`\n${signal} received. Starting graceful shutdown...`);

  // Stop accepting new connections
  server.close(async () => {
    console.log('HTTP server closed');

    // Close database connections
    await pool.end();
    console.log('Database connections closed');

    process.exit(0);
  });

  // Force shutdown after 30 seconds
  setTimeout(() => {
    console.error('Forced shutdown after timeout');
    process.exit(1);
  }, 30000);
}

// Handle termination signals
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));

// ============================================
// START SERVER
// ============================================

server = app.listen(PORT, '0.0.0.0', () => {
  console.log(`Server running on port ${PORT}`);
  console.log(`Environment: ${process.env.NODE_ENV}`);
  console.log(`Database: ${process.env.DB_HOST}`);
});

module.exports = app;
```

---

### 🐳 Docker Compose for Local Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:15-alpine
    container_name: banking-postgres
    environment:
      POSTGRES_DB: banking_db
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password123
      POSTGRES_INITDB_ARGS: "-E UTF8"
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d banking_db"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - banking-network

  # Account Service
  account-service:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: account-service
    environment:
      NODE_ENV: development
      PORT: 3000
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: banking_db
      DB_USER: admin
      DB_PASSWORD: password123
      AWS_REGION: us-east-1
      EVENT_BUS_NAME: banking-event-bus
    ports:
      - "3000:3000"
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - ./src:/app/src  # Hot reload for development
      - /app/node_modules
    healthcheck:
      test: ["CMD", "node", "-e", "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    networks:
      - banking-network

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: banking-redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    networks:
      - banking-network

  # LocalStack (AWS local emulation)
  localstack:
    image: localstack/localstack:latest
    container_name: banking-localstack
    environment:
      SERVICES: s3,dynamodb,sqs,sns,lambda,events
      DEFAULT_REGION: us-east-1
      DATA_DIR: /tmp/localstack/data
    ports:
      - "4566:4566"
    volumes:
      - localstack-data:/tmp/localstack
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - banking-network

volumes:
  postgres-data:
  redis-data:
  localstack-data:

networks:
  banking-network:
    driver: bridge
```

---

### 📊 Database Initialization Script

```sql
-- init-db.sql

-- Accounts table
CREATE TABLE IF NOT EXISTS accounts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_number VARCHAR(20) UNIQUE NOT NULL,
  customer_name VARCHAR(255) NOT NULL,
  email VARCHAR(255) NOT NULL,
  phone VARCHAR(20),
  account_type VARCHAR(50) NOT NULL CHECK (account_type IN ('SAVINGS', 'CHECKING', 'CURRENT', 'FIXED_DEPOSIT')),
  balance DECIMAL(15, 2) DEFAULT 0.00,
  status VARCHAR(20) DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE', 'INACTIVE', 'CLOSED', 'FROZEN')),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  version INTEGER DEFAULT 0
);

-- Indexes
CREATE INDEX idx_accounts_account_number ON accounts(account_number);
CREATE INDEX idx_accounts_email ON accounts(email);
CREATE INDEX idx_accounts_status ON accounts(status);

-- Audit log table
CREATE TABLE IF NOT EXISTS account_audit_log (
  id SERIAL PRIMARY KEY,
  account_id UUID REFERENCES accounts(id),
  action VARCHAR(50) NOT NULL,
  performed_by VARCHAR(255),
  details JSONB,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_audit_account_id ON account_audit_log(account_id);
CREATE INDEX idx_audit_created_at ON account_audit_log(created_at);

-- Insert sample data
INSERT INTO accounts (account_number, customer_name, email, phone, account_type, balance, status)
VALUES 
  ('1011234567890', 'Ahmed Al Mansoori', 'ahmed@example.com', '+971501234567', 'SAVINGS', 10000.00, 'ACTIVE'),
  ('1019876543210', 'Fatima Hassan', 'fatima@example.com', '+971509876543', 'CHECKING', 5000.00, 'ACTIVE'),
  ('1015555555555', 'Mohammed Ali', 'mohammed@example.com', '+971505555555', 'CURRENT', 25000.00, 'ACTIVE');

-- Create function for updated_at trigger
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = CURRENT_TIMESTAMP;
  RETURN NEW;
END;
$$ language 'plpgsql';

-- Trigger for auto-updating updated_at
CREATE TRIGGER update_accounts_updated_at
BEFORE UPDATE ON accounts
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();
```

---

### ☁️ ECS Task Definition (CDK)

```typescript
// infrastructure/cdk/ecs-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as ecr from 'aws-cdk-lib/aws-ecr';
import * as elbv2 from 'aws-cdk-lib/aws-elasticloadbalancingv2';
import * as logs from 'aws-cdk-lib/aws-logs';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as secretsmanager from 'aws-cdk-lib/aws-secretsmanager';
import * as rds from 'aws-cdk-lib/aws-rds';

export class ECSStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // VPC
    const vpc = new ec2.Vpc(this, 'BankingVPC', {
      maxAzs: 3,
      natGateways: 2,
      subnetConfiguration: [
        {
          name: 'Public',
          subnetType: ec2.SubnetType.PUBLIC,
          cidrMask: 24
        },
        {
          name: 'Private',
          subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,
          cidrMask: 24
        },
        {
          name: 'Isolated',
          subnetType: ec2.SubnetType.PRIVATE_ISOLATED,
          cidrMask: 24
        }
      ]
    });

    // RDS PostgreSQL
    const dbSecret = new secretsmanager.Secret(this, 'DBSecret', {
      generateSecretString: {
        secretStringTemplate: JSON.stringify({ username: 'admin' }),
        generateStringKey: 'password',
        excludePunctuation: true,
        passwordLength: 32
      }
    });

    const database = new rds.DatabaseInstance(this, 'PostgreSQL', {
      engine: rds.DatabaseInstanceEngine.postgres({
        version: rds.PostgresEngineVersion.VER_15_3
      }),
      instanceType: ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.MEDIUM),
      vpc,
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_ISOLATED },
      multiAz: true,
      allocatedStorage: 100,
      maxAllocatedStorage: 500,
      storageEncrypted: true,
      credentials: rds.Credentials.fromSecret(dbSecret),
      databaseName: 'banking_db',
      backupRetention: cdk.Duration.days(7),
      deletionProtection: true,
      removalPolicy: cdk.RemovalPolicy.RETAIN
    });

    // ECS Cluster
    const cluster = new ecs.Cluster(this, 'BankingCluster', {
      clusterName: 'banking-cluster',
      vpc,
      containerInsights: true
    });

    // ECR Repository
    const repository = new ecr.Repository(this, 'AccountServiceRepo', {
      repositoryName: 'account-service',
      imageScanOnPush: true,
      imageTagMutability: ecr.TagMutability.IMMUTABLE,
      lifecycleRules: [{
        maxImageCount: 10,
        description: 'Keep only 10 images'
      }]
    });

    // CloudWatch Log Group
    const logGroup = new logs.LogGroup(this, 'AccountServiceLogs', {
      logGroupName: '/ecs/account-service',
      retention: logs.RetentionDays.ONE_MONTH,
      removalPolicy: cdk.RemovalPolicy.DESTROY
    });

    // Task Execution Role
    const executionRole = new iam.Role(this, 'TaskExecutionRole', {
      assumedBy: new iam.ServicePrincipal('ecs-tasks.amazonaws.com'),
      managedPolicies: [
        iam.ManagedPolicy.fromAwsManagedPolicyName('service-role/AmazonECSTaskExecutionRolePolicy')
      ]
    });

    dbSecret.grantRead(executionRole);

    // Task Role
    const taskRole = new iam.Role(this, 'TaskRole', {
      assumedBy: new iam.ServicePrincipal('ecs-tasks.amazonaws.com')
    });

    // Grant EventBridge permissions
    taskRole.addToPolicy(new iam.PolicyStatement({
      actions: ['events:PutEvents'],
      resources: ['*']
    }));

    // Task Definition
    const taskDefinition = new ecs.FargateTaskDefinition(this, 'AccountServiceTask', {
      memoryLimitMiB: 512,
      cpu: 256,
      executionRole,
      taskRole
    });

    // Container Definition
    const container = taskDefinition.addContainer('AccountService', {
      image: ecs.ContainerImage.fromEcrRepository(repository, 'latest'),
      logging: ecs.LogDrivers.awsLogs({
        streamPrefix: 'account-service',
        logGroup
      }),
      environment: {
        NODE_ENV: 'production',
        PORT: '3000',
        AWS_REGION: this.region,
        EVENT_BUS_NAME: 'banking-event-bus',
        DB_HOST: database.dbInstanceEndpointAddress,
        DB_PORT: database.dbInstanceEndpointPort,
        DB_NAME: 'banking_db'
      },
      secrets: {
        DB_USER: ecs.Secret.fromSecretsManager(dbSecret, 'username'),
        DB_PASSWORD: ecs.Secret.fromSecretsManager(dbSecret, 'password')
      },
      healthCheck: {
        command: ['CMD-SHELL', 'node -e "require(\'http\').get(\'http://localhost:3000/health\', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"'],
        interval: cdk.Duration.seconds(30),
        timeout: cdk.Duration.seconds(5),
        retries: 3,
        startPeriod: cdk.Duration.seconds(60)
      }
    });

    container.addPortMappings({
      containerPort: 3000,
      protocol: ecs.Protocol.TCP
    });

    // Application Load Balancer
    const alb = new elbv2.ApplicationLoadBalancer(this, 'ALB', {
      vpc,
      internetFacing: true,
      loadBalancerName: 'account-service-alb'
    });

    const listener = alb.addListener('Listener', {
      port: 443,
      protocol: elbv2.ApplicationProtocol.HTTPS,
      certificates: [/* ACM certificate */]
    });

    // ECS Service
    const service = new ecs.FargateService(this, 'AccountService', {
      cluster,
      taskDefinition,
      serviceName: 'account-service',
      desiredCount: 2,
      minHealthyPercent: 100,
      maxHealthyPercent: 200,
      healthCheckGracePeriod: cdk.Duration.seconds(60),
      circuitBreaker: { rollback: true },
      enableECSManagedTags: true,
      propagateTags: ecs.PropagatedTagSource.SERVICE
    });

    // Allow ECS to connect to RDS
    database.connections.allowFrom(service, ec2.Port.tcp(5432));

    // Target Group
    const targetGroup = listener.addTargets('AccountServiceTarget', {
      port: 3000,
      protocol: elbv2.ApplicationProtocol.HTTP,
      targets: [service],
      healthCheck: {
        path: '/health',
        interval: cdk.Duration.seconds(30),
        timeout: cdk.Duration.seconds(5),
        healthyThresholdCount: 2,
        unhealthyThresholdCount: 3
      },
      deregistrationDelay: cdk.Duration.seconds(30)
    });

    // Auto Scaling
    const scaling = service.autoScaleTaskCount({
      minCapacity: 2,
      maxCapacity: 10
    });

    scaling.scaleOnCpuUtilization('CpuScaling', {
      targetUtilizationPercent: 70,
      scaleInCooldown: cdk.Duration.seconds(60),
      scaleOutCooldown: cdk.Duration.seconds(60)
    });

    scaling.scaleOnMemoryUtilization('MemoryScaling', {
      targetUtilizationPercent: 80
    });

    // Outputs
    new cdk.CfnOutput(this, 'LoadBalancerDNS', {
      value: alb.loadBalancerDnsName,
      description: 'Load Balancer DNS'
    });

    new cdk.CfnOutput(this, 'DatabaseEndpoint', {
      value: database.dbInstanceEndpointAddress,
      description: 'RDS Endpoint'
    });
  }
}
```

---

### 🎓 Interview Discussion Points - Q7

**Q1: Fargate vs EC2 launch type?**

**A**:
- **Fargate**: Serverless, no server management, pay per task
- **EC2**: More control, cheaper at scale, manage instances

**Q2: How do you handle secrets in containers?**

**A**: 
- AWS Secrets Manager or Parameter Store
- ECS injects secrets as environment variables
- Never hardcode in Dockerfile or image

**Q3: What is graceful shutdown and why important?**

**A**: Finish processing active requests before terminating. Prevents data loss and improves user experience. Handle SIGTERM signal properly.

---

## Question 8: NestJS + Kubernetes (EKS) Deployment

### 📋 Question Statement

Design and deploy a production-grade banking payment service using NestJS framework on Amazon EKS (Kubernetes). Include:
- NestJS application with dependency injection
- Kubernetes manifests (Deployment, Service, Ingress, ConfigMap, Secret)
- Helm charts for easy deployment
- Horizontal Pod Autoscaler (HPA)
- Service mesh integration (AWS App Mesh)
- Observability (Prometheus, Grafana)
- CI/CD pipeline with GitOps (ArgoCD)

---

### ☸️ Kubernetes Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    AWS EKS CLUSTER                           │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              Ingress Controller (ALB)               │    │
│  │  • SSL Termination                                 │    │
│  │  • Path-based Routing                              │    │
│  └────────────────────┬───────────────────────────────┘    │
│                       │                                     │
│         ┌─────────────┴─────────────┐                      │
│         │                           │                      │
│  ┌──────▼──────┐           ┌───────▼──────┐              │
│  │  Payment    │           │  Payment     │              │
│  │  Service    │           │  Service     │              │
│  │  Pod 1      │           │  Pod 2       │              │
│  │             │           │              │              │
│  │ ┌─────────┐ │           │ ┌─────────┐  │              │
│  │ │ NestJS  │ │           │ │ NestJS  │  │              │
│  │ │Container│ │           │ │Container│  │              │
│  │ └─────────┘ │           │ └─────────┘  │              │
│  └──────┬──────┘           └───────┬──────┘              │
│         │                          │                      │
│         └──────────┬───────────────┘                      │
│                    │                                      │
│         ┌──────────▼──────────┐                           │
│         │  Aurora Serverless  │                           │
│         │     PostgreSQL      │                           │
│         └─────────────────────┘                           │
│                                                            │
│  ┌────────────────────────────────────────────────────┐   │
│  │         Supporting Services                         │   │
│  │  • Prometheus (Metrics)                            │   │
│  │  • Grafana (Visualization)                         │   │
│  │  • Fluentd (Logging)                               │   │
│  │  • AWS App Mesh (Service Mesh)                     │   │
│  └────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

### 🏗️ NestJS Payment Service

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';
import * as helmet from 'helmet';
import * as compression from 'compression';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Security
  app.use(helmet());
  
  // Compression
  app.use(compression());

  // CORS
  app.enableCors({
    origin: process.env.ALLOWED_ORIGINS?.split(',') || '*',
    credentials: true
  });

  // Global validation pipe
  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true
  }));

  // Swagger API Documentation
  const config = new DocumentBuilder()
    .setTitle('Payment Service API')
    .setDescription('Emirates NBD Payment Service')
    .setVersion('1.0')
    .addBearerAuth()
    .build();
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api/docs', app, document);

  // Health check endpoint
  app.getUrl = () => `http://localhost:${process.env.PORT || 3000}`;

  // Graceful shutdown
  app.enableShutdownHooks();

  const port = process.env.PORT || 3000;
  await app.listen(port, '0.0.0.0');
  console.log(`Payment Service running on port ${port}`);
}

bootstrap();
```

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';
import { HealthModule } from './health/health.module';
import { PaymentModule } from './payment/payment.module';
import { EventModule } from './event/event.module';

@Module({
  imports: [
    // Configuration
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: `.env.${process.env.NODE_ENV || 'development'}`
    }),

    // Database
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: process.env.DB_HOST,
      port: parseInt(process.env.DB_PORT, 10) || 5432,
      username: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      database: process.env.DB_NAME,
      entities: [__dirname + '/**/*.entity{.ts,.js}'],
      synchronize: false, // Never use in production
      logging: process.env.NODE_ENV === 'development',
      ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false,
      extra: {
        max: 20,
        idleTimeoutMillis: 30000,
        connectionTimeoutMillis: 2000
      }
    }),

    // Modules
    HealthModule,
    PaymentModule,
    EventModule
  ]
})
export class AppModule {}
```

```typescript
// src/payment/payment.entity.ts
import { Entity, Column, PrimaryGeneratedColumn, CreateDateColumn, UpdateDateColumn, Index } from 'typeorm';

@Entity('payments')
@Index(['merchantId', 'status'])
@Index(['customerId', 'createdAt'])
export class Payment {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'uuid' })
  customerId: string;

  @Column({ type: 'uuid' })
  merchantId: string;

  @Column({ type: 'decimal', precision: 15, scale: 2 })
  amount: number;

  @Column({ type: 'varchar', length: 3, default: 'AED' })
  currency: string;

  @Column({ type: 'varchar', length: 50 })
  status: string; // PENDING, AUTHORIZED, CAPTURED, FAILED, REFUNDED

  @Column({ type: 'varchar', length: 100, nullable: true })
  paymentMethod: string;

  @Column({ type: 'varchar', length: 100, nullable: true })
  transactionId: string;

  @Column({ type: 'text', nullable: true })
  description: string;

  @Column({ type: 'jsonb', nullable: true })
  metadata: object;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

```typescript
// src/payment/payment.service.ts
import { Injectable, NotFoundException, BadRequestException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Payment } from './payment.entity';
import { CreatePaymentDto } from './dto/create-payment.dto';
import { EventService } from '../event/event.service';

@Injectable()
export class PaymentService {
  constructor(
    @InjectRepository(Payment)
    private paymentRepository: Repository<Payment>,
    private eventService: EventService
  ) {}

  async createPayment(createPaymentDto: CreatePaymentDto): Promise<Payment> {
    // Validate amount
    if (createPaymentDto.amount <= 0) {
      throw new BadRequestException('Amount must be positive');
    }

    // Create payment record
    const payment = this.paymentRepository.create({
      ...createPaymentDto,
      status: 'PENDING',
      transactionId: `TXN-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`
    });

    // Save to database
    const savedPayment = await this.paymentRepository.save(payment);

    // Publish event
    await this.eventService.publishPaymentCreated(savedPayment);

    return savedPayment;
  }

  async authorizePayment(paymentId: string): Promise<Payment> {
    const payment = await this.paymentRepository.findOne({ where: { id: paymentId } });

    if (!payment) {
      throw new NotFoundException('Payment not found');
    }

    if (payment.status !== 'PENDING') {
      throw new BadRequestException(`Cannot authorize payment in status: ${payment.status}`);
    }

    // Call payment gateway (simplified)
    const authorized = await this.callPaymentGateway(payment);

    if (authorized) {
      payment.status = 'AUTHORIZED';
      await this.paymentRepository.save(payment);
      await this.eventService.publishPaymentAuthorized(payment);
    } else {
      payment.status = 'FAILED';
      await this.paymentRepository.save(payment);
      await this.eventService.publishPaymentFailed(payment);
    }

    return payment;
  }

  async capturePayment(paymentId: string): Promise<Payment> {
    const payment = await this.paymentRepository.findOne({ where: { id: paymentId } });

    if (!payment) {
      throw new NotFoundException('Payment not found');
    }

    if (payment.status !== 'AUTHORIZED') {
      throw new BadRequestException('Payment must be authorized before capture');
    }

    payment.status = 'CAPTURED';
    const capturedPayment = await this.paymentRepository.save(payment);

    await this.eventService.publishPaymentCaptured(capturedPayment);

    return capturedPayment;
  }

  async refundPayment(paymentId: string, amount?: number): Promise<Payment> {
    const payment = await this.paymentRepository.findOne({ where: { id: paymentId } });

    if (!payment) {
      throw new NotFoundException('Payment not found');
    }

    if (payment.status !== 'CAPTURED') {
      throw new BadRequestException('Can only refund captured payments');
    }

    const refundAmount = amount || payment.amount;

    if (refundAmount > payment.amount) {
      throw new BadRequestException('Refund amount exceeds payment amount');
    }

    payment.status = 'REFUNDED';
    payment.metadata = { ...payment.metadata, refundAmount };
    const refundedPayment = await this.paymentRepository.save(payment);

    await this.eventService.publishPaymentRefunded(refundedPayment);

    return refundedPayment;
  }

  async findPaymentById(id: string): Promise<Payment> {
    const payment = await this.paymentRepository.findOne({ where: { id } });

    if (!payment) {
      throw new NotFoundException('Payment not found');
    }

    return payment;
  }

  async findPaymentsByCustomer(customerId: string): Promise<Payment[]> {
    return this.paymentRepository.find({
      where: { customerId },
      order: { createdAt: 'DESC' }
    });
  }

  private async callPaymentGateway(payment: Payment): Promise<boolean> {
    // Simulate payment gateway call
    console.log('Calling payment gateway for:', payment.id);
    
    // In real implementation, call external payment gateway API
    // For demo, authorize 90% of payments
    return Math.random() > 0.1;
  }
}
```

```typescript
// src/payment/payment.controller.ts
import { Controller, Post, Get, Param, Body, UseGuards } from '@nestjs/common';
import { ApiTags, ApiOperation, ApiBearerAuth } from '@nestjs/swagger';
import { PaymentService } from './payment.service';
import { CreatePaymentDto } from './dto/create-payment.dto';

@ApiTags('payments')
@Controller('payments')
export class PaymentController {
  constructor(private readonly paymentService: PaymentService) {}

  @Post()
  @ApiOperation({ summary: 'Create a new payment' })
  async createPayment(@Body() createPaymentDto: CreatePaymentDto) {
    return this.paymentService.createPayment(createPaymentDto);
  }

  @Post(':id/authorize')
  @ApiOperation({ summary: 'Authorize a payment' })
  async authorizePayment(@Param('id') id: string) {
    return this.paymentService.authorizePayment(id);
  }

  @Post(':id/capture')
  @ApiOperation({ summary: 'Capture an authorized payment' })
  async capturePayment(@Param('id') id: string) {
    return this.paymentService.capturePayment(id);
  }

  @Post(':id/refund')
  @ApiOperation({ summary: 'Refund a captured payment' })
  async refundPayment(@Param('id') id: string) {
    return this.paymentService.refundPayment(id);
  }

  @Get(':id')
  @ApiOperation({ summary: 'Get payment by ID' })
  async getPayment(@Param('id') id: string) {
    return this.paymentService.findPaymentById(id);
  }

  @Get('customer/:customerId')
  @ApiOperation({ summary: 'Get payments by customer' })
  async getCustomerPayments(@Param('customerId') customerId: string) {
    return this.paymentService.findPaymentsByCustomer(customerId);
  }
}
```

```typescript
// src/health/health.controller.ts
import { Controller, Get } from '@nestjs/common';
import { HealthCheck, HealthCheckService, TypeOrmHealthIndicator } from '@nestjs/terminus';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck('database')
    ]);
  }

  @Get('ready')
  ready() {
    return { status: 'ready', timestamp: new Date().toISOString() };
  }

  @Get('live')
  live() {
    return { status: 'alive', timestamp: new Date().toISOString() };
  }
}
```

---

### 🐳 Dockerfile for NestJS

```dockerfile
# Multi-stage Dockerfile for NestJS

# Stage 1: Builder
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
COPY tsconfig*.json ./

# Install dependencies
RUN npm ci

# Copy source code
COPY src ./src

# Build application
RUN npm run build

# Stage 2: Production
FROM node:20-alpine

# Install dumb-init
RUN apk add --no-cache dumb-init

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Copy built application
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/package*.json ./

# Environment
ENV NODE_ENV=production \
    PORT=3000

EXPOSE 3000

USER nodejs

HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

ENTRYPOINT ["dumb-init", "--"]

CMD ["node", "dist/main"]
```

---

### ☸️ Kubernetes Manifests

**Namespace**:
```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: banking
  labels:
    name: banking
    environment: production
```

**ConfigMap**:
```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: payment-service-config
  namespace: banking
data:
  NODE_ENV: "production"
  PORT: "3000"
  DB_HOST: "payment-db.banking.svc.cluster.local"
  DB_PORT: "5432"
  DB_NAME: "payment_db"
  LOG_LEVEL: "info"
  ALLOWED_ORIGINS: "https://app.emiratesnbd.com"
```

**Secret**:
```yaml
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: payment-service-secret
  namespace: banking
type: Opaque
stringData:
  DB_USER: "admin"
  DB_PASSWORD: "change-me-in-production"
  JWT_SECRET: "super-secret-jwt-key"
```

**Deployment**:
```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: banking
  labels:
    app: payment-service
    version: v1
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: payment-service-sa
      
      # Init container - wait for database
      initContainers:
      - name: wait-for-db
        image: postgres:15-alpine
        command:
        - sh
        - -c
        - |
          until pg_isready -h $DB_HOST -p $DB_PORT -U $DB_USER; do
            echo "Waiting for database..."
            sleep 2
          done
        envFrom:
        - configMapRef:
            name: payment-service-config
        - secretRef:
            name: payment-service-secret

      containers:
      - name: payment-service
        image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/payment-service:latest
        imagePullPolicy: Always
        
        ports:
        - name: http
          containerPort: 3000
          protocol: TCP
        
        envFrom:
        - configMapRef:
            name: payment-service-config
        - secretRef:
            name: payment-service-secret
        
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        livenessProbe:
          httpGet:
            path: /health/live
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
        
        securityContext:
          runAsNonRoot: true
          runAsUser: 1001
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
```

**Service**:
```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: banking
  labels:
    app: payment-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: ClusterIP
  selector:
    app: payment-service
  ports:
  - name: http
    port: 80
    targetPort: 3000
    protocol: TCP
  sessionAffinity: None
```

**Ingress**:
```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: payment-service-ingress
  namespace: banking
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123456789012:certificate/xxx
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-2017-01
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig": {"Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'
spec:
  rules:
  - host: api.emiratesnbd.com
    http:
      paths:
      - path: /payments
        pathType: Prefix
        backend:
          service:
            name: payment-service
            port:
              number: 80
```

**HorizontalPodAutoscaler**:
```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-service-hpa
  namespace: banking
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
      - type: Pods
        value: 4
        periodSeconds: 30
      selectPolicy: Max
```

---

### 📦 Helm Chart

```yaml
# helm/payment-service/Chart.yaml
apiVersion: v2
name: payment-service
description: Emirates NBD Payment Service
type: application
version: 1.0.0
appVersion: "1.0.0"
keywords:
  - banking
  - payment
  - nestjs
maintainers:
  - name: DevOps Team
    email: devops@emiratesnbd.com
```

```yaml
# helm/payment-service/values.yaml
replicaCount: 3

image:
  repository: 123456789012.dkr.ecr.us-east-1.amazonaws.com/payment-service
  pullPolicy: Always
  tag: "latest"

service:
  type: ClusterIP
  port: 80
  targetPort: 3000

ingress:
  enabled: true
  className: alb
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
  hosts:
    - host: api.emiratesnbd.com
      paths:
        - path: /payments
          pathType: Prefix

resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

env:
  NODE_ENV: production
  PORT: "3000"
  DB_HOST: payment-db.banking.svc.cluster.local
  DB_PORT: "5432"
  DB_NAME: payment_db

secrets:
  DB_USER: admin
  DB_PASSWORD: change-me
```

```yaml
# helm/payment-service/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "payment-service.fullname" . }}
  labels:
    {{- include "payment-service.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "payment-service.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "payment-service.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.service.targetPort }}
        env:
        {{- range $key, $value := .Values.env }}
        - name: {{ $key }}
          value: {{ $value | quote }}
        {{- end }}
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
```

---

### 🎓 Interview Discussion Points - Q8

**Q1: Kubernetes vs ECS - when to use which?**

**A**:
- **Kubernetes (EKS)**: Multi-cloud, complex orchestration, large scale, strong ecosystem
- **ECS**: AWS-native, simpler, tight AWS integration, lower learning curve

**Q2: What is HPA and how does it work?**

**A**: Horizontal Pod Autoscaler automatically scales pods based on CPU, memory, or custom metrics. Queries metrics every 15 seconds and adjusts replica count.

**Q3: What are init containers used for?**

**A**: Run before main containers, perform setup tasks like waiting for dependencies, database migrations, or configuration loading.

---

**End of File 4**

