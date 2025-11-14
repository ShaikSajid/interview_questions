# AWS Interview Questions: Advanced Services (Q34-Q36)

## Question 34: Explain Amazon CloudFront CDN and edge computing

### 📋 Answer

**Amazon CloudFront** is a fast content delivery network (CDN) that securely delivers data, videos, applications, and APIs with low latency and high transfer speeds.

### CloudFront Architecture:

```
CloudFront Architecture
├── Origins
│   ├── S3 Buckets
│   ├── HTTP Servers (EC2, ALB)
│   ├── MediaStore
│   └── Custom Origins
├── Edge Locations (400+ worldwide)
├── Regional Edge Caches
├── Behaviors
│   ├── Path patterns
│   ├── Cache policies
│   └── Origin request policies
└── Features
    ├── Lambda@Edge
    ├── CloudFront Functions
    ├── Signed URLs/Cookies
    └── Field-Level Encryption
```

### Complete CloudFront Implementation:

```javascript
// cloudfront-service.js
import {
  CloudFrontClient,
  CreateDistributionCommand,
  GetDistributionCommand,
  UpdateDistributionCommand,
  CreateInvalidationCommand,
  ListDistributionsCommand
} from '@aws-sdk/client-cloudfront';
import { getSignedUrl } from '@aws-sdk/cloudfront-signer';

const cloudFrontClient = new CloudFrontClient({ region: 'us-east-1' });

class CloudFrontService {
  // 1. Create Distribution
  async createDistribution(config) {
    const distributionConfig = {
      CallerReference: Date.now().toString(),
      Comment: config.comment || 'Created by automation',
      Enabled: true,
      DefaultRootObject: config.defaultRootObject || 'index.html',
      Origins: {
        Quantity: config.origins.length,
        Items: config.origins.map(origin => ({
          Id: origin.id,
          DomainName: origin.domainName,
          OriginPath: origin.path || '',
          CustomOriginConfig: origin.customOrigin ? {
            HTTPPort: 80,
            HTTPSPort: 443,
            OriginProtocolPolicy: 'https-only',
            OriginSslProtocols: {
              Quantity: 1,
              Items: ['TLSv1.2']
            }
          } : undefined,
          S3OriginConfig: origin.s3 ? {
            OriginAccessIdentity: origin.s3.oai || ''
          } : undefined
        }))
      },
      DefaultCacheBehavior: {
        TargetOriginId: config.origins[0].id,
        ViewerProtocolPolicy: 'redirect-to-https',
        AllowedMethods: {
          Quantity: 7,
          Items: ['GET', 'HEAD', 'OPTIONS', 'PUT', 'POST', 'PATCH', 'DELETE'],
          CachedMethods: {
            Quantity: 2,
            Items: ['GET', 'HEAD']
          }
        },
        CachePolicyId: config.cachePolicyId || '658327ea-f89d-4fab-a63d-7e88639e58f6',  // CachingOptimized
        OriginRequestPolicyId: config.originRequestPolicyId,
        Compress: true,
        TrustedSigners: {
          Enabled: false,
          Quantity: 0
        },
        MinTTL: 0,
        DefaultTTL: 86400,
        MaxTTL: 31536000
      },
      CacheBehaviors: {
        Quantity: config.cacheBehaviors?.length || 0,
        Items: config.cacheBehaviors || []
      },
      CustomErrorResponses: {
        Quantity: config.errorResponses?.length || 0,
        Items: config.errorResponses || []
      },
      ViewerCertificate: {
        CloudFrontDefaultCertificate: !config.acmCertificateArn,
        ACMCertificateArn: config.acmCertificateArn,
        SSLSupportMethod: config.acmCertificateArn ? 'sni-only' : undefined,
        MinimumProtocolVersion: 'TLSv1.2_2021'
      },
      Restrictions: {
        GeoRestriction: {
          RestrictionType: config.geoRestriction?.type || 'none',
          Quantity: config.geoRestriction?.locations?.length || 0,
          Items: config.geoRestriction?.locations || []
        }
      },
      PriceClass: config.priceClass || 'PriceClass_All',
      Aliases: {
        Quantity: config.aliases?.length || 0,
        Items: config.aliases || []
      }
    };
    
    const command = new CreateDistributionCommand({
      DistributionConfig: distributionConfig
    });
    
    try {
      const response = await cloudFrontClient.send(command);
      console.log('Distribution created:', response.Distribution.Id);
      return response.Distribution;
    } catch (error) {
      console.error('Failed to create distribution:', error);
      throw error;
    }
  }
  
  // 2. Create Invalidation
  async createInvalidation(distributionId, paths) {
    const command = new CreateInvalidationCommand({
      DistributionId: distributionId,
      InvalidationBatch: {
        CallerReference: Date.now().toString(),
        Paths: {
          Quantity: paths.length,
          Items: paths
        }
      }
    });
    
    try {
      const response = await cloudFrontClient.send(command);
      console.log('Invalidation created:', response.Invalidation.Id);
      return response.Invalidation;
    } catch (error) {
      console.error('Failed to create invalidation:', error);
      throw error;
    }
  }
  
  // 3. Generate Signed URL
  generateSignedUrl(url, keyPairId, privateKey, expirationMinutes = 60) {
    const expiration = new Date();
    expiration.setMinutes(expiration.getMinutes() + expirationMinutes);
    
    return getSignedUrl({
      url: url,
      keyPairId: keyPairId,
      dateLessThan: expiration.toISOString(),
      privateKey: privateKey
    });
  }
  
  // 4. Generate Signed Cookie
  generateSignedCookie(url, keyPairId, privateKey, expirationMinutes = 60) {
    const expiration = new Date();
    expiration.setMinutes(expiration.getMinutes() + expirationMinutes);
    
    // Implementation for signed cookies
    const policy = {
      Statement: [
        {
          Resource: url,
          Condition: {
            DateLessThan: {
              'AWS:EpochTime': Math.floor(expiration.getTime() / 1000)
            }
          }
        }
      ]
    };
    
    // Sign policy with private key
    // Return cookie values
    return {
      'CloudFront-Policy': Buffer.from(JSON.stringify(policy)).toString('base64'),
      'CloudFront-Signature': '...',  // Generated signature
      'CloudFront-Key-Pair-Id': keyPairId
    };
  }
  
  // 5. List Distributions
  async listDistributions() {
    const command = new ListDistributionsCommand({});
    const response = await cloudFrontClient.send(command);
    
    return response.DistributionList.Items.map(dist => ({
      id: dist.Id,
      domainName: dist.DomainName,
      status: dist.Status,
      enabled: dist.Enabled,
      comment: dist.Comment
    }));
  }
}

// Usage - S3 Static Website with CloudFront
const cloudfront = new CloudFrontService();

// Create distribution for S3 static website
const distribution = await cloudfront.createDistribution({
  comment: 'Static website distribution',
  defaultRootObject: 'index.html',
  origins: [
    {
      id: 'S3-my-website',
      domainName: 'my-website-bucket.s3.amazonaws.com',
      s3: {
        oai: 'origin-access-identity/cloudfront/ABCDEFGHIJKLMN'
      }
    }
  ],
  cacheBehaviors: [
    {
      PathPattern: '/api/*',
      TargetOriginId: 'S3-my-website',
      ViewerProtocolPolicy: 'https-only',
      AllowedMethods: {
        Quantity: 7,
        Items: ['GET', 'HEAD', 'OPTIONS', 'PUT', 'POST', 'PATCH', 'DELETE']
      },
      CachePolicyId: '4135ea2d-6df8-44a3-9df3-4b5a84be39ad',  // CachingDisabled
      MinTTL: 0,
      DefaultTTL: 0,
      MaxTTL: 0
    }
  ],
  errorResponses: [
    {
      ErrorCode: 404,
      ResponsePagePath: '/404.html',
      ResponseCode: '404',
      ErrorCachingMinTTL: 300
    },
    {
      ErrorCode: 403,
      ResponsePagePath: '/index.html',
      ResponseCode: '200',
      ErrorCachingMinTTL: 300
    }
  ],
  aliases: ['www.example.com', 'example.com'],
  acmCertificateArn: 'arn:aws:acm:us-east-1:123456789012:certificate/abc123',
  geoRestriction: {
    type: 'whitelist',
    locations: ['US', 'CA', 'GB']
  },
  priceClass: 'PriceClass_100'  // US, Canada, Europe
});

// Create cache invalidation
await cloudfront.createInvalidation(distribution.Id, [
  '/index.html',
  '/assets/*',
  '/*'
]);

// Generate signed URL for premium content
const signedUrl = cloudfront.generateSignedUrl(
  'https://d111111abcdef8.cloudfront.net/premium/video.mp4',
  'APKAEIBAERJR2EXAMPLE',
  privateKeyString,
  120  // 2 hours
);

console.log('Signed URL:', signedUrl);
```

