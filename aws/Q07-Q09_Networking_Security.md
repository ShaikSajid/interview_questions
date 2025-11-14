# AWS Interview Questions: Networking & Security (Q7-Q9)

## Question 7: Explain AWS IAM (Identity and Access Management) in detail

### 📋 Answer

**AWS IAM** is a web service that helps you securely control access to AWS resources. You use IAM to control who is authenticated (signed in) and authorized (has permissions) to use resources.

### IAM Components:

```
IAM
├── Users (Individual identities)
├── Groups (Collections of users)
├── Roles (Assumed by AWS resources or federated users)
├── Policies (JSON documents defining permissions)
└── Identity Providers (Federation with external systems)
```

### Key Concepts:

#### 1. **IAM Users**
Individual identities with long-term credentials

#### 2. **IAM Groups**
Collections of users with shared permissions

#### 3. **IAM Roles**
Temporary credentials assumed by AWS services or users

#### 4. **IAM Policies**
JSON documents that define permissions

### Policy Structure:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "UniqueStatementId",
      "Effect": "Allow" | "Deny",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/username"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        }
      }
    }
  ]
}
```

### Complete IAM Implementation:

```javascript
// iam-manager.js
import {
  IAMClient,
  CreateUserCommand,
  CreateGroupCommand,
  AddUserToGroupCommand,
  CreatePolicyCommand,
  AttachUserPolicyCommand,
  AttachGroupPolicyCommand,
  CreateRoleCommand,
  AttachRolePolicyCommand,
  CreateAccessKeyCommand,
  ListUsersCommand,
  GetUserCommand,
  DeleteUserCommand,
  ListAttachedUserPoliciesCommand
} from '@aws-sdk/client-iam';

const iamClient = new IAMClient({ region: 'us-east-1' });

// Create IAM User
async function createIAMUser(username) {
  const command = new CreateUserCommand({
    UserName: username,
    Tags: [
      { Key: 'Department', Value: 'Engineering' },
      { Key: 'Environment', Value: 'Production' }
    ]
  });
  
  try {
    const response = await iamClient.send(command);
    console.log('User created:', response.User.UserName);
    return response.User;
  } catch (error) {
    console.error('Failed to create user:', error);
    throw error;
  }
}

// Create Access Keys for User
async function createAccessKey(username) {
  const command = new CreateAccessKeyCommand({
    UserName: username
  });
  
  const response = await iamClient.send(command);
  console.log('Access Key created for:', username);
  
  // ⚠️ IMPORTANT: Store these securely, they won't be shown again
  return {
    accessKeyId: response.AccessKey.AccessKeyId,
    secretAccessKey: response.AccessKey.SecretAccessKey
  };
}

// Create IAM Group
async function createIAMGroup(groupName) {
  const command = new CreateGroupCommand({
    GroupName: groupName
  });
  
  const response = await iamClient.send(command);
  console.log('Group created:', response.Group.GroupName);
  return response.Group;
}

// Add User to Group
async function addUserToGroup(username, groupName) {
  const command = new AddUserToGroupCommand({
    UserName: username,
    GroupName: groupName
  });
  
  await iamClient.send(command);
  console.log(`User ${username} added to group ${groupName}`);
}

// Create Custom IAM Policy
async function createCustomPolicy(policyName, policyDocument) {
  const command = new CreatePolicyCommand({
    PolicyName: policyName,
    PolicyDocument: JSON.stringify(policyDocument),
    Description: 'Custom policy created via SDK'
  });
  
  try {
    const response = await iamClient.send(command);
    console.log('Policy created:', response.Policy.Arn);
    return response.Policy;
  } catch (error) {
    console.error('Failed to create policy:', error);
    throw error;
  }
}

// Attach Policy to User
async function attachPolicyToUser(username, policyArn) {
  const command = new AttachUserPolicyCommand({
    UserName: username,
    PolicyArn: policyArn
  });
  
  await iamClient.send(command);
  console.log(`Policy attached to user ${username}`);
}

// Attach Policy to Group
async function attachPolicyToGroup(groupName, policyArn) {
  const command = new AttachGroupPolicyCommand({
    GroupName: groupName,
    PolicyArn: policyArn
  });
  
  await iamClient.send(command);
  console.log(`Policy attached to group ${groupName}`);
}

