# APIs Interview Questions (Q24-Q26): DevOps & Deployment

## Q24: How do you containerize a Node.js banking API with Docker and implement multi-stage builds?

**Answer:**

Docker containerization ensures consistent deployments across environments and enables microservices architecture for banking applications.

### Docker Implementation:

```dockerfile
# ==========================
# MULTI-STAGE DOCKERFILE
# ==========================

# Stage 1: Build Stage
FROM node:18-alpine AS builder

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install ALL dependencies (including devDependencies for build)
RUN npm ci

# Copy source code
COPY . .

# Run build process (TypeScript compilation, etc.)
RUN npm run build

# Run tests
RUN npm run test

# Prune dev dependencies
RUN npm prune --production

# Stage 2: Production Stage
FROM node:18-alpine AS production

# Add metadata
LABEL maintainer="devops@emiratesnbd.com"
LABEL version="1.0.0"
LABEL description="ENBD Banking API"

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Set working directory
WORKDIR /app

# Copy only production node_modules from builder
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules

# Copy built application
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/package*.json ./

# Set NODE_ENV
ENV NODE_ENV=production
ENV PORT=3000

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

# Start application
CMD ["node", "dist/server.js"]

# ==========================
# DEVELOPMENT DOCKERFILE
# ==========================
# docker-compose.dev.yml uses this
FROM node:18-alpine AS development

WORKDIR /app

# Install development tools
RUN apk add --no-cache git

COPY package*.json ./
RUN npm install

COPY . .

USER node

EXPOSE 3000

CMD ["npm", "run", "dev"]
```

### Docker Compose for Multi-Service Setup:

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Node.js Banking API
  banking-api:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    container_name: enbd-banking-api
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
      PORT: 3000
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: enbd_banking
      DB_USER: postgres
      DB_PASSWORD: ${DB_PASSWORD}
      REDIS_HOST: redis
      REDIS_PORT: 6379
      JWT_SECRET: ${JWT_SECRET}
      OPENAI_API_KEY: ${OPENAI_API_KEY}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - banking-network
    volumes:
      - ./logs:/app/logs
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M

  # PostgreSQL Database
  postgres:
    image: postgres:15-alpine
    container_name: enbd-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: enbd_banking
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --locale=en_US.UTF-8"
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    networks:
      - banking-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: enbd-redis
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 256mb --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - banking-network
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  # Nginx Reverse Proxy & Load Balancer
  nginx:
    image: nginx:alpine
    container_name: enbd-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - nginx-logs:/var/log/nginx
    depends_on:
      - banking-api
    networks:
      - banking-network
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  postgres-data:
    driver: local
  redis-data:
    driver: local
  nginx-logs:
    driver: local

networks:
  banking-network:
    driver: bridge
```

### .dockerignore File:

```
# .dockerignore
node_modules
npm-debug.log
dist
build
.git
.gitignore
.env
.env.*
*.md
README.md
.vscode
.idea
coverage
.nyc_output
logs
*.log
.DS_Store
Thumbs.db
```

### Docker Security Best Practices:

```javascript
// docker-security-scan.js
const { exec } = require('child_process');
const util = require('util');
const execPromise = util.promisify(exec);

class DockerSecurityScanner {
    constructor(imageName) {
        this.imageName = imageName;
    }

    async scanWithTrivy() {
        console.log('🔍 Scanning image with Trivy...');
        
        try {
            const { stdout, stderr } = await execPromise(
                `trivy image --severity HIGH,CRITICAL ${this.imageName}`
            );
            
            console.log(stdout);
            
            if (stderr && stderr.includes('CRITICAL')) {
                throw new Error('Critical vulnerabilities found!');
            }
            
            return { passed: true, output: stdout };
        } catch (error) {
            console.error('❌ Trivy scan failed:', error.message);
            return { passed: false, error: error.message };
        }
    }

