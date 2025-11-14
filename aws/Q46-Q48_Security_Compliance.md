# AWS Interview Questions: Security & Compliance (Q46-Q48)

## Question 46: Explain AWS Security Hub for centralized security management

### 📋 Answer

**AWS Security Hub** provides a comprehensive view of your security state within AWS and helps you check your environment against security industry standards and best practices.

### Security Hub Architecture:

```
AWS Security Hub
├── Security Standards
│   ├── AWS Foundational Security Best Practices
│   ├── CIS AWS Foundations Benchmark
│   ├── PCI DSS v3.2.1
│   ├── NIST 800-53
│   └── Custom security standards
├── Findings
│   ├── AWS services (GuardDuty, Inspector, Macie)
│   ├── Third-party integrations
│   ├── Custom findings
│   └── Finding aggregation
├── Insights
│   ├── Pre-built insights
│   ├── Custom insights
│   └── Security score
├── Automated Remediation
│   ├── EventBridge integration
│   ├── Lambda functions
│   ├── Systems Manager automation
│   └── Step Functions workflows
└── Multi-Account
    ├── Central administrator account
    ├── Member accounts
    ├── Cross-region aggregation
    └── Organizational integration
```

### Complete Security Hub Implementation:

```javascript
// security-hub-service.js
import {
  SecurityHubClient,
  EnableSecurityHubCommand,
  BatchImportFindingsCommand,
  GetFindingsCommand,
  UpdateFindingsCommand,
  CreateInsightCommand,
  GetInsightResultsCommand,
  BatchEnableStandardsCommand,
  DescribeStandardsControlsCommand
} from '@aws-sdk/client-securityhub';

import {
  EventBridgeClient,
  PutRuleCommand,
  PutTargetsCommand
} from '@aws-sdk/client-eventbridge';

const securityHubClient = new SecurityHubClient({ region: 'us-east-1' });
const eventBridgeClient = new EventBridgeClient({ region: 'us-east-1' });

class SecurityHubService {
  // 1. Enable Security Hub
  async enableSecurityHub(enableDefaultStandards = true) {
    const command = new EnableSecurityHubCommand({
      EnableDefaultStandards: enableDefaultStandards,
      Tags: {
        Environment: 'Production',
        ManagedBy: 'SecurityTeam'
      }
    });
    
    try {
      await securityHubClient.send(command);
      console.log('Security Hub enabled');
    } catch (error) {
      console.error('Failed to enable Security Hub:', error);
      throw error;
    }
  }
  
  // 2. Enable Security Standards
  async enableStandards(standardsArns) {
    const command = new BatchEnableStandardsCommand({
      StandardsSubscriptionRequests: standardsArns.map(arn => ({
        StandardsArn: arn
      }))
    });
    
    const response = await securityHubClient.send(command);
    console.log('Standards enabled:', response.StandardsSubscriptions.length);
    return response.StandardsSubscriptions;
  }
  
  // 3. Get Findings
  async getFindings(filters = {}, maxResults = 100) {
    const command = new GetFindingsCommand({
      Filters: {
        ...filters,
        RecordState: filters.recordState || [{ Value: 'ACTIVE', Comparison: 'EQUALS' }]
      },
      MaxResults: maxResults,
      SortCriteria: [
        {
          Field: 'SeverityLabel',
          SortOrder: 'desc'
        },
        {
          Field: 'UpdatedAt',
          SortOrder: 'desc'
        }
      ]
    });
    
    const response = await securityHubClient.send(command);
    
    return response.Findings.map(finding => ({
      id: finding.Id,
      title: finding.Title,
      description: finding.Description,
      severity: finding.Severity.Label,
      productArn: finding.ProductArn,
      resourceType: finding.Resources[0]?.Type,
      resourceId: finding.Resources[0]?.Id,
      complianceStatus: finding.Compliance?.Status,
      workflowStatus: finding.Workflow?.Status,
      remediation: finding.Remediation?.Recommendation?.Text,
      createdAt: finding.CreatedAt,
      updatedAt: finding.UpdatedAt
    }));
  }
  
  // 4. Update Finding Status
  async updateFinding(findingId, productArn, workflowStatus, note) {
    const command = new UpdateFindingsCommand({
      Filters: {
        Id: [{ Value: findingId, Comparison: 'EQUALS' }],
        ProductArn: [{ Value: productArn, Comparison: 'EQUALS' }]
      },
      Note: {
        Text: note,
        UpdatedBy: 'SecurityAutomation'
      },
      Workflow: {
        Status: workflowStatus  // NEW, NOTIFIED, RESOLVED, SUPPRESSED
      }
    });
    
    await securityHubClient.send(command);
    console.log('Finding updated:', findingId);
  }
  
  // 5. Import Custom Findings
  async importCustomFinding(finding) {
    const command = new BatchImportFindingsCommand({
      Findings: [
        {
          SchemaVersion: '2018-10-08',
          Id: finding.id,
          ProductArn: finding.productArn,
          GeneratorId: finding.generatorId,
          AwsAccountId: finding.accountId,
          Types: finding.types,
          CreatedAt: new Date().toISOString(),
          UpdatedAt: new Date().toISOString(),
          Severity: {
            Label: finding.severity,  // INFORMATIONAL, LOW, MEDIUM, HIGH, CRITICAL
            Normalized: this.getSeverityScore(finding.severity)
          },
          Title: finding.title,
          Description: finding.description,
          Resources: finding.resources.map(resource => ({
            Type: resource.type,
            Id: resource.id,
            Partition: 'aws',
            Region: resource.region
          })),
          Compliance: {
            Status: finding.complianceStatus || 'FAILED'
          },
          Remediation: {
            Recommendation: {
              Text: finding.remediation,
              Url: finding.remediationUrl
            }
          },
          Workflow: {
            Status: 'NEW'
          },
          RecordState: 'ACTIVE'
        }
      ]
    });
    
    const response = await securityHubClient.send(command);
    
    if (response.FailedCount > 0) {
      console.error('Failed to import findings:', response.FailedFindings);
    }
    
    console.log('Imported findings:', response.SuccessCount);
    return response;
  }
  
  // 6. Create Custom Insight
  async createInsight(name, filters) {
    const command = new CreateInsightCommand({
      Name: name,
      Filters: filters,
      GroupByAttribute: 'ResourceType'
    });
    
    const response = await securityHubClient.send(command);
    console.log('Insight created:', response.InsightArn);
    return response.InsightArn;
  }
  
  // 7. Get Insight Results
  async getInsightResults(insightArn) {
    const command = new GetInsightResultsCommand({
      InsightArn: insightArn
    });
    
    const response = await securityHubClient.send(command);
    
    return response.InsightResults.ResultValues.map(result => ({
      groupByAttribute: result.GroupByAttributeValue,
      count: result.Count
    }));
  }
  
  // Helper: Convert severity to normalized score
  getSeverityScore(severity) {
    const scores = {
      'INFORMATIONAL': 0,
      'LOW': 1,
      'MEDIUM': 40,
      'HIGH': 70,
      'CRITICAL': 90
    };
    return scores[severity] || 0;
  }
}

// Automated Remediation System
class SecurityHubRemediationOrchestrator {
  constructor(securityHubService) {
    this.securityHub = securityHubService;
  }
  
  // Setup automated remediation
  async setupAutomatedRemediation(remediationLambdaArn) {
    // Create EventBridge rule for Security Hub findings
    const ruleName = 'security-hub-auto-remediation';
    
    const putRuleCommand = new PutRuleCommand({
      Name: ruleName,
      Description: 'Automatically remediate Security Hub findings',
      EventPattern: JSON.stringify({
        source: ['aws.securityhub'],
        'detail-type': ['Security Hub Findings - Imported'],
        detail: {
          findings: {
            Severity: {
              Label: ['HIGH', 'CRITICAL']
            },
            Workflow: {
              Status: ['NEW']
            },
            Compliance: {
              Status: ['FAILED']
            }
          }
        }
      }),
      State: 'ENABLED'
    });
    
    await eventBridgeClient.send(putRuleCommand);
    console.log('EventBridge rule created');
    
    // Add Lambda as target
    const putTargetsCommand = new PutTargetsCommand({
      Rule: ruleName,
      Targets: [
        {
          Id: '1',
          Arn: remediationLambdaArn,
          RetryPolicy: {
            MaximumRetryAttempts: 2,
            MaximumEventAge: 3600
          }
        }
      ]
    });
    
    await eventBridgeClient.send(putTargetsCommand);
    console.log('Lambda target added to rule');
  }
  
  // Generate remediation report
  async generateRemediationReport() {
    const findings = await this.securityHub.getFindings({
      SeverityLabel: [
        { Value: 'HIGH', Comparison: 'EQUALS' },
        { Value: 'CRITICAL', Comparison: 'EQUALS' }
      ]
    });
    
    const report = {
      totalFindings: findings.length,
      bySeverity: this.groupBy(findings, 'severity'),
      byResourceType: this.groupBy(findings, 'resourceType'),
      byComplianceStatus: this.groupBy(findings, 'complianceStatus'),
      topFindings: findings.slice(0, 10)
    };
    
    return report;
  }
  
  groupBy(array, key) {
    return array.reduce((result, item) => {
      const group = item[key] || 'Unknown';
      result[group] = (result[group] || 0) + 1;
      return result;
    }, {});
  }
}

// Security Standards Configuration
class SecurityStandardsConfig {
  static getStandardArns(accountId, region) {
    return {
      awsFoundational: `arn:aws:securityhub:${region}::standards/aws-foundational-security-best-practices/v/1.0.0`,
      cisBenchmark: `arn:aws:securityhub:::ruleset/cis-aws-foundations-benchmark/v/1.2.0`,
      pciDss: `arn:aws:securityhub:${region}::standards/pci-dss/v/3.2.1`,
      nist: `arn:aws:securityhub:${region}::standards/nist-800-53/v/5.0.0`
    };
  }
  
  // Common finding filters
  static getCommonFilters() {
    return {
      criticalFindings: {
        SeverityLabel: [{ Value: 'CRITICAL', Comparison: 'EQUALS' }],
        RecordState: [{ Value: 'ACTIVE', Comparison: 'EQUALS' }]
      },
      
      ec2Findings: {
        ResourceType: [{ Value: 'AwsEc2Instance', Comparison: 'EQUALS' }],
        RecordState: [{ Value: 'ACTIVE', Comparison: 'EQUALS' }]
      },
      
      s3Findings: {
        ResourceType: [{ Value: 'AwsS3Bucket', Comparison: 'EQUALS' }],
        RecordState: [{ Value: 'ACTIVE', Comparison: 'EQUALS' }]
      },
      
      iamFindings: {
        ResourceType: [{ Value: 'AwsIamUser', Comparison: 'PREFIX' }],
        RecordState: [{ Value: 'ACTIVE', Comparison: 'EQUALS' }]
      },
      
      nonCompliant: {
        ComplianceStatus: [{ Value: 'FAILED', Comparison: 'EQUALS' }],
        RecordState: [{ Value: 'ACTIVE', Comparison: 'EQUALS' }]
      },
      
      recentFindings: {
        UpdatedAt: [{
          Start: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString(),
          End: new Date().toISOString()
        }],
        RecordState: [{ Value: 'ACTIVE', Comparison: 'EQUALS' }]
      }
    };
  }
}

// Usage - Complete Security Hub Setup
const securityHub = new SecurityHubService();
const remediation = new SecurityHubRemediationOrchestrator(securityHub);

// Enable Security Hub
await securityHub.enableSecurityHub(true);

// Enable all security standards
const standards = SecurityStandardsConfig.getStandardArns('123456789012', 'us-east-1');
await securityHub.enableStandards([
  standards.awsFoundational,
  standards.cisBenchmark,
  standards.pciDss
]);

// Get all critical findings
const filters = SecurityStandardsConfig.getCommonFilters();
const criticalFindings = await securityHub.getFindings(filters.criticalFindings);

console.log(`Found ${criticalFindings.length} critical findings`);
criticalFindings.forEach(finding => {
  console.log(`- ${finding.title} (${finding.severity})`);
  console.log(`  Resource: ${finding.resourceType} - ${finding.resourceId}`);
  console.log(`  Remediation: ${finding.remediation}\n`);
});

// Import custom finding
await securityHub.importCustomFinding({
  id: 'custom-finding-001',
  productArn: 'arn:aws:securityhub:us-east-1:123456789012:product/123456789012/default',
  generatorId: 'custom-security-scanner',
  accountId: '123456789012',
  types: ['Software and Configuration Checks/Vulnerabilities/CVE'],
  severity: 'HIGH',
  title: 'Unencrypted database detected',
  description: 'RDS instance prod-db-1 does not have encryption enabled',
  resources: [
    {
      type: 'AwsRdsDbInstance',
      id: 'arn:aws:rds:us-east-1:123456789012:db:prod-db-1',
      region: 'us-east-1'
    }
  ],
  complianceStatus: 'FAILED',
  remediation: 'Enable encryption for the RDS instance',
  remediationUrl: 'https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.Encryption.html'
});

// Create custom insight for EC2 security
const insightArn = await securityHub.createInsight(
  'EC2 Security Findings',
  filters.ec2Findings
);

const insightResults = await securityHub.getInsightResults(insightArn);
console.log('EC2 Security Insight Results:', insightResults);

// Setup automated remediation
await remediation.setupAutomatedRemediation(
  'arn:aws:lambda:us-east-1:123456789012:function:security-remediation'
);

// Generate remediation report
const report = await remediation.generateRemediationReport();
console.log('Security Remediation Report:', JSON.stringify(report, null, 2));

// Update finding status after remediation
await securityHub.updateFinding(
  criticalFindings[0].id,
  criticalFindings[0].productArn,
  'RESOLVED',
  'Issue remediated by security team - encryption enabled'
);
```

