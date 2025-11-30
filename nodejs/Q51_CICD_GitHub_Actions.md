# Q51: CI/CD - GitHub Actions for Node.js

## 📋 Summary
This question covers **CI/CD pipelines** with GitHub Actions - automated testing, building, security scanning, and deployment of Node.js applications. Learn to create production-ready workflows for banking APIs with comprehensive quality gates and multi-environment deployments.

**Key Topics**:
- GitHub Actions fundamentals and syntax
- Automated testing (unit, integration, E2E)
- Docker image building and pushing
- Security scanning (dependencies, code, containers)
- Multi-environment deployments (dev, staging, prod)
- Secrets management
- Caching strategies for faster builds
- Branch protection and deployment gates
- Rollback mechanisms

**Banking Use Cases**:
- Automated API deployment pipeline
- Security-first CI/CD for financial apps
- Compliance-ready audit trails
- Automated database migrations
- Blue-green deployment automation
- Performance testing in CI

---

## 🎯 Understanding GitHub Actions

### What are GitHub Actions?

**GitHub Actions** is GitHub's built-in CI/CD platform that automates workflows triggered by repository events.

```
┌─────────────────────────────────────────────────────────────┐
│              GitHub Actions Architecture                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Repository Event                                           │
│  (push, PR, release)                                        │
│         │                                                    │
│         ▼                                                    │
│  ┌──────────────────┐                                       │
│  │   Workflow YAML  │  (.github/workflows/ci.yml)          │
│  └────────┬─────────┘                                       │
│           │                                                  │
│           ▼                                                  │
│  ┌──────────────────┐                                       │
│  │   Job: Test      │  (runs-on: ubuntu-latest)            │
│  │  - Checkout      │                                       │
│  │  - Setup Node    │                                       │
│  │  - Install deps  │                                       │
│  │  - Run tests     │                                       │
│  └────────┬─────────┘                                       │
│           │                                                  │
│           ▼                                                  │
│  ┌──────────────────┐                                       │
│  │   Job: Build     │  (depends on: test)                  │
│  │  - Build Docker  │                                       │
│  │  - Push to ECR   │                                       │
│  └────────┬─────────┘                                       │
│           │                                                  │
│           ▼                                                  │
│  ┌──────────────────┐                                       │
│  │  Job: Deploy     │  (environment: production)           │
│  │  - Deploy to ECS │                                       │
│  │  - Smoke tests   │                                       │
│  └──────────────────┘                                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Workflow** | Automated process defined in YAML |
| **Job** | Set of steps that execute on same runner |
| **Step** | Individual task (action or command) |
| **Action** | Reusable unit of code |
| **Runner** | Server that runs workflows |
| **Trigger** | Event that starts workflow |
| **Secret** | Encrypted environment variable |

### Workflow Syntax

```yaml
name: Workflow Name
on: [push, pull_request]  # Trigger events
jobs:
  job-name:
    runs-on: ubuntu-latest  # Runner
    steps:
      - uses: actions/checkout@v3  # Action
      - run: npm test  # Command
```

---

## 💡 Example 1: Complete Banking API CI/CD Pipeline

Production-ready GitHub Actions workflow with comprehensive quality gates.

### Project Structure

```
banking-api/
├── .github/
│   └── workflows/
│       ├── ci.yml              # Main CI pipeline
│       ├── deploy-prod.yml     # Production deployment
│       └── security-scan.yml   # Security scanning
├── src/
├── test/
├── Dockerfile
├── docker-compose.yml
└── package.json
```

### .github/workflows/ci.yml

```yaml
# ============================================
# Banking API - CI/CD Pipeline
# Triggers: Push, Pull Request
# ============================================

name: Banking API CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]
  workflow_dispatch:  # Manual trigger

