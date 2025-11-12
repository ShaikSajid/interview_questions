# Cost Optimization

## Question 33: Cost Management & Optimization

### 📋 Question Statement

Implement comprehensive cost optimization for Emirates NBD including right-sizing, reserved instances, savings plans, spot instances, and cost monitoring with AWS Cost Explorer and Budgets.

---

### 💰 Cost Optimization Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│              EMIRATES NBD COST OPTIMIZATION STRATEGY                        │
└────────────────────────────────────────────────────────────────────────────┘

    COMPUTE OPTIMIZATION              STORAGE OPTIMIZATION
    ───────────────────              ────────────────────
    
    ┌──────────────────┐             ┌──────────────────┐
    │  Reserved        │             │  S3 Intelligent  │
    │  Instances       │             │  Tiering         │
    │  (40% savings)   │             │  (70% savings)   │
    └──────────────────┘             └──────────────────┘
    
    ┌──────────────────┐             ┌──────────────────┐
    │  Savings Plans   │             │  EBS GP3 vs GP2  │
    │  (66% savings)   │             │  (20% savings)   │
    └──────────────────┘             └──────────────────┘
    
    ┌──────────────────┐             DATABASE OPTIMIZATION
    │  Spot Instances  │             ────────────────────
    │  (70% savings)   │             
    └──────────────────┘             ┌──────────────────┐
                                     │  Aurora          │
    ┌──────────────────┐             │  Serverless v2   │
    │  Auto-Scaling    │             │  (90% savings)   │
    │  (50% savings)   │             └──────────────────┘
    └──────────────────┘
```

### 🔧 Cost Optimization CDK Stack

```typescript
// infrastructure/cdk/cost-optimization-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as budgets from 'aws-cdk-lib/aws-budgets';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as subscriptions from 'aws-cdk-lib/aws-sns-subscriptions';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as ce from 'aws-cdk-lib/aws-ce';
import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';

export class CostOptimizationStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // ============================================
    // SNS TOPIC FOR COST ALERTS
    // ============================================
    const costAlertTopic = new sns.Topic(this, 'CostAlertTopic', {
      displayName: 'Cost Alerts for Banking Platform'
    });

    costAlertTopic.addSubscription(
      new subscriptions.EmailSubscription('finops-team@emiratesnbd.com')
    );

    // ============================================
    // MONTHLY BUDGET WITH ALERTS
    // ============================================
    new budgets.CfnBudget(this, 'MonthlyBudget', {
      budget: {
        budgetName: 'banking-monthly-budget',
        budgetType: 'COST',
        timeUnit: 'MONTHLY',
        budgetLimit: {
          amount: 50000, // $50,000 per month
          unit: 'USD'
        },
        costFilters: {
          TagKeyValue: ['Project$digital-banking']
        }
      },
      notificationsWithSubscribers: [
        {
          notification: {
            notificationType: 'ACTUAL',
            comparisonOperator: 'GREATER_THAN',
            threshold: 80, // Alert at 80%
            thresholdType: 'PERCENTAGE'
          },
          subscribers: [
            {
              subscriptionType: 'SNS',
              address: costAlertTopic.topicArn
            }
          ]
        },
        {
          notification: {
            notificationType: 'ACTUAL',
            comparisonOperator: 'GREATER_THAN',
            threshold: 90, // Critical alert at 90%
            thresholdType: 'PERCENTAGE'
          },
          subscribers: [
            {
              subscriptionType: 'SNS',
              address: costAlertTopic.topicArn
            }
          ]
        },
        {
          notification: {
            notificationType: 'FORECASTED',
            comparisonOperator: 'GREATER_THAN',
            threshold: 100, // Forecast alert
            thresholdType: 'PERCENTAGE'
          },
          subscribers: [
            {
              subscriptionType: 'SNS',
              address: costAlertTopic.topicArn
            }
          ]
        }
      ]
    });

    // ============================================
    // SERVICE-SPECIFIC BUDGETS
    // ============================================
    const services = [
      { name: 'ECS-Fargate', limit: 15000 },
      { name: 'RDS', limit: 12000 },
      { name: 'Lambda', limit: 5000 },
      { name: 'DynamoDB', limit: 8000 },
      { name: 'S3', limit: 3000 }
    ];

    services.forEach(service => {
      new budgets.CfnBudget(this, `${service.name}Budget`, {
        budget: {
          budgetName: `banking-${service.name.toLowerCase()}-budget`,
          budgetType: 'COST',
          timeUnit: 'MONTHLY',
          budgetLimit: {
            amount: service.limit,
            unit: 'USD'
          },
          costFilters: {
            Service: [service.name]
          }
        },
        notificationsWithSubscribers: [
          {
            notification: {
              notificationType: 'ACTUAL',
              comparisonOperator: 'GREATER_THAN',
              threshold: 90,
              thresholdType: 'PERCENTAGE'
            },
            subscribers: [
              {
                subscriptionType: 'SNS',
                address: costAlertTopic.topicArn
              }
            ]
          }
        ]
      });
    });

    // ============================================
    // COST ANOMALY DETECTION
    // ============================================
    new ce.CfnAnomalyMonitor(this, 'CostAnomalyMonitor', {
      monitorName: 'banking-cost-anomaly-monitor',
      monitorType: 'DIMENSIONAL',
      monitorDimension: 'SERVICE'
    });

    const anomalySubscription = new ce.CfnAnomalySubscription(this, 'AnomalySubscription', {
      subscriptionName: 'banking-anomaly-alerts',
      threshold: 100, // $100 anomaly threshold
      frequency: 'IMMEDIATE',
      monitorArnList: [
        this.formatArn({
          service: 'ce',
          resource: 'anomalymonitor',
          resourceName: 'banking-cost-anomaly-monitor'
        })
      ],
      subscribers: [
        {
          type: 'SNS',
          address: costAlertTopic.topicArn
        }
      ]
    });

    // ============================================
    // AUTOMATED COST OPTIMIZATION LAMBDA
    // ============================================
    const costOptimizerLambda = new lambda.Function(this, 'CostOptimizer', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/cost-optimizer'),
      timeout: cdk.Duration.minutes(5),
      environment: {
        ALERT_TOPIC_ARN: costAlertTopic.topicArn
      }
    });

    // Grant permissions
    costOptimizerLambda.addToRolePolicy(new cdk.aws_iam.PolicyStatement({
      actions: [
        'ec2:DescribeInstances',
        'ec2:DescribeVolumes',
        'rds:DescribeDBInstances',
        'lambda:ListFunctions',
        'ce:GetCostAndUsage',
        'ce:GetRightsizingRecommendation',
        'cloudwatch:PutMetricData'
      ],
      resources: ['*']
    }));

    costAlertTopic.grantPublish(costOptimizerLambda);

    // Schedule daily cost optimization checks
    new events.Rule(this, 'DailyCostCheck', {
      schedule: events.Schedule.cron({ hour: '9', minute: '0' }),
      targets: [new targets.LambdaFunction(costOptimizerLambda)]
    });
  }
}
```

### 📊 Cost Optimizer Lambda

```typescript
// lambda/cost-optimizer/index.ts
import { 
  EC2Client, 
  DescribeInstancesCommand,
  DescribeVolumesCommand 
} from '@aws-sdk/client-ec2';
import { 
  RDSClient, 
  DescribeDBInstancesCommand 
} from '@aws-sdk/client-rds';
import { 
  CostExplorerClient, 
  GetRightsizingRecommendationCommand,
  GetCostAndUsageCommand 
} from '@aws-sdk/client-cost-explorer';
import { SNSClient, PublishCommand } from '@aws-sdk/client-sns';

