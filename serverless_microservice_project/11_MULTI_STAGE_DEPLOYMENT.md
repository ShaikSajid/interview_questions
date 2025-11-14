# Multi-Stage Deployment - Serverless Order Processing Microservice

## 📋 Overview

Complete guide for deploying across dev, staging, and production environments with proper isolation, configuration management, and promotion workflows.

---

## 1️⃣ Environment Configuration

### Stage Configuration File

```typescript
// config/stages.ts
export interface StageConfig {
  stage: string;
  region: string;
  vpcId: string;
  subnetIds: string[];
  dynamodb: {
    ordersTable: string;
    inventoryTable: string;
    billingMode: 'PAY_PER_REQUEST' | 'PROVISIONED';
    readCapacity?: number;
    writeCapacity?: number;
  };
  sqs: {
    orderQueueName: string;
    inventoryQueueName: string;
  };
  sns: {
    notificationTopicName: string;
  };
  lambda: {
    timeout: number;
    memorySize: number;
    reservedConcurrency?: number;
  };
  apiGateway: {
    throttle: {
      rateLimit: number;
      burstLimit: number;
    };
  };
  monitoring: {
    logRetentionDays: number;
    enableXRay: boolean;
  };
}

export const stageConfigs: Record<string, StageConfig> = {
  dev: {
    stage: 'dev',
    region: 'us-east-1',
    vpcId: 'vpc-dev-12345',
    subnetIds: ['subnet-dev-1', 'subnet-dev-2'],
    dynamodb: {
      ordersTable: 'orders-dev',
      inventoryTable: 'inventory-dev',
      billingMode: 'PAY_PER_REQUEST'
    },
    sqs: {
      orderQueueName: 'order-processing-dev.fifo',
      inventoryQueueName: 'inventory-update-dev'
    },
    sns: {
      notificationTopicName: 'order-notifications-dev'
    },
    lambda: {
      timeout: 30,
      memorySize: 256
    },
    apiGateway: {
      throttle: {
        rateLimit: 100,
        burstLimit: 200
      }
    },
    monitoring: {
      logRetentionDays: 7,
      enableXRay: true
    }
  },
  
  staging: {
    stage: 'staging',
    region: 'us-east-1',
    vpcId: 'vpc-staging-67890',
    subnetIds: ['subnet-staging-1', 'subnet-staging-2', 'subnet-staging-3'],
    dynamodb: {
      ordersTable: 'orders-staging',
      inventoryTable: 'inventory-staging',
      billingMode: 'PROVISIONED',
      readCapacity: 10,
      writeCapacity: 5
    },
    sqs: {
      orderQueueName: 'order-processing-staging.fifo',
      inventoryQueueName: 'inventory-update-staging'
    },
    sns: {
      notificationTopicName: 'order-notifications-staging'
    },
    lambda: {
      timeout: 30,
      memorySize: 512
    },
    apiGateway: {
      throttle: {
        rateLimit: 500,
        burstLimit: 1000
      }
    },
    monitoring: {
      logRetentionDays: 30,
      enableXRay: true
    }
  },
  
  production: {
    stage: 'production',
    region: 'us-east-1',
    vpcId: 'vpc-prod-11111',
    subnetIds: ['subnet-prod-1', 'subnet-prod-2', 'subnet-prod-3'],
    dynamodb: {
      ordersTable: 'orders-production',
      inventoryTable: 'inventory-production',
      billingMode: 'PROVISIONED',
      readCapacity: 50,
      writeCapacity: 25
    },
    sqs: {
      orderQueueName: 'order-processing-production.fifo',
      inventoryQueueName: 'inventory-update-production'
    },
    sns: {
      notificationTopicName: 'order-notifications-production'
    },
    lambda: {
      timeout: 30,
      memorySize: 1024,
      reservedConcurrency: 100
    },
    apiGateway: {
      throttle: {
        rateLimit: 5000,
        burstLimit: 10000
      }
    },
    monitoring: {
      logRetentionDays: 90,
      enableXRay: true
    }
  }
};

export function getStageConfig(stage: string): StageConfig {
  const config = stageConfigs[stage];
  if (!config) {
    throw new Error(`Unknown stage: ${stage}`);
  }
  return config;
}
```

