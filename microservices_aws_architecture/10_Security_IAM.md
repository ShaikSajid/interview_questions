# Security & IAM for Microservices

## Question 19: Comprehensive Security Implementation

### 📋 Question Statement

Implement a complete security architecture for Emirates NBD microservices including IAM roles/policies, secrets management, encryption at rest and in transit, VPC security, WAF, and compliance (PCI-DSS) requirements.

---

### 🏗️ Security Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│              EMIRATES NBD SECURITY ARCHITECTURE                             │
└────────────────────────────────────────────────────────────────────────────┘

    NETWORK SECURITY                 IAM & AUTHENTICATION
    ───────────────                  ────────────────────
    
    ┌──────────────┐                 ┌──────────────────┐
    │     WAF      │                 │  Cognito User    │
    │  • Geo Block │                 │      Pool        │
    │  • Rate Limit│◄────────────────┤  • MFA           │
    └──────┬───────┘                 │  • Password Policy│
           │                         └──────────────────┘
           ▼
    ┌──────────────┐
    │  API Gateway │
    │  • TLS 1.3   │
    └──────┬───────┘
           │
    ┌──────▼───────────────────────────────────────────┐
    │                VPC (10.0.0.0/16)                 │
    │                                                   │
    │  PUBLIC SUBNET          PRIVATE SUBNET           │
    │  ─────────────         ──────────────            │
    │  ┌──────────┐          ┌──────────┐             │
    │  │   ALB    │          │   ECS    │             │
    │  │  (TLS)   │──────────►│ Services │             │
    │  └──────────┘          └────┬─────┘             │
    │                              │                    │
    │  ISOLATED SUBNET       ┌────▼─────┐             │
    │  ───────────────       │   RDS    │             │
    │                        │ (Encrypted)│            │
    │                        └──────────┘             │
    └───────────────────────────────────────────────────┘
    
    SECRETS MANAGEMENT              ENCRYPTION
    ──────────────                 ──────────
    
    ┌──────────────┐               ┌──────────────┐
    │   Secrets    │               │     KMS      │
    │   Manager    │               │  • CMK       │
    │  • DB Creds  │               │  • Key       │
    │  • API Keys  │               │    Rotation  │
    │  • Rotation  │               └──────────────┘
    └──────────────┘
```

---

### 🔧 Complete Security Stack (CDK)

```typescript
// infrastructure/cdk/security-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as secretsmanager from 'aws-cdk-lib/aws-secretsmanager';
import * as kms from 'aws-cdk-lib/aws-kms';
import * as wafv2 from 'aws-cdk-lib/aws-wafv2';
import * as cognito from 'aws-cdk-lib/aws-cognito';

export class SecurityStack extends cdk.Stack {
  public readonly vpc: ec2.Vpc;
  public readonly kmsKey: kms.Key;

  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // ============================================
    // 1. VPC WITH SECURITY LAYERS
    // ============================================
    this.vpc = new ec2.Vpc(this, 'SecureVPC', {
      maxAzs: 3,
      natGateways: 3,
      subnetConfiguration: [
        {
          cidrMask: 24,
          name: 'Public',
          subnetType: ec2.SubnetType.PUBLIC
        },
        {
          cidrMask: 24,
          name: 'Private',
          subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS
        },
        {
          cidrMask: 24,
          name: 'Isolated',
          subnetType: ec2.SubnetType.PRIVATE_ISOLATED
        }
      ],
      flowLogs: {
        'VPCFlowLogs': {
          trafficType: ec2.FlowLogTrafficType.ALL
        }
      }
    });

    // VPC Endpoints for AWS Services (no internet gateway needed)
    this.vpc.addInterfaceEndpoint('SecretsManagerEndpoint', {
      service: ec2.InterfaceVpcEndpointAwsService.SECRETS_MANAGER
    });

    this.vpc.addInterfaceEndpoint('ECREndpoint', {
      service: ec2.InterfaceVpcEndpointAwsService.ECR
    });

    this.vpc.addGatewayEndpoint('S3Endpoint', {
      service: ec2.GatewayVpcEndpointAwsService.S3
    });

    this.vpc.addGatewayEndpoint('DynamoDBEndpoint', {
      service: ec2.GatewayVpcEndpointAwsService.DYNAMODB
    });

    // ============================================
    // 2. KMS ENCRYPTION KEY
    // ============================================
    this.kmsKey = new kms.Key(this, 'BankingKMSKey', {
      enableKeyRotation: true,
      description: 'Emirates NBD Banking Services Encryption Key',
      alias: 'alias/banking-services',
      removalPolicy: cdk.RemovalPolicy.RETAIN
    });

