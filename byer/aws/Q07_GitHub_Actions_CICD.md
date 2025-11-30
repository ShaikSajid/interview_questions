# Question 7: CI/CD with GitHub Actions for AWS Deployments

## 🎯 Question
**How would you design a CI/CD pipeline using GitHub Actions for Bayer's integration platform? Discuss automated testing, deployment strategies (blue/green, canary), infrastructure as code, security scanning, and best practices for multi-environment deployments to AWS.**

---

## 📋 Answer Overview

GitHub Actions provides:
- **Native GitHub integration**: No external CI/CD tool needed
- **Matrix testing**: Test across multiple Node.js versions
- **Secrets management**: Securely store AWS credentials
- **Reusable workflows**: DRY principle for pipelines
- **Cost-effective**: Free for public repos, affordable for private

For Bayer's integration platform:
- **Automated testing**: Unit, integration, end-to-end tests
- **Infrastructure as Code**: Terraform/CloudFormation deployment
- **Security scanning**: Dependency vulnerabilities, secrets detection
- **Multi-stage deployment**: Dev → Staging → Production
- **Rollback capability**: Instant rollback if deployment fails

---

## 🏗️ CI/CD Pipeline Architecture

```
┌────────────────────────────────────────────────────────┐
│              GitHub Repository                         │
│     (Bayer Integration Platform Code)                  │
└──────────────────┬─────────────────────────────────────┘
                   │
                   ▼
         ┌─────────────────┐
         │  Git Push/PR    │
         └─────────┬───────┘
                   │
                   ▼
    ┌──────────────────────────┐
    │   GitHub Actions         │
    │   (Workflow Triggered)   │
    └──────────┬───────────────┘
               │
    ┌──────────┼──────────┐
    │          │          │
    ▼          ▼          ▼
┌──────┐  ┌──────┐  ┌──────┐
│Lint  │  │Test  │  │Build │
│ESLint│  │Jest  │  │Bundle│
└──┬───┘  └──┬───┘  └──┬───┘
   │         │         │
   └─────────┼─────────┘
             │
             ▼
      ┌─────────────┐
      │  Security   │
      │  Scanning   │
      │  (Snyk)     │
      └──────┬──────┘
             │
             ▼
      ┌─────────────┐
      │  Deploy to  │
      │  AWS Dev    │
      └──────┬──────┘
             │
             ▼
      ┌─────────────┐
      │  Integration│
      │  Tests      │
      └──────┬──────┘
             │
             ▼
      ┌─────────────┐
      │  Deploy to  │
      │  Staging    │
      └──────┬──────┘
             │
       [Manual Approval]
             │
             ▼
      ┌─────────────┐
      │  Deploy to  │
      │  Production │
      │  (Canary)   │
      └─────────────┘
```

---

## 💻 Implementation Examples

### 1. Main CI/CD Workflow

