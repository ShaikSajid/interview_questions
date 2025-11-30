# Playwright Interview Questions: Q33-Q34

## Q33: How do you use fixtures in Playwright?

### Answer:

Fixtures provide a way to set up test environment and share objects between tests.

### 1. Built-in Fixtures

```typescript
import { test } from '@playwright/test';

test('use built-in fixtures', async ({ page, context, browser }) => {
  // page - isolated page
  // context - browser context
  // browser - browser instance
  await page.goto('https://example.com');
});
```

### 2. Custom Fixtures

```typescript
// fixtures/customFixtures.ts
import { test as base } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';

type MyFixtures = {
  loginPage: LoginPage;
  authenticatedPage: Page;
};

export const test = base.extend<MyFixtures>({
  loginPage: async ({ page }, use) => {
    const loginPage = new LoginPage(page);
    await use(loginPage);
  },
  
  authenticatedPage: async ({ page }, use) => {
    // Auto-login for tests
    await page.goto('https://example.com/login');
    await page.locator('#email').fill('user@example.com');
    await page.locator('#password').fill('password');
    await page.locator('#login').click();
    await page.waitForURL('**/dashboard');
    await use(page);
  }
});

// Usage
import { test, expect } from './fixtures/customFixtures';

test('use custom fixture', async ({ authenticatedPage }) => {
  // Already logged in!
  await expect(authenticatedPage.locator('.user-name')).toBeVisible();
});
```

### 3. Fixture with API Setup

```typescript
export const test = base.extend({
  testUser: async ({ request }, use) => {
    // Create test user via API
    const response = await request.post('/api/users', {
      data: {
        email: `test${Date.now()}@example.com`,
        password: 'Test123!'
      }
    });
    const user = await response.json();
    
    await use(user);
    
    // Cleanup
    await request.delete(`/api/users/${user.id}`);
  }
});
```

### Related Topics
- **Q34**: Test hooks
- **Q31**: Page objects

---

## Q34: How do you implement test hooks and setup?

### Answer:

Test hooks allow setup and teardown logic before/after tests.

### 1. Test Hooks

```typescript
import { test } from '@playwright/test';

test.beforeAll(async ({ browser }) => {
  // Runs once before all tests in file
  console.log('Setup for all tests');
});

test.beforeEach(async ({ page }) => {
  // Runs before each test
  await page.goto('https://example.com');
});

test.afterEach(async ({ page }, testInfo) => {
  // Runs after each test
  if (testInfo.status === 'failed') {
    await page.screenshot({ path: `failures/${testInfo.title}.png` });
  }
});

test.afterAll(async () => {
  // Runs once after all tests
  console.log('Cleanup');
});

test('my test', async ({ page }) => {
  // Test runs with hooks
});
```

### 2. Describe-Level Hooks

```typescript
test.describe('User Management', () => {
  let adminToken: string;
  
  test.beforeAll(async ({ request }) => {
    const response = await request.post('/api/auth/login', {
      data: { email: 'admin@example.com', password: 'admin' }
    });
    const data = await response.json();
    adminToken = data.token;
  });
  
  test.beforeEach(async ({ page }) => {
    // Set auth for each test
    await page.addInitScript(token => {
      localStorage.setItem('authToken', token);
    }, adminToken);
  });
  
  test('create user', async ({ page }) => {});
  test('delete user', async ({ page }) => {});
});
```

### Related Topics
- **Q33**: Fixtures
- **Q15**: Authentication

---

*Document Version: 1.0*  
*Last Updated: November 2025*
