# Playwright Interview Questions: Q19-Q20

## Q19: How do you handle iframes in Playwright?

### Answer:

Playwright provides seamless iframe handling with automatic frame detection and navigation. Unlike Selenium, you don't need to explicitly switch contexts.

### 1. Iframe Methods

| Method | Purpose | Example |
|--------|---------|--------|
| **page.frameLocator()** | Locate iframe (recommended) | `page.frameLocator('iframe')` |
| **page.frame()** | Get frame by name/URL | `page.frame({ name: 'myframe' })` |
| **page.frames()** | Get all frames | `page.frames()` |
| **locator.contentFrame()** | Get frame from locator | `await locator.contentFrame()` |

### 2. Basic Iframe Interaction

```typescript
import { test, expect } from '@playwright/test';

test('interact with iframe', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Method 1: frameLocator (recommended)
  const frame = page.frameLocator('iframe#myframe');
  await frame.locator('#button').click();
  await expect(frame.locator('.result')).toHaveText('Success');
  
  // Method 2: frame() by name
  const namedFrame = page.frame({ name: 'myframe' });
  if (namedFrame) {
    await namedFrame.locator('#input').fill('test');
  }
  
  // Method 3: frame() by URL
  const urlFrame = page.frame({ url: /.*embed.*/  });
  if (urlFrame) {
    await urlFrame.locator('button').click();
  }
});
```

### 3. Nested Iframes

```typescript
test('nested iframes', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Access nested iframe
  const outerFrame = page.frameLocator('#outer-frame');
  const innerFrame = outerFrame.frameLocator('#inner-frame');
  
  await innerFrame.locator('#deep-button').click();
  await expect(innerFrame.locator('.message')).toBeVisible();
});
```

### 4. Real-World Scenario: Payment Gateway

```typescript
test('payment iframe integration', async ({ page }) => {
  await page.goto('https://shop.example.com/checkout');
  
  // Fill customer details
  await page.locator('#name').fill('John Doe');
  await page.locator('#email').fill('john@example.com');
  
  // Switch to payment iframe
  const paymentFrame = page.frameLocator('iframe[name="stripe-iframe"]');
  
  // Fill card details inside iframe
  await paymentFrame.locator('#card-number').fill('4242424242424242');
  await paymentFrame.locator('#expiry').fill('12/25');
  await paymentFrame.locator('#cvv').fill('123');
  
  // Back to main page
  await page.locator('#submit-payment').click();
  
  await expect(page.locator('.success-message')).toBeVisible();
});
```

### 5. Dynamic Iframes

```typescript
test('wait for iframe to load', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Click button that loads iframe
  await page.locator('#load-widget').click();
  
  // Wait for iframe to appear
  await page.locator('iframe#widget').waitFor({ state: 'attached' });
  
  const frame = page.frameLocator('iframe#widget');
  await expect(frame.locator('.widget-content')).toBeVisible();
});
```

### Related Topics
- **Q20**: Shadow DOM
- **Q11**: Advanced selectors

---

## Q20: How do you work with Shadow DOM?

### Answer:

Playwright automatically pierces Shadow DOM, making it seamless to interact with web components. No special API needed—selectors work across shadow boundaries.

### 1. Shadow DOM Support

```typescript
test('shadow DOM interaction', async ({ page }) => {
  await page.goto('https://example.com/web-components');
  
  // Playwright automatically pierces shadow DOM
  await page.locator('custom-button').click();
  
  // Deep selector through multiple shadow roots
  await page.locator('parent-component >> child-component >> button').click();
  
  // Text selector works through shadow DOM
  await page.locator('text=Click Me').click();
});
```

### 2. Real-World Scenario: Web Components

```typescript
test('interact with custom elements', async ({ page }) => {
  await page.goto('https://app.example.com');
  
  // Custom dropdown component with shadow DOM
  const dropdown = page.locator('custom-dropdown');
  await dropdown.locator('.trigger').click();
  await dropdown.locator('.option[value="option2"]').click();
  
  // Custom date picker
  await page.locator('date-picker >> input').fill('2025-12-31');
  
  // Verify selection
  await expect(dropdown.locator('.selected-value')).toHaveText('Option 2');
});
```

### 3. Multiple Shadow Roots

```typescript
test('nested shadow DOM', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Navigate through multiple shadow roots
  await page.locator(
    'app-root >> navigation-menu >> menu-item[label="Settings"]'
  ).click();
  
  // Complex component tree
  await page.locator(
    'dashboard >> widget-container >> chart-component >> .data-point'
  ).first().click();
});
```

### Related Topics
- **Q19**: Iframes
- **Q11**: Advanced selectors

---

*Document Version: 1.0*  
*Last Updated: November 2025*
