# Playwright Interview Questions: Q1-Q2

## Q1: What is Playwright and how does it differ from Selenium?

### Answer:

**Playwright** is a modern, open-source automation framework developed by Microsoft for end-to-end testing of web applications. It provides a single API to automate Chromium, Firefox, and WebKit browsers.

### Key Differences from Selenium:

| Feature | Playwright | Selenium |
|---------|-----------|----------|
| **Architecture** | Direct browser protocol | WebDriver protocol |
| **Auto-waiting** | Built-in smart waiting | Manual waits needed |
| **Browser Contexts** | Isolated contexts (fast) | Multiple browser instances (slow) |
| **Network Interception** | Native support | Requires proxies |
| **Speed** | Faster (2-3x) | Slower |
| **API** | Modern async/await | Mixed (sync/async) |
| **Screenshots** | Full page, element | Viewport only |
| **Mobile Emulation** | Built-in | Limited |

### Scenario: Login Test Comparison

**Selenium (WebDriverIO):**
```javascript
const { Builder, By, until } = require('selenium-webdriver');

async function loginTest() {
  const driver = await new Builder().forBrowser('chrome').build();
  
  try {
    await driver.get('https://example.com/login');
    
    // Manual wait needed
    await driver.wait(until.elementLocated(By.id('email')), 10000);
    
    await driver.findElement(By.id('email')).sendKeys('user@example.com');
    await driver.findElement(By.id('password')).sendKeys('password123');
    await driver.findElement(By.css('button[type="submit"]')).click();
    
    // Manual wait for navigation
    await driver.wait(until.urlContains('/dashboard'), 10000);
    
    const title = await driver.getTitle();
    console.log('Page title:', title);
  } finally {
    await driver.quit();
  }
}
```

**Playwright (Same Test):**
```typescript
import { test, expect } from '@playwright/test';

test('login test', async ({ page }) => {
  await page.goto('https://example.com/login');
  
  // Auto-waits for element to be visible and enabled
  await page.fill('#email', 'user@example.com');
  await page.fill('#password', 'password123');
  await page.click('button[type="submit"]');
  
  // Auto-waits for navigation
  await expect(page).toHaveURL(/.*dashboard/);
  
  const title = await page.title();
  console.log('Page title:', title);
});
```

### Why Playwright is Better:

✅ **Auto-waiting**: No need for explicit waits - Playwright waits for elements automatically  
✅ **Browser Contexts**: Run tests in parallel without starting new browsers  
✅ **Network Control**: Intercept and mock API requests natively  
✅ **Better Selectors**: CSS, XPath, text, role-based selectors  
✅ **Built-in Retry**: Automatic retry for flaky tests  
✅ **Trace Viewer**: Visual debugging with timeline  

### Real-World Scenario: E-commerce Testing

```typescript
import { test, expect } from '@playwright/test';

test.describe('E-commerce Product Purchase', () => {
  test('complete purchase flow with auto-waiting', async ({ page }) => {
    // Navigate to product page
    await page.goto('https://shop.example.com/products');
    
    // Search for product (auto-waits for input to be ready)
    await page.fill('[data-testid="search-input"]', 'laptop');
    await page.press('[data-testid="search-input"]', 'Enter');
    
    // Wait for search results (auto-waits)
    await expect(page.locator('.product-card')).toHaveCount(10);
    
    // Click first product (auto-waits for clickable)
    await page.click('.product-card:first-child');
    
    // Add to cart (auto-waits for navigation)
    await page.click('button:has-text("Add to Cart")');
    
    // Verify cart badge updated
    await expect(page.locator('.cart-badge')).toHaveText('1');
    
    // Proceed to checkout
    await page.click('a:has-text("Checkout")');
    
    // Fill shipping info (auto-waits for each field)
    await page.fill('#name', 'John Doe');
    await page.fill('#address', '123 Main St');
    await page.fill('#city', 'New York');
    await page.selectOption('#country', 'US');
    
    // Submit order
    await page.click('button:has-text("Place Order")');
    
    // Verify success message (auto-waits)
    await expect(page.locator('.success-message')).toBeVisible();
    await expect(page.locator('.order-number')).toContainText('ORDER-');
  });
});
```

