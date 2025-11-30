# Q54: TypeScript - Advanced Types

## 📋 Summary
This question covers **advanced TypeScript features** - generics, utility types, conditional types, mapped types, decorators, and advanced patterns for building scalable, type-safe banking systems.

**Key Topics**:
- Generic types and constraints
- Utility types (Partial, Required, Pick, Omit, etc.)
- Conditional types
- Mapped types and template literals
- Type inference and `infer` keyword
- Decorators (experimental)
- Advanced patterns (Builder, Factory)
- Type-safe API clients
- Discriminated unions
- Type branding

**Banking Use Cases**:
- Generic repository pattern
- Type-safe payment processors
- Validated configuration system
- Decorator-based validation
- Type-safe event system
- Branded types for account numbers

---

## 🎯 Advanced Type System Concepts

### Generics Deep Dive

**Generics** allow creating reusable, type-safe components that work with multiple types while maintaining type information.

```typescript
// ============================================
// Generic Repository Pattern
// ============================================

interface IEntity {
  id: string;
  createdAt: Date;
  updatedAt: Date;
}

class Repository<T extends IEntity> {
  private items: Map<string, T> = new Map();

  async create(item: Omit<T, 'id' | 'createdAt' | 'updatedAt'>): Promise<T> {
    const newItem = {
      ...item,
      id: this.generateId(),
      createdAt: new Date(),
      updatedAt: new Date(),
    } as T;

    this.items.set(newItem.id, newItem);
    return newItem;
  }

  async findById(id: string): Promise<T | undefined> {
    return this.items.get(id);
  }

  async findAll(): Promise<T[]> {
    return Array.from(this.items.values());
  }

  async update(id: string, updates: Partial<T>): Promise<T | undefined> {
    const item = this.items.get(id);
    if (!item) return undefined;

    const updated = {
      ...item,
      ...updates,
      updatedAt: new Date(),
    };

    this.items.set(id, updated);
    return updated;
  }

  async delete(id: string): Promise<boolean> {
    return this.items.delete(id);
  }

  private generateId(): string {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
}

// ============================================
// Usage with specific types
// ============================================

interface Account extends IEntity {
  accountNumber: string;
  balance: number;
  customerId: string;
}

interface Transaction extends IEntity {
  accountId: string;
  amount: number;
  type: string;
}

const accountRepo = new Repository<Account>();
const transactionRepo = new Repository<Transaction>();

// Type-safe operations
const account = await accountRepo.create({
  accountNumber: 'ACC123',
  balance: 1000,
  customerId: 'CUST001',
});

// TypeScript knows the return type
const found: Account | undefined = await accountRepo.findById(account.id);
```

### Utility Types

TypeScript provides built-in utility types for common type transformations.

```typescript
// ============================================
// Built-in Utility Types
// ============================================

interface User {
  id: string;
  username: string;
  email: string;
  password: string;
  age: number;
  isActive: boolean;
}

// Partial<T> - All properties optional
type PartialUser = Partial<User>;
// { id?: string; username?: string; ... }

// Required<T> - All properties required
type RequiredUser = Required<Partial<User>>;

// Readonly<T> - All properties readonly
type ReadonlyUser = Readonly<User>;
// { readonly id: string; readonly username: string; ... }

// Pick<T, K> - Select specific properties
type UserCredentials = Pick<User, 'username' | 'password'>;
// { username: string; password: string; }

// Omit<T, K> - Remove specific properties
type UserWithoutPassword = Omit<User, 'password'>;
// { id: string; username: string; email: string; age: number; isActive: boolean; }

// Record<K, T> - Object with specific key-value types
type UserRoles = Record<string, 'admin' | 'user' | 'guest'>;
// { [key: string]: 'admin' | 'user' | 'guest' }

// Extract<T, U> - Extract types from union
type OnlyStrings = Extract<string | number | boolean, string>;
// string

// Exclude<T, U> - Exclude types from union
type NoStrings = Exclude<string | number | boolean, string>;
// number | boolean

// NonNullable<T> - Remove null and undefined
type NonNullString = NonNullable<string | null | undefined>;
// string

// ReturnType<T> - Get function return type
function getUser(): User {
  return {} as User;
}
type UserReturnType = ReturnType<typeof getUser>;
// User

// Parameters<T> - Get function parameter types
function createUser(username: string, email: string): User {
  return {} as User;
}
type CreateUserParams = Parameters<typeof createUser>;
// [string, string]
```