// Create IAM Role (for EC2 instance)
async function createEC2Role(roleName) {
  // Trust policy allowing EC2 to assume this role
  const trustPolicy = {
    Version: '2012-10-17',
    Statement: [{
      Effect: 'Allow',
      Principal: {
        Service: 'ec2.amazonaws.com'
      },
      Action: 'sts:AssumeRole'
    }]
  };
  
  const command = new CreateRoleCommand({
    RoleName: roleName,
    AssumeRolePolicyDocument: JSON.stringify(trustPolicy),
    Description: 'Role for EC2 instances to access S3',
    Tags: [
      { Key: 'Purpose', Value: 'EC2-S3-Access' }
    ]
  });
  
  try {
    const response = await iamClient.send(command);
    console.log('Role created:', response.Role.RoleName);
    return response.Role;
  } catch (error) {
    console.error('Failed to create role:', error);
    throw error;
  }
}

// Attach Policy to Role
async function attachPolicyToRole(roleName, policyArn) {
  const command = new AttachRolePolicyCommand({
    RoleName: roleName,
    PolicyArn: policyArn
  });
  
  await iamClient.send(command);
  console.log(`Policy attached to role ${roleName}`);
}

// List all users
async function listAllUsers() {
  const command = new ListUsersCommand({});
  const response = await iamClient.send(command);
  
  return response.Users.map(user => ({
    username: user.UserName,
    userId: user.UserId,
    arn: user.Arn,
    createdDate: user.CreateDate
  }));
}

// Get user details with attached policies
async function getUserDetails(username) {
  const userCommand = new GetUserCommand({ UserName: username });
  const policiesCommand = new ListAttachedUserPoliciesCommand({ UserName: username });
  
  const [userResponse, policiesResponse] = await Promise.all([
    iamClient.send(userCommand),
    iamClient.send(policiesCommand)
  ]);
  
  return {
    user: userResponse.User,
    attachedPolicies: policiesResponse.AttachedPolicies
  };
}
```

### Common IAM Policy Examples:

```javascript
// 1. S3 Full Access Policy
const s3FullAccessPolicy = {
  Version: '2012-10-17',
  Statement: [{
    Effect: 'Allow',
    Action: 's3:*',
    Resource: '*'
  }]
};

// 2. S3 Read-Only Access to Specific Bucket
const s3ReadOnlyPolicy = {
  Version: '2012-10-17',
  Statement: [{
    Effect: 'Allow',
    Action: [
      's3:GetObject',
      's3:ListBucket'
    ],
    Resource: [
      'arn:aws:s3:::my-bucket',
      'arn:aws:s3:::my-bucket/*'
    ]
  }]
};

// 3. EC2 Limited Access
const ec2LimitedPolicy = {
  Version: '2012-10-17',
  Statement: [
    {
      Effect: 'Allow',
      Action: [
        'ec2:DescribeInstances',
        'ec2:DescribeImages',
        'ec2:DescribeKeyPairs',
        'ec2:DescribeSecurityGroups'
      ],
      Resource: '*'
    },
    {
      Effect: 'Allow',
      Action: [
        'ec2:StartInstances',
        'ec2:StopInstances'
      ],
      Resource: 'arn:aws:ec2:us-east-1:123456789012:instance/*',
      Condition: {
        StringEquals: {
          'ec2:ResourceTag/Environment': 'Development'
        }
      }
    }
  ]
};

// 4. DynamoDB Access with Conditions
const dynamoDBPolicy = {
  Version: '2012-10-17',
  Statement: [{
    Effect: 'Allow',
    Action: [
      'dynamodb:GetItem',
      'dynamodb:PutItem',
      'dynamodb:UpdateItem',
      'dynamodb:Query'
    ],
    Resource: 'arn:aws:dynamodb:us-east-1:123456789012:table/Users',
    Condition: {
      'ForAllValues:StringEquals': {
        'dynamodb:LeadingKeys': ['${aws:username}']
      }
    }
  }]
};

// 5. Lambda Execution Role Policy
const lambdaExecutionPolicy = {
  Version: '2012-10-17',
  Statement: [
    {
      Effect: 'Allow',
      Action: [
        'logs:CreateLogGroup',
        'logs:CreateLogStream',
        'logs:PutLogEvents'
      ],
      Resource: 'arn:aws:logs:*:*:*'
    },
    {
      Effect: 'Allow',
      Action: [
        'dynamodb:GetItem',
        'dynamodb:PutItem',
        'dynamodb:Query',
        'dynamodb:Scan'
      ],
      Resource: 'arn:aws:dynamodb:us-east-1:123456789012:table/*'
    }
  ]
};

// 6. Deny All Access to Production Resources
const denyProductionPolicy = {
  Version: '2012-10-17',
  Statement: [{
    Effect: 'Deny',
    Action: '*',
    Resource: '*',
    Condition: {
      StringEquals: {
        'aws:ResourceTag/Environment': 'Production'
      }
    }
  }]
};

