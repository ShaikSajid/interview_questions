# Playwright Interview Questions: Q5-Q6

## Q5: What are the different types of selectors in Playwright and when should you use each?

### Answer:

Playwright supports multiple selector strategies, each optimized for different scenarios. Choosing the right selector improves test reliability and maintainability.

### Selector Types Comparison:

| Selector Type | Syntax | Best For | Reliability |
|--------------|---------|----------|-------------|
| **CSS** | `#id`, `.class`, `[attr]` | Standard elements | Medium |
| **Text** | `text=Login`, `"Login"` | Visible text | High |
| **Role** | `role=button[name="Submit"]` | Accessibility | Very High |
| **Test ID** | `data-testid=submit-btn` | Stable testing | Very High |
| **XPath** | `//div[@class="item"]` | Complex DOM | Low |
| **nth-match** | `:nth-match(:text("Item"), 3)` | Multiple matches | Medium |
| **Has** | `article:has(h2:text("News"))` | Parent selection | High |

### 1. CSS Selectors

```typescript
import { test, expect } from '@playwright/test';

test('CSS selectors', async ({ page }) => {
  await page.goto('https://example.com');
  
  // ID selector
  await page.click('#login-button');
  
  // Class selector
  await page.click('.submit-btn');
  
  // Attribute selector
  await page.click('[data-test="submit"]');
  await page.click('[type="submit"]');
  await page.click('[aria-label="Close"]');
  
  // Descendant selector
  await page.click('form .submit-button');
  
  // Child selector
  await page.click('div > button');
  
  // Multiple classes
  await page.click('.btn.btn-primary');
  
  // Pseudo-class
  await page.click('button:first-child');
  await page.click('li:nth-child(2)');
  await page.click('input:disabled');
});
```

### 2. Text Selectors (Most Reliable)

```typescript
import { test, expect } from '@playwright/test';

test('text selectors', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Exact text match
  await page.click('text=Login');
  await page.click('text="Sign Up"');
  
  // Partial text match (case-insensitive)
  await page.click('text=log'); // Matches "Login", "Logout", "Dialog"
  
  // Case-sensitive exact match
  await page.click('text=/^Login$/');
  
  // Text in specific element
  await page.click('button:has-text("Submit")');
  
  // Button with exact text
  await page.click('button:text-is("Login")');
});
```

### 3. Role-Based Selectors (Accessibility)

```typescript
import { test, expect } from '@playwright/test';

test('role-based selectors', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Button role
  await page.click('role=button[name="Submit"]');
  await page.click('role=button[name=/submit/i]'); // Case-insensitive
  
  // Link role
  await page.click('role=link[name="Learn More"]');
  
  // Textbox role
  await page.fill('role=textbox[name="Email"]', 'user@example.com');
  
  // Checkbox role
  await page.check('role=checkbox[name="I agree"]');
  
  // Common roles
  await page.click('role=heading[name="Welcome"]');
  await page.click('role=img[name="Logo"]');
  await page.selectOption('role=combobox[name="Country"]', 'US');
  
  // With level
  await expect(page.locator('role=heading[level=1]')).toHaveText('Home');
});
```

### 4. Data-TestID Selectors (Best Practice)

```typescript
import { test, expect } from '@playwright/test';

test('data-testid selectors', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Using data-testid
  await page.click('[data-testid="login-button"]');
  
  // Playwright's shorthand
  await page.click('data-testid=login-button');
  
  // Custom test ID attribute (configure in playwright.config.ts)
  await page.click('[data-test="submit"]');
});

// HTML structure:
// <button data-testid="login-button">Login</button>
// <input data-testid="email-input" type="email" />
// <div data-testid="error-message" class="error">Invalid email</div>
```

### Real-World Scenario: E-commerce Product Search

```typescript
import { test, expect } from '@playwright/test';

test('product search using different selectors', async ({ page }) => {
  await page.goto('https://shop.example.com');
  
  // 1. Search using role (accessibility-friendly)
  await page.fill('role=searchbox[name="Search products"]', 'laptop');
  await page.click('role=button[name="Search"]');
  
  // 2. Wait for results using test ID
  await expect(page.locator('[data-testid="search-results"]')).toBeVisible();
  
  // 3. Click product using text content
  await page.click('text="MacBook Pro 16-inch"');
  
  // 4. Add to cart using CSS + text combination
  await page.click('button.add-to-cart:has-text("Add to Cart")');
  
  // 5. Verify cart badge using attribute selector
  await expect(page.locator('[data-testid="cart-badge"]')).toHaveText('1');
  
  // 6. Navigate to cart using role-based link
  await page.click('role=link[name="Cart"]');
  
  // 7. Verify product in cart using complex selector
  const cartItem = page.locator('[data-testid="cart-item"]:has-text("MacBook Pro")');
  await expect(cartItem).toBeVisible();
  
  // 8. Update quantity using nth-child
  await page.fill('[data-testid="cart-item"]:nth-child(1) input[type="number"]', '2');
  
  // 9. Proceed to checkout using button text
  await page.click('button:text-is("Proceed to Checkout")');
  
  // 10. Verify URL change
  await expect(page).toHaveURL(/.*checkout/);
});
```