### Remediation Lambda Function:

```javascript
// lambda-remediation-handler.js
import { EC2Client, ModifyInstanceAttributeCommand } from '@aws-sdk/client-ec2';
import { S3Client, PutBucketEncryptionCommand } from '@aws-sdk/client-s3';
import { RDSClient, ModifyDBInstanceCommand } from '@aws-sdk/client-rds';

const ec2Client = new EC2Client();
const s3Client = new S3Client();
const rdsClient = new RDSClient();

export const handler = async (event) => {
  console.log('Security Hub Finding:', JSON.stringify(event, null, 2));
  
  const finding = event.detail.findings[0];
  const resourceType = finding.Resources[0].Type;
  const resourceId = finding.Resources[0].Id;
  
  try {
    switch (resourceType) {
      case 'AwsEc2Instance':
        await remediateEC2(finding, resourceId);
        break;
      
      case 'AwsS3Bucket':
        await remediateS3(finding, resourceId);
        break;
      
      case 'AwsRdsDbInstance':
        await remediateRDS(finding, resourceId);
        break;
      
      default:
        console.log('No automated remediation for:', resourceType);
    }
    
    return { statusCode: 200, body: 'Remediation completed' };
  } catch (error) {
    console.error('Remediation failed:', error);
    return { statusCode: 500, body: 'Remediation failed' };
  }
};

async function remediateEC2(finding, instanceId) {
  // Example: Enable detailed monitoring
  if (finding.Title.includes('monitoring')) {
    const command = new ModifyInstanceAttributeCommand({
      InstanceId: instanceId.split('/').pop(),
      Monitoring: { Value: true }
    });
    
    await ec2Client.send(command);
    console.log('Enabled detailed monitoring for:', instanceId);
  }
}

async function remediateS3(finding, bucketArn) {
  const bucketName = bucketArn.split(':::')[1];
  
  // Enable default encryption
  if (finding.Title.includes('encryption')) {
    const command = new PutBucketEncryptionCommand({
      Bucket: bucketName,
      ServerSideEncryptionConfiguration: {
        Rules: [
          {
            ApplyServerSideEncryptionByDefault: {
              SSEAlgorithm: 'AES256'
            },
            BucketKeyEnabled: true
          }
        ]
      }
    });
    
    await s3Client.send(command);
    console.log('Enabled encryption for bucket:', bucketName);
  }
}

async function remediateRDS(finding, dbInstanceArn) {
  const dbInstanceId = dbInstanceArn.split(':db:')[1];
  
  // Enable automated backups
  if (finding.Title.includes('backup')) {
    const command = new ModifyDBInstanceCommand({
      DBInstanceIdentifier: dbInstanceId,
      BackupRetentionPeriod: 7,
      PreferredBackupWindow: '03:00-04:00'
    });
    
    await rdsClient.send(command);
    console.log('Enabled automated backups for:', dbInstanceId);
  }
}
```

