# Feature Flags & Experimentation

## Question 51: Feature Flags with LaunchDarkly/AWS AppConfig

### 📋 Question Statement

Implement feature flags and A/B testing for Emirates NBD using AWS AppConfig for gradual feature rollouts and experimentation.

---

### 🚩 Feature Flags Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    FEATURE FLAG SYSTEM                                      │
└────────────────────────────────────────────────────────────────────────────┘

    ┌──────────────┐         ┌─────────────────┐
    │ Application  │────────>│  AWS AppConfig  │
    │   Service    │         │  Agent/SDK      │
    └──────────────┘         └────────┬────────┘
                                      │
                                      v
                             ┌────────────────┐
                             │  AppConfig     │
                             │  Service       │
                             └────────┬───────┘
                                      │
                        ┌─────────────┴─────────────┐
                        │                           │
                        v                           v
                 ┌──────────┐              ┌───────────┐
                 │  Config  │              │CloudWatch │
                 │  Store   │              │  Metrics  │
                 └──────────┘              └───────────┘
```

### 📦 AppConfig CDK Infrastructure

```typescript
// infrastructure/cdk/appconfig-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as appconfig from 'aws-cdk-lib/aws-appconfig';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as s3 from 'aws-cdk-lib/aws-s3';

export class AppConfigStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // S3 bucket for config storage
    const configBucket = new s3.Bucket(this, 'ConfigBucket', {
      bucketName: 'emirates-nbd-feature-flags',
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED
    });

    // AppConfig Application
    const application = new appconfig.CfnApplication(this, 'BankingApp', {
      name: 'emirates-nbd-banking',
      description: 'Banking platform feature flags and configuration'
    });

    // Environments
    const environments = ['dev', 'staging', 'production'].map((env) =>
      new appconfig.CfnEnvironment(this, `${env}Environment`, {
        applicationId: application.ref,
        name: env,
        description: `${env.charAt(0).toUpperCase() + env.slice(1)} environment`,
        monitors: env === 'production' ? [
          {
            alarmArn: `arn:aws:cloudwatch:${this.region}:${this.account}:alarm:HighErrorRate`,
            alarmRoleArn: `arn:aws:iam::${this.account}:role/AppConfigMonitorRole`
          }
        ] : []
      })
    );

    // Configuration Profile - Feature Flags
    const featureFlagProfile = new appconfig.CfnConfigurationProfile(this, 'FeatureFlags', {
      applicationId: application.ref,
      name: 'feature-flags',
      description: 'Feature flags for progressive rollout',
      locationUri: 'hosted',
      type: 'AWS.AppConfig.FeatureFlags',
      validators: [
        {
          type: 'JSON_SCHEMA',
          content: JSON.stringify({
            $schema: 'http://json-schema.org/draft-07/schema#',
            type: 'object',
            properties: {
              flags: {
                type: 'object'
              }
            }
          })
        }
      ]
    });

    // Deployment Strategies
    const canary10 = new appconfig.CfnDeploymentStrategy(this, 'Canary10', {
      name: 'Canary10Percent10Minutes',
      description: 'Deploy to 10% for 10 minutes',
      deploymentDurationInMinutes: 10,
      growthFactor: 10,
      replicateTo: 'NONE',
      finalBakeTimeInMinutes: 5
    });

    const canary25 = new appconfig.CfnDeploymentStrategy(this, 'Canary25', {
      name: 'Canary25Percent20Minutes',
      description: 'Deploy to 25% for 20 minutes',
      deploymentDurationInMinutes: 20,
      growthFactor: 25,
      replicateTo: 'NONE',
      finalBakeTimeInMinutes: 10
    });

    const allAtOnce = new appconfig.CfnDeploymentStrategy(this, 'AllAtOnce', {
      name: 'AllAtOnce',
      description: 'Deploy immediately to 100%',
      deploymentDurationInMinutes: 0,
      growthFactor: 100,
      replicateTo: 'NONE',
      finalBakeTimeInMinutes: 0
    });

    // Lambda to manage feature flags
    const flagManagementFunction = new lambda.Function(this, 'FlagManagement', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/feature-flags'),
      environment: {
        APPLICATION_ID: application.ref,
        ENVIRONMENT_ID: environments[2].ref, // production
        CONFIGURATION_PROFILE_ID: featureFlagProfile.ref
      },
      timeout: cdk.Duration.seconds(30)
    });

    // Grant permissions
    flagManagementFunction.addToRolePolicy(
      new iam.PolicyStatement({
        actions: [
          'appconfig:GetConfiguration',
          'appconfig:StartConfigurationSession',
          'appconfig:GetLatestConfiguration'
        ],
        resources: ['*']
      })
    );

    // Outputs
    new cdk.CfnOutput(this, 'ApplicationId', {
      value: application.ref,
      description: 'AppConfig Application ID'
    });

    new cdk.CfnOutput(this, 'FeatureFlagProfileId', {
      value: featureFlagProfile.ref,
      description: 'Feature Flag Configuration Profile ID'
    });
  }
}
```

### 🎛️ Feature Flags Configuration

```json
{
  "flags": {
    "newPaymentUI": {
      "name": "New Payment UI",
      "description": "Enable new payment interface",
      "_deprecation": {
        "status": "NOT_DEPRECATED"
      },
      "attributes": {
        "userTier": {
          "constraints": {
            "type": "string",
            "enum": ["premium", "standard", "basic"]
          }
        },
        "region": {
          "constraints": {
            "type": "string",
            "enum": ["UAE", "KSA", "EGYPT"]
          }
        }
      }
    },
    "realTimeFraudDetection": {
      "name": "Real-time Fraud Detection",
      "description": "Enable ML-powered fraud detection",
      "_deprecation": {
        "status": "NOT_DEPRECATED"
      }
    },
    "instantTransfers": {
      "name": "Instant Transfers",
      "description": "Enable real-time payment processing",
      "_deprecation": {
        "status": "NOT_DEPRECATED"
      }
    },
    "biometricAuth": {
      "name": "Biometric Authentication",
      "description": "Enable fingerprint/face recognition",
      "_deprecation": {
        "status": "NOT_DEPRECATED"
      }
    }
  },
  "values": {
    "newPaymentUI": {
      "enabled": true,
      "rules": [
        {
          "conditions": [
            {
              "attribute": "userTier",
              "comparison": "equals",
              "value": "premium"
            }
          ],
          "value": {
            "enabled": true
          }
        },
        {
          "conditions": [
            {
              "attribute": "region",
              "comparison": "equals",
              "value": "UAE"
            }
          ],
          "value": {
            "enabled": true
          }
        }
      ],
      "rollout": {
        "percentage": 25,
        "seed": 12345
      }
    },
    "realTimeFraudDetection": {
      "enabled": true
    },
    "instantTransfers": {
      "enabled": false,
      "rollout": {
        "percentage": 10,
        "seed": 67890
      }
    },
    "biometricAuth": {
      "enabled": false
    }
  },
  "version": "1"
}
```

### 🔧 Feature Flag SDK Integration

```typescript
// services/common/feature-flags.ts
import { AppConfigDataClient, StartConfigurationSessionCommand, GetLatestConfigurationCommand } from '@aws-sdk/client-appconfigdata';