### 5. Chaining Selectors

```typescript
import { test, expect } from '@playwright/test';

test('chaining selectors', async ({ page }) => {
  await page.goto('https://example.com');
  
  // CSS + text
  await page.click('button:has-text("Login")');
  
  // Role + attribute
  await page.click('role=button[name="Submit"][disabled]');
  
  // Complex chaining
  await page.click('form#login button.primary:has-text("Sign In")');
  
  // Parent-child relationship
  await page.click('[data-testid="product-card"]:has-text("Laptop") button:has-text("Buy")');
});
```

### 6. XPath Selectors (Use Sparingly)

```typescript
import { test, expect } from '@playwright/test';

test('xpath selectors', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Basic XPath
  await page.click('//button[@id="submit"]');
  
  // XPath with text
  await page.click('//button[contains(text(), "Login")]');
  
  // XPath with multiple conditions
  await page.click('//button[@class="btn" and @type="submit"]');
  
  // Parent-child with XPath
  await page.click('//form[@id="login"]//button');
  
  // Following sibling
  await page.click('//label[text()="Email"]/following-sibling::input');
});
```

### Scenario: Complex Form with Multiple Selector Strategies

```typescript
import { test, expect } from '@playwright/test';

test('complete registration form', async ({ page }) => {
  await page.goto('https://example.com/register');
  
  // Personal Information
  await page.fill('role=textbox[name="First Name"]', 'John');
  await page.fill('role=textbox[name="Last Name"]', 'Doe');
  await page.fill('[data-testid="email-input"]', 'john.doe@example.com');
  
  // Password
  await page.fill('input[type="password"][name="password"]', 'SecurePass123!');
  await page.fill('input[type="password"][name="confirmPassword"]', 'SecurePass123!');
  
  // Date of birth (using CSS)
  await page.selectOption('select[name="month"]', 'January');
  await page.selectOption('select[name="day"]', '15');
  await page.selectOption('select[name="year"]', '1990');
  
  // Gender (radio button using role)
  await page.check('role=radio[name="Male"]');
  
  // Country (using label text)
  await page.selectOption('role=combobox[name="Country"]', 'United States');
  
  // Terms and conditions
  await page.check('role=checkbox[name="I agree to the terms"]');
  
  // Newsletter (using test ID)
  await page.check('[data-testid="newsletter-checkbox"]');
  
  // Submit (using button text)
  await page.click('button:text-is("Create Account")');
  
  // Verify success message (using text selector)
  await expect(page.locator('text=Account created successfully')).toBeVisible();
});
```

### 7. Playwright-Specific Selectors

```typescript
import { test, expect } from '@playwright/test';

test('playwright-specific selectors', async ({ page }) => {
  await page.goto('https://example.com');
  
  // :visible - only visible elements
  await page.click('button:visible');
  
  // :text() - text content
  await page.click('button:text("Login")');
  
  // :text-is() - exact text match
  await page.click('button:text-is("Login")');
  
  // :has() - contains element
  await page.click('article:has(h2:text("News"))');
  
  // :has-text() - contains text
  await page.click('div:has-text("Welcome")');
  
  // :left-of, :right-of, :above, :below
  await page.click('button:right-of(:text("Username"))');
  await page.fill('input:below(:text("Email"))', 'user@example.com');
  
  // :near() - within 50 pixels
  await page.click('button:near(:text("Product Name"))');
});
```

### Selector Best Practices:

```typescript
// ✅ GOOD: Stable, readable selectors
await page.click('role=button[name="Submit"]');
await page.fill('[data-testid="email-input"]', 'user@example.com');
await page.click('button:has-text("Add to Cart")');

// ❌ BAD: Fragile, unreadable selectors
await page.click('#root > div > div:nth-child(3) > button.MuiButton-root');
await page.click('div.css-xyz123 > span');
await page.click('//div[1]/div[2]/button[3]');

// ✅ GOOD: Combine selectors for specificity
await page.click('[data-testid="product-card"]:has-text("Laptop") button:has-text("Buy")');

// ❌ BAD: Too specific, breaks easily
await page.click('div.product-grid > div:nth-child(3) > div.card > button.buy-now');
```

---

## Q6: How do you use Locators in Playwright and what are their advantages?

### Answer:

**Locators** are Playwright's way to find elements on the page. They are lazy and represent a query to find elements, not actual DOM elements.