const ec2 = new EC2Client({});
const rds = new RDSClient({});
const ce = new CostExplorerClient({});
const sns = new SNSClient({});

interface CostSaving {
  resourceType: string;
  resourceId: string;
  currentCost: number;
  potentialSaving: number;
  recommendation: string;
}

export const handler = async () => {
  console.log('Starting cost optimization analysis');

  const savings: CostSaving[] = [];

  // Check for idle EC2 instances
  const idleInstances = await findIdleEC2Instances();
  savings.push(...idleInstances);

  // Check for underutilized RDS instances
  const underutilizedRDS = await findUnderutilizedRDS();
  savings.push(...underutilizedRDS);

  // Check for unattached EBS volumes
  const unattachedVolumes = await findUnattachedVolumes();
  savings.push(...unattachedVolumes);

  // Get AWS rightsizing recommendations
  const rightsizingRecommendations = await getRightsizingRecommendations();
  savings.push(...rightsizingRecommendations);

  // Calculate total potential savings
  const totalSavings = savings.reduce((sum, s) => sum + s.potentialSaving, 0);

  // Generate report
  const report = generateReport(savings, totalSavings);

  // Send alert if significant savings found
  if (totalSavings > 1000) {
    await sendCostAlert(report);
  }

  console.log(`Total potential monthly savings: $${totalSavings.toFixed(2)}`);

  return {
    statusCode: 200,
    body: JSON.stringify({
      totalSavings,
      savingsCount: savings.length,
      savings
    })
  };
};

async function findIdleEC2Instances(): Promise<CostSaving[]> {
  const savings: CostSaving[] = [];

  const response = await ec2.send(new DescribeInstancesCommand({}));

  for (const reservation of response.Reservations || []) {
    for (const instance of reservation.Instances || []) {
      // Check if instance has low CPU (would need CloudWatch metrics in real implementation)
      if (instance.State?.Name === 'running') {
        // Estimate cost based on instance type
        const monthlyCost = estimateEC2Cost(instance.InstanceType || '');
        
        savings.push({
          resourceType: 'EC2',
          resourceId: instance.InstanceId || '',
          currentCost: monthlyCost,
          potentialSaving: monthlyCost,
          recommendation: `Stop or terminate idle instance ${instance.InstanceId}`
        });
      }
    }
  }

  return savings;
}

