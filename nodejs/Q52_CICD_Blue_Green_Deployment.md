# Q52: CI/CD - Blue-Green Deployment

## 📋 Summary
This question covers **blue-green deployment** strategies for achieving zero-downtime deployments in production. Learn to implement safe, reversible deployments with instant rollback capabilities for Node.js banking applications.

**Key Topics**:
- Blue-green deployment pattern fundamentals
- Traffic switching mechanisms
- Health checks and validation
- Database migration strategies
- Rollback procedures
- AWS implementation (ECS, ALB, Route53)
- Kubernetes blue-green deployments
- Monitoring and observability
- Cost optimization

**Banking Use Cases**:
- Zero-downtime API updates
- Safe feature releases
- Instant rollback on errors
- Compliance with SLA requirements
- A/B testing capabilities
- Gradual traffic migration

---

## 🎯 Understanding Blue-Green Deployment

### What is Blue-Green Deployment?

**Blue-Green Deployment** is a release strategy that reduces downtime and risk by running two identical production environments ("blue" and "green") and switching traffic between them.

```
┌────────────────────────────────────────────────────────────────┐
│            Blue-Green Deployment Flow                          │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  INITIAL STATE                                                 │
│  ┌──────────────┐                                              │
│  │   Load       │                                              │
│  │  Balancer    │                                              │
│  └──────┬───────┘                                              │
│         │                                                       │
│         ▼                                                       │
│  ┌──────────────┐        ┌──────────────┐                     │
│  │   BLUE       │        │   GREEN      │                     │
│  │  (Active)    │        │   (Idle)     │                     │
│  │  v1.0.0      │        │  v1.0.0      │                     │
│  └──────────────┘        └──────────────┘                     │
│                                                                 │
│  ──────────────────────────────────────────────────            │
│                                                                 │
│  DEPLOYMENT PHASE                                              │
│  ┌──────────────┐                                              │
│  │   Load       │                                              │
│  │  Balancer    │                                              │
│  └──────┬───────┘                                              │
│         │                                                       │
│         ▼                                                       │
│  ┌──────────────┐        ┌──────────────┐                     │
│  │   BLUE       │        │   GREEN      │                     │
│  │  (Active)    │        │ (Deploying)  │                     │
│  │  v1.0.0      │        │  v1.1.0      │ ← Deploy new        │
│  └──────────────┘        └──────┬───────┘                     │
│                                  │                              │
│                                  ▼                              │
│                           Health checks                        │
│                           Smoke tests                          │
│                                                                 │
│  ──────────────────────────────────────────────────            │
│                                                                 │
│  TRAFFIC SWITCH                                                │
│  ┌──────────────┐                                              │
│  │   Load       │                                              │
│  │  Balancer    │                                              │
│  └──────────────┘                                              │
│         │                                                       │
│         ▼                                                       │
│  ┌──────────────┐        ┌──────────────┐                     │
│  │   BLUE       │        │   GREEN      │                     │
│  │   (Idle)     │        │  (Active)    │ ← Traffic switched  │
│  │  v1.0.0      │        │  v1.1.0      │                     │
│  └──────────────┘        └──────────────┘                     │
│                                                                 │
│  ──────────────────────────────────────────────────            │
│                                                                 │
│  ROLLBACK (if needed)                                          │
│  ┌──────────────┐                                              │
│  │   Load       │                                              │
│  │  Balancer    │                                              │
│  └──────┬───────┘                                              │
│         │                                                       │
│         ▼                                                       │
│  ┌──────────────┐        ┌──────────────┐                     │
│  │   BLUE       │        │   GREEN      │                     │
│  │  (Active)    │ ← Switch back         │  (Idle)     │       │
│  │  v1.0.0      │        │  v1.1.0      │                     │
│  └──────────────┘        └──────────────┘                     │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### Key Benefits

| Benefit | Description |
|---------|-------------|
| **Zero Downtime** | Traffic switches instantly between environments |
| **Instant Rollback** | Switch back to old version in seconds |
| **Easy Testing** | Test new version in production-like environment |
| **Reduced Risk** | Validate before full release |
| **Simple Strategy** | Easy to understand and implement |
| **A/B Testing** | Can send partial traffic to new version |

### Comparison with Other Strategies

| Strategy | Downtime | Rollback Speed | Complexity | Cost |
|----------|----------|----------------|------------|------|
| **Blue-Green** | None | Instant | Low | High (2x resources) |
| **Canary** | None | Fast | Medium | Medium |
| **Rolling** | None | Slow | Low | Low |
| **Recreate** | Yes | Slow | Very Low | Low |

---

## 💡 Example 1: AWS ECS Blue-Green Deployment

Complete implementation using AWS ECS, Application Load Balancer, and CodeDeploy.

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                   AWS Blue-Green Architecture                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Internet                                                       │
│      │                                                           │
│      ▼                                                           │
│  ┌──────────────────────┐                                       │
│  │  Route 53            │                                       │
│  │  (DNS)               │                                       │
│  └──────────┬───────────┘                                       │
│             │                                                    │
│             ▼                                                    │
│  ┌──────────────────────┐                                       │
│  │  Application Load    │                                       │
│  │  Balancer (ALB)      │                                       │
│  │  - Target Group Blue │                                       │
│  │  - Target Group Green│                                       │
│  └──────────┬───────────┘                                       │
│             │                                                    │
│       ┌─────┴─────┐                                             │
│       │           │                                             │
│       ▼           ▼                                             │
│  ┌─────────┐  ┌─────────┐                                      │
│  │  ECS    │  │  ECS    │                                      │
│  │ Service │  │ Service │                                      │
│  │  BLUE   │  │  GREEN  │                                      │
│  │         │  │         │                                      │
│  │ Tasks:  │  │ Tasks:  │                                      │
│  │  ├─ T1  │  │  ├─ T1  │                                      │
│  │  ├─ T2  │  │  ├─ T2  │                                      │
│  │  └─ T3  │  │  └─ T3  │                                      │
│  └─────────┘  └─────────┘                                      │
│       │           │                                             │
│       └─────┬─────┘                                             │
│             │                                                    │
│             ▼                                                    │
│  ┌──────────────────────┐                                       │
│  │  RDS PostgreSQL      │                                       │
│  │  (Shared)            │                                       │
│  └──────────────────────┘                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Project Structure

```
banking-api/
├── infrastructure/
│   ├── ecs-task-definition.json
│   ├── appspec.yaml
│   └── cloudformation/
│       ├── alb.yaml
│       ├── ecs-cluster.yaml
│       └── codedeploy.yaml
├── scripts/
│   ├── deploy-blue-green.sh
│   ├── health-check.sh
│   └── rollback.sh
├── src/
│   └── index.js
├── Dockerfile
└── package.json
```

### 1. ECS Task Definition

```json
{
  "family": "banking-api",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "banking-api",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/banking-api:PLACEHOLDER",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "NODE_ENV",
          "value": "production"
        },
        {
          "name": "PORT",
          "value": "3000"
        }
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:banking/db-password"
        },
        {
          "name": "JWT_SECRET",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:banking/jwt-secret"
        }
      ],
      "healthCheck": {
        "command": [
          "CMD-SHELL",
          "curl -f http://localhost:3000/health || exit 1"
        ],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/banking-api",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

### 2. CodeDeploy AppSpec

```yaml
# appspec.yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "arn:aws:ecs:us-east-1:123456789012:task-definition/banking-api:LATEST"
        LoadBalancerInfo:
          ContainerName: "banking-api"
          ContainerPort: 3000
        PlatformVersion: "LATEST"
        NetworkConfiguration:
          AwsvpcConfiguration:
            Subnets:
              - "subnet-12345678"
              - "subnet-87654321"
            SecurityGroups:
              - "sg-0123456789abcdef0"
            AssignPublicIp: "DISABLED"

Hooks:
  - BeforeInstall: "LambdaFunctionToValidateBeforeInstall"
  - AfterInstall: "LambdaFunctionToValidateAfterInstall"
  - AfterAllowTestTraffic: "LambdaFunctionToValidateAfterTestTraffic"
  - BeforeAllowTraffic: "LambdaFunctionToValidateBeforeAllowTraffic"
  - AfterAllowTraffic: "LambdaFunctionToValidateAfterAllowTraffic"
```

### 3. Deployment Script

```bash
#!/bin/bash
# scripts/deploy-blue-green.sh

set -e

# Configuration
AWS_REGION="us-east-1"
ECS_CLUSTER="banking-api-prod"
ECS_SERVICE="banking-api-service"
CODEDEPLOY_APP="banking-api-app"
CODEDEPLOY_DEPLOYMENT_GROUP="banking-api-dg"
IMAGE_TAG="${1:-latest}"
TASK_DEFINITION_FAMILY="banking-api"

echo "🚀 Starting Blue-Green Deployment"
echo "Image tag: $IMAGE_TAG"

# Step 1: Update task definition with new image
echo "📝 Updating task definition..."
TASK_DEF_JSON=$(aws ecs describe-task-definition \
  --task-definition $TASK_DEFINITION_FAMILY \
  --region $AWS_REGION \
  --query 'taskDefinition' \
  --output json)

# Replace image tag
NEW_TASK_DEF=$(echo $TASK_DEF_JSON | jq \
  --arg IMAGE "123456789012.dkr.ecr.us-east-1.amazonaws.com/banking-api:$IMAGE_TAG" \
  '.containerDefinitions[0].image = $IMAGE | del(.taskDefinitionArn, .revision, .status, .requiresAttributes, .compatibilities, .registeredAt, .registeredBy)')

# Register new task definition
echo "$NEW_TASK_DEF" > /tmp/task-def.json
NEW_TASK_DEF_ARN=$(aws ecs register-task-definition \
  --cli-input-json file:///tmp/task-def.json \
  --region $AWS_REGION \
  --query 'taskDefinition.taskDefinitionArn' \
  --output text)

echo "✅ New task definition registered: $NEW_TASK_DEF_ARN"

# Step 2: Create deployment configuration
cat > /tmp/appspec.yaml << EOF
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "$NEW_TASK_DEF_ARN"
        LoadBalancerInfo:
          ContainerName: "banking-api"
          ContainerPort: 3000
EOF

# Step 3: Start CodeDeploy deployment
echo "🚢 Creating CodeDeploy deployment..."
DEPLOYMENT_ID=$(aws deploy create-deployment \
  --application-name $CODEDEPLOY_APP \
  --deployment-group-name $CODEDEPLOY_DEPLOYMENT_GROUP \
  --deployment-config-name CodeDeployDefault.ECSAllAtOnce \
  --description "Blue-Green deployment of $IMAGE_TAG" \
  --revision "{\"revisionType\":\"AppSpecContent\",\"appSpecContent\":{\"content\":\"$(cat /tmp/appspec.yaml | base64)\"}}" \
  --region $AWS_REGION \
  --query 'deploymentId' \
  --output text)

echo "✅ Deployment created: $DEPLOYMENT_ID"
echo "🔗 https://$AWS_REGION.console.aws.amazon.com/codesuite/codedeploy/deployments/$DEPLOYMENT_ID"

# Step 4: Monitor deployment
echo "👀 Monitoring deployment..."
while true; do
  STATUS=$(aws deploy get-deployment \
    --deployment-id $DEPLOYMENT_ID \
    --region $AWS_REGION \
    --query 'deploymentInfo.status' \
    --output text)
  
  echo "Status: $STATUS"
  
  if [ "$STATUS" = "Succeeded" ]; then
    echo "✅ Deployment succeeded!"
    break
  elif [ "$STATUS" = "Failed" ] || [ "$STATUS" = "Stopped" ]; then
    echo "❌ Deployment failed!"
    exit 1
  fi
  
  sleep 30
done

# Step 5: Run smoke tests
echo "🧪 Running smoke tests..."
ALB_DNS=$(aws elbv2 describe-load-balancers \
  --names banking-api-alb \
  --region $AWS_REGION \
  --query 'LoadBalancers[0].DNSName' \
  --output text)

bash scripts/health-check.sh "https://$ALB_DNS"

echo "✅ Blue-Green deployment completed successfully!"
```

### 4. Health Check Script

```bash
#!/bin/bash
# scripts/health-check.sh

set -e

API_URL="${1:-http://localhost:3000}"
MAX_RETRIES=10
RETRY_DELAY=10

echo "🏥 Running health checks against $API_URL"

# Test 1: Health endpoint
echo "1️⃣ Testing health endpoint..."
for i in $(seq 1 $MAX_RETRIES); do
  if curl -sf "$API_URL/health" > /dev/null; then
    echo "✅ Health check passed"
    break
  else
    echo "⏳ Attempt $i/$MAX_RETRIES failed, retrying..."
    sleep $RETRY_DELAY
  fi
  
  if [ $i -eq $MAX_RETRIES ]; then
    echo "❌ Health check failed after $MAX_RETRIES attempts"
    exit 1
  fi
done

# Test 2: API functionality
echo "2️⃣ Testing API endpoints..."
RESPONSE=$(curl -s -w "\n%{http_code}" "$API_URL/api/v1/accounts")
HTTP_CODE=$(echo "$RESPONSE" | tail -n1)
BODY=$(echo "$RESPONSE" | head -n-1)

if [ "$HTTP_CODE" -eq 200 ]; then
  echo "✅ API endpoint test passed"
else
  echo "❌ API endpoint test failed with status $HTTP_CODE"
  exit 1
fi

# Test 3: Database connectivity
echo "3️⃣ Testing database connectivity..."
DB_STATUS=$(echo "$BODY" | jq -r '.database' 2>/dev/null || echo "unknown")
if [ "$DB_STATUS" = "connected" ]; then
  echo "✅ Database connectivity verified"
else
  echo "❌ Database connectivity failed"
  exit 1
fi

# Test 4: Response time check
echo "4️⃣ Testing response times..."
for i in {1..5}; do
  TIME=$(curl -o /dev/null -s -w '%{time_total}\n' "$API_URL/health")
  echo "  Response time: ${TIME}s"
  
  if (( $(echo "$TIME > 1.0" | bc -l) )); then
    echo "⚠️ Slow response detected"
  fi
done

echo "✅ All health checks passed!"
```

### 5. Rollback Script

```bash
#!/bin/bash
# scripts/rollback.sh

set -e

AWS_REGION="us-east-1"
CODEDEPLOY_APP="banking-api-app"
CODEDEPLOY_DEPLOYMENT_GROUP="banking-api-dg"

echo "🔄 Starting rollback..."

# Get latest deployment
LATEST_DEPLOYMENT=$(aws deploy list-deployments \
  --application-name $CODEDEPLOY_APP \
  --deployment-group-name $CODEDEPLOY_DEPLOYMENT_GROUP \
  --region $AWS_REGION \
  --max-items 1 \
  --query 'deployments[0]' \
  --output text)

echo "Latest deployment: $LATEST_DEPLOYMENT"

# Stop current deployment
echo "⏹️ Stopping current deployment..."
aws deploy stop-deployment \
  --deployment-id $LATEST_DEPLOYMENT \
  --auto-rollback-enabled \
  --region $AWS_REGION

echo "✅ Rollback initiated. Traffic will be switched back to previous version."

# Monitor rollback
while true; do
  STATUS=$(aws deploy get-deployment \
    --deployment-id $LATEST_DEPLOYMENT \
    --region $AWS_REGION \
    --query 'deploymentInfo.status' \
    --output text)
  
  echo "Rollback status: $STATUS"
  
  if [ "$STATUS" = "Stopped" ]; then
    echo "✅ Rollback completed!"
    break
  fi
  
  sleep 10
done
```

### 6. Application with Health Endpoint

```javascript
// src/index.js
const express = require('express');
const { Pool } = require('pg');

const app = express();
const PORT = process.env.PORT || 3000;
const VERSION = process.env.APP_VERSION || '1.0.0';

// Database connection
const pool = new Pool({
  host: process.env.DB_HOST,
  port: process.env.DB_PORT || 5432,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// ============================================
// Health Check Endpoint (Critical for Blue-Green)
// ============================================
app.get('/health', async (req, res) => {
  try {
    // Check database connection
    const dbCheck = await pool.query('SELECT 1');
    
    res.json({
      status: 'healthy',
      version: VERSION,
      timestamp: new Date().toISOString(),
      database: 'connected',
      uptime: process.uptime(),
      memory: process.memoryUsage(),
    });
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      error: error.message,
      database: 'disconnected',
    });
  }
});

// Readiness check (stricter than health)
app.get('/ready', async (req, res) => {
  try {
    // Verify database schema
    const result = await pool.query(`
      SELECT COUNT(*) FROM information_schema.tables 
      WHERE table_schema = 'public'
    `);
    
    if (result.rows[0].count > 0) {
      res.json({ status: 'ready' });
    } else {
      res.status(503).json({ status: 'not_ready', reason: 'database_not_initialized' });
    }
  } catch (error) {
    res.status(503).json({ status: 'not_ready', error: error.message });
  }
});

// ============================================
// API Endpoints
// ============================================
app.get('/api/v1/accounts', async (req, res) => {
  try {
    const result = await pool.query('SELECT * FROM accounts LIMIT 10');
    res.json({
      success: true,
      data: result.rows,
      version: VERSION,
      database: 'connected',
    });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

// ============================================
// Graceful Shutdown (Important for Blue-Green)
// ============================================
let server;
let isShuttingDown = false;

async function gracefulShutdown(signal) {
  if (isShuttingDown) return;
  isShuttingDown = true;
  
  console.log(`${signal} received. Starting graceful shutdown...`);
  
  // Stop accepting new connections
  server.close(() => {
    console.log('HTTP server closed');
  });
  
  // Close database connections
  await pool.end();
  console.log('Database connections closed');
  
  process.exit(0);
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));

// ============================================
// Start Server
// ============================================
server = app.listen(PORT, () => {
  console.log(`🏦 Banking API v${VERSION} listening on port ${PORT}`);
  console.log(`Health check: http://localhost:${PORT}/health`);
});
```

---

## 💡 Example 2: Kubernetes Blue-Green Deployment

Implementation using Kubernetes services and Ingress.

### Project Structure

```
k8s/
├── blue-deployment.yaml
├── green-deployment.yaml
├── service-blue.yaml
├── service-green.yaml
├── ingress.yaml
└── scripts/
    ├── deploy-green.sh
    ├── switch-traffic.sh
    └── rollback-to-blue.sh
