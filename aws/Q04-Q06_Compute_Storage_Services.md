# AWS Interview Questions: Compute & Storage Services (Q4-Q6)

## Question 4: Explain AWS Lambda and Serverless Computing

### 📋 Answer

**AWS Lambda** is a serverless compute service that lets you run code without provisioning or managing servers. You pay only for the compute time you consume.

### Key Concepts:

#### 1. **How Lambda Works**
```
Event Source → Lambda Function → Response
    ↓              ↓                ↓
API Gateway    Execute Code    Return Result
S3 Upload      Auto Scale      Update Database
CloudWatch     Pay per use     Send Notification
```

#### 2. **Lambda Features**

- **Automatic Scaling**: Handles 1 to 1000s of concurrent executions
- **Pay Per Use**: Billed in 1ms increments
- **Multiple Languages**: Node.js, Python, Java, Go, .NET, Ruby
- **Memory**: 128 MB to 10,240 MB
- **Timeout**: Max 15 minutes per execution
- **Trigger Sources**: 200+ AWS services

### Complete Lambda Implementation:

```javascript
// lambda-function.js - AWS Lambda Handler
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, PutCommand, GetCommand } from '@aws-sdk/lib-dynamodb';
import { S3Client, GetObjectCommand } from '@aws-sdk/client-s3';

const dynamoClient = new DynamoDBClient({ region: 'us-east-1' });
const docClient = DynamoDBDocumentClient.from(dynamoClient);
const s3Client = new S3Client({ region: 'us-east-1' });

// Lambda Handler - API Gateway Event
export const handler = async (event) => {
  console.log('Received event:', JSON.stringify(event, null, 2));
  
  try {
    // Parse request
    const httpMethod = event.httpMethod;
    const path = event.path;
    const body = event.body ? JSON.parse(event.body) : null;
    
    let response;
    
    // Route handling
    switch (`${httpMethod} ${path}`) {
      case 'POST /users':
        response = await createUser(body);
        break;
      case 'GET /users':
        response = await getUser(event.queryStringParameters?.id);
        break;
      default:
        response = { statusCode: 404, message: 'Route not found' };
    }
    
    // Return API Gateway response
    return {
      statusCode: response.statusCode || 200,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
      },
      body: JSON.stringify(response)
    };
    
  } catch (error) {
    console.error('Error:', error);
    
    return {
      statusCode: 500,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
      },
      body: JSON.stringify({
        error: 'Internal Server Error',
        message: error.message
      })
    };
  }
};

// Create User Function
async function createUser(userData) {
  const user = {
    id: generateId(),
    name: userData.name,
    email: userData.email,
    createdAt: new Date().toISOString()
  };
  
  const command = new PutCommand({
    TableName: 'Users',
    Item: user
  });
  
  await docClient.send(command);
  
  return {
    statusCode: 201,
    message: 'User created successfully',
    user
  };
}

// Get User Function
async function getUser(userId) {
  const command = new GetCommand({
    TableName: 'Users',
    Key: { id: userId }
  });
  
  const result = await docClient.send(command);
  
  if (!result.Item) {
    return {
      statusCode: 404,
      message: 'User not found'
    };
  }
  
  return {
    statusCode: 200,
    user: result.Item
  };
}

function generateId() {
  return `user_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
}

// S3 Event Handler - Process uploaded files
export const s3EventHandler = async (event) => {
  console.log('S3 Event received:', JSON.stringify(event, null, 2));
  
  for (const record of event.Records) {
    const bucket = record.s3.bucket.name;
    const key = decodeURIComponent(record.s3.object.key.replace(/\+/g, ' '));
    
    console.log(`Processing file: ${key} from bucket: ${bucket}`);
    
    try {
      // Get file from S3
      const getCommand = new GetObjectCommand({ Bucket: bucket, Key: key });
      const s3Object = await s3Client.send(getCommand);
      
      // Convert stream to string
      const fileContent = await streamToString(s3Object.Body);
      
      // Process file (example: parse JSON and save to DynamoDB)
      const data = JSON.parse(fileContent);
      
      const putCommand = new PutCommand({
        TableName: 'ProcessedFiles',
        Item: {
          id: generateId(),
          bucket,
          key,
          data,
          processedAt: new Date().toISOString()
        }
      });
      
      await docClient.send(putCommand);
      
      console.log(`Successfully processed: ${key}`);
      
    } catch (error) {
      console.error(`Error processing ${key}:`, error);
      throw error;
    }
  }
  
  return { statusCode: 200, message: 'Files processed successfully' };
};

