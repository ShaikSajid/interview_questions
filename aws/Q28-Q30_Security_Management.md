# AWS Interview Questions: Security & Secrets Management (Q28-Q30)

## Question 28: Explain AWS Systems Manager Parameter Store for configuration management

### 📋 Answer

**AWS Systems Manager Parameter Store** provides secure, hierarchical storage for configuration data management and secrets. It's free for standard parameters and integrates with AWS services.

### Parameter Store Features:

```
Parameter Store
├── Parameter Types
│   ├── String
│   ├── StringList
│   └── SecureString (KMS encrypted)
├── Parameter Tiers
│   ├── Standard (Free, 10,000 params, 4KB size)
│   └── Advanced ($0.05/param/month, 100,000 params, 8KB size)
├── Parameter Policies
│   ├── Expiration
│   ├── ExpirationNotification
│   └── NoChangeNotification
└── Features
    ├── Hierarchical naming (/app/db/host)
    ├── Versioning
    ├── Change notifications
    └── Parameter history
```

### Complete Parameter Store Implementation:

```javascript
// parameter-store.js
import {
  SSMClient,
  PutParameterCommand,
  GetParameterCommand,
  GetParametersCommand,
  GetParametersByPathCommand,
  DeleteParameterCommand,
  GetParameterHistoryCommand,
  DescribeParametersCommand,
  AddTagsToResourceCommand,
  LabelParameterVersionCommand
} from '@aws-sdk/client-ssm';

const ssmClient = new SSMClient({ region: 'us-east-1' });

class ParameterStoreService {
  // 1. Put Parameter
  async putParameter(name, value, type = 'String', options = {}) {
    const command = new PutParameterCommand({
      Name: name,
      Value: value,
      Type: type,  // String, StringList, SecureString
      Description: options.description,
      KeyId: options.kmsKeyId,  // For SecureString
      Overwrite: options.overwrite !== false,
      Tier: options.tier || 'Standard',  // Standard or Advanced
      Tags: options.tags || [],
      Policies: options.policies ? JSON.stringify(options.policies) : undefined
    });
    
    try {
      const response = await ssmClient.send(command);
      console.log(`Parameter saved: ${name} (version ${response.Version})`);
      return response;
    } catch (error) {
      console.error('Failed to put parameter:', error);
      throw error;
    }
  }
  
  // 2. Get Parameter
  async getParameter(name, decrypt = true) {
    const command = new GetParameterCommand({
      Name: name,
      WithDecryption: decrypt
    });
    
    try {
      const response = await ssmClient.send(command);
      return {
        name: response.Parameter.Name,
        value: response.Parameter.Value,
        type: response.Parameter.Type,
        version: response.Parameter.Version,
        lastModifiedDate: response.Parameter.LastModifiedDate,
        arn: response.Parameter.ARN
      };
    } catch (error) {
      if (error.name === 'ParameterNotFound') {
        return null;
      }
      throw error;
    }
  }
  
  // 3. Get Multiple Parameters
  async getParameters(names, decrypt = true) {
    const command = new GetParametersCommand({
      Names: names,
      WithDecryption: decrypt
    });
    
    const response = await ssmClient.send(command);
    
    const parameters = response.Parameters.reduce((acc, param) => {
      acc[param.Name] = param.Value;
      return acc;
    }, {});
    
    // Report invalid parameters
    if (response.InvalidParameters && response.InvalidParameters.length > 0) {
      console.warn('Invalid parameters:', response.InvalidParameters);
    }
    
    return parameters;
  }
  
  // 4. Get Parameters by Path (hierarchical)
  async getParametersByPath(path, decrypt = true, recursive = true) {
    let nextToken;
    const allParameters = {};
    
    do {
      const command = new GetParametersByPathCommand({
        Path: path,
        Recursive: recursive,
        WithDecryption: decrypt,
        NextToken: nextToken,
        MaxResults: 10
      });
      
      const response = await ssmClient.send(command);
      
      response.Parameters.forEach(param => {
        allParameters[param.Name] = param.Value;
      });
      
      nextToken = response.NextToken;
    } while (nextToken);
    
    return allParameters;
  }
  
  // 5. Delete Parameter
  async deleteParameter(name) {
    const command = new DeleteParameterCommand({
      Name: name
    });
    
    await ssmClient.send(command);
    console.log('Parameter deleted:', name);
  }
  
  // 6. Get Parameter History
  async getParameterHistory(name) {
    const command = new GetParameterHistoryCommand({
      Name: name,
      WithDecryption: true
    });
    
    const response = await ssmClient.send(command);
    
    return response.Parameters.map(param => ({
      version: param.Version,
      value: param.Value,
      lastModifiedDate: param.LastModifiedDate,
      lastModifiedUser: param.LastModifiedUser
    }));
  }
  
  // 7. Add Tags
  async addTags(parameterName, tags) {
    const parameter = await this.getParameter(parameterName);
    
    const command = new AddTagsToResourceCommand({
      ResourceType: 'Parameter',
      ResourceId: parameter.name,
      Tags: Object.entries(tags).map(([Key, Value]) => ({ Key, Value }))
    });
    
    await ssmClient.send(command);
    console.log('Tags added to parameter:', parameterName);
  }
  
  // 8. Label Parameter Version
  async labelVersion(name, version, labels) {
    const command = new LabelParameterVersionCommand({
      Name: name,
      ParameterVersion: version,
      Labels: labels
    });
    
    await ssmClient.send(command);
    console.log(`Version ${version} labeled:`, labels);
  }
  
  // 9. Put Parameter with Expiration Policy
  async putParameterWithExpiration(name, value, expirationDays) {
    const expirationDate = new Date();
    expirationDate.setDate(expirationDate.getDate() + expirationDays);
    
    const policies = [
      {
        Type: 'Expiration',
        Version: '1.0',
        Attributes: {
          Timestamp: expirationDate.toISOString()
        }
      },
      {
        Type: 'ExpirationNotification',
        Version: '1.0',
        Attributes: {
          Before: '15',  // Notify 15 days before expiration
          Unit: 'Days'
        }
      }
    ];
    
    return await this.putParameter(name, value, 'SecureString', {
      policies: policies,
      tier: 'Advanced'
    });
  }
}

// Configuration Manager
class ConfigurationManager {
  constructor(appName, environment) {
    this.appName = appName;
    this.environment = environment;
    this.basePath = `/${appName}/${environment}`;
    this.paramStore = new ParameterStoreService();
    this.cache = new Map();
    this.cacheTTL = 5 * 60 * 1000;  // 5 minutes
  }
  
  // Get configuration value
  async get(key, useCache = true) {
    const paramName = `${this.basePath}/${key}`;
    
    if (useCache && this.cache.has(paramName)) {
      const cached = this.cache.get(paramName);
      if (Date.now() - cached.timestamp < this.cacheTTL) {
        return cached.value;
      }
    }
    
    const parameter = await this.paramStore.getParameter(paramName);
    
    if (parameter) {
      this.cache.set(paramName, {
        value: parameter.value,
        timestamp: Date.now()
      });
      return parameter.value;
    }
    
    return null;
  }
  
  // Set configuration value
  async set(key, value, isSecret = false) {
    const paramName = `${this.basePath}/${key}`;
    
    await this.paramStore.putParameter(
      paramName,
      value,
      isSecret ? 'SecureString' : 'String',
      {
        description: `${this.appName} configuration`,
        tags: [
          { Key: 'Application', Value: this.appName },
          { Key: 'Environment', Value: this.environment }
        ]
      }
    );
    
    // Invalidate cache
    this.cache.delete(paramName);
  }
  
  // Get all configuration
  async getAll() {
    return await this.paramStore.getParametersByPath(this.basePath);
  }
  
  // Database configuration
  async getDatabaseConfig() {
    const host = await this.get('database/host');
    const port = await this.get('database/port');
    const name = await this.get('database/name');
    const username = await this.get('database/username');
    const password = await this.get('database/password');
    
    return { host, port, name, username, password };
  }
  
  // API keys
  async getAPIKey(service) {
    return await this.get(`api-keys/${service}`);
  }
  
  // Feature flags
  async getFeatureFlag(featureName) {
    const value = await this.get(`features/${featureName}`);
    return value === 'true' || value === '1';
  }
  
  // Refresh cache
  invalidateCache(key) {
    if (key) {
      this.cache.delete(`${this.basePath}/${key}`);
    } else {
      this.cache.clear();
    }
  }
}

// Usage
const config = new ConfigurationManager('myapp', 'production');

// Set configuration
await config.set('database/host', 'db.example.com');
await config.set('database/port', '5432');
await config.set('database/name', 'myapp_db');
await config.set('database/username', 'admin');
await config.set('database/password', 'SecurePass123!', true);  // Encrypted

// Set API keys
await config.set('api-keys/stripe', 'sk_live_xxx', true);
await config.set('api-keys/sendgrid', 'SG.xxx', true);

// Set feature flags
await config.set('features/new-dashboard', 'true');
await config.set('features/beta-feature', 'false');

// Get configuration
const dbConfig = await config.getDatabaseConfig();
console.log('Database config:', dbConfig);

const stripeKey = await config.getAPIKey('stripe');
console.log('Stripe key:', stripeKey);

const isDashboardEnabled = await config.getFeatureFlag('new-dashboard');
console.log('New dashboard enabled:', isDashboardEnabled);

// Get all configuration
const allConfig = await config.getAll();
console.log('All configuration:', allConfig);
```