---

## 💡 Example 1: Generic Payment Processor with Advanced Types

Complete type-safe payment processing system.

### Project Structure

```
payment-processor/
├── src/
│   ├── types/
│   │   ├── payment.types.ts
│   │   ├── processor.types.ts
│   │   └── utils.types.ts
│   ├── processors/
│   │   ├── BaseProcessor.ts
│   │   ├── CreditCardProcessor.ts
│   │   ├── BankTransferProcessor.ts
│   │   └── CryptoProcessor.ts
│   ├── validators/
│   │   └── decorators.ts
│   └── index.ts
└── tsconfig.json
```

### 1. Advanced Type Definitions

```typescript
// src/types/payment.types.ts

/**
 * Payment method discriminated union
 */
export type PaymentMethod = 
  | { type: 'CREDIT_CARD'; cardNumber: string; cvv: string; expiryDate: string; }
  | { type: 'BANK_TRANSFER'; accountNumber: string; routingNumber: string; }
  | { type: 'CRYPTO'; walletAddress: string; currency: 'BTC' | 'ETH'; }
  | { type: 'PAYPAL'; email: string; };

/**
 * Payment status
 */
export enum PaymentStatus {
  PENDING = 'PENDING',
  PROCESSING = 'PROCESSING',
  COMPLETED = 'COMPLETED',
  FAILED = 'FAILED',
  REFUNDED = 'REFUNDED'
}

/**
 * Base payment request
 */
export interface BasePaymentRequest {
  amount: number;
  currency: string;
  description: string;
  metadata?: Record<string, unknown>;
}

/**
 * Type-safe payment request with method
 */
export type PaymentRequest = BasePaymentRequest & {
  method: PaymentMethod;
};

/**
 * Payment result
 */
export type PaymentResult<T extends PaymentMethod = PaymentMethod> = {
  success: true;
  transactionId: string;
  status: PaymentStatus;
  method: T;
  processedAt: Date;
} | {
  success: false;
  error: string;
  errorCode: string;
  method: T;
};

/**
 * Branded types for security
 */
export type AccountNumber = string & { __brand: 'AccountNumber' };
export type TransactionId = string & { __brand: 'TransactionId' };
export type CardNumber = string & { __brand: 'CardNumber' };

/**
 * Helper to create branded types
 */
export function createAccountNumber(value: string): AccountNumber {
  if (!/^\d{10,12}$/.test(value)) {
    throw new Error('Invalid account number format');
  }
  return value as AccountNumber;
}

export function createCardNumber(value: string): CardNumber {
  if (!/^\d{16}$/.test(value)) {
    throw new Error('Invalid card number format');
  }
  return value as CardNumber;
}
```

```typescript
// src/types/processor.types.ts

import { PaymentMethod, PaymentResult } from './payment.types';

/**
 * Generic processor interface with constraints
 */
export interface IPaymentProcessor<T extends PaymentMethod> {
  processPayment(
    amount: number,
    method: T,
    metadata?: Record<string, unknown>
  ): Promise<PaymentResult<T>>;

  validatePaymentMethod(method: T): boolean;
  
  getProcessorName(): string;
}

/**
 * Processor configuration with mapped types
 */
export type ProcessorConfig<T extends PaymentMethod> = {
  enabled: boolean;
  maxAmount: number;
  minAmount: number;
} & (
  T extends { type: 'CREDIT_CARD' } ? {
    merchantId: string;
    apiKey: string;
  } :
  T extends { type: 'BANK_TRANSFER' } ? {
    bankId: string;
    routingPrefix: string;
  } :
  T extends { type: 'CRYPTO' } ? {
    networkId: string;
    confirmationsRequired: number;
  } :
  {}
);

/**
 * Conditional type helper
 */
export type ExtractMethodType<T> = T extends { type: infer U } ? U : never;

/**
 * Mapped type for processor registry
 */
export type ProcessorRegistry = {
  [K in PaymentMethod['type']]: IPaymentProcessor<Extract<PaymentMethod, { type: K }>>;
};
```

