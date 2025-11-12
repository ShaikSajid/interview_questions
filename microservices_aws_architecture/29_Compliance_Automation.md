# Compliance & Regulatory Automation

## Question 57: Automated Compliance Reporting

### 📋 Question Statement

Implement automated compliance reporting for Emirates NBD including PCI-DSS, GDPR, and local banking regulations using AWS Config, Security Hub, and custom Lambda functions.

---

### 📋 Compliance Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│               AUTOMATED COMPLIANCE & AUDIT SYSTEM                           │
└────────────────────────────────────────────────────────────────────────────┘

    ┌──────────────┐         ┌─────────────────┐
    │ AWS Resources│────────>│  AWS Config     │
    │  (Real-time) │         │  (Track Changes)│
    └──────────────┘         └────────┬────────┘
                                      │
                        ┌─────────────┴─────────────┐
                        │                           │
                        v                           v
                 ┌──────────┐              ┌───────────┐
                 │Config    │              │ Security  │
                 │  Rules   │              │   Hub     │
                 └────┬─────┘              └─────┬─────┘
                      │                          │
                      v                          v
                 ┌──────────┐              ┌───────────┐
                 │ Lambda   │              │ EventBridge│
                 │Compliance│              │   Rules   │
                 └────┬─────┘              └─────┬─────┘
                      │                          │
                      └──────────┬───────────────┘
                                 v
                          ┌──────────┐
                          │  S3      │
                          │ Reports  │
                          └──────────┘
```

### 📦 Compliance Stack CDK

```typescript
// infrastructure/cdk/compliance-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as config from 'aws-cdk-lib/aws-config';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as subscriptions from 'aws-cdk-lib/aws-sns-subscriptions';

