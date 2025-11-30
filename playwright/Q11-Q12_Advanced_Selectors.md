# Playwright Interview Questions: Q11-Q12

## Q11: How do you use advanced selectors for complex scenarios?

### Answer:

Advanced selectors in Playwright allow you to target elements in complex DOM structures using chaining, filtering, and combining multiple strategies.

### 1. Advanced Selector Strategies

| Strategy | Purpose | Example |
|----------|---------|---------|
| **Chaining** | Narrow down scope | `page.locator('.container').locator('.item')` |
| **Filter** | Conditional selection | `locator.filter({ hasText: 'Active' })` |
| **Has** | Parent contains child | `page.locator('article:has(img)')` |
| **And** | Multiple conditions | `locator.and(otherLocator)` |
| **Or** | Alternative conditions | `locator.or(otherLocator)` |
| **nth** | Specific index | `locator.nth(2)` |
| **first/last** | First/last element | `locator.first()`, `locator.last()` |

### 2. Chaining Selectors

```typescript
import { test, expect } from '@playwright/test';

test('chaining selectors demo', async ({ page }) => {
  await page.goto('https://shop.example.com');
  
  // Chain from broad to specific
  const product = page
    .locator('.products-grid')
    .locator('.product-card')
    .filter({ hasText: 'Laptop' })
    .first();
  
  await product.click();
  
  // Complex chaining
  const reviewSection = page
    .locator('#reviews')
    .locator('.review-item')
    .filter({ has: page.locator('.rating-5-stars') });
  
  await expect(reviewSection).toHaveCount(10);
  
  // Chain with nth
  const thirdReview = page
    .locator('.reviews-list')
    .locator('.review')
    .nth(2); // 0-indexed
  
  await expect(thirdReview).toBeVisible();
});
```

### 3. Filtering Locators

```typescript
test('filtering locators', async ({ page }) => {
  await page.goto('https://example.com/users');
  
  // Filter by text content
  const activeUsers = page
    .locator('.user-row')
    .filter({ hasText: 'Active' });
  
  await expect(activeUsers).toHaveCount(15);
  
  // Filter by NOT having text
  const inactiveUsers = page
    .locator('.user-row')
    .filter({ hasNotText: 'Active' });
  
  // Filter by child element
  const usersWithAvatar = page
    .locator('.user-row')
    .filter({ has: page.locator('.avatar') });
  
  // Filter by NOT having child
  const usersWithoutAvatar = page
    .locator('.user-row')
    .filter({ hasNot: page.locator('.avatar') });
  
  // Combine multiple filters
  const premiumActiveUsers = page
    .locator('.user-row')
    .filter({ hasText: 'Active' })
    .filter({ has: page.locator('.premium-badge') });
  
  await expect(premiumActiveUsers).toHaveCount(5);
});
```

### 4. Using Has Selector (Parent Contains Child)

```typescript
test('has selector for parent-child relationships', async ({ page }) => {
  await page.goto('https://blog.example.com');
  
  // Find articles that have an image
  const articlesWithImage = page.locator('article:has(img)');
  await expect(articlesWithImage).toHaveCount(8);
  
  // Find articles with video
  const articlesWithVideo = page.locator('article:has(video)');
  
  // Find products with discount badge
  const discountedProducts = page.locator('.product:has(.discount-badge)');
  
  // Complex has selector
  const featuredArticlesWithComments = page.locator(
    'article:has(.featured-badge):has(.comments-section)'
  );
  
  // Has with specific child content
  const articlesAboutPython = page.locator('article:has(h2:text("Python"))');
});
```

### 5. Logical Operators (And, Or)

```typescript
test('logical operators for selectors', async ({ page }) => {
  await page.goto('https://example.com/products');
  
  // AND: Element must match both conditions
  const expensiveLaptops = page
    .locator('.product')
    .filter({ hasText: 'Laptop' })
    .and(page.locator('[data-price-range="high"]'));
  
  await expect(expensiveLaptops).toHaveCount(3);
  
  // OR: Element matches either condition
  const laptopsOrTablets = page
    .locator('.product[data-category="laptop"]')
    .or(page.locator('.product[data-category="tablet"]'));
  
  await expect(laptopsOrTablets).toHaveCount(25);
  
  // Complex logical combination
  const searchResults = page
    .locator('.search-result')
    .filter({ hasText: 'MacBook' })
    .or(page.locator('.search-result').filter({ hasText: 'ThinkPad' }))
    .filter({ has: page.locator('.in-stock') });
});
```

