# Testing Strategies

## Question 31: Comprehensive Testing Implementation

### 📋 Question Statement

Implement a complete testing strategy for Emirates NBD including unit tests, integration tests, E2E tests, contract testing, and chaos engineering. Include Jest, Supertest, and AWS testing tools.

---

### 🏗️ Testing Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│              EMIRATES NBD TESTING STRATEGY                                  │
└────────────────────────────────────────────────────────────────────────────┘

    TEST PYRAMID                    
    ────────────                    
    
         ┌─────────┐
         │   E2E   │  ← Few, slow, expensive
         └─────────┘
       ┌─────────────┐
       │ Integration │  ← Some, medium speed
       └─────────────┘
     ┌─────────────────┐
     │   Unit Tests    │  ← Many, fast, cheap
     └─────────────────┘
```

---

### 🧪 Unit Tests (Jest)

```javascript
// tests/unit/account-service.test.js
const AccountService = require('../../src/services/account-service');
const AccountRepository = require('../../src/repositories/account-repository');

jest.mock('../../src/repositories/account-repository');

describe('AccountService', () => {
  let accountService;
  let mockRepository;

  beforeEach(() => {
    mockRepository = {
      getAccount: jest.fn(),
      createAccount: jest.fn(),
      updateBalance: jest.fn()
    };
    AccountRepository.mockImplementation(() => mockRepository);
    accountService = new AccountService();
  });

  describe('createAccount', () => {
    it('should create a new account successfully', async () => {
      const mockAccount = {
        accountId: 'ACC-001',
        customerId: 'CUST-001',
        accountType: 'SAVINGS',
        balance: 1000
      };

      mockRepository.createAccount.mockResolvedValue(mockAccount);

      const result = await accountService.createAccount({
        customerId: 'CUST-001',
        accountType: 'SAVINGS',
        initialBalance: 1000
      });

      expect(result).toEqual(mockAccount);
      expect(mockRepository.createAccount).toHaveBeenCalledTimes(1);
    });

    it('should throw error for invalid account type', async () => {
      await expect(
        accountService.createAccount({
          customerId: 'CUST-001',
          accountType: 'INVALID',
          initialBalance: 1000
        })
      ).rejects.toThrow('Invalid account type');
    });

    it('should throw error for negative initial balance', async () => {
      await expect(
        accountService.createAccount({
          customerId: 'CUST-001',
          accountType: 'SAVINGS',
          initialBalance: -100
        })
      ).rejects.toThrow('Initial balance cannot be negative');
    });
  });

  describe('deposit', () => {
    it('should successfully deposit money', async () => {
      const mockAccount = {
        accountId: 'ACC-001',
        balance: 1000
      };

      mockRepository.getAccount.mockResolvedValue(mockAccount);
      mockRepository.updateBalance.mockResolvedValue({
        ...mockAccount,
        balance: 1500
      });

      const result = await accountService.deposit('ACC-001', 500);

      expect(result.balance).toBe(1500);
      expect(mockRepository.updateBalance).toHaveBeenCalledWith(
        'ACC-001',
        500,
        'CREDIT'
      );
    });

    it('should throw error for non-existent account', async () => {
      mockRepository.getAccount.mockResolvedValue(null);

      await expect(
        accountService.deposit('ACC-999', 500)
      ).rejects.toThrow('Account not found');
    });
  });

  describe('withdraw', () => {
    it('should successfully withdraw money', async () => {
      const mockAccount = {
        accountId: 'ACC-001',
        balance: 1000
      };

      mockRepository.getAccount.mockResolvedValue(mockAccount);
      mockRepository.updateBalance.mockResolvedValue({
        ...mockAccount,
        balance: 700
      });

      const result = await accountService.withdraw('ACC-001', 300);

      expect(result.balance).toBe(700);
    });

    it('should throw error for insufficient funds', async () => {
      const mockAccount = {
        accountId: 'ACC-001',
        balance: 100
      };

      mockRepository.getAccount.mockResolvedValue(mockAccount);

      await expect(
        accountService.withdraw('ACC-001', 500)
      ).rejects.toThrow('Insufficient funds');
    });
  });
});
```

### 🔗 Integration Tests (Supertest)

```javascript
// tests/integration/account-api.test.js
const request = require('supertest');
const app = require('../../src/app');
const db = require('../../src/database/connection-pool');

