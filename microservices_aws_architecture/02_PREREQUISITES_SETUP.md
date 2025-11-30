# Prerequisites & Environment Setup - Enterprise Microservices on AWS

## 📋 Overview

This document outlines all prerequisites, tools, and environment setup required to build, deploy, and manage enterprise microservices on AWS.

---

## 🎯 Prerequisites Checklist

### Essential Requirements
- ✅ AWS Account with appropriate permissions
- ✅ Credit card for AWS billing (Free Tier available)
- ✅ Domain name (optional, but recommended for production)
- ✅ Basic understanding of cloud computing concepts
- ✅ Command-line interface experience
- ✅ Git version control knowledge

### Recommended Skills
- ⭐ AWS services familiarity (EC2, VPC, IAM)
- ⭐ Container technology (Docker)
- ⭐ Programming (Node.js, Python, or Java)
- ⭐ REST API concepts
- ⭐ Database basics (SQL and NoSQL)
- ⭐ CI/CD pipelines understanding

---

## 🔧 Required Tools & Software

### 1. AWS CLI (Command Line Interface)

```bash
# Installation on macOS
$ brew install awscli

# Installation on Linux
$ curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
$ unzip awscliv2.zip
$ sudo ./aws/install

# Installation on Windows (PowerShell as Administrator)
PS> msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi

# Verify installation
$ aws --version
# Expected output: aws-cli/2.13.25 Python/3.11.5 ...

# Configure AWS CLI
$ aws configure
AWS Access Key ID [None]: YOUR_ACCESS_KEY_ID
AWS Secret Access Key [None]: YOUR_SECRET_ACCESS_KEY
Default region name [None]: us-east-1
Default output format [None]: json

# Test configuration
$ aws sts get-caller-identity
{
    "UserId": "AIDAI...",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/your-user"
}
```

### 2. Docker & Docker Compose

```bash
# Installation on macOS
$ brew install --cask docker

# Installation on Linux (Ubuntu/Debian)
$ sudo apt-get update
$ sudo apt-get install -y docker.io docker-compose
$ sudo systemctl start docker
$ sudo systemctl enable docker
$ sudo usermod -aG docker $USER

# Installation on Windows
# Download Docker Desktop from: https://www.docker.com/products/docker-desktop

# Verify installation
$ docker --version
# Expected: Docker version 24.0.5, build ced0996

$ docker-compose --version
# Expected: docker-compose version 2.20.2

# Test Docker
$ docker run hello-world
# Should download and run hello-world container
```

### 3. Terraform (Infrastructure as Code)

```bash
# Installation on macOS
$ brew tap hashicorp/tap
$ brew install hashicorp/tap/terraform

# Installation on Linux
$ wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
$ echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
$ sudo apt update && sudo apt install terraform

# Installation on Windows (Chocolatey)
PS> choco install terraform

# Verify installation
$ terraform --version
# Expected: Terraform v1.6.0

# Initialize Terraform (in your project directory)
$ terraform init
```

### 4. kubectl (Kubernetes CLI) - For EKS

```bash
# Installation on macOS
$ brew install kubectl

# Installation on Linux
$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
$ sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Installation on Windows (Chocolatey)
PS> choco install kubernetes-cli

# Verify installation
$ kubectl version --client
# Expected: Client Version: v1.28.2

# Configure kubectl for EKS (after cluster creation)
$ aws eks update-kubeconfig --region us-east-1 --name my-cluster
```

### 5. eksctl (EKS Cluster Management)

```bash
# Installation on macOS
$ brew tap weaveworks/tap
$ brew install weaveworks/tap/eksctl

# Installation on Linux
$ curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
$ sudo mv /tmp/eksctl /usr/local/bin

# Installation on Windows (Chocolatey)
PS> choco install eksctl

# Verify installation
$ eksctl version
# Expected: 0.165.0
```

### 6. AWS SAM CLI (Serverless Application Model)

```bash
# Installation on macOS
$ brew tap aws/tap
$ brew install aws-sam-cli

# Installation on Linux
$ pip3 install aws-sam-cli

# Installation on Windows (MSI Installer)
# Download from: https://github.com/aws/aws-sam-cli/releases/latest

# Verify installation
$ sam --version
# Expected: SAM CLI, version 1.100.0

# Initialize a sample SAM project
$ sam init
```

### 7. Node.js & npm

```bash
# Installation on macOS
$ brew install node

# Installation on Linux
$ curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
$ sudo apt-get install -y nodejs

# Installation on Windows
# Download from: https://nodejs.org/

# Verify installation
$ node --version
# Expected: v20.9.0

$ npm --version
# Expected: 10.1.0
```

