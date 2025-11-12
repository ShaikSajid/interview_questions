# Legacy System Integration

## Question 55: Strangler Fig Pattern for Legacy Migration

### 📋 Question Statement

Implement legacy system integration for Emirates NBD using Strangler Fig pattern, anti-corruption layers, and message translation for gradual modernization.

---

### 🌳 Strangler Fig Pattern Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    STRANGLER FIG MIGRATION PATTERN                          │
└────────────────────────────────────────────────────────────────────────────┘

                        PHASE 1: Initial           PHASE 2: Partial         PHASE 3: Complete
                        ─────────────              ──────────────          ─────────────

    ┌──────────┐        ┌──────────┐              ┌──────────┐            ┌──────────┐
    │  Client  │        │  Client  │              │  Client  │            │  Client  │
    └────┬─────┘        └────┬─────┘              └────┬─────┘            └────┬─────┘
         │                   │                          │                       │
         v                   v                          v                       v
    ┌─────────┐        ┌──────────┐             ┌──────────┐            ┌──────────┐
    │ Legacy  │        │  Facade  │             │  Facade  │            │   New    │
    │ System  │        │  Router  │             │  Router  │            │  System  │
    └─────────┘        └────┬─────┘             └────┬─────┘            └──────────┘
                            │                         │
                   ┌────────┼────────┐       ┌────────┼────────┐
                   │        │        │       │        │        │
                   v        v        v       v        v        v
              ┌────────┐┌────────┐    ┌────────┐┌────────┐
              │ Legacy ││  New   │    │ Legacy ││  New   │
              │System  ││System  │    │(20%)   ││(80%)   │
              │(100%)  ││ (0%)   │    └────────┘└────────┘
              └────────┘└────────┘
```

### 🔧 API Facade Router

```typescript
// services/facade/api-router.ts
import express, { Request, Response, NextFunction } from 'express';
import { getFeatureFlags } from '../common/feature-flags';
import axios from 'axios';

export class StranglerFacadeRouter {
  private router = express.Router();
  private legacyBaseUrl: string;
  private modernBaseUrl: string;

  constructor(legacyBaseUrl: string, modernBaseUrl: string) {
    this.legacyBaseUrl = legacyBaseUrl;
    this.modernBaseUrl = modernBaseUrl;
    this.setupRoutes();
  }

  private setupRoutes(): void {
    // Account routes
    this.router.get('/accounts/:id', this.routeAccountGet.bind(this));
    this.router.post('/accounts', this.routeAccountCreate.bind(this));
    this.router.put('/accounts/:id', this.routeAccountUpdate.bind(this));

    // Transaction routes
    this.router.get('/transactions/:id', this.routeTransactionGet.bind(this));
    this.router.post('/transactions', this.routeTransactionCreate.bind(this));

    // Payment routes
    this.router.post('/payments', this.routePaymentCreate.bind(this));
  }

  private async routeAccountGet(req: Request, res: Response): Promise<void> {
    const accountId = req.params.id;
    const featureFlags = getFeatureFlags();

    // Determine routing based on feature flag
    const useModernSystem = await featureFlags.isEnabled('modernAccountService', {
      accountId,
      userId: req.user?.id
    });

    try {
      if (useModernSystem) {
        // Route to modern microservice
        const response = await axios.get(`${this.modernBaseUrl}/api/v1/accounts/${accountId}`, {
          headers: this.forwardHeaders(req)
        });
        res.json(this.transformModernResponse(response.data));
      } else {
        // Route to legacy system
        const response = await axios.get(`${this.legacyBaseUrl}/legacy/getAccount`, {
          params: { accountId },
          headers: this.forwardHeaders(req)
        });
        res.json(this.transformLegacyResponse(response.data));
      }
    } catch (error) {
      this.handleError(error, res);
    }
  }

