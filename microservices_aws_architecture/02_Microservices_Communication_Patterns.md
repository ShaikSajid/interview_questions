# Microservices Communication Patterns

## Question 3: Service Decomposition Strategies

### 📋 Question Statement

Emirates NBD wants to decompose their monolithic banking application into microservices. Design multiple service decomposition strategies and explain when to use each approach. Include:
- Decomposition by business capability
- Decomposition by subdomain (DDD)
- Decomposition by technical capability
- Self-contained service pattern
- Service per team pattern

Provide a complete banking example showing how to identify bounded contexts and service boundaries.

---

### 🎯 Decomposition Strategy Overview

```
MONOLITH → MICROSERVICES: Multiple Strategies

Strategy 1: Business Capability
┌─────────────────────────────────────────┐
│  Banking Business Capabilities          │
├─────────────────────────────────────────┤
│ • Account Management                    │
│ • Transaction Processing                │
│ • Payment Handling                      │
│ • Loan Management                       │
│ • Customer Service                      │
│ • Compliance & Reporting                │
└─────────────────────────────────────────┘
         ↓
Each capability = One microservice

Strategy 2: DDD Subdomain
┌─────────────────────────────────────────┐
│  Domain Model Analysis                  │
├─────────────────────────────────────────┤
│ Core Domain: Transaction Engine         │
│ Supporting: Customer Profile            │
│ Generic: Notification, Authentication   │
└─────────────────────────────────────────┘
         ↓
Each subdomain = One microservice

Strategy 3: Data Ownership
┌─────────────────────────────────────────┐
│  Database Tables & Relationships        │
├─────────────────────────────────────────┤
│ accounts, customers → Account Service   │
│ transactions, ledger → Transaction Svc  │
│ loans, collateral → Loan Service        │
└─────────────────────────────────────────┘
         ↓
Tables with strong cohesion = One service
```

---

### 📊 Strategy 1: Decomposition by Business Capability

**Approach**: Identify business capabilities from organizational perspective.

#### **Step 1: Business Capability Mapping**

```javascript
// business-capability-analyzer.js
/**
 * Analyzes business capabilities for Emirates NBD Digital Banking
 */
class BusinessCapabilityAnalyzer {
  identifyCapabilities() {
    return {
      // Core Banking Capabilities
      coreCapabilities: [
        {
          name: 'Account Management',
          description: 'Create, update, close accounts',
          complexity: 'HIGH',
          changeFrequency: 'MEDIUM',
          team: 'Account Team',
          services: ['AccountService', 'AccountValidationService']
        },
        {
          name: 'Transaction Processing',
          description: 'Process deposits, withdrawals, transfers',
          complexity: 'VERY_HIGH',
          changeFrequency: 'HIGH',
          team: 'Transaction Team',
          services: ['TransactionService', 'LedgerService']
        },
        {
          name: 'Payment Gateway',
          description: 'Process card payments, online payments',
          complexity: 'HIGH',
          changeFrequency: 'HIGH',
          team: 'Payment Team',
          services: ['PaymentService', 'PaymentGatewayService']
        }
      ],

      // Supporting Capabilities
      supportingCapabilities: [
        {
          name: 'Customer Profile',
          description: 'Manage customer information',
          complexity: 'MEDIUM',
          changeFrequency: 'LOW',
          team: 'Customer Team',
          services: ['CustomerService']
        },
        {
          name: 'Notification',
          description: 'Send SMS, email, push notifications',
          complexity: 'LOW',
          changeFrequency: 'MEDIUM',
          team: 'Platform Team',
          services: ['NotificationService']
        }
      ],

      // Generic Capabilities
      genericCapabilities: [
        {
          name: 'Authentication & Authorization',
          description: 'User login, OAuth, JWT',
          complexity: 'MEDIUM',
          changeFrequency: 'LOW',
          team: 'Security Team',
          services: ['AuthService', 'IAMService']
        },
        {
          name: 'Audit & Compliance',
          description: 'Regulatory reporting, audit logs',
          complexity: 'HIGH',
          changeFrequency: 'MEDIUM',
          team: 'Compliance Team',
          services: ['AuditService', 'ComplianceService']
        }
      ]
    };
  }

  // Analyze capability dependencies
  analyzeCapabilityDependencies() {
    return {
      'Account Management': {
        dependsOn: ['Customer Profile', 'Authentication'],
        usedBy: ['Transaction Processing', 'Loan Management']
      },
      'Transaction Processing': {
        dependsOn: ['Account Management', 'Ledger', 'Fraud Detection'],
        usedBy: ['Reporting', 'Analytics']
      },
      'Payment Gateway': {
        dependsOn: ['Account Management', 'Transaction Processing'],
        usedBy: ['E-commerce', 'Bill Payment']
      }
    };
  }

  // Calculate service size metrics
  calculateServiceMetrics(capability) {
    return {
      estimatedAPIs: this.estimateAPICount(capability),
      estimatedTeamSize: this.estimateTeamSize(capability.complexity),
      estimatedLinesOfCode: this.estimateLOC(capability.complexity),
      databaseTables: this.estimateTables(capability)
    };
  }

  estimateAPICount(capability) {
    const complexityMultiplier = {
      'LOW': 3,
      'MEDIUM': 5,
      'HIGH': 8,
      'VERY_HIGH': 12
    };
    return complexityMultiplier[capability.complexity] || 5;
  }

  estimateTeamSize(complexity) {
    const teamSize = {
      'LOW': '2-3 developers',
      'MEDIUM': '3-5 developers',
      'HIGH': '5-7 developers',
      'VERY_HIGH': '7-10 developers'
    };
    return teamSize[complexity] || '3-5 developers';
  }

  estimateLOC(complexity) {
    const loc = {
      'LOW': '5,000-10,000',
      'MEDIUM': '10,000-20,000',
      'HIGH': '20,000-40,000',
      'VERY_HIGH': '40,000-80,000'
    };
    return loc[complexity] || '10,000-20,000';
  }

  estimateTables(capability) {
    return Math.ceil(capability.complexity === 'VERY_HIGH' ? 8 : 
                     capability.complexity === 'HIGH' ? 5 : 3);
  }
}

// Usage
const analyzer = new BusinessCapabilityAnalyzer();
const capabilities = analyzer.identifyCapabilities();
console.log(JSON.stringify(capabilities, null, 2));
```

