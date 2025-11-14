# AWS Interview Questions: Container & Deployment Services (Q31-Q33)

## Question 31: Explain Amazon ECS (Elastic Container Service) vs EKS (Elastic Kubernetes Service)

### 📋 Answer

**Amazon ECS** and **Amazon EKS** are container orchestration services that allow you to run containerized applications at scale.

### ECS vs EKS Comparison:

| Feature | ECS | EKS |
|---------|-----|-----|
| **Orchestration** | AWS proprietary | Kubernetes (open-source) |
| **Learning Curve** | Easier | Steeper (K8s knowledge) |
| **Cost** | No control plane cost | $0.10/hour per cluster |
| **Launch Types** | EC2, Fargate | EC2, Fargate |
| **Portability** | AWS-specific | Portable (any K8s) |
| **AWS Integration** | Deep integration | Good integration |
| **Community** | AWS-focused | Large K8s community |
| **Use Case** | AWS-native apps | Multi-cloud, K8s ecosystem |

### Architecture:

```
ECS/EKS Architecture
├── Cluster
├── Services
│   ├── Task Definitions (ECS) / Pods (EKS)
│   ├── Auto Scaling
│   └── Load Balancers
├── Compute
│   ├── EC2 Instances
│   └── AWS Fargate (serverless)
└── Networking
    ├── VPC
    ├── Security Groups
    └── Service Discovery
```

### Complete ECS Implementation:

```javascript
// ecs-service.js
import {
  ECSClient,
  CreateClusterCommand,
  RegisterTaskDefinitionCommand,
  CreateServiceCommand,
  UpdateServiceCommand,
  DescribeServicesCommand,
  ListTasksCommand,
  DescribeTasksCommand,
  RunTaskCommand,
  StopTaskCommand
} from '@aws-sdk/client-ecs';

const ecsClient = new ECSClient({ region: 'us-east-1' });

class ECSService {
  // 1. Create Cluster
  async createCluster(clusterName) {
    const command = new CreateClusterCommand({
      clusterName: clusterName,
      tags: [
        { key: 'Environment', value: 'Production' },
        { key: 'ManagedBy', value: 'Automation' }
      ],
      settings: [
        {
          name: 'containerInsights',
          value: 'enabled'
        }
      ]
    });
    
    try {
      const response = await ecsClient.send(command);
      console.log('Cluster created:', response.cluster.clusterName);
      return response.cluster;
    } catch (error) {
      console.error('Failed to create cluster:', error);
      throw error;
    }
  }
  
  // 2. Register Task Definition
  async registerTaskDefinition(taskDefinition) {
    const command = new RegisterTaskDefinitionCommand(taskDefinition);
    
    try {
      const response = await ecsClient.send(command);
      console.log('Task definition registered:', response.taskDefinition.taskDefinitionArn);
      return response.taskDefinition;
    } catch (error) {
      console.error('Failed to register task definition:', error);
      throw error;
    }
  }
  
  // 3. Create Service
  async createService(clusterName, serviceName, taskDefinition, desiredCount, options = {}) {
    const command = new CreateServiceCommand({
      cluster: clusterName,
      serviceName: serviceName,
      taskDefinition: taskDefinition,
      desiredCount: desiredCount,
      launchType: options.launchType || 'FARGATE',
      networkConfiguration: options.networkConfiguration,
      loadBalancers: options.loadBalancers || [],
      healthCheckGracePeriodSeconds: options.healthCheckGracePeriodSeconds || 60,
      deploymentConfiguration: {
        maximumPercent: 200,
        minimumHealthyPercent: 100,
        deploymentCircuitBreaker: {
          enable: true,
          rollback: true
        }
      },
      enableECSManagedTags: true,
      propagateTags: 'SERVICE'
    });
    
    const response = await ecsClient.send(command);
    console.log('Service created:', response.service.serviceName);
    return response.service;
  }
  
  // 4. Update Service (Deployment)
  async updateService(clusterName, serviceName, taskDefinition) {
    const command = new UpdateServiceCommand({
      cluster: clusterName,
      service: serviceName,
      taskDefinition: taskDefinition,
      forceNewDeployment: true
    });
    
    const response = await ecsClient.send(command);
    console.log('Service updated, new deployment started');
    return response.service;
  }
  
  // 5. Scale Service
  async scaleService(clusterName, serviceName, desiredCount) {
    const command = new UpdateServiceCommand({
      cluster: clusterName,
      service: serviceName,
      desiredCount: desiredCount
    });
    
    const response = await ecsClient.send(command);
    console.log(`Service scaled to ${desiredCount} tasks`);
    return response.service;
  }
  
  // 6. Run Task (one-time execution)
  async runTask(clusterName, taskDefinition, networkConfiguration) {
    const command = new RunTaskCommand({
      cluster: clusterName,
      taskDefinition: taskDefinition,
      launchType: 'FARGATE',
      networkConfiguration: networkConfiguration,
      count: 1
    });
    
    const response = await ecsClient.send(command);
    console.log('Task started:', response.tasks[0].taskArn);
    return response.tasks[0];
  }
  
  // 7. Stop Task
  async stopTask(clusterName, taskArn, reason) {
    const command = new StopTaskCommand({
      cluster: clusterName,
      task: taskArn,
      reason: reason
    });
    
    await ecsClient.send(command);
    console.log('Task stopped:', taskArn);
  }
  
  // 8. List Running Tasks
  async listTasks(clusterName, serviceName) {
    const command = new ListTasksCommand({
      cluster: clusterName,
      serviceName: serviceName,
      desiredStatus: 'RUNNING'
    });
    
    const response = await ecsClient.send(command);
    return response.taskArns;
  }
  
  // 9. Describe Tasks
  async describeTasks(clusterName, taskArns) {
    const command = new DescribeTasksCommand({
      cluster: clusterName,
      tasks: taskArns
    });
    
    const response = await ecsClient.send(command);
    
    return response.tasks.map(task => ({
      taskArn: task.taskArn,
      lastStatus: task.lastStatus,
      desiredStatus: task.desiredStatus,
      cpu: task.cpu,
      memory: task.memory,
      containers: task.containers.map(c => ({
        name: c.name,
        status: c.lastStatus,
        exitCode: c.exitCode
      }))
    }));
  }
}

// Task Definition Builder
class TaskDefinitionBuilder {
  constructor(family) {
    this.taskDef = {
      family: family,
      networkMode: 'awsvpc',
      requiresCompatibilities: ['FARGATE'],
      cpu: '256',
      memory: '512',
      containerDefinitions: [],
      tags: []
    };
  }
  
  setExecutionRole(roleArn) {
    this.taskDef.executionRoleArn = roleArn;
    return this;
  }
  
  setTaskRole(roleArn) {
    this.taskDef.taskRoleArn = roleArn;
    return this;
  }
  
  setCpuMemory(cpu, memory) {
    this.taskDef.cpu = cpu;
    this.taskDef.memory = memory;
    return this;
  }
  
  addContainer(containerConfig) {
    this.taskDef.containerDefinitions.push({
      name: containerConfig.name,
      image: containerConfig.image,
      cpu: containerConfig.cpu || 0,
      memory: containerConfig.memory,
      memoryReservation: containerConfig.memoryReservation,
      essential: containerConfig.essential !== false,
      portMappings: containerConfig.portMappings || [],
      environment: containerConfig.environment || [],
      secrets: containerConfig.secrets || [],
      logConfiguration: containerConfig.logConfiguration || {
        logDriver: 'awslogs',
        options: {
          'awslogs-group': `/ecs/${this.taskDef.family}`,
          'awslogs-region': 'us-east-1',
          'awslogs-stream-prefix': 'ecs'
        }
      },
      healthCheck: containerConfig.healthCheck,
      command: containerConfig.command,
      entryPoint: containerConfig.entryPoint
    });
    return this;
  }
  
  addTag(key, value) {
    this.taskDef.tags.push({ key, value });
    return this;
  }
  
  build() {
    return this.taskDef;
  }
}

// Usage - Web Application Deployment
const ecs = new ECSService();

// Create cluster
await ecs.createCluster('production-cluster');

// Build task definition
const taskDef = new TaskDefinitionBuilder('web-app')
  .setCpuMemory('512', '1024')
  .setExecutionRole('arn:aws:iam::123456789012:role/ecsTaskExecutionRole')
  .setTaskRole('arn:aws:iam::123456789012:role/ecsTaskRole')
  .addContainer({
    name: 'web',
    image: '123456789012.dkr.ecr.us-east-1.amazonaws.com/web-app:latest',
    memory: 512,
    portMappings: [
      {
        containerPort: 3000,
        protocol: 'tcp'
      }
    ],
    environment: [
      { name: 'NODE_ENV', value: 'production' },
      { name: 'PORT', value: '3000' }
    ],
    secrets: [
      {
        name: 'DATABASE_URL',
        valueFrom: 'arn:aws:secretsmanager:us-east-1:123456789012:secret:db-url'
      }
    ],
    healthCheck: {
      command: ['CMD-SHELL', 'curl -f http://localhost:3000/health || exit 1'],
      interval: 30,
      timeout: 5,
      retries: 3,
      startPeriod: 60
    }
  })
  .addTag('Application', 'WebApp')
  .addTag('Environment', 'Production')
  .build();

// Register task definition
const registeredTaskDef = await ecs.registerTaskDefinition(taskDef);

// Create service
await ecs.createService(
  'production-cluster',
  'web-app-service',
  registeredTaskDef.taskDefinitionArn,
  2,  // 2 tasks
  {
    launchType: 'FARGATE',
    networkConfiguration: {
      awsvpcConfiguration: {
        subnets: ['subnet-12345', 'subnet-67890'],
        securityGroups: ['sg-12345'],
        assignPublicIp: 'ENABLED'
      }
    },
    loadBalancers: [
      {
        targetGroupArn: 'arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/web-app/abc123',
        containerName: 'web',
        containerPort: 3000
      }
    ],
    healthCheckGracePeriodSeconds: 60
  }
);

// Scale service
await ecs.scaleService('production-cluster', 'web-app-service', 5);

// Run one-time task (database migration)
await ecs.runTask(
  'production-cluster',
  'web-app:1',
  {
    awsvpcConfiguration: {
      subnets: ['subnet-12345'],
      securityGroups: ['sg-12345'],
      assignPublicIp: 'DISABLED'
    }
  }
);
```

