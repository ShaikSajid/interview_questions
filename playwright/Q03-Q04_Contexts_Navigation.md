# Playwright Interview Questions: Q3-Q4

## Q3: What are Browser Contexts in Playwright and why are they useful?

### Answer:

**Browser Context** is an isolated incognito-like session within a browser instance. Each context has its own cookies, local storage, and cache - completely isolated from other contexts.

### Key Concepts:

```
Browser Instance
├── Browser Context 1 (Isolated session)
│   ├── Page 1
│   ├── Page 2
│   └── Page 3
├── Browser Context 2 (Different session)
│   ├── Page 1
│   └── Page 2
└── Browser Context 3 (Another session)
    └── Page 1
```

### Why Browser Contexts are Powerful:

| Feature | Benefit |
|---------|---------|
| **Isolation** | Each context has separate cookies/storage |
| **Performance** | No need to restart browser |
| **Parallel Testing** | Run multiple tests simultaneously |
| **Multi-user Testing** | Test different users in parallel |
| **Fast Cleanup** | Just close context, not browser |

### Basic Usage:

```typescript
import { chromium } from '@playwright/test';

async function browserContextExample() {
  // Launch browser once
  const browser = await chromium.launch();
  
  // Create multiple isolated contexts
  const context1 = await browser.newContext();
  const context2 = await browser.newContext();
  
  // Each context has independent pages
  const page1 = await context1.newPage();
  const page2 = await context2.newPage();
  
  // Context 1: Login as User A
  await page1.goto('https://example.com/login');
  await page1.fill('#email', 'userA@example.com');
  await page1.fill('#password', 'passwordA');
  await page1.click('button[type="submit"]');
  
  // Context 2: Login as User B (simultaneously!)
  await page2.goto('https://example.com/login');
  await page2.fill('#email', 'userB@example.com');
  await page2.fill('#password', 'passwordB');
  await page2.click('button[type="submit"]');
  
  // Verify both users are logged in independently
  await page1.waitForURL('**/dashboard');
  await page2.waitForURL('**/dashboard');
  
  // Cleanup
  await context1.close();
  await context2.close();
  await browser.close();
}
```

### Real-World Scenario: Multi-User Chat Application

```typescript
import { test, expect, chromium } from '@playwright/test';

test('multiple users can chat simultaneously', async () => {
  const browser = await chromium.launch();
  
  // Create contexts for 3 different users
  const userAContext = await browser.newContext();
  const userBContext = await browser.newContext();
  const userCContext = await browser.newContext();
  
  // Create pages for each user
  const userAPage = await userAContext.newPage();
  const userBPage = await userBContext.newPage();
  const userCPage = await userCContext.newPage();
  
  // User A logs in
  await userAPage.goto('https://chat.example.com/login');
  await userAPage.fill('#username', 'Alice');
  await userAPage.click('button:has-text("Join Chat")');
  
  // User B logs in
  await userBPage.goto('https://chat.example.com/login');
  await userBPage.fill('#username', 'Bob');
  await userBPage.click('button:has-text("Join Chat")');
  
  // User C logs in
  await userCPage.goto('https://chat.example.com/login');
  await userCPage.fill('#username', 'Charlie');
  await userCPage.click('button:has-text("Join Chat")');
  
  // User A sends a message
  await userAPage.fill('#message-input', 'Hello everyone!');
  await userAPage.click('button:has-text("Send")');
  
  // Verify User B sees the message
  await expect(userBPage.locator('.message').last()).toContainText('Alice: Hello everyone!');
  
  // Verify User C sees the message
  await expect(userCPage.locator('.message').last()).toContainText('Alice: Hello everyone!');
  
  // User B replies
  await userBPage.fill('#message-input', 'Hi Alice!');
  await userBPage.click('button:has-text("Send")');
  
  // Verify all users see Bob's reply
  await expect(userAPage.locator('.message').last()).toContainText('Bob: Hi Alice!');
  await expect(userCPage.locator('.message').last()).toContainText('Bob: Hi Alice!');
  
  // Cleanup
  await userAContext.close();
  await userBContext.close();
  await userCContext.close();
  await browser.close();
});
```

### Context Options:

```typescript
import { chromium } from '@playwright/test';

async function contextWithOptions() {
  const browser = await chromium.launch();
  
  // Context with custom options
  const context = await browser.newContext({
    // Viewport size
    viewport: { width: 1920, height: 1080 },
    
    // Geolocation
    geolocation: { latitude: 40.7128, longitude: -74.0060 },
    permissions: ['geolocation'],
    
    // Timezone and locale
    locale: 'en-US',
    timezoneId: 'America/New_York',
    
    // Device emulation
    userAgent: 'Mozilla/5.0...',
    deviceScaleFactor: 2,
    isMobile: false,
    hasTouch: false,
    
    // HTTP credentials
    httpCredentials: {
      username: 'admin',
      password: 'admin123'
    },
    
    // Ignore HTTPS errors
    ignoreHTTPSErrors: true,
    
    // Offline mode
    offline: false,
    
    // Video recording
    recordVideo: {
      dir: './videos/',
      size: { width: 1280, height: 720 }
    },
    
    // Screenshot on navigation
    screenshot: 'only-on-failure',
  });
  
  const page = await context.newPage();
  await page.goto('https://example.com');
  
  await context.close();
  await browser.close();
}
```

### Scenario: E-commerce Multi-Account Testing

```typescript
import { test, expect, chromium } from '@playwright/test';

test('admin and customer can access different features', async () => {
  const browser = await chromium.launch();
  
  // Admin context
  const adminContext = await browser.newContext({
    storageState: './auth/admin-session.json' // Saved admin session
  });
  const adminPage = await adminContext.newPage();
  
  // Customer context
  const customerContext = await browser.newContext();
  const customerPage = await customerContext.newPage();
  
  // Admin adds a product
  await adminPage.goto('https://shop.example.com/admin/products');
  await adminPage.click('button:has-text("Add Product")');
  await adminPage.fill('#product-name', 'New Laptop');
  await adminPage.fill('#price', '999');
  await adminPage.click('button:has-text("Save")');
  await expect(adminPage.locator('.success-message')).toBeVisible();
  
  // Customer sees the new product
  await customerPage.goto('https://shop.example.com/products');
  await expect(customerPage.locator('text=New Laptop')).toBeVisible();
  
  // Customer adds to cart
  await customerPage.click('button:has-text("Add to Cart"):near(:text("New Laptop"))');
  await expect(customerPage.locator('.cart-count')).toHaveText('1');
  
  // Admin can see order stats (customer cannot)
  await adminPage.goto('https://shop.example.com/admin/orders');
  await expect(adminPage.locator('.order-stats')).toBeVisible();
  
  // Customer cannot access admin area
  await customerPage.goto('https://shop.example.com/admin');
  await expect(customerPage.locator('text=Access Denied')).toBeVisible();
  
  await adminContext.close();
  await customerContext.close();
  await browser.close();
});
```

### Saving and Reusing Context State:

```typescript
import { test, expect, chromium } from '@playwright/test';

// Save authenticated state
test('save authentication state', async () => {
  const browser = await chromium.launch();
  const context = await browser.newContext();
  const page = await context.newPage();
  
  // Login
  await page.goto('https://example.com/login');
  await page.fill('#email', 'user@example.com');
  await page.fill('#password', 'password123');
  await page.click('button[type="submit"]');
  await page.waitForURL('**/dashboard');
  
  // Save storage state (cookies, localStorage, sessionStorage)
  await context.storageState({ path: './auth/user-session.json' });
  
  await context.close();
  await browser.close();
});

// Reuse saved state (skip login)
test('reuse authentication state', async () => {
  const browser = await chromium.launch();
  
  // Load saved state
  const context = await browser.newContext({
    storageState: './auth/user-session.json'
  });
  
  const page = await context.newPage();
  
  // Already logged in!
  await page.goto('https://example.com/dashboard');
  await expect(page.locator('.user-profile')).toBeVisible();
  
  await context.close();
  await browser.close();
});
```

### Best Practices:

✅ **Use contexts for parallel testing** - Much faster than launching browsers  
✅ **One context per test** - Ensures isolation  
✅ **Save auth state** - Skip login in every test  
✅ **Close contexts** - Prevent memory leaks  
✅ **Use context options** - Configure geolocation, viewport, etc.  

### Common Patterns:

```typescript
import { test } from '@playwright/test';

// Playwright Test automatically creates contexts
test('automatic context creation', async ({ page, context }) => {
  // 'page' is from a fresh context
  // 'context' is the browser context
  
  await page.goto('https://example.com');
  
  // Create additional pages in same context
  const newPage = await context.newPage();
  await newPage.goto('https://example.com/about');
  
  // Both pages share cookies
});

// Custom context fixture
test('custom context with options', async ({ browser }) => {
  const context = await browser.newContext({
    viewport: { width: 1280, height: 720 },
    geolocation: { latitude: 51.5074, longitude: -0.1278 },
    permissions: ['geolocation']
  });
  
  const page = await context.newPage();
  await page.goto('https://maps.example.com');
  
  await context.close();
});
```

---

## Q4: How do you navigate pages and handle different navigation scenarios?

### Answer:

Playwright provides multiple methods for page navigation with built-in waiting and handling of various scenarios.

