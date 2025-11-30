# Playwright Interview Questions: Q47-Q48

## Q47: What are Playwright best practices?

### Answer:

Follow these best practices for maintainable, reliable tests.

### 1. Selectors

✅ **GOOD**: Use data-testid or role selectors
```typescript
await page.locator('[data-testid="submit-button"]').click();
await page.getByRole('button', { name: 'Submit' }).click();
```

❌ **BAD**: Use CSS classes or XPath
```typescript
await page.locator('.btn.btn-primary.submit').click();
```

### 2. Authentication

✅ **GOOD**: Reuse storage state
```typescript
use: { storageState: 'auth.json' }
```

❌ **BAD**: Login in every test
```typescript
test.beforeEach(async () => { /* login */ });
```

### 3. Test Independence

✅ **GOOD**: Isolated tests
```typescript
test('independent test', async ({ page }) => {
  // Setup own data
  // Test
  // Cleanup
});
```

❌ **BAD**: Tests depend on order
```typescript
test('create user', async () => {}); // Creates user
test('delete user', async () => {}); // Depends on previous test
```

### 4. Waits

✅ **GOOD**: Use auto-waiting
```typescript
await page.locator('#button').click();
```

❌ **BAD**: Fixed timeouts
```typescript
await page.waitForTimeout(3000);
```

### 5. Page Objects

✅ **GOOD**: Encapsulate pages
```typescript
const loginPage = new LoginPage(page);
await loginPage.login(email, password);
```

❌ **BAD**: Duplicate selectors
```typescript
await page.locator('#email').fill(email);
await page.locator('#email').fill(email); // Repeated
```

### Related Topics
- **Q48**: Error handling
- **Q31**: Page Object Model

---

## Q48: How do you implement error handling and recovery?

### Answer:

Proper error handling makes tests more robust and provides better debugging information.

### 1. Try-Catch

```typescript
test('handle errors', async ({ page }) => {
  try {
    await page.goto('https://example.com');
    await page.locator('#button').click();
  } catch (error) {
    console.error('Test failed:', error);
    await page.screenshot({ path: 'error.png' });
    throw error; // Re-throw to fail test
  }
});
```

### 2. Soft Assertions

```typescript
test('multiple checks', async ({ page }) => {
  await page.goto('https://example.com');
  
  await expect.soft(page.locator('.header')).toBeVisible();
  await expect.soft(page.locator('.footer')).toBeVisible();
  await expect.soft(page.locator('.content')).toBeVisible();
  // All assertions run even if some fail
});
```

### 3. Retry on Failure

```typescript
async function retryAction(action: () => Promise<void>, retries = 3) {
  for (let i = 0; i < retries; i++) {
    try {
      await action();
      return;
    } catch (error) {
      if (i === retries - 1) throw error;
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }
}

test('retry example', async ({ page }) => {
  await retryAction(async () => {
    await page.locator('#flaky-element').click();
  });
});
```

### Related Topics
- **Q30**: Test retry configuration
- **Q41**: Debugging

---

*Document Version: 1.0*  
*Last Updated: November 2025*
