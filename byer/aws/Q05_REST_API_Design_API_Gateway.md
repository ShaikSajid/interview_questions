# Question 5: REST API Design with Amazon API Gateway

## 🎯 Question
**How would you design, secure, and scale RESTful APIs using Amazon API Gateway for Bayer's integration platform? Discuss authentication, throttling, caching, versioning, request validation, and monitoring strategies.**

---

## 📋 Answer Overview

Amazon API Gateway is a fully managed service for creating RESTful APIs:
- **Managed infrastructure**: No servers to maintain
- **Built-in security**: API keys, OAuth, IAM, Cognito
- **Automatic scaling**: Handle millions of requests
- **Request/response transformation**: Modify data in-flight
- **Integration targets**: Lambda, HTTP endpoints, AWS services

For Bayer's integration platform, API Gateway provides:
- **Unified API layer**: Single entry point for all integrations
- **Security**: Authentication, authorization, rate limiting
- **Monitoring**: CloudWatch metrics and access logs
- **Versioning**: Support multiple API versions concurrently

---

## 🏗️ API Gateway Architecture

```
┌─────────────────────────────────────────────────────────┐
│               Client Applications                       │
│   (Manufacturing App, SAP, Salesforce, Mobile Apps)     │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
         ┌─────────────────────────┐
         │   Amazon API Gateway    │
         │  (REST API Endpoint)    │
         ├─────────────────────────┤
         │ - Authentication        │
         │ - Rate Limiting         │
         │ - Request Validation    │
         │ - Response Caching      │
         └────┬────────────────────┘
              │
     ┌────────┼────────────┐
     │        │            │
     ▼        ▼            ▼
┌────────┐ ┌────────┐ ┌────────┐
│Lambda  │ │Lambda  │ │Lambda  │
│Patient │ │Clinical│ │Mfg     │
│Service │ │Service │ │Service │
└────┬───┘ └───┬────┘ └───┬────┘
     │         │            │
     ▼         ▼            ▼
┌────────┐ ┌────────┐ ┌────────┐
│DynamoDB│ │RDS     │ │S3      │
│        │ │        │ │        │
└────────┘ └────────┘ └────────┘

API Structure:
https://api.bayer-integration.com/v1/patients/{id}
https://api.bayer-integration.com/v1/clinical-trials/{trialId}
https://api.bayer-integration.com/v1/manufacturing/batches
```

---

## 💻 Implementation Examples

### 1. API Gateway Definition (Terraform)