```typescript
// src/types/utils.types.ts

/**
 * Deep Partial - Make all nested properties optional
 */
export type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

/**
 * Deep Readonly - Make all nested properties readonly
 */
export type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object ? DeepReadonly<T[P]> : T[P];
};

/**
 * Require at least one property
 */
export type RequireAtLeastOne<T, Keys extends keyof T = keyof T> =
  Pick<T, Exclude<keyof T, Keys>>
  & {
      [K in Keys]-?: Required<Pick<T, K>> & Partial<Pick<T, Exclude<Keys, K>>>
    }[Keys];

/**
 * Require exactly one property
 */
export type RequireOnlyOne<T, Keys extends keyof T = keyof T> =
  Pick<T, Exclude<keyof T, Keys>>
  & {
      [K in Keys]-?:
        Required<Pick<T, K>>
        & Partial<Record<Exclude<Keys, K>, undefined>>
    }[Keys];

/**
 * Async return type helper
 */
export type Awaited<T> = T extends Promise<infer U> ? U : T;

/**
 * Function with specific this context
 */
export type BoundFunction<T, Args extends any[], R> = 
  (this: T, ...args: Args) => R;

/**
 * Extract promise result type
 */
export type PromiseType<T extends Promise<any>> = 
  T extends Promise<infer U> ? U : never;
```

### 2. Generic Base Processor

```typescript
// src/processors/BaseProcessor.ts

import { 
  PaymentMethod, 
  PaymentResult, 
  PaymentStatus,
  TransactionId 
} from '../types/payment.types';
import { IPaymentProcessor } from '../types/processor.types';

/**
 * Abstract base processor with generic type
 */
export abstract class BaseProcessor<T extends PaymentMethod> 
  implements IPaymentProcessor<T> {
  
  protected abstract processorName: string;
  protected abstract minAmount: number;
  protected abstract maxAmount: number;

  /**
   * Process payment (template method pattern)
   */
  async processPayment(
    amount: number,
    method: T,
    metadata?: Record<string, unknown>
  ): Promise<PaymentResult<T>> {
    try {
      // Validate amount
      if (!this.isValidAmount(amount)) {
        return this.createFailureResult(
          method,
          'INVALID_AMOUNT',
          `Amount must be between ${this.minAmount} and ${this.maxAmount}`
        );
      }

      // Validate payment method
      if (!this.validatePaymentMethod(method)) {
        return this.createFailureResult(
          method,
          'INVALID_METHOD',
          'Invalid payment method details'
        );
      }

      // Pre-process hook
      await this.preProcess(amount, method, metadata);

      // Actual payment processing (implemented by subclasses)
      const transactionId = await this.executePayment(amount, method, metadata);

      // Post-process hook
      await this.postProcess(transactionId, method);

      return {
        success: true,
        transactionId,
        status: PaymentStatus.COMPLETED,
        method,
        processedAt: new Date(),
      };
    } catch (error) {
      return this.createFailureResult(
        method,
        'PROCESSING_ERROR',
        error instanceof Error ? error.message : 'Unknown error'
      );
    }
  }

  /**
   * Validate payment method (to be implemented by subclasses)
   */
  abstract validatePaymentMethod(method: T): boolean;

  /**
   * Execute payment (to be implemented by subclasses)
   */
  protected abstract executePayment(
    amount: number,
    method: T,
    metadata?: Record<string, unknown>
  ): Promise<TransactionId>;

  /**
   * Get processor name
   */
  getProcessorName(): string {
    return this.processorName;
  }

  /**
   * Pre-process hook
   */
  protected async preProcess(
    amount: number,
    method: T,
    metadata?: Record<string, unknown>
  ): Promise<void> {
    console.log(`[${this.processorName}] Pre-processing payment of ${amount}`);
  }

  /**
   * Post-process hook
   */
  protected async postProcess(
    transactionId: TransactionId,
    method: T
  ): Promise<void> {
    console.log(`[${this.processorName}] Post-processing transaction ${transactionId}`);
  }

  /**
   * Validate amount
   */
  protected isValidAmount(amount: number): boolean {
    return amount >= this.minAmount && amount <= this.maxAmount;
  }

  /**
   * Generate transaction ID
   */
  protected generateTransactionId(): TransactionId {
    const timestamp = Date.now();
    const random = Math.random().toString(36).substr(2, 9);
    return `TXN-${timestamp}-${random}` as TransactionId;
  }

  /**
   * Create failure result
   */
  protected createFailureResult(
    method: T,
    errorCode: string,
    error: string
  ): PaymentResult<T> {
    return {
      success: false,
      error,
      errorCode,
      method,
    };
  }
}
```

