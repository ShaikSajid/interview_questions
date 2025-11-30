# Playwright Interview Questions: Q29-Q30

## Q29: How do you run tests in parallel?

### Answer:

Playwright runs tests in parallel by default across multiple workers for faster execution.

### 1. Parallel Configuration

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  workers: process.env.CI ? 2 : undefined, // CI: 2 workers, local: CPU cores
  fullyParallel: true, // Run tests in same file in parallel
});
```

### 2. Serial Tests

```typescript
import { test } from '@playwright/test';

test.describe.serial('login flow', () => {
  // These run in order
  test('step 1', async ({ page }) => { });
  test('step 2', async ({ page }) => { });
  test('step 3', async ({ page }) => { });
});
```

### 3. Worker Index

```typescript
test('use worker index', async ({ page }, testInfo) => {
  const workerIndex = testInfo.workerIndex;
  console.log('Worker:', workerIndex);
  
  // Use different test data per worker
  const testUser = `user${workerIndex}@example.com`;
});
```

### Related Topics
- **Q30**: Test retry
- **Q32**: Test organization

---

## Q30: How do you implement test retry logic?

### Answer:

Playwright automatically retries failed tests to handle flakiness.

### 1. Retry Configuration

```typescript
// playwright.config.ts
export default defineConfig({
  retries: process.env.CI ? 2 : 0, // Retry twice in CI
});
```

### 2. Per-Test Retry

```typescript
import { test } from '@playwright/test';

test('flaky test', async ({ page }) => {
  test.slow(); // Triple timeout
  test.setTimeout(60000); // 60 seconds
  
  await page.goto('https://example.com');
});

// Override retries
test.describe(() => {
  test.describe.configure({ retries: 3 });
  
  test('retry 3 times', async ({ page }) => {
    // This test retries up to 3 times
  });
});
```

### 3. Retry Info

```typescript
test('check retry attempt', async ({ page }, testInfo) => {
  console.log('Retry:', testInfo.retry);
  
  if (testInfo.retry === 0) {
    // First attempt
  } else {
    // Retry attempt
  }
});
```

### Related Topics
- **Q29**: Parallel execution
- **Q41**: Debugging

---

*Document Version: 1.0*  
*Last Updated: November 2025*
