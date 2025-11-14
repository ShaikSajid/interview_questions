# AWS Interview Questions: Advanced Topics (Q19-Q21)

## Question 19: Explain AWS Cognito for authentication and user management

### 📋 Answer

**Amazon Cognito** provides authentication, authorization, and user management for web and mobile applications.

### Cognito Components:

```
Amazon Cognito
├── User Pools
│   ├── Sign-up & Sign-in
│   ├── User Directory
│   ├── MFA & Password Policies
│   └── Social & Enterprise Identity
└── Identity Pools (Federated Identities)
    ├── AWS Credentials
    ├── Temporary IAM Roles
    └── Unauthenticated Access
```

### Complete Cognito Implementation:

```javascript
// cognito-auth-service.js
import {
  CognitoIdentityProviderClient,
  SignUpCommand,
  ConfirmSignUpCommand,
  InitiateAuthCommand,
  RespondToAuthChallengeCommand,
  GetUserCommand,
  UpdateUserAttributesCommand,
  ChangePasswordCommand,
  ForgotPasswordCommand,
  ConfirmForgotPasswordCommand,
  GlobalSignOutCommand
} from '@aws-sdk/client-cognito-identity-provider';
import {
  CognitoIdentityClient,
  GetIdCommand,
  GetCredentialsForIdentityCommand
} from '@aws-sdk/client-cognito-identity';
import crypto from 'crypto';

const cognitoClient = new CognitoIdentityProviderClient({ region: 'us-east-1' });
const identityClient = new CognitoIdentityClient({ region: 'us-east-1' });

class CognitoAuthService {
  constructor(userPoolId, clientId, clientSecret, identityPoolId) {
    this.userPoolId = userPoolId;
    this.clientId = clientId;
    this.clientSecret = clientSecret;
    this.identityPoolId = identityPoolId;
  }
  
  // Generate SECRET_HASH for client secret
  generateSecretHash(username) {
    const message = username + this.clientId;
    return crypto
      .createHmac('sha256', this.clientSecret)
      .update(message)
      .digest('base64');
  }
  
  // 1. Sign Up User
  async signUp(username, password, email, attributes = {}) {
    const userAttributes = [
      { Name: 'email', Value: email },
      ...Object.entries(attributes).map(([key, value]) => ({
        Name: key,
        Value: value
      }))
    ];
    
    const command = new SignUpCommand({
      ClientId: this.clientId,
      Username: username,
      Password: password,
      UserAttributes: userAttributes,
      SecretHash: this.generateSecretHash(username)
    });
    
    try {
      const response = await cognitoClient.send(command);
      console.log('User signed up:', username);
      return {
        userSub: response.UserSub,
        userConfirmed: response.UserConfirmed,
        codeDeliveryDetails: response.CodeDeliveryDetails
      };
    } catch (error) {
      console.error('Sign up failed:', error);
      throw error;
    }
  }
  
  // 2. Confirm Sign Up
  async confirmSignUp(username, confirmationCode) {
    const command = new ConfirmSignUpCommand({
      ClientId: this.clientId,
      Username: username,
      ConfirmationCode: confirmationCode,
      SecretHash: this.generateSecretHash(username)
    });
    
    try {
      await cognitoClient.send(command);
      console.log('User confirmed:', username);
    } catch (error) {
      console.error('Confirmation failed:', error);
      throw error;
    }
  }
  
  // 3. Sign In
  async signIn(username, password) {
    const command = new InitiateAuthCommand({
      AuthFlow: 'USER_PASSWORD_AUTH',
      ClientId: this.clientId,
      AuthParameters: {
        USERNAME: username,
        PASSWORD: password,
        SECRET_HASH: this.generateSecretHash(username)
      }
    });
    
    try {
      const response = await cognitoClient.send(command);
      
      if (response.ChallengeName) {
        return {
          challengeName: response.ChallengeName,
          session: response.Session,
          challengeParameters: response.ChallengeParameters
        };
      }
      
      return {
        accessToken: response.AuthenticationResult.AccessToken,
        idToken: response.AuthenticationResult.IdToken,
        refreshToken: response.AuthenticationResult.RefreshToken,
        expiresIn: response.AuthenticationResult.ExpiresIn
      };
    } catch (error) {
      console.error('Sign in failed:', error);
      throw error;
    }
  }
  
  // 4. Refresh Token
  async refreshToken(username, refreshToken) {
    const command = new InitiateAuthCommand({
      AuthFlow: 'REFRESH_TOKEN_AUTH',
      ClientId: this.clientId,
      AuthParameters: {
        REFRESH_TOKEN: refreshToken,
        SECRET_HASH: this.generateSecretHash(username)
      }
    });
    
    const response = await cognitoClient.send(command);
    
    return {
      accessToken: response.AuthenticationResult.AccessToken,
      idToken: response.AuthenticationResult.IdToken,
      expiresIn: response.AuthenticationResult.ExpiresIn
    };
  }
  
  // 5. Get User Info
  async getUserInfo(accessToken) {
    const command = new GetUserCommand({
      AccessToken: accessToken
    });
    
    const response = await cognitoClient.send(command);
    
    return {
      username: response.Username,
      attributes: response.UserAttributes.reduce((acc, attr) => {
        acc[attr.Name] = attr.Value;
        return acc;
      }, {}),
      mfaEnabled: response.UserMFASettingList
    };
  }
  
  // 6. Update User Attributes
  async updateUserAttributes(accessToken, attributes) {
    const userAttributes = Object.entries(attributes).map(([key, value]) => ({
      Name: key,
      Value: value
    }));
    
    const command = new UpdateUserAttributesCommand({
      AccessToken: accessToken,
      UserAttributes: userAttributes
    });
    
    await cognitoClient.send(command);
    console.log('User attributes updated');
  }
  
  // 7. Change Password
  async changePassword(accessToken, oldPassword, newPassword) {
    const command = new ChangePasswordCommand({
      AccessToken: accessToken,
      PreviousPassword: oldPassword,
      ProposedPassword: newPassword
    });
    
    try {
      await cognitoClient.send(command);
      console.log('Password changed successfully');
    } catch (error) {
      console.error('Password change failed:', error);
      throw error;
    }
  }
  
  // 8. Forgot Password
  async forgotPassword(username) {
    const command = new ForgotPasswordCommand({
      ClientId: this.clientId,
      Username: username,
      SecretHash: this.generateSecretHash(username)
    });
    
    const response = await cognitoClient.send(command);
    console.log('Password reset code sent');
    return response.CodeDeliveryDetails;
  }
  
  // 9. Confirm Forgot Password
  async confirmForgotPassword(username, confirmationCode, newPassword) {
    const command = new ConfirmForgotPasswordCommand({
      ClientId: this.clientId,
      Username: username,
      ConfirmationCode: confirmationCode,
      Password: newPassword,
      SecretHash: this.generateSecretHash(username)
    });
    
    await cognitoClient.send(command);
    console.log('Password reset successful');
  }
  
  // 10. Sign Out
  async signOut(accessToken) {
    const command = new GlobalSignOutCommand({
      AccessToken: accessToken
    });
    
    await cognitoClient.send(command);
    console.log('User signed out');
  }
  
  // 11. Get AWS Credentials (Identity Pool)
  async getAWSCredentials(idToken) {
    // Get Identity ID
    const getIdCommand = new GetIdCommand({
      IdentityPoolId: this.identityPoolId,
      Logins: {
        [`cognito-idp.us-east-1.amazonaws.com/${this.userPoolId}`]: idToken
      }
    });
    
    const { IdentityId } = await identityClient.send(getIdCommand);
    
    // Get Credentials
    const getCredsCommand = new GetCredentialsForIdentityCommand({
      IdentityId: IdentityId,
      Logins: {
        [`cognito-idp.us-east-1.amazonaws.com/${this.userPoolId}`]: idToken
      }
    });
    
    const response = await identityClient.send(getCredsCommand);
    
    return {
      accessKeyId: response.Credentials.AccessKeyId,
      secretKey: response.Credentials.SecretKey,
      sessionToken: response.Credentials.SessionToken,
      expiration: response.Credentials.Expiration
    };
  }
}

// Usage
const authService = new CognitoAuthService(
  'us-east-1_abcdefghi',  // User Pool ID
  '1234567890abcdefghijklmnop',  // Client ID
  'client-secret-here',  // Client Secret
  'us-east-1:12345678-1234-1234-1234-123456789012'  // Identity Pool ID
);

// Sign up
await authService.signUp('john.doe', 'SecurePass123!', 'john@example.com', {
  'name': 'John Doe',
  'phone_number': '+1234567890'
});

// Confirm sign up
await authService.confirmSignUp('john.doe', '123456');

// Sign in
const tokens = await authService.signIn('john.doe', 'SecurePass123!');

// Get user info
const userInfo = await authService.getUserInfo(tokens.accessToken);
console.log('User info:', userInfo);
```