### 3. Specific Processors

```typescript
// src/processors/CreditCardProcessor.ts

import { BaseProcessor } from './BaseProcessor';
import { PaymentMethod, TransactionId } from '../types/payment.types';

type CreditCardMethod = Extract<PaymentMethod, { type: 'CREDIT_CARD' }>;

export class CreditCardProcessor extends BaseProcessor<CreditCardMethod> {
  protected processorName = 'CreditCardProcessor';
  protected minAmount = 1;
  protected maxAmount = 10000;

  validatePaymentMethod(method: CreditCardMethod): boolean {
    // Validate card number (Luhn algorithm)
    if (!this.isValidCardNumber(method.cardNumber)) {
      return false;
    }

    // Validate CVV
    if (!/^\d{3,4}$/.test(method.cvv)) {
      return false;
    }

    // Validate expiry date
    if (!this.isValidExpiryDate(method.expiryDate)) {
      return false;
    }

    return true;
  }

  protected async executePayment(
    amount: number,
    method: CreditCardMethod,
    metadata?: Record<string, unknown>
  ): Promise<TransactionId> {
    // Simulate credit card processing
    await this.delay(1000);

    // Call payment gateway API (simulated)
    console.log(`Processing credit card payment: ${amount}`);
    console.log(`Card: ****${method.cardNumber.slice(-4)}`);

    return this.generateTransactionId();
  }

  /**
   * Validate card number using Luhn algorithm
   */
  private isValidCardNumber(cardNumber: string): boolean {
    if (!/^\d{16}$/.test(cardNumber)) {
      return false;
    }

    let sum = 0;
    let isEven = false;

    for (let i = cardNumber.length - 1; i >= 0; i--) {
      let digit = parseInt(cardNumber[i], 10);

      if (isEven) {
        digit *= 2;
        if (digit > 9) {
          digit -= 9;
        }
      }

      sum += digit;
      isEven = !isEven;
    }

    return sum % 10 === 0;
  }

  /**
   * Validate expiry date
   */
  private isValidExpiryDate(expiryDate: string): boolean {
    const [month, year] = expiryDate.split('/');
    const expiry = new Date(parseInt(`20${year}`), parseInt(month) - 1);
    return expiry > new Date();
  }

  private delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

```typescript
// src/processors/BankTransferProcessor.ts

import { BaseProcessor } from './BaseProcessor';
import { PaymentMethod, TransactionId } from '../types/payment.types';

type BankTransferMethod = Extract<PaymentMethod, { type: 'BANK_TRANSFER' }>;

export class BankTransferProcessor extends BaseProcessor<BankTransferMethod> {
  protected processorName = 'BankTransferProcessor';
  protected minAmount = 10;
  protected maxAmount = 100000;

