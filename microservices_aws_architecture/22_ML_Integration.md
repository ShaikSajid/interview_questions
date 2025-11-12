# Machine Learning Integration

## Question 43: ML-powered Fraud Detection

### 📋 Question Statement

Implement real-time fraud detection for Emirates NBD using SageMaker, Lambda, and Kinesis for ML model inference on transaction streams.

---

### 🤖 ML Fraud Detection Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│              SAGEMAKER FRAUD DETECTION PIPELINE                             │
└────────────────────────────────────────────────────────────────────────────┘

    ┌──────────────┐         ┌─────────────────┐
    │ Transaction  │────────>│  API Gateway    │
    │   Request    │         │   /predict      │
    └──────────────┘         └────────┬────────┘
                                      │
                                      v
                             ┌────────────────┐
                             │  Lambda        │
                             │  Preprocessor  │
                             └────────┬───────┘
                                      │
                                      v
                             ┌────────────────────┐
                             │  SageMaker         │
                             │  Real-time         │
                             │  Endpoint          │
                             └────────┬───────────┘
                                      │
                        ┌─────────────┴─────────────┐
                        │                           │
                        v                           v
                 ┌──────────┐              ┌───────────┐
                 │  DynamoDB│              │ CloudWatch│
                 │  Results │              │  Metrics  │
                 └──────────┘              └───────────┘
```

### 📦 SageMaker Training & Deployment CDK

```typescript
// infrastructure/cdk/sagemaker-fraud-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as sagemaker from 'aws-cdk-lib/aws-sagemaker';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as logs from 'aws-cdk-lib/aws-logs';