export class FeatureFlagService {
  private client: AppConfigDataClient;
  private configurationToken?: string;
  private cachedConfig: any = {};
  private lastPollTime: number = 0;
  private pollIntervalMs: number = 30000; // 30 seconds

  constructor(
    private applicationId: string,
    private environmentId: string,
    private configurationProfileId: string
  ) {
    this.client = new AppConfigDataClient({});
  }

  async initialize(): Promise<void> {
    // Start configuration session
    const sessionResponse = await this.client.send(
      new StartConfigurationSessionCommand({
        ApplicationIdentifier: this.applicationId,
        EnvironmentIdentifier: this.environmentId,
        ConfigurationProfileIdentifier: this.configurationProfileId
      })
    );

    this.configurationToken = sessionResponse.InitialConfigurationToken;
    await this.refreshConfiguration();
  }

  async isEnabled(flagName: string, context?: FlagContext): Promise<boolean> {
    // Refresh config if needed
    if (Date.now() - this.lastPollTime > this.pollIntervalMs) {
      await this.refreshConfiguration();
    }

    const flag = this.cachedConfig.values?.[flagName];
    if (!flag) return false;

    // Check if flag is globally enabled
    if (!flag.enabled) return false;

    // Check rules
    if (flag.rules && context) {
      for (const rule of flag.rules) {
        if (this.evaluateRule(rule, context)) {
          return rule.value.enabled;
        }
      }
    }

    // Check percentage rollout
    if (flag.rollout) {
      const hash = this.hashString(context?.userId || '', flag.rollout.seed);
      const bucket = hash % 100;
      return bucket < flag.rollout.percentage;
    }

    return flag.enabled;
  }

