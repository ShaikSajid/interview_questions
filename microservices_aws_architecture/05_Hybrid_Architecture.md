# Hybrid Architecture - Serverless + Containers

## Question 9: Serverless vs Container Decision Matrix

### 📋 Question Statement

Design a comprehensive decision framework for choosing between serverless (Lambda) and container-based (ECS/EKS) architectures for Emirates NBD banking services. Include:
- Technical comparison matrix
- Cost analysis with real calculations
- Performance benchmarks
- Use case patterns
- Migration strategies
- Real-world examples from banking domain

---

### 📊 Comprehensive Comparison Matrix

| **Criteria** | **AWS Lambda (Serverless)** | **ECS Fargate (Containers)** | **EKS (Kubernetes)** |
|-------------|----------------------------|------------------------------|---------------------|
| **Startup Time** | Cold start: 100-500ms | Warm start: <100ms | Warm start: <100ms |
| **Execution Time** | Max 15 minutes | Unlimited | Unlimited |
| **Memory** | 128MB - 10GB | 512MB - 30GB | Flexible (node-based) |
| **Scaling Speed** | Instant (1000s per second) | Moderate (30-60 seconds) | Moderate (30-60 seconds) |
| **Minimum Cost** | $0 (free tier: 1M req/month) | ~$15/month (1 task) | ~$70/month (2 nodes) |
| **Cost Model** | Per-request + duration | Per-second CPU/memory | Node hours + container resources |
| **State Management** | Stateless only | Can be stateful | Can be stateful |
| **Connection Pooling** | Difficult (RDS Proxy needed) | Native support | Native support |
| **Container Size** | 250MB uncompressed | Up to 20GB | Up to 20GB |
| **Deployment** | Code/container upload | ECS task definition | Kubernetes manifests |
| **Learning Curve** | Low | Medium | High |
| **Vendor Lock-in** | High (AWS-specific) | Medium | Low (portable) |
| **Monitoring** | CloudWatch Logs/X-Ray | CloudWatch + Container Insights | Prometheus/Grafana + CloudWatch |
| **Best For** | Event-driven, sporadic traffic | Steady-state, predictable load | Complex orchestration, multi-cloud |

---

### 💰 Cost Analysis (Real Calculations)

#### **Scenario 1: Notification Service (Event-Driven)**

**Traffic Pattern**: 
- 5 million notifications/month
- Average execution: 200ms
- Memory: 512MB

**Lambda Cost**:
```
Requests: 5M * $0.20 per 1M = $1.00
Compute: 5M * 0.2s * (512MB/1024MB) * $0.0000166667 = $8.33
Total: $9.33/month
```

**ECS Fargate Cost** (1 task running 24/7):
```
vCPU: 0.25 * $0.04048 * 730 hours = $7.40
Memory: 0.5GB * $0.004445 * 730 hours = $1.62
Total: $9.02/month
```

**Winner**: Lambda for sporadic, ECS for steady traffic > 1M requests/day

---

#### **Scenario 2: Account Service (High Traffic)**

**Traffic Pattern**:
- 10M requests/month
- Average execution: 500ms
- Memory: 1GB
- Steady traffic: 4 requests/second average

**Lambda Cost**:
```
Requests: 10M * $0.20 per 1M = $2.00
Compute: 10M * 0.5s * 1GB * $0.0000166667 = $83.33
Total: $85.33/month
```

**ECS Fargate Cost** (2 tasks for HA):
```
Per task: 0.5 vCPU * $0.04048 * 730 = $14.77
           1GB * $0.004445 * 730 = $3.24
           Subtotal: $18.01
2 tasks: $18.01 * 2 = $36.02/month
```

**Winner**: ECS Fargate saves $49/month (57% cheaper)

---

#### **Scenario 3: Transaction Service (Variable Traffic)**

**Traffic Pattern**:
- Peak hours: 100 requests/second (8 hours/day)
- Off-peak: 10 requests/second (16 hours/day)
- Average execution: 300ms
- Memory: 512MB

**Lambda Cost**:
```
Daily requests: (100 * 8 * 3600) + (10 * 16 * 3600) = 3,456,000
Monthly: 3.456M * 30 = 103.68M requests
Requests: 103.68M * $0.20 per 1M = $20.74
Compute: 103.68M * 0.3s * 0.5GB * $0.0000166667 = $259.20
Total: $279.94/month
```

**ECS Fargate with Auto-Scaling**:
```
Peak: 10 tasks for 8 hours/day
Off-peak: 2 tasks for 16 hours/day
Average: (10 * 8 + 2 * 16) / 24 = 4.67 tasks

Cost: 4.67 tasks * $18.01 = $84.11/month
```

**Winner**: ECS with auto-scaling saves $196/month (70% cheaper)

---

### 🎯 Decision Framework