**Output Architecture**:

```
Emirates NBD Banking Platform - Business Capability Decomposition

┌─────────────────────────────────────────────────────────────┐
│                    API Gateway / BFF Layer                   │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
┌───────▼────────┐  ┌──────▼───────┐  ┌───────▼────────┐
│ Account Service│  │ Transaction  │  │ Payment Service│
│  (ECS Fargate) │  │   Service    │  │   (EKS/K8s)    │
│                │  │  (Lambda)    │  │                │
│ - Create Acct  │  │ - Deposit    │  │ - Process Pay  │
│ - Update Acct  │  │ - Withdraw   │  │ - Card Payment │
│ - Close Acct   │  │ - Transfer   │  │ - Online Pay   │
│ - Get Balance  │  │ - History    │  │ - Refund       │
└───────┬────────┘  └──────┬───────┘  └───────┬────────┘
        │                  │                   │
        ▼                  ▼                   ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│ RDS Postgres  │  │   DynamoDB    │  │ Aurora MySQL  │
└───────────────┘  └───────────────┘  └───────────────┘

Supporting Services:
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Customer   │  │Notification │  │   Fraud     │
│  Service    │  │  Service    │  │ Detection   │
└─────────────┘  └─────────────┘  └─────────────┘
```

---

### 🧩 Strategy 2: Domain-Driven Design (DDD) Subdomain Decomposition

**Approach**: Identify bounded contexts through domain modeling.

#### **Step 1: Event Storming Workshop**

```javascript
// ddd-event-storming.js
/**
 * Event Storming Output - Emirates NBD Banking Domain
 */
class BankingDomainModel {
  constructor() {
    this.boundedContexts = this.identifyBoundedContexts();
  }

  identifyBoundedContexts() {
    return {
      // Core Domain - Highest business value
      coreContexts: [
        {
          name: 'Account Context',
          ubiquitousLanguage: {
            entities: ['Account', 'AccountHolder', 'AccountNumber'],
            valueObjects: ['Balance', 'AccountType', 'AccountStatus'],
            aggregates: ['Account Aggregate'],
            domainEvents: [
              'AccountOpened',
              'AccountClosed',
              'AccountStatusChanged',
              'BalanceUpdated'
            ],
            commands: [
              'OpenAccount',
              'CloseAccount',
              'UpdateAccountStatus',
              'DepositFunds'
            ]
          },
          businessRules: [
            'Account number must be unique',
            'Minimum balance for savings account: AED 1,000',
            'Cannot close account with positive balance',
            'Account holder must be 18+ years old'
          ],
          team: 'Account Domain Team',
          technology: 'ECS + PostgreSQL'
        },

        {
          name: 'Transaction Context',
          ubiquitousLanguage: {
            entities: ['Transaction', 'TransactionLeg', 'Ledger'],
            valueObjects: ['Amount', 'Currency', 'TransactionType', 'TransactionStatus'],
            aggregates: ['Transaction Aggregate'],
            domainEvents: [
              'TransactionInitiated',
              'TransactionCompleted',
              'TransactionFailed',
              'TransactionReversed'
            ],
            commands: [
              'InitiateTransfer',
              'ProcessDeposit',
              'ProcessWithdrawal',
              'ReverseTransaction'
            ]
          },
          businessRules: [
            'Double-entry bookkeeping (debit = credit)',
            'Transaction must be idempotent',
            'Cannot withdraw more than available balance',
            'Transfer requires both accounts to be active'
          ],
          team: 'Transaction Domain Team',
          technology: 'Lambda + DynamoDB + Step Functions'
        },

        {
          name: 'Payment Context',
          ubiquitousLanguage: {
            entities: ['Payment', 'PaymentMethod', 'Merchant'],
            valueObjects: ['PaymentAmount', 'PaymentStatus', 'CardDetails'],
            aggregates: ['Payment Aggregate'],
            domainEvents: [
              'PaymentAuthorized',
              'PaymentCaptured',
              'PaymentRefunded',
              'PaymentFailed'
            ],
            commands: [
              'AuthorizePayment',
              'CapturePayment',
              'RefundPayment',
              'VoidPayment'
            ]
          },
          businessRules: [
            'Payment authorization expires in 7 days',
            'Refund cannot exceed original payment amount',
            'Card CVV must be validated',
            'PCI-DSS compliance required'
          ],
          team: 'Payment Domain Team',
          technology: 'EKS + Aurora'
        }
      ],

      // Supporting Subdomains
      supportingContexts: [
        {
          name: 'Customer Context',
          entities: ['Customer', 'CustomerProfile', 'KYC'],
          valueObjects: ['Email', 'Phone', 'Address'],
          technology: 'Lambda + DynamoDB'
        },
        {
          name: 'Notification Context',
          entities: ['Notification', 'NotificationTemplate'],
          valueObjects: ['NotificationType', 'Channel'],
          technology: 'Lambda + SES/SNS'
        }
      ],

      // Generic Subdomains (Buy, don't build)
      genericContexts: [
        {
          name: 'Authentication Context',
          solution: 'AWS Cognito',
          reason: 'Standard OAuth/OIDC implementation'
        },
        {
          name: 'Email Service',
          solution: 'AWS SES',
          reason: 'Commodity email sending'
        }
      ]
    };
  }

  // Identify context boundaries
  defineContextMap() {
    return {
      relationships: [
        {
          upstream: 'Account Context',
          downstream: 'Transaction Context',
          relationship: 'Customer-Supplier',
          integration: 'REST API',
          note: 'Transaction Service calls Account Service to validate account status'
        },
        {
          upstream: 'Transaction Context',
          downstream: 'Notification Context',
          relationship: 'Publisher-Subscriber',
          integration: 'EventBridge',
          note: 'Transaction events trigger notifications'
        },
        {
          upstream: 'Payment Context',
          downstream: 'Transaction Context',
          relationship: 'Conformist',
          integration: 'REST API + Events',
          note: 'Payment Service creates transactions in Transaction Context'
        },
        {
          upstream: 'Authentication Context',
          downstream: 'All Services',
          relationship: 'Open Host Service',
          integration: 'JWT Token',
          note: 'All services validate JWT tokens from Auth Service'
        }
      ]
    };
  }

  // Anti-corruption layer between contexts
  defineACL() {
    return {
      'Transaction Context → Account Context': {
        purpose: 'Protect Transaction Context from Account Context changes',
        implementation: 'AccountServiceAdapter',
        translates: {
          from: 'Account Context API (account_number, balance_amt)',
          to: 'Transaction Domain Model (accountId, balance)'
        }
      }
    };
  }
}

// Export for use
module.exports = BankingDomainModel;
```

