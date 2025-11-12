# Chaos Engineering

## Question 53: Chaos Engineering with AWS FIS

### 📋 Question Statement

Implement chaos engineering for Emirates NBD using AWS Fault Injection Simulator to test system resilience and failure scenarios.

---

### 💥 Chaos Engineering Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                   AWS FAULT INJECTION SIMULATOR                             │
└────────────────────────────────────────────────────────────────────────────┘

    ┌──────────────┐         ┌─────────────────┐
    │   Operator   │────────>│   AWS FIS       │
    │   Console    │         │  Experiment     │
    └──────────────┘         └────────┬────────┘
                                      │
                        ┌─────────────┴─────────────┐
                        │                           │
                        v                           v
                 ┌──────────┐              ┌───────────┐
                 │  Target  │              │CloudWatch │
                 │Resources │              │ Alarms    │
                 │(EC2/ECS) │              │(Stop Cond)│
                 └──────────┘              └───────────┘
                        │
                        v
                 ┌──────────┐
                 │ Monitoring│
                 │  Metrics │
                 └──────────┘
```

### 📦 AWS FIS CDK Infrastructure

```typescript
// infrastructure/cdk/fis-chaos-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as fis from 'aws-cdk-lib/aws-fis';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as cloudwatch from 'aws-cdk-lib/aws-cloudwatch';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as subscriptions from 'aws-cdk-lib/aws-sns-subscriptions';