### 8. Python 3 & pip

```bash
# Installation on macOS
$ brew install python@3.11

# Installation on Linux
$ sudo apt-get update
$ sudo apt-get install python3 python3-pip

# Installation on Windows
# Download from: https://www.python.org/downloads/

# Verify installation
$ python3 --version
# Expected: Python 3.11.5

$ pip3 --version
# Expected: pip 23.2.1
```

### 9. Git

```bash
# Installation on macOS
$ brew install git

# Installation on Linux
$ sudo apt-get install git

# Installation on Windows
# Download from: https://git-scm.com/

# Verify installation
$ git --version
# Expected: git version 2.42.0

# Configure Git
$ git config --global user.name "Your Name"
$ git config --global user.email "your.email@example.com"
```

### 10. jq (JSON processor)

```bash
# Installation on macOS
$ brew install jq

# Installation on Linux
$ sudo apt-get install jq

# Installation on Windows (Chocolatey)
PS> choco install jq

# Verify installation
$ jq --version
# Expected: jq-1.7

# Test jq
$ echo '{"name":"John","age":30}' | jq '.name'
# Expected: "John"
```

---

## 🔑 AWS Account Setup

### 1. Create IAM User for Development

```bash
# Create IAM user via CLI
$ aws iam create-user --user-name microservices-dev

# Attach AdministratorAccess policy (for development only)
$ aws iam attach-user-policy \
    --user-name microservices-dev \
    --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Create access keys
$ aws iam create-access-key --user-name microservices-dev > credentials.json

# View credentials (SAVE THESE SECURELY!)
$ cat credentials.json
{
    "AccessKey": {
        "AccessKeyId": "AKIA...",
        "SecretAccessKey": "wJalrXUtn...",
        "Status": "Active",
        "CreateDate": "2025-11-15T10:00:00Z"
    }
}

# Configure AWS CLI with new credentials
$ aws configure --profile microservices
AWS Access Key ID [None]: <paste AccessKeyId>
AWS Secret Access Key [None]: <paste SecretAccessKey>
Default region name [None]: us-east-1
Default output format [None]: json

# Set default profile
$ export AWS_PROFILE=microservices

# Or use in commands
$ aws s3 ls --profile microservices
```

### 2. Production IAM Setup (Least Privilege)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ECSFullAccess",
      "Effect": "Allow",
      "Action": [
        "ecs:*",
        "ecr:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "LambdaFullAccess",
      "Effect": "Allow",
      "Action": [
        "lambda:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DynamoDBAccess",
      "Effect": "Allow",
      "Action": [
        "dynamodb:CreateTable",
        "dynamodb:DescribeTable",
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:Query",
        "dynamodb:Scan",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/microservices-*"
    },
    {
      "Sid": "RDSAccess",
      "Effect": "Allow",
      "Action": [
        "rds:CreateDBInstance",
        "rds:DescribeDBInstances",
        "rds:ModifyDBInstance",
        "rds:DeleteDBInstance",
        "rds:CreateDBSnapshot"
      ],
      "Resource": "*"
    },
    {
      "Sid": "VPCAccess",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVpc",
        "ec2:DescribeVpcs",
        "ec2:CreateSubnet",
        "ec2:DescribeSubnets",
        "ec2:CreateSecurityGroup",
        "ec2:DescribeSecurityGroups"
      ],
      "Resource": "*"
    },
    {
      "Sid": "IAMReadAccess",
      "Effect": "Allow",
      "Action": [
        "iam:GetRole",
        "iam:GetRolePolicy",
        "iam:ListRoles"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchAccess",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "cloudwatch:PutMetricData"
      ],
      "Resource": "*"
    }
  ]
}
```

### 3. Enable Required AWS Services

```bash
# Enable ECS
$ aws ecs create-cluster --cluster-name microservices-dev

# Enable ECR (Container Registry)
$ aws ecr create-repository --repository-name user-service
$ aws ecr create-repository --repository-name order-service
$ aws ecr create-repository --repository-name product-service
$ aws ecr create-repository --repository-name payment-service

# Enable CloudWatch Logs
$ aws logs create-log-group --log-group-name /aws/ecs/microservices
$ aws logs create-log-group --log-group-name /aws/lambda/microservices

# Enable Systems Manager Parameter Store
$ aws ssm put-parameter \
    --name "/microservices/dev/database/host" \
    --value "localhost" \
    --type "String"

# Enable Secrets Manager
$ aws secretsmanager create-secret \
    --name microservices/dev/db-credentials \
    --secret-string '{"username":"admin","password":"SecurePassword123!"}'