describe('Account API Integration Tests', () => {
  let createdAccountId;

  beforeAll(async () => {
    // Setup test database
    await db.query('BEGIN');
  });

  afterAll(async () => {
    // Rollback test data
    await db.query('ROLLBACK');
    await db.close();
  });

  describe('POST /accounts', () => {
    it('should create a new account', async () => {
      const response = await request(app)
        .post('/accounts')
        .send({
          customerId: 'CUST-TEST-001',
          accountType: 'SAVINGS',
          initialBalance: 1000
        })
        .expect(201);

      expect(response.body).toHaveProperty('accountId');
      expect(response.body.balance).toBe(1000);
      
      createdAccountId = response.body.accountId;
    });

    it('should return 400 for invalid data', async () => {
      await request(app)
        .post('/accounts')
        .send({
          customerId: 'CUST-TEST-001',
          accountType: 'INVALID'
        })
        .expect(400);
    });
  });

  describe('GET /accounts/:accountId', () => {
    it('should retrieve account details', async () => {
      const response = await request(app)
        .get(`/accounts/${createdAccountId}`)
        .expect(200);

      expect(response.body.accountId).toBe(createdAccountId);
      expect(response.body.balance).toBe(1000);
    });

    it('should return 404 for non-existent account', async () => {
      await request(app)
        .get('/accounts/ACC-NONEXISTENT')
        .expect(404);
    });
  });

  describe('POST /transactions', () => {
    it('should create a transaction', async () => {
      // Create second account
      const account2 = await request(app)
        .post('/accounts')
        .send({
          customerId: 'CUST-TEST-002',
          accountType: 'CURRENT',
          initialBalance: 500
        });

      // Transfer money
      const response = await request(app)
        .post('/transactions')
        .send({
          fromAccount: createdAccountId,
          toAccount: account2.body.accountId,
          amount: 200,
          type: 'TRANSFER'
        })
        .expect(201);

      expect(response.body.status).toBe('COMPLETED');

      // Verify balances
      const from = await request(app).get(`/accounts/${createdAccountId}`);
      const to = await request(app).get(`/accounts/${account2.body.accountId}`);

      expect(from.body.balance).toBe(800);
      expect(to.body.balance).toBe(700);
    });
  });
});
```

### 🎓 Interview Discussion Points - Q31

**Q1: What is the testing pyramid?**

**A**:
- **Base**: Many unit tests (70%)
- **Middle**: Some integration tests (20%)
- **Top**: Few E2E tests (10%)

**Q2: What is contract testing?**

**A**:
- Tests API contracts between services
- Uses Pact or similar tools
- Consumer-driven contracts
- Prevents breaking changes

---

## Question 32: E2E Testing with Playwright

### 📋 Question Statement

Implement end-to-end testing for Emirates NBD banking application using Playwright for browser automation, visual regression testing, and user journey validation with CI/CD integration.

---

### 🎭 Comprehensive Playwright Test Suite

```typescript
// tests/e2e/banking-flows.spec.ts
import { test, expect, Page } from '@playwright/test';

// Test configuration
test.use({
  baseURL: process.env.BASE_URL || 'https://banking.emiratesnbd.com',
  screenshot: 'only-on-failure',
  video: 'retain-on-failure',
  trace: 'retain-on-failure'
});

// Page Object Models
class LoginPage {
  constructor(private page: Page) {}

  async navigate() {
    await this.page.goto('/login');
  }

  async login(username: string, password: string) {
    await this.page.fill('[data-testid="username"]', username);
    await this.page.fill('[data-testid="password"]', password);
    await this.page.click('[data-testid="login-button"]');
    
    // Wait for navigation
    await this.page.waitForURL('**/dashboard');
  }

  async loginWithOTP(username: string, password: string, otp: string) {
    await this.login(username, password);
    await this.page.fill('[data-testid="otp-input"]', otp);
    await this.page.click('[data-testid="verify-otp"]');
  }
}

class DashboardPage {
  constructor(private page: Page) {}

  async getAccountBalance(accountNumber: string): Promise<number> {
    const balanceText = await this.page
      .locator(`[data-account="${accountNumber}"] .balance`)
      .textContent();
    
    return parseFloat(balanceText?.replace(/[^0-9.-]+/g, '') || '0');
  }

