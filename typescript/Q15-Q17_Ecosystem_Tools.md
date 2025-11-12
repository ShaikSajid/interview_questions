# Questions 15-17: TypeScript Ecosystem & Tools

## Question 15: How do you use TypeScript with ORMs (TypeORM, Prisma)?

### Answer:

**TypeScript with ORMs** provides type-safe database operations. Modern ORMs like TypeORM and Prisma offer excellent TypeScript integration with automatic type generation and type-safe queries.

### Key Concepts:
1. **Entity Definition** - Type-safe models
2. **Repository Pattern** - Data access layer
3. **Query Building** - Type-safe queries
4. **Relations** - Typed relationships
5. **Migrations** - Schema management

### Banking Scenario: ORM Integration at Emirates NBD

```typescript
console.log('=== TypeScript with ORMs - Emirates NBD ===\n');

// =============================================================================
// 1. TYPEORM ENTITIES
// =============================================================================

console.log('1. TYPEORM ENTITIES:\n');

console.log('Entity Decorators:\n');
console.log(`
import { Entity, PrimaryGeneratedColumn, Column, CreateDateColumn, 
         UpdateDateColumn, OneToMany, ManyToOne, Index } from 'typeorm';

@Entity('accounts')
@Index(['customerId', 'accountType'])
export class Account {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ unique: true })
  accountNumber: string;

  @Column()
  customerId: string;

  @Column({
    type: 'enum',
    enum: ['SAVINGS', 'CURRENT', 'FIXED'],
    default: 'SAVINGS'
  })
  accountType: 'SAVINGS' | 'CURRENT' | 'FIXED';

  @Column({ type: 'decimal', precision: 15, scale: 2, default: 0 })
  balance: number;

  @Column({ length: 3, default: 'AED' })
  currency: string;

  @Column({
    type: 'enum',
    enum: ['ACTIVE', 'FROZEN', 'CLOSED'],
    default: 'ACTIVE'
  })
  status: 'ACTIVE' | 'FROZEN' | 'CLOSED';

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @OneToMany(() => Transaction, transaction => transaction.account)
  transactions: Transaction[];

  @ManyToOne(() => Customer, customer => customer.accounts)
  customer: Customer;
}

@Entity('transactions')
export class Transaction {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  accountId: string;

  @Column({
    type: 'enum',
    enum: ['DEPOSIT', 'WITHDRAWAL', 'TRANSFER_IN', 'TRANSFER_OUT']
  })
  type: 'DEPOSIT' | 'WITHDRAWAL' | 'TRANSFER_IN' | 'TRANSFER_OUT';

  @Column({ type: 'decimal', precision: 15, scale: 2 })
  amount: number;

  @Column({ type: 'text', nullable: true })
  description?: string;

  @Column({ nullable: true })
  referenceNumber?: string;

  @CreateDateColumn()
  timestamp: Date;

  @ManyToOne(() => Account, account => account.transactions)
  account: Account;
}

@Entity('customers')
export class Customer {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  firstName: string;

  @Column()
  lastName: string;

  @Column({ unique: true })
  email: string;

  @Column({ unique: true })
  phoneNumber: string;

  @CreateDateColumn()
  createdAt: Date;

  @OneToMany(() => Account, account => account.customer)
  accounts: Account[];
}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. TYPEORM REPOSITORY
// =============================================================================

console.log('2. TYPEORM REPOSITORY:\n');

console.log('Type-safe Repository Operations:\n');
console.log(`
import { Repository, DataSource, FindOptionsWhere } from 'typeorm';

export class AccountRepository {
  private repository: Repository<Account>;

  constructor(dataSource: DataSource) {
    this.repository = dataSource.getRepository(Account);
  }

  async findById(id: string): Promise<Account | null> {
    return this.repository.findOne({
      where: { id },
      relations: ['transactions', 'customer']
    });
  }

  async findByAccountNumber(accountNumber: string): Promise<Account | null> {
    return this.repository.findOne({
      where: { accountNumber }
    });
  }

  async findByCustomerId(customerId: string): Promise<Account[]> {
    return this.repository.find({
      where: { customerId },
      order: { createdAt: 'DESC' }
    });
  }

  async create(data: {
    accountNumber: string;
    customerId: string;
    accountType: 'SAVINGS' | 'CURRENT' | 'FIXED';
    initialBalance: number;
  }): Promise<Account> {
    const account = this.repository.create({
      accountNumber: data.accountNumber,
      customerId: data.customerId,
      accountType: data.accountType,
      balance: data.initialBalance
    });
    
    return this.repository.save(account);
  }

  async updateBalance(id: string, newBalance: number): Promise<Account> {
    await this.repository.update(id, { balance: newBalance });
    const account = await this.findById(id);
    if (!account) throw new Error('Account not found');
    return account;
  }

  async findActiveAccounts(): Promise<Account[]> {
    return this.repository.find({
      where: { status: 'ACTIVE' },
      relations: ['customer']
    });
  }

  async getTotalBalance(customerId: string): Promise<number> {
    const result = await this.repository
      .createQueryBuilder('account')
      .select('SUM(account.balance)', 'total')
      .where('account.customerId = :customerId', { customerId })
      .andWhere('account.status = :status', { status: 'ACTIVE' })
      .getRawOne();
    
    return parseFloat(result.total) || 0;
  }

  async findWithHighBalance(minBalance: number): Promise<Account[]> {
    return this.repository
      .createQueryBuilder('account')
      .leftJoinAndSelect('account.customer', 'customer')
      .where('account.balance >= :minBalance', { minBalance })
      .orderBy('account.balance', 'DESC')
      .getMany();
  }
}\n`);

