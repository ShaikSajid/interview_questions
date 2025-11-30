# Q53: TypeScript - Basics with Node.js

## 📋 Summary
This question covers **TypeScript fundamentals** in Node.js applications - type safety, interfaces, classes, and practical benefits for building robust banking systems. Learn to catch errors at compile-time, improve code maintainability, and leverage IDE intelligence for faster development.

**Key Topics**:
- TypeScript setup and configuration
- Basic types and type annotations
- Interfaces and type aliases
- Classes and access modifiers
- Enums and literal types
- Type inference and assertions
- Modules and namespaces
- Compiler options (tsconfig.json)
- Integration with Express and Node.js
- Testing TypeScript code

**Banking Use Cases**:
- Type-safe transaction processing
- Strongly-typed account models
- Validated payment requests
- Error-free API contracts
- Database query type safety
- Configuration validation

---

## 🎯 What is TypeScript?

### Overview

**TypeScript** is a strongly-typed superset of JavaScript that compiles to plain JavaScript. It adds optional static typing, classes, interfaces, and modern ECMAScript features.

```
┌──────────────────────────────────────────────────────────┐
│            TypeScript Compilation Flow                   │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  TypeScript Code (.ts)                                   │
│  ┌─────────────────────────────────────┐                │
│  │ interface Account {                  │                │
│  │   id: string;                        │                │
│  │   balance: number;                   │                │
│  │ }                                    │                │
│  │                                      │                │
│  │ function deposit(                    │                │
│  │   account: Account,                  │                │
│  │   amount: number                     │                │
│  │ ): Account {                         │                │
│  │   return {                           │                │
│  │     ...account,                      │                │
│  │     balance: account.balance + amount│                │
│  │   };                                 │                │
│  │ }                                    │                │
│  └─────────────┬───────────────────────┘                │
│                │                                          │
│                ▼                                          │
│  ┌─────────────────────────────────────┐                │
│  │    TypeScript Compiler (tsc)        │                │
│  │    - Type Checking                  │                │
│  │    - Transpilation                  │                │
│  └─────────────┬───────────────────────┘                │
│                │                                          │
│                ▼                                          │
│  JavaScript Code (.js)                                   │
│  ┌─────────────────────────────────────┐                │
│  │ function deposit(account, amount) {  │                │
│  │   return {                           │                │
│  │     ...account,                      │                │
│  │     balance: account.balance + amount│                │
│  │   };                                 │                │
│  │ }                                    │                │
│  └─────────────────────────────────────┘                │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

### Benefits

| Benefit | Description |
|---------|-------------|
| **Type Safety** | Catch errors at compile-time, not runtime |
| **IDE Support** | Better autocomplete, refactoring, navigation |
| **Documentation** | Types serve as inline documentation |
| **Refactoring** | Rename, move code with confidence |
| **Scalability** | Easier to maintain large codebases |
| **Modern Features** | ES6+ features compiled to ES5 |

### TypeScript vs JavaScript

```typescript
// JavaScript - No type checking
function transfer(from, to, amount) {
  from.balance -= amount;  // What if amount is a string?
  to.balance += amount;    // What if balance is undefined?
}

transfer({ balance: 1000 }, { balance: 500 }, "100");  // Bug! String instead of number

// TypeScript - Type checking
interface Account {
  balance: number;
}

function transfer(from: Account, to: Account, amount: number): void {
  from.balance -= amount;
  to.balance += amount;
}

transfer({ balance: 1000 }, { balance: 500 }, "100");  // ❌ Compile error!
```

---

## 💡 Example 1: Complete Banking API with TypeScript

Production-ready Express application with full type safety.

### Project Setup

```bash
# Initialize project
mkdir banking-api-ts
cd banking-api-ts
npm init -y

# Install dependencies
npm install express pg redis dotenv
npm install --save-dev typescript @types/node @types/express @types/pg ts-node-dev