// 7. MFA Required for Sensitive Operations
const mfaRequiredPolicy = {
  Version: '2012-10-17',
  Statement: [
    {
      Sid: 'AllowAllActionsWithoutMFA',
      Effect: 'Allow',
      Action: [
        'ec2:Describe*',
        's3:List*',
        's3:GetObject'
      ],
      Resource: '*'
    },
    {
      Sid: 'DenyAllExceptListedIfNoMFA',
      Effect: 'Deny',
      NotAction: [
        'iam:CreateVirtualMFADevice',
        'iam:EnableMFADevice',
        'iam:ListMFADevices',
        'iam:ListUsers',
        'iam:ListVirtualMFADevices'
      ],
      Resource: '*',
      Condition: {
        BoolIfExists: {
          'aws:MultiFactorAuthPresent': 'false'
        }
      }
    }
  ]
};
```

### Complete Setup Example:

```javascript
// complete-iam-setup.js
async function setupDevelopmentTeam() {
  try {
    // 1. Create Developers Group
    const devsGroup = await createIAMGroup('Developers');
    
    // 2. Create custom policy for developers
    const devPolicy = {
      Version: '2012-10-17',
      Statement: [
        {
          Effect: 'Allow',
          Action: [
            'ec2:*',
            's3:*',
            'lambda:*',
            'dynamodb:*'
          ],
          Resource: '*',
          Condition: {
            StringEquals: {
              'aws:RequestedRegion': ['us-east-1', 'us-west-2']
            }
          }
        },
        {
          Effect: 'Deny',
          Action: [
            'ec2:TerminateInstances',
            's3:DeleteBucket'
          ],
          Resource: '*',
          Condition: {
            StringEquals: {
              'aws:ResourceTag/Environment': 'Production'
            }
          }
        }
      ]
    };
    
    const policy = await createCustomPolicy('DevelopersPolicy', devPolicy);
    
    // 3. Attach policy to group
    await attachPolicyToGroup('Developers', policy.Arn);
    
    // 4. Create developers
    const developers = ['john.doe', 'jane.smith', 'bob.wilson'];
    
    for (const dev of developers) {
      // Create user
      await createIAMUser(dev);
      
      // Add to group
      await addUserToGroup(dev, 'Developers');
      
      // Create access keys
      const credentials = await createAccessKey(dev);
      console.log(`Credentials for ${dev}:`, credentials);
    }
    
    // 5. Create EC2 role for application servers
    const ec2Role = await createEC2Role('AppServerRole');
    
    // Attach S3 read-only access
    await attachPolicyToRole(
      'AppServerRole',
      'arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess'
    );
    
    // Attach DynamoDB full access
    await attachPolicyToRole(
      'AppServerRole',
      'arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess'
    );
    
    console.log('Development team setup complete!');
    
  } catch (error) {
    console.error('Setup failed:', error);
    throw error;
  }
}

// Execute setup
// await setupDevelopmentTeam();
```

### IAM Best Practices:

1. ✅ **Enable MFA** for root account and privileged users
2. ✅ **Use IAM roles** instead of access keys for EC2
3. ✅ **Apply least privilege** principle
4. ✅ **Use groups** to assign permissions
5. ✅ **Rotate credentials** regularly
6. ✅ **Enable CloudTrail** for audit logging
7. ✅ **Use policy conditions** for fine-grained control
8. ✅ **Remove unused credentials** and users
9. ✅ **Use AWS Organizations** for multi-account management
10. ✅ **Monitor with IAM Access Analyzer**

---

## Question 8: What is Amazon CloudWatch and how do you use it for monitoring?

### 📋 Answer

**Amazon CloudWatch** is a monitoring and observability service that provides data and actionable insights for AWS resources, applications, and services.

### CloudWatch Components:

```
CloudWatch
├── Metrics (Performance data)
├── Logs (Application and system logs)
├── Alarms (Automated notifications)
├── Events/EventBridge (Event-driven automation)
├── Dashboards (Visualization)
└── Insights (Log analysis and queries)
```

### Key Features:

#### 1. **CloudWatch Metrics**
- Pre-defined metrics for AWS services
- Custom metrics from applications
- Resolution: 1 minute (standard) or 1 second (high-resolution)

#### 2. **CloudWatch Logs**
- Centralized log storage
- Real-time monitoring
- Log retention and archival

#### 3. **CloudWatch Alarms**
- Automated actions based on metrics
- SNS notifications, Auto Scaling, EC2 actions

#### 4. **CloudWatch Dashboards**
- Custom visualization of metrics

### Complete CloudWatch Implementation:

```javascript
// cloudwatch-manager.js
import {
  CloudWatchClient,
  PutMetricDataCommand,
  GetMetricStatisticsCommand,
  PutMetricAlarmCommand,
  DescribeAlarmsCommand,
  ListMetricsCommand
} from '@aws-sdk/client-cloudwatch';
import {
  CloudWatchLogsClient,
  CreateLogGroupCommand,
  CreateLogStreamCommand,
  PutLogEventsCommand,
  FilterLogEventsCommand,
  DescribeLogStreamsCommand
} from '@aws-sdk/client-cloudwatch-logs';

