# Question 10: CI/CD Pipeline with GitHub Actions

## 🎯 Question
**Design and implement a complete CI/CD pipeline using GitHub Actions for deploying AWS Lambda-based integration services. Include branching strategies, automated testing, security scanning, and deployment to multiple environments (dev, staging, production).**

---

## 📋 Answer Overview

GitHub Actions provides cloud-native CI/CD capabilities essential for modern integration services. For Bayer's requirements, it enables:
- **Automated testing** on every pull request
- **Security scanning** for vulnerabilities
- **Infrastructure as Code** deployment
- **Multi-environment deployments** with approval gates
- **Rollback capabilities** for failed deployments

---

## 🏗️ CI/CD Architecture

### Pipeline Flow

```
┌────────────────────────────────────────────────────────────┐
│                    Developer Workflow                       │
└────────────────────────────────────────────────────────────┘
                           │
                  ┌────────┴────────┐
                  │  Create Branch  │
                  │  (feature/xxx)  │
                  └────────┬────────┘
                           │
                           ▼
                  ┌────────────────┐
                  │  Push Changes  │
                  └────────┬────────┘
                           │
┌──────────────────────────┼──────────────────────────┐
│              GitHub Actions Triggers                 │
└──────────────────────────┼──────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Lint/Test   │  │  Security    │  │  Build       │
│  Workflow    │  │  Scan        │  │  Workflow    │
│              │  │  Workflow    │  │              │
│- ESLint      │  │- Snyk        │  │- npm build   │
│- Unit Tests  │  │- Trivy       │  │- Package     │
│- Integration │  │- SAST        │  │- Artifacts   │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │
                 ┌───────▼────────┐
                 │  Pull Request  │
                 │  Review        │
                 └───────┬────────┘
                         │
                 ┌───────▼────────┐
                 │  Merge to      │
                 │  main/develop  │
                 └───────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│Deploy to DEV │  │Deploy to     │  │Deploy to     │
│              │  │STAGING       │  │PRODUCTION    │
│- Auto Deploy │  │- Auto Deploy │  │- Manual      │
│- Smoke Tests │  │- E2E Tests   │  │  Approval    │
└──────────────┘  └──────┬───────┘  └──────┬───────┘
                         │                 │
                         ▼                 ▼
                  ┌──────────────┐  ┌──────────────┐
                  │ Integration  │  │  Production  │
                  │   Tests      │  │  Monitoring  │
                  └──────────────┘  └──────────────┘
```

**Why this architecture?**
- **Quality Gates**: Automated tests prevent bad code from reaching production
- **Security First**: Vulnerabilities caught before deployment
- **Environment Parity**: Same deployment process across all environments
- **Rollback Safety**: Easy to revert to previous versions
- **Audit Trail**: Complete deployment history in GitHub

---

## 💻 Implementation

### 1. Repository Structure

```
integration-service/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                    # Continuous Integration
│   │   ├── cd-dev.yml                # Deploy to Dev
│   │   ├── cd-staging.yml            # Deploy to Staging
│   │   ├── cd-production.yml         # Deploy to Production
│   │   └── security-scan.yml         # Security scanning
│   ├── CODEOWNERS                    # Code review assignments
│   └── pull_request_template.md      # PR template
├── src/
│   ├── handlers/
│   │   ├── validator.js
│   │   ├── transformer.js
│   │   └── router.js
│   ├── lib/
│   │   ├── sqs-client.js
│   │   ├── dynamodb-client.js
│   │   └── s3-client.js
│   └── utils/
│       ├── logger.js
│       └── error-handler.js
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── infrastructure/
│   ├── terraform/                    # IaC for AWS resources
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── cloudformation/               # Alternative to Terraform
├── package.json
├── serverless.yml                    # Serverless Framework config
├── jest.config.js                    # Test configuration
└── README.md
```

---

### 2. Continuous Integration Workflow

