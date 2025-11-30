# Playwright Interview Questions: Q7-Q8

## Q7: How do you handle different types of click actions in Playwright?

### Answer:

Playwright provides multiple click methods and options to handle various clicking scenarios including regular clicks, double clicks, right clicks, and handling special conditions.

### Basic Click Operations:

```typescript
import { test, expect } from '@playwright/test';

test('different click methods', async ({ page }) => {
  await page.goto('https://example.com');
  
  // 1. Regular click
  await page.click('button#submit');
  
  // 2. Double click
  await page.dblclick('.editable-text');
  
  // 3. Right click (context menu)
  await page.click('text=File', { button: 'right' });
  
  // 4. Click with modifier keys
  await page.click('a[href="/page"]', { modifiers: ['Control'] }); // Ctrl+Click
  await page.click('a[href="/page"]', { modifiers: ['Shift'] }); // Shift+Click
  await page.click('a[href="/page"]', { modifiers: ['Meta'] }); // Cmd+Click (Mac)
  
  // 5. Click at specific position
  await page.click('canvas', { position: { x: 100, y: 200 } });
  
  // 6. Force click (bypass actionability checks)
  await page.click('button', { force: true });
  
  // 7. Click without waiting for navigation
  await page.click('a[href="/page"]', { noWaitAfter: true });
});
```

### Click Options:

| Option | Description | Example |
|--------|-------------|---------|
| `button` | Which button to click | `'left'`, `'right'`, `'middle'` |
| `clickCount` | Number of clicks | `1`, `2`, `3` |
| `modifiers` | Keyboard modifiers | `['Control', 'Shift']` |
| `position` | Click position | `{ x: 10, y: 20 }` |
| `force` | Skip actionability checks | `true` |
| `noWaitAfter` | Don't wait after click | `true` |
| `timeout` | Maximum time to wait | `30000` |

### Real-World Scenario: E-commerce Product Interactions

```typescript
import { test, expect } from '@playwright/test';

test('product interactions with various clicks', async ({ page }) => {
  await page.goto('https://shop.example.com/products');
  
  // 1. Regular click to view product
  await page.click('[data-testid="product-1"]');
  
  // 2. Click image to zoom
  await page.click('.product-image');
  await expect(page.locator('.zoom-modal')).toBeVisible();
  
  // 3. Close zoom modal (click outside)
  await page.click('.zoom-modal-overlay');
  await expect(page.locator('.zoom-modal')).toBeHidden();
  
  // 4. Open product in new tab (Ctrl+Click)
  const [newPage] = await Promise.all([
    page.waitForEvent('popup'),
    page.click('a:has-text("Similar Products")', { modifiers: ['Control'] })
  ]);
  await newPage.close();
  
  // 5. Right-click on image to see options
  await page.click('.product-image', { button: 'right' });
  await expect(page.locator('.context-menu')).toBeVisible();
  
  // 6. Click color variant
  await page.click('[data-color="blue"]');
  await expect(page.locator('.selected-color')).toHaveText('Blue');
  
  // 7. Click size (with auto-wait for element to be ready)
  await page.click('button:has-text("Large")');
  
  // 8. Increase quantity (multiple clicks)
  await page.click('[data-testid="quantity-increase"]', { clickCount: 3 });
  await expect(page.locator('#quantity')).toHaveValue('4');
  
  // 9. Add to cart
  await page.click('button:has-text("Add to Cart")');
  await expect(page.locator('.cart-notification')).toBeVisible();
  
  // 10. Click to view cart (navigates to new page)
  await page.click('a:has-text("View Cart")');
  await expect(page).toHaveURL(/.*cart/);
});
```

### Handling Hidden/Covered Elements:

```typescript
import { test, expect } from '@playwright/test';

test('clicking hidden or covered elements', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Element covered by cookie banner
  // ❌ BAD: Will fail with "element is not visible"
  // await page.click('#hidden-button');
  
  // ✅ GOOD: Close cookie banner first
  await page.click('#close-cookies');
  await page.click('#hidden-button');
  
  // Element in collapsed accordion
  // Expand accordion first
  await page.click('.accordion-header');
  await page.click('.accordion-content button');
  
  // Element requires scrolling
  await page.click('button#footer-subscribe'); // Auto-scrolls
  
  // Force click (use sparingly!)
  await page.click('#covered-element', { force: true });
});
```

