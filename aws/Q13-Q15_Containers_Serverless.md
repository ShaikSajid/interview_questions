# AWS Interview Questions: Containers & Serverless (Q13-Q15)

## Question 13: Explain Amazon ECS (Elastic Container Service) and EKS (Elastic Kubernetes Service)

### 📋 Answer

**Amazon ECS** and **EKS** are container orchestration services for running Docker containers at scale.

### ECS vs EKS Comparison:

| Feature | ECS | EKS |
|---------|-----|-----|
| **Orchestrator** | AWS proprietary | Kubernetes |
| **Learning Curve** | Easy | Steep |
| **AWS Integration** | Native | Good |
| **Portability** | AWS only | Multi-cloud |
| **Cost** | Free (pay for resources) | $0.10/hour per cluster |
| **Community** | AWS | Huge (CNCF) |
| **Use Case** | AWS-native apps | Cloud-agnostic, complex |

### ECS Architecture:

```
ECS Cluster
├── Services (Long-running tasks)
│   ├── Task Definition (Container blueprint)
│   ├── Tasks (Running containers)
│   └── Load Balancer
├── Task Definitions
│   ├── Container Definitions
│   ├── CPU/Memory
│   └── Environment Variables
└── Capacity Providers
    ├── Fargate (Serverless)
    └── EC2 (Self-managed)
```

### Complete ECS Implementation:

```javascript
// ecs-manager.js
import {
  ECSClient,
  CreateClusterCommand,
  RegisterTaskDefinitionCommand,
  CreateServiceCommand,
  RunTaskCommand,
  DescribeTasksCommand,
  UpdateServiceCommand,
  DeleteServiceCommand,
  ListTasksCommand,
  DescribeClustersCommand
} from '@aws-sdk/client-ecs';

const ecsClient = new ECSClient({ region: 'us-east-1' });

// 1. Create ECS Cluster
async function createCluster(clusterName) {
  const command = new CreateClusterCommand({
    clusterName: clusterName,
    capacityProviders: ['FARGATE', 'FARGATE_SPOT'],
    defaultCapacityProviderStrategy: [
      {
        capacityProvider: 'FARGATE',
        weight: 1,
        base: 1
      }
    ],
    tags: [
      { key: 'Environment', value: 'Production' },
      { key: 'ManagedBy', value: 'Terraform' }
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
async function registerTaskDefinition(config) {
  const command = new RegisterTaskDefinitionCommand({
    family: config.family,
    networkMode: 'awsvpc',
    requiresCompatibilities: ['FARGATE'],
    cpu: '256',  // 0.25 vCPU
    memory: '512',  // 512 MB
    taskRoleArn: config.taskRoleArn,
    executionRoleArn: config.executionRoleArn,
    containerDefinitions: [
      {
        name: config.containerName,
        image: config.image,
        cpu: 256,
        memory: 512,
        essential: true,
        portMappings: [
          {
            containerPort: config.port || 3000,
            protocol: 'tcp'
          }
        ],
        environment: Object.entries(config.environment || {}).map(([name, value]) => ({
          name,
          value
        })),
        secrets: config.secrets?.map(secret => ({
          name: secret.name,
          valueFrom: secret.valueFrom  // SSM Parameter or Secrets Manager ARN
        })),
        logConfiguration: {
          logDriver: 'awslogs',
          options: {
            'awslogs-group': `/ecs/${config.family}`,
            'awslogs-region': 'us-east-1',
            'awslogs-stream-prefix': 'ecs'
          }
        },
        healthCheck: {
          command: ['CMD-SHELL', 'curl -f http://localhost:3000/health || exit 1'],
          interval: 30,
          timeout: 5,
          retries: 3,
          startPeriod: 60
        }
      }
    ]
  });
  
  try {
    const response = await ecsClient.send(command);
    console.log('Task definition registered:', response.taskDefinition.taskDefinitionArn);
    return response.taskDefinition;
  } catch (error) {
    console.error('Failed to register task definition:', error);
    throw error;
  }
}

// 3. Create ECS Service
async function createService(config) {
  const command = new CreateServiceCommand({
    cluster: config.clusterName,
    serviceName: config.serviceName,
    taskDefinition: config.taskDefinition,
    desiredCount: config.desiredCount || 2,
    launchType: 'FARGATE',
    networkConfiguration: {
      awsvpcConfiguration: {
        subnets: config.subnets,
        securityGroups: config.securityGroups,
        assignPublicIp: 'ENABLED'
      }
    },
    loadBalancers: config.loadBalancer ? [
      {
        targetGroupArn: config.loadBalancer.targetGroupArn,
        containerName: config.containerName,
        containerPort: config.port || 3000
      }
    ] : [],
    deploymentConfiguration: {
      maximumPercent: 200,
      minimumHealthyPercent: 100,
      deploymentCircuitBreaker: {
        enable: true,
        rollback: true
      }
    },
    enableExecuteCommand: true,  // Enable ECS Exec for debugging
    tags: [
      { key: 'Environment', value: 'Production' }
    ]
  });
  
  try {
    const response = await ecsClient.send(command);
    console.log('Service created:', response.service.serviceName);
    return response.service;
  } catch (error) {
    console.error('Failed to create service:', error);
    throw error;
  }
}

// 4. Run One-Time Task
async function runTask(config) {
  const command = new RunTaskCommand({
    cluster: config.clusterName,
    taskDefinition: config.taskDefinition,
    launchType: 'FARGATE',
    count: 1,
    networkConfiguration: {
      awsvpcConfiguration: {
        subnets: config.subnets,
        securityGroups: config.securityGroups,
        assignPublicIp: 'ENABLED'
      }
    },
    overrides: {
      containerOverrides: [
        {
          name: config.containerName,
          environment: Object.entries(config.environment || {}).map(([name, value]) => ({
            name,
            value
          }))
        }
      ]
    }
  });
  
  try {
    const response = await ecsClient.send(command);
    const taskArn = response.tasks[0].taskArn;
    console.log('Task started:', taskArn);
    return response.tasks[0];
  } catch (error) {
    console.error('Failed to run task:', error);
    throw error;
  }
}

// 5. Describe Tasks
async function describeTasks(clusterName, taskArns) {
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

// 6. Update Service (Deploy new version)
async function updateService(clusterName, serviceName, newTaskDefinition) {
  const command = new UpdateServiceCommand({
    cluster: clusterName,
    service: serviceName,
    taskDefinition: newTaskDefinition,
    forceNewDeployment: true
  });
  
  try {
    const response = await ecsClient.send(command);
    console.log('Service updated - deploying new version');
    return response.service;
  } catch (error) {
    console.error('Failed to update service:', error);
    throw error;
  }
}

// 7. Scale Service
async function scaleService(clusterName, serviceName, desiredCount) {
  const command = new UpdateServiceCommand({
    cluster: clusterName,
    service: serviceName,
    desiredCount: desiredCount
  });
  
  try {
    await ecsClient.send(command);
    console.log(`Service scaled to ${desiredCount} tasks`);
  } catch (error) {
    console.error('Failed to scale service:', error);
    throw error;
  }
}

// 8. List Running Tasks
async function listRunningTasks(clusterName, serviceName) {
  const command = new ListTasksCommand({
    cluster: clusterName,
    serviceName: serviceName,
    desiredStatus: 'RUNNING'
  });
  
  const response = await ecsClient.send(command);
  return response.taskArns;
}

// 9. Get Cluster Info
async function getClusterInfo(clusterName) {
  const command = new DescribeClustersCommand({
    clusters: [clusterName],
    include: ['STATISTICS', 'TAGS']
  });
  
  const response = await ecsClient.send(command);
  const cluster = response.clusters[0];
  
  return {
    name: cluster.clusterName,
    status: cluster.status,
    runningTasksCount: cluster.runningTasksCount,
    pendingTasksCount: cluster.pendingTasksCount,
    activeServicesCount: cluster.activeServicesCount,
    registeredContainerInstancesCount: cluster.registeredContainerInstancesCount
  };
}
```

### Complete Application Deployment:

```javascript
// deploy-app-to-ecs.js
async function deployApplication() {
  const clusterName = 'my-app-cluster';
  const serviceName = 'my-app-service';
  
  try {
    // 1. Create cluster
    console.log('Creating ECS cluster...');
    await createCluster(clusterName);
    
    // 2. Register task definition
    console.log('Registering task definition...');
    const taskDef = await registerTaskDefinition({
      family: 'my-app',
      containerName: 'app-container',
      image: '123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest',
      port: 3000,
      taskRoleArn: 'arn:aws:iam::123456789012:role/ECSTaskRole',
      executionRoleArn: 'arn:aws:iam::123456789012:role/ECSExecutionRole',
      environment: {
        NODE_ENV: 'production',
        PORT: '3000',
        DATABASE_HOST: 'my-db.us-east-1.rds.amazonaws.com'
      },
      secrets: [
        {
          name: 'DATABASE_PASSWORD',
          valueFrom: 'arn:aws:secretsmanager:us-east-1:123456789012:secret:db-password'
        }
      ]
    });
    
    // 3. Create service
    console.log('Creating ECS service...');
    await createService({
      clusterName: clusterName,
      serviceName: serviceName,
      taskDefinition: taskDef.taskDefinitionArn,
      containerName: 'app-container',
      port: 3000,
      desiredCount: 3,
      subnets: [
        'subnet-0123456789abcdef0',
        'subnet-0123456789abcdef1'
      ],
      securityGroups: ['sg-0123456789abcdef0'],
      loadBalancer: {
        targetGroupArn: 'arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-app-tg/abc123'
      }
    });
    
    console.log('Application deployed successfully!');
    
    // 4. Monitor deployment
    await monitorDeployment(clusterName, serviceName);
    
  } catch (error) {
    console.error('Deployment failed:', error);
    throw error;
  }
}

async function monitorDeployment(clusterName, serviceName) {
  console.log('Monitoring deployment...');
  
  for (let i = 0; i < 30; i++) {
    const taskArns = await listRunningTasks(clusterName, serviceName);
    const tasks = await describeTasks(clusterName, taskArns);
    
    const runningCount = tasks.filter(t => t.lastStatus === 'RUNNING').length;
    console.log(`Running tasks: ${runningCount}/${tasks.length}`);
    
    if (runningCount === 3) {
      console.log('Deployment successful!');
      return;
    }
    
    await new Promise(resolve => setTimeout(resolve, 10000));
  }
  
  throw new Error('Deployment timed out');
}

// Execute deployment
// await deployApplication();
```

### ECS Task Definition (JSON format):

```json
{
  "family": "my-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "taskRoleArn": "arn:aws:iam::123456789012:role/ECSTaskRole",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ECSExecutionRole",
  "containerDefinitions": [
    {
      "name": "app-container",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest",
      "cpu": 256,
      "memory": 512,
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
          "name": "DATABASE_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:db-password"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

### Best Practices:

1. ✅ Use Fargate for serverless containers
2. ✅ Implement health checks
3. ✅ Store secrets in Secrets Manager
4. ✅ Enable ECS Exec for debugging
5. ✅ Use Application Load Balancer
6. ✅ Configure auto-scaling
7. ✅ Monitor with CloudWatch Container Insights
8. ✅ Use deployment circuit breaker
9. ✅ Tag all resources
10. ✅ Implement blue/green deployments

---

## Question 14: Explain AWS API Gateway and building REST APIs

### 📋 Answer

**Amazon API Gateway** is a fully managed service for creating, publishing, maintaining, monitoring, and securing REST, HTTP, and WebSocket APIs at any scale.

### API Gateway Types:

1. **REST API**: Full-featured API management
2. **HTTP API**: Lower cost, faster performance
3. **WebSocket API**: Real-time two-way communication

### API Gateway Features:

```
API Gateway
├── REST APIs
│   ├── Resources & Methods
│   ├── Request/Response Transformations
│   ├── Authorizers (Lambda, IAM, Cognito)
│   ├── Usage Plans & API Keys
│   └── Stage Variables
├── Integration Types
│   ├── Lambda Function
│   ├── HTTP Endpoint
│   ├── AWS Service (DynamoDB, S3)
│   └── Mock
└── Features
    ├── Throttling
    ├── Caching
    ├── CORS
    └── Monitoring
