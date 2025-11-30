# Q50: Docker Compose - Multi-Container Applications

## 📋 Summary
This question covers **Docker Compose** for orchestrating multi-container Node.js applications with databases, caching, and networking. Learn production-ready compose files for complete banking platforms with PostgreSQL, Redis, NGINX, and monitoring.

**Key Topics**:
- Docker Compose fundamentals (YAML syntax)
- Multi-container orchestration
- Service networking and communication
- Volume management and persistence
- Environment configuration
- Health checks and dependencies
- Scaling services
- Production deployment patterns

**Banking Use Cases**:
- Complete banking platform (API + DB + Cache)
- Microservices architecture
- Development environment setup
- CI/CD testing environments
- Load-balanced deployments

---

## 🎯 Understanding Docker Compose

### What is Docker Compose?

**Docker Compose** orchestrates multiple containers as a single application using a YAML configuration file.

```
┌─────────────────────────────────────────────────────────────┐
│          Docker Compose Architecture                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  docker-compose.yml                                         │
│  ├─ api (Node.js)         ──┐                              │
│  ├─ db (PostgreSQL)       ──┼──→ Docker Network            │
│  ├─ cache (Redis)         ──┤    (banking-network)         │
│  ├─ nginx (Load Balancer) ──┘                              │
│  └─ monitoring (Grafana)                                    │
│                                                              │
│  Single Command: docker-compose up                          │
│  ↓                                                           │
│  Creates:                                                    │
│  - 5 containers                                             │
│  - 1 network                                                │
│  - 3 volumes                                                │
│  - All configured and connected                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Docker Compose vs Dockerfile

| Aspect | Dockerfile | Docker Compose |
|--------|-----------|----------------|
| **Purpose** | Build single image | Orchestrate multiple containers |
| **Scope** | One service | Multiple services |
| **Format** | Custom syntax | YAML |
| **Use Case** | Image definition | Application stack |
| **Command** | `docker build` | `docker-compose up` |

---

## 💡 Example: Complete Banking Platform

Full-stack banking application with API, database, cache, and load balancer.

### Project Structure

```
banking-platform/
├── docker-compose.yml
├── docker-compose.prod.yml
├── .env
├── api/
│   ├── Dockerfile
│   ├── src/
│   └── package.json
├── nginx/
│   ├── Dockerfile
│   └── nginx.conf
└── postgres/
    └── init.sql
```

### docker-compose.yml (Development)

```yaml
version: '3.8'

services:
  # ============================================
  # PostgreSQL Database
  # ============================================
  db:
    image: postgres:15-alpine
    container_name: banking-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: banking
      POSTGRES_USER: bankuser
      POSTGRES_PASSWORD: ${DB_PASSWORD:-bankpass}
      PGDATA: /var/lib/postgresql/data/pgdata
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U bankuser -d banking"]
      interval: 10s
      timeout: 3s
      retries: 5
    networks:
      - banking-network

  # ============================================
  # Redis Cache
  # ============================================
  cache:
    image: redis:7-alpine
    container_name: banking-cache
    restart: unless-stopped
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD:-redispass}
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    networks:
      - banking-network

  # ============================================
  # Banking API (Node.js)
  # ============================================
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
      args:
        NODE_ENV: development
    container_name: banking-api
    restart: unless-stopped
    environment:
      NODE_ENV: development
      PORT: 3000
      DB_HOST: db
      DB_PORT: 5432
      DB_NAME: banking
      DB_USER: bankuser
      DB_PASSWORD: ${DB_PASSWORD:-bankpass}
      REDIS_HOST: cache
      REDIS_PORT: 6379
      REDIS_PASSWORD: ${REDIS_PASSWORD:-redispass}
    ports:
      - "3000:3000"
    volumes:
      - ./api/src:/app/src
      - /app/node_modules
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "node", "-e", "require('http').get('http://localhost:3000/health')"]
      interval: 30s
      timeout: 3s
      retries: 3
    networks:
      - banking-network

  # ============================================
  # NGINX Load Balancer / Reverse Proxy
  # ============================================
  nginx:
    build:
      context: ./nginx
      dockerfile: Dockerfile
    container_name: banking-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - api
    networks:
      - banking-network

  # ============================================
  # Adminer (Database Management)
  # ============================================
  adminer:
    image: adminer:latest
    container_name: banking-adminer
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      ADMINER_DEFAULT_SERVER: db
    depends_on:
      - db
    networks:
      - banking-network

