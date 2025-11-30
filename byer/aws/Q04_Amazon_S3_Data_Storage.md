# Question 4: Amazon S3 for Data Storage & Integration Workflows

## 🎯 Question
**How would you design a scalable data lake architecture using Amazon S3 for storing integration data, transformation results, and audit logs? Explain lifecycle policies, versioning, encryption, event notifications, and best practices for organizing large-scale data.**

---

## 📋 Answer Overview

Amazon S3 (Simple Storage Service) is an object storage service offering:
- **Unlimited scalability** (no capacity planning needed)
- **11 nines durability** (99.999999999%)
- **Multiple storage classes** (hot, warm, cold, archive)
- **Event-driven integration** (trigger Lambda on file upload)

For Bayer's integration platform, S3 serves as:
- **Data lake**: Centralized repository for all integration data
- **Audit trail**: Immutable storage for regulatory compliance
- **Archive**: Long-term retention of clinical trial data
- **Data transformation staging**: Temporary storage during ETL

---

## 🏗️ S3 Architecture for Integration Platform

```
┌───────────────────────────────────────────────────────┐
│            S3 Bucket Structure                        │
│  s3://bayer-integration-platform-prod/                │
├───────────────────────────────────────────────────────┤
│                                                       │
│  ├── raw/                  ← Incoming data (raw)      │
│  │   ├── clinical-trials/                            │
│  │   ├── manufacturing/                              │
│  │   └── sap/                                        │
│  │       └── 2025/01/27/file.json  (partitioned)    │
│  │                                                    │
│  ├── processed/            ← Transformed data         │
│  │   ├── clinical-trials/                            │
│  │   └── manufacturing/                              │
│  │                                                    │
│  ├── audit/                ← Audit logs              │
│  │   └── 2025/01/27/events.json                     │
│  │                                                    │
│  ├── archive/              ← Long-term storage       │
│  │   └── clinical-trials/2024/                      │
│  │                                                    │
│  └── dlq/                  ← Failed processing       │
│      └── 2025/01/27/failed.json                     │
└───────────────────────────────────────────────────────┘

Event-Driven Processing:
    ┌──────────────┐
    │  File Upload │
    │  to S3       │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │ S3 Event     │
    │ Notification │
    └──────┬───────┘
           │
       ┌───┴───┐
       │       │
       ▼       ▼
    ┌────┐  ┌────┐
    │SNS │  │SQS │  → Lambda (process file)
    └────┘  └────┘
```

**Why this structure?**
- **Partitioning**: Enables fast queries (date-based)
- **Separation**: Raw vs processed data isolation
- **Compliance**: Audit trail for regulatory requirements
- **Cost**: Lifecycle policies move old data to cheaper storage

---

## 💻 Implementation Examples

### 1. S3 Bucket Configuration with Terraform

