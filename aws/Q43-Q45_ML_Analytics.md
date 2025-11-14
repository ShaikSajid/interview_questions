# AWS Interview Questions: Machine Learning & Analytics (Q43-Q45)

## Question 43: Explain Amazon SageMaker for machine learning

### 📋 Answer

**Amazon SageMaker** is a fully managed service that provides tools to build, train, and deploy machine learning models at scale.

### SageMaker Architecture:

```
Amazon SageMaker
├── Build
│   ├── Studio (IDE)
│   ├── Notebooks
│   ├── Data Wrangler
│   ├── Feature Store
│   └── Ground Truth (labeling)
├── Train
│   ├── Built-in algorithms
│   ├── Custom algorithms
│   ├── Distributed training
│   ├── Hyperparameter tuning
│   └── Training jobs
├── Deploy
│   ├── Real-time endpoints
│   ├── Batch transform
│   ├── Serverless inference
│   └── Multi-model endpoints
├── Monitor
│   ├── Model Monitor
│   ├── Clarify (bias detection)
│   ├── Experiments
│   └── Model Registry
└── MLOps
    ├── Pipelines
    ├── Projects
    ├── Model versioning
    └── CI/CD integration
```

### Complete SageMaker Implementation:

```javascript
// sagemaker-service.js
import {
  SageMakerClient,
  CreateTrainingJobCommand,
  DescribeTrainingJobCommand,
  CreateModelCommand,
  CreateEndpointConfigCommand,
  CreateEndpointCommand,
  DescribeEndpointCommand,
  InvokeEndpointCommand,
  DeleteEndpointCommand,
  CreateTransformJobCommand,
  DescribeTransformJobCommand,
  CreateHyperParameterTuningJobCommand
} from '@aws-sdk/client-sagemaker';

import {
  SageMakerRuntimeClient,
  InvokeEndpointCommand as RuntimeInvokeEndpointCommand
} from '@aws-sdk/client-sagemaker-runtime';

const sagemakerClient = new SageMakerClient({ region: 'us-east-1' });
const runtimeClient = new SageMakerRuntimeClient({ region: 'us-east-1' });

class SageMakerService {
  // 1. Create Training Job
  async createTrainingJob(config) {
    const command = new CreateTrainingJobCommand({
      TrainingJobName: config.jobName,
      RoleArn: config.roleArn,
      AlgorithmSpecification: {
        TrainingImage: config.trainingImage,
        TrainingInputMode: 'File',  // File or Pipe
        MetricDefinitions: config.metrics || []
      },
      InputDataConfig: [
        {
          ChannelName: 'training',
          DataSource: {
            S3DataSource: {
              S3DataType: 'S3Prefix',
              S3Uri: config.trainingData,
              S3DataDistributionType: 'FullyReplicated'
            }
          },
          ContentType: 'text/csv',
          CompressionType: 'None'
        },
        {
          ChannelName: 'validation',
          DataSource: {
            S3DataSource: {
              S3DataType: 'S3Prefix',
              S3Uri: config.validationData,
              S3DataDistributionType: 'FullyReplicated'
            }
          },
          ContentType: 'text/csv'
        }
      ],
      OutputDataConfig: {
        S3OutputPath: config.outputPath
      },
      ResourceConfig: {
        InstanceType: config.instanceType || 'ml.m5.xlarge',
        InstanceCount: config.instanceCount || 1,
        VolumeSizeInGB: config.volumeSize || 30
      },
      HyperParameters: config.hyperParameters || {},
      StoppingCondition: {
        MaxRuntimeInSeconds: config.maxRuntime || 86400  // 24 hours
      },
      EnableNetworkIsolation: false,
      EnableInterContainerTrafficEncryption: true,
      EnableManagedSpotTraining: config.useSpotInstances || false
    });
    
    try {
      const response = await sagemakerClient.send(command);
      console.log('Training job created:', response.TrainingJobArn);
      return response.TrainingJobArn;
    } catch (error) {
      console.error('Failed to create training job:', error);
      throw error;
    }
  }
  
  // 2. Wait for Training Job
  async waitForTrainingJob(jobName, pollInterval = 30000) {
    while (true) {
      const command = new DescribeTrainingJobCommand({
        TrainingJobName: jobName
      });
      
      const response = await sagemakerClient.send(command);
      const status = response.TrainingJobStatus;
      
      console.log('Training job status:', status);
      
      if (status === 'Completed') {
        return {
          modelArtifacts: response.ModelArtifacts.S3ModelArtifacts,
          trainingTime: response.TrainingTimeInSeconds,
          billableTime: response.BillableTimeInSeconds,
          finalMetrics: response.FinalMetricDataList
        };
      } else if (['Failed', 'Stopped'].includes(status)) {
        throw new Error(`Training ${status}: ${response.FailureReason}`);
      }
      
      await new Promise(resolve => setTimeout(resolve, pollInterval));
    }
  }
  
  // 3. Create Model
  async createModel(modelName, executionRoleArn, modelDataUrl, containerImage) {
    const command = new CreateModelCommand({
      ModelName: modelName,
      ExecutionRoleArn: executionRoleArn,
      PrimaryContainer: {
        Image: containerImage,
        ModelDataUrl: modelDataUrl,
        Environment: {}
      },
      EnableNetworkIsolation: false
    });
    
    const response = await sagemakerClient.send(command);
    console.log('Model created:', response.ModelArn);
    return response.ModelArn;
  }
  
  // 4. Create Endpoint Configuration
  async createEndpointConfig(configName, modelName, instanceType, initialInstanceCount) {
    const command = new CreateEndpointConfigCommand({
      EndpointConfigName: configName,
      ProductionVariants: [
        {
          VariantName: 'AllTraffic',
          ModelName: modelName,
          InstanceType: instanceType || 'ml.m5.large',
          InitialInstanceCount: initialInstanceCount || 1,
          InitialVariantWeight: 1
        }
      ],
      DataCaptureConfig: {
        EnableCapture: true,
        InitialSamplingPercentage: 100,
        DestinationS3Uri: 's3://my-bucket/model-monitor/data-capture/',
        CaptureOptions: [
          { CaptureMode: 'Input' },
          { CaptureMode: 'Output' }
        ]
      }
    });
    
    const response = await sagemakerClient.send(command);
    console.log('Endpoint config created:', response.EndpointConfigArn);
    return response.EndpointConfigArn;
  }
  
  // 5. Create Endpoint
  async createEndpoint(endpointName, configName) {
    const command = new CreateEndpointCommand({
      EndpointName: endpointName,
      EndpointConfigName: configName
    });
    
    const response = await sagemakerClient.send(command);
    console.log('Endpoint creation started:', response.EndpointArn);
    return response.EndpointArn;
  }
  
  // 6. Wait for Endpoint
  async waitForEndpoint(endpointName, pollInterval = 30000) {
    while (true) {
      const command = new DescribeEndpointCommand({
        EndpointName: endpointName
      });
      
      const response = await sagemakerClient.send(command);
      const status = response.EndpointStatus;
      
      console.log('Endpoint status:', status);
      
      if (status === 'InService') {
        return {
          endpointArn: response.EndpointArn,
          endpointUrl: response.EndpointArn
        };
      } else if (status === 'Failed') {
        throw new Error(`Endpoint failed: ${response.FailureReason}`);
      }
      
      await new Promise(resolve => setTimeout(resolve, pollInterval));
    }
  }
  
  // 7. Invoke Endpoint (Real-time Inference)
  async invokeEndpoint(endpointName, data, contentType = 'text/csv') {
    const command = new RuntimeInvokeEndpointCommand({
      EndpointName: endpointName,
      Body: data,
      ContentType: contentType,
      Accept: 'application/json'
    });
    
    try {
      const response = await runtimeClient.send(command);
      const result = Buffer.from(response.Body).toString('utf-8');
      return JSON.parse(result);
    } catch (error) {
      console.error('Inference failed:', error);
      throw error;
    }
  }
  
  // 8. Create Batch Transform Job
  async createBatchTransform(config) {
    const command = new CreateTransformJobCommand({
      TransformJobName: config.jobName,
      ModelName: config.modelName,
      TransformInput: {
        DataSource: {
          S3DataSource: {
            S3DataType: 'S3Prefix',
            S3Uri: config.inputData
          }
        },
        ContentType: 'text/csv',
        SplitType: 'Line'
      },
      TransformOutput: {
        S3OutputPath: config.outputPath,
        Accept: 'text/csv',
        AssembleWith: 'Line'
      },
      TransformResources: {
        InstanceType: config.instanceType || 'ml.m5.xlarge',
        InstanceCount: config.instanceCount || 1
      },
      BatchStrategy: 'MultiRecord',
      MaxPayloadInMB: 6,
      MaxConcurrentTransforms: config.maxConcurrent || 1
    });
    
    const response = await sagemakerClient.send(command);
    console.log('Batch transform job started:', response.TransformJobArn);
    return response.TransformJobArn;
  }
  
  // 9. Hyperparameter Tuning
  async createHyperParameterTuningJob(config) {
    const command = new CreateHyperParameterTuningJobCommand({
      HyperParameterTuningJobName: config.jobName,
      HyperParameterTuningJobConfig: {
        Strategy: 'Bayesian',  // Bayesian or Random
        HyperParameterTuningJobObjective: {
          Type: 'Maximize',
          MetricName: 'validation:accuracy'
        },
        ResourceLimits: {
          MaxNumberOfTrainingJobs: config.maxTrainingJobs || 20,
          MaxParallelTrainingJobs: config.maxParallelJobs || 2
        },
        ParameterRanges: {
          ContinuousParameterRanges: [
            {
              Name: 'learning_rate',
              MinValue: '0.001',
              MaxValue: '0.1',
              ScalingType: 'Logarithmic'
            }
          ],
          IntegerParameterRanges: [
            {
              Name: 'num_layers',
              MinValue: '2',
              MaxValue: '10',
              ScalingType: 'Linear'
            }
          ],
          CategoricalParameterRanges: [
            {
              Name: 'optimizer',
              Values: ['adam', 'sgd', 'rmsprop']
            }
          ]
        }
      },
      TrainingJobDefinition: {
        StaticHyperParameters: config.staticHyperParameters || {},
        AlgorithmSpecification: {
          TrainingImage: config.trainingImage,
          TrainingInputMode: 'File',
          MetricDefinitions: [
            {
              Name: 'validation:accuracy',
              Regex: 'validation-accuracy: ([0-9.]+)'
            }
          ]
        },
        RoleArn: config.roleArn,
        InputDataConfig: config.inputDataConfig,
        OutputDataConfig: {
          S3OutputPath: config.outputPath
        },
        ResourceConfig: {
          InstanceType: 'ml.m5.xlarge',
          InstanceCount: 1,
          VolumeSizeInGB: 30
        },
        StoppingCondition: {
          MaxRuntimeInSeconds: 3600
        }
      }
    });
    
    const response = await sagemakerClient.send(command);
    console.log('Hyperparameter tuning job started:', response.HyperParameterTuningJobArn);
    return response.HyperParameterTuningJobArn;
  }
  
  // 10. Delete Endpoint
  async deleteEndpoint(endpointName) {
    const command = new DeleteEndpointCommand({
      EndpointName: endpointName
    });
    
    await sagemakerClient.send(command);
    console.log('Endpoint deleted:', endpointName);
  }
}

// ML Pipeline Orchestrator
class MLPipelineOrchestrator {
  constructor(sagemakerService) {
    this.sagemaker = sagemakerService;
  }
  
  // Complete ML Pipeline: Train -> Deploy -> Inference
  async runMLPipeline(config) {
    console.log('Starting ML Pipeline...\n');
    
    // Step 1: Train Model
    console.log('Step 1: Training model...');
    const trainingJobName = `${config.modelName}-training-${Date.now()}`;
    
    await this.sagemaker.createTrainingJob({
      jobName: trainingJobName,
      roleArn: config.roleArn,
      trainingImage: config.trainingImage,
      trainingData: config.trainingData,
      validationData: config.validationData,
      outputPath: config.outputPath,
      hyperParameters: config.hyperParameters,
      instanceType: 'ml.m5.xlarge',
      useSpotInstances: true
    });
    
    const trainingResult = await this.sagemaker.waitForTrainingJob(trainingJobName);
    console.log('Training completed:', trainingResult);
    
    // Step 2: Create Model
    console.log('\nStep 2: Creating model...');
    const modelName = `${config.modelName}-${Date.now()}`;
    
    await this.sagemaker.createModel(
      modelName,
      config.roleArn,
      trainingResult.modelArtifacts,
      config.inferenceImage
    );
    
    // Step 3: Create Endpoint Config
    console.log('Step 3: Creating endpoint configuration...');
    const configName = `${config.modelName}-config-${Date.now()}`;
    
    await this.sagemaker.createEndpointConfig(
      configName,
      modelName,
      'ml.m5.large',
      1
    );
    
    // Step 4: Deploy Endpoint
    console.log('Step 4: Deploying endpoint...');
    const endpointName = `${config.modelName}-endpoint`;
    
    await this.sagemaker.createEndpoint(endpointName, configName);
    await this.sagemaker.waitForEndpoint(endpointName);
    
    console.log('Endpoint deployed successfully!');
    
    return {
      endpointName,
      modelName,
      trainingMetrics: trainingResult.finalMetrics
    };
  }
}

// Usage - Complete ML Workflow
const sagemaker = new SageMakerService();
const mlPipeline = new MLPipelineOrchestrator(sagemaker);

// Example: Customer Churn Prediction Model
const pipelineConfig = {
  modelName: 'customer-churn-model',
  roleArn: 'arn:aws:iam::123456789012:role/SageMakerRole',
  trainingImage: '683313688378.dkr.ecr.us-east-1.amazonaws.com/sagemaker-xgboost:1.5-1',
  inferenceImage: '683313688378.dkr.ecr.us-east-1.amazonaws.com/sagemaker-xgboost:1.5-1',
  trainingData: 's3://my-ml-bucket/data/training/',
  validationData: 's3://my-ml-bucket/data/validation/',
  outputPath: 's3://my-ml-bucket/models/',
  hyperParameters: {
    objective: 'binary:logistic',
    num_round: '100',
    max_depth: '5',
    eta: '0.2',
    subsample: '0.8',
    colsample_bytree: '0.8'
  }
};

// Run complete pipeline
const pipeline = await mlPipeline.runMLPipeline(pipelineConfig);

// Real-time inference
const prediction = await sagemaker.invokeEndpoint(
  pipeline.endpointName,
  '45,120000,3,2,yes,no,yes,0.8',  // Customer features
  'text/csv'
);

console.log('Churn prediction:', prediction);

// Batch inference for large dataset
const batchJobName = `batch-inference-${Date.now()}`;
await sagemaker.createBatchTransform({
  jobName: batchJobName,
  modelName: pipeline.modelName,
  inputData: 's3://my-ml-bucket/data/batch-input/',
  outputPath: 's3://my-ml-bucket/data/batch-output/',
  instanceType: 'ml.m5.xlarge',
  instanceCount: 2
});

// Hyperparameter tuning for optimization
const tuningJobName = `hpo-${Date.now()}`;
await sagemaker.createHyperParameterTuningJob({
  jobName: tuningJobName,
  roleArn: pipelineConfig.roleArn,
  trainingImage: pipelineConfig.trainingImage,
  outputPath: pipelineConfig.outputPath,
  inputDataConfig: [
    {
      ChannelName: 'training',
      DataSource: {
        S3DataSource: {
          S3DataType: 'S3Prefix',
          S3Uri: pipelineConfig.trainingData
        }
      }
    }
  ],
  staticHyperParameters: {
    objective: 'binary:logistic',
    num_round: '100'
  },
  maxTrainingJobs: 20,
  maxParallelJobs: 2
});
```