```

---

## 🌐 VPC & Network Prerequisites

### VPC Setup Script

```bash
#!/bin/bash
# vpc-setup.sh - Creates VPC with public and private subnets

# Variables
VPC_CIDR="10.0.0.0/16"
REGION="us-east-1"
ENV="dev"

echo "Creating VPC..."
VPC_ID=$(aws ec2 create-vpc \
    --cidr-block $VPC_CIDR \
    --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=microservices-$ENV-vpc}]" \
    --query 'Vpc.VpcId' \
    --output text)

echo "VPC Created: $VPC_ID"

# Enable DNS hostnames
aws ec2 modify-vpc-attribute \
    --vpc-id $VPC_ID \
    --enable-dns-hostnames

# Create Internet Gateway
echo "Creating Internet Gateway..."
IGW_ID=$(aws ec2 create-internet-gateway \
    --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=microservices-$ENV-igw}]" \
    --query 'InternetGateway.InternetGatewayId' \
    --output text)

# Attach IGW to VPC
aws ec2 attach-internet-gateway \
    --vpc-id $VPC_ID \
    --internet-gateway-id $IGW_ID

# Create Public Subnets
echo "Creating Public Subnets..."
PUBLIC_SUBNET_1=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.1.0/24 \
    --availability-zone ${REGION}a \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=microservices-$ENV-public-1a}]" \
    --query 'Subnet.SubnetId' \
    --output text)

PUBLIC_SUBNET_2=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.2.0/24 \
    --availability-zone ${REGION}b \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=microservices-$ENV-public-1b}]" \
    --query 'Subnet.SubnetId' \
    --output text)

# Create Private Subnets
echo "Creating Private Subnets..."
PRIVATE_SUBNET_1=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.11.0/24 \
    --availability-zone ${REGION}a \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=microservices-$ENV-private-1a}]" \
    --query 'Subnet.SubnetId' \
    --output text)

PRIVATE_SUBNET_2=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.12.0/24 \
    --availability-zone ${REGION}b \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=microservices-$ENV-private-1b}]" \
    --query 'Subnet.SubnetId' \
    --output text)

# Create NAT Gateway
echo "Creating NAT Gateway..."
EIP_ID=$(aws ec2 allocate-address \
    --domain vpc \
    --query 'AllocationId' \
    --output text)

NAT_GW_ID=$(aws ec2 create-nat-gateway \
    --subnet-id $PUBLIC_SUBNET_1 \
    --allocation-id $EIP_ID \
    --tag-specifications "ResourceType=natgateway,Tags=[{Key=Name,Value=microservices-$ENV-nat}]" \
    --query 'NatGateway.NatGatewayId' \
    --output text)

echo "Waiting for NAT Gateway to be available..."
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW_ID

# Create Route Tables
echo "Creating Route Tables..."
PUBLIC_RT=$(aws ec2 create-route-table \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=microservices-$ENV-public-rt}]" \
    --query 'RouteTable.RouteTableId' \
    --output text)

PRIVATE_RT=$(aws ec2 create-route-table \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=microservices-$ENV-private-rt}]" \
    --query 'RouteTable.RouteTableId' \
    --output text)

# Create Routes
aws ec2 create-route \
    --route-table-id $PUBLIC_RT \
    --destination-cidr-block 0.0.0.0/0 \
    --gateway-id $IGW_ID

aws ec2 create-route \
    --route-table-id $PRIVATE_RT \
    --destination-cidr-block 0.0.0.0/0 \
    --nat-gateway-id $NAT_GW_ID

# Associate Subnets with Route Tables
aws ec2 associate-route-table --subnet-id $PUBLIC_SUBNET_1 --route-table-id $PUBLIC_RT
aws ec2 associate-route-table --subnet-id $PUBLIC_SUBNET_2 --route-table-id $PUBLIC_RT
aws ec2 associate-route-table --subnet-id $PRIVATE_SUBNET_1 --route-table-id $PRIVATE_RT
aws ec2 associate-route-table --subnet-id $PRIVATE_SUBNET_2 --route-table-id $PRIVATE_RT

echo "VPC Setup Complete!"
echo "VPC ID: $VPC_ID"
echo "Public Subnets: $PUBLIC_SUBNET_1, $PUBLIC_SUBNET_2"
echo "Private Subnets: $PRIVATE_SUBNET_1, $PRIVATE_SUBNET_2"
echo "NAT Gateway: $NAT_GW_ID"