```

### Complete API Gateway Implementation:

```javascript
// api-gateway-manager.js
import {
  APIGatewayClient,
  CreateRestApiCommand,
  GetResourcesCommand,
  CreateResourceCommand,
  PutMethodCommand,
  PutIntegrationCommand,
  PutMethodResponseCommand,
  PutIntegrationResponseCommand,
  CreateDeploymentCommand,
  CreateStageCommand,
  CreateUsagePlanCommand,
  CreateApiKeyCommand,
  GetExportCommand
} from '@aws-sdk/client-api-gateway';

const apiGatewayClient = new APIGatewayClient({ region: 'us-east-1' });

// 1. Create REST API
async function createRestAPI(apiName) {
  const command = new CreateRestApiCommand({
    name: apiName,
    description: 'API for user management',
    endpointConfiguration: {
      types: ['REGIONAL']  // or 'EDGE' for CloudFront distribution
    },
    tags: {
      Environment: 'Production'
    }
  });
  
  try {
    const response = await apiGatewayClient.send(command);
    console.log('REST API created:', response.id);
    return response;
  } catch (error) {
    console.error('Failed to create API:', error);
    throw error;
  }
}

// 2. Get Root Resource
async function getRootResource(apiId) {
  const command = new GetResourcesCommand({ restApiId: apiId });
  const response = await apiGatewayClient.send(command);
  return response.items.find(item => item.path === '/');
}

// 3. Create Resource (endpoint path)
async function createResource(apiId, parentId, pathPart) {
  const command = new CreateResourceCommand({
    restApiId: apiId,
    parentId: parentId,
    pathPart: pathPart
  });
  
  const response = await apiGatewayClient.send(command);
  console.log(`Resource created: /${pathPart}`);
  return response;
}

// 4. Create Method (GET, POST, etc.)
async function createMethod(apiId, resourceId, httpMethod, authorizationType = 'NONE') {
  const command = new PutMethodCommand({
    restApiId: apiId,
    resourceId: resourceId,
    httpMethod: httpMethod,
    authorizationType: authorizationType,
    apiKeyRequired: false,
    requestParameters: {
      'method.request.querystring.userId': false,
      'method.request.header.Authorization': false
    }
  });
  
  await apiGatewayClient.send(command);
  console.log(`Method created: ${httpMethod}`);
}

// 5. Create Lambda Integration
async function createLambdaIntegration(apiId, resourceId, httpMethod, lambdaArn) {
  const command = new PutIntegrationCommand({
    restApiId: apiId,
    resourceId: resourceId,
    httpMethod: httpMethod,
    type: 'AWS_PROXY',  // Lambda proxy integration
    integrationHttpMethod: 'POST',
    uri: `arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/${lambdaArn}/invocations`,
    credentials: 'arn:aws:iam::123456789012:role/APIGatewayLambdaExecRole'
  });
  
  await apiGatewayClient.send(command);
  console.log('Lambda integration created');
}

// 6. Create Method Response
async function createMethodResponse(apiId, resourceId, httpMethod, statusCode) {
  const command = new PutMethodResponseCommand({
    restApiId: apiId,
    resourceId: resourceId,
    httpMethod: httpMethod,
    statusCode: statusCode,
    responseParameters: {
      'method.response.header.Access-Control-Allow-Origin': false
    },
    responseModels: {
      'application/json': 'Empty'
    }
  });
  
  await apiGatewayClient.send(command);
  console.log(`Method response created: ${statusCode}`);
}

// 7. Deploy API
async function deployAPI(apiId, stageName) {
  const command = new CreateDeploymentCommand({
    restApiId: apiId,
    stageName: stageName,
    stageDescription: `Deployment to ${stageName}`,
    description: `Deployment at ${new Date().toISOString()}`
  });
  
  const response = await apiGatewayClient.send(command);
  console.log(`API deployed to stage: ${stageName}`);
  return response;
}

// 8. Create Usage Plan
async function createUsagePlan(apiId, stageName) {
  const command = new CreateUsagePlanCommand({
    name: 'Basic Plan',
    description: 'Basic usage plan with throttling',
    apiStages: [
      {
        apiId: apiId,
        stage: stageName
      }
    ],
    throttle: {
      rateLimit: 100,  // requests per second
      burstLimit: 200   // concurrent requests
    },
    quota: {
      limit: 10000,     // total requests
      period: 'MONTH'
    }
  });
  
  const response = await apiGatewayClient.send(command);
  console.log('Usage plan created:', response.id);
  return response;
}