#### **Step 2: Bounded Context Implementation**

**Account Context - Domain Model**:

```javascript
// account-service/src/domain/account.aggregate.js
/**
 * Account Aggregate Root (DDD)
 * Enforces invariants and business rules
 */
class Account {
  constructor(id, accountNumber, accountHolder, accountType, balance, status) {
    this.id = id;
    this.accountNumber = accountNumber;
    this.accountHolder = accountHolder;
    this.accountType = accountType;
    this.balance = balance;
    this.status = status;
    this.domainEvents = [];
  }

  // Factory method - ensures valid creation
  static open(accountHolder, accountType, initialDeposit) {
    // Business rule validation
    if (accountHolder.age < 18) {
      throw new Error('Account holder must be 18 years or older');
    }

    if (accountType === 'SAVINGS' && initialDeposit < 1000) {
      throw new Error('Savings account requires minimum AED 1,000');
    }

    const account = new Account(
      generateUUID(),
      generateAccountNumber(),
      accountHolder,
      accountType,
      initialDeposit,
      'ACTIVE'
    );

    // Record domain event
    account.addDomainEvent(new AccountOpenedEvent(account.id, accountHolder, initialDeposit));

    return account;
  }

  // Business method - deposit funds
  deposit(amount) {
    if (amount <= 0) {
      throw new Error('Deposit amount must be positive');
    }

    if (this.status !== 'ACTIVE') {
      throw new Error('Cannot deposit to inactive account');
    }

    this.balance += amount;
    this.addDomainEvent(new BalanceUpdatedEvent(this.id, this.balance, amount, 'DEPOSIT'));
  }

  // Business method - withdraw funds
  withdraw(amount) {
    if (amount <= 0) {
      throw new Error('Withdrawal amount must be positive');
    }

    if (this.status !== 'ACTIVE') {
      throw new Error('Cannot withdraw from inactive account');
    }

    // Business rule: maintain minimum balance
    const minimumBalance = this.accountType === 'SAVINGS' ? 1000 : 0;
    if (this.balance - amount < minimumBalance) {
      throw new Error(`Insufficient balance. Minimum balance: AED ${minimumBalance}`);
    }

    this.balance -= amount;
    this.addDomainEvent(new BalanceUpdatedEvent(this.id, this.balance, -amount, 'WITHDRAWAL'));
  }

  // Business method - close account
  close() {
    if (this.balance > 0) {
      throw new Error('Cannot close account with positive balance. Withdraw all funds first.');
    }

    if (this.status === 'CLOSED') {
      throw new Error('Account already closed');
    }

    this.status = 'CLOSED';
    this.addDomainEvent(new AccountClosedEvent(this.id));
  }

  // Helper method to add domain events
  addDomainEvent(event) {
    this.domainEvents.push(event);
  }

  // Get domain events for publishing
  getDomainEvents() {
    return this.domainEvents;
  }

  // Clear domain events after publishing
  clearDomainEvents() {
    this.domainEvents = [];
  }
}

// Value Objects
class AccountNumber {
  constructor(value) {
    if (!this.isValid(value)) {
      throw new Error('Invalid account number format');
    }
    this.value = value;
  }

  isValid(value) {
    // Emirates NBD account numbers: 12 digits starting with "101"
    return /^101\d{9}$/.test(value);
  }

  toString() {
    return this.value;
  }
}

class Balance {
  constructor(amount, currency = 'AED') {
    if (amount < 0) {
      throw new Error('Balance cannot be negative');
    }
    this.amount = amount;
    this.currency = currency;
  }

  add(amount) {
    return new Balance(this.amount + amount, this.currency);
  }

  subtract(amount) {
    return new Balance(this.amount - amount, this.currency);
  }

  toString() {
    return `${this.currency} ${this.amount.toFixed(2)}`;
  }
}

// Domain Events
class AccountOpenedEvent {
  constructor(accountId, accountHolder, initialDeposit) {
    this.eventType = 'AccountOpened';
    this.accountId = accountId;
    this.accountHolder = accountHolder;
    this.initialDeposit = initialDeposit;
    this.timestamp = new Date();
  }
}

class BalanceUpdatedEvent {
  constructor(accountId, newBalance, changeAmount, transactionType) {
    this.eventType = 'BalanceUpdated';
    this.accountId = accountId;
    this.newBalance = newBalance;
    this.changeAmount = changeAmount;
    this.transactionType = transactionType;
    this.timestamp = new Date();
  }
}

class AccountClosedEvent {
  constructor(accountId) {
    this.eventType = 'AccountClosed';
    this.accountId = accountId;
    this.timestamp = new Date();
  }
}

module.exports = { Account, AccountNumber, Balance };
```

**Repository Pattern**:

```javascript
// account-service/src/infrastructure/account.repository.js
const { Pool } = require('pg');

class AccountRepository {
  constructor() {
    this.pool = new Pool({
      host: process.env.DB_HOST,
      database: process.env.DB_NAME,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      max: 20
    });
  }

  // Save aggregate to database
  async save(account) {
    const client = await this.pool.connect();
    
    try {
      await client.query('BEGIN');

      // Save account
      await client.query(`
        INSERT INTO accounts (
          id, account_number, account_holder_name, account_type, 
          balance, status, created_at, updated_at
        ) VALUES ($1, $2, $3, $4, $5, $6, NOW(), NOW())
        ON CONFLICT (id) DO UPDATE SET
          balance = EXCLUDED.balance,
          status = EXCLUDED.status,
          updated_at = NOW()
      `, [
        account.id,
        account.accountNumber.toString(),
        account.accountHolder.name,
        account.accountType,
        account.balance.amount,
        account.status
      ]);

      // Save domain events to outbox table
      for (const event of account.getDomainEvents()) {
        await client.query(`
          INSERT INTO domain_events_outbox (
            aggregate_id, event_type, payload, created_at
          ) VALUES ($1, $2, $3, NOW())
        `, [account.id, event.eventType, JSON.stringify(event)]);
      }

      await client.query('COMMIT');

      // Clear domain events after saving
      account.clearDomainEvents();

      return account;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  // Load aggregate from database
  async findById(accountId) {
    const result = await this.pool.query(`
      SELECT * FROM accounts WHERE id = $1
    `, [accountId]);

    if (result.rows.length === 0) {
      return null;
    }

    const row = result.rows[0];
    return new Account(
      row.id,
      new AccountNumber(row.account_number),
      { name: row.account_holder_name },
      row.account_type,
      new Balance(parseFloat(row.balance)),
      row.status
    );
  }

  async findByAccountNumber(accountNumber) {
    const result = await this.pool.query(`
      SELECT * FROM accounts WHERE account_number = $1
    `, [accountNumber]);

    if (result.rows.length === 0) {
      return null;
    }

    const row = result.rows[0];
    return new Account(
      row.id,
      new AccountNumber(row.account_number),
      { name: row.account_holder_name },
      row.account_type,
      new Balance(parseFloat(row.balance)),
      row.status
    );
  }
}

module.exports = AccountRepository;
```

---

### 📐 Strategy 3: Decomposition by Data Ownership

**Approach**: Group services by database table ownership and data access patterns.

```javascript
// data-ownership-analyzer.js
class DataOwnershipAnalyzer {
  analyzeTableRelationships() {
    return {
      strongCohesion: [
        {
          service: 'AccountService',
          tables: ['accounts', 'account_holders', 'account_audit_log'],
          rationale: 'Always accessed together, strong transactional consistency needed',
          joins: ['accounts ← account_holders (1:1)', 'accounts ← account_audit_log (1:N)']
        },
        {
          service: 'TransactionService',
          tables: ['transactions', 'transaction_legs', 'ledger_entries'],
          rationale: 'Double-entry bookkeeping, must maintain ACID consistency',
          joins: ['transactions ← transaction_legs (1:2)', 'transaction_legs ← ledger_entries (1:1)']
        },
        {
          service: 'LoanService',
          tables: ['loans', 'loan_payments', 'collateral'],
          rationale: 'Loan lifecycle management, payment schedules',
          joins: ['loans ← loan_payments (1:N)', 'loans ← collateral (1:N)']
        }
      ],

      weakCohesion: [
        {
          problem: 'customers table referenced by multiple services',
          currentState: 'Monolith: customers table has 20+ foreign keys',
          solution: 'Create CustomerService with read replicas for other services',
          pattern: 'CQRS - Write to CustomerService, Read from replicas'
        }
      ],

      crossServiceJoins: [
        {
          join: 'accounts JOIN transactions',
          problem: 'Cannot do SQL JOIN across microservices',
          solutions: [
            'API Composition Pattern: Fetch from Account Service, then Transaction Service',
            'CQRS: Maintain denormalized read model',
            'Database View in Transaction Service with replicated account data'
          ]
        }
      ]
    };
  }

  // Analyze data access patterns
  analyzeAccessPatterns() {
    return {
      'Account Balance Check': {
        frequency: 'Very High (1000+ req/sec)',
        pattern: 'Read-heavy',
        recommendation: 'Cache in Redis with TTL 5 seconds',
        service: 'AccountService'
      },
      'Transaction History': {
        frequency: 'High (500 req/sec)',
        pattern: 'Read-heavy, time-range queries',
        recommendation: 'DynamoDB with GSI on accountId + timestamp',
        service: 'TransactionService'
      },
      'Account Creation': {
        frequency: 'Low (10 req/min)',
        pattern: 'Write-heavy, transactional',
        recommendation: 'RDS PostgreSQL with ACID guarantees',
        service: 'AccountService'
      }
    };
  }
}

module.exports = DataOwnershipAnalyzer;
```

---

### 🎓 Interview Discussion Points - Q3

**Q1: How do you handle distributed transactions across services?**

**A**: Use **Saga Pattern** (choreography or orchestration):
- **Choreography**: Each service publishes events, others listen and react
  - Example: Transfer saga with compensating transactions
- **Orchestration**: Central orchestrator (Step Functions) coordinates
  - Better for complex workflows with many steps

**Q2: What if services need data from other services?**

**A**: Multiple approaches:
1. **API Composition**: Call multiple services and aggregate
2. **CQRS**: Maintain read-only replicas
3. **Event-Driven**: Subscribe to events and build local cache
4. **Anti-Corruption Layer**: Translate between contexts

**Q3: How granular should microservices be?**

**A**: Follow these principles:
- **Single Responsibility**: One business capability per service
- **Team Size**: Service should be maintainable by 2-pizza team (6-8 people)
- **Deployment**: Can deploy independently without affecting others
- **Database**: Owns its own data store

**Avoid**: Nano-services (too small, high overhead) or distributed monoliths (tightly coupled).

---

## Question 4: Inter-Service Communication Patterns

### 📋 Question Statement

Design comprehensive inter-service communication strategies for Emirates NBD microservices. Implement:
- Synchronous communication (REST, gRPC)
- Asynchronous messaging (SQS, SNS, EventBridge)
- API Gateway patterns (request routing, rate limiting, authentication)
- Circuit breaker and retry patterns
- Service mesh (AWS App Mesh)

Provide complete working code with Node.js for all communication patterns.

---

### 🔄 Communication Pattern Overview

