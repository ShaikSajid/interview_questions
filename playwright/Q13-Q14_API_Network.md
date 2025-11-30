# Playwright Interview Questions: Q13-Q14

## Q13: How do you perform API testing with Playwright?

### Answer:

Playwright provides a powerful `request` context for API testing that shares cookies/auth with browser tests, enabling end-to-end scenarios that combine UI and API testing.

### 1. API Testing Methods

| Feature | Description | Example |
|---------|-------------|---------|
| **request.get()** | GET request | `await request.get('/api/users')` |
| **request.post()** | POST request | `await request.post('/api/users', { data })` |
| **request.put()** | PUT request | `await request.put('/api/users/1', { data })` |
| **request.delete()** | DELETE request | `await request.delete('/api/users/1')` |
| **request.patch()** | PATCH request | `await request.patch('/api/users/1', { data })` |
| **request.fetch()** | Custom request | `await request.fetch(url, { method, headers })` |

### 2. Basic API Testing

```typescript
import { test, expect } from '@playwright/test';

test('basic API GET request', async ({ request }) => {
  // Simple GET request
  const response = await request.get('https://api.example.com/users');
  
  // Assert status
  expect(response.ok()).toBeTruthy();
  expect(response.status()).toBe(200);
  
  // Parse JSON response
  const users = await response.json();
  expect(users).toHaveLength(10);
  expect(users[0]).toHaveProperty('id');
  expect(users[0]).toHaveProperty('name');
  
  // Check headers
  expect(response.headers()['content-type']).toContain('application/json');
});

test('API POST request', async ({ request }) => {
  const newUser = {
    name: 'John Doe',
    email: 'john@example.com',
    role: 'admin'
  };
  
  const response = await request.post('https://api.example.com/users', {
    data: newUser
  });
  
  expect(response.status()).toBe(201);
  
  const createdUser = await response.json();
  expect(createdUser.name).toBe('John Doe');
  expect(createdUser).toHaveProperty('id');
});

test('API PUT and DELETE requests', async ({ request }) => {
  // Update user
  const updateData = { name: 'Jane Doe Updated' };
  const putResponse = await request.put('https://api.example.com/users/1', {
    data: updateData
  });
  
  expect(putResponse.status()).toBe(200);
  
  // Delete user
  const deleteResponse = await request.delete('https://api.example.com/users/1');
  expect(deleteResponse.status()).toBe(204);
  
  // Verify deletion
  const getResponse = await request.get('https://api.example.com/users/1');
  expect(getResponse.status()).toBe(404);
});
```

### 3. API Testing with Authentication

```typescript
test('API with Bearer token', async ({ request }) => {
  const response = await request.get('https://api.example.com/profile', {
    headers: {
      'Authorization': 'Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...'
    }
  });
  
  expect(response.ok()).toBeTruthy();
  const profile = await response.json();
  expect(profile.email).toBe('user@example.com');
});

test('API with Basic Auth', async ({ request }) => {
  const response = await request.get('https://api.example.com/data', {
    headers: {
      'Authorization': 'Basic ' + Buffer.from('username:password').toString('base64')
    }
  });
  
  expect(response.status()).toBe(200);
});

test.describe('API tests with shared auth', () => {
  let apiContext;
  
  test.beforeAll(async ({ playwright }) => {
    // Create API context with auth
    apiContext = await playwright.request.newContext({
      baseURL: 'https://api.example.com',
      extraHTTPHeaders: {
        'Authorization': 'Bearer YOUR_TOKEN',
        'Accept': 'application/json'
      }
    });
  });
  
  test.afterAll(async () => {
    await apiContext.dispose();
  });
  
  test('use shared context', async () => {
    const response = await apiContext.get('/protected/resource');
    expect(response.ok()).toBeTruthy();
  });
});
```

### 4. Real-World Scenario: E-commerce API Testing