// Helper: Convert stream to string
async function streamToString(stream) {
  const chunks = [];
  for await (const chunk of stream) {
    chunks.push(chunk);
  }
  return Buffer.concat(chunks).toString('utf-8');
}

// CloudWatch Scheduled Event Handler
export const scheduledHandler = async (event) => {
  console.log('Scheduled event triggered:', event);
  
  // Cleanup old records
  const cutoffDate = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000).toISOString();
  
  console.log(`Cleaning up records older than ${cutoffDate}`);
  
  // Implementation would scan and delete old records
  return { statusCode: 200, message: 'Cleanup completed' };
};
```

### Deploy Lambda with AWS SDK:

```javascript
// deploy-lambda.js
import { 
  LambdaClient, 
  CreateFunctionCommand,
  UpdateFunctionCodeCommand,
  InvokeCommand,
  GetFunctionCommand
} from '@aws-sdk/client-lambda';
import { readFileSync } from 'fs';
import { zip } from 'zip-a-folder';

const lambdaClient = new LambdaClient({ region: 'us-east-1' });

// Create Lambda Function
async function createLambdaFunction() {
  // First, zip your code
  await zip('./src', './function.zip');
  
  const zipBuffer = readFileSync('./function.zip');
  
  const params = {
    FunctionName: 'MyAPIHandler',
    Runtime: 'nodejs18.x',
    Role: 'arn:aws:iam::123456789012:role/lambda-execution-role',
    Handler: 'index.handler',
    Code: {
      ZipFile: zipBuffer
    },
    Description: 'API Handler for user management',
    Timeout: 30,
    MemorySize: 512,
    Environment: {
      Variables: {
        'TABLE_NAME': 'Users',
        'REGION': 'us-east-1'
      }
    },
    Tags: {
      'Environment': 'Production',
      'Project': 'UserAPI'
    }
  };
  
  try {
    const command = new CreateFunctionCommand(params);
    const response = await lambdaClient.send(command);
    console.log('Function created:', response.FunctionArn);
    return response;
  } catch (error) {
    console.error('Failed to create function:', error);
    throw error;
  }
}

// Update Lambda Function Code
async function updateLambdaCode(functionName, zipFilePath) {
  const zipBuffer = readFileSync(zipFilePath);
  
  const command = new UpdateFunctionCodeCommand({
    FunctionName: functionName,
    ZipFile: zipBuffer
  });
  
  const response = await lambdaClient.send(command);
  console.log('Function updated:', response.FunctionArn);
  return response;
}

// Invoke Lambda Function
async function invokeLambda(functionName, payload) {
  const command = new InvokeCommand({
    FunctionName: functionName,
    InvocationType: 'RequestResponse', // or 'Event' for async
    Payload: JSON.stringify(payload)
  });
  
  try {
    const response = await lambdaClient.send(command);
    const result = JSON.parse(Buffer.from(response.Payload).toString());
    console.log('Invocation result:', result);
    return result;
  } catch (error) {
    console.error('Invocation failed:', error);
    throw error;
  }
}

// Get Function Configuration
async function getFunctionInfo(functionName) {
  const command = new GetFunctionCommand({ FunctionName: functionName });
  const response = await lambdaClient.send(command);
  
  return {
    functionArn: response.Configuration.FunctionArn,
    runtime: response.Configuration.Runtime,
    timeout: response.Configuration.Timeout,
    memorySize: response.Configuration.MemorySize,
    lastModified: response.Configuration.LastModified
  };
}