```
┌─────────────────────────────────────────────────────────────┐
│              INTER-SERVICE COMMUNICATION                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  SYNCHRONOUS (Request-Response)                             │
│  ┌──────────┐  REST/HTTP   ┌──────────┐                    │
│  │ Service A│ ──────────→  │ Service B│                    │
│  │          │ ←────────────│          │                    │
│  └──────────┘   Response   └──────────┘                    │
│  • Use when: Immediate response needed                      │
│  • Examples: Account lookup, balance check                  │
│  • Pros: Simple, easy to debug                              │
│  • Cons: Tight coupling, cascading failures                 │
│                                                              │
│  ASYNCHRONOUS (Event-Driven)                                │
│  ┌──────────┐   Publish    ┌──────────┐                    │
│  │ Service A│ ──────────→  │EventBridge│                   │
│  └──────────┘              └─────┬────┘                     │
│                                   │ Subscribe               │
│                           ┌───────▼────────┐                │
│                           │   Service B    │                │
│                           │   Service C    │                │
│                           └────────────────┘                │
│  • Use when: Fire-and-forget, multiple consumers            │
│  • Examples: Order placed, payment processed                │
│  • Pros: Loose coupling, scalable                           │
│  • Cons: Eventual consistency, harder to debug              │
└─────────────────────────────────────────────────────────────┘
```

---

### 🌐 Pattern 1: RESTful HTTP Communication

**Account Service → Transaction Service Communication**

```javascript
// transaction-service/src/clients/account-service.client.js
const axios = require('axios');
const CircuitBreaker = require('opossum');

class AccountServiceClient {
  constructor() {
    this.baseURL = process.env.ACCOUNT_SERVICE_URL || 'http://account-service.internal:3000';
    this.timeout = 5000; // 5 seconds
    
    // Configure axios instance
    this.httpClient = axios.create({
      baseURL: this.baseURL,
      timeout: this.timeout,
      headers: {
        'Content-Type': 'application/json',
        'X-Service-Name': 'transaction-service'
      }
    });

    // Setup circuit breaker
    this.breaker = new CircuitBreaker(this.makeRequest.bind(this), {
      timeout: 5000,
      errorThresholdPercentage: 50,
      resetTimeout: 30000, // Try again after 30 seconds
      rollingCountTimeout: 10000,
      rollingCountBuckets: 10
    });

    // Circuit breaker event listeners
    this.breaker.on('open', () => {
      console.error('Circuit breaker OPENED - Account Service is unhealthy');
    });

    this.breaker.on('halfOpen', () => {
      console.warn('Circuit breaker HALF-OPEN - Testing Account Service');
    });

    this.breaker.on('close', () => {
      console.info('Circuit breaker CLOSED - Account Service is healthy');
    });
  }

  // Core request method (wrapped by circuit breaker)
  async makeRequest(method, url, data = null) {
    try {
      const response = await this.httpClient.request({
        method,
        url,
        data,
        // Add correlation ID for distributed tracing
        headers: {
          'X-Correlation-ID': this.generateCorrelationId()
        }
      });
      return response.data;
    } catch (error) {
      console.error(`Account Service request failed: ${error.message}`);
      throw error;
    }
  }

  // Get account details with retry and circuit breaker
  async getAccount(accountId) {
    try {
      return await this.breaker.fire('GET', `/api/accounts/${accountId}`);
    } catch (error) {
      if (error.message.includes('CircuitBreaker is open')) {
        // Fallback: Return cached data or default values
        console.warn('Circuit breaker open, using fallback');
        return this.getFallbackAccount(accountId);
      }
      throw error;
    }
  }

  // Check if account has sufficient balance
  async checkBalance(accountId, requiredAmount) {
    try {
      const account = await this.breaker.fire('GET', `/api/accounts/${accountId}`);
      
      return {
        sufficient: account.balance >= requiredAmount,
        currentBalance: account.balance,
        requiredAmount: requiredAmount,
        shortfall: Math.max(0, requiredAmount - account.balance)
      };
    } catch (error) {
      console.error('Balance check failed:', error);
      throw new Error('Unable to verify account balance');
    }
  }

  // Update account balance (debit/credit)
  async updateBalance(accountId, amount, transactionId) {
    try {
      return await this.breaker.fire('PUT', `/api/accounts/${accountId}/balance`, {
        amount,
        transactionId,
        timestamp: new Date().toISOString()
      });
    } catch (error) {
      console.error('Balance update failed:', error);
      throw new Error('Failed to update account balance');
    }
  }

  // Fallback when circuit breaker is open
  getFallbackAccount(accountId) {
    // Return cached data from Redis or default values
    return {
      id: accountId,
      balance: 0,
      status: 'UNKNOWN',
      cached: true,
      message: 'Account Service unavailable, returning cached data'
    };
  }

  // Generate correlation ID for distributed tracing
  generateCorrelationId() {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }

  // Retry with exponential backoff
  async retryWithBackoff(fn, maxRetries = 3, initialDelay = 1000) {
    for (let attempt = 0; attempt < maxRetries; attempt++) {
      try {
        return await fn();
      } catch (error) {
        if (attempt === maxRetries - 1) throw error;
        
        const delay = initialDelay * Math.pow(2, attempt);
        console.warn(`Retry attempt ${attempt + 1}/${maxRetries} after ${delay}ms`);
        await this.sleep(delay);
      }
    }
  }

  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

module.exports = AccountServiceClient;
```

**Using the Client in Transaction Service**:

```javascript
// transaction-service/src/handlers/transfer.handler.js
const AccountServiceClient = require('../clients/account-service.client');
const { publishEvent } = require('../utils/event-publisher');

class TransferHandler {
  constructor() {
    this.accountClient = new AccountServiceClient();
  }

  async processTransfer(fromAccountId, toAccountId, amount) {
    console.log(`Processing transfer: ${fromAccountId} → ${toAccountId}, Amount: ${amount}`);

    try {
      // Step 1: Validate source account and balance
      const balanceCheck = await this.accountClient.checkBalance(fromAccountId, amount);
      
      if (!balanceCheck.sufficient) {
        throw new Error(`Insufficient balance. Available: ${balanceCheck.currentBalance}, Required: ${amount}`);
      }

      // Step 2: Validate destination account
      const toAccount = await this.accountClient.getAccount(toAccountId);
      
      if (toAccount.status !== 'ACTIVE') {
        throw new Error('Destination account is not active');
      }

      // Step 3: Debit source account
      await this.accountClient.updateBalance(fromAccountId, -amount, 'TXN-' + Date.now());

      // Step 4: Credit destination account
      await this.accountClient.updateBalance(toAccountId, amount, 'TXN-' + Date.now());

      // Step 5: Record transaction
      const transaction = {
        id: 'TXN-' + Date.now(),
        fromAccountId,
        toAccountId,
        amount,
        status: 'COMPLETED',
        timestamp: new Date()
      };

      // Step 6: Publish event for other services
      await publishEvent({
        eventType: 'TransferCompleted',
        data: transaction
      });

      return transaction;

    } catch (error) {
      console.error('Transfer failed:', error);
      
      // Publish failure event for compensation
      await publishEvent({
        eventType: 'TransferFailed',
        data: { fromAccountId, toAccountId, amount, error: error.message }
      });

      throw error;
    }
  }
}

module.exports = TransferHandler;
```