```typescript
test.describe('E-commerce API workflow', () => {
  let authToken: string;
  let productId: string;
  let orderId: string;
  
  test('1. User registration', async ({ request }) => {
    const response = await request.post('https://api.shop.example.com/auth/register', {
      data: {
        email: 'test@example.com',
        password: 'SecurePass123!',
        firstName: 'Test',
        lastName: 'User'
      }
    });
    
    expect(response.status()).toBe(201);
    const data = await response.json();
    authToken = data.token;
    expect(authToken).toBeTruthy();
  });
  
  test('2. Browse products', async ({ request }) => {
    const response = await request.get('https://api.shop.example.com/products', {
      params: {
        category: 'electronics',
        minPrice: '100',
        maxPrice: '1000'
      }
    });
    
    expect(response.ok()).toBeTruthy();
    const products = await response.json();
    expect(products.items).toHaveLength(20);
    expect(products.total).toBeGreaterThan(20);
    
    productId = products.items[0].id;
  });
  
  test('3. Get product details', async ({ request }) => {
    const response = await request.get(`https://api.shop.example.com/products/${productId}`);
    
    expect(response.ok()).toBeTruthy();
    const product = await response.json();
    expect(product.id).toBe(productId);
    expect(product).toHaveProperty('name');
    expect(product).toHaveProperty('price');
    expect(product).toHaveProperty('stock');
    expect(product.stock).toBeGreaterThan(0);
  });
  
  test('4. Add to cart', async ({ request }) => {
    const response = await request.post('https://api.shop.example.com/cart/items', {
      headers: {
        'Authorization': `Bearer ${authToken}`
      },
      data: {
        productId: productId,
        quantity: 2
      }
    });
    
    expect(response.status()).toBe(201);
    const cart = await response.json();
    expect(cart.items).toHaveLength(1);
    expect(cart.items[0].productId).toBe(productId);
    expect(cart.items[0].quantity).toBe(2);
    expect(cart.total).toBeGreaterThan(0);
  });
  
  test('5. Create order', async ({ request }) => {
    const response = await request.post('https://api.shop.example.com/orders', {
      headers: {
        'Authorization': `Bearer ${authToken}`
      },
      data: {
        shippingAddress: {
          street: '123 Main St',
          city: 'New York',
          state: 'NY',
          zip: '10001'
        },
        paymentMethod: 'credit_card',
        paymentToken: 'tok_visa'
      }
    });
    
    expect(response.status()).toBe(201);
    const order = await response.json();
    orderId = order.id;
    expect(order.status).toBe('pending');
    expect(order.items).toHaveLength(1);
    expect(order.total).toBeGreaterThan(0);
  });
  
  test('6. Check order status', async ({ request }) => {
    const response = await request.get(`https://api.shop.example.com/orders/${orderId}`, {
      headers: {
        'Authorization': `Bearer ${authToken}`
      }
    });
    
    expect(response.ok()).toBeTruthy();
    const order = await response.json();
    expect(order.id).toBe(orderId);
    expect(['pending', 'processing', 'confirmed']).toContain(order.status);
  });
  
  test('7. Get order history', async ({ request }) => {
    const response = await request.get('https://api.shop.example.com/orders', {
      headers: {
        'Authorization': `Bearer ${authToken}`
      }
    });
    
    expect(response.ok()).toBeTruthy();
    const orders = await response.json();
    expect(orders.items.length).toBeGreaterThan(0);
    expect(orders.items[0].id).toBe(orderId);
  });
});
```

### 5. Combining UI and API Testing

```typescript
test('create user via API, verify in UI', async ({ page, request }) => {
  // Create user via API
  const apiResponse = await request.post('https://api.example.com/users', {
    data: {
      name: 'API User',
      email: 'apiuser@example.com',
      role: 'admin'
    }
  });
  
  expect(apiResponse.status()).toBe(201);
  const user = await apiResponse.json();
  
  // Verify in UI
  await page.goto('https://example.com/admin/users');
  await page.locator('#search').fill(user.email);
  await page.locator('#search-btn').click();
  
  await expect(page.locator('.user-row')).toHaveCount(1);
  await expect(page.locator('.user-name')).toHaveText('API User');
  await expect(page.locator('.user-role')).toHaveText('admin');
});

