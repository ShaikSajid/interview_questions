# Questions 4-6: TypeScript in Practice

## Question 4: Explain TypeScript decorators and their practical applications

### Answer:

**Decorators** are special declarations that can be attached to classes, methods, properties, or parameters to modify their behavior. They provide a way to add metadata and functionality through declarative syntax, commonly used in frameworks like Angular and NestJS.

### Key Decorator Types:
1. **Class Decorators** - Modify class definitions
2. **Method Decorators** - Modify method behavior
3. **Property Decorators** - Add metadata to properties
4. **Parameter Decorators** - Annotate method parameters
5. **Accessor Decorators** - Modify getters/setters

### Banking Scenario: Decorators in Banking System at Emirates NBD

```typescript
console.log('=== TypeScript Decorators - Emirates NBD ===\n');

// Enable experimentalDecorators in tsconfig.json

// =============================================================================
// 1. METHOD DECORATORS - LOGGING
// =============================================================================

console.log('1. METHOD DECORATORS:\n');

function LogMethod(target: any, propertyName: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;
    
    descriptor.value = function (...args: any[]) {
        console.log(`  [LOG] Calling ${propertyName} with args: ${JSON.stringify(args)}`);
        const result = originalMethod.apply(this, args);
        console.log(`  [LOG] ${propertyName} returned: ${JSON.stringify(result)}`);
        return result;
    };
    
    return descriptor;
}

function Measure(target: any, propertyName: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;
    
    descriptor.value = function (...args: any[]) {
        const start = performance.now();
        const result = originalMethod.apply(this, args);
        const end = performance.now();
        console.log(`  [PERF] ${propertyName} took ${(end - start).toFixed(2)}ms`);
        return result;
    };
    
    return descriptor;
}

class BankingService {
    @LogMethod
    @Measure
    calculateInterest(principal: number, rate: number, years: number): number {
        // Simulate some processing
        let result = principal;
        for (let i = 0; i < years; i++) {
            result *= (1 + rate);
        }
        return result - principal;
    }
    
    @LogMethod
    validateAccount(accountNumber: string): boolean {
        return /^ACC-\d{9}$/.test(accountNumber);
    }
}

const service = new BankingService();
console.log('Method Decorators:\n');
const interest = service.calculateInterest(100000, 0.035, 5);
console.log(`  Interest calculated: ${interest.toFixed(2)} AED\n`);

const isValid = service.validateAccount('ACC-123456789');
console.log(`  Validation result: ${isValid}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. CLASS DECORATORS - METADATA
// =============================================================================

console.log('2. CLASS DECORATORS:\n');

function Entity(tableName: string) {
    return function <T extends { new(...args: any[]): {} }>(constructor: T) {
        return class extends constructor {
            tableName = tableName;
            createdAt = new Date();
            
            getMetadata() {
                return {
                    table: this.tableName,
                    created: this.createdAt
                };
            }
        };
    };
}

function Singleton<T extends { new(...args: any[]): {} }>(constructor: T) {
    let instance: T | null = null;
    
    return class extends constructor {
        constructor(...args: any[]) {
            if (instance) {
                throw new Error('Singleton class already instantiated');
            }
            super(...args);
            instance = this as any;
        }
    } as T;
}

@Entity('bank_accounts')
class Account {
    constructor(
        public accountNumber: string,
        public balance: number
    ) {}
}

