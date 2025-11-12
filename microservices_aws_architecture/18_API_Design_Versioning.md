# API Design & Versioning

## Question 35: RESTful API Design & Versioning

### 📋 Question Statement

Implement comprehensive API design for Emirates NBD including RESTful principles, API versioning strategies, OpenAPI/Swagger documentation, rate limiting, and API gateway patterns.

---

### 🎯 RESTful API Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│              EMIRATES NBD RESTful API DESIGN                                │
└────────────────────────────────────────────────────────────────────────────┘

    CLIENT                    API GATEWAY                  SERVICES
    ──────                    ───────────                  ────────
    
    ┌──────────┐             ┌─────────────┐             ┌──────────┐
    │  Mobile  │             │   Routing   │             │ Account  │
    │   App    │────────────►│   /v1/      │────────────►│ Service  │
    └──────────┘             │   /v2/      │             └──────────┘
                             │             │             
    ┌──────────┐             │  Rate       │             ┌──────────┐
    │   Web    │────────────►│  Limiting   │────────────►│Transaction│
    │   App    │             │             │             │ Service  │
    └──────────┘             │  Auth       │             └──────────┘
                             │  (JWT)      │
    ┌──────────┐             │             │             ┌──────────┐
    │ Partner  │────────────►│  Caching    │────────────►│ Payment  │
    │   API    │             │             │             │ Service  │
    └──────────┘             │  Transform  │             └──────────┘
                             └─────────────┘
                             
    Versioning Strategies:
    • URI: /api/v1/accounts
    • Header: Accept: application/vnd.emiratesnbd.v1+json
    • Query: /api/accounts?version=1
```

### 📝 OpenAPI/Swagger Documentation

```yaml
# openapi/banking-api.yaml
openapi: 3.0.0
info:
  title: Emirates NBD Banking API
  version: 2.0.0
  description: Comprehensive banking services API for Emirates NBD
  contact:
    name: API Support
    email: api-support@emiratesnbd.com
  license:
    name: Proprietary

servers:
  - url: https://api.emiratesnbd.com/v2
    description: Production
  - url: https://staging-api.emiratesnbd.com/v2
    description: Staging
  - url: https://sandbox-api.emiratesnbd.com/v2
    description: Sandbox

security:
  - BearerAuth: []
  - ApiKeyAuth: []