### Best Practices:

1. ✅ Enable Security Hub in all regions
2. ✅ Enable multiple security standards
3. ✅ Integrate with GuardDuty, Inspector, Macie
4. ✅ Setup automated remediation with EventBridge
5. ✅ Use custom insights for specific use cases
6. ✅ Implement multi-account aggregation
7. ✅ Review findings regularly
8. ✅ Track remediation with workflow status
9. ✅ Export findings to SIEM systems
10. ✅ Monitor security score trends

---

## Question 47: Explain AWS Config for resource compliance monitoring

### 📋 Answer

**AWS Config** provides a detailed view of the configuration of AWS resources in your AWS account, including how they are related and how they were configured in the past.

### AWS Config Architecture:

```
AWS Config
├── Configuration Recorder
│   ├── Resource tracking
│   ├── Configuration history
│   ├── Configuration snapshots
│   └── Change notifications
├── Config Rules
│   ├── AWS managed rules (200+)
│   ├── Custom rules (Lambda)
│   ├── Conformance packs
│   └── Organization rules
├── Compliance
│   ├── Compliant/Non-compliant
│   ├── Compliance timeline
│   ├── Resource relationships
│   └── Change history
├── Remediation
│   ├── Automatic remediation
│   ├── Manual remediation
│   ├── SSM Automation documents
│   └── Retry logic
└── Aggregation
    ├── Multi-account
    ├── Multi-region
    ├── Organization-wide
    └── Custom aggregators
```

### Complete AWS Config Implementation:

```javascript
// config-service.js
import {
  ConfigServiceClient,
  PutConfigurationRecorderCommand,
  PutDeliveryChannelCommand,
  StartConfigurationRecorderCommand,
  PutConfigRuleCommand,
  DescribeComplianceByConfigRuleCommand,
  DescribeComplianceByResourceCommand,
  GetComplianceDetailsByConfigRuleCommand,
  PutRemediationConfigurationsCommand,
  StartRemediationExecutionCommand,
  PutConformancePackCommand
} from '@aws-sdk/client-config-service';

const configClient = new ConfigServiceClient({ region: 'us-east-1' });

class ConfigService {
  // 1. Setup Configuration Recorder
  async setupConfigurationRecorder(roleArn, bucketName) {
    // Create configuration recorder
    const recorderCommand = new PutConfigurationRecorderCommand({
      ConfigurationRecorder: {
        name: 'default',
        roleARN: roleArn,
        recordingGroup: {
          allSupported: true,
          includeGlobalResourceTypes: true,
          resourceTypes: []
        }
      }
    });
    
    await configClient.send(recorderCommand);
    console.log('Configuration recorder created');
    
    // Create delivery channel
    const channelCommand = new PutDeliveryChannelCommand({
      DeliveryChannel: {
        name: 'default',
        s3BucketName: bucketName,
        snsTopicARN: null,
        configSnapshotDeliveryProperties: {
          deliveryFrequency: 'TwentyFour_Hours'
        }
      }
    });
    
    await configClient.send(channelCommand);
    console.log('Delivery channel created');
    
    // Start recorder
    const startCommand = new StartConfigurationRecorderCommand({
      ConfigurationRecorderName: 'default'
    });
    
    await configClient.send(startCommand);
    console.log('Configuration recorder started');
  }
  
  // 2. Create Config Rule (Managed)
  async createManagedRule(ruleName, sourceIdentifier, parameters = {}) {
    const command = new PutConfigRuleCommand({
      ConfigRule: {
        ConfigRuleName: ruleName,
        Description: `Managed rule: ${sourceIdentifier}`,
        Source: {
          Owner: 'AWS',
          SourceIdentifier: sourceIdentifier
        },
        InputParameters: JSON.stringify(parameters),
        Scope: {
          ComplianceResourceTypes: []
        },
        ConfigRuleState: 'ACTIVE'
      }
    });
    
    try {
      await configClient.send(command);
      console.log('Config rule created:', ruleName);
    } catch (error) {
      console.error('Failed to create rule:', error);
      throw error;
    }
  }
  
  // 3. Create Custom Config Rule (Lambda)
  async createCustomRule(ruleName, lambdaArn, resourceTypes, parameters = {}) {
    const command = new PutConfigRuleCommand({
      ConfigRule: {
        ConfigRuleName: ruleName,
        Description: 'Custom compliance rule',
        Source: {
          Owner: 'CUSTOM_LAMBDA',
          SourceIdentifier: lambdaArn,
          SourceDetails: [
            {
              EventSource: 'aws.config',
              MessageType: 'ConfigurationItemChangeNotification'
            },
            {
              EventSource: 'aws.config',
              MessageType: 'OversizedConfigurationItemChangeNotification'
            }
          ]
        },
        Scope: {
          ComplianceResourceTypes: resourceTypes
        },
        InputParameters: JSON.stringify(parameters),
        ConfigRuleState: 'ACTIVE'
      }
    });
    
    await configClient.send(command);
    console.log('Custom config rule created:', ruleName);
  }
  
  // 4. Get Compliance Status
  async getComplianceByRule(ruleNames = []) {
    const command = new DescribeComplianceByConfigRuleCommand({
      ConfigRuleNames: ruleNames,
      ComplianceTypes: []  // Empty = all types
    });
    
    const response = await configClient.send(command);
    
    return response.ComplianceByConfigRules.map(rule => ({
      ruleName: rule.ConfigRuleName,
      complianceType: rule.Compliance.ComplianceType,
      contributorCount: rule.Compliance.ComplianceContributorCount?.CappedCount || 0
    }));
  }
  
  // 5. Get Non-Compliant Resources
  async getNonCompliantResources(ruleName) {
    const command = new GetComplianceDetailsByConfigRuleCommand({
      ConfigRuleName: ruleName,
      ComplianceTypes: ['NON_COMPLIANT'],
      Limit: 100
    });
    
    const response = await configClient.send(command);
    
    return response.EvaluationResults.map(result => ({
      resourceType: result.EvaluationResultIdentifier.EvaluationResultQualifier.ResourceType,
      resourceId: result.EvaluationResultIdentifier.EvaluationResultQualifier.ResourceId,
      complianceType: result.ComplianceType,
      annotation: result.Annotation,
      configRuleInvokedTime: result.ConfigRuleInvokedTime,
      resultRecordedTime: result.ResultRecordedTime
    }));
  }
  
  // 6. Setup Automatic Remediation
  async setupRemediation(ruleName, automationDocumentName, parameters, retryAttempts = 5) {
    const command = new PutRemediationConfigurationsCommand({
      RemediationConfigurations: [
        {
          ConfigRuleName: ruleName,
          TargetType: 'SSM_DOCUMENT',
          TargetIdentifier: automationDocumentName,
          TargetVersion: '1',
          Parameters: parameters,
          ResourceType: null,  // Apply to all resource types
          Automatic: true,
          MaximumAutomaticAttempts: retryAttempts,
          RetryAttemptSeconds: 60
        }
      ]
    });
    
    await configClient.send(command);
    console.log('Remediation configured for rule:', ruleName);
  }
  
  // 7. Start Manual Remediation
  async startRemediation(ruleName, resourceKeys) {
    const command = new StartRemediationExecutionCommand({
      ConfigRuleName: ruleName,
      ResourceKeys: resourceKeys.map(key => ({
        ResourceType: key.resourceType,
        ResourceId: key.resourceId
      }))
    });
    
    const response = await configClient.send(command);
    console.log('Remediation started for', response.FailedItems.length, 'resources');
    return response;
  }
  
  // 8. Deploy Conformance Pack
  async deployConformancePack(packName, templateBody, parameters = {}) {
    const command = new PutConformancePackCommand({
      ConformancePackName: packName,
      TemplateBody: templateBody,
      ConformancePackInputParameters: Object.entries(parameters).map(([key, value]) => ({
        ParameterName: key,
        ParameterValue: value
      }))
    });
    
    await configClient.send(command);
    console.log('Conformance pack deployed:', packName);
  }
}

// Config Rules Library
class ConfigRulesLibrary {
  // Common AWS managed rules
  static getManagedRules() {
    return {
      // EC2 Rules
      ec2EncryptedVolumes: {
        name: 'encrypted-volumes',
        sourceIdentifier: 'ENCRYPTED_VOLUMES',
        parameters: {}
      },
      
      ec2InstanceDetailedMonitoring: {
        name: 'ec2-instance-detailed-monitoring-enabled',
        sourceIdentifier: 'EC2_INSTANCE_DETAILED_MONITORING_ENABLED',
        parameters: {}
      },
      
      ec2ManagedInstanceAssociation: {
        name: 'ec2-managedinstance-association-compliance-status-check',
        sourceIdentifier: 'EC2_MANAGEDINSTANCE_ASSOCIATION_COMPLIANCE_STATUS_CHECK',
        parameters: {}
      },
      
      // S3 Rules
      s3BucketPublicReadProhibited: {
        name: 's3-bucket-public-read-prohibited',
        sourceIdentifier: 'S3_BUCKET_PUBLIC_READ_PROHIBITED',
        parameters: {}
      },
      
      s3BucketEncryption: {
        name: 's3-bucket-server-side-encryption-enabled',
        sourceIdentifier: 'S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED',
        parameters: {}
      },
      
      s3BucketVersioning: {
        name: 's3-bucket-versioning-enabled',
        sourceIdentifier: 'S3_BUCKET_VERSIONING_ENABLED',
        parameters: {}
      },
      
      // RDS Rules
      rdsEncryptedStorage: {
        name: 'rds-storage-encrypted',
        sourceIdentifier: 'RDS_STORAGE_ENCRYPTED',
        parameters: {}
      },
      
      rdsBackupEnabled: {
        name: 'db-backup-enabled',
        sourceIdentifier: 'DB_BACKUP_ENABLED',
        parameters: { backupRetentionPeriod: '7' }
      },
      
      rdsMultiAz: {
        name: 'rds-multi-az-support',
        sourceIdentifier: 'RDS_MULTI_AZ_SUPPORT',
        parameters: {}
      },
      
      // IAM Rules
      iamPasswordPolicy: {
        name: 'iam-password-policy',
        sourceIdentifier: 'IAM_PASSWORD_POLICY',
        parameters: {
          RequireUppercaseCharacters: 'true',
          RequireLowercaseCharacters: 'true',
          RequireSymbols: 'true',
          RequireNumbers: 'true',
          MinimumPasswordLength: '14',
          PasswordReusePrevention: '24',
          MaxPasswordAge: '90'
        }
      },
      
      iamRootAccessKey: {
        name: 'iam-root-access-key-check',
        sourceIdentifier: 'IAM_ROOT_ACCESS_KEY_CHECK',
        parameters: {}
      },
      
      iamUserMfa: {
        name: 'iam-user-mfa-enabled',
        sourceIdentifier: 'IAM_USER_MFA_ENABLED',
        parameters: {}
      },
      
      // VPC Rules
      vpcFlowLogsEnabled: {
        name: 'vpc-flow-logs-enabled',
        sourceIdentifier: 'VPC_FLOW_LOGS_ENABLED',
        parameters: {}
      },
      
      vpcDefaultSecurityGroupClosed: {
        name: 'vpc-default-security-group-closed',
        sourceIdentifier: 'VPC_DEFAULT_SECURITY_GROUP_CLOSED',
        parameters: {}
      },
      
      // CloudTrail Rules
      cloudTrailEnabled: {
        name: 'cloud-trail-enabled',
        sourceIdentifier: 'CLOUD_TRAIL_ENABLED',
        parameters: {}
      },
      
      cloudTrailLogFileValidation: {
        name: 'cloud-trail-log-file-validation-enabled',
        sourceIdentifier: 'CLOUD_TRAIL_LOG_FILE_VALIDATION_ENABLED',
        parameters: {}
      }
    };
  }
  
  // Remediation actions
  static getRemediationActions() {
    return {
      enableS3Encryption: {
        documentName: 'AWS-ConfigureS3BucketEncryption',
        parameters: {
          BucketName: { ResourceValue: { Value: 'RESOURCE_ID' } },
          SSEAlgorithm: { StaticValue: { Values: ['AES256'] } }
        }
      },
      
      enableS3Versioning: {
        documentName: 'AWS-ConfigureS3BucketVersioning',
        parameters: {
          BucketName: { ResourceValue: { Value: 'RESOURCE_ID' } },
          VersioningState: { StaticValue: { Values: ['Enabled'] } }
        }
      },
      
      enableVPCFlowLogs: {
        documentName: 'AWS-EnableVPCFlowLogs',
        parameters: {
          VpcId: { ResourceValue: { Value: 'RESOURCE_ID' } },
          TrafficType: { StaticValue: { Values: ['ALL'] } },
          LogDestinationType: { StaticValue: { Values: ['cloud-watch-logs'] } },
          LogGroupName: { StaticValue: { Values: ['/aws/vpc/flowlogs'] } }
        }
      },
      
      enableRDSBackup: {
        documentName: 'AWS-EnableRDSBackup',
        parameters: {
          DBInstanceIdentifier: { ResourceValue: { Value: 'RESOURCE_ID' } },
          BackupRetentionPeriod: { StaticValue: { Values: ['7'] } }
        }
      }
    };
  }
}

// Usage - Complete Config Setup
const config = new ConfigService();

// Setup Config
await config.setupConfigurationRecorder(
  'arn:aws:iam::123456789012:role/AWSConfigRole',
  'my-config-bucket'
);

// Deploy all common rules
const rules = ConfigRulesLibrary.getManagedRules();

// S3 Security Rules
await config.createManagedRule(
  rules.s3BucketPublicReadProhibited.name,
  rules.s3BucketPublicReadProhibited.sourceIdentifier
);

await config.createManagedRule(
  rules.s3BucketEncryption.name,
  rules.s3BucketEncryption.sourceIdentifier
);

// Setup automatic remediation for S3 encryption
const remediations = ConfigRulesLibrary.getRemediationActions();
await config.setupRemediation(
  rules.s3BucketEncryption.name,
  remediations.enableS3Encryption.documentName,
  remediations.enableS3Encryption.parameters,
  5  // retry 5 times
);

// IAM Security Rules
await config.createManagedRule(
  rules.iamPasswordPolicy.name,
  rules.iamPasswordPolicy.sourceIdentifier,
  rules.iamPasswordPolicy.parameters
);

await config.createManagedRule(
  rules.iamRootAccessKey.name,
  rules.iamRootAccessKey.sourceIdentifier
);

// RDS Security Rules
await config.createManagedRule(
  rules.rdsEncryptedStorage.name,
  rules.rdsEncryptedStorage.sourceIdentifier
);

await config.createManagedRule(
  rules.rdsBackupEnabled.name,
  rules.rdsBackupEnabled.sourceIdentifier,
  rules.rdsBackupEnabled.parameters
);

// Wait for evaluation
await new Promise(resolve => setTimeout(resolve, 60000));

// Check compliance status
const complianceStatus = await config.getComplianceByRule([
  rules.s3BucketEncryption.name,
  rules.s3BucketPublicReadProhibited.name,
  rules.rdsEncryptedStorage.name
]);

console.log('Compliance Status:');
complianceStatus.forEach(rule => {
  console.log(`- ${rule.ruleName}: ${rule.complianceType} (${rule.contributorCount} resources)`);
});

// Get non-compliant S3 buckets
const nonCompliantBuckets = await config.getNonCompliantResources(
  rules.s3BucketEncryption.name
);

console.log('\nNon-Compliant S3 Buckets:');
nonCompliantBuckets.forEach(resource => {
  console.log(`- ${resource.resourceId}`);
  console.log(`  Reason: ${resource.annotation}`);
});

// Manually trigger remediation for specific resources
if (nonCompliantBuckets.length > 0) {
  await config.startRemediation(
    rules.s3BucketEncryption.name,
    nonCompliantBuckets.map(r => ({
      resourceType: r.resourceType,
      resourceId: r.resourceId
    }))
  );
}
```