export class FISChaosStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // SNS Topic for chaos experiment notifications
    const chaosTopic = new sns.Topic(this, 'ChaosTopic', {
      displayName: 'Chaos Engineering Notifications'
    });

    chaosTopic.addSubscription(
      new subscriptions.EmailSubscription('platform-team@emiratesnbd.com')
    );

    // CloudWatch Alarms (Stop Conditions)
    const errorRateAlarm = new cloudwatch.Alarm(this, 'HighErrorRate', {
      alarmName: 'banking-high-error-rate',
      metric: new cloudwatch.Metric({
        namespace: 'AWS/ApplicationELB',
        metricName: 'HTTPCode_Target_5XX_Count',
        statistic: 'Sum',
        period: cdk.Duration.minutes(1)
      }),
      threshold: 100,
      evaluationPeriods: 2,
      comparisonOperator: cloudwatch.ComparisonOperator.GREATER_THAN_THRESHOLD
    });

    const latencyAlarm = new cloudwatch.Alarm(this, 'HighLatency', {
      alarmName: 'banking-high-latency',
      metric: new cloudwatch.Metric({
        namespace: 'AWS/ApplicationELB',
        metricName: 'TargetResponseTime',
        statistic: 'Average',
        period: cdk.Duration.minutes(1)
      }),
      threshold: 3, // 3 seconds
      evaluationPeriods: 2,
      comparisonOperator: cloudwatch.ComparisonOperator.GREATER_THAN_THRESHOLD
    });

    // IAM Role for FIS
    const fisRole = new iam.Role(this, 'FISRole', {
      assumedBy: new iam.ServicePrincipal('fis.amazonaws.com'),
      description: 'Role for AWS Fault Injection Simulator'
    });

    fisRole.addToPolicy(
      new iam.PolicyStatement({
        actions: [
          'ec2:*',
          'ecs:*',
          'rds:*',
          'elasticloadbalancing:*',
          'cloudwatch:DescribeAlarms',
          's3:GetObject',
          'sns:Publish'
        ],
        resources: ['*']
      })
    );

    // Experiment 1: EC2 Instance Termination
    const ec2TerminationExperiment = new fis.CfnExperimentTemplate(this, 'EC2Termination', {
      description: 'Terminate random EC2 instances to test auto-scaling and resilience',
      roleArn: fisRole.roleArn,
      stopConditions: [
        {
          source: 'aws:cloudwatch:alarm',
          value: errorRateAlarm.alarmArn
        }
      ],
      actions: {
        terminateInstances: {
          actionId: 'aws:ec2:terminate-instances',
          description: 'Terminate 30% of instances',
          parameters: {},
          targets: {
            Instances: 'bankingInstances'
          }
        }
      },
      targets: {
        bankingInstances: {
          resourceType: 'aws:ec2:instance',
          selectionMode: 'PERCENT(30)',
          resourceTags: {
            Environment: 'staging',
            Service: 'banking-api'
          }
        }
      },
      tags: {
        Name: 'ec2-termination-chaos',
        Type: 'resilience-test'
      }
    });

    // Experiment 2: ECS Task Termination
    const ecsTaskTerminationExperiment = new fis.CfnExperimentTemplate(this, 'ECSTaskTermination', {
      description: 'Stop ECS tasks to test service recovery',
      roleArn: fisRole.roleArn,
      stopConditions: [
        {
          source: 'aws:cloudwatch:alarm',
          value: errorRateAlarm.alarmArn
        }
      ],
      actions: {
        stopTasks: {
          actionId: 'aws:ecs:stop-task',
          description: 'Stop 50% of account-service tasks',
          parameters: {},
          targets: {
            Tasks: 'accountServiceTasks'
          }
        }
      },
      targets: {
        accountServiceTasks: {
          resourceType: 'aws:ecs:task',
          selectionMode: 'PERCENT(50)',
          resourceTags: {
            Service: 'account-service',
            Environment: 'staging'
          },
          filters: [
            {
              path: 'State.Name',
              values: ['running']
            }
          ]
        }
      },
      tags: {
        Name: 'ecs-task-termination-chaos'
      }
    });

    // Experiment 3: Network Throttling (requires SSM agent)
    const networkThrottlingExperiment = new fis.CfnExperimentTemplate(this, 'NetworkThrottling', {
      description: 'Add network latency to test timeout handling',
      roleArn: fisRole.roleArn,
      stopConditions: [
        {
          source: 'aws:cloudwatch:alarm',
          value: latencyAlarm.alarmArn
        }
      ],
      actions: {
        addLatency: {
          actionId: 'aws:ssm:send-command',
          description: 'Add 500ms network latency',
          parameters: {
            documentArn: 'arn:aws:ssm:us-east-1::document/AWSFIS-Run-Network-Latency',
            documentParameters: JSON.stringify({
              DurationSeconds: '300',
              Interface: 'eth0',
              DelayMilliseconds: '500',
              JitterMilliseconds: '100'
            }),
            duration: 'PT5M'
          },
          targets: {
            Instances: 'paymentServiceInstances'
          }
        }
      },
      targets: {
        paymentServiceInstances: {
          resourceType: 'aws:ec2:instance',
          selectionMode: 'COUNT(2)',
          resourceTags: {
            Service: 'payment-service',
            Environment: 'staging'
          }
        }
      },
      tags: {
        Name: 'network-latency-chaos'
      }
    });

    // Experiment 4: RDS Failover
    const rdsFailoverExperiment = new fis.CfnExperimentTemplate(this, 'RDSFailover', {
      description: 'Force RDS failover to test application resilience',
      roleArn: fisRole.roleArn,
      stopConditions: [
        {
          source: 'aws:cloudwatch:alarm',
          value: errorRateAlarm.alarmArn
        }
      ],
      actions: {
        rebootRDS: {
          actionId: 'aws:rds:reboot-db-instances',
          description: 'Reboot RDS instance with failover',
          parameters: {
            forceFailover: 'true'
          },
          targets: {
            DBInstances: 'bankingDatabase'
          }
        }
      },
      targets: {
        bankingDatabase: {
          resourceType: 'aws:rds:db',
          selectionMode: 'ALL',
          resourceTags: {
            Name: 'banking-db-staging',
            Environment: 'staging'
          }
        }
      },
      tags: {
        Name: 'rds-failover-chaos'
      }
    });

    // Experiment 5: AZ Failure Simulation
    const azFailureExperiment = new fis.CfnExperimentTemplate(this, 'AZFailure', {
      description: 'Simulate AZ failure by stopping all instances in one AZ',
      roleArn: fisRole.roleArn,
      stopConditions: [
        {
          source: 'aws:cloudwatch:alarm',
          value: errorRateAlarm.alarmArn
        }
      ],
      actions: {
        stopInstancesInAZ: {
          actionId: 'aws:ec2:stop-instances',
          description: 'Stop all instances in us-east-1a',
          parameters: {
            startInstancesAfterDuration: 'PT10M'
          },
          targets: {
            Instances: 'azInstances'
          }
        }
      },
      targets: {
        azInstances: {
          resourceType: 'aws:ec2:instance',
          selectionMode: 'ALL',
          resourceTags: {
            Environment: 'staging'
          },
          filters: [
            {
              path: 'Placement.AvailabilityZone',
              values: ['us-east-1a']
            },
            {
              path: 'State.Name',
              values: ['running']
            }
          ]
        }
      },
      tags: {
        Name: 'az-failure-chaos'
      }
    });

    // Outputs
    new cdk.CfnOutput(this, 'EC2ExperimentId', {
      value: ec2TerminationExperiment.ref,
      description: 'EC2 Termination Experiment Template ID'
    });

    new cdk.CfnOutput(this, 'ECSExperimentId', {
      value: ecsTaskTerminationExperiment.ref,
      description: 'ECS Task Termination Experiment Template ID'
    });

    new cdk.CfnOutput(this, 'NetworkExperimentId', {
      value: networkThrottlingExperiment.ref,
      description: 'Network Throttling Experiment Template ID'
    });
  }
}
```

### 🧪 Chaos Experiment Runner

```typescript
// scripts/chaos/run-experiment.ts
import { FISClient, StartExperimentCommand, GetExperimentCommand } from '@aws-sdk/client-fis';
import { CloudWatchClient, GetMetricStatisticsCommand } from '@aws-sdk/client-cloudwatch';

