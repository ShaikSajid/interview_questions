# Prerequisites Setup - Serverless Order Processing Microservice

## 📋 Overview

This guide covers all the prerequisites needed before starting the implementation of the serverless order processing microservice. Follow each step carefully to ensure a smooth development experience.

---

## 🎯 What You'll Need

### Required Knowledge
- **AWS Services**: Basic understanding of Lambda, API Gateway, DynamoDB, SQS, SNS
- **Programming**: JavaScript/TypeScript proficiency
- **Command Line**: Comfortable with terminal/PowerShell
- **JSON**: Understanding of JSON structure and syntax
- **REST APIs**: Understanding of RESTful principles
- **Git**: Basic version control knowledge

### Time Commitment
- **Setup Time**: 1-2 hours
- **Learning Curve**: Beginner-friendly with documentation

---

## 1️⃣ AWS Account Setup

### 1.1 Create AWS Account

```powershell
# Visit AWS Console
# https://aws.amazon.com/free/

# Steps:
# 1. Click "Create an AWS Account"
# 2. Enter email address
# 3. Choose account name
# 4. Verify email
# 5. Enter payment information (required, but using free tier services)
# 6. Verify identity (phone verification)
# 7. Select Support Plan (choose "Basic Support - Free")
```

### 1.2 Enable MFA (Multi-Factor Authentication)

```
Why: Security best practice to protect your account
How:
1. Go to IAM Console → Dashboard
2. Click "Activate MFA on your root account"
3. Choose "Virtual MFA device"
4. Use Google Authenticator or Authy app
5. Scan QR code
6. Enter two consecutive MFA codes
```

### 1.3 Verify Account Limits

```powershell
# Check your service quotas
# https://console.aws.amazon.com/servicequotas/

# Important limits to check:
# - Lambda concurrent executions: Default 1000
# - API Gateway requests: Default 10,000/sec
# - DynamoDB tables: Default 256 per region
# - SQS queues: Unlimited (practically)
```

---

## 2️⃣ IAM User Setup

### 2.1 Create IAM User for Development

**⚠️ IMPORTANT**: Never use root account for day-to-day operations

```javascript
// User Configuration
const iamUser = {
  username: 'serverless-developer',
  accessType: 'Programmatic and AWS Console',
  permissions: [
    'AdministratorAccess'  // For development only
    // In production, use least privilege policies
  ]
};
```

### 2.2 Step-by-Step IAM User Creation

```powershell
# Via AWS Console:
# 1. Navigate to IAM Console
# https://console.aws.amazon.com/iam/

# 2. Click "Users" → "Add User"

# 3. User Details:
#    - User name: serverless-developer
#    - Select: "Provide user access to the AWS Management Console"
#    - Console password: Custom password
#    - Uncheck "Users must create a new password at next sign-in"

# 4. Set Permissions:
#    - Select "Attach policies directly"
#    - Search and select "AdministratorAccess"
#    - (For production, create custom policies with least privilege)

# 5. Review and Create

# 6. Download Credentials:
#    - Click "Download .csv" button
#    - Save securely (you'll need this for AWS CLI)
```

### 2.3 Access Keys Configuration

```powershell
# After creating user:
# 1. Click on the user name
# 2. Go to "Security credentials" tab
# 3. Click "Create access key"
# 4. Choose "Command Line Interface (CLI)"
# 5. Check acknowledgment box
# 6. Click "Create access key"
# 7. Download or copy:
#    - Access Key ID
#    - Secret Access Key
# ⚠️ Store these securely! Secret key shown only once.
```

### 2.4 Create Password Policy

```javascript
// Recommended password policy
const passwordPolicy = {
  minimumLength: 14,
  requireSymbols: true,
  requireNumbers: true,
  requireUppercase: true,
  requireLowercase: true,
  allowUsersToChange: true,
  expirePasswords: false,  // or set to 90 days
  preventReuse: 5
};

// Apply in IAM Console:
// IAM → Account settings → Password policy
```

---

## 3️⃣ AWS CLI Installation

### 3.1 Windows Installation (PowerShell)

```powershell
# Method 1: Using MSI Installer (Recommended)
# Download from: https://awscli.amazonaws.com/AWSCLIV2.msi
# Run the installer and follow prompts

# Method 2: Using Chocolatey
choco install awscli -y

# Verify installation
aws --version
# Expected output: aws-cli/2.x.x Python/3.x.x Windows/10 exe/AMD64
```

### 3.2 Alternative: macOS Installation

```bash
# Using Homebrew
brew install awscli

# Using pkg installer
# Download from: https://awscli.amazonaws.com/AWSCLIV2.pkg
# Run installer

# Verify
aws --version
```

### 3.3 Alternative: Linux Installation