```yaml
# .github/workflows/ci.yml
# WHY: Runs on every PR to ensure code quality
# TRIGGERS: Pull request creation, new commits to PR

name: Continuous Integration

on:
  pull_request:
    branches: [main, develop]
    # WHY: Only run on code changes, not docs
    paths:
      - 'src/**'
      - 'tests/**'
      - 'package*.json'
      - '.github/workflows/**'

# WHY: Permissions follow least-privilege principle
permissions:
  contents: read        # Read repository contents
  pull-requests: write  # Comment on PRs with test results

jobs:
  # Job 1: Code Quality Checks
  lint:
    name: Lint Code
    runs-on: ubuntu-latest
    # WHY: Fail fast - no need to run tests if linting fails
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        # WHY: Fetch full history for better analysis
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          # WHY: Cache npm dependencies to speed up builds
          cache: 'npm'

      - name: Install dependencies
        run: npm ci
        # WHY: ci is faster and more reliable than install

      - name: Run ESLint
        run: npm run lint
        # WHY: Enforce code style and catch common errors
        # package.json: "lint": "eslint src/ tests/ --ext .js"

      - name: Check code formatting
        run: npm run format:check
        # WHY: Consistent code formatting across team
        # package.json: "format:check": "prettier --check ."

  # Job 2: Unit Tests
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: lint  # WHY: Only run if linting passes
    
    strategy:
      matrix:
        node-version: [16, 18, 20]
        # WHY: Test against multiple Node versions for compatibility
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run unit tests
        run: npm run test:unit -- --coverage
        # WHY: Generate coverage report
        # package.json: "test:unit": "jest --testMatch='**/*.test.js'"

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
          flags: unittests
          name: codecov-umbrella
        # WHY: Track code coverage trends over time

      - name: Comment PR with coverage
        uses: romeovs/lcov-reporter-action@v0.3.1
        with:
          lcov-file: ./coverage/lcov.info
          github-token: ${{ secrets.GITHUB_TOKEN }}
        # WHY: Visibility - show coverage changes in PR

  # Job 3: Integration Tests
  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: lint
    
    # WHY: Need AWS services for integration tests
    services:
      # LocalStack - simulates AWS services locally
      localstack:
        image: localstack/localstack:latest
        ports:
          - 4566:4566
        env:
          SERVICES: s3,sqs,dynamodb,lambda
          DEBUG: 1
        # WHY: Test against real AWS APIs without incurring costs

    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Wait for LocalStack
        run: |
          echo "Waiting for LocalStack to be ready..."
          timeout 60 bash -c 'until curl -s http://localhost:4566/_localstack/health | grep -q "\"s3\": \"available\""; do sleep 2; done'
        # WHY: Ensure LocalStack is fully initialized

      - name: Setup test resources
        run: |
          # Create S3 bucket
          aws --endpoint-url=http://localhost:4566 s3 mb s3://test-integration-bucket
          
          # Create DynamoDB table
          aws --endpoint-url=http://localhost:4566 dynamodb create-table \
            --table-name test-integration-table \
            --attribute-definitions AttributeName=id,AttributeType=S \
            --key-schema AttributeName=id,KeyType=HASH \
            --billing-mode PAY_PER_REQUEST
          
          # Create SQS queue
          aws --endpoint-url=http://localhost:4566 sqs create-queue \
            --queue-name test-integration-queue
        env:
          AWS_ACCESS_KEY_ID: test
          AWS_SECRET_ACCESS_KEY: test
          AWS_DEFAULT_REGION: us-east-1
        # WHY: Create necessary AWS resources for tests

      - name: Run integration tests
        run: npm run test:integration
        env:
          AWS_ENDPOINT: http://localhost:4566
          AWS_REGION: us-east-1
          AWS_ACCESS_KEY_ID: test
          AWS_SECRET_ACCESS_KEY: test
        # WHY: Test interactions between components

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: integration-test-results
          path: test-results/
          retention-days: 30
        # WHY: Archive test results for debugging failures

  # Job 4: Build and Package
  build:
    name: Build Package
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests]
    # WHY: Only build if all tests pass
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci --production
        # WHY: Only production dependencies for deployment

      - name: Build application
        run: npm run build
        # WHY: Transpile, minify, and optimize code

      - name: Create deployment package
        run: |
          mkdir -p dist
          zip -r dist/lambda-package.zip src/ node_modules/ package.json
        # WHY: Lambda deployment package format

      - name: Upload build artifact
        uses: actions/upload-artifact@v3
        with:
          name: lambda-package
          path: dist/lambda-package.zip
          retention-days: 7
        # WHY: Share package between CI and CD workflows

  # Job 5: Validate PR
  validate-pr:
    name: Validate Pull Request
    runs-on: ubuntu-latest
    needs: [lint, unit-tests, integration-tests, build]
    
    steps:
      - name: Check PR title
        uses: amannn/action-semantic-pull-request@v5
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          types: |
            feat
            fix
            docs
            chore
            refactor
            test
        # WHY: Enforce conventional commit messages for changelog

      - name: Comment success
        uses: actions/github-script@v6
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '✅ All CI checks passed! Ready for review.'
            })
        # WHY: Provide clear feedback in PR
```