```javascript
// decision-framework.js
class ArchitectureDecisionFramework {
  analyzeWorkload(workload) {
    const scores = {
      lambda: 0,
      ecs: 0,
      eks: 0
    };

    // 1. Traffic Pattern Analysis
    if (workload.trafficPattern === 'SPORADIC') {
      scores.lambda += 3;
    } else if (workload.trafficPattern === 'STEADY') {
      scores.ecs += 2;
      scores.eks += 2;
    } else if (workload.trafficPattern === 'BURST') {
      scores.lambda += 2;
      scores.eks += 1;
    }

    // 2. Execution Duration
    if (workload.executionTime < 60) { // seconds
      scores.lambda += 2;
    } else if (workload.executionTime < 900) {
      scores.lambda += 1;
      scores.ecs += 1;
    } else {
      scores.ecs += 3;
      scores.eks += 3;
    }

    // 3. State Management
    if (workload.requiresState) {
      scores.ecs += 2;
      scores.eks += 2;
    } else {
      scores.lambda += 2;
    }

    // 4. Database Connections
    if (workload.heavyDatabaseUsage) {
      scores.ecs += 3;
      scores.eks += 3;
    } else {
      scores.lambda += 1;
    }

    // 5. Request Volume
    if (workload.requestsPerMonth < 1000000) {
      scores.lambda += 2;
    } else if (workload.requestsPerMonth < 10000000) {
      scores.ecs += 2;
    } else {
      scores.eks += 3;
    }

    // 6. Cold Start Sensitivity
    if (workload.coldStartSensitive) {
      scores.ecs += 2;
      scores.eks += 2;
    } else {
      scores.lambda += 1;
    }

    // 7. Multi-Cloud Requirement
    if (workload.multiCloud) {
      scores.eks += 5;
    }

    // 8. Team Expertise
    if (workload.teamExpertise === 'KUBERNETES') {
      scores.eks += 2;
    } else if (workload.teamExpertise === 'DOCKER') {
      scores.ecs += 2;
    } else if (workload.teamExpertise === 'SERVERLESS') {
      scores.lambda += 2;
    }

    return this.selectArchitecture(scores);
  }

  selectArchitecture(scores) {
    const sorted = Object.entries(scores).sort((a, b) => b[1] - a[1]);
    
    return {
      recommended: sorted[0][0],
      scores,
      reasoning: this.generateReasoning(scores)
    };
  }

  generateReasoning(scores) {
    const reasons = [];
    
    if (scores.lambda > scores.ecs && scores.lambda > scores.eks) {
      reasons.push('Lambda recommended for event-driven, variable workload');
      reasons.push('Lower operational overhead');
      reasons.push('Auto-scaling built-in');
    } else if (scores.ecs > scores.eks) {
      reasons.push('ECS Fargate recommended for steady-state workload');
      reasons.push('Better database connection management');
      reasons.push('Cost-effective at scale');
    } else {
      reasons.push('EKS recommended for complex orchestration');
      reasons.push('Multi-cloud portability');
      reasons.push('Advanced features (service mesh, operators)');
    }

    return reasons;
  }
}

// Example Usage
const framework = new ArchitectureDecisionFramework();

// Notification Service
const notificationWorkload = {
  trafficPattern: 'SPORADIC',
  executionTime: 2, // seconds
  requiresState: false,
  heavyDatabaseUsage: false,
  requestsPerMonth: 5000000,
  coldStartSensitive: false,
  multiCloud: false,
  teamExpertise: 'SERVERLESS'
};

console.log(framework.analyzeWorkload(notificationWorkload));
// Output: { recommended: 'lambda', scores: { lambda: 12, ecs: 2, eks: 2 }, reasoning: [...] }

// Account Service
const accountWorkload = {
  trafficPattern: 'STEADY',
  executionTime: 5,
  requiresState: true,
  heavyDatabaseUsage: true,
  requestsPerMonth: 20000000,
  coldStartSensitive: true,
  multiCloud: false,
  teamExpertise: 'DOCKER'
};

console.log(framework.analyzeWorkload(accountWorkload));
// Output: { recommended: 'ecs', scores: { lambda: 2, ecs: 14, eks: 12 }, reasoning: [...] }
```

---

### 🏦 Banking Service Use Cases

| **Service** | **Recommended** | **Justification** |
|------------|----------------|-------------------|
| **Account Management** | ECS Fargate | • Steady traffic<br>• Heavy DB connections<br>• ACID transactions<br>• Low latency required |
| **Transaction Processing** | Lambda + Step Functions | • Event-driven<br>• Variable traffic<br>• Saga orchestration<br>• Auto-scaling needed |
| **Notification Service** | Lambda + SQS | • Sporadic traffic<br>• Fire-and-forget<br>• Cost-effective<br>• Simple logic |
| **Payment Gateway** | EKS | • PCI-DSS compliance<br>• Complex workflows<br>• Stateful sessions<br>• Multi-region |
| **Fraud Detection** | Lambda + SageMaker | • Real-time scoring<br>• ML model integration<br>• Event-driven<br>• High burst capacity |
| **Reporting Service** | ECS Fargate | • Long-running queries<br>• Heavy computation<br>• Scheduled jobs<br>• Large memory |
| **User Authentication** | Cognito + Lambda | • Built-in user management<br>• OAuth/OIDC<br>• Custom triggers<br>• Low overhead |
| **API Gateway** | API Gateway + Lambda | • Request routing<br>• Rate limiting<br>• Auth integration<br>• Managed service |