```

### 1. Blue Deployment

```yaml
# k8s/blue-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: banking-api-blue
  namespace: production
  labels:
    app: banking-api
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: banking-api
      version: blue
  template:
    metadata:
      labels:
        app: banking-api
        version: blue
    spec:
      containers:
      - name: banking-api
        image: banking-api:v1.0.0
        ports:
        - containerPort: 3000
          name: http
        env:
        - name: NODE_ENV
          value: "production"
        - name: APP_VERSION
          value: "1.0.0"
        envFrom:
        - secretRef:
            name: banking-api-secrets
        - configMapRef:
            name: banking-api-config
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
```

### 2. Green Deployment

```yaml
# k8s/green-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: banking-api-green
  namespace: production
  labels:
    app: banking-api
    version: green
spec:
  replicas: 0  # Initially 0, scaled up during deployment
  selector:
    matchLabels:
      app: banking-api
      version: green
  template:
    metadata:
      labels:
        app: banking-api
        version: green
    spec:
      containers:
      - name: banking-api
        image: banking-api:v1.1.0  # New version
        ports:
        - containerPort: 3000
          name: http
        env:
        - name: NODE_ENV
          value: "production"
        - name: APP_VERSION
          value: "1.1.0"
        envFrom:
        - secretRef:
            name: banking-api-secrets
        - configMapRef:
            name: banking-api-config
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
```

### 3. Service Configuration

```yaml
# k8s/service-blue.yaml
apiVersion: v1
kind: Service
metadata:
  name: banking-api-blue
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: banking-api
    version: blue
  ports:
  - port: 80
    targetPort: 3000
    protocol: TCP