# Save to parameter store
aws ssm put-parameter --name "/microservices/$ENV/vpc-id" --value "$VPC_ID" --type String --overwrite
aws ssm put-parameter --name "/microservices/$ENV/public-subnet-1" --value "$PUBLIC_SUBNET_1" --type String --overwrite
aws ssm put-parameter --name "/microservices/$ENV/public-subnet-2" --value "$PUBLIC_SUBNET_2" --type String --overwrite
aws ssm put-parameter --name "/microservices/$ENV/private-subnet-1" --value "$PRIVATE_SUBNET_1" --type String --overwrite
aws ssm put-parameter --name "/microservices/$ENV/private-subnet-2" --value "$PRIVATE_SUBNET_2" --type String --overwrite
```

---

## 📦 Project Structure Setup

```bash
# Create project directory
$ mkdir -p microservices-aws-project
$ cd microservices-aws-project

# Initialize Git repository
$ git init
$ git config user.name "Your Name"
$ git config user.email "your.email@example.com"

# Create directory structure
$ mkdir -p services/{user-service,product-service,order-service,payment-service}
$ mkdir -p infrastructure/{terraform,cloudformation}
$ mkdir -p scripts
$ mkdir -p docs
$ mkdir -p tests/{unit,integration,e2e}
$ mkdir -p .github/workflows

# Create .gitignore
$ cat > .gitignore << 'EOF'
# Dependencies
node_modules/
__pycache__/
*.pyc
venv/
.env

# IDE
.vscode/
.idea/
*.swp
*.swo

# AWS
.aws/
credentials.json
*.pem

# Terraform
**/.terraform/
*.tfstate
*.tfstate.backup
.terraform.lock.hcl

# OS
.DS_Store
Thumbs.db

# Logs
*.log
npm-debug.log*

# Build
dist/
build/
*.zip
EOF

# Project structure
$ tree -L 2
.
├── services/
│   ├── user-service/
│   ├── product-service/
│   ├── order-service/
│   └── payment-service/
├── infrastructure/
│   ├── terraform/
│   └── cloudformation/
├── scripts/
├── docs/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── .github/
│   └── workflows/
└── .gitignore
```

---

## 🐳 Docker Setup & Testing

```bash
# Test Docker installation
$ docker run hello-world

# Pull common base images
$ docker pull node:20-alpine
$ docker pull python:3.11-slim
$ docker pull nginx:alpine
$ docker pull postgres:15-alpine
$ docker pull redis:7-alpine

# Create Docker network for local development
$ docker network create microservices-network

# Test with sample containers
$ docker run -d \
    --name postgres-dev \
    --network microservices-network \
    -e POSTGRES_PASSWORD=postgres \
    -p 5432:5432 \
    postgres:15-alpine

$ docker run -d \
    --name redis-dev \
    --network microservices-network \
    -p 6379:6379 \
    redis:7-alpine

# Verify running containers
$ docker ps
CONTAINER ID   IMAGE                COMMAND                  STATUS
abc123         postgres:15-alpine   "docker-entrypoint.s…"   Up 10 seconds
def456         redis:7-alpine       "docker-entrypoint.s…"   Up 5 seconds

# Test connectivity
$ docker exec -it postgres-dev psql -U postgres -c "SELECT version();"
# Should return PostgreSQL version

# Clean up test containers
$ docker stop postgres-dev redis-dev
$ docker rm postgres-dev redis-dev
```

---

## 🧪 Environment Validation Script

```bash
#!/bin/bash
# validate-environment.sh - Validates all prerequisites

echo "🔍 Validating Development Environment..."
echo ""

# Color codes
GREEN='\033[0;32m'
RED='\033[0;31m'
NC='\033[0m' # No Color

check_command() {
    if command -v $1 &> /dev/null; then
        echo -e "${GREEN}✓${NC} $1 is installed: $($1 --version | head -n1)"
    else
        echo -e "${RED}✗${NC} $1 is NOT installed"
        return 1
    fi
}

check_aws_auth() {
    if aws sts get-caller-identity &> /dev/null; then
        ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
        echo -e "${GREEN}✓${NC} AWS CLI is configured (Account: $ACCOUNT)"
    else
        echo -e "${RED}✗${NC} AWS CLI is NOT configured properly"
        return 1
    fi
}

check_docker_running() {
    if docker ps &> /dev/null; then
        echo -e "${GREEN}✓${NC} Docker is running"
    else
        echo -e "${RED}✗${NC} Docker is NOT running"
        return 1
    fi
}

# Check all tools
check_command aws
check_command docker
check_command docker-compose
check_command terraform
check_command kubectl
check_command node
check_command npm
check_command python3
check_command pip3
check_command git
check_command jq