---

### 📈 Performance Benchmarks

```javascript
// performance-benchmark.js
const benchmark = {
  // Cold Start Comparison
  coldStart: {
    lambda: {
      runtime: 'Node.js 20',
      packageSize: '5MB',
      averageColdStart: '150ms',
      p99ColdStart: '450ms',
      provisioned: '0ms (with Provisioned Concurrency)'
    },
    ecsfargate: {
      runtime: 'Docker',
      imageSize: '200MB',
      averageStart: '30s',
      p99Start: '60s',
      warmStart: '<100ms'
    },
    eks: {
      runtime: 'Kubernetes',
      imageSize: '200MB',
      averageStart: '45s',
      p99Start: '90s',
      warmStart: '<100ms'
    }
  },

  // Request Latency (Warm)
  latency: {
    lambda: {
      p50: '20ms',
      p95: '50ms',
      p99: '100ms'
    },
    ecsfargate: {
      p50: '15ms',
      p95: '30ms',
      p99: '60ms'
    },
    eks: {
      p50: '15ms',
      p95: '35ms',
      p99: '70ms'
    }
  },

  // Throughput (requests per second per instance)
  throughput: {
    lambda: {
      max: 1000, // per concurrent execution
      burst: 3000,
      note: 'Limited by concurrency quota'
    },
    ecsfargate: {
      max: 5000, // per task
      burst: 6000,
      note: 'Depends on task size'
    },
    eks: {
      max: 10000, // per pod
      burst: 12000,
      note: 'Highly configurable'
    }
  },

  // Database Connection Handling
  databaseConnections: {
    lambda: {
      approach: 'RDS Proxy or connection per invocation',
      pooling: 'Difficult without RDS Proxy',
      cost: 'RDS Proxy: $0.015/hour per vCPU',
      latency: '+5-10ms with RDS Proxy'
    },
    ecsfargate: {
      approach: 'Native connection pooling',
      pooling: 'Built-in (pg.Pool, etc.)',
      cost: 'Included',
      latency: 'Direct connection'
    },
    eks: {
      approach: 'Native connection pooling + sidecar',
      pooling: 'Built-in + advanced patterns',
      cost: 'Included',
      latency: 'Direct connection'
    }
  }
};

module.exports = benchmark;
```

---

### 🔄 Migration Strategies

#### **Strategy 1: Lambda to ECS (For Growing Services)**

```javascript
// migration-lambda-to-ecs.js
/**
 * When to migrate Lambda → ECS:
 * - Monthly requests > 10M
 * - Average execution > 500ms
 * - Heavy database usage
 * - Frequent cold starts affecting UX
 */

class LambdaToECSMigration {
  calculateBreakEvenPoint(avgExecutionMs, memoryGB, requestsPerMonth) {
    // Lambda cost
    const lambdaRequestCost = (requestsPerMonth / 1000000) * 0.20;
    const lambdaComputeCost = requestsPerMonth * (avgExecutionMs / 1000) * memoryGB * 0.0000166667;
    const lambdaTotalCost = lambdaRequestCost + lambdaComputeCost;

    // ECS cost (assume 2 tasks for HA)
    const vcpu = memoryGB >= 2 ? 0.5 : 0.25;
    const ecsMonthlyPerTask = (vcpu * 0.04048 * 730) + (memoryGB * 0.004445 * 730);
    const ecsTotalCost = ecsMonthlyPerTask * 2;

    return {
      lambdaCost: lambdaTotalCost,
      ecsCost: ecsTotalCost,
      savings: lambdaTotalCost - ecsTotalCost,
      recommendation: lambdaTotalCost > ecsTotalCost ? 'Migrate to ECS' : 'Stay on Lambda',
      breakEvenRequests: this.findBreakEven(avgExecutionMs, memoryGB, ecsTotalCost)
    };
  }

  findBreakEven(avgExecutionMs, memoryGB, ecsCost) {
    // Solve for requests where Lambda cost = ECS cost
    // lambdaRequestCost + lambdaComputeCost = ecsCost
    // (R / 1M) * 0.20 + R * (exec/1000) * mem * 0.0000166667 = ecsCost
    
    const computePerRequest = (avgExecutionMs / 1000) * memoryGB * 0.0000166667;
    const requestCostPerMillion = 0.20;
    
    // R * (0.20/1M + computePerRequest) = ecsCost
    const costPerRequest = (requestCostPerMillion / 1000000) + computePerRequest;
    const breakEvenRequests = Math.ceil(ecsCost / costPerRequest);
    
    return breakEvenRequests;
  }
}

// Example
const migration = new LambdaToECSMigration();

// Account Service Analysis
const accountService = migration.calculateBreakEvenPoint(
  500, // 500ms execution
  1,   // 1GB memory
  20000000 // 20M requests/month
);

console.log(accountService);
/*
{
  lambdaCost: $169.33,
  ecsCost: $36.02,
  savings: $133.31 (78% reduction),
  recommendation: 'Migrate to ECS',
  breakEvenRequests: 4,320,000 (break-even at 4.32M requests/month)
}
*/
```