### Scenario: Interactive Canvas Drawing

```typescript
import { test, expect } from '@playwright/test';

test('canvas drawing with precise clicks', async ({ page }) => {
  await page.goto('https://draw.example.com');
  
  const canvas = page.locator('canvas#drawing-board');
  
  // Get canvas bounding box
  const box = await canvas.boundingBox();
  if (!box) throw new Error('Canvas not found');
  
  // Draw a square by clicking at specific positions
  const points = [
    { x: box.x + 100, y: box.y + 100 }, // Top-left
    { x: box.x + 200, y: box.y + 100 }, // Top-right
    { x: box.x + 200, y: box.y + 200 }, // Bottom-right
    { x: box.x + 100, y: box.y + 200 }, // Bottom-left
    { x: box.x + 100, y: box.y + 100 }, // Close square
  ];
  
  for (const point of points) {
    await page.mouse.click(point.x, point.y);
    await page.waitForTimeout(100); // Brief pause between clicks
  }
  
  // Verify drawing was created
  await expect(page.locator('.drawing-stats')).toContainText('5 points');
  
  // Double-click to finish
  await canvas.dblclick();
  await expect(page.locator('.shape-complete')).toBeVisible();
});
```

### Click with Waiting Strategies:

```typescript
import { test, expect } from '@playwright/test';

test('click with different wait strategies', async ({ page }) => {
  await page.goto('https://example.com');
  
  // 1. Click and wait for navigation
  await Promise.all([
    page.waitForNavigation(),
    page.click('a:has-text("Next Page")')
  ]);
  
  // 2. Click and wait for element to appear
  await page.click('button:has-text("Load More")');
  await page.waitForSelector('.new-items');
  
  // 3. Click and wait for API response
  await Promise.all([
    page.waitForResponse(resp => resp.url().includes('/api/data')),
    page.click('button:has-text("Refresh")')
  ]);
  
  // 4. Click and wait for element state
  await page.click('button:has-text("Delete")');
  await expect(page.locator('.confirm-dialog')).toBeVisible();
  
  // 5. Click without waiting (fire and forget)
  await page.click('button:has-text("Track Event"))', { noWaitAfter: true });
  // Continue immediately without waiting
});
```

### Scenario: Complex Dropdown Navigation

```typescript
import { test, expect } from '@playwright/test';

test('nested dropdown menu interactions', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Hover to open main menu
  await page.hover('nav a:has-text("Products")');
  await expect(page.locator('.dropdown-menu')).toBeVisible();
  
  // Hover over submenu item
  await page.hover('.dropdown-menu a:has-text("Electronics")');
  await expect(page.locator('.submenu')).toBeVisible();
  
  // Click final item
  await page.click('.submenu a:has-text("Laptops")');
  await expect(page).toHaveURL(/.*laptops/);
  
  // Right-click on product (context menu)
  await page.click('.product-card:first-child', { button: 'right' });
  await expect(page.locator('.context-menu')).toBeVisible();
  
  // Click "Open in new tab"
  const [newPage] = await Promise.all([
    page.waitForEvent('popup'),
    page.click('text=Open in new tab')
  ]);
  
  await expect(newPage).toHaveURL(/.*product/);
  await newPage.close();
});
```

### Mouse Operations (Low-Level):

```typescript
import { test, expect } from '@playwright/test';

test('low-level mouse operations', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Move mouse to element
  await page.mouse.move(100, 200);
  
  // Press mouse button
  await page.mouse.down();
  
  // Move while pressed (drag)
  await page.mouse.move(300, 400);
  
  // Release mouse button
  await page.mouse.up();
  
  // Double click at position
  await page.mouse.dblclick(150, 250);
  
  // Wheel scroll
  await page.mouse.wheel(0, 100); // Scroll down 100px
});
```