### Express.js Integration:

```javascript
// auth-middleware.js
import jwt from 'jsonwebtoken';
import jwksClient from 'jwks-rsa';

const userPoolId = 'us-east-1_abcdefghi';
const region = 'us-east-1';
const issuer = `https://cognito-idp.${region}.amazonaws.com/${userPoolId}`;

// JWKS client for token verification
const client = jwksClient({
  jwksUri: `${issuer}/.well-known/jwks.json`
});

function getKey(header, callback) {
  client.getSigningKey(header.kid, (err, key) => {
    if (err) {
      callback(err);
    } else {
      const signingKey = key.getPublicKey();
      callback(null, signingKey);
    }
  });
}

// Verify JWT token middleware
export function authenticateToken(req, res, next) {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];  // Bearer TOKEN
  
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }
  
  jwt.verify(token, getKey, {
    issuer: issuer,
    algorithms: ['RS256']
  }, (err, decoded) => {
    if (err) {
      console.error('Token verification failed:', err);
      return res.status(403).json({ error: 'Invalid token' });
    }
    
    req.user = {
      sub: decoded.sub,
      username: decoded['cognito:username'],
      email: decoded.email,
      groups: decoded['cognito:groups'] || []
    };
    
    next();
  });
}

// Check user groups
export function requireGroup(...allowedGroups) {
  return (req, res, next) => {
    const userGroups = req.user?.groups || [];
    const hasAccess = allowedGroups.some(group => userGroups.includes(group));
    
    if (!hasAccess) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    
    next();
  };
}

