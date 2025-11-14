# Testing Strategies - Serverless Order Processing Microservice

## 📋 Overview

Comprehensive testing approach covering unit tests, integration tests, E2E tests, load testing, and security testing.

---

## 1️⃣ Unit Testing with Jest

### Jest Configuration

```json
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src', '<rootDir>/tests'],
  testMatch: ['**/__tests__/**/*.ts', '**/?(*.)+(spec|test).ts'],
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/**/*.interface.ts'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  },
  setupFiles: ['<rootDir>/tests/setup.ts']
};
```

### Setup File

```typescript
// tests/setup.ts
process.env.AWS_REGION = 'us-east-1';
process.env.ORDERS_TABLE = 'orders-test';
process.env.INVENTORY_TABLE = 'inventory-test';
process.env.ORDER_QUEUE_URL = 'https://sqs.us-east-1.amazonaws.com/123456789/test-queue';
process.env.NOTIFICATION_TOPIC_ARN = 'arn:aws:sns:us-east-1:123456789:test-topic';
```

### Unit Test Examples

```typescript
// tests/unit/orderService.test.ts
import { OrderService } from '../../src/services/orderService';
import { DynamoDB } from 'aws-sdk';

// Mock AWS SDK
jest.mock('aws-sdk', () => {
  const mockDocClient = {
    put: jest.fn().mockReturnThis(),
    get: jest.fn().mockReturnThis(),
    query: jest.fn().mockReturnThis(),
    update: jest.fn().mockReturnThis(),
    promise: jest.fn()
  };
  
  return {
    DynamoDB: {
      DocumentClient: jest.fn(() => mockDocClient)
    }
  };
});

describe('OrderService', () => {
  let orderService: OrderService;
  let mockDocClient: any;
  
  beforeEach(() => {
    orderService = new OrderService();
    mockDocClient = new DynamoDB.DocumentClient();
    jest.clearAllMocks();
  });
  
  describe('createOrder', () => {
    it('should create order successfully', async () => {
      const orderData = {
        customerId: 'CUST-123',
        items: [
          { productId: 'PROD-1', quantity: 2, price: 29.99 }
        ]
      };
      
      mockDocClient.promise.mockResolvedValueOnce({});
      
      const result = await orderService.createOrder(orderData);
      
      expect(result).toHaveProperty('orderId');
      expect(result.customerId).toBe('CUST-123');
      expect(result.status).toBe('PENDING');
      expect(mockDocClient.put).toHaveBeenCalledTimes(1);
    });
    
    it('should throw error for invalid customer ID', async () => {
      const orderData = {
        customerId: '',
        items: []
      };
      
      await expect(orderService.createOrder(orderData))
        .rejects
        .toThrow('Invalid customer ID');
    });
    
    it('should calculate total correctly', async () => {
      const orderData = {
        customerId: 'CUST-123',
        items: [
          { productId: 'PROD-1', quantity: 2, price: 29.99 },
          { productId: 'PROD-2', quantity: 1, price: 19.99 }
        ]
      };
      
      mockDocClient.promise.mockResolvedValueOnce({});
      
      const result = await orderService.createOrder(orderData);
      
      expect(result.total).toBe(79.97);
    });
  });
  
  describe('getOrderById', () => {
    it('should retrieve order successfully', async () => {
      const mockOrder = {
        Item: {
          orderId: 'ORDER-123',
          customerId: 'CUST-123',
          status: 'PENDING'
        }
      };
      
      mockDocClient.promise.mockResolvedValueOnce(mockOrder);
      
      const result = await orderService.getOrderById('ORDER-123');
      
      expect(result).toEqual(mockOrder.Item);
      expect(mockDocClient.get).toHaveBeenCalledWith({
        TableName: 'orders-test',
        Key: { orderId: 'ORDER-123' }
      });
    });
    
    it('should return null for non-existent order', async () => {
      mockDocClient.promise.mockResolvedValueOnce({});
      
      const result = await orderService.getOrderById('INVALID-ID');
      
      expect(result).toBeNull();
    });
  });
  
  describe('updateOrderStatus', () => {
    it('should update status successfully', async () => {
      const mockUpdated = {
        Attributes: {
          orderId: 'ORDER-123',
          status: 'COMPLETED',
          updatedAt: new Date().toISOString()
        }
      };
      
      mockDocClient.promise.mockResolvedValueOnce(mockUpdated);
      
      const result = await orderService.updateOrderStatus('ORDER-123', 'COMPLETED');
      
      expect(result.status).toBe('COMPLETED');
      expect(mockDocClient.update).toHaveBeenCalled();
    });
  });
});
```

