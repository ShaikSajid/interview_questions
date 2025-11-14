# IAM Security Setup - Serverless Order Processing Microservice

## 📋 Overview

This guide covers comprehensive IAM (Identity and Access Management) setup for the serverless microservice, following the **principle of least privilege**. We'll create specific roles and policies for each Lambda function, API Gateway, and other AWS services.

---

## 🎯 Security Principles

### Core Principles
1. **Least Privilege**: Grant only necessary permissions
2. **Separation of Duties**: Different roles for different functions
3. **Defense in Depth**: Multiple layers of security
4. **Audit and Monitor**: Track all IAM activities
5. **Encrypt Everything**: Data at rest and in transit

---

## 1️⃣ IAM Roles Architecture

### Role Hierarchy

```javascript
const iamRolesArchitecture = {
  lambdaRoles: {
    apiHandler: {
      purpose: 'Handle API Gateway requests',
      permissions: ['DynamoDB Read/Write', 'SQS SendMessage', 'CloudWatch Logs']
    },
    orderProcessor: {
      purpose: 'Process orders from SQS',
      permissions: ['SQS ReceiveMessage/DeleteMessage', 'DynamoDB Read/Write', 'SNS Publish']
    },
    inventoryUpdater: {
      purpose: 'Update inventory levels',
      permissions: ['DynamoDB UpdateItem', 'CloudWatch Logs']
    },
    notificationHandler: {
      purpose: 'Send notifications',
      permissions: ['SNS Publish', 'SES SendEmail', 'CloudWatch Logs']
    }
  },
  
  apiGatewayRole: {
    purpose: 'Invoke Lambda functions',
    permissions: ['Lambda InvokeFunction']
  },
  
  deploymentRole: {
    purpose: 'Deploy CloudFormation stacks',
    permissions: ['Full CloudFormation', 'Create/Update resources']
  }
};
```

---

## 2️⃣ Lambda Execution Roles

### 2.1 API Handler Role

Create file: `infrastructure/cloudformation/iam-roles.yaml`

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'IAM Roles and Policies for Serverless Order Processing'

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - staging
      - production