// 9. Create API Key
async function createAPIKey(name) {
  const command = new CreateApiKeyCommand({
    name: name,
    description: `API key for ${name}`,
    enabled: true
  });
  
  const response = await apiGatewayClient.send(command);
  console.log('API key created:', response.value);
  return response;
}
```

### Build Complete REST API:

```javascript
// build-user-api.js
async function buildUserAPI() {
  const apiName = 'UserManagementAPI';
  
  try {
    // 1. Create REST API
    console.log('Creating REST API...');
    const api = await createRestAPI(apiName);
    const apiId = api.id;
    
    // 2. Get root resource
    const rootResource = await getRootResource(apiId);
    
    // 3. Create /users resource
    console.log('Creating /users resource...');
    const usersResource = await createResource(apiId, rootResource.id, 'users');
    
    // 4. Create /users/{userId} resource
    console.log('Creating /users/{userId} resource...');
    const userIdResource = await createResource(apiId, usersResource.id, '{userId}');
    
    // 5. Setup GET /users (List all users)
    console.log('Setting up GET /users...');
    await createMethod(apiId, usersResource.id, 'GET', 'AWS_IAM');
    await createLambdaIntegration(
      apiId,
      usersResource.id,
      'GET',
      'arn:aws:lambda:us-east-1:123456789012:function:ListUsers'
    );
    await createMethodResponse(apiId, usersResource.id, 'GET', '200');
    
    // 6. Setup POST /users (Create user)
    console.log('Setting up POST /users...');
    await createMethod(apiId, usersResource.id, 'POST', 'AWS_IAM');
    await createLambdaIntegration(
      apiId,
      usersResource.id,
      'POST',
      'arn:aws:lambda:us-east-1:123456789012:function:CreateUser'
    );
    await createMethodResponse(apiId, usersResource.id, 'POST', '201');
    
    // 7. Setup GET /users/{userId} (Get user)
    console.log('Setting up GET /users/{userId}...');
    await createMethod(apiId, userIdResource.id, 'GET', 'AWS_IAM');
    await createLambdaIntegration(
      apiId,
      userIdResource.id,
      'GET',
      'arn:aws:lambda:us-east-1:123456789012:function:GetUser'
    );
    await createMethodResponse(apiId, userIdResource.id, 'GET', '200');
    
    // 8. Setup PUT /users/{userId} (Update user)
    console.log('Setting up PUT /users/{userId}...');
    await createMethod(apiId, userIdResource.id, 'PUT', 'AWS_IAM');
    await createLambdaIntegration(
      apiId,
      userIdResource.id,
      'PUT',
      'arn:aws:lambda:us-east-1:123456789012:function:UpdateUser'
    );
    await createMethodResponse(apiId, userIdResource.id, 'PUT', '200');
    
    // 9. Setup DELETE /users/{userId} (Delete user)
    console.log('Setting up DELETE /users/{userId}...');
    await createMethod(apiId, userIdResource.id, 'DELETE', 'AWS_IAM');
    await createLambdaIntegration(
      apiId,
      userIdResource.id,
      'DELETE',
      'arn:aws:lambda:us-east-1:123456789012:function:DeleteUser'
    );
    await createMethodResponse(apiId, userIdResource.id, 'DELETE', '204');
    
    // 10. Deploy to production
    console.log('Deploying API...');
    await deployAPI(apiId, 'production');
    
    // 11. Create usage plan
    console.log('Creating usage plan...');
    const usagePlan = await createUsagePlan(apiId, 'production');
    
    // 12. Create API key
    console.log('Creating API key...');
    await createAPIKey('client-app-key');
    
    console.log('\n✅ API Gateway setup complete!');
    console.log(`API Endpoint: https://${apiId}.execute-api.us-east-1.amazonaws.com/production`);
    
  } catch (error) {
    console.error('Failed to build API:', error);
    throw error;
  }
}