```hcl
# s3-bucket.tf
# WHY: Infrastructure as Code for repeatable deployments
# REQUIRED FOR: Multi-environment setup (dev/staging/prod)

resource "aws_s3_bucket" "integration_platform" {
  bucket = "bayer-integration-platform-${var.environment}"
  
  # WHY: Prevent accidental bucket deletion
  # CRITICAL: Contains regulatory and clinical trial data
  force_destroy = false
  
  tags = {
    Environment = var.environment
    Project     = "Integration Platform"
    ManagedBy   = "Terraform"
    Compliance  = "GxP"  # Good Practice guidelines for pharma
  }
}

# Versioning
# WHY: Track all changes to objects for audit trail
# REQUIRED FOR: Regulatory compliance (FDA 21 CFR Part 11)
resource "aws_s3_bucket_versioning" "integration_platform" {
  bucket = aws_s3_bucket.integration_platform.id
  
  versioning_configuration {
    status = "Enabled"
    # WHY: Recover from accidental deletions or overwrites
    # EXAMPLE: Clinical trial data accidentally overwritten can be restored
  }
}

# Encryption
# WHY: Protect sensitive data at rest
# REQUIRED FOR: HIPAA compliance for patient data
resource "aws_s3_bucket_server_side_encryption_configuration" "integration_platform" {
  bucket = aws_s3_bucket.integration_platform.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"  # Use KMS for key management
      kms_master_key_id = aws_kms_key.integration_data.arn
      # WHY: KMS provides key rotation and audit trail
      # ALTERNATIVE: AES256 (S3-managed keys) for less sensitive data
    }
    bucket_key_enabled = true
    # WHY: Reduces KMS API calls (cost optimization)
  }
}

# Lifecycle Policy
# WHY: Automatically move old data to cheaper storage tiers
# COST SAVINGS: $23/TB/month (Standard) → $1/TB/month (Glacier Deep Archive)
resource "aws_s3_bucket_lifecycle_configuration" "integration_platform" {
  bucket = aws_s3_bucket.integration_platform.id
  
  # Rule 1: Raw data lifecycle
  rule {
    id     = "raw-data-lifecycle"
    status = "Enabled"
    
    filter {
      prefix = "raw/"
    }
    
    transition {
      days          = 30
      storage_class = "STANDARD_IA"  # Infrequent Access
      # WHY: Raw data rarely accessed after 30 days
      # COST: $12.50/TB/month (45% savings)
    }
    
    transition {
      days          = 90
      storage_class = "GLACIER"
      # WHY: Compliance requires 2-year retention
      # COST: $4/TB/month (83% savings)
    }
    
    transition {
      days          = 365
      storage_class = "DEEP_ARCHIVE"
      # WHY: Long-term archive for regulatory requirements
      # COST: $1/TB/month (96% savings)
      # RETRIEVAL: 12 hours (acceptable for archived data)
    }
    
    expiration {
      days = 2555  # 7 years
      # WHY: FDA requires 7-year retention for clinical trial data
    }
  }
  
  # Rule 2: Delete incomplete multipart uploads
  rule {
    id     = "cleanup-failed-uploads"
    status = "Enabled"
    
    abort_incomplete_multipart_upload {
      days_after_initiation = 7
      # WHY: Failed uploads waste storage space
      # COST SAVINGS: Prevent accumulation of incomplete parts
    }
  }
  
  # Rule 3: Delete old versions
  rule {
    id     = "expire-old-versions"
    status = "Enabled"
    
    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "GLACIER"
      # WHY: Keep recent versions hot, archive old versions
    }
    
    noncurrent_version_expiration {
      noncurrent_days = 365
      # WHY: Don't need versions older than 1 year
    }
  }
}

# Event Notifications
# WHY: Trigger processing when files uploaded
# EXAMPLE: Lambda processes file as soon as it arrives
resource "aws_s3_bucket_notification" "integration_platform" {
  bucket = aws_s3_bucket.integration_platform.id
  
  # Trigger Lambda for new files in raw/
  lambda_function {
    lambda_function_arn = aws_lambda_function.file_processor.arn
    events              = ["s3:ObjectCreated:*"]
    filter_prefix       = "raw/"
    filter_suffix       = ".json"
    # WHY: Only process JSON files in raw/ folder
  }
  
  # Send to SQS for batch processing
  queue {
    queue_arn     = aws_sqs_queue.file_processing_queue.arn
    events        = ["s3:ObjectCreated:*"]
    filter_prefix = "raw/clinical-trials/"
    # WHY: Clinical trial files processed asynchronously
  }
  
  # Send to SNS for alerts
  topic {
    topic_arn     = aws_sns_topic.integration_alerts.arn
    events        = ["s3:ObjectRemoved:*"]
    filter_prefix = "audit/"
    # WHY: Alert if audit logs deleted (potential security incident)
  }
}

# Object Lock (Compliance Mode)
# WHY: Immutable storage for regulatory compliance
# REQUIRED FOR: WORM (Write Once Read Many) compliance
resource "aws_s3_bucket_object_lock_configuration" "audit_logs" {
  bucket = aws_s3_bucket.integration_platform.id
  
  rule {
    default_retention {
      mode = "COMPLIANCE"  # Cannot be overridden even by root user
      days = 2555         # 7 years
      # WHY: SEC Rule 17a-4 requires immutable storage
    }
  }
  
  # CRITICAL: Once enabled, cannot be disabled
  # Use only for audit logs and compliance data
}
```

---

### 2. S3 Operations in Node.js