  async getValue<T>(flagName: string, defaultValue: T, context?: FlagContext): Promise<T> {
    const isEnabled = await this.isEnabled(flagName, context);
    if (!isEnabled) return defaultValue;

    const flag = this.cachedConfig.values?.[flagName];
    return flag.value ?? defaultValue;
  }

  private async refreshConfiguration(): Promise<void> {
    try {
      const response = await this.client.send(
        new GetLatestConfigurationCommand({
          ConfigurationToken: this.configurationToken
        })
      );

      if (response.Configuration) {
        const configString = new TextDecoder().decode(response.Configuration);
        this.cachedConfig = JSON.parse(configString);
        console.log('Configuration refreshed:', this.cachedConfig);
      }

      this.configurationToken = response.NextPollConfigurationToken;
      this.lastPollTime = Date.now();

    } catch (error) {
      console.error('Failed to refresh configuration:', error);
    }
  }

  private evaluateRule(rule: any, context: FlagContext): boolean {
    return rule.conditions.every((condition: any) => {
      const contextValue = (context as any)[condition.attribute];
      
      switch (condition.comparison) {
        case 'equals':
          return contextValue === condition.value;
        case 'notEquals':
          return contextValue !== condition.value;
        case 'in':
          return condition.value.includes(contextValue);
        case 'notIn':
          return !condition.value.includes(contextValue);
        default:
          return false;
      }
    });
  }

  private hashString(str: string, seed: number): number {
    let hash = seed;
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i);
      hash = hash & hash; // Convert to 32-bit integer
    }
    return Math.abs(hash);
  }
}

export interface FlagContext {
  userId?: string;
  userTier?: string;
  region?: string;
  deviceType?: string;
  [key: string]: any;
}

// Singleton instance
let featureFlagService: FeatureFlagService;

export async function initializeFeatureFlags(): Promise<void> {
  featureFlagService = new FeatureFlagService(
    process.env.APPCONFIG_APPLICATION_ID!,
    process.env.APPCONFIG_ENVIRONMENT_ID!,
    process.env.APPCONFIG_PROFILE_ID!
  );

  await featureFlagService.initialize();
}

export function getFeatureFlags(): FeatureFlagService {
  if (!featureFlagService) {
    throw new Error('Feature flags not initialized');
  }
  return featureFlagService;
}
```

### 🎯 Usage in Application

```typescript
// services/payment/payment-controller.ts
import { getFeatureFlags } from '../common/feature-flags';

export class PaymentController {
  
  async processPayment(req: any, res: any): Promise<void> {
    const { userId, amount, beneficiaryId } = req.body;
    const featureFlags = getFeatureFlags();

    // Check if instant transfers are enabled for this user
    const instantEnabled = await featureFlags.isEnabled('instantTransfers', {
      userId,
      userTier: req.user.tier,
      region: req.user.region
    });

    if (instantEnabled) {
      // Use instant transfer
      const result = await this.instantTransferService.process({
        userId,
        amount,
        beneficiaryId
      });

      res.json({
        transferId: result.id,
        status: 'COMPLETED',
        method: 'instant',
        completedAt: new Date().toISOString()
      });

    } else {
      // Use standard transfer (next business day)
      const result = await this.standardTransferService.schedule({
        userId,
        amount,
        beneficiaryId
      });

      res.json({
        transferId: result.id,
        status: 'SCHEDULED',
        method: 'standard',
        scheduledFor: result.scheduledDate
      });
    }
  }

