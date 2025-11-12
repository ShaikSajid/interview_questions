# Service Discovery & Service Mesh

## Question 13: AWS Cloud Map Service Discovery

### 📋 Question Statement

Implement dynamic service discovery for Emirates NBD microservices using AWS Cloud Map. Include service registration, health checks, DNS-based discovery, and integration with ECS and EKS.

---

### 🏗️ Service Discovery Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│              EMIRATES NBD SERVICE DISCOVERY ARCHITECTURE                    │
└────────────────────────────────────────────────────────────────────────────┘

                        ┌──────────────────────┐
                        │   Route 53 Hosted    │
                        │       Zone           │
                        │ (emiratesnbd.local)  │
                        └──────────┬───────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │     AWS Cloud Map            │
                    │  (Service Registry)          │
                    │                              │
                    │  Namespaces:                 │
                    │  • emiratesnbd.local (DNS)   │
                    │  • api.emiratesnbd.local     │
                    │                              │
                    │  Services:                   │
                    │  • account-service           │
                    │  • transaction-service       │
                    │  • payment-service           │
                    │  • loan-service              │
                    └──────────────┬───────────────┘
                                   │
            ┌──────────────────────┼──────────────────────┐
            │                      │                      │
            ▼                      ▼                      ▼
    ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
    │   ECS Tasks  │      │  EKS Pods    │      │   Lambda     │
    │              │      │              │      │  Functions   │
    │ • Auto-      │      │ • CoreDNS    │      │              │
    │   Register   │      │   Integration│      │ • Custom     │
    │ • Health     │      │ • ExternalDNS│      │   Register   │
    │   Checks     │      │ • Headless   │      │              │
    │              │      │   Service    │      │              │
    └──────────────┘      └──────────────┘      └──────────────┘
            │                      │                      │
            └──────────────────────┼──────────────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │    Service Discovery         │
                    │    Client SDKs               │
                    │                              │
                    │  • DNS Resolution            │
                    │  • API-based Discovery       │
                    │  • Health-aware Routing      │
                    └──────────────────────────────┘

    DISCOVERY FLOW:
    ───────────────
    
    1. Service Registration:
       ECS Task → Cloud Map → Route 53 (A/SRV records)
    
    2. Service Discovery:
       Consumer → DNS Query → Cloud Map → Healthy Instance IP
    
    3. Health Monitoring:
       Cloud Map → HTTP/TCP Check → Mark Unhealthy → Remove from DNS
```

---

### 🔧 Cloud Map Infrastructure (CDK)

```typescript
// infrastructure/cdk/cloud-map-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as servicediscovery from 'aws-cdk-lib/aws-servicediscovery';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as route53 from 'aws-cdk-lib/aws-route53';