  private async routeAccountCreate(req: Request, res: Response): Promise<void> {
    const featureFlags = getFeatureFlags();

    const useModernSystem = await featureFlags.isEnabled('modernAccountService', {
      userId: req.user?.id
    });

    try {
      if (useModernSystem) {
        // Create in modern system
        const response = await axios.post(
          `${this.modernBaseUrl}/api/v1/accounts`,
          req.body,
          { headers: this.forwardHeaders(req) }
        );

        // Sync to legacy (dual write during migration)
        await this.syncToLegacy('createAccount', req.body);

        res.status(201).json(this.transformModernResponse(response.data));
      } else {
        // Create in legacy system
        const response = await axios.post(
          `${this.legacyBaseUrl}/legacy/createAccount`,
          this.transformToLegacyFormat(req.body),
          { headers: this.forwardHeaders(req) }
        );

        res.status(201).json(this.transformLegacyResponse(response.data));
      }
    } catch (error) {
      this.handleError(error, res);
    }
  }

  private async routeAccountUpdate(req: Request, res: Response): Promise<void> {
    const accountId = req.params.id;
    const featureFlags = getFeatureFlags();

    const useModernSystem = await featureFlags.isEnabled('modernAccountService', {
      accountId,
      userId: req.user?.id
    });

    try {
      if (useModernSystem) {
        const response = await axios.put(
          `${this.modernBaseUrl}/api/v1/accounts/${accountId}`,
          req.body,
          { headers: this.forwardHeaders(req) }
        );

        // Sync to legacy
        await this.syncToLegacy('updateAccount', { accountId, ...req.body });

        res.json(this.transformModernResponse(response.data));
      } else {
        const response = await axios.put(
          `${this.legacyBaseUrl}/legacy/updateAccount`,
          this.transformToLegacyFormat({ accountId, ...req.body }),
          { headers: this.forwardHeaders(req) }
        );

        res.json(this.transformLegacyResponse(response.data));
      }
    } catch (error) {
      this.handleError(error, res);
    }
  }

  private async routeTransactionGet(req: Request, res: Response): Promise<void> {
    // Similar routing logic for transactions
    res.json({ message: 'Transaction routing not yet implemented' });
  }

  private async routeTransactionCreate(req: Request, res: Response): Promise<void> {
    // Always use modern system for new transactions
    try {
      const response = await axios.post(
        `${this.modernBaseUrl}/api/v1/transactions`,
        req.body,
        { headers: this.forwardHeaders(req) }
      );

      res.status(201).json(response.data);
    } catch (error) {
      this.handleError(error, res);
    }
  }

  private async routePaymentCreate(req: Request, res: Response): Promise<void> {
    // Payments always use modern system
    try {
      const response = await axios.post(
        `${this.modernBaseUrl}/api/v1/payments`,
        req.body,
        { headers: this.forwardHeaders(req) }
      );

      res.status(201).json(response.data);
    } catch (error) {
      this.handleError(error, res);
    }
  }

  private forwardHeaders(req: Request): Record<string, string> {
    return {
      'Authorization': req.headers.authorization || '',
      'X-Request-ID': req.headers['x-request-id'] as string || '',
      'Content-Type': 'application/json'
    };
  }

  private transformModernResponse(data: any): any {
    // Modern system already returns correct format
    return data;
  }

  private transformLegacyResponse(data: any): any {
    // Transform legacy XML/SOAP response to REST JSON
    return {
      accountId: data.AccountId || data.account_id,
      accountNumber: data.AccountNumber || data.account_number,
      balance: parseFloat(data.Balance || data.balance || '0'),
      currency: data.Currency || data.currency || 'AED',
      status: data.Status || data.status,
      createdAt: data.CreatedDate || data.created_at
    };
  }

  private transformToLegacyFormat(data: any): any {
    // Transform REST JSON to legacy format
    return {
      AccountId: data.accountId,
      AccountNumber: data.accountNumber,
      Balance: data.balance?.toString(),
      Currency: data.currency,
      Status: data.status
    };
  }