    // ============================================
    // 3. SECRETS MANAGER
    // ============================================
    const dbSecret = new secretsmanager.Secret(this, 'DatabaseSecret', {
      secretName: 'banking/database/credentials',
      generateSecretString: {
        secretStringTemplate: JSON.stringify({
          username: 'bankingadmin',
          host: 'rds-endpoint.amazonaws.com',
          port: 5432,
          dbname: 'banking'
        }),
        generateStringKey: 'password',
        excludePunctuation: true,
        passwordLength: 32
      },
      encryptionKey: this.kmsKey
    });

    // Automatic rotation
    dbSecret.addRotationSchedule('RotationSchedule', {
      automaticallyAfter: cdk.Duration.days(30)
    });

    const apiKeySecret = new secretsmanager.Secret(this, 'APIKeySecret', {
      secretName: 'banking/api/keys',
      generateSecretString: {
        secretStringTemplate: JSON.stringify({
          paymentGateway: 'placeholder',
          smsProvider: 'placeholder'
        }),
        generateStringKey: 'internalAPIKey',
        excludePunctuation: true,
        passwordLength: 64
      },
      encryptionKey: this.kmsKey
    });

    // ============================================
    // 4. IAM ROLES - LEAST PRIVILEGE
    // ============================================
    
    // ECS Task Execution Role
    const ecsExecutionRole = new iam.Role(this, 'ECSExecutionRole', {
      assumedBy: new iam.ServicePrincipal('ecs-tasks.amazonaws.com'),
      managedPolicies: [
        iam.ManagedPolicy.fromAwsManagedPolicyName('service-role/AmazonECSTaskExecutionRolePolicy')
      ]
    });

    // Allow pulling from ECR
    ecsExecutionRole.addToPolicy(new iam.PolicyStatement({
      effect: iam.Effect.ALLOW,
      actions: [
        'ecr:GetAuthorizationToken',
        'ecr:BatchCheckLayerAvailability',
        'ecr:GetDownloadUrlForLayer',
        'ecr:BatchGetImage'
      ],
      resources: ['*']
    }));

    // Allow reading secrets
    ecsExecutionRole.addToPolicy(new iam.PolicyStatement({
      effect: iam.Effect.ALLOW,
      actions: [
        'secretsmanager:GetSecretValue'
      ],
      resources: [dbSecret.secretArn, apiKeySecret.secretArn]
    }));

    // Allow KMS decryption
    this.kmsKey.grantDecrypt(ecsExecutionRole);

    // ECS Task Role (what the container can do)
    const ecsTaskRole = new iam.Role(this, 'ECSTaskRole', {
      assumedBy: new iam.ServicePrincipal('ecs-tasks.amazonaws.com')
    });

    // DynamoDB access (scoped to specific tables)
    ecsTaskRole.addToPolicy(new iam.PolicyStatement({
      effect: iam.Effect.ALLOW,
      actions: [
        'dynamodb:GetItem',
        'dynamodb:PutItem',
        'dynamodb:UpdateItem',
        'dynamodb:Query',
        'dynamodb:Scan'
      ],
      resources: [
        `arn:aws:dynamodb:${this.region}:${this.account}:table/banking-*`
      ]
    }));

    // Lambda Execution Role
    const lambdaRole = new iam.Role(this, 'LambdaExecutionRole', {
      assumedBy: new iam.ServicePrincipal('lambda.amazonaws.com'),
      managedPolicies: [
        iam.ManagedPolicy.fromAwsManagedPolicyName('service-role/AWSLambdaBasicExecutionRole'),
        iam.ManagedPolicy.fromAwsManagedPolicyName('service-role/AWSLambdaVPCAccessExecutionRole')
      ]
    });

    lambdaRole.addToPolicy(new iam.PolicyStatement({
      effect: iam.Effect.ALLOW,
      actions: [
        'dynamodb:GetItem',
        'dynamodb:PutItem',
        'dynamodb:Query'
      ],
      resources: [
        `arn:aws:dynamodb:${this.region}:${this.account}:table/banking-transactions`
      ]
    }));

    this.kmsKey.grantDecrypt(lambdaRole);

