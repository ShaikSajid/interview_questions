# Service Mesh Advanced Patterns

## Question 49: Advanced Service Mesh with Istio on EKS

### 📋 Question Statement

Implement advanced service mesh patterns for Emirates NBD using Istio on EKS including traffic management, security policies, and observability.

---

### 🕸️ Istio Service Mesh Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    ISTIO SERVICE MESH ON EKS                                │
└────────────────────────────────────────────────────────────────────────────┘

    ┌──────────────────────────────────────────────────────────────────┐
    │                      CONTROL PLANE                                │
    │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
    │  │  Istiod  │  │  Pilot   │  │  Citadel │  │  Galley  │       │
    │  │(Config)  │  │(Traffic) │  │  (mTLS)  │  │(Validation)│      │
    │  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
    └───────────────────────────┬──────────────────────────────────────┘
                                │
    ┌───────────────────────────┴──────────────────────────────────────┐
    │                      DATA PLANE                                   │
    │  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐           │
    │  │   Pod A     │   │   Pod B     │   │   Pod C     │           │
    │  │ ┌─────────┐ │   │ ┌─────────┐ │   │ ┌─────────┐ │           │
    │  │ │  App    │ │   │ │  App    │ │   │ │  App    │ │           │
    │  │ └────┬────┘ │   │ └────┬────┘ │   │ └────┬────┘ │           │
    │  │ ┌────▼────┐ │   │ ┌────▼────┐ │   │ ┌────▼────┐ │           │
    │  │ │ Envoy   │◄├───┼─┤ Envoy   │◄├───┼─┤ Envoy   │ │           │
    │  │ │ Proxy   │ │   │ │ Proxy   │ │   │ │ Proxy   │ │           │
    │  │ └─────────┘ │   │ └─────────┘ │   │ └─────────┘ │           │
    │  └─────────────┘   └─────────────┘   └─────────────┘           │
    └──────────────────────────────────────────────────────────────────┘
```

### 🏗️ EKS Cluster with Istio Setup

```bash
# scripts/setup-istio-eks.sh
#!/bin/bash

# Create EKS cluster
eksctl create cluster \
  --name emirates-nbd-banking \
  --region us-east-1 \
  --version 1.28 \
  --nodegroup-name standard-workers \
  --node-type m5.xlarge \
  --nodes 3 \
  --nodes-min 3 \
  --nodes-max 10 \
  --managed

# Install Istio
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.20.0 sh -
cd istio-1.20.0
export PATH=$PWD/bin:$PATH

# Install Istio with production profile
istioctl install --set profile=production -y

# Enable automatic sidecar injection
kubectl label namespace banking istio-injection=enabled

# Install observability addons
kubectl apply -f samples/addons/prometheus.yaml
kubectl apply -f samples/addons/grafana.yaml
kubectl apply -f samples/addons/jaeger.yaml
kubectl apply -f samples/addons/kiali.yaml

echo "Istio installed successfully on EKS!"
```

### 🔧 Istio Configuration with CDK

```typescript
// infrastructure/cdk/eks-istio-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as eks from 'aws-cdk-lib/aws-eks';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as iam from 'aws-cdk-lib/aws-iam';