const fis = new FISClient({});
const cloudwatch = new CloudWatchClient({});

interface ExperimentConfig {
  templateId: string;
  name: string;
  tags?: Record<string, string>;
}

async function runChaosExperiment(config: ExperimentConfig): Promise<void> {
  console.log(`Starting chaos experiment: ${config.name}`);

  // Collect baseline metrics
  const baselineMetrics = await collectMetrics();
  console.log('Baseline metrics:', baselineMetrics);

  try {
    // Start experiment
    const startResponse = await fis.send(
      new StartExperimentCommand({
        experimentTemplateId: config.templateId,
        tags: {
          ExecutedBy: 'automated-chaos-runner',
          ExecutedAt: new Date().toISOString(),
          ...config.tags
        }
      })
    );

    const experimentId = startResponse.experiment?.id!;
    console.log(`Experiment started: ${experimentId}`);

    // Monitor experiment
    await monitorExperiment(experimentId);

    // Collect post-experiment metrics
    const postMetrics = await collectMetrics();
    console.log('Post-experiment metrics:', postMetrics);

    // Generate report
    generateReport(baselineMetrics, postMetrics);

  } catch (error) {
    console.error('Chaos experiment failed:', error);
    throw error;
  }
}

async function monitorExperiment(experimentId: string): Promise<void> {
  let status = 'running';

  while (status === 'running' || status === 'initiating') {
    await new Promise(resolve => setTimeout(resolve, 10000)); // Poll every 10 seconds

    const response = await fis.send(
      new GetExperimentCommand({ id: experimentId })
    );

    status = response.experiment?.state?.status || 'unknown';
    console.log(`Experiment status: ${status}`);

    if (status === 'stopped') {
      const reason = response.experiment?.state?.reason;
      console.log(`Experiment stopped: ${reason}`);
      break;
    }

    if (status === 'failed') {
      throw new Error('Experiment failed');
    }
  }

  console.log('Experiment completed');
}