    // ============================================
    // 5. COGNITO USER POOL
    // ============================================
    const userPool = new cognito.UserPool(this, 'BankingUserPool', {
      userPoolName: 'emirates-nbd-users',
      selfSignUpEnabled: false, // Admin creates users
      signInAliases: {
        email: true,
        username: true
      },
      passwordPolicy: {
        minLength: 12,
        requireLowercase: true,
        requireUppercase: true,
        requireDigits: true,
        requireSymbols: true,
        tempPasswordValidity: cdk.Duration.days(1)
      },
      mfa: cognito.Mfa.REQUIRED,
      mfaSecondFactor: {
        sms: true,
        otp: true
      },
      accountRecovery: cognito.AccountRecovery.EMAIL_ONLY,
      advancedSecurityMode: cognito.AdvancedSecurityMode.ENFORCED,
      deviceTracking: {
        challengeRequiredOnNewDevice: true,
        deviceOnlyRememberedOnUserPrompt: true
      }
    });

    // User Pool Client
    const userPoolClient = userPool.addClient('BankingWebClient', {
      authFlows: {
        userPassword: true,
        userSrp: true
      },
      oAuth: {
        flows: {
          authorizationCodeGrant: true
        },
        scopes: [
          cognito.OAuthScope.EMAIL,
          cognito.OAuthScope.OPENID,
          cognito.OAuthScope.PROFILE
        ]
      },
      accessTokenValidity: cdk.Duration.minutes(60),
      idTokenValidity: cdk.Duration.minutes(60),
      refreshTokenValidity: cdk.Duration.days(30)
    });

    // ============================================
    // 6. WAF WEB ACL
    // ============================================
    const wafAcl = new wafv2.CfnWebACL(this, 'BankingWAF', {
      scope: 'REGIONAL',
      defaultAction: { allow: {} },
      visibilityConfig: {
        cloudWatchMetricsEnabled: true,
        metricName: 'BankingWAF',
        sampledRequestsEnabled: true
      },
      rules: [
        // Rate limiting
        {
          name: 'RateLimitRule',
          priority: 1,
          statement: {
            rateBasedStatement: {
              limit: 2000,
              aggregateKeyType: 'IP'
            }
          },
          action: { block: {} },
          visibilityConfig: {
            cloudWatchMetricsEnabled: true,
            metricName: 'RateLimitRule',
            sampledRequestsEnabled: true
          }
        },
        // Geo blocking
        {
          name: 'GeoBlockRule',
          priority: 2,
          statement: {
            geoMatchStatement: {
              countryCodes: ['AE', 'SA', 'QA', 'KW', 'BH', 'OM'] // GCC countries only
            }
          },
          action: { allow: {} },
          visibilityConfig: {
            cloudWatchMetricsEnabled: true,
            metricName: 'GeoBlockRule',
            sampledRequestsEnabled: true
          }
        },
        // AWS Managed Rules - Core
        {
          name: 'AWSManagedRulesCommonRuleSet',
          priority: 3,
          statement: {
            managedRuleGroupStatement: {
              vendorName: 'AWS',
              name: 'AWSManagedRulesCommonRuleSet'
            }
          },
          overrideAction: { none: {} },
          visibilityConfig: {
            cloudWatchMetricsEnabled: true,
            metricName: 'AWSManagedRulesCommonRuleSet',
            sampledRequestsEnabled: true
          }
        },
        // SQL Injection protection
        {
          name: 'AWSManagedRulesSQLiRuleSet',
          priority: 4,
          statement: {
            managedRuleGroupStatement: {
              vendorName: 'AWS',
              name: 'AWSManagedRulesSQLiRuleSet'
            }
          },
          overrideAction: { none: {} },
          visibilityConfig: {
            cloudWatchMetricsEnabled: true,
            metricName: 'AWSManagedRulesSQLiRuleSet',
            sampledRequestsEnabled: true
          }
        }
      ]
    });

    // ============================================
    // 7. SECURITY GROUPS
    // ============================================
    
    // ALB Security Group
    const albSecurityGroup = new ec2.SecurityGroup(this, 'ALBSecurityGroup', {
      vpc: this.vpc,
      description: 'Security group for Application Load Balancer',
      allowAllOutbound: false
    });

    albSecurityGroup.addIngressRule(
      ec2.Peer.anyIpv4(),
      ec2.Port.tcp(443),
      'Allow HTTPS from internet'
    );

    // ECS Security Group
    const ecsSecurityGroup = new ec2.SecurityGroup(this, 'ECSSecurityGroup', {
      vpc: this.vpc,
      description: 'Security group for ECS tasks',
      allowAllOutbound: true
    });

    ecsSecurityGroup.addIngressRule(
      albSecurityGroup,
      ec2.Port.tcp(3000),
      'Allow traffic from ALB'
    );

    // RDS Security Group
    const rdsSecurityGroup = new ec2.SecurityGroup(this, 'RDSSecurityGroup', {
      vpc: this.vpc,
      description: 'Security group for RDS database',
      allowAllOutbound: false
    });