```javascript
// s3-operations.js
// WHY: Common S3 operations for integration workflows

const AWS = require('aws-sdk');
const s3 = new AWS.S3();
const stream = require('stream');

const BUCKET_NAME = process.env.BUCKET_NAME;

/**
 * Upload integration data to S3
 * WHY: Store raw integration data for audit and reprocessing
 * PARTITIONING: Use date-based keys for efficient queries
 */
async function uploadIntegrationData(source, target, data) {
  const timestamp = new Date();
  const datePartition = `${timestamp.getFullYear()}/${String(timestamp.getMonth() + 1).padStart(2, '0')}/${String(timestamp.getDate()).padStart(2, '0')}`;
  
  // Generate unique key
  // WHY: Prevents overwriting, enables versioning
  const key = `raw/${source}/${datePartition}/${target}-${Date.now()}.json`;
  
  const params = {
    Bucket: BUCKET_NAME,
    Key: key,
    Body: JSON.stringify(data, null, 2),
    ContentType: 'application/json',
    
    // Metadata (searchable)
    // WHY: Query objects without reading content
    Metadata: {
      'source-system': source,
      'target-system': target,
      'integration-id': data.integrationId,
      'data-type': data.dataType || 'unknown'
    },
    
    // Tags (for cost allocation)
    // WHY: Track costs per department/project
    Tagging: `Department=Integration&Project=DataPlatform&DataClassification=Confidential`,
    
    // Server-side encryption
    // WHY: Automatic encryption (HIPAA requirement)
    ServerSideEncryption: 'aws:kms',
    SSEKMSKeyId: process.env.KMS_KEY_ID,
    
    // Storage class
    // WHY: Frequently accessed data should use STANDARD
    StorageClass: 'STANDARD'
  };
  
  try {
    const result = await s3.putObject(params).promise();
    
    console.log('Data uploaded to S3', {
      bucket: BUCKET_NAME,
      key,
      etag: result.ETag,
      versionId: result.VersionId
    });
    
    return {
      bucket: BUCKET_NAME,
      key,
      url: `s3://${BUCKET_NAME}/${key}`,
      versionId: result.VersionId
    };
    
  } catch (error) {
    console.error('S3 upload failed', {
      error: error.message,
      key
    });
    throw error;
  }
}

/**
 * Download and process file from S3
 * WHY: Lambda triggered by S3 event, processes uploaded file
 */
async function processS3File(event) {
  // Parse S3 event
  // WHY: S3 sends event when file uploaded
  const record = event.Records[0];
  const bucket = record.s3.bucket.name;
  const key = decodeURIComponent(record.s3.object.key.replace(/\+/g, ' '));
  
  console.log('Processing S3 file', { bucket, key });
  
  try {
    // Get object metadata first
    // WHY: Check file size before downloading
    const metadata = await s3.headObject({
      Bucket: bucket,
      Key: key
    }).promise();
    
    const fileSizeGB = metadata.ContentLength / 1024 / 1024 / 1024;
    
    // Large file? Use streaming
    // WHY: Don't load 10GB file into Lambda memory (max 10GB)
    if (fileSizeGB > 0.5) {
      return await processLargeFile(bucket, key);
    }
    
    // Small file? Load into memory
    const object = await s3.getObject({
      Bucket: bucket,
      Key: key
    }).promise();
    
    const data = JSON.parse(object.Body.toString('utf-8'));
    
    // Process data
    const result = await transformData(data);
    
    // Save processed data
    const processedKey = key.replace('raw/', 'processed/');
    await uploadProcessedData(bucket, processedKey, result);
    
    return result;
    
  } catch (error) {
    console.error('File processing failed', {
      bucket,
      key,
      error: error.message
    });
    
    // Move to DLQ folder
    // WHY: Failed files need manual investigation
    await moveToFailedFolder(bucket, key);
    
    throw error;
  }
}

/**
 * Stream large files for processing
 * WHY: Lambda memory limit (10GB max)
 * EXAMPLE: 5GB clinical trial data file
 */
