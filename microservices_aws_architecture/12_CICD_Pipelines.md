# CI/CD Pipelines

## Question 23: Complete CI/CD Implementation

### 📋 Question Statement

Implement a complete CI/CD pipeline for Emirates NBD microservices using GitHub Actions and AWS CodePipeline. Include multi-stage deployments (dev/staging/prod), automated testing, blue-green deployment, and canary releases.

---

### 🏗️ CI/CD Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│              EMIRATES NBD CI/CD PIPELINE ARCHITECTURE                       │
└────────────────────────────────────────────────────────────────────────────┘

    SOURCE                BUILD                TEST                DEPLOY
    ──────               ─────                ────                ──────
    
    ┌────────┐          ┌────────┐          ┌────────┐          ┌────────┐
    │ GitHub │          │ Docker │          │  Unit  │          │  Dev   │
    │  Repo  │─────────►│ Build  │─────────►│  Test  │─────────►│  Env   │
    └────────┘          └────────┘          └────────┘          └────────┘
                             │                    │                    │
                             │                    │                    │
                             ▼                    ▼                    ▼
                        ┌────────┐          ┌────────┐          ┌────────┐
                        │  ECR   │          │ E2E    │          │Staging │
                        │  Push  │          │ Test   │          │  Env   │
                        └────────┘          └────────┘          └────────┘
                                                 │                    │
                                                 │                    │
                                                 ▼                    ▼
                                            ┌────────┐          ┌────────┐
                                            │Security│          │  Prod  │
                                            │  Scan  │          │  Blue/ │
                                            └────────┘          │  Green │
                                                                └────────┘
```

---

### 🔧 GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy Banking Services

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: banking-services
  ECS_CLUSTER: banking-cluster
  ECS_SERVICE: account-service

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Run unit tests
        run: npm run test:unit
      
      - name: Run integration tests
        run: npm run test:integration
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info

  security-scan:
    name: Security Scanning
    runs-on: ubuntu-latest
    needs: test
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Run Snyk Security Scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'

  build-and-push:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest
    needs: [test, security-scan]
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}
          tags: |
            type=sha,prefix={{branch}}-
            type=ref,event=branch
            type=semver,pattern={{version}}
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            VERSION=${{ steps.meta.outputs.version }}

  deploy-dev:
    name: Deploy to Development
    runs-on: ubuntu-latest
    needs: build-and-push
    if: github.ref == 'refs/heads/develop'
    environment: development
    
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Update ECS Service
        run: |
          aws ecs update-service \
            --cluster ${{ env.ECS_CLUSTER }}-dev \
            --service ${{ env.ECS_SERVICE }} \
            --force-new-deployment
      
      - name: Wait for service stability
        run: |
          aws ecs wait services-stable \
            --cluster ${{ env.ECS_CLUSTER }}-dev \
            --services ${{ env.ECS_SERVICE }}

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build-and-push
    if: github.ref == 'refs/heads/main'
    environment: staging
    
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Update ECS Service
        run: |
          aws ecs update-service \
            --cluster ${{ env.ECS_CLUSTER }}-staging \
            --service ${{ env.ECS_SERVICE }} \
            --force-new-deployment
      
      - name: Run smoke tests
        run: |
          curl -f https://staging-api.emiratesnbd.com/health || exit 1

  deploy-production:
    name: Deploy to Production (Blue/Green)
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Deploy with Blue/Green
        run: |
          # Create new task definition revision
          NEW_TASK_DEF=$(aws ecs describe-task-definition \
            --task-definition ${{ env.ECS_SERVICE }} \
            --query taskDefinition | \
            jq '.containerDefinitions[0].image = "${{ needs.build-and-push.outputs.image-tag }}"' | \
            jq 'del(.taskDefinitionArn, .revision, .status, .requiresAttributes, .compatibilities, .registeredAt, .registeredBy)')
          
          # Register new task definition
          NEW_REVISION=$(echo $NEW_TASK_DEF | \
            aws ecs register-task-definition --cli-input-json file:///dev/stdin | \
            jq -r '.taskDefinition.revision')
          
          # Start CodeDeploy deployment
          aws deploy create-deployment \
            --application-name banking-app \
            --deployment-group-name production-dg \
            --revision revisionType=S3,s3Location={bucket=deployments,key=appspec.json}
      
      - name: Monitor deployment
        run: |
          DEPLOYMENT_ID=$(aws deploy list-deployments \
            --application-name banking-app \
            --deployment-group-name production-dg \
            --query 'deployments[0]' --output text)
          
          aws deploy wait deployment-successful --deployment-id $DEPLOYMENT_ID

  rollback:
    name: Rollback Production
    runs-on: ubuntu-latest
    if: failure() && github.ref == 'refs/heads/main'
    needs: deploy-production
    
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Trigger rollback
        run: |
          aws deploy stop-deployment \
            --deployment-id $DEPLOYMENT_ID \
            --auto-rollback-enabled
```