### Complete EKS Implementation:

```javascript
// eks-service.js
import {
  EKSClient,
  CreateClusterCommand,
  DescribeClusterCommand,
  CreateNodegroupCommand,
  DescribeNodegroupCommand,
  UpdateNodegroupConfigCommand
} from '@aws-sdk/client-eks';

const eksClient = new EKSClient({ region: 'us-east-1' });

class EKSService {
  // 1. Create EKS Cluster
  async createCluster(clusterName, roleArn, vpcConfig) {
    const command = new CreateClusterCommand({
      name: clusterName,
      version: '1.28',
      roleArn: roleArn,
      resourcesVpcConfig: vpcConfig,
      logging: {
        clusterLogging: [
          {
            types: ['api', 'audit', 'authenticator', 'controllerManager', 'scheduler'],
            enabled: true
          }
        ]
      },
      tags: {
        Environment: 'Production',
        ManagedBy: 'Terraform'
      }
    });
    
    try {
      const response = await eksClient.send(command);
      console.log('EKS cluster creation started:', response.cluster.name);
      return response.cluster;
    } catch (error) {
      console.error('Failed to create cluster:', error);
      throw error;
    }
  }
  
  // 2. Create Node Group
  async createNodeGroup(clusterName, nodeGroupName, nodeRoleArn, subnets, options = {}) {
    const command = new CreateNodegroupCommand({
      clusterName: clusterName,
      nodegroupName: nodeGroupName,
      scalingConfig: {
        minSize: options.minSize || 1,
        maxSize: options.maxSize || 3,
        desiredSize: options.desiredSize || 2
      },
      subnets: subnets,
      nodeRole: nodeRoleArn,
      instanceTypes: options.instanceTypes || ['t3.medium'],
      amiType: options.amiType || 'AL2_x86_64',
      diskSize: options.diskSize || 20,
      tags: {
        Environment: 'Production'
      }
    });
    
    const response = await eksClient.send(command);
    console.log('Node group creation started:', response.nodegroup.nodegroupName);
    return response.nodegroup;
  }
  
  // 3. Wait for Cluster
  async waitForCluster(clusterName, pollInterval = 30000) {
    while (true) {
      const command = new DescribeClusterCommand({ name: clusterName });
      const response = await eksClient.send(command);
      
      const status = response.cluster.status;
      console.log('Cluster status:', status);
      
      if (status === 'ACTIVE') {
        return response.cluster;
      } else if (status === 'FAILED') {
        throw new Error('Cluster creation failed');
      }
      
      await new Promise(resolve => setTimeout(resolve, pollInterval));
    }
  }
  
  // 4. Scale Node Group
  async scaleNodeGroup(clusterName, nodeGroupName, desiredSize) {
    const command = new UpdateNodegroupConfigCommand({
      clusterName: clusterName,
      nodegroupName: nodeGroupName,
      scalingConfig: {
        desiredSize: desiredSize
      }
    });
    
    await eksClient.send(command);
    console.log(`Node group scaled to ${desiredSize} nodes`);
  }
}

// Kubernetes Deployment with AWS SDK
import { KubernetesClient } from '@kubernetes/client-node';

class K8sDeploymentManager {
  constructor(clusterEndpoint, token) {
    this.k8sClient = new KubernetesClient();
    // Configure with EKS cluster credentials
  }
  
  // Create Deployment
  createDeployment(namespace, name, image, replicas, port) {
    return {
      apiVersion: 'apps/v1',
      kind: 'Deployment',
      metadata: {
        name: name,
        namespace: namespace,
        labels: {
          app: name
        }
      },
      spec: {
        replicas: replicas,
        selector: {
          matchLabels: {
            app: name
          }
        },
        template: {
          metadata: {
            labels: {
              app: name
            }
          },
          spec: {
            containers: [
              {
                name: name,
                image: image,
                ports: [
                  {
                    containerPort: port
                  }
                ],
                resources: {
                  requests: {
                    cpu: '100m',
                    memory: '128Mi'
                  },
                  limits: {
                    cpu: '500m',
                    memory: '512Mi'
                  }
                },
                livenessProbe: {
                  httpGet: {
                    path: '/health',
                    port: port
                  },
                  initialDelaySeconds: 30,
                  periodSeconds: 10
                },
                readinessProbe: {
                  httpGet: {
                    path: '/ready',
                    port: port
                  },
                  initialDelaySeconds: 5,
                  periodSeconds: 5
                }
              }
            ]
          }
        }
      }
    };
  }
  
  // Create Service
  createService(namespace, name, port, targetPort) {
    return {
      apiVersion: 'v1',
      kind: 'Service',
      metadata: {
        name: name,
        namespace: namespace
      },
      spec: {
        type: 'LoadBalancer',
        selector: {
          app: name
        },
        ports: [
          {
            port: port,
            targetPort: targetPort,
            protocol: 'TCP'
          }
        ]
      }
    };
  }
}

// Usage
const eks = new EKSService();

// Create EKS cluster
await eks.createCluster(
  'production-cluster',
  'arn:aws:iam::123456789012:role/eks-cluster-role',
  {
    subnetIds: ['subnet-12345', 'subnet-67890', 'subnet-abcde'],
    securityGroupIds: ['sg-12345'],
    endpointPublicAccess: true,
    endpointPrivateAccess: true
  }
);

// Wait for cluster to be active
await eks.waitForCluster('production-cluster');

// Create node group
await eks.createNodeGroup(
  'production-cluster',
  'default-node-group',
  'arn:aws:iam::123456789012:role/eks-node-role',
  ['subnet-12345', 'subnet-67890'],
  {
    minSize: 2,
    maxSize: 10,
    desiredSize: 3,
    instanceTypes: ['t3.medium', 't3.large']
  }
);
```

