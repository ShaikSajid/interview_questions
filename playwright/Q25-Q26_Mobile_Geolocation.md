# Playwright Interview Questions: Q25-Q26

## Q25: How do you emulate mobile devices in Playwright?

### Answer:

Playwright includes 100+ device profiles for mobile testing with proper viewport, user agent, and touch emulation.

### 1. Mobile Emulation

```typescript
import { test, expect, devices } from '@playwright/test';

test('test on iPhone', async ({ browser }) => {
  const context = await browser.newContext({
    ...devices['iPhone 13']
  });
  
  const page = await context.newPage();
  await page.goto('https://example.com');
  
  // Mobile-specific assertions
  await expect(page.locator('.mobile-menu')).toBeVisible();
  
  await context.close();
});

test('test on multiple devices', async ({ browser }) => {
  const deviceNames = ['iPhone 13', 'Pixel 5', 'iPad Pro'];
  
  for (const deviceName of deviceNames) {
    const context = await browser.newContext({
      ...devices[deviceName]
    });
    
    const page = await context.newPage();
    await page.goto('https://example.com');
    
    await expect(page.locator('h1')).toBeVisible();
    await context.close();
  }
});
```

### 2. Custom Viewport

```typescript
test('custom mobile viewport', async ({ browser }) => {
  const context = await browser.newContext({
    viewport: { width: 375, height: 667 },
    userAgent: 'Mozilla/5.0 (iPhone; CPU iPhone OS 15_0 like Mac OS X)',
    isMobile: true,
    hasTouch: true,
    deviceScaleFactor: 2
  });
  
  const page = await context.newPage();
  await page.goto('https://example.com');
});
```

### 3. Touch Gestures

```typescript
test('mobile touch interactions', async ({ browser }) => {
  const context = await browser.newContext({
    ...devices['iPhone 13']
  });
  const page = await context.newPage();
  
  await page.goto('https://example.com/gallery');
  
  // Tap
  await page.locator('.image').first().tap();
  
  // Swipe (scroll)
  await page.touchscreen.tap(100, 300);
  await page.mouse.down();
  await page.mouse.move(100, 100);
  await page.mouse.up();
  
  await context.close();
});
```

### Related Topics
- **Q26**: Geolocation and permissions

---

## Q26: How do you handle geolocation and permissions?

### Answer:

Playwright allows setting geolocation and granting permissions for notifications, camera, microphone, etc.

### 1. Geolocation

```typescript
import { test, expect } from '@playwright/test';

test('set geolocation', async ({ browser }) => {
  const context = await browser.newContext({
    geolocation: { latitude: 40.7128, longitude: -74.0060 }, // New York
    permissions: ['geolocation']
  });
  
  const page = await context.newPage();
  await page.goto('https://maps.example.com');
  
  await page.locator('#find-me').click();
  await expect(page.locator('.location')).toContainText('New York');
  
  await context.close();
});

test('change geolocation', async ({ browser }) => {
  const context = await browser.newContext({
    geolocation: { latitude: 40.7128, longitude: -74.0060 },
    permissions: ['geolocation']
  });
  
  const page = await context.newPage();
  await page.goto('https://example.com');
  
  // Change location
  await context.setGeolocation({ latitude: 51.5074, longitude: -0.1278 }); // London
  await page.reload();
  
  await expect(page.locator('.location')).toContainText('London');
  
  await context.close();
});
```

### 2. Permissions

```typescript
test('grant permissions', async ({ browser }) => {
  const context = await browser.newContext({
    permissions: ['camera', 'microphone', 'notifications']
  });
  
  const page = await context.newPage();
  await page.goto('https://video-call.example.com');
  
  // No permission prompt - auto-granted
  await page.locator('#start-call').click();
  await expect(page.locator('.camera-active')).toBeVisible();
  
  await context.close();
});

test('deny permissions', async ({ browser }) => {
  const context = await browser.newContext();
  // Don't grant permissions
  
  const page = await context.newPage();
  await page.goto('https://example.com');
  
  await page.locator('#enable-notifications').click();
  await expect(page.locator('.permission-denied')).toBeVisible();
  
  await context.close();
});
```

### 3. Real-World Scenario

```typescript
test('location-based search', async ({ browser }) => {
  const context = await browser.newContext({
    geolocation: { latitude: 37.7749, longitude: -122.4194 }, // San Francisco
    permissions: ['geolocation']
  });
  
  const page = await context.newPage();
  await page.goto('https://restaurant-finder.example.com');
  
  await page.locator('#use-my-location').click();
  
  // Should show San Francisco restaurants
  await expect(page.locator('.location-name')).toContainText('San Francisco');
  await expect(page.locator('.restaurant-card')).toHaveCount(10);
  
  await context.close();
});
```

### Related Topics
- **Q25**: Mobile device emulation
- **Q3**: Browser contexts

---

*Document Version: 1.0*  
*Last Updated: November 2025*
