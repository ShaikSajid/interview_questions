# WebSocket & Real-time Communication

## Question 45: WebSocket API with API Gateway

### 📋 Question Statement

Implement real-time banking notifications for Emirates NBD using API Gateway WebSocket APIs, Lambda, and DynamoDB for bidirectional communication.

---

### 🔌 WebSocket Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                 WEBSOCKET REAL-TIME NOTIFICATIONS                           │
└────────────────────────────────────────────────────────────────────────────┘

    ┌──────────────┐
    │ Mobile/Web   │
    │   Client     │
    └──────┬───────┘
           │ WSS://
           v
    ┌─────────────────┐
    │  API Gateway    │
    │   WebSocket     │
    └────────┬────────┘
             │
    ┌────────┼────────┬─────────────┐
    │        │        │             │
    v        v        v             v
┌─────────┐┌─────────┐┌──────────┐┌──────────┐
│ Lambda  ││ Lambda  ││  Lambda  ││  Lambda  │
│$connect ││$default ││$disconnect││broadcast │
└────┬────┘└────┬────┘└─────┬────┘└────┬─────┘
     │          │           │           │
     └──────────┴───────────┴───────────┘
                      │
                      v
              ┌──────────────┐
              │  DynamoDB    │
              │ Connections  │
              └──────────────┘
```

### 📦 WebSocket API CDK Infrastructure

```typescript
// infrastructure/cdk/websocket-api-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as apigatewayv2 from 'aws-cdk-lib/aws-apigatewayv2';
import * as integrations from 'aws-cdk-lib/aws-apigatewayv2-integrations';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as logs from 'aws-cdk-lib/aws-logs';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as subscriptions from 'aws-cdk-lib/aws-sns-subscriptions';