// Usage
// await createLambdaFunction();
// await updateLambdaCode('MyAPIHandler', './function.zip');
const result = await invokeLambda('MyAPIHandler', {
  httpMethod: 'POST',
  path: '/users',
  body: JSON.stringify({ name: 'John Doe', email: 'john@example.com' })
});
```

### Lambda Pricing Example:

```javascript
// Calculate Lambda costs
function calculateLambdaCost(options) {
  const {
    requestsPerMonth,
    avgDurationMs,
    memoryMB
  } = options;
  
  // Pricing (as of 2024)
  const requestCost = 0.20 / 1_000_000; // $0.20 per 1M requests
  const computeCostPerGBSecond = 0.0000166667; // $0.0000166667 per GB-second
  
  // Free tier
  const freeRequests = 1_000_000;
  const freeGBSeconds = 400_000;
  
  // Calculate billable requests
  const billableRequests = Math.max(0, requestsPerMonth - freeRequests);
  const requestsCost = billableRequests * requestCost;
  
  // Calculate compute cost
  const gbSeconds = (requestsPerMonth * avgDurationMs / 1000) * (memoryMB / 1024);
  const billableGBSeconds = Math.max(0, gbSeconds - freeGBSeconds);
  const computeCost = billableGBSeconds * computeCostPerGBSecond;
  
  const totalCost = requestsCost + computeCost;
  
  return {
    requestsCost: requestsCost.toFixed(4),
    computeCost: computeCost.toFixed(4),
    totalCost: totalCost.toFixed(4),
    gbSeconds: gbSeconds.toFixed(2)
  };
}

// Example calculation
const cost = calculateLambdaCost({
  requestsPerMonth: 5_000_000,
  avgDurationMs: 200,
  memoryMB: 512
});

console.log('Monthly Lambda Cost:', cost);
// Output: ~$1.47/month for 5M requests
```

### Use Cases:

1. **REST APIs**: Backend for web/mobile apps
2. **Data Processing**: ETL, file transformation
3. **Real-time Stream Processing**: Kinesis, DynamoDB Streams
4. **Scheduled Tasks**: Cron jobs, cleanups
5. **IoT Backend**: Process device data
6. **Image/Video Processing**: Thumbnails, transcoding
7. **Chatbots**: AI-powered assistants

---

## Question 5: What is Amazon RDS and how does it differ from EC2 databases?

### 📋 Answer

**Amazon RDS (Relational Database Service)** is a managed database service that makes it easy to set up, operate, and scale relational databases in the cloud.

### Supported Database Engines:

1. **Amazon Aurora** (MySQL/PostgreSQL compatible)
2. **MySQL**
3. **PostgreSQL**
4. **MariaDB**
5. **Oracle**
6. **SQL Server**

### RDS vs EC2 Database Comparison:

| Feature | RDS (Managed) | EC2 Database (Self-Managed) |
|---------|---------------|----------------------------|
| **Setup** | Automatic | Manual installation |
| **Patching** | Automatic | Manual |
| **Backups** | Automatic | Manual setup |
| **High Availability** | Multi-AZ with one click | Manual configuration |
| **Scaling** | Easy (vertical/read replicas) | Manual |
| **Monitoring** | Built-in CloudWatch | Manual setup |
| **Cost** | Higher (convenience) | Lower (more work) |
| **Customization** | Limited | Full control |
| **Performance Insights** | Included | Manual |

### Complete RDS Implementation:

```javascript
// rds-manager.js
import {
  RDSClient,
  CreateDBInstanceCommand,
  DescribeDBInstancesCommand,
  ModifyDBInstanceCommand,
  CreateDBSnapshotCommand,
  DeleteDBInstanceCommand
} from '@aws-sdk/client-rds';

const rdsClient = new RDSClient({ region: 'us-east-1' });