### Best Practices:

#### ECS:
1. ✅ Use Fargate for serverless containers
2. ✅ Enable Container Insights for monitoring
3. ✅ Use Service Discovery for microservices
4. ✅ Implement health checks
5. ✅ Use secrets manager for credentials
6. ✅ Enable deployment circuit breaker
7. ✅ Configure auto-scaling policies
8. ✅ Use task placement strategies
9. ✅ Implement blue/green deployments
10. ✅ Monitor with CloudWatch

#### EKS:
1. ✅ Use managed node groups
2. ✅ Enable cluster logging
3. ✅ Implement pod security policies
4. ✅ Use IAM roles for service accounts
5. ✅ Configure horizontal pod autoscaling
6. ✅ Use namespaces for isolation
7. ✅ Implement network policies
8. ✅ Use Helm for package management
9. ✅ Enable cluster autoscaler
10. ✅ Monitor with Prometheus/Grafana

---

## Question 32: Explain AWS CodePipeline for CI/CD automation

### 📋 Answer

**AWS CodePipeline** is a fully managed continuous delivery service that automates build, test, and deploy phases of your release process.

### CodePipeline Components:

```
AWS CodePipeline
├── Source Stage
│   ├── CodeCommit
│   ├── GitHub
│   ├── S3
│   └── ECR
├── Build Stage
│   ├── CodeBuild
│   ├── Jenkins
│   └── Custom
├── Test Stage
│   ├── Unit Tests
│   ├── Integration Tests
│   └── Security Scans
└── Deploy Stage
    ├── CodeDeploy
    ├── ECS
    ├── EKS
    ├── Lambda
    ├── S3
    └── CloudFormation
```