async function findUnderutilizedRDS(): Promise<CostSaving[]> {
  const savings: CostSaving[] = [];

  const response = await rds.send(new DescribeDBInstancesCommand({}));

  for (const db of response.DBInstances || []) {
    const currentClass = db.DBInstanceClass || '';
    const monthlyCost = estimateRDSCost(currentClass);

    // Check if we can downsize (simplified logic)
    if (currentClass.includes('large') && !currentClass.includes('xlarge')) {
      const recommendedClass = currentClass.replace('large', 'medium');
      const newCost = estimateRDSCost(recommendedClass);

      savings.push({
        resourceType: 'RDS',
        resourceId: db.DBInstanceIdentifier || '',
        currentCost: monthlyCost,
        potentialSaving: monthlyCost - newCost,
        recommendation: `Downsize from ${currentClass} to ${recommendedClass}`
      });
    }
  }

  return savings;
}

async function findUnattachedVolumes(): Promise<CostSaving[]> {
  const savings: CostSaving[] = [];

  const response = await ec2.send(new DescribeVolumesCommand({
    Filters: [
      {
        Name: 'status',
        Values: ['available'] // Unattached volumes
      }
    ]
  }));

  for (const volume of response.Volumes || []) {
    const size = volume.Size || 0;
    const monthlyCost = size * 0.10; // $0.10 per GB/month for gp3

    savings.push({
      resourceType: 'EBS',
      resourceId: volume.VolumeId || '',
      currentCost: monthlyCost,
      potentialSaving: monthlyCost,
      recommendation: `Delete unattached volume ${volume.VolumeId}`
    });
  }

  return savings;
}

async function getRightsizingRecommendations(): Promise<CostSaving[]> {
  const savings: CostSaving[] = [];

  try {
    const response = await ce.send(
      new GetRightsizingRecommendationCommand({
        Service: 'AmazonEC2',
        Configuration: {
          RecommendationTarget: 'SAME_INSTANCE_FAMILY',
          BenefitsConsidered: true
        }
      })
    );

    for (const recommendation of response.RightsizingRecommendations || []) {
      if (recommendation.CurrentInstance && recommendation.RightsizingType === 'Modify') {
        savings.push({
          resourceType: 'EC2',
          resourceId: recommendation.CurrentInstance.ResourceId || '',
          currentCost: parseFloat(recommendation.CurrentInstance.MonthlyCost || '0'),
          potentialSaving: parseFloat(recommendation.CurrentInstance.MonthlyCost || '0') - 
                          parseFloat(recommendation.ModifyRecommendationDetail?.TargetInstances?.[0]?.EstimatedMonthlyCost || '0'),
          recommendation: `Rightsize to ${recommendation.ModifyRecommendationDetail?.TargetInstances?.[0]?.ResourceDetails?.EC2ResourceDetails?.InstanceType}`
        });
      }
    }
  } catch (error) {
    console.error('Error getting rightsizing recommendations:', error);
  }

  return savings;
}

function estimateEC2Cost(instanceType: string): number {
  // Simplified cost estimation (actual costs vary by region)
  const costs: Record<string, number> = {
    't3.micro': 7.5,
    't3.small': 15,
    't3.medium': 30,
    't3.large': 60,
    'm5.large': 70,
    'm5.xlarge': 140,
    'c5.large': 62,
    'r5.large': 92
  };

  return costs[instanceType] || 50; // Default estimate
}

function estimateRDSCost(instanceClass: string): number {
  // Simplified cost estimation
  const costs: Record<string, number> = {
    'db.t3.micro': 12,
    'db.t3.small': 24,
    'db.t3.medium': 48,
    'db.t3.large': 96,
    'db.r5.large': 182,
    'db.r5.xlarge': 365
  };

  return costs[instanceClass] || 100;
}

function generateReport(savings: CostSaving[], total: number): string {
  let report = `Cost Optimization Report - ${new Date().toISOString()}\n\n`;
  report += `Total Potential Monthly Savings: $${total.toFixed(2)}\n`;
  report += `Number of Optimization Opportunities: ${savings.length}\n\n`;
  report += `Breakdown by Resource Type:\n`;

  const byType = savings.reduce((acc, s) => {
    acc[s.resourceType] = (acc[s.resourceType] || 0) + s.potentialSaving;
    return acc;
  }, {} as Record<string, number>);

  Object.entries(byType).forEach(([type, amount]) => {
    report += `  ${type}: $${amount.toFixed(2)}\n`;
  });

  report += `\nTop 10 Savings Opportunities:\n`;
  savings
    .sort((a, b) => b.potentialSaving - a.potentialSaving)
    .slice(0, 10)
    .forEach((s, i) => {
      report += `  ${i + 1}. ${s.resourceType} ${s.resourceId}: $${s.potentialSaving.toFixed(2)}\n`;
      report += `     ${s.recommendation}\n`;
    });

  return report;
}

