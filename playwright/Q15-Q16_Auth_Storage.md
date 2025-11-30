# Playwright Interview Questions: Q15-Q16

## Q15: How do you handle authentication in Playwright?

### Answer:

Playwright provides multiple strategies for handling authentication efficiently, including state reuse, API-based login, and browser context storage to avoid repeated login flows.

### 1. Authentication Strategies

| Strategy | Use Case | Speed | Pros | Cons |
|----------|----------|-------|------|------|
| **UI Login** | Testing login flow | Slow | Tests actual UI | Repeats for each test |
| **API Login** | Fast test setup | Fast | Quick, reliable | Skips UI validation |
| **Storage State** | Reuse across tests | Very Fast | No repeated login | Requires initial setup |
| **Browser Context** | Isolated sessions | Medium | Clean isolation | Setup per context |
| **HTTP Auth** | Basic/Digest auth | Fast | Simple | Limited use cases |

### 2. Basic UI Login

```typescript
import { test, expect } from '@playwright/test';

test('login via UI', async ({ page }) => {
  await page.goto('https://example.com/login');
  
  await page.locator('#email').fill('user@example.com');
  await page.locator('#password').fill('SecurePass123!');
  await page.locator('#login-btn').click();
  
  // Wait for navigation
  await page.waitForURL('**/dashboard');
  
  // Verify logged in
  await expect(page.locator('.user-profile')).toBeVisible();
  await expect(page.locator('.user-name')).toHaveText('John Doe');
});
```

### 3. Storage State (Recommended for Speed)

```typescript
// auth.setup.ts - Run once to save auth state
import { test as setup } from '@playwright/test';

const authFile = 'playwright/.auth/user.json';

setup('authenticate', async ({ page }) => {
  await page.goto('https://example.com/login');
  await page.locator('#email').fill('user@example.com');
  await page.locator('#password').fill('SecurePass123!');
  await page.locator('#login-btn').click();
  
  await page.waitForURL('**/dashboard');
  
  // Save signed-in state
  await page.context().storageState({ path: authFile });
});

// playwright.config.ts
export default defineConfig({
  projects: [
    { name: 'setup', testMatch: /.*\.setup\.ts/ },
    {
      name: 'chromium',
      use: { 
        ...devices['Desktop Chrome'],
        storageState: authFile,
      },
      dependencies: ['setup'],
    },
  ],
});

// Now all tests use authenticated state
test('authenticated test', async ({ page }) => {
  await page.goto('https://example.com/dashboard');
  // Already logged in!
  await expect(page.locator('.user-profile')).toBeVisible();
});
```

### 4. API-Based Authentication

```typescript
test('login via API', async ({ page, request }) => {
  // Login via API
  const response = await request.post('https://api.example.com/auth/login', {
    data: {
      email: 'user@example.com',
      password: 'SecurePass123!'
    }
  });
  
  expect(response.ok()).toBeTruthy();
  const { token } = await response.json();
  
  // Set token in page context
  await page.goto('https://example.com');
  await page.evaluate((authToken) => {
    localStorage.setItem('authToken', authToken);
  }, token);
  
  // Navigate to protected page
  await page.goto('https://example.com/dashboard');
  await expect(page.locator('.user-profile')).toBeVisible();
});
```

### 5. HTTP Basic Authentication

```typescript
test('HTTP basic auth', async ({ browser }) => {
  const context = await browser.newContext({
    httpCredentials: {
      username: 'admin',
      password: 'secret123'
    }
  });
  
  const page = await context.newPage();
  await page.goto('https://example.com/admin');
  
  // Automatically authenticated
  await expect(page.locator('h1')).toContainText('Admin Panel');
  
  await context.close();
});
```

### 6. Real-World Scenario: Multi-User Testing