export class EksIstioStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // VPC for EKS
    const vpc = new ec2.Vpc(this, 'BankingVPC', {
      maxAzs: 3,
      natGateways: 3
    });

    // EKS Cluster
    const cluster = new eks.Cluster(this, 'BankingCluster', {
      clusterName: 'emirates-nbd-banking',
      version: eks.KubernetesVersion.V1_28,
      vpc,
      defaultCapacity: 0, // We'll add managed node groups
      vpcSubnets: [{ subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS }]
    });

    // Managed node group
    cluster.addNodegroupCapacity('standard-workers', {
      instanceTypes: [new ec2.InstanceType('m5.xlarge')],
      minSize: 3,
      maxSize: 10,
      desiredSize: 3,
      diskSize: 100
    });

    // Banking namespace
    const bankingNamespace = cluster.addManifest('BankingNamespace', {
      apiVersion: 'v1',
      kind: 'Namespace',
      metadata: {
        name: 'banking',
        labels: {
          'istio-injection': 'enabled'
        }
      }
    });

    // Istio Helm Chart
    const istioBase = cluster.addHelmChart('IstioBase', {
      chart: 'base',
      repository: 'https://istio-release.storage.googleapis.com/charts',
      namespace: 'istio-system',
      createNamespace: true,
      release: 'istio-base',
      version: '1.20.0'
    });

    const istiod = cluster.addHelmChart('Istiod', {
      chart: 'istiod',
      repository: 'https://istio-release.storage.googleapis.com/charts',
      namespace: 'istio-system',
      release: 'istiod',
      version: '1.20.0',
      values: {
        global: {
          hub: 'docker.io/istio',
          tag: '1.20.0'
        },
        pilot: {
          autoscaleEnabled: true,
          autoscaleMin: 2,
          autoscaleMax: 5,
          resources: {
            requests: {
              cpu: '500m',
              memory: '2048Mi'
            }
          }
        },
        telemetry: {
          enabled: true,
          v2: {
            enabled: true,
            prometheus: {
              enabled: true
            }
          }
        }
      }
    });

    istiod.node.addDependency(istioBase);

    // Istio Ingress Gateway
    const ingressGateway = cluster.addHelmChart('IstioIngressGateway', {
      chart: 'gateway',
      repository: 'https://istio-release.storage.googleapis.com/charts',
      namespace: 'istio-system',
      release: 'istio-ingressgateway',
      version: '1.20.0',
      values: {
        service: {
          type: 'LoadBalancer',
          annotations: {
            'service.beta.kubernetes.io/aws-load-balancer-type': 'nlb',
            'service.beta.kubernetes.io/aws-load-balancer-scheme': 'internet-facing'
          }
        }
      }
    });

    ingressGateway.node.addDependency(istiod);

    // Output cluster details
    new cdk.CfnOutput(this, 'ClusterName', {
      value: cluster.clusterName
    });

    new cdk.CfnOutput(this, 'ClusterEndpoint', {
      value: cluster.clusterEndpoint
    });
  }
}
```

### 🔒 Istio Security Policies

```yaml
# kubernetes/istio/peer-authentication.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: banking
spec:
  mtls:
    mode: STRICT  # Enforce mutual TLS between all services
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: account-service-authz
  namespace: banking