  private async syncToLegacy(operation: string, data: any): Promise<void> {
    try {
      // Async sync to legacy system (fire and forget with retry)
      await axios.post(
        `${this.legacyBaseUrl}/legacy/${operation}`,
        this.transformToLegacyFormat(data),
        { timeout: 5000 }
      );
      console.log(`Synced to legacy: ${operation}`);
    } catch (error) {
      console.error(`Failed to sync to legacy: ${operation}`, error);
      // Queue for retry (publish to SQS)
    }
  }

  private handleError(error: any, res: Response): void {
    console.error('Routing error:', error);

    if (error.response) {
      res.status(error.response.status).json({
        error: error.response.data
      });
    } else {
      res.status(500).json({
        error: 'Internal server error'
      });
    }
  }

  getRouter(): express.Router {
    return this.router;
  }
}
```

### 🛡️ Anti-Corruption Layer

```typescript
// services/legacy-integration/anti-corruption-layer.ts
export class AntiCorruptionLayer {
  
  // Translate legacy account model to domain model
  translateAccount(legacyAccount: LegacyAccount): DomainAccount {
    return {
      id: legacyAccount.AccountId,
      accountNumber: this.formatAccountNumber(legacyAccount.AccountNumber),
      type: this.mapAccountType(legacyAccount.AccountType),
      balance: {
        amount: parseFloat(legacyAccount.Balance),
        currency: legacyAccount.Currency || 'AED'
      },
      status: this.mapAccountStatus(legacyAccount.Status),
      customer: {
        id: legacyAccount.CustomerId,
        name: legacyAccount.CustomerName
      },
      createdAt: new Date(legacyAccount.CreatedDate),
      updatedAt: new Date(legacyAccount.LastModifiedDate)
    };
  }

  // Translate domain model to legacy model
  translateToLegacy(domainAccount: DomainAccount): LegacyAccount {
    return {
      AccountId: domainAccount.id,
      AccountNumber: domainAccount.accountNumber,
      AccountType: this.reversemapAccountType(domainAccount.type),
      Balance: domainAccount.balance.amount.toFixed(2),
      Currency: domainAccount.balance.currency,
      Status: this.reverseMapAccountStatus(domainAccount.status),
      CustomerId: domainAccount.customer.id,
      CustomerName: domainAccount.customer.name,
      CreatedDate: domainAccount.createdAt.toISOString(),
      LastModifiedDate: domainAccount.updatedAt.toISOString()
    };
  }

  private formatAccountNumber(accountNumber: string): string {
    // Legacy: 1234567890
    // Modern: AE07-1234-5678-90
    return `AE07-${accountNumber.slice(0, 4)}-${accountNumber.slice(4, 8)}-${accountNumber.slice(8)}`;
  }

  private mapAccountType(legacyType: string): string {
    const typeMap: Record<string, string> = {
      'SAV': 'SAVINGS',
      'CUR': 'CURRENT',
      'FD': 'FIXED_DEPOSIT',
      'CC': 'CREDIT_CARD'
    };
    return typeMap[legacyType] || 'UNKNOWN';
  }

  private reversemapAccountType(modernType: string): string {
    const reverseMap: Record<string, string> = {
      'SAVINGS': 'SAV',
      'CURRENT': 'CUR',
      'FIXED_DEPOSIT': 'FD',
      'CREDIT_CARD': 'CC'
    };
    return reverseMap[modernType] || 'SAV';
  }

  private mapAccountStatus(legacyStatus: string): string {
    const statusMap: Record<string, string> = {
      'A': 'ACTIVE',
      'I': 'INACTIVE',
      'C': 'CLOSED',
      'B': 'BLOCKED'
    };
    return statusMap[legacyStatus] || 'UNKNOWN';
  }

  private reverseMapAccountStatus(modernStatus: string): string {
    const reverseMap: Record<string, string> = {
      'ACTIVE': 'A',
      'INACTIVE': 'I',
      'CLOSED': 'C',
      'BLOCKED': 'B'
    };
    return reverseMap[modernStatus] || 'A';
  }
}