// Usage in Express app
import express from 'express';
const app = express();

// Protected route
app.get('/api/profile', authenticateToken, async (req, res) => {
  try {
    const user = await getUserProfile(req.user.sub);
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: 'Failed to get profile' });
  }
});

// Admin-only route
app.post('/api/admin/users', 
  authenticateToken, 
  requireGroup('Admins'), 
  async (req, res) => {
    // Only users in 'Admins' group can access
    res.json({ message: 'Admin action executed' });
  }
);
```

### Best Practices:

1. ✅ Enable MFA for sensitive accounts
2. ✅ Use custom attributes for application data
3. ✅ Implement password policies
4. ✅ Use Lambda triggers for custom workflows
5. ✅ Enable advanced security features
6. ✅ Use groups for authorization
7. ✅ Implement token refresh logic
8. ✅ Use Identity Pools for AWS access
9. ✅ Enable account takeover protection
10. ✅ Monitor with CloudWatch Insights

---

## Question 20: Explain AWS Secrets Manager and Parameter Store for secrets management

### 📋 Answer

**AWS Secrets Manager** and **Systems Manager Parameter Store** help you manage sensitive configuration data and secrets.

### Secrets Manager vs Parameter Store:

| Feature | Secrets Manager | Parameter Store |
|---------|----------------|-----------------|
| **Cost** | $0.40/secret/month | Free (Standard), $0.05 (Advanced) |
| **Rotation** | Built-in | Manual |
| **Versioning** | Automatic | Yes |
| **Size Limit** | 64 KB | 8 KB (Standard), 64 KB (Advanced) |
| **Encryption** | Always encrypted | Optional |
| **Cross-region** | Yes | No |
| **Use Case** | Database credentials, API keys | Configuration, non-rotated secrets |

### Complete Secrets Manager Implementation:

```javascript
// secrets-manager.js
import {
  SecretsManagerClient,
  CreateSecretCommand,
  GetSecretValueCommand,
  UpdateSecretCommand,
  DeleteSecretCommand,
  ListSecretsCommand,
  RotateSecretCommand,
  PutSecretValueCommand
} from '@aws-sdk/client-secrets-manager';