---

### ⚡ Pattern 2: gRPC for High-Performance Communication

**Why gRPC?**
- Binary protocol (faster than JSON)
- HTTP/2 (multiplexing, streaming)
- Strong typing with Protocol Buffers
- Code generation for multiple languages

#### **Step 1: Define Protocol Buffers**

```protobuf
// protos/account.proto
syntax = "proto3";

package account;

service AccountService {
  rpc GetAccount (GetAccountRequest) returns (GetAccountResponse);
  rpc CheckBalance (CheckBalanceRequest) returns (CheckBalanceResponse);
  rpc UpdateBalance (UpdateBalanceRequest) returns (UpdateBalanceResponse);
  rpc StreamAccountUpdates (StreamRequest) returns (stream AccountUpdate);
}

message GetAccountRequest {
  string account_id = 1;
}

message GetAccountResponse {
  string account_id = 1;
  string account_number = 2;
  string customer_name = 3;
  string account_type = 4;
  double balance = 5;
  string status = 6;
  string created_at = 7;
}

message CheckBalanceRequest {
  string account_id = 1;
  double required_amount = 2;
}

message CheckBalanceResponse {
  bool sufficient = 1;
  double current_balance = 2;
  double required_amount = 3;
  double shortfall = 4;
}

message UpdateBalanceRequest {
  string account_id = 1;
  double amount = 2;
  string transaction_id = 3;
}

message UpdateBalanceResponse {
  bool success = 1;
  double new_balance = 2;
  string message = 3;
}

message StreamRequest {
  string account_id = 1;
}

message AccountUpdate {
  string account_id = 1;
  double new_balance = 2;
  string update_type = 3;
  string timestamp = 4;
}
```

#### **Step 2: gRPC Server Implementation (Account Service)**

```javascript
// account-service/src/grpc/server.js
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');
const AccountRepository = require('../infrastructure/account.repository');

// Load proto file
const PROTO_PATH = path.join(__dirname, '../../protos/account.proto');
const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true
});

const accountProto = grpc.loadPackageDefinition(packageDefinition).account;

class AccountGRPCServer {
  constructor() {
    this.server = new grpc.Server();
    this.repository = new AccountRepository();
    this.registerServices();
  }

  registerServices() {
    this.server.addService(accountProto.AccountService.service, {
      GetAccount: this.getAccount.bind(this),
      CheckBalance: this.checkBalance.bind(this),
      UpdateBalance: this.updateBalance.bind(this),
      StreamAccountUpdates: this.streamAccountUpdates.bind(this)
    });
  }

  // Unary RPC: GetAccount
  async getAccount(call, callback) {
    const { account_id } = call.request;
    
    try {
      const account = await this.repository.findById(account_id);
      
      if (!account) {
        return callback({
          code: grpc.status.NOT_FOUND,
          message: 'Account not found'
        });
      }

      callback(null, {
        account_id: account.id,
        account_number: account.accountNumber.toString(),
        customer_name: account.accountHolder.name,
        account_type: account.accountType,
        balance: account.balance.amount,
        status: account.status,
        created_at: account.createdAt.toISOString()
      });
    } catch (error) {
      callback({
        code: grpc.status.INTERNAL,
        message: error.message
      });
    }
  }

  // Unary RPC: CheckBalance
  async checkBalance(call, callback) {
    const { account_id, required_amount } = call.request;
    
    try {
      const account = await this.repository.findById(account_id);
      
      if (!account) {
        return callback({
          code: grpc.status.NOT_FOUND,
          message: 'Account not found'
        });
      }

      const sufficient = account.balance.amount >= required_amount;
      const shortfall = sufficient ? 0 : required_amount - account.balance.amount;

      callback(null, {
        sufficient,
        current_balance: account.balance.amount,
        required_amount,
        shortfall
      });
    } catch (error) {
      callback({
        code: grpc.status.INTERNAL,
        message: error.message
      });
    }
  }

  // Unary RPC: UpdateBalance
  async updateBalance(call, callback) {
    const { account_id, amount, transaction_id } = call.request;
    
    try {
      const account = await this.repository.findById(account_id);
      
      if (!account) {
        return callback({
          code: grpc.status.NOT_FOUND,
          message: 'Account not found'
        });
      }

      // Apply balance change
      if (amount > 0) {
        account.deposit(amount);
      } else {
        account.withdraw(Math.abs(amount));
      }

      await this.repository.save(account);

      callback(null, {
        success: true,
        new_balance: account.balance.amount,
        message: `Balance updated successfully. Transaction ID: ${transaction_id}`
      });
    } catch (error) {
      callback({
        code: grpc.status.INTERNAL,
        message: error.message
      });
    }
  }

  // Server-side streaming: StreamAccountUpdates
  async streamAccountUpdates(call) {
    const { account_id } = call.request;
    
    // Stream updates every 5 seconds (in real app, this would be event-driven)
    const interval = setInterval(async () => {
      try {
        const account = await this.repository.findById(account_id);
        
        if (account) {
          call.write({
            account_id: account.id,
            new_balance: account.balance.amount,
            update_type: 'PERIODIC_UPDATE',
            timestamp: new Date().toISOString()
          });
        }
      } catch (error) {
        console.error('Stream error:', error);
        call.end();
      }
    }, 5000);

    // Clean up on client disconnect
    call.on('cancelled', () => {
      clearInterval(interval);
      console.log('Client cancelled stream');
    });
  }

  start(port = 50051) {
    this.server.bindAsync(
      `0.0.0.0:${port}`,
      grpc.ServerCredentials.createInsecure(),
      (err, boundPort) => {
        if (err) {
          console.error('Failed to start gRPC server:', err);
          return;
        }
        console.log(`gRPC server running on port ${boundPort}`);
        this.server.start();
      }
    );
  }
}

// Start server
const server = new AccountGRPCServer();
server.start();

module.exports = AccountGRPCServer;
```