---

## 2️⃣ Serverless Framework Multi-Stage

```yaml
# serverless.yml
service: order-processing

frameworkVersion: '3'

provider:
  name: aws
  runtime: nodejs18.x
  stage: ${opt:stage, 'dev'}
  region: ${self:custom.stages.${self:provider.stage}.region}
  
  vpc:
    securityGroupIds:
      - !Ref LambdaSecurityGroup
    subnetIds: ${self:custom.stages.${self:provider.stage}.subnetIds}
  
  environment:
    STAGE: ${self:provider.stage}
    ORDERS_TABLE: ${self:custom.stages.${self:provider.stage}.dynamodb.ordersTable}
    INVENTORY_TABLE: ${self:custom.stages.${self:provider.stage}.dynamodb.inventoryTable}
    ORDER_QUEUE_URL: 
      Ref: OrderQueue
    INVENTORY_QUEUE_URL:
      Ref: InventoryQueue
    NOTIFICATION_TOPIC_ARN:
      Ref: NotificationTopic
    NODE_ENV: ${self:provider.stage}
  
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:Query
            - dynamodb:Scan
            - dynamodb:GetItem
            - dynamodb:PutItem
            - dynamodb:UpdateItem
            - dynamodb:DeleteItem
          Resource:
            - arn:aws:dynamodb:${self:provider.region}:*:table/${self:custom.stages.${self:provider.stage}.dynamodb.ordersTable}
            - arn:aws:dynamodb:${self:provider.region}:*:table/${self:custom.stages.${self:provider.stage}.dynamodb.inventoryTable}
        - Effect: Allow
          Action:
            - sqs:SendMessage
            - sqs:ReceiveMessage
            - sqs:DeleteMessage
          Resource:
            - !GetAtt OrderQueue.Arn
            - !GetAtt InventoryQueue.Arn
        - Effect: Allow
          Action:
            - sns:Publish
          Resource:
            - !Ref NotificationTopic
        - Effect: Allow
          Action:
            - xray:PutTraceSegments
            - xray:PutTelemetryRecords
          Resource: '*'

custom:
  stages:
    dev:
      region: us-east-1
      subnetIds:
        - subnet-dev-1
        - subnet-dev-2
      dynamodb:
        ordersTable: orders-dev
        inventoryTable: inventory-dev
      lambda:
        timeout: 30
        memorySize: 256
      apiThrottle:
        rateLimit: 100
        burstLimit: 200
    
    staging:
      region: us-east-1
      subnetIds:
        - subnet-staging-1
        - subnet-staging-2
        - subnet-staging-3
      dynamodb:
        ordersTable: orders-staging
        inventoryTable: inventory-staging
      lambda:
        timeout: 30
        memorySize: 512
      apiThrottle:
        rateLimit: 500
        burstLimit: 1000
    
    production:
      region: us-east-1
      subnetIds:
        - subnet-prod-1
        - subnet-prod-2
        - subnet-prod-3
      dynamodb:
        ordersTable: orders-production
        inventoryTable: inventory-production
      lambda:
        timeout: 30
        memorySize: 1024
        reservedConcurrency: 100
      apiThrottle:
        rateLimit: 5000
        burstLimit: 10000

functions:
  api:
    handler: dist/handlers/api.handler
    timeout: ${self:custom.stages.${self:provider.stage}.lambda.timeout}
    memorySize: ${self:custom.stages.${self:provider.stage}.lambda.memorySize}
    reservedConcurrency: ${self:custom.stages.${self:provider.stage}.lambda.reservedConcurrency, null}
    events:
      - http:
          path: /orders
          method: post
          throttling:
            maxRequestsPerSecond: ${self:custom.stages.${self:provider.stage}.apiThrottle.rateLimit}
            maxConcurrentRequests: ${self:custom.stages.${self:provider.stage}.apiThrottle.burstLimit}
      - http:
          path: /orders/{orderId}
          method: get

  orderProcessor:
    handler: dist/handlers/orderProcessor.handler
    timeout: ${self:custom.stages.${self:provider.stage}.lambda.timeout}
    memorySize: ${self:custom.stages.${self:provider.stage}.lambda.memorySize}
    events:
      - sqs:
          arn: !GetAtt OrderQueue.Arn
          batchSize: 10
          functionResponseType: ReportBatchItemFailures

resources:
  Resources:
    OrderQueue:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: ${self:custom.stages.${self:provider.stage}.dynamodb.ordersTable}-queue.fifo
        FifoQueue: true
        ContentBasedDeduplication: true
    
    LambdaSecurityGroup:
      Type: AWS::EC2::SecurityGroup
      Properties:
        GroupDescription: Security group for Lambda functions
        VpcId: ${self:custom.stages.${self:provider.stage}.vpcId}
        SecurityGroupEgress:
          - IpProtocol: -1
            CidrIp: 0.0.0.0/0
```