export class WebSocketApiStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // DynamoDB table for connection tracking
    const connectionsTable = new dynamodb.Table(this, 'Connections', {
      tableName: 'websocket-connections',
      partitionKey: { name: 'connectionId', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      timeToLiveAttribute: 'ttl',
      stream: dynamodb.StreamViewType.NEW_AND_OLD_IMAGES
    });

    // GSI for customer queries
    connectionsTable.addGlobalSecondaryIndex({
      indexName: 'customerIdIndex',
      partitionKey: { name: 'customerId', type: dynamodb.AttributeType.STRING },
      projectionType: dynamodb.ProjectionType.ALL
    });

    // Lambda execution role with API Gateway permissions
    const lambdaRole = new iam.Role(this, 'WebSocketLambdaRole', {
      assumedBy: new iam.ServicePrincipal('lambda.amazonaws.com'),
      managedPolicies: [
        iam.ManagedPolicy.fromAwsManagedPolicyName('service-role/AWSLambdaBasicExecutionRole')
      ]
    });

    connectionsTable.grantReadWriteData(lambdaRole);

    // Lambda: $connect
    const connectHandler = new lambda.Function(this, 'ConnectHandler', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/websocket-connect'),
      timeout: cdk.Duration.seconds(10),
      environment: {
        CONNECTIONS_TABLE: connectionsTable.tableName
      },
      role: lambdaRole,
      logRetention: logs.RetentionDays.ONE_WEEK
    });

    // Lambda: $disconnect
    const disconnectHandler = new lambda.Function(this, 'DisconnectHandler', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/websocket-disconnect'),
      timeout: cdk.Duration.seconds(10),
      environment: {
        CONNECTIONS_TABLE: connectionsTable.tableName
      },
      role: lambdaRole
    });

    // Lambda: $default (message handler)
    const messageHandler = new lambda.Function(this, 'MessageHandler', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/websocket-message'),
      timeout: cdk.Duration.seconds(30),
      environment: {
        CONNECTIONS_TABLE: connectionsTable.tableName
      },
      role: lambdaRole
    });

    // Lambda: Broadcast notifications
    const broadcastHandler = new lambda.Function(this, 'BroadcastHandler', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/websocket-broadcast'),
      timeout: cdk.Duration.minutes(5),
      memorySize: 1024,
      environment: {
        CONNECTIONS_TABLE: connectionsTable.tableName
      },
      role: lambdaRole
    });

    // WebSocket API
    const webSocketApi = new apigatewayv2.WebSocketApi(this, 'BankingWebSocketApi', {
      apiName: 'emirates-nbd-websocket-api',
      description: 'Real-time banking notifications',
      connectRouteOptions: {
        integration: new integrations.WebSocketLambdaIntegration('ConnectIntegration', connectHandler)
      },
      disconnectRouteOptions: {
        integration: new integrations.WebSocketLambdaIntegration('DisconnectIntegration', disconnectHandler)
      },
      defaultRouteOptions: {
        integration: new integrations.WebSocketLambdaIntegration('DefaultIntegration', messageHandler)
      }
    });

    // WebSocket Stage
    const stage = new apigatewayv2.WebSocketStage(this, 'ProductionStage', {
      webSocketApi,
      stageName: 'prod',
      autoDeploy: true,
      throttle: {
        rateLimit: 10000,
        burstLimit: 5000
      }
    });

    // Grant API Gateway management permissions
    const apiGatewayPolicy = new iam.PolicyStatement({
      actions: ['execute-api:ManageConnections'],
      resources: [
        `arn:aws:execute-api:${this.region}:${this.account}:${webSocketApi.apiId}/${stage.stageName}/POST/@connections/*`
      ]
    });

    connectHandler.addToRolePolicy(apiGatewayPolicy);
    disconnectHandler.addToRolePolicy(apiGatewayPolicy);
    messageHandler.addToRolePolicy(apiGatewayPolicy);
    broadcastHandler.addToRolePolicy(apiGatewayPolicy);

    // SNS topic for triggering broadcasts
    const notificationTopic = new sns.Topic(this, 'NotificationTopic', {
      topicName: 'websocket-notifications',
      displayName: 'WebSocket Broadcast Notifications'
    });

    notificationTopic.addSubscription(
      new subscriptions.LambdaSubscription(broadcastHandler)
    );

    // Store API endpoint in SSM or environment
    broadcastHandler.addEnvironment('WEBSOCKET_API_ENDPOINT', stage.callbackUrl);
    messageHandler.addEnvironment('WEBSOCKET_API_ENDPOINT', stage.callbackUrl);

    // Outputs
    new cdk.CfnOutput(this, 'WebSocketURL', {
      value: stage.url,
      description: 'WebSocket API URL'
    });

    new cdk.CfnOutput(this, 'NotificationTopicArn', {
      value: notificationTopic.topicArn,
      description: 'SNS Topic for broadcasting'
    });
  }
}
```

### 🔗 Connect Handler

```typescript
// lambda/websocket-connect/index.ts
import { APIGatewayProxyHandler } from 'aws-lambda';
import { DynamoDBClient, PutItemCommand } from '@aws-sdk/client-dynamodb';
import { unmarshall } from '@aws-sdk/util-dynamodb';

const dynamodb = new DynamoDBClient({});
const CONNECTIONS_TABLE = process.env.CONNECTIONS_TABLE!;

export const handler: APIGatewayProxyHandler = async (event) => {
  const connectionId = event.requestContext.connectionId!;
  const queryParams = event.queryStringParameters || {};
  
  console.log(`New connection: ${connectionId}`);

  try {
    // Extract customer ID from query params or auth token
    const customerId = queryParams.customerId || 'anonymous';
    const deviceType = queryParams.deviceType || 'unknown';

    // Store connection in DynamoDB
    await dynamodb.send(
      new PutItemCommand({
        TableName: CONNECTIONS_TABLE,
        Item: {
          connectionId: { S: connectionId },
          customerId: { S: customerId },
          deviceType: { S: deviceType },
          connectedAt: { S: new Date().toISOString() },
          lastActivity: { S: new Date().toISOString() },
          ttl: { N: String(Math.floor(Date.now() / 1000) + 86400) } // 24 hours
        }
      })
    );

    console.log(`Connection stored for customer ${customerId}`);

    return {
      statusCode: 200,
      body: JSON.stringify({ message: 'Connected successfully' })
    };

  } catch (error) {
    console.error('Connection error:', error);
    return {
      statusCode: 500,
      body: JSON.stringify({ message: 'Failed to connect' })
    };
  }
};
```

### ❌ Disconnect Handler

```typescript
// lambda/websocket-disconnect/index.ts
import { APIGatewayProxyHandler } from 'aws-lambda';
import { DynamoDBClient, DeleteItemCommand } from '@aws-sdk/client-dynamodb';