### 6. Layout-Based Selectors

```typescript
test('layout-based selectors', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Element to the right of another
  const labelText = await page
    .locator('button:right-of(:text("Cancel"))')
    .textContent();
  // Finds "OK" button to the right of "Cancel"
  
  // Element to the left
  await page.locator('button:left-of(:text("Submit"))').click();
  
  // Element above
  const header = page.locator('h1:above(:text("Sign in"))');
  
  // Element below
  const footer = page.locator('div:below(.main-content)');
  
  // Element near another (within 50px)
  const nearbyLabel = page.locator('label:near(#email-input)');
  await expect(nearbyLabel).toHaveText('Email Address');
  
  // Custom distance
  const closeElements = page.locator('button:near(#submit, 100)'); // within 100px
});
```

### 7. Real-World Scenario: Complex E-commerce Product Grid

```typescript
test('navigate complex product grid with advanced selectors', async ({ page }) => {
  await page.goto('https://shop.example.com/laptops');
  
  // Find all laptops with specific criteria:
  // - In stock
  // - Price between $1000-$2000
  // - Has 5-star rating
  // - Has discount
  
  const qualifyingProducts = page
    .locator('.product-card')
    .filter({ has: page.locator('.in-stock-badge') })
    .filter({ has: page.locator('[data-price]').filter({ 
      hasText: /\$1,[0-9]{3}|\$2,000/ 
    })})
    .filter({ has: page.locator('.rating-5-stars') })
    .filter({ has: page.locator('.discount-badge') });
  
  const count = await qualifyingProducts.count();
  console.log(`Found ${count} qualifying products`);
  
  // Get the first qualifying product
  const firstProduct = qualifyingProducts.first();
  
  // Get product details from the card
  const productName = await firstProduct.locator('.product-name').textContent();
  const productPrice = await firstProduct.locator('.product-price').textContent();
  const discountPercent = await firstProduct.locator('.discount-badge').textContent();
  
  console.log(`${productName} - ${productPrice} (${discountPercent} off)`);
  
  // Click "Quick View" button within this product card
  await firstProduct.locator('.quick-view-btn').click();
  
  // Verify modal opened
  await expect(page.locator('.product-modal')).toBeVisible();
  
  // Within modal, find specs section
  const specsSection = page
    .locator('.product-modal')
    .locator('.specs-section');
  
  // Find specific specs
  const ramSpec = specsSection
    .locator('.spec-row')
    .filter({ hasText: 'RAM' })
    .locator('.spec-value');
  
  await expect(ramSpec).toHaveText('16GB');
  
  // Find products in same price range
  const similarProducts = page
    .locator('.similar-products')
    .locator('.product-card')
    .filter({ 
      has: page.locator('[data-price]').filter(async (locator) => {
        const price = await locator.getAttribute('data-price');
        const numPrice = parseInt(price || '0');
        return numPrice >= 1000 && numPrice <= 2000;
      })
    });
});
```

### 8. Real-World Scenario: Complex Table Filtering

```typescript
test('filter and select from complex data table', async ({ page }) => {
  await page.goto('https://admin.example.com/users');
  
  // Find all rows where:
  // - Status is "Active"
  // - Role is "Admin" or "Manager"
  // - Last login within 7 days
  // - Has 2FA enabled
  
  const table = page.locator('table#users-table');
  const rows = table.locator('tbody tr');
  
  // Filter by status
  const activeRows = rows.filter({ 
    has: page.locator('td.status:has-text("Active")') 
  });
  
  // Filter by role
  const adminOrManagerRows = activeRows.filter({
    has: page.locator('td.role')
      .filter({ hasText: 'Admin' })
      .or(page.locator('td.role').filter({ hasText: 'Manager' }))
  });
  
  // Filter by 2FA
  const twoFactorRows = adminOrManagerRows.filter({
    has: page.locator('td.two-factor .enabled-icon')
  });
  
  const finalCount = await twoFactorRows.count();
  console.log(`Found ${finalCount} matching users`);
  
  // Click on first matching user
  await twoFactorRows.first().locator('td.name a').click();
  
  // Verify user details page
  await expect(page).toHaveURL(/\/users\/\d+/);
  
  // Alternative approach: Find specific user by multiple criteria
  const specificUser = rows
    .filter({ hasText: 'john.doe@example.com' })
    .filter({ has: page.locator('.status:has-text("Active")') })
    .filter({ has: page.locator('.role:has-text("Admin")') });
  
  // Edit this user
  await specificUser.locator('.edit-btn').click();
});
```