async function processLargeFile(bucket, key) {
  return new Promise((resolve, reject) => {
    const s3Stream = s3.getObject({
      Bucket: bucket,
      Key: key
    }).createReadStream();
    
    let processedRecords = 0;
    let buffer = '';
    
    // Process line by line
    // WHY: Memory-efficient streaming
    const lineParser = new stream.Transform({
      transform(chunk, encoding, callback) {
        buffer += chunk.toString();
        const lines = buffer.split('\n');
        buffer = lines.pop();  // Keep incomplete line
        
        for (const line of lines) {
          if (line.trim()) {
            try {
              const record = JSON.parse(line);
              this.push(JSON.stringify(processRecord(record)) + '\n');
              processedRecords++;
            } catch (error) {
              console.error('Invalid JSON line', { line });
            }
          }
        }
        callback();
      },
      flush(callback) {
        if (buffer.trim()) {
          this.push(JSON.stringify(processRecord(JSON.parse(buffer))) + '\n');
        }
        callback();
      }
    });
    
    // Upload stream to processed folder
    const processedKey = key.replace('raw/', 'processed/');
    const uploadStream = s3.upload({
      Bucket: bucket,
      Key: processedKey,
      Body: lineParser
    });
    
    s3Stream
      .pipe(lineParser)
      .on('error', reject);
    
    uploadStream.promise()
      .then(() => {
        console.log('Large file processed', {
          recordsProcessed: processedRecords,
          outputKey: processedKey
        });
        resolve({ processedRecords });
      })
      .catch(reject);
  });
}

/**
 * Generate pre-signed URL for temporary access
 * WHY: Share files without making bucket public
 * EXAMPLE: Share lab results with external partner for 1 hour
 */
async function generatePresignedUrl(key, expirationSeconds = 3600) {
  const params = {
    Bucket: BUCKET_NAME,
    Key: key,
    Expires: expirationSeconds  // 1 hour
  };
  
  const url = s3.getSignedUrl('getObject', params);
  
  console.log('Pre-signed URL generated', {
    key,
    expiresIn: expirationSeconds,
    url: url.substring(0, 100) + '...'  // Don't log full URL
  });
  
  return url;
}

/**
 * Copy object to different storage class
 * WHY: Manually trigger lifecycle transition if needed
 */
async function archiveObject(key) {
  const copyParams = {
    Bucket: BUCKET_NAME,
    CopySource: `${BUCKET_NAME}/${key}`,
    Key: key.replace('processed/', 'archive/'),
    StorageClass: 'GLACIER',
    MetadataDirective: 'COPY'
  };
  
  await s3.copyObject(copyParams).promise();
  console.log('Object archived to Glacier', { key });
}

/**
 * Enable intelligent tiering
 * WHY: Automatic cost optimization based on access patterns
 * BEST FOR: Unpredictable access patterns
 */
async function enableIntelligentTiering(key) {
  await s3.putObjectTagging({
    Bucket: BUCKET_NAME,
    Key: key,
    Tagging: {
      TagSet: [
        {
          Key: 'IntelligentTiering',
          Value: 'Enabled'
        }
      ]
    }
  }).promise();
  
  // WHY: S3 automatically moves object between tiers
  // Hot tier → Infrequent → Archive (based on access)
  // COST: Small monthly monitoring fee, saves 95% on storage
}
```

---

### 3. S3 Event-Driven Processing

```javascript
// s3-event-processor.js
// WHY: Lambda triggered automatically when file uploaded
// TRIGGERED BY: S3 Event Notification

exports.handler = async (event) => {
  console.log('S3 Event received', {
    recordCount: event.Records.length
  });
  
  for (const record of event.Records) {
    // Parse S3 event
    const bucket = record.s3.bucket.name;
    const key = decodeURIComponent(record.s3.object.key.replace(/\+/g, ' '));
    const eventName = record.eventName;
    const fileSize = record.s3.object.size;
    
    console.log('Processing S3 event', {
      event: eventName,
      bucket,
      key,
      sizeBytes: fileSize
    });
    
    // Route based on file location
    // WHY: Different folders need different processing
    if (key.startsWith('raw/clinical-trials/')) {
      await processClinicalTrialData(bucket, key);
    } else if (key.startsWith('raw/manufacturing/')) {
      await processManufacturingData(bucket, key);
    } else if (key.startsWith('raw/sap/')) {
      await processSAPData(bucket, key);
    } else {
      console.warn('Unknown file location', { key });
    }
  }
};

