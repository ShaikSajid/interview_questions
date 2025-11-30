# Q60: Production Scalability Strategies

## 📋 Summary
This question covers **production scalability strategies** for Node.js applications, including horizontal vs vertical scaling, load balancing, database sharding, caching strategies, and microservices architecture to handle millions of concurrent users in banking systems.

**Key Topics**:
- Horizontal vs vertical scaling
- Load balancing strategies (NGINX, HAProxy, AWS ELB)
- Database scaling (replication, sharding, read replicas)
- Caching layers (Redis, CDN)
- Message queues for decoupling
- Microservices architecture
- Auto-scaling and elasticity
- Performance optimization
- Monitoring and capacity planning
- Cost optimization

**Banking Use Cases**:
- Scaling payment processing to millions TPS
- Handling peak loads (Black Friday, paydays)
- Geographic distribution
- Real-time transaction processing
- High availability (99.99% uptime)
- Data consistency across regions
- Regulatory compliance at scale
- Multi-tenant architecture

---

## 🎯 Scalability Fundamentals

### Horizontal vs Vertical Scaling

```
┌─────────────────────────────────────────────────────────────┐
│           Vertical Scaling (Scale Up)                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Before:                    After:                         │
│   ┌──────────┐              ┌──────────┐                   │
│   │  Server  │              │  Server  │                   │
│   │ 4 CPUs   │     →        │ 16 CPUs  │                   │
│   │ 16GB RAM │              │ 64GB RAM │                   │
│   └──────────┘              └──────────┘                   │
│                                                              │
│   Pros:                      Cons:                          │
│   • Simple to implement      • Hardware limits             │
│   • No code changes          • Single point of failure     │
│   • Data consistency         • Downtime for upgrades       │
│                              • Expensive at scale          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│           Horizontal Scaling (Scale Out)                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Before:                    After:                         │
│   ┌──────────┐              ┌──────────┐                   │
│   │  Server  │              │  Server  │                   │
│   │ 4 CPUs   │              │ 4 CPUs   │                   │
│   │ 16GB RAM │              │ 16GB RAM │                   │
│   └──────────┘              └──────────┘                   │
│                              ┌──────────┐                   │
│                              │  Server  │                   │
│                      →       │ 4 CPUs   │                   │
│                              │ 16GB RAM │                   │
│                              └──────────┘                   │
│                              ┌──────────┐                   │
│                              │  Server  │                   │
│                              │ 4 CPUs   │                   │
│                              │ 16GB RAM │                   │
│                              └──────────┘                   │
│                                                              │
│   Pros:                      Cons:                          │
│   • No hardware limits       • More complex                │
│   • High availability        • Data consistency issues     │
│   • Cost-effective          • Load balancing needed        │
│   • Fault tolerance         • Session management           │
└─────────────────────────────────────────────────────────────┘
```

### Scalability Patterns