const cwClient = new CloudWatchClient({ region: 'us-east-1' });
const cwLogsClient = new CloudWatchLogsClient({ region: 'us-east-1' });

// 1. Send Custom Metrics
async function sendCustomMetric(metricData) {
  const command = new PutMetricDataCommand({
    Namespace: 'MyApp/API',
    MetricData: [
      {
        MetricName: metricData.name,
        Value: metricData.value,
        Unit: metricData.unit || 'None',
        Timestamp: new Date(),
        Dimensions: [
          {
            Name: 'Environment',
            Value: 'Production'
          },
          {
            Name: 'Service',
            Value: metricData.service
          }
        ]
      }
    ]
  });
  
  try {
    await cwClient.send(command);
    console.log('Metric sent:', metricData.name);
  } catch (error) {
    console.error('Failed to send metric:', error);
  }
}

// 2. Get Metric Statistics
async function getMetricStats(metricName, startTime, endTime) {
  const command = new GetMetricStatisticsCommand({
    Namespace: 'AWS/EC2',
    MetricName: metricName,
    StartTime: startTime,
    EndTime: endTime,
    Period: 300, // 5 minutes
    Statistics: ['Average', 'Maximum', 'Minimum'],
    Dimensions: [
      {
        Name: 'InstanceId',
        Value: 'i-1234567890abcdef0'
      }
    ]
  });
  
  const response = await cwClient.send(command);
  
  return response.Datapoints.map(dp => ({
    timestamp: dp.Timestamp,
    average: dp.Average,
    maximum: dp.Maximum,
    minimum: dp.Minimum
  }));
}

// 3. Create CloudWatch Alarm
async function createAlarm(config) {
  const command = new PutMetricAlarmCommand({
    AlarmName: config.alarmName,
    AlarmDescription: config.description,
    ActionsEnabled: true,
    AlarmActions: [config.snsTopicArn], // SNS topic for notifications
    MetricName: config.metricName,
    Namespace: config.namespace,
    Statistic: 'Average',
    Period: 300, // 5 minutes
    EvaluationPeriods: 2,
    Threshold: config.threshold,
    ComparisonOperator: config.comparisonOperator,
    Dimensions: config.dimensions,
    TreatMissingData: 'notBreaching'
  });
  
  try {
    await cwClient.send(command);
    console.log('Alarm created:', config.alarmName);
  } catch (error) {
    console.error('Failed to create alarm:', error);
    throw error;
  }
}

// 4. Create Log Group and Stream
async function createLogGroupAndStream(logGroupName, logStreamName) {
  try {
    // Create log group
    await cwLogsClient.send(new CreateLogGroupCommand({
      logGroupName: logGroupName
    }));
    console.log('Log group created:', logGroupName);
  } catch (error) {
    if (error.name !== 'ResourceAlreadyExistsException') {
      throw error;
    }
  }
  
  try {
    // Create log stream
    await cwLogsClient.send(new CreateLogStreamCommand({
      logGroupName: logGroupName,
      logStreamName: logStreamName
    }));
    console.log('Log stream created:', logStreamName);
  } catch (error) {
    if (error.name !== 'ResourceAlreadyExistsException') {
      throw error;
    }
  }
}

// 5. Send Logs to CloudWatch
let sequenceToken = null;

async function sendLogs(logGroupName, logStreamName, messages) {
  const logEvents = messages.map(msg => ({
    message: typeof msg === 'string' ? msg : JSON.stringify(msg),
    timestamp: Date.now()
  }));
  
  const params = {
    logGroupName: logGroupName,
    logStreamName: logStreamName,
    logEvents: logEvents
  };
  
  if (sequenceToken) {
    params.sequenceToken = sequenceToken;
  }
  
  try {
    const command = new PutLogEventsCommand(params);
    const response = await cwLogsClient.send(command);
    sequenceToken = response.nextSequenceToken;
    console.log('Logs sent successfully');
  } catch (error) {
    console.error('Failed to send logs:', error);
    throw error;
  }
}

// 6. Query Logs
async function queryLogs(logGroupName, filterPattern, startTime, endTime) {
  const command = new FilterLogEventsCommand({
    logGroupName: logGroupName,
    startTime: startTime,
    endTime: endTime,
    filterPattern: filterPattern
  });
  
  const response = await cwLogsClient.send(command);
  
  return response.events.map(event => ({
    timestamp: new Date(event.timestamp),
    message: event.message,
    logStreamName: event.logStreamName
  }));
}
```

### Application Monitoring Example:

```javascript
// app-monitoring.js
import express from 'express';