export class ComplianceStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // S3 Bucket for compliance reports
    const complianceBucket = new s3.Bucket(this, 'ComplianceBucket', {
      bucketName: 'emirates-nbd-compliance-reports',
      encryption: s3.BucketEncryption.S3_MANAGED,
      versioned: true,
      lifecycleRules: [
        {
          transitions: [
            {
              storageClass: s3.StorageClass.GLACIER,
              transitionAfter: cdk.Duration.days(90)
            }
          ],
          expiration: cdk.Duration.days(2555) // 7 years for banking records
        }
      ],
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL
    });

    // SNS Topic for compliance alerts
    const complianceTopic = new sns.Topic(this, 'ComplianceTopic', {
      displayName: 'Compliance Violations'
    });

    complianceTopic.addSubscription(
      new subscriptions.EmailSubscription('compliance@emiratesnbd.com')
    );

    // Lambda: Compliance report generator
    const reportGenerator = new lambda.Function(this, 'ComplianceReportGenerator', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/compliance-reports'),
      timeout: cdk.Duration.minutes(15),
      memorySize: 2048,
      environment: {
        REPORTS_BUCKET: complianceBucket.bucketName,
        COMPLIANCE_TOPIC: complianceTopic.topicArn
      }
    });

    complianceBucket.grantWrite(reportGenerator);
    complianceTopic.grantPublish(reportGenerator);

    reportGenerator.addToRolePolicy(
      new iam.PolicyStatement({
        actions: [
          'config:DescribeComplianceByConfigRule',
          'config:DescribeComplianceByResource',
          'config:GetComplianceDetailsByConfigRule',
          'securityhub:GetFindings',
          'guardduty:ListFindings',
          'iam:GetAccountSummary',
          'cloudtrail:LookupEvents'
        ],
        resources: ['*']
      })
    );

    // Schedule: Daily compliance report at 2 AM UTC
    new events.Rule(this, 'DailyComplianceReport', {
      schedule: events.Schedule.cron({
        minute: '0',
        hour: '2'
      }),
      targets: [new targets.LambdaFunction(reportGenerator)]
    });

    // AWS Config Rules

    // Rule 1: Encrypted volumes
    new config.ManagedRule(this, 'EncryptedVolumes', {
      identifier: config.ManagedRuleIdentifiers.EC2_EBS_ENCRYPTION_BY_DEFAULT,
      description: 'Ensures EBS volumes are encrypted (PCI-DSS 3.4)',
      configRuleName: 'ebs-encryption-required'
    });

    // Rule 2: S3 encryption
    new config.ManagedRule(this, 'S3Encryption', {
      identifier: config.ManagedRuleIdentifiers.S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED,
      description: 'Ensures S3 buckets have encryption (PCI-DSS 3.4)',
      configRuleName: 's3-encryption-required'
    });

    // Rule 3: RDS encryption
    new config.ManagedRule(this, 'RDSEncryption', {
      identifier: config.ManagedRuleIdentifiers.RDS_STORAGE_ENCRYPTED,
      description: 'Ensures RDS instances are encrypted (PCI-DSS 3.4)',
      configRuleName: 'rds-encryption-required'
    });

    // Rule 4: Root account MFA
    new config.ManagedRule(this, 'RootAccountMFA', {
      identifier: config.ManagedRuleIdentifiers.ROOT_ACCOUNT_MFA_ENABLED,
      description: 'Ensures root account has MFA enabled (PCI-DSS 8.3)',
      configRuleName: 'root-mfa-enabled',
      maximumExecutionFrequency: config.MaximumExecutionFrequency.ONE_HOUR
    });

    // Rule 5: IAM password policy
    new config.ManagedRule(this, 'IAMPasswordPolicy', {
      identifier: config.ManagedRuleIdentifiers.IAM_PASSWORD_POLICY,
      description: 'Ensures strong password policy (PCI-DSS 8.2)',
      configRuleName: 'iam-password-policy',
      inputParameters: {
        RequireUppercaseCharacters: 'true',
        RequireLowercaseCharacters: 'true',
        RequireSymbols: 'true',
        RequireNumbers: 'true',
        MinimumPasswordLength: '14',
        MaxPasswordAge: '90'
      }
    });

    // Rule 6: CloudTrail enabled
    new config.ManagedRule(this, 'CloudTrailEnabled', {
      identifier: config.ManagedRuleIdentifiers.CLOUD_TRAIL_ENABLED,
      description: 'Ensures CloudTrail is enabled (PCI-DSS 10.2)',
      configRuleName: 'cloudtrail-enabled'
    });

    // Custom Rule: Data retention compliance
    const dataRetentionCheck = new lambda.Function(this, 'DataRetentionCheck', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/data-retention-check'),
      timeout: cdk.Duration.minutes(5)
    });

    dataRetentionCheck.addToRolePolicy(
      new iam.PolicyStatement({
        actions: ['s3:GetLifecycleConfiguration', 's3:ListAllMyBuckets'],
        resources: ['*']
      })
    );

    new config.CustomRule(this, 'DataRetentionRule', {
      configRuleName: 'data-retention-compliance',
      description: 'Ensures data retention policies are configured (UAE Central Bank)',
      lambdaFunction: dataRetentionCheck,
      periodic: true,
      maximumExecutionFrequency: config.MaximumExecutionFrequency.TWENTY_FOUR_HOURS
    });

    // Outputs
    new cdk.CfnOutput(this, 'ComplianceBucket', {
      value: complianceBucket.bucketName,
      description: 'Compliance reports bucket'
    });

    new cdk.CfnOutput(this, 'ComplianceTopicArn', {
      value: complianceTopic.topicArn,
      description: 'Compliance alerts topic'
    });
  }
}
```

### 📊 Compliance Report Generator

```typescript
// lambda/compliance-reports/index.ts
import { ConfigServiceClient, DescribeComplianceByConfigRuleCommand, GetComplianceDetailsByConfigRuleCommand } from '@aws-sdk/client-config-service';
import { SecurityHubClient, GetFindingsCommand } from '@aws-sdk/client-securityhub';
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { SNSClient, PublishCommand } from '@aws-sdk/client-sns';