```
Performance at Scale
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Strategy              │ Complexity │ Cost  │ Max Scale │ Use Case
━━━━━━━━━━━━━━━━━━━━━━┼━━━━━━━━━━━━┼━━━━━━━┼━━━━━━━━━━━┼━━━━━━━━━━
Vertical Scaling      │ ⭐         │ $$$   │ ~1M RPM   │ MVP, small apps
Horizontal Scaling    │ ⭐⭐⭐      │ $$    │ Unlimited │ Production
Caching              │ ⭐⭐        │ $     │ 10x boost │ Read-heavy
Load Balancing       │ ⭐⭐        │ $     │ Unlimited │ High traffic
Database Sharding    │ ⭐⭐⭐⭐     │ $$    │ Unlimited │ Massive data
Microservices        │ ⭐⭐⭐⭐⭐    │ $$$   │ Unlimited │ Large teams
CDN                  │ ⭐         │ $     │ Global    │ Static content
Message Queues       │ ⭐⭐⭐      │ $$    │ Unlimited │ Async processing
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 💡 Example 1: Complete Load Balancing Setup

### 1. NGINX Load Balancer Configuration

```nginx
# /etc/nginx/nginx.conf

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging format
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time uct="$upstream_connect_time" '
                    'uht="$upstream_header_time" urt="$upstream_response_time"';

    access_log /var/log/nginx/access.log main;

    # Performance tuning
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    keepalive_requests 100;
    reset_timedout_connection on;
    client_body_timeout 10;
    send_timeout 2;
    
    # Buffer sizes
    client_body_buffer_size 128k;
    client_max_body_size 10m;
    client_header_buffer_size 1k;
    large_client_header_buffers 4 4k;
    output_buffers 1 32k;
    postpone_output 1460;

    # Gzip compression
    gzip on;
    gzip_min_length 1000;
    gzip_types text/plain text/css application/json application/javascript 
               text/xml application/xml application/xml+rss text/javascript;
    gzip_comp_level 6;
    gzip_vary on;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=login_limit:10m rate=5r/m;
    limit_conn_zone $binary_remote_addr zone=addr:10m;

    # Upstream servers (Node.js backends)
    upstream banking_api {
        least_conn;  # Load balancing algorithm
        
        server 10.0.1.10:3000 weight=1 max_fails=3 fail_timeout=30s;
        server 10.0.1.11:3000 weight=1 max_fails=3 fail_timeout=30s;
        server 10.0.1.12:3000 weight=1 max_fails=3 fail_timeout=30s;
        server 10.0.1.13:3000 weight=1 max_fails=3 fail_timeout=30s;
        
        # Backup server
        server 10.0.1.20:3000 backup;
        
        # Keepalive connections to backends
        keepalive 32;
        keepalive_timeout 60s;
        keepalive_requests 100;
    }

    # HTTP to HTTPS redirect
    server {
        listen 80;
        server_name api.banking.com;
        return 301 https://$server_name$request_uri;
    }

    # Main HTTPS server
    server {
        listen 443 ssl http2;
        server_name api.banking.com;

        # SSL configuration
        ssl_certificate /etc/nginx/ssl/api.banking.com.crt;
        ssl_certificate_key /etc/nginx/ssl/api.banking.com.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 10m;

        # Security headers
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;

        # Health check endpoint (bypass rate limiting)
        location /health {
            proxy_pass http://banking_api;
            access_log off;
        }

        # API endpoints with rate limiting
        location /api/ {
            # Rate limiting
            limit_req zone=api_limit burst=20 nodelay;
            limit_conn addr 10;

            # Proxy to Node.js backends
            proxy_pass http://banking_api;
            proxy_http_version 1.1;
            
            # Headers
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Connection "";
            
            # Timeouts
            proxy_connect_timeout 5s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
            
            # Buffering
            proxy_buffering on;
            proxy_buffer_size 4k;
            proxy_buffers 8 4k;
            proxy_busy_buffers_size 8k;
            
            # Retries
            proxy_next_upstream error timeout http_502 http_503 http_504;
            proxy_next_upstream_tries 2;
        }

        # Login endpoint with stricter rate limiting
        location /api/v1/auth/login {
            limit_req zone=login_limit burst=3 nodelay;
            
            proxy_pass http://banking_api;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # Static assets (with caching)
        location /static/ {
            alias /var/www/static/;
            expires 1y;
            add_header Cache-Control "public, immutable";
            access_log off;
        }

        # Metrics endpoint (internal only)
        location /metrics {
            allow 10.0.0.0/8;
            deny all;
            proxy_pass http://banking_api;
        }
    }
}
```

### 2. Node.js Application with Session Management

```javascript
// src/server.js - Session management for horizontal scaling

const express = require('express');
const session = require('express-session');
const RedisStore = require('connect-redis').default;
const { createClient } = require('redis');
const { logger } = require('./utils/logger');

const app = express();

/**
 * Setup session store for horizontal scaling
 */
