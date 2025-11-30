# Playwright Interview Questions: Q9-Q10

## Q9: What are the different types of assertions in Playwright?

### Answer:

Playwright provides **web-first assertions** that automatically wait and retry until the condition is met. These assertions are specifically designed for web testing with built-in auto-waiting mechanisms.

### 1. Types of Assertions

| Assertion Type | Purpose | Auto-Wait | Example |
|---------------|---------|-----------|---------|
| **expect(locator)** | Locator-based assertions | ✅ Yes | `expect(page.locator('#btn')).toBeVisible()` |
| **expect(page)** | Page-level assertions | ✅ Yes | `expect(page).toHaveTitle('Home')` |
| **expect(value)** | Generic assertions | ❌ No | `expect(count).toBe(5)` |
| **await expect().poll()** | Polling assertions | ✅ Yes | `await expect.poll(() => getCount()).toBe(5)` |
| **Soft assertions** | Continue on failure | ✅ Yes | `await expect.soft(locator).toBeVisible()` |

### 2. Common Locator Assertions

```typescript
import { test, expect } from '@playwright/test';

test('locator assertions demo', async ({ page }) => {
  await page.goto('https://example.com/shop');
  
  const addToCart = page.locator('#add-to-cart');
  
  // Visibility assertions
  await expect(addToCart).toBeVisible();
  await expect(addToCart).toBeHidden(); // opposite
  
  // State assertions
  await expect(addToCart).toBeEnabled();
  await expect(addToCart).toBeDisabled();
  await expect(addToCart).toBeChecked(); // for checkboxes
  await expect(addToCart).toBeEditable();
  await expect(addToCart).toBeFocused();
  
  // Text content assertions
  await expect(addToCart).toHaveText('Add to Cart');
  await expect(addToCart).toContainText('Add'); // partial match
  await expect(addToCart).toHaveText(/add to cart/i); // regex
  
  // Attribute assertions
  await expect(addToCart).toHaveAttribute('data-product-id', '12345');
  await expect(addToCart).toHaveClass('btn-primary');
  await expect(addToCart).toHaveClass(/btn-/); // regex for class
  await expect(addToCart).toHaveId('add-to-cart');
  
  // Value assertions (for inputs)
  const searchInput = page.locator('#search');
  await expect(searchInput).toHaveValue('laptops');
  await expect(searchInput).toHaveValue(/lap/i);
  
  // Count assertions
  await expect(page.locator('.product-card')).toHaveCount(12);
  
  // CSS assertions
  await expect(addToCart).toHaveCSS('background-color', 'rgb(0, 123, 255)');
});
```

### 3. Page-Level Assertions

```typescript
test('page assertions demo', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Title assertions
  await expect(page).toHaveTitle('Welcome to Example');
  await expect(page).toHaveTitle(/Example/);
  
  // URL assertions
  await expect(page).toHaveURL('https://example.com/');
  await expect(page).toHaveURL(/example\.com/);
  
  // Screenshot comparison (visual testing)
  await expect(page).toHaveScreenshot('homepage.png');
  await expect(page.locator('.header')).toHaveScreenshot('header.png');
});
```

### 4. Negation with .not

```typescript
test('negation assertions', async ({ page }) => {
  await page.goto('https://example.com');
  
  const errorMessage = page.locator('.error-message');
  
  // Negate any assertion
  await expect(errorMessage).not.toBeVisible();
  await expect(errorMessage).not.toHaveText('Error occurred');
  await expect(page.locator('.loading')).not.toBeAttached();
});
```

### 5. Real-World Scenario: E-commerce Cart Validation