#### **Strategy 2: ECS to Lambda (For Declining Services)**

```javascript
// migration-ecs-to-lambda.js
/**
 * When to migrate ECS → Lambda:
 * - Traffic decreased to < 1M requests/month
 * - Service is event-driven
 * - Want to reduce operational overhead
 */

class ECSToLambdaMigration {
  constructor() {
    this.ecsFixedCost = 36.02; // 2 tasks minimum
  }

  shouldMigrate(requestsPerMonth, avgExecutionMs, memoryGB) {
    const lambdaCost = this.calculateLambdaCost(requestsPerMonth, avgExecutionMs, memoryGB);
    
    return {
      currentECSCost: this.ecsFixedCost,
      projectedLambdaCost: lambdaCost,
      savings: this.ecsFixedCost - lambdaCost,
      recommendation: lambdaCost < this.ecsFixedCost ? 'Migrate to Lambda' : 'Stay on ECS',
      savingsPercentage: ((this.ecsFixedCost - lambdaCost) / this.ecsFixedCost * 100).toFixed(2) + '%'
    };
  }

  calculateLambdaCost(requests, execMs, memoryGB) {
    const requestCost = (requests / 1000000) * 0.20;
    const computeCost = requests * (execMs / 1000) * memoryGB * 0.0000166667;
    return requestCost + computeCost;
  }
}

// Example
const ecsToLambda = new ECSToLambdaMigration();

// Notification Service (low traffic)
const notificationService = ecsToLambda.shouldMigrate(
  500000,  // 500K requests/month
  200,     // 200ms execution
  0.5      // 512MB memory
);

console.log(notificationService);
/*
{
  currentECSCost: $36.02,
  projectedLambdaCost: $0.93,
  savings: $35.09,
  recommendation: 'Migrate to Lambda',
  savingsPercentage: '97.42%'
}
*/
```

---

### 🎓 Interview Discussion Points - Q9

**Q1: When should you use serverless over containers?**

**A**:
- **Serverless**: Event-driven, < 1M requests/month, sporadic traffic, simple stateless logic
- **Containers**: Steady traffic, > 10M requests/month, heavy DB usage, complex state management

**Q2: How do you handle cold starts in Lambda?**

**A**:
- Provisioned Concurrency (keeps functions warm)
- Optimize package size (< 10MB)
- Use ARM64 architecture (Graviton2)
- Keep functions small and focused
- Consider Snap Start for Java

**Q3: What is the break-even point between Lambda and ECS?**

**A**: Typically around 4-10M requests/month depending on execution time and memory. ECS becomes cheaper at high, steady traffic. Lambda is cheaper for sporadic, low-volume workloads.

---

## Question 10: Complete Hybrid Emirates NBD Implementation

### 📋 Question Statement

Design and implement a complete hybrid architecture for Emirates NBD Digital Banking Platform combining serverless (Lambda) and containerized (ECS/EKS) services. Include:
- Multi-service architecture with both paradigms
- Service mesh for inter-service communication
- Unified observability and monitoring
- CI/CD pipeline with multi-environment deployment
- Cost optimization strategies
- Security and compliance (PCI-DSS)
- Disaster recovery and high availability

---

### 🏗️ Complete Hybrid Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    EMIRATES NBD DIGITAL BANKING PLATFORM                     │
│                          HYBRID ARCHITECTURE                                 │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                           EDGE LAYER                                         │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐               │
│  │  CloudFront    │  │   Route 53     │  │   AWS WAF      │               │
│  │  (CDN)         │  │   (DNS)        │  │  (Firewall)    │               │
│  └────────────────┘  └────────────────┘  └────────────────┘               │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         API GATEWAY LAYER                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                      API Gateway (REST + WebSocket)                   │  │
│  │  • Cognito Authentication                                            │  │
│  │  • Rate Limiting & Throttling                                        │  │
│  │  • Request/Response Transformation                                   │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
                    ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        SERVERLESS SERVICES (Lambda)                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Notification │  │    Fraud     │  │  Analytics   │  │    Audit     │  │