async function setupSessionStore() {
  // Redis client for sessions
  const redisClient = createClient({
    socket: {
      host: process.env.REDIS_HOST || 'localhost',
      port: parseInt(process.env.REDIS_PORT || '6379'),
    },
    password: process.env.REDIS_PASSWORD,
  });

  redisClient.on('error', (err) => {
    logger.error('Redis session store error', { error: err.message });
  });

  await redisClient.connect();
  logger.info('Redis session store connected');

  // Session middleware with Redis store
  app.use(session({
    store: new RedisStore({ 
      client: redisClient,
      prefix: 'sess:',
    }),
    secret: process.env.SESSION_SECRET || 'super-secret-key',
    resave: false,
    saveUninitialized: false,
    rolling: true,
    cookie: {
      secure: process.env.NODE_ENV === 'production',
      httpOnly: true,
      maxAge: 1000 * 60 * 60 * 24, // 24 hours
      sameSite: 'strict',
    },
    name: 'sessionId',
  }));

  return redisClient;
}

/**
 * Sticky session alternative: JWT tokens
 */
const jwt = require('jsonwebtoken');

function generateToken(userId, payload = {}) {
  return jwt.sign(
    {
      userId,
      ...payload,
    },
    process.env.JWT_SECRET,
    {
      expiresIn: '24h',
      issuer: 'banking-api',
      audience: 'banking-app',
    }
  );
}

function verifyToken(token) {
  try {
    return jwt.verify(token, process.env.JWT_SECRET, {
      issuer: 'banking-api',
      audience: 'banking-app',
    });
  } catch (error) {
    return null;
  }
}

/**
 * Authentication middleware
 */
function authenticate(req, res, next) {
  const token = req.headers.authorization?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }

  const decoded = verifyToken(token);
  
  if (!decoded) {
    return res.status(401).json({ error: 'Invalid token' });
  }

  req.user = decoded;
  next();
}

module.exports = {
  app,
  setupSessionStore,
  generateToken,
  verifyToken,
  authenticate,
};
```

### 3. Auto-Scaling Configuration (AWS)

```yaml
# aws/autoscaling-config.yml

AWSTemplateFormatVersion: '2010-09-09'
Description: 'Auto-scaling configuration for Banking API'

Parameters:
  VPC:
    Type: AWS::EC2::VPC::Id
    Description: VPC for the application
  
  Subnets:
    Type: List<AWS::EC2::Subnet::Id>
    Description: Subnets for the instances
  
  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
    Description: EC2 Key Pair for SSH access