---

## 3️⃣ Environment Variables Management

```typescript
// src/config/environment.ts
import { getStageConfig } from './stages';

export class Environment {
  private static instance: Environment;
  private config: any;

  private constructor() {
    const stage = process.env.STAGE || 'dev';
    this.config = getStageConfig(stage);
  }

  public static getInstance(): Environment {
    if (!Environment.instance) {
      Environment.instance = new Environment();
    }
    return Environment.instance;
  }

  get stage(): string {
    return this.config.stage;
  }

  get ordersTable(): string {
    return process.env.ORDERS_TABLE || this.config.dynamodb.ordersTable;
  }

  get inventoryTable(): string {
    return process.env.INVENTORY_TABLE || this.config.dynamodb.inventoryTable;
  }

  get orderQueueUrl(): string {
    return process.env.ORDER_QUEUE_URL || '';
  }

  get notificationTopicArn(): string {
    return process.env.NOTIFICATION_TOPIC_ARN || '';
  }

  get isProduction(): boolean {
    return this.stage === 'production';
  }

  get isStaging(): boolean {
    return this.stage === 'staging';
  }

  get isDevelopment(): boolean {
    return this.stage === 'dev';
  }
}
```

---

## 4️⃣ Blue-Green Deployment

```yaml
# serverless-blue-green.yml
service: order-processing

provider:
  name: aws
  runtime: nodejs18.x
  
  deploymentSettings:
    type: Linear10PercentEvery1Minute
    alias: Live
    preTrafficHook: preDeploymentCheck
    postTrafficHook: postDeploymentCheck
    alarms:
      - ApiErrorsAlarm
      - ApiLatencyAlarm

functions:
  api:
    handler: dist/handlers/api.handler
    
  preDeploymentCheck:
    handler: dist/hooks/preDeployment.handler
  
  postDeploymentCheck:
    handler: dist/hooks/postDeployment.handler

resources:
  Resources:
    ApiErrorsAlarm:
      Type: AWS::CloudWatch::Alarm
      Properties:
        AlarmName: ${self:service}-${self:provider.stage}-api-errors
        MetricName: Errors
        Namespace: AWS/Lambda
        Statistic: Sum
        Period: 60
        EvaluationPeriods: 2
        Threshold: 5
        ComparisonOperator: GreaterThanThreshold
```

### Pre-Deployment Hook

