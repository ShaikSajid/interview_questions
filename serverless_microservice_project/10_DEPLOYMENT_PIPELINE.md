# Deployment Pipeline - Serverless Order Processing Microservice

## 📋 Overview

Complete CI/CD pipeline using AWS CodePipeline, CodeBuild, and CodeDeploy for automated testing and deployment across multiple stages.

---

## 1️⃣ Pipeline Architecture

```
GitHub → CodePipeline → CodeBuild (Test) → CodeBuild (Deploy Dev) → Manual Approval → Deploy Staging → Manual Approval → Deploy Production
```

---

## 2️⃣ buildspec.yml

```yaml
# buildspec.yml
version: 0.2

phases:
  install:
    runtime-versions:
      nodejs: 18
    commands:
      - echo "Installing dependencies..."
      - npm install -g serverless
      - npm install
  
  pre_build:
    commands:
      - echo "Running linting..."
      - npm run lint
      - echo "Running unit tests..."
      - npm test
      - echo "Building TypeScript..."
      - npm run build
  
  build:
    commands:
      - echo "Packaging application..."
      - serverless package --stage $STAGE
  
  post_build:
    commands:
      - echo "Deploying to $STAGE..."
      - serverless deploy --stage $STAGE --verbose

artifacts:
  files:
    - '**/*'
  name: order-processing-$STAGE-$(date +%Y%m%d-%H%M%S)

cache:
  paths:
    - 'node_modules/**/*'
```

---

## 3️⃣ CodePipeline CloudFormation

```yaml
# infrastructure/cloudformation/pipeline-stack.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'CI/CD Pipeline for Order Processing'

Parameters:
  GitHubRepo:
    Type: String
    Default: 'your-org/order-processing'
  
  GitHubBranch:
    Type: String
    Default: 'main'
  
  GitHubToken:
    Type: String
    NoEcho: true

Resources:
  # ========================================
  # S3 Bucket for Artifacts
  # ========================================
  ArtifactBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub '${AWS::AccountId}-pipeline-artifacts'
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      VersioningConfiguration:
        Status: Enabled
      LifecycleConfiguration:
        Rules:
          - Id: DeleteOldArtifacts
            Status: Enabled
            ExpirationInDays: 30

  # ========================================
  # CodeBuild Project - Test
  # ========================================
  TestProject:
    Type: AWS::CodeBuild::Project
    Properties:
      Name: order-processing-test
      ServiceRole: !GetAtt CodeBuildRole.Arn
      Artifacts:
        Type: CODEPIPELINE
      Environment:
        Type: LINUX_CONTAINER
        ComputeType: BUILD_GENERAL1_SMALL
        Image: aws/codebuild/standard:7.0
        EnvironmentVariables:
          - Name: NODE_ENV
            Value: test
      Source:
        Type: CODEPIPELINE
        BuildSpec: buildspec-test.yml

  # ========================================
  # CodeBuild Project - Deploy Dev
  # ========================================
  DeployDevProject:
    Type: AWS::CodeBuild::Project
    Properties:
      Name: order-processing-deploy-dev
      ServiceRole: !GetAtt CodeBuildRole.Arn
      Artifacts:
        Type: CODEPIPELINE
      Environment:
        Type: LINUX_CONTAINER
        ComputeType: BUILD_GENERAL1_SMALL
        Image: aws/codebuild/standard:7.0
        EnvironmentVariables:
          - Name: STAGE
            Value: dev
      Source:
        Type: CODEPIPELINE
        BuildSpec: buildspec.yml

  # ========================================
  # CodeBuild Project - Deploy Staging
  # ========================================
  DeployStagingProject:
    Type: AWS::CodeBuild::Project
    Properties:
      Name: order-processing-deploy-staging
      ServiceRole: !GetAtt CodeBuildRole.Arn
      Artifacts:
        Type: CODEPIPELINE
      Environment:
        Type: LINUX_CONTAINER
        ComputeType: BUILD_GENERAL1_SMALL
        Image: aws/codebuild/standard:7.0
        EnvironmentVariables:
          - Name: STAGE
            Value: staging
      Source:
        Type: CODEPIPELINE
        BuildSpec: buildspec.yml

  # ========================================
  # CodeBuild Project - Deploy Production
  # ========================================
  DeployProductionProject:
    Type: AWS::CodeBuild::Project
    Properties:
      Name: order-processing-deploy-production
      ServiceRole: !GetAtt CodeBuildRole.Arn
      Artifacts:
        Type: CODEPIPELINE
      Environment:
        Type: LINUX_CONTAINER
        ComputeType: BUILD_GENERAL1_SMALL
        Image: aws/codebuild/standard:7.0
        EnvironmentVariables:
          - Name: STAGE
            Value: production
      Source:
        Type: CODEPIPELINE
        BuildSpec: buildspec.yml

  # ========================================
  # CodePipeline
  # ========================================
  Pipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      Name: order-processing-pipeline
      RoleArn: !GetAtt PipelineRole.Arn
      ArtifactStore:
        Type: S3
        Location: !Ref ArtifactBucket
      Stages:
        # Source Stage
        - Name: Source
          Actions:
            - Name: SourceAction
              ActionTypeId:
                Category: Source
                Owner: ThirdParty
                Provider: GitHub
                Version: '1'
              Configuration:
                Owner: !Select [0, !Split ['/', !Ref GitHubRepo]]
                Repo: !Select [1, !Split ['/', !Ref GitHubRepo]]
                Branch: !Ref GitHubBranch
                OAuthToken: !Ref GitHubToken
              OutputArtifacts:
                - Name: SourceOutput

        # Test Stage
        - Name: Test
          Actions:
            - Name: UnitTests
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: '1'
              Configuration:
                ProjectName: !Ref TestProject
              InputArtifacts:
                - Name: SourceOutput
              OutputArtifacts:
                - Name: TestOutput

        # Deploy Dev Stage
        - Name: DeployDev
          Actions:
            - Name: DeployToDev
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: '1'
              Configuration:
                ProjectName: !Ref DeployDevProject
              InputArtifacts:
                - Name: SourceOutput
              OutputArtifacts:
                - Name: DevOutput

        # Manual Approval - Staging
        - Name: ApproveStaging
          Actions:
            - Name: ManualApproval
              ActionTypeId:
                Category: Approval
                Owner: AWS
                Provider: Manual
                Version: '1'
              Configuration:
                CustomData: 'Please review dev deployment and approve for staging'
                NotificationArn: !Ref ApprovalTopic

        # Deploy Staging Stage
        - Name: DeployStaging
          Actions:
            - Name: DeployToStaging
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: '1'
              Configuration:
                ProjectName: !Ref DeployStagingProject
              InputArtifacts:
                - Name: SourceOutput
              OutputArtifacts:
                - Name: StagingOutput

        # Manual Approval - Production
        - Name: ApproveProduction
          Actions:
            - Name: ManualApproval
              ActionTypeId:
                Category: Approval
                Owner: AWS
                Provider: Manual
                Version: '1'
              Configuration:
                CustomData: 'Please review staging deployment and approve for production'
                NotificationArn: !Ref ApprovalTopic

        # Deploy Production Stage
        - Name: DeployProduction
          Actions:
            - Name: DeployToProduction
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: '1'
              Configuration:
                ProjectName: !Ref DeployProductionProject
              InputArtifacts:
                - Name: SourceOutput
              OutputArtifacts:
                - Name: ProductionOutput

  # ========================================
  # IAM Roles
  # ========================================
  PipelineRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: codepipeline.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AWSCodePipelineFullAccess
      Policies:
        - PolicyName: PipelinePolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - s3:GetObject
                  - s3:PutObject
                Resource: !Sub '${ArtifactBucket.Arn}/*'
              - Effect: Allow
                Action:
                  - codebuild:BatchGetBuilds
                  - codebuild:StartBuild
                Resource: '*'
              - Effect: Allow
                Action:
                  - sns:Publish
                Resource: !Ref ApprovalTopic

  CodeBuildRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: codebuild.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/PowerUserAccess
      Policies:
        - PolicyName: CodeBuildPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource: !Sub 'arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/codebuild/*'
              - Effect: Allow
                Action:
                  - s3:GetObject
                  - s3:PutObject
                Resource: !Sub '${ArtifactBucket.Arn}/*'
              - Effect: Allow
                Action:
                  - cloudformation:*
                  - lambda:*
                  - apigateway:*
                  - dynamodb:*
                  - sqs:*
                  - sns:*
                  - iam:PassRole
                Resource: '*'

  # ========================================
  # SNS Topic for Approvals
  # ========================================
  ApprovalTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: pipeline-approval-notifications
      Subscription:
        - Endpoint: ops-team@example.com
          Protocol: email

Outputs:
  PipelineName:
    Value: !Ref Pipeline
  
  PipelineUrl:
    Value: !Sub 'https://console.aws.amazon.com/codesuite/codepipeline/pipelines/${Pipeline}/view'
```