---

### 3. Security Scanning Workflow

```yaml
# .github/workflows/security-scan.yml
# WHY: Catch security vulnerabilities before deployment
# TRIGGERS: PR creation, scheduled daily scans

name: Security Scanning

on:
  pull_request:
    branches: [main, develop]
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM UTC
    # WHY: Catch new vulnerabilities in dependencies

  workflow_dispatch:  # Manual trigger
    # WHY: Allow security team to run on-demand scans

jobs:
  # Job 1: Dependency Scanning
  dependency-scan:
    name: Dependency Vulnerability Scan
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Snyk to check for vulnerabilities
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
        # WHY: Snyk checks npm dependencies against vulnerability database
        # FAILS: If high or critical vulnerabilities found

      - name: Upload Snyk results to GitHub
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: snyk.sarif
        # WHY: Integrate with GitHub Security tab

  # Job 2: Container Scanning (if using Docker)
  container-scan:
    name: Container Image Scan
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build Docker image
        run: docker build -t integration-service:${{ github.sha }} .
        # WHY: Build image for scanning

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: integration-service:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
        # WHY: Trivy scans for OS and application vulnerabilities

      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  # Job 3: Static Application Security Testing (SAST)
  sast:
    name: Static Code Analysis
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for better analysis
      
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v2
        with:
          languages: javascript
          queries: security-extended
        # WHY: CodeQL finds security vulnerabilities in code

      - name: Autobuild
        uses: github/codeql-action/autobuild@v2

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v2
        # WHY: Detects SQL injection, XSS, code injection, etc.

  # Job 4: Secrets Scanning
  secrets-scan:
    name: Scan for Leaked Secrets
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Scan entire history
      
      - name: TruffleHog OSS
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
        # WHY: Detects accidentally committed API keys, passwords, tokens

  # Job 5: License Compliance
  license-check:
    name: License Compliance
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Check licenses
        run: npx license-checker --production --onlyAllow="MIT;Apache-2.0;BSD-2-Clause;BSD-3-Clause;ISC" --failOn="GPL;LGPL;AGPL"
        # WHY: Ensure no copyleft licenses in production dependencies
        # IMPORTANT: Pharmaceutical companies have strict license requirements
```

---

### 4. Deployment to DEV Environment