export class CloudMapStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // ============================================
    // 1. VPC
    // ============================================
    const vpc = new ec2.Vpc(this, 'BankingVPC', {
      maxAzs: 3
    });

    // ============================================
    // 2. CLOUD MAP NAMESPACE (Private DNS)
    // ============================================
    const privateNamespace = new servicediscovery.PrivateDnsNamespace(this, 'PrivateNamespace', {
      name: 'emiratesnbd.local',
      vpc,
      description: 'Private namespace for Emirates NBD services'
    });

    // Public namespace for external access
    const publicNamespace = new servicediscovery.PublicDnsNamespace(this, 'PublicNamespace', {
      name: 'api.emiratesnbd.com',
      description: 'Public namespace for Emirates NBD API services'
    });

    // HTTP namespace (API-based discovery, no DNS)
    const httpNamespace = new servicediscovery.HttpNamespace(this, 'HttpNamespace', {
      name: 'emiratesnbd-http',
      description: 'HTTP namespace for API-based service discovery'
    });

    // ============================================
    // 3. CLOUD MAP SERVICES
    // ============================================

    // Account Service
    const accountService = privateNamespace.createService('AccountService', {
      name: 'account-service',
      description: 'Account management service',
      dnsRecordType: servicediscovery.DnsRecordType.A,
      dnsTtl: cdk.Duration.seconds(30),
      healthCheck: {
        type: servicediscovery.HealthCheckType.HTTP,
        resourcePath: '/health',
        failureThreshold: 2
      },
      customHealthCheck: {
        failureThreshold: 1
      }
    });

    // Transaction Service with SRV records
    const transactionService = privateNamespace.createService('TransactionService', {
      name: 'transaction-service',
      dnsRecordType: servicediscovery.DnsRecordType.A_AAAA,
      dnsTtl: cdk.Duration.seconds(60),
      healthCheck: {
        type: servicediscovery.HealthCheckType.TCP,
        failureThreshold: 3
      }
    });

    // Payment Service
    const paymentService = privateNamespace.createService('PaymentService', {
      name: 'payment-service',
      dnsRecordType: servicediscovery.DnsRecordType.A,
      dnsTtl: cdk.Duration.seconds(30),
      healthCheck: {
        type: servicediscovery.HealthCheckType.HTTP,
        resourcePath: '/healthz',
        failureThreshold: 2
      }
    });

    // Loan Service (HTTP namespace for API discovery)
    const loanService = httpNamespace.createService('LoanService', {
      name: 'loan-service',
      description: 'Loan processing service'
    });

    // ============================================
    // 4. ECS CLUSTER WITH CLOUD MAP INTEGRATION
    // ============================================

    const ecsCluster = new ecs.Cluster(this, 'BankingCluster', {
      vpc,
      clusterName: 'banking-cluster',
      defaultCloudMapNamespace: {
        name: 'emiratesnbd.local',
        type: servicediscovery.NamespaceType.DNS_PRIVATE,
        vpc
      }
    });

    // ============================================
    // 5. ECS SERVICE WITH AUTO-REGISTRATION
    // ============================================

    const accountTaskDef = new ecs.FargateTaskDefinition(this, 'AccountTask', {
      memoryLimitMiB: 512,
      cpu: 256
    });

    accountTaskDef.addContainer('AccountContainer', {
      image: ecs.ContainerImage.fromRegistry('account-service:latest'),
      portMappings: [{ containerPort: 3000 }],
      logging: ecs.LogDrivers.awsLogs({ streamPrefix: 'account-service' }),
      environment: {
        SERVICE_NAME: 'account-service',
        NAMESPACE: 'emiratesnbd.local'
      },
      healthCheck: {
        command: ['CMD-SHELL', 'curl -f http://localhost:3000/health || exit 1'],
        interval: cdk.Duration.seconds(30),
        timeout: cdk.Duration.seconds(5),
        retries: 3
      }
    });

    const accountEcsService = new ecs.FargateService(this, 'AccountECSService', {
      cluster: ecsCluster,
      taskDefinition: accountTaskDef,
      desiredCount: 2,
      cloudMapOptions: {
        name: 'account-service',
        cloudMapNamespace: privateNamespace,
        dnsRecordType: servicediscovery.DnsRecordType.A,
        dnsTtl: cdk.Duration.seconds(30),
        failureThreshold: 1
      }
    });

    // Payment Service on ECS
    const paymentTaskDef = new ecs.FargateTaskDefinition(this, 'PaymentTask', {
      memoryLimitMiB: 1024,
      cpu: 512
    });

    paymentTaskDef.addContainer('PaymentContainer', {
      image: ecs.ContainerImage.fromRegistry('payment-service:latest'),
      portMappings: [{ containerPort: 8080 }],
      logging: ecs.LogDrivers.awsLogs({ streamPrefix: 'payment-service' }),
      environment: {
        ACCOUNT_SERVICE_URL: 'http://account-service.emiratesnbd.local:3000',
        TRANSACTION_SERVICE_URL: 'http://transaction-service.emiratesnbd.local:4000'
      },
      healthCheck: {
        command: ['CMD-SHELL', 'curl -f http://localhost:8080/healthz || exit 1'],
        interval: cdk.Duration.seconds(30),
        timeout: cdk.Duration.seconds(5),
        retries: 3
      }
    });

    new ecs.FargateService(this, 'PaymentECSService', {
      cluster: ecsCluster,
      taskDefinition: paymentTaskDef,
      desiredCount: 3,
      cloudMapOptions: {
        name: 'payment-service',
        cloudMapNamespace: privateNamespace
      }
    });

    // ============================================
    // 6. OUTPUTS
    // ============================================

    new cdk.CfnOutput(this, 'PrivateNamespaceId', {
      value: privateNamespace.namespaceId,
      description: 'Private DNS Namespace ID'
    });

    new cdk.CfnOutput(this, 'PrivateNamespaceName', {
      value: privateNamespace.namespaceName,
      description: 'Private DNS Namespace Name'
    });

    new cdk.CfnOutput(this, 'AccountServiceArn', {
      value: accountService.serviceArn,
      description: 'Account Service ARN'
    });
  }
}
```

---

### 💻 Service Registration (Custom Lambda)

```javascript
// lambdas/service-register/index.js
const { ServiceDiscoveryClient, RegisterInstanceCommand, DeregisterInstanceCommand } = require('@aws-sdk/client-servicediscovery');

const client = new ServiceDiscoveryClient({ region: 'us-east-1' });

exports.handler = async (event) => {
  console.log('Event:', JSON.stringify(event, null, 2));

  const { action, serviceId, instanceId, ipv4, port, attributes } = event;

  try {
    if (action === 'register') {
      return await registerInstance({ serviceId, instanceId, ipv4, port, attributes });
    } else if (action === 'deregister') {
      return await deregisterInstance({ serviceId, instanceId });
    } else {
      throw new Error(`Unknown action: ${action}`);
    }
  } catch (error) {
    console.error('Error:', error);
    throw error;
  }
};

async function registerInstance({ serviceId, instanceId, ipv4, port, attributes = {} }) {
  const response = await client.send(new RegisterInstanceCommand({
    ServiceId: serviceId,
    InstanceId: instanceId,
    Attributes: {
      AWS_INSTANCE_IPV4: ipv4,
      AWS_INSTANCE_PORT: port.toString(),
      ...attributes
    }
  }));

  console.log('Instance registered:', response.OperationId);
  return { statusCode: 200, operationId: response.OperationId };
}

async function deregisterInstance({ serviceId, instanceId }) {
  const response = await client.send(new DeregisterInstanceCommand({
    ServiceId: serviceId,
    InstanceId: instanceId
  }));

  console.log('Instance deregistered:', response.OperationId);
  return { statusCode: 200, operationId: response.OperationId };
}
```

---

### 🔍 Service Discovery Client

```javascript
// clients/service-discovery-client.js
const { ServiceDiscoveryClient, DiscoverInstancesCommand, ListInstancesCommand } = require('@aws-sdk/client-servicediscovery');
const dns = require('dns').promises;