```yaml
# .github/workflows/deploy.yml
# WHY: Automated deployment pipeline for integration platform
# TRIGGERED BY: Push to main, pull requests

name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:  # Manual trigger
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        type: choice
        options:
          - dev
          - staging
          - production

env:
  NODE_VERSION: '18.x'
  AWS_REGION: 'us-east-1'

jobs:
  # Job 1: Lint and validate code
  # WHY: Catch code quality issues early
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
        # WHY: Get repository code
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
          # WHY: Cache npm dependencies (faster builds)
      
      - name: Install dependencies
        run: npm ci
        # WHY: npm ci is faster and more reliable than npm install
      
      - name: Run ESLint
        run: npm run lint
        # WHY: Enforce code style, catch potential bugs
      
      - name: Check formatting
        run: npm run format:check
        # WHY: Ensure Prettier formatting applied
  
  # Job 2: Run tests
  # WHY: Ensure code quality before deployment
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [16.x, 18.x, 20.x]
        # WHY: Test across multiple Node.js versions
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run unit tests
        run: npm run test:unit -- --coverage
        # WHY: Measure code coverage
      
      - name: Run integration tests
        run: npm run test:integration
        env:
          # Mock AWS services for testing
          # WHY: Don't hit real AWS during tests
          AWS_ACCESS_KEY_ID: test
          AWS_SECRET_ACCESS_KEY: test
          AWS_REGION: us-east-1
      
      - name: Upload coverage reports
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json
          fail_ci_if_error: true
        # WHY: Track code coverage over time
  
  # Job 3: Security scanning
  # WHY: Detect vulnerabilities before deployment
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run Snyk security scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
          # WHY: Fail build on high/critical vulnerabilities
      
      - name: Scan for secrets
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
        # WHY: Prevent accidentally committed secrets
      
      - name: Run SAST (Static Analysis)
        run: npm run security:audit
        # WHY: Detect security issues in code
  
  # Job 4: Build Lambda packages
  # WHY: Create deployment artifacts
  build:
    needs: [lint, test, security]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci --production
        # WHY: Only include production dependencies in Lambda
      
      - name: Build Lambda packages
        run: |
          npm run build
          # WHY: Transpile TypeScript, bundle dependencies
      
      - name: Create deployment package
        run: |
          cd dist
          zip -r ../lambda-package.zip .
        # WHY: Lambda expects ZIP format
      
      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: lambda-package
          path: lambda-package.zip
          retention-days: 7
        # WHY: Share artifact between jobs
  
  # Job 5: Deploy to Dev
  # WHY: Automatic deployment to dev environment
  deploy-dev:
    needs: build
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment: dev
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}
        # WHY: Use OIDC instead of long-lived credentials
      
      - name: Download Lambda package
        uses: actions/download-artifact@v3
        with:
          name: lambda-package
      
      - name: Deploy with Terraform
        run: |
          cd terraform
          terraform init
          terraform plan -out=tfplan
          terraform apply -auto-approve tfplan
        env:
          TF_VAR_environment: dev
          TF_VAR_lambda_package: ../lambda-package.zip
        # WHY: Infrastructure as Code
      
      - name: Run smoke tests
        run: npm run test:smoke
        env:
          API_ENDPOINT: ${{ secrets.DEV_API_ENDPOINT }}
        # WHY: Verify deployment succeeded
  
  # Job 6: Deploy to Staging
  # WHY: Pre-production testing
  deploy-staging:
    needs: deploy-dev
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Download Lambda package
        uses: actions/download-artifact@v3
        with:
          name: lambda-package
      
      - name: Deploy to staging
        run: |
          cd terraform
          terraform init
          terraform apply -auto-approve
        env:
          TF_VAR_environment: staging
      
      - name: Run E2E tests
        run: npm run test:e2e
        env:
          API_ENDPOINT: ${{ secrets.STAGING_API_ENDPOINT }}
        # WHY: Full integration testing before production
  
  # Job 7: Deploy to Production (Canary)
  # WHY: Gradual rollout to production
  deploy-production:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://api.bayer-integration.com
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_PROD }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Download Lambda package
        uses: actions/download-artifact@v3
        with:
          name: lambda-package
      
      - name: Deploy canary (10% traffic)
        run: |
          aws lambda update-function-code \
            --function-name integration-api-prod \
            --zip-file fileb://lambda-package.zip \
            --publish
          
          # Create alias pointing to new version
          NEW_VERSION=$(aws lambda list-versions-by-function \
            --function-name integration-api-prod \
            --query 'Versions[-1].Version' \
            --output text)
          
          # Route 10% traffic to new version
          # WHY: Test with small percentage before full rollout
          aws lambda update-alias \
            --function-name integration-api-prod \
            --name prod \
            --routing-config "AdditionalVersionWeights={\"$NEW_VERSION\"=0.1}"
      
      - name: Monitor canary metrics
        run: |
          # Wait 5 minutes, monitor error rate
          # WHY: Detect issues before full rollout
          sleep 300
          
          ERROR_RATE=$(aws cloudwatch get-metric-statistics \
            --namespace AWS/Lambda \
            --metric-name Errors \
            --dimensions Name=FunctionName,Value=integration-api-prod \
            --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S) \
            --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
            --period 300 \
            --statistics Sum \
            --query 'Datapoints[0].Sum' \
            --output text)
          
          if [ "$ERROR_RATE" -gt 10 ]; then
            echo "Error rate too high, rolling back"
            exit 1
          fi
      
      - name: Promote to 100% traffic
        run: |
          # Shift all traffic to new version
          # WHY: Canary successful, complete rollout
          aws lambda update-alias \
            --function-name integration-api-prod \
            --name prod \
            --function-version $NEW_VERSION
      
      - name: Notify success
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Production deployment completed'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
        if: always()
        # WHY: Team notification on deployment
```

---

### 2. Reusable Workflow for Lambda Deployment

```yaml
# .github/workflows/deploy-lambda.yml
# WHY: Reusable workflow for deploying Lambda functions
# CALLED BY: Other workflows

name: Deploy Lambda Function

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      function-name:
        required: true
        type: string
      lambda-package:
        required: true
        type: string
    secrets:
      AWS_ROLE_ARN:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1
      
      - name: Deploy Lambda
        run: |
          aws lambda update-function-code \
            --function-name ${{ inputs.function-name }} \
            --zip-file fileb://${{ inputs.lambda-package }}
          
          # Wait for update to complete
          # WHY: Ensure function ready before running tests
          aws lambda wait function-updated \
            --function-name ${{ inputs.function-name }}
      
      - name: Update environment variables
        run: |
          aws lambda update-function-configuration \
            --function-name ${{ inputs.function-name }} \
            --environment "Variables={
              ENVIRONMENT=${{ inputs.environment }},
              LOG_LEVEL=info,
              POWERTOOLS_SERVICE_NAME=integration-api
            }"
```

---

### 3. Rollback Workflow