```typescript
// Setup multiple user states
// admin.setup.ts
setup('admin auth', async ({ page }) => {
  await page.goto('https://example.com/login');
  await page.locator('#email').fill('admin@example.com');
  await page.locator('#password').fill('AdminPass123!');
  await page.locator('#login-btn').click();
  await page.waitForURL('**/dashboard');
  await page.context().storageState({ path: 'playwright/.auth/admin.json' });
});

// user.setup.ts
setup('user auth', async ({ page }) => {
  await page.goto('https://example.com/login');
  await page.locator('#email').fill('user@example.com');
  await page.locator('#password').fill('UserPass123!');
  await page.locator('#login-btn').click();
  await page.waitForURL('**/dashboard');
  await page.context().storageState({ path: 'playwright/.auth/user.json' });
});

// playwright.config.ts
export default defineConfig({
  projects: [
    { name: 'setup', testMatch: /.*\.setup\.ts/ },
    {
      name: 'admin-tests',
      use: { storageState: 'playwright/.auth/admin.json' },
      dependencies: ['setup'],
      testMatch: '**/admin/**/*.spec.ts'
    },
    {
      name: 'user-tests',
      use: { storageState: 'playwright/.auth/user.json' },
      dependencies: ['setup'],
      testMatch: '**/user/**/*.spec.ts'
    }
  ]
});

// Test as admin
test('admin can delete users', async ({ page }) => {
  await page.goto('https://example.com/admin/users');
  await page.locator('[data-user-id="123"] .delete-btn').click();
  await page.locator('#confirm-delete').click();
  await expect(page.locator('.success-message')).toBeVisible();
});

// Test as regular user
test('user cannot access admin panel', async ({ page }) => {
  await page.goto('https://example.com/admin/users');
  await expect(page).toHaveURL('**/403'); // Forbidden
  await expect(page.locator('.error-message')).toContainText('Access Denied');
});
```

### 7. OAuth / Social Login

```typescript
test('Google OAuth login', async ({ page, context }) => {
  await page.goto('https://example.com/login');
  
  // Click "Sign in with Google"
  const [popup] = await Promise.all([
    context.waitForEvent('page'),
    page.locator('#google-login-btn').click()
  ]);
  
  // Fill Google login form
  await popup.waitForLoadState();
  await popup.locator('#identifierId').fill('testuser@gmail.com');
  await popup.locator('#identifierNext').click();
  
  await popup.locator('input[type="password"]').fill('password123');
  await popup.locator('#passwordNext').click();
  
  // Wait for redirect back to main page
  await page.waitForURL('**/dashboard');
  await expect(page.locator('.user-email')).toHaveText('testuser@gmail.com');
});
```

### 8. Token-Based Authentication

```typescript
test.describe('JWT token authentication', () => {
  let authToken: string;
  
  test.beforeAll(async ({ request }) => {
    const response = await request.post('https://api.example.com/auth/login', {
      data: {
        email: 'user@example.com',
        password: 'SecurePass123!'
      }
    });
    
    const data = await response.json();
    authToken = data.token;
  });
  
  test('access protected page with token', async ({ page }) => {
    // Set token before navigation
    await page.addInitScript((token) => {
      window.localStorage.setItem('authToken', token);
    }, authToken);
    
    await page.goto('https://example.com/dashboard');
    await expect(page.locator('.user-profile')).toBeVisible();
  });
  
  test('API calls with auth header', async ({ page }) => {
    await page.goto('https://example.com');
    
    // Intercept and add auth header
    await page.route('**/api/**', async (route) => {
      const headers = {
        ...route.request().headers(),
        'Authorization': `Bearer ${authToken}`
      };
      await route.continue({ headers });
    });
    
    await page.goto('https://example.com/profile');
    await expect(page.locator('.profile-data')).toBeVisible();
  });
});
```

### 9. Session Cookie Authentication

```typescript
test('login with session cookies', async ({ page, context }) => {
  // Login
  await page.goto('https://example.com/login');
  await page.locator('#email').fill('user@example.com');
  await page.locator('#password').fill('SecurePass123!');
  await page.locator('#login-btn').click();
  await page.waitForURL('**/dashboard');
  
  // Get cookies
  const cookies = await context.cookies();
  const sessionCookie = cookies.find(c => c.name === 'session_id');
  
  console.log('Session cookie:', sessionCookie?.value);
  
  // Create new context with same cookies
  const newContext = await page.context().browser()!.newContext();
  await newContext.addCookies([sessionCookie!]);
  
  const newPage = await newContext.newPage();
  await newPage.goto('https://example.com/dashboard');
  
  // Already authenticated
  await expect(newPage.locator('.user-profile')).toBeVisible();
  
  await newContext.close();
});
```

### 10. Real-World Scenario: E-commerce Multi-Account Testing