### Complete CodePipeline Implementation:

```javascript
// codepipeline-service.js
import {
  CodePipelineClient,
  CreatePipelineCommand,
  GetPipelineCommand,
  UpdatePipelineCommand,
  StartPipelineExecutionCommand,
  GetPipelineExecutionCommand,
  PutApprovalResultCommand
} from '@aws-sdk/client-codepipeline';

const pipelineClient = new CodePipelineClient({ region: 'us-east-1' });

class CodePipelineService {
  // Create Pipeline
  async createPipeline(pipelineName, roleArn, stages) {
    const command = new CreatePipelineCommand({
      pipeline: {
        name: pipelineName,
        roleArn: roleArn,
        artifactStore: {
          type: 'S3',
          location: 'my-pipeline-artifacts-bucket'
        },
        stages: stages
      },
      tags: [
        { key: 'Environment', value: 'Production' },
        { key: 'Application', value: 'WebApp' }
      ]
    });
    
    try {
      const response = await pipelineClient.send(command);
      console.log('Pipeline created:', response.pipeline.name);
      return response.pipeline;
    } catch (error) {
      console.error('Failed to create pipeline:', error);
      throw error;
    }
  }
  
  // Start Pipeline Execution
  async startExecution(pipelineName) {
    const command = new StartPipelineExecutionCommand({
      name: pipelineName
    });
    
    const response = await pipelineClient.send(command);
    console.log('Pipeline execution started:', response.pipelineExecutionId);
    return response.pipelineExecutionId;
  }
  
  // Get Pipeline Execution Status
  async getExecutionStatus(pipelineName, executionId) {
    const command = new GetPipelineExecutionCommand({
      pipelineName: pipelineName,
      pipelineExecutionId: executionId
    });
    
    const response = await pipelineClient.send(command);
    return {
      status: response.pipelineExecution.status,
      stages: response.pipelineExecution.artifactRevisions
    };
  }
  
  // Approve Manual Approval
  async approveManualApproval(pipelineName, stageName, actionName, token) {
    const command = new PutApprovalResultCommand({
      pipelineName: pipelineName,
      stageName: stageName,
      actionName: actionName,
      token: token,
      result: {
        status: 'Approved',
        summary: 'Approved by automation'
      }
    });
    
    await pipelineClient.send(command);
    console.log('Manual approval granted');
  }
}

// Pipeline Builder
class PipelineBuilder {
  constructor(pipelineName, roleArn, artifactBucket) {
    this.pipeline = {
      name: pipelineName,
      roleArn: roleArn,
      artifactStore: {
        type: 'S3',
        location: artifactBucket
      },
      stages: []
    };
  }
  
  // Add Source Stage (GitHub)
  addGitHubSource(actionName, owner, repo, branch, oauthToken) {
    this.pipeline.stages.push({
      name: 'Source',
      actions: [
        {
          name: actionName,
          actionTypeId: {
            category: 'Source',
            owner: 'ThirdParty',
            provider: 'GitHub',
            version: '1'
          },
          configuration: {
            Owner: owner,
            Repo: repo,
            Branch: branch,
            OAuthToken: oauthToken,
            PollForSourceChanges: false
          },
          outputArtifacts: [
            {
              name: 'SourceOutput'
            }
          ]
        }
      ]
    });
    return this;
  }
  
  // Add CodeCommit Source
  addCodeCommitSource(actionName, repositoryName, branchName) {
    this.pipeline.stages.push({
      name: 'Source',
      actions: [
        {
          name: actionName,
          actionTypeId: {
            category: 'Source',
            owner: 'AWS',
            provider: 'CodeCommit',
            version: '1'
          },
          configuration: {
            RepositoryName: repositoryName,
            BranchName: branchName,
            PollForSourceChanges: false
          },
          outputArtifacts: [
            {
              name: 'SourceOutput'
            }
          ]
        }
      ]
    });
    return this;
  }
  
  // Add Build Stage (CodeBuild)
  addBuildStage(actionName, projectName) {
    this.pipeline.stages.push({
      name: 'Build',
      actions: [
        {
          name: actionName,
          actionTypeId: {
            category: 'Build',
            owner: 'AWS',
            provider: 'CodeBuild',
            version: '1'
          },
          configuration: {
            ProjectName: projectName
          },
          inputArtifacts: [
            {
              name: 'SourceOutput'
            }
          ],
          outputArtifacts: [
            {
              name: 'BuildOutput'
            }
          ]
        }
      ]
    });
    return this;
  }
  
  // Add Test Stage
  addTestStage(actionName, projectName) {
    this.pipeline.stages.push({
      name: 'Test',
      actions: [
        {
          name: actionName,
          actionTypeId: {
            category: 'Test',
            owner: 'AWS',
            provider: 'CodeBuild',
            version: '1'
          },
          configuration: {
            ProjectName: projectName
          },
          inputArtifacts: [
            {
              name: 'BuildOutput'
            }
          ]
        }
      ]
    });
    return this;
  }
  
  // Add Manual Approval Stage
  addManualApprovalStage(actionName, snsTopicArn) {
    this.pipeline.stages.push({
      name: 'Approval',
      actions: [
        {
          name: actionName,
          actionTypeId: {
            category: 'Approval',
            owner: 'AWS',
            provider: 'Manual',
            version: '1'
          },
          configuration: {
            NotificationArn: snsTopicArn,
            CustomData: 'Please review and approve deployment to production'
          }
        }
      ]
    });
    return this;
  }
  
  // Add ECS Deploy Stage
  addECSDeployStage(actionName, clusterName, serviceName) {
    this.pipeline.stages.push({
      name: 'Deploy',
      actions: [
        {
          name: actionName,
          actionTypeId: {
            category: 'Deploy',
            owner: 'AWS',
            provider: 'ECS',
            version: '1'
          },
          configuration: {
            ClusterName: clusterName,
            ServiceName: serviceName,
            FileName: 'imagedefinitions.json'
          },
          inputArtifacts: [
            {
              name: 'BuildOutput'
            }
          ]
        }
      ]
    });
    return this;
  }
  
  // Add CloudFormation Deploy Stage
  addCloudFormationDeployStage(actionName, stackName, templatePath, roleArn) {
    this.pipeline.stages.push({
      name: 'Deploy',
      actions: [
        {
          name: actionName,
          actionTypeId: {
            category: 'Deploy',
            owner: 'AWS',
            provider: 'CloudFormation',
            version: '1'
          },
          configuration: {
            ActionMode: 'CREATE_UPDATE',
            StackName: stackName,
            TemplatePath: `BuildOutput::${templatePath}`,
            Capabilities: 'CAPABILITY_IAM',
            RoleArn: roleArn
          },
          inputArtifacts: [
            {
              name: 'BuildOutput'
            }
          ]
        }
      ]
    });
    return this;
  }
  
  build() {
    return this.pipeline;
  }
}

// Usage - Full CI/CD Pipeline
const codePipeline = new CodePipelineService();

const pipeline = new PipelineBuilder(
  'web-app-pipeline',
  'arn:aws:iam::123456789012:role/CodePipelineRole',
  'my-pipeline-artifacts'
)
  .addGitHubSource(
    'SourceAction',
    'myorg',
    'web-app',
    'main',
    process.env.GITHUB_TOKEN
  )
  .addBuildStage('BuildAction', 'web-app-build')
  .addTestStage('TestAction', 'web-app-test')
  .addManualApprovalStage(
    'ApprovalAction',
    'arn:aws:sns:us-east-1:123456789012:pipeline-approvals'
  )
  .addECSDeployStage('DeployAction', 'production-cluster', 'web-app-service')
  .build();

await codePipeline.createPipeline(
  pipeline.name,
  pipeline.roleArn,
  pipeline.stages
);

// Start execution
const executionId = await codePipeline.startExecution('web-app-pipeline');

// Monitor execution
const status = await codePipeline.getExecutionStatus('web-app-pipeline', executionId);
console.log('Pipeline status:', status);
```