const dynamodb = new DynamoDBClient({});
const CONNECTIONS_TABLE = process.env.CONNECTIONS_TABLE!;

export const handler: APIGatewayProxyHandler = async (event) => {
  const connectionId = event.requestContext.connectionId!;

  console.log(`Disconnecting: ${connectionId}`);

  try {
    await dynamodb.send(
      new DeleteItemCommand({
        TableName: CONNECTIONS_TABLE,
        Key: {
          connectionId: { S: connectionId }
        }
      })
    );

    console.log(`Connection ${connectionId} removed`);

    return {
      statusCode: 200,
      body: 'Disconnected'
    };

  } catch (error) {
    console.error('Disconnect error:', error);
    return {
      statusCode: 500,
      body: 'Failed to disconnect'
    };
  }
};
```

### 💬 Message Handler

```typescript
// lambda/websocket-message/index.ts
import { APIGatewayProxyHandler } from 'aws-lambda';
import { ApiGatewayManagementApiClient, PostToConnectionCommand } from '@aws-sdk/client-apigatewaymanagementapi';
import { DynamoDBClient, UpdateItemCommand } from '@aws-sdk/client-dynamodb';

const dynamodb = new DynamoDBClient({});
const CONNECTIONS_TABLE = process.env.CONNECTIONS_TABLE!;
const WEBSOCKET_API_ENDPOINT = process.env.WEBSOCKET_API_ENDPOINT!;

const apigwManagementApi = new ApiGatewayManagementApiClient({
  endpoint: WEBSOCKET_API_ENDPOINT
});

export const handler: APIGatewayProxyHandler = async (event) => {
  const connectionId = event.requestContext.connectionId!;
  const body = JSON.parse(event.body || '{}');

  console.log(`Message from ${connectionId}:`, body);

  try {
    // Update last activity
    await dynamodb.send(
      new UpdateItemCommand({
        TableName: CONNECTIONS_TABLE,
        Key: { connectionId: { S: connectionId } },
        UpdateExpression: 'SET lastActivity = :timestamp',
        ExpressionAttributeValues: {
          ':timestamp': { S: new Date().toISOString() }
        }
      })
    );

    // Handle different message types
    switch (body.action) {
      case 'ping':
        await sendToConnection(connectionId, { type: 'pong', timestamp: Date.now() });
        break;

      case 'subscribe':
        await handleSubscribe(connectionId, body.channels);
        break;

      case 'unsubscribe':
        await handleUnsubscribe(connectionId, body.channels);
        break;

      default:
        await sendToConnection(connectionId, { 
          type: 'error', 
          message: 'Unknown action' 
        });
    }

    return { statusCode: 200, body: 'Message processed' };

  } catch (error) {
    console.error('Message handling error:', error);
    return { statusCode: 500, body: 'Failed to process message' };
  }
};

async function sendToConnection(connectionId: string, data: any): Promise<void> {
  try {
    await apigwManagementApi.send(
      new PostToConnectionCommand({
        ConnectionId: connectionId,
        Data: Buffer.from(JSON.stringify(data))
      })
    );
  } catch (error: any) {
    if (error.statusCode === 410) {
      console.log(`Stale connection ${connectionId}, removing...`);
      await dynamodb.send(
        new UpdateItemCommand({
          TableName: CONNECTIONS_TABLE,
          Key: { connectionId: { S: connectionId } }
        })
      );
    }
    throw error;
  }
}