    async scanWithSnyk() {
        console.log('🔍 Scanning image with Snyk...');
        
        try {
            const { stdout } = await execPromise(
                `snyk container test ${this.imageName} --severity-threshold=high`
            );
            
            console.log(stdout);
            return { passed: true, output: stdout };
        } catch (error) {
            console.error('❌ Snyk scan failed:', error.message);
            return { passed: false, error: error.message };
        }
    }

    async checkDockerBestPractices() {
        console.log('📋 Checking Dockerfile best practices...');
        
        const checks = [
            {
                name: 'Non-root user',
                check: async () => {
                    const { stdout } = await execPromise(
                        `docker inspect ${this.imageName} --format='{{.Config.User}}'`
                    );
                    return stdout.trim() !== 'root' && stdout.trim() !== '';
                }
            },
            {
                name: 'Health check defined',
                check: async () => {
                    const { stdout } = await execPromise(
                        `docker inspect ${this.imageName} --format='{{.Config.Healthcheck}}'`
                    );
                    return stdout.trim() !== '<nil>';
                }
            },
            {
                name: 'Small image size',
                check: async () => {
                    const { stdout } = await execPromise(
                        `docker images ${this.imageName} --format='{{.Size}}'`
                    );
                    const sizeInMB = this.parseSize(stdout.trim());
                    return sizeInMB < 500; // Less than 500MB
                }
            },
            {
                name: 'No secrets in image',
                check: async () => {
                    const { stdout } = await execPromise(
                        `docker history ${this.imageName} --no-trunc`
                    );
                    const secrets = ['password', 'secret', 'key', 'token', 'api_key'];
                    return !secrets.some(s => stdout.toLowerCase().includes(s));
                }
            }
        ];

        const results = [];
        for (const { name, check } of checks) {
            try {
                const passed = await check();
                results.push({ name, passed, status: passed ? '✅' : '❌' });
                console.log(`${passed ? '✅' : '❌'} ${name}`);
            } catch (error) {
                results.push({ name, passed: false, status: '❌', error: error.message });
                console.log(`❌ ${name} - Error: ${error.message}`);
            }
        }

        return results;
    }

    parseSize(sizeStr) {
        const match = sizeStr.match(/^([\d.]+)(\w+)$/);
        if (!match) return 0;
        
        const [, value, unit] = match;
        const multipliers = {
            'B': 1 / (1024 * 1024),
            'KB': 1 / 1024,
            'MB': 1,
            'GB': 1024
        };
        
        return parseFloat(value) * (multipliers[unit] || 1);
    }

    async runAllScans() {
        console.log(`\n🔒 Running security scans for: ${this.imageName}\n`);
        
        const trivyResults = await this.scanWithTrivy();
        const snykResults = await this.scanWithSnyk();
        const bestPractices = await this.checkDockerBestPractices();
        
        const allPassed = trivyResults.passed && 
                         snykResults.passed && 
                         bestPractices.every(r => r.passed);
        
        console.log(`\n${allPassed ? '✅' : '❌'} Overall Security Status: ${allPassed ? 'PASSED' : 'FAILED'}\n`);
        
        return {
            passed: allPassed,
            trivy: trivyResults,
            snyk: snykResults,
            bestPractices
        };
    }
}

// Usage
if (require.main === module) {
    const scanner = new DockerSecurityScanner('enbd-banking-api:latest');
    scanner.runAllScans().then(results => {
        if (!results.passed) {
            process.exit(1);
        }
    });
}

module.exports = DockerSecurityScanner;
```

### Docker Build & Push Script:

```bash
#!/bin/bash
# build-and-push.sh

set -e

# Configuration
IMAGE_NAME="enbd-banking-api"
REGISTRY="registry.emiratesnbd.com"
VERSION="${1:-latest}"
PLATFORMS="linux/amd64,linux/arm64"

echo "🐳 Building Docker image: $IMAGE_NAME:$VERSION"