### CodeBuild buildspec.yml:

```yaml
# buildspec.yml
version: 0.2

env:
  variables:
    NODE_ENV: production
  parameter-store:
    DATABASE_URL: /myapp/production/database-url
  secrets-manager:
    DOCKER_HUB_PASSWORD: dockerhub:password

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/web-app
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
      
  build:
    commands:
      - echo Build started on `date`
      - echo Installing dependencies...
      - npm ci
      - echo Running tests...
      - npm test
      - echo Building Docker image...
      - docker build -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
      
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing Docker image...
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - echo Writing image definitions file...
      - printf '[{"name":"web","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
    - appspec.yml
    - taskdef.json

cache:
  paths:
    - 'node_modules/**/*'
```

### Best Practices:

1. ✅ Use separate pipelines for environments
2. ✅ Implement manual approval for production
3. ✅ Enable pipeline notifications (SNS)
4. ✅ Use CodeBuild for build/test stages
5. ✅ Store artifacts in S3
6. ✅ Use parameter store for secrets
7. ✅ Enable CloudWatch Events for triggers
8. ✅ Implement rollback mechanisms
9. ✅ Use blue/green deployments
10. ✅ Monitor with CloudWatch dashboards

---

## Question 33: Explain AWS Lambda best practices and optimization