### Locator vs ElementHandle:

| Feature | Locator | ElementHandle |
|---------|---------|---------------|
| **Type** | Query (lazy) | Actual element |
| **Auto-wait** | Yes | No |
| **Auto-retry** | Yes | No |
| **Stale** | Never | Can become stale |
| **Recommended** | ✅ Always use | ❌ Rarely needed |

### Basic Locator Usage:

```typescript
import { test, expect } from '@playwright/test';

test('basic locators', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Create locator (doesn't query DOM yet)
  const submitButton = page.locator('button[type="submit"]');
  
  // Locator actions (queries DOM and auto-waits)
  await submitButton.click();
  
  // Multiple elements
  const products = page.locator('.product-card');
  const count = await products.count();
  console.log(`Found ${count} products`);
  
  // First, last, nth
  await products.first().click();
  await products.last().click();
  await products.nth(2).click();
  
  // Filter locators
  const activeProducts = products.filter({ hasText: 'In Stock' });
  await expect(activeProducts).toHaveCount(5);
});
```

### Locator Methods:

```typescript
import { test, expect } from '@playwright/test';

test('locator methods', async ({ page }) => {
  await page.goto('https://example.com');
  
  const locator = page.locator('[data-testid="product-list"]');
  
  // Get element count
  const count = await locator.count();
  
  // Get text content
  const text = await locator.textContent();
  
  // Get inner text
  const innerText = await locator.innerText();
  
  // Get inner HTML
  const html = await locator.innerHTML();
  
  // Get attribute
  const href = await locator.getAttribute('href');
  
  // Check if visible
  const isVisible = await locator.isVisible();
  
  // Check if enabled
  const isEnabled = await locator.isEnabled();
  
  // Check if checked (checkbox/radio)
  const isChecked = await locator.isChecked();
});
```

### Real-World Scenario: Product List with Locators

```typescript
import { test, expect } from '@playwright/test';

test('product filtering and selection', async ({ page }) => {
  await page.goto('https://shop.example.com/products');
  
  // Get all products
  const products = page.locator('[data-testid="product-card"]');
  
  // Verify total count
  await expect(products).toHaveCount(20);
  
  // Filter products by price range
  const affordableProducts = products.filter({
    has: page.locator('.price:text-matches("\\$[1-9]\\d{0,2}")') // $1-$999
  });
  
  console.log(`Affordable products: ${await affordableProducts.count()}`);
  
  // Filter products in stock
  const inStockProducts = products.filter({
    hasText: 'In Stock'
  });
  
  // Filter by both conditions
  const availableAffordable = products
    .filter({ hasText: 'In Stock' })
    .filter({ has: page.locator('.price:text-matches("\\$[1-9]\\d{0,2}")') });
  
  // Click first available affordable product
  await availableAffordable.first().click();
  
  // Verify product page loaded
  await expect(page.locator('h1.product-title')).toBeVisible();
});
```

### Chaining Locators:

```typescript
import { test, expect } from '@playwright/test';

test('chaining locators', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Start with parent locator
  const form = page.locator('form#login');
  
  // Chain to find children
  const emailInput = form.locator('input[name="email"]');
  const passwordInput = form.locator('input[name="password"]');
  const submitButton = form.locator('button[type="submit"]');
  
  // Use chained locators
  await emailInput.fill('user@example.com');
  await passwordInput.fill('password123');
  await submitButton.click();
  
  // Complex chaining
  const productCard = page
    .locator('[data-testid="products"]')
    .locator('.product-card')
    .filter({ hasText: 'Laptop' })
    .locator('button:has-text("Buy")');
  
  await productCard.click();
});
```

### Scenario: Complex Table Operations

```typescript
import { test, expect } from '@playwright/test';

test('table operations with locators', async ({ page }) => {
  await page.goto('https://example.com/users');
  
  // Get table
  const table = page.locator('table#users');
  
  // Get all rows
  const rows = table.locator('tbody tr');
  const rowCount = await rows.count();
  console.log(`Total users: ${rowCount}`);
  
  // Find specific user row
  const johnRow = rows.filter({ hasText: 'John Doe' });
  
  // Get cells in John's row
  const johnEmail = johnRow.locator('td').nth(1);
  await expect(johnEmail).toHaveText('john@example.com');
  
  // Click edit button in John's row
  await johnRow.locator('button:has-text("Edit")').click();
  
  // Verify edit modal opened
  await expect(page.locator('[data-testid="edit-user-modal"]')).toBeVisible();
  
  // Find all active users
  const activeUsers = rows.filter({
    has: page.locator('.status-badge:has-text("Active")')
  });
  
  console.log(`Active users: ${await activeUsers.count()}`);
  
  // Get all admin users
  const adminUsers = rows.filter({ hasText: 'Admin' });
  
  // Click delete on first admin
  await adminUsers.first().locator('button:has-text("Delete")').click();
});
```