interface LegacyAccount {
  AccountId: string;
  AccountNumber: string;
  AccountType: string;
  Balance: string;
  Currency: string;
  Status: string;
  CustomerId: string;
  CustomerName: string;
  CreatedDate: string;
  LastModifiedDate: string;
}

interface DomainAccount {
  id: string;
  accountNumber: string;
  type: string;
  balance: {
    amount: number;
    currency: string;
  };
  status: string;
  customer: {
    id: string;
    name: string;
  };
  createdAt: Date;
  updatedAt: Date;
}
```

### 📊 Migration Progress Tracking

```typescript
// services/migration/migration-tracker.ts
import { DynamoDBClient, PutItemCommand, QueryCommand } from '@aws-sdk/client-dynamodb';

export class MigrationProgressTracker {
  private dynamodb: DynamoDBClient;
  private tableName = 'migration-progress';

  constructor() {
    this.dynamodb = new DynamoDBClient({});
  }

  async trackMigration(entityType: string, entityId: string, status: MigrationStatus): Promise<void> {
    await this.dynamodb.send(
      new PutItemCommand({
        TableName: this.tableName,
        Item: {
          entityType: { S: entityType },
          entityId: { S: entityId },
          status: { S: status },
          migratedAt: { S: new Date().toISOString() },
          system: { S: status === 'MIGRATED' ? 'MODERN' : 'LEGACY' }
        }
      })
    );
  }

  async getMigrationStats(entityType: string): Promise<MigrationStats> {
    const response = await this.dynamodb.send(
      new QueryCommand({
        TableName: this.tableName,
        KeyConditionExpression: 'entityType = :type',
        ExpressionAttributeValues: {
          ':type': { S: entityType }
        }
      })
    );

    const items = response.Items || [];
    const total = items.length;
    const migrated = items.filter(i => i.status.S === 'MIGRATED').length;
    const pending = total - migrated;

    return {
      entityType,
      total,
      migrated,
      pending,
      percentComplete: (migrated / total * 100).toFixed(2) + '%'
    };
  }
}

type MigrationStatus = 'PENDING' | 'IN_PROGRESS' | 'MIGRATED' | 'FAILED';

interface MigrationStats {
  entityType: string;
  total: number;
  migrated: number;
  pending: number;
  percentComplete: string;
}
```

### 🎓 Interview Discussion Points - Q55

**Q1: What is the Strangler Fig Pattern?**

**A**:
- **Definition**: Gradually replace legacy system by routing new features to modern system
- **Phases**:
  1. Build facade/router
  2. Route some requests to new system
  3. Incrementally migrate features
  4. Decommission legacy once fully migrated
- **Benefits**: Low risk, continuous delivery, reversible

**Q2: How to handle data synchronization during migration?**

**A**:
- **Dual write**: Write to both systems during transition
- **Event sourcing**: Replay events to sync data
- **CDC (Change Data Capture)**: Track changes in legacy DB
- **Reconciliation**: Periodic jobs to compare and fix discrepancies
- **Eventually consistent**: Accept temporary inconsistency

**Q3: What is an Anti-Corruption Layer (ACL)?**

**A**:
- **Purpose**: Prevent legacy concepts from polluting domain model
- **Functions**:
  - Translate between models
  - Adapt protocols (SOAP → REST)
  - Isolate domain from legacy changes
- **Location**: Between legacy and modern systems

**Q4: How to route traffic during migration?**

**A**:
- **Feature flags**: Enable modern system per user/feature
- **Percentage-based**: Route X% to new system
- **Canary**: Start with 1-5%, increase gradually
- **User segments**: Beta testers first, then wider rollout
- **Shadow mode**: Route to both, compare results

**Q5: How to handle rollback during migration?**

**A**:
- **Feature flags**: Instantly switch back to legacy
- **Data sync**: Keep legacy system updated
- **Monitoring**: Track error rates, latency
- **Graceful degradation**: Fall back to legacy on errors
- **Runbook**: Document rollback procedures

---

## Question 56: API Translation Layer & Protocol Adapters

### 📋 Question Statement

Implement API translation layer for Emirates NBD to integrate legacy SOAP/XML systems with modern REST/JSON microservices using protocol adapters, message transformation, and data format conversion.

---

### 🔄 Protocol Adapter Service

```typescript
// services/legacy-integration/protocol-adapter.ts
import * as soap from 'soap';
import { XMLParser, XMLBuilder } from 'fast-xml-parser';
import axios from 'axios';

