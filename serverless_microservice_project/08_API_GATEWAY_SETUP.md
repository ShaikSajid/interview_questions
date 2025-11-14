# API Gateway Setup - Serverless Order Processing Microservice

## 📋 Overview

Complete REST API configuration with API Gateway including custom domains, CORS, throttling, caching, and request/response mapping.

---

## 🎯 API Architecture

```javascript
const apiArchitecture = {
  type: 'REST API',
  authentication: ['AWS_IAM', 'Cognito', 'API_KEY'],
  stages: ['dev', 'staging', 'production'],
  features: ['Throttling', 'Caching', 'CORS', 'Request Validation', 'Custom Domain']
};
```

---

## 1️⃣ CloudFormation - API Gateway

```yaml
# infrastructure/cloudformation/api-gateway-stack.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'API Gateway for Order Processing'

Parameters:
  Environment:
    Type: String
    Default: dev

Resources:
  OrderProcessingApi:
    Type: AWS::ApiGateway::RestApi
    Properties:
      Name: !Sub '${Environment}-order-processing-api'
      Description: REST API for order management
      EndpointConfiguration:
        Types:
          - REGIONAL
      Policy:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal: '*'
            Action: 'execute-api:Invoke'
            Resource: '*'

  # /orders resource
  OrdersResource:
    Type: AWS::ApiGateway::Resource
    Properties:
      RestApiId: !Ref OrderProcessingApi
      ParentId: !GetAtt OrderProcessingApi.RootResourceId
      PathPart: orders

  # POST /orders
  CreateOrderMethod:
    Type: AWS::ApiGateway::Method
    Properties:
      RestApiId: !Ref OrderProcessingApi
      ResourceId: !Ref OrdersResource
      HttpMethod: POST
      AuthorizationType: AWS_IAM
      Integration:
        Type: AWS_PROXY
        IntegrationHttpMethod: POST
        Uri: !Sub 'arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${ApiHandlerFunction.Arn}/invocations'
      MethodResponses:
        - StatusCode: 201
          ResponseModels:
            application/json: Empty
        - StatusCode: 400
        - StatusCode: 500

  # GET /orders
  ListOrdersMethod:
    Type: AWS::ApiGateway::Method
    Properties:
      RestApiId: !Ref OrderProcessingApi
      ResourceId: !Ref OrdersResource
      HttpMethod: GET
      AuthorizationType: AWS_IAM
      RequestParameters:
        method.request.querystring.customerId: true
        method.request.querystring.limit: false
      Integration:
        Type: AWS_PROXY
        IntegrationHttpMethod: POST
        Uri: !Sub 'arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${ApiHandlerFunction.Arn}/invocations'

  # /orders/{orderId}
  OrderIdResource:
    Type: AWS::ApiGateway::Resource
    Properties:
      RestApiId: !Ref OrderProcessingApi
      ParentId: !Ref OrdersResource
      PathPart: '{orderId}'

  # GET /orders/{orderId}
  GetOrderMethod:
    Type: AWS::ApiGateway::Method
    Properties:
      RestApiId: !Ref OrderProcessingApi
      ResourceId: !Ref OrderIdResource
      HttpMethod: GET
      AuthorizationType: AWS_IAM
      RequestParameters:
        method.request.path.orderId: true
      Integration:
        Type: AWS_PROXY
        IntegrationHttpMethod: POST
        Uri: !Sub 'arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${ApiHandlerFunction.Arn}/invocations'

  # Lambda Permission
  ApiGatewayInvokePermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref ApiHandlerFunction
      Action: lambda:InvokeFunction
      Principal: apigateway.amazonaws.com
      SourceArn: !Sub 'arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${OrderProcessingApi}/*'

  # Deployment
  ApiDeployment:
    Type: AWS::ApiGateway::Deployment
    DependsOn:
      - CreateOrderMethod
      - ListOrdersMethod
      - GetOrderMethod
    Properties:
      RestApiId: !Ref OrderProcessingApi
      StageName: !Ref Environment
      StageDescription:
        ThrottlingBurstLimit: !If [IsProduction, 2000, 200]
        ThrottlingRateLimit: !If [IsProduction, 1000, 100]
        CachingEnabled: !If [IsProduction, true, false]
        CacheTtlInSeconds: 300
        CacheDataEncrypted: true
        LoggingLevel: INFO
        DataTraceEnabled: true
        MetricsEnabled: true

  # Usage Plan
  ApiUsagePlan:
    Type: AWS::ApiGateway::UsagePlan
    Properties:
      UsagePlanName: !Sub '${Environment}-usage-plan'
      ApiStages:
        - ApiId: !Ref OrderProcessingApi
          Stage: !Ref Environment
      Throttle:
        BurstLimit: !If [IsProduction, 2000, 200]
        RateLimit: !If [IsProduction, 1000, 100]
      Quota:
        Limit: !If [IsProduction, 1000000, 100000]
        Period: MONTH

  # API Key
  ApiKey:
    Type: AWS::ApiGateway::ApiKey
    Properties:
      Name: !Sub '${Environment}-api-key'
      Enabled: true

  # Associate Key with Usage Plan
  UsagePlanKey:
    Type: AWS::ApiGateway::UsagePlanKey
    Properties:
      KeyId: !Ref ApiKey
      KeyType: API_KEY
      UsagePlanId: !Ref ApiUsagePlan

Conditions:
  IsProduction: !Equals [!Ref Environment, 'production']

Outputs:
  ApiUrl:
    Value: !Sub 'https://${OrderProcessingApi}.execute-api.${AWS::Region}.amazonaws.com/${Environment}'
    Export:
      Name: !Sub '${Environment}-ApiUrl'
  
  ApiKey:
    Value: !Ref ApiKey
    Export:
      Name: !Sub '${Environment}-ApiKey'
```