console.log('Query Builder (Complex Queries):\n');
console.log(`
// Complex join with conditions
const accounts = await accountRepository
  .createQueryBuilder('account')
  .leftJoinAndSelect('account.transactions', 'transaction')
  .leftJoinAndSelect('account.customer', 'customer')
  .where('account.accountType = :type', { type: 'SAVINGS' })
  .andWhere('account.balance > :minBalance', { minBalance: 10000 })
  .andWhere('transaction.timestamp >= :startDate', { 
    startDate: new Date('2025-01-01') 
  })
  .orderBy('account.balance', 'DESC')
  .take(10)
  .getMany();

// Aggregations
const stats = await accountRepository
  .createQueryBuilder('account')
  .select('account.accountType', 'type')
  .addSelect('COUNT(account.id)', 'count')
  .addSelect('SUM(account.balance)', 'totalBalance')
  .addSelect('AVG(account.balance)', 'avgBalance')
  .groupBy('account.accountType')
  .getRawMany();\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. PRISMA SCHEMA & CLIENT
// =============================================================================

console.log('3. PRISMA SCHEMA:\n');

console.log('schema.prisma:\n');
console.log(`
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Customer {
  id          String    @id @default(uuid())
  firstName   String
  lastName    String
  email       String    @unique
  phoneNumber String    @unique
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  
  accounts    Account[]
  
  @@map("customers")
}

model Account {
  id            String      @id @default(uuid())
  accountNumber String      @unique
  customerId    String
  accountType   AccountType @default(SAVINGS)
  balance       Decimal     @default(0) @db.Decimal(15, 2)
  currency      String      @default("AED") @db.VarChar(3)
  status        AccountStatus @default(ACTIVE)
  createdAt     DateTime    @default(now())
  updatedAt     DateTime    @updatedAt
  
  customer      Customer    @relation(fields: [customerId], references: [id])
  transactions  Transaction[]
  
  @@index([customerId, accountType])
  @@map("accounts")
}

model Transaction {
  id              String          @id @default(uuid())
  accountId       String
  type            TransactionType
  amount          Decimal         @db.Decimal(15, 2)
  description     String?         @db.Text
  referenceNumber String?
  timestamp       DateTime        @default(now())
  
  account         Account         @relation(fields: [accountId], references: [id])
  
  @@index([accountId, timestamp])
  @@map("transactions")
}

enum AccountType {
  SAVINGS
  CURRENT
  FIXED
}

enum AccountStatus {
  ACTIVE
  FROZEN
  CLOSED
}

enum TransactionType {
  DEPOSIT
  WITHDRAWAL
  TRANSFER_IN
  TRANSFER_OUT
}\n`);

console.log('Generated Types (Automatic):');
console.log('  • Type-safe Prisma Client');
console.log('  • All models as TypeScript types');
console.log('  • Query result types');
console.log('  • Relation types\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. PRISMA CLIENT USAGE
// =============================================================================

console.log('4. PRISMA CLIENT:\n');

console.log('Type-safe Queries:\n');
console.log(`
import { PrismaClient, AccountType, AccountStatus } from '@prisma/client';

const prisma = new PrismaClient();

// Create with relations
const account = await prisma.account.create({
  data: {
    accountNumber: 'ACC-123456',
    accountType: AccountType.SAVINGS,
    balance: 10000,
    currency: 'AED',
    customer: {
      connect: { id: 'customer-id' }
    }
  },
  include: {
    customer: true
  }
});

// Type: Account & { customer: Customer }

// Find with filters
const accounts = await prisma.account.findMany({
  where: {
    customerId: 'customer-id',
    status: AccountStatus.ACTIVE,
    balance: {
      gte: 1000
    }
  },
  include: {
    transactions: {
      orderBy: { timestamp: 'desc' },
      take: 10
    }
  }
});

// Type: (Account & { transactions: Transaction[] })[]

// Update
const updated = await prisma.account.update({
  where: { id: 'account-id' },
  data: { balance: 15000 }
});

// Transactions
const result = await prisma.$transaction(async (tx) => {
  // Deduct from source
  await tx.account.update({
    where: { id: 'source-account' },
    data: { balance: { decrement: 1000 } }
  });
  
  // Add to destination
  await tx.account.update({
    where: { id: 'dest-account' },
    data: { balance: { increment: 1000 } }
  });
  
  // Record transactions
  await tx.transaction.createMany({
    data: [
      {
        accountId: 'source-account',
        type: 'TRANSFER_OUT',
        amount: 1000
      },
      {
        accountId: 'dest-account',
        type: 'TRANSFER_IN',
        amount: 1000
      }
    ]
  });
  
  return { success: true };
});

// Aggregations
const stats = await prisma.account.aggregate({
  where: { status: AccountStatus.ACTIVE },
  _sum: { balance: true },
  _avg: { balance: true },
  _count: true
});

// Group by
const byType = await prisma.account.groupBy({
  by: ['accountType'],
  _sum: { balance: true },
  _count: true
});

// Raw queries (when needed)
const result = await prisma.$queryRaw\`
  SELECT account_type, COUNT(*) as count, SUM(balance) as total
  FROM accounts
  WHERE status = 'ACTIVE'
  GROUP BY account_type
\`;
\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. ORM COMPARISON & BEST PRACTICES
// =============================================================================

console.log('5. ORM COMPARISON:\n');

console.log('TypeORM vs Prisma:\n');
console.log(`
┌─────────────────┬──────────────────────┬──────────────────────┐
│ Feature         │ TypeORM              │ Prisma               │
├─────────────────┼──────────────────────┼──────────────────────┤
│ Type Generation │ Manual decorators    │ Auto-generated       │
│ Schema          │ Code-first           │ Schema-first         │
│ Migrations      │ Manual/auto          │ Prisma Migrate       │
│ Query Builder   │ Full featured        │ Limited              │
│ Relations       │ Decorators           │ Schema definition    │
│ Type Safety     │ Good                 │ Excellent            │
│ Performance     │ Good                 │ Excellent            │
│ Learning Curve  │ Moderate             │ Easy                 │
└─────────────────┴──────────────────────┴──────────────────────┘\n`);

console.log('Best Practices:\n');

console.log('1. Use Transactions for Multi-Step Operations:');
console.log(`
// TypeORM
await dataSource.transaction(async (manager) => {
  await manager.save(account1);
  await manager.save(account2);
  await manager.save(transaction);
});

// Prisma
await prisma.$transaction([
  prisma.account.update({ where: { id: '1' }, data: { balance: 5000 } }),
  prisma.account.update({ where: { id: '2' }, data: { balance: 3000 } }),
  prisma.transaction.create({ data: { ... } })
]);\n`);

console.log('2. Optimize Queries with Select/Include:');
console.log(`
// Only fetch needed fields
const account = await prisma.account.findUnique({
  where: { id: 'account-id' },
  select: {
    id: true,
    accountNumber: true,
    balance: true,
    customer: {
      select: {
        firstName: true,
        lastName: true
      }
    }
  }
});\n`);

console.log('3. Handle Relations Efficiently:');
console.log(`
// Avoid N+1 queries - use eager loading
const accounts = await prisma.account.findMany({
  include: {
    customer: true,
    transactions: true
  }
});

// Or use dataloader pattern for GraphQL\n`);

console.log('4. Use Soft Deletes:');
console.log(`
model Account {
  id        String    @id
  deletedAt DateTime?
  
  @@map("accounts")
}

// Query only active records
const accounts = await prisma.account.findMany({
  where: { deletedAt: null }
});\n`);

console.log('5. Index Critical Fields:');
console.log(`
model Account {
  customerId String
  status     String
  
  @@index([customerId])
  @@index([status])
  @@index([customerId, status])
}\n`);

console.log('='.repeat(70) + '\n');

console.log('ORM SUMMARY:\n');

console.log('TypeORM Benefits:');
console.log('  ✓ Flexible query builder');
console.log('  ✓ Multiple database support');
console.log('  ✓ Advanced features');
console.log('  ✓ Active Record pattern\n');

console.log('Prisma Benefits:');
console.log('  ✓ Excellent TypeScript support');
console.log('  ✓ Auto-generated types');
console.log('  ✓ Better developer experience');
console.log('  ✓ Built-in migrations\n');

console.log('Common Best Practices:');
console.log('  • Use connection pooling');
console.log('  • Enable query logging in dev');
console.log('  • Add indexes for performance');
console.log('  • Use transactions for consistency');
console.log('  • Validate at application layer');
console.log('  • Handle errors gracefully');
console.log('  • Monitor query performance');

console.log();
```

---

## Question 16: How do you implement middleware and interceptors in TypeScript?

### Answer:

**Middleware and Interceptors** are patterns for cross-cutting concerns like authentication, logging, validation, and error handling. TypeScript provides type safety for these patterns.

### Key Concepts:
1. **Express Middleware** - Request/response pipeline
2. **NestJS Interceptors** - Transform data flow
3. **Middleware Chain** - Sequential processing
4. **Error Middleware** - Error handling
5. **Custom Decorators** - Metadata-based middleware

### Banking Scenario: Middleware at Emirates NBD

```typescript
console.log('=== Middleware & Interceptors - Emirates NBD ===\n');

// =============================================================================
// 1. EXPRESS MIDDLEWARE
// =============================================================================

console.log('1. EXPRESS MIDDLEWARE:\n');

console.log('Type-safe Middleware:\n');
console.log(`
import { Request, Response, NextFunction } from 'express';

// Extend Request type for custom properties
declare global {
  namespace Express {
    interface Request {
      user?: {
        id: string;
        email: string;
        role: string;
      };
      startTime?: number;
      requestId?: string;
    }
  }
}

// Middleware type
type Middleware = (
  req: Request,
  res: Response,
  next: NextFunction
) => void | Promise<void>;

// Authentication middleware
const authenticate: Middleware = async (req, res, next) => {
  try {
    const token = req.headers.authorization?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'No token provided' });
    }
    
    // Verify token (simulated)
    const user = await verifyToken(token);
    req.user = user;
    
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

// Authorization middleware factory
const requireRole = (role: string): Middleware => {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Not authenticated' });
    }
    
    if (req.user.role !== role) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    
    next();
  };
};

// Request logging middleware
const requestLogger: Middleware = (req, res, next) => {
  req.startTime = Date.now();
  req.requestId = \`REQ-\${Date.now()}\`;
  
  console.log(\`[\${req.requestId}] \${req.method} \${req.path}\`);
  
  res.on('finish', () => {
    const duration = Date.now() - req.startTime!;
    console.log(
      \`[\${req.requestId}] \${res.statusCode} - \${duration}ms\`
    );
  });
  
  next();
};

// Validation middleware factory
const validateBody = <T>(schema: Schema<T>): Middleware => {
  return async (req, res, next) => {
    try {
      const validated = await schema.validate(req.body);
      req.body = validated;
      next();
    } catch (error) {
      res.status(400).json({
        error: 'Validation failed',
        details: error
      });
    }
  };
};

// Rate limiting middleware
const rateLimiter = (maxRequests: number, windowMs: number): Middleware => {
  const requests = new Map<string, number[]>();
  
  return (req, res, next) => {
    const clientId = req.ip || 'unknown';
    const now = Date.now();
    const windowStart = now - windowMs;
    
    // Get recent requests
    let clientRequests = requests.get(clientId) || [];
    clientRequests = clientRequests.filter(time => time > windowStart);
    
    if (clientRequests.length >= maxRequests) {
      return res.status(429).json({
        error: 'Too many requests',
        retryAfter: Math.ceil((clientRequests[0] + windowMs - now) / 1000)
      });
    }
    
    clientRequests.push(now);
    requests.set(clientId, clientRequests);
    
    next();
  };
};

// Usage
app.use(requestLogger);
app.use('/api', rateLimiter(100, 60000)); // 100 req/min

app.post('/api/accounts',
  authenticate,
  requireRole('ADMIN'),
  validateBody(accountSchema),
  accountController.create
);\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. ERROR HANDLING MIDDLEWARE
// =============================================================================

console.log('2. ERROR HANDLING:\n');

console.log('Custom Error Classes:\n');
console.log(`
class AppError extends Error {
  constructor(
    message: string,
    public statusCode: number = 500,
    public code: string = 'INTERNAL_ERROR',
    public isOperational: boolean = true
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

class ValidationError extends AppError {
  constructor(message: string, public fields?: Record<string, string>) {
    super(message, 400, 'VALIDATION_ERROR');
  }
}

class NotFoundError extends AppError {
  constructor(resource: string) {
    super(\`\${resource} not found\`, 404, 'NOT_FOUND');
  }
}

class UnauthorizedError extends AppError {
  constructor(message: string = 'Unauthorized') {
    super(message, 401, 'UNAUTHORIZED');
  }
}

class InsufficientFundsError extends AppError {
  constructor(accountId: string, required: number, available: number) {
    super(
      \`Insufficient funds in account \${accountId}\`,
      400,
      'INSUFFICIENT_FUNDS'
    );
  }
}\n`);

console.log('Error Handler Middleware:\n');
console.log(`
import { ErrorRequestHandler } from 'express';

const errorHandler: ErrorRequestHandler = (err, req, res, next) => {
  // Log error
  console.error('Error:', {
    name: err.name,
    message: err.message,
    stack: err.stack,
    requestId: req.requestId
  });
  
  // Handle known errors
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      error: {
        code: err.code,
        message: err.message,
        ...(err instanceof ValidationError && { fields: err.fields })
      }
    });
  }
  
  // Handle unexpected errors
  res.status(500).json({
    error: {
      code: 'INTERNAL_ERROR',
      message: 'An unexpected error occurred'
    }
  });
};

// Async error wrapper
const asyncHandler = (fn: Middleware): Middleware => {
  return (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
};

// Usage
app.post('/api/transfer', asyncHandler(async (req, res) => {
  const { fromAccount, toAccount, amount } = req.body;
  
  // Business logic that may throw errors
  const result = await transferService.transfer(fromAccount, toAccount, amount);
  
  res.json(result);
}));

// Error handler must be last
app.use(errorHandler);\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. NESTJS INTERCEPTORS
// =============================================================================

console.log('3. NESTJS INTERCEPTORS:\n');

console.log('Basic Interceptor:\n');
console.log(`
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap, map } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const { method, url } = request;
    const now = Date.now();
    
    console.log(\`Before: \${method} \${url}\`);
    
    return next.handle().pipe(
      tap(() => {
        const duration = Date.now() - now;
        console.log(\`After: \${method} \${url} - \${duration}ms\`);
      })
    );
  }
}

// Transform response interceptor
@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(
    context: ExecutionContext,
    next: CallHandler
  ): Observable<Response<T>> {
    return next.handle().pipe(
      map(data => ({
        success: true,
        data,
        timestamp: new Date().toISOString()
      }))
    );
  }
}

interface Response<T> {
  success: boolean;
  data: T;
  timestamp: string;
}

// Cache interceptor
@Injectable()
export class CacheInterceptor implements NestInterceptor {
  private cache = new Map<string, any>();
  
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const cacheKey = \`\${request.method}:\${request.url}\`;
    
    if (request.method === 'GET' && this.cache.has(cacheKey)) {
      console.log('Cache hit:', cacheKey);
      return of(this.cache.get(cacheKey));
    }
    
    return next.handle().pipe(
      tap(response => {
        if (request.method === 'GET') {
          this.cache.set(cacheKey, response);
        }
      })
    );
  }
}

// Usage with decorators
@Controller('accounts')
@UseInterceptors(LoggingInterceptor, TransformInterceptor)
export class AccountsController {
  @Get(':id')
  @UseInterceptors(CacheInterceptor)
  async getAccount(@Param('id') id: string) {
    return this.accountService.findById(id);
  }
}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. CUSTOM MIDDLEWARE PATTERNS
// =============================================================================

console.log('4. ADVANCED PATTERNS:\n');

console.log('Middleware Composition:\n');
console.log(`
// Compose multiple middleware
const compose = (...middleware: Middleware[]): Middleware => {
  return (req, res, next) => {
    let index = 0;
    
    const dispatch = (i: number): void => {
      if (i >= middleware.length) {
        return next();
      }
      
      const fn = middleware[i];
      fn(req, res, () => dispatch(i + 1));
    };
    
    dispatch(0);
  };
};

// Usage
const authFlow = compose(
  authenticate,
  requireRole('ADMIN'),
  validateBody(schema)
);

app.post('/api/accounts', authFlow, accountController.create);

// Conditional middleware
const conditional = (
  condition: (req: Request) => boolean,
  middleware: Middleware
): Middleware => {
  return (req, res, next) => {
    if (condition(req)) {
      return middleware(req, res, next);
    }
    next();
  };
};

// Usage
const requireAuthForPrivate = conditional(
  (req) => req.path.startsWith('/api/private'),
  authenticate
);

app.use(requireAuthForPrivate);\n`);

console.log('Middleware Factory Pattern:\n');
console.log(`
interface MiddlewareConfig {
  enabled: boolean;
  options?: any;
}

class MiddlewareFactory {
  static createAuth(config: MiddlewareConfig): Middleware {
    if (!config.enabled) {
      return (req, res, next) => next();
    }
    
    return authenticate;
  }
  
  static createRateLimit(config: {
    enabled: boolean;
    maxRequests: number;
    windowMs: number;
  }): Middleware {
    if (!config.enabled) {
      return (req, res, next) => next();
    }
    
    return rateLimiter(config.maxRequests, config.windowMs);
  }
}

// Configuration-based middleware setup
const middlewareConfig = {
  auth: { enabled: true },
  rateLimit: { enabled: true, maxRequests: 100, windowMs: 60000 }
};

app.use(MiddlewareFactory.createAuth(middlewareConfig.auth));
app.use(MiddlewareFactory.createRateLimit(middlewareConfig.rateLimit));\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// MIDDLEWARE SUMMARY
// =============================================================================

console.log('MIDDLEWARE SUMMARY:\n');

console.log('Express Middleware:');
console.log('  • Authentication & authorization');
console.log('  • Request logging');
console.log('  • Validation');
console.log('  • Rate limiting');
console.log('  • Error handling\n');

console.log('NestJS Interceptors:');
console.log('  • Transform responses');
console.log('  • Logging & monitoring');
console.log('  • Caching');
console.log('  • Exception mapping');
console.log('  • Timeout handling\n');

console.log('Best Practices:');
console.log('  ✓ Type all middleware properly');
console.log('  ✓ Use async/await with asyncHandler');
console.log('  ✓ Create custom error classes');
console.log('  ✓ Implement proper error handling');
console.log('  ✓ Use middleware factories');
console.log('  ✓ Compose middleware logically');
console.log('  ✓ Add request context (ID, timing)');
console.log('  ✓ Log important events');

console.log();
```

---

## Question 17: How do you handle environment configuration and secrets in TypeScript?

### Answer:

**Configuration Management** is critical for managing different environments (dev, staging, production) and securing sensitive data. TypeScript provides type-safe configuration patterns.

### Key Concepts:
1. **Environment Variables** - .env files
2. **Type-safe Config** - Validated configuration
3. **Secret Management** - Secure sensitive data
4. **Multiple Environments** - Dev/staging/production
5. **Configuration Schema** - Validation

### Banking Scenario: Configuration at Emirates NBD

```typescript
console.log('=== Environment Configuration - Emirates NBD ===\n');

// =============================================================================
// 1. ENVIRONMENT VARIABLES
// =============================================================================

console.log('1. ENVIRONMENT VARIABLES:\n');

console.log('.env file structure:\n');
console.log(`
# Application
NODE_ENV=development
PORT=3000
APP_NAME=Emirates NBD Banking API
API_VERSION=v1

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/banking
DATABASE_POOL_MIN=2
DATABASE_POOL_MAX=10

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0

# JWT
JWT_SECRET=your-secret-key-here
JWT_EXPIRY=24h
JWT_REFRESH_SECRET=refresh-secret-key
JWT_REFRESH_EXPIRY=7d

# External APIs
PAYMENT_GATEWAY_URL=https://api.payment.com
PAYMENT_GATEWAY_KEY=pk_test_123456
PAYMENT_GATEWAY_SECRET=sk_test_789012

# AWS
AWS_REGION=eu-west-1
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
AWS_S3_BUCKET=emirates-nbd-documents

# Monitoring
SENTRY_DSN=https://key@sentry.io/project
LOG_LEVEL=debug

# Feature Flags
ENABLE_TRANSFER_FEATURE=true
ENABLE_INTERNATIONAL_TRANSFER=false
MAX_TRANSFER_AMOUNT=100000\n`);

console.log('Multiple environment files:');
console.log('  .env              - Default values');
console.log('  .env.development  - Development overrides');
console.log('  .env.staging      - Staging configuration');
console.log('  .env.production   - Production configuration');
console.log('  .env.local        - Local overrides (git-ignored)\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. TYPE-SAFE CONFIGURATION
// =============================================================================

console.log('2. TYPE-SAFE CONFIGURATION:\n');

console.log('Configuration Types & Validation:\n');
console.log(`
import { z } from 'zod';
import dotenv from 'dotenv';

// Load environment variables
dotenv.config();

// Configuration schema with validation
const configSchema = z.object({
  // App
  nodeEnv: z.enum(['development', 'staging', 'production', 'test']),
  port: z.number().int().positive(),
  appName: z.string(),
  apiVersion: z.string(),
  
  // Database
  database: z.object({
    url: z.string().url(),
    poolMin: z.number().int().min(1),
    poolMax: z.number().int().min(1)
  }),
  
  // Redis
  redis: z.object({
    host: z.string(),
    port: z.number().int(),
    password: z.string().optional(),
    db: z.number().int().default(0)
  }),
  
  // JWT
  jwt: z.object({
    secret: z.string().min(32),
    expiry: z.string(),
    refreshSecret: z.string().min(32),
    refreshExpiry: z.string()
  }),
  
  // Payment Gateway
  paymentGateway: z.object({
    url: z.string().url(),
    apiKey: z.string(),
    apiSecret: z.string()
  }),
  
  // AWS
  aws: z.object({
    region: z.string(),
    accessKeyId: z.string(),
    secretAccessKey: z.string(),
    s3Bucket: z.string()
  }),
  
  // Monitoring
  monitoring: z.object({
    sentryDsn: z.string().url().optional(),
    logLevel: z.enum(['debug', 'info', 'warn', 'error'])
  }),
  
  // Feature Flags
  features: z.object({
    enableTransfer: z.boolean(),
    enableInternationalTransfer: z.boolean(),
    maxTransferAmount: z.number().positive()
  })
});

// Parse and validate
function loadConfig() {
  const rawConfig = {
    nodeEnv: process.env.NODE_ENV,
    port: parseInt(process.env.PORT || '3000', 10),
    appName: process.env.APP_NAME,
    apiVersion: process.env.API_VERSION,
    
    database: {
      url: process.env.DATABASE_URL,
      poolMin: parseInt(process.env.DATABASE_POOL_MIN || '2', 10),
      poolMax: parseInt(process.env.DATABASE_POOL_MAX || '10', 10)
    },
    
    redis: {
      host: process.env.REDIS_HOST || 'localhost',
      port: parseInt(process.env.REDIS_PORT || '6379', 10),
      password: process.env.REDIS_PASSWORD || undefined,
      db: parseInt(process.env.REDIS_DB || '0', 10)
    },
    
    jwt: {
      secret: process.env.JWT_SECRET!,
      expiry: process.env.JWT_EXPIRY || '24h',
      refreshSecret: process.env.JWT_REFRESH_SECRET!,
      refreshExpiry: process.env.JWT_REFRESH_EXPIRY || '7d'
    },
    
    paymentGateway: {
      url: process.env.PAYMENT_GATEWAY_URL!,
      apiKey: process.env.PAYMENT_GATEWAY_KEY!,
      apiSecret: process.env.PAYMENT_GATEWAY_SECRET!
    },
    
    aws: {
      region: process.env.AWS_REGION!,
      accessKeyId: process.env.AWS_ACCESS_KEY_ID!,
      secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!,
      s3Bucket: process.env.AWS_S3_BUCKET!
    },
    
    monitoring: {
      sentryDsn: process.env.SENTRY_DSN,
      logLevel: process.env.LOG_LEVEL || 'info'
    },
    
    features: {
      enableTransfer: process.env.ENABLE_TRANSFER_FEATURE === 'true',
      enableInternationalTransfer: 
        process.env.ENABLE_INTERNATIONAL_TRANSFER === 'true',
      maxTransferAmount: 
        parseInt(process.env.MAX_TRANSFER_AMOUNT || '100000', 10)
    }
  };
  
  try {
    const config = configSchema.parse(rawConfig);
    console.log('✓ Configuration loaded successfully');
    return config;
  } catch (error) {
    console.error('❌ Configuration validation failed:', error);
    process.exit(1);
  }
}

// Export typed configuration
export const config = loadConfig();
export type Config = z.infer<typeof configSchema>;

// Usage
import { config } from './config';

console.log(\`Running in \${config.nodeEnv} mode\`);
console.log(\`Port: \${config.port}\`);
console.log(\`Database: \${config.database.url}\`);

if (config.features.enableTransfer) {
  console.log('Transfer feature enabled');
}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. CONFIGURATION SERVICE
// =============================================================================

console.log('3. CONFIGURATION SERVICE:\n');

console.log('Centralized Config Service:\n');
console.log(`
class ConfigService {
  private static instance: ConfigService;
  private config: Config;
  
  private constructor() {
    this.config = loadConfig();
  }
  
  static getInstance(): ConfigService {
    if (!ConfigService.instance) {
      ConfigService.instance = new ConfigService();
    }
    return ConfigService.instance;
  }
  
  get<K extends keyof Config>(key: K): Config[K] {
    return this.config[key];
  }
  
  isDevelopment(): boolean {
    return this.config.nodeEnv === 'development';
  }
  
  isProduction(): boolean {
    return this.config.nodeEnv === 'production';
  }
  
  isTest(): boolean {
    return this.config.nodeEnv === 'test';
  }
  
  getDatabaseUrl(): string {
    return this.config.database.url;
  }
  
  getJwtSecret(): string {
    return this.config.jwt.secret;
  }
  
  isFeatureEnabled(feature: keyof Config['features']): boolean {
    return this.config.features[feature];
  }
  
  getAwsConfig() {
    return this.config.aws;
  }
}

// Export singleton instance
export const configService = ConfigService.getInstance();

// Usage
import { configService } from './config.service';

if (configService.isDevelopment()) {
  console.log('Development mode');
}

const dbUrl = configService.getDatabaseUrl();
const jwtSecret = configService.getJwtSecret();

if (configService.isFeatureEnabled('enableTransfer')) {
  // Enable transfer feature
}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. SECRET MANAGEMENT
// =============================================================================

console.log('4. SECRET MANAGEMENT:\n');

console.log('AWS Secrets Manager Integration:\n');
console.log(`
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

class SecretManager {
  private client: SecretsManagerClient;
  private cache = new Map<string, any>();
  
  constructor() {
    this.client = new SecretsManagerClient({
      region: config.aws.region
    });
  }
  
  async getSecret(secretName: string): Promise<any> {
    // Check cache first
    if (this.cache.has(secretName)) {
      return this.cache.get(secretName);
    }
    
    try {
      const command = new GetSecretValueCommand({
        SecretId: secretName
      });
      
      const response = await this.client.send(command);
      const secret = JSON.parse(response.SecretString || '{}');
      
      // Cache the secret
      this.cache.set(secretName, secret);
      
      return secret;
    } catch (error) {
      console.error(\`Failed to fetch secret \${secretName}:, error\`);
      throw error;
    }
  }
  
  async getDatabaseCredentials(): Promise<{
    username: string;
    password: string;
    host: string;
    port: number;
  }> {
    return this.getSecret('banking-api/database');
  }
  
  async getJwtSecrets(): Promise<{
    secret: string;
    refreshSecret: string;
  }> {
    return this.getSecret('banking-api/jwt');
  }
}

export const secretManager = new SecretManager();\n`);

console.log('Environment-Specific Secrets:\n');
console.log(`
// config/secrets.ts
export async function loadSecrets() {
  if (config.isProduction()) {
    // Production: Use AWS Secrets Manager
    const dbCreds = await secretManager.getDatabaseCredentials();
    const jwtSecrets = await secretManager.getJwtSecrets();
    
    return {
      database: {
        ...config.database,
        username: dbCreds.username,
        password: dbCreds.password
      },
      jwt: {
        ...config.jwt,
        secret: jwtSecrets.secret,
        refreshSecret: jwtSecrets.refreshSecret
      }
    };
  } else {
    // Development: Use .env file
    return {
      database: config.database,
      jwt: config.jwt
    };
  }
}\n`);

console.log('Encryption for Sensitive Config:\n');
console.log(`
import crypto from 'crypto';

class ConfigEncryption {
  private algorithm = 'aes-256-gcm';
  private key: Buffer;
  
  constructor(masterKey: string) {
    // Derive encryption key from master key
    this.key = crypto.scryptSync(masterKey, 'salt', 32);
  }
  
  encrypt(text: string): string {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv(this.algorithm, this.key, iv);
    
    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    const authTag = cipher.getAuthTag();
    
    return \`\${iv.toString('hex')}:\${authTag.toString('hex')}:\${encrypted}\`;
  }
  
  decrypt(encrypted: string): string {
    const [ivHex, authTagHex, encryptedText] = encrypted.split(':');
    
    const iv = Buffer.from(ivHex, 'hex');
    const authTag = Buffer.from(authTagHex, 'hex');
    
    const decipher = crypto.createDecipheriv(this.algorithm, this.key, iv);
    decipher.setAuthTag(authTag);
    
    let decrypted = decipher.update(encryptedText, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
}

// Usage
const encryption = new ConfigEncryption(process.env.MASTER_KEY!);

// Encrypt sensitive config
const encryptedApiKey = encryption.encrypt(config.paymentGateway.apiSecret);

// Decrypt when needed
const apiSecret = encryption.decrypt(encryptedApiKey);\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// CONFIGURATION SUMMARY
// =============================================================================

console.log('CONFIGURATION SUMMARY:\n');

console.log('Best Practices:');
console.log('  ✓ Validate all configuration on startup');
console.log('  ✓ Use type-safe configuration objects');
console.log('  ✓ Never commit secrets to git');
console.log('  ✓ Use environment-specific config files');
console.log('  ✓ Implement secret rotation');
console.log('  ✓ Use secret managers in production');
console.log('  ✓ Encrypt sensitive data at rest');
console.log('  ✓ Document all config variables\n');

console.log('Configuration Strategy:');
console.log('  • Development: .env files');
console.log('  • Staging: Environment variables + Vault');
console.log('  • Production: AWS Secrets Manager/Azure Key Vault');
console.log('  • CI/CD: GitHub Secrets/GitLab Variables\n');

console.log('Security:');
console.log('  • Use strong encryption for secrets');
console.log('  • Rotate secrets regularly');
console.log('  • Limit secret access');
console.log('  • Audit secret usage');
console.log('  • Use least privilege principle');

console.log();
```

### Key Takeaways:

**ORM Integration**:
- TypeORM with decorators and query builder
- Prisma with schema-first approach and auto-generated types
- Type-safe queries and relations
- Transaction management
- Performance optimization with indexes

**Middleware & Interceptors**:
- Express middleware for authentication, validation, logging
- NestJS interceptors for transforming data flow
- Custom error classes and error handling
- Middleware composition and factories
- Type-safe request/response handling

**Configuration Management**:
- Type-safe environment variables with validation
- Configuration service pattern
- Secret management with AWS Secrets Manager
- Environment-specific configurations
- Encryption for sensitive data

---

**End of Questions 15-17**