Resources:
  # Launch Template
  LaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateName: banking-api-template
      LaunchTemplateData:
        ImageId: ami-0c55b159cbfafe1f0  # Amazon Linux 2
        InstanceType: t3.medium
        KeyName: !Ref KeyName
        
        IamInstanceProfile:
          Arn: !GetAtt InstanceProfile.Arn
        
        SecurityGroupIds:
          - !Ref InstanceSecurityGroup
        
        UserData:
          Fn::Base64: !Sub |
            #!/bin/bash
            yum update -y
            yum install -y nodejs npm git
            
            # Install PM2
            npm install -g pm2
            
            # Clone application
            cd /home/ec2-user
            git clone https://github.com/company/banking-api.git
            cd banking-api
            
            # Install dependencies
            npm ci --production
            
            # Start application
            pm2 start ecosystem.config.js --env production
            pm2 save
            pm2 startup systemd -u ec2-user --hp /home/ec2-user
        
        TagSpecifications:
          - ResourceType: instance
            Tags:
              - Key: Name
                Value: banking-api-instance
              - Key: Environment
                Value: production

  # Auto Scaling Group
  AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      AutoScalingGroupName: banking-api-asg
      VPCZoneIdentifier: !Ref Subnets
      LaunchTemplate:
        LaunchTemplateId: !Ref LaunchTemplate
        Version: !GetAtt LaunchTemplate.LatestVersionNumber
      
      MinSize: 2
      MaxSize: 10
      DesiredCapacity: 4
      HealthCheckType: ELB
      HealthCheckGracePeriod: 300
      
      TargetGroupARNs:
        - !Ref TargetGroup
      
      Tags:
        - Key: Name
          Value: banking-api-instance
          PropagateAtLaunch: true

  # Scaling Policies
  ScaleUpPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AutoScalingGroupName: !Ref AutoScalingGroup
      PolicyType: TargetTrackingScaling
      TargetTrackingConfiguration:
        PredefinedMetricSpecification:
          PredefinedMetricType: ASGAverageCPUUtilization
        TargetValue: 70.0

  ScaleDownPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AutoScalingGroupName: !Ref AutoScalingGroup
      PolicyType: TargetTrackingScaling
      TargetTrackingConfiguration:
        PredefinedMetricSpecification:
          PredefinedMetricType: ASGAverageCPUUtilization
        TargetValue: 70.0
        ScaleInCooldown: 300

  # CloudWatch Alarms
  HighCPUAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: banking-api-high-cpu
      AlarmDescription: Alert when CPU exceeds 80%
      MetricName: CPUUtilization
      Namespace: AWS/EC2
      Statistic: Average
      Period: 300
      EvaluationPeriods: 2
      Threshold: 80
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: AutoScalingGroupName
          Value: !Ref AutoScalingGroup

  # Application Load Balancer
  LoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: banking-api-alb
      Type: application
      Scheme: internet-facing
      SecurityGroups:
        - !Ref LoadBalancerSecurityGroup
      Subnets: !Ref Subnets
      Tags:
        - Key: Name
          Value: banking-api-alb

  # Target Group
  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: banking-api-tg
      Port: 3000
      Protocol: HTTP
      VpcId: !Ref VPC
      HealthCheckEnabled: true
      HealthCheckPath: /health/ready
      HealthCheckIntervalSeconds: 30
      HealthCheckTimeoutSeconds: 5
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 3
      TargetGroupAttributes:
        - Key: deregistration_delay.timeout_seconds
          Value: '30'
        - Key: stickiness.enabled
          Value: 'false'

  # Listener
  Listener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref LoadBalancer
      Port: 443
      Protocol: HTTPS
      SslPolicy: ELBSecurityPolicy-TLS-1-2-2017-01
      Certificates:
        - CertificateArn: !Ref SSLCertificate
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref TargetGroup

  # Security Groups
  LoadBalancerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: banking-api-alb-sg
      GroupDescription: Security group for ALB
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  InstanceSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: banking-api-instance-sg
      GroupDescription: Security group for instances
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3000
          ToPort: 3000
          SourceSecurityGroupId: !Ref LoadBalancerSecurityGroup
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 10.0.0.0/8  # Internal only

Outputs:
  LoadBalancerDNS:
    Description: DNS name of the load balancer
    Value: !GetAtt LoadBalancer.DNSName
```

---

## 💡 Example 2: Database Scaling Strategies

### 1. Read Replica Configuration

```javascript
// src/database/replication.js

const { Pool } = require('pg');
const { logger } = require('../utils/logger');

/**
 * Database connection manager with read replicas
 */
class DatabaseManager {
  constructor() {
    this.masterPool = null;
    this.replicaPools = [];
    this.currentReplicaIndex = 0;
  }

  /**
   * Initialize master and replica connections
   */
  async initialize() {
    // Master database (writes)
    this.masterPool = new Pool({
      host: process.env.DB_MASTER_HOST,
      port: parseInt(process.env.DB_PORT || '5432'),
      database: process.env.DB_NAME,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      max: 20,
      min: 5,
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 5000,
    });

    logger.info('Master database pool initialized');

    // Read replicas
    const replicaHosts = (process.env.DB_REPLICA_HOSTS || '').split(',');
    
    for (const host of replicaHosts) {
      if (!host) continue;

      const pool = new Pool({
        host: host.trim(),
        port: parseInt(process.env.DB_PORT || '5432'),
        database: process.env.DB_NAME,
        user: process.env.DB_USER,
        password: process.env.DB_PASSWORD,
        max: 20,
        min: 5,
        idleTimeoutMillis: 30000,
        connectionTimeoutMillis: 5000,
      });

      this.replicaPools.push(pool);
      logger.info(`Read replica pool initialized`, { host });
    }

    // Test connections
    await this.testConnections();
  }