test('setup test data via API', async ({ page, request }) => {
  // Create multiple products via API
  const productIds = [];
  
  for (let i = 1; i <= 5; i++) {
    const response = await request.post('https://api.shop.example.com/products', {
      data: {
        name: `Test Product ${i}`,
        price: i * 100,
        stock: 50
      }
    });
    
    const product = await response.json();
    productIds.push(product.id);
  }
  
  // Test UI with created products
  await page.goto('https://shop.example.com/products');
  
  for (const id of productIds) {
    await expect(page.locator(`[data-product-id="${id}"]`)).toBeVisible();
  }
  
  // Cleanup via API
  for (const id of productIds) {
    await request.delete(`https://api.shop.example.com/products/${id}`);
  }
});
```

### 6. API Error Handling and Validation

```typescript
test('handle API errors', async ({ request }) => {
  // Test 404
  const notFound = await request.get('https://api.example.com/users/99999');
  expect(notFound.status()).toBe(404);
  const error404 = await notFound.json();
  expect(error404.message).toContain('not found');
  
  // Test 400 Bad Request
  const badRequest = await request.post('https://api.example.com/users', {
    data: {
      // Missing required fields
      name: 'John'
    }
  });
  expect(badRequest.status()).toBe(400);
  const error400 = await badRequest.json();
  expect(error400.errors).toContain('email is required');
  
  // Test 401 Unauthorized
  const unauthorized = await request.get('https://api.example.com/admin/users');
  expect(unauthorized.status()).toBe(401);
  
  // Test 403 Forbidden
  const forbidden = await request.delete('https://api.example.com/admin/users/1', {
    headers: {
      'Authorization': 'Bearer USER_TOKEN' // Not admin
    }
  });
  expect(forbidden.status()).toBe(403);
  
  // Test 500 Internal Server Error
  const serverError = await request.post('https://api.example.com/trigger-error');
  expect(serverError.status()).toBe(500);
});
```

### 7. API Response Validation

```typescript
test('validate API response structure', async ({ request }) => {
  const response = await request.get('https://api.example.com/products/1');
  expect(response.ok()).toBeTruthy();
  
  const product = await response.json();
  
  // Validate structure
  expect(product).toHaveProperty('id');
  expect(product).toHaveProperty('name');
  expect(product).toHaveProperty('price');
  expect(product).toHaveProperty('description');
  expect(product).toHaveProperty('category');
  expect(product).toHaveProperty('images');
  expect(product).toHaveProperty('createdAt');
  expect(product).toHaveProperty('updatedAt');
  
  // Validate types
  expect(typeof product.id).toBe('string');
  expect(typeof product.name).toBe('string');
  expect(typeof product.price).toBe('number');
  expect(Array.isArray(product.images)).toBeTruthy();
  
  // Validate values
  expect(product.price).toBeGreaterThan(0);
  expect(product.name.length).toBeGreaterThan(0);
  expect(product.category).toMatch(/^(electronics|clothing|books)$/);
  
  // Validate dates
  expect(new Date(product.createdAt).getTime()).toBeLessThan(Date.now());
});
```

### 8. API Performance Testing

```typescript
test('API performance benchmark', async ({ request }) => {
  const startTime = Date.now();
  
  const response = await request.get('https://api.example.com/products');
  
  const endTime = Date.now();
  const responseTime = endTime - startTime;
  
  expect(response.ok()).toBeTruthy();
  expect(responseTime).toBeLessThan(1000); // Should respond within 1 second
  
  console.log(`API response time: ${responseTime}ms`);
});

test('concurrent API requests', async ({ request }) => {
  const requests = [];
  const count = 10;
  
  const startTime = Date.now();
  
  for (let i = 0; i < count; i++) {
    requests.push(request.get(`https://api.example.com/products/${i + 1}`));
  }
  
  const responses = await Promise.all(requests);
  
  const endTime = Date.now();
  const totalTime = endTime - startTime;
  
  // All requests successful
  responses.forEach(response => {
    expect(response.ok()).toBeTruthy();
  });
  
  console.log(`${count} concurrent requests completed in ${totalTime}ms`);
  expect(totalTime).toBeLessThan(3000); // All 10 should complete within 3 seconds
});
```

### 9. Best Practices

✅ **GOOD: Use request context for API testing**
```typescript
const response = await request.post('/api/users', { data });
expect(response.ok()).toBeTruthy();
```

❌ **BAD: Use page.evaluate for API calls**
```typescript
const response = await page.evaluate(async () => {
  return await fetch('/api/users', { method: 'POST' });
});
```

✅ **GOOD: Combine UI and API for efficiency**
```typescript
// Setup via API
await request.post('/api/users', { data: testUser });
// Test UI
await page.goto('/users');
```

❌ **BAD: Create everything through UI**
```typescript
// Slow
await page.fill('#name', 'Test');
await page.click('#submit');
```

### Related Topics
- **Q14**: Network interception
- **Q15**: Authentication in tests
- **Q27**: API mocking

---

## Q14: How do you intercept and modify network requests?

### Answer:

Playwright can intercept, monitor, modify, and mock network requests and responses using `page.route()` and `page.waitForResponse()` methods.

### 1. Network Interception Methods

| Method | Purpose | Example |
|--------|---------|---------|
| **page.route()** | Intercept and modify requests | `await page.route('**/*.css', route => route.abort())` |
| **page.unroute()** | Remove route handler | `await page.unroute(url)` |
| **route.continue()** | Continue with modifications | `await route.continue({ headers })` |
| **route.fulfill()** | Mock response | `await route.fulfill({ body: 'mocked' })` |
| **route.abort()** | Block request | `await route.abort('failed')` |
| **page.on('request')** | Monitor requests | `page.on('request', req => {})` |
| **page.on('response')** | Monitor responses | `page.on('response', res => {})` |

### 2. Basic Request Interception

```typescript
import { test, expect } from '@playwright/test';