  async navigateToTransfer() {
    await this.page.click('[data-testid="transfer-link"]');
    await this.page.waitForURL('**/transfer');
  }

  async navigateToAccounts() {
    await this.page.click('[data-testid="accounts-link"]');
    await this.page.waitForURL('**/accounts');
  }
}

class TransferPage {
  constructor(private page: Page) {}

  async initiateTransfer(params: {
    fromAccount: string;
    toAccount: string;
    amount: number;
    description?: string;
  }) {
    await this.page.selectOption('[data-testid="from-account"]', params.fromAccount);
    await this.page.fill('[data-testid="to-account"]', params.toAccount);
    await this.page.fill('[data-testid="amount"]', params.amount.toString());
    
    if (params.description) {
      await this.page.fill('[data-testid="description"]', params.description);
    }

    await this.page.click('[data-testid="submit-transfer"]');
  }

  async confirmTransfer() {
    await this.page.click('[data-testid="confirm-button"]');
    
    // Wait for success message
    await expect(this.page.locator('.success-message')).toBeVisible();
  }

  async getTransferConfirmation(): Promise<string> {
    const confirmationText = await this.page
      .locator('[data-testid="confirmation-id"]')
      .textContent();
    
    return confirmationText || '';
  }
}

// ============================================
// TEST SUITE: Complete Banking Flows
// ============================================

test.describe('Emirates NBD Banking - Critical User Journeys', () => {
  let loginPage: LoginPage;
  let dashboardPage: DashboardPage;
  let transferPage: TransferPage;

  test.beforeEach(async ({ page }) => {
    loginPage = new LoginPage(page);
    dashboardPage = new DashboardPage(page);
    transferPage = new TransferPage(page);
    
    // Login before each test
    await loginPage.navigate();
    await loginPage.login('testuser@emiratesnbd.com', 'SecurePass123!');
  });

  test('should complete successful money transfer', async ({ page }) => {
    // Navigate to transfer page
    await dashboardPage.navigateToTransfer();

    // Get initial balances
    const initialBalance = await dashboardPage.getAccountBalance('ACC-001');

    // Initiate transfer
    await transferPage.initiateTransfer({
      fromAccount: 'ACC-001',
      toAccount: 'ACC-002',
      amount: 100,
      description: 'Test transfer'
    });

    // Review and confirm
    await expect(page.locator('.transfer-review')).toContainText('AED 100.00');
    await transferPage.confirmTransfer();

    // Verify confirmation
    const confirmationId = await transferPage.getTransferConfirmation();
    expect(confirmationId).toMatch(/TXN-\d{10}/);

    // Verify balance update
    await dashboardPage.navigateToAccounts();
    const newBalance = await dashboardPage.getAccountBalance('ACC-001');
    expect(newBalance).toBe(initialBalance - 100);
  });

  test('should prevent transfer with insufficient funds', async ({ page }) => {
    await dashboardPage.navigateToTransfer();

    // Try to transfer more than balance
    await transferPage.initiateTransfer({
      fromAccount: 'ACC-001',
      toAccount: 'ACC-002',
      amount: 999999
    });

    // Should show error
    await expect(page.locator('.error-message')).toBeVisible();
    await expect(page.locator('.error-message'))
      .toContainText('Insufficient funds');
  });

  test('should handle concurrent transactions', async ({ page, context }) => {
    // Open second tab
    const page2 = await context.newPage();
    const dashboard2 = new DashboardPage(page2);
    const transfer2 = new TransferPage(page2);

    await page2.goto('/transfer');

    // Initiate transfers simultaneously
    await Promise.all([
      transferPage.initiateTransfer({
        fromAccount: 'ACC-001',
        toAccount: 'ACC-002',
        amount: 50
      }),
      transfer2.initiateTransfer({
        fromAccount: 'ACC-001',
        toAccount: 'ACC-003',
        amount: 50
      })
    ]);

    // Both should complete or one should fail
    const results = await Promise.allSettled([
      transferPage.confirmTransfer(),
      transfer2.confirmTransfer()
    ]);

    const successful = results.filter(r => r.status === 'fulfilled').length;
    expect(successful).toBeGreaterThan(0);
  });
});