async function sendCostAlert(report: string) {
  await sns.send(
    new PublishCommand({
      TopicArn: process.env.ALERT_TOPIC_ARN,
      Subject: 'Cost Optimization Report - Action Required',
      Message: report
    })
  );
}
```

### 💡 Cost Optimization Strategies

```typescript
// services/cost/optimization-strategies.ts
export class CostOptimizationStrategies {
  
  // 1. Compute Savings
  static getComputeRecommendations() {
    return {
      reservedInstances: {
        strategy: 'Purchase 1-year or 3-year RIs for baseline capacity',
        savings: 'Up to 72% vs On-Demand',
        useCase: 'Predictable workloads (account service, transaction service)',
        example: 'Reserved m5.large: $50/month vs On-Demand: $70/month'
      },
      
      savingsPlans: {
        strategy: 'Compute Savings Plans for flexible compute',
        savings: 'Up to 66% vs On-Demand',
        useCase: 'Mixed workloads (Lambda + Fargate + EC2)',
        example: '$1000/month commitment = $1500-1700 worth of compute'
      },
      
      spotInstances: {
        strategy: 'Use Spot for batch jobs, dev/test environments',
        savings: 'Up to 90% vs On-Demand',
        useCase: 'Report generation, data processing, testing',
        example: 'Spot m5.large: $7/month vs On-Demand: $70/month'
      },
      
      autoScaling: {
        strategy: 'Scale down during off-peak hours',
        savings: '40-60% on compute costs',
        useCase: 'Scale ECS services 3 → 1 during nights/weekends',
        example: 'Running 24/7: $2100/month → Scaled: $1000/month'
      },
      
      graviton: {
        strategy: 'Switch to ARM-based Graviton2/3 processors',
        savings: '20% better price/performance',
        useCase: 'Lambda, ECS, RDS read replicas',
        example: 'm6g.large: $56/month vs m5.large: $70/month'
      }
    };
  }

  // 2. Storage Savings
  static getStorageRecommendations() {
    return {
      s3IntelligentTiering: {
        strategy: 'Automatically move objects between access tiers',
        savings: 'Up to 70% on infrequent access',
        useCase: 'Document storage, backups, archives',
        cost: 'No overhead, $0.0025 per 1000 objects monitoring'
      },
      
      s3Lifecycle: {
        strategy: 'Transition old data to cheaper storage classes',
        policy: [
          '0-30 days: S3 Standard ($0.023/GB)',
          '30-90 days: S3 IA ($0.0125/GB)',
          '90-365 days: S3 Glacier ($0.004/GB)',
          '365+ days: S3 Glacier Deep Archive ($0.00099/GB)'
        ],
        savings: '95% vs keeping all in S3 Standard'
      },
      
      ebsGP3: {
        strategy: 'Migrate from gp2 to gp3 volumes',
        savings: '20% cheaper, 20% better performance',
        example: '100GB gp2: $10/month → gp3: $8/month'
      },
      
      ebsSnapshots: {
        strategy: 'Delete old snapshots beyond retention policy',
        savings: '$0.05/GB/month',
        example: 'Delete 1TB of old snapshots = $50/month saved'
      }
    };
  }

  // 3. Database Savings
  static getDatabaseRecommendations() {
    return {
      auroraServerless: {
        strategy: 'Use Aurora Serverless v2 for variable workloads',
        savings: 'Up to 90% for infrequent usage',
        useCase: 'Dev/test databases, staging environments',
        example: 'Dev database: $200/month → $20/month with auto-pause'
      },
      
      readReplicas: {
        strategy: 'Stop read replicas during off-hours',
        savings: '67% (16 hours/day stopped)',
        useCase: 'Analytics read replicas only needed during business hours',
        example: '$350/month → $115/month'
      },
      
      dynamoDBOnDemand: {
        strategy: 'Switch from provisioned to on-demand for unpredictable traffic',
        savings: 'Pay only for what you use',
        useCase: 'New services, testing, variable load',
        note: 'Provisioned cheaper for consistent traffic'
      }
    };
  }