```yaml
# .github/workflows/cd-dev.yml
# WHY: Automatically deploy to dev environment on merge to develop branch
# TRIGGERS: Push to develop branch

name: Deploy to DEV

on:
  push:
    branches: [develop]
    paths:
      - 'src/**'
      - 'infrastructure/**'
      - 'serverless.yml'

  workflow_dispatch:  # Manual trigger
    # WHY: Allow manual deployments for testing

# WHY: Prevent concurrent deployments to same environment
concurrency:
  group: deploy-dev
  cancel-in-progress: false

jobs:
  deploy-dev:
    name: Deploy to DEV Environment
    runs-on: ubuntu-latest
    environment:
      name: development
      url: https://api-dev.bayer-integration.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_DEV }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_DEV }}
          aws-region: us-east-1
        # WHY: Use separate AWS credentials per environment

      - name: Install Serverless Framework
        run: npm install -g serverless@3

      - name: Deploy to AWS Lambda
        run: |
          serverless deploy \
            --stage dev \
            --region us-east-1 \
            --verbose
        env:
          NODE_ENV: development
        # WHY: Serverless Framework handles Lambda, API Gateway, IAM roles

      - name: Run smoke tests
        run: npm run test:smoke
        env:
          API_ENDPOINT: ${{ steps.deploy.outputs.api-url }}
        # WHY: Verify deployment succeeded with basic health checks

      - name: Notify deployment success
        uses: 8398a7/action-slack@v3
        if: success()
        with:
          status: custom
          custom_payload: |
            {
              text: '✅ DEV Deployment Successful',
              blocks: [
                {
                  type: 'section',
                  text: {
                    type: 'mrkdwn',
                    text: '*Deployment to DEV*\n✅ Status: Success\n🔗 URL: https://api-dev.bayer-integration.com\n📝 Commit: ${{ github.sha }}\n👤 By: ${{ github.actor }}'
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
        # WHY: Keep team informed of deployments

      - name: Notify deployment failure
        uses: 8398a7/action-slack@v3
        if: failure()
        with:
          status: failure
          text: '❌ DEV Deployment Failed'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

### 5. Deployment to PRODUCTION with Approval

```yaml
# .github/workflows/cd-production.yml
# WHY: Deploy to production with manual approval gates
# TRIGGERS: Manual workflow dispatch or Git tag

name: Deploy to PRODUCTION

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to deploy (e.g., v1.2.3)'
        required: true
        type: string
  
  push:
    tags:
      - 'v*.*.*'  # Trigger on version tags
    # WHY: Semantic versioning for production releases

