# CloudWatch Monitoring - Serverless Order Processing Microservice

## 📋 Overview

Comprehensive monitoring setup with CloudWatch Logs, Metrics, Dashboards, Alarms, and X-Ray tracing.

---

## 1️⃣ CloudWatch Log Groups

```yaml
# CloudFormation
OrderProcessingLogGroup:
  Type: AWS::Logs::LogGroup
  Properties:
    LogGroupName: !Sub '/aws/lambda/${Environment}-api-handler'
    RetentionInDays: !If [IsProduction, 90, 7]
    KmsKeyId: !ImportValue 'Fn::Sub': '${Environment}-LogsKMSKey'

OrderProcessorLogGroup:
  Type: AWS::Logs::LogGroup
  Properties:
    LogGroupName: !Sub '/aws/lambda/${Environment}-order-processor'
    RetentionInDays: !If [IsProduction, 90, 7]
```

---

## 2️⃣ Custom Metrics

```typescript
// src/utils/metrics.ts
import { CloudWatch } from 'aws-sdk';

const cloudwatch = new CloudWatch({ region: process.env.AWS_REGION });

export async function putMetric(
  metricName: string,
  value: number,
  unit: string = 'Count',
  dimensions?: Record<string, string>
): Promise<void> {
  const params: CloudWatch.PutMetricDataInput = {
    Namespace: `${process.env.STAGE}/OrderProcessing`,
    MetricData: [
      {
        MetricName: metricName,
        Value: value,
        Unit: unit,
        Timestamp: new Date(),
        Dimensions: dimensions ? Object.entries(dimensions).map(([Name, Value]) => ({
          Name,
          Value
        })) : []
      }
    ]
  };

  try {
    await cloudwatch.putMetricData(params).promise();
  } catch (error) {
    console.error('Error putting metric:', error);
  }
}

// Usage in Lambda
await putMetric('OrderCreated', 1, 'Count', {
  Environment: process.env.STAGE!,
  OrderStatus: 'PENDING'
});

await putMetric('OrderProcessingTime', 1250, 'Milliseconds', {
  Function: 'order-processor'
});
```

---

## 3️⃣ CloudWatch Dashboard

```yaml
OrderProcessingDashboard:
  Type: AWS::CloudWatch::Dashboard
  Properties:
    DashboardName: !Sub '${Environment}-order-processing'
    DashboardBody: !Sub |
      {
        "widgets": [
          {
            "type": "metric",
            "properties": {
              "metrics": [
                ["AWS/Lambda", "Invocations", {"stat": "Sum"}],
                [".", "Errors", {"stat": "Sum"}],
                [".", "Throttles", {"stat": "Sum"}]
              ],
              "period": 300,
              "stat": "Sum",
              "region": "${AWS::Region}",
              "title": "Lambda Metrics",
              "yAxis": {"left": {"min": 0}}
            }
          },
          {
            "type": "metric",
            "properties": {
              "metrics": [
                ["AWS/Lambda", "Duration", {"stat": "Average"}],
                ["...", {"stat": "p99"}]
              ],
              "period": 300,
              "stat": "Average",
              "region": "${AWS::Region}",
              "title": "Lambda Duration"
            }
          },
          {
            "type": "metric",
            "properties": {
              "metrics": [
                ["AWS/SQS", "ApproximateNumberOfMessagesVisible"],
                [".", "ApproximateAgeOfOldestMessage"]
              ],
              "period": 300,
              "stat": "Average",
              "region": "${AWS::Region}",
              "title": "SQS Queue Depth"
            }
          },
          {
            "type": "metric",
            "properties": {
              "metrics": [
                ["AWS/DynamoDB", "ConsumedReadCapacityUnits", {"stat": "Sum"}],
                [".", "ConsumedWriteCapacityUnits", {"stat": "Sum"}]
              ],
              "period": 300,
              "stat": "Sum",
              "region": "${AWS::Region}",
              "title": "DynamoDB Capacity"
            }
          },
          {
            "type": "log",
            "properties": {
              "query": "SOURCE '/aws/lambda/${Environment}-api-handler'\n| fields @timestamp, @message\n| filter @message like /ERROR/\n| sort @timestamp desc\n| limit 20",
              "region": "${AWS::Region}",
              "title": "Recent Errors"
            }
          }
        ]
      }
```

---

## 4️⃣ CloudWatch Alarms

```yaml
# High error rate alarm
LambdaErrorAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: !Sub '${Environment}-lambda-high-errors'
    AlarmDescription: Alert when Lambda error rate is high
    MetricName: Errors
    Namespace: AWS/Lambda
    Statistic: Sum
    Period: 300
    EvaluationPeriods: 2
    Threshold: 10
    ComparisonOperator: GreaterThanThreshold
    Dimensions:
      - Name: FunctionName
        Value: !Ref ApiHandlerFunction
    AlarmActions:
      - !Ref AlertTopic
    TreatMissingData: notBreaching

# DLQ messages alarm
DLQMessagesAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: !Sub '${Environment}-dlq-messages'
    MetricName: ApproximateNumberOfMessagesVisible
    Namespace: AWS/SQS
    Statistic: Sum
    Period: 300
    EvaluationPeriods: 1
    Threshold: 1
    ComparisonOperator: GreaterThanOrEqualToThreshold
    Dimensions:
      - Name: QueueName
        Value: !Sub '${Environment}-order-processing-dlq.fifo'
    AlarmActions:
      - !Ref AlertTopic

# API Gateway 5XX errors
ApiGateway5XXAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: !Sub '${Environment}-api-5xx-errors'
    MetricName: 5XXError
    Namespace: AWS/ApiGateway
    Statistic: Sum
    Period: 300
    EvaluationPeriods: 2
    Threshold: 10
    ComparisonOperator: GreaterThanThreshold
    Dimensions:
      - Name: ApiName
        Value: !Ref OrderProcessingApi
    AlarmActions:
      - !Ref AlertTopic

# DynamoDB throttle alarm
DynamoDBThrottleAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: !Sub '${Environment}-dynamodb-throttles'
    MetricName: UserErrors
    Namespace: AWS/DynamoDB
    Statistic: Sum
    Period: 300
    EvaluationPeriods: 2
    Threshold: 5
    ComparisonOperator: GreaterThanThreshold
    Dimensions:
      - Name: TableName
        Value: !Ref OrdersTable
    AlarmActions:
      - !Ref AlertTopic
```