    rdsSecurityGroup.addIngressRule(
      ecsSecurityGroup,
      ec2.Port.tcp(5432),
      'Allow PostgreSQL from ECS'
    );

    // Lambda Security Group (if in VPC)
    const lambdaSecurityGroup = new ec2.SecurityGroup(this, 'LambdaSecurityGroup', {
      vpc: this.vpc,
      description: 'Security group for Lambda functions',
      allowAllOutbound: true
    });

    rdsSecurityGroup.addIngressRule(
      lambdaSecurityGroup,
      ec2.Port.tcp(5432),
      'Allow PostgreSQL from Lambda'
    );

    // Outputs
    new cdk.CfnOutput(this, 'VPCId', { value: this.vpc.vpcId });
    new cdk.CfnOutput(this, 'KMSKeyId', { value: this.kmsKey.keyId });
    new cdk.CfnOutput(this, 'UserPoolId', { value: userPool.userPoolId });
    new cdk.CfnOutput(this, 'UserPoolClientId', { value: userPoolClient.userPoolClientId });
  }
}
```

### 🔐 Secrets Management Implementation

```javascript
// services/shared/src/secrets-manager.js
const { SecretsManagerClient, GetSecretValueCommand } = require('@aws-sdk/client-secretsmanager');

class SecretsManager {
  constructor() {
    this.client = new SecretsManagerClient({});
    this.cache = new Map();
    this.cacheTTL = 5 * 60 * 1000; // 5 minutes
  }

  async getSecret(secretName) {
    // Check cache first
    const cached = this.cache.get(secretName);
    if (cached && Date.now() - cached.timestamp < this.cacheTTL) {
      return cached.value;
    }

    try {
      const response = await this.client.send(new GetSecretValueCommand({
        SecretId: secretName
      }));

      const secret = JSON.parse(response.SecretString);
      
      // Cache the secret
      this.cache.set(secretName, {
        value: secret,
        timestamp: Date.now()
      });

      return secret;
    } catch (error) {
      console.error(`Error retrieving secret ${secretName}:`, error);
      throw error;
    }
  }

  async getDatabaseCredentials() {
    return this.getSecret('banking/database/credentials');
  }

  async getAPIKeys() {
    return this.getSecret('banking/api/keys');
  }

  clearCache() {
    this.cache.clear();
  }
}

module.exports = new SecretsManager();
```

### 🔒 Data Encryption Service

```javascript
// services/shared/src/encryption-service.js
const { KMSClient, EncryptCommand, DecryptCommand, GenerateDataKeyCommand } = require('@aws-sdk/client-kms');
const crypto = require('crypto');

class EncryptionService {
  constructor() {
    this.kms = new KMSClient({});
    this.keyId = process.env.KMS_KEY_ID || 'alias/banking-services';
  }

  // Encrypt sensitive data using KMS
  async encryptWithKMS(plaintext) {
    const response = await this.kms.send(new EncryptCommand({
      KeyId: this.keyId,
      Plaintext: Buffer.from(plaintext)
    }));

    return response.CiphertextBlob.toString('base64');
  }

  // Decrypt data using KMS
  async decryptWithKMS(ciphertext) {
    const response = await this.kms.send(new DecryptCommand({
      CiphertextBlob: Buffer.from(ciphertext, 'base64')
    }));

    return response.Plaintext.toString();
  }

  // Envelope encryption for large data
  async envelopeEncrypt(data) {
    // Generate data key
    const dataKeyResponse = await this.kms.send(new GenerateDataKeyCommand({
      KeyId: this.keyId,
      KeySpec: 'AES_256'
    }));

    const plaintextKey = dataKeyResponse.Plaintext;
    const encryptedKey = dataKeyResponse.CiphertextBlob;

    // Encrypt data with data key
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv('aes-256-gcm', plaintextKey, iv);
    
    let encrypted = cipher.update(data, 'utf8', 'base64');
    encrypted += cipher.final('base64');
    const authTag = cipher.getAuthTag();

    return {
      encryptedData: encrypted,
      encryptedKey: encryptedKey.toString('base64'),
      iv: iv.toString('base64'),
      authTag: authTag.toString('base64')
    };
  }

  // Envelope decryption
  async envelopeDecrypt(envelope) {
    // Decrypt data key
    const keyResponse = await this.kms.send(new DecryptCommand({
      CiphertextBlob: Buffer.from(envelope.encryptedKey, 'base64')
    }));

    const plaintextKey = keyResponse.Plaintext;
    const iv = Buffer.from(envelope.iv, 'base64');
    const authTag = Buffer.from(envelope.authTag, 'base64');

    // Decrypt data
    const decipher = crypto.createDecipheriv('aes-256-gcm', plaintextKey, iv);
    decipher.setAuthTag(authTag);

    let decrypted = decipher.update(envelope.encryptedData, 'base64', 'utf8');
    decrypted += decipher.final('utf8');

    return decrypted;
  }