#### **Step 3: gRPC Client Implementation (Transaction Service)**

```javascript
// transaction-service/src/clients/account-grpc.client.js
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');

const PROTO_PATH = path.join(__dirname, '../../protos/account.proto');
const packageDefinition = protoLoader.loadSync(PROTO_PATH);
const accountProto = grpc.loadPackageDefinition(packageDefinition).account;

class AccountGRPCClient {
  constructor() {
    const serverAddress = process.env.ACCOUNT_GRPC_URL || 'account-service.internal:50051';
    
    this.client = new accountProto.AccountService(
      serverAddress,
      grpc.credentials.createInsecure()
    );
  }

  // Get account details
  getAccount(accountId) {
    return new Promise((resolve, reject) => {
      this.client.GetAccount({ account_id: accountId }, (error, response) => {
        if (error) {
          reject(error);
        } else {
          resolve(response);
        }
      });
    });
  }

  // Check balance
  checkBalance(accountId, requiredAmount) {
    return new Promise((resolve, reject) => {
      this.client.CheckBalance(
        { account_id: accountId, required_amount: requiredAmount },
        (error, response) => {
          if (error) {
            reject(error);
          } else {
            resolve(response);
          }
        }
      );
    });
  }

  // Update balance
  updateBalance(accountId, amount, transactionId) {
    return new Promise((resolve, reject) => {
      this.client.UpdateBalance(
        { account_id: accountId, amount, transaction_id: transactionId },
        (error, response) => {
          if (error) {
            reject(error);
          } else {
            resolve(response);
          }
        }
      );
    });
  }

  // Stream account updates (server-side streaming)
  streamAccountUpdates(accountId, onUpdate) {
    const call = this.client.StreamAccountUpdates({ account_id: accountId });

    call.on('data', (update) => {
      onUpdate(update);
    });

    call.on('end', () => {
      console.log('Stream ended');
    });

    call.on('error', (error) => {
      console.error('Stream error:', error);
    });

    return call; // Return call object so caller can cancel
  }
}

// Usage example
async function example() {
  const client = new AccountGRPCClient();

  // Get account
  const account = await client.getAccount('account-123');
  console.log('Account:', account);

  // Check balance
  const balanceCheck = await client.checkBalance('account-123', 500);
  console.log('Balance check:', balanceCheck);

  // Stream updates
  const stream = client.streamAccountUpdates('account-123', (update) => {
    console.log('Balance update:', update);
  });

  // Cancel stream after 30 seconds
  setTimeout(() => {
    stream.cancel();
  }, 30000);
}

module.exports = AccountGRPCClient;
```

---

### 📨 Pattern 3: Asynchronous Messaging with EventBridge

```javascript
// shared/event-bridge-publisher.js
const { EventBridgeClient, PutEventsCommand } = require('@aws-sdk/client-eventbridge');

class EventBridgePublisher {
  constructor() {
    this.client = new EventBridgeClient({ region: process.env.AWS_REGION || 'us-east-1' });
    this.eventBusName = process.env.EVENT_BUS_NAME || 'banking-event-bus';
  }

  async publishEvent(source, detailType, detail) {
    const event = {
      Time: new Date(),
      Source: source,
      DetailType: detailType,
      Detail: JSON.stringify(detail),
      EventBusName: this.eventBusName
    };

    try {
      const command = new PutEventsCommand({ Entries: [event] });
      const response = await this.client.send(command);

      if (response.FailedEntryCount > 0) {
        console.error('Failed to publish event:', response.Entries);
        throw new Error('Event publication failed');
      }

      console.log(`Event published: ${detailType}`, detail);
      return response;
    } catch (error) {
      console.error('EventBridge error:', error);
      throw error;
    }
  }

  // Publish account-related events
  async publishAccountEvent(eventType, accountData) {
    return this.publishEvent('account.service', eventType, accountData);
  }

  // Publish transaction-related events
  async publishTransactionEvent(eventType, transactionData) {
    return this.publishEvent('transaction.service', eventType, transactionData);
  }
}

module.exports = EventBridgePublisher;
```

**Event Subscriber (Notification Service)**:

```javascript
// notification-service/src/handlers/transaction-event.handler.js
const { SESClient, SendEmailCommand } = require('@aws-sdk/client-ses');
const { SNSClient, PublishCommand } = require('@aws-sdk/client-sns');

class TransactionEventHandler {
  constructor() {
    this.sesClient = new SESClient({ region: 'us-east-1' });
    this.snsClient = new SNSClient({ region: 'us-east-1' });
  }

  // Lambda handler for EventBridge events
  async handler(event) {
    console.log('Received event:', JSON.stringify(event, null, 2));

    const { detail, 'detail-type': detailType } = event;

    try {
      switch (detailType) {
        case 'TransferCompleted':
          await this.handleTransferCompleted(detail);
          break;
        case 'TransferFailed':
          await this.handleTransferFailed(detail);
          break;
        case 'AccountOpened':
          await this.handleAccountOpened(detail);
          break;
        default:
          console.log(`Unhandled event type: ${detailType}`);
      }

      return { statusCode: 200, body: 'Event processed successfully' };
    } catch (error) {
      console.error('Event processing failed:', error);
      throw error;
    }
  }

  async handleTransferCompleted(detail) {
    const { fromAccountId, toAccountId, amount, timestamp } = detail;

    // Send email notification
    await this.sendEmail(
      'from_customer@example.com',
      'Transfer Completed',
      `Your transfer of AED ${amount} has been completed successfully.`
    );

    // Send SMS notification
    await this.sendSMS(
      '+971501234567',
      `Emirates NBD: Transfer of AED ${amount} completed on ${new Date(timestamp).toLocaleString()}`
    );

    console.log('Transfer notification sent');
  }

  async handleTransferFailed(detail) {
    const { fromAccountId, amount, error } = detail;

    await this.sendEmail(
      'from_customer@example.com',
      'Transfer Failed',
      `Your transfer of AED ${amount} failed. Reason: ${error}`
    );

    console.log('Transfer failure notification sent');
  }

  async handleAccountOpened(detail) {
    const { accountNumber, customerName } = detail;

    await this.sendEmail(
      detail.email,
      'Welcome to Emirates NBD!',
      `Dear ${customerName}, your account ${accountNumber} has been created successfully.`
    );
  }

  async sendEmail(to, subject, body) {
    const command = new SendEmailCommand({
      Source: 'noreply@emiratesnbd.com',
      Destination: { ToAddresses: [to] },
      Message: {
        Subject: { Data: subject },
        Body: { Text: { Data: body } }
      }
    });

    return this.sesClient.send(command);
  }

  async sendSMS(phoneNumber, message) {
    const command = new PublishCommand({
      PhoneNumber: phoneNumber,
      Message: message
    });

    return this.snsClient.send(command);
  }
}

// Export Lambda handler
exports.handler = async (event) => {
  const handler = new TransactionEventHandler();
  return handler.handler(event);
};
```