  async renderPaymentUI(req: any, res: any): Promise<void> {
    const featureFlags = getFeatureFlags();

    // Check which UI version to show
    const newUIEnabled = await featureFlags.isEnabled('newPaymentUI', {
      userId: req.user.id,
      userTier: req.user.tier,
      region: req.user.region
    });

    if (newUIEnabled) {
      res.render('payment-v2', {
        features: ['quick-pay', 'beneficiary-groups', 'scheduled-payments']
      });
    } else {
      res.render('payment-v1');
    }
  }
}
```

### 🎓 Interview Discussion Points - Q51

**Q1: What are feature flags and why use them?**

**A**:
**Benefits:**
- **Gradual rollout**: Deploy code but enable for subset of users
- **A/B testing**: Test variations with different user segments
- **Kill switch**: Quickly disable problematic features
- **Canary releases**: Test with 1-10% before full rollout
- **Targeted releases**: Enable for specific users/regions

**Q2: How to implement percentage rollouts?**

**A**:
- **Consistent hashing**: Hash user ID with seed
- **Bucket assignment**: Assign to bucket 0-99
- **Percentage check**: Enable if bucket < percentage
- **Sticky behavior**: Same user always in same bucket
- **Gradual increase**: 1% → 5% → 10% → 25% → 50% → 100%

**Q3: How to handle feature flag technical debt?**

**A**:
- **Expiration dates**: Set TTL on flags
- **Deprecation status**: Mark flags for removal
- **Monitoring**: Track unused flags
- **Cleanup automation**: Remove after full rollout
- **Code review**: Enforce flag removal in PRs

**Q4: What are feature flag best practices?**

**A**:
- **Short-lived**: Remove after rollout complete
- **Clear naming**: Descriptive flag names
- **Documentation**: Document purpose and rollout plan
- **Testing**: Test both enabled/disabled paths
- **Monitoring**: Track flag evaluation metrics
- **Fallback**: Always have safe default value

**Q5: How to manage feature flags at scale?**

**A**:
- **Centralized service**: AWS AppConfig, LaunchDarkly
- **Caching**: Cache config locally (30-60s TTL)
- **Async updates**: Background polling
- **Graceful degradation**: Continue with cached values if service down
- **Versioning**: Track config changes
- **Audit logs**: Who changed what and when

---

## Question 52: Feature Flag Targeting & Progressive Rollout

### 📋 Question Statement

Implement advanced feature flag targeting for Emirates NBD including user segmentation, percentage rollouts, environment-based flags, and automated progressive delivery with monitoring.

---

### 🎯 AWS AppConfig Feature Flag Setup

```typescript
// infrastructure/cdk/appconfig-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as appconfig from 'aws-cdk-lib/aws-appconfig';
import * as iam from 'aws-cdk-lib/aws-iam';

export class AppConfigStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // AppConfig Application
    const application = new appconfig.CfnApplication(this, 'BankingFeatureFlags', {
      name: 'banking-feature-flags',
      description: 'Feature flags for Emirates NBD Banking Platform'
    });

    // Environment (Production)
    const environment = new appconfig.CfnEnvironment(this, 'Production', {
      applicationId: application.ref,
      name: 'production',
      description: 'Production environment'
    });

    // Configuration Profile
    const configProfile = new appconfig.CfnConfigurationProfile(this, 'FeatureFlagProfile', {
      applicationId: application.ref,
      name: 'feature-flags',
      locationUri: 'hosted',
      type: 'AWS.AppConfig.FeatureFlags'
    });

    // Deployment Strategy (Canary 10% for 10 minutes)
    const deploymentStrategy = new appconfig.CfnDeploymentStrategy(this, 'Canary10Percent', {
      name: 'Canary10Percent10Minutes',
      deploymentDurationInMinutes: 10,
      growthFactor: 10,
      replicateTo: 'NONE',
      finalBakeTimeInMinutes: 5
    });

    // Feature flags configuration
    const flagsConfig = {
      flags: {
        enableNewPaymentUI: {
          name: 'Enable New Payment UI',
          description: 'Rollout new payment interface',
          _deprecation: { status: 'NOT_DEPRECATED' },
          attributes: {
            userSegment: {
              constraints: {
                type: 'string',
                enum: ['premium', 'standard', 'basic']
              }
            }
          }
        },
        enableRealTimeFraudDetection: {
          name: 'Enable Real-time Fraud Detection',
          _deprecation: { status: 'NOT_DEPRECATED' }
        },
        maxTransactionLimit: {
          name: 'Maximum Transaction Limit',
          _deprecation: { status: 'NOT_DEPRECATED' },
          attributes: {
            limit: {
              constraints: {
                type: 'number',
                minimum: 1000,
                maximum: 100000
              }
            }
          }
        }
      },
      values: {
        enableNewPaymentUI: {
          enabled: true,
          rules: [
            {
              conditions: [
                {
                  attribute: 'userSegment',
                  value: 'premium'
                }
              ],
              value: { enabled: true }
            }
          ],
          _default: { enabled: false }
        },
        enableRealTimeFraudDetection: {
          enabled: true
        },
        maxTransactionLimit: {
          rules: [
            {
              conditions: [
                {
                  attribute: 'userSegment',
                  value: 'premium'
                }
              ],
              value: { limit: 100000 }
            }
          ],
          _default: { limit: 10000 }
        }
      },
      version: '1'
    };

    // Hosted Configuration Version
    const hostedConfig = new appconfig.CfnHostedConfigurationVersion(this, 'FlagsV1', {
      applicationId: application.ref,
      configurationProfileId: configProfile.ref,
      content: JSON.stringify(flagsConfig),
      contentType: 'application/json'
    });
  }
}
```

### 🚀 Feature Flag Service

```typescript
// services/feature-flags/flag-service.ts
import { AppConfig } from 'aws-sdk';