### SageMaker Built-in Algorithms:

```javascript
// Common built-in algorithms and their use cases
const SAGEMAKER_ALGORITHMS = {
  // Supervised Learning
  XGBoost: {
    image: '683313688378.dkr.ecr.us-east-1.amazonaws.com/sagemaker-xgboost:1.5-1',
    useCase: 'Classification and regression (tabular data)',
    inputFormat: 'CSV, LibSVM, Parquet'
  },
  LinearLearner: {
    image: '382416733822.dkr.ecr.us-east-1.amazonaws.com/linear-learner:1',
    useCase: 'Linear regression and classification',
    inputFormat: 'RecordIO-protobuf, CSV'
  },
  
  // Deep Learning
  ImageClassification: {
    image: '811284229777.dkr.ecr.us-east-1.amazonaws.com/image-classification:1',
    useCase: 'Image classification with transfer learning',
    inputFormat: 'RecordIO, Image files'
  },
  ObjectDetection: {
    image: '811284229777.dkr.ecr.us-east-1.amazonaws.com/object-detection:1',
    useCase: 'Detect objects in images',
    inputFormat: 'RecordIO, JSON'
  },
  
  // NLP
  BlazingText: {
    image: '811284229777.dkr.ecr.us-east-1.amazonaws.com/blazingtext:1',
    useCase: 'Text classification and word embeddings',
    inputFormat: 'Text file'
  },
  
  // Unsupervised
  KMeans: {
    image: '382416733822.dkr.ecr.us-east-1.amazonaws.com/kmeans:1',
    useCase: 'Clustering',
    inputFormat: 'RecordIO-protobuf, CSV'
  },
  PCA: {
    image: '382416733822.dkr.ecr.us-east-1.amazonaws.com/pca:1',
    useCase: 'Dimensionality reduction',
    inputFormat: 'RecordIO-protobuf, CSV'
  },
  
  // Forecasting
  DeepAR: {
    image: '522234722520.dkr.ecr.us-east-1.amazonaws.com/forecasting-deepar:1',
    useCase: 'Time series forecasting',
    inputFormat: 'JSON Lines'
  }
};
```