paths:
  /accounts:
    get:
      summary: List customer accounts
      tags: [Accounts]
      parameters:
        - name: customerId
          in: query
          required: true
          schema:
            type: string
        - name: type
          in: query
          schema:
            type: string
            enum: [SAVINGS, CHECKING, INVESTMENT]
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
            maximum: 100
        - name: offset
          in: query
          schema:
            type: integer
            default: 0
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AccountList'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '429':
          $ref: '#/components/responses/TooManyRequests'
    
    post:
      summary: Create new account
      tags: [Accounts]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateAccountRequest'
      responses:
        '201':
          description: Account created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Account'
        '400':
          $ref: '#/components/responses/BadRequest'

  /accounts/{accountId}:
    get:
      summary: Get account details
      tags: [Accounts]
      parameters:
        - name: accountId
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: Account details
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Account'
        '404':
          $ref: '#/components/responses/NotFound'
    
    patch:
      summary: Update account
      tags: [Accounts]
      parameters:
        - name: accountId
          in: path
          required: true
          schema:
            type: string
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UpdateAccountRequest'
      responses:
        '200':
          description: Account updated
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Account'
    
    delete:
      summary: Close account
      tags: [Accounts]
      parameters:
        - name: accountId
          in: path
          required: true
          schema:
            type: string
      responses:
        '204':
          description: Account closed

  /accounts/{accountId}/transactions:
    get:
      summary: Get account transactions
      tags: [Transactions]
      parameters:
        - name: accountId
          in: path
          required: true
          schema:
            type: string
        - name: startDate
          in: query
          schema:
            type: string
            format: date
        - name: endDate
          in: query
          schema:
            type: string
            format: date
        - name: type
          in: query
          schema:
            type: string
            enum: [DEBIT, CREDIT]
      responses:
        '200':
          description: Transaction list
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/TransactionList'

  /transfers:
    post:
      summary: Create transfer
      tags: [Transfers]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateTransferRequest'
      responses:
        '201':
          description: Transfer created
          headers:
            Location:
              schema:
                type: string
              description: URI of created transfer
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Transfer'
        '400':
          $ref: '#/components/responses/BadRequest'
        '422':
          $ref: '#/components/responses/UnprocessableEntity'

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
    ApiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key

  schemas:
    Account:
      type: object
      properties:
        accountId:
          type: string
          example: ACC-123456
        customerId:
          type: string
        accountNumber:
          type: string
          example: "1234567890"
        type:
          type: string
          enum: [SAVINGS, CHECKING, INVESTMENT]
        status:
          type: string
          enum: [ACTIVE, INACTIVE, CLOSED]
        balance:
          type: number
          format: double
          example: 5000.00
        currency:
          type: string
          example: AED
        createdAt:
          type: string
          format: date-time
        updatedAt:
          type: string
          format: date-time

    AccountList:
      type: object
      properties:
        accounts:
          type: array
          items:
            $ref: '#/components/schemas/Account'
        pagination:
          $ref: '#/components/schemas/Pagination'

    CreateAccountRequest:
      type: object
      required:
        - customerId
        - type
        - initialDeposit
      properties:
        customerId:
          type: string
        type:
          type: string
          enum: [SAVINGS, CHECKING, INVESTMENT]
        initialDeposit:
          type: number
          minimum: 0

    UpdateAccountRequest:
      type: object
      properties:
        status:
          type: string
          enum: [ACTIVE, INACTIVE]

    Transaction:
      type: object
      properties:
        transactionId:
          type: string
        accountId:
          type: string
        type:
          type: string
          enum: [DEBIT, CREDIT]
        amount:
          type: number
        description:
          type: string
        timestamp:
          type: string
          format: date-time

    TransactionList:
      type: object
      properties:
        transactions:
          type: array
          items:
            $ref: '#/components/schemas/Transaction'
        pagination:
          $ref: '#/components/schemas/Pagination'

    CreateTransferRequest:
      type: object
      required:
        - fromAccountId
        - toAccountId
        - amount
      properties:
        fromAccountId:
          type: string
        toAccountId:
          type: string
        amount:
          type: number
          minimum: 0.01
        description:
          type: string

    Transfer:
      type: object
      properties:
        transferId:
          type: string
        fromAccountId:
          type: string
        toAccountId:
          type: string
        amount:
          type: number
        status:
          type: string
          enum: [PENDING, COMPLETED, FAILED]
        createdAt:
          type: string
          format: date-time

    Pagination:
      type: object
      properties:
        limit:
          type: integer
        offset:
          type: integer
        total:
          type: integer

    Error:
      type: object
      properties:
        code:
          type: string
        message:
          type: string
        details:
          type: object

  responses:
    BadRequest:
      description: Bad request
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    Unauthorized:
      description: Unauthorized
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    NotFound:
      description: Resource not found
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    TooManyRequests:
      description: Rate limit exceeded
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    UnprocessableEntity:
      description: Validation error
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
```

### 🚀 Express.js REST API Implementation

```typescript
// services/account-api/src/app.ts
import express, { Request, Response, NextFunction } from 'express';
import swaggerUi from 'swagger-ui-express';
import YAML from 'yamljs';
import rateLimit from 'express-rate-limit';
import helmet from 'helmet';
import cors from 'cors';

const app = express();
const swaggerDocument = YAML.load('./openapi/banking-api.yaml');

// Middleware
app.use(helmet()); // Security headers
app.use(cors()); // CORS
app.use(express.json());

// Rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // 100 requests per window
  message: 'Too many requests, please try again later'
});

app.use('/api/', limiter);

// Swagger documentation
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerDocument));

// API Versioning Middleware
app.use((req: Request, res: Response, next: NextFunction) => {
  // Extract version from URI (/v1/accounts)
  const uriVersion = req.path.match(/^\/v(\d+)/)?.[1];
  
  // Extract version from header (Accept: application/vnd.emiratesnbd.v2+json)
  const headerVersion = req.headers.accept?.match(/vnd\.emiratesnbd\.v(\d+)/)?.[1];
  
  // Extract version from query (?version=1)
  const queryVersion = req.query.version as string;
  
  req.apiVersion = uriVersion || headerVersion || queryVersion || '2'; // Default v2
  next();
});

// ============================================
// ROUTES - Version 2 (Current)
// ============================================

// GET /v2/accounts
app.get('/v2/accounts', async (req: Request, res: Response) => {
  const { customerId, type, limit = 20, offset = 0 } = req.query;

  if (!customerId) {
    return res.status(400).json({
      code: 'MISSING_CUSTOMER_ID',
      message: 'customerId is required'
    });
  }

  // Fetch accounts from database
  const accounts = await getAccounts({
    customerId: customerId as string,
    type: type as string,
    limit: parseInt(limit as string),
    offset: parseInt(offset as string)
  });

  res.json({
    accounts,
    pagination: {
      limit: parseInt(limit as string),
      offset: parseInt(offset as string),
      total: accounts.length
    }
  });
});

// POST /v2/accounts
app.post('/v2/accounts', async (req: Request, res: Response) => {
  const { customerId, type, initialDeposit } = req.body;

  // Validation
  if (!customerId || !type || initialDeposit === undefined) {
    return res.status(400).json({
      code: 'VALIDATION_ERROR',
      message: 'Missing required fields'
    });
  }

  // Create account
  const account = await createAccount({
    customerId,
    type,
    initialDeposit
  });

  res.status(201)
    .location(`/v2/accounts/${account.accountId}`)
    .json(account);
});

// GET /v2/accounts/:accountId
app.get('/v2/accounts/:accountId', async (req: Request, res: Response) => {
  const { accountId } = req.params;

  const account = await getAccountById(accountId);

  if (!account) {
    return res.status(404).json({
      code: 'ACCOUNT_NOT_FOUND',
      message: `Account ${accountId} not found`
    });
  }

  res.json(account);
});

// PATCH /v2/accounts/:accountId
app.patch('/v2/accounts/:accountId', async (req: Request, res: Response) => {
  const { accountId } = req.params;
  const updates = req.body;

  const account = await updateAccount(accountId, updates);

  res.json(account);
});

// DELETE /v2/accounts/:accountId
app.delete('/v2/accounts/:accountId', async (req: Request, res: Response) => {
  const { accountId } = req.params;

  await closeAccount(accountId);

  res.status(204).send();
});

// GET /v2/accounts/:accountId/transactions
app.get('/v2/accounts/:accountId/transactions', async (req: Request, res: Response) => {
  const { accountId } = req.params;
  const { startDate, endDate, type } = req.query;

  const transactions = await getTransactions({
    accountId,
    startDate: startDate as string,
    endDate: endDate as string,
    type: type as string
  });

  res.json({
    transactions,
    pagination: {
      total: transactions.length
    }
  });
});

// POST /v2/transfers
app.post('/v2/transfers', async (req: Request, res: Response) => {
  const { fromAccountId, toAccountId, amount, description } = req.body;

  // Validation
  if (!fromAccountId || !toAccountId || !amount) {
    return res.status(400).json({
      code: 'VALIDATION_ERROR',
      message: 'Missing required fields'
    });
  }

  if (amount <= 0) {
    return res.status(422).json({
      code: 'INVALID_AMOUNT',
      message: 'Amount must be greater than 0'
    });
  }

  // Create transfer
  const transfer = await createTransfer({
    fromAccountId,
    toAccountId,
    amount,
    description
  });

  res.status(201)
    .location(`/v2/transfers/${transfer.transferId}`)
    .json(transfer);
});