### Custom Config Rule Lambda:

```javascript
// custom-config-rule-lambda.js
import { ConfigServiceClient, PutEvaluationsCommand } from '@aws-sdk/client-config-service';

const configClient = new ConfigServiceClient();

export const handler = async (event) => {
  console.log('Config Rule Event:', JSON.stringify(event, null, 2));
  
  const configurationItem = JSON.parse(event.configurationItem);
  const ruleParameters = JSON.parse(event.ruleParameters || '{}');
  
  // Evaluate compliance
  const compliance = evaluateCompliance(configurationItem, ruleParameters);
  
  // Report evaluation result
  const evaluation = {
    ComplianceResourceType: configurationItem.resourceType,
    ComplianceResourceId: configurationItem.resourceId,
    ComplianceType: compliance.compliant ? 'COMPLIANT' : 'NON_COMPLIANT',
    Annotation: compliance.reason,
    OrderingTimestamp: new Date(configurationItem.configurationItemCaptureTime)
  };
  
  const command = new PutEvaluationsCommand({
    Evaluations: [evaluation],
    ResultToken: event.resultToken
  });
  
  await configClient.send(command);
  
  return { statusCode: 200 };
};

function evaluateCompliance(configItem, parameters) {
  // Example: Check if S3 bucket has logging enabled
  if (configItem.resourceType === 'AWS::S3::Bucket') {
    const logging = configItem.configuration.logging;
    
    if (!logging || !logging.enabled) {
      return {
        compliant: false,
        reason: 'S3 bucket logging is not enabled'
      };
    }
  }
  
  // Example: Check if EC2 instance has required tags
  if (configItem.resourceType === 'AWS::EC2::Instance') {
    const requiredTags = parameters.requiredTags || ['Environment', 'Owner'];
    const tags = configItem.tags || {};
    
    const missingTags = requiredTags.filter(tag => !tags[tag]);
    
    if (missingTags.length > 0) {
      return {
        compliant: false,
        reason: `Missing required tags: ${missingTags.join(', ')}`
      };
    }
  }
  
  return { compliant: true, reason: 'Resource is compliant' };
}
```