export class SageMakerFraudStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // S3 buckets
    const trainingDataBucket = new s3.Bucket(this, 'TrainingData', {
      bucketName: 'emirates-nbd-ml-training-data',
      encryption: s3.BucketEncryption.S3_MANAGED,
      versioned: true
    });

    const modelArtifactsBucket = new s3.Bucket(this, 'ModelArtifacts', {
      bucketName: 'emirates-nbd-ml-models',
      encryption: s3.BucketEncryption.S3_MANAGED,
      versioned: true
    });

    // DynamoDB for predictions
    const predictionsTable = new dynamodb.Table(this, 'Predictions', {
      tableName: 'fraud-predictions',
      partitionKey: { name: 'transactionId', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'timestamp', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      stream: dynamodb.StreamViewType.NEW_IMAGE,
      timeToLiveAttribute: 'ttl'
    });

    // SageMaker execution role
    const sageMakerRole = new iam.Role(this, 'SageMakerRole', {
      assumedBy: new iam.ServicePrincipal('sagemaker.amazonaws.com'),
      managedPolicies: [
        iam.ManagedPolicy.fromAwsManagedPolicyName('AmazonSageMakerFullAccess')
      ]
    });

    trainingDataBucket.grantReadWrite(sageMakerRole);
    modelArtifactsBucket.grantReadWrite(sageMakerRole);

    // SageMaker Model
    const fraudModel = new sagemaker.CfnModel(this, 'FraudDetectionModel', {
      modelName: 'fraud-detection-xgboost',
      executionRoleArn: sageMakerRole.roleArn,
      primaryContainer: {
        image: `683313688378.dkr.ecr.${this.region}.amazonaws.com/sagemaker-xgboost:1.5-1`,
        modelDataUrl: `s3://${modelArtifactsBucket.bucketName}/fraud-model/model.tar.gz`,
        environment: {
          MODEL_NAME: 'fraud-detection',
          INFERENCE_TIMEOUT: '60000'
        }
      }
    });

    // Endpoint Configuration
    const endpointConfig = new sagemaker.CfnEndpointConfig(this, 'EndpointConfig', {
      endpointConfigName: 'fraud-detection-endpoint-config',
      productionVariants: [
        {
          variantName: 'AllTraffic',
          modelName: fraudModel.modelName!,
          initialInstanceCount: 2,
          instanceType: 'ml.m5.xlarge',
          initialVariantWeight: 1.0
        }
      ],
      dataCaptureConfig: {
        enableCapture: true,
        initialSamplingPercentage: 100,
        destinationS3Uri: `s3://${modelArtifactsBucket.bucketName}/data-capture/`,
        captureOptions: [
          { captureMode: 'Input' },
          { captureMode: 'Output' }
        ]
      }
    });

    endpointConfig.addDependency(fraudModel);

    // SageMaker Endpoint
    const endpoint = new sagemaker.CfnEndpoint(this, 'FraudEndpoint', {
      endpointName: 'fraud-detection-endpoint',
      endpointConfigName: endpointConfig.endpointConfigName!
    });

    endpoint.addDependency(endpointConfig);

    // Lambda for inference
    const inferenceFunction = new lambda.Function(this, 'InferenceFunction', {
      runtime: lambda.Runtime.PYTHON_3_11,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/ml-inference'),
      timeout: cdk.Duration.seconds(30),
      memorySize: 1024,
      environment: {
        ENDPOINT_NAME: endpoint.endpointName!,
        PREDICTIONS_TABLE: predictionsTable.tableName,
        FRAUD_THRESHOLD: '0.7'
      },
      logRetention: logs.RetentionDays.ONE_WEEK
    });

    // Grant permissions
    inferenceFunction.addToRolePolicy(new iam.PolicyStatement({
      actions: ['sagemaker:InvokeEndpoint'],
      resources: [endpoint.ref]
    }));

    predictionsTable.grantWriteData(inferenceFunction);

    // API Gateway
    const api = new apigateway.RestApi(this, 'FraudDetectionAPI', {
      restApiName: 'Fraud Detection API',
      description: 'Real-time fraud detection API',
      deployOptions: {
        stageName: 'prod',
        throttlingRateLimit: 1000,
        throttlingBurstLimit: 2000,
        loggingLevel: apigateway.MethodLoggingLevel.INFO,
        dataTraceEnabled: true,
        metricsEnabled: true
      }
    });

    const predict = api.root.addResource('predict');
    predict.addMethod('POST', new apigateway.LambdaIntegration(inferenceFunction), {
      apiKeyRequired: true,
      requestValidator: new apigateway.RequestValidator(this, 'RequestValidator', {
        restApi: api,
        validateRequestBody: true
      })
    });

    // API Key
    const apiKey = api.addApiKey('FraudDetectionApiKey', {
      apiKeyName: 'fraud-detection-key'
    });

    const usagePlan = api.addUsagePlan('UsagePlan', {
      name: 'Standard',
      throttle: {
        rateLimit: 1000,
        burstLimit: 2000
      },
      quota: {
        limit: 1000000,
        period: apigateway.Period.MONTH
      }
    });

    usagePlan.addApiKey(apiKey);
    usagePlan.addApiStage({
      stage: api.deploymentStage
    });

    // Outputs
    new cdk.CfnOutput(this, 'ApiEndpoint', {
      value: api.url,
      description: 'Fraud Detection API Endpoint'
    });

    new cdk.CfnOutput(this, 'SageMakerEndpoint', {
      value: endpoint.endpointName!,
      description: 'SageMaker Endpoint Name'
    });

    new cdk.CfnOutput(this, 'ApiKeyId', {
      value: apiKey.keyId,
      description: 'API Key ID'
    });
  }
}
```

### 🧠 Model Training Script

```python
# training/train.py
import argparse
import os
import pandas as pd
import numpy as np
import xgboost as xgb
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score, precision_score, recall_score, f1_score, roc_auc_score
import joblib
import json

def load_data(data_path):
    """Load training data from S3"""
    print(f"Loading data from {data_path}")
    df = pd.read_csv(f"{data_path}/transactions.csv")
    return df

def feature_engineering(df):
    """Create features for fraud detection"""
    # Amount-based features
    df['amount_log'] = np.log1p(df['amount'])
    df['amount_zscore'] = (df['amount'] - df['amount'].mean()) / df['amount'].std()
    
    # Time-based features
    df['hour'] = pd.to_datetime(df['timestamp']).dt.hour
    df['day_of_week'] = pd.to_datetime(df['timestamp']).dt.dayofweek
    df['is_weekend'] = df['day_of_week'].isin([5, 6]).astype(int)
    df['is_night'] = df['hour'].between(22, 6).astype(int)
    
    # Location-based features
    df['is_foreign'] = (df['country'] != 'UAE').astype(int)
    
    # Velocity features (transactions in last hour)
    df = df.sort_values(['customer_id', 'timestamp'])
    df['txn_count_1h'] = df.groupby('customer_id')['timestamp'].transform(
        lambda x: pd.to_datetime(x).rolling('1H').count()
    )
    
    # Merchant category risk
    high_risk_categories = ['GAMBLING', 'ADULT', 'CRYPTO']
    df['is_high_risk_merchant'] = df['merchant_category'].isin(high_risk_categories).astype(int)
    
    return df