### Lambda Handler Unit Test

```typescript
// tests/unit/handlers/api.test.ts
import { handler } from '../../../src/handlers/api';
import { OrderService } from '../../../src/services/orderService';

jest.mock('../../../src/services/orderService');

describe('API Handler', () => {
  let mockOrderService: jest.Mocked<OrderService>;
  
  beforeEach(() => {
    mockOrderService = new OrderService() as jest.Mocked<OrderService>;
    jest.clearAllMocks();
  });
  
  describe('POST /orders', () => {
    it('should create order and return 201', async () => {
      const event = {
        httpMethod: 'POST',
        path: '/orders',
        body: JSON.stringify({
          customerId: 'CUST-123',
          items: [
            { productId: 'PROD-1', quantity: 2, price: 29.99 }
          ]
        })
      };
      
      const mockOrder = {
        orderId: 'ORDER-123',
        customerId: 'CUST-123',
        status: 'PENDING',
        total: 59.98
      };
      
      mockOrderService.createOrder.mockResolvedValueOnce(mockOrder);
      
      const result = await handler(event);
      
      expect(result.statusCode).toBe(201);
      expect(JSON.parse(result.body)).toEqual(mockOrder);
    });
    
    it('should return 400 for invalid request', async () => {
      const event = {
        httpMethod: 'POST',
        path: '/orders',
        body: JSON.stringify({
          customerId: ''
        })
      };
      
      const result = await handler(event);
      
      expect(result.statusCode).toBe(400);
    });
    
    it('should return 500 for internal errors', async () => {
      const event = {
        httpMethod: 'POST',
        path: '/orders',
        body: JSON.stringify({
          customerId: 'CUST-123',
          items: []
        })
      };
      
      mockOrderService.createOrder.mockRejectedValueOnce(
        new Error('Database error')
      );
      
      const result = await handler(event);
      
      expect(result.statusCode).toBe(500);
    });
  });
  
  describe('GET /orders/{orderId}', () => {
    it('should return order details', async () => {
      const event = {
        httpMethod: 'GET',
        path: '/orders/ORDER-123',
        pathParameters: { orderId: 'ORDER-123' }
      };
      
      const mockOrder = {
        orderId: 'ORDER-123',
        customerId: 'CUST-123',
        status: 'PENDING'
      };
      
      mockOrderService.getOrderById.mockResolvedValueOnce(mockOrder);
      
      const result = await handler(event);
      
      expect(result.statusCode).toBe(200);
      expect(JSON.parse(result.body)).toEqual(mockOrder);
    });
    
    it('should return 404 for non-existent order', async () => {
      const event = {
        httpMethod: 'GET',
        path: '/orders/INVALID',
        pathParameters: { orderId: 'INVALID' }
      };
      
      mockOrderService.getOrderById.mockResolvedValueOnce(null);
      
      const result = await handler(event);
      
      expect(result.statusCode).toBe(404);
    });
  });
});
```

---

## 2️⃣ Integration Testing

```typescript
// tests/integration/orderFlow.test.ts
import AWS from 'aws-sdk';
import { v4 as uuidv4 } from 'uuid';

// Use real AWS services in test account
const dynamodb = new AWS.DynamoDB.DocumentClient({
  region: 'us-east-1'
});

const sqs = new AWS.SQS({ region: 'us-east-1' });
const lambda = new AWS.Lambda({ region: 'us-east-1' });

describe('Order Processing Integration Tests', () => {
  const testOrdersTable = 'orders-integration-test';
  const testQueueUrl = process.env.TEST_QUEUE_URL!;
  
  beforeAll(async () => {
    // Setup test data
    await seedTestData();
  });
  
  afterAll(async () => {
    // Cleanup test data
    await cleanupTestData();
  });
  
  it('should process complete order flow', async () => {
    // Step 1: Create order via API
    const orderId = uuidv4();
    const orderData = {
      orderId,
      customerId: 'TEST-CUST-1',
      items: [
        { productId: 'TEST-PROD-1', quantity: 2, price: 29.99 }
      ],
      status: 'PENDING',
      createdAt: new Date().toISOString()
    };
    
    await dynamodb.put({
      TableName: testOrdersTable,
      Item: orderData
    }).promise();
    
    // Step 2: Send message to SQS
    await sqs.sendMessage({
      QueueUrl: testQueueUrl,
      MessageBody: JSON.stringify(orderData),
      MessageGroupId: orderId,
      MessageDeduplicationId: orderId
    }).promise();
    
    // Step 3: Wait for Lambda to process
    await sleep(5000);
    
    // Step 4: Verify order was processed
    const result = await dynamodb.get({
      TableName: testOrdersTable,
      Key: { orderId }
    }).promise();
    
    expect(result.Item).toBeDefined();
    expect(result.Item?.status).toBe('COMPLETED');
  });
  
  it('should handle inventory reservation', async () => {
    // Test inventory check and reservation
    const productId = 'TEST-PROD-1';
    const quantity = 2;
    
    // Get initial inventory
    const initialInventory = await dynamodb.get({
      TableName: 'inventory-integration-test',
      Key: { productId }
    }).promise();
    
    const initialQty = initialInventory.Item?.quantity || 0;
    
    // Create order
    const orderId = uuidv4();
    await createTestOrder(orderId, productId, quantity);
    
    // Wait for processing
    await sleep(3000);
    
    // Verify inventory was reserved
    const updatedInventory = await dynamodb.get({
      TableName: 'inventory-integration-test',
      Key: { productId }
    }).promise();
    
    expect(updatedInventory.Item?.quantity).toBe(initialQty - quantity);
  });
});

async function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms));
}
```