---
# k8s/service-green.yaml
apiVersion: v1
kind: Service
metadata:
  name: banking-api-green
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: banking-api
    version: green
  ports:
  - port: 80
    targetPort: 3000
    protocol: TCP

---
# k8s/service-main.yaml (Active service - points to blue or green)
apiVersion: v1
kind: Service
metadata:
  name: banking-api
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: banking-api
    version: blue  # Initially points to blue
  ports:
  - port: 80
    targetPort: 3000
    protocol: TCP
```

### 4. Ingress Configuration

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: banking-api-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  tls:
  - hosts:
    - api.banking.example.com
    secretName: banking-api-tls
  rules:
  - host: api.banking.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: banking-api  # Points to active version
            port:
              number: 80
```

### 5. Deployment Script

```bash
#!/bin/bash
# k8s/scripts/deploy-green.sh

set -e

NAMESPACE="production"
NEW_VERSION="${1:-v1.1.0}"

echo "🚀 Starting Green deployment (version: $NEW_VERSION)"

# Step 1: Update green deployment with new version
echo "📝 Updating green deployment..."
kubectl set image deployment/banking-api-green \
  banking-api=banking-api:$NEW_VERSION \
  -n $NAMESPACE

# Step 2: Scale up green deployment
echo "⬆️ Scaling up green deployment..."
kubectl scale deployment/banking-api-green --replicas=3 -n $NAMESPACE

# Step 3: Wait for green pods to be ready
echo "⏳ Waiting for green pods to be ready..."
kubectl rollout status deployment/banking-api-green -n $NAMESPACE

# Step 4: Run health checks
echo "🏥 Running health checks on green pods..."
GREEN_PODS=$(kubectl get pods -n $NAMESPACE -l version=green -o jsonpath='{.items[*].metadata.name}')

for pod in $GREEN_PODS; do
  echo "Testing pod: $pod"
  kubectl exec -n $NAMESPACE $pod -- curl -f http://localhost:3000/health
done

echo "✅ Green deployment ready for traffic switch"
echo "Run ./switch-traffic.sh to switch traffic to green"
```

