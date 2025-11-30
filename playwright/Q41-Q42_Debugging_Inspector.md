# Playwright Interview Questions: Q41-Q42

## Q41: How do you debug Playwright tests?

### Answer:

Playwright offers multiple debugging methods including headed mode, debug mode, and VS Code integration.

### 1. Debug Mode

```bash
# Run with debugger
npx playwright test --debug

# Debug specific test
npx playwright test login.spec.ts --debug

# Debug from specific line
npx playwright test --debug -g "test name"
```

### 2. Headed Mode

```bash
# See browser UI
npx playwright test --headed

# Slow motion
npx playwright test --headed --slow-mo=1000
```

### 3. VS Code Debugging

```json
// .vscode/launch.json
{
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug Playwright",
      "program": "${workspaceFolder}/node_modules/@playwright/test/cli.js",
      "args": ["test", "--debug"]
    }
  ]
}
```

### 4. Pause and Step

```typescript
import { test } from '@playwright/test';

test('debug test', async ({ page }) => {
  await page.goto('https://example.com');
  
  await page.pause(); // Opens inspector
  
  await page.locator('#button').click();
});
```

### Related Topics
- **Q42**: Playwright Inspector
- **Q40**: Trace viewer

---

## Q42: How do you use Playwright Inspector?

### Answer:

Playwright Inspector provides a GUI for debugging with step-through execution and selector playground.

### 1. Launch Inspector

```bash
# With test
npx playwright test --debug

# Codegen mode
npx playwright codegen https://example.com
```

### 2. Features

- **Step through**: Execute actions one by one
- **Pick locator**: Click elements to generate selectors
- **Console**: Run commands interactively
- **Source**: View test code
- **Network**: Monitor requests

### 3. Programmatic Pause

```typescript
test('debug with inspector', async ({ page }) => {
  await page.goto('https://example.com');
  await page.pause(); // Opens inspector here
  await page.locator('#button').click();
});
```

### Related Topics
- **Q41**: Debugging methods
- **Q44**: Codegen

---

*Document Version: 1.0*  
*Last Updated: November 2025*