### 🚀 Blue/Green Deployment (CDK)

```typescript
// infrastructure/cdk/blue-green-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as elbv2 from 'aws-cdk-lib/aws-elasticloadbalancingv2';
import * as codedeploy from 'aws-cdk-lib/aws-codedeploy';

export class BlueGreenDeploymentStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Target Groups
    const blueTargetGroup = new elbv2.ApplicationTargetGroup(this, 'BlueTargetGroup', {
      vpc: props.vpc,
      port: 80,
      protocol: elbv2.ApplicationProtocol.HTTP,
      targetType: elbv2.TargetType.IP,
      healthCheck: {
        path: '/health',
        interval: cdk.Duration.seconds(30),
        timeout: cdk.Duration.seconds(5),
        healthyThresholdCount: 2,
        unhealthyThresholdCount: 3
      }
    });

    const greenTargetGroup = new elbv2.ApplicationTargetGroup(this, 'GreenTargetGroup', {
      vpc: props.vpc,
      port: 80,
      protocol: elbv2.ApplicationProtocol.HTTP,
      targetType: elbv2.TargetType.IP,
      healthCheck: {
        path: '/health',
        interval: cdk.Duration.seconds(30)
      }
    });

    // Application Load Balancer
    const alb = new elbv2.ApplicationLoadBalancer(this, 'ALB', {
      vpc: props.vpc,
      internetFacing: true
    });

    const prodListener = alb.addListener('ProdListener', {
      port: 443,
      protocol: elbv2.ApplicationProtocol.HTTPS,
      certificates: [props.certificate],
      defaultAction: elbv2.ListenerAction.forward([blueTargetGroup])
    });

    const testListener = alb.addListener('TestListener', {
      port: 8443,
      protocol: elbv2.ApplicationProtocol.HTTPS,
      certificates: [props.certificate],
      defaultAction: elbv2.ListenerAction.forward([greenTargetGroup])
    });

    // ECS Service
    const service = new ecs.FargateService(this, 'Service', {
      cluster: props.cluster,
      taskDefinition: props.taskDefinition,
      desiredCount: 3,
      deploymentController: {
        type: ecs.DeploymentControllerType.CODE_DEPLOY
      }
    });

    service.attachToApplicationTargetGroup(blueTargetGroup);

    // CodeDeploy Application
    const application = new codedeploy.EcsApplication(this, 'Application', {
      applicationName: 'banking-app'
    });

    // Deployment Group
    const deploymentGroup = new codedeploy.EcsDeploymentGroup(this, 'DeploymentGroup', {
      application,
      service,
      blueGreenDeploymentConfig: {
        blueTargetGroup,
        greenTargetGroup,
        listener: prodListener,
        testListener,
        terminationWaitTime: cdk.Duration.minutes(5)
      },
      deploymentConfig: codedeploy.EcsDeploymentConfig.CANARY_10PERCENT_5MINUTES,
      autoRollback: {
        failedDeployment: true,
        stoppedDeployment: true,
        deploymentInAlarm: true
      },
      alarms: [
        new cloudwatch.Alarm(this, 'HighErrorRate', {
          metric: new cloudwatch.Metric({
            namespace: 'AWS/ApplicationELB',
            metricName: 'HTTPCode_Target_5XX_Count',
            statistic: 'Sum',
            period: cdk.Duration.minutes(1)
          }),
          threshold: 10,
          evaluationPeriods: 2
        })
      ]
    });
  }
}
```

### 🎓 Interview Discussion Points - Q23

**Q1: What is blue/green deployment?**

**A**:
- Two identical environments (blue/green)
- Deploy to inactive environment
- Switch traffic instantly
- Quick rollback if issues

**Q2: What is canary deployment?**

**A**:
- Gradual traffic shift (10% → 50% → 100%)
- Monitor metrics during shift
- Auto-rollback on errors
- Less risky than big bang