### Best Practices:

1. ✅ Use managed spot training for cost savings
2. ✅ Enable data capture for model monitoring
3. ✅ Use SageMaker Experiments for tracking
4. ✅ Implement A/B testing with production variants
5. ✅ Use Feature Store for feature reuse
6. ✅ Enable auto-scaling for endpoints
7. ✅ Use batch transform for large-scale inference
8. ✅ Implement CI/CD with SageMaker Pipelines
9. ✅ Monitor model drift with Model Monitor
10. ✅ Use SageMaker Clarify for bias detection

---

## Question 44: Explain Amazon Athena for serverless SQL queries

### 📋 Answer

**Amazon Athena** is an interactive query service that makes it easy to analyze data in Amazon S3 using standard SQL.

### Athena Architecture:

```
Amazon Athena
├── Query Engine
│   ├── Presto-based
│   ├── ANSI SQL support
│   └── Federated queries
├── Data Sources
│   ├── S3 (primary)
│   ├── Data Catalog
│   ├── External databases
│   └── Custom connectors
├── Features
│   ├── Serverless
│   ├── Pay per query
│   ├── Partitioning support
│   └── Result caching
├── Optimization
│   ├── Columnar formats (Parquet, ORC)
│   ├── Compression (GZIP, Snappy)
│   ├── Partitioning
│   └── File size optimization
└── Integration
    ├── QuickSight
    ├── Glue Data Catalog
    ├── Lake Formation
    └── JDBC/ODBC
```