const secretsClient = new SecretsManagerClient({ region: 'us-east-1' });

// 1. Create Secret
async function createSecret(secretName, secretValue, description) {
  const command = new CreateSecretCommand({
    Name: secretName,
    Description: description,
    SecretString: JSON.stringify(secretValue),
    Tags: [
      { Key: 'Environment', Value: 'Production' },
      { Key: 'Application', Value: 'WebApp' }
    ]
  });
  
  try {
    const response = await secretsClient.send(command);
    console.log('Secret created:', response.ARN);
    return response;
  } catch (error) {
    console.error('Failed to create secret:', error);
    throw error;
  }
}

// 2. Get Secret Value
async function getSecret(secretName) {
  const command = new GetSecretValueCommand({
    SecretId: secretName
  });
  
  try {
    const response = await secretsClient.send(command);
    
    if (response.SecretString) {
      return JSON.parse(response.SecretString);
    } else {
      // Binary secret
      return Buffer.from(response.SecretBinary);
    }
  } catch (error) {
    console.error('Failed to get secret:', error);
    throw error;
  }
}

// 3. Update Secret
async function updateSecret(secretName, newValue) {
  const command = new PutSecretValueCommand({
    SecretId: secretName,
    SecretString: JSON.stringify(newValue)
  });
  
  const response = await secretsClient.send(command);
  console.log('Secret updated:', response.VersionId);
  return response;
}

// 4. Delete Secret
async function deleteSecret(secretName, recoveryWindowInDays = 30) {
  const command = new DeleteSecretCommand({
    SecretId: secretName,
    RecoveryWindowInDays: recoveryWindowInDays  // 7-30 days
  });
  
  const response = await secretsClient.send(command);
  console.log('Secret scheduled for deletion:', response.DeletionDate);
  return response;
}

// 5. List Secrets
async function listSecrets() {
  const command = new ListSecretsCommand({
    MaxResults: 100
  });
  
  const response = await secretsClient.send(command);
  
  return response.SecretList.map(secret => ({
    name: secret.Name,
    arn: secret.ARN,
    lastChangedDate: secret.LastChangedDate,
    lastAccessedDate: secret.LastAccessedDate
  }));
}

// 6. Rotate Secret
async function rotateSecret(secretName, lambdaArn) {
  const command = new RotateSecretCommand({
    SecretId: secretName,
    RotationLambdaARN: lambdaArn,
    RotationRules: {
      AutomaticallyAfterDays: 30
    }
  });
  
  const response = await secretsClient.send(command);
  console.log('Secret rotation initiated');
  return response;
}
```

### Parameter Store Implementation:

```javascript
// parameter-store.js
import {
  SSMClient,
  PutParameterCommand,
  GetParameterCommand,
  GetParametersCommand,
  GetParametersByPathCommand,
  DeleteParameterCommand,
  DescribeParametersCommand
} from '@aws-sdk/client-ssm';

const ssmClient = new SSMClient({ region: 'us-east-1' });

// 1. Put Parameter
async function putParameter(name, value, type = 'String', encrypted = false) {
  const command = new PutParameterCommand({
    Name: name,
    Value: value,
    Type: encrypted ? 'SecureString' : type,  // String, StringList, SecureString
    Description: 'Application parameter',
    Tags: [
      { Key: 'Environment', Value: 'Production' }
    ],
    Tier: 'Standard',  // Standard (free) or Advanced ($0.05/parameter/month)
    Overwrite: true
  });
  
  try {
    const response = await ssmClient.send(command);
    console.log('Parameter saved:', name);
    return response;
  } catch (error) {
    console.error('Failed to put parameter:', error);
    throw error;
  }
}