### Express.js Integration:

```javascript
// config-middleware.js
import { ConfigurationManager } from './parameter-store.js';

const config = new ConfigurationManager('myapi', process.env.NODE_ENV || 'development');

// Initialize configuration at startup
export async function initializeConfig() {
  try {
    const dbConfig = await config.getDatabaseConfig();
    const apiKeys = {
      stripe: await config.getAPIKey('stripe'),
      sendgrid: await config.getAPIKey('sendgrid')
    };
    
    // Store in process.env or global config object
    global.appConfig = {
      database: dbConfig,
      apiKeys: apiKeys
    };
    
    console.log('Configuration loaded successfully');
  } catch (error) {
    console.error('Failed to load configuration:', error);
    process.exit(1);
  }
}

// Middleware to inject config
export function configMiddleware(req, res, next) {
  req.config = global.appConfig;
  next();
}

// Periodic cache refresh
setInterval(async () => {
  console.log('Refreshing configuration cache...');
  config.invalidateCache();
  await initializeConfig();
}, 5 * 60 * 1000);  // Every 5 minutes
```

### Best Practices:

1. ✅ Use hierarchical naming convention
2. ✅ Use SecureString for sensitive data
3. ✅ Implement caching to reduce API calls
4. ✅ Tag parameters for organization
5. ✅ Use parameter policies for lifecycle
6. ✅ Enable CloudTrail for audit
7. ✅ Use IAM policies for access control
8. ✅ Version parameters for rollback
9. ✅ Use Standard tier for cost optimization
10. ✅ Monitor with CloudWatch metrics