async function collectMetrics(): Promise<MetricsSnapshot> {
  const endTime = new Date();
  const startTime = new Date(endTime.getTime() - 5 * 60 * 1000); // Last 5 minutes

  const [errorRate, latency, throughput] = await Promise.all([
    getMetric('AWS/ApplicationELB', 'HTTPCode_Target_5XX_Count', startTime, endTime),
    getMetric('AWS/ApplicationELB', 'TargetResponseTime', startTime, endTime),
    getMetric('AWS/ApplicationELB', 'RequestCount', startTime, endTime)
  ]);

  return { errorRate, latency, throughput };
}

async function getMetric(
  namespace: string,
  metricName: string,
  startTime: Date,
  endTime: Date
): Promise<number> {
  const response = await cloudwatch.send(
    new GetMetricStatisticsCommand({
      Namespace: namespace,
      MetricName: metricName,
      StartTime: startTime,
      EndTime: endTime,
      Period: 60,
      Statistics: ['Average']
    })
  );

  const datapoints = response.Datapoints || [];
  const average = datapoints.reduce((sum, dp) => sum + (dp.Average || 0), 0) / datapoints.length;
  return average || 0;
}

function generateReport(baseline: MetricsSnapshot, post: MetricsSnapshot): void {
  const report = {
    baseline,
    post,
    changes: {
      errorRateChange: ((post.errorRate - baseline.errorRate) / baseline.errorRate * 100).toFixed(2) + '%',
      latencyChange: ((post.latency - baseline.latency) / baseline.latency * 100).toFixed(2) + '%',
      throughputChange: ((post.throughput - baseline.throughput) / baseline.throughput * 100).toFixed(2) + '%'
    },
    verdict: post.errorRate < baseline.errorRate * 2 ? 'PASSED' : 'FAILED'
  };

  console.log('\n=== Chaos Experiment Report ===');
  console.log(JSON.stringify(report, null, 2));
}

interface MetricsSnapshot {
  errorRate: number;
  latency: number;
  throughput: number;
}

// Run experiments
const experiments: ExperimentConfig[] = [
  { templateId: 'EXT123abc', name: 'EC2 Instance Termination' },
  { templateId: 'EXT456def', name: 'ECS Task Termination' },
  { templateId: 'EXT789ghi', name: 'Network Latency Injection' }
];

(async () => {
  for (const experiment of experiments) {
    await runChaosExperiment(experiment);
    await new Promise(resolve => setTimeout(resolve, 60000)); // Wait 1 minute between experiments
  }
})();
```

### 🎓 Interview Discussion Points - Q53

**Q1: What is chaos engineering?**

**A**:
- **Definition**: Deliberately injecting failures to test system resilience
- **Goal**: Identify weaknesses before they cause production outages
- **Principles**:
  - Build hypothesis about steady state
  - Inject real-world failures
  - Measure system behavior
  - Minimize blast radius
  - Automate experiments

**Q2: What failures should you test?**

**A**:
- **Infrastructure**: EC2 termination, AZ failure, network partitioning
- **Services**: Service crashes, high latency, dependency failures
- **Resources**: CPU spike, memory exhaustion, disk full
- **Network**: Packet loss, latency injection, bandwidth throttling
- **Data**: Database failover, cache invalidation

**Q3: How to run chaos experiments safely?**

**A**:
- **Start small**: Test in staging first
- **Stop conditions**: CloudWatch alarms to halt experiment
- **Blast radius**: Limit to subset of resources (30%)
- **Business hours**: Run during working hours with team ready
- **Rollback plan**: Automated recovery procedures
- **Monitoring**: Real-time dashboards during experiment

**Q4: What are AWS FIS capabilities?**

**A**:
- **Actions**: EC2 stop/terminate, ECS stop tasks, RDS failover, network latency
- **Targets**: Select resources by tags, filters, percentage
- **Stop conditions**: CloudWatch alarms halt experiment
- **SSM integration**: Network fault injection via SSM documents
- **IAM**: Fine-grained permissions for experiments

**Q5: How to measure chaos experiment success?**

**A**:
- **SLO compliance**: Did SLOs remain within bounds?
- **Recovery time**: How long to return to steady state?
- **Customer impact**: Were customers affected?
- **Alerts**: Did monitoring detect the issue?
- **Team response**: Was on-call notified appropriately?

---

## Question 54: Fault Injection Testing & Resilience Validation

### 📋 Question Statement

Implement comprehensive fault injection testing for Emirates NBD including network latency injection, pod termination, AZ failure simulation, and automated resilience validation with rollback capabilities.

---

### 💥 AWS FIS Experiment Templates

```typescript
// infrastructure/cdk/fis-experiments-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as fis from 'aws-cdk-lib/aws-fis';
import * as iam from 'aws-cdk-lib/aws-iam';