```typescript
// src/hooks/preDeployment.ts
import { CloudWatch } from 'aws-sdk';

const cloudwatch = new CloudWatch();

export const handler = async () => {
  console.log('Running pre-deployment checks...');
  
  // Check current error rate
  const errorRate = await getErrorRate();
  if (errorRate > 5) {
    throw new Error(`Current error rate too high: ${errorRate}%`);
  }
  
  // Check DynamoDB capacity
  const hasCapacity = await checkDynamoDBCapacity();
  if (!hasCapacity) {
    throw new Error('DynamoDB capacity insufficient');
  }
  
  console.log('Pre-deployment checks passed');
  return { statusCode: 200 };
};

async function getErrorRate(): Promise<number> {
  const params = {
    MetricDataQueries: [{
      Id: 'm1',
      MetricStat: {
        Metric: {
          Namespace: 'AWS/Lambda',
          MetricName: 'Errors',
          Dimensions: [{
            Name: 'FunctionName',
            Value: process.env.AWS_LAMBDA_FUNCTION_NAME || ''
          }]
        },
        Period: 300,
        Stat: 'Sum'
      }
    }],
    StartTime: new Date(Date.now() - 300000),
    EndTime: new Date()
  };
  
  const result = await cloudwatch.getMetricData(params).promise();
  return result.MetricDataResults?.[0]?.Values?.[0] || 0;
}

async function checkDynamoDBCapacity(): Promise<boolean> {
  // Check DynamoDB consumed capacity
  // Implementation here
  return true;
}
```

---

## 5️⃣ Canary Deployment

```yaml
# serverless-canary.yml
provider:
  deploymentSettings:
    type: Canary10Percent5Minutes
    alias: Live
```

### Deployment Types
- **Linear10PercentEvery1Minute**: Traffic shifts 10% every minute
- **Linear10PercentEvery10Minutes**: Traffic shifts 10% every 10 minutes
- **Canary10Percent5Minutes**: 10% for 5 minutes, then 100%
- **Canary10Percent10Minutes**: 10% for 10 minutes, then 100%
- **AllAtOnce**: Immediate full deployment

---

## 6️⃣ Stage Promotion Workflow

```powershell
# scripts/promote-stage.ps1
param(
    [Parameter(Mandatory=$true)]
    [ValidateSet('dev', 'staging', 'production')]
    [string]$FromStage,
    
    [Parameter(Mandatory=$true)]
    [ValidateSet('staging', 'production')]
    [string]$ToStage
)

Write-Host "Promoting from $FromStage to $ToStage..." -ForegroundColor Cyan

# Step 1: Run tests on source stage
Write-Host "`n1. Running integration tests on $FromStage..." -ForegroundColor Yellow
npm run test:integration -- --stage=$FromStage

if ($LASTEXITCODE -ne 0) {
    Write-Host "Tests failed on $FromStage. Aborting promotion." -ForegroundColor Red
    exit 1
}

# Step 2: Get current deployment version
Write-Host "`n2. Getting deployment version from $FromStage..." -ForegroundColor Yellow
$version = aws cloudformation describe-stacks `
    --stack-name order-processing-$FromStage `
    --query "Stacks[0].Tags[?Key=='Version'].Value" `
    --output text

Write-Host "Current version: $version" -ForegroundColor Green

# Step 3: Create backup of target stage
Write-Host "`n3. Creating backup of $ToStage..." -ForegroundColor Yellow
aws s3 cp s3://deployments/order-processing-$ToStage.zip `
    s3://deployments/backup/order-processing-$ToStage-$(Get-Date -Format 'yyyyMMdd-HHmmss').zip

# Step 4: Deploy to target stage
Write-Host "`n4. Deploying to $ToStage..." -ForegroundColor Yellow
serverless deploy --stage $ToStage --verbose

if ($LASTEXITCODE -ne 0) {
    Write-Host "Deployment failed. Rolling back..." -ForegroundColor Red
    ./scripts/rollback.ps1 -Stage $ToStage -Version $version
    exit 1
}

# Step 5: Run smoke tests
Write-Host "`n5. Running smoke tests on $ToStage..." -ForegroundColor Yellow
npm run test:smoke -- --stage=$ToStage

if ($LASTEXITCODE -ne 0) {
    Write-Host "Smoke tests failed. Rolling back..." -ForegroundColor Red
    ./scripts/rollback.ps1 -Stage $ToStage -Version $version
    exit 1
}

# Step 6: Monitor for 5 minutes
Write-Host "`n6. Monitoring $ToStage for 5 minutes..." -ForegroundColor Yellow
$endTime = (Get-Date).AddMinutes(5)

