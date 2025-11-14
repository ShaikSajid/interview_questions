# AWS Interview Questions: Fundamentals & Basics (Q1-Q3)

## Question 1: What is AWS and what are its key services?

### 📋 Answer

**Amazon Web Services (AWS)** is a comprehensive cloud computing platform provided by Amazon that offers over 200+ fully-featured services from data centers globally.

### Key Service Categories:

#### 1. **Compute Services**
- **EC2 (Elastic Compute Cloud)**: Virtual servers in the cloud
- **Lambda**: Serverless compute (run code without managing servers)
- **ECS/EKS**: Container orchestration services
- **Elastic Beanstalk**: Platform as a Service (PaaS)

#### 2. **Storage Services**
- **S3 (Simple Storage Service)**: Object storage
- **EBS (Elastic Block Store)**: Block storage for EC2
- **EFS (Elastic File System)**: Managed file storage
- **Glacier**: Long-term archival storage

#### 3. **Database Services**
- **RDS (Relational Database Service)**: Managed SQL databases
- **DynamoDB**: NoSQL database
- **Aurora**: High-performance managed database
- **ElastiCache**: In-memory caching (Redis/Memcached)

#### 4. **Networking Services**
- **VPC (Virtual Private Cloud)**: Isolated network environment
- **Route 53**: DNS service
- **CloudFront**: Content Delivery Network (CDN)
- **API Gateway**: API management service

#### 5. **Security & Identity**
- **IAM (Identity and Access Management)**: User and permission management
- **KMS (Key Management Service)**: Encryption key management
- **Secrets Manager**: Store and manage secrets
- **WAF (Web Application Firewall)**: Protect web applications

#### 6. **Monitoring & Management**
- **CloudWatch**: Monitoring and observability
- **CloudTrail**: Audit and governance
- **Systems Manager**: Operational insights
- **Config**: Resource inventory and compliance

### Code Example: Basic AWS SDK Usage

```javascript
// AWS SDK v3 for Node.js
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { EC2Client, DescribeInstancesCommand } from '@aws-sdk/client-ec2';

// Initialize S3 Client
const s3Client = new S3Client({ region: 'us-east-1' });

// Upload file to S3
async function uploadToS3(bucketName, key, body) {
  const command = new PutObjectCommand({
    Bucket: bucketName,
    Key: key,
    Body: body,
    ContentType: 'application/json'
  });
  
  try {
    const response = await s3Client.send(command);
    console.log('Upload successful:', response);
    return response;
  } catch (error) {
    console.error('Upload failed:', error);
    throw error;
  }
}

// List EC2 Instances
const ec2Client = new EC2Client({ region: 'us-east-1' });

async function listEC2Instances() {
  const command = new DescribeInstancesCommand({});
  
  try {
    const response = await ec2Client.send(command);
    response.Reservations.forEach(reservation => {
      reservation.Instances.forEach(instance => {
        console.log(`Instance ID: ${instance.InstanceId}`);
        console.log(`State: ${instance.State.Name}`);
        console.log(`Type: ${instance.InstanceType}`);
      });
    });
    return response;
  } catch (error) {
    console.error('Failed to list instances:', error);
    throw error;
  }
}

// Usage
await uploadToS3('my-bucket', 'data.json', JSON.stringify({ foo: 'bar' }));
await listEC2Instances();
```

### Benefits of AWS:

1. **Scalability**: Scale up or down based on demand
2. **Cost-Effective**: Pay only for what you use
3. **Global Reach**: Data centers in multiple regions worldwide
4. **Security**: Enterprise-grade security and compliance
5. **Reliability**: High availability and fault tolerance
6. **Innovation**: Access to latest technologies (AI/ML, IoT, etc.)

---

## Question 2: What is EC2 and explain its key features?

### 📋 Answer

**Amazon EC2 (Elastic Compute Cloud)** is a web service that provides resizable compute capacity in the cloud. It allows you to launch virtual servers (instances) with various configurations.

### Key Features:

#### 1. **Instance Types**
Different instance types optimized for different use cases:

- **General Purpose** (T3, M5): Balanced compute, memory, and networking
- **Compute Optimized** (C5): High-performance processors
- **Memory Optimized** (R5, X1): Large memory for in-memory databases
- **Storage Optimized** (I3, D2): High sequential read/write to local storage
- **GPU Instances** (P3, G4): Machine learning and graphics

#### 2. **Pricing Models**

**On-Demand**: Pay by the hour/second
```javascript
// Cost: $0.0116/hour for t3.micro
const onDemandCost = 0.0116 * 24 * 30; // ~$8.35/month
```

**Reserved Instances**: 1 or 3-year commitment (up to 75% discount)
```javascript
// 1-year reserved: ~$4/month (52% savings)
const reservedCost = onDemandCost * 0.48;
```