### Basic Navigation Methods:

```typescript
import { test, expect } from '@playwright/test';

test('basic navigation methods', async ({ page }) => {
  // 1. goto - Navigate to URL
  await page.goto('https://example.com');
  
  // 2. goto with options
  await page.goto('https://example.com/products', {
    waitUntil: 'networkidle', // Wait for network to be idle
    timeout: 30000 // 30 seconds timeout
  });
  
  // 3. Go back
  await page.goBack();
  
  // 4. Go forward
  await page.goForward();
  
  // 5. Reload page
  await page.reload();
  
  // 6. Current URL
  const currentUrl = page.url();
  console.log('Current URL:', currentUrl);
});
```

### Wait Until Options:

| Option | Description | When to Use |
|--------|-------------|-------------|
| `load` | Wait for `load` event (default) | Most pages |
| `domcontentloaded` | Wait for DOM ready | Fast navigation |
| `networkidle` | Wait for network to be idle | SPA, AJAX-heavy |
| `commit` | Wait for navigation commit | Very fast check |

```typescript
import { test } from '@playwright/test';

test('waitUntil options', async ({ page }) => {
  // Default: wait for load event
  await page.goto('https://example.com');
  
  // Wait for DOM ready (faster)
  await page.goto('https://example.com/fast', {
    waitUntil: 'domcontentloaded'
  });
  
  // Wait for network idle (for SPAs)
  await page.goto('https://example.com/spa', {
    waitUntil: 'networkidle'
  });
  
  // Wait for commit only (very fast)
  await page.goto('https://example.com/quick', {
    waitUntil: 'commit'
  });
});
```

### Scenario: SPA Navigation (React/Angular/Vue)

```typescript
import { test, expect } from '@playwright/test';

test('SPA navigation with React Router', async ({ page }) => {
  // Initial load
  await page.goto('https://react-app.example.com');
  
  // Click navigation link (no page reload in SPA)
  await page.click('a[href="/about"]');
  
  // Wait for URL change (not a full navigation)
  await page.waitForURL('**/about');
  
  // Verify content loaded
  await expect(page.locator('h1')).toHaveText('About Us');
  
  // Navigate to products
  await page.click('a[href="/products"]');
  await page.waitForURL('**/products');
  
  // Wait for dynamic content
  await page.waitForSelector('.product-card');
  
  // Verify products loaded
  await expect(page.locator('.product-card')).toHaveCount(10);
});
```

### Handling Navigation with Popups:

```typescript
import { test, expect } from '@playwright/test';

test('navigation that opens new tab', async ({ page, context }) => {
  await page.goto('https://example.com');
  
  // Click link that opens new tab
  const [newPage] = await Promise.all([
    context.waitForEvent('page'), // Wait for new page
    page.click('a[target="_blank"]') // Click link
  ]);
  
  // Work with new page
  await newPage.waitForLoadState();
  await expect(newPage).toHaveTitle(/New Page/);
  
  // Close new page
  await newPage.close();
  
  // Continue with original page
  await expect(page).toHaveURL('https://example.com');
});
```

### Navigation with Form Submission:

```typescript
import { test, expect } from '@playwright/test';

test('navigation after form submission', async ({ page }) => {
  await page.goto('https://example.com/search');
  
  // Fill and submit form
  await page.fill('#search-input', 'playwright testing');
  
  // Wait for navigation after form submit
  await Promise.all([
    page.waitForNavigation(), // Wait for navigation
    page.click('button[type="submit"]') // Trigger navigation
  ]);
  
  // Verify we're on results page
  await expect(page).toHaveURL(/.*search\?q=playwright/);
  await expect(page.locator('.search-results')).toBeVisible();
});
```

### Scenario: Multi-Step Checkout Process

```typescript
import { test, expect } from '@playwright/test';

test('complete checkout flow with multiple navigations', async ({ page }) => {
  // Step 1: Product page
  await page.goto('https://shop.example.com/products/laptop');
  await page.click('button:has-text("Add to Cart")');
  
  // Step 2: Cart page
  await page.click('a:has-text("View Cart")');
  await page.waitForURL('**/cart');
  await expect(page.locator('.cart-item')).toHaveCount(1);
  
  // Step 3: Shipping info
  await page.click('button:has-text("Proceed to Checkout")');
  await page.waitForURL('**/checkout/shipping');
  
  await page.fill('#name', 'John Doe');
  await page.fill('#address', '123 Main St');
  await page.fill('#city', 'New York');
  await page.selectOption('#state', 'NY');
  await page.fill('#zip', '10001');
  
  await page.click('button:has-text("Continue to Payment")');
  
  // Step 4: Payment info
  await page.waitForURL('**/checkout/payment');
  
  await page.fill('#card-number', '4242424242424242');
  await page.fill('#expiry', '12/25');
  await page.fill('#cvv', '123');
  
  // Step 5: Place order (final navigation)
  await Promise.all([
    page.waitForNavigation({ waitUntil: 'networkidle' }),
    page.click('button:has-text("Place Order")')
  ]);
  
  // Step 6: Confirmation page
  await expect(page).toHaveURL(/.*order-confirmation/);
  await expect(page.locator('.success-message')).toBeVisible();
  await expect(page.locator('.order-number')).toContainText('ORDER-');
});
```