// ============================================
// ROUTES - Version 1 (Legacy, deprecated)
// ============================================

app.get('/v1/accounts', async (req: Request, res: Response) => {
  // V1 response format (different structure)
  const accounts = await getAccounts({ customerId: req.query.customerId as string });
  
  res.json({
    data: accounts, // V1 wrapped in 'data'
    count: accounts.length
  });
});

// ============================================
// ERROR HANDLING
// ============================================

app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  console.error(err);
  
  res.status(500).json({
    code: 'INTERNAL_ERROR',
    message: 'An unexpected error occurred'
  });
});

// Placeholder functions (implement with actual DB logic)
async function getAccounts(params: any) { return []; }
async function createAccount(params: any) { return {}; }
async function getAccountById(id: string) { return {}; }
async function updateAccount(id: string, updates: any) { return {}; }
async function closeAccount(id: string) { }
async function getTransactions(params: any) { return []; }
async function createTransfer(params: any) { return {}; }

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`API server running on port ${PORT}`);
  console.log(`API documentation: http://localhost:${PORT}/api-docs`);
});
```

### 🎓 Interview Discussion Points - Q35

**Q1: What are REST API best practices?**

**A**:
- **Use nouns, not verbs**: `/accounts` not `/getAccounts`
- **HTTP methods correctly**: GET (read), POST (create), PUT/PATCH (update), DELETE (delete)
- **Status codes**: 200 OK, 201 Created, 400 Bad Request, 404 Not Found, 500 Server Error
- **Pagination**: Limit/offset or cursor-based
- **Versioning**: URI, header, or query parameter
- **HATEOAS**: Include links to related resources
- **Idempotency**: PUT/DELETE should be idempotent

**Q2: What versioning strategy is best?**

**A**:
- **URI versioning** (`/v1/accounts`): ✅ Simple, visible, cacheable
- **Header versioning** (`Accept: application/vnd.api.v1+json`): ✅ Clean URLs, flexible
- **Query parameter** (`/accounts?version=1`): ⚠️ Less clean, but simple

**Recommendation**: URI versioning for public APIs (simplicity), header for internal APIs (flexibility)

**Q3: How to handle API deprecation?**

**A**:
1. **Announce** deprecation 6-12 months ahead
2. **Add headers**: `Sunset: Sat, 31 Dec 2024 23:59:59 GMT`
3. **Return warnings**: `Warning: 299 - "API v1 will be deprecated on 2024-12-31"`
4. **Maintain old version** during transition period
5. **Monitor usage**: Track v1 API calls
6. **Provide migration guide**: Document breaking changes

**Q4: What is rate limiting and how to implement it?**

**A**:
- **Purpose**: Prevent abuse, ensure fair usage, protect backend
- **Strategies**:
  - Fixed window: 100 requests per 15 minutes
  - Sliding window: More accurate
  - Token bucket: Burst traffic support
- **Headers**: `X-RateLimit-Limit: 100`, `X-RateLimit-Remaining: 45`, `X-RateLimit-Reset: 1234567890`
- **Response**: 429 Too Many Requests with `Retry-After` header

**Q5: How to design pagination for large datasets?**

**A**:
- **Offset/Limit**: `/accounts?limit=20&offset=40`
  - Simple but slow for large offsets
  - Use for small datasets
- **Cursor-based**: `/accounts?limit=20&after=cursor123`
  - Faster for large datasets
  - Handles inserts/deletes better
  - Use for infinite scroll

---

## Question 36: API Gateway Advanced Features

### 📋 Question Statement

Implement advanced API Gateway features for Emirates NBD including request/response transformation, custom authorizers, usage plans with API keys, caching, throttling, and WAF integration.

---

### 🔐 Custom Authorizer (Lambda Authorizer)

```typescript
// lambda/authorizers/jwt-authorizer.ts
import { APIGatewayAuthorizerResult, APIGatewayTokenAuthorizerEvent } from 'aws-lambda';
import * as jwt from 'jsonwebtoken';