**Spot Instances**: Bid for unused capacity (up to 90% discount)
```javascript
// Spot price: ~$0.0035/hour
const spotCost = 0.0035 * 24 * 30; // ~$2.52/month
```

**Savings Plans**: Flexible pricing model

#### 3. **Elastic IP Addresses**
Static IPv4 addresses for dynamic cloud computing

#### 4. **Security Groups**
Virtual firewall to control inbound/outbound traffic

#### 5. **Amazon Machine Images (AMI)**
Pre-configured templates for instances

### Practical Example: Launch EC2 Instance

```javascript
import { 
  EC2Client, 
  RunInstancesCommand,
  TerminateInstancesCommand,
  DescribeInstancesCommand 
} from '@aws-sdk/client-ec2';

const ec2Client = new EC2Client({ region: 'us-east-1' });

// Launch EC2 Instance
async function launchEC2Instance() {
  const params = {
    ImageId: 'ami-0c55b159cbfafe1f0', // Amazon Linux 2 AMI
    InstanceType: 't3.micro',
    MinCount: 1,
    MaxCount: 1,
    KeyName: 'my-key-pair',
    SecurityGroupIds: ['sg-0123456789abcdef0'],
    SubnetId: 'subnet-0bb1c79de3EXAMPLE',
    TagSpecifications: [
      {
        ResourceType: 'instance',
        Tags: [
          { Key: 'Name', Value: 'WebServer' },
          { Key: 'Environment', Value: 'Production' }
        ]
      }
    ],
    UserData: Buffer.from(`#!/bin/bash
      yum update -y
      yum install -y httpd
      systemctl start httpd
      systemctl enable httpd
      echo "<h1>Hello from EC2!</h1>" > /var/www/html/index.html
    `).toString('base64')
  };
  
  try {
    const command = new RunInstancesCommand(params);
    const response = await ec2Client.send(command);
    const instanceId = response.Instances[0].InstanceId;
    
    console.log('Instance launched:', instanceId);
    return instanceId;
  } catch (error) {
    console.error('Failed to launch instance:', error);
    throw error;
  }
}

// Get Instance Details
async function getInstanceDetails(instanceId) {
  const command = new DescribeInstancesCommand({
    InstanceIds: [instanceId]
  });
  
  const response = await ec2Client.send(command);
  const instance = response.Reservations[0].Instances[0];
  
  return {
    instanceId: instance.InstanceId,
    state: instance.State.Name,
    publicIp: instance.PublicIpAddress,
    privateIp: instance.PrivateIpAddress,
    instanceType: instance.InstanceType,
    launchTime: instance.LaunchTime
  };
}

// Terminate Instance
async function terminateInstance(instanceId) {
  const command = new TerminateInstancesCommand({
    InstanceIds: [instanceId]
  });
  
  const response = await ec2Client.send(command);
  console.log('Instance terminated:', instanceId);
  return response;
}

// Usage
const instanceId = await launchEC2Instance();
await new Promise(resolve => setTimeout(resolve, 30000)); // Wait 30s
const details = await getInstanceDetails(instanceId);
console.log('Instance Details:', details);
// await terminateInstance(instanceId);
```

### EC2 Instance Lifecycle:

```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌──────────┐
│ Pending │────>│ Running │────>│ Stopping │────>│ Stopped  │
└─────────┘     └────┬────┘     └─────────┘     └────┬─────┘
                     │                                │
                     │         ┌────────────┐         │
                     └────────>│Terminating │<────────┘
                               └─────┬──────┘
                                     │
                                     v
                               ┌──────────┐
                               │Terminated│
                               └──────────┘
```

### Best Practices:

1. ✅ Use appropriate instance types for workload
2. ✅ Enable detailed monitoring for production
3. ✅ Use Auto Scaling for high availability
4. ✅ Regular backup of EBS volumes
5. ✅ Use IAM roles instead of access keys
6. ✅ Tag resources for cost tracking
7. ✅ Use Spot Instances for fault-tolerant workloads

---

## Question 3: Explain S3 (Simple Storage Service) and its use cases

### 📋 Answer

**Amazon S3** is an object storage service that offers industry-leading scalability, data availability, security, and performance. It's designed to store and retrieve any amount of data from anywhere.

### Key Concepts:

#### 1. **S3 Structure**
```
AWS Account
  └── Bucket (globally unique name)
      └── Object (file)
          ├── Key (file path/name)
          ├── Value (file content)
          ├── Version ID
          └── Metadata
```

#### 2. **Storage Classes**