---

## 2️⃣ CORS Configuration

```yaml
# Add OPTIONS method for CORS preflight
OptionsMethod:
  Type: AWS::ApiGateway::Method
  Properties:
    RestApiId: !Ref OrderProcessingApi
    ResourceId: !Ref OrdersResource
    HttpMethod: OPTIONS
    AuthorizationType: NONE
    Integration:
      Type: MOCK
      RequestTemplates:
        application/json: '{"statusCode": 200}'
      IntegrationResponses:
        - StatusCode: 200
          ResponseParameters:
            method.response.header.Access-Control-Allow-Headers: "'Content-Type,X-Amz-Date,Authorization,X-Api-Key'"
            method.response.header.Access-Control-Allow-Methods: "'GET,POST,PUT,DELETE,OPTIONS'"
            method.response.header.Access-Control-Allow-Origin: "'*'"
          ResponseTemplates:
            application/json: ''
    MethodResponses:
      - StatusCode: 200
        ResponseParameters:
          method.response.header.Access-Control-Allow-Headers: false
          method.response.header.Access-Control-Allow-Methods: false
          method.response.header.Access-Control-Allow-Origin: false
```

---

## 3️⃣ Custom Domain Setup

```powershell
# Request ACM certificate
aws acm request-certificate `
  --domain-name api.yourdomain.com `
  --validation-method DNS `
  --region us-east-1

# Create custom domain
aws apigateway create-domain-name `
  --domain-name api.yourdomain.com `
  --certificate-arn arn:aws:acm:us-east-1:123456789012:certificate/abc123 `
  --endpoint-configuration types=REGIONAL `
  --security-policy TLS_1_2

# Create base path mapping
aws apigateway create-base-path-mapping `
  --domain-name api.yourdomain.com `
  --rest-api-id abc123 `
  --stage production
```

---

## 4️⃣ Request Validation

```yaml
RequestValidator:
  Type: AWS::ApiGateway::RequestValidator
  Properties:
    RestApiId: !Ref OrderProcessingApi
    Name: BodyAndParametersValidator
    ValidateRequestBody: true
    ValidateRequestParameters: true

CreateOrderModel:
  Type: AWS::ApiGateway::Model
  Properties:
    RestApiId: !Ref OrderProcessingApi
    Name: CreateOrderModel
    ContentType: application/json
    Schema:
      $schema: 'http://json-schema.org/draft-04/schema#'
      type: object
      required:
        - customerId
        - items
      properties:
        customerId:
          type: string
        items:
          type: array
          items:
            type: object
            required:
              - productId
              - quantity
            properties:
              productId:
                type: string
              quantity:
                type: integer
                minimum: 1
```

---

## 5️⃣ Testing API

```powershell
# Get API endpoint
$API_URL = aws cloudformation describe-stacks `
  --stack-name dev-api-gateway `
  --query 'Stacks[0].Outputs[?OutputKey==`ApiUrl`].OutputValue' `
  --output text

# Create order
$body = @{
  customerId = "CUST-001"
  items = @(
    @{
      productId = "PROD-001"
      quantity = 2
      unitPrice = 100
    }
  )
  shippingAddress = @{
    street = "123 Main St"
    city = "New York"
    state = "NY"
    zipCode = "10001"
    country = "USA"
  }
  billingAddress = @{
    street = "123 Main St"
    city = "New York"
    state = "NY"
    zipCode = "10001"
    country = "USA"
  }
} | ConvertTo-Json

Invoke-WebRequest -Uri "$API_URL/orders" `
  -Method POST `
  -Body $body `
  -ContentType "application/json"
```

---

## 🎯 Best Practices

### DO ✅
1. Enable caching for production
2. Set appropriate throttling limits
3. Use custom domains
4. Enable CloudWatch logging
5. Implement CORS properly
6. Use API keys or IAM auth
7. Enable request validation
8. Monitor API metrics
9. Use stages for environments
10. Implement WAF for security

### DON'T ❌
1. Don't expose without authentication
2. Don't skip rate limiting
3. Don't ignore monitoring
4. Don't hardcode API URLs
5. Don't skip error handling

---

**Next:** [CloudWatch Monitoring](./09_CLOUDWATCH_MONITORING.md)

---

**Last Updated**: November 12, 2025