### 9. Working with Complex Forms

```typescript
test('complex dynamic form with conditional fields', async ({ page }) => {
  await page.goto('https://forms.example.com/application');
  
  // Select user type which reveals different fields
  await page.locator('#user-type').selectOption('business');
  
  // Wait for business-specific fields to appear
  const businessFields = page
    .locator('.form-group')
    .filter({ has: page.locator('[data-business-only="true"]') });
  
  await expect(businessFields).toBeVisible();
  
  // Fill only visible required fields
  const requiredFields = page
    .locator('input[required]')
    .filter({ 
      hasNot: page.locator('[style*="display: none"]') 
    })
    .filter({ 
      hasNot: page.locator('.hidden') 
    });
  
  const fieldCount = await requiredFields.count();
  
  for (let i = 0; i < fieldCount; i++) {
    const field = requiredFields.nth(i);
    const fieldType = await field.getAttribute('type');
    
    if (fieldType === 'text' || fieldType === 'email') {
      await field.fill('test value');
    } else if (fieldType === 'checkbox') {
      await field.check();
    }
  }
  
  // Find submit button that's not disabled
  const activeSubmitButton = page
    .locator('button[type="submit"]')
    .filter({ hasNot: page.locator('[disabled]') });
  
  await activeSubmitButton.click();
});
```

### 10. Advanced Text Matching

```typescript
test('advanced text matching patterns', async ({ page }) => {
  await page.goto('https://example.com/products');
  
  // Exact text match
  await page.locator('button:has-text("Add to Cart")').click();
  
  // Partial text match (case-insensitive)
  const items = page.locator('.item:has-text("laptop")');
  
  // Regex for flexible matching
  const priceItems = page.locator('.price:text-matches("\\$[0-9,]+\\.[0-9]{2}")');
  
  // Multiple words (any order)
  const searchResults = page.locator('article:has-text("Python"):has-text("Tutorial")');
  
  // Text with special characters
  const emailLinks = page.locator('a:has-text("support@example.com")');
  
  // Text in specific element type
  const h2WithText = page.locator('h2:text("Features")');
  
  // Combining text with attributes
  const importantErrors = page
    .locator('.message')
    .filter({ hasText: 'Error' })
    .filter({ has: page.locator('[data-severity="high"]') });
});
```

### 11. Working with Shadow DOM

```typescript
test('selectors in shadow DOM', async ({ page }) => {
  await page.goto('https://example.com/web-components');
  
  // Pierce shadow DOM
  const shadowButton = page.locator('my-component >> button.action');
  await shadowButton.click();
  
  // Multiple shadow DOM levels
  const deepElement = page.locator(
    'parent-component >> child-component >> .deep-element'
  );
  
  // Combining shadow piercing with other selectors
  const shadowWithFilter = page
    .locator('custom-list >> .item')
    .filter({ hasText: 'Active' });
  
  await expect(shadowWithFilter).toHaveCount(3);
});
```

### 12. Best Practices

✅ **GOOD: Use semantic selectors with filters**
```typescript
const activeAdmins = page
  .locator('[role="row"]')
  .filter({ hasText: 'Active' })
  .filter({ has: page.locator('.role:has-text("Admin")') });
```

❌ **BAD: Complex XPath**
```typescript
const items = page.locator('//div[@class="row"][contains(text(), "Active")]//span[@class="role" and text()="Admin"]/ancestor::div');
```

✅ **GOOD: Readable chaining**
```typescript
const product = page
  .locator('.products')
  .locator('.product-card')
  .filter({ hasText: 'Laptop' })
  .first();
```

❌ **BAD: Single complex selector**
```typescript
const product = page.locator('.products .product-card:has-text("Laptop"):first');
```