  // Hash sensitive data (e.g., PII for search)
  hashData(data) {
    return crypto.createHash('sha256').update(data).digest('hex');
  }

  // PCI-DSS compliant card masking
  maskCardNumber(cardNumber) {
    if (!cardNumber || cardNumber.length < 13) {
      return '****';
    }
    const last4 = cardNumber.slice(-4);
    const first6 = cardNumber.slice(0, 6);
    return `${first6}****${last4}`;
  }

  // Mask sensitive fields in logs
  maskSensitiveData(obj) {
    const masked = { ...obj };
    const sensitiveFields = ['password', 'cardNumber', 'cvv', 'pin', 'ssn', 'accountNumber'];
    
    for (const key in masked) {
      if (sensitiveFields.includes(key)) {
        masked[key] = '***REDACTED***';
      }
    }
    
    return masked;
  }
}

module.exports = new EncryptionService();
```

### 🛡️ Security Middleware

```javascript
// services/account-service/src/middleware/security-middleware.js
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');
const mongoSanitize = require('express-mongo-sanitize');
const xss = require('xss-clean');
const hpp = require('hpp');

// Helmet for security headers
const helmetConfig = helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", 'data:', 'https:']
    }
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  }
});

// Rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Limit each IP to 100 requests per windowMs
  message: 'Too many requests from this IP, please try again later.',
  standardHeaders: true,
  legacyHeaders: false
});

// Strict rate limiting for sensitive endpoints
const strictLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  message: 'Too many login attempts, please try again later.'
});

// Input validation middleware
const validateInput = (req, res, next) => {
  const { body, params, query } = req;
  
  // Check for SQL injection patterns
  const sqlPatterns = /(\b(SELECT|INSERT|UPDATE|DELETE|DROP|CREATE|ALTER|EXEC|EXECUTE)\b)/gi;
  const checkString = JSON.stringify({ ...body, ...params, ...query });
  
  if (sqlPatterns.test(checkString)) {
    return res.status(400).json({ error: 'Invalid input detected' });
  }
  
  next();
};

// Request logging (with sensitive data masking)
const requestLogger = (req, res, next) => {
  const maskedBody = { ...req.body };
  if (maskedBody.password) maskedBody.password = '***';
  if (maskedBody.cardNumber) maskedBody.cardNumber = '****';
  
  console.log({
    method: req.method,
    path: req.path,
    ip: req.ip,
    userAgent: req.get('user-agent'),
    body: maskedBody
  });
  
  next();
};

module.exports = {
  helmetConfig,
  limiter,
  strictLimiter,
  validateInput,
  requestLogger,
  mongoSanitize: mongoSanitize(),
  xss: xss(),
  hpp: hpp()
};
```

### 🔑 JWT Authentication

```javascript
// services/shared/src/auth/jwt-handler.js
const jwt = require('jsonwebtoken');
const SecretsManager = require('../secrets-manager');

class JWTHandler {
  constructor() {
    this.algorithm = 'RS256';
    this.expiresIn = '1h';
  }

  async getKeys() {
    const secrets = await SecretsManager.getSecret('banking/jwt/keys');
    return {
      privateKey: secrets.privateKey,
      publicKey: secrets.publicKey
    };
  }

  async generateToken(payload) {
    const { privateKey } = await this.getKeys();
    
    return jwt.sign(
      {
        ...payload,
        iss: 'emirates-nbd',
        aud: 'banking-api'
      },
      privateKey,
      {
        algorithm: this.algorithm,
        expiresIn: this.expiresIn
      }
    );
  }

  async verifyToken(token) {
    try {
      const { publicKey } = await this.getKeys();
      
      const decoded = jwt.verify(token, publicKey, {
        algorithms: [this.algorithm],
        issuer: 'emirates-nbd',
        audience: 'banking-api'
      });
      
      return { valid: true, payload: decoded };
    } catch (error) {
      return { valid: false, error: error.message };
    }
  }

  async refreshToken(oldToken) {
    const verification = await this.verifyToken(oldToken);
    
    if (!verification.valid) {
      throw new Error('Invalid token');
    }
    
    // Generate new token with same payload (excluding exp, iat)
    const { exp, iat, ...payload } = verification.payload;
    return this.generateToken(payload);
  }
}