class ServiceDiscoveryClient {
  constructor({ region = 'us-east-1', namespace = 'emiratesnbd.local' }) {
    this.client = new ServiceDiscoveryClient({ region });
    this.namespace = namespace;
    this.cache = new Map();
    this.cacheTTL = 30000; // 30 seconds
  }

  // DNS-based discovery
  async discoverByDNS(serviceName) {
    const fqdn = `${serviceName}.${this.namespace}`;
    
    try {
      const addresses = await dns.resolve4(fqdn);
      return addresses.map(ip => ({ ip, port: 3000 })); // Default port
    } catch (error) {
      console.error(`DNS resolution failed for ${fqdn}:`, error);
      throw error;
    }
  }

  // API-based discovery with health-aware routing
  async discoverInstances(serviceName, healthStatusFilter = 'HEALTHY') {
    const cacheKey = `${serviceName}-${healthStatusFilter}`;
    
    // Check cache
    const cached = this.cache.get(cacheKey);
    if (cached && Date.now() - cached.timestamp < this.cacheTTL) {
      return cached.instances;
    }

    try {
      const response = await this.client.send(new DiscoverInstancesCommand({
        NamespaceName: this.namespace,
        ServiceName: serviceName,
        HealthStatus: healthStatusFilter,
        MaxResults: 100
      }));

      const instances = response.Instances.map(inst => ({
        id: inst.InstanceId,
        ip: inst.Attributes.AWS_INSTANCE_IPV4,
        port: parseInt(inst.Attributes.AWS_INSTANCE_PORT || '3000'),
        attributes: inst.Attributes,
        healthStatus: inst.HealthStatus
      }));

      // Update cache
      this.cache.set(cacheKey, {
        instances,
        timestamp: Date.now()
      });

      return instances;
    } catch (error) {
      console.error(`Failed to discover instances for ${serviceName}:`, error);
      throw error;
    }
  }

  // Get a single instance (with load balancing)
  async getInstance(serviceName, strategy = 'random') {
    const instances = await this.discoverInstances(serviceName);
    
    if (instances.length === 0) {
      throw new Error(`No healthy instances found for ${serviceName}`);
    }

    switch (strategy) {
      case 'random':
        return instances[Math.floor(Math.random() * instances.length)];
      case 'round-robin':
        return this.getRoundRobinInstance(serviceName, instances);
      default:
        return instances[0];
    }
  }

  getRoundRobinInstance(serviceName, instances) {
    if (!this.roundRobinCounters) {
      this.roundRobinCounters = new Map();
    }

    const counter = this.roundRobinCounters.get(serviceName) || 0;
    const instance = instances[counter % instances.length];
    
    this.roundRobinCounters.set(serviceName, counter + 1);
    return instance;
  }

  // Build service URL
  async getServiceUrl(serviceName, path = '') {
    const instance = await this.getInstance(serviceName);
    return `http://${instance.ip}:${instance.port}${path}`;
  }

  // Clear cache
  clearCache() {
    this.cache.clear();
  }
}

module.exports = ServiceDiscoveryClient;

// Example usage
/*
const ServiceDiscoveryClient = require('./service-discovery-client');

const discoveryClient = new ServiceDiscoveryClient({
  region: 'us-east-1',
  namespace: 'emiratesnbd.local'
});

async function callAccountService() {
  try {
    // Get service URL
    const url = await discoveryClient.getServiceUrl('account-service', '/api/accounts');
    
    // Make HTTP request
    const response = await fetch(url);
    const data = await response.json();
    
    console.log('Account data:', data);
  } catch (error) {
    console.error('Error calling account service:', error);
  }
}

callAccountService();
*/
```

---

### 🐳 ECS Task with Service Discovery

```javascript
// services/account-service/src/server.js
const express = require('express');
const ServiceDiscoveryClient = require('./clients/service-discovery-client');

const app = express();
const PORT = process.env.PORT || 3000;

// Initialize service discovery client
const discoveryClient = new ServiceDiscoveryClient({
  namespace: process.env.NAMESPACE || 'emiratesnbd.local'
});

app.use(express.json());

// Health check endpoint (for Cloud Map)
app.get('/health', (req, res) => {
  res.status(200).json({
    status: 'healthy',
    service: 'account-service',
    timestamp: new Date().toISOString()
  });
});

// Account endpoints
app.get('/api/accounts/:id', async (req, res) => {
  try {
    // Get account from database
    const account = await getAccount(req.params.id);
    
    // Call transaction service to get recent transactions
    const transactionUrl = await discoveryClient.getServiceUrl(
      'transaction-service',
      `/api/transactions?accountId=${req.params.id}`
    );
    
    const transactionsResponse = await fetch(transactionUrl);
    const transactions = await transactionsResponse.json();
    
    res.json({
      account,
      recentTransactions: transactions
    });
  } catch (error) {
    console.error('Error:', error);
    res.status(500).json({ error: error.message });
  }
});

app.post('/api/accounts', async (req, res) => {
  try {
    const account = await createAccount(req.body);
    
    // Publish event to EventBridge
    // ... (event publishing code)
    
    res.status(201).json(account);
  } catch (error) {
    console.error('Error:', error);
    res.status(500).json({ error: error.message });
  }
});