echo ""
echo "🔑 Checking AWS Configuration..."
check_aws_auth

echo ""
echo "🐳 Checking Docker..."
check_docker_running

echo ""
echo "✅ Environment Validation Complete!"
```

```bash
# Make script executable and run
$ chmod +x validate-environment.sh
$ ./validate-environment.sh
```

---

## 📚 Additional Resources

### AWS Free Tier Resources

```
Service                  Free Tier Allowance
─────────────────────────────────────────────
EC2                      750 hours/month (t2.micro)
Lambda                   1M requests + 400K GB-seconds/month
DynamoDB                 25 GB storage + 25 read/write capacity units
RDS                      750 hours/month (db.t2.micro)
S3                       5 GB storage
CloudWatch               10 custom metrics + 10 alarms
API Gateway              1M API calls/month
SNS                      1M publishes/month
SQS                      1M requests/month
```

### Useful AWS CLI Commands

```bash
# List all running EC2 instances
$ aws ec2 describe-instances \
    --query 'Reservations[*].Instances[*].[InstanceId,State.Name,InstanceType]' \
    --output table

# List all Lambda functions
$ aws lambda list-functions --query 'Functions[*].[FunctionName,Runtime,MemorySize]' --output table

# List all DynamoDB tables
$ aws dynamodb list-tables --output table

# List all S3 buckets
$ aws s3 ls

# Get current costs
$ aws ce get-cost-and-usage \
    --time-period Start=2025-11-01,End=2025-11-15 \
    --granularity MONTHLY \
    --metrics "UnblendedCost" \
    --query 'ResultsByTime[*].[TimePeriod.Start,Total.UnblendedCost.Amount]' \
    --output table
```

### IDE Recommendations

#### VS Code Extensions
- AWS Toolkit
- Docker
- Terraform
- Kubernetes
- YAML
- ESLint
- Prettier
- GitLens

```bash
# Install VS Code extensions via CLI
$ code --install-extension amazonwebservices.aws-toolkit-vscode
$ code --install-extension ms-azuretools.vscode-docker
$ code --install-extension hashicorp.terraform
$ code --install-extension ms-kubernetes-tools.vscode-kubernetes-tools
```

---

## 🎓 Next Steps

After completing this setup:

1. ✅ Verify all tools are installed correctly
2. ✅ Configure AWS credentials and test access
3. ✅ Create VPC and networking infrastructure
4. ✅ Set up local Docker environment
5. ✅ Initialize Git repository
6. 👉 Proceed to **[03_VPC_NETWORKING_SETUP.md](./03_VPC_NETWORKING_SETUP.md)**

---

## ❓ Troubleshooting

### Common Issues

#### Issue: AWS CLI not configured
```bash
# Solution:
$ aws configure
# Enter your credentials when prompted
```

#### Issue: Docker daemon not running
```bash
# macOS/Windows: Start Docker Desktop application
# Linux:
$ sudo systemctl start docker
$ sudo systemctl enable docker
```

#### Issue: Permission denied errors with Docker
```bash
# Linux: Add user to docker group
$ sudo usermod -aG docker $USER
$ newgrp docker
# Then log out and log back in
```

#### Issue: Terraform state lock
```bash
# Force unlock (use with caution)
$ terraform force-unlock <LOCK_ID>
```

#### Issue: kubectl not connecting to EKS
```bash
# Update kubeconfig
$ aws eks update-kubeconfig --region us-east-1 --name my-cluster

# Test connection
$ kubectl cluster-info
```

---

## 💰 Cost Estimation

### Development Environment (Free Tier Usage)
- **Monthly Cost**: $0 - $50
  - Lambda: Free (under 1M requests)
  - DynamoDB: Free (under 25 GB)
  - S3: Free (under 5 GB)
  - EC2: Minimal (using t2.micro)

### Staging Environment
- **Monthly Cost**: $200 - $500
  - ECS Fargate: $100-200
  - RDS: $50-100
  - ALB: $20-30
  - Data Transfer: $20-50
  - Other services: $50-100

### Production Environment
- **Monthly Cost**: $1,000 - $5,000+
  - ECS/EKS: $500-2000
  - RDS Multi-AZ: $200-800
  - DynamoDB: $100-500
  - ALB/CloudFront: $100-300
  - Data Transfer: $100-500
  - Monitoring: $50-200
  - Other services: $150-700

---

**Last Updated**: November 15, 2025  
**Version**: 1.0.0  
**Next:** [VPC & Networking Setup](./03_VPC_NETWORKING_SETUP.md)