### 6. Traffic Switch Script

```bash
#!/bin/bash
# k8s/scripts/switch-traffic.sh

set -e

NAMESPACE="production"

echo "🔄 Switching traffic from blue to green..."

# Update main service selector to point to green
kubectl patch service banking-api -n $NAMESPACE -p '{"spec":{"selector":{"version":"green"}}}'

echo "✅ Traffic switched to green"
echo "Monitoring for 5 minutes..."

# Monitor for issues
sleep 300

# Check error rates
ERROR_COUNT=$(kubectl logs -n $NAMESPACE -l version=green --since=5m | grep -c "ERROR" || true)
echo "Error count in last 5 minutes: $ERROR_COUNT"

if [ $ERROR_COUNT -gt 10 ]; then
  echo "⚠️ High error rate detected! Consider rolling back."
  exit 1
fi

echo "✅ Traffic switch successful"
echo "Blue deployment can now be scaled down or updated"

# Optional: Scale down blue
read -p "Scale down blue deployment? (y/n) " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
  kubectl scale deployment/banking-api-blue --replicas=0 -n $NAMESPACE
  echo "✅ Blue deployment scaled down"
fi
```

### 7. Rollback Script

```bash
#!/bin/bash
# k8s/scripts/rollback-to-blue.sh

set -e

NAMESPACE="production"

echo "🔄 Rolling back traffic to blue..."

# Switch service back to blue
kubectl patch service banking-api -n $NAMESPACE -p '{"spec":{"selector":{"version":"blue"}}}'

echo "✅ Traffic rolled back to blue"

# Optional: Scale down green
read -p "Scale down green deployment? (y/n) " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
  kubectl scale deployment/banking-api-green --replicas=0 -n $NAMESPACE
  echo "✅ Green deployment scaled down"
fi

echo "✅ Rollback complete"
```