---

### 🔒 Pattern 4: API Gateway with Authentication & Rate Limiting

```javascript
// infrastructure/cdk/api-gateway-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as cognito from 'aws-cdk-lib/aws-cognito';
import * as logs from 'aws-cdk-lib/aws-logs';

export class APIGatewayStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Cognito User Pool for authentication
    const userPool = new cognito.UserPool(this, 'BankingUserPool', {
      userPoolName: 'emirates-nbd-users',
      selfSignUpEnabled: true,
      signInAliases: { email: true, phone: true },
      autoVerify: { email: true, phone: true },
      passwordPolicy: {
        minLength: 8,
        requireLowercase: true,
        requireUppercase: true,
        requireDigits: true,
        requireSymbols: true
      },
      mfa: cognito.Mfa.OPTIONAL,
      mfaSecondFactor: { sms: true, otp: true }
    });

    // User Pool Client
    const userPoolClient = userPool.addClient('WebClient', {
      authFlows: {
        userPassword: true,
        userSrp: true
      },
      oAuth: {
        flows: { authorizationCodeGrant: true },
        scopes: [cognito.OAuthScope.OPENID, cognito.OAuthScope.EMAIL]
      }
    });

    // API Gateway
    const api = new apigateway.RestApi(this, 'BankingAPI', {
      restApiName: 'Emirates NBD Banking API',
      description: 'Unified API Gateway for all banking services',
      deployOptions: {
        stageName: 'prod',
        loggingLevel: apigateway.MethodLoggingLevel.INFO,
        dataTraceEnabled: true,
        metricsEnabled: true,
        tracingEnabled: true
      },
      defaultCorsPreflightOptions: {
        allowOrigins: apigateway.Cors.ALL_ORIGINS,
        allowMethods: apigateway.Cors.ALL_METHODS
      }
    });

    // Cognito Authorizer
    const authorizer = new apigateway.CognitoUserPoolsAuthorizer(this, 'APIAuthorizer', {
      cognitoUserPools: [userPool],
      authorizerName: 'CognitoAuthorizer',
      identitySource: 'method.request.header.Authorization'
    });

    // Usage Plan for rate limiting
    const usagePlan = api.addUsagePlan('StandardUsagePlan', {
      name: 'Standard',
      throttle: {
        rateLimit: 1000,  // Requests per second
        burstLimit: 2000  // Max burst
      },
      quota: {
        limit: 10000,     // Requests per day
        period: apigateway.Period.DAY
      }
    });

    // API Key for partners
    const apiKey = api.addApiKey('PartnerAPIKey', {
      apiKeyName: 'partner-integration-key'
    });

    usagePlan.addApiKey(apiKey);
    usagePlan.addApiStage({ stage: api.deploymentStage });

    // Account Service integration
    const accountsResource = api.root.addResource('accounts');
    
    accountsResource.addMethod('GET', 
      new apigateway.HttpIntegration('http://account-service.internal:3000/api/accounts'),
      {
        authorizer,
        authorizationType: apigateway.AuthorizationType.COGNITO,
        apiKeyRequired: false,
        requestParameters: {
          'method.request.header.Authorization': true
        }
      }
    );

    // Transaction Service integration (Lambda proxy)
    const transactionsResource = api.root.addResource('transactions');
    
    const transactionLambda = lambda.Function.fromFunctionArn(
      this,
      'TransactionFunction',
      'arn:aws:lambda:us-east-1:123456789012:function:transaction-service'
    );

    transactionsResource.addMethod('POST',
      new apigateway.LambdaIntegration(transactionLambda),
      {
        authorizer,
        authorizationType: apigateway.AuthorizationType.COGNITO
      }
    );

    // Custom authorizer (Lambda-based)
    const customAuthorizerLambda = new lambda.Function(this, 'CustomAuthorizer', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromInline(`
        exports.handler = async (event) => {
          const token = event.authorizationToken;
          
          // Custom validation logic
          if (token === 'valid-token') {
            return {
              principalId: 'user',
              policyDocument: {
                Version: '2012-10-17',
                Statement: [{
                  Action: 'execute-api:Invoke',
                  Effect: 'Allow',
                  Resource: event.methodArn
                }]
              }
            };
          }
          
          throw new Error('Unauthorized');
        };
      `)
    });

    const customAuthorizer = new apigateway.TokenAuthorizer(this, 'CustomTokenAuthorizer', {
      handler: customAuthorizerLambda,
      identitySource: 'method.request.header.X-API-Token'
    });

    // Output API URL
    new cdk.CfnOutput(this, 'APIEndpoint', {
      value: api.url,
      description: 'API Gateway Endpoint'
    });
  }
}
```

---

### 🎓 Interview Discussion Points - Q4

**Q1: REST vs gRPC - When to use which?**

**A**:
- **REST**: Browser clients, public APIs, simple CRUD
- **gRPC**: Internal service-to-service, high performance, streaming

**Q2: How do you handle service failures?**

**A**:
- Circuit breaker pattern (Opossum library)
- Retry with exponential backoff
- Fallback responses
- Timeout configuration

**Q3: Synchronous vs Asynchronous communication trade-offs?**

**A**:
- **Sync**: Simple debugging, immediate feedback, but tight coupling
- **Async**: Loose coupling, scalability, but eventual consistency

---

**End of File 2**