**Q3: What should be in a CI/CD pipeline?**

**A**:
- **Build**: Compile, package
- **Test**: Unit, integration, E2E
- **Security**: Vulnerability scanning
- **Deploy**: Multi-stage with approvals

---

## Question 24: Infrastructure as Code

### 📋 Question Statement

Implement complete infrastructure as code for Emirates NBD using AWS CDK with TypeScript. Include all services (VPC, ECS, Lambda, RDS, DynamoDB), configuration management, and environment-specific deployments.

---

### 🏗️ IaC Structure

```
infrastructure/
├── bin/
│   └── app.ts                    # CDK App entry point
├── lib/
│   ├── networking-stack.ts       # VPC, subnets, security groups
│   ├── database-stack.ts         # RDS, DynamoDB
│   ├── compute-stack.ts          # ECS, Lambda
│   ├── monitoring-stack.ts       # CloudWatch, X-Ray
│   └── pipeline-stack.ts         # CI/CD
├── config/
│   ├── dev.ts
│   ├── staging.ts
│   └── prod.ts
└── cdk.json
```

### 📦 Complete CDK Application

```typescript
// bin/app.ts
#!/usr/bin/env node
import 'source-map-support/register';
import * as cdk from 'aws-cdk-lib';
import { NetworkingStack } from '../lib/networking-stack';
import { DatabaseStack } from '../lib/database-stack';
import { ComputeStack } from '../lib/compute-stack';
import { MonitoringStack } from '../lib/monitoring-stack';
import { getConfig } from '../config';

const app = new cdk.App();

const environment = app.node.tryGetContext('environment') || 'dev';
const config = getConfig(environment);

const env = {
  account: process.env.CDK_DEFAULT_ACCOUNT,
  region: process.env.CDK_DEFAULT_REGION || 'us-east-1'
};

// Networking
const networkingStack = new NetworkingStack(app, `Banking-Networking-${environment}`, {
  env,
  config
});

// Databases
const databaseStack = new DatabaseStack(app, `Banking-Database-${environment}`, {
  env,
  vpc: networkingStack.vpc,
  config
});

// Compute
const computeStack = new ComputeStack(app, `Banking-Compute-${environment}`, {
  env,
  vpc: networkingStack.vpc,
  database: databaseStack,
  config
});

// Monitoring
new MonitoringStack(app, `Banking-Monitoring-${environment}`, {
  env,
  cluster: computeStack.cluster,
  config
});

app.synth();
```

```typescript
// config/index.ts
export interface EnvironmentConfig {
  environment: string;
  vpcCidr: string;
  natGateways: number;
  ecsDesiredCount: number;
  rdsInstanceType: string;
  rdsMultiAz: boolean;
  enableBackups: boolean;
  logRetentionDays: number;
}

export function getConfig(env: string): EnvironmentConfig {
  const configs: Record<string, EnvironmentConfig> = {
    dev: {
      environment: 'dev',
      vpcCidr: '10.0.0.0/16',
      natGateways: 1,
      ecsDesiredCount: 1,
      rdsInstanceType: 't3.small',
      rdsMultiAz: false,
      enableBackups: false,
      logRetentionDays: 7
    },
    staging: {
      environment: 'staging',
      vpcCidr: '10.1.0.0/16',
      natGateways: 2,
      ecsDesiredCount: 2,
      rdsInstanceType: 't3.medium',
      rdsMultiAz: true,
      enableBackups: true,
      logRetentionDays: 30
    },
    prod: {
      environment: 'prod',
      vpcCidr: '10.2.0.0/16',
      natGateways: 3,
      ecsDesiredCount: 3,
      rdsInstanceType: 'r5.large',
      rdsMultiAz: true,
      enableBackups: true,
      logRetentionDays: 365
    }
  };

  return configs[env] || configs.dev;
}
```

### 🎓 Interview Discussion Points - Q24

**Q1: Why use Infrastructure as Code?**

**A**:
- **Version control**: Track changes
- **Reproducible**: Consistent environments
- **Automated**: No manual clicking
- **Testable**: Validate before deploy

**Q2: CDK vs CloudFormation vs Terraform?**

**A**:
- **CDK**: TypeScript/Python, AWS-native
- **CloudFormation**: JSON/YAML, AWS-native
- **Terraform**: HCL, multi-cloud

---

**End of File 12**