while ((Get-Date) -lt $endTime) {
    $errors = aws cloudwatch get-metric-statistics `
        --namespace AWS/Lambda `
        --metric-name Errors `
        --dimensions Name=FunctionName,Value=order-processing-$ToStage-api `
        --start-time ((Get-Date).AddMinutes(-5).ToString('o')) `
        --end-time (Get-Date).ToString('o') `
        --period 300 `
        --statistics Sum `
        --query "Datapoints[0].Sum" `
        --output text
    
    if ($errors -gt 10) {
        Write-Host "High error rate detected. Rolling back..." -ForegroundColor Red
        ./scripts/rollback.ps1 -Stage $ToStage -Version $version
        exit 1
    }
    
    Write-Host "No issues detected. Waiting..." -ForegroundColor Green
    Start-Sleep -Seconds 60
}

Write-Host "`nPromotion to $ToStage completed successfully!" -ForegroundColor Green
```

---

## 7️⃣ Cross-Stage Isolation

### Resource Naming Convention

```typescript
// src/utils/resourceNaming.ts
export class ResourceNaming {
  private stage: string;
  
  constructor(stage: string) {
    this.stage = stage;
  }
  
  // DynamoDB tables
  getOrdersTableName(): string {
    return `orders-${this.stage}`;
  }
  
  getInventoryTableName(): string {
    return `inventory-${this.stage}`;
  }
  
  // SQS queues
  getOrderQueueName(): string {
    return `order-processing-${this.stage}.fifo`;
  }
  
  getInventoryQueueName(): string {
    return `inventory-update-${this.stage}`;
  }
  
  // SNS topics
  getNotificationTopicName(): string {
    return `order-notifications-${this.stage}`;
  }
  
  // Lambda functions
  getLambdaFunctionName(functionName: string): string {
    return `order-processing-${this.stage}-${functionName}`;
  }
  
  // CloudWatch log groups
  getLogGroupName(functionName: string): string {
    return `/aws/lambda/order-processing-${this.stage}-${functionName}`;
  }
  
  // API Gateway
  getApiName(): string {
    return `order-processing-${this.stage}`;
  }
}
```

### IAM Boundary Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        },
        "StringLike": {
          "aws:PrincipalTag/Stage": "${aws:ResourceTag/Stage}"
        }
      }
    }
  ]
}
```

---

## 8️⃣ Deployment Scripts

### Deploy All Stages

```powershell
# scripts/deploy-all-stages.ps1
$stages = @('dev', 'staging', 'production')

foreach ($stage in $stages) {
    Write-Host "`nDeploying to $stage..." -ForegroundColor Cyan
    
    serverless deploy --stage $stage
    
    if ($LASTEXITCODE -ne 0) {
        Write-Host "Deployment to $stage failed" -ForegroundColor Red
        exit 1
    }
    
    Write-Host "$stage deployment successful" -ForegroundColor Green
    
    # Wait between deployments
    if ($stage -ne 'production') {
        Write-Host "Waiting 2 minutes before next deployment..."
        Start-Sleep -Seconds 120
    }
}

Write-Host "`nAll stages deployed successfully!" -ForegroundColor Green
```

---

## 🎯 Multi-Stage Best Practices

### DO ✅
1. Use separate AWS accounts for each stage
2. Implement strict IAM policies per stage
3. Tag all resources with stage identifier
4. Use different scaling configurations per stage
5. Implement automated smoke tests
6. Monitor deployments closely
7. Keep staging identical to production
8. Use blue-green or canary deployments
9. Automate promotion workflows
10. Document stage-specific configurations

### DON'T ❌
1. Don't share resources across stages
2. Don't skip testing in lower stages
3. Don't deploy to production without approval
4. Don't use production data in dev/staging
5. Don't ignore monitoring alerts

---

**Next:** [Testing Strategies](./12_TESTING_STRATEGIES.md)

---

**Last Updated**: November 12, 2025