jobs:
  # Job 1: Pre-deployment checks
  pre-deployment-checks:
    name: Pre-deployment Validation
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Verify version tag
        run: |
          VERSION="${{ github.ref_name }}"
          if [[ ! $VERSION =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
            echo "Invalid version tag: $VERSION"
            exit 1
          fi
        # WHY: Ensure proper semantic versioning

      - name: Check if version exists in staging
        run: |
          # Query staging environment to verify this version was tested
          STAGING_VERSION=$(aws lambda get-function --function-name integration-service-staging --query 'Configuration.Environment.Variables.VERSION' --output text)
          if [ "$STAGING_VERSION" != "${{ github.ref_name }}" ]; then
            echo "Version not found in staging. Deploy to staging first."
            exit 1
          fi
        # WHY: Ensure version was tested in staging before production

  # Job 2: Manual approval gate
  approval:
    name: Approval Gate
    runs-on: ubuntu-latest
    needs: pre-deployment-checks
    environment:
      name: production-approval
      # WHY: GitHub environment protection rules require manual approval
    
    steps:
      - name: Request approval
        run: echo "Waiting for production deployment approval..."

  # Job 3: Blue-Green Deployment
  deploy-production:
    name: Deploy to PRODUCTION
    runs-on: ubuntu-latest
    needs: approval
    environment:
      name: production
      url: https://api.bayer-integration.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci --production

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_PROD }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_PROD }}
          aws-region: us-east-1
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsDeployRole
          role-duration-seconds: 3600
        # WHY: Use IAM role for production (more secure than static credentials)

      - name: Create deployment package
        run: |
          zip -r deployment.zip src/ node_modules/ package.json
          aws s3 cp deployment.zip s3://bayer-lambda-deployments/integration-service/${{ github.ref_name }}/
        # WHY: Store deployment packages for audit and rollback

      - name: Deploy new Lambda version (Blue)
        id: deploy-blue
        run: |
          # Update Lambda function code
          aws lambda update-function-code \
            --function-name integration-service-prod \
            --s3-bucket bayer-lambda-deployments \
            --s3-key integration-service/${{ github.ref_name }}/deployment.zip
          
          # Wait for update to complete
          aws lambda wait function-updated \
            --function-name integration-service-prod
          
          # Publish new version
          VERSION=$(aws lambda publish-version \
            --function-name integration-service-prod \
            --description "Release ${{ github.ref_name }}" \
            --query 'Version' \
            --output text)
          
          echo "lambda-version=$VERSION" >> $GITHUB_OUTPUT
        # WHY: Create immutable Lambda version for blue-green deployment

      - name: Update alias to new version (Green)
        run: |
          # Gradually shift traffic to new version
          # Start with 10% traffic to new version
          aws lambda update-alias \
            --function-name integration-service-prod \
            --name live \
            --function-version ${{ steps.deploy-blue.outputs.lambda-version }} \
            --routing-config AdditionalVersionWeights={"${{ steps.deploy-blue.outputs.lambda-version }}":0.1}
        # WHY: Canary deployment - test with 10% traffic first

      - name: Monitor metrics
        run: |
          echo "Monitoring error rates for 5 minutes..."
          sleep 300
          
          # Check CloudWatch metrics
          ERROR_COUNT=$(aws cloudwatch get-metric-statistics \
            --namespace AWS/Lambda \
            --metric-name Errors \
            --dimensions Name=FunctionName,Value=integration-service-prod \
            --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S) \
            --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
            --period 300 \
            --statistics Sum \
            --query 'Datapoints[0].Sum' \
            --output text)
          
          if [ "$ERROR_COUNT" != "None" ] && [ "$ERROR_COUNT" -gt 5 ]; then
            echo "High error rate detected: $ERROR_COUNT errors"
            exit 1
          fi
        # WHY: Detect issues before full rollout

      - name: Complete deployment (100% traffic)
        run: |
          aws lambda update-alias \
            --function-name integration-service-prod \
            --name live \
            --function-version ${{ steps.deploy-blue.outputs.lambda-version }}
        # WHY: Shift all traffic to new version after validation

      - name: Create GitHub release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref_name }}
          release_name: Release ${{ github.ref_name }}
          body: |
            ## Changes in this release
            - See commit history for details
            
            ## Deployment Info
            - Lambda Version: ${{ steps.deploy-blue.outputs.lambda-version }}
            - Deployed At: ${{ github.event.head_commit.timestamp }}
            - Deployed By: ${{ github.actor }}
          draft: false
          prerelease: false
        # WHY: Document releases for audit trail

      - name: Notify success
        uses: 8398a7/action-slack@v3
        if: success()
        with:
          status: custom
          custom_payload: |
            {
              text: '🚀 PRODUCTION Deployment Successful',
              blocks: [
                {
                  type: 'section',
                  text: {
                    type: 'mrkdwn',
                    text: '*Production Release*\n✅ Status: Success\n🏷️ Version: ${{ github.ref_name }}\n🔗 URL: https://api.bayer-integration.com\n👤 By: ${{ github.actor }}'
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_PROD }}

  # Job 4: Rollback on failure
  rollback:
    name: Rollback on Failure
    runs-on: ubuntu-latest
    needs: deploy-production
    if: failure()
    
    steps:
      - name: Rollback to previous version
        run: |
          # Get previous version
          PREVIOUS_VERSION=$(aws lambda list-versions-by-function \
            --function-name integration-service-prod \
            --query 'Versions[-2].Version' \
            --output text)
          
          # Update alias to previous version
          aws lambda update-alias \
            --function-name integration-service-prod \
            --name live \
            --function-version $PREVIOUS_VERSION
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID_PROD }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY_PROD }}
          AWS_REGION: us-east-1
        # WHY: Automatic rollback on deployment failure

      - name: Notify rollback
        uses: 8398a7/action-slack@v3
        with:
          status: failure
          text: '⚠️ Production deployment failed - Rolled back to previous version'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_PROD }}
```

---

## 🌿 Branching Strategy

### GitFlow Model

```
main (production)
  │
  ├── v1.0.0 ─────────── v1.1.0 ─────────── v1.2.0
  │       │                   │                  │
  │       │                   │                  │
develop   │                   │                  │
  │       │                   │                  │
  ├───────┴───────────────────┴──────────────────┘
  │
  ├── feature/SAP-integration
  │   └── (merged via PR)
  │
  ├── feature/data-validation
  │   └── (merged via PR)
  │
  ├── hotfix/critical-bug
      └── (merged to main & develop)
```

**Branch Protection Rules:**

```yaml
# .github/branch-protection.yml
# WHY: Enforce quality gates