// Execute
// await buildUserAPI();
```

### Lambda Function for API Gateway:

```javascript
// lambda-handler.js - API Gateway Lambda Integration
export const handler = async (event) => {
  console.log('Received event:', JSON.stringify(event, null, 2));
  
  const httpMethod = event.httpMethod;
  const path = event.path;
  const pathParameters = event.pathParameters;
  const queryStringParameters = event.queryStringParameters;
  const body = event.body ? JSON.parse(event.body) : null;
  const headers = event.headers;
  
  try {
    let response;
    
    // Route handling
    if (path === '/users' && httpMethod === 'GET') {
      response = await listUsers(queryStringParameters);
    } else if (path === '/users' && httpMethod === 'POST') {
      response = await createUser(body);
    } else if (path.match(/^\/users\/[^/]+$/) && httpMethod === 'GET') {
      response = await getUser(pathParameters.userId);
    } else if (path.match(/^\/users\/[^/]+$/) && httpMethod === 'PUT') {
      response = await updateUser(pathParameters.userId, body);
    } else if (path.match(/^\/users\/[^/]+$/) && httpMethod === 'DELETE') {
      response = await deleteUser(pathParameters.userId);
    } else {
      return formatResponse(404, { error: 'Not Found' });
    }
    
    return formatResponse(response.statusCode, response.body);
    
  } catch (error) {
    console.error('Error:', error);
    return formatResponse(500, { error: 'Internal Server Error', message: error.message });
  }
};

// Format API Gateway response
function formatResponse(statusCode, body) {
  return {
    statusCode: statusCode,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Headers': 'Content-Type,Authorization',
      'Access-Control-Allow-Methods': 'GET,POST,PUT,DELETE,OPTIONS'
    },
    body: JSON.stringify(body)
  };
}

// API handlers
async function listUsers(query) {
  const limit = parseInt(query?.limit) || 10;
  const offset = parseInt(query?.offset) || 0;
  
  // Fetch from database
  const users = await db.listUsers(limit, offset);
  
  return {
    statusCode: 200,
    body: {
      users: users,
      count: users.length,
      offset: offset
    }
  };
}

async function createUser(userData) {
  // Validate input
  if (!userData.email || !userData.name) {
    return {
      statusCode: 400,
      body: { error: 'Missing required fields' }
    };
  }
  
  const user = await db.createUser(userData);
  
  return {
    statusCode: 201,
    body: user
  };
}

async function getUser(userId) {
  const user = await db.getUserById(userId);
  
  if (!user) {
    return {
      statusCode: 404,
      body: { error: 'User not found' }
    };
  }
  
  return {
    statusCode: 200,
    body: user
  };
}

async function updateUser(userId, updates) {
  const user = await db.updateUser(userId, updates);
  
  if (!user) {
    return {
      statusCode: 404,
      body: { error: 'User not found' }
    };
  }
  
  return {
    statusCode: 200,
    body: user
  };
}

async function deleteUser(userId) {
  await db.deleteUser(userId);
  
  return {
    statusCode: 204,
    body: {}
  };
}
```

### API Gateway with Custom Authorizer:

```javascript
// custom-authorizer.js
export const handler = async (event) => {
  const token = event.authorizationToken;  // From 'Authorization' header
  const methodArn = event.methodArn;
  
  try {
    // Validate token (JWT, API key, etc.)
    const decodedToken = await validateToken(token);
    
    // Generate IAM policy
    const policy = generatePolicy(decodedToken.userId, 'Allow', methodArn, {
      userId: decodedToken.userId,
      email: decodedToken.email,
      roles: decodedToken.roles
    });
    
    return policy;
    
  } catch (error) {
    console.error('Authorization failed:', error);
    throw new Error('Unauthorized');
  }
};

function generatePolicy(principalId, effect, resource, context) {
  return {
    principalId: principalId,
    policyDocument: {
      Version: '2012-10-17',
      Statement: [
        {
          Action: 'execute-api:Invoke',
          Effect: effect,
          Resource: resource
        }
      ]
    },
    context: context  // Passed to Lambda as event.requestContext.authorizer
  };
}