```typescript
test('validate shopping cart with comprehensive assertions', async ({ page }) => {
  // Navigate to e-commerce site
  await page.goto('https://shop.example.com');
  
  // Add product to cart
  await page.locator('[data-product-id="12345"]').click();
  
  // Cart icon should update
  const cartBadge = page.locator('.cart-badge');
  await expect(cartBadge).toBeVisible();
  await expect(cartBadge).toHaveText('1');
  await expect(cartBadge).toHaveCSS('background-color', 'rgb(220, 53, 69)');
  
  // Navigate to cart page
  await page.locator('#cart-link').click();
  await expect(page).toHaveURL(/\/cart/);
  await expect(page).toHaveTitle(/Shopping Cart/);
  
  // Validate cart items
  const cartItems = page.locator('.cart-item');
  await expect(cartItems).toHaveCount(1);
  
  // Validate first item details
  const firstItem = cartItems.first();
  await expect(firstItem.locator('.product-name')).toHaveText('Laptop Pro 15');
  await expect(firstItem.locator('.product-price')).toHaveText('$1,299.99');
  await expect(firstItem.locator('.quantity-input')).toHaveValue('1');
  
  // Validate total
  const subtotal = page.locator('.subtotal');
  await expect(subtotal).toContainText('$1,299.99');
  
  // Checkout button should be enabled
  const checkoutBtn = page.locator('#checkout-btn');
  await expect(checkoutBtn).toBeEnabled();
  await expect(checkoutBtn).not.toHaveClass(/disabled/);
  
  // Remove button should be visible
  await expect(firstItem.locator('.remove-btn')).toBeVisible();
  
  // Promo code section
  const promoInput = page.locator('#promo-code');
  await expect(promoInput).toBeEditable();
  await expect(promoInput).toHaveAttribute('placeholder', 'Enter promo code');
});
```

### 6. Soft Assertions (Continue Testing on Failure)

```typescript
test('validate product details with soft assertions', async ({ page }) => {
  await page.goto('https://shop.example.com/product/12345');
  
  // All these will be checked even if some fail
  await expect.soft(page.locator('.product-title')).toBeVisible();
  await expect.soft(page.locator('.product-price')).toContainText('$');
  await expect.soft(page.locator('.product-image')).toHaveAttribute('alt');
  await expect.soft(page.locator('.add-to-cart')).toBeEnabled();
  await expect.soft(page.locator('.product-rating')).toHaveCount(1);
  await expect.soft(page.locator('.stock-status')).toHaveText('In Stock');
  
  // Test continues even if assertions above failed
  // Final summary shows all failures
});
```

### 7. Custom Timeout for Assertions

```typescript
test('assertions with custom timeout', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Wait up to 10 seconds instead of default 5
  await expect(page.locator('.slow-loading-element')).toBeVisible({ timeout: 10000 });
  
  // Wait up to 30 seconds for API response
  await expect(page.locator('.data-loaded')).toBeVisible({ timeout: 30000 });
});
```

### 8. Polling Assertions for Custom Conditions

```typescript
test('polling assertions for custom logic', async ({ page }) => {
  await page.goto('https://example.com/dashboard');
  
  // Poll a custom function until condition is met
  await expect.poll(async () => {
    const count = await page.locator('.notification').count();
    return count;
  }).toBe(3);
  
  // Poll with custom interval and timeout
  await expect.poll(async () => {
    const text = await page.locator('.status').textContent();
    return text;
  }, {
    intervals: [500, 1000, 2000], // Custom retry intervals
    timeout: 15000
  }).toBe('Completed');
  
  // Complex polling example
  await expect.poll(async () => {
    const items = await page.locator('.list-item').all();
    const texts = await Promise.all(items.map(item => item.textContent()));
    return texts.filter(t => t?.includes('Active')).length;
  }).toBeGreaterThan(5);
});
```

### 9. Real-World Scenario: Multi-User Chat Application

