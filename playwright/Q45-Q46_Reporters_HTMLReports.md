# Playwright Interview Questions: Q45-Q46

## Q45: What are the different test reporters in Playwright?

### Answer:

Playwright supports multiple built-in reporters and custom reporters.

### 1. Built-in Reporters

```typescript
// playwright.config.ts
export default defineConfig({
  reporter: [
    ['list'], // Default console output
    ['html'], // HTML report
    ['json', { outputFile: 'test-results.json' }],
    ['junit', { outputFile: 'results.xml' }],
    ['github'], // GitHub Actions annotations
    ['dot'], // Minimal output
  ],
});
```

### 2. Reporter Options

```bash
# View HTML report
npx playwright show-report

# Run with specific reporter
npx playwright test --reporter=list
npx playwright test --reporter=html
```

### 3. Custom Reporter

```typescript
// my-reporter.ts
import { Reporter, TestCase, TestResult } from '@playwright/test/reporter';

class MyReporter implements Reporter {
  onTestEnd(test: TestCase, result: TestResult) {
    console.log(`${test.title}: ${result.status}`);
  }
}

export default MyReporter;
```

### Related Topics
- **Q46**: HTML reports
- **Q35**: CI/CD integration

---

## Q46: How do you customize HTML reports?

### Answer:

HTML reporter generates interactive reports with screenshots, videos, and traces.

### 1. Configuration

```typescript
// playwright.config.ts
export default defineConfig({
  reporter: [[
    'html',
    { 
      outputFolder: 'playwright-report',
      open: 'never', // 'always', 'never', 'on-failure'
    }
  ]],
});
```

### 2. View Report

```bash
# Generate and open
npx playwright test
npx playwright show-report

# Serve on specific port
npx playwright show-report --port 9323
```

### 3. Features

- Test results with duration
- Screenshots on failure
- Videos of test execution
- Trace files for debugging
- Filterable by status/browser

### Related Topics
- **Q45**: Other reporters
- **Q40**: Trace viewer

---

*Document Version: 1.0*  
*Last Updated: November 2025*