### Complete Athena Implementation:

```javascript
// athena-service.js
import {
  AthenaClient,
  StartQueryExecutionCommand,
  GetQueryExecutionCommand,
  GetQueryResultsCommand,
  CreateNamedQueryCommand,
  ListNamedQueriesCommand,
  CreateWorkGroupCommand,
  UpdateWorkGroupCommand
} from '@aws-sdk/client-athena';

const athenaClient = new AthenaClient({ region: 'us-east-1' });

class AthenaService {
  constructor(outputLocation, database = 'default') {
    this.outputLocation = outputLocation;
    this.database = database;
  }
  
  // 1. Execute Query
  async executeQuery(queryString, workGroup = 'primary') {
    const command = new StartQueryExecutionCommand({
      QueryString: queryString,
      QueryExecutionContext: {
        Database: this.database,
        Catalog: 'AwsDataCatalog'
      },
      ResultConfiguration: {
        OutputLocation: this.outputLocation,
        EncryptionConfiguration: {
          EncryptionOption: 'SSE_S3'
        }
      },
      WorkGroup: workGroup
    });
    
    try {
      const response = await athenaClient.send(command);
      console.log('Query started:', response.QueryExecutionId);
      return response.QueryExecutionId;
    } catch (error) {
      console.error('Query execution failed:', error);
      throw error;
    }
  }
  
  // 2. Wait for Query Completion
  async waitForQuery(queryExecutionId, pollInterval = 2000) {
    while (true) {
      const command = new GetQueryExecutionCommand({
        QueryExecutionId: queryExecutionId
      });
      
      const response = await athenaClient.send(command);
      const state = response.QueryExecution.Status.State;
      
      console.log('Query state:', state);
      
      if (state === 'SUCCEEDED') {
        return {
          dataScannedInBytes: response.QueryExecution.Statistics.DataScannedInBytes,
          executionTimeInMillis: response.QueryExecution.Statistics.EngineExecutionTimeInMillis,
          outputLocation: response.QueryExecution.ResultConfiguration.OutputLocation
        };
      } else if (['FAILED', 'CANCELLED'].includes(state)) {
        throw new Error(`Query ${state}: ${response.QueryExecution.Status.StateChangeReason}`);
      }
      
      await new Promise(resolve => setTimeout(resolve, pollInterval));
    }
  }
  
  // 3. Get Query Results
  async getQueryResults(queryExecutionId, maxResults = 1000) {
    const command = new GetQueryResultsCommand({
      QueryExecutionId: queryExecutionId,
      MaxResults: maxResults
    });
    
    const response = await athenaClient.send(command);
    
    // Parse results
    const columns = response.ResultSet.ResultSetMetadata.ColumnInfo.map(col => ({
      name: col.Name,
      type: col.Type
    }));
    
    const rows = response.ResultSet.Rows.slice(1).map(row => {
      const rowData = {};
      row.Data.forEach((cell, index) => {
        rowData[columns[index].name] = cell.VarCharValue;
      });
      return rowData;
    });
    
    return {
      columns,
      rows,
      nextToken: response.NextToken
    };
  }
  
  // 4. Execute Query and Get Results
  async query(queryString, workGroup = 'primary') {
    const queryId = await this.executeQuery(queryString, workGroup);
    const stats = await this.waitForQuery(queryId);
    const results = await this.getQueryResults(queryId);
    
    return {
      ...results,
      statistics: stats
    };
  }
  
  // 5. Create Named Query (Saved Query)
  async createNamedQuery(name, description, queryString) {
    const command = new CreateNamedQueryCommand({
      Name: name,
      Description: description,
      Database: this.database,
      QueryString: queryString
    });
    
    const response = await athenaClient.send(command);
    console.log('Named query created:', response.NamedQueryId);
    return response.NamedQueryId;
  }
  
  // 6. Create Work Group
  async createWorkGroup(name, config) {
    const command = new CreateWorkGroupCommand({
      Name: name,
      Configuration: {
        ResultConfigurationUpdates: {
          OutputLocation: config.outputLocation,
          EncryptionConfiguration: {
            EncryptionOption: 'SSE_S3'
          }
        },
        EnforceWorkGroupConfiguration: true,
        PublishCloudWatchMetricsEnabled: true,
        BytesScannedCutoffPerQuery: config.bytesScannedLimit || 100000000000,  // 100 GB
        RequesterPaysEnabled: false
      },
      Description: config.description,
      Tags: config.tags || []
    });
    
    await athenaClient.send(command);
    console.log('Work group created:', name);
  }
}

// Athena Query Builder
class AthenaQueryBuilder {
  // Create external table
  static createExternalTable(tableName, location, columns, partitions = []) {
    const columnDefs = columns.map(col => `${col.name} ${col.type}`).join(',\n  ');
    
    const partitionDef = partitions.length > 0
      ? `PARTITIONED BY (${partitions.map(p => `${p.name} ${p.type}`).join(', ')})`
      : '';
    
    return `