# Initialize TypeScript
npx tsc --init
```

### Project Structure

```
banking-api-ts/
├── src/
│   ├── types/
│   │   ├── account.types.ts
│   │   ├── transaction.types.ts
│   │   └── index.ts
│   ├── models/
│   │   ├── Account.ts
│   │   └── Transaction.ts
│   ├── services/
│   │   ├── AccountService.ts
│   │   └── TransactionService.ts
│   ├── controllers/
│   │   ├── AccountController.ts
│   │   └── TransactionController.ts
│   ├── middleware/
│   │   ├── errorHandler.ts
│   │   └── validator.ts
│   ├── routes/
│   │   └── index.ts
│   ├── config/
│   │   └── database.ts
│   └── index.ts
├── tsconfig.json
├── package.json
└── .env
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "moduleResolution": "node",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "allowSyntheticDefaultImports": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.spec.ts"]
}
```

### 1. Type Definitions

```typescript
// src/types/account.types.ts

/**
 * Account status enum
 */
export enum AccountStatus {
  ACTIVE = 'ACTIVE',
  INACTIVE = 'INACTIVE',
  FROZEN = 'FROZEN',
  CLOSED = 'CLOSED'
}

/**
 * Account type enum
 */
export enum AccountType {
  CHECKING = 'CHECKING',
  SAVINGS = 'SAVINGS',
  BUSINESS = 'BUSINESS',
  INVESTMENT = 'INVESTMENT'
}

/**
 * Account interface
 */
export interface IAccount {
  id: string;
  accountNumber: string;
  customerId: string;
  accountType: AccountType;
  balance: number;
  currency: string;
  status: AccountStatus;
  createdAt: Date;
  updatedAt: Date;
}

/**
 * Create account request
 */
export interface ICreateAccountRequest {
  customerId: string;
  accountType: AccountType;
  initialDeposit: number;
  currency?: string;
}

/**
 * Account balance response
 */
export interface IAccountBalance {
  accountNumber: string;
  balance: number;
  currency: string;
  availableBalance: number;
  pendingTransactions: number;
}

/**
 * Type guard for account
 */
export function isValidAccount(obj: any): obj is IAccount {
  return (
    typeof obj === 'object' &&
    typeof obj.id === 'string' &&
    typeof obj.accountNumber === 'string' &&
    typeof obj.balance === 'number' &&
    Object.values(AccountStatus).includes(obj.status)
  );
}
```

```typescript
// src/types/transaction.types.ts

/**
 * Transaction type enum
 */
export enum TransactionType {
  DEPOSIT = 'DEPOSIT',
  WITHDRAWAL = 'WITHDRAWAL',
  TRANSFER = 'TRANSFER',
  PAYMENT = 'PAYMENT',
  FEE = 'FEE',
  INTEREST = 'INTEREST'
}

/**
 * Transaction status enum
 */
export enum TransactionStatus {
  PENDING = 'PENDING',
  COMPLETED = 'COMPLETED',
  FAILED = 'FAILED',
  CANCELLED = 'CANCELLED'
}

/**
 * Transaction interface
 */
export interface ITransaction {
  id: string;
  accountId: string;
  type: TransactionType;
  amount: number;
  currency: string;
  status: TransactionStatus;
  description: string;
  referenceNumber: string;
  metadata?: Record<string, any>;
  createdAt: Date;
  completedAt?: Date;
}

/**
 * Transfer request
 */
export interface ITransferRequest {
  fromAccountId: string;
  toAccountId: string;
  amount: number;
  currency: string;
  description?: string;
  metadata?: Record<string, any>;
}

/**
 * Transaction response
 */
export interface ITransactionResponse {
  success: boolean;
  transaction: ITransaction;
  message: string;
}

/**
 * Transaction filter
 */
export interface ITransactionFilter {
  accountId?: string;
  type?: TransactionType;
  status?: TransactionStatus;
  startDate?: Date;
  endDate?: Date;
  minAmount?: number;
  maxAmount?: number;
  limit?: number;
  offset?: number;
}
```

### 2. Account Model

```typescript
// src/models/Account.ts