```typescript
test('validate real-time chat with assertions', async ({ page }) => {
  await page.goto('https://chat.example.com');
  
  // Login
  await page.locator('#username').fill('user1');
  await page.locator('#login-btn').click();
  
  // Wait for chat to load
  await expect(page.locator('.chat-container')).toBeVisible();
  await expect(page.locator('.user-status')).toHaveText('Online');
  
  // Send message
  const messageInput = page.locator('#message-input');
  await messageInput.fill('Hello everyone!');
  await page.locator('#send-btn').click();
  
  // Verify message appears
  await expect(page.locator('.message').last()).toContainText('Hello everyone!');
  await expect(page.locator('.message').last()).toHaveAttribute('data-sender', 'user1');
  
  // Poll for message count to increase (real-time updates)
  const initialCount = await page.locator('.message').count();
  await expect.poll(async () => {
    return await page.locator('.message').count();
  }).toBeGreaterThan(initialCount);
  
  // Typing indicator
  await messageInput.focus();
  await messageInput.fill('Typing...');
  await expect(page.locator('.typing-indicator')).toBeVisible();
  
  // Clear and indicator disappears
  await messageInput.clear();
  await expect(page.locator('.typing-indicator')).not.toBeVisible();
  
  // Check online users count
  await expect(page.locator('.online-users .user-avatar')).toHaveCount(5);
});
```

### 10. Best Practices

✅ **GOOD: Use web-first assertions with auto-wait**
```typescript
await expect(page.locator('#button')).toBeVisible();
```

❌ **BAD: Manual waits + generic assertions**
```typescript
await page.waitForSelector('#button');
const isVisible = await page.locator('#button').isVisible();
expect(isVisible).toBe(true);
```

✅ **GOOD: Use specific text matchers**
```typescript
await expect(page.locator('.error')).toHaveText('Invalid email');
```

❌ **BAD: Get text then compare**
```typescript
const text = await page.locator('.error').textContent();
expect(text).toBe('Invalid email');
```

### 11. Common Pitfalls

1. **Using generic expect() for locators**
   ```typescript
   // ❌ No auto-wait
   const count = await page.locator('.item').count();
   expect(count).toBe(5);
   
   // ✅ Auto-waits for elements
   await expect(page.locator('.item')).toHaveCount(5);
   ```

2. **Not using soft assertions for multiple checks**
   ```typescript
   // ❌ Stops at first failure
   await expect(locator1).toBeVisible();
   await expect(locator2).toBeVisible();
   await expect(locator3).toBeVisible();
   
   // ✅ Checks all and reports all failures
   await expect.soft(locator1).toBeVisible();
   await expect.soft(locator2).toBeVisible();
   await expect.soft(locator3).toBeVisible();
   ```

### Related Topics
- **Q10**: Wait strategies and auto-waiting
- **Q5**: Selectors for finding elements to assert
- **Q47**: Best practices for writing assertions

---

## Q10: How do you implement wait strategies in Playwright?

### Answer:

Playwright has **built-in auto-waiting** for most actions, but sometimes you need explicit wait strategies for complex scenarios. Unlike Selenium, Playwright performs automatic actionability checks before every action.

### 1. Auto-Waiting (Built-in)

Playwright automatically waits for elements to be actionable before performing actions:

| Action | Auto-Wait Checks |
|--------|------------------|
| `click()` | Visible, Stable, Enabled, Not obscured |
| `fill()` | Visible, Enabled, Editable |
| `check()` | Visible, Stable, Enabled |
| `select()` | Visible, Enabled |

```typescript
import { test, expect } from '@playwright/test';

test('auto-waiting demonstration', async ({ page }) => {
  await page.goto('https://example.com');
  
  // No explicit wait needed - Playwright waits automatically
  await page.locator('#submit-btn').click(); // Waits for visible + enabled + stable
  await page.locator('#email').fill('test@example.com'); // Waits for visible + editable
  await page.locator('#agree').check(); // Waits for visible + enabled
});
```

### 2. Wait Strategies Comparison

| Strategy | Use Case | Example |
|----------|----------|---------|
| **Auto-wait** | Default for actions | `await page.click('#btn')` |
| **waitForSelector** | Wait for element | `await page.waitForSelector('.item')` |
| **waitForLoadState** | Page loading | `await page.waitForLoadState('networkidle')` |
| **waitForURL** | URL navigation | `await page.waitForURL('**/dashboard')` |
| **waitForResponse** | API call | `await page.waitForResponse('**/api/data')` |
| **waitForFunction** | Custom condition | `await page.waitForFunction(() => window.loaded)` |
| **waitForTimeout** | Fixed delay (⚠️ avoid) | `await page.waitForTimeout(5000)` |