Resources:
  # ========================================
  # Lambda Role: API Handler
  # ========================================
  ApiHandlerRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub '${Environment}-api-handler-role'
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole
        - arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess
      Policies:
        - PolicyName: ApiHandlerPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              # DynamoDB Permissions
              - Effect: Allow
                Action:
                  - dynamodb:GetItem
                  - dynamodb:PutItem
                  - dynamodb:Query
                  - dynamodb:Scan
                Resource:
                  - !Sub 'arn:aws:dynamodb:${AWS::Region}:${AWS::AccountId}:table/${Environment}-orders'
                  - !Sub 'arn:aws:dynamodb:${AWS::Region}:${AWS::AccountId}:table/${Environment}-orders/index/*'
                  - !Sub 'arn:aws:dynamodb:${AWS::Region}:${AWS::AccountId}:table/${Environment}-inventory'
              
              # SQS Permissions
              - Effect: Allow
                Action:
                  - sqs:SendMessage
                  - sqs:GetQueueAttributes
                Resource:
                  - !Sub 'arn:aws:sqs:${AWS::Region}:${AWS::AccountId}:${Environment}-order-processing.fifo'
              
              # CloudWatch Logs
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource:
                  - !Sub 'arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/lambda/${Environment}-api-handler:*'
              
              # CloudWatch Metrics
              - Effect: Allow
                Action:
                  - cloudwatch:PutMetricData
                Resource: '*'
                Condition:
                  StringEquals:
                    cloudwatch:namespace: !Sub '${Environment}/OrderProcessing'
              
              # Secrets Manager (for sensitive config)
              - Effect: Allow
                Action:
                  - secretsmanager:GetSecretValue
                Resource:
                  - !Sub 'arn:aws:secretsmanager:${AWS::Region}:${AWS::AccountId}:secret:${Environment}/api/*'
              
              # KMS (for encryption)
              - Effect: Allow
                Action:
                  - kms:Decrypt
                  - kms:DescribeKey
                Resource:
                  - !Sub 'arn:aws:kms:${AWS::Region}:${AWS::AccountId}:key/*'
                Condition:
                  StringEquals:
                    kms:ViaService:
                      - !Sub 'dynamodb.${AWS::Region}.amazonaws.com'
                      - !Sub 'sqs.${AWS::Region}.amazonaws.com'
      Tags:
        - Key: Environment
          Value: !Ref Environment
        - Key: Service
          Value: order-processing

  # ========================================
  # Lambda Role: Order Processor
  # ========================================
  OrderProcessorRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub '${Environment}-order-processor-role'
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole
        - arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess
        - arn:aws:iam::aws:policy/service-role/AWSLambdaSQSQueueExecutionRole
      Policies:
        - PolicyName: OrderProcessorPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              # SQS Permissions (read from queue)
              - Effect: Allow
                Action:
                  - sqs:ReceiveMessage
                  - sqs:DeleteMessage
                  - sqs:GetQueueAttributes
                  - sqs:ChangeMessageVisibility
                Resource:
                  - !Sub 'arn:aws:sqs:${AWS::Region}:${AWS::AccountId}:${Environment}-order-processing.fifo'
              
              # DynamoDB Permissions
              - Effect: Allow
                Action:
                  - dynamodb:GetItem
                  - dynamodb:PutItem
                  - dynamodb:UpdateItem
                  - dynamodb:Query
                Resource:
                  - !Sub 'arn:aws:dynamodb:${AWS::Region}:${AWS::AccountId}:table/${Environment}-orders'
                  - !Sub 'arn:aws:dynamodb:${AWS::Region}:${AWS::AccountId}:table/${Environment}-inventory'
              
              # SNS Permissions (publish notifications)
              - Effect: Allow
                Action:
                  - sns:Publish
                Resource:
                  - !Sub 'arn:aws:sns:${AWS::Region}:${AWS::AccountId}:${Environment}-order-notifications'
              
              # CloudWatch Logs
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource:
                  - !Sub 'arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/lambda/${Environment}-order-processor:*'
      Tags:
        - Key: Environment
          Value: !Ref Environment

  # ========================================
  # Lambda Role: Inventory Updater
  # ========================================
  InventoryUpdaterRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub '${Environment}-inventory-updater-role'
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole
      Policies:
        - PolicyName: InventoryUpdaterPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              # DynamoDB Permissions (inventory table only)
              - Effect: Allow
                Action:
                  - dynamodb:GetItem
                  - dynamodb:UpdateItem
                  - dynamodb:ConditionCheckItem
                Resource:
                  - !Sub 'arn:aws:dynamodb:${AWS::Region}:${AWS::AccountId}:table/${Environment}-inventory'
              
              # CloudWatch Logs
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource:
                  - !Sub 'arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/lambda/${Environment}-inventory-updater:*'
      Tags:
        - Key: Environment
          Value: !Ref Environment

  # ========================================
  # Lambda Role: Notification Handler
  # ========================================
  NotificationHandlerRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub '${Environment}-notification-handler-role'
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole
      Policies:
        - PolicyName: NotificationHandlerPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              # SNS Permissions
              - Effect: Allow
                Action:
                  - sns:Publish
                  - sns:Subscribe
                  - sns:Unsubscribe
                Resource:
                  - !Sub 'arn:aws:sns:${AWS::Region}:${AWS::AccountId}:${Environment}-*'
              
              # SES Permissions (for email)
              - Effect: Allow
                Action:
                  - ses:SendEmail
                  - ses:SendRawEmail
                Resource: '*'
                Condition:
                  StringEquals:
                    ses:FromAddress: !Sub 'noreply@${Environment}.example.com'
              
              # CloudWatch Logs
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource:
                  - !Sub 'arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/lambda/${Environment}-notification-handler:*'
      Tags:
        - Key: Environment
          Value: !Ref Environment

  # ========================================
  # API Gateway Role
  # ========================================
  ApiGatewayRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub '${Environment}-api-gateway-role'
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: apigateway.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: ApiGatewayPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              # Lambda Invoke
              - Effect: Allow
                Action:
                  - lambda:InvokeFunction
                Resource:
                  - !GetAtt ApiHandlerRole.Arn
              
              # CloudWatch Logs
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource:
                  - !Sub 'arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/api-gateway/${Environment}-*'
      Tags:
        - Key: Environment
          Value: !Ref Environment