import { IAccount, AccountStatus, AccountType } from '../types/account.types';

export class Account implements IAccount {
  id: string;
  accountNumber: string;
  customerId: string;
  accountType: AccountType;
  balance: number;
  currency: string;
  status: AccountStatus;
  createdAt: Date;
  updatedAt: Date;

  constructor(data: Partial<IAccount>) {
    this.id = data.id || '';
    this.accountNumber = data.accountNumber || '';
    this.customerId = data.customerId || '';
    this.accountType = data.accountType || AccountType.CHECKING;
    this.balance = data.balance || 0;
    this.currency = data.currency || 'USD';
    this.status = data.status || AccountStatus.ACTIVE;
    this.createdAt = data.createdAt || new Date();
    this.updatedAt = data.updatedAt || new Date();
  }

  /**
   * Check if account is active
   */
  isActive(): boolean {
    return this.status === AccountStatus.ACTIVE;
  }

  /**
   * Check if withdrawal is allowed
   */
  canWithdraw(amount: number): boolean {
    if (!this.isActive()) {
      return false;
    }

    if (amount <= 0) {
      return false;
    }

    return this.balance >= amount;
  }

  /**
   * Check if deposit is allowed
   */
  canDeposit(amount: number): boolean {
    if (!this.isActive()) {
      return false;
    }

    return amount > 0;
  }

  /**
   * Calculate available balance (with pending holds)
   */
  getAvailableBalance(pendingHolds: number = 0): number {
    return Math.max(0, this.balance - pendingHolds);
  }

  /**
   * Generate account summary
   */
  getSummary(): string {
    return `Account ${this.accountNumber} (${this.accountType}): ${this.currency} ${this.balance.toFixed(2)} - ${this.status}`;
  }

  /**
   * Convert to JSON
   */
  toJSON(): IAccount {
    return {
      id: this.id,
      accountNumber: this.accountNumber,
      customerId: this.customerId,
      accountType: this.accountType,
      balance: this.balance,
      currency: this.currency,
      status: this.status,
      createdAt: this.createdAt,
      updatedAt: this.updatedAt,
    };
  }
}
```

### 3. Account Service

```typescript
// src/services/AccountService.ts

import { Pool } from 'pg';
import { Account } from '../models/Account';
import { 
  ICreateAccountRequest, 
  IAccountBalance, 
  AccountStatus,
  AccountType,
  IAccount
} from '../types/account.types';
import { randomBytes } from 'crypto';

export class AccountService {
  private pool: Pool;

  constructor(pool: Pool) {
    this.pool = pool;
  }

  /**
   * Create new account
   */
  async createAccount(request: ICreateAccountRequest): Promise<Account> {
    const accountNumber = this.generateAccountNumber();
    const id = randomBytes(16).toString('hex');

    const query = `
      INSERT INTO accounts (
        id, account_number, customer_id, account_type, 
        balance, currency, status, created_at, updated_at
      )
      VALUES ($1, $2, $3, $4, $5, $6, $7, NOW(), NOW())
      RETURNING *
    `;

    const values = [
      id,
      accountNumber,
      request.customerId,
      request.accountType,
      request.initialDeposit,
      request.currency || 'USD',
      AccountStatus.ACTIVE,
    ];

    try {
      const result = await this.pool.query(query, values);
      return new Account(this.mapRowToAccount(result.rows[0]));
    } catch (error) {
      throw new Error(`Failed to create account: ${error}`);
    }
  }

  /**
   * Get account by ID
   */
  async getAccountById(accountId: string): Promise<Account | null> {
    const query = 'SELECT * FROM accounts WHERE id = $1';

    try {
      const result = await this.pool.query(query, [accountId]);

      if (result.rows.length === 0) {
        return null;
      }

      return new Account(this.mapRowToAccount(result.rows[0]));
    } catch (error) {
      throw new Error(`Failed to get account: ${error}`);
    }
  }