// Graceful shutdown
process.on('SIGTERM', () => {
  console.log('SIGTERM received, shutting down gracefully');
  server.close(() => {
    console.log('Server closed');
    process.exit(0);
  });
});

const server = app.listen(PORT, () => {
  console.log(`Account service listening on port ${PORT}`);
  console.log(`Service discovery namespace: ${process.env.NAMESPACE}`);
});
```

---

### ☸️ Kubernetes Integration with ExternalDNS

```yaml
# kubernetes/externaldns-deployment.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
rules:
- apiGroups: [""]
  resources: ["services", "endpoints", "pods"]
  verbs: ["get", "watch", "list"]
- apiGroups: ["extensions", "networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "watch", "list"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
- kind: ServiceAccount
  name: external-dns
  namespace: kube-system
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: registry.k8s.io/external-dns/external-dns:v0.13.5
        args:
        - --source=service
        - --source=ingress
        - --provider=aws-sd
        - --aws-zone-type=public
        - --aws-zone-type=private
        - --registry=txt
        - --txt-owner-id=eks-cluster
        - --log-level=info
        env:
        - name: AWS_DEFAULT_REGION
          value: us-east-1
---
# Payment Service with Cloud Map registration
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: banking
  annotations:
    external-dns.alpha.kubernetes.io/hostname: payment-service.emiratesnbd.local
    external-dns.alpha.kubernetes.io/aws-sd-service-id: srv-xxxxxxxxxxxxx
spec:
  type: ClusterIP
  selector:
    app: payment-service
  ports:
  - port: 8080
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: banking
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      containers:
      - name: payment-service
        image: payment-service:latest
        ports:
        - containerPort: 8080
        env:
        - name: ACCOUNT_SERVICE_URL
          value: "http://account-service.emiratesnbd.local:3000"
        - name: TRANSACTION_SERVICE_URL
          value: "http://transaction-service.emiratesnbd.local:4000"
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

---

### 🎓 Interview Discussion Points - Q13

**Q1: What's the difference between Cloud Map and Route 53?**

**A**:
- **Cloud Map**: Service discovery with health checks, API-based discovery, integration with ECS/EKS
- **Route 53**: DNS service, supports routing policies (weighted, latency), no automatic service registration

**Q2: When should you use DNS-based vs API-based discovery?**

**A**:
- **DNS**: Simple, low overhead, cached by OS, works with any HTTP client
- **API**: Health-aware, real-time updates, supports custom attributes, better for frequent changes

**Q3: How do you handle service discovery in multi-region deployments?**

**A**:
- Use Route 53 for global DNS with latency-based routing
- Create separate Cloud Map namespaces per region
- Use API Gateway regional endpoints
- Implement circuit breakers for failover

**Q4: What are Cloud Map limitations?**

**A**:
- **Instances per service**: 1,000 (soft limit)
- **Services per namespace**: 50 (soft limit)
- **API calls**: 2,000 req/sec (soft limit)
- **Health check interval**: Minimum 30 seconds

---

## Question 14: AWS App Mesh for Microservices

### 📋 Question Statement

Implement AWS App Mesh for Emirates NBD to provide service mesh capabilities including traffic management, observability, security, and resilience patterns (circuit breakers, retries, timeouts).

---

### 🏗️ App Mesh Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                   EMIRATES NBD APP MESH ARCHITECTURE                        │
└────────────────────────────────────────────────────────────────────────────┘

                           ┌─────────────────┐
                           │   App Mesh      │
                           │   Control Plane │
                           └────────┬────────┘
                                    │
                     ┌──────────────┼──────────────┐
                     │              │              │
                     ▼              ▼              ▼
            ┌────────────────────────────────────────────┐
            │        Virtual Gateway (Ingress)           │
            │  • TLS Termination                         │
            │  • Request Routing                         │
            │  • Rate Limiting                           │
            └────────────────┬───────────────────────────┘
                             │
                 ┌───────────┼───────────┐
                 │           │           │
                 ▼           ▼           ▼
        ┌──────────────────────────────────────────────────────┐
        │             VIRTUAL SERVICES (Logical)               │
        │  • account-service.emiratesnbd.local                 │
        │  • transaction-service.emiratesnbd.local             │
        │  • payment-service.emiratesnbd.local                 │
        └──────────────────┬───────────────────────────────────┘
                           │
            ┌──────────────┼──────────────┐
            │              │              │
            ▼              ▼              ▼
   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
   │  Virtual     │ │  Virtual     │ │  Virtual     │
   │  Router      │ │  Router      │ │  Router      │
   │  (Routing)   │ │  (Canary)    │ │  (A/B Test)  │
   └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
          │                │                │
          ▼                ▼                ▼
   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
   │  Virtual     │ │  Virtual     │ │  Virtual     │
   │  Node v1     │ │  Node v1     │ │  Node v1     │
   │  (90%)       │ │  (50%)       │ │  (Group A)   │
   └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
          │                │                │
   ┌──────┴───────┐ ┌──────┴───────┐ ┌──────┴───────┐
   │  Envoy Proxy │ │  Envoy Proxy │ │  Envoy Proxy │
   │  Sidecar     │ │  Sidecar     │ │  Sidecar     │
   └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
          │                │                │
   ┌──────┴───────┐ ┌──────┴───────┐ ┌──────┴───────┐
   │  ECS Task    │ │  ECS Task    │ │  EKS Pod     │
   │  (App)       │ │  (App)       │ │  (App)       │
   └──────────────┘ └──────────────┘ └──────────────┘

   TRAFFIC FLOW WITH RESILIENCE:
   ──────────────────────────────
   
   Client Request
       │
       ▼
   Virtual Gateway (TLS, Auth)
       │
       ▼
   Virtual Router (Route, Retry, Timeout)
       │
       ▼
   Envoy Proxy (Circuit Breaker, Load Balance)
       │
       ▼
   Application Container

   OBSERVABILITY:
   ──────────────
   Envoy → X-Ray (Traces)
   Envoy → CloudWatch (Metrics)
   Envoy → CloudWatch Logs (Access Logs)
```

---

### 🔧 App Mesh Infrastructure (CDK)

```typescript
// infrastructure/cdk/app-mesh-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as appmesh from 'aws-cdk-lib/aws-appmesh';
import * as servicediscovery from 'aws-cdk-lib/aws-servicediscovery';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as acm from 'aws-cdk-lib/aws-certificatemanager';

export class AppMeshStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // ============================================
    // 1. VPC & CLOUD MAP
    // ============================================
    const vpc = new ec2.Vpc(this, 'BankingVPC', { maxAzs: 3 });

    const privateNamespace = new servicediscovery.PrivateDnsNamespace(this, 'Namespace', {
      name: 'emiratesnbd.local',
      vpc
    });

    // ============================================
    // 2. APP MESH
    // ============================================
    const mesh = new appmesh.Mesh(this, 'BankingMesh', {
      meshName: 'banking-mesh',
      egressFilter: appmesh.MeshFilterType.ALLOW_ALL
    });

    // ============================================
    // 3. VIRTUAL GATEWAY (Ingress)
    // ============================================

    // Certificate for TLS
    const certificate = acm.Certificate.fromCertificateArn(
      this,
      'Certificate',
      'arn:aws:acm:us-east-1:123456789012:certificate/xxxxx'
    );

    const virtualGateway = new appmesh.VirtualGateway(this, 'VirtualGateway', {
      mesh,
      virtualGatewayName: 'banking-gateway',
      listeners: [
        appmesh.VirtualGatewayListener.http({
          port: 80
        }),
        appmesh.VirtualGatewayListener.http({
          port: 443,
          tls: {
            mode: appmesh.TlsMode.STRICT,
            certificate: appmesh.TlsCertificate.acm(certificate)
          }
        })
      ],
      accessLog: appmesh.AccessLog.fromFilePath('/dev/stdout'),
      backendDefaults: {
        tlsClientPolicy: {
          validation: {
            trust: appmesh.TlsValidationTrust.file('/etc/ssl/certs/ca-bundle.crt')
          }
        }
      }
    });

    // ============================================
    // 4. VIRTUAL SERVICES
    // ============================================

    // Account Service
    const accountVirtualService = new appmesh.VirtualService(this, 'AccountVirtualService', {
      virtualServiceProvider: appmesh.VirtualServiceProvider.none(mesh),
      virtualServiceName: 'account-service.emiratesnbd.local'
    });

    // Transaction Service
    const transactionVirtualService = new appmesh.VirtualService(this, 'TransactionVirtualService', {
      virtualServiceProvider: appmesh.VirtualServiceProvider.none(mesh),
      virtualServiceName: 'transaction-service.emiratesnbd.local'
    });

    // Payment Service
    const paymentVirtualService = new appmesh.VirtualService(this, 'PaymentVirtualService', {
      virtualServiceProvider: appmesh.VirtualServiceProvider.none(mesh),
      virtualServiceName: 'payment-service.emiratesnbd.local'
    });

    // ============================================
    // 5. VIRTUAL NODES
    // ============================================

    // Account Service Virtual Node v1
    const accountNode = new appmesh.VirtualNode(this, 'AccountNode', {
      mesh,
      virtualNodeName: 'account-node-v1',
      serviceDiscovery: appmesh.ServiceDiscovery.cloudMap({
        service: privateNamespace.createService('AccountCloudMapService', {
          name: 'account-service'
        })
      }),
      listeners: [
        appmesh.VirtualNodeListener.http({
          port: 3000,
          timeout: {
            idle: cdk.Duration.seconds(60),
            perRequest: cdk.Duration.seconds(30)
          },
          healthCheck: appmesh.HealthCheck.http({
            path: '/health',
            interval: cdk.Duration.seconds(30),
            timeout: cdk.Duration.seconds(5),
            healthyThreshold: 2,
            unhealthyThreshold: 3
          }),
          connectionPool: appmesh.HttpConnectionPool.create({
            maxConnections: 100,
            maxPendingRequests: 50
          }),
          outlierDetection: {
            baseEjectionDuration: cdk.Duration.seconds(30),
            interval: cdk.Duration.seconds(10),
            maxEjectionPercent: 50,
            maxServerErrors: 5
          }
        })
      ],
      accessLog: appmesh.AccessLog.fromFilePath('/dev/stdout'),
      backends: [
        appmesh.Backend.virtualService(transactionVirtualService)
      ]
    });

    // Account Service Virtual Node v2 (for canary deployment)
    const accountNodeV2 = new appmesh.VirtualNode(this, 'AccountNodeV2', {
      mesh,
      virtualNodeName: 'account-node-v2',
      serviceDiscovery: appmesh.ServiceDiscovery.cloudMap({
        service: privateNamespace.createService('AccountCloudMapServiceV2', {
          name: 'account-service-v2'
        })
      }),
      listeners: [
        appmesh.VirtualNodeListener.http({
          port: 3000,
          timeout: {
            idle: cdk.Duration.seconds(60),
            perRequest: cdk.Duration.seconds(30)
          },
          healthCheck: appmesh.HealthCheck.http({
            path: '/health',
            interval: cdk.Duration.seconds(30)
          })
        })
      ],
      accessLog: appmesh.AccessLog.fromFilePath('/dev/stdout')
    });

    // Transaction Service Virtual Node
    const transactionNode = new appmesh.VirtualNode(this, 'TransactionNode', {
      mesh,
      virtualNodeName: 'transaction-node',
      serviceDiscovery: appmesh.ServiceDiscovery.cloudMap({
        service: privateNamespace.createService('TransactionCloudMapService', {
          name: 'transaction-service'
        })
      }),
      listeners: [
        appmesh.VirtualNodeListener.http({
          port: 4000,
          timeout: {
            idle: cdk.Duration.seconds(120),
            perRequest: cdk.Duration.seconds(60)
          }
        })
      ],
      backends: [
        appmesh.Backend.virtualService(paymentVirtualService)
      ]
    });

    // Payment Service Virtual Node
    const paymentNode = new appmesh.VirtualNode(this, 'PaymentNode', {
      mesh,
      virtualNodeName: 'payment-node',
      serviceDiscovery: appmesh.ServiceDiscovery.cloudMap({
        service: privateNamespace.createService('PaymentCloudMapService', {
          name: 'payment-service'
        })
      }),
      listeners: [
        appmesh.VirtualNodeListener.http({
          port: 8080,
          timeout: {
            idle: cdk.Duration.seconds(300),
            perRequest: cdk.Duration.seconds(120)
          },
          healthCheck: appmesh.HealthCheck.http({
            path: '/healthz',
            interval: cdk.Duration.seconds(30)
          })
        })
      ]
    });

    // ============================================
    // 6. VIRTUAL ROUTERS
    // ============================================

    // Account Service Router with Canary (90/10 split)
    const accountRouter = new appmesh.VirtualRouter(this, 'AccountRouter', {
      mesh,
      virtualRouterName: 'account-router',
      listeners: [
        appmesh.VirtualRouterListener.http(3000)
      ]
    });

    accountRouter.addRoute('AccountRoute', {
      routeName: 'account-route',
      routeSpec: appmesh.RouteSpec.http({
        weightedTargets: [
          {
            virtualNode: accountNode,
            weight: 90
          },
          {
            virtualNode: accountNodeV2,
            weight: 10
          }
        ],
        retryPolicy: {
          retryAttempts: 3,
          retryTimeout: cdk.Duration.seconds(5),
          httpRetryEvents: [
            appmesh.HttpRetryEvent.SERVER_ERROR,
            appmesh.HttpRetryEvent.GATEWAY_ERROR
          ],
          tcpRetryEvents: [appmesh.TcpRetryEvent.CONNECTION_ERROR]
        },
        timeout: {
          idle: cdk.Duration.seconds(60),
          perRequest: cdk.Duration.seconds(30)
        }
      })
    });

    // Update virtual service to use router
    new appmesh.VirtualService(this, 'AccountVirtualServiceWithRouter', {
      virtualServiceProvider: appmesh.VirtualServiceProvider.virtualRouter(accountRouter),
      virtualServiceName: 'account-service.emiratesnbd.local',
      mesh
    });

    // Transaction Service Router
    const transactionRouter = new appmesh.VirtualRouter(this, 'TransactionRouter', {
      mesh,
      virtualRouterName: 'transaction-router',
      listeners: [appmesh.VirtualRouterListener.http(4000)]
    });

    transactionRouter.addRoute('TransactionRoute', {
      routeName: 'transaction-route',
      routeSpec: appmesh.RouteSpec.http({
        weightedTargets: [
          {
            virtualNode: transactionNode,
            weight: 100
          }
        ],
        retryPolicy: {
          retryAttempts: 2,
          retryTimeout: cdk.Duration.seconds(3),
          httpRetryEvents: [appmesh.HttpRetryEvent.SERVER_ERROR]
        }
      })
    });

    new appmesh.VirtualService(this, 'TransactionVirtualServiceWithRouter', {
      virtualServiceProvider: appmesh.VirtualServiceProvider.virtualRouter(transactionRouter),
      virtualServiceName: 'transaction-service.emiratesnbd.local',
      mesh
    });

    // Payment Service Router
    const paymentRouter = new appmesh.VirtualRouter(this, 'PaymentRouter', {
      mesh,
      virtualRouterName: 'payment-router',
      listeners: [appmesh.VirtualRouterListener.http(8080)]
    });

    paymentRouter.addRoute('PaymentRoute', {
      routeName: 'payment-route',
      routeSpec: appmesh.RouteSpec.http({
        weightedTargets: [
          {
            virtualNode: paymentNode,
            weight: 100
          }
        ],
        timeout: {
          idle: cdk.Duration.seconds(300),
          perRequest: cdk.Duration.seconds(120)
        }
      })
    });

    new appmesh.VirtualService(this, 'PaymentVirtualServiceWithRouter', {
      virtualServiceProvider: appmesh.VirtualServiceProvider.virtualRouter(paymentRouter),
      virtualServiceName: 'payment-service.emiratesnbd.local',
      mesh
    });

    // ============================================
    // 7. GATEWAY ROUTES
    // ============================================

    new appmesh.GatewayRoute(this, 'AccountGatewayRoute', {
      virtualGateway,
      gatewayRouteName: 'account-gateway-route',
      routeSpec: appmesh.GatewayRouteSpec.http({
        routeTarget: accountVirtualService,
        match: {
          path: appmesh.HttpGatewayRoutePathMatch.startsWith('/api/accounts')
        }
      })
    });

    new appmesh.GatewayRoute(this, 'TransactionGatewayRoute', {
      virtualGateway,
      gatewayRouteName: 'transaction-gateway-route',
      routeSpec: appmesh.GatewayRouteSpec.http({
        routeTarget: transactionVirtualService,
        match: {
          path: appmesh.HttpGatewayRoutePathMatch.startsWith('/api/transactions')
        }
      })
    });

    new appmesh.GatewayRoute(this, 'PaymentGatewayRoute', {
      virtualGateway,
      gatewayRouteName: 'payment-gateway-route',
      routeSpec: appmesh.GatewayRouteSpec.http({
        routeTarget: paymentVirtualService,
        match: {
          path: appmesh.HttpGatewayRoutePathMatch.startsWith('/api/payments')
        }
      })
    });

    // ============================================
    // 8. OUTPUTS
    // ============================================

    new cdk.CfnOutput(this, 'MeshArn', {
      value: mesh.meshArn,
      description: 'App Mesh ARN'
    });

    new cdk.CfnOutput(this, 'VirtualGatewayArn', {
      value: virtualGateway.virtualGatewayArn,
      description: 'Virtual Gateway ARN'
    });
  }
}
```

---

### 🐳 ECS Task Definition with Envoy Sidecar

```typescript
// infrastructure/cdk/ecs-with-appmesh.ts
import * as cdk from 'aws-cdk-lib';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as appmesh from 'aws-cdk-lib/aws-appmesh';
import * as iam from 'aws-cdk-lib/aws-iam';