spec:
  selector:
    matchLabels:
      app: account-service
  action: ALLOW
  rules:
    # Allow from transaction service
    - from:
        - source:
            principals: ["cluster.local/ns/banking/sa/transaction-service"]
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/v1/accounts/*"]
    
    # Allow from API gateway
    - from:
        - source:
            namespaces: ["istio-system"]
      to:
        - operation:
            methods: ["GET", "POST", "PUT", "DELETE"]
    
    # Deny all other traffic
  rules:
    - from:
        - source:
            notNamespaces: ["banking", "istio-system"]
      to:
        - operation: {}
```

### 🚦 Traffic Management

```yaml
# kubernetes/istio/virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: account-service
  namespace: banking
spec:
  hosts:
    - account-service
    - account-service.banking.svc.cluster.local
  gateways:
    - istio-ingressgateway
  http:
    # Timeout configuration
    - timeout: 30s
      retries:
        attempts: 3
        perTryTimeout: 10s
        retryOn: gateway-error,connect-failure,refused-stream
      
      # Fault injection for testing
      fault:
        delay:
          percentage:
            value: 0.1  # 10% requests
          fixedDelay: 5s
        abort:
          percentage:
            value: 0.01  # 1% requests
          httpStatus: 503
      
      # Traffic routing
      route:
        - destination:
            host: account-service
            subset: v1
          weight: 90
        - destination:
            host: account-service
            subset: v2
          weight: 10
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: account-service
  namespace: banking
spec:
  host: account-service
  trafficPolicy:
    # Connection pool settings
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 100
        maxRequestsPerConnection: 2
    
    # Load balancing
    loadBalancer:
      simple: LEAST_REQUEST
    
    # Outlier detection (circuit breaker)
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 30
  
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

### 📊 Observability Configuration

```yaml
# kubernetes/istio/telemetry.yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: banking-telemetry
  namespace: banking
spec:
  tracing:
    - providers:
        - name: jaeger
      randomSamplingPercentage: 100.0
      customTags:
        environment:
          literal:
            value: "production"
        region:
          literal:
            value: "us-east-1"
  
  metrics:
    - providers:
        - name: prometheus
      overrides:
        - match:
            metric: ALL_METRICS
          tagOverrides:
            request_protocol:
              operation: UPSERT
              value: "istio"
```

### 🔍 Service Mesh Deployment

```yaml
# kubernetes/deployments/account-service.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: account-service
  namespace: banking
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: account-service-v1
  namespace: banking
spec:
  replicas: 3
  selector:
    matchLabels:
      app: account-service
      version: v1
  template:
    metadata:
      labels:
        app: account-service
        version: v1
      annotations:
        sidecar.istio.io/inject: "true"
        proxy.istio.io/config: |
          tracing:
            zipkin:
              address: jaeger-collector.istio-system:9411
    spec:
      serviceAccountName: account-service
      containers:
        - name: account-service
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/account-service:v1
          ports:
            - containerPort: 3000
              name: http
          env:
            - name: PORT
              value: "3000"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: database-credentials
                  key: url
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 1Gi
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
---
apiVersion: v1
kind: Service
metadata:
  name: account-service
  namespace: banking
  labels:
    app: account-service
spec:
  ports:
    - port: 80
      targetPort: 3000
      protocol: TCP
      name: http
  selector:
    app: account-service
```

### 🎓 Interview Discussion Points - Q49

**Q1: What is a service mesh and why use it?**

**A**:
**Service Mesh Benefits:**
- **Traffic management**: Canary, A/B testing, circuit breaking
- **Security**: Mutual TLS (mTLS), authorization policies
- **Observability**: Distributed tracing, metrics, logs
- **Resilience**: Retries, timeouts, circuit breakers
- **Service discovery**: Automatic service registration

**Q2: How does Istio sidecar injection work?**

**A**:
- **Automatic injection**: Label namespace with `istio-injection=enabled`
- **Mutating webhook**: Intercepts pod creation
- **Sidecar container**: Envoy proxy injected into pod
- **Init container**: Sets up iptables rules
- **Traffic interception**: All traffic routed through Envoy

**Q3: What is mTLS and how does Istio implement it?**

**A**:
- **Mutual TLS**: Both client and server authenticate
- **Automatic certificates**: Istio CA issues short-lived certs
- **Rotation**: Automatic cert rotation
- **Modes**: STRICT, PERMISSIVE, DISABLE
- **Zero trust**: No plaintext traffic between services

**Q4: How to implement circuit breaking in Istio?**

**A**:
```yaml
outlierDetection:
  consecutiveErrors: 5      # Eject after 5 errors
  interval: 30s             # Check every 30s
  baseEjectionTime: 30s     # Keep ejected for 30s
  maxEjectionPercent: 50    # Max 50% of hosts ejected
```
- Prevents cascading failures
- Health checks on endpoints
- Automatic re-inclusion after recovery

**Q5: How to monitor Istio service mesh?**

**A**:
- **Kiali**: Service mesh topology visualization
- **Prometheus**: Metrics collection
- **Grafana**: Dashboards for latency, throughput, errors
- **Jaeger**: Distributed tracing
- **CloudWatch**: AWS integration for centralized logs

---

## Question 50: Istio Traffic Splitting & Canary Deployments

### 📋 Question Statement

Implement advanced traffic management for Emirates NBD using Istio VirtualServices, DestinationRules, and canary deployments with automated progressive rollout based on success metrics.

---

### 🚦 Istio Traffic Splitting Configuration

```yaml
# kubernetes/istio/virtual-service-canary.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: account-service
  namespace: banking
spec:
  hosts:
    - account-service
  http:
    - match:
        - headers:
            x-user-group:
              exact: beta-testers
      route:
        - destination:
            host: account-service
            subset: v2
          weight: 100
    - route:
        - destination:
            host: account-service
            subset: v1
          weight: 90
        - destination:
            host: account-service
            subset: v2
          weight: 10
---
apiVersion: networking.istio.io/v1beta1
kind:DestinationRule
metadata:
  name: account-service
  namespace: banking
spec:
  host: account-service
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 100
        maxRequestsPerConnection: 2
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
```

### 🎯 Progressive Canary Controller

```typescript
// services/istio/canary-controller.ts
import * as k8s from '@kubernetes/client-node';
import { PrometheusDriver } from 'prometheus-query';

export class IstioCanaryController {
  private k8sApi: k8s.CustomObjectsApi;
  private prometheus: PrometheusDriver;

  constructor() {
    const kc = new k8s.KubeConfig();
    kc.loadFromDefault();
    this.k8sApi = kc.makeApiClient(k8s.CustomObjectsApi);
    this.prometheus = new PrometheusDriver({
      endpoint: 'http://prometheus.istio-system:9090'
    });
  }

  async executeProgressiveRollout(
    serviceName: string,
    namespace: string,
    canaryVersion: string
  ): Promise<RolloutResult> {
    const stages = [
      { weight: 10, duration: 300 },   // 10% for 5 minutes
      { weight: 25, duration: 300 },   // 25% for 5 minutes
      { weight: 50, duration: 600 },   // 50% for 10 minutes
      { weight: 75, duration: 600 },   // 75% for 10 minutes
      { weight: 100, duration: 0 }     // 100% (complete)
    ];

    for (const stage of stages) {
      console.log(`Setting traffic to ${stage.weight}% canary`);

      // Update VirtualService
      await this.updateTrafficSplit(serviceName, namespace, {
        stable: 100 - stage.weight,
        canary: stage.weight
      });

      if (stage.duration > 0) {
        // Wait and monitor
        await this.sleep(stage.duration * 1000);

        // Check metrics
        const metrics = await this.getCanaryMetrics(serviceName, namespace, canaryVersion);

        if (!this.isCanaryHealthy(metrics)) {
          console.error('Canary metrics unhealthy, rolling back');
          await this.rollback(serviceName, namespace);
          return {
            success: false,
            stage: stage.weight,
            reason: 'Unhealthy metrics detected'
          };
        }
      }
    }

    console.log('Canary rollout successful!');
    return { success: true, stage: 100 };
  }

  private async updateTrafficSplit(
    serviceName: string,
    namespace: string,
    weights: { stable: number; canary: number }
  ): Promise<void> {
    const virtualService = {
      apiVersion: 'networking.istio.io/v1beta1',
      kind: 'VirtualService',
      metadata: {
        name: serviceName,
        namespace
      },
      spec: {
        hosts: [serviceName],
        http: [
          {
            route: [
              {
                destination: {
                  host: serviceName,
                  subset: 'v1'
                },
                weight: weights.stable
              },
              {
                destination: {
                  host: serviceName,
                  subset: 'v2'
                },
                weight: weights.canary
              }
            ]
          }
        ]
      }
    };

    await this.k8sApi.patchNamespacedCustomObject(
      'networking.istio.io',
      'v1beta1',
      namespace,
      'virtualservices',
      serviceName,
      virtualService
    );
  }

  private async getCanaryMetrics(
    serviceName: string,
    namespace: string,
    version: string
  ): Promise<CanaryMetrics> {
    const now = Date.now() / 1000;
    const fiveMinutesAgo = now - 300;

    // Success rate
    const successRateQuery = `
      sum(rate(istio_requests_total{
        destination_service=~"${serviceName}.*",
        destination_version="${version}",
        response_code!~"5.*"
      }[5m])) / 
      sum(rate(istio_requests_total{
        destination_service=~"${serviceName}.*",
        destination_version="${version}"
      }[5m]))
    `;

    const successRate = await this.prometheus.instantQuery(successRateQuery);

    // P99 latency
    const latencyQuery = `
      histogram_quantile(0.99, 
        sum(rate(istio_request_duration_milliseconds_bucket{
          destination_service=~"${serviceName}.*",
          destination_version="${version}"
        }[5m])) by (le)
      )
    `;

    const p99Latency = await this.prometheus.instantQuery(latencyQuery);

    return {
      successRate: parseFloat(successRate.result[0]?.value[1] || '0'),
      p99Latency: parseFloat(p99Latency.result[0]?.value[1] || '0'),
      requestRate: 0
    };
  }

  private isCanaryHealthy(metrics: CanaryMetrics): boolean {
    // Success rate must be > 99%
    if (metrics.successRate < 0.99) {
      console.error(`Low success rate: ${metrics.successRate}`);
      return false;
    }

    // P99 latency must be < 500ms
    if (metrics.p99Latency > 500) {
      console.error(`High latency: ${metrics.p99Latency}ms`);
      return false;
    }

    return true;
  }

  private async rollback(serviceName: string, namespace: string): Promise<void> {
    // Set 100% traffic to stable version
    await this.updateTrafficSplit(serviceName, namespace, {
      stable: 100,
      canary: 0
    });

    console.log('Rollback complete');
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

interface CanaryMetrics {
  successRate: number;
  p99Latency: number;
  requestRate: number;
}

interface RolloutResult {
  success: boolean;
  stage: number;
  reason?: string;
}
```

### 📊 A/B Testing with Istio

```yaml
# kubernetes/istio/ab-testing.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service-ab-test
  namespace: banking
spec:
  hosts:
    - payment-service
  http:
    # Route based on user cohort
    - match:
        - headers:
            x-user-cohort:
              exact: "A"
      route:
        - destination:
            host: payment-service
            subset: variant-a
    - match:
        - headers:
            x-user-cohort:
              exact: "B"
      route:
        - destination:
            host: payment-service
            subset: variant-b
    # Default: 50/50 split
    - route:
        - destination:
            host: payment-service
            subset: variant-a
          weight: 50
        - destination:
            host: payment-service
            subset: variant-b
          weight: 50
```

### 🎓 Interview Discussion Points - Q50

**Q1: What is traffic splitting in Istio?**

**A**:
- **Weight-based**: 90% v1, 10% v2
- **Header-based**: Route by user group
- **Use cases**: Canary deployment, A/B testing
- **Zero downtime**: Gradual rollout

**Q2: How does Istio canary differ from Kubernetes rolling update?**

**A**:
- **Kubernetes**: Replaces pods gradually, no traffic control
- **Istio**: Controls traffic %, both versions run simultaneously
- **Istio advantage**: Rollback instantly, fine-grained control
- **Example**: Istio can do 5% canary, K8s minimum is 1 pod

**Q3: What metrics determine canary health?**

**A**:
- **Success rate**: >99% (5xx errors)
- **Latency**: P99 < 500ms
- **Error rate**: <1%
- **Business metrics**: Transaction success, conversion rate

**Q4: How to implement automated rollback?**

**A**:
```typescript
if (canaryErrorRate > stableErrorRate * 1.5) {
  await rollback();
}
```
**Triggers**: Error rate spike, latency increase, manual intervention

**Q5: What is shadow traffic in Istio?**

**A**:
```yaml
http:
  - mirror:
      host: payment-service
      subset: v2
    mirrorPercentage:
      value: 100
```
**Purpose**: Test new version with production traffic without impacting users

---

**End of File 25**