### Best Practices:

1. ✅ Enable Config in all regions
2. ✅ Use conformance packs for compliance frameworks
3. ✅ Setup automatic remediation where possible
4. ✅ Monitor Config rule compliance regularly
5. ✅ Use aggregators for multi-account view
6. ✅ Enable SNS notifications for changes
7. ✅ Integrate with Security Hub
8. ✅ Use custom rules for specific requirements
9. ✅ Track configuration changes over time
10. ✅ Export data to S3 for analysis

---

## Question 48: Explain Amazon GuardDuty for threat detection

### 📋 Answer

**Amazon GuardDuty** is a threat detection service that continuously monitors for malicious activity and unauthorized behavior to protect your AWS accounts, workloads, and data.

### GuardDuty Architecture:

```
Amazon GuardDuty
├── Data Sources
│   ├── CloudTrail event logs
│   ├── VPC Flow Logs
│   ├── DNS logs
│   ├── EKS audit logs
│   ├── RDS login activity
│   └── S3 data events
├── Threat Intelligence
│   ├── AWS security intelligence
│   ├── CrowdStrike threat feeds
│   ├── Proofpoint threat feeds
│   └── Custom threat lists
├── Machine Learning
│   ├── Anomaly detection
│   ├── Behavioral analysis
│   └── Pattern recognition
├── Finding Types
│   ├── Reconnaissance (port scanning)
│   ├── Instance compromise
│   ├── Account compromise
│   ├── Malware
│   ├── Data exfiltration
│   └── Cryptocurrency mining
└── Integration
    ├── Security Hub
    ├── EventBridge
    ├── Detective
    └── Lambda (automated response)
```

### Complete GuardDuty Implementation:

```javascript
// guardduty-service.js
import {
  GuardDutyClient,
  CreateDetectorCommand,
  GetDetectorCommand,
  UpdateDetectorCommand,
  ListFindingsCommand,
  GetFindingsCommand,
  ArchiveFindingsCommand,
  CreateThreatIntelSetCommand,
  CreateIPSetCommand,
  CreateFilterCommand
} from '@aws-sdk/client-guardduty';

const guarddutyClient = new GuardDutyClient({ region: 'us-east-1' });

class GuardDutyService {
  constructor(detectorId = null) {
    this.detectorId = detectorId;
  }
  
  // 1. Enable GuardDuty
  async enableGuardDuty() {
    const command = new CreateDetectorCommand({
      Enable: true,
      FindingPublishingFrequency: 'FIFTEEN_MINUTES',  // FIFTEEN_MINUTES, ONE_HOUR, SIX_HOURS
      DataSources: {
        S3Logs: {
          Enable: true
        },
        Kubernetes: {
          AuditLogs: {
            Enable: true
          }
        }
      },
      Tags: {
        Environment: 'Production',
        ManagedBy: 'SecurityTeam'
      }
    });
    
    try {
      const response = await guarddutyClient.send(command);
      this.detectorId = response.DetectorId;
      console.log('GuardDuty enabled:', this.detectorId);
      return this.detectorId;
    } catch (error) {
      console.error('Failed to enable GuardDuty:', error);
      throw error;
    }
  }
  
  // 2. Get Findings
  async getFindings(criteria = {}, maxResults = 50) {
    // List finding IDs
    const listCommand = new ListFindingsCommand({
      DetectorId: this.detectorId,
      FindingCriteria: {
        Criterion: {
          'severity': { Gte: criteria.minSeverity || 4 },  // Medium and above
          'service.archived': { Eq: ['false'] },
          ...criteria.additional
        }
      },
      SortCriteria: {
        AttributeName: 'severity',
        OrderBy: 'DESC'
      },
      MaxResults: maxResults
    });
    
    const listResponse = await guarddutyClient.send(listCommand);
    
    if (listResponse.FindingIds.length === 0) {
      return [];
    }
    
    // Get finding details
    const getCommand = new GetFindingsCommand({
      DetectorId: this.detectorId,
      FindingIds: listResponse.FindingIds
    });
    
    const getResponse = await guarddutyClient.send(getCommand);
    
    return getResponse.Findings.map(finding => ({
      id: finding.Id,
      type: finding.Type,
      severity: finding.Severity,
      title: finding.Title,
      description: finding.Description,
      resource: {
        type: finding.Resource.ResourceType,
        instanceId: finding.Resource.InstanceDetails?.InstanceId,
        accessKeyId: finding.Resource.AccessKeyDetails?.AccessKeyId
      },
      service: {
        action: finding.Service.Action,
        count: finding.Service.Count,
        firstSeen: finding.Service.EventFirstSeen,
        lastSeen: finding.Service.EventLastSeen
      },
      createdAt: finding.CreatedAt,
      updatedAt: finding.UpdatedAt
    }));
  }
  
  // 3. Archive Findings
  async archiveFindings(findingIds) {
    const command = new ArchiveFindingsCommand({
      DetectorId: this.detectorId,
      FindingIds: findingIds
    });
    
    await guarddutyClient.send(command);
    console.log('Findings archived:', findingIds.length);
  }
  
  // 4. Create Threat Intel Set
  async createThreatIntelSet(name, location, format = 'TXT') {
    const command = new CreateThreatIntelSetCommand({
      DetectorId: this.detectorId,
      Name: name,
      Format: format,  // TXT, STIX, OTX_CSV, ALIEN_VAULT, PROOF_POINT, FIRE_EYE
      Location: location,  // S3 URI
      Activate: true
    });
    
    const response = await guarddutyClient.send(command);
    console.log('Threat intel set created:', response.ThreatIntelSetId);
    return response.ThreatIntelSetId;
  }
  
  // 5. Create Trusted IP Set
  async createTrustedIPSet(name, location) {
    const command = new CreateIPSetCommand({
      DetectorId: this.detectorId,
      Name: name,
      Format: 'TXT',
      Location: location,
      Activate: true
    });
    
    const response = await guarddutyClient.send(command);
    console.log('Trusted IP set created:', response.IpSetId);
    return response.IpSetId;
  }
  
  // 6. Create Suppression Filter
  async createSuppressionFilter(name, description, findingCriteria) {
    const command = new CreateFilterCommand({
      DetectorId: this.detectorId,
      Name: name,
      Description: description,
      Action: 'ARCHIVE',  // Archive matching findings
      FindingCriteria: {
        Criterion: findingCriteria
      },
      Rank: 1
    });
    
    const response = await guarddutyClient.send(command);
    console.log('Suppression filter created:', response.Name);
    return response.Name;
  }
  
  // 7. Generate Security Report
  async generateSecurityReport() {
    const findings = await this.getFindings({ minSeverity: 0 }, 1000);
    
    const report = {
      totalFindings: findings.length,
      bySeverity: {
        critical: findings.filter(f => f.severity >= 7).length,
        high: findings.filter(f => f.severity >= 4 && f.severity < 7).length,
        medium: findings.filter(f => f.severity >= 2 && f.severity < 4).length,
        low: findings.filter(f => f.severity < 2).length
      },
      byType: this.groupBy(findings, 'type'),
      topFindings: findings.slice(0, 10).map(f => ({
        type: f.type,
        title: f.title,
        severity: f.severity,
        resource: f.resource.instanceId || f.resource.accessKeyId,
        count: f.service.count
      })),
      timeline: {
        oldest: findings[findings.length - 1]?.createdAt,
        newest: findings[0]?.createdAt
      }
    };
    
    return report;
  }
  
  groupBy(array, key) {
    return array.reduce((result, item) => {
      const group = item[key];
      result[group] = (result[group] || 0) + 1;
      return result;
    }, {});
  }
}

// Automated Response System
class GuardDutyResponseOrchestrator {
  // Setup automated response to findings
  static async setupAutomatedResponse() {
    // EventBridge rule for high severity findings
    const eventPattern = {
      source: ['aws.guardduty'],
      'detail-type': ['GuardDuty Finding'],
      detail: {
        severity: [{ numeric: ['>=', 4] }]  // Medium and above
      }
    };
    
    return {
      ruleName: 'guardduty-high-severity-response',
      eventPattern: JSON.stringify(eventPattern),
      targets: [
        {
          lambdaArn: 'arn:aws:lambda:us-east-1:123456789012:function:guardduty-response',
          description: 'Automated response to GuardDuty findings'
        }
      ]
    };
  }
  
  // Response actions by finding type
  static getResponseActions() {
    return {
      'UnauthorizedAccess:EC2/SSHBruteForce': {
        action: 'isolate_instance',
        description: 'Isolate EC2 instance under SSH brute force attack'
      },
      
      'CryptoCurrency:EC2/BitcoinTool.B!DNS': {
        action: 'terminate_instance',
        description: 'Terminate instance running cryptocurrency mining'
      },
      
      'UnauthorizedAccess:IAMUser/MaliciousIPCaller.Custom': {
        action: 'disable_access_key',
        description: 'Disable IAM access key used from malicious IP'
      },
      
      'Trojan:EC2/BlackholeTraffic': {
        action: 'isolate_and_snapshot',
        description: 'Isolate instance and create forensic snapshot'
      },
      
      'Recon:EC2/PortProbeUnprotectedPort': {
        action: 'restrict_security_group',
        description: 'Restrict security group rules'
      }
    };
  }
}

// Usage - Complete GuardDuty Setup
const guardduty = new GuardDutyService();

// Enable GuardDuty
const detectorId = await guardduty.enableGuardDuty();

// Create trusted IP set (corporate IP ranges)
const trustedIPs = `
10.0.0.0/8
172.16.0.0/12
192.168.1.0/24
`;

// Upload to S3 first
// await s3.putObject({ Bucket: 'security-config', Key: 'trusted-ips.txt', Body: trustedIPs });

await guardduty.createTrustedIPSet(
  'corporate-network-ips',
  's3://security-config/trusted-ips.txt'
);

// Create suppression filter for known false positives
await guardduty.createSuppressionFilter(
  'suppress-scheduled-task-findings',
  'Suppress findings from scheduled maintenance tasks',
  {
    'type': {
      Eq: ['Recon:EC2/PortProbeUnprotectedPort']
    },
    'resource.instanceDetails.instanceId': {
      Eq: ['i-maintenance12345']
    }
  }
);

// Get current findings
const findings = await guardduty.getFindings({ minSeverity: 4 });

console.log(`Found ${findings.length} medium+ severity findings`);

findings.forEach(finding => {
  console.log(`\n[${finding.severity}] ${finding.type}`);
  console.log(`Title: ${finding.title}`);
  console.log(`Description: ${finding.description}`);
  console.log(`Resource: ${finding.resource.type}`);
  console.log(`First Seen: ${finding.service.firstSeen}`);
  console.log(`Count: ${finding.service.count}`);
});

// Generate security report
const report = await guardduty.generateSecurityReport();

console.log('\n=== GuardDuty Security Report ===');
console.log(`Total Findings: ${report.totalFindings}`);
console.log('\nBy Severity:');
console.log(`- Critical: ${report.bySeverity.critical}`);
console.log(`- High: ${report.bySeverity.high}`);
console.log(`- Medium: ${report.bySeverity.medium}`);
console.log(`- Low: ${report.bySeverity.low}`);

console.log('\nTop 5 Finding Types:');
Object.entries(report.byType)
  .sort((a, b) => b[1] - a[1])
  .slice(0, 5)
  .forEach(([type, count]) => {
    console.log(`- ${type}: ${count}`);
  });

// Archive resolved findings
const resolvedFindingIds = findings
  .filter(f => f.title.includes('Resolved'))
  .map(f => f.id);

if (resolvedFindingIds.length > 0) {
  await guardduty.archiveFindings(resolvedFindingIds);
}
```