```hcl
# api-gateway.tf
# WHY: Infrastructure as Code for API Gateway
# REQUIRED FOR: Production-grade API management

resource "aws_api_gateway_rest_api" "integration_api" {
  name        = "bayer-integration-api-${var.environment}"
  description = "RESTful API for Bayer Integration Platform"
  
  # Enable request compression
  # WHY: Reduce bandwidth costs (50% savings for JSON)
  minimum_compression_size = 1024  # 1 KB
  
  endpoint_configuration {
    types = ["REGIONAL"]
    # WHY: Lower latency for regional clients
    # ALTERNATIVE: "EDGE" for global CDN distribution
  }
  
  tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

# Root resource (/)
resource "aws_api_gateway_resource" "root" {
  rest_api_id = aws_api_gateway_rest_api.integration_api.id
  parent_id   = aws_api_gateway_rest_api.integration_api.root_resource_id
  path_part   = "v1"
  # WHY: Version in URL path for backward compatibility
}

# /v1/patients resource
resource "aws_api_gateway_resource" "patients" {
  rest_api_id = aws_api_gateway_rest_api.integration_api.id
  parent_id   = aws_api_gateway_resource.root.id
  path_part   = "patients"
}

# /v1/patients/{patientId} resource
resource "aws_api_gateway_resource" "patient_by_id" {
  rest_api_id = aws_api_gateway_rest_api.integration_api.id
  parent_id   = aws_api_gateway_resource.patients.id
  path_part   = "{patientId}"
  # WHY: Path parameter for resource identification
}

# GET /v1/patients/{patientId}
resource "aws_api_gateway_method" "get_patient" {
  rest_api_id   = aws_api_gateway_rest_api.integration_api.id
  resource_id   = aws_api_gateway_resource.patient_by_id.id
  http_method   = "GET"
  authorization = "CUSTOM"
  authorizer_id = aws_api_gateway_authorizer.lambda_authorizer.id
  
  # Request validation
  # WHY: Reject invalid requests before invoking Lambda
  request_validator_id = aws_api_gateway_request_validator.validate_params.id
  
  # Path parameters
  request_parameters = {
    "method.request.path.patientId" = true  # Required
    "method.request.header.X-Correlation-ID" = false  # Optional
  }
  
  # API Key required
  # WHY: Track usage per client, enforce quotas
  api_key_required = true
}

# Lambda integration
resource "aws_api_gateway_integration" "get_patient_lambda" {
  rest_api_id             = aws_api_gateway_rest_api.integration_api.id
  resource_id             = aws_api_gateway_resource.patient_by_id.id
  http_method             = aws_api_gateway_method.get_patient.http_method
  integration_http_method = "POST"  # Lambda always uses POST
  type                    = "AWS_PROXY"
  uri                     = aws_lambda_function.get_patient.invoke_arn
  
  # WHY: AWS_PROXY passes entire request to Lambda
  # ALTERNATIVE: AWS (non-proxy) for request/response transformation in API Gateway
}

# Method response (200 OK)
resource "aws_api_gateway_method_response" "get_patient_200" {
  rest_api_id = aws_api_gateway_rest_api.integration_api.id
  resource_id = aws_api_gateway_resource.patient_by_id.id
  http_method = aws_api_gateway_method.get_patient.http_method
  status_code = "200"
  
  # Response headers
  # WHY: CORS support, caching control
  response_parameters = {
    "method.response.header.Access-Control-Allow-Origin" = true
    "method.response.header.Cache-Control" = true
  }
  
  # Response model (JSON schema)
  response_models = {
    "application/json" = aws_api_gateway_model.patient_response.name
  }
}

# Request validator
resource "aws_api_gateway_request_validator" "validate_params" {
  name                        = "validate-params-and-body"
  rest_api_id                 = aws_api_gateway_rest_api.integration_api.id
  validate_request_body       = true
  validate_request_parameters = true
  # WHY: Reject malformed requests early (cost savings)
}

# Response model (JSON schema)
resource "aws_api_gateway_model" "patient_response" {
  rest_api_id  = aws_api_gateway_rest_api.integration_api.id
  name         = "PatientResponse"
  description  = "Patient data schema"
  content_type = "application/json"
  
  # JSON Schema for validation
  # WHY: Ensure consistent API responses
  schema = jsonencode({
    "$schema" = "http://json-schema.org/draft-04/schema#"
    type      = "object"
    required  = ["patientId", "firstName", "lastName"]
    properties = {
      patientId = {
        type = "string"
        pattern = "^[A-Z0-9]{10}$"
      }
      firstName = { type = "string" }
      lastName  = { type = "string" }
      dateOfBirth = {
        type   = "string"
        format = "date"
      }
    }
  })
}

# Lambda authorizer (custom authentication)
resource "aws_api_gateway_authorizer" "lambda_authorizer" {
  name                   = "lambda-authorizer"
  rest_api_id            = aws_api_gateway_rest_api.integration_api.id
  authorizer_uri         = aws_lambda_function.authorizer.invoke_arn
  authorizer_credentials = aws_iam_role.api_gateway_auth_invocation.arn
  type                   = "TOKEN"
  identity_source        = "method.request.header.Authorization"
  
  # Cache TTL
  # WHY: Avoid invoking authorizer for every request
  # COST SAVINGS: 99% reduction in authorizer invocations
  authorizer_result_ttl_in_seconds = 300  # 5 minutes
}

# Usage plan (rate limiting)
resource "aws_api_gateway_usage_plan" "standard" {
  name = "standard-usage-plan"
  
  api_stages {
    api_id = aws_api_gateway_rest_api.integration_api.id
    stage  = aws_api_gateway_stage.production.stage_name
  }
  
  # Rate limits
  # WHY: Prevent abuse, protect backend systems
  throttle_settings {
    burst_limit = 100   # Max concurrent requests
    rate_limit  = 50    # Requests per second
    # EXAMPLE: Manufacturing app can make 50 req/sec sustained
  }
  
  # Quota
  # WHY: Monthly request allowance per API key
  quota_settings {
    limit  = 100000     # 100K requests/month
    period = "MONTH"
    # COST: Predictable costs, prevents runaway usage
  }
}

# API key for client authentication
resource "aws_api_gateway_api_key" "manufacturing_app" {
  name        = "manufacturing-app-key"
  description = "API key for manufacturing application"
  enabled     = true
}

# Associate API key with usage plan
resource "aws_api_gateway_usage_plan_key" "manufacturing_app" {
  key_id        = aws_api_gateway_api_key.manufacturing_app.id
  key_type      = "API_KEY"
  usage_plan_id = aws_api_gateway_usage_plan.standard.id
}

# Deployment
resource "aws_api_gateway_deployment" "production" {
  rest_api_id = aws_api_gateway_rest_api.integration_api.id
  
  # WHY: Trigger redeployment when any resource changes
  triggers = {
    redeployment = sha1(jsonencode([
      aws_api_gateway_resource.patients.id,
      aws_api_gateway_method.get_patient.id,
      aws_api_gateway_integration.get_patient_lambda.id,
    ]))
  }
  
  lifecycle {
    create_before_destroy = true
  }
}

# Stage (production)
resource "aws_api_gateway_stage" "production" {
  deployment_id = aws_api_gateway_deployment.production.id
  rest_api_id   = aws_api_gateway_rest_api.integration_api.id
  stage_name    = "prod"
  
  # Enable caching
  # WHY: Reduce Lambda invocations (cost savings)
  cache_cluster_enabled = true
  cache_cluster_size    = "0.5"  # 0.5 GB cache
  
  # Access logging
  # WHY: Track API usage, debug issues
  access_log_settings {
    destination_arn = aws_cloudwatch_log_group.api_gateway_logs.arn
    format = jsonencode({
      requestId      = "$context.requestId"
      ip             = "$context.identity.sourceIp"
      caller         = "$context.identity.caller"
      user           = "$context.identity.user"
      requestTime    = "$context.requestTime"
      httpMethod     = "$context.httpMethod"
      resourcePath   = "$context.resourcePath"
      status         = "$context.status"
      protocol       = "$context.protocol"
      responseLength = "$context.responseLength"
      integrationLatency = "$context.integration.latency"
      responseLatency = "$context.responseLatency"
    })
  }
  
  # X-Ray tracing
  # WHY: Distributed tracing for debugging
  xray_tracing_enabled = true
}

# CloudWatch log group
resource "aws_cloudwatch_log_group" "api_gateway_logs" {
  name              = "/aws/apigateway/integration-api-${var.environment}"
  retention_in_days = 30
}
```