### 3. Wait for Element States

```typescript
test('wait for element states', async ({ page }) => {
  await page.goto('https://example.com/dashboard');
  
  // Wait for element to be visible
  await page.locator('.loading-spinner').waitFor({ state: 'visible' });
  await page.locator('.loading-spinner').waitFor({ state: 'hidden' });
  
  // Wait for element to be attached to DOM
  await page.locator('.dynamic-content').waitFor({ state: 'attached' });
  
  // Wait for element to be detached from DOM
  await page.locator('.modal').waitFor({ state: 'detached' });
  
  // With custom timeout
  await page.locator('.slow-element').waitFor({ 
    state: 'visible', 
    timeout: 30000 
  });
});
```

### 4. Wait for Page Load States

```typescript
test('wait for page load states', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Wait for 'load' event (default)
  await page.waitForLoadState('load');
  
  // Wait for 'domcontentloaded' (DOM ready, but images/stylesheets may still load)
  await page.waitForLoadState('domcontentloaded');
  
  // Wait for network to be idle (no network connections for 500ms)
  await page.waitForLoadState('networkidle');
  
  // Wait after navigation
  await page.locator('#link').click();
  await page.waitForLoadState('networkidle');
});
```

### 5. Wait for URL Changes

```typescript
test('wait for URL changes', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Wait for specific URL
  await page.locator('#dashboard-link').click();
  await page.waitForURL('https://example.com/dashboard');
  
  // Wait for URL pattern
  await page.waitForURL('**/dashboard/**');
  await page.waitForURL(/dashboard\/\d+/);
  
  // Wait for URL with custom timeout
  await page.waitForURL('**/profile', { timeout: 10000 });
  
  // Wait for URL to not be something
  await page.waitForURL(url => !url.pathname.includes('/loading'));
});
```

### 6. Wait for Network Requests/Responses

```typescript
test('wait for network activity', async ({ page }) => {
  await page.goto('https://example.com/dashboard');
  
  // Wait for specific request
  const requestPromise = page.waitForRequest('**/api/users');
  await page.locator('#refresh-btn').click();
  await requestPromise;
  
  // Wait for response
  const responsePromise = page.waitForResponse('**/api/products');
  await page.locator('#load-products').click();
  const response = await responsePromise;
  console.log(response.status()); // 200
  
  // Wait for response with validation
  const validResponsePromise = page.waitForResponse(
    response => response.url().includes('/api/data') && response.status() === 200
  );
  await page.locator('#fetch-btn').click();
  await validResponsePromise;
  
  // Wait for multiple responses
  const [response1, response2] = await Promise.all([
    page.waitForResponse('**/api/users'),
    page.waitForResponse('**/api/products'),
    page.locator('#load-all').click()
  ]);
});
```

### 7. Wait for Custom Conditions (waitForFunction)

```typescript
test('wait for custom JavaScript conditions', async ({ page }) => {
  await page.goto('https://example.com/charts');
  
  // Wait for window property
  await page.waitForFunction(() => window.dataLoaded === true);
  
  // Wait for element count
  await page.waitForFunction(() => {
    return document.querySelectorAll('.chart').length === 5;
  });
  
  // Wait with parameters
  await page.waitForFunction(
    (minCount) => document.querySelectorAll('.item').length >= minCount,
    10 // parameter
  );
  
  // Wait for complex condition
  await page.waitForFunction(() => {
    const loadingBar = document.querySelector('.loading-bar');
    return loadingBar && loadingBar.style.width === '100%';
  });
  
  // Return value from waitForFunction
  const cartCount = await page.waitForFunction(() => {
    const badge = document.querySelector('.cart-badge');
    return badge ? parseInt(badge.textContent) : 0;
  });
  console.log(await cartCount.jsonValue()); // 3
});
```

### 8. Real-World Scenario: E-commerce Product Search with Loading