def train_model(train_data, val_data):
    """Train XGBoost fraud detection model"""
    feature_columns = [
        'amount', 'amount_log', 'amount_zscore',
        'hour', 'day_of_week', 'is_weekend', 'is_night',
        'is_foreign', 'txn_count_1h', 'is_high_risk_merchant'
    ]
    
    X_train = train_data[feature_columns]
    y_train = train_data['is_fraud']
    
    X_val = val_data[feature_columns]
    y_val = val_data['is_fraud']
    
    # Handle class imbalance
    scale_pos_weight = len(y_train[y_train == 0]) / len(y_train[y_train == 1])
    
    params = {
        'objective': 'binary:logistic',
        'eval_metric': 'auc',
        'max_depth': 6,
        'learning_rate': 0.1,
        'n_estimators': 100,
        'scale_pos_weight': scale_pos_weight,
        'colsample_bytree': 0.8,
        'subsample': 0.8,
        'random_state': 42
    }
    
    model = xgb.XGBClassifier(**params)
    
    model.fit(
        X_train, y_train,
        eval_set=[(X_val, y_val)],
        early_stopping_rounds=10,
        verbose=True
    )
    
    # Evaluate
    y_pred = model.predict(X_val)
    y_pred_proba = model.predict_proba(X_val)[:, 1]
    
    metrics = {
        'accuracy': accuracy_score(y_val, y_pred),
        'precision': precision_score(y_val, y_pred),
        'recall': recall_score(y_val, y_pred),
        'f1': f1_score(y_val, y_pred),
        'auc': roc_auc_score(y_val, y_pred_proba)
    }
    
    print("Model Performance:")
    for metric, value in metrics.items():
        print(f"{metric}: {value:.4f}")
    
    return model, feature_columns, metrics

def save_model(model, feature_columns, model_dir):
    """Save model artifacts"""
    os.makedirs(model_dir, exist_ok=True)
    
    # Save model
    model_path = os.path.join(model_dir, 'model.pkl')
    joblib.dump(model, model_path)
    
    # Save feature columns
    features_path = os.path.join(model_dir, 'features.json')
    with open(features_path, 'w') as f:
        json.dump(feature_columns, f)
    
    print(f"Model saved to {model_dir}")

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('--train-data', type=str, default=os.environ.get('SM_CHANNEL_TRAIN'))
    parser.add_argument('--model-dir', type=str, default=os.environ.get('SM_MODEL_DIR'))
    
    args = parser.parse_args()
    
    # Load data
    df = load_data(args.train_data)
    
    # Feature engineering
    df = feature_engineering(df)
    
    # Split data
    train_data, val_data = train_test_split(df, test_size=0.2, random_state=42, stratify=df['is_fraud'])
    
    # Train model
    model, feature_columns, metrics = train_model(train_data, val_data)
    
    # Save model
    save_model(model, feature_columns, args.model_dir)
```

### 🔮 Lambda Inference Function

```python
# lambda/ml-inference/index.py
import json
import boto3
import os
from datetime import datetime

sagemaker_runtime = boto3.client('sagemaker-runtime')
dynamodb = boto3.resource('dynamodb')

ENDPOINT_NAME = os.environ['ENDPOINT_NAME']
PREDICTIONS_TABLE = os.environ['PREDICTIONS_TABLE']
FRAUD_THRESHOLD = float(os.environ.get('FRAUD_THRESHOLD', '0.7'))

table = dynamodb.Table(PREDICTIONS_TABLE)

def handler(event, context):
    """Lambda handler for fraud detection inference"""
    try:
        body = json.loads(event['body'])
        transaction = body['transaction']
        
        # Feature engineering
        features = engineer_features(transaction)
        
        # Invoke SageMaker endpoint
        response = sagemaker_runtime.invoke_endpoint(
            EndpointName=ENDPOINT_NAME,
            ContentType='application/json',
            Body=json.dumps(features)
        )
        
        # Parse prediction
        prediction = json.loads(response['Body'].read().decode())
        fraud_score = prediction['score']
        is_fraud = fraud_score > FRAUD_THRESHOLD
        
        # Store prediction
        result = {
            'transactionId': transaction['transactionId'],
            'timestamp': datetime.utcnow().isoformat(),
            'fraudScore': fraud_score,
            'isFraud': is_fraud,
            'threshold': FRAUD_THRESHOLD,
            'features': features,
            'ttl': int(datetime.utcnow().timestamp()) + (30 * 24 * 60 * 60)  # 30 days
        }
        
        table.put_item(Item=result)
        
        return {
            'statusCode': 200,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'transactionId': transaction['transactionId'],
                'fraudScore': round(fraud_score, 4),
                'isFraud': is_fraud,
                'riskLevel': get_risk_level(fraud_score),
                'recommendation': get_recommendation(fraud_score)
            })
        }
        
    except Exception as e:
        print(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }

def engineer_features(transaction):
    """Transform transaction to model features"""
    import math
    from datetime import datetime
    
    amount = transaction['amount']
    timestamp = datetime.fromisoformat(transaction['timestamp'])
    
    features = {
        'amount': amount,
        'amount_log': math.log1p(amount),
        'amount_zscore': (amount - 5000) / 2000,  # From training statistics
        'hour': timestamp.hour,
        'day_of_week': timestamp.weekday(),
        'is_weekend': 1 if timestamp.weekday() >= 5 else 0,
        'is_night': 1 if timestamp.hour >= 22 or timestamp.hour <= 6 else 0,
        'is_foreign': 1 if transaction.get('country', 'UAE') != 'UAE' else 0,
        'txn_count_1h': transaction.get('recentTransactionCount', 1),
        'is_high_risk_merchant': 1 if transaction.get('merchantCategory') in ['GAMBLING', 'ADULT', 'CRYPTO'] else 0
    }
    
    return features

def get_risk_level(fraud_score):
    """Categorize risk level"""
    if fraud_score < 0.3:
        return 'LOW'
    elif fraud_score < 0.7:
        return 'MEDIUM'
    else:
        return 'HIGH'

def get_recommendation(fraud_score):
    """Get action recommendation"""
    if fraud_score < 0.3:
        return 'APPROVE'
    elif fraud_score < 0.7:
        return 'REVIEW'
    else:
        return 'DECLINE'
```

### 🎓 Interview Discussion Points - Q43

**Q1: How to handle ML model versioning?**

**A**:
- **SageMaker Model Registry**: Track model versions, metadata, approval status
- **A/B Testing**: Deploy multiple versions with traffic splitting
- **Canary Deployments**: Gradually roll out new model (10% → 50% → 100%)
- **Shadow Mode**: Run new model alongside production without impacting results
- **Rollback Strategy**: Quick rollback if performance degrades

**Q2: How to monitor ML model performance?**

**A**:
- **Data Capture**: Log inputs/outputs to S3
- **Model Metrics**: Accuracy, precision, recall, F1, AUC
- **Business Metrics**: False positive rate, detection rate
- **Data Drift**: Monitor feature distributions
- **Prediction Latency**: Track inference time (P50, P99)
- **CloudWatch Alarms**: Alert on metric degradation

**Q3: How to handle model retraining?**

**A**:
- **Scheduled Retraining**: Weekly/monthly with new data
- **Trigger-based**: Retrain when performance drops
- **Data Pipeline**: Automate data collection and labeling
- **Feature Store**: Centralized feature management
- **MLOps Pipeline**: Automated training, validation, deployment

**Q4: What are SageMaker inference options?**

**A**:
**Real-time:**
- Hosted endpoints (ml.m5.xlarge)
- Low latency (<100ms)
- Auto-scaling based on traffic

**Batch:**
- Batch Transform jobs
- Process large datasets
- Cost-effective for non-urgent predictions

**Serverless:**
- SageMaker Serverless Inference
- Pay per inference
- Auto-scales to zero

**Q5: How to optimize inference costs?**

**A**:
- **Instance type**: Right-size (ml.t3.medium vs ml.p3.8xlarge)
- **Auto-scaling**: Scale based on invocation rate
- **Batch inference**: Use Batch Transform for bulk predictions
- **Model optimization**: Quantization, pruning, distillation
- **Serverless**: For variable/low traffic
- **Multi-model endpoints**: Host multiple models on one endpoint

---

## Question 44: Model Deployment & Versioning with A/B Testing

### 📋 Question Statement

Implement ML model deployment, versioning, and A/B testing for Emirates NBD using SageMaker endpoints, canary deployments, and automated model rollback strategies.

---

### 🚀 SageMaker Model Deployment CDK

```typescript
// infrastructure/cdk/sagemaker-deployment-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as sagemaker from 'aws-cdk-lib/aws-sagemaker';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as s3 from 'aws-cdk-lib/aws-s3';