---

## 🎯 Best Practices

### 1. Database Migrations

```javascript
// Backward-compatible migrations
// migrations/001_add_account_status.js

// ✅ GOOD: Add nullable column
exports.up = async (knex) => {
  await knex.schema.table('accounts', (table) => {
    table.string('status').nullable();  // Nullable allows old version to work
  });
  
  // Set default for existing rows
  await knex('accounts').update({ status: 'active' });
};

// ❌ BAD: Non-nullable column breaks old version
exports.up = async (knex) => {
  await knex.schema.table('accounts', (table) => {
    table.string('status').notNullable();  // Old version crashes!
  });
};
```

### 2. Feature Flags

```javascript
// Use feature flags for gradual rollout
const unleash = require('unleash-client');

unleash.initialize({
  url: 'http://unleash.example.com/api',
  appName: 'banking-api',
  environment: process.env.NODE_ENV,
});

app.get('/api/v1/transfer', async (req, res) => {
  // New feature only enabled for green deployment
  if (unleash.isEnabled('new_transfer_flow', { version: process.env.APP_VERSION })) {
    return handleNewTransferFlow(req, res);
  }
  
  // Fallback to old implementation
  return handleOldTransferFlow(req, res);
});
```

### 3. Monitoring During Deployment

```javascript
// Enhanced logging with version info
const winston = require('winston');

const logger = winston.createLogger({
  defaultMeta: {
    service: 'banking-api',
    version: process.env.APP_VERSION,
    environment: process.env.NODE_ENV,
  },
  transports: [
    new winston.transports.Console(),
    new winston.transports.CloudWatch({
      logGroupName: '/ecs/banking-api',
      logStreamName: `${process.env.APP_VERSION}-${process.env.HOSTNAME}`,
    }),
  ],
});

// Alert on high error rates
setInterval(async () => {
  const errorRate = await getErrorRate();
  
  if (errorRate > 1.0) {  // 1% error rate
    logger.error('High error rate detected', {
      errorRate,
      alert: true,
      severity: 'critical',
    });
  }
}, 60000);  // Check every minute
```