---

## Question 29: Explain AWS KMS (Key Management Service) for encryption

### 📋 Answer

**AWS KMS** is a managed service that makes it easy to create and control the encryption keys used to encrypt your data.

### KMS Concepts:

```
AWS KMS
├── Customer Master Keys (CMKs)
│   ├── AWS Managed Keys
│   ├── Customer Managed Keys
│   └── Custom Key Stores
├── Key Operations
│   ├── Encrypt/Decrypt
│   ├── GenerateDataKey
│   ├── Re-encrypt
│   └── Sign/Verify
├── Key Policies
│   ├── Resource-based
│   └── IAM policies
└── Features
    ├── Automatic rotation
    ├── Audit logging
    ├── Multi-region keys
    └── Import your own keys
```

### Complete KMS Implementation:

```javascript
// kms-service.js
import {
  KMSClient,
  CreateKeyCommand,
  EncryptCommand,
  DecryptCommand,
  GenerateDataKeyCommand,
  ReEncryptCommand,
  DescribeKeyCommand,
  EnableKeyRotationCommand,
  GetKeyRotationStatusCommand,
  ListKeysCommand,
  CreateAliasCommand,
  ListAliasesCommand
} from '@aws-sdk/client-kms';
import crypto from 'crypto';

const kmsClient = new KMSClient({ region: 'us-east-1' });

class KMSService {
  // 1. Create Customer Managed Key
  async createKey(description, tags = []) {
    const command = new CreateKeyCommand({
      Description: description,
      KeyUsage: 'ENCRYPT_DECRYPT',
      Origin: 'AWS_KMS',
      Tags: tags.map(({ Key, Value }) => ({ TagKey: Key, TagValue: Value }))
    });
    
    try {
      const response = await kmsClient.send(command);
      console.log('Key created:', response.KeyMetadata.KeyId);
      return response.KeyMetadata.KeyId;
    } catch (error) {
      console.error('Failed to create key:', error);
      throw error;
    }
  }
  
  // 2. Create Key Alias
  async createAlias(aliasName, keyId) {
    const command = new CreateAliasCommand({
      AliasName: `alias/${aliasName}`,
      TargetKeyId: keyId
    });
    
    await kmsClient.send(command);
    console.log('Alias created:', aliasName);
  }
  
  // 3. Encrypt Data
  async encrypt(keyId, plaintext) {
    const command = new EncryptCommand({
      KeyId: keyId,
      Plaintext: Buffer.from(plaintext)
    });
    
    const response = await kmsClient.send(command);
    return Buffer.from(response.CiphertextBlob).toString('base64');
  }
  
  // 4. Decrypt Data
  async decrypt(ciphertextBase64) {
    const command = new DecryptCommand({
      CiphertextBlob: Buffer.from(ciphertextBase64, 'base64')
    });
    
    const response = await kmsClient.send(command);
    return Buffer.from(response.Plaintext).toString('utf8');
  }
  
  // 5. Generate Data Key (for envelope encryption)
  async generateDataKey(keyId, keySpec = 'AES_256') {
    const command = new GenerateDataKeyCommand({
      KeyId: keyId,
      KeySpec: keySpec  // AES_128, AES_256
    });
    
    const response = await kmsClient.send(command);
    
    return {
      plaintextKey: response.Plaintext,
      encryptedKey: Buffer.from(response.CiphertextBlob).toString('base64'),
      keyId: response.KeyId
    };
  }
  
  // 6. Re-encrypt (change encryption key)
  async reEncrypt(ciphertextBase64, newKeyId) {
    const command = new ReEncryptCommand({
      CiphertextBlob: Buffer.from(ciphertextBase64, 'base64'),
      DestinationKeyId: newKeyId
    });
    
    const response = await kmsClient.send(command);
    return Buffer.from(response.CiphertextBlob).toString('base64');
  }
  
  // 7. Enable Automatic Key Rotation
  async enableKeyRotation(keyId) {
    const command = new EnableKeyRotationCommand({
      KeyId: keyId
    });
    
    await kmsClient.send(command);
    console.log('Key rotation enabled for:', keyId);
  }
  
  // 8. Get Key Rotation Status
  async getKeyRotationStatus(keyId) {
    const command = new GetKeyRotationStatusCommand({
      KeyId: keyId
    });
    
    const response = await kmsClient.send(command);
    return response.KeyRotationEnabled;
  }
  
  // 9. Describe Key
  async describeKey(keyId) {
    const command = new DescribeKeyCommand({
      KeyId: keyId
    });
    
    const response = await kmsClient.send(command);
    
    return {
      keyId: response.KeyMetadata.KeyId,
      arn: response.KeyMetadata.Arn,
      creationDate: response.KeyMetadata.CreationDate,
      enabled: response.KeyMetadata.Enabled,
      description: response.KeyMetadata.Description,
      keyState: response.KeyMetadata.KeyState
    };
  }
  
  // 10. List Keys
  async listKeys() {
    const command = new ListKeysCommand({});
    const response = await kmsClient.send(command);
    return response.Keys;
  }
}

// Envelope Encryption Helper
class EnvelopeEncryption {
  constructor(kmsKeyId) {
    this.kmsKeyId = kmsKeyId;
    this.kmsService = new KMSService();
  }
  
  // Encrypt large data using envelope encryption
  async encrypt(plaintext) {
    // Generate data key
    const { plaintextKey, encryptedKey } = await this.kmsService.generateDataKey(this.kmsKeyId);
    
    // Encrypt data with data key
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv('aes-256-cbc', plaintextKey, iv);
    
    let encrypted = cipher.update(plaintext, 'utf8', 'base64');
    encrypted += cipher.final('base64');
    
    return {
      encryptedData: encrypted,
      encryptedDataKey: encryptedKey,
      iv: iv.toString('base64'),
      algorithm: 'aes-256-cbc'
    };
  }
  
  // Decrypt envelope encrypted data
  async decrypt(envelope) {
    // Decrypt data key
    const plaintextKey = await this.kmsService.decrypt(envelope.encryptedDataKey);
    
    // Decrypt data
    const iv = Buffer.from(envelope.iv, 'base64');
    const decipher = crypto.createDecipheriv('aes-256-cbc', Buffer.from(plaintextKey, 'base64'), iv);
    
    let decrypted = decipher.update(envelope.encryptedData, 'base64', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
}

// Usage
const kms = new KMSService();

// Create key
const keyId = await kms.createKey('Application encryption key', [
  { Key: 'Application', Value: 'MyApp' },
  { Key: 'Environment', Value: 'Production' }
]);

// Create alias
await kms.createAlias('myapp-encryption-key', keyId);

// Enable rotation
await kms.enableKeyRotation(keyId);

// Small data encryption (< 4KB)
const encrypted = await kms.encrypt(keyId, 'Sensitive data');
console.log('Encrypted:', encrypted);

const decrypted = await kms.decrypt(encrypted);
console.log('Decrypted:', decrypted);

// Large data encryption (envelope encryption)
const envelopeEncryption = new EnvelopeEncryption(keyId);

const largeData = 'Very large sensitive data...'.repeat(1000);
const envelope = await envelopeEncryption.encrypt(largeData);
console.log('Envelope:', envelope);

const decryptedLarge = await envelopeEncryption.decrypt(envelope);
console.log('Decrypted large data:', decryptedLarge.substring(0, 50) + '...');
```