export class SageMakerDeploymentStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // S3 bucket for model artifacts
    const modelBucket = new s3.Bucket(this, 'ModelArtifacts', {
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED
    });

    // SageMaker execution role
    const sageMakerRole = new iam.Role(this, 'SageMakerRole', {
      assumedBy: new iam.ServicePrincipal('sagemaker.amazonaws.com'),
      managedPolicies: [
        iam.ManagedPolicy.fromAwsManagedPolicyName('AmazonSageMakerFullAccess')
      ]
    });

    modelBucket.grantRead(sageMakerRole);

    // Model - Version 1 (Production)
    const modelV1 = new sagemaker.CfnModel(this, 'FraudDetectionModelV1', {
      modelName: 'fraud-detection-v1',
      executionRoleArn: sageMakerRole.roleArn,
      primaryContainer: {
        image: `763104351884.dkr.ecr.us-east-1.amazonaws.com/pytorch-inference:2.0.0-cpu-py310`,
        modelDataUrl: `s3://${modelBucket.bucketName}/models/fraud-detection/v1/model.tar.gz`,
        environment: {
          MODEL_VERSION: 'v1',
          INFERENCE_TIMEOUT: '60'
        }
      }
    });

    // Model - Version 2 (Canary)
    const modelV2 = new sagemaker.CfnModel(this, 'FraudDetectionModelV2', {
      modelName: 'fraud-detection-v2',
      executionRoleArn: sageMakerRole.roleArn,
      primaryContainer: {
        image: `763104351884.dkr.ecr.us-east-1.amazonaws.com/pytorch-inference:2.0.0-cpu-py310`,
        modelDataUrl: `s3://${modelBucket.bucketName}/models/fraud-detection/v2/model.tar.gz`,
        environment: {
          MODEL_VERSION: 'v2',
          INFERENCE_TIMEOUT: '60'
        }
      }
    });

    // Endpoint Configuration with Traffic Splitting
    const endpointConfig = new sagemaker.CfnEndpointConfig(this, 'EndpointConfig', {
      endpointConfigName: 'fraud-detection-config',
      productionVariants: [
        {
          variantName: 'ProductionVariant',
          modelName: modelV1.modelName!,
          initialInstanceCount: 2,
          instanceType: 'ml.m5.xlarge',
          initialVariantWeight: 90 // 90% traffic
        },
        {
          variantName: 'CanaryVariant',
          modelName: modelV2.modelName!,
          initialInstanceCount: 1,
          instanceType: 'ml.m5.xlarge',
          initialVariantWeight: 10 // 10% traffic
        }
      ],
      dataCaptureConfig: {
        enableCapture: true,
        initialSamplingPercentage: 100,
        destinationS3Uri: `s3://${modelBucket.bucketName}/data-capture`,
        captureOptions: [
          { captureMode: 'Input' },
          { captureMode: 'Output' }
        ]
      }
    });

    endpointConfig.addDependency(modelV1);
    endpointConfig.addDependency(modelV2);

    // Endpoint
    const endpoint = new sagemaker.CfnEndpoint(this, 'Endpoint', {
      endpointName: 'fraud-detection-endpoint',
      endpointConfigName: endpointConfig.endpointConfigName!
    });

    endpoint.addDependency(endpointConfig);

    new cdk.CfnOutput(this, 'EndpointName', {
      value: endpoint.endpointName!
    });
  }
}
```

### 📊 Model Version Manager

```typescript
// services/ml/model-version-manager.ts
import { SageMaker, S3, DynamoDB } from 'aws-sdk';

export class ModelVersionManager {
  private sagemaker: SageMaker;
  private s3: S3;
  private dynamoDB: DynamoDB.DocumentClient;

  constructor() {
    this.sagemaker = new SageMaker();
    this.s3 = new S3();
    this.dynamoDB = new DynamoDB.DocumentClient();
  }