// Create RDS Instance
async function createRDSInstance(config) {
  const params = {
    DBInstanceIdentifier: config.identifier,
    DBInstanceClass: 'db.t3.micro', // Instance type
    Engine: 'postgres',
    EngineVersion: '15.3',
    MasterUsername: 'admin',
    MasterUserPassword: config.password,
    AllocatedStorage: 20, // GB
    StorageType: 'gp3', // General Purpose SSD
    StorageEncrypted: true,
    BackupRetentionPeriod: 7, // days
    PreferredBackupWindow: '03:00-04:00',
    PreferredMaintenanceWindow: 'sun:04:00-sun:05:00',
    MultiAZ: true, // High availability
    PubliclyAccessible: false,
    VpcSecurityGroupIds: [config.securityGroupId],
    DBSubnetGroupName: config.subnetGroup,
    EnableCloudwatchLogsExports: ['postgresql'],
    DeletionProtection: true,
    Tags: [
      { Key: 'Environment', Value: 'Production' },
      { Key: 'Application', Value: 'UserAPI' }
    ]
  };
  
  try {
    const command = new CreateDBInstanceCommand(params);
    const response = await rdsClient.send(command);
    console.log('RDS instance created:', response.DBInstance.DBInstanceIdentifier);
    return response.DBInstance;
  } catch (error) {
    console.error('Failed to create RDS instance:', error);
    throw error;
  }
}

// Get RDS Instance Details
async function getRDSInstance(identifier) {
  const command = new DescribeDBInstancesCommand({
    DBInstanceIdentifier: identifier
  });
  
  const response = await rdsClient.send(command);
  const instance = response.DBInstances[0];
  
  return {
    identifier: instance.DBInstanceIdentifier,
    status: instance.DBInstanceStatus,
    endpoint: instance.Endpoint?.Address,
    port: instance.Endpoint?.Port,
    engine: instance.Engine,
    engineVersion: instance.EngineVersion,
    instanceClass: instance.DBInstanceClass,
    storage: instance.AllocatedStorage,
    multiAZ: instance.MultiAZ,
    availabilityZone: instance.AvailabilityZone
  };
}

// Scale RDS Instance
async function scaleRDSInstance(identifier, newInstanceClass, newStorage) {
  const command = new ModifyDBInstanceCommand({
    DBInstanceIdentifier: identifier,
    DBInstanceClass: newInstanceClass,
    AllocatedStorage: newStorage,
    ApplyImmediately: false // Apply during maintenance window
  });
  
  const response = await rdsClient.send(command);
  console.log('RDS instance modified:', response.DBInstance.DBInstanceIdentifier);
  return response.DBInstance;
}

// Create Snapshot
async function createSnapshot(dbIdentifier, snapshotIdentifier) {
  const command = new CreateDBSnapshotCommand({
    DBInstanceIdentifier: dbIdentifier,
    DBSnapshotIdentifier: snapshotIdentifier,
    Tags: [
      { Key: 'CreatedBy', Value: 'AutomatedBackup' },
      { Key: 'Date', Value: new Date().toISOString() }
    ]
  });
  
  const response = await rdsClient.send(command);
  console.log('Snapshot created:', response.DBSnapshot.DBSnapshotIdentifier);
  return response.DBSnapshot;
}

// Usage example
const dbConfig = {
  identifier: 'my-postgres-db',
  password: 'SecurePassword123!',
  securityGroupId: 'sg-0123456789abcdef0',
  subnetGroup: 'my-db-subnet-group'
};

// await createRDSInstance(dbConfig);

// Wait for instance to be available
setTimeout(async () => {
  const instance = await getRDSInstance('my-postgres-db');
  console.log('Instance details:', instance);
}, 300000); // Check after 5 minutes
```

### Connect to RDS from Application:

```javascript
// database-connection.js
import pg from 'pg';
const { Pool } = pg;

// RDS Connection Pool
const pool = new Pool({
  host: 'my-postgres-db.c9akciq32.us-east-1.rds.amazonaws.com',
  port: 5432,
  database: 'myapp',
  user: 'admin',
  password: process.env.DB_PASSWORD,
  max: 20, // Maximum pool size
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
  ssl: {
    rejectUnauthorized: true,
    ca: readFileSync('./rds-ca-bundle.pem').toString()
  }
});

// Database Operations
class UserRepository {
  // Create user
  async createUser(user) {
    const query = `
      INSERT INTO users (name, email, created_at)
      VALUES ($1, $2, NOW())
      RETURNING *
    `;
    
    try {
      const result = await pool.query(query, [user.name, user.email]);
      return result.rows[0];
    } catch (error) {
      console.error('Failed to create user:', error);
      throw error;
    }
  }
  