const config = new ConfigServiceClient({});
const securityHub = new SecurityHubClient({});
const s3 = new S3Client({});
const sns = new SNSClient({});

const REPORTS_BUCKET = process.env.REPORTS_BUCKET!;
const COMPLIANCE_TOPIC = process.env.COMPLIANCE_TOPIC!;

export const handler = async () => {
  console.log('Generating compliance report...');

  try {
    // Collect compliance data
    const [configCompliance, securityFindings] = await Promise.all([
      getConfigCompliance(),
      getSecurityHubFindings()
    ]);

    // Generate report
    const report = generateComplianceReport(configCompliance, securityFindings);

    // Upload to S3
    await uploadReport(report);

    // Send alerts if violations found
    if (report.violations.length > 0) {
      await sendComplianceAlert(report);
    }

    console.log('Compliance report generated successfully');
    return { statusCode: 200, body: 'Report generated' };

  } catch (error) {
    console.error('Error generating report:', error);
    throw error;
  }
};

async function getConfigCompliance() {
  const response = await config.send(
    new DescribeComplianceByConfigRuleCommand({})
  );

  const rules = response.ComplianceByConfigRules || [];
  const compliance = [];

  for (const rule of rules) {
    if (rule.Compliance?.ComplianceType !== 'COMPLIANT') {
      const details = await config.send(
        new GetComplianceDetailsByConfigRuleCommand({
          ConfigRuleName: rule.ConfigRuleName,
          ComplianceTypes: ['NON_COMPLIANT']
        })
      );

      compliance.push({
        ruleName: rule.ConfigRuleName,
        complianceType: rule.Compliance?.ComplianceType,
        resources: details.EvaluationResults?.map(r => ({
          resourceType: r.EvaluationResultIdentifier?.EvaluationResultQualifier?.ResourceType,
          resourceId: r.EvaluationResultIdentifier?.EvaluationResultQualifier?.ResourceId,
          annotation: r.Annotation
        }))
      });
    }
  }

  return compliance;
}

async function getSecurityHubFindings() {
  const response = await securityHub.send(
    new GetFindingsCommand({
      Filters: {
        RecordState: [{ Value: 'ACTIVE', Comparison: 'EQUALS' }],
        SeverityLabel: [
          { Value: 'CRITICAL', Comparison: 'EQUALS' },
          { Value: 'HIGH', Comparison: 'EQUALS' }
        ]
      },
      MaxResults: 100
    })
  );

  return (response.Findings || []).map(f => ({
    title: f.Title,
    severity: f.Severity?.Label,
    resourceType: f.Resources?.[0]?.Type,
    resourceId: f.Resources?.[0]?.Id,
    description: f.Description,
    remediationText: f.Remediation?.Recommendation?.Text
  }));
}

function generateComplianceReport(configCompliance: any[], securityFindings: any[]) {
  const timestamp = new Date().toISOString();
  const date = timestamp.split('T')[0];

  return {
    reportId: `COMPLIANCE-${Date.now()}`,
    generatedAt: timestamp,
    reportingPeriod: date,
    summary: {
      totalConfigRules: configCompliance.length,
      totalSecurityFindings: securityFindings.length,
      complianceScore: calculateComplianceScore(configCompliance, securityFindings)
    },
    violations: [
      ...configCompliance,
      ...securityFindings.map(f => ({
        ruleName: f.title,
        complianceType: 'SECURITY_FINDING',
        severity: f.severity,
        resources: [{
          resourceType: f.resourceType,
          resourceId: f.resourceId,
          annotation: f.description
        }]
      }))
    ],
    pciDssCompliance: assessPCIDSS(configCompliance),
    gdprCompliance: assessGDPR(configCompliance),
    uaeCentralBankCompliance: assessUAECentralBank(configCompliance)
  };
}