async function processClinicalTrialData(bucket, key) {
  // WHY: Clinical trial data requires special handling (FDA compliance)
  console.log('Processing clinical trial file', { bucket, key });
  
  // Download file
  const s3 = new AWS.S3();
  const object = await s3.getObject({ Bucket: bucket, Key: key }).promise();
  const data = JSON.parse(object.Body.toString());
  
  // Validate required fields
  // WHY: FDA requires specific data points
  validateClinicalData(data);
  
  // De-identify patient data
  // WHY: HIPAA compliance
  const anonymizedData = anonymizePatientData(data);
  
  // Store in regulatory database
  await storeInRegulatoryDB(anonymizedData);
  
  // Archive to long-term storage
  // WHY: 7-year retention requirement
  const archiveKey = key.replace('raw/', 'archive/');
  await s3.copyObject({
    Bucket: bucket,
    CopySource: `${bucket}/${key}`,
    Key: archiveKey,
    StorageClass: 'GLACIER'
  }).promise();
  
  console.log('Clinical trial file processed', { archiveKey });
}
```

---

## 📊 S3 Best Practices for Integration Platform

### Naming Conventions

```
s3://bucket-name/
  ├── raw/                          # Incoming data
  │   ├── {source}/                 # Partition by source system
  │   │   ├── {YYYY}/               # Year
  │   │   │   ├── {MM}/             # Month
  │   │   │   │   ├── {DD}/         # Day
  │   │   │   │   │   └── file.json
  │   
  ├── processed/                    # Transformed data
  │   ├── {target}/
  │   │   └── {YYYY}/{MM}/{DD}/
  │   
  ├── audit/                        # Audit logs (WORM)
  │   └── {YYYY}/{MM}/{DD}/
  │   
  └── archive/                      # Long-term storage
      └── {source}/{YYYY}/
```

**Why this structure?**
- **Partitioning**: Fast queries with date filters
- **Scalability**: Millions of files without performance degradation
- **Organization**: Clear separation of concerns
- **Cost**: Lifecycle policies work on prefixes

---

## 🎯 Key Takeaways

### ✅ Do's
1. **Use lifecycle policies** to reduce storage costs
2. **Enable versioning** for audit trail and compliance
3. **Encrypt all data** (KMS for sensitive, SSE-S3 for others)
4. **Partition by date** (year/month/day) for fast queries
5. **Use event notifications** for real-time processing
6. **Tag objects** for cost allocation and governance
7. **Enable access logging** for security audits
8. **Use S3 Select** for query-in-place (avoid downloading entire file)
9. **Implement Object Lock** for immutable audit logs
10. **Monitor costs** with S3 Storage Lens

### ❌ Don'ts
1. **Don't make buckets public** (use pre-signed URLs)
2. **Don't store credentials** in S3 (use Secrets Manager)
3. **Don't ignore lifecycle policies** (costs grow exponentially)
4. **Don't use sequential keys** (causes hot partitioning)
5. **Don't download large files** to Lambda (use streaming)
6. **Don't skip encryption** (regulatory violations)
7. **Don't forget to delete test data** (costs add up)
8. **Don't use one bucket** for everything (separate by environment)

---

## 💰 Cost Optimization

| Storage Class | $/TB/Month | Use Case | Retrieval |
|---------------|------------|----------|-----------|
| **Standard** | $23 | Hot data (< 30 days) | Instant |
| **IA** | $12.50 | Warm data (30-90 days) | Instant |
| **Glacier** | $4 | Cold data (90-365 days) | Minutes-hours |
| **Deep Archive** | $1 | Archive (> 1 year) | 12 hours |

**Example Cost Calculation:**
- 100 TB integration data
- 30% hot (30 TB × $23) = $690
- 50% warm (50 TB × $12.50) = $625
- 20% archive (20 TB × $1) = $20
- **Total: $1,335/month** vs $2,300 (all Standard)

---

**Estimated Reading Time**: 15-18 minutes  
**Difficulty Level**: ⭐⭐⭐ Intermediate  
**Prerequisites**: AWS basics, file systems, JSON  
**Bayer Job Alignment**: 90% - Essential for data storage