export class ECSWithAppMeshStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Assume mesh and cluster already exist
    const mesh = appmesh.Mesh.fromMeshName(this, 'Mesh', 'banking-mesh');
    const cluster = ecs.Cluster.fromClusterArn(this, 'Cluster', 'arn:...');

    // Task Definition with App Mesh integration
    const taskDef = new ecs.FargateTaskDefinition(this, 'AccountTask', {
      memoryLimitMiB: 1024,
      cpu: 512,
      proxyConfiguration: new ecs.AppMeshProxyConfiguration({
        containerName: 'envoy',
        properties: {
          appPorts: [3000],
          proxyIngressPort: 15000,
          proxyEgressPort: 15001,
          egressIgnoredIPs: ['169.254.170.2', '169.254.169.254'],
          ignoredUID: 1337
        }
      })
    });

    // Add Envoy sidecar
    const envoyContainer = taskDef.addContainer('envoy', {
      image: ecs.ContainerImage.fromRegistry('public.ecr.aws/appmesh/aws-appmesh-envoy:v1.27.2.0-prod'),
      essential: true,
      memoryLimitMiB: 256,
      user: '1337',
      environment: {
        APPMESH_RESOURCE_ARN: 'arn:aws:appmesh:us-east-1:123456789012:mesh/banking-mesh/virtualNode/account-node-v1',
        ENABLE_ENVOY_XRAY_TRACING: '1',
        ENABLE_ENVOY_STATS_TAGS: '1'
      },
      healthCheck: {
        command: [
          'CMD-SHELL',
          'curl -s http://localhost:9901/server_info | grep state | grep -q LIVE'
        ],
        interval: cdk.Duration.seconds(5),
        timeout: cdk.Duration.seconds(2),
        retries: 3
      },
      logging: ecs.LogDrivers.awsLogs({
        streamPrefix: 'envoy'
      })
    });

    envoyContainer.addUlimits({
      name: ecs.UlimitName.NOFILE,
      softLimit: 15000,
      hardLimit: 15000
    });

    // Add X-Ray sidecar for tracing
    taskDef.addContainer('xray', {
      image: ecs.ContainerImage.fromRegistry('public.ecr.aws/xray/aws-xray-daemon:latest'),
      essential: false,
      memoryLimitMiB: 256,
      portMappings: [{
        containerPort: 2000,
        protocol: ecs.Protocol.UDP
      }],
      logging: ecs.LogDrivers.awsLogs({
        streamPrefix: 'xray'
      })
    });

    // Add application container
    const appContainer = taskDef.addContainer('app', {
      image: ecs.ContainerImage.fromRegistry('account-service:latest'),
      essential: true,
      memoryLimitMiB: 512,
      portMappings: [{
        containerPort: 3000
      }],
      environment: {
        PORT: '3000',
        TRANSACTION_SERVICE_URL: 'http://transaction-service.emiratesnbd.local:4000',
        AWS_XRAY_DAEMON_ADDRESS: 'localhost:2000'
      },
      dependsOn: [
        {
          containerName: 'envoy',
          condition: ecs.ContainerDependencyCondition.HEALTHY
        }
      ],
      logging: ecs.LogDrivers.awsLogs({
        streamPrefix: 'account-service'
      })
    });

    // Grant permissions
    taskDef.taskRole.addManagedPolicy(
      iam.ManagedPolicy.fromAwsManagedPolicyName('AWSAppMeshEnvoyAccess')
    );
    taskDef.taskRole.addManagedPolicy(
      iam.ManagedPolicy.fromAwsManagedPolicyName('AWSXRayDaemonWriteAccess')
    );
  }
}
```

---

### 📊 Monitoring and Observability

```javascript
// monitoring/appmesh-metrics.js
const { CloudWatchClient, GetMetricStatisticsCommand } = require('@aws-sdk/client-cloudwatch');