```typescript
test.describe('E-commerce multi-account scenarios', () => {
  test('customer and admin interactions', async ({ browser }) => {
    // Create customer context
    const customerContext = await browser.newContext({
      storageState: 'playwright/.auth/customer.json'
    });
    const customerPage = await customerContext.newPage();
    
    // Create admin context
    const adminContext = await browser.newContext({
      storageState: 'playwright/.auth/admin.json'
    });
    const adminPage = await adminContext.newPage();
    
    // Customer places order
    await customerPage.goto('https://shop.example.com/products');
    await customerPage.locator('[data-product-id="123"] .add-to-cart').click();
    await customerPage.locator('.cart-icon').click();
    await customerPage.locator('#checkout-btn').click();
    
    // Fill checkout
    await customerPage.locator('#card-number').fill('4242424242424242');
    await customerPage.locator('#place-order').click();
    
    // Get order ID
    await customerPage.waitForURL('**/order/**');
    const orderUrl = customerPage.url();
    const orderId = orderUrl.match(/order\/([^/]+)/)?.[1];
    
    // Admin verifies order
    await adminPage.goto(`https://shop.example.com/admin/orders/${orderId}`);
    await expect(adminPage.locator('.order-status')).toHaveText('Pending');
    
    // Admin processes order
    await adminPage.locator('#process-order').click();
    await adminPage.locator('#confirm').click();
    
    // Customer sees updated status
    await customerPage.goto(`https://shop.example.com/orders/${orderId}`);
    await expect(customerPage.locator('.order-status')).toHaveText('Processing');
    
    await customerContext.close();
    await adminContext.close();
  });
});
```

### 11. Two-Factor Authentication (2FA)

```typescript
test('login with 2FA', async ({ page }) => {
  await page.goto('https://example.com/login');
  
  // Step 1: Username and password
  await page.locator('#email').fill('user@example.com');
  await page.locator('#password').fill('SecurePass123!');
  await page.locator('#login-btn').click();
  
  // Step 2: 2FA code (mock/bypass in tests)
  await page.waitForURL('**/2fa');
  
  // Option 1: Use test account with known TOTP secret
  // Option 2: Mock the verification endpoint
  await page.route('**/api/2fa/verify', route => {
    route.fulfill({
      status: 200,
      body: JSON.stringify({ success: true })
    });
  });
  
  await page.locator('#2fa-code').fill('123456');
  await page.locator('#verify-btn').click();
  
  // Successfully logged in
  await page.waitForURL('**/dashboard');
  await expect(page.locator('.user-profile')).toBeVisible();
});
```

### 12. Best Practices

✅ **GOOD: Reuse authentication state**
```typescript
// Setup once, use everywhere
use: { storageState: 'playwright/.auth/user.json' }
```

❌ **BAD: Login in every test**
```typescript
test.beforeEach(async ({ page }) => {
  // Slow! Repeats login for every test
  await page.goto('/login');
  await page.fill('#email', 'user@example.com');
  // ...
});
```

✅ **GOOD: Use API for auth when possible**
```typescript
const { token } = await request.post('/api/login', { data });
```

❌ **BAD: Always use UI login**
```typescript
// Much slower
await page.click('#login');
await page.fill(...);
```

### 13. Common Pitfalls

1. **Not waiting for auth to complete**
   ```typescript
   // ❌ May not be fully authenticated
   await page.locator('#login-btn').click();
   await page.goto('/dashboard');
   
   // ✅ Wait for auth confirmation
   await page.locator('#login-btn').click();
   await page.waitForURL('**/dashboard');
   ```

2. **Sharing state between incompatible tests**
   ```typescript
   // ❌ Tests interfere with each other
   test('delete account', async ({ page }) => {
     // Deletes the shared auth account!
   });
   
   // ✅ Use separate storage states
   test.use({ storageState: 'temp-user.json' });
   ```

### Related Topics
- **Q16**: Cookies and storage management
- **Q13**: API authentication
- **Q31**: Page Object Model for login pages

---

## Q16: How do you work with cookies and local storage?

### Answer:

Playwright provides comprehensive APIs for managing cookies, localStorage, and sessionStorage, enabling tests to manipulate browser storage for authentication, preferences, and state management.

### 1. Storage Management APIs

| API | Purpose | Scope | Example |
|-----|---------|-------|---------|
| **context.cookies()** | Get all cookies | Browser context | `await context.cookies()` |
| **context.addCookies()** | Add cookies | Browser context | `await context.addCookies([cookie])` |
| **context.clearCookies()** | Clear cookies | Browser context | `await context.clearCookies()` |
| **page.evaluate()** | Access localStorage | Page | `await page.evaluate(() => localStorage)` |
| **page.addInitScript()** | Set storage before load | Page | Set values before navigation |
| **storageState()** | Save/load all storage | Browser context | `await context.storageState()` |

### 2. Working with Cookies

```typescript
import { test, expect } from '@playwright/test';