---

### 2. Lambda Authorizer (Custom Authentication)

```javascript
// authorizer.js
// WHY: Custom authentication logic (JWT, OAuth, API keys)
// INVOKED BY: API Gateway for each request (unless cached)

const jwt = require('jsonwebtoken');
const AWS = require('aws-sdk');
const secretsManager = new AWS.SecretsManager();

/**
 * Lambda authorizer handler
 * WHY: Validate JWT tokens, check permissions
 * RETURNS: IAM policy (allow/deny)
 */
exports.handler = async (event) => {
  console.log('Authorizer invoked', {
    methodArn: event.methodArn,
    authorizationToken: event.authorizationToken ? 'present' : 'missing'
  });
  
  // Extract token from Authorization header
  // FORMAT: "Bearer <JWT_TOKEN>"
  const token = event.authorizationToken.replace('Bearer ', '');
  
  try {
    // Get JWT secret from Secrets Manager
    // WHY: Don't hardcode secrets in Lambda code
    const secret = await getJWTSecret();
    
    // Verify JWT
    // WHY: Ensure token not tampered with, not expired
    const decoded = jwt.verify(token, secret, {
      algorithms: ['HS256'],
      maxAge: '1h'  // Token expires in 1 hour
    });
    
    console.log('Token verified', {
      userId: decoded.userId,
      roles: decoded.roles
    });
    
    // Generate IAM policy
    // WHY: Grant access to specific API resources
    const policy = generatePolicy(decoded.userId, 'Allow', event.methodArn, decoded);
    
    return policy;
    
  } catch (error) {
    console.error('Authorization failed', {
      error: error.message
    });
    
    // Deny access
    // WHY: Invalid/expired token
    throw new Error('Unauthorized');
  }
};

/**
 * Get JWT secret from Secrets Manager
 * WHY: Secure secret storage, automatic rotation
 */
async function getJWTSecret() {
  const secretName = process.env.JWT_SECRET_NAME;
  
  try {
    const data = await secretsManager.getSecretValue({
      SecretId: secretName
    }).promise();
    
    return JSON.parse(data.SecretString).jwtSecret;
    
  } catch (error) {
    console.error('Failed to get JWT secret', {
      error: error.message
    });
    throw error;
  }
}

/**
 * Generate IAM policy for API Gateway
 * WHY: Fine-grained access control
 */
function generatePolicy(principalId, effect, resource, context) {
  const authResponse = {
    principalId: principalId
  };
  
  if (effect && resource) {
    authResponse.policyDocument = {
      Version: '2012-10-17',
      Statement: [
        {
          Action: 'execute-api:Invoke',
          Effect: effect,  // 'Allow' or 'Deny'
          Resource: resource
          // EXAMPLE: arn:aws:execute-api:us-east-1:123456789012:abcdefg/prod/GET/v1/patients/*
        }
      ]
    };
  }
  
  // Context passed to Lambda function
  // WHY: Include user metadata (userId, roles) in request
  authResponse.context = {
    userId: context.userId,
    email: context.email,
    roles: JSON.stringify(context.roles),
    // USAGE: event.requestContext.authorizer.userId in Lambda
  };
  
  return authResponse;
}
```