test.describe('Account Management', () => {
  test('should create new savings account', async ({ page }) => {
    await page.goto('/accounts/new');

    await page.selectOption('[data-testid="account-type"]', 'SAVINGS');
    await page.fill('[data-testid="initial-deposit"]', '5000');
    await page.fill('[data-testid="account-name"]', 'Emergency Fund');
    
    await page.click('[data-testid="create-account"]');

    // Verify account created
    await expect(page.locator('.success-notification')).toBeVisible();
    await expect(page).toHaveURL(/.*accounts\/ACC-\d{6}/);

    // Verify account appears in dashboard
    await page.goto('/dashboard');
    await expect(page.locator('[data-testid="account-list"]'))
      .toContainText('Emergency Fund');
  });

  test('should update account preferences', async ({ page }) => {
    await page.goto('/accounts/ACC-001/settings');

    await page.check('[data-testid="email-notifications"]');
    await page.check('[data-testid="sms-alerts"]');
    await page.selectOption('[data-testid="statement-frequency"]', 'MONTHLY');

    await page.click('[data-testid="save-preferences"]');

    // Verify saved
    await expect(page.locator('.success-message')).toBeVisible();
    
    // Reload and verify persistence
    await page.reload();
    await expect(page.locator('[data-testid="email-notifications"]')).toBeChecked();
  });
});

// ============================================
// VISUAL REGRESSION TESTING
// ============================================

test.describe('Visual Regression Tests', () => {
  test('dashboard should match baseline', async ({ page }) => {
    await page.goto('/dashboard');
    
    // Wait for all content to load
    await page.waitForLoadState('networkidle');
    
    // Take screenshot and compare
    await expect(page).toHaveScreenshot('dashboard.png', {
      maxDiffPixels: 100
    });
  });

  test('transfer form should match baseline', async ({ page }) => {
    await page.goto('/transfer');
    
    await expect(page).toHaveScreenshot('transfer-form.png', {
      fullPage: true
    });
  });

  test('mobile responsive layout', async ({ page }) => {
    await page.setViewportSize({ width: 375, height: 667 }); // iPhone SE
    await page.goto('/dashboard');
    
    await expect(page).toHaveScreenshot('mobile-dashboard.png');
  });
});

// ============================================
// PERFORMANCE TESTING
// ============================================

test.describe('Performance Tests', () => {
  test('dashboard should load within 2 seconds', async ({ page }) => {
    const startTime = Date.now();
    
    await page.goto('/dashboard');
    await page.waitForLoadState('networkidle');
    
    const loadTime = Date.now() - startTime;
    expect(loadTime).toBeLessThan(2000);
  });

  test('transfer should complete within 3 seconds', async ({ page }) => {
    await page.goto('/transfer');
    
    const startTime = Date.now();
    
    await page.selectOption('[data-testid="from-account"]', 'ACC-001');
    await page.fill('[data-testid="to-account"]', 'ACC-002');
    await page.fill('[data-testid="amount"]', '100');
    await page.click('[data-testid="submit-transfer"]');
    
    await expect(page.locator('.success-message')).toBeVisible();
    
    const transferTime = Date.now() - startTime;
    expect(transferTime).toBeLessThan(3000);
  });
});

// ============================================
// ACCESSIBILITY TESTING
// ============================================

test.describe('Accessibility Tests', () => {
  test('should have no accessibility violations', async ({ page }) => {
    await page.goto('/dashboard');
    
    // Inject axe-core
    await page.addScriptTag({
      url: 'https://cdnjs.cloudflare.com/ajax/libs/axe-core/4.7.2/axe.min.js'
    });

    // Run accessibility scan
    const violations = await page.evaluate(() => {
      return (window as any).axe.run();
    });

    expect(violations.violations).toHaveLength(0);
  });

  test('should be keyboard navigable', async ({ page }) => {
    await page.goto('/dashboard');
    
    // Tab through elements
    await page.keyboard.press('Tab');
    await expect(page.locator(':focus')).toBeVisible();
    
    await page.keyboard.press('Tab');
    await expect(page.locator(':focus')).toBeVisible();
  });
});
```

### 🔧 Playwright Configuration

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  
  reporter: [
    ['html'],
    ['junit', { outputFile: 'test-results/junit.xml' }],
    ['json', { outputFile: 'test-results/results.json' }]
  ],

  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    
    // Authentication state
    storageState: 'tests/.auth/user.json'
  },

  projects: [
    // Setup authentication
    {
      name: 'setup',
      testMatch: /.*\.setup\.ts/
    },
    
    // Desktop browsers
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
      dependencies: ['setup']
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
      dependencies: ['setup']
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
      dependencies: ['setup']
    },

    // Mobile browsers
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] },
      dependencies: ['setup']
    },
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 12'] },
      dependencies: ['setup']
    }
  ],

  webServer: {
    command: 'npm run start',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 120 * 1000
  }
});
```