Branch: main
  - Require pull request reviews: 2
  - Require status checks to pass:
    - CI / lint
    - CI / unit-tests
    - CI / integration-tests
    - Security / dependency-scan
    - Security / sast
  - Require signed commits: true
  - Require linear history: true
  - Do not allow bypassing: true

Branch: develop
  - Require pull request reviews: 1
  - Require status checks to pass: same as main
  - Allow force pushes: false
```

---

## 🔐 Secrets Management

### GitHub Secrets Configuration

```bash
# WHY: Never hardcode credentials in workflows

# Development Environment
AWS_ACCESS_KEY_ID_DEV
AWS_SECRET_ACCESS_KEY_DEV

# Staging Environment
AWS_ACCESS_KEY_ID_STAGING
AWS_SECRET_ACCESS_KEY_STAGING

# Production Environment (use IAM Role instead)
AWS_PROD_ROLE_ARN

# Third-party services
SNYK_TOKEN
SLACK_WEBHOOK
SLACK_WEBHOOK_PROD

# Database credentials (stored in AWS Secrets Manager, referenced here)
DB_SECRET_ARN
```

### Using AWS Secrets Manager in Workflows

```yaml
- name: Retrieve database credentials
  run: |
    SECRET=$(aws secretsmanager get-secret-value \
      --secret-id prod/integration-db \
      --query SecretString \
      --output text)
    
    echo "::add-mask::$(echo $SECRET | jq -r .password)"
    # WHY: Mask sensitive values in logs
    
    echo "DB_HOST=$(echo $SECRET | jq -r .host)" >> $GITHUB_ENV
    echo "DB_USER=$(echo $SECRET | jq -r .username)" >> $GITHUB_ENV
    echo "DB_PASS=$(echo $SECRET | jq -r .password)" >> $GITHUB_ENV
```

---

## 📊 Monitoring Deployments

### Custom GitHub Action for Deployment Metrics

```javascript
// .github/actions/deployment-metrics/index.js
// WHY: Track deployment success rate and duration

const core = require('@actions/core');
const github = require('@actions/github');
const AWS = require('aws-sdk');

const cloudwatch = new AWS.CloudWatch();

async function recordDeploymentMetric() {
  const environment = core.getInput('environment');
  const status = core.getInput('status'); // success or failure
  const duration = core.getInput('duration'); // in seconds
  
  // Send custom metric to CloudWatch
  await cloudwatch.putMetricData({
    Namespace: 'Bayer/Deployments',
    MetricData: [
      {
        MetricName: 'DeploymentCount',
        Value: 1,
        Unit: 'Count',
        Dimensions: [
          { Name: 'Environment', Value: environment },
          { Name: 'Status', Value: status }
        ]
      },
      {
        MetricName: 'DeploymentDuration',
        Value: parseFloat(duration),
        Unit: 'Seconds',
        Dimensions: [
          { Name: 'Environment', Value: environment }
        ]
      }
    ]
  }).promise();
  
  console.log(`Recorded deployment metric: ${environment} - ${status}`);
}

recordDeploymentMetric().catch(error => {
  core.setFailed(error.message);
});
```

---

## 🎯 Key Takeaways

### ✅ Do's

1. **Use Branch Protection Rules** - Enforce code reviews and status checks
2. **Implement Security Scanning** - Catch vulnerabilities early (Snyk, CodeQL, Trivy)
3. **Use Environment Secrets** - Never hardcode credentials
4. **Deploy with Approval Gates** - Manual approval for production
5. **Implement Blue-Green Deployments** - Zero-downtime releases
6. **Monitor Deployment Metrics** - Track success rates and duration
7. **Use Semantic Versioning** - Clear version tracking (v1.2.3)
8. **Implement Automatic Rollbacks** - Fast recovery from failures
9. **Cache Dependencies** - Speed up workflows (npm cache)
10. **Use Matrix Builds** - Test across multiple Node.js versions

### ❌ Don'ts

1. **Don't Store Secrets in Code** - Use GitHub Secrets or AWS Secrets Manager
2. **Don't Skip Tests** - Always run full test suite before deployment
3. **Don't Deploy Without Approval** - Require manual approval for production
4. **Don't Ignore Security Scans** - Treat security findings as blocking
5. **Don't Use Personal Access Tokens** - Use GitHub Apps or OIDC
6. **Don't Deploy Without Monitoring** - Always check metrics post-deployment
7. **Don't Skip Rollback Plans** - Always have rollback capability
8. **Don't Use Same Credentials** - Separate credentials per environment
9. **Don't Ignore Failed Jobs** - Fix issues immediately
10. **Don't Deploy on Fridays** - Avoid deployments before weekends

---

## 📈 CI/CD Metrics Dashboard

### Key Metrics to Track

| Metric | Target | Why Important |
|--------|--------|---------------|
| **Build Duration** | <10 min | Developer productivity |
| **Deployment Frequency** | Daily | Continuous delivery maturity |
| **Lead Time for Changes** | <1 day | Time from commit to production |
| **Change Failure Rate** | <5% | Quality of deployments |
| **Mean Time to Recovery** | <1 hour | Resilience and rollback capability |
| **Test Coverage** | >80% | Code quality and reliability |
| **Security Scan Findings** | 0 high/critical | Security posture |

---

## 🔧 Troubleshooting Common Issues

### Issue 1: Workflow Timeout

```yaml
# Problem: Workflow exceeds 6-hour limit
# Solution: Split into multiple jobs