  // 4. Network Savings
  static getNetworkRecommendations() {
    return {
      natGateways: {
        strategy: 'Reduce number of NAT Gateways',
        cost: '$32.40/month per NAT Gateway + data transfer',
        recommendation: 'Use 1 per AZ (3 total) instead of 1 per subnet',
        savings: '$100-500/month depending on setup'
      },
      
      vpcEndpoints: {
        strategy: 'Use VPC endpoints for S3/DynamoDB',
        savings: 'Eliminate NAT Gateway data transfer costs',
        example: '1TB S3 transfer via NAT: $45 → via endpoint: $0'
      },
      
      cloudFront: {
        strategy: 'Cache static content at edge locations',
        savings: 'Reduce origin data transfer costs by 80%',
        useCase: 'Mobile app static assets, web assets'
      }
    };
  }
}
```

### 🎓 Interview Discussion Points - Q33

**Q1: What is the difference between Reserved Instances and Savings Plans?**

**A**:
- **Reserved Instances (RI)**:
  - Specific instance type (e.g., m5.large)
  - Regional or zonal commitment
  - Up to 72% discount
  - Less flexible

- **Savings Plans**:
  - Flexible across instance types, regions, and compute services
  - Hourly spend commitment (e.g., $10/hour)
  - Up to 66% discount
  - More flexible (covers Lambda, Fargate, EC2)

**Q2: When should you use Spot Instances?**

**A**:
- **Good for**:
  - Batch jobs (report generation, data processing)
  - CI/CD pipelines
  - Dev/test environments
  - Fault-tolerant applications
  - Big data processing (EMR)

- **NOT good for**:
  - Production databases
  - Stateful applications
  - Real-time transaction processing
  - Applications requiring guaranteed availability

**Q3: What is S3 Intelligent-Tiering?**

**A**:
- **Automatically moves objects** between access tiers based on usage
- **Tiers**:
  - Frequent Access (first 30 days)
  - Infrequent Access (30-90 days)
  - Archive Instant Access (90-180 days)
  - Archive Access (180+ days)
- **Cost**: $0.0025 per 1,000 objects monitoring fee
- **Savings**: Up to 70% vs S3 Standard for infrequent access

**Q4: How do you implement cost allocation and chargeback?**

**A**:
- **Tag all resources** with cost allocation tags (Environment, Service, Team, CostCenter)
- **Activate tags** in Cost Allocation Tags console
- **Use Cost Explorer** to filter and group by tags
- **Generate reports** for each team/service/environment
- **Automated dashboards** with QuickSight or custom tools
- **Showback**: Share costs to raise awareness (no actual billing)
- **Chargeback**: Actually bill teams for their usage

**Q5: What is the typical cost breakdown for a microservices application?**

**A**:
- **Compute (40-50%)**: ECS/Lambda/EC2
- **Database (20-30%)**: RDS/DynamoDB
- **Storage (10-15%)**: S3/EBS
- **Network (10-15%)**: Data transfer, NAT Gateways, Load Balancers
- **Other (5-10%)**: CloudWatch, Secrets Manager, etc.

**Optimization priority**: Focus on compute and database first for maximum impact.

---

## Question 34: Cost Allocation & Tagging Strategy

### 📋 Question Statement

Implement comprehensive cost allocation and tagging strategy for Emirates NBD to track costs by service, environment, team, and business unit with automated budget alerts and showback reports.

---

### 🏷️ Tagging Strategy

```typescript
// infrastructure/cdk/tagging-strategy.ts
import * as cdk from 'aws-cdk-lib';
import { Tags } from 'aws-cdk-lib';

export interface BankingTags {
  Environment: 'dev' | 'staging' | 'production';
  Service: string;
  BusinessUnit: 'retail' | 'corporate' | 'investment' | 'shared';
  CostCenter: string;
  Owner: string;
  Project: string;
  DataClassification: 'public' | 'internal' | 'confidential' | 'restricted';
}

export class TaggingStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Apply mandatory tags to all resources in this stack
    this.applyMandatoryTags({
      Environment: 'production',
      Service: 'account-service',
      BusinessUnit: 'retail',
      CostCenter: 'CC-9001',
      Owner: 'platform-team',
      Project: 'digital-banking',
      DataClassification: 'confidential'
    });
  }

  private applyMandatoryTags(tags: BankingTags) {
    Object.entries(tags).forEach(([key, value]) => {
      Tags.of(this).add(key, value);
    });
  }
}

// Tag Compliance Lambda
export class TagComplianceChecker {
  async checkResourceTags(resourceArn: string): Promise<{
    compliant: boolean;
    missingTags: string[];
  }> {
    const requiredTags = [
      'Environment',
      'Service',
      'BusinessUnit',
      'CostCenter',
      'Owner'
    ];

    const resourceTags = await this.getResourceTags(resourceArn);
    const missingTags = requiredTags.filter(tag => !resourceTags[tag]);

    return {
      compliant: missingTags.length === 0,
      missingTags
    };
  }

  private async getResourceTags(resourceArn: string): Promise<Record<string, string>> {
    // AWS SDK call to get tags
    return {};
  }
}
```

### 💰 Cost Allocation CDK Stack

```typescript
// infrastructure/cdk/cost-allocation-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as budgets from 'aws-cdk-lib/aws-budgets';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as subscriptions from 'aws-cdk-lib/aws-sns-subscriptions';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';