function calculateComplianceScore(configCompliance: any[], securityFindings: any[]): string {
  const totalIssues = configCompliance.length + securityFindings.length;
  const maxScore = 100;
  const penaltyPerIssue = 2;
  const score = Math.max(0, maxScore - (totalIssues * penaltyPerIssue));
  return score.toFixed(1) + '%';
}

function assessPCIDSS(compliance: any[]): any {
  const pciRules = compliance.filter(c =>
    c.ruleName.includes('encryption') ||
    c.ruleName.includes('mfa') ||
    c.ruleName.includes('password') ||
    c.ruleName.includes('cloudtrail')
  );

  return {
    status: pciRules.length === 0 ? 'COMPLIANT' : 'NON_COMPLIANT',
    violations: pciRules.length,
    requirements: {
      'Requirement 3.4 (Encryption)': pciRules.filter(r => r.ruleName.includes('encryption')).length === 0,
      'Requirement 8.2 (Strong Auth)': pciRules.filter(r => r.ruleName.includes('password')).length === 0,
      'Requirement 8.3 (MFA)': pciRules.filter(r => r.ruleName.includes('mfa')).length === 0,
      'Requirement 10.2 (Audit Logs)': pciRules.filter(r => r.ruleName.includes('cloudtrail')).length === 0
    }
  };
}

function assessGDPR(compliance: any[]): any {
  return {
    status: 'COMPLIANT',
    dataProtection: 'ENABLED',
    dataRetention: 'CONFIGURED',
    rightToErasure: 'SUPPORTED'
  };
}

function assessUAECentralBank(compliance: any[]): any {
  return {
    status: 'COMPLIANT',
    dataResidency: 'UAE',
    backupRetention: '7_YEARS',
    disasterRecovery: 'CONFIGURED'
  };
}

async function uploadReport(report: any): Promise<void> {
  const date = new Date().toISOString().split('T')[0];
  const key = `compliance-reports/${date}/${report.reportId}.json`;

  await s3.send(
    new PutObjectCommand({
      Bucket: REPORTS_BUCKET,
      Key: key,
      Body: JSON.stringify(report, null, 2),
      ContentType: 'application/json',
      ServerSideEncryption: 'AES256'
    })
  );

  console.log(`Report uploaded to s3://${REPORTS_BUCKET}/${key}`);
}