CREATE EXTERNAL TABLE IF NOT EXISTS ${tableName} (
  ${columnDefs}
)
${partitionDef}
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
WITH SERDEPROPERTIES (
  'serialization.format' = ',',
  'field.delim' = ','
)
LOCATION '${location}'
TBLPROPERTIES ('has_encrypted_data'='false');
    `.trim();
  }
  
  // Create Parquet table
  static createParquetTable(tableName, location, columns, partitions = []) {
    const columnDefs = columns.map(col => `${col.name} ${col.type}`).join(',\n  ');
    
    const partitionDef = partitions.length > 0
      ? `PARTITIONED BY (${partitions.map(p => `${p.name} ${p.type}`).join(', ')})`
      : '';
    
    return `
CREATE EXTERNAL TABLE IF NOT EXISTS ${tableName} (
  ${columnDefs}
)
${partitionDef}
STORED AS PARQUET
LOCATION '${location}'
TBLPROPERTIES (
  'parquet.compression'='SNAPPY',
  'projection.enabled'='true'
);
    `.trim();
  }
  
  // Add partition
  static addPartition(tableName, partition, location) {
    const partitionSpec = Object.entries(partition)
      .map(([key, value]) => `${key}='${value}'`)
      .join(', ');
    
    return `
ALTER TABLE ${tableName}
ADD IF NOT EXISTS PARTITION (${partitionSpec})
LOCATION '${location}';
    `.trim();
  }
  
  // Repair partitions (auto-discover)
  static repairPartitions(tableName) {
    return `MSCK REPAIR TABLE ${tableName};`;
  }
  
  // Create CTAS (Create Table As Select)
  static createTableAsSelect(newTable, selectQuery, format = 'PARQUET', partitions = []) {
    const partitionDef = partitions.length > 0
      ? `PARTITIONED BY (${partitions.join(', ')})`
      : '';
    
    return `
CREATE TABLE ${newTable}
WITH (
  format = '${format}',
  external_location = 's3://my-bucket/${newTable}/',
  ${partitionDef ? `partitioned_by = ARRAY[${partitions.map(p => `'${p}'`).join(', ')}],` : ''}
  bucketed_by = ARRAY['id'],
  bucket_count = 10
) AS
${selectQuery};
    `.trim();
  }
  
  // Optimize query with partition pruning
  static optimizedSelect(tableName, columns, whereClause, partitionColumn, partitionValue) {
    return `
SELECT ${columns.join(', ')}
FROM ${tableName}
WHERE ${partitionColumn} = '${partitionValue}'
  AND ${whereClause}
LIMIT 10000;
    `.trim();
  }
}

// Usage - Analytics Queries
const athena = new AthenaService(
  's3://my-athena-results/',
  'analytics_db'
);

// Create table for S3 logs
const createLogsTable = AthenaQueryBuilder.createParquetTable(
  'application_logs',
  's3://my-logs-bucket/applications/logs/',
  [
    { name: 'timestamp', type: 'timestamp' },
    { name: 'level', type: 'string' },
    { name: 'message', type: 'string' },
    { name: 'user_id', type: 'string' },
    { name: 'request_id', type: 'string' },
    { name: 'duration_ms', type: 'int' }
  ],
  [
    { name: 'year', type: 'int' },
    { name: 'month', type: 'int' },
    { name: 'day', type: 'int' }
  ]
);

await athena.query(createLogsTable);

// Repair partitions to auto-discover
await athena.query(AthenaQueryBuilder.repairPartitions('application_logs'));

// Query 1: Error analysis
const errorAnalysis = await athena.query(`
  SELECT 
    DATE_TRUNC('hour', timestamp) as hour,
    level,
    COUNT(*) as error_count,
    COUNT(DISTINCT user_id) as affected_users
  FROM application_logs
  WHERE year = 2024
    AND month = 1
    AND day = 15
    AND level IN ('ERROR', 'FATAL')
  GROUP BY DATE_TRUNC('hour', timestamp), level
  ORDER BY hour DESC, error_count DESC