```bash
# For Ubuntu/Debian
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify
aws --version
```

---

## 4️⃣ Configure AWS CLI

### 4.1 Basic Configuration

```powershell
# Run AWS configure
aws configure

# Enter when prompted:
# AWS Access Key ID: <YOUR_ACCESS_KEY_ID>
# AWS Secret Access Key: <YOUR_SECRET_ACCESS_KEY>
# Default region name: us-east-1
# Default output format: json
```

### 4.2 Named Profiles (Multiple Accounts)

```powershell
# Configure dev profile
aws configure --profile dev

# Configure staging profile
aws configure --profile staging

# Configure production profile
aws configure --profile production

# Use specific profile
aws s3 ls --profile dev

# Set default profile
$env:AWS_PROFILE="dev"
```

### 4.3 Configuration Files Location

```powershell
# Windows:
# Credentials: C:\Users\<username>\.aws\credentials
# Config: C:\Users\<username>\.aws\config

# View current configuration
cat ~\.aws\credentials
cat ~\.aws\config
```

### 4.4 Verify AWS CLI Configuration

```powershell
# Test AWS CLI
aws sts get-caller-identity

# Expected output:
# {
#     "UserId": "AIDAXXXXXXXXXXXXXXXXX",
#     "Account": "123456789012",
#     "Arn": "arn:aws:iam::123456789012:user/serverless-developer"
# }

# List S3 buckets (if any)
aws s3 ls

# Get current region
aws configure get region
```

---

## 5️⃣ Node.js and npm Setup

### 5.1 Install Node.js (Windows)

```powershell
# Method 1: Official Installer (Recommended)
# Download LTS version from: https://nodejs.org/
# Version required: 18.x or 20.x
# Run installer and follow prompts

# Method 2: Using Chocolatey
choco install nodejs-lts -y

# Verify installation
node --version
# Expected: v18.x.x or v20.x.x

npm --version
# Expected: 9.x.x or higher
```

### 5.2 Alternative: Using nvm (Node Version Manager)

```powershell
# Install nvm for Windows
# Download from: https://github.com/coreybutler/nvm-windows/releases

# After installation:
# Install Node.js 18
nvm install 18

# Use Node.js 18
nvm use 18

# Verify
node --version
```

### 5.3 Configure npm

```powershell
# Set npm registry (default is fine)
npm config get registry
# Should output: https://registry.npmjs.org/

# Increase memory for large projects (optional)
npm config set maxsockets 5

# Set init defaults
npm config set init.author.name "Your Name"
npm config set init.author.email "your.email@example.com"
npm config set init.license "MIT"
```

---

## 6️⃣ Install Serverless Framework

### 6.1 Global Installation

```powershell
# Install Serverless Framework globally
npm install -g serverless

# Verify installation
serverless --version
# Expected: Framework Core: 3.x.x

# Alternative short command
sls --version
```

### 6.2 Serverless Framework Configuration

```powershell
# Configure Serverless with AWS credentials
# (Already done via AWS CLI, but can also do directly)
serverless config credentials --provider aws --key <ACCESS_KEY> --secret <SECRET_KEY>

# Verify Serverless can access AWS
sls info --verbose
```

### 6.3 Install Useful Serverless Plugins

```powershell
# Create a temporary directory to install plugins globally
mkdir C:\serverless-plugins
cd C:\serverless-plugins

# Install common plugins
npm install -g serverless-offline
npm install -g serverless-plugin-typescript
npm install -g serverless-dotenv-plugin
npm install -g serverless-webpack
```

---

## 7️⃣ Install Development Tools

### 7.1 Visual Studio Code

```powershell
# Download from: https://code.visualstudio.com/
# Run installer

# Or using Chocolatey:
choco install vscode -y
```

### 7.2 VS Code Extensions

```javascript
// Recommended extensions for serverless development
const extensions = [
  {
    name: 'AWS Toolkit',
    id: 'amazonwebservices.aws-toolkit-vscode',
    purpose: 'AWS service integration, Lambda debugging'
  },
  {
    name: 'ESLint',
    id: 'dbaeumer.vscode-eslint',
    purpose: 'JavaScript/TypeScript linting'
  },
  {
    name: 'Prettier',
    id: 'esbenp.prettier-vscode',
    purpose: 'Code formatting'
  },
  {
    name: 'YAML',
    id: 'redhat.vscode-yaml',
    purpose: 'YAML syntax and validation'
  },
  {
    name: 'GitLens',
    id: 'eamodio.gitlens',
    purpose: 'Git integration and history'
  },
  {
    name: 'Thunder Client',
    id: 'rangav.vscode-thunder-client',
    purpose: 'API testing (Postman alternative)'
  },
  {
    name: 'DynamoDB Viewer',
    id: 'rafwilinski.dynamodb-viewer',
    purpose: 'View DynamoDB tables in VS Code'
  }
];

// Install via command line:
code --install-extension amazonwebservices.aws-toolkit-vscode
code --install-extension dbaeumer.vscode-eslint
code --install-extension esbenp.prettier-vscode
code --install-extension redhat.vscode-yaml
code --install-extension eamodio.gitlens
code --install-extension rangav.vscode-thunder-client
code --install-extension rafwilinski.dynamodb-viewer
```

