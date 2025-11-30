# Playwright Interview Questions: Q23-Q24

## Q23: How do you handle dialogs and alerts?

### Answer:

Playwright automatically dismisses dialogs. Use event listeners to handle them explicitly.

### 1. Dialog Types

```typescript
import { test, expect } from '@playwright/test';

test('handle alert', async ({ page }) => {
  page.on('dialog', dialog => {
    console.log(dialog.message());
    expect(dialog.type()).toBe('alert');
    dialog.accept();
  });
  
  await page.goto('https://example.com');
  await page.locator('#trigger-alert').click();
});

test('handle confirm', async ({ page }) => {
  page.on('dialog', async dialog => {
    expect(dialog.type()).toBe('confirm');
    expect(dialog.message()).toBe('Are you sure?');
    await dialog.accept(); // or dialog.dismiss()
  });
  
  await page.goto('https://example.com');
  await page.locator('#delete-btn').click();
});

test('handle prompt', async ({ page }) => {
  page.on('dialog', async dialog => {
    expect(dialog.type()).toBe('prompt');
    await dialog.accept('User Input');
  });
  
  await page.goto('https://example.com');
  await page.locator('#prompt-btn').click();
});
```

### 2. Real-World Scenario

```typescript
test('delete confirmation flow', async ({ page }) => {
  await page.goto('https://app.example.com/users');
  
  page.once('dialog', dialog => {
    expect(dialog.message()).toContain('Delete user');
    dialog.accept();
  });
  
  await page.locator('[data-user-id="123"] .delete').click();
  await expect(page.locator('.success-message')).toBeVisible();
});
```

### Related Topics
- **Q24**: Popup windows

---

## Q24: How do you handle browser popups and windows?

### Answer:

Playwright provides context management for handling popup windows and new tabs.

### 1. Popup Handling

```typescript
test('handle popup window', async ({ page, context }) => {
  await page.goto('https://example.com');
  
  // Wait for popup
  const [popup] = await Promise.all([
    context.waitForEvent('page'),
    page.locator('#open-popup').click()
  ]);
  
  // Interact with popup
  await popup.waitForLoadState();
  await expect(popup.locator('h1')).toHaveText('Popup Content');
  await popup.locator('#close').click();
});

test('multiple popups', async ({ page, context }) => {
  await page.goto('https://example.com');
  
  const popups = [];
  context.on('page', page => {
    popups.push(page);
  });
  
  // Open multiple popups
  await page.locator('#open1').click();
  await page.locator('#open2').click();
  
  expect(popups).toHaveLength(2);
  
  // Close all popups
  for (const popup of popups) {
    await popup.close();
  }
});
```

### 2. Real-World Scenario: OAuth Login

```typescript
test('OAuth popup login', async ({ page, context }) => {
  await page.goto('https://app.example.com/login');
  
  const [oauthPopup] = await Promise.all([
    context.waitForEvent('page'),
    page.locator('#login-with-google').click()
  ]);
  
  // Fill OAuth form in popup
  await oauthPopup.waitForLoadState();
  await oauthPopup.locator('#email').fill('user@gmail.com');
  await oauthPopup.locator('#password').fill('password');
  await oauthPopup.locator('#signin').click();
  
  // Wait for redirect back to main page
  await page.waitForURL('**/dashboard');
  await expect(page.locator('.user-name')).toBeVisible();
});
```

### Related Topics
- **Q23**: Dialogs and alerts
- **Q3**: Browser contexts

---

*Document Version: 1.0*  
*Last Updated: November 2025*