---

## 4️⃣ GitHub Actions Alternative

```yaml
# .github/workflows/deploy.yml
name: Deploy Order Processing Service

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '18'
  AWS_REGION: 'us-east-1'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Run tests
        run: npm test
      
      - name: Build
        run: npm run build

  deploy-dev:
    needs: test
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
      
      - name: Install Serverless Framework
        run: npm install -g serverless
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Deploy to Dev
        run: |
          npm ci
          npm run build
          serverless deploy --stage dev

  deploy-staging:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Deploy to Staging
        run: |
          npm ci
          npm run build
          serverless deploy --stage staging

  deploy-production:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_PROD }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_PROD }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Deploy to Production
        run: |
          npm ci
          npm run build
          serverless deploy --stage production
```

---

## 5️⃣ Rollback Strategy

```powershell
# scripts/rollback.ps1
param(
    [Parameter(Mandatory=$true)]
    [string]$Stage,
    
    [Parameter(Mandatory=$true)]
    [string]$Version
)

Write-Host "Rolling back $Stage to version $Version..."

# Get previous version from S3
aws s3 cp "s3://deployment-artifacts/order-processing-$Stage-$Version.zip" ./rollback.zip

# Extract and deploy
Expand-Archive -Path ./rollback.zip -DestinationPath ./rollback
cd rollback
serverless deploy --stage $Stage

Write-Host "Rollback complete"
```

---

## 🎯 Deployment Best Practices

### DO ✅
1. Run automated tests before deployment
2. Use separate AWS accounts for stages
3. Implement manual approval for production
4. Tag deployments with version numbers
5. Store artifacts for rollback
6. Monitor deployments with CloudWatch
7. Use canary deployments
8. Implement smoke tests post-deployment
9. Document deployment procedures
10. Practice disaster recovery

### DON'T ❌
1. Don't skip testing stages
2. Don't deploy directly to production
3. Don't ignore failed deployments
4. Don't skip backup before deployment
5. Don't deploy during peak hours

---

**Next:** [Multi-Stage Deployment](./11_MULTI_STAGE_DEPLOYMENT.md)

---

**Last Updated**: November 12, 2025