  // Get user by ID
  async getUserById(id) {
    const query = 'SELECT * FROM users WHERE id = $1';
    const result = await pool.query(query, [id]);
    return result.rows[0];
  }
  
  // Update user
  async updateUser(id, updates) {
    const query = `
      UPDATE users
      SET name = $1, email = $2, updated_at = NOW()
      WHERE id = $3
      RETURNING *
    `;
    
    const result = await pool.query(query, [updates.name, updates.email, id]);
    return result.rows[0];
  }
  
  // Transaction example
  async transferBalance(fromUserId, toUserId, amount) {
    const client = await pool.connect();
    
    try {
      await client.query('BEGIN');
      
      // Deduct from sender
      await client.query(
        'UPDATE accounts SET balance = balance - $1 WHERE user_id = $2',
        [amount, fromUserId]
      );
      
      // Add to receiver
      await client.query(
        'UPDATE accounts SET balance = balance + $1 WHERE user_id = $2',
        [amount, toUserId]
      );
      
      // Record transaction
      await client.query(
        'INSERT INTO transactions (from_user, to_user, amount, created_at) VALUES ($1, $2, $3, NOW())',
        [fromUserId, toUserId, amount]
      );
      
      await client.query('COMMIT');
      console.log('Transfer completed successfully');
      
    } catch (error) {
      await client.query('ROLLBACK');
      console.error('Transfer failed, rolled back:', error);
      throw error;
    } finally {
      client.release();
    }
  }
}

// Connection monitoring
pool.on('connect', () => {
  console.log('New database connection established');
});

pool.on('error', (err) => {
  console.error('Unexpected database error:', err);
});

export const userRepo = new UserRepository();
```

### RDS Features:

#### 1. **Automated Backups**
- Automatic daily backups
- Point-in-time recovery
- Retention: 0-35 days

#### 2. **Multi-AZ Deployment**
```
Primary DB (AZ-1) ←→ Synchronous Replication ←→ Standby DB (AZ-2)
       ↓
Automatic Failover (< 2 minutes)
```

#### 3. **Read Replicas**
```
Primary DB → Asynchronous Replication → Read Replica 1
                                      → Read Replica 2
                                      → Read Replica 3
