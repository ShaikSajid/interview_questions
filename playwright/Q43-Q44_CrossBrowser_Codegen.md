# Playwright Interview Questions: Q43-Q44

## Q43: How do you test across multiple browsers?

### Answer:

Playwright runs tests on Chromium, Firefox, and WebKit with a single codebase.

### 1. Configuration

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
    { name: 'mobile-chrome', use: { ...devices['Pixel 5'] } },
    { name: 'mobile-safari', use: { ...devices['iPhone 13'] } },
  ],
});
```

### 2. Run Specific Browser

```bash
# All browsers
npx playwright test

# Specific browser
npx playwright test --project=chromium
npx playwright test --project=firefox

# Multiple browsers
npx playwright test --project=chromium --project=firefox
```

### Related Topics
- **Q25**: Mobile emulation

---

## Q44: How do you use Playwright Codegen?

### Answer:

Codegen records user interactions and generates test code automatically.

### 1. Basic Codegen

```bash
# Start recording
npx playwright codegen https://example.com

# With specific browser
npx playwright codegen --browser=firefox https://example.com

# With device emulation
npx playwright codegen --device="iPhone 13" https://example.com

# Save to file
npx playwright codegen --target=typescript -o tests/recorded.spec.ts https://example.com
```

### 2. Codegen with Auth

```bash
# Load saved storage state
npx playwright codegen --load-storage=auth.json https://example.com/dashboard
```

### 3. Features

- Records clicks, typing, navigation
- Generates TypeScript/JavaScript/Python/Java/C# code
- Exports locators
- Supports mobile devices

### Related Topics
- **Q42**: Inspector
- **Q5**: Selectors

---

*Document Version: 1.0*  
*Last Updated: November 2025*