### 7.3 Git Installation

```powershell
# Download from: https://git-scm.com/download/win
# Run installer

# Or using Chocolatey:
choco install git -y

# Configure Git
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Verify
git --version
```

---

## 8️⃣ Install Additional Tools

### 8.1 Postman (API Testing)

```powershell
# Download from: https://www.postman.com/downloads/
# Or use Thunder Client extension in VS Code (recommended)

# Or using Chocolatey:
choco install postman -y
```

### 8.2 NoSQL Workbench (DynamoDB)

```powershell
# Download from AWS:
# https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/workbench.settingup.html

# Benefits:
# - Visual data modeling
# - Sample data generation
# - Query and scan operations
# - Connection to local DynamoDB
```

### 8.3 LocalStack (Optional - Local AWS)

```powershell
# Install Docker Desktop first
# Download from: https://www.docker.com/products/docker-desktop/

# Install LocalStack
pip install localstack

# Or using Docker:
docker pull localstack/localstack

# Start LocalStack
localstack start

# Verify
curl http://localhost:4566/_localstack/health
```

---

## 9️⃣ Create Project Directory Structure

### 9.1 Initialize Project

```powershell
# Navigate to your projects folder
cd C:\Users\user\Documents\interview_preparation

# Already have serverless_microservice_project folder
cd serverless_microservice_project

# Initialize npm project
npm init -y

# This creates package.json
```

### 9.2 Install Project Dependencies

```powershell
# Core dependencies
npm install --save aws-sdk
npm install --save uuid
npm install --save joi  # For validation

# Development dependencies
npm install --save-dev typescript
npm install --save-dev @types/node
npm install --save-dev @types/aws-lambda
npm install --save-dev ts-node
npm install --save-dev eslint
npm install --save-dev prettier
npm install --save-dev jest
npm install --save-dev @types/jest
npm install --save-dev ts-jest
```

### 9.3 Create TypeScript Configuration

```powershell
# Create tsconfig.json
New-Item -Path "tsconfig.json" -ItemType File
```

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "moduleResolution": "node",
    "allowSyntheticDefaultImports": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

### 9.4 Create ESLint Configuration

```powershell
# Create .eslintrc.json
New-Item -Path ".eslintrc.json" -ItemType File
```

```json
{
  "env": {
    "node": true,
    "es2020": true,
    "jest": true
  },
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended"
  ],
  "parser": "@typescript-eslint/parser",
  "parserOptions": {
    "ecmaVersion": 2020,
    "sourceType": "module"
  },
  "rules": {
    "no-console": "warn",
    "@typescript-eslint/no-explicit-any": "warn",
    "@typescript-eslint/explicit-module-boundary-types": "off"
  }
}
```

### 9.5 Create Prettier Configuration

```powershell
# Create .prettierrc
New-Item -Path ".prettierrc" -ItemType File
```

```json
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false
}
```

### 9.6 Create .gitignore

```powershell
# Create .gitignore
New-Item -Path ".gitignore" -ItemType File
```

```
# Dependencies
node_modules/
package-lock.json

# Build outputs
dist/
build/
*.js.map

# Environment variables
.env
.env.local
.env.*.local

# AWS
.aws-sam/
.serverless/

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Logs
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Test coverage
coverage/
.nyc_output/

# Misc
*.bak
*.tmp
```

---

## 🔟 Verify Complete Setup

### 10.1 Verification Checklist

```powershell
# Run all verification commands

# 1. AWS CLI
aws --version
aws sts get-caller-identity

# 2. Node.js and npm
node --version
npm --version

# 3. Serverless Framework
serverless --version

# 4. Git
git --version

# 5. TypeScript
npx tsc --version

# 6. Project dependencies
npm list --depth=0
```

### 10.2 Verification Script

