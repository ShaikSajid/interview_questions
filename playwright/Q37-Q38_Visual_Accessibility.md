# Playwright Interview Questions: Q37-Q38

## Q37: How do you perform visual testing in Playwright?

### Answer:

Playwright has built-in visual comparison testing for detecting UI regressions.

### 1. Visual Regression Tests

```typescript
import { test, expect } from '@playwright/test';

test('homepage visual test', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Compare with baseline screenshot
  await expect(page).toHaveScreenshot('homepage.png');
});

test('element visual test', async ({ page }) => {
  await page.goto('https://example.com');
  
  await expect(page.locator('.product-card').first()).toHaveScreenshot('product.png');
});

test('masked visual test', async ({ page }) => {
  await page.goto('https://dashboard.example.com');
  
  // Mask dynamic content
  await expect(page).toHaveScreenshot('dashboard.png', {
    mask: [page.locator('.timestamp'), page.locator('.live-data')],
    maxDiffPixels: 100
  });
});
```

### 2. Generate Baselines

```bash
# Generate baseline screenshots
npx playwright test --update-snapshots

# Update specific test
npx playwright test visual.spec.ts --update-snapshots
```

### Related Topics
- **Q17**: Screenshots
- **Q38**: Accessibility testing

---

## Q38: How do you test accessibility with Playwright?

### Answer:

Playwright integrates with accessibility testing tools like axe-core.

### 1. Axe Integration

```typescript
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('homepage accessibility', async ({ page }) => {
  await page.goto('https://example.com');
  
  const accessibilityScanResults = await new AxeBuilder({ page }).analyze();
  
  expect(accessibilityScanResults.violations).toEqual([]);
});

test('specific element accessibility', async ({ page }) => {
  await page.goto('https://example.com');
  
  const results = await new AxeBuilder({ page })
    .include('.form-container')
    .analyze();
  
  expect(results.violations).toEqual([]);
});
```

### 2. ARIA Testing

```typescript
test('check ARIA labels', async ({ page }) => {
  await page.goto('https://example.com/form');
  
  // Use role selectors
  await page.getByRole('button', { name: 'Submit' }).click();
  await page.getByRole('textbox', { name: 'Email' }).fill('test@example.com');
  
  // Verify ARIA
  await expect(page.locator('[aria-label="Close"]')).toBeVisible();
});
```

### Related Topics
- **Q37**: Visual testing
- **Q5**: Role selectors

---

*Document Version: 1.0*  
*Last Updated: November 2025*