  /**
   * Test all database connections
   */
  async testConnections() {
    // Test master
    const masterClient = await this.masterPool.connect();
    await masterClient.query('SELECT 1');
    masterClient.release();
    logger.info('Master database connection verified');

    // Test replicas
    for (let i = 0; i < this.replicaPools.length; i++) {
      const client = await this.replicaPools[i].connect();
      await client.query('SELECT 1');
      client.release();
      logger.info(`Replica ${i + 1} connection verified`);
    }
  }

  /**
   * Get pool for write operations (master)
   */
  getWritePool() {
    return this.masterPool;
  }

  /**
   * Get pool for read operations (round-robin replica selection)
   */
  getReadPool() {
    // If no replicas, use master
    if (this.replicaPools.length === 0) {
      return this.masterPool;
    }

    // Round-robin selection
    const pool = this.replicaPools[this.currentReplicaIndex];
    this.currentReplicaIndex = (this.currentReplicaIndex + 1) % this.replicaPools.length;
    
    return pool;
  }

  /**
   * Execute write query (master)
   */
  async write(query, params = []) {
    const start = Date.now();
    
    try {
      const result = await this.masterPool.query(query, params);
      const duration = Date.now() - start;
      
      logger.debug('Write query executed', { duration, rows: result.rowCount });
      
      return result;
    } catch (error) {
      logger.error('Write query failed', {
        error: error.message,
        query: query.substring(0, 100),
      });
      throw error;
    }
  }

  /**
   * Execute read query (replica with fallback to master)
   */
  async read(query, params = []) {
    const start = Date.now();
    const pool = this.getReadPool();
    
    try {
      const result = await pool.query(query, params);
      const duration = Date.now() - start;
      
      logger.debug('Read query executed', { duration, rows: result.rowCount });
      
      return result;
    } catch (error) {
      logger.warn('Read query failed on replica, falling back to master', {
        error: error.message,
      });
      
      // Fallback to master
      return this.masterPool.query(query, params);
    }
  }