module.exports = new JWTHandler();
```

### 🎓 Interview Discussion Points - Q19

**Q1: What is the principle of least privilege?**

**A**:
- Grant only permissions needed to perform a task
- No more, no less
- Reduces attack surface
- Use IAM roles, not access keys

**Q2: How do you secure data at rest?**

**A**:
- **Encryption**: Use KMS for encryption keys
- **Database**: Enable encryption on RDS, DynamoDB
- **S3**: Enable default encryption
- **EBS**: Encrypt volumes

**Q3: How do you secure data in transit?**

**A**:
- **TLS 1.2+**: All external communication
- **VPC**: Private networking between services
- **VPN/PrivateLink**: For on-premises connectivity
- **Certificate Management**: Use ACM

**Q4: What is envelope encryption?**

**A**:
- **Data Key**: Encrypts actual data
- **Master Key**: Encrypts data key (stored in KMS)
- **Benefits**: Fast, secure, key rotation
- **Use case**: Encrypting large files

---

## Question 20: Audit Logging & Compliance

### 📋 Question Statement

Implement comprehensive audit logging for Emirates NBD with CloudTrail, CloudWatch Logs, and custom audit tables. Include PCI-DSS compliance requirements, data retention policies, and security incident detection.

---

### 🏗️ Audit Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│              EMIRATES NBD AUDIT & COMPLIANCE ARCHITECTURE                   │
└────────────────────────────────────────────────────────────────────────────┘

    APPLICATION LOGS              AWS SERVICE LOGS
    ────────────────             ─────────────────
    
    ┌──────────────┐             ┌──────────────┐
    │   Services   │             │  CloudTrail  │
    │  • ECS       │────┐        │  • API Calls │
    │  • Lambda    │    │        │  • IAM       │
    └──────────────┘    │        └──────┬───────┘
                        │               │
                        ▼               ▼
                ┌────────────────────────────┐
                │     CloudWatch Logs        │
                │  • Service Logs            │
                │  • Security Events         │
                │  • Audit Trail             │
                └────────┬───────────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
    ┌─────────┐   ┌─────────┐   ┌─────────┐
    │   S3    │   │ Lambda  │   │ Kinesis │
    │ Archive │   │ Alerts  │   │ Stream  │
    └─────────┘   └─────────┘   └────┬────┘
                                      │
                                      ▼
                              ┌──────────────┐
                              │   Athena     │
                              │  Analytics   │
                              └──────────────┘
```

---

### 🔧 Audit Stack (CDK)

```typescript
// infrastructure/cdk/audit-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as logs from 'aws-cdk-lib/aws-logs';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as cloudtrail from 'aws-cdk-lib/aws-cloudtrail';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as iam from 'aws-cdk-lib/aws-iam';

export class AuditStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // S3 bucket for audit logs
    const auditBucket = new s3.Bucket(this, 'AuditLogsBucket', {
      bucketName: `banking-audit-logs-${this.account}`,
      encryption: s3.BucketEncryption.S3_MANAGED,
      versioned: true,
      lifecycleRules: [
        {
          id: 'MoveToGlacier',
          transitions: [
            {
              storageClass: s3.StorageClass.GLACIER,
              transitionAfter: cdk.Duration.days(90)
            }
          ],
          expiration: cdk.Duration.days(2555) // 7 years for PCI-DSS
        }
      ],
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL
    });

    // CloudTrail for API auditing
    const trail = new cloudtrail.Trail(this, 'BankingAuditTrail', {
      bucket: auditBucket,
      isMultiRegionTrail: true,
      includeGlobalServiceEvents: true,
      managementEvents: cloudtrail.ReadWriteType.ALL,
      sendToCloudWatchLogs: true
    });

    // Audit log group
    const auditLogGroup = new logs.LogGroup(this, 'AuditLogGroup', {
      logGroupName: '/banking/audit',
      retention: logs.RetentionDays.ONE_YEAR
    });

    // Audit DynamoDB table
    const auditTable = new dynamodb.Table(this, 'AuditTable', {
      tableName: 'banking-audit-log',
      partitionKey: { name: 'userId', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'timestamp', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      pointInTimeRecovery: true,
      stream: dynamodb.StreamViewType.NEW_IMAGE
    });

    auditTable.addGlobalSecondaryIndex({
      indexName: 'EventTypeIndex',
      partitionKey: { name: 'eventType', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'timestamp', type: dynamodb.AttributeType.STRING }
    });

    // Security event detector Lambda
    const securityDetector = new lambda.Function(this, 'SecurityEventDetector', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambdas/security-detector'),
      environment: {
        SNS_TOPIC_ARN: 'arn:aws:sns:region:account:security-alerts'
      }
    });

    auditLogGroup.grantWrite(securityDetector);

    new cdk.CfnOutput(this, 'AuditBucket', { value: auditBucket.bucketName });
    new cdk.CfnOutput(this, 'AuditTable', { value: auditTable.tableName });
  }
}
```