test('block CSS and images', async ({ page }) => {
  // Block CSS files
  await page.route('**/*.css', route => route.abort());
  
  // Block images
  await page.route('**/*.{png,jpg,jpeg,gif}', route => route.abort());
  
  await page.goto('https://example.com');
  
  // Page loads without images and CSS
  await expect(page.locator('h1')).toBeVisible();
});

test('continue with modified headers', async ({ page }) => {
  await page.route('**/*', route => {
    const headers = {
      ...route.request().headers(),
      'X-Custom-Header': 'CustomValue',
      'Authorization': 'Bearer test-token'
    };
    route.continue({ headers });
  });
  
  await page.goto('https://example.com');
});
```

### 3. Mock API Responses

```typescript
test('mock API response', async ({ page }) => {
  await page.route('**/api/users', route => {
    route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify([
        { id: 1, name: 'Mocked User 1' },
        { id: 2, name: 'Mocked User 2' }
      ])
    });
  });
  
  await page.goto('https://example.com/users');
  
  // UI should display mocked users
  await expect(page.locator('.user-name')).toHaveCount(2);
  await expect(page.locator('.user-name').first()).toHaveText('Mocked User 1');
});

test('mock API with dynamic response', async ({ page }) => {
  await page.route('**/api/products?category=*', async (route) => {
    const url = route.request().url();
    const category = new URL(url).searchParams.get('category');
    
    const mockData = {
      electronics: [{ id: 1, name: 'Laptop' }, { id: 2, name: 'Phone' }],
      books: [{ id: 3, name: 'Novel' }, { id: 4, name: 'Textbook' }]
    };
    
    await route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify(mockData[category] || [])
    });
  });
  
  await page.goto('https://shop.example.com');
  await page.selectOption('#category', 'electronics');
  
  await expect(page.locator('.product-name')).toHaveCount(2);
});
```

### 4. Real-World Scenario: Testing Error Scenarios

```typescript
test('simulate network errors', async ({ page }) => {
  // Simulate failed API call
  await page.route('**/api/products', route => {
    route.abort('failed');
  });
  
  await page.goto('https://shop.example.com');
  
  // Should show error message
  await expect(page.locator('.error-message')).toBeVisible();
  await expect(page.locator('.error-message')).toContainText('Failed to load products');
});

test('simulate slow network', async ({ page }) => {
  await page.route('**/api/data', async (route) => {
    // Delay response by 5 seconds
    await new Promise(resolve => setTimeout(resolve, 5000));
    await route.continue();
  });
  
  await page.goto('https://example.com/dashboard');
  
  // Loading indicator should appear
  await expect(page.locator('.loading-spinner')).toBeVisible();
  
  // Eventually data loads
  await expect(page.locator('.data-loaded')).toBeVisible({ timeout: 10000 });
});

test('simulate 500 server error', async ({ page }) => {
  await page.route('**/api/submit', route => {
    route.fulfill({
      status: 500,
      contentType: 'application/json',
      body: JSON.stringify({
        error: 'Internal Server Error',
        message: 'Database connection failed'
      })
    });
  });
  
  await page.goto('https://example.com/form');
  await page.locator('#submit-btn').click();
  
  // Should show error handling
  await expect(page.locator('.error-notification')).toBeVisible();
  await expect(page.locator('.error-notification')).toContainText('Server Error');
});
```

### 5. Monitoring Network Traffic

```typescript
test('monitor all network requests', async ({ page }) => {
  const requests = [];
  const responses = [];
  
  page.on('request', request => {
    requests.push({
      url: request.url(),
      method: request.method(),
      headers: request.headers(),
      postData: request.postData()
    });
    console.log(`>> ${request.method()} ${request.url()}`);
  });
  
  page.on('response', response => {
    responses.push({
      url: response.url(),
      status: response.status(),
      headers: response.headers()
    });
    console.log(`<< ${response.status()} ${response.url()}`);
  });
  
  await page.goto('https://example.com');
  
  // Analyze collected data
  const apiRequests = requests.filter(r => r.url.includes('/api/'));
  console.log(`Total API requests: ${apiRequests.length}`);
  
  const failedResponses = responses.filter(r => r.status >= 400);
  expect(failedResponses).toHaveLength(0); // No failed requests
});