const account1: any = new Account('ACC-123456789', 50000);
console.log('Class Decorator:\n');
console.log(`  Account: ${account1.accountNumber}`);
console.log(`  Metadata: ${JSON.stringify(account1.getMetadata())}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. PROPERTY DECORATORS - VALIDATION
// =============================================================================

console.log('3. PROPERTY DECORATORS:\n');

function MinValue(min: number) {
    return function (target: any, propertyKey: string) {
        let value: number;
        
        const getter = function () {
            return value;
        };
        
        const setter = function (newValue: number) {
            if (newValue < min) {
                throw new Error(`${propertyKey} must be at least ${min}`);
            }
            value = newValue;
        };
        
        Object.defineProperty(target, propertyKey, {
            get: getter,
            set: setter,
            enumerable: true,
            configurable: true
        });
    };
}

function MaxLength(max: number) {
    return function (target: any, propertyKey: string) {
        let value: string;
        
        const getter = function () {
            return value;
        };
        
        const setter = function (newValue: string) {
            if (newValue.length > max) {
                throw new Error(`${propertyKey} cannot exceed ${max} characters`);
            }
            value = newValue;
        };
        
        Object.defineProperty(target, propertyKey, {
            get: getter,
            set: setter,
            enumerable: true,
            configurable: true
        });
    };
}

class Transaction {
    @MinValue(0)
    amount: number = 0;
    
    @MaxLength(50)
    description: string = '';
    
    constructor(amount: number, description: string) {
        this.amount = amount;
        this.description = description;
    }
}

console.log('Property Decorators:\n');

try {
    const txn1 = new Transaction(5000, 'Salary deposit');
    console.log(`  ✓ Valid transaction: ${txn1.amount} - ${txn1.description}\n`);
    
    const txn2 = new Transaction(-1000, 'Invalid');
} catch (error: any) {
    console.log(`  ✗ Validation error: ${error.message}\n`);
}

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. AUTHORIZATION DECORATOR
// =============================================================================

console.log('4. AUTHORIZATION DECORATOR:\n');

function RequireRole(role: string) {
    return function (target: any, propertyName: string, descriptor: PropertyDescriptor) {
        const originalMethod = descriptor.value;
        
        descriptor.value = function (this: any, ...args: any[]) {
            if (!this.currentUser || !this.currentUser.roles.includes(role)) {
                throw new Error(`Access denied. Required role: ${role}`);
            }
            console.log(`  ✓ Authorization passed for role: ${role}`);
            return originalMethod.apply(this, args);
        };
        
        return descriptor;
    };
}

function Authenticated(target: any, propertyName: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;
    
    descriptor.value = function (this: any, ...args: any[]) {
        if (!this.currentUser) {
            throw new Error('Authentication required');
        }
        console.log(`  ✓ User authenticated: ${this.currentUser.email}`);
        return originalMethod.apply(this, args);
    };
    
    return descriptor;
}

interface User {
    email: string;
    roles: string[];
}

class AccountManager {
    currentUser?: User;
    
    setUser(user: User) {
        this.currentUser = user;
    }
    
    @Authenticated
    @RequireRole('ADMIN')
    deleteAccount(accountNumber: string): void {
        console.log(`  Deleting account: ${accountNumber}`);
    }
    
    @Authenticated
    @RequireRole('MANAGER')
    approveTransaction(transactionId: string): void {
        console.log(`  Approving transaction: ${transactionId}`);
    }
    
    @Authenticated
    viewBalance(accountNumber: string): number {
        console.log(`  Viewing balance for: ${accountNumber}`);
        return 50000;
    }
}

const manager = new AccountManager();
manager.setUser({ email: 'admin@emiratesnbd.com', roles: ['ADMIN', 'MANAGER'] });

console.log('Authorization Decorators:\n');
manager.deleteAccount('ACC-123456789');
console.log();
manager.approveTransaction('TXN-001');
console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. CACHING DECORATOR
// =============================================================================

console.log('5. CACHING DECORATOR:\n');

function Memoize(target: any, propertyName: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;
    const cache = new Map<string, any>();
    
    descriptor.value = function (...args: any[]) {
        const key = JSON.stringify(args);
        
        if (cache.has(key)) {
            console.log(`  [CACHE HIT] ${propertyName}(${key})`);
            return cache.get(key);
        }
        
        console.log(`  [CACHE MISS] ${propertyName}(${key})`);
        const result = originalMethod.apply(this, args);
        cache.set(key, result);
        return result;
    };
    
    return descriptor;
}

class InterestCalculator {
    @Memoize
    calculateCompoundInterest(principal: number, rate: number, years: number): number {
        console.log(`    Computing interest...`);
        return principal * Math.pow(1 + rate, years) - principal;
    }
}

const calculator = new InterestCalculator();

console.log('Memoization Decorator:\n');
console.log(`Interest: ${calculator.calculateCompoundInterest(100000, 0.035, 5).toFixed(2)}\n`);
console.log(`Interest: ${calculator.calculateCompoundInterest(100000, 0.035, 5).toFixed(2)}\n`);
console.log(`Interest: ${calculator.calculateCompoundInterest(50000, 0.04, 3).toFixed(2)}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 6. RETRY DECORATOR
// =============================================================================

console.log('6. RETRY DECORATOR:\n');

function Retry(maxAttempts: number = 3, delay: number = 1000) {
    return function (target: any, propertyName: string, descriptor: PropertyDescriptor) {
        const originalMethod = descriptor.value;
        
        descriptor.value = async function (...args: any[]) {
            let lastError: Error | undefined;
            
            for (let attempt = 1; attempt <= maxAttempts; attempt++) {
                try {
                    console.log(`  Attempt ${attempt}/${maxAttempts}`);
                    return await originalMethod.apply(this, args);
                } catch (error) {
                    lastError = error as Error;
                    console.log(`  ✗ Failed: ${lastError.message}`);
                    
                    if (attempt < maxAttempts) {
                        console.log(`  Retrying in ${delay}ms...\n`);
                        await new Promise(resolve => setTimeout(resolve, delay));
                    }
                }
            }
            
            throw lastError;
        };
        
        return descriptor;
    };
}

class PaymentService {
    private attemptCount = 0;
    
    @Retry(3, 100)
    async processPayment(amount: number): Promise<{ success: boolean }> {
        this.attemptCount++;
        
        // Simulate failure for first 2 attempts
        if (this.attemptCount < 3) {
            throw new Error('Network timeout');
        }
        
        console.log(`  ✓ Payment processed: ${amount} AED\n`);
        return { success: true };
    }
}

(async () => {
    const paymentService = new PaymentService();
    
    console.log('Retry Decorator:\n');
    const result = await paymentService.processPayment(5000);
    console.log(`Result: ${JSON.stringify(result)}\n`);
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // DECORATORS SUMMARY
    // =============================================================================
    
    console.log('DECORATORS SUMMARY:\n');
    
    console.log('Decorator Types:');
    console.log('  • Class Decorators - Modify class definition');
    console.log('  • Method Decorators - Wrap method behavior');
    console.log('  • Property Decorators - Add property metadata');
    console.log('  • Parameter Decorators - Annotate parameters');
    console.log('  • Accessor Decorators - Modify getters/setters\n');
    
    console.log('Common Use Cases:');
    console.log('  • Logging - Log method calls and results');
    console.log('  • Validation - Validate input/properties');
    console.log('  • Authorization - Check permissions');
    console.log('  • Caching - Memoize expensive operations');
    console.log('  • Retry - Automatic retry on failure');
    console.log('  • Performance - Measure execution time\n');
    
    console.log('Frameworks Using Decorators:');
    console.log('  • Angular - @Component, @Injectable');
    console.log('  • NestJS - @Controller, @Get, @Post');
    console.log('  • TypeORM - @Entity, @Column');
    console.log('  • MobX - @observable, @action\n');
    
    console.log('Configuration:');
    console.log('  tsconfig.json:');
    console.log('  {');
    console.log('    "experimentalDecorators": true,');
    console.log('    "emitDecoratorMetadata": true');
    console.log('  }\n');
    
    console.log('Benefits:');
    console.log('  ✓ Declarative syntax');
    console.log('  ✓ Code reusability');
    console.log('  ✓ Separation of concerns');
    console.log('  ✓ Cross-cutting concerns');
    console.log('  ✓ Clean, readable code');
    
    console.log();
})();
```

---

## Question 5: How do you configure and use TypeScript with Node.js and Express for backend development?

### Answer:

**TypeScript with Node.js** provides type safety and better development experience for backend applications. Setting up TypeScript with Express requires proper configuration, type definitions, and project structure.

### Key Components:
1. **tsconfig.json** - TypeScript compiler configuration
2. **Type Definitions** - @types packages for libraries
3. **Project Structure** - Organized folder layout
4. **Build Process** - Compilation to JavaScript
5. **Development Workflow** - Hot reload and debugging

### Banking Scenario: TypeScript Express API at Emirates NBD

```typescript
console.log('=== TypeScript with Node.js/Express - Emirates NBD ===\n');

// =============================================================================
// 1. PROJECT SETUP
// =============================================================================

console.log('1. PROJECT SETUP:\n');

console.log('Installation commands:\n');
console.log('  npm init -y');
console.log('  npm install express');
console.log('  npm install -D typescript @types/node @types/express');
console.log('  npm install -D ts-node nodemon');
console.log('  npx tsc --init\n');

console.log('tsconfig.json:\n');
const tsConfig = {
    compilerOptions: {
        target: 'ES2020',
        module: 'commonjs',
        lib: ['ES2020'],
        outDir: './dist',
        rootDir: './src',
        strict: true,
        esModuleInterop: true,
        skipLibCheck: true,
        forceConsistentCasingInFileNames: true,
        resolveJsonModule: true,
        moduleResolution: 'node',
        experimentalDecorators: true,
        emitDecoratorMetadata: true
    },
    include: ['src/**/*'],
    exclude: ['node_modules', 'dist']
};

console.log(JSON.stringify(tsConfig, null, 2));
console.log();

console.log('package.json scripts:\n');
const scripts = {
    'dev': 'nodemon --exec ts-node src/server.ts',
    'build': 'tsc',
    'start': 'node dist/server.js',
    'test': 'jest'
};

Object.entries(scripts).forEach(([key, value]) => {
    console.log(`  "${key}": "${value}"`);
});
console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. TYPE-SAFE EXPRESS APPLICATION
// =============================================================================

console.log('2. TYPE-SAFE EXPRESS APP:\n');

// Simulated Express types and setup
interface Request<P = any, B = any, Q = any> {
    params: P;
    body: B;
    query: Q;
    headers: Record<string, string>;
    user?: {
        id: string;
        email: string;
        roles: string[];
    };
}

interface Response {
    status(code: number): Response;
    json(data: any): Response;
    send(data: any): Response;
}

type NextFunction = () => void;

type RequestHandler<P = any, B = any, Q = any> = (
    req: Request<P, B, Q>,
    res: Response,
    next: NextFunction
) => void | Promise<void>;

console.log('Express with TypeScript types:\n');

// DTOs (Data Transfer Objects)
interface CreateAccountDto {
    accountNumber: string;
    accountType: 'SAVINGS' | 'CURRENT' | 'FIXED';
    initialBalance: number;
    currency: string;
}

interface TransferDto {
    fromAccount: string;
    toAccount: string;
    amount: number;
    description?: string;
}

interface AccountResponseDto {
    id: string;
    accountNumber: string;
    accountType: string;
    balance: number;
    currency: string;
    createdAt: Date;
}

console.log('  ✓ DTOs defined for request/response types\n');

// Service layer with types
class AccountService {
    private accounts: Map<string, any> = new Map();
    
    async createAccount(dto: CreateAccountDto): Promise<AccountResponseDto> {
        const account: AccountResponseDto = {
            id: `ACC-${Date.now()}`,
            accountNumber: dto.accountNumber,
            accountType: dto.accountType,
            balance: dto.initialBalance,
            currency: dto.currency,
            createdAt: new Date()
        };
        
        this.accounts.set(account.id, account);
        console.log(`  Created account: ${account.accountNumber}`);
        return account;
    }
    
    async getAccount(id: string): Promise<AccountResponseDto | null> {
        return this.accounts.get(id) || null;
    }
    
    async getAllAccounts(): Promise<AccountResponseDto[]> {
        return Array.from(this.accounts.values());
    }
}

const accountService = new AccountService();

// Controller with typed handlers
class AccountController {
    createAccount: RequestHandler<{}, CreateAccountDto> = async (req, res) => {
        try {
            const account = await accountService.createAccount(req.body);
            res.status(201).json({
                success: true,
                data: account
            });
        } catch (error: any) {
            res.status(400).json({
                success: false,
                error: error.message
            });
        }
    };
    
    getAccount: RequestHandler<{ id: string }> = async (req, res) => {
        try {
            const account = await accountService.getAccount(req.params.id);
            
            if (!account) {
                res.status(404).json({
                    success: false,
                    error: 'Account not found'
                });
                return;
            }
            
            res.status(200).json({
                success: true,
                data: account
            });
        } catch (error: any) {
            res.status(500).json({
                success: false,
                error: error.message
            });
        }
    };
    
    getAllAccounts: RequestHandler = async (req, res) => {
        try {
            const accounts = await accountService.getAllAccounts();
            res.status(200).json({
                success: true,
                data: accounts,
                count: accounts.length
            });
        } catch (error: any) {
            res.status(500).json({
                success: false,
                error: error.message
            });
        }
    };
}

const accountController = new AccountController();

console.log('  ✓ Service and Controller with full type safety\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. MIDDLEWARE WITH TYPES
// =============================================================================

console.log('3. TYPED MIDDLEWARE:\n');

// Error handler
interface ApiError extends Error {
    statusCode?: number;
    code?: string;
}

const errorHandler: RequestHandler = (req, res, next) => {
    // Error handling middleware
    console.log('  Error handler registered');
};

// Authentication middleware
const authenticate: RequestHandler = (req, res, next) => {
    const token = req.headers['authorization'];
    
    if (!token) {
        res.status(401).json({
            success: false,
            error: 'Authentication required'
        });
        return;
    }
    
    // Simulate token verification
    req.user = {
        id: 'user-123',
        email: 'user@emiratesnbd.com',
        roles: ['USER']
    };
    
    console.log(`  Authenticated user: ${req.user.email}`);
    next();
};

// Authorization middleware
function authorize(...roles: string[]): RequestHandler {
    return (req, res, next) => {
        if (!req.user) {
            res.status(401).json({
                success: false,
                error: 'Authentication required'
            });
            return;
        }
        
        const hasRole = roles.some(role => req.user!.roles.includes(role));
        
        if (!hasRole) {
            res.status(403).json({
                success: false,
                error: 'Insufficient permissions'
            });
            return;
        }
        
        console.log(`  Authorized with roles: ${roles.join(', ')}`);
        next();
    };
}

// Validation middleware
function validateBody<T>(schema: (data: any) => T): RequestHandler<any, any> {
    return (req, res, next) => {
        try {
            req.body = schema(req.body);
            console.log('  Request body validated');
            next();
        } catch (error: any) {
            res.status(400).json({
                success: false,
                error: 'Validation failed',
                details: error.message
            });
        }
    };
}

console.log('  ✓ Typed middleware for auth, validation, errors\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. API ROUTES STRUCTURE
// =============================================================================

console.log('4. API ROUTES:\n');

console.log('Project structure:\n');
console.log('src/');
console.log('├── controllers/');
console.log('│   ├── account.controller.ts');
console.log('│   └── transaction.controller.ts');
console.log('├── services/');
console.log('│   ├── account.service.ts');
console.log('│   └── transaction.service.ts');
console.log('├── models/');
console.log('│   ├── account.model.ts');
console.log('│   └── transaction.model.ts');
console.log('├── dto/');
console.log('│   ├── create-account.dto.ts');
console.log('│   └── transfer.dto.ts');
console.log('├── middleware/');
console.log('│   ├── auth.middleware.ts');
console.log('│   ├── validation.middleware.ts');
console.log('│   └── error.middleware.ts');
console.log('├── routes/');
console.log('│   ├── account.routes.ts');
console.log('│   └── transaction.routes.ts');
console.log('├── config/');
console.log('│   ├── database.ts');
console.log('│   └── env.ts');
console.log('├── utils/');
console.log('│   ├── logger.ts');
console.log('│   └── validators.ts');
console.log('└── server.ts\n');

console.log('Route definitions:\n');

interface RouteDefinition {
    method: 'GET' | 'POST' | 'PUT' | 'DELETE';
    path: string;
    handler: string;
    middleware: string[];
}

const routes: RouteDefinition[] = [
    {
        method: 'POST',
        path: '/api/accounts',
        handler: 'accountController.createAccount',
        middleware: ['authenticate', 'authorize(ADMIN)']
    },
    {
        method: 'GET',
        path: '/api/accounts/:id',
        handler: 'accountController.getAccount',
        middleware: ['authenticate']
    },
    {
        method: 'GET',
        path: '/api/accounts',
        handler: 'accountController.getAllAccounts',
        middleware: ['authenticate', 'authorize(ADMIN)']
    },
    {
        method: 'POST',
        path: '/api/transactions/transfer',
        handler: 'transactionController.transfer',
        middleware: ['authenticate', 'validateBody']
    }
];

routes.forEach(route => {
    console.log(`  ${route.method.padEnd(6)} ${route.path}`);
    console.log(`         Handler: ${route.handler}`);
    console.log(`         Middleware: [${route.middleware.join(', ')}]\n`);
});

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. ENVIRONMENT AND CONFIGURATION
// =============================================================================

console.log('5. CONFIGURATION:\n');

// Environment variables with types
interface EnvironmentConfig {
    NODE_ENV: 'development' | 'production' | 'test';
    PORT: number;
    DATABASE_URL: string;
    JWT_SECRET: string;
    API_VERSION: string;
}

class ConfigService {
    private config: EnvironmentConfig;
    
    constructor() {
        this.config = {
            NODE_ENV: (process.env.NODE_ENV as any) || 'development',
            PORT: parseInt(process.env.PORT || '3000'),
            DATABASE_URL: process.env.DATABASE_URL || 'mongodb://localhost:27017/banking',
            JWT_SECRET: process.env.JWT_SECRET || 'secret-key',
            API_VERSION: 'v1'
        };
        
        this.validate();
    }
    
    private validate(): void {
        console.log('  Validating configuration...');
        
        if (!this.config.JWT_SECRET || this.config.JWT_SECRET === 'secret-key') {
            console.warn('  ⚠ WARNING: Using default JWT secret');
        }
        
        console.log('  ✓ Configuration validated\n');
    }
    
    get<K extends keyof EnvironmentConfig>(key: K): EnvironmentConfig[K] {
        return this.config[key];
    }
    
    getAll(): Readonly<EnvironmentConfig> {
        return { ...this.config };
    }
}

const config = new ConfigService();
console.log('Environment Configuration:');
console.log(`  NODE_ENV: ${config.get('NODE_ENV')}`);
console.log(`  PORT: ${config.get('PORT')}`);
console.log(`  API_VERSION: ${config.get('API_VERSION')}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 6. TESTING WITH TYPES
// =============================================================================

console.log('6. TESTING:\n');

console.log('Test setup:\n');
console.log('  npm install -D jest @types/jest ts-jest supertest @types/supertest\n');

console.log('jest.config.js:\n');
console.log(`module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src'],
  testMatch: ['**/*.test.ts'],
  collectCoverageFrom: ['src/**/*.ts', '!src/**/*.d.ts']
};\n`);

console.log('Example test:\n');
console.log(`import { AccountService } from './account.service';

describe('AccountService', () => {
  let service: AccountService;
  
  beforeEach(() => {
    service = new AccountService();
  });
  
  describe('createAccount', () => {
    it('should create account with valid data', async () => {
      const dto = {
        accountNumber: 'ACC-123456789',
        accountType: 'SAVINGS' as const,
        initialBalance: 50000,
        currency: 'AED'
      };
      
      const account = await service.createAccount(dto);
      
      expect(account.accountNumber).toBe(dto.accountNumber);
      expect(account.balance).toBe(dto.initialBalance);
    });
    
    it('should throw error for negative balance', async () => {
      const dto = {
        accountNumber: 'ACC-123456789',
        accountType: 'SAVINGS' as const,
        initialBalance: -1000,
        currency: 'AED'
      };
      
      await expect(service.createAccount(dto)).rejects.toThrow();
    });
  });
});\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// TYPESCRIPT NODE.JS SUMMARY
// =============================================================================

console.log('TYPESCRIPT NODE.JS SUMMARY:\n');

console.log('Setup Steps:');
console.log('  1. Initialize TypeScript project');
console.log('  2. Install type definitions (@types/*)');
console.log('  3. Configure tsconfig.json');
console.log('  4. Setup build and dev scripts');
console.log('  5. Organize project structure\n');

console.log('Key Files:');
console.log('  • tsconfig.json - Compiler options');
console.log('  • package.json - Scripts and dependencies');
console.log('  • .env - Environment variables');
console.log('  • jest.config.js - Test configuration\n');

console.log('Type Safety:');
console.log('  • Request/Response types');
console.log('  • DTOs for data validation');
console.log('  • Service layer types');
console.log('  • Middleware types');
console.log('  • Configuration types\n');

console.log('Development Workflow:');
console.log('  • npm run dev - Development with hot reload');
console.log('  • npm run build - Compile to JavaScript');
console.log('  • npm start - Run production build');
console.log('  • npm test - Run tests\n');

console.log('Benefits:');
console.log('  ✓ Compile-time error detection');
console.log('  ✓ Better IDE support');
console.log('  ✓ Self-documenting APIs');
console.log('  ✓ Easier refactoring');
console.log('  ✓ Improved maintainability');

console.log();
```

---

## Question 6: Explain TypeScript's type system edge cases, challenges, and best practices

### Answer:

**TypeScript's Type System** is powerful but has edge cases and challenges that developers should understand. Knowing these helps write more robust and maintainable code.

### Key Topics:
1. **Type Assertion Pitfalls** - Risks of overriding types
2. **Any vs Unknown** - Appropriate use cases
3. **Type Narrowing** - Runtime type checks
4. **Structural Typing** - Duck typing implications
5. **Generic Constraints** - Complex type relationships

### Banking Scenario: Type System Best Practices at Emirates NBD

```typescript
console.log('=== TypeScript Edge Cases & Best Practices - Emirates NBD ===\n');

// =============================================================================
// 1. ANY VS UNKNOWN
// =============================================================================

console.log('1. ANY VS UNKNOWN:\n');

// ❌ BAD: Using 'any' loses all type safety
function processDataBad(data: any): number {
    return data.balance.toFixed(2); // No type checking, can crash
}

// ✅ GOOD: Using 'unknown' requires type checking
function processDataGood(data: unknown): number {
    if (typeof data === 'object' && data !== null && 'balance' in data) {
        const account = data as { balance: number };
        return account.balance;
    }
    throw new Error('Invalid data');
}

console.log('Any vs Unknown:\n');
console.log('  ✗ any: No type checking, unsafe');
console.log('  ✓ unknown: Requires type narrowing, safe\n');

const testData: unknown = { balance: 50000 };
try {
    const balance = processDataGood(testData);
    console.log(`  Balance: ${balance}\n`);
} catch (error: any) {
    console.log(`  Error: ${error.message}\n`);
}

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. TYPE ASSERTIONS AND GUARDS
// =============================================================================

console.log('2. TYPE ASSERTIONS AND GUARDS:\n');

interface SavingsAccount {
    type: 'savings';
    interestRate: number;
    minimumBalance: number;
}

interface CurrentAccount {
    type: 'current';
    overdraftLimit: number;
    monthlyFee: number;
}

type Account = SavingsAccount | CurrentAccount;

// ❌ BAD: Type assertion without validation
function getBadInterestRate(account: Account): number {
    const savings = account as SavingsAccount; // Unsafe!
    return savings.interestRate; // May be undefined
}

// ✅ GOOD: Type guard
function isSavingsAccount(account: Account): account is SavingsAccount {
    return account.type === 'savings';
}

function getGoodInterestRate(account: Account): number {
    if (isSavingsAccount(account)) {
        return account.interestRate; // TypeScript knows type
    }
    throw new Error('Not a savings account');
}

console.log('Type Guards:\n');

const savings: Account = {
    type: 'savings',
    interestRate: 0.035,
    minimumBalance: 1000
};

const current: Account = {
    type: 'current',
    overdraftLimit: 10000,
    monthlyFee: 50
};

if (isSavingsAccount(savings)) {
    console.log(`  ✓ Savings interest rate: ${savings.interestRate * 100}%`);
}

if (!isSavingsAccount(current)) {
    console.log(`  ✓ Current account overdraft: ${current.overdraftLimit}\n`);
}

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. STRUCTURAL VS NOMINAL TYPING
// =============================================================================

console.log('3. STRUCTURAL TYPING:\n');

interface AccountId {
    value: string;
}

interface TransactionId {
    value: string;
}

// TypeScript uses structural typing - these are compatible!
const accountId: AccountId = { value: 'ACC-123' };
const transactionId: TransactionId = accountId; // No error!

console.log('Structural Typing Issue:');
console.log(`  AccountId and TransactionId are structurally identical`);
console.log(`  TypeScript allows assignment between them\n`);

// ✅ SOLUTION: Branded types (nominal typing simulation)
type Brand<K, T> = K & { __brand: T };

type BrandedAccountId = Brand<string, 'AccountId'>;
type BrandedTransactionId = Brand<string, 'TransactionId'>;

function createAccountId(id: string): BrandedAccountId {
    return id as BrandedAccountId;
}

function createTransactionId(id: string): BrandedTransactionId {
    return id as BrandedTransactionId;
}

const brandedAccId = createAccountId('ACC-123');
const brandedTxnId = createTransactionId('TXN-456');

// const wrong: BrandedAccountId = brandedTxnId; // Type error!

console.log('Branded Types Solution:');
console.log(`  Account ID: ${brandedAccId}`);
console.log(`  Transaction ID: ${brandedTxnId}`);
console.log(`  ✓ Cannot be assigned to each other\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. COMPLEX GENERIC CONSTRAINTS
// =============================================================================

console.log('4. GENERIC CONSTRAINTS:\n');

// Ensure type has required methods
interface Comparable<T> {
    compareTo(other: T): number;
}

class Money implements Comparable<Money> {
    constructor(
        public amount: number,
        public currency: string
    ) {}
    
    compareTo(other: Money): number {
        if (this.currency !== other.currency) {
            throw new Error('Cannot compare different currencies');
        }
        return this.amount - other.amount;
    }
}

function findMax<T extends Comparable<T>>(items: T[]): T | undefined {
    if (items.length === 0) return undefined;
    
    let max = items[0];
    for (const item of items) {
        if (item.compareTo(max) > 0) {
            max = item;
        }
    }
    return max;
}

const balances = [
    new Money(50000, 'AED'),
    new Money(75000, 'AED'),
    new Money(30000, 'AED')
];

const maxBalance = findMax(balances);
console.log('Generic with Constraints:');
console.log(`  Maximum balance: ${maxBalance?.amount} ${maxBalance?.currency}\n`);

// Conditional types for complex scenarios
type GetReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

function calculateInterest(principal: number): number {
    return principal * 0.035;
}

type InterestReturn = GetReturnType<typeof calculateInterest>; // number

console.log('Conditional Types:');
console.log(`  Extracted return type: number\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. READONLY AND IMMUTABILITY
// =============================================================================

console.log('5. READONLY AND IMMUTABILITY:\n');

interface Transaction {
    readonly id: string;
    readonly timestamp: Date;
    amount: number;
}

const txn: Transaction = {
    id: 'TXN-001',
    timestamp: new Date(),
    amount: 5000
};

// txn.id = 'TXN-002'; // Error: Cannot assign to readonly property
txn.amount = 6000; // OK

console.log('Readonly Properties:');
console.log(`  ID is readonly: ${txn.id}`);
console.log(`  Amount can change: ${txn.amount}\n`);

// Deep readonly
type DeepReadonly<T> = {
    readonly [P in keyof T]: T[P] extends object ? DeepReadonly<T[P]> : T[P];
};

interface Account {
    accountNumber: string;
    balance: number;
    details: {
        type: string;
        branch: string;
    };
}

type ImmutableAccount = DeepReadonly<Account>;

const immutableAccount: ImmutableAccount = {
    accountNumber: 'ACC-123',
    balance: 50000,
    details: {
        type: 'SAVINGS',
        branch: 'Dubai Mall'
    }
};

// immutableAccount.details.type = 'CURRENT'; // Error: Readonly

console.log('Deep Readonly:');
console.log(`  All nested properties are readonly\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 6. BEST PRACTICES
// =============================================================================

console.log('6. BEST PRACTICES:\n');

// ✅ Use strict mode
console.log('Enable Strict Mode:');
console.log(`  tsconfig.json:
  {
    "compilerOptions": {
      "strict": true,
      "noImplicitAny": true,
      "strictNullChecks": true,
      "strictFunctionTypes": true
    }
  }\n`);

// ✅ Prefer interfaces over type aliases for objects
interface BankAccount {
    accountNumber: string;
    balance: number;
}

// ✅ Use type aliases for unions and primitives
type AccountType = 'SAVINGS' | 'CURRENT' | 'FIXED';
type TransactionStatus = 'PENDING' | 'COMPLETED' | 'FAILED';

console.log('Interface vs Type:');
console.log('  ✓ interface - For objects (can be extended)');
console.log('  ✓ type - For unions, primitives, utilities\n');

// ✅ Avoid type assertions when possible
interface ApiResponse<T> {
    data: T;
    error?: string;
}

function handleResponse<T>(response: ApiResponse<T>): T {
    if (response.error) {
        throw new Error(response.error);
    }
    return response.data; // Type-safe, no assertion needed
}

console.log('Avoid Type Assertions:');
console.log('  ✓ Design APIs to avoid assertions');
console.log('  ✓ Use type guards instead\n');

// ✅ Use const assertions for literals
const ACCOUNT_TYPES = ['SAVINGS', 'CURRENT', 'FIXED'] as const;
type AccountTypeFromArray = typeof ACCOUNT_TYPES[number];
// Result: 'SAVINGS' | 'CURRENT' | 'FIXED'

console.log('Const Assertions:');
console.log('  ✓ Preserve literal types');
console.log('  ✓ Make arrays/objects readonly\n');

// ✅ Leverage utility types
type RequiredFields = Required<Partial<BankAccount>>;
type OptionalFields = Partial<Required<BankAccount>>;

console.log('Use Utility Types:');
console.log('  ✓ Partial, Required, Readonly');
console.log('  ✓ Pick, Omit, Record');
console.log('  ✓ Avoid manual type manipulation\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// EDGE CASES SUMMARY
// =============================================================================

console.log('TYPESCRIPT EDGE CASES SUMMARY:\n');

console.log('Type Safety:');
console.log('  • Prefer unknown over any');
console.log('  • Use type guards for narrowing');
console.log('  • Avoid unchecked assertions');
console.log('  • Enable strict mode\n');

console.log('Structural Typing:');
console.log('  • TypeScript uses duck typing');
console.log('  • Use branded types for nominal typing');
console.log('  • Be aware of compatible structures\n');

console.log('Generics:');
console.log('  • Use constraints to ensure capabilities');
console.log('  • Leverage conditional types');
console.log('  • Avoid overcomplicating\n');

console.log('Immutability:');
console.log('  • Use readonly for properties');
console.log('  • DeepReadonly for nested objects');
console.log('  • const assertions for literals\n');

console.log('Best Practices:');
console.log('  ✓ Enable all strict flags');
console.log('  ✓ Use interfaces for objects');
console.log('  ✓ Use type for unions/primitives');
console.log('  ✓ Prefer type guards over assertions');
console.log('  ✓ Leverage built-in utility types');
console.log('  ✓ Use const assertions');
console.log('  ✓ Document complex types');
console.log('  ✓ Keep types simple and readable\n');

console.log('Common Pitfalls:');
console.log('  ✗ Using any instead of unknown');
console.log('  ✗ Unsafe type assertions');
console.log('  ✗ Ignoring strict null checks');
console.log('  ✗ Over-engineering types');
console.log('  ✗ Not using type guards\n');

console.log('Remember:');
console.log('  "TypeScript\'s type system is your friend.');
console.log('   Use it to catch errors early, not to fight against."\n');

console.log();
```

### Key Takeaways:

**Decorators**:
- Method decorators for logging, performance
- Class decorators for metadata
- Property decorators for validation
- Used in Angular, NestJS, TypeORM
- Enable experimentalDecorators

**TypeScript + Node.js**:
- Proper tsconfig.json configuration
- Type-safe Express applications
- DTOs for request/response
- Middleware with types
- Organized project structure

**Edge Cases & Best Practices**:
- Prefer unknown over any
- Use type guards for safety
- Branded types for nominal typing
- Generic constraints for flexibility
- Readonly for immutability
- Enable strict mode always

---

**End of Questions 4-6**