test('get and set cookies', async ({ page, context }) => {
  await page.goto('https://example.com');
  
  // Get all cookies
  const cookies = await context.cookies();
  console.log('All cookies:', cookies);
  
  // Get specific cookie
  const sessionCookie = cookies.find(c => c.name === 'session_id');
  console.log('Session cookie:', sessionCookie);
  
  // Add cookie
  await context.addCookies([{
    name: 'user_preference',
    value: 'dark_mode',
    domain: 'example.com',
    path: '/',
    expires: Date.now() / 1000 + 86400 // 24 hours
  }]);
  
  // Reload to see cookie in action
  await page.reload();
  
  // Get cookies for specific URL
  const specificCookies = await context.cookies('https://example.com/profile');
});

test('clear cookies', async ({ page, context }) => {
  await page.goto('https://example.com');
  
  // Clear all cookies
  await context.clearCookies();
  
  // Clear specific cookie
  await context.clearCookies({ name: 'session_id' });
  
  // Clear cookies for specific domain
  await context.clearCookies({ domain: 'example.com' });
  
  // Verify cleared
  const cookies = await context.cookies();
  expect(cookies).toHaveLength(0);
});
```

### 3. Working with localStorage

```typescript
test('localStorage operations', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Set localStorage item
  await page.evaluate(() => {
    localStorage.setItem('theme', 'dark');
    localStorage.setItem('language', 'en');
    localStorage.setItem('user_settings', JSON.stringify({
      notifications: true,
      autoSave: true
    }));
  });
  
  // Get localStorage item
  const theme = await page.evaluate(() => localStorage.getItem('theme'));
  expect(theme).toBe('dark');
  
  // Get all localStorage
  const allStorage = await page.evaluate(() => {
    return Object.keys(localStorage).reduce((acc, key) => {
      acc[key] = localStorage.getItem(key);
      return acc;
    }, {} as Record<string, string | null>);
  });
  console.log('All localStorage:', allStorage);
  
  // Remove item
  await page.evaluate(() => localStorage.removeItem('theme'));
  
  // Clear all localStorage
  await page.evaluate(() => localStorage.clear());
  
  // Verify cleared
  const count = await page.evaluate(() => localStorage.length);
  expect(count).toBe(0);
});
```

### 4. Working with sessionStorage

```typescript
test('sessionStorage operations', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Set sessionStorage
  await page.evaluate(() => {
    sessionStorage.setItem('temp_data', 'temporary');
    sessionStorage.setItem('form_draft', JSON.stringify({
      name: 'John',
      email: 'john@example.com'
    }));
  });
  
  // Get sessionStorage
  const tempData = await page.evaluate(() => 
    sessionStorage.getItem('temp_data')
  );
  expect(tempData).toBe('temporary');
  
  // Navigate to another page (sessionStorage persists)
  await page.goto('https://example.com/other');
  const stillThere = await page.evaluate(() => 
    sessionStorage.getItem('temp_data')
  );
  expect(stillThere).toBe('temporary');
  
  // Clear sessionStorage
  await page.evaluate(() => sessionStorage.clear());
});
```

### 5. Setting Storage Before Page Load

```typescript
test('set localStorage before navigation', async ({ page }) => {
  // Add init script that runs before page loads
  await page.addInitScript(() => {
    localStorage.setItem('authToken', 'fake-jwt-token');
    localStorage.setItem('user_id', '12345');
  });
  
  // Now navigate - storage already set
  await page.goto('https://example.com/dashboard');
  
  // App reads token from localStorage and shows dashboard
  await expect(page.locator('.user-profile')).toBeVisible();
});