async function sendComplianceAlert(report: any): Promise<void> {
  const message = `
⚠️ Compliance Violations Detected

Report ID: ${report.reportId}
Generated: ${report.generatedAt}
Compliance Score: ${report.summary.complianceScore}

Total Violations: ${report.violations.length}
- Config Rules: ${report.summary.totalConfigRules}
- Security Findings: ${report.summary.totalSecurityFindings}

PCI-DSS Status: ${report.pciDssCompliance.status}
GDPR Status: ${report.gdprCompliance.status}
UAE Central Bank Status: ${report.uaeCentralBankCompliance.status}

Please review the full report in S3: ${REPORTS_BUCKET}/compliance-reports/
`;

  await sns.send(
    new PublishCommand({
      TopicArn: COMPLIANCE_TOPIC,
      Subject: `⚠️ Compliance Report - ${report.reportId}`,
      Message: message
    })
  );
}
```

### 🎓 Interview Discussion Points - Q57

**Q1: What are key banking compliance requirements?**

**A**:
**PCI-DSS:**
- Encrypt cardholder data (Req 3.4)
- Strong authentication (Req 8.2)
- MFA for all access (Req 8.3)
- Log all access (Req 10.2)

**GDPR:**
- Data encryption
- Right to erasure
- Data portability
- Breach notification (72 hours)

**UAE Central Bank:**
- Data residency in UAE
- 7-year data retention
- Disaster recovery
- Cybersecurity framework

**Q2: How to automate compliance checks?**

**A**:
- **AWS Config**: Track resource configurations
- **Config Rules**: Managed + custom rules
- **Security Hub**: Aggregate findings
- **EventBridge**: Trigger remediation
- **Lambda**: Custom compliance logic
- **S3**: Store audit reports (7 years)

**Q3: What is AWS Config and how does it work?**

**A**:
- **Configuration tracking**: Records resource changes
- **Compliance rules**: Evaluates against policies
- **History**: Full audit trail of changes
- **Remediation**: Automatic or manual fixes
- **Multi-account**: Aggregate across org

**Q4: How to handle compliance violations?**

**A**:
- **Detection**: Config Rules evaluate continuously
- **Alert**: SNS notification to compliance team
- **Ticket**: Create Jira/ServiceNow ticket
- **Remediation**: Auto-fix via SSM Automation
- **Escalation**: If not fixed in SLA, escalate
- **Audit**: Log all actions in CloudTrail

**Q5: How to prove compliance to auditors?**

**A**:
- **Config timeline**: Show resource history
- **Compliance reports**: Daily/monthly summaries
- **CloudTrail logs**: Who did what, when
- **S3 retention**: 7-year audit trail
- **Dashboard**: Real-time compliance score
- **Evidence**: Automated report generation

---

## Question 58: Automated Policy Enforcement & Remediation

### 📋 Question Statement

Implement automated policy enforcement for Emirates NBD using AWS Config Rules, custom compliance checks, automatic remediation actions, and compliance dashboards for PCI-DSS, GDPR, and UAE Central Bank regulations.

---

### 🛡️ AWS Config Custom Rules

```typescript
// lambda/config-rules/encryption-check.ts
import { ConfigServiceClient, PutEvaluationsCommand } from '@aws-sdk/client-config-service';

export const handler = async (event: any) => {
  const configClient = new ConfigServiceClient({});
  const evaluations: any[] = [];

  // Get resource configuration
  const configurationItem = JSON.parse(event.configurationItem);
  const resourceType = configurationItem.resourceType;
  const resourceId = configurationItem.resourceId;

  let compliance = 'NON_COMPLIANT';
  let annotation = '';

  // Check encryption based on resource type
  switch (resourceType) {
    case 'AWS::S3::Bucket':
      const encryptionConfig = configurationItem.configuration.serverSideEncryptionConfiguration;
      if (encryptionConfig && encryptionConfig.rules) {
        compliance = 'COMPLIANT';
        annotation = 'S3 bucket has encryption enabled';
      } else {
        annotation = 'S3 bucket encryption is not enabled - PCI-DSS requirement 3.4';
      }
      break;

    case 'AWS::RDS::DBInstance':
      const storageEncrypted = configurationItem.configuration.storageEncrypted;
      if (storageEncrypted) {
        compliance = 'COMPLIANT';
        annotation = 'RDS instance has encryption enabled';
      } else {
        annotation = 'RDS instance encryption is not enabled - PCI-DSS requirement 3.4';
      }
      break;

    case 'AWS::DynamoDB::Table':
      const sseDescription = configurationItem.configuration.sSEDescription;
      if (sseDescription && sseDescription.status === 'ENABLED') {
        compliance = 'COMPLIANT';
        annotation = 'DynamoDB table has encryption enabled';
      } else {
        annotation = 'DynamoDB table encryption is not enabled - PCI-DSS requirement 3.4';
      }
      break;
  }

  evaluations.push({
    ComplianceResourceType: resourceType,
    ComplianceResourceId: resourceId,
    ComplianceType: compliance,
    Annotation: annotation,
    OrderingTimestamp: new Date()
  });

  // Submit evaluations
  await configClient.send(new PutEvaluationsCommand({
    Evaluations: evaluations,
    ResultToken: event.resultToken
  }));
};
```

### 🔧 Config Rules CDK Stack

```typescript
// infrastructure/cdk/config-rules-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as config from 'aws-cdk-lib/aws-config';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as sns from 'aws-cdk-lib/aws-sns';