export class ProtocolAdapter {
  private xmlParser: XMLParser;
  private xmlBuilder: XMLBuilder;

  constructor() {
    this.xmlParser = new XMLParser({
      ignoreAttributes: false,
      attributeNamePrefix: '@_'
    });
    this.xmlBuilder = new XMLBuilder({
      ignoreAttributes: false,
      attributeNamePrefix: '@_'
    });
  }

  async restToSoap(
    restRequest: RestRequest,
    soapEndpoint: string,
    soapAction: string
  ): Promise<RestResponse> {
    // Convert REST JSON to SOAP XML
    const soapRequest = this.convertRestToSoap(restRequest);

    // Call legacy SOAP service
    const client = await soap.createClientAsync(soapEndpoint);
    const result = await client[soapAction + 'Async'](soapRequest);

    // Convert SOAP XML response to REST JSON
    return this.convertSoapToRest(result);
  }

  async soapToRest(
    soapXml: string,
    restEndpoint: string
  ): Promise<string> {
    // Parse SOAP XML
    const soapData = this.xmlParser.parse(soapXml);
    const body = soapData['soap:Envelope']['soap:Body'];

    // Extract business data
    const businessData = this.extractBusinessData(body);

    // Convert to REST JSON
    const restPayload = this.transformToRest(businessData);

    // Call modern REST API
    const response = await axios.post(restEndpoint, restPayload, {
      headers: { 'Content-Type': 'application/json' }
    });

    // Convert response back to SOAP XML
    return this.convertRestToSoapXml(response.data);
  }

  private convertRestToSoap(restRequest: RestRequest): any {
    // Map REST fields to SOAP structure
    return {
      accountNumber: restRequest.accountId,
      transactionAmount: restRequest.amount,
      transactionType: this.mapTransactionType(restRequest.type),
      merchantName: restRequest.merchant || '',
      transactionDate: new Date().toISOString()
    };
  }

  private convertSoapToRest(soapResponse: any): RestResponse {
    // Map SOAP response to REST format
    return {
      transactionId: soapResponse.transactionId,
      status: this.mapStatus(soapResponse.statusCode),
      timestamp: soapResponse.timestamp,
      confirmationNumber: soapResponse.confirmationId
    };
  }

  private convertRestToSoapXml(restData: any): string {
    const soapEnvelope = {
      'soap:Envelope': {
        '@_xmlns:soap': 'http://schemas.xmlsoap.org/soap/envelope/',
        'soap:Body': {
          TransactionResponse: {
            TransactionId: restData.transactionId,
            StatusCode: restData.status === 'SUCCESS' ? '0' : '1',
            Timestamp: restData.timestamp,
            Message: restData.message || 'Transaction processed'
          }
        }
      }
    };

    return this.xmlBuilder.build(soapEnvelope);
  }

  private extractBusinessData(soapBody: any): any {
    // Extract relevant business data from SOAP structure
    const request = soapBody.ProcessTransactionRequest || soapBody.TransactionRequest;
    return request;
  }