const cloudwatch = new CloudWatchClient({ region: 'us-east-1' });

class AppMeshMetrics {
  async getVirtualNodeMetrics(meshName, virtualNodeName, hours = 1) {
    const endTime = new Date();
    const startTime = new Date(endTime.getTime() - hours * 60 * 60 * 1000);

    const metrics = [
      'TargetResponseTime',
      'TargetProcessedBytes',
      'HTTPCode_Target_2XX_Count',
      'HTTPCode_Target_4XX_Count',
      'HTTPCode_Target_5XX_Count',
      'TargetConnectionErrorCount'
    ];

    const results = {};

    for (const metricName of metrics) {
      const response = await cloudwatch.send(new GetMetricStatisticsCommand({
        Namespace: 'AWS/AppMesh',
        MetricName: metricName,
        Dimensions: [
          { Name: 'MeshName', Value: meshName },
          { Name: 'VirtualNodeName', Value: virtualNodeName }
        ],
        StartTime: startTime,
        EndTime: endTime,
        Period: 300,
        Statistics: ['Average', 'Sum', 'Maximum']
      }));

      results[metricName] = response.Datapoints;
    }

    return results;
  }

  async calculateErrorRate(meshName, virtualNodeName, hours = 1) {
    const metrics = await this.getVirtualNodeMetrics(meshName, virtualNodeName, hours);

    const success = metrics['HTTPCode_Target_2XX_Count'].reduce((sum, dp) => sum + dp.Sum, 0);
    const clientErrors = metrics['HTTPCode_Target_4XX_Count'].reduce((sum, dp) => sum + dp.Sum, 0);
    const serverErrors = metrics['HTTPCode_Target_5XX_Count'].reduce((sum, dp) => sum + dp.Sum, 0);

    const total = success + clientErrors + serverErrors;
    const errorRate = total > 0 ? ((clientErrors + serverErrors) / total) * 100 : 0;

    return {
      success,
      clientErrors,
      serverErrors,
      total,
      errorRate: errorRate.toFixed(2) + '%'
    };
  }