```

Benefits:
- Offload read traffic
- Cross-region replication
- Can be promoted to primary

### Best Practices:

1. ✅ Enable Multi-AZ for production
2. ✅ Use Read Replicas for read-heavy workloads
3. ✅ Enable automated backups
4. ✅ Use encryption at rest and in transit
5. ✅ Configure security groups properly
6. ✅ Monitor with CloudWatch and Performance Insights
7. ✅ Use parameter groups for optimization
8. ✅ Implement connection pooling
9. ✅ Regular snapshot testing
10. ✅ Use IAM database authentication

---

## Question 6: Explain Amazon VPC (Virtual Private Cloud) and its components

### 📋 Answer

**Amazon VPC** lets you provision a logically isolated section of AWS Cloud where you can launch AWS resources in a virtual network that you define.

### VPC Architecture:

```
┌─────────────────── VPC (10.0.0.0/16) ────────────────────┐
│                                                            │
│  ┌──────────── Availability Zone A ─────────────┐        │
│  │                                                │        │
│  │  ┌─── Public Subnet (10.0.1.0/24) ────┐     │        │
│  │  │  - NAT Gateway                      │     │        │
│  │  │  - Load Balancer                    │     │        │
│  │  │  - Bastion Host                     │     │        │
│  │  └─────────────────────────────────────┘     │        │
│  │                                                │        │
│  │  ┌─── Private Subnet (10.0.2.0/24) ───┐     │        │
│  │  │  - EC2 Instances                    │     │        │
│  │  │  - RDS Primary                      │     │        │
│  │  └─────────────────────────────────────┘     │        │
│  └────────────────────────────────────────────────┘        │
│                                                            │
│  ┌──────────── Availability Zone B ─────────────┐        │
│  │                                                │        │
│  │  ┌─── Public Subnet (10.0.3.0/24) ────┐     │        │
│  │  │  - NAT Gateway                      │     │        │
│  │  └─────────────────────────────────────┘     │        │
│  │                                                │        │
│  │  ┌─── Private Subnet (10.0.4.0/24) ───┐     │        │
│  │  │  - EC2 Instances                    │     │        │
│  │  │  - RDS Standby                      │     │        │
│  │  └─────────────────────────────────────┘     │        │
│  └────────────────────────────────────────────────┘        │
│                                                            │
│  Internet Gateway ←→ Public Subnets                       │
│  VPN Gateway / Direct Connect                             │
└────────────────────────────────────────────────────────────┘
```

### VPC Components:

#### 1. **Subnets**
- Public Subnet: Has route to Internet Gateway
- Private Subnet: No direct internet access

#### 2. **Route Tables**
Controls network traffic routing

#### 3. **Internet Gateway (IGW)**
Allows communication between VPC and internet

#### 4. **NAT Gateway**
Enables private subnet instances to access internet

#### 5. **Security Groups**
Virtual firewall for instances (stateful)

#### 6. **Network ACLs**
Firewall for subnets (stateless)

### Create VPC with AWS SDK:

```javascript
// vpc-setup.js
import {
  EC2Client,
  CreateVpcCommand,
  CreateSubnetCommand,
  CreateInternetGatewayCommand,
  AttachInternetGatewayCommand,
  CreateRouteTableCommand,
  CreateRouteCommand,
  AssociateRouteTableCommand,
  CreateSecurityGroupCommand,
  AuthorizeSecurityGroupIngressCommand,
  CreateNatGatewayCommand,
  AllocateAddressCommand
} from '@aws-sdk/client-ec2';

const ec2Client = new EC2Client({ region: 'us-east-1' });