const app = express();
const LOG_GROUP = '/aws/application/myapp';
const LOG_STREAM = `app-${Date.now()}`;

// Initialize logging
await createLogGroupAndStream(LOG_GROUP, LOG_STREAM);

// Custom logger
class ApplicationLogger {
  async log(level, message, metadata = {}) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      level: level,
      message: message,
      ...metadata
    };
    
    console.log(logEntry);
    
    // Send to CloudWatch
    await sendLogs(LOG_GROUP, LOG_STREAM, [logEntry]);
  }
  
  async info(message, metadata) {
    await this.log('INFO', message, metadata);
  }
  
  async error(message, error, metadata) {
    await this.log('ERROR', message, {
      ...metadata,
      error: {
        name: error.name,
        message: error.message,
        stack: error.stack
      }
    });
  }
  
  async warn(message, metadata) {
    await this.log('WARN', message, metadata);
  }
}

const logger = new ApplicationLogger();

// Metrics tracker
class MetricsTracker {
  async trackRequest(req, res, duration) {
    // Track response time
    await sendCustomMetric({
      name: 'ResponseTime',
      value: duration,
      unit: 'Milliseconds',
      service: 'API'
    });
    
    // Track status code
    await sendCustomMetric({
      name: 'HTTPStatusCode',
      value: res.statusCode,
      unit: 'Count',
      service: 'API'
    });
    
    // Track request count
    await sendCustomMetric({
      name: 'RequestCount',
      value: 1,
      unit: 'Count',
      service: 'API'
    });
  }
  
  async trackError(error) {
    await sendCustomMetric({
      name: 'ErrorCount',
      value: 1,
      unit: 'Count',
      service: 'API'
    });
  }
  
  async trackDatabaseQuery(duration) {
    await sendCustomMetric({
      name: 'DatabaseQueryTime',
      value: duration,
      unit: 'Milliseconds',
      service: 'Database'
    });
  }
}

const metrics = new MetricsTracker();

// Middleware to track requests
app.use(async (req, res, next) => {
  const startTime = Date.now();
  
  // Log request
  await logger.info('Incoming request', {
    method: req.method,
    path: req.path,
    ip: req.ip
  });
  
  // Track response
  res.on('finish', async () => {
    const duration = Date.now() - startTime;
    
    await logger.info('Request completed', {
      method: req.method,
      path: req.path,
      statusCode: res.statusCode,
      duration: duration
    });
    
    await metrics.trackRequest(req, res, duration);
  });
  
  next();
});