  async deployNewModel(
    modelName: string,
    version: string,
    s3ModelPath: string,
    strategy: 'CANARY' | 'BLUE_GREEN' | 'ALL_AT_ONCE'
  ): Promise<DeploymentResult> {
    // Register model version
    await this.registerModelVersion({
      modelName,
      version,
      s3Path: s3ModelPath,
      status: 'DEPLOYING',
      timestamp: new Date().toISOString()
    });

    // Create new SageMaker model
    const newModel = await this.sagemaker.createModel({
      ModelName: `${modelName}-${version}`,
      ExecutionRoleArn: process.env.SAGEMAKER_ROLE_ARN!,
      PrimaryContainer: {
        Image: process.env.CONTAINER_IMAGE!,
        ModelDataUrl: s3ModelPath,
        Environment: {
          MODEL_VERSION: version
        }
      }
    }).promise();

    // Deploy based on strategy
    let result: DeploymentResult;

    switch (strategy) {
      case 'CANARY':
        result = await this.canaryDeployment(modelName, version);
        break;
      case 'BLUE_GREEN':
        result = await this.blueGreenDeployment(modelName, version);
        break;
      case 'ALL_AT_ONCE':
        result = await this.allAtOnceDeployment(modelName, version);
        break;
    }

    // Update model version status
    await this.updateModelVersion(modelName, version, 'DEPLOYED');

    return result;
  }

  private async canaryDeployment(modelName: string, newVersion: string): Promise<DeploymentResult> {
    const endpointName = `${modelName}-endpoint`;
    
    // Get current endpoint configuration
    const currentEndpoint = await this.sagemaker.describeEndpoint({
      EndpointName: endpointName
    }).promise();

    const currentConfig = await this.sagemaker.describeEndpointConfig({
      EndpointConfigName: currentEndpoint.EndpointConfigName!
    }).promise();

    const currentModelName = currentConfig.ProductionVariants![0].ModelName!;

    // Create new endpoint config with traffic split
    const newConfig = await this.sagemaker.createEndpointConfig({
      EndpointConfigName: `${modelName}-config-${Date.now()}`,
      ProductionVariants: [
        {
          VariantName: 'Production',
          ModelName: currentModelName,
          InitialInstanceCount: 2,
          InstanceType: 'ml.m5.xlarge',
          InitialVariantWeight: 95 // 95% on current
        },
        {
          VariantName: 'Canary',
          ModelName: `${modelName}-${newVersion}`,
          InitialInstanceCount: 1,
          InstanceType: 'ml.m5.xlarge',
          InitialVariantWeight: 5 // 5% on new (canary)
        }
      ]
    }).promise();

    // Update endpoint
    await this.sagemaker.updateEndpoint({
      EndpointName: endpointName,
      EndpointConfigName: newConfig.EndpointConfigArn!
    }).promise();

    console.log('Canary deployment started: 5% traffic to new model');

    return {
      strategy: 'CANARY',
      endpointName,
      productionVariant: currentModelName,
      canaryVariant: `${modelName}-${newVersion}`,
      trafficSplit: { production: 95, canary: 5 }
    };
  }

  async promoteCanary(modelName: string): Promise<void> {
    const endpointName = `${modelName}-endpoint`;

    // Gradually increase traffic: 5% → 25% → 50% → 100%
    const stages = [
      { production: 95, canary: 5 },
      { production: 75, canary: 25 },
      { production: 50, canary: 50 },
      { production: 0, canary: 100 }
    ];

    for (const stage of stages) {
      await this.updateTrafficDistribution(endpointName, stage);
      console.log(`Traffic updated: ${stage.canary}% on canary`);
      
      // Wait and monitor metrics before next stage
      await this.sleep(5 * 60 * 1000); // 5 minutes
      
      const metrics = await this.getModelMetrics(endpointName, 'Canary');
      
      if (metrics.errorRate > 0.05) { // 5% error threshold
        console.error('High error rate detected, rolling back');
        await this.rollbackDeployment(modelName);
        throw new Error('Deployment rolled back due to high error rate');
      }
    }

    console.log('Canary promoted to production successfully');
  }

  async rollbackDeployment(modelName: string): Promise<void> {
    const endpointName = `${modelName}-endpoint`;

    // Get previous stable endpoint configuration
    const currentEndpoint = await this.sagemaker.describeEndpoint({
      EndpointName: endpointName
    }).promise();

    // Retrieve last stable config from DynamoDB
    const stableConfig = await this.dynamoDB.get({
      TableName: 'ModelDeployments',
      Key: { modelName, status: 'STABLE' }
    }).promise();

    if (!stableConfig.Item) {
      throw new Error('No stable configuration found for rollback');
    }

    // Rollback to stable configuration
    await this.sagemaker.updateEndpoint({
      EndpointName: endpointName,
      EndpointConfigName: stableConfig.Item.endpointConfigName
    }).promise();

    console.log('Rollback complete');
  }