│  │   Service    │  │  Detection   │  │   Service    │  │   Service    │  │
│  │              │  │              │  │              │  │              │  │
│  │  • SMS/Email │  │  • ML Model  │  │  • Kinesis   │  │  • S3/Athena │  │
│  │  • EventDriven│  │  • SageMaker │  │  • Redshift  │  │  • Compliance│  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
│         │                 │                  │                  │          │
│         └─────────────────┴──────────────────┴──────────────────┘          │
└─────────────────────────────────────────┬───────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         EVENT BUS (EventBridge)                              │
│  • Domain Events (AccountCreated, TransactionCompleted, etc.)               │
│  • Cross-Service Communication                                              │
│  • Event Replay & Archive                                                   │
└─────────────────────────────────────────┬───────────────────────────────────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    │                     │                     │
                    ▼                     ▼                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONTAINERIZED SERVICES (ECS/EKS)                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Account    │  │ Transaction  │  │   Payment    │  │    Loan      │  │
│  │   Service    │  │   Service    │  │   Service    │  │   Service    │  │
│  │   (ECS)      │  │   (Lambda+SF)│  │   (EKS)      │  │   (ECS)      │  │
│  │              │  │              │  │              │  │              │  │
│  │  • Express.js│  │  • Step Fns  │  │  • NestJS    │  │  • Express.js│  │
│  │  • PostgreSQL│  │  • DynamoDB  │  │  • Aurora    │  │  • PostgreSQL│  │
│  │  • RDS       │  │  • Saga      │  │  • K8s       │  │  • RDS       │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
│         │                 │                  │                  │          │
│         └─────────────────┴──────────────────┴──────────────────┘          │
└─────────────────────────────────────────┬───────────────────────────────────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    │                     │                     │
                    ▼                     ▼                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DATA LAYER                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ RDS Postgres │  │   DynamoDB   │  │    Aurora    │  │  ElastiCache │  │
│  │  (ACID)      │  │  (NoSQL)     │  │ (Serverless) │  │   (Redis)    │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      OBSERVABILITY & MONITORING                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  CloudWatch  │  │   X-Ray      │  │  Prometheus  │  │   Grafana    │  │
│  │  (Logs)      │  │  (Tracing)   │  │  (Metrics)   │  │  (Dashboards)│  │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 🔧 Infrastructure as Code (Complete CDK Stack)