### 13. Common Pitfalls

1. **Over-filtering and getting no results**
   ```typescript
   // ❌ Too restrictive
   const items = page
     .locator('.item')
     .filter({ hasText: 'Exact Match Only' })
     .filter({ has: page.locator('.rare-class') });
   
   // ✅ Gradually filter and check count
   const items = page.locator('.item');
   console.log(await items.count());
   const filtered = items.filter({ hasText: 'Match' });
   console.log(await filtered.count());
   ```

2. **Not considering element visibility**
   ```typescript
   // ❌ Includes hidden elements
   const buttons = page.locator('button');
   
   // ✅ Only visible elements
   const visibleButtons = page.locator('button:visible');
   ```

### Related Topics
- **Q5**: Basic selectors
- **Q6**: Locator methods
- **Q12**: Handling dynamic content
- **Q20**: Shadow DOM navigation

---

## Q12: How do you handle dynamic content in Playwright?

### Answer:

Dynamic content includes elements that load asynchronously, change frequently, or have dynamic IDs/classes. Playwright's auto-waiting and smart selectors handle most dynamic scenarios automatically.

### 1. Dynamic Content Patterns

| Pattern | Challenge | Solution |
|---------|-----------|----------|
| **Lazy Loading** | Content loads on scroll | `await page.evaluate()` to scroll, wait for elements |
| **Infinite Scroll** | More items load continuously | Detect scroll position, wait for new items |
| **Dynamic IDs** | IDs change on each load | Use stable attributes (data-testid, aria-label) |
| **AJAX Updates** | Content updates without navigation | Wait for network responses, observe DOM |
| **WebSocket Data** | Real-time updates | Use `waitForFunction` with custom logic |
| **Skeleton Screens** | Placeholder → real content | Wait for skeleton to disappear |

### 2. Handling Lazy-Loaded Content

```typescript
import { test, expect } from '@playwright/test';

test('handle lazy-loaded images', async ({ page }) => {
  await page.goto('https://example.com/gallery');
  
  // Scroll to trigger lazy loading
  await page.evaluate(() => window.scrollTo(0, document.body.scrollHeight));
  
  // Wait for images to load
  await page.locator('img[data-src]').first().waitFor({ state: 'hidden' });
  await page.locator('img[src]').first().waitFor({ state: 'visible' });
  
  // Alternative: scroll step by step
  const images = page.locator('.lazy-image');
  const count = await images.count();
  
  for (let i = 0; i < count; i++) {
    await images.nth(i).scrollIntoViewIfNeeded();
    await images.nth(i).waitFor({ state: 'visible' });
  }
  
  // Verify all images loaded
  await expect(images).toHaveCount(50);
});
```

### 3. Infinite Scroll

```typescript
test('handle infinite scroll feed', async ({ page }) => {
  await page.goto('https://social.example.com/feed');
  
  let previousCount = 0;
  let currentCount = await page.locator('.post').count();
  let scrollAttempts = 0;
  const maxScrolls = 10;
  
  while (scrollAttempts < maxScrolls && currentCount > previousCount) {
    previousCount = currentCount;
    
    // Scroll to bottom
    await page.evaluate(() => window.scrollTo(0, document.body.scrollHeight));
    
    // Wait for new items to load
    await page.waitForFunction(
      (prevCount) => document.querySelectorAll('.post').length > prevCount,
      previousCount,
      { timeout: 5000 }
    ).catch(() => {}); // Ignore timeout if no more items
    
    currentCount = await page.locator('.post').count();
    scrollAttempts++;
    
    console.log(`Loaded ${currentCount} posts after ${scrollAttempts} scrolls`);
  }
  
  console.log(`Final count: ${currentCount} posts`);
});
```

### 4. Handling Dynamic IDs and Classes