export class FISExperimentsStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // IAM role for FIS
    const fisRole = new iam.Role(this, 'FISRole', {
      assumedBy: new iam.ServicePrincipal('fis.amazonaws.com'),
      managedPolicies: [
        iam.ManagedPolicy.fromAwsManagedPolicyName('PowerUserAccess')
      ]
    });

    // Experiment 1: Terminate random ECS tasks
    const terminateECSTask = new fis.CfnExperimentTemplate(this, 'TerminateECSTasks', {
      description: 'Terminate 50% of ECS tasks in account-service',
      roleArn: fisRole.roleArn,
      stopConditions: [
        {
          source: 'aws:cloudwatch:alarm',
          value: 'arn:aws:cloudwatch:us-east-1:123456789012:alarm:high-error-rate'
        }
      ],
      actions: {
        TerminateTasks: {
          actionId: 'aws:ecs:stop-task',
          parameters: {
            taskDefinitionArn: props.accountServiceTaskArn
          },
          targets: {
            Tasks: 'account-service-tasks'
          }
        }
      },
      targets: {
        'account-service-tasks': {
          resourceType: 'aws:ecs:task',
          resourceTags: {
            Service: 'account-service',
            Environment: 'staging'
          },
          selectionMode: 'PERCENT(50)'
        }
      },
      tags: {
        Name: 'ecs-task-termination-test',
        Team: 'platform'
      }
    });

    // Experiment 2: Inject network latency
    const injectLatency = new fis.CfnExperimentTemplate(this, 'InjectNetworkLatency', {
      description: 'Add 500ms latency to payment-service',
      roleArn: fisRole.roleArn,
      stopConditions: [
        {
          source: 'none'
        }
      ],
      actions: {
        InjectLatency: {
          actionId: 'aws:ec2:network-latency',
          parameters: {
            duration: 'PT10M', // 10 minutes
            latencyMilliseconds: '500'
          },
          targets: {
            Instances: 'payment-service-instances'
          }
        }
      },
      targets: {
        'payment-service-instances': {
          resourceType: 'aws:ec2:instance',
          resourceTags: {
            Service: 'payment-service'
          },
          selectionMode: 'ALL'
        }
      }
    });

    // Experiment 3: RDS failover
    const rdsFailover = new fis.CfnExperimentTemplate(this, 'RDSFailover', {
      description: 'Trigger RDS failover for banking database',
      roleArn: fisRole.roleArn,
      stopConditions: [
        {
          source: 'aws:cloudwatch:alarm',
          value: props.databaseConnectionAlarm
        }
      ],
      actions: {
        FailoverRDS: {
          actionId: 'aws:rds:reboot-db-instances',
          parameters: {
            forceFailover: 'true'
          },
          targets: {
            DBInstances: 'banking-database'
          }
        }
      },
      targets: {
        'banking-database': {
          resourceType: 'aws:rds:db',
          resourceArns: [props.databaseArn],
          selectionMode: 'ALL'
        }
      }
    });
  }
}
```

### 🧪 Chaos Testing Orchestrator

```typescript
// services/chaos/chaos-orchestrator.ts
import { FIS, CloudWatch } from 'aws-sdk';