# ============================================
# Networks
# ============================================
networks:
  banking-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.25.0.0/16

# ============================================
# Volumes (Persistent Data)
# ============================================
volumes:
  postgres-data:
    driver: local
  redis-data:
    driver: local
```

### .env File

```bash
# .env - Environment configuration
NODE_ENV=development
DB_PASSWORD=secure_bank_password
REDIS_PASSWORD=secure_redis_password

# API Configuration
API_PORT=3000
JWT_SECRET=your-jwt-secret-key

# Database Configuration
DB_HOST=db
DB_PORT=5432
DB_NAME=banking
DB_USER=bankuser
```

### API Dockerfile (api/Dockerfile)

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

EXPOSE 3000

CMD ["npm", "run", "dev"]
```

### API Server (api/src/server.js)

```javascript
const express = require('express');
const { Pool } = require('pg');
const redis = require('redis');

const app = express();
app.use(express.json());

// PostgreSQL connection
const pool = new Pool({
  host: process.env.DB_HOST,
  port: process.env.DB_PORT,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  max: 20,
  idleTimeoutMillis: 30000
});

// Redis connection
const redisClient = redis.createClient({
  socket: {
    host: process.env.REDIS_HOST,
    port: process.env.REDIS_PORT
  },
  password: process.env.REDIS_PASSWORD
});

redisClient.connect().catch(console.error);

// Health check
app.get('/health', async (req, res) => {
  try {
    await pool.query('SELECT 1');
    await redisClient.ping();
    
    res.json({
      status: 'healthy',
      database: 'connected',
      cache: 'connected',
      uptime: process.uptime()
    });
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      error: error.message
    });
  }
});

// Get accounts
app.get('/api/accounts', async (req, res) => {
  try {
    // Try cache first
    const cached = await redisClient.get('accounts');
    if (cached) {
      return res.json(JSON.parse(cached));
    }
    
    // Query database
    const result = await pool.query('SELECT * FROM accounts LIMIT 10');
    
    // Cache result
    await redisClient.setEx('accounts', 60, JSON.stringify(result.rows));
    
    res.json(result.rows);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create account
app.post('/api/accounts', async (req, res) => {
  const { account_number, account_type, balance } = req.body;
  
  try {
    const result = await pool.query(
      'INSERT INTO accounts (account_number, account_type, balance) VALUES ($1, $2, $3) RETURNING *',
      [account_number, account_type, balance]
    );
    
    // Invalidate cache
    await redisClient.del('accounts');
    
    res.status(201).json(result.rows[0]);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Transactions
app.post('/api/transactions', async (req, res) => {
  const { account_id, amount, type } = req.body;
  
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // Update balance
    const updateQuery = type === 'deposit'
      ? 'UPDATE accounts SET balance = balance + $1 WHERE id = $2'
      : 'UPDATE accounts SET balance = balance - $1 WHERE id = $2';
    
    await client.query(updateQuery, [amount, account_id]);
    
    // Record transaction
    const result = await client.query(
      'INSERT INTO transactions (account_id, amount, type) VALUES ($1, $2, $3) RETURNING *',
      [account_id, amount, type]
    );
    
    await client.query('COMMIT');
    
    res.status(201).json(result.rows[0]);
  } catch (error) {
    await client.query('ROLLBACK');
    res.status(400).json({ error: error.message });
  } finally {
    client.release();
  }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, '0.0.0.0', () => {
  console.log(`🚀 Banking API running on port ${PORT}`);
});
```