test('set storage with parameters', async ({ page }) => {
  const authData = {
    token: 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...',
    userId: '12345',
    email: 'user@example.com'
  };
  
  await page.addInitScript((data) => {
    localStorage.setItem('auth', JSON.stringify(data));
  }, authData);
  
  await page.goto('https://example.com');
});
```

### 6. Storage State (Save and Load)

```typescript
test('save storage state', async ({ page, context }) => {
  await page.goto('https://example.com/login');
  
  // Login
  await page.locator('#email').fill('user@example.com');
  await page.locator('#password').fill('SecurePass123!');
  await page.locator('#login-btn').click();
  await page.waitForURL('**/dashboard');
  
  // Save everything (cookies + localStorage + sessionStorage)
  await context.storageState({ path: 'state.json' });
  
  // File contains:
  // {
  //   "cookies": [...],
  //   "origins": [
  //     {
  //       "origin": "https://example.com",
  //       "localStorage": [...],
  //       "sessionStorage": [...]
  //     }
  //   ]
  // }
});

test('load storage state', async ({ browser }) => {
  // Create context with saved state
  const context = await browser.newContext({
    storageState: 'state.json'
  });
  
  const page = await context.newPage();
  
  // Already authenticated!
  await page.goto('https://example.com/dashboard');
  await expect(page.locator('.user-profile')).toBeVisible();
  
  await context.close();
});
```

### 7. Real-World Scenario: Testing User Preferences

```typescript
test('user preferences persistence', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Set preferences via UI
  await page.locator('#theme-toggle').click();
  await page.locator('[data-theme="dark"]').click();
  
  await page.locator('#language-select').selectOption('es');
  
  await page.locator('#notifications-toggle').check();
  
  // Verify saved in localStorage
  const preferences = await page.evaluate(() => {
    return JSON.parse(localStorage.getItem('userPreferences') || '{}');
  });
  
  expect(preferences.theme).toBe('dark');
  expect(preferences.language).toBe('es');
  expect(preferences.notifications).toBe(true);
  
  // Reload page
  await page.reload();
  
  // Preferences should persist
  await expect(page.locator('body')).toHaveClass(/dark-theme/);
  await expect(page.locator('html')).toHaveAttribute('lang', 'es');
  await expect(page.locator('#notifications-toggle')).toBeChecked();
});
```

### 8. Real-World Scenario: Shopping Cart Persistence

```typescript
test('shopping cart localStorage persistence', async ({ page }) => {
  await page.goto('https://shop.example.com');
  
  // Add items to cart
  await page.locator('[data-product-id="123"] .add-to-cart').click();
  await page.locator('[data-product-id="456"] .add-to-cart').click();
  
  // Check cart in localStorage
  const cart = await page.evaluate(() => {
    return JSON.parse(localStorage.getItem('cart') || '[]');
  });
  
  expect(cart).toHaveLength(2);
  expect(cart[0].productId).toBe('123');
  expect(cart[1].productId).toBe('456');
  
  // Simulate browser close and reopen
  const cartData = await page.evaluate(() => localStorage.getItem('cart'));
  await page.close();
  
  // New page
  const newPage = await page.context().newPage();
  
  // Restore cart
  await newPage.addInitScript((data) => {
    localStorage.setItem('cart', data);
  }, cartData);
  
  await newPage.goto('https://shop.example.com');
  
  // Cart should be restored
  await expect(newPage.locator('.cart-badge')).toHaveText('2');
  await newPage.locator('.cart-icon').click();
  await expect(newPage.locator('.cart-item')).toHaveCount(2);
});
```

### 9. Cookie Manipulation for Testing

```typescript
test('test with different cookie scenarios', async ({ page, context }) => {
  await page.goto('https://example.com');
  
  // Scenario 1: User accepted cookies
  await context.addCookies([{
    name: 'cookie_consent',
    value: 'accepted',
    domain: 'example.com',
    path: '/'
  }]);
  
  await page.reload();
  await expect(page.locator('.cookie-banner')).not.toBeVisible();
  
  // Scenario 2: User has premium account
  await context.clearCookies();
  await context.addCookies([
    {
      name: 'user_type',
      value: 'premium',
      domain: 'example.com',
      path: '/'
    },
    {
      name: 'session_id',
      value: 'premium_session_123',
      domain: 'example.com',
      path: '/',
      httpOnly: true,
      secure: true
    }
  ]);
  
  await page.reload();
  await expect(page.locator('.premium-badge')).toBeVisible();
  
  // Scenario 3: Session expired
  await context.addCookies([{
    name: 'session_id',
    value: 'expired_session',
    domain: 'example.com',
    path: '/',
    expires: 0 // Expired
  }]);
  
  await page.reload();
  await expect(page).toHaveURL('**/login');
});
```

### 10. Testing Cookie Consent

```typescript
test('cookie consent flow', async ({ page, context }) => {
  await page.goto('https://example.com');
  
  // Cookie banner should appear
  await expect(page.locator('.cookie-consent-banner')).toBeVisible();
  
  // No cookies set yet (except essential)
  let cookies = await context.cookies();
  const trackingCookie = cookies.find(c => c.name === 'analytics');
  expect(trackingCookie).toBeUndefined();
  
  // Accept cookies
  await page.locator('#accept-cookies-btn').click();
  
  // Cookie set
  cookies = await context.cookies();
  const consentCookie = cookies.find(c => c.name === 'cookie_consent');
  expect(consentCookie?.value).toBe('accepted');
  
  // Banner disappears
  await expect(page.locator('.cookie-consent-banner')).not.toBeVisible();
  
  // Reload - banner should not appear again
  await page.reload();
  await expect(page.locator('.cookie-consent-banner')).not.toBeVisible();
});
```

### 11. Cross-Domain Storage

```typescript
test('cross-domain storage handling', async ({ page }) => {
  // Set storage on domain A
  await page.goto('https://domainA.com');
  await page.evaluate(() => {
    localStorage.setItem('domainA_data', 'valueA');
  });
  
  // Navigate to domain B
  await page.goto('https://domainB.com');
  
  // Cannot access domain A's storage
  const domainBStorage = await page.evaluate(() => {
    return localStorage.getItem('domainA_data');
  });
  expect(domainBStorage).toBeNull();
  
  // Set storage on domain B
  await page.evaluate(() => {
    localStorage.setItem('domainB_data', 'valueB');
  });
  
  // Go back to domain A
  await page.goto('https://domainA.com');
  
  // Domain A storage still there
  const domainAStorage = await page.evaluate(() => {
    return localStorage.getItem('domainA_data');
  });
  expect(domainAStorage).toBe('valueA');
});
```

### 12. Testing Storage Limits

```typescript
test('localStorage size limit handling', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Try to fill localStorage
  const result = await page.evaluate(() => {
    try {
      const largeData = 'x'.repeat(1024 * 1024); // 1MB
      for (let i = 0; i < 10; i++) {
        localStorage.setItem(`large_key_${i}`, largeData);
      }
      return { success: true };
    } catch (e) {
      return { success: false, error: (e as Error).message };
    }
  });
  
  if (!result.success) {
    console.log('Storage limit reached:', result.error);
    // App should handle quota exceeded error
    await expect(page.locator('.storage-error')).toBeVisible();
  }
});
```

### 13. Best Practices

✅ **GOOD: Use storageState for auth**
```typescript
await context.storageState({ path: 'auth.json' });
// Reuse in next test
const context = await browser.newContext({ storageState: 'auth.json' });
```

❌ **BAD: Manually copy cookies**
```typescript
const cookies = await context.cookies();
// Manually reconstruct everything...
```

✅ **GOOD: Set storage before navigation**
```typescript
await page.addInitScript(() => {
  localStorage.setItem('token', 'value');
});
await page.goto('/');
```

❌ **BAD: Set after navigation**
```typescript
await page.goto('/');
await page.evaluate(() => localStorage.setItem('token', 'value'));
await page.reload(); // Unnecessary reload
```

### 14. Common Pitfalls

1. **Forgetting domain restrictions**
   ```typescript
   // ❌ Won't work - wrong domain
   await context.addCookies([{
     name: 'test',
     value: 'value',
     domain: 'example.com' // But page is on different.com
   }]);
   
   // ✅ Match the domain
   await page.goto('https://example.com');
   await context.addCookies([{
     name: 'test',
     value: 'value',
     domain: 'example.com'
   }]);
   ```

2. **Not handling storage security**
   ```typescript
   // ❌ Insecure cookie
   await context.addCookies([{
     name: 'session',
     value: 'secret',
     secure: false // Should be true for HTTPS
   }]);
   
   // ✅ Secure cookie
   await context.addCookies([{
     name: 'session',
     value: 'secret',
     secure: true,
     httpOnly: true,
     sameSite: 'Strict'
   }]);
   ```

### Related Topics
- **Q15**: Authentication strategies
- **Q3**: Browser contexts and isolation
- **Q28**: Testing offline with service workers

---

*Document Version: 1.0*  
*Last Updated: November 2025*