  async generateReport(meshName, virtualNodeName) {
    console.log(`App Mesh Metrics Report for ${virtualNodeName}`);
    console.log('='.repeat(60));

    const metrics = await this.getVirtualNodeMetrics(meshName, virtualNodeName);
    const errorRate = await this.calculateErrorRate(meshName, virtualNodeName);

    console.log('\nResponse Time:');
    console.log(JSON.stringify(metrics['TargetResponseTime'], null, 2));

    console.log('\nError Rate:');
    console.log(JSON.stringify(errorRate, null, 2));

    console.log('\nConnection Errors:');
    console.log(JSON.stringify(metrics['TargetConnectionErrorCount'], null, 2));
  }
}

// Run report
(async () => {
  const metrics = new AppMeshMetrics();
  await metrics.generateReport('banking-mesh', 'account-node-v1');
})();
```

---

### 🎓 Interview Discussion Points - Q14

**Q1: What is a service mesh and why use it?**

**A**: A service mesh is an infrastructure layer for managing service-to-service communication. Benefits:
- **Traffic management**: Canary, blue/green deployments
- **Resilience**: Retries, circuit breakers, timeouts
- **Security**: mTLS, authorization
- **Observability**: Distributed tracing, metrics

**Q2: What's the difference between App Mesh and Istio?**

**A**:
- **App Mesh**: AWS-managed, simpler, ECS/EKS integration, no CRDs on EKS
- **Istio**: Open-source, more features, community-driven, Kubernetes-native

**Q3: How does App Mesh handle circuit breaking?**

**A**: Through Envoy's outlier detection:
- Monitors error rates and latency
- Ejects unhealthy endpoints temporarily
- Automatically re-adds when healthy
- Configurable thresholds and timeouts

**Q4: What are the performance implications of using App Mesh?**

**A**:
- **Latency**: +1-5ms per hop (Envoy overhead)
- **CPU**: 50-200m per sidecar
- **Memory**: 128-256MB per sidecar
- **Benefits**: Better failure handling, observability

**Q5: How do you implement canary deployments with App Mesh?**

**A**:
- Create two virtual nodes (v1 and v2)
- Configure virtual router with weighted targets
- Gradually shift traffic (e.g., 90/10 → 50/50 → 0/100)
- Monitor metrics and rollback if needed
- Use CloudWatch alarms for automated rollback

---

**End of File 7**

