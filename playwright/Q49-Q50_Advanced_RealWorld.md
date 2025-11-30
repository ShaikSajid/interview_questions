# Playwright Interview Questions: Q49-Q50

## Q49: What are advanced Playwright patterns?

### Answer:

Advanced patterns for complex testing scenarios.

### 1. Custom Test Extension

```typescript
import { test as base } from '@playwright/test';

class TestHelper {
  constructor(private page: Page) {}
  
  async loginAs(role: 'admin' | 'user') {
    // Complex login logic
  }
  
  async createTestData() {
    // API calls to create data
  }
}

export const test = base.extend<{ helper: TestHelper }>({
  helper: async ({ page }, use) => {
    const helper = new TestHelper(page);
    await use(helper);
  },
});

test('use helper', async ({ helper }) => {
  await helper.loginAs('admin');
  await helper.createTestData();
});
```

### 2. Global Setup/Teardown

```typescript
// global-setup.ts
import { chromium, FullConfig } from '@playwright/test';

async function globalSetup(config: FullConfig) {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  
  // Perform global authentication
  await page.goto('https://example.com/login');
  await page.fill('#email', 'admin@example.com');
  await page.fill('#password', 'password');
  await page.click('#login');
  
  await page.context().storageState({ path: 'auth.json' });
  await browser.close();
}

export default globalSetup;

// playwright.config.ts
export default defineConfig({
  globalSetup: require.resolve('./global-setup'),
  use: { storageState: 'auth.json' },
});
```

### 3. Test Sharding

```bash
# Split tests across 4 machines
npx playwright test --shard=1/4
npx playwright test --shard=2/4
npx playwright test --shard=3/4
npx playwright test --shard=4/4
```

### Related Topics
- **Q50**: Real-world scenarios
- **Q33**: Fixtures

---

## Q50: How do you handle real-world testing scenarios?

### Answer:

Practical approaches to complex real-world testing challenges.

### 1. Multi-User Scenarios

```typescript
test('multi-user chat', async ({ browser }) => {
  // Create 3 users
  const user1Context = await browser.newContext();
  const user2Context = await browser.newContext();
  const user3Context = await browser.newContext();
  
  const user1Page = await user1Context.newPage();
  const user2Page = await user2Context.newPage();
  const user3Page = await user3Context.newPage();
  
  // All users join chat
  await Promise.all([
    user1Page.goto('https://chat.example.com'),
    user2Page.goto('https://chat.example.com'),
    user3Page.goto('https://chat.example.com'),
  ]);
  
  // User 1 sends message
  await user1Page.locator('#message').fill('Hello!');
  await user1Page.locator('#send').click();
  
  // User 2 and 3 see the message
  await expect(user2Page.locator('.message').last()).toContainText('Hello!');
  await expect(user3Page.locator('.message').last()).toContainText('Hello!');
  
  await user1Context.close();
  await user2Context.close();
  await user3Context.close();
});
```

### 2. E2E E-commerce Flow

```typescript
test('complete purchase flow', async ({ page, request }) => {
  // 1. Browse products
  await page.goto('https://shop.example.com');
  await page.locator('#search').fill('laptop');
  await page.locator('#search').press('Enter');
  
  // 2. Add to cart
  await page.locator('.product-card').first().click();
  await page.locator('#add-to-cart').click();
  await expect(page.locator('.cart-badge')).toHaveText('1');
  
  // 3. Checkout
  await page.locator('.cart-icon').click();
  await page.locator('#checkout').click();
  
  // 4. Fill shipping
  await page.locator('#address').fill('123 Main St');
  await page.locator('#city').fill('New York');
  await page.locator('#zip').fill('10001');
  await page.locator('#next').click();
  
  // 5. Payment (mocked)
  await page.route('**/api/payment', route => {
    route.fulfill({
      status: 200,
      body: JSON.stringify({ success: true, orderId: '12345' })
    });
  });
  
  await page.locator('#card-number').fill('4242424242424242');
  await page.locator('#expiry').fill('12/25');
  await page.locator('#cvv').fill('123');
  await page.locator('#pay').click();
  
  // 6. Verify order
  await expect(page.locator('.success')).toBeVisible();
  await expect(page.locator('.order-id')).toContainText('12345');
  
  // 7. Verify in admin (via API)
  const orderResponse = await request.get('/api/orders/12345');
  expect(orderResponse.ok()).toBeTruthy();
});
```

### 3. Complex Form with Validation

```typescript
test('multi-step form validation', async ({ page }) => {
  await page.goto('https://forms.example.com');
  
  // Step 1: Personal Info
  await page.locator('#firstName').fill('');
  await page.locator('#next').click();
  await expect(page.locator('.error')).toContainText('First name required');
  
  await page.locator('#firstName').fill('John');
  await page.locator('#lastName').fill('Doe');
  await page.locator('#email').fill('invalid-email');
  await page.locator('#next').click();
  await expect(page.locator('.error')).toContainText('Invalid email');
  
  await page.locator('#email').fill('john@example.com');
  await page.locator('#next').click();
  
  // Step 2: File Upload
  await page.locator('input[type="file"]').setInputFiles('resume.pdf');
  await page.locator('#next').click();
  
  // Step 3: Review and Submit
  await expect(page.locator('.review-name')).toHaveText('John Doe');
  await expect(page.locator('.review-email')).toHaveText('john@example.com');
  
  await page.locator('#submit').click();
  await expect(page.locator('.success')).toBeVisible();
});
```

### Conclusion

This completes all 50 Playwright interview questions covering:
- ✅ Fundamentals (Q1-Q10)
- ✅ Intermediate Topics (Q11-Q30)
- ✅ Advanced Patterns (Q31-Q50)

Each question includes theory, code examples, real-world scenarios, and best practices.

### Related Topics
- **Q31**: Page Object Model
- **Q35**: CI/CD Integration
- **Q41**: Debugging

---

*Document Version: 1.0*  
*Last Updated: November 2025*
