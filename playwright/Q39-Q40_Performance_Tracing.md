# Playwright Interview Questions: Q39-Q40

## Q39: How do you measure performance with Playwright?

### Answer:

Playwright can capture performance metrics and analyze page load times.

### 1. Performance Metrics

```typescript
import { test, expect } from '@playwright/test';

test('measure page performance', async ({ page }) => {
  await page.goto('https://example.com');
  
  const metrics = await page.evaluate(() => {
    const perfData = performance.getEntriesByType('navigation')[0] as PerformanceNavigationTiming;
    return {
      domContentLoaded: perfData.domContentLoadedEventEnd - perfData.domContentLoadedEventStart,
      loadComplete: perfData.loadEventEnd - perfData.loadEventStart,
      domInteractive: perfData.domInteractive - perfData.fetchStart,
      firstPaint: performance.getEntriesByType('paint')[0]?.startTime
    };
  });
  
  console.log('Performance:', metrics);
  expect(metrics.loadComplete).toBeLessThan(3000); // Under 3s
});
```

### 2. Web Vitals

```typescript
test('measure Core Web Vitals', async ({ page }) => {
  await page.goto('https://example.com');
  
  const vitals = await page.evaluate(() => {
    return new Promise(resolve => {
      new PerformanceObserver(list => {
        const entries = list.getEntries();
        const lcp = entries.find(e => e.entryType === 'largest-contentful-paint');
        resolve({ LCP: lcp?.startTime });
      }).observe({ entryTypes: ['largest-contentful-paint'] });
    });
  });
  
  console.log('LCP:', vitals);
});
```

### Related Topics
- **Q40**: Trace viewer

---

## Q40: How do you use Playwright Trace Viewer?

### Answer:

Trace Viewer provides detailed execution logs, screenshots, and network activity for debugging.

### 1. Enable Tracing

```typescript
// playwright.config.ts
export default defineConfig({
  use: {
    trace: 'on-first-retry', // or 'on', 'off', 'retain-on-failure'
  },
});

// Programmatically
test('with tracing', async ({ browser }) => {
  const context = await browser.newContext();
  await context.tracing.start({ screenshots: true, snapshots: true });
  
  const page = await context.newPage();
  await page.goto('https://example.com');
  
  await context.tracing.stop({ path: 'trace.zip' });
});
```

### 2. View Traces

```bash
# Open trace viewer
npx playwright show-trace trace.zip

# After test run
npx playwright show-report
```

### Related Topics
- **Q41**: Debugging
- **Q39**: Performance testing

---

*Document Version: 1.0*  
*Last Updated: November 2025*