### Best Practices:

✅ **Use high-level click()** instead of mouse operations  
✅ **Let Playwright auto-wait** - don't force clicks unnecessarily  
✅ **Use role-based selectors** for better accessibility  
✅ **Handle overlays** (close banners/modals) before clicking  
✅ **Test modifier keys** (Ctrl+Click) for power users  

```typescript
// ✅ GOOD: Let Playwright handle waiting
await page.click('button:has-text("Submit")');

// ❌ BAD: Manual waiting before click
await page.waitForSelector('button', { state: 'visible' });
await page.waitForTimeout(1000);
await page.click('button', { force: true });
```

---

## Q8: How do you fill forms and handle different input types in Playwright?

### Answer:

Playwright provides methods to interact with various form elements including text inputs, checkboxes, radio buttons, select dropdowns, file uploads, and more.

### Basic Form Filling:

```typescript
import { test, expect } from '@playwright/test';

test('basic form filling', async ({ page }) => {
  await page.goto('https://example.com/form');
  
  // 1. Text input
  await page.fill('#name', 'John Doe');
  
  // 2. Type with delay (simulate real typing)
  await page.type('#email', 'john@example.com', { delay: 100 });
  
  // 3. Clear and fill
  await page.fill('#address', ''); // Clear
  await page.fill('#address', '123 Main St');
  
  // 4. Press keys
  await page.press('#search', 'Enter');
  await page.press('#code', 'Control+A'); // Select all
  
  // 5. Checkbox
  await page.check('#agree'); // Check
  await page.uncheck('#newsletter'); // Uncheck
  
  // 6. Radio button
  await page.check('#gender-male');
  
  // 7. Select dropdown
  await page.selectOption('#country', 'US'); // By value
  await page.selectOption('#country', { label: 'United States' }); // By label
  
  // 8. Multiple select
  await page.selectOption('#languages', ['en', 'es', 'fr']);
});
```

### Form Input Types:

| Input Type | Method | Example |
|------------|--------|---------|
| Text/Email/Password | `fill()`, `type()` | `await page.fill('#email', 'test@example.com')` |
| Checkbox | `check()`, `uncheck()` | `await page.check('#agree')` |
| Radio | `check()` | `await page.check('#option-a')` |
| Select | `selectOption()` | `await page.selectOption('#country', 'US')` |
| File | `setInputFiles()` | `await page.setInputFiles('#file', 'path/to/file.pdf')` |
| Textarea | `fill()` | `await page.fill('textarea', 'Long text...')` |
| Date | `fill()` | `await page.fill('#date', '2025-12-31')` |
| Range | `fill()` | `await page.fill('#slider', '75')` |

### Real-World Scenario: Complete Registration Form

```typescript
import { test, expect } from '@playwright/test';

test('user registration form', async ({ page }) => {
  await page.goto('https://example.com/register');
  
  // Personal Information
  await page.fill('#firstName', 'John');
  await page.fill('#lastName', 'Doe');
  await page.fill('#email', 'john.doe@example.com');
  
  // Password with strength check
  const passwordInput = page.locator('#password');
  await passwordInput.fill('WeakPass');
  await expect(page.locator('.password-strength')).toHaveText('Weak');
  
  await passwordInput.fill('StrongP@ssw0rd!');
  await expect(page.locator('.password-strength')).toHaveText('Strong');
  
  await page.fill('#confirmPassword', 'StrongP@ssw0rd!');
  
  // Date of Birth
  await page.fill('#birthdate', '1990-01-15');
  
  // Phone number (with formatting)
  await page.fill('#phone', '5551234567');
  await expect(page.locator('#phone')).toHaveValue('(555) 123-4567');
  
  // Address
  await page.fill('#address', '123 Main Street');
  await page.fill('#city', 'New York');
  await page.selectOption('#state', 'NY');
  await page.fill('#zipcode', '10001');
  await page.selectOption('#country', 'United States');
  
  // Gender (radio buttons)
  await page.check('#gender-male');
  
  // Interests (checkboxes)
  await page.check('#interest-technology');
  await page.check('#interest-sports');
  await page.check('#interest-music');
  
  // Preferred contact method
  await page.selectOption('#contact-method', 'email');
  
  // Profile picture upload
  await page.setInputFiles('#profile-picture', './test-data/avatar.jpg');
  await expect(page.locator('.file-name')).toContainText('avatar.jpg');
  
  // Bio (textarea)
  await page.fill('textarea#bio', 'Software engineer with 10 years of experience...');
  
  // Newsletter preferences (multiple select)
  await page.selectOption('#newsletters', ['weekly', 'monthly']);
  
  // Terms and conditions
  await page.check('#terms');
  
  // Verify submit button is enabled
  await expect(page.locator('button[type="submit"]')).toBeEnabled();
  
  // Submit form
  await page.click('button[type="submit"]');
  
  // Verify success
  await expect(page.locator('.success-message')).toBeVisible();
  await expect(page.locator('.success-message')).toContainText('Registration successful');
});
```