async function validateToken(token) {
  // Implement token validation (JWT verification, database lookup, etc.)
  // For JWT:
  const decoded = jwt.verify(token, process.env.JWT_SECRET);
  return decoded;
}
```

### Best Practices:

1. ✅ Use Lambda proxy integration for flexibility
2. ✅ Enable CORS for browser access
3. ✅ Implement request/response validation
4. ✅ Use custom authorizers for authentication
5. ✅ Enable CloudWatch logging
6. ✅ Configure throttling and quotas
7. ✅ Use stage variables for environment configs
8. ✅ Enable API caching for performance
9. ✅ Use usage plans for API monetization
10. ✅ Monitor with CloudWatch and X-Ray

---

## Question 15: Explain AWS Step Functions for workflow orchestration

### 📋 Answer

**AWS Step Functions** is a serverless orchestration service that lets you coordinate multiple AWS services into serverless workflows.

### Key Concepts:

1. **State Machine**: Workflow definition
2. **States**: Individual steps (Task, Choice, Parallel, etc.)
3. **Transitions**: Flow between states
4. **Input/Output**: Data passed between states
5. **Error Handling**: Retry and catch configurations

### State Types:

```
Step Functions States
├── Task (Execute work)
├── Choice (Conditional branching)
├── Parallel (Execute branches in parallel)
├── Wait (Delay execution)
├── Pass (Pass input to output)
├── Succeed (Successful termination)
├── Fail (Failed termination)
└── Map (Iterate over items)
```

### Complete Step Functions Example:

```json
{
  "Comment": "Order Processing Workflow",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ValidateOrder",
      "ResultPath": "$.validation",
      "Next": "CheckInventory",
      "Catch": [
        {
          "ErrorEquals": ["ValidationError"],
          "ResultPath": "$.error",
          "Next": "OrderRejected"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": ["States.TaskFailed"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ]
    },
    
    "CheckInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:CheckInventory",
      "ResultPath": "$.inventory",
      "Next": "IsInStock"
    },
    
    "IsInStock": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.inventory.inStock",
          "BooleanEquals": true,
          "Next": "ProcessPayment"
        }
      ],
      "Default": "OutOfStock"
    },
    
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ProcessPayment",
      "ResultPath": "$.payment",
      "Next": "ParallelProcessing",
      "Catch": [
        {
          "ErrorEquals": ["PaymentError"],
          "ResultPath": "$.error",
          "Next": "PaymentFailed"
        }
      ]
    },
    
    "ParallelProcessing": {
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "UpdateInventory",
          "States": {
            "UpdateInventory": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:us-east-1:123456789012:function:UpdateInventory",
              "End": true
            }
          }
        },
        {
          "StartAt": "SendConfirmationEmail",
          "States": {
            "SendConfirmationEmail": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:us-east-1:123456789012:function:SendEmail",
              "End": true
            }
          }
        },
        {
          "StartAt": "CreateShipment",
          "States": {
            "CreateShipment": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:us-east-1:123456789012:function:CreateShipment",
              "End": true
            }
          }
        }
      ],
      "Next": "OrderCompleted"
    },
    
    "OutOfStock": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:NotifyOutOfStock",
      "Next": "OrderRejected"
    },
    
    "PaymentFailed": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:HandlePaymentFailure",
      "Next": "OrderRejected"
    },
    
    "OrderRejected": {
      "Type": "Fail",
      "Error": "OrderProcessingFailed",
      "Cause": "Order could not be processed"
    },
    
    "OrderCompleted": {
      "Type": "Succeed"
    }
  }
}
```

### Manage Step Functions with SDK:

```javascript
// step-functions-manager.js
import {
  SFNClient,
  CreateStateMachineCommand,
  StartExecutionCommand,
  DescribeExecutionCommand,
  GetExecutionHistoryCommand,
  StopExecutionCommand,
  ListExecutionsCommand
} from '@aws-sdk/client-sfn';
import { readFileSync } from 'fs';

const sfnClient = new SFNClient({ region: 'us-east-1' });

// 1. Create State Machine
async function createStateMachine(name, definitionPath, roleArn) {
  const definition = readFileSync(definitionPath, 'utf8');
  
  const command = new CreateStateMachineCommand({
    name: name,
    definition: definition,
    roleArn: roleArn,
    type: 'STANDARD',  // or 'EXPRESS' for high-volume, short-duration
    loggingConfiguration: {
      level: 'ALL',
      includeExecutionData: true,
      destinations: [
        {
          cloudWatchLogsLogGroup: {
            logGroupArn: 'arn:aws:logs:us-east-1:123456789012:log-group:/aws/stepfunctions/order-processing'
          }
        }
      ]
    },
    tags: [
      { key: 'Environment', value: 'Production' }
    ]
  });
  
  try {
    const response = await sfnClient.send(command);
    console.log('State machine created:', response.stateMachineArn);
    return response;
  } catch (error) {
    console.error('Failed to create state machine:', error);
    throw error;
  }
}