```typescript
test('handle dynamic IDs and classes', async ({ page }) => {
  await page.goto('https://example.com/dashboard');
  
  // ❌ BAD: Dynamic ID that changes
  // await page.locator('#button-1234567890').click();
  
  // ✅ GOOD: Use stable data attributes
  await page.locator('[data-testid="submit-button"]').click();
  
  // ✅ GOOD: Use ARIA labels
  await page.locator('[aria-label="Submit form"]').click();
  
  // ✅ GOOD: Use role with name
  await page.getByRole('button', { name: 'Submit' }).click();
  
  // ✅ GOOD: Use relative positioning
  await page.locator('form').locator('button[type="submit"]').click();
  
  // For dynamic classes (e.g., "btn-abc123 btn-primary")
  // Use partial class matching
  await page.locator('[class*="btn-primary"]').click();
  
  // Or use multiple stable classes
  await page.locator('.btn.primary').click();
});
```

### 5. Waiting for AJAX/API Updates

```typescript
test('handle AJAX content updates', async ({ page }) => {
  await page.goto('https://dashboard.example.com');
  
  // Method 1: Wait for specific network response
  const responsePromise = page.waitForResponse(
    response => response.url().includes('/api/metrics') && response.status() === 200
  );
  
  await page.locator('#refresh-btn').click();
  const response = await responsePromise;
  const data = await response.json();
  
  console.log('Received metrics:', data);
  
  // Method 2: Wait for DOM to update
  await page.locator('#loading-indicator').waitFor({ state: 'visible' });
  await page.locator('#loading-indicator').waitFor({ state: 'hidden' });
  
  // Method 3: Wait for specific content to change
  const oldValue = await page.locator('.metric-value').textContent();
  await page.locator('#refresh-btn').click();
  
  await page.waitForFunction(
    (oldVal) => {
      const newVal = document.querySelector('.metric-value')?.textContent;
      return newVal !== oldVal;
    },
    oldValue
  );
  
  // Verify new content
  const newValue = await page.locator('.metric-value').textContent();
  expect(newValue).not.toBe(oldValue);
});
```

### 6. Real-World Scenario: Dynamic Product Catalog

```typescript
test('dynamic product catalog with filters', async ({ page }) => {
  await page.goto('https://shop.example.com/products');
  
  // Wait for initial products to load
  await page.waitForResponse('**/api/products**');
  await page.locator('.product-card').first().waitFor({ state: 'visible' });
  
  const initialCount = await page.locator('.product-card').count();
  console.log(`Initial products: ${initialCount}`);
  
  // Apply price filter
  await page.locator('#price-min').fill('100');
  await page.locator('#price-max').fill('500');
  
  // Wait for filter API call
  const filterResponse = page.waitForResponse(
    response => response.url().includes('/api/products') && 
                response.url().includes('price')
  );
  
  await page.locator('#apply-filters').click();
  await filterResponse;
  
  // Wait for products to update (skeleton loading)
  await page.locator('.skeleton-card').first().waitFor({ 
    state: 'visible',
    timeout: 1000
  }).catch(() => {}); // Skeleton might be too fast
  
  await page.locator('.skeleton-card').first().waitFor({ 
    state: 'detached',
    timeout: 10000
  }).catch(() => {});
  
  // Wait for real products
  await page.locator('.product-card').first().waitFor({ state: 'visible' });
  
  const filteredCount = await page.locator('.product-card').count();
  console.log(`Filtered products: ${filteredCount}`);
  
  expect(filteredCount).toBeLessThan(initialCount);
  
  // Verify all visible products match price range
  const productCards = await page.locator('.product-card').all();
  
  for (const card of productCards) {
    const priceText = await card.locator('.price').textContent();
    const price = parseFloat(priceText?.replace(/[^0-9.]/g, '') || '0');
    expect(price).toBeGreaterThanOrEqual(100);
    expect(price).toBeLessThanOrEqual(500);
  }
  
  // Apply category filter (updates URL and content)
  await page.locator('[data-category="electronics"]').click();
  
  // Wait for URL to update
  await page.waitForURL('**/products?**category=electronics**');
  
  // Wait for new API call
  await page.waitForResponse(response => 
    response.url().includes('category=electronics')
  );
  
  // Verify category
  const categoryBadges = await page.locator('.product-card .category-badge').all();
  for (const badge of categoryBadges) {
    await expect(badge).toHaveText('Electronics');
  }
});
```

### 7. Real-World Scenario: Real-Time Chat Application