async function handleSubscribe(connectionId: string, channels: string[]): Promise<void> {
  await dynamodb.send(
    new UpdateItemCommand({
      TableName: CONNECTIONS_TABLE,
      Key: { connectionId: { S: connectionId } },
      UpdateExpression: 'SET subscriptions = :channels',
      ExpressionAttributeValues: {
        ':channels': { SS: channels }
      }
    })
  );

  await sendToConnection(connectionId, {
    type: 'subscribed',
    channels
  });
}

async function handleUnsubscribe(connectionId: string, channels: string[]): Promise<void> {
  // Remove specific channels
  await sendToConnection(connectionId, {
    type: 'unsubscribed',
    channels
  });
}
```

### 📢 Broadcast Handler

```typescript
// lambda/websocket-broadcast/index.ts
import { SNSHandler } from 'aws-lambda';
import { DynamoDBClient, QueryCommand } from '@aws-sdk/client-dynamodb';
import { ApiGatewayManagementApiClient, PostToConnectionCommand } from '@aws-sdk/client-apigatewaymanagementapi';

const dynamodb = new DynamoDBClient({});
const CONNECTIONS_TABLE = process.env.CONNECTIONS_TABLE!;
const WEBSOCKET_API_ENDPOINT = process.env.WEBSOCKET_API_ENDPOINT!;

const apigwManagementApi = new ApiGatewayManagementApiClient({
  endpoint: WEBSOCKET_API_ENDPOINT
});

export const handler: SNSHandler = async (event) => {
  for (const record of event.Records) {
    const message = JSON.parse(record.Sns.Message);
    
    console.log('Broadcasting message:', message);

    try {
      await broadcastToCustomers(message);
    } catch (error) {
      console.error('Broadcast error:', error);
    }
  }
};

async function broadcastToCustomers(message: any): Promise<void> {
  const { customerId, type, data } = message;

  // Query connections for this customer
  const response = await dynamodb.send(
    new QueryCommand({
      TableName: CONNECTIONS_TABLE,
      IndexName: 'customerIdIndex',
      KeyConditionExpression: 'customerId = :customerId',
      ExpressionAttributeValues: {
        ':customerId': { S: customerId }
      }
    })
  );

  const connections = response.Items || [];
  console.log(`Found ${connections.length} connections for customer ${customerId}`);

  // Send to all connections
  const sendPromises = connections.map(async (conn) => {
    const connectionId = conn.connectionId.S!;

    try {
      await apigwManagementApi.send(
        new PostToConnectionCommand({
          ConnectionId: connectionId,
          Data: Buffer.from(JSON.stringify({ type, data, timestamp: Date.now() }))
        })
      );
      console.log(`Sent to connection ${connectionId}`);
    } catch (error: any) {
      if (error.statusCode === 410) {
        console.log(`Stale connection ${connectionId}, will be cleaned up by TTL`);
      } else {
        console.error(`Failed to send to ${connectionId}:`, error);
      }
    }
  });

  await Promise.allSettled(sendPromises);
}
```

### 📱 Client Example

```typescript
// client/websocket-client.ts
class BankingWebSocketClient {
  private ws: WebSocket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;

  constructor(
    private url: string,
    private customerId: string,
    private onMessage: (data: any) => void
  ) {}

  connect(): void {
    const wsUrl = `${this.url}?customerId=${this.customerId}&deviceType=web`;
    this.ws = new WebSocket(wsUrl);

    this.ws.onopen = () => {
      console.log('WebSocket connected');
      this.reconnectAttempts = 0;

      // Subscribe to channels
      this.send({ action: 'subscribe', channels: ['transactions', 'alerts'] });

      // Start heartbeat
      this.startHeartbeat();
    };

    this.ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      this.onMessage(data);
    };

    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };

    this.ws.onclose = () => {
      console.log('WebSocket closed');
      this.reconnect();
    };
  }

  send(data: any): void {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(data));
    }
  }

  private startHeartbeat(): void {
    setInterval(() => {
      this.send({ action: 'ping' });
    }, 30000); // 30 seconds
  }

  private reconnect(): void {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++;
      const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30000);
      console.log(`Reconnecting in ${delay}ms (attempt ${this.reconnectAttempts})`);
      setTimeout(() => this.connect(), delay);
    }
  }

  disconnect(): void {
    if (this.ws) {
      this.ws.close();
      this.ws = null;
    }
  }
}