  validatePaymentMethod(method: BankTransferMethod): boolean {
    // Validate account number (10-12 digits)
    if (!/^\d{10,12}$/.test(method.accountNumber)) {
      return false;
    }

    // Validate routing number (9 digits)
    if (!/^\d{9}$/.test(method.routingNumber)) {
      return false;
    }

    return true;
  }

  protected async executePayment(
    amount: number,
    method: BankTransferMethod,
    metadata?: Record<string, unknown>
  ): Promise<TransactionId> {
    // Simulate bank transfer processing
    await this.delay(2000);

    console.log(`Processing bank transfer: ${amount}`);
    console.log(`Account: ****${method.accountNumber.slice(-4)}`);
    console.log(`Routing: ${method.routingNumber}`);

    return this.generateTransactionId();
  }

  private delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

```typescript
// src/processors/CryptoProcessor.ts

import { BaseProcessor } from './BaseProcessor';
import { PaymentMethod, TransactionId } from '../types/payment.types';

type CryptoMethod = Extract<PaymentMethod, { type: 'CRYPTO' }>;

export class CryptoProcessor extends BaseProcessor<CryptoMethod> {
  protected processorName = 'CryptoProcessor';
  protected minAmount = 0.0001;
  protected maxAmount = 1000000;

  validatePaymentMethod(method: CryptoMethod): boolean {
    // Validate wallet address based on currency
    if (method.currency === 'BTC') {
      return /^[13][a-km-zA-HJ-NP-Z1-9]{25,34}$/.test(method.walletAddress);
    } else if (method.currency === 'ETH') {
      return /^0x[a-fA-F0-9]{40}$/.test(method.walletAddress);
    }

    return false;
  }

  protected async executePayment(
    amount: number,
    method: CryptoMethod,
    metadata?: Record<string, unknown>
  ): Promise<TransactionId> {
    // Simulate crypto transaction
    await this.delay(3000);

    console.log(`Processing crypto payment: ${amount} ${method.currency}`);
    console.log(`Wallet: ${method.walletAddress.slice(0, 10)}...`);

    return this.generateTransactionId();
  }

  private delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

### 4. Decorators for Validation

```typescript
// src/validators/decorators.ts

/**
 * Method decorator for logging
 */
export function Log(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const originalMethod = descriptor.value;

  descriptor.value = async function (...args: any[]) {
    console.log(`[${target.constructor.name}] Calling ${propertyKey}`);
    console.log(`Arguments:`, args);

    const result = await originalMethod.apply(this, args);

    console.log(`[${target.constructor.name}] ${propertyKey} completed`);
    console.log(`Result:`, result);

    return result;
  };

  return descriptor;
}

/**
 * Method decorator for timing
 */
export function Timed(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const originalMethod = descriptor.value;

  descriptor.value = async function (...args: any[]) {
    const start = Date.now();
    const result = await originalMethod.apply(this, args);
    const duration = Date.now() - start;

    console.log(`[${target.constructor.name}] ${propertyKey} took ${duration}ms`);

    return result;
  };

  return descriptor;
}

/**
 * Method decorator for retry logic
 */
export function Retry(maxRetries: number = 3, delay: number = 1000) {
  return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;

    descriptor.value = async function (...args: any[]) {
      let lastError: any;

      for (let attempt = 1; attempt <= maxRetries; attempt++) {
        try {
          return await originalMethod.apply(this, args);
        } catch (error) {
          lastError = error;
          console.log(`[${target.constructor.name}] ${propertyKey} attempt ${attempt} failed`);

          if (attempt < maxRetries) {
            await new Promise(resolve => setTimeout(resolve, delay));
          }
        }
      }

      throw lastError;
    };

    return descriptor;
  };
}

/**
 * Parameter decorator for validation
 */
export function Validate(validator: (value: any) => boolean) {
  return function (target: any, propertyKey: string, parameterIndex: number) {
    const existingValidators = Reflect.getMetadata('validators', target, propertyKey) || {};
    existingValidators[parameterIndex] = validator;
    Reflect.defineMetadata('validators', existingValidators, target, propertyKey);
  };
}
```

### 5. Payment Manager with Generics

```typescript
// src/index.ts

