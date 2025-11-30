# Playwright Interview Questions: Q17-Q18

## Q17: How do you capture screenshots in Playwright?

### Answer:

Playwright provides powerful screenshot capabilities for visual testing, debugging, and documentation. Screenshots can be taken of full pages, specific elements, or viewport areas.

### 1. Screenshot Methods

| Method | Captures | Use Case | Example |
|--------|----------|----------|--------|
| **page.screenshot()** | Full page or viewport | Full page capture | `await page.screenshot({ path: 'page.png' })` |
| **locator.screenshot()** | Specific element | Element capture | `await locator.screenshot({ path: 'el.png' })` |
| **fullPage: true** | Entire scrollable page | Long pages | `{ fullPage: true }` |
| **clip** | Custom region | Specific area | `{ clip: { x, y, width, height } }` |

### 2. Basic Screenshots

```typescript
import { test, expect } from '@playwright/test';

test('capture page screenshot', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Save to file
  await page.screenshot({ path: 'screenshots/homepage.png' });
  
  // Get as buffer
  const buffer = await page.screenshot();
  console.log(buffer.length);
  
  // Full page screenshot (scrolls automatically)
  await page.screenshot({ 
    path: 'screenshots/fullpage.png',
    fullPage: true 
  });
  
  // Viewport only
  await page.screenshot({ 
    path: 'screenshots/viewport.png',
    fullPage: false 
  });
});

test('capture element screenshot', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Screenshot of specific element
  await page.locator('.header').screenshot({ 
    path: 'screenshots/header.png' 
  });
  
  // Product card
  await page.locator('[data-product-id="123"]').screenshot({ 
    path: 'screenshots/product.png' 
  });
});
```

### 3. Screenshot Options

```typescript
test('screenshot with options', async ({ page }) => {
  await page.goto('https://example.com');
  
  await page.screenshot({
    path: 'screenshot.png',
    type: 'png', // or 'jpeg'
    quality: 90, // 0-100 for jpeg
    fullPage: true,
    omitBackground: true, // Transparent background
    animations: 'disabled', // Disable CSS animations
    scale: 'css', // or 'device' for device pixel ratio
    timeout: 30000 // Wait up to 30s
  });
});

test('clip specific region', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Capture specific coordinates
  await page.screenshot({
    path: 'region.png',
    clip: {
      x: 100,
      y: 200,
      width: 500,
      height: 300
    }
  });
});
```

### 4. Visual Comparison Testing

```typescript
test('visual regression test', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Compare with baseline
  await expect(page).toHaveScreenshot('homepage.png');
  
  // With threshold
  await expect(page).toHaveScreenshot('homepage.png', {
    maxDiffPixels: 100 // Allow 100 pixels difference
  });
  
  // Element comparison
  await expect(page.locator('.product-card')).toHaveScreenshot('product.png');
});

test('masked visual test', async ({ page }) => {
  await page.goto('https://example.com/dashboard');
  
  // Mask dynamic elements
  await expect(page).toHaveScreenshot('dashboard.png', {
    mask: [page.locator('.timestamp'), page.locator('.live-data')],
    animations: 'disabled'
  });
});
```

### 5. Real-World Scenario: E-commerce Visual Testing

```typescript
test('product page visual testing', async ({ page }) => {
  await page.goto('https://shop.example.com/product/123');
  
  // Wait for images to load
  await page.waitForLoadState('networkidle');
  
  // Main product image
  await expect(page.locator('.product-main-image')).toHaveScreenshot(
    'product-main.png',
    { animations: 'disabled' }
  );
  
  // Thumbnail gallery
  await expect(page.locator('.thumbnail-gallery')).toHaveScreenshot(
    'thumbnails.png'
  );
  
  // Pricing section (mask dynamic discount)
  await expect(page.locator('.pricing')).toHaveScreenshot('pricing.png', {
    mask: [page.locator('.countdown-timer')]
  });
  
  // Reviews section
  await page.locator('#reviews-tab').click();
  await expect(page.locator('.reviews-section')).toHaveScreenshot(
    'reviews.png',
    { mask: [page.locator('.review-date')] }
  );
});
```

### Related Topics
- **Q18**: Video recording
- **Q40**: Trace viewer for debugging

---

## Q18: How do you record videos of test execution?

### Answer:

Playwright can record videos of test execution for debugging, documentation, and CI/CD failure analysis. Videos are automatically captured when configured.

### 1. Video Recording Configuration

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  use: {
    video: 'on', // Always record
    // video: 'off', // Never record
    // video: 'retain-on-failure', // Only keep failed tests
    // video: 'on-first-retry', // Record on retry
    
    videoSize: { width: 1280, height: 720 },
  },
});
```

### 2. Accessing Video Files

```typescript
test('get video path', async ({ page }, testInfo) => {
  await page.goto('https://example.com');
  await page.locator('#button').click();
  
  // Video path available after test
});

test.afterEach(async ({ page }, testInfo) => {
  const videoPath = await page.video()?.path();
  console.log('Video saved at:', videoPath);
  
  // Attach video to test report
  if (videoPath) {
    await testInfo.attach('video', { 
      path: videoPath, 
      contentType: 'video/webm' 
    });
  }
});
```

### 3. Programmatic Video Control

```typescript
test('control video recording', async ({ browser }) => {
  const context = await browser.newContext({
    recordVideo: {
      dir: 'videos/',
      size: { width: 1920, height: 1080 }
    }
  });
  
  const page = await context.newPage();
  await page.goto('https://example.com');
  
  // Perform actions
  await page.locator('#button').click();
  
  // Close to save video
  await context.close();
  
  const videoPath = await page.video()?.path();
  console.log('Video saved:', videoPath);
});
```

### 4. CI/CD Video Configuration

```typescript
export default defineConfig({
  use: {
    video: process.env.CI ? 'retain-on-failure' : 'off',
  },
  
  projects: [
    {
      name: 'chromium',
      use: { 
        ...devices['Desktop Chrome'],
        video: 'retain-on-failure'
      },
    },
  ],
});
```

### Related Topics
- **Q17**: Screenshots
- **Q40**: Trace viewer
- **Q35**: CI/CD integration

---

*Document Version: 1.0*  
*Last Updated: November 2025*