### Lambda@Edge Implementation:

```javascript
// lambda-edge-viewer-request.js
// Runs on every viewer request (before cache lookup)

export const handler = async (event) => {
  const request = event.Records[0].cf.request;
  const headers = request.headers;
  
  // 1. Authentication check
  const authHeader = headers['authorization'];
  if (!authHeader || !authHeader[0] || !isValidToken(authHeader[0].value)) {
    return {
      status: '401',
      statusDescription: 'Unauthorized',
      headers: {
        'www-authenticate': [{ key: 'WWW-Authenticate', value: 'Basic' }]
      }
    };
  }
  
  // 2. A/B Testing
  const cookie = headers['cookie'];
  if (!cookie || !cookie[0].value.includes('experiment=')) {
    const variant = Math.random() < 0.5 ? 'A' : 'B';
    headers['cookie'] = [{
      key: 'Cookie',
      value: `experiment=${variant}`
    }];
  }
  
  // 3. Redirect mobile users
  const userAgent = headers['user-agent'][0].value;
  if (isMobileDevice(userAgent)) {
    return {
      status: '302',
      statusDescription: 'Found',
      headers: {
        'location': [{ 
          key: 'Location', 
          value: 'https://m.example.com' + request.uri 
        }]
      }
    };
  }
  
  // 4. URL Rewrite
  if (request.uri === '/') {
    request.uri = '/index.html';
  } else if (!request.uri.includes('.')) {
    request.uri += '.html';
  }
  
  return request;
};

function isValidToken(token) {
  // Token validation logic
  return true;
}

function isMobileDevice(userAgent) {
  return /mobile|android|iphone|ipad/i.test(userAgent);
}
```