  private async updateTrafficDistribution(
    endpointName: string,
    weights: { production: number; canary: number }
  ): Promise<void> {
    await this.sagemaker.updateEndpointWeightsAndCapacities({
      EndpointName: endpointName,
      DesiredWeightsAndCapacities: [
        { VariantName: 'Production', DesiredWeight: weights.production },
        { VariantName: 'Canary', DesiredWeight: weights.canary }
      ]
    }).promise();
  }

  private async getModelMetrics(
    endpointName: string,
    variantName: string
  ): Promise<ModelMetrics> {
    // Query CloudWatch for model metrics
    return {
      errorRate: 0.01,
      latencyP99: 150,
      invocations: 1000
    };
  }

  private async registerModelVersion(version: ModelVersion): Promise<void> {
    await this.dynamoDB.put({
      TableName: 'ModelVersions',
      Item: version
    }).promise();
  }

  private async updateModelVersion(
    modelName: string,
    version: string,
    status: string
  ): Promise<void> {
    await this.dynamoDB.update({
      TableName: 'ModelVersions',
      Key: { modelName, version },
      UpdateExpression: 'SET #status = :status, updatedAt = :timestamp',
      ExpressionAttributeNames: { '#status': 'status' },
      ExpressionAttributeValues: {
        ':status': status,
        ':timestamp': new Date().toISOString()
      }
    }).promise();
  }

  private async blueGreenDeployment(modelName: string, newVersion: string): Promise<DeploymentResult> {
    // Implementation similar to canary but immediate 100% switch
    return {
      strategy: 'BLUE_GREEN',
      endpointName: `${modelName}-endpoint`,
      productionVariant: `${modelName}-${newVersion}`,
      trafficSplit: { production: 100 }
    };
  }

  private async allAtOnceDeployment(modelName: string, newVersion: string): Promise<DeploymentResult> {
    // Replace endpoint entirely
    return {
      strategy: 'ALL_AT_ONCE',
      endpointName: `${modelName}-endpoint`,
      productionVariant: `${modelName}-${newVersion}`,
      trafficSplit: { production: 100 }
    };
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

interface ModelVersion {
  modelName: string;
  version: string;
  s3Path: string;
  status: string;
  timestamp: string;
}

interface DeploymentResult {
  strategy: string;
  endpointName: string;
  productionVariant: string;
  canaryVariant?: string;
  trafficSplit: Record<string, number>;
}

interface ModelMetrics {
  errorRate: number;
  latencyP99: number;
  invocations: number;
}
```

### 🎓 Interview Discussion Points - Q44

**Q1: What is model versioning in ML?**

**A**:
- **Track versions**: v1, v2, v3 with metadata
- **Reproducibility**: Can redeploy exact same model
- **Rollback**: Revert to previous version if issues
- **A/B testing**: Compare performance of versions
- **Storage**: S3 with versioning enabled

**Q2: What deployment strategies exist for ML models?**

**A**:
- **Canary**: 5% → 25% → 50% → 100% gradual rollout
- **Blue/Green**: Switch 100% traffic instantly, keep old for rollback
- **Shadow**: New model processes requests but doesn't serve results (validation)
- **A/B**: Split traffic 50/50 for experimentation

**Q3: How to detect model degradation?**

**A**:
- **Accuracy drop**: Monitor prediction accuracy
- **Data drift**: Input distribution changes
- **Concept drift**: Relationship between features and target changes
- **Latency increase**: Model slower than expected
- **Error rate**: Increased 4xx/5xx errors

**Q4: What is SageMaker Model Monitor?**

**A**:
- **Data quality**: Detects schema changes, missing values
- **Model quality**: Monitors accuracy, precision, recall
- **Bias drift**: Detects unfair predictions
- **Feature attribution**: Explains predictions
- **Alerts**: CloudWatch alarms on violations

**Q5: How to implement auto-rollback?**

**A**:
```typescript
if (canaryErrorRate > productionErrorRate * 1.5) {
  await rollbackDeployment();
}
```
**Conditions**: Error rate, latency P99, prediction distribution shift

---

**End of File 22**