---

## 5️⃣ X-Ray Tracing

```typescript
// Enable X-Ray in Lambda
import AWSXRay from 'aws-xray-sdk-core';
import AWS from 'aws-sdk';

// Wrap AWS SDK
const XAWS = AWSXRay.captureAWS(AWS);

// Use wrapped SDK
const dynamodb = new XAWS.DynamoDB.DocumentClient();
const sqs = new XAWS.SQS();

// Add custom subsegments
export async function processWithTracing(orderId: string): Promise<void> {
  const segment = AWSXRay.getSegment();
  const subsegment = segment?.addNewSubsegment('ProcessOrder');
  
  try {
    subsegment?.addAnnotation('orderId', orderId);
    subsegment?.addMetadata('orderData', { orderId, timestamp: new Date() });
    
    // Your processing logic
    await processOrder(orderId);
    
    subsegment?.close();
  } catch (error) {
    subsegment?.addError(error as Error);
    subsegment?.close();
    throw error;
  }
}
```

```yaml
# Enable X-Ray in CloudFormation
ApiHandlerFunction:
  Type: AWS::Lambda::Function
  Properties:
    TracingConfig:
      Mode: Active
```

---

## 6️⃣ Log Insights Queries

```sql
-- Query 1: Find errors in last hour
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 50

-- Query 2: Average order processing time
fields @timestamp, duration
| filter @message like /Order processed/
| parse @message "duration: * ms" as duration
| stats avg(duration) as avg_duration by bin(5m)

-- Query 3: Orders by status
fields @timestamp, orderStatus
| filter @message like /Order created/
| parse @message "status: *" as orderStatus
| stats count() by orderStatus

-- Query 4: Failed orders
fields @timestamp, orderId, errorMessage
| filter @message like /Order failed/
| parse @message "orderId: *, error: *" as orderId, errorMessage
| sort @timestamp desc

-- Query 5: API latency percentiles
fields @timestamp, @duration
| filter @type = "REPORT"
| stats avg(@duration), pct(@duration, 50), pct(@duration, 95), pct(@duration, 99) by bin(5m)
```

---

## 7️⃣ Composite Alarms

```yaml
# Critical system alarm (multiple failures)
CriticalSystemAlarm:
  Type: AWS::CloudWatch::CompositeAlarm
  Properties:
    AlarmName: !Sub '${Environment}-critical-system'
    AlarmDescription: Multiple critical issues detected
    ActionsEnabled: true
    AlarmActions:
      - !Ref PagerDutyTopic
    AlarmRule: !Sub |
      (ALARM(${LambdaErrorAlarm}) OR ALARM(${DLQMessagesAlarm})) 
      AND ALARM(${ApiGateway5XXAlarm})
```

---

## 8️⃣ Monitoring Script

```powershell
# scripts/monitor-metrics.ps1

# Get Lambda metrics
aws cloudwatch get-metric-statistics `
  --namespace AWS/Lambda `
  --metric-name Errors `
  --dimensions Name=FunctionName,Value=dev-api-handler `
  --start-time (Get-Date).AddHours(-1).ToString("yyyy-MM-ddTHH:mm:ss") `
  --end-time (Get-Date).ToString("yyyy-MM-ddTHH:mm:ss") `
  --period 300 `
  --statistics Sum

# Get SQS queue depth
aws cloudwatch get-metric-statistics `
  --namespace AWS/SQS `
  --metric-name ApproximateNumberOfMessagesVisible `
  --dimensions Name=QueueName,Value=dev-order-processing.fifo `
  --start-time (Get-Date).AddHours(-1).ToString("yyyy-MM-ddTHH:mm:ss") `
  --end-time (Get-Date).ToString("yyyy-MM-ddTHH:mm:ss") `
  --period 300 `
  --statistics Average

# Query logs
aws logs start-query `
  --log-group-name "/aws/lambda/dev-api-handler" `
  --start-time ((Get-Date).AddHours(-1) | Get-Date -UFormat %s) `
  --end-time (Get-Date | Get-Date -UFormat %s) `
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 20'
```

---

## 🎯 Monitoring Checklist

### DO ✅
1. Set up CloudWatch alarms for all critical metrics
2. Create comprehensive dashboards
3. Enable X-Ray tracing
4. Use structured logging (JSON)
5. Monitor DLQ depth
6. Track custom business metrics
7. Set up SNS alerts
8. Monitor API Gateway metrics
9. Use Log Insights for troubleshooting
10. Review metrics regularly

### DON'T ❌
1. Don't ignore CloudWatch costs
2. Don't skip log retention policies
3. Don't forget to encrypt logs
4. Don't over-alert (alarm fatigue)
5. Don't ignore slow queries

---

**Next:** [Deployment Pipeline](./10_DEPLOYMENT_PIPELINE.md)

---

**Last Updated**: November 12, 2025