// 2. Get Parameter
async function getParameter(name, decrypt = true) {
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
      lastModifiedDate: response.Parameter.LastModifiedDate
    };
  } catch (error) {
    console.error('Failed to get parameter:', error);
    throw error;
  }
}

// 3. Get Multiple Parameters
async function getParameters(names, decrypt = true) {
  const command = new GetParametersCommand({
    Names: names,
    WithDecryption: decrypt
  });
  
  const response = await ssmClient.send(command);
  
  return response.Parameters.reduce((acc, param) => {
    acc[param.Name] = param.Value;
    return acc;
  }, {});
}

// 4. Get Parameters by Path (hierarchical)
async function getParametersByPath(path, decrypt = true) {
  const command = new GetParametersByPathCommand({
    Path: path,
    Recursive: true,
    WithDecryption: decrypt
  });
  
  const response = await ssmClient.send(command);
  
  return response.Parameters.reduce((acc, param) => {
    acc[param.Name] = param.Value;
    return acc;
  }, {});
}

// 5. Delete Parameter
async function deleteParameter(name) {
  const command = new DeleteParameterCommand({
    Name: name
  });
  
  await ssmClient.send(command);
  console.log('Parameter deleted:', name);
}
```

### Configuration Management Service:

```javascript
// config-service.js
class ConfigService {
  constructor() {
    this.cache = new Map();
    this.cacheTTL = 5 * 60 * 1000;  // 5 minutes
  }
  
  // Get configuration with caching
  async getConfig(key, useCache = true) {
    if (useCache && this.cache.has(key)) {
      const cached = this.cache.get(key);
      
      if (Date.now() - cached.timestamp < this.cacheTTL) {
        console.log('Cache hit:', key);
        return cached.value;
      }
    }
    
    console.log('Cache miss, fetching:', key);
    
    try {
      // Try Secrets Manager first
      const value = await getSecret(key);
      this.cache.set(key, { value, timestamp: Date.now() });
      return value;
    } catch (error) {
      // Fallback to Parameter Store
      const param = await getParameter(key);
      const value = param.value;
      this.cache.set(key, { value, timestamp: Date.now() });
      return value;
    }
  }
  
  // Get database credentials
  async getDatabaseConfig() {
    const config = await this.getConfig('/myapp/database');
    return config;
  }
  
  // Get API keys
  async getAPIKey(service) {
    const key = await this.getConfig(`/myapp/api-keys/${service}`);
    return key;
  }
  
  // Get all app config
  async getAllConfig() {
    const params = await getParametersByPath('/myapp/config/');
    return params;
  }
  
  // Refresh cache
  invalidateCache(key) {
    if (key) {
      this.cache.delete(key);
    } else {
      this.cache.clear();
    }
  }
}

// Usage
const config = new ConfigService();

// Application startup
async function initializeApp() {
  // Get database config
  const dbConfig = await config.getDatabaseConfig();
  console.log('Database config loaded');
  
  // Get API keys
  const stripeKey = await config.getAPIKey('stripe');
  const sendgridKey = await config.getAPIKey('sendgrid');
  
  // Get all configuration
  const appConfig = await config.getAllConfig();
  
  return {
    database: dbConfig,
    apiKeys: {
      stripe: stripeKey,
      sendgrid: sendgridKey
    },
    config: appConfig
  };
}
```

### Lambda Rotation Function:

```javascript
// rotation-lambda.js
import { RDSClient, ModifyDBInstanceCommand } from '@aws-sdk/client-rds';