### 4. Automated Rollback

```javascript
// scripts/auto-rollback.js
const AWS = require('aws-sdk');
const cloudwatch = new AWS.CloudWatch();

const METRIC_NAMESPACE = 'Banking/API';
const ERROR_THRESHOLD = 1.0;  // 1%
const DEPLOYMENT_ID = process.env.DEPLOYMENT_ID;

async function checkMetrics() {
  const params = {
    Namespace: METRIC_NAMESPACE,
    MetricName: 'ErrorRate',
    StartTime: new Date(Date.now() - 5 * 60 * 1000),
    EndTime: new Date(),
    Period: 300,
    Statistics: ['Average'],
  };
  
  const data = await cloudwatch.getMetricStatistics(params).promise();
  
  if (data.Datapoints.length > 0) {
    const errorRate = data.Datapoints[0].Average;
    
    if (errorRate > ERROR_THRESHOLD) {
      console.log(`Error rate ${errorRate}% exceeds threshold. Triggering rollback...`);
      await triggerRollback(DEPLOYMENT_ID);
    }
  }
}

async function triggerRollback(deploymentId) {
  const codedeploy = new AWS.CodeDeploy();
  
  await codedeploy.stopDeployment({
    deploymentId,
    autoRollbackEnabled: true,
  }).promise();
  
  console.log('Rollback initiated');
}

// Run check every 60 seconds for 10 minutes
const interval = setInterval(checkMetrics, 60000);
setTimeout(() => clearInterval(interval), 10 * 60 * 1000);
```