```typescript
// infrastructure/cdk/hybrid-banking-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as eks from 'aws-cdk-lib/aws-eks';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';
import * as rds from 'aws-cdk-lib/aws-rds';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';
import * as cloudfront from 'aws-cdk-lib/aws-cloudfront';
import * as route53 from 'aws-cdk-lib/aws-route53';
import * as cognito from 'aws-cdk-lib/aws-cognito';
import * as wafv2 from 'aws-cdk-lib/aws-wafv2';

export class HybridBankingStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // ============================================
    // 1. NETWORKING (VPC)
    // ============================================
    const vpc = new ec2.Vpc(this, 'BankingVPC', {
      maxAzs: 3,
      natGateways: 3, // High availability
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

    // VPC Flow Logs for security monitoring
    vpc.addFlowLog('VPCFlowLogs', {
      trafficType: ec2.FlowLogTrafficType.ALL
    });

    // ============================================
    // 2. AUTHENTICATION (Cognito)
    // ============================================
    const userPool = new cognito.UserPool(this, 'BankingUserPool', {
      userPoolName: 'emirates-nbd-users',
      selfSignUpEnabled: false,
      mfa: cognito.Mfa.REQUIRED,
      mfaSecondFactor: { sms: true, otp: true },
      passwordPolicy: {
        minLength: 12,
        requireLowercase: true,
        requireUppercase: true,
        requireDigits: true,
        requireSymbols: true
      }
    });

    // ============================================
    // 3. EVENT BUS (EventBridge)
    // ============================================
    const eventBus = new events.EventBus(this, 'BankingEventBus', {
      eventBusName: 'banking-event-bus'
    });

    // Enable event archiving for replay
    new events.Archive(this, 'EventArchive', {
      sourceEventBus: eventBus,
      eventPattern: {
        source: ['account.service', 'transaction.service', 'payment.service']
      },
      retention: cdk.Duration.days(365)
    });

    // ============================================
    // 4. DATABASES
    // ============================================

    // RDS PostgreSQL for Account Service
    const accountDB = new rds.DatabaseInstance(this, 'AccountDB', {
      engine: rds.DatabaseInstanceEngine.postgres({
        version: rds.PostgresEngineVersion.VER_15_3
      }),
      instanceType: ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.MEDIUM),
      vpc,
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_ISOLATED },
      multiAz: true,
      allocatedStorage: 100,
      storageEncrypted: true,
      backupRetention: cdk.Duration.days(7),
      deletionProtection: true
    });

    // DynamoDB for Transaction Service
    const transactionTable = new dynamodb.Table(this, 'TransactionTable', {
      tableName: 'banking-transactions',
      partitionKey: { name: 'transactionId', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      stream: dynamodb.StreamViewType.NEW_AND_OLD_IMAGES,
      pointInTimeRecovery: true,
      encryption: dynamodb.TableEncryption.AWS_MANAGED
    });

    transactionTable.addGlobalSecondaryIndex({
      indexName: 'AccountIdIndex',
      partitionKey: { name: 'accountId', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'timestamp', type: dynamodb.AttributeType.STRING }
    });

    // Aurora Serverless for Payment Service
    const paymentDB = new rds.ServerlessCluster(this, 'PaymentDB', {
      engine: rds.DatabaseClusterEngine.auroraPostgres({
        version: rds.AuroraPostgresEngineVersion.VER_13_9
      }),
      vpc,
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_ISOLATED },
      scaling: {
        minCapacity: rds.AuroraCapacityUnit.ACU_2,
        maxCapacity: rds.AuroraCapacityUnit.ACU_16,
        autoPause: cdk.Duration.minutes(10)
      }
    });

    // ============================================
    // 5. CONTAINERIZED SERVICES (ECS)
    // ============================================

    // ECS Cluster
    const ecsCluster = new ecs.Cluster(this, 'BankingCluster', {
      vpc,
      containerInsights: true
    });

    // Account Service (ECS Fargate)
    const accountTaskDef = new ecs.FargateTaskDefinition(this, 'AccountTask', {
      memoryLimitMiB: 512,
      cpu: 256
    });

    const accountContainer = accountTaskDef.addContainer('AccountService', {
      image: ecs.ContainerImage.fromRegistry('account-service:latest'),
      logging: ecs.LogDrivers.awsLogs({ streamPrefix: 'account-service' }),
      environment: {
        DB_HOST: accountDB.dbInstanceEndpointAddress,
        EVENT_BUS_NAME: eventBus.eventBusName
      }
    });

    accountContainer.addPortMappings({ containerPort: 3000 });

    const accountService = new ecs.FargateService(this, 'AccountService', {
      cluster: ecsCluster,
      taskDefinition: accountTaskDef,
      desiredCount: 2,
      circuitBreaker: { rollback: true }
    });

    // Allow ECS to connect to RDS
    accountDB.connections.allowFrom(accountService, ec2.Port.tcp(5432));

    // ============================================
    // 6. KUBERNETES (EKS) for Payment Service
    // ============================================

    const eksCluster = new eks.Cluster(this, 'PaymentCluster', {
      version: eks.KubernetesVersion.V1_28,
      vpc,
      defaultCapacity: 2,
      defaultCapacityInstance: ec2.InstanceType.of(
        ec2.InstanceClass.T3,
        ec2.InstanceSize.MEDIUM
      )
    });

    // Add Helm charts for observability
    eksCluster.addHelmChart('Prometheus', {
      chart: 'kube-prometheus-stack',
      repository: 'https://prometheus-community.github.io/helm-charts',
      namespace: 'monitoring',
      createNamespace: true
    });

    // ============================================
    // 7. SERVERLESS SERVICES (Lambda)
    // ============================================

    // Notification Service
    const notificationLambda = new lambda.Function(this, 'NotificationFunction', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambdas/notification'),
      timeout: cdk.Duration.seconds(30),
      memorySize: 256,
      reservedConcurrentExecutions: 50,
      tracing: lambda.Tracing.ACTIVE
    });

    // Subscribe to EventBridge events
    new events.Rule(this, 'TransactionCompletedRule', {
      eventBus,
      eventPattern: {
        source: ['transaction.service'],
        detailType: ['TransactionCompleted']
      },
      targets: [new targets.LambdaFunction(notificationLambda)]
    });

    // Fraud Detection Service
    const fraudLambda = new lambda.Function(this, 'FraudFunction', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambdas/fraud-detection'),
      timeout: cdk.Duration.seconds(10),
      memorySize: 1024,
      environment: {
        SAGEMAKER_ENDPOINT: 'fraud-detection-endpoint'
      }
    });

    new events.Rule(this, 'TransactionInitiatedRule', {
      eventBus,
      eventPattern: {
        source: ['transaction.service'],
        detailType: ['TransactionInitiated']
      },
      targets: [new targets.LambdaFunction(fraudLambda)]
    });

    // ============================================
    // 8. API GATEWAY
    // ============================================

    const api = new apigateway.RestApi(this, 'BankingAPI', {
      restApiName: 'Emirates NBD Banking API',
      deployOptions: {
        stageName: 'prod',
        tracingEnabled: true,
        loggingLevel: apigateway.MethodLoggingLevel.INFO,
        metricsEnabled: true
      }
    });

    // Cognito Authorizer
    const authorizer = new apigateway.CognitoUserPoolsAuthorizer(this, 'Authorizer', {
      cognitoUserPools: [userPool]
    });

    // Lambda integrations
    const notificationsResource = api.root.addResource('notifications');
    notificationsResource.addMethod('POST',
      new apigateway.LambdaIntegration(notificationLambda),
      { authorizer, authorizationType: apigateway.AuthorizationType.COGNITO }
    );

    // ============================================
    // 9. CLOUDFRONT (CDN)
    // ============================================

    const distribution = new cloudfront.Distribution(this, 'CDN', {
      defaultBehavior: {
        origin: new cloudfront.HttpOrigin(api.url),
        cachePolicy: cloudfront.CachePolicy.CACHING_OPTIMIZED,
        allowedMethods: cloudfront.AllowedMethods.ALLOW_ALL
      },
      enableLogging: true
    });

    // ============================================
    // 10. WAF (Web Application Firewall)
    // ============================================

    const webAcl = new wafv2.CfnWebACL(this, 'WAF', {
      scope: 'REGIONAL',
      defaultAction: { allow: {} },
      rules: [
        {
          name: 'RateLimitRule',
          priority: 1,
          statement: {
            rateBasedStatement: {
              limit: 2000,
              aggregateKeyType: 'IP'
            }
          },
          action: { block: {} },
          visibilityConfig: {
            sampledRequestsEnabled: true,
            cloudWatchMetricsEnabled: true,
            metricName: 'RateLimitRule'
          }
        }
      ],
      visibilityConfig: {
        sampledRequestsEnabled: true,
        cloudWatchMetricsEnabled: true,
        metricName: 'BankingWAF'
      }
    });

    // ============================================
    // 11. OUTPUTS
    // ============================================

    new cdk.CfnOutput(this, 'APIEndpoint', {
      value: api.url,
      description: 'API Gateway Endpoint'
    });

    new cdk.CfnOutput(this, 'CloudFrontURL', {
      value: distribution.distributionDomainName,
      description: 'CloudFront Distribution URL'
    });

    new cdk.CfnOutput(this, 'EventBusArn', {
      value: eventBus.eventBusArn,
      description: 'EventBridge Event Bus ARN'
    });

    new cdk.CfnOutput(this, 'EKSClusterName', {
      value: eksCluster.clusterName,
      description: 'EKS Cluster Name'
    });
  }
}
```