```javascript
// verification-script.js
const { execSync } = require('child_process');

const checks = [
  { name: 'AWS CLI', command: 'aws --version' },
  { name: 'AWS Authentication', command: 'aws sts get-caller-identity' },
  { name: 'Node.js', command: 'node --version' },
  { name: 'npm', command: 'npm --version' },
  { name: 'Serverless Framework', command: 'serverless --version' },
  { name: 'Git', command: 'git --version' },
  { name: 'TypeScript', command: 'npx tsc --version' },
];

console.log('🔍 Running prerequisite checks...\n');

checks.forEach(({ name, command }) => {
  try {
    const output = execSync(command, { encoding: 'utf-8' });
    console.log(`✅ ${name}: ${output.trim()}`);
  } catch (error) {
    console.log(`❌ ${name}: FAILED`);
    console.log(`   Error: ${error.message}`);
  }
});

console.log('\n✨ Verification complete!');
```

```powershell
# Run verification
node verification-script.js
```

### 10.3 Test AWS Access

```javascript
// test-aws-access.js
const AWS = require('aws-sdk');

async function testAWSAccess() {
  console.log('Testing AWS service access...\n');

  // Test STS (always available)
  try {
    const sts = new AWS.STS();
    const identity = await sts.getCallerIdentity().promise();
    console.log('✅ STS Access: SUCCESS');
    console.log(`   Account: ${identity.Account}`);
    console.log(`   User ARN: ${identity.Arn}`);
  } catch (error) {
    console.log('❌ STS Access: FAILED');
    console.log(`   Error: ${error.message}`);
  }

  // Test Lambda (list functions)
  try {
    const lambda = new AWS.Lambda();
    await lambda.listFunctions({ MaxItems: 1 }).promise();
    console.log('✅ Lambda Access: SUCCESS');
  } catch (error) {
    console.log('❌ Lambda Access: FAILED');
    console.log(`   Error: ${error.message}`);
  }

  // Test DynamoDB (list tables)
  try {
    const dynamodb = new AWS.DynamoDB();
    await dynamodb.listTables({ Limit: 1 }).promise();
    console.log('✅ DynamoDB Access: SUCCESS');
  } catch (error) {
    console.log('❌ DynamoDB Access: FAILED');
    console.log(`   Error: ${error.message}`);
  }

  console.log('\n✨ AWS access test complete!');
}

testAWSAccess().catch(console.error);
```

```powershell
# Run AWS access test
node test-aws-access.js
```

---

## 📚 Additional Resources

### Documentation Links

```javascript
const resources = {
  aws: {
    documentation: 'https://docs.aws.amazon.com/',
    lambda: 'https://docs.aws.amazon.com/lambda/',
    apiGateway: 'https://docs.aws.amazon.com/apigateway/',
    dynamoDB: 'https://docs.aws.amazon.com/dynamodb/',
    sqs: 'https://docs.aws.amazon.com/sqs/',
    sns: 'https://docs.aws.amazon.com/sns/',
  },
  serverless: {
    framework: 'https://www.serverless.com/framework/docs',
    examples: 'https://github.com/serverless/examples',
    plugins: 'https://www.serverless.com/plugins',
  },
  learning: {
    awsTraining: 'https://aws.amazon.com/training/',
    freeCodeCamp: 'https://www.freecodecamp.org/news/tag/aws/',
    youTube: 'AWS Online Tech Talks channel',
  },
};
```

### Cost Monitoring Setup

```powershell
# Enable AWS Cost Explorer
# https://console.aws.amazon.com/cost-management/home#/cost-explorer

# Create billing alert
# 1. Go to CloudWatch Console
# 2. Create alarm on EstimatedCharges metric
# 3. Set threshold (e.g., $10, $50, $100)
# 4. Add SNS topic for email notification
```

---

## 🎯 Next Steps

You've now completed all prerequisites! You should have:

- ✅ AWS account configured with MFA
- ✅ IAM user with programmatic access
- ✅ AWS CLI installed and configured
- ✅ Node.js and npm installed
- ✅ Serverless Framework installed
- ✅ VS Code with extensions
- ✅ Git configured
- ✅ Project directory initialized
- ✅ All dependencies installed

**Ready to proceed?**

👉 [VPC & Networking Setup](./03_VPC_NETWORKING_SETUP.md)

---

## ⚠️ Troubleshooting

### Common Issues

**Issue**: AWS CLI not found after installation
```powershell
# Solution: Add to PATH
$env:Path += ";C:\Program Files\Amazon\AWSCLIV2"
```

**Issue**: npm install fails with EACCES
```powershell
# Solution: Run PowerShell as Administrator
# Or fix npm permissions:
npm config set prefix %APPDATA%\npm
```

**Issue**: Serverless command not found
```powershell
# Solution: Install globally
npm install -g serverless

# Or use npx:
npx serverless --version
```

**Issue**: AWS credentials not working
```powershell
# Solution: Verify credentials file
cat ~\.aws\credentials

# Reconfigure
aws configure
```

---

**Last Updated**: November 12, 2025  
**Version**: 1.0.0