export const handler = async (event) => {
  const token = event.Token;
  const step = event.Step;
  const secretArn = event.SecretId;
  
  switch (step) {
    case 'createSecret':
      await createSecret(secretArn, token);
      break;
    case 'setSecret':
      await setSecret(secretArn, token);
      break;
    case 'testSecret':
      await testSecret(secretArn, token);
      break;
    case 'finishSecret':
      await finishSecret(secretArn, token);
      break;
    default:
      throw new Error(`Invalid step: ${step}`);
  }
};

async function createSecret(secretArn, token) {
  // Generate new password
  const newPassword = generateSecurePassword();
  
  // Store AWSPENDING version
  await updateSecret(secretArn, { password: newPassword }, 'AWSPENDING');
  
  console.log('New secret version created');
}

async function setSecret(secretArn, token) {
  // Get pending secret
  const pendingSecret = await getSecret(secretArn, 'AWSPENDING');
  
  // Update database password
  const rdsClient = new RDSClient({ region: 'us-east-1' });
  await rdsClient.send(new ModifyDBInstanceCommand({
    DBInstanceIdentifier: 'my-database',
    MasterUserPassword: pendingSecret.password,
    ApplyImmediately: true
  }));
  
  console.log('Database password updated');
}

async function testSecret(secretArn, token) {
  // Get pending secret
  const pendingSecret = await getSecret(secretArn, 'AWSPENDING');
  
  // Test connection with new password
  await testDatabaseConnection(pendingSecret);
  
  console.log('New password tested successfully');
}

async function finishSecret(secretArn, token) {
  // Mark AWSPENDING as AWSCURRENT
  console.log('Rotation complete');
}

function generateSecurePassword() {
  // Implementation
  return 'SecurePassword123!';
}
```

### Best Practices:

1. ✅ Use Secrets Manager for credentials that rotate
2. ✅ Use Parameter Store for static configuration
3. ✅ Implement caching to reduce API calls
4. ✅ Enable automatic rotation for databases
5. ✅ Use hierarchical parameter names
6. ✅ Tag secrets for organization
7. ✅ Use IAM policies for access control
8. ✅ Enable CloudTrail logging
9. ✅ Use SecureString for sensitive data
10. ✅ Implement graceful fallback

---

## Question 21: Explain AWS X-Ray for distributed tracing

### 📋 Answer

**AWS X-Ray** helps you analyze and debug distributed applications by providing end-to-end tracing of requests.

### X-Ray Concepts:

```
X-Ray Tracing
├── Trace (End-to-end request flow)
├── Segments (Service execution)
├── Subsegments (Operations within service)
├── Annotations (Indexed metadata)
├── Metadata (Non-indexed data)
└── Service Map (Visual representation)
```

### Complete X-Ray Implementation:

```javascript
// x-ray-instrumentation.js
import AWSXRay from 'aws-xray-sdk-core';
import AWS from 'aws-sdk';
import http from 'http';
import express from 'express';

// Instrument AWS SDK
const AWSX = AWSXRay.captureAWS(AWS);
const dynamoDB = new AWSX.DynamoDB.DocumentClient();
const s3 = new AWSX.S3();

// Instrument HTTP client
AWSXRay.captureHTTPsGlobal(http);

// Express app with X-Ray middleware
const app = express();

// Add X-Ray middleware (must be first)
app.use(AWSXRay.express.openSegment('MyApplication'));