export class ChaosOrchestrator {
  private fis: FIS;
  private cloudwatch: CloudWatch;

  constructor() {
    this.fis = new FIS();
    this.cloudwatch = new CloudWatch();
  }

  async runExperiment(
    experimentTemplateId: string,
    validationChecks: ValidationCheck[]
  ): Promise<ExperimentResult> {
    console.log(`Starting chaos experiment: ${experimentTemplateId}`);

    // Capture baseline metrics
    const baselineMetrics = await this.captureMetrics();

    // Start experiment
    const experiment = await this.fis.startExperiment({
      experimentTemplateId,
      tags: {
        RunBy: 'automated-chaos-testing',
        Timestamp: new Date().toISOString()
      }
    }).promise();

    const experimentId = experiment.experiment!.id!;

    // Monitor experiment
    await this.monitorExperiment(experimentId);

    // Wait for completion
    await this.waitForCompletion(experimentId);

    // Capture post-experiment metrics
    const postMetrics = await this.captureMetrics();

    // Run validation checks
    const validationResults = await this.runValidationChecks(
      validationChecks,
      baselineMetrics,
      postMetrics
    );

    // Analyze results
    const passed = validationResults.every(r => r.passed);

    return {
      experimentId,
      passed,
      baselineMetrics,
      postMetrics,
      validationResults
    };
  }

  async runPodTerminationTest(
    namespace: string,
    deploymentName: string,
    terminationPercentage: number
  ): Promise<void> {
    console.log(`Terminating ${terminationPercentage}% of pods in ${deploymentName}`);

    // Use kubectl via exec (or k8s API)
    const pods = await this.getPods(namespace, deploymentName);
    const podsToTerminate = Math.ceil(pods.length * terminationPercentage / 100);

    for (let i = 0; i < podsToTerminate; i++) {
      const pod = pods[i];
      console.log(`Terminating pod: ${pod}`);
      await this.terminatePod(namespace, pod);
      await this.sleep(2000); // 2 second delay between terminations
    }
  }

  async injectNetworkLatency(
    targetService: string,
    latencyMs: number,
    durationMinutes: number
  ): Promise<void> {
    console.log(`Injecting ${latencyMs}ms latency to ${targetService} for ${durationMinutes} minutes`);

    // Use Istio VirtualService for latency injection
    // Or use tc (traffic control) on EC2 instances
    // This is a simplified example
    await this.updateIstioFaultInjection(targetService, {
      delay: {
        fixedDelay: `${latencyMs}ms`,
        percentage: { value: 100 }
      }
    });

    // Wait for duration
    await this.sleep(durationMinutes * 60 * 1000);

    // Remove fault injection
    await this.removeIstioFaultInjection(targetService);
  }

  async simulateAZFailure(availabilityZone: string): Promise<void> {
    console.log(`Simulating failure in AZ: ${availabilityZone}`);

    // This would:
    // 1. Block traffic to AZ using security groups
    // 2. Terminate instances in AZ
    // 3. Validate traffic shifts to other AZs
    // 4. Check auto-scaling responds correctly
  }

  private async monitorExperiment(experimentId: string): Promise<void> {
    // Poll experiment status and metrics
    const interval = setInterval(async () => {
      const experiment = await this.fis.getExperiment({
        id: experimentId
      }).promise();

      console.log(`Experiment state: ${experiment.experiment!.state!.status}`);

      if (experiment.experiment!.state!.status === 'stopped') {
        clearInterval(interval);
      }
    }, 5000);
  }