```typescript
test('e-commerce product search with wait strategies', async ({ page }) => {
  await page.goto('https://shop.example.com');
  
  // Wait for page to be fully loaded
  await page.waitForLoadState('networkidle');
  
  // Search for products
  await page.locator('#search-input').fill('laptop');
  
  // Wait for search API call
  const searchResponse = page.waitForResponse(
    response => response.url().includes('/api/search') && response.status() === 200
  );
  await page.locator('#search-btn').click();
  await searchResponse;
  
  // Wait for loading spinner to appear then disappear
  const spinner = page.locator('.loading-spinner');
  await spinner.waitFor({ state: 'visible' });
  await spinner.waitFor({ state: 'hidden', timeout: 10000 });
  
  // Wait for products to render
  await page.locator('.product-card').first().waitFor({ state: 'visible' });
  
  // Wait for all product images to load
  await page.waitForFunction(() => {
    const images = document.querySelectorAll('.product-card img');
    return Array.from(images).every(img => img.complete);
  });
  
  // Wait for product count to be displayed
  await expect(page.locator('.results-count')).toBeVisible();
  
  // Verify results loaded
  await expect(page.locator('.product-card')).toHaveCount(12);
  
  // Click on first product and wait for navigation
  await page.locator('.product-card').first().click();
  await page.waitForURL('**/product/**');
  await page.waitForLoadState('domcontentloaded');
  
  // Wait for product details API
  await page.waitForResponse('**/api/product/**');
  
  // Wait for add-to-cart button to be enabled
  await page.locator('#add-to-cart').waitFor({ state: 'visible' });
  await expect(page.locator('#add-to-cart')).toBeEnabled();
});
```

### 9. Real-World Scenario: Real-Time Dashboard with Polling

```typescript
test('real-time dashboard with periodic updates', async ({ page }) => {
  await page.goto('https://dashboard.example.com');
  
  // Login
  await page.locator('#username').fill('admin');
  await page.locator('#password').fill('password');
  
  // Wait for login API and redirect
  const [loginResponse] = await Promise.all([
    page.waitForResponse('**/api/auth/login'),
    page.locator('#login-btn').click()
  ]);
  
  await page.waitForURL('**/dashboard');
  
  // Wait for initial data load
  await page.waitForResponse('**/api/metrics');
  
  // Wait for WebSocket connection
  await page.waitForFunction(() => {
    return window.websocket && window.websocket.readyState === 1; // OPEN
  });
  
  // Wait for charts to render
  await page.waitForFunction(() => {
    const charts = document.querySelectorAll('.chart-container canvas');
    return charts.length >= 4;
  });
  
  // Store initial metric value
  const initialValue = await page.locator('.metric-value').first().textContent();
  
  // Wait for metric to update (polling)
  await page.waitForFunction(
    (oldValue) => {
      const currentValue = document.querySelector('.metric-value')?.textContent;
      return currentValue !== oldValue;
    },
    initialValue,
    { timeout: 60000 } // Wait up to 60 seconds for update
  );
  
  // Wait for notification to appear
  const notification = page.locator('.notification');
  await notification.waitFor({ state: 'visible', timeout: 30000 });
  await expect(notification).toContainText('New data available');
  
  // Wait for notification to auto-dismiss
  await notification.waitFor({ state: 'hidden', timeout: 5000 });
});
```

### 10. Combining Multiple Wait Strategies