### Common Pitfalls:

❌ **Don't**: Use `setTimeout()` for waiting  
✅ **Do**: Use Playwright's auto-waiting and assertions

❌ **Don't**: `await page.waitForTimeout(5000)`  
✅ **Do**: `await expect(page.locator('.element')).toBeVisible()`

---

## Q2: How do you install and set up Playwright in a Node.js project?

### Answer:

Playwright installation is straightforward and includes test runners, browsers, and dependencies.

### Installation Steps:

#### 1. Create New Project

```bash
# Create project directory
mkdir playwright-testing
cd playwright-testing

# Initialize npm project
npm init -y
```

#### 2. Install Playwright

```bash
# Install Playwright with browsers
npm init playwright@latest

# Or manually install
npm install -D @playwright/test
npx playwright install
```

#### 3. Installation Options

**Interactive Setup:**
```bash
npm init playwright@latest

# Prompts:
# ✔ Do you want to use TypeScript or JavaScript? · TypeScript
# ✔ Where to put your end-to-end tests? · tests
# ✔ Add a GitHub Actions workflow? · true
# ✔ Install Playwright browsers? · true
```

**Manual Installation:**
```bash
# Install test runner
npm install -D @playwright/test

# Install browsers (Chromium, Firefox, WebKit)
npx playwright install

# Install only specific browser
npx playwright install chromium

# Install with dependencies (for CI/CD)
npx playwright install --with-deps
```

### Project Structure:

```
playwright-testing/
├── tests/
│   ├── example.spec.ts
│   └── auth.spec.ts
├── playwright.config.ts
├── package.json
└── .gitignore
```