  private transformToRest(businessData: any): any {
    // Transform to modern REST format
    return {
      accountId: businessData.AccountNumber || businessData.accountNumber,
      amount: parseFloat(businessData.Amount || businessData.transactionAmount),
      type: this.mapTransactionTypeToRest(businessData.Type || businessData.transactionType),
      merchant: businessData.MerchantName || businessData.merchantName,
      metadata: {
        legacySystem: true,
        originalFormat: 'SOAP'
      }
    };
  }

  private mapTransactionType(restType: string): string {
    const mapping: Record<string, string> = {
      'TRANSFER': 'TXN_TRANSFER',
      'PAYMENT': 'TXN_PAYMENT',
      'WITHDRAWAL': 'TXN_WITHDRAWAL',
      'DEPOSIT': 'TXN_DEPOSIT'
    };
    return mapping[restType] || 'TXN_UNKNOWN';
  }

  private mapTransactionTypeToRest(soapType: string): string {
    const mapping: Record<string, string> = {
      'TXN_TRANSFER': 'TRANSFER',
      'TXN_PAYMENT': 'PAYMENT',
      'TXN_WITHDRAWAL': 'WITHDRAWAL',
      'TXN_DEPOSIT': 'DEPOSIT'
    };
    return mapping[soapType] || 'UNKNOWN';
  }

  private mapStatus(statusCode: string): string {
    const mapping: Record<string, string> = {
      '0': 'SUCCESS',
      '1': 'FAILED',
      '2': 'PENDING',
      '3': 'CANCELLED'
    };
    return mapping[statusCode] || 'UNKNOWN';
  }
}

interface RestRequest {
  accountId: string;
  amount: number;
  type: string;
  merchant?: string;
}

interface RestResponse {
  transactionId: string;
  status: string;
  timestamp: string;
  confirmationNumber: string;
}
```

### 🛡️ Anti-Corruption Layer

```typescript
// services/legacy-integration/anti-corruption-layer.ts
export class AntiCorruptionLayer {
  private protocolAdapter: ProtocolAdapter;

  constructor() {
    this.protocolAdapter = new ProtocolAdapter();
  }

  async createTransaction(
    modernRequest: ModernTransactionRequest
  ): Promise<ModernTransactionResponse> {
    // Translate to legacy format
    const legacyRequest = this.toLegacyFormat(modernRequest);

    // Call legacy system via protocol adapter
    const legacyResponse = await this.protocolAdapter.restToSoap(
      legacyRequest,
      process.env.LEGACY_SOAP_ENDPOINT!,
      'ProcessTransaction'
    );

    // Translate back to modern format
    return this.toModernFormat(legacyResponse);
  }

  private toLegacyFormat(modern: ModernTransactionRequest): any {
    // Map modern domain model to legacy format
    return {
      accountId: modern.account.id,
      amount: modern.transaction.amount,
      type: modern.transaction.type,
      merchant: modern.transaction.merchant,
      customerId: modern.account.customerId,
      // Legacy system requires these fields
      branchCode: '001', // Default branch
      tellerCode: 'SYSTEM', // Automated transaction
      referenceNumber: this.generateLegacyReference()
    };
  }

  private toModernFormat(legacy: any): ModernTransactionResponse {
    // Map legacy response to modern domain model
    return {
      transaction: {
        id: legacy.transactionId,
        status: this.mapLegacyStatus(legacy.status),
        timestamp: new Date(legacy.timestamp),
        confirmationCode: legacy.confirmationNumber
      },
      account: {
        balanceAfter: legacy.newBalance || 0
      }
    };
  }

  private mapLegacyStatus(legacyStatus: string): TransactionStatus {
    const mapping: Record<string, TransactionStatus> = {
      'SUCCESS': TransactionStatus.COMPLETED,
      'FAILED': TransactionStatus.FAILED,
      'PENDING': TransactionStatus.PENDING
    };
    return mapping[legacyStatus] || TransactionStatus.UNKNOWN;
  }

  private generateLegacyReference(): string {
    // Legacy system expects specific format: YYYYMMDD-NNNNNN
    const date = new Date().toISOString().slice(0, 10).replace(/-/g, '');
    const random = Math.floor(Math.random() * 1000000).toString().padStart(6, '0');
    return `${date}-${random}`;
  }
}