### Locator Assertions:

```typescript
import { test, expect } from '@playwright/test';

test('locator assertions', async ({ page }) => {
  await page.goto('https://example.com');
  
  const element = page.locator('[data-testid="status"]');
  
  // Visibility
  await expect(element).toBeVisible();
  await expect(element).toBeHidden();
  
  // Text content
  await expect(element).toHaveText('Active');
  await expect(element).toContainText('Act');
  await expect(element).toHaveText(/active/i);
  
  // Count
  await expect(page.locator('.item')).toHaveCount(10);
  
  // Attribute
  await expect(element).toHaveAttribute('class', 'status-active');
  
  // CSS
  await expect(element).toHaveCSS('color', 'rgb(0, 128, 0)');
  
  // Value (form inputs)
  await expect(page.locator('#email')).toHaveValue('user@example.com');
  
  // Enabled/Disabled
  await expect(page.locator('button')).toBeEnabled();
  await expect(page.locator('button')).toBeDisabled();
  
  // Checked (checkbox/radio)
  await expect(page.locator('#agree')).toBeChecked();
  await expect(page.locator('#disagree')).not.toBeChecked();
});
```

### Scenario: Form Validation with Locators

```typescript
import { test, expect } from '@playwright/test';

test('form validation feedback', async ({ page }) => {
  await page.goto('https://example.com/register');
  
  const emailInput = page.locator('#email');
  const emailError = page.locator('#email-error');
  const passwordInput = page.locator('#password');
  const passwordError = page.locator('#password-error');
  const submitButton = page.locator('button[type="submit"]');
  
  // Initially no errors
  await expect(emailError).toBeHidden();
  await expect(passwordError).toBeHidden();
  
  // Submit button disabled
  await expect(submitButton).toBeDisabled();
  
  // Enter invalid email
  await emailInput.fill('invalid-email');
  await emailInput.blur();
  
  // Error appears
  await expect(emailError).toBeVisible();
  await expect(emailError).toHaveText('Please enter a valid email');
  await expect(emailError).toHaveCSS('color', 'rgb(220, 53, 69)'); // Red
  
  // Fix email
  await emailInput.fill('user@example.com');
  await emailInput.blur();
  
  // Error disappears
  await expect(emailError).toBeHidden();
  
  // Enter weak password
  await passwordInput.fill('123');
  await passwordInput.blur();
  
  // Password error appears
  await expect(passwordError).toBeVisible();
  await expect(passwordError).toContainText('at least 8 characters');
  
  // Enter strong password
  await passwordInput.fill('SecurePass123!');
  await passwordInput.blur();
  
  // Error disappears, button enabled
  await expect(passwordError).toBeHidden();
  await expect(submitButton).toBeEnabled();
  
  // Submit form
  await submitButton.click();
  
  // Verify success
  await expect(page.locator('.success-message')).toBeVisible();
});
```

### Get-By Methods (Convenience):

```typescript
import { test, expect } from '@playwright/test';

test('get-by convenience methods', async ({ page }) => {
  await page.goto('https://example.com');
  
  // getByRole
  await page.getByRole('button', { name: 'Submit' }).click();
  
  // getByText
  await page.getByText('Welcome').click();
  await page.getByText(/welcome/i).click(); // Case-insensitive
  
  // getByLabel
  await page.getByLabel('Email').fill('user@example.com');
  
  // getByPlaceholder
  await page.getByPlaceholder('Enter your name').fill('John');
  
  // getByTestId
  await page.getByTestId('submit-button').click();
  
  // getByTitle
  await page.getByTitle('Close').click();
  
  // getByAltText (images)
  await expect(page.getByAltText('Company Logo')).toBeVisible();
});
```

### Best Practices:

✅ **Always use Locators** instead of ElementHandles  
✅ **Create reusable locators** for common elements  
✅ **Chain locators** for complex queries  
✅ **Use filters** to narrow down results  
✅ **Leverage auto-waiting** - no manual waits needed  
✅ **Use get-by methods** for cleaner code  

```typescript
// ✅ GOOD: Reusable locators
class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  
  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.locator('#email');
    this.passwordInput = page.locator('#password');
    this.submitButton = page.locator('button[type="submit"]');
  }
  
  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
}

// ❌ BAD: Repeating selectors
test('login', async ({ page }) => {
  await page.fill('#email', 'user@example.com');
  await page.fill('#password', 'pass');
  await page.click('button[type="submit"]');
});
```

---

**Related Topics:**
- Q7-Q8: Click actions and form filling
- Q9-Q10: Assertions and wait strategies
- Q31-Q32: Page Object Model

---

*Document Version: 1.0*  
*Last Updated: November 2025*