### 🔐 Authentication Setup

```typescript
// tests/auth.setup.ts
import { test as setup } from '@playwright/test';

const authFile = 'tests/.auth/user.json';

setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  
  await page.fill('[data-testid="username"]', process.env.TEST_USER!);
  await page.fill('[data-testid="password"]', process.env.TEST_PASSWORD!);
  await page.click('[data-testid="login-button"]');

  // Wait for successful login
  await page.waitForURL('**/dashboard');

  // Save authenticated state
  await page.context().storageState({ path: authFile });
});
```

### 🚀 CI/CD Integration

```yaml
# .github/workflows/e2e-tests.yml
name: E2E Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Install Playwright browsers
        run: npx playwright install --with-deps
        
      - name: Run E2E tests
        run: npm run test:e2e
        env:
          BASE_URL: ${{ secrets.STAGING_URL }}
          TEST_USER: ${{ secrets.TEST_USER }}
          TEST_PASSWORD: ${{ secrets.TEST_PASSWORD }}
          
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
          
      - name: Upload screenshots
        if: failure()
        uses: actions/upload-artifact@v3
        with:
          name: screenshots
          path: test-results/
```

### 📊 Test Reporting

```typescript
// tests/utils/custom-reporter.ts
import { Reporter, TestCase, TestResult } from '@playwright/test/reporter';

class CustomReporter implements Reporter {
  onTestEnd(test: TestCase, result: TestResult) {
    const duration = result.duration;
    const status = result.status;
    
    console.log(`${test.title}: ${status} (${duration}ms)`);
    
    // Send to monitoring system
    if (process.env.CI) {
      this.sendMetrics({
        testName: test.title,
        status,
        duration,
        timestamp: Date.now()
      });
    }
  }

  private async sendMetrics(metrics: any) {
    // Send to CloudWatch or DataDog
    await fetch(process.env.METRICS_ENDPOINT!, {
      method: 'POST',
      body: JSON.stringify(metrics)
    });
  }
}

export default CustomReporter;
```

### 🎓 Interview Discussion Points - Q32

**Q1: When should you use E2E tests vs unit/integration tests?**

**A**:
- **E2E Tests**: Critical user journeys (login, payment, transfer)
- **Integration Tests**: API contracts, service interactions
- **Unit Tests**: Business logic, utilities, validators
- **Ratio**: 70% unit, 20% integration, 10% E2E (test pyramid)

**Q2: How do you handle flaky E2E tests?**

**A**:
- **Retries**: Playwright auto-retry (2-3 times in CI)
- **Wait strategies**: `waitForLoadState`, `waitForSelector`
- **Stable selectors**: Use `data-testid` attributes, not CSS classes
- **Isolation**: Each test independent, clean state
- **Timeouts**: Increase for slow environments

**Q3: How do you test authentication flows?**

**A**:
- **Setup script**: Login once, save auth state
- **Reuse state**: Load saved cookies/tokens for all tests
- **Separate project**: Authentication tests run first
- **Mock MFA**: Use test OTP codes in non-prod

**Q4: What is visual regression testing?**

**A**:
- **Screenshots**: Capture baseline images
- **Compare**: Detect pixel differences on changes
- **Threshold**: Allow minor differences (e.g., 100 pixels)
- **Use cases**: UI consistency, CSS changes, responsive design

**Q5: How do you optimize E2E test execution time?**

**A**:
- **Parallel execution**: Run tests concurrently (Playwright default)
- **Sharding**: Distribute tests across multiple CI workers
- **Smart retries**: Only retry failed tests
- **Reuse browser context**: Share authentication state
- **Selective running**: Only run affected tests on PR
- **Typical**: 50 tests in 5 minutes with 10 workers

---

**End of File 16**