### Database Initialization (postgres/init.sql)

```sql
-- postgres/init.sql - Database schema
CREATE TABLE IF NOT EXISTS accounts (
    id SERIAL PRIMARY KEY,
    account_number VARCHAR(20) UNIQUE NOT NULL,
    account_type VARCHAR(20) NOT NULL,
    balance DECIMAL(15, 2) DEFAULT 0.00,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS transactions (
    id SERIAL PRIMARY KEY,
    account_id INTEGER REFERENCES accounts(id),
    amount DECIMAL(15, 2) NOT NULL,
    type VARCHAR(20) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_account_number ON accounts(account_number);
CREATE INDEX idx_account_transactions ON transactions(account_id);

-- Seed data
INSERT INTO accounts (account_number, account_type, balance) VALUES
('ACC-001', 'checking', 5000.00),
('ACC-002', 'savings', 10000.00),
('ACC-003', 'credit', 2500.00);
```

### NGINX Configuration (nginx/nginx.conf)

```nginx
events {
    worker_connections 1024;
}

http {
    upstream api {
        server api:3000;
    }

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://api;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location /health {
            proxy_pass http://api/health;
            access_log off;
        }
    }
}
```

---

## 💡 Production Docker Compose

### docker-compose.prod.yml

```yaml
version: '3.8'

services:
  db:
    image: postgres:15-alpine
    restart: always
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - banking-network
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G

  cache:
    image: redis:7-alpine
    restart: always
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis-data:/data
    networks:
      - banking-network

  api:
    image: banking-api:${VERSION:-latest}
    restart: always
    environment:
      NODE_ENV: production
      DB_HOST: db
      REDIS_HOST: cache
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1'
          memory: 512M
    networks:
      - banking-network

  nginx:
    image: nginx:alpine
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - api
    networks:
      - banking-network

networks:
  banking-network:
    driver: bridge

volumes:
  postgres-data:
  redis-data:
```

---

## 🎯 Docker Compose Commands

### Basic Commands

```bash
# Start all services
docker-compose up

# Start in detached mode
docker-compose up -d

# Start specific service
docker-compose up api

# Build and start
docker-compose up --build

# Stop all services
docker-compose down

# Stop and remove volumes
docker-compose down -v

# View logs
docker-compose logs
docker-compose logs -f api

# List services
docker-compose ps

# Execute command in service
docker-compose exec api sh
docker-compose exec db psql -U bankuser -d banking

# Scale service
docker-compose up --scale api=3

# Restart service
docker-compose restart api

# View resource usage
docker-compose top
```

### Production Commands

```bash
# Use production compose file
docker-compose -f docker-compose.prod.yml up -d

# Multiple compose files (merge)
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Pull latest images
docker-compose pull

# Build images
docker-compose build

# Push images to registry
docker-compose push
```

---

## 📚 Best Practices

### 1. Environment Variables

```yaml
# Use .env file
environment:
  DB_PASSWORD: ${DB_PASSWORD}

# Or env_file
env_file:
  - .env
  - .env.local
```

### 2. Health Checks

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

### 3. Dependencies

```yaml
depends_on:
  db:
    condition: service_healthy  # Wait for health check
```

### 4. Resource Limits

```yaml
deploy:
  resources:
    limits:
      cpus: '0.5'
      memory: 512M
    reservations:
      cpus: '0.25'
      memory: 256M
```

---

## ✅ Summary

### Core Concepts

1. **Services**: Containers in your application
2. **Networks**: Communication between containers
3. **Volumes**: Persistent data storage
4. **Dependencies**: Service startup order

### Banking Stack

```
NGINX (80) → API (3000) → PostgreSQL (5432)
                        → Redis (6379)
```

### Commands

- `docker-compose up -d`: Start
- `docker-compose logs -f`: View logs
- `docker-compose down -v`: Stop and clean
- `docker-compose exec api sh`: Shell access

---

**Status**: ✅ Complete with production-ready Docker Compose banking platform!