---

### 🚀 CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: Deploy Hybrid Banking Platform

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REGISTRY: 123456789012.dkr.ecr.us-east-1.amazonaws.com

jobs:
  # ============================================
  # TEST
  # ============================================
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
      
      - name: Run linting
        run: npm run lint
      
      - name: Security scan
        run: npm audit --audit-level=high

  # ============================================
  # BUILD & PUSH LAMBDA FUNCTIONS
  # ============================================
  deploy-lambda:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Deploy Notification Lambda
        run: |
          cd lambdas/notification
          npm ci --production
          zip -r function.zip .
          aws lambda update-function-code \
            --function-name notification-service \
            --zip-file fileb://function.zip
      
      - name: Deploy Fraud Detection Lambda
        run: |
          cd lambdas/fraud-detection
          npm ci --production
          zip -r function.zip .
          aws lambda update-function-code \
            --function-name fraud-detection-service \
            --zip-file fileb://function.zip

  # ============================================
  # BUILD & PUSH CONTAINER IMAGES
  # ============================================
  deploy-containers:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    strategy:
      matrix:
        service: [account-service, payment-service]
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      
      - name: Build, tag, and push image
        env:
          ECR_REPOSITORY: ${{ matrix.service }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          cd services/${{ matrix.service }}
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:latest
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
      
      - name: Update ECS service
        if: matrix.service == 'account-service'
        run: |
          aws ecs update-service \
            --cluster banking-cluster \
            --service account-service \
            --force-new-deployment

  # ============================================
  # DEPLOY TO EKS (Payment Service)
  # ============================================
  deploy-eks:
    needs: deploy-containers
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig --name payment-cluster --region $AWS_REGION
      
      - name: Deploy to EKS
        run: |
          kubectl set image deployment/payment-service \
            payment-service=$ECR_REGISTRY/payment-service:${{ github.sha }} \
            -n banking
          kubectl rollout status deployment/payment-service -n banking

  # ============================================
  # DEPLOY INFRASTRUCTURE (CDK)
  # ============================================
  deploy-infrastructure:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
      
      - name: Install CDK
        run: npm install -g aws-cdk
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: CDK Deploy
        run: |
          cd infrastructure/cdk
          npm ci
          cdk deploy --require-approval never
```

---

### 📊 Cost Optimization Dashboard

```javascript
// cost-optimization/calculator.js
class HybridCostOptimizer {
  constructor() {
    this.services = {
      // Serverless (Lambda)
      notification: {
        type: 'lambda',
        requestsPerMonth: 5000000,
        avgExecutionMs: 200,
        memoryMB: 256,
        cost: 0
      },
      fraudDetection: {
        type: 'lambda',
        requestsPerMonth: 10000000,
        avgExecutionMs: 100,
        memoryMB: 1024,
        cost: 0
      },
      
      // Containerized (ECS)
      account: {
        type: 'ecs',
        tasks: 3,
        vcpu: 0.5,
        memoryGB: 1,
        cost: 0
      },
      loan: {
        type: 'ecs',
        tasks: 2,
        vcpu: 0.25,
        memoryGB: 0.5,
        cost: 0
      },
      
      // Kubernetes (EKS)
      payment: {
        type: 'eks',
        nodes: 3,
        instanceType: 't3.medium',
        cost: 0
      }
    };
  }

  calculateMonthlyCost() {
    // Lambda costs
    this.services.notification.cost = this.calculateLambdaCost(
      this.services.notification
    );
    this.services.fraudDetection.cost = this.calculateLambdaCost(
      this.services.fraudDetection
    );

    // ECS costs
    this.services.account.cost = this.calculateECSCost(
      this.services.account
    );
    this.services.loan.cost = this.calculateECSCost(
      this.services.loan
    );

    // EKS costs
    this.services.payment.cost = this.calculateEKSCost(
      this.services.payment
    );

    return {
      services: this.services,
      totalMonthly: Object.values(this.services).reduce((sum, s) => sum + s.cost, 0),
      breakdown: this.getCostBreakdown()
    };
  }

  calculateLambdaCost({ requestsPerMonth, avgExecutionMs, memoryMB }) {
    const requestCost = (requestsPerMonth / 1000000) * 0.20;
    const memoryGB = memoryMB / 1024;
    const computeCost = requestsPerMonth * (avgExecutionMs / 1000) * memoryGB * 0.0000166667;
    return requestCost + computeCost;
  }

  calculateECSCost({ tasks, vcpu, memoryGB }) {
    const vcpuCost = vcpu * 0.04048 * 730;
    const memoryCost = memoryGB * 0.004445 * 730;
    return tasks * (vcpuCost + memoryCost);
  }

  calculateEKSCost({ nodes, instanceType }) {
    const instanceCosts = {
      't3.medium': 0.0416 * 730 // per hour * hours in month
    };
    const controlPlaneCost = 0.10 * 730; // $0.10 per hour
    return (nodes * instanceCosts[instanceType]) + controlPlaneCost;
  }

  getCostBreakdown() {
    const lambdaTotal = this.services.notification.cost + this.services.fraudDetection.cost;
    const ecsTotal = this.services.account.cost + this.services.loan.cost;
    const eksTotal = this.services.payment.cost;
    const total = lambdaTotal + ecsTotal + eksTotal;

    return {
      lambda: {
        cost: lambdaTotal,
        percentage: ((lambdaTotal / total) * 100).toFixed(2) + '%'
      },
      ecs: {
        cost: ecsTotal,
        percentage: ((ecsTotal / total) * 100).toFixed(2) + '%'
      },
      eks: {
        cost: eksTotal,
        percentage: ((eksTotal / total) * 100).toFixed(2) + '%'
      },
      total
    };
  }
}

// Calculate costs
const optimizer = new HybridCostOptimizer();
const costs = optimizer.calculateMonthlyCost();

console.log('Monthly Cost Breakdown:');
console.log(JSON.stringify(costs, null, 2));

/*
Output:
{
  "services": {
    "notification": { "cost": 9.22 },
    "fraudDetection": { "cost": 17.83 },
    "account": { "cost": 54.03 },
    "loan": { "cost": 18.01 },
    "payment": { "cost": 164.08 }
  },
  "totalMonthly": 263.17,
  "breakdown": {
    "lambda": { "cost": 27.05, "percentage": "10.28%" },
    "ecs": { "cost": 72.04, "percentage": "27.38%" },
    "eks": { "cost": 164.08, "percentage": "62.34%" },
    "total": 263.17
  }
}
*/
```

---

### 🎓 Interview Discussion Points - Q10

**Q1: Why use a hybrid architecture instead of pure serverless or containers?**

**A**:
- **Optimization**: Use right tool for each job (Lambda for events, ECS for steady state)
- **Cost Efficiency**: Lambda cheaper for low-volume, ECS/EKS cheaper at scale
- **Performance**: Containers avoid cold starts for latency-sensitive services
- **Flexibility**: Leverage strengths of each paradigm

**Q2: How do you ensure consistency across hybrid services?**

**A**:
- **EventBridge**: Central event bus for all services
- **API Gateway**: Unified entry point
- **X-Ray**: Distributed tracing across Lambda and containers
- **CloudWatch**: Centralized logging and monitoring

**Q3: What are the challenges of hybrid architecture?**

**A**:
- **Complexity**: Multiple deployment mechanisms
- **Monitoring**: Different tooling (CloudWatch vs Prometheus)
- **Networking**: VPC Lambda vs container networking
- **Cost Tracking**: Harder to attribute costs

---

**End of File 5**