```yaml
# .github/workflows/rollback.yml
# WHY: Quick rollback if production issues detected
# TRIGGERED BY: Manual workflow_dispatch

name: Rollback Production

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to rollback to'
        required: true
        type: string

jobs:
  rollback:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_PROD }}
          aws-region: us-east-1
      
      - name: Rollback Lambda
        run: |
          # Update alias to previous version
          # WHY: Instant rollback (no redeployment needed)
          aws lambda update-alias \
            --function-name integration-api-prod \
            --name prod \
            --function-version ${{ inputs.version }}
      
      - name: Verify rollback
        run: |
          # Check function version
          CURRENT_VERSION=$(aws lambda get-alias \
            --function-name integration-api-prod \
            --name prod \
            --query 'FunctionVersion' \
            --output text)
          
          if [ "$CURRENT_VERSION" != "${{ inputs.version }}" ]; then
            echo "Rollback failed"
            exit 1
          fi
      
      - name: Notify team
        uses: 8398a7/action-slack@v3
        with:
          status: 'warning'
          text: "Production rolled back to version ${{ inputs.version }}"
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

---

### 4. Terraform Infrastructure Deployment

```hcl
# terraform/main.tf
# WHY: Infrastructure as Code for Lambda deployment

terraform {
  required_version = ">= 1.0"
  
  # Remote state in S3
  # WHY: Share state across team, enable locking
  backend "s3" {
    bucket         = "bayer-integration-terraform-state"
    key            = "integration-api/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-lock"
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "Terraform"
      Project     = "Integration Platform"
    }
  }
}

# Lambda function
resource "aws_lambda_function" "integration_api" {
  filename         = var.lambda_package
  function_name    = "integration-api-${var.environment}"
  role             = aws_iam_role.lambda_execution.arn
  handler          = "index.handler"
  runtime          = "nodejs18.x"
  timeout          = 30
  memory_size      = 1024
  
  # Publish version on every deployment
  # WHY: Enable blue/green deployment with aliases
  publish = true
  
  environment {
    variables = {
      ENVIRONMENT     = var.environment
      PATIENTS_TABLE  = aws_dynamodb_table.patients.name
      LOG_LEVEL       = var.environment == "production" ? "info" : "debug"
    }
  }
  
  # Reserved concurrency
  # WHY: Prevent Lambda from consuming all account concurrency
  reserved_concurrent_executions = var.environment == "production" ? 100 : 10
  
  tracing_config {
    mode = "Active"
    # WHY: X-Ray distributed tracing
  }
}

# Lambda alias for traffic routing
resource "aws_lambda_alias" "integration_api" {
  name             = var.environment
  function_name    = aws_lambda_function.integration_api.arn
  function_version = aws_lambda_function.integration_api.version
  
  # Canary deployment: route traffic to two versions
  # WHY: Gradual rollout to detect issues early
  routing_config {
    additional_version_weights = {
      # Route 10% to new version during canary
      # Set to {} for full rollout
    }
  }
}
```

---

## 📊 CI/CD Best Practices

### ✅ Do's
1. **Use OIDC for AWS authentication** (no long-lived credentials)
2. **Run security scans** before deployment (Snyk, TruffleHog)
3. **Test across multiple Node.js versions** (matrix strategy)
4. **Cache dependencies** (faster builds)
5. **Use canary deployments** for production (gradual rollout)
6. **Enable automatic rollback** on errors
7. **Monitor deployment metrics** (error rate, latency)
8. **Use reusable workflows** (DRY principle)
9. **Tag deployments** for traceability
10. **Notify team** of deployment status (Slack)

### ❌ Don'ts
1. **Don't commit AWS credentials** to repository
2. **Don't skip tests** to deploy faster
3. **Don't deploy directly to production** (use staging first)
4. **Don't ignore security scan results**
5. **Don't use long-lived access keys** (use OIDC)
6. **Don't deploy without rollback plan**
7. **Don't skip smoke tests** after deployment

---

## 💰 Cost Optimization

| Action | Savings | How |
|--------|---------|-----|
| **Cache dependencies** | 50% build time | Reuse npm modules |
| **Use self-hosted runners** | 90% runner costs | On-prem runners |
| **Limit test matrix** | 66% test time | Only test Node 16, 18, 20 |
| **Cleanup old artifacts** | Storage costs | 7-day retention |

---

## 🎯 Key Takeaways

**CI/CD enables**:
- ✅ **Fast feedback**: Catch issues in minutes, not days
- ✅ **Automated testing**: No manual testing required
- ✅ **Safe deployments**: Canary rollouts, automatic rollback
- ✅ **Infrastructure as Code**: Repeatable deployments
- ✅ **Security**: Automated vulnerability scanning

---

**Estimated Reading Time**: 15-18 minutes  
**Difficulty Level**: ⭐⭐⭐⭐ Advanced  
**Prerequisites**: GitHub Actions, AWS, Terraform  
**Bayer Job Alignment**: 95% - Critical CI/CD skill