# ========================================
# Outputs
# ========================================
Outputs:
  ApiHandlerRoleArn:
    Description: ARN of API Handler Lambda Role
    Value: !GetAtt ApiHandlerRole.Arn
    Export:
      Name: !Sub '${Environment}-ApiHandlerRole-Arn'

  OrderProcessorRoleArn:
    Description: ARN of Order Processor Lambda Role
    Value: !GetAtt OrderProcessorRole.Arn
    Export:
      Name: !Sub '${Environment}-OrderProcessorRole-Arn'

  InventoryUpdaterRoleArn:
    Description: ARN of Inventory Updater Lambda Role
    Value: !GetAtt InventoryUpdaterRole.Arn
    Export:
      Name: !Sub '${Environment}-InventoryUpdaterRole-Arn'

  NotificationHandlerRoleArn:
    Description: ARN of Notification Handler Lambda Role
    Value: !GetAtt NotificationHandlerRole.Arn
    Export:
      Name: !Sub '${Environment}-NotificationHandlerRole-Arn'

  ApiGatewayRoleArn:
    Description: ARN of API Gateway Role
    Value: !GetAtt ApiGatewayRole.Arn
    Export:
      Name: !Sub '${Environment}-ApiGatewayRole-Arn'
```

---

## 3️⃣ KMS Encryption Setup

### KMS Key for Encryption

Create file: `infrastructure/cloudformation/kms-stack.yaml`

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'KMS Keys for Encryption'

Parameters:
  Environment:
    Type: String
    Default: dev

Resources:
  # ========================================
  # KMS Key for DynamoDB
  # ========================================
  DynamoDBKMSKey:
    Type: AWS::KMS::Key
    Properties:
      Description: !Sub 'KMS key for ${Environment} DynamoDB tables'
      KeyPolicy:
        Version: '2012-10-17'
        Statement:
          - Sid: Enable IAM User Permissions
            Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:root'
            Action: 'kms:*'
            Resource: '*'
          
          - Sid: Allow DynamoDB to use the key
            Effect: Allow
            Principal:
              Service: dynamodb.amazonaws.com
            Action:
              - kms:Decrypt
              - kms:DescribeKey
              - kms:CreateGrant
            Resource: '*'
          
          - Sid: Allow Lambda roles to use the key
            Effect: Allow
            Principal:
              AWS:
                - !Sub 'arn:aws:iam::${AWS::AccountId}:role/${Environment}-api-handler-role'
                - !Sub 'arn:aws:iam::${AWS::AccountId}:role/${Environment}-order-processor-role'
            Action:
              - kms:Decrypt
              - kms:DescribeKey
            Resource: '*'
      Tags:
        - Key: Environment
          Value: !Ref Environment
        - Key: Purpose
          Value: DynamoDB Encryption

  DynamoDBKMSKeyAlias:
    Type: AWS::KMS::Alias
    Properties:
      AliasName: !Sub 'alias/${Environment}-dynamodb'
      TargetKeyId: !Ref DynamoDBKMSKey

  # ========================================
  # KMS Key for SQS
  # ========================================
  SQSKMSKey:
    Type: AWS::KMS::Key
    Properties:
      Description: !Sub 'KMS key for ${Environment} SQS queues'
      KeyPolicy:
        Version: '2012-10-17'
        Statement:
          - Sid: Enable IAM User Permissions
            Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:root'
            Action: 'kms:*'
            Resource: '*'
          
          - Sid: Allow SQS to use the key
            Effect: Allow
            Principal:
              Service: sqs.amazonaws.com
            Action:
              - kms:Decrypt
              - kms:GenerateDataKey
            Resource: '*'
          
          - Sid: Allow Lambda to decrypt messages
            Effect: Allow
            Principal:
              AWS:
                - !Sub 'arn:aws:iam::${AWS::AccountId}:role/${Environment}-api-handler-role'
                - !Sub 'arn:aws:iam::${AWS::AccountId}:role/${Environment}-order-processor-role'
            Action:
              - kms:Decrypt
            Resource: '*'
      Tags:
        - Key: Environment
          Value: !Ref Environment

  SQSKMSKeyAlias:
    Type: AWS::KMS::Alias
    Properties:
      AliasName: !Sub 'alias/${Environment}-sqs'
      TargetKeyId: !Ref SQSKMSKey

  # ========================================
  # KMS Key for Secrets Manager
  # ========================================
  SecretsKMSKey:
    Type: AWS::KMS::Key
    Properties:
      Description: !Sub 'KMS key for ${Environment} Secrets Manager'
      KeyPolicy:
        Version: '2012-10-17'
        Statement:
          - Sid: Enable IAM User Permissions
            Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:root'
            Action: 'kms:*'
            Resource: '*'
          
          - Sid: Allow Secrets Manager to use the key
            Effect: Allow
            Principal:
              Service: secretsmanager.amazonaws.com
            Action:
              - kms:Decrypt
              - kms:GenerateDataKey
            Resource: '*'
      Tags:
        - Key: Environment
          Value: !Ref Environment

  SecretsKMSKeyAlias:
    Type: AWS::KMS::Alias
    Properties:
      AliasName: !Sub 'alias/${Environment}-secrets'
      TargetKeyId: !Ref SecretsKMSKey

Outputs:
  DynamoDBKMSKeyId:
    Description: KMS Key ID for DynamoDB
    Value: !Ref DynamoDBKMSKey
    Export:
      Name: !Sub '${Environment}-DynamoDB-KMS-Key-ID'

  SQSKMSKeyId:
    Description: KMS Key ID for SQS
    Value: !Ref SQSKMSKey
    Export:
      Name: !Sub '${Environment}-SQS-KMS-Key-ID'

  SecretsKMSKeyId:
    Description: KMS Key ID for Secrets Manager
    Value: !Ref SecretsKMSKey
    Export:
      Name: !Sub '${Environment}-Secrets-KMS-Key-ID'
```