export class CostAllocationStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // SNS Topic for budget alerts
    const budgetAlertTopic = new sns.Topic(this, 'BudgetAlerts', {
      displayName: 'Emirates NBD Budget Alerts'
    });

    budgetAlertTopic.addSubscription(
      new subscriptions.EmailSubscription('platform-team@emiratesnbd.com')
    );

    // Budget for Account Service (Production)
    new budgets.CfnBudget(this, 'AccountServiceBudget', {
      budget: {
        budgetName: 'account-service-production',
        budgetType: 'COST',
        timeUnit: 'MONTHLY',
        budgetLimit: {
          amount: 5000,
          unit: 'USD'
        },
        costFilters: {
          TagKeyValue: [
            'user:Service$account-service',
            'user:Environment$production'
          ]
        }
      },
      notificationsWithSubscribers: [
        {
          notification: {
            notificationType: 'ACTUAL',
            comparisonOperator: 'GREATER_THAN',
            threshold: 80,
            thresholdType: 'PERCENTAGE'
          },
          subscribers: [
            {
              subscriptionType: 'SNS',
              address: budgetAlertTopic.topicArn
            }
          ]
        },
        {
          notification: {
            notificationType: 'FORECASTED',
            comparisonOperator: 'GREATER_THAN',
            threshold: 100,
            thresholdType: 'PERCENTAGE'
          },
          subscribers: [
            {
              subscriptionType: 'SNS',
              address: budgetAlertTopic.topicArn
            }
          ]
        }
      ]
    });

    // Budget for entire Retail Banking business unit
    new budgets.CfnBudget(this, 'RetailBankingBudget', {
      budget: {
        budgetName: 'retail-banking-all',
        budgetType: 'COST',
        timeUnit: 'MONTHLY',
        budgetLimit: {
          amount: 50000,
          unit: 'USD'
        },
        costFilters: {
          TagKeyValue: ['user:BusinessUnit$retail']
        }
      },
      notificationsWithSubscribers: [
        {
          notification: {
            notificationType: 'ACTUAL',
            comparisonOperator: 'GREATER_THAN',
            threshold: 90,
            thresholdType: 'PERCENTAGE'
          },
          subscribers: [
            {
              subscriptionType: 'SNS',
              address: budgetAlertTopic.topicArn
            }
          ]
        }
      ]
    });

    // Lambda for cost anomaly detection
    const costAnomalyFunction = new lambda.Function(this, 'CostAnomalyDetector', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromInline(`
        const AWS = require('aws-sdk');
        const costExplorer = new AWS.CostExplorer();
        const sns = new AWS.SNS();

        exports.handler = async (event) => {
          const endDate = new Date().toISOString().split('T')[0];
          const startDate = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000)
            .toISOString()
            .split('T')[0];

          const params = {
            TimePeriod: { Start: startDate, End: endDate },
            Granularity: 'DAILY',
            Metrics: ['UnblendedCost'],
            GroupBy: [
              { Type: 'TAG', Key: 'Service' },
              { Type: 'TAG', Key: 'Environment' }
            ]
          };

          const result = await costExplorer.getCostAndUsage(params).promise();
          
          // Check for anomalies (cost increase > 50% day-over-day)
          const anomalies = detectAnomalies(result.ResultsByTime);
          
          if (anomalies.length > 0) {
            await sns.publish({
              TopicArn: process.env.ALERT_TOPIC_ARN,
              Subject: 'Cost Anomaly Detected',
              Message: JSON.stringify(anomalies, null, 2)
            }).promise();
          }
          
          return { anomaliesDetected: anomalies.length };
        };

        function detectAnomalies(results) {
          const anomalies = [];
          for (let i = 1; i < results.length; i++) {
            const yesterday = parseFloat(results[i-1].Total.UnblendedCost.Amount);
            const today = parseFloat(results[i].Total.UnblendedCost.Amount);
            
            if (today > yesterday * 1.5) {
              anomalies.push({
                date: results[i].TimePeriod.Start,
                increase: ((today - yesterday) / yesterday * 100).toFixed(2) + '%',
                cost: today
              });
            }
          }
          return anomalies;
        }
      `),
      environment: {
        ALERT_TOPIC_ARN: budgetAlertTopic.topicArn
      }
    });

    // Run daily at 9 AM
    const rule = new events.Rule(this, 'DailyCostCheck', {
      schedule: events.Schedule.cron({ hour: '9', minute: '0' })
    });

    rule.addTarget(new targets.LambdaFunction(costAnomalyFunction));
    budgetAlertTopic.grantPublish(costAnomalyFunction);
  }
}
```

### 📊 Cost Explorer API Usage

```typescript
// services/cost-reporting/cost-explorer-service.ts
import { CostExplorer } from 'aws-sdk';

export class CostExplorerService {
  private ce: CostExplorer;

  constructor() {
    this.ce = new CostExplorer({ region: 'us-east-1' });
  }

  async getServiceCosts(
    service: string,
    startDate: string,
    endDate: string
  ): Promise<CostReport> {
    const params: CostExplorer.GetCostAndUsageRequest = {
      TimePeriod: {
        Start: startDate,
        End: endDate
      },
      Granularity: 'DAILY',
      Metrics: ['UnblendedCost', 'UsageQuantity'],
      Filter: {
        Tags: {
          Key: 'Service',
          Values: [service]
        }
      },
      GroupBy: [
        { Type: 'DIMENSION', Key: 'SERVICE' },
        { Type: 'TAG', Key: 'Environment' }
      ]
    };

    const result = await this.ce.getCostAndUsage(params).promise();
    return this.transformCostData(result);
  }