// Complete VPC Setup
async function createCompleteVPC() {
  try {
    // 1. Create VPC
    console.log('Creating VPC...');
    const vpcResponse = await ec2Client.send(new CreateVpcCommand({
      CidrBlock: '10.0.0.0/16',
      TagSpecifications: [{
        ResourceType: 'vpc',
        Tags: [{ Key: 'Name', Value: 'MyAppVPC' }]
      }]
    }));
    const vpcId = vpcResponse.Vpc.VpcId;
    console.log('VPC created:', vpcId);
    
    // 2. Create Internet Gateway
    console.log('Creating Internet Gateway...');
    const igwResponse = await ec2Client.send(new CreateInternetGatewayCommand({
      TagSpecifications: [{
        ResourceType: 'internet-gateway',
        Tags: [{ Key: 'Name', Value: 'MyAppIGW' }]
      }]
    }));
    const igwId = igwResponse.InternetGateway.InternetGatewayId;
    
    // Attach IGW to VPC
    await ec2Client.send(new AttachInternetGatewayCommand({
      InternetGatewayId: igwId,
      VpcId: vpcId
    }));
    console.log('Internet Gateway attached:', igwId);
    
    // 3. Create Public Subnet (AZ-1)
    console.log('Creating Public Subnet...');
    const publicSubnetResponse = await ec2Client.send(new CreateSubnetCommand({
      VpcId: vpcId,
      CidrBlock: '10.0.1.0/24',
      AvailabilityZone: 'us-east-1a',
      TagSpecifications: [{
        ResourceType: 'subnet',
        Tags: [{ Key: 'Name', Value: 'PublicSubnet-AZ1' }]
      }]
    }));
    const publicSubnetId = publicSubnetResponse.Subnet.SubnetId;
    console.log('Public Subnet created:', publicSubnetId);
    
    // 4. Create Private Subnet (AZ-1)
    console.log('Creating Private Subnet...');
    const privateSubnetResponse = await ec2Client.send(new CreateSubnetCommand({
      VpcId: vpcId,
      CidrBlock: '10.0.2.0/24',
      AvailabilityZone: 'us-east-1a',
      TagSpecifications: [{
        ResourceType: 'subnet',
        Tags: [{ Key: 'Name', Value: 'PrivateSubnet-AZ1' }]
      }]
    }));
    const privateSubnetId = privateSubnetResponse.Subnet.SubnetId;
    console.log('Private Subnet created:', privateSubnetId);
    
    // 5. Create Route Table for Public Subnet
    console.log('Creating Public Route Table...');
    const publicRTResponse = await ec2Client.send(new CreateRouteTableCommand({
      VpcId: vpcId,
      TagSpecifications: [{
        ResourceType: 'route-table',
        Tags: [{ Key: 'Name', Value: 'PublicRouteTable' }]
      }]
    }));
    const publicRouteTableId = publicRTResponse.RouteTable.RouteTableId;
    
    // Add route to Internet Gateway
    await ec2Client.send(new CreateRouteCommand({
      RouteTableId: publicRouteTableId,
      DestinationCidrBlock: '0.0.0.0/0',
      GatewayId: igwId
    }));
    
    // Associate with public subnet
    await ec2Client.send(new AssociateRouteTableCommand({
      RouteTableId: publicRouteTableId,
      SubnetId: publicSubnetId
    }));
    console.log('Public Route Table configured');
    
    // 6. Allocate Elastic IP for NAT Gateway
    console.log('Allocating Elastic IP...');
    const eipResponse = await ec2Client.send(new AllocateAddressCommand({
      Domain: 'vpc'
    }));
    const allocationId = eipResponse.AllocationId;
    
    // 7. Create NAT Gateway
    console.log('Creating NAT Gateway...');
    const natResponse = await ec2Client.send(new CreateNatGatewayCommand({
      SubnetId: publicSubnetId,
      AllocationId: allocationId,
      TagSpecifications: [{
        ResourceType: 'natgateway',
        Tags: [{ Key: 'Name', Value: 'MyAppNAT' }]
      }]
    }));
    const natGatewayId = natResponse.NatGateway.NatGatewayId;
    console.log('NAT Gateway created:', natGatewayId);
    
    // Wait for NAT Gateway to be available
    console.log('Waiting for NAT Gateway to be available...');
    await new Promise(resolve => setTimeout(resolve, 60000));
    
    // 8. Create Route Table for Private Subnet
    console.log('Creating Private Route Table...');
    const privateRTResponse = await ec2Client.send(new CreateRouteTableCommand({
      VpcId: vpcId,
      TagSpecifications: [{
        ResourceType: 'route-table',
        Tags: [{ Key: 'Name', Value: 'PrivateRouteTable' }]
      }]
    }));
    const privateRouteTableId = privateRTResponse.RouteTable.RouteTableId;
    
    // Add route to NAT Gateway
    await ec2Client.send(new CreateRouteCommand({
      RouteTableId: privateRouteTableId,
      DestinationCidrBlock: '0.0.0.0/0',
      NatGatewayId: natGatewayId
    }));
    
    // Associate with private subnet
    await ec2Client.send(new AssociateRouteTableCommand({
      RouteTableId: privateRouteTableId,
      SubnetId: privateSubnetId
    }));
    console.log('Private Route Table configured');
    
    // 9. Create Security Groups
    console.log('Creating Security Groups...');
    
    // Web Server Security Group
    const webSGResponse = await ec2Client.send(new CreateSecurityGroupCommand({
      GroupName: 'WebServerSG',
      Description: 'Security group for web servers',
      VpcId: vpcId
    }));
    const webSGId = webSGResponse.GroupId;
    
    // Allow HTTP
    await ec2Client.send(new AuthorizeSecurityGroupIngressCommand({
      GroupId: webSGId,
      IpPermissions: [{
        IpProtocol: 'tcp',
        FromPort: 80,
        ToPort: 80,
        IpRanges: [{ CidrIp: '0.0.0.0/0' }]
      }]
    }));
    
    // Allow HTTPS
    await ec2Client.send(new AuthorizeSecurityGroupIngressCommand({
      GroupId: webSGId,
      IpPermissions: [{
        IpProtocol: 'tcp',
        FromPort: 443,
        ToPort: 443,
        IpRanges: [{ CidrIp: '0.0.0.0/0' }]
      }]
    }));
    
    console.log('Web Server Security Group created:', webSGId);
    
    // Database Security Group
    const dbSGResponse = await ec2Client.send(new CreateSecurityGroupCommand({
      GroupName: 'DatabaseSG',
      Description: 'Security group for database',
      VpcId: vpcId
    }));
    const dbSGId = dbSGResponse.GroupId;
    
    // Allow PostgreSQL from Web Servers only
    await ec2Client.send(new AuthorizeSecurityGroupIngressCommand({
      GroupId: dbSGId,
      IpPermissions: [{
        IpProtocol: 'tcp',
        FromPort: 5432,
        ToPort: 5432,
        UserIdGroupPairs: [{ GroupId: webSGId }]
      }]
    }));
    
    console.log('Database Security Group created:', dbSGId);
    
    return {
      vpcId,
      igwId,
      publicSubnetId,
      privateSubnetId,
      natGatewayId,
      webSGId,
      dbSGId
    };
    
  } catch (error) {
    console.error('VPC setup failed:', error);
    throw error;
  }
}