---

## 3️⃣ API Testing with Postman

### Postman Collection

```json
{
  "info": {
    "name": "Order Processing API",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "variable": [
    {
      "key": "baseUrl",
      "value": "https://api-dev.example.com"
    }
  ],
  "item": [
    {
      "name": "Create Order",
      "request": {
        "method": "POST",
        "header": [
          {
            "key": "Content-Type",
            "value": "application/json"
          }
        ],
        "body": {
          "mode": "raw",
          "raw": "{\n  \"customerId\": \"CUST-123\",\n  \"items\": [\n    {\n      \"productId\": \"PROD-1\",\n      \"quantity\": 2,\n      \"price\": 29.99\n    }\n  ]\n}"
        },
        "url": {
          "raw": "{{baseUrl}}/orders",
          "host": ["{{baseUrl}}"],
          "path": ["orders"]
        }
      },
      "response": []
    },
    {
      "name": "Get Order",
      "request": {
        "method": "GET",
        "url": {
          "raw": "{{baseUrl}}/orders/{{orderId}}",
          "host": ["{{baseUrl}}"],
          "path": ["orders", "{{orderId}}"]
        }
      },
      "response": []
    }
  ]
}
```

### Newman (CLI) Testing

```powershell
# scripts/run-api-tests.ps1
param(
    [Parameter(Mandatory=$true)]
    [string]$Environment
)

$apiUrl = switch ($Environment) {
    'dev' { 'https://api-dev.example.com' }
    'staging' { 'https://api-staging.example.com' }
    'production' { 'https://api.example.com' }
}

Write-Host "Running API tests against $apiUrl..." -ForegroundColor Cyan

newman run tests/postman/collection.json `
    --environment tests/postman/environment-$Environment.json `
    --reporters cli,json,html `
    --reporter-html-export reports/api-tests-$Environment.html `
    --bail

if ($LASTEXITCODE -ne 0) {
    Write-Host "API tests failed" -ForegroundColor Red
    exit 1
}

Write-Host "API tests passed" -ForegroundColor Green
```

---

## 4️⃣ Load Testing with Artillery

### Artillery Configuration

```yaml
# tests/load/artillery-config.yml
config:
  target: 'https://api-staging.example.com'
  phases:
    # Warm up
    - duration: 60
      arrivalRate: 5
      name: "Warm up"
    
    # Ramp up
    - duration: 120
      arrivalRate: 5
      rampTo: 50
      name: "Ramp up load"
    
    # Sustained load
    - duration: 300
      arrivalRate: 50
      name: "Sustained load"
    
    # Spike test
    - duration: 60
      arrivalRate: 200
      name: "Spike test"
  
  plugins:
    expect: {}
    metrics-by-endpoint: {}
  
  processor: "./load-test-processor.js"

scenarios:
  - name: "Create and retrieve orders"
    weight: 70
    flow:
      - post:
          url: "/orders"
          json:
            customerId: "{{ $randomString() }}"
            items:
              - productId: "{{ $randomString() }}"
                quantity: "{{ $randomNumber(1, 5) }}"
                price: 29.99
          capture:
            - json: "$.orderId"
              as: "orderId"
          expect:
            - statusCode: 201
      
      - think: 2
      
      - get:
          url: "/orders/{{ orderId }}"
          expect:
            - statusCode: 200
            - hasProperty: "orderId"
  
  - name: "Get order details"
    weight: 30
    flow:
      - get:
          url: "/orders/{{ $randomString() }}"
          expect:
            - statusCode: [200, 404]
```