# Build multi-platform image
docker buildx build \
  --platform $PLATFORMS \
  --tag $REGISTRY/$IMAGE_NAME:$VERSION \
  --tag $REGISTRY/$IMAGE_NAME:latest \
  --cache-from type=registry,ref=$REGISTRY/$IMAGE_NAME:buildcache \
  --cache-to type=registry,ref=$REGISTRY/$IMAGE_NAME:buildcache,mode=max \
  --push \
  --progress=plain \
  .

echo "✅ Build complete"

# Security scan
echo "🔍 Running security scan..."
trivy image --severity HIGH,CRITICAL $REGISTRY/$IMAGE_NAME:$VERSION

echo "✅ Security scan complete"

# Push to registry
echo "📦 Image pushed to: $REGISTRY/$IMAGE_NAME:$VERSION"
```

---

## Q25: How do you deploy Node.js APIs to Kubernetes with auto-scaling and zero-downtime deployments?

**Answer:**

Kubernetes provides orchestration for containerized banking APIs with features like auto-scaling, load balancing, and rolling updates.

### Kubernetes Deployment Configuration:

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: enbd-banking-api
  namespace: banking
  labels:
    app: banking-api
    version: v1
    environment: production
spec:
  replicas: 3
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max 1 extra pod during update
      maxUnavailable: 0  # Zero downtime
  selector:
    matchLabels:
      app: banking-api
  template:
    metadata:
      labels:
        app: banking-api
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
    spec:
      # Security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      
      # Service account for RBAC
      serviceAccountName: banking-api-sa
      
      # Init container for database migrations
      initContainers:
      - name: db-migration
        image: registry.emiratesnbd.com/enbd-banking-api:latest
        command: ['npm', 'run', 'migrate']
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: banking-config
              key: db.host
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: banking-secrets
              key: db.password
      
      containers:
      - name: banking-api
        image: registry.emiratesnbd.com/enbd-banking-api:latest
        imagePullPolicy: Always
        
        ports:
        - name: http
          containerPort: 3000
          protocol: TCP
        
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "3000"
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: banking-config
              key: db.host
        - name: DB_NAME
          valueFrom:
            configMapKeyRef:
              name: banking-config
              key: db.name
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: banking-secrets
              key: db.user
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: banking-secrets
              key: db.password
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: banking-secrets
              key: jwt.secret
        - name: REDIS_HOST
          valueFrom:
            configMapKeyRef:
              name: banking-config
              key: redis.host
        
        # Resource limits
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        # Liveness probe (restart if unhealthy)
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
          successThreshold: 1
        
        # Readiness probe (remove from load balancer if not ready)
        readinessProbe:
          httpGet:
            path: /health/ready
            port: http
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 2
          successThreshold: 1
        
        # Startup probe (for slow-starting containers)
        startupProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 0
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 30
        
        # Volume mounts
        volumeMounts:
        - name: logs
          mountPath: /app/logs
        - name: config
          mountPath: /app/config
          readOnly: true
        
        # Security context
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1001
          capabilities:
            drop:
            - ALL
      
      # Volumes
      volumes:
      - name: logs
        emptyDir: {}
      - name: config
        configMap:
          name: banking-config
      
      # Image pull secrets
      imagePullSecrets:
      - name: registry-credentials
      
      # Affinity rules (spread across zones)
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - banking-api
              topologyKey: kubernetes.io/hostname
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role
                operator: In
                values:
                - api-worker

---
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: banking-api-service
  namespace: banking
  labels:
    app: banking-api
spec:
  type: ClusterIP
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 hours
  ports:
  - port: 80
    targetPort: http
    protocol: TCP
    name: http
  selector:
    app: banking-api

---
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: banking-api-hpa
  namespace: banking
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: enbd-banking-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
  # CPU-based scaling
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  # Memory-based scaling
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  # Custom metric: requests per second
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  # External metric: queue depth
  - type: External
    external:
      metric:
        name: rabbitmq_queue_depth
        selector:
          matchLabels:
            queue: transactions
      target:
        type: AverageValue
        averageValue: "100"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # 5 minutes cooldown
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Min
    scaleUp:
      stabilizationWindowSeconds: 0  # Immediate scale up
      policies:
      - type: Percent
        value: 50
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max

---
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: banking-config
  namespace: banking
data:
  db.host: "postgres-service.banking.svc.cluster.local"
  db.port: "5432"
  db.name: "enbd_banking"
  redis.host: "redis-service.banking.svc.cluster.local"
  redis.port: "6379"
  log.level: "info"
  api.timeout: "30000"
  rate.limit: "1000"

---
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: banking-secrets
  namespace: banking
type: Opaque
stringData:
  db.user: "postgres"
  db.password: "Sup3rS3cr3t!"
  jwt.secret: "jwt-secret-key-change-in-production"
  redis.password: "redis-password"
  openai.api.key: "sk-..."

---
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: banking-api-ingress
  namespace: banking
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/limit-rps: "50"
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://emiratesnbd.com"
spec:
  tls:
  - hosts:
    - api.emiratesnbd.com
    secretName: banking-api-tls
  rules:
  - host: api.emiratesnbd.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: banking-api-service
            port:
              number: 80

---
# k8s/pdb.yaml (Pod Disruption Budget)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: banking-api-pdb
  namespace: banking
spec:
  minAvailable: 2  # Always keep at least 2 pods running
  selector:
    matchLabels:
      app: banking-api
```