```javascript
// lambda-edge-origin-response.js
// Runs after receiving response from origin

export const handler = async (event) => {
  const response = event.Records[0].cf.response;
  const headers = response.headers;
  
  // 1. Add security headers
  headers['strict-transport-security'] = [{
    key: 'Strict-Transport-Security',
    value: 'max-age=63072000; includeSubdomains; preload'
  }];
  
  headers['x-content-type-options'] = [{
    key: 'X-Content-Type-Options',
    value: 'nosniff'
  }];
  
  headers['x-frame-options'] = [{
    key: 'X-Frame-Options',
    value: 'DENY'
  }];
  
  headers['x-xss-protection'] = [{
    key: 'X-XSS-Protection',
    value: '1; mode=block'
  }];
  
  headers['content-security-policy'] = [{
    key: 'Content-Security-Policy',
    value: "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'"
  }];
  
  // 2. Add caching headers
  const request = event.Records[0].cf.request;
  if (request.uri.match(/\.(jpg|jpeg|png|gif|css|js)$/)) {
    headers['cache-control'] = [{
      key: 'Cache-Control',
      value: 'public, max-age=31536000, immutable'
    }];
  }
  
  // 3. Add custom headers
  headers['x-powered-by'] = [{
    key: 'X-Powered-By',
    value: 'CloudFront Lambda@Edge'
  }];
  
  return response;
};
```

### CloudFront Functions (lighter, faster than Lambda@Edge):

```javascript
// cloudfront-function-url-rewrite.js
function handler(event) {
  var request = event.request;
  var uri = request.uri;
  
  // Redirect root to /index.html
  if (uri === '/') {
    request.uri = '/index.html';
  }
  
  // Add .html to paths without extension
  else if (!uri.includes('.') && !uri.endsWith('/')) {
    request.uri = uri + '.html';
  }
  
  // Pretty URLs: /blog/post-title -> /blog/post-title/index.html
  else if (uri.endsWith('/')) {
    request.uri = uri + 'index.html';
  }
  
  return request;
}
```