### S3 Encryption with KMS:

```javascript
// s3-kms-encryption.js
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';

const s3Client = new S3Client({ region: 'us-east-1' });

// Upload with KMS encryption
async function uploadWithKMS(bucket, key, data, kmsKeyId) {
  const command = new PutObjectCommand({
    Bucket: bucket,
    Key: key,
    Body: data,
    ServerSideEncryption: 'aws:kms',
    SSEKMSKeyId: kmsKeyId
  });
  
  await s3Client.send(command);
  console.log('File uploaded with KMS encryption');
}

// Download and decrypt automatically
async function downloadEncrypted(bucket, key) {
  const command = new GetObjectCommand({
    Bucket: bucket,
    Key: key
  });
  
  const response = await s3Client.send(command);
  const data = await response.Body.transformToString();
  
  console.log('File downloaded and decrypted');
  return data;
}

// Usage
await uploadWithKMS(
  'my-bucket',
  'sensitive-data.txt',
  'Confidential information',
  'alias/myapp-encryption-key'
);

const content = await downloadEncrypted('my-bucket', 'sensitive-data.txt');
```

### Best Practices:

1. ✅ Enable automatic key rotation
2. ✅ Use customer managed keys for control
3. ✅ Implement envelope encryption for large data
4. ✅ Use key policies for access control
5. ✅ Enable CloudTrail logging
6. ✅ Use aliases for key management
7. ✅ Monitor key usage with CloudWatch
8. ✅ Implement key deletion with waiting period
9. ✅ Use different keys for different purposes
10. ✅ Regularly audit key usage