---

## 📚 Common Interview Questions

### Q1: What's the difference between blue-green and canary deployment?

**Answer**:
- **Blue-Green**: All traffic switches at once (100% → 0% or vice versa)
- **Canary**: Gradual rollout (10% → 25% → 50% → 100%)

Blue-green is simpler but requires 2x resources. Canary is safer but more complex.

### Q2: How do you handle database migrations in blue-green deployments?

**Answer**:
1. **Backward-compatible changes**: Add nullable columns, don't drop
2. **Multi-phase migrations**: 
   - Phase 1: Add new column (nullable)
   - Deploy new code
   - Phase 2: Backfill data
   - Phase 3: Make column non-nullable
3. **Feature flags**: Hide new features until migration complete

### Q3: What happens if the new version has bugs?

**Answer**:
Instant rollback:
- Switch traffic back to old version (seconds)
- No need to redeploy
- Old version still running and warm
- Monitor health checks continuously

### Q4: How do you minimize cost with blue-green?

**Answer**:
1. **Scale down idle environment** after validation
2. **Use spot instances** for green during testing
3. **Shared resources**: Database, cache shared between environments
4. **Time-box**: Auto-scale down green if not promoted within X hours

### Q5: How do you test the green environment before switching traffic?

**Answer**:
1. **Smoke tests**: Basic functionality checks
2. **Synthetic monitoring**: Run test transactions
3. **Parallel testing**: Send duplicate traffic (shadowing)
4. **Gradual rollout**: Send 1% traffic first (canary within blue-green)