  /**
   * Get account by account number
   */
  async getAccountByNumber(accountNumber: string): Promise<Account | null> {
    const query = 'SELECT * FROM accounts WHERE account_number = $1';

    try {
      const result = await this.pool.query(query, [accountNumber]);

      if (result.rows.length === 0) {
        return null;
      }

      return new Account(this.mapRowToAccount(result.rows[0]));
    } catch (error) {
      throw new Error(`Failed to get account: ${error}`);
    }
  }

  /**
   * Get account balance with pending transactions
   */
  async getAccountBalance(accountId: string): Promise<IAccountBalance> {
    const account = await this.getAccountById(accountId);

    if (!account) {
      throw new Error('Account not found');
    }

    // Get pending transaction count
    const pendingQuery = `
      SELECT COUNT(*), COALESCE(SUM(amount), 0) as pending_amount
      FROM transactions
      WHERE account_id = $1 AND status = 'PENDING'
    `;

    const pendingResult = await this.pool.query(pendingQuery, [accountId]);
    const pendingCount = parseInt(pendingResult.rows[0].count);
    const pendingAmount = parseFloat(pendingResult.rows[0].pending_amount);

    return {
      accountNumber: account.accountNumber,
      balance: account.balance,
      currency: account.currency,
      availableBalance: account.balance - pendingAmount,
      pendingTransactions: pendingCount,
    };
  }

  /**
   * Update account balance
   */
  async updateBalance(
    accountId: string, 
    amount: number, 
    client?: Pool | any
  ): Promise<Account> {
    const db = client || this.pool;

    const query = `
      UPDATE accounts
      SET balance = balance + $1, updated_at = NOW()
      WHERE id = $2
      RETURNING *
    `;

    try {
      const result = await db.query(query, [amount, accountId]);

      if (result.rows.length === 0) {
        throw new Error('Account not found');
      }

      return new Account(this.mapRowToAccount(result.rows[0]));
    } catch (error) {
      throw new Error(`Failed to update balance: ${error}`);
    }
  }

  /**
   * Freeze account
   */
  async freezeAccount(accountId: string): Promise<Account> {
    const query = `
      UPDATE accounts
      SET status = $1, updated_at = NOW()
      WHERE id = $2
      RETURNING *
    `;

    try {
      const result = await this.pool.query(query, [AccountStatus.FROZEN, accountId]);

      if (result.rows.length === 0) {
        throw new Error('Account not found');
      }

      return new Account(this.mapRowToAccount(result.rows[0]));
    } catch (error) {
      throw new Error(`Failed to freeze account: ${error}`);
    }
  }

  /**
   * Get accounts by customer ID
   */
  async getAccountsByCustomerId(customerId: string): Promise<Account[]> {
    const query = 'SELECT * FROM accounts WHERE customer_id = $1 ORDER BY created_at DESC';

    try {
      const result = await this.pool.query(query, [customerId]);
      return result.rows.map(row => new Account(this.mapRowToAccount(row)));
    } catch (error) {
      throw new Error(`Failed to get accounts: ${error}`);
    }
  }

  /**
   * Generate unique account number
   */
  private generateAccountNumber(): string {
    const timestamp = Date.now().toString().slice(-8);
    const random = Math.floor(Math.random() * 10000).toString().padStart(4, '0');
    return `ACC${timestamp}${random}`;
  }

  /**
   * Map database row to Account interface
   */
  private mapRowToAccount(row: any): IAccount {
    return {
      id: row.id,
      accountNumber: row.account_number,
      customerId: row.customer_id,
      accountType: row.account_type as AccountType,
      balance: parseFloat(row.balance),
      currency: row.currency,
      status: row.status as AccountStatus,
      createdAt: new Date(row.created_at),
      updatedAt: new Date(row.updated_at),
    };
  }
}
```

### 4. Transaction Service

```typescript
// src/services/TransactionService.ts

import { Pool } from 'pg';
import { 
  ITransaction, 
  ITransferRequest, 
  ITransactionResponse,
  TransactionType,
  TransactionStatus
} from '../types/transaction.types';
import { AccountService } from './AccountService';
import { randomBytes } from 'crypto';