---

## 4️⃣ Secrets Manager Setup

### Store Sensitive Configuration

```javascript
// scripts/create-secrets.js
const AWS = require('aws-sdk');
const secretsManager = new AWS.SecretsManager({ region: 'us-east-1' });

async function createSecrets(environment) {
  // API Keys secret
  const apiKeysSecret = {
    Name: `${environment}/api/keys`,
    Description: `API keys for ${environment} environment`,
    SecretString: JSON.stringify({
      externalApiKey: 'your-external-api-key',
      webhookSecret: 'your-webhook-secret',
      jwtSecret: 'your-jwt-secret'
    }),
    KmsKeyId: `alias/${environment}-secrets`,
    Tags: [
      { Key: 'Environment', Value: environment },
      { Key: 'ManagedBy', Value: 'Terraform' }
    ]
  };

  try {
    const result = await secretsManager.createSecret(apiKeysSecret).promise();
    console.log(`✅ Created secret: ${result.ARN}`);
  } catch (error) {
    if (error.code === 'ResourceExistsException') {
      console.log(`⚠️  Secret already exists: ${apiKeysSecret.Name}`);
    } else {
      console.error(`❌ Error creating secret: ${error.message}`);
    }
  }

  // Database credentials (if using RDS)
  const dbSecret = {
    Name: `${environment}/database/credentials`,
    Description: `Database credentials for ${environment}`,
    SecretString: JSON.stringify({
      username: 'admin',
      password: 'generate-strong-password-here',
      engine: 'postgres',
      host: 'db-endpoint.amazonaws.com',
      port: 5432,
      dbname: 'orders'
    }),
    KmsKeyId: `alias/${environment}-secrets`
  };

  try {
    const result = await secretsManager.createSecret(dbSecret).promise();
    console.log(`✅ Created secret: ${result.ARN}`);
  } catch (error) {
    if (error.code === 'ResourceExistsException') {
      console.log(`⚠️  Secret already exists: ${dbSecret.Name}`);
    } else {
      console.error(`❌ Error creating secret: ${error.message}`);
    }
  }
}

// Usage
const environment = process.argv[2] || 'dev';
createSecrets(environment);
```