### 📊 Audit Logger Implementation

```javascript
// services/shared/src/audit-logger.js
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, PutCommand } = require('@aws-sdk/lib-dynamodb');
const { CloudWatchLogsClient, PutLogEventsCommand } = require('@aws-sdk/client-cloudwatch-logs');

class AuditLogger {
  constructor() {
    const client = new DynamoDBClient({});
    this.docClient = DynamoDBDocumentClient.from(client);
    this.cwLogs = new CloudWatchLogsClient({});
    this.tableName = 'banking-audit-log';
    this.logGroupName = '/banking/audit';
  }

  async logEvent(event) {
    const auditEntry = {
      userId: event.userId || 'SYSTEM',
      timestamp: new Date().toISOString(),
      eventType: event.eventType,
      action: event.action,
      resource: event.resource,
      resourceId: event.resourceId,
      ipAddress: event.ipAddress,
      userAgent: event.userAgent,
      status: event.status,
      details: event.details,
      sessionId: event.sessionId,
      correlationId: event.correlationId,
      ttl: Math.floor(Date.now() / 1000) + (7 * 365 * 24 * 60 * 60) // 7 years
    };

    // Store in DynamoDB
    await this.docClient.send(new PutCommand({
      TableName: this.tableName,
      Item: auditEntry
    }));

    // Also log to CloudWatch
    await this.logToCloudWatch(auditEntry);

    return auditEntry;
  }

  async logToCloudWatch(entry) {
    try {
      await this.cwLogs.send(new PutLogEventsCommand({
        logGroupName: this.logGroupName,
        logStreamName: new Date().toISOString().split('T')[0],
        logEvents: [{
          message: JSON.stringify(entry),
          timestamp: Date.now()
        }]
      }));
    } catch (error) {
      console.error('CloudWatch logging failed:', error);
    }
  }

  // Specific audit methods
  async logLogin(userId, success, ipAddress, userAgent) {
    return this.logEvent({
      userId,
      eventType: 'AUTHENTICATION',
      action: 'LOGIN',
      resource: 'USER',
      resourceId: userId,
      ipAddress,
      userAgent,
      status: success ? 'SUCCESS' : 'FAILED'
    });
  }

  async logTransaction(userId, transaction) {
    return this.logEvent({
      userId,
      eventType: 'FINANCIAL_TRANSACTION',
      action: transaction.type,
      resource: 'ACCOUNT',
      resourceId: transaction.fromAccount,
      status: transaction.status,
      details: {
        amount: transaction.amount,
        toAccount: transaction.toAccount,
        reference: transaction.reference
      }
    });
  }

  async logDataAccess(userId, resource, action, resourceId) {
    return this.logEvent({
      userId,
      eventType: 'DATA_ACCESS',
      action,
      resource,
      resourceId,
      status: 'SUCCESS'
    });
  }

  async logSecurityEvent(eventType, details) {
    return this.logEvent({
      userId: 'SECURITY_SYSTEM',
      eventType: 'SECURITY_INCIDENT',
      action: eventType,
      resource: 'SYSTEM',
      status: 'ALERT',
      details
    });
  }
}

module.exports = new AuditLogger();
```

### 🚨 Security Event Detector

```javascript
// lambdas/security-detector/index.js
const { SNSClient, PublishCommand } = require('@aws-sdk/client-sns');

const sns = new SNSClient({});
const ALERT_TOPIC = process.env.SNS_TOPIC_ARN;

exports.handler = async (event) => {
  for (const record of event.Records) {
    const logEvent = JSON.parse(record.dynamodb.NewImage.details.S);
    
    // Detect suspicious patterns
    if (await detectSuspiciousActivity(logEvent)) {
      await sendSecurityAlert(logEvent);
    }
  }
};

async function detectSuspiciousActivity(logEvent) {
  // Multiple failed login attempts
  if (logEvent.eventType === 'AUTHENTICATION' && logEvent.status === 'FAILED') {
    const failedAttempts = await countRecentFailedLogins(logEvent.userId);
    if (failedAttempts >= 5) {
      return { type: 'BRUTE_FORCE_ATTACK', severity: 'HIGH' };
    }
  }

  // Large transaction
  if (logEvent.eventType === 'FINANCIAL_TRANSACTION' && logEvent.details.amount > 100000) {
    return { type: 'LARGE_TRANSACTION', severity: 'MEDIUM' };
  }

  // Access from unusual location
  if (logEvent.ipAddress && await isUnusualLocation(logEvent.ipAddress, logEvent.userId)) {
    return { type: 'UNUSUAL_LOCATION', severity: 'HIGH' };
  }

  // Off-hours access
  const hour = new Date().getHours();
  if (hour < 6 || hour > 22) {
    return { type: 'OFF_HOURS_ACCESS', severity: 'LOW' };
  }

  return null;
}

async function sendSecurityAlert(logEvent) {
  await sns.send(new PublishCommand({
    TopicArn: ALERT_TOPIC,
    Subject: `Security Alert: ${logEvent.type}`,
    Message: JSON.stringify({
      alert: logEvent.type,
      severity: logEvent.severity,
      userId: logEvent.userId,
      timestamp: new Date().toISOString(),
      details: logEvent
    })
  }));
}
```