export class ConfigRulesStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // SNS topic for compliance violations
    const complianceTopic = new sns.Topic(this, 'ComplianceAlerts', {
      displayName: 'Banking Compliance Violations'
    });

    // Custom Lambda for encryption check
    const encryptionCheckLambda = new lambda.Function(this, 'EncryptionCheck', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'encryption-check.handler',
      code: lambda.Code.fromAsset('lambda/config-rules'),
      timeout: cdk.Duration.seconds(30)
    });

    encryptionCheckLambda.addToRolePolicy(new iam.PolicyStatement({
      actions: ['config:PutEvaluations'],
      resources: ['*']
    }));

    // Config Rule: Encryption enabled
    new config.CfnConfigRule(this, 'EncryptionEnabledRule', {
      configRuleName: 'encryption-enabled-all-resources',
      description: 'Ensure all resources have encryption enabled (PCI-DSS 3.4)',
      source: {
        owner: 'CUSTOM_LAMBDA',
        sourceIdentifier: encryptionCheckLambda.functionArn,
        sourceDetails: [
          {
            eventSource: 'aws.config',
            messageType: 'ConfigurationItemChangeNotification'
          }
        ]
      },
      scope: {
        complianceResourceTypes: [
          'AWS::S3::Bucket',
          'AWS::RDS::DBInstance',
          'AWS::DynamoDB::Table',
          'AWS::EBS::Volume'
        ]
      }
    });

    // Managed Config Rules
    
    // PCI-DSS: MFA enabled for root account
    new config.ManagedRule(this, 'RootAccountMFAEnabled', {
      identifier: 'ROOT_ACCOUNT_MFA_ENABLED',
      description: 'PCI-DSS Requirement 8.3: MFA for root account'
    });

    // PCI-DSS: CloudTrail enabled
    new config.ManagedRule(this, 'CloudTrailEnabled', {
      identifier: 'CLOUD_TRAIL_ENABLED',
      description: 'PCI-DSS Requirement 10.1: Audit logging enabled'
    });

    // GDPR: S3 versioning enabled (data retention)
    new config.ManagedRule(this, 'S3VersioningEnabled', {
      identifier: 'S3_BUCKET_VERSIONING_ENABLED',
      description: 'GDPR Article 5(1)(f): Data integrity and availability',
      ruleScope: {
        complianceResourceTypes: ['AWS::S3::Bucket']
      }
    });

    // PCI-DSS: VPC flow logs enabled
    new config.ManagedRule(this, 'VPCFlowLogsEnabled', {
      identifier: 'VPC_FLOW_LOGS_ENABLED',
      description: 'PCI-DSS Requirement 10.2: Network traffic logging'
    });

    // Automatic Remediation
    const remediationLambda = new lambda.Function(this, 'AutoRemediation', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'remediation.handler',
      code: lambda.Code.fromAsset('lambda/auto-remediation'),
      environment: {
        COMPLIANCE_TOPIC_ARN: complianceTopic.topicArn
      }
    });

    remediationLambda.addToRolePolicy(new iam.PolicyStatement({
      actions: [
        's3:PutEncryptionConfiguration',
        'rds:ModifyDBInstance',
        'dynamodb:UpdateTable',
        'ec2:ModifyVolumeAttribute'
      ],
      resources: ['*']
    }));

    complianceTopic.grantPublish(remediationLambda);
  }
}
```

### 🤖 Auto-Remediation Service

```typescript
// lambda/auto-remediation/remediation.ts
import { S3, RDS, DynamoDB, SNS } from 'aws-sdk';