### Express.js Integration with Signed URLs:

```javascript
// cloudfront-express.js
import express from 'express';
import { CloudFrontService } from './cloudfront-service.js';

const app = express();
const cloudfront = new CloudFrontService();

// Serve protected content with signed URLs
app.get('/premium/video/:id', authenticate, async (req, res) => {
  const videoId = req.params.id;
  const videoUrl = `https://d111111abcdef8.cloudfront.net/videos/${videoId}.mp4`;
  
  // Generate signed URL valid for 2 hours
  const signedUrl = cloudfront.generateSignedUrl(
    videoUrl,
    process.env.CLOUDFRONT_KEY_PAIR_ID,
    process.env.CLOUDFRONT_PRIVATE_KEY,
    120
  );
  
  res.json({
    url: signedUrl,
    expiresIn: 7200  // seconds
  });
});

// Serve files with signed cookies (for multiple files)
app.get('/premium/access', authenticate, async (req, res) => {
  const cookies = cloudfront.generateSignedCookie(
    'https://d111111abcdef8.cloudfront.net/premium/*',
    process.env.CLOUDFRONT_KEY_PAIR_ID,
    process.env.CLOUDFRONT_PRIVATE_KEY,
    480  // 8 hours
  );
  
  // Set cookies
  Object.entries(cookies).forEach(([name, value]) => {
    res.cookie(name, value, {
      domain: '.example.com',
      path: '/',
      secure: true,
      httpOnly: true,
      maxAge: 8 * 60 * 60 * 1000
    });
  });
  
  res.json({ message: 'Access granted' });
});

function authenticate(req, res, next) {
  // Authentication logic
  next();
}

app.listen(3000);
```

### Best Practices:

1. ✅ Use Origin Access Identity for S3
2. ✅ Enable compression for text files
3. ✅ Configure appropriate cache policies
4. ✅ Use signed URLs/cookies for protected content
5. ✅ Implement custom error pages
6. ✅ Use CloudFront Functions for simple rewrites
7. ✅ Use Lambda@Edge for complex logic
8. ✅ Enable logging to S3
9. ✅ Set up CloudWatch alarms
10. ✅ Use WAF for security

---

## Question 35: Explain Amazon Cognito for user authentication

### 📋 Answer

**Amazon Cognito** provides authentication, authorization, and user management for web and mobile apps.

### Cognito Architecture:

```
Amazon Cognito
├── User Pools
│   ├── User Directory
│   ├── Sign-up/Sign-in
│   ├── MFA
│   ├── Password policies
│   └── Social/Enterprise federation
├── Identity Pools
│   ├── AWS Credentials
│   ├── Federated identities
│   └── Guest access
└── Features
    ├── JWT tokens
    ├── Custom attributes
    ├── Lambda triggers
    └── Hosted UI
```

### Complete Cognito Implementation:

```javascript
// cognito-auth.js
import {
  CognitoIdentityProviderClient,
  SignUpCommand,
  ConfirmSignUpCommand,
  InitiateAuthCommand,
  RespondToAuthChallengeCommand,
  GetUserCommand,
  GlobalSignOutCommand,
  AdminCreateUserCommand,
  AdminSetUserPasswordCommand,
  AdminAddUserToGroupCommand
} from '@aws-sdk/client-cognito-identity-provider';
import crypto from 'crypto';

const cognitoClient = new CognitoIdentityProviderClient({ region: 'us-east-1' });

class CognitoAuthService {
  constructor(userPoolId, clientId, clientSecret) {
    this.userPoolId = userPoolId;
    this.clientId = clientId;
    this.clientSecret = clientSecret;
  }
  
  // Generate SECRET_HASH
  generateSecretHash(username) {
    return crypto
      .createHmac('SHA256', this.clientSecret)
      .update(username + this.clientId)
      .digest('base64');
  }
  
