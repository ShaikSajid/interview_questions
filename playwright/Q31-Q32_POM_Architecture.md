# Playwright Interview Questions: Q31-Q32

## Q31: What is Page Object Model and how do you implement it?

### Answer:

Page Object Model (POM) is a design pattern that creates object repositories for web elements, improving test maintainability and reducing code duplication.

### 1. Basic Page Object

```typescript
// pages/LoginPage.ts
import { Page, Locator } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly loginButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.locator('#email');
    this.passwordInput = page.locator('#password');
    this.loginButton = page.locator('#login-btn');
    this.errorMessage = page.locator('.error-message');
  }

  async goto() {
    await this.page.goto('https://example.com/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.loginButton.click();
  }

  async getErrorMessage() {
    return await this.errorMessage.textContent();
  }
}

// test file
import { test, expect } from '@playwright/test';
import { LoginPage } from './pages/LoginPage';

test('login with valid credentials', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login('user@example.com', 'Password123!');
  
  await expect(page).toHaveURL('**/dashboard');
});

test('login with invalid credentials', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login('invalid@example.com', 'wrong');
  
  const error = await loginPage.getErrorMessage();
  expect(error).toContain('Invalid credentials');
});
```

### 2. Advanced Page Object with Methods

```typescript
// pages/DashboardPage.ts
import { Page, Locator, expect } from '@playwright/test';

export class DashboardPage {
  readonly page: Page;
  readonly header: Locator;
  readonly userMenu: Locator;
  readonly logoutButton: Locator;

  constructor(page: Page) {
    this.page = page;
    this.header = page.locator('.dashboard-header');
    this.userMenu = page.locator('#user-menu');
    this.logoutButton = page.locator('#logout');
  }

  async goto() {
    await this.page.goto('https://example.com/dashboard');
  }

  async getUserName() {
    return await this.page.locator('.user-name').textContent();
  }

  async logout() {
    await this.userMenu.click();
    await this.logoutButton.click();
  }

  async verifyLoaded() {
    await expect(this.header).toBeVisible();
    await expect(this.page).toHaveURL('**/dashboard');
  }
}
```

### 3. Real-World E-commerce Page Objects

```typescript
// pages/ProductPage.ts
export class ProductPage {
  readonly page: Page;
  
  constructor(page: Page) {
    this.page = page;
  }

  // Locators
  productTitle = () => this.page.locator('.product-title');
  productPrice = () => this.page.locator('.product-price');
  addToCartButton = () => this.page.locator('#add-to-cart');
  quantityInput = () => this.page.locator('#quantity');
  sizeSelector = () => this.page.locator('#size');
  
  async goto(productId: string) {
    await this.page.goto(`https://shop.example.com/product/${productId}`);
  }

  async addToCart(quantity: number = 1, size?: string) {
    if (size) {
      await this.sizeSelector().selectOption(size);
    }
    await this.quantityInput().fill(quantity.toString());
    await this.addToCartButton().click();
  }

  async getPrice() {
    const priceText = await this.productPrice().textContent();
    return parseFloat(priceText?.replace(/[^0-9.]/g, '') || '0');
  }
}

// pages/CartPage.ts
export class CartPage {
  readonly page: Page;
  
  constructor(page: Page) {
    this.page = page;
  }

  cartItems = () => this.page.locator('.cart-item');
  subtotal = () => this.page.locator('.subtotal');
  checkoutButton = () => this.page.locator('#checkout');
  
  async goto() {
    await this.page.goto('https://shop.example.com/cart');
  }

  async getItemCount() {
    return await this.cartItems().count();
  }

  async removeItem(index: number) {
    await this.cartItems().nth(index).locator('.remove-btn').click();
  }

  async proceedToCheckout() {
    await this.checkoutButton().click();
  }
}

// test
test('complete purchase flow', async ({ page }) => {
  const productPage = new ProductPage(page);
  const cartPage = new CartPage(page);
  
  // Add product
  await productPage.goto('laptop-123');
  await productPage.addToCart(1, 'medium');
  
  // Go to cart
  await cartPage.goto();
  expect(await cartPage.getItemCount()).toBe(1);
  
  // Checkout
  await cartPage.proceedToCheckout();
});
```

### 4. Component-Based Page Objects

```typescript
// components/Header.ts
export class Header {
  readonly page: Page;
  
  constructor(page: Page) {
    this.page = page;
  }

  searchBox = () => this.page.locator('#search');
  cartIcon = () => this.page.locator('.cart-icon');
  cartBadge = () => this.page.locator('.cart-badge');
  
  async search(query: string) {
    await this.searchBox().fill(query);
    await this.searchBox().press('Enter');
  }

  async openCart() {
    await this.cartIcon().click();
  }

  async getCartCount() {
    return await this.cartBadge().textContent();
  }
}

// pages/BasePage.ts
export class BasePage {
  readonly page: Page;
  readonly header: Header;
  
  constructor(page: Page) {
    this.page = page;
    this.header = new Header(page);
  }
}

// pages/HomePage.ts
export class HomePage extends BasePage {
  featuredProducts = () => this.page.locator('.featured-product');
  
  async goto() {
    await this.page.goto('https://shop.example.com');
  }

  async searchProduct(name: string) {
    await this.header.search(name);
  }
}
```

### Related Topics
- **Q32**: Test architecture
- **Q33**: Fixtures

---

## Q32: How do you organize test structure and architecture?

### Answer:

Proper test organization improves maintainability, reusability, and scalability.

### 1. Folder Structure

```
tests/
├── e2e/
│   ├── auth/
│   │   ├── login.spec.ts
│   │   └── signup.spec.ts
│   ├── checkout/
│   │   └── purchase.spec.ts
│   └── dashboard/
│       └── dashboard.spec.ts
├── api/
│   └── users.spec.ts
├── pages/
│   ├── LoginPage.ts
│   ├── DashboardPage.ts
│   └── BasePage.ts
├── fixtures/
│   └── testData.ts
├── utils/
│   └── helpers.ts
└── playwright.config.ts
```

### 2. Test Groups

```typescript
import { test } from '@playwright/test';

test.describe('User Authentication', () => {
  test.describe('Login', () => {
    test('valid credentials', async ({ page }) => {});
    test('invalid credentials', async ({ page }) => {});
    test('password reset', async ({ page }) => {});
  });
  
  test.describe('Signup', () => {
    test('new user registration', async ({ page }) => {});
    test('duplicate email', async ({ page }) => {});
  });
});
```

### 3. Shared Utilities

```typescript
// utils/helpers.ts
export class TestHelper {
  static generateRandomEmail() {
    return `test${Date.now()}@example.com`;
  }
  
  static async waitForApiResponse(page: Page, url: string) {
    return await page.waitForResponse(response => 
      response.url().includes(url) && response.status() === 200
    );
  }
}

// Usage
import { TestHelper } from './utils/helpers';

test('signup', async ({ page }) => {
  const email = TestHelper.generateRandomEmail();
  // Use in test
});
```

### Related Topics
- **Q31**: Page Object Model
- **Q33**: Fixtures

---

*Document Version: 1.0*  
*Last Updated: November 2025*