### Kubernetes Deployment Script:

```javascript
// deploy.js
const k8s = require('@kubernetes/client-node');
const fs = require('fs');
const yaml = require('js-yaml');

class KubernetesDeployer {
    constructor() {
        this.kc = new k8s.KubeConfig();
        this.kc.loadFromDefault();
        
        this.appsApi = this.kc.makeApiClient(k8s.AppsV1Api);
        this.coreApi = this.kc.makeApiClient(k8s.CoreV1Api);
        this.autoscalingApi = this.kc.makeApiClient(k8s.AutoscalingV2Api);
    }

    async deployApplication(manifestsDir, namespace = 'banking') {
        console.log(`🚀 Deploying to namespace: ${namespace}`);
        
        try {
            // Ensure namespace exists
            await this.ensureNamespace(namespace);
            
            // Apply manifests in order
            const order = [
                'configmap.yaml',
                'secret.yaml',
                'deployment.yaml',
                'service.yaml',
                'hpa.yaml',
                'ingress.yaml',
                'pdb.yaml'
            ];
            
            for (const file of order) {
                const filePath = `${manifestsDir}/${file}`;
                if (fs.existsSync(filePath)) {
                    await this.applyManifest(filePath, namespace);
                }
            }
            
            // Wait for rollout to complete
            await this.waitForRollout('enbd-banking-api', namespace);
            
            console.log('✅ Deployment successful');
            return { success: true };
            
        } catch (error) {
            console.error('❌ Deployment failed:', error.message);
            throw error;
        }
    }

    async ensureNamespace(namespace) {
        try {
            await this.coreApi.readNamespace(namespace);
            console.log(`✓ Namespace ${namespace} exists`);
        } catch (error) {
            if (error.response?.statusCode === 404) {
                console.log(`Creating namespace: ${namespace}`);
                await this.coreApi.createNamespace({
                    metadata: { name: namespace }
                });
            } else {
                throw error;
            }
        }
    }

    async applyManifest(filePath, namespace) {
        console.log(`📄 Applying: ${filePath}`);
        
        const fileContent = fs.readFileSync(filePath, 'utf8');
        const manifests = yaml.loadAll(fileContent);
        
        for (const manifest of manifests) {
            if (!manifest) continue;
            
            manifest.metadata = manifest.metadata || {};
            manifest.metadata.namespace = namespace;
            
            await this.applyResource(manifest);
        }
    }

    async applyResource(manifest) {
        const { kind, metadata } = manifest;
        const name = metadata.name;
        const namespace = metadata.namespace;
        
        try {
            switch (kind) {
                case 'Deployment':
                    await this.appsApi.replaceNamespacedDeployment(
                        name, namespace, manifest
                    );
                    break;
                    
                case 'Service':
                    await this.coreApi.replaceNamespacedService(
                        name, namespace, manifest
                    );
                    break;
                    
                case 'ConfigMap':
                    await this.coreApi.replaceNamespacedConfigMap(
                        name, namespace, manifest
                    );
                    break;
                    
                case 'Secret':
                    await this.coreApi.replaceNamespacedSecret(
                        name, namespace, manifest
                    );
                    break;
                    
                case 'HorizontalPodAutoscaler':
                    await this.autoscalingApi.replaceNamespacedHorizontalPodAutoscaler(
                        name, namespace, manifest
                    );
                    break;
            }
            
            console.log(`  ✓ ${kind}: ${name}`);
            
        } catch (error) {
            if (error.response?.statusCode === 404) {
                // Resource doesn't exist, create it
                await this.createResource(manifest);
            } else {
                throw error;
            }
        }
    }

    async createResource(manifest) {
        const { kind, metadata } = manifest;
        const namespace = metadata.namespace;
        
        switch (kind) {
            case 'Deployment':
                await this.appsApi.createNamespacedDeployment(namespace, manifest);
                break;
            case 'Service':
                await this.coreApi.createNamespacedService(namespace, manifest);
                break;
            case 'ConfigMap':
                await this.coreApi.createNamespacedConfigMap(namespace, manifest);
                break;
            case 'Secret':
                await this.coreApi.createNamespacedSecret(namespace, manifest);
                break;
            case 'HorizontalPodAutoscaler':
                await this.autoscalingApi.createNamespacedHorizontalPodAutoscaler(namespace, manifest);
                break;
        }
    }

    async waitForRollout(deploymentName, namespace, timeout = 300000) {
        console.log(`⏳ Waiting for rollout: ${deploymentName}`);
        
        const startTime = Date.now();
        
        while (Date.now() - startTime < timeout) {
            try {
                const { body } = await this.appsApi.readNamespacedDeployment(
                    deploymentName, namespace
                );
                
                const { status } = body;
                const { replicas = 0, updatedReplicas = 0, readyReplicas = 0 } = status;
                
                console.log(`  Replicas: ${readyReplicas}/${replicas} ready, ${updatedReplicas} updated`);
                
                if (replicas === updatedReplicas && replicas === readyReplicas) {
                    console.log('✅ Rollout complete');
                    return true;
                }
                
            } catch (error) {
                console.error('Error checking rollout status:', error.message);
            }
            
            await new Promise(resolve => setTimeout(resolve, 5000));
        }
        
        throw new Error('Rollout timeout');
    }

    async rollback(deploymentName, namespace, revision = null) {
        console.log(`🔄 Rolling back deployment: ${deploymentName}`);
        
        // Get rollout history
        const { body } = await this.appsApi.readNamespacedDeployment(
            deploymentName, namespace
        );
        
        // Rollback to specific revision or previous
        const rollbackRevision = revision || (body.metadata.generation - 1);
        
        await this.appsApi.patchNamespacedDeployment(
            deploymentName,
            namespace,
            {
                spec: {
                    rollbackTo: { revision: rollbackRevision }
                }
            },
            undefined,
            undefined,
            undefined,
            undefined,
            { headers: { 'Content-Type': 'application/strategic-merge-patch+json' } }
        );
        
        console.log(`✅ Rolled back to revision: ${rollbackRevision}`);
    }
}

// Usage
if (require.main === module) {
    const deployer = new KubernetesDeployer();
    deployer.deployApplication('./k8s', 'banking').catch(console.error);
}

module.exports = KubernetesDeployer;
```