export const handler = async (event: any) => {
  const s3 = new S3();
  const rds = new RDS();
  const dynamodb = new DynamoDB();
  const sns = new SNS();

  for (const record of event.Records) {
    const message = JSON.parse(record.Sns.Message);
    const configRule = message.configRuleName;
    const resourceType = message.resourceType;
    const resourceId = message.resourceId;
    const complianceType = message.newEvaluationResult.complianceType;

    if (complianceType === 'NON_COMPLIANT') {
      console.log(`Auto-remediating ${resourceType}: ${resourceId}`);

      try {
        switch (resourceType) {
          case 'AWS::S3::Bucket':
            await remediateS3Encryption(s3, resourceId);
            break;

          case 'AWS::RDS::DBInstance':
            await remediateRDSEncryption(rds, resourceId);
            break;

          case 'AWS::DynamoDB::Table':
            await remediateDynamoDBEncryption(dynamodb, resourceId);
            break;
        }

        // Send success notification
        await sns.publish({
          TopicArn: process.env.COMPLIANCE_TOPIC_ARN!,
          Subject: `✅ Auto-Remediation Successful: ${resourceId}`,
          Message: `Successfully remediated ${resourceType} ${resourceId} for compliance rule ${configRule}`
        }).promise();

      } catch (error) {
        console.error('Remediation failed:', error);

        // Send failure notification
        await sns.publish({
          TopicArn: process.env.COMPLIANCE_TOPIC_ARN!,
          Subject: `❌ Auto-Remediation Failed: ${resourceId}`,
          Message: `Failed to remediate ${resourceType} ${resourceId}: ${error.message}`
        }).promise();
      }
    }
  }
};

async function remediateS3Encryption(s3: S3, bucketName: string) {
  await s3.putBucketEncryption({
    Bucket: bucketName,
    ServerSideEncryptionConfiguration: {
      Rules: [
        {
          ApplyServerSideEncryptionByDefault: {
            SSEAlgorithm: 'AES256'
          }
        }
      ]
    }
  }).promise();

  console.log(`Enabled encryption for S3 bucket: ${bucketName}`);
}

async function remediateRDSEncryption(rds: RDS, dbInstanceId: string) {
  // Note: Cannot enable encryption on existing RDS instance
  // Must create snapshot, copy with encryption, restore
  console.log(`RDS encryption requires manual intervention: ${dbInstanceId}`);
  throw new Error('RDS encryption remediation requires snapshot/restore process');
}

async function remediateDynamoDBEncryption(dynamodb: DynamoDB, tableName: string) {
  await dynamodb.updateTable({
    TableName: tableName,
    SSESpecification: {
      Enabled: true,
      SSEType: 'KMS'
    }
  }).promise();

  console.log(`Enabled encryption for DynamoDB table: ${tableName}`);
}
```

### 📊 Compliance Dashboard

```typescript
// services/compliance/dashboard-service.ts
import { ConfigService, SecurityHub } from 'aws-sdk';

export class ComplianceDashboardService {
  private config: ConfigService;
  private securityHub: SecurityHub;

  constructor() {
    this.config = new ConfigService();
    this.securityHub = new SecurityHub();
  }

  async getComplianceOverview(): Promise<ComplianceOverview> {
    // Get Config compliance summary
    const configCompliance = await this.config.describeComplianceByConfigRule({}).promise();

    let compliant = 0;
    let nonCompliant = 0;

    configCompliance.ComplianceByConfigRules?.forEach(rule => {
      if (rule.Compliance?.ComplianceType === 'COMPLIANT') {
        compliant++;
      } else if (rule.Compliance?.ComplianceType === 'NON_COMPLIANT') {
        nonCompliant++;
      }
    });

    const compliancePercentage = (compliant / (compliant + nonCompliant)) * 100;

    // Get Security Hub findings
    const findings = await this.securityHub.getFindings({
      Filters: {
        ComplianceStatus: [
          { Value: 'FAILED', Comparison: 'EQUALS' }
        ]
      },
      MaxResults: 100
    }).promise();

    return {
      totalRules: compliant + nonCompliant,
      compliantRules: compliant,
      nonCompliantRules: nonCompliant,
      compliancePercentage: Math.round(compliancePercentage),
      criticalFindings: findings.Findings?.filter(f => f.Severity?.Label === 'CRITICAL').length || 0,
      highFindings: findings.Findings?.filter(f => f.Severity?.Label === 'HIGH').length || 0
    };
  }