// Usage
const client = new BankingWebSocketClient(
  'wss://abc123.execute-api.us-east-1.amazonaws.com/prod',
  'customer-12345',
  (data) => {
    console.log('Received:', data);
    
    if (data.type === 'transaction') {
      showNotification(`New transaction: $${data.data.amount}`);
    } else if (data.type === 'alert') {
      showAlert(data.data.message);
    }
  }
);

client.connect();
```

### 🎓 Interview Discussion Points - Q45

**Q1: How does API Gateway WebSocket work?**

**A**:
- **Persistent connections**: Long-lived bidirectional connections
- **Route selection**: $connect, $disconnect, $default, custom routes
- **Connection ID**: Unique identifier for each connection
- **Callback URL**: Used to send messages to clients
- **Integration**: Lambda, HTTP, AWS services

**Q2: How to handle connection scaling?**

**A**:
- **Auto-scaling**: API Gateway auto-scales (up to 10,000 connections/sec)
- **Connection limits**: 100,000 concurrent connections per account
- **DynamoDB GSI**: Query connections by customer ID
- **Sharding**: Partition connections across multiple tables
- **TTL**: Auto-cleanup of stale connections

**Q3: How to broadcast messages efficiently?**

**A**:
- **DynamoDB Streams**: Trigger Lambda on connection changes
- **SNS Fan-out**: Publish once, multiple Lambda instances process
- **Batch processing**: Send to multiple connections in parallel
- **Connection pooling**: Reuse API Gateway management API clients
- **Error handling**: Handle 410 Gone for stale connections

**Q4: What are WebSocket pricing considerations?**

**A**:
- **Connection minutes**: $0.25 per million minutes
- **Message charges**: $1.00 per million messages (up to 128 KB)
- **Data transfer**: Standard AWS data transfer rates
- **Lambda invocations**: Per route handler invocation
- **DynamoDB costs**: Read/write capacity for connection tracking

**Q5: How to secure WebSocket APIs?**

**A**:
- **IAM authorization**: AWS_IAM authorizer
- **Custom authorizers**: Lambda authorizer for JWT validation
- **Connection validation**: Verify customer ID on connect
- **Rate limiting**: Throttle messages per connection
- **Message validation**: Validate message format and size
- **TLS encryption**: WSS protocol for encrypted connections

---

## Question 46: WebSocket Connection Management & Scaling

### 📋 Question Statement

Implement WebSocket connection management for Emirates NBD including connection tracking in DynamoDB, broadcast messaging, auto-scaling, and connection lifecycle handling (connect, disconnect, idle timeout).

---

### 🔌 WebSocket Connection Manager

```typescript
// services/websocket/connection-manager.ts
import { DynamoDB, ApiGatewayManagementApi } from 'aws-sdk';

export class WebSocketConnectionManager {
  private dynamoDB: DynamoDB.DocumentClient;
  private apiGateway: ApiGatewayManagementApi;

  constructor(endpoint: string) {
    this.dynamoDB = new DynamoDB.DocumentClient();
    this.apiGateway = new ApiGatewayManagementApi({ endpoint });
  }

  async registerConnection(
    connectionId: string,
    customerId: string,
    metadata: ConnectionMetadata
  ): Promise<void> {
    await this.dynamoDB.put({
      TableName: 'WebSocketConnections',
      Item: {
        connectionId,
        customerId,
        connectedAt: new Date().toISOString(),
        lastActivity: new Date().toISOString(),
        ttl: Math.floor(Date.now() / 1000) + (24 * 60 * 60), // 24 hours
        ...metadata
      }
    }).promise();

    console.log(`Connection registered: ${connectionId} for customer ${customerId}`);
  }

  async removeConnection(connectionId: string): Promise<void> {
    await this.dynamoDB.delete({
      TableName: 'WebSocketConnections',
      Key: { connectionId }
    }).promise();

    console.log(`Connection removed: ${connectionId}`);
  }