  private async waitForCompletion(experimentId: string): Promise<void> {
    while (true) {
      const experiment = await this.fis.getExperiment({
        id: experimentId
      }).promise();

      const status = experiment.experiment!.state!.status;

      if (status === 'completed' || status === 'stopped' || status === 'failed') {
        return;
      }

      await this.sleep(5000);
    }
  }

  private async captureMetrics(): Promise<SystemMetrics> {
    // Capture current system metrics from CloudWatch
    return {
      errorRate: await this.getMetric('ErrorRate'),
      latencyP99: await this.getMetric('Latency', 'p99'),
      throughput: await this.getMetric('RequestCount')
    };
  }

  private async getMetric(metricName: string, stat: string = 'Average'): Promise<number> {
    const result = await this.cloudwatch.getMetricStatistics({
      Namespace: 'Banking/Application',
      MetricName: metricName,
      StartTime: new Date(Date.now() - 5 * 60 * 1000),
      EndTime: new Date(),
      Period: 300,
      Statistics: [stat]
    }).promise();

    return result.Datapoints[0]?.[stat] || 0;
  }

  private async runValidationChecks(
    checks: ValidationCheck[],
    baseline: SystemMetrics,
    post: SystemMetrics
  ): Promise<ValidationResult[]> {
    return checks.map(check => ({
      name: check.name,
      passed: check.validator(baseline, post),
      message: check.message
    }));
  }

  private async getPods(namespace: string, deployment: string): Promise<string[]> {
    // K8s API call to get pods
    return ['pod-1', 'pod-2', 'pod-3'];
  }

  private async terminatePod(namespace: string, podName: string): Promise<void> {
    // kubectl delete pod
  }

  private async updateIstioFaultInjection(service: string, fault: any): Promise<void> {
    // Update Istio VirtualService
  }

  private async removeIstioFaultInjection(service: string): Promise<void> {
    // Remove fault injection from VirtualService
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

interface ValidationCheck {
  name: string;
  validator: (baseline: SystemMetrics, post: SystemMetrics) => boolean;
  message: string;
}

interface SystemMetrics {
  errorRate: number;
  latencyP99: number;
  throughput: number;
}

interface ValidationResult {
  name: string;
  passed: boolean;
  message: string;
}

interface ExperimentResult {
  experimentId: string;
  passed: boolean;
  baselineMetrics: SystemMetrics;
  postMetrics: SystemMetrics;
  validationResults: ValidationResult[];
}
```

### 🎓 Interview Discussion Points - Q54

**Q1: What is chaos engineering?**

**A**:
- **Definition**: Intentionally inject failures to test resilience
- **Goal**: Find weaknesses before they cause outages
- **Examples**: Pod termination, network latency, AZ failure
- **Principle**: "Break things on purpose in controlled way"

**Q2: What are common chaos experiments?**

**A**:
- **Pod/instance termination**: Test auto-scaling
- **Network latency**: Validate timeouts
- **AZ failure**: Test multi-AZ resilience
- **RDS failover**: Test database redundancy
- **Resource exhaustion**: CPU/memory limits

**Q3: How to safely run chaos experiments?**

**A**:
- **Start in staging**: Never production first
- **Business hours**: When team is available
- **Stop conditions**: CloudWatch alarms trigger abort
- **Gradual**: 1 pod → 10% → 50%
- **Observability**: Monitor everything

**Q4: What is a stop condition?**

**A**:
```typescript
stopConditions: [{
  source: 'aws:cloudwatch:alarm',
  value: 'high-error-rate-alarm'
}]
```
**Purpose**: Auto-abort experiment if system degraded

**Q5: How to measure resilience?**

**A**:
- **Recovery time**: How fast system recovers
- **Error rate**: Did errors spike?
- **User impact**: Were customers affected?
- **Auto-scaling**: Did it respond correctly?
- **Target**: Zero customer impact

---

**End of File 27**