test('track specific API calls', async ({ page }) => {
  const apiCalls = [];
  
  await page.route('**/api/**', async (route) => {
    const request = route.request();
    const startTime = Date.now();
    
    await route.continue();
    
    apiCalls.push({
      url: request.url(),
      method: request.method(),
      responseTime: Date.now() - startTime
    });
  });
  
  await page.goto('https://dashboard.example.com');
  
  // Verify API performance
  apiCalls.forEach(call => {
    expect(call.responseTime).toBeLessThan(2000); // All APIs under 2 seconds
    console.log(`${call.method} ${call.url}: ${call.responseTime}ms`);
  });
});
```

### 6. Modify Request Data

```typescript
test('modify POST request body', async ({ page }) => {
  await page.route('**/api/users', async (route) => {
    if (route.request().method() === 'POST') {
      const originalData = JSON.parse(route.request().postData() || '{}');
      
      // Modify the data
      const modifiedData = {
        ...originalData,
        addedField: 'injected-value',
        timestamp: Date.now()
      };
      
      await route.continue({
        postData: JSON.stringify(modifiedData)
      });
    } else {
      await route.continue();
    }
  });
  
  await page.goto('https://example.com/signup');
  await page.locator('#name').fill('John Doe');
  await page.locator('#email').fill('john@example.com');
  await page.locator('#submit').click();
  
  // Server receives modified data
});

test('modify query parameters', async ({ page }) => {
  await page.route('**/api/search**', async (route) => {
    const url = new URL(route.request().url());
    
    // Add tracking parameter
    url.searchParams.set('source', 'test-automation');
    url.searchParams.set('userId', '12345');
    
    await route.continue({ url: url.toString() });
  });
  
  await page.goto('https://example.com');
  await page.locator('#search').fill('playwright');
  await page.locator('#search-btn').click();
});
```

### 7. Real-World Scenario: E-commerce Testing with Mocked APIs

```typescript
test('e-commerce flow with mocked payment API', async ({ page }) => {
  // Mock product API
  await page.route('**/api/products', route => {
    route.fulfill({
      status: 200,
      body: JSON.stringify({
        items: [
          { id: 'p1', name: 'Laptop Pro', price: 1299.99, stock: 10 },
          { id: 'p2', name: 'Mouse', price: 29.99, stock: 50 }
        ]
      })
    });
  });
  
  // Mock add to cart API
  await page.route('**/api/cart', route => {
    if (route.request().method() === 'POST') {
      route.fulfill({
        status: 201,
        body: JSON.stringify({
          cartId: 'cart-123',
          items: [{ productId: 'p1', quantity: 1 }],
          total: 1299.99
        })
      });
    } else {
      route.continue();
    }
  });
  
  // Mock payment API to always succeed
  await page.route('**/api/payment/process', route => {
    route.fulfill({
      status: 200,
      body: JSON.stringify({
        transactionId: 'txn-test-123',
        status: 'success',
        message: 'Payment processed successfully'
      })
    });
  });
  
  // Mock order creation
  await page.route('**/api/orders', route => {
    route.fulfill({
      status: 201,
      body: JSON.stringify({
        orderId: 'order-test-456',
        status: 'confirmed',
        total: 1299.99
      })
    });
  });
  
  // Execute test flow
  await page.goto('https://shop.example.com');
  
  // Add product
  await page.locator('[data-product-id="p1"] .add-to-cart').click();
  await expect(page.locator('.cart-badge')).toHaveText('1');
  
  // Go to checkout
  await page.locator('.cart-icon').click();
  await page.locator('#checkout-btn').click();
  
  // Fill payment info
  await page.locator('#card-number').fill('4242424242424242');
  await page.locator('#expiry').fill('12/25');
  await page.locator('#cvv').fill('123');
  await page.locator('#pay-btn').click();
  
  // Verify success
  await expect(page.locator('.success-message')).toBeVisible();
  await expect(page.locator('.order-id')).toContainText('order-test-456');
});
```

### 8. Conditional Request Handling

```typescript
test('handle requests based on conditions', async ({ page }) => {
  let requestCount = 0;
  
  await page.route('**/api/data', async (route) => {
    requestCount++;
    
    if (requestCount === 1) {
      // First request: return data
      await route.fulfill({
        status: 200,
        body: JSON.stringify({ items: ['item1', 'item2'] })
      });
    } else if (requestCount === 2) {
      // Second request: simulate error
      await route.fulfill({ status: 500 });
    } else {
      // Subsequent requests: use real API
      await route.continue();
    }
  });
  
  await page.goto('https://example.com/dashboard');
  // First load succeeds
  await page.locator('#refresh').click();
  // Second load fails (shows error)
  await page.locator('#refresh').click();
  // Third load uses real API
});