---

## ✅ Summary & Key Takeaways

### Blue-Green Deployment Checklist

```yaml
✅ Pre-Deployment:
  - [ ] Green environment provisioned
  - [ ] Database migrations backward-compatible
  - [ ] Feature flags configured
  - [ ] Monitoring dashboards ready
  - [ ] Rollback plan documented

✅ Deployment:
  - [ ] Deploy to green environment
  - [ ] Run health checks
  - [ ] Run smoke tests
  - [ ] Validate database connectivity
  - [ ] Check application logs

✅ Traffic Switch:
  - [ ] Switch load balancer to green
  - [ ] Monitor error rates (5-10 minutes)
  - [ ] Check latency metrics
  - [ ] Verify business metrics
  - [ ] Keep blue running (don't scale down yet)

✅ Post-Deployment:
  - [ ] Monitor for 30-60 minutes
  - [ ] Alert team of successful deployment
  - [ ] Scale down blue (optional)
  - [ ] Update green to be new blue
  - [ ] Document any issues
```

### Key Strategies

```
┌─────────────────────────────────────────────────────────┐
│         Deployment Strategy Comparison                  │
├─────────────┬──────────┬───────────┬──────────┬────────┤
│ Strategy    │ Downtime │ Rollback  │ Cost     │ Risk   │
├─────────────┼──────────┼───────────┼──────────┼────────┤
│ Blue-Green  │ None     │ Instant   │ High     │ Low    │
│ Canary      │ None     │ Fast      │ Medium   │ Low    │
│ Rolling     │ None     │ Slow      │ Low      │ Medium │
│ Recreate    │ Yes      │ Slow      │ Low      │ High   │
└─────────────┴──────────┴───────────┴──────────┴────────┘
```

### Critical Success Factors

1. **Health Checks**: Comprehensive and reliable
2. **Monitoring**: Real-time metrics and alerting
3. **Rollback Speed**: Automated and tested
4. **Database Strategy**: Backward-compatible migrations
5. **Cost Management**: Scale down idle environments

---

**Status**: ✅ Complete with production-ready blue-green deployment strategies for zero-downtime banking API releases!