---

### 3. API Handler with Request Validation

```javascript
// get-patient.js
// WHY: Lambda function backing GET /v1/patients/{patientId}

const AWS = require('aws-sdk');
const dynamodb = new AWS.DynamoDB.DocumentClient();

/**
 * Get patient by ID
 * WHY: Retrieve patient data from DynamoDB
 */
exports.handler = async (event) => {
  console.log('GET patient request', {
    pathParameters: event.pathParameters,
    userId: event.requestContext.authorizer.userId
  });
  
  // Extract path parameter
  // WHY: API Gateway passes {patientId} in pathParameters
  const patientId = event.pathParameters.patientId;
  
  // Validate patient ID format
  // WHY: Prevent SQL injection, invalid queries
  if (!/^[A-Z0-9]{10}$/.test(patientId)) {
    return {
      statusCode: 400,
      headers: getCORSHeaders(),
      body: JSON.stringify({
        error: 'Invalid patient ID format',
        message: 'Patient ID must be 10 alphanumeric characters'
      })
    };
  }
  
  // Check authorization
  // WHY: Ensure user has permission to view this patient
  const userId = event.requestContext.authorizer.userId;
  const userRoles = JSON.parse(event.requestContext.authorizer.roles);
  
  if (!userRoles.includes('HEALTHCARE_PROVIDER') && !userRoles.includes('ADMIN')) {
    return {
      statusCode: 403,
      headers: getCORSHeaders(),
      body: JSON.stringify({
        error: 'Forbidden',
        message: 'Insufficient permissions to view patient data'
      })
    };
  }
  
  try {
    // Get patient from DynamoDB
    const result = await dynamodb.get({
      TableName: process.env.PATIENTS_TABLE,
      Key: { patientId }
    }).promise();
    
    if (!result.Item) {
      return {
        statusCode: 404,
        headers: getCORSHeaders(),
        body: JSON.stringify({
          error: 'Not Found',
          message: `Patient ${patientId} not found`
        })
      };
    }
    
    // Remove sensitive fields
    // WHY: Don't expose SSN, billing info in API
    const patient = sanitizePatientData(result.Item);
    
    return {
      statusCode: 200,
      headers: {
        ...getCORSHeaders(),
        'Cache-Control': 'max-age=300',  // Cache for 5 minutes
        'X-Request-ID': event.requestContext.requestId
      },
      body: JSON.stringify(patient)
    };
    
  } catch (error) {
    console.error('Failed to get patient', {
      patientId,
      error: error.message
    });
    
    return {
      statusCode: 500,
      headers: getCORSHeaders(),
      body: JSON.stringify({
        error: 'Internal Server Error',
        message: 'Failed to retrieve patient data'
      })
    };
  }
};

/**
 * CORS headers
 * WHY: Allow browser-based apps to call API
 */
function getCORSHeaders() {
  return {
    'Access-Control-Allow-Origin': process.env.ALLOWED_ORIGIN || '*',
    'Access-Control-Allow-Headers': 'Content-Type,Authorization,X-Api-Key',
    'Access-Control-Allow-Methods': 'GET,POST,PUT,DELETE,OPTIONS',
    'Content-Type': 'application/json'
  };
}

/**
 * Remove sensitive fields from patient data
 * WHY: HIPAA compliance, data minimization
 */
function sanitizePatientData(patient) {
  const { ssn, billingInfo, internalNotes, ...publicData } = patient;
  return publicData;
}
```