export class FeatureFlagService {
  private appConfig: AppConfig;
  private cache: Map<string, FlagValue>;
  private cacheExpiry: number = 60000; // 1 minute

  constructor(
    private applicationId: string,
    private environmentId: string,
    private configurationProfileId: string
  ) {
    this.appConfig = new AppConfig();
    this.cache = new Map();
  }

  async isFeatureEnabled(
    flagName: string,
    context: UserContext
  ): Promise<boolean> {
    const flagValue = await this.getFlagValue(flagName, context);
    return flagValue.enabled;
  }

  async getFlagValue(
    flagName: string,
    context: UserContext
  ): Promise<FlagValue> {
    const cacheKey = `${flagName}-${JSON.stringify(context)}`;
    
    // Check cache
    const cached = this.cache.get(cacheKey);
    if (cached && Date.now() - cached.timestamp < this.cacheExpiry) {
      return cached;
    }

    // Fetch from AppConfig
    const configuration = await this.appConfig.getConfiguration({
      Application: this.applicationId,
      Environment: this.environmentId,
      Configuration: this.configurationProfileId,
      ClientId: context.userId
    }).promise();

    const flags = JSON.parse(configuration.Content!.toString());
    const flagValue = this.evaluateFlag(flags, flagName, context);

    // Cache result
    this.cache.set(cacheKey, {
      ...flagValue,
      timestamp: Date.now()
    });

    return flagValue;
  }

  private evaluateFlag(
    flags: any,
    flagName: string,
    context: UserContext
  ): FlagValue {
    const flagConfig = flags.values[flagName];

    if (!flagConfig) {
      return { enabled: false, value: null };
    }

    // Evaluate rules
    if (flagConfig.rules) {
      for (const rule of flagConfig.rules) {
        if (this.matchesConditions(rule.conditions, context)) {
          return rule.value;
        }
      }
    }

    // Return default
    return flagConfig._default || { enabled: false };
  }

  private matchesConditions(
    conditions: any[],
    context: UserContext
  ): boolean {
    return conditions.every(condition => {
      const contextValue = context[condition.attribute];
      return contextValue === condition.value;
    });
  }

  async rolloutToPercentage(
    flagName: string,
    percentage: number
  ): Promise<void> {
    // Update flag configuration to rollout to percentage of users
    const hash = this.hashUserId(flagName);
    const enabled = hash % 100 < percentage;

    console.log(`Rollout ${flagName}: ${percentage}% (enabled: ${enabled})`);
  }

  private hashUserId(userId: string): number {
    let hash = 0;
    for (let i = 0; i < userId.length; i++) {
      hash = (hash << 5) - hash + userId.charCodeAt(i);
      hash = hash & hash;
    }
    return Math.abs(hash);
  }
}

export interface UserContext {
  userId: string;
  userSegment?: string;
  region?: string;
  accountType?: string;
  [key: string]: any;
}

interface FlagValue {
  enabled: boolean;
  value?: any;
  timestamp?: number;
}
```

### 📈 Progressive Rollout Orchestrator

```typescript
// services/feature-flags/progressive-rollout.ts
import { FeatureFlagService } from './flag-service';
import { CloudWatch } from 'aws-sdk';

export class ProgressiveRolloutOrchestrator {
  private flagService: FeatureFlagService;
  private cloudwatch: CloudWatch;

  constructor(flagService: FeatureFlagService) {
    this.flagService = flagService;
    this.cloudwatch = new CloudWatch();
  }