  // 1. Sign Up
  async signUp(username, password, email, attributes = {}) {
    const command = new SignUpCommand({
      ClientId: this.clientId,
      Username: username,
      Password: password,
      SecretHash: this.generateSecretHash(username),
      UserAttributes: [
        { Name: 'email', Value: email },
        ...Object.entries(attributes).map(([key, value]) => ({
          Name: key,
          Value: value
        }))
      ]
    });
    
    try {
      const response = await cognitoClient.send(command);
      console.log('User signed up:', username);
      return {
        userSub: response.UserSub,
        userConfirmed: response.UserConfirmed
      };
    } catch (error) {
      console.error('Sign up failed:', error);
      throw error;
    }
  }
  
  // 2. Confirm Sign Up
  async confirmSignUp(username, code) {
    const command = new ConfirmSignUpCommand({
      ClientId: this.clientId,
      Username: username,
      ConfirmationCode: code,
      SecretHash: this.generateSecretHash(username)
    });
    
    await cognitoClient.send(command);
    console.log('User confirmed:', username);
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
      
      // Handle MFA challenge
      if (response.ChallengeName === 'SMS_MFA' || response.ChallengeName === 'SOFTWARE_TOKEN_MFA') {
        return {
          challengeName: response.ChallengeName,
          session: response.Session
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
  
  // 4. Respond to MFA Challenge
  async respondToMFAChallenge(username, mfaCode, session) {
    const command = new RespondToAuthChallengeCommand({
      ClientId: this.clientId,
      ChallengeName: 'SMS_MFA',
      Session: session,
      ChallengeResponses: {
        USERNAME: username,
        SMS_MFA_CODE: mfaCode,
        SECRET_HASH: this.generateSecretHash(username)
      }
    });
    
    const response = await cognitoClient.send(command);
    
    return {
      accessToken: response.AuthenticationResult.AccessToken,
      idToken: response.AuthenticationResult.IdToken,
      refreshToken: response.AuthenticationResult.RefreshToken
    };
  }
  
  // 5. Refresh Token
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
  
  // 6. Get User Info
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
      }, {})
    };
  }
  
  // 7. Sign Out
  async signOut(accessToken) {
    const command = new GlobalSignOutCommand({
      AccessToken: accessToken
    });
    
    await cognitoClient.send(command);
    console.log('User signed out');
  }
  
  // 8. Admin Create User
  async adminCreateUser(username, email, temporaryPassword) {
    const command = new AdminCreateUserCommand({
      UserPoolId: this.userPoolId,
      Username: username,
      UserAttributes: [
        { Name: 'email', Value: email },
        { Name: 'email_verified', Value: 'true' }
      ],
      TemporaryPassword: temporaryPassword,
      MessageAction: 'SUPPRESS'  // Don't send welcome email
    });
    
    const response = await cognitoClient.send(command);
    console.log('Admin user created:', username);
    return response.User;
  }
  
  // 9. Admin Set Permanent Password
  async adminSetPassword(username, password) {
    const command = new AdminSetUserPasswordCommand({
      UserPoolId: this.userPoolId,
      Username: username,
      Password: password,
      Permanent: true
    });
    
    await cognitoClient.send(command);
    console.log('Password set for:', username);
  }
  
  // 10. Add User to Group
  async addUserToGroup(username, groupName) {
    const command = new AdminAddUserToGroupCommand({
      UserPoolId: this.userPoolId,
      Username: username,
      GroupName: groupName
    });
    
    await cognitoClient.send(command);
    console.log(`User ${username} added to group ${groupName}`);
  }
}

// Usage
const auth = new CognitoAuthService(
  'us-east-1_abc123',
  'client-id-here',
  'client-secret-here'
);

// Sign up new user
await auth.signUp('john.doe', 'SecurePass123!', 'john@example.com', {
  'name': 'John Doe',
  'phone_number': '+1234567890'
});

// Confirm sign up
await auth.confirmSignUp('john.doe', '123456');

// Sign in
const tokens = await auth.signIn('john.doe', 'SecurePass123!');
console.log('Access token:', tokens.accessToken);

// Get user info
const userInfo = await auth.getUserInfo(tokens.accessToken);
console.log('User info:', userInfo);
```

### JWT Token Verification Middleware:

```javascript
// jwt-middleware.js
import jwt from 'jsonwebtoken';
import jwksClient from 'jwks-rsa';