```typescript
test('real-time chat with WebSocket updates', async ({ page }) => {
  await page.goto('https://chat.example.com');
  
  // Login
  await page.locator('#username').fill('testuser');
  await page.locator('#room').fill('general');
  await page.locator('#join-btn').click();
  
  // Wait for WebSocket connection
  await page.waitForFunction(() => {
    return window.websocket && window.websocket.readyState === 1;
  });
  
  // Initial message count
  const initialCount = await page.locator('.message').count();
  
  // Send a message
  await page.locator('#message-input').fill('Hello everyone!');
  await page.locator('#send-btn').click();
  
  // Wait for message to appear
  await page.waitForFunction(
    (oldCount) => document.querySelectorAll('.message').length > oldCount,
    initialCount
  );
  
  // Verify our message
  const lastMessage = page.locator('.message').last();
  await expect(lastMessage).toContainText('Hello everyone!');
  await expect(lastMessage).toHaveAttribute('data-sender', 'testuser');
  
  // Simulate receiving message from another user
  // (In real test, might have second page/context)
  const beforeReceive = await page.locator('.message').count();
  
  // Wait for new message (with timeout)
  await page.waitForFunction(
    (count) => document.querySelectorAll('.message').length > count,
    beforeReceive,
    { timeout: 30000 }
  ).catch(() => {
    console.log('No new messages received in 30 seconds');
  });
  
  // Check for typing indicators
  await page.locator('#message-input').fill('Typing...');
  await expect(page.locator('.typing-indicator')).toBeVisible();
  
  // Clear input and indicator should disappear
  await page.locator('#message-input').clear();
  await expect(page.locator('.typing-indicator')).not.toBeVisible();
  
  // Monitor connection status
  await page.waitForFunction(() => {
    const status = document.querySelector('.connection-status');
    return status?.textContent === 'Connected';
  });
});
```

### 8. Handling Skeleton Screens and Loading States

```typescript
test('handle skeleton screens', async ({ page }) => {
  await page.goto('https://example.com/dashboard');
  
  // Skeleton appears immediately
  await expect(page.locator('.skeleton-card')).toBeVisible();
  
  // Wait for API response
  await page.waitForResponse('**/api/dashboard-data');
  
  // Skeleton disappears, real content appears
  await page.locator('.skeleton-card').first().waitFor({ 
    state: 'detached',
    timeout: 10000 
  });
  
  // Real cards are visible
  await expect(page.locator('.data-card')).toHaveCount(6);
  
  // Alternative: Wait for class change
  await page.waitForFunction(() => {
    const container = document.querySelector('.dashboard-grid');
    return !container?.classList.contains('loading');
  });
});
```

### 9. Handling Dynamic Tables with Sorting/Pagination

```typescript
test('handle dynamic table with sorting and pagination', async ({ page }) => {
  await page.goto('https://admin.example.com/users');
  
  // Initial page load
  await page.waitForResponse('**/api/users?page=1**');
  const initialFirstUser = await page.locator('tbody tr').first().locator('td.name').textContent();
  
  // Sort by name
  await page.locator('th.name').click();
  await page.waitForResponse(response => 
    response.url().includes('/api/users') && 
    response.url().includes('sort=name')
  );
  
  // Wait for table to update
  await page.waitForFunction(
    (oldName) => {
      const newName = document.querySelector('tbody tr:first-child td.name')?.textContent;
      return newName !== oldName;
    },
    initialFirstUser
  );
  
  // Verify sorting
  const sortedFirstUser = await page.locator('tbody tr').first().locator('td.name').textContent();
  expect(sortedFirstUser).not.toBe(initialFirstUser);
  
  // Go to page 2
  await page.locator('[aria-label="Go to page 2"]').click();
  await page.waitForResponse('**/api/users?page=2**');
  
  // Wait for URL to update
  await page.waitForURL('**/users?**page=2**');
  
  // Wait for new data
  await page.waitForFunction(() => {
    const pageIndicator = document.querySelector('.page-indicator');
    return pageIndicator?.textContent === 'Page 2 of 10';
  });
  
  // Verify page 2 content
  const page2FirstUser = await page.locator('tbody tr').first().locator('td.name').textContent();
  expect(page2FirstUser).not.toBe(sortedFirstUser);
});
```

### 10. Handling Autocomplete/Typeahead

