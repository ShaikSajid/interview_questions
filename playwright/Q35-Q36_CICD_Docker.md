# Playwright Interview Questions: Q35-Q36

## Q35: How do you integrate Playwright with CI/CD pipelines?

### Answer:

Playwright integrates with all major CI/CD platforms with minimal configuration.

### 1. GitHub Actions

```yaml
# .github/workflows/playwright.yml
name: Playwright Tests
on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]
jobs:
  test:
    timeout-minutes: 60
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-node@v3
      with:
        node-version: 18
    - name: Install dependencies
      run: npm ci
    - name: Install Playwright Browsers
      run: npx playwright install --with-deps
    - name: Run Playwright tests
      run: npx playwright test
    - uses: actions/upload-artifact@v3
      if: always()
      with:
        name: playwright-report
        path: playwright-report/
        retention-days: 30
```

### 2. GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - test

playwright-tests:
  stage: test
  image: mcr.microsoft.com/playwright:v1.40.0-focal
  script:
    - npm ci
    - npx playwright test
  artifacts:
    when: always
    paths:
      - playwright-report/
    expire_in: 30 days
```

### 3. Configuration for CI

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 2 : undefined,
  reporter: process.env.CI ? 'blob' : 'html',
  use: {
    trace: 'on-first-retry',
    video: 'retain-on-failure',
    screenshot: 'only-on-failure',
  },
});
```

### Related Topics
- **Q36**: Docker
- **Q45**: Test reporters

---

## Q36: How do you run Playwright tests in Docker?

### Answer:

Playwright provides official Docker images for containerized testing.

### 1. Dockerfile

```dockerfile
FROM mcr.microsoft.com/playwright:v1.40.0-focal

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

CMD ["npx", "playwright", "test"]
```

### 2. Docker Compose

```yaml
# docker-compose.yml
version: '3'
services:
  playwright:
    image: mcr.microsoft.com/playwright:v1.40.0-focal
    volumes:
      - ./:/app
    working_dir: /app
    command: npx playwright test
```

### 3. Run Commands

```bash
# Build and run
docker build -t my-playwright-tests .
docker run -it my-playwright-tests

# With docker-compose
docker-compose up

# Interactive debugging
docker run -it --rm -v $(pwd):/app -w /app \
  mcr.microsoft.com/playwright:v1.40.0-focal \
  npx playwright test --debug
```

### Related Topics
- **Q35**: CI/CD integration

---

*Document Version: 1.0*  
*Last Updated: November 2025*