export class TransactionService {
  private pool: Pool;
  private accountService: AccountService;

  constructor(pool: Pool, accountService: AccountService) {
    this.pool = pool;
    this.accountService = accountService;
  }

  /**
   * Transfer funds between accounts (with transaction)
   */
  async transfer(request: ITransferRequest): Promise<ITransactionResponse> {
    const client = await this.pool.connect();

    try {
      await client.query('BEGIN');

      // Validate accounts
      const fromAccount = await this.accountService.getAccountById(request.fromAccountId);
      const toAccount = await this.accountService.getAccountById(request.toAccountId);

      if (!fromAccount || !toAccount) {
        throw new Error('Account not found');
      }

      if (!fromAccount.canWithdraw(request.amount)) {
        throw new Error('Insufficient funds or account inactive');
      }

      if (!toAccount.canDeposit(request.amount)) {
        throw new Error('Destination account cannot accept deposits');
      }

      // Debit from account
      await this.accountService.updateBalance(
        request.fromAccountId,
        -request.amount,
        client
      );

      // Credit to account
      await this.accountService.updateBalance(
        request.toAccountId,
        request.amount,
        client
      );

      // Create transaction record
      const transaction = await this.createTransactionRecord(
        {
          accountId: request.fromAccountId,
          type: TransactionType.TRANSFER,
          amount: request.amount,
          currency: request.currency,
          description: request.description || `Transfer to ${toAccount.accountNumber}`,
          metadata: {
            ...request.metadata,
            toAccountId: request.toAccountId,
            toAccountNumber: toAccount.accountNumber,
          },
        },
        client
      );

      await client.query('COMMIT');

      return {
        success: true,
        transaction,
        message: 'Transfer completed successfully',
      };
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  /**
   * Deposit funds
   */
  async deposit(
    accountId: string, 
    amount: number, 
    description?: string
  ): Promise<ITransactionResponse> {
    const client = await this.pool.connect();

    try {
      await client.query('BEGIN');

      const account = await this.accountService.getAccountById(accountId);

      if (!account || !account.canDeposit(amount)) {
        throw new Error('Cannot deposit to this account');
      }

      await this.accountService.updateBalance(accountId, amount, client);

      const transaction = await this.createTransactionRecord(
        {
          accountId,
          type: TransactionType.DEPOSIT,
          amount,
          currency: account.currency,
          description: description || 'Deposit',
        },
        client
      );

      await client.query('COMMIT');

      return {
        success: true,
        transaction,
        message: 'Deposit completed successfully',
      };
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  /**
   * Withdraw funds
   */
  async withdraw(
    accountId: string, 
    amount: number, 
    description?: string
  ): Promise<ITransactionResponse> {
    const client = await this.pool.connect();

    try {
      await client.query('BEGIN');

      const account = await this.accountService.getAccountById(accountId);

      if (!account || !account.canWithdraw(amount)) {
        throw new Error('Cannot withdraw from this account');
      }

      await this.accountService.updateBalance(accountId, -amount, client);

      const transaction = await this.createTransactionRecord(
        {
          accountId,
          type: TransactionType.WITHDRAWAL,
          amount,
          currency: account.currency,
          description: description || 'Withdrawal',
        },
        client
      );

      await client.query('COMMIT');

      return {
        success: true,
        transaction,
        message: 'Withdrawal completed successfully',
      };
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  /**
   * Get transaction history
   */
  async getTransactionHistory(
    accountId: string, 
    limit: number = 10, 
    offset: number = 0
  ): Promise<ITransaction[]> {
    const query = `
      SELECT * FROM transactions
      WHERE account_id = $1
      ORDER BY created_at DESC
      LIMIT $2 OFFSET $3
    `;

    try {
      const result = await this.pool.query(query, [accountId, limit, offset]);
      return result.rows.map(row => this.mapRowToTransaction(row));
    } catch (error) {
      throw new Error(`Failed to get transaction history: ${error}`);
    }
  }

  /**
   * Create transaction record
   */
  private async createTransactionRecord(
    data: {
      accountId: string;
      type: TransactionType;
      amount: number;
      currency: string;
      description: string;
      metadata?: Record<string, any>;
    },
    client?: any
  ): Promise<ITransaction> {
    const db = client || this.pool;
    const id = randomBytes(16).toString('hex');
    const referenceNumber = this.generateReferenceNumber();

    const query = `
      INSERT INTO transactions (
        id, account_id, type, amount, currency, status,
        description, reference_number, metadata, created_at
      )
      VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, NOW())
      RETURNING *
    `;

    const values = [
      id,
      data.accountId,
      data.type,
      data.amount,
      data.currency,
      TransactionStatus.COMPLETED,
      data.description,
      referenceNumber,
      JSON.stringify(data.metadata || {}),
    ];

    const result = await db.query(query, values);
    return this.mapRowToTransaction(result.rows[0]);
  }

  /**
   * Generate reference number
   */
  private generateReferenceNumber(): string {
    const timestamp = Date.now().toString();
    const random = Math.floor(Math.random() * 100000).toString().padStart(5, '0');
    return `TXN${timestamp}${random}`;
  }

  /**
   * Map database row to Transaction interface
   */
  private mapRowToTransaction(row: any): ITransaction {
    return {
      id: row.id,
      accountId: row.account_id,
      type: row.type as TransactionType,
      amount: parseFloat(row.amount),
      currency: row.currency,
      status: row.status as TransactionStatus,
      description: row.description,
      referenceNumber: row.reference_number,
      metadata: row.metadata ? JSON.parse(row.metadata) : undefined,
      createdAt: new Date(row.created_at),
      completedAt: row.completed_at ? new Date(row.completed_at) : undefined,
    };
  }
}
```

### 5. Express Controllers

```typescript
// src/controllers/AccountController.ts

import { Request, Response, NextFunction } from 'express';
import { AccountService } from '../services/AccountService';
import { ICreateAccountRequest } from '../types/account.types';

export class AccountController {
  private accountService: AccountService;

  constructor(accountService: AccountService) {
    this.accountService = accountService;
  }

  /**
   * Create new account
   */
  createAccount = async (
    req: Request<{}, {}, ICreateAccountRequest>,
    res: Response,
    next: NextFunction
  ): Promise<void> => {
    try {
      const account = await this.accountService.createAccount(req.body);

      res.status(201).json({
        success: true,
        data: account.toJSON(),
        message: 'Account created successfully',
      });
    } catch (error) {
      next(error);
    }
  };

  /**
   * Get account by ID
   */
  getAccount = async (
    req: Request<{ id: string }>,
    res: Response,
    next: NextFunction
  ): Promise<void> => {
    try {
      const account = await this.accountService.getAccountById(req.params.id);

      if (!account) {
        res.status(404).json({
          success: false,
          message: 'Account not found',
        });
        return;
      }

      res.json({
        success: true,
        data: account.toJSON(),
      });
    } catch (error) {
      next(error);
    }
  };

  /**
   * Get account balance
   */
  getBalance = async (
    req: Request<{ id: string }>,
    res: Response,
    next: NextFunction
  ): Promise<void> => {
    try {
      const balance = await this.accountService.getAccountBalance(req.params.id);

      res.json({
        success: true,
        data: balance,
      });
    } catch (error) {
      next(error);
    }
  };
}
```

### 6. Main Application

```typescript
// src/index.ts

import express, { Application, Request, Response, NextFunction } from 'express';
import { Pool } from 'pg';
import dotenv from 'dotenv';
import { AccountService } from './services/AccountService';
import { TransactionService } from './services/TransactionService';
import { AccountController } from './controllers/AccountController';

dotenv.config();

const app: Application = express();
const PORT: number = parseInt(process.env.PORT || '3000', 10);

// Middleware
app.use(express.json());

// Database connection
const pool = new Pool({
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT || '5432', 10),
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  max: 20,
});

// Services
const accountService = new AccountService(pool);
const transactionService = new TransactionService(pool, accountService);

// Controllers
const accountController = new AccountController(accountService);

// Routes
app.post('/api/accounts', accountController.createAccount);
app.get('/api/accounts/:id', accountController.getAccount);
app.get('/api/accounts/:id/balance', accountController.getBalance);

// Health check
app.get('/health', (req: Request, res: Response) => {
  res.json({ status: 'healthy', timestamp: new Date().toISOString() });
});

// Error handler
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  console.error(err.stack);
  res.status(500).json({
    success: false,
    message: err.message || 'Internal server error',
  });
});

// Start server
app.listen(PORT, () => {
  console.log(`🏦 Banking API (TypeScript) running on port ${PORT}`);
});
```

### package.json Scripts

```json
{
  "scripts": {
    "dev": "ts-node-dev --respawn --transpile-only src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "type-check": "tsc --noEmit",
    "lint": "eslint src/**/*.ts"
  }
}
```

---

## 🎯 Best Practices

### 1. Use Strict Mode

```typescript
// tsconfig.json - Enable all strict checks
{
  "compilerOptions": {
    "strict": true,                     // Enable all strict checks
    "noImplicitAny": true,              // Error on implicit any
    "strictNullChecks": true,           // Null checking
    "strictFunctionTypes": true,        // Function type checking
    "strictBindCallApply": true,        // Strict bind/call/apply
    "strictPropertyInitialization": true // Class property init
  }
}
```

### 2. Avoid `any` Type

```typescript
// ❌ BAD: Using any
function processData(data: any) {
  return data.value.toUpperCase();  // No type safety!
}

// ✅ GOOD: Specific types
interface DataWithValue {
  value: string;
}

function processData(data: DataWithValue): string {
  return data.value.toUpperCase();  // Type-safe!
}
```

### 3. Use Readonly for Immutability

```typescript
// Readonly properties
interface Config {
  readonly apiKey: string;
  readonly timeout: number;
}

const config: Config = {
  apiKey: 'secret',
  timeout: 5000,
};

// config.apiKey = 'new'; // ❌ Error: Cannot assign to readonly

// Readonly arrays
const currencies: readonly string[] = ['USD', 'EUR', 'GBP'];
// currencies.push('JPY'); // ❌ Error: push does not exist
```

### 4. Use Type Guards

```typescript
// Type guard function
function isAccount(obj: any): obj is IAccount {
  return (
    obj &&
    typeof obj.id === 'string' &&
    typeof obj.balance === 'number' &&
    typeof obj.accountNumber === 'string'
  );
}

// Usage
function processAccount(data: unknown) {
  if (isAccount(data)) {
    console.log(data.balance);  // TypeScript knows it's IAccount
  } else {
    throw new Error('Invalid account data');
  }
}
```

### 5. Use Union Types

```typescript
// Union type for API responses
type ApiResponse<T> = 
  | { success: true; data: T }
  | { success: false; error: string };

async function fetchAccount(id: string): Promise<ApiResponse<IAccount>> {
  try {
    const account = await getAccount(id);
    return { success: true, data: account };
  } catch (error) {
    return { success: false, error: error.message };
  }
}

// Usage with type narrowing
const response = await fetchAccount('123');

if (response.success) {
  console.log(response.data.balance);  // TypeScript knows data exists
} else {
  console.error(response.error);  // TypeScript knows error exists
}
```

---

## 📚 Common Interview Questions

### Q1: What are the benefits of using TypeScript?

**Answer**:
1. **Type safety**: Catch errors at compile-time
2. **Better tooling**: IDE autocomplete, refactoring, navigation
3. **Code documentation**: Types serve as inline docs
4. **Scalability**: Easier to maintain large codebases
5. **Modern features**: ES6+ features with backward compatibility

### Q2: Explain `interface` vs `type` in TypeScript

**Answer**:
- **Interface**: Can be extended, merged, better for objects
- **Type**: Can use unions, intersections, better for primitives and complex types

```typescript
// Interface
interface User {
  name: string;
}
interface User {  // Declaration merging
  age: number;
}

// Type
type ID = string | number;  // Union (not possible with interface)
type Point = { x: number; y: number };
```

### Q3: What is `strict` mode in TypeScript?

**Answer**:
Strict mode enables all strict type checking options:
- `noImplicitAny`: No implicit any types
- `strictNullChecks`: Null/undefined handling
- `strictFunctionTypes`: Function parameter checking
- `strictBindCallApply`: Strict bind/call/apply
- `strictPropertyInitialization`: Class property initialization

### Q4: How do you handle errors in TypeScript?

**Answer**:
```typescript
// Custom error class
class BankingError extends Error {
  constructor(
    message: string,
    public code: string,
    public statusCode: number = 500
  ) {
    super(message);
    this.name = 'BankingError';
  }
}

// Usage
throw new BankingError('Insufficient funds', 'INSUFFICIENT_FUNDS', 400);
```

### Q5: What are generics in TypeScript?

**Answer**:
Generics allow creating reusable components that work with multiple types:

```typescript
// Generic function
function getFirst<T>(arr: T[]): T | undefined {
  return arr[0];
}

const firstNum = getFirst([1, 2, 3]);     // number
const firstStr = getFirst(['a', 'b']);    // string

// Generic interface
interface ApiResponse<T> {
  data: T;
  status: number;
}

const accountResponse: ApiResponse<IAccount> = {
  data: account,
  status: 200,
};
```

---

## ✅ Summary & Key Takeaways

### TypeScript Basics Checklist

```yaml
✅ Setup:
  - [ ] Install TypeScript and types
  - [ ] Configure tsconfig.json
  - [ ] Enable strict mode
  - [ ] Setup ts-node-dev for development

✅ Type System:
  - [ ] Use interfaces for object shapes
  - [ ] Define enums for constants
  - [ ] Use type aliases for complex types
  - [ ] Avoid any type
  - [ ] Use readonly for immutability

✅ Best Practices:
  - [ ] Type all function parameters and returns
  - [ ] Use type guards for runtime checks
  - [ ] Prefer union types over any
  - [ ] Use generics for reusable code
  - [ ] Enable all strict checks

✅ Development:
  - [ ] Type check before commit
  - [ ] Use ESLint with TypeScript rules
  - [ ] Generate source maps for debugging
  - [ ] Build before deployment
```

### Key Concepts

```
┌──────────────────────────────────────────────────────┐
│          TypeScript Type System                      │
├──────────────────────────────────────────────────────┤
│                                                       │
│  Basic Types:                                        │
│  ├─ string, number, boolean                          │
│  ├─ array, tuple, enum                               │
│  └─ any, unknown, void, never                        │
│                                                       │
│  Complex Types:                                      │
│  ├─ interface (object shapes)                        │
│  ├─ type (aliases, unions, intersections)            │
│  ├─ class (with access modifiers)                    │
│  └─ generics (reusable types)                        │
│                                                       │
│  Type Operations:                                    │
│  ├─ Union (A | B)                                    │
│  ├─ Intersection (A & B)                             │
│  ├─ Type guards (is, typeof, instanceof)             │
│  └─ Type assertions (as, <Type>)                     │
│                                                       │
└──────────────────────────────────────────────────────┘
```

### Benefits Summary

| Feature | JavaScript | TypeScript |
|---------|-----------|------------|
| **Type Checking** | Runtime only | Compile-time + Runtime |
| **IDE Support** | Basic | Advanced (autocomplete, refactor) |
| **Errors** | Found at runtime | Found at compile-time |
| **Refactoring** | Manual, risky | Safe, automated |
| **Documentation** | Comments only | Types + Comments |
| **Learning Curve** | Easy | Moderate |

---

**Status**: ✅ Complete with production-ready TypeScript basics for Node.js banking applications!