interface JWTPayload {
  sub: string;
  email: string;
  role: string;
  customerId: string;
}

export const handler = async (
  event: APIGatewayTokenAuthorizerEvent
): Promise<APIGatewayAuthorizerResult> => {
  try {
    const token = event.authorizationToken.replace('Bearer ', '');
    
    // Verify JWT token
    const decoded = jwt.verify(token, process.env.JWT_SECRET!) as JWTPayload;

    // Generate IAM policy
    const policy = generatePolicy(decoded.sub, 'Allow', event.methodArn, {
      customerId: decoded.customerId,
      role: decoded.role,
      email: decoded.email
    });

    console.log('Authorization successful for user:', decoded.sub);
    return policy;
  } catch (error) {
    console.error('Authorization failed:', error);
    throw new Error('Unauthorized');
  }
};

function generatePolicy(
  principalId: string,
  effect: 'Allow' | 'Deny',
  resource: string,
  context?: Record<string, string>
): APIGatewayAuthorizerResult {
  return {
    principalId,
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
    context: context || {}
  };
}
```

### 🔧 API Gateway CDK Stack with Advanced Features

```typescript
// infrastructure/cdk/api-gateway-advanced-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as wafv2 from 'aws-cdk-lib/aws-wafv2';
import * as logs from 'aws-cdk-lib/aws-logs';

export class AdvancedAPIGatewayStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Backend Lambda functions
    const accountFunction = new lambda.Function(this, 'AccountFunction', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/account-service')
    });

    const transactionFunction = new lambda.Function(this, 'TransactionFunction', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/transaction-service')
    });

    // Custom JWT Authorizer
    const authorizer = new lambda.Function(this, 'JWTAuthorizer', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'jwt-authorizer.handler',
      code: lambda.Code.fromAsset('lambda/authorizers'),
      environment: {
        JWT_SECRET: process.env.JWT_SECRET || 'change-me-in-production'
      }
    });

    // API Gateway with request validation
    const api = new apigateway.RestApi(this, 'EmiratesNBDAPI', {
      restApiName: 'Emirates NBD Banking API',
      description: 'Advanced API with custom authorizers and throttling',
      deployOptions: {
        stageName: 'prod',
        tracingEnabled: true,
        dataTraceEnabled: true,
        loggingLevel: apigateway.MethodLoggingLevel.INFO,
        accessLogDestination: new apigateway.LogGroupLogDestination(
          new logs.LogGroup(this, 'APIGatewayLogs')
        ),
        accessLogFormat: apigateway.AccessLogFormat.jsonWithStandardFields()
      },
      defaultCorsPreflightOptions: {
        allowOrigins: ['https://www.emiratesnbd.com'],
        allowMethods: ['GET', 'POST', 'PUT', 'DELETE'],
        allowHeaders: ['Content-Type', 'Authorization']
      }
    });

    // Lambda Authorizer
    const lambdaAuthorizer = new apigateway.TokenAuthorizer(this, 'TokenAuthorizer', {
      handler: authorizer,
      resultsCacheTtl: cdk.Duration.minutes(5)
    });

    // Request/Response Models
    const accountModel = api.addModel('AccountModel', {
      contentType: 'application/json',
      modelName: 'Account',
      schema: {
        type: apigateway.JsonSchemaType.OBJECT,
        required: ['customerId', 'accountType', 'initialBalance'],
        properties: {
          customerId: { type: apigateway.JsonSchemaType.STRING },
          accountType: { 
            type: apigateway.JsonSchemaType.STRING,
            enum: ['SAVINGS', 'CURRENT', 'FIXED_DEPOSIT']
          },
          initialBalance: { 
            type: apigateway.JsonSchemaType.NUMBER,
            minimum: 0
          }
        }
      }
    });

    // Request Validator
    const requestValidator = api.addRequestValidator('BodyValidator', {
      validateRequestBody: true,
      validateRequestParameters: true
    });

    // Accounts Resource
    const accounts = api.root.addResource('accounts');
    
    accounts.addMethod('POST', new apigateway.LambdaIntegration(accountFunction, {
      requestTemplates: {
        'application/json': JSON.stringify({
          body: '$input.json("$")',
          customerId: '$context.authorizer.customerId',
          requestId: '$context.requestId'
        })
      },
      integrationResponses: [
        {
          statusCode: '201',
          responseTemplates: {
            'application/json': JSON.stringify({
              accountId: '$input.path("$.accountId")',
              status: 'created',
              timestamp: '$context.requestTime'
            })
          }
        }
      ]
    }), {
      authorizer: lambdaAuthorizer,
      requestValidator,
      requestModels: {
        'application/json': accountModel
      },
      methodResponses: [
        { statusCode: '201' },
        { statusCode: '400' },
        { statusCode: '401' }
      ]
    });

    // GET /accounts/{accountId} with caching
    const accountById = accounts.addResource('{accountId}');
    accountById.addMethod('GET', new apigateway.LambdaIntegration(accountFunction), {
      authorizer: lambdaAuthorizer,
      requestParameters: {
        'method.request.path.accountId': true
      },
      methodResponses: [
        {
          statusCode: '200',
          responseParameters: {
            'method.response.header.Cache-Control': true
          }
        }
      ]
    });

    // Usage Plan with API Key
    const usagePlan = api.addUsagePlan('StandardUsagePlan', {
      name: 'Standard Plan',
      throttle: {
        rateLimit: 1000,   // requests per second
        burstLimit: 2000    // concurrent requests
      },
      quota: {
        limit: 1000000,     // 1M requests per month
        period: apigateway.Period.MONTH
      }
    });

    const apiKey = api.addApiKey('PartnerAPIKey', {
      apiKeyName: 'partner-integration-key'
    });

    usagePlan.addApiKey(apiKey);
    usagePlan.addApiStage({
      stage: api.deploymentStage
    });

    // Enable caching
    const deployment = new apigateway.Deployment(this, 'Deployment', {
      api
    });

    const stage = new apigateway.Stage(this, 'ProdStage', {
      deployment,
      stageName: 'prod',
      cacheClusterEnabled: true,
      cacheClusterSize: '0.5',  // 0.5 GB cache
      cacheTtl: cdk.Duration.minutes(5),
      cacheDataEncrypted: true
    });

    // Method-level cache settings
    stage.addMethodCachingConfig({
      path: '/accounts/{accountId}',
      method: 'GET',
      cachingEnabled: true,
      cacheTtl: cdk.Duration.minutes(5),
      cacheDataEncrypted: true
    });

    // WAF Web ACL
    const webAcl = new wafv2.CfnWebACL(this, 'APIGatewayWAF', {
      scope: 'REGIONAL',
      defaultAction: { allow: {} },
      rules: [
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
            sampledRequestsEnabled: true,
            cloudWatchMetricsEnabled: true,
            metricName: 'RateLimitRule'
          }
        },
        {
          name: 'AWSManagedRulesCommonRuleSet',
          priority: 2,
          statement: {
            managedRuleGroupStatement: {
              vendorName: 'AWS',
              name: 'AWSManagedRulesCommonRuleSet'
            }
          },
          overrideAction: { none: {} },
          visibilityConfig: {
            sampledRequestsEnabled: true,
            cloudWatchMetricsEnabled: true,
            metricName: 'CommonRuleSet'
          }
        }
      ],
      visibilityConfig: {
        sampledRequestsEnabled: true,
        cloudWatchMetricsEnabled: true,
        metricName: 'APIGatewayWAF'
      }
    });

    // Associate WAF with API Gateway
    new wafv2.CfnWebACLAssociation(this, 'WebACLAssociation', {
      resourceArn: api.arnForExecuteApi(),
      webAclArn: webAcl.attrArn
    });

    // Output
    new cdk.CfnOutput(this, 'APIURL', {
      value: api.url,
      description: 'API Gateway URL'
    });

    new cdk.CfnOutput(this, 'APIKey', {
      value: apiKey.keyId,
      description: 'API Key ID'
    });
  }
}
```

### 🎯 Response Transformation with VTL

```velocity
## VTL template for response transformation
#set($inputRoot = $input.path('$'))
{
  "statusCode": 200,
  "body": {
    "account": {
      "id": "$inputRoot.accountId",
      "type": "$inputRoot.accountType",
      "balance": $inputRoot.balance,
      "currency": "AED",
      "status": "$inputRoot.status"
    },
    "metadata": {
      "requestId": "$context.requestId",
      "timestamp": "$context.requestTime",
      "region": "me-south-1"
    }
  },
  "headers": {
    "X-Request-ID": "$context.requestId",
    "Cache-Control": "max-age=300"
  }
}
```

### 📊 Advanced Throttling Configuration

```typescript
// infrastructure/cdk/throttling-config.ts
import * as apigateway from 'aws-cdk-lib/aws-apigateway';