  async updateLastActivity(connectionId: string): Promise<void> {
    await this.dynamoDB.update({
      TableName: 'WebSocketConnections',
      Key: { connectionId },
      UpdateExpression: 'SET lastActivity = :timestamp',
      ExpressionAttributeValues: {
        ':timestamp': new Date().toISOString()
      }
    }).promise();
  }

  async sendToConnection(
    connectionId: string,
    message: any
  ): Promise<boolean> {
    try {
      await this.apiGateway.postToConnection({
        ConnectionId: connectionId,
        Data: JSON.stringify(message)
      }).promise();

      return true;
    } catch (error: any) {
      if (error.statusCode === 410) {
        // Connection is stale, remove it
        console.log(`Stale connection detected: ${connectionId}`);
        await this.removeConnection(connectionId);
        return false;
      }
      throw error;
    }
  }

  async broadcastToCustomer(
    customerId: string,
    message: any
  ): Promise<BroadcastResult> {
    // Get all connections for this customer
    const connections = await this.dynamoDB.query({
      TableName: 'WebSocketConnections',
      IndexName: 'CustomerIdIndex',
      KeyConditionExpression: 'customerId = :customerId',
      ExpressionAttributeValues: {
        ':customerId': customerId
      }
    }).promise();

    let successCount = 0;
    let failureCount = 0;

    for (const connection of connections.Items || []) {
      const success = await this.sendToConnection(connection.connectionId, message);
      if (success) successCount++;
      else failureCount++;
    }

    return { successCount, failureCount, totalConnections: connections.Items?.length || 0 };
  }

  async broadcastToAll(message: any, filter?: ConnectionFilter): Promise<BroadcastResult> {
    // Scan for all active connections (or use GSI for filtered broadcast)
    let connections;

    if (filter) {
      connections = await this.getFilteredConnections(filter);
    } else {
      connections = await this.dynamoDB.scan({
        TableName: 'WebSocketConnections'
      }).promise();
    }

    let successCount = 0;
    let failureCount = 0;

    // Process in batches of 100 to avoid overwhelming API Gateway
    const batchSize = 100;
    const items = connections.Items || [];

    for (let i = 0; i < items.length; i += batchSize) {
      const batch = items.slice(i, i + batchSize);
      
      await Promise.all(
        batch.map(async (connection) => {
          const success = await this.sendToConnection(connection.connectionId, message);
          if (success) successCount++;
          else failureCount++;
        })
      );
    }

    return { successCount, failureCount, totalConnections: items.length };
  }

  async getConnectionStats(): Promise<ConnectionStats> {
    const result = await this.dynamoDB.scan({
      TableName: 'WebSocketConnections',
      Select: 'COUNT'
    }).promise();

    return {
      totalConnections: result.Count || 0,
      scannedCount: result.ScannedCount || 0
    };
  }

  async cleanupIdleConnections(idleMinutes: number = 30): Promise<number> {
    const cutoffTime = new Date(Date.now() - idleMinutes * 60 * 1000).toISOString();

    const idleConnections = await this.dynamoDB.scan({
      TableName: 'WebSocketConnections',
      FilterExpression: 'lastActivity < :cutoff',
      ExpressionAttributeValues: {
        ':cutoff': cutoffTime
      }
    }).promise();

    let cleanedCount = 0;

    for (const connection of idleConnections.Items || []) {
      await this.removeConnection(connection.connectionId);
      cleanedCount++;
    }

    console.log(`Cleaned up ${cleanedCount} idle connections`);
    return cleanedCount;
  }

  private async getFilteredConnections(filter: ConnectionFilter) {
    // Query with filter criteria (e.g., by accountType, region, etc.)
    return await this.dynamoDB.scan({
      TableName: 'WebSocketConnections',
      FilterExpression: filter.expression,
      ExpressionAttributeValues: filter.values
    }).promise();
  }
}

interface ConnectionMetadata {
  userAgent?: string;
  ipAddress?: string;
  accountType?: string;
  region?: string;
}