| Storage Class | Use Case | Retrieval Time | Cost |
|---------------|----------|----------------|------|
| **S3 Standard** | Frequently accessed data | Milliseconds | $$$ |
| **S3 Intelligent-Tiering** | Unknown/changing access patterns | Milliseconds | $$ |
| **S3 Standard-IA** | Infrequently accessed | Milliseconds | $$ |
| **S3 One Zone-IA** | Infrequent, non-critical | Milliseconds | $ |
| **S3 Glacier Instant** | Archive, instant access | Milliseconds | $ |
| **S3 Glacier Flexible** | Archive, 1-5 min retrieval | Minutes | $ |
| **S3 Glacier Deep Archive** | Long-term archive | Hours | ¢ |

#### 3. **Key Features**

- **Durability**: 99.999999999% (11 nines)
- **Availability**: 99.99%
- **Unlimited Storage**: No limit on data amount
- **Object Size**: 0 bytes to 5TB per object
- **Versioning**: Keep multiple versions of objects
- **Encryption**: Server-side and client-side
- **Access Control**: Bucket policies, ACLs, IAM

### Complete S3 Operations Example:

```javascript
import {
  S3Client,
  PutObjectCommand,
  GetObjectCommand,
  DeleteObjectCommand,
  ListObjectsV2Command,
  CopyObjectCommand,
  HeadObjectCommand
} from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import fs from 'fs';
import { Readable } from 'stream';

const s3Client = new S3Client({ region: 'us-east-1' });
const BUCKET_NAME = 'my-application-bucket';

// 1. Upload File to S3
async function uploadFile(key, filePath) {
  const fileContent = fs.readFileSync(filePath);
  
  const command = new PutObjectCommand({
    Bucket: BUCKET_NAME,
    Key: key,
    Body: fileContent,
    ContentType: 'application/json',
    Metadata: {
      'uploaded-by': 'nodejs-app',
      'upload-date': new Date().toISOString()
    },
    // Server-side encryption
    ServerSideEncryption: 'AES256',
    // Storage class
    StorageClass: 'STANDARD'
  });
  
  try {
    const response = await s3Client.send(command);
    console.log('File uploaded successfully:', response);
    return response;
  } catch (error) {
    console.error('Upload failed:', error);
    throw error;
  }
}

// 2. Download File from S3
async function downloadFile(key, downloadPath) {
  const command = new GetObjectCommand({
    Bucket: BUCKET_NAME,
    Key: key
  });
  
  try {
    const response = await s3Client.send(command);
    
    // Convert stream to buffer
    const chunks = [];
    for await (const chunk of response.Body) {
      chunks.push(chunk);
    }
    const buffer = Buffer.concat(chunks);
    
    // Save to file
    fs.writeFileSync(downloadPath, buffer);
    console.log('File downloaded successfully');
    
    return {
      content: buffer,
      contentType: response.ContentType,
      lastModified: response.LastModified,
      metadata: response.Metadata
    };
  } catch (error) {
    console.error('Download failed:', error);
    throw error;
  }
}

// 3. List Objects in Bucket
async function listObjects(prefix = '') {
  const command = new ListObjectsV2Command({
    Bucket: BUCKET_NAME,
    Prefix: prefix,
    MaxKeys: 1000
  });
  
  try {
    const response = await s3Client.send(command);
    
    const objects = response.Contents?.map(obj => ({
      key: obj.Key,
      size: obj.Size,
      lastModified: obj.LastModified,
      storageClass: obj.StorageClass
    })) || [];
    
    console.log(`Found ${objects.length} objects`);
    return objects;
  } catch (error) {
    console.error('List failed:', error);
    throw error;
  }
}

// 4. Delete Object
async function deleteObject(key) {
  const command = new DeleteObjectCommand({
    Bucket: BUCKET_NAME,
    Key: key
  });
  
  try {
    await s3Client.send(command);
    console.log('Object deleted:', key);
  } catch (error) {
    console.error('Delete failed:', error);
    throw error;
  }
}

// 5. Copy Object
async function copyObject(sourceKey, destinationKey) {
  const command = new CopyObjectCommand({
    Bucket: BUCKET_NAME,
    CopySource: `${BUCKET_NAME}/${sourceKey}`,
    Key: destinationKey
  });
  
  try {
    await s3Client.send(command);
    console.log('Object copied successfully');
  } catch (error) {
    console.error('Copy failed:', error);
    throw error;
  }
}

// 6. Get Object Metadata
async function getMetadata(key) {
  const command = new HeadObjectCommand({
    Bucket: BUCKET_NAME,
    Key: key
  });
  
  try {
    const response = await s3Client.send(command);
    return {
      contentType: response.ContentType,
      contentLength: response.ContentLength,
      lastModified: response.LastModified,
      etag: response.ETag,
      metadata: response.Metadata
    };
  } catch (error) {
    console.error('Failed to get metadata:', error);
    throw error;
  }
}

// 7. Generate Pre-signed URL (temporary access)
async function generatePresignedUrl(key, expiresIn = 3600) {
  const command = new GetObjectCommand({
    Bucket: BUCKET_NAME,
    Key: key
  });
  
  try {
    const url = await getSignedUrl(s3Client, command, { expiresIn });
    console.log('Pre-signed URL:', url);
    return url;
  } catch (error) {
    console.error('Failed to generate URL:', error);
    throw error;
  }
}

// 8. Upload Large File (Multipart Upload)
import { Upload } from '@aws-sdk/lib-storage';

async function uploadLargeFile(key, filePath) {
  const fileStream = fs.createReadStream(filePath);
  
  const upload = new Upload({
    client: s3Client,
    params: {
      Bucket: BUCKET_NAME,
      Key: key,
      Body: fileStream,
      ContentType: 'video/mp4'
    },
    // Multipart upload configuration
    queueSize: 4, // Concurrent uploads
    partSize: 5 * 1024 * 1024 // 5MB per part
  });
  
  upload.on('httpUploadProgress', (progress) => {
    console.log(`Uploaded: ${progress.loaded}/${progress.total} bytes`);
  });
  
  try {
    const response = await upload.done();
    console.log('Large file uploaded:', response);
    return response;
  } catch (error) {
    console.error('Upload failed:', error);
    throw error;
  }
}

// Usage Examples
async function demonstrateS3Operations() {
  try {
    // Upload
    await uploadFile('documents/report.json', './report.json');
    
    // List
    const objects = await listObjects('documents/');
    console.log('Objects:', objects);
    
    // Download
    await downloadFile('documents/report.json', './downloaded-report.json');
    
    // Get metadata
    const metadata = await getMetadata('documents/report.json');
    console.log('Metadata:', metadata);
    
    // Generate pre-signed URL (valid for 1 hour)
    const url = await generatePresignedUrl('documents/report.json', 3600);
    
    // Copy
    await copyObject('documents/report.json', 'backup/report.json');
    
    // Delete
    // await deleteObject('documents/report.json');
    
  } catch (error) {
    console.error('Operation failed:', error);
  }
}

demonstrateS3Operations();
```

