# Playwright Interview Questions: Q27-Q28

## Q27: How do you intercept and mock API requests?

### Answer:

Playwright's route() method enables comprehensive API mocking for testing without backend dependencies.

### 1. API Mocking

```typescript
import { test, expect } from '@playwright/test';

test('mock API response', async ({ page }) => {
  await page.route('**/api/users', route => {
    route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify([
        { id: 1, name: 'Mock User 1' },
        { id: 2, name: 'Mock User 2' }
      ])
    });
  });
  
  await page.goto('https://example.com/users');
  await expect(page.locator('.user-name').first()).toHaveText('Mock User 1');
});

test('mock with different responses', async ({ page }) => {
  let callCount = 0;
  
  await page.route('**/api/data', route => {
    callCount++;
    if (callCount === 1) {
      route.fulfill({ body: JSON.stringify({ data: 'first' }) });
    } else {
      route.fulfill({ body: JSON.stringify({ data: 'updated' }) });
    }
  });
  
  await page.goto('https://example.com');
  await page.locator('#refresh').click();
});
```

### Related Topics
- **Q14**: Network interception
- **Q28**: Offline testing

---

## Q28: How do you test offline scenarios?

### Answer:

Playwright can simulate offline mode to test Progressive Web Apps and offline functionality.

### 1. Offline Mode

```typescript
import { test, expect } from '@playwright/test';

test('test offline mode', async ({ browser }) => {
  const context = await browser.newContext({
    offline: true
  });
  
  const page = await context.newPage();
  await page.goto('https://example.com').catch(() => {});
  
  // Should show offline page
  await expect(page.locator('.offline-message')).toBeVisible();
  
  await context.close();
});

test('toggle offline mode', async ({ page, context }) => {
  await page.goto('https://pwa-example.com');
  
  // Go offline
  await context.setOffline(true);
  await page.reload();
  
  // Check cached content still works
  await expect(page.locator('.content')).toBeVisible();
  
  // Go back online
  await context.setOffline(false);
  await page.reload();
});
```

### 2. Service Worker Testing

```typescript
test('service worker caching', async ({ page, context }) => {
  // Load page online
  await page.goto('https://pwa.example.com');
  await page.waitForLoadState('networkidle');
  
  // Wait for service worker
  await page.waitForFunction(() => {
    return navigator.serviceWorker.controller !== null;
  });
  
  // Go offline
  await context.setOffline(true);
  
  // Navigate - should work from cache
  await page.goto('https://pwa.example.com/page2');
  await expect(page.locator('h1')).toBeVisible();
});
```

### Related Topics
- **Q27**: API mocking
- **Q14**: Network control

---

*Document Version: 1.0*  
*Last Updated: November 2025*