test('A/B testing with route interception', async ({ page }) => {
  const variant = 'B';
  
  await page.route('**/api/config', route => {
    route.fulfill({
      status: 200,
      body: JSON.stringify({
        variant: variant,
        features: {
          newCheckout: variant === 'B',
          recommendedProducts: true
        }
      })
    });
  });
  
  await page.goto('https://example.com');
  
  // Verify variant B features
  await expect(page.locator('.new-checkout-button')).toBeVisible();
});
```

### 9. Intercept and Log GraphQL Requests

```typescript
test('intercept GraphQL queries', async ({ page }) => {
  const graphqlRequests = [];
  
  await page.route('**/graphql', async (route) => {
    const postData = route.request().postData();
    const graphqlQuery = JSON.parse(postData || '{}');
    
    graphqlRequests.push({
      operationName: graphqlQuery.operationName,
      query: graphqlQuery.query,
      variables: graphqlQuery.variables
    });
    
    console.log(`GraphQL ${graphqlQuery.operationName}`);
    
    // Mock response
    if (graphqlQuery.operationName === 'GetUser') {
      await route.fulfill({
        status: 200,
        body: JSON.stringify({
          data: {
            user: {
              id: '1',
              name: 'Test User',
              email: 'test@example.com'
            }
          }
        })
      });
    } else {
      await route.continue();
    }
  });
  
  await page.goto('https://example.com/profile');
  
  // Verify GraphQL calls
  expect(graphqlRequests).toHaveLength(1);
  expect(graphqlRequests[0].operationName).toBe('GetUser');
});
```

### 10. Best Practices

✅ **GOOD: Use specific URL patterns**
```typescript
await page.route('**/api/users/**', route => route.fulfill({ body: '[]' }));
```

❌ **BAD: Intercept all requests**
```typescript
await page.route('**/*', route => {
  // Complex logic for every request - slow!
});
```

✅ **GOOD: Mock only necessary endpoints**
```typescript
await page.route('**/api/critical-data', route => route.fulfill({ ... }));
// Let other requests pass through
```

❌ **BAD: Mock everything unnecessarily**
```typescript
// Makes tests brittle and disconnected from reality
```

✅ **GOOD: Clean up routes when done**
```typescript
await page.unroute('**/api/users');
```

### 11. Common Pitfalls

1. **Not handling async properly**
   ```typescript
   // ❌ Doesn't wait for route
   page.route('**/api/data', route => route.fulfill({ body: '{}' }));
   await page.goto('/'); // May not be intercepted
   
   // ✅ Wait for route setup
   await page.route('**/api/data', route => route.fulfill({ body: '{}' }));
   await page.goto('/');
   ```

2. **Forgetting to continue unmatched routes**
   ```typescript
   // ❌ Blocks all requests
   await page.route('**/*', async (route) => {
     if (route.request().url().includes('api')) {
       await route.fulfill({ body: '{}' });
     }
     // Other requests hang!
   });
   
   // ✅ Continue others
   await page.route('**/*', async (route) => {
     if (route.request().url().includes('api')) {
       await route.fulfill({ body: '{}' });
     } else {
       await route.continue();
     }
   });
   ```

### Related Topics
- **Q13**: API testing
- **Q27**: Advanced mocking strategies
- **Q28**: Testing offline scenarios

---

*Document Version: 1.0*  
*Last Updated: November 2025*