### Handling Special Input Types:

```typescript
import { test, expect } from '@playwright/test';

test('special input types', async ({ page }) => {
  await page.goto('https://example.com/advanced-form');
  
  // 1. Date picker
  await page.fill('input[type="date"]', '2025-12-25');
  
  // 2. Time picker
  await page.fill('input[type="time"]', '14:30');
  
  // 3. Datetime-local
  await page.fill('input[type="datetime-local"]', '2025-12-25T14:30');
  
  // 4. Month picker
  await page.fill('input[type="month"]', '2025-12');
  
  // 5. Week picker
  await page.fill('input[type="week"]', '2025-W52');
  
  // 6. Color picker
  await page.fill('input[type="color"]', '#ff0000');
  
  // 7. Range slider
  await page.fill('input[type="range"]', '75');
  await expect(page.locator('.range-value')).toHaveText('75');
  
  // 8. Number input
  await page.fill('input[type="number"]', '42');
  
  // 9. Email input (with validation)
  await page.fill('input[type="email"]', 'test@example.com');
  
  // 10. URL input
  await page.fill('input[type="url"]', 'https://example.com');
  
  // 11. Tel input
  await page.fill('input[type="tel"]', '+1-555-123-4567');
  
  // 12. Search input
  await page.fill('input[type="search"]', 'playwright testing');
  await page.press('input[type="search"]', 'Enter');
});
```

### File Upload Scenarios:

```typescript
import { test, expect } from '@playwright/test';

test('file upload scenarios', async ({ page }) => {
  await page.goto('https://example.com/upload');
  
  // 1. Single file upload
  await page.setInputFiles('#single-file', './test-data/document.pdf');
  
  // 2. Multiple files upload
  await page.setInputFiles('#multiple-files', [
    './test-data/file1.pdf',
    './test-data/file2.jpg',
    './test-data/file3.doc'
  ]);
  
  // 3. Upload from buffer
  await page.setInputFiles('#buffer-file', {
    name: 'test.txt',
    mimeType: 'text/plain',
    buffer: Buffer.from('File content here')
  });
  
  // 4. Clear file input
  await page.setInputFiles('#file-input', []);
  
  // 5. Drag and drop file upload
  const fileInput = page.locator('#drag-drop-area input[type="file"]');
  await fileInput.setInputFiles('./test-data/image.png');
  await expect(page.locator('.uploaded-file')).toContainText('image.png');
  
  // 6. Verify file size
  await expect(page.locator('.file-size')).toContainText('1.2 MB');
});
```

### Scenario: Dynamic Form with Conditional Fields