import { 
  PaymentMethod, 
  PaymentRequest, 
  PaymentResult 
} from './types/payment.types';
import { ProcessorRegistry } from './types/processor.types';
import { CreditCardProcessor } from './processors/CreditCardProcessor';
import { BankTransferProcessor } from './processors/BankTransferProcessor';
import { CryptoProcessor } from './processors/CryptoProcessor';

/**
 * Payment manager with type-safe processor selection
 */
export class PaymentManager {
  private processors: Partial<ProcessorRegistry> = {};

  constructor() {
    this.registerProcessor('CREDIT_CARD', new CreditCardProcessor());
    this.registerProcessor('BANK_TRANSFER', new BankTransferProcessor());
    this.registerProcessor('CRYPTO', new CryptoProcessor());
  }

  /**
   * Register processor with type safety
   */
  private registerProcessor<T extends PaymentMethod['type']>(
    type: T,
    processor: ProcessorRegistry[T]
  ): void {
    this.processors[type] = processor;
  }

  /**
   * Process payment with automatic processor selection
   */
  async processPayment<T extends PaymentMethod>(
    request: PaymentRequest & { method: T }
  ): Promise<PaymentResult<T>> {
    const processor = this.processors[request.method.type];

    if (!processor) {
      return {
        success: false,
        error: `No processor registered for ${request.method.type}`,
        errorCode: 'NO_PROCESSOR',
        method: request.method,
      } as PaymentResult<T>;
    }

    // TypeScript ensures type safety here
    return processor.processPayment(
      request.amount,
      request.method as any,
      request.metadata
    ) as Promise<PaymentResult<T>>;
  }

  /**
   * Get available processors
   */
  getAvailableProcessors(): PaymentMethod['type'][] {
    return Object.keys(this.processors) as PaymentMethod['type'][];
  }
}

// ============================================
// Usage Examples
// ============================================

async function main() {
  const manager = new PaymentManager();

  // Example 1: Credit card payment
  const creditCardResult = await manager.processPayment({
    amount: 100,
    currency: 'USD',
    description: 'Order #12345',
    method: {
      type: 'CREDIT_CARD',
      cardNumber: '4532015112830366',
      cvv: '123',
      expiryDate: '12/25',
    },
  });

  console.log('Credit Card Result:', creditCardResult);

  // Example 2: Bank transfer
  const bankTransferResult = await manager.processPayment({
    amount: 5000,
    currency: 'USD',
    description: 'Wire transfer',
    method: {
      type: 'BANK_TRANSFER',
      accountNumber: '1234567890',
      routingNumber: '123456789',
    },
  });

  console.log('Bank Transfer Result:', bankTransferResult);

  // Example 3: Crypto payment
  const cryptoResult = await manager.processPayment({
    amount: 0.5,
    currency: 'BTC',
    description: 'Crypto purchase',
    method: {
      type: 'CRYPTO',
      walletAddress: '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
      currency: 'BTC',
    },
  });

  console.log('Crypto Result:', cryptoResult);
}

main().catch(console.error);
```

### tsconfig.json with Decorators

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
    "sourceMap": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

---

## 💡 Example 2: Type-Safe Event System

Advanced event emitter with full type safety.

```typescript
// ============================================
// Type-safe Event System
// ============================================

/**
 * Event map defining all possible events
 */
interface EventMap {
  'account:created': { accountId: string; customerId: string; balance: number };
  'account:updated': { accountId: string; changes: Partial<any> };
  'transaction:completed': { transactionId: string; amount: number; from: string; to: string };
  'transaction:failed': { transactionId: string; error: string };
  'user:login': { userId: string; timestamp: Date };
  'user:logout': { userId: string; timestamp: Date };
}

/**
 * Type-safe event emitter
 */
class TypedEventEmitter<T extends Record<string, any>> {
  private listeners = new Map<keyof T, Set<(data: any) => void>>();