  async executeProgressiveRollout(
    flagName: string,
    stages: RolloutStage[]
  ): Promise<RolloutResult> {
    console.log(`Starting progressive rollout for ${flagName}`);

    for (const stage of stages) {
      console.log(`Stage: ${stage.percentage}% for ${stage.durationMinutes} minutes`);

      // Update flag to target percentage
      await this.flagService.rolloutToPercentage(flagName, stage.percentage);

      // Wait for stage duration
      await this.sleep(stage.durationMinutes * 60 * 1000);

      // Monitor metrics
      const metrics = await this.getFeatureMetrics(flagName);

      // Check health
      if (!this.isStageHealthy(metrics, stage.successCriteria)) {
        console.error('Stage unhealthy, rolling back');
        await this.rollback(flagName);
        return {
          success: false,
          failedAtStage: stage.percentage,
          reason: 'Health check failed'
        };
      }

      console.log(`Stage ${stage.percentage}% successful`);
    }

    console.log('Progressive rollout complete!');
    return { success: true };
  }

  private async getFeatureMetrics(flagName: string): Promise<FeatureMetrics> {
    const now = new Date();
    const fiveMinutesAgo = new Date(now.getTime() - 5 * 60 * 1000);

    // Query CloudWatch for custom metrics
    const errorRateData = await this.cloudwatch.getMetricStatistics({
      Namespace: 'Banking/FeatureFlags',
      MetricName: 'ErrorRate',
      Dimensions: [
        { Name: 'FeatureName', Value: flagName }
      ],
      StartTime: fiveMinutesAgo,
      EndTime: now,
      Period: 300,
      Statistics: ['Average']
    }).promise();

    const conversionData = await this.cloudwatch.getMetricStatistics({
      Namespace: 'Banking/FeatureFlags',
      MetricName: 'ConversionRate',
      Dimensions: [
        { Name: 'FeatureName', Value: flagName }
      ],
      StartTime: fiveMinutesAgo,
      EndTime: now,
      Period: 300,
      Statistics: ['Average']
    }).promise();

    return {
      errorRate: errorRateData.Datapoints[0]?.Average || 0,
      conversionRate: conversionData.Datapoints[0]?.Average || 0
    };
  }

  private isStageHealthy(
    metrics: FeatureMetrics,
    criteria: SuccessCriteria
  ): boolean {
    if (metrics.errorRate > criteria.maxErrorRate) {
      console.error(`Error rate too high: ${metrics.errorRate}`);
      return false;
    }

    if (metrics.conversionRate < criteria.minConversionRate) {
      console.error(`Conversion rate too low: ${metrics.conversionRate}`);
      return false;
    }

    return true;
  }

  private async rollback(flagName: string): Promise<void> {
    await this.flagService.rolloutToPercentage(flagName, 0);
    console.log('Rollback complete');
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

interface RolloutStage {
  percentage: number;
  durationMinutes: number;
  successCriteria: SuccessCriteria;
}

interface SuccessCriteria {
  maxErrorRate: number;
  minConversionRate: number;
}

interface FeatureMetrics {
  errorRate: number;
  conversionRate: number;
}

interface RolloutResult {
  success: boolean;
  failedAtStage?: number;
  reason?: string;
}
```

### 🎓 Interview Discussion Points - Q52

**Q1: What are feature flags used for?**

**A**:
- **Progressive rollout**: 1% → 10% → 50% → 100%
- **A/B testing**: Compare variants
- **Kill switch**: Disable problematic features instantly
- **Environment-based**: Different flags per env
- **User targeting**: Enable for specific segments

**Q2: How to implement percentage-based rollout?**

**A**:
```typescript
const hash = hashUserId(userId) % 100;
const enabled = hash < rolloutPercentage;
```
**Consistent**: Same user always gets same variant

**Q3: What is the difference between feature flags and canary deployment?**

**A**:
- **Feature flags**: Application-level toggle, instant on/off
- **Canary deployment**: Infrastructure-level, traffic routing
- **Use both**: Canary for infrastructure, flags for features

**Q4: How to handle flag removal?**

**A**:
1. Rollout to 100%
2. Wait 2 weeks (ensure no rollback needed)
3. Remove flag code
4. Clean up flag configuration
**Anti-pattern**: Leaving flags indefinitely (technical debt)

**Q5: How to prevent flag evaluation latency?**

**A**:
- **Client-side caching**: 1-5 minute TTL
- **Server-side caching**: In-memory cache
- **Default values**: Fail gracefully if AppConfig unavailable
- **Bulk fetch**: Get all flags in one call

---

**End of File 26**