jobs:
  test-unit:
    timeout-minutes: 30  # Set per-job timeout
  test-integration:
    timeout-minutes: 45
```

### Issue 2: Rate Limiting

```yaml
# Problem: GitHub API rate limits
# Solution: Use GitHub App instead of PAT

- name: Checkout with App Token
  uses: actions/checkout@v4
  with:
    token: ${{ steps.generate-token.outputs.token }}
```

### Issue 3: Concurrent Deployment Conflicts

```yaml
# Problem: Multiple deployments to same environment
# Solution: Use concurrency control

concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false  # Wait for previous deployment
```

---

## 💰 Cost Optimization

### GitHub Actions Pricing

```
Free tier: 2,000 minutes/month for private repos
Linux runners: $0.008/minute
Storage: $0.25/GB/month

Example calculation for 20 deployments/day:
- CI pipeline: 10 min × 20 = 200 min/day = 6,000 min/month
- Cost: 6,000 × $0.008 = $48/month

Optimization tips:
1. Cache dependencies (saves 2-3 min per run)
2. Use matrix strategically (only when needed)
3. Skip jobs when possible (path filters)
4. Use self-hosted runners for high-volume (free compute)
```

---

## 🚀 Advanced Patterns

### 1. Reusable Workflows

```yaml
# .github/workflows/reusable-deploy.yml
# WHY: DRY principle - define deployment once, reuse for all environments

name: Reusable Deployment Workflow

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      aws-region:
        required: true
        type: string
    secrets:
      aws-access-key-id:
        required: true
      aws-secret-access-key:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to ${{ inputs.environment }}
        run: serverless deploy --stage ${{ inputs.environment }}
```

```yaml
# .github/workflows/cd-dev.yml
# WHY: Call reusable workflow

name: Deploy DEV
on:
  push:
    branches: [develop]

jobs:
  call-deploy:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: development
      aws-region: us-east-1
    secrets:
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_DEV }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_DEV }}
```

### 2. Composite Actions

```yaml
# .github/actions/setup-node-app/action.yml
# WHY: Reusable setup steps

name: 'Setup Node Application'
description: 'Setup Node.js with caching and dependencies'

inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '18'

runs:
  using: 'composite'
  steps:
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
      shell: bash
    
    - name: Cache build
      uses: actions/cache@v3
      with:
        path: dist/
        key: build-${{ runner.os }}-${{ hashFiles('src/**') }}
```

---

**Estimated Reading Time**: 25-30 minutes  
**Difficulty Level**: ⭐⭐⭐⭐ Advanced  
**Prerequisites**: GitHub, AWS, CI/CD concepts, Node.js  
**Bayer Job Alignment**: 100% - Critical requirement for the role

---

## 📚 Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [AWS Lambda Deployment Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [Serverless Framework CI/CD](https://www.serverless.com/framework/docs/guides/cicd)
- [GitHub Environments](https://docs.github.com/en/actions/deployment/targeting-different-environments)