```powershell
# Create secrets
node scripts/create-secrets.js dev
```

### Retrieve Secrets in Lambda

```javascript
// src/utils/secrets.ts
import { SecretsManager } from 'aws-sdk';

const secretsManager = new SecretsManager();
const secretsCache: Map<string, any> = new Map();

export async function getSecret(secretName: string): Promise<any> {
  // Check cache first
  if (secretsCache.has(secretName)) {
    return secretsCache.get(secretName);
  }

  try {
    const result = await secretsManager
      .getSecretValue({ SecretId: secretName })
      .promise();

    if (!result.SecretString) {
      throw new Error('Secret string is empty');
    }

    const secret = JSON.parse(result.SecretString);
    
    // Cache for Lambda container reuse
    secretsCache.set(secretName, secret);
    
    return secret;
  } catch (error) {
    console.error(`Error retrieving secret ${secretName}:`, error);
    throw error;
  }
}

// Usage in Lambda function
export async function getApiKeys() {
  const environment = process.env.ENVIRONMENT || 'dev';
  const secretName = `${environment}/api/keys`;
  return await getSecret(secretName);
}
```

---

## 5️⃣ API Gateway Authorization

### Cognito User Pool Setup

```yaml
# Add to iam-roles.yaml or create cognito-stack.yaml

CognitoUserPool:
  Type: AWS::Cognito::UserPool
  Properties:
    UserPoolName: !Sub '${Environment}-order-users'
    AutoVerifiedAttributes:
      - email
    UsernameAttributes:
      - email
    Schema:
      - Name: email
        AttributeDataType: String
        Required: true
        Mutable: false
      - Name: name
        AttributeDataType: String
        Required: true
        Mutable: true
    Policies:
      PasswordPolicy:
        MinimumLength: 12
        RequireUppercase: true
        RequireLowercase: true
        RequireNumbers: true
        RequireSymbols: true
    AccountRecoverySetting:
      RecoveryMechanisms:
        - Name: verified_email
          Priority: 1
    Tags:
      - Key: Environment
        Value: !Ref Environment

CognitoUserPoolClient:
  Type: AWS::Cognito::UserPoolClient
  Properties:
    ClientName: !Sub '${Environment}-order-app'
    UserPoolId: !Ref CognitoUserPool
    GenerateSecret: false
    ExplicitAuthFlows:
      - ALLOW_USER_PASSWORD_AUTH
      - ALLOW_REFRESH_TOKEN_AUTH
    PreventUserExistenceErrors: ENABLED
    RefreshTokenValidity: 30
    AccessTokenValidity: 1
    IdTokenValidity: 1
    TokenValidityUnits:
      AccessToken: hours
      IdToken: hours
      RefreshToken: days

CognitoAuthorizer:
  Type: AWS::ApiGateway::Authorizer
  Properties:
    Name: !Sub '${Environment}-cognito-authorizer'
    Type: COGNITO_USER_POOLS
    IdentitySource: method.request.header.Authorization
    RestApiId: !Ref ApiGateway  # Reference to your API Gateway
    ProviderARNs:
      - !GetAtt CognitoUserPool.Arn
```

---

## 6️⃣ Deploy IAM Stack

### Deployment Commands

```powershell
# Deploy IAM roles
aws cloudformation create-stack `
  --stack-name dev-iam-roles `
  --template-body file://infrastructure/cloudformation/iam-roles.yaml `
  --parameters ParameterKey=Environment,ParameterValue=dev `
  --capabilities CAPABILITY_NAMED_IAM `
  --region us-east-1