---

## Q26: How do you implement CI/CD pipelines for banking APIs with automated testing and deployment?

**Answer:**

CI/CD pipelines automate testing, building, and deployment of banking APIs, ensuring code quality and rapid delivery.

### GitHub Actions CI/CD Pipeline:

```yaml
# .github/workflows/ci-cd.yml
name: ENBD Banking API CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  release:
    types: [created]

env:
  NODE_VERSION: '18.x'
  REGISTRY: registry.emiratesnbd.com
  IMAGE_NAME: enbd-banking-api

jobs:
  # Job 1: Lint and Test
  test:
    name: Test & Quality Check
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
      with:
        fetch-depth: 0  # Full history for SonarQube
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: ${{ env.NODE_VERSION }}
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run linting
      run: npm run lint
    
    - name: Run TypeScript type check
      run: npm run type-check
    
    - name: Run unit tests
      run: npm run test:unit
      env:
        DB_HOST: localhost
        DB_PORT: 5432
        DB_NAME: test_db
        DB_USER: postgres
        DB_PASSWORD: postgres
        REDIS_HOST: localhost
        REDIS_PORT: 6379
    
    - name: Run integration tests
      run: npm run test:integration
      env:
        DB_HOST: localhost
        REDIS_HOST: localhost
    
    - name: Run E2E tests
      run: npm run test:e2e
      env:
        API_URL: http://localhost:3000
    
    - name: Generate coverage report
      run: npm run test:coverage
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        files: ./coverage/lcov.info
        flags: unittests
        name: codecov-umbrella
    
    - name: SonarQube Scan
      uses: sonarsource/sonarqube-scan-action@master
      env:
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
    
    - name: Check Quality Gate
      uses: sonarsource/sonarqube-quality-gate-action@master
      timeout-minutes: 5
      env:
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  # Job 2: Security Scan
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Run npm audit
      run: npm audit --audit-level=high
      continue-on-error: true
    
    - name: Run Snyk security scan
      uses: snyk/actions/node@master
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      with:
        args: --severity-threshold=high
    
    - name: Run OWASP Dependency Check
      uses: dependency-check/Dependency-Check_Action@main
      with:
        project: 'enbd-banking-api'
        path: '.'
        format: 'HTML'
    
    - name: Upload dependency check results
      uses: actions/upload-artifact@v3
      with:
        name: dependency-check-report
        path: dependency-check-report.html

  # Job 3: Build Docker Image
  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [test, security]
    if: github.event_name != 'pull_request'
    
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Log in to Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ secrets.REGISTRY_USERNAME }}
        password: ${{ secrets.REGISTRY_PASSWORD }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
          type=sha,prefix={{branch}}-
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache
        cache-to: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache,mode=max
        build-args: |
          NODE_ENV=production
          BUILD_DATE=${{ github.event.head_commit.timestamp }}
          VCS_REF=${{ github.sha }}
    
    - name: Scan Docker image with Trivy
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'CRITICAL,HIGH'
    
    - name: Upload Trivy results to GitHub Security
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'

  # Job 4: Deploy to Staging
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging-api.emiratesnbd.com
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Setup kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'v1.28.0'
    
    - name: Configure kubectl
      run: |
        echo "${{ secrets.KUBE_CONFIG_STAGING }}" | base64 -d > kubeconfig
        export KUBECONFIG=./kubeconfig
    
    - name: Deploy to Kubernetes
      run: |
        kubectl set image deployment/enbd-banking-api \
          banking-api=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
          -n banking-staging
        
        kubectl rollout status deployment/enbd-banking-api -n banking-staging --timeout=5m
    
    - name: Run smoke tests
      run: |
        npm ci
        npm run test:smoke
      env:
        API_URL: https://staging-api.emiratesnbd.com
    
    - name: Notify deployment
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        text: 'Deployed to staging: ${{ github.sha }}'
        webhook_url: ${{ secrets.SLACK_WEBHOOK }}

  # Job 5: Deploy to Production
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: build
    if: github.event_name == 'release'
    environment:
      name: production
      url: https://api.emiratesnbd.com
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Setup kubectl
      uses: azure/setup-kubectl@v3
    
    - name: Configure kubectl
      run: |
        echo "${{ secrets.KUBE_CONFIG_PRODUCTION }}" | base64 -d > kubeconfig
        export KUBECONFIG=./kubeconfig
    
    - name: Deploy with Blue-Green strategy
      run: |
        # Deploy green environment
        kubectl apply -f k8s/deployment-green.yaml -n banking-prod
        kubectl rollout status deployment/enbd-banking-api-green -n banking-prod --timeout=10m
        
        # Run smoke tests on green
        npm ci && npm run test:smoke
        
        # Switch traffic to green
        kubectl patch service banking-api-service -n banking-prod \
          -p '{"spec":{"selector":{"version":"green"}}}'
        
        # Wait for traffic switch
        sleep 30
        
        # Delete blue deployment
        kubectl delete deployment enbd-banking-api-blue -n banking-prod
      env:
        API_URL: https://api.emiratesnbd.com
    
    - name: Create Datadog deployment marker
      run: |
        curl -X POST "https://api.datadoghq.com/api/v1/events" \
          -H "DD-API-KEY: ${{ secrets.DATADOG_API_KEY }}" \
          -d @- << EOF
        {
          "title": "Banking API Deployment",
          "text": "Deployed version ${{ github.ref }}",
          "tags": ["environment:production", "service:banking-api"]
        }
        EOF
    
    - name: Notify success
      uses: 8398a7/action-slack@v3
      with:
        status: 'success'
        text: '🚀 Production deployment successful!'
        webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

### Jenkins Pipeline (Alternative):

```groovy
// Jenkinsfile
pipeline {
    agent any
    
    environment {
        REGISTRY = 'registry.emiratesnbd.com'
        IMAGE_NAME = 'enbd-banking-api'
        NODE_VERSION = '18'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Install Dependencies') {
            steps {
                nodejs(nodeJSInstallationName: "Node ${NODE_VERSION}") {
                    sh 'npm ci'
                }
            }
        }
        
        stage('Lint & Type Check') {
            parallel {
                stage('ESLint') {
                    steps {
                        sh 'npm run lint'
                    }
                }
                stage('TypeScript') {
                    steps {
                        sh 'npm run type-check'
                    }
                }
            }
        }
        
        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'npm run test:unit'
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh 'npm run test:integration'
                    }
                }
            }
            post {
                always {
                    junit 'test-results/**/*.xml'
                    publishHTML([
                        reportDir: 'coverage',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                sh 'npm audit --audit-level=high'
                snykSecurity(
                    snykInstallation: 'Snyk',
                    snykTokenId: 'snyk-api-token',
                    severity: 'high'
                )
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    docker.withRegistry("https://${REGISTRY}", 'registry-credentials') {
                        def image = docker.build("${IMAGE_NAME}:${BUILD_NUMBER}")
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                kubernetesDeploy(
                    configs: 'k8s/*.yaml',
                    kubeconfigId: 'kubeconfig-staging',
                    enableConfigSubstitution: true
                )
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Deploy to production?', ok: 'Deploy'
                kubernetesDeploy(
                    configs: 'k8s/*.yaml',
                    kubeconfigId: 'kubeconfig-production',
                    enableConfigSubstitution: true
                )
            }
        }
    }
    
    post {
        success {
            slackSend(
                color: 'good',
                message: "Build ${BUILD_NUMBER} succeeded!"
            )
        }
        failure {
            slackSend(
                color: 'danger',
                message: "Build ${BUILD_NUMBER} failed!"
            )
        }
    }
}
```

---