### S3 Bucket Policy Example:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    },
    {
      "Sid": "DenyUnencryptedUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "AES256"
        }
      }
    }
  ]
}
```

### Common Use Cases:

1. **Static Website Hosting**
```javascript
// Enable static website hosting
import { PutBucketWebsiteCommand } from '@aws-sdk/client-s3';

const websiteConfig = {
  Bucket: BUCKET_NAME,
  WebsiteConfiguration: {
    IndexDocument: { Suffix: 'index.html' },
    ErrorDocument: { Key: 'error.html' }
  }
};

await s3Client.send(new PutBucketWebsiteCommand(websiteConfig));
```

2. **Data Lake Storage**
3. **Backup and Disaster Recovery**
4. **Application Data Storage**
5. **Media Storage (images, videos)**
6. **Log Storage**
7. **Big Data Analytics**

### S3 Lifecycle Policy:

```javascript
import { PutBucketLifecycleConfigurationCommand } from '@aws-sdk/client-s3';

const lifecycleConfig = {
  Bucket: BUCKET_NAME,
  LifecycleConfiguration: {
    Rules: [
      {
        Id: 'TransitionToGlacier',
        Status: 'Enabled',
        Prefix: 'logs/',
        Transitions: [
          {
            Days: 30,
            StorageClass: 'GLACIER'
          },
          {
            Days: 90,
            StorageClass: 'DEEP_ARCHIVE'
          }
        ],
        Expiration: {
          Days: 365
        }
      }
    ]
  }
};

await s3Client.send(new PutBucketLifecycleConfigurationCommand(lifecycleConfig));
```

### Best Practices:

1. ✅ Enable versioning for important data
2. ✅ Use lifecycle policies to optimize costs
3. ✅ Enable server-side encryption
4. ✅ Use IAM roles instead of access keys
5. ✅ Enable S3 access logging
6. ✅ Use CloudFront for content delivery
7. ✅ Implement proper bucket policies
8. ✅ Enable MFA delete for critical buckets
9. ✅ Use S3 Transfer Acceleration for large files
10. ✅ Monitor with CloudWatch metrics

---

## Key Takeaways

### Question 1 (AWS Overview):
- AWS offers 200+ services across compute, storage, database, networking
- Pay-as-you-go pricing model
- Global infrastructure with multiple regions

### Question 2 (EC2):
- Virtual servers with multiple instance types
- Various pricing models (On-Demand, Reserved, Spot)
- Key features: AMI, Security Groups, Elastic IP, Auto Scaling

### Question 3 (S3):
- Object storage with 99.999999999% durability
- Multiple storage classes for cost optimization
- Use cases: Static hosting, backups, data lakes, media storage
- Features: Versioning, encryption, lifecycle policies, pre-signed URLs