```typescript
import { test, expect } from '@playwright/test';

test('dynamic form with conditional fields', async ({ page }) => {
  await page.goto('https://example.com/dynamic-form');
  
  // Select account type
  await page.selectOption('#account-type', 'business');
  
  // Business-specific fields appear
  await expect(page.locator('#company-name')).toBeVisible();
  await expect(page.locator('#tax-id')).toBeVisible();
  
  // Fill business fields
  await page.fill('#company-name', 'Acme Corp');
  await page.fill('#tax-id', '12-3456789');
  
  // Number of employees triggers additional fields
  await page.selectOption('#employee-count', '50-100');
  await expect(page.locator('#hr-contact')).toBeVisible();
  await page.fill('#hr-contact', 'hr@acme.com');
  
  // Toggle additional services
  await page.check('#service-support');
  await expect(page.locator('#support-level')).toBeVisible();
  await page.selectOption('#support-level', 'premium');
  
  // Premium support shows phone field
  await expect(page.locator('#support-phone')).toBeVisible();
  await page.fill('#support-phone', '555-0100');
  
  // Submit and verify
  await page.click('button:has-text("Submit")');
  await expect(page.locator('.confirmation')).toContainText('Business Account Created');
});
```

### Form Validation Testing:

```typescript
import { test, expect } from '@playwright/test';

test('form validation checks', async ({ page }) => {
  await page.goto('https://example.com/contact');
  
  // Submit empty form
  await page.click('button[type="submit"]');
  
  // Verify validation errors
  await expect(page.locator('#name-error')).toHaveText('Name is required');
  await expect(page.locator('#email-error')).toHaveText('Email is required');
  
  // Fill with invalid data
  await page.fill('#name', 'Jo'); // Too short
  await page.fill('#email', 'invalid-email'); // Invalid format
  
  await page.click('button[type="submit"]');
  
  // Verify specific error messages
  await expect(page.locator('#name-error')).toHaveText('Name must be at least 3 characters');
  await expect(page.locator('#email-error')).toHaveText('Please enter a valid email');
  
  // Fix errors
  await page.fill('#name', 'John Doe');
  await page.fill('#email', 'john@example.com');
  await page.fill('#message', 'Test message');
  
  // Verify errors cleared
  await expect(page.locator('#name-error')).toBeHidden();
  await expect(page.locator('#email-error')).toBeHidden();
  
  // Submit valid form
  await page.click('button[type="submit"]');
  await expect(page.locator('.success-message')).toBeVisible();
});
```

### Autocomplete/Typeahead:

```typescript
import { test, expect } from '@playwright/test';

test('autocomplete field interaction', async ({ page }) => {
  await page.goto('https://example.com');
  
  const searchInput = page.locator('#search-autocomplete');
  
  // Start typing
  await searchInput.type('play', { delay: 100 });
  
  // Wait for suggestions
  await page.waitForSelector('.autocomplete-suggestions');
  
  // Verify suggestions appeared
  await expect(page.locator('.autocomplete-suggestion')).toHaveCount(5);
  
  // Click specific suggestion
  await page.click('.autocomplete-suggestion:has-text("Playwright Testing")');
  
  // Verify selection
  await expect(searchInput).toHaveValue('Playwright Testing');
  
  // Navigate with keyboard
  await searchInput.fill('');
  await searchInput.type('test', { delay: 100 });
  await page.keyboard.press('ArrowDown'); // First suggestion
  await page.keyboard.press('ArrowDown'); // Second suggestion
  await page.keyboard.press('Enter'); // Select
  
  await expect(searchInput).toHaveValue(/test/i);
});
```

### Best Practices:

✅ **Use fill() instead of type()** for speed (unless testing typing animation)  
✅ **Wait for form validation** messages to appear  
✅ **Test both valid and invalid inputs**  
✅ **Verify field states** (enabled/disabled) before interacting  
✅ **Test conditional fields** (show/hide based on selections)  
✅ **Handle file uploads** with proper MIME types  

```typescript
// ✅ GOOD: Fast and reliable
await page.fill('#email', 'user@example.com');
await expect(page.locator('#email')).toHaveValue('user@example.com');

// ❌ BAD: Slower, no added benefit
await page.click('#email');
await page.keyboard.type('user@example.com');
```

---

**Related Topics:**
- Q9-Q10: Assertions and wait strategies
- Q21-Q22: File uploads and downloads
- Q23-Q24: Dialogs and alerts

---

*Document Version: 1.0*  
*Last Updated: November 2025*