// 2. Start Execution
async function startExecution(stateMachineArn, input, executionName) {
  const command = new StartExecutionCommand({
    stateMachineArn: stateMachineArn,
    name: executionName || `execution-${Date.now()}`,
    input: JSON.stringify(input)
  });
  
  try {
    const response = await sfnClient.send(command);
    console.log('Execution started:', response.executionArn);
    return response;
  } catch (error) {
    console.error('Failed to start execution:', error);
    throw error;
  }
}

// 3. Describe Execution (Get status)
async function describeExecution(executionArn) {
  const command = new DescribeExecutionCommand({
    executionArn: executionArn
  });
  
  const response = await sfnClient.send(command);
  
  return {
    name: response.name,
    status: response.status,
    startDate: response.startDate,
    stopDate: response.stopDate,
    input: JSON.parse(response.input),
    output: response.output ? JSON.parse(response.output) : null,
    error: response.error,
    cause: response.cause
  };
}

// 4. Get Execution History
async function getExecutionHistory(executionArn) {
  const command = new GetExecutionHistoryCommand({
    executionArn: executionArn,
    maxResults: 100,
    reverseOrder: false
  });
  
  const response = await sfnClient.send(command);
  
  return response.events.map(event => ({
    timestamp: event.timestamp,
    type: event.type,
    id: event.id,
    details: event
  }));
}

// 5. Stop Execution
async function stopExecution(executionArn, reason) {
  const command = new StopExecutionCommand({
    executionArn: executionArn,
    error: 'ExecutionStopped',
    cause: reason
  });
  
  await sfnClient.send(command);
  console.log('Execution stopped');
}

// 6. List Executions
async function listExecutions(stateMachineArn, statusFilter) {
  const command = new ListExecutionsCommand({
    stateMachineArn: stateMachineArn,
    statusFilter: statusFilter,  // 'RUNNING', 'SUCCEEDED', 'FAILED', etc.
    maxResults: 100
  });
  
  const response = await sfnClient.send(command);
  
  return response.executions.map(exec => ({
    executionArn: exec.executionArn,
    name: exec.name,
    status: exec.status,
    startDate: exec.startDate,
    stopDate: exec.stopDate
  }));
}

// Usage
const orderInput = {
  orderId: 'order-12345',
  customerId: 'customer-67890',
  items: [
    { productId: 'prod-001', quantity: 2, price: 29.99 },
    { productId: 'prod-002', quantity: 1, price: 49.99 }
  ],
  totalAmount: 109.97
};

const execution = await startExecution(
  'arn:aws:states:us-east-1:123456789012:stateMachine:OrderProcessing',
  orderInput
);

// Monitor execution
const status = await describeExecution(execution.executionArn);
console.log('Execution status:', status);
```

### Best Practices:

1. ✅ Use error handling (Retry and Catch)
2. ✅ Implement idempotency in Lambda functions
3. ✅ Use Pass states for data transformation
4. ✅ Enable CloudWatch logging
5. ✅ Use Map state for parallel processing
6. ✅ Set appropriate timeouts
7. ✅ Use Choice states for conditional logic
8. ✅ Monitor execution metrics
9. ✅ Use Express workflows for high-volume
10. ✅ Version your state machines

---

## Key Takeaways

### Question 13 (ECS/EKS):
- Container orchestration on AWS
- ECS for AWS-native, EKS for Kubernetes
- Fargate for serverless containers
- Task definitions define container configuration
- Auto-scaling and load balancing built-in

### Question 14 (API Gateway):
- Fully managed API service
- REST, HTTP, and WebSocket APIs
- Lambda proxy integration for serverless backends
- Built-in authorization, throttling, caching
- Usage plans and API keys for monetization

### Question 15 (Step Functions):
- Workflow orchestration service
- Visual workflow designer
- Built-in error handling and retry logic
- Parallel and conditional execution
- Integrates with 200+ AWS services