---

## Question 30: Explain AWS IAM (Identity and Access Management) best practices

### 📋 Answer

**AWS IAM** enables you to manage access to AWS services and resources securely using users, groups, roles, and policies.

### IAM Components:

```
AWS IAM
├── Users
│   ├── Console access
│   ├── Programmatic access
│   └── MFA
├── Groups
│   └── User collections
├── Roles
│   ├── EC2 roles
│   ├── Lambda roles
│   ├── Cross-account roles
│   └── Service roles
├── Policies
│   ├── Managed policies
│   ├── Inline policies
│   └── Resource-based policies
└── Features
    ├── Least privilege
    ├── Temporary credentials
    ├── Audit with CloudTrail
    └── Conditions
```

### Complete IAM Implementation:

```javascript
// iam-service.js
import {
  IAMClient,
  CreateUserCommand,
  CreateGroupCommand,
  AddUserToGroupCommand,
  CreateRoleCommand,
  CreatePolicyCommand,
  AttachUserPolicyCommand,
  AttachRolePolicyCommand,
  GetUserCommand,
  ListUsersCommand,
  ListRolesCommand,
  PutRolePolicyCommand
} from '@aws-sdk/client-iam';

const iamClient = new IAMClient({ region: 'us-east-1' });

class IAMService {
  // 1. Create User
  async createUser(userName, tags = []) {
    const command = new CreateUserCommand({
      UserName: userName,
      Tags: tags
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
  
  // 2. Create Group
  async createGroup(groupName) {
    const command = new CreateGroupCommand({
      GroupName: groupName
    });
    
    const response = await iamClient.send(command);
    console.log('Group created:', response.Group.GroupName);
    return response.Group;
  }
  
  // 3. Add User to Group
  async addUserToGroup(userName, groupName) {
    const command = new AddUserToGroupCommand({
      UserName: userName,
      GroupName: groupName
    });
    
    await iamClient.send(command);
    console.log(`User ${userName} added to group ${groupName}`);
  }
  
  // 4. Create Role
  async createRole(roleName, assumeRolePolicyDocument, description) {
    const command = new CreateRoleCommand({
      RoleName: roleName,
      AssumeRolePolicyDocument: JSON.stringify(assumeRolePolicyDocument),
      Description: description
    });
    
    const response = await iamClient.send(command);
    console.log('Role created:', response.Role.RoleName);
    return response.Role;
  }
  
  // 5. Create Policy
  async createPolicy(policyName, policyDocument, description) {
    const command = new CreatePolicyCommand({
      PolicyName: policyName,
      PolicyDocument: JSON.stringify(policyDocument),
      Description: description
    });
    
    const response = await iamClient.send(command);
    console.log('Policy created:', response.Policy.PolicyName);
    return response.Policy;
  }
  
  // 6. Attach Policy to User
  async attachPolicyToUser(userName, policyArn) {
    const command = new AttachUserPolicyCommand({
      UserName: userName,
      PolicyArn: policyArn
    });
    
    await iamClient.send(command);
    console.log(`Policy attached to user ${userName}`);
  }
  
  // 7. Attach Policy to Role
  async attachPolicyToRole(roleName, policyArn) {
    const command = new AttachRolePolicyCommand({
      RoleName: roleName,
      PolicyArn: policyArn
    });
    
    await iamClient.send(command);
    console.log(`Policy attached to role ${roleName}`);
  }
  
  // 8. Put Inline Policy on Role
  async putRoleInlinePolicy(roleName, policyName, policyDocument) {
    const command = new PutRolePolicyCommand({
      RoleName: roleName,
      PolicyName: policyName,
      PolicyDocument: JSON.stringify(policyDocument)
    });
    
    await iamClient.send(command);
    console.log(`Inline policy ${policyName} added to role ${roleName}`);
  }
}

// Policy Templates
class IAMPolicyTemplates {
  // S3 Read-Only Policy
  static s3ReadOnly(bucketName) {
    return {
      Version: '2012-10-17',
      Statement: [
        {
          Effect: 'Allow',
          Action: [
            's3:GetObject',
            's3:ListBucket'
          ],
          Resource: [
            `arn:aws:s3:::${bucketName}`,
            `arn:aws:s3:::${bucketName}/*`
          ]
        }
      ]
    };
  }
  
  // S3 Full Access Policy
  static s3FullAccess(bucketName) {
    return {
      Version: '2012-10-17',
      Statement: [
        {
          Effect: 'Allow',
          Action: 's3:*',
          Resource: [
            `arn:aws:s3:::${bucketName}`,
            `arn:aws:s3:::${bucketName}/*`
          ]
        }
      ]
    };
  }
  
  // DynamoDB Table Access
  static dynamoDBTableAccess(tableName) {
    return {
      Version: '2012-10-17',
      Statement: [
        {
          Effect: 'Allow',
          Action: [
            'dynamodb:GetItem',
            'dynamodb:PutItem',
            'dynamodb:UpdateItem',
            'dynamodb:DeleteItem',
            'dynamodb:Query',
            'dynamodb:Scan'
          ],
          Resource: `arn:aws:dynamodb:*:*:table/${tableName}`
        }
      ]
    };
  }
  
  // Lambda Execution Role
  static lambdaExecutionRole() {
    return {
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
        }
      ]
    };
  }
  
  // EC2 Instance Role Trust Policy
  static ec2TrustPolicy() {
    return {
      Version: '2012-10-17',
      Statement: [
        {
          Effect: 'Allow',
          Principal: {
            Service: 'ec2.amazonaws.com'
          },
          Action: 'sts:AssumeRole'
        }
      ]
    };
  }
  
  // Lambda Trust Policy
  static lambdaTrustPolicy() {
    return {
      Version: '2012-10-17',
      Statement: [
        {
          Effect: 'Allow',
          Principal: {
            Service: 'lambda.amazonaws.com'
          },
          Action: 'sts:AssumeRole'
        }
      ]
    };
  }
  
  // Conditional Policy (MFA required)
  static mfaRequired() {
    return {
      Version: '2012-10-17',
      Statement: [
        {
          Effect: 'Allow',
          Action: '*',
          Resource: '*',
          Condition: {
            Bool: {
              'aws:MultiFactorAuthPresent': 'true'
            }
          }
        }
      ]
    };
  }
  
  // Time-based Access
  static timeBasedAccess(startTime, endTime) {
    return {
      Version: '2012-10-17',
      Statement: [
        {
          Effect: 'Allow',
          Action: '*',
          Resource: '*',
          Condition: {
            DateGreaterThan: {
              'aws:CurrentTime': startTime
            },
            DateLessThan: {
              'aws:CurrentTime': endTime
            }
          }
        }
      ]
    };
  }
  
  // IP-based Access
  static ipBasedAccess(allowedIPs) {
    return {
      Version: '2012-10-17',
      Statement: [
        {
          Effect: 'Allow',
          Action: '*',
          Resource: '*',
          Condition: {
            IpAddress: {
              'aws:SourceIp': allowedIPs
            }
          }
        }
      ]
    };
  }
}

// Usage Examples
const iam = new IAMService();

// Create developer user
const user = await iam.createUser('john.developer', [
  { Key: 'Department', Value: 'Engineering' },
  { Key: 'Team', Value: 'Backend' }
]);

// Create developer group
await iam.createGroup('Developers');

// Add user to group
await iam.addUserToGroup('john.developer', 'Developers');

// Create S3 read-only policy
const s3Policy = await iam.createPolicy(
  'S3ReadOnlyAccess',
  IAMPolicyTemplates.s3ReadOnly('my-app-bucket'),
  'Read-only access to application S3 bucket'
);

// Attach policy to user
await iam.attachPolicyToUser('john.developer', s3Policy.Arn);

// Create Lambda execution role
const lambdaRole = await iam.createRole(
  'MyLambdaExecutionRole',
  IAMPolicyTemplates.lambdaTrustPolicy(),
  'Role for Lambda function execution'
);

// Attach policies to Lambda role
await iam.attachPolicyToRole(
  'MyLambdaExecutionRole',
  'arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole'
);

await iam.putRoleInlinePolicy(
  'MyLambdaExecutionRole',
  'DynamoDBAccess',
  IAMPolicyTemplates.dynamoDBTableAccess('Users')
);
```

### IAM Best Practices Implementation:

```javascript
// iam-best-practices.js
class IAMBestPractices {
  // 1. Least Privilege Policy Generator
  static generateLeastPrivilegePolicy(actions, resources) {
    return {
      Version: '2012-10-17',
      Statement: [
        {
          Effect: 'Allow',
          Action: actions,  // Only necessary actions
          Resource: resources  // Only necessary resources
        }
      ]
    };
  }
  
  // 2. Password Policy
  static strictPasswordPolicy() {
    return {
      MinimumPasswordLength: 14,
      RequireSymbols: true,
      RequireNumbers: true,
      RequireUppercaseCharacters: true,
      RequireLowercaseCharacters: true,
      AllowUsersToChangePassword: true,
      ExpirePasswords: true,
      MaxPasswordAge: 90,
      PasswordReusePrevention: 24,
      HardExpiry: false
    };
  }
  
  // 3. Assume Role with Conditions
  static assumeRoleWithMFA(roleArn, accountId) {
    return {
      Version: '2012-10-17',
      Statement: [
        {
          Effect: 'Allow',
          Action: 'sts:AssumeRole',
          Resource: roleArn,
          Condition: {
            StringEquals: {
              'sts:ExternalId': accountId
            },
            Bool: {
              'aws:MultiFactorAuthPresent': 'true'
            }
          }
        }
      ]
    };
  }
  
  // 4. Service Control Policy (Organizations)
  static denyRootAccountAccess() {
    return {
      Version: '2012-10-17',
      Statement: [
        {
          Effect: 'Deny',
          Action: '*',
          Resource: '*',
          Condition: {
            StringLike: {
              'aws:PrincipalArn': 'arn:aws:iam::*:root'
            }
          }
        }
      ]
    };
  }
  
  // 5. Restrict Region Access
  static restrictToRegions(allowedRegions) {
    return {
      Version: '2012-10-17',
      Statement: [
        {
          Effect: 'Deny',
          Action: '*',
          Resource: '*',
          Condition: {
            StringNotEquals: {
              'aws:RequestedRegion': allowedRegions
            }
          }
        }
      ]
    };
  }
}
```

### Best Practices Summary:

1. ✅ **Enable MFA** for all users
2. ✅ **Use roles** instead of long-term credentials
3. ✅ **Apply least privilege** principle
4. ✅ **Use groups** to assign permissions
5. ✅ **Rotate credentials** regularly
6. ✅ **Use IAM Access Analyzer** for policy validation
7. ✅ **Enable CloudTrail** for audit logging
8. ✅ **Use service control policies** (SCPs) in Organizations
9. ✅ **Implement password policies**
10. ✅ **Remove unused credentials** and roles

---

## Key Takeaways

### Question 28 (Parameter Store):
- Hierarchical configuration storage
- Free for Standard tier
- SecureString with KMS encryption
- Parameter versioning and history
- Integration with CloudFormation
- Parameter policies for lifecycle

### Question 29 (KMS):
- Managed encryption key service
- Automatic key rotation
- Envelope encryption for large data
- Integration with AWS services
- CloudTrail logging for audit
- Multi-region keys for DR

### Question 30 (IAM):
- Identity and access management
- Least privilege principle
- Roles for temporary credentials
- MFA for enhanced security
- Policy conditions for fine-grained control
- CloudTrail for access auditing