  /**
   * Execute transaction (always on master)
   */
  async transaction(callback) {
    const client = await this.masterPool.connect();
    
    try {
      await client.query('BEGIN');
      const result = await callback(client);
      await client.query('COMMIT');
      return result;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  /**
   * Close all connections
   */
  async close() {
    await this.masterPool.end();
    logger.info('Master database pool closed');

    for (const pool of this.replicaPools) {
      await pool.end();
    }
    logger.info('All replica pools closed');
  }
}

// Singleton instance
const dbManager = new DatabaseManager();

module.exports = dbManager;
```

### 2. Database Sharding Strategy

```javascript
// src/database/sharding.js

const { Pool } = require('pg');
const crypto = require('crypto');
const { logger } = require('../utils/logger');

/**
 * Database sharding manager for horizontal partitioning
 */
class ShardingManager {
  constructor() {
    this.shards = [];
    this.shardCount = 0;
  }

  /**
   * Initialize database shards
   */
  async initialize() {
    const shardConfigs = [
      { id: 0, host: process.env.DB_SHARD_0_HOST, range: [0, 255] },
      { id: 1, host: process.env.DB_SHARD_1_HOST, range: [256, 511] },
      { id: 2, host: process.env.DB_SHARD_2_HOST, range: [512, 767] },
      { id: 3, host: process.env.DB_SHARD_3_HOST, range: [768, 1023] },
    ];

    for (const config of shardConfigs) {
      const pool = new Pool({
        host: config.host,
        port: parseInt(process.env.DB_PORT || '5432'),
        database: process.env.DB_NAME,
        user: process.env.DB_USER,
        password: process.env.DB_PASSWORD,
        max: 20,
      });

      this.shards.push({
        id: config.id,
        pool,
        range: config.range,
      });

      logger.info(`Shard ${config.id} initialized`, { host: config.host });
    }

    this.shardCount = this.shards.length;
  }

  /**
   * Calculate shard ID from user ID
   */
  getShardId(userId) {
    // Hash-based sharding
    const hash = crypto.createHash('md5').update(userId.toString()).digest();
    const hashValue = hash.readUInt16BE(0); // 0-65535
    
    // Map to shard range (0-1023)
    const normalizedHash = Math.floor((hashValue / 65536) * 1024);
    
    // Find shard
    for (const shard of this.shards) {
      if (normalizedHash >= shard.range[0] && normalizedHash <= shard.range[1]) {
        return shard.id;
      }
    }

    return 0; // Default to first shard
  }

  /**
   * Get shard pool for a user
   */
  getShardPool(userId) {
    const shardId = this.getShardId(userId);
    return this.shards[shardId].pool;
  }

  /**
   * Execute query on specific shard
   */
  async queryUserShard(userId, query, params = []) {
    const pool = this.getShardPool(userId);
    
    try {
      return await pool.query(query, params);
    } catch (error) {
      logger.error('Shard query failed', {
        userId,
        error: error.message,
      });
      throw error;
    }
  }

  /**
   * Execute query across all shards (scatter-gather)
   */
  async queryAllShards(query, params = []) {
    const promises = this.shards.map(shard =>
      shard.pool.query(query, params).catch(error => {
        logger.error(`Query failed on shard ${shard.id}`, { error: error.message });
        return { rows: [], error };
      })
    );

    const results = await Promise.all(promises);
    
    // Combine results
    const rows = results.flatMap(result => result.rows || []);
    
    return { rows };
  }

  /**
   * Close all shard connections
   */
  async close() {
    for (const shard of this.shards) {
      await shard.pool.end();
      logger.info(`Shard ${shard.id} closed`);
    }
  }
}

module.exports = new ShardingManager();
```

### 3. Caching Strategy

```javascript
// src/cache/strategy.js

const Redis = require('ioredis');
const { logger } = require('../utils/logger');

/**
 * Multi-layer caching strategy
 */
class CacheManager {
  constructor() {
    this.redis = null;
    this.localCache = new Map();
    this.localCacheMaxSize = 1000;
  }

  /**
   * Initialize Redis connection
   */
  async initialize() {
    this.redis = new Redis({
      host: process.env.REDIS_HOST || 'localhost',
      port: parseInt(process.env.REDIS_PORT || '6379'),
      password: process.env.REDIS_PASSWORD,
      db: 0,
      retryStrategy: (times) => {
        const delay = Math.min(times * 50, 2000);
        return delay;
      },
      maxRetriesPerRequest: 3,
    });

    this.redis.on('error', (err) => {
      logger.error('Redis error', { error: err.message });
    });

    this.redis.on('connect', () => {
      logger.info('Redis connected');
    });

    // Test connection
    await this.redis.ping();
    logger.info('Redis cache manager initialized');
  }

  /**
   * Get value from cache (L1 local → L2 Redis)
   */
  async get(key) {
    // L1: Local cache (in-memory)
    if (this.localCache.has(key)) {
      logger.debug('Cache hit (local)', { key });
      return this.localCache.get(key);
    }

    // L2: Redis cache
    try {
      const value = await this.redis.get(key);
      
      if (value) {
        logger.debug('Cache hit (redis)', { key });
        
        // Store in local cache
        this.setLocal(key, JSON.parse(value));
        
        return JSON.parse(value);
      }
    } catch (error) {
      logger.error('Redis get failed', { key, error: error.message });
    }

    logger.debug('Cache miss', { key });
    return null;
  }

  /**
   * Set value in cache
   */
  async set(key, value, ttl = 3600) {
    // Store in local cache
    this.setLocal(key, value);

    // Store in Redis
    try {
      await this.redis.setex(key, ttl, JSON.stringify(value));
      logger.debug('Cache set', { key, ttl });
    } catch (error) {
      logger.error('Redis set failed', { key, error: error.message });
    }
  }

  /**
   * Set in local cache with LRU eviction
   */
  setLocal(key, value) {
    // Evict oldest if cache is full
    if (this.localCache.size >= this.localCacheMaxSize) {
      const firstKey = this.localCache.keys().next().value;
      this.localCache.delete(firstKey);
    }

    this.localCache.set(key, value);
  }

  /**
   * Delete from cache
   */
  async delete(key) {
    this.localCache.delete(key);
    
    try {
      await this.redis.del(key);
      logger.debug('Cache deleted', { key });
    } catch (error) {
      logger.error('Redis delete failed', { key, error: error.message });
    }
  }

  /**
   * Cache-aside pattern wrapper
   */
  async getOrSet(key, fetchFn, ttl = 3600) {
    // Try cache first
    let value = await this.get(key);
    
    if (value !== null) {
      return value;
    }

    // Fetch from source
    value = await fetchFn();
    
    // Store in cache
    await this.set(key, value, ttl);
    
    return value;
  }

  /**
   * Invalidate pattern (e.g., "user:123:*")
   */
  async invalidatePattern(pattern) {
    try {
      const keys = await this.redis.keys(pattern);
      
      if (keys.length > 0) {
        await this.redis.del(...keys);
        logger.info('Cache pattern invalidated', { pattern, count: keys.length });
      }
    } catch (error) {
      logger.error('Pattern invalidation failed', { pattern, error: error.message });
    }
  }

  /**
   * Close connections
   */
  async close() {
    await this.redis.quit();
    this.localCache.clear();
    logger.info('Cache manager closed');
  }
}

module.exports = new CacheManager();
```

---

## 🎯 Microservices Architecture

```
Microservices Banking System
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌─────────────────────────────────────────────────────┐
│              API Gateway (Kong/AWS)                  │
│  • Authentication                                    │
│  • Rate limiting                                     │
│  • Request routing                                   │
└────────────┬────────────────────────────────────────┘
             │
       ┌─────┴─────┬─────────┬──────────┬──────────┐
       ▼           ▼         ▼          ▼          ▼
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│  Auth    │ │ Account  │ │Transaction│ │ Payment  │ │ Fraud    │
│ Service  │ │ Service  │ │  Service  │ │ Service  │ │Detection │
│          │ │          │ │           │ │          │ │ Service  │
│ Node.js  │ │ Node.js  │ │  Node.js  │ │ Node.js  │ │  Python  │
│ (3 inst) │ │ (5 inst) │ │ (10 inst) │ │ (8 inst) │ │ (4 inst) │
└────┬─────┘ └────┬─────┘ └─────┬─────┘ └────┬─────┘ └─────┬────┘
     │            │              │             │              │
     └────────────┴──────────────┴─────────────┴──────────────┘
                              │
                              ▼
                  ┌───────────────────────┐
                  │  Message Queue        │
                  │  (Kafka/RabbitMQ)     │
                  └───────────────────────┘
                              │
                              ▼
                  ┌───────────────────────┐
                  │  Shared Services      │
                  │  • Redis Cache        │
                  │  • PostgreSQL (Shard) │
                  │  • Elasticsearch      │
                  └───────────────────────┘
```

---

## 📚 Common Interview Questions

### Q1: Horizontal vs Vertical Scaling - when to use each?

**Answer**:
- **Vertical Scaling** (Scale Up):
  - Simple applications, early stage
  - Monolithic architecture
  - Single-tenant systems
  - When you can't modify code
  - Budget constraints (initially cheaper)

- **Horizontal Scaling** (Scale Out):
  - High traffic applications
  - Need high availability
  - Cost-effective at scale
  - Microservices architecture
  - Production systems

### Q2: How do you handle sessions in horizontally scaled apps?

**Answer**:
Three approaches:
1. **Sticky sessions**: Route user to same server (not recommended)
2. **Session store**: Redis/Memcached for shared sessions
3. **Stateless with JWT**: No server-side sessions (best)

### Q3: What's the difference between load balancing algorithms?

**Answer**:
- **Round Robin**: Equal distribution, simple
- **Least Connections**: Route to server with fewest active connections
- **IP Hash**: Same client → same server (sticky)
- **Weighted**: Route based on server capacity
- **Least Response Time**: Route to fastest server

### Q4: How do you handle database scaling?

**Answer**:
- **Vertical**: Upgrade hardware (limited)
- **Read Replicas**: Scale reads (most common)
- **Sharding**: Partition data by key (complex)
- **Caching**: Reduce database load (essential)
- **Connection Pooling**: Reuse connections

### Q5: What's the CAP theorem and how does it affect scaling?

**Answer**:
Can only have 2 of 3:
- **Consistency**: All nodes see same data
- **Availability**: System always responds
- **Partition Tolerance**: Works despite network splits

Most distributed systems choose AP (available + partition tolerant) or CP (consistent + partition tolerant).

---

## ✅ Scalability Checklist

```yaml
✅ Application Layer:
  - [ ] Stateless design
  - [ ] Horizontal scaling enabled
  - [ ] Load balancer configured
  - [ ] Session management (Redis/JWT)
  - [ ] Health checks implemented
  - [ ] Graceful shutdown
  - [ ] Auto-scaling configured

✅ Database Layer:
  - [ ] Connection pooling
  - [ ] Read replicas configured
  - [ ] Query optimization
  - [ ] Indexes on common queries
  - [ ] Sharding strategy (if needed)
  - [ ] Database monitoring

✅ Caching Strategy:
  - [ ] Redis/Memcached configured
  - [ ] Cache invalidation strategy
  - [ ] CDN for static assets
  - [ ] API response caching
  - [ ] Database query caching

✅ Message Queues:
  - [ ] Async processing with Kafka/RabbitMQ
  - [ ] Event-driven architecture
  - [ ] Worker processes
  - [ ] Dead letter queues
  - [ ] Retry mechanisms

✅ Monitoring & Observability:
  - [ ] APM tools (New Relic/DataDog)
  - [ ] Distributed tracing
  - [ ] Log aggregation (ELK)
  - [ ] Metrics collection (Prometheus)
  - [ ] Alerting configured
  - [ ] Capacity planning

✅ Infrastructure:
  - [ ] Multi-AZ deployment
  - [ ] Auto-scaling groups
  - [ ] CDN configured
  - [ ] DNS failover
  - [ ] Backup & disaster recovery
```

---

## 🎯 Capacity Planning

```
Scalability Math
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Current Metrics:
  • 1,000 requests/second
  • 50ms average response time
  • 4 instances @ 60% CPU

Target:
  • Scale to 10,000 requests/second (10x)

Calculation:
  • Current capacity: 1,000 req/s ÷ 0.6 CPU = 1,666 req/s at 100% CPU
  • Per instance: 1,666 ÷ 4 = 416 req/s per instance
  • Target instances: 10,000 ÷ 416 = 24 instances
  • With 60% target: 24 ÷ 0.6 = 40 instances

Cost Analysis:
  • Current: 4 instances × $100/month = $400/month
  • Target: 40 instances × $100/month = $4,000/month
  • Cost per 1K req/s: $400

Optimization:
  • Add caching: 50% cache hit rate → 20 instances needed
  • Optimize code: 2x faster → 20 instances needed
  • Combined: 10 instances needed → $1,000/month
```

---

**Status**: ✅ Complete with production-ready scalability strategies including load balancing, database sharding, caching, auto-scaling, and microservices architecture for banking applications at scale!