env:
  NODE_VERSION: '18.x'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ============================================
  # JOB 1: Code Quality & Linting
  # ============================================
  lint:
    name: Lint & Format Check
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run ESLint
        run: npm run lint
      
      - name: Check formatting
        run: npm run format:check
      
      - name: TypeScript type check
        run: npm run type-check
        continue-on-error: false

  # ============================================
  # JOB 2: Security Scanning
  # ============================================
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Run npm audit
        run: npm audit --audit-level=high
        continue-on-error: true
      
      - name: Run Snyk to check for vulnerabilities
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
      
      - name: CodeQL Analysis
        uses: github/codeql-action/init@v2
        with:
          languages: javascript
      
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v2

  # ============================================
  # JOB 3: Unit & Integration Tests
  # ============================================
  test:
    name: Test Suite
    runs-on: ubuntu-latest
    needs: [lint]
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_DB: banking_test
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run unit tests
        run: npm run test:unit
        env:
          NODE_ENV: test
      
      - name: Run integration tests
        run: npm run test:integration
        env:
          NODE_ENV: test
          DB_HOST: localhost
          DB_PORT: 5432
          DB_NAME: banking_test
          DB_USER: testuser
          DB_PASSWORD: testpass
          REDIS_HOST: localhost
          REDIS_PORT: 6379
      
      - name: Generate coverage report
        run: npm run test:coverage
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/lcov.info
          fail_ci_if_error: true
      
      - name: Coverage threshold check
        run: |
          COVERAGE=$(node -p "require('./coverage/coverage-summary.json').total.lines.pct")
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage is below 80%: $COVERAGE%"
            exit 1
          fi

  # ============================================
  # JOB 4: Build Docker Image
  # ============================================
  build:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    needs: [test, security]
    if: github.event_name == 'push'
    
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,prefix={{branch}}-
            type=semver,pattern={{version}}
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            NODE_ENV=production
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            VERSION=${{ github.sha }}
      
      - name: Scan Docker image for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.meta.outputs.tags }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
      
      - name: Upload Trivy results to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  # ============================================
  # JOB 5: Deploy to Development
  # ============================================
  deploy-dev:
    name: Deploy to Development
    runs-on: ubuntu-latest
    needs: [build]
    if: github.ref == 'refs/heads/develop'
    environment:
      name: development
      url: https://dev.banking-api.example.com
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster banking-api-dev \
            --service api-service \
            --force-new-deployment
      
      - name: Wait for deployment
        run: |
          aws ecs wait services-stable \
            --cluster banking-api-dev \
            --services api-service
      
      - name: Smoke tests
        run: |
          curl -f https://dev.banking-api.example.com/health || exit 1

  # ============================================
  # JOB 6: Deploy to Staging
  # ============================================
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [build]
    if: github.ref == 'refs/heads/main'
    environment:
      name: staging
      url: https://staging.banking-api.example.com
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Run database migrations
        run: |
          npm ci
          npm run migrate:staging
        env:
          DB_HOST: ${{ secrets.STAGING_DB_HOST }}
          DB_NAME: ${{ secrets.STAGING_DB_NAME }}
          DB_USER: ${{ secrets.STAGING_DB_USER }}
          DB_PASSWORD: ${{ secrets.STAGING_DB_PASSWORD }}
      
      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster banking-api-staging \
            --service api-service \
            --force-new-deployment
      
      - name: Wait for deployment
        run: |
          aws ecs wait services-stable \
            --cluster banking-api-staging \
            --services api-service
      
      - name: Run E2E tests
        run: npm run test:e2e
        env:
          API_URL: https://staging.banking-api.example.com
      
      - name: Performance tests
        run: |
          npm install -g artillery
          artillery run tests/performance/load-test.yml
      
      - name: Notify Slack
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "✅ Banking API deployed to staging",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Banking API Deployment*\n✅ Deployed to staging\n*Branch:* ${{ github.ref_name }}\n*Commit:* ${{ github.sha }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

  # ============================================
  # JOB 7: Create Release
  # ============================================
  release:
    name: Create Release
    runs-on: ubuntu-latest
    needs: [deploy-staging]
    if: github.ref == 'refs/heads/main' && startsWith(github.event.head_commit.message, 'Release')
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Get version from package.json
        id: version
        run: echo "version=$(node -p "require('./package.json').version")" >> $GITHUB_OUTPUT
      
      - name: Create GitHub Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: v${{ steps.version.outputs.version }}
          release_name: Release v${{ steps.version.outputs.version }}
          body: |
            ## Changes in this Release
            ${{ github.event.head_commit.message }}
            
            ## Docker Image
            `ghcr.io/${{ env.IMAGE_NAME }}:v${{ steps.version.outputs.version }}`
          draft: false
          prerelease: false
```

### .github/workflows/deploy-prod.yml

```yaml
# ============================================
# Production Deployment (Manual Trigger Only)
# Requires: Approval from team leads
# ============================================

name: Deploy to Production

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to deploy (e.g., v1.2.3)'
        required: true
      skip-tests:
        description: 'Skip pre-deployment tests (not recommended)'
        type: boolean
        default: false

jobs:
  # ============================================
  # Pre-deployment Validation
  # ============================================
  validate:
    name: Pre-deployment Validation
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.inputs.version }}
      
      - name: Verify version exists
        run: |
          if ! git tag | grep -q "^${{ github.event.inputs.version }}$"; then
            echo "Version ${{ github.event.inputs.version }} not found"
            exit 1
          fi
      
      - name: Check production readiness
        run: |
          # Verify all required checks passed on this version
          echo "Validating version ${{ github.event.inputs.version }}"

  # ============================================
  # Production Deployment
  # ============================================
  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [validate]
    environment:
      name: production
      url: https://api.banking.example.com
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.inputs.version }}
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.PROD_AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.PROD_AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
          role-to-assume: ${{ secrets.PROD_DEPLOYMENT_ROLE }}
          role-duration-seconds: 3600
      
      - name: Backup current production
        run: |
          # Create snapshot of current production state
          aws ecs describe-services \
            --cluster banking-api-prod \
            --services api-service > backup-${{ github.run_id }}.json
          
          aws s3 cp backup-${{ github.run_id }}.json \
            s3://banking-backups/ecs-states/
      
      - name: Run database migrations (if needed)
        run: |
          npm ci
          npm run migrate:prod
        env:
          DB_HOST: ${{ secrets.PROD_DB_HOST }}
          DB_NAME: ${{ secrets.PROD_DB_NAME }}
          DB_USER: ${{ secrets.PROD_DB_USER }}
          DB_PASSWORD: ${{ secrets.PROD_DB_PASSWORD }}
        continue-on-error: false
      
      - name: Blue-Green Deployment
        run: |
          # Update task definition with new image
          TASK_DEF=$(aws ecs describe-task-definition \
            --task-definition banking-api-prod \
            --query 'taskDefinition' \
            --output json)
          
          NEW_TASK_DEF=$(echo $TASK_DEF | jq \
            --arg IMAGE "ghcr.io/${{ env.IMAGE_NAME }}:${{ github.event.inputs.version }}" \
            '.containerDefinitions[0].image = $IMAGE')
          
          aws ecs register-task-definition --cli-input-json "$NEW_TASK_DEF"
          
          # Update service
          aws ecs update-service \
            --cluster banking-api-prod \
            --service api-service \
            --task-definition banking-api-prod
      
      - name: Wait for deployment
        run: |
          aws ecs wait services-stable \
            --cluster banking-api-prod \
            --services api-service \
            --max-attempts 60
      
      - name: Smoke tests
        if: ${{ !github.event.inputs.skip-tests }}
        run: |
          curl -f https://api.banking.example.com/health || exit 1
          curl -f https://api.banking.example.com/api/status || exit 1
      
      - name: Load test
        if: ${{ !github.event.inputs.skip-tests }}
        run: |
          npm install -g artillery
          artillery quick \
            --count 100 \
            --num 10 \
            https://api.banking.example.com/health
      
      - name: Monitor error rates
        run: |
          # Check CloudWatch metrics for 5 minutes
          sleep 300
          
          ERROR_RATE=$(aws cloudwatch get-metric-statistics \
            --namespace Banking/API \
            --metric-name ErrorRate \
            --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S) \
            --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
            --period 300 \
            --statistics Average \
            --query 'Datapoints[0].Average' \
            --output text)
          
          if (( $(echo "$ERROR_RATE > 1.0" | bc -l) )); then
            echo "High error rate detected: $ERROR_RATE%"
            exit 1
          fi
      
      - name: Create deployment record
        run: |
          # Log deployment to audit trail
          echo '{
            "version": "${{ github.event.inputs.version }}",
            "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'",
            "deployer": "${{ github.actor }}",
            "run_id": "${{ github.run_id }}"
          }' | aws s3 cp - s3://banking-audit/deployments/$(date +%Y%m%d-%H%M%S).json
      
      - name: Notify team
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "🚀 Banking API deployed to PRODUCTION",
              "blocks": [
                {
                  "type": "header",
                  "text": {
                    "type": "plain_text",
                    "text": "🚀 Production Deployment Complete"
                  }
                },
                {
                  "type": "section",
                  "fields": [
                    {
                      "type": "mrkdwn",
                      "text": "*Version:*\n${{ github.event.inputs.version }}"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*Deployed by:*\n${{ github.actor }}"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*URL:*\nhttps://api.banking.example.com"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*Run ID:*\n${{ github.run_id }}"
                    }
                  ]
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

  # ============================================
  # Rollback (if deployment fails)
  # ============================================
  rollback:
    name: Rollback Deployment
    runs-on: ubuntu-latest
    if: failure()
    needs: [deploy]
    
    steps:
      - name: Restore previous version
        run: |
          aws s3 cp \
            s3://banking-backups/ecs-states/backup-${{ github.run_id }}.json \
            ./backup.json
          
          # Restore previous task definition
          aws ecs update-service \
            --cluster banking-api-prod \
            --service api-service \
            --task-definition $(cat backup.json | jq -r '.services[0].taskDefinition')
      
      - name: Notify rollback
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "⚠️ Production deployment FAILED - Rolling back",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "⚠️ *Production Deployment Failed*\nRolling back to previous version\n*Version attempted:* ${{ github.event.inputs.version }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### package.json Scripts

```json
{
  "scripts": {
    "test": "jest",
    "test:unit": "jest --testPathPattern=unit",
    "test:integration": "jest --testPathPattern=integration",
    "test:e2e": "jest --testPathPattern=e2e",
    "test:coverage": "jest --coverage",
    "lint": "eslint src/**/*.js",
    "format:check": "prettier --check src/**/*.js",
    "format": "prettier --write src/**/*.js",
    "type-check": "tsc --noEmit",
    "migrate:dev": "knex migrate:latest --env development",
    "migrate:staging": "knex migrate:latest --env staging",
    "migrate:prod": "knex migrate:latest --env production"
  }
}
```

---

## 🎯 Best Practices

### 1. Secrets Management

```yaml
# Never hardcode secrets
steps:
  - name: Deploy
    env:
      DB_PASSWORD: ${{ secrets.DB_PASSWORD }}  # ✅ Use secrets
      API_KEY: my-hardcoded-key                # ❌ Never do this
```

### 2. Caching Dependencies

```yaml
- name: Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version: '18'
    cache: 'npm'  # Automatically caches node_modules

# Or manual caching
- name: Cache dependencies
  uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

### 3. Matrix Testing

```yaml
jobs:
  test:
    strategy:
      matrix:
        node-version: [16.x, 18.x, 20.x]
        os: [ubuntu-latest, windows-latest, macos-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
```

### 4. Conditional Execution

```yaml
- name: Deploy to production
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  run: npm run deploy

- name: Skip on draft PR
  if: github.event.pull_request.draft == false
  run: npm test
```

### 5. Parallel Jobs

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
  
  lint:
    runs-on: ubuntu-latest
  
  build:
    needs: [test, lint]  # Wait for both
    runs-on: ubuntu-latest
```

---

## 📚 Common Interview Questions

### Q1: How do you secure secrets in GitHub Actions?

**Answer**:
1. **GitHub Secrets**: Store in repository settings
2. **Environment secrets**: Scope to specific environments
3. **OIDC**: Use temporary credentials (AWS, Azure)
4. **Secrets scanning**: Enable secret scanning in repo
5. **Never log secrets**: Use `::add-mask::` if needed

### Q2: Explain caching in GitHub Actions

**Answer**:
Caching saves time by reusing dependencies:
- **npm cache**: `cache: 'npm'` in setup-node
- **Custom cache**: Use `actions/cache` with key based on lock file hash
- **Docker layers**: Use BuildKit cache with `cache-from` and `cache-to`

### Q3: How do you handle failed deployments?

**Answer**:
1. **Rollback job**: Triggered on failure with `if: failure()`
2. **Health checks**: Verify deployment before considering success
3. **Gradual rollout**: Deploy to small percentage first
4. **Monitoring**: Alert on error rate spikes
5. **Blue-green**: Keep old version running until new is validated

### Q4: What's the difference between `needs` and `if` conditions?

**Answer**:
- **`needs`**: Job dependency (waits for completion)
- **`if`**: Conditional execution (skips if false)

```yaml
job-b:
  needs: job-a      # Wait for job-a
  if: success()     # Only if job-a succeeded
```

### Q5: How do you optimize workflow execution time?

**Answer**:
1. **Caching**: Cache dependencies and build outputs
2. **Parallelization**: Run independent jobs concurrently
3. **Matrix strategy**: Test multiple versions in parallel
4. **Reusable workflows**: Share common workflows
5. **Self-hosted runners**: Faster than GitHub-hosted

---

## ✅ Summary & Key Takeaways

### Core Concepts

1. **Workflows**: Automated processes triggered by events
2. **Jobs**: Run in parallel by default
3. **Steps**: Sequential tasks within a job
4. **Secrets**: Encrypted environment variables
5. **Environments**: Deployment targets with protection rules

### Banking CI/CD Pipeline

```
Push → Lint → Test → Security → Build → Deploy Dev → Deploy Staging → Manual Prod
        ↓      ↓       ↓         ↓          ↓              ↓
     ESLint  Jest   Snyk    Docker    ECS Dev      E2E Tests
```

### Production Checklist

```yaml
✅ CI/CD Readiness:
  - [ ] Automated tests (unit, integration, E2E)
  - [ ] Security scanning (Snyk, CodeQL, Trivy)
  - [ ] Code coverage threshold (>80%)
  - [ ] Linting and formatting checks
  - [ ] Docker image vulnerability scanning
  - [ ] Database migration automation
  - [ ] Multi-environment deployments
  - [ ] Health checks and smoke tests
  - [ ] Performance/load testing
  - [ ] Rollback mechanism
  - [ ] Deployment notifications (Slack)
  - [ ] Audit trail logging
```

### Key Actions

```yaml
actions/checkout@v4        # Clone repository
actions/setup-node@v4      # Setup Node.js
actions/cache@v3           # Cache dependencies
docker/build-push-action@v5 # Build Docker images
aws-actions/configure-aws-credentials@v4 # AWS access
```

---

**Status**: ✅ Complete with production-ready GitHub Actions workflows for banking API deployment with comprehensive quality gates!