### Automated Response Lambda:

```javascript
// guardduty-response-lambda.js
import { EC2Client, ModifyInstanceAttributeCommand, CreateSnapshotCommand } from '@aws-sdk/client-ec2';
import { IAMClient, UpdateAccessKeyCommand } from '@aws-sdk/client-iam';
import { SNSClient, PublishCommand } from '@aws-sdk/client-sns';

const ec2Client = new EC2Client();
const iamClient = new IAMClient();
const snsClient = new SNSClient();

export const handler = async (event) => {
  console.log('GuardDuty Finding:', JSON.stringify(event, null, 2));
  
  const finding = event.detail;
  const severity = finding.severity;
  const findingType = finding.type;
  
  let action = 'notified';
  
  try {
    // Response based on finding type
    if (findingType.includes('CryptoCurrency')) {
      await terminateInstance(finding);
      action = 'instance_terminated';
    } else if (findingType.includes('SSHBruteForce')) {
      await isolateInstance(finding);
      action = 'instance_isolated';
    } else if (findingType.includes('MaliciousIPCaller')) {
      await disableAccessKey(finding);
      action = 'access_key_disabled';
    } else if (findingType.includes('BlackholeTraffic')) {
      await isolateAndSnapshot(finding);
      action = 'instance_isolated_and_snapshot_created';
    }
    
    // Send notification
    await sendSecurityNotification(finding, action);
    
    return { statusCode: 200, action };
  } catch (error) {
    console.error('Response failed:', error);
    await sendSecurityNotification(finding, 'auto_response_failed');
    throw error;
  }
};

async function isolateInstance(finding) {
  const instanceId = finding.resource.instanceDetails.instanceId;
  
  // Create forensic security group (deny all traffic)
  const command = new ModifyInstanceAttributeCommand({
    InstanceId: instanceId,
    Groups: ['sg-forensics-isolated']  // Pre-created SG with no rules
  });
  
  await ec2Client.send(command);
  console.log('Instance isolated:', instanceId);
}

async function terminateInstance(finding) {
  const instanceId = finding.resource.instanceDetails.instanceId;
  
  // In production, you might want to snapshot first
  await createSnapshot(instanceId);
  
  // Then terminate
  // await ec2Client.send(new TerminateInstancesCommand({ InstanceIds: [instanceId] }));
  console.log('Instance terminated:', instanceId);
}

async function disableAccessKey(finding) {
  const accessKeyId = finding.resource.accessKeyDetails.accessKeyId;
  const userName = finding.resource.accessKeyDetails.userName;
  
  const command = new UpdateAccessKeyCommand({
    AccessKeyId: accessKeyId,
    UserName: userName,
    Status: 'Inactive'
  });
  
  await iamClient.send(command);
  console.log('Access key disabled:', accessKeyId);
}

async function isolateAndSnapshot(finding) {
  const instanceId = finding.resource.instanceDetails.instanceId;
  
  // Create snapshot for forensics
  await createSnapshot(instanceId);
  
  // Isolate instance
  await isolateInstance(finding);
}

async function createSnapshot(instanceId) {
  // Get volume IDs from instance
  // Create snapshots
  console.log('Creating forensic snapshot for:', instanceId);
}

async function sendSecurityNotification(finding, action) {
  const message = {
    Subject: `[GuardDuty] ${finding.severity} - ${finding.type}`,
    Message: `
GuardDuty Finding Alert

Severity: ${finding.severity}
Type: ${finding.type}
Title: ${finding.title}
Description: ${finding.description}

Resource:
- Type: ${finding.resource.resourceType}
- Instance: ${finding.resource.instanceDetails?.instanceId || 'N/A'}
- Access Key: ${finding.resource.accessKeyDetails?.accessKeyId || 'N/A'}

Action Taken: ${action}

First Seen: ${finding.service.eventFirstSeen}
Last Seen: ${finding.service.eventLastSeen}
Count: ${finding.service.count}

Account: ${finding.accountId}
Region: ${finding.region}
    `.trim()
  };
  
  const command = new PublishCommand({
    TopicArn: 'arn:aws:sns:us-east-1:123456789012:security-alerts',
    Subject: message.Subject,
    Message: message.Message
  });
  
  await snsClient.send(command);
}
```

### Best Practices:

1. ✅ Enable GuardDuty in all regions
2. ✅ Enable S3 and EKS protection
3. ✅ Setup automated response with EventBridge
4. ✅ Create trusted IP lists for known sources
5. ✅ Use suppression filters for false positives
6. ✅ Integrate with Security Hub
7. ✅ Export findings to S3 for analysis
8. ✅ Monitor finding trends over time
9. ✅ Review and archive findings regularly
10. ✅ Enable multi-account GuardDuty

---

## Key Takeaways

### Question 46 (Security Hub):
- Centralized security view across AWS
- Aggregates findings from multiple services
- Security standards compliance checking
- Automated remediation with EventBridge
- Custom insights for security metrics
- Multi-account and multi-region support

### Question 47 (AWS Config):
- Track resource configuration changes
- 200+ managed compliance rules
- Automatic remediation with SSM
- Configuration history and snapshots
- Conformance packs for compliance frameworks
- Integration with Security Hub

### Question 48 (GuardDuty):
- AI/ML-powered threat detection
- Continuous monitoring of AWS accounts
- Analyzes CloudTrail, VPC Flow, DNS logs
- Identifies compromised instances and accounts
- Automated response to security threats
- Low operational overhead