### 📋 Answer

**AWS Lambda** is a serverless compute service that runs code in response to events. Optimization is crucial for performance and cost.

### Lambda Optimization Areas:

```
Lambda Optimization
├── Cold Start Reduction
│   ├── Provisioned Concurrency
│   ├── Keep functions warm
│   └── Minimize package size
├── Memory Optimization
│   ├── Right-sizing memory
│   └── CPU scales with memory
├── Code Optimization
│   ├── Connection pooling
│   ├── Minimize dependencies
│   └── Use Lambda layers
└── Cost Optimization
    ├── Reduce execution time
    ├── Optimize memory
    └── Use ARM64 (Graviton2)
```

### Optimized Lambda Implementation:

```javascript
// optimized-lambda.js

// ❌ Bad: Creating client inside handler
export const badHandler = async (event) => {
  const { DynamoDBClient } = await import('@aws-sdk/client-dynamodb');
  const client = new DynamoDBClient({});  // Created on every invocation
  // ...
};

// ✅ Good: Reuse client across invocations
import { DynamoDBClient, GetItemCommand } from '@aws-sdk/client-dynamodb';

// Initialize outside handler (reused across invocations)
const dynamoClient = new DynamoDBClient({
  region: process.env.AWS_REGION,
  maxAttempts: 3
});

// Connection pool for RDS
import { createPool } from 'mysql2/promise';

let pool;

function getPool() {
  if (!pool) {
    pool = createPool({
      host: process.env.DB_HOST,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      database: process.env.DB_NAME,
      connectionLimit: 1,  // Lambda: 1 connection per instance
      enableKeepAlive: true,
      keepAliveInitialDelay: 0
    });
  }
  return pool;
}

// ✅ Optimized Handler
export const handler = async (event) => {
  try {
    // Parse event
    const body = JSON.parse(event.body);
    
    // DynamoDB operation
    const command = new GetItemCommand({
      TableName: 'Users',
      Key: {
        id: { S: body.userId }
      }
    });
    
    const result = await dynamoClient.send(command);
    
    // Database operation (reused connection)
    const dbPool = getPool();
    const [rows] = await dbPool.execute(
      'SELECT * FROM orders WHERE user_id = ?',
      [body.userId]
    );
    
    return {
      statusCode: 200,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
      },
      body: JSON.stringify({
        user: result.Item,
        orders: rows
      })
    };
    
  } catch (error) {
    console.error('Error:', error);
    
    return {
      statusCode: 500,
      body: JSON.stringify({
        error: error.message
      })
    };
  }
};
```

### Lambda Performance Patterns:

```javascript
// lambda-patterns.js

// 1. Lazy Loading (reduce cold start)
let heavyLibrary;

async function getHeavyLibrary() {
  if (!heavyLibrary) {
    heavyLibrary = await import('heavy-library');
  }
  return heavyLibrary;
}

// 2. Caching with TTL
const cache = new Map();
const CACHE_TTL = 5 * 60 * 1000;  // 5 minutes

function getCached(key) {
  const cached = cache.get(key);
  if (cached && Date.now() - cached.timestamp < CACHE_TTL) {
    return cached.value;
  }
  return null;
}

function setCache(key, value) {
  cache.set(key, {
    value,
    timestamp: Date.now()
  });
}

// 3. Parallel Processing
export const batchHandler = async (event) => {
  const items = event.Records;
  
  // Process in parallel
  const results = await Promise.allSettled(
    items.map(item => processItem(item))
  );
  
  const failures = results.filter(r => r.status === 'rejected');
  
  if (failures.length > 0) {
    console.error('Failed items:', failures);
    // Return failed items for retry
    return {
      batchItemFailures: failures.map(f => ({
        itemIdentifier: f.reason.itemId
      }))
    };
  }
  
  return { statusCode: 200 };
};

async function processItem(item) {
  // Processing logic
}

// 4. Streaming Response (for large data)
import { Readable } from 'stream';

export const streamHandler = awslambda.streamifyResponse(
  async (event, responseStream, context) => {
    const httpResponseMetadata = {
      statusCode: 200,
      headers: {
        'Content-Type': 'application/json',
        'X-Custom-Header': 'Example'
      }
    };
    
    responseStream = awslambda.HttpResponseStream.from(
      responseStream,
      httpResponseMetadata
    );
    
    // Stream large dataset
    for (let i = 0; i < 1000; i++) {
      const data = await fetchDataChunk(i);
      responseStream.write(JSON.stringify(data) + '\n');
    }
    
    responseStream.end();
  }
);

// 5. Memory Optimization
export const memoryOptimizedHandler = async (event) => {
  const startTime = Date.now();
  const startMemory = process.memoryUsage().heapUsed;
  
  // Your logic here
  const result = await processData(event);
  
  const endTime = Date.now();
  const endMemory = process.memoryUsage().heapUsed;
  
  console.log('Metrics:', {
    executionTime: endTime - startTime,
    memoryUsed: (endMemory - startMemory) / 1024 / 1024,  // MB
    configuredMemory: process.env.AWS_LAMBDA_FUNCTION_MEMORY_SIZE
  });
  
  return result;
};

// 6. Error Handling with Retries
import { SQSClient, SendMessageCommand } from '@aws-sdk/client-sqs';

const sqsClient = new SQSClient({});

export const reliableHandler = async (event) => {
  const maxRetries = 3;
  let attempt = 0;
  
  while (attempt < maxRetries) {
    try {
      return await processEvent(event);
    } catch (error) {
      attempt++;
      
      if (attempt >= maxRetries) {
        // Send to DLQ
        await sqsClient.send(new SendMessageCommand({
          QueueUrl: process.env.DLQ_URL,
          MessageBody: JSON.stringify({
            event,
            error: error.message,
            attempts: attempt
          })
        }));
        
        throw error;
      }
      
      // Exponential backoff
      await new Promise(resolve => 
        setTimeout(resolve, Math.pow(2, attempt) * 1000)
      );
    }
  }
};
```

### Lambda Layer (Shared Dependencies):

```javascript
// Create layer structure
// nodejs/
//   node_modules/
//     @aws-sdk/
//     lodash/
//     etc.

// Deploy layer
import { LambdaClient, PublishLayerVersionCommand } from '@aws-sdk/client-lambda';
import { readFileSync } from 'fs';

const lambdaClient = new LambdaClient({ region: 'us-east-1' });

async function createLayer(layerName, zipFile, runtimes) {
  const command = new PublishLayerVersionCommand({
    LayerName: layerName,
    Description: 'Shared dependencies layer',
    Content: {
      ZipFile: readFileSync(zipFile)
    },
    CompatibleRuntimes: runtimes,
    CompatibleArchitectures: ['x86_64', 'arm64']
  });
  
  const response = await lambdaClient.send(command);
  console.log('Layer created:', response.LayerVersionArn);
  return response.LayerVersionArn;
}

// Usage
await createLayer(
  'shared-dependencies',
  './layer.zip',
  ['nodejs18.x', 'nodejs20.x']
);
```

### Best Practices:

1. ✅ Use provisioned concurrency for latency-sensitive workloads
2. ✅ Initialize SDK clients outside handler
3. ✅ Use connection pooling for databases
4. ✅ Implement caching for frequently accessed data
5. ✅ Use Lambda layers for shared dependencies
6. ✅ Right-size memory allocation (test different sizes)
7. ✅ Use ARM64 (Graviton2) for 20% cost savings
8. ✅ Enable X-Ray tracing for debugging
9. ✅ Implement proper error handling and retries
10. ✅ Monitor with CloudWatch metrics and alarms

---

## Key Takeaways

### Question 31 (ECS vs EKS):
- ECS: AWS-native, easier, no control plane cost
- EKS: Kubernetes standard, portable, $0.10/hour
- Both support EC2 and Fargate launch types
- ECS for AWS-specific, EKS for multi-cloud
- Use Fargate for serverless containers
- Implement health checks and auto-scaling

### Question 32 (CodePipeline):
- Fully managed CI/CD service
- Source, Build, Test, Deploy stages
- Integrates with CodeCommit, GitHub, S3
- Manual approval for production gates
- Blue/green deployments support
- SNS notifications for pipeline events

### Question 33 (Lambda Optimization):
- Initialize clients outside handler
- Use connection pooling for databases
- Implement caching with TTL
- Right-size memory allocation
- Use Lambda layers for dependencies
- Use ARM64 for cost savings
- Monitor cold starts and duration