  async getCostByBusinessUnit(
    businessUnit: string,
    month: string
  ): Promise<BusinessUnitCostReport> {
    const startDate = `${month}-01`;
    const endDate = this.getLastDayOfMonth(month);

    const params: CostExplorer.GetCostAndUsageRequest = {
      TimePeriod: { Start: startDate, End: endDate },
      Granularity: 'MONTHLY',
      Metrics: ['UnblendedCost'],
      Filter: {
        Tags: {
          Key: 'BusinessUnit',
          Values: [businessUnit]
        }
      },
      GroupBy: [
        { Type: 'TAG', Key: 'Service' },
        { Type: 'TAG', Key: 'CostCenter' }
      ]
    };

    const result = await this.ce.getCostAndUsage(params).promise();

    return {
      businessUnit,
      month,
      totalCost: parseFloat(result.ResultsByTime[0]?.Total.UnblendedCost.Amount || '0'),
      breakdown: result.ResultsByTime[0]?.Groups.map(group => ({
        service: group.Keys[0],
        costCenter: group.Keys[1],
        cost: parseFloat(group.Metrics.UnblendedCost.Amount)
      })) || []
    };
  }

  async getCostForecast(
    service: string,
    days: number = 30
  ): Promise<ForecastReport> {
    const startDate = new Date().toISOString().split('T')[0];
    const endDate = new Date(Date.now() + days * 24 * 60 * 60 * 1000)
      .toISOString()
      .split('T')[0];

    const params: CostExplorer.GetCostForecastRequest = {
      TimePeriod: { Start: startDate, End: endDate },
      Metric: 'UNBLENDED_COST',
      Granularity: 'DAILY',
      Filter: {
        Tags: {
          Key: 'Service',
          Values: [service]
        }
      }
    };

    const result = await this.ce.getCostForecast(params).promise();

    return {
      service,
      forecastPeriod: { start: startDate, end: endDate },
      predictedCost: parseFloat(result.Total.Amount),
      confidence: 'HIGH' // AWS provides prediction interval
    };
  }

  async getRightsizingRecommendations(): Promise<RightsizingRecommendation[]> {
    const params: CostExplorer.GetRightsizingRecommendationRequest = {
      Service: 'AmazonEC2',
      Configuration: {
        RecommendationTarget: 'SAME_INSTANCE_FAMILY',
        BenefitsConsidered: true
      }
    };

    const result = await this.ce.getRightsizingRecommendation(params).promise();

    return result.RightsizingRecommendations.map(rec => ({
      currentInstance: rec.CurrentInstance.ResourceId,
      currentType: rec.CurrentInstance.InstanceName,
      recommendedType: rec.ModifyRecommendationDetail.TargetInstances[0].InstanceType,
      monthlySavings: parseFloat(
        rec.ModifyRecommendationDetail.TargetInstances[0].EstimatedMonthlySavings
      )
    }));
  }

  private transformCostData(result: CostExplorer.GetCostAndUsageResponse): CostReport {
    return {
      period: {
        start: result.ResultsByTime[0].TimePeriod.Start,
        end: result.ResultsByTime[result.ResultsByTime.length - 1].TimePeriod.End
      },
      dailyCosts: result.ResultsByTime.map(day => ({
        date: day.TimePeriod.Start,
        cost: parseFloat(day.Total.UnblendedCost.Amount),
        groups: day.Groups.map(g => ({
          service: g.Keys[0],
          environment: g.Keys[1],
          cost: parseFloat(g.Metrics.UnblendedCost.Amount)
        }))
      }))
    };
  }

  private getLastDayOfMonth(month: string): string {
    const [year, mon] = month.split('-').map(Number);
    const lastDay = new Date(year, mon, 0).getDate();
    return `${month}-${lastDay.toString().padStart(2, '0')}`;
  }
}

interface CostReport {
  period: { start: string; end: string };
  dailyCosts: Array<{
    date: string;
    cost: number;
    groups: Array<{ service: string; environment: string; cost: number }>;
  }>;
}

interface BusinessUnitCostReport {
  businessUnit: string;
  month: string;
  totalCost: number;
  breakdown: Array<{
    service: string;
    costCenter: string;
    cost: number;
  }>;
}

interface ForecastReport {
  service: string;
  forecastPeriod: { start: string; end: string };
  predictedCost: number;
  confidence: string;
}

interface RightsizingRecommendation {
  currentInstance: string;
  currentType: string;
  recommendedType: string;
  monthlySavings: number;
}
```

### 📈 Cost Dashboard & Reporting

```typescript
// services/cost-reporting/dashboard-generator.ts
import { CostExplorerService } from './cost-explorer-service';
import * as cloudwatch from 'aws-sdk/clients/cloudwatch';

export class CostDashboardGenerator {
  private costExplorer: CostExplorerService;
  private cloudwatch: cloudwatch;

  constructor() {
    this.costExplorer = new CostExplorerService();
    this.cloudwatch = new cloudwatch();
  }