# Deploy KMS keys
aws cloudformation create-stack `
  --stack-name dev-kms-stack `
  --template-body file://infrastructure/cloudformation/kms-stack.yaml `
  --parameters ParameterKey=Environment,ParameterValue=dev `
  --region us-east-1

# Wait for completion
aws cloudformation wait stack-create-complete --stack-name dev-iam-roles
aws cloudformation wait stack-create-complete --stack-name dev-kms-stack
```

---

## 7️⃣ IAM Best Practices Implementation

### Resource-Based Policies

```javascript
// Example: S3 bucket policy for Lambda access
const s3BucketPolicy = {
  Version: '2012-10-17',
  Statement: [
    {
      Sid: 'AllowLambdaRead',
      Effect: 'Allow',
      Principal: {
        AWS: 'arn:aws:iam::123456789012:role/dev-api-handler-role'
      },
      Action: ['s3:GetObject', 's3:ListBucket'],
      Resource: [
        'arn:aws:s3:::dev-order-documents',
        'arn:aws:s3:::dev-order-documents/*'
      ]
    }
  ]
};
```

### Service Control Policies (SCPs)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyPublicS3Access",
      "Effect": "Deny",
      "Action": [
        "s3:PutAccountPublicAccessBlock",
        "s3:DeletePublicAccessBlock"
      ],
      "Resource": "*"
    },
    {
      "Sid": "RequireEncryption",
      "Effect": "Deny",
      "Action": "s3:PutObject",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "AES256"
        }
      }
    }
  ]
}
```

---

## 8️⃣ IAM Access Analyzer

### Enable IAM Access Analyzer

```yaml
IAMAccessAnalyzer:
  Type: AWS::AccessAnalyzer::Analyzer
  Properties:
    AnalyzerName: !Sub '${Environment}-analyzer'
    Type: ACCOUNT
    Tags:
      - Key: Environment
        Value: !Ref Environment
```

### Review Findings

```powershell
# List findings
aws accessanalyzer list-findings `
  --analyzer-arn arn:aws:access-analyzer:us-east-1:123456789012:analyzer/dev-analyzer

# Get specific finding
aws accessanalyzer get-finding `
  --analyzer-arn arn:aws:access-analyzer:us-east-1:123456789012:analyzer/dev-analyzer `
  --id <finding-id>
```

---

## 9️⃣ Testing IAM Permissions

### Test Script

```javascript
// test-iam-permissions.js
const AWS = require('aws-sdk');

async function testIAMPermissions() {
  const sts = new AWS.STS();
  
  console.log('🔐 Testing IAM permissions...\n');

  try {
    // Get current identity
    const identity = await sts.getCallerIdentity().promise();
    console.log('✅ Current Identity:');
    console.log(`   ARN: ${identity.Arn}`);
    console.log(`   Account: ${identity.Account}\n`);

    // Test DynamoDB access
    const dynamodb = new AWS.DynamoDB();
    try {
      await dynamodb.listTables({ Limit: 1 }).promise();
      console.log('✅ DynamoDB: Access granted');
    } catch (error) {
      console.log(`❌ DynamoDB: ${error.message}`);
    }

    // Test SQS access
    const sqs = new AWS.SQS();
    try {
      await sqs.listQueues({ MaxResults: 1 }).promise();
      console.log('✅ SQS: Access granted');
    } catch (error) {
      console.log(`❌ SQS: ${error.message}`);
    }

    // Test SNS access
    const sns = new AWS.SNS();
    try {
      await sns.listTopics().promise();
      console.log('✅ SNS: Access granted');
    } catch (error) {
      console.log(`❌ SNS: ${error.message}`);
    }

    // Test Lambda access
    const lambda = new AWS.Lambda();
    try {
      await lambda.listFunctions({ MaxItems: 1 }).promise();
      console.log('✅ Lambda: Access granted');
    } catch (error) {
      console.log(`❌ Lambda: ${error.message}`);
    }

    // Test KMS access
    const kms = new AWS.KMS();
    try {
      await kms.listKeys({ Limit: 1 }).promise();
      console.log('✅ KMS: Access granted');
    } catch (error) {
      console.log(`❌ KMS: ${error.message}`);
    }

    console.log('\n✨ IAM permission test complete!');
  } catch (error) {
    console.error('❌ Error:', error.message);
  }
}

testIAMPermissions();
```