  /**
   * Subscribe to event with type-safe callback
   */
  on<K extends keyof T>(event: K, callback: (data: T[K]) => void): () => void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }

    this.listeners.get(event)!.add(callback);

    // Return unsubscribe function
    return () => {
      this.listeners.get(event)?.delete(callback);
    };
  }

  /**
   * Subscribe once to event
   */
  once<K extends keyof T>(event: K, callback: (data: T[K]) => void): void {
    const unsubscribe = this.on(event, (data) => {
      callback(data);
      unsubscribe();
    });
  }

  /**
   * Emit event with type-safe data
   */
  emit<K extends keyof T>(event: K, data: T[K]): void {
    const callbacks = this.listeners.get(event);

    if (callbacks) {
      callbacks.forEach(callback => callback(data));
    }
  }

  /**
   * Remove all listeners for event
   */
  removeAllListeners<K extends keyof T>(event: K): void {
    this.listeners.delete(event);
  }
}

// ============================================
// Usage
// ============================================

const events = new TypedEventEmitter<EventMap>();

// Type-safe subscription
events.on('account:created', (data) => {
  console.log(`Account created: ${data.accountId}`);
  console.log(`Balance: ${data.balance}`);
  // TypeScript knows the exact shape of data
});

events.on('transaction:completed', (data) => {
  console.log(`Transaction ${data.transactionId} completed`);
  console.log(`Amount: ${data.amount} from ${data.from} to ${data.to}`);
});

// Type-safe emission
events.emit('account:created', {
  accountId: 'ACC123',
  customerId: 'CUST001',
  balance: 1000,
});

// ❌ TypeScript error: wrong data shape
// events.emit('account:created', { wrong: 'data' });
```

---

## 🎯 Best Practices

### 1. Use Discriminated Unions

```typescript
// ✅ GOOD: Discriminated union for type safety
type Result<T, E> =
  | { success: true; data: T }
  | { success: false; error: E };

function processResult<T, E>(result: Result<T, E>): void {
  if (result.success) {
    console.log(result.data);  // TypeScript knows data exists
  } else {
    console.error(result.error);  // TypeScript knows error exists
  }
}
```

### 2. Use `infer` for Type Extraction

```typescript
// Extract return type from async function
type AsyncReturnType<T> = T extends (...args: any[]) => Promise<infer R> 
  ? R 
  : never;

async function getUser() {
  return { id: '1', name: 'John' };
}

type User = AsyncReturnType<typeof getUser>;  // { id: string; name: string }
```

### 3. Use Template Literal Types

```typescript
// Type-safe string building
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE';
type ApiVersion = 'v1' | 'v2';
type Endpoint = 'users' | 'accounts' | 'transactions';

type ApiRoute = `/${ApiVersion}/${Endpoint}`;
// "/v1/users" | "/v1/accounts" | ... | "/v2/transactions"

type HttpRequest = `${HttpMethod} ${ApiRoute}`;
// "GET /v1/users" | "POST /v1/users" | ... etc
```

### 4. Use Conditional Types

```typescript
// Smart type selection based on condition
type IsArray<T> = T extends any[] ? true : false;

type Test1 = IsArray<string[]>;  // true
type Test2 = IsArray<string>;    // false

// Practical example: unwrap arrays
type Unwrap<T> = T extends Array<infer U> ? U : T;

type UnwrappedArray = Unwrap<string[]>;  // string
type UnwrappedValue = Unwrap<number>;    // number
```

### 5. Use Mapped Types

```typescript
// Make all properties nullable
type Nullable<T> = {
  [P in keyof T]: T[P] | null;
};

interface User {
  id: string;
  name: string;
  age: number;
}

type NullableUser = Nullable<User>;
// { id: string | null; name: string | null; age: number | null }

// Make specific properties optional
type OptionalKeys<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;