  async generateMonthlyShowbackReport(month: string): Promise<ShowbackReport> {
    const businessUnits = ['retail', 'corporate', 'investment', 'shared'];
    
    const reports = await Promise.all(
      businessUnits.map(bu => 
        this.costExplorer.getCostByBusinessUnit(bu, month)
      )
    );

    const totalCost = reports.reduce((sum, r) => sum + r.totalCost, 0);

    return {
      month,
      totalCost,
      businessUnits: reports.map(r => ({
        name: r.businessUnit,
        cost: r.totalCost,
        percentage: (r.totalCost / totalCost * 100).toFixed(2) + '%',
        services: r.breakdown
      }))
    };
  }

  async publishCostMetrics(service: string, environment: string, cost: number) {
    await this.cloudwatch.putMetricData({
      Namespace: 'Banking/Costs',
      MetricData: [
        {
          MetricName: 'DailyCost',
          Value: cost,
          Unit: 'None',
          Timestamp: new Date(),
          Dimensions: [
            { Name: 'Service', Value: service },
            { Name: 'Environment', Value: environment }
          ]
        }
      ]
    }).promise();
  }

  async createCostAllocationReport(
    startDate: string,
    endDate: string
  ): Promise<AllocationReport> {
    const services = [
      'account-service',
      'transaction-service',
      'payment-service',
      'loan-service'
    ];

    const costsByService = await Promise.all(
      services.map(async service => {
        const costs = await this.costExplorer.getServiceCosts(
          service,
          startDate,
          endDate
        );

        const totalCost = costs.dailyCosts.reduce((sum, day) => sum + day.cost, 0);

        return {
          service,
          totalCost,
          dailyAverage: totalCost / costs.dailyCosts.length,
          environments: this.aggregateByEnvironment(costs.dailyCosts)
        };
      })
    );

    const grandTotal = costsByService.reduce((sum, s) => sum + s.totalCost, 0);

    return {
      period: { start: startDate, end: endDate },
      grandTotal,
      services: costsByService,
      recommendations: await this.generateCostOptimizationRecommendations(costsByService)
    };
  }

  private aggregateByEnvironment(
    dailyCosts: Array<{ date: string; cost: number; groups: any[] }>
  ): Record<string, number> {
    const envCosts: Record<string, number> = {};

    dailyCosts.forEach(day => {
      day.groups.forEach(group => {
        envCosts[group.environment] = (envCosts[group.environment] || 0) + group.cost;
      });
    });

    return envCosts;
  }

  private async generateCostOptimizationRecommendations(
    services: Array<{ service: string; totalCost: number }>
  ): Promise<string[]> {
    const recommendations: string[] = [];

    // Check for services with high costs
    services.forEach(service => {
      if (service.totalCost > 5000) {
        recommendations.push(
          `${service.service}: Consider reserved instances or savings plans (current: $${service.totalCost}/month)`
        );
      }
    });

    // Get EC2 rightsizing recommendations
    const rightsizing = await this.costExplorer.getRightsizingRecommendations();
    rightsizing.forEach(rec => {
      recommendations.push(
        `Downsize ${rec.currentInstance} from ${rec.currentType} to ${rec.recommendedType} (save $${rec.monthlySavings}/month)`
      );
    });

    return recommendations;
  }
}

interface ShowbackReport {
  month: string;
  totalCost: number;
  businessUnits: Array<{
    name: string;
    cost: number;
    percentage: string;
    services: any[];
  }>;
}

interface AllocationReport {
  period: { start: string; end: string };
  grandTotal: number;
  services: Array<{
    service: string;
    totalCost: number;
    dailyAverage: number;
    environments: Record<string, number>;
  }>;
  recommendations: string[];
}
```

### 🎓 Interview Discussion Points - Q34

**Q1: Why is cost allocation tagging important?**

**A**:
- **Accountability**: Track costs by team/project
- **Chargeback/Showback**: Bill internal customers
- **Optimization**: Identify high-cost services
- **Budgeting**: Forecast costs accurately
- **Compliance**: Meet financial reporting requirements

**Q2: What are AWS best practices for tagging?**

**A**:
- **Mandatory tags**: Environment, Owner, CostCenter, Project
- **Standardize**: Use consistent naming (PascalCase recommended)
- **Automate**: Use AWS Tag Editor, Config rules
- **Governance**: Enforce with IAM policies requiring tags
- **Lifecycle**: Tag at creation, not retroactively

**Q3: How do you prevent untagged resources?**

**A**:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Action": "ec2:RunInstances",
    "Resource": "*",
    "Condition": {
      "StringNotLike": {
        "aws:RequestTag/Environment": "*",
        "aws:RequestTag/Owner": "*"
      }
    }
  }]
}
```

**Q4: What's the difference between budgets and cost anomaly detection?**

**A**:
- **Budgets**: Set thresholds, get alerts when exceeded
- **Anomaly Detection**: ML-based detection of unusual spending patterns
- **Use Both**: Budgets for planned limits, anomaly detection for unexpected spikes

**Q5: How to handle shared costs (like NAT Gateway)?**

**A**:
- **Tag shared resources**: BusinessUnit=shared
- **Allocate proportionally**: Split by usage percentage
- **Cost allocation tags**: Use AWS Cost Allocation Tags
- **Showback reports**: Transparently show shared cost distribution

---