### 📜 PCI-DSS Compliance Checklist

```typescript
// compliance/pci-dss-checklist.ts

interface PCIDSSRequirement {
  id: string;
  requirement: string;
  implementation: string;
  status: 'COMPLIANT' | 'PARTIAL' | 'NON_COMPLIANT';
}

const pciDSSCompliance: PCIDSSRequirement[] = [
  {
    id: 'REQ-1',
    requirement: 'Install and maintain a firewall configuration',
    implementation: 'AWS WAF + Security Groups + NACLs',
    status: 'COMPLIANT'
  },
  {
    id: 'REQ-2',
    requirement: 'Do not use vendor-supplied defaults',
    implementation: 'Custom passwords, rotated secrets via Secrets Manager',
    status: 'COMPLIANT'
  },
  {
    id: 'REQ-3',
    requirement: 'Protect stored cardholder data',
    implementation: 'KMS encryption, tokenization, no PAN storage',
    status: 'COMPLIANT'
  },
  {
    id: 'REQ-4',
    requirement: 'Encrypt transmission of cardholder data',
    implementation: 'TLS 1.3, VPC private networks',
    status: 'COMPLIANT'
  },
  {
    id: 'REQ-5',
    requirement: 'Protect systems against malware',
    implementation: 'GuardDuty, Inspector, patched AMIs',
    status: 'COMPLIANT'
  },
  {
    id: 'REQ-6',
    requirement: 'Develop secure systems and applications',
    implementation: 'SAST/DAST in CI/CD, vulnerability scanning',
    status: 'COMPLIANT'
  },
  {
    id: 'REQ-7',
    requirement: 'Restrict access by business need-to-know',
    implementation: 'IAM least privilege, role-based access',
    status: 'COMPLIANT'
  },
  {
    id: 'REQ-8',
    requirement: 'Identify and authenticate access',
    implementation: 'Cognito with MFA, IAM roles',
    status: 'COMPLIANT'
  },
  {
    id: 'REQ-9',
    requirement: 'Restrict physical access',
    implementation: 'AWS data center physical security',
    status: 'COMPLIANT'
  },
  {
    id: 'REQ-10',
    requirement: 'Track and monitor all access',
    implementation: 'CloudTrail, audit logs, DynamoDB audit table',
    status: 'COMPLIANT'
  },
  {
    id: 'REQ-11',
    requirement: 'Regularly test security systems',
    implementation: 'Penetration testing, vulnerability scans',
    status: 'PARTIAL'
  },
  {
    id: 'REQ-12',
    requirement: 'Maintain information security policy',
    implementation: 'Security policies documented in Confluence',
    status: 'COMPLIANT'
  }
];

export default pciDSSCompliance;
```

### 🎓 Interview Discussion Points - Q20

**Q1: What data must be audited for PCI-DSS?**

**A**:
- All access to cardholder data
- Administrative actions
- Failed access attempts
- Changes to authentication mechanisms
- Retain logs for 1 year (3 months online)

**Q2: How do you ensure log integrity?**

**A**:
- **Immutable storage**: Write to S3 with object lock
- **Encryption**: Encrypt logs at rest
- **Hash chains**: Link logs cryptographically
- **Centralized**: Don't allow local log modification

**Q3: What are common security incidents to detect?**

**A**:
- Multiple failed logins (brute force)
- Unusual access patterns (time, location)
- Large transactions
- Privilege escalation
- Data exfiltration

**Q4: How do you handle a security breach?**

**A**:
1. **Contain**: Isolate affected systems
2. **Investigate**: Analyze logs, determine scope
3. **Remediate**: Patch vulnerabilities
4. **Notify**: Inform affected parties
5. **Review**: Update security policies

---

**End of File 10 - Security & IAM**