```typescript
test('complex multi-step form with various waits', async ({ page }) => {
  await page.goto('https://forms.example.com/application');
  
  // Step 1: Personal Information
  await page.locator('#firstName').fill('John');
  await page.locator('#lastName').fill('Doe');
  
  // Wait for validation API
  const validationResponse = page.waitForResponse('**/api/validate-name');
  await page.locator('#lastName').blur();
  await validationResponse;
  
  await page.locator('#next-step-1').click();
  
  // Wait for next step to load
  await page.locator('#step-2').waitFor({ state: 'visible' });
  await page.waitForFunction(() => {
    return document.querySelector('#step-1')?.classList.contains('completed');
  });
  
  // Step 2: Address (with autocomplete)
  const addressInput = page.locator('#address');
  await addressInput.fill('123 Main');
  
  // Wait for autocomplete suggestions
  await page.waitForResponse('**/api/address/autocomplete');
  await page.locator('.autocomplete-item').first().waitFor({ state: 'visible' });
  
  // Select first suggestion
  await page.locator('.autocomplete-item').first().click();
  
  // Wait for address to be populated
  await expect(page.locator('#city')).not.toHaveValue('');
  await expect(page.locator('#state')).not.toHaveValue('');
  
  await page.locator('#next-step-2').click();
  
  // Step 3: File Upload
  await page.locator('#step-3').waitFor({ state: 'visible' });
  
  const fileInput = page.locator('#document-upload');
  await fileInput.setInputFiles('path/to/document.pdf');
  
  // Wait for upload to complete
  const uploadResponse = page.waitForResponse(
    response => response.url().includes('/api/upload') && response.status() === 200
  );
  await uploadResponse;
  
  // Wait for upload progress bar to reach 100%
  await page.waitForFunction(() => {
    const progress = document.querySelector('.upload-progress');
    return progress && progress.getAttribute('data-progress') === '100';
  });
  
  // Wait for success checkmark
  await page.locator('.upload-success-icon').waitFor({ state: 'visible' });
  
  // Submit form
  const [submitResponse] = await Promise.all([
    page.waitForResponse('**/api/submit'),
    page.locator('#submit-form').click()
  ]);
  
  // Wait for success page
  await page.waitForURL('**/success');
  await expect(page.locator('.success-message')).toBeVisible();
});
```

### 11. Best Practices

✅ **GOOD: Rely on auto-waiting**
```typescript
await page.locator('#button').click(); // Auto-waits for actionability
```

❌ **BAD: Unnecessary explicit waits**
```typescript
await page.waitForSelector('#button');
await page.locator('#button').click();
```

✅ **GOOD: Wait for API responses when needed**
```typescript
const [response] = await Promise.all([
  page.waitForResponse('**/api/data'),
  page.locator('#load').click()
]);
```

❌ **BAD: Fixed timeout waits**
```typescript
await page.locator('#load').click();
await page.waitForTimeout(5000); // Brittle and slow
```

### 12. Common Pitfalls

1. **Using waitForTimeout instead of condition-based waits**
   ```typescript
   // ❌ Brittle and slow
   await page.waitForTimeout(3000);
   
   // ✅ Waits exactly as long as needed
   await page.locator('.loaded').waitFor({ state: 'visible' });
   ```

2. **Not waiting for network activity**
   ```typescript
   // ❌ May assert before data loads
   await page.locator('#fetch').click();
   await expect(page.locator('.data')).toHaveCount(10);
   
   // ✅ Wait for API response
   await Promise.all([
     page.waitForResponse('**/api/data'),
     page.locator('#fetch').click()
   ]);
   await expect(page.locator('.data')).toHaveCount(10);
   ```

3. **Over-using waitForLoadState('networkidle')**
   ```typescript
   // ❌ Slow, waits for ALL network to be idle
   await page.waitForLoadState('networkidle');
   
   // ✅ Wait for specific resource you need
   await page.waitForResponse('**/api/critical-data');
   ```

### 13. Timeout Configuration

```typescript
test('configure timeouts', async ({ page }) => {
  // Per-action timeout
  await page.locator('#slow-button').click({ timeout: 30000 });
  
  // Per-assertion timeout
  await expect(page.locator('.result')).toBeVisible({ timeout: 10000 });
  
  // Default navigation timeout (in playwright.config.ts)
  // navigationTimeout: 30000
  
  // Default action timeout (in playwright.config.ts)
  // actionTimeout: 10000
  
  // Default expect timeout (in playwright.config.ts)
  // expect: { timeout: 5000 }
});
```

### Related Topics
- **Q9**: Assertions with auto-waiting
- **Q4**: Navigation and page loading
- **Q13**: Waiting for API responses

---

*Document Version: 1.0*  
*Last Updated: November 2025*