  async getComplianceByStandard(standard: 'PCI-DSS' | 'GDPR' | 'UAE-CB'): Promise<StandardCompliance> {
    const rules = await this.getRulesForStandard(standard);
    
    const compliance = await Promise.all(
      rules.map(async rule => {
        const status = await this.config.describeComplianceByConfigRule({
          ConfigRuleNames: [rule.ruleName]
        }).promise();

        return {
          requirement: rule.requirement,
          status: status.ComplianceByConfigRules?.[0]?.Compliance?.ComplianceType || 'UNKNOWN'
        };
      })
    );

    return {
      standard,
      requirements: compliance
    };
  }

  private getRulesForStandard(standard: string): Array<{ ruleName: string; requirement: string }> {
    const mapping: Record<string, Array<{ ruleName: string; requirement: string }>> = {
      'PCI-DSS': [
        { ruleName: 'encryption-enabled-all-resources', requirement: '3.4 - Encryption' },
        { ruleName: 'CLOUD_TRAIL_ENABLED', requirement: '10.1 - Audit Logging' },
        { ruleName: 'ROOT_ACCOUNT_MFA_ENABLED', requirement: '8.3 - MFA' }
      ],
      'GDPR': [
        { ruleName: 'S3_BUCKET_VERSIONING_ENABLED', requirement: 'Article 5(1)(f) - Data Integrity' },
        { ruleName: 'encryption-enabled-all-resources', requirement: 'Article 32 - Security' }
      ],
      'UAE-CB': [
        { ruleName: 'VPC_FLOW_LOGS_ENABLED', requirement: 'Network Monitoring' }
      ]
    };

    return mapping[standard] || [];
  }
}

interface ComplianceOverview {
  totalRules: number;
  compliantRules: number;
  nonCompliantRules: number;
  compliancePercentage: number;
  criticalFindings: number;
  highFindings: number;
}

interface StandardCompliance {
  standard: string;
  requirements: Array<{
    requirement: string;
    status: string;
  }>;
}
```

### 🎓 Interview Discussion Points - Q58

**Q1: What is AWS Config used for?**

**A**:
- **Configuration tracking**: Record resource changes
- **Compliance checking**: Evaluate against rules
- **Automated remediation**: Fix non-compliant resources
- **Audit history**: Who changed what and when

**Q2: What are common PCI-DSS requirements?**

**A**:
- **3.4**: Encryption at rest and in transit
- **8.3**: MFA for administrative access
- **10.1**: Audit all access to cardholder data
- **10.2**: Implement automated audit trails
- **11.4**: Use file-integrity monitoring

**Q3: How to implement automated remediation safely?**

**A**:
- **Whitelist**: Only remediate approved resource types
- **Approval workflow**: Require manual approval for critical resources
- **Testing**: Test remediation in sandbox first
- **Notifications**: Alert team before/after remediation
- **Rollback**: Ability to revert changes

**Q4: What is the difference between AWS Config and Security Hub?**

**A**:
- **Config**: Configuration compliance, resource tracking
- **Security Hub**: Security findings aggregation, multi-account
- **Use both**: Config for compliance rules, Security Hub for security posture

**Q5: How to prove compliance during audit?**

**A**:
- **Config timeline**: Show resource configuration history
- **Compliance reports**: Export Config rule evaluations
- **CloudTrail logs**: Prove who accessed what
- **Automated screenshots**: Dashboard snapshots daily
- **Evidence collection**: S3 bucket with compliance artifacts

---

**End of File 29 - Compliance Automation**