interface ModernTransactionRequest {
  account: {
    id: string;
    customerId: string;
  };
  transaction: {
    amount: number;
    type: string;
    merchant?: string;
  };
}

interface ModernTransactionResponse {
  transaction: {
    id: string;
    status: TransactionStatus;
    timestamp: Date;
    confirmationCode: string;
  };
  account: {
    balanceAfter: number;
  };
}

enum TransactionStatus {
  COMPLETED = 'COMPLETED',
  PENDING = 'PENDING',
  FAILED = 'FAILED',
  UNKNOWN = 'UNKNOWN'
}
```

### 🔌 Message Queue Integration

```typescript
// services/legacy-integration/message-bridge.ts
import { SQS } from 'aws-sdk';
import * as amqp from 'amqplib';

export class MessageBridge {
  private sqs: SQS;

  constructor() {
    this.sqs = new SQS();
  }

  async bridgeLegacyMQToSQS(
    mqHost: string,
    mqQueue: string,
    sqsQueueUrl: string
  ): Promise<void> {
    // Connect to legacy IBM MQ / RabbitMQ
    const connection = await amqp.connect(mqHost);
    const channel = await connection.createChannel();

    await channel.assertQueue(mqQueue);

    console.log(`Bridging ${mqQueue} -> ${sqsQueueUrl}`);

    // Consume from legacy MQ
    channel.consume(mqQueue, async (msg) => {
      if (msg) {
        const legacyMessage = msg.content.toString();

        // Transform message format
        const modernMessage = this.transformMessage(legacyMessage);

        // Send to SQS
        await this.sqs.sendMessage({
          QueueUrl: sqsQueueUrl,
          MessageBody: JSON.stringify(modernMessage),
          MessageAttributes: {
            Source: { DataType: 'String', StringValue: 'LegacyMQ' },
            TransformedAt: { DataType: 'String', StringValue: new Date().toISOString() }
          }
        }).promise();

        // Acknowledge message
        channel.ack(msg);
        console.log('Message bridged successfully');
      }
    });
  }

  private transformMessage(legacyXml: string): any {
    // Parse legacy XML message and convert to JSON
    const parser = new XMLParser();
    const parsed = parser.parse(legacyXml);

    return {
      type: parsed.MessageType,
      payload: parsed.Payload,
      timestamp: new Date().toISOString()
    };
  }
}
```

### 🎓 Interview Discussion Points - Q56

**Q1: What is an anti-corruption layer?**

**A**:
- **Purpose**: Isolate modern system from legacy complexity
- **Translation**: Convert between domain models
- **Protection**: Prevent legacy concepts from polluting modern code
- **Example**: Legacy uses branch codes, modern doesn't

**Q2: How to handle data format mismatches?**

**A**:
- **JSON ↔ XML**: Use parsers (fast-xml-parser)
- **Field mapping**: accountNumber → accountId
- **Date formats**: ISO 8601 vs legacy formats
- **Default values**: Provide required legacy fields

**Q3: What are common legacy integration patterns?**

**A**:
- **Protocol adapter**: REST ↔ SOAP translation
- **Message bridge**: SQS ↔ IBM MQ
- **Database sync**: CDC from legacy DB to modern
- **API gateway**: Facade for legacy services

**Q4: How to version API translations?**

**A**:
```typescript
if (legacyVersion === 'v1') {
  return translateV1ToModern(data);
} else if (legacyVersion === 'v2') {
  return translateV2ToModern(data);
}
```
**Support multiple legacy versions simultaneously**

**Q5: How to test legacy integrations?**

**A**:
- **Mocking**: Mock SOAP responses
- **Contract testing**: Verify request/response schemas
- **Integration tests**: Test against legacy sandbox
- **Monitoring**: Track translation errors

---

**End of File 28**