export class ThrottlingConfiguration {
  static getUsagePlans() {
    return {
      free: {
        name: 'Free Tier',
        throttle: {
          rateLimit: 10,      // 10 req/sec
          burstLimit: 20       // 20 concurrent
        },
        quota: {
          limit: 10000,        // 10k req/month
          period: apigateway.Period.MONTH
        }
      },
      standard: {
        name: 'Standard Plan',
        throttle: {
          rateLimit: 100,
          burstLimit: 200
        },
        quota: {
          limit: 1000000,
          period: apigateway.Period.MONTH
        }
      },
      premium: {
        name: 'Premium Plan',
        throttle: {
          rateLimit: 1000,
          burstLimit: 2000
        },
        quota: {
          limit: 10000000,
          period: apigateway.Period.MONTH
        }
      },
      internal: {
        name: 'Internal Services',
        throttle: {
          rateLimit: 10000,
          burstLimit: 20000
        },
        quota: {
          limit: 100000000,
          period: apigateway.Period.MONTH
        }
      }
    };
  }

  static getMethodThrottling() {
    return {
      'GET /accounts/{accountId}': {
        rateLimit: 1000,
        burstLimit: 2000
      },
      'POST /transactions': {
        rateLimit: 500,
        burstLimit: 1000
      },
      'GET /transactions/history': {
        rateLimit: 100,
        burstLimit: 200
      }
    };
  }
}
```

### 🛡️ API Key Management Service

```typescript
// services/api-key-management/api-key-service.ts
import { APIGateway, DynamoDB } from 'aws-sdk';

export class APIKeyManagementService {
  private apiGateway: APIGateway;
  private dynamoDB: DynamoDB.DocumentClient;

  constructor() {
    this.apiGateway = new APIGateway();
    this.dynamoDB = new DynamoDB.DocumentClient();
  }

  async createAPIKey(customerId: string, plan: string): Promise<APIKeyInfo> {
    // Create API key
    const apiKey = await this.apiGateway.createApiKey({
      name: `${customerId}-${plan}`,
      description: `API key for customer ${customerId}`,
      enabled: true
    }).promise();

    // Associate with usage plan
    const usagePlanId = await this.getUsagePlanId(plan);
    await this.apiGateway.createUsagePlanKey({
      usagePlanId,
      keyId: apiKey.id!,
      keyType: 'API_KEY'
    }).promise();

    // Store in DynamoDB
    await this.dynamoDB.put({
      TableName: 'APIKeys',
      Item: {
        apiKeyId: apiKey.id,
        customerId,
        plan,
        createdAt: new Date().toISOString(),
        enabled: true,
        value: apiKey.value
      }
    }).promise();

    return {
      apiKeyId: apiKey.id!,
      apiKeyValue: apiKey.value!,
      plan,
      createdAt: new Date().toISOString()
    };
  }