// API endpoints
app.get('/api/users', async (req, res) => {
  try {
    const dbStartTime = Date.now();
    
    // Simulate database query
    const users = await getUsers();
    
    const dbDuration = Date.now() - dbStartTime;
    await metrics.trackDatabaseQuery(dbDuration);
    
    res.json(users);
    
  } catch (error) {
    await logger.error('Failed to get users', error);
    await metrics.trackError(error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Error handler
app.use(async (error, req, res, next) => {
  await logger.error('Unhandled error', error, {
    path: req.path,
    method: req.method
  });
  
  await metrics.trackError(error);
  
  res.status(500).json({ error: 'Internal server error' });
});

app.listen(3000, () => {
  logger.info('Server started', { port: 3000 });
});
```

### Setup Alarms:

```javascript
// setup-alarms.js
async function setupApplicationAlarms() {
  const snsTopicArn = 'arn:aws:sns:us-east-1:123456789012:AlertTopic';
  
  // 1. High CPU Alarm
  await createAlarm({
    alarmName: 'HighCPUUtilization',
    description: 'Alert when CPU exceeds 80%',
    namespace: 'AWS/EC2',
    metricName: 'CPUUtilization',
    threshold: 80,
    comparisonOperator: 'GreaterThanThreshold',
    snsTopicArn: snsTopicArn,
    dimensions: [
      { Name: 'InstanceId', Value: 'i-1234567890abcdef0' }
    ]
  });
  
  // 2. High Error Rate
  await createAlarm({
    alarmName: 'HighErrorRate',
    description: 'Alert when error rate exceeds 5%',
    namespace: 'MyApp/API',
    metricName: 'ErrorCount',
    threshold: 50,
    comparisonOperator: 'GreaterThanThreshold',
    snsTopicArn: snsTopicArn,
    dimensions: [
      { Name: 'Service', Value: 'API' }
    ]
  });
  
  // 3. Slow Response Time
  await createAlarm({
    alarmName: 'SlowResponseTime',
    description: 'Alert when response time exceeds 2 seconds',
    namespace: 'MyApp/API',
    metricName: 'ResponseTime',
    threshold: 2000,
    comparisonOperator: 'GreaterThanThreshold',
    snsTopicArn: snsTopicArn,
    dimensions: [
      { Name: 'Service', Value: 'API' }
    ]
  });
  
  // 4. Low Disk Space
  await createAlarm({
    alarmName: 'LowDiskSpace',
    description: 'Alert when disk usage exceeds 90%',
    namespace: 'System/Linux',
    metricName: 'DiskSpaceUtilization',
    threshold: 90,
    comparisonOperator: 'GreaterThanThreshold',
    snsTopicArn: snsTopicArn,
    dimensions: [
      { Name: 'InstanceId', Value: 'i-1234567890abcdef0' }
    ]
  });
  
  console.log('All alarms created successfully');
}

// await setupApplicationAlarms();
```

### CloudWatch Insights Queries:

```javascript
// Common log queries
const queries = {
  // Find all errors in last hour
  errors: `
    fields @timestamp, @message
    | filter @message like /ERROR/
    | sort @timestamp desc
    | limit 100
  `,
  
  // Average response time by endpoint
  responseTime: `
    fields @timestamp, duration, path
    | stats avg(duration) as avg_duration by path
    | sort avg_duration desc
  `,
  
  // Count requests by status code
  statusCodes: `
    fields @timestamp, statusCode
    | stats count() by statusCode
    | sort statusCode
  `,
  
  // Top 10 slowest requests
  slowRequests: `
    fields @timestamp, path, duration
    | sort duration desc
    | limit 10
  `,
  
  // Error rate over time
  errorRate: `
    fields @timestamp, level
    | stats count(level) as total, 
            sum(level = "ERROR") as errors by bin(5m)
    | fields total, errors, (errors / total * 100) as error_rate
  `
};
```

### Best Practices:

1. ✅ Set up alarms for critical metrics
2. ✅ Use structured logging (JSON format)
3. ✅ Tag all metrics with relevant dimensions
4. ✅ Create dashboards for key metrics
5. ✅ Set appropriate log retention policies
6. ✅ Use CloudWatch Logs Insights for analysis
7. ✅ Monitor billing with budget alarms
8. ✅ Use composite alarms for complex scenarios
9. ✅ Enable detailed monitoring for EC2
10. ✅ Integrate with AWS X-Ray for tracing

---

## Question 9: Explain Amazon Route 53 and DNS management in AWS

### 📋 Answer

**Amazon Route 53** is a highly available and scalable Domain Name System (DNS) web service designed to route end users to internet applications.

### Route 53 Features:

1. **Domain Registration**
2. **DNS Routing**
3. **Health Checking**
4. **Traffic Management**
5. **DNSSEC Support**
6. **Private DNS for VPC**

### Routing Policies:

```
Route 53 Routing Policies
├── Simple Routing (Single resource)
├── Weighted Routing (Traffic distribution by weight)
├── Latency Routing (Lowest network latency)
├── Failover Routing (Active-passive failover)
├── Geolocation Routing (Based on user location)
├── Geoproximity Routing (Based on resource location)
├── Multivalue Answer Routing (Multiple healthy resources)
└── IP-based Routing (Based on CIDR blocks)
```

### Complete Route 53 Implementation:

```javascript
// route53-manager.js
import {
  Route53Client,
  CreateHostedZoneCommand,
  ListHostedZonesCommand,
  ChangeResourceRecordSetsCommand,
  ListResourceRecordSetsCommand,
  GetHealthCheckCommand,
  CreateHealthCheckCommand,
  GetHostedZoneCommand
} from '@aws-sdk/client-route-53';

const route53Client = new Route53Client({ region: 'us-east-1' });

// 1. Create Hosted Zone
async function createHostedZone(domainName) {
  const command = new CreateHostedZoneCommand({
    Name: domainName,
    CallerReference: Date.now().toString(),
    HostedZoneConfig: {
      Comment: 'Hosted zone for application',
      PrivateZone: false
    }
  });
  
  try {
    const response = await route53Client.send(command);
    console.log('Hosted zone created:', response.HostedZone.Id);
    return response.HostedZone;
  } catch (error) {
    console.error('Failed to create hosted zone:', error);
    throw error;
  }
}

// 2. Create DNS Records
async function createDNSRecord(hostedZoneId, recordConfig) {
  const command = new ChangeResourceRecordSetsCommand({
    HostedZoneId: hostedZoneId,
    ChangeBatch: {
      Changes: [{
        Action: 'CREATE',
        ResourceRecordSet: recordConfig
      }]
    }
  });
  
  try {
    const response = await route53Client.send(command);
    console.log('DNS record created:', recordConfig.Name);
    return response;
  } catch (error) {
    console.error('Failed to create DNS record:', error);
    throw error;
  }
}

// 3. Create A Record (Simple Routing)
async function createARecord(hostedZoneId, domain, ipAddress) {
  return await createDNSRecord(hostedZoneId, {
    Name: domain,
    Type: 'A',
    TTL: 300,
    ResourceRecords: [{ Value: ipAddress }]
  });
}

// 4. Create CNAME Record
async function createCNAMERecord(hostedZoneId, subdomain, target) {
  return await createDNSRecord(hostedZoneId, {
    Name: subdomain,
    Type: 'CNAME',
    TTL: 300,
    ResourceRecords: [{ Value: target }]
  });
}

// 5. Create Weighted Routing Policy
async function createWeightedRecords(hostedZoneId, domain, targets) {
  const changes = targets.map(target => ({
    Action: 'CREATE',
    ResourceRecordSet: {
      Name: domain,
      Type: 'A',
      SetIdentifier: target.id,
      Weight: target.weight,
      TTL: 60,
      ResourceRecords: [{ Value: target.ip }]
    }
  }));
  
  const command = new ChangeResourceRecordSetsCommand({
    HostedZoneId: hostedZoneId,
    ChangeBatch: { Changes: changes }
  });
  
  const response = await route53Client.send(command);
  console.log('Weighted routing configured');
  return response;
}

// 6. Create Latency-Based Routing
async function createLatencyRecords(hostedZoneId, domain, regions) {
  const changes = regions.map(region => ({
    Action: 'CREATE',
    ResourceRecordSet: {
      Name: domain,
      Type: 'A',
      SetIdentifier: region.id,
      Region: region.awsRegion,
      TTL: 60,
      ResourceRecords: [{ Value: region.ip }]
    }
  }));
  
  const command = new ChangeResourceRecordSetsCommand({
    HostedZoneId: hostedZoneId,
    ChangeBatch: { Changes: changes }
  });
  
  const response = await route53Client.send(command);
  console.log('Latency-based routing configured');
  return response;
}

// 7. Create Health Check
async function createHealthCheck(config) {
  const command = new CreateHealthCheckCommand({
    HealthCheckConfig: {
      Type: config.type || 'HTTPS',
      ResourcePath: config.path || '/',
      FullyQualifiedDomainName: config.domain,
      Port: config.port || 443,
      RequestInterval: 30,
      FailureThreshold: 3
    },
    CallerReference: Date.now().toString()
  });
  
  try {
    const response = await route53Client.send(command);
    console.log('Health check created:', response.HealthCheck.Id);
    return response.HealthCheck;
  } catch (error) {
    console.error('Failed to create health check:', error);
    throw error;
  }
}

// 8. Create Failover Routing with Health Check
async function createFailoverRecords(hostedZoneId, domain, primary, secondary) {
  // Create health check for primary
  const healthCheck = await createHealthCheck({
    domain: primary.ip,
    path: '/health',
    type: 'HTTPS'
  });
  
  const changes = [
    {
      Action: 'CREATE',
      ResourceRecordSet: {
        Name: domain,
        Type: 'A',
        SetIdentifier: 'Primary',
        Failover: 'PRIMARY',
        TTL: 60,
        ResourceRecords: [{ Value: primary.ip }],
        HealthCheckId: healthCheck.Id
      }
    },
    {
      Action: 'CREATE',
      ResourceRecordSet: {
        Name: domain,
        Type: 'A',
        SetIdentifier: 'Secondary',
        Failover: 'SECONDARY',
        TTL: 60,
        ResourceRecords: [{ Value: secondary.ip }]
      }
    }
  ];
  
  const command = new ChangeResourceRecordSetsCommand({
    HostedZoneId: hostedZoneId,
    ChangeBatch: { Changes: changes }
  });
  
  const response = await route53Client.send(command);
  console.log('Failover routing configured');
  return response;
}

// 9. Create Alias Record (for AWS resources)
async function createAliasRecord(hostedZoneId, domain, targetConfig) {
  return await createDNSRecord(hostedZoneId, {
    Name: domain,
    Type: 'A',
    AliasTarget: {
      HostedZoneId: targetConfig.hostedZoneId,
      DNSName: targetConfig.dnsName,
      EvaluateTargetHealth: true
    }
  });
}

// 10. List all records in hosted zone
async function listRecords(hostedZoneId) {
  const command = new ListResourceRecordSetsCommand({
    HostedZoneId: hostedZoneId
  });
  
  const response = await route53Client.send(command);
  return response.ResourceRecordSets.map(record => ({
    name: record.Name,
    type: record.Type,
    ttl: record.TTL,
    values: record.ResourceRecords?.map(r => r.Value) || []
  }));
}
```

### Complete DNS Setup Example:

```javascript
// complete-dns-setup.js
async function setupProductionDNS() {
  const domain = 'myapp.com';
  
  try {
    // 1. Create hosted zone
    const hostedZone = await createHostedZone(domain);
    const zoneId = hostedZone.Id;
    
    console.log('Name servers:', hostedZone.DelegationSet.NameServers);
    
    // 2. Create apex record (www.myapp.com -> myapp.com)
    await createARecord(zoneId, domain, '54.123.45.67');
    
    // 3. Create www subdomain
    await createCNAMERecord(zoneId, `www.${domain}`, domain);
    
    // 4. Create API subdomain with weighted routing
    // 70% traffic to new servers, 30% to old
    await createWeightedRecords(zoneId, `api.${domain}`, [
      { id: 'new-servers', ip: '54.123.45.68', weight: 70 },
      { id: 'old-servers', ip: '54.123.45.69', weight: 30 }
    ]);
    
    // 5. Create multi-region setup with latency routing
    await createLatencyRecords(zoneId, `app.${domain}`, [
      { id: 'us-east', awsRegion: 'us-east-1', ip: '54.123.45.70' },
      { id: 'eu-west', awsRegion: 'eu-west-1', ip: '34.245.12.34' },
      { id: 'ap-south', awsRegion: 'ap-south-1', ip: '13.127.45.67' }
    ]);
    
    // 6. Create failover for critical service
    await createFailoverRecords(
      zoneId,
      `critical.${domain}`,
      { ip: '54.123.45.71' },  // Primary
      { ip: '54.123.45.72' }   // Secondary
    );
    
    // 7. Create alias record for CloudFront distribution
    await createAliasRecord(zoneId, domain, {
      hostedZoneId: 'Z2FDTNDATAQYW2', // CloudFront hosted zone ID
      dnsName: 'd1234567890abc.cloudfront.net'
    });
    
    // 8. Create MX records for email
    await createDNSRecord(zoneId, {
      Name: domain,
      Type: 'MX',
      TTL: 300,
      ResourceRecords: [
        { Value: '10 mail1.example.com' },
        { Value: '20 mail2.example.com' }
      ]
    });
    
    // 9. Create TXT record for domain verification
    await createDNSRecord(zoneId, {
      Name: domain,
      Type: 'TXT',
      TTL: 300,
      ResourceRecords: [
        { Value: '"v=spf1 include:_spf.google.com ~all"' }
      ]
    });
    
    console.log('DNS setup complete!');
    
    // List all records
    const records = await listRecords(zoneId);
    console.log('Created records:', records);
    
  } catch (error) {
    console.error('DNS setup failed:', error);
    throw error;
  }
}

// await setupProductionDNS();
```

### Traffic Flow Example:

```
User Request: www.myapp.com
         ↓
    Route 53 DNS Query
         ↓
   [Routing Policy Applied]
         ↓
    ┌─────────────────┐
    │ Latency Routing │
    └────────┬────────┘
             │
    ┌────────┴─────────────────┐
    │                          │
US User (50ms)          EU User (150ms)
    │                          │
    ↓                          ↓
US Server                  EU Server
54.123.45.70              34.245.12.34
```

### DNS Record Types:

```javascript
const recordTypes = {
  A: 'IPv4 address',
  AAAA: 'IPv6 address',
  CNAME: 'Canonical name (alias)',
  MX: 'Mail exchange',
  TXT: 'Text records',
  NS: 'Name server',
  SOA: 'Start of authority',
  SRV: 'Service locator',
  PTR: 'Pointer (reverse DNS)',
  CAA: 'Certificate authority authorization',
  ALIAS: 'AWS alias record (for AWS resources)'
};
```

### Best Practices:

1. ✅ Use health checks for failover
2. ✅ Implement multi-region routing for global apps
3. ✅ Use alias records for AWS resources
4. ✅ Set appropriate TTL values
5. ✅ Enable DNSSEC for security
6. ✅ Use traffic policies for complex routing
7. ✅ Monitor DNS queries with CloudWatch
8. ✅ Use private hosted zones for internal DNS
9. ✅ Implement geolocation routing for compliance
10. ✅ Regular testing of failover scenarios

---

## Key Takeaways

### Question 7 (IAM):
- Fine-grained access control with policies
- Users, Groups, Roles for identity management
- Always apply least privilege principle
- Use MFA for sensitive operations

### Question 8 (CloudWatch):
- Centralized monitoring and logging
- Custom metrics for application monitoring
- Alarms for automated notifications
- Logs Insights for powerful querying

### Question 9 (Route 53):
- Managed DNS service with multiple routing policies
- Health checks for automatic failover
- Latency-based routing for global applications
- Alias records for seamless AWS integration