---

## 🔟 Security Monitoring

### CloudTrail Configuration

```yaml
OrderProcessingTrail:
  Type: AWS::CloudTrail::Trail
  Properties:
    TrailName: !Sub '${Environment}-order-processing-trail'
    S3BucketName: !Ref CloudTrailBucket
    IncludeGlobalServiceEvents: true
    IsLogging: true
    IsMultiRegionTrail: true
    EventSelectors:
      - IncludeManagementEvents: true
        ReadWriteType: All
        DataResources:
          - Type: AWS::Lambda::Function
            Values:
              - !Sub 'arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function/${Environment}-*'
          - Type: AWS::DynamoDB::Table
            Values:
              - !Sub 'arn:aws:dynamodb:${AWS::Region}:${AWS::AccountId}:table/${Environment}-*'
```

### GuardDuty Integration

```powershell
# Enable GuardDuty
aws guardduty create-detector --enable --region us-east-1

# Get detector ID
aws guardduty list-detectors --region us-east-1
```

---

## 🎯 Security Checklist

### DO ✅

1. **Use separate roles**: One role per Lambda function
2. **Implement least privilege**: Only necessary permissions
3. **Enable MFA**: For all human users
4. **Rotate credentials**: Regularly rotate access keys
5. **Use KMS**: Encrypt sensitive data
6. **Enable CloudTrail**: Track all API calls
7. **Use IAM Access Analyzer**: Find overly permissive policies
8. **Tag everything**: For cost tracking and security
9. **Use VPC endpoints**: Avoid NAT Gateway costs
10. **Review regularly**: Audit IAM policies quarterly

### DON'T ❌

1. **Don't use root account**: Create IAM users instead
2. **Don't hardcode credentials**: Use IAM roles and Secrets Manager
3. **Don't use wildcards**: Be specific in policies (`*` is dangerous)
4. **Don't share credentials**: One user per person/service
5. **Don't skip encryption**: Always encrypt at rest and in transit
6. **Don't ignore alerts**: Monitor GuardDuty and Security Hub
7. **Don't grant `AdministratorAccess`**: Use specific permissions
8. **Don't disable CloudTrail**: Always keep audit logs
9. **Don't use long-lived access keys**: Prefer temporary credentials
10. **Don't skip reviews**: Regularly audit permissions

---

## 📊 Cost Implications

```javascript
const iamCosts = {
  iam: {
    roles: 'Free',
    policies: 'Free',
    users: 'Free (up to 5000)'
  },
  kms: {
    keys: '$1/month per key',
    requests: '$0.03 per 10,000 requests',
    estimated: '$3-5/month for 3 keys'
  },
  secretsManager: {
    secrets: '$0.40/month per secret',
    apiCalls: '$0.05 per 10,000 API calls',
    estimated: '$2-3/month for 5 secrets'
  },
  cloudTrail: {
    management: 'First trail free',
    dataEvents: '$0.10 per 100,000 events',
    estimated: '$5-20/month'
  },
  guardDuty: {
    cloudTrail: '$4.50 per million events',
    vpcFlow: '$1.05 per GB',
    estimated: '$10-30/month'
  },
  total: '$20-60/month for comprehensive security'
};
```

---

## 🎯 Next Steps

IAM security setup complete! You now have:

- ✅ Separate IAM roles for each Lambda function
- ✅ Least privilege policies
- ✅ KMS encryption keys
- ✅ Secrets Manager for sensitive data
- ✅ API Gateway authorization
- ✅ CloudTrail and GuardDuty enabled
- ✅ IAM Access Analyzer configured

**Ready to proceed?**

👉 [DynamoDB Setup](./05_DYNAMODB_SETUP.md)

---

**Last Updated**: November 12, 2025  
**Version**: 1.0.0