### Handling Redirects:

```typescript
import { test, expect } from '@playwright/test';

test('follow redirects', async ({ page }) => {
  // Navigate to URL that redirects
  await page.goto('https://example.com/old-page');
  
  // Playwright automatically follows redirects
  await expect(page).toHaveURL('https://example.com/new-page');
});

test('check redirect chain', async ({ page }) => {
  const response = await page.goto('https://example.com/redirect');
  
  // Get redirect chain
  const chain = response?.request().redirectedFrom();
  console.log('Redirected from:', chain?.url());
});
```

### Waiting for Navigation:

```typescript
import { test, expect } from '@playwright/test';

test('explicit navigation wait', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Method 1: waitForNavigation
  await Promise.all([
    page.waitForNavigation({ url: '**/dashboard' }),
    page.click('a:has-text("Dashboard")')
  ]);
  
  // Method 2: waitForURL (simpler)
  await page.click('a:has-text("Profile")');
  await page.waitForURL('**/profile');
  
  // Method 3: waitForLoadState
  await page.click('a:has-text("Settings")');
  await page.waitForLoadState('networkidle');
});
```

### Scenario: Login with Multiple Redirects

```typescript
import { test, expect } from '@playwright/test';

test('login flow with redirects', async ({ page }) => {
  // Navigate to protected page (redirects to login)
  await page.goto('https://example.com/dashboard');
  
  // Should redirect to login
  await page.waitForURL('**/login');
  
  // Perform login
  await page.fill('#email', 'user@example.com');
  await page.fill('#password', 'password123');
  
  // Click login (redirects back to dashboard)
  await Promise.all([
    page.waitForURL('**/dashboard'),
    page.click('button:has-text("Login")')
  ]);
  
  // Verify successful login
  await expect(page.locator('.user-menu')).toBeVisible();
  await expect(page.locator('.welcome-message')).toContainText('Welcome back');
});
```

### Navigation with Network Interception:

```typescript
import { test, expect } from '@playwright/test';

test('navigate with API mocking', async ({ page }) => {
  // Mock API response
  await page.route('**/api/products', route => {
    route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify([
        { id: 1, name: 'Product 1', price: 99 },
        { id: 2, name: 'Product 2', price: 149 }
      ])
    });
  });
  
  // Navigate to page that calls the API
  await page.goto('https://example.com/products');
  
  // Verify mocked data is displayed
  await expect(page.locator('.product-card')).toHaveCount(2);
  await expect(page.locator('.product-card').first()).toContainText('Product 1');
});
```

### Error Handling:

```typescript
import { test, expect } from '@playwright/test';

test('handle navigation errors', async ({ page }) => {
  try {
    // Try to navigate to invalid URL
    await page.goto('https://invalid-domain-12345.com', {
      timeout: 10000
    });
  } catch (error) {
    console.log('Navigation failed:', error.message);
  }
  
  // Navigate to valid page instead
  await page.goto('https://example.com');
});

test('handle 404 pages', async ({ page }) => {
  const response = await page.goto('https://example.com/nonexistent');
  
  // Check response status
  expect(response?.status()).toBe(404);
  
  // Verify 404 page content
  await expect(page.locator('h1')).toContainText('Page Not Found');
});
```

### Best Practices:

✅ **Use waitForURL** instead of waitForNavigation (simpler)  
✅ **Choose appropriate waitUntil** option for your app type  
✅ **Handle navigation promises** with Promise.all()  
✅ **Check response status** for error pages  
✅ **Use networkidle** for AJAX-heavy SPAs  
✅ **Save navigation state** for debugging  

### Common Pitfalls:

❌ **Don't**: `await page.click(); await page.waitForNavigation();` (race condition)  
✅ **Do**: `await Promise.all([page.waitForNavigation(), page.click()]);`

❌ **Don't**: Use fixed timeouts for navigation  
✅ **Do**: Use waitForURL or waitForLoadState

---

**Related Topics:**
- Q5-Q6: Selectors and locators
- Q13-Q14: API testing and network interception
- Q27-Q28: Request interception and mocking

---

*Document Version: 1.0*  
*Last Updated: November 2025*