`);

console.log('Error Analysis:', errorAnalysis.rows);
console.log('Data Scanned:', errorAnalysis.statistics.dataScannedInBytes / 1024 / 1024, 'MB');

// Query 2: Performance analysis
const performanceAnalysis = await athena.query(`
  SELECT 
    user_id,
    COUNT(*) as request_count,
    AVG(duration_ms) as avg_duration,
    PERCENTILE(duration_ms, 0.95) as p95_duration,
    MAX(duration_ms) as max_duration
  FROM application_logs
  WHERE year = 2024
    AND month = 1
    AND duration_ms IS NOT NULL
  GROUP BY user_id
  HAVING COUNT(*) > 100
  ORDER BY p95_duration DESC
  LIMIT 100
`);

console.log('Top Slow Users:', performanceAnalysis.rows);

// Query 3: Create aggregated table (CTAS)
const createAggregatedTable = AthenaQueryBuilder.createTableAsSelect(
  'daily_log_summary',
  `
    SELECT 
      DATE(timestamp) as log_date,
      level,
      COUNT(*) as log_count,
      COUNT(DISTINCT user_id) as unique_users,
      AVG(duration_ms) as avg_duration
    FROM application_logs
    WHERE year = 2024 AND month = 1
    GROUP BY DATE(timestamp), level
  `,
  'PARQUET',
  ['log_date']
);

await athena.query(createAggregatedTable);

// Create named query for reuse
await athena.createNamedQuery(
  'daily-error-report',
  'Daily error report with user impact',
  `
    SELECT 
      log_date,
      SUM(CASE WHEN level = 'ERROR' THEN log_count ELSE 0 END) as errors,
      SUM(CASE WHEN level = 'FATAL' THEN log_count ELSE 0 END) as fatals,
      SUM(unique_users) as total_active_users
    FROM daily_log_summary
    WHERE log_date >= CURRENT_DATE - INTERVAL '7' DAY
    GROUP BY log_date
    ORDER BY log_date DESC
  `
);

// Create work group with cost controls
await athena.createWorkGroup('analytics-team', {
  outputLocation: 's3://analytics-athena-results/',
  bytesScannedLimit: 100000000000,  // 100 GB per query limit
  description: 'Work group for analytics team with cost controls',
  tags: [
    { Key: 'Team', Value: 'Analytics' },
    { Key: 'CostCenter', Value: 'DataEngineering' }
  ]
});
```

### Athena Optimization Techniques:

```javascript
// Optimization strategies
const ATHENA_OPTIMIZATIONS = {
  // 1. Use columnar formats
  parquetFormat: {
    benefit: '80-90% reduction in data scanned',
    implementation: 'STORED AS PARQUET',
    compression: 'SNAPPY or GZIP'
  },
  
  // 2. Partition data
  partitioning: {
    benefit: 'Scan only relevant partitions',
    strategy: 'By date (year/month/day) or other frequent filters',
    example: 'PARTITIONED BY (year INT, month INT, day INT)'
  },
  
  // 3. Optimize file sizes
  fileSizing: {
    optimal: '128 MB - 1 GB per file',
    avoid: 'Many small files (< 128 MB) or very large files (> 1 GB)',
    solution: 'Use CTAS or AWS Glue to compact files'
  },
  
  // 4. Use compression
  compression: {
    formats: ['GZIP', 'SNAPPY', 'ZSTD', 'LZ4'],
    tradeoff: 'SNAPPY = faster, GZIP = smaller',
    recommendation: 'SNAPPY for Parquet, GZIP for text'
  },
  
  // 5. Column pruning
  columnPruning: {
    do: 'SELECT col1, col2 FROM table',
    dont: 'SELECT * FROM table',
    benefit: 'Scan only needed columns (works best with Parquet/ORC)'
  },
  
  // 6. Predicate pushdown
  predicatePushdown: {
    benefit: 'Filter data before reading',
    example: 'WHERE year=2024 AND month=1',
    requirement: 'Works best with partitioned data'
  }
};
```

### Best Practices:

1. ✅ Use Parquet or ORC format for best performance
2. ✅ Partition data by date or frequently filtered columns
3. ✅ Compress data with SNAPPY or GZIP
4. ✅ Keep file sizes between 128 MB and 1 GB
5. ✅ Use CTAS to optimize existing tables
6. ✅ Enable result caching for repeated queries
7. ✅ Use workgroups for cost control
8. ✅ Monitor query costs with CloudWatch
9. ✅ Use EXPLAIN to analyze query plans
10. ✅ Leverage AWS Glue Data Catalog

---

## Question 45: Explain Amazon QuickSight for business intelligence

### 📋 Answer

**Amazon QuickSight** is a fast, cloud-powered business intelligence service that makes it easy to deliver insights to everyone in your organization.

### QuickSight Architecture:

```
Amazon QuickSight
├── Data Sources
│   ├── AWS services (S3, Athena, RDS, Redshift)
│   ├── Databases (MySQL, PostgreSQL, SQL Server)
│   ├── SaaS applications (Salesforce, ServiceNow)
│   └── Files (CSV, Excel, JSON)
├── SPICE Engine
│   ├── In-memory calculation
│   ├── Fast query performance
│   ├── Auto-refresh
│   └── Incremental updates
├── Visualizations
│   ├── Charts (bar, line, pie, etc.)
│   ├── Tables and pivot tables
│   ├── KPIs and gauges
│   ├── Maps and heat maps
│   └── Custom visuals
├── Features
│   ├── Auto-generated insights (ML)
│   ├── Forecasting
│   ├── Anomaly detection
│   └── Natural language queries (Q)
└── Sharing
    ├── Dashboards
    ├── Embedded analytics
    ├── Email reports
    └── Row-level security
```

### Complete QuickSight Implementation:

```javascript
// quicksight-service.js
import {
  QuickSightClient,
  CreateDataSourceCommand,
  CreateDataSetCommand,
  CreateAnalysisCommand,
  CreateDashboardCommand,
  UpdateDashboardPermissionsCommand,
  CreateTemplateCommand,
  GenerateEmbedUrlForRegisteredUserCommand
} from '@aws-sdk/client-quicksight';

const quicksightClient = new QuickSightClient({ region: 'us-east-1' });

class QuickSightService {
  constructor(awsAccountId) {
    this.awsAccountId = awsAccountId;
  }
  
  // 1. Create Athena Data Source
  async createAthenaDataSource(dataSourceId, name, workGroup = 'primary') {
    const command = new CreateDataSourceCommand({
      AwsAccountId: this.awsAccountId,
      DataSourceId: dataSourceId,
      Name: name,
      Type: 'ATHENA',
      DataSourceParameters: {
        AthenaParameters: {
          WorkGroup: workGroup
        }
      },
      Permissions: [
        {
          Principal: `arn:aws:quicksight:us-east-1:${this.awsAccountId}:user/default/admin`,
          Actions: [
            'quicksight:DescribeDataSource',
            'quicksight:DescribeDataSourcePermissions',
            'quicksight:PassDataSource',
            'quicksight:UpdateDataSource',
            'quicksight:DeleteDataSource',
            'quicksight:UpdateDataSourcePermissions'
          ]
        }
      ]
    });
    
    try {
      const response = await quicksightClient.send(command);
      console.log('Data source created:', response.Arn);
      return response;
    } catch (error) {
      console.error('Failed to create data source:', error);
      throw error;
    }
  }
  
  // 2. Create Dataset
  async createDataSet(dataSetId, name, dataSourceArn, tableName, columns) {
    const command = new CreateDataSetCommand({
      AwsAccountId: this.awsAccountId,
      DataSetId: dataSetId,
      Name: name,
      ImportMode: 'SPICE',  // SPICE or DIRECT_QUERY
      PhysicalTableMap: {
        'table1': {
          RelationalTable: {
            DataSourceArn: dataSourceArn,
            Schema: 'default',
            Name: tableName,
            InputColumns: columns.map(col => ({
              Name: col.name,
              Type: col.type  // STRING, INTEGER, DECIMAL, DATETIME
            }))
          }
        }
      },
      LogicalTableMap: {
        'logical1': {
          Alias: name,
          Source: {
            PhysicalTableId: 'table1'
          },
          DataTransforms: []
        }
      },
      Permissions: [
        {
          Principal: `arn:aws:quicksight:us-east-1:${this.awsAccountId}:user/default/admin`,
          Actions: [
            'quicksight:DescribeDataSet',
            'quicksight:DescribeDataSetPermissions',
            'quicksight:PassDataSet',
            'quicksight:DescribeIngestion',
            'quicksight:ListIngestions',
            'quicksight:UpdateDataSet',
            'quicksight:DeleteDataSet',
            'quicksight:CreateIngestion',
            'quicksight:CancelIngestion',
            'quicksight:UpdateDataSetPermissions'
          ]
        }
      ]
    });
    
    const response = await quicksightClient.send(command);
    console.log('Dataset created:', response.Arn);
    return response;
  }
  
  // 3. Create Dashboard
  async createDashboard(dashboardId, name, templateArn, dataSetArns) {
    const command = new CreateDashboardCommand({
      AwsAccountId: this.awsAccountId,
      DashboardId: dashboardId,
      Name: name,
      SourceEntity: {
        SourceTemplate: {
          Arn: templateArn,
          DataSetReferences: dataSetArns.map((arn, index) => ({
            DataSetPlaceholder: `dataset${index + 1}`,
            DataSetArn: arn
          }))
        }
      },
      Permissions: [
        {
          Principal: `arn:aws:quicksight:us-east-1:${this.awsAccountId}:user/default/admin`,
          Actions: [
            'quicksight:DescribeDashboard',
            'quicksight:ListDashboardVersions',
            'quicksight:UpdateDashboardPermissions',
            'quicksight:QueryDashboard',
            'quicksight:UpdateDashboard',
            'quicksight:DeleteDashboard',
            'quicksight:DescribeDashboardPermissions',
            'quicksight:UpdateDashboardPublishedVersion'
          ]
        }
      ],
      DashboardPublishOptions: {
        AdHocFilteringOption: {
          AvailabilityStatus: 'ENABLED'
        },
        ExportToCSVOption: {
          AvailabilityStatus: 'ENABLED'
        },
        SheetControlsOption: {
          VisibilityState: 'EXPANDED'
        }
      }
    });
    
    const response = await quicksightClient.send(command);
    console.log('Dashboard created:', response.Arn);
    return response;
  }
  
  // 4. Update Dashboard Permissions (for sharing)
  async shareDashboard(dashboardId, userArns) {
    const grantPermissions = userArns.map(userArn => ({
      Principal: userArn,
      Actions: [
        'quicksight:DescribeDashboard',
        'quicksight:ListDashboardVersions',
        'quicksight:QueryDashboard'
      ]
    }));
    
    const command = new UpdateDashboardPermissionsCommand({
      AwsAccountId: this.awsAccountId,
      DashboardId: dashboardId,
      GrantPermissions: grantPermissions
    });
    
    await quicksightClient.send(command);
    console.log('Dashboard permissions updated');
  }
  
  // 5. Generate Embed URL
  async generateEmbedUrl(dashboardId, userArn, sessionLifetime = 600) {
    const command = new GenerateEmbedUrlForRegisteredUserCommand({
      AwsAccountId: this.awsAccountId,
      UserArn: userArn,
      SessionLifetimeInMinutes: sessionLifetime,
      ExperienceConfiguration: {
        Dashboard: {
          InitialDashboardId: dashboardId
        }
      }
    });
    
    const response = await quicksightClient.send(command);
    return response.EmbedUrl;
  }
}

// Dashboard Definition Helper
class DashboardBuilder {
  // Create sales dashboard definition
  static salesDashboard() {
    return {
      sheets: [
        {
          name: 'Sales Overview',
          visuals: [
            {
              type: 'KPI',
              title: 'Total Revenue',
              metric: 'SUM(amount)',
              comparison: 'Previous Period'
            },
            {
              type: 'LineChart',
              title: 'Revenue Trend',
              xAxis: 'date',
              yAxis: 'SUM(amount)',
              colorBy: 'product_category'
            },
            {
              type: 'BarChart',
              title: 'Revenue by Region',
              xAxis: 'region',
              yAxis: 'SUM(amount)',
              sort: 'Descending'
            },
            {
              type: 'PieChart',
              title: 'Revenue by Product',
              dimension: 'product_name',
              metric: 'SUM(amount)'
            }
          ],
          filters: [
            {
              field: 'date',
              type: 'DateRange',
              default: 'Last 30 Days'
            },
            {
              field: 'region',
              type: 'MultiSelect',
              default: 'All'
            }
          ]
        },
        {
          name: 'Customer Analytics',
          visuals: [
            {
              type: 'Table',
              title: 'Top Customers',
              columns: ['customer_name', 'total_orders', 'total_revenue'],
              sort: 'total_revenue DESC',
              limit: 20
            },
            {
              type: 'HeatMap',
              title: 'Sales by Day and Hour',
              rows: 'day_of_week',
              columns: 'hour_of_day',
              values: 'COUNT(order_id)'
            }
          ]
        }
      ],
      parameters: [
        {
          name: 'DateRangeParameter',
          type: 'DateTime',
          defaultValue: 'Last30Days'
        },
        {
          name: 'RegionParameter',
          type: 'String',
          defaultValue: 'All'
        }
      ]
    };
  }
}

// Usage - Create Complete BI Solution
const quicksight = new QuickSightService('123456789012');

// Create Athena data source
const dataSource = await quicksight.createAthenaDataSource(
  'sales-data-source',
  'Sales Analytics Data Source',
  'primary'
);

// Create dataset
const dataset = await quicksight.createDataSet(
  'sales-dataset',
  'Sales Transactions',
  dataSource.Arn,
  'sales_transactions',
  [
    { name: 'transaction_id', type: 'INTEGER' },
    { name: 'date', type: 'DATETIME' },
    { name: 'customer_name', type: 'STRING' },
    { name: 'product_name', type: 'STRING' },
    { name: 'amount', type: 'DECIMAL' },
    { name: 'quantity', type: 'INTEGER' },
    { name: 'region', type: 'STRING' }
  ]
);

// Create dashboard (assumes template exists)
const dashboard = await quicksight.createDashboard(
  'sales-dashboard',
  'Sales Analytics Dashboard',
  'arn:aws:quicksight:us-east-1:123456789012:template/sales-template',
  [dataset.Arn]
);

// Share dashboard with team
await quicksight.shareDashboard('sales-dashboard', [
  'arn:aws:quicksight:us-east-1:123456789012:user/default/analyst1',
  'arn:aws:quicksight:us-east-1:123456789012:user/default/analyst2'
]);

// Generate embed URL for application
const embedUrl = await quicksight.generateEmbedUrl(
  'sales-dashboard',
  'arn:aws:quicksight:us-east-1:123456789012:user/default/webapp-user',
  600  // 10 hours
);

console.log('Dashboard embed URL:', embedUrl);
```

### Best Practices:

1. ✅ Use SPICE for better performance
2. ✅ Implement row-level security (RLS)
3. ✅ Schedule automatic data refreshes
4. ✅ Use calculated fields for custom metrics
5. ✅ Leverage ML Insights for anomaly detection
6. ✅ Embed dashboards in applications
7. ✅ Use parameters for dynamic filtering
8. ✅ Optimize dataset with aggregations
9. ✅ Monitor usage with CloudTrail
10. ✅ Use Q for natural language queries

---

## Key Takeaways

### Question 43 (SageMaker):
- End-to-end ML platform
- Built-in algorithms and custom models
- Training with spot instances
- Real-time and batch inference
- Hyperparameter tuning
- Model monitoring and drift detection
- MLOps with Pipelines

### Question 44 (Athena):
- Serverless SQL queries on S3
- Pay per query (data scanned)
- Use Parquet/ORC for 80-90% cost savings
- Partition data for performance
- Federated queries to multiple sources
- Integration with Glue Data Catalog
- CTAS for data transformation

### Question 45 (QuickSight):
- Cloud-native BI service
- SPICE in-memory engine
- ML-powered insights
- Embedded analytics
- Pay-per-session pricing
- Natural language queries (Q)
- Row-level security for data governance
