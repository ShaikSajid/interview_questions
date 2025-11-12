# Questions 1-3: TypeScript Fundamentals

## Question 1: What is TypeScript and what are its key features? Explain type system basics.

### Answer:

**TypeScript** is a strongly-typed superset of JavaScript that compiles to plain JavaScript. It adds optional static typing, classes, interfaces, and modern JavaScript features, providing better tooling, error detection at compile-time, and improved code maintainability.

### Key Features:
1. **Static Typing** - Type checking at compile time
2. **Type Inference** - Automatic type detection
3. **Interfaces** - Contract definitions for objects
4. **Generics** - Reusable type-safe components
5. **Enums** - Named constants
6. **Advanced Types** - Union, Intersection, Conditional types

### Banking Scenario: Type-Safe Banking System at Emirates NBD

```typescript
console.log('=== TypeScript Fundamentals - Emirates NBD ===\n');

// =============================================================================
// 1. BASIC TYPES
// =============================================================================

console.log('1. BASIC TYPES:\n');

// Primitive types
const accountNumber: string = 'ACC-123456789';
const balance: number = 50000;
const isActive: boolean = true;
const lastLogin: Date = new Date();

console.log('Primitive Types:');
console.log(`  Account: ${accountNumber} (string)`);
console.log(`  Balance: ${balance} (number)`);
console.log(`  Active: ${isActive} (boolean)`);
console.log(`  Last Login: ${lastLogin.toISOString()} (Date)\n`);

// Arrays
const transactionIds: string[] = ['TXN-001', 'TXN-002', 'TXN-003'];
const amounts: Array<number> = [5000, 2000, 10000];

console.log('Array Types:');
console.log(`  Transaction IDs: ${transactionIds.join(', ')}`);
console.log(`  Amounts: ${amounts.join(', ')}\n`);

// Tuple - fixed-length array with specific types
const accountInfo: [string, number, boolean] = ['ACC-123456789', 50000, true];

console.log('Tuple Type:');
console.log(`  Account Info: [${accountInfo.join(', ')}]\n`);

// Enum - named constants
enum AccountType {
    SAVINGS = 'SAVINGS',
    CURRENT = 'CURRENT',
    FIXED_DEPOSIT = 'FIXED_DEPOSIT',
    BUSINESS = 'BUSINESS'
}

enum TransactionType {
    DEPOSIT = 'DEPOSIT',
    WITHDRAWAL = 'WITHDRAWAL',
    TRANSFER = 'TRANSFER',
    INTEREST = 'INTEREST'
}

console.log('Enum Types:');
console.log(`  Account Types: ${Object.values(AccountType).join(', ')}`);
console.log(`  Transaction Types: ${Object.values(TransactionType).join(', ')}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. INTERFACES AND TYPE ALIASES
// =============================================================================

console.log('2. INTERFACES AND TYPE ALIASES:\n');

// Interface - describes object shape
interface BankAccount {
    accountNumber: string;
    accountType: AccountType;
    balance: number;
    currency: string;
    isActive: boolean;
    createdAt: Date;
    updatedAt?: Date; // Optional property
}

// Type Alias - alternative to interface
type Transaction = {
    id: string;
    accountNumber: string;
    type: TransactionType;
    amount: number;
    timestamp: Date;
    description?: string;
};

// Interface extending another interface
interface SavingsAccount extends BankAccount {
    interestRate: number;
    minimumBalance: number;
}

// Type intersection
type AccountWithTransactions = BankAccount & {
    transactions: Transaction[];
};

// Using interfaces
const savingsAccount: SavingsAccount = {
    accountNumber: 'ACC-123456789',
    accountType: AccountType.SAVINGS,
    balance: 50000,
    currency: 'AED',
    isActive: true,
    createdAt: new Date(),
    interestRate: 0.035,
    minimumBalance: 1000
};

console.log('Savings Account:');
console.log(`  Account: ${savingsAccount.accountNumber}`);
console.log(`  Type: ${savingsAccount.accountType}`);
console.log(`  Balance: ${savingsAccount.balance} ${savingsAccount.currency}`);
console.log(`  Interest Rate: ${(savingsAccount.interestRate * 100).toFixed(2)}%\n`);

// Using type alias
const transaction: Transaction = {
    id: 'TXN-001',
    accountNumber: 'ACC-123456789',
    type: TransactionType.DEPOSIT,
    amount: 5000,
    timestamp: new Date(),
    description: 'Cash deposit'
};