interface BroadcastResult {
  successCount: number;
  failureCount: number;
  totalConnections: number;
}

interface ConnectionStats {
  totalConnections: number;
  scannedCount: number;
}

interface ConnectionFilter {
  expression: string;
  values: Record<string, any>;
}
```

### 📊 WebSocket Auto-Scaling Configuration

```typescript
// infrastructure/cdk/websocket-autoscaling-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as applicationautoscaling from 'aws-cdk-lib/aws-applicationautoscaling';
import * as cloudwatch from 'aws-cdk-lib/aws-cloudwatch';

export class WebSocketAutoScalingStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // DynamoDB table for connections
    const connectionsTable = new dynamodb.Table(this, 'WebSocketConnections', {
      partitionKey: { name: 'connectionId', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST, // On-demand for variable load
      timeToLiveAttribute: 'ttl',
      stream: dynamodb.StreamViewType.NEW_AND_OLD_IMAGES,
      pointInTimeRecovery: true
    });

    // GSI for customer ID lookups
    connectionsTable.addGlobalSecondaryIndex({
      indexName: 'CustomerIdIndex',
      partitionKey: { name: 'customerId', type: dynamodb.AttributeType.STRING },
      projectionType: dynamodb.ProjectionType.ALL
    });

    // CloudWatch dashboard
    const dashboard = new cloudwatch.Dashboard(this, 'WebSocketDashboard', {
      dashboardName: 'Banking-WebSocket-Metrics'
    });

    dashboard.addWidgets(
      new cloudwatch.GraphWidget({
        title: 'Active Connections',
        left: [
          new cloudwatch.Metric({
            namespace: 'AWS/ApiGateway',
            metricName: 'ConnectionCount',
            statistic: 'Sum',
            period: cdk.Duration.minutes(1)
          })
        ]
      }),
      new cloudwatch.GraphWidget({
        title: 'Message Rate',
        left: [
          new cloudwatch.Metric({
            namespace: 'AWS/ApiGateway',
            metricName: 'MessageCount',
            statistic: 'Sum',
            period: cdk.Duration.minutes(1)
          })
        ]
      })
    );

    // Alarm for high connection count
    const highConnectionAlarm = new cloudwatch.Alarm(this, 'HighConnectionCount', {
      metric: new cloudwatch.Metric({
        namespace: 'AWS/ApiGateway',
        metricName: 'ConnectionCount',
        statistic: 'Maximum',
        period: cdk.Duration.minutes(5)
      }),
      threshold: 10000,
      evaluationPeriods: 2,
      alarmDescription: 'Alert when WebSocket connections exceed threshold'
    });
  }
}
```

### 🎓 Interview Discussion Points - Q46

**Q1: How to handle WebSocket connection limits?**

**A**:
- **API Gateway limit**: 10,000 connections per route
- **Scale horizontally**: Multiple API Gateway instances
- **Connection pooling**: Group users by region/shard
- **Fallback**: Long polling for overflow

**Q2: Why store connections in DynamoDB?**

**A**:
- **Persistence**: Survive Lambda cold starts
- **Broadcast**: Query all connections for customer
- **Analytics**: Track connection patterns
- **TTL**: Auto-cleanup stale connections

**Q3: How to implement heartbeat/ping-pong?**

**A**:
```typescript
// Client sends ping every 30s
setInterval(() => {
  ws.send(JSON.stringify({ action: 'ping' }));
}, 30000);

// Server responds with pong and updates lastActivity
```

**Q4: What is the cost model for WebSocket APIs?**

**A**:
- **Connection**: $0.25 per million connection minutes
- **Messages**: $1.00 per million messages
- **Data transfer**: Standard rates
- **Example**: 1000 users, 1 hour = $0.25

**Q5: How to broadcast to 100,000+ users?**

**A**:
- **Fan-out with SQS/SNS**: Distribute work across Lambdas
- **Batch processing**: 100 connections per Lambda invocation
- **Async**: Don't wait for all sends to complete
- **Dead letter queue**: Handle failed deliveries
- **Sharding**: Partition connections across multiple endpoints

---

**End of File 23**