const region = 'us-east-1';
const userPoolId = 'us-east-1_abc123';
const issuer = `https://cognito-idp.${region}.amazonaws.com/${userPoolId}`;

// JWKS client
const client = jwksClient({
  jwksUri: `${issuer}/.well-known/jwks.json`,
  cache: true,
  cacheMaxAge: 3600000  // 1 hour
});

function getKey(header, callback) {
  client.getSigningKey(header.kid, (err, key) => {
    if (err) {
      callback(err);
    } else {
      callback(null, key.getPublicKey());
    }
  });
}

// Verify JWT middleware
export function verifyToken(req, res, next) {
  const authHeader = req.headers.authorization;
  const token = authHeader?.split(' ')[1];  // Bearer TOKEN
  
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }
  
  jwt.verify(token, getKey, {
    issuer: issuer,
    algorithms: ['RS256']
  }, (err, decoded) => {
    if (err) {
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
export function requireGroup(...groups) {
  return (req, res, next) => {
    const userGroups = req.user?.groups || [];
    const hasAccess = groups.some(g => userGroups.includes(g));
    
    if (!hasAccess) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    
    next();
  };
}

// Usage
import express from 'express';
const app = express();

app.get('/api/profile', verifyToken, (req, res) => {
  res.json({ user: req.user });
});

app.delete('/api/admin/users/:id', 
  verifyToken, 
  requireGroup('Admins'), 
  (req, res) => {
    // Only admins can access
    res.json({ message: 'User deleted' });
  }
);
```

### Best Practices:

1. ✅ Enable MFA for sensitive accounts
2. ✅ Use strong password policies
3. ✅ Implement account takeover protection
4. ✅ Use Lambda triggers for custom workflows
5. ✅ Enable advanced security features
6. ✅ Use groups for authorization
7. ✅ Implement token refresh logic
8. ✅ Use hosted UI for quick setup
9. ✅ Enable user pool analytics
10. ✅ Monitor with CloudWatch

---

## Question 36: Explain AWS WAF (Web Application Firewall)

### 📋 Answer

**AWS WAF** helps protect web applications from common web exploits and bots that could affect availability, compromise security, or consume excessive resources.

### WAF Components:

```
AWS WAF
├── Web ACLs
│   ├── Rules
│   ├── Rule Groups
│   └── Default Action
├── Rule Types
│   ├── Rate-based rules
│   ├── IP set rules
│   ├── Geo match rules
│   ├── String match rules
│   └── Managed rule groups
├── Protected Resources
│   ├── CloudFront
│   ├── ALB
│   ├── API Gateway
│   └── AppSync
└── Features
    ├── Bot Control
    ├── Account Takeover Prevention
    ├── Fraud Control
    └── CAPTCHA
```

### Complete WAF Implementation:

```javascript
// waf-service.js
import {
  WAFV2Client,
  CreateWebACLCommand,
  CreateIPSetCommand,
  CreateRuleGroupCommand,
  AssociateWebACLCommand,
  GetWebACLCommand,
  UpdateWebACLCommand
} from '@aws-sdk/client-wafv2';

const wafClient = new WAFV2Client({ region: 'us-east-1' });

class WAFService {
  // 1. Create IP Set
  async createIPSet(name, addresses, scope = 'REGIONAL') {
    const command = new CreateIPSetCommand({
      Name: name,
      Scope: scope,  // REGIONAL or CLOUDFRONT
      IPAddressVersion: 'IPV4',
      Addresses: addresses,  // ['192.0.2.0/24', '198.51.100.0/24']
      Description: 'Blocked IP addresses',
      Tags: [
        { Key: 'Environment', Value: 'Production' }
      ]
    });
    
    try {
      const response = await wafClient.send(command);
      console.log('IP Set created:', response.Summary.Id);
      return response.Summary;
    } catch (error) {
      console.error('Failed to create IP set:', error);
      throw error;
    }
  }
  
  // 2. Create Web ACL
  async createWebACL(name, rules, scope = 'REGIONAL') {
    const command = new CreateWebACLCommand({
      Name: name,
      Scope: scope,
      DefaultAction: {
        Allow: {}  // or Block: {}
      },
      Rules: rules,
      VisibilityConfig: {
        SampledRequestsEnabled: true,
        CloudWatchMetricsEnabled: true,
        MetricName: name
      },
      Description: 'Web ACL for application protection',
      Tags: [
        { Key: 'Application', Value: 'WebApp' }
      ]
    });
    
    try {
      const response = await wafClient.send(command);
      console.log('Web ACL created:', response.Summary.Id);
      return response.Summary;
    } catch (error) {
      console.error('Failed to create Web ACL:', error);
      throw error;
    }
  }
  
  // 3. Associate Web ACL with Resource
  async associateWebACL(webACLArn, resourceArn) {
    const command = new AssociateWebACLCommand({
      WebACLArn: webACLArn,
      ResourceArn: resourceArn  // ALB, API Gateway, etc.
    });
    
    await wafClient.send(command);
    console.log('Web ACL associated with resource');
  }
}

// WAF Rule Builder
class WAFRuleBuilder {
  constructor() {
    this.rules = [];
  }
  
  // Rate limiting rule
  addRateLimitRule(name, limit, priority) {
    this.rules.push({
      Name: name,
      Priority: priority,
      Statement: {
        RateBasedStatement: {
          Limit: limit,  // Requests per 5 minutes
          AggregateKeyType: 'IP'
        }
      },
      Action: {
        Block: {
          CustomResponse: {
            ResponseCode: 429,
            CustomResponseBodyKey: 'rate-limit-exceeded'
          }
        }
      },
      VisibilityConfig: {
        SampledRequestsEnabled: true,
        CloudWatchMetricsEnabled: true,
        MetricName: name
      }
    });
    return this;
  }
  
  // IP block rule
  addIPBlockRule(name, ipSetArn, priority) {
    this.rules.push({
      Name: name,
      Priority: priority,
      Statement: {
        IPSetReferenceStatement: {
          Arn: ipSetArn
        }
      },
      Action: {
        Block: {}
      },
      VisibilityConfig: {
        SampledRequestsEnabled: true,
        CloudWatchMetricsEnabled: true,
        MetricName: name
      }
    });
    return this;
  }
  
  // Geo blocking rule
  addGeoBlockRule(name, countryCodes, priority) {
    this.rules.push({
      Name: name,
      Priority: priority,
      Statement: {
        GeoMatchStatement: {
          CountryCodes: countryCodes  // ['CN', 'RU', 'KP']
        }
      },
      Action: {
        Block: {}
      },
      VisibilityConfig: {
        SampledRequestsEnabled: true,
        CloudWatchMetricsEnabled: true,
        MetricName: name
      }
    });
    return this;
  }
  
  // SQL injection protection
  addSQLiProtectionRule(name, priority) {
    this.rules.push({
      Name: name,
      Priority: priority,
      Statement: {
        SqliMatchStatement: {
          FieldToMatch: {
            Body: {
              OversizeHandling: 'CONTINUE'
            }
          },
          TextTransformations: [
            {
              Priority: 0,
              Type: 'URL_DECODE'
            },
            {
              Priority: 1,
              Type: 'HTML_ENTITY_DECODE'
            }
          ]
        }
      },
      Action: {
        Block: {}
      },
      VisibilityConfig: {
        SampledRequestsEnabled: true,
        CloudWatchMetricsEnabled: true,
        MetricName: name
      }
    });
    return this;
  }
  
  // XSS protection
  addXSSProtectionRule(name, priority) {
    this.rules.push({
      Name: name,
      Priority: priority,
      Statement: {
        XssMatchStatement: {
          FieldToMatch: {
            Body: {
              OversizeHandling: 'CONTINUE'
            }
          },
          TextTransformations: [
            {
              Priority: 0,
              Type: 'URL_DECODE'
            },
            {
              Priority: 1,
              Type: 'HTML_ENTITY_DECODE'
            }
          ]
        }
      },
      Action: {
        Block: {}
      },
      VisibilityConfig: {
        SampledRequestsEnabled: true,
        CloudWatchMetricsEnabled: true,
        MetricName: name
      }
    });
    return this;
  }
  
  // String match rule (e.g., block user agents)
  addStringMatchRule(name, field, matchStrings, priority) {
    this.rules.push({
      Name: name,
      Priority: priority,
      Statement: {
        ByteMatchStatement: {
          FieldToMatch: field,  // e.g., { SingleHeader: { Name: 'user-agent' } }
          PositionalConstraint: 'CONTAINS',
          SearchString: matchStrings[0],
          TextTransformations: [
            {
              Priority: 0,
              Type: 'LOWERCASE'
            }
          ]
        }
      },
      Action: {
        Block: {}
      },
      VisibilityConfig: {
        SampledRequestsEnabled: true,
        CloudWatchMetricsEnabled: true,
        MetricName: name
      }
    });
    return this;
  }
  
  // AWS Managed Rules
  addManagedRuleGroup(name, vendorName, ruleName, priority) {
    this.rules.push({
      Name: name,
      Priority: priority,
      Statement: {
        ManagedRuleGroupStatement: {
          VendorName: vendorName,  // 'AWS'
          Name: ruleName  // e.g., 'AWSManagedRulesCommonRuleSet'
        }
      },
      OverrideAction: {
        None: {}
      },
      VisibilityConfig: {
        SampledRequestsEnabled: true,
        CloudWatchMetricsEnabled: true,
        MetricName: name
      }
    });
    return this;
  }
  
  build() {
    return this.rules;
  }
}

// Usage - Comprehensive Web ACL
const waf = new WAFService();

// Create IP set for blocked IPs
const ipSet = await waf.createIPSet(
  'blocked-ips',
  [
    '192.0.2.0/24',
    '198.51.100.42/32'
  ],
  'REGIONAL'
);

// Build rules
const rules = new WAFRuleBuilder()
  .addRateLimitRule('rate-limit-rule', 2000, 1)  // 2000 req per 5 min
  .addIPBlockRule('ip-block-rule', ipSet.ARN, 2)
  .addGeoBlockRule('geo-block-rule', ['CN', 'RU'], 3)
  .addSQLiProtectionRule('sqli-protection', 4)
  .addXSSProtectionRule('xss-protection', 5)
  .addStringMatchRule(
    'block-bad-bots',
    { SingleHeader: { Name: 'user-agent' } },
    ['badbot', 'scraper'],
    6
  )
  .addManagedRuleGroup('aws-core-rules', 'AWS', 'AWSManagedRulesCommonRuleSet', 7)
  .addManagedRuleGroup('aws-known-bad-inputs', 'AWS', 'AWSManagedRulesKnownBadInputsRuleSet', 8)
  .build();

// Create Web ACL
const webACL = await waf.createWebACL('my-web-acl', rules, 'REGIONAL');

// Associate with ALB
await waf.associateWebACL(
  webACL.ARN,
  'arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-alb/abc123'
);
```

### Best Practices:

1. ✅ Use AWS Managed Rules as baseline
2. ✅ Implement rate limiting
3. ✅ Enable logging to S3 or CloudWatch
4. ✅ Use CAPTCHA for suspicious requests
5. ✅ Block known malicious IPs
6. ✅ Implement geo-blocking if needed
7. ✅ Monitor WAF metrics in CloudWatch
8. ✅ Use count mode before blocking
9. ✅ Regularly review and update rules
10. ✅ Integrate with AWS Shield for DDoS protection

---

## Key Takeaways

### Question 34 (CloudFront):
- Global CDN with 400+ edge locations
- Lambda@Edge for request/response manipulation
- CloudFront Functions for lightweight rewrites
- Signed URLs/cookies for protected content
- Integrates with S3, EC2, ALB
- Cache behaviors for different content types

### Question 35 (Cognito):
- User Pools for authentication
- Identity Pools for AWS credentials
- JWT tokens for API authorization
- Built-in MFA and password policies
- Social and enterprise identity federation
- Lambda triggers for custom workflows

### Question 36 (WAF):
- Web application firewall
- Rate limiting to prevent abuse
- SQL injection and XSS protection
- Geo-blocking and IP filtering
- AWS Managed Rules for common threats
- Integrates with CloudFront, ALB, API Gateway