// Your routes
app.get('/api/users/:id', async (req, res) => {
  const userId = req.params.id;
  
  try {
    // Create subsegment for database query
    const segment = AWSXRay.getSegment();
    const subsegment = segment.addNewSubsegment('GetUserFromDB');
    
    // Add annotations (indexed, searchable)
    subsegment.addAnnotation('userId', userId);
    subsegment.addAnnotation('operation', 'GetUser');
    
    // Add metadata (not indexed)
    subsegment.addMetadata('requestTime', new Date().toISOString());
    
    try {
      // Database query
      const user = await dynamoDB.get({
        TableName: 'Users',
        Key: { id: userId }
      }).promise();
      
      subsegment.close();
      
      // Create subsegment for S3
      const s3Subsegment = segment.addNewSubsegment('GetUserAvatar');
      
      const avatar = await s3.getObject({
        Bucket: 'user-avatars',
        Key: `${userId}/avatar.jpg`
      }).promise();
      
      s3Subsegment.close();
      
      res.json({
        user: user.Item,
        avatar: avatar.Body.toString('base64')
      });
      
    } catch (error) {
      subsegment.addError(error);
      subsegment.close();
      throw error;
    }
    
  } catch (error) {
    console.error('Error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Close X-Ray segment
app.use(AWSXRay.express.closeSegment());

app.listen(3000);
```

### Custom Instrumentation:

```javascript
// custom-tracing.js
import AWSXRay from 'aws-xray-sdk-core';

class OrderService {
  async processOrder(orderId) {
    const segment = AWSXRay.getSegment();
    
    // Create subsegment for order processing
    const subsegment = segment.addNewSubsegment('ProcessOrder');
    subsegment.addAnnotation('orderId', orderId);
    
    try {
      // Validate order
      await this.captureAsync('ValidateOrder', async () => {
        await this.validateOrder(orderId);
      });
      
      // Check inventory
      await this.captureAsync('CheckInventory', async () => {
        await this.checkInventory(orderId);
      });
      
      // Process payment
      await this.captureAsync('ProcessPayment', async () => {
        await this.processPayment(orderId);
      });
      
      // Create shipment
      await this.captureAsync('CreateShipment', async () => {
        await this.createShipment(orderId);
      });
      
      subsegment.close();
      
      return { success: true, orderId };
      
    } catch (error) {
      subsegment.addError(error);
      subsegment.close();
      throw error;
    }
  }
  
  async captureAsync(name, fn) {
    const segment = AWSXRay.getSegment();
    const subsegment = segment.addNewSubsegment(name);
    
    try {
      const result = await fn();
      subsegment.close();
      return result;
    } catch (error) {
      subsegment.addError(error);
      subsegment.close();
      throw error;
    }
  }
  
  async validateOrder(orderId) {
    // Implementation
  }
  
  async checkInventory(orderId) {
    // Implementation
  }
  
  async processPayment(orderId) {
    // Implementation
  }
  
  async createShipment(orderId) {
    // Implementation
  }
}
```

### Lambda Integration:

```javascript
// lambda-with-xray.js
import AWSXRay from 'aws-xray-sdk-core';
import AWS from 'aws-sdk';

const AWSX = AWSXRay.captureAWS(AWS);
const dynamoDB = new AWSX.DynamoDB.DocumentClient();

export const handler = async (event) => {
  console.log('Event:', JSON.stringify(event, null, 2));
  
  const segment = AWSXRay.getSegment();
  
  // Add annotations
  segment.addAnnotation('userId', event.userId);
  segment.addAnnotation('operation', 'CreateUser');
  
  // Add metadata
  segment.addMetadata('event', event);
  
  try {
    // Create subsegment for database operation
    const dbSubsegment = segment.addNewSubsegment('DynamoDB.PutItem');
    
    try {
      await dynamoDB.put({
        TableName: 'Users',
        Item: {
          id: event.userId,
          name: event.name,
          email: event.email,
          createdAt: new Date().toISOString()
        }
      }).promise();
      
      dbSubsegment.close();
      
    } catch (error) {
      dbSubsegment.addError(error);
      dbSubsegment.close();
      throw error;
    }
    
    // Call external API
    const apiSubsegment = segment.addNewSubsegment('ExternalAPI');
    
    try {
      const response = await fetch('https://api.example.com/notify', {
        method: 'POST',
        body: JSON.stringify({ userId: event.userId })
      });
      
      apiSubsegment.close();
      
    } catch (error) {
      apiSubsegment.addError(error);
      apiSubsegment.close();
    }
    
    return {
      statusCode: 200,
      body: JSON.stringify({ success: true })
    };
    
  } catch (error) {
    console.error('Error:', error);
    
    return {
      statusCode: 500,
      body: JSON.stringify({ error: error.message })
    };
  }
};
```

### X-Ray SDK Configuration:

```javascript
// xray-config.js
import AWSXRay from 'aws-xray-sdk-core';

// Configure X-Ray
AWSXRay.config([
  AWSXRay.plugins.ECSPlugin,
  AWSXRay.plugins.EC2Plugin
]);

// Set sampling rules
AWSXRay.middleware.setSamplingRules({
  version: 2,
  rules: [
    {
      description: 'High priority requests',
      host: '*',
      http_method: '*',
      url_path: '/api/critical/*',
      fixed_target: 1,
      rate: 1.0  // 100% sampling
    },
    {
      description: 'Default rule',
      host: '*',
      http_method: '*',
      url_path: '*',
      fixed_target: 1,
      rate: 0.1  // 10% sampling
    }
  ],
  default: {
    fixed_target: 1,
    rate: 0.05  // 5% sampling
  }
});

// Set daemon address
AWSXRay.setDaemonAddress('127.0.0.1:2000');

// Enable automatic mode
AWSXRay.enableAutomaticMode();
```

### Query Traces:

```javascript
// query-xray.js
import {
  XRayClient,
  GetTraceSummariesCommand,
  BatchGetTracesCommand,
  GetServiceGraphCommand
} from '@aws-sdk/client-xray';

const xrayClient = new XRayClient({ region: 'us-east-1' });

// Get trace summaries
async function getTraceSummaries(startTime, endTime, filterExpression) {
  const command = new GetTraceSummariesCommand({
    StartTime: startTime,
    EndTime: endTime,
    FilterExpression: filterExpression,
    Sampling: false
  });
  
  const response = await xrayClient.send(command);
  
  return response.TraceSummaries.map(trace => ({
    id: trace.Id,
    duration: trace.Duration,
    responseTime: trace.ResponseTime,
    hasError: trace.HasError,
    hasFault: trace.HasFault,
    http: trace.Http
  }));
}

// Get full traces
async function getTraces(traceIds) {
  const command = new BatchGetTracesCommand({
    TraceIds: traceIds
  });
  
  const response = await xrayClient.send(command);
  return response.Traces;
}

// Get service graph
async function getServiceGraph(startTime, endTime) {
  const command = new GetServiceGraphCommand({
    StartTime: startTime,
    EndTime: endTime
  });
  
  const response = await xrayClient.send(command);
  return response.Services;
}

// Usage
const now = new Date();
const oneHourAgo = new Date(now.getTime() - 60 * 60 * 1000);

// Find slow requests
const slowTraces = await getTraceSummaries(
  oneHourAgo,
  now,
  'duration > 5 AND service("MyApplication")'
);

// Find errors
const errorTraces = await getTraceSummaries(
  oneHourAgo,
  now,
  'error = true AND annotation.userId = "user123"'
);

// Get service map
const serviceGraph = await getServiceGraph(oneHourAgo, now);
```

### Best Practices:

1. ✅ Enable X-Ray on all services
2. ✅ Use sampling to control costs
3. ✅ Add meaningful annotations
4. ✅ Instrument external HTTP calls
5. ✅ Use subsegments for operations
6. ✅ Add error information to traces
7. ✅ Monitor service map regularly
8. ✅ Set up CloudWatch alarms
9. ✅ Use filter expressions for analysis
10. ✅ Integrate with CloudWatch Insights

---

## Key Takeaways

### Question 19 (Cognito):
- User authentication and authorization
- User Pools for user directory
- Identity Pools for AWS credentials
- Built-in MFA and password policies
- Social and enterprise identity federation

### Question 20 (Secrets Manager & Parameter Store):
- Secrets Manager for rotating credentials
- Parameter Store for static configuration
- Automatic secret rotation with Lambda
- Hierarchical parameter organization
- Encryption and IAM-based access control

### Question 21 (X-Ray):
- Distributed tracing for debugging
- Service map visualization
- Annotations for searchable metadata
- Subsegments for detailed tracking
- Sampling rules to control costs
- Integration with CloudWatch for monitoring