type UserWithOptionalAge = OptionalKeys<User, 'age'>;
// { id: string; name: string; age?: number }
```

---

## 📚 Common Interview Questions

### Q1: Explain generics with constraints

**Answer**:
Generics with constraints limit the types that can be used:

```typescript
// Generic with extends constraint
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { id: '1', name: 'John', age: 30 };
const name = getProperty(user, 'name');  // ✅ OK
// const invalid = getProperty(user, 'invalid');  // ❌ Error
```

### Q2: What are conditional types?

**Answer**:
Conditional types select types based on conditions:

```typescript
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;  // true
type B = IsString<number>;  // false

// Practical: NonNullable implementation
type NonNullable<T> = T extends null | undefined ? never : T;
```

### Q3: Explain the `infer` keyword

**Answer**:
`infer` extracts types from within conditional types:

```typescript
// Extract return type
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

// Extract array element type
type ElementType<T> = T extends (infer E)[] ? E : never;

type NumArray = ElementType<number[]>;  // number
```

### Q4: What are mapped types?

**Answer**:
Mapped types transform properties of existing types:

```typescript
// Make all properties readonly
type Readonly<T> = {
  readonly [P in keyof T]: T[P];
};

// Make all properties optional
type Partial<T> = {
  [P in keyof T]?: T[P];
};
```

### Q5: How do decorators work in TypeScript?

**Answer**:
Decorators are functions that modify classes, methods, or properties:

```typescript
// Method decorator
function Log(target: any, key: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value;
  descriptor.value = function(...args: any[]) {
    console.log(`Calling ${key} with`, args);
    return original.apply(this, args);
  };
}

class Service {
  @Log
  getData(id: string) {
    return { id, data: 'some data' };
  }
}
```

---

## ✅ Summary & Key Takeaways

### Advanced TypeScript Checklist

```yaml
✅ Generics:
  - [ ] Use generic constraints (extends)
  - [ ] Create generic classes and functions
  - [ ] Use multiple type parameters
  - [ ] Apply conditional types with generics

✅ Utility Types:
  - [ ] Master built-in utilities (Partial, Pick, Omit, etc.)
  - [ ] Create custom utility types
  - [ ] Use mapped types for transformations
  - [ ] Apply template literal types

✅ Advanced Patterns:
  - [ ] Use discriminated unions
  - [ ] Implement type guards
  - [ ] Apply branded types for safety
  - [ ] Use conditional types effectively

✅ Decorators:
  - [ ] Enable experimentalDecorators
  - [ ] Create method decorators
  - [ ] Use class decorators
  - [ ] Apply parameter decorators
```

### Type System Levels

```
┌────────────────────────────────────────────────────┐
│        TypeScript Type System Hierarchy            │
├────────────────────────────────────────────────────┤
│                                                     │
│  Level 1: Basic Types                              │
│  ├─ string, number, boolean, array, object         │
│  └─ interfaces, type aliases                       │
│                                                     │
│  Level 2: Intermediate Types                       │
│  ├─ Generics with constraints                      │
│  ├─ Union and intersection types                   │
│  └─ Utility types (Partial, Pick, Omit)            │
│                                                     │
│  Level 3: Advanced Types                           │
│  ├─ Conditional types (T extends U ? X : Y)        │
│  ├─ Mapped types ([P in keyof T])                  │
│  ├─ Template literal types                         │
│  └─ Type inference with infer                      │
│                                                     │
│  Level 4: Expert Types                             │
│  ├─ Recursive types                                │
│  ├─ Branded types for nominal typing               │
│  ├─ Higher-order type functions                    │
│  └─ Complex type transformations                   │
│                                                     │
└────────────────────────────────────────────────────┘
```

### Key Advanced Features

| Feature | Use Case |
|---------|----------|
| **Generics** | Reusable type-safe components |
| **Conditional Types** | Type selection based on conditions |
| **Mapped Types** | Transform existing types |
| **Template Literals** | Type-safe string building |
| **Decorators** | Metadata and behavior modification |
| **Branded Types** | Nominal typing for primitives |
| **Discriminated Unions** | Type-safe state machines |

---

**Status**: ✅ Complete with advanced TypeScript patterns for production banking systems!