  async revokeAPIKey(apiKeyId: string): Promise<void> {
    await this.apiGateway.updateApiKey({
      apiKeyId,
      patchOperations: [
        {
          op: 'replace',
          path: '/enabled',
          value: 'false'
        }
      ]
    }).promise();

    await this.dynamoDB.update({
      TableName: 'APIKeys',
      Key: { apiKeyId },
      UpdateExpression: 'SET enabled = :enabled, revokedAt = :revokedAt',
      ExpressionAttributeValues: {
        ':enabled': false,
        ':revokedAt': new Date().toISOString()
      }
    }).promise();
  }

  async getUsageStatistics(apiKeyId: string, startDate: string, endDate: string) {
    const usagePlanId = await this.getUsagePlanIdForKey(apiKeyId);

    const usage = await this.apiGateway.getUsage({
      usagePlanId,
      keyId: apiKeyId,
      startDate,
      endDate
    }).promise();

    return {
      totalRequests: Object.values(usage.items || {}).reduce(
        (sum, day) => sum + (day[0][0] || 0),
        0
      ),
      dailyUsage: usage.items
    };
  }

  private async getUsagePlanId(plan: string): Promise<string> {
    const usagePlans = await this.apiGateway.getUsagePlans().promise();
    const usagePlan = usagePlans.items?.find(p => p.name === plan);
    
    if (!usagePlan) {
      throw new Error(`Usage plan ${plan} not found`);
    }

    return usagePlan.id!;
  }

  private async getUsagePlanIdForKey(apiKeyId: string): Promise<string> {
    const key = await this.dynamoDB.get({
      TableName: 'APIKeys',
      Key: { apiKeyId }
    }).promise();

    return await this.getUsagePlanId(key.Item!.plan);
  }
}

interface APIKeyInfo {
  apiKeyId: string;
  apiKeyValue: string;
  plan: string;
  createdAt: string;
}
```

### 🎓 Interview Discussion Points - Q36

**Q1: What is the difference between Lambda Authorizer and Cognito Authorizer?**

**A**:
- **Lambda Authorizer**: Custom logic, JWT validation, database lookups
- **Cognito Authorizer**: Managed by AWS, OAuth 2.0, OIDC
- **Use Lambda**: Complex authorization rules, custom identity providers
- **Use Cognito**: Standard OAuth flows, user pools

**Q2: How does API Gateway caching work?**

**A**:
- **Cache key**: Includes path, query params, headers (configurable)
- **TTL**: 0-3600 seconds
- **Per-method**: Can enable/disable per endpoint
- **Cache invalidation**: Manual or automatic (TTL expiry)
- **Cost**: Charged per cache size (0.5GB, 1.6GB, 6GB, etc.)

**Q3: What happens when rate limit is exceeded?**

**A**:
- Returns **429 Too Many Requests**
- Includes `Retry-After` header
- Client should implement exponential backoff
- Burst allows temporary spikes above rate limit

**Q4: How to implement API versioning?**

**A**:
```
URI versioning: /v1/accounts, /v2/accounts
Header versioning: Accept: application/vnd.api+json; version=2
Query param: /accounts?version=2
```
**Best practice**: URI versioning (explicit, cacheable)

**Q5: What is request/response transformation used for?**

**A**:
- **Adapt legacy systems**: Transform request format
- **Enrich requests**: Add context (requestId, userId)
- **Simplify responses**: Remove unnecessary fields
- **Protocol translation**: REST → SOAP
- **VTL (Velocity Template Language)**: Used for transformations

---