// Execute VPC setup
// const vpcResources = await createCompleteVPC();
// console.log('VPC Setup Complete:', vpcResources);
```

### Security Group vs Network ACL:

```javascript
// Security Group Example (Stateful)
const securityGroupRules = {
  inbound: [
    { protocol: 'tcp', port: 80, source: '0.0.0.0/0', action: 'ALLOW' },
    { protocol: 'tcp', port: 443, source: '0.0.0.0/0', action: 'ALLOW' },
    { protocol: 'tcp', port: 22, source: '10.0.0.0/16', action: 'ALLOW' }
  ],
  // Outbound automatically allowed for established connections (stateful)
  outbound: [
    { protocol: 'all', port: 'all', destination: '0.0.0.0/0', action: 'ALLOW' }
  ]
};

// Network ACL Example (Stateless)
const networkACLRules = {
  inbound: [
    { rule: 100, protocol: 'tcp', port: 80, source: '0.0.0.0/0', action: 'ALLOW' },
    { rule: 110, protocol: 'tcp', port: 443, source: '0.0.0.0/0', action: 'ALLOW' },
    { rule: 120, protocol: 'tcp', port: 22, source: '10.0.0.0/16', action: 'ALLOW' },
    { rule: '*', protocol: 'all', port: 'all', source: '0.0.0.0/0', action: 'DENY' }
  ],
  // Must explicitly allow outbound (stateless)
  outbound: [
    { rule: 100, protocol: 'tcp', port: '1024-65535', destination: '0.0.0.0/0', action: 'ALLOW' },
    { rule: 110, protocol: 'tcp', port: 80, destination: '0.0.0.0/0', action: 'ALLOW' },
    { rule: 120, protocol: 'tcp', port: 443, destination: '0.0.0.0/0', action: 'ALLOW' },
    { rule: '*', protocol: 'all', port: 'all', destination: '0.0.0.0/0', action: 'DENY' }
  ]
};
```

### VPC Best Practices:

1. ✅ Use multiple Availability Zones
2. ✅ Separate public and private subnets
3. ✅ Use NAT Gateways for high availability
4. ✅ Implement least privilege security groups
5. ✅ Use VPC Flow Logs for monitoring
6. ✅ Enable VPC endpoints for AWS services
7. ✅ Use proper CIDR planning
8. ✅ Tag all resources consistently
9. ✅ Implement network segmentation
10. ✅ Regular security audits

---

## Key Takeaways

### Question 4 (Lambda):
- Serverless compute with automatic scaling
- Pay per use (1ms billing increments)
- 200+ event sources
- Max 15-minute execution time

### Question 5 (RDS):
- Managed relational database service
- Automatic backups, patching, scaling
- Multi-AZ for high availability
- Read replicas for performance

### Question 6 (VPC):
- Isolated virtual network in AWS
- Public/private subnets for security
- Internet Gateway for public access
- NAT Gateway for private subnet internet access
- Security Groups (stateful) vs NACLs (stateless)