console.log('Transaction:');
console.log(`  ID: ${transaction.id}`);
console.log(`  Type: ${transaction.type}`);
console.log(`  Amount: ${transaction.amount}`);
console.log(`  Description: ${transaction.description || 'N/A'}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. TYPE INFERENCE AND TYPE ASSERTIONS
// =============================================================================

console.log('3. TYPE INFERENCE:\n');

// Type inference - TypeScript automatically infers types
const inferredString = 'ACC-987654321'; // string
const inferredNumber = 30000; // number
const inferredArray = [1000, 2000, 3000]; // number[]

console.log('Type Inference (automatic):');
console.log(`  inferredString: "${inferredString}" is inferred as string`);
console.log(`  inferredNumber: ${inferredNumber} is inferred as number`);
console.log(`  inferredArray: [${inferredArray}] is inferred as number[]\n`);

// Type assertion - telling TypeScript the type
const accountData: any = { accountNumber: 'ACC-111111111', balance: 40000 };
const typedAccount = accountData as BankAccount;

// Alternative syntax: <BankAccount>accountData

console.log('Type Assertion:');
console.log(`  Asserted type from 'any' to BankAccount`);
console.log(`  Account: ${typedAccount.accountNumber}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. UNION AND INTERSECTION TYPES
// =============================================================================

console.log('4. UNION AND INTERSECTION TYPES:\n');

// Union type - value can be one of several types
type AccountIdentifier = string | number;
type TransactionAmount = number | string; // Can accept "5000" or 5000

function getAccount(identifier: AccountIdentifier): string {
    if (typeof identifier === 'string') {
        return `Account: ${identifier}`;
    } else {
        return `Account ID: ${identifier}`;
    }
}

console.log('Union Types:');
console.log(`  ${getAccount('ACC-123456789')}`);
console.log(`  ${getAccount(12345)}\n`);

// Discriminated unions (tagged unions)
interface DepositTransaction {
    kind: 'deposit';
    amount: number;
    source: string;
}

interface WithdrawalTransaction {
    kind: 'withdrawal';
    amount: number;
    destination: string;
}

interface TransferTransaction {
    kind: 'transfer';
    amount: number;
    from: string;
    to: string;
}

type BankTransaction = DepositTransaction | WithdrawalTransaction | TransferTransaction;

function processTransaction(txn: BankTransaction): string {
    switch (txn.kind) {
        case 'deposit':
            return `Deposited ${txn.amount} from ${txn.source}`;
        case 'withdrawal':
            return `Withdrew ${txn.amount} to ${txn.destination}`;
        case 'transfer':
            return `Transferred ${txn.amount} from ${txn.from} to ${txn.to}`;
    }
}

console.log('Discriminated Unions:');
console.log(`  ${processTransaction({ kind: 'deposit', amount: 5000, source: 'ATM' })}`);
console.log(`  ${processTransaction({ kind: 'transfer', amount: 3000, from: 'ACC-001', to: 'ACC-002' })}\n`);

// Intersection type - combine multiple types
interface Timestamped {
    createdAt: Date;
    updatedAt: Date;
}

interface Auditable {
    createdBy: string;
    updatedBy: string;
}

type AuditedAccount = BankAccount & Timestamped & Auditable;

const auditedAccount: AuditedAccount = {
    accountNumber: 'ACC-555555555',
    accountType: AccountType.CURRENT,
    balance: 60000,
    currency: 'AED',
    isActive: true,
    createdAt: new Date('2024-01-01'),
    updatedAt: new Date(),
    createdBy: 'admin@emiratesnbd.com',
    updatedBy: 'system@emiratesnbd.com'
};

console.log('Intersection Types:');
console.log(`  Account: ${auditedAccount.accountNumber}`);
console.log(`  Created By: ${auditedAccount.createdBy}`);
console.log(`  Updated At: ${auditedAccount.updatedAt.toISOString()}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. LITERAL TYPES AND TYPE NARROWING
// =============================================================================

console.log('5. LITERAL TYPES:\n');

// String literal types
type Currency = 'AED' | 'USD' | 'EUR' | 'GBP';
type TransactionStatus = 'PENDING' | 'COMPLETED' | 'FAILED' | 'CANCELLED';

interface CurrencyAccount {
    accountNumber: string;
    balance: number;
    currency: Currency; // Only accepts these specific strings
    status: TransactionStatus;
}

const usdAccount: CurrencyAccount = {
    accountNumber: 'ACC-USD-001',
    balance: 10000,
    currency: 'USD', // TypeScript ensures only valid currencies
    status: 'COMPLETED'
};

console.log('Literal Types:');
console.log(`  Account: ${usdAccount.accountNumber}`);
console.log(`  Currency: ${usdAccount.currency} (only AED, USD, EUR, GBP allowed)`);
console.log(`  Status: ${usdAccount.status}\n`);

// Type narrowing with type guards
function formatAmount(amount: string | number): string {
    if (typeof amount === 'string') {
        // TypeScript knows amount is string here
        return `AED ${parseFloat(amount).toFixed(2)}`;
    } else {
        // TypeScript knows amount is number here
        return `AED ${amount.toFixed(2)}`;
    }
}

console.log('Type Narrowing:');
console.log(`  ${formatAmount(5000)}`);
console.log(`  ${formatAmount('7500.50')}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 6. PRACTICAL BANKING CLASS WITH TYPES
// =============================================================================

console.log('6. PRACTICAL TYPE-SAFE BANKING:\n');

interface AccountConfig {
    minimumBalance: number;
    overdraftLimit?: number;
    interestRate?: number;
}

class TypeSafeBankAccount {
    private _balance: number;
    readonly accountNumber: string;
    readonly accountType: AccountType;
    private transactions: Transaction[] = [];
    
    constructor(
        accountNumber: string,
        accountType: AccountType,
        initialBalance: number,
        private config: AccountConfig
    ) {
        this.accountNumber = accountNumber;
        this.accountType = accountType;
        this._balance = initialBalance;
        
        console.log(`  Created ${accountType} account: ${accountNumber}`);
    }
    
    get balance(): number {
        return this._balance;
    }
    
    deposit(amount: number, description?: string): number {
        if (amount <= 0) {
            throw new Error('Amount must be positive');
        }
        
        this._balance += amount;
        this.recordTransaction(TransactionType.DEPOSIT, amount, description);
        console.log(`  Deposited: ${amount} AED`);
        return this._balance;
    }
    
    withdraw(amount: number, description?: string): number {
        if (amount <= 0) {
            throw new Error('Amount must be positive');
        }
        
        const availableBalance = this._balance + (this.config.overdraftLimit || 0);
        
        if (amount > availableBalance) {
            throw new Error('Insufficient funds');
        }
        
        if (this._balance - amount < this.config.minimumBalance) {
            throw new Error('Minimum balance requirement not met');
        }
        
        this._balance -= amount;
        this.recordTransaction(TransactionType.WITHDRAWAL, amount, description);
        console.log(`  Withdrew: ${amount} AED`);
        return this._balance;
    }
    
    private recordTransaction(
        type: TransactionType,
        amount: number,
        description?: string
    ): void {
        this.transactions.push({
            id: `TXN-${Date.now()}`,
            accountNumber: this.accountNumber,
            type,
            amount,
            timestamp: new Date(),
            description
        });
    }
    
    getTransactions(): readonly Transaction[] {
        // Return readonly array to prevent external modifications
        return this.transactions;
    }
    
    getAccountSummary(): Readonly<{
        accountNumber: string;
        accountType: AccountType;
        balance: number;
        transactionCount: number;
    }> {
        return {
            accountNumber: this.accountNumber,
            accountType: this.accountType,
            balance: this._balance,
            transactionCount: this.transactions.length
        };
    }
}

const account = new TypeSafeBankAccount(
    'ACC-777777777',
    AccountType.SAVINGS,
    50000,
    { minimumBalance: 1000, interestRate: 0.035 }
);

console.log();
account.deposit(5000, 'Salary deposit');
account.withdraw(2000, 'ATM withdrawal');

console.log();
const summary = account.getAccountSummary();
console.log('Account Summary:');
console.log(`  Account: ${summary.accountNumber}`);
console.log(`  Type: ${summary.accountType}`);
console.log(`  Balance: ${summary.balance} AED`);
console.log(`  Transactions: ${summary.transactionCount}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// TYPESCRIPT FUNDAMENTALS SUMMARY
// =============================================================================

console.log('TYPESCRIPT FUNDAMENTALS SUMMARY:\n');

console.log('Basic Types:');
console.log('  • string, number, boolean - Primitives');
console.log('  • Array<T> or T[] - Arrays');
console.log('  • [string, number] - Tuples');
console.log('  • enum - Named constants');
console.log('  • any, unknown, never - Special types\n');

console.log('Type Definitions:');
console.log('  • interface - Object shape contracts');
console.log('  • type - Type aliases');
console.log('  • Union (A | B) - One of multiple types');
console.log('  • Intersection (A & B) - Combination of types');
console.log('  • Literal types - Specific values\n');

console.log('Type Features:');
console.log('  • Type inference - Automatic type detection');
console.log('  • Type assertions - Manual type specification');
console.log('  • Type guards - Runtime type checking');
console.log('  • Readonly - Immutable properties');
console.log('  • Optional (?) - Optional properties\n');

console.log('Benefits:');
console.log('  ✓ Compile-time error detection');
console.log('  ✓ Better IDE support (autocomplete, refactoring)');
console.log('  ✓ Self-documenting code');
console.log('  ✓ Easier refactoring');
console.log('  ✓ Improved code maintainability');
console.log('  ✓ Better team collaboration\n');

console.log('When to Use:');
console.log('  ✓ Large-scale applications');
console.log('  ✓ Team projects');
console.log('  ✓ Complex business logic');
console.log('  ✓ Long-term maintenance projects');
console.log('  ✓ When type safety is critical (banking, healthcare)');

console.log();
```

---

## Question 2: Explain TypeScript Generics and how they enable reusable type-safe code

### Answer:

**Generics** allow you to create reusable components that work with multiple types while maintaining type safety. They enable you to write flexible, type-safe code without sacrificing the benefits of static typing.

### Key Concepts:
1. **Generic Functions** - Functions that work with multiple types
2. **Generic Classes** - Classes with type parameters
3. **Generic Constraints** - Limiting generic types
4. **Generic Interfaces** - Reusable interface patterns
5. **Utility Types** - Built-in generic types

### Banking Scenario: Generic Banking Components at Emirates NBD

```typescript
console.log('=== TypeScript Generics - Emirates NBD ===\n');

// =============================================================================
// 1. GENERIC FUNCTIONS
// =============================================================================

console.log('1. GENERIC FUNCTIONS:\n');

// Generic function to get first element
function getFirst<T>(array: T[]): T | undefined {
    return array[0];
}

// Type is inferred from usage
const firstAccount = getFirst(['ACC-001', 'ACC-002', 'ACC-003']);
const firstAmount = getFirst([5000, 3000, 10000]);

console.log('Generic Function:');
console.log(`  First Account: ${firstAccount} (string)`);
console.log(`  First Amount: ${firstAmount} (number)\n`);

// Generic function with multiple type parameters
function createPair<K, V>(key: K, value: V): { key: K; value: V } {
    return { key, value };
}

const accountBalance = createPair('ACC-123456789', 50000);
const transactionStatus = createPair('TXN-001', 'COMPLETED');

console.log('Multiple Generic Parameters:');
console.log(`  Account: ${accountBalance.key} = ${accountBalance.value}`);
console.log(`  Transaction: ${transactionStatus.key} = ${transactionStatus.value}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. GENERIC CLASSES
// =============================================================================

console.log('2. GENERIC CLASSES:\n');

// Generic Repository pattern
class Repository<T extends { id: string }> {
    private items: Map<string, T> = new Map();
    
    add(item: T): void {
        this.items.set(item.id, item);
        console.log(`  Added item: ${item.id}`);
    }
    
    get(id: string): T | undefined {
        return this.items.get(id);
    }
    
    getAll(): T[] {
        return Array.from(this.items.values());
    }
    
    update(id: string, item: Partial<T>): void {
        const existing = this.items.get(id);
        if (existing) {
            this.items.set(id, { ...existing, ...item });
            console.log(`  Updated item: ${id}`);
        }
    }
    
    delete(id: string): boolean {
        const deleted = this.items.delete(id);
        if (deleted) {
            console.log(`  Deleted item: ${id}`);
        }
        return deleted;
    }
    
    count(): number {
        return this.items.size;
    }
}

interface Account {
    id: string;
    accountNumber: string;
    balance: number;
    type: string;
}

interface Transaction {
    id: string;
    accountNumber: string;
    amount: number;
    type: string;
    timestamp: Date;
}

console.log('Generic Repository:\n');

// Account repository
const accountRepo = new Repository<Account>();
accountRepo.add({
    id: 'ACC-001',
    accountNumber: 'ACC-123456789',
    balance: 50000,
    type: 'SAVINGS'
});
accountRepo.add({
    id: 'ACC-002',
    accountNumber: 'ACC-987654321',
    balance: 30000,
    type: 'CURRENT'
});

console.log(`  Account count: ${accountRepo.count()}\n`);

// Transaction repository
const transactionRepo = new Repository<Transaction>();
transactionRepo.add({
    id: 'TXN-001',
    accountNumber: 'ACC-123456789',
    amount: 5000,
    type: 'DEPOSIT',
    timestamp: new Date()
});

console.log(`  Transaction count: ${transactionRepo.count()}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. GENERIC CONSTRAINTS
// =============================================================================

console.log('3. GENERIC CONSTRAINTS:\n');

// Constraint: Type must have specific properties
interface HasBalance {
    balance: number;
}

function calculateTotalBalance<T extends HasBalance>(items: T[]): number {
    return items.reduce((sum, item) => sum + item.balance, 0);
}

const accounts: Account[] = [
    { id: 'ACC-001', accountNumber: 'ACC-001', balance: 50000, type: 'SAVINGS' },
    { id: 'ACC-002', accountNumber: 'ACC-002', balance: 30000, type: 'CURRENT' },
    { id: 'ACC-003', accountNumber: 'ACC-003', balance: 100000, type: 'FIXED' }
];

const total = calculateTotalBalance(accounts);
console.log('Generic with Constraints:');
console.log(`  Total balance across ${accounts.length} accounts: ${total} AED\n`);

// Constraint: Using keyof
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
    return obj[key];
}

const account: Account = {
    id: 'ACC-001',
    accountNumber: 'ACC-123456789',
    balance: 50000,
    type: 'SAVINGS'
};

console.log('Generic with keyof:');
console.log(`  Account Number: ${getProperty(account, 'accountNumber')}`);
console.log(`  Balance: ${getProperty(account, 'balance')}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. GENERIC INTERFACES
// =============================================================================

console.log('4. GENERIC INTERFACES:\n');

// Generic response wrapper
interface ApiResponse<T> {
    success: boolean;
    data?: T;
    error?: string;
    timestamp: Date;
}

// Generic paginated response
interface PaginatedResponse<T> {
    items: T[];
    page: number;
    pageSize: number;
    total: number;
    hasMore: boolean;
}

function createSuccessResponse<T>(data: T): ApiResponse<T> {
    return {
        success: true,
        data,
        timestamp: new Date()
    };
}

function createErrorResponse<T>(error: string): ApiResponse<T> {
    return {
        success: false,
        error,
        timestamp: new Date()
    };
}

const accountResponse = createSuccessResponse<Account>({
    id: 'ACC-001',
    accountNumber: 'ACC-123456789',
    balance: 50000,
    type: 'SAVINGS'
});

console.log('Generic API Response:');
console.log(`  Success: ${accountResponse.success}`);
console.log(`  Account: ${accountResponse.data?.accountNumber}`);
console.log(`  Balance: ${accountResponse.data?.balance}\n`);

const paginatedAccounts: PaginatedResponse<Account> = {
    items: accounts,
    page: 1,
    pageSize: 10,
    total: 3,
    hasMore: false
};

console.log('Generic Pagination:');
console.log(`  Items: ${paginatedAccounts.items.length}`);
console.log(`  Page: ${paginatedAccounts.page}`);
console.log(`  Has More: ${paginatedAccounts.hasMore}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. PRACTICAL GENERIC SERVICE
// =============================================================================

console.log('5. GENERIC SERVICE LAYER:\n');

interface Entity {
    id: string;
    createdAt: Date;
    updatedAt: Date;
}

interface ValidationResult {
    isValid: boolean;
    errors: string[];
}

abstract class BaseService<T extends Entity> {
    protected repository: Repository<T>;
    
    constructor() {
        this.repository = new Repository<T>();
    }
    
    abstract validate(entity: Partial<T>): ValidationResult;
    
    async create(entity: Omit<T, 'id' | 'createdAt' | 'updatedAt'>): Promise<T> {
        const now = new Date();
        const newEntity = {
            ...entity,
            id: this.generateId(),
            createdAt: now,
            updatedAt: now
        } as T;
        
        const validation = this.validate(newEntity);
        if (!validation.isValid) {
            throw new Error(`Validation failed: ${validation.errors.join(', ')}`);
        }
        
        this.repository.add(newEntity);
        console.log(`  Created entity: ${newEntity.id}`);
        return newEntity;
    }
    
    async findById(id: string): Promise<T | undefined> {
        return this.repository.get(id);
    }
    
    async findAll(): Promise<T[]> {
        return this.repository.getAll();
    }
    
    async update(id: string, updates: Partial<T>): Promise<T | undefined> {
        const entity = await this.findById(id);
        if (!entity) return undefined;
        
        const validation = this.validate(updates);
        if (!validation.isValid) {
            throw new Error(`Validation failed: ${validation.errors.join(', ')}`);
        }
        
        this.repository.update(id, { ...updates, updatedAt: new Date() } as Partial<T>);
        return this.findById(id);
    }
    
    async delete(id: string): Promise<boolean> {
        return this.repository.delete(id);
    }
    
    private generateId(): string {
        return `ID-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
    }
}

interface BankAccountEntity extends Entity {
    accountNumber: string;
    balance: number;
    type: string;
    currency: string;
}

class AccountService extends BaseService<BankAccountEntity> {
    validate(entity: Partial<BankAccountEntity>): ValidationResult {
        const errors: string[] = [];
        
        if (entity.accountNumber && !/^ACC-\d{9}$/.test(entity.accountNumber)) {
            errors.push('Invalid account number format');
        }
        
        if (entity.balance !== undefined && entity.balance < 0) {
            errors.push('Balance cannot be negative');
        }
        
        return {
            isValid: errors.length === 0,
            errors
        };
    }
    
    async deposit(accountId: string, amount: number): Promise<BankAccountEntity | undefined> {
        const account = await this.findById(accountId);
        if (!account) throw new Error('Account not found');
        
        return this.update(accountId, { balance: account.balance + amount } as Partial<BankAccountEntity>);
    }
    
    async withdraw(accountId: string, amount: number): Promise<BankAccountEntity | undefined> {
        const account = await this.findById(accountId);
        if (!account) throw new Error('Account not found');
        
        if (account.balance < amount) {
            throw new Error('Insufficient funds');
        }
        
        return this.update(accountId, { balance: account.balance - amount } as Partial<BankAccountEntity>);
    }
}

// Using the generic service
(async () => {
    console.log('Generic Service in Action:\n');
    
    const accountService = new AccountService();
    
    const newAccount = await accountService.create({
        accountNumber: 'ACC-111111111',
        balance: 50000,
        type: 'SAVINGS',
        currency: 'AED'
    });
    
    console.log(`  Account created: ${newAccount.accountNumber}`);
    console.log(`  Balance: ${newAccount.balance}\n`);
    
    await accountService.deposit(newAccount.id, 5000);
    console.log(`  Deposited 5000 AED\n`);
    
    const updated = await accountService.findById(newAccount.id);
    console.log(`  New balance: ${updated?.balance}\n`);
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // GENERICS SUMMARY
    // =============================================================================
    
    console.log('GENERICS SUMMARY:\n');
    
    console.log('Generic Syntax:');
    console.log('  • function name<T>(param: T): T - Generic function');
    console.log('  • class Name<T> { } - Generic class');
    console.log('  • interface Name<T> { } - Generic interface');
    console.log('  • <T extends Type> - Generic constraint\n');
    
    console.log('Common Patterns:');
    console.log('  • Repository<T> - Data access layer');
    console.log('  • Response<T> - API responses');
    console.log('  • List<T> - Collections');
    console.log('  • Promise<T> - Async operations');
    console.log('  • Observable<T> - Reactive streams\n');
    
    console.log('Built-in Utility Types:');
    console.log('  • Partial<T> - All properties optional');
    console.log('  • Required<T> - All properties required');
    console.log('  • Readonly<T> - All properties readonly');
    console.log('  • Pick<T, K> - Select specific properties');
    console.log('  • Omit<T, K> - Exclude specific properties');
    console.log('  • Record<K, V> - Key-value mapping\n');
    
    console.log('Benefits:');
    console.log('  ✓ Code reusability');
    console.log('  ✓ Type safety across different data types');
    console.log('  ✓ Better IntelliSense and autocomplete');
    console.log('  ✓ Compile-time error checking');
    console.log('  ✓ Self-documenting code');
    
    console.log();
})();
```

---

## Question 3: Explain Advanced TypeScript types - Utility types, Mapped types, and Conditional types

### Answer:

**Advanced Types** in TypeScript provide powerful tools for type transformation and manipulation. They enable sophisticated type-level programming, making your code more flexible and maintainable.

### Key Advanced Types:
1. **Utility Types** - Built-in type transformations
2. **Mapped Types** - Transform properties of existing types
3. **Conditional Types** - Types based on conditions
4. **Template Literal Types** - String manipulation at type level
5. **Index Signatures** - Dynamic property types

### Banking Scenario: Advanced Type System at Emirates NBD

```typescript
console.log('=== Advanced TypeScript Types - Emirates NBD ===\n');

// =============================================================================
// 1. UTILITY TYPES
// =============================================================================

console.log('1. UTILITY TYPES:\n');

interface BankAccount {
    id: string;
    accountNumber: string;
    balance: number;
    currency: string;
    type: 'SAVINGS' | 'CURRENT' | 'FIXED';
    isActive: boolean;
    createdAt: Date;
    updatedAt: Date;
}

// Partial<T> - Makes all properties optional
type PartialAccount = Partial<BankAccount>;

const accountUpdate: PartialAccount = {
    balance: 60000,
    updatedAt: new Date()
};

console.log('Partial<T>:');
console.log(`  Update only specific fields: balance=${accountUpdate.balance}\n`);

// Required<T> - Makes all properties required
type RequiredAccount = Required<BankAccount>;

// Readonly<T> - Makes all properties readonly
type ReadonlyAccount = Readonly<BankAccount>;

const immutableAccount: ReadonlyAccount = {
    id: 'ACC-001',
    accountNumber: 'ACC-123456789',
    balance: 50000,
    currency: 'AED',
    type: 'SAVINGS',
    isActive: true,
    createdAt: new Date(),
    updatedAt: new Date()
};

console.log('Readonly<T>:');
console.log(`  Account cannot be modified: ${immutableAccount.accountNumber}\n`);

// Pick<T, K> - Select specific properties
type AccountSummary = Pick<BankAccount, 'accountNumber' | 'balance' | 'type'>;

const summary: AccountSummary = {
    accountNumber: 'ACC-123456789',
    balance: 50000,
    type: 'SAVINGS'
};

console.log('Pick<T, K>:');
console.log(`  Selected properties: ${Object.keys(summary).join(', ')}\n`);

// Omit<T, K> - Exclude specific properties
type AccountWithoutDates = Omit<BankAccount, 'createdAt' | 'updatedAt'>;

const accountNoDates: AccountWithoutDates = {
    id: 'ACC-002',
    accountNumber: 'ACC-987654321',
    balance: 30000,
    currency: 'AED',
    type: 'CURRENT',
    isActive: true
};

console.log('Omit<T, K>:');
console.log(`  Excluded properties: createdAt, updatedAt\n`);

// Record<K, V> - Create object type with specific keys and values
type AccountBalances = Record<string, number>;

const balances: AccountBalances = {
    'ACC-123456789': 50000,
    'ACC-987654321': 30000,
    'ACC-555555555': 100000
};

console.log('Record<K, V>:');
Object.entries(balances).forEach(([acc, bal]) => {
    console.log(`  ${acc}: ${bal} AED`);
});
console.log();

// ReturnType<T> - Extract return type of function
function getAccount(): BankAccount {
    return {
        id: 'ACC-001',
        accountNumber: 'ACC-123456789',
        balance: 50000,
        currency: 'AED',
        type: 'SAVINGS',
        isActive: true,
        createdAt: new Date(),
        updatedAt: new Date()
    };
}

type AccountReturnType = ReturnType<typeof getAccount>;

console.log('ReturnType<T>:');
console.log(`  Extracted return type from function\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. MAPPED TYPES
// =============================================================================

console.log('2. MAPPED TYPES:\n');

// Make all properties nullable
type Nullable<T> = {
    [P in keyof T]: T[P] | null;
};

type NullableAccount = Nullable<BankAccount>;

const nullableAccount: NullableAccount = {
    id: 'ACC-001',
    accountNumber: 'ACC-123456789',
    balance: null, // Can be null
    currency: 'AED',
    type: 'SAVINGS',
    isActive: true,
    createdAt: new Date(),
    updatedAt: null // Can be null
};

console.log('Nullable<T>:');
console.log(`  Balance: ${nullableAccount.balance ?? 'null'}`);
console.log(`  Updated: ${nullableAccount.updatedAt ?? 'null'}\n`);

// Make all properties optional and nullable
type Optional<T> = {
    [P in keyof T]?: T[P] | null;
};

// Add readonly modifier to all properties
type DeepReadonly<T> = {
    readonly [P in keyof T]: T[P];
};

// Transform all properties to promises
type Promisify<T> = {
    [P in keyof T]: Promise<T[P]>;
};

type AsyncAccount = Promisify<Pick<BankAccount, 'balance' | 'isActive'>>;

console.log('Mapped Type - Promisify:');
console.log(`  All properties wrapped in Promise<T>\n`);

// Transform property types
type Getters<T> = {
    [P in keyof T as `get${Capitalize<string & P>}`]: () => T[P];
};

type AccountGetters = Getters<Pick<BankAccount, 'balance' | 'type'>>;

// Usage would be: getBalance(), getType()

console.log('Mapped Type - Getters:');
console.log(`  Transformed to getter methods: getBalance(), getType()\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. CONDITIONAL TYPES
// =============================================================================

console.log('3. CONDITIONAL TYPES:\n');

// Basic conditional type
type IsString<T> = T extends string ? 'yes' : 'no';

type Test1 = IsString<string>; // 'yes'
type Test2 = IsString<number>; // 'no'

console.log('Conditional Type:');
console.log(`  IsString<string> = 'yes'`);
console.log(`  IsString<number> = 'no'\n`);

// Extract only function properties
type FunctionPropertyNames<T> = {
    [K in keyof T]: T[K] extends Function ? K : never;
}[keyof T];

type NonFunctionPropertyNames<T> = {
    [K in keyof T]: T[K] extends Function ? never : K;
}[keyof T];

interface AccountService {
    accountNumber: string;
    balance: number;
    deposit(amount: number): void;
    withdraw(amount: number): void;
    getBalance(): number;
}

type AccountMethods = FunctionPropertyNames<AccountService>;
// Result: 'deposit' | 'withdraw' | 'getBalance'

type AccountProperties = NonFunctionPropertyNames<AccountService>;
// Result: 'accountNumber' | 'balance'

console.log('Conditional Type - Extract Functions:');
console.log(`  Methods: deposit, withdraw, getBalance`);
console.log(`  Properties: accountNumber, balance\n`);

// Unwrap Promise type
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;

type PromiseAccount = Promise<BankAccount>;
type UnwrappedAccount = UnwrapPromise<PromiseAccount>; // BankAccount

console.log('Conditional Type - Unwrap Promise:');
console.log(`  Promise<BankAccount> → BankAccount\n`);

// Filter null and undefined
type NonNullable<T> = T extends null | undefined ? never : T;

type MaybeString = string | null | undefined;
type DefiniteString = NonNullable<MaybeString>; // string

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. TEMPLATE LITERAL TYPES
// =============================================================================

console.log('4. TEMPLATE LITERAL TYPES:\n');

// Create specific string patterns
type AccountPrefix = 'ACC' | 'SAV' | 'CUR';
type AccountNumber = `${AccountPrefix}-${string}`;

const validAccount: AccountNumber = 'ACC-123456789';
// const invalidAccount: AccountNumber = 'INV-123'; // Error

console.log('Template Literal Types:');
console.log(`  Valid format: ${validAccount}\n`);

// Event naming convention
type EventType = 'deposit' | 'withdrawal' | 'transfer';
type AccountEvent = `account:${EventType}`;

const depositEvent: AccountEvent = 'account:deposit';
const withdrawEvent: AccountEvent = 'account:withdrawal';

console.log('Event Pattern:');
console.log(`  ${depositEvent}`);
console.log(`  ${withdrawEvent}\n`);

// API endpoint types
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE';
type Endpoint = '/accounts' | '/transactions' | '/transfers';
type ApiRoute = `${HttpMethod} ${Endpoint}`;

const getAccounts: ApiRoute = 'GET /accounts';
const createTransaction: ApiRoute = 'POST /transactions';

console.log('API Routes:');
console.log(`  ${getAccounts}`);
console.log(`  ${createTransaction}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. PRACTICAL ADVANCED TYPE SYSTEM
// =============================================================================

console.log('5. PRACTICAL TYPE SYSTEM:\n');

// Type-safe event system
type EventMap = {
    'account:created': { accountNumber: string; type: string };
    'account:updated': { accountNumber: string; balance: number };
    'transaction:completed': { transactionId: string; amount: number };
    'transaction:failed': { transactionId: string; error: string };
};

class TypeSafeEventEmitter<T extends Record<string, any>> {
    private listeners: {
        [K in keyof T]?: Array<(data: T[K]) => void>;
    } = {};
    
    on<K extends keyof T>(event: K, listener: (data: T[K]) => void): void {
        if (!this.listeners[event]) {
            this.listeners[event] = [];
        }
        this.listeners[event]!.push(listener);
        console.log(`  Registered listener for: ${String(event)}`);
    }
    
    emit<K extends keyof T>(event: K, data: T[K]): void {
        const eventListeners = this.listeners[event];
        if (eventListeners) {
            console.log(`  Emitting event: ${String(event)}`);
            eventListeners.forEach(listener => listener(data));
        }
    }
}

const emitter = new TypeSafeEventEmitter<EventMap>();

console.log('Type-Safe Event System:\n');

emitter.on('account:created', (data) => {
    console.log(`    Account created: ${data.accountNumber} (${data.type})`);
});

emitter.on('transaction:completed', (data) => {
    console.log(`    Transaction completed: ${data.transactionId} - ${data.amount} AED`);
});

console.log();
emitter.emit('account:created', {
    accountNumber: 'ACC-123456789',
    type: 'SAVINGS'
});

emitter.emit('transaction:completed', {
    transactionId: 'TXN-001',
    amount: 5000
});

console.log();

// Type-safe builder pattern
class AccountBuilder {
    private account: Partial<BankAccount> = {};
    
    setAccountNumber(accountNumber: string): this {
        this.account.accountNumber = accountNumber;
        return this;
    }
    
    setBalance(balance: number): this {
        this.account.balance = balance;
        return this;
    }
    
    setType(type: 'SAVINGS' | 'CURRENT' | 'FIXED'): this {
        this.account.type = type;
        return this;
    }
    
    setCurrency(currency: string): this {
        this.account.currency = currency;
        return this;
    }
    
    build(): BankAccount {
        if (!this.account.accountNumber || !this.account.balance) {
            throw new Error('Missing required fields');
        }
        
        return {
            id: `ID-${Date.now()}`,
            accountNumber: this.account.accountNumber,
            balance: this.account.balance,
            currency: this.account.currency || 'AED',
            type: this.account.type || 'SAVINGS',
            isActive: true,
            createdAt: new Date(),
            updatedAt: new Date()
        };
    }
}

console.log('Type-Safe Builder:\n');

const builtAccount = new AccountBuilder()
    .setAccountNumber('ACC-999999999')
    .setBalance(75000)
    .setType('SAVINGS')
    .setCurrency('AED')
    .build();

console.log(`  Built account: ${builtAccount.accountNumber}`);
console.log(`  Balance: ${builtAccount.balance} ${builtAccount.currency}`);
console.log(`  Type: ${builtAccount.type}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// ADVANCED TYPES SUMMARY
// =============================================================================

console.log('ADVANCED TYPES SUMMARY:\n');

console.log('Utility Types:');
console.log('  • Partial<T> - All properties optional');
console.log('  • Required<T> - All properties required');
console.log('  • Readonly<T> - All properties readonly');
console.log('  • Pick<T, K> - Select properties');
console.log('  • Omit<T, K> - Exclude properties');
console.log('  • Record<K, V> - Key-value mapping\n');

console.log('Mapped Types:');
console.log('  • Transform existing types');
console.log('  • Add/remove modifiers (readonly, optional)');
console.log('  • Rename properties');
console.log('  • Filter properties\n');

console.log('Conditional Types:');
console.log('  • T extends U ? X : Y - Type conditions');
console.log('  • infer - Extract types');
console.log('  • Type filtering');
console.log('  • Unwrap types\n');

console.log('Template Literals:');
console.log('  • String pattern types');
console.log('  • Type-safe string unions');
console.log('  • API route patterns');
console.log('  • Event naming\n');

console.log('Benefits:');
console.log('  ✓ Type-level programming');
console.log('  ✓ Flexible type transformations');
console.log('  ✓ Better code reusability');
console.log('  ✓ Stricter type safety');
console.log('  ✓ Self-documenting patterns');

console.log();
```

### Key Takeaways:

**TypeScript Fundamentals**:
- Static typing with type inference
- Interfaces and type aliases
- Union and intersection types
- Literal types and type narrowing
- Enums for named constants

**Generics**:
- Reusable type-safe components
- Generic functions, classes, interfaces
- Generic constraints (extends)
- Repository pattern
- Type-safe service layers

**Advanced Types**:
- Utility types (Partial, Pick, Omit, Record)
- Mapped types for transformations
- Conditional types for logic
- Template literal types for patterns
- Type-safe builders and event systems

---

**End of Questions 1-3**