```typescript
test('handle autocomplete dropdown', async ({ page }) => {
  await page.goto('https://example.com/search');
  
  const searchInput = page.locator('#search-input');
  await searchInput.fill('pla');
  
  // Wait for autocomplete API
  await page.waitForResponse('**/api/autocomplete?q=pla**');
  
  // Wait for dropdown to appear
  await page.locator('.autocomplete-dropdown').waitFor({ state: 'visible' });
  
  // Wait for items to load
  await page.locator('.autocomplete-item').first().waitFor({ state: 'visible' });
  
  // Verify suggestions
  await expect(page.locator('.autocomplete-item')).toHaveCount(5);
  
  // Continue typing
  await searchInput.fill('playw');
  
  // Wait for new results
  const oldFirstItem = await page.locator('.autocomplete-item').first().textContent();
  await page.waitForResponse('**/api/autocomplete?q=playw**');
  
  await page.waitForFunction(
    (oldText) => {
      const newText = document.querySelector('.autocomplete-item')?.textContent;
      return newText !== oldText;
    },
    oldFirstItem
  );
  
  // Select suggestion
  await page.locator('.autocomplete-item').filter({ hasText: 'Playwright' }).click();
  
  // Dropdown closes
  await expect(page.locator('.autocomplete-dropdown')).not.toBeVisible();
  
  // Input filled with selection
  await expect(searchInput).toHaveValue('Playwright');
});
```

### 11. Handling Dynamic Modals and Overlays

```typescript
test('handle dynamic modals', async ({ page }) => {
  await page.goto('https://example.com');
  
  // Click button that triggers modal
  await page.locator('#open-modal').click();
  
  // Wait for modal API (if it loads data)
  await page.waitForResponse('**/api/modal-content**').catch(() => {});
  
  // Wait for modal to animate in
  const modal = page.locator('.modal');
  await modal.waitFor({ state: 'visible' });
  
  // Wait for modal to be fully loaded (no loading spinner)
  await page.locator('.modal .loading-spinner').waitFor({ 
    state: 'detached',
    timeout: 5000
  }).catch(() => {});
  
  // Interact with modal content
  await expect(modal.locator('.modal-title')).toBeVisible();
  await modal.locator('#modal-input').fill('test');
  await modal.locator('#modal-submit').click();
  
  // Wait for modal to close
  await modal.waitFor({ state: 'hidden' });
  
  // Verify success message appears
  await expect(page.locator('.success-toast')).toBeVisible();
});
```

### 12. Best Practices

✅ **GOOD: Use stable selectors**
```typescript
await page.locator('[data-testid="product-card"]').click();
```

❌ **BAD: Use dynamic IDs**
```typescript
await page.locator('#dynamic-id-1234567890').click();
```

✅ **GOOD: Wait for API responses**
```typescript
await Promise.all([
  page.waitForResponse('**/api/data'),
  page.locator('#load').click()
]);
```

❌ **BAD: Fixed timeouts**
```typescript
await page.locator('#load').click();
await page.waitForTimeout(3000);
```

### 13. Common Pitfalls

1. **Not waiting for content to stabilize**
   ```typescript
   // ❌ Might get stale data
   await page.locator('#refresh').click();
   const value = await page.locator('.value').textContent();
   
   // ✅ Wait for update
   const oldValue = await page.locator('.value').textContent();
   await page.locator('#refresh').click();
   await page.waitForFunction((old) => {
     return document.querySelector('.value')?.textContent !== old;
   }, oldValue);
   const newValue = await page.locator('.value').textContent();
   ```

2. **Race conditions with multiple updates**
   ```typescript
   // ❌ May catch wrong update
   await page.locator('#filter1').click();
   await page.locator('#filter2').click();
   await page.waitForResponse('**/api/data');
   
   // ✅ Wait for both responses
   await Promise.all([
     page.waitForResponse(r => r.url().includes('filter1')),
     page.locator('#filter1').click()
   ]);
   await Promise.all([
     page.waitForResponse(r => r.url().includes('filter2')),
     page.locator('#filter2').click()
   ]);
   ```

### Related Topics
- **Q10**: Wait strategies
- **Q11**: Advanced selectors
- **Q27**: API mocking for testing dynamic content

---

*Document Version: 1.0*  
*Last Updated: November 2025*