### Load Test Processor

```javascript
// tests/load/load-test-processor.js
module.exports = {
  randomString: function() {
    return `TEST-${Math.random().toString(36).substr(2, 9)}`;
  },
  
  randomNumber: function(min, max) {
    return Math.floor(Math.random() * (max - min + 1)) + min;
  }
};
```

### Run Load Tests

```powershell
# scripts/run-load-tests.ps1
param(
    [Parameter(Mandatory=$true)]
    [string]$Environment
)

Write-Host "Running load tests against $Environment..." -ForegroundColor Cyan

artillery run tests/load/artillery-config.yml `
    --target "https://api-$Environment.example.com" `
    --output reports/load-test-$Environment.json

artillery report reports/load-test-$Environment.json `
    --output reports/load-test-$Environment.html

Write-Host "Load test complete. Report: reports/load-test-$Environment.html" -ForegroundColor Green
```

---

## 5️⃣ Security Testing

### OWASP ZAP Scan

```powershell
# scripts/security-scan.ps1
param(
    [Parameter(Mandatory=$true)]
    [string]$TargetUrl
)

Write-Host "Running security scan on $TargetUrl..." -ForegroundColor Cyan

docker run -v ${PWD}:/zap/wrk/:rw `
    -t owasp/zap2docker-stable zap-baseline.py `
    -t $TargetUrl `
    -r security-report.html

Write-Host "Security scan complete" -ForegroundColor Green
```

### Dependency Vulnerability Scan

```powershell
# Check for vulnerable dependencies
npm audit --audit-level=high

# Fix vulnerabilities
npm audit fix

# Check specific packages
npm outdated
```

---

## 6️⃣ End-to-End Testing

```typescript
// tests/e2e/orderLifecycle.test.ts
import axios from 'axios';
import AWS from 'aws-sdk';

const API_URL = process.env.API_URL || 'https://api-dev.example.com';
const dynamodb = new AWS.DynamoDB.DocumentClient();

describe('E2E: Order Lifecycle', () => {
  it('should complete full order lifecycle', async () => {
    // Step 1: Create order
    const createResponse = await axios.post(`${API_URL}/orders`, {
      customerId: 'E2E-CUST-1',
      items: [
        { productId: 'PROD-1', quantity: 2, price: 29.99 }
      ]
    });
    
    expect(createResponse.status).toBe(201);
    const orderId = createResponse.data.orderId;
    
    // Step 2: Verify order in database
    const dbResult = await dynamodb.get({
      TableName: 'orders-dev',
      Key: { orderId }
    }).promise();
    
    expect(dbResult.Item).toBeDefined();
    expect(dbResult.Item?.status).toBe('PENDING');
    
    // Step 3: Wait for processing (SQS + Lambda)
    await sleep(10000);
    
    // Step 4: Get order status
    const getResponse = await axios.get(`${API_URL}/orders/${orderId}`);
    
    expect(getResponse.status).toBe(200);
    expect(getResponse.data.status).toBe('COMPLETED');
    
    // Step 5: Verify inventory was updated
    const inventoryResult = await dynamodb.get({
      TableName: 'inventory-dev',
      Key: { productId: 'PROD-1' }
    }).promise();
    
    expect(inventoryResult.Item?.reservedQuantity).toBeGreaterThanOrEqual(2);
  });
});
```

---

## 7️⃣ Test Automation in CI/CD

```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci
      - run: npm run test:unit
      - run: npm run test:coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info

  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm run test:integration
  
  e2e-tests:
    runs-on: ubuntu-latest
    needs: integration-tests
    steps:
      - uses: actions/checkout@v3
      - run: npm ci
      - run: npm run test:e2e
```

---

## 🎯 Testing Best Practices

### DO ✅
1. Maintain >80% code coverage
2. Use mocks for external dependencies
3. Test error scenarios
4. Run tests in CI/CD pipeline
5. Perform load testing before production
6. Implement integration tests
7. Use test data factories
8. Test timeout scenarios
9. Validate input/output schemas
10. Monitor test execution time

### DON'T ❌
1. Don't test implementation details
2. Don't skip edge cases
3. Don't use production data in tests
4. Don't ignore flaky tests
5. Don't test third-party libraries

---

**Next:** [Challenges & Solutions](./13_CHALLENGES_SOLUTIONS.md)

---

**Last Updated**: November 12, 2025