---

## 📊 API Gateway Best Practices

### ✅ Do's
1. **Use API versioning** (v1, v2 in URL path)
2. **Enable caching** for GET requests (reduce Lambda invocations)
3. **Implement rate limiting** (protect backend from abuse)
4. **Validate requests** in API Gateway (before Lambda invocation)
5. **Use Lambda authorizers** for custom authentication
6. **Enable CloudWatch logs** (access logs + execution logs)
7. **Use stage variables** (different Lambda per environment)
8. **Implement CORS** properly (preflight OPTIONS requests)
9. **Use compression** (minimum_compression_size)
10. **Monitor with X-Ray** (distributed tracing)

### ❌ Don'ts
1. **Don't skip authentication** (always require API keys or JWT)
2. **Don't expose internal errors** (sanitize error messages)
3. **Don't cache sensitive data** (use Cache-Control: no-store)
4. **Don't use GET for mutations** (use POST/PUT/DELETE)
5. **Don't return 200 for errors** (use proper HTTP status codes)
6. **Don't forget CORS** (browsers will block requests)
7. **Don't hardcode URLs** (use stage variables)
8. **Don't skip request validation** (costs add up)

---

## 💰 API Gateway Pricing

| Component | Cost | Optimization |
|-----------|------|--------------|
| **API Requests** | $3.50/million | Use caching, batch requests |
| **Caching** | $0.02/hour/GB | Enable for GET endpoints |
| **Data Transfer** | $0.09/GB | Use compression |

**Example Cost:**
- 10 million requests/month
- 50% cache hit rate (5M Lambda invocations saved)
- **Cost: $35 + $14 (cache) = $49/month**

---

**Estimated Reading Time**: 16-19 minutes  
**Difficulty Level**: ⭐⭐⭐⭐ Advanced  
**Prerequisites**: REST APIs, AWS Lambda, authentication  
**Bayer Job Alignment**: 95% - Critical for integration APIs