### Configuration File: `playwright.config.ts`

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  // Test directory
  testDir: './tests',
  
  // Maximum time one test can run
  timeout: 30 * 1000,
  
  // Run tests in parallel
  fullyParallel: true,
  
  // Fail the build on CI if you accidentally left test.only
  forbidOnly: !!process.env.CI,
  
  // Retry on CI only
  retries: process.env.CI ? 2 : 0,
  
  // Number of parallel workers
  workers: process.env.CI ? 1 : undefined,
  
  // Reporter to use
  reporter: [
    ['html'],
    ['json', { outputFile: 'test-results.json' }],
    ['junit', { outputFile: 'test-results.xml' }]
  ],
  
  // Shared settings for all projects
  use: {
    // Base URL
    baseURL: 'http://localhost:3000',
    
    // Browser options
    headless: true,
    viewport: { width: 1280, height: 720 },
    
    // Collect trace when retrying failed tests
    trace: 'on-first-retry',
    
    // Screenshot on failure
    screenshot: 'only-on-failure',
    
    // Video on failure
    video: 'retain-on-failure',
  },
  
  // Configure projects for multiple browsers
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
    
    // Mobile viewports
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] },
    },
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 12'] },
    },
  ],
  
  // Run local dev server before tests
  webServer: {
    command: 'npm run start',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

### First Test: `tests/example.spec.ts`

```typescript
import { test, expect } from '@playwright/test';

test('basic test example', async ({ page }) => {
  // Navigate to page
  await page.goto('https://playwright.dev/');
  
  // Verify title
  await expect(page).toHaveTitle(/Playwright/);
  
  // Click Get Started button
  await page.click('text=Get started');
  
  // Verify navigation
  await expect(page).toHaveURL(/.*intro/);
});

test('search functionality', async ({ page }) => {
  await page.goto('https://playwright.dev/');
  
  // Click search button
  await page.click('[aria-label="Search"]');
  
  // Type in search box
  await page.fill('#docsearch-input', 'locators');
  
  // Wait for results
  await page.waitForSelector('.DocSearch-Hit');
  
  // Verify results
  const results = page.locator('.DocSearch-Hit');
  await expect(results).toHaveCount(5);
});
```

### Package.json Scripts:

```json
{
  "name": "playwright-testing",
  "version": "1.0.0",
  "scripts": {
    "test": "playwright test",
    "test:headed": "playwright test --headed",
    "test:debug": "playwright test --debug",
    "test:chromium": "playwright test --project=chromium",
    "test:firefox": "playwright test --project=firefox",
    "test:webkit": "playwright test --project=webkit",
    "test:mobile": "playwright test --project='Mobile Chrome'",
    "report": "playwright show-report",
    "codegen": "playwright codegen"
  },
  "devDependencies": {
    "@playwright/test": "^1.40.0"
  }
}
```

### Running Tests:

```bash
# Run all tests
npm test

# Run specific test file
npx playwright test tests/example.spec.ts

# Run tests in headed mode (see browser)
npm run test:headed

# Run tests in debug mode
npm run test:debug

# Run specific browser
npm run test:chromium

# Run with UI mode
npx playwright test --ui

# Generate HTML report
npm run report
```

### Real-World Scenario: Testing SaaS Application

```typescript
// tests/saas-app.spec.ts
import { test, expect } from '@playwright/test';

test.describe('SaaS Application Tests', () => {
  test.beforeEach(async ({ page }) => {
    // Navigate to login page before each test
    await page.goto('https://app.example.com/login');
  });
  
  test('user can sign up', async ({ page }) => {
    await page.click('text=Sign Up');
    
    await page.fill('#email', 'newuser@example.com');
    await page.fill('#password', 'SecurePass123!');
    await page.fill('#confirmPassword', 'SecurePass123!');
    
    await page.check('#terms');
    await page.click('button:has-text("Create Account")');
    
    // Verify welcome message
    await expect(page.locator('.welcome-message')).toBeVisible();
    await expect(page).toHaveURL(/.*dashboard/);
  });
  
  test('user can login', async ({ page }) => {
    await page.fill('#email', 'testuser@example.com');
    await page.fill('#password', 'password123');
    await page.click('button[type="submit"]');
    
    // Wait for dashboard
    await expect(page).toHaveURL(/.*dashboard/);
    
    // Verify user menu
    await expect(page.locator('.user-menu')).toBeVisible();
  });
  
  test('invalid login shows error', async ({ page }) => {
    await page.fill('#email', 'wrong@example.com');
    await page.fill('#password', 'wrongpassword');
    await page.click('button[type="submit"]');
    
    // Verify error message
    await expect(page.locator('.error-message')).toHaveText('Invalid credentials');
    
    // Should stay on login page
    await expect(page).toHaveURL(/.*login/);
  });
});
```

### Environment-Specific Configuration:

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

const isCI = !!process.env.CI;
const baseURL = process.env.BASE_URL || 'http://localhost:3000';

export default defineConfig({
  testDir: './tests',
  timeout: 30000,
  retries: isCI ? 2 : 0,
  workers: isCI ? 2 : 4,
  
  use: {
    baseURL,
    trace: isCI ? 'on-first-retry' : 'on',
    screenshot: 'only-on-failure',
    video: isCI ? 'retain-on-failure' : 'off',
  },
  
  projects: [
    { name: 'chromium', use: { channel: 'chrome' } },
  ],
  
  // Different config for staging vs production
  ...(process.env.ENV === 'staging' && {
    use: {
      baseURL: 'https://staging.example.com',
      ignoreHTTPSErrors: true,
    },
  }),
});
```

### Docker Setup:

```dockerfile
# Dockerfile
FROM mcr.microsoft.com/playwright:v1.40.0-focal

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

CMD ["npm", "test"]
```

```bash
# Build and run in Docker
docker build -t playwright-tests .
docker run -it playwright-tests
```

### Best Practices:

✅ **Use TypeScript** for better type safety and IDE support  
✅ **Configure retries** for flaky tests in CI  
✅ **Enable trace** on first retry for debugging  
✅ **Run tests in parallel** for faster execution  
✅ **Use baseURL** to avoid repeating full URLs  
✅ **Configure different environments** (dev, staging, prod)  

### Common Issues:

❌ **Browser not installed**: Run `npx playwright install`  
❌ **Port conflict**: Change `webServer.url` in config  
❌ **Timeout errors**: Increase `timeout` in config  
❌ **CI failures**: Use `--with-deps` flag for system dependencies  

---

**Related Topics:**
- Q3-Q4: Browser contexts and page navigation
- Q5-Q6: Selectors and locators
- Q35-Q36: CI/CD integration

---

*Document Version: 1.0*  
*Last Updated: November 2025*
